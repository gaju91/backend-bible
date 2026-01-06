# Section 4: Authentication & Authorization

> "The most dangerous question in backend engineering: 'Who sent this request?'"

---

## The Problem This Solves

Sarah's request has arrived. It's been routed. It's been parsed.

But how do we know this is actually Sarah?

Anyone can send:
```http
POST /api/orders
{"product_id": "jordan-1", "user_id": 123}
```

Without authentication, you can't trust:
- The user is who they claim to be
- They're allowed to perform this action
- The request wasn't intercepted and modified

---

## First Principles

### Principle 1: HTTP is Stateless
The server doesn't remember previous requests. Each request must prove identity independently.

This is WHY tokens exist. Not because they're trendy, but because HTTP gave us no choice.

### Principle 2: Authentication ≠ Authorization
- **Authentication:** "Who are you?" (Identity)
- **Authorization:** "What can you do?" (Permissions)

You can be authenticated (logged in) but not authorized (not allowed to delete users).

### Principle 3: Trust Must Be Verified at Every Boundary
Internal services, external APIs, user requests - all need verification. "Trust but verify" becomes "verify, then trust."

---

## Core Concepts

### 1. The Session vs Token Debate

**Session-Based (Stateful):**
```
┌──────────┐         ┌──────────┐         ┌─────────┐
│  Client  │ ──1──→  │  Server  │ ──2──→  │ Session │
│          │ ←──3──  │          │ ←──4──  │  Store  │
└──────────┘         └──────────┘         └─────────┘

1. Login with credentials
2. Create session in store (Redis/DB)
3. Return session ID (cookie)
4. Each request: lookup session by ID
```

**Pros:** Can revoke instantly, smaller cookie size
**Cons:** Requires session store, scaling complexity

**Token-Based (Stateless):**
```
┌──────────┐         ┌──────────┐
│  Client  │ ──1──→  │  Server  │
│          │ ←──2──  │          │
│          │ ──3──→  │          │  (No external store needed)
└──────────┘         └──────────┘

1. Login with credentials
2. Return signed JWT token
3. Each request: verify token signature
```

**Pros:** No session store, horizontal scaling easy
**Cons:** Can't revoke easily, larger payload

### 2. JWT (JSON Web Token) Deep Dive

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IlNhcmFoIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

│_________________│_________________________________│_____________________│
      Header                  Payload                    Signature
```

**Header:** Algorithm and token type
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload:** Claims (user data)
```json
{
  "sub": "user_123",
  "name": "Sarah",
  "role": "buyer",
  "iat": 1516239022,
  "exp": 1516242622
}
```

**Signature:** Verification
```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret_key
)
```

**Why signatures matter:** Anyone can decode the payload. Only the server with the secret can create valid signatures.

### 3. The Token Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                      TOKEN LIFECYCLE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  LOGIN                                                      │
│    │                                                        │
│    ▼                                                        │
│  ┌─────────────────┐                                       │
│  │ Access Token    │  Short-lived (15 min - 1 hour)        │
│  │ (JWT)           │  Used for API requests                │
│  └────────┬────────┘                                       │
│           │                                                 │
│           │ Expires                                         │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │ Refresh Token   │  Long-lived (7-30 days)               │
│  │ (Opaque/JWT)    │  Used only to get new access tokens   │
│  └────────┬────────┘                                       │
│           │                                                 │
│           │ Expires                                         │
│           ▼                                                 │
│      RE-LOGIN                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why two tokens?**
- Access tokens are sent with every request → if stolen, limited damage (short expiry)
- Refresh tokens are stored securely → only used occasionally

### 4. OAuth 2.0 - Delegated Authorization

When you click "Login with Google":

```
┌──────┐      ┌──────────┐      ┌──────────┐
│ User │      │ Your App │      │  Google  │
└──┬───┘      └────┬─────┘      └────┬─────┘
   │               │                  │
   │ 1. Click      │                  │
   │ "Login with   │                  │
   │  Google"      │                  │
   │──────────────→│                  │
   │               │ 2. Redirect to   │
   │               │    Google        │
   │←──────────────│──────────────────│
   │                                  │
   │ 3. Login to Google               │
   │──────────────────────────────────│
   │                                  │
   │ 4. "Allow YourApp to access      │
   │     your profile?"               │
   │──────────────────────────────────│
   │                                  │
   │ 5. Redirect back with code       │
   │←─────────────────────────────────│
   │               │                  │
   │               │ 6. Exchange code │
   │               │    for tokens    │
   │               │─────────────────→│
   │               │←─────────────────│
   │               │                  │
   │ 7. Create     │                  │
   │    session    │                  │
   │←──────────────│                  │
```

**Key insight:** Your app never sees the user's Google password. Google handles auth, you get a token.

### 5. Authorization Models

**RBAC (Role-Based Access Control):**
```
Roles:
  - admin: [create, read, update, delete]
  - editor: [create, read, update]
  - viewer: [read]

User "Sarah" has role "editor"
→ Sarah can create, read, update
→ Sarah cannot delete
```

**ABAC (Attribute-Based Access Control):**
```
Policy: "Users can edit posts they created"

Request: Sarah wants to edit Post 123
Attributes:
  - user.id = "sarah_456"
  - post.author_id = "sarah_456"
  - action = "edit"

Evaluation: user.id == post.author_id → ALLOW
```

**When to use which:**
- RBAC: Simple permission structures, clear hierarchies
- ABAC: Complex rules, dynamic conditions, fine-grained control

---

## The Chain of Necessity

```
HTTP is Stateless (Section 1)
        │
        ▼
Server can't remember users
        │
        ▼
Each request must prove identity
        │
        ▼
Tokens become necessary (JWT)
        │
        ▼
Tokens must be validated every request
        │
        ▼
Validation is expensive
        │
        ▼
Caching becomes necessary (Section 8)
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 1 (Networking)** | HTTP statelessness forces token design |
| **Section 5 (Validation)** | Auth tokens must be validated |
| **Section 6 (Middleware)** | Auth is typically first middleware |
| **Section 8 (Caching)** | Token validation results are cached |
| **Section 10 (Events)** | Auth changes trigger events (logout all devices) |
| **Section 14 (Observability)** | Auth failures need monitoring |

---

## Real-World Scenarios

### Scenario 1: The Token Revocation Problem
**Problem:** User's token is stolen. They change password. Old token still works for 1 hour.
**Solutions:**
1. Short expiry times (15 min)
2. Token blacklist (check on each request)
3. Token versioning (increment on password change)

### Scenario 2: The Timing Attack
**Problem:** Hackers measure response time for "Invalid Password" vs "Invalid User"
```
Invalid user: 50ms (no password hash check)
Invalid password: 150ms (password hash check)
```
**Solution:** Always perform password check, use constant-time comparison.

### Scenario 3: The JWT in localStorage Debate
**Problem:** Where to store JWT in browser?
- `localStorage`: Vulnerable to XSS
- `httpOnly cookie`: Vulnerable to CSRF
**Solution:** httpOnly cookie + CSRF token, or localStorage with strict CSP

---

## Security Checklist

- [ ] Passwords hashed with bcrypt/argon2 (not MD5/SHA1)
- [ ] Tokens have expiration times
- [ ] Refresh tokens stored securely
- [ ] HTTPS only (no HTTP)
- [ ] Rate limiting on login endpoints
- [ ] Account lockout after failed attempts
- [ ] Constant-time password comparison
- [ ] No sensitive data in JWT payload

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Implement JWT-based authentication
- [ ] Design OAuth 2.0 flows
- [ ] Choose between session and token approaches
- [ ] Implement RBAC/ABAC authorization
- [ ] Handle token refresh securely

---

## Seniority Challenges

### Junior Level
"What's the difference between authentication and authorization?"

### Mid Level
"Design an auth system for an app with: regular users, premium users, moderators, and admins. Users can belong to multiple organizations with different roles in each."

### Senior Level
"Our system has 50 microservices. Currently each service validates JWTs independently. How do we implement: service-to-service auth, user impersonation for support, and centralized permission management?"

---

## Key Takeaways

1. **Statelessness forces tokens** - It's physics, not preference
2. **Authentication ≠ Authorization** - Different problems, different solutions
3. **Short-lived access, long-lived refresh** - Minimize damage from stolen tokens
4. **Trust boundaries everywhere** - Internal services need auth too
5. **Security is not optional** - One breach ruins everything

---

## Prerequisites
- Section 1: Networking (HTTP, statelessness)
- Section 3: Serialization (JSON/JWT structure)

## Next Section
→ [Section 5: Validation](./05-validation.md) - We know WHO, now is their DATA safe?
