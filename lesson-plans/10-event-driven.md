# Section 10: Event-Driven Architecture

> "The best integration is no integration at all. Services shouldn't know about each other."

---

## The Problem This Solves

Your order service needs to:
- Update inventory service
- Notify payment service
- Tell shipping service
- Inform analytics service
- Update search service

With direct calls:
```python
def create_order(order):
    save_order(order)
    inventory_service.decrement(order.items)    # Direct call
    payment_service.charge(order.total)          # Direct call
    shipping_service.create_shipment(order)      # Direct call
    analytics_service.track(order)               # Direct call
    search_service.index(order)                  # Direct call
```

**Problems:**
- Order service knows about 5 other services
- If shipping service is slow, order service is slow
- Adding a new service = changing order service code
- One failure cascades to all

**Event-driven solution:** Order service publishes "order.created" event. Other services subscribe. Order service doesn't know or care who's listening.

---

## First Principles

### Principle 1: Producers Shouldn't Know About Consumers
The order service says "an order was created." It doesn't say "send to inventory, payment, shipping." This decoupling is the core benefit.

### Principle 2: Events Are Facts, Not Commands
- Command: "Decrement inventory!" (telling someone what to do)
- Event: "Order was created." (stating what happened)

Commands create coupling. Events enable independence.

### Principle 3: Eventual Consistency is the Trade-off
Services update independently. The system is consistent "eventually" but not immediately. This is the price of decoupling.

---

## Core Concepts

### 1. Event Types

**Domain Events:**
```json
{
  "type": "order.created",
  "timestamp": "2024-01-15T12:00:00Z",
  "data": {
    "order_id": "ord_123",
    "user_id": "user_456",
    "items": [{"product_id": "prod_789", "quantity": 1}],
    "total": 99.99
  }
}
```

**Integration Events:**
```json
{
  "type": "user.email_verified",
  "source": "auth-service",
  "timestamp": "2024-01-15T12:00:00Z",
  "data": {
    "user_id": "user_456",
    "email": "user@example.com"
  }
}
```

### 2. Pub/Sub Pattern

```
                    ┌─────────────────────────────────────────┐
                    │           MESSAGE BROKER                │
                    │         (Kafka / RabbitMQ)              │
                    │                                         │
   PUBLISHER        │     Topic: "orders"                     │        SUBSCRIBERS
┌──────────────┐    │  ┌─────────────────────────────────┐   │    ┌──────────────┐
│              │    │  │                                 │   │    │   Inventory  │
│    Order     │───→│  │  order.created ──────────────────────→  │   Service    │
│   Service    │    │  │  order.updated                  │   │    └──────────────┘
│              │    │  │  order.cancelled                │   │    ┌──────────────┐
└──────────────┘    │  │                                 │   │    │   Payment    │
                    │  └─────────────────────────────────┘   │───→│   Service    │
                    │                                         │    └──────────────┘
                    │     Topic: "users"                      │    ┌──────────────┐
┌──────────────┐    │  ┌─────────────────────────────────┐   │    │   Shipping   │
│              │    │  │                                 │   │───→│   Service    │
│    User      │───→│  │  user.registered               │   │    └──────────────┘
│   Service    │    │  │  user.updated                   │   │    ┌──────────────┐
│              │    │  │                                 │   │    │  Analytics   │
└──────────────┘    │  └─────────────────────────────────┘   │───→│   Service    │
                    │                                         │    └──────────────┘
                    └─────────────────────────────────────────┘
```

### 3. Event Sourcing

Instead of storing current state, store all events:

```
Traditional: Store current state
┌────────────────────────────────────────┐
│ Account: #12345                        │
│ Balance: $500                          │
│ Last updated: 2024-01-15               │
└────────────────────────────────────────┘

Event Sourcing: Store all events
┌────────────────────────────────────────┐
│ Event 1: AccountOpened($1000)          │
│ Event 2: Withdrawal($200)              │
│ Event 3: Deposit($100)                 │
│ Event 4: Withdrawal($400)              │
│                                        │
│ Current Balance: $1000 - $200 + $100   │
│                  - $400 = $500         │
└────────────────────────────────────────┘
```

**Benefits:**
- Complete audit trail
- Time-travel debugging ("What was the state at 2pm yesterday?")
- Rebuild state from events

**Costs:**
- Complexity
- Storage growth
- Query performance (need projections)

### 4. CQRS (Command Query Responsibility Segregation)

Separate write path from read path:

```
                WRITE PATH                          READ PATH
                (Commands)                          (Queries)

┌──────────────┐                          ┌──────────────┐
│  Create      │                          │  Get Order   │
│  Order       │                          │  Details     │
└──────┬───────┘                          └──────┬───────┘
       │                                         │
       ▼                                         ▼
┌──────────────┐                          ┌──────────────┐
│   Command    │                          │    Query     │
│   Handler    │                          │   Handler    │
└──────┬───────┘                          └──────┬───────┘
       │                                         │
       ▼                                         ▼
┌──────────────┐     Events          ┌──────────────┐
│   Write DB   │───────────────────→ │   Read DB    │
│ (Normalized) │    Projection       │(Denormalized)│
└──────────────┘                      └──────────────┘
```

**Why separate?**
- Write DB optimized for consistency (normalized, ACID)
- Read DB optimized for queries (denormalized, fast)
- Scale reads and writes independently

### 5. Saga Pattern

Distributed transactions across services:

```
SAGA: Order Processing

Step 1: Reserve Inventory
        │
        ├── Success → Step 2
        └── Failure → END (nothing to rollback)

Step 2: Process Payment
        │
        ├── Success → Step 3
        └── Failure → Compensate: Release Inventory → END

Step 3: Create Shipment
        │
        ├── Success → END (complete)
        └── Failure → Compensate: Refund Payment
                   → Compensate: Release Inventory
                   → END
```

**Implementation:**
```python
class OrderSaga:
    def execute(self, order):
        try:
            # Step 1
            inventory_reservation = inventory_service.reserve(order.items)

            try:
                # Step 2
                payment = payment_service.charge(order.user, order.total)

                try:
                    # Step 3
                    shipment = shipping_service.create(order)
                    return Success(order, shipment)

                except ShippingError:
                    # Compensate step 2
                    payment_service.refund(payment)
                    raise

            except PaymentError:
                # Compensate step 1
                inventory_service.release(inventory_reservation)
                raise

        except InventoryError:
            # Nothing to compensate
            raise
```

---

## Event Schema Design

### Good Event Design

```json
{
  "id": "evt_abc123",              // Unique event ID for idempotency
  "type": "order.created",         // Namespaced event type
  "source": "order-service",       // Which service produced it
  "timestamp": "2024-01-15T12:00:00.000Z",
  "correlation_id": "req_xyz789", // For tracing
  "data": {
    "order_id": "ord_123",
    "user_id": "user_456",
    "total": 99.99
  },
  "metadata": {
    "version": "1.0",
    "schema": "https://schemas.example.com/order-created/v1"
  }
}
```

### Schema Evolution

**Adding fields (safe):**
```json
// v1
{ "order_id": "123", "total": 99.99 }

// v2 - Added field, old consumers ignore it
{ "order_id": "123", "total": 99.99, "currency": "USD" }
```

**Removing fields (unsafe):**
```json
// v1
{ "order_id": "123", "total": 99.99 }

// v2 - Removed total, old consumers BREAK
{ "order_id": "123", "amount": { "value": 99.99, "currency": "USD" } }

// Solution: Keep both during migration period
{ "order_id": "123", "total": 99.99, "amount": { "value": 99.99, "currency": "USD" } }
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 8 (Database)** | Event sourcing as storage pattern |
| **Section 9 (Async)** | Events trigger async processing |
| **Section 11 (APIs)** | Events complement request/response |
| **Section 12 (Distributed)** | Events enable loose coupling |
| **Section 13 (Data)** | Events feed analytics pipelines |
| **Section 14 (Observability)** | Correlation IDs for tracing |

---

## Real-World Scenarios

### Scenario 1: The Lost Event
**Problem:** Inventory wasn't decremented for some orders.
**Root Cause:** Consumer crashed after processing, before committing offset. Event reprocessed, but not idempotent.
**Fix:** Consumer checks if already processed before acting.

### Scenario 2: The Ordering Problem
**Problem:** User received "Order Shipped" email before "Order Confirmed."
**Root Cause:** Events processed out of order (parallel consumers).
**Fix:** Sequence numbers, or process all events for same entity on same partition.

### Scenario 3: The Schema Break
**Problem:** New event field broke all consumers.
**Root Cause:** Changed `total` from number to object without migration.
**Fix:** Schema registry, backward-compatible changes only.

---

## Kafka vs RabbitMQ

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Model | Log-based | Queue-based |
| Retention | Keeps messages | Deletes after consume |
| Replay | Yes | No (by default) |
| Ordering | Per partition | Per queue |
| Throughput | Very high | High |
| Latency | Medium | Low |
| Use case | Event streaming, analytics | Task queues, RPC |

**Choose Kafka when:** Event sourcing, high throughput, replay needed, multiple consumers
**Choose RabbitMQ when:** Task queues, complex routing, low latency

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design event-driven architectures
- [ ] Implement pub/sub patterns
- [ ] Handle eventual consistency
- [ ] Design event schemas for evolution
- [ ] Implement saga patterns for distributed transactions

---

## Seniority Challenges

### Junior Level
"What's the difference between synchronous API calls and event-driven communication?"

### Mid Level
"Design an event-driven system for e-commerce: order placement should update inventory, trigger payment, and initiate shipping. How do you handle failures? What events do you publish?"

### Senior Level
"We're splitting our monolith into microservices. Currently, all data is in one database with foreign keys. Design the event-driven migration: what events, what order of extraction, how to maintain consistency during transition, and how to handle the dual-write problem."

---

## Key Takeaways

1. **Events = Facts** - Immutable records of what happened
2. **Producers don't know consumers** - Core decoupling benefit
3. **Eventual consistency** - Accept this trade-off consciously
4. **Idempotency required** - Events may be delivered multiple times
5. **Schema evolution** - Design for change from day one

---

## Prerequisites
- Section 9: Async Systems (understanding background processing)

## Next Section
→ [Section 11: Modern API Patterns](./11-modern-apis.md) - Beyond REST: GraphQL, gRPC, WebSockets.
