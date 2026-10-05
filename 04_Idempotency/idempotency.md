# Idempotency Strategy

## 1. Problem

A customer may accidentally submit the same payment request
multiple times.

Network retries may also cause the same request to reach the
Payment Service more than once.

Without idempotency, this could result in duplicate payments.

## 2. Idempotency Key

Each payment request contains a unique Idempotency-Key.

Example:

Idempotency-Key: ABC123

The Payment Service stores the payment result associated with
this key.

## 3. Duplicate Payment Request

If the same Idempotency-Key is received again:

- The Payment Service does not execute the payment again.
- The previously stored result is returned.
- The customer is not charged twice.

## 4. Order Idempotency

PaymentSucceeded events may also be delivered more than once.

The Order Service checks whether the event has already been
processed.

If the event was already processed, the duplicate event is ignored.

If it has not been processed, the Order Service creates the order.

## 5. Reliability Rule

The same logical payment or order operation must produce the same
final result even if the request or event is received multiple times.