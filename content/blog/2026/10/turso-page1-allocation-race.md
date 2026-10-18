---
title: "Thirty-Two Connections and an Empty File: A Race in Turso's Page-1 Allocation"
description: "Why concurrent first writes to an empty database could panic Turso, and the fix."
date: 2026-10-18T12:30:00-07:00
tags: [ ephpm, turso, sqlite, rust, concurrency, debugging ]
categories: [ ephpm, databases, rust ]
images:
 - src: img/2026/10/turso-page1-allocation-race-races.svg
   alt: "Sequence diagram of two connections and init_lock: in race 1, B checks for an empty file, A publishes page 1 and releases the lock, then B takes the lock and panics on its assert; in race 2, A's lock is dropped while page 1 is being written, so B takes it and writes page 1 again"
   stretch: "none"
draft: true
---

[ePHPm](https://github.com/ephpm/ephpm) runs PHP inside a Rust server. For SQLite-style databases it uses
[Turso](https://github.com/tursodatabase/turso), a rewrite of SQLite in Rust that is still labeled beta. It sits behind
[litewire](https://github.com/ephpm/litewire). I recently hammered the per-site mode with fresh sites. A burst of first
requests to a brand-new site could panic inside the engine. This post is about where that panic came from and how to
fix it. How Turso got into ePHPm is in
[Clustered SQLite, Part 2](/blog/2026/08/clustered-sqlite-adding-turso/).

### Every site gets its own empty file

In per-site mode each virtual host gets its own database file. The file is created on first traffic, not up front.
Every PHP worker thread has its own connection to it. litewire sets a 5 second busy timeout on each connection.

Put those together and think about a new site's first burst of traffic. Many connections make their very first write
to the same 0-byte file at the same moment. The first write to an empty SQLite file has to create page 1. That's the
header page that every later read depends on. Exactly one connection should write it.

### The panic

The stress harness creates about 200 new sites per round. Each one gets 32 concurrent first writes. On ePHPm `1d47014`
with stock Turso 0.8.3, one round logged 59 engine panics and the next logged 46. Every one was the same line in
`turso_core`:

```text
thread 'ephpm-fpm-4' panicked at .../turso_core-0.8.3/storage/pager.rs:5648:17
```

On that commit the panic was contained. Before [#586](https://github.com/ephpm/ephpm/pull/586) it wasn't. The panic
unwound into the `extern "C"` bridge between PHP and Rust and aborted the whole server. One new site could take every
other site down with it. #586 catches it and turns it into a per-statement SQL error. It's merged and coming in the
next ePHPm release.

Containing the panic is the right thing for the server. It's not a fix. A statement that hits it may or may not have
been applied, and the PHP code gets an error. So I went looking for the cause.

### Reading `maybe_allocate_page1`

Line 5648 is the first line of `allocate_page1`'s `Start` state:

```rust
turso_assert!(!self.db_initialized());
```

It asserts that page 1 doesn't exist yet. A write gets there through `maybe_allocate_page1` in
`core/storage/pager.rs`. This is the version on upstream main. 0.8.3 is the same apart from the `shared.` prefix:

```rust
if !self.db_initialized() {
    if let Some(_lock) = self.shared.init_lock.try_lock() {
        return Ok(self.allocate_page1()?.map(|_| ()));
    }
    // Give a chance for the allocation to happen elsewhere
    io_yield_one!(Completion::new_yield());
}
Ok(IOResult::Done(()))
```

`init_lock` is shared by every connection to the database. `db_initialized()` is true once page 1 has been published.
Each connection keeps its own `allocate_page1` state machine. It goes `Start`, then `Writing`, then `Syncing`, then
`Done`. There are two problems in those eight lines.

### Race 1: check, then lock

The code checks `db_initialized()` and then takes the lock. It never checks again once it has the lock.

Say connection B checks and sees an empty file. Then connection A finishes. It publishes page 1 and drops the lock.
Now B's `try_lock` succeeds. B enters `allocate_page1` with its own state still at `Start`. The assert fires, because
the file isn't empty any more.

That's the panic. The window between the check and the `try_lock` is tiny, though. The second problem is what makes
it easy to hit.

### Race 2: the lock isn't held across IO

`_lock` is a local variable. It's dropped every time `allocate_page1` returns, and it returns every time it waits on
IO. That happens once after it submits the page-1 write. With a WAL it happens again after the fsync.

So while A waits on its write, the lock is free. B sees an empty file and takes the lock. B's own state is `Start`, so
B builds a page-1 image and writes it to offset 0. Then C does the same, and D. The lock doesn't make the allocation
exclusive. It only makes the synchronous part of each copy exclusive.

![Sequence diagram of connection A, init_lock and connection B. Race 1: B sees an empty file while A holds the lock; A publishes page 1 and releases it; B takes the lock and its assert that the file is empty fails. Race 2: A takes the lock, writes page 1 and waits on IO while the lock is free; B sees an empty file, takes the lock and writes page 1 again; A then fsyncs and publishes](/img/2026/10/turso-page1-allocation-race-races.svg)

The duplicate writes are identical bytes, so the file ends up fine. The losers are the problem. A connection that is
still in `Writing` or `Syncing` when someone else publishes page 1 never finishes its state machine. Its pinned copy of
page 1 is leaked.

More importantly, this is why race 1 gets hit at all. With as many as 26 of 32 connections writing page 1 for one file, the
lock changes hands constantly. Someone is always sitting in the gap between the check and the lock when the winner finishes.

### Why ePHPm finds this

The trigger needs a specific mix:

- a database per tenant, created on demand
- many threads, each with its own connection
- a busy timeout
- hundreds of fresh files

The busy timeout matters more than I expected. Without one, the losers fail fast with "database is locked" and stop
contending. I ran 5 files with 32 connections and no busy timeout. That gave 0 panics and 155 "locked" errors. With a
timeout they keep retrying, and every retry is another trip through the gap.

I searched Turso's tracker for the function names and the assert text. I found no open or closed issue or PR for this
race. The closest is [#2062](https://github.com/tursodatabase/turso/issues/2062), "CREATE TABLE isn't safe between two
connections". Its log shows two connections inside `maybe_allocate_page1` and `allocate_page1` at the same time. It was
closed in 2025 as "seems fixed now" without a change to this code. Its main symptom may well have had another cause,
so I won't claim it's the same bug.

The same harness found something else earlier. On Turso 0.8.2, switching to MVCC journal mode inside a transaction
destroyed the database. 0.8.3 refuses that switch, and ePHPm now blocks the rest.

### Reproducing it without ePHPm

I wanted to see it with just the crate. The repro opens one `turso::Database` per fresh file. It runs 32 tasks on 32
tokio worker threads. Each task connects, sets a busy timeout and waits at a barrier. Then they all run
`CREATE TABLE IF NOT EXISTS` and an `INSERT` at once. Afterwards each file is checked for row count and
`PRAGMA integrity_check`.

The stock crate panicked on 2 of 200 files. Each panic cost that connection's row. Integrity was fine everywhere.

Then I made a probe build of `turso_core` 0.8.3. It counts entries into the `Start` state per file. That's every time
a connection starts building page 1. A runtime switch picks the logic. The table is from a separate run on that probe build. That's why its
stock row shows 4 panics, not 2:

| Build | Page-1 allocations | Files with >1 writer | Panics |
|---|--:|--:|--:|
| Stock logic | 2,477 | 200 / 200 | 4 |
| Re-check only | 2,398 | 199 / 200 | 0 |
| Re-check + lock held | 200 | 0 / 200 | 0 |

![Three bar charts for 200 fresh files with 32 connections each. Page-1 allocations: stock 2,477, re-check only 2,398, re-check plus lock held 200. Files with more than one writer: 200, 199 and 0 of 200. Panics: 4, 0 and 0](/img/2026/10/turso-page1-allocation-race-repro.svg)

In short:

- **Every file** had more than one connection write page 1 on stock logic. The worst file had 26 of 32.
- **A re-check alone stops the panic.** It does nothing about the duplicate writes.
- **Holding the lock across the IO** gives exactly one allocation per file and no panics.

The panic rate here is lower than in ePHPm. That's 1 to 2 panics per 100 files, against 23 to 30 in the ePHPm
rounds. It's
the same assert. My guess is the wider IO window inside ePHPm keeps the lock free for longer.

### The fix

The lock moves into `allocate_page1`'s `Start` state. It's taken with `try_lock_arc`, which returns an owned guard.
The code re-checks `db_initialized()` under the lock. Then the guard is stored inside the `Writing` and `Syncing`
states:

```rust
AllocatePage1State::Start => {
    let Some(init_lock) = self.shared.init_lock.try_lock_arc() else {
        // Give a chance for the allocation to happen elsewhere
        io_yield_one!(Completion::new_yield());
    };
    // The pager that held the lock may have just finished.
    if self.db_initialized() {
        return Ok(IOResult::Done(()));
    }
    // ... build page 1 and submit the write ...
    *self.allocate_page1_state.write() = AllocatePage1State::Writing {
        page: page1,
        init_lock: Arc::new(init_lock),
    };
    io_yield_one!(c);
}
```

The lock now lives as long as the allocation does. It's released only after page 1 has been published. A connection
that takes it next always finds the database initialized. `maybe_allocate_page1` shrinks to the `db_initialized()`
fast path in front of `allocate_page1`.

![Sequence diagram with the fix: A takes the lock and re-checks, writes page 1 and keeps holding the lock through the IO; B sees an empty file but finds the lock held and yields; A fsyncs, publishes page 1 and only then releases the lock; B takes the lock, re-checks and finds the database already initialized](/img/2026/10/turso-page1-allocation-race-fix.svg)

Holding a lock across IO has a cost, so two more changes came with it:

- **An IO error resets the state to `Start`.** That drops the guard. Otherwise a failed write would leave the lock
  held in a state nothing drives again.
- **`set_initial_page_size` uses `try_lock`.** On main it did a blocking `lock()`. Now that an allocation can hold the
  lock across IO, blocking there could stall the thread that has to drive that IO. A held lock is treated like an
  initialized database, since the page size is already fixed at that point.

Upstream already uses a blocking lock-then-recheck in `set_initial_page_size`. It came in with
[#9315](https://github.com/tursodatabase/turso/pull/9315). It just never made it to `maybe_allocate_page1`.

### Tests that fail first

The second commit adds the tests and nothing else. The third adds the fix. So you can check out the middle commit and
watch them fail:

```bash
cargo test -p turso_core --features io_memory_yield allocate_page1_tests
```

| Test | Before the fix | After |
|---|--:|--:|
| Second writer waits while page 1 is in flight | 50 / 50 fail | 50 / 50 pass |
| Writer that loses the check-then-lock gap | 50 / 50 fail | 50 / 50 pass |
| Concurrent first writes allocate page 1 once | 5 / 5 fail | 5 / 5 pass |

The first test is race 2. Turso's `MemoryYieldIO` holds every IO completion until the test steps it, which parks A
right after its page-1 write. B must never start its own allocation. The second is race 1. A `#[cfg(test)]` hook runs
in B's gap between the check and the lock, and lets A allocate page 1 and commit. B must then see an initialized
database instead of panicking. The third puts 16 connections on each of 10 fresh files with real IO. The fix branches for main and for 0.8.3
give the same results.

### Back in ePHPm

Then I pointed the same ePHPm harness at the patched engine. That's ePHPm `cecefd7` built against the 0.8.3 fix branch.
I ran three rounds through the `ephpm_db_*` bridge and two through stock `pdo_mysql`.

![Bar chart of engine panics per round with 200 fresh site databases and 32 concurrent first writes each. Before, stock Turso 0.8.3 on ePHPm 1d47014: 59 and 46. After, the patched fork on ePHPm cecefd7: 0 in three bridge rounds and 0 in two pdo_mysql rounds](/img/2026/10/turso-page1-allocation-race-ephpm.svg)

Zero panics in all five rounds. All 1,000 fresh databases passed `integrity_check` in both Turso and `sqlite3`.

### If you can't patch the engine

There's a workaround on the consumer side. Make one connection do a write before anyone else does, and await it. A
`CREATE TABLE IF NOT EXISTS` on a scratch table is enough. After that, `db_initialized()` is true and every other
connection skips allocation entirely.

The catch is the word "await". The first write has to finish before any other connection starts its first write. So it
belongs inside whatever already serializes opening a site's database for the first time. Fire and forget doesn't count.
On stock logic, 100 files with that one awaited write gave 0 panics and exactly one allocation per file.

### What's next

- **Get the fix merged upstream.** It's up as [turso#10117](https://github.com/tursodatabase/turso/pull/10117).
- **Move ePHPm onto a Turso release with the fix** once there is one. Until then #586 keeps a panic from taking the
  server down.

### Caveats

- **The panic counts depend on timing.** The isolated repro ran on Windows. The ePHPm rounds ran a Linux build.
  Different IO gives a different window, so compare rows within a table.
- **Versions.** The race is in Turso 0.7.2, 0.8.2 and 0.8.3. It's also on upstream main at `f5090d05a` as I write
  this. The fix isn't merged yet.
- **#586 is unreleased.** It's merged on ePHPm main and coming in the next release.

### Links

- [luthermonson/turso `fix/page1-allocation-race`](https://github.com/luthermonson/turso/tree/fix/page1-allocation-race): the fix and its tests, on main
- [luthermonson/turso `fix/page1-allocation-race-0.8.3`](https://github.com/luthermonson/turso/tree/fix/page1-allocation-race-0.8.3): the same fix on 0.8.3
- [turso#10117](https://github.com/tursodatabase/turso/pull/10117): the upstream PR
- [ephpm#586](https://github.com/ephpm/ephpm/pull/586): containing engine panics in the bridge
- [turso#2062](https://github.com/tursodatabase/turso/issues/2062): the closest earlier report
- [turso#9315](https://github.com/tursodatabase/turso/pull/9315): lock-then-recheck in `set_initial_page_size`
- [Turso](https://github.com/tursodatabase/turso)
