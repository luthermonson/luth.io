---
title: "Rolling Your Own KV Store Instead of Embedding Redis"
description: "Why ePHPm has its own Redis-compatible cache built in instead of running Redis, a bug that hid in it for four months, and how it compares to real Redis on the same machine."
date: 2026-08-17T20:44:00-07:00
tags: [ ephpm, rust, redis, kv, dashmap, caching, performance ]
categories: [ ephpm, rust, databases ]
images:
 - src: img/2026/09/rolling-your-own-kv-store-bench.svg
   alt: "Bar chart of redis-benchmark GET and SET throughput against Redis 7.4, ePHPm v0.5.2 and ePHPm v0.2.0"
   stretch: "none"
draft: false
---

Every PHP deployment I've run eventually grew a Redis. Sessions, the WordPress object cache, Laravel's cache and locks,
rate limiting. [ePHPm](https://github.com/ephpm/ephpm) is supposed to be one binary that replaces nginx + php-fpm +
Redis, so it needed a cache in the box. I didn't want to write one. This post is why I did anyway, how it's built, the
bug that sat in its eviction code from March until last month, and how it holds up against a stock Redis on the same
machine with `redis-benchmark` doing the talking.

### What I tried to embed first

The goal was a library: something I could link into the same process, call from Rust, and expose to PHP. The usual
suspects, and why each one didn't work:

- **Redis** is a server, not a library. It owns its process, runs its own `ae` event loop and expects to be the only
  thing in `main()`. There is no `libredis.a`, and carving one out of the codebase would be a fork I'd maintain forever.
  Its license also moved to RSALv2/SSPL with 7.4 in 2024 (AGPL was added as an option with 8.0), which is a lot of legal
  surface for something I'd be redistributing inside another product.
- **Dragonfly** is the most interesting one technically and the clearest no. It's a standalone C++ server built on
  `io_uring` and its own fiber runtime, which would fight tokio for the same cores. And its BSL 1.1 license permits use "only as
  part of your own product or service, provided it is not an in-memory data store product or service." A PHP server
  whose headline feature includes a clustered in-memory cache is close enough to that line that I'm not going near it.
- **Dragonfly's threading model** is also wrong for this. It's shared-nothing: each thread owns a slice of the keyspace
  and requests get routed to the owning thread. In ePHPm any PHP worker thread needs to read any key, synchronously,
  from inside a request.
- **KeyDB, Garnet, Valkey**: different licenses (Valkey is BSD, which is great), same shape. They're all servers that
  want to own a process.

And all of them are built for millions of ops per second across many gigabytes, persistence, replication, Lua, streams.
What PHP apps actually use Redis for is much narrower: get/set with a TTL, counters, a few hashes, `SETNX` for locks.

### What I built instead

`crates/ephpm-kv` is the whole thing. As of v0.5.2 it's about 5,900 lines including its unit tests: a store, a RESP2 parser and
encoder, a command dispatcher, and a TCP server. The core is two [DashMap](https://docs.rs/dashmap/)s, one for strings and
one for hashes, so `WRONGTYPE` falls out naturally:

```rust
pub struct Store {
    /// The main data map (string values).
    data: DashMap<String, Entry>,
    /// Hash values stored separately to avoid Entry enum complexity.
    hashes: DashMap<String, HashEntry>,
    /// Approximate total memory used by all entries.
    mem_used: AtomicUsize,
    config: StoreConfig,
}
```

DashMap is a sharded concurrent hash map: a power-of-two number of shards (about 4 × CPU count by default), each an `RwLock` around a hashbrown
table. Reads on different shards never touch the same lock, and reads on the same shard share a read lock. That's the
property Dragonfly's model doesn't give me: any thread, any key, no routing.

There are two ways in. PHP code can call native functions (`ephpm_kv_get()` and friends) that go straight from the Zend
engine into this store with no socket at all. And there's a RESP2 listener, off by default, so existing clients
(phpredis, Predis, `redis-cli`, Laravel's Redis driver) work unchanged against `127.0.0.1:6379`. The command set is
deliberately the cache subset: `GET SET SETEX SETNX MGET MSET INCR/DECR[BY] APPEND STRLEN GETSET DEL EXISTS EXPIRE
PEXPIRE TTL PTTL PERSIST TYPE RENAME KEYS DBSIZE FLUSHDB FLUSHALL`, the `H*` hash commands, `PING ECHO SELECT INFO`, and
`SET` with `EX/PX/NX/XX/GET`. No lists, sets, sorted sets, streams, pub/sub, Lua or `MULTI`. If you need a queue, run
a real Redis.

### Expiry: lazy on read, swept in the background

Each entry carries an optional absolute expiry. A read that finds an expired entry deletes it and reports a miss, so
you never see stale data even if nothing else runs:

```rust
pub fn get(&self, key: &str) -> Option<Bytes> {
    let entry = self.data.get(key)?;
    if entry.is_expired() {
        drop(entry);
        self.remove_local(key);
        return None;
    }
    entry.touch(self.now_nanos());
    if entry.compressed {
        decompress_value(&entry.data, self.config.compression.algo)
    } else {
        Some(entry.data.clone()) // Bytes: a refcount bump, no memcpy
    }
}
```

Lazy expiry alone leaks memory for keys that are set and never read again (every abandoned session), so a background
task runs `expire_pass()` periodically. Originally that walked every entry in the map to find expired ones. In July
([#173](https://github.com/ephpm/ephpm/pull/173)) it got a side index, a `DashSet` of keys that have a TTL at all, so
the sweep is proportional to the number of keys that *can* expire rather than all keys. The index is a hint, not the
truth: the authoritative check is still `is_expired()` on the entry, and a stale hint just costs one lookup and gets
pruned.

### Eviction, and the bug that hid in it

There's a `memory_limit` (256MB by default) with Redis's policies: `noeviction`, `allkeys-lru`, `volatile-lru`,
`allkeys-random`. Memory accounting is approximate on purpose (key + value + struct + a fixed overhead per entry), kept
in an atomic, and checked on every write.

An exact LRU needs a linked list threaded through every entry and re-linked on every read, which means a second lock
on the read path. Redis doesn't do that either. It samples a handful of keys and evicts the one that was touched longest
ago. ePHPm does the same: sample 16, evict the oldest, repeat until there's room.

This is what that looked like from March until July:

```rust
// Sample keys and find the one with the oldest last_accessed.
for entry in &self.data {
    if volatile_only && entry.value().expires_at.is_none() {
        continue;
    }
    // ... keep the oldest ...
    count += 1;
    if count >= sample_size {
        break;
    }
}
```

That's not a sample. `&self.data` iterates DashMap from shard 0 every time, so the "random 16" were always the first 16
entries of the first shard or two. Keys in every other shard were never eviction candidates at all, and the same handful
of keys got re-examined on every attempt. Under memory pressure it evicted whatever happened to hash into shard 0, hot or
not. The tests passed because they used small stores where the first 16 entries were the whole store.

The fix ([#197](https://github.com/ephpm/ephpm/pull/197), late July) uses DashMap's `raw-api` to start at a
pseudo-random shard and walk shards from there, taking candidates until it has 16. It's the one `unsafe` block in the
crate (dereferencing raw hashbrown buckets while holding that shard's read lock), and it's bounded: at most
`shard_count + 16` entries visited per victim regardless of store size.

Two other read-path changes in July are worth mentioning because they're the kind of thing you only find with a
profiler:

- **Clone-on-read** ([#146](https://github.com/ephpm/ephpm/pull/146)). `get` returned `Vec<u8>`, a full copy of the value,
  and the RESP encoder then boxed it again. Values are now `bytes::Bytes`, so a `GET` is a refcount bump from the store
  to the socket. The RESP parser also slices bulk strings out of the read buffer with `split_to().freeze()`, so a `SET`
  payload goes from the network buffer into the store without a copy either.
- **The LRU touch** ([#146](https://github.com/ephpm/ephpm/pull/146), [#173](https://github.com/ephpm/ephpm/pull/173)).
  `last_accessed` was an `Instant`, which meant every `GET` took the shard's *write* lock to update it, so concurrent
  reads of the same shard serialized. It's now an `AtomicU64` of nanoseconds since a store-level anchor, updated with a
  relaxed store under the read lock. Then it got coarser: the touch only writes if the stored time is more than 100ms old.
  Hot keys read thousands of times per second were bouncing that cache line between cores on every read; now a read is
  a load, and the line stays shared.

And one that wasn't in the store at all: every RESP connection, and the DB proxy, was missing `TCP_NODELAY`
([#161](https://github.com/ephpm/ephpm/pull/161)). Nagle plus delayed ACK added a ~44ms stall on certain round trips.
The HTTP accept loop already set it on every accepted socket, which is why nobody noticed on the web side.

### Against a real Redis

I wanted to know whether any of this was competitive or just "good enough for PHP," so I pointed `redis-benchmark` at
three servers on the same box: Redis 7.4.11 in a container, the v0.5.2 release of ePHPm (all of the above), and the v0.2.0
release from the end of June (before the July read-path work), each with the RESP listener enabled. 50 clients, 256-byte
values, 100,000 random keys, 1,000,000 requests per test, once without pipelining and once with 16 commands per
pipeline, three runs each:

![Bar chart of GET and SET throughput for Redis 7.4, ePHPm v0.5.2 and ePHPm v0.2.0](/img/2026/09/rolling-your-own-kv-store-bench.svg)

| ops/s (median of 3) | Redis 7.4.11 | ePHPm v0.5.2 | ePHPm v0.2.0 |
|---------------------|-------------:|-------------:|-------------:|
| GET, no pipelining  | 51,929 | 235,238 | 95,211 |
| SET, no pipelining  | 51,929 | 235,239 | 64,491 |
| GET, pipeline 16    | 547,345 | 1,175,641 | 17,975 |
| SET, pipeline 16    | 547,345 | 1,175,779 | 17,582 |

Before anyone screenshots that and posts "Rust cache 4x faster than Redis": it isn't, not in the way that headline
means. Redis executes commands on one thread. ePHPm handles RESP connections on a multi-threaded tokio runtime with 32
worker threads on this box, against a sharded map. With 50 concurrent connections and a client that can push hard enough
(`redis-benchmark --threads 4`), the multi-threaded server wins on throughput, which is what you'd expect from 32 threads
against one. Two more runs make the shape clearer:

| | Redis 7.4.11 | ePHPm v0.5.2 |
|---|---:|---:|
| 1 client, GET, p50 latency | 0.079 ms (10,934 ops/s) | 0.079 ms (11,632 ops/s) |
| 50 clients, single-threaded benchmark client, GET | 71,628 ops/s | 67,096 ops/s |

One connection at a time, they're the same speed: the round trip over loopback dominates, and neither server is the
bottleneck. With a single-threaded benchmark client, the client is the bottleneck and Redis is a little ahead. So the
honest summary is: per command, ePHPm's store is in the same league as Redis, and under concurrent load it scales
across cores where stock Redis doesn't (Redis's I/O threads would close some of that gap; I ran it with defaults).
That's plenty for a PHP cache.

The more interesting comparison is v0.2.0 to v0.5.2, same server code path, a month apart. Unpipelined throughput went
up 2.5x to 3.6x. Pipelined throughput went from about 18,000 ops/s, which is *slower* than the same version without
pipelining, to 1.18 million. I haven't bisected which of the July changes fixed the pipelining collapse (it predates
the copy removal, the atomic LRU touch, the coarse clock and `TCP_NODELAY`, and any of them could interact with
batched writes), but it's a good reminder that "works in the tests" and "works under a pipelining client" are different
claims.

### What it isn't

It's a cache. There's no persistence, so a restart is an empty cache (that's what a cache is). The memory accounting is an
estimate. There's no `MULTI`, no Lua, and no data types beyond strings and hashes. Every one of those is a deliberate
"not yet," and the RESP listener means the exit is cheap: if an app outgrows it, point the Redis client at a real Redis
and nothing else changes.

What it is: no second daemon, no second config file, no extra port open between PHP and its cache, and when PHP calls the
native functions instead of RESP there's no socket at all. That last path is a whole topic of its own.

### How I measured

- AMD Ryzen 9 5950X (16C/32T), 128 GB (62 GiB visible to WSL2), Windows 11 with WSL2. ePHPm ran in the Ubuntu WSL distro; Redis
  (`docker.io/library/redis:7-alpine`, 7.4.11, `--save '' --appendonly no`) and `redis-benchmark` ran in Podman with host
  networking, which on WSL2 shares the network namespace with the Ubuntu distro, so everything is loopback.
- ePHPm `v0.5.2+php8.4.23` and `v0.2.0+php8.4.22` release binaries, `[kv] memory_limit = "2GB"`, compression off,
  `[kv.redis_compat] enabled = true`.
- `redis-benchmark -t set,get -n 1000000 -r 100000 -d 256 -c 50 -P 1 --threads 4 --csv` and the same with
  `-P 16 -n 10000000` (`-n 1000000` for v0.2.0, which would otherwise take ten minutes), three runs per configuration,
  median reported. The two extra rows are `-c 1 -t get -n 100000` and `-c 50` without `--threads`, one run each.
- Same machine for client and servers, so the client competes for CPU. Redis is single-threaded for command execution;
  ePHPm runs its RESP connections on a multi-threaded tokio runtime. That's an honest description of how each runs by
  default, not a tuned comparison.

### Links

- [ePHPm](https://github.com/ephpm/ephpm), the store is `crates/ephpm-kv`
- [#146](https://github.com/ephpm/ephpm/pull/146) Bytes values and atomic LRU touch,
  [#161](https://github.com/ephpm/ephpm/pull/161) TCP_NODELAY,
  [#173](https://github.com/ephpm/ephpm/pull/173) coarse LRU clock and TTL index,
  [#197](https://github.com/ephpm/ephpm/pull/197) unbiased sampled eviction
- [DashMap](https://github.com/xacrimon/dashmap)
- [Redis key eviction](https://redis.io/docs/latest/develop/reference/eviction/) (the sampled LRU this copies)
- [Dragonfly license](https://github.com/dragonflydb/dragonfly/blob/main/LICENSE.md)
