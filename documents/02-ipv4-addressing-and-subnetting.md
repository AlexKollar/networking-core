# IPv4 Addressing and Subnetting (CIS 170 — CCNA)

For live practice while you read: the **navi-netkit** chip does subnet, CIDR, port, and MAC math on demand. This document teaches the method so the answers stop being magic.

## Anatomy of an IPv4 Address

An IPv4 address is 32 bits, written as four decimal octets (dotted decimal), e.g. `192.168.10.37`. Every address has two parts — **network** and **host** — and the **subnet mask** (or prefix length, `/24`) draws the line between them. Mask bits that are `1` are network; bits that are `0` are host.

- `192.168.10.37/24` — mask `255.255.255.0`. Network part: `192.168.10`. Host part: `37`.
- Two addresses are on the **same subnet** when their network parts match under the same mask. Same subnet = talk directly (Layer 2). Different subnet = send it to the default gateway (a router).

Three addresses matter in every subnet:

- **Network address** — all host bits `0`. Names the subnet itself; never assign it to a host.
- **Broadcast address** — all host bits `1`. Reaches every host in the subnet; never assign it.
- **Usable hosts** — everything between them: `2^(host bits) − 2`.

## Classes — Historical Framing, Still Tested Lightly

Classful addressing is how the Internet *started*; CIDR replaced it in practice, but the vocabulary remains:

| Class | First octet | Original mask | Notes |
|-------|-------------|---------------|-------|
| A | 1–126 | /8 (255.0.0.0) | Huge networks. `127.x` is loopback, not Class A hosts |
| B | 128–191 | /16 (255.255.0.0) | Medium networks |
| C | 192–223 | /24 (255.255.255.0) | Small networks |
| D | 224–239 | — | Multicast — no host addressing |
| E | 240–255 | — | Experimental / reserved |

Modern routing is **classless (CIDR)** — the prefix length in the address is the truth, not the class. When a question mentions a class, it is usually testing whether you know the *default* mask or the private range below.

## Private and Special Addresses — Memorize These

**RFC 1918 private ranges** (not routed on the public Internet; used with NAT):

- `10.0.0.0/8` — 10.0.0.0 to 10.255.255.255
- `172.16.0.0/12` — 172.16.0.0 to 172.31.255.255 (the range everyone gets wrong — it stops at 172.31, not 172.255)
- `192.168.0.0/16` — 192.168.0.0 to 192.168.255.255

**Special addresses:**

| Address / range | Meaning |
|-----------------|---------|
| `127.0.0.1` (::1 is the IPv6 twin) | Loopback — the host itself |
| `169.254.0.0/16` | APIPA / link-local — self-assigned when DHCP fails; a symptom, not a plan |
| `0.0.0.0` | "This host / unspecified"; in a route table, `0.0.0.0/0` is the default route |
| `255.255.255.255` | Limited broadcast — this local network only, never forwarded by a router |
| `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` | Documentation (TEST-NET) ranges — the examples in books and in recon/scribe sample data |
| `100.64.0.0/10` | Carrier-grade NAT shared space — *not* RFC 1918 private, a common trap |

## The Subnetting Method — Step by Step

Given a network and a requirement, work it in this order, every time:

1. **Find how many bits you must borrow.** Need N subnets? Smallest `s` with `2^s ≥ N`. Need H usable hosts per subnet? Smallest `h` with `2^h − 2 ≥ H`. Borrowed bits come from the host portion.
2. **New prefix = old prefix + borrowed subnet bits.** Write the new mask in dotted decimal.
3. **Block size (increment)** = `256 − the interesting mask octet` — the subnets step by that number in that octet.
4. **List subnets by counting in block-size steps** from zero.
5. **For each subnet:** network = the step value, broadcast = next network − 1, usable = network+1 to broadcast−1.

Powers of two to know without thinking: 2^1=2, 2^2=4, 2^3=8, 2^4=16, 2^5=32, 2^6=64, 2^7=128, 2^8=256. Common masks: /25=255.255.255.128, /26=.192, /27=.224, /28=.240, /29=.248, /30=.252.

### Worked Example 1 — Equal Subnets (verified)

**Task:** split `192.168.10.0/24` into 4 subnets.

- Step 1: 4 subnets needs `2^2 = 4`, so borrow **2 bits**. Host bits left: 8 − 2 = 6 → `2^6 − 2 = 62` usable hosts each.
- Step 2: new prefix = 24 + 2 = **/26**, mask `255.255.255.192`.
- Step 3: block size = 256 − 192 = **64**.
- Steps 4–5:

| Subnet | Network | Usable range | Broadcast |
|--------|---------|--------------|-----------|
| 1 | 192.168.10.0/26 | 192.168.10.1 – 192.168.10.62 | 192.168.10.63 |
| 2 | 192.168.10.64/26 | 192.168.10.65 – 192.168.10.126 | 192.168.10.127 |
| 3 | 192.168.10.128/26 | 192.168.10.129 – 192.168.10.190 | 192.168.10.191 |
| 4 | 192.168.10.192/26 | 192.168.10.193 – 192.168.10.254 | 192.168.10.255 |

### Worked Example 2 — VLSM (verified)

**VLSM (Variable Length Subnet Masking)** means subnets of *different* sizes carved from one block — always carve **largest first**, so blocks stay aligned.

**Task:** carve `192.168.33.0/24` for: Sales 50 hosts, Engineering 25, IoT 12, one point-to-point WAN link (2).

| Need | Host bits | Prefix | Network | Usable range | Broadcast |
|------|-----------|--------|---------|--------------|-----------|
| Sales 50 | 6 (62 usable) | /26 | 192.168.33.0 | .1 – .62 | 192.168.33.63 |
| Eng 25 | 5 (30 usable) | /27 | 192.168.33.64 | .65 – .94 | 192.168.33.95 |
| IoT 12 | 4 (14 usable) | /28 | 192.168.33.96 | .97 – .110 | 192.168.33.111 |
| WAN 2 | 2 (2 usable) | /30 | 192.168.33.112 | .113 – .114 | 192.168.33.115 |

Used: .0–.115. Free: `192.168.33.116–192.168.33.255` for growth. Notes exams love: a /30 gives exactly 2 usable addresses — the classic point-to-point size (and /31 is used for point-to-point links in modern practice per RFC 3021, where both addresses are usable because there is no broadcast on a two-endpoint link).

## Route Summarization

Summarization (aggregation) advertises several contiguous subnets as one route. Find the common prefix bits — the summary covers the lowest to the highest address in the group.

**Example (verified):** `192.168.4.0/24`, `192.168.5.0/24`, `192.168.6.0/24`, `192.168.7.0/24` summarize to **`192.168.4.0/22`** — four /24s = 2 borrowed bits back (24 − 2 = 22), covering .4.0 through .7.255. Summaries only work cleanly on power-of-two, boundary-aligned groups; a group that is not aligned cannot be covered by a single exact summary without also covering addresses outside it.
