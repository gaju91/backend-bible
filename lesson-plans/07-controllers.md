# Section 7: Controllers & Business Logic

> "Controllers are translators. They speak HTTP to the outside world and domain language to the inside."

---

## The Problem This Solves

The middleware pipeline has done its job:
- Request authenticated ✓
- Data validated ✓
- Rate limits checked ✓

Now it's time for the actual work. Sarah wants to buy shoes.

But where does this logic live?
- In the route handler? (Gets messy fast)
- In one giant file? (Unmaintainable)
- Scattered everywhere? (Untestable)

Controllers organize the translation from HTTP concepts to business operations.

---

## First Principles

### Principle 1: Separate Transport from Logic
HTTP status codes, headers, query parameters - these are transport concerns. "Can Sarah buy these shoes?" - this is business logic. They shouldn't mix.

### Principle 2: Business Rules Apply Regardless of Entry Point
The rule "max 2 items per customer" should apply whether the order comes from:
- REST API
- GraphQL
- CLI tool
- Background job
- gRPC service

### Principle 3: The Domain Model is the Truth
Your code should reflect how the business thinks, not how HTTP works.

---

## Core Concepts

### 1. The Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                       │
│  • HTTP Controllers (REST)                                  │
│  • GraphQL Resolvers                                        │
│  • gRPC Handlers                                            │
│  • CLI Commands                                             │
│                                                             │
│  Responsibility: Translate external requests to internal    │
│                  operations and format responses            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                        │
│  • Use Cases / Services                                     │
│  • Orchestration logic                                      │
│  • Transaction boundaries                                   │
│                                                             │
│  Responsibility: Coordinate business operations,            │
│                  enforce workflows                          │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER                           │
│  • Entities (User, Order, Product)                          │
│  • Value Objects (Money, Address)                           │
│  • Domain Services                                          │
│  • Business Rules                                           │
│                                                             │
│  Responsibility: Core business logic, invariants            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE LAYER                       │
│  • Repositories (Database access)                           │
│  • External APIs                                            │
│  • Message queues                                           │
│  • File storage                                             │
│                                                             │
│  Responsibility: Technical implementations, external        │
│                  system communication                       │
└─────────────────────────────────────────────────────────────┘
```

### 2. Controller Responsibilities

**What Controllers DO:**
```python
class OrderController:
    def create_order(self, request):
        # 1. Extract data from HTTP request
        product_id = request.json['product_id']
        quantity = request.json['quantity']
        user_id = request.context.user.id

        # 2. Call service layer
        order = self.order_service.create_order(
            user_id=user_id,
            product_id=product_id,
            quantity=quantity
        )

        # 3. Format HTTP response
        return Response(
            status=201,
            body=OrderSerializer(order).to_json(),
            headers={'Location': f'/orders/{order.id}'}
        )
```

**What Controllers DON'T DO:**
```python
class OrderController:
    def create_order(self, request):
        # ❌ Don't do database queries directly
        product = db.query("SELECT * FROM products WHERE id = ?", product_id)

        # ❌ Don't implement business rules
        if user.order_count_today >= 5:
            raise TooManyOrdersError()

        # ❌ Don't send emails
        email_service.send_confirmation(user.email)

        # ❌ Don't talk to external services
        payment_gateway.charge(user.card, total)
```

### 3. Service Layer Pattern

Services orchestrate operations and contain business logic:

```python
class OrderService:
    def __init__(self, order_repo, product_repo, inventory_service, payment_service):
        self.order_repo = order_repo
        self.product_repo = product_repo
        self.inventory_service = inventory_service
        self.payment_service = payment_service

    def create_order(self, user_id: str, product_id: str, quantity: int) -> Order:
        # Business rule: max 2 per customer per day
        today_orders = self.order_repo.count_user_orders_today(user_id)
        if today_orders >= 2:
            raise OrderLimitExceededError("Maximum 2 orders per day")

        # Get product
        product = self.product_repo.get(product_id)
        if not product:
            raise ProductNotFoundError(product_id)

        # Check and reserve inventory
        if not self.inventory_service.reserve(product_id, quantity):
            raise OutOfStockError(product_id)

        try:
            # Create order
            order = Order(
                user_id=user_id,
                product_id=product_id,
                quantity=quantity,
                total=product.price * quantity,
                status=OrderStatus.PENDING
            )

            # Save order
            self.order_repo.save(order)

            return order

        except Exception as e:
            # Rollback inventory reservation on failure
            self.inventory_service.release(product_id, quantity)
            raise
```

### 4. Domain Entities vs DTOs

**Entity (Domain Layer):**
```python
class Order:
    """Rich domain object with behavior"""

    def __init__(self, user_id, items, status=OrderStatus.DRAFT):
        self.id = generate_id()
        self.user_id = user_id
        self.items = items
        self.status = status
        self.created_at = datetime.now()

    def calculate_total(self):
        """Domain logic lives here"""
        subtotal = sum(item.price * item.quantity for item in self.items)
        tax = subtotal * 0.08
        return subtotal + tax

    def can_cancel(self):
        """Business rule: can only cancel if not shipped"""
        return self.status in [OrderStatus.DRAFT, OrderStatus.PENDING]

    def cancel(self):
        if not self.can_cancel():
            raise OrderCannotBeCancelledError()
        self.status = OrderStatus.CANCELLED
```

**DTO (Data Transfer Object):**
```python
class OrderResponse:
    """Simple data structure for API response"""

    def __init__(self, order: Order):
        self.id = order.id
        self.total = str(order.calculate_total())  # String for JSON precision
        self.status = order.status.value
        self.item_count = len(order.items)
        # Note: No business methods, just data
```

### 5. The Repository Pattern

Repositories abstract data access:

```python
# Interface (what controllers/services see)
class OrderRepository(ABC):
    @abstractmethod
    def get(self, order_id: str) -> Optional[Order]:
        pass

    @abstractmethod
    def save(self, order: Order) -> None:
        pass

    @abstractmethod
    def find_by_user(self, user_id: str) -> List[Order]:
        pass

# Implementation (infrastructure detail)
class PostgresOrderRepository(OrderRepository):
    def get(self, order_id: str) -> Optional[Order]:
        row = self.db.query(
            "SELECT * FROM orders WHERE id = %s",
            [order_id]
        )
        return self._row_to_order(row) if row else None

    def save(self, order: Order) -> None:
        self.db.execute(
            "INSERT INTO orders (id, user_id, total, status) VALUES (%s, %s, %s, %s)",
            [order.id, order.user_id, order.total, order.status.value]
        )
```

**Why abstract?**
- Services don't know/care about PostgreSQL vs MongoDB
- Easy to test with in-memory repository
- Can swap databases without changing business logic

---

## Request → Response Flow

```
POST /api/orders
{
  "product_id": "jordan-1",
  "quantity": 1
}

         ┌─────────────────────────────────────────────────────┐
         │                   OrderController                   │
         │                                                     │
         │  • Extract product_id, quantity from request        │
         │  • Get user_id from request.context (set by auth)   │
         │  • Call OrderService.create_order()                 │
         │  • Format response with status 201                  │
         │                                                     │
         └──────────────────────┬──────────────────────────────┘
                                │
                                ▼
         ┌─────────────────────────────────────────────────────┐
         │                    OrderService                     │
         │                                                     │
         │  • Check order limit (business rule)                │
         │  • Get product from repository                      │
         │  • Reserve inventory                                │
         │  • Create Order entity                              │
         │  • Save to repository                               │
         │  • Return Order                                     │
         │                                                     │
         └──────────────────────┬──────────────────────────────┘
                                │
                                ▼
         ┌─────────────────────────────────────────────────────┐
         │              Repositories / Infrastructure          │
         │                                                     │
         │  • ProductRepository.get() → SQL query              │
         │  • InventoryService.reserve() → Redis + SQL         │
         │  • OrderRepository.save() → SQL insert              │
         │                                                     │
         └─────────────────────────────────────────────────────┘

Response:
{
  "id": "order_abc123",
  "total": "169.99",
  "status": "pending"
}
```

---

## Connections to Other Sections

| Connected To | How |
|--------------|-----|
| **Section 6 (Middleware)** | Middleware pipeline feeds into controller |
| **Section 8 (Database)** | Repositories abstract database access |
| **Section 9 (Async)** | Services trigger async jobs after operations |
| **Section 10 (Events)** | Domain events published after state changes |
| **Section 11 (APIs)** | Same services, different presentation layers |
| **Section 12 (Distributed)** | Services may call other microservices |

---

## Real-World Scenarios

### Scenario 1: The Fat Controller
**Problem:** Controller has 500 lines, mixes HTTP handling with business rules.
**Symptom:** Same business logic duplicated in CLI tool.
**Fix:** Extract to service layer. Controller becomes thin translator.

### Scenario 2: The Anemic Domain
**Problem:** Entities are just data holders. All logic in services.
```python
# Anemic (bad)
class Order:
    id: str
    total: float
    status: str

class OrderService:
    def calculate_total(self, order): ...
    def can_cancel(self, order): ...
    def cancel(self, order): ...
```
**Fix:** Move behavior to entities. Services orchestrate, entities contain rules.

### Scenario 3: The Leaky Abstraction
**Problem:** Controller catches `SQLException`. Now tied to database choice.
**Fix:** Repositories catch infrastructure exceptions, raise domain exceptions.
```python
class OrderRepository:
    def save(self, order):
        try:
            self.db.insert(order)
        except UniqueConstraintViolation:
            raise DuplicateOrderError(order.id)  # Domain exception
```

---

## Testing Strategy

**Unit Test Services (Mock repositories):**
```python
def test_order_limit_exceeded():
    mock_repo = Mock()
    mock_repo.count_user_orders_today.return_value = 2

    service = OrderService(mock_repo, ...)

    with pytest.raises(OrderLimitExceededError):
        service.create_order(user_id="123", product_id="abc", quantity=1)
```

**Integration Test Repositories (Real database):**
```python
def test_order_saved_correctly():
    repo = PostgresOrderRepository(test_db)
    order = Order(user_id="123", items=[...])

    repo.save(order)
    retrieved = repo.get(order.id)

    assert retrieved.user_id == "123"
```

**E2E Test Controllers (HTTP requests):**
```python
def test_create_order_endpoint():
    response = client.post('/api/orders', json={
        'product_id': 'jordan-1',
        'quantity': 1
    }, headers={'Authorization': 'Bearer token'})

    assert response.status_code == 201
    assert 'id' in response.json
```

---

## What You'll Build Understanding Of

After this section, you can:
- [ ] Structure code in layers with clear responsibilities
- [ ] Write thin controllers and rich services
- [ ] Design domain entities with behavior
- [ ] Use repositories for data access abstraction
- [ ] Test each layer appropriately

---

## Seniority Challenges

### Junior Level
"What's the difference between a controller and a service?"

### Mid Level
"Design the service layer for an e-commerce checkout that needs to: validate cart, check inventory, apply discount codes, calculate shipping, process payment, and create order. How do you handle partial failures?"

### Senior Level
"Our Order service has grown to 3000 lines with 50 methods. It handles orders, payments, shipping, notifications, and analytics. How do we refactor without breaking existing functionality? What's the decomposition strategy?"

---

## Key Takeaways

1. **Controllers translate, services orchestrate, entities contain rules**
2. **Same business logic, multiple entry points** - Services enable reuse
3. **Rich domain models** - Behavior lives with data
4. **Repository pattern** - Abstract persistence details
5. **Test at every layer** - Unit, integration, E2E

---

## Prerequisites
- Section 6: Middleware (understanding request pipeline)

## Next Section
→ [Section 8: Databases & Caching](./08-database-caching.md) - How do we persist this order reliably?
