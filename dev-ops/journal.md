## 2026-07-03 - Networking fundamentals: subnetting, ARP, routing, NAT (on the VPS)

Covered (concept -> key insight):
- CIDR/masks: prefix = count of 1-bits, not an octet value. /26 = 255.255.255.192, not .4.
- Subnet math: block size = 256 - last-octet; /24 -> /26 = 4 subnets stepping 64, 62 usable each.
- ARP/NDP: LAN delivers by MAC; broadcast request, unicast reply. Key: ARP never leaves the
  subnet, so for outside IPs you resolve the GATEWAY's MAC.
- MAC vs IP across hops: IP dst constant end-to-end, MAC rewritten every hop.
- Routing: longest-prefix match; `onlink` explains the /32 interface + out-of-subnet default gw.
- ip route get: routing stage only, no NAT - proved routing and NAT are separate netfilter stages.
- NAT: MASQUERADE = SNAT in POSTROUTING, DNAT = inbound in PREROUTING. Confirmed the VPS runs
  DOUBLE NAT: 10.8.1.x -> 172.29.172.2 -> 153.76.117.52.
- FORWARD: policy DROP + Docker chains; ufw forward hooks show 0 pkts because Docker accepts
  container transit first (ufw = host INPUT only).

Analyzed real VPS output: ip neigh (live ARP/NDP cache; IPv6 gw FAILED), ip route (onlink default),
ip -6 route, ip route get (transit sim), iptables nat (PREROUTING DOCKER jump, two MASQUERADE layers,
DNAT udp 31657 -> amnezia container 172.29.172.2), FORWARD (DOCKER-USER/DOCKER-FORWARD, ufw bypass).

No code changed - learning + infra analysis session. No commit.

Open / resume:
- NEXT: expand DOCKER-FORWARD (`iptables -L DOCKER-FORWARD -vn`) to see the mirror ACCEPT rules
  for amn0 and close the loop on the VPN forwarding path.
- IPv6 gateway 2a12:bec4:1ac0::1 is FAILED (no NDP reply) - outbound IPv6 likely broken.
- conntrack pkg not confirmed installed; `apt install conntrack` if we want translation tuples.
