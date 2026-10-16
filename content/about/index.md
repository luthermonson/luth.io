---
title: "About the Blogger"
date: 2022-01-26T08:26:16-07:00
author: "Luther Monson"
comments: false
---

Originally born and raised in the Midwest, Luther received a Bachelor of the Arts in Computer Science from
[Gustavus Adolphus College](https://gustavus.edu/) in Southern Minnesota. After that he moved to the expansive deserts
of the American Southwest. In Arizona, Luther spent his career building web APIs and utilities for various industries
in PHP and NodeJS. From there he migrated to Open Source DevOps tooling using Containers and Virtualization.

Luther works at [Akamai](https://www.akamai.com/) on [LKE Enterprise](https://www.linode.com/products/kubernetes/),
the enterprise tier of the Linode Kubernetes Engine. Before Akamai he spent years in the Kubernetes space at
Rancher Labs and [SUSE](https://suse.com/) working on the Multi-Cluster Management tool
[Rancher](https://rancher.com/). As the Team Lead for [Windows on Rancher](https://github.com/rancher/windows) he
specialized in Kubernetes on alternative operating systems and architectures including Windows, arm64 and s390x.
After Rancher he worked at [Spectro Cloud](https://www.spectrocloud.com/) on Cluster API providers for AWS and Azure.
That included OIDC/IRSA support for self-managed clusters on AWS.
Working in this unique space and working from home full time has made him a home lab aficionado and consumer of
[Proxmox VE](https://www.proxmox.com/en/).

### ePHPm

[ePHPm](https://github.com/ephpm/ephpm) runs the same PHP app the same way from dev to prod as one Rust binary. It
embeds real PHP through a custom SAPI. It bundles a web server with automatic TLS, an embedded SQLite-compatible
database, a KV cache, a pooling SQL proxy and gossip clustering. A PHP app runs as a single process with no other
services to operate.

### ephemerd

[ephemerd](https://github.com/ephpm/ephemerd) is ephemeral GitHub Actions runners in a single binary for every
platform: containers on Linux, Hyper-V on Windows, VMs on macOS. Every job gets an isolated, disposable environment
that is thrown away when the job finishes.

### litewire

[litewire](https://github.com/ephpm/litewire) speaks the MySQL, PostgreSQL and SQL Server wire protocols in front of
SQLite. Point `pdo_mysql` at `:3306` and an unmodified app runs on embedded SQLite. There's no database server and no
code changes. It is the database layer inside ePHPm.

### shipmates

[shipmates](https://github.com/luthermonson/shipmates) gives Claude Code subagents that remember. It provides
role-specialized personas with persistent per-project memory. It also has a live lead-and-crew channel for dispatching
work, steering an agent mid-task and approving actions with a human in the loop.

### go-proxmox

[go-proxmox](https://github.com/luthermonson/go-proxmox) is a Go client, with types and tests, for the Proxmox VE
REST API. It started in 2021 as a side project for the home lab. It's now used by projects like the
[Cluster API provider for Proxmox](https://github.com/ionos-cloud/cluster-api-provider-proxmox).

### Other Projects
* [tail-gunner](https://github.com/luthermonson/tail-gunner): a drop-in `tail` that promotes mid-stream into a live-filtering TUI
* [linode-tui](https://github.com/luthermonson/linode-tui): a k9s-inspired terminal UI for the Linode API
* [goodhosts](https://github.com/goodhosts/cli): hosts file management from a Go CLI, on every OS
* [quiso](https://github.com/luthermonson/quiso): QUickISO builder for cloud-init

### Open Source Contributions
* [Rancher Machine](https://github.com/rancher/machine/pulls?q=is%3Apr+author%3Aluthermonson) and the [vSphere Node Driver](https://github.com/rancher/machine/pulls?q=is%3Apr+author%3Aluthermonson+vsphere)
* [RKE1](https://github.com/rancher-archives/rke/commits?author=luthermonson), [RKE2](https://github.com/rancher/rke2/pulls?q=is%3Apr+author%3Aluthermonson) and [remotedialer](https://github.com/rancher/remotedialer/pulls?q=is%3Apr+author%3Aluthermonson)
* [Cluster API](https://github.com/kubernetes-sigs/cluster-api/pulls?q=is%3Apr+author%3Aluthermonson) and its providers for
  [AWS](https://github.com/kubernetes-sigs/cluster-api-provider-aws/pulls?q=is%3Apr+author%3Aluthermonson) (OIDC/IRSA support for self-managed clusters)
  and [Azure](https://github.com/kubernetes-sigs/cluster-api-provider-azure/pulls?q=is%3Apr+author%3Aluthermonson),
  plus Spectro Cloud's [AWS](https://github.com/spectrocloud-public/cluster-api-provider-aws/pulls?q=is%3Apr+author%3Aluthermonson)
  and [Azure](https://github.com/spectrocloud-public/cluster-api-provider-azure/pulls?q=is%3Apr+author%3Aluthermonson) provider forks
* [sig-windows-dev-tools](https://github.com/kubernetes-sigs/sig-windows-dev-tools/pulls?q=is%3Apr+author%3Aluthermonson)
* [Flannel](https://github.com/flannel-io/flannel/pulls?q=is%3Apr+author%3Aluthermonson)
* [govmomi](https://github.com/vmware/govmomi/pulls?q=is%3Apr+author%3Aluthermonson)
* [go-diskfs](https://github.com/diskfs/go-diskfs/pulls?q=is%3Apr+author%3Aluthermonson)
* [static-php-cli](https://github.com/crazywhalecc/static-php-cli/pulls?q=is%3Apr+author%3Aluthermonson)
* [php-src](https://github.com/php/php-src/pulls?q=is%3Apr+author%3Aluthermonson)
* [Turso](https://github.com/tursodatabase/turso/pulls?q=is%3Apr+author%3Aluthermonson)
