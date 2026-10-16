---
title: "ephemerd: One Daemon, a Fresh Runner Per Job, Every OS"
description: "A single Go binary that runs self-hosted GitHub Actions runners on Linux, Windows and macOS, with a new container or VM for every job."
date: 2026-08-18T21:05:00-07:00
tags: [ go, github-actions, containerd, kubernetes, hyper-v, macos, ephemerd ]
categories: [ ci, go, containers ]
series: [ "ephemerd" ]
images:
 - src: img/2026/08/ephemerd-one-daemon-every-os-arch.svg
   alt: "ephemerd architecture across Linux, Windows and macOS hosts"
   stretch: "none"
draft: false
---

I have CI that needs Linux amd64, Linux arm64, Windows and macOS, and GitHub-hosted runners for all four get expensive fast,
especially the arm64 and Mac minutes. Self-hosted runners fix the bill and create a different problem. A stock
`actions/runner` on a box you own runs whatever the workflow says, keeps whatever the last job left behind, and is one fork
PR away from somebody running code on your LAN.

The usual answers didn't fit. [ARC](https://github.com/actions/actions-runner-controller) wants a Kubernetes cluster, and
realistically Linux nodes: Windows only works in its legacy mode if you bring Windows Kubernetes nodes and build your
own runner image, and macOS isn't on the table. The Firecracker-based runners are Linux only. I wanted one thing I could
install on every box in the rack, Windows and Mac included, that gives each job a brand new environment and throws it
away afterwards. That turned into [ephemerd](https://github.com/ephpm/ephemerd), which I've been building since April
and which now runs all of my CI.

![ephemerd architecture](/img/2026/08/ephemerd-one-daemon-every-os-arch.svg)

### What it is

ephemerd is a single Go binary (MIT licensed) that runs as a system service. It watches your forge for queued jobs, either by
polling the API every 30 seconds or by receiving webhooks, and for each job it:

1. asks GitHub for a just-in-time runner config, good for exactly one job
2. creates a fresh container or VM for that job
3. starts the embedded Actions runner inside it with the JIT config
4. destroys the whole environment when the job completes

There's no runner pool sitting around between jobs and nothing carries over from one job to the next. GitHub is the main
provider, and there are providers for Forgejo, Gitea, GitLab and Woodpecker too.

The part I'm happiest with is that it embeds containerd as a Go library instead of talking to a containerd you installed. If
you've used [k3s](https://k3s.io) or RKE2 that pattern will look familiar, and after my years at Rancher it was the obvious
way to go. It means one binary, one version of containerd, and no "which containerd is on this host" support questions.

### Linux: containerd and runc

On Linux the job is an OCI container on the host, created through the embedded containerd with runc. This is the fast path,
a couple hundred milliseconds to get a job container up. The runner image is a normal OCI image, so the same Dockerfile
builds the job environment for every Linux host.

Since v0.2.2 there's also an opt-in [Kata Containers](https://katacontainers.io/) runtime, which gives each job its own kernel
in a lightweight VM:

```toml
[runner.linux]
runtime = "kata"   # default is "runc"
```

It's not the default because it isn't free. I measured it before merging and the file I/O cost is a lot bigger than the boot
cost, which is a story for its own post. If you set `kata` and the shim or `/dev/kvm` is missing, ephemerd refuses to start
rather than quietly running your jobs on runc.

### Windows: Hyper-V isolated containers, plus a Linux VM

On a Windows host, Windows jobs run in Hyper-V isolated containers. Every job gets its own utility VM and kernel, which is
the isolation level I want for untrusted code on Windows.

Linux jobs on the same Windows box go to a Linux VM that ephemerd boots itself. It started out on WSL2, which can't run under
`LocalSystem`, so ephemerd now boots an Alpine kernel directly through the Host Compute Service API. I wrote that up
in [Escaping WSL: Booting My Own Linux VM Through Hyper-V's HCS API](/blog/2026/06/escaping-wsl-hcs-linux-vm/). The Windows
daemon keeps the GitHub credentials and the scheduler, and sends container lifecycle calls to the ephemerd inside the VM
over gRPC.

### macOS: a VM per job

On Apple silicon there are two kinds of jobs. macOS jobs (Xcode, Swift, anything that needs a real macOS) get their own
macOS VM through Virtualization.framework. The base disk comes from one of Cirrus Labs'
[Tart](https://tart.run) macOS images, pulled as an OCI artifact, and each job's disk is an APFS clone of it. A clone is
copy-on-write, so creating one is instant and costs no disk until the job writes something. When the job is done the clone
is deleted.

Linux jobs on the Mac go to a long-lived Linux VM (also Virtualization.framework) running containerd, the same model as the
Windows host. The same Linux runner image works on all three.

### The parts jobs actually notice

A lot of the work wasn't about isolation at all, it was about making real workflows run unmodified inside it.

**A Docker API.** Half of all workflows run `docker build` or `docker run` somewhere, and `container:` jobs need a Docker
daemon for the runner to drive. Rather than run Docker-in-Docker, ephemerd serves a Docker-compatible API per job (backed by
containerd in a per-job namespace) and points the job at it, with BuildKit embedded for `docker build`. Sibling containers a
job starts are torn down with the job.

**Pull-through caches.** Every job starting from nothing means every job downloads the world. ephemerd runs pull-through
caches for Go modules, npm, pip, cargo and pub, and injects the right environment variables (`GOPROXY` and friends) into the
job. Container pulls can be routed through a registry mirror on your LAN. The motivation is in the registry mirror's own
package docs: one Linux node pulled 294 GB in 4.1 days, and the single biggest contributor was one 1.1 GB CI image pulled
163 times in a week.

**An egress fence.** A job container can reach the internet and nothing on the private ranges: `10.0.0.0/8`,
`172.16.0.0/12`, `192.168.0.0/16` and `169.254.0.0/16` are blocked. On Linux that's iptables. On Windows it turned out to
be a long fight, and since v0.2.0 the working answer is an opt-in L2Bridge network with VFP ACLs.

### Getting started

Installation is the same command on all three platforms, and it registers the right kind of service (systemd, launchd or the
Windows SCM):

```bash
ephemerd install
ephemerd start
ephemerd status
```

The minimum config is the owner you want runners for, plus credentials (a token in the environment, or GitHub App
credentials in the config):

```toml
[github]
owner = "your-org"
# repos = ["repo1", "repo2"]   # omit for org-level runners
```

And from a workflow it looks like any other self-hosted runner:

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on ephemerd"
```

If you'd rather not expose anything to the internet, the default polling mode needs no inbound ports at all. For instant
dispatch it can run a Cloudflare tunnel, ngrok or localtunnel as a managed subprocess and register the webhooks itself.

### Where it's at

It's at v0.2.3 as I write this, and it runs my whole fleet: Linux amd64 and arm64 nodes, a Windows Server 2025
node and a Mac mini. The repo has a containment test suite that runs from inside a real job and tries to reach things it
shouldn't, which has been the most useful thing in the project for finding out which of my assumptions were wrong.

There's a Hugo docs site in the repo, and `ephemerd doctor` checks the host prerequisites for whatever platform it's on.
There's no Windows arm64 build yet. Next up is writing down some of the more painful lessons from the last few months,
starting with Windows networking.

### Links

- [ephemerd on GitHub](https://github.com/ephpm/ephemerd)
- [Releases](https://github.com/ephpm/ephemerd/releases)
- [Escaping WSL: Booting My Own Linux VM Through Hyper-V's HCS API](/blog/2026/06/escaping-wsl-hcs-linux-vm/)
- [containerd](https://containerd.io/), [Kata Containers](https://katacontainers.io/), [Tart](https://tart.run)
