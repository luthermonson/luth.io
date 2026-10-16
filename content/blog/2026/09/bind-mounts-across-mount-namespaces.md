---
title: "Bind Mounts Across Mount Namespaces: Four PRs and a Container Escape"
description: "Making GitHub Actions container: jobs work through ephemerd's Docker API shim took four tries, then a TOCTOU race turned the working version into an escape."
date: 2026-09-24T21:02:00-07:00
tags: [ linux, containers, runc, containerd, mount-namespaces, security, github-actions, ephemerd ]
categories: [ containers, ci, security ]
series: [ "ephemerd" ]
images:
 - src: img/2026/09/bind-mounts-across-mount-namespaces-toctou.svg
   alt: "Sequence diagram of the bind mount TOCTOU race and the openat2 plus staging fix"
   stretch: "none"
draft: false
---

GitHub Actions has a `container:` key that runs a job's steps inside an image of your choosing. It's popular because it
gives you a reproducible toolchain without building a custom runner image. On a self-hosted runner it works by having the
runner drive Docker, and on [ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) "Docker" is a shim that speaks the
Docker API and creates containers in containerd. Getting bind mounts right through that shim took four PRs in a week. A
couple of months later the version that worked turned out to be an escape.

### What the runner asks for

For a `container:` job the runner does a `docker create` of a sibling container with a long list of binds, then runs every
step as `docker exec sh -e /__w/_temp/<uuid>.sh`:

```
-v /home/runner/_work:/__w
-v /home/runner/_work/_temp:/__w/_temp
-v /home/runner/_work/_actions:/__w/_actions
-v /home/runner/externals:/__e:ro
-v /var/run/docker.sock:/var/run/docker.sock
...
```

Every one of those source paths is a path *inside the runner's container*. The runner writes the step script into its own
`_temp`, and expects the sibling to see it at `/__w/_temp`. But the Docker API server (ephemerd) runs on the host, in a
different mount namespace, where `/home/runner/_work` doesn't exist. Translating "a path in container A's filesystem" into
"something I can bind into container B" is the whole problem.

### #82: binds were silently dropped

The first version of the shim `os.Stat`'d each bind source on its own filesystem and quietly skipped any that didn't exist.
Since none of them existed from the host's point of view, every bind was dropped. The sibling started fine, and the first
step died with:

```
sh: 0: cannot open /__w/_temp/<uuid>.sh: No such file
```

which is confusing because the script very much exists. [#82](https://github.com/ephpm/ephemerd/pull/82) added a
translation layer. The runtime registers the runner's snapshot and its own bind table (docker.sock, `/etc/hosts`, the
runner directory) with the shim, and each `-v` source resolves against, in order: the bind table by longest prefix, the
overlayfs upperdir (read-write), then the lowerdirs (read-only, since image layers are shared between jobs). Anything else
gets a 400 instead of a silent drop.

### #83: the wrong layer

That got the first script running and then `actions/checkout` failed:

```
exec: "/__e/node20/bin/node": stat /__e/node20/bin/node: no such file or directory
```

The resolver returned the first lowerdir that had the requested path. In the runner image, `/home/runner/externals/` exists
as a directory entry in one layer, but `node20/bin/node` lives in a later one. Binding the first match gave the sibling an
empty tree. Overlay merging is the kernel's job and I was doing it badly by hand.

[#83](https://github.com/ephpm/ephemerd/pull/83) let the kernel do it: use `/proc/<runner-pid>/root/<src>`. That magic link
walks into the runner's mount namespace and sees the merged overlay exactly as the runner does.

### #84: mount(2) won't cross namespaces

Which works perfectly for reading, and not at all as a bind source:

```
error mounting "/proc/<pid>/root/home/runner/_work/_temp/_github_workflow"
to rootfs ... MS_BIND|MS_REC: invalid argument
```

A bind source has to resolve within the *calling* process's mount namespace, and `/proc/<pid>/root` resolves into somebody
else's. [#84](https://github.com/ephpm/ephemerd/pull/84) switched to `os.Readlink("/proc/<pid>/root")` once at registration,
which should return the host path of the bundle rootfs where runc mounted the merged overlay. That's a normal directory in
the host namespace, and it has every layer's content.

### #85: readlink says "/"

Inside the Linux VM, that readlink returned `/`. Every time, before and after `task.Start`. I never fully traced why; the
log line was enough:

```
runner rootfs registered ... rootfs_path=/ bind_table_entries=3
```

So every source resolved to itself on the VM's root filesystem and got rejected. [#85](https://github.com/ephpm/ephemerd/pull/85)
stopped asking procfs and built the path from containerd's own state layout, which ephemerd fully controls since it embeds
containerd:

```go
runnerRootfsPath := filepath.Join(r.cfg.DataDir, "containerd", "state",
	"io.containerd.runtime.v2.task", namespace, id, "rootfs")
```

That's where the runc shim mounts the merged overlay, every time. A probe workflow wrote a script into `_temp` from the
runner, ran it from a sibling via `docker run -v`, and got `HELLO_FROM_SIBLING` back. `container:` jobs worked.

### The escape

The design at that point: take the job's source path, join it to the runner's rootfs path, `os.Stat` it, check that it's
still inside the rootfs, and put the resulting **path string** into the sibling's OCI spec. Then runc, in its own process,
some milliseconds later, walks that string again to do the mount.

The job owns every byte of the runner's rootfs. So between ephemerd's check and runc's walk, the job can rename the
validated directory and drop a symlink in its place. runc follows the symlink and mounts whatever it points at. That's
issue #125, and [#173](https://github.com/ephpm/ephemerd/pull/173) reproduced it against real runc 1.3.4, the same version
ephemerd embeds:

```
A  path-string source (the code at the time)   runc rc=0  CONTAINER-SAW: SWAPPED   <- escape reproduced
B  /proc/<pid>/fd/N                             mount ... : invalid argument
C  staging bind                                 runc rc=0  CONTAINER-SAW: PINNED
```

![TOCTOU and the fix](/img/2026/09/bind-mounts-across-mount-namespaces-toctou.svg)

Case B is worth a moment. The textbook TOCTOU fix is to resolve once, hold a file descriptor, and hand over the descriptor
instead of the name. There was even a "hardening" branch that did exactly that, putting `/proc/<pid>/fd/N` in the spec. It
fails for the same reason #84 did: runc runs in its own mount namespace, the fd path doesn't resolve there as a bind source,
and *every* bind fails, legitimate ones included. That branch wasn't a fix, it was a regression that nobody had run against
a real runc. The test that proves this is kept in the repo on purpose (`TestBindStaging_RealRunc_ProcFdSourceIsRejected`).

### The fix: pin it, then stage it

Two steps, and both are needed.

**Resolve once, to a descriptor, with the kernel enforcing containment.** `openat2(2)` with `RESOLVE_IN_ROOT` treats the
rootfs fd as `/` for the whole walk, so `..` and absolute symlinks clamp inside it rather than escaping, and the result is
held as an `O_PATH` fd. Renaming things afterwards can't change what the fd points at:

```go
how := &unix.OpenHow{
	Flags:   unix.O_PATH | unix.O_CLOEXEC,
	Resolve: unix.RESOLVE_IN_ROOT | unix.RESOLVE_NO_MAGICLINKS,
}
fd, err := unix.Openat2(rootFd, strings.Join(comps, "/"), how)
```

I used `openat2` rather than Go's `os.Root` on purpose. `os.Root` rejects absolute symlinks outright, and merged-usr images
(`/bin -> /usr/bin`) are full of them, so it would break every Ubuntu runner image. `RESOLVE_IN_ROOT` reinterprets them
relative to the root, which is what a symlink inside a container rootfs means. Kernels older than 5.6 get an
`O_PATH|O_NOFOLLOW` component-by-component walk instead, and a test asserts both resolvers land on the same inode.

**Stage it, because a descriptor can't go in the spec.** ephemerd does the bind itself, in its own mount namespace where
`/proc/self/fd/N` does resolve, onto a directory it owns. The spec gets the staging path:

```go
target := filepath.Join(m.dir, strconv.FormatUint(m.seq, 10))
// ... create a dir or empty file to match the source type ...

// /proc/self/fd/<n> is resolvable here because this IS the process that
// opened it and this IS its mount namespace — the two conditions runc
// cannot satisfy.
src := "/proc/self/fd/" + strconv.Itoa(p.fd)
if err := unix.Mount(src, target, "", unix.MS_BIND|unix.MS_REC, ""); err != nil {
	_ = os.Remove(target)
	return "", fmt.Errorf("staging bind source %s at %s: %w ...", p.logical, target, err)
}
```

runc still walks a path, but now it's `<data>/dind-binds/<job>/<n>`. The leaf is a bare counter, nothing from the request
ends up in the name, and every directory above it has to be root-owned and not group- or world-writable (checked, not
assumed, by `ensureTrustedAncestry`). There's nothing left for the job to swap.

The validation is the part I'd point other people at. The privileged tests run real runc against real containerd in a
root shell, and the swap test contains its own control: it runs the old path-string code in the same test and asserts it
*does* see `SWAPPED`. Without the control, a green run proves nothing, and that's precisely how the fd branch got through
review.

### Deleting through a live bind mount

One detail in #173 deserves its own heading. The obvious place for the staging dir was under the job's directory,
`<data>/jobs/<job-id>/`. But the runtime's orphan sweep `os.RemoveAll`s that tree, and **recursively deleting a directory
that contains a live bind mount deletes the files you see through the mount**. Here that's the runner's own rootfs. So
staging lives at `<data>/dind-binds/`, the teardown refuses to remove a directory that still has mounts under it, and the
startup sweep unmounts staged binds *before* sweeping containers and snapshots (a leaked staging mount also pins the rootfs
so the snapshot can't be removed).

Pins are held for the container's whole life, not just until the task starts, because `docker restart` makes runc read the
spec again.

### #201: sweeping someone else's staging dir

That startup sweep came back to bite me. A later change added a periodic reaper for dead job containers, and it called the
whole-tree `SweepStagedBinds` on every pass. That function's own doc comment said "startup only", for exactly this reason:
it doesn't know which jobs are live.

The symptom was jobs running several `docker run -v "$PWD":/w` steps failing partway through:

```
docker: Error response from daemon: bind mount /home/runner/_work/ephpm/ephpm -> /w rejected:
  creating bind staging mountpoint /var/lib/ephemerd/dind-binds/ephemerd-github-ephpm-sure_hopper/2:
  mkdir /var/lib/ephemerd/dind-binds/... : no such file or directory
```

The `/2` was the tell. Binds 0 and 1 staged fine, then a reaper tick for some unrelated dead container wiped every job's
staging directory, including this one. Already-running containers didn't notice (their mounts live in their own
namespaces), but the next `docker run -v` couldn't create its mountpoint. [#201](https://github.com/ephpm/ephemerd/pull/201)
scopes the periodic sweep to the container actually being reaped, and the test asserts the live job's directory survives.

### What I'd tell myself in May

- Paths are names, and names can change between when you check and when somebody else uses them. If a job controls any
  component of a path, that path can't cross a process boundary as a string.
- Mount namespaces are the constraint for every one of these bugs. `/proc/<pid>/root`, `/proc/<pid>/fd/N` and a readlink
  that says `/` are all the same lesson about which namespace a path resolves in.
- Security fixes need a test that fails on the unfixed code, run against the real thing. Mocks of runc don't have mount
  namespaces.

### Links

- PRs: [#82](https://github.com/ephpm/ephemerd/pull/82), [#83](https://github.com/ephpm/ephemerd/pull/83),
  [#84](https://github.com/ephpm/ephemerd/pull/84), [#85](https://github.com/ephpm/ephemerd/pull/85),
  [#173](https://github.com/ephpm/ephemerd/pull/173), [#201](https://github.com/ephpm/ephemerd/pull/201)
- [`docs/arch/dind-bind-translation.md`](https://github.com/ephpm/ephemerd/blob/main/docs/arch/dind-bind-translation.md)
- [openat2(2)](https://man7.org/linux/man-pages/man2/openat2.2.html)
