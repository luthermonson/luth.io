---
title: "Faking MySQL's Quirks Well Enough to Fool WordPress"
description: "Passing WordPress's install tests didn't mean litewire behaved like MySQL. This is the second round of fixes: the small MySQL behaviors that WordPress and Laravel quietly depend on."
date: 2026-08-16T15:50:00-07:00
tags: [ litewire, ephpm, sqlite, mysql, rust, wordpress ]
categories: [ databases, ephpm, php ]
images:
 - src: img/2026/08/faking-mysql-quirks-wordpress-cover.svg
   alt: "A table of MySQL behaviors WordPress relies on, with what litewire answered before and after the August fixes"
   stretch: "none"
draft: false
---

Last month I wrote about [running SQLite behind a MySQL socket](/blog/2026/07/sqlite-production-database-protocol-translation/)
with [litewire](https://github.com/ephpm/litewire), and ended on "passing an install suite is not the same thing as
behaving like MySQL". That turned out to be accurate. On August 10th a few things happened at once: I put a real
WordPress 7.0.3 site through a benchmark on ePHPm, A/B tested the new in-process database bridge against the wire path, and started building
[ephpm/db-wordpress](https://github.com/ephpm/db-wordpress), a `wpdb` drop-in that talks to the database in-process
instead of over a socket. Between them they produced six litewire issues in about three hours. None of them broke
the install. All of them made WordPress quietly do the wrong thing.

The common thread is that the client is code I don't control. WordPress, Laravel, Doctrine and PDO all make decisions
based on what MySQL says back, and some of those decisions are made by matching strings. If litewire answers in the
wrong shape, the app doesn't crash. It just takes the wrong branch.

### FOUND_ROWS() was always zero

`WP_Query` paginates the old way:

```sql
SELECT SQL_CALC_FOUND_ROWS wp_posts.ID FROM wp_posts WHERE ... ORDER BY wp_posts.post_date DESC LIMIT 10, 10;
SELECT FOUND_ROWS();
```

MySQL 8 deprecates the hint, WordPress still uses it unconditionally, and the second statement is how `found_posts`
gets set. In July the translator stripped the hint (sqlparser can't parse it) and rewrote `FOUND_ROWS()` into
something SQLite would accept, which got the install suite past it. Here's what it did with that second statement:

```text
IN : SELECT FOUND_ROWS()
OUT: SELECT abs(0)
```

So the benchmark probe ran a calc query that returned 56 rows and got `0` back from `FOUND_ROWS()`. That's
`found_posts = 0` on every WordPress running on ePHPm: `X-WP-Total: 0` on every REST list response and broken totals on
every archive page ([litewire#23](https://github.com/ephpm/litewire/issues/23)). The fix that made the install pass was
the one that hid the bug.

`FOUND_ROWS()` is per-connection state, so it can't live in the stateless translator. It lives in
`litewire-session`'s `Session`, next to `in_transaction`, which means the wire path and ePHPm's in-process bridge both
get it. After a statement carrying the hint succeeds, the session runs one extra `COUNT(*)`. The count statement is
derived through the AST, not by string surgery: take the fully translated statement, remove `LIMIT`, `ORDER BY` and
`FETCH`, and wrap it.

```rust
let stmt = crate::rewrite_statement(statements.pop()?, dialect).ok()?;
let Statement::Query(mut query) = stmt else {
    return None;
};

if limit_clause_has_placeholder(query.limit_clause.as_ref()) {
    return None;
}
query.limit_clause = None;
query.order_by = None;
query.fetch = None;

Some(format!("SELECT COUNT(*) FROM ({query})"))
```

For the `WP_Query` statement above, at litewire `cb707ab`, that produces:

```sql
SELECT COUNT(*) FROM (SELECT wp_posts.ID FROM wp_posts WHERE 1 = 1
  AND wp_posts.post_type = 'post' AND wp_posts.post_status = 'publish')
```

It's bound with the same parameter list, so `WHERE` placeholders keep working. A bare `SELECT FOUND_ROWS()` is then
answered from session state without touching the backend. Statements without the hint pay one allocation-free ASCII
scan and zero extra round trips, and there's a counting-backend test that pins that down. Two cases fall back to the
returned row count instead: the hint inside multi-statement input, and a `LIMIT` bound as a placeholder, since stripping
it would desynchronize the positional parameters.

The end-to-end tests for this immediately found a worse bug. `rewrite_limit_clause` took the limit clause out of the
AST to check for MySQL's `LIMIT offset, count` form, and only put it back if it matched:

```rust
fn rewrite_limit_clause(query: &mut sqlparser::ast::Query) {
    if let Some(LimitClause::OffsetCommaLimit { offset, limit }) = query.limit_clause.take() {
        query.limit_clause = Some(LimitClause::LimitOffset { /* LIMIT count OFFSET offset */ });
    }
}
```

Everything else was dropped on the floor. Running it at the July revision:

```text
SELECT * FROM wp_posts LIMIT 5            => SELECT * FROM wp_posts
SELECT * FROM wp_posts LIMIT 5 OFFSET 10  => SELECT * FROM wp_posts
SELECT * FROM wp_posts LIMIT 10, 5        => SELECT * FROM wp_posts LIMIT 5 OFFSET 10
```

WordPress mostly emits the comma form, which is how this hid. An existing test even hedged with "sqlparser may or may
not preserve LIMIT". The fix is one `other => query.limit_clause = other` arm, and it rode along in
[litewire#26](https://github.com/ephpm/litewire/pull/26) because `FOUND_ROWS()` is meaningless if `LIMIT` doesn't apply.

### Error codes are an API

SQLite's `no such table: wp_foo` fell through to MySQL's catch-all, 1105 / `HY000`. That carries no information, and
frameworks branch on the real code: Doctrine raises `TableNotFoundException` off 1146, and MySQL clients expect
`42S02` to come with it. Mapping it was a substring match, and it also catches the Turso engine's wording, which
prefixes the same text with `Parse error: `.

Duplicate keys were already 1062 / `23000`, but with SQLite's message, and that's the more interesting failure.
Frameworks mostly key off the code, so 1062 / `23000` already got Laravel and Doctrine onto the right branch, but
anything that greps the message for MySQL's `Duplicate entry '...' for key '...'` shape never matched
`UNIQUE constraint failed: users.email`. Now the message gets a MySQL-shaped prefix and keeps SQLite's words:

```
Duplicate entry '<unknown>' for key 'users.email' (UNIQUE constraint failed: users.email)
```

There are two honest gaps in that line. SQLite's message doesn't include the duplicated value and the statement's
parameters aren't available at this layer, so the slot says `<unknown>` instead of inventing one. And MySQL names the
index where SQLite names the columns. For a single-column unique index, and for the name-the-index-after-the-column
convention WordPress and Laravel both follow, those are the same thing.

The unit tests passed the first time and the real-backend tests didn't:

```
got: Duplicate entry '<unknown>' for key 'UNIQUE constraint failed: users.email'
     (SQLite error: UNIQUE constraint failed: users.email)
```

rusqlite wraps the message in `SQLite error: `, so splitting on the first `": "` grabbed the wrapper's colon. The
column list is now found by the `constraint failed: ` marker instead. The unit tests fed the classifier an idealized
string and could never have seen it ([litewire#31](https://github.com/ephpm/litewire/pull/31)).

### Three versions, and one that never ran

A client can ask for the server version three ways, and each was answered by its own string literal:

| Path | Read by | Was |
|---|---|---|
| wire handshake | `mysqli_get_server_info()`, `PDO::ATTR_SERVER_VERSION` | `8.0.36-litewire` |
| `SELECT VERSION()` | `$wpdb->db_version()` | `8.0.0-litewire` |
| `@@version` | capability checks | `8.0.0-litewire` |

They all read one constant now. Writing the end-to-end test for that showed `SELECT VERSION()` had never returned a
version at all. It returned an error:

```
SQLite error: wrong number of arguments to function coalesce()
in SELECT coalesce('8.0.0-litewire') at offset 7
```

The rewrite pass only sees the `Function` node, not its parent expression, so a MySQL built-in with no SQLite analogue
can't become a bare literal. It has to stay a function call that returns a constant, and the call chosen was
`coalesce()`, which SQLite rejects with fewer than two arguments. `VERSION`, `DATABASE`, `SCHEMA`, `USER`,
`CURRENT_USER`, `SESSION_USER`, `SYSTEM_USER` and `CONNECTION_ID` were all emitted that way. The funny part is that the
`FOUND_ROWS` arm sitting between the broken ones had a comment explaining exactly this ("coalesce and max need >= 2
args") and used `abs(0)`. The unit tests asserted on the emitted SQL string and never executed it. Constant folding
now goes through two helpers, `constant_text` (`trim('...')`) and `constant_number` (`abs(n)`), so the single-argument
rule lives in one place ([litewire#30](https://github.com/ephpm/litewire/pull/30)):

```text
IN : SELECT VERSION()
OUT: SELECT trim('8.0.36-litewire')
```

### A stray OK packet

This one came from an A/B benchmark of the in-process bridge against the wire path. Under PHP persistent connections
(`PDO::ATTR_PERSISTENT`) the wire path desynced twice, with nothing in the server log:

```
Wrong COM_STMT_PREPARE response size. Received 7
Packets out of order. Expected 1 received 2
2006 MySQL server has gone away
```

opensrv-mysql 0.7.0 parses nine client commands. Every other command byte falls into one catch-all that writes a
default OK packet and logs nothing. OK is the right reply to some of those and the wrong shape for others, and mysqlnd
sends housekeeping commands on persistent handles. A reply of the wrong shape leaves the client one packet behind, and
every response after that belongs to the previous command. `Received 7` is exactly the size of that OK payload.

litewire can't change opensrv's command loop, so [litewire#28](https://github.com/ephpm/litewire/pull/28) puts a
`CommandFilter` between the socket and opensrv's reader that looks at the first byte of every command packet:

- Commands opensrv handles go through byte for byte.
- `COM_STMT_RESET`, `COM_RESET_CONNECTION` and `COM_CHANGE_USER` are rewritten into a `COM_PING`, which is opensrv's own
  OK path, so the reply is encoded for whatever capability flags the connection negotiated. The reset ones also raise
  a flag the handler drains before the next statement, rolling back an open transaction and dropping prepared
  statements. Before this, a reset was acknowledged and did nothing.
- Anything else gets `ER_UNKNOWN_COM_ERROR` (1047), which is what a real server sends, and a `warn` log with the byte.

Command packets are told apart from handshake and continuation packets by sequence id, which resets to 0 at the start
of every command. `COM_CHANGE_USER` is only a no-op on the single-backend path. When litewire is choosing a backend per
tenant during the handshake, answering OK would tell the client it's someone else while the connection keeps talking
to the original tenant's database, so there it's refused.

I want to be straight about this one: the original desync was probabilistic and never reduced to a repro, so the PR
doesn't claim to fix that specific trace. What it does guarantee is that no command reaches the catch-all anymore.

### REGEXP, YEAR() and LIKE

The last three were feature degradations WordPress logged and moved past.

`populate_options()` cleans up stale RSS transients at install time with
`WHERE option_name REGEXP '^rss_[0-9a-f]{32}(_ts)?$'`. SQLite parses `REGEXP` but ships nothing behind it: the operator
calls a `regexp(pattern, subject)` function the host has to register. I could have translated anchored patterns to
`GLOB`, but that covers only the patterns that happen to be expressible and silently mis-answers the rest. Registering a
real function is faithful for the whole pattern language, so every rusqlite session now gets one backed by the `regex`
crate, with the compiled pattern cached in SQLite's aux-data slot so a constant pattern compiles once per statement
instead of once per row. The Turso engine already registers an equivalent built-in on the same crate. The differences
from MySQL are documented: `regex` syntax rather than ICU, and case-sensitive, where MySQL usually isn't under
`utf8mb4_general_ci`. I picked matching Turso over matching MySQL so one query can't mean two things depending on the
backend.

`YEAR()`, `MONTH()` and `DAYOFMONTH()` are used by `wp_old_slug_redirect` and the 404 permalink guesser. The cast is
the part that matters:

```text
IN : SELECT * FROM wp_posts WHERE YEAR(post_date) = 2024 AND MONTH(post_date) = 3
OUT: SELECT * FROM wp_posts WHERE CAST(strftime('%Y', post_date) AS INTEGER) = 2024
       AND CAST(strftime('%m', post_date) AS INTEGER) = 3
```

`strftime` returns text, and SQLite applies no affinity to a function result, so `'2024' = 2024` is false. A bare
`strftime` translation would parse, run without error and match nothing, which is worse than the error it replaced
([litewire#29](https://github.com/ephpm/litewire/pull/29)).

And `LIKE`. MySQL's `LIKE` uses backslash as its escape character by default (only `NO_BACKSLASH_ESCAPES` turns it
off); SQLite has no default escape character at all. `wpdb::esc_like('50%')` produces `50\%`, which on SQLite matched a
literal backslash followed by a live wildcard. Every MySQL-dialect `LIKE` without an explicit `ESCAPE` now gets one, through the AST visitor so it reaches
subqueries and `UPDATE`/`DELETE` predicates too ([litewire#25](https://github.com/ephpm/litewire/pull/25)):

```text
IN : SELECT option_name FROM wp_options WHERE option_name LIKE '50\%'
OUT: SELECT option_name FROM wp_options WHERE option_name LIKE '50\%' ESCAPE '\'
```

### What I took away from it

Every one of these had unit tests around it that passed. The translator tests asserted on the SQL string that came
out; the error map tests fed it the string SQLite was supposed to produce. What found the bugs was executing the
statement against a real backend or driving a real `mysql_async` client over the wire and asserting on what came back.
That's now the rule in the repo: a wire-visible fix lands with a wire-level test reproducing what the client saw.

ePHPm picked these up in two pin bumps, [ephpm#265](https://github.com/ephpm/ephpm/pull/265) and [ephpm#297](https://github.com/ephpm/ephpm/pull/297). Next is the other half of the August work, which was less about
WordPress and more about not shipping a use-after-free.

### Links

- [litewire](https://github.com/ephpm/litewire)
- [litewire#26: SQL_CALC_FOUND_ROWS / FOUND_ROWS() and the dropped LIMIT](https://github.com/ephpm/litewire/pull/26)
- [litewire#31: 1146 and a MySQL-shaped 1062](https://github.com/ephpm/litewire/pull/31)
- [litewire#30: one server version](https://github.com/ephpm/litewire/pull/30)
- [litewire#28: the stray OK packet](https://github.com/ephpm/litewire/pull/28)
- [litewire#29: REGEXP and YEAR/MONTH/DAYOFMONTH](https://github.com/ephpm/litewire/pull/29)
- [litewire#25: LIKE escape](https://github.com/ephpm/litewire/pull/25)
- [ephpm/db-wordpress](https://github.com/ephpm/db-wordpress)
