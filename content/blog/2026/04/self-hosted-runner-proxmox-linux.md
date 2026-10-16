---
title: "Self-Hosted Runner Part 4: Linux x86_64 on Proxmox"
description: "The easiest runner in the series: a Proxmox VM from an Ubuntu cloud image, scripted with go-proxmox, and what a runner that never gets wiped does to your builds."
date: 2026-04-25T16:10:00-07:00
tags: [ github-actions, self-hosted-runners, linux, proxmox, go, go-proxmox, cloud-init, homelab ]
categories: [ ci, homelab ]
series: [ "Self-Hosted Runners on Every Arch" ]
images:
 - src: img/2026/04/self-hosted-runner-proxmox-linux-state.svg
   alt: "Diagram of a runner VM on Proxmox showing which files and state survive between jobs"
   stretch: "none"
draft: true
---

If you have a home lab you probably already have the hardware for this one. Mine is the
[Proxmox VE](https://www.proxmox.com/en/) host I've been [building Go tooling around](/blog/2022/02/recreating-proxmox-ve-xtermjs-in-go/)
for a few years, and it was the first runner I set up for this
[series](/blog/2026/04/self-hosted-runners-every-arch/). It's the one with real history behind it too: on April 1st I
moved php-sdk's Linux x86_64 build from `ubuntu-latest` to `[self-hosted, linux, x64]`
([`9f809bb`](https://github.com/ephpm/php-sdk/commit/9f809bb)), and the runner that picked it up was a VM on this box
called `linux-github-runner`.

### VM or LXC

Proxmox gives you two choices, a QEMU VM or an LXC container. LXC is lighter, boots in a second and shares the host's
memory more efficiently, and for a lot of home lab services it's what I'd pick. For a CI runner I use a VM:

- **Docker.** Linux jobs want Docker. Docker inside an unprivileged LXC needs the `nesting` and `keyctl` features turned
  on, and you will still run into the odd storage driver or cgroup issue. In a VM it's just Docker.
- **Kernel isolation.** An LXC container shares the Proxmox host's kernel. A runner executes whatever a workflow says,
  and I'd rather a bad job be stuck in a guest kernel than one step away from the hypervisor.
- **Same shape as the others.** The Windows runner in [Part 5](/blog/2026/04/self-hosted-runner-proxmox-windows/) has to
  be a VM anyway. Keeping both as VMs means one template workflow and one way to back them up.

### Prerequisites

- A Proxmox VE node with some spare cores and memory
- An API token for the scripted version (Datacenter → Permissions → API Tokens)
- A GitHub org or repo you can register runners against, and [gh](https://cli.github.com/)

### A template from the Ubuntu cloud image

Ubuntu publishes cloud images that boot straight into cloud-init, which beats clicking through an installer. Once on the
Proxmox host:

```sh
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

qm create 9000 --name ubuntu-2404-runner --memory 8192 --cores 4 --cpu host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single --agent enabled=1 \
  --serial0 socket --vga serial0
qm set 9000 --scsi0 local-lvm:0,import-from=/root/noble-server-cloudimg-amd64.img
qm set 9000 --boot order=scsi0
qm disk resize 9000 scsi0 64G
qm template 9000
```

`--cpu host` matters for a build box. The default CPU model hides instruction set extensions from the guest, and
compilers that tune for the machine they run on (or tests that check for AVX2) see a lowest-common-denominator CPU.
`--agent enabled=1` turns on the QEMU guest agent channel, which the scripted version below waits on.

<!-- TODO(luther): swap in the VMID, storage name, bridge and sizing you actually use for linux-github-runner. -->

### Cloning it with go-proxmox

You can clone the template in the UI, paste a cloud-init config in, and be done. I script it with
[go-proxmox](https://github.com/luthermonson/go-proxmox), because then making a second runner is a command, not an
afternoon. This is against v0.4.1, which I tagged on April 2nd:

```go
package main

import (
	"context"
	"crypto/tls"
	"net/http"
	"os"
	"strings"

	"github.com/luthermonson/go-proxmox"
)

func main() {
	ctx := context.Background()

	// my PVE has a self-signed cert; use a real CA bundle if yours doesn't
	insecure := &http.Client{Transport: &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}}
	client := proxmox.NewClient(os.Getenv("PVE_URL"),
		proxmox.WithHTTPClient(insecure),
		proxmox.WithAPIToken(os.Getenv("PVE_TOKEN_ID"), os.Getenv("PVE_SECRET")),
	)

	node, err := client.Node(ctx, "pve1")
	check(err)
	tmpl, err := node.VirtualMachine(ctx, 9000)
	check(err)

	newid, task, err := tmpl.Clone(ctx, &proxmox.VirtualMachineCloneOptions{
		Name: "linux-github-runner",
		Full: 1, // full clone, the runner shouldn't depend on the template's disk
	})
	check(err)
	check(task.WaitFor(ctx, 300))

	vm, err := node.VirtualMachine(ctx, newid)
	check(err)

	raw, err := os.ReadFile("runner-user-data.yaml")
	check(err)
	// RUNNER_TOKEN comes from `gh api -X POST .../registration-token`
	userdata := strings.ReplaceAll(string(raw), "REGISTRATION_TOKEN", os.Getenv("RUNNER_TOKEN"))
	metadata := "instance-id: linux-github-runner\nlocal-hostname: linux-github-runner\n"

	// builds a NoCloud ISO, uploads it to the node's ISO storage and attaches it
	check(vm.CloudInit(ctx, "ide2", userdata, metadata, "", ""))

	task, err = vm.Start(ctx)
	check(err)
	check(task.WaitFor(ctx, 60))

	// cloud-init installs the guest agent, so this returns once first boot is done enough
	check(vm.WaitForAgent(ctx, 600))
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}
```

`CloudInit` is the part that saved me the most clicking when I wrote it. It builds the `cidata` ISO locally, uploads it
to the node and wires it in as a CD-ROM, so the VM finds its config on first boot the same way it would on a cloud.

### The cloud-init that installs the runner

The user-data does the install from the earlier posts, unattended. The registration token comes from
`gh api -X POST orgs/ephpm/actions/runners/registration-token --jq .token` right before you clone:

```yaml
#cloud-config
users:
  - default
  - name: runner
    shell: /bin/bash
    sudo: "ALL=(ALL) NOPASSWD:ALL"
packages:
  - qemu-guest-agent
  - docker.io
  - git
  - build-essential
runcmd:
  - systemctl enable --now qemu-guest-agent
  - usermod -aG docker runner
  - |
    su - runner -c '
      set -e
      mkdir -p actions-runner && cd actions-runner
      curl -fsSL -o runner.tar.gz https://github.com/actions/runner/releases/download/v2.333.1/actions-runner-linux-x64-2.333.1.tar.gz
      echo "18f8f68ed1892854ff2ab1bab4fcaa2f5abeedc98093b6cb13638991725cab74  runner.tar.gz" | sha256sum -c
      tar xzf runner.tar.gz
    '
  - /home/runner/actions-runner/bin/installdependencies.sh
  - su - runner -c 'cd actions-runner && ./config.sh --unattended --url https://github.com/ephpm --token REGISTRATION_TOKEN --name linux-github-runner'
  - cd /home/runner/actions-runner && ./svc.sh install runner && ./svc.sh start
```

Two notes on that. The passwordless `sudo` is there because jobs written for hosted runners call `sudo apt-get install`
freely, and php-sdk's Linux job is one of them. And the registration token ends up in a file on the Proxmox ISO storage.
It's only good for registering a runner and it expires in an hour, but delete the ISO after first boot anyway.

Default labels: `self-hosted`, `Linux`, `X64`.

### The runner that never forgets

![What survives between jobs on a plain runner VM](/img/2026/04/self-hosted-runner-proxmox-linux-state.svg)

This is the part nobody tells you when they say "just run your own runner." A hosted runner is a fresh VM per job. Mine is
the same VM, job after job, and the workflow has to deal with that. php-sdk taught me this inside of an hour.

The first thing that broke was writing to `/usr/local/bin`. On a hosted runner that works. On mine the runner user doesn't
own it, so the `spc` download moved into the runner's home ([`183619f`](https://github.com/ephpm/php-sdk/commit/183619f)):

```yaml
- name: Download spc
  run: |
    mkdir -p "$HOME/bin"
    curl -fsSL "https://github.com/crazywhalecc/static-php-cli/releases/download/${{ env.SPC_VERSION }}/spc-linux-${{ matrix.spc_arch }}.tar.gz" \
      | tar xz -C "$HOME/bin"
    echo "$HOME/bin" >> "$GITHUB_PATH"
```

The second thing was leftovers. The php-sdk build doesn't check out its own repo, it downloads PHP sources and builds
them in the workspace. With no `actions/checkout`, nothing ever runs `git clean`, so the previous run's `buildroot/`,
`source/` and `downloads/` were still sitting there. The fix was a cleanup step that only runs on my runners
([`662d8d0`](https://github.com/ephpm/php-sdk/commit/662d8d0)):

```yaml
- name: Clean up previous builds
  if: ${{ !contains(matrix.runner, 'ubuntu') }}
  run: rm -rf buildroot source downloads sdk *.tar.gz
```

That works, and it's also the wrong layer. Every workflow that targets the runner has to remember to clean up after the
last one. The runner has a better hook for this: put a script path in `ACTIONS_RUNNER_HOOK_JOB_STARTED` in the runner's
`.env` file and it runs before every job, whatever repo it's for.

```sh
# /home/runner/actions-runner/.env
ACTIONS_RUNNER_HOOK_JOB_STARTED=/home/runner/hooks/job-started.sh
```

```sh
#!/bin/bash
# /home/runner/hooks/job-started.sh: start every job with an empty workspace
set -eu
if [ -n "${GITHUB_WORKSPACE:-}" ] && [ -d "$GITHUB_WORKSPACE" ]; then
  find "$GITHUB_WORKSPACE" -mindepth 1 -maxdepth 1 -exec rm -rf {} +
fi
```

Restart the service after editing `.env`. That handles the workspace. It does nothing for a job that `apt-get install`s
a package, starts a background process, fills `/tmp`, or writes into `~/.config`. Those stick around too, and a job from a
pull request can leave things behind on purpose. Keep plain runners like this on repos where you trust everyone who can
open a PR.

Proxmox does give you one cheap reset button. Snapshot the VM once it's registered and clean, and when it gets into a
weird state, `qm rollback <vmid> <snapshot>` puts it back. The runner registration lives on disk, so a rolled-back runner
comes back online as the same runner.

### Dedicated runner vs sharing the host

The runner VM shares the Proxmox node with the rest of my lab, and a build is the worst kind of neighbor. A C build with
`-j$(nproc)` on a 4-core VM will keep 4 host cores pinned for the length of the job. For reference, the php-sdk
Linux x86_64 build took 9m57s on the evening of April 2nd and 9m59s on the evening of April 4th on this runner (job
timings from runs
[23934915779](https://github.com/ephpm/php-sdk/actions/runs/23934915779) and
[23994291885](https://github.com/ephpm/php-sdk/actions/runs/23994291885)). That's ten minutes of the host's CPU going to
one job.

Things that help:

- Give the runner fewer cores than the node has. It's one job at a time per runner, so cores beyond what one build uses
  are wasted anyway.
- Lower its `cpuunits` (CPU weight) so the scheduler favors your other VMs when they compete.
- Don't overcommit memory on the node. A linker that gets ballooned at the wrong time fails in confusing ways.
- If you're running more than one runner VM, put them on a node by themselves when you can.

<!-- TODO(luther): what node the runner VM lives on and what else shares it. -->

### Why it's the easy one

Compared to the Mac and the Dev Kit there are no quirks to work around. Linux has a real init system, the runner's
`svc.sh` writes a normal systemd unit, the VM doesn't sleep, and nothing needs a logged-in user. If you only set up one
self-hosted runner, make it this one.

The other x86_64 VM on the same box is the harder one:
[Windows on Proxmox](/blog/2026/04/self-hosted-runner-proxmox-windows/).

### Links

- [go-proxmox](https://github.com/luthermonson/go-proxmox)
- [Ubuntu cloud images](https://cloud-images.ubuntu.com/noble/current/)
- [Running scripts before or after a job](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/run-scripts)
- [ephpm/php-sdk build workflow](https://github.com/ephpm/php-sdk/blob/main/.github/workflows/build.yml)
