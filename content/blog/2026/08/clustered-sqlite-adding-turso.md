---
title: "Clustered SQLite in ePHPm, Part 2: Adding Turso"
description: "Deciding a beta database engine is ready by testing it instead of reading its version number, and replacing both rusqlite and sqld with Turso in ePHPm v0.7.0."
date: 2026-08-19T20:37:00-07:00
tags: [ ephpm, sqlite, turso, rust, benchmarks, php ]
categories: [ ephpm, databases ]
series: [ "Clustered SQLite in ePHPm" ]
images:
 - src: img/2026/08/clustered-sqlite-adding-turso-four-lanes.svg
   alt: "Bar chart of four SQLite lanes at 16 concurrent connections: rusqlite and Turso single node, sqld and Turso CDC clustered"
   stretch: "none"
draft: false
---

[Part 1](/blog/2026/08/clustered-sqlite-sqld-sidecar/) ended with ePHPm running two SQLite engines: rusqlite in-process
for a single node, and an embedded sqld child process for a cluster. The sidecar fell off a cliff past four concurrent
writers and the best fix I could ship was a semaphore set to 1. This post is about replacing both engines with one, and
about how I decided that engine was ready.

- [Part 1: Replication by sidecar with sqld](/blog/2026/08/clustered-sqlite-sqld-sidecar/)
- Part 2: Adding Turso (this post)
- [Part 3: CDC over the cluster channel](/blog/2026/09/clustered-sqlite-cdc-cluster-channel/)

### The thesis

[Turso Database](https://github.com/tursodatabase/turso) is a rewrite of SQLite in Rust, MIT licensed, file-format
compatible, with MVCC for concurrent writers and change data capture built into the engine. Read that list again from the
perspective of Part 1. An in-process engine means no child process and no Hrana hop. MVCC means writers don't queue up
behind one lock. Built-in CDC means the engine itself can tell me what changed, so ePHPm's existing cluster layer can ship
it. One engine for both modes, and clustered SQLite with zero child processes.

I wrote that down as a roadmap page on 2026-07-10 ([#166](https://github.com/ephpm/ephpm/pull/166)). At the time Turso was
on its v0.7.0-pre line and labeled beta. I'm comfortable running a mature beta. The engine is MIT, the sync protocol is
open, it has a real user base, and it holds to SQLite's file format. What I'm not comfortable with is a label deciding for
me in either direction. "Beta" doesn't tell you whether it's safe for your workload, and neither does "1.0". So the roadmap
set five gates:

1. Upstream GA
2. Performance at parity or better on my matrix, tails included
3. File-format round trip, verified by me
4. Crash recovery
5. WordPress and Laravel end to end

Gate 1 is somebody else's decision. The other four I could close or fail with my own evidence.

### Phase 1: one knob, nothing else changes

The first PR ([#183](https://github.com/ephpm/ephpm/pull/183)) added exactly one setting:

```toml
[db.sqlite]
engine = "turso"   # default "sqlite"
```

Single node only, a warning at startup that it's experimental, and clustered plus Turso was a hard startup error. Unknown
values are also a startup error rather than a quiet fallback to the default. If you typo the engine name you should find
out at boot, not after a week of writing data into the engine you didn't pick.

litewire got a Turso backend ([litewire#9](https://github.com/ephpm/litewire/pull/9)) behind the same `Backend` trait the
other engines use, so the gate measurements could run at that seam: same SQL, same harness, back to back, on WSL2.

| op | rusqlite p50 | Turso p50 |
|---|---|---|
| point SELECT | 81.2 µs | 2.9 µs |
| INSERT (autocommit) | 138.5 µs | 17.3 µs |
| 10-query page | 840.1 µs | 30.2 µs |
| 8 concurrent writers | 3,609 inserts/s | 14,451 inserts/s |

Before anybody quotes the 28x on point SELECT: most of that gap is litewire's rusqlite backend paying a `spawn_blocking`
round trip (about 80 µs on that host) that the async-native Turso backend doesn't pay. It's a real cost ePHPm was paying,
but it's not "Turso reads 28x faster than SQLite." The INSERT gap and the concurrent writers number are the engine. Eight
writers, 2,000 rows, zero busy errors on either side, and MVCC did about 4x the throughput.

Gate 3 was a round trip with 500 rows of hostile data (`i64::MIN`/`MAX`, floats near ±1e308, NULLs in every column, CJK and
combining characters, BLOBs, a UNIQUE index), written by one engine and read by the other in both directions, checksums
compared. Both matched. Gate 4 got a smoke test: `kill -9` mid-write ten times, ten clean recoveries. That is a smoke test
and not the soak the gate asks for, and the gate stayed open.

### Benchmark the defaults before you benchmark the engines

The first run of that matrix made Turso look terrible on writes: an autocommit INSERT p50 of 1,815 µs, and 351 inserts/s
for the eight writers. That's 41x slower than the number in the table above.

The cause was in the Turso source, not my harness. Turso's default sync mode is `FULL`. litewire's rusqlite backend had
always set `synchronous=NORMAL` per session, so I was comparing an engine that fsyncs every commit to one that doesn't.
The Turso backend now sets `PRAGMA synchronous = NORMAL` per session, the same as the rusqlite one. If you are about to
compare Turso to a tuned SQLite, check this first. Otherwise you are benchmarking two durability settings and calling it an
engine comparison.

### Gate 5: the failures weren't Turso's

Gate 5 ([#188](https://github.com/ephpm/ephpm/pull/188)) installed WordPress and Laravel on both engines through
`pdo_mysql`, with rusqlite as the control. As pinned, it failed. Neither app installed on *either* engine. Running the
control first is what made that easy to read: all nine blockers were in litewire's MySQL frontend and SQL translation
layer, and they fired before any engine code ran. With a local litewire patch for those nine, both apps went green on both
engines, and Turso matched the control on all 20 matrix rows.

The one Turso-specific thing it found was a real bug. Selecting from `pragma_table_info()` left the connection in a
phantom transaction state. Later writes reported success and were never durable. WordPress "installed" into a 4 KB
database with an empty WAL. Turso already had this as
[issue #7466](https://github.com/tursodatabase/turso/issues/7466), so I traced it into the VDBE, found that virtual table
cursors weren't closed before transaction finalization, and sent the fix upstream as
[turso#7855](https://github.com/tursodatabase/turso/pull/7855). It was merged the next day.

That's what I mean by a mature beta. The failures in my integration were mostly mine, the one engine bug was already
reported upstream, and a fix from an outside contributor went from PR to merged in about a day. That tells me more about
whether I can depend on a project than the word "beta" does.

### The benchmark that made the case

For v0.6.0 I measured four lanes with the same fixtures, one machine, one session, `--cpus 1` per node, 100% 2xx checked
on every cell ([#216](https://github.com/ephpm/ephpm/pull/216), release numbers in the
[benchmark results](https://ephpm.dev/benchmarking/results/)):

- **A**: rusqlite, single node
- **B**: Turso, single node
- **C**: rusqlite + sqld sidecar, clustered (the shipping clustered path at the time)
- **D**: Turso + CDC replication, clustered, no sidecar (experimental, Part 3)

![four lanes at c=16](/img/2026/08/clustered-sqlite-adding-turso-four-lanes.svg)

| lane | reads c=16 RPS | reads p99 | writes c=1 RPS | writes c=16 RPS | writes p99 |
|---|---|---|---|---|---|
| A rusqlite single | 612 | 46.02 ms | 648 | 1130 | 18.68 ms |
| B Turso single | 646 | 30.93 ms | 666 | 1120 | 20.77 ms |
| C sqld cluster | 240 | 92.07 ms | 384 | 0 completed | - |
| D Turso cluster | 601 | 33.19 ms | 558 | 876 | 26.34 ms |

Reads are the 10-SELECT `db.php` fixture, writes are one INSERT per request. The zero for lane C is not a rounding error.
That cell completed nothing in both 20 second reps.

Three things I take from this:

- **Single node is a wash on writes and a win on read tails.** Writes are even (rusqlite has a slightly better c=16 p99,
  within this harness's run-to-run spread). Reads are 15% more throughput at c=1 (371 vs 322 RPS in
  the release tables) and a much steadier c=16 tail.
- **Clustered is not close.** CDC replication does 2.5x the read throughput of the sidecar with a third of the p99.
- **Cost of replication against each engine's own baseline.** At c=16 writes, clustered Turso keeps 78% of single-node
  Turso's throughput while shipping every change to a replica. Clustered rusqlite keeps 0%.

These are A/B numbers on one box under podman on WSL2. The absolute RPS won't transfer to your hardware. The deltas are
the claim.

### v0.7.0: Turso only

Gate 2 was substantially evidenced and gate 3 was met. Gate 5 had the matrix, plus a live WordPress and WooCommerce demo
on Turso, but not yet an automated suite in CI. Gates 1 and 4 were open. Then a pen-test of multi-site mode turned up
[#274](https://github.com/ephpm/ephpm/issues/274): `ATTACH DATABASE` from one tenant's PHP could write a PHP file into
another tenant's docroot and execute it. Cross-tenant RCE, through the rusqlite path. That changed the trade-off. Keeping
an engine with that primitive around was more dangerous than adopting a beta engine that had passed my gates, so the
remaining gates now gate the GA label, not the ship.

[#278](https://github.com/ephpm/ephpm/pull/278) is the demolition, +681 / -2075 lines. Gone:

- the `ephpm-sqld` crate, the `include_bytes!` embedding, the extract/chmod/spawn/health/SIGTERM state machine
- the sqld download in `xtask` and the Windows special case in the release build
- the rusqlite single-node path in ePHPm (litewire still ships the backend for its other users)
- `[db.sqlite.sqld]` and `write_permits`, kept as parsed-but-warned shims so an old config tells you what happened

The `engine` knob now defaults to `"turso"` and that's the only accepted value. `engine = "sqlite"` is a hard startup
error that points you at the v0.7.0 upgrade notes. The reasoning in the PR: a silent fallback would have run Turso on
data the user thought was rusqlite, which is exactly the failure mode the error exists to prevent. A silent swap would
be worse than an error you have to read once.

The migration got verified rather than assumed. Standard SQLite files made by the `sqlite3` CLI 3.46.1, in WAL and in
rollback journal mode, with non-ASCII text, BLOBs and an index, opened through ePHPm's actual Turso path with
`PRAGMA integrity_check` returning `ok` and every row intact. So a cleanly shut down 0.6.x database upgrades in place, with
no dump and reload. There are two caveats in the docs: a database with a hot, uncheckpointed `-wal` left by a crash wasn't
verified to replay (shut down cleanly before you upgrade), and non-UTF-8 TEXT may not round trip because Turso surfaces
TEXT as a Rust `String`.

v0.7.0 was tagged on 2026-08-18 with `turso_core` 0.7.2, which includes the #7855 fix. Clustering now always goes through
the CDC path, and the cluster channel binds whenever SQLite resolves to clustered mode. Windows doesn't have a sqld
blocker anymore, but clustered on Windows is untested so the docs don't claim it.

### What's next

The engine is in-process now and so is replication, so one binary is actually one process. But the replication path in
v0.7.0 has only been exercised the friendly way: bring up nodes, write, watch rows show up. Nobody has killed the primary
in a test with data on it yet. That, plus the cluster channel that carries the CDC stream, is
[Part 3](/blog/2026/09/clustered-sqlite-cdc-cluster-channel/).

### Links

- [Turso Database](https://github.com/tursodatabase/turso) and [turso#7855](https://github.com/tursodatabase/turso/pull/7855)
- [#166 roadmap: Turso engine](https://github.com/ephpm/ephpm/pull/166)
- [#183 Phase 1 engine knob and gate evidence](https://github.com/ephpm/ephpm/pull/183)
- [#188 gate 5: WordPress and Laravel on Turso](https://github.com/ephpm/ephpm/pull/188)
- [#216 Turso vs SQLite, measured](https://github.com/ephpm/ephpm/pull/216)
- [#274 cross-tenant RCE via ATTACH](https://github.com/ephpm/ephpm/issues/274)
- [#278 v0.7.0 Turso only](https://github.com/ephpm/ephpm/pull/278)
- [litewire#9 Turso backend](https://github.com/ephpm/litewire/pull/9)
