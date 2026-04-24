# High-Level Design (HLD)
## E-Commerce Platform — Amazon/Flipkart-Like System

---

## 1. Functional Requirements

1. Users can search and find products by title, name, or category.
2. Users can view product details: description, images, price, available quantity, and reviews.
3. Users can select quantity and add products to a cart.
4. Users can check out and make payments (debit/credit card, UPI, net banking, wallets).
5. Users can track the status of their orders.
6. The system must handle concurrent purchases of limited-stock items (race conditions / flash sales).

---

## 2. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Daily Active Users (DAU) | 100 Million |
| Orders per Second | 10 orders/sec (peak) |
| Search Latency | < 200 ms (p99) |
| Checkout Latency | < 500 ms (p99) |
| Availability (Search/Browse) | 99.99% (High Availability) |
| Consistency (Payment/Inventory) | Strong Consistency (ACID) |
| Scalability | Horizontal scaling for all stateless services |
| Data Durability | 99.999999999% (S3-level for assets) |

### CAP Theorem Considerations

The system is split by CAP requirements per domain:

- **Search & Product Browsing → AP (Availability + Partition Tolerance)**: Users should always be able to browse products. Slight staleness from CDC replication lag (seconds) is acceptable.
- **Checkout, Payment & Inventory → CP (Consistency + Partition Tolerance)**: Overselling is unacceptable. Distributed locks and ACID transactions are enforced even at the cost of brief unavailability.
- **Cart → AP with eventual consistency**: Cart data is not mission-critical; MongoDB replication provides eventual consistency.

---

## 3. Core Entities

| Entity | Description |
|---|---|
| User | Registered buyer with profile, address, and auth credentials |
| Product | Item listed for sale with attributes, price, images, and description |
| Inventory | Stock count per product, managed independently for concurrency control |
| Cart | A user's collection of products selected for purchase |
| Order | A confirmed purchase record after successful checkout |
| Payment | Payment attempt linked to an order; tracks status and mode |

---

## 4. High-Level Architecture

### System Architecture Diagram

```
                            ┌─────────────────────────────────────────────────────┐
                            │                  CLIENTS                            │
                            │         (Web Browser / Mobile App)                  │
                            └────────────────────┬────────────────────────────────┘
                                                 │ HTTPS
                                    ┌────────────▼────────────┐
                                    │       AWS CloudFront     │
                                    │   (CDN for Static Assets)│
                                    └────────────┬────────────┘
                                                 │
                                    ┌────────────▼────────────┐
                                    │      API Gateway         │
                                    │  • Routing               │
                                    │  • Rate Limiting         │
                                    │  • Auth (JWT Validation) │
                                    │  • Load Balancing        │
                                    └──┬──┬──┬──┬──┬──┬───────┘
                                       │  │  │  │  │  │
          ┌────────────────────────────┘  │  │  │  │  └─────────────────────┐
          │              ┌────────────────┘  │  │  └──────────────────┐     │
          │              │         ┌──────────┘  └──────────────┐     │     │
          ▼              ▼         ▼                             ▼     ▼     ▼
   ┌──────────┐  ┌──────────┐ ┌──────────┐  ┌──────────┐ ┌─────────┐ ┌──────────┐
   │  User    │  │ Search   │ │ Product  │  │  Cart    │ │ Order   │ │Checkout  │
   │ Service  │  │ Service  │ │ Service  │  │ Service  │ │ Status  │ │ Service  │
   └────┬─────┘  └────┬─────┘ └────┬─────┘  └────┬─────┘ │ Service │ └────┬─────┘
        │             │            │               │       └────┬────┘      │
        ▼             ▼            ▼               ▼            │           ▼
   ┌─────────┐  ┌──────────┐ ┌─────────┐  ┌──────────┐        │    ┌──────────────┐
   │  MySQL  │  │OpenSearch│ │  MySQL  │  │ MongoDB  │        │    │ Inventory    │
   │  Users  │  │  Index   │ │Products │  │  Carts   │        │    │  Service     │
   └─────────┘  └──────────┘ └────┬────┘  └──────────┘        │    └──────┬───────┘
                      ▲           │                             │           │
                      │      ┌────▼────────────────────┐        │    ┌──────▼───────┐
                      │      │   CDC PIPELINE           │        │    │ Redis Locks  │
                      │      │  Debezium Connector →   │        │    │ + PostgreSQL │
                      │      │  Kafka → OpenSearch      │        │    │  Inventory   │
                      │      └─────────────────────────┘        │    └──────────────┘
                      │                                          │
                      │                                   ┌──────▼──────┐
                      │                                   │   MySQL     │
                      └───────────────────────────────────│   Orders    │
                                                          └─────────────┘
                                                                │
                                                    ┌───────────▼──────────┐
                                                    │   Payment Service    │
                                                    └───────────┬──────────┘
                                                                │
                                                    ┌───────────▼──────────┐
                                                    │  Third-Party Gateway │
                                                    │  (Razorpay/Stripe)   │
                                                    └──────────────────────┘

  Async Event Flow (Kafka):
  Checkout Service → [order.created] → Order Service
  Checkout Service → [payment.success] → Inventory Service (decrement stock)
  Inventory Service → [stock.updated] → CDC → OpenSearch
```

---

## 5. Component Descriptions

### API Gateway
The single entry point for all client requests. Handles JWT validation (delegating to Auth Service for token issuance), request routing to downstream microservices, rate limiting per user/IP, and acts as a Layer-7 load balancer distributing traffic across service replicas.

### Auth / User Service
Manages registration, login, and profile management. On login, verifies username/password and issues a signed JWT. The JWT is passed in subsequent requests as a Bearer token; the API Gateway validates the signature without a DB call on every request.

### Search Service
Backed by Amazon OpenSearch (Elasticsearch). Uses inverted indexing and tokenization to enable full-text search with O(1) lookup time. The Search Service does not query MySQL directly — it reads from the OpenSearch index which is kept in sync via the CDC Pipeline.

### Product Service
Manages the product catalog in MySQL. On a product detail page request, it fetches product metadata. Product images are stored in AWS S3 and served via CloudFront CDN. The Product DB is the source of truth for the CDC Pipeline.

### Cart Service
Uses MongoDB for flexible document storage, accommodating products with varying attributes. Cart documents store CartID, UserID, and a list of (ProductID, Quantity) tuples. At checkout time, the Cart Service re-queries the Product Service to verify current prices (since cart items may be stale).

### Checkout Service
Orchestrates the multi-step transaction: verifying cart contents, confirming inventory availability (via Inventory Service), initiating payment (via Payment Service), and publishing order events to Kafka. Uses the Saga Pattern to handle partial failures gracefully.

### Inventory Service
A dedicated service responsible for stock management. Uses Redis distributed locks (with TTL) to serialize concurrent access to the same product's stock count during checkout. The authoritative stock count is stored in PostgreSQL with row-level locking for the final decrement.

### Payment Service
Wraps the third-party payment gateway. Implements idempotency keys to prevent duplicate charge on retries. Records payment attempts and final status in a MySQL Payment DB.

### Order Status Service
A read-heavy service that exposes order status to users. Reads from the Orders MySQL DB. Subscribes to Kafka order events to keep the order record up to date.

### Kafka (Message Queue)
Decouples the Checkout Service from downstream consumers. Key topics: `order.created`, `payment.success`, `payment.failed`, `stock.decremented`. Enables at-least-once delivery with idempotent consumers.

### CDC Pipeline (Change Data Capture)
Debezium watches the MySQL Product DB binlog. On any INSERT/UPDATE/DELETE in the Products table, Debezium produces a Kafka message. A Kafka consumer feeds these changes into OpenSearch in near-real-time (seconds of lag), keeping search results fresh without polling the DB.

---

## 6. Data Flow

### Flow 1: User Search

1. User types "iPhone 16" in the search bar.
2. Request hits API Gateway → routed to Search Service.
3. Search Service queries OpenSearch with tokenized query.
4. OpenSearch returns a ranked list of ProductIDs using inverted index.
5. Search Service returns paginated ProductID list to client.
6. Client fetches product thumbnails for each ProductID from Product Service (or CDN cache).

### Flow 2: Add to Cart

1. Authenticated user selects a product and quantity, clicks "Add to Cart."
2. Request with JWT hits API Gateway → Cart Service.
3. Cart Service upserts a cart document in MongoDB with the new item.
4. Returns CartID to the client.

### Flow 3: Checkout & Payment

1. User clicks "Proceed to Buy."
2. Checkout Service fetches cart from Cart Service.
3. Checkout Service calls Inventory Service to verify and reserve stock (Redis lock acquired).
4. Checkout Service calls Payment Service → third-party gateway processes payment.
5. On `payment.success`:
   - Checkout Service publishes `order.created` to Kafka.
   - Order Consumer creates an Order record in MySQL.
   - Inventory Consumer decrements the stock in PostgreSQL; Redis lock released.
6. On `payment.failure`:
   - Checkout Service publishes `payment.failed` to Kafka.
   - Inventory Consumer releases the Redis lock without decrementing stock.
   - User is notified of failure.

### Flow 4: Order Status Check

1. User navigates to "My Orders."
2. Request hits API Gateway → Order Status Service.
3. Order Status Service queries Orders MySQL DB by UserID.
4. Returns order list with current status.
