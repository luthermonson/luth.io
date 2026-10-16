---
title: "Composer Without PHP: Embedding vivacity in ePHPm"
description: "I embedded vivacity, a pure-Rust Composer-compatible installer, in ePHPm so one binary can install a PHP app's dependencies without PHP. This is what it took upstream and in ePHPm, and how it does against Composer on Windows."
date: 2026-10-14T23:20:00-07:00
tags: [ php, rust, ephpm, composer, vivacity, windows ]
categories: [ ephpm, php, rust ]
images:
 - src: img/2026/10/composer-without-php-vivacity-bench.svg
   alt: "Bar chart comparing Composer 2.10.3 and ephpm composer on a Laravel install on native Windows"
   stretch: "none"
draft: false
---

The pitch for [ePHPm](https://github.com/ephpm/ephpm) is one binary in place of php-fpm, nginx and Redis. There was
always a hole in it though. To deploy a PHP app you still need to run `composer install`, and that drags PHP and
`composer.phar` back onto the box, or into the container build, just for that one step. `ephpm composer install` now
runs a pure-Rust, Composer-compatible installer called [vivacity](https://github.com/Adelagric/vivacity) inside the
ePHPm binary. One binary serves the app and installs its dependencies, and on Windows that works with no PHP installed at
all. Getting it right took five upstream PRs, a fork, a docs rewrite that took back claims we'd already made, and
teaching `ephpm` to pretend to be `php`.

### Why vivacity

vivacity's promise is a **byte-for-byte identical `vendor/`** compared to real Composer. Not "close enough", not
"works for the common cases", identical files. That's what makes it a drop-in: if the output is the same bytes, nothing
downstream can tell the difference.

The embed itself is small ([#523](https://github.com/ephpm/ephpm/pull/523)). `ephpm composer` captures every argument
after `composer` verbatim, prepends a program name, calls `vivacity::run(argv)` in-process and exits with whatever code
it returns, so it drops straight into CI scripts:

```bash
ephpm composer install --no-dev --optimize-autoloader
ephpm composer require monolog/monolog
ephpm composer --version
```

### Contributing upstream first

vivacity didn't run on Windows, and it couldn't be linked into ePHPm. Both had to be fixed upstream before the embed
was worth anything.

**Windows support** ([vivacity #2](https://github.com/Adelagric/vivacity/pull/2)). The workspace already *compiled*
on Windows, because every Unix-ism was `#[cfg(unix)]`-gated, but the Windows side of every gate was a silent no-op.
Symlinks were dropped, there were no cmd-usable `vendor/bin` proxies, cache directories were wrong, and verbatim
`\\?\` paths leaked into generated autoload files. The PR fills all of that in, and it ports Composer's `bin-compat`
rule for the `.bat` proxies (written on Windows and WSL by default, everywhere with `bin-compat: full`), so `vendor/bin`
matches what Composer produces on every platform.

To prove it I built a small workbench, `vivace-bench`: official PHP 8.3.33 for Windows, Composer 2.10.3, and a test
project pinning monolog, psr/log and nikic/php-parser (which ships a `vendor/bin` binary, so it exercises the `.bat`
path). A script runs native Windows `composer install`, snapshots `vendor/`, wipes it, runs vivacity from the same lock
and compares a sha256 manifest of every file. Result: 429 files each, every hash matching, and `dump-autoload -o`
identical too. That includes `vendor/bin/php-parse.bat` with its CRLF line endings.

**Making it embeddable.** This is where it got interesting. ePHPm statically links `libphp`, and the PHP SDK already
bundles bzip2, xz, zstd, zlib and PCRE2 as plain C symbols. Any Rust crate that pulls in a second copy of one of those C
libraries collides at link time:

```
rust-lld: error: duplicate symbol: BZ2_bzDecompress
(...and the rest of BZ2_*)
error: could not compile `ephpm` (bin "ephpm")
```

vivacity's own code never touched those libraries; they arrived through the `zip` crate's default features. Since
extraction only ever decompresses, the fix was to switch bzip2 to the pure-Rust `libbz2-rs-sys` backend (its exports
carry a version prefix, so nothing collides), keep deflate and LZMA on their existing pure-Rust backends, and drop the
zip xz and zstd methods, which no Composer dist uses, instead of linking 219 colliding `lzma_*` symbols. Upstream and
I landed on the same `zip` change 47 minutes apart: the maintainer's commit shipped as 0.11.1, and
[vivacity #6](https://github.com/Adelagric/vivacity/pull/6) went in on top (released in 0.12.0) with a CI check that
keeps C libraries out of the graph, plus the symbol measurements. Upstream had already prefixed PCRE2 in 0.11.0.

TLS had the same shape of problem. vivacity hard-enabled reqwest's `rustls-tls`, which compiles in the `ring` crypto
provider. ePHPm is pinned to `aws-lc-rs`, and with two providers compiled in rustls refuses to pick one and panics on the
first bare `ClientConfig::builder()`. That's a bug ePHPm had shipped once before, and there's a test guarding against it
now. Cargo features are additive and can't be subtracted from outside, so the knob had to exist upstream:
[vivacity #5](https://github.com/Adelagric/vivacity/pull/5) added `rustls-tls-no-provider` alongside the default
`rustls-tls-ring`.

And while I was in there, [vivacity #4](https://github.com/Adelagric/vivacity/pull/4) parallelized store-to-vendor
materialization and the classmap scan with rayon, while keeping the output byte-identical. The classmap merge is replayed
sequentially in Composer's exact order, so parallelism can't change a byte. Measured with hyperfine on WSL2 against
v0.7.0, a wiped-`vendor/` install from a warm store was 2.24x faster on Sylius and 1.60x on Laravel, and the upstream
differential harness stayed at 0 diff across all six fixtures. Upstream kept the file reads sequential on APFS, where
the fan-out measured slower.

The resulting dependency line in ePHPm says exactly why it looks the way it does:

```toml
vivacity = { version = "=0.11.1", default-features = false, features = ["rustls-tls-no-provider"] }
```

### Keeping up with a crate that ships daily

vivacity moves fast. The pin went 0.11.1 to 0.15.0 two days after the embed merged, and 0.15 to 0.16 to 0.17 happened
in three days. "Vivacity Pin Watch" ([#532](https://github.com/ephpm/ephpm/pull/532)) is a daily job that opens a bump
PR whenever crates.io is ahead of the exact `=` pin. It never auto-merges. The human gets a linking checklist instead:
no new C-linking crate (a pure-Rust addition like `zlib-rs` is fine, `libz-sys` is not), still exactly one crypto
provider, `vivacity::run()` unchanged. The pin is exact precisely so a release can't sneak a new `-sys` crate into a
binary that statically links PHP.

Side note worth knowing if you build one of these: PRs opened with the default `GITHUB_TOKEN` don't trigger CI.
That's GitHub's recursion guard. It's documented, and it's still easy to forget. Close and reopen the PR, push to it,
or use a PAT or GitHub App token for the job.

### The honest docs rewrite

The docs had been saying `ephpm composer` "needs no PHP runtime". [#542](https://github.com/ephpm/ephpm/pull/542)
(merged after v0.11.3, so its corrected `--help` text ships in the upcoming v0.12.0) re-checked every claim against vivacity 0.17.0's source and real runs on a Windows box with no `php` and no `composer`
on `PATH`, and that one was wrong for `update`, `require` and `remove`. vivacity checks the platform by running an
external `php`, never ePHPm's embedded one, and without one those commands exit 1 (`cannot run php: program not found`),
even with `--ignore-platform-reqs`.

It also found the CI footgun. The completely standard

```
composer install --no-dev --no-interaction --prefer-dist -o
```

**exited 2**, because `--no-interaction` and `--prefer-dist` were usage errors on `install`. Other gaps got documented
instead of glossed over: scripts are skipped silently (so Laravel's `package:discover` needs a workaround), `vcs`
repositories are a hard error, and unknown plugins, source installs and path repos on Windows fall back to a real
Composer if one exists.

I'd rather ship a reference page full of caveats that are true than a short one that isn't.

### ePHPm pretends to be php

The platform-check gap had a better fix than documentation. Under `ephpm php`, `PHP_BINARY` is the `ephpm`
executable, which is correct. But Symfony Process, Composer `@php` scripts, PHPUnit process isolation and spatie/async
all re-run `PHP_BINARY script.php`, and that hit ePHPm's own argument parser. Worse, a bare `PHP_BINARY` with a program
on stdin *started the dev server*, since no subcommand means `ephpm dev`.

[#558](https://github.com/ephpm/ephpm/pull/558), still open as I write this, adds a "php personality", decided by a small pure function so the rules
are unit-tested. `ephpm` hands all of argv to the embedded php-cli when:

1. **argv[0] is named like php** (`php`, `php8.4`, `php84`, `.exe` ignored). So `ln -s ephpm /usr/local/bin/php` is a
   complete PHP install.
2. **It inherited an `EPHPM_PHP_CLI=1` marker**, unless argv[1] is a real ePHPm subcommand. `ephpm php` and
   `ephpm composer` export the marker, so their children inherit it, and `ephpm kv ...` called from a PHP script still
   means ePHPm.
3. **argv[1] is php-shaped** with no marker: an option ePHPm doesn't have, or an existing file that isn't a
   subcommand name. Those shapes were all usage errors before, so nothing that used to work changes meaning, and
   `ephpm sevre` still gets "did you mean `serve`".

I rejected pointing `PHP_BINARY` at a generated `php` symlink (filesystem side effects, and it needs privileges on
Windows) and treating every unknown argument as php (you lose the typo diagnostics, and bare `ephpm` has to stay the dev
server).

The payoff is one line: `ephpm composer` sets `VIVACITY_PHP` to its own executable when the build links PHP (and you
haven't set it yourself). vivacity's platform probe then runs on the *embedded* PHP, which is the PHP that actually
serves the app, and once #558 merges, `update`, `require` and `remove` work with no system PHP. The new conformance
case for re-invoking `PHP_BINARY` (script, `-r`, `-d`, stdin, exit status through `proc_open`) is byte-identical to
upstream php 8.5.4.

### Contributing the CI flags

The last piece was making that standard CI line run. Composer accepts `-n`, `-q`, `-v` and `--no-ansi` as global flags
on every command. vivacity only accepted some of them on some commands. The change is small, and it went in
first as a commit on our fork ([ephpm/vivace#2](https://github.com/ephpm/vivace/pull/2), merged) applied through
`[patch.crates-io]` in [#561](https://github.com/ephpm/ephpm/pull/561), together with the `--prefer-dist` upstream
added in 0.19 (the pin moves to 0.20.0). #561 is still open. With both,

```
ephpm composer install --no-dev --no-interaction --no-ansi --prefer-dist -o
```

exits 0 with no PHP on `PATH`. The exact pin stays, so Pin Watch keeps working: a bot bump to a newer crates.io version
leaves the patch unused (Cargo warns), which is the cue to check whether upstream took the change and drop the patch.
Upstream did take it: [vivacity #7](https://github.com/Adelagric/vivacity/pull/7) merged tonight and should be in the
next release, and [#576](https://github.com/ephpm/ephpm/pull/576), a draft, drops the fork in favor of that release once
it's on crates.io.

### How fast is it?

vivacity's own benchmarks were run on macOS and Linux. Since the Windows story was the reason I got involved, I
measured it there: Composer 2.10.3 on PHP 8.3.33 against the released ePHPm v0.11.2 (embedding vivacity 0.17.0),
installing vivacity's Laravel fixture, 109 packages, on native Windows.

![Composer vs ephpm composer on Windows](/img/2026/10/composer-without-php-vivacity-bench.svg)

A fresh install from warm caches (delete `vendor/`, install from the lock) took a median of 107.3 seconds with
Composer and 13.9 seconds with `ephpm composer`. A no-op install was 6.7 seconds versus 0.73, and `dump-autoload -o`
was 6.5 seconds versus 0.38. Then the part that matters more than any of the timings: both `vendor/` trees were 8853
files, and every sha256 matched.

Take the absolute numbers as "Windows with Defender on". Extracting 8853 files is where NTFS and real-time scanning
hurt the most, and both tools pay that. vivacity's advantage is mostly that it extracts each package once into its
store and materializes `vendor/` from there in parallel, where Composer unzips every archive again on every install.
I didn't measure Linux or macOS for this post. Upstream's own numbers from a Mac Studio (`bench/M5-perf.md` in the
vivacity repo) put warm installs at roughly 10x to 14x faster than Composer.

### How I measured

- AMD Ryzen 9 5950X, 128 GB RAM, ADATA LEGEND 860 SSD, Windows 11 Pro (build 26200), Windows Defender real-time
  protection on (as it would be on a developer machine).
- Composer 2.10.3 on the official PHP 8.3.33 NTS x64 build (`openssl`, `mbstring`, `curl`, `zip` enabled), run as
  `php composer.phar <cmd> --no-interaction --no-scripts -q`. `--no-scripts` because vivacity never runs scripts.
- `ephpm.exe` from the v0.11.2 release (`ephpm-v0.11.2+php8.5.7-windows-x86_64.tar.gz`, sha256 checked against
  `SHA256SUMS`), `ephpm composer --version` reporting `vivacity 0.17.0`.
- Project: `fixtures/projects/laravel` from the vivacity repo (Laravel 13 skeleton with its lock). Both tools got
  `--ignore-platform-reqs` on install, since packages in the lock require PHP 8.4.1 and the Composer side ran on 8.3.
- Package caches warm for both (Composer's download cache, vivacity's store), separate project copies per tool.
  "Fresh" deletes `vendor/` before each run; "no-op" re-runs `install -o` on a complete `vendor/`; plus `dump-autoload -o`.
  Wall-clock per command from a bash loop: 3 fresh installs, then 2 no-op installs, then 2 dumps per tool, one tool at
  a time, nothing else benchmarking on the machine. Raw runs (ms):

  | | Composer | ephpm composer |
  |---|---|---|
  | install, `vendor/` deleted | 109975, 99507, 107324 | 14652, 13856, 13364 |
  | install, no-op | 6667, 6790 | 759, 701 |
  | `dump-autoload -o` | 6724, 6374 | 301, 458 |

- Afterwards I compared a sha256 manifest of every file in both `vendor/` trees: 8853 files each, byte-identical.

### Links

- Embed: [ephpm #523](https://github.com/ephpm/ephpm/pull/523) (v0.11.0); Pin Watch: [#532](https://github.com/ephpm/ephpm/pull/532) (v0.11.2)
- Docs rewrite: [#542](https://github.com/ephpm/ephpm/pull/542) (merged, v0.12.0); php personality: [#558](https://github.com/ephpm/ephpm/pull/558) (open); CI flags: [#561](https://github.com/ephpm/ephpm/pull/561) (open), [ephpm/vivace#2](https://github.com/ephpm/vivace/pull/2); dropping the fork: [#576](https://github.com/ephpm/ephpm/pull/576) (draft)
- Upstream, all merged: [vivacity #2](https://github.com/Adelagric/vivacity/pull/2) (Windows, 0.7.0), [#4](https://github.com/Adelagric/vivacity/pull/4) (parallel install, 0.9.0), [#5](https://github.com/Adelagric/vivacity/pull/5) (TLS backend, 0.9.0), [#6](https://github.com/Adelagric/vivacity/pull/6) (pure-Rust extraction, 0.12.0), [#7](https://github.com/Adelagric/vivacity/pull/7) (global CI flags, merged, not released yet)
- [vivacity](https://github.com/Adelagric/vivacity) and [Composer](https://getcomposer.org/)
