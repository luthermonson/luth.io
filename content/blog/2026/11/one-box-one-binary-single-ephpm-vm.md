---
title: "Replacing a k3s Cluster With One ePHPm Process"
description: "How three PHP sites, their databases, cron and backups collapse onto one small VM, and what that costs once you match the bill to the size of the problem."
date: 2026-11-18T07:30:00-07:00
tags: [ aws, ec2, ephpm, spot, sqlite ]
categories: [ ephpm, containers ]
images:
 - src: img/2026/11/one-box-one-binary-single-ephpm-vm-cover.svg
   alt: "A tall single-node k3s stack of Traefik, Tailscale, three PHP pods, three CronJobs, a litestream sidecar, PVs and the control plane collapses into one Debian VM running a single ephpm process that is the web server, PHP runtime, per-site SQLite, KV store, cron scheduler and S3 backup agent"
   stretch: "none"
draft: false
---

A couple of weeks ago I wrote up [surviving a node rebuild](/blog/2026/10/durable-data-single-node-k3s-ebs-rebuild/)
on single-node k3s. Put the stateful bytes on a dedicated EBS volume, treat the node as cattle, and rebuild the machine
without losing the data. It works.

Then I looked at what I was actually running on that node and felt a little silly.

The workload is three PHP sites (an admin app, self-hosted Sendy for email, a wholesale catalog) plus three crons.
That's it. To run it I had a k3s control plane, a kubelet, Traefik for ingress, Tailscale for access, a CSI driver, a
litestream sidecar for backups, three k8s CronJobs, and a stack of Helm charts holding it together. Every piece of that is
something I patch, monitor, and get paged for, and none of it is the app. It was a datacenter wrapped around three PHP
sites and some crons.

So I migrated it. The whole thing now runs as a single process on one AWS VM: [ePHPm](https://github.com/ephpm/ephpm).
The one binary is the web server, the PHP runtime, the per-site SQLite engine, the KV store, the cron scheduler, and the
backup agent: the whole stack above, collapsed into one. The only other thing on the machine is the app code.

![A tall single-node k3s stack collapses into one Debian VM running a single ephpm process that is the web server, PHP runtime, per-site SQLite, KV store, cron scheduler and S3 backup agent](/img/2026/11/one-box-one-binary-single-ephpm-vm-cover.svg)

### What ePHPm absorbs

The reason a single binary can replace that stack is that ePHPm already *is* most of the stack. It embeds PHP with the ZTS
build, so it is the runtime directly, no php-fpm pool. It serves HTTP and terminates TLS itself, so there's no reverse
proxy. Each site gets its own SQLite database through the embedded Turso engine, so there's no MySQL to run. And it carries
a Redis-compatible KV store for sessions, a cron scheduler, and a backup agent that snapshots every database to S3.

I covered the [clustered SQLite](/blog/2026/08/clustered-sqlite-adding-turso/) and the
[KV store](/blog/2026/08/rolling-your-own-kv-store/) before; what matters here is the deployment shape. One `ephpm.toml`
describes all three sites. The apps connect to their databases with stock `pdo_mysql` over a loopback listener, and ePHPm
injects each site's credentials, so no app file holds a password. Sendy's scheduled send, its CSV import, and the wholesale
catalog refresh are three `[[cron.jobs]]` entries: the former k8s CronJobs, now config lines the scheduler runs exactly
once even when more than one node is up.

Sendy is the one that proves the isolation is real. It's a classic `.htaccess` app (dozens of top-level PHP scripts and
pretty tracking URLs), so it needs a per-site `[[rewrite]]` block, plus a scoped capability grant that lifts `mail` and
`fsockopen` for that vhost alone. Admin and wholesale never get those; they stay hardened. One binary, three very different
apps, and the loose one can't reach past its own vhost.

One structural fact matters below: the box is a baked, immutable image, and its state lives on a separate EBS volume. The
instance is disposable; the data isn't. How that image is built is
[part two](/blog/2026/11/operating-one-box-ephpm-backups-blue-green/). Here it just has to be true, because it's what
makes the cost math work.

### What it costs

This is the part people get wrong about going from a cluster to a box. They assume you traded capability for thrift. You
didn't. You matched the bill to the size of the problem. Here's the whole monthly bill, us-west-2, on-demand, nothing
reserved.

| Line item | Spec | Monthly |
| --- | --- | --- |
| `t3a.small` on-demand | 2 vCPU / 2 GB, ~730 hrs | $13.72 |
| EBS gp3 | 20 GB root + 20 GB data @ $0.08/GB | $3.20 |
| Public IPv4 (Elastic IP, in use) | 1 address @ $0.005/hr | $3.65 |
| S3 backups | gzipped snapshots, retention-tuned | ~$2-3 |
| **All-in** | | **~$23/mo** |

The fixed floor (compute, disk, IP) is $20.57. Two line items are easy to miss. gp3 is cheaper than the old gp2 default
and still gives you 3,000 IOPS baseline for free, so 40 GB is $3.20 and not a penny more. And since February 2024 AWS
charges for every public IPv4 address, in use or idle: about $3.65/mo for the one Elastic IP, a line nobody budgeted for
two years ago. A "free" static IP is a real part of this comparison.

I did shop around. Linode and DigitalOcean bundle the static IP, and Oracle's always-free ARM tier is genuinely capable for
nothing. Both are real options, and the ARM pricing is better. I stayed on AWS for two unglamorous reasons: the account,
Terraform, and S3 bucket are already there, and the admin app's imagick build is pinned to amd64 and glibc 2.36, so
Oracle's ARM (Ampere) tier means rebuilding that ABI story, not copying a config. A $3.65/mo static IP that just works beat
a free one on a platform I'd have to re-learn.

None of this is about squeezing pennies. It's about matching the infrastructure bill *and* the operational cost to the
actual size of the problem. Three PHP sites and some crons is a small problem. It should run on a small thing.

### The outage you can afford

Everything above is on-demand, the no-tricks number. But there's a cheaper gear, and whether you can shift into it is a
question about your *apps*, not your infrastructure.

First, right-size. I load-tested the box and it idles around 400 MB of RAM and a few percent of one CPU. Even under a
sustained flood (fifty concurrent connections on the heaviest catalog page), memory barely moved, because ePHPm caps PHP
concurrency to the core count (floored at two) and queues the rest; traffic can't balloon the heap, it just waits. A 1 GB
`t3a.micro` has plenty of headroom. The 2 GB in the table was caution, not need.

Then spot. Spot is the same hardware at a fraction of the price (a `t3a.micro` runs about $2-3/mo on spot against $6.86
on-demand) with one string: AWS can take the box back on two minutes' notice. For a lot of workloads that's a dealbreaker;
for *this* one it isn't, since the storefront is Shopify, never this box, and what's here (the admin panel, the wholesale
catalog, Sendy) is internal tools and a mailer that can go dark for a couple of minutes without breaking a checkout or
losing a dollar, exactly the profile where spot stops being reckless and starts being obvious.

Right-sized and on spot, it's the same box for less than half the bill:

| | On-demand, as billed above | Right-sized on spot |
| --- | --- | --- |
| Instance | `t3a.small`, 2 GB | `t3a.micro`, 1 GB |
| Compute | $13.72/mo | ~$2-3/mo |
| All-in | ~$23/mo | ~$10/mo |
| Interruptions | none | AWS can reclaim on two minutes' notice |

So the box runs in a single-node spot Auto Scaling Group. The data already lives on its own EBS volume, so when a spot node
dies the disk outlives it. A replacement boots, re-attaches that same volume (seconds, not a copy; the 1.3 GB never moves),
re-grabs the Elastic IP, and starts serving. A small watcher catches the two-minute interruption notice and stops ePHPm
gracefully first, so it flushes its WAL and finishes the in-flight Sendy SES batch; the next cron tick resumes the send at
the exact queue offset, because Sendy's queue *is* the resume point.

I didn't want to take that on faith, so I tested it the rude way and timed the gap, twice:

- **Hard kill, no notice.** I terminated the running spot node cold, the worst case. The ASG launched a replacement, it
  re-attached the volume and the IP, and the site answered again **81 seconds** later. The databases came back
  byte-for-byte intact, because the replacement replayed Sendy's uncheckpointed WAL on startup, which is the whole point of
  a write-ahead log.
- **Real rebalance, minutes later.** AWS sent a genuine rebalance recommendation and the ASG swapped the node *again*, and
  the data sailed through that one too.

Two unplanned replacements, zero data loss.

The second swap taught me one thing worth passing on: for a *single* node, leave capacity rebalancing off. It replaces an
at-risk instance before it's actually interrupted, which is what you want when a second node covers the swap and what you
*don't* want when there's one node and every swap is 80 seconds dark for an interruption that might never come. One node
rides until a real interruption; a fleet rebalances early.

Is it zero-downtime? No. One node on spot has a real, non-zero chance of a few minutes dark, and no arrangement of spot
drives that to zero. If I needed never-interrupted I'd keep one on-demand node as a floor (`on_demand_base = 1`) or run a
warm second node; both cost more, in dollars or moving parts. I need neither. Call it 99.98% on internal tools whose
outages self-heal and never reach a customer; not a number worth paying to improve. Match the availability to the problem:
some apps can trade a sliver of uptime for a chunk of cost, and these are those apps. Doing it drops the all-in nearer
**$10/mo** than $23.

### What's next: closing the 81-second gap

That 81 seconds is a cold, reactive replacement. The node dies, then the ASG notices, then a new one boots, attaches the
volume, grabs the IP, and starts serving. Every step waits on the one before it, and nothing starts until the box is
already gone. The obvious next move is to stop waiting for it to die.

Spot tells you first. The two-minute interruption notice and the rebalance recommendation both land before the box
actually goes away, and right now I only use that window to drain cleanly. The idea is to use it to act: on the signal,
launch the replacement immediately so it boots and reaches ready while the old node is still serving, then hand off the EBS
volume and the Elastic IP at the last second. The dark window shrinks from "boot a whole VM" to "detach, attach, flip the
IP," which is a few seconds, not eighty.

The catch is the one from the blue/green section: a single EBS volume can't attach to two live nodes at once, so the warm
replacement can boot and initialize but can't take the data until the old node lets go. There's still a hard serialization
point at the handoff. Whether that lands at five seconds or twenty-five is the thing I actually want to measure, so the
experiment is to wire a watcher to the interruption and rebalance signals, have it pre-launch the replacement and pre-stage
the attach, then force the reclaim and time it against the 81-second cold-start baseline. Same rude test, one variable
changed. If it lands where I think it will, a planned spot reclaim costs a handful of seconds and the whole "outage you can
afford" math gets easier still.

Baking the image, backing it up, and shipping new code without downtime is
[the second half](/blog/2026/11/operating-one-box-ephpm-backups-blue-green/).

### Links

- [Surviving a Node Rebuild: Durable Data on Single-Node k3s](/blog/2026/10/durable-data-single-node-k3s-ebs-rebuild/)
- [Part two: Operating the One-Box ePHPm Setup](/blog/2026/11/operating-one-box-ephpm-backups-blue-green/)
- [ePHPm](https://github.com/ephpm/ephpm)
- [Clustered SQLite: Adding Turso](/blog/2026/08/clustered-sqlite-adding-turso/)
- [Rolling Your Own KV Store](/blog/2026/08/rolling-your-own-kv-store/)
- [AWS public IPv4 address charge](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)
- [Amazon EBS gp3 pricing](https://aws.amazon.com/ebs/pricing/)
