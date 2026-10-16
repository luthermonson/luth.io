---
title: "os/exec's StdoutPipe Footgun: How cmd.Wait() Ate My Last Frame (Twice)"
description: "Calling Wait while a StdoutPipe reader is still draining throws away the child's last output. I shipped this bug twice in shipmates."
date: 2026-09-17T20:12:00-07:00
tags: [ go, os-exec, concurrency, shipmates, debugging ]
categories: [ go ]
series: [ "shipmates" ]
images:
 - src: img/2026/09/os-exec-stdoutpipe-wait-footgun-cover.svg
   alt: "Timeline of a child process exiting, Wait closing the pipe, and the reader losing buffered frames"
   stretch: "none"
draft: false
---

[shipmates](/blog/2026/08/shipmates-persona-based-ai-coding-crews/) spends most of its life spawning `claude` (and `codex`)
processes and reading newline-delimited JSON off their stdout. The last frame of a turn is the important one, a
`result` frame with the cost, the model and the final text. Stderr matters too, because that's where Claude Code
tells you `Invalid API key · Please run /login`. For a while I was throwing both away under load, and the cause is
documented right there in the `os/exec` godoc. I fixed it in one package and then found the exact same bug again
a month later in another.

### The symptoms

It started as a flaky test. `TestSpawnCapturesChildStderr` failed on the linux-arm64 CI leg with a terminal event
that had no stderr in it, just `exited exit status 1`. It passed 20 times in a row locally. The [issue I
filed](https://github.com/luthermonson/shipmates/issues/41) blamed a 2 second drain grace:

```go
if ok && p.stderrDone != nil {
    select {
    case <-p.stderrDone:
    case <-time.After(stderrDrainGrace): // = 2 * time.Second
    }
}
```

The theory was that on a loaded runner the stderr reader didn't finish inside 2 seconds, so the terminal event went
out with an empty tail. That's plausible, and it's wrong. Raising the timeout would have made the flake rarer and
fixed nothing.

There was a second symptom that looked unrelated. Every so often the stdout scanner died with
`read |0: file already closed`, which the read loop reported as a framing failure. That meant the terminal event got
published as a `string` instead of the `Terminal` struct consumers expected, so anything type-switching on the payload
quietly took a different branch. A third test in the codex adapter reported `unexpected_eof` for a frame the child
had definitely written.

Same race, three different losers.

### What the docs actually say

Both packages read the child with `cmd.StdoutPipe()` and `cmd.StderrPipe()`, and the supervisor called `cmd.Wait()`
on its own goroutine, concurrently with the readers. Here's the [`StdoutPipe` doc](https://pkg.go.dev/os/exec#Cmd.StdoutPipe):

> Wait will close the pipe after seeing the command exit, so most callers need not close it themselves.
> It is thus incorrect to call Wait before all reads from the pipe have completed.

`os/exec` owns the parent's end of any pipe it creates for you, and `Wait` closes it the moment the child is reaped.
Whatever the kernel still had buffered in that pipe goes with it. The child's last words weren't late, they were
gone, and `io.Copy` had already returned with `os.ErrClosed` before the drain grace was even consulted. That's why
a longer timeout couldn't help. `stderrDone` was closing *early*, with nothing in it.

It's very easy to write. You start the process, kick off a goroutine per pipe, and somebody has to call `Wait`
eventually so you don't leak a zombie. If that somebody isn't sequenced after the readers, you have this bug.

### A reproducer you can run

I wanted to see it fail on demand, not once a week on an arm64 runner. The child writes 200 JSON lines with a
`result` frame last and exits immediately. The parent reads slowly (1 ms per frame, standing in for real work) and
calls `Wait` concurrently, the same shape shipmates had.

```go
// go run . stdoutpipe   # the footgun
// go run . ospipe       # own the pipe, drain, then Wait
package main

import (
	"bufio"
	"fmt"
	"os"
	"os/exec"
	"time"
)

const frames = 200

func child() {
	w := bufio.NewWriter(os.Stdout)
	for i := 0; i < frames-1; i++ {
		fmt.Fprintf(w, `{"type":"assistant","seq":%d}`+"\n", i)
	}
	fmt.Fprintln(w, `{"type":"result","cost_usd":0.42}`)
	w.Flush()
	os.Exit(0) // everything is sitting in the pipe buffer now
}

func consume(sc *bufio.Scanner) (n int, sawResult bool, err error) {
	for sc.Scan() {
		time.Sleep(time.Millisecond) // a consumer that does real work per frame
		n++
		if sc.Text() == `{"type":"result","cost_usd":0.42}` {
			sawResult = true
		}
	}
	return n, sawResult, sc.Err()
}

func main() {
	if len(os.Args) > 1 && os.Args[1] == "child" {
		child()
	}
	mode := "stdoutpipe"
	if len(os.Args) > 1 {
		mode = os.Args[1]
	}
	cmd := exec.Command(os.Args[0], "child")

	type result struct {
		n   int
		ok  bool
		err error
	}
	done := make(chan result, 1)

	switch mode {
	case "stdoutpipe":
		out, _ := cmd.StdoutPipe()
		if err := cmd.Start(); err != nil {
			panic(err)
		}
		go func() {
			n, ok, err := consume(bufio.NewScanner(out))
			done <- result{n, ok, err}
		}()
		werr := cmd.Wait() // concurrent with the reader: documented as incorrect
		r := <-done
		fmt.Printf("%-10s wait=%v frames=%d/%d result=%v scanErr=%v\n", mode, werr, r.n, frames, r.ok, r.err)

	case "ospipe":
		pr, pw, _ := os.Pipe()
		cmd.Stdout = pw // an *os.File goes straight to the child; os/exec never closes our end
		if err := cmd.Start(); err != nil {
			panic(err)
		}
		pw.Close() // the child has its own copy; ours must go or we never see EOF
		go func() {
			n, ok, err := consume(bufio.NewScanner(pr))
			done <- result{n, ok, err}
		}()
		werr := cmd.Wait() // reaping no longer touches the read end
		r := <-done        // the reader ends at a clean EOF
		pr.Close()         // closed by us, after the drain
		fmt.Printf("%-10s wait=%v frames=%d/%d result=%v scanErr=%v\n", mode, werr, r.n, frames, r.ok, r.err)
	}
}
```

On Linux (WSL2, kernel 6.18, built with Go 1.26.5):

```
stdoutpipe wait=<nil> frames=136/200 result=false scanErr=read |0: file already closed
stdoutpipe wait=<nil> frames=136/200 result=false scanErr=read |0: file already closed
stdoutpipe wait=<nil> frames=136/200 result=false scanErr=read |0: file already closed
ospipe     wait=<nil> frames=200/200 result=true scanErr=<nil>
ospipe     wait=<nil> frames=200/200 result=true scanErr=<nil>
ospipe     wait=<nil> frames=200/200 result=true scanErr=<nil>
```

The reader gets whatever the scanner had already pulled into its buffer before `Wait` closed the pipe, then
`file already closed`. The `result` frame, the one with the cost in it, never arrives. Note that `Wait` returns `nil`,
because the child exited 0, so nothing at the process level tells you anything went wrong.

One thing surprised me. The PR that fixed this in shipmates says the flakes were Linux-only because on Windows
`internal/poll` defers `CloseHandle` until a blocked read returns. That holds when the reader is parked inside
`Read` at the moment of the close. In this reproducer the reader is sleeping between frames, so there's no read in
flight to protect it, and Windows 11 with Go 1.26.5 loses the same frames:

```
stdoutpipe wait=<nil> frames=136/200 result=false scanErr=read |0: file already closed
ospipe     wait=<nil> frames=200/200 result=true scanErr=<nil>
```

So don't count on the platform to save you. A reader that does any work between reads is exposed everywhere.

### The fix: own the pipes, one reaper closes them

The fix in [PR #44](https://github.com/luthermonson/shipmates/pull/44) is the `ospipe` branch above, grown up. Create
the pipes with `os.Pipe()` and wire the write ends onto `cmd.Stdout` and `cmd.Stderr`. When you hand `os/exec` an
`*os.File`, it passes it straight to the child and never tracks it, so `Wait` can't close it. Close your copies of the
write ends right after `Start`, and the read ends hit a clean EOF once every writer is gone.

From `internal/runtime/claude/claude.go`:

```go
stdoutR, stdoutW, err := os.Pipe()
if err != nil {
	return nil, err
}
stderrR, stderrW, err := os.Pipe()
if err != nil {
	closeFiles(stdoutR, stdoutW)
	return nil, err
}
cmd.Stdout = stdoutW
cmd.Stderr = stderrW
handle, err := r.supervise.Start(cmd)
if err != nil {
	closeFiles(stdoutR, stdoutW, stderrR, stderrW)
	return nil, err
}
// The child holds its own duplicates of the write ends now; ours have to go
// or the read ends would never reach EOF.
closeFiles(stdoutW, stderrW)
```

Then exactly one goroutine is allowed to close the read ends, and only after both readers say they're done:

```go
func (r *Runtime) reapPipes(p *proc) {
	select {
	case <-p.termDone:
		// The process is gone. Anything still holding a write end is a
		// descendant that inherited it, not the child, so bound the wait.
		t := time.NewTimer(readerDrainGrace)
		defer t.Stop()
		waitClosed(p.stdoutDone, t.C, r.stopFan)
		waitClosed(p.stderrDone, t.C, r.stopFan)
	case <-r.stopFan:
		// Explicit shutdown: Close must not sit through a drain window.
	}
	p.closePipes()
}
```

There's a cost, and it's the reason `StderrPipe` gets picked in the first place. With your own pipe, EOF needs *every*
writer gone, and a grandchild that inherited the child's stdout can hold it open long after the child is dead.
`readerDrainGrace` is 10 seconds and exists only for that case. In the normal path the child has been reaped, the
pipe is already at EOF, and draining takes microseconds. If the backstop does fire, the terminal event now says
"stderr not collected" instead of implying the child said nothing. Those used to be spelled the same way.

Two smaller pieces finished it off:

- A scan error that is `os.ErrClosed` or `io.ErrClosedPipe` is classified as EOF, not a malformed stream. A
  genuinely broken stream still shows up as `bufio.ErrTooLong` or a real I/O error.
- The read loop waits on `stderrDone` unconditionally once the terminal event arrives. The leftover timeout is a
  deadlock guard, not a budget.

Each symptom got a test with the timing forced instead of hoped for. `TestReadLoopWaitsForTheStderrDrainBeforePublishing`
holds the stderr reader past the old 2 s bound, and `TestReapDoesNotDiscardFramesTheChildAlreadyWrote` puts the frame in
the pipe, completes the reap, and only then starts the read loop. Both fail on the old code.

### Twice

That was mid-August and went out in v0.5.0. Then a cross-platform sweep after v0.6.0 turned up
[PR #60](https://github.com/luthermonson/shipmates/pull/60): the server's own spawn path for live crew members,
`spawnCrewLive`, still used `StdoutPipe`/`StderrPipe` with a concurrent `Wait`. Same bug, different file. On the
resume path a `claude -p` that exited would lose its final `result` frame and the tail of its stderr.

It had a second bug riding along. `Wait` was only called on the resume path. Every *fresh* spawn was never reaped,
which is a zombie per mate on Unix and a leaked process handle on Windows, piling up for the life of a long-running
ship. The fix mirrors `reapPipes`: a single `reapLive` goroutine is the only caller of `cmd.Wait()` and the only
closer of the read ends, and it starts unconditionally so both paths get reaped.

```go
// The single reaper: the ONLY caller of cmd.Wait() and the ONLY closer of
// the pipe read ends. Started here so it runs on BOTH the fresh and resume
// paths.
go s.reapLive(lp)
```

The test for that one uses a fake `claude` (via `TestMain`) that writes a terminal `result` frame and exits
immediately, then asserts the frame arrives on both paths and that the fresh child gets reaped.

### What I took away from it

- If you call `Wait` from anywhere other than "after every reader returned", don't use `StdoutPipe`/`StderrPipe`.
  Use `os.Pipe` or `cmd.Stdout = someWriter` and let `os/exec` do the copying.
- One goroutine owns `Wait` and one goroutine owns closing the read ends. Ideally the same one.
- A timeout around a drain is a deadlock guard. If it's the thing that decides whether you have the data, you have a
  race, not a slow reader.
- When you fix a bug class, grep for the rest of it. I fixed `internal/runtime/claude` and `internal/codexapp` and
  didn't look at `internal/server`, which had been doing the same thing the whole time.

### How I measured

The reproducer above, built with Go 1.26.5 on Windows 11 Pro (AMD Ryzen 9 5950X, 128 GB) and cross-compiled for
Linux, then run under WSL2 (kernel 6.18.33.2-microsoft-standard-WSL2). Three runs of each mode on Linux, one of each
on Windows; every run produced the same numbers shown above.

### Links

- [shipmates issue #41](https://github.com/luthermonson/shipmates/issues/41) and [PR #44](https://github.com/luthermonson/shipmates/pull/44) (the first fix)
- [shipmates PR #60](https://github.com/luthermonson/shipmates/pull/60) (the twin in `spawnCrewLive`)
- [`os/exec` Cmd.StdoutPipe docs](https://pkg.go.dev/os/exec#Cmd.StdoutPipe)
