# 68. Design an Order Management System

---

## 1. Functional Requirements (FR)

- **Place order**: Create order from cart with shipping address, payment method, and delivery preferences
- **Order lifecycle**: Track states: placed, payment_confirmed, processing, shipped, out_for_delivery, delivered, returned, cancelled
- **Multi-item orders**: Orders with items from multiple sellers/warehouses (split shipments)
- **Order tracking**: Real-time shipment tracking with carrier integration (FedEx, UPS, USPS)
- **Returns & refunds**: Initiate return, generate return label, process refund on receipt
- **Order history**: View past orders with search, filter, and reorder capability
- **Notifications**: Email/SMS/push at each state transition
- **Invoice generation**: Generate PDF invoices for each order
- **Cancellation**: Cancel order before shipment; partial cancellation for multi-item orders
- **Order modification**: Change shipping address or delivery date before processing cutoff

---

## 2. Non-Functional Requirements (NFRs)

- **Strong Consistency**: Order state transitions must be ACID — no lost orders, no double charges
- **Availability**: 99.99% — order placement is revenue-critical
- **Low Latency**: Order placement in < 2 seconds (including payment)
- **Scalability**: 10M+ orders/day; 50M+ order status checks/day
- **Idempotent**: Duplicate order submission must not create two orders
- **Auditability**: Every state change immutably logged
- **Durability**: Order data retained for 7+ years (regulatory compliance)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Orders / day | 10M |
| Orders / sec | ~115 (peak 10K during sales) |
| Order status checks / day | 50M |
| Avg items per order | 3 |
| Order record size | ~5 KB (with items, addresses, payment) |
| Storage / day | 10M x 5 KB = 50 GB |
| Storage / year | ~18 TB |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    ORDER PLACEMENT (SAGA ORCHESTRATION)                 │
│                                                                        │
│  Client --> API Gateway --> Order Service (Saga Orchestrator)           │
│                                  |                                     │
│         Step 1                Step 2              Step 3               │
│    +-----v------+       +------v-------+    +------v-------+          │
│    | Create     |       | Inventory    |    | Payment      |          │
│    | Order      |       | Service      |    | Service      |          │
│    | (status =  |       | (reserve     |    | (authorize   |          │
│    |  PLACED)   |       |  stock at    |    |  + capture   |          │
│    |            |       |  selected    |    |  payment)    |          │
│    | PostgreSQL |       |  warehouse)  |    |              |          │
│    +-----+------+       +------+-------+    +------+-------+          │
│          |                     |                    |                  │
│          |              On failure:           On failure:              │
│          |              → cancel order        → release inventory     │
│          |              (compensate)          → cancel order           │
│          |                                    (compensate)             │
│          |                                                            │
│    Step 4 (on success)                                                │
│    +-----v------+                                                     │
│    | Order      |                                                     │
│    | Confirmed  |                                                     │
│    | (status =  |                                                     │
│    | CONFIRMED) |                                                     │
│    +-----+------+                                                     │
│          |                                                            │
│    +-----v------+                                                     │
│    |   Kafka    |  (order-events topic)                               │
│    +-----+------+                                                     │
│          |                                                            │
│    +-----+----------+----------+-----------+-----------+              │
│    |     |          |          |           |           |              │
│ +--v--+ +v------+ +v-------+ +v--------+ +v--------+ +v---------+   │
│ |Notif| |Invoice| |Shipping| |Analytics| |Search   | |Recommend|   │
│ |Svc  | |Svc    | |Svc     | |(Click-  | |Index    | |ation    |   │
│ |(email| |(PDF   | |(create | | House)  | |(ES:     | |Engine   |   │
│ | SMS, | | gen + | | label, | |         | | order   | |(bought  |   │
│ | push)| | S3)   | | assign | |         | | search) | | together|   │
│ +------+ +-------+ | carrier| +---------+ +---------+ +---------+   │
│                     +---+----+                                        │
│                         |                                             │
│                   +-----v------+                                      │
│                   | Carrier    |                                      │
│                   | Integration|                                      │
│                   | (FedEx,UPS,|                                      │
│                   |  USPS API  |                                      │
│                   |  webhooks  |                                      │
│                   |  for       |                                      │
│                   |  tracking) |                                      │
│                   +-----+------+                                      │
│                         |                                             │
│                   +-----v------+                                      │
│                   | Tracking   |                                      │
│                   | Service    |                                      │
│                   | (poll      |                                      │
│                   |  carrier   |                                      │
│                   |  API every |                                      │
│                   |  30 min +  |                                      │
│                   |  webhook)  |                                      │
│                   +------------+                                      │
│                                                                       │
│  DATA STORES:                                                         │
│  +-------------+  +--------+  +-------+  +-----------+  +----------+ │
│  | PostgreSQL  |  | Redis  |  | Kafka |  | Elastic-  |  | S3       | │
│  | (orders,    |  | (order |  | (order|  | search    |  | (invoices| │
│  |  items,     |  |  status|  |  events| | (order    |  |  return  | │
│  |  shipments, |  |  cache,|  |  ship- | |  search   |  |  labels) | │
│  |  state_log, |  |  idemp-|  |  ment  | |  for CS   |  |          | │
│  |  returns)   |  |  otency|  |  events| |  dashboard|  |          | │
│  +-------------+  +--------+  +-------+  +-----------+  +----------+ │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Order State Machine — The Core

```
                    +----------+
                    |  PLACED  | <-- User submits order
                    +----+-----+
                         |
                   +-----v------+
              +----|  PAYMENT   |----+
              |    |  PENDING   |    |
              |    +-----+------+    |
              |          |           |
        +-----v----+ +--v-------+ +-v----------+
        | PAYMENT  | | PAYMENT  | | PAYMENT    |
        | FAILED   | | CONFIRMED| | CANCELLED  |
        +----------+ +----+-----+ +------------+
                          |
                    +-----v------+
                    | PROCESSING | <-- Warehouse picks & packs
                    +-----+------+
                          |
                    +-----v------+
                    |  SHIPPED   | <-- Carrier has package
                    +-----+------+
                          |
                    +-----v------+
                    |OUT_FOR_    |
                    |DELIVERY    |
                    +-----+------+
                          |
                    +-----v------+     +------------+
                    | DELIVERED  |---->| RETURN     |
                    +------------+     | INITIATED  |
                                       +-----+------+
                                             |
                                       +-----v------+
                                       | RETURN     |
                                       | RECEIVED   |
                                       +-----+------+
                                             |
                                       +-----v------+
                                       |  REFUNDED  |
                                       +------------+

State transition rules (enforced in code + DB):
  PLACED --> PAYMENT_PENDING (always, automatic)
  PAYMENT_PENDING --> PAYMENT_CONFIRMED | PAYMENT_FAILED | PAYMENT_CANCELLED
  PAYMENT_CONFIRMED --> PROCESSING (automatic, after inventory reserved)
  PROCESSING --> SHIPPED (warehouse confirms dispatch)
  SHIPPED --> OUT_FOR_DELIVERY (carrier scan)
  OUT_FOR_DELIVERY --> DELIVERED (carrier confirmation or customer confirmation)
  DELIVERED --> RETURN_INITIATED (within return window, e.g., 30 days)
  
Invalid transitions REJECTED:
  SHIPPED --> PLACED (can't go backward)
  DELIVERED --> PROCESSING (can't reprocess)
  
Implementation:
  def transition(order_id, new_state):
      order = db.get(order_id)
      if new_state not in VALID_TRANSITIONS[order.state]:
          raise InvalidTransitionError(f"{order.state} -> {new_state}")
      
      BEGIN TRANSACTION
        UPDATE orders SET state = new_state, updated_at = NOW() WHERE order_id = ?
        INSERT INTO order_state_log (order_id, from_state, to_state, timestamp, actor)
        VALUES (?, order.state, new_state, NOW(), current_actor)
      COMMIT
      
      publish_kafka('order-events', { order_id, new_state, timestamp })
```

#### Order Placement — The Orchestration Saga

```
Placing an order involves multiple services. Use SAGA pattern (not 2PC):

Step 1: Create order (Order Service)
  INSERT order with status = 'PLACED'
  
Step 2: Reserve inventory (Inventory Service)
  For each item: reserve stock at selected warehouse
  If ANY item out of stock --> compensate: cancel order, release other reservations
  
Step 3: Charge payment (Payment Service)
  Authorize + capture payment
  If payment fails --> compensate: release all inventory reservations, mark order PAYMENT_FAILED
  
Step 4: Confirm order (Order Service)
  Update status = 'PAYMENT_CONFIRMED'
  Publish order-confirmed event
  
Step 5 (async): Generate invoice, send confirmation email, update analytics

Saga compensation (rollback):
  If Step 3 fails (payment declined):
    - Undo Step 2: release inventory reservations
    - Undo Step 1: mark order as PAYMENT_FAILED
    - Notify user: "Payment failed. Your items are still in your cart."

  If Step 2 fails (out of stock):
    - Undo Step 1: mark order as CANCELLED
    - Notify user: "Sorry, [item] is no longer available."

Why SAGA (not distributed transaction / 2PC)?
  2PC: requires all services to hold locks simultaneously --> blocks everything if one is slow
  SAGA: each step commits independently; compensating transactions undo on failure
  At e-commerce scale, SAGA is the industry standard (Amazon, Shopify, etc.)
```

#### Split Shipments

```
Order has 3 items:
  Item A: only at WH-NYC
  Item B: only at WH-LAX
  Item C: at both (allocated to WH-NYC for proximity)

Result: 2 shipments
  Shipment 1 (WH-NYC): Item A + Item C --> FedEx tracking #12345
  Shipment 2 (WH-LAX): Item B --> UPS tracking #67890

Each shipment has its own:
  - Tracking number
  - Carrier
  - State machine (independent of other shipments)
  - Delivery date

Order state = aggregate of shipment states:
  If any shipment is SHIPPED --> order is PARTIALLY_SHIPPED
  When ALL shipments DELIVERED --> order is DELIVERED
  
Partial cancellation:
  User cancels Item B before shipment --> cancel Shipment 2 only
  Refund for Item B; Shipment 1 proceeds normally
```

---

## 5. APIs

### Place Order
```http
POST /api/v1/orders
Idempotency-Key: "order-abc-123"
{
  "items": [
    {"sku_id": "SKU-123", "quantity": 2, "price": 29.99},
    {"sku_id": "SKU-456", "quantity": 1, "price": 49.99}
  ],
  "shipping_address": { "street": "...", "city": "...", "zip": "...", "country": "US" },
  "payment_method_id": "pm-uuid",
  "delivery_preference": "standard"
}
Response: 201 Created
{
  "order_id": "order-uuid",
  "status": "payment_pending",
  "estimated_delivery": "2026-03-18",
  "total": 109.97,
  "shipments": [
    {"shipment_id": "ship-1", "items": ["SKU-123","SKU-456"], "warehouse": "WH-NYC"}
  ]
}
```

### Get Order Status
```http
GET /api/v1/orders/{order_id}
Response: 200 OK
{
  "order_id": "order-uuid",
  "status": "shipped",
  "placed_at": "2026-03-14T10:00:00Z",
  "items": [...],
  "shipments": [
    {"shipment_id": "ship-1", "status": "in_transit", "carrier": "FedEx",
     "tracking_number": "794644790132", "estimated_delivery": "2026-03-18",
     "tracking_url": "https://fedex.com/track?id=794644790132"}
  ],
  "payment": { "method": "Visa ending 4242", "amount": 109.97, "status": "captured" }
}
```

### Cancel Order
```http
POST /api/v1/orders/{order_id}/cancel
{ "reason": "changed_mind" }
Response: 200 OK
{ "status": "cancelled", "refund_amount": 109.97, "refund_status": "processing" }
```

### Initiate Return
```http
POST /api/v1/orders/{order_id}/return
{
  "items": [{"sku_id": "SKU-123", "quantity": 1, "reason": "defective"}],
  "return_method": "mail"
}
Response: 200 OK
{
  "return_id": "ret-uuid",
  "return_label_url": "https://s3.../return-label.pdf",
  "refund_amount": 29.99,
  "refund_status": "pending_return_receipt"
}
```

---

## 6. Data Model

### PostgreSQL — Source of Truth

```sql
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'placed',
    subtotal        DECIMAL(10,2),
    tax             DECIMAL(10,2),
    shipping_cost   DECIMAL(10,2),
    total           DECIMAL(10,2),
    currency        CHAR(3) DEFAULT 'USD',
    shipping_address JSONB,
    payment_method_id VARCHAR(64),
    payment_status  VARCHAR(20),
    idempotency_key VARCHAR(64) UNIQUE,
    placed_at       TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_user (user_id, placed_at DESC),
    INDEX idx_status (status)
);

CREATE TABLE order_items (
    item_id         BIGSERIAL PRIMARY KEY,
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    sku_id          VARCHAR(50) NOT NULL,
    quantity        INT NOT NULL,
    unit_price      DECIMAL(10,2),
    total_price     DECIMAL(10,2),
    shipment_id     UUID,
    status          VARCHAR(20) DEFAULT 'active',
    INDEX idx_order (order_id)
);

CREATE TABLE shipments (
    shipment_id     UUID PRIMARY KEY,
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    warehouse_id    VARCHAR(50),
    carrier         VARCHAR(20),
    tracking_number VARCHAR(64),
    status          VARCHAR(30) DEFAULT 'pending',
    shipped_at      TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    INDEX idx_order (order_id),
    INDEX idx_tracking (tracking_number)
);

CREATE TABLE order_state_log (
    log_id          BIGSERIAL PRIMARY KEY,
    order_id        UUID NOT NULL,
    from_state      VARCHAR(30),
    to_state        VARCHAR(30) NOT NULL,
    actor           VARCHAR(50),
    reason          TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_order (order_id, created_at)
);

CREATE TABLE returns (
    return_id       UUID PRIMARY KEY,
    order_id        UUID NOT NULL,
    user_id         UUID NOT NULL,
    status          VARCHAR(20) DEFAULT 'initiated',
    refund_amount   DECIMAL(10,2),
    reason          TEXT,
    return_label_url TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_order (order_id)
);
```

### Redis — Caching + Idempotency

```
# Order status cache (avoid DB hit for frequent polling)
order_status:{order_id}   --> Hash { status, tracking, updated_at }
TTL: 300

# Idempotency (prevent duplicate order placement)
idempotency:{key}         --> order_id (if already processed)
TTL: 86400

# User's recent orders (for quick display)
user_orders:{user_id}     --> List of order_ids (last 20)
TTL: 3600
```

### Kafka Topics

```
Topic: order-events       (placed, confirmed, shipped, delivered -- all state transitions)
Topic: payment-events     (payment success/failure -- consumed by order service)
Topic: shipment-events    (tracking updates from carrier webhooks)
Topic: return-events      (return initiated, received, refunded)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Duplicate order** | Idempotency key (unique per checkout attempt); DB UNIQUE constraint |
| **Payment succeeds but order update fails** | Saga with compensation: payment service publishes event; order service consumes and updates; if missed, reconciliation job retries |
| **Inventory reserved but payment fails** | Compensation step releases inventory; TTL-based reservation auto-expires |
| **State machine corruption** | Valid transitions enforced in code + DB CHECK constraint; state_log is immutable audit trail |
| **Carrier webhook missed** | Poll carrier API every 30 min for orders in SHIPPED state; reconcile |
| **DB failover** | PostgreSQL synchronous replication; promote standby with zero data loss |

### Specific: Idempotent Order Placement

```
Problem: User clicks "Place Order" and network times out. User clicks again.
  Without idempotency: two identical orders created, charged twice.

Solution:
  Client generates idempotency_key (UUID) on checkout page load.
  Both clicks send same idempotency_key.
  
  Server:
    1. Check Redis: GET idempotency:{key}
       If exists --> return cached order (already processed)
    2. Check PostgreSQL: SELECT order_id FROM orders WHERE idempotency_key = ?
       If exists --> return existing order
    3. If neither exists --> proceed with order creation
    4. After creation: SET idempotency:{key} {order_id} in Redis (TTL 24h)
    
  Result: exactly one order, regardless of retries.
```

---

## 8. Deep Dive: Engineering Trade-offs

### Saga vs 2PC for Distributed Order Placement

```
2PC (Two-Phase Commit):
  Coordinator asks all services: "Can you commit?"
  All say yes --> "Commit." All commit atomically.
  
  Problems at e-commerce scale:
  - Coordinator is SPOF
  - All services hold locks during prepare phase --> latency, blocking
  - If any service is slow --> ALL are blocked
  - Not supported across heterogeneous systems (different DBs, services)

Saga (Choreography or Orchestration):
  Each step commits locally. Failures trigger compensating transactions.
  
  Choreography: each service publishes event; next service reacts
    Order created --> Inventory listens, reserves stock --> Payment listens, charges
    Loose coupling but hard to debug (no central view of saga progress)
  
  Orchestration (recommended): central orchestrator coordinates steps
    Order Service calls Inventory, then Payment, then Shipping
    On failure: orchestrator calls compensating actions in reverse order
    Easy to monitor, debug, and modify

  Trade-off:
    2PC: strong consistency, but fragile and slow at scale
    Saga: eventual consistency during saga execution, but resilient and fast
    For e-commerce: Saga is the clear winner. Brief inconsistency during
    the 2-second order placement window is acceptable.
```

### Order Search: Why Elasticsearch Alongside PostgreSQL

```
Customer service needs: "Find all orders for user X with product Y shipped in March"

PostgreSQL can do this, but:
  - Full-text search across product names, addresses is slow
  - Complex compound filters with pagination are expensive
  - At 10M orders/day, queries slow down without heavy indexing

Elasticsearch:
  - Index order data (denormalized): order_id, user, items, status, dates
  - Support full-text search + filters + aggregations
  - Sub-200ms response for complex queries
  
  CDC pipeline: PostgreSQL --> Debezium --> Kafka --> ES consumer --> Elasticsearch
  Lag: < 5 seconds from order placed to searchable in ES
  
  Use PostgreSQL for: order placement, state transitions (ACID)
  Use Elasticsearch for: search, customer service dashboard, analytics queries
```

