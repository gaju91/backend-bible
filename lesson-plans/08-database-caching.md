# Section 8: Databases & Caching

> "The database is where optimism goes to die. Concurrent users, limited resources, physics - this is where reality hits."

---

## The Problem This Solves

Sarah clicks "Buy Now." Jake clicks "Buy Now." Same millisecond. One item left.

Your business logic says "check inventory, then decrement." But between check and decrement, both see `quantity: 1`.

Result: You sold the same item twice. Lawsuit incoming.

This section is about:
- Making data operations reliable (ACID)
- Making data access fast (Caching)
- Making concurrent access correct (Locking)

---

## First Principles

### Principle 1: Disk is Slow, Network is Slower
```
RAM access:     100 nanoseconds
SSD read:       150,000 nanoseconds (1,500x slower)
Network call:   500,000 nanoseconds (5,000x slower)
```
Every database query is expensive. This drives caching.

### Principle 2: Concurrent Access Requires Coordination
Two operations reading and writing the same data must be coordinated, or corruption happens.

### Principle 3: Durability Has a Cost
Writing to disk (durability) is slow. Keeping data in memory is fast but risky. Trade-offs everywhere.

---

## Core Concepts: ACID

### Atomicity
"All or nothing" - A transaction either fully completes or fully rolls back.

```sql
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE id = 'sarah';
    UPDATE accounts SET balance = balance + 100 WHERE id = 'jake';
COMMIT;

-- If second UPDATE fails, first UPDATE is rolled back
-- Sarah doesn't lose money without Jake receiving it
```

### Consistency
Database moves from one valid state to another. Constraints are always satisfied.

```sql
-- Constraint: balance >= 0
UPDATE accounts SET balance = balance - 1000 WHERE id = 'sarah';
-- Fails if Sarah only has $500 - consistency maintained
```

### Isolation
Concurrent transactions don't interfere with each other.

```
Transaction A reads balance: $500
Transaction B reads balance: $500
Transaction A withdraws $500 (balance now $0)
Transaction B withdraws $500...

Without isolation: Balance = -$500 (bad!)
With isolation: Transaction B waits, sees $0, fails gracefully
```

### Durability
Once committed, data survives crashes.

```
1. Transaction commits
2. Server crashes 1 second later
3. Server restarts
4. Data is still there ✓
```

---

## The Race Condition Problem

### Without Proper Locking

```python
# BAD: Race condition
def purchase(user_id, product_id):
    product = db.query("SELECT quantity FROM products WHERE id = ?", product_id)

    if product.quantity > 0:  # Sarah sees 1, Jake sees 1
        # Time passes... context switch...
        db.execute("UPDATE products SET quantity = quantity - 1 WHERE id = ?", product_id)
        # Both succeed! Quantity goes to -1
```

### With Optimistic Locking

```python
# BETTER: Optimistic locking with version
def purchase(user_id, product_id):
    product = db.query("SELECT quantity, version FROM products WHERE id = ?", product_id)

    if product.quantity > 0:
        result = db.execute("""
            UPDATE products
            SET quantity = quantity - 1, version = version + 1
            WHERE id = ? AND version = ?
        """, product_id, product.version)

        if result.rows_affected == 0:
            raise ConcurrentModificationError()  # Retry!
```

### With Pessimistic Locking

```python
# BEST for high contention: Pessimistic locking
def purchase(user_id, product_id):
    # SELECT ... FOR UPDATE acquires row lock
    product = db.query("""
        SELECT quantity FROM products
        WHERE id = ?
        FOR UPDATE
    """, product_id)  # Jake's query blocks here until Sarah commits

    if product.quantity > 0:
        db.execute("UPDATE products SET quantity = quantity - 1 WHERE id = ?", product_id)
    else:
        raise OutOfStockError()
```

---

## Query Optimization

### The N+1 Problem

```python
# BAD: N+1 queries
users = db.query("SELECT * FROM users LIMIT 100")  # 1 query
for user in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)  # 100 queries
# Total: 101 queries
```

```python
# GOOD: Eager loading
users = db.query("""
    SELECT users.*, orders.*
    FROM users
    LEFT JOIN orders ON orders.user_id = users.id
    LIMIT 100
""")  # 1 query
# Total: 1 query
```

### Index Basics

```sql
-- Without index: Full table scan (O(n))
SELECT * FROM orders WHERE user_id = 'sarah_123';
-- Scans 1,000,000 rows to find 50 matches

-- With index: B-tree lookup (O(log n))
CREATE INDEX idx_orders_user_id ON orders(user_id);
SELECT * FROM orders WHERE user_id = 'sarah_123';
-- Jumps directly to Sarah's orders
```

**When to index:**
- Columns in WHERE clauses
- Columns in JOIN conditions
- Columns in ORDER BY

**When NOT to index:**
- Tables with few rows
- Columns with low cardinality (e.g., boolean)
- Columns that change frequently (index maintenance cost)

### Explain Your Queries

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 'sarah_123';

-- Output shows:
-- Index Scan vs Seq Scan (index scan = good)
-- Estimated rows vs actual rows (big difference = stale statistics)
-- Actual time (the real performance)
```

---

## Caching

### The Cache Hierarchy

```
Request
   │
   ▼
┌─────────────────┐
│ Application     │  In-process cache (HashMap)
│ Memory Cache    │  Latency: ~1μs
│ (Hot data)      │  Size: Limited by app memory
└────────┬────────┘
         │ Miss
         ▼
┌─────────────────┐
│ Distributed     │  Redis / Memcached
│ Cache           │  Latency: ~1ms
│ (Warm data)     │  Size: Large, shared across instances
└────────┬────────┘
         │ Miss
         ▼
┌─────────────────┐
│ Database        │  PostgreSQL / MySQL
│                 │  Latency: ~10ms
│                 │  Size: Unlimited (disk)
└─────────────────┘
```

### Caching Strategies

**Cache-Aside (Lazy Loading):**
```python
def get_user(user_id):
    # Try cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return deserialize(cached)

    # Cache miss - go to database
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)

    # Store in cache for next time
    redis.setex(f"user:{user_id}", 3600, serialize(user))  # 1 hour TTL

    return user
```

**Write-Through:**
```python
def update_user(user_id, data):
    # Update database
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)

    # Update cache immediately
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(f"user:{user_id}", 3600, serialize(user))
```

**Write-Behind (Async):**
```python
def update_user(user_id, data):
    # Update cache immediately (fast response)
    redis.setex(f"user:{user_id}", 3600, serialize(data))

    # Queue database write (happens later)
    queue.publish("user_updates", {"user_id": user_id, "data": data})
```

### Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things."

**Time-Based (TTL):**
```python
redis.setex("product:123", 300, data)  # Expires in 5 minutes
# Simple, but data might be stale for up to 5 minutes
```

**Event-Based:**
```python
def update_product(product_id, data):
    db.update(product_id, data)
    redis.delete(f"product:{product_id}")  # Invalidate immediately
    publish_event("product.updated", product_id)  # Notify other services
```

**Version-Based:**
```python
def get_product(product_id):
    version = redis.get(f"product:{product_id}:version")
    cached = redis.get(f"product:{product_id}:v{version}")
    if cached:
        return cached

    # Fetch fresh, increment version
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    new_version = int(version or 0) + 1
    redis.set(f"product:{product_id}:version", new_version)
    redis.set(f"product:{product_id}:v{new_version}", serialize(product))
```

---

## Cache Failure Modes

### Cache Stampede

```
Cache expires at 12:00:00
→ 1000 concurrent requests hit at 12:00:01
→ All see cache miss
→ All query database simultaneously
→ Database overwhelmed
→ System crash
```

**Solution: Mutex lock**
```python
def get_with_lock(key):
    value = redis.get(key)
    if value:
        return value

    # Try to acquire lock
    lock_acquired = redis.setnx(f"lock:{key}", "1")
    if lock_acquired:
        redis.expire(f"lock:{key}", 10)  # Lock timeout
        try:
            value = expensive_database_query()
            redis.setex(key, 3600, value)
            return value
        finally:
            redis.delete(f"lock:{key}")
    else:
        # Someone else is rebuilding, wait and retry
        time.sleep(0.1)
        return get_with_lock(key)
```

### Cache Avalanche

```
All cache keys set with same TTL (1 hour)
→ After 1 hour, ALL keys expire simultaneously
→ Massive database load
→ System crash
```

**Solution: Jittered TTL**
```python
base_ttl = 3600  # 1 hour
jitter = random.randint(0, 300)  # 0-5 minutes random
redis.setex(key, base_ttl + jitter, value)
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 1 (Networking)** | Latency numbers drive caching decisions |
| **Section 7 (Controllers)** | Repositories abstract database access |
| **Section 9 (Async)** | Write-behind caching uses async |
| **Section 10 (Events)** | Cache invalidation via events |
| **Section 12 (Distributed)** | Distributed caching across services |
| **Section 14 (Observability)** | Cache hit rates need monitoring |

---

## Real-World Scenarios

### Scenario 1: The $200K Migration
**Problem:** App slow. Team migrates from Node.js to Go. Still slow.
**Root Cause:** Missing database index on frequently queried column.
**Lesson:** Profile before rewriting. The database is usually the bottleneck.

### Scenario 2: The Cache That Killed
**Problem:** Redis goes down for 5 minutes. Entire system crashes.
**Root Cause:** Database couldn't handle 100x traffic spike without cache.
**Fix:** Circuit breaker, cache warming, database connection pooling.

### Scenario 3: The Stale Data Disaster
**Problem:** User changes email, but old email keeps appearing.
**Root Cause:** Cache-aside without invalidation on update.
**Fix:** Write-through or event-based invalidation.

---

## Database Selection Guide

| Use Case | Database | Why |
|----------|----------|-----|
| Transactions, relationships | PostgreSQL, MySQL | ACID, SQL joins |
| Document storage | MongoDB | Flexible schema, nested data |
| Caching, sessions | Redis | In-memory, fast |
| Time-series | TimescaleDB, InfluxDB | Optimized for time queries |
| Full-text search | Elasticsearch | Inverted index |
| Graph relationships | Neo4j | Traversal performance |

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design database schemas with proper indexing
- [ ] Implement ACID transactions correctly
- [ ] Handle concurrent access with locking
- [ ] Choose appropriate caching strategies
- [ ] Debug performance issues with EXPLAIN

---

## Seniority Challenges

### Junior Level
"What does ACID stand for and why does it matter?"

### Mid Level
"Design the database schema and caching strategy for a product catalog with: 1M products, frequent reads, occasional updates, and flash sales that spike read traffic 100x."

### Senior Level
"Our main database is at 80% CPU during peak hours. We have 50M rows in the orders table, growing 1M/month. Reads are 95% of traffic. Design a scaling strategy that doesn't require application changes."

---

## Key Takeaways

1. **Database is the bottleneck** - Optimize here first
2. **ACID prevents corruption** - But has performance cost
3. **Locking prevents race conditions** - Choose optimistic vs pessimistic wisely
4. **Cache everything that's read-heavy** - But have invalidation strategy
5. **Cache failures cascade** - Design for cache being unavailable

---

## Prerequisites
- Section 7: Controllers (understanding data flow)

## Next Section
→ [Section 9: Asynchronous Systems](./09-async-systems.md) - Data is saved, now how do we respond fast?
