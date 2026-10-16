---
title: "Clustered ACME in Production: Three Bugs That Took TLS Down"
description: "Moving wildcard TLS onto a live 3-node ePHPm cluster found a leader election that wasn't, a certificate only two nodes could see, and an HTTP port speaking TLS."
date: 2026-09-02T19:30:00-07:00
tags: [ rust, ephpm, tls, acme, letsencrypt, clustering ]
categories: [ ephpm, security, rust ]
images:
 - src: img/2026/09/clustered-acme-three-bugs-cover.svg
   alt: "Diagram comparing a sharded KV write that leaves one of three nodes without the certificate to a broadcast write that reaches all three"
   stretch: "none"
draft: false
---

[ePHPm](https://github.com/ephpm/ephpm) gets its own certificates from Let's Encrypt. In a cluster only one node should
be talking to the CA. The nodes elect an ACME leader over the shared KV store. The leader orders the cert, and everybody
else installs it from KV. That all had unit tests and had worked on single nodes for months. Then I tried to move
wildcard TLS termination for our preview environment off a separate proxy and onto the 3-node cluster itself. In about
two days it found three separate bugs that each took TLS down. None of them showed up in a unit test. All of them are
the kind of thing you only learn by putting the feature on real machines.

### The setup

Preview deployments get a subdomain each, something like `pr-123.preview.example.com`. Issuing a cert per subdomain
would burn through Let's Encrypt's 50 certificates per registered domain per week in no time. So the plan was one
wildcard, `*.preview.example.com`. TLS-ALPN-01 can't validate a wildcard, because there's no single hostname for the CA
to connect to. That meant DNS-01:

- publish a `_acme-challenge` TXT record
- wait for it to propagate
- tell the CA you're ready

That lane landed first ([#411](https://github.com/ephpm/ephpm/pull/411)), built on `instant-acme`. It reused the
existing clustered pieces from the TLS-ALPN-01 lane: the `acme:leader` election and the KV cert distribution. Reusing
the hard, correctness-critical parts instead of writing them twice is the right call. It also means you inherit their
bugs.

Three nodes, a Linode NodeBalancer in front, Let's Encrypt **staging** first to protect the production rate limit.

### Bug 1: leader election by start time

All three nodes logged that they'd acquired ACME leadership. Then they took it back from each other, over and over:

```
node-1  acquired ACME leadership node_id="1787985979173-..."
node-2  acquired ACME leadership node_id="1787986064600-..."
node-3  acquired ACME leadership node_id="1787986130867-..."
node-1  taking over ACME leadership (lower node id wins the tie-break) ...
node-2  taking over ACME leadership (lower node id wins the tie-break) ...
```

The election's tie-break is "lowest node id wins", which is fine. The problem was the id. The ACME code minted its own
per-process identity:

```
1787985979173-22749-173895959
└─ epoch millis ─┘ └pid┘ └random┘
```

With an epoch-millis prefix, "lowest id wins" really means "whichever process started earliest". That reshuffles on
every restart. While nodes are rolling, leadership bounces around. The cycle on the cluster was roughly 70 seconds. A
DNS-01 order spends about 60 seconds just waiting on TXT propagation. So no node held leadership long enough to finish
an order, and **two of three nodes never got a certificate**. The third only had one because it loaded an old cert
from its disk cache.

It also finally explained something I'd poked at before and left unexplained: "~7 minute leader flapping" in the logs.
The cadence came from restart timing, not from any timer. That's exactly why looking for a timer never found it.

The fix ([#418](https://github.com/ephpm/ephpm/pull/418)) uses the configured `[cluster] node_id`. That's the same
stable identity gossip already advertises. The generated form stays only as a fallback for a node without a configured
id. That's a single-node deployment where there's nothing to elect against. One subtle bit: a blank or whitespace-only
`node_id` also falls back to the generated form. Otherwise every node would collapse onto the same empty identity, and
every node would believe it holds the same claim.

### Bug 2: the certificate only two nodes could see

With a stable leader, node-1 issued a valid certificate and served it. Nodes 2 and 3 never installed it. Ten minutes
later they were still failing handshakes with `tlsv1 alert access denied`, because they had no certificate at all.

The cert was written with `Store::set`, which in clustered mode routes by value size. Small values ride the gossip layer
to every node. Large values go to a sharded tier and land on `replication_factor` nodes, which defaults to 2. The issued
chain was **5140 bytes** and `small_key_threshold` is **4096**. So the certificate got sharded like a cache entry. The
read side used the plain, local-only `Store::get`, which has no "fetch it from the owner" path. On a 3-node cluster with
a replication factor of 2, one node could never see the certificate.

![Where the certificate went](/img/2026/09/clustered-acme-three-bugs-cover.svg)

The obvious knob is raising `small_key_threshold`, and it's the wrong one. The small tier is
[chitchat](https://github.com/quickwit-oss/chitchat) gossip over **UDP**, so a multi-kilobyte payload can't ride it.
That's the whole reason the threshold exists. The bug wasn't the threshold. It was the assumption that "too big for
gossip" means "shard it". Sharding is a capacity optimization. It's the wrong routing for state every node needs
locally, and every node terminates TLS with this certificate.

So [#417](https://github.com/ephpm/ephpm/pull/417) added two things, and both are load-bearing:

1. **A broadcast write tier** (`Store::set_broadcast`) that writes to every alive node regardless of size. Peer
   writes are awaited rather than fire-and-forget. A certificate gets written a few times a year, so cluster-wide
   read-your-writes matters a lot more than write latency. A partial delivery logs a warning instead of failing
   silently.
2. **A fetch-through read** (`Store::get_cluster`). On a local miss, it falls back to the key's replica set over the
   data plane. Broadcast only reaches nodes that are alive at write time. Without the read fallback, a node that joined
   after issuance would wait up to 60 days for the next renewal to get a certificate.

The TLS-ALPN-01 lane had the identical defect on its own cert and account cache, so it got the same fix. ACME account
material moved to the broadcast tier too. Every node needs the same ACME identity. Otherwise a promoted follower
registers a second account with its own rate-limit budget.

The test for this is the one I like best in the batch. It spins up three real gossip nodes on `127.0.0.x` with a real
TCP data plane. It writes a 5140-byte payload (the size from the incident) through the same `Store` calls the ACME code
uses. Then it asserts the value is in the *raw local store* of the node the sharded tier would skip. Reverting
`set_broadcast` to the old routing makes it fail with `left: None` on that node. That's the live outage reproduced in a
test.

#### The same bug from another direction

A follow-up showed up the moment the cluster was restarted as a whole. Lowest node id wins. The leader comes up holding
a still-valid cert in its disk cache, seeds from disk, and never orders anything. Broadcasts only happened at
*issuance*, so it never published. Followers found KV empty, and two of three nodes served nothing on :443 again.
[#420](https://github.com/ephpm/ephpm/pull/420) republishes the cached cert to KV on the leadership-acquisition
transition. It's a no-op when KV already has it. After the redeploy, node-1 logged `re-published cached certificate to
the cluster on leadership acquisition` and all three nodes served the staging wildcard.

That same PR fixed the next landmine before it went off. The DNS-01 cert cache keys in KV and the files on disk were
keyed only by domain set, never by ACME directory. Flip `staging = false` and the staging cert is sitting under the
exact key the production reader looks up. The cache is read first and nobody checks the issuer. The nodes would have
kept serving an untrusted staging cert forever and never ordered a production one. Every cache entry is now namespaced
by a hash of the directory URL.

It wasn't quite the whole story. When I flipped to production, the cluster still came back with staging certs. The
account *credentials* on disk were a single `account.json`. After the flip the production KV key missed, so the code
fell back to disk and loaded the staging account. `instant-acme` then reconnected to the directory URL embedded in those
credentials. That was [#421](https://github.com/ephpm/ephpm/pull/421): `account-<directory_hash>.json`. It was the third
facet of the same bug. The order went to staging, so no production rate limit was spent finding it.

### Bug 3: the HTTP port that spoke TLS

This one had already been misdiagnosed once. The config was `[server] listen = "0.0.0.0:8080"` plus
`[server.tls] listen = "0.0.0.0:443"`. The NodeBalancer health-checks plain HTTP on 8080, and the health check failed.
That pulled nodes out of rotation and sent 503s to users. The earlier write-up called it "port 8080 connection refused".

It wasn't refused. Sending a plaintext `GET` with `nc` got this back:

```
15 03 03 00 02 02 32
```

That's a TLS alert record. `0x15` is the alert content type and `03 03` is TLS 1.2, then a fatal `decode_error`. The
port was listening and accepting the whole time. It was speaking TLS. Clients reported a connection-level failure, so
the port looked dead.

The cause was in `dispatch_main_connection`. Plain HTTP on the main listener was gated on
`has_tls_listener && redirect_http`, and `redirect_http` defaults to `false`. So a separate TLS listener without a
redirect fell straight through to the TLS-mode match. That wrapped the main listener in TLS too. The code contradicted
its own documented contract ("when set, `server.listen` serves HTTP and this address serves HTTPS"). That contract was
already correct in two places. Now whenever a separate TLS listener is bound, the main listener serves plain HTTP.
`redirect_http` only decides whether that's a 301 or a normal response. There's no TLS-mode arm on that path anymore,
so manual and ACME modes can't diverge again.

The tests drive the real `dispatch_main_connection` over a real socket and assert on the first byte written back. That's
`H` for `HTTP/1.1`, or `0x16`/`0x15` for a TLS record. It's the same signal the live diagnosis used.

### A couple of earlier ones

While I was in here, two older TLS bugs belong in the same list.

Once the certs were on the nodes, every browser got a 404 or a redirect loop on multi-tenant sites. `curl` worked fine.
Browsers negotiate HTTP/2 over TLS. HTTP/2 carries the host in the `:authority` pseudo-header with an absolute-form
request URI, not a `Host` header. ePHPm only looked at `Host`, which left three problems:

- vhost resolution defaulted to `localhost`
- `$_SERVER['HTTP_HOST']` was empty
- `REQUEST_URI` was `https://host/path`

WordPress turned that into a canonical redirect to a URL with the host glued in twice.
[#414](https://github.com/ephpm/ephpm/pull/414) synthesizes a `Host` header from the authority at ingress. It also emits
`REQUEST_URI` in origin form for every protocol version.

And earlier in August, manual cert/key TLS panicked at startup in v0.6.0 and v0.6.1
([#243](https://github.com/ephpm/ephpm/pull/243)). The workspace pinned rustls to `ring`. Meanwhile `rustls-acme`,
`rcgen` and `hyper-rustls` enabled `aws-lc-rs`. Cargo unions features, so both providers got compiled in. rustls then
refused to pick a process default. Nobody noticed because ACME mode always passes a provider explicitly. Every unit test
in `tls.rs` also called `install_default()` first, which masked the panic for the whole test binary. That's since been
converged on a single `aws-lc-rs` provider ([#368](https://github.com/ephpm/ephpm/pull/368)).

### What I took away

Every one of these passed its unit tests. The election was "correct" given its inputs. The KV write was "correct" for a
cache entry. The listener was "correct" for one config shape. They broke on assumptions that only a real cluster
exercises: restart timing, value sizes, a load balancer health check that speaks plain HTTP.

The other takeaway is about staging. Running the whole thing against Let's Encrypt staging first made it cheap to find
the cache-collision bugs. The flip to production was where they'd have burned a week of wildcard rate limit. Do the
staging run, and then treat the flip itself as a separate thing to test.

### Links

- DNS-01 wildcard lane: [ephpm #411](https://github.com/ephpm/ephpm/pull/411)
- Leader election on the configured node id: [ephpm #418](https://github.com/ephpm/ephpm/pull/418)
- Plain HTTP listener + broadcast cert distribution: [ephpm #417](https://github.com/ephpm/ephpm/pull/417)
- Republish on promotion + directory namespacing: [ephpm #420](https://github.com/ephpm/ephpm/pull/420), [#421](https://github.com/ephpm/ephpm/pull/421)
- HTTP/2 `:authority`: [ephpm #414](https://github.com/ephpm/ephpm/pull/414)
- Two rustls crypto providers: [ephpm #243](https://github.com/ephpm/ephpm/pull/243)
- [Let's Encrypt rate limits](https://letsencrypt.org/docs/rate-limits/) and [staging environment](https://letsencrypt.org/docs/staging-environment/)
