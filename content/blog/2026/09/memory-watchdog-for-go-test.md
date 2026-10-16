---
title: "A 1 TB Test: Building a Memory Watchdog for go test"
description: "A runaway test binary hit about 1 TB of commit charge and took my desktop down. Here's the PowerShell and POSIX sh wrapper I put around go test, and the bug in it that the Windows box couldn't show me."
date: 2026-09-22T20:48:00-07:00
tags: [ go, testing, windows, linux, powershell, shipmates ]
categories: [ go, windows ]
series: [ "shipmates" ]
images:
 - src: img/2026/09/memory-watchdog-for-go-test-cover.svg
   alt: "go test process tree with per-package test binaries watched by the safe-test wrapper"
   stretch: "none"
draft: false
---

On August 16th a `server.test.exe` from [shipmates](/blog/2026/08/shipmates-persona-based-ai-coding-crews/) climbed to
roughly **1 TB of commit charge** on my Windows workstation. Windows Terminal, which was hosting several agent sessions
in tabs, hard-faulted with `E_OUTOFMEMORY` and every one of those sessions died at the same moment. It wasn't the first
time either. An earlier one hit about 236 GB through an infinite `filepath.Dir` loop.

The machine has 128 GB of RAM. Commit charge doesn't care, it's backed by the page file, and Windows will happily let a
process reserve-and-commit until the commit limit is gone. Then everything else on the desktop starts failing
allocations too.

I wanted the worst case to be a killed test process instead of a dead desktop.

### Watch the children, not go

`go test ./...` builds one test binary per package and runs it as a child: `server.test.exe`,
`permissions.test.exe` and so on. A test that allocates without bound blows up in that child. The parent `go` process
stays small the entire time, so anything that watches `go` (a job limit on the shell, a `ulimit` on the parent's
RSS, eyeballing Task Manager for `go.exe`) misses it.

So [PR #54](https://github.com/luthermonson/shipmates/pull/54) added two scripts with the same contract,
`scripts/safe-test.ps1` for Windows and `scripts/safe-test.sh` for Linux, macOS and CI:

1. Start `go test -count=1 <target>` with output captured to a temp file outside the repo.
2. Every 0.5 s, walk the process tree from *that* `go` pid down and pick out the `*.test` binaries. A sibling
   `go test` running for another agent is left alone, which matters on a box running four or five crews.
3. Record a per-package peak.
4. If a child crosses the cap (default 2000 MB), kill it, kill the whole `go` tree, print which package tripped and
   the tail of its output, and exit 1.

The Windows tree walk is a `Win32_Process` snapshot and a stack:

```powershell
function Get-Descendants($snapshot, [int]$root) {
    $desc = [System.Collections.Generic.HashSet[int]]::new()
    $stack = [System.Collections.Generic.Stack[int]]::new()
    $stack.Push($root)
    while ($stack.Count -gt 0) {
        $cur = $stack.Pop()
        if ($snapshot.ChildrenOf.ContainsKey($cur)) {
            foreach ($c in $snapshot.ChildrenOf[$cur]) {
                if ($desc.Add($c)) { $stack.Push($c) }
            }
        }
    }
    return $desc
}
```

### Commit charge on Windows, RSS everywhere else

The two scripts deliberately measure different things.

On Windows the runaway *was* commit charge. Working set (physical RAM actually in use) could stay small while commit
ran to a terabyte, because the pages were committed but mostly never touched. So the PowerShell side reads
`PagedMemorySize64`, which is the process's page-file-backed committed memory:

```powershell
# PagedMemorySize64 is the process's committed (page-file-backed)
# memory -- the commit charge, which is what the OOM was about. Working
# set (physical RAM) would have stayed small while commit ran to a TB.
$commit = [int64]$gp.PagedMemorySize64
$pkg = $proc.Name -replace '\.test\.exe$', ''
```

On Linux the equivalent number is useless. Every Go process reserves a big chunk of virtual address space up front and
Linux overcommits by default, so VSZ is large and harmless. What takes a Linux box down is resident memory, so the sh
script reads RSS from `ps`.

### Validating it the right way

The 1 TB test itself is the more interesting part of the story. It was a test of a *bounded* behavior (the ring-buffer or
event-log-cap kind), and to prove the cap mattered it exercised the "fails without the guard" case by disabling the
bound and running unbounded. That turns a bounded test into an unbounded allocator. The rule that went into `CONTRIBUTING.md`:

> When you test that a limit holds, do not verify the "fails without the guard" case by disabling the bound and running
> unbounded. Assert against an injected small cap instead (fill a 4-slot buffer, prove the 5th is dropped), or run under
> the watchdog with a low cap.

The watchdog got the same treatment. You don't prove a kill switch works by letting something run away and hoping. The
PR validated it against a throwaway allocator with `-CapMB 300`: a healthy `./internal/server/...` run peaked at
104.3 MB and passed, and the allocator tripped the kill at 300.2 MB and exited 1.

I reran the same idea for this post with a test that grows by 16 MiB every 50 ms, touches every page so it shows up in
RSS too, and stops itself at 1.5 GiB so a watchdog that misses it can't hurt anything:

```go
func TestRunaway(t *testing.T) {
	var keep [][]byte
	for i := 0; i < 96; i++ {
		b := make([]byte, 16<<20)
		for j := 0; j < len(b); j += 4096 {
			b[j] = 1
		}
		keep = append(keep, b)
		time.Sleep(50 * time.Millisecond)
	}
	runtime.KeepAlive(keep)
}
```

On Windows:

```
safe-test: go test -count=1 ./permissions/...
safe-test: cap=300 MB  timeout=15 min  poll=0.5 s
------------------------------------------------------------
exit status 0xffffffff
FAIL	runaway/permissions	1.365s
FAIL

------------------------------------------------------------
safe-test: per-package peak commit charge
     354.0 MB  permissions
------------------------------------------------------------

WATCHDOG: package 'permissions' (pid 55232) crossed the 300 MB cap at 354 MB commit charge.
WATCHDOG: killed it and the parent 'go' process. This is the runaway -- fix the test before re-running the full suite.
```

It overshoots the cap by 54 MB because it samples. At 16 MiB per 50 ms, a half-second poll can miss about 160 MB of
growth between looks. That's the honest limit of this design, and it's spelled out in the contributing guide now: the
watchdog catches runaway *growth*, which is what has actually taken this machine down. It is not a hard per-allocation
ceiling. If you want that on Windows, a Job Object memory limit is the tool, which is what shipmates uses for the agent
processes themselves. That deserves its own post.

### The bug the Windows box hid

You may have noticed the package in my runaway test is named `permissions`. That's on purpose.

A review of the watchdog before #54 merged (the PR sat open from mid-August to mid-September) found a hole in the sh
version ([`166d208`](https://github.com/luthermonson/shipmates/commit/166d208)), so the fix went out with the rest of it in v0.6.0.
It matched test children with `ps -o comm=`, and Linux truncates `comm` to 15 characters (`TASK_COMM_LEN` is 16
including the NUL). Here's a binary named `permissions.test`:

```
comm=[permissions.tes]
args=[./permissions.test]
```

`permissions.tes` doesn't match `\.test$`, so the watchdog never saw it. Same for `personaname` and `containment`, which
are, of course, the security-sensitive packages. PowerShell's `Process.Name` isn't truncated, so the Windows script was
fine, and I only ever ran the Windows script at my desk.

Here's the original sh script (from `291f29e`) and the fixed one against the same runaway test under WSL2:

```
# before: matched on comm
safe-test: go test -count=1 ./permissions/...
safe-test: cap=300 MB  timeout=15 min  poll=1 s
------------------------------------------------------------
ok  	runaway/permissions	6.180s

------------------------------------------------------------
safe-test: per-package peak resident memory
  (no *.test child observed -- packages may have had no tests, or ran faster than one poll)
------------------------------------------------------------

safe-test: clean pass.
```

A 1.5 GiB test against a 300 MB cap, reported as a clean pass with exit 0. That's the worst thing a guardrail can do.
After the fix:

```
safe-test: go test -count=1 ./permissions/...
safe-test: cap=300 MB  timeout=15 min  poll=0.5 s
------------------------------------------------------------

------------------------------------------------------------
safe-test: per-package peak resident memory
     229.7 MB  permissions
------------------------------------------------------------

WATCHDOG: package 'permissions' (pid 405042) crossed the 300 MB cap at 373 MB resident.
WATCHDOG: killed it and the parent 'go' process. This is the runaway -- fix the test before re-running the full suite.
```

(The peak line is the last sample *under* the cap; the breaching sample is the one in the WATCHDOG line.)

The fix reads `args` instead, which comes from `/proc/<pid>/cmdline` and isn't truncated, takes argv[0], and basenames
it. The whole tree walk and filter is one awk pass:

```sh
snapshot() {
    ps -Ao pid=,ppid=,rss=,args= | awk -v root="$1" '
    {
        pid=$1; ppid=$2; rss=$3; exe=$4;   # exe = argv[0], first token of args
        RSS[pid]=rss; EXE[pid]=exe;
        kids[ppid] = kids[ppid] " " pid;
    }
    END {
        n=0; queue[n++]=root;
        for (i=0; i<n; i++) {
            m=split(kids[queue[i]], a, " ");
            for (j=1; j<=m; j++) if (a[j] != "") { desc[a[j]]=1; queue[n++]=a[j]; }
        }
        for (p in desc) {
            c=EXE[p]; sub(/.*\//, "", c);
            if (c ~ /\.test$/) { pkg=c; sub(/\.test$/, "", pkg); print pkg, p, RSS[p]; }
        }
    }'
}
```

The same commit dropped the default poll from 1 s to 0.5 s to shrink the window between samples.

The lesson is the same one twice. Test the guardrail against a deliberately small cap, and test it on the platform it's
going to run on. The commit message for the fix even says the live Linux `ps` behavior "can't be exercised on this
Windows dev box". It could, it just needed WSL and five minutes.

### Using it

```powershell
powershell -ExecutionPolicy Bypass -File scripts\safe-test.ps1
powershell -ExecutionPolicy Bypass -File scripts\safe-test.ps1 -Target ./internal/server/... -CapMB 500
```

```sh
scripts/safe-test.sh
scripts/safe-test.sh -t ./internal/server/... -c 500
```

Both exit non-zero if they kill anything or if `go test` fails, so they work as a pre-push hook. CI still calls
`go test` directly; the runners are ephemeral VMs and a runaway there only hurts itself.

### How I measured

Windows 11 Pro on an AMD Ryzen 9 5950X with 128 GB RAM, Windows PowerShell 5.1, Go 1.26. Linux runs were under WSL2
(kernel 6.18.33.2, procps-ng 4.0.4, Go 1.26.0). The runaway module lives in a scratch directory with a single
`permissions` package containing the test above. Commands:

```
powershell -NoProfile -ExecutionPolicy Bypass -File safe-test.ps1 -Target ./permissions/... -CapMB 300
sh safe-test-old.sh -t ./permissions/... -c 300   # git show 291f29e:scripts/safe-test.sh
sh safe-test-new.sh -t ./permissions/... -c 300   # scripts/safe-test.sh at 166d208
```

The 104.3 MB healthy peak and 300.2 MB trip are from the PR #54 description, not my runs.

### Links

- [shipmates PR #54](https://github.com/luthermonson/shipmates/pull/54) (the watchdog)
- [`166d208`](https://github.com/luthermonson/shipmates/commit/166d208) (argv[0] instead of `comm`)
- [`scripts/safe-test.ps1`](https://github.com/luthermonson/shipmates/blob/main/scripts/safe-test.ps1) and [`scripts/safe-test.sh`](https://github.com/luthermonson/shipmates/blob/main/scripts/safe-test.sh)
- [proc(5): `/proc/pid/comm`](https://man7.org/linux/man-pages/man5/proc_pid_comm.5.html)
