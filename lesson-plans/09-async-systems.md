# Section 9: Asynchronous Systems

> "The fastest response is the one that doesn't wait for work to complete."

---

## The Problem This Solves

Sarah's order is saved. Now you need to:
- Send confirmation email (500ms)
- Update analytics dashboard (200ms)
- Notify warehouse system (800ms)
- Generate invoice PDF (1000ms)
- Update search index (300ms)

Total: 2.8 seconds. Sarah is staring at a spinner. She's already opening a competitor's tab.

**The insight:** Sarah doesn't need to WAIT for these things. She just needs to know her order succeeded.

---

## First Principles

### Principle 1: User Perception Matters More Than Completion
Users measure speed by response time, not total processing time. Return fast, finish later.

### Principle 2: Not All Work is Equal
- **Synchronous:** User needs the result to proceed (e.g., "What's my order total?")
- **Asynchronous:** User doesn't need immediate result (e.g., "Email me the receipt")

### Principle 3: Async Requires Reliability Guarantees
If you promise to send an email later, you MUST send it. Failures need retry mechanisms.

---

## Core Concepts

### 1. The Async Pattern

```
SYNCHRONOUS (User waits)
┌──────┐    ┌────────┐    ┌───────┐
│Client│───→│ Server │───→│ Email │
│      │    │        │    │Service│
│      │←───│        │←───│       │
└──────┘    └────────┘    └───────┘
   │                           │
   └─────── 500ms ─────────────┘

ASYNCHRONOUS (User doesn't wait)
┌──────┐    ┌────────┐    ┌───────┐    ┌───────┐
│Client│───→│ Server │───→│ Queue │   →│Worker │───→│Email│
│      │←───│        │    │       │    │       │    │     │
└──────┘    └────────┘    └───────┘    └───────┘    └─────┘
   │             │
   └── 50ms ─────┘
            (Email sent later in background)
```

### 2. Message Queues

```
                    PRODUCER                  QUEUE                    CONSUMER
               ┌──────────────┐        ┌─────────────────┐        ┌──────────────┐
               │              │        │                 │        │              │
               │ Order Service│───────→│  send_email     │───────→│ Email Worker │
               │              │        │  ┌───┐┌───┐┌───┐│        │              │
               └──────────────┘        │  │ 1 ││ 2 ││ 3 ││        └──────────────┘
                                       │  └───┘└───┘└───┘│
                                       │                 │
                                       │  update_search  │───────→┌──────────────┐
                                       │  ┌───┐┌───┐     │        │Search Worker │
                                       │  │ A ││ B │     │        └──────────────┘
                                       │  └───┘└───┘     │
                                       └─────────────────┘
```

**Popular Queue Systems:**
| System | Best For |
|--------|----------|
| Redis (Bull/Celery) | Simple jobs, low latency |
| RabbitMQ | Complex routing, reliability |
| Amazon SQS | Managed, scalable |
| Kafka | High throughput, event streaming |

### 3. Job/Task Anatomy

```python
# Producing a job
def on_order_created(order):
    # Immediate response to user
    response = {"order_id": order.id, "status": "confirmed"}

    # Queue background jobs
    queue.enqueue(
        job_type="send_confirmation_email",
        payload={"order_id": order.id, "email": order.user.email},
        priority="high",
        retry_count=3,
        retry_delay=60  # seconds
    )

    queue.enqueue(
        job_type="update_analytics",
        payload={"order_id": order.id, "amount": order.total},
        priority="low"
    )

    return response

# Consuming a job
@worker.task("send_confirmation_email")
def send_confirmation_email(payload):
    order_id = payload["order_id"]
    email = payload["email"]

    order = OrderRepository.get(order_id)
    EmailService.send(
        to=email,
        template="order_confirmation",
        data={"order": order}
    )
```

### 4. Delivery Guarantees

**At-Most-Once:**
```
Send job → Worker processes → Done (no ack required)
If worker crashes → Job lost
Use for: Non-critical analytics, metrics
```

**At-Least-Once:**
```
Send job → Worker processes → Worker sends ACK → Job removed
If worker crashes before ACK → Job redelivered
Use for: Most business operations (email, notifications)
⚠️ Must handle duplicates (idempotency)
```

**Exactly-Once:**
```
Complex coordination between producer, queue, consumer
Expensive, rarely truly achievable
Use for: Financial transactions
Usually implemented via: At-least-once + idempotency
```

### 5. Idempotency - The Key to Reliability

If a job runs twice, the result should be the same as running once.

```python
# BAD: Not idempotent
def send_email(order_id):
    order = get_order(order_id)
    send_email(order.user.email, "Your order is confirmed!")
    # If job runs twice, user gets two emails 😞

# GOOD: Idempotent
def send_email(order_id):
    order = get_order(order_id)

    # Check if already sent
    if cache.exists(f"email_sent:{order_id}"):
        return  # Already done, skip

    send_email(order.user.email, "Your order is confirmed!")

    # Mark as sent
    cache.set(f"email_sent:{order_id}", "true", ttl=86400)
```

**Idempotency patterns:**
- Idempotency keys (client provides unique ID)
- State checks (already processed?)
- Database constraints (unique index prevents duplicates)

---

## Job Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                        JOB STATES                               │
│                                                                 │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐              │
│  │ PENDING  │─────→│  ACTIVE  │─────→│COMPLETED │              │
│  └──────────┘      └────┬─────┘      └──────────┘              │
│       │                 │                                       │
│       │                 │ Failure                               │
│       │                 ▼                                       │
│       │           ┌──────────┐                                  │
│       │           │ RETRYING │──────────┐                       │
│       │           └────┬─────┘          │                       │
│       │                │                │ Max retries           │
│       │                │ Retry          │ exceeded              │
│       │                ▼                ▼                       │
│       │           Back to PENDING  ┌──────────┐                 │
│       │                            │  FAILED  │                 │
│       │                            └────┬─────┘                 │
│       │                                 │                       │
│       │                                 ▼                       │
│       │                          ┌─────────────┐                │
│       └─────────────────────────→│ DEAD LETTER │                │
│         (Failed after all       │   QUEUE     │                │
│          retries)               └─────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Dead Letter Queue (DLQ)

Jobs that fail all retries go to the DLQ for investigation:

```python
# DLQ handler - alert and store for analysis
@worker.dlq_handler
def handle_dead_letter(job):
    alert_ops_team(
        message=f"Job {job.id} failed permanently",
        job_type=job.type,
        payload=job.payload,
        error=job.last_error
    )

    # Store for manual retry
    db.insert("failed_jobs", {
        "job_id": job.id,
        "type": job.type,
        "payload": job.payload,
        "error": job.last_error,
        "failed_at": datetime.now()
    })
```

---

## Retry Strategies

### Exponential Backoff

```python
def calculate_retry_delay(attempt):
    """
    Attempt 1: 1 second
    Attempt 2: 2 seconds
    Attempt 3: 4 seconds
    Attempt 4: 8 seconds
    ...with jitter to prevent thundering herd
    """
    base_delay = 2 ** attempt  # Exponential
    jitter = random.uniform(0, 1)  # Random 0-1 second
    return base_delay + jitter
```

### Circuit Breaker for External Services

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN

    def call(self, func):
        if self.state == "OPEN":
            raise CircuitOpenError("Service unavailable")

        try:
            result = func()
            self.failures = 0
            self.state = "CLOSED"
            return result
        except Exception as e:
            self.failures += 1
            if self.failures >= self.threshold:
                self.state = "OPEN"
                self.schedule_reset()
            raise
```

---

## Scheduled Jobs (Cron)

```python
# Run every day at 2 AM
@scheduler.cron("0 2 * * *")
def daily_report():
    generate_daily_sales_report()
    send_to_management()

# Run every 5 minutes
@scheduler.interval(minutes=5)
def health_check():
    check_external_services()
    update_status_page()

# Run once at specific time
@scheduler.at("2024-12-31 23:59:59")
def new_year_notification():
    send_to_all_users("Happy New Year!")
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 7 (Controllers)** | Controllers enqueue jobs after operations |
| **Section 8 (Database)** | Jobs often read/write database |
| **Section 10 (Events)** | Events trigger async processing |
| **Section 12 (Distributed)** | Cross-service async communication |
| **Section 14 (Observability)** | Job metrics, failure tracking |

---

## Real-World Scenarios

### Scenario 1: The Missing Email
**Problem:** Users report not receiving order confirmations.
**Investigation:** Email job succeeded (no errors), but email never sent.
**Root Cause:** Email service was down. Job "succeeded" (API call returned 200) but email wasn't actually queued.
**Fix:** End-to-end confirmation, check email service response properly.

### Scenario 2: The Infinite Retry
**Problem:** Worker CPU at 100%, processing same job repeatedly.
**Root Cause:** Job throws exception, retries, throws exception... forever.
**Fix:** Max retry limit, exponential backoff, DLQ for failed jobs.

### Scenario 3: The Priority Inversion
**Problem:** High-priority payment jobs stuck behind millions of low-priority analytics jobs.
**Fix:** Separate queues per priority, dedicated workers for high-priority.

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design async workflows appropriately
- [ ] Implement reliable job processing
- [ ] Handle failures with retries and DLQ
- [ ] Make operations idempotent
- [ ] Monitor and debug async systems

---

## Seniority Challenges

### Junior Level
"What's the difference between synchronous and asynchronous processing?"

### Mid Level
"Design an email notification system that: sends order confirmations, handles failures gracefully, doesn't send duplicates, and can be monitored for issues."

### Senior Level
"Our async job system processes 1M jobs/day. We need to: add job prioritization, implement rate limiting per customer, handle cross-datacenter job distribution, and maintain exactly-once semantics for payment processing. Design the architecture."

---

## Key Takeaways

1. **Return fast, process later** - User experience first
2. **Idempotency is mandatory** - Jobs will be retried
3. **Failures are expected** - Design for retry and DLQ
4. **Monitor everything** - Async failures are invisible without monitoring
5. **Separate priorities** - Don't let bulk jobs block critical ones

---

## Prerequisites
- Section 8: Database (understanding data persistence)

## Next Section
→ [Section 10: Event-Driven Architecture](./10-event-driven.md) - Beyond jobs: how services communicate without coupling.
