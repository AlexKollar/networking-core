# Transport Layer and Core Network Services (CIS 170/172 — CCNA)

For live log follow-up on several services below, the **navi-scribe** chip parses syslog and service logs and flags what matters — this document explains what those services *are*.

## TCP vs UDP — An Honest Comparison

Both live at Layer 4 and both use **port numbers** to deliver data to the right application process on a host. They differ in promises:

| | TCP | UDP |
|---|-----|-----|
| Connection | Connection-oriented: a session is set up first | Connectionless: just send it |
| Reliability | Guaranteed: sequence numbers, acknowledgments, retransmission | Best-effort: no ACKs, no retransmission |
| Ordering | Delivers bytes in order | No ordering guarantee |
| Overhead | 20-byte minimum header, setup cost | 8-byte header, minimal |
| Flow / congestion control | Yes — sliding window, congestion awareness | None built in |
| Used by | Web (HTTP/HTTPS), SSH, SMTP, FTP — correctness matters | DNS, DHCP, VoIP, streaming, SNMP, NTP — speed matters, or the app handles recovery |

The honest framing: UDP is not "unreliable TCP" as a flaw — it trades guarantees for latency and simplicity, and real-time traffic would rather drop a packet than wait for a re-send of stale audio. QUIC (HTTP/3's base) shows the third path: reliability rebuilt *on top of* UDP by the application.

## The TCP Three-Way Handshake and Teardown

A TCP connection opens with three segments — the question CCNA always asks, in this exact order:

1. **SYN** — client → server: "I want to talk; my starting sequence number is X."
2. **SYN-ACK** — server → client: "Acknowledged (X+1); my starting number is Y."
3. **ACK** — client → server: "Acknowledged (Y+1)." Data may now flow.

During the session, **sequence numbers** order the byte stream, **acknowledgments** confirm receipt, lost segments are retransmitted, and the **window size** advertises how much the receiver can buffer (flow control). Teardown is a FIN/ACK exchange in both directions (often drawn as four steps; a RST aborts immediately). Key flags by name: SYN, ACK, FIN, RST, PSH, URG — SYN and ACK are the pair the handshake question is built from.

## Port Numbers

Ports are 16-bit (0–65535), in three ranges:

- **Well-known: 0–1023** — assigned services (HTTP 80, HTTPS 443, DNS 53…).
- **Registered: 1024–49151** — vendor/application registrations.
- **Dynamic / ephemeral: 49152–65535** — the client side's temporary source ports. A connection is uniquely identified by the 4-tuple: source IP, source port, destination IP, destination port.

The navi-netkit chip carries the full exam port table (43 services) — this pack's service map below ties the big ones to their jobs.

## The Service Map — What Each Does and Where It Sits

### DNS — Domain Name System (UDP 53, TCP 53 for large responses/zone transfers)
Translates names to addresses (`www.example.com` → an A record for IPv4, AAAA for IPv6). Resolution chain: local cache → recursive resolver → root → TLD → authoritative server. Record types to know: **A, AAAA, CNAME** (alias), **MX** (mail), **NS** (nameservers), **PTR** (reverse lookup), **TXT**. DNS is the service whose failure *looks like* "the Internet is down."

### DHCP — Dynamic Host Configuration Protocol (UDP 67 server / 68 client)
Hands out IP address, mask, default gateway, and DNS servers automatically. The exchange is **DORA**:

1. **Discover** — client broadcasts "any DHCP servers?" (it has no address yet: source 0.0.0.0, destination 255.255.255.255).
2. **Offer** — a server proposes an address and lease terms.
3. **Request** — client broadcasts acceptance of one offer (broadcast, so other servers withdraw theirs).
4. **Acknowledgment (ACK)** — server confirms; the lease starts.

Details that get tested: a relay agent (`ip helper-address` on Cisco) forwards DHCP broadcasts across router boundaries, because routers do not forward broadcasts by default. A client that gets no offer self-assigns **APIPA 169.254.x.x** — seeing that address in troubleshooting *is* the diagnosis. Renewal normally happens at half the lease time.

### NAT / PAT — Network / Port Address Translation
NAT rewrites private source addresses to public ones at the network edge, because RFC 1918 space cannot route on the Internet.

- **Static NAT** — one private ↔ one public, permanently (for a published server).
- **Dynamic NAT** — a pool of public addresses shared first-come.
- **PAT (NAT overload)** — many private hosts share **one** public address, distinguished by rewritten source port numbers. This is what every home router does and what "NAT" usually means in conversation.

NAT conserves addresses and hides internal structure, at real costs: it breaks strict end-to-end assumptions and complicates protocols that embed addresses in their payloads. IPv6's abundance is explicitly designed to retire the *need* for it.

### NTP — Network Time Protocol (UDP 123)
Synchronizes clocks across the network. Time is infrastructure: log correlation, certificate validity, and authentication protocols all fail strangely when clocks disagree. Accuracy is organized in **strata** — stratum 0 is a reference clock (GPS/atomic), stratum 1 servers sync to it directly, each hop adds a stratum. Point devices at a small set of internal NTP sources rather than the whole Internet.

### SNMP — Simple Network Management Protocol (UDP 161 queries, 162 traps)
How management stations monitor network devices: **GET** (ask a value), **SET** (change a value — restricted in practice), **TRAP / INFORM** (device-initiated alerts). Data is organized in MIBs, addressed by OIDs. Versions matter for security: v1/v2c "community strings" are effectively clear-text passwords shared by every device; **SNMPv3** adds authentication and encryption and is the only version to recommend.

### Syslog (UDP 514, conventionally)
The standard channel by which devices report events to a logging server. Severity levels run 0–7 and the numbering is inverted against intuition — **0 (Emergency) is the worst**, 7 (Debug) is the noisiest: 0 Emergency, 1 Alert, 2 Critical, 3 Error, 4 Warning, 5 Notification, 6 Informational, 7 Debugging. A common exam trap is to treat 7 as most severe; it is the least. Facilities identify the source subsystem. Centralized syslog is what lets navi-scribe read a network's memory after the fact.

### Also on the exam map
- **FTP** (TCP 20/21) — file transfer with a separate data channel; clear-text, superseded by SFTP (over SSH, 22) and FTPS.
- **TFTP** (UDP 69) — trivial file transfer; no authentication; lives on in network-boot and device config/image transfer, on trusted segments only.
- **SSH** (TCP 22) vs **Telnet** (TCP 23) — remote management, encrypted vs clear-text: the single most repeated security substitution in the curriculum.
- **SMTP/POP3/IMAP** (25 / 110 / 143, secure variants 587, 995, 993) — sending vs the two retrieval models (download-and-delete heritage vs server-side folders).
