---
title: "PHP Signal Safety - The Init Ordering Bug Nobody Warns You About"
description: "Embedding PHP in a multithreaded server means PHP's timeout signals can land on any thread. How ePHPm got that under control."
date: 2026-04-12T20:15:00-07:00
tags: [ ephpm, php, rust, signals, linux, tokio, debugging ]
categories: [ ephpm, php, rust ]
images:
 - src: img/2026/04/php-signal-safety-init-ordering-cover.svg
   alt: "Thread timeline: the main thread installs PHP's signal handlers during php_embed_init, a PHP thread later arms ITIMER_PROF, and SIGPROF is delivered to a tokio worker that has no PHP state"
   stretch: "none"
draft: false
---

Last week I wrote about [why ePHPm embeds PHP from Rust](/blog/2026/04/why-rust-for-php-embedding/) and promised the
signal bug its own post. Here it is. It's the kind of bug where the fix is about ten lines and the explanation is the
whole post, because nothing in PHP's embed documentation (there's barely any) tells you that initializing PHP has
process-wide side effects that assume PHP is the only thing in the process.

### The setup

The first working version of `ephpm serve` looked like every async Rust program: `#[tokio::main]`-style, build the
runtime, then inside it load config, initialize PHP, start the HTTP server. PHP requests run on `spawn_blocking`
threads. Simplified:

```rust
async fn run_serve(command: Option<Commands>) -> anyhow::Result<ExitCode> {
    let config = load_serve_config(command)?;

    // tokio's worker threads already exist at this point
    ephpm_php::PhpRuntime::init().context("failed to initialize PHP runtime")?;

    ephpm_server::serve(config).await?;
    // ...
}
```

It served pages. It also took a `SIGSEGV` inside a signal handler, on a thread that had never executed a line of PHP.
The stack under the handler was tokio's, not mine and not Zend's, which is a confusing thing to stare at. I blamed
PHP's `SIGPROF` timer and built the fix around that theory. Going back through php-src to write this up,
the theory was too simple: init doesn't arm that timer at all. The hazard is real, but it's narrower than I thought,
and knowing which builds it applies to is the useful part.

### What php_embed_init() does that you didn't ask for

`php_embed_init()` does module startup and then starts a request. Module startup sets the `LC_CTYPE` locale. Request
startup activates PHP's `zend_signal` layer, which installs deferred handlers for `SIGPROF`, `SIGHUP`, `SIGINT`,
`SIGQUIT`, `SIGTERM`, `SIGUSR1` and `SIGUSR2`, and then calls `zend_set_timeout()` to enforce `max_execution_time`.
The embed SAPI hard-codes `max_execution_time=0` (it overrides whatever php.ini says), so at init that call only
installs the timeout handler. The timer gets armed later, by the first `set_time_limit()` or
`ini_set('max_execution_time', ...)` with a non-zero value, from whatever thread is running that script. On Linux
builds without per-thread execution timers, the arming path is, trimmed down, this:

```c
/* Zend/zend_execute_API.c, the setitimer branch of zend_set_timeout_ex() */
if (seconds) {
    t_r.it_value.tv_sec = seconds;
    t_r.it_value.tv_usec = t_r.it_interval.tv_sec = t_r.it_interval.tv_usec = 0;
    setitimer(ITIMER_PROF, &t_r, NULL);
}
signo = SIGPROF;

if (reset_signals) {
    zend_signal(signo, zend_timeout_handler);
}
```

Two process-wide things are in play:

1. **Process-wide signal handlers**, installed at init: `SIGPROF` plus the `zend_signal` list above. Signal
   dispositions belong to the process, not to the thread that set them.
2. **A process-wide CPU timer**, once anything sets a non-zero limit. `ITIMER_PROF` counts CPU time consumed by the
   *whole process*, every thread, user and system. With a 30-second limit, it fires after 30 seconds of CPU across all
   threads.

That second point also means the bug is load-dependent, which is what makes it slippery. An idle server burns almost no
CPU, so the timer barely moves. A busy one with 32 threads chewing through requests can burn 30 CPU-seconds in about a
second of wall time.

### Which thread gets the signal

A process-directed signal on Linux is delivered to *one* thread that doesn't have it blocked, and the kernel picks. It
is not "the thread that called `setitimer`." PHP was written for a world where the process has one thread (CLI,
php-fpm, mod_php prefork) or where every thread is a PHP thread (ZTS under a threaded Apache MPM), so that distinction
never mattered to it.

You can watch the kernel make that choice with a page of C. `main()` plays the PHP thread: it installs a
`SIGPROF` handler and arms `ITIMER_PROF` (10ms, repeating, so we get plenty of samples). Then it starts eight threads
that burn CPU, like busy tokio workers, and goes to sleep. The handler increments a counter for whatever thread it
happens to run on:

```c
static atomic_int hits[WORKERS + 1];       // [0] = main thread
static __thread int my_slot = 0;           // thread-local, like PHP's TSRM globals

static void on_sigprof(int sig) {
    atomic_fetch_add(&hits[my_slot], 1);
}

int main(void) {
    struct sigaction sa = {0};
    sa.sa_handler = on_sigprof;
    sigaction(SIGPROF, &sa, NULL);

    struct itimerval t = {{0, 10000}, {0, 10000}};
    setitimer(ITIMER_PROF, &t, NULL);

    for (long i = 0; i < WORKERS; i++) pthread_create(&th[i], NULL, worker, (void *)(i + 1));
    sleep(3);
    /* ... stop, join, print the histogram ... */
}
```

```
$ ./sigprof
thread       SIGPROF
main              0
worker-1         95
worker-2        135
worker-3        120
worker-4        118
worker-5        103
worker-6        109
worker-7         98
worker-8         75
total           853
```

The thread that installed the handler and armed the timer got none of them, in this run and in a second one (the split
across workers moved around, main stayed at 0). It was asleep, like a PHP thread waiting on a database, and all 853
deliveries landed on threads that had nothing to do with the timer. In this toy the handler only touches a thread-local
`int`, so it's harmless. PHP's handler isn't.

### What that does to PHP

PHP's timeout handler isn't a counter bump. It reaches into engine state: the executor globals, the flags that tell the
VM to stop at the next opcode. What happens when it runs on the wrong thread depends on how PHP was built.

On the NTS build I started with, the globals are real globals, so a handler running on a tokio worker sets the flags
just fine and the PHP thread dies with "Maximum execution time exceeded" at its next opcode. The problem is the clock.
Under php-fpm a worker blocked on a database query burns no CPU, so `ITIMER_PROF` stands still while it waits. In ePHPm
the other 31 threads keep burning CPU, so a script with a 30-second limit can be killed after a second of wall time
for CPU it never used. It gets worse: NTS PHP re-arms the same timer for `hard_timeout` (2 seconds by default), and if
it fires again while the PHP thread is still stuck somewhere that doesn't check the flag, the handler calls
`_exit(124)`. On a busy box two process CPU-seconds go by in well under a tenth of a second, and that `_exit` takes the
whole server down, not one request.

Since last week ePHPm runs ZTS PHP, where the question is whether the handler can find its globals at all. Every
"global" lives in a per-thread slot handed out by TSRM. A thread gets a slot when it registers with `ts_resource(0)`,
which ePHPm does on the first PHP request a blocking thread runs, and a tokio worker that only accepts sockets never
registers. PHP 8.2 and earlier went looking for that slot anyway. PHP 8.3 added a `tsrm_is_managed_thread()` check to
both handlers: the deferred signal handler forwards to the original disposition, and the timeout handler prints
"zend_timeout_handler() called in a thread not managed by PHP" to stderr and returns. So on a current ZTS build the
symptom is a timeout that silently never happens, not a crash.

And on Linux, PHP 8.3+ turns on per-thread execution timers for ZTS builds by default
(`--enable-zend-max-execution-timers`): a POSIX timer per thread, aimed at that thread with `SIGRTMIN`, so the kernel
isn't picking a thread at all. The ZTS libphp I switched to last week has them; static-php-cli passes the flag
explicitly as well. So the `SIGPROF` problem is real for NTS builds, for ZTS builds older than 8.3 or built without
those timers, and for platforms that don't have them. It is not a problem for the build ePHPm runs today, which matters
for the second half of the fix.

### The fix, part one: do the global stuff while you're alone

The first rule I took from this: anything that touches process-global state (signal dispositions, timers, `atexit`,
locale) has to happen before there's a second thread. So PHP init moved out of the async world entirely. `main` is
synchronous again, and the tokio runtime is built only after PHP is up:

```rust
/// PHP must be initialized BEFORE the tokio runtime is created. PHP's
/// `php_embed_init()` starts a SIGPROF timer for `max_execution_time`.
/// If tokio worker threads exist when the signal fires, it gets delivered
/// to a non-PHP thread whose signal handler dereferences NULL → SIGSEGV.
///
/// The sequence is:
/// 1. Load config + init tracing (no threads)
/// 2. Init PHP + disable SIGPROF timer (still single-threaded)
/// 3. Create tokio runtime (spawns worker threads — now safe)
/// 4. Run HTTP server
fn run_serve_sync(command: Option<Commands>) -> anyhow::Result<ExitCode> {
    let config = load_serve_config(command)?;

    ephpm_php::PhpRuntime::init().context("failed to initialize PHP runtime")?;
    ephpm_php::PhpRuntime::finalize_for_http()
        .context("failed to finalize PHP runtime for HTTP")?;

    // Now safe to create the multi-threaded tokio runtime
    let rt = tokio::runtime::Runtime::new().context("failed to create tokio runtime")?;
    rt.block_on(async { ephpm_server::serve(config).await })
}
```

The comment overstates it. With `max_execution_time` at 0, init installs the handler and leaves the timer alone; I'd
lumped init together with the first script that sets a limit. The ordering rule it enforces is still right: signal
dispositions, the locale and the rest of PHP's process state get set up while nothing else is running.
(`finalize_for_http()` actually just closes out the embed's initial request; the timer is dealt with by the linker
trick below.)

That fixes init. It isn't enough on its own, because init isn't the only time PHP touches the timer, so the same change
also took the timer away from PHP entirely.

### The fix, part two: take the timer away from PHP

Every `php_request_startup()` calls `zend_set_timeout()` again, and a script's `set_time_limit()` does too, so on a
`SIGPROF` build the timer can be re-armed from a worker thread long after `main` was alone.

ePHPm links `libphp.a` statically, which means the linker gets a say. GNU ld's `--wrap=symbol` rewrites every
undefined reference to `symbol` into `__wrap_symbol`, so if the binary provides `__wrap_zend_set_timeout`, PHP's own
calls go there instead. That has to go in the *binary* crate's `build.rs`, because `rustc-link-arg` from a library crate
doesn't reach the final link (trimmed):

```rust
// crates/ephpm/build.rs
if env::var_os("PHP_SDK_PATH").is_some() {
    for func in &[
        "zend_signal_startup",
        "zend_signal_init",
        "zend_signal_deactivate",
        "zend_signal_activate",
        "zend_signal_handler_unblock",
        // zend_set_timeout directly calls sigaction(SIGPROF) +
        // setitimer(ITIMER_PROF), bypassing the zend_signal system.
        "zend_set_timeout",
        "zend_unset_timeout",
    ] {
        println!("cargo::rustc-link-arg=-Wl,--wrap={func}");
    }
}
```

And the C wrapper provides the replacements, which do nothing (trimmed, comments shortened):

```c
void __wrap_zend_signal_startup(void)        { /* skip process-wide handler install */ }
void __wrap_zend_signal_init(void)           { /* skip per-request setup + SIGPROF unblock */ }
void __wrap_zend_signal_deactivate(void)     { }
void __wrap_zend_signal_activate(void)       { }
void __wrap_zend_signal_handler_unblock(void){ }

void __wrap_zend_set_timeout(long seconds, int reset_signals)
{
    (void)seconds;
    (void)reset_signals;
    /* no-op: ePHPm manages request timeouts at the HTTP server level */
}

void __wrap_zend_unset_timeout(void) { }
```

The original definitions are still in the archive under their own names; `__real_zend_set_timeout` is how you'd reach
one if you wanted to, and nothing does.

### What that costs

Being honest about the trade, because it's not free:

- **`max_execution_time` is not enforced by PHP anymore.** A runaway script is bounded by ePHPm's HTTP request timeout
  instead, which returns a 504 to the client. The PHP thread itself keeps running until the script finishes. That's
  worse than php-fpm. PHP 8.3+ can use per-thread execution timers instead (`--enable-zend-max-execution-timers`,
  POSIX timers aimed at the owning thread via `SIGRTMIN`), and it turns them on by default for ZTS builds on Linux, so
  the ZTS libphp I switched to last week already has them. The wrap above still covers the NTS build, but on the ZTS
  build it is now throwing away a timer that was already thread-safe; letting PHP keep it is the real fix and it's on
  the list.
- **`pcntl_signal()` doesn't do anything useful.** I'm fine with this for a web server. FrankenPHP has had to work
  around the same problem (it blocks the pcntl-style signals on all of Go's threads and only lets its CLI mode see
  them).
- **`--wrap` is a GNU ld feature.** macOS's linker doesn't have it, so the macOS build will need a different answer.

### Why nobody warns you

The embed SAPI has very few serious users. Most of PHP's process-level assumptions are documented implicitly by
php-fpm's architecture: one process, one request at a time, so a process-directed signal can only ever land on the PHP
thread. The moment you put PHP in a process that has its own threads, those assumptions don't hold, and nothing in
the API tells you which functions carry process-wide side effects. You find out from a crash, or from a timeout that
makes no sense.

The broader lesson, which I've now written on a sticky note: when you embed a C runtime in your process, you inherit
all of its global side effects. Signal handlers, timers, `atexit` hooks, `setlocale`, environment reads, thread-locals
it expects someone to initialize. The runtime doesn't know it's not alone anymore. Do its global init while your process
is still single-threaded, then go through what it does per-request and decide which of those global effects you're
willing to let it keep.

### How I ran the repro

Ubuntu under WSL2 on a Ryzen 9 5950X, gcc 15.2, `gcc -O2 -pthread -o sigprof sigprof.c && ./sigprof`. The full file is
55 lines including the printing; the excerpt above is the part that matters. Exact counts will differ from run to
run and machine to machine. The point is the distribution, not the numbers.

### Links

- [ePHPm](https://github.com/ephpm/ephpm)
- [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html), see "Signal disposition" and the note that a
  process-directed signal may be delivered to any one of the threads that does not have it blocked
- [`setitimer(2)`](https://man7.org/linux/man-pages/man2/setitimer.2.html) for `ITIMER_PROF`
- GNU ld [`--wrap`](https://sourceware.org/binutils/docs/ld/Options.html)
- [PHP embed SAPI](https://github.com/php/php-src/tree/master/sapi/embed)
