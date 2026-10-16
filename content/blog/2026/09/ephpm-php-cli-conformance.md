---
title: "Making ephpm php Byte-for-Byte Indistinguishable From php-cli"
description: "A differential test harness against upstream php, and the bugs it found in an embedded PHP CLI on day one."
date: 2026-09-10T20:25:00-07:00
tags: [ php, rust, ephpm, testing, cli ]
categories: [ ephpm, php ]
images:
 - src: img/2026/09/ephpm-php-cli-conformance-results.svg
   alt: "Bar chart of conformance corpus results: 65 pass and 10 xfail when the harness landed, 79 pass and 1 xfail after the fixes"
   stretch: "none"
draft: false
---

[ePHPm](https://github.com/ephpm/ephpm) embeds PHP into a single Rust binary, and the obvious follow-up question from
anybody deploying it is "do I still need a `php` on the box?" Composer, WP-CLI, artisan, PHPUnit and every build
script in the world call `php`. If `ephpm php` can stand in for it then the answer is no, and the single binary actually
means a single binary. "Mostly works" isn't good enough for that though. These tools parse PHP's output, check its exit
codes and re-invoke it. So the goal was simple to state: `ephpm php` should be indistinguishable from php-cli.

### Getting to "drop-in" the first time

The first pass ([#306](https://github.com/ephpm/ephpm/pull/306)) made `ephpm php` report `PHP_SAPI === "cli"`
(set before `php_embed_init()` so OPcache's startup verdict sees it), read programs from stdin, honor `-d`, `-c` and
`-n`, support line mode (`-B`, `-R`, `-F`, `-E`), define the `STDIN`/`STDOUT`/`STDERR` constants WP-CLI needs, and
actually implement the `--rf`/`--rc`/`--re`/`--rz`/`--ri` reflection flags that had been declared and silently did
nothing. `composer.phar create-project laravel/laravel` completed clean and `wp-cli.phar --version` was byte-identical
to the system php.

A couple of things are honest errors rather than fakes. `-S` isn't aliased to `ephpm serve`, because the
embedded `libphp.a` contains zero `php_cli_server_*` symbols and pretending otherwise would just move the surprise. `-a`
says "not supported" because the interactive shell isn't linked into the embed build.

The second pass ([#317](https://github.com/ephpm/ephpm/pull/317)) was the tedious one: checking every rule against
`sapi/cli/php_cli.c` and every output against a real php 8.5.4. Stdin programs with `--` args, `$argv[0]` being
`"Standard input code"` (not `"-"`, which our docs had claimed matched php-cli and never did), exit codes taken from
`EG(exit_status)` unconditionally so `exit(3)`, fatals and uncaught exceptions stopped all reporting 0, `-w` and `-s`
reading stdin instead of silently *executing* a piped program, and `$argi` being 1-based.

That PR was verified with a throwaway diff script over 55 cases. Throwaway is the problem. The next regression
wouldn't get caught by anything.

### The differential harness

[#342](https://github.com/ephpm/ephpm/pull/342) turned that into a real harness. `tests/cli-conformance/` holds the
corpus. The `0xx` cases cover the language and runtime surface (var_dump of every type, float formatting, INF/NAN,
sprintf, mbstring, sort stability, JSON edge cases, serialize, preg backtrack limits, uncaught exception format,
shutdown and destructor ordering, enums, readonly and so on). The `1xx` cases cover CLI behavior: exit codes,
`-r` with and without `--`, `-l`, `-d`, `$argv`/`$argc`/`$_SERVER` in every mode, `PHP_BINARY`, shebang stripping, line
mode, `-w`, `-s`, `-v`, `--rf`. A case is a `.php` file or an `.args` file, optionally with `.stdin` and `.meta`:

```
# 115-php-binary.args
-r
echo PHP_BINARY === '' ? "(empty)" : PHP_BINARY, "\n"; var_dump(is_string(PHP_BINARY));
```

The runner is `cargo xtask cli-conformance --php <upstream> --ephpm <bin>`. It runs each case against both binaries
with an identical minimal environment (`TZ=UTC`, `LC_ALL=C`, a per-side scratch directory) and compares stdout, stderr
and exit code byte for byte. Any unexplained mismatch prints a unified diff and fails the run. A nightly workflow runs
it over PHP 8.3, 8.4 and 8.5 against the matching upstream php from the ondrej PPA.

Three design choices in the runner are the reason I trust it.

**It checks the binary's identity before running anything.** `<ephpm> php -n -v` has to print a PHP banner first, so
handing a plain `php` to `--ephpm` fails immediately. This isn't paranoia. A `$VAR` expansion bug in the WSL bridge
had repeatedly made earlier harnesses test the system php against itself, which passes beautifully.

**Expected failures are marked, and a stale marker fails the run.** A known divergence gets an `xfail` with a reason
and an issue number. If an xfail case starts matching, the runner reports XPASS and goes red until somebody deletes the
marker. Markers can't quietly rot into lies.

**Normalization is a tiny, justified registry.** There are exactly four normalizers (`strip-versions`,
`strip-zts-marker`, `strip-distro-banner`, `strip-binary-path`), and every use has to explain itself in the `.meta`
file:

```
# 115-php-binary.meta
# strip-binary-path: the two sides are different executables by definition,
# so each side's own binary path is folded to <BINARY>; the case then proves
# PHP_BINARY is a non-empty string naming the invoked binary on BOTH sides.
normalize = ["strip-binary-path"]
```

Things that are packaging facts rather than behavior, like the `-m` extension list of a static SDK versus a distro
php, or `--ini` search paths, get `skip` with a reason.

### What it found on day one

Against a build of main, the first real run was `total 77: 65 pass, 0 fail, 2 skip, 10 xfail`. Seven of those xfails
were new, real divergences, all filed as issues:

| Issue | Divergence |
|---|---|
| #334 | `register_shutdown_function` callbacks and end-of-script destructors **never ran** |
| #335 | `--rf` on a missing symbol exited 0 (php-cli: 1) |
| #336 | an unknown option (`-Z`) was silently ignored, exit 0 |
| #337 | `-d` of unregistered ini entries invisible to `get_cfg_var()` |
| #338 | CLI `$_SERVER` lacked environment variables and `DOCUMENT_ROOT` |
| #339 | `PHP_BINARY` was `""` |
| #340 | `STDIN`/`STDOUT`/`STDERR` lacked `php://` wrapper metadata |

The first one is the scary one. A CLI that never runs shutdown functions or destructors breaks every tool that flushes
a buffer or writes a lock file on the way out. The root cause ([#347](https://github.com/ephpm/ephpm/pull/347)) was
that `ephpm_cli_main` never called `php_request_shutdown()`. Teardown ran invisibly at process exit, into the
discarded HTTP capture buffer, after the exit code had already been taken. The fix runs shutdown in php-cli's exact
order, with nested-bailout protection, and takes the exit status afterwards so `exit()` inside a shutdown function
works.

### The fixes, checked against php_cli.c

[#356](https://github.com/ephpm/ephpm/pull/356) took the next five. A few of them are worth the detail.

**`-Z` was accepted silently.** The flag fell through both getopt passes, no mode got selected, and the CLI read empty
stdin as a program. A typo'd flag ran nothing and reported success. Now `php_getopt` is called with `show_err = 1`
like php-cli's `main()`, so stderr gets the exact `Error in argument 2, char 2: option not found Z` and the exit is 1.
That had a consequence: the option table is now load-bearing, since any option php-cli accepts that the table lacks
turns a working command line into a failure. So the table was diffed against php-src's `OPTIONS[]`, and the one missing
entry (`--syntax-highlighting`, the long spelling of `-s`) was added. The one deliberate gap is php-cli's internal
`--repeat`, which php-src itself labels a testing option; accepting and ignoring it would silently run a script once
when the caller asked for N times.

**`PHP_BINARY` was empty on Unix.** On Windows `php_binary_init()` asks the OS, which is why it looked fine there. On
everything else it reads `sapi_module.executable_location`, and `php_embed_init()` assigns that `argv[0]`, which ePHPm
passes as the bare string `"ephpm"`. PHP then searches `PATH` for something named `ephpm`, finds nothing and leaves it
NULL. The trap is *where* to set it. Set it before `php_embed_init()` and it gets overwritten. Set it after
`php_module_startup()` and it's too late. The only window is inside ePHPm's startup shim, right before it hands off:

```c
static int ephpm_module_startup(sapi_module_struct *sm)
{
    sm->additional_functions = g_cli_mode ? ephpm_cli_functions : ephpm_kv_functions;

    /* PHP_BINARY: php_embed_init() has just overwritten executable_location
     * with argv[0] ("ephpm"), and php_module_startup() below both copies *sm
     * into sapi_module and calls php_binary_init(). NULL outside `ephpm php`,
     * so the server's PHP_BINARY is unchanged. */
    if (cli_executable_location) {
        sm->executable_location = cli_executable_location;
    }
    /* ... -d splice, worker module ... */
}
```

The first cut set the field pre-init, built clean, and passed on Windows while staying broken everywhere else. The
Linux binary is what caught it, which is a good argument for running the harness on the platform where the code path
actually differs. The value recorded is `std::env::current_exe()`, so it names *this* binary rather than whatever
`ephpm` is first on `PATH`.

That same `ephpm_module_startup()` window is where `-d` now goes too ([#341](https://github.com/ephpm/ephpm/pull/341)).
OPcache decides once, at module startup, whether it will ever be active in the CLI. We were applying `-d` after
`php_embed_init()`, so `ini_get('opcache.enable_cli')` said `1` while `opcache_get_status()` said `false`. Splicing the
`-d` entries into `ini_entries` right there, after the embed SAPI's hardcoded block, gives php-cli's exact precedence.
It also means `-d extension=` and `-d zend_extension=` genuinely load now.

**`$_SERVER` in the CLI lacked the environment.** php-cli imports the process environment into `$_SERVER` (the "S"
of `EGPCS` in a CLI process is its environment) and registers an empty `DOCUMENT_ROOT`. That's now done, but only on the
CLI path. `ephpm serve` is multi-tenant and the process environment holds cross-tenant material, so it must never reach
a web request's `$_SERVER`. That was verified by starting the server with a probe variable in its environment and
confirming no request could see it. Being byte-compatible doesn't mean copying every php-cli behavior into the
server.

After #356 the run was `total 78: 75 pass, 0 fail, 2 skip, 1 xfail`. The remaining xfail is permanent and intentional:
the usage screen says `Usage: ephpm php` and omits flags this build doesn't implement. The part that matters, the
stderr diagnostic and exit status, is pinned byte-for-byte from the ePHPm side by an e2e test.

![CLI conformance results](/img/2026/09/ephpm-php-cli-conformance-results.svg)

### The last mile: reflection flags

The most recent fix ([#481](https://github.com/ephpm/ephpm/pull/481)) is a good example of how literal "byte for
byte" gets. `--rf ArrayObject::count` was printing `Exception: Function ArrayObject::count() does not exist` and exiting
1. php-cli prints the method, exits 0, and on 8.5 also prints this first, on stdout:

```
Deprecated: Calling ReflectionMethod::__construct() with 1 argument is deprecated,
use ReflectionMethod::createFromMethodName() instead in Unknown on line 0
```

That's php-cli's own deprecation, from php-cli's own code. Matching it meant giving up on evaluating an
`echo new ReflectionX($name)` snippet. php-cli constructs the reflection object under a zeroed execute-data frame,
and that frame is exactly what produces `in Unknown on line 0`. Evaluated code stamps the eval's filename and line, so no
`zend_eval_string` shape can produce those bytes. The five reflection flags now run php_cli.c's algorithm in C,
including `--ri main` printing the core ini table and `--ri nosuchext` exiting 1. The corpus is at 83 cases:
79 pass, 3 skip, 1 xfail.

### What's still open

`PHP_BINARY` now names the `ephpm` executable, which is correct and creates the next problem. Tools that re-invoke PHP
through it, like PHPUnit process isolation, Symfony's `PhpExecutableFinder` and Composer `@php` scripts, run
`PHP_BINARY script.php`, and `ephpm script.php` isn't a PHP CLI. For now that's documented with a wrapper-script
workaround. Making `ephpm` answer as php when it's re-invoked that way is next on the list.

The harness is the part I'd reuse on any project that claims compatibility with something else. Diff against the real
thing, compare bytes not vibes, make the binary prove its identity, and make every exception to the rule explain itself
in a file that fails the build when it stops being true.

### Links

- `ephpm php` drop-in: [#306](https://github.com/ephpm/ephpm/pull/306), [#317](https://github.com/ephpm/ephpm/pull/317)
- Conformance harness: [#342](https://github.com/ephpm/ephpm/pull/342)
- Shutdown functions and destructors: [#347](https://github.com/ephpm/ephpm/pull/347)
- `-d` at startup, process titles: [#341](https://github.com/ephpm/ephpm/pull/341)
- Exit codes, unknown options, `$_SERVER`, `PHP_BINARY`, stdio metadata: [#356](https://github.com/ephpm/ephpm/pull/356)
- Reflection flags: [#481](https://github.com/ephpm/ephpm/pull/481)
- [php-src `sapi/cli/php_cli.c`](https://github.com/php/php-src/blob/master/sapi/cli/php_cli.c)
