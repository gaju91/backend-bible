# Section 11: Modern API Patterns

> "Different problems need different protocols. REST is not the answer to everything."

---

## The Problem This Solves

REST works great for CRUD. But what about:

- **Mobile app needs 10 related resources** → 10 HTTP requests, slow on 3G
- **Real-time chat** → Polling every second is wasteful
- **Microservice calling another 1000 times/second** → JSON parsing overhead adds up
- **Complex query with nested filters** → URL becomes unmanageable

One protocol can't optimize for all these scenarios.

---

## First Principles

### Principle 1: Protocol Choice is a Trade-off
Each protocol optimizes for different things:
- REST: Simplicity, cacheability, universality
- GraphQL: Client flexibility, reduced round trips
- gRPC: Performance, type safety
- WebSockets: Real-time bidirectional

### Principle 2: Match Protocol to Use Case
- Public API consumed by many clients → REST
- Mobile app with limited bandwidth → GraphQL
- Internal service-to-service → gRPC
- Live updates → WebSockets

### Principle 3: You Can Mix Protocols
REST for public API, gRPC internally, WebSockets for real-time features. They coexist.

---

## Core Concepts

### 1. REST (Representational State Transfer)

**What it is:** Resource-oriented, stateless, HTTP-based.

```http
GET    /users/123           # Read user
POST   /users               # Create user
PUT    /users/123           # Update user
DELETE /users/123           # Delete user
```

**Strengths:**
- Simple, well-understood
- HTTP caching works
- Stateless (scales easily)
- Universal client support

**Weaknesses:**
- Over-fetching (get full user when you need just name)
- Under-fetching (need user, orders, reviews = 3 requests)
- Versioning complexity

**Best for:** Public APIs, simple CRUD, cacheable resources.

### 2. GraphQL

**What it is:** Query language for APIs. Client specifies exactly what data it needs.

```graphql
# Client asks for exactly what it needs
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
      items {
        productName
        quantity
      }
    }
  }
}

# Server returns exactly that
{
  "data": {
    "user": {
      "name": "Sarah",
      "email": "sarah@example.com",
      "orders": [
        {
          "id": "ord_1",
          "total": 99.99,
          "items": [
            { "productName": "Shoes", "quantity": 1 }
          ]
        }
      ]
    }
  }
}
```

**Strengths:**
- No over-fetching or under-fetching
- Single request for complex data
- Strong typing with schema
- Introspection (self-documenting)

**Weaknesses:**
- Caching is harder (POST requests, dynamic queries)
- N+1 query problem on server
- Complexity for simple cases
- Security (query depth attacks)

**Best for:** Mobile apps, complex UIs, when clients need flexibility.

### 3. gRPC (Google Remote Procedure Call)

**What it is:** Binary protocol using Protocol Buffers. Function calls over HTTP/2.

```protobuf
// Define service in .proto file
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

```python
# Client code (generated from .proto)
user = user_service_stub.GetUser(GetUserRequest(user_id="123"))
print(user.name)  # Type-safe access
```

**Strengths:**
- 10x faster than JSON (binary, smaller, faster parsing)
- Strong typing (compile-time errors)
- Bidirectional streaming
- Code generation in any language

**Weaknesses:**
- Not human-readable (hard to debug)
- Browser support limited (needs proxy)
- Schema required (no ad-hoc queries)
- Tooling less mature than REST

**Best for:** Internal service-to-service, high-throughput, polyglot environments.

### 4. WebSockets

**What it is:** Persistent bidirectional connection over TCP.

```javascript
// Client
const ws = new WebSocket('wss://api.example.com/chat');

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};

// Server can push anytime
ws.send(JSON.stringify({ type: 'message', text: 'Hello!' }));
```

**Strengths:**
- True real-time (no polling)
- Bidirectional (server can push)
- Low latency (persistent connection)
- Efficient for frequent updates

**Weaknesses:**
- Stateful (harder to scale)
- No built-in request/response correlation
- Connection management complexity
- Not cacheable

**Best for:** Chat, live dashboards, gaming, collaborative editing.

---

## Protocol Comparison

```
                    REST        GraphQL      gRPC         WebSocket
                    ────        ───────      ────         ─────────
Data format         JSON        JSON         Protobuf     Any
Transport           HTTP/1.1    HTTP/1.1     HTTP/2       TCP
Caching             Easy        Hard         Medium       None
Browser support     Full        Full         Limited      Full
Type safety         Optional    Schema       Strong       None
Real-time           Polling     Subscriptions Streaming   Native
Learning curve      Low         Medium       High         Medium
Debugging           Easy        Medium       Hard         Medium
```

---

## API Gateway Pattern

When you have multiple protocols, an API Gateway unifies them:

```
                              ┌─────────────────────────────────────────┐
                              │            API GATEWAY                  │
                              │                                         │
 Browser ──── REST ────────→  │  ┌─────────────────────────────────────┐│
                              │  │ • Authentication                    ││
 Mobile ──── GraphQL ──────→  │  │ • Rate Limiting                     ││
                              │  │ • Protocol Translation               ││
 IoT ─────── MQTT ─────────→  │  │ • Load Balancing                    ││
                              │  │ • Caching                           ││
                              │  └─────────────────────────────────────┘│
                              │                    │                    │
                              └────────────────────┼────────────────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              │                              │
                    ▼                              ▼                              ▼
            ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
            │User Service  │              │Order Service │              │Product Service│
            │   (gRPC)     │              │   (gRPC)     │              │   (gRPC)     │
            └──────────────┘              └──────────────┘              └──────────────┘
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 1 (Networking)** | HTTP/2 for gRPC, WebSocket upgrades |
| **Section 2 (Routing)** | GraphQL single endpoint vs REST routes |
| **Section 3 (Serialization)** | JSON vs Protobuf |
| **Section 8 (Caching)** | REST caching vs GraphQL challenges |
| **Section 10 (Events)** | GraphQL subscriptions, gRPC streaming |
| **Section 12 (Distributed)** | Internal gRPC, external REST |

---

## Real-World Scenarios

### Scenario 1: The Mobile Data Explosion
**Problem:** Mobile app makes 15 REST calls to load home screen. Users on slow networks wait 10 seconds.
**Solution:** GraphQL endpoint that returns everything in one request. Load time: 2 seconds.

### Scenario 2: The Chat Polling Problem
**Problem:** Chat app polls `/messages` every second. 1000 users = 60,000 requests/minute. Mostly empty responses.
**Solution:** WebSocket connection. Server pushes only when new message. Requests dropped 99%.

### Scenario 3: The Service Mesh Overhead
**Problem:** Internal services communicate via REST. JSON parsing takes 40% of CPU at scale.
**Solution:** gRPC for internal communication. CPU usage dropped to 15%.

---

## Choosing the Right Protocol

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROTOCOL DECISION TREE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Is it a public API?                                                        │
│  ├── YES: Do clients need flexibility?                                      │
│  │        ├── YES → GraphQL                                                 │
│  │        └── NO  → REST                                                    │
│  │                                                                          │
│  └── NO (internal):                                                         │
│       Is it real-time bidirectional?                                        │
│       ├── YES → WebSocket or gRPC streaming                                 │
│       └── NO: Is performance critical?                                      │
│              ├── YES → gRPC                                                 │
│              └── NO  → REST (simpler debugging)                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Choose appropriate protocol for use case
- [ ] Design GraphQL schemas
- [ ] Implement gRPC services
- [ ] Build WebSocket-based features
- [ ] Set up API gateways

---

## Seniority Challenges

### Junior Level
"What's the main difference between REST and GraphQL?"

### Mid Level
"Design the API layer for a social media app with: feed (personalized, real-time updates), messaging (real-time), profile CRUD, and search. Which protocols for which features?"

### Senior Level
"We're exposing our API to third-party developers. Requirements: versioning, rate limiting, authentication, documentation, SDKs for 5 languages, and real-time webhooks. Design the API platform architecture."

---

## Key Takeaways

1. **No universal solution** - Each protocol has trade-offs
2. **REST is the default** - Use others when REST falls short
3. **GraphQL for flexibility** - When clients have diverse needs
4. **gRPC for performance** - Internal high-throughput communication
5. **WebSockets for real-time** - When polling isn't good enough

---

## Prerequisites
- Section 1: Networking (HTTP, protocols)
- Section 3: Serialization (JSON, Protobuf)

## Next Section
→ [Section 12: Distributed Systems](./12-distributed-systems.md) - When one server isn't enough.
