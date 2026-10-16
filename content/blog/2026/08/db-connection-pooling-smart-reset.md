---
title: "Database Connection Pooling with Smart Reset"
description: "PHP opens a new MySQL connection on every request, but ePHPm reuses them and cleans up only the ones that need it. Here's how that works, two bugs it had and how much load it takes off MySQL."
date: 2026-08-09T16:25:00-07:00
tags: [ ephpm, mysql, php, rust, connection pooling, performance ]
categories: [ ephpm, databases ]
images:
 - src: img/2026/08/db-connection-pooling-smart-reset-cover.svg
   alt: "PHP request connecting to the ePHPm MySQL proxy, which lends it a pooled backend connection and resets it only if the session was dirtied"
   stretch: "none"
draft: false
---

PHP's share-nothing model is one of its best properties. It has one expensive side effect: a request can't keep a
database connection for the next request. php-fpm gives you `PDO::ATTR_PERSISTENT` and `mysqli`'s `p:` hosts. But
those are per-worker-process. PDO's keep whatever session state the last request left behind. mysqli at least resets
the session with `COM_CHANGE_USER` on reuse. And almost nobody turns them on. So in practice every PHP request does a
TCP connect, a MySQL handshake and an auth exchange. Then it maybe sends some `SET`s, runs its queries and sends
`COM_QUIT`. Multiply that by your request rate and MySQL spends a real share of its time authenticating clients it
just said goodbye to.

[ePHPm](https://github.com/ephpm/ephpm) puts a pooling proxy in the same process as PHP. PHP connects to
`127.0.0.1:3306` exactly like it would to MySQL. The proxy lends it a connection that's already authenticated. This
post covers how that works and what "smart" reset means. It also covers the two bugs that made the default
configuration unusable until June, and then again until this month. It ends with some numbers on what it does to the
database.

### The proxy in 30 seconds

Configuration is a URL and a listen address:

```toml
[db.mysql]
url = "mysql://app:secret@db.internal:3306/app"
listen = "127.0.0.1:3306"
min_connections = 2
max_connections = 20
reset_strategy = "smart"   # or "always" / "never"
```

When PHP connects, the proxy doesn't touch the backend at all for the handshake. It sends PHP a synthetic
`HandshakeV10` built from the real server's saved metadata and a fresh challenge. It reads PHP's `HandshakeResponse41`
and answers `OK`. It doesn't validate the credentials. The listener is meant for loopback, and the docs say in bold
not to bind it anywhere else. The real credentials only ever travel on the pool's own connections to MySQL. Those
support both `mysql_native_password` and `caching_sha2_password`, including the RSA full-auth path for MySQL 8's
default plugin.

Then the proxy checks out a pooled backend and relays packets in both directions until PHP's session ends.

### What has to be reset, and when

A pooled connection carries session state from whoever used it last. That includes user variables, `SET SESSION`
changes, temp tables and prepared statements. It also includes an open transaction and `LAST_INSERT_ID()`. Handing
that to the next PHP request is how you get the worst kind of bug. That's the one where request B sees request A's
data. MySQL 5.7 added `COM_RESET_CONNECTION` for exactly this. It's one packet (`0x1F`) and one `OK` back, and the
session is clean without re-authenticating.

```rust
/// Resets: transaction state, user variables, prepared statements, temporary
/// tables, and `LAST_INSERT_ID()`. Available since MySQL 5.7.
async fn reset_connection(mut stream: TcpStream) -> Result<TcpStream, DbError> {
    // COM_RESET_CONNECTION = 0x1F, sequence = 0.
    write_packet(&mut stream, 0, &[0x1F]).await?;
    let (_, resp) = read_packet(&mut stream).await?;
    if resp.first() != Some(&0x00) {
        return Err(DbError::Protocol("COM_RESET_CONNECTION did not return OK".into()));
    }
    Ok(stream)
}
```

That's a round trip. The three strategies are about when to pay it:

- **`always`**: reset every connection when the session ends. It's correct and costs one extra round trip per PHP
  request.
- **`never`**: park the connection as-is. It's the fastest, and only safe if you control every query that runs (dev
  boxes).
- **`smart`** (the default): track a dirty bit per session and reset only if something could have changed state.

The dirty tracking happens in the relay loop. That loop already has to parse packet boundaries. More on why below.
Each command PHP sends gets classified:

```rust
let writeish = match cmd {
    0x0E /* COM_PING */ => false,
    COM_QUERY => {
        let sql = std::str::from_utf8(&payload[1..]).unwrap_or("");
        !matches!(classify_mysql_query(sql), QueryKind::Read)
    }
    _ => true,
};
if writeish {
    dirty_w.store(true, Ordering::Relaxed);
}
```

and `classify_mysql_query` looks at the first keyword:

- `SELECT`, `SHOW`, `EXPLAIN` and `DESCRIBE` are reads, unless the statement contains `FOR UPDATE` or `FOR SHARE`.
- `BEGIN`/`START` and `COMMIT`/`ROLLBACK` are transaction boundaries.
- Everything else is treated as a write, because "unknown" has to mean "dirty."

Any command that isn't `COM_QUERY` or `COM_PING` also marks the session dirty. That includes prepared statements and
`COM_INIT_DB`.

When PHP's session ends:

```rust
match outcome.backend {
    Some(backend) => match self.reset_strategy {
        ResetStrategy::Never => checkout.return_to_pool(backend),
        ResetStrategy::Always => checkout.return_with_reset(backend).await,
        ResetStrategy::Smart => {
            if outcome.dirty {
                checkout.return_with_reset(backend).await;
            } else {
                checkout.return_to_pool(backend);
            }
        }
    },
    None => checkout.retire(), // backend broke mid-session: never recycle it
}
```

### The honest limits of "smart"

First-keyword classification is cheap, and it's a heuristic. Two things it can't see:

- **A `SELECT` with side effects.** `SELECT @x := 1`, `SELECT GET_LOCK('job', 10)`, `SELECT LAST_INSERT_ID(42)` and
  `SELECT ... INTO @var` all start with `SELECT` and all change session state. A session that does only that is
  "clean" and gets parked without a reset. The named lock case is the nasty one. The lock is held by the connection.
  So the next PHP request to draw it from the pool inherits a lock it never took. If your code uses `GET_LOCK`, use
  `always`.
- **Most real apps are dirty on connect.** WordPress's `wpdb` sets the charset on every connection, and mysqlnd sends
  that as a `SET NAMES`. On a stock MySQL 8 `sql_mode` it also runs `SET SESSION sql_mode=...` to drop the modes it
  doesn't support. All of that happens before its first real query. Those are writes by this definition, and correctly
  so, since they change session state. So for WordPress, `smart` behaves exactly like `always`. The win only shows up
  for code that connects and reads.

Neither of these makes `smart` wrong as a default. But it's not magic, and the second one is why the numbers below
have two workloads.

### Bug one: smart reset hung on the first query

Until June, `smart` didn't use the relay above. It used a per-command routing loop built for read/write splitting.
That loop acquired a backend from the pool *for every command*. Two problems compounded
([#97](https://github.com/ephpm/ephpm/pull/97)):

1. Its response forwarder stopped at the first `EOF` packet. In a MySQL result set that's the one that ends the column
   definitions, not the rows. The rows sat unread in the backend's buffer and the client waited forever. The first
   query `mysql_async` sends on every connection is `SELECT @@max_allowed_packet,@@wait_timeout`. That's two columns
   and one row, which was enough.
2. Underneath that, picking a new backend per command destroys session continuity. `SET @v = 42` lands on backend A
   and `SELECT @v` on backend B. Transactions and prepared statements break the same way. Bug 1 just hid it.

The integration tests only exercised `always`, which used a different path. The WordPress compose example had quietly
shipped with `reset_strategy = "always"` as a workaround. That tells you how it was found. The fix was to use the
per-command loop only when read/write splitting is actually on. Otherwise the proxy holds one backend for the whole
client session.

### Bug two: the pool served corpses on quiet sites

The second one is my favorite bug of the year. It got *worse* the less traffic you had
([#221](https://github.com/ephpm/ephpm/pull/221)). With the default config, a site would serve a couple of requests
and then fail every request after that with `MySQL server has gone away`. Permanently. The number of successes tracked
the pool's warm connections. Once they were used up, every request drew a corpse.

PHP doesn't know the proxy pools anything. At the end of every request PDO destroys its handle and mysqlnd sends
`COM_QUIT`, which means "close this connection." The relay classified `COM_QUIT` as not-dirty and then forwarded it to
the backend anyway. MySQL closed the pooled connection. A clean session then parked that dead socket as healthy. The
next request drew it and the write failed with a broken pipe. The error path mapped that to a successful session end
and parked the corpse again.

Under load you'd never see it. A connection returned and immediately re-acquired hasn't been noticed as closed yet. On
a quiet site a connection sits idle for a moment between requests, and every one of them is dead. The fix has three
parts:

- `COM_QUIT` is intercepted and never forwarded. It ends the *client* session, full stop. This is also why the relay
  has to parse packet boundaries. There's no correct dumb byte-copy for a pooled backend.
- A connection that has been idle for at least 500ms gets a `COM_PING` on checkout. Below that it can't have died for
  any reason a ping would detect. Busy pools never pay for it. Quiet sites always do, and they're the ones that expose
  dead connections.
- A backend that errors mid-session is retired, never re-parked.

### What it does to MySQL

To put numbers on it I ran MySQL 8.4 in a container with three copies of the ePHPm v0.6.2 proxy in front of it, one
per reset strategy. I drove them from PHP itself (`ephpm php`, PHP 8.4, `pdo_mysql`). Each "request" is `new PDO`, the
session's statements, then `$pdo = null`. There were two session shapes, 2,000 sessions each, run sequentially:

- **read-only**: one primary-key `SELECT`
- **WordPress-shaped**: `SET NAMES utf8mb4`, `SET SESSION sql_mode=...`, then the same `SELECT`

Before and after each run I read MySQL's own counters. That's `Connections` from `SHOW GLOBAL STATUS`, plus the
`statement/com/Reset Connection` and `statement/com/Ping` counts from `performance_schema`.

Medians of three runs, 2,000 sessions each:

| Target | Session | µs per session | MySQL `Connections` | `COM_RESET_CONNECTION` | `COM_PING` |
|--------|---------|---------------:|--------------------:|-----------------------:|-----------:|
| MySQL directly | read-only | 479 | 2,000 | 0 | 0 |
| proxy, `smart` | read-only | 467 | 0 | 0 | 4 |
| proxy, `always` | read-only | 456 | 0 | 2,000 | 4 |
| proxy, `never` | read-only | 470 | 0 | 0 | 4 |
| MySQL directly | WordPress-shaped | 664 | 2,000 | 0 | 0 |
| proxy, `smart` | WordPress-shaped | 816 | 0 | 2,000 | 4 |
| proxy, `always` | WordPress-shaped | 822 | 0 | 2,000 | 4 |
| proxy, `never` | WordPress-shaped | 834 | 0 | 0 | 4 |

(One of the three `smart` WordPress-shaped runs saw one new backend connection and 16 pings. The other two saw 0 and
4-5. I didn't chase it.)

What that says, in order of how much it matters:

**MySQL stopped accepting connections.** There were 6,000 PHP sessions per strategy and session shape, and one new
backend connection in total. That was the one odd run. The pool's warm connections did all the work. Outside that one
run MySQL never ran an authentication exchange after startup. That's the whole point of the proxy, and it does exactly
that. The four or five pings per run are mostly the four warm connections being validated on first checkout. They sat
idle between runs, so that's the quiet-site fix doing its job.

**`smart` is only smart for read-only sessions.** Read-only: zero resets. WordPress-shaped: 2,000 resets, identical to
`always`. `SET NAMES` and `SET SESSION sql_mode` dirty every session. If you run WordPress through this proxy, `smart`
and `always` are the same setting.

**The reset isn't on the client's clock.** `always` was no slower than `smart` from PHP's point of view. The proxy
sends `COM_RESET_CONNECTION` after PHP's session has already ended. So the cost lands on the pool, not on the request.
The connection just comes back a round trip later. Under a saturated pool that round trip would start to matter. At
this load it didn't show up at all.

**On loopback, pooling doesn't make PHP faster, and the extra hop can make it slower.** This was the one I didn't
expect going in. A direct MySQL connect plus auth on loopback is cheap here. So a read-only session costs about the
same either way. The WordPress-shaped session got *slower* through the proxy, 664 to about 820µs. Each of its three
statements now crosses two sockets instead of one. That extra hop costs more than the connect it saved. Across a real
network with TLS to the database, a fresh connection is several round trips before the first query. The balance should
tip the other way there. But I didn't measure that here, so I'm not going to claim a number for it.

So the honest pitch for the pool is the database side:

- No connection storm when traffic spikes.
- No per-request auth on MySQL.
- A hard cap (`max_connections`) on how many backend connections a fleet of PHP workers can hold.
- Session state that can't leak between requests.

Lower request latency isn't on that list unless your database is far away.

### How I measured

- AMD Ryzen 9 5950X, Windows 11 + WSL2. MySQL 8.4 (`docker.io/library/mysql:8.4`) ran in Podman with host networking
  on port 3307. ePHPm `v0.6.2+php8.4.23` proxies ran on 3311/3312/3313 (`smart`/`always`/`never`, `min_connections =
  4`, `max_connections = 20`) in the Ubuntu WSL distro. The client is `ephpm php bench.php <port> <profile> 2000` from
  the same binary. Everything is loopback on one machine, so absolute latencies are lower than over a real network. If
  anything, the difference between direct and pooled connects is understated.
- Three repetitions per cell, with medians reported. Counter deltas exclude the one connection the script uses to read them.

### Links

- [ePHPm](https://github.com/ephpm/ephpm), the proxy is `crates/ephpm-db`
- [#97](https://github.com/ephpm/ephpm/pull/97) smart reset hang, [#221](https://github.com/ephpm/ephpm/pull/221) `COM_QUIT` and dead pooled connections
- [MySQL `COM_RESET_CONNECTION`](https://dev.mysql.com/doc/dev/mysql-server/latest/page_protocol_com_reset_connection.html)
- [WordPress `wpdb::db_connect()`](https://developer.wordpress.org/reference/classes/wpdb/db_connect/), which sets the charset and `sql_mode` on every connection
