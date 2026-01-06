# Section 16: Code Quality & Standards

> "The best code is the code that's easy to change. The worst code is the code that's afraid to change."

---

## The Problem This Solves

You inherited a codebase:
- No tests → Afraid to change anything
- No documentation → No idea what it does
- No patterns → Every file is different
- No standards → 5 ways to do the same thing

Result: Simple changes take weeks. Bugs multiply. Team velocity collapses.

Code quality is about sustainable velocity: moving fast today WITHOUT slowing down tomorrow.

---

## First Principles

### Principle 1: Code is Read More Than Written
You write code once. You (and others) read it hundreds of times. Optimize for readability.

### Principle 2: Tests are Documentation
Tests describe what the code SHOULD do. When in doubt, read the tests.

### Principle 3: Complexity is the Enemy
Every layer of abstraction, every clever trick, every "just in case" feature - they all add cognitive load. Simple beats clever.

---

## Core Concepts

### 1. The Testing Pyramid

```
                         ┌───────────┐
                         │   E2E     │  Few, slow, expensive
                         │   Tests   │  Test full user journeys
                         └─────┬─────┘
                               │
                    ┌──────────┴──────────┐
                    │   Integration       │  Some, medium speed
                    │      Tests          │  Test component interactions
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┴─────────────────────┐
         │              Unit Tests                   │  Many, fast, cheap
         │                                           │  Test individual functions
         └───────────────────────────────────────────┘
```

**Unit Tests (70%):**
```python
def test_calculate_total_with_discount():
    order = Order(items=[Item(price=100)])
    order.apply_discount(percent=10)
    assert order.total == 90  # Fast, isolated
```

**Integration Tests (20%):**
```python
def test_create_order_saves_to_database():
    service = OrderService(real_database)
    order = service.create_order(user_id="123", items=[...])
    assert database.get_order(order.id) is not None  # Tests real integration
```

**E2E Tests (10%):**
```python
def test_complete_checkout_flow():
    browser.login("user@example.com")
    browser.add_to_cart("product-123")
    browser.checkout()
    assert browser.sees("Order Confirmed")  # Full user journey
```

### 2. SOLID Principles

**S - Single Responsibility:**
```python
# BAD: Does everything
class User:
    def save_to_database(self): ...
    def send_email(self): ...
    def generate_report(self): ...

# GOOD: One reason to change
class User:
    # Just user data and behavior

class UserRepository:
    def save(self, user): ...

class EmailService:
    def send_welcome(self, user): ...
```

**O - Open/Closed:**
```python
# BAD: Must modify to add new discount type
def calculate_discount(order, discount_type):
    if discount_type == "percent":
        return order.total * 0.1
    elif discount_type == "fixed":
        return 10
    # Add new types here...

# GOOD: Extend without modifying
class Discount(ABC):
    @abstractmethod
    def apply(self, order): pass

class PercentDiscount(Discount):
    def apply(self, order):
        return order.total * self.percent

class FixedDiscount(Discount):
    def apply(self, order):
        return self.amount
```

**L - Liskov Substitution:**
```python
# BAD: Square breaks rectangle expectations
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):
    def set_width(self, w):
        self.width = w
        self.height = w  # Breaks expectation!

# GOOD: Don't inherit if behavior differs
class Shape(ABC):
    @abstractmethod
    def area(self): pass
```

**I - Interface Segregation:**
```python
# BAD: Force implementation of unused methods
class Worker(ABC):
    @abstractmethod
    def work(self): pass
    @abstractmethod
    def eat(self): pass  # Robots don't eat!

# GOOD: Split interfaces
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Eatable(ABC):
    @abstractmethod
    def eat(self): pass
```

**D - Dependency Inversion:**
```python
# BAD: High-level depends on low-level
class OrderService:
    def __init__(self):
        self.db = PostgresDatabase()  # Concrete dependency

# GOOD: Depend on abstractions
class OrderService:
    def __init__(self, repository: OrderRepository):  # Interface
        self.repository = repository
```

### 3. Code Review Standards

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CODE REVIEW CHECKLIST                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CORRECTNESS                                                                │
│  □ Does it do what it's supposed to?                                       │
│  □ Are edge cases handled?                                                 │
│  □ Are errors handled appropriately?                                       │
│                                                                             │
│  READABILITY                                                                │
│  □ Is the code self-explanatory?                                           │
│  □ Are names meaningful?                                                   │
│  □ Is complexity minimized?                                                │
│                                                                             │
│  TESTING                                                                    │
│  □ Are there tests?                                                        │
│  □ Do tests cover edge cases?                                              │
│  □ Are tests readable and maintainable?                                    │
│                                                                             │
│  SECURITY                                                                   │
│  □ Is user input validated?                                                │
│  □ Are there SQL injection risks?                                          │
│  □ Is sensitive data protected?                                            │
│                                                                             │
│  PERFORMANCE                                                                │
│  □ Are there N+1 queries?                                                  │
│  □ Is caching considered?                                                  │
│  □ Are there obvious bottlenecks?                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4. Architecture Patterns

**Layered Architecture:**
```
┌─────────────────────────────────────────┐
│  Presentation Layer                     │  Controllers, APIs
├─────────────────────────────────────────┤
│  Application Layer                      │  Use cases, services
├─────────────────────────────────────────┤
│  Domain Layer                           │  Entities, business rules
├─────────────────────────────────────────┤
│  Infrastructure Layer                   │  Database, external APIs
└─────────────────────────────────────────┘

Rule: Dependencies point DOWN only
```

**Clean Architecture:**
```
┌───────────────────────────────────────────────────────────────┐
│                         Frameworks                            │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                    Adapters                           │   │
│  │  ┌───────────────────────────────────────────────┐   │   │
│  │  │              Use Cases                        │   │   │
│  │  │  ┌───────────────────────────────────────┐   │   │   │
│  │  │  │            Entities                   │   │   │   │
│  │  │  │     (Business rules, no deps)         │   │   │   │
│  │  │  └───────────────────────────────────────┘   │   │   │
│  │  └───────────────────────────────────────────────┘   │   │
│  └───────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘

Rule: Dependencies point INWARD only
```

### 5. Technical Debt

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TECHNICAL DEBT                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INTENTIONAL DEBT                    UNINTENTIONAL DEBT                     │
│  "We know this is a shortcut"       "We didn't know better"                │
│                                                                             │
│  Examples:                           Examples:                              │
│  - Quick hack for deadline           - Overcomplicated design              │
│  - Skip tests for MVP                - Missing tests (forgot)               │
│  - TODO comments                     - Copy-paste code                      │
│                                                                             │
│  Track it!                           Learn from it!                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MANAGING DEBT:                                                             │
│                                                                             │
│  1. Make it visible (track in issues)                                      │
│  2. Pay interest (allocate % of sprints to debt)                           │
│  3. Don't bankrupt (set limits on debt accumulation)                       │
│  4. Prioritize by pain (fix what hurts most)                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6. Documentation

**Code Comments - When to use:**
```python
# BAD: Explains WHAT (code already says that)
# Increment counter by 1
counter += 1

# GOOD: Explains WHY (not obvious from code)
# Use 1.1 multiplier to account for timezone edge cases
# See: https://github.com/org/project/issues/123
adjustment_factor = 1.1
```

**README template:**
```markdown
# Service Name

## What
Brief description of what this service does.

## Why
Why does this service exist? What problem does it solve?

## How
High-level architecture. How does it work?

## Getting Started
How to run locally.

## API Reference
Endpoint documentation (or link to OpenAPI spec).

## Configuration
Environment variables and their purposes.
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 7 (Controllers)** | Architecture patterns apply |
| **Section 14 (Observability)** | Testing requires observability |
| **Section 15 (Cloud-Native)** | CI/CD enforces quality |

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Design testable architectures
- [ ] Apply SOLID principles
- [ ] Conduct effective code reviews
- [ ] Manage technical debt
- [ ] Write meaningful documentation

---

## Seniority Challenges

### Junior Level
"What's the difference between unit tests and integration tests?"

### Mid Level
"Our service has 50% test coverage but tests are brittle and break with every refactor. How do we improve the testing strategy without rewriting everything?"

### Senior Level
"Our 3-year-old codebase has: no consistent architecture, 10% test coverage, 500+ TODO comments, and 3 different ways to do database access. New features take 3x longer than expected. Design a technical debt reduction strategy that doesn't halt feature development."

---

## Key Takeaways

1. **Tests enable change** - Without tests, you're afraid to refactor
2. **Simple beats clever** - Readable code is maintainable code
3. **SOLID for flexibility** - Principles that make change easier
4. **Debt is okay, bankruptcy isn't** - Track and pay down debt
5. **Documentation is for humans** - Write for the next developer

---

## Prerequisites
- Section 7: Controllers (code organization)
- Section 15: Cloud-Native (CI/CD context)

## Conclusion
This is the final section. Go back to [Overview](./00-overview.md) to see how all 16 sections connect.
