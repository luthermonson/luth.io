---
title: "Security and Performance Hardening of a Rust Database Proxy"
description: "What a week of hardening litewire turned up. A dependency had a memory bug, one site could open its neighbor's database, a security rule lived only in a comment and every query paid for a needless 1 MB clear."
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
wrote about [how it works](/blog/2026/07/sqlite-production-database-protocol-translation/) and about [the MySQL quirks
WordPress depends on](/blog/2026/08/faking-mysql-quirks-wordpress/). This one is the less visible half of August:

- an audit pass on litewire
- a multi-tenant mode that turned a database proxy into a security boundary
- a performance regression I shipped and then had to go find

It's written in Rust. Memory safety is supposed to be the part I don't have to think about, and it mostly is. The
use-after-free in this story was in a dependency. The worst bug wasn't a memory bug at all.

### A gate instead of a cleanup

The first item from the audit was the RustSec advisory list. litewire had been accumulating them. `lru` 0.12.5 had
[RUSTSEC-2026-0253](https://rustsec.org/advisories/RUSTSEC-2026-0253.html), a use-after-free in `LruCache::pop`.
Version 0.12.5 was the one backing the production `TranslateCache` that sits in front of every translated statement.
It also had the older `IterMut` soundness advisory. Bumping to `lru` 0.18.2 needed no code changes, and the cache's
tests passed as-is. A handful of others went away with `cargo update`. Those were anyhow, crossbeam-epoch on the Turso
path, postgres-protocol and rustls-webpki.

Fixing the list once doesn't stop it from growing back. So [litewire#34](https://github.com/ephpm/litewire/pull/34)
added a `deny.toml` and a `cargo-deny` CI job that fails the build on a new advisory. The part I care about is that
nothing is ignored silently. Every entry in the ignore list says why it's there and when it can go:

```toml
# time 0.3.45 stack-exhaustion DoS in parsing. The fixed release (0.3.47)
# requires a newer Rust than this workspace's MSRV (1.85), so the
# MSRV-aware resolver cannot take it. Reached via mysql_common (wire
# date/time types) and turso's tracing-appender. Drop when the MSRV moves.
"RUSTSEC-2026-0009",
```

The ignores are:

- Old `rustls-webpki` copies pinned under opensrv-mysql and the tiberius test client. Under opensrv-mysql it's
  server-side TLS, which doesn't exercise the name-constraint and CRL paths those advisories are about.
- Two unmaintained crates with no runtime exposure.
- Two fixes blocked by the MSRV.

The gate paid for itself two days later. A new advisory landed for `h2` and reached litewire via axum and hyper.
RUSTSEC-2026-0258 is about empty DATA frames queued without limit. CI went red on an unrelated PR until the lockfile
moved to 0.4.16.

### A tenant could ATTACH its neighbour

Earlier in August litewire grew per-connection backend selection
([litewire#27](https://github.com/ephpm/litewire/pull/27)). A `ConnectionAuthenticator` runs during the MySQL
handshake and returns the backend that connection is bound to. That's how ePHPm puts many sites behind one listener
with a SQLite file each. SQLite has no `GRANT` and no per-schema ACL. The database file is the only tenant boundary,
and the authenticator is what's supposed to enforce it.

The audit asked whether anything screened what an authenticated tenant could send. The answer was worse than "no".
`ATTACH DATABASE '<path>'` and the path-bearing `PRAGMA`s pass MySQL-dialect translation verbatim. Every tenant's file
is opened by the same process as the same user. So on a rusqlite deployment a tenant could attach a neighbouring
tenant's file and read or write it. The Turso backend happened to refuse the statement, but that's not a property
litewire owns.

The fix wraps every authenticator-established session in a `BackendConn` decorator. Before the statement reaches any
engine, the decorator refuses:

- `ATTACH`/`DETACH`
- `VACUUM INTO '<path>'`
- the `data_store_directory`, `temp_store_directory` and `writable_schema` pragmas

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

A first-keyword check is easy to get around. So the screen goes further:

- It's quote-, comment- and multi-statement-aware.
- It looks through `EXPLAIN` and `EXPLAIN QUERY PLAN`.
- It matches every dot-separated component of a pragma name. That catches `PRAGMA main.data_store_directory` and
  `PRAGMA "writable_schema"`.
- It screens the `COM_STMT_PREPARE` path too.
- It treats SQL it can't make sense of as forbidden.

It keys off the session being tenant-scoped, not off the backend type. A future engine that accepts `ATTACH` doesn't
reopen the hole. Single-tenant use keeps full SQL, `ATTACH` included, because in a one-user embedded setup it's a
legitimate thing to do. The wire-level test attaches the other tenant's actual file path from a real `mysql_async`
client. It asserts that the client gets the screen's error and the connection survives.

### A security property guarded by a comment

The third one came out of reviewing ePHPm's per-site MySQL auth. opensrv-mysql calls the shim's `authenticate()` only
when the parsed handshake has a username. Its parser yields no username on the pre-TLS `CLIENT_SSL` branch. With
opensrv's default `tls` feature on, a re-parse makes that branch unreachable. Turn default features off and the
no-username path comes back. That's a perfectly normal thing to do to trim a dependency. On the authenticating path it
skips the only tenant boundary there is.

Nothing was exploitable as shipped. But the only thing protecting it was a comment in a *different repository*
([litewire#32](https://github.com/ephpm/litewire/issues/32)). Now the feature is declared explicitly with a `SECURITY`
comment in the workspace `Cargo.toml`. There's also a compile-time fence in `litewire-mysql`. It's an import of a
re-export that only exists when `tls` is enabled. I verified it by flipping the manifest to `default-features = false`
and watching the build fail at the fence. It wouldn't have failed before, since litewire's own code only uses ungated
APIs.

### A cache that doesn't allocate on a hit

The translate cache from July was a `Mutex<LruCache<(Dialect, String), Vec<TranslateResult>>>`. Every lookup built an
owned `(Dialect, String)` key just to hash it. Every hit deep-cloned the `Vec` of results for the caller.
[litewire#35](https://github.com/ephpm/litewire/pull/35) gives each dialect its own LRU keyed by `Box<str>`. That type
implements `Borrow<str>`, so a lookup can use the `&str` the client sent. Hits hand back a shared
`Arc<[TranslateResult]>`:

| translate-cache hit, single-thread hot key (release, median of 9 x 2M) | ns/op |
|---|---:|
| before: owned `String` key + `Vec` clone | 69.8 |
| after: borrowed `&str` key + `Arc` clone | 23.8 |

Those are the PR's numbers. I also tried sharding it and measured it as a net loss. An LRU `get` is a write because it
promotes the entry to most-recently-used. Concurrent hits on a hot statement contend on that entry's list links no
matter how many locks guard the map. The extra hash to pick a shard cost more than the lookup. Splitting a fixed
capacity across shards turned a skewed key distribution into thrashing. It didn't go in.

### The 1 MiB memset on every packet

This is the one I shipped. ePHPm's lab compared v0.7.0 against v0.6.3 and found the `pdo_mysql` wire path 14-31%
slower. The in-process `ephpm_db_*` bridge was flat while running in the same process against the same backend
([ephpm#322](https://github.com/ephpm/ephpm/issues/322)). Bridge flat and wire down puts the cost in the MySQL
frontend.

v0.7.0 had moved two pins at once: litewire and the Turso engine. A microbenchmark of all four combinations showed the
Turso bump was a no-op. Peeling the new frontend apart layer by layer put the entire regression on the `CommandFilter`
from [the stray OK packet fix](/blog/2026/08/faking-mysql-quirks-wordpress/). With the filter removed, throughput was
back where it started.

The filter reads the five-byte header and decides what to do. Then it passes the body through. The header is four
bytes of length and sequence plus the command byte. The filter returned those five bytes from one `poll_read` and made
the caller poll again for the body. That's a perfectly legal `AsyncRead`. It's also pathological for the one reader
that consumes it, opensrv-mysql's `PacketReader::next_async`:

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
body guarantees that second iteration. With it comes a `Vec::resize(1_048_576, 0)` for every command. From litewire's
side it looked like a five-byte read. `perf` on the regressed build put 12.8% of cycles in
`__memset_avx2_unaligned_erms`. That symbol doesn't appear in either the pre-filter build or the fixed one.

The fix ([litewire#37](https://github.com/ephpm/litewire/pull/37)) is to fall through from emitting the header into
passing the body inside the same `poll_read`. opensrv gets the whole packet the way an unfiltered socket delivered it.
That has a correctness corollary I'd missed the first time I thought about it. Once `poll_read` has written into the
caller's buffer it must not return `Pending`. The caller keeps the `ReadBuf`, so those bytes would sit there until the
next wakeup. In a half-duplex protocol the client is waiting on the reply to the command we're holding. So that wakeup
never comes. A small macro enforces it:

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

The end-to-end check is in [ephpm#323](https://github.com/ephpm/ephpm/pull/323). It used two ePHPm binaries built from
the same commit, with the litewire pin as the only difference. The server ran on one core. Every response was
body-checked, and each cell got two passes:

| cell | c | before (pass 1 / 2) | after (pass 1 / 2) | change |
|---|---:|---:|---:|---:|
| wire-point | 1 | 807.5 / 813.8 | 933.7 / 933.3 | +15.1% |
| wire-point | 16 | 820.0 / 852.6 | 1142.1 / 1142.9 | +36.7% |
| wire-wide | 1 | 1339.5 / 1343.0 | 1411.7 / 1411.8 | +5.3% |
| wire-wide | 16 | 1448.0 / 1499.6 | 1959.9 / 1965.0 | +33.1% |
| wire-write | 1 | 1702.3 / 1718.5 | 1815.0 / 1819.2 | +6.2% |

![ePHPm pdo_mysql wire path before and after the litewire fix, from ephpm#323](/img/2026/08/hardening-rust-database-proxy-e2e.svg)

The biggest wins are at c=16, which makes sense. A 1 MiB memset per command is pure CPU, and the server was pinned to
one core.

### Measuring it again myself

Those are lab numbers through PHP and HTTP. I wanted to see the frontend on its own. So I wrote a throwaway
integration test that starts litewire's MySQL frontend over a file-backed rusqlite database. It seeds 100 rows and
then runs 20,000 sequential point `SELECT`s over one `mysql_async` connection, three times. One persistent connection
isolates the per-command cost from connect and handshake. I built it at three revisions:

- `e34c639`, before the filter existed
- `10345a8`, the last revision before the fix, with the filter in place
- `344d2c2`, the fix

Then I ran them interleaved for three rounds.

![Point SELECT throughput over the litewire MySQL frontend at three revisions](/img/2026/08/hardening-rust-database-proxy-wire.svg)

| revision | rep 1 | rep 2 | rep 3 | mean of 9 reps |
|---|---:|---:|---:|---:|
| `e34c639` pre-filter | 10,706 | 8,158 | 7,972 | 8,945 q/s |
| `10345a8` filter | 9,251 | 6,921 | 6,959 | 7,710 q/s |
| `344d2c2` fix | 10,923 | 8,476 | 7,885 | 9,095 q/s |

Each rep column is the mean of the three rounds. The filter build is 13.8% slower than the pre-filter build. The fix
is 18.0% faster than the filter build and back within 2% of where it started. That lines up with the 15.7% the
microbenchmark in the PR measured on Linux over the Turso backend. Every arm got slower after its first rep, in the
same pattern. So I'm only comparing like with like and not reading anything into the absolute numbers. One caveat on
the pre-filter arm. At `e34c639` the dev-only `mysql_async` client was still 0.34, so that bar has a different client.
The security batch moved it to 0.37. The filter-versus-fix comparison is the one that matters, and it uses the same
client on both sides.

How I measured:

- AMD Ryzen 9 5950X (16 cores / 32 threads)
- 128 GB RAM
- Windows 11 Pro
- rustc 1.97.1, release build
- client and server in one process on a two-worker tokio runtime over loopback

I first tried the PR's shape, a fresh connection plus ten `SELECT`s per iteration. On Windows that run degraded as it
went. It eventually failed once the loopback connections piled up, so I switched to one long-lived connection. The
test was dropped into a temporary git worktree at each revision. It was built against the repo's existing target
directory:

```bash
BENCH_MODE=persistent BENCH_Q=20000 BENCH_REPS=3 \
  cargo test --release -p litewire --test blog_wire_bench -- --ignored --nocapture
```

### What I'd tell someone building one of these

Being in Rust took care of my own code's memory safety and nothing else. The use-after-free came from a dependency.
The fix was making the advisory database a build gate instead of a thing I check sometimes. The `ATTACH` hole was
plain logic. The moment a proxy picks a backend per connection it's a security boundary. Every statement the engine
accepts is part of the attack surface, whether the translator thought about it or not. The `tls` fence was about
moving a security property out of a comment and into something the compiler checks.

And the performance bug was a correct `AsyncRead` meeting a reader with an assumption I didn't know about. I only
found it because the lab runs the same workload against every release. The bridge path also gave me a control in the
same process.

### Links

- [litewire](https://github.com/ephpm/litewire)
- [litewire#34: cargo-deny gate, lru UAF fix, tenant SQL screen, tls fence](https://github.com/ephpm/litewire/pull/34)
- [litewire#35: borrowed-key Arc cache](https://github.com/ephpm/litewire/pull/35)
- [litewire#37: a whole command packet per poll_read](https://github.com/ephpm/litewire/pull/37)
- [ephpm#322: the regression report](https://github.com/ephpm/ephpm/issues/322)
- [ephpm#323: attribution and end-to-end verification](https://github.com/ephpm/ephpm/pull/323)
- [RUSTSEC-2026-0253](https://rustsec.org/advisories/RUSTSEC-2026-0253.html)
- [cargo-deny](https://github.com/EmbarkStudios/cargo-deny)
