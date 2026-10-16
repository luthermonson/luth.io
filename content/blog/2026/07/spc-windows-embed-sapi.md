---
title: "Contributing Windows Embed SAPI Support to static-php-cli (SPC)"
description: "Two closed PRs, ten release candidates in a day, and two php-src PRs: getting a static php8embed.lib out of static-php-cli on Windows."
date: 2026-07-26T19:45:00-07:00
tags: [ php, windows, static-php-cli, msvc, embed, open-source, ephpm ]
categories: [ php, windows ]
images:
 - src: img/2026/07/spc-windows-embed-sapi-fat-lib.svg
   alt: "Diagram comparing the default thin php8embed.lib that needs php8ts.dll with a fat static php8embed.lib linked into a single ephpm.exe"
   stretch: "none"
draft: true
---

<!-- TODO(luther): php/php-src#22166 is still open. Flip draft to false once it merges (or once you're happy publishing with it open) and update the last section. -->

[ePHPm](https://github.com/ephpm/ephpm) embeds PHP inside a Rust binary. On Linux and macOS that means linking against a
static `libphp.a`, and the tool everyone uses to produce one is [static-php-cli](https://github.com/crazywhalecc/static-php-cli)
(SPC). It builds PHP and every library it depends on from source and hands you a static binary, or with `--build-embed`, a
static library and headers. On Windows, though, the embed SAPI was a TODO. So I went and did it, and it turned into a
four-month lesson in where a fix is supposed to live.

![PHP logo](/img/2026/07/spc-windows-embed-sapi-php-logo.svg)

<small>PHP logo by Colin Viebrock, [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), via
[php.net/logos](https://www.php.net/download-logos.php) and
[Wikimedia Commons](https://commons.wikimedia.org/wiki/File:PHP-logo.svg).</small>

### The stub

In SPC v2 (the `main` branch back in April), `WindowsBuilder::buildPHP()` hit the embed flag and did this:

```php
if ($enableEmbed) {
    logger()->warning('Windows does not currently support embed SAPI.');
    // logger()->info('building embed');
    $this->buildEmbed();
}
```

and `buildEmbed()` was a commented-out sketch:

```php
public function buildEmbed(): void
{
    // TODO: add embed support for windows
    /*
    FileSystem::writeFile(SOURCE_PATH . '\php-src\nmake_embed_wrapper.bat', 'nmake /nologo %*');

    cmd()->cd(SOURCE_PATH . '\php-src')
        ->exec("{$this->sdk_prefix} nmake_embed_wrapper.bat --task-args php8embed.lib");
    */
}
```

The sketch had the right idea. PHP's Windows build already knows how to make `php8embed.lib`, you just have to ask nmake
for that target and then put the output somewhere useful.

### The first PR (v2)

Over a weekend in early April I filled it in and opened [#1079](https://github.com/crazywhalecc/static-php-cli/pull/1079).
The core of it is an nmake wrapper with the libraries the embed target actually needs:

```php
public function buildEmbed(): void
{
    $extra_libs = getenv('SPC_EXTRA_LIBS') ?: '';

    // Add debug symbols for release build if --no-strip is specified
    $debug_overrides = '';
    $makefile_content = file_get_contents(SOURCE_PATH . '\php-src\Makefile');
    if ($this->getOption('no-strip', false)) {
        if (preg_match('/^CFLAGS=(.+?)$/m', $makefile_content, $matches)) {
            $cflags = str_replace('/Ox ', '/O2 /Zi ', $matches[1]);
            $debug_overrides = '"CFLAGS=' . $cflags . '" "LDFLAGS=/DEBUG /LTCG /INCREMENTAL:NO secur32.lib" ';
        }
    }
    // Needed by curl's SSPI support (InitSecurityInterfaceA in libcurl_a.lib).
    if ($debug_overrides === '') {
        $debug_overrides = '"LDFLAGS=secur32.lib" ';
    }

    FileSystem::writeFile(SOURCE_PATH . '\php-src\nmake_embed_wrapper.bat',
        "nmake /nologo {$debug_overrides}LIBS_EMBED=\"ws2_32.lib shell32.lib {$extra_libs}\" %*");

    cmd()->cd(SOURCE_PATH . '\php-src')
        ->exec("{$this->sdk_prefix} nmake_embed_wrapper.bat --task-args php8embed.lib");

    $this->deploySAPIBinary(BUILD_TARGET_EMBED);
    // ... then copy headers (below)
}
```

`secur32.lib` was the one that took a minute. Static curl on Windows uses SSPI, so `InitSecurityInterfaceA` has to come from
somewhere, and nothing else in the link pulls it in.

Then `deploySAPIBinary()` needed to learn that embed isn't like the other SAPIs. `php.exe`, `php-cgi.exe` and `micro.sfx`
all go to `buildroot/bin` with a `.pdb` next to them. The embed output is a library, so it goes to `buildroot/lib`, it has
no PDB of its own, and the build also produces a `php8embed.dll` that should travel with it:

```php
BUILD_TARGET_EMBED => [SOURCE_PATH . "\\php-src\\x64\\{$rel_type}{$ts}", 'php8embed.lib', null],
// ...
$dst = ($type === BUILD_TARGET_EMBED ? BUILD_ROOT_PATH . '\lib' : BUILD_BIN_PATH) . '\\' . basename($src);
```

The last piece was headers. A library is useless to a consumer like ePHPm's Rust FFI build without the headers to generate
bindings against, and on Windows that includes the generated `config.w32.h`:

```php
$include_dst = BUILD_ROOT_PATH . '\include\php';
foreach (['main', 'Zend', 'TSRM', 'sapi', 'ext'] as $dir) {
    $src_dir = "{$php_src}\\{$dir}";
    if (is_dir($src_dir)) {
        cmd()->exec('xcopy /E /I /Y ' . escapeshellarg($src_dir . '\*.h') . ' ' . escapeshellarg($include_dst . '\\' . $dir . '\\'));
    }
}
$config_h = "{$php_src}\\x64\\{$rel_type}{$ts}\\config.w32.h";
if (file_exists($config_h)) {
    cmd()->exec('copy ' . escapeshellarg($config_h) . ' ' . escapeshellarg($include_dst . '\main\config.w32.h'));
}
```

The same PR carried two other Windows fixes I needed to get that far: an
[archive extraction fix](/blog/2026/07/spc-windows-archive-extraction/) that got its own post, and a patch to
`ext/libxml/php_libxml2.def` so `php8ts.dll` would link against a current libxml2, which deserves one too.

### "Embarrassingly, I have already implemented..."

The maintainer's reply came within the hour, and it was the most polite possible version of bad news: Windows embed was
already done in **v3**, a full rewrite living on a development branch, along with fixes for the tar and SDK wrapper problems
I'd hit. I'd spent my weekend on a branch that was about to stop being maintained. I didn't know v3 existed, and v3 itself
still had the same `// TODO` in its old `src/SPC` tree, because the new code lives in a different directory (`src/StaticPHP`).

That's on me for not looking around before I started. I said I'd keep running my fork until v3 was further along, and the
PR was closed in mid-April. If there's a lesson in this part it's a boring one: before you implement a missing feature in
someone else's project, read the open PRs and the branches, not just `main`.

### v3, and two days of release candidates

When I moved my pipeline to v3 in May, the embed target existed, but a *static* embed build for a host program wasn't right. v3 already patched the nmake target to build a fat `php8embed.lib` containing the PHP core
and every static extension instead of a thin wrapper around an import library. The problem was how `php_embed.c` itself was
compiled.

PHP's Windows build compiles SAPIs as **consumers of the PHP DLL**. Every PHP, Zend, SAPI and TSRM API they call is declared
`__declspec(dllimport)`, so `php_embed.obj` references `__imp_php_execute_script` instead of `php_execute_script`. Pack that
object into a static library next to the real definitions and the linker has to paper over the mismatch. The existing smoke
test got it to link with `/FORCE:MULTIPLE`, which is the MSVC way of saying "I know, do it anyway".

My fork's release tags tell this story better than I can. From `v3.0.0-pgo-rc6` on the evening of May 25th to
`v3.0.0-pgo-rc15` the next night, in order:

- **rc6**: the smoke test link was missing `pathcch.lib` (`PathCchCanonicalizeEx` from PHP's `ioutil.obj`).
- **rc7, rc8**: compile the smoke test and then `CFLAGS_EMBED` with `/D PHP_EXPORTS /D LIBZEND_EXPORTS /D SAPI_EXPORTS /D TSRM_EXPORTS`, so the embed
  objects reference symbols directly instead of through import thunks.
- **rc9**: that broke ZTS. `_tsrm_ls_cache` is a `__declspec(thread)` variable, and with `TSRM_EXPORTS` set the extern
  `dllexport` declaration produces a definition in `php_embed.obj` too, duplicating the one in `zend.obj`. LNK4006, then
  LNK4088 "image may not run" under `/FORCE:MULTIPLE`. So drop `TSRM_EXPORTS`.
- **rc10**: the real source of the duplicate is `php_embed.c` calling the TSRM cache define macro itself, because in a
  DLL world each SAPI gets its own. In one static binary there should be exactly one. Patch it out, restore `TSRM_EXPORTS`.
- **rc11**: still duplicated, so drop `TSRM_EXPORTS` again.
- **rc12**: the actual bug in rc10. `php_embed.c` spells it `ZEND_TSRMLS_CACHE_DEFINE()`, not `TSRMLS_CACHE_DEFINE()`, and has for years. My
  `replaceFileStr` had been searching for a string that wasn't there, and rc10 and rc11 were testing a patch that did nothing.
- **rc13**: with the define really gone, restore `TSRM_EXPORTS` and remove `/FORCE:MULTIPLE` from the smoke test so any
  remaining duplicate is a hard error instead of a silent one.
- **rc14**: an `&argv` typo in the smoke test.
- **rc15**: the smoke test itself. The `fprintf(stderr, ...)` breadcrumbs I'd added in rc13 got captured alongside stdout,
  and the check was `trim($output) !== 'hello'`, so a *working* binary failed validation with exit code 0.

Ten tags in about 24 hours. The end state is two small source patches:

```php
// Embed SAPI objects are compiled as DLL consumers (dllimport for PHP/Zend APIs).
// For our fat static lib, they need direct linkage.
$content = preg_replace(
    '/^CFLAGS_EMBED=(.+)$/m',
    'CFLAGS_EMBED=$1 /D PHP_EXPORTS /D LIBZEND_EXPORTS /D SAPI_EXPORTS /D TSRM_EXPORTS',
    $content,
    1
);

// In DLL builds each SAPI has its own _tsrm_ls_cache via TSRMLS_CACHE_DEFINE().
// In our fat static lib all objects share one binary, so the duplicate TLS definition
// from php_embed.c collides with the one in zend.c. Remove it so php_embed.obj uses the core's copy.
$embed_src = "{$package->getSourceDir()}\\sapi\\embed\\php_embed.c";
if (file_exists($embed_src)) {
    FileSystem::replaceFileStr($embed_src, 'ZEND_TSRMLS_CACHE_DEFINE()', '/* removed for static embed */');
}
```

and a smoke test that no longer needs `/FORCE:MULTIPLE`:

```c
int main(int argc, char **argv) {
    if (php_embed_init(argc, argv) == FAILURE) {
        return 1;
    }

    zend_first_try {
        zend_file_handle file_handle;
        zend_stream_init_filename(&file_handle, "embed.php");
        php_execute_script(&file_handle);
    } zend_end_try();

    php_embed_shutdown();
    return 0;
}
```

### The second PR, and the right layer

On the evening of May 27th I opened [#1176](https://github.com/crazywhalecc/static-php-cli/pull/1176) against v3 with those patches, plus
[#1173](https://github.com/crazywhalecc/static-php-cli/pull/1173) for a related PHP 8.5 problem (the new `uri` extension
needs `LEXBOR_STATIC` defined to link statically on Windows).

The review on #1173 was short: this should be fixed in php-src, not patched by a build tool. Which is correct. SPC rewriting
`php_embed.c` and the generated Makefile on every build is a workaround for something PHP's own `configure.js` could just do.
Unix already supports `--enable-embed=static` in `sapi/embed/config.m4`; the Windows `config.w32` ignored the `static` value.

So I closed both and went upstream:

- [php/php-src#22165](https://github.com/php/php-src/pull/22165), `ext/uri`: define `LEXBOR_STATIC` for static Windows builds.
  Merged the next day.
- [php/php-src#22166](https://github.com/php/php-src/pull/22166), honor `--enable-embed=static` on Windows: add the export
  defines to `CFLAGS_EMBED` and skip the TSRM cache define in `php_embed.c` behind a new `PHP_EMBED_STATIC`.

The first review on #22166 pushed it further. A `--enable-embed=static` that only fixes the `dllimport`s but still needs a
downstream tool to assemble the fat library isn't a feature anyone can test. If the flag exists, it should produce the real
static library. So the Makefile composition SPC was doing moved into php-src too.

### Where it stands

In mid-July I posted end-to-end validation on #22166: PHP 8.5 from my branch, built with
`--enable-embed=static --enable-zts` and about 40 static extensions through the same SPC-based CI pipeline, and a host
program linked against the resulting library plus the usual Windows system libs, and no PHP DLL:

```c
PHP_EMBED_START_BLOCK(argc, argv)
zend_eval_string("echo 'EMBEDOK ' . PHP_VERSION . ' zts=' . (ZEND_THREAD_SAFE ? '1' : '0') . ' sapi=' . php_sapi_name();", NULL, "smoke");
PHP_EMBED_END_BLOCK()
```

```
EMBEDOK 8.5.8-dev zts=1 sapi=embed
```

One executable, no PHP DLL anywhere at runtime. The NTS variant of the same change has been ePHPm's Windows build path since
May. The PR is still open as I write this.

<!-- TODO(luther): update this paragraph with the php-src#22166 outcome. -->

### What I'd tell myself in April

- **Look at every branch before you write code.** The feature I needed was already written, just not where I looked.
- **Patching the thing you build is a smell.** If your build tool `str_replace`s a C file in the project it builds, the fix
  probably belongs in that project. It took a maintainer saying so for me to go do it.
- **Make the test fail loudly.** `/FORCE:MULTIPLE` turned a duplicate symbol into a binary that linked and then misbehaved,
  and `trim() !== 'hello'` turned a working binary into a failure. Both cost more time than the actual bug.

### Links

- [crazywhalecc/static-php-cli#1079](https://github.com/crazywhalecc/static-php-cli/pull/1079) - v2 Windows embed PR (closed, superseded by v3)
- [crazywhalecc/static-php-cli#1176](https://github.com/crazywhalecc/static-php-cli/pull/1176) - v3 static embed PR (closed, moved to php-src)
- [php/php-src#22165](https://github.com/php/php-src/pull/22165) - `LEXBOR_STATIC` for static Windows builds
- [php/php-src#22166](https://github.com/php/php-src/pull/22166) - `--enable-embed=static` on Windows
- [ePHPm](https://github.com/ephpm/ephpm)
