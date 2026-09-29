# Networking — Complete Interview Guide

> Document 2 of your DevOps interview series: **Linux → Networking → Docker → Kubernetes → CI/CD → AWS → Terraform.**
> Goal: trace a real request packet-by-packet, troubleshoot systematically, and walk into AWS networking later without relying on memorized answers. Covers what a 3–5 YOE DevOps engineer needs today — not every networking topic that exists.

**Callout legend:** 🔵 `NOTE` = Anchor Fact · 🟡 `WARNING` = Failure Scenario · 🟢 `TIP` = Interview Answer · 🔧 `IMPORTANT` (HANDS-ON) = Practical Lab · 🟣 `IMPORTANT` (LIVE-CODING/LIVE-DIAGNOSIS) = Interview Task.

**Core vs Stretch:** Each Foundations topic is tagged. **CORE** = must be instantly fluent in this for a 3–5 YOE interview. **STRETCH** = good depth that differentiates you, but not where to spend limited time first if you're behind schedule.

---

## Quick Jump Index

**Part 1 — Foundations**
1. [Why Networking Exists, and the Layered Model](#1-why-networking-exists-and-the-layered-model) `CORE`
2. [IP Addressing & CIDR](#2-ip-addressing--cidr) `CORE`
3. [Ports & the Transport Layer: TCP vs UDP](#3-ports--the-transport-layer-tcp-vs-udp) `CORE`
4. [DNS Resolution](#4-dns-resolution) `CORE`
5. [The TCP Handshake & Connection Lifecycle](#5-the-tcp-handshake--connection-lifecycle) `CORE`
6. [Routing & Gateways](#6-routing--gateways) `CORE`
7. [NAT](#7-nat) `CORE`
8. [Firewalls, Security Groups & NACLs](#8-firewalls-security-groups--nacls) `CORE`
9. [HTTP/HTTPS & TLS](#9-httphttps--tls) `CORE`
10. [Load Balancers](#10-load-balancers) `CORE`
11. [VPC Endpoints](#11-vpc-endpoints) `CORE`
12. [Connecting VPCs and On-Premises: Peering, TGW, VPN, Direct Connect](#12-connecting-vpcs-and-on-premises-peering-transit-gateway-vpn-direct-connect) `CORE`

**Part 2 — Deeper Mechanics**
- [The Full Request Journey, Packet by Packet](#the-full-request-journey-packet-by-packet) `CORE`
- [Sockets and the ss Model](#sockets-and-the-ss-model) `CORE`
- [How traceroute/mtr Actually Work](#how-traceroutemtr-actually-work) `CORE`
- [MTU & Fragmentation](#mtu--fragmentation) `STRETCH`
- [CGNAT & Public vs Private IP Space](#cgnat--public-vs-private-ip-space) `STRETCH`
- [ARP & ICMP: The Control-Plane Basics Underneath IP](#arp--icmp-the-control-plane-basics-underneath-ip) `CORE`
- [Bridge to Container Networking (Preview)](#bridge-to-container-networking-preview--full-depth-in-later-documents) `CORE`

**Part 3 — Interview Corner**
- [DNS & Resolution — full 9-angle drill](#dns--resolution-drill)
- [TCP, Ports & Connections — full 9-angle drill](#tcp-ports--connections-drill)
- [Firewalls & Security Groups — full 9-angle drill](#firewalls--security-groups-drill)
- [Load Balancing — full 9-angle drill](#load-balancing-drill)
- [Quick-Coverage Round](#quick-coverage-round)
- [Hot Topics Right Now](#hot-topics-right-now)
- [Research-Backed Additions (Current Market, 2025–2026)](#research-backed-additions-current-market-20252026)

**Part 4 — Real Incident / Story Bank**
- [Story 1: The ECR VPC Endpoint for Private EKS Nodes](#story-1-the-ecr-vpc-endpoint-for-private-eks-nodes)
- [Story 2: The ALB Controller Webhook Timeout](#story-2-the-alb-controller-webhook-timeout)
- [Practice Scenario (rehearsal only)](#practice-scenario-rehearsal-only)

**Part 5 — Master Glossary (Active Recall)**
- [A–Z Quick Recall](#master-glossary-active-recall)

**Part 6 — Additional Practical Exercises**
- [1: Subnetting Under Pressure](#exercise-1-subnetting-under-pressure)
- [2: Firewall Rule for a New Service (moved here from Linux)](#exercise-2-firewall-rule-for-a-new-service)

---

# Part 1 — Foundations

## 1. Why Networking Exists, and the Layered Model
`CORE`

**Why it exists:** A single computer doing everything alone is limited. Networking lets separate machines — a browser on a laptop, a web server, a database — exchange data as if they were talking directly, even though the actual path between them involves dozens of intermediate devices they never see. Almost every real-world DevOps incident that "isn't a code bug" is a networking problem wearing a different costume — DNS not resolving, a security group silently dropping traffic, a load balancer health check failing.

**The layered model, practically:** You'll hear both OSI (7 layers) and TCP/IP (4 layers) — in real troubleshooting, only a handful of layers matter daily:

| Layer (practical) | What lives here | DevOps relevance |
|---|---|---|
| Application | HTTP, DNS, SSH, TLS | App-level errors, status codes, cert issues |
| Transport | TCP, UDP, ports | Security groups, load balancer listeners, connection state |
| Network | IP, routing | Route tables, VPC peering, NAT |
| Link | Ethernet, ARP, MAC | Rarely touched directly in cloud — ENIs abstract this away |

> [!NOTE]
> **Anchor Fact:** You will almost never debug at the Link layer in a cloud/DevOps role — ENIs and virtual networking abstract it away. Nearly all real troubleshooting happens at Application, Transport, and Network. Don't over-invest study time in OSI trivia below Layer 3.

> [!TIP]
> **Interview Answer:** "I think in terms of four practical layers: application protocols like HTTP and DNS, the transport layer where TCP and UDP manage ports and connection state, the network layer where IP addressing and routing happen, and the link layer underneath that I rarely touch directly in a cloud environment since virtual networking abstracts it away."

---

## 2. IP Addressing & CIDR
`CORE`

**Why it exists:** Every device on a network needs a unique address so traffic can be routed to exactly the right place — the same problem a postal system solves with street addresses. CIDR notation exists because early IP address allocation (rigid "classes") was wasteful; CIDR lets you carve a network into precisely the size you need.

**Physical flow — how an address identifies a destination:**
1. An IP address (e.g., `10.0.1.15`) is 32 bits (IPv4), split conceptually into a network portion and a host portion.
2. The CIDR prefix (`/24`) tells you exactly where that split happens — `/24` means the first 24 bits are the fixed network portion, leaving 8 bits (256 addresses) for hosts.
3. Any device needing to send a packet checks: is the destination IP on my own local network (same network portion), or does it need to go through a gateway?
4. This single check — same network or not — is the most basic routing decision every packet makes.

**Components involved:** IP address, subnet mask/CIDR prefix, network address (first address in the range), broadcast address (last address), gateway.

**Exact identifiers:** `10.0.0.0/16` = 65,536 addresses. `10.0.1.0/24` = 256 addresses, a subnet carved from that /16. In cloud VPCs, the first 4 and last 1 address of every subnet are reserved by the provider (AWS reserves 5 total per subnet), so a /24 gives you 251 usable host addresses, not 256.

**Decision point:** Smaller prefix number = bigger network (a common source of confusion) — `/16` has far more addresses than `/24`. Choosing subnet sizes when designing a VPC is a real, recurring exercise: pick a large CIDR for the whole VPC, then carve smaller subnets per availability zone with room to grow.

**Plain example:** A `/16` is like reserving an entire zip code's worth of addresses; a `/24` carved out of it is like reserving just one street within that zip code.

> [!NOTE]
> **Anchor Fact:** `/24` = 256 addresses total, 254 usable on a plain network (minus network + broadcast address). In an AWS VPC subnet specifically, 5 addresses are reserved, leaving 251 usable.

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Carve a VPC-style subnet layout**
> **Concept:** A commonly reported real VPC design exercise (Globant: "designing a secure VPC with private and public subnets").
> **Implementation:** Given `10.0.0.0/16`, carve out: 2 public subnets (`10.0.1.0/24`, `10.0.2.0/24`) across two availability zones, and 2 private subnets (`10.0.11.0/24`, `10.0.12.0/24`) across the same two zones — leaving the rest of the /16 unused for future growth.
> **Hands-on:** Do this on paper first, then verify with an online CIDR calculator. Practice explaining out loud why you left gaps (`10.0.3.0` through `10.0.10.0`) unused rather than numbering subnets consecutively — real designs leave room to insert more subnets later without renumbering everything.
> **Failure:** You run out of address space because you carved subnets too large for a VPC that needs many small, purpose-specific subnets later (e.g., separate subnets per microservice tier).
> **Investigation:** Count how many /24s a /16 actually holds (256) versus how many you've already committed.
> **Fix:** Use smaller subnets (e.g., /26 or /27) for tiers that will never need 251 hosts, preserving address space.
> **Verification:** Confirm no two subnet ranges overlap (a very common real mistake) using a CIDR calculator.
> **Interview Explanation:** "I'd pick a large enough VPC CIDR upfront — a /16 — then carve smaller subnets per AZ and per tier, leaving numbering gaps for future subnets. Getting the top-level CIDR too small is a mistake you can't easily fix later, since it's expensive to change a VPC's CIDR after resources are already deployed in it."

---

## 3. Ports & the Transport Layer: TCP vs UDP
`CORE`

**Why it exists:** A single server needs to run many services at once — a web server, SSH, a database — all reachable at the same IP address. Ports let the kernel route incoming packets to the correct listening process. TCP and UDP exist because different applications need different tradeoffs: guaranteed, ordered delivery versus raw speed with no guarantees.

**Physical flow:**
1. A client wants to reach a service — it needs the destination IP AND a destination port (e.g., `443` for HTTPS).
2. The client's own connection also has a source IP and a source port — usually a random high-numbered "ephemeral" port assigned by the OS.
3. Together, these four values — source IP, source port, destination IP, destination port — form the connection's unique identity, called the **4-tuple**.
4. The protocol (TCP or UDP) determines what happens next: TCP negotiates a connection first (the handshake, Section 5); UDP just sends the packet immediately with no setup and no guarantee of delivery.
5. The receiving kernel uses the destination port to hand the packet to the correct listening process.

**Components involved:** Source/destination IP, source/destination port, protocol number (TCP=6, UDP=17 at the IP header level), the OS's port table.

**Exact identifiers:** Well-known ports worth having memorized: `22` SSH, `53` DNS, `80` HTTP, `443` HTTPS, `3306` MySQL, `5432` PostgreSQL, `6379` Redis, `3389` RDP. Ephemeral (client-side) ports are typically in the `32768–60999` range on Linux.

**Decision point:** TCP vs UDP is a deliberate choice per protocol — DNS actually uses both (UDP for typical small queries, TCP as fallback for large responses or zone transfers), which surprises people who assume DNS is UDP-only.

**Plain example:** A 4-tuple is like a phone call uniquely identified by both parties' phone numbers AND which specific line each is calling from on a multi-line office phone system — the same two office buildings can have many simultaneous calls, each uniquely identified by that combination.

> [!NOTE]
> **Anchor Fact:** The 4-tuple — source IP, source port, destination IP, destination port — uniquely identifies a TCP connection. This is precisely why a server can handle thousands of simultaneous connections on the same destination port (443): each client's combination is different.

> [!WARNING]
> **Failure Scenario:** A service should be listening on port 8443, but a client can't connect. **Symptom:** connection refused or timeout. **Investigation:** `ss -tulnp | grep 8443` on the server — is anything actually listening? If yes, is it bound to `127.0.0.1` (localhost only, unreachable from outside) instead of `0.0.0.0` (all interfaces)? **Root cause:** binding to the wrong interface is one of the most common real "works locally, unreachable remotely" causes. **Fix:** rebind the service to `0.0.0.0` (or the specific interface that should accept external traffic). **Verification:** `ss -tulnp` again, then test from a remote host with `nc -zv <host> 8443`.

> [!TIP]
> **Interview Answer:** "Ports let one IP address host many services simultaneously — the kernel routes incoming packets to the right process by destination port. A connection's real identity is the full 4-tuple: source IP, source port, destination IP, destination port — which is how a server juggles thousands of simultaneous connections to the same port without confusing them."

---

## 4. DNS Resolution
`CORE`

**Why it exists:** Computers route traffic using numeric IP addresses, but humans (and most application configs) work with names like `api.example.com`. DNS translates names to addresses, and does it hierarchically so no single server needs to know every address on the internet.

**Physical flow:**
1. An application asks to resolve `api.example.com`. The OS first checks its local resolver cache and `/etc/hosts` for a static override.
2. If not cached, the query goes to a configured **resolver** (often your ISP's, your company's internal DNS, or a public one like `1.1.1.1`/`8.8.8.8`).
3. The resolver checks its own cache; on a miss, it walks the hierarchy: it asks a **root** server, which points it to the right **TLD** server (`.com`), which points it to the domain's **authoritative nameserver**.
4. The authoritative nameserver holds the actual record and returns the answer — commonly an `A` record (IPv4 address), `AAAA` (IPv6), or `CNAME` (alias to another name).
5. The resolver caches the answer for the record's **TTL** (time-to-live) and returns it to the application, which now has an IP address to actually connect to.

**Components involved:** Resolver, root servers, TLD servers, authoritative nameserver, record types (A, AAAA, CNAME, MX, TXT, NS), TTL.

**Exact identifiers:** DNS uses UDP port 53 for typical queries, falling back to TCP port 53 for responses too large for a single UDP packet (over 512 bytes historically, though EDNS0 raises this) or for zone transfers.

**Decision point:** A `CNAME` cannot coexist with other records at the same name (a real, commonly-hit DNS design constraint) — this is why a domain's root/apex record often uses provider-specific alias mechanisms (like AWS Route 53's "Alias" record) instead of a plain CNAME.

**Plain example:** DNS resolution is like asking a local librarian for a book (your resolver cache) — if they don't know, they check a directory of directories (root/TLD) that tells them exactly which specialized library (the authoritative nameserver) actually holds the answer, rather than one librarian trying to remember every book in the world.

> [!NOTE]
> **Anchor Fact:** DNS primarily uses UDP port 53, falling back to TCP port 53 for large responses or zone transfers. TTL controls how long a resolver caches an answer before re-querying — a low TTL means faster propagation of changes, at the cost of more repeated lookups.

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — The DNS-first diagnostic habit**
> **Concept:** Multiple current sources independently emphasize the same sequence for "the site is down": resolve name first, then reachability, then port, then application layer — because a DNS failure, a routing failure, and an application failure produce different symptoms that look identical from a user's error message alone.
> **Implementation:**
> ```bash
> dig +short api.example.com          # does the name even resolve?
> dig +trace api.example.com          # walk the full hierarchy if resolution itself looks wrong
> dig @1.1.1.1 api.example.com        # compare against a known-good public resolver
> ```
> **Hands-on:** Compare `dig @<your-configured-resolver>` against `dig @1.1.1.1` for a real domain — if they disagree, your configured resolver is the problem, not the domain.
> **Failure:** An application reports "connection failed" and the on-call engineer immediately starts checking firewall rules.
> **Investigation:** `dig` the hostname first — if it doesn't resolve at all, no firewall rule was ever going to matter; the request never had anywhere to go.
> **Fix:** Address DNS (stale record, wrong resolver config, expired TTL causing a fallback) before touching anything downstream.
> **Verification:** `dig +short` returns the expected IP, then proceed to the reachability/port checks in later sections.
> **Interview Explanation:** "Before touching firewalls or the application, I resolve the name first — a DNS failure and a network failure produce different error messages, and skipping this step wastes time chasing the wrong layer."
## 5. The TCP Handshake & Connection Lifecycle
`CORE`

**Why it exists:** Before two machines exchange real data reliably, they need to agree that both sides are actually there and ready — otherwise data could be sent into a void with no way to know it arrived. The TCP handshake establishes this shared state before any application data flows.

**Physical flow:**
1. Client sends a `SYN` packet to the server's IP and port — "I want to connect, here's my starting sequence number."
2. Server responds with `SYN-ACK` — "I acknowledge your request, here's mine."
3. Client responds with `ACK` — connection is now established (the "three-way handshake").
4. Application data flows in both directions, each side acknowledging received data.
5. Closing is its own negotiated sequence: `FIN` from one side, `ACK`, `FIN` from the other side, final `ACK` — after which the closing side sits briefly in `TIME_WAIT` to catch any stray delayed packets.

**Components involved:** SYN/SYN-ACK/ACK flags, sequence numbers, the connection state machine (`LISTEN`, `SYN_SENT`, `ESTABLISHED`, `FIN_WAIT`, `TIME_WAIT`, `CLOSED`).

**Exact identifiers:** `ss -tan` shows connection states directly. A server under load with thousands of connections stuck in `TIME_WAIT` is a real, known operational pattern (not usually dangerous, but worth recognizing).

**Decision point:** A half-open connection (client sent SYN, server never responded) behaves very differently from a connection actively refused — a refusal gets an immediate `RST` (reset) response; a silent drop (common with security groups/firewalls) gets no response at all, and the client just waits until it times out. This distinction is one of the most useful diagnostic signals in networking.

**Plain example:** The handshake is like confirming a phone call connected before either person starts talking — "can you hear me?" / "yes, can you hear me?" / "yes" — only after that three-step confirmation does the actual conversation (data) begin.

> [!NOTE]
> **Anchor Fact:** An actively refused connection gets an immediate `RST` — this is the host's own OS/TCP stack responding on behalf of a port nothing is listening on (or a listening service that's actively closing you out); either way, it proves the HOST is reachable, just not that port. A silently dropped connection (typically a firewall/security-group rule with no explicit reject) gets no response at all and simply times out. These look identical to an impatient user, but mean completely different things to you as the engineer.

> [!WARNING]
> **Failure Scenario:** `curl` to a service hangs for a long time with zero output, not even an error. **Symptom:** total silence, then eventual timeout. **Hypothesis:** packets are being silently dropped somewhere in the path — a security group, a NACL, or an overly aggressive firewall rule with no reject response configured. **Investigation:** `tcpdump` on both the client and server ends of the connection shows exactly where the SYN packet stops appearing. **Root cause:** confirmed by checking security group/NACL/firewall rules for that exact port and source. **Fix:** add the missing allow rule. **Verification:** re-run the `curl`, confirm the handshake completes.

> [!TIP]
> **Interview Answer:** "A refused connection and a hanging connection tell you different things — refused means the host itself is reachable and its TCP stack actively sent back an RST, typically because nothing's listening on that specific port. Hanging with no response at all usually means a packet is being silently dropped somewhere in the path — a security group or firewall rule — and tcpdump on both ends is how I'd find exactly where it stops."

---

## 6. Routing & Gateways
`CORE`

**Why it exists:** Once a packet knows it's not on its own local network (Section 2), something needs to decide where to send it next. Routing is that decision — a chain of "who do I hand this off to" choices made independently by every device along the path.

**Physical flow:**
1. A device checks its own route table for the destination IP — the most specific matching route wins.
2. If the destination isn't on the local network, the packet goes to the **default gateway** — the device's "send it here if I don't know a better route" address.
3. That gateway (a router) checks its own route table and forwards the packet toward the next hop.
4. This repeats, hop by hop, until the packet reaches a router that knows the destination's local network directly, and delivers it.
5. In a cloud VPC, this same concept exists as a **route table** attached to each subnet, directing traffic to an Internet Gateway, a NAT Gateway, a peering connection, or another subnet.

**Components involved:** Route table, default gateway, next hop, Internet Gateway (IGW), TTL (used to prevent infinite routing loops — decremented at each hop, packet dropped if it hits zero).

**Decision point:** In AWS specifically, a subnet is "public" or "private" purely because of what its route table's default route points to — an Internet Gateway (public) or a NAT Gateway/nothing (private). It is not a special flag on the subnet itself — a very commonly misunderstood point (and one directly confirmed as an interview topic: Globant's "designing a secure VPC with private and public subnets").

**Plain example:** Routing is like asking for directions at every intersection rather than knowing the whole route upfront — each intersection (router) only needs to know which direction gets you closer, not the entire path to your final destination.

> [!TIP]
> **Interview Answer:** "Routing is a series of independent, local decisions — each router just needs to know which next hop gets a packet closer to its destination, not the full path. In a VPC, this is literally the route table attached to a subnet — a subnet is 'public' purely because its route table sends internet-bound traffic to an Internet Gateway, nothing more special than that."

---

## 7. NAT
`CORE`

**Why it exists:** Private IP address space (like `10.0.0.0/8`) isn't routable on the public internet, and there aren't enough public IPv4 addresses for every device to have one anyway. NAT (Network Address Translation) lets many devices with private addresses share a smaller number of public addresses by rewriting packet headers in transit.

**Physical flow:**
1. A private-subnet resource (e.g., `10.0.11.5`) sends a request to a public address.
2. Its route table sends outbound traffic to a NAT Gateway, which itself sits in a public subnet with its own public IP.
3. The NAT Gateway rewrites the packet's source IP (and source port) to its own public IP before sending it onward — the destination sees the request as coming from the NAT Gateway, not the original private IP.
4. The NAT Gateway keeps a translation table mapping "this public-facing port belongs to that private IP" so the response can be routed back correctly.
5. Critically, this table only allows return traffic for connections that were *initiated from inside* — nothing on the internet can start a new connection in through the NAT.

**Components involved:** NAT Gateway (or NAT instance), the translation table, public vs private IP space.

**Decision point:** This is exactly why a private subnet resource can reach out to the internet (pull a package, call an external API) but nothing external can reach in to it directly — asymmetric by design, not a limitation to work around.

**Plain example:** NAT is like an office's single reception phone number — many employees inside can call out and their caller ID shows the shared office number, but nobody outside can dial a specific employee's desk directly without going through reception first, and reception only connects calls that are *responses* to a call an employee already placed.

> [!TIP]
> **Interview Answer:** "NAT lets many private-IP resources share a smaller number of public IPs by rewriting the source address on the way out and keeping a translation table to route responses back correctly. It's inherently one-directional in effect — outbound-initiated traffic works, but nothing external can initiate a new inbound connection through it, which is exactly the security property that makes private subnets useful."

---

## 8. Firewalls, Security Groups & NACLs
`CORE`

**Why it exists:** Not all traffic that *can* reach a device *should* be allowed to. A firewall is a rule-based checkpoint the kernel or cloud provider evaluates against every packet — allow or deny based on source, destination, port, and protocol.

**Physical flow:**
1. A packet arrives at a boundary (a host's kernel firewall, or in AWS, a security group and/or NACL).
2. The rule set is evaluated: does this packet's source, destination, port, and protocol match an allow rule?
3. **Security groups** (AWS, instance-level) are **stateful** — if you allow inbound traffic on a port, the return traffic is automatically permitted, no separate outbound rule needed.
4. **NACLs** (AWS, subnet-level) are **stateless** — you must explicitly allow both the inbound request AND the outbound response (typically on high, ephemeral ports), and rules are evaluated in numbered order, first match wins.
5. A packet that matches no allow rule is dropped — usually silently in cloud environments (no RST, no ICMP unreachable), which is exactly why a firewall-blocked connection hangs rather than fails fast (Section 5).

**Components involved:** Security group, NACL, rule direction (inbound/outbound), stateful vs stateless evaluation.

**Decision point:** People commonly conflate "traffic reaches the instance" with "traffic is allowed" — three independent conditions must ALL hold for external traffic to reach a resource: the subnet has a route to get there, the resource has a reachable IP, and the security group/NACL both permit it. Missing any one looks identical from the outside (unreachable).

**Plain example:** A security group is like a guest list checked once at the door, and once you're in, you're free to have a full conversation without re-checking every sentence. A NACL is like a checkpoint that inspects every single sentence going both in AND out, with no memory of what was said before.

> [!NOTE]
> **Anchor Fact:** Security groups = stateful, instance-level, only need inbound rules for inbound traffic (return traffic is automatic). NACLs = stateless, subnet-level, ordered rule evaluation, need explicit inbound AND outbound rules.

> [!IMPORTANT]
> 🟣 **LIVE-DIAGNOSIS TASK — "Can you deny traffic using a security group?"**
> A real, verbatim-reported interview question (PwC, Bengaluru). The precise, correct answer matters: AWS security groups only support **allow** rules — there is no explicit "deny" rule type. To block specific traffic, you either don't add an allow rule for it (default-deny is implicit), or you use a NACL, which does support explicit deny rules because it's stateless and rule-ordered. Getting this distinction right, unprompted, signals real hands-on experience rather than memorized theory.

> [!TIP]
> **Interview Answer:** "Security groups are stateful and instance-level — allow-only, no explicit deny, and return traffic is automatically permitted. NACLs are stateless and subnet-level, evaluated in rule order, and do support explicit deny rules, but need both inbound and outbound rules stated separately since they don't track connection state."

---

## 9. HTTP/HTTPS & TLS
`CORE`

**Why it exists:** Once a TCP connection exists, applications need an agreed format for requests and responses — that's HTTP. HTTPS adds TLS encryption on top so the conversation can't be read or tampered with by anything in between.

**Physical flow:**
1. After the TCP handshake (Section 5), for HTTPS, a **TLS handshake** happens first: the client and server negotiate a shared encryption key and the server presents a certificate proving its identity.
2. Only after that succeeds does the actual HTTP request get sent — method (GET/POST/etc.), path, headers, optionally a body — all now encrypted.
3. The server processes the request and returns a response: a status code (`200`, `404`, `500`...), headers, and a body.
4. The connection may be kept alive for further requests (HTTP/1.1 keep-alive, or multiplexed in HTTP/2) rather than renegotiating TLS every time.

**Components involved:** TLS certificate, cipher negotiation, HTTP methods, status codes, headers.

**Exact identifiers:** `curl -v` shows every stage explicitly: DNS resolution, TCP connect, TLS handshake (cert details), request headers sent, response headers received, body.

**Decision point:** A `curl` failing at the TLS stage (certificate error) is a completely different problem from one failing at the HTTP stage (a 500 error) — `curl -v` output tells you exactly which stage failed, so read the whole output, not just the final line.

**Plain example:** HTTP is the shared language of the conversation (what to ask, how to answer). TLS is a sealed, tamper-evident envelope wrapped around that conversation so nobody along the path can read or alter it without both ends noticing.

> [!TIP]
> **Interview Answer:** "curl -v is my first tool for any HTTP-layer issue because it shows every stage in order — DNS, TCP connect, TLS handshake, request, response — and where it fails tells me immediately whether this is a certificate problem, a network problem, or an application problem returning a bad status code."

---

## 10. Load Balancers
`CORE`

**Why it exists:** A single server has a ceiling on how much traffic it can handle, and is a single point of failure. A load balancer distributes incoming traffic across multiple backend servers, and stops sending traffic to any backend that's unhealthy.

**Physical flow:**
1. A client connects to the load balancer's IP/domain, not directly to any backend server.
2. The load balancer terminates the incoming connection (and for HTTPS, often the TLS as well).
3. Based on its configuration (round-robin, least-connections, or routing rules), it picks a healthy backend and forwards the request.
4. It continuously runs **health checks** against each backend — if one fails enough checks, it's removed from rotation until it passes again.
5. The response flows back through the load balancer to the client.

**Components involved:** Listener (the port/protocol the LB accepts traffic on), target group (the backends), health check configuration.

**Decision point — L4 vs L7, a frequently tested distinction:** A Layer 4 load balancer (like AWS's NLB) works purely at the TCP/UDP level — it doesn't understand HTTP at all, just forwards packets based on IP/port, extremely fast, preserves the original client IP naturally. A Layer 7 load balancer (like AWS's ALB, or an Nginx reverse proxy) understands HTTP — it can route based on URL path or hostname, inspect headers, and terminate TLS — at the cost of more processing overhead per request.

**Plain example:** An L4 load balancer is like a mail sorting facility that only reads the outer envelope address and forwards accordingly, never opening it. An L7 load balancer actually opens the envelope, reads the letter's content, and can route differently based on what it says inside — more capable, but slower per piece of mail.

> [!NOTE]
> **Anchor Fact:** ALB = Layer 7 (HTTP-aware, path/host routing). NLB = Layer 4 (raw TCP/UDP, extreme throughput, preserves source IP naturally). This exact comparison ("Diff ELB and NLB") was directly reported as an asked interview question (PwC, Bengaluru).

> [!WARNING]
> **Failure Scenario:** A load balancer shows all backend targets as "unhealthy" even though the application is clearly running when you check it directly on the server. **Symptom:** healthy app, unhealthy target group. **Investigation:** check the health check's configured path and port — does it actually match what the app serves (not just any endpoint, but the SPECIFIC path/port configured)? Check the security group on the backend — does it allow inbound traffic from the load balancer's security group specifically, not just "the internet"? **Root cause:** commonly a health check path typo, or a security group that allows the public internet but not the load balancer's own subnet/security group. **Fix:** correct the health check config or security group rule. **Verification:** target group shows healthy within one or two check intervals.

> [!TIP]
> **Interview Answer:** "ALB operates at Layer 7 — it understands HTTP, so it can route by path or host and terminate TLS. NLB operates at Layer 4 — pure TCP/UDP forwarding, no HTTP awareness, but extremely fast and it naturally preserves the client's source IP. I'd pick NLB for raw throughput or non-HTTP protocols, ALB when I need content-based routing."
## 11. VPC Endpoints
`CORE`

**Why it exists:** Without one, a private-subnet resource calling an AWS service like S3 has to route out through a NAT Gateway to the public AWS endpoint and back — extra cost, extra latency, and the traffic technically leaves your private network boundary even though both ends are AWS. A VPC endpoint creates a private path directly to the AWS service instead.

**Physical flow — Interface endpoint (the more common case for most services):**
1. You create an interface endpoint for a specific service (e.g., ECR) in a specific subnet.
2. AWS provisions an ENI (a network interface) with a private IP address inside that subnet.
3. Requests to that AWS service, resolved via AWS's private DNS for the endpoint, now go to that private IP instead of a public one.
4. A dedicated security group on the endpoint's ENI controls exactly what can use it.

**Physical flow — Gateway endpoint (only for S3 and DynamoDB):**
1. Instead of an ENI, AWS adds an entry directly to the subnet's route table pointing at the S3/DynamoDB service.
2. No ENI, no security group on the endpoint itself, and it's free — cheaper and simpler, but only available for these two services.

**Decision point:** Gateway endpoints (S3, DynamoDB) are a route-table entry — free, simple, no ENI. Interface endpoints (almost everything else — ECR, STS, Secrets Manager, SSM, etc.) create a billable ENI with its own security group, using AWS PrivateLink underneath.

> [!NOTE]
> **Anchor Fact:** Only S3 and DynamoDB support Gateway endpoints (route-table based, free). Everything else needing a VPC endpoint uses an Interface endpoint (ENI-based, billed hourly plus data processing).

> [!WARNING]
> **Failure Scenario:** A private-subnet resource (e.g., an EKS worker node with no NAT Gateway) fails to pull an image from ECR. **Investigation:** confirm whether a route to the internet exists at all (NAT Gateway) — if not, an ECR interface endpoint is required instead. **Root cause:** commonly, the endpoint exists but its security group doesn't allow inbound HTTPS (443) from the resource trying to use it, or the endpoint's private DNS isn't enabled, so the service name still resolves to a public IP instead of the endpoint's private one. **Fix:** correct the endpoint's security group, and confirm "Enable DNS name" is checked on the endpoint. **Verification:** `dig` the service's hostname from inside the private subnet — it should now resolve to a private IP in your VPC's CIDR range.

> [!TIP]
> **Interview Answer:** "A VPC endpoint gives a private-subnet resource a path to an AWS service without going through a NAT Gateway or the public internet. S3 and DynamoDB use a free, route-table-based Gateway endpoint; everything else — ECR, STS, Secrets Manager — uses an Interface endpoint, which provisions an ENI with its own security group using PrivateLink. This is directly why private EKS nodes can still pull images from ECR with no internet access at all, as long as the right interface endpoints exist."

---

## 12. Connecting VPCs and On-Premises: Peering, Transit Gateway, VPN, Direct Connect
`CORE`

**Why it exists:** Real organizations rarely live in one isolated VPC — you need VPCs to talk to each other, and often an on-premises data center needs to reach cloud resources (and vice versa) as if directly connected. Multiple real, current interview reports (Genpact Bengaluru specifically) confirm this is actively tested, not theoretical.

**The four mechanisms, compared:**

| Mechanism | Connects | How | Key limitation |
|---|---|---|---|
| VPC Peering | Two VPCs | Direct, private network connection, as if on the same network | Non-transitive — if A peers with B and B peers with C, A cannot reach C through B |
| Transit Gateway (TGW) | Many VPCs (and on-prem) | A central hub every VPC connects to once | Solves peering's non-transitivity at the cost of one more component to manage |
| Site-to-Site VPN | On-prem ↔ VPC | An encrypted tunnel over the public internet | Bandwidth/latency bound by the public internet path |
| Direct Connect | On-prem ↔ AWS | A dedicated, private physical network link | Higher cost and setup lead time, but consistent bandwidth/latency, not subject to public internet variability |

**Decision point:** VPC Peering's non-transitivity is the single most commonly misunderstood point here — connecting many VPCs with peering alone means a full mesh of individual connections, which becomes unmanageable past a handful of VPCs. Transit Gateway exists specifically to solve that at scale.

**Plain example:** Peering is like running a private phone line directly between two specific offices — great for two, unmanageable for twenty. A Transit Gateway is like routing every office through one central switchboard instead — add a new office, connect it once to the switchboard, and it can already reach everyone else through it.

> [!NOTE]
> **Anchor Fact:** VPC Peering is non-transitive. Worth memorizing verbatim — it's a frequently tested, easily-confused detail about peering specifically.

> [!TIP]
> **Interview Answer:** "VPC Peering directly connects two VPCs privately, but it's non-transitive — peering doesn't chain, so connecting many VPCs this way becomes a full mesh fast. Transit Gateway solves that by acting as a central hub every VPC connects to once. For on-premises connectivity, Site-to-Site VPN is an encrypted tunnel over the public internet — quick to set up, but bandwidth depends on the internet path — while Direct Connect is a dedicated physical link with consistent performance, at higher cost and longer setup time."
---

# Part 2 — Deeper Mechanics

## The Full Request Journey, Packet by Packet
`CORE`

**Why this section exists:** Every concept in Part 1 is a piece. This section is the assembled machine — tracing one real request end to end, the way an interviewer wants to hear it walked through, and the way you should be able to picture it in your head during any troubleshooting session.

**The full flow — a browser requesting `https://api.example.com/users`:**
1. **DNS resolution:** OS checks cache/`/etc/hosts` → queries configured resolver → resolver walks root → TLD → authoritative nameserver (or hits its own cache) → returns an IP, e.g., `203.0.113.10`.
2. **Routing decision (client side):** the OS checks: is `203.0.113.10` on my local network? No → send to default gateway.
3. **TCP handshake:** client picks an ephemeral source port (e.g., `54321`) → SYN to `203.0.113.10:443` → SYN-ACK back → ACK → connection established. The 4-tuple for this connection is now `(client_IP, 54321, 203.0.113.10, 443)`.
4. **Routing en route:** the packet passes through the client's gateway, possibly a NAT Gateway (if the client is in a private subnet), across the internet through multiple ISP/backbone hops (each decrementing TTL by one), until it reaches a router that knows how to deliver to `203.0.113.10` directly.
5. **Arrival at the destination boundary:** a security group and/or NACL evaluate the incoming SYN packet against their rules — permitted, so it proceeds (if it weren't, this is exactly where a silent drop would happen).
6. **Load balancer:** in a real production setup, `203.0.113.10` is actually the load balancer's IP — it terminates the TCP connection (and TLS) itself, picks a healthy backend from its target group, and opens its OWN separate TCP connection to that backend (a different 4-tuple entirely).
7. **TLS handshake:** certificate presented, encryption negotiated — now the connection is secure.
8. **HTTP request sent:** `GET /users HTTP/1.1`, headers, over the now-encrypted connection.
9. **Backend processes the request:** the backend's kernel receives it on whatever port it's listening on, hands it to the application process via that process's file descriptor for the socket.
10. **Response path:** application generates a response → backend → load balancer → back across the same general path → client's browser renders it.

> [!TIP]
> **Interview Answer (the one worth having word-perfect):** "A request starts with DNS resolving the hostname to an IP. The client checks its routing table, sends the packet toward its gateway, and a TCP handshake establishes a connection using a 4-tuple of source/destination IP and port. Along the way, any NAT, firewall, or security group either permits or silently drops it — silent drops are why a blocked connection hangs instead of failing fast. At the destination, if there's a load balancer, it terminates that connection and opens a completely separate one to a healthy backend. Once connected, TLS negotiates encryption, then the actual HTTP request and response travel over that secured channel."

---

## Sockets and the ss Model
`CORE`

**Why it exists:** A process needs an addressable endpoint to send/receive network data — a socket is that endpoint, and the OS needs a way to track every one that currently exists so you (and the kernel) can answer "what's listening, what's connected, to whom."

**Physical flow:** A listening socket (`LISTEN` state) waits for incoming connections on a specific port. Each new incoming connection that completes the handshake becomes its own separate `ESTABLISHED` socket, with its own unique 4-tuple, while the original listening socket keeps waiting for the next one.

**Exact identifiers:**
```bash
ss -tulnp        # t=tcp, u=udp, l=listening only, n=numeric ports, p=owning process
ss -tan           # all TCP connections with their current state
ss -tan state established           # only established connections
ss -tan state time-wait | wc -l     # count of connections cooling down after close
```

> [!NOTE]
> **Anchor Fact:** One listening socket can spawn thousands of simultaneous `ESTABLISHED` sockets on the same port — each is a distinct 4-tuple. `ss -tan state established | wc -l` is a fast way to see real concurrent load on a server.

> [!TIP]
> **Interview Answer:** "ss -tulnp tells me what's actually listening and which process owns it — that's my first command whenever something 'should' be reachable but isn't. ss -tan with a state filter shows me the live connection picture, which is useful for spotting things like a TIME_WAIT pileup under high connection churn."

---

## How traceroute/mtr Actually Work
`CORE`

**Why it exists:** When a connection fails somewhere in the middle of a multi-hop path, you need a way to see WHICH hop is the problem, not just that "the internet" isn't working.

**Physical flow:**
1. `traceroute` sends packets toward the destination with a deliberately low **TTL** (Time To Live) — starting at 1.
2. Each router that forwards a packet decrements its TTL by 1; when TTL hits 0, that router drops the packet and sends back an ICMP "Time Exceeded" message, revealing itself.
3. `traceroute` repeats this with TTL=1, then 2, then 3... — each round reveals one more hop along the path, building the full route one router at a time.
4. `mtr` does the same thing but continuously, showing per-hop packet loss and latency over time — a real, meaningful upgrade over a one-shot `traceroute`, since a single traceroute can look worse (or better) than reality due to one bad sample.

**Decision point:** Many ISPs and cloud networks rate-limit or deprioritize ICMP specifically — a hop showing `* * *` in traceroute doesn't necessarily mean that hop is down, it might just be deprioritizing/dropping the ICMP probe itself while still forwarding real traffic fine. `mtr -T` (TCP mode, probing an actual port) sidesteps this ICMP-specific deprioritization.

> [!NOTE]
> **Anchor Fact:** A `* * *` in the middle of a traceroute does NOT reliably mean that hop is broken — many networks deprioritize ICMP specifically while forwarding real traffic normally. Only persistent loss continuing to the destination, or loss confirmed with `mtr -T` against the actual port/protocol in use, is a reliable signal.

> [!TIP]
> **Interview Answer:** "traceroute works by sending packets with increasing TTL and reading the ICMP Time Exceeded response each router sends back as its TTL hits zero — that's how it reveals one hop per round. mtr does the same thing continuously and shows per-hop loss over time, which is more reliable than a single traceroute snapshot, and I'd use mtr -T in TCP mode specifically when ICMP looks broken but the actual service doesn't, since a lot of networks deprioritize ICMP without actually blocking real traffic."

---

## MTU & Fragmentation
`STRETCH`

**Why it exists:** Every network link has a maximum packet size it can carry (the MTU — commonly 1500 bytes on Ethernet). A packet larger than the path's smallest MTU either gets fragmented into smaller pieces or, if fragmentation is disallowed, dropped outright — a genuinely tricky class of failure because it can affect only large packets while small ones (like a basic ping) work fine.

**Failure signature:** A connection establishes fine, small requests work, but anything with a larger payload hangs or fails — classic MTU mismatch, often seen with VPNs or tunneled connections (like a VPN or overlay network) where the effective MTU is smaller than the standard 1500 due to added tunnel headers.

> [!NOTE]
> **Anchor Fact:** Standard Ethernet MTU is 1500 bytes. VPNs/tunnels typically reduce the EFFECTIVE MTU because of added header overhead — a common, sneaky cause of "small requests work, large ones hang" failures.

> [!TIP]
> **Interview Answer:** "If small requests succeed but larger ones hang specifically over a VPN or tunneled connection, I'd suspect an MTU mismatch before anything else — the tunnel's overhead reduces the effective MTU, and packets that don't fit either fragment badly or get silently dropped if a device in the path has fragmentation disabled."

---

## CGNAT & Public vs Private IP Space
`STRETCH`

**Why it exists:** IPv4 address exhaustion means many ISPs can no longer give every customer a true public IP — Carrier-Grade NAT (CGNAT) shares a single public IP across many customers, one more NAT layer beyond your own home router.

**Recognizing it:** `curl -4 ifconfig.me` shows your apparent public IP — if it falls in the `100.64.0.0/10` range, you're behind CGNAT, and inbound connections to your home network genuinely will not work no matter how you configure your own router, because the address your router has isn't actually publicly routable at all.

> [!NOTE]
> **Anchor Fact:** `100.64.0.0/10` is the reserved range specifically for CGNAT — seeing this as your "public" IP means an ISP-level NAT layer exists beyond your control, not a misconfiguration on your end.
---

## ARP & ICMP: The Control-Plane Basics Underneath IP
`CORE`

**Why they exist:** IP routing decisions (Section 6) tell a packet WHERE to go logically, but on a local network segment, delivery still has to happen at the hardware (MAC address) level — ARP bridges that gap. ICMP exists as IP's own error-reporting and diagnostic protocol — it's not for carrying application data at all, purely for the network to talk about itself.

**ARP — physical flow:**
1. A device knows the destination IP is on its own local network (Section 2's same-network check), but to actually put a frame on the wire, it needs the destination's MAC (hardware) address.
2. It broadcasts an ARP request to the whole local network: "who has this IP?"
3. The device owning that IP replies directly with its MAC address.
4. The requester caches this mapping (the ARP cache/table) briefly, avoiding repeating this for every packet.

**ICMP — what it's actually for:**
- `ping` uses ICMP Echo Request/Reply — purely a reachability/latency check, carrying no application data.
- Routers send ICMP "Time Exceeded" when a TTL hits zero — this is literally how traceroute works (see the Deeper Mechanics section above).
- Routers/hosts send ICMP "Destination Unreachable" when they know a packet can't be delivered — this is a genuinely different, more informative signal than a security-group-style silent drop.

**Decision point:** Many production security groups and firewalls block ICMP by design (often for perceived security reasons) — meaning `ping` failing tells you nothing conclusive about whether the actual service (on its real port, over TCP) is reachable. Never conclude "the server is down" from a failed ping alone; confirm with a port-level or application-level check.

> [!NOTE]
> **Anchor Fact:** A failed `ping` proves ICMP is blocked or the host is down — it does NOT prove the actual service is unreachable, since ICMP is very commonly blocked independently of real application traffic. Always confirm with a port/application-level check (`nc -zv`, `curl`) before concluding a host is down based on ping alone.

> [!TIP]
> **Interview Answer:** "ARP resolves an IP to a MAC address for delivery on the local network segment — it's why you need a broadcast mechanism even after routing has already decided a packet's logical next hop. ICMP is IP's own diagnostic protocol — ping and traceroute both rely on it — but a lot of production environments block ICMP deliberately, so a failed ping alone never proves a service is actually down. I'd always follow up with a port-level check before concluding that."

---

## Bridge to Container Networking (Preview — Full Depth in Later Documents)
`CORE`

**Why this belongs here:** Everything above is what Docker and Kubernetes networking is actually built on top of — this is a short forward-reference, not a substitute for the dedicated Docker and Kubernetes documents coming next in this series.

- A Docker container on a user-defined bridge network gets its own network namespace and virtual interface — the same NAT and routing concepts from Sections 6–7 apply, just virtualized on a single host instead of across physical routers.
- Kubernetes' `kube-proxy` implements the same idea as a load balancer's target-group health-based routing (Section 10), but for Services routing traffic to healthy Pods instead of instances behind an ALB/NLB.
- A Kubernetes NetworkPolicy is conceptually a security group (Section 8) applied at the Pod level instead of the instance level — same allow-list mental model, different enforcement point.

> [!TIP]
> **Interview Answer (if asked to connect the dots early):** "Container networking isn't a different set of concepts — it's the same IP, port, NAT, and routing ideas from traditional networking, just virtualized inside one host or one cluster instead of spanning physical routers. A Kubernetes Service doing endpoint-based routing to healthy Pods is conceptually the same problem a load balancer's health checks solve for EC2 instances."
---

# Part 3 — Interview Corner

Four full 9-angle drills on the topics research shows get pushed hardest, then a quick-coverage round for everything else.

## DNS & Resolution Drill

1. **Basic:** What does DNS actually do? — Translates human-readable names into the IP addresses computers use to route traffic, hierarchically, so no single server needs to know every address on the internet.
2. **Why:** Why is DNS hierarchical instead of one giant lookup table? — Scale — no single server could hold or serve every domain's records for the entire internet with acceptable performance; delegation lets each level (root, TLD, authoritative) own a manageable slice.
3. **How:** Walk through what happens when you run `dig api.example.com`. — Resolver checks cache → if miss, queries root → root refers to the `.com` TLD server → TLD refers to the domain's authoritative nameserver → authoritative server returns the actual record → resolver caches it for the TTL and returns it to you.
4. **Scenario:** A DNS record was just updated, but some users still see the old IP. — TTL caching — resolvers that already cached the old record won't re-query until that TTL expires, so propagation is only as fast as the TTL you set (or previously had set, if it hasn't expired since the last change).
5. **Troubleshooting:** `dig` from your laptop returns the correct IP, but the application server still can't resolve the same name. — Check what resolver the SERVER is actually configured to use (`/etc/resolv.conf`), not your laptop's — they may be pointed at entirely different resolvers, especially in a VPC with a custom DNS setup.
6. **Architecture:** How would you design DNS for a multi-region application needing fast failover? — Low TTLs on the records that need to fail over quickly, combined with health-check-aware DNS routing (like Route 53 health checks) so a failed region's records stop being returned automatically — accepting the tradeoff that very low TTLs increase query volume/cost.
7. **Comparison:** `A` record vs `CNAME` vs Route 53 "Alias" — when does it matter? — `A` points directly to an IP. `CNAME` points to another name (which then resolves further) but cannot coexist with other records at the same name — a real constraint that blocks using a CNAME at a domain's bare apex. An "Alias" record (AWS-specific) behaves like a CNAME but is resolved internally by Route 53 and CAN be used at the apex, specifically solving that limitation.
8. **Follow-up:** Why does DNS use UDP by default instead of TCP? — Speed — most queries and responses are small and a full TCP handshake would be unnecessary overhead for a single small request; TCP is the fallback specifically for responses too large for one UDP packet, or for zone transfers between nameservers.
9. **Pressure:** Production traffic to your service suddenly drops to zero, but the service itself is healthy and reachable by IP. What's your first suspicion, and how do you confirm it fast? — DNS — resolve the hostname from multiple vantage points (`dig @1.1.1.1`, `dig` from inside the affected network) to see if it's returning wrong/no records; a healthy service unreachable by name but fine by IP is one of the most textbook DNS-failure signatures there is.

---

## TCP, Ports & Connections Drill

1. **Basic:** What's the 4-tuple, and why does it matter? — Source IP, source port, destination IP, destination port — together they uniquely identify one TCP connection, which is how a server serves many simultaneous clients on the same destination port.
2. **Why:** Why does TCP need a handshake before sending real data at all? — Both sides need to confirm the other is actually there and agree on starting sequence numbers before reliable, ordered delivery is even possible — sending data blind with no acknowledgment mechanism would make loss detection and retransmission impossible to reason about.
3. **How:** Walk through exactly what `ss -tan` state `LISTEN` vs `ESTABLISHED` vs `TIME_WAIT` mean. — `LISTEN`: a socket waiting for incoming connections, not itself a connection. `ESTABLISHED`: an active, fully-handshaken connection currently able to send/receive data. `TIME_WAIT`: a connection that's been closed by this side but is held briefly to absorb any delayed/duplicate packets before the 4-tuple is fully released for reuse.
4. **Scenario:** A load test shows connection failures under high concurrency, and `ss` shows thousands of sockets in `TIME_WAIT`. — Rapid connection churn (opening/closing many short-lived connections) can exhaust the ephemeral port range faster than `TIME_WAIT` sockets are released, especially if all connections share the same destination IP/port — the practical fixes are connection reuse/keep-alive on the client side, or tuning kernel TIME_WAIT reuse settings server-side.
5. **Troubleshooting:** `curl` to a port returns "Connection refused" instantly. What does that tell you, specifically? — Instantly and specifically means the host is reachable and its own TCP stack actively sent back an RST — typically because nothing is bound to that exact port, though a listening service can also actively reset a connection itself. Either way, this is meaningfully different information from a hang (Section 5's silent-drop distinction) — don't go check firewall rules first here; check whether the service is actually running and bound correctly.
6. **Architecture:** How would you design a service to handle a sudden 10x spike in concurrent connections without falling over? — Connection pooling/reuse rather than one-connection-per-request where possible, tuning the backlog queue size for listening sockets, horizontal scaling behind a load balancer so no single backend absorbs the full spike, and monitoring for ephemeral port exhaustion specifically under high outbound connection churn.
7. **Comparison:** TCP vs UDP — beyond "reliable vs unreliable," what's the real engineering tradeoff? — TCP's reliability (retransmission, ordering, flow control) comes with overhead and head-of-line blocking — one lost packet can stall delivery of everything behind it until it's retransmitted. UDP has no such guarantee but also no such stall — which is exactly why latency-sensitive, loss-tolerant traffic (video, voice, some gaming, and DNS's small queries) prefers it.
8. **Follow-up:** What's a SYN flood, at a level you could explain the defense for? — An attacker sends many SYN packets without ever completing the handshake, exhausting the server's table of half-open connections. Defenses include SYN cookies (not committing real state until the handshake actually completes) and rate-limiting at the network edge.
9. **Pressure:** A service works fine under normal load but starts refusing new connections only during traffic spikes, with no CPU or memory pressure visible. What do you check? — The connection backlog queue (`listen()` backlog size) and the OS's ephemeral port range/reuse settings — a service can be starved for available connection slots or ports well before CPU/memory become the bottleneck, and this is a genuinely different failure class than a resource-exhaustion problem.

---

## Firewalls & Security Groups Drill

1. **Basic:** What's the fundamental difference between a security group and a NACL? — Security groups are stateful and instance-level (return traffic auto-permitted); NACLs are stateless and subnet-level (inbound and outbound must both be explicitly allowed), evaluated in numbered rule order.
2. **Why:** Why does AWS give you both security groups AND NACLs instead of just one? — Defense in depth, and different granularities — security groups let you scope rules tightly per-resource/role, while NACLs give a coarser subnet-wide safety net that doesn't depend on every instance's security group being configured correctly.
3. **How:** How does a stateless NACL actually "remember" to allow a response, if it has no connection state? — It doesn't — you configure a separate explicit outbound rule permitting the response, typically for the ephemeral port range (`1024–65535`), which is a static allowance rather than genuine per-connection tracking.
4. **Scenario:** An application in a private subnet can reach an internal database but a newly-added replica in a different subnet is unreachable. — Compare the security group rules and NACLs on the NEW subnet against the working one — a very common cause is the new subnet's NACL or the replica's security group not yet including the application's security group/subnet as an allowed source, even though the "same" application code is unchanged.
5. **Troubleshooting:** `Can you deny traffic using a security group?` — No — security groups are allow-only; there's no explicit deny rule type. To actively block specific traffic you'd use a NACL (which does support explicit deny, being stateless and rule-ordered) or simply not include an allow rule (implicit default-deny).
6. **Architecture:** Design network access control for a three-tier app (web, app, database) using security groups. — Reference each tier's security group by ID as the SOURCE in the next tier's inbound rule (web SG → allowed into app SG on the app port; app SG → allowed into database SG on the DB port) rather than by IP range — this way the rule stays correct automatically even as instances are replaced, since it's referencing the group, not a specific changing IP.
7. **Comparison:** Referencing a security group by ID as a rule's source vs. using a CIDR range — why does it matter? — Referencing by security group ID means the rule automatically applies to any current member of that group, surviving instance replacement/autoscaling without editing rules; a CIDR range is a static snapshot that can drift out of sync as the actual fleet changes.
8. **Follow-up:** Why do NACL rules need explicit rule numbers, and why does the order matter? — NACLs evaluate rules in ascending numeric order and stop at the FIRST match — an early broad "deny all" rule would silently override a more specific "allow" rule numbered after it, so ordering (lower number = evaluated first) is a real, easy-to-get-wrong design decision.
9. **Pressure:** Traffic is being denied and you cannot find any explicit deny rule anywhere in the security group or NACL. What are you missing? — Check for an IMPLICIT default: security groups default-deny anything not explicitly allowed (there's no visible "deny all" rule to find because it's the built-in default, not a rule you configured), and NACLs typically have an explicit trailing default-deny rule (rule `*`) that's easy to overlook since it's not usually shown alongside your custom numbered rules.

---

## Load Balancing Drill

1. **Basic:** What problem does a load balancer solve that a single server can't? — Distributes traffic across multiple backends so no single server is a bottleneck or a single point of failure, and automatically routes around unhealthy backends.
2. **Why:** Why terminate TLS at the load balancer instead of at each backend? — Centralizes certificate management (one place to renew/rotate certs instead of every backend), offloads the CPU cost of encryption from application servers, and simplifies backend configuration — at the cost of the load balancer-to-backend hop potentially being unencrypted unless you deliberately re-encrypt it.
3. **How:** How does a load balancer actually decide a backend is "unhealthy"? — Configured health checks — periodic requests (often HTTP GET to a specific path, or a plain TCP connect) on a schedule; after a configured number of consecutive failures, the backend is pulled from rotation until it passes a configured number of consecutive successes again.
4. **Scenario:** After a deploy, the load balancer's target group takes several minutes to shift traffic to new instances, causing brief errors. — Health check interval and unhealthy/healthy threshold counts directly control this delay — a longer interval or higher threshold count means slower detection of both failure and recovery; tuning these is a genuine tradeoff between false-positive flapping and detection speed.
5. **Troubleshooting:** All targets show unhealthy in the target group, but the app responds fine when you curl it directly on the instance. — Check the health check's exact configured path/port against what the app actually serves there, and check whether the backend's security group allows inbound traffic specifically from the load balancer (its security group or subnet), not just general internet access.
6. **Architecture:** Design a zero-downtime deployment using a load balancer and two backend groups. — Blue-green: deploy the new version to a separate target group, verify it's fully healthy, then shift the load balancer's listener/routing to the new group, keeping the old group briefly available for instant rollback if something's wrong.
7. **Comparison:** ALB vs NLB — when would you deliberately choose NLB despite ALB's richer feature set? — Non-HTTP protocols entirely (raw TCP/UDP services), extreme throughput/low-latency requirements, needing the true client source IP preserved without extra configuration, or needing a static IP for the load balancer itself (NLB supports this natively; ALB doesn't).
8. **Follow-up:** What's connection draining / deregistration delay, and why does it matter during a deploy? — A grace period before a target is fully removed from rotation, during which existing in-flight requests to it are allowed to finish rather than being abruptly cut — skipping this causes visible request failures exactly during deploys, which is a common, avoidable self-inflicted incident.
9. **Pressure:** Users report intermittent 502 errors that you can't reproduce on demand, and all targets show healthy. What's your investigation path? — Check target-level metrics for individual backend error rates (one flaky backend among many healthy ones can produce intermittent errors that "average out" to look fine in aggregate), check backend response timeouts against the load balancer's configured timeout (a slow-but-working backend can look like a 502 if it exceeds the LB's patience), and check connection draining behavior during any deploys that might correlate with the reports.

---

## Quick-Coverage Round

**Routing & NAT**
- What determines if a subnet is public or private? Its route table's default route — to an Internet Gateway (public) or NAT Gateway/nothing (private) — not a special flag.
- Why can a private subnet resource reach the internet outbound but not be reached inbound through NAT? NAT only permits return traffic for connections it initiated; nothing external can start a new one in through it.

**Sockets & Connections**
- What's the practical difference between `netstat` and `ss`? `ss` is the modern replacement — faster, reads kernel state more directly — `netstat` still works on many systems but is considered legacy.
- What does a huge count of `TIME_WAIT` sockets usually indicate? High connection churn — many short-lived connections opening and closing rapidly, often solvable with connection reuse/keep-alive.

**HTTP/TLS**
- What does `curl -v` show, stage by stage? DNS resolution, TCP connect, TLS handshake (cert details), request headers, response headers, body.
- Why might a cert error appear only sometimes for the same domain? Multiple backends behind a load balancer with inconsistent certificate deployment, or an expiring cert right at the boundary of your test window.

**Load Balancing / VPC (real reported topics)**
- VPC peering, one line: a direct, non-transitive network connection between two VPCs, as if they were on the same network, without going through the public internet.
- Site-to-site VPN, one line: an encrypted tunnel connecting an on-premises network to a VPC over the public internet, so on-prem resources can reach cloud resources (and vice versa) as if directly connected.
- Gateway vs Interface VPC endpoint? Gateway = route-table entry, free, S3/DynamoDB only. Interface = ENI-based, billed, PrivateLink, everything else.
- Why does VPC Peering's non-transitivity matter in practice? Connecting many VPCs via peering alone requires a full mesh of individual connections — Transit Gateway exists specifically to avoid that at scale.

**ARP/ICMP**
- Does a failed ping mean a service is down? No — ICMP is frequently blocked independently of real application traffic; always confirm with a port/application-level check too.
- What does ARP actually resolve? An IP address to a MAC address, for delivery on the local network segment — a separate step from the routing decision.

---

## Hot Topics Right Now

- **IPv6 awareness** — not commonly the focus of a 3–5 YOE round yet, but knowing that `AAAA` records exist and that dual-stack (IPv4+IPv6) environments are increasingly common is worth a one-line acknowledgment if it comes up.
- **Service mesh networking (brief awareness)** — tools like Istio/Linkerd insert a sidecar proxy that intercepts pod-to-pod traffic for encryption, retries, and observability without application code changes — worth knowing the concept exists even without deep hands-on, since it connects directly to the Kubernetes networking topics coming in Document 4.
- **eBPF-based networking (Cilium and similar)** — an emerging alternative to traditional iptables-based Kubernetes networking, offering better performance and visibility — name recognition is enough at this level.

---

## Research-Backed Additions (Current Market, 2025–2026)

> [!NOTE]
> **What's actually confirmed from recent candidate reports:** A Genpact Bengaluru report (February 2026) listed these exact topics from a real DevOps interview: AWS VPC setup steps, cost optimization in cloud, on-premises-to-AWS connection, site-to-site VPN, IAM and Active Directory integration, Terraform pipeline explanation, CI/CD pipeline explanation. Pure "networking trivia" was not the focus — VPC connectivity and hybrid on-prem/cloud networking was.

> [!NOTE]
> **A PwC Bangalore report** included these exact questions, verbatim: "Can you deny traffic using a security group," "Diff ELB and NLB," "VPC peering." All three are covered directly above with the precise, correct answers.

> [!NOTE]
> **The sequential-diagnosis habit is what's actually being screened for, repeatedly, across multiple independent sources** — not memorized trivia. The consistent advice across current DevOps/SRE-focused writing: resolve DNS first, then check reachability, then check the port, then the HTTP layer, then packet capture if still stuck — in that order, because each layer's failure looks different and skipping ahead wastes time chasing the wrong layer.

- **A Globant report** described a VPC design question — "designing a secure VPC with private and public subnets and how to control access using firewall rules and IAM roles" — directly matching the subnetting lab in Section 2.
---

# Part 4 — Real Incident / Story Bank

## Story 1: The ECR VPC Endpoint for Private EKS Nodes

**Symptom:** EKS worker nodes, deliberately placed in private subnets with no NAT Gateway, needed to pull container images from ECR — with no direct path to the public internet at all.

**Initial hypothesis:** Without a NAT Gateway, private-subnet nodes have no route to ECR's public endpoint, so image pulls would simply fail.

**Evidence/investigation:** Confirmed the private subnets' route tables had no default route to a NAT Gateway or Internet Gateway — this was intentional (a stated architectural decision to keep nodes off the public internet entirely), not an oversight to "fix" by adding NAT.

**Narrowing:** The real question was how to give the nodes a path to ECR specifically, without opening general internet access — which is exactly what a VPC endpoint is for.

**Root cause:** Not a bug — a deliberate security posture (no internet access for worker nodes) that required the correct AWS networking primitive (interface VPC endpoints for ECR, plus the related STS and S3 endpoints ECR pulls also depend on) rather than the more common but less secure NAT Gateway approach.

**Fix:** Set up an ECR VPC interface endpoint, with a dedicated security group scoped to allow HTTPS from the node security group specifically.

**Verification:** Confirmed nodes could pull images successfully, and confirmed (via VPC Flow Logs, at a later stage) that no traffic from these nodes was reaching the public internet at all.

**Prevention/what this taught:** Private-subnet-only worker nodes are a genuinely stronger security posture than "private subnet with a NAT Gateway," but require deliberately identifying every AWS service dependency (ECR itself, plus STS for IAM role assumption, plus S3 since ECR uses S3 for image layer storage under the hood) and provisioning an endpoint for each — missing even one silently breaks the pull with a networking-layer symptom that looks identical to a permissions problem at first glance.

> [!TIP]
> **How to tell this in an interview:** "I set up EKS worker nodes in private subnets with no NAT Gateway at all, for a stronger security posture than the more common private-subnet-plus-NAT approach. That meant nodes had no path to ECR's public endpoint, so I provisioned an ECR interface VPC endpoint with a security group scoped to the node group specifically. The part that actually took investigation was realizing ECR itself depends on STS and S3 under the hood, so getting image pulls fully working meant identifying and endpointing all three, not just ECR."

---

## Story 2: The ALB Controller Webhook Timeout

**Symptom:** The AWS Load Balancer Controller, responsible for provisioning ALBs/NLBs from Kubernetes Ingress/Service objects, was failing with webhook timeout errors during deployment.

**Initial hypothesis:** The controller's admission webhook — a component the Kubernetes API server calls synchronously before accepting certain resource changes — wasn't reachable from the API server.

**Evidence/investigation:** Checked whether the webhook's Service and the underlying controller Pod were actually healthy and reachable within the cluster network — this is a within-cluster networking path, distinct from anything involving the public internet or the ALB itself.

**Narrowing:** Timeouts specifically on the admission webhook path point at either the webhook Pod not being ready yet when the API server tried to call it, or a network policy/security boundary blocking that specific internal path.

**Root cause:** A timing/ordering issue — the webhook configuration was registered with the API server before the controller Pod was fully ready to receive traffic, so early admission requests timed out against a target that wasn't yet listening.

**Fix:** Ensured the controller's readiness probe accurately reflected true webhook readiness (not just "process started"), so the API server's calls only reached it once genuinely ready, combined with a short retry/backoff tolerance on the webhook configuration itself.

**Verification:** Confirmed subsequent deployments no longer showed webhook timeout errors, and that ALB provisioning from Ingress objects succeeded reliably afterward.

**Prevention/what this taught:** A synchronous admission webhook is a hard dependency in the critical path of API operations — if it's not ready, everything depending on it fails immediately and confusingly, which is a different failure mode than an asynchronous controller simply reconciling slowly. This is a genuinely good example of why "is the process running" and "is the process actually ready to serve" are different questions with different consequences.

> [!TIP]
> **How to tell this in an interview:** "I hit webhook timeout errors from the AWS Load Balancer Controller during a deployment — the root cause was the webhook being registered as available before the controller Pod's readiness actually reflected it being able to serve requests, so the API server's synchronous calls to it timed out. The fix was tightening the readiness probe so it only reported ready once the webhook could genuinely respond, plus adding retry tolerance. It's a good example of why 'the container started' and 'the container is ready to be depended on' are different guarantees, especially for something in a synchronous critical path like an admission webhook."

---

## Practice Scenario (rehearsal only — not a personal account)

The following is a worked example for practicing the physical-flow narration format under pressure — not something to present as your own history.

*A user reports "the site is slow, sometimes it doesn't load at all." First check: `dig` the domain — resolves correctly, fast. Second: `curl -v` against the resolved IP directly — TCP connects fine, but the TLS handshake occasionally takes 4–5 seconds before completing, and occasionally times out entirely. This points away from DNS and toward either the network path or the load balancer/backend itself. `mtr` toward the IP shows clean, low-loss hops all the way to the destination — the public network path itself is fine. Attention shifts to the load balancer's target group: one of three backend targets shows an elevated response time in its own metrics compared to the other two, while still passing health checks (health checks were hitting a lightweight `/health` endpoint, not representative of real request load). The slow backend, once checked directly, was running with significantly higher CPU usage than its siblings — an uneven load distribution from a load-balancing algorithm change made the week before, not yet fully rolled back. Fix: revert the load-balancing algorithm change; verify by comparing response-time metrics across all three targets afterward, confirming they equalized.*

The lesson worth keeping regardless of whose story it is: **a health check passing does not mean a backend is performing well under real load** — health checks and real request-path performance are different questions, and intermittent, hard-to-reproduce symptoms often mean uneven distribution across backends rather than a single clean failure.

---

# Part 5 — Master Glossary (Active Recall)

Cover the definition, say the term's meaning out loud from memory, then check.

| Term | Definition |
|---|---|
| CIDR | Prefix notation for network size, e.g. /24 = 256 addresses (251 usable in an AWS subnet) |
| 4-tuple | Source IP, source port, destination IP, destination port — uniquely identifies a TCP connection |
| TTL | Decremented at each router hop; packet dropped and ICMP Time Exceeded sent when it hits zero |
| DNS resolver | The service an OS queries first; walks root → TLD → authoritative nameserver on a cache miss |
| A / CNAME / Alias record | Direct IP / alias to another name (can't coexist with other records at that name) / AWS-specific apex-safe alias |
| TCP handshake | SYN → SYN-ACK → ACK before any application data flows |
| RST vs silent drop | Active refusal — the host's own TCP stack responds, proving the host is reachable even though that port isn't — vs no response at all, commonly a firewall/SG dropping silently |
| Default gateway | Where a device sends traffic when the destination isn't on its local network |
| Public vs private subnet | Determined entirely by the route table's default route (IGW vs NAT Gateway/nothing) |
| NAT Gateway | Rewrites private source IPs to a shared public IP for outbound-only internet access |
| Security Group | Stateful, instance-level, allow-only, auto-permits return traffic |
| NACL | Stateless, subnet-level, ordered rules, supports explicit deny, needs explicit inbound AND outbound rules |
| ALB vs NLB | Layer 7 HTTP-aware path/host routing vs Layer 4 raw TCP/UDP, extreme throughput, preserves source IP |
| VPC Peering | Direct, non-transitive private connection between two VPCs |
| Transit Gateway | Central hub solving peering's non-transitivity for many VPCs |
| Site-to-Site VPN | Encrypted tunnel, on-prem to VPC, over the public internet |
| Direct Connect | Dedicated physical link, on-prem to AWS, consistent performance, higher cost/lead time |
| Gateway VPC Endpoint | Free, route-table-based, S3 and DynamoDB only |
| Interface VPC Endpoint | ENI-based, billed, PrivateLink, everything else (ECR, STS, Secrets Manager...) |
| ARP | Resolves an IP address to a MAC address for local-network delivery |
| ICMP | IP's own diagnostic/error protocol — ping and traceroute both rely on it; frequently blocked independently of real traffic |
| MTU | Max packet size a link can carry (1500 bytes standard Ethernet); VPNs/tunnels often reduce the effective MTU |
| CGNAT | ISP-level NAT sharing one public IP across many customers; look for 100.64.0.0/10 |
| TIME_WAIT | A brief post-close state absorbing delayed/duplicate packets before a 4-tuple is released for reuse |
| Health check (LB) | Periodic check determining if a backend stays in or leaves rotation — passing doesn't guarantee good real-world performance |
| Connection draining | Grace period letting in-flight requests finish before a target is fully removed from rotation |

---

# Part 6 — Additional Practical Exercises

## Exercise 1: Subnetting Under Pressure

**Concept:** Fast, correct CIDR math is a real, timed skill tested live in interviews (confirmed directly: Globant's VPC design question).

**Implementation:** Practice this pattern until it's fast: given `10.0.0.0/16`, produce 4 subnets of `/24` each, 2 public (with IGW routes) and 2 private (with NAT Gateway routes), across 2 AZs, with no overlapping ranges.

**Hands-on:** Time yourself. Then add a constraint mid-exercise — "now add a third AZ" — and practice extending the layout without renumbering what already exists.

**Failure:** You run out of clean address space or create overlapping ranges under time pressure.

**Investigation:** Recount bits — a /16 holds exactly 256 /24s; verify with a CIDR calculator, don't just estimate.

**Fix:** Leave deliberate numbering gaps between tiers from the start (e.g., reserve `10.0.1.0`–`10.0.9.0` for public, `10.0.11.0`–`10.0.19.0` for private) so a new AZ or tier can be inserted without renumbering.

**Verification:** No two ranges overlap; every subnet's role (public/private) is consistent with its intended route table.

**Interview Explanation:** "I leave numbering gaps between tiers when designing subnet layouts specifically so a new AZ or subnet can be added later without renumbering everything already deployed — renumbering a live VPC's subnets is disruptive and best avoided by planning gaps upfront."

---

## Exercise 2: Firewall Rule for a New Service

**Concept:** Moved here from the Linux guide, since it's fundamentally a networking-layer task. A recurring real task: allow a specific service through without accidentally opening more than intended.

**Implementation (Debian/Ubuntu — ufw):**
```bash
sudo ufw status verbose         # INSPECT current rules first
sudo ufw allow 8443/tcp         # EXECUTE — scoped to exactly the port needed
sudo ufw status numbered        # VERIFY the rule is present and correctly scoped
```

**Implementation (RHEL family — firewalld):**
```bash
sudo firewall-cmd --list-all                        # INSPECT
sudo firewall-cmd --permanent --add-port=8443/tcp    # EXECUTE (staged, not live yet)
sudo firewall-cmd --reload                           # apply
sudo firewall-cmd --list-ports                       # VERIFY
```

**Implementation (AWS security group, via CLI — treat as a change requiring the same INSPECT → VALIDATE → EXECUTE → VERIFY discipline as any production-facing rule):**
```bash
aws ec2 describe-security-groups --group-ids sg-0123456789 # INSPECT current rules
# VALIDATE: is this the correct, narrowest source (another SG ID, not 0.0.0.0/0)?
aws ec2 authorize-security-group-ingress --group-id sg-0123456789 \
  --protocol tcp --port 8443 --source-group sg-0987654321      # EXECUTE, scoped by SG reference
aws ec2 describe-security-groups --group-ids sg-0123456789 # VERIFY
```

**Failure:** After adding the rule, the service is still unreachable.

**Investigation:** Confirm the service is actually listening on that port at all (`ss -tulnp | grep 8443`) before assuming it's a firewall/security-group problem — a rule can be perfect and still not fix an application that isn't bound to the port, or is bound only to `127.0.0.1` instead of `0.0.0.0`.

**Fix:** Address whichever layer is actually broken.

**Verification:** `curl`/`nc -zv` the port from another host once both layers are confirmed correct — never assume success from the rule's presence alone.

**Interview Explanation:** "Before touching the firewall or security group, I confirm the service is actually listening with ss — a large share of 'firewall problems' are really the application not listening where expected. Once that's confirmed, I add the narrowest rule that solves the actual need — referencing a source security group instead of an open CIDR wherever possible — and verify from outside the host, not just by reading the rule back."

---

*End of Networking — Document 2 of 7. Next up: Docker.*
