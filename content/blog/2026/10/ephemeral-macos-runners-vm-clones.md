---
title: "Ephemeral macOS Runners: A Fresh VM Per Job on Apple Silicon"
description: "How ephemerd runs each macOS CI job in a throwaway Virtualization.framework VM, and the slot leaks and disk limits I had to fix to make it reliable."
date: 2026-10-23T11:00:00-07:00
tags: [ go, macos, github-actions, virtualization, ci, ephemerd ]
categories: [ ephemerd, ci, go ]
series: [ "ephemerd" ]
images:
 - src: img/2026/10/ephemeral-macos-runners-vm-clones-lifecycle.svg
   alt: "A macOS CI job lifecycle: clone the base image, boot and grow the disk, set up over SSH, run the job, then stop and sweep, with a band showing the single slot held the whole time"
   stretch: "none"
draft: false
---

[ephemerd](https://github.com/ephpm/ephemerd) gives every CI job a brand new environment and throws it away after.
On Linux and Windows that environment is a container. On macOS it can't be. Apple's license wants macOS running on
Apple hardware, so the Mac in my rack is the real thing. The isolation comes from a VM instead.

Each macOS job gets a fresh [Virtualization.framework](https://developer.apple.com/documentation/virtualization) VM
cloned from a base image. The job runs inside it. The VM is destroyed when the job finishes. That's the same contract
as the container path. Getting there took four fixes that I only found by running the fleet. This post walks the VM
lifecycle and the three ways it bit me:

- a wedged VM that held the only slot
- jobs that fetched straight from the internet
- job disks that ran out mid-build

If you want the background on ephemerd itself, I wrote that up in
[One Daemon, a Fresh Runner Per Job, Every OS](/blog/2026/08/ephemerd-one-daemon-every-os/).

### The VM-clone lifecycle

The Mac holds one base image. ephemerd never boots it. For each job it makes an APFS clone of `base.img`. APFS clones
are copy-on-write. The clone starts out sharing every block with the base. The guest only consumes host disk for the
blocks it writes. So a clone is cheap to make and cheap to throw away.

![macOS VM job lifecycle and the single concurrency slot](/img/2026/10/ephemeral-macos-runners-vm-clones-lifecycle.svg)

The steps are:

1. clone `base.img` for this job
2. boot the VM from the clone
3. set the runner up over SSH inside the guest
4. start the embedded Actions runner with a just-in-time config good for one job
5. stop the VM and remove the clone

The container paths reach "tracked and running" almost immediately. The macOS path has a slow provisioning phase in
between. It boots a whole OS. Then it waits for the guest's SSH port to open. Then it runs setup commands over SSH.
That gap is where the trouble lived.

The Mac pool runs with `max_concurrent = 1`. One VM at a time. So there is exactly one concurrency slot. A job takes
the slot the moment it starts cloning. It gives the slot back when the VM is gone. Hold that fact. It's the whole
story of the first two fixes.

One slot is my choice. Two is the hard limit. Apple's licensing lets a single Mac run at most two macOS VMs at once.
A faster Mac doesn't change that. So the ceiling here is two, not the core count. I run one job VM at a time. That
leaves the second slot free. It matters during teardown. The old job's VM can still be stopping while the next one
boots, and both stay under the cap.

### A wedged VM that never let go

The first failure took out macOS CI fleet-wide. One job logged `provisioning macOS VM runner`, then `booting macOS
VM`, then `waiting for macOS VM runner to become reachable`. Then it went silent for about ten hours. The VM kept
running the whole time at 57% CPU. Because the pool holds one slot, that one stuck VM pinned it. Every later macOS job
just logged its queued webhook and sat there. macOS CI was dead for about twelve hours. The worst part: `ephemerd jobs`
reported no running jobs the entire time. Recovery needed a human to kill the VM and restart the daemon.

The root cause was an unbounded wait inside SSH. The provisioning step opens an SSH session to the guest and runs
setup commands. The Go SSH client's `Timeout` bounds connection setup only. It does not bound command execution. Those
command calls took no context either. So a guest command that never returned blocked the whole provisioning wait
forever. The job hung before it was ever tracked as running. That's why it was invisible to `ephemerd jobs` and to the
orphan sweep. It already held the slot, though.

[PR #178](https://github.com/ephpm/ephemerd/pull/178) bounded the whole thing. The scheduler now wraps the
reachability wait in a provisioning deadline. The default is five minutes. It's a config knob, `provision_timeout`
under `[vm.macos]`. On the deadline it force-stops the VM. Force-stopping drops the guest connection, which unblocks
the stuck SSH call. Then the normal failure path releases the slot and deregisters the runner. Each remote SSH command
also got its own hard deadline. The VM's `Stop()` became idempotent, because a watchdog and a normal teardown can both
call it.

### The deadline path leaked too

I thought #178 closed it. It didn't. A few weeks later the Mac served zero macOS jobs for about twenty-eight hours
while reporting itself healthy. Status was `ok`. Active jobs read zero. `/healthz` returned 200. Every queued macOS
job sat until GitHub cancelled it at the 24-hour timeout.

The deadline path had its own bug. After the force-stop, the code waited for the provisioning goroutine to unwind.
That wait was unbounded. It assumed the force-stop would always drop the guest connection and let the wait return. On
metal the guest ignored the stop. The log even showed `macOS VM did not stop gracefully, forcing`. So the wait parked
forever, still holding the slot. The decisive evidence was a log line that never appeared. The error branch logs
`macOS VM runner not reachable`. That line was nowhere in the twenty-eight-hour window. There was a second,
independent leak on the retry path for the same reason.

[PR #197](https://github.com/ephpm/ephemerd/pull/197) stopped coupling the slot release to teardown. The VM stop runs
off-thread now. The unwind is bounded. The slot is released before teardown on every macOS path. If a wedged stop
never returns, that one goroutine is leaked on purpose and logged. Leaking one goroutine is the correct trade against
pinning a pool's only slot.

The reason this hid for a day straight was observability. So that got fixed in the same PR. `/healthz`, the status RPC
and `ephemerd status` now report `held_slots` and `slot_capacity` per pool. Held equal to capacity with zero active
jobs is the outage, visible at a glance. A blocked slot acquire now logs too. It warns at thirty seconds and
repeats every five minutes. After fifteen minutes blocked, with slots held and nothing tracked, it escalates to an
error named `suspected slot leak`.

### Jobs were fetching straight from the internet

The next one wasn't an outage. It was a bandwidth bill. The Mac was pulling around 130 GB a day of WAN traffic. ephemerd
runs local caching proxies for gomod, cargo, GitHub releases and Composer. Those caches were still at zero bytes.

The cause was structural. The cache-proxy environment is injected into containers. A macOS VM job has no container. Its
runner starts over SSH inside the guest, and nothing was passing it any proxy env. So every macOS job fetched
dependencies straight from the internet.

[PR #211](https://github.com/ephpm/ephemerd/pull/211) routes the env in through the runner's `.env` file, which the
GitHub runner loads into every job. The proxy addresses point at the host's NAT gateway as seen from inside the guest.
Before the runner starts, one bounded SSH command probes every proxy address from inside the VM. It only stages the
env if all of them answer. The log says either `cache proxies reachable from the macOS VM; advertised to the job` or a
warning naming the unreachable address.

The all-or-nothing rule matters. Go falls back to `|direct` on an unreachable proxy. Cargo and rustup do not. A partial
env could fail a build that would have succeeded uncached. So any failure stages nothing, and the job runs exactly as
before, just over the open internet.

### The disks had to grow

The last one showed up as a hard failure. ePHPm's nightly `Build (PHP x, macos-aarch64)` builds PHP-linked debug test
binaries. Each one embeds libphp and ICU. That filled the job VM to 98-99% during the link step. It failed with `No
space left on device` every night from mid-October on.

Here's the math. The base image is 50 GB. Every job boots from a clone of it, so the guest sees about 41 GiB usable.
A debug build of that test suite doesn't fit.

[PR #216](https://github.com/ephpm/ephemerd/pull/216) added `disk_size_gb` under `[vm.macos]`. The default is 0, which
keeps today's behavior. When set, two things happen. On the host, before boot, a truncate extends the job's clone. The
clone is sparse on APFS, so no host disk is spent until the guest writes, and the blocks shared with the base stay
shared. In the guest, during the SSH setup step, two `diskutil` calls finish the job. `diskutil repairDisk` moves the GPT
backup header to the new end of the disk. `diskutil apfs resizeContainer` grows the container into the new space. The physical store isn't
hard-coded. It's read from `diskutil info` and validated before it goes anywhere near a command. Every failure is a
warning, and the job falls back to the base-size disk. On my Mac I set it to 100.

That PR came with a bonus bug. The Mac's jobs directory held a 38 GB clone from the start of October that was never
removed. A VM deleted its clones in exactly one place: at the end of `Stop`, after the Virtualization.framework stop
returned. When that stop hangs, the scheduler abandons the VM and the deletion never runs. A daemon crash mid-job
leaks the same files.

So #216 closed every path. `Stop` now runs the guest stop on its own goroutine and removes the files when it returns
or after 45 seconds, whichever comes first. Unlinking under a wedged guest is safe, because the guest keeps its open
descriptor. `Start` removes the clones on any error before the VM exists. A live-VM registry tracks VMs that are
provisioning or stuck stopping, which the running set doesn't cover. And a periodic sweep runs at startup and every
thirty minutes. It removes only job clones that no live VM owns and that haven't been written for two hours. It never
touches the base image. It reports `macos_clones_swept` and `macos_clone_bytes_swept` in status.

### Release status

All four of these are merged and shipped as I write this on 2026-10-23. The slot-leak bound landed in v0.2.5. The
deadline-path release landed in v0.2.11. The cache proxies for macOS jobs landed in v0.2.19. The bigger disks and the
clone sweep landed in v0.2.22, which is the current release.

One honest caveat. The darwin VM file needs cgo and Virtualization.framework, so it only compiles on a macOS builder.
The scheduler and slot logic are cross-platform and heavily tested. The guest-side `diskutil` resize is not. Neither is
the 45-second file removal under a truly wedged stop. Those two are covered by a canary job on the Mac and by fakes,
not by a unit test on every platform.

### What's next

The cache proxies cover gomod, cargo, GitHub releases and Composer. Homebrew, Xcode and the Actions downloads still go
over the open internet from the Mac. Those are the next slice of that 130 GB a day. I also want the slot observability
from #197 wired into mayfly so a pinned pool pages me instead of waiting for me to notice the graph.

### Links

- [ephemerd](https://github.com/ephpm/ephemerd)
- [PR #178: bound macOS VM provisioning so a wedged VM frees its slot](https://github.com/ephpm/ephemerd/pull/178)
- [PR #197: release the macOS slot on the provisioning-deadline path](https://github.com/ephpm/ephemerd/pull/197)
- [PR #211: give macOS VM jobs the cache proxies](https://github.com/ephpm/ephemerd/pull/211)
- [PR #216: bigger macOS job disks and a sweep for leaked clones](https://github.com/ephpm/ephemerd/pull/216)
- [ephemerd: One Daemon, a Fresh Runner Per Job, Every OS](/blog/2026/08/ephemerd-one-daemon-every-os/)
- [Kata Containers for CI: Boot Cost Isn't the Problem, File I/O Is](/blog/2026/09/kata-containers-for-ci/)
