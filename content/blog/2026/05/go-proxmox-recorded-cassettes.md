---
title: "Recording Reality: Replacing Hand-Written Proxmox Mocks With Cassettes"
description: "go-proxmox's unit tests run against 39k lines of hand-written JSON that drifted from what Proxmox actually returns. The plan is to use go-proxmox to stand up a nested Proxmox, record real traffic with go-vcr and scrub it."
date: 2026-05-16T21:10:00-07:00
tags: [ proxmox, go, go-proxmox, testing, go-vcr, homelab ]
categories: [ go, homelab ]
images:
 - src: img/2026/05/go-proxmox-recorded-cassettes-pipeline.svg
   alt: "Pipeline diagram: outer Proxmox host provisions a nested Proxmox, tests record against it, a scrub hook cleans the cassettes, CI replays them"
   stretch: "none"
draft: true
---

<!-- TODO(luther): PR #269 is still open and no real cassettes have been recorded yet. Publish once `mage record:pve9`
has run against the lab, and add the first real numbers (cassette count/size vs the gock fixtures). -->

This one is a work in progress, written while the PR is still open. The unit tests in
[go-proxmox](https://github.com/luthermonson/go-proxmox) don't talk to Proxmox. They talk to
[gock](https://github.com/h2non/gock) mocks under `tests/mocks/pve{6,7,8,9}x/`. As of the PR's base commit those
directories are 39,146 lines of Go. Almost all of it is JSON literals somebody typed in by hand, and that somebody was
mostly me. The tests are fast and CI never needs a Proxmox host, which is why it's built that way. The problem is that a
hand-written fixture describes what you *think* Proxmox returns.

### Two bugs the mocks couldn't catch

The two I fixed this past week are perfect examples.

[#198](https://github.com/luthermonson/go-proxmox/issues/198): listing VMs on a node with a template on it failed with
`failed to match ^[0-9.]*$: null`. Templates aren't running, so Proxmox sends `"pid": null`. None of the fixtures had a
template in them. So none of them had a `null` PID, and the tests were green the whole time.

[#211](https://github.com/luthermonson/go-proxmox/issues/211): a VM with more than ten NICs lost every interface past
`net9`. The struct had explicit `Net0` through `Net9` fields. `encoding/json` drops keys it has no field for. Every
fixture VM had one or two NICs.

Both of those would have shown up the first time anyone recorded a real response from a real cluster with a template and
a wide VM on it. So that's the plan.

### go-vcr, with two modes

[go-vcr](https://github.com/dnaeon/go-vcr) sits in the HTTP transport. It either records real interactions to a YAML
cassette or replays them from one. [PR #269](https://github.com/luthermonson/go-proxmox/pull/269) adds a
`tests/recorder` package. The mode is picked by whether `PROXMOX_URL` is set:

```go
func New(t *testing.T) *vcrrecorder.Recorder {
	t.Helper()

	realURL := os.Getenv("PROXMOX_URL")
	mode := vcrrecorder.ModeReplayOnly
	if realURL != "" {
		mode = vcrrecorder.ModeRecordOnce
	}

	cassettePath := filepath.Join("testdata", t.Name())

	opts := []vcrrecorder.Option{
		vcrrecorder.WithMode(mode),
		vcrrecorder.WithSkipRequestLatency(true),
		vcrrecorder.WithMatcher(matchMethodURLBody),
		// Proxmox uses self-signed TLS by default; the recorder needs a
		// transport that tolerates it during recording. No effect in replay.
		vcrrecorder.WithRealTransport(&http.Transport{
			TLSClientConfig: &tls.Config{InsecureSkipVerify: true}, //nolint:gosec
		}),
	}
	// ... scrub hooks, t.Cleanup(r.Stop)
}
```

CI never sets `PROXMOX_URL`, so it's replay only. An interaction that isn't on disk is a hard failure rather than a
surprise network call. The matcher compares method, URL and body and ignores headers on purpose. The User-Agent and the
auth headers don't change what Proxmox sends back. The auth headers aren't going to be in the cassette anyway.

### Scrubbing before anything hits the disk

Cassettes get committed, so every detail that identifies my lab has to come out before the file is written. That's a
chain of `BeforeSaveHook`s:

- drop `Authorization`, `Cookie` and `CSRFPreventionToken` from requests and `Set-Cookie` from responses
- rewrite the real host to `pve.example.test:8006`
- regex the bodies for SSH keys, PEM blocks and `PVE:` tickets

IPs and MACs get special treatment. Replacing them all with one placeholder would make every re-record a giant diff. It
would also collapse two different NICs into one. So they're hashed into documentation ranges. IPv4 goes into TEST-NET-1.
MACs go into `BC:24:11`, the prefix Proxmox itself uses for generated MACs:

```go
// redactIPv4 maps an IPv4 address to a deterministic address in TEST-NET-1
// (192.0.2.0/24, RFC 5737). Deterministic so re-records produce stable diffs.
func redactIPv4(s string) string {
	var sum uint32
	for i := 0; i < len(s); i++ {
		sum = sum*31 + uint32(s[i])
	}
	return fmt.Sprintf("192.0.2.%d", (sum%254)+1)
}
```

Same input, same output, so a re-record only shows a diff where the response really changed. There's a known hole I
haven't closed. 254 buckets means two real addresses can land on the same TEST-NET-1 address. For a lab with a handful
of IPs that's unlikely. But it would make a test that compares two addresses lie, so it'll probably grow a collision map.

### Using go-proxmox to test go-proxmox

The part I'm most happy with is where the recording target comes from. Nobody should have to keep a pet Proxmox around
with exactly the right VMs on it. So `mage record:pve9` builds one from nothing, using the library against an outer
Proxmox host:

![Recording pipeline](/img/2026/05/go-proxmox-recorded-cassettes-pipeline.svg)

1. Download the installer ISO to the outer host's storage (skipped if cached, SHA-256 checked against `versions.go`).
2. Run `proxmox-auto-install-assistant prepare-iso` on the outer host so the installer fetches its `answer.toml` over
   HTTP at boot.
3. Create a nested VM with `cpu=host`, attach the ISO, start it.
4. Serve `answer.toml` and a first-boot script from a short-lived HTTP server on my workstation. The first-boot script
   seeds known state. That's a `recorder@pve!cassettes` token, pinned VMIDs and tags. The VMIDs are a template at 9001,
   a running VM at 9002, a wide config at 9003 with devices past index 9, and two containers.
5. Wait up to 25 minutes for install plus seed, polling `/version` and checking for a marker file over SSH.
6. Run `go test ./tests/recorder/...` in record mode against the nested host.
7. Destroy the nested VM on every exit path, including a failed install.

The seed is the important part. VMID 9001 is a template because of #198 and 9003 has devices past index 9 because of #211.
Every bug that turns out to be "the fixture didn't look like reality" should end up as one more line in the seed.

Getting the answer file onto the installer took two tries. The first version baked `answer.toml` and the seed script into
the ISO. That meant re-mastering the ISO over SSH every time the seed changed. Now the ISO is a one-time skeleton that
knows where to fetch its answers. Iterating on the seed is just editing a file and re-running. Picking the workstation
IP to serve from had its own gotcha on Windows. The auto-detect takes the interface whose subnet most specifically
contains the nested VM's gateway. That's because the WSL and Hyper-V virtual adapters have masks wide enough to look
like they cover the LAN when they can't route to it.

There's also `mage record:preflight`. It checks the things that would otherwise show up as a 25 minute timeout:

- the token works
- the node and storage exist
- the VMID is free
- SSH works
- the bridge exists
- the assistant is installed
- `kvm.nested` is on

PVE 7 is out of scope. The auto-install assistant needs 8.2 or newer, and 7 went end of life in July 2024. So the
hand-written `pve7x` fixtures stay as they are until a major version drops them.

### Where it stands

The scaffolding is in the PR. `mage record:smoke` and `mage record:verify` pass against a hand-crafted smoke cassette.
`mage record:plan` prints what a full run would do without touching anything. What hasn't happened yet is the first
real `mage record:all` against the lab. None of the existing gock tests have moved. The plan after that is to migrate
one `tests/mocks/pveNx/*.go` file per PR and drop gock from `go.mod` when the last one is gone.

<!-- TODO(luther): first real record run — cassette count, size, and anything it caught. -->

### Links
- [PR #269, go-vcr migration scaffolding](https://github.com/luthermonson/go-proxmox/pull/269)
- [#198 null PID on templates](https://github.com/luthermonson/go-proxmox/issues/198)
- [#211 devices past index 9](https://github.com/luthermonson/go-proxmox/issues/211)
- [go-vcr](https://github.com/dnaeon/go-vcr)
- [Proxmox automated installation](https://pve.proxmox.com/wiki/Automated_Installation)
- [RFC 5737, documentation address blocks](https://www.rfc-editor.org/rfc/rfc5737)
