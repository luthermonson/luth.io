---
title: "Five Ways to URL-Encode an SSH Key (and Why Proxmox Rejects Them)"
description: "Proxmox's sshkeys parameter wants Python-style percent encoding that no single Go function produces. A two-year-old issue, a one-line fix, and a related double-encoding bug in guest file writes."
date: 2026-07-28T20:12:00-07:00
tags: [ proxmox, go, go-proxmox, cloud-init, encoding ]
categories: [ go, homelab ]
images:
 - src: img/2026/07/five-ways-to-url-encode-an-ssh-key-results.svg
   alt: "Table of seven ways to encode an SSH public key and whether each passes Proxmox's urlencoded validator"
   stretch: "none"
draft: false
---

[Issue #144](https://github.com/luthermonson/go-proxmox/issues/144) on go-proxmox was opened in April 2024 with the title
"Weird sshkeys SSH public key validation error" and it is one of my less proud threads. Thirteen comments of people,
including me, trying to get a public key into a cloud-init VM through the API, and the working answer took more than a
year to show up. The fix that finally landed in May is one line. Here's why it took so long.

### What the API wants

Setting `sshkeys` on a VM is how you get a key into cloud-init's `authorized_keys`. With `qm` on the node you can point it
at a file (`--sshkeys ~/.ssh/id_rsa.pub`), which is where the thread started: the reporter passed a path through the API
and got `500 SSH public key validation error`, because over the API you have to send the key text itself. That part is
fair. The confusing part is the next error, once you send the key:

```
400 Parameter verification failed. - {"sshkeys":"invalid format - invalid urlencoded string: ssh-rsa%20AAAAB ...
```

The schema declares the parameter with `format => 'urlencoded'`, and the format check lives in
[pve-common's JSONSchema.pm](https://git.proxmox.com/?p=pve-common.git;a=blob;f=src/PVE/JSONSchema.pm):

```perl
sub pve_verify_urlencoded {
    my ($text, $noerr) = @_;
    if ($text !~ /^[-%a-zA-Z0-9_.!~*'()]*\z/) {
        return undef if $noerr;
        die "invalid urlencoded string: $text\n";
    }
    return $text;
}
```

That's the entire allowed character set: letters, digits, `- _ . ! ~ * ' ( )` and `%`. Everything else has to be
percent-encoded, including `+`, `/`, `=`, `@` and space, which are exactly the characters that show up in an OpenSSH
public key (base64 body, `user@host` comment, spaces between the fields). Multiple keys are separated by newlines, so
`%0A` too.

### Trying them all

The thread worked through `url.PathEscape`, `url.QueryEscape` and three flavors of base64. My own contribution was
"try `url.QueryEscape`", which the reporter had already said didn't work. A couple of hours later I linked the regex above,
which was the right thing to look at, and then nobody, me included, followed it through. So here it is followed
through. This program generates a throwaway ed25519 key whose base64 contains both `+` and `/`, runs each candidate
through the same regex, and checks whether URI-unescaping the result gives the original key back (which is what Proxmox
does with it before handing it to cloud-init):

```go
var pveURLEncoded = regexp.MustCompile(`^[-%a-zA-Z0-9_.!~*'()]*$`)

cands := []struct{ name, val string }{
	{"raw", key},
	{"url.PathEscape", url.PathEscape(key)},
	{"url.QueryEscape", url.QueryEscape(key)},
	{"base64.StdEncoding", base64.StdEncoding.EncodeToString([]byte(key))},
	{"base64.URLEncoding", base64.URLEncoding.EncodeToString([]byte(key))},
	{"base64.RawURLEncoding", base64.RawURLEncoding.EncodeToString([]byte(key))},
	{"EncodeSSHKeys", strings.ReplaceAll(url.QueryEscape(key), "+", "%20")},
}
```

```
key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJvjKHeVkHgJQH4UQ5/xmNW/x9zm+bx0PLNpFG9hC0gB luther@example
raw                    validator=false roundtrip=true  rejected=" /+@" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJvjKHeVkHgJ...
url.PathEscape         validator=false roundtrip=true  rejected="+@"   ssh-ed25519%20AAAAC3NzaC1lZDI1NTE5AAAAIJvjKHeVkH...
url.QueryEscape        validator=false roundtrip=false rejected="+"    ssh-ed25519+AAAAC3NzaC1lZDI1NTE5AAAAIJvjKHeVkHgJ...
base64.StdEncoding     validator=false roundtrip=false rejected="="    c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFB...
base64.URLEncoding     validator=false roundtrip=false rejected="="    c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFB...
base64.RawURLEncoding  validator=true  roundtrip=false rejected=""     c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFB...
EncodeSSHKeys          validator=true  roundtrip=true  rejected=""     ssh-ed25519%20AAAAC3NzaC1lZDI1NTE5AAAAIJvjKHeVkH...
```

![Results](/img/2026/07/five-ways-to-url-encode-an-ssh-key-results.svg)

Going through the failures:

- **`PathEscape`** is for a single path segment, so it encodes the space and the `/` but leaves `+` and `@` alone
  because they're legal in a path. Proxmox doesn't want them.
- **`QueryEscape`** does HTML form encoding, where a space is `+`. The validator rejects `+` outright, and even if it
  didn't, a URI unescape that isn't form-aware turns it back into a literal `+` and you'd get a corrupted key.
- **Standard and URL-safe base64** end in `=` padding here, which fails the regex. It doesn't matter much, because...
- **`RawURLEncoding`** sails through the validator, and it's the worst one of the bunch. The alphabet is entirely legal
  characters, so the format check passes and Proxmox stores the base64 *text* as the key. Then the key check, or
  cloud-init, gets a string that isn't an OpenSSH key.

What actually works is RFC 3986 percent encoding with nothing marked safe, which is what Python's
`urllib.parse.quote(key, safe='')` does and what a Proxmox staff member recommended on the forum years ago. Go doesn't
have that as a single function, but `QueryEscape` is one substitution away from it: it percent-encodes everything
except unreserved characters, and the only thing it does differently is spaces as `+`.

### The fix

[@abbaszai posted a working Go version](https://github.com/luthermonson/go-proxmox/issues/144) in August 2025, with some
extra trimming and decoding logic around the core idea. [#273](https://github.com/luthermonson/go-proxmox/pull/273)
reduced it to the core and put it in the library next to `MakeTag`:

```go
// EncodeSSHKeys returns the given SSH public keys encoded the way Proxmox's
// sshkeys parameter requires. Multiple keys are joined with newlines (PVE's
// documented separator) before encoding.
func EncodeSSHKeys(keys ...string) string {
	return strings.ReplaceAll(url.QueryEscape(strings.Join(keys, "\n")), "+", "%20")
}
```

```go
opts := []proxmox.VirtualMachineOption{
	{Name: "sshkeys", Value: proxmox.EncodeSSHKeys(pubKey1, pubKey2)},
}
```

The `+` to `%20` swap is safe because `QueryEscape` already turned every literal `+` in the input into `%2B`, so the only
`+` characters left in its output are spaces. The test table includes a key with a literal `+` to prove that.

It would be easy to do this automatically inside the VM config call, and I deliberately didn't. Everyone who hit #144
has a workaround in their code that already encodes the key one way or another, and auto-encoding would encode their
already-encoded value a second time. `%20` becomes `%2520`, the validator is perfectly happy with that, and the key that
lands in the guest is garbage with no error anywhere. An opt-in helper and a doc comment on the `SSHKeys` field is the
version that can't silently break anyone.

### The same mistake, in the other direction

Encoding flags bit again this month. The QEMU guest agent wrapper has `AgentFileWrite`, for writing a file into a running
VM. The guest agent wants base64 content, and the Proxmox endpoint has an `encode` parameter that the API docs describe
like this:

> If set, the content will be encoded as base64 (required by QEMU). Otherwise the content needs to be encoded beforehand - defaults to true.

The original wrapper base64-encoded the content itself and *also* sent `encode: 1`, with a confident comment saying
that `encode=1` "tells PVE the content is already base64". It means the opposite. Proxmox encoded it again, the guest
agent decoded it once, and every file written through that method landed in the guest as base64 text. Someone writing a
TOML config and a PEM file into a guest found it when the service inside crash-looped on
`expected value but found '='`. [#343](https://github.com/luthermonson/go-proxmox/pull/343) keeps the client-side
encoding (binary safe) and sends `encode: 0`:

```go
body := map[string]interface{}{
	"file":    file,
	"content": base64.StdEncoding.EncodeToString(content),
	"encode":  0,
}
```

Both bugs are the same lesson. When two layers each might encode something, decide which one does it and write the
decision down where the next person will see it, and when there's a flag for it, read the description of the flag on the
side that receives it rather than guessing from the name.

### Links
- [#144 the original thread](https://github.com/luthermonson/go-proxmox/issues/144) and [#273 EncodeSSHKeys](https://github.com/luthermonson/go-proxmox/pull/273)
- [#343 AgentFileWrite double encoding](https://github.com/luthermonson/go-proxmox/pull/343)
- [pve-common JSONSchema.pm](https://git.proxmox.com/?p=pve-common.git;a=blob;f=src/PVE/JSONSchema.pm)
- [Proxmox forum, using pvesh to set sshkeys](https://forum.proxmox.com/threads/how-to-use-pvesh-set-vms-sshkeys.52570/#post-243381)
- [Python urllib.parse.quote](https://docs.python.org/3/library/urllib.parse.html#urllib.parse.quote)
