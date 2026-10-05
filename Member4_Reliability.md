# SALESTORM — Reliability, Concurrency and Failure Handling

## 1. Inventory Concurrency

SALESTORM may receive thousands of customers attempting to purchase
the same product simultaneously.

For example:

- Available stock = 100
- Concurrent customers = 10,000

The Inventory Service uses an atomic conditional database update.

```sql
UPDATE inventory
SET available_quantity = available_quantity - 1,
    reserved_quantity = reserved_quantity + 1
WHERE product_id = ?
AND available_quantity >= 1;