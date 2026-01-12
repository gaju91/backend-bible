# Section 1: Networking - Start Here

> "First understand how data travels, then writing code becomes easy"

---

## Welcome to Networking

This section is all about networking - when Sarah clicked "Buy Now", how did her request reach the server? What does DNS do? Why does a TCP handshake happen? What actually is HTTP?

You'll find answers to all these questions in this section.

---

## How We've Organized This

In README.md we listed 11 topics (1.1 to 1.11). For easier learning, we've grouped them into **5 broader categories**.

Each category = **1 Blog + 1 Video + Hands-On Project**

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│   README Topics (1.1 - 1.11)                                               │
│          │                                                                 │
│          ▼                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     5 LEARNING MODULES                              │  │
│   │                                                                     │  │
│   │   [0] Foundations     → OSI Model, Protocol Evolution               │  │
│   │   [1] The Journey     → DNS, TCP, Network Topology                  │  │
│   │   [2] Speaking HTTP   → Request/Response, Methods, Status Codes     │  │
│   │   [3] Headers         → Headers, Caching, CORS                      │  │
│   │   [4] Production      → TLS, Performance, nginx                     │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```
---
## README.md Topic Mapping

If you want to directly read a specific topic:

| README Topic                    | Covered In                  |
| ------------------------------- | --------------------------- |
| 1.1 Request Lifecycle Deep Dive | Module 1: The Journey       |
| 1.2 Network Topology            | Module 1: The Journey       |
| 1.3 HTTP Fundamentals           | Module 2: Speaking HTTP     |
| 1.4 Headers                     | Module 3: Headers & Control |
| 1.5 Methods                     | Module 2: Speaking HTTP     |
| 1.6 CORS                        | Module 3: Headers & Control |
| 1.7 Status Codes                | Module 2: Speaking HTTP     |
| 1.8 Caching                     | Module 3: Headers & Control |
| 1.9 Protocol Evolution          | Module 0: Foundations       |
| 1.10 Performance                | Module 4: Production        |
| 1.11 Transport Security         | Module 4: Production        |

---

## Learning Path

```
Module 0: Foundations
    │
    │   "Understand the map first"
    ▼
Module 1: The Journey
    │
    │   "From DNS to TCP, the request's journey"
    │   🔧 Build: Raw TCP Server
    ▼
Module 2: Speaking HTTP
    │
    │   "Now how do we communicate? HTTP!"
    │   🔧 Build: HTTP Server on TCP
    ▼
Module 3: Headers & Control
    │
    │   "Fine-tune the communication"
    │   🔧 Build: Add Headers, Test CORS
    ▼
Module 4: Production
    │
    │   "Now become production ready"
    │   🔧 Build: nginx + TLS setup

    ✅ Section Complete!
```

---

## The 5 Modules

### Module 0: The Foundations

**What:** OSI Model, TCP/IP Model, HTTP Evolution (1.1 → 2 → 3)

**Why:** For debugging, you need to know which layer the problem is at

**Hands-On:** `ping`, `traceroute`, see packets with Wireshark

📄 [Read the Blog](./00-foundations.md) | 🎥 [Watch the Video](#)

---

### Module 1: The Journey

**What:** DNS Resolution, TCP Handshake, Network Topology

**Why:** How does Sarah's click reach the server - complete trace

**Hands-On:** Build a Raw TCP Server (Python) - without libraries!

📄 [Read the Blog](./01-the-journey.md) | 🎥 [Watch the Video](#)

---

### Module 2: Speaking HTTP

**What:** Request/Response structure, HTTP Methods, Status Codes

**Why:** "HTTP is just text" - prove it yourself

**Hands-On:** Build an HTTP Server on raw TCP - this is what Express does internally

📄 [Read the Blog](./02-speaking-http.md) | 🎥 [Watch the Video](#)

---

### Module 3: Headers & Control

**What:** Request/Response Headers, Caching, CORS

**Why:** Headers control behavior - caching, security, everything

**Hands-On:** Add headers to your server, test CORS

📄 [Read the Blog](./03-headers-and-control.md) | 🎥 [Watch the Video](#)

---

### Module 4: Production Hardening

**What:** TLS/HTTPS, Performance (compression, keep-alive), nginx

**Why:** From development to production - become real world ready

**Hands-On:** nginx setup with load balancing + self-signed certificate

📄 [Read the Blog](./04-production.md) | 🎥 [Watch the Video](#)

---

## What You'll Build

By the end of this section, you will have built:

1. **Raw TCP Server** - Using sockets, without libraries
2. **HTTP Server** - On top of TCP, like Express but your own
3. **nginx Configuration** - Load balancing, reverse proxy
4. **TLS Setup** - Self-signed certificate

After building all this, frameworks won't feel like magic - they'll feel like logic.

---

## Prerequisites

- Basic programming knowledge (Python examples will be used)
- Terminal/Command line basics
- Curiosity 🧠

---

## Let's Start!

Ready? Start with [Module 0: Foundations](./00-foundations.md).

Or if you want hands-on directly, jump to [Module 1: The Journey](./01-the-journey.md) where we build our first TCP server.

---

**Next Section:** [02-the-routing](../02-the-routing/) - Request reached the server, now who handles it?
