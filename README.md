# Coupon & Discount Engine — Spring Boot

Enterprise-grade coupon and discount system for an e-commerce platform.
Supports flat, percentage, and conditional discounts with full concurrency safety,
idempotency, and the Strategy Pattern.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Spring Boot 3.2.0 |
| Language | Java 17 |
| ORM | Spring Data JPA + Hibernate |
| Database | H2 In-Memory (no install needed) |
| Validation | Hibernate Validator (Jakarta) |
| API Docs | SpringDoc OpenAPI 2 (Swagger UI) |
| Build | Maven |
| Boilerplate | Lombok |

---

## How to Run in Eclipse

### Prerequisites
- JDK 17 installed and configured in Eclipse
- Eclipse IDE for Enterprise Java (2023-06 or later)
- Maven (bundled with Eclipse)

### Steps

1. **Import the project**
   - File → Import → Maven → Existing Maven Projects
   - Browse to the `coupon-discount-engine` folder → Finish

2. **Wait for Maven to download dependencies** (~1 minute, first time only)

3. **Run the application**
   - Expand the project → `src/main/java`
   - Navigate to `com.ecommerce.coupon`
   - Right-click `CouponDiscountEngineApplication.java`
   - Run As → Spring Boot App

4. **Verify startup** — look for this line in the Console:
   ```
   Started CouponDiscountEngineApplication in X.XXX seconds
   ```

5. **Access the application:**

| Tool | URL |
|------|-----|
| Swagger UI | http://localhost:8080/swagger-ui.html |
| H2 Console | http://localhost:8080/h2-console |
| API Docs JSON | http://localhost:8080/api-docs |

   **H2 Console settings:**
   - JDBC URL: `jdbc:h2:mem:coupondb`
   - Username: `sa`
   - Password: *(leave empty)*

---

## Project Structure

```
coupon-discount-engine/
├── src/main/java/com/ecommerce/coupon/
│   ├── CouponDiscountEngineApplication.java   ← Main class
│   ├── config/
│   │   └── SwaggerConfig.java
│   ├── controller/
│   │   ├── CouponController.java              ← CRUD for coupons
│   │   ├── DiscountController.java            ← Apply coupons to orders
│   │   ├── OrderController.java
│   │   └── UserController.java
│   ├── dto/
│   │   └── CouponDTOs.java                   ← All request/response DTOs
│   ├── entity/
│   │   ├── AppUser.java
│   │   ├── Coupon.java                        ← @Version for optimistic lock
│   │   ├── CouponType.java                    ← FLAT | PERCENTAGE | CONDITIONAL
│   │   ├── Order.java
│   │   ├── OrderDiscount.java                 ← Idempotency key stored here
│   │   └── UserCoupon.java
│   ├── exception/
│   │   ├── CouponAlreadyUsedException.java
│   │   ├── CouponException.java
│   │   ├── GlobalExceptionHandler.java        ← @RestControllerAdvice
│   │   └── ResourceNotFoundException.java
│   ├── repository/
│   │   ├── CouponRepository.java              ← PESSIMISTIC_WRITE lock query
│   │   ├── OrderDiscountRepository.java
│   │   ├── OrderRepository.java               ← Fetch join to prevent N+1
│   │   ├── UserCouponRepository.java
│   │   └── UserRepository.java
│   └── service/
│       ├── CouponService.java                 ← Core business logic
│       └── strategy/
│           ├── ConditionalDiscountStrategy.java
│           ├── DiscountStrategy.java          ← Strategy Pattern interface
│           ├── DiscountStrategyFactory.java   ← Auto-resolves by coupon type
│           ├── FlatDiscountStrategy.java
│           └── PercentageDiscountStrategy.java
├── src/main/resources/
│   ├── application.properties
│   └── data.sql                              ← Seed data (coupons, users, orders)
└── pom.xml
```

---

## API Reference

### Coupon Management — `/api/coupons`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/coupons` | Create a new coupon |
| GET | `/api/coupons` | List all coupons |
| GET | `/api/coupons/{code}` | Get coupon by code |

#### Create Coupon Request Body
```json
{
  "code": "SUMMER25",
  "type": "PERCENTAGE",
  "discountValue": 25.00,
  "expiryDate": "2025-12-31",
  "usageLimit": 500,
  "minOrderValue": 300.00,
  "applicableCategory": null
}
```
> `type` must be one of: `FLAT`, `PERCENTAGE`, `CONDITIONAL`

---

### Discount Engine — `/api/discounts`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/discounts/apply` | Apply a single coupon to an order |
| POST | `/api/discounts/apply-multiple` | Apply multiple coupons at once |
| GET | `/api/discounts/orders/{orderId}/summary` | Get discount summary for an order |

#### Apply Single Coupon
```json
{
  "orderId": 1,
  "userId": 1,
  "couponCode": "FLAT50",
  "idempotencyKey": "req-abc-123"
}
```

#### Apply Multiple Coupons
```json
{
  "orderId": 2,
  "userId": 2,
  "couponCodes": ["SAVE10PCT", "FLAT50"],
  "idempotencyKey": "batch-xyz-456"
}
```

#### Sample Response — Order Summary
```json
{
  "success": true,
  "data": {
    "orderId": 1,
    "userId": 1,
    "totalAmount": 500.00,
    "finalAmount": 450.00,
    "totalDiscount": 50.00,
    "status": "PENDING",
    "discounts": [
      {
        "couponCode": "FLAT50",
        "discountType": "FLAT",
        "discountAmount": 50.00,
        "appliedAt": "2025-06-01T10:30:00"
      }
    ]
  }
}
```

---

### User & Order Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/users` | Create a user |
| GET | `/api/users` | List users |
| GET | `/api/users/{id}` | Get user |
| POST | `/api/orders` | Create an order |
| GET | `/api/orders` | List orders |
| GET | `/api/orders/{id}` | Get order |

#### Create Order
```json
{
  "userId": 1,
  "totalAmount": 800.00,
  "category": "ELECTRONICS"
}
```

---

## Seed Data (Ready to Use)

### Coupons

| Code | Type | Value | Min Order | Category | Limit |
|------|------|-------|-----------|----------|-------|
| FLAT50 | FLAT | ₹50 off | ₹200 | Any | 100 |
| SAVE10PCT | PERCENTAGE | 10% off | ₹100 | Any | 200 |
| ELECTRONICS20 | PERCENTAGE | 20% off | ₹500 | ELECTRONICS | 50 |
| BOGO100 | CONDITIONAL | ₹100+ bonus | ₹1000 | Any | 30 |
| SINGLEUSE | FLAT | ₹75 off | ₹150 | Any | 1 (single use) |
| EXPIRED10 | FLAT | ₹10 off | — | Any | Expired |

### Users: IDs 1 (john_doe), 2 (jane_smith), 3 (test_user)
### Orders: IDs 1 (₹500 GENERAL), 2 (₹1200 ELECTRONICS), 3 (₹800 GENERAL)

---

## Quick Test Scenarios

### Scenario 1 — Apply flat discount
```
POST /api/discounts/apply
{ "orderId": 1, "userId": 1, "couponCode": "FLAT50" }
→ Expected: ₹50 discount, finalAmount = ₹450
```

### Scenario 2 — Apply category-restricted coupon
```
POST /api/discounts/apply
{ "orderId": 2, "userId": 2, "couponCode": "ELECTRONICS20" }
→ Expected: 20% discount on ₹1200 = ₹240 off, finalAmount = ₹960
```

### Scenario 3 — Apply expired coupon
```
POST /api/discounts/apply
{ "orderId": 1, "userId": 1, "couponCode": "EXPIRED10" }
→ Expected: 400 Bad Request — "Coupon has expired"
```

### Scenario 4 — Idempotent retry (same key)
```
POST /api/discounts/apply
{ "orderId": 1, "userId": 1, "couponCode": "FLAT50", "idempotencyKey": "my-key-001" }
→ Apply twice with same key → second call returns { "alreadyApplied": true }
```

### Scenario 5 — Multiple coupons
```
POST /api/discounts/apply-multiple
{ "orderId": 3, "userId": 3, "couponCodes": ["FLAT50", "SAVE10PCT"] }
→ Both applied sequentially, total discount shown
```

### Scenario 6 — Single-use coupon reuse
```
POST /api/discounts/apply → apply SINGLEUSE once → success
POST /api/discounts/apply → apply SINGLEUSE again → 409 Conflict
```

---

## Design Decisions

### Strategy Pattern
Each coupon type (`FLAT`, `PERCENTAGE`, `CONDITIONAL`) has its own `DiscountStrategy`
implementation. The `DiscountStrategyFactory` resolves the correct strategy at runtime.
**Adding a new discount type only requires creating a new `@Component` class** — no changes
to the factory or service needed.

### Idempotency
Every `applyCoupon` request generates or accepts an idempotency key.
Before applying, the system checks `OrderDiscount.idempotencyKey`.
If a matching record exists, the stored result is returned immediately without re-processing.

### Concurrency Safety
- **Pessimistic Write Lock** (`SELECT ... FOR UPDATE`) on the `Coupon` row prevents two
  concurrent threads from both reading `currentUsage = 4` and writing `5`.
- **`@Version`** on `Coupon` provides optimistic locking as a secondary guard.
- **`@Transactional`** on all write methods ensures atomicity across the
  Coupon / UserCoupon / OrderDiscount / Order updates.

### N+1 Prevention
`OrderRepository.findByIdWithDiscounts()` uses a JPQL fetch join:
```sql
SELECT o FROM Order o
LEFT JOIN FETCH o.orderDiscounts od
LEFT JOIN FETCH od.coupon
WHERE o.id = :orderId
```
This loads the order + all its discounts + coupon data in a **single SQL query**.

### Validation
Hibernate Validator annotations on request DTOs (`@NotBlank`, `@NotNull`, `@Positive`, etc.)
are enforced at the controller boundary. `GlobalExceptionHandler` converts
`MethodArgumentNotValidException` into structured JSON error responses.

---

## Running Tests

In Eclipse:
- Right-click the project → Run As → JUnit Test

Or via Maven:
```bash
mvn test
```

---

## Common Issues

| Problem | Solution |
|---------|----------|
| Port 8080 already in use | Change `server.port` in `application.properties` |
| Maven dependencies not resolved | Right-click project → Maven → Update Project |
| H2 console shows no tables | Ensure `spring.sql.init.mode=always` is set |
| Lombok errors | Install Lombok Eclipse plugin from https://projectlombok.org/setup/eclipse |
