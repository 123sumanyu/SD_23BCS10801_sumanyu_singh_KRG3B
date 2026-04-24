# E-Commerce Platform System Design
### Amazon / Flipkart-Like Online Shopping System

---

## Project Overview

This project presents a complete system design for a production-grade, distributed e-commerce platform capable of handling 100 Million Daily Active Users (DAU) and processing 10 orders per second. The design covers everything from high-level microservices architecture to low-level database schema, API contracts, and scalability strategies.

The system enables users to search for products, view details, manage carts, check out, make payments, and track orders — all under strict latency, availability, and consistency requirements.

---

## Assumptions

- Users are authenticated via JWT tokens issued by the Auth service.
- Product catalog is pre-loaded and updated via a seller/admin portal (not in scope here).
- Payment processing is delegated to a third-party gateway (e.g., Razorpay, Stripe).
- Images and media assets are stored on AWS S3 / BLOB storage, not in the database.
- The system is deployed on AWS with managed services (RDS, ElastiCache, MSK, OpenSearch).
- Flash sales are infrequent but must be handled with strong consistency via distributed locks.
- Cart data has a TTL of 30 days; prices are re-verified at checkout time.
- Sellers have a separate management portal; buyer-facing flows are the primary focus.

---

## Tech Stack & Justification

| Layer | Technology | Justification |
|---|---|---|
| API Gateway | AWS API Gateway / Kong | Rate limiting, routing, auth enforcement at edge |
| Auth Service | Node.js + JWT + bcrypt | Stateless token-based auth; fast verification |
| User Service | Node.js + MySQL | Relational, consistent user profile data |
| Search Service | Amazon OpenSearch (Elasticsearch) | Full-text search, inverted indexing, O(1) lookup |
| Product Service | Node.js + MySQL | Strong consistency for product catalog |
| Cart Service | Node.js + MongoDB | Flexible schema for varied product attributes |
| Checkout Service | Java / Node.js + MySQL | ACID transactions for order consistency |
| Inventory Service | Node.js + Redis + PostgreSQL | Redis for distributed locks; Postgres for stock counts |
| Payment Service | Node.js + MySQL | Idempotent payment records |
| Order Status Service | Node.js + MySQL | Simple read-heavy service for status tracking |
| Message Queue | Apache Kafka | Async decoupling between Checkout, Inventory, Orders |
| CDC Pipeline | Debezium + Kafka | Real-time sync from Product DB to Elasticsearch |
| Cache Layer | Redis | Product details cache, session cache, distributed locks |
| Image Storage | AWS S3 | BLOB storage for product images, CDN-served |
| CDN | AWS CloudFront | Low-latency image and static asset delivery |
| Load Balancer | AWS ALB | Layer-7 load balancing across service replicas |
| Monitoring | Prometheus + Grafana + ELK | Metrics, logs, alerting |

---

## Trade-offs Taken

1. **Availability over Consistency for Search**: Product search uses Elasticsearch which may have slight replication lag from CDC. This is acceptable since a few seconds of stale search results do not harm the user experience significantly.

2. **Consistency over Availability for Checkout & Inventory**: ACID transactions and Redis distributed locks are used here, potentially increasing latency slightly. This is necessary to prevent overselling.

3. **MongoDB for Cart (Schema Flexibility vs. Join Performance)**: Cart items vary in attributes per product category. MongoDB's flexible document model fits better than forcing normalization in MySQL, even though joins across services require extra API calls.

4. **Kafka for Async Order Processing**: Using an event queue introduces eventual consistency between order placement and inventory decrement, but massively improves throughput and fault isolation.

5. **Redis TTL Locks vs. Database Pessimistic Locks**: Redis distributed locks (with TTL) are used during flash sales instead of DB-level pessimistic locking to avoid connection pool exhaustion at scale.

6. **Separate Inventory Service**: Rather than baking inventory checks into the Product Service, a dedicated Inventory Service is used. This increases the number of network calls but provides cleaner separation, independent scalability, and dedicated locking logic.

---

## Future Improvements

- **Recommendation Engine**: Integrate ML-based product recommendations using user purchase history and browsing behavior.
- **GraphQL API**: Replace some REST endpoints with GraphQL for flexible front-end data fetching.
- **Seller Portal**: Full-fledged seller onboarding, product listing, and analytics module.
- **Real-Time Notifications**: WebSocket or Server-Sent Events for order status updates.
- **A/B Testing Framework**: Feature flags and experiment management for UI and pricing experiments.
- **Multi-Region Deployment**: Geo-distributed deployments with latency-based routing for global users.
- **AI-Powered Fraud Detection**: ML model to flag suspicious payment patterns in real time.
- **Returns & Refunds Module**: End-to-end return initiation, logistics tracking, and refund processing.

---

## Repository Structure

```
/ecommerce-system-design
├── README.md       ← Project overview, assumptions, tech stack, trade-offs
├── HLD.md          ← High-Level Design, architecture diagram, data flow
├── LLD.md          ← Low-Level Design, DB schema, class diagrams, sequences
├── api.md          ← REST API contracts, request/response, status codes
└── scaling.md      ← Scalability, caching, sharding, failure handling
```
