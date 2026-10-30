---
title: "Docker-in-Docker in CI Without Handing Jobs Root"
description: "ephemerd runs Docker-in-Docker for container: jobs without a privileged container. Here is what a job could still reach, and the four PRs that closed each hole."
date: 2026-10-24T10:00:00-07:00
tags: [ docker, dind, containers, security, seccomp, overlayfs, cgroups, runc, ephemerd ]
categories: [ ephemerd, containers, security, ci ]
series: [ "ephemerd" ]
images:
 - src: img/2026/10/docker-in-docker-without-root-cracks.svg
   alt: "Four dind sandbox cracks and the ephemerd PR that closed each one"
   stretch: "none"
draft: false
---

[ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) runs a fresh GitHub Actions runner per job. Some of those
jobs need Docker. A workflow builds an image, runs a service container, or uses the `container:` key. The usual way to
give a self-hosted job Docker is Docker-in-Docker, or dind.

The catch is what dind normally costs you. The standard recipe is a `--privileged` container. Privileged hands the job
most of root on the host. For CI that runs fork pull requests, that is a non-starter. A fork PR is a stranger's code.

So ephemerd runs dind without `--privileged` and without a real root hand-off. This post is the hardening that made
that safe. Four holes, four PRs. One of them I already wrote up, so I'll link it and move on.

### What a dind job is here

A `container:` job on ephemerd is a runc container. The job process runs as uid 0 inside it. There is no user
namespace. So container uid 0 is host uid 0.

Read that again. The job is root, and its root is the host's root number.

The only thing between the job and the host is the sandbox. That sandbox is five things:

- the seccomp filter
- the dropped capabilities
- the mount namespace
- the network namespace
- the cgroup limits

Each hole below was a crack in one of them.

![Four dind sandbox cracks and the ephemerd fix for each](/img/2026/10/docker-in-docker-without-root-cracks.svg)

### 1. Re-walking a job-controlled path

[The bind-mount post](/blog/2026/09/bind-mounts-across-mount-namespaces/) covered this one in full, so here is the
one-line version. The dind shim validated a bind source as a path string. runc then walked that string again in its
own process. The job renamed the directory in between and pointed a symlink at `/`. runc mounted the host root into
the container. [#173](https://github.com/ephpm/ephemerd/pull/173) fixed it by resolving the path once to a descriptor
and staging the bind in ephemerd's own mount namespace. It shipped in v0.2.4. Go read the other post for the details.

### 2. Stripping its own seccomp sandbox

ephemerd has a gate. It is the config knob `dind.allow_privileged`. With it off, a sibling container cannot ask for the
elevation stack. `--privileged` is refused. Added capabilities are refused.

The gate had a hole. It checked the `Privileged` and `CapAdd` fields and nothing else. Meanwhile a separate loop read
the `SecurityOpt` field unconditionally. A job could pass `--security-opt seccomp=unconfined`. That mapped straight to
running the container with no seccomp filter. The gate never looked.

seccomp is the filter that blocks dangerous syscalls. Dropping it on a container whose uid 0 is the host's uid 0 is
exactly the elevation the gate was meant to deny.

[#174](https://github.com/ephpm/ephemerd/pull/174) closed it. The fix is one predicate, `elevatingSecurityOpt`, that
answers one question. Does this option ask for elevation? The gate scans `SecurityOpt` through it. The applier switches
on the same predicate's output. Gate and applier can no longer disagree, because they read the same function. That
divergence was the bug.

With the gate closed the job gets a 403:

```
--security-opt seccomp=unconfined is disabled on this host: it removes the seccomp
sandbox that keeps an unprivileged sibling container unprivileged
(set dind.allow_privileged = true in ephemerd config to enable)
```

The PR also did the obvious follow-up. If one field was missed, what else reaches the OCI spec? The audit found the
Docker `HostConfig` decoder keeps only eleven fields. Everything else is dropped at decode time. `--device`,
`--pid=host`, `--sysctl`, `CgroupParent` and a dozen more never reach the spec at all. `NetworkMode: host` is not
honoured either. The sibling always gets its own network namespace. The gate held everywhere else. #174 shipped in
v0.2.4.

One honest footnote. `--tmpfs` options replace the default `nosuid,nodev` wholesale. So a job can mount a tmpfs with
neither set. That matches real Docker's behaviour. It is low severity here and it is filed, not fixed.

### 3. Poisoning the shared runner tree

This one is my favourite, because it needs no privilege at all.

Each job gets its own copy of the runner binaries. On Linux that copy used to be a `cp -al` hardlink farm over a
shared, persistent tree. Hardlinks share inodes. The per-job directory was bind-mounted read-write, and container root
is host root. So a write in place went straight to the shared inode.

The GitHub Actions runner writes in place at startup. `run.sh` regenerates `run-helper.sh` on every start. That alone
mutated the shared file. A malicious job did not even have to try. An `echo x >> /actions-runner/run.sh` from one job
landed in every concurrent job and every future one. Including a maintainer's job carrying secrets. That is the thing
that makes anonymous fork-PR CI unsafe.

A plain read-only bind does not work. The runner has to write `.env`, `.path`, `_diag/` and the whole `_work/` tree.
Make the binaries read-only and the runner breaks on its first start.

[#176](https://github.com/ephpm/ephemerd/pull/176) used an overlay instead. The shared tree is the read-only
`lowerdir`. Each job gets a private `upperdir` and `workdir`. The runner sees a fully writable `/actions-runner`.
Every write copies up into the job's own layer. No write can reach the shared inodes. The dind bind table points the
runner mount at the job's `upperdir`, so a sibling's `-v /actions-runner/_work/...` resolves to the exact bytes the
runner sees. It shipped in v0.2.5.

The test I like has teeth. It reproduces the old `cp -al` farm and asserts the check catches it, then asserts the
overlay passes. A green test that cannot fail proves nothing.

### 4. Starving the neighbours

The last hole is not an escape. It is one job hurting the others on the box.

ephemerd packs several jobs onto a node. The heavy work in a `container:` job runs in dind siblings. Those siblings
share one parent cgroup per job. Nothing limited that parent's memory. So one job could use the whole node.

On a 40 GB node with five job slots, one morning a single out-of-memory event killed processes in three concurrent
jobs at once. One of them was a 2 GB linker. One greedy job, three failed builds that had done nothing wrong.

There was a slow leak too. runc creates the per-job parent cgroup and only ever removes the leaf cgroups under it. The
empty parent stayed. They piled up into the hundreds per node, over five hundred on the busiest.

[#219](https://github.com/ephpm/ephemerd/pull/219) added an opt-in per-job memory budget. It is off by default. In
config:

```toml
[runtime]
memory_limit_mb = 16384      # 0 or unset = no limit, unchanged behaviour
memory_high_percent = 90     # memory.high = 90% of the limit
```

It is a per-job budget, not per container. A per-container limit would let a job use N times the limit across its
siblings. So the runner container moves under the same `/ephemerd-dind-<id>` parent the siblings already share. The
parent carries `memory.max` and `memory.high`. The job gets throttled and its page cache reclaimed before anything
dies. If the parent cannot be created, the job runs unlimited with a warning. Fail open.

`memory.oom.group` is deliberately not set. If an OOM has to kill, it kills the largest process in the job, usually the
linker, and the step fails. Setting `oom.group` would kill the runner too. That turns a failed step into a lost job.

The leak got a teardown step and a sweeper. Teardown removes the parent after the task is deleted. The sweeper removes
any `ephemerd-dind-*` cgroup with no processes, no children, and ten minutes idle. It touches nothing else.

This is the one that has not shipped yet. It merged on 2026-10-23, just after v0.2.22 was cut. It is in `main` and
coming in the next release. The budget still needs a run on real cgroupfs before I trust the numbers.

### In short

- dind on ephemerd runs without `--privileged`. The job is still root in its container, and that is host uid 0, so the
  sandbox is the whole defence.
- The privileged gate reads one predicate now, so a job cannot strip its own seccomp filter past it. (#174, v0.2.4)
- The shared runner tree is a read-only overlay lower, so no job can write into another's bytes. (#176, v0.2.5)
- Jobs can get a per-job memory budget on their shared cgroup, and leaked cgroups get swept. (#219, merged, next
  release)
- The TOCTOU bind escape has its own post. (#173, v0.2.4)

### Caveats

Everything here is Linux-only. Windows jobs get a Hyper-V isolated container and macOS jobs get a VM, so the mechanisms
differ. The memory budget and cgroup sweep in #219 have passed their tests under WSL. They have not run on a live node's
cgroupfs yet, and that PR is not in a release. The `--tmpfs` option gap from #174 is noted but not closed. None of this
makes a shared-kernel runc container as strong as a VM. For the stronger boundary I looked at
[Kata Containers](/blog/2026/09/kata-containers-for-ci/), which is a different trade-off.

### What's next

The memory budget gets a staged rollout. One node first, with a few days of per-job memory sampling before the limit
goes on. After that, three follow-ups are still open:

- the `--tmpfs` option gap
- the hardening flags that get silently ignored, which confuse more than they harm
- the exec process spec, which still needs a Linux host to settle

### Links

- [ephemerd: one daemon, a fresh runner per job, every OS](/blog/2026/08/ephemerd-one-daemon-every-os/)
- [Bind mounts across mount namespaces: four PRs and a container escape](/blog/2026/09/bind-mounts-across-mount-namespaces/)
- [Kata Containers for CI: boot cost isn't the problem, file I/O is](/blog/2026/09/kata-containers-for-ci/)
- [My iptables rule matched 0 packets in 9.8 GB](/blog/2026/08/iptables-rule-zero-packets/)
- PRs: [#173](https://github.com/ephpm/ephemerd/pull/173), [#174](https://github.com/ephpm/ephemerd/pull/174), [#176](https://github.com/ephpm/ephemerd/pull/176), [#219](https://github.com/ephpm/ephemerd/pull/219)
