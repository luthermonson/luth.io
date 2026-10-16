---
title: "Clustered SQLite in ePHPm, Part 1: Replication by Sidecar"
description: "ePHPm first made SQLite highly available by embedding sqld as a child process. This is how that worked and the list of things that went wrong with it."
date: 2026-08-08T20:14:00-07:00
tags: [ ephpm, sqlite, sqld, libsql, rust, clustering, php ]
categories: [ ephpm, databases ]
series: [ "Clustered SQLite in ePHPm" ]
images:
 - src: img/2026/08/clustered-sqlite-sqld-sidecar-topology.svg
   alt: "Two ePHPm nodes, each running an embedded sqld child process, connected by gossip, a KV data plane and sqld gRPC"
   stretch: "none"
draft: false
---

Most PHP sites I have run in the last twenty years did not need a MySQL server. They needed *a* database, and MySQL was the
one every framework spoke. With [ePHPm](https://github.com/ephpm/ephpm) I have been trying to get rid of that server for
the common case. The piece that makes it possible is [litewire](https://github.com/ephpm/litewire). PHP connects with
plain `pdo_mysql`. litewire speaks the MySQL wire protocol, and the queries land in a SQLite file. WordPress and Laravel
don't know the difference. No `mysqld`, no child processes, just `ephpm` and a `.db` file.

That works great until somebody asks the obvious question. What happens when the box dies? SQLite has no replication.
This is a three-part series about how ePHPm went from "SQLite file plus a sidecar" to replication done entirely inside
the process. The PHP side never changes across all three posts. Everything interesting happens behind litewire.

- Part 1: Replication by sidecar with sqld (this post)
- [Part 2: Adding Turso, one engine in-process](/blog/2026/08/clustered-sqlite-adding-turso/)
- [Part 3: CDC over the cluster channel](/blog/2026/09/clustered-sqlite-cdc-cluster-channel/)

### The first answer: sqld

When I wanted HA in March, the shortest path was [sqld](https://github.com/tursodatabase/libsql). That's libsql-server,
Turso's MIT-licensed SQLite server. It already does primary/replica replication. Replicas pull WAL frames from the
primary over gRPC and forward writes back to it. I pinned v0.24.32 and the first wiring landed on 2026-03-30. The
initial commit ("single node sqld") got reverted 17 minutes later. "clustering sqld" landed that night. That should have
been a hint about how this was going to go.

The thing I care about most in ePHPm is the single binary, so sqld had to be inside it. The release build downloads the
sqld binary. A `build.rs` copies it into `OUT_DIR` so `include_bytes!()` can embed it:

```rust
fn main() {
    println!("cargo::rustc-check-cfg=cfg(sqld_embedded)");
    println!("cargo::rerun-if-env-changed=SQLD_BINARY_PATH");

    if let Ok(path) = std::env::var("SQLD_BINARY_PATH") {
        let src = std::path::Path::new(&path);
        assert!(src.exists(), "SQLD_BINARY_PATH points to non-existent file: {path}");

        let out_dir = std::env::var("OUT_DIR").expect("OUT_DIR not set");
        let dest = std::path::Path::new(&out_dir).join("sqld");
        std::fs::copy(src, &dest).expect("failed to copy sqld binary to OUT_DIR");

        println!("cargo::rustc-cfg=sqld_embedded");
        println!("cargo::rustc-env=SQLD_BINARY_PATH={}", dest.display());
    }
}
```

```rust
#[cfg(sqld_embedded)]
static SQLD_BINARY: &[u8] = include_bytes!(env!("SQLD_BINARY_PATH"));
```

At startup the clustered path does a few steps:

- writes those bytes to a temp file and marks it executable
- spawns it as a child process
- polls its `/health` endpoint
- hands litewire a different backend

Single-node mode used rusqlite in-process. Clustered mode swapped litewire's backend to a Hrana HTTP client pointed at
the local sqld:

```rust
let mut backend = litewire::backend::HranaClient::new(&sqld_http_url);
```

So "single binary" was technically true. It was a single binary with another binary inside it. There were now two modes
running two different engines. rusqlite ran one node and sqld ran a cluster. That split is the seed of everything in
Parts 2 and 3.

![sqld sidecar topology](/img/2026/08/clustered-sqlite-sqld-sidecar-topology.svg)

### sqld doesn't elect anybody

sqld knows how to be a primary or a replica. It does not decide which one it is. That is your problem. ePHPm already had a
gossip layer for the KV store. That's [chitchat](https://github.com/quickwit-oss/chitchat), scuttlebutt-style
anti-entropy with a phi-accrual failure detector. The election went on top of it. The lowest-ordinal alive node wins. The
winner writes a claim into the gossip KV at `sqlite:primary` and refreshes it every 5 seconds. The claim has a 10 second
TTL.

```rust
/// Gossip KV key for the primary node identity.
const PRIMARY_KEY: &str = "sqlite:primary";

/// How often the primary refreshes its claim.
const HEARTBEAT_INTERVAL: Duration = Duration::from_secs(5);

/// TTL for the primary claim. If not refreshed, the key expires and
/// triggers re-election.
const PRIMARY_TTL: Duration = Duration::from_secs(10);
```

The claim value is just `node_id|grpc_addr`. When the role changes, a watcher task restarts sqld with new arguments. On
Unix that means SIGTERM and a wait of up to 5 seconds. If it's still there it gets SIGKILL, and then it spawns again:

```rust
pub async fn restart(&mut self, new_role: SqldRole) -> anyhow::Result<()> {
    self.stop_child().await?;
    self.role = new_role;
    self.child = spawn_child(&self.temp_path, &self.config, &self.role)?;
    Ok(())
}
```

That's a failover. Kill a process and start it again with different flags. It works, in the sense that each piece does
what it says. The rest of this post is the list of ways it didn't work. That list is the reason the next two posts
exist.

### Failure 1: async replication loses data on failover

sqld replicas pull frames from the primary. Say the primary committed a write and acknowledged it to PHP but hadn't
shipped it yet. That write is gone when the primary dies and a replica gets promoted. That's the nature of async
replication. It's not a sqld bug, but it is a real window and it was mine to document.

### Failure 2: no Windows

There is no sqld build for Windows. ePHPm runs on Windows (I spend a lot of time making sure it does). Clustered SQLite
was simply a startup error there:

```rust
#[cfg(target_os = "windows")]
{
    anyhow::bail!(
        "clustered SQLite (sqld sidecar) is not supported on Windows. \
         Use single-node mode (remove [db.sqlite.replication] or set role = \"auto\" \
         without clustering), or run on Linux/macOS/WSL."
    );
}
```

### Failure 3: replica mode had never started

This one stings. In July I built a [two-node comparison fixture](https://github.com/ephpm/turso-cluster-e2e) to measure sqld
against the new Turso path (Part 2). The sqld half wouldn't come up. sqld's CLI rejects `--grpc-listen-addr` combined with
`--primary-grpc-url`, and ePHPm was passing both to replicas. Every two-node bringup had died at spawn. The fix
([#189](https://github.com/ephpm/ephpm/pull/189)) was a few lines:

```rust
match role {
    SqldRole::Primary => {
        cmd.arg("--grpc-listen-addr").arg(&config.grpc_listen);
    }
    SqldRole::Replica { primary_grpc_url } => {
        // sqld rejects --grpc-listen-addr when --primary-grpc-url is
        // set. A replica syncs from the primary and does not serve
        // WAL frames to further replicas in this topology.
        cmd.arg("--primary-grpc-url").arg(primary_grpc_url);
    }
}
```

The fix is boring. What it says isn't. The feature had unit tests, an election, docs, and config. It had never once run
with two nodes. "Implemented" and "exercised" are different words. I had been using them as if they were the same.

### Failure 4: the control plane was open

A security audit pass in late July ([#201](https://github.com/ephpm/ephpm/pull/201)) found three things. All of them were
rooted in the same threat model, a plaintext and unauthenticated cluster control plane:

- **Empty secret meant plaintext.** With `[cluster] secret` unset, gossip and the KV data plane ran unauthenticated. The
  only sign was a startup warning. Any host on the network could forge KV writes: sessions, rate-limit counters, ACME
  keys. Now clustering refuses to start without a secret unless you set `allow_insecure_no_auth = true` on purpose.
- **Forged primary claims.** Replicas dialed whatever `grpc_addr` was in the gossip claim. Inject a claim and you could
  point every replica at an attacker's sqld. Claims are now validated against live gossip membership before anyone
  dials them. The check is by host, since gossip and gRPC use different ports on the same machine.
- **Predictable extraction path.** The embedded binary was written to `temp_dir()/ephpm-sqld-<pid>`, then chmodded and
  executed. Classic symlink race on a shared `/tmp`. It now goes into a fresh `0700` directory from `mkdtemp`, created
  with `O_EXCL`.

None of these are sqld's fault either. They are what happens when you bolt a process manager and an election onto a
gossip layer that was designed for cache keys.

### Failure 5: the write cliff

This is the one that ended it. For v0.6.0 I benchmarked four lanes. They were rusqlite and Turso, each single-node and
clustered. Everything ran on one machine with `--cpus 1` per node and one autocommit INSERT per request. Here's what the
clustered sqld lane did as concurrent writers went up ([issue #217](https://github.com/ephpm/ephpm/issues/217)):

![clustered sqld write throughput by concurrency](/img/2026/08/clustered-sqlite-sqld-sidecar-write-cliff.svg)

| writers | RPS | p50 | p99 | HTTP 500s |
|---|---|---|---|---|
| 1 | 458 | 2.12 ms | 2.99 ms | 0 |
| 2 | 635 | 3.04 ms | 4.63 ms | 0 |
| 4 | **744** (peak) | 4.99 ms | 12.96 ms | 2 |
| 8 | 453 | 9.6 ms | 29.8 ms | 8 |
| 16 | **37** | 20.4 ms | **5.04 s** | 0 |

It peaked at four writers and dropped to half that at eight. At sixteen it fell over. In the release-build run, the c=16
cell completed *zero* requests in both 20 second reps. When there was an error, it looked like this:

```
SQLSTATE[HY000]: General error: 1205 SQLite error: [SQLITE_BUSY] database is locked
```

That is SQLite's single write lock. Add an HTTP round trip from litewire to sqld for every statement and a busy timeout
that runs out. Mostly, though, there were no errors. Look at the 500 column at c=16: zero. The requests hung. The
primary logged nothing, with no busy line and no timeout. A health check watching for 5xx would have called that node
healthy while it served nobody.

Every other lane got faster with more writers. Single-node rusqlite went 648 to 1130 RPS from c=1 to c=16. The
clustered Turso lane went 558 to 876. sqld was the only one that went off a cliff.

### The band-aid that proved the point

For the 0.6.x line I added write admission control ([#222](https://github.com/ephpm/ephpm/pull/222)). It's a semaphore
in front of the Hrana backend, `[db.sqlite.sqld] write_permits`. Reads never take a permit. With the knob unset, c=8 and
c=16 completed nothing. With `write_permits = 1`:

| `write_permits` | c=1 | c=4 | c=8 | c=16 |
|---|---|---|---|---|
| 0 (default) | 442-455 | 118-552 | 0 completed | 0 completed |
| 1 | 415-445 | 561-595 | 527-593 | 551-598 |
| 8 | 444-451 | 232-266 | 0 completed | 0 completed |

(Ranges are across two reps and two separate cluster bring-ups, from the PR.) One permit fixed it. That means one write
at a time. Eight permits sits above the cliff and reproduces it exactly. The best fix I could ship for clustered writes
was to stop doing them concurrently. That is a useful knob and a terrible architecture.

### Where that left things

Count the moving parts in the diagram above. Gossip on UDP 7946. The KV data plane on TCP 7947. sqld gRPC between nodes.
Hrana HTTP between litewire and sqld on every node. A child process per node with its own lifecycle, extracted from bytes
inside the main binary. Two engines depending on whether the config said `[cluster]`. No Windows. And concurrent writes
needed a semaphore set to 1.

A sidecar means another process, another protocol hop, and another failure domain. WAL plus a busy timeout doesn't hold
up under concurrent writers. The same benchmark that drew the cliff above had a fourth lane in it. That was Turso with
in-process change data capture. It sustained 876 RPS at c=16 on the same fixture. That's
[Part 2](/blog/2026/08/clustered-sqlite-adding-turso/).

### Links

- ePHPm: [github.com/ephpm/ephpm](https://github.com/ephpm/ephpm), litewire: [github.com/ephpm/litewire](https://github.com/ephpm/litewire)
- sqld / libsql-server: [github.com/tursodatabase/libsql](https://github.com/tursodatabase/libsql)
- [#189 fix(sqld): omit --grpc-listen-addr for replicas](https://github.com/ephpm/ephpm/pull/189)
- [#201 Harden cluster/sqld control plane](https://github.com/ephpm/ephpm/pull/201)
- [#216 Turso vs SQLite performance, measured](https://github.com/ephpm/ephpm/pull/216)
- [#217 Clustered SQLite (sqld) collapses under concurrent writes](https://github.com/ephpm/ephpm/issues/217)
- [#222 sqld write admission control](https://github.com/ephpm/ephpm/pull/222)
- Two-node fixture: [github.com/ephpm/turso-cluster-e2e](https://github.com/ephpm/turso-cluster-e2e)
