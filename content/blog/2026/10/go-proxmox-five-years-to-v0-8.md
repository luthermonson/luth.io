---
title: "Breaking Changes Without Breaking Trust: Five Years to go-proxmox v0.8"
description: "I measured five years of go-proxmox releases and found a broken tag that can never be deleted, a release that didn't compile and 285 breaking changes in v0.7.0. This is that history and the rule I follow now."
date: 2026-10-04T19:40:00-07:00
tags: [ go, go-proxmox, proxmox, semver, open-source ]
categories: [ go, homelab ]
images:
 - src: img/2026/10/go-proxmox-five-years-to-v0-8-apidiff.svg
   alt: "Horizontal bar chart of apidiff incompatible and compatible changes for every go-proxmox release from v0.1.0 to v0.8.2"
   stretch: "none"
draft: false
---

[go-proxmox](https://github.com/luthermonson/go-proxmox) had its first commit in June 2021 and v0.8.2 went out on
September 30th. In between there were 366 commits and 43 people besides me with a merged PR. More to the point of this
post, there are downstream projects that I don't control pinning it in their `go.mod`. IONOS's
[Cluster API provider](https://github.com/ionos-cloud/cluster-api-provider-proxmox) and Sidero's
[Omni Proxmox provider](https://github.com/siderolabs/omni-infra-provider-proxmox) are both on v0.8.1 today. A library
with users like that gets judged less on its API. It gets judged more on what happens to their build when they bump the
version. I didn't always get that right, so here is the history, measured.

### What apidiff says

[`apidiff`](https://pkg.go.dev/golang.org/x/exp/cmd/apidiff) compares the exported API of two versions of a package. It
sorts every difference into compatible (additions) and incompatible (anything that can break a caller). I ran it across
every pair of consecutive releases from the last beta to v0.8.2:

![apidiff per release](/img/2026/10/go-proxmox-five-years-to-v0-8-apidiff.svg)

The 285 at v0.7.0 is the planned one, and I'll get to it. The ones that bother me more are the small numbers in places
they shouldn't be. v0.2.3 was a patch release with seven incompatible changes in it:

```
- (*Client).UpdateACL: changed from func(context.Context, ACL) error to func(context.Context, ACLOptions) error
- (*Container).NewFirewallIPSet: changed from func(context.Context, *FirewallIPSet) error to func(context.Context, FirewallIPSetCreationOption) error
- (*Container).RRDData: changed from func(context.Context, Timeframe, ConsolidationFunction) ([]*RRDData, error) to func(context.Context, Timeframe, ...ConsolidationFunction) ([]*RRDData, error)
- (*Node).Networks: changed from func(context.Context) (NodeNetworks, error) to func(context.Context, ...string) (NodeNetworks, error)
- (*VirtualMachine).ResizeDisk: changed from func(context.Context, string, string) error to func(context.Context, string, string) (*Task, error)
- CPUInfo.Mode: removed
- NodeNetwork.Active: changed from int to StringOrInt
```

Every one of those was a good change. `ResizeDisk` should return the task. `UpdateACL` was sending the wrong parameters
and never worked. But anyone who ran `go get -u` on a patch release got compile errors. v0.4.1 did it again with
`VNet.Tag` going from `uint16` to `uint32`. On a v0 module semver technically allows all of this. That isn't the same
thing as it being fine.

### A year and a half of alphas and betas

For a long time the answer to "when is the next release" was a tag like `v0.0.0-alpha7`. There were ten alphas and six
betas between January 2023 and March 2024. The API moved under all of them. In March 2024 someone opened
[#138 "Release v0.1.0"](https://github.com/luthermonson/go-proxmox/issues/138). I said I thought we were there. Then
nothing happened for almost three months. The IONOS folks asked again because their Cluster API provider needed a
tagged release. My reply was "ah no sorry... i just forgot. ill do it now". That's the honest version of how a lot of
small open source releases work. It's a bad experience for the people downstream.

Then in October 2024 came [#167](https://github.com/luthermonson/go-proxmox/issues/167): "Release v0.2.0 is tagged with
v0.1.2". I'd made the GitHub release with the wrong tag name. I deleted the tag and created the right one. I said "there,
v0.2.0 is created and v0.1.2 is deleted" and closed the issue. It is not deleted. The Go module proxy caches every
version it has ever been asked for, and it was asked:

```
$ curl -s https://proxy.golang.org/github.com/luthermonson/go-proxmox/@v/v0.1.2.info
{"Version":"v0.1.2","Time":"2024-10-03T23:58:17Z","Origin":{"VCS":"git","URL":"https://github.com/luthermonson/go-proxmox","Hash":"824e06bcfc27fd56c30a966a02a4582c77eaa09d","Ref":"refs/tags/v0.1.2"}}
$ curl -s https://proxy.golang.org/github.com/luthermonson/go-proxmox/@v/v0.2.0.info
{"Version":"v0.2.0","Time":"2024-10-03T23:58:17Z","Origin":{"VCS":"git","URL":"https://github.com/luthermonson/go-proxmox","Hash":"824e06bcfc27fd56c30a966a02a4582c77eaa09d","Ref":"refs/tags/v0.2.0"}}
```

Same commit, two versions, forever. In this case it's harmless because the code is identical. But a deleted Go tag is
never really deleted. If a tag is wrong, the fix is a new version (and a `retract` directive in `go.mod` if it's bad),
not a delete.

And on New Year's Eve 2025 I tagged v0.3.0 after merging a batch of PRs. Two of them were merged back to back. Each
added a `Node.Status` method to the same file. It doesn't compile:

```
# github.com/luthermonson/go-proxmox
.\nodes.go:42:16: method Node.Status already declared at .\nodes.go:29:16
```

(That's why the chart compares v0.3.1 to v0.2.4. apidiff can't load v0.3.0.) Two people sent the same four-line fix on
New Year's Day and v0.3.1 went out that afternoon. A release that doesn't build is about the most visible way to tell
your users you didn't run `go build` before tagging.

### The shape argument

Not every breaking change is an accident. The biggest design decision got argued about early. In March 2024
[#137](https://github.com/luthermonson/go-proxmox/issues/137) made a good case against the handle style.
`client.Node(ctx, name)` makes an API call to build the `*Node`. Then `node.Version(ctx)` makes a second one. A flat
`client.NodeVersion(ctx, name)` would be one call. I argued that loading the node gives you a handle you reuse for
everything after it. It also validates that the node exists, and you can always call `client.Get` directly. We
disagreed politely, and the reporter went on to keep writing their own library. Both are reasonable designs.

I kept the handles, and in May this year doubled down on them. The rule in `AGENTS.md` is now that anything with an
`/{id}/` in its Proxmox path gets a getter that returns a handle *without* an API call. So
`node.CephOSD(id).Scrub(ctx, false)` costs exactly one request. That's partly an answer to #137 two years late. It also
cost the biggest single break in the project's history, because snapshots and firewall rules had shipped the other way
for years.

The earliest compatibility decision was smaller and has held up better.
[#39](https://github.com/luthermonson/go-proxmox/issues/39) in January 2023 pointed out that `Clone` and `MoveDisk` had
hardcoded arguments. They couldn't express half of what the endpoints accept. Instead of adding more positional
parameters, required arguments stay positional. Everything optional moves into an options struct you can pass as `nil`.
Adding a field to an options struct is a compatible change. Adding a parameter is not. That pattern is why most
endpoints have been able to grow without breaking anyone. v0.8.0's only break was retrofitting it to
`VirtualMachine.Delete`:

```go
// before
task, err := vm.Delete(ctx)

// after — nil preserves the old default behaviour
task, err := vm.Delete(ctx, nil)
```

### v0.6 to v0.8: doing the big break on purpose

By May 2026 there was a pile of known-wrong things that could only be fixed by breaking the API:

- snapshot and firewall rule methods that threaded IDs through the parent
- `Net0`..`Net9` fields that couldn't represent `net10`
- about 30 config fields whose types let Go's zero value overwrite Proxmox defaults
- another 25 booleans typed as Go `bool` that couldn't express what Proxmox sends on the wire

The choice was a trickle of breaks across several releases or one release that took all of them. v0.7.0 took all of
them. That's the 285.

What made it acceptable, I hope, is the migration guide.
[migration/v0.7.0.md](https://github.com/luthermonson/go-proxmox/blob/main/migration/v0.7.0.md) groups the 285 lines
into seven themes. Each has before and after code. There's a TL;DR table of who is affected. It also has two sections
I'd now put in every guide. One is "What didn't change" (the wire format, `NewClient` with no options, the auth flow).
The other is "What's new (non-breaking)". Nobody upgrading should have to read raw apidiff output to find out whether
they're affected.

It's also a rule now. `AGENTS.md` says every release with at least one `apidiff` incompatible change ships a
`migration/v<VERSION>.md`. It goes in the release PR, not retrofitted later. Here are the commands to check:

```shell
go install golang.org/x/exp/cmd/apidiff@latest
git worktree add /tmp/old-version v<previous-version>
cd /tmp/old-version && apidiff -w /tmp/old.api .
cd <repo>          && apidiff -w /tmp/new.api .
apidiff -incompatible /tmp/old.api /tmp/new.api
```

Releases with no incompatible changes don't get a file. The directory is a record of breaks, not a ritual.

### Deprecate, then remove

v0.7.1 was a correctness release. It came from auditing every exported struct against the Proxmox and QEMU guest agent
schemas ([#337](https://github.com/luthermonson/go-proxmox/pull/337)). It found JSON tags that had never matched
anything:

- `"disallowed_leaders: "` with a colon and a space *inside* the tag
- `icmp_type` where Proxmox uses `icmp-type`
- `encryption` where the key is `encrypted`

My favorite is the firewall option `Ntp`. It shipped in v0.1.x and was always supposed to be `NDP` (IPv6 neighbor
discovery). Those fields had never once been populated.

Where a fix was a pure tag change, it just shipped. Where it needed a rename, the old field stayed and got neutralized
rather than deleted:

```go
// NDP — Neighbor Discovery Protocol toggle (PVE default 1). Use this
// field rather than Ntp; the upstream API field is `ndp`, not `ntp`.
NDP IntOrBool `json:"ndp,omitempty"`
// Deprecated: PVE never had an `ntp` firewall option — this was a typo
// shipped since v0.1.x. The intended field is NDP (above). Setting Ntp
// has no effect on PVE and reads always return zero. Will be removed in
// v0.8.0.
Ntp                     IntOrBool `json:"-"`
```

Code that references `.Ntp` still compiles and gets a deprecation warning from linters. It reads the zero value it was
always reading. That comment says it goes away in v0.8.0, and v0.8.2 still has it. That's its own small lesson about
putting version numbers in deprecation notices. It will go in the next release that has a migration guide anyway. Two
fields on `StorageContent` couldn't be handled that way. One was renamed and one retyped. That's why v0.7.1 has a small
migration file of its own.

### What I'd tell myself in 2021

- Tag real versions early. A v0.1.0 that you break in v0.2.0 is more useful to people than ten alphas.
- Never delete a pushed tag. The proxy already has it.
- Run `go build ./...` and `apidiff` before tagging. If a patch release shows incompatible changes, it's a minor release.
- Optional parameters go in an options struct from day one.
- When a big break is coming, batch it and write the guide in the same PR.

v0.8.2 was a docs example and a single code fix. VNC connections now notify the consumer and close cleanly when the VM
on the other end stops. Zero incompatible changes. That's the column I want to stay empty from here on.

### How I measured

I used `apidiff` from `golang.org/x/exp/cmd/apidiff` (installed with `go install golang.org/x/exp/cmd/apidiff@latest`).
I ran it against each tag's source extracted with `git archive <tag> | tar -x`. The steps were:

- `apidiff -w <tag>.api .` per tag
- `apidiff <prev>.api <tag>.api` for each consecutive pair from `v0.0.0-beta6` to `v0.8.2`
- count the lines under "Incompatible changes" and "Compatible changes"

v0.3.0 fails to type-check, so v0.3.1 is compared to v0.2.4. Windows 11, Go 1.26.5.

### Links
- [go-proxmox releases](https://github.com/luthermonson/go-proxmox/releases) and [migration guides](https://github.com/luthermonson/go-proxmox/tree/main/migration)
- [#39 compatibility policy](https://github.com/luthermonson/go-proxmox/issues/39), [#137 handles vs a flat client](https://github.com/luthermonson/go-proxmox/issues/137), [#138 release v0.1.0](https://github.com/luthermonson/go-proxmox/issues/138), [#167 mistagged release](https://github.com/luthermonson/go-proxmox/issues/167)
- [#227 the v0.3.0 build fix](https://github.com/luthermonson/go-proxmox/pull/227), [#335 migration guide convention](https://github.com/luthermonson/go-proxmox/pull/335), [#337 phantom JSON tags](https://github.com/luthermonson/go-proxmox/pull/337)
- [apidiff](https://pkg.go.dev/golang.org/x/exp/cmd/apidiff)
- [Go modules reference, retract](https://go.dev/ref/mod#go-mod-file-retract)
