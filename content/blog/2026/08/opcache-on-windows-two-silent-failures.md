---
title: "OPcache on Windows: Two Silent Failures"
description: "An ASLR fatal on Windows looked like a memory-size lottery. Separately, Windows PHP 8.3 builds never had an opcode cache, and the fix sat unpicked for three weeks."
date: 2026-08-29T16:45:00-07:00
tags: [ php, opcache, windows, ephpm, debugging ]
categories: [ php, windows, ephpm ]
images:
 - src: img/2026/08/opcache-windows-silent-failures-cover.svg
   alt: "Two ePHPm processes computing the same OPcache shared-memory name; the second reattaches and dies on the ASLR check"
   stretch: "none"
draft: false
---

Windows is the platform where [ePHPm](https://github.com/ephpm/ephpm) gets the least traffic and the most surprises.
Last week closed out two OPcache bugs that only exist there. Both were quiet in their own way. One killed the server
at startup with nothing in our own logs. The other didn't kill anything. It just ran PHP 8.3 with no opcode cache at
all. For three weeks after a fixed SDK was published, every check stayed green.

For context, ePHPm is a Rust server with PHP embedded as a static library. On Windows that's a ZTS
`php8embed.lib` built by our [php-sdk](https://github.com/ephpm/php-sdk) pipeline and linked into `ephpm.exe`. The
versions here are PHP 8.5.7 ZTS on Windows 11, plus the 8.3 and 8.4 SDK pins we ship.

### "Opcode handlers are unusable due to ASLR"

The report was that an `ephpm.exe` sometimes dies at launch with the message below. PHP writes it to stderr and to the
Windows Event Log. Then the process calls `exit(-2)` from inside `php_module_startup`:

```
Fatal Error Opcode handlers are unusable due to ASLR. Please setup opcache.file_cache
and opcache.file_cache_fallback directives for more convenient Opcache usage
```

ePHPm's own log shows the autotune lines and then nothing. And it looked like a size problem, which sent the first
round of debugging in the wrong direction. `opcache.memory_consumption = 512` died. 256, 333 and 400 all worked.
Turning the JIT off "fixed" it too. That reads like "512 MB is too big to reserve contiguously once ASLR has scattered
things around". It isn't.

### The segment has a name, and the size is part of it

On Windows, OPcache's shared memory is a *named, session-global section object*. The name comes from
`create_name_with_username()` in `ext/opcache/shared_alloc_win32.c`:

```
ZendOPcache.SharedMemoryArea@<md5(username + opcache.cache_id)>@<sapi_name>@<zend_system_id><requested_size_hex>
```

A second process that computes the **same name** doesn't create its own segment. It reattaches to the first one's.
That's how PHP on Windows shares an opcode cache across php-cgi processes. Reattaching is only legal if the VM's
opcode handlers sit at the same address in both processes. Cached `op_array`s hold absolute pointers to handlers
inside the loaded image. `zend_shared_alloc_reattach()` checks exactly that:

```c
wanted_mapping_base = ((void**)mapping_base)[0];
execute_ex_base     = ((void**)mapping_base)[1];
...
execute_ex_moved = (void *)execute_ex != execute_ex_base;
```

When it has moved, it logs at `ACCEL_LOG_FATAL`. Whatever the name suggests, that isn't a log level.
`zend_accelerator_debug.c` handles it outside the verbosity check:

```c
/* perform error handling even without logging the error */
switch (type) {
    case ACCEL_LOG_ERROR: zend_bailout(); break;
    case ACCEL_LOG_FATAL: exit(-2);       break;
}
```

So `opcache.log_verbosity_level` can't suppress it. `zend_try` can't catch it either, because it's `exit()` rather
than a bailout. Two details then explain every symptom:

1. **The requested size is in the name** (`memory_consumption` plus the page-aligned `jit_buffer_size`). Whether two
   processes collide depends on configured size. That's why it presented as a size lottery. It's also why turning the
   JIT off "fixed" it, since that removes the JIT buffer from the size.
2. **Windows picks a randomized image base per file and keeps reusing it, in practice until reboot.** Run the same
   `ephpm.exe` twice and both processes get the same base, so the reattach check passes. Copy the exact same bytes to
   a different path and the copy gets a different base. That's why it was deterministic per binary and not random.

Autotune derived 512 MB on any host with more than about 2.8 GB of RAM. In practice every Windows box was one renamed
binary away from this. The reproduction is three commands, with instance A still running:

```
ephpm.exe serve --config a.toml          # creates the 512 MB-named segment, OK
ephpm.exe serve --config b.toml          # same binary, same base: reattach OK
copy ephpm.exe ephpm-call.exe
ephpm-call.exe serve --config c.toml     # same bytes, new path, new base: dies
```

The controls that confirmed it, all with instance A holding the segment:

| Variant | Result |
|---|---|
| copy, default 512 MB | dies with the ASLR fatal |
| copy, `opcache_memory_consumption = 256` | starts (different size, different name) |
| copy, still 512 MB, `opcache.cache_id` set via `ini_overrides` | starts and serves PHP |
| same binary, same path, 512 MB | starts (same image base) |

The third row is the one that settles it. Same bytes, same 512 MB, same concurrent instance, and the only change is
the namespace.

### The fix: never reattach

ePHPm is one multi-threaded ZTS process that never forks workers. It has no use for a segment shared with an unrelated
process. On Linux and macOS PHP doesn't support cross-process reattachment at all. The `ZEND_OPCACHE_SHM_REATTACHMENT`
define in `zend_portability.h` is literally "Currently only possible on Windows." So the fix gives every process its
own namespace:

```
opcache.cache_id = ephpm-<pid>
```

With a private name, ePHPm always takes the *create* path, which performs no `execute_ex` check. `opcache.cache_id`
only exists on Windows, so it's only emitted there. It's written before the operator's `ini_overrides`, so anyone who
really wants a shared segment can still pin their own.

The options I rejected are worth listing, because each would have looked like a fix:

- **Cap the size.** That just moves the collision to a different size. Same lottery, different numbers.
- **`/DYNAMICBASE:NO`.** Turning off ASLR in a server process that runs untrusted PHP to work around a startup bug is
  a bad trade.
- **`opcache.file_cache` + `file_cache_fallback`.** Turns the fatal into a warning and a much slower file-cache-only
  mode. It also adds an on-disk cache with its own multi-tenant questions.
- **`opcache.mmap_base`.** Moves the segment, not `execute_ex`. Doesn't help.

The PR also caps the *derived* `memory_consumption` at 256 MB on Windows, for a separate reason. The Windows section
is pagefile-backed and charged against the commit limit in full at startup. Linux commits lazily. To be clear, the cap
doesn't fix the ASLR abort. The negative control still dies. That control forced both binaries onto one shared
`cache_id` at 256 MB. An explicit operator value is still honored, with a startup warning. That's
[ephpm#363](https://github.com/ephpm/ephpm/pull/363).

### PHP 8.3 on Windows ran with no OPcache, and the fix sat unpicked for three weeks

The second one turned up while I was verifying the JIT default on Windows. In serve mode on our PHP 8.3 build,
`opcache_get_status()` returned `false`. The extension was loaded. It just never engaged.

Up through PHP 8.4, OPcache only activates for SAPIs it knows about. There's a `supported_sapis` allowlist in
`ZendAccelerator.c` (apache2handler, fpm-fcgi, cli-server, litespeed, frankenphp and so on). An unknown SAPI name
quietly gets no opcode cache. ePHPm's SAPI is named `ephpm`, so our SDK builds patch the allowlist. PHP 8.5 removed
the allowlist upstream, which is why the 8.5 builds were never affected.

To check whether a static library actually carries the patch, you don't need a debugger. Grep the string table. Use
SAPI names you know are in the upstream list as controls, so you know you're reading the right table:

| Windows SDK | `ephpm` in allowlist | `frankenphp` / `fuzzer` controls |
|---|---|---|
| 8.3.31 (what we pinned) | missing | both present |
| 8.3.33 | present | both present |
| 8.4.23 (what we pin) | present | both present |
| 8.4.24 | present | both present |

So the SDK pipeline was fine. 8.3.33 had carried the patch since August 2nd. We were just pinning an older 8.3 that
predates the allowlist patch. That means our Windows 8.3 builds had gone without OPcache since 8.3 joined the release
matrix in June. The original report (php-sdk#47) had also blamed 8.4, which was never broken. The fix in
[ephpm#373](https://github.com/ephpm/ephpm/pull/373) moves one pin, 8.3.31 to 8.3.33. It deliberately leaves 8.4 alone
so an OPcache fix doesn't drag in an unrelated PHP version change.

### Why nobody noticed: the bump PR that couldn't push

We have a nightly "SDK Bump" workflow that opens a PR whenever a newer SDK is published. It had been trying to land
exactly this 8.3.33 bump every night since it shipped. It kept failing at the push:

```
! [remote rejected] bump/php-8.3.33 (refusing to allow a GitHub App to create or
  update workflow `.github/workflows/release.yml` without `workflows` permission)
```

`GITHUB_TOKEN` can't modify files under `.github/workflows/`, and `release.yml` is one of the pin sites. The failure
was in a scheduled workflow, so nobody saw it. Fixing that needs a token with workflow scope. That's its own decision
and got its own follow-up.

Using the bump tool by hand then turned up two more bugs in it. Substitutions that targeted the same file each started
from a fresh read of the file, so the last write won. `bump-php-pin 8.3 8.3.33` moved three pin sites in `release.yml`
and silently dropped the other two. That would have left the two Linux release legs building the old SDK. Its own
`--check` flagged the result as drift, which is how I caught it. The nightly run had produced the same incomplete
diff. Substitutions now compose per file, through a pure function that's unit tested without touching disk. The second
bug was that `windows-php-check.yml` hard-coded the 8.3 SDK path, so it wasn't a known pin site at all. It is now.

### The pattern

The same lesson showed up in July. It turned out SHA-NI had been off in our Linux 8.3 and 8.4 builds for the life of
the project. A C++-only compiler flag leaked into the C flags and made an autoconf probe fail. The hardware sha256
path compiled out. sha256 ran at roughly 2.7x its potential cost
([benchmarking findings](https://ephpm.dev/benchmarking/findings/)). The fix added a build guard that greps for the
`SHA256_Transform_shani` symbol.

All three bugs have the same shape. A config field existing, or a feature "being enabled", doesn't mean the machine
code is in the binary. It doesn't mean the code path runs either. On Windows especially, the cheap check is the
artifact itself. Grep the `.lib` for the string, or `nm` it for the symbol, with a control you know should be there.
For OPcache the runtime check is `opcache_get_status()` returning an array and not `false`. That's the check worth
running on every platform you ship, not just the one CI runs on.

### Links

- [ephpm#362](https://github.com/ephpm/ephpm/issues/362) / [ephpm#363](https://github.com/ephpm/ephpm/pull/363) - the ASLR startup abort and the private `opcache.cache_id`
- [ephpm#373](https://github.com/ephpm/ephpm/pull/373) - pin 8.3 to 8.3.33; Windows OPcache was silently off
- [ephpm/php-sdk#47](https://github.com/ephpm/php-sdk/issues/47) - the original allowlist report
- [ephpm/php-sdk#36](https://github.com/ephpm/php-sdk/pull/36) - re-enabling SHA-NI, PCLMUL and AVX2 in the SDK build
- php-src: [`ext/opcache/shared_alloc_win32.c`](https://github.com/php/php-src/blob/master/ext/opcache/shared_alloc_win32.c), [`ext/opcache/zend_accelerator_debug.c`](https://github.com/php/php-src/blob/master/ext/opcache/zend_accelerator_debug.c)
