# Ethernet and Switching (CIS 171 — CCNA)

## The Ethernet Frame

A Layer 2 Ethernet frame wraps the packet for one hop across one link:

| Field | Purpose |
|-------|---------|
| Preamble + SFD | Clock sync and "frame starts now" — on the wire, not counted in frame size |
| Destination MAC | 6 bytes — who on *this link* receives it |
| Source MAC | 6 bytes — who on *this link* sent it |
| Type / EtherType | What is inside (0x0800 = IPv4, 0x86DD = IPv6, 0x0806 = ARP) |
| Data / Payload | The Layer 3 packet (46–1500 bytes; the 1500 is the standard MTU) |
| FCS | Frame Check Sequence — CRC error detection for this hop only |

A **MAC address** is 48 bits: the first 24 are the **OUI** (vendor block), the last 24 are device-specific. The first octet's low bits carry meaning: I/G bit set = group (multicast/broadcast), U/L bit set = locally administered (VMs, randomized phone Wi-Fi addresses). Broadcast is `FF:FF:FF:FF:FF:FF` — every device on the VLAN.

For MAC anatomy practice, the **navi-netkit** chip parses MAC forms, OUI, and the I/G and U/L bits live.

## How a Switch Learns and Forwards

A switch builds a **MAC address table (CAM table)** by watching source addresses:

1. Frame arrives on port 5 with source MAC `AA`. Switch records: `AA → port 5` (with a VLAN).
2. Destination is known → forward out that one port only (**forward**).
3. Destination unknown (unknown unicast), broadcast, or multicast → **flood** out all ports in the VLAN except the arrival port.
4. A frame whose destination already lives on the arrival port is **filtered** (dropped) — no need to forward it back.

Entries age out (default 300 seconds on Cisco switches) so moved devices are relearned. Troubleshooting anchor: most switching mysteries are answered by `show mac address-table` — is the MAC learned, on the port and in the VLAN you expect?

Switches forward in hardware (ASICs) and forward each VLAN as its own broadcast domain. Collisions are history on full-duplex point-to-point links — CSMA/CD still exists in the standards for half-duplex, which you should recognize but will rarely meet.

## VLANs and Trunking

A **VLAN (Virtual LAN)** splits one physical switch (or switch fabric) into separate logical broadcast domains: ports in VLAN 10 cannot hear VLAN 20 broadcasts, and traffic between VLANs must go through a Layer 3 device (router or Layer 3 switch SVI). Benefits exams cite: segmentation, security boundaries, smaller broadcast/failure domains, grouping by function instead of physical location.

- **Access port** — carries exactly one VLAN; what end devices plug into. Untagged traffic.
- **Trunk port** — carries many VLANs switch-to-switch (or switch-to-router/server); frames are **tagged** so the far end knows each frame's VLAN.
- **802.1Q** is the tagging standard: it inserts a 4-byte tag (with a 12-bit VLAN ID, range 1–4094) after the source MAC. VLAN 1 is the factory default VLAN.
- **Native VLAN** — the one VLAN whose frames cross a trunk **untagged** (default VLAN 1; best practice: change it to an unused VLAN and match it on both ends). A native VLAN mismatch leaks frames between VLANs and generates warnings — it is both a fault and an attack surface.
- **Voice VLAN** — a separate VLAN assigned on the same access port for an IP phone, so voice gets its own broadcast domain and QoS treatment while the PC behind the phone sits in the data VLAN.

Legacy note: ISL was Cisco's pre-standard trunking protocol, and DTP is Cisco's *automatic* trunk negotiation — see the security doc in this pack for why trunk negotiation is disabled in hardened designs.

Inter-VLAN routing forms: **router-on-a-stick** (one router interface, subinterfaces per VLAN over a trunk) vs **Layer 3 switch with SVIs** (switched virtual interfaces — faster, the modern default inside a site).

## Spanning Tree in Plain Words (STP / RSTP)

Loops are fatal at Layer 2: frames have no TTL, so a looped broadcast circles forever and multiplies — a broadcast storm. **Spanning Tree Protocol (802.1D)** prevents loops by electing a reference point and *blocking* redundant links until a failure needs them.

1. **Root bridge election** — switches exchange BPDUs; the switch with the lowest **Bridge ID** (priority, default 32768, plus VLAN, then lowest MAC) becomes root. Design rule: *choose* your root deliberately by lowering its priority; never leave it to MAC-address chance.
2. **Root ports** — each non-root switch picks its single best path toward the root (lowest cumulative path cost; ties broken by neighbor BID, then port ID).
3. **Designated ports** — each segment elects one port to forward toward the root's segments. Everything else **blocks**.

Classic STP port states, in order a port passes through on the way up: **Blocking → Listening → Learning → Forwarding** (Disabled sits outside the sequence). Classic convergence takes 30–50 seconds. **RSTP (802.1w)** is the rapid version taught for modern networks: it collapses states to **Discarding, Learning, Forwarding**, negotiates point-to-point links with a proposal/agreement handshake, and converges in about a second or less. Cisco names to recognize: PVST+ (per-VLAN classic STP, Cisco default heritage), Rapid PVST+, and MST (802.1s, maps VLAN groups to instances).

Port roles worth one line each: **PortFast** (Cisco) skips the waiting states on edge/access ports where a loop cannot form; **BPDU Guard** err-disables a PortFast port if a switch is plugged into it (see the security doc).

## EtherChannel — Bundling Links

**EtherChannel (port channel)** bundles 2–8 identical physical links into one logical link: STP sees a single link (no blocking waste), bandwidth adds up, and one member failing just reduces capacity. All members must match on speed, duplex, VLAN/trunk settings — a mismatch is the standard cause of a channel that will not form. Negotiation protocols: **LACP (802.3ad, IEEE standard — prefer this)** and Cisco's legacy PAgP; "on" mode forces the bundle with no negotiation at all. Load balancing hashes flow attributes (source/destination MAC, IP, or port) — one flow always rides one member, so a single transfer never exceeds one link's speed.

## CDP and LLDP — Neighbor Discovery at Layer 2

- **CDP (Cisco Discovery Protocol)** — Cisco proprietary, on by default on Cisco gear: neighbor device ID, platform, IP address, and port, via `show cdp neighbors`. 
- **LLDP (802.1AB)** — the vendor-neutral standard, same idea; often must be enabled on Cisco devices.

Both are the fastest way to map an undocumented closet: plug in, ask the switch who it sees. Both also *advertise* your topology to anything plugged in — hardened designs weigh that (security doc).
