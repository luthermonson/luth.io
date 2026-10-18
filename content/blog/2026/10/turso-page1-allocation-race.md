---
title: "Thirty-Two Connections and an Empty File: A Race in Turso's Page-1 Allocation"
description: "Why concurrent first writes to an empty database could panic Turso, and the fix."
date: 2026-10-18T14:30:00-07:00
tags: [ ephpm, turso, sqlite, rust, concurrency, debugging ]
categories: [ ephpm, databases, rust ]
images:
 - src: img/2026/10/turso-page1-allocation-race-races.svg
   alt: "Sequence diagram of two connections and init_lock: in race 1, B checks for an empty file, A publishes page 1 and releases the lock, then B takes the lock and panics on its assert; in race 2, A's lock is dropped while page 1 is being written, so B takes it and writes page 1 again"
   stretch: "none"
draft: true
---

[ePHPm](https://github.com/ephpm/ephpm) runs PHP inside a Rust server. For SQLite-style databases it uses
[Turso](https://github.com/tursodatabase/turso), a rewrite of SQLite in Rust that is still labeled beta. I recently
hammered ePHPm's per-site mode with brand-new sites. A burst of first requests to a new site could panic inside the
engine. This post is about where that panic came from and how to fix it. How Turso got into ePHPm is in
[Clustered SQLite, Part 2](/blog/2026/08/clustered-sqlite-adding-turso/).

### Every site gets its own empty file

In per-site mode each site gets its own database file. The file is created on first traffic. Every PHP worker thread
has its own connection to it, and each connection waits a few seconds when the database is busy instead of failing.

Now think about a new site's first burst of traffic. Many connections make their first write to the same empty file
at the same moment. The first write to an empty SQLite file has to create page 1. That's the header page everything
else depends on. Exactly one connection should write it.

### The panic

My stress test creates about 200 new sites per round and sends each one 32 first writes at once. One round logged 59
engine panics. The next logged 46. Every one was the same assert inside Turso, which checks that page 1 doesn't exist
yet.

Before [#586](https://github.com/ephpm/ephpm/pull/586), that panic took the whole ePHPm server down. One new site could
take every other site with it. #586 catches the panic and turns it into a normal SQL error. It's merged and coming in
the next ePHPm release.

That keeps the server up, but it's not a fix. The PHP code still gets an error, and it can't tell whether its write
happened. So I went looking for the cause.

### Two races in eight lines

The first write goes through this function in Turso's pager:

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

If page 1 doesn't exist, try to take a lock that all connections share, then create page 1. If someone else has the
lock, step aside and try again later. It looks fine. It has two problems.

**Race 1: check, then lock.** The code checks for page 1 before it takes the lock. It never checks again after. Say
connection B checks and sees an empty file. Then connection A finishes creating page 1 and lets go of the lock. B takes
the lock and starts creating page 1 too. The assert fires, because page 1 already exists. That's the panic.

**Race 2: the lock is let go during the write.** The lock only lives as long as one call. Writing page 1 to disk means
waiting on IO, and the call returns while it waits. So the lock is free for the whole write. Connection B sees an empty
file, takes the lock and writes its own page 1. Then C does, then D.

![Sequence diagram of connection A, init_lock and connection B. Race 1: B sees an empty file while A holds the lock; A publishes page 1 and releases it; B takes the lock and its assert that the file is empty fails. Race 2: A takes the lock, writes page 1 and waits on IO while the lock is free; B sees an empty file, takes the lock and writes page 1 again; A then fsyncs and publishes](/img/2026/10/turso-page1-allocation-race-races.svg)

The extra copies of page 1 are identical, so the file ends up fine. But race 2 is why race 1 happens so often. With
dozens of connections all creating page 1, the lock changes hands constantly. Someone is always in the gap between the
check and the lock when the winner finishes.

### Why ePHPm finds this

It takes a specific mix to hit this:

- a database per site, created on demand
- many threads, each with its own connection
- connections that wait and retry when the database is busy
- hundreds of fresh files

The waiting matters more than I expected. I ran the same test without it. The losing connections failed fast with
"database is locked" and gave up. That gave 0 panics. With waiting turned on, they keep retrying, and every retry is
another trip through the gap.

I didn't find an existing Turso issue for this race. The closest is
[#2062](https://github.com/tursodatabase/turso/issues/2062), "CREATE TABLE isn't safe between two connections". It was
closed in 2025 without a change to this code, and it may have had a different cause.

### Reproducing it with just Turso

To rule out ePHPm, I wrote a small Rust program that uses only the Turso crate. It opens 200 fresh files. Each one gets
32 connections that all do their first write at the same instant.

I also made a test build of Turso that counts how many times a connection starts creating page 1. It can switch
between three versions of the code:

| Version | Page 1 created | Files with >1 writer | Panics |
|---|--:|--:|--:|
| Stock | 2,477 | 200 / 200 | 4 |
| Re-check only | 2,398 | 199 / 200 | 0 |
| Re-check + hold the lock | 200 | 0 / 200 | 0 |

![Three bar charts for 200 fresh files with 32 connections each. Page-1 allocations: stock 2,477, re-check only 2,398, re-check plus lock held 200. Files with more than one writer: 200, 199 and 0 of 200. Panics: 4, 0 and 0](/img/2026/10/turso-page1-allocation-race-repro.svg)

In short:

- **On stock Turso, every file had more than one connection create page 1.** The worst file had 26 of 32.
- **Checking again under the lock stops the panic.** It does nothing about the extra writes.
- **Holding the lock through the write** gives exactly one page 1 per file and no panics.

Panics are rarer here than in ePHPm, but it's the same assert. My guess is that ePHPm's longer IO waits keep the lock
free for longer.

### The fix

The fix does both things from the table:

- **Hold the lock for the whole job.** The lock is taken once and kept until page 1 is on disk. Other connections step
  aside until then.
- **Check again under the lock.** If someone else finished first, there's nothing to do.

![Sequence diagram with the fix: A takes the lock and re-checks, writes page 1 and keeps holding the lock through the IO; B sees an empty file but finds the lock held and yields; A fsyncs, publishes page 1 and only then releases the lock; B takes the lock, re-checks and finds the database already initialized](/img/2026/10/turso-page1-allocation-race-fix.svg)

Holding a lock while waiting on IO has one risk. If the write fails or gets abandoned, the lock must still be let go.
Otherwise every other connection would wait forever. The fix lets go of the lock in those cases. Tests cover a dropped statement and a closed connection.

The PR also adds tests for both races. They fail on the old code every time and pass with the fix.

### Back in ePHPm

Then I ran the same ePHPm stress test against the fixed Turso. Five rounds, 200 new sites each.

![Bar chart of engine panics per round with 200 fresh site databases and 32 concurrent first writes each. Before, with stock Turso: 59 and 46. After, with the fix: 0 in three bridge rounds and 0 in two pdo_mysql rounds](/img/2026/10/turso-page1-allocation-race-ephpm.svg)

Zero panics in all five rounds. All 1,000 new databases passed an integrity check.

### If you can't patch Turso

There's a workaround. Have one connection do a single write to a new database and wait for it to finish. Then let the
other connections in. After that first write, page 1 exists and nobody else tries to create it.

The waiting is the important part. Fire and forget doesn't count. In my test, 100 files done this way on stock Turso
gave 0 panics.

### What's next

- **Get the fix merged upstream.** It's up as [turso#10117](https://github.com/tursodatabase/turso/pull/10117).
- **Move ePHPm onto a Turso release with the fix** once there is one. Until then #586 keeps a panic from taking the
  server down.

### Caveats

- **Timing matters.** Panic counts depend on IO speed, so compare numbers within one table, not across them.
- **Not merged yet.** The race is in current Turso releases and on main. The fix is in an open PR.
- **#586 is unreleased.** It's merged on ePHPm main and coming in the next release.

### Links

- [turso#10117](https://github.com/tursodatabase/turso/pull/10117): the fix and its tests
- [ephpm#586](https://github.com/ephpm/ephpm/pull/586): keeping an engine panic from taking ePHPm down
- [turso#2062](https://github.com/tursodatabase/turso/issues/2062): the closest earlier report
- [Turso](https://github.com/tursodatabase/turso)
