# Section 6: Middleware Pipeline

> "The orchestra conductor doesn't play an instrument - they ensure everyone plays at the right time, in the right order."

---

## The Problem This Solves

You've built:
- Authentication (Section 4)
- Validation (Section 5)
- Logging
- Rate limiting
- Error handling
- Request tracing

Now every endpoint needs ALL of these. Do you copy-paste into every controller?

```python
# Without middleware - copy-paste nightmare
def get_user(request):
    log_request(request)           # Logging
    check_rate_limit(request)      # Rate limiting
    user = authenticate(request)   # Auth
    validate_input(request)        # Validation
    trace_id = start_trace()       # Tracing
    try:
        result = actual_business_logic()  # THE ACTUAL WORK
        log_response(result)
        return result
    except Exception as e:
        handle_error(e)            # Error handling

def get_orders(request):
    log_request(request)           # Same thing again...
    check_rate_limit(request)
    user = authenticate(request)
    # ... you get the point
```

Middleware solves this: define once, apply everywhere.

---

## First Principles

### Principle 1: Cross-Cutting Concerns Need Central Management
Logging, auth, validation - these apply to EVERY request. They shouldn't be in business logic.

### Principle 2: Order Matters
Rate limit → Auth → Validation → Business Logic

Why? Rate limit first (block attacks early), auth second (why validate if unauthorized?), business logic last.

### Principle 3: Each Middleware Should Do One Thing
Auth middleware doesn't validate. Logging middleware doesn't rate limit. Single responsibility.

---

## Core Concepts

### 1. The Middleware Pipeline

```
HTTP Request
      │
      ▼
┌───────────────────────────────────────────────────────────────┐
│                    MIDDLEWARE PIPELINE                        │
│                                                               │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐       │
│  │ Logger  │ → │  Auth   │ → │Validate │ → │  Rate   │ → ... │
│  │         │   │         │   │         │   │  Limit  │       │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘       │
│       │             │             │             │             │
│       │             │             │             │             │
│  ┌────┴────────────┴─────────────┴─────────────┴────┐        │
│  │              REQUEST CONTEXT                      │        │
│  │  • request_id: "abc-123"                          │        │
│  │  • user: User(id=456)                             │        │
│  │  • start_time: 1234567890                         │        │
│  │  • validated_data: {...}                          │        │
│  └───────────────────────────────────────────────────┘        │
│                                                               │
└───────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────┐
│ Controller  │ ← Only business logic, everything else handled
└─────────────┘
      │
      ▼
┌───────────────────────────────────────────────────────────────┐
│                 RESPONSE MIDDLEWARE                           │
│  (Same pipeline in reverse for response processing)           │
└───────────────────────────────────────────────────────────────┘
      │
      ▼
HTTP Response
```

### 2. Middleware Patterns

**Onion Model (Most Common):**
```
Request enters from outside, passes through each layer
Response exits from inside, passes through each layer in reverse

        ┌───────────────────────────────────────┐
        │              Logger                   │
        │    ┌───────────────────────────┐     │
        │    │          Auth             │     │
        │    │    ┌───────────────┐     │     │
        │    │    │   Validate    │     │     │
        │    │    │    ┌─────┐    │     │     │
Request →    →    →    │Route│    ←    ←    ← Response
        │    │    │    │r    │    │     │     │
        │    │    │    └─────┘    │     │     │
        │    │    └───────────────┘     │     │
        │    └───────────────────────────┘     │
        └───────────────────────────────────────┘
```

**Chain of Responsibility:**
```python
def middleware_1(request, next):
    # Pre-processing
    log.info(f"Request started: {request.path}")

    response = next(request)  # Call next middleware

    # Post-processing
    log.info(f"Request completed: {response.status}")
    return response

def middleware_2(request, next):
    user = authenticate(request)
    request.user = user
    return next(request)

# Pipeline: middleware_1 → middleware_2 → ... → handler
```

### 3. Common Middleware Types

**Request Lifecycle Middleware:**
```
┌─────────────────────────────────────────────────────────────┐
│  Request ID Generation                                      │
│  • Assigns unique ID to each request                        │
│  • Enables distributed tracing                              │
│  • Example: X-Request-ID: "550e8400-e29b-41d4-a716-446655440000"
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Request Timing                                             │
│  • Records start time                                       │
│  • Calculates duration on response                          │
│  • Feeds into metrics (Section 14)                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Request Logging                                            │
│  • Logs method, path, headers                               │
│  • Logs response status, duration                           │
│  • Structured format for analysis                           │
└─────────────────────────────────────────────────────────────┘
```

**Security Middleware:**
```
┌─────────────────────────────────────────────────────────────┐
│  Authentication                                             │
│  • Extracts token from header/cookie                        │
│  • Validates token                                          │
│  • Attaches user to request context                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Authorization                                              │
│  • Checks user permissions                                  │
│  • Route-specific access control                            │
│  • Returns 403 if unauthorized                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Rate Limiting                                              │
│  • Tracks requests per user/IP                              │
│  • Returns 429 if limit exceeded                            │
│  • Prevents abuse and DoS                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  CORS (Cross-Origin Resource Sharing)                       │
│  • Handles preflight OPTIONS requests                       │
│  • Sets appropriate headers                                 │
│  • Controls which origins can access API                    │
└─────────────────────────────────────────────────────────────┘
```

**Data Middleware:**
```
┌─────────────────────────────────────────────────────────────┐
│  Body Parser                                                │
│  • Parses JSON/XML/form data                                │
│  • Handles content-type negotiation                         │
│  • Size limits enforcement                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Compression                                                │
│  • Compresses response body (gzip, brotli)                  │
│  • Checks Accept-Encoding header                            │
│  • Reduces bandwidth usage                                  │
└─────────────────────────────────────────────────────────────┘
```

### 4. Request Context

The request context is how middleware communicates with each other and with handlers:

```python
class RequestContext:
    request_id: str        # Set by Request ID middleware
    user: User            # Set by Auth middleware
    start_time: float     # Set by Timing middleware
    validated_data: dict  # Set by Validation middleware
    trace_span: Span      # Set by Tracing middleware

# Middleware can read from and write to context
def auth_middleware(request, next):
    token = request.headers.get('Authorization')
    user = validate_token(token)
    request.context.user = user  # Store for later middleware/handlers
    return next(request)

def handler(request):
    # Access context populated by middleware
    user = request.context.user
    return f"Hello, {user.name}"
```

### 5. Error Handling Middleware

```python
def error_handler_middleware(request, next):
    try:
        return next(request)
    except ValidationError as e:
        return Response(status=400, body={"error": str(e)})
    except AuthenticationError as e:
        return Response(status=401, body={"error": "Unauthorized"})
    except AuthorizationError as e:
        return Response(status=403, body={"error": "Forbidden"})
    except NotFoundError as e:
        return Response(status=404, body={"error": "Not found"})
    except Exception as e:
        log.error(f"Unhandled error: {e}")
        return Response(status=500, body={"error": "Internal server error"})
```

This middleware wraps everything, converting exceptions to proper HTTP responses.

---

## Middleware Order Matters

**Correct Order:**
```
1. Request ID     ← First: everything needs an ID for tracing
2. Logging        ← Second: log all requests, including failed auth
3. Error Handler  ← Third: catch all errors from below
4. Rate Limiter   ← Fourth: block attacks before expensive operations
5. Auth           ← Fifth: identify user before authorization
6. Authorization  ← Sixth: check permissions
7. Validation     ← Seventh: validate data for authorized users
8. → Handler      ← Finally: business logic
```

**Wrong Order Example:**
```
1. Validation     ← Wasted effort on unauthorized requests
2. Auth           ← Attackers can probe validation without auth
3. Rate Limiter   ← Too late, already did expensive validation
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 4 (Auth)** | Auth middleware extracts and validates tokens |
| **Section 5 (Validation)** | Validation middleware validates request data |
| **Section 7 (Controllers)** | Middleware pipeline leads to controller |
| **Section 14 (Observability)** | Request ID, timing, logging middleware |
| **Section 12 (Distributed)** | Trace propagation middleware |

---

## Real-World Scenarios

### Scenario 1: The Silent 500 Error
**Problem:** Users report errors, but logs show nothing.
**Root Cause:** Error handler middleware wasn't at the top of stack. Errors in auth middleware bypassed logging.
**Fix:** Error handler as outermost middleware.

### Scenario 2: The Rate Limit Bypass
**Problem:** Rate limiter based on user ID. Unauthenticated requests have no user ID, so no rate limiting.
**Fix:** Rate limit by IP for unauthenticated, by user ID for authenticated.

### Scenario 3: The Request ID Mystery
**Problem:** Distributed trace shows gaps. Some services don't have request IDs.
**Root Cause:** Request ID middleware added after auth. Failed auth requests had no ID.
**Fix:** Request ID as first middleware.

---

## Framework Examples

**Express.js (Node.js):**
```javascript
app.use(requestId());      // First
app.use(morgan('combined')); // Logging
app.use(cors());            // CORS
app.use(rateLimit());       // Rate limiting
app.use(authenticate);      // Auth
app.use(express.json());    // Body parsing
app.use('/api', routes);    // Routes
app.use(errorHandler);      // Error handling (last)
```

**Django (Python):**
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'myapp.middleware.RateLimitMiddleware',
]
```

**Go (Gin):**
```go
r := gin.New()
r.Use(gin.Logger())        // Logging
r.Use(gin.Recovery())      // Error recovery
r.Use(cors.Default())      // CORS
r.Use(RateLimiter())       // Rate limiting
r.Use(Authenticate())      // Auth
```

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design middleware pipelines for different needs
- [ ] Order middleware correctly
- [ ] Use request context effectively
- [ ] Handle errors centrally
- [ ] Debug middleware-related issues

---

## Seniority Challenges

### Junior Level
"What is middleware and why is it useful?"

### Mid Level
"Design a middleware stack for an API that needs: authentication (JWT), rate limiting (100 req/min), request logging, and error handling. What order should they be in and why?"

### Senior Level
"Our microservices need consistent middleware behavior: tracing, auth propagation, circuit breaking, retry logic. How do we ensure all 50 services have consistent middleware without copy-paste? Consider: service mesh vs library approach."

---

## Key Takeaways

1. **Middleware = Cross-cutting concerns** - Define once, apply everywhere
2. **Order matters critically** - Rate limit before auth, auth before validation
3. **Request context is shared state** - How middleware communicates
4. **Error handling wraps everything** - Single place for error-to-response conversion
5. **First middleware runs first AND last** - Onion model for request/response

---

## Prerequisites
- Section 4: Authentication
- Section 5: Validation

## Next Section
→ [Section 7: Controllers & Business Logic](./07-controllers.md) - The pipeline is complete, now the real work begins.
