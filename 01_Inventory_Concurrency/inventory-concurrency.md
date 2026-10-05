# Inventory Concurrency and Overselling Prevention

## 1. Problem

SALESTORM may receive thousands of customers trying to purchase the
same product at the same time.

Example:

- Available stock: 100 units
- Concurrent customers: 10,000

The system must ensure that no more than 100 units are successfully
reserved or sold.

## 2. Overselling Risk

A simple check-then-update approach can cause overselling.

Example:

1. Customer A checks stock and sees 1 unit.
2. Customer B checks stock and also sees 1 unit.
3. Both customers attempt to purchase.
4. Both may succeed if the check and stock update are separate operations.

Therefore, the inventory check and stock reduction must be atomic.

## 3. Atomic Inventory Reservation

The system performs a conditional database update:

UPDATE inventory
SET available_quantity = available_quantity - 1,
    reserved_quantity = reserved_quantity + 1
WHERE product_id = ?
AND available_quantity >= 1;

If one row is updated:

- Reservation succeeds.
- Available quantity decreases by 1.
- Reserved quantity increases by 1.

If zero rows are updated:

- There is not enough stock.
- Reservation fails.
- Customer receives an out-of-stock response.

## 4. Concurrency Rule

The inventory service must guarantee:

> A reservation succeeds only when sufficient inventory is available.

This prevents two concurrent requests from reserving the same unit.

## 5. Expected Result

For 100 available units and 10,000 concurrent customers:

- Maximum successful reservations = 100
- Remaining available inventory = 0
- Additional customers receive OUT_OF_STOCK
- Inventory must never become negative