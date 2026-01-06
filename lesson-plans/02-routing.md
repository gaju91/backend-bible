# Section 2: Routing Architecture

> "A URL is not just an address - it's a contract with your users."

---

## The Problem This Solves

Raw HTTP bytes have arrived at your server. Now what?

```
POST /api/v2/users/123/orders HTTP/1.1
```

Your server needs to answer:
- Which controller handles this?
- What does `123` mean?
- Is `v2` different from `v1`?
- Should `POST` and `GET` go to the same place?

Bad routing design leads to:
- APIs that can't evolve without breaking clients
- Confusing URL structures that developers hate
- Performance issues from inefficient route matching

---

## First Principles

### Principle 1: URLs Are Your Public Interface
Your code can change. Your URLs are promised to the world.

### Principle 2: Routes Map Intentions to Handlers
`GET /users` = "I want to read users"
`POST /users` = "I want to create a user"
Same URL, different intentions.

### Principle 3: Route Design Reveals Domain Model
Your URLs expose how you think about your business:
- `/orders/123/items` reveals orders contain items
- `/users/456/settings` reveals users have settings

---

## Core Concepts

### 1. HTTP Methods as Intentions

| Method | Intention | Idempotent? | Safe? |
|--------|-----------|-------------|-------|
| GET | Read data | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

**Why idempotency matters:** If a request fails and retries, will it cause problems?
- `DELETE /orders/123` - Safe to retry (order is deleted either way)
- `POST /orders` - Dangerous to retry (might create duplicate orders)

### 2. URL Structure Patterns

```
/api/v2/users/123/orders/456/items

 │   │    │    │    │     │    │
 │   │    │    │    │     │    └── Collection: items
 │   │    │    │    │     └── Resource ID: 456
 │   │    │    │    └── Sub-collection: orders
 │   │    │    └── Resource ID: 123
 │   │    └── Collection: users
 │   └── Version: v2
 └── API prefix
```

### 3. Path Parameters vs Query Parameters

**Path Parameters** - Identify the resource:
```
GET /users/123          ← User 123
GET /orders/456/items   ← Items in order 456
```

**Query Parameters** - Filter/modify the response:
```
GET /users?role=admin&active=true
GET /products?sort=price&order=desc&limit=20
```

**Rule of thumb:** If it identifies WHAT you're getting, use path. If it modifies HOW you get it, use query.

### 4. API Versioning Strategies

**URL Path Versioning:**
```
/api/v1/users
/api/v2/users
```
Pros: Clear, easy to route
Cons: "Ugly" URLs, version proliferation

**Header Versioning:**
```
GET /api/users
Accept: application/vnd.myapi.v2+json
```
Pros: Clean URLs
Cons: Harder to test, hidden versioning

**Query Parameter:**
```
GET /api/users?version=2
```
Pros: Simple
Cons: Cacheing complications

**Recommendation:** URL path versioning. Clarity > elegance.

### 5. Route Organization

```
/api
├── /v1
│   ├── /auth
│   │   ├── POST /login
│   │   ├── POST /logout
│   │   └── POST /refresh
│   ├── /users
│   │   ├── GET /           (list)
│   │   ├── POST /          (create)
│   │   ├── GET /:id        (read)
│   │   ├── PUT /:id        (update)
│   │   └── DELETE /:id     (delete)
│   └── /orders
│       ├── GET /
│       ├── POST /
│       └── /:id
│           ├── GET /
│           ├── /items
│           │   ├── GET /
│           │   └── POST /
│           └── /status
│               └── PATCH /
```

---

## Route Matching Internals

How does the router find the right handler?

### Simple Linear Search (Slow)
```python
routes = [
    ("/users", handler1),
    ("/users/:id", handler2),
    ("/orders", handler3),
]
# Check each route until match - O(n)
```

### Trie-Based Routing (Fast)
```
        /
       api
      /   \
   users  orders
   /   \
  :id  (list)
```
Lookup is O(path_length), not O(num_routes).

**Why it matters:** At 1000+ routes, linear search adds milliseconds. Trie stays constant.

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 1 (Networking)** | Routes receive requests that networking delivered |
| **Section 3 (Serialization)** | Route handlers receive parsed data |
| **Section 6 (Middleware)** | Routes are wrapped by middleware pipelines |
| **Section 7 (Controllers)** | Routes point to controller methods |
| **Section 11 (APIs)** | GraphQL uses single route, gRPC uses service definitions |
| **Section 12 (Distributed)** | API Gateway routes to microservices |

---

## Real-World Scenarios

### Scenario 1: The Breaking Change
**Problem:** You renamed `/users/:id/profile` to `/users/:id/settings`. Apps broke.
**Lesson:** URLs are contracts. Add new routes, deprecate old ones with redirects.

### Scenario 2: The Versioning Nightmare
**Problem:** v1, v2, v3, v4... 4 versions to maintain, bugs to fix in all.
**Solution:** Sunset policy. v1 deprecated after 12 months. Max 2 active versions.

### Scenario 3: The Nested Resource Debate
**Problem:** Team argues about `/users/123/orders` vs `/orders?user_id=123`
**Resolution:**
- Use nested when relationship is ownership (user HAS orders)
- Use query params when it's just filtering

---

## URL Design Guidelines

### Do This
```
GET  /api/v1/users              # List users
GET  /api/v1/users/123          # Get user 123
POST /api/v1/users              # Create user
GET  /api/v1/users/123/orders   # User's orders
```

### Don't Do This
```
GET  /api/v1/getUsers           # Verb in URL (GET already says it)
GET  /api/v1/user/123           # Singular (use plural)
POST /api/v1/users/create       # Redundant (POST already says create)
GET  /api/v1/users/123/getOrders # Double verbs
```

### Naming Conventions
- Use **kebab-case**: `/user-profiles` not `/userProfiles`
- Use **plurals**: `/users` not `/user`
- Use **nouns**: `/orders` not `/order-creation`
- Be **consistent**: Don't mix `/users` and `/product`

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design RESTful URL structures
- [ ] Choose appropriate versioning strategy
- [ ] Organize routes for large applications
- [ ] Understand route matching performance
- [ ] Handle backward compatibility

---

## Seniority Challenges

### Junior Level
"What's the difference between PUT and PATCH?"

### Mid Level
"Design the URL structure for an e-commerce API with products, categories, users, orders, and reviews."

### Senior Level
"Our API has 500 endpoints across v1-v4. How do we consolidate without breaking existing clients? What's the migration strategy?"

---

## Key Takeaways

1. **URLs are contracts** - Breaking them breaks trust
2. **HTTP methods have meaning** - Use them correctly
3. **Path = identity, Query = modifiers** - Keep this distinction
4. **Version explicitly** - URL versioning is clearest
5. **Design for evolution** - Today's `/users` is tomorrow's legacy

---

## Prerequisites
- Section 1: Networking (understand HTTP basics)

## Next Section
→ [Section 3: Serialization](./03-serialization.md) - Route matched, now let's parse that data.
