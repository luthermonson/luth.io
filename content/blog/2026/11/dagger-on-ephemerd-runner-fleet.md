---
title: "Dagger on ephemerd, and a Warm Cache on Throwaway Runners"
description: "A fresh engine per job means a cold cache every job — unless the cache lives on the node instead of in the engine. Benchmarked: an ephemeral BuildKit engine warms itself to within a few seconds of a long-running one, no persistent engine required."
date: 2026-11-18T09:00:00-07:00
tags: [ infrastructure, ephemerd ]
categories: [ ephemerd, ci, containers ]
images:
 - src: img/2026/11/dagger-ephemerd-cache-architecture.svg
   alt: "GitHub fires a workflow_job webhook at ephemerd on the node, which provisions a fresh VLAN-isolated runner that is destroyed after the job. Inside it a cold per-job BuildKit engine imports and exports cached layers over the GitHub Actions Cache v2 protocol to ephemerd's Actions-cache proxy, which persists the cache in a registry; point one shared registry in the VLAN at every host for a single fleet-wide cache."
   stretch: "none"
draft: false
---

A friend asked me whether Dagger replaces ephemerd. It doesn't — they sit at different layers, and I wrote half this post explaining that before I noticed the more interesting question hiding underneath it.

Ephemeral runners throw the whole machine away after every job. That is deliberate: a fork PR gets a clean, single-use box and nothing it does survives. But the build cache does not survive either. Every job starts cold — re-pull the base images, recompile from scratch, re-download every dependency. The usual fix is a long-running build engine with a persistent disk, which for a hardened fleet means standing up a privileged box that sits *outside* your isolation. That trade bothered me enough to measure the alternative.

The alternative works. **A fresh BuildKit engine, started cold on every job, can warm itself from a cache that lives on the node — and it lands within a few seconds of a long-running engine without keeping one running.** Here is the measurement, the wiring, and where Dagger does and does not fit.

### A fresh engine is a cold engine

Dagger wraps BuildKit; so does `docker buildx`. The speed of both comes from a content-addressed cache in the engine's state dir — solved build steps, extracted image layers, `RUN --mount=type=cache` scratch. A long-lived engine holds all of it and the second build of anything is nearly free. An engine that is created fresh for one job and destroyed at the end holds none of it. The Dagger docs put it plainly: "Each new machine starts with an empty cache."

So the honest baseline on an ephemeral runner is: full cost, every single run. In my benchmark a representative build — a Go program cross-compiled for six targets, about 31 seconds of actual compiler time — costs **~41 seconds every run** with no cache, forever. It never gets faster, because there is nothing to get faster from.

The fix everyone reaches for is a persistent engine. It works — the same build drops to **0.6 seconds** warm — but it is a long-running, privileged service. On my fleet, where every runner is pinned to an isolated VLAN behind an egress fence, a shared engine host is a box that lives *outside* that fence, holds every job's state at once, and is reachable over the network. That is a real containment cost for a cache win. (I get into exactly why privileged and placement matter in the safety section below.)

### Put the cache on the node, not in the engine

The thing I missed the first time: BuildKit can export and import its cache to an **external backend** — `registry`, `local`, `s3`, or `gha` (the GitHub Actions Cache service). The engine stays throwaway; the cache it produces gets pushed somewhere that outlives the job, and the *next* cold engine imports it at startup. The engine is ephemeral. The cache is not.

And ephemerd already runs exactly the backend this needs. Its Actions-cache proxy speaks the **GitHub Actions Cache v2** protocol, stores entries in a writable OCI registry it is pointed at, scopes them per repo, and already injects the cache URL and token into every job container for `actions/cache`. That registry is a config URL, not a fixed local box: aim one `registry:2` in the VLAN at every ephemerd host and the cache is shared across the whole fleet, keyed by repo — so a cold engine on one host warms from what another host built. BuildKit's `gha` cache backend speaks that same protocol. Point a cold per-job `buildkitd` at it with `--import-cache`/`--export-cache type=gha` and you get a warm cache on a throwaway engine, served by infrastructure the fleet already has.

### The numbers

I ran three scenarios, three runs each, same build and same output target every time so the only variable is the cache. Fresh BuildKit engine per run except the persistent column; every ephemeral engine logged an empty local cache (`buildctl du` = `0 B`) before building.

![Build time per run: no cache stays ~41s every run; the GHA cache pays ~56s on run 1 to populate then drops to ~3-5s; a persistent engine is ~0.6s](/img/2026/11/dagger-ephemerd-cache-benchmark.svg)

| scenario | run 1 (cold) | run 2 | run 3 | compile cached? |
|---|---|---|---|---|
| no cache (ephemeral) | 41.1 s | 41.5 s | 41.3 s | never |
| **GHA cache (ephemeral)** | 56.0 s | 4.8 s | 2.9 s | runs 2–3 |
| persistent engine | 41.6 s | 0.7 s | 0.6 s | runs 2–3 |

The headline: **ephemeral with the GHA cache is 3.9 seconds warm against 41.4 seconds with no cache — 10.7× faster, 91% of the build time gone.** Of the 40.8-second gap between "no cache" and a full long-running engine, the throwaway-engine-plus-external-cache path recovers **92%**, and it keeps nothing running between jobs.

It is not free, and the chart shows the catch honestly: **run 1 with the GHA cache is *slower* than no cache** — 56 seconds against 41 — because that first run pays a one-time cost to upload the whole cache (`mode=max`, base image included, ~1.25 GB). You pay once to populate; every run after rides it. The warm runs genuinely fetch from the server — each engine started at 0 B and pulled the cached layer back down — so the ~3–4 seconds is round-trips and a small download, not compute. A single long-running engine is still faster warm (0.6 s vs 3.9 s; the gap is network, not work), but it costs you a permanent privileged service, and this does not.

### How it is wired

The pieces here are spread across three trust zones. Here is the whole path, then the same thing in words.

![GitHub fires a webhook at ephemerd, which provisions a VLAN-isolated throwaway runner holding a cold per-job buildkitd. The engine imports and exports cache over type=gha to ephemerd's Actions-cache proxy, which stores it in a registry. Point one shared registry in the VLAN and every ephemerd host shares a single fleet-wide build cache.](/img/2026/11/dagger-ephemerd-cache-architecture.svg)

1. GitHub fires a `workflow_job` webhook at ephemerd on the node.
2. ephemerd provisions a **fresh runner** for that one job — a container on an isolated VLAN, behind the egress fence, destroyed when the job ends.
3. Inside the runner, the job starts a **per-job `buildkitd`** (privileged — see below) with an empty local cache.
4. The build runs with `--import-cache type=gha` (warm start) and `--export-cache type=gha` (push new layers), pointed at **ephemerd's Actions-cache proxy** — which runs on the node, speaks GHA Cache v2, and stores entries in a registry it is configured to use.
5. That cache **survives teardown**. The next job — on this host, or any other host pointed at the same registry — gets a brand-new cold engine that warms itself from it.

The engine is the disposable part; the cache is the durable part; and the durable part lives in a registry inside the fleet — one per node, or a single one shared across every host — not on some long-running box outside it. In a workflow it is a handful of lines around your existing build step:

```yaml
    # ephemerd already injects ACTIONS_RESULTS_URL + ACTIONS_RUNTIME_TOKEN
    # into the job for actions/cache; BuildKit's gha backend reads the same.
    - name: Build with a node-side cache
      run: |
        buildctl build \
          --frontend dockerfile.v0 \
          --local context=. --local dockerfile=. \
          --import-cache type=gha \
          --export-cache type=gha,mode=max
```

### Privileged is the price, and it is a safety decision

Running a BuildKit engine — Dagger's or buildx's — inside the job means a **`--privileged`** container. The engine needs it; the Dagger docs say it cannot run rootless, and buildkitd is the same. On ephemerd that means the pool needs `dind.allow_privileged = true`, which ephemerd defaults to **off on every platform**. Before you flip it, be exact about what it does and does not do, because it is easy to assume it hands the job the whole network. It does not.

**What privileged does NOT change: the network.** ephemerd enforces its egress allow-list, its RFC1918 blocks, and its cross-job fence as iptables chains in the *host* network namespace, and the VLAN at the switch. A privileged container holds `CAP_NET_ADMIN` only inside its own namespace and cannot reach any of those rules. The network blast radius of a privileged job is identical to a normal one.

**What privileged DOES change: escape.** It turns off the per-container hardening ephemerd normally applies — the trimmed capability set, seccomp, AppArmor. That is the containment that keeps a job from breaking out to the runner, and from the runner to the node. A privileged job that escapes lands in the host namespace, which is exactly where the cache, the firewall, and the other jobs live.

These are the rules:

- **Never run code you have not read under a privileged engine.** Fork PRs, drive-by contributions, anything from outside your team — do not let it execute here without reviewing the diff first. A privileged escape in a job running outside code is a node compromise, not a job failure.
- **Keep privileged pools single-tenant and trusted.** Gate them to internal pushes or maintainer-approved runs, the same way ephemerd already gates fork PRs. On a single-tenant pool the worst case of an escape is one ephemeral runner that is destroyed the moment the job ends.
- Safety first: if you are unsure, leave `allow_privileged` off. A cold cache is cheap; a privileged escape running someone else's code is not.

Keeping the cache on the node has a second benefit: the privilege and the escape risk stay inside the single-use runner, while the cache — the thing you actually want to keep — lives on the node, out of reach of the job's privilege.

### Where Dagger fits, and where it doesn't

There is a catch, and it is specific to Dagger. Everything above is **BuildKit/buildx**. Dagger's own engine does **not** expose cache export/import to any external backend — no `type=gha`, no `type=registry`, nothing you can point at ephemerd's proxy. (The `--cache-to type=registry` examples you will find online are the retired 0.2 CUE engine.) Modern Dagger's only sanctioned cross-run cache is a **long-lived engine with a persistent disk** or **Dagger Cloud**. And you cannot share one engine state dir across concurrent ephemeral engines — BuildKit's metadata store is single-writer and corrupts.

So the cache win is specifically for plain buildx. For Dagger itself you are back to a long-running engine: one inside the job (privileged, but no warmer than a plain build, because its cache still dies with the job) or one shared engine as a fleet peer (warm, but outside ephemerd's isolation unless you put the engine host inside the same VLAN and egress fence). Dagger buys you a real thing — the pipeline as one definition in Go that runs the same on a laptop as on the fleet. It does not buy you the warm-cache-on-a-throwaway-engine trick. buildx does.

That answers my friend's question better than the layer diagram did. If what you want is fast builds on isolated ephemeral runners, you do not need Dagger and you do not need a long-running engine. ephemerd already runs the cache backend that makes it work.

### What I tested

The build was a two-stage Dockerfile (`golang:1.23-alpine` cross-compiling six targets, ~31 s of real compile, then a `scratch` output stage so the export is ~37 MB and does not bury the cache effect). Engine was `moby/buildkit` v0.35.0, `--privileged`, fresh per run with no persistent volume; `buildctl du` confirmed `0 B` before each ephemeral build. Wall time is the `buildctl build` call only. The `gha` backend needs `url_v2=` to select the v2 client in BuildKit v0.35. Every number in the table is captured output, N=3; the GHA warm runs varied 2.9–4.8 s.

The mechanism — a cold engine warming from an external GHA v2 cache across runs — holds against a `registry` backend, an open-source GHA v2 server, and **ephemerd's own Actions-cache proxy**. For the last one I imported ephemerd's actual `actionscache` package and ran it the way the scheduler drives it, backed by a real registry, then pointed a fresh cold `buildkitd` at it. Run 1 populated the cache — the proxy stored the 162 MB build layer as 155 Azure blocks. A brand-new cold engine (`buildctl du` = `0 B`) then imported it: the compile step came back `CACHED` and the blobs streamed back from the proxy, taking the engine cache from 0 B to 1.19 GB, with no image pushed anywhere to shortcut it. BuildKit's Go client and ephemerd's proxy agreed on the Twirp JSON field names, the Azure block-upload sequence, and the download, with no changes to either.

One surprise: ephemerd does not check the `ACTIONS_RUNTIME_TOKEN` — it can't verify GitHub's token — so it attributes every cache request to a job by its **source IP** instead. The token only has to be well-formed because BuildKit's client parses it locally. The job's network identity is the credential, which is a clean fit for a fleet where every runner already has its own address.

### Links

- [Dagger](https://github.com/dagger/dagger) and the [self-hosting docs](https://docs.dagger.io/self-hosting)
- [BuildKit cache backends](https://docs.docker.com/build/cache/backends/) (registry, gha, s3, local)
- [Dagger 0.21 custom runner reference](https://docs.dagger.io/0.21/reference/configuration/custom-runner)
- [ephemerd](https://github.com/ephemerd/ephemerd)
- [mayfly](https://github.com/ephemerd/mayfly)
