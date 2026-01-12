# Module 0: How Computers Talk — The Foundation

> **By the end of this module, you will understand how data travels across networks, why the internet is structured in layers, and how to diagnose any connection problem in 30 seconds.**

---

## The Fundamental Question

Before we write a single line of backend code, we need to answer a question that most tutorials skip:

**How does one computer talk to another computer?**

When you type `fetch('https://api.stripe.com/charges')` in your code, what actually happens? How does your request travel from your laptop in Mumbai to a server in Virginia, and how does the response find its way back?

This isn't academic curiosity. Every production outage, every "works on my machine" mystery, every `Connection Refused` error traces back to this question. If you don't understand how computers communicate, you're debugging blind.

Let's build this understanding from scratch.

---

## Part 1: The Need for Order

### The Chaos Before Standards

Imagine it's 1970. You're an engineer at IBM, and you want your computer to talk to another IBM computer across the room. You write some code, run some wires, and it works.

Now your boss says: "Make it talk to that DEC computer too."

Problem: DEC designed their systems completely differently. Different wire signals, different data formats, different everything. Your IBM code is useless.

So you write new code specifically for IBM-to-DEC communication. It works.

Then your boss says: "Now add the Honeywell machine."

You see where this is going. If you have 10 different computer manufacturers, you potentially need 45 different translation systems. With 100 manufacturers? 4,950 combinations.

```
The Incompatibility Nightmare:

    IBM ─────── DEC
     │╲        ╱│
     │ ╲      ╱ │
     │  ╲    ╱  │
     │   ╲  ╱   │
     │    ╲╱    │
     │    ╱╲    │
     │   ╱  ╲   │
     │  ╱    ╲  │
     │ ╱      ╲ │
     │╱        ╲│
Honeywell ─── Wang

Every pair needs custom translation.
This doesn't scale.
```

This was the reality of early networking: **chaos**. Every vendor did their own thing. Networks couldn't talk to each other. The promise of connected computers was drowning in incompatibility.

The industry needed a common language—a **standard model** that everyone could follow.

### The Two Solutions

Two different groups tackled this problem:

**1. The International Standards Organization (ISO)** — A committee of academics and industry representatives who spent years designing the "perfect" theoretical model. They produced the **OSI Model** (Open Systems Interconnection) in 1984.

**2. DARPA Researchers** — Engineers at the U.S. Department of Defense who were actually building ARPANET (the internet's predecessor). They needed something that worked *now*. They produced the **TCP/IP Model** in the 1970s.

One was designed by committee. One was forged in practice.

Guess which one the internet actually runs on?

---

## Part 2: The Two Models

### The OSI Model: The Theoretical Framework

The OSI model divides networking into **seven layers**, each handling one specific aspect of communication:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 7: Application    │  What the user/app interacts    │
│                          │  with (HTTP, FTP, SMTP)         │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 6: Presentation   │  Data formatting, encryption,   │
│                          │  compression                    │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 5: Session        │  Managing connections           │
│                          │  (start, maintain, end)         │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 4: Transport      │  Reliable delivery, ports       │
│                          │  (TCP, UDP)                     │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 3: Network        │  Addressing and routing         │
│                          │  (IP addresses)                 │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 2: Data Link      │  Node-to-node transfer          │
│                          │  (MAC addresses, switches)      │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 1: Physical       │  Raw bits on the wire           │
│                          │  (cables, signals, voltages)    │
└─────────────────────────────────────────────────────────────┘
```

The [OSI model](https://en.wikipedia.org/wiki/OSI_model) is elegant. It's comprehensive. It's taught in every networking course.

**But here's the secret: almost nobody implements it exactly.**

Layers 5 and 6 (Session and Presentation) rarely exist as distinct layers in real systems. They're usually absorbed into the Application layer or handled by libraries.

So why learn it? Because the OSI model gave us **vocabulary**. When engineers say "Layer 7 load balancer" or "Layer 4 firewall," they're using OSI terminology. You need to understand the reference even if the implementation differs.

### The TCP/IP Model: The Practical Reality

While the ISO committee was perfecting their seven-layer design, DARPA engineers were shipping code. They created a simpler, four-layer model that actually ran the ARPANET:

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Application    │  HTTP, FTP, SMTP, DNS, SSH      │
│                          │  (Combines OSI layers 5, 6, 7)  │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 3: Transport      │  TCP, UDP                       │
│                          │  (Same as OSI layer 4)          │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 2: Internet       │  IP, ICMP, ARP                  │
│                          │  (Same as OSI layer 3)          │
├──────────────────────────┼──────────────────────────────────┤
│  Layer 1: Network Access │  Ethernet, WiFi, Fiber          │
│                          │  (Combines OSI layers 1 & 2)    │
└─────────────────────────────────────────────────────────────┘
```

The [TCP/IP model](https://en.wikipedia.org/wiki/Internet_protocol_suite) won because:

1. **It worked.** While OSI was being debated, TCP/IP was running the ARPANET.
2. **It was simpler.** Four layers are easier to implement than seven.
3. **It was free.** The specifications were public and not controlled by any vendor.
4. **It scaled.** It grew into the modern internet.

### Mapping OSI to TCP/IP

Here's how the two models align:

```
        OSI Model                    TCP/IP Model
    ┌───────────────┐           ┌───────────────────┐
    │ 7 Application │           │                   │
    ├───────────────┤           │                   │
    │ 6 Presentation│──────────►│  4 Application    │
    ├───────────────┤           │                   │
    │ 5 Session     │           │                   │
    ├───────────────┼───────────┼───────────────────┤
    │ 4 Transport   │──────────►│  3 Transport      │
    ├───────────────┼───────────┼───────────────────┤
    │ 3 Network     │──────────►│  2 Internet       │
    ├───────────────┤           ├───────────────────┤
    │ 2 Data Link   │──────────►│                   │
    ├───────────────┤           │  1 Network Access │
    │ 1 Physical    │──────────►│                   │
    └───────────────┘           └───────────────────┘
```

**For your career, remember this:**

- The **TCP/IP model** is how the internet actually works
- The **OSI numbers** (L3, L4, L7) are the vocabulary everyone uses
- When someone says "L7 load balancer," they mean Application layer (OSI terminology) even though the system runs TCP/IP

---

## Part 3: Why Layers? The Concept of Abstraction

Before we dive into what each layer does, let's understand **why** we split networking into layers at all.

### The Problem Layers Solve

Imagine if every application had to handle:

- Converting data to electrical signals
- Dealing with wire interference and errors
- Finding a path through thousands of routers
- Making sure data arrives in order
- Retransmitting lost packets
- Formatting data for the destination
- Encrypting sensitive information

No one would write networked applications. It's too much complexity.

**Layers solve this through abstraction.** Each layer:

1. **Handles one specific concern** — and does it well
2. **Hides complexity from layers above** — upper layers don't need to know the details
3. **Provides a consistent interface** — so layers can be swapped without breaking everything

### How Layers Communicate

Layers don't talk directly to their peers across the network. Instead, data flows **down** through the layers on the sending side, across the network, and **up** through the layers on the receiving side.

```
Computer A                                           Computer B

┌─────────────┐                                     ┌─────────────┐
│ Application │◄─────── Logical Connection ────────►│ Application │
├─────────────┤                                     ├─────────────┤
│  Transport  │◄─────── Logical Connection ────────►│  Transport  │
├─────────────┤                                     ├─────────────┤
│   Network   │◄─────── Logical Connection ────────►│   Network   │
├─────────────┤                                     ├─────────────┤
│  Physical   │                                     │  Physical   │
└──────┬──────┘                                     └──────┬──────┘
       │                                                   │
       └─────────────► Physical Medium ◄───────────────────┘
                    (actual data transmission)
```

Each layer thinks it's talking directly to its peer on the other machine. But in reality, the data travels down the stack, across the wire, and back up.

### Encapsulation: The Russian Doll

As data moves down through the layers, each layer wraps the data with its own header (and sometimes trailer). This is called **encapsulation**.

Think of it like sending a letter:

1. You write a **letter** (your data)
2. You put it in an **envelope** with a name (application layer)
3. The postal service puts that in a **shipping container** with an address (network layer)
4. The shipping container goes on a **truck** with a route number (physical layer)

Each layer adds information needed for its job, without modifying what's inside.

```
┌─────────────────────────────────────────────────────────────┐
│                    ENCAPSULATION                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Application Data:     [  Your JSON payload  ]              │
│                                                             │
│                              │                              │
│                              ▼                              │
│                                                             │
│  + Transport Header:   [TCP][  Your JSON payload  ]         │
│                                                             │
│                              │                              │
│                              ▼                              │
│                                                             │
│  + Network Header:    [IP][TCP][  Your JSON payload  ]      │
│                                                             │
│                              │                              │
│                              ▼                              │
│                                                             │
│  + Frame Header/Trailer: [ETH][IP][TCP][ JSON ][ETH]        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

At the destination, each layer strips its header and passes the data up. The application receives exactly what was sent—it never sees the headers added by lower layers.

### Layer Names for Data

Each layer calls its unit of data by a different name:

| Layer | Data Unit Name |
|-------|----------------|
| Application | **Data** or **Message** |
| Transport | **Segment** (TCP) or **Datagram** (UDP) |
| Network | **Packet** |
| Data Link | **Frame** |
| Physical | **Bits** |

When someone says "the packet was dropped," they're talking about Layer 3. When they say "I'm looking at the frame," they're talking about Layer 2.

---

## Part 4: The Three Layers That Matter

Now that you understand the models and how layers work, let's focus on the three layers you'll interact with daily as a backend engineer:

| OSI Layer | Name | TCP/IP Equivalent | Your Mental Model |
|-----------|------|-------------------|-------------------|
| **L3** | Network | Internet | **"Can I find it?"** — IP addresses, routing |
| **L4** | Transport | Transport | **"Can I connect?"** — Ports, TCP/UDP, firewalls |
| **L7** | Application | Application | **"Do we understand each other?"** — HTTP, protocols |

Layers 1-2 (Physical, Data Link) are handled by your operating system and network hardware. You rarely interact with them directly.

Layers 5-6 (Session, Presentation) are absorbed into Layer 7 in practice.

**Your world is L3, L4, and L7.**

---

## Part 5: Layer 3 — The Network Layer

**The Question:** *Can I find the server?*

### What Layer 3 Does

Layer 3 handles **addressing** and **routing**. It answers: "Given a destination address, how do I get a packet there?"

**Key Protocol:** IP (Internet Protocol)

**Responsibilities:**

- Assigning unique addresses to every device (IP addresses)
- Finding a path from source to destination (routing)
- Splitting large messages into smaller packets if needed (fragmentation)

**What it does NOT do:**

- Guarantee delivery (packets can be lost)
- Guarantee order (packets can arrive shuffled)
- Know which application should receive the data

### IP Addresses: The Postal System

An IP address is like a postal address—it identifies where a device is on the network.

**IPv4 format:** Four numbers from 0-255, separated by dots.

```
192.168.1.105
```

32 bits total = approximately 4.3 billion possible addresses. We've run out, which is why IPv6 exists (128 bits), but IPv4 still dominates.

### Public vs. Private IP Addresses

**Public IPs** are globally unique—routable on the internet. Stripe's server might be `54.187.174.169`. Only one device can have this address.

**Private IPs** are reserved for internal networks:

```
10.0.0.0    – 10.255.255.255     (10.x.x.x)
172.16.0.0  – 172.31.255.255     (172.16-31.x.x)
192.168.0.0 – 192.168.255.255    (192.168.x.x)
```

Your laptop's `192.168.1.105` is a private IP. It only means something inside your home network. To reach the internet, your router performs **NAT (Network Address Translation)**—it swaps your private IP for its public IP on outgoing traffic and reverses it for responses.

```
Your Home Network                           Internet

┌─────────────────────────┐
│                         │
│  Laptop: 192.168.1.105  │
│  Phone:  192.168.1.106  │◄──► Router ◄──────► Stripe: 54.187.174.169
│  TV:     192.168.1.107  │    (NAT)
│                         │  Public IP:
│  Private IPs            │  203.0.113.42
└─────────────────────────┘
```

### Routing: How Packets Find Their Way

The internet is millions of networks connected by routers. Each router maintains a **routing table**—rules for forwarding packets toward their destination.

When you send a request to `54.187.174.169`:

1. Your laptop sends to your home router
2. Your router forwards to your ISP
3. Your ISP forwards toward the destination network
4. Each hop gets the packet closer
5. Eventually it arrives at Stripe's server

This happens in milliseconds.

### DNS: Names to Addresses

Humans don't memorize IP addresses. [DNS](https://www.cloudflare.com/learning/dns/what-is-dns/) (Domain Name System) translates `stripe.com` to `54.187.174.169`.

When you request `api.stripe.com`:

1. Your computer checks its local cache
2. If not cached, asks a DNS resolver
3. The resolver queries the DNS hierarchy (root → .com → stripe.com)
4. Returns the IP address
5. Your application can now connect

### Diagnostic Tools for Layer 3

**`ping`** — Tests if a host is reachable

```bash
ping google.com
```

```
PING google.com (142.250.190.78): 56 data bytes
64 bytes from 142.250.190.78: icmp_seq=0 ttl=117 time=14.2 ms
```

[`ping`](https://en.wikipedia.org/wiki/Ping_(networking_utility)) sends [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) packets and waits for a response. Success proves DNS works and the host is reachable.

**`traceroute`** — Shows the path packets take

```bash
traceroute google.com
```

```
 1  router.local (192.168.1.1)      2.1 ms
 2  isp-gateway (10.0.0.1)         12.3 ms
 3  backbone-router (198.51.100.1) 22.4 ms
 ...
10  google-server (142.250.190.78) 14.8 ms
```

Each line is a router. Useful for finding where latency or drops occur.

**`dig`** — Queries DNS directly

```bash
dig api.stripe.com
```

```
;; ANSWER SECTION:
api.stripe.com.     300    IN    A    54.187.174.169
```

Shows the IP address and TTL (how long it's cached).

### Layer 3 Failure Symptoms

| Error | Meaning |
|-------|---------|
| `Unknown host` / `ENOTFOUND` | DNS can't resolve the hostname |
| `No route to host` | IP exists but no network path |
| `Network unreachable` | Your machine has no connectivity |
| `Request timeout` | Host may be down, or blocking ICMP |

---

## Part 6: Layer 4 — The Transport Layer

**The Question:** *Can I connect to the right application?*

### What Layer 4 Does

Layer 3 gets packets to the right machine. Layer 4 gets them to the right **application** on that machine—and optionally ensures reliable delivery.

**Key Protocols:** TCP, UDP

**Responsibilities:**

- Identifying applications via port numbers
- Reliable delivery (TCP) or fast delivery (UDP)
- Flow control and congestion control
- Breaking data into segments

### Ports: Apartment Numbers

A server runs many applications. Port numbers identify which one should receive the data.

```
54.187.174.169:443

54.187.174.169  = Building (which machine)
443             = Apartment (which application)
```

**Common ports:**

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |

Port numbers range from 0-65535. Ports 0-1023 are "well-known" and typically require admin privileges.

### TCP vs. UDP

**TCP (Transmission Control Protocol)** — Reliable delivery

- **Connection-oriented:** Establishes connection before sending data
- **Reliable:** Guarantees all data arrives, in order
- **Flow control:** Won't overwhelm the receiver
- **Use cases:** HTTP, databases, file transfer—anything where accuracy matters

**UDP (User Datagram Protocol)** — Fast delivery

- **Connectionless:** Just sends packets, no setup
- **Unreliable:** Packets can be lost, duplicated, or reordered
- **Fast:** No overhead of reliability mechanisms
- **Use cases:** Video streaming, gaming, DNS—where speed beats accuracy

**Backend development is 95% TCP.** HTTP, database connections, Redis—all TCP.

### The TCP Handshake

Before sending data, TCP establishes a connection via a three-way handshake:

```
Client                                Server
   │                                     │
   │──────── SYN (seq=100) ─────────────►│  "I want to connect"
   │                                     │
   │◄─── SYN-ACK (seq=300, ack=101) ─────│  "OK, acknowledged"
   │                                     │
   │──────── ACK (ack=301) ─────────────►│  "Connection established"
   │                                     │
   │         Connection is now open      │
   │◄────────── Data flows ─────────────►│
```

This ensures both sides are ready. When this handshake fails, you get:

- **Connection refused:** SYN reached server, but nothing is listening (RST sent back)
- **Connection timed out:** SYN got no response (firewall dropping, or server down)

### Firewalls

Firewalls examine packets and allow or block based on rules. Most operate at Layer 4, filtering by:

- Source/destination IP
- Destination port
- Protocol (TCP/UDP)

**Cloud firewalls:**

| Service | Scope |
|---------|-------|
| **Security Groups** (AWS/GCP) | Per-instance |
| **Network ACLs** (AWS) | Per-subnet |
| **iptables** (Linux) | Per-server |

When your app works locally but times out in production, check firewall rules first.

### Diagnostic Tool for Layer 4

**`nc` (netcat)** — Tests if a port is open

```bash
nc -zv stripe.com 443
```

[`nc`](https://en.wikipedia.org/wiki/Netcat) attempts to establish a TCP connection.

**Success:**

```
Connection to stripe.com port 443 [tcp/https] succeeded!
```

**Failures:**

```
Connection refused      # Nothing listening on that port
Operation timed out     # Firewall dropping packets
```

The distinction matters:

- **Refused** = Server is reachable, port is closed
- **Timeout** = Packets aren't getting through (firewall)

### Layer 4 Failure Symptoms

| Error | Meaning |
|-------|---------|
| `Connection refused` | Port reachable, but nothing listening |
| `Connection timed out` | Firewall blocking, or server unreachable |
| `Connection reset` | Server forcibly closed the connection |

---

## Part 7: Layer 7 — The Application Layer

**The Question:** *Do we understand each other?*

### What Layer 7 Does

Layers 3 and 4 deliver bytes reliably. Layer 7 gives those bytes **meaning**.

**Key Protocols:** HTTP, HTTPS, SSH, SMTP, DNS, PostgreSQL protocol, Redis protocol, etc.

**Responsibilities:**

- Defining message formats (requests, responses)
- Encoding/decoding data
- Authentication and authorization
- Application-specific logic

### What is a Protocol?

A protocol is an agreed-upon format for communication. Without it, two applications would exchange meaningless bytes.

[HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) (HyperText Transfer Protocol) defines:

- Request format: method, path, headers, body
- Response format: status code, headers, body
- Meaning of status codes: 200 = OK, 404 = Not Found, etc.

When your code calls `fetch('https://api.stripe.com/charges')`, HTTP turns that into:

```
GET /charges HTTP/1.1
Host: api.stripe.com
Authorization: Bearer sk_test_xxx
Accept: application/json
```

And interprets the response:

```
HTTP/1.1 200 OK
Content-Type: application/json

{"data": [...]}
```

### HTTP Status Codes

Status codes tell you what happened:

**2xx — Success**

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |

**3xx — Redirection**

| Code | Meaning |
|------|---------|
| 301 | Moved Permanently |
| 302 | Found (Temporary Redirect) |
| 304 | Not Modified (use cache) |

**4xx — Client Error**

| Code | Meaning |
|------|---------|
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 429 | Too Many Requests |

**5xx — Server Error**

| Code | Meaning |
|------|---------|
| 500 | Internal Server Error |
| 502 | Bad Gateway |
| 503 | Service Unavailable |
| 504 | Gateway Timeout |

### Diagnostic Tool for Layer 7

**`curl`** — Makes HTTP requests

```bash
curl -v https://api.stripe.com/v1/charges
```

[`curl`](https://curl.se/) with `-v` shows the full conversation:

```
* Trying 54.187.174.169:443...
* Connected to api.stripe.com (54.187.174.169) port 443
* SSL connection using TLSv1.3

> GET /v1/charges HTTP/1.1
> Host: api.stripe.com
> User-Agent: curl/7.79.1

< HTTP/1.1 401 Unauthorized
< Content-Type: application/json

{"error": {"message": "You did not provide an API key."}}
```

- `*` lines — Connection info (L3/L4)
- `>` lines — Your request
- `<` lines — Server response

### Layer 7 Failure Symptoms

| Error | Meaning |
|-------|---------|
| 4xx status | Client-side issue (auth, bad request) |
| 5xx status | Server-side issue (crash, overload) |
| Malformed response | Server bug or misconfiguration |

---

## Part 8: Putting It All Together

### The Journey of a Request

When your code executes:

```javascript
fetch('https://api.stripe.com/v1/charges', {
  method: 'POST',
  headers: { 'Authorization': 'Bearer sk_test_xxx' },
  body: JSON.stringify({ amount: 1000 })
})
```

**Layer 7 (Application):** Formats the HTTP request

```
POST /v1/charges HTTP/1.1
Host: api.stripe.com
Authorization: Bearer sk_test_xxx
Content-Type: application/json

{"amount": 1000}
```

**Layer 4 (Transport):** Wraps in TCP segment with ports

```
[TCP: src=54321, dst=443] + HTTP request
```

**Layer 3 (Network):** Wraps in IP packet with addresses

```
[IP: src=203.0.113.42, dst=54.187.174.169] + TCP + HTTP
```

**Layer 1-2 (Physical/Link):** Converts to signals, transmits

At Stripe's server, the process reverses. Headers are stripped layer by layer until the application receives your JSON.

### The Debugging Methodology

**Debug bottom-up: L3 → L4 → L7**

Each layer depends on layers below. If L3 is broken, L4 and L7 can't work.

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Step 1: Test Layer 3                                       │
│  Command: ping <hostname>                                   │
│                                                             │
│  ✓ Responds      → Proceed to Step 2                        │
│  ✗ Unknown host  → DNS issue                                │
│  ✗ No route      → Routing/network issue                    │
│  ✗ Timeout       → Server down or blocking ICMP             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 2: Test Layer 4                                       │
│  Command: nc -zv <hostname> <port>                          │
│                                                             │
│  ✓ Succeeded         → Proceed to Step 3                    │
│  ✗ Connection refused → Service not listening               │
│  ✗ Timed out          → Firewall blocking                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 3: Test Layer 7                                       │
│  Command: curl -v <url>                                     │
│                                                             │
│  ✓ 2xx → Working correctly                                  │
│  ✗ 4xx → Client-side problem (auth, request format)         │
│  ✗ 5xx → Server-side problem (check logs)                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 9: Practice Scenarios

### Scenario 1

```
Error: getaddrinfo ENOTFOUND database.internal
```

**Layer:** 3 (DNS failure)

**Tool:** `dig database.internal` or `ping database.internal`

**Causes:** Typo in hostname, wrong DNS server, not on VPN

---

### Scenario 2

```
Error: connect ECONNREFUSED 10.0.0.5:5432
```

**Layer:** 4 (Nothing listening)

**Tool:** `nc -zv 10.0.0.5 5432`

**Causes:** PostgreSQL not running, wrong port, listening on localhost only

---

### Scenario 3

```
HTTP 502 Bad Gateway
```

**Layer:** 7 (Upstream failed)

**Tool:** `curl -v https://api.example.com/health`

**Causes:** Application crashed, timeout, misconfigured proxy

---

### Scenario 4

App works locally, `Connection timed out` in production.

**Layer:** 4 (Firewall)

**Tool:** `nc -zv database.example.com 5432` from production server

**Causes:** Security Group missing inbound/outbound rule

---

## Part 10: Hands-On Exercise

Test all three layers:

```bash
# Layer 3: Can I find it?
ping google.com

# Layer 3: What's the path?
traceroute google.com

# Layer 4: Can I connect to the port?
nc -zv google.com 443

# Layer 7: Does the application respond?
curl -I https://google.com
```

You've just walked the entire network stack.

---

## Quick Reference

### Diagnostic Commands

| Layer | Tool | Command | Tests |
|-------|------|---------|-------|
| L3 | ping | `ping <host>` | DNS + reachability |
| L3 | traceroute | `traceroute <host>` | Network path |
| L3 | dig | `dig <host>` | DNS resolution |
| L4 | nc | `nc -zv <host> <port>` | Port connectivity |
| L7 | curl | `curl -v <url>` | HTTP request/response |

### Error → Layer Mapping

| Error Pattern | Layer | Likely Cause |
|---------------|-------|--------------|
| `Unknown host`, `ENOTFOUND` | L3 | DNS failure |
| `No route to host` | L3 | Routing issue |
| `Connection refused` | L4 | Service not running |
| `Connection timed out` | L4 | Firewall blocking |
| `4xx` HTTP status | L7 | Client error |
| `5xx` HTTP status | L7 | Server error |

### Common Ports

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |

---

## Summary

You now understand:

1. **Why models exist** — The chaos before standards demanded a common framework
2. **OSI vs TCP/IP** — OSI is the vocabulary, TCP/IP is the reality
3. **Why layers matter** — Abstraction, separation of concerns, debuggability
4. **How layers communicate** — Encapsulation down, decapsulation up
5. **The three layers you care about:**
   - **L3 (Network):** IP addresses, routing, DNS
   - **L4 (Transport):** Ports, TCP/UDP, firewalls
   - **L7 (Application):** HTTP, protocols, status codes
6. **How to debug systematically** — ping → nc → curl

When you see a connection error, you no longer guess. You identify the layer, use the right tool, and fix the root cause.

---

## What's Next

This module gave you the foundational understanding of networking models and layers. In **Module 1: The Journey**, we'll trace a complete request in action:

- The full DNS resolution process
- The TCP three-way handshake in detail
- How HTTP structures the conversation

And you'll **build a raw TCP server from scratch**—no frameworks, no libraries. You'll open a socket, accept connections, and respond to clients.

**[→ Continue to Module 1: The Journey](./01-the-journey.md)**
