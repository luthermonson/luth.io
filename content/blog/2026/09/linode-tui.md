---
title: "Building k9s for Linode With Bubble Tea"
description: "A k9s-style terminal app for managing Linode: search for any resource, browse it, and jump straight into k9s, a database shell or SSH."
date: 2026-09-09T21:22:00-07:00
tags:
 - go
 - bubbletea
 - tui
 - linode
 - kubernetes
 - k9s
categories: [ go, tools ]
images:
 - src: img/2026/09/linode-tui-linodes.svg
   alt: "linode-tui showing the Linodes view with the command bar open and tab completion suggesting lke"
   stretch: "none"
draft: false
---

A lot of my day is spent in [k9s](https://k9scli.io/), and when I need something from the Linode side of the house (a node that won't
boot, a volume nobody cleaned up, the kubeconfig for a cluster I just made) I'm back in a browser tab or typing `linode-cli` incantations.
I wanted the k9s workflow for the Linode API itself: `:` to jump to a resource type, `/` to filter, single-key actions on the selected row,
and Enter to go deeper. That turned into [linode-tui](https://github.com/luthermonson/linode-tui), a Go TUI built on
[Bubble Tea](https://github.com/charmbracelet/bubbletea), [linodego](https://github.com/linode/linodego) and urfave/cli v3.

It started as a private repo in May, got restructured for open source in June (flattened `internal/`, slimmed the CLI down, moved to
linodego v2), and went public with v0.1.0 at the end of August. Everything here is as of [v0.2.1](https://github.com/luthermonson/linode-tui/releases/tag/v0.2.1).

![linode-tui Linodes view with the command bar open](/img/2026/09/linode-tui-linodes.svg)

### One list view, thirteen resources

There are 13 resource views: Linodes, Volumes, NodeBalancers, Firewalls, Images, Domains, LKE clusters, VPCs, Placement Groups,
StackScripts, Object Storage, databases and Events. Writing 13 Bubble Tea models by hand would have been a lot of copy and paste, so
there's one generic `listView[T]` and every resource is a set of options handed to it. This is most of the Linodes view:

```go
func init() {
	Register("instances", []string{"linodes", "inst", "li"}, newInstances)
}

func newInstances(d Deps) View {
	return newListView(listOpts[linodego.Instance]{
		Deps:  d,
		Title: "Linodes",
		Columns: []Col{
			{Title: "ID", Width: 10, MinWidth: 6, Priority: PriPinned},
			{Title: "LABEL", Width: 28, MinWidth: 12, Priority: PriPinned, Flex: true},
			{Title: "REGION", Width: 14, MinWidth: 8, Priority: PriMed},
			{Title: "TYPE", Width: 18, MinWidth: 10, Priority: PriMed},
			{Title: "STATUS", Width: 12, MinWidth: 8, Priority: PriHigh},
			{Title: "IPv4", Width: 16, MinWidth: 12, Priority: PriLow},
			{Title: "TAGS", Width: 24, MinWidth: 8, Priority: PriLowest},
		},
		Lister: func(ctx context.Context, c *linode.Client) ([]linodego.Instance, error) {
			return c.Raw().ListInstances(ctx, nil)
		},
		// Rower, Matcher, Sort, Actions, KeyHandlers ...
	})
}
```

The list view owns polling, filtering, selection, bookmarks, the confirm modal and the status line. A view supplies a lister, a function
that turns one item into a table row, a matcher for `/`, and its actions. Reboot, for example, is just a key, a label, a prompt, and a
`Run` that calls linodego:

```go
{
	Key:    "R",
	Label:  "reboot",
	Prompt: func(it linodego.Instance) string { return fmt.Sprintf("Reboot %s (id %d)?", it.Label, it.ID) },
	Run: func(ctx context.Context, c *linode.Client, it linodego.Instance) error {
		return c.Raw().RebootInstance(ctx, it.ID, linodego.InstanceRebootOptions{})
	},
},
```

`Register` puts the view in a registry with its aliases, and the command bar resolves against that: exact names and aliases win, then
prefix matches. That's why `:lk` in the screenshot above offers `lke` on Tab. Drill-in views that only make sense with a parent (a
NodeBalancer's configs, a Domain's records) register with `RegisterChild` so they never show up as top-level verbs.

### Drill-ins: hand the terminal to the real tool

I didn't want to rebuild k9s inside linode-tui. When you're on an LKE cluster and press `c`, it fetches the kubeconfig from the API,
writes it to a 0600 temp file, and runs k9s against it. For DBaaS it builds a connection URL and runs
[lazysql](https://github.com/jorgerojas26/lazysql). On a Linode, `c` is `ssh root@<ip>` and `C` is [lish](https://techdocs.akamai.com/cloud-computing/docs/access-your-system-console-using-lish)
through Linode's bastion, which works even when the box has no public IP.

![linode-tui LKE clusters view](/img/2026/09/linode-tui-lke.svg)

Bubble Tea has exactly the primitive for this. `tea.ExecProcess` releases the terminal, runs the child process attached to it, and
resumes the TUI when the child exits:

```go
case config.ModeTUI, "":
	c := exec.CommandContext(ctx, bin, args...)
	c.Stdin = os.Stdin
	c.Stdout = os.Stdout
	c.Stderr = os.Stderr
	prepare(c)
	return tea.ExecProcess(c, func(err error) tea.Msg { return ExitMsg{Kind: kind, Err: err} }), nil
```

Every tool is configurable (`tools.kubernetes.exec`, `tools.mysql.exec` and so on), so you can point it at Lens or MySQL Workbench with
`mode: gui` and it launches detached instead. If k9s or lazysql isn't installed, linode-tui offers to download a pinned release, verify its
checksum, and put it in an install dir you pick on first run.

The kubeconfig temp file was also the subject of the first bug report from somebody other than me ([#2](https://github.com/luthermonson/linode-tui/issues/2)): `c` from the
LKE list worked, but `c` from the cluster's detail page launched k9s and immediately died with `Watcher failed for contexts -- stat ...`.
The detail view was cleaning up the kubeconfig like this:

```go
return tea.Sequence(exec, func() tea.Msg {
    if cleanup != nil { cleanup() }   // runs immediately
    return lkeDetailDrillDoneMsg{}
})
```

`tea.Sequence` runs the commands in order, but `ExecProcess`'s command isn't blocking: it just returns a message asking
the program to run the process. So the cleanup closure ran as soon as that message was handed back, and the file was gone
before k9s had even started. The list view didn't have the bug because its closure only returned a done message and ran the cleanup when that message came back through `Update`, which happens after k9s exits.
[#3](https://github.com/luthermonson/linode-tui/pull/3) moved the detail view to the same pattern, and
[#16](https://github.com/luthermonson/linode-tui/pull/16) collapsed both paths into one shared function so it can't drift again:

```go
func RunDrillIn(deps Deps, msg DrillInMsg, done func(cleanup func()) tea.Msg) (tea.Cmd, error) {
	runner := tools.New(deps.Cfg)
	exec, err := runner.RunWithEnv(context.Background(), msg.Tool, msg.Vars, msg.Env)
	if err != nil {
		if missing, ok := tools.IsToolMissing(err); ok {
			return func() tea.Msg { return InstallNeededMsg{Kind: missing.Kind, Drill: msg} }, nil
		}
		if msg.Cleanup != nil {
			msg.Cleanup()
		}
		return nil, err
	}
	cleanup := msg.Cleanup
	return tea.Sequence(exec, func() tea.Msg { return done(cleanup) }), nil
}
```

If you write Bubble Tea apps that shell out, that one is worth remembering.

### Hardening for release

A tool that holds API tokens and has a `:clear-account` command that deletes everything in an account deserves a hard look before
other people run it. Most of what went into v0.2.0 was that pass, done as a handful of large PRs.

**Config persistence ([#12](https://github.com/luthermonson/linode-tui/pull/12))**. The worst one: a `--token` flag or `LINODE_TOKEN` got
stored in memory as a synthetic `__cli__` account, and the first incidental `Save()` (starring a row, installing a tool) wrote your
environment token into `config.yaml` in plain text. Save now marshals a scrubbed copy. It also stopped truncating the file in place,
because a crash between truncate and write lost every account:

```go
// Write-then-rename: os.WriteFile truncates in place, so a crash (or a
// full disk) between truncate and write left every account and token
// gone. Rename is atomic on POSIX and on Windows (MoveFileEx).
tmp := c.path + ".tmp"
if err := os.WriteFile(tmp, data, 0o600); err != nil {
	return err
}
if err := os.Rename(tmp, c.path); err != nil {
	_ = os.Remove(tmp)
	return err
}
```

Saves are behind a mutex (background tool installs save from goroutines), the config dir went from 0755 to 0700, and `refresh: 1ms`
gets clamped to 1s instead of becoming a self-inflicted 429 generator. The same PR fixed `:clear-account`, which only guarded on the
*name* of the local account. `accounts: {dev: {token: <prod token>}}` is one bad paste away, so it now resolves `/profile` and refuses
to delete anything if the token belongs to a different user than the one you confirmed. It also deletes LKE clusters before instances
now. Otherwise the node pools recreate the workers you just deleted.

**App state ([#13](https://github.com/luthermonson/linode-tui/pull/13))**. Opening a modal swallowed the list's tick message, which
quietly ended that pane's refresh loop for the rest of the session. Switching accounts rebuilt only the focused pane, so split panes kept
polling the old account with the old token. A bare `:export` panicked on `args[1:]` of an empty slice. A new test now dispatches every
command-bar verb and alias and fails if any of them comes back as `unknown command`. That test is what found the `:export` panic.

**Responsive columns ([#16](https://github.com/luthermonson/linode-tui/pull/16))**. The Linodes table wants about 140 cells. On a narrower
terminal the bubbles table wrapped every row into garbage. Columns now carry a width, a minimum and a priority. The list drops the
lowest-priority column until the minimums fit, then spends the leftover space growing the survivors and the `Flex` column. ID and LABEL
are pinned and never dropped, and the status line tells you `»3 cols hidden — widen terminal`.

**The yaml tag ([#23](https://github.com/luthermonson/linode-tui/pull/23))**. #13 made `audit_retention_days: 0` mean "keep forever",
like the docs always said. Then the review caught that the field still had `omitempty`, so an explicit `0` got dropped on save, and the next
load re-seeded the 90-day default. Users who opted out of pruning would have been opted back in by any unrelated save. The fix was deleting
`,omitempty` and adding a Save→Load round-trip test. `omitempty` on an int where zero means something is a bug that's easy to write.

### Tokens

The config doesn't have to hold a token. A literal `token:` works but isn't recommended. Accounts can point at a 1Password reference and linode-tui shells out to `op` to resolve it:

```yaml
default_account: dev
accounts:
  dev:
    op_ref: "op://Private/linode-dev/token"
  e2e:
    op_ref: "op://Private/linode-test/token"
```

`:account e2e` switches live. There's also an audit log of every mutating action (`~/.cache/linode-tui/audit.log`), which is what powers
`:undo` and the `ctrl+y` replay.

### How the screenshots were made

I didn't want a real account in a blog post, so the screenshots run against a fake API. linodego honors `LINODE_URL`, so I wrote a
small Go server that returns canned JSON for `/v4/profile`, `/v4/linode/instances` and `/v4/lke/clusters` (IPs from the documentation
ranges) and ran linode-tui v0.2.1 in a 118x16 tmux pane with `LINODE_URL=http://127.0.0.1:18080`, a dummy token, and an empty `HOME`.
The server log shows the polling doing what it should, a list refetch every 2 seconds:

```
GET /v4/linode/instances
GET /v4/profile
GET /v4/linode/instances
GET /v4/linode/instances
GET /v4/linode/instances
GET /v4/lke/clusters
GET /v4/lke/clusters
```

The images are `tmux capture-pane -e` output converted to SVG with the colors exactly as tmux reported them.

### Install

```bash
go install github.com/luthermonson/linode-tui/cmd/linode-tui@latest
export LINODE_TOKEN=...
linode-tui
```

Or grab a binary from the releases page (GoReleaser builds Linux and macOS on amd64 and arm64, and Windows on amd64). If you already live in
`linode-cli`, the README has a five-line shell function that routes `linode tui` to this and everything else to the real CLI.
A Homebrew tap is next on the list.

### Links
- [linode-tui](https://github.com/luthermonson/linode-tui)
- [docs/USAGE.md](https://github.com/luthermonson/linode-tui/blob/main/docs/USAGE.md)
- [k9s](https://k9scli.io/)
- [Bubble Tea](https://github.com/charmbracelet/bubbletea)
- [linodego](https://github.com/linode/linodego)
- [lazysql](https://github.com/jorgerojas26/lazysql)
