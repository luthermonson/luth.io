---
title: "Self-Hosted Runner Part 2: Windows arm64 on the Windows Dev Kit 2023"
description: "Microsoft's discontinued Snapdragon mini PC as a Windows arm64 runner: service setup, power settings, and catching x64 emulation in your toolchain."
date: 2026-04-21T21:05:00-07:00
tags: [ github-actions, self-hosted-runners, windows, arm64, snapdragon, homelab, go ]
categories: [ ci, homelab, windows ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/04/self-hosted-runner-windows-dev-kit-arm64-cover.svg
   alt: "Diagram of what runs natively on arm64 versus under x64 emulation on the Windows Dev Kit runner"
   stretch: "none"
draft: true
---

Windows on arm64 is the target most projects skip, and it's the one where "it compiled" means the least, because Windows
on Arm will quietly run x64 binaries under emulation. A job can go green without a single arm64 instruction in it. So
the second box in this [series](/blog/2026/04/self-hosted-runners-every-arch/) is a Windows Dev Kit 2023.

### What the Dev Kit is

Microsoft announced it as "Project Volterra" and
[shipped it on October 24th, 2022](https://blogs.windows.com/windowsdeveloper/2022/10/24/available-today-windows-dev-kit-2023-aka-project-volterra/)
as a developer box for building native Arm apps on Arm. Per the
[Learn page](https://learn.microsoft.com/en-us/windows/arm/dev-kit/) it's a Snapdragon 8cx Gen 3 with 32GB of LPDDR4x,
a 512GB NVMe drive, Ethernet, three USB-A and two USB-C ports. It shipped with Windows 11 Pro for about
[$600](https://www.cnx-software.com/2022/10/25/600-windows-dev-kit-2023-is-powered-by-qualcomm-snapdragon-8cx-gen-3-processor/).
Microsoft has since stopped selling it ("no longer available for new purchases") and points people at Copilot+ PCs
instead.

That's what makes it a home lab bargain. 32GB of RAM and a real NVMe in a box the size of a paperback is a lot for a
discontinued dev kit, and it's Windows 11 **Pro**, so Hyper-V is there. That matters for
[Part 3](/blog/2026/04/self-hosted-runner-linux-arm64-hyper-v/), where the same box runs the Linux arm64 runner too.

<!-- TODO(luther): where you got your Dev Kit and what you paid. A photo of the actual box would be good here; I couldn't find one on Wikimedia Commons with a usable license. -->

GitHub does have a hosted Windows arm64 runner now, billed at $0.010 a minute for the 2-core size, the same as Windows x64.
It's not expensive. The reason to own one is the same as for the other boxes: caches that stick around, more than two
cores, and a machine I can RDP into when a test fails in a way I can't reproduce on my x64 machines.

### Prerequisites

- The Dev Kit on Windows Update with a local admin account
- Ethernet plugged in (it has a real RJ45 port, use it)
- [gh](https://cli.github.com/) on your workstation to mint registration tokens

### Keep it awake

The Learn page says it plainly: the device "defaults into Connected Standby Mode when not in use." That's Modern Standby,
which is what a laptop wants and exactly what a CI box doesn't. A runner that's asleep can't long-poll GitHub, so jobs
queue until somebody touches the mouse. From an elevated PowerShell:

```powershell
# what sleep states this hardware supports
powercfg /a

# on AC power (it has no battery, AC is all there is), never sleep or hibernate
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
powercfg /change monitor-timeout-ac 10
```

Turning the monitor off is fine. The display isn't what matters, the system going to standby is.

<!-- TODO(luther): did you also need to change Windows Update active hours / restart behavior? Add it here if so. -->

### Install and register

Same story as the Mac: grab the arm64 package for the release, check it against the hash in the
[v2.333.1 release notes](https://github.com/actions/runner/releases/tag/v2.333.1), and extract it into a folder at the root
of the drive. The release notes recommend `C:\actions-runner` because service permissions and long paths both get
easier there.

```powershell
mkdir \actions-runner ; cd \actions-runner
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-win-arm64-2.333.1.zip -OutFile actions-runner-win-arm64-2.333.1.zip
(Get-FileHash .\actions-runner-win-arm64-2.333.1.zip -Algorithm SHA256).Hash -eq 'C665C4CE2BF1D79E1CE49051D91A9E84822A19891CEB442BDAAE9FB0D82EB642'
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD\actions-runner-win-arm64-2.333.1.zip", "$PWD")
```

The `win-arm64` package is properly native. The runner is .NET, and the JavaScript actions it runs (`actions/checkout`,
`actions/setup-go` and friends) use the `node.exe` it bundles under `externals\`. For `win-arm64` that's an arm64
`node.exe` (it's in the runner's
[`externals.sh`](https://github.com/actions/runner/blob/v2.333.1/src/Misc/externals.sh)), so the action plumbing itself
isn't going through emulation.

Get a token on your workstation and configure with `--runasservice`. That flag only exists on Windows:

```powershell
# on the workstation
gh api -X POST orgs/ephpm/actions/runners/registration-token --jq .token

# on the Dev Kit, elevated
.\config.cmd --unattended `
  --url https://github.com/ephpm `
  --token <token> `
  --name devkit `
  --runasservice
```

Default labels are `self-hosted`, `Windows`, `ARM64`.

<!-- TODO(luther): confirm the runner name and org/repo you registered this one against. -->

### Running as a Windows service

`--runasservice` creates and starts a service named `actions.runner.<org>.<name>`. By default it runs as
`NT AUTHORITY\NETWORK SERVICE` (the runner's own tests assert exactly that). The usual service commands work:

```powershell
Get-Service "actions.runner.*"
Restart-Service "actions.runner.*"
```

NETWORK SERVICE is a reasonable default for security and an annoying one for builds:

- It has no real user profile. Tools you installed per-user as yourself (rustup into `%USERPROFILE%\.cargo`, a
  user-scoped winget install, a Go toolchain under your profile) aren't on its `PATH`.
- It isn't an admin, so a job that tries to install something machine-wide fails.
- Files a job writes to "the user's home" go under the service profile in `C:\Windows\ServiceProfiles`, which is a
  great place to lose a cache.

So install toolchains machine-wide, or give the service a dedicated local account with `--windowslogonaccount` and
`--windowslogonpassword` and install its tools as that user.

Two more things the runner doesn't give you on Windows. First, Git. The runner doesn't ship it, and `actions/checkout`
without Git on the `PATH` falls back to downloading the repo through the REST API. That gets you the files and no `.git`
directory, so the first step that runs `git describe` or `git log` fails. Install Git for Windows machine-wide so the
service account sees it. Second, PowerShell 7. A `shell: pwsh` step needs `pwsh.exe` installed; only Windows PowerShell
5.1 (`shell: powershell`) comes with the OS.

### arm64 native vs x64 emulation

![What a job on the Dev Kit actually executes](/img/2026/04/self-hosted-runner-windows-dev-kit-arm64-cover.svg)

Windows 11 on Arm runs x64 binaries through emulation, and it's good enough that you might not notice. That's the trap.
If the `go.exe` on the runner's `PATH` is the x64 build, `go test ./...` compiles amd64 test binaries and runs them
emulated, and your "Windows arm64" job never tested arm64. The same goes for any tool you pulled from a script that
hardcodes an `amd64` or `x64` download.

The first check is cheap. Put this in the job:

```yaml
- name: Toolchain arch
  shell: powershell
  run: |
    go env GOARCH GOHOSTARCH
    go version
```

`GOHOSTARCH` is the architecture the Go toolchain itself was built for. On the Dev Kit both should say `arm64`.

That only covers Go. For the rest of the toolchain I wrote a tiny Go program that resolves each name through `PATH` the
way the shell would and reads the machine type out of the PE header with `debug/pe`:

```go
// peaudit prints the machine type of each executable named on the
// command line, resolved through PATH like the shell would.
package main

import (
	"debug/pe"
	"fmt"
	"os"
	"os/exec"
)

var machines = map[uint16]string{
	pe.IMAGE_FILE_MACHINE_AMD64: "x64",
	pe.IMAGE_FILE_MACHINE_ARM64: "arm64",
	pe.IMAGE_FILE_MACHINE_I386:  "x86",
}

func main() {
	for _, name := range os.Args[1:] {
		path, err := exec.LookPath(name)
		if err != nil {
			fmt.Printf("%-12s not found\n", name)
			continue
		}
		f, err := pe.Open(path)
		if err != nil {
			fmt.Printf("%-12s %v\n", name, err)
			continue
		}
		arch, ok := machines[f.FileHeader.Machine]
		if !ok {
			arch = fmt.Sprintf("0x%x", f.FileHeader.Machine)
		}
		fmt.Printf("%-12s %-6s %s\n", name, arch, path)
		f.Close()
	}
}
```

Build it for `GOARCH=arm64`, drop the binary on the runner, and run it as a step with the tools your build depends on:

```powershell
.\peaudit.exe go git tar cmake clang node
```

<!-- TODO(luther): run peaudit on the Dev Kit runner and paste the real output here. I only have output from an x64 workstation, which isn't useful for this post. -->

One caveat: Arm64EC binaries (Microsoft's "emulation compatible" ABI for porting x64 apps piece by piece) carry the x64
machine type in the header so they can load into x64 processes. So read `x64` as "x64 or Arm64EC" and look closer at
a binary that surprises you.

### What doesn't exist for Windows arm64

Some of the toolchain isn't there at all, and no runner setup fixes that. The one I care about most is PHP. There are no
official PHP builds for Windows arm64, which means my php-sdk Windows leg is x86_64 only for now and the Dev Kit doesn't
build ePHPm at all. Go is the opposite: the Go download page has had a `windows-arm64` build of the toolchain since Go
1.17, so my Go projects are the main thing this runner tests.

Visual Studio has had a native arm64 build since 2022 17.4, and .NET has supported Arm64 since .NET 6, so MSVC-based C
builds work. Check each tool your build actually calls, though, rather than assuming.

### What's next

The Dev Kit has more than enough RAM left over to do a second job. In
[Part 3](/blog/2026/04/self-hosted-runner-linux-arm64-hyper-v/) it gets a Hyper-V VM running Linux arm64, which gives the
series its Linux arm64 runner without buying another box.

### Links

- [Windows Dev Kit 2023 on Microsoft Learn](https://learn.microsoft.com/en-us/windows/arm/dev-kit/)
- [actions/runner v2.333.1](https://github.com/actions/runner/releases/tag/v2.333.1)
- [Arm64EC overview](https://learn.microsoft.com/en-us/windows/arm/arm64ec)
- [Go's debug/pe package](https://pkg.go.dev/debug/pe)
