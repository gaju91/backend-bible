# Section 14: Observability & Security

> "Without observability, you're flying blind. Without security, you shouldn't fly at all."

---

## The Problem This Solves

It's 3 AM. Users are complaining. Your response:
- "What's broken?" (No idea)
- "Which service?" (No idea)
- "Since when?" (No idea)
- "What changed?" (No idea)
- "How many users affected?" (No idea)

You restart everything and hope for the best.

This is the life without observability. With it, you'd know in 30 seconds: "Order service has 50% error rate since 2:47 AM when deployment abc123 went out. 500 users affected. Rollback initiated."

---

## First Principles

### Principle 1: You Can't Fix What You Can't See
Systems are black boxes until instrumented. Every interaction should be observable.

### Principle 2: Three Pillars Work Together
- **Logs:** What happened (events)
- **Metrics:** How much/fast (numbers)
- **Traces:** What caused what (flow)

Each answers different questions. All are needed.

### Principle 3: Observability is Not Optional
It's not a "nice to have." It's the only way to operate production systems responsibly.

---

## Core Concepts

### 1. The Three Pillars

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         THREE PILLARS OF OBSERVABILITY                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐                                                   │
│  │       LOGS          │  "What happened?"                                 │
│  │                     │                                                   │
│  │  2024-01-15 12:00:01 ERROR OrderService: Payment failed for order_123  │
│  │  2024-01-15 12:00:01 INFO  OrderService: Retrying payment...           │
│  │  2024-01-15 12:00:02 ERROR OrderService: Retry failed, circuit open    │
│  │                     │                                                   │
│  │  → Debug specific issues                                                │
│  │  → Audit trail                                                          │
│  │  → High cardinality (unique values)                                     │
│  └─────────────────────┘                                                   │
│                                                                             │
│  ┌─────────────────────┐                                                   │
│  │      METRICS        │  "How much? How fast?"                            │
│  │                     │                                                   │
│  │  http_requests_total{service="orders", status="500"} = 1523            │
│  │  http_request_duration_seconds{service="orders", p99} = 2.5            │
│  │  orders_created_total = 50000                                          │
│  │                     │                                                   │
│  │  → Dashboards                                                           │
│  │  → Alerting                                                             │
│  │  → Capacity planning                                                    │
│  └─────────────────────┘                                                   │
│                                                                             │
│  ┌─────────────────────┐                                                   │
│  │      TRACES         │  "What caused what?"                              │
│  │                     │                                                   │
│  │  Request abc-123:                                                       │
│  │  ├── API Gateway (10ms)                                                 │
│  │  ├── Order Service (150ms)                                              │
│  │  │   └── Database query (140ms) ← SLOW!                                │
│  │  └── Total: 200ms                                                       │
│  │                     │                                                   │
│  │  → Find bottlenecks                                                     │
│  │  → Understand distributed flows                                         │
│  │  → Debug latency issues                                                 │
│  └─────────────────────┘                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. Structured Logging

**Unstructured (Bad):**
```
2024-01-15 12:00:01 ERROR Payment failed for user 123 order 456 amount $99.99
```
Hard to parse. Hard to search. Hard to aggregate.

**Structured (Good):**
```json
{
  "timestamp": "2024-01-15T12:00:01Z",
  "level": "ERROR",
  "service": "payment-service",
  "message": "Payment failed",
  "user_id": "123",
  "order_id": "456",
  "amount": 99.99,
  "error_code": "CARD_DECLINED",
  "trace_id": "abc-123"
}
```
Queryable: `level:ERROR AND error_code:CARD_DECLINED AND amount>100`

### 3. Metrics Types

```
┌─────────────────────────────────────────────────────────────────┐
│  COUNTER - Only goes up                                         │
│  Example: http_requests_total                                   │
│  "How many requests since start?"                               │
│                                                                 │
│  ────────────────────────────────────────────────────────────  │
│                                                     ╱           │
│                                           ╱────────             │
│                             ╱────────────                       │
│              ╱─────────────                                     │
│  ───────────                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  GAUGE - Goes up and down                                       │
│  Example: memory_usage_bytes                                    │
│  "What's the current value?"                                    │
│                                                                 │
│  ────────────────────────────────────────────────────────────  │
│            ╱╲              ╱╲                                   │
│           ╱  ╲    ╱╲     ╱  ╲    ╱╲                            │
│          ╱    ╲  ╱  ╲   ╱    ╲  ╱  ╲                           │
│  ───────╱      ╲╱    ╲─╱      ╲╱    ╲───────                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  HISTOGRAM - Distribution of values                             │
│  Example: http_request_duration_seconds                         │
│  "How are response times distributed?"                          │
│                                                                 │
│  Count │                                                        │
│        │     ██                                                 │
│        │     ██ ██                                              │
│        │  ██ ██ ██                                              │
│        │  ██ ██ ██ ██                                           │
│        │  ██ ██ ██ ██ ██                                        │
│        └──────────────────                                      │
│          0.1 0.2 0.5 1.0 2.0  seconds                          │
│                                                                 │
│  p50 = 0.2s, p99 = 1.5s                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 4. SLIs, SLOs, and SLAs

```
┌─────────────────────────────────────────────────────────────────┐
│  SLI (Service Level Indicator)                                  │
│  The metric itself                                              │
│  Example: "99.5% of requests complete in < 200ms"               │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  SLO (Service Level Objective)                                  │
│  Internal target                                                │
│  Example: "We aim for 99.9% availability"                       │
└───────────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  SLA (Service Level Agreement)                                  │
│  External contract                                              │
│  Example: "We guarantee 99.5% uptime or refund"                 │
│                                                                 │
│  SLA < SLO (buffer for mistakes)                                │
└─────────────────────────────────────────────────────────────────┘
```

### 5. Alerting Strategy

**Alert on symptoms, not causes:**
```
BAD:  Alert when CPU > 80%
      (CPU might be fine at 90%, false positives)

GOOD: Alert when error rate > 1%
      Alert when p99 latency > 2 seconds
      (User-impacting symptoms)
```

**Alert severity:**
```
PAGE (Wake someone up):
  - Service down
  - Error rate > 10%
  - Payment failures

TICKET (Fix during business hours):
  - Disk space < 20%
  - Error rate 1-5%
  - Slow but working

LOG (Just record):
  - Individual request failures
  - Expected errors
```

---

## Security Essentials

### OWASP Top 10 (Most Common Attacks)

```
1. Injection (SQL, Command)     → Parameterized queries
2. Broken Authentication        → Proper session management
3. Sensitive Data Exposure      → Encryption at rest/transit
4. XML External Entities        → Disable XXE
5. Broken Access Control        → Verify permissions
6. Security Misconfiguration    → Secure defaults
7. XSS (Cross-Site Scripting)   → Escape output
8. Insecure Deserialization     → Validate input types
9. Using Vulnerable Components  → Keep dependencies updated
10. Insufficient Logging        → This whole section!
```

### Defense in Depth

```
                    ┌─────────────────────────────────┐
                    │          Internet               │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │    WAF (Web Application        │  Layer 1
                    │    Firewall)                   │
                    │    Block known attack patterns │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │    Rate Limiting               │  Layer 2
                    │    Prevent brute force         │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │    Authentication              │  Layer 3
                    │    Verify identity             │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │    Authorization               │  Layer 4
                    │    Verify permissions          │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │    Input Validation            │  Layer 5
                    │    Sanitize all data           │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │    Application Logic           │
                    └─────────────────────────────────┘
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 6 (Middleware)** | Logging/tracing middleware |
| **Section 9 (Async)** | Track job success/failure |
| **Section 12 (Distributed)** | Distributed tracing essential |
| **Section 13 (Data)** | Metrics are time-series data |
| **Section 15 (Cloud)** | Infrastructure monitoring |

---

## Observability Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON STACK                                 │
│                                                                 │
│  Logs:    ELK (Elasticsearch + Logstash + Kibana)              │
│           or Loki + Grafana                                     │
│                                                                 │
│  Metrics: Prometheus + Grafana                                  │
│           or Datadog / New Relic (SaaS)                        │
│                                                                 │
│  Traces:  Jaeger or Zipkin                                      │
│           or Datadog / Honeycomb (SaaS)                        │
│                                                                 │
│  Alerts:  PagerDuty / OpsGenie                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Implement structured logging
- [ ] Define meaningful metrics
- [ ] Set up distributed tracing
- [ ] Create effective alerts
- [ ] Design SLIs and SLOs
- [ ] Apply security best practices

---

## Seniority Challenges

### Junior Level
"What's the difference between logs, metrics, and traces?"

### Mid Level
"Design the observability strategy for an e-commerce checkout flow: what do you log, what metrics do you track, what alerts do you set up?"

### Senior Level
"We have 50 microservices with: inconsistent logging formats, no distributed tracing, metrics only on 10 services, and alerts that page 20 times/night. Design a 6-month observability improvement plan with measurable outcomes."

---

## Key Takeaways

1. **Three pillars together** - Logs, metrics, traces answer different questions
2. **Structured logging** - Machine-readable, queryable
3. **Alert on symptoms** - User-impacting issues, not CPU usage
4. **Security is observability** - Detect attacks through logs
5. **SLOs before SLAs** - Internal targets with buffer

---

## Prerequisites
- Section 6: Middleware (request context)
- Section 12: Distributed (tracing needs)

## Next Section
→ [Section 15: Cloud-Native & DevOps](./15-cloud-native.md) - Deploying and operating at scale.
