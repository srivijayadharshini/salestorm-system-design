# SALESTORM API and Event Design

This folder contains the REST API specification and the event design for the SALESTORM flash-sale system.

| File | Description |
|---|---|
| `API_Specification.md` | Endpoints, authentication, request/response, status codes, error handling, idempotency |
| `Event_Design.md` | Commands/events, producers and consumers, broker flow, retries, DLQ, recovery, sync vs async |

Based on the service architecture:

```
Client → API Gateway → Product / Cart / Sale → Inventory & Reservation → Checkout → Payment → Order → Message Broker → Shipment / Notification
```

MySQL is the source of truth for inventory (see `../database/`). Redis is a cache only.
