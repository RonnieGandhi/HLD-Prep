# 21. Design a Food Delivery Platform (DoorDash / Zomato)

---

## 1. Functional Requirements (FR)

- **Customer**: Browse restaurants, view menus, place orders, track delivery in real-time
- **Restaurant**: Manage menu items, accept/reject orders, update order status (preparing → ready)
- **Delivery Partner (Dasher)**: Go online/offline, accept delivery requests, navigate to pickup/dropoff
- **Order lifecycle**: Browse → Cart → Checkout → Restaurant Accept → Preparing → Ready → Picked Up → Delivered
- **Search**: Search restaurants by name, cuisine, location
- **Ratings & Reviews**: Rate restaurant and dasher after delivery
- **Payment**: Process payment, handle tips, restaurant payouts, dasher earnings
- **Promotions**: Coupons, discounts, free delivery offers
- **ETA**: Estimated delivery time considering prep time + pickup + travel

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99% — especially during meal rushes (lunch/dinner peaks)
- **Low Latency**: Search results < 200 ms, order placement < 1 s
- **Real-time Tracking**: Dasher location updates every 4 seconds
- **Scalability**: 30M+ orders/day, 1M+ restaurants, 5M+ dashers
- **Consistency**: Order and payment must be strongly consistent (no double charges, no lost orders)
- **Fault Tolerant**: Active orders must survive any single component failure

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Orders / day | 30M |
| Orders / sec | ~350 (peak 1,500 during dinner) |
| Restaurants | 1M |
| Active dashers at any time | 500K |
| Dasher location updates / sec | 125K |
| Menu items total | 50M |
| Search QPS | 50K |

---

## 4. High-Level Design (HLD)

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Customer │  │Restaurant│  │  Dasher  │
│   App    │  │  Tablet  │  │   App    │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
┌────▼──────────────▼──────────────▼────┐
│             API Gateway               │
└────┬──────────────┬──────────────┬────┘
     │              │              │
┌────▼────┐   ┌─────▼─────┐  ┌────▼──────┐
│Restaurant│  │  Order    │  │  Dasher   │
│ & Menu  │  │  Service  │  │  Service  │
│ Service │  │           │  │(Location +│
└────┬────┘  └─────┬─────┘  │ Matching) │
     │              │        └────┬──────┘
┌────▼────┐   ┌─────▼─────┐      │
│Elastic  │   │   Kafka   │      │
│Search   │   │(order-    │  ┌───▼──────┐
│(Search) │   │ events)   │  │ GeoIndex │
└─────────┘   └─────┬─────┘  │(QuadTree/│
                     │        │ Redis)   │
              ┌──────▼──────┐ └──────────┘
              │ Dispatch /  │
              │ Assignment  │
              │ Service     │
              └──────┬──────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
   ┌─────▼────┐ ┌───▼────┐ ┌───▼──────┐
   │ Payment  │ │  ETA   │ │Notif     │
   │ Service  │ │ Service│ │Service   │
   └──────────┘ └────────┘ └──────────┘

Data Stores:
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│PostgreSQL│ │  Redis   │ │Cassandra │ │  S3      │
│(Orders,  │ │(Geo, Cart│ │(Tracking,│ │(Menu     │
│ Payments)│ │ Sessions)│ │ History) │ │ Images)  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### Component Deep Dive

#### Restaurant & Menu Service
- **Menu hierarchy**: Restaurant → Menu Categories → Menu Items → Modifiers/Add-ons
- **Dynamic menus**: Items can be available/unavailable based on time of day, stock
- **Search**: Elasticsearch indexes restaurant name, cuisine type, menu items, tags
- **Recommendation**: "Popular near you", "Based on your past orders"

#### Order Service — The Core State Machine

```
                ┌──────────┐
                │  PLACED  │ ← Customer clicks "Place Order"
                └────┬─────┘
                     │ notify restaurant
                ┌────▼─────┐
           ┌────│ CONFIRMED│ ← Restaurant accepts
           │    └────┬─────┘
           │         │
           │    ┌────▼─────┐
           │    │PREPARING │ ← Restaurant starts cooking
           │    └────┬─────┘
           │         │
           │    ┌────▼─────┐
           │    │  READY   │ ← Food ready for pickup
           │    └────┬─────┘
           │         │ dasher dispatched (may be earlier)
           │    ┌────▼─────┐
           │    │PICKED UP │ ← Dasher collected food
           │    └────┬─────┘
           │         │
           │    ┌────▼──────┐
           │    │ DELIVERED │ ← Dasher arrived at customer
           │    └───────────┘
           │
           │    ┌───────────┐
           └───▶│ CANCELLED │ ← Restaurant rejects or customer cancels
                └───────────┘
```

- **Idempotent transitions**: Each state change has a unique event_id → replay-safe
- **Saga pattern**: Order placement involves multiple services (inventory, payment, restaurant, dasher). Use saga with compensating actions on failure

#### Dispatch / Assignment Service — Complex Optimization

Unlike Uber (1 rider → 1 driver), food delivery has a **batching opportunity**: one dasher can pick up orders from the same restaurant or nearby restaurants.

**Matching Algorithm**:
1. When an order is ready (or near-ready), find candidate dashers:
   - Available dashers within radius of restaurant
   - Dashers with current deliveries whose route passes near the restaurant
2. Score each candidate:
   - Distance/ETA to restaurant
   - Current delivery load (0, 1, or 2 active orders)
   - Dasher's acceptance rate
   - Customer priority (premium subscribers)
3. **Batching optimization**: If two orders are from the same restaurant → assign to same dasher
4. Assign top-scored dasher, send dispatch request (30-second timeout to accept)

**Proactive dispatching**: Start finding a dasher BEFORE the food is ready → dasher arrives at restaurant just as food is ready (reduces total delivery time)
- **ETA model**: `prep_time_estimate + travel_to_restaurant` → dispatch when `prep_time_remaining ≈ travel_to_restaurant`

#### ETA Service
- **Components of delivery ETA**:
  ```
  total_eta = prep_time + dasher_to_restaurant_time + restaurant_to_customer_time + buffer
  ```
- **Prep time estimation**: ML model trained on historical data per restaurant
  - Features: restaurant, day of week, time of day, order complexity, current pending orders
- **Travel time**: Routing engine (OSRM/Google Maps) with real-time traffic
- **Buffer**: 5-minute padding for pickup logistics

#### Payment Service
- **Payment flow**: Authorization on order placement → Capture on delivery confirmation
- **Split payment**: Customer pays → platform takes commission → restaurant gets food amount → dasher gets delivery fee + tip
- **Refunds**: For missing items, late delivery, quality issues → automated rules + manual review

---

## 5. APIs

### Search Restaurants
```http
GET /api/v1/restaurants?lat=37.77&lng=-122.42&cuisine=italian&sort=rating&radius=5km
```

### Place Order
```http
POST /api/v1/orders
{
  "restaurant_id": "rest-uuid",
  "items": [
    {"item_id": "item-1", "quantity": 2, "modifiers": ["extra cheese"]},
    {"item_id": "item-2", "quantity": 1}
  ],
  "delivery_address": {"lat": 37.77, "lng": -122.42, "address": "..."},
  "payment_method_id": "pm-uuid",
  "tip_amount": 5.00,
  "promo_code": "SAVE20"
}
Response: 201 Created
{
  "order_id": "order-uuid",
  "status": "placed",
  "estimated_delivery_at": "2026-03-13T11:15:00Z",
  "total": {"subtotal": 28.00, "delivery_fee": 3.99, "tax": 2.52, "tip": 5.00, "discount": -5.60, "total": 33.91}
}
```

### Restaurant Accept/Reject
```http
POST /api/v1/orders/{order_id}/accept
{ "estimated_prep_time_minutes": 20 }

POST /api/v1/orders/{order_id}/reject
{ "reason": "too_busy" }
```

### Track Order (Real-time)
```http
GET /api/v1/orders/{order_id}/track
WebSocket: /ws/v1/orders/{order_id}/track
```

---

## 6. Data Model

### PostgreSQL — Orders (ACID required)

```sql
CREATE TABLE orders (
    order_id            UUID PRIMARY KEY,
    customer_id         UUID NOT NULL,
    restaurant_id       UUID NOT NULL,
    dasher_id           UUID,
    status              VARCHAR(20),
    delivery_address    JSONB,
    subtotal            DECIMAL(10,2),
    delivery_fee        DECIMAL(10,2),
    tax                 DECIMAL(10,2),
    tip                 DECIMAL(10,2),
    discount            DECIMAL(10,2),
    total               DECIMAL(10,2),
    promo_code          VARCHAR(32),
    estimated_prep_min  INT,
    estimated_delivery_at TIMESTAMP,
    actual_delivered_at TIMESTAMP,
    placed_at           TIMESTAMP,
    payment_status      VARCHAR(20),
    INDEX idx_customer (customer_id, placed_at DESC),
    INDEX idx_restaurant (restaurant_id, placed_at DESC),
    INDEX idx_dasher (dasher_id, placed_at DESC)
);

CREATE TABLE order_items (
    order_item_id   UUID PRIMARY KEY,
    order_id        UUID REFERENCES orders,
    menu_item_id    UUID,
    item_name       VARCHAR(256),
    quantity        INT,
    unit_price      DECIMAL(10,2),
    modifiers       JSONB,
    special_notes   TEXT
);
```

### MySQL — Restaurant & Menu

```sql
CREATE TABLE restaurants (
    restaurant_id   UUID PRIMARY KEY,
    name            VARCHAR(256),
    cuisine_type    VARCHAR(64),
    address         TEXT,
    lat             DECIMAL(10,7),
    lng             DECIMAL(10,7),
    rating          DECIMAL(2,1),
    price_range     TINYINT,
    avg_prep_time   INT,
    is_active       BOOLEAN,
    hours           JSON
);

CREATE TABLE menu_items (
    item_id         UUID PRIMARY KEY,
    restaurant_id   UUID,
    category        VARCHAR(64),
    name            VARCHAR(256),
    description     TEXT,
    price           DECIMAL(10,2),
    image_url       TEXT,
    is_available    BOOLEAN,
    modifiers       JSON,
    INDEX idx_restaurant (restaurant_id)
);
```

### Redis — Cart, Dasher Geo, Dispatch

```
# Shopping cart (session-based)
Key:    cart:{customer_id}
Value:  Hash { restaurant_id, items: JSON, updated_at }
TTL:    3600

# Dasher geospatial
GEOADD dashers:available:{city} {lng} {lat} {dasher_id}

# Active orders per dasher
Key:    dasher:orders:{dasher_id}
Value:  Set of order_ids (max 2)
```

### Kafka Topics

```
Topic: order-events        (placed, confirmed, preparing, ready, picked_up, delivered, cancelled)
Topic: dasher-locations     (high throughput)
Topic: dispatch-requests    (matching assignments)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Order placed but payment fails** | Saga: compensate by cancelling order, releasing restaurant capacity |
| **Restaurant tablet offline** | SMS/phone call fallback; auto-cancel after 5 min if not confirmed |
| **Dasher goes offline mid-delivery** | Detect via heartbeat timeout → reassign to new dasher |
| **Duplicate order submission** | Idempotency key on order placement API |
| **Payment double charge** | Payment authorization with idempotent capture |
| **Peak load (dinner rush)** | Auto-scale order service; queue orders in Kafka; circuit breaker on payment |

### Saga Pattern for Order Placement
```
1. Create Order (Order Service)          → Compensate: Delete Order
2. Authorize Payment (Payment Service)   → Compensate: Release Authorization
3. Notify Restaurant (Restaurant Svc)    → Compensate: Cancel Order at Restaurant
4. Dispatch Dasher (Dispatch Service)    → Compensate: Cancel Dispatch

If step 3 fails → execute compensations for steps 2, 1 (reverse order)
```

---

## 8. Additional Considerations

### Dynamic Delivery Fees
```python
delivery_fee = base_fee + distance_fee + surge_fee
  base_fee = $2.99
  distance_fee = $0.50 per mile beyond 3 miles
  surge_fee = demand_multiplier * $1.00  (peak hours)
```

### Restaurant Capacity Management
- Restaurants can set max concurrent orders (e.g., 15)
- When at capacity → mark as "busy" in search results, increase estimated prep time
- Prevent overloading the kitchen during peak hours

### Batched Delivery
- One dasher picks up from Restaurant A + Restaurant B (nearby) → delivers both
- Routing optimization: minimize total route distance
- Customer sees a slight delay but cheaper delivery (shared fee)

### Fraud Detection
- Fake deliveries (dasher marks delivered without delivery)
- Multiple refund abuse by customers
- Detect via: GPS verification at delivery address, photo proof of delivery, ML anomaly detection

