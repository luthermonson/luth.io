---
title: "Per-Tenant Network Policy in One Process: ePHPm's eBPF Thread Tags"
description: "Every site on an ePHPm server runs inside the same process, so a normal firewall can't tell them apart. A small eBPF program tags each thread with its site and keeps sites from reaching each other's local ports, at under a microsecond per connection."
date: 2026-08-28T20:40:00-07:00
tags: [ rust, ephpm, ebpf, linux, multi-tenant, cgroups ]
categories: [ security, ephpm, containers ]
images:
 - src: img/2026/08/ephpm-ebpf-per-vhost-network-flow.svg
   alt: "Flow from a PHP pool thread writing its vhost tag into a BPF map, through the bind and connect cgroup hooks, to allow, rewrite or EPERM"
   stretch: "none"
draft: false
---

In multi-tenant mode [ePHPm](https://github.com/ephpm/ephpm) serves every site in `sites_dir` from one process. Every
tenant's PHP runs as the same OS user, on the same pool of ZTS threads, and a thread that just served `blog.example`
will serve `shop.example` a few milliseconds later. That's the density win. It also means the kernel has no idea
which tenant is behind a given `connect()`. As far as Linux is concerned it's all `ephpm-web`.

Most of the isolation work in ePHPm happens above the kernel: `open_basedir`, the function denylist, per-site
databases and KV keyspaces. Network is the one place where that isn't enough. This week we brought eBPF in to give
the kernel a per-tenant identity. It went from a roadmap doc on Wednesday morning to merged that evening
([#413](https://github.com/ephpm/ephpm/pull/413), aimed at v0.8.1). v0.8.1 never got a release because its Linux
legs failed to link. v0.8.2 shipped with the BPF object silently stubbed out, and v0.8.3 was the first release that
actually carried it. This post covers what it enforces, what it costs, what broke on the way out the door, and what
it doesn't do.

![How a request's vhost reaches the connect hook](/img/2026/08/ephpm-ebpf-per-vhost-network-flow.svg)

### Why uid rules can't see tenants

The hardened preview host already had an nftables table scoped to `meta skuid ephpm-web`: loopback sealed, cloud
metadata and RFC1918 dropped, DNS pinned. That's a good floor, and it treats every tenant the same, because a uid
match is all it has to go on. Two things fall out of that:

- **No sidecars.** If tenant A wants to run something on `127.0.0.1:8080` for its own PHP to talk to, you either seal
  loopback for everyone (A can't reach its own sidecar) or open it for everyone (B can reach A's sidecar too).
- **No per-tenant ports.** Two tenants that both hardcode `127.0.0.1:8080` collide on bind.

Network namespaces are the textbook answer, but they're built for one process per context. For ePHPm that means a
`setns()` per request plus `CAP_SYS_ADMIN`, and a veth, route, NAT and DNS setup per tenant. That doesn't fit a
model where a 1 GB node is supposed to hold [a lot of sites](/blog/2026/08/what-does-one-more-wordpress-site-cost/).
What we needed was a way to attach an identity to a *thread* for the duration of one request and have the kernel
read it at `bind()`/`connect()` time.

### A tag per thread

The whole design hangs off one BPF hash map, `tag`, keyed by thread ID with a numeric vhost ID as the value. Before a
request's PHP runs, the router writes `tag[tid] = vhost`, and it deletes the entry when the request is done. From
`crates/ephpm-server/src/tenant_ebpf.rs`:

```rust
pub fn tag_current_thread(self: &Arc<Self>, site_key: &str) -> TagGuard {
    let vhost = self.site_ids.id_for(site_key);
    let tid = current_tid();
    if let Ok(mut tag) = self.tag.lock()
        && let Err(e) = tag.insert(tid, vhost, 0)
    {
        tracing::warn!(tid, vhost, error = %e, "tenant_ebpf: failed to write tag");
    }
    TagGuard { owner: Arc::clone(self), tid }
}

impl Drop for TagGuard {
    fn drop(&mut self) {
        if let Ok(mut tag) = self.owner.tag.lock() {
            let _ = tag.remove(&self.tid);
        }
    }
}
```

The router creates the guard at the top of `run_php`, the per-request closure both fpm engines run, so it's dropped
on the same pool thread when that closure returns:

```rust
// crates/ephpm-server/src/router.rs
let run_php = move || -> Result<ephpm_php::response::PhpResponse, ephpm_php::PhpError> {
    let _ebpf_tag_guard = ebpf_tag.as_ref().map(|(e, key)| e.tag_current_thread(key));
    // ... per-site KV, DB, open_basedir, then PhpRuntime::execute ...
```

The one invariant that matters here is that a stale tag must never survive onto the next request. If it did, tenant B
would inherit tenant A's network identity on a reused thread, which is the exact cross-tenant leak this is supposed
to prevent. The `Drop` is a `bpf_map_delete_elem` syscall, not a PHP call, so it doesn't cross PHP's
`setjmp`/`longjmp` boundary, and `PhpRuntime::execute` catches its own bailout and returns a `Result`, so the closure
always unwinds normally. The shed, 503 and hung-thread paths return before `run_php` runs at all, so they never write
a tag in the first place.

A failed tag write logs and lets the request run untagged. Keep that in mind for the limits section.

### Which thread is this, really?

On the kernel side the obvious helper is `bpf_get_current_pid_tgid()`, whose low 32 bits are the TID. The catch,
which turned up in the proof of concept before any of this was merged, is that it returns the *global* TID. If ePHPm
runs inside a PID namespace (a container, Kubernetes, or WSL2) the `gettid()` ePHPm sees is a different number, the
lookup misses, and every thread looks untagged. So the loader writes the `{dev, ino}` of `/proc/self/ns/pid` into a
one-slot `nscfg` map at startup, and the programs resolve the TID in that namespace:

```c
// crates/ephpm-server/bpf/vhostnet.bpf.c
static __always_inline __u32 resolve_tid(void)
{
    __u32 zero = 0;
    struct nscfg_t *c = bpf_map_lookup_elem(&nscfg, &zero);
    if (c && c->dev) {
        struct bpf_pidns_info ns = {};
        if (bpf_get_ns_current_pid_tgid(c->dev, c->ino, &ns, sizeof(ns)) == 0)
            return ns.pid; // .pid == TID in that namespace
    }
    return (__u32)bpf_get_current_pid_tgid(); // global TID (low 32 bits)
}
```

I got to re-learn this one personally while benchmarking for this post. I loaded the object with `bpftool` instead of
ePHPm's loader and forgot to fill `nscfg`. Inside WSL2 the tagged bind came back on the port I asked for, and a
tagged connect to a port nobody owned got `Connection refused` from the network stack instead of `EPERM` from the
hook. The hooks were attached and doing nothing. Filling `nscfg` the way the loader does fixed it immediately:

```
## nscfg left empty
tagged -> unowned 127.0.0.1:3307    r=-1 errno=Connection refused
tagged bind 127.0.0.1:8080 -> real port 8080

## nscfg = {dev, ino} of /proc/self/ns/pid
tagged -> unowned 127.0.0.1:3307    r=-1 errno=Operation not permitted
tagged bind 127.0.0.1:8080 -> real port 20000
```

That's the failure mode in a nutshell: a mismatch doesn't error out, it looks like a working box with no policy.

### What connect4 decides

Four programs attach to ePHPm's own cgroup: `cgroup/bind4`, `bind6`, `connect4` and `connect6`. They only touch the
UAPI fields of `struct bpf_sock_addr` (`user_ip4`, `user_ip6`, `user_port`), so there's no CO-RE relocation and no
`vmlinux.h`. `connect4` calls it loopback when the destination is `127.0.0.1` and `connect6` when it's `::1`, then
both hand off to the same function, which is short enough to show whole:

```c
static __always_inline int do_connect(struct bpf_sock_addr *ctx, int is_loopback)
{
    __u32 *vhostp = current_vhost();
    if (!vhostp)
        return 1; // untagged: connect unchanged
    __u32 vhost = *vhostp;

    if (!is_loopback)
        return 1; // non-loopback egress handled by the per-vhost allowlist layer

    __u32 dport = bpf_ntohs(ctx->user_port);

    // 1) Virtual port for this vhost -> rewrite to its private real port.
    __u64 key = ((__u64)vhost << 32) | dport;
    __u32 *rport = bpf_map_lookup_elem(&assigned, &key);
    if (rport) {
        ctx->user_port = bpf_htons((__u16)*rport);
        return 1;
    }

    // 2) Shared ePHPm infra port (MySQL wire, KV RESP) -> always allow.
    if (bpf_map_lookup_elem(&infra_ports, &dport))
        return 1;

    // 3) Direct connect to a real port -> authorize by ownership.
    struct owner_t *o = bpf_map_lookup_elem(&port_owner, &dport);
    if (o)
        return (o->vhost == vhost) ? 1 : 0; // own sidecar allow; cross-vhost DENY

    // 4) Unowned loopback port: DENY for managed vhosts. This is the loopback
    //    floor that REPLACES the static nft loopback-drop for tagged traffic.
    return 0;
}
```

Returning `0` from a cgroup sockaddr program fails the syscall with `EPERM`. Step 4 is the important one. It's the
loopback floor, and it means a tagged tenant can't reach a loopback port it doesn't own and isn't infra, which
includes ePHPm's own HTTP listener and whatever else the box runs on localhost.

Read that comment on the non-loopback branch carefully, though. It says public egress is "handled by the per-vhost
allowlist layer". There is no per-vhost allowlist layer. The roadmap's phase 1 described one, but what shipped passes
every non-loopback connect straight through to whatever the operator's nftables table does, and that table is
per-uid. So the eBPF policy is a per-tenant *loopback* policy. More on that below.

The tag is also only written in fpm mode with `sites_dir` set. `validate()` rejects `ebpf_policy = true` off Linux,
without `sites_dir`, and in worker mode. Worker mode's persistent loop would need per-event tagging inside the PSR-7
envelope, and that isn't built.

### Every tenant gets its own :8080

The bind side is what makes sidecars usable. When a tagged thread binds, `bind4` pops a free port from a pool ePHPm
owns, rewrites the bind to it and records who owns it. The app asked for `127.0.0.1:8080`, the socket really lands
on `:20000`, and when the same tenant later connects to `:8080`, step 1 above rewrites the destination to `:20000`.
Another tenant binding `:8080` gets `:20001`. Neither app knows anything happened.

```c
// Idempotent rebind: same (vhost, vport) reuses its real port.
__u32 *ex = bpf_map_lookup_elem(&assigned, &key);
if (ex) {
    ctx->user_port = bpf_htons((__u16)*ex);
    return 1;
}

// Per-vhost quota (anti port-bomb), enforced in-kernel on the tag.
__u32 *cntp = bpf_map_lookup_elem(&sidecar_count, &vhost);
__u32 cnt = cntp ? *cntp : 0;
if (cnt >= quota_max())
    return 0; // DENY: this vhost is at its sidecar cap

__u32 rport = 0;
if (bpf_map_pop_elem(&port_pool, &rport) != 0)
    return 0; // DENY: pool exhausted (box-wide sidecar cap reached)
```

The pool is a `BPF_MAP_TYPE_QUEUE` that the loader fills from `sidecar_port_range` (default `20000-32767`) at
startup, so allocating is an O(1) pop with no scan. Before filling it, the loader reads
`net.ipv4.ip_local_port_range` and refuses to start if the ranges overlap. Otherwise an outbound connect could get an
ephemeral source port that ePHPm also wants to hand out as a sidecar port.

The part that didn't make it is giving ports back. Returning a port to the pool when a listener closes needs either
a `cgroup/sock_release` program that reads the socket's source port (the verifier rejects that read on 6.18) or a
`BPF_MAP_TYPE_SK_STORAGE` map to hang ownership on the socket (aya 0.13 can't load that map type). So ports and
per-vhost counts come back only when the process exits. The `assigned` map keeps a rebind of the *same* virtual port
idempotent, so a sidecar that restarts on `:8080` doesn't burn a second slot, but a tenant that cycles through many
different ports will hit its `max_sidecar_ports_per_vhost` cap (default 8) until ePHPm restarts. The code comments
and the guide say that's coming in v0.8.2. v0.8.5 is out and it still isn't there, because neither blocker has moved.

### Watching it work

There's no automated test in the repo yet that loads these programs. They need root and a writable cgroup v2
hierarchy, and nothing in CI has either. So I compiled the v0.8.5 `vhostnet.bpf.c` with `build.rs`'s flags, attached
it to a scratch cgroup with `bpftool`, and ran a small C program as three tenants. Alice and Bob each bind
`127.0.0.1:8080` from a tagged thread, and the main thread then re-tags itself as one tenant or the other and
connects:

```
alice bind 127.0.0.1:8080 -> real port 20000
bob   bind 127.0.0.1:8080 -> real port 20001
bob      -> 127.0.0.1:8080  (own virtual port)     connected, read "hello from bob"
bob      -> 127.0.0.1:20001 (own real port)        connected, read "hello from bob"
bob      -> 127.0.0.1:20000 (alice's real port)    Operation not permitted
bob      -> 127.0.0.1:3306  (infra port)           connected, read "hello from infra"
bob      -> 127.0.0.1:9999  (unowned port)         Operation not permitted
alice    -> 127.0.0.1:8080  (own virtual port)     connected, read "hello from alice"
alice    -> 127.0.0.1:20001 (bob's real port)      Operation not permitted
untagged -> 127.0.0.1:20000 (alice's real port)    connected, read "hello from alice"
carol    bind 127.0.0.1:9001 -> real port 20002
carol    bind 127.0.0.1:9008 -> real port 20009
carol    bind 127.0.0.1:9009 -> Operation not permitted
```

Both tenants hardcoded `:8080` and each one reached its own listener. A cross-tenant connect to the other's real port
is a kernel `EPERM`, not a `Connection refused`. Carol's ninth distinct port hits the quota of 8. The untagged line
is the one to remember for later: a thread with no tag isn't a tenant, so the policy doesn't apply to it.

The other half of verifying this is cleanup. The loader attaches with fd-based `bpf_link`s and pins nothing to bpffs.
A graceful shutdown drops the `aya::Ebpf` and the kernel detaches. A `kill -9` closes the fds and the kernel
refcounts everything to zero on its own. The loader commit ([`0cb5ec7`](https://github.com/ephpm/ephpm/commit/0cb5ec7))
records `bpftool` showing 0 programs, 0 maps and 0 cgroup attachments after a `kill -9`. I didn't repeat that with
aya myself, since my harness used `bpftool` to load. A cold restart never has stale BPF state to reconcile.

### What it costs

The hook runs once per `bind()` or `connect()`, not per packet, and it's a handful of hash lookups. The other cost
is the two map syscalls per PHP request. I measured both with the same harness (test setup below):

![Added connect() latency and per-request tag cost](/img/2026/08/ephpm-ebpf-per-vhost-network-overhead.svg)

A loopback TCP `connect()` took about 35.4 µs at p50 with nothing attached. With the four programs attached it took
0.7 to 0.9 µs longer, about 2 to 2.5%, and the tagged paths (lookup plus rewrite or infra allow) cost a little more
than the untagged one. The p99 sat at 53 to 59 µs in every configuration and didn't separate from the noise. The
tag write plus delete that every request pays came out at 1.64 µs p50, about 800 ns per `bpf()` syscall. That's
several times the roadmap's "tens to low-hundreds of nanoseconds" estimate, but next to running a PHP request it's
still noise.

### Shipping it was the hard part

The kernel code was the quick part. Getting a C BPF object into a Rust binary that builds on four CI shapes took
five follow-up commits over Wednesday and Thursday, with v0.8.2 through v0.8.4 cut along the way.

**clang isn't always there.** The BPF program is C, compiled by `build.rs` with `clang -target bpf` and embedded with
`aya::include_bytes_aligned!`. (Plain `include_bytes!` puts the blob at 1-byte alignment and aya's ELF parser fails
with "error parsing ELF data".) The reasoning was that clang is already a build dependency for bindgen. It isn't:
stub-mode CI installs `libclang-dev` for bindgen but not the `clang` binary, and the first version of `build.rs`
panicked and broke every job that compiled `ephpm-server`. Now `build.rs` never panics:

```rust
// crates/ephpm-server/build.rs
if !matches!(&run, Ok(status) if status.success()) {
    std::fs::write(&out, [])
        .expect("write placeholder eBPF object so include_bytes_aligned! compiles");
    // ... loud cargo:warning naming why ebpf_policy is unavailable ...
    return;
}
```

An empty object fails closed at load time, the feature is off by default, and the release is smoke-tested to make
sure the real object shipped. (The module doc comment at the top of `build.rs` still says it fails the build when
clang is missing. It doesn't.)

**`gettid()` isn't in old glibc.** The release links against an older glibc to keep the binary's floor low, and the
glibc `gettid()` wrapper only exists from 2.30. The v0.8.1 release legs failed with `undefined symbol: gettid` while
CI on newer runners was green. The fix is the raw syscall, which has always existed:

```rust
let tid = unsafe { libc::syscall(libc::SYS_gettid) };
```

**AlmaLinux 8 has no libbpf headers.** Even with clang installed on the release legs, v0.8.2 shipped with the eBPF
object *stubbed*. The release builds in an almalinux8 image, `libbpf-devel` isn't packaged there,
`bpf/bpf_helpers.h` wasn't found, and the never-panic path dutifully wrote the empty placeholder. The fix was to
vendor the five program-side libbpf headers under `bpf/include` and pass `-I bpf/include`. That's the downside of a
build that degrades: it degrades quietly, which is why the release smoke test exists.

**`LimitMEMLOCK=infinity` in the unit.** The loader calls `setrlimit(RLIMIT_MEMLOCK, infinity)` itself, but the
hardened unit sets `NoNewPrivileges` and doesn't grant `CAP_SYS_RESOURCE`, so the raise is refused. Per
[`bd2dd03`](https://github.com/ephpm/ephpm/commit/bd2dd03), map creation then failed with `Operation not permitted`
even with `CAP_BPF` and `CAP_NET_ADMIN` present, an error that sends you looking at the capabilities, which are fine.
I didn't reproduce that one. The loader's own comment says the rlimit only matters before 5.11, when BPF memory moved
to cgroup accounting, so I'd treat the setting as cheap insurance and keep it. The drop-in:

```ini
# /etc/systemd/system/ephpm.service.d/ebpf.conf
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE CAP_BPF CAP_NET_ADMIN
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_BPF CAP_NET_ADMIN
LimitMEMLOCK=infinity
```

**The firewall has to step aside.** The cgroup connect hook runs before the packet ever reaches nftables. If your
nft table still drops loopback for the ePHPm uid, the hook authorizes a sidecar connect and nft kills it one step
later, and the app just sees a refused connection. With `ebpf_policy` on, the loopback drop has to come out of the
nft table so `connect4` is the only thing deciding loopback, while the metadata, LAN and DNS rules stay put. ePHPm
can't see your firewall, so all it can do is name the handoff in the startup error when the load fails. (The guide
says it also logs a warning on a successful start. I couldn't find one in the code.)

Related: [`6df4cdc`](https://github.com/ephpm/ephpm/commit/6df4cdc) landed alongside the loader. With egress enforced
below PHP, blocking `fsockopen` while `curl` and `stream_socket_client` reach the same places is just friction, so
`network_egress_externally_managed = true` drops it from the denylist. `pfsockopen` stays blocked. That one is about
persistent sockets outliving a request on a shared thread, which the network layer does nothing about.

### What this doesn't do

- **Public egress isn't per tenant.** The eBPF layer governs loopback and sidecar ownership. Every non-loopback
  connect passes through to the operator's nft table, which ePHPm doesn't ship and which is per-uid. With the
  hardened table in place a hostile tenant still reaches public addresses.
- **It covers `bind()` and `connect()`, nothing else.** Unix domain sockets aren't covered by these hooks, and
  neither is anything that doesn't go through a cgroup sockaddr bind or connect.
- **Untagged means unrestricted.** Threads with no tag pass through by design: ePHPm's own async runtime, and any
  request for a Host that matched no vhost. A failed tag write also leaves the request untagged. The loader's comment
  says that fails closed. From the code it looks to me like an untagged request is let through, not denied.
- **Only request threads are tagged.** A process started from PHP has its own TIDs and none of them are in the map.
  The config doc for `cgroup_path` says the policy covers "any sidecars it spawns". The hooks fire for that process
  because it's in the cgroup, but they treat it as untagged.
- **Linux only, with privileges.** Kernel 5.10 or newer with `CONFIG_CGROUP_BPF`, cgroup v2, `CAP_BPF` +
  `CAP_NET_ADMIN`, and the memlock limit above. That's two more capabilities than the hardened unit had, and that's
  the trade. On Windows and macOS, setting `ebpf_policy = true` is a config error.
- **Ports leak until restart**, as covered above.
- **No test loads it.** Nothing in the test suite attaches the BPF programs. The e2e hardening suite covers the
  function denylist and cross-tenant `ATTACH` (the litewire side of that is in the
  [database proxy hardening post](/blog/2026/08/hardening-rust-database-proxy/)), not eBPF. Getting a privileged
  runner that can exercise it is on the list.
- **It's still one process.** A tenant that crashes the process takes everyone down. This is a confidentiality
  boundary, not an availability one.

It's off by default, labeled experimental, and fails closed at startup: if the programs can't load or attach, or the
port range overlaps the ephemeral range, ePHPm refuses to start rather than run without the policy. Next up is a
privileged test that runs the real loader, then close-time port reclaim, which is waiting on either a loader that can
do `SK_STORAGE` or another close signal the verifier will accept.

### Test setup

- WSL2 on Windows 11, kernel 6.18, cgroup v2, BTF present.
- ePHPm [v0.8.5](https://github.com/ephpm/ephpm/releases/tag/v0.8.5)
  ([`d7b1898`](https://github.com/ephpm/ephpm/commit/d7b1898)). I compiled its `vhostnet.bpf.c` with the clang flags
  from its `build.rs` (`-O2 -g -Wall -Werror -target bpf -I bpf/include`), then `llvm-strip -g`.
- I loaded it with `bpftool prog loadall`, attached the four programs to a scratch cgroup, filled `port_pool` with
  101 ports, set the quota to 8, added `3306` to `infra_ports` and wrote `nscfg` the way the loader does. Same kernel
  object, different loader.
- The harness is a C program running in that cgroup. It times only the `connect()` syscall to a loopback listener
  (with `SO_LINGER` 0 so closes don't pile up `TIME_WAIT`), 20,000 connects per case, five rounds with programs
  attached and five without. The tag cost is a `BPF_MAP_UPDATE_ELEM` plus `BPF_MAP_DELETE_ELEM` on the `tag` map,
  20,000 times per round. The chart shows the mean of the per-round p50s.
- All of this is loopback on one machine inside a VM, so treat the absolute numbers as this box's numbers. The deltas
  are the useful part. I didn't measure end-to-end request throughput through a PHP-linked ePHPm with the policy
  on.

### Links

- Design: [roadmap doc](https://github.com/ephpm/ephpm/blob/v0.8.5/site/content/roadmap/ebpf-per-vhost-network.md),
  [#412](https://github.com/ephpm/ephpm/pull/412), [`4df45eb`](https://github.com/ephpm/ephpm/commit/4df45eb)
- Feature PR: [#413](https://github.com/ephpm/ephpm/pull/413)
- Config and loader: [`56ea958`](https://github.com/ephpm/ephpm/commit/56ea958),
  [`0cb5ec7`](https://github.com/ephpm/ephpm/commit/0cb5ec7)
- `fsockopen` lifted when egress is managed below PHP: [`6df4cdc`](https://github.com/ephpm/ephpm/commit/6df4cdc)
- Build and release fixes: [`557d96c`](https://github.com/ephpm/ephpm/commit/557d96c) and
  [`dda1e1d`](https://github.com/ephpm/ephpm/commit/dda1e1d) (clang, `build.rs` never panics),
  [`e9ff5fe`](https://github.com/ephpm/ephpm/commit/e9ff5fe) (`SYS_gettid`),
  [`84efd26`](https://github.com/ephpm/ephpm/commit/84efd26) (vendored libbpf headers),
  [`bd2dd03`](https://github.com/ephpm/ephpm/commit/bd2dd03) (`LimitMEMLOCK=infinity`)
- Source at v0.8.5: [`vhostnet.bpf.c`](https://github.com/ephpm/ephpm/blob/v0.8.5/crates/ephpm-server/bpf/vhostnet.bpf.c),
  [`tenant_ebpf.rs`](https://github.com/ephpm/ephpm/blob/v0.8.5/crates/ephpm-server/src/tenant_ebpf.rs),
  [`build.rs`](https://github.com/ephpm/ephpm/blob/v0.8.5/crates/ephpm-server/build.rs)
- Guides at v0.8.5: [per-vhost network policy (eBPF)](https://github.com/ephpm/ephpm/blob/v0.8.5/site/content/guides/ebpf-per-vhost-network.md),
  [multi-tenant hardening](https://github.com/ephpm/ephpm/blob/v0.8.5/site/content/guides/multi-tenant-hardening.md)
- Related on luth.io: [what one more WordPress site costs](/blog/2026/08/what-does-one-more-wordpress-site-cost/),
  [hardening a Rust database proxy](/blog/2026/08/hardening-rust-database-proxy/)
- [aya](https://aya-rs.dev/)
