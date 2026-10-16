---
title: "We Disabled PHP's Stack Guard Ourselves: Containing Segfaults in a Single-Process PHP Server"
description: "How ePHPm went from dying silently on a segfault to containing stack overflows per thread. It also covers the linker flag that had turned off PHP's own guard."
date: 2026-09-06T15:05:00-07:00
tags: [ php, rust, ephpm, segfault, signals, multi-tenant ]
categories: [ ephpm, php, security ]
images:
 - src: img/2026/09/php-stack-guard-cover.svg
   alt: "Diagram of a PHP thread stack, the guard window below it, and which faults get contained versus which kill the process"
   stretch: "none"
draft: false
---

php-fpm has a property nobody thinks about until they leave it. When a worker segfaults, you lose one request. The
master forks a new child and life goes on. [ePHPm](https://github.com/ephpm/ephpm) runs every tenant's PHP as threads
inside one process. That's where the memory and latency wins come from. It also means one tenant's segfault takes down
every other tenant on the box.

This is the story of closing that gap in four steps over about two weeks. The last step turned out to be a bug we'd
put there ourselves. It's also the payoff for the thread pool work in
[the previous post](/blog/2026/09/own-your-php-thread-pool/). Everything below that actually shipped depends on ePHPm
owning its own PHP threads.

### Step 1: at least say something on the way out

The first problem wasn't recovery at all. Kubernetes restarts a crashed pod in well under a second. We measured 61 to
90 ms to respawn. The problem was that ePHPm died with **no output**. A null dereference in an extension produced
total silence. The only things that ever said anything were Rust's own stack-overflow message and glibc's malloc
checker. An operator got a restarted pod and no way to tell whether an extension, a native middleware or PHP itself
did it.

[ephpm#253](https://github.com/ephpm/ephpm/pull/253) added a fatal-signal reporter for SIGSEGV, SIGBUS, SIGILL, SIGFPE
and SIGABRT. Everything about it is shaped by one fact. It runs inside a process that's already broken:

- **Async-signal-safe throughout.** Output is a raw `write(2)` to fd 2. Integers and pointers are formatted by hand
  into a fixed buffer. Only lock-free atomics are used. `tracing`, `format!` and `println!` all allocate or lock, and
  the faulting thread might already hold the allocator lock.
- **`backtrace()` is warmed at startup** so its lazy init doesn't happen in a broken process. Raw addresses are
  written *before* symbolization is attempted, because `_dl_addr` takes the loader lock.
- **`sigaltstack` + `SA_ONSTACK`**, or a stack overflow can't be reported at all. The handler would need stack to run
  on.
- **It chains to the previous handler and dies with the original disposition.** Every fault type was run twice on the
  same binary, with the handler on and off. The exit codes matched: 139 for segfaults and 134 for aborts. Kubernetes
  and exit-code alerting see exactly what they saw before.

And deliberately **no recovery**. The reasoning from the PR is worth quoting: "a ZMM smash was measured returning
HTTP 200 to its own client and killing the process 17 ms later on a *different* thread serving an unrelated request."
After heap corruption there's no request to blame and no thread to isolate.

### Step 2: the spike, and the wall it hit

Then a pentest found the process-killer that made this urgent. It's ordinary PHP. Build a linked list of around 50,000
plain objects and let it go out of scope. Freeing it recurses in C. `zend_object_std_dtor` goes into
`zend_objects_store_del` and back, one pair of frames per node, until the thread runs out of stack. Nothing in
userland reaches it. Not `try`/`catch`, not `disable_functions` and not `memory_limit`.

But a stack overflow is a different animal from heap corruption. The fault is in a *thread-local* resource: the
thread's own stack. The objects being freed live in that thread's Zend MM heap and `EG(objects_store)`. Zend MM
`mmap`s its own 2 MiB chunks instead of going through glibc `malloc`. So no process-wide allocator lock is held when
it faults. Abandon that one thread and the rest of the process is fine.

The spike was [ephpm#294](https://github.com/ephpm/ephpm/pull/294), and it was never merged on purpose. It built that
in pure libc. That way it compiles and is unit tested without PHP at all. A per-thread `sigsetjmp` region wraps each
request. The fatal signal handler asks a classifier first:

```c
int ephpm_guard_try_recover(uintptr_t fault_addr)
{
    if (!g_armed) {
        return 0;
    }
    if (g_stack_lo == 0) {
        return 0; /* bounds unknown on this platform: never contain */
    }
    /* Below the guard window? (written to avoid unsigned underflow) */
    if (fault_addr + EPHPM_GUARD_WINDOW < g_stack_lo) {
        return 0;
    }
    /* Well above the low bound: a wild write, not an overflow. */
    if (fault_addr > g_stack_lo + EPHPM_GUARD_SLACK) {
        return 0;
    }
    g_armed = 0;
    siglongjmp(g_recover_buf, 1);
    return 0; /* unreachable */
}
```

Only a fault at or just below **this thread's own** stack low bound gets contained. The window is 64 KiB below it,
with one page of slack above. The bounds come from `pthread_getattr_np` at arm time. They never come from inside the
handler, because that call isn't async-signal-safe. The state lives in `__thread` variables because a signal handler
can read those safely. A wild write lands nowhere near the stack. It isn't contained and dies with the usual
diagnostic and exit code. The cover image above is that function as a picture.

The mechanism worked. The process survived 100 concurrent plus 120 paced crashes. There was zero corruption across
3,591 concurrent requests to a second tenant that round-tripped its own database. It also hit two walls. Both came
from running PHP on tokio's `spawn_blocking` pool:

1. **A poisoned thread can't be retired.** After a contained crash, that thread's Zend context is garbage. It has to
   refuse all further PHP. But tokio owns the thread. There's no API to say "kill this one and give me a fresh one".
   During a 100-crash storm, 17 threads got poisoned and the healthy pool shrank. The co-tenant's success rate was
   **13.7%**.
2. **Graceful shutdown aborted after even one contained crash.** `php_module_shutdown()` walks every TSRM entry,
   including the abandoned poisoned one. It frees that entry's corrupt heap from the main thread and gets
   `zend_mm_heap corrupted`. Exit 134 on every rolling deploy after any tenant crash.

Containment turned "one crash kills every tenant until restart" into "degraded but alive". That's better, but it isn't
isolation.

### Step 3: ship it on threads we own

The fix for both walls was the owned thread pool. Once ePHPm runs PHP on its own `std::thread`s, retiring a poisoned
one is a `break` out of the job loop:

```rust
let poisoned = matches!(&out, Err(PhpError::Contained(_)));
// The receiver is gone only if the router already timed out this
// request (504) and moved on; dropping the response is correct.
let _ = respond_to.send(out);
if poisoned {
    counter!("ephpm_fpm_pool_contained_crashes_total").increment(1);
    exit = ThreadExit::Poisoned;
    break;
}
```

followed by the rule that matters most:

```rust
if exit == ThreadExit::Poisoned {
    // DO NOT run worker_thread_shutdown() here. It is php_request_shutdown()
    // + ts_free_thread(), and both walk this thread's ZMM free lists and
    // EG(objects_store), which the contained siglongjmp left mid-mutation.
    // Running them would free out of a corrupt heap and abort the process,
    // undoing the containment.
    counter!("ephpm_fpm_pool_recycles_total", "reason" => "poisoned").increment(1);
} else {
    PhpRuntime::worker_thread_shutdown();
}
```

A poisoned thread is abandoned and never cleaned up. That's the same trade a hung thread already got. It leads to the
process-level invariant from [ephpm#302](https://github.com/ephpm/ephpm/pull/302): **once any crash has been
contained, this process must never call `php_embed_shutdown()`.** That closed the second wall. Consecutive poisons
inside a 10-second window also back off 25 ms up to 500 ms before respawning. A client in a crash loop can't make us
spin thread creation and TSRM registration at request rate.

![Bar chart: co-tenant requests succeeded during a 100-crash storm, 13.7% on the spike versus 100% on the owned pool](/img/2026/09/php-stack-guard-containment-storm.svg)

These are the numbers from the PR, on PHP 8.5.7 ZTS with a 4-thread pool. They come from two different storm setups.
Read the chart as before and after rather than as a controlled A/B:

| Check | Result |
|---|---|
| Crashes contained | 1,400+, each one a `500`, PID never changed |
| Threads retired and replaced | live thread count flat at 37 across all 1,400 |
| Co-tenant availability | 100 concurrent crashes + 1,500 concurrent requests: **1,500/1,500 OK** (spike: 13.7%) |
| RSS | +90 MiB over the first ~1,000 crashes, then flat |
| SIGTERM after a contained crash | exit 0, module shutdown skipped (was 134) |
| Heap-corruption class | `tgkill` SIGSEGV to a thread inside an armed guard: still exit 139 |

That last row is the one I care most about. The classifier has to say no just as reliably as it says yes. It's tested
both ways: live, and with a forked-child unit test. In that test a real null-deref write inside an armed guard has to
re-raise and die. A later benchmark put the guard's happy-path cost at +0.4% RPS, inside the noise. It still ships as
`[php] crash_containment = false` by default and labeled experimental. There are two reasons. Each contained crash
leaks a Zend context, which is bounded but real. And it only covers one fault class.

### Step 4: the twist, it was our linker flag

Meanwhile there was a much older open issue,
[ephpm#116](https://github.com/ephpm/ephpm/issues/116): WordPress block themes crash the worker. `do_blocks()` on
nested block markup took down the resident PHP worker. It had been filed as a WordPress-in-worker-mode problem and
sat on the "adapter quirks" pile.

When I went back to it, it still reproduced on `main`. And not just in worker mode:

| | depth | result |
|---|---|---|
| worker mode | 1,000 | `200` |
| worker mode | 20,000 | `thread 'ephpm-worker-0' has overflowed its stack`, process SIGABRT |
| fpm mode | 3,000 | process SIGABRT on a `tokio-rt-worker` thread |
| stock `php` | any | `Error: Maximum call stack size of 8339456 bytes ... reached` |

Stock PHP throws a *catchable error* for that. PHP 8.3 added a C-stack guard. At request startup
`zend_call_stack_init()` reads the thread's real stack bounds into `EG(stack_limit)`. The VM and `zend_call_function`
check it before recursing. Some recursion re-enters the VM through an internal function like `array_map`,
`preg_replace_callback` or `call_user_func_array`. That's exactly what `do_blocks()` to `render_block()` to
`apply_filters()` does once per nesting level. That kind of recursion raises an `Error` instead of running off the end
of the stack.

ePHPm had switched it off. `build.rs` had this, from early in the project:

```rust
let mut wrap_funcs: Vec<&str> = vec![
    "zend_signal_startup",
    "zend_signal_init",
    // ...
    // zend_call_stack_init probes stack boundaries on each request
    // startup. Fails on tokio spawn_blocking threads with small/
    // non-standard stacks. We disable stack checking anyway.
    "zend_call_stack_init",
];
for func in &wrap_funcs {
    println!("cargo::rustc-link-arg=-Wl,--wrap={func}");
}
```

and the C side defined the wrapper as a no-op:

```c
void __wrap_zend_call_stack_init(void)
{
    /* no-op: stack checking is disabled */
}
```

`-Wl,--wrap=sym` tells GNU ld to send every call to `sym` to `__wrap_sym` instead. It's how you override a function
inside a static archive you didn't compile. The theory in the comment was wrong on both counts. Probing doesn't fail
on tokio threads. PHP reads the bounds with `pthread_getattr_np`, which is exact on any thread glibc creates. And even
a probe that *did* fail would just leave `EG(stack_limit)` NULL. That means the guard is off, not a crash. What the
wrap actually did was leave `EG(stack_limit)` NULL forever. So `zend_call_stack_overflowed()` was permanently false.
That was on Linux only. macOS and Windows have no `--wrap` and kept the guard the whole time.

There was a second layer of the same misunderstanding. The wrapper also set `EG(max_allowed_stack_size) = 0` in several
places, with comments saying "disable stack size checking". Zero is PHP's *default* and means "auto-detect from the
real stack". The disable value is `-1`. Those writes never disabled anything. They just made the code look like it
was doing something on purpose.

### The fix is two parts, and the second one matters

[ephpm#360](https://github.com/ephpm/ephpm/pull/360) dropped the wrap and the no-op. It also deleted the
`max_allowed_stack_size` writes. But re-arming the guard alone would have swapped a crash for a spurious error at a
quarter of php-fpm's depth. PHP derives the ceiling from the running thread's stack. Rust and tokio threads default to
2 MiB. php-fpm runs PHP on the process main thread with `ulimit -s`, which is 8 MiB on every mainstream Linux. So
every thread that runs PHP now gets an 8 MiB stack:

```rust
pub const PHP_THREAD_STACK: usize = 8 * 1024 * 1024;

let builder = std::thread::Builder::new()
    .name(format!("ephpm-fpm-{thread_id}"))
    .stack_size(ephpm_php::PHP_THREAD_STACK);
```

That's applied to the tokio runtime threads, the per-request pool and the worker pool. It's reserved address space
that gets committed a page at a time. There's no RSS cost. Measured on an 8 MiB stack, the test fixture completes
11,000 levels and fatals at 13,000. That's the same on PHP 8.3.31 and 8.5.7, about 750 bytes of C stack per level. The
e2e test asserts 5,000 levels complete. That's more than 2x margin. It's also about twice what a 2 MiB thread would
allow. So it's a real test of the thread stack size and not a tautology.

| after | depth | result |
|---|---|---|
| worker mode | 5,000 | `200` |
| worker mode | 20,000 / 60,000 / 200,000 | clean `500`, server alive, next request served |
| fpm mode | 20,000+ | clean `500` with PHP's own message |

And because the overflow is now a catchable `Error`, a worker adapter that catches it keeps serving with no recycle
at all. That was #116's acceptance criterion.

### Both guards stay

PHP's guard and the signal-level containment don't overlap. Here's precisely why. PHP's guard is a *checkpoint*. It's
tested in the VM and in `zend_call_function`. Recursion that goes through PHP code or internal callbacks hits a
checkpoint and becomes an `Error`. The destructor cascade from the pentest recurses purely through C destructors and
never passes one. PHP can't see it. The signal handler is still the only thing that catches that one, and
`crash_containment` still needs the owned pool. Worker mode can't contain that class yet. It owns its OS threads,
which is the precondition. So it's tractable, but it's a feature-sized change of its own.

### Takeaway

The uncomfortable part of this story is where the bug lived. The bug that took down every tenant on a block-theme page
wasn't in PHP or WordPress. It was a linker flag with a confident comment. It was written early on, when PHP threads
lived on tokio's blocking pool and "small, weird stacks" seemed like a reasonable thing to defend against. Nobody
re-tested the assumption, because nothing ever failed in a way that pointed at it.

`--wrap` is a powerful tool for embedding a C runtime. ePHPm still uses it for PHP's process-wide signal handling.
Every wrapped symbol is a feature of the runtime you've switched off. It deserves the same scrutiny as any other
config default. Each one needs a test that would fail if the reason for it stopped being true. As it happens, this
wasn't the only feature we'd turned off this way. `zend_set_timeout` had been wrapped to a no-op too. That's why
`max_execution_time` did nothing in ePHPm until August. That one gets its own post.

### Links

- [ephpm#253](https://github.com/ephpm/ephpm/pull/253) - report fatal signals instead of dying silently
- [ephpm#294](https://github.com/ephpm/ephpm/pull/294) - the containment spike (not merged, kept as the record)
- [ephpm#302](https://github.com/ephpm/ephpm/pull/302) - contain stack-overflow crashes and retire the poisoned pool thread
- [ephpm#116](https://github.com/ephpm/ephpm/issues/116) / [ephpm#360](https://github.com/ephpm/ephpm/pull/360) - re-arm PHP's C-stack overflow guard
- [ephpm/multitenant-scalebench BEFORE-AFTER.md](https://github.com/ephpm/multitenant-scalebench/blob/main/report/BEFORE-AFTER.md) - the guard's happy-path cost (point 5)
- php-src: [`Zend/zend_call_stack.c`](https://github.com/php/php-src/blob/master/Zend/zend_call_stack.c)
- [GNU ld `--wrap`](https://sourceware.org/binutils/docs/ld/Options.html)
