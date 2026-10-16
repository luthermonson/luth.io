---
title: "Step Debugging a Worker That Never Ends"
description: "A persistent PHP worker never finishes its request, so Xdebug never sees your trigger. Here's how ePHPm routes the debugged request to its own fresh worker instead of trying to reset anything."
date: 2026-10-12T21:10:00-07:00
tags: [ php, xdebug, rust, ephpm, debugging ]
categories: [ ephpm, php ]
images:
 - src: img/2026/10/xdebug-worker-mode-cover.svg
   alt: "Diagram of the ePHPm router sending untriggered requests to the warm worker pool and Xdebug-triggered requests to a one-request debug lane"
   stretch: "none"
draft: false
---

Persistent workers are the big performance win in modern PHP. You boot Laravel or Symfony once, keep the container
in memory, and serve thousands of requests out of the same process. [ePHPm](https://github.com/ephpm/ephpm) has had
a worker mode for a while. Like most worker runtimes it had a quiet hole in it. You couldn't step debug a request with
Xdebug. I wanted that fixed without patching Xdebug and without giving up the warm pool. The answer ended up being to
stop fighting the constraint and route around it. The debug lane is merged on main and coming in the upcoming v0.12.0
release.

### Why worker mode breaks Xdebug

To PHP, a worker's entire life is one request. Request startup (RINIT) runs when the worker boots. Then the script sits
in a loop pulling HTTP requests off ePHPm. Request shutdown (RSHUTDOWN) only runs when the worker retires. A minimal
worker looks like this:

```php
<?php
// boot once: build the kernel, container, config
$served = 0;

while (($envelope = \Ephpm\Worker\take_request()) !== null) {
    $served++;
    $vars = $envelope->serverVars();

    \Ephpm\Worker\send_response(
        200,
        ['Content-Type' => 'text/plain'],
        "hello {$vars['REQUEST_URI']} (request #$served)\n",
    );
}
```

Xdebug decides whether to open a DBGp session at RINIT. It looks for `XDEBUG_TRIGGER` or `XDEBUG_SESSION` in
`$_GET`, `$_POST` and `$_COOKIE`. If it finds one it connects to your IDE, and it closes the session at RSHUTDOWN. On a
warm worker none of that lines up. The "request" Xdebug saw at RINIT was the boot, which had no HTTP data at all. So it
never sees your trigger. And if it did open a session, that session would last for the entire life of the worker.

The first idea everybody has is "reset Xdebug between loop iterations". That's a dead end. While the worker is parked in
`take_request()` its own PHP frames are still live on the stack. There's no clean point where you can tear down and
restart Xdebug's per-request state without pulling the floor out from under the script.

### Route, don't reset

The design that worked flips the problem. Instead of making a warm worker behave like a fresh request, give the
debugged request an actual fresh request.

![Worker mode debug lane](/img/2026/10/xdebug-worker-mode-cover.svg)

1. The router decides *before any PHP runs* whether Xdebug would start a session for this request.
2. If it would, the request skips the warm pool and goes to a fresh, one-request worker thread. That thread's
   `SG(request_info)` is filled in from the triggering HTTP request before `php_request_startup()`. Xdebug's own
   RINIT sees the trigger natively and connects to the IDE before the first line of PHP runs.
3. The worker script boots *inside* the debug session, so breakpoints in container setup and service providers
   actually hit. That's something you don't usually get without some fiddling even under php-fpm.
4. The thread serves exactly that one request through the normal `take_request()` / `send_response()` loop with
   `max_requests = 1`. You step through the same code path production uses, not a debug-only copy.
5. The loop ends and `php_request_shutdown()` runs. Xdebug sends a clean DBGp `stopping`, same as php-fpm.

The C side of step 2 is short. It registers the thread with TSRM and points PHP's request globals at the triggering
request. Only then does it start the PHP request:

```c
int ephpm_thread_init_preseeded(const char *method, const char *uri,
    const char *query_string, const char *content_type, const char *cookie,
    size_t content_length, const char *path_translated,
    const char *post_data, size_t post_data_len)
{
    ts_resource(0);
    /* ...thread-local mirrors for the read_cookies / read_post callbacks... */

    SG(request_info).request_method = (char *)method;
    SG(request_info).request_uri = (char *)uri;
    SG(request_info).query_string = (char *)query_string;
    SG(request_info).content_type = content_type;
    SG(request_info).cookie_data = (char *)cookie;
    SG(request_info).content_length = (zend_long)content_length;
    SG(request_info).path_translated = (char *)path_translated;

    /* Only for a buffered urlencoded POST: let sapi_activate() parse $_POST. */
    SG(server_context) = post_data ? (void *)&ephpm_server_context_marker : NULL;

    int ret = php_request_startup();

    /* Back to the pool-worker state, success or not. */
    SG(server_context) = NULL;
    /* ... */
    request_active = 1;
    return ret == SUCCESS ? 0 : -1;
}
```

For a form POST trigger, PHP gets a private copy of the body during startup only. The app still receives the whole
body when it calls `take_request()`. The end-to-end test checks the md5 to prove it.

Every other request stays on the warm pool. In the PR's end-to-end run I held a debug session paused on a breakpoint
for 6 seconds and kept hitting the server. The warm pool answered in 0.58 to 0.67 ms per request the whole time. It
served them from the same two boot ids it had before the session started.

The honest cost is one framework boot per debugged request. That's what php-fpm pays on every request anyway. Paying it
also means some bugs won't reproduce on a debugged request. A bug that only shows up on a warm worker's Nth request (a
leaked static, a stale cache) is one of them. The lane can't help with that class of bug and the docs say so.

### Reimplementing Xdebug's trigger logic in Rust, exactly

The routing decision has to match Xdebug's own decision. If the router says yes and Xdebug says no, you wasted a
framework boot. If the router says no and Xdebug would have said yes, the request silently doesn't debug. That's the
worst kind of developer experience. So `xdebug_trigger.rs` is a port of three Xdebug 3.5.3 functions:

- `trigger_enabled()`
- `xdebug_lib_find_in_globals()`
- `xdebug_debug_init_if_requested_at_startup()`

It's checked against php-src 8.5.7's `php_variables.c` for the decoding rules.

The corner cases are the fun part:

- `XDEBUG_TRIGGER` is looked up first, then the legacy `XDEBUG_SESSION`. Each is checked in `$_GET`, then `$_POST`,
  then `$_COOKIE`. Xdebug also falls back to the environment. That can't vary per request, so the router ignores it.
  The *first* place a name shows up decides.
- `XDEBUG_IGNORE` vetoes everything (unless its value is `no` or `0`).
- `xdebug.trigger_value` can be set to a shared secret, optionally a comma-separated list. Then the trimmed trigger
  value has to match, and the legacy `XDEBUG_SESSION_START` is refused outright.
- Under `start_with_request=yes`, any trigger *name* counts and its value is ignored.

Here's the core of it:

```rust
pub fn routes(&self, input: &TriggerInput<'_>) -> bool {
    // Cheap pre-filter: a request mentioning none of the raw bytes
    // (and no % escape that could spell them) cannot match.
    if !might_mention(input.query_string.as_bytes())
        && !might_mention(input.cookie_data.as_bytes())
        && !input.buffered_body.is_some_and(might_mention)
    {
        return false;
    }

    let lookup = Lookup::build(input);

    if let Some(value) = lookup.first_found(XDEBUG_IGNORE)
        && !matches!(value, Value::Str(v) if v == b"no" || v == b"0")
    {
        return false;
    }

    match self.start_with_request {
        StartWithRequest::No => false,
        StartWithRequest::Yes => {
            lookup.first_found(XDEBUG_TRIGGER).is_some()
                || lookup.first_found(XDEBUG_SESSION).is_some()
                || lookup.session_start
        }
        StartWithRequest::Default | StartWithRequest::Trigger => {
            self.trigger_enabled(&lookup) || (self.secrets.is_none() && lookup.session_start)
        }
    }
}
```

The part that took the longest wasn't the decision. It was building `Lookup` so it decodes names and values the way PHP
does. `+` is a space in query strings and form bodies but not in cookies. Duplicate GET and POST keys are last-wins.
Duplicate cookies are first-wins. PHP mangles variable names (`.` and space become `_`, `[...]` makes an array). If any
of that is off, an encoding trick could route a request Xdebug would refuse, or the reverse. Even Xdebug's mode parser
got ported. It matches each item as a prefix (`xdebug.mode=d` means `develop`, not `debug`).

Ini values are per-thread under ZTS. So `xdebug.mode`, `start_with_request` and `trigger_value` are read once on the
`php_embed_init()` thread and frozen into a `TriggerPolicy`. The router never touches PHP state.

### A debugger is an attack surface

The lane is reachable by any client that can reach the server. Anyone who can send a trigger can force a fresh
framework boot. With `xdebug.discover_client_host` on, they can point the debugger at a host they control. A debugger
there can run arbitrary code. Xdebug is a dev tool and should never be loaded in production. But "should never" isn't
a control, so the lane is bounded:

- `[php.worker] debug_workers` (default 1) is separate from `[php] concurrency`. A developer paused on a breakpoint
  never takes a serving slot.
- Over the cap, triggered requests *wait* instead of getting a 503. With a browser helper extension every AJAX call
  from your tab carries the trigger cookie. Failing them would break the page you're debugging.
- The wait is bounded though: 16 waiters per slot, then an immediate 503.
- A thread paused inside Xdebug can't be killed. If the request times out or the client goes away, the slot is freed
  right away. It's held by the HTTP request, not the thread. But the thread lives on until you resume it in the IDE.
  So at most `2 x debug_workers` debug threads can be alive at once. Past that, a triggered request waits two seconds
  for one to finish. Then it gets a 503 and a log line telling you to go resume your abandoned sessions.
- Startup logs a WARN whenever the lane is armed. It logs another when `trigger_value` is empty, and another when
  `discover_client_host` is on.
- Only the request *path* is logged for lane requests, never the query string. The query string can carry the
  `trigger_value` secret.

The abandoned-session case from the end-to-end run shows the slot handoff working with a 2 second request timeout:

```
[+0.010s] session A: status=break (holding it paused)
[+2.009s] HTTP A: status=504 after 2.009s
[+2.015s] session B connected 0.003s after request B was sent (A still paused)
[+2.016s] HTTP B: status=200 after 0.003s
[+2.016s] session A resumed: status=stopping ; stop -> stopped ; engine closed connection
```

Mirroring Xdebug matters here too, instead of "route any trigger and let Xdebug sort it out". With `trigger_value` set,
a client without the secret can't make the server boot a debug worker at all.

### The bug Xdebug flushed out

Building this exposed something that had nothing to do with worker mode. In per-request mode (the default), ePHPm ran
`php_request_shutdown()` *lazily, at the start of the thread's next request*. That was left over from an older
thread-reuse model and nothing depended on it.

Under Xdebug that meant the `stopping` packet, the socket close and any profiler output all waited. They waited for
some unrelated later request to land on the same thread. The same root cause made sessions hang under the multi-thread
pool. The session module writes the session and releases its lock in RSHUTDOWN. The idle thread held the lock while the
next request for that session blocked on it on another thread. Reproduced on the released v0.11.2:

```
req 1: HTTP 200 in 0.0018s  n=1 handler=files
req 2: HTTP 000 in 8.0027s
req 3: HTTP 000 in 8.0028s
req 4: HTTP 000 in 8.0028s
```

The fix ([#541](https://github.com/ephpm/ephpm/pull/541)) split the request lifecycle into
`ephpm_begin_request()`, `ephpm_run_request()` and `ephpm_finish_request()`. The pool sends the response first and then
finishes the request. Teardown adds no latency, and it happens when the request ends. That one already shipped in
v0.11.3. Same script after the fix:

```
req 1: HTTP 200 in 0.000865s  n=1 handler=files
req 2: HTTP 200 in 0.001767s  n=2 handler=files
req 3: HTTP 200 in 0.000727s  n=3 handler=files
req 4: HTTP 200 in 0.000707s  n=4 handler=files
```

It also had a cross-tenant side. Tenant A's destructors were running under whatever tenant reached the thread next.
That deserves its own write-up. But I like this as a story about a debugger exposing a bug that had nothing to do with
debugging.

### Making it one flag: `ephpm dev --xdebug`

All of the above still left a real wall for users. Xdebug has to be a ZTS build that matches the embedded PHP's
`ZEND_MODULE_API_NO`. The distro and Sury packages are NTS, so they won't load. Telling people to build Xdebug from
source against matching headers is accurate and useless.

So `ephpm dev` is growing a flag ([#554](https://github.com/ephpm/ephpm/pull/554), in review):

```
ephpm dev --xdebug                          # catalog build, cached, offline after first run
ephpm dev --xdebug-path ./xdebug.so         # local build, no network (implies --xdebug)
          --xdebug-client-host <HOST>       # default 127.0.0.1 (WSL -> Windows IDE: host IP)
          --xdebug-client-port <PORT>       # default 9003
```

`--xdebug` does a few things:

- reads the binary's own PHP version, API number, ZTS flag and arch
- pulls the matching build from the php-sdk extension catalog
- checks the manifest's ABI fields and the sha256 of every file
- caches the result so later runs need no network

A mismatch is a hard refusal, never a quiet skip to another release. It also lifts the request deadline so a paused
breakpoint doesn't come back as a 504. Then it prints a banner with the IDE address and a VS Code `launch.json` hint:

```
  Xdebug 3.5.3 loaded (dev only: never load Xdebug in production)
    ide:      DBGp to 127.0.0.1:9003 (start the IDE listening first)
    trigger:  http://127.0.0.1:18080/?XDEBUG_TRIGGER=1
              or cookie XDEBUG_SESSION=1, or a browser "Xdebug helper" extension
    timeouts: request deadline and max_execution_time lifted while debugging
    vscode:   .vscode/launch.json → { "name": "Listen for Xdebug", "type": "php", "request": "launch", "port": 9003 }
              (no pathMappings needed when the IDE and ephpm share the filesystem)
```

One more papercut fell out of this. Xdebug is a Zend extension, so it has to be loaded with `zend_extension=` and an
absolute path. A relative path resolves against ePHPm's baked-in placeholder `extension_dir` and fails. Now
`[php] extensions` reads each shared object's symbols. Anything that exports both `zend_extension_entry` and
`extension_version_info` gets the right ini line written for it automatically.

Platform reality:

- Linux glibc works straight from the catalog, which ships Xdebug 3.5.3 in `ext-8.5.7`.
- macOS arm64 has catalog builds too, but it needs a binary newer than v0.11.3. Shared extensions only load there now
  that [#566](https://github.com/ephpm/ephpm/pull/566) exports the binary's symbols.
- The default Windows binary can't do it, because the static `php8embed.lib` can't load extension DLLs.
  [#564](https://github.com/ephpm/ephpm/pull/564) (in review) adds an opt-in shared-engine build that links the
  official `php8ts.dll`. With that one, `ephpm dev --xdebug` pulls `php_xdebug.dll` from the catalog and works on
  Windows too.

The debug lane and the macOS fix are merged on main and coming in the upcoming v0.12.0 release. The `dev` flag and the
Windows build are in review. Even once it lands, the Windows shared-engine build will be build-from-source at first.
Release artifacts ship the static build. So after that comes getting it published, since Windows is the platform where
I'd most like to use this.

### Links

- Worker-mode debug lane: [ephpm #552](https://github.com/ephpm/ephpm/pull/552) (`crates/ephpm-server/src/debug_lane.rs`, `xdebug_trigger.rs`)
- `ephpm dev --xdebug`: [ephpm #554](https://github.com/ephpm/ephpm/pull/554) (in review)
- Finish each request when it ends: [ephpm #541](https://github.com/ephpm/ephpm/pull/541)
- Export symbols on macOS so shared extensions load: [ephpm #566](https://github.com/ephpm/ephpm/pull/566)
- Windows shared-engine build: [ephpm #564](https://github.com/ephpm/ephpm/pull/564) (in review)
- [Xdebug step debugging docs](https://xdebug.org/docs/step_debug)
- [DBGp protocol](https://xdebug.org/docs/dbgp)
