---
title: "Your Zero Value Just Turned Off My TLS Verification: omitempty, Go Defaults and Proxmox"
description: "A plain Go field can't tell unset from false. In go-proxmox the fix for one omitempty bug created the opposite bug, until an audit against the Proxmox schema found 75 fields and v0.7.0 moved them to pointers."
date: 2026-06-13T19:52:00-07:00
tags: [ proxmox, go, go-proxmox, json ]
categories: [ go, homelab ]
images:
 - src: img/2026/06/omitempty-zero-values-proxmox-defaults-table.svg
   alt: "Table showing how verify-certificates was sent by go-proxmox before #210, between v0.2.3 and v0.6.x, and in v0.7.0"
   stretch: "none"
draft: false
---

Every Go developer knows `omitempty` drops zero values. What takes longer to sink in is that a plain `bool` only has two
states and a caller filling in an options struct means one of three: "set it true", "set it false", and "I didn't touch
this, do whatever the server does by default". With `omitempty` you lose "false". Without it you lose "didn't touch it".
In [go-proxmox](https://github.com/luthermonson/go-proxmox) we managed to ship both bugs, in that order, and the second one
quietly turned off TLS certificate verification for anyone downloading ISOs through the API.

![Three versions of one field](/img/2026/06/omitempty-zero-values-proxmox-defaults-table.svg)

### Bug one: you can't say no

[#209](https://github.com/luthermonson/go-proxmox/issues/209) from @kernel-sanders was a clean report. Create an API
token with `Privsep: false` because you want it to carry the user's permissions, and Proxmox creates a token *with*
privilege separation:

```go
type Token struct {
	TokenID string    `json:"tokenid,omitempty"`
	Comment string    `json:"comment,omitempty"`
	Expire  int       `json:"expire,omitempty"`
	Privsep IntOrBool `json:"privsep,omitempty"`
}
```

`false` is the zero value, `omitempty` drops it, Proxmox never sees the field and applies its default, which is `1`.
Same story for disabling a user, creating an ACL that doesn't propagate, and a handful of others. The issue listed them
all.

### Bug two: the fix

[#210](https://github.com/luthermonson/go-proxmox/pull/210), also from @kernel-sanders, took option one from the issue, remove `omitempty` from the
affected booleans, and I merged it in August 2025. It went out in v0.2.3:

```diff
 type StorageDownloadURLOptions struct {
 	...
-	VerifyCertificates IntOrBool `json:"verify-certificates,omitempty"`
+	VerifyCertificates IntOrBool `json:"verify-certificates"`
 }
 ...
 type VirtualMachineBackupOptions struct {
 	...
-	Remove             bool                                   `json:"remove,omitempty"`
+	Remove             bool                                   `json:"remove"`
 	...
-	StdExcludes        bool                                   `json:"stdexcludes,omitempty"`
+	StdExcludes        bool                                   `json:"stdexcludes"`
```

Now `false` can be sent. It's also sent when you didn't set anything, and every one of those fields defaults to `1` in
Proxmox. Here's what actually goes over the wire, from a small program using the same type and tags:

```
pre-#210 Token{Privsep: false}                   {"tokenid":"ci"}
v0.2.3 StorageDownloadURLOptions{} (unset)       {"url":"https://example.com/x.iso","verify-certificates":0}
v0.7.0 StorageDownloadURLOptions{} (unset)       {"url":"https://example.com/x.iso"}
v0.7.0 VerifyCertificates: Ptr(false)            {"url":"https://example.com/x.iso","verify-certificates":0}
```

So from v0.2.3 until v0.7.0, `node.StorageDownloadURL` with a struct that didn't mention `VerifyCertificates` told
Proxmox to download the file without checking the server's certificate. Every `node.Vzdump` with default options sent
`remove: false` and `stdexcludes: false`, which Proxmox reads as 0, so backups stopped pruning old archives per the retention policy and started including
`/tmp` and log files. Nobody filed an issue for either one, which is the scary part. These don't fail, they just do the
wrong thing.

(One more trap while I'm here: `IntOrBool`'s `MarshalJSON` has a pointer receiver. Marshal a struct holding one *by value*
and the method isn't in the method set, so you get `"verify-certificates":false` instead of `0`. The client always marshals
through a pointer so it doesn't bite there, but it surprised me when I wrote the test program.)

### The general case

The underlying issue was already open. [#199](https://github.com/luthermonson/go-proxmox/issues/199) from @xortim, June
2025: `ContainerConfig.Console` is an `IntOrBool` with `omitempty`, Proxmox defaults it to `1`, and the more general
problem is any field where Go's zero value differs from Proxmox's default. And
[#178](https://github.com/luthermonson/go-proxmox/issues/178) pointed out that `VirtualMachineConfig.OnBoot` is an `int`
when the schema says boolean. Two classes of bug, both of the form "the Go type doesn't match what the schema says", and
both had been getting fixed one report at a time.

So instead of waiting for reports I wrote an audit ([#266](https://github.com/luthermonson/go-proxmox/pull/266)). It
loads the Proxmox API schema, walks `types.go` with `go/parser`, and for each config or options struct joins every field's
JSON tag against the parameters of the endpoint that struct is sent to, preferring the create (`POST`) endpoint because
that's where Proxmox documents defaults. Indexed parameters like `net[n]` get matched to `net0` and friends. For each
match it compares the Go zero value to the schema default and the Go type to the schema type, and writes a table:

| Field | json tag | Go type | PVE type | Go zero | PVE default | Recommended fix |
|---|---|---|---|---|---|---|
| `Console` | `console` | `IntOrBool` | `boolean` | `0` | `1` | `*IntOrBool` |
| `Memory` | `memory` | `int` | `integer` | `0` | `512` | `*int` |
| `KVM` | `kvm` | `int` | `boolean` | `0` | `1` | `*IntOrBool` |
| `VerifyCertificates` | `verify-certificates` | `IntOrBool` | `boolean` | `0` | `1` | `*IntOrBool` |
| `Remove` | `remove` | `bool` | `boolean` | `false` | `1` | `*IntOrBool` |

It flagged **75 fields**: 38 where the default differs on a non-boolean (needs `*T`), 9 where the default differs and the
schema says boolean (`*IntOrBool`), and 28 that are only the wrong type (`IntOrBool`, no pointer needed because `0` is
the default anyway). That PR only added the tool, the report and a `FIXME` on each field, so the breaking change could
land as one migration instead of a trickle.

One finding is worth being precise about. The audit's note on `FirewallNodeOption.Enable` said an unset value would
silently disable the node firewall. With that field still a `bool` with `omitempty` on a `PUT`, unset is omitted and
the firewall stays as it is, so it couldn't actually disable it. What it couldn't do was let you turn the firewall *off*,
which is bug one again. Both are fixed by the same change, but a tool's description of the failure is a hypothesis until
you've traced the code path.

### The fix: pointers, and a helper so pointers aren't miserable

v0.7.0 changed about 30 fields from `T` to `*T` and about 25 from `bool`/`int` to `IntOrBool`. `nil` means "let Proxmox
decide", non-nil means "send this". The cost is that you can't write `Cores: 4` in a struct literal anymore, so there's a
one-line generic helper:

```go
// Ptr returns a pointer to v. It exists so callers can construct optional
// fields on config/options structs inline without a temporary local variable.
func Ptr[T any](v T) *T { return &v }
```

```go
cfg := &proxmox.VirtualMachineConfig{
	Memory:   proxmox.Ptr(proxmox.StringOrInt(4096)),
	Cores:    proxmox.Ptr(4),
	OSType:   proxmox.Ptr("l26"),
	CPUUnits: proxmox.Ptr(1024),
}
```

And the regression tests check both directions, because only checking one is how we got here. Unset must be absent,
explicit false or zero must be present:

```go
func TestContainerConfig_DefaultsOmittedWhenUnset(t *testing.T) {
	// Bare struct — none of the pointer-typed fields should appear.
	m := marshalToMap(t, ContainerConfig{})
	for _, k := range []string{"arch", "cmode", "console", "cpuunits", "memory", "swap", "tty"} {
		_, present := m[k]
		assert.Falsef(t, present, "%q should be omitted when unset so PVE applies its default", k)
	}
}

func TestContainerConfig_ExplicitValuesAppearOnTheWire(t *testing.T) {
	cfg := ContainerConfig{
		Arch:     Ptr("amd64"),
		CMode:    Ptr("tty"),
		Console:  Ptr(IntOrBool(false)), // explicit disable — must reach PVE as 0, not be silently swallowed
		CPUUnits: Ptr(1024),
		Memory:   Ptr(512),
		Swap:     Ptr(0), // explicit no-swap — must reach PVE as 0, not be omitted
		TTY:      Ptr(2),
	}
	m := marshalToMap(t, cfg)
	assert.Equal(t, float64(0), m["console"]) // IntOrBool false marshals to 0
	assert.Equal(t, float64(0), m["swap"])
	// ...
}
```

`Swap` is a nice example of why "just use `omitempty`" doesn't work even for integers: the Proxmox default is 512, and
`0` is a perfectly reasonable value meaning "no swap" that you need to be able to send.

### Making it stick

A one-time sweep doesn't help if the next endpoint wrapper reintroduces it, and in v0.7 a lot of new wrappers were being
written. So the rule went into `AGENTS.md` ([#271](https://github.com/luthermonson/go-proxmox/pull/271)) as a decision
procedure: look up the field's default in the API viewer, use a pointer if it differs from Go's zero, leave it unboxed if
it matches, and add a test that marshals the struct with the field unset. The one-line version is the question at the
end of that section: *if the user never touches this field, will an unintended value reach Proxmox?*

There was one more lesson. The sweep that converted the fields stripped the `FIXME` comments and left nothing in their
place, so six months from now someone would see `CPUUnits *int` and "simplify" it back. [#268](https://github.com/luthermonson/go-proxmox/pull/268)
put a comment on every pointerized field saying which default it protects:

```go
// Remove — PVE default 1 (remove old backups per retention policy).
// Schema is boolean; unset on the wire keeps pruning enabled. See #178 + #199.
Remove *IntOrBool `json:"remove,omitempty"`
Script string     `json:"script,omitempty"`
// StdExcludes — PVE default 1 (skip /tmp, /var/log, etc.). Unset on
// the wire keeps those default exclusions. See #178 + #199.
StdExcludes *IntOrBool `json:"stdexcludes,omitempty"`
```

If you're on v0.2.3 through v0.6.x and you call `StorageDownloadURL` or `Vzdump`, set those fields explicitly or upgrade.
The full list of changed fields is in [migration/v0.7.0.md](https://github.com/luthermonson/go-proxmox/blob/main/migration/v0.7.0.md).

### Links
- [#209](https://github.com/luthermonson/go-proxmox/issues/209), [#210](https://github.com/luthermonson/go-proxmox/pull/210), [#199](https://github.com/luthermonson/go-proxmox/issues/199), [#178](https://github.com/luthermonson/go-proxmox/issues/178)
- [#266 the audit](https://github.com/luthermonson/go-proxmox/pull/266), [#268 rationale comments](https://github.com/luthermonson/go-proxmox/pull/268), [#271 AGENTS.md rule](https://github.com/luthermonson/go-proxmox/pull/271), [#274 cluster wrappers with the pointer rule](https://github.com/luthermonson/go-proxmox/pull/274)
- [defaults_omit_test.go](https://github.com/luthermonson/go-proxmox/blob/main/defaults_omit_test.go)
- [golang/go#13284, the long-running omitempty discussion](https://github.com/golang/go/issues/13284)
