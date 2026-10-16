---
title: "Zero-Dependency Deployments - Single Binary for WordPress"
description: "WordPress with no nginx, no php-fpm, no Redis and no MySQL: one ePHPm binary, a SQLite file, and a distroless container image. What's in the binary, how it gets built, and what it actually takes to run."
date: 2026-09-08T20:25:00-07:00
tags: [ ephpm, wordpress, php, containers, sqlite, deployment ]
categories: [ ephpm, php, containers ]
images:
 - src: img/2026/09/single-binary-wordpress-cover.svg
   alt: "Side by side: nginx, php-fpm, Redis and MySQL as four services, versus one ephpm process containing the HTTP server, embedded PHP, KV store and SQLite"
   stretch: "none"
draft: false
---

The standard way to run a WordPress site is four services: nginx for HTTP and TLS, php-fpm to run PHP, MySQL for data,
and Redis once the site gets slow enough that someone installs an object cache plugin. That's four processes, four
config files, three sockets between them, and four things to upgrade. For a big site that's fine. For a brochure site, a
staging copy, a PR preview, a CI job that needs a real WordPress to test a plugin against, or a box at the edge, it's
absurd.

[ePHPm](https://github.com/ephpm/ephpm) is my attempt at making that one thing. This post walks through running stock
WordPress from a single binary with no other services, building a container image that is just that binary and the site,
and what's actually inside it, as of [v0.8.2](https://github.com/ephpm/ephpm/releases/tag/v0.8.2).

### What's in the binary

One `ephpm` executable (about 132 MB for the Linux x86_64 build with PHP 8.5.7) carries:

- **An HTTP server**: hyper with HTTP/1.1 and HTTP/2, rustls TLS with ACME certificates, static files with ETags and
  compression, virtual hosts.
- **PHP itself**: `libphp` statically linked, ZTS, with OPcache and the usual extensions compiled in (`mysqli`,
  `pdo_mysql`, `pdo_sqlite`, `gd`, `intl`, `mbstring`, `curl`, `zip` and friends). The same binary is also a PHP CLI:
  `ephpm php -v`.
- **A KV store** that PHP can call directly through native functions, which is what the WordPress object cache uses.
- **A database layer**: either a pooling proxy in front of a real MySQL, or no database server at all.

"Zero dependency" needs one footnote. The Linux release is dynamically linked against glibc, not a static musl binary:

```
$ ldd ephpm
	linux-vdso.so.1
	libdl.so.2 => /usr/lib/x86_64-linux-gnu/libdl.so.2
	libm.so.6 => /usr/lib/x86_64-linux-gnu/libm.so.6
	libpthread.so.0 => /usr/lib/x86_64-linux-gnu/libpthread.so.0
	librt.so.1 => /usr/lib/x86_64-linux-gnu/librt.so.1
	libgcc_s.so.1 => /usr/lib/x86_64-linux-gnu/libgcc_s.so.1
	libutil.so.1 => /usr/lib/x86_64-linux-gnu/libutil.so.1
	libc.so.6 => /usr/lib/x86_64-linux-gnu/libc.so.6
	/lib64/ld-linux-x86-64.so.2

$ objdump -T ephpm | grep -o 'GLIBC_[0-9.]*' | sort -uV | tail -1
GLIBC_2.28
```

Everything above is glibc's own family plus `libgcc_s`. The newest symbol version it needs is 2.28, because release
builds happen in an AlmaLinux 8 container, so it runs on basically any Linux from 2018 onward. I went glibc over musl on
purpose: native middleware modules are loaded with `dlopen`, and a fully static musl binary can't do that.

### How the binary gets built

Building PHP is the slow, fragile part of all of this, so it's done once per PHP version, not once per ePHPm build.
[static-php-cli](https://github.com/crazywhalecc/static-php-cli) compiles `libphp` with the embed SAPI and its library
dependencies into static archives, and [ephpm/php-sdk](https://github.com/ephpm/php-sdk) publishes those as release
assets per OS, architecture and PHP version. ePHPm's build is a cargo `xtask`:

```bash
cargo xtask release 8.5     # resolve 8.5 -> pinned 8.5.7, download the SDK, build with PHP linked
```

`xtask release` resolves the PHP minor to a pinned patch version, downloads and unpacks the matching SDK, points
`PHP_SDK_PATH` at it, and runs the cargo build with the shipped feature set. `build.rs` compiles the C SAPI wrapper and
emits the link flags for `libphp.a`. A plain `cargo build` without an SDK builds a stub that answers every PHP request
with a "PHP not linked" page, so the Rust side can be developed and tested without PHP at all.

It's not a cross-compiler: each platform's release is built on that platform (Linux x86_64 and aarch64 in the AlmaLinux
container, macOS arm64 and Windows x86_64 natively), for PHP 8.3, 8.4 and 8.5.

### WordPress with nothing else

Here's the whole setup. Grab WordPress, the official
[SQLite Database Integration](https://wordpress.org/plugins/sqlite-database-integration/) drop-in (WordPress's own
project for running on SQLite), and the [ephpm/cache-wordpress](https://github.com/ephpm/cache-wordpress) object cache:

```bash
curl -sSL https://wordpress.org/wordpress-7.1.tar.gz | tar xz

# SQLite instead of MySQL
curl -sSLO https://downloads.wordpress.org/plugin/sqlite-database-integration.3.0.1.zip
unzip -q sqlite-database-integration.3.0.1.zip -d wordpress/wp-content/plugins/
cp wordpress/wp-content/plugins/sqlite-database-integration/db.copy wordpress/wp-content/db.php
mkdir -p wordpress/wp-content/database

# Persistent object cache backed by ePHPm's in-process KV store
git clone https://github.com/ephpm/cache-wordpress wordpress/wp-content/cache-wordpress
printf "<?php\nrequire __DIR__ . '/cache-wordpress/dropin/object-cache.php';\n" \
  > wordpress/wp-content/object-cache.php
```

Copy `wp-config-sample.php` to `wp-config.php`, leave the DB credentials as placeholders (the SQLite drop-in ignores
them), add `define( 'WP_CACHE', true );`, and start it:

```bash
ephpm dev --port 8088 --document-root "$(pwd)/wordpress"
```

That's a working WordPress on `http://127.0.0.1:8088`, with the installer, admin, themes and plugins. The database is a
file at `wp-content/database/.ht.sqlite`. The object cache lives in the same process as PHP: `wp_cache_get()` goes
straight from the Zend engine into the KV store through `ephpm_kv_*` native functions, no socket and no Redis.

### The container is the binary and the site

Since there's nothing else to run, the image doesn't need a distro. `gcr.io/distroless/cc-debian12` is glibc, libssl,
CA certificates, tzdata and `libgcc`, with no shell and no package manager, and that covers exactly what the `ldd`
output above asks for:

```dockerfile
# The whole runtime: glibc + libgcc from distroless, one binary, the site.
FROM gcr.io/distroless/cc-debian12:nonroot
COPY --chmod=755 ephpm /usr/local/bin/ephpm
COPY ephpm.toml /etc/ephpm/ephpm.toml
COPY --chown=65532:65532 wordpress/ /srv/www/
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/ephpm", "serve", "--config", "/etc/ephpm/ephpm.toml"]
```

with a config that's about as small as configs get:

```toml
[server]
listen = "0.0.0.0:8080"
document_root = "/srv/www"

[kv]
memory_limit = "64MB"
```

I built that with v0.8.2 (PHP 8.5.7), WordPress 7.1, SQLite Database Integration 3.0.1 and cache-wordpress, ran it,
and installed WordPress by posting the installer form with `curl`:

```
$ podman images
localhost/blog-wp-zerodep:latest       271 MB
gcr.io/distroless/cc-debian12:nonroot  25.1 MB

$ podman history --format '{{.Size}} {{.CreatedBy}}' localhost/blog-wp-zerodep | head -5
0B      ENTRYPOINT ["/usr/local/bin/ephpm", ...
0B      EXPOSE 8080
114MB   COPY dir:...          # WordPress 7.1 + plugins
3.07kB  COPY file:...         # ephpm.toml
132MB   COPY file:...         # ephpm

install step2: 200
<h1>Success!</h1>
```

After that the front page served normally. To confirm the pieces were really the ones I thought, I dropped a five-line `check.php` into the docroot that loads
WordPress and prints what it's running on:

```
persistent object cache: yes
cache class: Ephpm\Cache\WordPress\ObjectCache
db driver: WP_SQLite_DB
sapi: ephpm
```

The container sat at about 76 MB of memory after the install and a handful of page views (`podman stats`), and the
whole database was one small SQLite file. Out of 271 MB of image, about half is the WordPress tree and half is the binary;
distroless itself is 25 MB. There's no shell in the container at all, which is nice for security and occasionally
annoying for debugging (`podman exec ... /usr/local/bin/ephpm php -r '...'` is the escape hatch, since the binary is also
a PHP CLI).

### Where this is the right tool, and where it isn't

Where one binary with a SQLite file makes sense:

- **CI**: testing a plugin or theme against a real WordPress without starting MySQL and a web server in the job.
- **PR previews and staging copies**, where spinning up and tearing down is the main workload.
- **Edge and small boxes**: a Raspberry Pi, a $5 VM, an appliance. Anywhere the LEMP stack is most of the resource budget.
- **Low-traffic sites** whose real requirement is "it stays up and I don't have to babysit it."

Where it isn't:

- **Write-heavy, multi-writer sites.** SQLite is one writer at a time. A busy WooCommerce store with concurrent checkouts
  wants a real MySQL. ePHPm still helps there: point `[db.mysql]` at the MySQL server and its proxy pools connections, and
  you've still dropped nginx, php-fpm and Redis.
- **Plugins that assume MySQL-specific SQL** in ways the SQLite drop-in can't translate. WordPress's SQLite project has
  come a long way, but test your plugin list before you commit.
- **Anything that needs the cache to survive a restart.** The KV store is a cache; the object cache starts cold.

### What's next

The single binary is the easy part to show. The interesting work is behind it: making the embedded engine behave like
php-fpm for the apps that assume php-fpm, and making the database side do more than SQLite-in-a-file without bringing a
server back. That's where most of the ePHPm commits are going right now.

### How I ran it

AMD Ryzen 9 5950X, Windows 11 with WSL2 and Podman 5 (rootful machine). Image built from the files above with
`podman build`, run with `-p 18300:8080`, installer posted from the Ubuntu WSL distro with `curl`. The three `GET /`
timings are the first three requests after install from a cold OPcache, not a benchmark.

### Links

- [ePHPm](https://github.com/ephpm/ephpm) and the [v0.8.2 release](https://github.com/ephpm/ephpm/releases/tag/v0.8.2)
- [ephpm/php-sdk](https://github.com/ephpm/php-sdk), [static-php-cli](https://github.com/crazywhalecc/static-php-cli)
- [SQLite Database Integration](https://github.com/WordPress/sqlite-database-integration)
- [ephpm/cache-wordpress](https://github.com/ephpm/cache-wordpress)
- [distroless](https://github.com/GoogleContainerTools/distroless)
