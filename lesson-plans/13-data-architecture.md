# Section 13: Data Architecture & Streaming

> "Your production database is not your analytics database. Trying to make it both will kill both."

---

## The Problem This Solves

Your product manager wants:
- "Show me sales trends for the last 90 days"
- "Which products are trending right now?"
- "What's our customer lifetime value by region?"

Running these queries on your production database:
- Locks tables
- Consumes all CPU
- Slows down user requests
- Eventually crashes

Production databases are optimized for OLTP (fast transactions). Analytics needs OLAP (fast aggregations). Different problems, different architectures.

---

## First Principles

### Principle 1: Separate OLTP from OLAP
- **OLTP** (Online Transaction Processing): Insert order, update inventory, quick lookups
- **OLAP** (Online Analytical Processing): "Total sales by region last quarter"

Mixing them degrades both.

### Principle 2: Data Should Flow, Not Be Copied
Instead of periodic dumps, stream changes in real-time. The system stays in sync automatically.

### Principle 3: Storage is Cheap, Compute is Expensive
Store raw data forever. Transform when needed. Don't pre-aggregate everything—you'll need it differently later.

---

## Core Concepts

### 1. ETL vs ELT

**ETL (Extract, Transform, Load):**
```
Source DB → [Transform in ETL Tool] → Data Warehouse
               (clean, aggregate)

Traditional approach. Transform before loading.
Problem: Changes to transformation require re-processing everything.
```

**ELT (Extract, Load, Transform):**
```
Source DB → [Load Raw] → Data Warehouse → [Transform in Warehouse]
                              (raw data)      (SQL/dbt)

Modern approach. Load raw, transform with SQL.
Benefit: Raw data preserved. Transform differently later.
```

### 2. The Data Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA ARCHITECTURE                                 │
│                                                                             │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐             │
│  │  Operational  │     │   Streaming   │     │    Data       │             │
│  │   Databases   │────→│   Platform    │────→│  Warehouse    │             │
│  │  (PostgreSQL) │     │   (Kafka)     │     │ (Snowflake)   │             │
│  └───────────────┘     └───────────────┘     └───────┬───────┘             │
│         │                      │                     │                      │
│         │ CDC                  │ Stream              │ Query                │
│         │                      │ Processing          │                      │
│         ▼                      ▼                     ▼                      │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐             │
│  │   Change      │     │    Flink /    │     │      BI       │             │
│  │   Capture     │     │    Spark      │     │    Tools      │             │
│  │  (Debezium)   │     │  (Real-time)  │     │  (Looker)     │             │
│  └───────────────┘     └───────────────┘     └───────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3. Change Data Capture (CDC)

Stream database changes without impacting production:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANGE DATA CAPTURE                          │
│                                                                 │
│  PostgreSQL                    Kafka                           │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │ users table     │         │ users-changes   │               │
│  │                 │         │                 │               │
│  │ INSERT user A ──┼────────→│ {op: INSERT,    │               │
│  │                 │         │  after: {A}}    │               │
│  │                 │         │                 │               │
│  │ UPDATE user A ──┼────────→│ {op: UPDATE,    │               │
│  │                 │         │  before: {A},   │               │
│  │                 │         │  after: {A'}}   │               │
│  │                 │         │                 │               │
│  │ DELETE user A ──┼────────→│ {op: DELETE,    │               │
│  │                 │         │  before: {A}}   │               │
│  └─────────────────┘         └─────────────────┘               │
│                                                                 │
│  Production DB doesn't know CDC exists!                         │
│  Reads transaction log (WAL) instead of querying tables.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Batch vs Stream Processing

**Batch Processing:**
```
Input: All orders from yesterday
Process: Aggregate by region, calculate totals
Output: Daily sales report

Latency: Hours (runs once per day)
Use for: Historical reports, ML training
Tools: Spark, Hadoop, SQL batch jobs
```

**Stream Processing:**
```
Input: Each order as it happens
Process: Update running totals, detect patterns
Output: Real-time dashboard

Latency: Seconds/minutes
Use for: Real-time analytics, fraud detection
Tools: Kafka Streams, Flink, Spark Streaming
```

### 5. Data Warehouse vs Data Lake

**Data Warehouse:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Structured, Schema-on-Write                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  fact_orders                                            │   │
│  │  ├── order_id (INT)                                     │   │
│  │  ├── user_id (INT)                                      │   │
│  │  ├── total (DECIMAL)                                    │   │
│  │  └── created_at (TIMESTAMP)                             │   │
│  │                                                         │   │
│  │  dim_users                                              │   │
│  │  ├── user_id (INT)                                      │   │
│  │  ├── name (VARCHAR)                                     │   │
│  │  └── region (VARCHAR)                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Best for: SQL analytics, BI dashboards                         │
│  Tools: Snowflake, BigQuery, Redshift                          │
└─────────────────────────────────────────────────────────────────┘
```

**Data Lake:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Any format, Schema-on-Read                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  s3://data-lake/                                        │   │
│  │  ├── raw/                                               │   │
│  │  │   ├── orders/2024-01-15/orders.parquet              │   │
│  │  │   ├── logs/2024-01-15/app.log.gz                    │   │
│  │  │   └── images/products/*.jpg                         │   │
│  │  └── processed/                                         │   │
│  │      └── ml-features/user_vectors.parquet              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Best for: Raw data storage, ML, diverse formats               │
│  Tools: S3/GCS + Delta Lake/Iceberg                            │
└─────────────────────────────────────────────────────────────────┘
```

### 6. Lambda Architecture

Combine batch and stream for complete picture:

```
                    ┌─────────────────────────────────────────────┐
                    │                RAW DATA                     │
                    └───────────────────┬─────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   │                   ▼
          ┌─────────────────┐          │         ┌─────────────────┐
          │   BATCH LAYER   │          │         │  SPEED LAYER    │
          │                 │          │         │                 │
          │ - Full history  │          │         │ - Real-time     │
          │ - Complete but  │          │         │ - Fast but      │
          │   hours old     │          │         │   incomplete    │
          │                 │          │         │                 │
          └────────┬────────┘          │         └────────┬────────┘
                   │                   │                  │
                   └───────────────────┼──────────────────┘
                                       │
                                       ▼
                            ┌─────────────────┐
                            │  SERVING LAYER  │
                            │                 │
                            │  Merge batch +  │
                            │  stream results │
                            │                 │
                            └─────────────────┘
```

---

## Time-Series Data

Special patterns for time-based data (metrics, IoT, logs):

```sql
-- Regular table: Full scan for time range
SELECT avg(temperature) FROM readings
WHERE timestamp > now() - interval '1 day';

-- Time-series optimized: Partition by time
CREATE TABLE readings (
    timestamp TIMESTAMPTZ,
    sensor_id INT,
    temperature FLOAT
) PARTITION BY RANGE (timestamp);

-- Automatic partition pruning: Only scans today's partition
```

**Tools:** TimescaleDB, InfluxDB, QuestDB

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 8 (Database)** | Source of CDC streams |
| **Section 10 (Events)** | Events feed data pipelines |
| **Section 12 (Distributed)** | Stream processing is distributed |
| **Section 14 (Observability)** | Metrics/logs are time-series data |

---

## Real-World Scenarios

### Scenario 1: The Analytics Query That Killed Production
**What happened:** PM ran "SELECT * FROM orders JOIN users JOIN products" on production for a report. 3-hour query locked tables, site went down.
**Fix:** Replicate to analytics database. Run reports there.

### Scenario 2: The Stale Dashboard
**What happened:** Dashboard shows sales from 6 hours ago. CEO wants real-time.
**Root cause:** ETL job runs every 6 hours.
**Fix:** CDC + stream processing. Dashboard updates in seconds.

### Scenario 3: The Data Scientist's Disaster
**What happened:** Data scientist deleted production data "by accident" (had write access).
**Fix:** Read replicas for analytics. No write access. Ever.

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design OLTP vs OLAP separation
- [ ] Implement CDC pipelines
- [ ] Choose batch vs stream processing
- [ ] Build data warehouse schemas
- [ ] Handle time-series data

---

## Seniority Challenges

### Junior Level
"Why can't we just run analytics queries on the production database?"

### Mid Level
"Design a pipeline that: captures order changes in real-time, aggregates them for dashboards, and stores raw data for historical analysis. What tools would you use?"

### Senior Level
"We have 50TB of historical data in PostgreSQL, 10M new events/day, and need: real-time dashboards, ML training data, and 7-year retention for compliance. Current ETL takes 12 hours. Design the migration to a modern data architecture."

---

## Key Takeaways

1. **OLTP ≠ OLAP** - Different workloads, different databases
2. **CDC > Bulk exports** - Real-time, no impact on production
3. **Store raw, transform later** - You'll need it differently
4. **Batch + Stream = Complete** - Lambda architecture
5. **Data lakes for variety** - Not everything is structured

---

## Prerequisites
- Section 8: Database (understanding production DBs)
- Section 10: Events (streaming foundation)

## Next Section
→ [Section 14: Observability](./14-observability.md) - You can't fix what you can't see.
