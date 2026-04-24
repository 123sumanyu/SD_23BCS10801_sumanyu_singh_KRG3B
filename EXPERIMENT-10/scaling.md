# Scalability & Reliability Design
## E-Commerce Platform — Amazon/Flipkart-Like System

---

## 1. Load Balancing Strategy

### API Gateway as Layer-7 Load Balancer

The API Gateway (AWS API Gateway or Kong) acts as the primary entry point and distributes requests across service replica pools using a **Round Robin** algorithm by default. For stateful-aware routing (e.g., sticky sessions during payment), **Least Connections** is used.

### AWS Application Load Balancer (ALB) Per Service

Each microservice runs as a cluster of containerized replicas behind its own AWS ALB. The ALB performs:

- **Health Checks**: HTTP GET `/health` every 10 seconds; unhealthy instances are removed from the pool within 30 seconds.
- **Target Group Routing**: Routes based on path patterns (e.g., `/search/*` → Search Service target group).
- **Connection Draining**: In-flight requests on deregistered instances complete before the instance is fully removed.

### Geographic Load Balancing

AWS Route 53 with **latency-based routing** directs users to the nearest AWS region (e.g., Mumbai, Singapore, Frankfurt). Each region runs its own stack with data replication across regions for read services.

---

## 2. Horizontal vs. Vertical Scaling

### Policy by Service

| Service | Strategy | Reasoning |
|---|---|---|
| Search Service | Horizontal (auto-scale) | Stateless; scales linearly with query load |
| Product Service | Horizontal (auto-scale) | Stateless; reads are cache-friendly |
| Cart Service | Horizontal | Stateless; MongoDB handles distributed storage |
| Checkout Service | Horizontal (controlled) | Stateless per request; Kafka absorbs burst |
| Inventory Service | Horizontal + Redis sharding | Concurrent lock management requires careful coordination |
| Payment Service | Horizontal | Stateless gateway wrapper; idempotency prevents double-charge |
| Order Status Service | Horizontal (read replicas) | Purely read-heavy; offload to MySQL read replicas |
| MySQL (Users, Orders, Payments) | Vertical first, then read replicas | Write-heavy ACID workloads; replicas handle read traffic |
| OpenSearch | Horizontal (shard expansion) | Add shards as document count grows |
| Kafka | Horizontal (partition scaling) | Add partitions and consumer group members |

### Auto-Scaling Triggers (AWS ECS / EKS)

Auto-scaling rules based on CloudWatch metrics:

- **CPU Utilization > 70%** for 2 minutes → scale out (+2 replicas)
- **Request count per target > 1000/min** → scale out
- **CPU Utilization < 20%** for 10 minutes → scale in
- Minimum replicas: 2 per service (for high availability)
- Maximum replicas: 50 per service (cost guard)

---

## 3. Caching Strategy

### Layer 1: CDN (AWS CloudFront)

Product images and static assets (CSS, JS, thumbnails) are served from CloudFront edge locations. Cache-Control headers set TTL to 7 days for product images (image URLs contain a version hash; changing the image changes the URL, busting the cache).

### Layer 2: Redis Application Cache

Redis (AWS ElastiCache — cluster mode) is used for:

| Cache Key Pattern | Content | TTL | Invalidation |
|---|---|---|---|
| `product:{product_id}` | Full product detail response | 10 minutes | On product update via CDC event |
| `search:{query_hash}:{page}` | Search result page (ProductIDs) | 2 minutes | Time-based (search freshness vs. cost) |
| `inventory:count:{product_id}` | Available stock count | 30 seconds | On inventory decrement/increment |
| `user:{user_id}:cart` | Cart summary | 5 minutes | On cart mutation |
| `session:{jwt_jti}:blacklist` | Invalidated JWTs (logout) | Until JWT expiry | On logout |

### Cache-Aside Pattern

Product Service uses cache-aside: check Redis first; on miss, query MySQL, write to Redis, return. Product update events from Kafka trigger cache invalidation (`DEL product:{id}`).

### Write-Through for Inventory

To prevent stale stock counts during flash sales, inventory cache is write-through: every stock decrement writes atomically to both Redis (for fast reads) and PostgreSQL (for durability). The Redis DECR operation is atomic and prevents race conditions at the cache layer.

---

## 4. Database Scaling

### MySQL (Users, Products, Orders, Payments)

**Read Replicas**: Each MySQL DB has 2 read replicas. Read traffic (e.g., Order Status queries, Product lookups) is routed to replicas via a proxy (ProxySQL or AWS RDS Proxy). Write traffic (inserts, updates) goes to the primary.

**Vertical Scaling**: MySQL primaries start on db.r6g.2xlarge and scale up to db.r6g.16xlarge before considering sharding.

**Sharding (Future)**: Orders table will be range-sharded by `order_id` when it exceeds 500M rows. Shard key: `user_id % num_shards` for even distribution.

**Connection Pooling**: ProxySQL sits between services and MySQL, multiplexing thousands of application connections into a smaller pool of DB connections (typically 100–200 per primary), preventing connection exhaustion.

### PostgreSQL (Inventory)

Inventory DB uses **row-level locking** (`SELECT ... FOR UPDATE`) for the final stock decrement in conjunction with Redis locks. Configured with synchronous_commit = on to ensure durability.

Replication: Streaming replication to 1 standby (synchronous) for failover RTO < 30 seconds.

### MongoDB (Cart)

Cart data is sharded by `user_id` using hash-based sharding across 3 shard nodes. Each shard has a replica set (1 primary + 2 secondaries). Write concern: `w: majority` to ensure durability. Cart TTL index automatically expires documents after 30 days.

### OpenSearch (Search Index)

OpenSearch cluster: 3 master nodes + 6 data nodes. Products index: 6 primary shards + 1 replica shard each (12 shards total). New shards are added when average shard size exceeds 30 GB. Index lifecycle management (ILM) archives old product versions.

---

## 5. Bottlenecks & Optimizations

### Bottleneck 1: Flash Sale Race Condition (Inventory)

**Problem**: During a flash sale with 10,000 concurrent users competing for 100 units, naive DB reads lead to overselling. The problem: `SELECT qty → verify > 0 → UPDATE qty-1` is not atomic across concurrent requests.

**Solution**:
1. **Redis Distributed Lock with TTL**: Before any checkout involving the flash-sale product, the Inventory Service acquires `SET lock:product:{id} {order_id} NX PX 60000` (set-if-not-exists, 60-second TTL). Only one request holds the lock at a time.
2. **Atomic Redis DECR**: After acquiring the lock, check Redis counter `inventory:count:{id}`. If > 0, atomically DECR. Then asynchronously write the new count to PostgreSQL via Kafka.
3. **Queue Remaining Requests**: Requests that fail to acquire the lock within 2 retries (exponential backoff) receive `422 Insufficient Stock` immediately rather than queuing indefinitely.
4. **Result**: At most 100 successful purchases regardless of concurrent request count. TTL on the lock prevents deadlock if the service crashes mid-transaction.

### Bottleneck 2: Search Under High Load

**Problem**: At 100M DAU with peak browsing traffic, OpenSearch could be overwhelmed if every user search hits the cluster directly.

**Solution**:
- Redis caches popular search queries for 2 minutes (cache key: `search:{sha256(query+filters)}:{page}`). Repeat searches served from cache with sub-millisecond latency.
- OpenSearch cluster auto-scales data nodes based on CPU metrics.
- Search results are paginated (max 20 items/page) to reduce payload size and OpenSearch scoring overhead.

### Bottleneck 3: Checkout Multi-Step Failure (Consistency)

**Problem**: The checkout flow involves 3 steps (inventory check, payment, order creation). A payment may succeed but the order record may fail to be created, leaving the user charged with no order.

**Solution — Saga Pattern with Kafka**:
1. Checkout Service publishes `checkout.initiated` event and returns `order_id` immediately.
2. On `payment.success` from Payment Service, Kafka consumer creates the Order record and decrements inventory.
3. On `payment.failed`, Kafka consumer releases the inventory reservation.
4. If the Order Consumer fails (e.g., crashes), Kafka's consumer group offset mechanism ensures the event is reprocessed (at-least-once delivery). Order creation is idempotent (unique constraint on `order_id` prevents duplicates).
5. A dead-letter queue (DLT) captures events that fail after 3 retries for manual intervention.

### Bottleneck 4: Single Database for Multiple Services (Avoided by Design)

**Problem (identified in the original design)**: A single shared database across all services creates a coupling bottleneck — one slow query blocks others, and schema changes require coordination.

**Solution**: Each microservice owns its own database (Database-per-Service pattern). Cross-service data needs are satisfied via API calls (synchronous) or Kafka events (asynchronous), never direct DB joins across service boundaries.

---

## 6. Failure Handling

### Retry with Exponential Backoff

All inter-service HTTP calls (e.g., Cart → Product for price check) use:

- **3 retries** with backoff: 100ms, 200ms, 400ms
- **Jitter** added to prevent thundering herd: `delay = base_delay * 2^attempt + random(0, 100ms)`
- Retries only on `5xx` and network errors (not on `4xx` — those are client errors, not transient).

### Circuit Breaker (Resilience4j / Hystrix)

| Service Pair | Threshold | Wait Duration |
|---|---|---|
| Checkout → Inventory | 50% failure in 10 calls → OPEN | 30 seconds |
| Checkout → Payment | 30% failure in 10 calls → OPEN | 60 seconds |
| Product → Redis Cache | 80% failure in 5 calls → OPEN (fallback to DB) | 10 seconds |

When the circuit is OPEN, fallback behavior is triggered (e.g., return cached data, return "currently unavailable" error) rather than cascading failures.

### Kafka Consumer Failure Handling

- **At-Least-Once Delivery**: Kafka commits offsets only after the consumer successfully processes a message.
- **Dead Letter Topic (DLT)**: After 3 processing failures, the message is routed to `{topic}.DLT` for alerting and manual replay.
- **Idempotent Consumers**: Order creation and inventory decrement use idempotency keys (`order_id`, `payment_id`) with unique DB constraints to safely handle replayed events.

### Database Failover

- MySQL: AWS RDS Multi-AZ deployment. Automatic failover to standby in < 60 seconds if primary fails.
- PostgreSQL (Inventory): Synchronous streaming replica. Manual failover promoted to primary with < 30s RTO.
- Redis: ElastiCache cluster mode with automatic shard failover. Replica promoted on primary shard failure.
- MongoDB: Replica set auto-elects new primary within 10 seconds of primary failure.

### Graceful Degradation

- If Search Service / OpenSearch is down: Product Service falls back to MySQL FULLTEXT search (slower but functional).
- If Inventory Service is unavailable during checkout: Checkout Service rejects the request with `503 Service Unavailable` rather than proceeding without stock validation.
- If Payment Gateway is down: Checkout Service returns `502 Bad Gateway` and the inventory reservation lock expires via TTL (no stock decrement, no orphaned locks).
