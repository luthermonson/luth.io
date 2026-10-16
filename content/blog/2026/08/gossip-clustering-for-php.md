---
title: "Gossip-Based Clustering for PHP Apps"
description: "How ePHPm servers find each other, share data and handle a server joining or leaving. It also measures how fast a write reaches the other servers and covers a bug I found while measuring it."
date: 2026-08-04T20:52:00-07:00
tags: [ ephpm, rust, php, clustering, gossip, distributed systems ]
categories: [ ephpm, rust ]
images:
 - src: img/2026/08/gossip-clustering-php-cover.svg
   alt: "Three ePHPm nodes: small values replicated to every node over gossip, a large value stored on its owner and one replica over a TCP data plane"
   stretch: "none"
draft: false
---

Most PHP developers never touch clustering primitives. That's by design. PHP is share-nothing. Each request starts
from zero, and anything shared lives somewhere else. Sessions go in Redis. Cache goes in Redis or Memcached. Locks go
in Redis. When you run two app servers, the "cluster" is really two stateless boxes pointed at the same Redis. That
works until Redis is the thing you're trying not to run.

[ePHPm](https://github.com/ephpm/ephpm) has its own KV store in the same process as PHP. On one node that's
straightforward. On three nodes behind a load balancer, a session written on node 1 has to be readable on node 3. That
means the nodes have to know about each other and agree on where data lives. This post is how that works as of
[v0.5.2](https://github.com/ephpm/ephpm/releases/tag/v0.5.2), including the parts I'm not happy with yet.

### No coordinator

The first decision was what *not* to use. etcd and Consul are the standard answers for membership, and they're both
very good. They're also another cluster to deploy, secure, back up and upgrade. That's precisely the operational
weight ePHPm exists to remove. Raft-based anything also means a quorum. A two-node ePHPm deployment is common for
small sites, and it can't have a meaningful one.

So membership is gossip. Every node runs Quickwit's [chitchat](https://github.com/quickwit-oss/chitchat) crate. It's
often lumped in with SWIM, and ePHPm's own source comments call it that. But it's actually scuttlebutt-style
anti-entropy. Each node owns a small key/value state. Nodes periodically exchange digests of what versions they've
seen and send each other only the deltas. Liveness comes from a phi-accrual failure detector on heartbeats rather than
SWIM's ping/ping-req probing. For ePHPm's purposes the important properties are the same:

- There's no leader.
- Any node can be a seed.
- A node that disappears gets marked dead without anyone coordinating it.

Starting it is a config block:

```toml
[cluster]
enabled = true
bind = "10.0.0.11:7946"
join = ["10.0.0.11:7946"]
secret = "a-long-random-string"

[cluster.kv]
small_key_threshold = 512   # bytes: <= rides gossip, > goes to the data plane
replication_factor = 2
data_port = 7947
```

and wrapping chitchat is about thirty lines:

```rust
let chitchat_config = ChitchatConfig {
    chitchat_id: ChitchatId::new(node_id.clone(), generation_id, listen_addr),
    cluster_id: config.cluster_id.clone(),
    gossip_interval: Duration::from_secs(1),
    listen_addr,
    seed_nodes: config.join.clone(),
    failure_detector_config: FailureDetectorConfig::default(),
    marked_for_deletion_grace_period: Duration::from_secs(60),
    catchup_callback: None,
    extra_liveness_predicate: None,
};

let handle = if encrypted {
    let transport = EncryptedUdpTransport::new(&config.secret);
    spawn_chitchat(chitchat_config, vec![], &transport).await
} else {
    spawn_chitchat(chitchat_config, vec![], &UdpTransport).await
}
.context("failed to spawn chitchat gossip")?;
```

The `generation_id` is the start time in seconds. A node that restarts with the same `node_id` is recognized as a new
incarnation instead of a zombie.

### Gossip is authenticated or it doesn't start

Gossip over plaintext UDP means anyone who can send a packet to port 7946 can join your cluster and write keys into
it. When `secret` is set, both the gossip datagrams and the TCP data plane are sealed with ChaCha20-Poly1305. Each
plane gets its own key, derived from the secret via HKDF-SHA256. The info strings are `"ephpm-gossip-v1"` and
`"ephpm-kv-data-v1"`. Datagrams that fail authentication are dropped silently. A node with the wrong secret is simply
invisible.

chitchat's transport trait sends and receives typed messages with serialization inside the socket. There's no
byte-level hook to wrap. So the encrypted transport owns its own UDP socket and mirrors chitchat's `UdpTransport`. It
seals after serialization and opens before deserialization.

As of late July an empty secret is a hard startup error when clustering is on
([#201](https://github.com/ephpm/ephpm/pull/201)). You can still opt out with `allow_insecure_no_auth = true`, and you get
a loud warning for it. Failing closed is the right default for something that replicates session data.

### Two tiers, because gossip is the wrong place for big values

chitchat state is `String -> String`. Every node gets a copy of every other node's state. That's perfect for small
things everybody needs. Think sessions, counters, rate-limit buckets and feature flags. It also fits the "this vhost's
OPcache version changed" key that `ephpm deploy` writes. It's a terrible place for a 40 KB rendered page fragment.
That would be replicated to every node, encoded into gossip deltas and resent to anyone catching up.

So `ClusteredStore::set` splits on size:

```rust
pub async fn set(&self, key: String, value: Vec<u8>, ttl: Option<Duration>) -> bool {
    if value.len() <= self.config.small_key_threshold {
        // Gossip tier: replicate to all nodes.
        self.cluster.gossip_set(&key, &value, ttl).await;
        if self.config.hot_key_cache {
            self.maybe_bump_hot_version(&key).await;
        }
        return true;
    }

    // Large value: write to the replica set (primary + N-1 peers).
    let ok = self.set_large(&key, &value, ttl).await;
    if ok && self.config.hot_key_cache {
        self.maybe_bump_hot_version(&key).await;
    }
    ok
}
```

**The gossip tier** stores each value in the writing node's chitchat state as `"{expiry_ms}:{write_ms}:{base64}"`. A
read on any node scans its own state and every live peer's for that key. It takes the entry with the highest
`write_ms`. If two nodes write the same session at nearly the same time, the later write wins everywhere once gossip
settles. Deletes are a tombstone value (`"TS:{write_ms}"`) rather than chitchat's own delete. chitchat doesn't fire
its change subscription on a real delete, so peers would never hear about it. That was
[#165](https://github.com/ephpm/ephpm/pull/165) in July. Before it, a delete on one node quietly didn't propagate.

`write_ms` is the origin node's wall clock. That's a real limitation. With enough clock skew between nodes, "last
writer wins" means "writer with the fastest clock wins." Run NTP. For sessions and caches that's an acceptable model.
It wouldn't be for a bank ledger, and nobody should put one in a PHP cache.

**The data plane** is a small length-prefixed TCP protocol on port 7947. Each message is `[op][key_len][key]`, plus
`[value_len][value]` for a SET. Large values are written to an owner node and replicated to `replication_factor - 1`
more. `ClusteredStore::get` checks gossip, then the local store, then a local hot-key cache. Finally it asks the
owner, falling back to the replicas in order if the owner is down:

```rust
for node in self.replica_nodes(key).await {
    if node.id == self_id {
        continue; // we'd have served it from the local store already
    }
    let Some(addr) = node_data_addr(&node, self.config.data_port) else { continue };
    match crate::kv_data_plane::fetch_remote(addr, key, self.cipher.as_deref()).await {
        Ok(Some(value)) => {
            self.track_remote_fetch(key, &value);
            return Some(value);
        }
        Ok(None) | Err(_) => continue,
    }
}
```

`track_remote_fetch` is the hot-key cache. If the same node fetches the same remote key five times in ten seconds, it
keeps a local copy for up to 30 seconds. The owner publishes a version counter for hot keys through gossip. A bump
invalidates every cached copy. That handles the "every node hammers the owner for the homepage fragment" case without
a client-side cache library.

### Where a key lives: hash % nodes

Here's the part I'm least happy with. The owner of a large key is:

```rust
let start = (key_hash as usize) % n;   // n = number of alive nodes, sorted by node id
// replicas: the next factor-1 nodes after start, wrapping around
```

The docs and a couple of my own notes call this a "hash ring," and it isn't one. It's modulo over the sorted list of
alive nodes. That's simple and correct while membership is stable. When membership changes, it moves almost
everything. I measured how many of 200,000 keys change owner when one node joins. I compared that rule to a textbook
consistent-hash ring with 160 virtual nodes per node:

![Line chart: percentage of keys that move when one node joins, modulo vs consistent hashing vs the ideal 1/n](/img/2026/08/gossip-clustering-php-remap.svg)

| Join | `hash % n` | ring, 160 vnodes | ideal (1/n) |
|------|-----------:|-----------------:|------------:|
| 2 → 3 | 66.8% | 30.8% | 33.3% |
| 3 → 4 | 75.0% | 23.0% | 25.0% |
| 4 → 5 | 79.9% | 19.7% | 20.0% |
| 8 → 9 | 89.0% | 10.0% | 11.1% |

Going from three nodes to four, three quarters of the large keys get a new owner. There's another honest caveat in the
code. Replication happens only at write time, and there's no anti-entropy or rebalancing for the data plane. Combine
the two and a membership change means most large values look like cache misses until they're rewritten. For a cache
that's a performance event, not data loss. The gossip tier isn't affected at all, because every node has every small
value. But it's the wrong behavior for a rolling deploy, and a ring is not much code. That one's on the list.

There's a second, smaller trap in the same function. `hash_key` uses Rust's `DefaultHasher`. The standard library
explicitly doesn't promise to keep its algorithm stable across releases. Every node in a cluster today runs the same
binary, so they agree. Two ePHPm versions built with different Rust toolchains might not agree in the middle of a
rolling upgrade. A fixed hash (FNV or xxHash) costs nothing and removes the question.

### What a write actually costs

The other question everyone asks about gossip is "how long until node 3 sees it?" I ran three nodes from the v0.5.2
release binary on one machine, on separate ports with encryption on. I wrote a 100-byte value with `SET` on node 1 and
polled `GET` on node 3 every millisecond until it showed up. I did that 60 times, with a random 0.1 to 1.1 second gap
between writes:

![CDF of time from SET on node 1 to visibility on node 3 for 60 writes](/img/2026/08/gossip-clustering-php-convergence.svg)

| | ms after `SET` returned |
|---|---:|
| fastest | 1.7 |
| median | 385 |
| p90 | 789 |
| slowest | 833 |

That's gossip doing exactly what it's configured to do. Every node gossips with a peer once per `gossip_interval`,
which is 1 second. A write becomes visible on the next exchange that carries it. That's anywhere from "right now" to
just under a second later. The spread is close to uniform across that window, which is what you'd expect. Eventually
consistent, on the order of a second.

The random gap matters, by the way. My first run used a fixed 300ms pause between writes. All but one of 40 samples
came back at almost exactly 700ms. The loop period had locked onto the 1 second gossip tick. I was measuring my own
benchmark's phase, not the cluster. If you measure anything that runs on a timer, randomize when you poke it.

In practice, say PHP writes a session on node 1 and the load balancer sends the very next request to node 3. That
request can miss it. Sticky sessions on the load balancer are still a good idea for login flows.

### The large-value path has a gap

I meant to time large values the same way and couldn't. The reason is worth writing down. Two things got in the way.

The first is my test rig. A node finds a peer's data plane at the peer's gossip IP plus *its own* `data_port`. So
every node in a cluster has to use the same port, which you can't do with three processes on one IP. That's a fine
assumption for real deployments with one node per host. It's just not one for a laptop.

The second is not my rig. Reading the code to figure out the first problem, I found that v0.5.2's RESP listener and
PHP's native `ephpm_kv_*` functions read the node's local `Store` directly. Writes go through a replication hook into
`ClusteredStore`, so large values do get written to their owner and replica. But the read-through above lives in
`ClusteredStore::get`, and neither RESP nor PHP calls it. A large value is readable on the nodes that hold a copy.
Everywhere else it's a cache miss. With `replication_factor = 2` on a three-node cluster, that's one node in three
missing every large key it doesn't own.

For a cache that's a miss, not corruption. But it means the two-tier picture above is accurate for writes and only
half true for reads. Wiring a read-through into `Store` is the next thing on the clustering list, ahead of the ring.
That's a "fetch from the cluster on miss" hook next to the existing write hook.

### Why this is unusual for PHP

None of this is new in distributed systems. What's unusual is having it inside the PHP runtime. As far as I can tell,
FrankenPHP, RoadRunner and Swoole all leave anything shared between nodes to an external store like Redis or
Memcached. With ePHPm, PHP code calls `ephpm_kv_set()` and the cluster handles the rest. It can also talk RESP to
localhost with an existing Redis client. There's no extra daemon and no new protocol for the app to learn. The same
gossip channel already carries cluster-wide OPcache invalidation for deploys. It's also what the clustered SQLite work
is being built on.

So here's the list, in order:

1. A read-through hook so RESP and PHP reads can reach the owner.
2. A real ring and a stable hash.
3. Some form of handoff so a membership change doesn't look like a cache flush.

### How I measured

- Remap numbers: a small Go program with 200,000 keys named `session:<i>`. Hashing is FNV-1a with a splitmix64
  finalizer. Nodes are named `node-00` to `node-08` and sorted by name, like ePHPm sorts node ids. The modulo rule is
  ePHPm's. The ring is a standard 160-vnode ring. The point is the shape, which doesn't depend on the hash function.
- Convergence: ePHPm v0.5.2 (`linux-x86_64`, PHP 8.4.23) as three processes on one Ubuntu WSL2 box (Ryzen 9 5950X).
  Gossip ran on loopback with `secret` set and default `[cluster.kv]` settings. Nodes 2 and 3 were seeded from node 1.
  A small Go RESP client wrote to node 1 and polled node 3 every millisecond, with a random 100 to 1100ms gap between
  writes. Loopback has no real network latency, so on a real network add a round trip or two.

### Links

- [ePHPm](https://github.com/ephpm/ephpm), clustering lives in `crates/ephpm-cluster`
- [chitchat](https://github.com/quickwit-oss/chitchat)
- [Scuttlebutt paper: Efficient Reconciliation and Flow Control for Anti-Entropy Protocols](https://www.cs.cornell.edu/home/rvr/papers/flowgossip.pdf)
- [The φ Accrual Failure Detector](https://doi.org/10.1109/RELDIS.2004.1353004)
- [#165](https://github.com/ephpm/ephpm/pull/165) replicate deletes and TTL updates, [#201](https://github.com/ephpm/ephpm/pull/201) mandatory cluster secret
