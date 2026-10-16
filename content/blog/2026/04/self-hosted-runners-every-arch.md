---
title: "Self-Hosted GitHub Runners on Every Arch"
description: "Series overview: five OS/arch runner targets on three boxes, and what the hosted alternative costs per minute."
date: 2026-04-08T21:41:00-07:00
tags: [ github-actions, ci, self-hosted-runners, homelab, arm64, windows, macos, linux ]
categories: [ ci, homelab ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/04/self-hosted-runners-every-arch-matrix.svg
   alt: "Diagram of three machines (M1 Mac mini, Windows Dev Kit 2023, Proxmox host) running five runner targets, all dialing out to github.com"
   stretch: "none"
draft: true
---

On April 1st I started a repo called [php-sdk](https://github.com/ephpm/php-sdk). It builds PHP as a static embed library
for ePHPm, and it has to do that for Linux x86_64, Linux aarch64, macOS arm64 and Windows x86_64 (plus macOS x86_64,
for the first night). The first commit ran everything hosted: `ubuntu-latest` for both Linux legs (aarch64
cross-compiled), `macos-latest` and `macos-13`, and a "Windows" job that just downloaded windows.php.net binaries on
Ubuntu; 40 minutes later the aarch64 leg moved to `ubuntu-24.04-arm`. About two hours in, the Linux x86_64 leg was on
`[self-hosted, linux, x64]` ([`9f809bb`](https://github.com/ephpm/php-sdk/commit/9f809bb)) and an hour after that
Windows was too ([`662d8d0`](https://github.com/ephpm/php-sdk/commit/662d8d0)).

The hosted legs that were left didn't last long either. From the first run on April 2nd (and again in
[run 23994291885](https://github.com/ephpm/php-sdk/actions/runs/23994291885) on April 4th) the macOS and arm64 jobs
never started: GitHub refused to schedule hosted jobs with the annotation "recent account payments have failed or your
spending limit needs to be increased".

<!-- TODO(luther): confirm the ephpm repos were still private at this point (that's why hosted minutes were billed at all), and say in a sentence what the billing situation was. -->

Meanwhile the two jobs on my own boxes, `linux-github-runner` and `windows-github-runner`, went green. That about sums up
the argument for this series.

### Why bother

Most of what I write is Go, and Go cross-compiles so well that it's easy to forget you never actually *ran* the thing on
the other platforms. [go-proxmox](https://github.com/luthermonson/go-proxmox) gets used from Macs and Windows laptops.
[goodhosts](https://github.com/goodhosts/cli) edits `/etc/hosts` and `C:\Windows\System32\drivers\etc\hosts`, and the
Windows path is the one that breaks. ePHPm embeds PHP through C, so a build that compiles for `aarch64` and crashes at
startup on `aarch64` is a real possibility. Cross-compiling tells you it links. It doesn't tell you it works.

So the goal is a runner for each of these:

| Target | Hardware | How | Post |
|---|---|---|---|
| macOS arm64 | M1 Mac mini | bare metal, launchd | [Part 1](/blog/2026/04/self-hosted-runner-m1-mac-mini/) |
| Windows arm64 | Windows Dev Kit 2023 | bare metal, Windows service | [Part 2](/blog/2026/04/self-hosted-runner-windows-dev-kit-arm64/) |
| Linux arm64 | Hyper-V VM on the Dev Kit | VM, systemd | [Part 3](/blog/2026/04/self-hosted-runner-linux-arm64-hyper-v/) |
| Linux x86_64 | Proxmox VE | VM, systemd | [Part 4](/blog/2026/04/self-hosted-runner-proxmox-linux/) |
| Windows x86_64 | Proxmox VE | VM, Windows service | [Part 5](/blog/2026/04/self-hosted-runner-proxmox-windows/) |
| All of them | | one workflow matrix | [Part 6](/blog/2026/05/self-hosted-runner-matrix-workflow/) |

![Three machines and five runner targets](/img/2026/04/self-hosted-runners-every-arch-matrix.svg)

Three physical boxes cover five targets. The Proxmox host was already in my lab. The other two are cheap because they're
discontinued: the M1 Mac mini shipped in November 2020 at
[$699](https://www.apple.com/newsroom/2020/11/introducing-the-next-generation-of-mac/) and has since been replaced
twice, and Microsoft's [Windows Dev Kit 2023](https://learn.microsoft.com/en-us/windows/arm/dev-kit/) (Snapdragon 8cx
Gen 3, 32GB of RAM, 512GB NVMe) is "no longer available for new purchases". Both show up used and refurbished.

<!-- TODO(luther): what you actually paid for the Mac mini and the Dev Kit, where you bought them, and the RAM/SSD config of the Mac mini. -->

### What GitHub charges for the same thing

Self-hosted runners use no billed Actions minutes. GitHub announced a $0.002/minute "cloud platform" charge
for self-hosted runners in December, set to start March 1st, then
[postponed it](https://github.blog/changelog/2025-12-16-coming-soon-simpler-pricing-and-a-better-experience-for-github-actions/)
to re-evaluate. Hosted runner prices went down on
[January 1st](https://github.blog/changelog/2026-01-01-reduced-pricing-for-github-hosted-runners-usage/). Public repos
on standard hosted runners are free, and private repos get a monthly allowance (2,000 minutes on the Free plan) before
billing kicks in.

Past the allowance, these are the per-minute rates on the
[runner pricing page](https://docs.github.com/en/billing/reference/actions-runner-pricing)
([snapshot from the evening of April 8th, 2026](http://web.archive.org/web/20260409042038/https://docs.github.com/en/billing/reference/actions-runner-pricing))
for the standard runner that matches each target:

| Hosted runner | SKU | USD / minute |
|---|---|---|
| Linux 2-core (x64) | `actions_linux` | $0.006 |
| Linux 2-core (arm64) | `actions_linux_arm` | $0.005 |
| Windows 2-core (x64) | `actions_windows` | $0.010 |
| Windows 2-core (arm64) | `actions_windows_arm` | $0.010 |
| macOS 3-core or 4-core (M1 or Intel) | `actions_macos` | $0.062 |

![Cost of 1,000 minutes on each standard hosted runner](/img/2026/04/self-hosted-runners-every-arch-cost.svg)

Linux and Windows minutes are cheap enough that you'd need a lot of builds before hardware pays for itself. macOS is the
outlier. At $0.062 a minute, 1,000 minutes is $62, which is more than ten times the Linux price. A PHP build with a
static toolchain is not a two minute job, and a matrix that rebuilds on every push burns through minutes quickly.

<!-- TODO(luther): if you want a break-even line, add your hardware cost and rough monthly build minutes here. Don't estimate them for the post, use real numbers. -->

Money isn't the only reason though. The standard hosted runners for private repos are 2-core (3 or 4 on macOS) unless
you pay for larger ones, they
start from a clean image so your caches are gone on each job, and you can't put a test database or a Proxmox API next to
them. My runners sit on the same network as my lab, which matters when the thing under test is a Proxmox client.

### The catch

A self-hosted runner is a long-lived machine that runs whatever code a workflow tells it to. GitHub's own docs say not to
use them with public repos for exactly that reason, because a pull request from a fork can run on your box. Jobs also
leave state behind: the php-sdk workflow grew a `Clean up previous builds` step the same night it moved to self-hosted
runners, because a runner that never gets wiped still has the last run's `buildroot/`, `source/` and `downloads/`
sitting in its work directory. I'll come back to both of these in
[Part 4](/blog/2026/04/self-hosted-runner-proxmox-linux/) and
[Part 6](/blog/2026/05/self-hosted-runner-matrix-workflow/).

### What this series covers

These are plain [actions/runner](https://github.com/actions/runner) installs, version 2.333.1 (released March 27th).
Register, install as a service, label, and point a workflow at it. No autoscaling controller, no Kubernetes. Each post
covers getting one target registered and the quirks I hit on that platform, and the last post puts them in a single
workflow matrix.

### Links

- [actions/runner releases](https://github.com/actions/runner/releases)
- [GitHub Actions runner pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)
- [Windows Dev Kit 2023 on Microsoft Learn](https://learn.microsoft.com/en-us/windows/arm/dev-kit/)
- [ephpm/php-sdk](https://github.com/ephpm/php-sdk)
