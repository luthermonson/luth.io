---
title: "The Fast PHP Interpreter Windows Never Got"
description: "Windows PHP being slow is mostly a missing-implementation story, not an OS fact. Here is why ePHPm ships a separate clang-cl build with the TAILCALL interpreter, and what it proves."
date: 2026-08-21T10:00:00-07:00
tags: [ php, windows, ephpm, performance, rust ]
categories: [ ephpm, php ]
images:
 - src: img/2026/08/windows-php-fast-vm-tailcall.svg
   alt: "Bar chart of upstream bench.php run time for three Zend dispatch kinds on one machine: CALL (MSVC, the slow fallback) at 1.783s, HYBRID (GCC, default since 7.2) at 1.006s, and TAILCALL (Clang, new in PHP 8.5) at 1.017s, showing TAILCALL recovers almost all of HYBRID's advantage over CALL"
   stretch: "none"
draft: false
---

"PHP is slow on Windows" is one of those things everyone knows and nobody questions. I believed it too, right up until I went looking for *where* the slowness actually lives. Part of it is real and unfixable. But a big chunk of the interpreter half turns out to be something much more boring: nobody ever wrote the fast dispatch loop for the Windows compiler. That is not the same claim as "the OS is slower." It is a missing implementation.

[ePHPm](https://github.com/ephpm/ephpm) embeds PHP into a single Rust binary, so I ship my own Windows PHP build and I care a lot about how fast it runs. That is how I ended up with a second, separate Windows binary that closes most of the interpreter gap to Linux. To explain why it has to be a separate binary, I have to explain how the Zend interpreter actually dispatches an opcode.

### Three ways to run a bytecode loop

PHP compiles your script to opcodes and then runs them in a loop. For each opcode it has to pick the handler, run it, and move to the next. How it does that pick is the "VM kind," and the Zend engine has three that actually ship.

**CALL** is the portable one. Every opcode handler is its own function, and the loop makes one indirect function call per opcode. That sounds fine until you count the cost. Every call pays a full register-save prologue on the way in, re-loads the interpreter's state from memory, does its work, then restores registers on the way out. You pay that tax on every single opcode. It is the fallback that works on any C compiler, and it is what MSVC produces.

**HYBRID** is the fast one. Instead of separate functions, all the handlers are blocks inside one giant function, and dispatch is a computed goto straight to the next block. No call, no prologue, no epilogue. On top of that, the two hottest interpreter variables — the current execution frame and the instruction pointer — get pinned into CPU registers for the entire life of the process. They never get reloaded from memory because they never leave the register. This is the default on Linux and has been since PHP 7.2. It is also a GCC-only trick: computed goto and global register variables are GCC extensions.

**TAILCALL** is the new one, added in PHP 8.5. Handlers stay separate functions, but each one ends by tail-calling the next handler directly, using Clang's `[[clang::musttail]]` and the `preserve_none` calling convention. The guaranteed tail call means no stack growth and no save/restore churn between handlers, and `preserve_none` lets the compiler stop babysitting callee-saved registers. The result lands within about one percent of HYBRID. It is Clang-only.

There are two more kinds, SWITCH and GOTO, but they are legacy generator modes and nobody ships them pre-generated, so ignore them.

Here is what the three shipped kinds cost on the same machine, from the upstream benchmark that introduced TAILCALL:

![Bar chart of bench.php run time for CALL, HYBRID and TAILCALL on one machine: CALL 1.783s, HYBRID 1.006s, TAILCALL 1.017s](/img/2026/08/windows-php-fast-vm-tailcall.svg)

CALL is about 1.77x slower than HYBRID on this opcode-bound benchmark. TAILCALL claws essentially all of that back. The important detail is that this is one machine, one OS. It is a pure compiler-and-dispatch effect. Nothing about Windows is in that number.

### Why MSVC gets the slow one

The VM kind is not configured. It is selected by the C preprocessor when php-src is compiled, by a chain that boils down to this: if you are on GCC with global register variables, you get HYBRID; else if you are on Clang with musttail and preserve_none on x86-64 or arm64, you get TAILCALL; otherwise you get CALL.

MSVC is the "otherwise." It does not define `__GNUC__`, it has no labels-as-values, it has no global register variables, and it has no guaranteed-tail-call attribute. So both fast arms fail and MSVC falls through to CALL. Every stock Windows PHP build is in this bucket. The php.net zip, FrankenPHP's Windows bundle, my own regular ePHPm Windows artifact — all of them run CALL, because all of them are built with MSVC, and MSVC is the only compiler PHP officially supports on Windows. When I benchmarked the php.net build and the FrankenPHP build on the same CPU loop they came out within a rounding error of each other, which makes sense: same interpreter lineage, same slow dispatch.

So "Windows PHP is slow" in the interpreter is really "the supported Windows compiler can't emit the fast interpreter, and until 8.5 there was no fast interpreter it *could* emit." That second clause is the part that changed. TAILCALL was built for exactly the compiler Windows can actually use.

### You can't flip it on — so I ship a second binary

This is the part that trips people up, so let me say it plainly. There is no `opcache.vm_kind`. There is no `php.ini` setting and no CLI flag. The VM kind is baked into the binary by how it was compiled. You cannot turn TAILCALL "on" in an MSVC build, because the MSVC build does not contain the TAILCALL code at all. The preprocessor compiled it out.

That is why ePHPm ships TAILCALL as a *separate* Windows download rather than a config knob. It is a whole second binary, built with clang-cl instead of MSVC. clang-cl is Clang wearing the MSVC command-line interface, so it produces a Microsoft-ABI object that links into my unchanged Rust pipeline, but it qualifies for the Clang arm of that preprocessor chain and emits TAILCALL. The regular MSVC artifact stays the default. The clang build sits next to it with a `-tailcall` suffix for people doing CPU-heavy work on PHP 8.5.

### What it's actually worth

Measured end to end through ePHPm on Windows, on a pure CPU loop with the JIT off, the TAILCALL build runs the interpreter about 1.72x faster than the MSVC build. Roughly 4.8ms drops to roughly 2.8ms. That matches the upstream dispatch number almost exactly, which is reassuring: the win is the dispatch kind, nothing fancier.

Now the honest part. That full 1.72x lands with the JIT off. That happens to be ePHPm's shipped default on every platform right now, so for ePHPm the number is real and it is the common case, not a corner case. But I want to be clear about why, because it is easy to oversell.

The JIT compiles hot code to native machine code that bypasses the C interpreter loop entirely. When hot code is running JIT-compiled, the dispatch quality of the interpreter stops mattering, because you are not in the interpreter. So on a warm hot loop with the tracing JIT on, the MSVC build and the TAILCALL build land on top of each other. TAILCALL's value with a warm JIT is not steady-state throughput. It is cold-path latency: framework bootstrap, first-request code, short scripts, anything that runs the interpreter before the JIT has traced it. There TAILCALL still wins by about 1.55x even with the JIT enabled.

So the reframe I want to land is this. Every platform already runs the fastest interpreter its compiler can emit. Linux on GCC gets HYBRID. Clang targets — Windows through clang-cl, and macOS — get TAILCALL. There is no reason to switch Linux to TAILCALL, because HYBRID is already as fast and is the GCC path. TAILCALL was never a project-wide upgrade. It was built specifically for the platforms GCC can't serve, where the only alternative was the slow CALL fallback. Windows is the headline example. The reputation was never "the OS is slower." It was "the Windows toolchain was never handed a fast VM." Now, on 8.5 and later, it has one.

### The other half is the filesystem, and that one's real

I don't want to leave you thinking a clang build fixes Windows PHP, because it only fixes half the story, and arguably the smaller half for a real app.

The other half is filesystem metadata. A `stat` or a `file_exists` on Windows costs roughly 10x what it costs on Linux. I measured a deep `vendor/` walk with a couple thousand existence checks at about 207ms on Windows against about 20ms on Linux, same hardware. That is not Defender, and it is not a tuning mistake. Every one of those calls pays a per-call UTF-8 to UTF-16 conversion, a path canonicalization, and a full `CreateFileW` handle open that walks the whole NT filter-driver stack. The Linux side is one syscall served out of the dentry cache. A framework that stats hundreds of files per request multiplies that structural difference straight into your request time.

And here is the thing: that half has no compiler fix. There is no second binary that makes a `stat` cheap. You can only pay it less often — production opcache settings, authoritative autoloader classmaps, Dev Drive, turning off timestamp validation. The first touch of every file is always full price. On a filesystem-bound app the interpreter win is a couple of percent and the filesystem is the whole game.

Which brings me to the thing I actually can't stop thinking about. The metadata cost is bad on a local NTFS volume. It gets genuinely ugly over a network filesystem, and NFS performance for this kind of stat-heavy, metadata-chatty PHP workload is its own special misery that I have been staring at for a while now. I don't have an answer for it yet. It is an open problem I would like to crack one day, the same way the interpreter half turned out to be crackable once I stopped treating "Windows is slow" as a law of nature and went looking for the actual mechanism. No promises, no roadmap line. Just a hope that one day the network-filesystem half has as clean an answer as the interpreter half finally got.

### What's next

The TAILCALL lane is still experimental and non-gating in CI, so it has less soak time than the MSVC artifact. If you run CPU-bound PHP on Windows, grab the `-tailcall` build, run your own test suite against it, and tell me what breaks. The mechanism is upstream and solid; the packaging is where I still want more miles. And if you have ever made PHP behave over NFS, I would genuinely love to hear how.

### Links

- [php-src PR #17849 — the tail-call VM](https://github.com/php/php-src/pull/17849) — the upstream change that added TAILCALL in PHP 8.5, with the bench.php numbers (HYBRID 1.006s / CALL 1.783s / TAILCALL 1.017s).
- [Dmitry Stogov introducing HYBRID](https://externals.io/message/99043) — the php-internals thread for the GCC dispatch that has been the Linux default since PHP 7.2.
- [ephpm/ephpm#329](https://github.com/ephpm/ephpm/issues/329) — the clang-cl TAILCALL lane for ePHPm's Windows builds, with the measured end-to-end numbers.
