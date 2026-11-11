# OSI and TCP/IP Models (CIS 170 — CCNA)

## Why Two Models Exist

The **OSI model** (Open Systems Interconnection, 7 layers) is the teaching and troubleshooting reference — it names every job precisely. The **TCP/IP model** (4 layers in Cisco's curriculum, sometimes shown as 5) describes how the real Internet protocol suite is built. Exams test OSI vocabulary; real gear runs TCP/IP. Learn OSI layer names cold, then map TCP/IP onto them.

Memory hook for OSI, top to bottom: **All People Seem To Need Data Processing** (Application, Presentation, Session, Transport, Network, Data Link, Physical). Bottom to top: **Please Do Not Throw Sausage Pizza Away**.

## OSI Layer by Layer

| # | Layer | Job in plain words | Lives here | PDU name |
|---|-------|--------------------|------------|----------|
| 7 | Application | The interface the user/program actually touches; network services to applications | HTTP, HTTPS, DNS, DHCP, FTP, SMTP, SNMP, SSH, Telnet | Data |
| 6 | Presentation | Format, encrypt, compress — makes data readable to the receiver | TLS/SSL, JPEG, ASCII/Unicode encoding | Data |
| 5 | Session | Open, manage, and close conversations between applications | Session setup/teardown, RPC-style dialogs | Data |
| 4 | Transport | End-to-end delivery between processes: segmentation, reliability, ports | TCP, UDP | **Segment** (TCP) / Datagram (UDP) |
| 3 | Network | Logical addressing and routing between networks | IPv4, IPv6, ICMP, routers, routing protocols (OSPF) | **Packet** |
| 2 | Data Link | Delivery on one local link: framing, MAC addresses, switching | Ethernet, 802.1Q, PPP, switches, NICs | **Frame** |
| 1 | Physical | Bits on a medium: voltages, light, radio, connectors, cabling | Copper, fiber, hubs, repeaters, Wi-Fi radio | **Bit** |

Exam traps:

- A **switch** is a Layer 2 device (it forwards by MAC address). A **router** is Layer 3 (it forwards by IP address). A hub/repeater is Layer 1 — it understands nothing.
- Layer 3 PDU is the **packet**, Layer 2 is the **frame**, Layer 4 TCP is the **segment**. Getting these three backwards is the most common model question miss.
- Encryption is discussed at Layer 6 (Presentation) in OSI teaching, even though TLS in practice sits between Transport and Application in the real stack. Answer the model the question asks about.

## TCP/IP Model

Cisco's 4-layer TCP/IP model collapses OSI like this:

| TCP/IP layer | Covers OSI | Protocols / examples |
|--------------|------------|----------------------|
| Application | 7 + 6 + 5 | HTTP, DNS, DHCP, SMTP, SSH, TLS |
| Transport | 4 | TCP, UDP |
| Internet | 3 | IPv4, IPv6, ICMP, IPsec |
| Network Access (Link) | 2 + 1 | Ethernet, Wi-Fi (802.11), ARP* |

*ARP is the classic argument: it maps Layer 3 addresses to Layer 2 addresses, so different books file it under Internet or Network Access. Cisco places ARP discussion with the Ethernet/Data Link world. If a question forces one answer, follow the curriculum in front of you and note the ambiguity — do not fight it.

Some textbooks use a 5-layer hybrid (Physical and Data Link kept separate). It is the same territory, sliced once more.

## Encapsulation — The One Process Everything Hangs On

As data travels **down** the sending stack, each layer wraps it with its own header (and at Layer 2, a trailer). The receiver unwraps in reverse. Names change at each step:

```
Application data
  -> Transport adds port + control header      = SEGMENT
  -> Network adds source/destination IP        = PACKET
  -> Data Link adds MAC header + FCS trailer   = FRAME
  -> Physical sends BITS
```

Say it as a sentence: **data becomes a segment, becomes a packet, becomes a frame, becomes bits.** On receipt it is de-encapsulation: bits → frame → packet → segment → data.

Key details exams probe:

- The Layer 2 **FCS (Frame Check Sequence)** trailer detects corruption on that one link. It is checked and rebuilt at every hop — it is error *detection*, not correction.
- A router **strips and rebuilds the frame at every hop**: the packet (and its IP addresses) stays essentially the same end to end, while MAC addresses change link by link. This single fact explains most OSI troubleshooting questions.
- Ports live at Layer 4. IP addresses live at Layer 3. MAC addresses live at Layer 2. When a scenario says "the address did not change across the router," ask *which* address — the answer splits by layer.

## Troubleshooting With the Model

Work bottom-up for physical faults (cable, link light, speed/duplex), top-down for application reports, and divide-and-conquer from Layer 3 (`ping`) when you have no lead. Naming the layer a tool works at tells you what it can prove: `ping` proves Layer 3 reachability, not that the web service (Layer 7) is up; a link light proves Layer 1, not an IP address.

For hands-on address math while you study this layering, the **navi-netkit** chip computes subnet, port, and MAC facts live — this pack explains the models, netkit does the numbers.
