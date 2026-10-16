---
title: "Every Way a Tenant Escaped: A Cross-Tenant Breakout Field Guide"
description: "Eight ways one site broke out of its sandbox on a multi-tenant PHP server, all found by attacking it on purpose, all fixed."
date: 2026-09-13T16:20:00-07:00
tags: [ php, rust, ephpm, security, multi-tenant ]
categories: [ security, ephpm, php ]
images:
 - src: img/2026/09/cross-tenant-breakout-field-guide-cover.svg
   alt: "Table of tenant isolation boundaries in ePHPm, how each was crossed, and the PR that fixed it"
   stretch: "none"
draft: false
---

[ePHPm](https://github.com/ephpm/ephpm) can serve a directory full of sites out of one process. Point `sites_dir` at
a folder, every subdirectory is a virtual host, and the Host header decides which one you get. That's how the PR preview
environment runs: every pull request gets a subdomain, and the code behind it is whatever somebody pushed. One process
holding many tenants means every per-tenant boundary is something ePHPm enforces itself, in-process. There's no
container wall between site A and site B unless you add one.

So I spent a couple of weeks attacking it. Every item below was found by deliberately trying to break out,
reproduced against a real PHP-linked binary, and is fixed. I'm writing them up together because they're a better
checklist than any I could have written from scratch, and because they cluster into a handful of patterns that apply to
any multi-tenant runtime.

![The boundaries that broke](/img/2026/09/cross-tenant-breakout-field-guide-cover.svg)

One note before the list: ePHPm's own docs say plainly that `open_basedir` is an in-process boundary, not a container
boundary. Untrusted per-PR code still deserves real per-preview isolation. None of what follows changes that advice.
It's about making the in-process boundary actually hold.

### 1. The Host header was a path

In `sites_dir` mode the document root was `sites_dir.join(raw_host)`, with no sanitization of the Host header. The
downstream containment checks were anchored to *that* root, so they canonicalized to the escaped directory and passed.
Pre-auth, no tenant code needed:

| Request | Result before the fix |
|---|---|
| `Host: ../../../../../etc`, `GET /passwd` | 200, served `/etc/passwd` |
| `Host: ../../../../../ETC.` (trailing dot + case) | 200, served `/etc/passwd` |
| `Host: ../single`, `GET /pwn.php` | 200, **executed PHP from an arbitrary docroot** |

The only prior mitigation was `trusted_hosts`, which is empty by default and exact-match only, so it's unusable for
per-PR hostnames.

The fix ([#280](https://github.com/ephpm/ephpm/pull/280)) validates the normalized host as a strict DNS-label
allowlist: `[a-z0-9._-]`, and no empty label. That one "no empty label" rule rejects `.`, `..`, `a..b`, and leading or
trailing dots in a single test, and `/`, `\` and NUL fall outside the charset. After that, `sites_dir.join(key)` can only
ever name a direct child. A malformed host is a 404 before any filesystem join.

The fix is lexical on purpose. The obvious alternative is `canonicalize().starts_with(sites_dir)`, but the router
deliberately supports a site directory that's a symlink to a release tree *outside* `sites_dir`, which is the
atomic-deploy layout. A canonical containment check would break that. Once the label can't contain traversal
primitives, a direct child is all you need.

### 2. The suffix that collapsed every tenant into one

Preview hosts look like `pr-123.preview.example.com`, and `sites_domain_suffix` strips the shared suffix to get the
site key. The validation from #1 ran on the host *before* stripping, and the stripped result was used without being
checked again. With a dotless suffix (say `localhost`), the request `Host: localhost` stripped to the empty string,
and `sites_dir.join("")` is `sites_dir` itself. That's one virtual host whose document root and `open_basedir` are
the entire tenant fleet, with read and write to every site. The empty key also selected DB file `.db`, KV keyspace
`""` and the override lookup, so the whole tenant identity collapsed.

[#401](https://github.com/ephpm/ephpm/pull/401) re-validates the stripped key (an empty or invalid key falls through to
the default site, never to a `sites_dir`-rooted basedir) and rejects a suffix without a leading dot at config load. A
legitimate suffix always starts with `.`, so a dotless one has no valid use.

### 3. Swapping your own web root for a symlink

Sites can declare a document root override, so `document_root = "web"` serves `<site>/web` while PHP can still
`require` upward into the site's container. The override was validated with `starts_with(container)` once, at load. But
the resolved root got re-canonicalized every couple of seconds, following symlinks, with no containment re-check. And
the static file path's only boundary is that root; it has no `open_basedir` backstop.

The tenant can't plant the link from PHP. `symlink()` checks both the link path and its target against `open_basedir`,
so `symlink('/', 'web')` from a tenant script just fails with an open_basedir warning. But PHP isn't the only thing that
writes into a site directory. On the preview environment the site is a checkout of whatever somebody pushed, and git
stores symlinks as symlinks:

```sh
git rm -r web
ln -s / web
git add web && git commit -m 'tidy' && git push
```

The same goes for an archive the deploy unpacks, SFTP, or a shell on the box. Once a link like that exists after the
override was validated, nothing looked at it again. The test for the fix plants it from outside PHP the same way, and on
a Windows test box the next static requests looked like this:

| Request (static) | Before | After |
|---|---|---|
| `GET /Windows/win.ini` | 200, `; for 16-bit app support` | 404 |
| `GET /.../sites/victim.test/secret.db` | 200, the other tenant's DB | 404 |
| `GET /index.txt` (legit) | 200 | 200 |

[#398](https://github.com/ephpm/ephpm/pull/398) re-asserts `canonical_document_root.starts_with(canonical_container)`
on every resolve, against the operator-chosen container that no override can change. It fails closed (static 404, PHP
403), and when no override is in play the second canonicalize is skipped, so it costs nothing on normal sites. This
one never shipped in a release; it was caught between the override feature merging and the next tag.

### 4. Everybody's /tmp

Every vhost's `open_basedir` included the same system temp directory, and `session.save_path`, `sys_temp_dir` and
`upload_tmp_dir` were never set per tenant. So every site's sessions sat in one shared directory that was inside every
site's sandbox:

```
# from site-b:
CROSS_READ_site_a_session_file (/tmp/sess_bb469cd9...) =>
  READ: 'site_a_session_secret|s:20:"SITE_A_SESSION_VALUE";'
sess_glob_own_tmp=[7 other tenants' sess_* files]
```

That's session hijack and tamper across tenants. [#285](https://github.com/ephpm/ephpm/pull/285) gives each vhost a
private state root, `<temp>/ephpm-vhosts/<label>-<digest>`, derived from the already-validated document root, with
`tmp/` and `sessions/` underneath at `0700`. `open_basedir` becomes the document root plus that state root only.

Fixing it exposed a second bug. Per-request ini overrides were applied *after* `php_request_startup()`, but
`php_request_startup()` is where the multipart parser writes uploaded files, using `upload_tmp_dir` as it is at that
moment. Uploads kept landing in shared `/tmp`, now outside `open_basedir`, so `move_uploaded_file()` failed. The ini
replay now runs both before and after startup.

### 5. ATTACH DATABASE as a filesystem primitive

This was the worst one, filed as #274 after live exploitation. The embedded SQLite engine runs inside the ePHPm
process, not under any tenant's `open_basedir`, and in multi-site mode every vhost shared one database. From a hostile
vhost:

1. `mysqli_connect` with junk credentials connected (the proxy didn't validate them) and read another site's
   `wp_users.user_pass`.
2. `ATTACH DATABASE '.../site-b.db'` read and wrote another tenant's database.
3. `ATTACH DATABASE '.../sites/site-a/shell2.php'` *wrote a PHP file into another tenant's docroot*, where PHP's own
   `file_put_contents` had been correctly denied.
4. `GET /shell2.php` with that tenant's Host executed it as them. Full cross-tenant RCE.
5. The same `ATTACH` worked through the always-on native `ephpm_db_*` bridge, so firewalling the TCP proxy wouldn't
   have closed it.

[#284](https://github.com/ephpm/ephpm/pull/284) made the database *file* the tenant boundary, since the engine has no
per-schema ACLs: `[db.sqlite] dir` gives every site its own `<key>.db`, required and fail-closed in multi-site mode. The
shared wire listeners don't start in multi-site mode at all (one listener for all tenants is the original bug), and
`ATTACH`, `DETACH`, `VACUUM` and path `PRAGMA`s are rejected on the tenant path with ePHPm's own error, independent of
the engine's defaults. [#351](https://github.com/ephpm/ephpm/pull/351) pins it with an end-to-end test from real PHP
userland on both the native and wire paths.

The follow-up pentest found the screen matched on a statement's *leading keyword*, so `EXPLAIN ATTACH DATABASE '...'`
sailed past it, stopped only by the engine's "ATTACH is experimental" default.
[#287](https://github.com/ephpm/ephpm/pull/287) peels `EXPLAIN` and `EXPLAIN QUERY PLAN` (stacked, with comments in
between) before checking the verb. A denylist has to see the statement the way the engine will see it.

### 6. Two names, one docroot, two databases

The same pentest round found the derivations disagreeing with each other. Docroot lookup stripped the domain suffix and
the DB key didn't, so `Host: shop.local` and `Host: shop` shared a document root and a session directory but wrote to
different databases. The KV keyspace used the raw Host, so `ALICE` and `alice` were different keyspaces on the same
site. [#293](https://github.com/ephpm/ephpm/pull/293) didn't patch the DB key. It made `resolve_site` return the key it
actually matched and derived everything else (DB file, wire credential, KV keyspace, OPcache vhost, temp root) from
that one value. A host that matches no site mints no identity at all.

### 7. Loopback has more than one address

The per-vhost eBPF policy lets a tenant run a sidecar on localhost and stops other tenants from connecting to its
ports. The `cgroup/connect` classifier recognized exactly `127.0.0.1` and `::1` as loopback. Linux treats all of
`127.0.0.0/8` as loopback, and an IPv4-mapped IPv6 address reaches the same IPv4 stack. Anything the classifier didn't
call loopback skipped the port-ownership check:

| Vector (Bob to Alice's sidecar) | Before | After |
|---|---|---|
| `127.0.0.1:port` | EPERM | EPERM |
| `::ffff:127.0.0.1:port` | **connected, read Alice's secret** | EPERM |
| `127.0.0.2:port` (Alice on 0.0.0.0) | **connected** | EPERM |
| `::ffff:127.0.0.2:port` | **connected** | EPERM |

[#489](https://github.com/ephpm/ephpm/pull/489) puts one shared classifier behind both `connect4` and `connect6`:

```c
static __always_inline int is_loopback4_be(__u32 ip_be)
{
    return (ip_be & bpf_htonl(0xff000000)) == bpf_htonl(0x7f000000);
}

static __always_inline int is_loopback6(const __u32 ip6[4])
{
    // ::1 (network order: high 96 bits zero, low word == 1).
    if (ip6[0] == 0 && ip6[1] == 0 && ip6[2] == 0 && ip6[3] == bpf_htonl(1))
        return 1;
    // ::ffff:0:0/96 — IPv4-mapped: first 80 bits zero, next 16 bits 0xffff.
    if (ip6[0] == 0 && ip6[1] == 0 && ip6[2] == bpf_htonl(0x0000ffff))
        return is_loopback4_be(ip6[3]);
    return 0;
}
```

Deprecated IPv4-*compatible* addresses (`::a.b.c.d`) are deliberately not folded in: Linux doesn't route them onto v4
loopback, and treating that prefix as "classify by embedded v4" would misclassify `::1` itself.

### 8. A login cookie that worked on every site

Previews sit behind a GitHub OAuth gate. One module does the OAuth round trip at login and mints an HS256 session with
a `site` claim; a second verifies that session on every request. The issuer always set `site`. The verifier never
checked it, so a session for preview A verified on every preview on the node.
[#488](https://github.com/ephpm/ephpm/pull/488) passes the request's canonical site key into the verifier as
`expected_site`, and a missing, non-string or mismatched claim fails closed. The issuer now takes its site from the
same canonical key instead of re-normalizing the raw Host, so the two agree by construction.

### The patterns

Looking back at the list, almost none of these were exotic. They fall into four groups:

- **The identity of a tenant was derived more than once.** Host to docroot, host to DB file, host to KV keyspace,
  issuer versus verifier. Every time two pieces of code derived "which tenant is this" separately, they eventually
  disagreed (#1, #2, #6, #8). The fix that stuck was one derivation, passed everywhere.
- **A check ran once and the thing it checked kept changing.** Validate-at-load with live symlink resolution (#3), and
  validate-before-strip (#2). If a value can change after you check it, check it where you use it.
- **The boundary was in the wrong layer.** `open_basedir` doesn't cover the SQLite engine (#5) or the static file
  path (#3). Any component running inside the process but outside PHP needs its own boundary.
- **"Loopback" and "keyword" aren't single values.** Address families and statement prefixes (#5, #7). Match the
  way the kernel or the engine will interpret it, not the way you'd write it.

The other habit that paid off was writing each fix with a test that fails on the old code. Several of the PRs above
say exactly that in their test sections, including temporarily reverting the fix to watch the test go red. When the
subject is security, a test you've never seen fail isn't telling you much.

### Links

- Host header traversal: [#280](https://github.com/ephpm/ephpm/pull/280)
- Suffix-stripped empty key: [#401](https://github.com/ephpm/ephpm/pull/401)
- Document-root symlink swap: [#398](https://github.com/ephpm/ephpm/pull/398)
- Per-vhost temp and sessions: [#285](https://github.com/ephpm/ephpm/pull/285)
- Per-site databases and the ATTACH screen: [#284](https://github.com/ephpm/ephpm/pull/284), [#287](https://github.com/ephpm/ephpm/pull/287), [#351](https://github.com/ephpm/ephpm/pull/351)
- One canonical site key: [#293](https://github.com/ephpm/ephpm/pull/293)
- eBPF loopback classifier: [#489](https://github.com/ephpm/ephpm/pull/489)
- Preview session bound to its site: [#488](https://github.com/ephpm/ephpm/pull/488)
