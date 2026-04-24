# Low-Level Design (LLD)
## E-Commerce Platform — Amazon/Flipkart-Like System

---

## 1. Database Schema

### Users Table (MySQL — User Service)

```sql
CREATE TABLE users (
    user_id       BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(100)        NOT NULL,
    email         VARCHAR(150)        NOT NULL UNIQUE,
    password_hash VARCHAR(255)        NOT NULL,
    phone         VARCHAR(15),
    created_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_email (email)
);

CREATE TABLE addresses (
    address_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id       BIGINT UNSIGNED     NOT NULL,
    street        VARCHAR(255)        NOT NULL,
    city          VARCHAR(100)        NOT NULL,
    state         VARCHAR(100)        NOT NULL,
    pincode       VARCHAR(10)         NOT NULL,
    is_default    BOOLEAN             DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_user_id (user_id)
);
```

### Products Table (MySQL — Product Service)

```sql
CREATE TABLE products (
    product_id    BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name          VARCHAR(255)        NOT NULL,
    description   TEXT,
    category      VARCHAR(100)        NOT NULL,
    price         DECIMAL(10, 2)      NOT NULL,
    seller_id     BIGINT UNSIGNED     NOT NULL,
    image_url     VARCHAR(500),           -- S3 URL
    is_active     BOOLEAN             DEFAULT TRUE,
    created_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_category (category),
    INDEX idx_seller (seller_id),
    FULLTEXT INDEX ft_name_desc (name, description)  -- fallback; primary search via OpenSearch
);

CREATE TABLE product_reviews (
    review_id     BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    product_id    BIGINT UNSIGNED     NOT NULL,
    user_id       BIGINT UNSIGNED     NOT NULL,
    rating        TINYINT UNSIGNED    NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text   TEXT,
    created_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    INDEX idx_product_rating (product_id, rating)
);
```

### Inventory Table (PostgreSQL — Inventory Service)

```sql
CREATE TABLE inventory (
    inventory_id  BIGSERIAL PRIMARY KEY,
    product_id    BIGINT              NOT NULL UNIQUE,
    quantity      INT                 NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    reserved      INT                 NOT NULL DEFAULT 0,  -- items locked during checkout
    updated_at    TIMESTAMP           DEFAULT NOW(),
    INDEX         idx_product_id (product_id)
);

-- Inventory transaction log for auditing
CREATE TABLE inventory_log (
    log_id        BIGSERIAL PRIMARY KEY,
    product_id    BIGINT              NOT NULL,
    delta         INT                 NOT NULL,   -- positive = restock, negative = sold
    reason        VARCHAR(50)         NOT NULL,   -- 'SALE', 'RESTOCK', 'RETURN', 'LOCK', 'UNLOCK'
    order_id      BIGINT,
    created_at    TIMESTAMP           DEFAULT NOW()
);
```

### Cart Collection (MongoDB — Cart Service)

```json
// Collection: carts
{
  "_id": "ObjectId",
  "cart_id": "cart_101",
  "user_id": 4,
  "items": [
    {
      "product_id": 17,
      "product_name": "iPhone 16",
      "quantity": 1,
      "price_at_add": 89999.00,    // price when added; re-verified at checkout
      "image_url": "https://cdn.example.com/iphone16.jpg"
    },
    {
      "product_id": 22,
      "product_name": "AirPods Pro",
      "quantity": 2,
      "price_at_add": 24999.00,
      "image_url": "https://cdn.example.com/airpods.jpg"
    }
  ],
  "created_at": "ISODate",
  "updated_at": "ISODate",
  "expires_at": "ISODate"          // TTL index: 30 days
}

// Indexes:
// db.carts.createIndex({ "user_id": 1 }, { unique: true })
// db.carts.createIndex({ "expires_at": 1 }, { expireAfterSeconds: 0 })
```

### Orders Table (MySQL — Order Service)

```sql
CREATE TABLE orders (
    order_id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id       BIGINT UNSIGNED     NOT NULL,
    payment_id    BIGINT UNSIGNED,
    total_price   DECIMAL(12, 2)      NOT NULL,
    status        ENUM('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED') DEFAULT 'PENDING',
    address_id    BIGINT UNSIGNED     NOT NULL,
    created_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP           DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user_status (user_id, status),
    INDEX idx_created (created_at)
);

CREATE TABLE order_items (
    order_item_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id      BIGINT UNSIGNED     NOT NULL,
    product_id    BIGINT UNSIGNED     NOT NULL,
    quantity      INT UNSIGNED        NOT NULL,
    unit_price    DECIMAL(10, 2)      NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    INDEX idx_order_id (order_id)
);
```

### Payments Table (MySQL — Payment Service)

```sql
CREATE TABLE payments (
    payment_id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id          BIGINT UNSIGNED     NOT NULL UNIQUE,
    user_id           BIGINT UNSIGNED     NOT NULL,
    amount            DECIMAL(12, 2)      NOT NULL,
    payment_mode      ENUM('CARD','UPI','NETBANKING','WALLET') NOT NULL,
    gateway_ref_id    VARCHAR(100),           -- reference from payment gateway
    idempotency_key   VARCHAR(100)        NOT NULL UNIQUE,
    status            ENUM('PENDING','SUCCESS','FAILED','REFUNDED') DEFAULT 'PENDING',
    created_at        TIMESTAMP           DEFAULT CURRENT_TIMESTAMP,
    updated_at        TIMESTAMP           DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_order_id (order_id),
    INDEX idx_idempotency (idempotency_key)
);
```

---

## 2. Class Diagrams (OOP + SOLID Principles)

### User Domain

```
┌────────────────────────────────┐
│           <<Entity>>           │
│              User              │
├────────────────────────────────┤
│ - userId: Long                 │
│ - name: String                 │
│ - email: String                │
│ - passwordHash: String         │
│ - phone: String                │
│ - addresses: List<Address>     │
├────────────────────────────────┤
│ + getDefaultAddress(): Address │
└────────────────────────────────┘
         ◇ 1     * ◇
┌────────────────────────────────┐
│           <<Entity>>           │
│             Address            │
├────────────────────────────────┤
│ - addressId: Long              │
│ - street: String               │
│ - city: String                 │
│ - pincode: String              │
│ - isDefault: boolean           │
└────────────────────────────────┘

┌──────────────────────────────────────┐
│         <<Interface>>                │
│         UserRepository               │
├──────────────────────────────────────┤
│ + findById(id: Long): Optional<User> │
│ + findByEmail(email: String): User   │
│ + save(user: User): User             │
└──────────────────┬───────────────────┘
                   │ implements
┌──────────────────▼───────────────────┐
│        MySQLUserRepository           │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│         <<Service>>                  │
│           UserService                │
├──────────────────────────────────────┤
│ - repo: UserRepository               │
│ - passwordEncoder: PasswordEncoder   │
│ - jwtProvider: JwtProvider           │
├──────────────────────────────────────┤
│ + register(dto: RegisterDto): User   │
│ + login(dto: LoginDto): String (JWT) │
│ + getProfile(userId: Long): User     │
└──────────────────────────────────────┘
```

### Product & Search Domain

```
┌──────────────────────────────────────┐
│           <<Entity>>                 │
│             Product                  │
├──────────────────────────────────────┤
│ - productId: Long                    │
│ - name: String                       │
│ - description: String                │
│ - category: String                   │
│ - price: BigDecimal                  │
│ - imageUrl: String                   │
│ - isActive: boolean                  │
└──────────────────────────────────────┘

┌──────────────────────────────────────────┐
│         <<Interface>>                    │
│         SearchService                    │
├──────────────────────────────────────────┤
│ + search(query: String, page: int,       │
│          size: int): List<ProductId>     │
│ + indexProduct(product: Product): void   │
└──────────────────┬───────────────────────┘
                   │ implements
┌──────────────────▼───────────────────────┐
│      ElasticsearchSearchService          │
│ - esClient: OpenSearchClient             │
└──────────────────────────────────────────┘
```

### Cart Domain

```
┌──────────────────────────────────────────┐
│           <<Entity>>                     │
│               Cart                       │
├──────────────────────────────────────────┤
│ - cartId: String                         │
│ - userId: Long                           │
│ - items: List<CartItem>                  │
│ - expiresAt: LocalDateTime               │
├──────────────────────────────────────────┤
│ + addItem(item: CartItem): void          │
│ + removeItem(productId: Long): void      │
│ + updateQuantity(productId, qty): void   │
│ + getTotal(): BigDecimal                 │
└──────────────────────────────────────────┘
         ◇ 1       * ◇
┌──────────────────────────────────────────┐
│           <<Value Object>>               │
│               CartItem                   │
├──────────────────────────────────────────┤
│ - productId: Long                        │
│ - productName: String                    │
│ - quantity: int                          │
│ - priceAtAdd: BigDecimal                 │
└──────────────────────────────────────────┘
```

### Checkout & Inventory Domain

```
┌──────────────────────────────────────────┐
│         <<Interface>>                    │
│        InventoryService                  │
├──────────────────────────────────────────┤
│ + checkAvailability(productId, qty)      │
│       : boolean                          │
│ + reserveStock(productId, qty, orderId)  │
│       : boolean                          │
│ + releaseReservation(productId, orderId) │
│       : void                             │
│ + decrementStock(productId, qty)         │
│       : void                             │
└──────────────────┬───────────────────────┘
                   │ implements
┌──────────────────▼───────────────────────┐
│    RedisLockedInventoryService            │
│ - redisTemplate: RedisTemplate           │
│ - inventoryRepo: InventoryRepository     │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│         <<Service>>                      │
│        CheckoutService                   │
├──────────────────────────────────────────┤
│ - cartService: CartService               │
│ - inventoryService: InventoryService     │
│ - paymentService: PaymentService         │
│ - eventPublisher: KafkaEventPublisher    │
├──────────────────────────────────────────┤
│ + checkout(userId, cartId,               │
│            paymentDto): OrderId          │
└──────────────────────────────────────────┘
```

### Design Patterns Used

| Pattern | Where Applied | Why |
|---|---|---|
| Repository Pattern | All data services | Decouples business logic from data access; supports DI and mocking |
| Strategy Pattern | PaymentService | Multiple payment modes (Card, UPI, Wallet) swapped at runtime |
| Factory Pattern | PaymentGatewayFactory | Creates correct gateway client (Razorpay, Stripe) based on config |
| Singleton Pattern | Database connection pools, Kafka producer | Only one instance needed; manages shared resources |
| Saga Pattern | CheckoutService (orchestration) | Coordinates multi-step distributed transaction with compensating actions |
| Observer / Event-Driven | Kafka producers & consumers | Decouples Checkout from Order and Inventory services |
| Decorator Pattern | Cached ProductService | Wraps base ProductService with Redis caching layer transparently |

---

## 3. Relationships & Indexing Strategy

### Key Indexes

| Table | Index | Type | Purpose |
|---|---|---|---|
| users | email | UNIQUE | Fast login lookup |
| products | category | B-TREE | Filter by category |
| products | (name, description) | FULLTEXT | Fallback text search |
| inventory | product_id | UNIQUE | O(1) stock lookup per product |
| orders | (user_id, status) | COMPOSITE | "My Orders" filtered by status |
| payments | idempotency_key | UNIQUE | Prevent duplicate payments |
| carts (Mongo) | user_id | UNIQUE | One cart per user |
| carts (Mongo) | expires_at | TTL | Auto-delete stale carts |

---

## 4. Sequence Diagrams

### Sequence 1: User Login & Product Search

```
Client          API Gateway      User Service     MySQL (Users)    Search Svc     OpenSearch
  │                  │                │                │               │               │
  │──POST /login────>│                │                │               │               │
  │                  │──validate──────>│               │               │               │
  │                  │                │──SELECT────────>│              │               │
  │                  │                │<──User row──────│              │               │
  │                  │                │──bcrypt verify  │              │               │
  │                  │                │──sign JWT       │              │               │
  │                  │<──JWT Token────│                │               │               │
  │<──200 JWT────────│                │                │               │               │
  │                  │                │                │               │               │
  │──GET /search?q=iPhone 16─────────────────────────>│               │               │
  │  (Bearer: JWT)   │                │                │               │               │
  │                  │──validate JWT  │                │               │               │
  │                  │──route to Search Svc────────────────────────────>│              │
  │                  │                │                │               │──query───────>│
  │                  │                │                │               │  tokenize,    │
  │                  │                │                │               │  inverted idx │
  │                  │                │                │               │<──[IDs]───────│
  │<──200 [ProductIDs, page 1]────────────────────────────────────────│               │
```

### Sequence 2: Checkout with Inventory Lock (Race Condition Handling)

```
Client     API GW    Checkout Svc    Inventory Svc     Redis     Payment Svc   Kafka    Order Svc
  │           │             │               │             │             │          │          │
  │─POST /checkout─────────>│               │             │             │          │          │
  │           │             │──fetch cart   │             │             │          │          │
  │           │             │──checkAvailability(productId=17, qty=1)──>│          │          │
  │           │             │               │──SET lock:17 NX TTL 60s──>│          │          │
  │           │             │               │<──OK (lock acquired)───────│          │          │
  │           │             │               │──SELECT quantity from PG   │          │          │
  │           │             │               │  WHERE product_id=17       │          │          │
  │           │             │               │──qty=10 > 1 → reserve      │          │          │
  │           │             │<──reserved OK─│             │             │          │          │
  │           │             │──initiatePayment────────────────────────────────────>│          │
  │           │             │               │             │             │          │          │
  │    [if PAYMENT SUCCESS]                 │             │             │          │          │
  │           │             │──publish order.created──────────────────────────────>│─────────>│
  │           │             │──publish payment.success────────────────────────────>│          │
  │           │             │               │<────────────────── consume stock.decrement ──────│
  │           │             │               │──UPDATE inventory SET qty=qty-1 WHERE product=17 │
  │           │             │               │──DEL lock:17────────────>│          │          │
  │           │             │               │                          │          │          │
  │<──200 {order_id}────────│               │                          │          │          │
  │           │             │                                          │          │          │
  │    [if PAYMENT FAILED]                  │                          │          │          │
  │           │             │──publish payment.failed─────────────────────────────>│          │
  │           │             │               │<────────────────── consume release.lock──────────│
  │           │             │               │──DEL lock:17────────────>│          │          │
  │<──402 Payment Failed────│               │                                                  │
```
