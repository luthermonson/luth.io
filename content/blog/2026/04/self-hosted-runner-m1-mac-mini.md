---
title: "Self-Hosted Runner Part 1: macOS arm64 on an M1 Mac mini"
description: "Registering a GitHub Actions runner on a headless M1 Mac mini, and the launchd, sleep and Xcode quirks that come with it."
date: 2026-04-19T19:22:00-07:00
tags: [ github-actions, self-hosted-runners, macos, arm64, apple-silicon, homelab ]
categories: [ ci, homelab ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/04/self-hosted-runner-m1-mac-mini-ports.jpg
   alt: "Rear ports of a 2020 M1 Mac mini"
   stretch: "none"
draft: true
---

macOS is the expensive target in this series. A hosted macOS minute is $0.062, ten times a Linux minute (see the
[overview](/blog/2026/04/self-hosted-runners-every-arch/) for the numbers), and it's the one platform you can't
virtualize on generic x86 hardware. So the first box is an M1 Mac mini, which is about the cheapest way to own an Apple
Silicon machine that can sit in a closet with no screen.

![Mac mini 2020 (M1) rear ports](/img/2026/04/self-hosted-runner-m1-mac-mini-ports.jpg)

*Photo: [Gerd Fahrenhorst](https://commons.wikimedia.org/wiki/User:Gerd_Fahrenhorst),
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0), via
[Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Mac_mini_2020_(M1)_Anschl%C3%BCsse.jpg).*

### Picking one up

The M1 Mac mini (Macmini9,1) came out in November 2020. Every one of them has the same 8-core M1; the choices were 8GB
or 16GB of unified memory and the SSD size. For a build box, memory is the one to care about. You can't upgrade it
later and a parallel C build with `-j8` on 8GB will swap.

<!-- TODO(luther): where you bought yours (refurb/used/eBay), what you paid, and the memory/SSD config. A photo of it in the rack/closet would be better than the stock photo above. -->

### Prerequisites

- macOS set up with an admin account, plugged into Ethernet
- A GitHub org or repo you're an admin of, and the [gh CLI](https://cli.github.com/) logged in on your workstation
- [Homebrew](https://brew.sh/) if your jobs use it (it lives in `/opt/homebrew` on Apple Silicon)

### Making a Mac behave like a server

A Mac assumes someone is sitting in front of it. A runner needs the opposite, so the first chunk of work is turning that
off. Over SSH or at the console once:

```sh
# SSH in from now on
sudo systemsetup -setremotelogin on

# power back on by itself after an outage, and never sleep
sudo pmset autorestart 1
sudo pmset -a sleep 0
sudo pmset -a disksleep 0

pmset -g | grep -E 'autorestart|sleep'
```

`autorestart` is the GUI's "Start up automatically after a power failure" setting. Without it a power blip takes your
macOS runner offline until somebody walks over and presses the button on the back.

I also give the runner its own account instead of running it as me. Jobs write into the runner user's home, Homebrew
caches end up there, and a job that does something dumb with `rm` only gets to do it to that user:

```sh
sudo sysadminctl -addUser runner -fullName "GitHub Actions runner" -password -
sudo dseditgroup -o edit -a runner -t user admin   # only if jobs need sudo
```

Leave out the second line if your jobs don't need `sudo`. Mine do, because Xcode's command line tools get installed by
the job (more on that below).

### Installing the runner

Log in as the runner user and pull the `osx-arm64` package. Check the hash against the one in the
[release notes](https://github.com/actions/runner/releases/tag/v2.333.1):

```sh
mkdir ~/actions-runner && cd ~/actions-runner
curl -O -L https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-osx-arm64-2.333.1.tar.gz
echo "69d8c30aff69592fd0e3da6db1a477506ae5942cd438967f098bbdbc4e954c90  actions-runner-osx-arm64-2.333.1.tar.gz" | shasum -a 256 -c
tar xzf ./actions-runner-osx-arm64-2.333.1.tar.gz
```

Use `curl` here and not a browser. Safari tags downloads with the `com.apple.quarantine` attribute and Gatekeeper will
then stop to ask about every unsigned binary in the tarball. `curl` doesn't set it.

Registration needs a short-lived token. I grab it from my workstation with `gh` instead of clicking through the
settings page. Use the org endpoint for org-wide runners, or the repo endpoint
(`repos/OWNER/REPO/actions/runners/registration-token`) for a single repo:

```sh
gh api -X POST orgs/ephpm/actions/runners/registration-token --jq .token
```

The token is good for an hour. Then on the Mac:

```sh
./config.sh --unattended \
  --url https://github.com/ephpm \
  --token <token> \
  --name mac-mini \
  --work _work
```

With no `--labels` the runner gets the defaults: `self-hosted`, `macOS` and `ARM64`. That's all a workflow needs to
target it. I'll get into custom labels in [Part 6](/blog/2026/05/self-hosted-runner-matrix-workflow/).

<!-- TODO(luther): confirm the runner name and org/repo you actually registered against. -->

### launchd, and why it has to be logged in

On macOS `svc.sh` installs a launchd job. Run it as the runner user, not with `sudo`. The script refuses root outright
(`Must not run with sudo`):

```sh
./svc.sh install
./svc.sh start
./svc.sh status
```

What it creates is a **LaunchAgent** in `~/Library/LaunchAgents`, not a LaunchDaemon. Here's the important part of the
template it renders (`bin/actions.runner.plist.template`):

```xml
<key>ProgramArguments</key>
<array>
  <string>{{RunnerRoot}}/runsvc.sh</string>
</array>
<key>UserName</key>
<string>{{User}}</string>
<key>RunAtLoad</key>
<true/>
<key>ProcessType</key>
<string>Interactive</string>
<key>SessionCreate</key>
<true/>
```

A LaunchAgent loads when its user logs in. After a reboot, with nobody logged in, the runner doesn't start, and GitHub
shows it as offline. That's the thing that bit me after the first macOS update. The fix is
**System Settings → Users & Groups → Automatically log in as** the runner user. That option is greyed out while FileVault
is on, so on this box FileVault is off. That's a tradeoff I'm fine with for a machine that holds nothing but build
caches. Don't make it on a Mac that holds your own files.

<!-- TODO(luther): confirm the macOS update/reboot story above matches what happened to you, or cut it. -->

Logs end up in `~/Library/Logs/actions.runner.<org>.<name>/`, and `./svc.sh status` tells you whether launchd thinks it's
running.

### Xcode command line tools, headless

Half of what a macOS job does needs `clang`, `git` and `make`, and on a fresh Mac those come from the Xcode command line
tools. The normal way to install them is `xcode-select --install`, which pops a GUI dialog. On a headless box nobody is
there to click it.

`softwareupdate` can install them if you create the trigger file it looks for. This is the step I added to the macOS build
job in [`b21af9c`](https://github.com/ephpm/ephemerd/commit/b21af9c) so it works on a machine that has never seen Xcode:

```yaml
- name: Install Xcode CLI tools
  run: |
    if ! xcode-select -p &>/dev/null; then
      # Headless install — create the trigger file and use softwareupdate
      sudo touch /tmp/.com.apple.dt.CommandLineTools.installondemand.in-progress
      PROD=$(softwareupdate -l 2>/dev/null | grep -o '.*Command Line Tools.*' | head -1 | sed 's/^[* ]*//')
      sudo softwareupdate -i "$PROD" --verbose 2>&1 | tail -5
      sudo rm -f /tmp/.com.apple.dt.CommandLineTools.installondemand.in-progress
    fi
    xcode-select -p
```

On a persistent runner it only does real work once. After that `xcode-select -p` succeeds and the step takes a second.

### Apple clang and Homebrew

The macOS leg of php-sdk is also where I hit the first "works on hosted, breaks on mine" difference that wasn't really
about the runner. Apple's clang crashes compiling PHP's JIT (`zend_jit_vm_helpers.c`, exit code 70), so the job installs
Homebrew's LLVM and puts it first on the path ([`662d8d0`](https://github.com/ephpm/php-sdk/commit/662d8d0)):

```yaml
# Use Homebrew LLVM instead of Apple clang — Apple clang crashes
# on PHP's JIT code (zend_jit_vm_helpers.c, exit code 70).
- name: Install LLVM and prepend to PATH
  run: |
    brew install llvm
    echo "$(brew --prefix llvm)/bin" >> "$GITHUB_PATH"
```

On a hosted runner that `brew install` happens on a throwaway image. On my Mac mini it changes the machine. The first run
installs LLVM into `/opt/homebrew`, and from then on every job on that box has it. A `brew install` of something that's
outdated triggers an upgrade, so one job can change the compiler the next job gets. That's convenient right up until a
build breaks and you can't tell if it was your commit or a formula that moved underneath you.

### What works and what doesn't

The good: it's fast. Native arm64, real cores, caches that survive between jobs. A Go module cache and Homebrew bottles
that are already on disk save a lot of minutes compared to a fresh hosted image.

The annoying:

- It's one job at a time per runner process. You can register a second runner from a second directory, but they share
  the same user, the same `/opt/homebrew` and the same keychain.
- macOS updates want to reboot, and a reboot means relying on auto-login to bring the LaunchAgent back.
- No clean slate. Whatever a job installs or leaves in `/tmp` is still there for the next one.
- TCC prompts ("would like to access files in your Documents folder") can block a job waiting on a dialog nobody will
  click. Keeping the runner's work directory in its own home under `_work` avoids the protected folders.

### Where the Mac goes next

The clean-slate problem is the one that keeps bugging me, so I've started running jobs inside throwaway macOS VMs on this
same Mac mini through Apple's Virtualization.framework. It's early, but two things already surprised me. First, the IP of
a VM on the built-in NAT has to be dug out of the host's ARP table, and macOS's `arp` prints MAC addresses without
leading zeros (`a:b:c:...`) while Virtualization.framework hands them to you zero-padded. A string compare never matches
until you normalize them ([`97cd0fe`](https://github.com/ephpm/ephemerd/commit/97cd0fe)):

```go
func normalizeMAC(mac string) string {
	parts := strings.Split(strings.ToLower(strings.TrimSpace(mac)), ":")
	if len(parts) != 6 {
		return strings.ToLower(mac)
	}
	for i, p := range parts {
		if len(p) == 1 {
			parts[i] = "0" + p
		}
	}
	return strings.Join(parts, ":")
}
```

Second, files the host writes into an APFS clone of the VM's disk weren't visible inside the guest, so copying the runner
in ended up as a plain uncompressed `tar` over SSH ([`26e8345`](https://github.com/ephpm/ephemerd/commit/26e8345)). More
on that at the end of the series. For now the plain runner is doing the job.

Next up is the [Windows Dev Kit 2023](/blog/2026/04/self-hosted-runner-windows-dev-kit-arm64/), the arm64 Windows box.

### Links

- [actions/runner v2.333.1](https://github.com/actions/runner/releases/tag/v2.333.1)
- [Configuring the self-hosted runner application as a service](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/configure-the-application)
- [Apple's M1 Mac mini announcement](https://www.apple.com/newsroom/2020/11/introducing-the-next-generation-of-mac/)
