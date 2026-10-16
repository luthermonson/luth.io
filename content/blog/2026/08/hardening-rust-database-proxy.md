---
title: "Security and Performance Hardening of a Rust Database Proxy"
description: "One week of hardening litewire: a memory bug in a dependency, a site that could open its neighbor's database, a security rule that lived only in a comment, and a needless 1 MB clear on every query."
date: 2026-08-27T20:30:00-07:00
tags: [ litewire, ephpm, rust, mysql, sqlite, security, performance ]
categories: [ databases, rust, security ]
images:
 - src: img/2026/08/hardening-rust-database-proxy-wire.svg
   alt: "Bar chart of point SELECT throughput over the litewire MySQL frontend at three revisions: before the command filter, with the filter, and with the poll_read fix"
   stretch: "none"
draft: false
---

[litewire](https://github.com/ephpm/litewire) is the piece of ePHPm that lets PHP apps talk MySQL to a SQLite file. I
wrote about [how it works](/blog/2026/07/sqlite-production-database-protocol-translation/) and about
[the MySQL quirks WordPress depends on](/blog/2026/08/faking-mysql-quirks-wordpress/). This one is the less visible
half of August: an audit pass on litewire, a multi-tenant mode that turned a database proxy into a security boundary,
and a performance regression I shipped and then had to go find.

It's written in Rust, so memory safety is supposed to be the part I don't have to think about. It mostly is. The
use-after-free in this story was in a dependency, and the worst bug wasn't a memory bug at all.

### A gate instead of a cleanup

The first item from the audit was the RustSec advisory list. litewire had been accumulating them: `lru` 0.12.5 had
[RUSTSEC-2026-0253](https://rustsec.org/advisories/RUSTSEC-2026-0253.html), a use-after-free in `LruCache::pop`, and
0.12.5 was the version backing the production `TranslateCache` that sits in front of every translated statement. It
also had the older `IterMut` soundness advisory. Bumping to `lru` 0.18.2 needed no code changes and the cache's tests
passed as-is. A handful of others went away with `cargo update` (anyhow, crossbeam-epoch on the Turso path,
postgres-protocol, rustls-webpki).

Fixing the list once doesn't stop it from growing back, so [litewire#34](https://github.com/ephpm/litewire/pull/34)
added a `deny.toml` and a `cargo-deny` CI job that fails the build on a new advisory. The part I care about is that
nothing is ignored silently. Every entry in the ignore list says why it's there and when it can go:

```toml
# time 0.3.45 stack-exhaustion DoS in parsing. The fixed release (0.3.47)
# requires a newer Rust than this workspace's MSRV (1.85), so the
# MSRV-aware resolver cannot take it. Reached via mysql_common (wire
# date/time types) and turso's tracing-appender. Drop when the MSRV moves.
"RUSTSEC-2026-0009",
```

The ignores are old `rustls-webpki` copies pinned under opensrv-mysql (server-side TLS, which doesn't exercise the
name-constraint and CRL paths those advisories are about) and the tiberius test client, two unmaintained crates with no
runtime exposure, and two fixes blocked by the MSRV. The gate paid for itself two days later: a new advisory landed
for `h2` (RUSTSEC-2026-0258, empty DATA frames queued without limit) reaching litewire via axum and hyper, and CI went
red on an unrelated PR until the lockfile moved to 0.4.16.

### A tenant could ATTACH its neighbour

Earlier in August litewire grew per-connection backend selection
([litewire#27](https://github.com/ephpm/litewire/pull/27)). A `ConnectionAuthenticator` runs during the MySQL handshake
and returns the backend that connection is bound to, which is how ePHPm puts many sites behind one listener with a
SQLite file each. SQLite has no `GRANT` and no per-schema ACL, so the database file is the only tenant boundary, and
the authenticator is what's supposed to enforce it.

The audit asked whether anything screened what an authenticated tenant could send. The answer was worse than "no".
`ATTACH DATABASE '<path>'` and the path-bearing `PRAGMA`s pass MySQL-dialect translation verbatim, and every tenant's
file is opened by the same process as the same user. So on a rusqlite deployment a tenant could attach a neighbouring
tenant's file and read or write it. The Turso backend happened to refuse the statement, but
that's not a property litewire owns.

The fix wraps every authenticator-established session in a `BackendConn` decorator that refuses `ATTACH`/`DETACH`,
`VACUUM INTO '<path>'` and the `data_store_directory`, `temp_store_directory` and `writable_schema` pragmas before the
statement reaches any engine:

```rust
/// Statement keywords always rejected on a tenant session: the
/// cross-database primitives.
const FORBIDDEN_KEYWORDS: [&str; 2] = ["ATTACH", "DETACH"];

const FORBIDDEN_PRAGMAS: [&str; 3] = [
    "writable_schema",
    "temp_store_directory",
    "data_store_directory",
];
```

A first-keyword check is easy to get around, so the screen is quote-, comment- and multi-statement-aware, looks through
`EXPLAIN` and `EXPLAIN QUERY PLAN`, matches every dot-separated component of a pragma name so
`PRAGMA main.data_store_directory` and `PRAGMA "writable_schema"` are caught, screens the `COM_STMT_PREPARE` path too,
and treats SQL it can't make sense of as forbidden. It keys off the session being tenant-scoped, not off the backend
type, so a future engine that accepts `ATTACH` doesn't reopen the hole. Single-tenant use keeps full SQL, `ATTACH`
included, because in a one-user embedded setup it's a legitimate thing to do. The wire-level test attaches the other
tenant's actual file path from a real `mysql_async` client and asserts it gets the screen's error and the connection
survives.

### A security property guarded by a comment

The third one came out of reviewing ePHPm's per-site MySQL auth. opensrv-mysql calls the shim's `authenticate()` only
when the parsed handshake has a username, and its parser yields no username on the pre-TLS `CLIENT_SSL` branch. With
opensrv's default `tls` feature on, a re-parse makes that branch unreachable. Turn default features off, which is a
perfectly normal thing to do to trim a dependency, and the no-username path comes back. On the authenticating path,
that skips the only tenant boundary there is.

Nothing was exploitable as shipped. But the only thing protecting it was a comment in a *different repository*
([litewire#32](https://github.com/ephpm/litewire/issues/32)). Now the feature is declared explicitly with a `SECURITY`
comment in the workspace `Cargo.toml`, and there's a compile-time fence in `litewire-mysql`: an import of a re-export
that only exists when `tls` is enabled. I verified it by flipping the manifest to `default-features = false` and
watching the build fail at the fence, which it wouldn't have done before since litewire's own code only uses ungated
APIs.

### A cache that doesn't allocate on a hit

The translate cache from July was a `Mutex<LruCache<(Dialect, String), Vec<TranslateResult>>>`. Every lookup built an
owned `(Dialect, String)` key just to hash it, and every hit deep-cloned the `Vec` of results for the caller.
[litewire#35](https://github.com/ephpm/litewire/pull/35) gives each dialect its own LRU keyed by `Box<str>`, which
implements `Borrow<str>` so a lookup can use the `&str` the client sent, and hands back a shared `Arc<[TranslateResult]>`:

| translate-cache hit, single-thread hot key (release, median of 9 x 2M) | ns/op |
|---|---:|
| before: owned `String` key + `Vec` clone | 69.8 |
| after: borrowed `&str` key + `Arc` clone | 23.8 |

Those are the PR's numbers. I also tried sharding it and measured it as a net loss. An LRU `get` is a write, because it
promotes the entry to most-recently-used, so concurrent hits on a hot statement contend on that entry's list links no
matter how many locks guard the map. The extra hash to pick a shard cost more than the lookup, and splitting a fixed
capacity across shards turned a skewed key distribution into thrashing. It didn't go in.

### The 1 MiB memset on every packet

This is the one I shipped. ePHPm's lab compared v0.7.0 against v0.6.3 and found the `pdo_mysql` wire path 14-31%
slower, while the in-process `ephpm_db_*` bridge, running in the same process against the same backend, was flat
([ephpm#322](https://github.com/ephpm/ephpm/issues/322)). Bridge flat and wire down puts the cost in the MySQL frontend.

v0.7.0 had moved two pins at once, litewire and the Turso engine. A microbenchmark of all four combinations showed the
Turso bump was a no-op, and peeling the new frontend apart layer by layer put the entire regression on the
`CommandFilter` from [the stray OK packet fix](/blog/2026/08/faking-mysql-quirks-wordpress/). With the filter removed,
throughput was back where it started.

The filter reads the five-byte header (four bytes of length and sequence, plus the command byte), decides what to do,
and then passes the body through. It returned those five bytes from one `poll_read` and made the caller poll again for
the body. That's a perfectly legal `AsyncRead`. It's also pathological for the one reader that consumes it,
opensrv-mysql's `PacketReader::next_async`:

```rust
let mut buffer_size = PACKET_BUFFER_SIZE;      // 4 KiB
loop {
    ...                                        // parse attempt
    if self.bytes.len() - end < buffer_size {
        self.bytes.resize(max(buffer_size, end * 2), 0);   // <- memset
    }
    let read = self.r.read(&mut self.bytes[end..]).await?;
    buffer_size = PACKET_LARGE_BUFFER_SIZE;    // 1 MiB
}
```

The scratch buffer escalates from 4 KiB to 1 MiB on the *second* iteration. A reader that hands back a header with no
body guarantees that second iteration, and with it a `Vec::resize(1_048_576, 0)` for every command. From litewire's
side it looked like a five-byte read. `perf` on the regressed build put 12.8% of cycles in
`__memset_avx2_unaligned_erms`, a symbol that doesn't appear in either the pre-filter build or the fixed one.

The fix ([litewire#37](https://github.com/ephpm/litewire/pull/37)) is to fall through from emitting the header into
passing the body inside the same `poll_read`, so opensrv gets the whole packet the way an unfiltered socket delivered
it. That has a correctness corollary I'd missed the first time I thought about it: once `poll_read` has written into
the caller's buffer it must not return `Pending`. The caller keeps the `ReadBuf`, so those bytes would sit there until
the next wakeup, and in a half-duplex protocol where the client is waiting on the reply to the command we're holding,
that wakeup never comes. A small macro enforces it:

```rust
macro_rules! poll_io {
    ($poll:expr) => {
        match $poll {
            Poll::Ready(Ok(v)) => v,
            Poll::Ready(Err(e)) => return Poll::Ready(Err(e)),
            Poll::Pending => {
                return if out.filled().len() > started {
                    Poll::Ready(Ok(()))
                } else {
                    Poll::Pending
                };
            }
        }
    };
}
```

Both properties have tests: `one_poll_read_yields_the_whole_packet` and
`partial_packet_returns_what_it_has_instead_of_parking`.

End to end, two ePHPm binaries built from the same commit with the litewire pin as the only difference, server on one
core, every response body-checked, two passes each ([ephpm#323](https://github.com/ephpm/ephpm/pull/323)):

| cell | c | before (pass 1 / 2) | after (pass 1 / 2) | change |
|---|---:|---:|---:|---:|
| wire-point | 1 | 807.5 / 813.8 | 933.7 / 933.3 | +15.1% |
| wire-point | 16 | 820.0 / 852.6 | 1142.1 / 1142.9 | +36.7% |
| wire-wide | 1 | 1339.5 / 1343.0 | 1411.7 / 1411.8 | +5.3% |
| wire-wide | 16 | 1448.0 / 1499.6 | 1959.9 / 1965.0 | +33.1% |
| wire-write | 1 | 1702.3 / 1718.5 | 1815.0 / 1819.2 | +6.2% |

![ePHPm pdo_mysql wire path before and after the litewire fix, from ephpm#323](/img/2026/08/hardening-rust-database-proxy-e2e.svg)

The biggest wins are at c=16, which makes sense: a 1 MiB memset per command is pure CPU, and the server was pinned to
one core.

### Measuring it again myself

Those are lab numbers through PHP and HTTP. I wanted to see the frontend on its own, so I wrote a throwaway
integration test that starts litewire's MySQL frontend over a file-backed rusqlite database, seeds 100 rows, and then
runs 20,000 sequential point `SELECT`s over one `mysql_async` connection, three times. One persistent connection
isolates the per-command cost from connect and handshake. I built it at three revisions: `e34c639` (before the
filter existed), `10345a8` (the last revision before the fix, filter in place) and `344d2c2` (the fix), then ran them interleaved for three rounds.

![Point SELECT throughput over the litewire MySQL frontend at three revisions](/img/2026/08/hardening-rust-database-proxy-wire.svg)

| revision | rep 1 | rep 2 | rep 3 | mean of 9 reps |
|---|---:|---:|---:|---:|
| `e34c639` pre-filter | 10,706 | 8,158 | 7,972 | 8,945 q/s |
| `10345a8` filter | 9,251 | 6,921 | 6,959 | 7,710 q/s |
| `344d2c2` fix | 10,923 | 8,476 | 7,885 | 9,095 q/s |

(Each rep column is the mean of the three rounds.) The filter build is 13.8% slower than the pre-filter build, and the
fix is 18.0% faster than the filter build and back within 2% of where it started. That lines up with the 15.7% the
microbenchmark in the PR measured on Linux over the Turso backend. Every arm got slower after its first rep, in the same
pattern, so I'm only comparing like with like and not reading anything into the absolute numbers. One caveat on the
pre-filter arm: at `e34c639` the dev-only `mysql_async` client was still 0.34 (the security batch moved it to 0.37), so
that bar has a different client. The filter-versus-fix comparison, which is the one that matters, uses the same client
on both sides.

How I measured: AMD Ryzen 9 5950X (16 cores / 32 threads), 128 GB RAM, Windows 11 Pro, rustc 1.97.1, release build,
client and server in one process on a two-worker tokio runtime over loopback. I first tried the PR's shape (a fresh
connection plus ten `SELECT`s per iteration), but on Windows that run degraded as it went and eventually failed once
the loopback connections piled up, so I switched to one long-lived connection. The test was dropped into a temporary
git worktree at each revision and built against the repo's existing target directory:

```bash
BENCH_MODE=persistent BENCH_Q=20000 BENCH_REPS=3 \
  cargo test --release -p litewire --test blog_wire_bench -- --ignored --nocapture
```

### What I'd tell someone building one of these

Being in Rust took care of my own code's memory safety and nothing else. The use-after-free came from a dependency, and
the fix was making the advisory database a build gate instead of a thing I check sometimes. The `ATTACH` hole was plain
logic: the moment a proxy picks a backend per connection it's a security boundary, and every statement the engine
accepts is part of the attack surface whether the translator thought about it or not. The `tls` fence was about moving a
security property out of a comment and into something the compiler checks.

And the performance bug was a correct `AsyncRead` meeting a reader with an assumption I didn't know about. I only found
it because the lab runs the same workload against every release and the bridge path gave me a control in the same
process.

### Links

- [litewire](https://github.com/ephpm/litewire)
- [litewire#34: cargo-deny gate, lru UAF fix, tenant SQL screen, tls fence](https://github.com/ephpm/litewire/pull/34)
- [litewire#35: borrowed-key Arc cache](https://github.com/ephpm/litewire/pull/35)
- [litewire#37: a whole command packet per poll_read](https://github.com/ephpm/litewire/pull/37)
- [ephpm#322: the regression report](https://github.com/ephpm/ephpm/issues/322)
- [ephpm#323: attribution and end-to-end verification](https://github.com/ephpm/ephpm/pull/323)
- [RUSTSEC-2026-0253](https://rustsec.org/advisories/RUSTSEC-2026-0253.html)
- [cargo-deny](https://github.com/EmbarkStudios/cargo-deny)
