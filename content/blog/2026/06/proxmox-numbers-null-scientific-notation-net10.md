---
title: "Proxmox JSON, Part 2: null PIDs, 1.12589990684262e+15 and net10"
description: "Three more ways the Proxmox VE API's JSON doesn't fit Go's encoding/json: a null I fixed and then broke, petabyte storage in scientific notation, and devices past index 9 that were silently dropped."
date: 2026-06-23T20:31:00-07:00
tags: [ proxmox, go, go-proxmox, json ]
categories: [ go, homelab ]
series: [ "Proxmox JSON Types" ]
images:
 - src: img/2022/02/proxmox-logo.png
   alt: "Proxmox VE Logo"
   stretch: "none"
   removeBlur: true
draft: false
---

In 2023 I wrote about [why go-proxmox has `StringOrInt` and friends](/blog/2023/11/proxmox-doesnt-know-what-type-a-number-is/):
Proxmox sends the same number quoted in one place and bare in another, so the client has types that accept both. That
post ended on a pretty confident note. Since then three more shapes of "the JSON is valid and Go still won't take it"
have come in, and one of them is a bug I'd already fixed once and then reintroduced myself.

### null, twice

[#198](https://github.com/luthermonson/go-proxmox/issues/198), June 2025: listing VMs on a node fails with

```
failed to match ^[0-9.]*$: null
```

if any of the VMs is a template. Templates never run, so `/nodes/{node}/qemu` sends `"pid": null` for them, while a
running VM on the same list gets `"pid": "14558"`, a quoted string. `PID` is a `StringOrUint64`, and its unmarshaller
strips quotes and requires what's left to match `^[0-9.]*$`. `null` doesn't.

The embarrassing part is that this worked in May 2023. A contributor's fix for stopped VMs returning `null` had the
unmarshaller treat anything without a digit in it as zero:

```go
numeric := numericRegex.MatchString(str) // `\d`
if !numeric {
	*d = StringOrUint64(0)
	return nil
}
```

Three months later I rewrote all the `StringOr` types to parse floats and truncate, which is the version in the 2023
post. "Non-numeric means zero" became "non-numeric is an error, except the empty string", because I wanted garbage to fail
loudly. `null` is not garbage, it's Proxmox saying the field doesn't apply, and no test fixture had a `null` in it, so
nothing noticed for two years. The fix in [#265](https://github.com/luthermonson/go-proxmox/pull/265) is one condition on
each of the three types:

```go
str := strings.ReplaceAll(string(b), "\"", "")
// Empty string and JSON null both yield the zero value. Proxmox returns
// null for fields that are simply absent on the resource (e.g. PID on a
// stopped VM template — see issue #198).
if str == "" || str == "null" {
	*d = StringOrInt(0)
	return nil
}
```

Note that `encoding/json` calls `UnmarshalJSON` with the literal bytes `null` for a custom type rather than skipping it,
which is why this has to be handled in the type at all.

### A petabyte in scientific notation

[#191](https://github.com/luthermonson/go-proxmox/issues/191): a directory storage backed by a MinIO bucket over a FUSE
mount. The FUSE driver can't tell how big the bucket is, so it reports 1 PiB, and `pvesh` shows exactly that:

```
content   storage        type active avail           enabled shared total           used
backup    minio_backup   dir  1      1125899906842624 1       0      1125899906842624 0
```

The API sends something else:

```
json: cannot unmarshal number 1.12589990684262e+15 into Go struct field Storage.Total of type uint64
```

Somewhere in Proxmox's storage code that value became a Perl floating point number, and a float that big gets printed
with 15 significant digits, in exponent form. It's valid JSON. It's not a valid `uint64`, and `json.Number.Int64()`
won't parse it either:

```
uint64: json: cannot unmarshal number 1.12589990684262e+15 into Go struct field .total of type uint64
json.Number Int64 err: strconv.ParseInt: parsing "1.12589990684262e+15": invalid syntax
json.Number Float64 -> uint64: 1125899906842620
actual bytes (2^50):           1125899906842624
```

The fix ([#237](https://github.com/luthermonson/go-proxmox/pull/237), from a contributor's earlier attempt in #195) is
a custom `UnmarshalJSON` on `Storage` that decodes the numeric fields as `json.Number`, tries `Int64` first and falls back
to `Float64`:

```go
if aux.Total != "" {
	// Try int64 first, then fall back to float64 for scientific notation
	if val, err := aux.Total.Int64(); err == nil {
		s.Total = uint64(val)
	} else if val, err := aux.Total.Float64(); err == nil {
		s.Total = uint64(val)
	}
}
```

Look at the last two lines of that output though. The client now reads 1,125,899,906,842,620 for a storage that is
1,125,899,906,842,624 bytes. Those four bytes were gone before the response left the server, when the number was printed
with 15 digits, and no amount of client-side parsing brings them back. For a fake petabyte nobody cares. It's worth knowing
that Proxmox capacity numbers above 10^15 are approximate.

That fix then broke something else in a way I would not have guessed.
[#241](https://github.com/luthermonson/go-proxmox/issues/241) came from running the
[cluster-api-provider-proxmox](https://github.com/ionos-cloud/cluster-api-provider-proxmox) test suite against `main`:
marshal a `Storage`, unmarshal it again, and `Name` comes back empty. The struct had two fields that map to the same key
once you remember `encoding/json` matches keys case-insensitively:

```go
Name    string `json:"storage"` // marshals as "storage"
Storage string                  // marshals as "Storage"
```

Marshal writes both `"storage":"local"` and `"Storage":""`. On the way back in, the new unmarshaller's `aux` struct has a
single `json:"storage"` field, both keys match it, the later one wins, and it's the empty one. The old `Storage` field was
a leftover from before `Name` existed. [#243](https://github.com/luthermonson/go-proxmox/pull/243) tagged it `json:"-"`,
marked it deprecated, and the round trip works again.

### net10 and the fields that weren't there

[#211](https://github.com/luthermonson/go-proxmox/issues/211): a VM with eleven or more network interfaces only shows the
first ten. `VirtualMachineConfig` had explicit fields for indexed devices, `Net0` through `Net9`, `SCSI0` through
`SCSI30`, `Unused0` through `Unused9` and so on, plus helper maps (`Nets`, `SCSIs`, ...) that a custom `UnmarshalJSON`
filled in by reflecting over those fields. Proxmox allows `net0` to `net31`, `unused0` to `unused255`, and so on, and
`encoding/json` throws away any key that doesn't match a struct field before your code ever gets a look at it. So
`net10` was dropped on the floor, silently.

The fix in [#260](https://github.com/luthermonson/go-proxmox/pull/260) unmarshals the body a second time into
`map[string]json.RawMessage` and routes every `<prefix><digits>` key into the matching map:

```go
routes := vmc.indexedDeviceMaps() // "net" -> &vmc.Nets, "scsi" -> &vmc.SCSIs, ...
for k, v := range raw {
	prefix, ok := indexedDeviceKey(k)
	if !ok {
		continue
	}
	target, ok := routes[prefix]
	if !ok {
		continue
	}
	var s string
	if err := json.Unmarshal(v, &s); err != nil {
		// non-string indexed value — skip rather than fail the whole config
		continue
	}
	if *target == nil {
		*target = make(map[string]string)
	}
	(*target)[k] = s
}
```

The obvious version of this is `strings.HasPrefix(k, "scsi")`, and an earlier PR for the same issue did exactly that.
It's wrong, because `scsihw` (the SCSI controller type) starts with `scsi`, and the VM config also has a plain `numa`
flag next to `numa0`..`numa9`. So the key parser requires the prefix to be followed by digits and *only* digits:

```go
// indexedDeviceKey returns ("net", true) for "net10" and ("", false) for keys
// like "scsihw" or "numa" that share a prefix but lack a pure-digit suffix.
func indexedDeviceKey(k string) (prefix string, ok bool) {
	for i := 0; i < len(k); i++ {
		c := k[i]
		if c >= '0' && c <= '9' {
			if i == 0 {
				return "", false
			}
			// rest of the string must be all digits
			for j := i + 1; j < len(k); j++ {
				if k[j] < '0' || k[j] > '9' {
					return "", false
				}
			}
			return k[:i], true
		}
	}
	return "", false
}
```

There are unit tests for exactly those two collisions. #260 kept the explicit `Net0..Net9` fields populated for
compatibility, and v0.7.0 ([#261](https://github.com/luthermonson/go-proxmox/pull/261)) removed them, so the maps are now
the only shape: `cfg.Nets["net10"]`. It's one of the bigger sections of the v0.7.0 migration guide because a lot of code
reads `cfg.Net0`.

### The pattern

All three of these are the same bug as the 2023 post wearing a different hat. The JSON is fine, the Go type was written
from the docs or from one sample response, and the real API sends a shape nobody put in a test fixture: a `null`, a
float, an eleventh NIC. Each fix is small. Finding them is the hard part, and it's why I'm now more interested in testing
against recorded responses from a real cluster than in typing more fixtures by hand.

### Links
- [Part 1: Proxmox Doesn't Know What Type a Number Is](/blog/2023/11/proxmox-doesnt-know-what-type-a-number-is/)
- [#198 null PID](https://github.com/luthermonson/go-proxmox/issues/198) and [#265](https://github.com/luthermonson/go-proxmox/pull/265)
- [#191 large storage](https://github.com/luthermonson/go-proxmox/issues/191), [#237](https://github.com/luthermonson/go-proxmox/pull/237), [#241](https://github.com/luthermonson/go-proxmox/issues/241), [#243](https://github.com/luthermonson/go-proxmox/pull/243)
- [#211 indexed devices](https://github.com/luthermonson/go-proxmox/issues/211), [#260](https://github.com/luthermonson/go-proxmox/pull/260), [#261](https://github.com/luthermonson/go-proxmox/pull/261)
- [encoding/json Unmarshal (case-insensitive key matching)](https://pkg.go.dev/encoding/json#Unmarshal)
