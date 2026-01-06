# Section 12: Distributed Systems

> "A distributed system is one where a computer you didn't know existed can break your application."

---

## The Problem This Solves

Your monolith is:
- Deployed by 50 engineers who step on each other
- Scaled vertically until the biggest server isn't enough
- A single failure point that takes down everything

You split into microservices. Now you have:
- 30 services that must communicate reliably
- Network failures between every call
- Data that lives in multiple databases
- Transactions that span services

Welcome to distributed systems: where everything that can fail, will fail.

---

## First Principles

### Principle 1: The Network is Unreliable
Messages can be: delayed, dropped, duplicated, reordered.
Design assuming network WILL fail.

### Principle 2: The CAP Theorem
You can have at most 2 of:
- **C**onsistency: Every read gets the latest write
- **A**vailability: Every request gets a response
- **P**artition tolerance: System works despite network splits

Since network partitions WILL happen, you're really choosing between C and A during partitions.

### Principle 3: Failure is Normal
In a system with 100 servers, if each has 99.9% uptime, you'll have ~2.4 hours of at least one server down per day.

---

## Core Concepts

### 1. The Fallacies of Distributed Computing

**Eight False Assumptions:**
1. The network is reliable ❌
2. Latency is zero ❌
3. Bandwidth is infinite ❌
4. The network is secure ❌
5. Topology doesn't change ❌
6. There is one administrator ❌
7. Transport cost is zero ❌
8. The network is homogeneous ❌

**Reality:**
```
Service A calls Service B:
- Request might timeout (retry? duplicate work?)
- Response might be lost (success or failure?)
- Service B might be slow (wait or give up?)
- Service B might return error (retry or fail?)
```

### 2. Circuit Breaker Pattern

Prevent cascade failures by failing fast:

```
                    ┌─────────────────────────────────────────────────────────┐
                    │               CIRCUIT BREAKER STATES                    │
                    │                                                         │
                    │  ┌────────────┐                                        │
                    │  │   CLOSED   │  Normal operation                       │
                    │  │            │  Requests go through                    │
                    │  └─────┬──────┘                                        │
                    │        │                                                │
                    │        │ Failures exceed threshold                      │
                    │        ▼                                                │
                    │  ┌────────────┐                                        │
                    │  │    OPEN    │  Fail immediately                       │
                    │  │            │  Don't even try calling                 │
                    │  └─────┬──────┘                                        │
                    │        │                                                │
                    │        │ Timeout expires                                │
                    │        ▼                                                │
                    │  ┌────────────┐                                        │
                    │  │ HALF-OPEN  │  Test with single request              │
                    │  │            │  Success → CLOSED, Fail → OPEN         │
                    │  └────────────┘                                        │
                    │                                                         │
                    └─────────────────────────────────────────────────────────┘
```

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30):
        self.state = "CLOSED"
        self.failures = 0
        self.threshold = failure_threshold
        self.timeout = timeout

    def call(self, func):
        if self.state == "OPEN":
            if time.now() - self.opened_at > self.timeout:
                self.state = "HALF-OPEN"
            else:
                raise CircuitOpenError()

        try:
            result = func()
            if self.state == "HALF-OPEN":
                self.state = "CLOSED"
                self.failures = 0
            return result
        except Exception:
            self.failures += 1
            if self.failures >= self.threshold:
                self.state = "OPEN"
                self.opened_at = time.now()
            raise
```

### 3. Service Discovery

How does Service A find Service B?

```
                    ┌─────────────────────────────────────────────┐
                    │            SERVICE REGISTRY                 │
                    │       (Consul / Eureka / etcd)              │
                    │                                             │
                    │   ┌─────────────────────────────────────┐  │
                    │   │ user-service:                       │  │
                    │   │   - 192.168.1.10:8080 (healthy)     │  │
                    │   │   - 192.168.1.11:8080 (healthy)     │  │
                    │   │   - 192.168.1.12:8080 (unhealthy)   │  │
                    │   │                                     │  │
                    │   │ order-service:                      │  │
                    │   │   - 192.168.1.20:8080 (healthy)     │  │
                    │   └─────────────────────────────────────┘  │
                    │                                             │
                    └────────────────┬────────────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
         ▼                           ▼                           ▼
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│   Service A     │        │   Service B     │        │   Service C     │
│  Registers      │        │  Registers      │        │  Discovers      │
│  itself         │        │  itself         │        │  A and B        │
└─────────────────┘        └─────────────────┘        └─────────────────┘
```

### 4. Distributed Tracing

Follow a request across services:

```
Request ID: trace-abc-123

┌─────────────────────────────────────────────────────────────────────────────┐
│ API Gateway                                                                 │
│ trace-abc-123 | 0ms                                              200ms      │
│ ├────────────────────────────────────────────────────────────────────┤     │
└─┬───────────────────────────────────────────────────────────────────────────┘
  │
  │  ┌─────────────────────────────────────────────────────────────────────┐
  ├─→│ User Service                                                        │
  │  │ trace-abc-123 | 10ms                                      50ms      │
  │  │ ├──────────────────────────────────────────────────────────┤       │
  │  └────────────────────────────────────────────────────────────────────┘
  │
  │  ┌─────────────────────────────────────────────────────────────────────┐
  └─→│ Order Service                                                       │
     │ trace-abc-123 | 50ms                                     180ms      │
     │ ├──────────────────────────────────────────────────────────────┤   │
     └─┬───────────────────────────────────────────────────────────────────┘
       │
       │  ┌─────────────────────────────────────────────────────────────┐
       └─→│ Inventory Service                                           │
          │ trace-abc-123 | 60ms                            150ms       │
          │ ├─────────────────────────────────────────────────────┤     │
          └─────────────────────────────────────────────────────────────┘
```

Every service propagates the trace ID. Tools like Jaeger/Zipkin visualize the flow.

### 5. Consensus and Leader Election

When multiple nodes need to agree:

```
                    ┌─────────────────────────────────────────────┐
                    │            LEADER ELECTION                  │
                    │              (Raft / Paxos)                  │
                    │                                             │
                    │  Node 1 ──────┐                             │
                    │               │  Vote for                   │
                    │  Node 2 ──────┼──────→  Node 3 = LEADER    │
                    │               │                             │
                    │  Node 3 ◄─────┘                             │
                    │    │                                        │
                    │    │  All writes go to leader               │
                    │    │  Leader replicates to followers        │
                    │    ▼                                        │
                    │  ┌─────────────────────────────────────┐   │
                    │  │ Write "x=5"                         │   │
                    │  │   → Node 3 (leader) commits         │   │
                    │  │   → Replicates to Node 1, 2         │   │
                    │  │   → Acknowledges when majority has   │   │
                    │  └─────────────────────────────────────┘   │
                    │                                             │
                    └─────────────────────────────────────────────┘
```

### 6. Consistency Patterns

**Strong Consistency:**
```
Write to Node A → Read from Node B → Always see latest write
(Expensive: requires coordination)
```

**Eventual Consistency:**
```
Write to Node A → Read from Node B → Might see stale data
Wait long enough → Eventually see latest write
(Cheap: no coordination, but stale reads possible)
```

**Read Your Writes:**
```
You write → You read → You see your write
Others might see stale data
(Middle ground)
```

---

## Microservices Patterns

### Bulkhead Pattern

Isolate failures:

```
┌───────────────────────────────────────────────────────────────────┐
│                         WITHOUT BULKHEAD                          │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Shared Thread Pool (100 threads)               │ │
│  │   [User API] [Order API] [Payment API] [Shipping API]      │ │
│  │                                                             │ │
│  │   Payment API slow → All 100 threads waiting for payment   │ │
│  │   → User API, Order API, Shipping API all starved          │ │
│  │   → ENTIRE SYSTEM DOWN                                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                          WITH BULKHEAD                            │
│                                                                   │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────┐│
│  │   User API   │ │  Order API   │ │ Payment API  │ │ Shipping ││
│  │  25 threads  │ │  25 threads  │ │  25 threads  │ │25 threads││
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────┘│
│                                                                   │
│   Payment API slow → Only Payment's 25 threads affected          │
│   → User, Order, Shipping continue working                       │
│   → System degraded but functional                               │
└───────────────────────────────────────────────────────────────────┘
```

### Retry with Exponential Backoff

```python
def call_with_retry(func, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            return func()
        except TransientError:
            if attempt == max_attempts - 1:
                raise

            delay = (2 ** attempt) + random.uniform(0, 1)  # Exponential + jitter
            time.sleep(delay)
            # Attempt 1: ~1 second
            # Attempt 2: ~2 seconds
            # Attempt 3: ~4 seconds
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 1 (Networking)** | Network failures are the norm |
| **Section 8 (Database)** | Distributed databases, consistency |
| **Section 10 (Events)** | Async communication between services |
| **Section 11 (APIs)** | gRPC for internal, REST for external |
| **Section 14 (Observability)** | Distributed tracing is essential |
| **Section 15 (Cloud)** | Container orchestration for services |

---

## Real-World Scenarios

### Scenario 1: The Cascade Failure
**What happened:** Payment service slowed down. Order service threads blocked waiting for payment. Order service stopped responding. Gateway queued requests. Memory exhausted. Everything crashed.
**Fix:** Circuit breaker + bulkhead + timeout.

### Scenario 2: The Split Brain
**What happened:** Network partition split cluster into two halves. Both halves elected leaders. Both accepted writes. Partition healed. Conflicting data.
**Fix:** Quorum-based consensus (need majority to elect leader).

### Scenario 3: The Retry Storm
**What happened:** Service B went down. 100 instances of Service A immediately retried. Service B came back up, got 1000 requests instantly, crashed again.
**Fix:** Exponential backoff with jitter.

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design resilient microservices architectures
- [ ] Implement circuit breakers and bulkheads
- [ ] Set up service discovery
- [ ] Configure distributed tracing
- [ ] Reason about consistency trade-offs

---

## Seniority Challenges

### Junior Level
"What's the difference between a monolith and microservices?"

### Mid Level
"Design a system where Order Service needs to call User Service, Inventory Service, and Payment Service. How do you handle: User Service being slow, Inventory Service being down, Payment Service returning errors?"

### Senior Level
"Our 30-microservice system has: no service discovery (hardcoded IPs), no circuit breakers, synchronous chains of 5+ services, no distributed tracing, and p99 latency of 5 seconds. Design a 6-month roadmap to fix this without major rewrites."

---

## Key Takeaways

1. **Networks fail** - Design for it
2. **CAP theorem** - Choose your trade-off consciously
3. **Circuit breakers prevent cascades** - Fail fast when dependencies fail
4. **Distributed tracing is mandatory** - You can't debug without it
5. **Eventual consistency is usually enough** - Strong consistency is expensive

---

## Prerequisites
- Section 10: Events (async communication)
- Section 11: Modern APIs (service protocols)

## Next Section
→ [Section 13: Data Architecture](./13-data-architecture.md) - Moving and processing data at scale.
