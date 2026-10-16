---
title: "Finding a Data Race in rancher/remotedialer, Then Pinning Only Two-Thirds of the Fix"
description: "Two data races in remotedialer v0.6.1 that upstream's tests never ran with -race, and how my replace directive ended up pinned to the middle commit of my own fix."
date: 2026-09-01T21:05:00-07:00
tags: [ go, race-detector, remotedialer, rancher, go-modules, shipmates ]
categories: [ go ]
series: [ "shipmates" ]
images:
 - src: img/2026/09/remotedialer-data-race-two-thirds-fix-cover.svg
   alt: "Three commits of remotedialer PR 265 and which one the replace directive pointed at"
   stretch: "none"
draft: false
---

[shipmates](/blog/2026/08/shipmates-persona-based-ai-coding-crews/) ships connect to Fleet Command over outbound
websocket tunnels so nothing on the ship side has to be exposed to the internet. I didn't write that tunnel layer,
I used [rancher/remotedialer](https://github.com/rancher/remotedialer), the same library Rancher uses to reach
downstream cluster agents. I spent years around that code at Rancher, so it was the obvious pick. What I didn't
expect was for my CI's race job to find two data races in it in the first week.

### CI was the only place -race could run

`-race` needs cgo, which needs a C compiler. Neither machine I was writing shipmates on had one at the time, and every
agent session that tried `go test -race` hit `cgo: C compiler "gcc" not found`. So when I moved shipmates CI onto my
[ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) runners in [PR #26](https://github.com/luthermonson/shipmates/pull/26),
I added a dedicated race job on linux-x64. Today it runs in a `golang:1.26.6-bookworm` container, which ships gcc:

```yaml
race:
  name: race (linux-x64)
  runs-on: [self-hosted, linux, x64]
  container: golang:1.26.6-bookworm
  timeout-minutes: 30
  steps:
    - uses: actions/checkout@v4
    # ...
    - name: Test with race detector
      run: go test -race -count=1 ./...
```

The shipmates fleet tests open and tear down real tunnels quickly, and that job went red inside remotedialer
v0.6.1, not in my code.

### Race 1: Session.pingCancel

`ConnectToProxyWithDialer` builds the racing pair itself. It runs `Serve` on a goroutine and defers `Close`:

```go
go func() { session.Serve(ctx) }()   // Serve -> startPings: s.pingCancel = cancel   (write, no lock)
defer session.Close()                // Close -> stopPings:  if s.pingCancel == nil  (read, no lock)
```

Here's v0.6.1:

```go
func (s *Session) startPings(rootCtx context.Context) {
	ctx, cancel := context.WithCancel(rootCtx)
	s.pingCancel = cancel
	s.pingWait.Add(1)
	go func() {
		defer s.pingWait.Done()
		// ticker loop ...
	}()
}

func (s *Session) stopPings() {
	if s.pingCancel == nil {
		return
	}
	s.pingCancel()
	s.pingWait.Wait()
}
```

`Session` already embeds a `sync.RWMutex` and the neighboring fields use it. `pingCancel` just never got put under
it. In production the write and the read are separated by the lifetime of a session, so you only see it when sessions
open and close quickly: a reconnect storm, or a test suite driving real tunnels.

### Race 2: connection.err

`doTunnelClose` does an unguarded check-then-set on `c.err`, and it can be called at the same time from a pipe copy
noticing its peer failed, from `Session.Close` tearing everything down, and from `serveMessage` on a protocol error.
`connection` had no synchronization at all. The kicker is that remotedialer's own `TestExceedBuffer` trips this
race as soon as you turn the detector on. Upstream's Makefile `test` target didn't use `-race`, which is how both of
these survived.

### Fork, reproduce first, then fix

I forked it and did the red/green thing on purpose so the detector's report against unfixed code would be on record:

- [`cf2b335`](https://github.com/luthermonson/remotedialer/commit/cf2b3354b0f9f6eac268f08f81fe8c980a9e6aa0): add
  `TestSessionCloseDoesNotRaceServeStartup` and turn on `-race` in `make test`. This commit fails.
- [`7e65b78`](https://github.com/luthermonson/remotedialer/commit/7e65b789c52478b79589c8f1e8f63d0b3943dfac): lock
  `pingCancel`, add an `errMu` around `connection.err`. Green.

The test does what `ConnectToProxyWithDialer` does, in a loop, so the detector gets repeated shots at the startup
window:

```go
func TestSessionCloseDoesNotRaceServeStartup(t *testing.T) {
	for i := 0; i < 200; i++ {
		conn := &blockingWSConn{closed: make(chan struct{})}
		s := newSession(int64(i), "race-probe", conn)
		s.client = true // only the client side runs startPings

		ctx, cancel := context.WithCancel(context.Background())
		served := make(chan struct{})
		go func() {
			defer close(served)
			_, _ = s.Serve(ctx)
		}()

		s.Close() // the deferred Close, arriving while Serve is still starting

		_ = conn.Close()
		cancel()
		<-served
		s.pingWait.Wait()
	}
}
```

The fix for `stopPings` grabs the cancel func under the lock and releases the lock *before* cancelling, so
`pingWait.Wait()` is never awaited while holding a lock the ping goroutine's callees (`getSessionKeys` via
`sendSyncConnections`) also take. Nil-ing the field makes a second `Close` a no-op.

Then in shipmates, [`84e0d0e`](https://github.com/luthermonson/shipmates/commit/84e0d0e) pointed the dependency at the
fork:

```
replace github.com/rancher/remotedialer => github.com/luthermonson/remotedialer v0.2.6-0.20260805014355-7e65b789c524
```

v0.6.1 and upstream main had zero Go-file difference at that point, so the fork was exactly the pinned version plus
these fixes. Upstream PR to follow, drop the replace when it lands. Done, I thought.

### The third commit

About three hours after those two commits, and a minute after pinning shipmates to them, I opened
[rancher/remotedialer#265](https://github.com/rancher/remotedialer/pull/265) with them, and within the hour the new test caught one more thing on a machine with more hardware threads
(it failed about half of runs at 32). Look at what `7e65b78` did to `startPings`:

```go
s.Lock()
s.pingCancel = cancel
s.Unlock()
s.pingWait.Add(1)   // still outside the lock
```

`stopPings` now reads `pingCancel` under the lock, sees it non-nil, and goes on to `pingWait.Wait()`. But nothing says
the `Add(1)` has happened yet. A `Close` that lands between the `Unlock` and the `Add` calls `Wait` concurrently with
`Add` on a zero counter. The race detector flags it, and at runtime it can panic with
`sync: WaitGroup misuse: Add called concurrently with Wait`, in exactly the reconnect storm the PR was supposed to fix.

The third commit, `00b5151`, moves the `Add` inside the same critical section:

```go
s.Lock()
s.pingCancel = cancel
s.pingWait.Add(1)
s.Unlock()
```

Now observing a non-nil cancel implies the `Add` happened-before the `Wait`. I pushed it to the PR branch and moved on.

What I didn't do was bump the `replace`. shipmates stayed pinned to `7e65b789`, the *middle* commit of the PR, stamped
about three hours before the PR even existed. I had the `connection.err` fix in full and two-thirds of the
`pingCancel` fix. It took until a review pass right before the v0.5.0 release, thirteen days later, to notice, and
[PR #50](https://github.com/luthermonson/shipmates/pull/50) bumped it to the PR head:

```
replace github.com/rancher/remotedialer => github.com/luthermonson/remotedialer v0.2.6-0.20260805053951-00b5151280f8
```

and I checked that the resolved module actually carries `Add(1)` between `s.Lock()` and `s.Unlock()` rather than
trusting the pseudo-version.

### Seeing all three states

To write this up I ran the same reproducer against all three versions straight out of the module cache: upstream
v0.6.1, the middle commit, and the PR head, 20 iterations each with `-race` on Linux.

v0.6.1 is race 1, exactly as described. Write in `startPings`, read in `stopPings`:

```
WARNING: DATA RACE
Write at 0x00c000416de0 by goroutine 2007:
  github.com/rancher/remotedialer.(*Session).startPings()
      session.go:173 +0x58
  github.com/rancher/remotedialer.(*Session).Serve()
      session.go:223 +0x57

Previous read at 0x00c000416de0 by goroutine 1614:
  github.com/rancher/remotedialer.(*Session).stopPings()
      session.go:213 +0x31
  github.com/rancher/remotedialer.(*Session).Close()
      session.go:314 +0x3b
...
--- FAIL: TestSessionCloseDoesNotRaceServeStartup (0.01s)
    testing.go:1712: race detected during execution of test
```

The middle commit, the one I'd been shipping, still fails, but now it's the WaitGroup. `session.go:231` is
`s.pingWait.Wait()` and `session.go:179` is the `s.pingWait.Add(1)` that sat outside the lock:

```
WARNING: DATA RACE
Write at 0x00c00047e6a0 by goroutine 6057:
  runtime.racewrite()
      <autogenerated>:1 +0x1e
  github.com/rancher/remotedialer.(*Session).stopPings()
      session.go:231 +0xa7
  github.com/rancher/remotedialer.(*Session).Close()
      session.go:327 +0x3b

Previous read at 0x00c00047e6a0 by goroutine 6276:
  runtime.raceread()
      <autogenerated>:1 +0x1e
  github.com/rancher/remotedialer.(*Session).startPings()
      session.go:179 +0xaf
  github.com/rancher/remotedialer.(*Session).Serve()
      session.go:236 +0x57
...
--- FAIL: TestSessionCloseDoesNotRaceServeStartup (0.01s)
    testing.go:1712: race detected during execution of test
```

And the head is clean:

```
ok  	github.com/rancher/remotedialer	1.218s
```

The lesson I keep relearning: a `replace` to a pseudo-version is a pin to a commit, not to "my fix". If the branch
moves after you pin, you're on whatever you pinned, and nothing will tell you.

### Side effect: go install stops working

A `replace` directive in your `go.mod` means people can't `go install` your module:

```
$ go install github.com/luthermonson/shipmates@v0.5.0
go: github.com/luthermonson/shipmates@v0.5.0 (in github.com/luthermonson/shipmates@v0.5.0):
	The go.mod file for the module providing named packages contains one or
	more replace directives. It must not contain directives that would cause
	it to be interpreted differently than if it were the main module.
```

Release binaries are fine, goreleaser builds in the module so it honors the replace. So the README and the v0.5.0
release notes ([`f2c4bdf`](https://github.com/luthermonson/shipmates/commit/f2c4bdf)) say to grab a release binary or
`git clone` and `go build` until #265 merges. As of this writing the upstream PR is open, has no reviews, and is
showing merge conflicts, so the fork stays for now.

### Takeaways

- If your project depends on something concurrent, run *your* tests with `-race` even if upstream doesn't. A suite
  that churns real connections finds things code reading won't.
- Put the reproducer in its own commit before the fix. It made the upstream PR easy to explain, and it's how I could
  rerun all three states here.
- After you pin a fork, check the pin again when the PR is final.

### How I measured

The race output above came from copying each version of the module out of my Go module cache (`rancher/remotedialer@v0.6.1`,
`luthermonson/remotedialer@...-7e65b789c524` and `@...-00b5151280f8`), dropping the fork's `session_close_race_test.go`
into the two that didn't have it, and running `go test -race -count=20 -run TestSessionCloseDoesNotRaceServeStartup .`
in each under WSL2 (Go 1.26.0, gcc 15.2, 32 hardware threads on a Ryzen 9 5950X). Paths in the output are trimmed.

### Links

- [rancher/remotedialer#265](https://github.com/rancher/remotedialer/pull/265) (the upstream PR, three commits)
- [shipmates PR #26](https://github.com/luthermonson/shipmates/pull/26) (CI on ephemerd, the race job)
- [shipmates PR #50](https://github.com/luthermonson/shipmates/pull/50) (bumping the pin to the complete fix)
- [`go.mod` reference: replace directive](https://go.dev/ref/mod#go-mod-file-replace)
