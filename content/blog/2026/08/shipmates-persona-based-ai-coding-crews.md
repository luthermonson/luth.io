---
title: "shipmates: Persona-Based AI Coding Crews That Remember"
description: "A Go CLI and daemon that runs crews of AI coding agents across machines, with per-project memory, a shared task graph, voice control and a real permission model."
date: 2026-08-21T21:18:00-07:00
tags: [ go, ai-agents, claude-code, shipmates, open-source ]
categories: [ ai-agents, go ]
series: [ "shipmates" ]
images:
 - src: img/2026/08/shipmates-persona-based-ai-coding-crews-cover.svg
   alt: "shipmates hierarchy: Admiral, Fleet Command, ships, captains and mates"
   stretch: "none"
draft: false
---

Since June most of my evenings have gone into [shipmates](https://github.com/luthermonson/shipmates), a Go CLI and daemon
for running crews of AI coding agents. I tagged [v0.5.0](https://github.com/luthermonson/shipmates/releases/tag/v0.5.0)
on August 17th, two months and about 240 commits after the first architecture doc, so it's a good time to write down what
it is and why it exists.

### The problem

I run a lot of agent sessions. On a normal night this workstation has four or five projects going at once, each with a
few Claude Code sessions doing different jobs: one reviewing, one writing tests, one chasing a bug. Two things kept
annoying me.

First, every session starts from zero. The "security reviewer" prompt you found on the internet is a headless expert:
a role description and a tool allowlist. It doesn't remember that you rejected a pattern three weeks ago, or which
constant is load-bearing for reasons the code doesn't explain. You end up re-explaining the project in every prompt, and
the reviewer gives you generic best-practice advice you mostly already knew.

Second, coordination. Several projects, more than one machine, and I was the message bus between all of them, alt-tabbing through terminals to see who was waiting on a permission prompt.

### The pitch

Each persona gets a per-project memory directory. It reads it at session start and writes to it as it learns. Two weeks
in, the architect reviewing a PR doesn't just see the diff, it sees the diff against a remembered project: "this
re-introduces the pattern we rejected in #45, has anything changed?"

Then Fleet Command adds the second dimension: coordinate ships on multiple machines from one browser, dispatch work across
them, and talk to the whole fleet by voice.

### The naval bit

I leaned into the metaphor because it maps cleanly onto roles:

- **Admiral**: me, giving direction.
- **Fleet Command**: one coordinator node that hosts the web UI, the voice loop and the shared task graph.
- **Commodore**: the AI officer at Fleet Command that turns spoken orders into fleet tool calls (list captains, tell a
  captain, assign work, approve a pending request).
- **Ship**: one per project. A machine can run several.
- **Captain**: the per-ship lead persona, coordinating that ship's crew.
- **Mates**: the individual agents, `architect`, `security`, `frontend`, `backend`, `tester` and so on.

### Single ship

You don't need any of the fleet parts. Install the binary (the persona catalog is embedded), then in a project:

```bash
cd my-project
shipmates init --crew captain,architect,security,frontend,backend,tester
```

That vendors persona files into `.claude/agents/<name>.md`, which Claude Code reads natively, seeds each persona's
memory under `.shipmates/memory/<name>/`, drops a `policy.yaml` overlay for the personas that ship one, and wires a `SessionStart` hook into
`.claude/settings.json`. The hook shells out to `shipmates hook load-memory`, which reads the persona's memory and
injects it as session context. That's deliberate: memory loading shouldn't depend on the model deciding to go read a
file first.

Then you talk to the crew three ways:

```bash
# one-shot delegation; resumes the persona's session, so it remembers
shipmates ask security "review the diff for auth regressions"

# a live crew member you can message while it works
shipmates tell security "double-check PR 10"
shipmates feed

# an interactive session as the persona
shipmates open captain
```

There's also `shipmates bridge`, a tabbed terminal UI (Bubble Tea) over every live mate on the machine, with a live PTY
per tab and one-key approvals.

### The fleet

Ships open **outbound** websocket tunnels to Fleet Command, so nothing on the ship side is exposed and it works from
behind NAT. The tunnels are [rancher/remotedialer](https://github.com/rancher/remotedialer), which I know well from my
Rancher days. (It also produced the first real bug story of the project, more on that soon.) Fleet Command can sit
behind cloudflared or any TLS proxy.

Coordination between ships goes through a shared task graph, [beads](https://github.com/gastownhall/beads), a
Dolt-backed SQL database that rides on hidden git refs inside the project's own repo. The fleet's work is versioned,
merge-safe, and stored somewhere I already trust instead of in another service.

In the browser you get every captain in a sidebar, a live terminal per mate, an event stream per captain, the task
graph, and an approval queue. You can drag a screenshot in and it lands in the target ship's inbox, and the mate reads
it with Claude Code's own multimodal Read tool.

Voice is the part that makes it feel like a fleet. Press the mic, speak an order, and it goes speech-to-text (whisper.cpp
or anything OpenAI-compatible), into an LLM backend that plays the Commodore and calls fleet tools, then out through
text-to-speech (Kokoro, Edge voices, or OpenAI-compatible). Every stage is a pluggable endpoint, so the whole voice stack
can run on a local GPU.

### Permissions, the Brig, and containment

Agents running shell commands on my machines need a real permission model, not vibes. shipmates evaluates every tool
call through layered rules in Claude Code's own syntax: a fleet-wide deny list pushed from Fleet Command (which binds
even personas running in bypass mode), per-persona policy, project settings, user settings. Deny beats ask beats allow.

On top of that sits the Brig: fifteen "Ship's Articles" (no piped execution, no self-escalation, stay aboard, and so on)
bound to every persona the moment it's installed, ten of them compiled into deny and ask rules, plus `shipmates freeze` to stop every write-class tool
call across a ship without killing sessions, and an append-only denial log.

And every agent process runs under containment: Windows Job Objects, Unix process groups, and an RSS/CPU watchdog, so a
turn that ends takes its whole process tree with it and an optional memory or CPU budget gets enforced.

### What v0.5.0 added

The v0.5.0 release was big:

- **Voyages**: `plan`, `commission` and `sail`. A multi-task plan executed by a Go orchestrator. Dependencies, retries,
  model-tier escalation and state live in code, and the model is spent only on the tasks.
- **Multi-runtime**: `--runtime claude|codex|openai` across the command surface. Claude Code is still the main one, but
  Codex and any OpenAI-compatible endpoint work too.
- **The Brig** landed in this release.
- **A security pass** that changed where several settings are allowed to live. Persona execution config (which binary to
  spawn, whether to skip permissions) moved out of the checkout into `~/.shipmates/personas.yaml`, because anything in the
  repo arrives with `git clone` and is attacker-controlled on an unreviewed repo. The captain's local API now requires a
  token, plaintext fleet URLs to non-loopback hosts are refused, and the permission gate stopped trusting the first word of
  a shell command.

If you run a GitHub-routed crew on a public repo, everything an outsider can write (issue titles, bodies, comments, diffs,
branch names) is treated as data to evaluate, never instructions to follow.

### How it's built

It's all Go (1.26), a single binary for macOS, Linux and Windows on amd64 and arm64, released with goreleaser. CI runs on
my own [ephemerd](/blog/2026/08/ephemerd-one-daemon-every-os/) runners across linux-x64, linux-arm64, macOS arm64 and
Windows x64, with a dedicated race-detector job. Development happens on Windows first, which has turned out to be a great
way to find cross-platform bugs, and a terrible way to find the ones that only show up on Linux.

One install caveat for now: `go install github.com/luthermonson/shipmates@latest` doesn't work, because shipmates pins a
fork of remotedialer with a `replace` directive. Grab a [release binary](https://github.com/luthermonson/shipmates/releases)
or `git clone` and `go build -o shipmates .`.

### What's next

I've got a backlog of posts out of the last two months, mostly about the things that broke. Next up is the data race in
remotedialer that my CI found and how I managed to pin only two-thirds of my own fix. After that, NTFS and atomic file
replacement, and a fun one about `os/exec`.

### Links

- [github.com/luthermonson/shipmates](https://github.com/luthermonson/shipmates) (MIT)
- [v0.5.0 release notes](https://github.com/luthermonson/shipmates/releases/tag/v0.5.0)
- [docs/PHILOSOPHY.md](https://github.com/luthermonson/shipmates/blob/main/docs/PHILOSOPHY.md), the longer "why"
- [docs/security.md](https://github.com/luthermonson/shipmates/blob/main/docs/security.md)
- [beads](https://github.com/gastownhall/beads)
