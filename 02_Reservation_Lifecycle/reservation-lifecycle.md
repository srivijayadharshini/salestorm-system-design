# Reservation Lifecycle

## 1. Purpose

A reservation temporarily holds inventory for a customer
while the payment process is completed.

## 2. Reservation States

AVAILABLE → RESERVED → PAYMENT_PENDING → CONFIRMED

## 3. Payment Failure

If payment fails:

PAYMENT_PENDING → PAYMENT_FAILED → RELEASED → AVAILABLE

The released inventory becomes available for other customers.

## 4. Reservation Expiry

If the customer does not complete payment before the reservation
expires:

RESERVED → EXPIRED → RELEASED → AVAILABLE

The inventory is returned to available stock.

## 5. Successful Payment

If payment succeeds:

RESERVED → PAYMENT_PENDING → CONFIRMED

The reservation becomes associated with the successful order.

## 6. Important Rule

Every reservation must have an expiry time so that inventory
cannot remain locked forever.