---
title: "Cross-Platform Process Containment: Job Objects vs Process Groups vs cgroups"
description: "How shipmates bounds and tears down agent process trees on Windows, Linux and macOS, why I deleted 1,206 lines of cgroup code, and the grandchild that escaped the memory cap everywhere except my dev box."
date: 2026-09-26T16:20:00-07:00
tags: [ go, windows, linux, macos, job-objects, cgroups, process-groups, shipmates ]
categories: [ go, windows, containers ]
series: [ "shipmates" ]
images:
 - src: img/2026/09/process-containment-job-objects-process-groups-cgroups-cover.svg
   alt: "Agent process tree with the scope of kill and measurement on Windows, Linux and macOS"
   stretch: "none"
draft: false
---

Every turn a [shipmates](/blog/2026/08/shipmates-persona-based-ai-coding-crews/) crew member takes is a subprocess:
`claude` or the `codex` app-server, which in turn spawns shells, `go test`, `npm`, whatever the task calls for. Two
things have to be true about that tree. When a turn ends or gets killed, the *whole* tree goes, including the
grandchildren. And if somebody sets a memory or CPU budget, the budget applies to the whole tree, not just the
process I happened to start.

Each OS gives you a different primitive for that, and they're not equivalent. This is how shipmates ended up using
Job Objects on Windows and process groups plus sampling on Linux and macOS, why the cgroup path got deleted, and the bug
that only existed on the platforms I wasn't sitting in front of.

### The three primitives

| | Windows Job Object | Unix process group | Linux cgroup v2 |
|---|---|---|---|
| Who's in it | every descendant, automatically, unless it breaks away | every descendant that doesn't call `setpgid`/`setsid` | every descendant, can't leave without privileges |
| Tree kill | `TerminateJobObject`, or `KILL_ON_JOB_CLOSE` when the last handle closes | `kill(-pgid, SIGKILL)` | `cgroup.kill` (5.14+) |
| Memory cap | kernel-enforced, `JobMemoryLimit` on committed memory | none, you sample and kill | kernel-enforced, `memory.max` |
| Process count cap | kernel-enforced, `ActiveProcessLimit` | none | `pids.max` |
| Privileges | none | none | a delegated subtree (systemd scope, or root) |

On paper cgroups win. In practice, "a delegated subtree" is where it falls apart for a tool you install with a single
binary: on a desktop you need a systemd user scope with the right controllers delegated, in a container you usually
don't get one at all, and macOS doesn't have cgroups. Job Objects need nothing. Process groups need nothing.

### Deleting the cgroup launcher

The codex runtime originally carried its own Linux-only, cgroup-delegating launcher inside `internal/codexapp`. It
was reachable only through the codex transport, only when an operator set `containment.mode: cgroup`, and its single
real test was gated behind `SHIPMATES_REAL_CONTAINMENT_TEST=1`. Which meant it had **never run in CI**. When I
[moved CI to ephemerd](https://github.com/luthermonson/shipmates/pull/26) I even wrote down that the delegated-scope job
was the gap. Everywhere else, and by default, codex ran with no containment at all.

[`24bd788`](https://github.com/luthermonson/shipmates/commit/24bd788) deleted it: `containment_linux.go`,
`cgroup_discovery_linux.go`, their tests, `containment_pidfd_linux.go` and `containment_other.go`, 1,206 lines. Codex
now spawns its app-server through the same `containment.Watcher` that Claude uses, so every runtime gets the same
bounds on every OS.

The part I care most about is what happens to configs that still say `cgroup`. The old docs said it "degrades to
`watchdog` with a warning". Now the config fails to resolve, with `config.ErrCgroupModeRemoved`:

```
containment mode "cgroup" has been removed: shipmates now contains every runtime with the portable
watchdog (RSS/CPU sampling plus process-tree teardown on Linux, macOS and Windows). Use mode: watchdog
— your memory_limit_mb, cpu_limit_seconds and other limits carry over unchanged
```

An operator who asked for kernel enforcement and quietly got polling has been told something untrue about their own
deployment. A warning in a log nobody reads is still a lie. Same reasoning for `Caps.Containment`, which is now derived
from whether the supervisor actually bounds anything, so the `none` watcher can't report that it contains.

### Windows: suspended, assigned, resumed

On Windows the watchdog creates the child suspended, puts it in a Job Object before it runs a single instruction, then
resumes it. Assigning a running process leaves a window where it can spawn a child outside the job.

```go
func prepare(cmd *exec.Cmd, _ containment.Limits) error {
	if cmd.SysProcAttr == nil {
		cmd.SysProcAttr = &syscall.SysProcAttr{}
	}
	cmd.SysProcAttr.CreationFlags |= windows.CREATE_SUSPENDED | windows.CREATE_NEW_PROCESS_GROUP
	return nil
}
```

Then `attach` creates the job with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` and layers on whatever limits were asked for:

```go
info := windows.JOBOBJECT_EXTENDED_LIMIT_INFORMATION{
	BasicLimitInformation: windows.JOBOBJECT_BASIC_LIMIT_INFORMATION{
		LimitFlags: windows.JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE,
	},
}
if limits.MaxRSSBytes > 0 {
	info.BasicLimitInformation.LimitFlags |= windows.JOB_OBJECT_LIMIT_JOB_MEMORY
	info.JobMemoryLimit = uintptr(limits.MaxRSSBytes)
}
if limits.MaxProcesses > 0 {
	info.BasicLimitInformation.LimitFlags |= windows.JOB_OBJECT_LIMIT_ACTIVE_PROCESS
	info.BasicLimitInformation.ActiveProcessLimit = limits.MaxProcesses
}
```

Each flag is guarded on a non-zero limit, because setting a `LimitFlags` bit without its paired value is a no-op at best,
and a caller who asked for nothing shouldn't silently get kernel enforcement. `KILL_ON_JOB_CLOSE` is always on, so even
if shipmates itself dies, closing the last job handle takes the tree with it. That's the closest thing Windows has to
Linux's `Pdeathsig`.

`JobMemoryLimit` is committed memory across *every* process in the job, enforced by the kernel when the allocation
happens. To see it, I put a Go "agent" in a job with a 256 MiB limit and had it spawn a "tool" grandchild that commits
16 MiB at a time:

```
parent: agent in a job with a 256 MiB limit
tool: 16 MiB committed
tool: 32 MiB committed
...
tool: 208 MiB committed
tool: 224 MiB committed
runtime: VirtualAlloc of 16777216 bytes failed with errno=1455
fatal error: out of memory
```

Errno 1455 is `ERROR_COMMITMENT_LIMIT`. The grandchild was never touched by my code, it just inherited the job, and the
kernel stopped it short of the budget the parent was given (the agent and the Go runtime overhead eat the rest).
No polling gap.

### Linux and macOS: process groups and a sampler

On Unix the child is started as a process group leader with `Setpgid`, so its pid is the pgid and every descendant
inherits the group. Teardown signals the group:

```go
func killTree(cmd *exec.Cmd, kill bool) error {
	if cmd.Process == nil {
		return nil
	}
	sig := syscall.SIGTERM
	if kill {
		sig = syscall.SIGKILL
	}
	pgid, err := syscall.Getpgid(cmd.Process.Pid)
	if err != nil {
		pgid = cmd.Process.Pid
	}
	return syscall.Kill(-pgid, sig)
}
```

There's no kernel memory cap here, so a sampler polls RSS and CPU time on an interval and calls `killTree` when the
budget is exceeded. Which brings us to the bug.

### The bug my dev box hid

A cross-platform sweep after v0.6.0 found it ([PR #58](https://github.com/luthermonson/shipmates/pull/58)). The Unix
sampler measured RSS and CPU for `cmd.Process.Pid`, the root, and nothing else. The kill covered the whole process
group. So the *scope of measurement* and the *scope of enforcement* didn't match: a tool subprocess that hogged memory
or spun CPU was invisible to the sampler, and the cap never fired.

On Windows the Job Object enforces memory across the whole tree in the kernel, so the bug simply doesn't exist there.
I develop shipmates on Windows. Every manual check I'd ever done of the memory cap passed.

Here's a small Linux program showing exactly what the old sampler saw. The "agent" is a tiny Go process that runs a
"tool" child which touches 512 MiB:

```
t=0.5s  root pid RSS:    2.7 MiB   process group RSS:  519.2 MiB (2 procs)
t=1.0s  root pid RSS:    2.7 MiB   process group RSS:  519.2 MiB (2 procs)
t=1.5s  root pid RSS:    2.7 MiB   process group RSS:  519.2 MiB (2 procs)
t=2.0s  root pid RSS:    2.7 MiB   process group RSS:  519.2 MiB (2 procs)
```

With a 256 MB cap, the old sampler would read 2.7 MiB forever.

The fix ([`1642864`](https://github.com/luthermonson/shipmates/commit/1642864)) makes the Unix samplers measure the group.
On Linux that's a walk of `/proc`, reading field 5 of each `/proc/<pid>/stat` (the pgrp) and summing `statm` RSS and
`utime+stime` for the members. Field parsing has to count from the *last* `)`, because the comm field is parenthesized
and can itself contain spaces and parentheses:

```go
func parseProcStatPgrp(s string) (int, error) {
	closeIdx := strings.LastIndex(s, ")")
	if closeIdx < 0 {
		return 0, fmt.Errorf("watchdog: malformed stat: %q", s)
	}
	after := strings.Fields(s[closeIdx+1:])
	if len(after) < 3 {
		return 0, fmt.Errorf("watchdog: stat missing pgrp field")
	}
	pgrp, err := strconv.Atoi(after[2]) // state ppid pgrp ...
	if err != nil {
		return 0, fmt.Errorf("watchdog: bad pgrp: %w", err)
	}
	return pgrp, nil
}
```

macOS has no `/proc`, and `ps` has no dependable "select by process group" flag, so it snapshots every process with
`ps -Ao pgid=,rss=` (and `pgid=,time=` for CPU) and sums the rows whose pgid matches. Both versions treat "no processes
found in the group" as a failed sample, not a zero. A gone group leader skips the tick rather than reporting a reading
comfortably below the cap.

The summation logic is factored into pure functions so it's unit tested on every platform, and there's a `linux || darwin`
test that spawns a memory-hogging grandchild and fails against the old root-only sampler.

### What's still uneven

The Windows sampler is still root-only, on purpose. Memory is enforced tree-wide by the job, so the sampler there is
defense in depth. CPU seconds are the exception. The Job Object doesn't express a CPU-seconds budget the way the
watchdog does, so `cpu_limit_seconds` on Windows is approximated from the root process's times. A grandchild spinning
CPU on Windows escapes that cap today. The cheap fix is probably `QueryInformationJobObject` with
`JobObjectBasicAccountingInformation`, which reports total user and kernel time for everything that has ever run in the
job, no tree walk required. That's next.

And sampling is sampling. A process group can grow between two polls, and anything that calls `setsid` leaves the group
entirely. For an agent's tool calls that's an acceptable trade for a single binary that needs no privileges. If you need
a hard wall on Linux, run the ship inside a systemd scope or a container with `memory.max` set and let shipmates' own
limits be the inner layer.

### How I measured

Both demos are small standalone Go programs (Go 1.26.5 on Windows 11 Pro, cross-compiled and run under WSL2 kernel
6.18.33.2 for the Linux one) on an AMD Ryzen 9 5950X with 128 GB RAM. The Linux one starts an "agent" with
`Setpgid: true`, which runs a "tool" that allocates 32 x 16 MiB with every page touched, and prints `/proc/<pid>/statm`
RSS for the root next to the sum over the process group every 500 ms. The Windows one creates a Job Object with
`JOB_OBJECT_LIMIT_JOB_MEMORY` set to 256 MiB, assigns the agent, and lets the agent's child allocate 16 MiB at a time.

### Links

- [`24bd788`](https://github.com/luthermonson/shipmates/commit/24bd788): contain codex with the portable watchdog, delete cgroups
- [shipmates PR #58](https://github.com/luthermonson/shipmates/pull/58): sample the whole process group
- [Job Objects (Microsoft Learn)](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)
- [JOBOBJECT_BASIC_ACCOUNTING_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_basic_accounting_information)
- [cgroup v2 documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html)
