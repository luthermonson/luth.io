---
title: "Proxmox Doesn't Know What Type a Number Is"
description: "The Proxmox VE API returns the same field as 1234 in one place and \"1234\" in another. Why go-proxmox has StringOrInt, StringOrUint64, StringOrFloat64 and IntOrBool instead of plain Go types."
date: 2023-11-25T20:05:00-07:00
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

Every couple of months somebody opens an issue on [go-proxmox](https://github.com/luthermonson/go-proxmox) with a title
that starts with `json: cannot unmarshal string into Go struct field`. The most recent one was fixed last week, and it's the same
root cause every time: the Proxmox VE API is backed by Perl, Perl doesn't really care whether a scalar is a number or a
string, and the JSON encoder on the other end writes whatever the scalar happens to be at the time. So a field that is
documented as an integer comes back as `2048` from one endpoint and `"2048"` from another, and sometimes from the same
endpoint depending on how the value got into the config.

Go's `encoding/json` does care, a lot. Here's how the client deals with it.

### The cpulimit report

The cleanest example is [#66](https://github.com/luthermonson/go-proxmox/issues/66) and its follow up
[#69](https://github.com/luthermonson/go-proxmox/issues/69). `CPULimit` was an `int` in `VirtualMachineConfig`, which is
wrong on its face because the Proxmox docs describe it as a float (`1.0` is 100% of a core, `0.4` is 40%). So it became
a `float32`, and then the reporter came back with the actual response from
`/nodes/{node}/qemu/{vmid}/config`:

```json
{
  "data": {
    "cores": 2,
    "memory": 2048,
    "cpulimit": "0.4",
    "scsihw": "virtio-scsi-pci"
  }
}
```

(trimmed, the real payload has the rest of the VM config in it). `cores` and `memory` are numbers. `cpulimit` is a
string. Same object, same endpoint.

### Why not `,string`

The reporter's first suggestion was the `,string` struct tag option, which tells `encoding/json` the value is a number
wrapped in quotes. That handles exactly one shape. I tried it and it was a mess, so here is the comparison as a little
program you can run, three inputs against a plain `float64`, a `float64` with `,string`, and the type we ended up with:

```go
type plain struct {
	CPULimit float64 `json:"cpulimit"`
}
type withTag struct {
	CPULimit float64 `json:"cpulimit,string"`
}
type withOr struct {
	CPULimit StringOrFloat64 `json:"cpulimit"`
}
```

```
{"cpulimit":"0.4"}   float64: 0      err=json: cannot unmarshal string into Go struct field plain.cpulimit of type float64
                     ,string: 0.4    err=<nil>
                     StringOrFloat64: 0.4    err=<nil>
{"cpulimit":0.4}     float64: 0.4    err=<nil>
                     ,string: 0      err=json: invalid use of ,string struct tag, trying to unmarshal unquoted value into float64
                     StringOrFloat64: 0.4    err=<nil>
{"cpulimit":""}      float64: 0      err=json: cannot unmarshal string into Go struct field plain.cpulimit of type float64
                     ,string: 0      err=json: invalid use of ,string struct tag, trying to unmarshal "" into float64
                     StringOrFloat64: 0      err=<nil>
```

(Go 1.21.4.) `,string` fixes the quoted case and breaks the unquoted one, and it can't cope with an empty string, which
Proxmox also sends. If the same struct is used for two endpoints that disagree, and in this client a lot of structs are,
a tag can't help you. You need a type that accepts both.

### The StringOr types

The idea goes back to the first version of the client: a named numeric type with its own `UnmarshalJSON` that strips any
quotes and parses what's left. One type per Go numeric kind we need. The float version I wrote for #69:

```go
var (
	isFloat = regexp.MustCompile(`^[0-9.]*$`)
)

type StringOrFloat64 float64

func (d *StringOrFloat64) UnmarshalJSON(b []byte) error {
	str := strings.Replace(string(b), "\"", "", -1)
	if str == "" {
		*d = StringOrFloat64(0)
		return nil
	}

	if !isFloat.MatchString(str) {
		return fmt.Errorf("failed to match %s: %s", isFloat.String(), str)
	}

	parsed, err := strconv.ParseFloat(str, 64)
	if err != nil {
		return err
	}
	*d = StringOrFloat64(parsed)
	return nil
}
```

`StringOrInt` and `StringOrUint64` are the same thing with a `math.Trunc` before the conversion, so `"2048"`, `2048` and
`2048.0` all land as `2048`. Before that change the integer versions used `strconv.ParseUint`, which chokes on a decimal
point, so moving everything to `ParseFloat` and truncating fixed a second class of report for free. The regex is there so
garbage fails loudly with the offending value in the error instead of quietly becoming zero.

[#59](https://github.com/luthermonson/go-proxmox/issues/59) asked the obvious question back in May: wouldn't it be easier
to always convert to `uint64`? My answer then is still my answer now. The API was returning `1234` and `"1234"` and
interchanging them a lot, and one unmarshaller on a `StringOrUint64` means you don't write a custom `UnmarshalJSON` for
every struct that has a numeric field which might show up quoted. The cost is that you have to convert at the edges
(`uint64(vm.VMID)`), which I'll take.

Going the other way is a non-issue. None of these types have a `MarshalJSON`, so they serialize as plain numbers, and
Proxmox is perfectly happy to accept a number for a field it would hand back to you as a string.

### When upstream changes the type under you

[#105](https://github.com/luthermonson/go-proxmox/pull/105), merged last week, is the other way this bites. Reading a VM
config started failing on `memory` after a Proxmox update. Johann tracked it down to
[qemu-server 8.0.8](https://git.proxmox.com/?p=qemu-server.git;a=commit;h=7f8c808772979f274cdfac1dc7264771a3b7a7ae),
where `memory` got its own property-string parser (`[current=]<integer>`) "in preparation to add more properties to the
memory configuration", and from then on the API returns it quoted. The fix was one line:

```go
Memory    StringOrInt `json:"memory,omitempty"` // See commit 7f8c808772979f274cdfac1dc7264771a3b7a7ae on qemu-server
```

That's the real argument for the StringOr types over hand-picking `int` vs `string` per field. A field can flip between
Proxmox point releases, and when it does you want a type that already accepts both rather than a broken release for
everyone who upgraded their cluster.

### Booleans have the same problem

Booleans are their own flavor. Some endpoints return `true`/`false`, most return `1`/`0`, and the config files on disk
are all `1`/`0`. `IntOrBool` leans on the fact that `strconv.ParseBool` already accepts `1`, `0`, `true` and `false`:

```go
type IntOrBool bool

func (b *IntOrBool) UnmarshalJSON(i []byte) error {
	parsed, err := strconv.ParseBool(string(i))
	if err != nil {
		return err
	}
	*b = IntOrBool(parsed)
	return nil
}

func (b *IntOrBool) MarshalJSON() ([]byte, error) {
	if *b {
		return []byte("1"), nil
	}
	return []byte("0"), nil
}
```

This one does marshal, and always as `1`/`0`, because that's the form every write endpoint accepts.

The rule I follow now when adding a field: if Proxmox documents it as a number, use the matching `StringOr` type unless
I've seen it come back unquoted from every endpoint that shares the struct. It's a little uglier in the struct
definitions and it means far fewer `cannot unmarshal string` issues.

### Links
- [go-proxmox](https://github.com/luthermonson/go-proxmox)
- [#69 cpulimit as a string](https://github.com/luthermonson/go-proxmox/issues/69) and [#71 StringOrFloat64](https://github.com/luthermonson/go-proxmox/pull/71)
- [#59 Use of StringOrUint/64](https://github.com/luthermonson/go-proxmox/issues/59)
- [#105 memory field parsing after qemu-server 8.0.8](https://github.com/luthermonson/go-proxmox/pull/105)
- [encoding/json, the ,string option](https://pkg.go.dev/encoding/json#Marshal)
