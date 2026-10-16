---
title: "Fixing Windows Archive Extraction in SPC"
description: "\"The pipe has been ended\" wasn't a pipe problem. It was the wrong tar program and Windows symlinks, and fixing one line of static-php-cli took three tries."
date: 2026-07-18T16:10:00-07:00
tags: [ php, windows, static-php-cli, tar, 7zip, open-source ]
categories: [ php, windows ]
images:
 - src: img/2026/07/spc-windows-archive-extraction-pipe.svg
   alt: "Diagram: 7za decompressing into a pipe read by MinGW tar, which exits on a C:\\ path and leaves 7za reporting The pipe has been ended"
   stretch: "none"
draft: false
---

One of the first things that broke when I started building PHP on Windows with
[static-php-cli](https://github.com/crazywhalecc/static-php-cli) (SPC) wasn't PHP at all. It was unpacking the source
tarballs. libxml2, xz and a couple of others would die partway through extraction with:

```
ERROR: The pipe has been ended.
```

Over the last few months I've "fixed" this two different ways, and the maintainers fixed it a third. It's a small piece
of code. But it's a good example of how much Linux assumption hides in a single shell pipeline, so here's the whole
thing.

### The pipeline

![7za piping into MinGW tar, which exits on a Windows path](/img/2026/07/spc-windows-archive-extraction-pipe.svg)

SPC v2 extracted archives on Windows with one line. Here's the block from `FileSystem::extractArchive()`. I kept the
maintainer's comments, which tell you how much fun this code has been for everyone:

```php
} elseif (PHP_OS_FAMILY === 'Windows') {
    // use php-sdk-binary-tools/bin/7za.exe
    $_7z = self::convertPath(getenv('PHP_SDK_PATH') . '/bin/7za.exe');

    // Windows notes: I hate windows tar.......
    // When extracting .tar.gz like libxml2, it shows a symlink error and returns code[1].
    // ...
    // And MinGW tar.exe cannot work on temporarily storage ??? (GitHub Actions hosted runner)
    // Yeah, I will be an MS HATER !
    match (self::extname($filename)) {
        'tar' => f_passthru("tar -xf {$filename} -C {$target} --strip-components 1"),
        'xz', 'txz', 'gz', 'tgz', 'bz2' => cmd()->execWithResult(
            "\"{$_7z}\" x -so {$filename} | tar -f - -x -C \"{$target}\" --strip-components 1"),
        'zip' => self::unzipWithStrip($filename, $target),
        default => throw new FileSystemException("unknown archive format: {$filename}"),
    };
}
```

7za decompresses the `.xz`/`.gz`/`.bz2` layer to stdout (`-so`). `tar` reads the inner tarball from stdin and handles
`--strip-components 1`. That flattens the `libxml2-2.x.y/` top-level directory into the target directory. On Linux this
is the most boring command in the world.

### First fix: stop piping

I was building on a Windows runner that also has Git for Windows installed. My first read of the error was the obvious
one. Windows pipes between processes are flaky with big streams, so don't stream. Decompress to disk, then extract from
disk. That went into my SPC PR [#1079](https://github.com/crazywhalecc/static-php-cli/pull/1079) in April:

```php
'xz', 'txz', 'gz', 'tgz', 'bz2' => (function () use ($_7z, $filename, $target) {
    $dir = dirname($filename);
    cmd()->execWithResult("\"{$_7z}\" x \"{$filename}\" -o\"{$dir}\" -y");
    $tarFile = preg_replace('/\.(xz|txz|gz|tgz|bz2)$/i', '', $filename);
    if (!file_exists($tarFile)) {
        $tarFile .= '.tar';
    }
    $winTar = getenv('SystemRoot') . '\System32\tar.exe';
    f_passthru("\"{$winTar}\" -xf \"{$tarFile}\" -C \"{$target}\" --strip-components 1");
    @unlink($tarFile);
})(),
```

It's two steps. 7za writes `foo.tar` next to the archive. Then `C:\Windows\System32\tar.exe` extracts it and the
intermediate file gets deleted. It worked. But look at the second-to-last line. That's the real fix, and the pipe story
I told myself was wrong.

### What was really happening

Notice that the original command says `tar`, not a path. On a stock Windows box that resolves to
`C:\Windows\System32\tar.exe`. That's bsdtar from libarchive and it handles Windows paths fine. Some boxes have Git for
Windows (or MSYS2) on the `PATH` ahead of System32. There `tar` resolves to `C:\Program Files\Git\usr\bin\tar.exe`
instead. That's GNU tar built for MSYS, and GNU tar has opinions about colons.

I reproduced it again for this post with a `.tar.xz` that unpacks to about 67 MB, piping 7za into each tar:

```
> 7za x -so big.tar.xz | "C:\Program Files\Git\usr\bin\tar.exe" -f - -x -C "C:\...\out2" --strip-components 1
/usr/bin/tar: C\:\\Users\\...\\out2: Cannot open: No such file or directory
/usr/bin/tar: Error is not recoverable: exiting now
ERROR: The pipe has been ended.

> 7za x -so big.tar.xz | C:\Windows\System32\tar.exe -f - -x -C "C:\...\out2" --strip-components 1
(exit 0, files extracted)
```

GNU tar can't use the `C:\...` target directory and exits immediately. 7za is still writing decompressed bytes into the
pipe, but the reader is gone. So 7za reports the write failure as `The pipe has been ended`. The error everyone sees is
printed by the *writer*, about the *reader* dying. Nothing is wrong with the pipe.

The same tar without the pipe gets a different and even stranger error. To GNU tar, `-f C:\foo.tar` means "a tape on
the remote host named `C`":

```
> "C:\Program Files\Git\usr\bin\tar.exe" -xf C:\...\pkg.tar -C C:\...\out --strip-components 1
/usr/bin/tar: Cannot connect to C: resolve failed
```

So my two-step fix worked for the boring reason that it hardcoded `System32\tar.exe`. The decompress-to-disk part only
added an extra copy of every tarball on disk. Lesson learned. When a pipeline fails, check that each side of it is the
binary you think it is before blaming the pipe.

### Second fix: v3 and the symlink problem

That PR didn't merge. SPC was in the middle of a ground-up v3 rewrite, and v3's shell layer already pinned the binary:

```php
$tar = SystemTarget::isUnix() ? 'tar' : '"C:\Windows\system32\tar.exe"';
```

So in v3 the PATH problem was gone, and the pipe was fine. What was left was the other thing in the maintainer's
comment: **symlinks**. Some upstream tarballs ship symbolic links. zstd ships `tests/cli-tests/bin/unzstd -> zstd`, for
example. Windows only lets a process create symlinks with admin rights or Developer Mode turned on. bsdtar fails each
one with `Can't create '...': Invalid argument` and exits non-zero. On a CI runner without those rights, the extraction
fails and leaves a half-written source tree behind.

My answer in May was [#1175](https://github.com/crazywhalecc/static-php-cli/pull/1175). Drop `tar.exe` entirely and let
7za do both layers. 7za doesn't fail the extraction on symlinks the way bsdtar does. 7za has no `--strip-components`, so
that moved into PHP. The version below is lightly trimmed, with the logging taken out and comments added:

```php
private function extract7zWithStripComponents(string $archive_path, string $target_path, string $_7z, int $strip = 1): void
{
    $temp_dir = sys_get_temp_dir() . DIRECTORY_SEPARATOR . 'spc_' . bin2hex(random_bytes(8));
    FileSystem::createDir($temp_dir);
    $archive_arg = escapeshellarg(FileSystem::convertPath($archive_path));
    $temp_arg = escapeshellarg($temp_dir);

    // pass 1: .tar.xz -> .tar
    $this->passthru("{$_7z} x {$archive_arg} -o{$temp_arg} -y", $this->console_putput);

    // pass 2: .tar -> files
    $inner_tar = glob($temp_dir . DIRECTORY_SEPARATOR . '*.tar');
    if ($inner_tar !== []) {
        $tar_arg = escapeshellarg($inner_tar[0]);
        $this->passthru("{$_7z} x {$tar_arg} -o{$temp_arg} -y", $this->console_putput);
        @unlink($inner_tar[0]);
    }

    // emulate --strip-components 1: a single top-level dir gets its contents copied up
    if ($strip > 0) {
        $entries = array_diff(scandir($temp_dir), ['.', '..']);
        if (count($entries) === 1) {
            $inner = $temp_dir . DIRECTORY_SEPARATOR . reset($entries);
            if (is_dir($inner)) {
                FileSystem::copyDir($inner, $target_path);
                FileSystem::removeDir($temp_dir);
                return;
            }
        }
    }

    FileSystem::copyDir($temp_dir, $target_path);
    FileSystem::removeDir($temp_dir);
}
```

It worked in my fork and that's what my Windows builds ran on for the next couple of months.

### What upstream did instead

The review feedback was fair. crazywhalecc said a full 7za path was a reasonable direction. But `FileSystem::copyDir` is
slow, and my version copies every extracted source tree a second time to fake `--strip-components`. It also only
approximates it. If an archive has more than one top-level entry, nothing gets stripped.

Then two things landed in v3. In June, PHP's old `php-sdk-binary-tools` were replaced with MSYS2 plus a standalone
`7za-win` tool package ([#1193](https://github.com/crazywhalecc/static-php-cli/pull/1193)). And the shell layer grew
`passthruTolerateSymlinks()`. It does three things:

- keeps the 7za-into-`System32\tar.exe` pipe
- captures tar's output
- if the *only* errors are symlinks Windows refused to create, logs a warning and keeps going

Anything else still throws.

```php
private function passthruTolerateSymlinks(string $cmd): void
{
    // Symlink creation only fails on a Windows host; elsewhere extraction handles symlinks fine.
    if (PHP_OS_FAMILY !== 'Windows') {
        $this->passthru($cmd, $this->console_putput);
        return;
    }

    $result = $this->passthru($cmd, $this->console_putput, capture_output: true, throw_on_error: false);
    if ($result['code'] === 0) {
        return;
    }
    if ($this->isSymlinkOnlyExtractFailure($result['output'])) {
        logger()->warning('Some symbolic links could not be created during extraction and were skipped ...');
        return;
    }
    throw new ExecutionException(/* ... */);
}
```

That keeps the single streaming pass and real `--strip-components`. It only forgives the one failure mode we know is
harmless. #1175 was closed as superseded on July 13th, which is the right call. Their version is better than mine.

### Takeaways

- **Pin the binary.** `tar` in a command string means "whatever is first on the `PATH`". On Windows developer machines
  and CI runners that's frequently GNU tar from Git or MSYS2, which reads `C:` as a hostname.
- **"The pipe has been ended" means the other side died.** Go look at what the reader printed before it exited.
- **Tolerate narrowly.** Skipping symlinks wholesale (7za) and failing on them wholesale (tar) are both wrong. Matching the
  specific error and letting everything else fail is the version that survives.

### Links

- [crazywhalecc/static-php-cli#1079](https://github.com/crazywhalecc/static-php-cli/pull/1079) - the v2 PR with the two-step extraction
- [crazywhalecc/static-php-cli#1175](https://github.com/crazywhalecc/static-php-cli/pull/1175) - the v3 7za-only PR
- [crazywhalecc/static-php-cli#1193](https://github.com/crazywhalecc/static-php-cli/pull/1193) - MSYS2 + 7za-win replacing php-sdk-binary-tools
- [libarchive (bsdtar)](https://github.com/libarchive/libarchive), which is what ships as `C:\Windows\System32\tar.exe`
- [Create symbolic links](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/create-symbolic-links) - the Windows privilege behind the symlink failures
