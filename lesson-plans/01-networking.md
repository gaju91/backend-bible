# Section 1: Networking Fundamentals

> "Before your code runs a single line, the request has already traveled thousands of miles."

---

## The Problem This Solves

Your user clicks a button. Your server is in Virginia. The user is in Mumbai.

**The question:** How do those bytes travel 13,000 kilometers, find the right server among millions, establish a secure connection, and deliver the request - all in under 100 milliseconds?

If you don't understand this journey, you can't debug:
- Why your API is "slow" for some users
- Why your SSL certificate "doesn't work"
- Why load balancing "randomly fails"

---

## First Principles

### Principle 1: Data Travels Through Physical Infrastructure
```
User's Device → ISP → Internet Backbone → Data Center → Your Server
```
Each hop adds latency. Physics cannot be cheated.

### Principle 2: Names Must Resolve to Addresses
Humans use `google.com`. Machines use `142.250.190.14`. DNS bridges this gap.

### Principle 3: Connections Must Be Established Before Data Flows
TCP's three-way handshake (SYN → SYN-ACK → ACK) ensures both parties are ready.

### Principle 4: Security Requires Encryption Overhead
TLS handshake adds 1-2 round trips. This is the "HTTPS tax" - worth paying.

---

## Core Concepts

### 1. DNS Resolution
```
Browser: "Where is api.mystore.com?"
    ↓
Local DNS Cache (check first)
    ↓ (miss)
ISP DNS Resolver
    ↓
Root DNS Server → .com DNS → mystore.com DNS
    ↓
Answer: "192.168.1.100"
```

**Why it matters:** DNS failures = "Site not found" even if your server is perfect.

**Production reality:** DNS TTL (Time To Live) affects how fast you can failover. Low TTL = fast failover, more DNS queries. High TTL = slow failover, fewer queries.

### 2. TCP Connection
```
Client: SYN (Hey, want to talk?)
Server: SYN-ACK (Sure, I heard you)
Client: ACK (Great, let's go)
```

**Why it matters:** This takes 1 round trip. For a user in Australia connecting to US servers, that's ~150ms before any data flows.

### 3. TLS Handshake
```
Client: Hello, I support these encryption methods
Server: I choose this method, here's my certificate
Client: I verified your certificate, here's the encrypted key
Server: Got it, we're secure now
```

**Why it matters:** Another 1-2 round trips. TLS 1.3 reduced this to 1 round trip.

### 4. HTTP Request/Response
```
GET /api/products HTTP/1.1
Host: api.mystore.com
Authorization: Bearer eyJhbG...
```

**Why it matters:** HTTP is stateless by design. This constraint shapes everything that follows (auth tokens, session management, caching).

### 5. Load Balancing
```
                    ┌─→ Server 1
Client → Load Balancer─→ Server 2
                    └─→ Server 3
```

**Strategies:**
- Round Robin: Simple rotation
- Least Connections: Send to least busy
- IP Hash: Same user → same server (sticky sessions)

---

## The Latency Stack

| Step | Latency | Cumulative |
|------|---------|------------|
| DNS Lookup | 20-120ms | 20-120ms |
| TCP Handshake | 20-150ms | 40-270ms |
| TLS Handshake | 40-300ms | 80-570ms |
| HTTP Request | 20-100ms | 100-670ms |

**This is why:** First request to a new domain feels slow. Subsequent requests reuse connections (keep-alive).

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 2 (Routing)** | After network delivers bytes, routing finds the handler |
| **Section 4 (Auth)** | HTTP's statelessness forces token-based auth |
| **Section 8 (Caching)** | CDNs cache at the network edge to reduce latency |
| **Section 11 (APIs)** | HTTP/2, HTTP/3, WebSockets are networking evolutions |
| **Section 14 (Observability)** | Network metrics (latency, packet loss) need monitoring |

---

## Real-World Scenarios

### Scenario 1: The "Sometimes Slow" API
**Symptom:** API is fast for US users, slow for Asia users.
**Root Cause:** Single US data center, 200ms network latency to Asia.
**Solution:** CDN for static content, regional deployments for APIs.

### Scenario 2: The SSL Certificate Mystery
**Symptom:** "Certificate not trusted" errors in production only.
**Root Cause:** Missing intermediate certificates in the chain.
**Solution:** Include full certificate chain in server config.

### Scenario 3: The Load Balancer Lottery
**Symptom:** Users randomly get logged out.
**Root Cause:** Session stored in server memory, round-robin load balancing.
**Solution:** External session store (Redis) or stateless JWT tokens.

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Trace a request from browser to server
- [ ] Debug DNS issues using `dig` and `nslookup`
- [ ] Analyze TCP connections using `netstat` and `ss`
- [ ] Understand why HTTPS "costs" performance
- [ ] Configure load balancer strategies appropriately

---

## Seniority Challenges

### Junior Level
"What's the difference between HTTP and HTTPS?"

### Mid Level
"Our API has P99 latency of 2 seconds for users in Europe. We're hosted in US-East. What are our options?"

### Senior Level
"Design a global API infrastructure that keeps latency under 100ms for users worldwide while maintaining data consistency."

---

## Key Takeaways

1. **Network latency is physics** - You can't make light travel faster
2. **DNS is a single point of failure** - Have a backup strategy
3. **Connection setup is expensive** - Reuse connections (HTTP keep-alive)
4. **HTTPS is mandatory** - The performance cost is worth the security
5. **Load balancing affects state** - Stateless design is essential

---

## Prerequisites
None - this is where it all begins.

## Next Section
→ [Section 2: Routing](./02-routing.md) - Now that bytes have arrived, how do we find the right handler?
