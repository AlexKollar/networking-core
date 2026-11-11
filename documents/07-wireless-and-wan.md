# Wireless LANs and WAN Concepts (CIS 171/172 — CCNA)

For WLAN security configuration tied to this material, the **navi-cisco** chip's wireless topics carry the IOS/WLC-side detail — this document builds the concepts underneath.

## 802.11 Generations and Bands

Wi-Fi standards are IEEE 802.11 amendments; the Wi-Fi Alliance's generation names (Wi-Fi 4/5/6…) are the friendly labels.

| Standard | Name | Bands | Headline capability* |
|----------|------|-------|----------------------|
| 802.11a | — | 5 GHz | 54 Mbps — the original 5 GHz (1999) |
| 802.11b | — | 2.4 GHz | 11 Mbps (1999) |
| 802.11g | — | 2.4 GHz | 54 Mbps |
| 802.11n | Wi-Fi 4 | 2.4 + 5 GHz | MIMO (multiple spatial streams), channel bonding to 40 MHz; up to 600 Mbps |
| 802.11ac | Wi-Fi 5 | 5 GHz only | Wider channels (80/160 MHz), MU-MIMO downlink; multi-Gbps class |
| 802.11ax | Wi-Fi 6 / 6E | 2.4 + 5 (+ 6 GHz for 6E) | OFDMA, better dense-environment efficiency — gains are about *sharing*, not just top speed |

*Headline numbers are laboratory PHY maxima, not throughput you will measure. Expect roughly half or less after protocol overhead, contention, and distance — an honest frame the exam increasingly rewards.

**The band trade, in one breath:** 2.4 GHz travels farther and penetrates walls better but offers few clean channels and shares space with Bluetooth, microwaves, and neighbors. 5 GHz (and 6 GHz) give more, wider channels and cleaner spectrum, at shorter range. Modern designs are 5 GHz-first, with 2.4 GHz kept for range and legacy/IoT.

## Channels — The 1/6/11 Rule

In the 2.4 GHz band, channel centers sit only 5 MHz apart while a signal occupies ~22 MHz — so most adjacent channels **overlap** and interfere. Only **channels 1, 6, and 11** (in North American regulatory domains) are mutually non-overlapping. Designing a multi-AP 2.4 GHz network means reusing only 1/6/11 in a honeycomb so neighboring cells never share or partially overlap a channel. Partial overlap (e.g., channel 3 next to channel 1) is *worse* than sharing a channel outright: co-channel APs can coordinate via CSMA/CA and take turns; partially-overlapping ones just corrupt each other as noise.

In 5 GHz, channels are spaced so standard 20 MHz channels do not overlap, and channel bonding (40/80/160 MHz) trades the pool of independent channels for per-link width — a real design decision in dense spaces, not a free upgrade.

How WLANs share the air: **CSMA/CA** — *Collision Avoidance*, because radios cannot listen while transmitting. Devices wait for a clear channel plus a randomized backoff; hidden nodes (two clients that cannot hear each other but both reach the AP) are mitigated with RTS/CTS. This contention model is why Wi-Fi is half-duplex per channel and why airtime, not headline speed, is the scarce resource.

## WLAN Architecture Vocabulary

- **BSS** — one access point and its clients (one *Basic Service Set*, identified by the AP radio's BSSID — essentially a MAC).
- **ESS** — multiple APs advertising the same **SSID** (the human-facing network name) so clients **roam** between them as they move.
- **Distribution system** — the wired network tying the APs together.
- **Controller models** — autonomous APs (each configured alone) vs **split-MAC / lightweight APs** reporting to a **Wireless LAN Controller (WLC)** over CAPWAP tunnels. Controllers centralize RF management — channel/power assignment, roaming, and rogue detection — and are the curriculum's enterprise model.
- A client chooses which AP to join (the standard leaves it to the client) — design guides clients with band steering and sane power levels, not force.

## WLAN Security — WPA2 vs WPA3

- **Open** — no authentication or encryption; acceptable only behind a captive portal for genuinely public access, and traffic is sniffable by design.
- **WEP** — broken cryptography; **WPA (TKIP)** — its stopgap successor, also deprecated. If either appears in a scenario, the correct answer is "replace it."
- **WPA2 (802.11i)** — AES-CCMP encryption. Two authentication models: **Personal** (one pre-shared key, PSK, for everyone) and **Enterprise** (802.1X/EAP against a RADIUS server — per-user credentials, the business standard).
- **WPA3** — the current generation: replaces the PSK handshake with **SAE (Simultaneous Authentication of Equals)**, which resists offline dictionary attacks against captured handshakes and gives forward secrecy; mandates **Protected Management Frames (PMF/802.11w)** so deauthentication frames cannot be forged casually (the classic "deauth attack" is thereby blunted, not merely detected). Enterprise mode can require 192-bit-class cryptography suites. Transition mode runs WPA2/WPA3 together for older clients — convenient, and a downgrade surface to close when the fleet allows.

Rogue APs (an unauthorized AP plugged into your LAN) and evil twins (an AP impersonating your SSID) are the wireless-specific intrusions a WLC's rogue detection exists to catch.

## WAN Topologies and Terms

- **WAN** — a network spanning distance, typically over provider links you lease rather than media you own. The CE (customer edge) router meets the provider at a **demarcation point**; terms DCE/DTE survive from serial days.
- **Topologies** — point-to-point (two sites, one link), hub-and-spoke (branches through one center — simple, and the center is a chokepoint and single point of failure), full mesh (every site to every site — resilient, costs grow quadratically), partial mesh (the practical compromise), dual-homed (a branch with links to two hubs).
- **Access technologies in the curriculum's frame** — broadband (DSL, cable) for small sites; Metro Ethernet; MPLS (provider label-switched private WAN, traffic classes over shared provider core); and plain Internet with VPN overlaid — the steadily winning economics for many designs.

## VPN Concepts

A **Virtual Private Network** builds an encrypted tunnel over an untrusted path, principally the Internet.

- **Site-to-site VPN** — gateways encrypt between whole networks; hosts never know. The standard branch-office connector over Internet links.
- **Remote-access VPN** — one user's device tunnels to a head-end on demand (client-based, or clientless via a browser portal).
- **IPsec** (concept level) — the framework site-to-site VPNs are built from: **IKE** negotiates keys and builds security associations (Phase 1 builds the management tunnel; Phase 2 builds the data tunnels), **AH** authenticates/integrity-checks (no encryption), **ESP** encrypts *and* authenticates — ESP is what real deployments use. IPsec runs in tunnel mode (whole original packet wrapped — the VPN case) or transport mode (endpoints themselves speak IPsec).
- **GRE** — a simple tunnel that can carry routing protocols and multicast, but has **no encryption**; GRE over IPsec is the classic pairing when you need both dynamic routing through the tunnel and confidentiality.
- **TLS VPNs** — remote access over TCP/UDP 443, chosen for friendliness through firewalls (DTLS restores UDP behavior where possible).

Decision habit: tunneling answers "how do packets get there privately," encryption answers "who can read them in transit" — a tunnel without encryption (GRE alone, or a bare encapsulation question) is not private.
