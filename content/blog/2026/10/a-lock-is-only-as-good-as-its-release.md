---
title: "A Lock Is Only as Good as Its Release"
description: "SETNX gets all the attention, but the bugs in ePHPm's KV locks were all on the release side: blind deletes, a missing compare-and-delete, and a cluster where two nodes could both win."
date: 2026-10-10T21:15:00-07:00
tags: [ ephpm, php, rust, locking, redis, clustering ]
categories: [ ephpm, php ]
images:
 - src: img/2026/10/a-lock-is-only-as-good-as-its-release-cover.svg
   alt: "Timeline: request A's lock expires, B acquires it, A's blind DEL frees B's lock and C gets in"
   stretch: "none"
draft: true
---

<!-- TODO(luther): this post depends on ephpm#562, which merged on 2026-10-11 (after this draft's date). Re-date to
1-21 days after the merge, re-check the merged diff against the section on cluster routing, and flip draft to false. -->

Every "build a lock on Redis" tutorial spends its time on acquisition: `SET key value NX PX 30000`, one winner, done. The
release gets one line, `DEL key`, and that line is where every lock bug I've had in ePHPm lived. Over about three months
the same mistake showed up in the native session handler, in two of the Composer wrapper packages, and finally in the
cluster, each time in a slightly different costume.

### Version one: SETNX and a blind DEL

The native session handler got locking in [#111](https://github.com/ephpm/ephpm/pull/111) back in July. Without it, two
concurrent requests with the same session cookie both read the blob, both change their copy, and the second write
silently wins. The design is the one php-fpm's files handler uses, built on the KV store:

- `PS_READ` takes `session_lock:<sid>` with `set_nx` and a 30 second TTL.
- On contention it spins with backoff, 10 ms doubling to a 100 ms cap, for up to 30 seconds.
- If it still can't get the lock it logs a warning and proceeds without it. A lost update is bad, a deadlocked PHP thread
  is worse.
- `PS_CLOSE`, which PHP calls after the write even when the request bailed out, releases it.

The release was this:

```c
if (g_kv_ops.del) {
    char *lock_key = ephpm_session_make_prefixed_key(
        EPHPM_SESSION_LOCK_PREFIX, EPHPM_SESSION_LOCK_PREFIX_LEN,
        session_lock_sid, session_lock_sid_len,
        stack, sizeof(stack), &used_heap);
    (void)g_kv_ops.del(lock_key);
    ephpm_session_key_free(lock_key, used_heap);
}
```

And the lock value was the string `"1"`. The comment above it was honest about it:

```c
 * KNOWN LIMITATION (accepted for v1): if a holder outlives the 30s TTL,
 * the lock expires and another request may acquire it. Our release path
 * is an unconditional DEL — the KV ops table has no compare-and-delete —
 * so the original holder would then release the *new* holder's lock,
 * letting a third request in early.
```

That's the diagram at the top. A holds the lock and runs past its TTL. The lock expires, B acquires it and believes it
has the session to itself. A finishes and deletes the key, which is now B's lock. C walks straight in while B is still
working. A TTL lock already has the problem that A is running unlocked once it overruns. The blind release adds a second
problem: it breaks the lock for B and C, who did everything right.

### The wrappers made the same choice

The [wrapper packages](/blog/2026/09/calling-rust-from-php-without-a-socket-part-2/) that put Laravel, Symfony and
WordPress on the KV store needed locks too. `cache-laravel` got an atomic lock on September 10th, modelled on Laravel's
own `MemcachedLock`. Acquisition goes through `ephpm_kv_setnx`. Release reads the owner and deletes if it matches:

```php
public function release(): bool
{
    if ($this->isOwnedByCurrentProcess()) {
        return $this->ops->del($this->name) > 0;
    }

    return false;
}
```

That's better than a blind delete because it checks the owner, but the check and the delete are two calls. Between them
the lock can expire and be re-acquired, and the delete removes the new holder's lock anyway. The docblock says so plainly:
"The ePHPm SAPI has no compare-and-delete / CAS primitive". The userland `session-handler` package's opt-in lock makes the
same trade and documents it the same way.

The fix for both sides of this already existed in Redis land. You can't fix release in PHP, so the store needs a
primitive.

### Compare-and-delete

[#501](https://github.com/ephpm/ephpm/pull/501) merged the day after the Laravel lock was written. It adds `del_if_eq`:
delete the key only while its value is still exactly this token. It's on the `Store`, as a RESP command (`DELIFEQ key
token`), and as `ephpm_kv_del_if_eq()` in PHP, appended to the end of the ops table like every other addition.

The implementation is short because DashMap already has the right primitive. `remove_if` evaluates its predicate while
holding the shard's write lock, so the compare and the delete are one step:

```rust
pub fn del_if_eq(&self, key: &str, token: &[u8]) -> bool {
    let removed = self.data.remove_if(key, |_, entry| {
        if entry.is_expired() {
            return false;
        }
        if entry.compressed {
            decompress_value(&entry.data, self.config.compression.algo)
                .is_some_and(|plain| plain.as_ref() == token)
        } else {
            entry.data.as_ref() == token
        }
    });
    let Some((_, old)) = removed else {
        return false;
    };
    self.mem_sub(old.mem_size);
    self.ttl_keys.remove(key);
    self.notify_write(key);
    if let Some(rep) = self.active_replicator() {
        rep.replicate_remove_published(key.to_string());
    }
    true
}
```

Two details matter. An expired entry counts as absent and never matches, so a late release can't remove a lock that's
logically gone and about to be reaped. And the compare is against the decompressed value, so turning on KV compression
doesn't quietly make every release fail.

The PHP pattern becomes: generate a random token per acquisition, store it as the value, release with the token.

```php
$token = bin2hex(random_bytes(16));
if (ephpm_kv_setnx('lock:nightly-report', $token, 30)) {
    try {
        build_report();
    } finally {
        ephpm_kv_del_if_eq('lock:nightly-report', $token);
    }
}
```

Here's the scenario from the diagram as a real script against the v0.10.6 release: A takes the lock with a 1 second TTL,
sleeps 1.2 seconds, B acquires, A releases, and C tries.

```text
$ curl -s localhost:8131/lock.php?release=del
A acquires (1s TTL):     true
B acquires (30s TTL):    true
A releases with del:     1
C tries to acquire:      true
lock holder now:         'c'

$ curl -s localhost:8131/lock.php?release=del_if_eq
A acquires (1s TTL):     true
B acquires (30s TTL):    true
A releases with del_if_eq: false
C tries to acquire:      false
lock holder now:         'B'
```

With `del`, A's release returns 1 (it deleted something, just not its own lock) and C gets in on top of B. With
`del_if_eq`, A's release is a no-op, B keeps the lock and C is turned away.

### Then the cluster

That fixed one node. ePHPm can also run as a cluster, with the KV store replicated between nodes over gossip and a data
plane, and the clustering comment on `del_if_eq` was careful to say what it did and didn't promise: "per-node atomic;
cluster-wide it is last-arrival-wins". The same was true of `set_nx`. Each node ran the check-and-insert on its own map
and then published the result.

Spell that out for a lock and it's not a lock. Two nodes get a request for the same key at about the same time. Each
checks its own map, each sees absent, each inserts, each publishes, and both callers were told they acquired it. The
release side had the matching bug. A holder whose lock had expired and been granted to someone on another node still had
its old token in its local copy, so `del_if_eq` matched against the stale copy, succeeded, and broadcast a tombstone that
deleted the new holder's lock everywhere.

[#562](https://github.com/ephpm/ephpm/pull/562) changes where the decision is made. Both `SETNX` and `DELIFEQ` go to the
key's primary owner, `hash(key) % sorted alive nodes` (the same owner the large-value tier already used), and the owner
decides under its own shard lock. The `Replicator` trait gets two hooks that default to "decide locally", which is still
right for a single node:

```rust
fn route_set_nx(&self, key: &str, value: &[u8], ttl: Option<Duration>) -> Routed<SetNxOutcome> {
    let _ = (key, value, ttl);
    Routed::Local
}

fn route_del_if_eq(&self, key: &str, token: &[u8]) -> Routed<bool> {
    let _ = (key, token);
    Routed::Local
}
```

and `set_nx_outcome` asks before touching its own map:

```rust
if let Some(rep) = self.active_replicator() {
    match rep.route_set_nx(&key, &value, ttl) {
        Routed::Local => {}
        Routed::Owner(outcome) => return outcome,
        Routed::OwnerUnreachable => return SetNxOutcome::OwnerUnreachable,
    }
}
self.set_nx_outcome_local(key, value, ttl)
```

The owner runs the `_local` variant and never re-routes, so two nodes that briefly disagree about who owns a key can't
bounce a request back and forth. When the owner can't be reached the caller gets a failure, not a guess: `set_nx`
returns false and RESP answers `-CLUSTERDOWN ... the outcome is unknown` instead of `:0`, because a request that timed
out may still have landed on the owner. Falling back to a local decision would bring back the exact bug the routing
exists to fix.

The PR's tests are the convincing part. A real three-node in-process cluster test, with routing stubbed back to `Local`,
produces three winners for one lock, a stale release that returns `Ok(true)`, and an unreachable owner that still reports
`Inserted`. With the routing in place all three pass, along with a 32-thread, four-"node" test that expects exactly one
winner.

I want to be precise about what this is: single-arbiter routing, not consensus. Every node computes the owner from its
own view of live membership. During a join, a leave, or the few seconds it takes gossip to declare a node dead, two nodes
can name different owners. A new owner only knows about locks that had already replicated to it. For session locks and
cron de-duplication that's a fine trade. For anything where two holders would corrupt data, you want a fencing token or
a real consensus store, and the docs now say that instead of implying it.

The same PR finally closes the session handler's known limitation. The lock value is a random 128-bit token from PHP's
CSPRNG, hex encoded, instead of `"1"`, and release (including the out-of-memory undo path) is `del_if_eq` against it.
The new test `session_lock_release_cannot_delete_a_lock_it_no_longer_holds` fails against the old wrapper with
`lock value ... got "1"`. Acquisition changed too: an unreachable owner is no longer treated as contention, so instead
of retrying for up to 30 seconds the request proceeds after one timed-out attempt, with a warning. It still holds its
token, so if the grant landed on the owner anyway, `PS_CLOSE` releases it.

### The wrappers lag the server

This is the part that's easy to forget. A primitive in the server doesn't fix anything until callers use it. As of this
writing `cache-laravel`'s lock and the userland `session-handler` still release with a plain delete and still say there's
no compare-and-delete, because they were written against the API as it was the day before. The acquisition side has
mostly caught up (`cache-wordpress` moved `wp_cache_add()` to `setnx` in the same week), and the release side is next.

<!-- TODO(luther): if cache-laravel / session-handler adopt ephpm_kv_del_if_eq before this goes out, replace this
section with the change. -->

The general lesson I took from it: when you review a lock, read the release first. Acquisition with `SETNX` is hard to
get wrong. Release needs a token you generated, a compare-and-delete that's atomic where the decision is made, and in a
cluster, agreement about where that is.

### Links

- [ephpm #111: multi-tenant defaults and KV session locking](https://github.com/ephpm/ephpm/pull/111)
- [ephpm #501: compare-and-delete (del_if_eq / DELIFEQ)](https://github.com/ephpm/ephpm/pull/501)
- [ephpm #562: owner-arbitrated SETNX/DELIFEQ and the token-released session lock](https://github.com/ephpm/ephpm/pull/562)
- [ephpm/cache-laravel](https://github.com/ephpm/cache-laravel) and [ephpm/session-handler](https://github.com/ephpm/session-handler)
- [Calling Rust From PHP Without a Socket, Part 2](/blog/2026/09/calling-rust-from-php-without-a-socket-part-2/)
- [Redis docs: distributed locks](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
