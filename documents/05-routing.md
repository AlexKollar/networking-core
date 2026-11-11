# Routing Fundamentals (CIS 171/172 — CCNA)

For the configs behind these concepts, the **navi-cisco** chip builds the actual IOS commands — this document teaches how the router *decides*, so the config stops being memorized text.

## What a Router Actually Does

A router connects networks. For each arriving packet it: strips the incoming frame, reads the **destination IP**, looks it up in the **routing table**, picks the best matching route, re-encapsulates the packet in a *new* frame for the outgoing link (new MACs, same IPs), and forwards it. If no route matches and no default exists, the packet is dropped and (usually) an ICMP unreachable goes back.

Three tables, kept separate in your head:

- **Routing table (RIB)** — chosen best routes; what `show ip route` prints.
- **MAC/frame handling** — per-link, rebuilt every hop (Layer 2 is local).
- **ARP cache / IPv6 neighbor table** — IP-to-MAC for the *next hop* on each local link.

## How a Router Decides — The Three-Step Rule

When several routes could carry a packet, the router applies these tests **in this order**:

1. **Longest prefix match wins.** The most specific route (longest mask) is always preferred, regardless of source. A `/25` beats a `/24` beats a `/16` beats the default `/0`. This is the rule students forget because the other two get more airtime — and it is applied *first*.
2. **Lowest Administrative Distance (AD)** breaks ties *between different sources* offering the **same prefix length to the same destination**. AD rates how trustworthy the source of the route is — lower is believed more.
3. **Lowest metric** breaks ties *within one routing protocol*. Each protocol computes cost its own way; metrics are never compared across protocols (that is what AD already settled).

Reference AD values (Cisco defaults — memorize the top of the list):

| Route source | AD |
|--------------|----|
| Connected interface | 0 |
| Static route | 1 |
| EIGRP summary | 5 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| iBGP | 200 |
| Unreachable / unknown | 255 (never installed) |

Mnemonic shape: direct knowledge (0, 1) beats everything; a protocol you configured neighbor-by-neighbor (BGP external, 20) beats math-heavy interior protocols; OSPF's 110 vs RIP's 120 settles the classic pairing.

## Static and Default Routes

**Static routes** are hand-entered: destination network, mask, and either a next-hop IP or an exit interface. They are the right tool for small/stable networks, stubs, and backup paths — and the wrong tool when the topology grows, because every change is manual and nothing adapts to failure automatically.

- **Default route** — `0.0.0.0/0` (IPv6: `::/0`): "if nothing else matches, go here." Typically points toward the Internet edge. A default is just the least-specific longest-match entry (length 0), which is why rule 1 puts it last automatically.
- **Floating static** — a static route given a deliberately *worse AD* than the primary dynamic route, so it installs only when the primary disappears. The standard backup-link pattern.
- **Static route forms** — recursive (next-hop IP only; router must resolve how to reach it), directly attached (exit interface only; fine on point-to-point, problematic on multi-access Ethernet), and fully specified (both — the form that avoids ambiguity).

Reading `show ip route` codes: `C` connected, `L` local (the router's own interface address, /32), `S` static, `O` OSPF, `D` EIGRP, `R` RIP, `B` BGP, `S*` candidate default, `*` in the *Gateway of last resort* line is your default in effect.

## OSPF Single-Area Concepts

**OSPF (Open Shortest Path First)** is the link-state interior protocol of the curriculum. Link-state means: every router learns the *map* of the area (the LSDB — link-state database, identical on all routers in the area), then runs Dijkstra's SPF algorithm to compute its own shortest tree. Contrast distance-vector protocols (RIP) that trade rumors of distances instead.

Core vocabulary:

- **Area** — a contained LSDB domain. Single-area designs use **Area 0 (the backbone)**. Multi-area OSPF must connect every other area to Area 0; that is beyond single-area scope but the rule is testable.
- **Router ID (RID)** — a 32-bit dotted-decimal name, chosen by: manual `router-id` command > highest loopback IP > highest active interface IP. Set it manually — stability beats surprises.
- **Neighbor / adjacency** — routers on a link exchange Hellos (every 10 s on broadcast/ point-to-point links; dead after 40 s by default). Adjacent routers sync LSDBs. `show ip ospf neighbor` is the health check.
- **Cost** — OSPF metric = reference bandwidth ÷ interface bandwidth (default reference 10^8, so Fast Ethernet and faster all start flattening at cost 1 — modern designs raise the reference bandwidth). Lowest cumulative cost wins *inside* OSPF, after longest-match and AD have had their say.
- **Designated Router (DR) / Backup DR (BDR)** — on multi-access segments (Ethernet), one router is elected **DR** so all others form adjacencies to it instead of to everyone (n routers would otherwise need n×(n−1)/2 adjacencies). BDR listens ready to take over. Election: highest OSPF interface **priority** (default 1; 0 = never), tie-break highest RID. On point-to-point links there is **no DR election** — a favorite exam distinction.
- **Neighbor states** (concept level, in order): Down → Init → 2-Way (DR/BDR election happens here; staying at 2-Way with non-DR routers is *normal*) → ExStart → Exchange → Loading → **Full**. Stuck states are troubleshooting gold: mismatched area, timers, MTU, or authentication all stall the sequence at identifiable steps — navi-cisco's OSPF troubleshoot topic walks that checklist.

Wildcard-mask note: OSPF `network` statements and ACLs use **wildcard masks** (inverse of subnet mask — /24 → 0.0.0.255). Navi-cisco computes and verifies them; the routing doc's job is the why: the network statement means "interfaces whose IP falls in this range join OSPF in this area."

## First-Hop Redundancy — Concept

The default gateway is a single point of failure: if the one router a subnet points at dies, every host is stranded even if a second router sits idle beside it. **First Hop Redundancy Protocols (FHRPs)** solve this: two (or more) routers share one **virtual IP** (and virtual MAC) that hosts use as their gateway; if the active forwarder fails, the standby takes the identity over and hosts notice nothing.

- **HSRP** — Cisco's protocol: active/standby roles, priority elects (highest wins; preemption must be enabled to take the role *back*).
- **VRRP** — the open standard, same idea, master/backup vocabulary.
- **GLBP** — Cisco, adds load balancing across forwarders.

Curriculum expectation is the concept, the virtual-IP mechanism, and comparing the three names — full configuration depth is lighter here than for OSPF.
