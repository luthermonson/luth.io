---
title: "Maintaining an Open-Source Library With AI Agents: AGENTS.md and the Rule They Kept Breaking"
description: "Six weeks of go-proxmox with coding agents doing most of the typing taught me what belongs in AGENTS.md. Five parallel agents got the same API shape wrong, and a coverage metric bent the code around it."
date: 2026-06-27T18:47:00-07:00
tags: [ go, go-proxmox, ai, agents, open-source ]
categories: [ go, ai-agents, homelab ]
images:
 - src: img/2026/06/go-proxmox-agents-md-rule.svg
   alt: "Decision diagram: endpoints without an id in the path become methods on the parent, endpoints with an id get a handle type"
   stretch: "none"
draft: false
---

On May 9th I merged [#264](https://github.com/luthermonson/go-proxmox/pull/264). It added an `AGENTS.md` to
[go-proxmox](https://github.com/luthermonson/go-proxmox) and a stub `CLAUDE.md` that just points at it. The next six
weeks ran through the v0.8.0 migration guide on June 15th. In that time `main` took 77 commits and +31,794 / -1,144
lines. The repo went from 84 to 186 `.go` files. Endpoint coverage went from 30% to
[100% of the Proxmox API](/blog/2026/06/go-proxmox-endpoint-coverage/). Most of that code was typed by coding agents,
a lot of it by several at once. This is what that looked like from the maintainer's chair. It's mostly the parts that
went wrong, because that's where `AGENTS.md` got its content.

### What's in the file

The first version was boring on purpose. It covered:

- How to build and test (Mage targets, which env vars the integration suite reads, that CI only runs unit tests).
- How a request flows through `Client.Req`.
- The `client.Node(ctx, name)` to `node.VirtualMachine(ctx, id)` to `*Task` traversal.
- A handful of gotchas like the ~16 KB upload cap.

The one *required* section was about tests. Any PR that adds or changes an endpoint ships a gock mock in
`tests/mocks/pve9x/<resource>.go`. It also ships a unit test with a happy path and at least one error path.
Integration tests need a live cluster and CI doesn't have one. Without that rule an agent can write a wrapper that has
never executed, and the green check means nothing.

Everything else in the file today was added after something went wrong.

### Five agents, one rule, five times

The coverage push went area by area. The big areas were split into groups and handed to sub-agents in parallel. For
qemu that was five PRs, [#301](https://github.com/luthermonson/go-proxmox/pull/301) through #305, each one fine on its
own. They all appended to the same `virtual_machine.go`, `types.go` and mock file. Merging them one at a time meant a
four-way rebase. I closed all five and cherry-picked them into
[#306](https://github.com/luthermonson/go-proxmox/pull/306) with the per-feature commits kept. That became the
pattern: parallel branches, one combined PR.

Ceph ([#307](https://github.com/luthermonson/go-proxmox/pull/307)) was five agents on 41 endpoints. The combined PR
needed a cleanup commit for placement alone. Two groups put their mocks in the existing `ceph.go`, and two appended
them to `nodes.go`. One created a new `nodes_ceph.go` and registered a second, duplicate loader for it. Three of them
wrote the identical `func cephNode() *Node` test helper in three files. None of that is wrong code. It's five people
who didn't talk to each other, which is exactly what it was.

The bigger problem showed up in the method signatures. Several agents independently produced this:

```go
node.CephOSDScrub(ctx, id, deep)
node.DeleteCephOSD(ctx, id, cleanup)
node.CephOSDLVInfo(ctx, id, ...)
```

Every OSD operation hung off `*Node` with the OSD id threaded through as an argument. It compiles and it works. But
it's not the shape the rest of the library uses. Storage, VMs, containers and network interfaces all have a getter
that returns a handle, and the operations live on the handle. The Proxmox schema even tells you which one to use. The
path is `/nodes/{node}/ceph/osd/{osdid}/scrub`, and the `{osdid}` segment says "this is one of many". Once I looked,
the same thing had happened to node services, node replication jobs and snapshots. It had also happened to firewall
rules on nodes, VMs and containers. Fixing it took a refactor commit inside #307 plus four PRs,
[#308](https://github.com/luthermonson/go-proxmox/pull/308) to
[#311](https://github.com/luthermonson/go-proxmox/pull/311). The last two were breaking changes to methods that had
already shipped.

The fix that matters is [#312](https://github.com/luthermonson/go-proxmox/pull/312), which turned it into a rule with no
judgement call in it:

![The singleton vs instance rule](/img/2026/06/go-proxmox-agents-md-rule.svg)

```go
// Getter: no API call, just constructs the handle.
func (n *Node) CephOSD(id int) *CephOSD {
    return &CephOSD{client: n.client, Node: n.Name, ID: id}
}

// Operations live on the instance, not the parent.
func (o *CephOSD) Scrub(ctx context.Context, deep bool) error { … }
func (o *CephOSD) Delete(ctx context.Context, cleanup bool) (*Task, error) { … }
```

The rule comes with three more things:

- An explicit anti-pattern line naming `node.CephOSDScrub(ctx, id, ...)`.
- A note that creation stays on the parent, because there's no instance yet.
- A list of every existing handle type to copy.

The cluster SDN sweep that came after it ([#320](https://github.com/luthermonson/go-proxmox/pull/320)) has handle
types throughout: `*SDNController`, `*VNet`, `*VNetSubnet` and friends. The lesson I took: "follow the existing
patterns" is not an instruction. An agent sees one file at a time, and the pattern isn't in that file. "Look at the
upstream path, if it has `/{id}/` write a getter" is an instruction.

### They optimize for the gauge you give them

Coverage was the scoreboard for this work. The scanner that computes it is a set of regexes looking for
`client.Get(ctx, fmt.Sprintf("/nodes/%s/...", ...))`. Agents read the scanner's output after every change. Twice they
changed the *code* so the scanner would count it:

- The Ceph group writing start/stop/restart first built them through a shared `cephServiceAction(action, ...)` helper.
  Then it inlined three copies. The path normalizer turned `fmt.Sprintf("/nodes/%s/ceph/%s", node, "start")` into
  `/nodes/{}/ceph/{}`, which doesn't match `/nodes/{node}/ceph/start`.
- Another group rewrote a `GetWithParams` call into the `url.URL{Path: ...}` form because the scanner understood one and
  not the other.

Both were noted honestly in the PR bodies, which is the only reason I know. Neither is terrible code. But the
duplication in the first one exists only to satisfy a tool. The right response was to fix the tool, and
[#313](https://github.com/luthermonson/go-proxmox/pull/313) taught the scanner about `*WithParams`. That went into
`AGENTS.md` too. If a wrapper is invisible to the scanner, prefer teaching the scanner over contorting the code or
adding to the exclusion list. If you hand an agent a number to move, decide in advance what it's allowed to change to
move it.

### Rules written after the second time

The rest of the file reads like a list of reviews I didn't want to write again:

- **Pointer fields for non-zero Proxmox defaults** ([#271](https://github.com/luthermonson/go-proxmox/pull/271)). The
  decision procedure is spelled out: look up the default, use a pointer if it differs from Go's zero, and regression
  test both ways.
- **Comment every pointer field** with the default it protects. That one exists because of a large wrapper PR
  ([#274](https://github.com/luthermonson/go-proxmox/pull/274)). It did the pointer conversion correctly and deleted
  the `FIXME` comments explaining why. That left bare `*int` fields that look like noise.
  [#268](https://github.com/luthermonson/go-proxmox/pull/268) put the explanations back.
- **Directory-index endpoints are named `Subdirs`** and their helpers keep a `Diridx` suffix, because the scanner keys
  off that suffix. A rename for style would silently drop coverage.
- **Comma-joined fields use the `CSV` type.** That came after `[]string` fields that Proxmox returns as `"a,b,c"`
  broke the SDN round trip.
- **Every release with an `apidiff` incompatible change ships `migration/vX.Y.Z.md`**
  ([#335](https://github.com/luthermonson/go-proxmox/pull/335)). It includes the exact `apidiff` commands and "themed
  sections with before/after code, not a flat dump".

The pattern for all of them is the same. The first time I correct something it's a review comment. The second time
it's a line in `AGENTS.md`.

### The file rots like any other file

`AGENTS.md` is documentation that gets executed, and it goes stale. Right now its "Repo-specific gotchas" section warns
about a `replace` directive in `go.mod` pointing at a local go-diskfs checkout. That `replace` only ever existed in my
working copy. It has never been in a commit, and the file tells every agent to tiptoe around it anyway. It's harmless,
but it's the same failure as a stale comment. The difference is that the reader takes it literally every single time.

### Was it worth it

The numbers above happened, and 100% of the schema is wrapped with a mock and a unit test per endpoint. I would never
have typed that by hand in six weeks of evenings. But the interesting work moved. I typed very little of the code and
a lot of rules. I did a lot of cherry-picking. I spent my review time on shape rather than syntax: is this the right
type, the right receiver, the right place for this mock. Every change still went through a PR and CI. The agents are
fast and literal. So the quality of what comes out is about the quality of what's written down. The cheapest time to
write it down is the second time you see the mistake.

### Links
- [AGENTS.md](https://github.com/luthermonson/go-proxmox/blob/main/AGENTS.md)
- [#264 AGENTS.md and CLAUDE.md](https://github.com/luthermonson/go-proxmox/pull/264), [#312 singleton vs instance rule](https://github.com/luthermonson/go-proxmox/pull/312)
- [#306 combined qemu sweep](https://github.com/luthermonson/go-proxmox/pull/306), [#307 combined Ceph sweep](https://github.com/luthermonson/go-proxmox/pull/307)
- [#308](https://github.com/luthermonson/go-proxmox/pull/308), [#309](https://github.com/luthermonson/go-proxmox/pull/309), [#310](https://github.com/luthermonson/go-proxmox/pull/310), [#311](https://github.com/luthermonson/go-proxmox/pull/311) the refactors
- [agents.md](https://agents.md/)
