---
title: "The Pragmatism of ePHPm - Every Decision Has a \"Because\" Behind It"
description: "ePHPm hit v0.1.0 this week. It isn't a rewrite-it-in-Rust project. Here's what I built, what I borrowed, what I sent upstream, and what I deliberately haven't built."
date: 2026-06-21T19:05:00-07:00
tags: [ ephpm, php, rust, open source, architecture ]
categories: [ ephpm, php, rust ]
images:
 - src: img/2026/06/ephpm-pragmatism-cover.svg
   alt: "ePHPm in the center with six boxes: Rust and C wrapper, KV store, chitchat gossip, static-php-cli upstream fixes, litewire, and a dashed box of features not built yet"
   stretch: "none"
draft: false
---

[ePHPm](https://github.com/ephpm/ephpm) tagged [v0.1.0](https://github.com/ephpm/ephpm/releases/tag/v0.1.0) this
week, three months after the first commit. Before the feature list grows any further I want to write down how
decisions get made on it, mostly for myself, because the easiest way to ruin a project like this is to let it turn into
a checklist of things other projects have.

The pitch is simple: one binary that runs a PHP app with fewer moving parts than nginx + php-fpm + Redis + MySQL. Every
feature has to trace back to that. If I can't finish the sentence "this exists because..." with a real problem from
running PHP in production, it doesn't go in.

### Rust, because I measured cgo

This isn't a "rewrite it in Rust" project. I write Go for a living and I'd have been faster in Go. I picked Rust because
PHP calls *back into* its host constantly (every `echo`, every header, every cache read once the KV functions exist),
and I wanted that direction to be a plain function call. When I [actually measured it](/blog/2026/04/why-rust-for-php-embedding/)
the cgo tax turned out to be a lot smaller than my own design doc claimed (about 24ns per call on Go 1.26, not 200), and
I said so. The decision survived the correction because the reasons were never only that one number: one runtime in the
process, no GC next to Zend, and no second project to upstream into before I can change how a request is dispatched.

The part I didn't get to pick is the C wrapper. PHP's error handling is `setjmp`/`longjmp`, and that has to live in C
regardless of what language is on the other side. So the answer to "is it Rust?" is "it's Rust plus about 1,900 lines of
C that I'd rather not have."

### A KV store, because there was nothing to embed

I did not want to write a KV store. Building databases isn't the fun part of this for me. But a PHP app server without a
cache just means the user runs Redis next to it, and then the single binary pitch is gone.

So I went looking for something to embed:

- **Redis** isn't a library. It's a server that wants to own a process and its event loop. There's no `libredis.a`.
  (Its licensing also changed in 2024, which made me even less interested in building a product on top of it.)
- **Dragonfly** is also a standalone server, and its BSL 1.1 license explicitly excludes use in an "in-memory data store
  product or service." A PHP server with a built-in clustered cache is close enough to that line that I'm not going near it.
- **KeyDB, Garnet, Valkey**: different licenses, same shape. Servers, not libraries. Dragonfly's shared-nothing model
  (each thread owns a slice of the keyspace) is also the opposite of what ePHPm needs, where any PHP thread has to be
  able to read any key.

What PHP apps actually use Redis for is caching, sessions and counters. Strings, a few hashes, TTLs. That's a
`DashMap`, a RESP2 parser so existing clients work, lazy expiry plus a sweeper, and an eviction policy. A few thousand
lines of Rust instead of a second daemon. It's not Redis and it doesn't pretend to be: no lists, no sets, no pub/sub, no
persistence. If you need a queue, run a real Redis.

### Gossip, because etcd would be a second thing to run

Multi-node is where it'd be easiest to reach for etcd or Consul. Both are excellent and both are another cluster you
have to operate, which defeats the point. I also didn't write a membership protocol, because that's a solved problem.
Quickwit's [chitchat](https://github.com/quickwit-oss/chitchat) crate does scuttlebutt gossip with a phi-accrual failure
detector, it's a library, it's MIT licensed, and it runs on tokio, which ePHPm already has. Nodes find each other from a
seed list and that's the entire control plane.

That's the general rule for what gets borrowed: if a well-maintained crate does the job and fits the runtime and the
license, use it. hyper, rustls, tokio and chitchat are all in that bucket.

### SQLite behind a MySQL socket, because the code only speaks pdo_mysql

A lot of the sites I'd want to run on this don't need a MySQL server. A brochure WordPress, an internal Laravel tool,
something on an edge box. They do need MySQL *as far as their code knows*, because that's what `pdo_mysql` and
`mysqli` talk to. [litewire](https://github.com/ephpm/litewire) speaks the MySQL wire protocol on one side and SQLite on
the other, so the app's config says `DB_HOST=127.0.0.1` and nothing else changes. It exists because the alternative is
asking people to port their database layer, and they won't. It's not there because SQLite is fashionable, and it's
optional: point the proxy at a real MySQL and it pools connections instead.

### Upstream, because a fork is forever

ePHPm needs a static `libphp` for every OS and architecture it ships on, and
[static-php-cli](https://github.com/crazywhalecc/static-php-cli) is the project that builds those. When it broke for me,
the fix went upstream rather than into a private fork: test fixes for Windows path separators, a `vswhere` fix for
Windows static builds, dropping `-fno-plt` from macOS static CFLAGS, a GNU mirror change, a log filter so GitHub tokens
don't end up in CI logs, a mongodb build fix. Two more Windows build fixes are still open, and the Windows embed SAPI work itself moved to php-src, where it's still under review. Everyone building
static PHP hits the same bugs, and every patch I carry privately is a patch I have to rebase forever.

The other half of that is not rebuilding PHP on every CI run. Compiling PHP and its libraries from source on each build
was the slowest thing in the pipeline, so the builds moved into a separate repo, [ephpm/php-sdk](https://github.com/ephpm/php-sdk),
that publishes prebuilt SDKs per OS/arch/PHP version. ePHPm's build downloads one and links against it.

So the three-way split I keep coming back to:

- **Contribute upstream** when the thing is shared infrastructure that others are already maintaining (SPC).
- **Use what exists** when a library does the job as-is (chitchat, hyper, rustls).
- **Build my own** only when nothing can be embedded or the license won't allow it (the KV store, the C wrapper).

### What isn't built

The list of things that don't exist matters as much as the list of things that do. As of v0.1.0:

- **No admin UI.** Early docs listed an embedded dashboard as an advantage over FrankenPHP and RoadRunner. In May I
  pulled those rows out of the comparison pages, because claiming something you haven't shipped as a feature is how
  docs turn into marketing. It's on the roadmap. It'll get built when there's something worth looking at that
  Prometheus and Grafana don't already show.
- **No OpenTelemetry.** Prometheus metrics exist because I use them. Tracing will come when I need a trace to debug
  something real.
- **No KV persistence, no lists or sets.** It's a cache.
- **Some knobs are parsed and not implemented**, and the config reference says so in bold: `auto_explain`, the
  `lag-aware` read/write split strategy, `max_replica_lag`, and the Unix socket for the Redis-compatible KV listener. Read/write splitting itself is
  the one feature I'll admit went in early on a hunch rather than a need, and the half-built strategy options next to it
  are what that looks like. I'd rather label those honestly than either delete the parsing or pretend they work.

That's the contrast I'm trying to avoid: projects that chase a competitor's feature matrix, or build the thing that
makes a good conference demo. The feature matrix in ePHPm's docs is there to be honest about status, not to win rows.

### Real apps, not hello world

The proving ground is my home lab and real applications: WordPress, Laravel, the stuff people actually deploy,
installed and clicked through on boxes I can break, on top of the e2e suite that covers the server itself. Synthetic "hello world" benchmarks are good for exactly one thing,
which is measuring dispatch overhead. Every interesting bug I've hit so far (the [SIGPROF crash](/blog/2026/04/php-signal-safety-init-ordering/)
is the best example) showed up in a real app or under real load, never in a 200-byte PHP file.

### What's next

The next stretch is about correctness under real apps rather than new boxes on the diagram: request isolation,
WordPress end to end, and making the KV store fast enough that nobody misses Redis. If something new shows up, it'll come
with its "because."

### Links

- [ePHPm](https://github.com/ephpm/ephpm) and the [v0.1.0 release](https://github.com/ephpm/ephpm/releases/tag/v0.1.0)
- [litewire](https://github.com/ephpm/litewire)
- [ephpm/php-sdk](https://github.com/ephpm/php-sdk)
- [chitchat](https://github.com/quickwit-oss/chitchat)
- [static-php-cli](https://github.com/crazywhalecc/static-php-cli), e.g. [#1149](https://github.com/crazywhalecc/static-php-cli/pull/1149) and [#1150](https://github.com/crazywhalecc/static-php-cli/pull/1150)
- [Dragonfly's license](https://github.com/dragonflydb/dragonfly/blob/main/LICENSE.md)
