---
title: "Kata Containers for CI: Boot Cost Isn't the Problem, File I/O Is"
description: "Kata containers are safer for CI, but slower. Starting one takes 12x longer, which is fine, and reading small files is about 40x slower for the entire job, which isn't."
date: 2026-09-03T21:35:00-07:00
tags: [ kata-containers, containerd, linux, kvm, virtualization, ci, performance, ephemerd ]
categories: [ containers, ci ]
series: [ "ephemerd" ]
images:
 - src: img/2026/08/kata-containers-for-ci-ratio.svg
   alt: "Bar chart of Kata slowdown versus runc for each measurement"
   stretch: "none"
draft: false
---

Of the three platforms [ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) runs jobs on, Linux has the weakest
isolation. Windows jobs get a Hyper-V isolated container with its own kernel, macOS jobs get a whole VM, and a Linux job is
an ordinary runc container sharing the host kernel. A kernel escape from a Linux job is a host compromise.
[Kata Containers](https://katacontainers.io/) fixes that by putting each container in a lightweight VM with its own kernel,
and it plugs into containerd as just another runtime handler. So I added it in
[#155](https://github.com/ephpm/ephemerd/pull/155), measured it, and left it off by default. This post is why.

### Wiring it in

From containerd's point of view Kata is a different shim, `io.containerd.kata.v2`, so the change to the runtime is small. The
job-container creation site used to hardcode runc and now resolves the handler from config:

```go
func resolveRuntimeName(linuxRuntime, goos string) string {
	if goos == "windows" {
		return "io.containerd.runhcs.v1"
	}
	if linuxRuntime == "" {
		return "io.containerd.runc.v2"
	}
	return linuxRuntime
}
```

```toml
[runner.linux]
runtime = "kata"   # "runc" (default) or "kata"
```

The more interesting decision was what happens when Kata is configured but can't run. ephemerd checks for
`containerd-shim-kata-v2` on the `PATH` and that `/dev/kvm` exists *and opens*, and if either fails, the daemon refuses to
start:

```go
f, err := os.OpenFile("/dev/kvm", os.O_RDWR, 0)
if err != nil {
	hint := "this host has no usable KVM device"
	if os.IsNotExist(err) {
		hint = "/dev/kvm does not exist — on a VM this usually means nested " +
			"virtualization is not enabled for this guest"
	} else if os.IsPermission(err) {
		hint = "/dev/kvm exists but is not openable — check that ephemerd runs " +
			"as root or is in the kvm group"
	}
	return fmt.Errorf(`runner.linux.runtime = "kata" but %s: %w`, hint, err)
}
```

The tempting alternative is falling back to runc with a warning. But then the config says "VM isolation" while untrusted CI
runs on the host kernel, and nobody reads warnings. An outage is a better failure than a silent downgrade of an isolation
guarantee.

### The numbers

Measured on one of my amd64 fleet nodes: an 8-core Proxmox guest with nested virtualization, Kata 4.0.0 with QEMU, the same
runner image, runc and Kata runs interleaved so neither got a quieter period. The full tables are in
[#155](https://github.com/ephpm/ephemerd/pull/155) and the
[configuration docs](https://github.com/ephpm/ephemerd/blob/main/docs/getting-started/configuration.md); these charts are
straight from them.

| Measurement | runc | Kata | Ratio |
|---|---|---|---|
| Provision a job container, dind on (median, n=9) | 203 ms | 2,498 ms | 12.3x |
| Create to running, dind off (median, n=18) | 140 ms | 4,146 ms | 30x |
| Host RSS per idle container | ~13.6 MB | ~313 MB | 23x |
| CPU-bound compile (median, n=4) | 7.5 s | 10.2 s | 1.35x |
| Create 3,000 files on the rootfs (median, n=4) | 645 ms | 5,195 ms | 8.1x |
| Read 3,000 files on the rootfs | 79 ms | 2,890 ms | 37x |
| Create 3,000 files on the bind-mounted runner dir | 435 ms | 4,712 ms | 10.8x |
| Read 3,000 files on the bind-mounted runner dir | 73 ms | 2,984 ms | 41x |

![Kata slowdown by measurement](/img/2026/08/kata-containers-for-ci-ratio.svg)

The first thing everyone asks about with VM-per-container is boot time, and yes, it's there. A job container with dind
enabled goes from about 200 ms to about 2.5 seconds, nearly all of it guest kernel boot during `task create`:

![Provisioning time](/img/2026/08/kata-containers-for-ci-start.svg)

But I don't care much about 2.3 seconds per job. A CI job runs for minutes. The boot is paid once and it's done.

### File I/O is the real cost

The file numbers are the ones that matter, because they're paid for the entire life of the job. Both the guest's root
filesystem and the bind-mounted runner directory reach the host through virtio-fs, so every `open()` and `stat()` crosses the
VM boundary. Reading 3,000 small files goes from 79 ms to 2.9 seconds:

![File I/O](/img/2026/08/kata-containers-for-ci-fileio.svg)

And small-file I/O is most of what CI does. `actions/checkout`, `npm ci`, `go mod download`, a `cargo build` walking its
target directory, tests that glob fixtures. A CPU-bound compile only lost 35%, which would be an easy trade. A job that
installs dependencies and runs a test suite is going to feel the 8-41x on every step.

Memory is the other cost. About 310 MB of host RSS per idle job, most of it the QEMU process (~267 MB resident) versus
~14 MB for runc. On a node running four jobs at a time, that's about 1.2 GB of headroom it didn't need before.

One caveat from the PR worth repeating: the node kept serving production CI while I measured. The workload and memory runs
had stable load. The dind-off start-latency runs saw load climb, so the 4.1 s median there overstates an idle node, which is
part of why the dind-on number (2.5 s, tighter spread of 2,472 to 2,528 ms) is the one I quote for start cost.

### Swapping the hypervisor doesn't fix it

Kata ships Cloud Hypervisor and Firecracker alongside QEMU, selectable by symlinking a different shim
(`containerd-shim-kata-clh-v2`, `containerd-shim-kata-fc-v2`). I ran all three in the same session:

![Hypervisor comparison](/img/2026/08/kata-containers-for-ci-hv.svg)

Firecracker came in about 9% under QEMU, Cloud Hypervisor about 8% over. (The repo docs summarize this as "neither beat
QEMU", which isn't quite what the Firecracker number says.) Either way, every one of them is in the five to six second range
against runc's sub-200 ms (that session ran slower across the board than the runs in the table above, QEMU included).
I only compared start latency across them. I didn't re-run the file tests under the other VMMs; with QEMU the file
I/O cost comes from virtio-fs, and Firecracker's Kata config uses a block-device rootfs instead, so that comparison is
still open. There's no cheap start-time win here from switching VMMs on stock configs.

### It's not just slower, it's different

A few behavior changes showed up that matter as much as the numbers:

- **arm64 can't run it at all on my fleet.** The arm64 nodes have no `/dev/kvm`, so a fleet-wide default was never on the
  table.
- **AppArmor doesn't apply inside the guest.** There's no AppArmor in the Kata kernel, so the profile ephemerd puts on job
  containers is gone. You trade in-guest defense-in-depth for the VM boundary.
- **cgroup memory limits don't translate.** The guest reported 2.3 GB `MemTotal` regardless of a 512 MB limit.
- **The Docker socket can't cross the boundary.** ephemerd normally bind-mounts a host unix socket at `/var/run/docker.sock`.
  Under Kata the bind becomes a virtio-fs passthrough that carries the socket *inode* into the guest but not its
  connectability, so the socket is there and every `connect()` gets `ECONNREFUSED`.
- **Sibling `-v` mounts from the job's own filesystem don't work.** `docker run -v $GITHUB_WORKSPACE:/x` from inside a Kata
  job used to produce an empty directory with no error. Now it's an explicit 400.

The Docker socket one had a fix sitting right there. Hyper-V isolated Windows containers are VMs with exactly the same
constraint, and Windows jobs have always gotten the Docker API over TCP on the bridge gateway with `DOCKER_HOST=tcp://...`.
The transport had been chosen by build tag, which was really standing in for "does this container share the host kernel".
Now it's chosen at runtime: unix socket for runc, TCP for Kata, TCP always on Windows.

Moving it to TCP changes the security story, though. The Docker API authenticates nothing, and on Linux every container can
address the bridge gateway. So the firewall is now the only thing deciding who can drive which job's Docker daemon. Each
job's port gets an `ACCEPT` for that one container's `/32` followed by a `DROP` for the port from everyone else, in an
`EPHEMERD-DIND` chain jumped from both `INPUT` and `EPHEMERD-FORWARD`. The isolation was checked with a negative control:
the probe from a neighbour container times out, deleting the `DROP` makes it connect, restoring it blocks it again. That
`INPUT` detail also led me to a bigger hole, which I wrote up in
[My iptables Rule Matched 0 Packets in 9.8 GB](/blog/2026/08/iptables-rule-zero-packets/).

### So, default or not?

Not, for now. Kata is the right answer for a node that runs untrusted public-fork PRs and can afford the I/O hit, and it's a
one-line config change for those nodes. For my own repos on my own hardware, a 37x slower checkout for a kernel-escape class
that AppArmor, dropped capabilities and the egress firewall already narrow down isn't worth it.

If I come back to this, it'll be for the file I/O, not the boot. Getting the job's working directory off virtio-fs and onto
something guest-local is where I'd start looking.

### How these were measured

I didn't re-run these for this post. Kata isn't installed on my workstation and I wasn't going to fake a nested-virt setup
to get numbers that would be less representative than the originals. Everything above comes from the measurements recorded
with the PR: an 8-core amd64 Proxmox guest, Kata 4.0.0, QEMU, containerd handler `io.containerd.kata.v2`, runc and Kata
interleaved against the same image on the same node, sample sizes as listed in the table.

### Links

- [#155 opt-in Kata Containers runtime](https://github.com/ephpm/ephemerd/pull/155)
- [ephemerd configuration docs, `[runner.linux]`](https://github.com/ephpm/ephemerd/blob/main/docs/getting-started/configuration.md)
- [Kata Containers](https://katacontainers.io/)
- [virtio-fs](https://virtio-fs.gitlab.io/)
