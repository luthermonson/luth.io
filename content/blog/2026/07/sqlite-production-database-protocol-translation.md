---
title: "SQLite as a Production Database via Protocol Translation"
description: "litewire speaks the MySQL wire protocol on one side and SQLite on the other, so WordPress and Laravel run against a .db file without a code change. Here's how it works and what it took."
date: 2026-07-25T20:40:00-07:00
tags: [ litewire, ephpm, sqlite, mysql, rust, wordpress, laravel ]
categories: [ databases, rust, ephpm ]
images:
 - src: img/2026/07/sqlite-production-database-protocol-translation-cover.svg
   alt: "A PHP app connecting over the MySQL wire protocol on port 3306 to litewire, which translates the SQL and runs it against a SQLite file"
   stretch: "none"
draft: false
---

Most of the PHP sites I'd put on [ePHPm](https://github.com/ephpm/ephpm) don't need a MySQL server. A brochure WordPress,
an internal Laravel tool, a preview environment that lives for an afternoon. What they do need is something that
*looks* like MySQL, because `mysqli` and `pdo_mysql` are the only database drivers their code has ever been pointed at.
I mentioned this in the [ePHPm pragmatism post](/blog/2026/06/ephpm-pragmatism/) as a one-paragraph "because". This is the
longer version, now that WordPress 7.0 and Laravel 13 both install and run end to end on it.

The piece doing the work is [litewire](https://github.com/ephpm/litewire), a standalone Rust project (MIT) that I
started at the end of March. It accepts MySQL, PostgreSQL, SQL Server (TDS) and Hrana HTTP connections, translates the
dialect to SQLite on the fly, and runs it against a pluggable backend. ePHPm embeds it as a library.

### The challenge to "you need MySQL"

The usual objection is that SQLite isn't a production database. For a lot of workloads that's just habit. A single-node
site with a few writes a second and mostly reads is squarely what SQLite in WAL mode is good at, and the operational
difference is large: there's no server to patch, no credentials to rotate, no connection limit to tune, and the backup
is a file.

What actually stops people is the application. WordPress doesn't have a SQLite driver in core, it has a
[plugin](https://wordpress.org/plugins/sqlite-database-integration/) that rewrites queries inside PHP. Laravel supports
SQLite, but every plugin, package and raw query in a real app was written and tested against MySQL. Asking someone to
port that is asking them not to use your thing. So the translation has to happen below the app, at the wire.

### The shape of it

```
PHP (pdo_mysql, mysqli)
        |  MySQL wire protocol, 127.0.0.1:3306
        v
  litewire-mysql      opensrv-mysql shim: handshake, COM_QUERY, prepared statements
        |
  litewire-translate  metadata emulation, no-ops, text pre-passes, sqlparser AST rewrite
        |
  litewire-backend    Backend::connect() -> one BackendConn per wire connection
        |
     SQLite (rusqlite, WAL)
```

The MySQL frontend is built on [opensrv-mysql](https://github.com/datafuselabs/opensrv), which handles the packet
framing and the handshake and calls into a shim for each command. PostgreSQL uses [pgwire](https://github.com/sunng87/pgwire).
SQL Server has no Rust server-side TDS crate (tiberius is client only) so that frontend is hand-rolled and
experimental. The Hrana frontend skips translation entirely since the SQL is already SQLite.

The translator is where the real work is. It runs a fixed pipeline on every statement:

1. **Metadata detection** before parsing, because some of the input isn't valid SQL to begin with. `SHOW FULL COLUMNS`,
   `DESCRIBE`, `INFORMATION_SCHEMA.*`, `SELECT @@vars` are answered by emulation queries against `sqlite_master` and
   `PRAGMA`.
2. **Transaction rewrites**: `START TRANSACTION`, `BEGIN WORK` and friends become `BEGIN`.
3. **No-ops**: `SET NAMES`, `SET time_zone`, `SET sql_mode`, isolation level changes. They return OK without touching
   SQLite.
4. **Text pre-passes** for MySQL syntax [sqlparser-rs](https://github.com/apache/datafusion-sqlparser-rs) can't
   represent, like `bigint(20)` display widths and `KEY k (col(191))` index prefix lengths.
5. **Parse, rewrite the AST, emit** with sqlparser's `Display`.

Rather than describe the rewrites, here's what the translator does with real WordPress and Laravel statements. I ran
these through `litewire_translate::translate(sql, Dialect::MySQL)` at litewire `699e634`:

```text
IN : CREATE TABLE wp_usermeta (
       umeta_id bigint(20) unsigned NOT NULL auto_increment,
       user_id bigint(20) unsigned NOT NULL default '0',
       meta_key varchar(255) default NULL,
       meta_value longtext,
       PRIMARY KEY  (umeta_id),
       KEY user_id (user_id),
       KEY meta_key (meta_key(191))
     ) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_520_ci
OUT: CREATE TABLE wp_usermeta (umeta_id INTEGER NOT NULL, user_id INTEGER NOT NULL DEFAULT '0',
       meta_key TEXT DEFAULT NULL, meta_value TEXT, PRIMARY KEY (umeta_id))

IN : INSERT INTO `wp_options` (`option_name`, `option_value`, `autoload`)
       VALUES ('siteurl', 'http://example.test', 'yes')
       ON DUPLICATE KEY UPDATE `option_name` = VALUES(`option_name`),
       `option_value` = VALUES(`option_value`), `autoload` = VALUES(`autoload`)
OUT: INSERT INTO `wp_options` (`option_name`, `option_value`, `autoload`)
       VALUES ('siteurl', 'http://example.test', 'yes')
       ON CONFLICT DO UPDATE SET `option_name` = excluded.`option_name`,
       `option_value` = excluded.`option_value`, `autoload` = excluded.`autoload`

IN : SELECT ... FROM wp_posts WHERE ... ORDER BY wp_posts.post_date DESC LIMIT 10, 10
OUT: SELECT ... FROM wp_posts WHERE ... ORDER BY wp_posts.post_date DESC LIMIT 10 OFFSET 10

IN : alter table `users` add unique `users_email_unique`(`email`)
OUT: CREATE UNIQUE INDEX `users_email_unique` ON `users`(`email`)

IN : START TRANSACTION          OUT: BEGIN
IN : SET NAMES utf8mb4          OUT: <noop>
IN : SELECT LAST_INSERT_ID()    OUT: SELECT last_insert_rowid()
IN : SHOW FULL COLUMNS FROM `wp_options`
OUT: <metadata ShowColumns { table: "wp_options" }>
```

Two of those are worth a closer look. The `wp_usermeta` table keeps `PRIMARY KEY (umeta_id)` on an `INTEGER` column,
which SQLite treats as a rowid alias, so `auto_increment` behaves the way WordPress expects without the keyword. And
the two inline `KEY` lines are gone. SQLite has no inline secondary index syntax, and the translator drops them rather
than failing the whole `CREATE TABLE`. That's a deliberate trade, written down in the source as "a performance concern,
not a correctness one", and it's one I'd like to revisit by emitting real `CREATE INDEX` statements the way the
`ALTER TABLE ... ADD UNIQUE` path already does.

The upsert one is load-bearing: `add_option()` uses that exact shape (and `update_option()` falls back to it for a new option), so if `VALUES(col)` doesn't
become `excluded.col` WordPress can't save a setting.

All of that sits behind a bounded LRU cache keyed on the SQL text. A WordPress-shaped join measured 39 µs per
translation cold and 277 ns warm in release mode, in the PR that added the cache
([litewire#5](https://github.com/ephpm/litewire/pull/5)). PHP apps send the same few hundred statement shapes over and
over, so the parser runs rarely after warmup.

### Wiring it into ePHPm

On the ePHPm side the config is a file path. The MySQL listener defaults to `127.0.0.1:3306`, and you only need the
proxy block if you want to move it or add the Hrana listener for external tools:

```toml
[db.sqlite]
path = "/var/lib/ephpm/myapp.db"

[db.sqlite.proxy]
mysql_listen = "127.0.0.1:3306"
hrana_listen = "127.0.0.1:8080"  # optional
```

The app's config stays the MySQL config it already had. For Laravel that's the stock `.env`:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=ephpm
DB_PASSWORD=
```

In single-node mode litewire runs in-process with the rusqlite backend, so there's no child process and nothing else
listening. Clustered mode swaps the backend for a client to a replicated SQLite server, which is a separate story.
There's also an experimental `engine = "turso"` knob that runs the same frontends over the Turso engine instead of
SQLite's C library. And if you do want a real MySQL, `[db.mysql]` turns the same listener into a pooling proxy and
SQLite drops out of the picture entirely.

### What it took to make WordPress actually install

For most of litewire's life, "WordPress works" was a claim in the README that nothing tested. I took it out in July
([litewire#4](https://github.com/ephpm/litewire/pull/4)) and replaced it with what the end-to-end tests actually cover.
Then I put full installs through it. ePHPm's Turso evaluation needed a WordPress and Laravel suite run against both
engines anyway, so I ran a WordPress 7.0.1 `wp core install` and a Laravel 13 `php artisan migrate` through the MySQL
frontend on the then-default SQLite engine as the control. Neither got past installation:

```
Error: Error: WordPress 7.0.1 requires MySQL 5.5.5 or higher
```

opensrv-mysql's default handshake banner is `5.1.10-alpha-msql-proxy`, and WordPress 6.5 and later refuse to talk to
anything that old. Dropping back to WordPress 6.4.3 got further and then died in `dbDelta`:

```
wp_usermeta -> SQL translation error: SQL parse error: sql parser error: Expected: ), found: ( at Line: 9, Column: 24
```

That's the `meta_key(191)` index prefix length, and it hit 7 of the 12 core tables. Laravel failed on the very first
statement of `migrate`, because the table options went straight through to SQLite:

```
SQLSTATE[HY000]: General error: 1105 SQLite error: near "DEFAULT": syntax error in
CREATE TABLE `migrations` (...) DEFAULT CHARACTER SET = utf8mb4 COLLATE = 'utf8mb4_unicode_ci' at offset 115
```
 The
five WordPress tables that did parse were never actually created, because their inline `KEY` constraints were emitted
verbatim.

The full list came to nine gaps: the handshake version, index prefix lengths, table options, inline `KEY`/`INDEX`,
`ON DUPLICATE KEY UPDATE ... VALUES(col)`, a mis-shaped `SHOW FULL COLUMNS` (WordPress read every core option back as
an empty string), `SQL_CALC_FOUND_ROWS`, `information_schema` existence probes that Laravel uses before migrating, and
qualified `UPDATE t SET t.col = ...` references. The important thing about the list is that both engines failed it
identically, which made it obvious every one of them lived in the wire and translate layer, not the database.

They landed together in [litewire#11](https://github.com/ephpm/litewire/pull/11) with 13 regression tests. After it,
the WordPress suite (install, theme render, posts, comments, a hundred options, generated content, search-replace,
plugin DDL) and the Laravel suite (migrate, seed, CRUD, transaction commit and rollback, migrate rollback, re-migrate)
went 20/20 green on both engines. ePHPm picked up the pin in [ephpm#190](https://github.com/ephpm/ephpm/pull/190).

### Two bugs a test suite wouldn't have found

Getting the install green was the visible part. Two problems that would have hurt in production came out of putting
real PHP traffic through it in the same couple of weeks.

**Transactions weren't per client.** The rusqlite backend held one `Arc<Mutex<Connection>>` shared by every wire
connection. The MySQL handler tracked `in_transaction` per client in Rust, but the actual `BEGIN` ran on the shared
handle. Client A's `BEGIN` meant client B's next `INSERT` landed inside A's transaction, and either one's `COMMIT`
finalized both. The mutex also serialized every query, which threw away WAL's concurrent readers before they ever
reached SQLite. The fix in [litewire#6](https://github.com/ephpm/litewire/pull/6) turned `Backend` into a factory:

```rust
#[async_trait]
pub trait Backend: Send + Sync + 'static {
    /// Open a new session. Each BackendConn has its own transaction state.
    async fn connect(&self) -> Result<Box<dyn BackendConn>, BackendError>;
    // stateless query/execute/describe_columns kept as defaults
}
```

Every accepted connection calls `connect()` once and holds its own rusqlite connection to the same file. SQLite's file
locking coordinates writers, `busy_timeout` (5 s default) handles `SQLITE_BUSY`, and litewire defaults each session to
`synchronous=NORMAL`, the usual setting for WAL mode. One wrinkle: `open_in_memory()` gives each connection a *different* database, and the shared-cache URI
workaround defeats WAL, so `Rusqlite::memory()` now quietly backs itself with a temp file.

**Every SELECT took 44 ms.** Measured from ePHPm with PHP PDO on loopback, a point `SELECT` was 44.01 ms p50 and an
`INSERT` was 1.36 ms. The asymmetry is the clue. An `INSERT` reply is one OK packet; a result set is a column count,
column definitions, an EOF, rows and another EOF, and opensrv-mysql wrote each of those as its own TCP segment. Something
on the loopback path sat on those small segments for about 40 ms, which is the classic Nagle/delayed-ACK stall. I never
got a packet capture inside the test host to prove which side was holding, and setting `TCP_NODELAY` on the server side
([litewire#3](https://github.com/ephpm/litewire/pull/3)) was the right thing to do and didn't cure it on its own. Wrapping the write half in
a 64 KiB `BufWriter` so the whole result set goes out as one write did
([litewire#7](https://github.com/ephpm/litewire/pull/7)):

| op | before | after |
|----|-------:|------:|
| point `SELECT` p50 | 44.01 ms | 0.23 ms |
| `INSERT` p50 | 1.36 ms | 0.30 ms |

Those numbers are from the PR, measured on a rebuilt ePHPm image. It only ever reproduced through PHP;
`mysql_async` and the `mysql` CLI against the same server never showed it, and I don't have a satisfying explanation for
the difference yet. mysqlnd sets `TCP_NODELAY` on its sockets too, so it isn't that.

### Where it doesn't make sense

I'd rather be specific about this than have someone find out the hard way:

- **One writer at a time.** WAL gives you concurrent readers and a single writer. If the app does heavy concurrent
  writes, run MySQL or Postgres and use ePHPm's proxy mode instead.
- **One process owns the file.** Don't point a second server or a cron job's own SQLite client at the live database.
- **Not every MySQL behavior is emulated.** `SET autocommit = 0` is a no-op with a warning, because half-emulating
  MySQL's implicit transactions with a hidden `BEGIN` would be worse than saying so. `LOCK TABLES` is also a no-op with a
  warning, since SQLite only has file-level locking. Inline secondary indexes are dropped, as above.
- **PostgreSQL and TDS are CRUD-level.** The MySQL frontend is the one that gets tested against real applications.

Inside those lines it's a good fit: CI and preview environments, a single VPS, edge boxes, small production sites. The
same `.db` file runs under `ephpm dev` on a laptop and on the server, and the app never knows.

Next up is the long tail. Passing an install suite is not the same thing as behaving like MySQL on every query a
plugin might send, and I expect real traffic to find the next round.

### Links

- [litewire](https://github.com/ephpm/litewire)
- [ePHPm](https://github.com/ephpm/ephpm)
- [litewire#11: WordPress and Laravel run over the MySQL frontend](https://github.com/ephpm/litewire/pull/11)
- [litewire#6: per-wire-connection sessions](https://github.com/ephpm/litewire/pull/6)
- [litewire#7: coalescing the result-set write](https://github.com/ephpm/litewire/pull/7)
- [litewire#5: translate cache](https://github.com/ephpm/litewire/pull/5)
- [opensrv-mysql](https://github.com/datafuselabs/opensrv)
- [sqlparser-rs](https://github.com/apache/datafusion-sqlparser-rs)
