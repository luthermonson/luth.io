---
title: "My iptables Rule Matched 0 Packets in 9.8 GB"
description: "Container-to-host traffic goes through INPUT, not FORWARD, so ephemerd's carefully written allow-list never matched a single packet."
date: 2026-08-26T20:30:00-07:00
tags: [ linux, iptables, netfilter, containers, security, ephemerd ]
categories: [ containers, security, ci ]
series: [ "ephemerd" ]
images:
 - src: img/2026/08/iptables-rule-zero-packets-flow.svg
   alt: "Netfilter flow showing container traffic to the host taking INPUT instead of FORWARD"
   stretch: "none"
draft: false
---

After a week of Windows firewall rules that [listed fine and enforced nothing](/blog/2026/08/windows-container-egress-firewall/),
I went back to the Linux side of [ephemerd](https://github.com/ephpm/ephemerd) feeling pretty good about it. iptables is
the boring, well-understood one. Then I looked at the packet counters on a live node. I found a rule that had never
matched a packet in its life. That node had pushed 9.8 GB through the same chain.

### The rules I thought I had

Job containers sit on a CNI bridge, `10.88.0.0/16`, with the host on `10.88.0.1`. ephemerd puts its egress policy in its
own chain, `EPHEMERD-FORWARD`. `FORWARD` jumps to it for anything to or from the container subnet. From a live node:

```
-P INPUT ACCEPT
-P FORWARD ACCEPT
-N EPHEMERD-FORWARD
-A FORWARD -d 10.88.0.0/16 -j EPHEMERD-FORWARD
-A FORWARD -s 10.88.0.0/16 -j EPHEMERD-FORWARD
-A EPHEMERD-FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A EPHEMERD-FORWARD -s 10.88.0.0/16 -d 10.88.0.0/16 -j ACCEPT
-A EPHEMERD-FORWARD -d 10.88.0.1/32 -p udp -m udp --dport 53 -j ACCEPT
-A EPHEMERD-FORWARD -d 10.88.0.1/32 -p tcp -m tcp --dport 53 -j ACCEPT
-A EPHEMERD-FORWARD -d 10.88.0.1/32 -p tcp -m tcp --dport 8082 -j ACCEPT
-A EPHEMERD-FORWARD -d 10.0.0.0/8 -j REJECT --reject-with icmp-net-unreachable
-A EPHEMERD-FORWARD -d 172.16.0.0/12 -j REJECT --reject-with icmp-net-unreachable
-A EPHEMERD-FORWARD -d 192.168.0.0/16 -j REJECT --reject-with icmp-net-unreachable
-A EPHEMERD-FORWARD -d 169.254.0.0/16 -j REJECT --reject-with icmp-net-unreachable
-A EPHEMERD-FORWARD -j ACCEPT
```

Reads well. DNS and the Go module proxy (tcp/8082) on the gateway are allowed. The private ranges are rejected. The
internet is open. The `GatewayPorts` config setting is the allow-list of host ports a job may reach. The security docs
said containers can't reach `10.0.0.0/8`.

### The counters

`iptables -S` tells you what rules exist. `iptables -L -v -n --line-numbers` tells you what they've done:

```
Chain EPHEMERD-FORWARD (2 references)
num   pkts bytes target   prot  source        destination
1    1614K 9814M ACCEPT   0     0.0.0.0/0     0.0.0.0/0     ctstate RELATED,ESTABLISHED
2        0     0 ACCEPT   0     10.88.0.0/16  10.88.0.0/16
3        0     0 ACCEPT   17    0.0.0.0/0     10.88.0.1     udp dpt:53
4        0     0 ACCEPT   6     0.0.0.0/0     10.88.0.1     tcp dpt:53
5        0     0 ACCEPT   6     0.0.0.0/0     10.88.0.1     tcp dpt:8082
6        0     0 REJECT   0     0.0.0.0/0     10.0.0.0/8    reject-with icmp-net-unreachable
```

Rule 5 is the module proxy allow. Zero packets, zero bytes, next to 9.8 GB of established traffic in rule 1. And the module
proxy isn't idle. It's in daily use.

![Netfilter flow](/img/2026/08/iptables-rule-zero-packets-flow.svg)

### Why: local delivery goes through INPUT

This one is in every netfilter diagram, and I still walked right past it. After `PREROUTING` the kernel makes a routing
decision. If the destination is an address the host itself owns, the packet is delivered locally and goes through `INPUT`.
Only packets the host is going to route somewhere else go through `FORWARD`.

The bridge gateway `10.88.0.1` is an address the host owns. Say a container talks to the module proxy, DNS on the
gateway or another service listening on the host. That packet goes PREROUTING, routing decision, INPUT, local socket. It
never touches `FORWARD`, so nothing in `EPHEMERD-FORWARD` can match it. Rules 3, 4 and 5 weren't allowing anything. The
traffic was getting through because nothing was looking at it.

That had two worse consequences than a dead allow rule:

1. **The allow-list wasn't a list.** Anything ephemerd or the operator bound on the gateway was reachable from every job,
   whatever `GatewayPorts` said. The only `INPUT` rules ephemerd ever wrote were drops for its own control ports. Those
   were only configured on the in-VM worker path. A bare-metal Linux node installed no `INPUT` rules at all, and the
   distro default is `INPUT` policy `ACCEPT`.
2. **The RFC1918 block had a host-shaped hole.** The node's own LAN address (something like `10.0.0.44/24`) is inside
   `10.0.0.0/8`. Containers couldn't reach *other* machines on `10.0.0.0/8`. But they could reach the host itself on
   every port, because that's local delivery too.

There was a third, smaller one inside the Linux VM on Windows and macOS hosts. The control-port drops matched
`-d <gateway>`. But the in-VM dispatch gRPC server binds `0.0.0.0`. So it was also listening on the VM's `eth0` address,
where that drop doesn't apply.

### How I found it

Sideways, at first. Getting the Docker API to work under Kata meant moving it from a unix socket to a TCP port on the
gateway. The Docker API authenticates nothing, so that port had to be scoped to the one container that owns it. That
per-job chain (`EPHEMERD-DIND`) had to be jumped from `INPUT` as well as `FORWARD`. `INPUT` is where gateway-addressed
packets land. The Kata PR noted that this meant `GatewayPorts` alone wasn't constraining anything. A later security
sweep pinned it down with the counters above (issue #156).

### The fix: EPHEMERD-INPUT

PR #158 adds a second chain for the container-to-host path. `INPUT` jumps to it at position 1 for anything sourced from
the container subnet. It ends in an unconditional `DROP`:

```go
func containerInputRules(gateway string, gatewayPorts, controlPorts []int) [][]string {
	rules := [][]string{
		{inputChainName, "-i", "lo", "-j", "ACCEPT"},
		{inputChainName, "-m", "conntrack", "--ctstate", "RELATED,ESTABLISHED", "-j", "ACCEPT"},
	}
	for _, proto := range []string{"udp", "tcp"} {
		rules = append(rules, []string{
			inputChainName, "-d", gateway, "-p", proto, "--dport", "53", "-j", "ACCEPT",
		})
	}
	// ... one ACCEPT per GatewayPorts entry, -d gateway, skipping control ports ...

	// Default deny. DROP rather than REJECT to give a job no ICMP oracle
	// for probing which host services exist.
	rules = append(rules, []string{inputChainName, "-j", "DROP"})
	return rules
}
```

The details that make it hold up:

- **The deny is a rule in our chain**, not the absence of rules, so it doesn't matter what the host's `INPUT` policy is.
- **The jump matches on source**, `-s 10.88.0.0/16`, rather than `-i ephemerd0`. A packet arriving from the LAN with a
  forged container address gets caught too.
- **`RELATED,ESTABLISHED` can't be abused.** The opening SYN of a new connection is `ctstate NEW` and hits the `DROP`. The
  accept only covers replies to connections the host opened toward a container.
- **Every allow is scoped `-d <gateway>`.** An unscoped port allow would also open that port on the host's LAN address, and
  a test asserts none exist.
- **The `lo` accept is deliberate.** The host reaching its own module proxy on the bridge address goes out `lo` with a
  source inside the container subnet. A container can't forge that. It has its own network namespace and arrives on the
  bridge.
- **The jump goes in last.** Say a chain is half-built when `INPUT` already points at it. It ends at the implicit
  `RETURN`, not the `DROP`, and fails open for as long as the install takes:

```go
// Jump last, so the chain is never half-built while INPUT already points
// at it — a partially built chain ends at the implicit RETURN, not at the
// DROP, which would fail open for as long as the install took.
jump := containerInputJumpRule(l.cfg.subnet())
if err := iptables(append([]string{"-C"}, jump...)...); err != nil {
	if err := iptables(append([]string{"-I", jump[0], "1"}, jump[1:]...)...); err != nil {
		return fmt.Errorf("inserting %s jump rule: %w", inputChainName, err)
	}
}
```

There's deliberately no IPv6 mirror. The CNI bridge is IPv4-only, so containers have no v6 address and no v6 path to the
host. A v6 default-deny would have to match on destination, since there's no v6 container subnet to match a source
against. It would take out the host's own link-local and NDP traffic on any dual-stack node. That would be a real outage
to close a hole that doesn't exist.

The old `GatewayPorts` allows in `EPHEMERD-FORWARD` stayed, with a comment saying they aren't the enforcement point. I
like the chain being a complete statement of intent even where it's redundant.

Before merging, the generated iptables argv was replayed inside a throwaway `ip netns` on a fleet node. That checked
real-kernel syntax and that `-C` made it idempotent. Then the namespace was torn down and the node's real chains diffed
to make sure nothing changed.

### And if the rules don't install, don't run jobs

Looking at all this made me notice the other half of the problem. `InstallFirewallRules()` could fail on Linux from
missing `iptables` or no `CAP_NET_ADMIN`. When it did, ephemerd logged a warning and kept going. A node with no
containment at all would happily run fork PRs. [#179](https://github.com/ephpm/ephemerd/pull/179) makes that fatal
before the scheduler starts:

```go
func enforceFirewallInstall(goos string, install func() error, log *slog.Logger) error {
	if err := install(); err != nil {
		if goos == "linux" {
			return fmt.Errorf("installing container firewall rules: %w — refusing to run jobs without container egress/host containment (fail closed)", err)
		}
		log.Warn("failed to install firewall rules (containers may access LAN)", "error", err)
	}
	return nil
}
```

`serve()` returns before `sched.Run`, so there's no code path that dispatches a job without the chains present. systemd
shows a failed unit, which is a lot easier to notice than a log line. It's the same stance as the Kata preflight. If the
config promises isolation and the host can't deliver it, refuse to start rather than quietly downgrading.

### Check the counters

The lesson I'm keeping from this one: a rule that never matches looks exactly like a rule that works. `iptables -S`
showed me a correct policy. The `-v` counters showed me that half of it was decoration. When I add a firewall rule now, I
generate the traffic it's meant to match and watch the counter move. It's the same way I'd want to see a test fail
before I trust it passing.

```bash
iptables -L EPHEMERD-INPUT -v -n --line-numbers
```

### Links

- [ephemerd](https://github.com/ephpm/ephemerd) (issue #156 and PR #158 have the full analysis)
- [#179 fail closed when the firewall can't install](https://github.com/ephpm/ephemerd/pull/179)
- [Netfilter packet flow](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg) (Wikimedia Commons)
