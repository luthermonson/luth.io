---
title: "Self-Hosted Runner Part 5: Windows x86_64 on Proxmox"
description: "A Windows VM on Proxmox as a GitHub Actions runner: VirtIO drivers, sizing, and the PATH and pipe bugs that showed up on the first night."
date: 2026-04-28T21:15:00-07:00
tags: [ github-actions, self-hosted-runners, windows, proxmox, virtio, php, homelab ]
categories: [ ci, homelab, windows ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/04/self-hosted-runner-proxmox-windows-durations.svg
   alt: "Bar chart of php-sdk build job durations on the self-hosted Linux and Windows runners"
   stretch: "none"
draft: true
---

The second VM on the Proxmox box is Windows x86_64. It's the runner that went live the same night as the Linux one
([`662d8d0`](https://github.com/ephpm/php-sdk/commit/662d8d0), April 1st), registered as `windows-github-runner`, and
it's the one that generated the most commits in the shortest time. This post is the VM setup, the runner setup (short,
since [Part 2](/blog/2026/04/self-hosted-runner-windows-dev-kit-arm64/) already covered it), and the Windows problems
that a fresh hosted image hides from you.

### The VM

Windows on Proxmox is a solved problem, but it has more knobs than Linux. What I set when creating it:

| Setting | Value | Why |
|---|---|---|
| OS type | Microsoft Windows, 11/2022/2025 | Proxmox tunes timers and Hyper-V enlightenments for the guest |
| Machine | q35 | modern chipset, PCIe |
| BIOS | OVMF (UEFI) + EFI disk | Windows 11 needs UEFI, Server is happier with it |
| TPM | v2.0 state disk | only if it's Windows 11 |
| SCSI controller | VirtIO SCSI single | fast disk, needs a driver at install time |
| Disk | 128GB+, discard on, IO thread on | Visual Studio Build Tools and a PHP build tree add up quickly |
| CPU | type `host` | same reason as the Linux VM: let the compiler see the real CPU |
| Network | VirtIO (paravirtualized) | needs the NetKVM driver |
| Agent | QEMU guest agent enabled | clean shutdowns and IP reporting from the Proxmox side |

The VirtIO bits are what make Windows on QEMU fast, and Windows doesn't ship the drivers. Attach the
[virtio-win ISO](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md) as a second CD-ROM. When
the installer says it can't find a disk, click **Load driver** and point it at `vioscsi\<version>\amd64` on that ISO.
After the install finishes, run `virtio-win-guest-tools.exe` from the same ISO. It installs the network driver, the
balloon driver and service, and the QEMU guest agent in one go. Without the balloon service, Proxmox can't see how much
memory the guest is really using and its graphs show the VM at close to its full allocation all the time.

<!-- TODO(luther): which Windows you run on windows-github-runner (Server 2022/2025? Windows 11 Pro?) and how it's licensed (eval, retail key, MSDN/VL). Licensing is the one part of this I can't write for you. -->

<!-- TODO(luther): actual vCPU / RAM / disk for the VM. -->

### Runner setup

Same as on the Dev Kit, with the x64 package. From an elevated PowerShell in the VM:

```powershell
mkdir \actions-runner ; cd \actions-runner
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-win-x64-2.333.1.zip -OutFile actions-runner-win-x64-2.333.1.zip
(Get-FileHash .\actions-runner-win-x64-2.333.1.zip -Algorithm SHA256).Hash -eq 'D0C4FCB91F8F0754D478DB5D61DB533BBA14CAD6C4676A9B93C0B7C2A3969AA0'
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD\actions-runner-win-x64-2.333.1.zip", "$PWD")

.\config.cmd --unattended `
  --url https://github.com/ephpm `
  --token <token from gh api> `
  --name windows-github-runner `
  --runasservice
```

Labels: `self-hosted`, `Windows`, `X64`. Install Git for Windows machine-wide before the first job, for the
`actions/checkout` reason from Part 2.

### First night: the PHP build

php-sdk's Windows leg builds PHP's embed SAPI with [static-php-cli](https://github.com/crazywhalecc/static-php-cli) (spc).
On a hosted runner the job would start from a clean image. On mine, the first version of the job had to clean up after
itself before it did any real work ([`662d8d0`](https://github.com/ephpm/php-sdk/commit/662d8d0)):

```yaml
- name: Clean up previous builds
  shell: pwsh
  run: |
    foreach ($dir in @('buildroot', 'source', 'downloads', 'sdk')) {
      if (Test-Path $dir) { Remove-Item -Recurse -Force $dir }
    }
    Remove-Item -Force *.tar.gz -ErrorAction SilentlyContinue

- name: Install PHP (host toolchain for spc)
  shell: pwsh
  run: |
    if (Test-Path C:\php) { Remove-Item -Recurse -Force C:\php }
    Invoke-WebRequest -Uri "https://windows.php.net/downloads/releases/php-8.4.19-nts-Win32-vs17-x64.zip" -OutFile php.zip
    Expand-Archive -Path php.zip -DestinationPath C:\php
    "C:\php" | Out-File -Append -FilePath $env:GITHUB_PATH
```

Note `C:\php`. That's outside the workspace, on the VM's system drive, and it persists between jobs, so the job deletes
it first. (spc is a PHP app, so it needs a host PHP to run. That PHP has nothing to do with the version being built.)

Then the actual problems started.

### Problem 1: the pipe has been ended

spc extracts source tarballs by piping `7za` into `tar`. On Linux that's fine. On Windows, for large archives, it died
with "The pipe has been ended". My first fix was to pre-extract PHP's source in the workflow in two steps, decompress the
`.tar.xz` to a `.tar` on disk with `7za`, then untar it, and write spc's `.spc-hash` file so it wouldn't extract again.

### Problem 2: which tar?

That fix promptly hit the next problem. To get `patch` and friends, the job had added Git's MinGW tools to the path
([`f2f62f1`](https://github.com/ephpm/php-sdk/commit/f2f62f1)):

```powershell
# spc needs Git MinGW tools (patch, etc.)
"C:\Program Files\Git\usr\bin" | Out-File -Append -FilePath $env:GITHUB_PATH
```

`GITHUB_PATH` *prepends* to `PATH` for later steps, so after that line, `tar` meant Git's MinGW `tar.exe`, not Windows'
own `C:\Windows\System32\tar.exe` (bsdtar). MinGW tar doesn't understand `C:\...` paths. The fix was to call the system
one by full path ([`e731253`](https://github.com/ephpm/php-sdk/commit/e731253)):

```powershell
# Step 2: extract tar → source/php-src (use Windows tar, not MinGW tar)
& "$env:SystemRoot\System32\tar.exe" -xf $tarFile -C $dest --strip-components 1
```

You can see the shadowing for yourself with the `peaudit` tool from Part 2, which resolves names through `PATH` and
prints what it found. Same machine, two shells:

```
# Git Bash
go           x64    C:\Program Files\Go\bin\go.exe
git          x64    C:\Program Files\Git\mingw64\bin\git.exe
tar          x64    C:\Program Files\Git\usr\bin\tar.exe
powershell   x64    C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe

# Windows PowerShell
go           x64    C:\Program Files\Go\bin\go.exe
git          x64    C:\Program Files\Git\cmd\git.exe
tar          x64    C:\WINDOWS\system32\tar.exe
powershell   x64    C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe
```

Two different `tar.exe` binaries and two different `git.exe` binaries depending on which shell's `PATH` you inherited.
A runner step that adds a directory to `GITHUB_PATH` changes that for every later step in the job, in every shell.

### Problem 3: where the fix belongs

By the end of the evening I backed the workarounds out
([`c2c4995`](https://github.com/ephpm/php-sdk/commit/c2c4995)). The commit message is the whole lesson:

```
ci: revert extraction workarounds, fix applied to spc on runner

The pipe bug fix belongs in FileSystem.php on the self-hosted runner,
not as CI step workarounds. Reverts the pre-extraction steps.
```

The extraction is spc's bug, so it got fixed in spc. And here's the part I'm less proud of: for the next three weeks that
patched spc lived only on the runner, in `C:\spc-src`, along with the Windows embed SAPI support spc didn't have yet.
The workflow called it by path:

```powershell
C:\php\php.exe C:\spc-src\bin\spc build ${{ env.WIN_PHP_EXTENSIONS }} `
  --build-embed `
  --enable-zts `
  --no-strip `
  --debug
```

So the Windows build worked on exactly one machine in the world, until I moved the patch into a public fork of
static-php-cli on April 25th and the workflow started downloading its `spc.exe` instead. That's the trade a plain
self-hosted runner lets you make: you can patch the toolchain in place and keep moving, and the cost is a runner nobody
else can reproduce. In [run 23934915779](https://github.com/ephpm/php-sdk/actions/runs/23934915779) the Windows job
finally went green on attempt 6, after five failed tries; the run itself stayed red because the hosted macOS and arm64
legs were blocked. The 27m52s below is the attempt-9 re-run.

### Windows is slower, budget for it

![php-sdk build durations on the self-hosted runners](/img/2026/04/self-hosted-runner-proxmox-windows-durations.svg)

These are the php-sdk SDK builds from that run (attempt 9) and
[run 23994291885](https://github.com/ephpm/php-sdk/actions/runs/23994291885) a couple of hours after it, read from the job start and finish times in the Actions API.
Linux x86_64 took 9m57s and 9m59s on `linux-github-runner`. Windows x86_64 took 27m52s and 27m05s on
`windows-github-runner`. It isn't an apples-to-apples comparison (musl and gcc on one side, MSVC and nmake on the other,
with a different extension list), but it's the number that matters for planning: the Windows leg is the long pole, close
to three times the Linux one, and a matrix run isn't done until it is. If you only have room to give one VM more cores,
give them to Windows.

### Compared to Windows on arm64

Setting up the runner is identical on both. The `config.cmd` flags, the service, NETWORK SERVICE, Git and `pwsh` all
work the same way. The differences are around it:

- **Toolchain.** On x64 the toolchains I need all exist. PHP for Windows ships x64 builds, and nothing runs under emulation. On the Dev
  Kit you have to check each tool.
- **Power.** The VM never sleeps. The Dev Kit needed `powercfg` to stop Modern Standby from taking it offline.
- **Recovery.** This one has Proxmox snapshots. When a job leaves the VM in a strange state, I roll it back. The Dev Kit
  is bare metal, so recovering it means fixing it by hand.
- **Speed.** The VM shares a host with the rest of my lab. The Dev Kit's runner has the whole machine to itself, apart
  from the Linux VM from [Part 3](/blog/2026/04/self-hosted-runner-linux-arm64-hyper-v/).

### How I checked

The `peaudit` output above is from my Windows workstation (AMD Ryzen 9 5950X, Windows 11 Pro build 26200), with Git for
Windows and Go installed in their default locations: `go run . go git tar powershell` once from Git Bash and once from
Windows PowerShell. The durations are `completed_at - started_at` for the `Linux (x86_64)` and `Windows (x86_64)` jobs
from `gh api repos/ephpm/php-sdk/actions/runs/<id>/jobs`.

That's all five targets. The last post puts them in [one workflow matrix](/blog/2026/05/self-hosted-runner-matrix-workflow/)
and asks whether the whole thing was worth it.

### Links

- [virtio-win drivers](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)
- [Proxmox wiki: Windows 2022 guest best practices](https://pve.proxmox.com/wiki/Windows_2022_guest_best_practices)
- [actions/runner v2.333.1](https://github.com/actions/runner/releases/tag/v2.333.1)
- [static-php-cli](https://github.com/crazywhalecc/static-php-cli)
- [Workflow commands: adding a system path](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands#adding-a-system-path)
