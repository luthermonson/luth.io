---
title: "Fourth Place in Worker Mode, and Why It Wasn't PHP"
description: "ePHPm placed fourth in worker mode on the Yii3 benchmark. The slow part was the Rust layer, not PHP, and three fixes cut most of the gap."
date: 2026-10-15T19:45:00-07:00
tags: [ php, ephpm, rust, performance, benchmarking, mimalloc ]
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

None of this is released yet. The fixes are merged and will ship in the upcoming v0.12.0 release.

### It wasn't PHP

My first guess was PHP itself. It wasn't. Per request, ePHPm's PHP threads did almost exactly as much work as
Workerman's. The difference was that ePHPm's PHP threads spent a lot of time *waiting*. Under full load they were only
busy about 70% of the time.

If PHP is waiting while requests pile up, the problem is in ePHPm's Rust code. That means the HTTP server and the step
that hands each request to a PHP thread. The [thread pool post](/blog/2026/09/own-your-php-thread-pool/) explains why
ePHPm runs PHP on its own threads.

### The fixes

**A batch of small fixes ([#575](https://github.com/ephpm/ephpm/pull/575)).** Profiling pointed straight at a few
things. Simple GET requests were handled as if they might have an upload attached. That cost an extra round trip for
nothing. Finding the file for each request checked the disk more often than it needed to. And idle PHP threads went
to sleep the instant they ran out of work. The next request was usually a fraction of a millisecond away.

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

### Results

Every runtime ran the same app on the same machine under the same load. Each number is the average of three runs.

| Version | Requests/sec | Slowest 1% of requests | Peak memory |
|---|--:|--:|--:|
| Before | 50,855 | 1,586 ms* | 307 MiB |
| Small fixes (#575) | 56,629 | 30.5 ms | 305 MiB |
| + one queue per thread (#578) | 59,064 | 15.9 ms | 308 MiB |
| + #578 and mimalloc (#577) | 58,122 | 14.2 ms | 170 MiB |
| Swoole | 61,580 | 35.5 ms | 838 MiB |
| Workerman | 64,521 | 198.7 ms | not measured the same way |

\* Before the fixes, ePHPm was already past its limit at this load. Requests kept piling up.

![Bar charts of requests/sec and peak memory: ePHPm rises from 50,855 to 58,122 requests/sec with all three fixes, below Swoole's 61,580 and Workerman's 64,521, while peak memory drops from 307 MiB to 170 MiB against Swoole's 838 MiB](/img/2026/10/ephpm-worker-mode-performance-results.svg)

In short:

- **About 15% faster** than before. The slowest requests are answered far sooner.
- **About 45% less memory.** That's roughly one fifth of what Swoole uses.
- **No errors** from ePHPm at any load we tried, up to 200k requests a second. Workerman dropped requests to reach its
  numbers. It had 9,427 errors in the main test.
- **The database endpoint and classic mode** didn't get faster, because other limits apply there. Classic mode does
  use less memory.

The queue change alone is about 1% faster than the queue plus mimalloc. I'd rather give back that 1% for half the
memory. The plan is to ship both.

### Why Workerman is still faster

Workerman is written in PHP. It reads and handles each request in the same process that runs the app. Nothing is
handed over between threads.

That's a different trade-off, not cheating. ePHPm puts a Rust HTTP server in front of PHP. That server handles TLS,
HTTP/2 and HTTP/3, static files, middleware and clustering. Handing each request from it to a PHP thread is the price
of those features. This work makes that price smaller without dropping any of them. At a slightly lighter load of 60k
requests a second, all three were within about 1% of each other.

### What's next

ePHPm is now 4–7% behind Swoole and 9–12% behind Workerman. PHP threads are busy about 80% of the time. What's left
is the cost of each request, not waiting.

Next up:

- Trim per-request overhead in the HTTP layer.
- Wake threads less often.
- Tune how many PHP threads to run.
- Rerun the comparison on PHP 8.5.11, now that [#574](https://github.com/ephpm/ephpm/pull/574) has landed.

### Caveats

- **One benchmark machine.** Your absolute numbers will differ. The relative gains are the point.
- **PHP versions.** In this run ePHPm used PHP 8.5.7 and the others used 8.5.11. The upgrade (#574) has since been
  merged.
- **Not released yet.** #575, #577 and #578 are merged and will ship in the upcoming v0.12.0 release. The current
  v0.11.3 doesn't have them.

### Links

- [Yii3-Benchmarks/app-api](https://github.com/Yii3-Benchmarks/app-api): the benchmark suite
- [ephpm/app-api](https://github.com/ephpm/app-api): our fork with the ePHPm runner
- [#575](https://github.com/ephpm/ephpm/pull/575): the small fixes and the connection leak
- [#578](https://github.com/ephpm/ephpm/pull/578): one queue per PHP thread
- [#577](https://github.com/ephpm/ephpm/pull/577): mimalloc for the whole process on Linux
- [#574](https://github.com/ephpm/ephpm/pull/574): PHP 8.5.11
