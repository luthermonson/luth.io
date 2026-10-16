---
title: "Why Rust for PHP Embedding (Not Go)"
description: "I'm a Go developer building a PHP application server in Rust. Here's the FFI reasoning, the C wrapper it forced on me, and what cgo actually costs when you measure it."
date: 2026-04-05T19:40:00-07:00
tags: [ ephpm, rust, go, php, ffi, cgo ]
categories: [ ephpm, rust, php ]
images:
 - src: img/2026/04/why-rust-for-php-embedding-cover.svg
   alt: "Diagram of a request flowing from hyper/tokio through a Rust blocking thread, a C wrapper with setjmp, and libphp, with SAPI callbacks writing into thread-local C buffers"
   stretch: "none"
draft: false
---

Go is my primary language and has been for a long time, so when I started building [ePHPm](https://github.com/ephpm/ephpm)
a few weeks ago the obvious move was to write it in Go. ePHPm is a PHP application server: one binary that embeds the
real PHP interpreter (`libphp`, the Zend engine) and puts an HTTP server, TLS, a KV cache and a database proxy around it,
so a PHP app doesn't need nginx + php-fpm + Redis to run. [FrankenPHP](https://frankenphp.dev/) already proved the
"PHP inside a Go web server" idea works, and it works well. I wrote it in Rust anyway. This post is the reasoning, the
part of the design that Rust made easier and the part it didn't, and some numbers I should have measured before I wrote
the design doc instead of after.

### What embedding PHP actually means

PHP is a C library if you build it with `--enable-embed=static` (plain `--enable-embed` gives you a shared
`libphp.so`). You get `libphp.a`, a header set, and the embed SAPI
(`php_embed_init()` / `php_embed_shutdown()`). Everything else is your problem: where output goes, where the POST body
comes from, how `$_SERVER` gets filled. PHP asks for all of that through the SAPI, a struct of function pointers
(`ub_write`, `read_post`, `send_headers`, `register_server_variables` and friends) that the host fills in. php-fpm,
Apache's mod_php and the CLI are all just different implementations of that struct.

So the host language talks to PHP in two directions:

- **Host calls PHP**: start a request, execute a script, read back results.
- **PHP calls host**: every `echo` is a `ub_write` callback, reading `php://input` is `read_post`, building `$_SERVER`
  is `register_server_variables`.

The second direction is the one that matters for this comparison, because it happens many times per request and it's
where the language boundary gets crossed from the "wrong" side.

### What my design doc said

When I sat down to compare runtimes I wrote up FrankenPHP in some detail (its Caddy module, its worker loop, how it
parks a PHP thread on a Go channel). The headline in my own project notes (it's in `AGENTS.md` and the architecture doc) was this line:

> Go's CGO incurs ~200ns overhead per call. FrankenPHP crosses the CGO boundary 11+ times per request (~2.2μs+ overhead).
> Rust FFI to libphp is zero overhead.

The 11 crossings came from walking a request through FrankenPHP: dispatch to the PHP thread, the SAPI callbacks for
superglobals, headers, cookies, body reads, output writes, and the trip back to finish the request. The 200ns was a
number I'd carried around in my head for years from older Go releases. I didn't measure it. That's the kind of thing
that's fine in a design doc and not fine in a blog post, so I measured it.

### Measuring the boundary

The benchmark is about as small as it gets. Three C functions, compiled with `gcc -O2` and marked
`noinline`, linked into both a Go test binary and a Rust binary (each side gets its own copy of the file; this is the
Go one):

```c
#include <stddef.h>
__attribute__((noinline)) void noop(void) {}
__attribute__((noinline)) size_t write_buf(char *p, size_t n) { return n + (p != 0); }
extern size_t goUbWrite(char *p, size_t n);   /* rust_ub_write on the Rust side */
__attribute__((noinline)) size_t c_calls_host(char *p, size_t n) { return goUbWrite(p, n); }
```

`write_buf` is the shape of `ub_write` (a pointer and a length), and `c_calls_host` is C calling back into the host
language, which is what every SAPI callback does if the host owns the output buffer. On the Go side it's a cgo package
with an `//export`'d `goUbWrite` and a normal `testing.B` loop. On the Rust side it's an `unsafe extern "C"` block, a
`#[unsafe(no_mangle)] pub extern "C" fn rust_ub_write`, and a loop of 200 million calls timed with `Instant`, with
`black_box` on the arguments so nothing gets folded away. Each case ran five times and I took the median.

![Bar chart of nanoseconds per call for Go cgo versus Rust extern C across four cases](/img/2026/04/why-rust-for-php-embedding-ffi-cost.svg)

| Case | Go 1.26 cgo | Rust 1.94 |
|------|------------:|----------:|
| Plain function call, no FFI (baseline) | 1.20 ns | 1.15 ns |
| Empty C call | 23.6 ns | 1.15 ns |
| C call with pointer + length | 29.3 ns | 1.39 ns |
| C calls back into the host | 61.2 ns | 1.60 ns |

So my doc was wrong, and wrong in Go's favor. A cgo call on Go 1.26 is about 24ns on this box, not 200. Going the other
way, C calling into an exported Go function, is about 61ns. Rust calling C is indistinguishable from Rust calling Rust,
because it *is* the same thing: the same calling convention, the same stack, a `call` instruction.

If I redo the napkin math with real numbers, 11 crossings is somewhere between roughly 0.25µs (all cheap calls) and 0.7µs
(all callbacks) per request. That is real, it's measurable, and it is still small next to running a WordPress page.
Anyone who tells you cgo overhead alone is what makes Go the wrong choice here is doing what I did in the design doc.

### So why Rust anyway

The per-call number isn't the whole cost of putting a Go runtime and the Zend engine in the same process. The things
that actually pushed me over:

**There's no second runtime in the way.** A cgo call isn't just a jump. The goroutine has to move onto the system stack
and tell the scheduler it's leaving Go, and a callback from C into Go has to get back onto a goroutine (and a C thread
that Go didn't create has to be adopted by the runtime first). FrankenPHP creates each PHP thread in C with
`pthread_create`, calls back into Go from it, and parks it on a channel select waiting for the next request, so the
goroutine scheduler never gets to move it. All of that is
well-engineered and all of it is machinery that exists because there are two runtimes in the process with different
ideas about threads and stacks. In Rust a PHP thread is an OS thread. I call `ts_resource(0)` on it once to register it
with PHP's thread-safe resource manager and that's the whole story.

**No GC sharing the process with Zend.** PHP has its own allocator and its own refcounting/GC. Go's GC doesn't scan C
memory, so it's not a correctness problem, but it is a second source of pauses in the same process as your PHP
requests, and tail latency is one of the numbers I care about for this project.

**The scope is bigger than an HTTP front end.** ePHPm isn't only going to serve PHP. The plan is an in-process KV store
that PHP can call into directly, a pooling MySQL/Postgres proxy, gossip clustering. Most of those are hot paths where PHP
calls *out* of the Zend engine into host code, and that is exactly the 61ns-vs-1.6ns row in the table. I'd rather that
direction be free from day one than try to batch around it later.

**I want to own the binary.** FrankenPHP is a Caddy build. That's a feature for them. For ePHPm I wanted the HTTP stack
(hyper), the TLS/ACME stack, and the PHP lifecycle all under my control, with nothing that has to be upstreamed into a
second project before I can change how a request is dispatched.

None of that makes Go bad at this. FrankenPHP exists and people run real traffic on it. It's a trade I'm comfortable
with because I'm the one paying the Rust learning tax.

### The part Rust doesn't solve: setjmp/longjmp

Here's the catch that every embedder hits regardless of language. PHP's error handling is `setjmp`/`longjmp`. A fatal
error, an `exit()` in older paths, a memory limit, a timeout: the Zend engine calls `zend_bailout()`, which `longjmp`s
back to the nearest `zend_try`. The `zend_try`/`zend_catch` macros expand to a `SETJMP` on a jump buffer stored in
executor globals.

You can't do that from Rust. `setjmp` "returns twice," which Rust's compiler doesn't model, and a `longjmp` across Rust
frames skips their destructors and is undefined behavior. If PHP longjmps through a Rust frame that holds a
`MutexGuard` or a `Vec`, you've leaked or corrupted something and you won't find out until later.

The fix is boring and mandatory: Rust never calls into the Zend engine directly for anything that can bail out. It calls
a C function, and the C function owns the jump buffer. This is the core of `ephpm_execute_request()` in
`crates/ephpm-php/ephpm_wrapper.c` (trimmed):

```c
int ephpm_execute_request(const char *filename)
{
    /* ... reset buffers, fill SG(request_info), rebuild superglobals ... */

    int result = 0;
    JMP_BUF *__orig_bailout = EG(bailout);
    JMP_BUF __bailout;

    EG(bailout) = &__bailout;
    if (SETJMP(__bailout) == 0) {
        zend_file_handle file_handle;
        zend_stream_init_filename(&file_handle, filename);
        php_execute_script(&file_handle);

        /* PHP 8.x: exit()/die() throws an unwind exit exception instead
         * of calling zend_bailout(). Treat it like the old bailout path. */
        if (EG(exception) && zend_is_unwind_exit(EG(exception))) {
            zend_clear_exception();
            result = -2;
        }
    } else {
        /* PHP bailed out (fatal error) */
        result = -2;
    }
    EG(bailout) = __orig_bailout;

    capture_response_headers();
    response_status_code = SG(sapi_headers).http_response_code;
    return result;
}
```

The `longjmp` lands in a C frame, the C function returns an `int`, and Rust only ever sees a normal return. The rules
the rest of the code follows fall out of that:

- Every Rust → PHP entry point goes through a C function that sets up the bailout.
- SAPI callbacks are C, and they write into thread-local C buffers (`output_buf`, `headers_buf`). They don't call into
  Rust while PHP might still bail out above them.
- Rust reads the buffers back after the C function returns.

On the Rust side, the call is a handful of `CString`s that have to outlive the call, and then a plain function call:

```rust
unsafe extern "C" {
    pub fn ephpm_request_set_info(
        method: *const c_char, uri: *const c_char, query_string: *const c_char,
        content_type: *const c_char, cookie: *const c_char,
        post_data: *const c_char, post_data_len: usize, path_translated: *const c_char,
    );
    pub fn ephpm_execute_request(filename: *const c_char) -> c_int;
    pub fn ephpm_get_output_buf(out_len: *mut usize) -> *const c_char;
}

// method, uri, script_filename... are CStrings owned by this stack frame,
// so the pointers C holds stay valid until ephpm_execute_request returns.
let ret = unsafe { ffi::ephpm_execute_request(script_filename.as_ptr()) };

let body = {
    let mut len: usize = 0;
    let ptr = unsafe { ffi::ephpm_get_output_buf(&mut len) };
    if ptr.is_null() || len == 0 { Vec::new() }
    else { unsafe { std::slice::from_raw_parts(ptr.cast::<u8>(), len) }.to_vec() }
};
```

The C wrapper is just under 1,400 lines at this point, and most of it is SAPI plumbing I'll write up separately. The
Rust side is mostly `CString` conversions and `// SAFETY:` comments. That split, Rust for everything that's async or
long-lived, C for anything that touches Zend's control flow, has held up so far.

### Threads

The other thing the diagram at the top hides is threading. PHP comes in two flavors: NTS (non-thread-safe, one request
at a time per process, which is why php-fpm forks) and ZTS (Zend Thread Safety, where every global lives in a
per-thread slot managed by TSRM). For the first couple of weeks ePHPm ran NTS PHP behind a mutex, which is a fine way
to get correct output and a terrible way to serve concurrent requests.

As of this week it's ZTS. hyper and tokio handle connections asynchronously, and each PHP request runs on a tokio
`spawn_blocking` thread. The first time a given thread runs PHP it calls into C to register with TSRM and start a
request context; after that it's a thread-local `bool` check. All the per-request C state in the wrapper is
`__thread`. Rust's threads are OS threads, so there's no question about which thread a piece of code runs on.

One thing that is *not* free, and that I learned the hard way the same week I wrote the code above: PHP's init installs
process-wide signal handlers (including the `SIGPROF` handler behind `max_execution_time`), and those do not care that
your process has 32 tokio threads in it.
That bug gets its own post.

### What I'd tell a Go developer considering this

If you want PHP inside a Go web server, use FrankenPHP. It's mature, it's under the PHP Foundation now, and 24ns per
crossing is not what's going to hurt you. If you're building something where PHP calls back into host code constantly,
where you want one runtime in the process, and where you're willing to write a C shim for Zend's control flow either way,
Rust gets out of the way in a way cgo structurally can't.

And measure your own numbers before you put them in a design doc.

### How I measured

- AMD Ryzen 9 5950X (16 cores / 32 threads), Ubuntu under WSL2, kernel 6.18, gcc 15.2
- Go 1.26.0 (`go test -bench`, five runs of about 2 seconds each), Rust 1.94.0 (`cargo build --release`, opt-level 3)
- The same three C functions are linked into both binaries (two copies differing only in `const` and the callback
  name); they are `noinline` and live in a separate object, so neither compiler can inline across the boundary (no
  cross-language LTO)
- Numbers are the median of five runs per case

### Links

- [ePHPm](https://github.com/ephpm/ephpm)
- [FrankenPHP](https://frankenphp.dev/) and its [source](https://github.com/php/frankenphp)
- [cgo documentation](https://pkg.go.dev/cmd/cgo)
- [The Rustonomicon on FFI](https://doc.rust-lang.org/nomicon/ffi.html), whose section on FFI and unwinding is the reason you don't let anything unwind or `longjmp` through a Rust frame
- [PHP embed SAPI source](https://github.com/php/php-src/tree/master/sapi/embed)
