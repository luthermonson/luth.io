---
title: "Self-Hosted Runner Part 3: Linux arm64 in a Hyper-V VM on the Dev Kit"
description: "Getting a second runner out of the Windows Dev Kit 2023: an Ubuntu arm64 VM on Hyper-V, sized, networked and registered."
date: 2026-04-23T20:30:00-07:00
tags: [ github-actions, self-hosted-runners, linux, arm64, hyper-v, windows, homelab ]
categories: [ ci, homelab, windows ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/04/self-hosted-runner-linux-arm64-hyper-v-network.svg
   alt: "Diagram comparing Hyper-V Default Switch NAT and an External switch for the Linux arm64 runner VM"
   stretch: "none"
draft: true
---

The Windows Dev Kit from [Part 2](/blog/2026/04/self-hosted-runner-windows-dev-kit-arm64/) has 8 cores and 32GB of RAM,
and a Windows arm64 runner doing one job at a time doesn't come close to using that. It's also running Windows 11 Pro, so
Hyper-V is available. That's the Linux arm64 runner for this
[series](/blog/2026/04/self-hosted-runners-every-arch/) without buying a Raspberry Pi or paying for an arm64 cloud VM:
an Ubuntu arm64 VM on the same box.

You could also get Linux arm64 from WSL2. I didn't, for one reason: WSL is a per-user feature that starts when a user
starts it, and I want the Linux runner to come back on its own after the Dev Kit reboots for updates with nobody logged in. A Hyper-V VM with an
automatic start action does that.

### Prerequisites

- The Dev Kit from Part 2 (or another Windows 11 Pro arm64 machine)
- An elevated PowerShell on it
- The Ubuntu Server 24.04 LTS **arm64** ISO. The arm64 images aren't on `releases.ubuntu.com` with the amd64 ones, they're
  under [cdimage.ubuntu.com](https://cdimage.ubuntu.com/releases/24.04/release/)

### Turn on Hyper-V

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Restart-Computer
```

That's the same command as on x64. After the reboot `Get-VMHost` should answer.

### Networking first

Decide this before you create the VM, because it's the one setting that's annoying to change later.

![Hyper-V Default Switch vs External switch](/img/2026/04/self-hosted-runner-linux-arm64-hyper-v-network.svg)

Hyper-V gives you a **Default Switch** out of the box. It's NAT with its own DHCP, the VM gets an address on a private
subnet behind the host, and outbound traffic just works. For a runner that's technically enough. A self-hosted runner only
ever makes outbound HTTPS connections to GitHub (it long-polls for jobs), so nothing has to reach the VM from outside.

The problems are all on my side:

- The Default Switch subnet isn't fixed, and it can move after the host reboots. Any note you made of the VM's IP is
  stale.
- Nothing on the LAN can reach the VM. I want to SSH in when a job fails, and I want the VM to be able to talk to things
  in the lab (a test Proxmox API, a local registry) the same way the other runners can.

So I use an **External** switch, which bridges the VM onto the Dev Kit's Ethernet port. The VM gets a lease from my
router like any other machine, and the host keeps its own network because of `-AllowManagementOS`:

```powershell
Get-NetAdapter    # find the wired adapter's name
New-VMSwitch -Name LAN -NetAdapterName "Ethernet" -AllowManagementOS $true
```

Expect the host to drop its network for a few seconds while the switch takes over the adapter. Don't do this over RDP
unless you like driving across town.

<!-- TODO(luther): confirm you went with an External switch (and the adapter name), or rewrite this section around the Default Switch if that's what you actually run. -->

### Create the VM

Arm64 has no legacy BIOS to emulate, so this is a Generation 2 (UEFI) VM. The other gotcha is Secure Boot: the default template is
`MicrosoftWindows`, which won't boot Ubuntu's shim. Switch it to the UEFI CA template, which is what Linux distros sign
against.

```powershell
$name = "devkit-linux"
$iso  = "C:\ISO\ubuntu-24.04.4-live-server-arm64.iso"

New-VM -Name $name -Generation 2 `
  -MemoryStartupBytes 12GB `
  -NewVHDPath "C:\VMs\$name.vhdx" -NewVHDSizeBytes 128GB `
  -SwitchName LAN

Set-VMProcessor -VMName $name -Count 4
Set-VMMemory    -VMName $name -DynamicMemoryEnabled $false
Set-VMFirmware  -VMName $name -SecureBootTemplate MicrosoftUEFICertificateAuthority

Add-VMDvdDrive  -VMName $name -Path $iso
Set-VMFirmware  -VMName $name -FirstBootDevice (Get-VMDvdDrive -VMName $name)

# come back after host reboots, shut down cleanly, no surprise checkpoints
Set-VM -Name $name -AutomaticStartAction Start -AutomaticStopAction ShutDown -CheckpointType Disabled

Start-VM -Name $name
vmconnect localhost $name
```

A few choices in there worth explaining:

- **4 of 8 cores, 12GB static.** That leaves the Windows runner on the host with half the CPU and well over half the
  memory. Two builds at once on this box will both be slower than one, and that's fine.
- **Dynamic memory off.** Linux builds allocate in bursts (a linker step can jump by gigabytes) and I'd rather the VM
  just own its memory than have Hyper-V balloon it at the wrong moment.
- **128GB disk.** The VHDX is dynamically expanding, so it only uses what's written. Docker images and Go build caches
  fill it faster than you'd think.
- **Checkpoints off.** Production checkpoints are the default and I don't want them piling up on a 512GB drive.

<!-- TODO(luther): replace the sizing above with what you actually gave the VM if it's different. -->

Run through the Ubuntu installer in `vmconnect`. Pick OpenSSH server when it asks, and give it a regular user. The Hyper-V
guest drivers are in Ubuntu's stock kernel, so there's nothing extra to install. After the reboot, eject the ISO:

```powershell
Get-VMDvdDrive -VMName $name | Set-VMDvdDrive -Path $null
```

From then on I don't use the console, I SSH in from my workstation.

### The runner inside the VM

This is just a Linux arm64 runner install, and it's the easiest one in the series. On the VM:

```sh
sudo useradd -m -s /bin/bash runner
sudo -iu runner

mkdir actions-runner && cd actions-runner
curl -O -L https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-linux-arm64-2.333.1.tar.gz
echo "69ac7e5692f877189e7dddf4a1bb16cbbd6425568cd69a0359895fac48b9ad3b  actions-runner-linux-arm64-2.333.1.tar.gz" | sha256sum -c
tar xzf ./actions-runner-linux-arm64-2.333.1.tar.gz
```

The runner tarball includes a script that installs the .NET runtime dependencies for your distro. Run it once as root:

```sh
sudo ./bin/installdependencies.sh
```

Then register with a token from `gh api -X POST orgs/ephpm/actions/runners/registration-token --jq .token`:

```sh
./config.sh --unattended \
  --url https://github.com/ephpm \
  --token <token> \
  --name devkit-linux
```

Default labels: `self-hosted`, `Linux`, `ARM64`.

On Linux `svc.sh` writes a systemd unit. It has to be run with `sudo`, and it takes the user the service should run as:

```sh
exit                                  # back to your admin user
cd /home/runner/actions-runner
sudo ./svc.sh install runner
sudo ./svc.sh start
systemctl status 'actions.runner.*'
```

Unlike macOS there's no login session involved. systemd starts it at boot, Hyper-V starts the VM when the host boots, and
the runner shows up as idle in the org settings a minute or so after the Dev Kit comes back from an update.

<!-- TODO(luther): confirm the runner name. -->

### Docker in the VM

A lot of Linux jobs want Docker, and here that's just Docker on an arm64 Ubuntu box:

```sh
sudo apt-get install -y docker.io
sudo usermod -aG docker runner
sudo ./svc.sh stop && sudo ./svc.sh start
```

Be honest with yourself about that `usermod` line. Membership in the `docker` group is root on the VM, so any job that
runs here can do what it likes to it. The VM boundary is the only thing between that and the Windows host. That's a big
part of why this is a VM and not WSL.

What you won't get is nested virtualization for free. Check before you plan on running VMs inside the runner:

```sh
ls -l /dev/kvm
```

<!-- TODO(luther): paste what that printed on the Dev Kit VM. If /dev/kvm isn't there, say so and drop the "for free" phrasing. -->

### What it's good for

This runner is where arm64 Linux builds of my Go projects actually *run*, which catches the stuff `GOARCH=arm64 go build`
on an x86 box can't: cgo code with alignment assumptions, `unsafe` tricks that only hold on amd64, and test fixtures that
somebody generated on x86 and checked in. It's also native arm64 Docker, so `docker build` produces an arm64 image without
QEMU user emulation in the loop.

Next the series moves to x86_64, starting with the box that's already in most home labs:
[a Linux runner on Proxmox](/blog/2026/04/self-hosted-runner-proxmox-linux/).

### Links

- [Ubuntu 24.04 arm64 server images](https://cdimage.ubuntu.com/releases/24.04/release/)
- [actions/runner v2.333.1](https://github.com/actions/runner/releases/tag/v2.333.1)
- [Create a virtual switch for Hyper-V](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-switch-for-hyper-v-virtual-machines)
