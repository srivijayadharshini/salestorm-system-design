# Database Constraints and Indexes

## Primary Keys

Every transactional table has a unique primary key (`customer_id`, `category_id`, `product_id`, `sale_id`, `deal_id`, `inventory_id`, `reservation_id`, `cart_id`, `cart_item_id`, `order_id`, `order_item_id`, `payment_id`, `shipment_id`, `notification_id`).

## Foreign Keys

Foreign keys maintain relationships between customers, products, inventory, reservations, carts, orders, payments and shipments.

| Child column | References |
|---|---|
| PRODUCT.category_id | CATEGORY |
| DEAL.sale_id, DEAL.product_id | SALE, PRODUCT |
| INVENTORY.product_id | PRODUCT |
| INVENTORY_RESERVATION.inventory_id, customer_id | INVENTORY, CUSTOMER |
| CART.customer_id | CUSTOMER |
| CART_ITEM.cart_id, product_id | CART, PRODUCT |
| ORDER.customer_id, reservation_id | CUSTOMER, INVENTORY_RESERVATION |
| ORDER_ITEM.order_id, product_id | ORDER, PRODUCT |
| PAYMENT.order_id | ORDER |
| SHIPMENT.order_id | ORDER |
| NOTIFICATION.customer_id, order_id | CUSTOMER, ORDER |

## Unique Constraints

The following values must be unique:

- Customer email
- Reservation idempotency key
- Payment idempotency key
- Payment transaction reference
- Shipment tracking number
- INVENTORY.product_id (one stock row per product)
- ORDER.reservation_id (one order per reservation)

## Check Constraints

- INVENTORY: `available_quantity >= 0`, `reserved_quantity >= 0`, `sold_quantity >= 0`
- INVENTORY_RESERVATION: `quantity > 0`
- CART_ITEM / ORDER_ITEM: `quantity > 0`
- PAYMENT: `amount > 0`
- PRODUCT: `price >= 0`
- SALE: `end_time > start_time`

## Important Indexes

Recommended indexes:

- PRODUCT(category_id)
- INVENTORY(product_id)
- INVENTORY_RESERVATION(inventory_id)
- INVENTORY_RESERVATION(customer_id)
- INVENTORY_RESERVATION(expires_at)
- INVENTORY_RESERVATION(status, expires_at) — expiry worker scan
- ORDER(customer_id)
- ORDER(status)
- PAYMENT(order_id)
- PAYMENT(idempotency_key)
- PAYMENT(transaction_reference)

## Inventory Consistency

The relational database is the authoritative inventory source.

Successful inventory reservations must never exceed available inventory.

Inventory updates must be performed transactionally.

The version field supports optimistic concurrency control.

Invariant: `available_quantity + reserved_quantity + sold_quantity` equals the original stock of the sale (for example 100) at all times, and no quantity may be negative.

Stock is reserved with a single conditional update (`... WHERE available_quantity >= :qty`), so two concurrent buyers can never both take the last unit.

## Transaction Boundaries

- Reservation creation and the stock update commit together or not at all.
- Payment outcome handling updates reservation, inventory and order through idempotent steps that can be retried.

## Idempotency

Reservation and payment requests use idempotency keys to prevent duplicate operations.

A retried request with the same key returns the original result and changes nothing.

## Audit

Every table carries `created_at` / `updated_at`; inventory changes also bump `version`.

## Cache Rule

Redis is only a cache.

A cache miss or stale cache value must not cause inventory to be oversold.
