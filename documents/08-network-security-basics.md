# Network Security Basics for Switching and Routing (CIS 172 — CCNA)

For the lab side of several topics here — ACL building and port scanning your *own* ranges — the **navi-cisco** chip builds ACLs and the **navi-recon** chip scans lab ranges with a defender's explainer; the **Security+ Core** pack in this series carries the wider security curriculum. This document is the network-infrastructure slice.

## AAA — The Framework Behind Logins to Devices

Managing who may configure a router or switch uses the same three words as the Security+ pack, applied to device administration:

- **Authentication** — prove who you are (local username database, or centrally via **RADIUS** / **TACACS+**).
- **Authorization** — what your proven identity may do (privilege levels and command sets).
- **Accounting** — a record of sessions and commands, sent to the AAA server — the audit trail after an incident.

RADIUS vs TACACS+, the comparison exams always draw: **TACACS+** is Cisco-origin, TCP 49, encrypts the **entire** AAA payload, and separates authentication from authorization (fine-grained command control) — the device-admin favorite. **RADIUS** is the open standard, UDP 1645/1812 (accounting 1646/1813), encrypts only the password field, and combines authentication+authorization — the *network-access* favorite (802.1X). Choose TACACS+ to administer the boxes; choose RADIUS to admit users onto the network.

## Access Control Lists — Concepts

An **ACL** is an ordered list of permit/deny rules that a router (or Layer 3 switch) matches packets against, top to bottom, **first match wins**, with an **implicit deny at the end** — traffic matching nothing is dropped, whether you wrote that line or not.

| | Standard ACL | Extended ACL |
|---|--------------|--------------|
| Matches | Source IP only | Source + destination IP, protocol, ports |
| Numbered range | 1–99 (and 1300–1999) | 100–199 (and 2000–2699) |
| Placement rule | **Close to the destination** | **Close to the source** |
| Why | It is blunt (source only) — place it late so it does not block traffic to places it was never meant to govern | It is precise — filter early and save the network from carrying doomed traffic |

Placement logic in one scenario: an extended ACL denying one subnet web access can sit at that subnet's own edge router, dropping the traffic where it is born. The equivalent standard ACL must sit near the web server — put it at the source and it blocks that subnet from *everything*, not just the web.

Other load-bearing facts: ACLs are applied to an interface in a direction (`in` or `out` — inbound ACLs save CPU because packets are dropped before routing). Wildcard masks express the address range (inverse of the subnet mask). Named ACLs are preferred in real work — they are editable line by line; numbered ACLs historically are rebuilt whole. And an ACL applied with a typo does not fail safe or fail open — it fails *exactly as written*, including its implicit deny, which is why ACL changes are made with a reload safety net in the field.

## Layer 2 Attack Vocabulary and the Standard Mitigations

Switches trust by default on old assumptions: that everything plugged in is a friend. These four attacks are the CCNA security canon — each pairs with its mitigation, which is how questions are structured.

### VLAN Hopping
Reaching a VLAN you were not assigned, either by impersonating a switch and negotiating a trunk (**switch spoofing** via DTP auto-negotiation), or by nesting 802.1Q tags (**double tagging**, which needs the attacker's port in the trunk's native VLAN to work). **Mitigations:** hard-set ports to access mode and disable DTP (`switchport nonegotiate`), park all unused ports in a dead VLAN and shut them down, change the native VLAN to an unused one and (on hardened trunks) tag native VLAN traffic.

### STP Manipulation
An attacker injects superior BPDUs and becomes root bridge, bending the topology through their device (a man-in-the-middle position) or destabilizing it. **Mitigations:** **BPDU Guard** (err-disables an edge port that receives any BPDU — pairs with PortFast on access ports) and **Root Guard** (prevents a port from accepting a superior BPDU that would relocate the root).

### ARP Poisoning / Spoofing
ARP has no authentication, so an attacker answers "who has the gateway's IP?" with *their own* MAC — and hosts happily send traffic through them (man-in-the-middle/DHCP-impersonation follow-ons). **Mitigation: Dynamic ARP Inspection (DAI)** — the switch validates ARP messages against the DHCP snooping binding table and drops ARP claims that contradict known leases (rate-limited, because validation costs CPU).

### DHCP Starvation and Rogue DHCP Servers
Starvation exhausts the pool with forged requests so the real server has nothing left to offer; a rogue server then (or instead) answers clients with itself as gateway/DNS. **Mitigation: DHCP Snooping** — the switch classifies ports as **trusted** (toward legitimate servers/uplinks) or **untrusted** (user ports); DHCP *server* messages arriving on untrusted ports are dropped, and a per-client binding table (MAC, IP, VLAN, port) is built as a side effect. That binding table is the foundation DAI and IP Source Guard both stand on — snooping is the one to install first.

### MAC Flooding and Port Security
A switch whose CAM table overflows degrades to flooding everything (a hub's behavior) — so attackers flood fake source MACs to make traffic sniffable. **Port security** is the direct answer: limit how many MACs a port may learn (and optionally pin exact addresses), with a defined violation reaction — **shutdown** (err-disable, the secure default), **restrict** (drop + count/log), or **protect** (drop silently). "Sticky" learning harvests the current legitimate MACs into the config automatically.

## Device Hardening Baseline — The Checklist Shape

The curriculum's minimal shape for any infrastructure device:

- Unique named accounts, no shared defaults; enable secret hashed strongly (Type 8/9/scrypt-family where supported — plain MD5-based Type 5 and reversible Type 7 are legacy).
- SSH only for management (disable Telnet), with a domain name, RSA key, and VTY lines restricted by an ACL to management hosts.
- Disable unused services and unused ports; put unused ports in a parking VLAN.
- A login banner stating authorized use (legal function, not decoration).
- Logging turned on with accurate time (syslog to a central collector + NTP sync — see this pack's services doc) and saved configs versioned off-box.
- Control-plane thinking: management traffic belongs on its own VLAN/VRF where the design supports it.

The thread tying the whole document together: every mitigation is about **not trusting the edge by default** — ports prove their role, ARP proves its claim, DHCP proves its serverhood, and admins prove their identity. That posture, applied at the infrastructure layer, is the practical core of this course.
