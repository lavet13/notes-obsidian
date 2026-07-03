---
id: networking-knowledge
aliases: []
tags: []
---

## CIDR prefix -> subnet mask
- The number after `/` = count of leading 1-bits in the 32-bit mask, NOT a value written into an octet.
- Build: N ones left-to-right, rest zeros, convert each octet to decimal.
- Bit weights in one octet: 128 64 32 16 8 4 2 1.
- /24 = 11111111.11111111.11111111.00000000 = 255.255.255.0
- /26 = ...11000000 = 255.255.255.192  (two high bits: 128+64)
- Last-octet cheat sheet: /25=128 /26=192 /27=224 /28=240 /29=248 /30=252 (running sum of 128 64 32 16 8 4)
- GOTCHA: /26 is NOT 255.255.255.4. The prefix counts BITS; it is not a literal octet value.

## Subnet math (block size)
- Host bits = 32 - prefix. Addresses/subnet = 2^host_bits. Usable = that - 2 (network + broadcast).
- /26 -> 6 host bits -> 64 addresses -> 62 usable.
- Block size = 256 - (last mask octet). /26 -> 256-192 = 64.
- /24 split into /26 = 4 subnets stepping by 64: .0/26 .64/26 .128/26 .192/26

## ARP / NDP (IP -> MAC resolution)
- LAN delivery is by MAC (L2); IP is L3. To build the frame the sender needs the dst MAC,
  so it resolves IP -> MAC. ARP for IPv4, NDP for IPv6 (same `ip neigh` table).
- Request = broadcast (dst MAC ff:ff:ff:ff:ff:ff): "who has IP X, tell Y". Only the owner
  replies, unicast, with its MAC. Sender caches the pair (expires in minutes).
- GOTCHA: ARP never leaves the subnet. For an outside dst (8.8.8.8) you ARP the GATEWAY's
  MAC, not the destination. Frame dst MAC -> gateway; packet dst IP stays 8.8.8.8.

## MAC vs IP across hops
- IP dst = final destination, constant end-to-end (unless NAT rewrites it).
- MAC = next-hop addressing, rewritten at EVERY router (src/dst = the two ends of that hop).
- A router receives a frame (dst MAC = its own), strips L2, reads dst IP, looks up next hop,
  builds a NEW frame with new src/dst MAC.

## Neighbor states (ip neigh)
- REACHABLE = recently confirmed | STALE = valid but not recently used (re-checked on next use)
- INCOMPLETE = resolving | FAILED = no reply, MAC unknown | PERMANENT = static, never expires

## Routing (ip route)
- Decision = longest-prefix match: most specific matching route wins; unmatched -> default (0.0.0.0/0).
- `scope link` = reachable directly, no gateway. `proto kernel` = auto-added when IP assigned.
  `src` = preferred source addr for host-originated traffic.
- `onlink` = treat gateway as reachable on this link even if not in the interface's subnet.
  Enables /32 interface + default via an out-of-subnet gateway (common VPS pattern).
- `ip route get <ip> [from <src> iif <if>]` = which route/src/gw the kernel would pick.
  GOTCHA: routing lookup ONLY, no NAT - MASQUERADE runs later in POSTROUTING.
- net.ipv4.ip_forward must = 1 for the host to route transit packets. 0 = drops them (NAT/VPN dead).

## NAT (iptables -t nat)
- MASQUERADE = source NAT in POSTROUTING: rewrites src IP to the outgoing interface's IP
  (dynamic SNAT for outbound). `-s <net> ! -o <lan-if>` = masq this net when leaving toward outside.
- DNAT = destination NAT in PREROUTING: rewrites dst to an internal host (inbound port publishing),
  e.g. `udp dpt:31657 to:172.29.172.2:31657`.
- Chains: PREROUTING (inbound, pre-routing) | OUTPUT (host-originated) | POSTROUTING (post-routing, outbound).
- Double NAT = two SNAT layers in series (container netns, then host): 10.8.1.x -> 172.29.172.2 -> public.
  Each layer has its OWN conntrack; a host `conntrack -L` won't show addresses translated inside a
  container's namespace.

## Verifying a NAT path
- Counters: `iptables -t nat -L POSTROUTING -vn`, watch the matching rule's pkts grow. Both
  layers growing together = traffic crossed both.
- Per hop: `tcpdump -ni <if> icmp`, watch src walk hop to hop (10.8.1.x -> 172.29.172.2 -> public).
- `conntrack -L -s <src>` on host: original vs translated tuple (needs conntrack pkg).

## FORWARD chain + Docker/ufw
- policy DROP = safe default; only explicitly-allowed transit is forwarded.
- Docker inserts DOCKER-USER (yours to fill; survives `docker restart`) and DOCKER-FORWARD
  (Docker's own rules) at the top of FORWARD. Container traffic is accepted here BEFORE ufw's forward hooks.
- GOTCHA: ufw does NOT filter container transit (its forward hooks show 0 pkts). ufw governs
  host INPUT only. Put custom container-forward rules in DOCKER-USER.
