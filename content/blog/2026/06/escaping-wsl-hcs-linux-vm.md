---
title: "Escaping WSL: Booting My Own Linux VM Through Hyper-V's HCS API"
description: "WSL2 refuses to run under LocalSystem, so ephemerd boots an Alpine kernel directly through vmcompute.dll instead."
date: 2026-06-14T20:12:00-07:00
tags: [ go, windows, hyper-v, hcs, linux, ephemerd, github-actions ]
categories: [ windows, ci, go ]
series: [ "ephemerd" ]
images:
 - src: img/2026/06/escaping-wsl-hcs-linux-vm-boot.svg
   alt: "Diagram of ephemerd booting a Linux VM through HCS and reading its IP from the serial console"
   stretch: "none"
draft: false
---

I've been building [ephemerd](https://github.com/ephpm/ephemerd), a daemon that runs self-hosted GitHub Actions runners
with a fresh environment per job. On a Windows host, Windows jobs run in Hyper-V isolated containers, but Linux jobs still
need a Linux kernel from somewhere. The obvious answer on Windows in 2026 is WSL2, and that's what I used first. It worked
great from my terminal and fell over the moment ephemerd became a real Windows service.

### WSL2 doesn't do LocalSystem

A Windows service that wants to create containers and Hyper-V VMs runs as `LocalSystem`. WSL2 doesn't support that context
at all. You get `WSL_E_LOCAL_SYSTEM_NOT_SUPPORTED` and that's the end of the conversation. There's no flag for it, WSL is a
per-user feature and a service account isn't a user.

So the options were: run the service as a real user account (and inherit every weird profile-loading problem that comes
with it), or skip WSL and talk to Hyper-V directly. I went with Hyper-V. The commit that did it is
[`fa41077`](https://github.com/ephpm/ephemerd/commit/fa41077), "replace WSL with Hyper-V for Linux VM on Windows", and the
rest of this post is what that took.

![HCS boot flow](/img/2026/06/escaping-wsl-hcs-linux-vm-boot.svg)

### HCS is the API under all of it

Docker, containerd's Windows shim and WSL itself all end up in the Host Compute Service, exposed through `vmcompute.dll`.
Microsoft's Go library for it is [hcsshim](https://github.com/microsoft/hcsshim), and hcsshim already knows how to boot a
Linux VM. That's what LCOW (Linux Containers on Windows) does: `internal/uvm.makeLCOWDoc` builds a JSON document describing a
"utility VM" with a kernel, an initrd and some devices, and hands it to HCS.

The catch is the word `internal`. Go won't let you import another module's internal packages, so the code I wanted to reuse
was off limits. What I could do is call `vmcompute.dll` directly, which is exactly what hcsshim does under the covers, and
define my own copies of the schema structs:

```go
var (
	modvmcompute = windows.NewLazySystemDLL("vmcompute.dll")

	procHcsCreateComputeSystem     = modvmcompute.NewProc("HcsCreateComputeSystem")
	procHcsStartComputeSystem      = modvmcompute.NewProc("HcsStartComputeSystem")
	procHcsTerminateComputeSystem  = modvmcompute.NewProc("HcsTerminateComputeSystem")
	procHcsEnumerateComputeSystems = modvmcompute.NewProc("HcsEnumerateComputeSystems")
	// ...
)

type hcsLinuxKernelDirect struct {
	KernelFilePath string `json:"KernelFilePath,omitempty"`
	InitRdPath     string `json:"InitRdPath,omitempty"`
	KernelCmdLine  string `json:"KernelCmdLine,omitempty"`
}
```

The whole VM is one document. Schema 2.1, a `LinuxKernelDirect` chipset so there's no firmware or bootloader, a SCSI
controller with the root disk, a network adapter bound to an HCN endpoint on the Default Switch, and a COM port mapped to a
named pipe. Trimmed down from `pkg/vm/linuxvm_windows.go`:

```go
doc := &hcsComputeSystem{
	Owner:                             "ephemerd",
	SchemaVersion:                     &hcsVersion{Major: 2, Minor: 1},
	ShouldTerminateOnLastHandleClosed: true,
	VirtualMachine: &hcsVM{
		StopOnReset: true,
		Chipset: &hcsChipset{
			LinuxKernelDirect: &hcsLinuxKernelDirect{
				KernelFilePath: kernelPath,
				InitRdPath:     initrdPath,
				KernelCmdLine:  cmdline,
			},
		},
		ComputeTopology: &hcsTopology{
			Memory:    &hcsMemory{SizeInMB: l.cfg.MemoryMB, AllowOvercommit: true},
			Processor: &hcsProcessor{Count: uint32(l.cfg.CPUs)},
		},
		Devices: &hcsDevices{
			Scsi: map[string]hcsScsi{
				scsiControllerGUIDs[0]: {Attachments: map[string]hcsAttachment{
					"0": {Type_: "VirtualDisk", Path: l.rootVHDXPath()},
				}},
			},
			NetworkAdapters: map[string]hcsNetworkAdapter{"default": {EndpointId: endpointID}},
			ComPorts:        map[string]hcsComPort{"0": {NamedPipe: consolePipe}},
		},
	},
}
```

The SCSI controller GUID isn't arbitrary either. hcsshim derives its controller IDs as v5 UUIDs from a fixed namespace, and
I copied the first one (`df6d0690-79e5-55b6-a5ec-c1e2f77f580a`) rather than inventing one and finding out what HCS thought
of it.

Calling it is a plain `syscall.SyscallN` with UTF-16 strings. The one thing that tripped me up is that create and start are
asynchronous. A return of `HCS_E_OPERATION_PENDING` (`0xC0370103`) isn't an error, it means the handle is good and HCS is
still working:

```go
r0, _, _ := syscall.SyscallN(
	procHcsCreateComputeSystem.Addr(),
	uintptr(unsafe.Pointer(idPtr)),
	uintptr(unsafe.Pointer(configPtr)),
	0, // identity
	uintptr(unsafe.Pointer(&handle)),
	uintptr(unsafe.Pointer(&resultPtr)),
)
if uint32(r0) == hcsOperationPending {
	hcsResultString(resultPtr) // free result
	time.Sleep(3 * time.Second)
	return handle, nil
}
```

### The kernel has to be an uncompressed ELF

`LinuxKernelDirect` loads the kernel itself, so it wants a raw `vmlinux`, not the `bzImage` every distro ships. I use
Alpine's `linux-virt` package (6.12 series at the time), and the build step pulls the gzip payload out of the bzImage by
scanning for gzip magic bytes and keeping the first stream that decompresses to something starting with `\x7fELF`:

```go
magic := []byte{0x1f, 0x8b, 0x08}
for i := 0; i <= len(data)-3; i++ {
	if data[i] != magic[0] || data[i+1] != magic[1] || data[i+2] != magic[2] {
		continue
	}
	gr, gzErr := gzip.NewReader(bytes.NewReader(data[i:]))
	if gzErr != nil {
		continue
	}
	gr.Multistream(false)
	raw, readErr := io.ReadAll(gr)
	_ = gr.Close()
	if readErr != nil || len(raw) < 64 {
		continue
	}
	if raw[0] == 0x7f && raw[1] == 'E' && raw[2] == 'L' && raw[3] == 'F' {
		return os.WriteFile(dest, raw, 0o644)
	}
}
```

It's the same trick as the kernel's own `extract-vmlinux` script, just without needing a Linux box at build time.

The kernel command line had one trap worth writing down. My first attempt said `root=/dev/sda init=/init`, which tells the
kernel to mount the (still unformatted) disk and look for init there. What I wanted is `rdinit=/init`, run init from the
initrd, and let the init script format and mount the disk itself:

```go
cmdline := fmt.Sprintf(
	"rdinit=/init ephemerd.containerd_port=%d ephemerd.root_disk=/dev/sda%s "+
		"pci=off brd.rd_nr=0 pmtmr=0 nr_cpus=%d "+
		"8250_core.nr_uarts=1 8250_core.skip_txen_test=1 console=ttyS0,115200",
	l.cfg.ContainerdPort, dindFlag, l.cfg.CPUs,
)
```

### Finding the VM's IP through the serial console

Once the VM is up I need its IP address to dial containerd inside it. There's no guest agent, and asking the Default Switch
for DHCP leases from a service isn't something I wanted to depend on. The VM already has a serial port wired to a named
pipe, though, so the init script just prints its address:

```sh
echo "EPHEMERD_IP=$MY_IP"
echo "ephemerd-init: gateway=$GW"
```

and the host reads the pipe, writes every line to a `console.log`, and watches for the marker:

```go
scanner := bufio.NewScanner(pipe)
for scanner.Scan() {
	line := scanner.Text()
	fmt.Fprintf(logFile, "%s\n", line)
	if strings.HasPrefix(line, "EPHEMERD_IP=") {
		if ip := strings.TrimSpace(strings.TrimPrefix(line, "EPHEMERD_IP=")); ip != "" {
			select {
			case l.ipCh <- ip:
			default:
			}
		}
	}
}
```

Crude, but it has a nice side effect. Every kernel panic and init failure lands in `console.log`, which is the first place
I look when the VM doesn't come up. From there it's a TCP poll until containerd and the dispatch gRPC server answer. The
first working version booted in about 20 seconds from service start to containerd ready.

### The things HCS didn't tell me

A handful of failures that cost me real time, in case you go down this road too:

- **The compute system ID has to be a GUID.** HCS uses the ID as the identity for `GrantVmAccess`, and a human-readable
  name gets rejected. ephemerd generates a v4 UUID for the ID and keeps a friendly name for logs.
- **File ACLs.** The VM worker process opens the VHDX as a virtual account. Without an explicit grant on the file, create
  succeeds and start fails with `HCS_E_SYSTEM_ALREADY_STOPPED`, which reads like a lifecycle bug and is really a permissions
  bug ([`98d786c`](https://github.com/ephpm/ephemerd/commit/98d786c)). ephemerd runs `icacls` on the VHDX before every boot.
- **Clock drift.** Without the `hv_utils` module the guest has no Hyper-V TimeSync, the clock wanders, and the GitHub
  runner's JWTs start getting rejected as expired. That one didn't look like a VM problem at all
  ([`317b92a`](https://github.com/ephpm/ephemerd/commit/317b92a)).
- **Stale VMs.** If the service dies hard the VM can outlive it. On startup ephemerd enumerates compute systems owned by
  `ephemerd` and terminates them before booting a new one.

### Stop baking the binary into the initrd

The first design was a "fat initrd": busybox, the rootfs and the cross-compiled Linux build of ephemerd all in one cpio at
build time. It was about 190 MB, and it had a nasty failure mode. A plain `go build` of `ephemerd.exe` didn't regenerate the
initrd, so I shipped several rounds of fixes that were running yesterday's Linux code inside the VM before I noticed.

The fix leans on a kernel feature I'd forgotten about: you can concatenate cpio archives and the kernel unpacks them in
order into one initramfs, with later entries overriding earlier ones. So the Linux binary moved into its own `go:embed`, and
on every boot ephemerd appends a tiny gzipped cpio tail to the base initrd
([`fb96345`](https://github.com/ephpm/ephemerd/commit/fb96345)). The build-time initrd dropped from ~190 MB to ~7.6 MB and
the VM always runs the binary that's actually on disk.

That same tail turned out to be the right way to get config into the VM. Every new setting the VM needed was a struct
field, a kernel command line flag, a parser in the init script and a rebuild. [#89](https://github.com/ephpm/ephemerd/pull/89)
puts the host's `config.toml` into the tail with mode `0600`:

```go
if err := writeCPIOEntry(gw, "assets/ephemerd-linux", 0o100755, binData, ""); err != nil {
	return fmt.Errorf("cpio: ephemerd-linux: %w", err)
}
if cfgData != nil {
	if err := writeCPIOEntry(gw, "assets/config.toml", 0o100600, cfgData, ""); err != nil {
		return fmt.Errorf("cpio: config.toml: %w", err)
	}
}
```

The first draft of that feature used a Hyper-V Plan9 share instead, and it failed twice. Starting the VM failed
with `HRESULT 0xc0370110`, `HCS_E_SYSTEM_ALREADY_STOPPED` again, because HCS couldn't start the Plan9 device document I
built, and that took Linux CI on my dev rig down for about 100 minutes. And even with the device accepted,
Hyper-V serves Plan9 over hvsock, not virtio, and mainline `mount -t 9p` has no hvsock transport. LCOW gets around that with
its own guest agent passing an `AF_VSOCK` fd to `trans=fd`. That's a lot of machinery for one file read once at boot. The
details are in [`docs/arch/host-config-initrd.md`](https://github.com/ephpm/ephemerd/blob/main/docs/arch/host-config-initrd.md).

### Running KIND inside it

The real test of a homemade VM is whether someone else's software runs in it. One of my repos runs KIND clusters in CI, which
means kube-proxy inside a `kindest/node` container inside a job container inside this VM. It didn't work until two kernel
modules went into the initrd:

- `br_netfilter`, plus `bridge-nf-call-iptables=1`. Without it, pod traffic crossing the CNI bridge skips iptables, CoreDNS
  can't reach `10.96.0.1:443`, and everything downstream times out
  ([`fbc1e68`](https://github.com/ephpm/ephemerd/commit/fbc1e68)).
- `ipt_REJECT` (with `nf_reject_ipv4` under it). kube-proxy's first rule is a REJECT for services with no endpoints. Without the module `iptables-restore`
  fails on that line, aborts the whole batch, and `KUBE-SERVICES` stays empty
  ([`88ebc24`](https://github.com/ephpm/ephemerd/commit/88ebc24)).

Neither error mentions a missing module unless you go and read kube-proxy's own logs from inside the node container.

### What's next

The boot is still three handoffs: initrd, then a pivot into an Alpine rootfs, then ephemerd. There's a proposal in the repo
([`docs/arch/quickboot-image.md`](https://github.com/ephpm/ephemerd/blob/main/docs/arch/quickboot-image.md)) to collapse that
into a custom kernel plus an initramfs where ephemerd itself is PID 1. It's a proposal and nothing more yet, but after this
round I'm a lot less scared of owning the whole boot path.

### Links

- [ephemerd](https://github.com/ephpm/ephemerd)
- [`fa41077` replace WSL with Hyper-V](https://github.com/ephpm/ephemerd/commit/fa41077)
- [#89 host config via the boot initrd tail](https://github.com/ephpm/ephemerd/pull/89)
- [microsoft/hcsshim](https://github.com/microsoft/hcsshim)
- [Host Compute Service overview](https://learn.microsoft.com/en-us/virtualization/api/hcs/overview)
