# Security and Observability

## 1. Security

### HTTPS / TLS

All communication should use HTTPS/TLS to protect data in transit.

### Authentication

Authentication verifies the identity of the customer or service.

### Authorization

Authorization ensures that an authenticated user is allowed to
perform the requested operation.

### Rate Limiting

Rate limiting protects the system from excessive traffic and abuse.

### Input Validation

Requests should be validated before processing.

### Secrets Management

Sensitive credentials, API keys, and secrets must not be stored
directly in source code.

### Audit Logging

Important security and business actions should be recorded for
investigation and accountability.

## 2. Observability

### Metrics

Important metrics include:

- Requests per second
- Error rate
- P95/P99 latency
- Reservation success and failure
- Payment success and failure
- Message queue backlog
- Inventory mismatch

### Logs

Logs should contain useful correlation information such as:

- request_id
- user_id
- reservation_id
- payment_id
- order_id
- timestamp
- status
- error information

### Distributed Tracing

Tracing follows a request across services:

API Gateway → Inventory Service → Payment Service → Order Service

## 3. Flash Sale Monitoring

For 100 available units and 10,000 concurrent customers:

- Successful reservations must be at most 100.
- Inventory must never become negative.
- Duplicate payments must be zero.
- Duplicate orders must be zero.