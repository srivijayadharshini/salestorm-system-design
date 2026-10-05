# Service-to-Database Mapping

| Service | Primary Data |
|---|---|
| Product Service | PRODUCT, CATEGORY |
| Sale Service | SALE, DEAL |
| Cart Service | CART, CART_ITEM |
| Inventory & Reservation Service | INVENTORY, INVENTORY_RESERVATION |
| Checkout Service | Coordinates checkout using reservation, payment and order data |
| Payment Service | PAYMENT |
| Order Service | ORDER, ORDER_ITEM |
| Shipment Service | SHIPMENT |
| Notification Service | NOTIFICATION |

CUSTOMER is a shared relational entity used by the relevant services; no separate Customer Service is introduced in the current architecture.

## Data Ownership

Each service owns its domain data.

The Inventory & Reservation Service is the critical consistency boundary.

Inventory state is authoritative in MySQL.

Redis is used only as a cache.

Cross-service workflows use APIs for synchronous operations and the Message Broker for asynchronous events.

## Architecture Flow

```
Client
  ↓
API Gateway
  ↓
Product / Cart / Sale
  ↓
Inventory & Reservation
  ↓
Checkout
  ↓
Payment
  ↓
Order
  ↓
Message Broker
  ├── Shipment
  └── Notification
```

## Access Rules

- A service writes only to the tables it owns.
- Other services obtain that data through its API or through events, not by writing to its tables.
- Only the Inventory & Reservation Service may change `INVENTORY` quantities.
