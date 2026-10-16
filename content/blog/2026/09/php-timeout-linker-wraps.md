---
title: "The Bugs Were Our Own Linker Wraps: Making a PHP Timeout Actually Time Out"
description: "max_execution_time did nothing in ePHPm because we had wrapped zend_set_timeout to a no-op. The SDK already had per-thread timers; we just had to stop switching them off."
date: 2026-09-16T19:35:00-07:00
tags: [ php, rust, ephpm, linker, signals, config ]
categories: [ ephpm, php ]
images:
 - src: img/2026/09/php-timeout-linker-wraps-cover.svg
   alt: "Side by side: a process-wide SIGPROF timer that can land on the wrong thread, versus a per-thread POSIX timer that delivers SIGRTMIN to the PHP thread that armed it"
   stretch: "none"
draft: false
---

In [the stack guard post](/blog/2026/09/php-stack-guard-segfault-containment/) the villain was a `-Wl,--wrap` linker
flag that turned off PHP's C-stack overflow guard. It wasn't the only one. Until August, `max_execution_time` did
nothing in [ePHPm](https://github.com/ephpm/ephpm) on Linux, for the same reason: we had wrapped the function that
arms it to a no-op. A runaway script ran until the HTTP layer's request timeout gave up on it and sent a 504, and the
PHP thread kept burning CPU after that.

This one is shorter, and it's really about a habit: a setting you parse and don't enforce is worse than a setting you
don't have.

### Why the wrap existed in the first place

When you embed PHP, you inherit its global side effects, and signals are the worst of them. PHP's classic timeout uses
`setitimer(ITIMER_PROF)`, which delivers SIGPROF to the *process*. The kernel then hands it to whatever thread it
likes. In php-fpm that's fine, since each process has one PHP thread. In ePHPm, SIGPROF could land on a tokio worker
thread with no PHP state at all, and PHP's handler would dereference per-request globals that don't exist there and
segfault.

So early in the project, `build.rs` wrapped PHP's signal plumbing and the timeout functions, and the C wrapper
replaced them with no-ops:

```c
void __wrap_zend_set_timeout(long seconds, int reset_signals)
{
    (void)seconds;
    (void)reset_signals;
    /* no-op: ePHPm manages request timeouts at the HTTP server level */
}

void __wrap_zend_unset_timeout(void)
{
    /* no-op */
}
```

`--wrap=zend_set_timeout` makes GNU ld resolve every call to `zend_set_timeout` inside `libphp.a` to
`__wrap_zend_set_timeout` instead. It's a clean way to override a function in a static archive you didn't compile,
and at the time it was the correct call. But it meant `[php] max_execution_time` was parsed, documented and ignored.
A docs audit landing the same week had it marked "parsed but not enforced".

### The SDK already had the fix compiled in

PHP added `--enable-zend-max-execution-timers` partway through the 8.1 line (8.1.18 and 8.2.5, April 2023). It
replaces the process-wide itimer with a per-thread POSIX timer. The question was whether our SDK had it. php-fpm users never think about this flag, and the docs for it are
thin, so I checked the artifacts directly:

- `php_config.h` in the Linux ZTS SDK has `#define ZEND_MAX_EXECUTION_TIMERS 1`.
- `libphp.a` defines `zend_max_execution_timer_init`, `_settime` and `_shutdown`, and they reference
  `timer_create` and `timer_settime`.
- Disassembling `zend_max_execution_timer_init` shows the important part:
  `timer_create(CLOCK_BOOTTIME, SIGEV_THREAD_ID -> gettid(), SIGRTMIN)`, with the handler installed through a raw
  `sigaction` and not through the `zend_signal_*` functions we also wrap.

That last point matters. `SIGEV_THREAD_ID` means the kernel delivers the signal to one specific thread, the PHP thread
that armed it. The whole reason for the wrap was gone. The source matches what the disassembly shows,
in `Zend/zend_max_execution_timer.c`:

```c
sev.sigev_notify = SIGEV_THREAD_ID;
sev.sigev_signo = SIGRTMIN;
...
if (timer_create(ZEND_MAX_EXECUTION_TIMERS_CLOCK, &sev, &EG(max_execution_timer_timer)) != 0) {
```

and `zend_set_timeout()` calls into `zend_max_execution_timer_settime()` when that's compiled in. So the only thing
standing between ePHPm and a working timeout was our own linker flag.

### Probe, don't assume

The fix in [ephpm#277](https://github.com/ephpm/ephpm/pull/277) doesn't just delete the wrap, because not every SDK
we build against has per-thread timers (macOS doesn't, for one). Both build scripts read `php_config.h` and only drop
the wrap when the define is present:

```rust
let has_exec_timers = php_config_has_max_exec_timers(&sdk_path.join("include").join("php"));
if has_exec_timers {
    println!("cargo::rustc-cfg=php_max_exec_timers");
}

let mut wrap_funcs: Vec<&str> = vec![
    "zend_signal_startup",
    "zend_signal_init",
    "zend_signal_deactivate",
    "zend_signal_activate",
    "zend_signal_handler_unblock",
    // still wrapped at the time; #360 (the stack guard post) removed it
    "zend_call_stack_init",
];
if !has_exec_timers {
    // No per-thread timers: zend_set_timeout() arms a process-wide
    // setitimer(ITIMER_PROF) + SIGPROF handler, which crashes when the
    // signal lands on a tokio worker thread.
    wrap_funcs.push("zend_set_timeout");
    wrap_funcs.push("zend_unset_timeout");
}
```

The C side gets the matching `EPHPM_NATIVE_EXEC_TIMER` define, and the no-op stubs only compile when it's absent.
On builds without the timers nothing changes: the HTTP request timeout is the only ceiling, and startup warns.

### The second thing turning it off: the embed SAPI

Dropping the wrap wasn't enough on its own. The timer got created and stayed disarmed. The embed SAPI ships its own
defaults, and `max_execution_time=0` is one of them:

```c
static const char HARDCODED_INI[] =
	"html_errors=0\n"
	"implicit_flush=1\n"
	"output_buffering=0\n"
	"max_execution_time=0\n"
	"max_input_time=-1\n\0";
```

In our setup, by the time a script ran, both the ini entry and `EG(timeout_seconds)` read 0, so PHP's own
ini-to-timer chain armed nothing. Instead of fighting that, ePHPm records the configured value once from Rust after
`php_embed_init()` and arms the timer explicitly per request, right before executing the script:

```c
#ifdef EPHPM_NATIVE_EXEC_TIMER
    /* php_request_startup above already created the timer but armed it from
     * EG(timeout_seconds), which the embed SAPI has reset to 0, so we arm
     * explicitly from the configured value here. reset_signals = 1 also
     * (re)installs the SIGRTMIN disposition. */
    zend_set_timeout(g_configured_max_exec_secs, 1);
#endif
```

Arming from a process-wide configured value has a nice side effect: a previous request's `set_time_limit(0)` (which
changes the ini entry at runtime) can't leak into the next request on the same thread. `set_time_limit()` during a
request still re-arms the timer live on top of it, the same as in php-fpm.

The result also needed its own status code. A timeout unwinds through `zend_bailout()` like an out-of-memory fatal,
and since [ephpm#255](/blog/2026/08/requests-that-end-wrong/) a bailout's partial output gets discarded. But a timeout's
output is a legitimate php-fpm-style error page, so it's told apart by PHP's own message and delivered:

```c
if (PG(last_error_message)
    && strncmp(ZSTR_VAL(PG(last_error_message)),
               "Maximum execution time of", 25) == 0) {
    result = EPHPM_EXEC_TIMEOUT;
} else {
    result = EPHPM_EXEC_BAILOUT;
}
```

### What it does now

From the PR, on a PHP-linked build of PHP 8.5.7 with `max_execution_time = 2` and a 30-second request timeout:

| case | result |
|---|---|
| CPU busy loop | 500 at 2.004 s, buffered output flushed, shutdown function ran |
| `sleep(5)` | 500 at 2.002 s (`CLOCK_BOOTTIME` is wall clock, so sleeping counts) |
| `set_time_limit(10)` | 500 at 10.002 s |
| `set_time_limit(0)` | 504 at 30.002 s, only the outer request timeout fires |
| blocking `fread` on an empty FIFO | 500 at 2.004 s |
| normal request afterwards | 200, same thread, no poisoning |

The FIFO row is my favorite. The timer interrupts a read that's blocked in the kernel, which the HTTP-layer timeout
could never do: it could only stop waiting for the thread, not stop the thread. Worker mode got the same treatment,
plus disarm-on-idle so a worker sitting in `take_request()` for longer than the limit doesn't fatal itself, and a
cross-request test proving a `set_time_limit(0)` in one request doesn't leave the next one unlimited.

So there are two layers now, and the docs say so: `max_execution_time` is PHP's own catchable, `set_time_limit`-able,
wall-clock limit that answers 500, and `[server.timeouts] request` is the outer 504 backstop. Startup warns when
`max_execution_time >= request`, since the inner limit could then never fire first.

### The wider pattern: silent settings

Two linker wraps in this project switched off a PHP feature and left the config pretending otherwise. Once I started
looking, the same thing turned up without any linker involved.

**An idle timeout that was parsed and thrown away.** `[server.timeouts] idle` was read into a variable and dropped,
so the documented 60-second default did nothing. [ephpm#106](https://github.com/ephpm/ephpm/pull/106) implemented it
(an I/O adapter stamps activity on every byte and a re-arming watchdog triggers hyper's graceful shutdown after a
quiet window), and in the same pass turned every other silent no-op it found into either a working knob or a loud
startup warning.

**Typos and knobs from newer versions.** A config key the binary didn't know was silently ignored. This bit for real:
`per_site = true` on a binary that predated the knob parsed fine, got dropped on the floor, and brought the node up
with every tenant sharing one database, with every health check green.
[ephpm#434](https://github.com/ephpm/ephpm/pull/434) made every config section `deny_unknown_fields`, so an unknown
key is a startup error that names the key. The top-level struct deliberately stays lenient, because figment maps
every `EPHPM_*` environment variable to a top-level key, and ePHPm sets one itself when it runs as a Windows service.
The migration note in that PR has my favorite sentence of the year: if startup fails naming a key, that key was
already doing nothing.

**Estimates nobody measured.** Two old perf issues estimated that rebuilding `$_SERVER` per request and round-tripping
response headers as text cost about 0.15 ms and 0.05 ms. When [ephpm#457](https://github.com/ephpm/ephpm/pull/457)
actually measured them, they were about 4 µs and 1 µs, off by roughly 40x. The fix still went in (it also closed a
header-injection gap on the worker path), but it's a few microseconds per request, not the 20% the estimates implied.

The common thread: a knob that's parsed but doesn't do anything, a feature compiled in and wrapped out, a number
nobody measured. Each one looks fine from the outside. For the linker case specifically, every `--wrap` in an
embedder's build is a runtime feature you've switched off, and it should have a test that fails the day its reason
stops being true. The stack-guard regression tests do that now: they fail with a transport error, not a status
mismatch, if the guard ever goes away again. The timeout matrix above was still run by hand when it merged, and
turning it into an automated e2e test is on the follow-up list. Writing this post is a decent reminder to go do it.

### Links

- [ephpm#277](https://github.com/ephpm/ephpm/pull/277) - natively enforce `max_execution_time` via per-thread timers
- [ephpm#360](https://github.com/ephpm/ephpm/pull/360) - the other wrap: re-arming PHP's C-stack guard
- [ephpm#106](https://github.com/ephpm/ephpm/pull/106) - make silent no-op config knobs real or loud
- [ephpm#434](https://github.com/ephpm/ephpm/pull/434) - reject unknown keys in every config section
- [ephpm#457](https://github.com/ephpm/ephpm/pull/457) - the `$_SERVER` and header round-trip, measured
- php-src: [`Zend/zend_max_execution_timer.c`](https://github.com/php/php-src/blob/master/Zend/zend_max_execution_timer.c), [`sapi/embed/php_embed.c`](https://github.com/php/php-src/blob/master/sapi/embed/php_embed.c)
- [GNU ld `--wrap`](https://sourceware.org/binutils/docs/ld/Options.html)
