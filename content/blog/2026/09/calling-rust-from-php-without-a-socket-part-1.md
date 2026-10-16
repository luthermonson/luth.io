---
title: "Calling Rust From PHP Without a Socket, Part 1: The Native Functions"
description: "How ePHPm exposes its in-process KV store and SQLite bridge to PHP as plain functions, and what it took to make that safe under ZTS."
date: 2026-09-14T20:40:00-07:00
tags: [ ephpm, php, rust, ffi, redis, sqlite, zts ]
categories: [ ephpm, rust, php ]
series: [ "Calling Rust From PHP Without a Socket" ]
images:
 - src: img/2026/09/calling-rust-from-php-without-a-socket-part-1-cover.svg
   alt: "Two paths from PHP to the ePHPm KV store: over a RESP socket, or through a native function and a function pointer table"
   stretch: "none"
draft: false
---

[ePHPm](https://github.com/ephpm/ephpm) is a single Rust binary that embeds PHP. It carries its own KV store and its own
SQLite-backed database in the same process. For a long time PHP code could only reach that KV store the same way it
reaches Redis. It opened a TCP socket to the RESP listener ePHPm runs on localhost, encoded a command and waited for the
reply. That works, and every PHP Redis client in existence already speaks it. But it is a strange thing to do when the
hash map you're talking to lives a few pointers away in the same address space.

So ePHPm registers its own PHP functions: `ephpm_kv_get()`, `ephpm_kv_set()`, `ephpm_db_query()` and friends. Calling one
is a C function call that hops through a function pointer into Rust and back. This is part one of two. This post is the
plumbing:

- how the functions get registered
- how C and Rust share a table of function pointers
- who owns which bytes
- the thread-teardown bugs that came with it

[Part 2](/blog/2026/09/calling-rust-from-php-without-a-socket-part-2/) is the 13 Composer packages that let phpredis,
Predis, WordPress and Laravel code use them without being rewritten.

### Is it worth it?

Before writing a post about avoiding a socket I wanted my own numbers, not the ones in the docs. I took the
[ephpm-lab](https://github.com/ephpm/lab) KV harness and ran it against the v0.10.6 release. The harness is one PHP
request that loops over the operation and times it with `hrtime()`. I pointed the same raw-socket RESP client at two
servers. One was ePHPm's own RESP listener and the other was a stock Redis 7 in a container.

![KV GET latency: native function vs ePHPm RESP vs Redis](/img/2026/09/calling-rust-from-php-without-a-socket-part-1-kv-get.svg)

Medians of three interleaved runs:

| | 64 B | 4 KB | 64 KB |
| --- | ---: | ---: | ---: |
| `ephpm_kv_get()` | 0.105 µs | 0.176 µs | 2.04 µs |
| `ephpm_kv_set()` | 0.165 µs | 0.210 µs | 1.19 µs |
| GET over RESP to ePHPm | 83.0 µs | 86.7 µs | 117.8 µs |
| GET over RESP to Redis 7.4 | 86.8 µs | 88.9 µs | 123.3 µs |

A small `get` through the native function is about 105 ns. The same `get` over a socket is about 83 µs. That's roughly
790 times slower, and it barely matters which server is on the other end. Real Redis comes in within 5% of ePHPm's own
listener. That's the useful result. The cost isn't the server, it's the round trip.

I'd be careful with the absolute socket number. It's high because this is WSL2, where a loopback round trip is
expensive. To check it wasn't my PHP client, I ran `redis-benchmark -c 1` inside the Redis container. That's plain C
with no PHP involved, and it got a p50 of 79 µs. On bare-metal Linux loopback is a lot cheaper, so expect the gap to
shrink by some factor there. It doesn't go away, though. A native call doesn't have a round trip at all.

The other thing the table shows is that "no serialization" isn't "no copy". At 64 KB the native `get` is memcpy-bound at
about 2 µs. The value is copied out of the store and then copied again into a PHP string.

### What PHP sees

Everything is registered from one macro in `ephpm_wrapper.c`. The server and the CLI mode, `ephpm php`, get the same
list:

```c
#define EPHPM_COMMON_FUNCTION_ENTRIES \
    PHP_FE(ephpm_kv_get,       arginfo_ephpm_kv_get) \
    PHP_FE(ephpm_kv_set,       arginfo_ephpm_kv_set) \
    PHP_FE(ephpm_kv_setnx,     arginfo_ephpm_kv_setnx) \
    PHP_FE(ephpm_kv_del,       arginfo_ephpm_kv_del) \
    PHP_FE(ephpm_kv_del_if_eq, arginfo_ephpm_kv_del_if_eq) \
    /* ... exists, incr, decr, incr_by, expire, ttl, pttl, flush_all, wait */ \
    PHP_FE(ephpm_db_query,          arginfo_ephpm_db_query) \
    PHP_FE(ephpm_db_execute,        arginfo_ephpm_db_execute) \
    PHP_FE(ephpm_db_run,            arginfo_ephpm_db_run) \
    /* ... columns, in_transaction, available, errno, error */ \
    PHP_FE(ephpm_ws_send,           arginfo_ephpm_ws_send) \
    /* ... eight more ephpm_ws_* */ \
    PHP_FE(ephpm_middleware_config, arginfo_ephpm_middleware_config)
```

That's 14 KV functions, 8 database functions, 9 WebSocket functions and the middleware config accessor. On top of those
there's a small module with the `\Ephpm\Worker\*` functions for worker mode. There's also a native session save handler
you turn on with `session.save_handler = ephpm`. Every one of them follows the same pattern, so I'll stick to the KV and
DB side.

One function worth calling out is `ephpm_kv_wait()`. It blocks until a key's version changes. You can't do that over
RESP without pub/sub. It replaced a poll-and-`usleep` loop in the Datastar demo. In
[#184](https://github.com/ephpm/ephpm/pull/184) eight idle SSE clients went from 0.35% of a core to 0.02%. Event latency
went from a p50 of 100 ms, the poll interval, to 1.2 ms.

### Getting the functions registered at all

This took three tries. The first two looked fine in the code and failed in the e2e suite.

The obvious move is to set `additional_functions` on the SAPI module struct before calling `php_embed_init()`. That does
nothing. `php_embed_init()` overwrites `php_embed_module.additional_functions` with its own one-entry array, just
`dl()`. That happens at `sapi/embed/php_embed.c:219`, after `sapi_startup()` and before module startup. Every request
died with "Call to undefined function ephpm_kv_get()".

The second attempt (`752890eb`) called `zend_register_functions()` right after `php_embed_init()` returned. On a non-ZTS
build that's fine. But ePHPm runs PHP threaded (ZTS). In ZTS, `zend_post_startup()` runs at the tail of
`php_module_startup()`. It copies the main thread's `CG(function_table)` into a static `GLOBAL_FUNCTION_TABLE` and frees
the original. Every new PHP thread builds its function table by copying from `GLOBAL_FUNCTION_TABLE` in
`compiler_globals_ctor`. Register a function after that point and it lands in a table no worker thread will ever look
at.

The window that works is "after the embed SAPI wipes `additional_functions`, before `php_module_startup()` reads it".
`php_embed_init()` rewrites `additional_functions` but leaves the `startup` hook alone. So the fix (`a50a69ba`) swaps the
startup hook for a shim:

```c
static int ephpm_module_startup(sapi_module_struct *sm)
{
    sm->additional_functions = g_cli_mode ? ephpm_cli_functions : ephpm_kv_functions;

    /* worker module goes in as php_module_startup's additional_module so its
     * functions land in CG(function_table) DURING zend_startup */
    int ret = php_module_startup(sm, &ephpm_worker_module_entry);

    /* after: ext/session's MINIT has to have run. Before the embed init
     * returns: under ZTS that is the last moment new threads will see it */
    if (ret == SUCCESS) {
        (void)php_session_register_module(&ps_mod_ephpm);
    }
    return ret;
}

void ephpm_pre_init(void)
{
    php_embed_module.startup = ephpm_module_startup;
}
```

The same window turned out to be the right place for several later things. The CLI's `-d` directives and `PHP_BINARY`
are spliced in there too. That says something about how much of the embed SAPI's setup happens in that one call.

### The ops table

The C side doesn't know anything about Rust types. It declares a struct of function pointers and keeps one static copy:

```c
typedef struct {
    int  (*get)(const char *key);
    void (*get_result)(const char **ptr, size_t *len);
    int  (*set)(const char *key, const char *val, size_t val_len, long long ttl_ms);
    int  (*set_nx)(const char *key, const char *val, size_t val_len, long long ttl_ms);
    long (*del)(const char *key);
    /* ... exists, incr_by, expire, pttl, flush_all, wait ... */
    /* Must stay LAST-appended: the layout mirrors kv_bridge.rs. */
    int  (*del_if_eq)(const char *key, const char *token, size_t token_len);
} EphpmKvOps;

static EphpmKvOps g_kv_ops = {0};

void ephpm_set_kv_ops(const EphpmKvOps *ops)
{
    if (ops) {
        g_kv_ops = *ops;
    }
}
```

Rust mirrors it field for field as a `#[repr(C)]` struct of `Option<unsafe extern "C" fn(...)>` and fills in a
`static`. An `Option` of a function pointer has the same layout as a nullable C pointer:

```rust
#[repr(C)]
pub struct EphpmKvOps {
    pub get: Option<unsafe extern "C" fn(key: *const c_char) -> c_int>,
    pub get_result: Option<unsafe extern "C" fn(ptr: *mut *const c_char, len: *mut usize)>,
    // ...
    pub del_if_eq: Option<unsafe extern "C" fn(key: *const c_char, token: *const c_char, token_len: usize) -> c_int>,
}

pub static KV_OPS: EphpmKvOps = EphpmKvOps {
    get: Some(kv_get),
    get_result: Some(kv_get_result),
    // ...
    del_if_eq: Some(kv_del_if_eq),
};

// at startup, before any PHP thread runs a script
unsafe { ffi::ephpm_set_kv_ops(&kv_bridge::KV_OPS) };
```

The table is copied into the C global once, before any PHP thread runs. It's never written again. That's the entire
thread-safety story. A struct that's read-only after startup needs no lock, no atomics, nothing. Each `PHP_FUNCTION`
checks its pointer for NULL first. A build where the KV store isn't wired returns a miss instead of crashing.

C and Rust each declare the layout by hand, so the rule is append-only. New operations go at the end. The last field
carries a "must stay LAST-appended" comment on both sides. `wait` was added after `flush_all`, and `del_if_eq` after
`wait`. The DB table has the same comment on `available`. It's the same discipline as ePHPm's native middleware ABI.
It's boring on purpose.

### No malloc crosses the boundary

This is the rule I care about most. Memory allocated by Rust is freed by Rust. Memory allocated by PHP is freed by PHP.
Nothing crosses.

For a `get`, Rust copies the value into a thread-local buffer and returns 1. C then asks for a pointer and length into
that buffer and copies the bytes into a PHP string:

```c
PHP_FUNCTION(ephpm_kv_get)
{
    char *key; size_t key_len;
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_STRING(key, key_len)
    ZEND_PARSE_PARAMETERS_END();

    if (!g_kv_ops.get) { RETURN_NULL(); }
    if (!g_kv_ops.get(key)) { RETURN_NULL(); }

    const char *ptr; size_t len;
    g_kv_ops.get_result(&ptr, &len);
    RETURN_STRINGL(ptr, len);
}
```

The borrowed pointer is valid until the next `get` on the same thread. PHP is single-threaded per request, so there is
no next `get` until this one returns. That's one copy out of the store into the buffer and one copy into the zval. It's
why a 64 KB value costs a couple of microseconds and a 64-byte value doesn't.

The DB side stages more. A statement's rows go in one thread-local. `finish()` releases them as soon as C has turned them
into zvals. The metadata goes in a separate "last call" record that survives `finish()`. That covers whether it returned
rows, the column names and declared types, affected rows and last insert id. Any error goes in that record too. That
split is what made [#357](https://github.com/ephpm/ephpm/pull/357) possible. `ephpm_db_errno()` still works after the
exception was caught. `ephpm_db_columns()` can describe a `SELECT` that matched zero rows.

### Errors stay on the C side

Rust callbacks never call into Zend. That's deliberate. A PHP fatal unwinds with `longjmp`. A `longjmp` through Rust
frames skips their destructors, which is undefined behavior. So errors come back as return codes and C decides what to
do with them. For the DB functions that means throwing an exception, which doesn't `longjmp`:

```c
int rc = g_db_ops.run(sql, sql_len);
if (rc == -2) {
    zend_throw_exception(zend_ce_exception, EPHPM_DB_UNAVAILABLE_MSG,
                         EPHPM_DB_ERR_UNAVAILABLE);
    return EPHPM_DB_THREW;
}
if (rc < 0) {
    g_db_ops.error_info(&code, &sqlstate, &msg, &msg_len);
    zend_throw_exception_ex(zend_ce_exception, (zend_long)code,
        "SQLSTATE[%.5s]: %.*s", sqlstate ? sqlstate : "HY000",
        (int)msg_len, msg ? msg : "");
    g_db_ops.finish();
    return EPHPM_DB_THREW;
}
```

SQL errors carry the MySQL server error code litewire mapped them to. That's 1062 for a duplicate key and 1064 for a
syntax error. Infrastructure failures use 2000 through 2004:

- no database configured
- no site context
- can't connect
- unbindable parameter
- thread shutting down

Those sit in MySQL's client-error range, which a server never sends. An adapter can tell "your SQL is wrong" from "the
bridge isn't there" by number instead of by matching message text.

### Which keyspace?

ePHPm can serve many sites from one process, and each site gets its own KV store. The C side never sees that. Before PHP
runs, the router calls `set_site_store()` for the request. Every Rust callback resolves its store through a
thread-local:

```rust
fn effective_store() -> Option<Arc<Store>> {
    KV_SITE_STORE
        .try_with(|s| {
            let site = s.borrow();
            if let Some(ref store) = *site {
                Some(Arc::clone(store))
            } else {
                KV_STORE.get().cloned()
            }
        })
        .unwrap_or(None)
}
```

The DB bridge does the same with the site key. In per-site mode a query with no site context fails instead of falling
back to a shared database.

This had a bug that's worth knowing about if you build something similar. [#208](https://github.com/ephpm/ephpm/pull/208)
found two separate `MultiTenantStore` instances. The router built one for the PHP path and the RESP listener built the
other. Each lazily created its own per-hostname `Store`. So in vhost mode `ephpm_kv_set()` from PHP and a Predis `SET` for
the same site landed in different keyspaces. Neither could see the other's writes. Single-site mode was fine, which is
why it lived for a while. The fix is the obvious one: build it once and hand the same instance to both.

### Thread teardown

The nastiest bugs in the bridge weren't in calls. They were in thread exit.

The DB bridge keeps one litewire session per PHP thread, in a `thread_local!`. When tokio retired an idle blocking
thread, that thread-local's destructor set off a chain of drops:

- it dropped the session
- the session dropped the Turso connection
- the connection dropped pager buffers
- `Buffer::drop` recycles into Turso's own thread-local buffer cache

TLS destructors run in reverse registration order. Turso's cache was registered after ours, inside our first query. So it
was already gone. `LocalKey::with` on a destroyed thread-local panics. A panic in a TLS destructor aborts the process, and
every tenant on the box went down. [#300](https://github.com/ephpm/ephpm/issues/300) was found by an overload test where
3 of 10 flooded instances died this way. They died at different points: mid-flood, during the post-flood cooldown and at
SIGTERM.

There's no `try_with` I can add to fix that. The panicking `with` is several frames down inside Turso. What I can do is
not run that drop on a dying thread. The session is parked in a process-wide "graveyard" instead. The next request on
any live thread drains and drops it. In steady state the graveyard is empty and the drain is a single relaxed atomic
load.

[#269](https://github.com/ephpm/ephpm/issues/269) is the mirror image. Thread exit runs `php_request_shutdown()`. That
runs `register_shutdown_function()` callbacks and destructors. Those can call `ephpm_kv_*` or `ephpm_db_*` after the
bridge's own thread-locals are destroyed. Every access reachable from PHP now uses `try_with` and degrades. A KV call
reports a miss. A DB call returns `-3` and C throws a distinctly worded exception. And it fails closed. When the per-site
slot is gone, `effective_store()` returns `None`, not the global store. Falling back to the global store would let one
tenant's shutdown function write into another tenant's keyspace.

### The DB bridge

`ephpm_db_*` goes through the same litewire `Session` the MySQL wire frontend uses. The same MySQL-dialect translation,
`SHOW`/`DESCRIBE` emulation and transaction tracking apply. There's one session per PHP thread per site. A thread that
moves to a different site gets a new session.

`Session::query` is async and PHP callbacks are synchronous. So the bridge pins the server's tokio `Handle` and uses
`block_on`. That's only legal because these callbacks always run on PHP OS threads, never inside an async task. If one
ever did, tokio would panic. That's a much better failure than a deadlock.

Thread-scoped sessions have one obvious trap, and I fell into it. The session outlives the request. A script that ran
`BEGIN` and then fataled left the transaction open on that thread. So did one that just forgot to `COMMIT`. The next
unrelated request on the same thread silently joined it. Its writes could later be rolled back with someone else's.
[#258](https://github.com/ephpm/ephpm/pull/258) added `on_request_end()`. It runs at request teardown in both execution
modes and rolls back whatever is still open, with a warning in the log.

### A worked example: the session handler

The native session handler (`17493149`) shows the whole thing end to end. It's written in C against the ops table
directly, with no PHP in between. Sessions live at `session:<sid>`. The TTL comes from `session.gc_maxlifetime`. GC is a
no-op because the store expires keys itself.

Two details took real debugging. `session.lazy_write` is on by default. With it on, PHP calls `update_timestamp` instead
of `write` when the data didn't change. The handler refreshes the TTL with `expire` instead of rewriting the blob. And
`open` has to set `*mod_data` to something non-NULL:

```c
static int ephpm_session_mod_data_marker = 0;

PS_OPEN_FUNC(ephpm)
{
    (void)save_path;
    (void)session_name;
    *mod_data = (void *)&ephpm_session_mod_data_marker;
    return SUCCESS;
}
```

ext/session only calls a handler's `write` and `close` when `PS(mod_data)` is set. A native handler that keeps no state
and leaves it NULL never gets written. There's no error. You get a warning that it failed to write session data, and
nothing persisted. The marker is never dereferenced. It just has to exist.

### Known limitations

Keys cross the boundary as C strings. `PHP_FUNCTION` parses the key with `Z_PARAM_STRING`, which gives a pointer and a
length. But only the pointer goes through the ops table. Rust reads it with `CStr::from_ptr` and then requires valid
UTF-8:

```rust
let key_str = unsafe { CStr::from_ptr(key) };
let Ok(key_str) = key_str.to_str() else {
    return 0;
};
```

So a key with an embedded NUL byte (`"a\0b"`) addresses the key `"a"`. A key that isn't valid UTF-8 comes back as a
miss or a failed write with no error. Values are binary-safe, since they go through as pointer and length. Real
application keys are almost always ASCII. But if you hash something into a raw binary key, hex it first.

### How I measured

- Hardware: AMD Ryzen 9 5950X, WSL2 on Windows 11 (kernel 6.18.33.2-microsoft-standard-WSL2, 32 vCPU, 62 GiB).
- ePHPm: the published v0.10.6 release, `ephpm-v0.10.6+php8.5.7-linux-x86_64`. It ran as a bare process in
  `per_request` mode, with `[kv.redis_compat]` listening on `127.0.0.1` and no AUTH or compression.
- Redis: `redis:7-alpine` (7.4.11) under Podman with `--network host`, persistence off.
- Harness: the `kv/fixtures/kv_sapi.php` and `kv_resp.php` scripts from [ephpm/lab](https://github.com/ephpm/lab/tree/main/kv).
  The native lane is 200,000 ops per loop, timed with `hrtime()` inside a single request. It's a bare-loop number with
  no request dispatch in it. The socket lanes are 20,000 strict ping-pong round trips over one raw PHP stream socket
  with `tcp_nodelay`. The same fixture is pointed at each port. Client libraries like Predis add their own overhead on
  top.
- Each fixture checks the value it read back. A loop that got optimized into nothing would fail instead of looking
  fast. Three reps, lanes interleaved, median reported.

### What's next

The functions are only half of it. Nobody wants to rewrite their WordPress object cache or their Laravel cache store
around `ephpm_kv_set()`. [Part 2](/blog/2026/09/calling-rust-from-php-without-a-socket-part-2/) covers the wrapper
packages. They make existing phpredis, Predis, `WP_Object_Cache` and `wpdb` code use the native path unmodified. It
also covers the semantics bugs that showed up once real applications ran on them.

### Links

- [ephpm/ephpm](https://github.com/ephpm/ephpm), `crates/ephpm-php/ephpm_wrapper.c`, `kv_bridge.rs` and `db_bridge.rs`
- [#184 ephpm_kv_wait()](https://github.com/ephpm/ephpm/pull/184)
- [#208 the vhost keyspace split](https://github.com/ephpm/ephpm/pull/208)
- [#258 rollback at request end](https://github.com/ephpm/ephpm/pull/258)
- [#300 SIGABRT on thread retirement](https://github.com/ephpm/ephpm/issues/300) and [#269 try_with at teardown](https://github.com/ephpm/ephpm/issues/269)
- [#357 bridge API for adapters](https://github.com/ephpm/ephpm/pull/357)
- [ephpm/lab kv-micro harness](https://github.com/ephpm/lab/tree/main/kv)
