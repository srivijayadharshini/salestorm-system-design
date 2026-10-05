# Stress Scenarios

## 1. 10,000 Customers and 100 Stock

### Scenario

10,000 customers attempt to purchase a product with only 100
available units.

### Expected Behavior

The Inventory Service performs an atomic conditional update.

Only requests that successfully reserve available inventory are
allowed to continue.

### Expected Result

- Maximum successful reservations = 100
- Remaining inventory = 0
- Inventory never becomes negative
- Remaining customers receive OUT_OF_STOCK

---

## 2. Duplicate Payment Request

### Scenario

A customer clicks the Pay button twice.

### Expected Behavior

Both requests use the same Idempotency-Key.

The Payment Service processes the first request and returns the
existing result for the duplicate request.

### Expected Result

The customer is charged only once.

---

## 3. Payment Failure

### Scenario

The payment gateway rejects the payment.

### Expected Behavior

The payment is marked as failed and the inventory reservation is
released.

### Expected Result

The inventory becomes available for another customer.

---

## 4. Payment Timeout

### Scenario

The payment gateway does not respond within the timeout.

### Expected Behavior

The system does not blindly retry the payment.

It checks the payment status or performs reconciliation.

### Expected Result

If payment succeeded, continue the order workflow.

If payment failed, release the reservation.

---

## 5. Order Service Down

### Scenario

Payment succeeds but Order Service is unavailable for 30 seconds.

### Expected Behavior

PaymentSucceeded is stored in a durable message queue.

The event remains available while Order Service is down.

After recovery, Order Service consumes the event.

### Expected Result

The order is created without losing the successful payment.

---

## 6. Duplicate PaymentSucceeded Event

### Scenario

The same PaymentSucceeded event is delivered more than once.

### Expected Behavior

Order Service checks whether the event has already been processed.

### Expected Result

Only one order is created.

---

## 7. Payment Gateway Failure

### Scenario

The payment gateway repeatedly fails.

### Expected Behavior

The system performs limited retries with backoff.

If failures continue, the circuit breaker opens.

### Expected Result

Unnecessary calls to the failing gateway are stopped temporarily.

---

## 8. Message Processing Failure

### Scenario

Order Service repeatedly fails to process an event.

### Expected Behavior

The message is retried a limited number of times.

After repeated failure, it is moved to the Dead Letter Queue.

### Expected Result

The failed message is preserved for investigation and
reprocessing.

---

## 9. Final Reliability Guarantees

The system must guarantee:

- No overselling
- No negative inventory
- No duplicate payments
- No duplicate orders
- Successful payments are not lost
- Failed reservations are released
- Expired reservations return inventory
- Failed messages can be recovered