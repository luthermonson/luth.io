---
title: "Webhooks Arrive Out of Order: How My Scheduler Killed Live Builds"
description: "ephemerd's orphan sweep trusted webhook history over reality and destroyed a runner mid-build. Now the rules only nominate, and a live busy check gets a veto."
date: 2026-10-07T21:10:00-07:00
tags: [ go, github-actions, webhooks, distributed-systems, scheduling, ephemerd ]
categories: [ ci, go ]
series: [ "ephemerd" ]
images:
 - src: img/2026/10/webhooks-out-of-order-scheduler-timeline.svg
   alt: "Timeline of three same-label jobs where a runner executing a build is destroyed 2.3 seconds before its in_progress webhook arrives"
   stretch: "none"
draft: false
---

The scheduler in [ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) is event-driven. GitHub sends a
`workflow_job` webhook when a job is queued, when it starts and when it completes. ephemerd creates and destroys runners
to match. For a while that worked well enough that I stopped thinking about it. Then a `docker-build` job failed with
"The self-hosted runner lost communication with the server". I pulled the node's journal. ephemerd had destroyed that
runner itself, on purpose, while it was running the build.

No webhook was lost. Nothing was delayed by more than a couple of seconds. This is the normal shape of a busy queue.

### JIT runners are fungible

ephemerd registers a just-in-time runner per queued job. The thing that's easy to miss is that a JIT runner isn't bound
to the job you registered it for. GitHub hands it *any* queued job whose labels it satisfies. Queue three jobs with the
same labels and register three runners "for" them. GitHub assigns them in whatever permutation it likes.

So the scheduler keeps a ledger with three parts:

- the intent: which job each runner was dispatched for
- `started`: which jobs have been seen running
- `bound`: which runners have been named in an `in_progress` delivery

An orphan sweep cleans up runners that never got work. An idle JIT runner holds a concurrency slot until somebody kills
it.

### The first round: balance on labels, not job ids

The first time this bit me was on the Mac. That node runs one job at a time, and its log had 266 orphan destroys. In
the most recent stretch there were 50 orphans against 222 completions, so about 18% of dispatches were wasted. Jobs that
ran in one to three minutes were sitting in the queue for one to two and a half hours.

Here's the mechanism. A dispatch for job A would block waiting for the single slot. Meanwhile GitHub gave job A to some
other runner that was already up, and it ran and finished. When the slot freed, the blocked dispatch registered a fresh
runner for a job that was already done. Nothing would ever be assigned to it. So it sat on the only slot until the
node's configured 90-minute orphan grace expired. Then it got swept, and its exit re-queued... you get the idea.

The fix in #131 was to stop accounting by job id at all. N queued jobs with the same label set are served by N runners
of that class, in any order. So the label set is the unit that has to balance:

```go
func labelSetKey(labels []string) string {
	norm := make([]string, 0, len(labels))
	for _, l := range labels {
		l = strings.ToLower(strings.TrimSpace(l))
		if l != "" {
			norm = append(norm, l)
		}
	}
	sort.Strings(norm)
	// ... dedupe ...
	return strings.Join(uniq, "\x1f")
}
```

With demand counted per class, the sweep got a second rule. Some runners are surplus. A runner is surplus when all three
of these hold:

- it's unbound
- its intent job has been seen running (discharged)
- it's in a class with zero remaining demand

Retire it right away instead of waiting out the grace window. That rule is the one that killed the build.

### The killed build

Three `dind-test` jobs with identical labels on the amd64 Linux node. Straight from the journal:

```
23:39:33      queued 556, 551, 567
23:39:34      JIT runner vivid_heisenberg  (intent 551)
23:39:35      JIT runner warm_ride         (intent 556)
23:39:35      JIT runner slim_mayer        (intent 567)

23:39:41.967  in_progress 556 -> vivid_heisenberg   # sets started[556]
23:39:41.967  in_progress 567 -> slim_mayer
23:39:45.368  vivid_heisenberg exits (556 done)
23:39:48.403  completed 567 -> handleCompleted -> sweepOrphanRunners()
23:39:48.403  "retiring discharged runner" runner=warm_ride dispatched_for_job=556
23:39:48.403  "destroying runner environment" id=warm_ride

23:39:50.716  in_progress 551 -> warm_ride          # 2.3s too late
23:49:42      completed 551 -> failure
```

![Incident timeline](/img/2026/10/webhooks-out-of-order-scheduler-timeline.svg)

GitHub gave job 556 to `vivid_heisenberg` and job 551 (the `docker-build`) to `warm_ride`. When `in_progress 556`
arrived it set `started[556]`. And 556 was `warm_ride`'s *intent*. So as far as the ledger was concerned, `warm_ride`
was:

- unbound, because its own `in_progress` hadn't been processed yet
- discharged, because a sibling had started the job it was dispatched for
- in a class with zero demand, because three runners served three jobs

That's the surplus rule exactly. The sweep runs on every job completion, and `slim_mayer` finished at 23:39:48.
`warm_ride` was destroyed 2.3 seconds before the webhook that would have bound it. The window is just the skew between
`in_progress` deliveries of the same burst. A few hundred milliseconds is enough.

### The fix: rules nominate, reality vetoes

[#151 (`8d6a288`)](https://github.com/ephpm/ephemerd/commit/8d6a288) keeps all the sweep logic but takes away its
authority. The rules now produce **nominations**. Nothing is torn down until a check confirms no job is running. That
check is made *at the moment of teardown*, from ground truth rather than event history. The invariant is in the PR
title: never destroy a runner that is executing a job.

Ground truth turned out to be simple. The Actions runner's listener process lives for the runner's whole life. But it
only forks a `Runner.Worker` child while a job is executing. So "is there a `Runner.Worker` in there" is the question.
Every platform has a way to ask it:

| Runner lives in | Probe |
|---|---|
| Linux container (runc) | `task.Pids()`, then `argv[0]` from `/proc` |
| Windows Hyper-V container | hcsshim `ProcessList()`, since guest processes are invisible to the host |
| macOS VM | `pgrep -x Runner.Worker` over the existing SSH channel |
| Linux VM on a Windows/macOS host | not available, falls through to the GitHub API |

When the local probe can't answer, the fallback is `GET .../actions/runners/{id}` and its `busy` flag. That's only for
runners already nominated. A 404 means not busy, since an ephemeral runner deregisters when its job ends. Any other
error is "unknown". The zero value of the verdict type is `Unknown`, so nothing can accidentally read as idle.

The decision itself is a pure function, which made it easy to test as a matrix:

```go
func decideReap(in reapInput) reapDecision {
	age := in.Now.Sub(in.DispatchedAt)

	if age >= in.Policy.HardBound && in.Busy != runnerbusy.Idle {
		return reapDecision{Destroy: true, Escape: true, Outcome: outcomeEscaped,
			Reason: "wedged: still not idle past the hard bound"}
	}
	switch in.Busy {
	case runnerbusy.Idle:
		return reapDecision{Destroy: true, Outcome: outcomeReaped,
			Reason: "verified idle: no worker process on the runner"}
	case runnerbusy.Busy:
		return reapDecision{Outcome: outcomeVetoed,
			Reason: "vetoed: the runner is executing a job"}
	default:
		if age >= in.Policy.Grace {
			return reapDecision{Destroy: true, Escape: true, Outcome: outcomeEscaped,
				Reason: "busy state undeterminable for the whole grace window"}
		}
		return reapDecision{Outcome: outcomeVetoed,
			Reason: "vetoed: busy state could not be determined"}
	}
}
```

There are two escape hatches, both based on time. An "unknown" verdict is held only for the grace window. That's
exactly the old behavior, so no platform got worse. A positive "busy" verdict is only overridden past
`job_timeout + 30m`. By then the job's context has been cancelled anyway, and a runner still claiming to be busy is
wedged.

I went back and forth on time versus "N failed probes in a row". The sweep runs on every completion. On a busy node a
probe-count escape would fire within milliseconds, in exactly the same-label burst this exists to survive. On a quiet
node it would practically never fire. Wedged-ness is a property of duration, so duration bounds it.

The probes do I/O, so they run with the scheduler lock released. That opens a window for an `in_progress` to bind a
runner mid-probe. Before unhooking anything, teardown re-validates under the lock that it's the same binding and still
unbound.

### Proving it on the node that broke

I deployed a test build to the same node. I set the orphan grace to 5 seconds so the rules would nominate aggressively.
Then I fed it a stream of same-label `dind-test` jobs:

```
01:19:58.039  orphan sweep nomination vetoed  runner=steady_hopper  busy_verdict=busy probe=container
01:19:58.371  in_progress -> steady_hopper      (+332ms)
01:20:26.213  orphan sweep nomination vetoed  runner=sure_darwin    busy_verdict=busy probe=container
01:20:26.976  in_progress -> sure_darwin        (+763ms)
01:20:41.179  orphan sweep nomination vetoed  runner=sure_sagan     busy_verdict=busy probe=container
01:20:41.786  in_progress -> sure_sagan         (+607ms)
```

Three nominations, each a fraction of a second before the `in_progress` that would have protected it. That's the same
race that killed `warm_ride`. All three were vetoed and all three jobs passed. In the same window a genuinely idle
runner was still reaped, so the veto isn't a blanket:

```
ephemerd_orphan_reap_decisions_total{outcome="vetoed",verdict="busy"} 3
ephemerd_orphan_reap_decisions_total{outcome="reaped",verdict="idle"} 1
ephemerd_orphan_reap_decisions_total{outcome="escaped",...}          0
```

It isn't perfect, and I'd rather say so. That idle runner got its `in_progress` 1.37 seconds after it was reaped.
GitHub had assigned it a job, but the worker hadn't forked yet, so the probe correctly said idle. The re-provision path
picked the job back up and nothing was lost. But the busy check shrinks the window to the gap between "GitHub
assigns" and "the worker spawns". It doesn't close it. Covering that last sub-second is what the grace window is still
for. It only came into play here because the test ran with a 5-second grace instead of the 10-minute default.

### Same lesson, twice more

Two later bugs were the same mistake in different clothes.

The Mac node served zero macOS jobs for about 28 hours ([#197](https://github.com/ephpm/ephemerd/pull/197)). The whole
time it reported `status: ok`, `active_jobs: 0` and a 200 from `/healthz`. A wedged VM was force-stopped to reclaim its
slot, and the guest ignored the stop. The code then waited forever on a channel whose comment assumed the stop always
works. The slot was never released. Nothing in the health output looked at whether jobs were moving.

Then on 2026-10-05 the Windows node ran about 10 real jobs in a day
([#210](https://github.com/ephpm/ephemerd/pull/210)). Meanwhile it booted about 23 runners for jobs GitHub had already
finished or cancelled, some three days earlier. The scheduler decided "already ran" from an in-memory `started` map that
expires. But a dispatch could wait on a slot with no time limit. So the fix asks the platform before acting on a stale
decision:

```go
// JobAwaitingRunner reports whether the job is still waiting for a runner
// to pick it up. false means it is running elsewhere, finished, or gone.
// An error means "could not determine" — callers must fail OPEN and
// provision anyway, because a skipped live job is worse than one wasted
// runner.
JobAwaitingRunner(ctx context.Context, event *JobEvent) (bool, error)
```

For GitHub that's one `GET /repos/{owner}/{repo}/actions/jobs/{id}`. It only happens when the dispatch waited for its
slot. A dispatch that got a slot immediately is acting on a webhook that's seconds old and skips the call. Slot waits
are now capped at 6 hours too. A job that's still queued gets re-dispatched by the reconcile poll anyway.

### Webhooks are hints

Every one of these bugs came from treating the event history as the state of the world. Webhooks arrive out of order
and arrive late. They describe a moment that's already gone by the time you act on them. That's fine for deciding what
to *try*. Anything destructive needs a source of truth to check right before acting. So does anything that burns a
scarce slot for 90 minutes. That could be a process table, the forge's API, or whatever is closest to reality. Make the
cheap event-driven logic nominate, and let reality veto.

### Links

- [ephemerd](https://github.com/ephpm/ephemerd) (PRs #131 and #151 have the full journals)
- [#197 macOS slot release](https://github.com/ephpm/ephemerd/pull/197)
- [#210 stop provisioning runners for jobs that no longer exist](https://github.com/ephpm/ephemerd/pull/210)
- [`docs/arch/webhook-job-lifecycle.md`](https://github.com/ephpm/ephemerd/blob/main/docs/arch/webhook-job-lifecycle.md)
- [GitHub: just-in-time runners](https://docs.github.com/en/rest/actions/self-hosted-runners#create-configuration-for-a-just-in-time-runner-for-a-repository)
