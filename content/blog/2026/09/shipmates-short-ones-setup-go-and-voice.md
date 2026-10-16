---
title: "shipmates Short Ones: 8 Minutes Extracting Go, and Three Voice-Loop Bugs"
description: "Four small stories from shipmates: setup-go stuck extracting on Windows CI, Cloudflare's 100 second timeout, a voice loop that transcribed itself, and events lost to a timestamp watermark."
date: 2026-09-12T14:35:00-07:00
tags: [ go, github-actions, ci, windows, cloudflare, voice, shipmates ]
categories: [ ai-agents, go, ci ]
series: [ "shipmates" ]
images:
 - src: img/2026/09/shipmates-short-ones-setup-go-and-voice-cover.svg
   alt: "setup-go on Windows CI: 7 seconds to download, 473 seconds extracting before the timeout"
   stretch: "none"
draft: false
---

A few [shipmates](/blog/2026/08/shipmates-persona-based-ai-coding-crews/) bugs that don't need a whole post each but
were too good to leave in commit messages.

### setup-go spends 8 minutes extracting Go on Windows CI

shipmates CI runs on my own [ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) runners, and on Windows those jobs
run inside a Hyper-V utility VM. The windows-x64 leg kept timing out in `actions/setup-go`. Here's the log from
[PR #48](https://github.com/luthermonson/shipmates/pull/48):

```
20:58:46  Setup go version spec 1.26.6
20:58:46  Attempting to download 1.26.6...     <- tool-cache MISS
20:58:48  Acquiring 1.26.6 from ...zip
20:58:53  Extracting Go...
21:06:46  ERROR: The action 'Run actions/setup-go@v5' has timed out after 8 minutes
```

Seven seconds from the start of the step to `Extracting Go...`, then almost eight minutes extracting until the step
timeout killed it.

![setup-go timing on the windows-x64 leg](/img/2026/09/shipmates-short-ones-setup-go-and-voice-cover.svg)

(Durations computed from the timestamps in the PR #48 job log above.)

The Go toolchain is about 15,000 small files, and every write under `_work` crosses a VSMB share from the VM to the
host. Small-file metadata over that share is brutally slow, and no timeout or retry in the workflow fixes it. The
toolchain has to stop being installed per job.

The plan was to take Go from the job's container image instead, since ephemerd intercepts `container:` on every platform
(an ordinary container on Linux, image layers extracted onto the VM disk on Windows and macOS), unlike GitHub's own
runner, which only supports it on Linux:

- Linux x64 and arm64 got `container: golang:1.26.6-bookworm`. `library/golang` is a real multi-arch manifest, so the
  same tag resolves natively on the arm64 runner.
- Windows x64 got `container: golang:1.26.6-windowsservercore-ltsc2025`.
- macOS keeps `setup-go`. There's no upstream image, and it wasn't the problem leg.
- `setup-go` is *skipped* on the containerized legs, not left to no-op. It never looks at `PATH`, it probes
  `RUNNER_TOOL_CACHE` and installs on a miss, so a toolchain already in the image wouldn't have stopped it.

A nice side effect: the race job lost its ~30 line "Ensure a C toolchain" step. `golang:*-bookworm` is built on
`buildpack-deps` and already has gcc, so no more run-time `apt-get` and no more negotiating whether the runner came up as
root, a sudo user, or neither.

Keep the image tag in lockstep with `go.mod`, by the way. The Linux images set `GOTOOLCHAIN=local`, so a `go.mod` bump
past the image fails loudly instead of quietly downloading a toolchain, which is what you want.

Now the honest part. The Linux legs went green. The Windows leg didn't get Go from the image. It failed at
`Initialize containers` in 13 seconds:

```
##[error]Container operations are only supported on Linux runners
```

That message is the actions runner's own Linux-only check, so on that leg the job never got as far as the image. The
8 minute timeout became a 13 second failure, which is better for the queue and still not a passing build. The fix
belongs on the ephemerd side, not in the shipmates workflow. PR #26 also has a note worth repeating if you're thinking about self-hosted
runners: a public repo on self-hosted runners lets a fork PR run code on your hardware. I did it on purpose, because
ephemerd exists to make that safe and shipmates is part of testing that, but it's a decision and it's written down as one.

### Surviving Cloudflare's 100 second edge timeout

Fleet Command sits behind cloudflared. The Commodore's conversation endpoint runs an LLM tool-calling loop, and a
multi-tool turn on a CPU model can take longer than Cloudflare's 100 second proxy limit. The operator got a raw 524 HTML
page dumped into a chat bubble.

The fix ([`bb93111`](https://github.com/luthermonson/shipmates/commit/bb93111)) streams whitespace while the model thinks.
Bytes on the wire reset the edge timer, and `JSON.parse` ignores leading whitespace, so the client didn't change at all:

```go
w.Header().Set("Content-Type", "application/json")
w.Header().Set("X-Accel-Buffering", "no")
fl, canFlush := w.(http.Flusher)
heartbeat := make(chan struct{})
go func() {
	ticker := time.NewTicker(15 * time.Second)
	defer ticker.Stop()
	for {
		select {
		case <-heartbeat:
			return
		case <-ticker.C:
			if canFlush {
				_, _ = w.Write([]byte(" "))
				fl.Flush()
			}
		}
	}
}()
finish := func(v conversationResponse) {
	close(heartbeat)
	_ = json.NewEncoder(w).Encode(v)
}
```

The catch is that once the first space goes out, the headers are committed with a 200. An `http.Error(w, …, 502)` later
does nothing useful. So errors moved into the JSON body (`"error": "llm: …"`) and ride the 200. Same commit, the client
started sending only the last 12 history messages, because the prompt was growing every turn and making the slow turns
slower.

### The voice loop that transcribed itself

In hands-free mode the browser listens with a voice activity detector, sends your speech to STT, sends the transcript to
the Commodore, and plays the reply through TTS. Then it listens again. Except it started transcribing its own replies back as
my input. One of them came back as "I'm always here ready to exist", which is a better line than anything I wrote that
week.

`speak()` was fire-and-forget. The loop checked `if (audioEl.paused || audioEl.ended) resumeListening()` right after
sending, and since playback hadn't *started* yet, the element was still paused, so the mic reopened immediately and
caught the assistant's voice from the speakers. [`9350c68`](https://github.com/luthermonson/shipmates/commit/9350c68)
made `speak()` resolve when playback finishes (on `ended`, `error`, or `pause`, since an interrupted reply counts as done),
`sendMessage` awaits it, and the loop resumes 350 ms later to let the speaker tail die out.

### Same-second events, gone

The bridge streams each captain's event log to the browser over server-sent events. It deduplicated with a watermark on
the event timestamp, which was a string at one second resolution:

```go
for _, e := range events {
	if e.Time <= high {
		continue
	}
	// send e
	high = e.Time
}
```

Any batch of events in the same second delivered only the first one. That's exactly what a broadcast produces: tell three
mates something and you saw one tell and one reply. The log is append-only, so [`2e317d5`](https://github.com/luthermonson/shipmates/commit/2e317d5)
switched to an index watermark, which is exact, and resets when the log shrinks (the lead restarted):

```go
if len(events) < sent {
	sent = 0 // lead restarted and its log reset; replay from the top
}
for _, e := range events[sent:] {
	// send e
}
sent = len(events)
```

If your log is append-only, the position is the cursor. Timestamps are for humans.

### Links

- [shipmates PR #48](https://github.com/luthermonson/shipmates/pull/48) (Go from the job image) and [PR #26](https://github.com/luthermonson/shipmates/pull/26) (CI on ephemerd)
- [`bb93111`](https://github.com/luthermonson/shipmates/commit/bb93111), [`9350c68`](https://github.com/luthermonson/shipmates/commit/9350c68), [`2e317d5`](https://github.com/luthermonson/shipmates/commit/2e317d5)
- [Cloudflare error 524](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/)
