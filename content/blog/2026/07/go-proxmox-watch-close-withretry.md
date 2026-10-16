---
title: "Two Small go-proxmox Fixes: A Watch Channel That Never Closed, and WithRetry"
description: "go-proxmox had a task log channel that only closed on the happy path. It also gained a composable RoundTripper that retries the 502s and 503s Proxmox throws during cluster transitions."
date: 2026-07-11T17:26:00-07:00
tags: [ proxmox, go, go-proxmox, concurrency, http ]
categories: [ go, homelab ]
images:
 - src: img/2022/02/proxmox-logo.png
   alt: "Proxmox VE Logo"
   stretch: "none"
   removeBlur: true
draft: false
---

Here are two changes from the v0.5 to v0.7 stretch of [go-proxmox](https://github.com/luthermonson/go-proxmox). They're
too small for a post each. They're worth knowing about if you write long-running tooling against Proxmox. One is a bug
in code I wrote about in 2022. The other is a feature that replaces a retry loop everybody was writing for themselves.

### The channel that only closed when things went well

In 2022 I wrote up [how tasks and `Watch` work](/blog/2022/03/proxmox-tasks-upid-wait-go/). `Watch` returns a channel of
log lines. A goroutine tails the task log until the task stops. Then it closes the channel so you can `range` over it.
Here is the tail loop from that post:

```go
func tasktail(start int, watch chan string, task *Task) error {
	for {
		if err := task.Ping(); err != nil {
			return err
		}

		if task.Status != TaskRunning {
			close(watch)
			return nil
		}

		logs, err := task.Log(start, 50)
		if err != nil {
			return err
		}
		// ...
	}
}
```

The channel is closed in exactly one place: when the task is no longer running. Every `return err` leaves it open. Since
context support was added in 2023 there's a new way to hit those error paths. You cancel the context.
[@aisbergg](https://github.com/aisbergg) reported precisely that in [#254](https://github.com/luthermonson/go-proxmox/issues/254):

```go
logChan, err := task.Watch(ctx, 0)
if err != nil {
	return err
}

for line := range logChan {
	logLine(line)
}
```

Here's what happens when you cancel `ctx`:

- the next `Ping` fails and `tasktail` returns the error
- the goroutine logs it and exits
- the `range` waits forever on a channel nobody will ever write to or close

A network blip did the same thing. It was just rarer.

![Timeline: before the fix a cancelled ctx makes tasktail return without closing the watch channel, so range logChan blocks forever; after #255 defer close(watch) closes it and the loop ends. Below, an example run: WithRetry retries a 503 and a 502 with full-jitter sleeps of up to 200ms and then 400ms before a 200 on attempt 3](/img/2026/07/go-proxmox-watch-close-withretry-timeline.svg)

The fix, [#255](https://github.com/luthermonson/go-proxmox/pull/255), also came from @aisbergg. It's the rule I should
have followed in the first place. The goroutine that owns the sends owns the close. It does it with `defer` so every
exit path is covered.

```diff
-	watch := make(chan string)
-
 	log, err := t.Log(ctx, start, 50)
 	if err != nil {
-		return watch, err
+		return nil, err
 	}
 ...
+	watch := make(chan string)
+
 	go func() {
+		defer close(watch)
 		t.client.log.Debugf("logs found for task %s", t.UPID)
 		for _, ln := range log {
 			watch <- ln
 ...
 		if task.Status != TaskRunning {
 			task.client.log.Debugf("task %s is no longer running, closing down watcher", task.UPID)
-			close(watch)
 			return nil
 		}
```

The other half of the diff is quieter but matters too. The error returns before the goroutine starts used to hand back a
real, unclosed channel alongside the error. Anyone who ranged over it without checking `err` first would block forever.
Now they get `nil`, and the channel is only created once there's a goroutine that will close it.

Both halves are @aisbergg's work: the diagnosis, the diff and the PR. My part was reviewing it and merging it. I tagged
v0.5.0 that evening so it shipped. The bug itself is mine, from 2022.

What the caller loses is the reason the stream ended. A closed channel looks the same whether the task finished or the
context was cancelled. Check `ctx.Err()` after the loop, or `Ping` the task and look at `ExitStatus`.

### WithRetry

Proxmox returns 502 and 503 while a cluster is doing something to itself. That could be `pveproxy` restarting after an
update, a node rejoining, or a quorum change. Every tool I've seen built on the client wraps its calls in the same
three-line retry loop. [#326](https://github.com/luthermonson/go-proxmox/pull/326) moves that into the client as an
option:

```go
client := proxmox.NewClient(url,
	proxmox.WithAPIToken(tokenID, secret),
	proxmox.WithRetry(), // 3 attempts, 200ms-5s backoff, retries 502/503/504/429 and network errors
)
```

It's an `http.RoundTripper` wrapped around whatever transport the client ends up with, rather than logic inside
`Client.Req`. That's what lets it compose with the other transport options in any order. Those are
`WithInsecureSkipVerify`, `WithRootCAs`, `WithProxy` and `WithRequestInterceptor`. It's installed in a finalize step
after all the options have run. So `WithRetry()` before or after `WithHTTPClient(custom)` both end up wrapping the
custom client's transport.

The loop itself is the standard shape with a few details I care about:

```go
for attempt := 0; attempt < r.policy.maxAttempts; attempt++ {
	// Reset the body before every attempt after the first.
	if attempt > 0 && rewind != nil {
		req.Body = rewind()
	}

	res, err := r.base.RoundTrip(req)

	// Honor user-initiated cancellation immediately — never retry it.
	if err != nil && ctxErr(ctx) != nil {
		return nil, ctx.Err()
	}

	lastRes, lastErr = res, err

	if !r.policy.condition(res, err) {
		return res, err
	}
	// ... non-idempotent check, last-attempt check ...

	delay := r.computeBackoff(attempt, res)

	// Drain and close the body so the underlying connection can be
	// reused on the next attempt.
	drainResponse(res)

	if err := r.policy.sleep(ctx, delay); err != nil {
		return nil, err
	}
}
```

- **Full jitter.** The wait is a random duration in `[0, min(5s, 200ms * 2^attempt))`. A dozen workers that all got a
  503 at the same moment don't all come back at the same moment.
- **`Retry-After` wins** whenever the server sends one, in both the seconds and HTTP-date forms. It's capped at the
  configured max so a misconfigured proxy can't park the client for an hour.
- **Cancellation is final.** A cancelled context is never retried. The sleep between attempts is a `select` on
  `ctx.Done()`, so cancelling during a backoff returns immediately.
- **The body is replayable.** The request body is buffered once and a fresh reader is handed to each attempt. In this
  client every body is already a `[]byte`, so that costs nothing.
- **4xx other than 429 fails fast.** A 400 isn't going to get better on attempt two.

The knobs are `WithRetryMax(n)`, `WithRetryBackoff(initial, max)` and `WithRetryCondition(func(*http.Response, error) bool)`.

One thing to think about before turning it on. Because the body is replayable, `POST` is retried along with the
idempotent verbs. That's the right call for a 503 from `pveproxy` itself. That means the request never reached the API
handler. A 502 from a reverse proxy in front of Proxmox is less clear-cut. The proxy may have given up after the backend
already accepted the request. A retried `POST /nodes/{node}/qemu` could then try to create the VM twice. With a fixed
VMID the second attempt fails because the ID is already taken. That's noisy but safe. If you create things without
pinning IDs, pass a `WithRetryCondition` that leaves 502 alone. Or only enable retry on read-heavy clients.

### Links
- [#254 Watch channel not closed](https://github.com/luthermonson/go-proxmox/issues/254) and [#255 the fix](https://github.com/luthermonson/go-proxmox/pull/255), both by [@aisbergg](https://github.com/aisbergg)
- [#326 WithRetry](https://github.com/luthermonson/go-proxmox/pull/326) and [retry.go](https://github.com/luthermonson/go-proxmox/blob/main/retry.go)
- [Waiting on Proxmox: UPIDs, Tasks and WaitFor in Go (2022)](/blog/2022/03/proxmox-tasks-upid-wait-go/)
- [AWS Architecture Blog, Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
