---
title: "Middleware in PHP: Compiling It With elephc, Then Not Needing To"
description: "Fitting PHP into ePHPm's native middleware ABI with an ahead-of-time compiler, and what that experiment said about where PHP middleware should actually run."
date: 2026-08-31T20:50:00-07:00
tags: [ php, rust, ephpm, elephc, middleware, ffi ]
categories: [ ephpm, php, rust ]
images:
 - src: img/2026/08/php-middleware-elephc-cover.svg
   alt: "Timeline of an ePHPm request showing the native middleware lane before the body read and php: mounts inside the PHP request"
   stretch: "none"
draft: false
---

[ePHPm](https://github.com/ephpm/ephpm) middleware runs *before* PHP. It's a small versioned C ABI, loaded with
`dlopen`, that gets to look at a request and reject it before PHP boots or the request body is read. That's the
right shape for a gateway. It's the wrong shape for most PHP shops, who want to write their middleware in PHP. This is
the story of compiling PHP into a native `.so` to fit that ABI and finding exactly where it broke. Then I built the lane
that should have existed in the first place.

### The native middleware ABI

A module is a shared library (`.so`, `.dylib` or `.dll`) that exports four symbols. A fifth, optional response-phase
hook arrived in v0.8.0:

```c
int32_t     ephpm_middleware_init(uint32_t abi_version,
                                  const char *config_json,
                                  const ephpm_host_v1 *host);
int32_t     ephpm_middleware_invoke(const ephpm_request_t *req,
                                    ephpm_response_t *out);
void        ephpm_middleware_shutdown(void);
const char *ephpm_middleware_describe(void);   /* optional */
```

The host hands the module a callback table at `init`. It holds request accessors, the `tracing` logger and the
cluster-replicated KV store. The module never `dlsym`s host symbols. That would need `-rdynamic` on Linux and has no
clean equivalent on Windows. The top byte of `abi_version` is the major and gates compatibility. The lower bytes are a
minor that only ever grows the table by appending. A module built against a shorter table stays correct on a newer
host. It just can't see the slots it didn't declare.

`invoke` returns one of three verdicts: CONTINUE, RESPOND or REWRITE. RESPOND short-circuits, so PHP never runs. The
contract is plain C, so ePHPm doesn't care what language produced the `.so`. That's the opening an ahead-of-time PHP
compiler walks through.

### The elephc experiment

[elephc](https://github.com/illegalstudio/elephc) is a real, MIT-licensed, ahead-of-time PHP-to-native compiler. It
parses PHP, type-checks it, lowers it through its own SSA IR and emits assembly. There's no Zend Engine involved.
`--emit cdylib` turns functions marked `#[Export]` into C exports. So the question was simple. Can I write ePHPm
middleware in PHP, compile it and load it into a stock ePHPm release?

The answer, verified on 2026-08-20 against elephc 0.26.4 and the published ePHPm v0.7.3 Linux binary, was yes. This
PHP ran as native code before PHP dispatch:

```php
<?php
extern function mw_set_header(string $name, string $value): void;
extern function mw_log(string $msg): void;

#[Export]
function mw_invoke(string $method, string $path): int {
    mw_log("elephc-mw: " . $method . " " . $path);

    mw_set_header("X-Elephc", "1");
    mw_set_header("X-Route", classify($path));

    return 0; // ACTION_CONTINUE
}

function classify(string $path): string {
    if (strlen($path) >= 5 && substr($path, 0, 5) === "/api/") {
        return "api";
    }
    return "web";
}
```

And the headers showed up on live responses. The log lines landed in ePHPm's own tracing output:

```
$ curl -sS -D- -o /dev/null http://127.0.0.1:8299/api/users.php
HTTP/1.1 200 OK
x-elephc: 1
x-route: api

INFO ephpm_server::middleware: middleware initialised
     module=/root/mwx/libelephc_mw.so
     describe=elephc-php-middleware (experimental adapter)
INFO ephpm_middleware: elephc-mw: GET /api/users.php
```

### The catch: a mandatory C shim

elephc's cdylib marshaller only handles `int`, `float`, `bool` and `string` in exported signatures. ePHPm's ABI is
built on struct pointers. Trying to export the entry points straight from PHP fails at compile time:

```
$ elephc --emit cdylib abi_probe.php
error[abi_probe.php:11:1]: exported function 'ephpm_middleware_init'
  parameter #3 has unsupported type for --emit cdylib v1;
  supported: int, float, bool, string
```

So there's a ~180 line C shim that's not optional. It does five jobs:

- exports the four ABI symbols
- dereferences the host table and request struct
- flattens the request into strings
- calls the PHP `mw_invoke`
- provides `mw_set_header` and `mw_log`

The PHP side's `extern` declarations are left as undefined symbols in the elephc output. They get resolved against the
shim at `dlopen`:

```
ePHPm  --dlopen-->  libelephc_mw.so (C shim, exports ephpm_middleware_*)
                          |
                          +--NEEDED--> libmw_php.so (elephc output, exports mw_invoke)

$ nm -D libmw_php.so | grep -E ' T | U '
T mw_invoke
U mw_log            <-- resolved from the shim at dlopen
U mw_set_header
```

A couple of small gotchas came up along the way. The shim needs `-Wl,-rpath,'$ORIGIN'` or ePHPm fails at startup with
`cannot open shared object file`. And elephc exports can't return strings, so headers go out through callbacks instead
of return values.

### Do your Composer libraries work? No.

This was the claim most worth being careful about, so I tested it instead of assuming it. I installed three real
packages with Composer and fed them to the compiler:

| Package | Result | Root cause |
|---|---|---|
| Composer's own `vendor/autoload.php` | fails | assignment inside a `while` condition, typed static properties |
| `nikic/fast-route` | fails | `new $options['routeCollector']` (dynamic class name after `new`) |
| `firebase/php-jwt` | fails | `(object)` cast, `#[\SensitiveParameter]` parameter attribute |

The first row is the important one. Composer's generated class loader doesn't compile. So `require
'vendor/autoload.php'` is a hard stop for basically every modern PHP project. The failing files are kept in the repo
as regression tests for when elephc's coverage improves. elephc's own README is candid about this: "not a drop-in
replacement for an entire dynamic framework today". Its stdlib coverage is broad. The gap is parser and feature
coverage on real-world library code. What works today is first-party PHP: functions, string and array builtins, control
flow, regex. That's a useful subset for middleware. But it's *your* code, not your dependency tree.

More hard limits came out of the run:

- elephc's cdylib runtime has no locking around its heap, and ePHPm invokes middleware concurrently. So the shim wraps
  every call in a global mutex, and every middleware call runs one at a time.
- No Windows target, even though ePHPm's middleware lane loads `.dll`s fine.
- A PHP fatal inside the cdylib terminates the process.
- The headline RESPOND path, rejecting a request before PHP boots, was never exercised. Only CONTINUE was.

### The contrast case: the same ABI from Rust

After publishing the elephc repo it bothered me that our only worked example of the ABI needed a hand-written C
shim. That makes the ABI look harder than it is. So I added an in-tree Rust example
([#375](https://github.com/ephpm/ephpm/pull/375)) called `api-gate`. It's an API key check with a fixed-window rate
limit on `kv_incr_ttl` and runtime revocation through KV. It's ~200 lines of ordinary safe Rust with zero `unsafe` in
the module itself. The `declare!` macro supplies the rest:

- the four exports
- the version handshake
- config parsing
- panic containment
- response marshaling

```rust
fn invoke(&self, req: &Request<'_>) -> Response {
    if !req.path().starts_with(&self.prefix) {
        return Response::cont().response_header("X-Api-Gate", "bypass");
    }
    let host = req.host();

    let Some(tenant) = req.header("X-Api-Key").and_then(|key| self.keys.get(key)) else {
        return Self::reject(401, "missing or unknown X-Api-Key")
            .header("WWW-Authenticate", "ApiKey realm=\"api-gate\"");
    };
    // ... revocation via host.kv_get(), rate limit via host.kv_incr_ttl() ...
}

ephpm_middleware::declare!(ApiGate);
```

It exercises RESPOND three ways (401, 403, 429). The missing `x-powered-by` header on those responses proves PHP never
ran:

```
HTTP/1.1 401 Unauthorized
www-authenticate: ApiKey realm="api-gate"
{"error":"missing or unknown X-Api-Key"}
```

The shim was elephc's limitation, not the ABI's. Don't let your first example make your ABI look harder than it is.

### What PHP developers actually wanted: the `php:` lane

Here's the thing the elephc run made obvious. Nobody asking for "middleware in PHP" was asking for PHP compiled to
native code. They wanted to write a PHP file, point config at it and have it run before their app. So that's what
[#382](https://github.com/ephpm/ephpm/pull/382) built:

```toml
[[middleware]]
library = "php:middleware.php"   # relative to the request's document root
match   = "/api/*"
order   = 30
config  = { realm = "admin" }
```

```php
<?php
$cfg  = json_decode(ephpm_middleware_config() ?? '{}', true);
$auth = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
if (!str_starts_with($auth, 'Bearer ')) {
    http_response_code(401);
    header('WWW-Authenticate: Bearer realm="' . $cfg['realm'] . '"');
    echo 'missing bearer token';
    exit;                                     // RESPOND, the app never runs
}
$_SERVER['HTTP_X_TOKEN'] = substr($auth, 7);  // REWRITE
header('X-Auth-Checked: 1');                  // response header
// falling off the end = CONTINUE
```

![Two middleware lanes in ePHPm](/img/2026/08/php-middleware-elephc-cover.svg)

A `php:` mount runs in the same PHP request as the application, immediately before it. That's where
`auto_prepend_file` runs, except a mount is scoped by path pattern, supports multiple files and carries config. That
position is the whole design. The middleware inherits the superglobals, `open_basedir`, the OPcache vhost, the per-site
DB session, the KV keyspace, the execution timer and the crash guard *because it's the same request*. There's no second
PHP context to configure, so there's none to get wrong.

### Three things only running it revealed

**You can't use `php_execute_script()`.** PHP 8 implements `exit()` as an unwind-exit exception. Both
`php_execute_script()` and `zend_execute_scripts()` hand every pending exception to `zend_exception_error()`. That
clears `EG(exception)` and returns FAILURE. It does that whether the script called `exit()`, threw or failed to parse.
On return there's nothing left to tell "called `exit()`" apart from "blew up". The first cut didn't look at the return
value at all. So the short-circuit tests failed with the app's output appended after the middleware's 401. The chain now
runs PHP's own loop body inline with that one case split out.

**An uncaught `Throwable` fails open.** `zend_exception_error` reports it with `E_DONT_BAIL`, so execution returns
normally. Inside the inline loop, `PG(last_error_type)` is what's left to record it. Without an explicit check the app
runs anyway. For an auth middleware that's an auth bypass. It's checked after every mount now. Every failure mode is a
500 where the application script does not run. That covers a missing file, a parse error, an uncaught exception and a
fatal.

**The `chdir` had to go.** Matching `auto_prepend_file` exactly meant a getcwd, chdir and restore around the chain.
That measured ~55 µs per request on Windows, about 5x everything else the lane does. I removed it and the docs say to
use `__DIR__`.

### What it costs

These numbers are from the PR's benchmark. It used a release build with OPcache on. Each run was 1000 sequential
keep-alive requests to `<?php echo 'ok';`, best of 11. Each middleware set three response headers. The baseline with no
middleware was 365.7 µs per request.

![Added latency per middleware lane](/img/2026/08/php-middleware-elephc-cost.svg)

In isolation the Rust builtin's marginal cost is 476 ns. So a `php:` mount is ~25x a Rust builtin's isolated cost. Over
HTTP the gap shrinks to ~3x, as the chart shows. It's also ~3% of a trivial PHP request. Both framings are true, so pick
the honest one for your audience. An empty middleware file costs the same as a working one. The cost is engaging the
lane, not your PHP.

### The tradeoff to own

Native modules run before the body is read. The exception is if you opt into buffering a bounded body for them with
`middleware_body_limit`. `php:` mounts run after. A PHP middleware can't reject an upload before it's transferred, and
no `order` value changes that. So startup logs the resolved plan and it's never a surprise.

For multi-tenancy, mount paths resolve against the *request's* document root. Each tenant brings their own file, and it
has no more reach than their `index.php`. Absolute paths are rejected at startup in every mode. Sharing one operator
file across tenants would need a hole in every vhost's `open_basedir`. Worker mode is a startup error rather than a
warning. There's no per-request point to run a mount, so it would silently never run.

### Takeaway

The elephc run answered the feasibility question. A stale tracking note in the ePHPm repo still listed "native
middleware via elephc cdylib" as planned, spec only, no code. It's now verified working with known limits. It also
showed what the real request was. So both lanes stay: native for "reject before the body", PHP for everything else.

The other thing I'd repeat is how that README was written. Every claim is marked **Verified** or **Not verified**.
That's what made the follow-up decision easy. It's very tempting to write up a prototype as if everything you designed
also ran. It's much more useful to write down exactly which parts did.

### Links

- [ephpm/elephc-middleware-example](https://github.com/ephpm/elephc-middleware-example)
- [elephc](https://github.com/illegalstudio/elephc) and its [cdylib docs](https://elephc.dev/docs/beyond-php/cdylib/)
- Rust `api-gate` example: [ephpm #375](https://github.com/ephpm/ephpm/pull/375)
- `php:` middleware lane: [ephpm #382](https://github.com/ephpm/ephpm/pull/382)
- Guide: `site/content/guides/native-middleware.md` in the [ePHPm repo](https://github.com/ephpm/ephpm)
