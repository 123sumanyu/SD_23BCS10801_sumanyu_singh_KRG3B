# API Design
## E-Commerce Platform — REST API Specification

---

## Conventions

- **Base URL**: `https://api.shopx.com/v1`
- **Versioning Strategy**: URI versioning (`/v1/`, `/v2/`). Major breaking changes increment the version. Old versions are deprecated with a 6-month sunset period announced via a `Deprecation` response header.
- **Authentication**: All endpoints (except `/auth/*`) require `Authorization: Bearer <JWT>` header. The JWT is validated at the API Gateway.
- **Content-Type**: All request and response bodies use `application/json`.
- **Pagination**: All list endpoints support `?page=0&size=20`. Defaults: page=0, size=20, max size=100.
- **Timestamps**: All timestamps are ISO 8601 UTC strings (e.g., `2025-06-01T10:30:00Z`).
- **Error Format**: All errors return a standard error envelope (see Error Format section).

---

## Standard Error Format

```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product with ID 999 does not exist.",
    "timestamp": "2025-06-01T10:30:00Z",
    "path": "/v1/products/999"
  }
}
```

---

## Status Codes Used

| Code | Meaning | When Used |
|---|---|---|
| 200 OK | Success | GET, PUT, DELETE |
| 201 Created | Resource created | POST that creates a new resource |
| 204 No Content | Success, no body | DELETE |
| 400 Bad Request | Invalid input | Validation failure |
| 401 Unauthorized | Missing / invalid JWT | Token absent or expired |
| 403 Forbidden | Authenticated but not allowed | Accessing another user's data |
| 404 Not Found | Resource not found | Product/Order/Cart not found |
| 409 Conflict | Duplicate resource | Duplicate idempotency key |
| 422 Unprocessable Entity | Business rule violation | Insufficient stock |
| 429 Too Many Requests | Rate limit exceeded | |
| 500 Internal Server Error | Unexpected server error | |
| 502 Bad Gateway | Upstream service error | Payment gateway unreachable |

---

## Rate Limiting Strategy

Rate limiting is enforced at the API Gateway using a token bucket algorithm per `user_id` (authenticated) or `IP` (unauthenticated).

| Endpoint Group | Limit |
|---|---|
| `GET /products/*`, `GET /search` | 1000 req/min per user |
| `POST /cart/*`, `PUT /cart/*` | 100 req/min per user |
| `POST /checkout`, `POST /payment` | 10 req/min per user |
| `POST /auth/login` | 5 req/min per IP (brute-force protection) |

When exceeded, the API returns `429 Too Many Requests` with headers:
```
Retry-After: 30
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1717235460
```

---

## Idempotency Handling

For mutating endpoints (`POST /checkout`, `POST /payment`), clients must include an `Idempotency-Key` header (UUIDv4). The server stores the response for this key for 24 hours. If the same key is received again (e.g., on a client retry), the stored response is returned without reprocessing.

```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

---

## API Endpoints

---

### AUTH SERVICE

#### POST /auth/register
Register a new user.

**Request Body:**
```json
{
  "name": "Rohan Sharma",
  "email": "rohan@example.com",
  "password": "SecurePass@123",
  "phone": "9876543210"
}
```

**Response 201:**
```json
{
  "user_id": 42,
  "name": "Rohan Sharma",
  "email": "rohan@example.com",
  "created_at": "2025-06-01T10:00:00Z"
}
```

---

#### POST /auth/login
Authenticate a user and receive a JWT.

**Request Body:**
```json
{
  "email": "rohan@example.com",
  "password": "SecurePass@123"
}
```

**Response 200:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Errors:** `401` invalid credentials, `429` rate limit.

---

### SEARCH SERVICE

#### GET /products/search
Search products by keyword. Returns paginated list of product summaries.

**Query Parameters:**
- `q` (required): Search keyword (e.g., `iPhone 16`)
- `category` (optional): Filter by category
- `min_price`, `max_price` (optional): Price range filter
- `sort` (optional): `relevance` (default), `price_asc`, `price_desc`, `rating`
- `page`, `size`: Pagination

**Request:**
```
GET /v1/products/search?q=iPhone+16&category=mobiles&sort=price_asc&page=0&size=20
Authorization: Bearer <JWT>
```

**Response 200:**
```json
{
  "query": "iPhone 16",
  "total_results": 85,
  "page": 0,
  "size": 20,
  "products": [
    {
      "product_id": 17,
      "name": "Apple iPhone 16 128GB",
      "price": 89999.00,
      "rating": 4.5,
      "review_count": 1243,
      "image_url": "https://cdn.shopx.com/products/iphone16.jpg",
      "in_stock": true
    }
  ]
}
```

**Errors:** `400` missing `q` param.

---

### PRODUCT SERVICE

#### GET /products/{product_id}
Get full product details.

**Request:**
```
GET /v1/products/17
Authorization: Bearer <JWT>
```

**Response 200:**
```json
{
  "product_id": 17,
  "name": "Apple iPhone 16 128GB",
  "description": "A17 Pro chip, 6.1-inch Super Retina XDR display...",
  "category": "mobiles",
  "price": 89999.00,
  "seller": {
    "seller_id": 5,
    "name": "Apple India Official Store"
  },
  "images": [
    "https://cdn.shopx.com/products/iphone16-1.jpg",
    "https://cdn.shopx.com/products/iphone16-2.jpg"
  ],
  "average_rating": 4.5,
  "review_count": 1243,
  "in_stock": true,
  "available_quantity": 48
}
```

**Errors:** `404` product not found.

---

### CART SERVICE

#### POST /cart/items
Add an item to the cart (creates cart if none exists).

**Request Body:**
```json
{
  "product_id": 17,
  "quantity": 1
}
```

**Request Headers:**
```
Authorization: Bearer <JWT>
```

**Response 200:**
```json
{
  "cart_id": "cart_101",
  "item_count": 2,
  "total_price": 114998.00
}
```

**Errors:** `404` product not found, `422` quantity exceeds available stock.

---

#### GET /cart
Get the current user's cart.

**Response 200:**
```json
{
  "cart_id": "cart_101",
  "user_id": 42,
  "items": [
    {
      "product_id": 17,
      "product_name": "Apple iPhone 16 128GB",
      "quantity": 1,
      "unit_price": 89999.00,
      "image_url": "https://cdn.shopx.com/products/iphone16.jpg"
    },
    {
      "product_id": 22,
      "product_name": "AirPods Pro 2nd Gen",
      "quantity": 1,
      "unit_price": 24999.00,
      "image_url": "https://cdn.shopx.com/products/airpods.jpg"
    }
  ],
  "subtotal": 114998.00,
  "expires_at": "2025-07-01T10:00:00Z"
}
```

---

#### PUT /cart/items/{product_id}
Update quantity of a cart item.

**Request Body:**
```json
{
  "quantity": 2
}
```

**Response 200:**
```json
{
  "cart_id": "cart_101",
  "updated_item": {
    "product_id": 17,
    "quantity": 2
  },
  "total_price": 204997.00
}
```

**Errors:** `404` item not in cart, `422` quantity > stock.

---

#### DELETE /cart/items/{product_id}
Remove an item from the cart.

**Response 204:** (No content)

**Errors:** `404` item not in cart.

---

### CHECKOUT SERVICE

#### POST /checkout
Initiate checkout. Verifies cart, validates stock, and creates a pending order.

**Request Headers:**
```
Authorization: Bearer <JWT>
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

**Request Body:**
```json
{
  "cart_id": "cart_101",
  "address_id": 7,
  "payment_mode": "UPI"
}
```

**Response 200:**
```json
{
  "order_id": 5001,
  "total_price": 114998.00,
  "items": [
    { "product_id": 17, "quantity": 1, "unit_price": 89999.00 },
    { "product_id": 22, "quantity": 1, "unit_price": 24999.00 }
  ],
  "payment_url": "https://pay.razorpay.com/session/xyz123",
  "expires_at": "2025-06-01T10:15:00Z"
}
```

**Errors:** `409` idempotency conflict, `422` insufficient stock.

---

### PAYMENT SERVICE

#### POST /payment
Confirm and process payment for an order.

**Request Headers:**
```
Authorization: Bearer <JWT>
Idempotency-Key: 660e8400-e29b-41d4-a716-446655440111
```

**Request Body:**
```json
{
  "order_id": 5001,
  "payment_mode": "UPI",
  "gateway_token": "pay_token_from_razorpay"
}
```

**Response 200:**
```json
{
  "payment_id": 9001,
  "order_id": 5001,
  "status": "SUCCESS",
  "amount": 114998.00,
  "gateway_ref_id": "rzp_live_abc123",
  "paid_at": "2025-06-01T10:05:00Z"
}
```

**Response (failure):**
```json
{
  "payment_id": 9001,
  "order_id": 5001,
  "status": "FAILED",
  "error": "Insufficient funds"
}
```

**Status Codes:** `200` for both success and failure (payment outcome is in body), `402` for system-level payment gateway error, `409` idempotency conflict.

---

### ORDER STATUS SERVICE

#### GET /orders
Get all orders for the authenticated user.

**Query Parameters:** `status` (optional), `page`, `size`

**Response 200:**
```json
{
  "total": 3,
  "orders": [
    {
      "order_id": 5001,
      "status": "SHIPPED",
      "total_price": 114998.00,
      "item_count": 2,
      "created_at": "2025-06-01T10:05:00Z",
      "estimated_delivery": "2025-06-05"
    }
  ]
}
```

---

#### GET /orders/{order_id}
Get detailed status of a specific order.

**Response 200:**
```json
{
  "order_id": 5001,
  "user_id": 42,
  "status": "SHIPPED",
  "total_price": 114998.00,
  "items": [
    { "product_id": 17, "product_name": "Apple iPhone 16 128GB", "quantity": 1, "unit_price": 89999.00 },
    { "product_id": 22, "product_name": "AirPods Pro 2nd Gen", "quantity": 1, "unit_price": 24999.00 }
  ],
  "shipping_address": {
    "street": "12, MG Road",
    "city": "Bengaluru",
    "state": "Karnataka",
    "pincode": "560001"
  },
  "payment": {
    "payment_id": 9001,
    "status": "SUCCESS",
    "mode": "UPI",
    "paid_at": "2025-06-01T10:05:00Z"
  },
  "tracking": {
    "carrier": "Blue Dart",
    "tracking_id": "BD123456789IN",
    "estimated_delivery": "2025-06-05"
  },
  "created_at": "2025-06-01T10:05:00Z"
}
```

**Errors:** `404` order not found, `403` order belongs to another user.
