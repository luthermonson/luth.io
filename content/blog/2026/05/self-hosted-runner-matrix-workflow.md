---
title: "Self-Hosted Runner Part 6: One Workflow Matrix Across Five Runners"
description: "Labels, a matrix that hits macOS arm64, Windows arm64, Linux arm64, Linux x64 and Windows x64, keeping the runners updated, and whether it was worth it."
date: 2026-05-02T15:40:00-07:00
tags: [ github-actions, self-hosted-runners, ci, go, homelab, arm64 ]
categories: [ ci, homelab ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/05/self-hosted-runner-matrix-workflow-labels.svg
   alt: "Diagram mapping five runs-on label sets to the five registered self-hosted runners"
   stretch: "none"
draft: true
---

Five runners are registered: the [Mac mini](/blog/2026/04/self-hosted-runner-m1-mac-mini/), the
[Dev Kit](/blog/2026/04/self-hosted-runner-windows-dev-kit-arm64/), the
[Linux VM on the Dev Kit](/blog/2026/04/self-hosted-runner-linux-arm64-hyper-v/), and the
[Linux](/blog/2026/04/self-hosted-runner-proxmox-linux/) and [Windows](/blog/2026/04/self-hosted-runner-proxmox-windows/)
VMs on Proxmox. This last post in the [series](/blog/2026/04/self-hosted-runners-every-arch/) is about using them from a
workflow, keeping them alive, and whether the whole thing paid off.

### How runs-on picks a runner

When `runs-on` is a list, a runner has to have **every** label in it to take the job. It's a subset match, not an OR.
`[self-hosted, Linux]` would match both Linux runners, and you don't get to choose which one. `[self-hosted, Linux, ARM64]`
only matches the VM on the Dev Kit. Label matching is case-insensitive, so `linux` and `Linux` are the same label.

![runs-on label sets matched to runners](/img/2026/05/self-hosted-runner-matrix-workflow-labels.svg)

Every runner registered with `config.sh` gets three labels unless you pass `--no-default-labels`:

| Runner | Default labels |
|---|---|
| Mac mini | `self-hosted`, `macOS`, `ARM64` |
| Dev Kit | `self-hosted`, `Windows`, `ARM64` |
| Dev Kit Linux VM | `self-hosted`, `Linux`, `ARM64` |
| `linux-github-runner` | `self-hosted`, `Linux`, `X64` |
| `windows-github-runner` | `self-hosted`, `Windows`, `X64` |

<!-- TODO(luther): fill in the real runner names for the Mac mini, Dev Kit and the Hyper-V VM (the diagram uses mac-mini / devkit / devkit-linux as placeholders). -->

With one runner per OS/arch pair those three labels are already a complete addressing scheme. That's the main reason I
haven't added many custom ones.

### Labeling strategy

When I do add labels, they describe **what a runner can do**, never which box it is:

- `docker` on the runners where a job can talk to a Docker daemon (the two Linux VMs)
- `lab` on runners that sit on the lab network and can reach things like a test Proxmox API

A workflow that needs Docker asks for `[self-hosted, Linux, X64, docker]`. If I add a second Linux x64 runner with Docker
later, jobs spread across both and nothing changes in the YAML. If I'd labeled them `proxmox-vm-1` I'd be editing
workflows every time the hardware changed.

<!-- TODO(luther): list the custom labels you actually use, or delete the examples above if you only use defaults. -->

Labels can go on at registration with `--labels docker,lab`, or onto an existing runner through the API without
re-registering it:

```sh
# find the runner's id
gh api orgs/ephpm/actions/runners --jq '.runners[] | "\(.id) \(.name) \([.labels[].name] | join(","))"'

# add labels to it
gh api -X POST orgs/ephpm/actions/runners/<id>/labels -f 'labels[]=docker' -f 'labels[]=lab'
```

Both need a token with org admin rights for self-hosted runners.

For org-level runners, also look at the runner group. The default group doesn't let public repositories use its runners
unless you turn that on, and you can restrict it to a list of repos. That's the setting that keeps a random public repo in
the org from landing jobs on the Mac mini.

### The workflow

Here's the matrix I use for Go projects. It builds and tests on each target natively. No cross compiling, no emulation:

```yaml
name: test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    name: ${{ matrix.os }}-${{ matrix.arch }}
    # keep fork PRs off the self-hosted runners
    if: github.event_name == 'push' || github.event.pull_request.head.repo.full_name == github.repository
    runs-on: [self-hosted, "${{ matrix.os }}", "${{ matrix.arch }}"]
    strategy:
      fail-fast: false
      matrix:
        include:
          - { os: macOS,   arch: ARM64 }
          - { os: Windows, arch: ARM64 }
          - { os: Linux,   arch: ARM64 }
          - { os: Linux,   arch: X64 }
          - { os: Windows, arch: X64 }
    defaults:
      run:
        shell: bash
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: false   # the module and build cache already live on the runner

      - name: Toolchain
        run: go env GOOS GOARCH GOHOSTARCH

      - name: Test
        run: go test ./...

      - name: Build
        run: go build -o out/ ./...
```

The parts that are specific to self-hosted runners:

- **`fail-fast: false`.** One slow or broken runner shouldn't cancel the other four. When the Windows arm64 leg fails I
  still want to know whether macOS passed.
- **The `if:` guard.** A pull request from a fork runs the PR author's code. On a hosted runner that's GitHub's VM to
  worry about. On mine it's a machine on my network that keeps state between jobs. The guard skips the job for PRs from
  forks entirely; those get a hosted-runner workflow or a maintainer runs them by hand.
- **`cache: false`.** setup-go's cache uploads the Go module and build cache to GitHub's cache service and restores it at
  the start of each job. On a hosted runner that's a big speedup. On a self-hosted runner the caches are already on disk
  from the last job, and round-tripping them through GitHub is slower than not doing it.
- **`shell: bash` on Windows.** On a Windows runner `shell: bash` means the bash that comes with Git for Windows. It lets
  one step work on all five targets. Remember the [tar story from Part 5](/blog/2026/04/self-hosted-runner-proxmox-windows/)
  though: inside Git Bash, Git's MinGW tools come first on `PATH`.
- **`GOHOSTARCH` in the log.** It's how I find out that a toolchain on the Dev Kit got swapped for an x64 one and the
  "arm64" leg is quietly running under emulation.

You can mix hosted runners into the same matrix by making `runs-on` come from the matrix. That's how php-sdk ran through most of
April, with `runner: [self-hosted, linux, x64]` next to `runner: ubuntu-24.04-arm` and `runs-on: ${{ matrix.runner }}`,
until the arm64 leg moved to a self-hosted runner too.

<!-- TODO(luther): link a real repo/run where this matrix ran green across all five, or adjust the YAML to match what you actually use. -->

### Keeping them running

**The runner software updates itself.** By default the runner checks for new versions and updates in place between jobs.
Don't turn that off with `--disableupdate` unless you have something else doing it: per the
[self-hosted runner reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners), if a runner
isn't updated within 30 days of a new release, GitHub stops queueing jobs to it. The series was written against 2.333.1
(March 27th), and 2.334.0 came out on April 21st.

**The operating systems don't.** Each box has its own update story:

- The Mac mini needs macOS updates applied, then has to auto-login so the LaunchAgent comes back.
- The Dev Kit and the Windows VM follow Windows Update's restart schedule. Set active hours so a reboot doesn't land in
  the middle of a build.
- The Linux VMs get `unattended-upgrades`, plus an occasional manual reboot for kernel updates.

**Toolchains drift.** On a hosted runner, the image is updated for you weekly. On these runners, Go, Homebrew formulae,
Visual Studio Build Tools and whatever the last job installed stay exactly as they were until I change them. That's good
for reproducibility and bad when a project bumps its minimum Go version and three of the five legs fail at the same time.

**Check that they're online.** The org runners page shows status, and so does the API:

```sh
gh api orgs/ephpm/actions/runners \
  --jq '.runners[] | "\(.name)\t\(.status)\tbusy=\(.busy)"'
```

A runner that says `offline` is usually a Mac that rebooted and didn't log in, or a Dev Kit that went to sleep.

<!-- TODO(luther): what you automated for monitoring (a cron job, Uptime Kuma, etc.). -->

### Was it worth it

**Coverage: yes, easily.** This is the thing I couldn't get any other way at this price. Five native targets, including the
two that hosted CI is worst at, Windows on arm64 and macOS. Cross-compiled binaries are a promise that the code builds.
Running the tests natively is what checks that it works.

<!-- TODO(luther): the bug or behavior difference this setup actually caught that cross-compiling wouldn't have. That's the best paragraph this post could have. -->

**Cost: it depends on your mix.** Linux and Windows hosted minutes are cheap ($0.006 and $0.010 a minute on the
[April pricing](https://docs.github.com/en/billing/reference/actions-runner-pricing)), and if your repos are public, standard
hosted runners cost nothing. macOS at $0.062 a minute is where owning a $699-at-launch Mac mini starts to make sense, and
the blocked macOS and arm64 jobs in the [overview](/blog/2026/04/self-hosted-runners-every-arch/) are what pushed me into
doing this in the first place.

<!-- TODO(luther): what you paid for the hardware vs. what the same minutes would have cost hosted, using real build minutes from your Actions usage page. -->

**Convenience: mixed.** The good part is speed and warm caches, plus being able to RDP or SSH into the exact machine a job
failed on. The bad part is the "keeping them running" section above, and one thing that keeps coming up in every
post of this series: these runners never forget. php-sdk's first night needed cleanup steps for leftover build trees, a
`C:\php` the job deletes before it starts, and a patched static-php-cli in `C:\spc-src` that, for three weeks, existed on
exactly one machine. A runner that keeps state between jobs makes builds fast, and it also lets your builds start depending on what's
installed on that one machine. And none of it is safe for code you don't trust.

### What's next

That last problem, a fresh environment for each job on hardware I own, on Linux, Windows and macOS, turned into its own
project. It's already running php-sdk's builds. It gets its own posts.

### Links

- [Choosing the runner for a job](https://docs.github.com/actions/using-jobs/choosing-the-runner-for-a-job)
- [Self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
- [REST API: self-hosted runners](https://docs.github.com/en/rest/actions/self-hosted-runners)
- [actions/runner releases](https://github.com/actions/runner/releases)
