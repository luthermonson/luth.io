---
title: "Windows Grab-Bag: SIGTERM That Doesn't Exist and Sparse Files That Break Docker"
description: "Go can't SIGTERM a child on Windows, so ephemerd's cloudflared processes piled up. Separately, Delivery Optimization's sparse temp files broke its Windows image layer commits."
date: 2026-10-03T14:30:00-07:00
tags: [ go, windows, cloudflare, containers, docker, ephemerd ]
categories: [ windows, ci, go ]
series: [ "ephemerd" ]
images:
 - src: img/2026/10/windows-sigterm-sparse-files-connections.svg
   alt: "Bar chart: 96 Cloudflare edge connections held by orphaned cloudflared processes versus 4 for the live daemon"
   stretch: "none"
draft: false
---

Two short ones from the Windows node of [ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) from the last few weeks. Neither is
complicated once you know what's going on. Both took longer than they should have to find, because the failure showed up a
long way from the cause.

### Part 1: there is no SIGTERM on Windows

ephemerd can run a Cloudflare tunnel so GitHub webhooks reach a node behind NAT. It does that by running `cloudflared` as a
managed subprocess. On Linux the child is tied to ephemerd's lifetime by the kernel with `Pdeathsig`:

```go
func applyPdeathsig(cmd *exec.Cmd) {
	if cmd.SysProcAttr == nil {
		cmd.SysProcAttr = &syscall.SysProcAttr{}
	}
	cmd.SysProcAttr.Pdeathsig = syscall.SIGTERM
	cmd.SysProcAttr.Setpgid = true
}
```

If ephemerd exits for any reason (clean stop, panic, `SIGKILL`) the kernel sends `cloudflared` a SIGTERM. On Windows
`SysProcAttr` has no `Pdeathsig`. The comment in the non-Linux file said the graceful `Close()` path was the sole
shutdown mechanism there. Both the context-cancel hook and `close()` did the same thing,
`cmd.Process.Signal(syscall.SIGTERM)`:

```go
cmd.Cancel = func() error { return cmd.Process.Signal(syscall.SIGTERM) }
```

Go's `os.Process.Signal` on Windows supports exactly one signal, `os.Kill`. Everything else returns an error. That error was
logged at `Debug` and dropped. On top of that, every config push and every upgrade on that node stops the service. In
practice each restart left a `cloudflared` behind. Measured on the node on 2026-09-22:

```
live ephemerd pid : 6908
children of live  : 1
orphaned          : 25
orphan parents still alive: 0
orphan RAM MB     : 868
```

The oldest orphan was 13 days old. 868 MB of RAM is annoying but it's not the real problem. All 25 were running the same
tunnel config and holding live connections to Cloudflare's edge:

![Edge connections](/img/2026/10/windows-sigterm-sparse-files-connections.svg)

Cloudflare load-balances incoming requests across every connection registered for a tunnel. 96 of the 100 connections
belonged to processes whose ephemerd was gone. Most webhook deliveries were being handed to clients that had nowhere to
forward them. That makes it the prime suspect for something I'd been chasing separately. The webhook listener on that
node seemed to go deaf for hours after an upgrade.

#### Job Objects are Windows' Pdeathsig

The Windows answer is a Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`. Put the child in the job and hold the job
handle. When the last handle closes, the kernel kills everything in it. A process's handles close when it exits by any
means. So this covers service stop, crash and `taskkill` the same way
([#206](https://github.com/ephpm/ephemerd/pull/206)):

```go
func bindChildLifetime(cmd *exec.Cmd) (io.Closer, error) {
	job, err := windows.CreateJobObject(nil, nil)
	if err != nil {
		return nil, fmt.Errorf("cloudflared: create job object: %w", err)
	}
	info := windows.JOBOBJECT_EXTENDED_LIMIT_INFORMATION{
		BasicLimitInformation: windows.JOBOBJECT_BASIC_LIMIT_INFORMATION{
			LimitFlags: windows.JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE,
		},
	}
	if _, err := windows.SetInformationJobObject(job,
		windows.JobObjectExtendedLimitInformation,
		uintptr(unsafe.Pointer(&info)), uint32(unsafe.Sizeof(info))); err != nil {
		_ = windows.CloseHandle(job)
		return nil, fmt.Errorf("cloudflared: set job limits: %w", err)
	}

	// OpenProcess rather than reusing the handle inside os.Process: that one is
	// owned by the runtime and closed when the Process is released, which would
	// make assignment racy against Wait.
	h, err := windows.OpenProcess(windows.PROCESS_SET_QUOTA|windows.PROCESS_TERMINATE,
		false, uint32(cmd.Process.Pid))
	if err != nil {
		_ = windows.CloseHandle(job)
		return nil, fmt.Errorf("cloudflared: open child process: %w", err)
	}
	defer func() { _ = windows.CloseHandle(h) }()

	if err := windows.AssignProcessToJobObject(job, h); err != nil {
		_ = windows.CloseHandle(job)
		return nil, fmt.Errorf("cloudflared: assign to job object: %w", err)
	}
	return &jobHandle{h: job}, nil
}
```

Two small decisions in there. Closing the job is itself a guaranteed kill. So `close()` releases it *after* the graceful
attempts rather than instead of them. That lets a clean shutdown close its edge connections properly. And if binding the
job fails, ephemerd warns and keeps the tunnel running. A tunnel that might leak beats no tunnel. But it's a `Warn`, not
a `Debug`. It's the difference between one stray process and one per restart forever.

The test is revert-verified. Reduce `bindChildLifetime` to the old no-op and it fails with "the child is unbound".

#### Cleaning up the ones already running

A Job Object only covers children this binary starts. It can't adopt the 25 already running. Left alone they'd survive
until a reboot. They weren't idle either. Each one kept re-registering edge connections for the same tunnel, and past
some count Cloudflare started refusing. The node was logging `unknown error registering the connection` about 300 times
an hour. Meanwhile the live daemon limped along on 3 of its 4 connections. PR #207 adds a sweep at startup, before
spawning:

```go
for _, pid := range pids {
	if pid == 0 || pid == self {
		continue
	}
	path, ok := processImagePath(pid)
	if !ok || strings.ToLower(path) != want {
		continue
	}
	if parentPID(pid) == self {
		continue // our own child, not a stray
	}
	if err := killPID(pid); err != nil {
		log.Warn("could not kill stray cloudflared", "pid", pid, "error", err)
		continue
	}
	killed++
}
```

The important line is the image path match. It only touches `cloudflared` running out of *this* ephemerd's data
directory, never by process name. Somebody's own tunnel on the same box is safe. Killing an operator's unrelated tunnel
would be a worse bug than the leak.

Here's what the errors did around the reap, from the hours recorded in the PR:

![Registration errors per hour](/img/2026/10/windows-sigterm-sparse-files-errors.svg)

Testing this one had a trap I fell right into. To test an orphan reaper you need real orphans. If the test spawns
`cloudflared` stand-ins directly, they're the test process's own children. The reaper deliberately skips those. My first
version did exactly that and passed while proving nothing. The test now spawns them through a launcher that exits. That
makes them genuine orphans. It's revert-verified twice:

- a no-op reaper fails with `stray pid 41292 survived the reap`
- matching on name instead of path fails with `killed an unrelated cloudflared`

### Part 2: sparse files and "unknown stream ID 9"

The other one came from building a Windows CI image with the MSVC toolchain baked in. That way Windows jobs stop
installing Visual Studio Build Tools every run. The build failed three times, for three different reasons:

1. the checkout was killed by an unrelated teardown incident on the node
2. the shell-form `RUN` for the `rustup` step lost its double quotes to Windows command-line parsing before PowerShell
   ever saw them ([#191](https://github.com/ephpm/ephemerd/pull/191))
3. this one, which isn't in my Dockerfile at all

```
failed to commit ... during finalize: failed to reimport snapshot:
Files/Windows/Temp/DO32B1.tmp: unknown stream ID 9
```

`DO*.tmp` in `C:\Windows\Temp` belongs to the Delivery Optimization service (DoSvc). That's the thing that peer-to-peer
distributes Windows updates. It runs inside Server Core containers and writes **sparse** temp files on its own schedule.
When a layer commits, the Windows snapshotter serializes the layer with go-winio's backuptar. That reads files through
the Win32 backup stream API. Stream ID 9 is `BACKUP_SPARSE_BLOCK`. The go-winio in that stack (v0.6.2) rejects it in its
layer-export pass with exactly that error. Upstream merged a fix for that path in September, but it isn't in a tagged
release yet. Until it is, whichever `RUN` step happens to be live when DoSvc drops a file loses its commit.

That's why it was nondeterministic. Earlier runs committed the same steps fine. The failing step was a read-only
verification `RUN` that didn't write anything itself.

The fix ([#192](https://github.com/ephpm/ephemerd/pull/192)) is one early layer, right after `SHELL`. It turns the
service off before anything else happens:

```dockerfile
RUN Stop-Service DoSvc -Force -ErrorAction SilentlyContinue; `
    Set-Service DoSvc -StartupType Disabled -ErrorAction SilentlyContinue; `
    Remove-Item -Path 'C:\Windows\Temp\DO*.tmp' -Force -ErrorAction SilentlyContinue; `
    Write-Host 'Delivery Optimization disabled (sparse DO*.tmp files break Windows layer commits)'
```

Service start type is registry state, so it persists into every later layer. `SilentlyContinue` is on all of it, because
a future base image without DoSvc should count as success. The cost was one rebuild of the cached Build Tools layer
(about 25 minutes). That cache would have been poisoned by the same bug sooner or later anyway.

Say you build Windows images with BuildKit or containerd and see `unknown stream ID 9`. Look for a background service
writing sparse files (and a go-winio without that September fix) before you look at your own steps.

### The common thread

Both of these were the platform quietly not doing what the code assumed. `Signal(SIGTERM)` returned an error into a
debug log. A service I'd never heard of wrote a file type the layer archiver can't handle. In both cases the fix was
small and the expensive part was noticing.

### Links

- [#206 Job Object for cloudflared](https://github.com/ephpm/ephemerd/pull/206)
- [#192 disable DoSvc in the MSVC image](https://github.com/ephpm/ephemerd/pull/192)
- [Job Objects (Win32)](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)
- [WIN32_STREAM_ID and BACKUP_SPARSE_BLOCK](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-win32_stream_id)
- [microsoft/go-winio](https://github.com/microsoft/go-winio)
