---
title: "Requests That End Wrong: A 200 That Was Really a Crash"
description: "Three ePHPm bugs lived in how an embedded PHP request starts and ends. A bailout returned 200, $_POST came up empty and a SIGTERM corrupted the heap."
date: 2026-08-13T21:14:00-07:00
tags: [ php, rust, ephpm, zend, debugging ]
categories: [ ephpm, php, rust ]
images:
 - src: img/2026/08/requests-that-end-wrong-cover.svg
   alt: "Diagram of a bare zend_bailout being absorbed by php_execute_script's own zend_try instead of reaching ePHPm's guard"
   stretch: "none"
draft: false
---

The scariest response a server can send isn't a 500. It's a 200 that should have been a 500. A crash is loud. The pod
restarts, an alert fires, somebody looks. A truncated page with a success status code is quiet. Your CDN caches it and
the health check calls the box healthy. A client parses half a document as if it were the whole thing.

[ePHPm](https://github.com/ephpm/ephpm) embeds PHP inside a Rust server. So we own the part of PHP that php-fpm
normally owns: starting a request, running the script and ending it. Over the last couple of months I've fixed three
bugs that all live in that start-and-end code. None of them were in the SAPI I/O callbacks like `ub_write` and
`read_post`, where you'd expect to find embed bugs. They were in the Zend request lifecycle and in TSRM teardown.
That's code you never touch when you run PHP behind php-fpm.

Versions for context: PHP 8.5.7 ZTS from our static SDK, ePHPm around v0.6.x.

### A bare zend_bailout() came back as HTTP 200

Here is what a script terminated by a bare `zend_bailout()` returned. This is a real PHP-linked release build with a
small shared ZTS extension I use for fault injection:

```
HTTP/1.1 200 OK
content-type: text/html; charset=UTF-8
content-length: 31

calling ephpm_crash('bailout')
```

Thirty-one bytes, status 200, and nothing in the server log. `zend_bailout()` is PHP's "abandon this request" escape
hatch. It's a `longjmp` to the nearest `zend_try`. You don't call it from userland. C extensions, resource limits and
OPcache can raise it directly, and a fatal `zend_error(E_ERROR)` uses it too.

ePHPm already had a guard for this. The C wrapper puts its own `SETJMP` around the script. The Rust side turns a
bailout into a 500:

```c
EG(bailout) = &__bailout;
if (SETJMP(__bailout) == 0) {
    zend_file_handle file_handle;
    zend_stream_init_filename(&file_handle, filename);
    php_execute_script(&file_handle);
} else {
    result = EPHPM_EXEC_BAILOUT;
}
EG(bailout) = __orig_bailout;
```

It looked like it worked, because fatals came back as 500s. The trouble is *which* fatals. `php_execute_script()`
wraps the compile and execute in its **own** `zend_try` / `zend_end_try`. `zend_end_try` does not re-raise. It
restores `EG(bailout)` and falls through. So a bailout raised anywhere inside the script gets absorbed one frame below
us. It never reaches our `SETJMP`. Our `else` branch only ever ran for a bailout raised *outside*
`php_execute_script`, which basically never happens.

So why did the other fatals return 500? Different signal. There was a second check on `PG(last_error_type)`. Anything
routed through `zend_error()` sets it, like `E_ERROR`, parse errors and an uncaught `Throwable`. A bare
`zend_bailout()` sets no error type at all. It just jumps.

### CG(unclean_shutdown) is the signal that can't be absorbed

I went looking for something the engine writes on *every* bailout, no matter which `zend_try` ends up catching it.
That's `CG(unclean_shutdown)`. `_zend_bailout()` sets it to 1 unconditionally, right before the `LONGJMP`. Nothing
else in the engine writes it. `init_compiler()` clears it per request. We clear it again ourselves before running the
script, so a value can never leak over from the previous request on a reused thread:

```c
/* _zend_bailout() sets CG(unclean_shutdown) = 1 immediately before its
 * LONGJMP, unconditionally and regardless of which zend_try catches it, and
 * nothing else in the engine sets that flag. */
#define ephpm_bailout_reset()    (CG(unclean_shutdown) = 0)
#define ephpm_bailout_observed() (CG(unclean_shutdown) != 0)
```

Then comes the decision after the script. It also waits until shutdown functions and output buffers are flushed, since
a bailout in either of those truncates the response too:

```c
int fatal_error_mask = E_ERROR | E_CORE_ERROR | E_COMPILE_ERROR
                       | E_USER_ERROR | E_RECOVERABLE_ERROR | E_PARSE;
if (ephpm_bailout_observed()) {
    response_status_code = 500;
    result = EPHPM_EXEC_BAILOUT;
} else if ((PG(last_error_type) & fatal_error_mask)
           && response_status_code == 200) {
    response_status_code = 500;
}
```

The two branches deliberately have different rules. A bailout forces a 500 *unconditionally*. It's never something a
script asks for, since PHP 8's `exit()` throws an unwind-exit exception instead of bailing out. So any status the
script set earlier describes a response it never finished producing. The `last_error_type` branch only overrides a
default 200. A framework's own error handler legitimately sets its status before the engine records the fatal.

### The rule: never complete a response for a script that bailed out

Forcing the status isn't enough. What you do with the partial body depends on whether it already went out the door.
That split falls out of ePHPm's two execution modes:

- **Per-request (fpm-style) mode** buffers the whole response before it reaches the wire. So on a bailout it's still
  ours to throw away. We force the 500 and discard the truncated body *and* the script's headers. Then we log the
  script, method, URI, discarded byte count and the tail of the discarded output at `ERROR`.
- **Worker mode, headers not sent yet**: same thing. The worker stops building a response from the capture buffers,
  and the supervisor answers 500.
- **Worker mode, headers already sent**: this is the case that needs thought. A streamed response has already told the
  client `200`, and you can't take that back. So the response gets *deliberately broken*. Worker streaming uses
  `Transfer-Encoding: chunked`. A flag is set before the chunk sender is dropped. The hyper body stream reads it at
  end-of-channel and ends the body with an `io::Error`. hyper then never writes the terminating zero-length chunk.

That last one was the sneakiest part of the original bug. Before the fix, the flushed case just dropped the sender. On
the wire that's byte-for-byte identical to a finished download. With the body broken, curl exits 18 ("transfer closed
with outstanding read data remaining"). A cache won't store it. Here's the before and after from the fault-injection
lab:

| Probe | Before | After |
|---|---|---|
| bare `zend_bailout()` | 200, 31-byte truncated body, no log | 500, body discarded, `ERROR` log with output tail |
| `zend_error(E_ERROR)` | 500 | 500 |
| uncaught `Throwable` | 500 + fatal text | 500 + fatal text |
| worker, bailout mid-stream | clean 200, curl exit 0 | first chunk delivered, no terminating chunk, curl exit 18 |

There's one user-visible change beyond the bug. Fatals that do bail out, like the memory limit or `E_USER_ERROR`, used
to come back as a 500 *with* the partial output and PHP's fatal text in the body. Now that text goes to the server log
instead, and the client gets a generic 500. That follows the rule, and I'm fine with it. A half-rendered page with a
stack trace glued to the bottom isn't something I want to send anybody. That's
[ephpm#255](https://github.com/ephpm/ephpm/pull/255).

### Earlier: $_POST was empty and status codes leaked between requests

The bailout bug has a cousin from back in June. That's when ePHPm first started running a real
`php_request_shutdown()` / `php_request_startup()` cycle per request
([ephpm#104](https://github.com/ephpm/ephpm/pull/104)). Before that, a reused thread kept its user functions, classes
and globals across requests. Vanilla WordPress only rendered the first request per thread.

Moving to a real per-request cycle exposed two problems in the code around it. The first was a SIGSEGV under load. The
old wrapper rebuilt the superglobals by hand after startup. That was fine when startup didn't run per request. Once it
did, the hand-rolled rebuild destroyed the `PG(http_globals)` arrays startup had just created and re-ran `treat_data`
over them. That's a use-after-free, and it faulted on tokio's blocking threads under load. The fix was to delete the
rebuild and fill in `SG(request_info)` *before* `php_request_startup()`. Then PHP builds the superglobals itself, the
way php-fpm does.

That's when `$_POST` went empty. `php://input` still worked, but `$_POST` stayed empty for every form post. The reason
is a gate in `sapi_activate()`. It only reads and parses the POST body into `$_POST` when `SG(server_context)` is
non-NULL. That's how PHP tells a real request apart from a CLI run, and the embed SAPI never sets it. Our callbacks
don't use that pointer at all, so the fix is a sentinel:

```c
/* sapi_activate() only reads and parses the POST body into $_POST when
 * SG(server_context) is non-NULL. A stable non-NULL sentinel is all that's
 * needed to enable native POST parsing. */
SG(server_context) = &ephpm_server_context_marker;

if (php_request_startup() != SUCCESS) {
    return EPHPM_EXEC_STARTUP_FAIL;
}
```

The value is never dereferenced. It only has to be non-NULL.

The same PR caught a status leak. `http_response_code` isn't reset by request startup. The reset in `sapi_activate()`
is commented out in php-src. So a `201` on one request could come back on the next `200` served by the same thread. A
`500` from a fatal could too. Resetting it, `headers_sent` and `no_headers` after startup fixed it. A 40-way
concurrent e2e test now checks that every request sees fresh per-request state.

### SIGTERM after WordPress: zend_mm_heap corrupted

The last one is about how the *process* ends, and it was deterministic. Serve a few WordPress pages and send SIGTERM.
Every shutdown died with `zend_mm_heap corrupted` and exit code 134. Plain docroots never tripped it. The fatal-signal
reporter I'd added a few days before ([ephpm#253](https://github.com/ephpm/ephpm/pull/253)) symbolized the chain.
That's the only reason this took an afternoon instead of a week:

```
PhpRuntime::shutdown -> php_embed_shutdown -> php_module_shutdown
  -> module_destructor -> ts_free_id -> (pcre GSHUTDOWN)
  -> zend_string_release_ex -> _efree -> zend_mm_panic
```

Under ZTS, every thread that registers with TSRM is supposed to free its own slot with `ts_free_thread()`. It has to
happen *on that thread*, before the main thread runs `php_module_shutdown()`. ePHPm registered its PHP threads and
never unregistered them. On top of that, a thread's last request stayed open. Request teardown was lazy and ran at the
top of the thread's *next* request. So `PHP_RSHUTDOWN(pcre)` never ran for it.

WordPress is what made it fatal. PHP's pcre extension caches compiled patterns per thread. For patterns with named
subpatterns it also caches the group names as request-lifetime strings. Those are `emalloc`'d on the compiling
thread's own Zend MM heap. WordPress compiles named-group regexes on every serving thread. At shutdown, `ts_free_id()`
walks *every* thread's TSRM entry and runs pcre's global shutdown for each copy *on the main thread*. So the main
thread `efree`s a pointer that belongs to worker thread X's heap, and the allocator panics.

You can't fix that by running module shutdown on "the right thread", because the strings belong to N different heaps.
The fix is the mirror image of how init works. Init is pinned to a single-threaded phase before tokio starts, and
shutdown now goes back to one:

1. Every PHP thread retires its own TSRM slot at thread exit. A thread-local guard's `Drop` runs
   `php_request_shutdown()` + `ts_free_thread()` on the exiting thread, so request-lifetime state is freed into the
   heap that allocated it.
2. Global shutdown waits until every PHP thread is gone:

```rust
// Drop the runtime BEFORE tearing down PHP (issue #266). Dropping joins
// every worker and blocking-pool thread; each PHP-registered blocking
// thread's TLS guard then runs php_request_shutdown + ts_free_thread on
// the thread itself.
drop(rt);

// Shutdown PHP runtime: all PHP threads are gone, only the main
// thread's TSRM entry remains.
ephpm_php::PhpRuntime::shutdown().context("failed to shutdown PHP runtime")?;
```

Before: 3 of 3 runs exited 134. After: 3 of 3 exited 0, in both per-request and worker mode. That's
[ephpm#268](https://github.com/ephpm/ephpm/pull/268). It matters more than it sounds. On Kubernetes, every rolling
restart was logging a crash. Exit-code alerting can't tell "crashed during graceful shutdown" from "crashed".

### What the three have in common

All three bugs are about the edges of a request. That's the instant it starts, the instant the script stops and the
instant the thread or the process goes away. php-fpm handles those edges for you. When you embed PHP you inherit them.
The documentation for this part of PHP is mostly the source. So the fixes all came from reading `main.c`, `SAPI.c`,
`zend.c` and `TSRM.c` until the behavior made sense.

The pattern I'm taking from it is simple. For each of these, find the one piece of engine state that's written on
every path and key off that. Here that was `CG(unclean_shutdown)`, a non-NULL `SG(server_context)` and "this thread
freed its own slot". Don't key off whatever flag happens to be set on the path you tested.

### Links

- [ephpm#255](https://github.com/ephpm/ephpm/pull/255) - a bare `zend_bailout()` returned HTTP 200 with a truncated
  body
- [ephpm#104](https://github.com/ephpm/ephpm/pull/104) - per-request state isolation, the `$_POST` sentinel and the
  status leak
- [ephpm#266](https://github.com/ephpm/ephpm/issues/266) / [ephpm#268](https://github.com/ephpm/ephpm/pull/268) -
  SIGTERM after WordPress aborted with `zend_mm_heap corrupted`
- [ephpm#253](https://github.com/ephpm/ephpm/pull/253) - the async-signal-safe fatal-signal reporter that symbolized
  the shutdown crash
- PHP source: [`Zend/zend.c`](https://github.com/php/php-src/blob/master/Zend/zend.c) (`_zend_bailout`),
  [`main/SAPI.c`](https://github.com/php/php-src/blob/master/main/SAPI.c) (`sapi_activate`),
  [`TSRM/TSRM.c`](https://github.com/php/php-src/blob/master/TSRM/TSRM.c) (`ts_free_id`, `ts_free_thread`)
