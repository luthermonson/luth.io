---
title: "ePHPm Odds and Ends: OOM-Killed Links, Opcode Scanning, WebSockets and a 44 ms Stall"
description: "Four smaller ePHPm stories that didn't need a post each but were too useful to leave in PR descriptions."
date: 2026-09-18T20:05:00-07:00
tags: [ php, rust, ephpm, ci, websockets, tcp ]
categories: [ ephpm, php ]
images:
 - src: img/2026/09/ephpm-smaller-angles-nodelay.svg
   alt: "Bar chart: db.php p50 at c=1 dropped from 444 ms on v0.4.0 to 4.4 ms on v0.4.1"
   stretch: "none"
draft: false
---

Not every fix in [ePHPm](https://github.com/ephpm/ephpm) earns a full write-up, but a few of the smaller ones have
stuck with me because the lesson in each was bigger than the diff. Here are four from this summer.

### A release build the OOM killer kept eating

ePHPm statically links libphp, ICU and a pile of other C libraries into one Rust binary, and it ships for four targets.
For the v0.4.2 performance release I set the release profile to `lto = "fat"` with `codegen-units = 1`
([#171](https://github.com/ephpm/ephpm/pull/171)). Fat LTO pulls every crate's LLVM IR into one module in one process
and optimizes it as a unit, and one codegen unit removes the parallelism that would otherwise spread that memory out.
That's one very large process at the final link.

The linux-aarch64 legs died first, rustc killed with `(signal: 9, SIGKILL)` during the link on three consecutive
matrix runs, including one where the leg had the host to itself. The fix ladder:

- **Thin LTO for arm64** ([#181](https://github.com/ephpm/ephpm/pull/181)). Enough for about ten days.
- **LTO off for arm64 and Windows** ([#198](https://github.com/ephpm/ephpm/pull/198)) once the Turso engine landed and
  grew the IR past what thin LTO fit into. The git history made the tipping point obvious: Windows built fine on fat LTO
  through v0.5.0 and started OOMing in v0.5.1, the first release with Turso linked in.
- **`RUST_MIN_STACK=32MiB` on Windows** ([#199](https://github.com/ephpm/ephpm/pull/199)). With LTO off, the
  Windows build got far enough to hit a *different* wall: `sqlparser`'s deeply recursive AST types overflowed rustc's
  default thread stack (`STATUS_STACK_OVERFLOW`). Only on MSVC; arm64 compiled it fine.
- **The same fix for nightly** ([#203](https://github.com/ephpm/ephpm/pull/203)), which had failed arm64 for eight
  straight runs. In most of those the OOM killer took the *runner agent* too, so GitHub reported "the self-hosted runner
  lost communication with the server" instead of a build error. Same failure, different hat, and that's exactly why it
  had been read as flaky infrastructure.

One detail from #203 I'll repeat to anybody who touches a workflow file: the value is quoted, `"off"`. Bare `off` in
YAML is boolean `false`, and Cargo's `lto = false` means *thin-local*, not off.

Every one of those PRs said `codegen-units = 1` was a contributing factor, and nobody had measured it. So instead of
changing release CI on reasoning, [#249](https://github.com/ephpm/ephpm/pull/249) added an opt-in `workflow_dispatch`
probe that builds the same commit with `lto` and `codegen_units` as inputs and records two numbers: total peak memory
(what the OOM killer acts on) and the largest single process (what `rustc-LLVM ERROR: out of memory` is about). The
first successful Windows run, LTO off with one codegen unit, peaked at 5.58 GiB total and 2.66 GiB for rustc alone,
with a 1001-second build on a 10 GiB host. That's the baseline any change to `codegen-units` now has to beat.

### Using the embedded Zend compiler as a static analyzer

`ephpm analyze` ([#502](https://github.com/ephpm/ephpm/pull/502)) is a fail-closed gate you run before deploying PHP:
an analyzer framework, a policy engine, SARIF output, and some analyzers that shell out to existing tools (composer
audit, semgrep, YARA). Any analyzer error, timeout or panic floors the verdict at quarantine, because a run where
something that should have spoken didn't can never be an "allow".

The interesting one is `opcode-scan` ([#509](https://github.com/ephpm/ephpm/pull/509)). PHPStan, Psalm and semgrep all
re-implement PHP parsing. ePHPm already has PHP's actual compiler linked in, so `opcode-scan` compiles each file with
`zend_compile_file` under `ZEND_COMPILE_WITHOUT_EXECUTION` (the `opcache_compile_file()` model) and walks the opcode
stream for dangerous calls. Nothing executes. It looks for `ZEND_INCLUDE_OR_EVAL` with `ZEND_EVAL`, and for statically
named calls through the `INIT_FCALL` family to `system`, `exec`, `shell_exec`, `passthru`, `proc_open`, `popen`,
`create_function`, `assert` and `unserialize`, in every op_array: functions, methods, closures, arrow functions and
conditional declarations.

Asking the compiler beats a text scan in both directions. Opcodes have no comments and no strings-as-code, so `system`
in a docblock produces nothing. And the compiler lowers things a text scan can't see: backticks arrive as a genuine
`shell_exec` call. So findings are reported as confirmed rather than suspected. Some boring details matter for a
scanner that runs inside a server process: the whole compile-and-walk runs inside `zend_try`/`zend_catch` so a fatal
never longjmps into Rust, a file that doesn't compile is a gating error (code that can't compile can't be vouched for),
and class and function table registrations get pruned after each file so two files declaring the same symbol both scan.

My favorite footnote is in #502's review notes: the test fixtures assemble webshell-shaped strings at runtime, because
Windows Defender had quarantined earlier fixture *source files* as `Backdoor:PHP/*`. If your security scanner's tests
don't trip the antivirus at least once, are they even testing anything?

### WebSockets where PHP never holds the socket

[#304](https://github.com/ephpm/ephpm/pull/304) added native WebSockets, but not the Swoole or ReactPHP way. Rust owns
the socket and PHP runs once per event, which is the API Gateway (`$connect`/`$disconnect`/message) shape. An idle
connection costs a registry entry and some reactor memory, not a PHP process, so ten thousand idle connections cost no
PHP at all (once you raise `max_connections_per_site`, which defaults to 1,000 and counts a single-site server as one
site). Every event goes through the same `handle_php` path as an HTTP request, so the per-site database, KV
keyspace, `open_basedir` sandbox and crash guard all apply without a parallel implementation to keep in sync.

One script, `websocket.php` at the vhost's document root, routes on `$_SERVER['WS_EVENT']`:

```php
<?php
$event = $_SERVER['WS_EVENT'];
$conn  = $_SERVER['WS_CONNECTION_ID'];

switch ($event) {
    case 'connect':
        // a non-2xx response refuses the upgrade, verbatim
        if (verify_token($_GET['token'] ?? '') === null) {
            http_response_code(401);
            return;
        }
        ephpm_ws_subscribe('lobby');
        http_response_code(200);
        return;

    case 'message':
        $payload = json_decode(file_get_contents('php://input'), true);
        ephpm_ws_broadcast('lobby', json_encode(['text' => $payload['text'] ?? '']));
        return;
}
```

The `connect` event runs on the real upgrade request, before the handshake completes, so it sees the cookies and query
string that auth needs and can refuse with a real 401. Two decisions I'd copy elsewhere. The implicit `ephpm_ws_send()`
called from an ordinary HTTP request *throws* rather than returning false, because a push API that silently succeeds is
the worst possible failure mode. And every bound is a bound: each connection has a fixed outbound queue, and on overflow
the socket closes with 1013 instead of buffering. One slow reader costs one connection, never the server's memory.

### The 44 ms stall, and the fix that got reverted

The single most productive class of performance bug in ePHPm has been `TCP_NODELAY` on small-response paths. The
worst case was the MySQL wire frontend in [litewire](https://github.com/ephpm/litewire). With PHP talking to the
in-process SQLite over `pdo_mysql`, single-row INSERTs took about a millisecond and SELECTs were pinned at a fixed ~44
ms. That asymmetry is the fingerprint of Nagle plus delayed ACK: a one-packet response (INSERT OK) can't trigger it, a
multi-packet result set can. A fixed timer value is a timer, not a slow query.

The non-obvious part: setting `TCP_NODELAY` on the server side alone did *not* fix it, because the mysqlnd client's
side of the exchange mattered too. The real fix was coalescing the whole result set into a single write. From the lab's
v0.4.1 retest, ten sequential PDO queries per request:

![db.php latency before and after](/img/2026/09/ephpm-smaller-angles-nodelay.svg)

That's 444.0 ms p50 down to 4.4 ms at c=1, and 35 to 106 requests per second at c=16. The same audit found the main
HTTP listener didn't set nodelay either, on the assumption that hyper does it. It doesn't.

The part that actually changed how I release things: a rebase conflict during a stacked-PR merge silently dropped
the litewire `set_nodelay` lines. The commit was in history, but its changes had been overwritten by a `--theirs`
resolution. CI was green. The DB benchmark on the release candidate still showed the full 44 ms stall, and that's the
only reason it didn't ship as a headline that was false on the flagship path. "The code merged and CI is green" isn't
the same claim as "the fix is in the binary". Measure the release candidate.

### Links

- LTO and the arm64/Windows OOMs: [#171](https://github.com/ephpm/ephpm/pull/171), [#181](https://github.com/ephpm/ephpm/pull/181), [#198](https://github.com/ephpm/ephpm/pull/198), [#199](https://github.com/ephpm/ephpm/pull/199), [#203](https://github.com/ephpm/ephpm/pull/203), [#249](https://github.com/ephpm/ephpm/pull/249)
- `ephpm analyze` and `opcode-scan`: [#502](https://github.com/ephpm/ephpm/pull/502), [#509](https://github.com/ephpm/ephpm/pull/509)
- Native WebSockets: [#304](https://github.com/ephpm/ephpm/pull/304)
- TCP_NODELAY findings: `site/content/benchmarking/findings.md` in the ePHPm repo; v0.4.1 retest numbers from the ePHPm lab's `docs/ephpm-0.4.1-retest.md`
