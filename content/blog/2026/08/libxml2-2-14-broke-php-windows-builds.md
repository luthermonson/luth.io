---
title: "libxml2 2.14 Broke PHP Windows Builds - And Nobody Noticed"
description: "Why a static PHP build on Windows dies with hundreds of LNK2001 errors against modern libxml2, and why Linux never sees it."
date: 2026-08-11T20:36:00-07:00
tags: [ php, windows, libxml2, msvc, static-php-cli, linker ]
categories: [ php, windows ]
images:
 - src: img/2026/08/libxml2-214-php-windows-def-missing-symbols.svg
   alt: "Bar chart: legacy php_libxml2.def entries missing from libxml2 headers - 0 for 2.11.9 and 2.13.0, 294 for 2.14.0 and 2.15.0"
   stretch: "none"
draft: false
---

Back in April I was getting the embed SAPI building on Windows for [ePHPm](https://github.com/ephpm/ephpm). I was
using my fork of [static-php-cli](https://github.com/crazywhalecc/static-php-cli) (SPC). The link step for
`php8ts.dll` fell over with a wall of errors that all looked like this:

```
php8ts.dll.def : error LNK2001: unresolved external symbol UTF8ToHtml
php8ts.dll.def : error LNK2001: unresolved external symbol UTF8Toisolat1
php8ts.dll.def : error LNK2001: unresolved external symbol __htmlDefaultSAXHandler
...
php8ts.dll.def : error LNK2001: unresolved external symbol attributeDecl
...
```

(That excerpt is from [php/php-src#22752](https://github.com/php/php-src/issues/22752). It's the same failure someone
else hit three months later. My log is long gone, and theirs is public.)

Nothing about it says "libxml2". The error is blamed on a `.def` file I never wrote. The same build on Linux and macOS
is perfectly happy. This one cost me an evening, and the upstream fix landed last week. So here is the whole story.

### What a .def file actually does

PHP's Windows build is driven by `configure.js` and a pile of `config.w32` files. The libxml extension can be built
statically into `php8ts.dll`. When it is, `ext/libxml/config.w32` adds a module-definition file to the link:

```javascript
if (!PHP_LIBXML_SHARED) {
    ADD_DEF_FILE("ext\\libxml\\php_libxml2.def");
}
```

That file is a list of libxml2 function names under an `EXPORTS` header. The point is to re-export libxml2's API out of
`php8ts.dll`. Then shared extensions like `dom`, `xsl` or `simplexml` can call into the copy of libxml2 that lives
inside the PHP DLL.

The catch is how MSVC treats it. Every name listed under `EXPORTS` is a **hard requirement**. If `link.exe` can't find a
definition for it in some object or library on the command line, you get `LNK2001` and no DLL. It doesn't matter that
nothing in PHP calls the function. The `.def` file asked for it, so it must exist.

Linux and macOS don't use `.def` files at all. Symbols are exported by visibility, and a symbol nobody references simply
isn't there. That is the entire reason this is a Windows-only failure.

### What libxml2 took away

The list in `php_libxml2.def` was written a long time ago. It had been edited a handful of times since
([2022](https://github.com/php/php-src/commit/ca011bbf) for 2.10,
[2024](https://github.com/php/php-src/commit/b08def51) for 2.13). At that point it was 1,514 lines long. Meanwhile
libxml2 had been deprecating chunks of its API for years. In **2.14.0 (March 27, 2025)** it started deleting them.
From the [2.14.0 NEWS](https://gitlab.gnome.org/GNOME/libxml2/-/blob/v2.14.0/NEWS):

> The FTP module and related functions were removed.
>
> Several legacy symbols and the functions in xmlunicode.h were removed.
>
> The shell was moved from libxml2 to xmllint. Several related functions are no longer available.

`xmlunicode.h` is the big one for PHP. In 2.13.0 it was a 366-line header declaring 166 `xmlUCSIs*` functions. There
was one per Unicode block and category, like `xmlUCSIsArabic` and `xmlUCSIsAegeanNumbers`. In 2.14.0 the whole header,
minus a comment block, is this:

```c
#ifndef __XML_UNICODE_H__
#define __XML_UNICODE_H__

#ifdef __GNUC__
  #warning "libxml/xmlunicode.h is deprecated"
#endif

#endif /* __XML_UNICODE_H__ */
```

The implementations are still in `xmlunicode.c` because the regex engine uses them. But 128 of them are now `static
int`, and the rest are `XML_HIDDEN` in a private header. They're internal now. PHP's `.def` file still listed all 166
of them. It also listed:

- the `xmlNanoFTP*` client
- the `xmlShell*` functions
- a pile of unprefixed SAX1 names like `characters`, `startElement` and `attributeDecl` that libxml2 used to export
  straight into the global namespace

2.15.0 (September 2025) kept going. It removed the HTTP client and the old `win32` build system. The `xmlNanoHTTP*`
declarations are now behind a stubs flag. But the break for PHP starts at 2.14.

### Measuring the gap

My fix (below) works by scanning libxml2's installed headers for what they declare. I reused the same idea to answer
"which libxml2 releases can link the old `.def` at all". Take every symbol in the pre-fix `php_libxml2.def`. Then
check whether any public header in a given libxml2 tag declares it with `XMLPUBFUN`/`XMLPUBVAR`, or defines it through
the `XML_OP` globals macro:

![Legacy php_libxml2.def entries missing from libxml2 public headers per release](/img/2026/08/libxml2-214-php-windows-def-missing-symbols.svg)

Zero are missing through 2.13.0. Then 294 are missing at 2.14.0, and the same 294 at 2.15.0. Of those, 166 are
`xmlUCSIs*` and 26 are the FTP client (`xmlNanoFTP*` plus `xmlIOFTP*`). It's a cliff, and it's at 2.14.

### Why nobody noticed

Because almost nobody builds PHP on Windows against a current libxml2.

The official Windows builds use the prebuilt dependency series from windows.php.net. Here is the libxml2 line out of
[packages-8.4-vs17-x64-stable.txt](https://downloads.php.net/~windows/php-sdk/deps/series/packages-8.4-vs17-x64-stable.txt):

```
libxml2-2.11.9-8-vs17-x64.zip
```

2.11.9. Every vs17 libxml2 package in that directory is a 2.11.9 build, an unsuffixed one plus `-1` through `-8`. So
the official pipeline never sees a libxml2 that's missing anything. Linux distros moved to 2.14 and 2.15 and didn't
care, because no `.def`. That leaves the people who build PHP on Windows *and* bring their own libxml2. That's a
pretty short list. SPC is on it, because SPC builds every library from source. Its libxml2 entry tracks the newest
`v2.x.y` tag on [GNOME/libxml2](https://github.com/GNOME/libxml2), which by this spring was 2.15.x.

So the `.def` file sat there broken against current libxml2 for over a year. The only people who'd hit it were static
builders on Windows. Which is to say me, in April.

### The fix I carried in SPC

My first instinct was a blocklist: strip lines starting with `xmlUCSIs`, `xmlNanoFTP`, `xmlShell` and so on. That
works for exactly one libxml2 version. It breaks the next time they delete something. The better source of truth is
sitting right there in the build root: libxml2's own installed headers. libxml2 marks its entire public API with
`XMLPUBFUN` (functions) and `XMLPUBVAR` (variables). So a `.def` entry without a matching declaration can be dropped.

This went into `SourcePatcher::patchBeforeMake()` for Windows builds as part of
[crazywhalecc/static-php-cli#1079](https://github.com/crazywhalecc/static-php-cli/pull/1079). Trimmed:

```php
public static function patchLibxml2DefForWindows(): void
{
    $def_file = SOURCE_PATH . '/php-src/ext/libxml/php_libxml2.def';
    $include_dir = BUILD_ROOT_PATH . '/include/libxml2/libxml';

    // features compiled out of this libxml2 build: #undef LIBXML_<FEATURE>_ENABLED
    $disabled_features = [];
    $version_h = file_get_contents("{$include_dir}/xmlversion.h");
    if (preg_match_all('/#undef\s+(LIBXML_\w+_ENABLED)/', $version_h, $m)) {
        foreach ($m[1] as $feat) {
            $disabled_features[$feat] = true;
        }
    }

    $header_symbols = [];
    foreach (glob("{$include_dir}/*.h") as $header) {
        $ifdef_depth = 0;
        $disabled_depth = 0;
        foreach (explode("\n", file_get_contents($header)) as $hline) {
            $trimmed = ltrim($hline);
            if (preg_match('/^#\s*if(?:def|ndef)?\s+.*?(LIBXML_\w+_ENABLED)/', $trimmed, $im)) {
                ++$ifdef_depth;
                if (isset($disabled_features[$im[1]])) {
                    $disabled_depth = $ifdef_depth;
                }
            } elseif (preg_match('/^#\s*if\b/', $trimmed)) {
                ++$ifdef_depth;
            } elseif (preg_match('/^#\s*endif/', $trimmed)) {
                if ($ifdef_depth === $disabled_depth) {
                    $disabled_depth = 0;
                }
                $ifdef_depth = max(0, $ifdef_depth - 1);
            }
            if ($disabled_depth > 0) {
                continue; // declared, but not compiled into this build
            }
            if (preg_match('/XMLPUBFUN\s+\S+\s+(?:XMLCALL\s+|XMLCDECL\s+)?(\w+)\s*\(/', $trimmed, $fm)) {
                $header_symbols[$fm[1]] = true;
            }
            if (preg_match('/XMLPUBVAR\s+\S+\s+(\w+)/', $trimmed, $vm)) {
                $header_symbols[$vm[1]] = true;
            }
        }
    }

    $filtered = [];
    $removed = 0;
    foreach (file($def_file, FILE_IGNORE_NEW_LINES) as $line) {
        $sym = trim($line);
        if ($sym === '' || $sym === 'EXPORTS' || str_starts_with($sym, ';') || isset($header_symbols[$sym])) {
            $filtered[] = $line;
        } else {
            ++$removed;
        }
    }
    file_put_contents($def_file, implode("\n", $filtered) . "\n");
    logger()->info("Stripped {$removed} missing symbols from php_libxml2.def (libxml2 compat)");
}
```

The `#ifdef` tracking was the part I didn't have on the first pass. A header can declare a function inside `#ifdef
LIBXML_HTTP_ENABLED` while the library you built has HTTP turned off. So "it's in the header" isn't the same as "it's
in `libxml2_a.lib`". `xmlversion.h` tells you which features were compiled out. Declarations inside one of those
blocks get skipped. It also doesn't need `dumpbin` on the `PATH`. That matters because this runs in plain PHP before
`nmake` is ever invoked.

That PR got closed for a different reason. SPC was in the middle of a v3 rewrite, and the maintainer had already
reworked the Windows embed build there. So the patch lived in my fork.

### The upstream fix, and the 2.14 hole in it

In July someone else hit it building plain `php.exe` against libxml2 2.15.3 and filed
[GH-22752](https://github.com/php/php-src/issues/22752). On August 8th [the
fix](https://github.com/php/php-src/commit/bc7537db) landed:

- The old list was preserved as `php_libxml2_legacy.def`.
- A new `php_libxml2.def` dropped 314 entries and added a few newer APIs.
- `config.w32` now picks one based on the libxml2 version:

```javascript
if (libxml_ver >= 21500) {
    /* The HTTP and FTP modules, SAX1, the shell, xmlunicode.h
       and various other legacy symbols were removed in 2.15. */
    ADD_DEF_FILE("ext\\libxml\\php_libxml2.def");
} else {
    ADD_DEF_FILE("ext\\libxml\\php_libxml2_legacy.def");
}
```

That's a reasonable fix, and I'm glad it's in. But read the comment against the libxml2 NEWS above. The FTP module,
`xmlunicode.h` and the shell went away in **2.14**, not 2.15. The reporter tested 2.13.9 (works) and 2.15.3 (broken)
and never tried 2.14. The libxml2 maintainer even pointed at 2.14 in the thread. The fix that landed still gates
at 21500. As far as I can tell from reading it, a build against any libxml2 2.14.x still gets the legacy list. It still
asks for 166 `xmlUCSIs*` exports and still dies the same way. The chart above says the gate should be at 21400.

2.14 is an odd version to be stuck on, since 2.15 has been out for nearly a year. In practice this probably bites very
few people. It's the same shape as the original bug though: a version boundary nobody exercised on Windows.

### Takeaways

- **A `.def` file is a contract with every library you link.** Each line is a promise that some object file provides
  that name. When a dependency deletes API, Linux shrugs and Windows refuses to link. The error points at the `.def`
  instead of the library that changed.
- **Pinned dependencies hide breakage.** The official PHP Windows builds pinning libxml2 2.11.9 is a perfectly sane
  choice. It just means the people who'd notice first are the ones building from source.
- **Derive, don't enumerate.** A blocklist of removed prefixes would have needed updating for 2.14, again for 2.15 and
  again for whatever comes next. Asking the headers what exists keeps working.

### How I measured

The chart is a static scan, no compiler involved. There were two inputs:

- `ext/libxml/php_libxml2.def` from php-src at
  [b08def51](https://github.com/php/php-src/blob/b08def51/ext/libxml/php_libxml2.def), the last version before the
  August fix (1,514 lines)
- `include/libxml/*.h` from the GNOME/libxml2 tags `v2.11.9`, `v2.13.0`, `v2.14.0` and `v2.15.0`

For each tag I collected every name declared with `XMLPUBFUN`/`XMLPUBVAR`. Comments were stripped, and declarations
were allowed to span lines. I added the globals generated by `XML_OP(name, ...)`, which also produce a `__name`
accessor. Then I counted `.def` symbols with no match. It ignores `#ifdef` feature guards. So it's a floor rather than
an exact list of linker errors for any one build configuration. The 166 `xmlUCSIs*` count is a straight count of
`.def` lines and of `XMLPUBFUN` declarations in the 2.13.0 `xmlunicode.h`.

### Links

- [php/php-src#22752](https://github.com/php/php-src/issues/22752) - "Build failure with libxml 2.15"
- [php/php-src@bc7537db](https://github.com/php/php-src/commit/bc7537db) - the upstream fix
- [crazywhalecc/static-php-cli#1079](https://github.com/crazywhalecc/static-php-cli/pull/1079) - my SPC PR with the
  header-scan patch
- [libxml2 NEWS](https://gitlab.gnome.org/GNOME/libxml2/-/blob/master/NEWS)
- [Module-definition (.def)
  files](https://learn.microsoft.com/en-us/cpp/build/reference/module-definition-dot-def-files) on Microsoft Learn
- [PHP Windows dependency series](https://downloads.php.net/~windows/php-sdk/deps/series/)
