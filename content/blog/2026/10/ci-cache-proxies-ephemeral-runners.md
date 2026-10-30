---
title: "Cache Proxies for Ephemeral CI Runners"
description: "A fresh runner per job starts with an empty cache. Here is how ephemerd caches Go modules, Packagist, GitHub releases and the Actions cache on the node instead of re-downloading them every job."
date: 2026-10-23T15:00:00-07:00
tags: [ go, github-actions, ci, caching, ephemerd ]
categories: [ ephemerd, ci, go, containers ]
series: [ "ephemerd" ]
images:
 - src: img/2026/10/ci-cache-proxies-ephemeral-runners-flow.svg
   alt: "A fresh runner per job points go mod, cargo, composer, release downloads and actions/cache at matching cache proxies on the ephemerd node, where the caches persist and only reach the internet on a miss"
   stretch: "none"
draft: false
---

[ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) gives every CI job a brand new runner. A new container on
Linux, a new VM on macOS and Windows. The job runs, the runner is thrown away, and the next job starts from a clean
image. That is the whole point. Nothing a job does can leak into the next one.

Clean has a cost. A fresh runner has an empty cache. So every job downloads the world again. Go modules, crates, the
Rust toolchain, Composer packages, release tarballs. On a busy repo that is gigabytes a day, pulled over and over,
most of it byte-for-byte identical to what the last job pulled.

The fix is to put the cache somewhere the job can't throw away. That means the node. ephemerd runs a set of cache
proxies on the host. The caches outlive every job. The job only has to be told where to look.

![A fresh runner per job points go mod, cargo, composer, release downloads and actions/cache at matching cache proxies on the ephemerd node. The caches persist on the node and only reach the internet on a miss.](/img/2026/10/ci-cache-proxies-ephemeral-runners-flow.svg)

### How the proxies reach a job

Each proxy is a small HTTP server bound to the node. It speaks the protocol one package tool already expects. The Go
module proxy speaks the `GOPROXY` protocol. The Packagist proxy speaks the Composer repository protocol. And so on.

ephemerd does not intercept traffic transparently. It sets an environment variable in the job that points the tool at
the proxy. `GOPROXY` for Go. `CARGO_*` and `RUSTUP_DIST_SERVER` for Rust. `COMPOSER_REPO_PACKAGIST` for Composer. The
tool opts in because its own config tells it to. Unset the variable and the job goes straight upstream, exactly as it
would on a GitHub-hosted runner or a laptop. That matters. It keeps a build correct in places where the proxy does not
exist.

### The surprise was which ecosystems cost the most

Before adding more proxies I wanted to know what actually burned bandwidth. An inventory of the org's workflows
([PR #202](https://github.com/ephpm/ephemerd/pull/202)) found the answer was not where the caches were.

![Bar chart of workflow files per ecosystem. cargo build 39 with its proxy switched off, composer install 22 with no cache, releases download 19 with no cache, rust-toolchain 11 via the cargo proxy, and go mod download 4, the only one that already had a cache.](/img/2026/10/ci-cache-proxies-ephemeral-runners-inventory.svg)

Go modules had a working cache. It was also the smallest consumer, at 4 workflow files. The cargo proxy existed but
was switched off, across 39 files. Composer had no cache at all, across 22 files. GitHub release downloads had no
cache, across 19 files. On one real build of our PHP SDK, 40 of the fetched artifacts were cacheable source. They
accounted for 87 seconds of a 591 second build.

So PR #202 added two proxies. One for Packagist, one for GitHub release assets. It shipped in **v0.2.14**.

### GitHub releases have no registry

A release download is a plain `github.com` URL. There is no registry in front of it and no `GOPROXY`-style redirect
to hook. The URL is baked into whatever tool fetches it.

I did not want to intercept it transparently. Doing that means terminating TLS for job containers. That means a CA in
every image and a real change to what untrusted job code is exposed to. So the release proxy advertises a
`GHREL_PROXY` variable and lets tools opt in. Unset, every release download goes upstream unchanged.

There is one trap here. GitHub lets you delete an asset and re-upload a different one under the same URL. A plain
"cache forever" would serve the old bytes. The proxy caches release metadata with a TTL and revalidates a stale entry
with a conditional GET before serving it.

That revalidation exposed a latent bug worth calling out. The internal `ServeArtifact` helper served any cached entry
unconditionally. It never consulted the TTL. That was fine for npm, PyPI and pub. Those registries
never re-publish a file. Their proxies mark entries immutable. GitHub does not. PR #202 made `ServeArtifact` honour the
TTL, gated so the immutable callers are untouched.

### Packagist is almost all immutable

Composer splits into metadata and distribution archives. The metadata changes. The dist URLs name an exact commit, so
they are genuinely immutable. The proxy fronts the metadata with a TTL and caches dist archives permanently.

The archive route re-checks the host allowlist on every request. Without that check a job could hand the proxy an
arbitrary URL and use ephemerd as an open proxy on the LAN. The check keeps it closed.

### The cache was healthy and nobody could reach it

Here is where it got humbling. The proxies worked. The caches stayed at zero bytes anyway. It took three fixes to
understand why, and all three were the same mistake wearing different clothes.

Every proxy advertised the address it was asked to bind. That address is the CNI bridge gateway. It is correct only
where job containers share the host's bridge, which is Linux. On a macOS or Windows host the job runs inside a VM. From
in there, that gateway is the VM's own bridge, where nothing listens. The proxies are out on the host.

[PR #208](https://github.com/ephpm/ephemerd/pull/208) taught the proxies to rewrite the advertised address per
platform. On Linux the bridge gateway is right, so it is left alone. On macOS the daemon finds the host's address on
the guest NAT and rewrites the env to it. Then it probes every advertised address. If any probe fails, it advertises
nothing. On Windows it still advertises nothing. An unreachable `GOPROXY` hangs a Windows build rather than failing over, so
that stays suppressed until the host address is confirmed on metal. It shipped in **v0.2.17**.

Minutes after rolling v0.2.17 I read the env on a live M1 Mac mini node. Three of the four variables were correct. Go
was still pointing at an address that does not exist. `GOPROXY` is not a URL. It is a fallback list. Go accepts
elements separated by a comma or a pipe, mixed with the literals `direct` and `off`. Production emits something like
`http://…|direct`. Parsing that whole string as a URL yields no usable host, so the single-URL rewrite passed it
straight through.

[PR #209](https://github.com/ephpm/ephemerd/pull/209) rewrites each element of the list and leaves the literals alone.
The old test used a fixture I had invented, `GOPROXY=http://host:port`, which hid the bug. The new fixture is copied
verbatim off a real node, with a comment telling the next person not to tidy it. It shipped the same evening, in the
**v0.2.18 through v0.2.20** run of releases.

Even then the Mac's caches stayed empty. The macOS Mac mini was pulling on the order of 130 GB a day. The reason was
the third version of the same mistake. Cache-proxy env was only ever applied to containers. A macOS VM job has no
container. Its runner is started over SSH inside a fresh macOS VM, and nothing passed it any proxy env. Every job on
that Mac fetched straight from the internet.

[PR #211](https://github.com/ephpm/ephemerd/pull/211) threads the env into the VM through the runner's `.env` file. The
GitHub runner loads that file into every job. Before the runner starts, one bounded SSH command probes every proxy
address from inside the guest. It stages the env only if all of them answer. Any failure leaves nothing staged, and the
job runs exactly as before, just uncached. The all-or-nothing matters. Go falls back on `direct`. Cargo and rustup do
not fall back on an unreachable proxy. So a partial env could fail a build that would otherwise pass.

### Two fixes from a week of logs

A performance review of seven days of CI logs turned up two more problems. [PR
#214](https://github.com/ephpm/ephemerd/pull/214) fixed both. It shipped in **v0.2.21**.

The per-repo image cache for Docker-in-Docker never engaged on webhook-dispatched jobs. The main dispatch path built
its config without the provider and repo, so the cache namespace came out empty and the image cache was skipped. On one
node `mysql:8.0` was pulled 78 times in three days, at 7 to 45 seconds per database job. The fix passes the event's
provider and repo through.

The other was uglier. Cargo git dependencies bypass the crate cache, and libgit2 does not bound a silent TLS
connection. In one job a git dependency stalled for 13 minutes, then another 5.5, before failing with an SSL syscall
error. Both container configs now set `net.git-fetch-with-cli = true`. The proxy exports a low-speed limit and a 30
second window, so git drops a stalled transfer after 30 seconds and cargo's retry reconnects. Windows containers are
excluded, because their images ship no git.

### A LAN cache for actions/cache

The last piece is the biggest. It is also the one I am most careful about. [PR
#217](https://github.com/ephpm/ephemerd/pull/217) serves GitHub Actions' own cache service from the node. That is the
service `actions/cache`, `Swatinem/rust-cache` and the caching `setup-*` actions talk to. Entries are stored in a
writable OCI registry on the LAN. The baseline from the design work: 26% of a Rust job's time went to moving cache
archives over the WAN, and those archives run 0.4 to 1.6 GB.

Three parts of this were harder than they looked.

The Actions toolkit builds its request URLs in a way that discards the path, so a per-job secret in the URL does not
survive. Requests are attributed by source address instead, reusing the per-job network membership ephemerd already
tracks. The signed upload and download URLs still carry an unguessable token, so the attribution is a second gate, not
the only one.

The cache scope cannot come from the job a runner was dispatched for. JIT runners are fungible. A runner started for a
`main` push can be handed a fork PR. So a runner is tracked at create, where every lookup misses and every save is
refused, and bound only when GitHub reports the job as in progress. Nothing a PR saves is ever visible to a branch. A
fork whose branch is named `main` cannot write the `main` cache, and there is a test for exactly that.

Every failure is a cache miss, never an error. The server answers "not found" in three cases:

- the registry is down
- the runner isn't bound to a repo yet
- the entry is missing

The client treats each as a miss or a refused save. The server never returns a 5xx, because the client would retry
that five times and then log an error. If the runner patch can't be applied, the job just uses GitHub's cache.

This one is **off by default**. With the `[actions_cache]` config block absent, nothing about a job changes. It is also
**merged but not in a release yet**. The latest tag as I write this is v0.2.22, which was cut a couple of hours before
PR #217 landed. And it needs the other half, on the mayfly side, to
provision the writable registry before it can be switched on. It is Linux only. I have driven the wire format with Go
tests against a fake registry. I have not yet run the real Node client or the Azure SDK against it. The first job on
metal is that test.

### What's next

The macOS fixes recovered Go modules, cargo, rustup, release assets and Packagist on the Mac. What they did not recover
is Homebrew, Xcode and the macOS system downloads, which may be most of that daily pull. I would rather measure what the
node actually fetches than guess at a proxy for it.

The LAN Actions cache needs its first real client. Once the registry is provisioned I roll it to one node. I will watch
whether the Azure SDK accepts the registry's responses. Then I compare against the baseline before it goes any wider.

### Links

- [PR #202 — cache GitHub release artifacts and Packagist](https://github.com/ephpm/ephemerd/pull/202)
- [PR #208 — advertise an address jobs can actually reach](https://github.com/ephpm/ephemerd/pull/208)
- [PR #209 — rewrite GOPROXY's fallback list](https://github.com/ephpm/ephemerd/pull/209)
- [PR #211 — give macOS VM jobs the cache proxies](https://github.com/ephpm/ephemerd/pull/211)
- [PR #214 — dind image cache on webhook jobs; bound stalled cargo git fetches](https://github.com/ephpm/ephemerd/pull/214)
- [PR #217 — LAN GitHub Actions cache backed by a writable registry](https://github.com/ephpm/ephemerd/pull/217)
- [ephemerd: one daemon, a fresh runner per job, every OS](/blog/2026/08/ephemerd-one-daemon-every-os/)
- [Kata Containers for CI: boot cost isn't the problem, file I/O is](/blog/2026/09/kata-containers-for-ci/)
