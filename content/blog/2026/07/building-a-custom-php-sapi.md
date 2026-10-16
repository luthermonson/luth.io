---
title: "Building a Custom SAPI - How PHP Actually Talks to the Outside World"
description: "php-fpm, mod_php and the PHP command line all plug into PHP the same way. Here's how ePHPm does it, and the shortcut that broke WordPress."
date: 2026-07-30T20:18:00-07:00
tags: [ ephpm, php, rust, sapi, php internals, wordpress ]
categories: [ ephpm, php, rust ]
images:
 - src: img/2026/08/building-a-custom-php-sapi-cover.svg
   alt: "Table mapping PHP operations like echo, header(), $_POST, $_COOKIE and $_SERVER to SAPI callbacks and to the Rust side of ePHPm"
   stretch: "none"
draft: false
---

If you've only ever run PHP behind nginx you've probably never had to think about how `echo` reaches a socket. When you
embed PHP, as [ePHPm](https://github.com/ephpm/ephpm) does, it's the first thing you have to answer, and the answer is a
SAPI. There is very little written about implementing one. Most of what I know came from reading `sapi/cli`,
`sapi/cgi` and `sapi/embed` in php-src and then finding out which parts I'd misread. This is the write-up I wanted back
in March.

### What a SAPI is

The Server API is PHP's boundary with whatever is hosting it. In C it's `sapi_module_struct`: a name plus a few dozen
function pointers that the engine calls whenever it needs something from the outside world. php-fpm is a SAPI, Apache's
mod_php is a SAPI, the CLI is a SAPI, and so are the built-in dev server and the embed SAPI. They all run the same
engine. What differs is what these callbacks do:

| Callback | PHP calls it when... |
|----------|---------------------|
| `ub_write` | anything is output: `echo`, `print`, templates, the error page |
| `flush` | output is flushed |
| `send_headers` | the first output byte is about to go out and headers need to be committed |
| `read_post` | it needs request body bytes (`$_POST`, `$_FILES`, `php://input`) |
| `read_cookies` | it wants the raw `Cookie` header to build `$_COOKIE` |
| `register_server_variables` | it's building `$_SERVER` |
| `log_message` | `error_log()` or the engine logs something |

ePHPm starts from the embed SAPI (`php_embed_init()` gives you a working engine with stdout-oriented defaults) and then
overwrites the callbacks with its own:

```c
void ephpm_install_sapi(void)
{
    sapi_module.ub_write = ephpm_sapi_ub_write;
    sapi_module.flush = ephpm_sapi_flush;
    sapi_module.send_headers = ephpm_sapi_send_headers;
    sapi_module.read_post = ephpm_sapi_read_post;
    sapi_module.read_cookies = ephpm_sapi_read_cookies;
    sapi_module.register_server_variables = ephpm_sapi_register_server_variables;
    sapi_module.log_message = ephpm_sapi_log_message;

    sapi_module.name = "ephpm";
    sapi_module.pretty_name = "ePHPm Embedded Server";
}
```

`sapi_module.name` is what `php_sapi_name()` returns, so frameworks that branch on `cli` vs everything else see `ephpm`
and treat it as a web SAPI. All of this is C, not Rust, for the reason I covered in
[the Rust post](/blog/2026/04/why-rust-for-php-embedding/): PHP can `longjmp` out from under any of these callbacks, so
they never call into Rust while the engine is live. They read and write thread-local C buffers, and Rust copies things
in before the script runs and out after it finishes.

### Output: ub_write

Output is the simplest one. PHP hands over a pointer and a length and expects to be told how many bytes were taken.
ePHPm appends to a per-thread buffer that doubles as needed:

```c
static size_t ephpm_sapi_ub_write(const char *str, size_t str_length)
{
    if (output_len + str_length > output_cap) {
        size_t new_cap = (output_cap == 0) ? 8192 : output_cap;
        while (new_cap < output_len + str_length)
            new_cap *= 2;
        char *new_buf = realloc(output_buf, new_cap);
        if (!new_buf) return 0;
        output_buf = new_buf;
        output_cap = new_cap;
    }
    memcpy(output_buf + output_len, str, str_length);
    output_len += str_length;
    return str_length;
}
```

The buffer is `__thread` (`__declspec(thread)` on MSVC, behind a macro), because under ZTS several threads run PHP at
once. It's never freed between requests, only reset to length zero, so a thread that served one big page keeps that
capacity around. That's a deliberate trade: no allocation on the hot path in exchange for some memory per thread.

Returning `0` from `ub_write` tells PHP the client went away. If `realloc` fails, that's the honest answer.

### Headers: send_headers is almost a no-op

This one surprised me. `send_headers` is called once, right before the first byte of output, so a SAPI like php-fpm
can write the status line and headers to the FastCGI stream. ePHPm doesn't stream PHP output in this mode, it buffers
the whole response, so its `send_headers` just says "done":

```c
static int ephpm_sapi_send_headers(sapi_headers_struct *sapi_headers)
{
    (void)sapi_headers;
    return SAPI_HEADER_SENT_SUCCESSFULLY;
}
```

The real header capture happens after the script finishes, by walking `SG(sapi_headers).headers` (the `zend_llist`
that `header()` appends to) while the request is still alive, and reading `SG(sapi_headers).http_response_code` for the
status. One subtlety: if a script never outputs anything, the default `Content-Type` never makes it into that list,
because PHP adds it as part of sending headers. So the capture code synthesizes it from `SG(sapi_headers).mimetype`, or
`default_mimetype` + `default_charset`, when no `Content-Type` was set.

### Input: read_post and read_cookies

`read_post` is a cursor over the request body Rust handed in:

```c
static size_t ephpm_sapi_read_post(char *buffer, size_t count_bytes)
{
    if (!req_post_data || req_post_data_offset >= req_post_data_len)
        return 0;

    size_t remaining = req_post_data_len - req_post_data_offset;
    size_t to_copy = remaining < count_bytes ? remaining : count_bytes;
    memcpy(buffer, req_post_data + req_post_data_offset, to_copy);
    req_post_data_offset += to_copy;
    return to_copy;
}
```

and `read_cookies` returns the raw `Cookie` header and lets PHP parse it.

Here's one of the things you only learn by reading `main/SAPI.c`: `sapi_activate()` only reads and parses the POST body
into `$_POST` if `SG(server_context)` is non-NULL. That pointer is how the CGI and built-in server SAPIs say "this is a
real request" as opposed to a CLI run. ePHPm's callbacks don't use it for anything, so it gets a stable non-NULL
sentinel, and without that line `$_POST` was quietly empty while `php://input` still worked. Another: PHP 8.4 exports
`sapi_read_post_data()`, but in 8.3 the same function is `static` inside `main/SAPI.c`, so there's nothing to call.
ePHPm supports 8.3, so for `PHP_VERSION_ID < 80400` the wrapper carries a verbatim copy of 8.4's function.

### $_SERVER from a hyper request

`register_server_variables` walks a list of key/value pairs that Rust built from the hyper request. Here's the original
version of that builder (it's since been reworked to skip the per-header allocations and map names in a single ASCII
pass, but the mapping is the same):

```rust
let mut vars = vec![
    ("REQUEST_METHOD".into(), self.method.clone()),
    ("REQUEST_URI".into(), self.uri.clone()),
    ("SCRIPT_FILENAME".into(), self.script_filename.to_string_lossy().into_owned()),
    ("SCRIPT_NAME".into(), script_name.clone()),
    ("DOCUMENT_ROOT".into(), self.document_root.to_string_lossy().into_owned()),
    ("SERVER_NAME".into(), self.server_name.clone()),
    ("SERVER_PORT".into(), self.server_port.to_string()),
    ("SERVER_PROTOCOL".into(), self.protocol.clone()),
    ("GATEWAY_INTERFACE".into(), "CGI/1.1".into()),
    ("QUERY_STRING".into(), self.query_string.clone()),
    ("PHP_SELF".into(), script_name),
    ("REMOTE_ADDR".into(), self.remote_addr.ip().to_string()),
    ("REMOTE_PORT".into(), self.remote_addr.port().to_string()),
    ("REDIRECT_STATUS".into(), "200".into()),
];

// Map HTTP headers to $_SERVER variables
for (name, value) in &self.headers {
    let key = match name.to_lowercase().as_str() {
        "host" => "HTTP_HOST".to_string(),
        "cookie" => "HTTP_COOKIE".to_string(),
        "content-type" => "CONTENT_TYPE".to_string(),
        "content-length" => "CONTENT_LENGTH".to_string(),
        // "Accept-Encoding" -> "HTTP_ACCEPT_ENCODING"
        _ => format!("HTTP_{}", name.to_uppercase().replace('-', "_")),
    };
    vars.push((key, value.clone()));
}
```

That's the CGI/1.1 variable set every PHP framework expects. `SCRIPT_NAME` is derived from the resolved script path
relative to the document root, not from the URI, so it stays right after a front-controller rewrite
(`/blog/hello` → `/index.php`). `HTTPS=on` is added for TLS requests, and `REDIRECT_STATUS=200` keeps code that was
written for `cgi.force_redirect` happy.

Two sharp edges in that header loop, both worth knowing about if you write your own:

- **httpoxy.** A client-sent `Proxy:` header becomes `HTTP_PROXY`, which a lot of HTTP client libraries read as "use
  this proxy for outbound requests." That's the 2016 [httpoxy](https://httpoxy.org/) bug class, and the generic
  `HTTP_*` mapping walks right into it. As of last week ePHPm strips `Proxy` (and any configured JWT claims header) at
  request ingest, before middleware runs and before anything crosses into PHP
  ([#200](https://github.com/ephpm/ephpm/pull/200)).
- **A fixed-size table.** On the C side the server variables live in a `__thread` array of 128 pairs, and
  `ephpm_request_add_server_var()` silently drops anything past that. Sixteen-odd CGI variables plus a request with more
  than about a hundred headers will lose the tail. Nobody has hit it, but "silently" is the word I don't like there.

### The shortcut that broke WordPress

The interesting part of a SAPI isn't the callbacks, it's the lifecycle around them. A normal SAPI does this for every
request:

```
php_request_startup()   -> superglobals built, executor fresh
php_execute_script()
php_request_shutdown()  -> user functions, classes, constants, globals, statics destroyed
```

In March, `php_request_startup()` crashed on tokio's `spawn_blocking` threads (the signal and stack-probe functions it
calls were the problem, [same family as the SIGPROF bug](/blog/2026/04/php-signal-safety-init-ordering/)). So the first
version of ePHPm took a shortcut: start one request when the thread is registered with TSRM, and *reuse it forever*.
Between HTTP requests it only reset the SAPI request info and rebuilt the superglobals by hand: destroy the
`PG(http_globals)` arrays, rebuild `$_SERVER` through the callback, run `treat_data` over the query string for `$_GET`,
call `sapi_handle_post()` for `$_POST`/`$_FILES`, parse cookies manually, merge `$_REQUEST`.

That passed every test I had, because every test was a script that didn't care what the previous request left behind.
WordPress cares. [Bryan Spears](https://github.com/bryanspears) tracked it down at the end of June
([`9715d61`](https://github.com/ephpm/ephpm/commit/9715d61f893a910b1b0655bb211113a15035d937)): user functions, classes,
constants, function statics and the global symbol table all survived between requests on the same thread. WordPress's
`wp-blog-header.php` checks `$wp_did_header` (a global) and defines `WP_USE_THEMES` (a constant), so on the second request
to a warmed thread it decided it had already bootstrapped and skipped everything. Request #1 on a thread was a full 68KB
page. Requests #2 and later were 119-byte blank pages containing a "constant already defined" warning.

By then the functions that made `php_request_startup()` crash were already `--wrap`'d to no-ops, so the fix was to do
what every other SAPI does:

```c
int ephpm_execute_request(const char *filename)
{
    /* Tear down the previous request and start a fresh one. */
    if (request_active) {
        php_request_shutdown(NULL);
        request_active = 0;
    }

    output_len = 0;
    headers_buf_len = 0;
    req_post_data_offset = 0;

    /* Populate SG(request_info) BEFORE php_request_startup(): PHP builds the
     * superglobals during startup using our callbacks, which read these. */
    SG(request_info).request_method = (char *)req_method;
    SG(request_info).request_uri = (char *)req_uri;
    SG(request_info).query_string = (char *)req_query_string;
    SG(request_info).content_type = req_content_type;
    SG(request_info).cookie_data = (char *)req_cookie_data;
    SG(request_info).content_length = (long)req_post_data_len;
    SG(request_info).path_translated = (char *)req_path_translated;
    SG(server_context) = &ephpm_server_context_marker;

    if (php_request_startup() != SUCCESS) {
        return -1;
    }
    request_active = 1;

    /* startup doesn't reset these on the embed path, so a 201 or 500 from the
     * previous request on this thread would leak into this one */
    SG(sapi_headers).http_response_code = 200;
    SG(headers_sent) = 0;
    SG(request_info).no_headers = 0;

    /* ... replay per-request INI (open_basedir etc.), then execute under SETJMP ... */
}
```

The ordering in there is the lesson. `SG(request_info)` has to be filled in *before* `php_request_startup()`, because
startup is what builds the superglobals, through the callbacks, from those fields. Once startup owned that job, the old
hand-rebuild became actively harmful: it destroyed the arrays startup had just created and re-ran `treat_data` over
them, which was a use-after-free inside `php_default_treat_data` under load. The correct amount of superglobal code in
a SAPI turned out to be none. Set the inputs, let PHP build them.

The cost: about 5% on a trivial no-op script, per the commit's own measurement, and nothing noticeable on a real page,
because OPcache's compiled bytecode and the JIT buffer live in shared memory and survive request shutdown. It's the
classic php-fpm + OPcache model, just with threads instead of processes. WordPress went from one good page per warmed
thread to 40 out of 40 under concurrency.

The status code reset in that block is the same kind of bug in miniature: `http_response_code(201)` on one request
came back as the status of the next request on that thread, because the embed path doesn't reset it during startup.

### What I'd tell someone writing a SAPI

- Read `sapi/cgi/cgi_main.c` and `sapi/cli/php_cli_server.c` before you read the embed SAPI. Embed is the smallest, but
  it's built for "run one script and exit," and the gaps are exactly the parts a server needs.
- Never skip `php_request_shutdown()`. It is the isolation boundary between requests. If something crashes during
  request startup, fix that instead of reusing a request.
- Set `SG(request_info)` before startup and let PHP build the superglobals.
- Treat every global in `SG()`/`PG()` as suspect between requests and check what startup does and doesn't reset.
- Test with WordPress. It relies on more global state than anything else you're likely to run, and it will find your
  lifecycle bugs immediately.

### Links

- [ePHPm](https://github.com/ephpm/ephpm), the SAPI lives in `crates/ephpm-php/ephpm_wrapper.c`
- [`9715d61`](https://github.com/ephpm/ephpm/commit/9715d61f893a910b1b0655bb211113a15035d937) per-request shutdown/startup
- [#200](https://github.com/ephpm/ephpm/pull/200) strip `Proxy` and claims headers at ingest
- [php-src `main/SAPI.h`](https://github.com/php/php-src/blob/master/main/SAPI.h) for `sapi_module_struct`
- [php-src `sapi/`](https://github.com/php/php-src/tree/master/sapi) for every SAPI that ships with PHP
- [httpoxy.org](https://httpoxy.org/)
