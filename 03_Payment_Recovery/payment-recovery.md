# Payment Failure and Recovery

## 1. Payment Success

The customer initiates payment for a valid reservation.

The Payment Service communicates with the payment gateway.

If payment succeeds, the Payment Service records the successful
payment and publishes a PaymentSucceeded event.

The event is sent to a durable message queue.

The Order Service consumes the event and creates the order.

## 2. Payment Failure

If payment fails:

Payment → PAYMENT_FAILED → Release Reservation

The reserved inventory is returned to available inventory.

## 3. Payment Timeout

A payment timeout does not necessarily mean that payment failed.

The system must not blindly retry the payment because the original
payment may have succeeded.

The Payment Service checks the payment status with the payment
gateway or performs reconciliation.

If payment succeeded, the order workflow continues.

If payment failed, the reservation is released.

## 4. Order Service Failure

If payment succeeds but the Order Service is unavailable, the
PaymentSucceeded event remains in the durable message queue.

When the Order Service recovers, it consumes the event and creates
the order.

This prevents successful payments from being lost.

## 5. Duplicate Events

Message delivery may result in duplicate events.

The Order Service must use idempotency to ensure that processing
the same PaymentSucceeded event multiple times does not create
multiple orders.

## 6. Key Reliability Rule

Payment success must never be lost, and a customer must never be
charged twice because of a retry or timeout.