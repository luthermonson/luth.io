---
title: "Fourth Place in Worker Mode, and Why It Wasn't PHP"
description: "ePHPm placed fourth in worker mode on the Yii3 benchmark. The slow part was the Rust layer, not PHP. Six changes in v0.12.0 made it 13% faster on 57% less memory."
date: 2026-10-15T19:45:00-07:00
lastmod: 2026-10-16T18:00:00-07:00
tags: [ php, ephpm, rust, performance, benchmarking, mimalloc, swoole, workerman ]
categories: [ ephpm, php ]
images:
 - src: img/2026/10/ephpm-worker-mode-performance-cover.svg
   alt: "Diagram: before, hyper/tokio hands each request to one shared bounded channel whose waiter list all 20 PHP threads contend on; after, a dispatcher hands the job straight into the mailbox of the most recently idle PHP thread and only queues it in a FIFO overflow queue when every thread is busy"
   stretch: "none"
draft: false
---

We added [ePHPm](https://github.com/ephpm/ephpm) to the
[Yii3 runtime benchmark](https://github.com/Yii3-Benchmarks/app-api). It runs the same Yii3 API app on a bunch of PHP
runtimes and compares them. Our fork with the ePHPm runner is [ephpm/app-api](https://github.com/ephpm/app-api).

Most of the results were good. ePHPm came first in classic mode and first on the database endpoint in both modes.
Classic mode is the PHP-FPM style, where every request starts fresh.

Worker mode was the surprise. In worker mode the app boots once and stays in memory, like Octane or FrankenPHP's
worker mode. We came fourth of ten at about 50.7k requests a second. Workerman (~65k) and Swoole (~62k) were ahead.

First I checked that the numbers were worth trusting. Our ranking of the other runtimes closely matched the suite
maintainer's published results. Repeat runs were within 1.5% of each other. So the gap was real.

> **Updated October 16.** Everything here shipped in
> [ePHPm v0.12.0](https://github.com/ephpm/ephpm/releases/tag/v0.12.0). That's the three fixes from the first version
> of this post plus three more. I reran the whole comparison on the release image, with every runtime on PHP 8.5.11
> and five runs each. The results section uses those numbers. It also corrects two claims the first version got
> wrong: the memory comparison with Swoole, and why Workerman had errors.

### It wasn't PHP

My first guess was PHP itself. It wasn't. Per request, ePHPm's PHP threads did almost exactly as much work as
Workerman's. The difference was that ePHPm's PHP threads spent a lot of time *waiting*. Under full load they were only
busy about 70% of the time.

If PHP is waiting while requests pile up, the problem is in ePHPm's Rust code. That means the HTTP server and the step
that hands each request to a PHP thread. The [thread pool post](/blog/2026/09/own-your-php-thread-pool/) explains why
ePHPm runs PHP on its own threads.

![Stacked bars of PHP thread time at 65k requests/sec offered: v0.11.3 runs PHP 69% of the time, waits for a CPU 13% and is idle or blocked 18%; v0.12.0 runs PHP 77%, waits 16% and is idle or blocked 7%](/img/2026/10/ephpm-worker-mode-performance-thread-time.svg)

That's the whole story in one picture. The idle slice shrank from 18% to 7%. The time went to running PHP.

### Round one: getting requests to PHP faster

**A batch of small fixes ([#575](https://github.com/ephpm/ephpm/pull/575)).** Profiling pointed straight at a few
things. Simple GET requests were handled as if they might have an upload attached. That cost an extra round trip for
nothing. Finding the file for each request checked the disk more often than it needed to: `/` took seven `stat` calls
and now takes three. And idle PHP threads went to sleep the instant they ran out of work. The next request was usually
a fraction of a millisecond away. Now they wait up to 200 µs for it first.

The same runs also turned up a leak. Since PHP 8.4, persistent connections weren't cleaned up when a worker thread was
replaced. A long-running server slowly piled up open connections to its cache. After about 10 million requests there
were 979 of them. That's fixed too.

**One queue per PHP thread ([#578](https://github.com/ephpm/ephpm/pull/578)).** This is the big one. Every request
used to go into one shared line, and all 20 PHP threads pulled from it. Under load they spent real time getting in
each other's way. Now a request goes straight to a free thread. It only waits in line when every thread is busy, and
it's still served in order.

![Before and after: one shared line for all PHP threads vs. handing each request straight to a free thread](/img/2026/10/ephpm-worker-mode-performance-cover.svg)

**A different memory allocator ([#577](https://github.com/ephpm/ephpm/pull/577)).** Worker mode used around 300 MiB.
That looked high. Most of it was memory the standard Linux allocator holds on to after worker threads are replaced.
ePHPm's Rust code already used mimalloc, a faster allocator. Now PHP and its extensions use it too. This is on for
Linux only. macOS and Windows stay as they were.

Development builds, three runs each:

| Version | Requests/sec | Slowest 1% of requests | Peak memory |
|---|--:|--:|--:|
| Before | 50,855 | 1,586 ms* | 307 MiB |
| Small fixes (#575) | 56,629 | 30.5 ms | 305 MiB |
| + one queue per thread (#578) | 59,064 | 15.9 ms | 308 MiB |
| + #578 and mimalloc (#577) | 58,122 | 14.2 ms | 170 MiB |

\* Before the fixes, ePHPm was already past its limit at this load. Requests kept piling up.

The queue change alone is about 1% faster than the queue plus mimalloc. I'd rather give back that 1% for half the
memory, so both shipped.

### Round two: fewer syscalls, smarter waiting, more PHP threads

After round one the PHP threads were busy about 80% of the time. The rest of the gap was the cost of each request. So
the next round went after smaller things.

**One timer per request instead of four ([#581](https://github.com/ephpm/ephpm/pull/581)).** PHP enforces
`max_execution_time` with a timer on each thread. Every worker request was setting that timer four times, plus two
more signal syscalls along the way. Now a busy thread sets it once per request. It's only switched off when the thread
actually goes idle. Under load that's 6 syscalls per request down to about 1.

**Adaptive spinning ([#582](https://github.com/ephpm/ephpm/pull/582)).** The 200 µs wait from #575 is great under load.
On a quiet server it's waste: the thread burns CPU waiting for a request that isn't coming. Now each thread learns. If
the next request usually shows up inside its budget, it keeps spinning. If not, it halves its budget, down to zero. New
requests go to a thread that's still spinning before one that's asleep.

![Grouped bars of CPU per request at 1k, 5k and 20k requests/sec: the fixed 200 µs spin costs 524, 467 and 412 µs; adaptive spin costs 423, 384 and 393 µs; no spin at all costs 362, 329 and 363 µs](/img/2026/10/ephpm-worker-mode-performance-adaptive-spin.svg)

At full load it changes nothing, which is the point. At 1k–5k requests a second it uses 18–19% less CPU per request,
with the same latency. That's where most real servers spend most of their time. Dropping the spin entirely is cheaper
still when the server is quiet, but it costs 3–4% of throughput at full load.

**Splitting the cores ([#580](https://github.com/ephpm/ephpm/pull/580)).** ePHPm's HTTP side runs on tokio, which by
default starts one I/O thread per CPU. That's 32 threads on this machine, competing with 20 PHP threads for 32 cores.
The new `[server] io_threads` setting lets you shift that balance:

```toml
[server]
io_threads = 16

[php]
concurrency = 24
```

That's 24 PHP threads and 16 I/O threads. Fewer PHP threads went the wrong way: 16 cost 6% on the home page and 11% on
the database route.

Each change alone on top of round one, three to five runs each:

| Change | Requests/sec | vs round one | Slowest 1% at 60k |
|---|--:|--:|--:|
| Round one | 58,520 | — | 706 ms |
| One timer (#581) | 59,529 | +1.7% | 411 ms |
| Adaptive spin (#582) | 58,805 | +0.5% | 694 ms |
| 24 PHP / 16 I/O threads (#580) | 60,674 | +3.7% | 99 ms |
| All three | 60,796 | +3.9% | 78 ms |

The timer and the spin are on by default. The thread split isn't, yet. The right numbers depend on the machine and
the app, and 16 I/O threads hasn't been tested alone with the default PHP thread count. So v0.12.0 ships the setting
and leaves tokio's default in place.

### Results: v0.12.0

Every runtime ran the same app on the same machine under the same load. That's a Ryzen 9 5950X with 32 threads under
WSL2, with the load generator, Postgres and Valkey on the same host. Each test is 30 seconds at a fixed request rate.
The six fastest runtimes got five runs each and the slower two got three, interleaved. Every runtime used PHP 8.5.11
and its config from the benchmark repo, which runs 20 PHP workers for everyone.

| Runtime | Requests/sec at 65k offered | Errors | Slowest 1% at 55k | Database route | CPU per request | Peak memory |
|---|--:|--:|--:|--:|--:|--:|
| ePHPm v0.11.3 | 51,417 | 0 | 1,493 ms | 5,855 | 354 µs | 358 MiB |
| **ePHPm v0.12.0** | **58,101** | **0** | **22 ms** | **5,842** | **337 µs** | **153 MiB** |
| ePHPm v0.12.0, 24/16 threads | 60,492 | 0 | 14 ms | 6,004 | 336 µs | 152 MiB |
| Swoole | 61,279 | 0 | 36 ms | 5,685 | 297 µs | 99 MiB |
| Workerman, as shipped | 64,510 | ~3,100 per test | 173 ms | 5,586 | 272 µs | 92 MiB |
| Workerman, recycle removed | 63,021 | 0 | 1,116 ms | 5,663 | 272 µs | 112 MiB |
| FrankenPHP worker | 34,667 | 0 | 10.2 s | 5,353 | 517 µs | 130 MiB |
| RoadRunner | 19,430 | 0 | 19.1 s | 4,461 | 879 µs | 649 MiB |

Requests/sec counts only successful requests. "Slowest 1%" is the median p99 across runs. The database route is
`/postgres/orders` at 6k requests a second offered. Memory is the whole container.

![Three bar charts. Requests/sec at 65k offered: ePHPm v0.11.3 51,417; v0.12.0 58,101; v0.12.0 with 24/16 threads 60,492; Swoole 61,279; Workerman with no recycle 63,021; Workerman as shipped 64,510 with about 3,100 errors per test. CPU per request: 354, 337, 336, 297, 272 and 272 µs. Peak container memory: 358, 153, 152, 99, 112 and 92 MiB](/img/2026/10/ephpm-worker-mode-performance-results.svg)

In short:

- **13% faster than v0.11.3** at full load: 51.4k to 58.1k requests a second.
- **The slowest requests are back under control.** At 55k a second, v0.11.3's slowest 1% took 1.5 seconds because
  it couldn't keep up. v0.12.0's took 22 ms.
- **57% less memory** than v0.11.3: 358 MiB down to 153 MiB.
- **No errors from ePHPm** in any test. Each run replaced about 600 worker threads under full load without failing a
  request.
- **First on the database route.** With 24 PHP threads it was the only runtime that served the full 6k a second, with
  its slowest 1% at 27 ms.

![Slowest 1% of requests at 55k and 60k requests/sec offered, log scale: ePHPm v0.11.3 1.5 s and 4.3 s; v0.12.0 22 ms and 544 ms; v0.12.0 with 24/16 threads 14 ms and 103 ms; Swoole 36 ms and 80 ms; Workerman as shipped 173 ms and 560 ms; Workerman with no recycle 1.1 s and 5.9 s](/img/2026/10/ephpm-worker-mode-performance-latency.svg)

Classic mode didn't get faster, and it didn't need to. ePHPm was already first. It still is, on less memory:

| Classic mode | Requests/sec (limit) | vs PHP-FPM | Peak memory |
|---|--:|--:|--:|
| **ePHPm v0.12.0** | **5,496** | **+9.6%** | **84 MiB** |
| ePHPm v0.11.3 | 5,455 | +8.8% | 136 MiB |
| PHP-FPM + nginx | 5,015 | — | 96 MiB |
| FrankenPHP classic | 4,911 | −2.1% | 107 MiB |

**Two corrections to the first version of this post.** It said ePHPm used about a fifth of Swoole's memory. That was
wrong. The old method added up the memory of every process. Swoole and Workerman each run 20 worker processes that
share most of their memory, so the same pages were counted many times. Measured per container, Swoole uses 99 MiB and
Workerman 92 MiB, against ePHPm's 153 MiB. ePHPm's own number barely moves either way.

It also said Workerman dropped requests to reach its numbers. That's not quite fair, as the next section explains.

### About Workerman's errors

The benchmark's Workerman script restarts each worker process after 10,000 requests. Swoole, FrankenPHP and ePHPm use
the same limit, so that part is fair. The difference is what a restart does. When a Workerman worker stops, it closes its idle
keep-alive connections on the spot, and the load generator counts each one as an error. That's about 3,100 errors
in every 30-second test, every time.

ePHPm, Swoole and RoadRunner hold the connections outside the PHP worker, so replacing a worker drops nothing.

With the restart removed, Workerman still does 63.0k a second with zero errors. That's 8% ahead of v0.12.0, so its
speed is real. But its slowest requests get much worse: about 6 seconds at 60k. Without restarts, each connection
stays stuck to whichever of the 20 processes accepted it, and the busy ones fall behind. The restarts were quietly
rebalancing them.

### Why Swoole and Workerman are still faster

ePHPm's PHP threads aren't waiting much any more. What's left is CPU: ePHPm spends 337 µs per request, Swoole 297
and Workerman 272.

Workerman is written in PHP. It reads and handles each request in the same process that runs the app. Nothing is
handed over between threads. Swoole is a C extension with its own lean HTTP server built for exactly this.

That's a different trade-off, not cheating. ePHPm puts a Rust HTTP server in front of PHP. That server handles TLS,
HTTP/2 and HTTP/3, static files, middleware and clustering. Handing each request from it to a PHP thread is the price
of those features. This work made that price a lot smaller without dropping any of them. Just below the limit, at 60k
requests a second, ePHPm, Swoole and Workerman were within about 1% of each other.

### What's next

- **Profile the CPU cost per request.** That's the 40 µs between us and Swoole. The likely places are building the
  request object PHP sees, header handling, and copying the response back out.
- **Pick a better default thread split** once it's been measured on more than one machine.
- **Get ePHPm into the upstream benchmark**, and suggest 24 PHP threads for it in this app.

### Caveats

- **One benchmark machine.** Your absolute numbers will differ. The relative gains are the point.
- **A shared host.** The load generator and databases run beside the server. For about an hour, a few runs of several
  runtimes came in low, probably from something on the Windows side. I kept them. Medians, and a cut with only the
  cleaner runs, give the same ranking.
- **Same settings for everyone.** Every runtime used the benchmark's own config with 20 workers. The 24/16 row is my
  tuning for ePHPm. Nobody tuned the others.
- **Different runs.** The round one and round two tables come from development builds and earlier runs. Compare rows
  within a table, not across tables.

### Links

- [ePHPm v0.12.0](https://github.com/ephpm/ephpm/releases/tag/v0.12.0): the release with all of this
- [Yii3-Benchmarks/app-api](https://github.com/Yii3-Benchmarks/app-api): the benchmark suite
- [ephpm/app-api](https://github.com/ephpm/app-api): our fork with the ePHPm runner
- [#575](https://github.com/ephpm/ephpm/pull/575): the small fixes and the connection leak
- [#578](https://github.com/ephpm/ephpm/pull/578): one queue per PHP thread
- [#577](https://github.com/ephpm/ephpm/pull/577): mimalloc for the whole process on Linux
- [#581](https://github.com/ephpm/ephpm/pull/581): one execution timer per request
- [#582](https://github.com/ephpm/ephpm/pull/582): adaptive spinning before a PHP thread sleeps
- [#580](https://github.com/ephpm/ephpm/pull/580): the `[server] io_threads` setting
- [#574](https://github.com/ephpm/ephpm/pull/574): PHP 8.5.11
