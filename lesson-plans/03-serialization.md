# Section 3: Serialization

> "Data is useless until it crosses the boundary between text and meaning."

---

## The Problem This Solves

Sarah's browser sent this:
```
{"product_id": "jordan-1", "size": 9, "quantity": 1}
```

Your Go/Python/Node server sees... bytes. Just bytes.

```
7b 22 70 72 6f 64 75 63 74 5f 69 64 22 3a 20 22 6a 6f 72 64 61 6e ...
```

**Serialization** is the bridge between human-readable formats and machine-processable objects.

Without understanding this:
- You'll choose wrong formats (JSON when you need Protobuf)
- You'll break clients when schemas evolve
- You'll waste bandwidth and CPU on inefficient parsing

---

## First Principles

### Principle 1: Data Must Cross Language Boundaries
JavaScript client → JSON → Python server → Protocol Buffers → Go microservice

Each boundary needs translation.

### Principle 2: There's No Free Lunch
- Human readable (JSON) = Larger size, slower parsing
- Binary efficient (Protobuf) = Smaller size, faster parsing, harder debugging

### Principle 3: Schemas Change Over Time
Version 1 has `user_name`. Version 2 splits into `first_name` and `last_name`. How do you not break existing clients?

---

## Core Concepts

### 1. Serialization vs Deserialization

```
SERIALIZATION (Object → Bytes)
┌──────────────┐      ┌─────────────┐
│ Python Object│  →   │ JSON String │
│ {"size": 9}  │      │ '{"size":9}'│
└──────────────┘      └─────────────┘

DESERIALIZATION (Bytes → Object)
┌─────────────┐      ┌──────────────┐
│ JSON String │  →   │ Python Object│
│ '{"size":9}'│      │ {"size": 9}  │
└─────────────┘      └──────────────┘
```

### 2. Format Comparison

| Format | Type | Size | Parse Speed | Human Readable | Schema |
|--------|------|------|-------------|----------------|--------|
| JSON | Text | Large | Slow | Yes | Optional |
| Protocol Buffers | Binary | Small | Fast | No | Required |
| MessagePack | Binary | Medium | Fast | No | Optional |
| XML | Text | Largest | Slowest | Yes | Optional |
| YAML | Text | Large | Slow | Very | Optional |

### 3. JSON - The Universal Format

```json
{
  "user": {
    "id": 123,
    "name": "Sarah",
    "email": "sarah@example.com",
    "roles": ["buyer", "reviewer"],
    "active": true,
    "balance": 99.50
  }
}
```

**Pros:**
- Human readable
- Universally supported
- Easy debugging
- Self-describing

**Cons:**
- Verbose (field names repeated)
- Slow parsing
- No native binary support
- No schema enforcement

**Use when:** Client-facing APIs, debugging, configuration files

### 4. Protocol Buffers - The Efficient Format

```protobuf
// user.proto
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  repeated string roles = 4;
  bool active = 5;
  float balance = 6;
}
```

**Wire format:** Just field numbers and values, no field names.
```
08 7b 12 05 53 61 72 61 68 ...
│  │  │  │  └─ "Sarah" (5 bytes)
│  │  │  └─ length prefix
│  │  └─ field 2 (name), type string
│  └─ value 123
└─ field 1 (id), type varint
```

**Pros:**
- 3-10x smaller than JSON
- 20-100x faster parsing
- Strong typing
- Backward/forward compatible

**Cons:**
- Not human readable
- Requires schema compilation
- Debugging harder

**Use when:** Service-to-service communication, high-throughput systems

### 5. Schema Evolution - The Hard Problem

**The scenario:**
```
v1: { "name": "Sarah Connor" }
v2: { "first_name": "Sarah", "last_name": "Connor" }
```

How do you deploy v2 servers while v1 clients still exist?

**Strategies:**

**Additive Changes (Safe):**
```json
// v1
{ "name": "Sarah" }

// v2 - Added field, v1 clients ignore it
{ "name": "Sarah", "email": "sarah@example.com" }
```

**Field Deprecation (Safe with care):**
```json
// v1
{ "name": "Sarah" }

// v2 - Keep old field, add new ones
{
  "name": "Sarah",           // Deprecated, still populated
  "first_name": "Sarah",     // New
  "last_name": "Connor"      // New
}
```

**Breaking Changes (Dangerous):**
```json
// v1
{ "name": "Sarah" }

// v2 - Removed field, v1 clients BREAK
{ "full_name": "Sarah Connor" }  // name is gone!
```

**Rule:** Add fields freely. Never remove or rename without versioning.

---

## Content Negotiation

How client and server agree on format:

```http
# Client says "I want JSON"
GET /api/users/123
Accept: application/json

# Server responds with JSON
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 123, "name": "Sarah"}
```

```http
# Client says "I want Protobuf"
GET /api/users/123
Accept: application/x-protobuf

# Server responds with binary
HTTP/1.1 200 OK
Content-Type: application/x-protobuf

<binary data>
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 2 (Routing)** | Routing delivers raw bytes, serialization parses them |
| **Section 5 (Validation)** | Deserialized data must be validated |
| **Section 8 (Database)** | ORMs serialize objects to SQL and back |
| **Section 10 (Events)** | Events are serialized for queue transport |
| **Section 11 (APIs)** | gRPC uses Protobuf, GraphQL uses JSON |
| **Section 13 (Data)** | Data pipelines need efficient serialization |

---

## Real-World Scenarios

### Scenario 1: The Payload Explosion
**Problem:** Mobile app uses 10GB/month of data on your API.
**Diagnosis:** JSON responses average 50KB each.
**Solution:** Switch to Protobuf for heavy endpoints. Same data in 8KB.

### Scenario 2: The Breaking Deployment
**Problem:** New server deployment crashes all mobile apps.
**Root Cause:** Renamed `user_id` to `userId` in JSON response.
**Lesson:** Serialization changes are API changes. Version them.

### Scenario 3: The Debugging Nightmare
**Problem:** "What's in this Protobuf message?" - Developer at 3 AM
**Solution:** Keep JSON endpoints for debugging. Use Protobuf for production throughput.

---

## Performance Numbers

Parsing 1MB of data (approximate):

| Format | Parse Time | Memory |
|--------|------------|--------|
| JSON | 50-100ms | 3-5MB |
| Protobuf | 2-5ms | 1-2MB |
| MessagePack | 10-20ms | 2-3MB |

**At scale:** 1M requests/day × 50ms savings = 14 hours of CPU saved daily.

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Choose appropriate serialization format for use case
- [ ] Design schemas that evolve without breaking clients
- [ ] Implement content negotiation
- [ ] Debug serialization issues
- [ ] Optimize payload sizes

---

## Seniority Challenges

### Junior Level
"What's the difference between JSON and XML? When would you use each?"

### Mid Level
"Our API sends user data including a `status` field that's currently a string. We need to change it to an object with `code` and `message`. How do we do this without breaking existing clients?"

### Senior Level
"Design a serialization strategy for a system that needs to support: browser clients (JS), mobile apps (iOS/Android), internal microservices, and third-party integrations. Consider performance, debugging, and schema evolution."

---

## Key Takeaways

1. **Format choice is a trade-off** - Readability vs performance
2. **JSON for humans, Protobuf for machines** - Use both where appropriate
3. **Schemas WILL change** - Design for evolution from day one
4. **Never remove fields** - Deprecate and maintain
5. **Content negotiation** - Let clients choose their format

---

## Prerequisites
- Section 1: Networking (HTTP basics)
- Section 2: Routing (request handling)

## Next Section
→ [Section 4: Authentication](./04-authentication.md) - Data is parsed, now who sent it?
