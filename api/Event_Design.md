# SALESTORM Event Design

## 1. Commands vs events

| Kind | Meaning | Examples |
|---|---|---|
| Command (synchronous API call to one owner) | "Do this" | ReserveInventory, CreateOrder, AuthorizePayment |
| Event (asynchronous message, already happened) | "This happened" — many consumers | PaymentSucceeded, OrderConfirmed |

## 2. Event catalogue and ownership

| Event | Producer (owner) | Consumers | Trigger |
|---|---|---|---|
| ReservationCreated | Inventory & Reservation | Checkout | stock reserved |
| ReservationReleased | Inventory & Reservation | Checkout, Order | released by customer |
| ReservationExpired | Inventory & Reservation | Order, Notification | `expires_at` passed |
| PaymentSucceeded | Payment | Inventory & Reservation, Order, Notification | provider success |
| PaymentFailed | Payment | Inventory & Reservation, Order, Notification | payment declined |
| PaymentTimedOut | Payment | Reconciliation | provider timeout |
| OrderCreated | Order | Notification | checkout |
| OrderConfirmed | Order | Shipment, Notification | after PaymentSucceeded |
| OrderCancelled | Order | Inventory & Reservation, Notification | payment failure / expiry |
| ShipmentCreated | Shipment | Order, Notification | order confirmed |
| ShipmentDelivered | Shipment | Order, Notification | delivery |

Rule: only the owning service publishes its events; consumers react and publish their own events. No service writes another service's tables.

## 3. Event envelope
```json
{
  "eventId": "evt-7f3a-4c1b",
  "eventType": "PaymentSucceeded",
  "version": 1,
  "occurredAt": "2026-10-05T12:05:10Z",
  "producer": "payment-service",
  "correlationId": "c-91f3",
  "payload": {
    "paymentId": 3001,
    "orderId": 7001,
    "reservationId": 5001,
    "amount": 1799.00,
    "transactionReference": "TXN-ABC123"
  }
}
```
`eventId` is the de-duplication key. The partition/ordering key is `orderId`, so events of one order are processed in order.

## 4. Event flow
```
Payment Service
   │ PaymentSucceeded
   ▼
Message Broker ──► Inventory & Reservation (reservation CONFIRMED, reserved → sold)
   ├────────────► Order Service (order CONFIRMED) ── OrderConfirmed ──► Message Broker
   └────────────► Notification                                            ├──► Shipment
                                                                          └──► Notification
```

## 5. Stock effect per event

| Event handled by Inventory & Reservation | available | reserved | sold |
|---|---|---|---|
| Reservation created (sync API) | −qty | +qty | – |
| PaymentSucceeded | – | −qty | +qty |
| PaymentFailed, ReservationExpired, ReservationReleased, OrderCancelled (before payment) | +qty | −qty | – |

## 6. Failure case: payment succeeded but Order Service is down

```
Payment SUCCESS → PaymentSucceeded published → Message Broker → Order Service DOWN
```
1. The broker keeps the event until it is acknowledged.
2. When Order Service restarts it consumes the event and confirms the order.
3. `OrderConfirmed` is then published to Shipment and Notification.

Safeguards:
- **Retry** with exponential backoff (1 s, 5 s, 30 s, 2 min, 10 min; max 5 attempts).
- **Dead Letter Queue**: messages that still fail go to `<topic>.dlq`, are monitored with alerts, and are replayable.
- **Idempotent consumers**: each consumer records processed `eventId`s (or checks the current state of the order/reservation) and skips duplicates, so replays never double-confirm stock or create two orders.
- **Reconciliation job** (every 1–5 minutes): finds payments with `SUCCESS` whose order is not `CONFIRMED`, and payments `PENDING`/`TIMEOUT` older than the threshold; checks the provider by `transaction_reference` and re-emits the event or compensates.
- **Compensation**: if payment succeeded but the order/stock cannot be honoured (for example the reservation already expired), refund the payment, publish `OrderCancelled` and notify the customer.

Recommended enhancement for reliable publishing: a transactional outbox table so the state change and the event are committed atomically. It is not part of the current ER diagram; add it if the team wants it.

## 7. Reservation expiry
A background worker in the Inventory & Reservation Service runs every 10–30 seconds:
1. Select reservations in status `RESERVED` with `expires_at < NOW()` (use `FOR UPDATE SKIP LOCKED` so several workers can run safely).
2. Set `EXPIRED`, then `RELEASED`; return `reserved → available`.
3. Publish `ReservationExpired`; Order cancels any pending order.

Late-payment race: if `PaymentSucceeded` arrives for an expired reservation, Inventory tries to re-reserve; if no stock remains, the payment is refunded and `OrderCancelled` is published.

## 8. Synchronous vs asynchronous communication

| Operation | Type | Reason |
|---|---|---|
| Product lookup | Sync | immediate read (cache allowed) |
| Add to cart | Sync | immediate feedback |
| Reserve inventory | Sync | customer needs an immediate yes/no; must be atomic with the stock update in MySQL |
| Checkout / order creation | Sync | customer needs the order id and amount |
| Payment authorization | Sync | customer needs the payment result |
| PaymentSucceeded → order confirmation | Async | decouples Payment from Order; survives an Order outage |
| Stock confirmation (reserved → sold) | Async | event-driven and idempotent |
| Shipment creation | Async | downstream work that need not block the customer |
| Notification | Async | must never block or fail a purchase |
| Reservation expiry | Async / background | time-driven |
| Reconciliation | Async / background | safety net |

Rule of thumb: use a synchronous call when the caller cannot continue without the answer; use an asynchronous event when the work is a consequence that can happen slightly later and must survive failures.
