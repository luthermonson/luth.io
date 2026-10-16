---
title: "Calling Rust From PHP Without a Socket, Part 2: Making Existing Code Use It"
description: "Thirteen Composer packages let existing Redis and database code in WordPress, Laravel and Symfony use ePHPm's built-in versions. This post covers what's in them and the bugs they turned up."
date: 2026-09-21T19:55:00-07:00
tags: [ ephpm, php, redis, wordpress, laravel, composer ]
categories: [ ephpm, php ]
series: [ "Calling Rust From PHP Without a Socket" ]
images:
 - src: img/2026/09/calling-rust-from-php-without-a-socket-part-2-cover.svg
   alt: "The APIs PHP apps already call, mapped through ePHPm wrapper packages onto ephpm_kv_* and ephpm_db_*"
   stretch: "none"
draft: false
---

[Part 1](/blog/2026/09/calling-rust-from-php-without-a-socket-part-1/) covered the plumbing. ePHPm registers
`ephpm_kv_*` and `ephpm_db_*` functions that call straight into Rust. A cache read through one costs about a tenth of a
microsecond instead of a loopback round trip. That's nice and also useless on its own. Nobody is going to rewrite their
WordPress object cache or their Laravel cache store around a function called `ephpm_kv_set()`. Applications talk to
existing APIs instead:

- Redis through phpredis or Predis
- the cache through PSR-6/16 or `wp_cache_*`
- the database through `wpdb`, mysqli, Eloquent or DBAL

So there's a set of Composer packages in the [ephpm org](https://github.com/ephpm) whose whole job is to sit under those
APIs and route them to the native functions. This post covers what's in them and the rule that decides when they switch
on. It also covers the bugs that only showed up once real applications ran on top.

### The packages

There are 13 of them, seven on the KV side and six on the database side:

| Package | What it stands in for |
| --- | --- |
| `phpredis-shim` | a userland `\Redis` class (the phpredis API) |
| `predis-connection` | a Predis `Connection`, so `Predis\Client` works with no socket |
| `cache-wordpress` | `wp-content/object-cache.php`, the `WP_Object_Cache` drop-in |
| `cache` | PSR-16 `SimpleCache` and PSR-6 pools |
| `cache-laravel` | a Laravel cache store, auto-discovered |
| `cache-symfony` | a Symfony Cache adapter |
| `session-handler` | a userland `SessionHandlerInterface` |
| `db` | the base typed wrapper, exception hierarchy and IDE stubs for `ephpm_db_*` |
| `db-wordpress` | `wp-content/db.php`, a `wpdb` with no mysqli |
| `mysqli-shim` | the `mysqli` API, procedural and OO |
| `db-laravel` | a Laravel database driver (`'driver' => 'ephpm'`) |
| `db-doctrine` | a Doctrine DBAL 4 driver |
| `db-codeigniter` | a CodeIgniter 4 database driver |

They aren't on Packagist. You add the GitHub repo as a Composer `vcs` repository and require it. The Predis one is the
easiest to show because it's the smallest change to an existing app:

```php
use Ephpm\Predis\KvConnection;
use Predis\Client;

$redis = new Client('ephpm:', [
    'connections' => ['ephpm' => KvConnection::class],
]);

$redis->incrby('hits:home', 1);
$visits = $redis->get('hits:home');
```

Everything else in the app keeps calling Predis. `KvConnection` takes the command objects Predis builds. Instead of
serializing them to RESP, it dispatches on the command ID and calls `ephpm_kv_*`.

### When a shim switches on

Every package that impersonates a global API follows the same rule: **alias the global name only when the real
extension isn't loaded**. The namespaced class always works. Here's the entire activation file for the phpredis shim,
loaded through Composer's `autoload.files`:

```php
use Ephpm\Phpredis\Redis;
use Ephpm\Phpredis\RedisException;

if (\extension_loaded('redis')) {
    return;
}

if (!\class_exists('Redis', false)) {
    \class_alias(Redis::class, 'Redis');
}

if (!\class_exists('RedisException', false)) {
    \class_alias(RedisException::class, 'RedisException');
}
```

A userland class can't replace a loaded extension's class anyway. Anything cleverer would just fatal on a box that has
phpredis. The same rule has a funny consequence for the mysqli shim. The ePHPm SDK compiles mysqli in, so under ePHPm
the global `mysqli` is always the real extension. Only the namespaced shim API is active. The global surface is for
plain PHP builds without ext-mysqli. For WordPress that doesn't matter, because the `db.php` drop-in replaces `$wpdb`
instead of replacing mysqli.

The WordPress drop-ins also have to survive being copied to a host that isn't ePHPm. `db.php` checks for
`ephpm_db_query()` first. If it's missing, it logs a notice and returns without setting `$wpdb`. WordPress then builds
the stock mysqli `wpdb` exactly as if the drop-in weren't there. `object-cache.php` does the same when it can't load its
classes. A drop-in that fatals your site when you move hosts is worse than no drop-in.

### Throwing beats a wrong answer

The KV store behind `ephpm_kv_*` models strings, counters and TTLs. It doesn't have pipelines, `MULTI`, Lua, pub/sub
or key iteration. Through the native functions it doesn't have hashes, lists and sets either. The obvious way to write
a compatibility shim is to fake those. Run a "pipeline" as a sequence of calls, implement `SCAN` by returning nothing,
and so on. Every one of those fakes gives callers an answer that's wrong in a way they can't detect.

So the phpredis shim throws instead:

```php
public function multi(int $mode = self::MULTI): never
{
    throw NotImplementedException::for('MULTI/transactions', 'the KV store has no transaction/pipeline state');
}

public function scan(mixed &$iterator, mixed $pattern = null, int $count = 0, ?string $type = null): never
{
    throw NotImplementedException::for('SCAN', 'the KV store exposes no key iteration');
}
```

`NotImplementedException` extends `BadMethodCallException`, deliberately not `RedisException`. Plenty of application
code wraps Redis calls in `catch (\RedisException $e)` and treats it as "Redis is down, carry on without cache". Calling
a command the shim doesn't implement is a bug in your setup, not a transient outage. It shouldn't get swallowed by that
catch block. The Predis connection does the same with `SET ... XX`, `GET`, `KEEPTTL` and friends.

The WordPress object cache had the same question with `wp_cache_flush_group()`. With no key scan there is no way to
delete every key in a persistent group. So `flush_group()` clears the in-request copy and returns `false` for persistent
groups. `wp_cache_supports('flush_group')` says `false`. Plugins that check support fall back to their own invalidation.

### The semantics fixes

These are the ones I'd point at if someone asked "why can't I just write a quick adapter".

**`wp_cache_add()` is a mutex.** The first version of `add()` in `cache-wordpress` checked whether the key existed and
then set it. WordPress code uses `wp_cache_add()` as a cheap mutex. Only the first caller's add succeeds, so whoever
gets `true` does the work. With check-then-set, two concurrent requests could both see "absent" and both win. `19226c2`
routes the persistent path through `ephpm_kv_setnx`. That's one insert-if-absent under the store's per-shard lock:

```php
$inserted = $this->ops->setnx(
    $this->build_key($key, $group),
    $this->serialize($data),
    \max(0, $ttl),
);
if (!$inserted) {
    return false;
}
```

**`(int) false` is zero.** `ephpm_kv_incr_by()` returns `false` when the stored value isn't an integer. Five of the
packages had `return (int) \ephpm_kv_incr_by($key, $delta);`, which turns that error into `0`. A counter that silently
resets to zero is worse than one that throws. The same one-line bug got the same fix in each of them in the September
audit:

```php
$result = \ephpm_kv_incr_by($key, $delta);
if ($result === false) {
    throw new \RuntimeException(
        "value at key '{$key}' is not an integer and cannot be incremented"
    );
}
return (int) $result;
```

**`'O\'Reilly'` broke the installer.** `wpdb` without mysqli has to escape strings itself. The first version did what
`mysql_real_escape_string()` does and backslash-escaped the single quote. litewire's tenant path rejected `\'` as
malformed SQL while accepting every other backslash escape. So any content with an apostrophe failed. The first place
that showed up was the twentytwentyfive theme's block-pattern transient, which 500'd the installer during its last step.
`35db6a8` doubles the quote (`''`), which both MySQL and SQLite accept. It keeps doubling backslashes so a string still
can't be broken out of. Doctrine's `quote()` and the mysqli shim's `real_escape_string()` got the same change later.

**Native ints printed "0 responses".** The bridge hands back real PHP ints and floats. mysqli doesn't. Unless
`MYSQLI_OPT_INT_AND_FLOAT_NATIVE` is set, every scalar column comes back as a string, and `wpdb` never sets it.
WordPress core depends on that. The comments-title block does `'0' === get_comments_number()`. With native ints every
post with no comments rendered a "0 responses" heading. I found it by rendering the same pages under stock `wpdb` and
under the drop-in, then diffing the HTML byte for byte. `ad746e9` stringifies non-NULL scalars during row hydration.
NULL stays null, like mysqli.

**Stop guessing from the first keyword.** Every adapter that implements a single `query()` entry point had to pick
between `ephpm_db_query()` (rows) and `ephpm_db_execute()` (affected rows). They all did it by looking at the first word
of the SQL. That breaks on `WITH ... INSERT`, `INSERT ... RETURNING` and anything else that doesn't fit.
[#357](https://github.com/ephpm/ephpm/pull/357) added `ephpm_db_run()`. It reports `has_rowset` from the statement that
executed. Doctrine deleted its `QueryClassifier` class outright (`e991c20`). `db-wordpress` and the mysqli shim moved
over too.

### Known limitations

I found two things while writing this up. Both come from the boundary rather than the wrappers' own logic.

Keys go through the native functions as C strings and must be valid UTF-8, as covered in part 1. In the Predis
connection, a plain `SET` that returns `false` is reported as Redis' OOM error. The reasoning is that an overwrite can
only fail for memory. But a key that isn't valid UTF-8 also gets `false` from the bridge. It surfaces as
`OOM command not allowed when used memory > maxmemory.`, which will send you looking in the wrong place.

The WordPress object cache stores integers raw so the native `incr` keeps working on them. It applies that to
integer-looking strings too:

```php
private function serialize(mixed $value): string
{
    if (\is_int($value)) {
        return (string) $value;
    }
    if (\is_string($value) && \preg_match('/^-?\d+$/', $value) === 1) {
        return $value;
    }
    return \serialize($value);
}

private function unserialize(string $value): mixed
{
    if (\preg_match('/^-?\d+$/', $value) === 1) {
        return (int) $value;
    }
    return \unserialize($value);
}
```

On the next request (once it's past the in-request cache), a string `"02134"` comes back as the int `2134`. `"5"` comes
back as `5`. For most WordPress caching nobody notices. A ZIP code or a zero-padded ID cached as a string does notice.
Distinguishing the two needs a type tag in the stored value, which is a format change. For now it's a documented quirk
rather than a quick patch.

The Laravel lock acquires atomically with `setnx`. But it releases by reading the owner and then deleting. That's the
same non-atomic release Laravel's own Memcached lock has. It was written the day before the store grew a
compare-and-delete, so it says "no CAS in the SAPI" in its docblock. That one's a story of its own.

### Trying it

The [wordpress-sample](https://github.com/ephpm/wordpress-sample) repo puts most of this together:

- WordPress on the embedded database through `db-wordpress`
- the object cache on the KV store through `cache-wordpress`
- a WebSocket broadcast from PHP through `ephpm_ws_broadcast`

It runs WooCommerce and Elementor, which between them exercise more MySQL-isms than WordPress core does.

The theme across all thirteen packages is the same as part 1. The boundary is the easy part. The work goes into
matching what callers already depend on: string typing, atomic `add()`, errors that are errors.

### Links

- [Part 1: The Native Functions](/blog/2026/09/calling-rust-from-php-without-a-socket-part-1/)
- KV side: [phpredis-shim](https://github.com/ephpm/phpredis-shim), [predis-connection](https://github.com/ephpm/predis-connection), [cache-wordpress](https://github.com/ephpm/cache-wordpress), [cache](https://github.com/ephpm/cache), [cache-laravel](https://github.com/ephpm/cache-laravel), [cache-symfony](https://github.com/ephpm/cache-symfony), [session-handler](https://github.com/ephpm/session-handler)
- DB side: [db](https://github.com/ephpm/db), [db-wordpress](https://github.com/ephpm/db-wordpress), [mysqli-shim](https://github.com/ephpm/mysqli-shim), [db-laravel](https://github.com/ephpm/db-laravel), [db-doctrine](https://github.com/ephpm/db-doctrine), [db-codeigniter](https://github.com/ephpm/db-codeigniter)
- [ephpm #357: ephpm_db_run and has_rowset](https://github.com/ephpm/ephpm/pull/357)
- [ephpm/wordpress-sample](https://github.com/ephpm/wordpress-sample)
