---
title: "Fourth Place in Worker Mode, and Why It Wasn't PHP"
description: "ePHPm placed fourth in worker mode on the Yii3 benchmark. The slow part was the Rust layer, not PHP, and three fixes cut most of the gap."
date: 2026-10-16T08:10:00-07:00
tags: [ php, ephpm, rust, performance, benchmarking, mimalloc ]
categories: [ ephpm, php ]
images:
 - src: img/2026/10/ephpm-worker-mode-performance-cover.svg
   alt: "Diagram: before, hyper/tokio hands each request to one shared bounded channel whose waiter list all 20 PHP threads contend on; after, a dispatcher hands the job straight into the mailbox of the most recently idle PHP thread and only queues it in a FIFO overflow queue when every thread is busy"
   stretch: "none"
draft: false
---

We added [ePHPm](https://github.com/ephpm/ephpm) to the
[Yii3 runtime benchmark](https://github.com/Yii3-Benchmarks/app-api), which runs the same Yii3 API app on a bunch of
PHP runtimes and compares them (our fork with the ePHPm runner is [ephpm/app-api](https://github.com/ephpm/app-api)).
Most of the results were good. ePHPm came first in classic mode, the PHP-FPM style where every request starts fresh,
and first on the database endpoint in both modes.

Worker mode was the surprise. That's the mode where the app boots once and stays in memory, like Octane or
FrankenPHP's worker mode. We came fourth of ten, at about 50.7k requests a second, behind Workerman (~65k) and
Swoole (~62k).

Before chasing it, I checked the numbers were worth trusting. Our ranking of the other runtimes matched the suite
maintainer's published results closely, and repeat runs were within 1.5% of each other. So the gap was real.

None of this is released yet. The fixes are merged and will ship in the upcoming v0.12.0 release.

### It wasn't PHP

My first guess was PHP itself. It wasn't: per request, ePHPm's PHP threads did almost exactly as much work as
Workerman's. The difference was that ePHPm's PHP threads spent a lot of time *waiting*. Under full load they were only
busy about 70% of the time.

When PHP is waiting while requests pile up, the problem is in the part of ePHPm that's written in Rust: the HTTP
server, and the step that hands each request over to a PHP thread. (The
[thread pool post](/blog/2026/09/own-your-php-thread-pool/) explains why ePHPm runs PHP on its own threads.)

### The fixes

**A batch of small fixes ([#575](https://github.com/ephpm/ephpm/pull/575)).** Profiling pointed at a few things
directly. Simple GET requests were being handled as if they might have an upload attached, which cost an extra
round trip for nothing. Finding the file for each request checked the disk more times than it needed to. And idle
PHP threads went to sleep the instant they ran out of work, when the next request was usually a fraction of a
millisecond away.

The same runs also turned up a leak. Since PHP 8.4, persistent connections weren't cleaned up when a worker thread was
replaced, so a long-running server slowly collected open connections to its cache: 979 of them after about
10 million requests. That's fixed too.

**One queue per PHP thread ([#578](https://github.com/ephpm/ephpm/pull/578)).** This is the big one. Every request
used to go into one shared line that all 20 PHP threads pulled from, and under load they spent real time getting in
each other's way. Now a request goes straight to a thread that's free. It only waits in line when every thread is
busy, and then it's still served in order.

![Before and after: one shared line for all PHP threads vs. handing each request straight to a free thread](/img/2026/10/ephpm-worker-mode-performance-cover.svg)

**A different memory allocator ([#577](https://github.com/ephpm/ephpm/pull/577)).** Worker mode used around 300 MiB,
which looked high. Most of it was memory the standard Linux allocator holds on to after worker threads are replaced.
ePHPm's Rust code already used mimalloc, a faster allocator. Now PHP and its extensions use it too. This is on
for Linux only; macOS and Windows stay as they were.

### Results

Same app, same machine, same load for every runtime, averaged over three runs:

| Version | Requests/sec | Slowest 1% of requests | Peak memory |
|---|--:|--:|--:|
| Before | 50,855 | 1,586 ms* | 307 MiB |
| Small fixes (#575) | 56,629 | 30.5 ms | 305 MiB |
| + one queue per thread (#578) | 59,064 | 15.9 ms | 308 MiB |
| + #578 and mimalloc (#577) | 58,122 | 14.2 ms | 170 MiB |
| Swoole | 61,580 | 35.5 ms | 838 MiB |
| Workerman | 64,521 | 198.7 ms | not measured the same way |

\* Before the fixes, ePHPm was already past its limit at this load, so requests kept piling up.

In short:

- **About 15% faster** than before, with the slowest requests answered far sooner.
- **About 45% less memory**, roughly one fifth of what Swoole uses.
- **No errors** from ePHPm at any load we tried, up to 200k requests a second offered. Workerman reached its numbers
  while dropping requests: 9,427 errors in the main test.
- **The database endpoint and classic mode** didn't get faster, because other limits apply there, but classic mode
  does use less memory.

The queue change alone is about 1% faster than queue plus mimalloc. I'd rather give that 1% back for half the memory,
so the plan is to ship both.

### Why Workerman is still faster

Workerman is written in PHP. It reads and handles each request in the same process that runs the app, so nothing is
handed over between threads.

That's a different trade-off, not cheating. ePHPm puts a Rust HTTP server in front of PHP, and that's where TLS,
HTTP/2 and HTTP/3, static files, middleware and clustering live. Passing each request from that server to a PHP
thread is the price of those features, and this work is about making that price smaller rather than dropping them. At
a slightly lighter load (60k requests a second), all three were within about 1% of each other.

### What's next

ePHPm is now 4–7% behind Swoole and 9–12% behind Workerman. PHP threads are busy about 80% of the time, so what's left
is the cost of each request rather than waiting. Next up: trimming per-request overhead in the HTTP layer, waking
threads less often, tuning how many PHP threads to run, and rerunning the comparison on PHP 8.5.11 now that
[#574](https://github.com/ephpm/ephpm/pull/574) has landed.

### Caveats

- **One benchmark machine.** Your absolute numbers will differ. The relative gains are the point.
- **PHP versions.** In this run ePHPm used PHP 8.5.7 and the others 8.5.11. The upgrade, #574, has since been merged.
- **Not released yet.** #575, #577 and #578 are merged and will ship in the upcoming v0.12.0 release. The current
  v0.11.3 doesn't have them.

### Links

- [Yii3-Benchmarks/app-api](https://github.com/Yii3-Benchmarks/app-api), the benchmark suite, and
  [ephpm/app-api](https://github.com/ephpm/app-api), our fork
- [#575](https://github.com/ephpm/ephpm/pull/575): the small fixes and the connection leak
- [#578](https://github.com/ephpm/ephpm/pull/578): one queue per PHP thread
- [#577](https://github.com/ephpm/ephpm/pull/577): mimalloc for the whole process on Linux
- [#574](https://github.com/ephpm/ephpm/pull/574): PHP 8.5.11
