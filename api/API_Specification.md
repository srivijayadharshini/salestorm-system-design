# SALESTORM REST API Specification

Base path: `/api/v1` — JSON over HTTPS — all requests go through the **API Gateway**.

## 1. Conventions

### Authentication
- `Authorization: Bearer <JWT>` is required on every endpoint except `POST /auth/register`, `POST /auth/login` and public product reads (`GET /products...`).
- The customer is identified from the token. A `customerId` in a body must match the token, otherwise `403 FORBIDDEN`.
- Missing, invalid or expired token → `401 UNAUTHORIZED`.

### Headers
| Header | Used on | Meaning |
|---|---|---|
| `Authorization` | protected endpoints | JWT bearer token |
| `Idempotency-Key` | `POST /reservations`, `POST /checkout`, `POST /payments` | Same key + same request → same result, no second side effect |
| `X-Correlation-Id` | optional, all | Trace id across services |

### Error format
```json
{
  "error": "INSUFFICIENT_INVENTORY",
  "message": "Product is no longer available",
  "timestamp": "2026-10-05T12:00:00Z",
  "path": "/api/v1/reservations"
}
```

### Status codes
| Code | Use |
|---|---|
| 200 | Successful read, or replay of an idempotent request |
| 201 | Resource created |
| 202 | Accepted, completes asynchronously (e.g. payment timeout) |
| 400 | Validation error / malformed request |
| 401 | Not authenticated |
| 402 | Payment declined |
| 403 | Resource belongs to another customer |
| 404 | Resource not found |
| 409 | Conflict (stock exhausted, wrong state) |
| 410 | Reservation expired |
| 422 | Idempotency key reused with a different request body |
| 429 | Rate limit exceeded (`Retry-After` header) |
| 500 / 503 | Server error / dependency unavailable — retry with the same Idempotency-Key |

### Error codes
`VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `PRODUCT_NOT_FOUND`, `PRODUCT_NOT_ACTIVE`, `SALE_NOT_ACTIVE`, `INSUFFICIENT_INVENTORY`, `RESERVATION_NOT_FOUND`, `RESERVATION_EXPIRED`, `INVALID_RESERVATION_STATE`, `IDEMPOTENCY_KEY_REUSED`, `ORDER_NOT_FOUND`, `INVALID_ORDER_STATE`, `AMOUNT_MISMATCH`, `PAYMENT_NOT_FOUND`, `PAYMENT_DECLINED`, `PAYMENT_TIMEOUT`, `RATE_LIMITED`, `SERVICE_UNAVAILABLE`.

## 2. Endpoint summary

| # | Method | Endpoint | Service | Auth | Idempotency-Key |
|---|---|---|---|---|---|
| 1 | POST | `/auth/register`, `/auth/login` | Gateway/Auth | No | – |
| 2 | GET | `/products/{productId}` | Product | No | – |
| 3 | GET | `/products?categoryId={id}` | Product | No | – |
| 4 | GET | `/sales/active` | Sale | No | – |
| 5 | POST | `/cart/items` | Cart | Yes | – |
| 6 | GET | `/cart` | Cart | Yes | – |
| 7 | **POST** | **`/reservations`** | Inventory & Reservation | Yes | **Required** |
| 8 | GET | `/reservations/{reservationId}` | Inventory & Reservation | Yes | – |
| 9 | POST | `/reservations/{reservationId}/release` | Inventory & Reservation | Yes | – |
| 10 | POST | `/checkout` | Checkout | Yes | Required |
| 11 | POST | `/payments` | Payment | Yes | **Required** |
| 12 | GET | `/payments/{paymentId}` | Payment | Yes | – |
| 13 | GET | `/orders/{orderId}` | Order | Yes | – |
| 14 | GET | `/orders/{orderId}/shipment` | Shipment | Yes | – |

> **Order creation:** the order is created by `POST /checkout` with status `PAYMENT_PENDING`, *before* payment, because a payment must reference an `orderId`. The order becomes `CONFIRMED` when the `PaymentSucceeded` event is processed. Therefore there is no separate `POST /orders`.

## 3. Endpoints

### 3.1 Auth
`POST /auth/register` — `{ "name", "email", "password", "phone" }` → `201 { "customerId": 1001, "email": "..." }`; `409 EMAIL_EXISTS`.
`POST /auth/login` — `{ "email", "password" }` → `200 { "accessToken": "<jwt>", "expiresIn": 3600 }`; `401 INVALID_CREDENTIALS`.

### 3.2 GET /products/{productId}  (Product Service; may be served from Redis cache)
**200**
```json
{
  "productId": 1,
  "name": "Flash Sale Phone",
  "price": 1999.00,
  "currency": "INR",
  "categoryId": 10,
  "status": "ACTIVE",
  "salePrice": 1799.00,
  "saleEndsAt": "2026-10-05T18:00:00Z"
}
```
Errors: `404 PRODUCT_NOT_FOUND`.
Stock shown to users is informational (cache). Only a successful reservation guarantees stock.

### 3.3 POST /cart/items  (Cart Service)
```json
{ "productId": 1, "quantity": 1 }
```
**201** `{ "cartId": 55, "items": [ { "cartItemId": 900, "productId": 1, "quantity": 1, "unitPrice": 1799.00 } ] }`
Errors: `400 VALIDATION_ERROR`, `404 PRODUCT_NOT_FOUND`, `409 PRODUCT_NOT_ACTIVE`.

### 3.4 POST /reservations  ⭐ (Inventory & Reservation Service)
Headers: `Authorization: Bearer <token>`, `Idempotency-Key: RES-1001-001`
```json
{ "productId": 1, "customerId": 1001, "quantity": 1 }
```
**201 Created**
```json
{
  "reservationId": 5001,
  "productId": 1,
  "quantity": 1,
  "status": "RESERVED",
  "expiresAt": "2026-10-05T12:10:00Z"
}
```
**200 OK** — same `Idempotency-Key` replayed: the original reservation is returned, stock is not reserved again.

| Status | error | Reason |
|---|---|---|
| 400 | VALIDATION_ERROR | quantity < 1 or above the per-customer limit |
| 401 | UNAUTHORIZED | invalid token |
| 404 | PRODUCT_NOT_FOUND | |
| 409 | SALE_NOT_ACTIVE | outside the sale window |
| 409 | **INSUFFICIENT_INVENTORY** | `{"error":"INSUFFICIENT_INVENTORY","message":"Product is no longer available"}` |
| 422 | IDEMPOTENCY_KEY_REUSED | same key, different body |
| 429 | RATE_LIMITED | |

Behaviour: one DB transaction using the conditional update `available_quantity >= :qty` (see `database/Database_Schema.md`). Guarantee: with 100 units, no more than 100 reservations can succeed; Redis is never consulted to decide availability.

### 3.5 GET /reservations/{reservationId}
**200** `{ "reservationId": 5001, "productId": 1, "quantity": 1, "status": "RESERVED", "expiresAt": "2026-10-05T12:10:00Z" }`
Statuses: `RESERVED`, `CONFIRMED`, `EXPIRED`, `RELEASED`. Errors: `404 RESERVATION_NOT_FOUND`, `403 FORBIDDEN`.

### 3.6 POST /reservations/{reservationId}/release
No body. **200** `{ "reservationId": 5001, "status": "RELEASED" }`
Errors: `404`, `403`, `409 INVALID_RESERVATION_STATE` (already CONFIRMED). Releasing an already released/expired reservation returns `200` (idempotent).

### 3.7 POST /checkout  (Checkout Service)
Headers: `Idempotency-Key: CHK-5001`
```json
{ "reservationId": 5001 }
```
**201 Created**
```json
{
  "orderId": 7001,
  "status": "PAYMENT_PENDING",
  "reservationId": 5001,
  "totalAmount": 1799.00,
  "currency": "INR",
  "payBefore": "2026-10-05T12:10:00Z"
}
```
Errors: `404 RESERVATION_NOT_FOUND`, `410 RESERVATION_EXPIRED`, `409 INVALID_RESERVATION_STATE`.
Checkout validates the reservation, creates the order and order items, and marks the cart checked out.

### 3.8 POST /payments  (Payment Service)
Headers: `Authorization`, `Idempotency-Key: PAY-ORD-7001`
```json
{ "orderId": 7001, "amount": 1799.00, "currency": "INR", "paymentMethodToken": "tok_abc" }
```
**201 Created — success**
```json
{ "paymentId": 3001, "orderId": 7001, "transactionReference": "TXN-ABC123", "status": "SUCCESS" }
```
**200 OK** — replay of the same key returns the stored payment; the customer is never charged twice.

**402 — declined**
```json
{ "paymentId": 3001, "status": "FAILED", "reason": "PAYMENT_DECLINED" }
```
**202 — provider timeout**: `{ "paymentId": 3001, "status": "TIMEOUT" }`; reconciliation resolves it; the client polls `GET /payments/{paymentId}`.

| Status | error | Reason |
|---|---|---|
| 400 | AMOUNT_MISMATCH / VALIDATION_ERROR | amount differs from the order total |
| 404 | ORDER_NOT_FOUND | |
| 409 | INVALID_ORDER_STATE | order not PAYMENT_PENDING or already paid |
| 410 | RESERVATION_EXPIRED | hold lapsed before payment |
| 422 | IDEMPOTENCY_KEY_REUSED | |
| 503 | SERVICE_UNAVAILABLE | provider down; retry with the same key |

After the outcome is stored, the Payment Service publishes `PaymentSucceeded` / `PaymentFailed` / `PaymentTimedOut` to the Message Broker (see `Event_Design.md`).

### 3.9 GET /payments/{paymentId}
**200** `{ "paymentId": 3001, "orderId": 7001, "amount": 1799.00, "currency": "INR", "status": "SUCCESS", "transactionReference": "TXN-ABC123" }`
Payment statuses: `PENDING`, `SUCCESS`, `FAILED`, `TIMEOUT`, `REFUNDED`. Errors: `404 PAYMENT_NOT_FOUND`, `403 FORBIDDEN`.

### 3.10 GET /orders/{orderId}  (Order Service)
**200**
```json
{
  "orderId": 7001,
  "status": "CONFIRMED",
  "items": [ { "productId": 1, "quantity": 1, "unitPrice": 1799.00, "subtotal": 1799.00 } ],
  "totalAmount": 1799.00,
  "currency": "INR",
  "createdAt": "2026-10-05T12:03:00Z"
}
```
Order statuses: `CREATED`, `PAYMENT_PENDING`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `OUT_FOR_DELIVERY`, `DELIVERED`, `CANCELLED`.
Shortly after a successful payment the order may still read `PAYMENT_PENDING` (confirmation is asynchronous); clients poll.

### 3.11 GET /orders/{orderId}/shipment  (Shipment Service)
**200** `{ "shipmentId": 1, "trackingNumber": "TRK123", "carrier": "BlueDart", "status": "SHIPPED" }` — `404` until a shipment exists.

## 4. Idempotency rules
1. The key is stored with the result (`INVENTORY_RESERVATION.idempotency_key`, `PAYMENT.idempotency_key`; both UNIQUE).
2. Same key and same body → original response is returned (`200`), no new side effect.
3. Same key with a different body → `422 IDEMPOTENCY_KEY_REUSED`.
4. `PAYMENT.transaction_reference` is a second UNIQUE guard against duplicate payments.
5. The client generates one key per user action and reuses it on every network retry.

## 5. Request flow
```
Client → API Gateway
  1. GET /products/{id}            Product Service (cache allowed)
  2. POST /cart/items              Cart Service
  3. POST /reservations            Inventory & Reservation  (sync, MySQL transaction)
  4. POST /checkout                Checkout  → creates order PAYMENT_PENDING
  5. POST /payments                Payment   (sync result)
  6. PaymentSucceeded → Message Broker → Order (CONFIRMED) → Shipment, Notification
```

## 6. Non-functional notes
- Rate limiting at the API Gateway, especially on `/reservations` and `/payments`.
- Clients retry timeouts with the **same** Idempotency-Key.
- Timestamps are ISO-8601 UTC; money is a decimal number with a currency code.
