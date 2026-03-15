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
┌────────────────────────────────────────────────────────────────────────┐
│                            CLIENTS                                     │
│                                                                        │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐           │
│  │ Customer App │     │ Restaurant   │     │  Dasher App  │           │
│  │              │     │ Tablet       │     │              │           │
│  │ • Browse /   │     │              │     │ • GPS stream │           │
│  │   search     │     │ • Accept /   │     │   (every 4s) │           │
│  │ • Place order│     │   reject     │     │ • Accept     │           │
│  │ • Track      │     │ • Update     │     │   dispatch   │           │
│  │   dasher     │     │   prep time  │     │ • Navigate   │           │
│  │ • Rate + tip │     │ • Mark ready │     │ • Confirm    │           │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘           │
│         │ REST + WS          │ REST                │ WS (location)    │
└─────────│────────────────────│─────────────────────│───────────────────┘
          │                    │                     │
┌─────────▼────────────────────▼─────────────────────▼───────────────────┐
│                         API Gateway                                    │
│  Auth (JWT), rate limiting, geo-routing to nearest DC                 │
└─────────┬────────────────────┬─────────────────────┬───────────────────┘
          │                    │                     │
┌─────────▼───────┐   ┌───────▼───────┐    ┌────────▼────────┐
│ Restaurant &    │   │ Order Service │    │ Dasher Service  │
│ Menu Service    │   │ (Saga Orch.)  │    │                 │
│                 │   │               │    │ • Ingest GPS    │
│ • Menu CRUD    │   │ States:       │    │ • Update geo    │
│ • Dynamic      │   │ PLACED →      │    │   index         │
│   availability │   │ CONFIRMED →   │    │ • Publish to    │
│ • Search via   │   │ PREPARING →   │    │   Kafka for     │
│   Elasticsearch│   │ READY →       │    │   tracking      │
│ • Restaurant   │   │ PICKED_UP →   │    │                 │
│   rating cache │   │ DELIVERED     │    │ Geospatial      │
│                │   │               │    │ Index:          │
│                │   │ Each step →   │    │ Redis GEOADD    │
│                │   │ Kafka event   │    │ or QuadTree     │
└────────┬────────┘   │ + compensate │    │ (in-memory)     │
         │           │ on failure   │    └────────┬────────┘
    ┌────▼────┐      └───────┬───────┘             │
    │Elastic  │              │                     │
    │Search   │              │                     │
    │(menus,  │     ┌────────▼─────────────────────▼───────────────┐
    │cuisine, │     │            Kafka (Event Bus)                  │
    │location)│     │  order-events, dasher-location, dispatch-    │
    └─────────┘     │  events, tracking-events, payment-events     │
                    └────────┬──────────┬──────────┬───────────────┘
                             │          │          │
                    ┌────────▼────┐ ┌───▼───────┐  │
                    │ Dispatch /  │ │ Real-Time │  │
                    │ Assignment  │ │ Tracking  │  │
                    │ Service     │ │ Service   │  │
                    │             │ │           │  │
                    │ • Find      │ │ • Consume │  │
                    │   nearby    │ │   dasher  │  │
                    │   dashers   │ │   location│  │
                    │ • Score:    │ │   from    │  │
                    │   ETA +     │ │   Kafka   │  │
                    │   load +    │ │ • Push to │  │
                    │   rating    │ │   customer│  │
                    │ • Batch:    │ │   via     │  │
                    │   same-     │ │   WebSocket│ │
                    │   restaurant│ │ • Store   │  │
                    │   orders →  │ │   trail   │  │
                    │   1 dasher  │ │   in      │  │
                    │ • Proactive:│ │   Cassandra│ │
                    │   dispatch  │ │           │  │
                    │   when      │ └───────────┘  │
                    │   prep_time │                 │
                    │   ≈ travel  │                 │
                    └─────────────┘                 │
                                                    │
                    ┌───────────────────────────────▼──────────┐
                    │         DOWNSTREAM CONSUMERS             │
                    │                                          │
                    │  ┌──────────┐ ┌─────────┐ ┌───────────┐ │
                    │  │ Payment  │ │ ETA     │ │ Notif     │ │
                    │  │ Service  │ │ Service │ │ Service   │ │
                    │  │          │ │         │ │           │ │
                    │  │ Auth on  │ │ ML prep │ │ Push:     │ │
                    │  │ place →  │ │ time +  │ │ "Order    │ │
                    │  │ capture  │ │ route   │ │  confirmed│ │
                    │  │ on       │ │ time +  │ │ ", "Dasher│ │
                    │  │ delivery │ │ traffic │ │  arriving"│ │
                    │  │          │ │ buffer  │ │ , receipt │ │
                    │  │ Split:   │ │         │ │           │ │
                    │  │ platform │ │         │ │ APNs/FCM  │ │
                    │  │ + rest.  │ │         │ │ + SMS     │ │
                    │  │ + dasher │ │         │ │           │ │
                    │  └──────────┘ └─────────┘ └───────────┘ │
                    └──────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                     │
│                                                                        │
│  ┌──────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ PostgreSQL   │  │  Redis   │  │Cassandra │  │  S3      │          │
│  │              │  │          │  │          │  │          │          │
│  │ • Orders     │  │ • Dasher │  │ • Delivery│ │ • Menu   │          │
│  │ • Payments   │  │   geo    │  │   location│ │   images │          │
│  │ • Restaurants│  │   (GEOADD│  │   trail   │ │ • Receipt│          │
│  │ • Users      │  │   )      │  │   (per    │ │   PDFs   │          │
│  │ • Addresses  │  │ • Cart   │  │   order,  │ │          │          │
│  │              │  │ • Session│  │   TTL 30d)│ │          │          │
│  │ ACID for     │  │ • ETA    │  │          │ │          │          │
│  │ order state  │  │   cache  │  │ Write-    │ │          │          │
│  │ transitions  │  │ • Surge  │  │ optimized │ │          │          │
│  │              │  │   zones  │  │ time-     │ │          │          │
│  │              │  │          │  │ series    │ │          │          │
│  └──────────────┘  └──────────┘  └──────────┘  └──────────┘          │
└────────────────────────────────────────────────────────────────────────┘
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

#### Dasher Service (Location + Geospatial Index)
- **GPS ingestion**: Dasher app sends location every 4 seconds via WebSocket → Dasher Service updates the geospatial index
- **Geospatial index**: Redis `GEOADD dashers:available {lng} {lat} {dasher_id}` for finding nearby dashers → `GEORADIUS` queries during dispatch
- **Dasher state**: Redis hash `dasher:{id}` → `{status: available|picking_up|delivering, active_orders: [order_ids], lat, lng, last_updated}`
- **Kafka publishing**: Every location update → Kafka `dasher-location` topic → consumed by Real-Time Tracking Service and Analytics
- **Dasher availability management**: When dasher goes online/offline, update availability index. When dispatched, set status to `picking_up` → remove from available pool

#### Real-Time Tracking Service
- **Purpose**: Stream live dasher location to the customer who is waiting for their order
- **How**: 
  1. Consume dasher location events from Kafka `dasher-location` topic
  2. Filter: only forward locations for dashers with active deliveries
  3. Look up customer WebSocket connection for that order
  4. Push location update to customer app via WebSocket (every 4 seconds)
- **Customer UX**: Map shows dasher's live position, estimated time of arrival, and distance
- **Location trail storage**: Write dasher route to Cassandra `delivery_trail` table (partition key = order_id, clustering key = timestamp) — used for dispute resolution and analytics (TTL 30 days)

#### Notification Service
- Consumes order events from Kafka and triggers notifications to all parties:
  - **Customer**: "Order confirmed", "Restaurant is preparing your food", "Dasher is on the way", "Arriving in 2 minutes", "Delivered! Rate your experience"
  - **Restaurant**: "New order received" (with sound alert on tablet), "Dasher arriving for pickup"
  - **Dasher**: "New delivery available" (with pickup/dropoff details and earnings estimate)
- **Channels**: Push notification (APNs/FCM), in-app alerts, SMS (for delivery confirmation to customers who don't have the app open)
- **Urgency-based**: "New order" to restaurant = high priority (immediate sound alert). "Rate your experience" = low priority (silent push, delayed 30 minutes after delivery)

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

### Dispatch Matching Algorithm — The Core Challenge
```
Given: 100 ready orders + 80 available dashers. Assign optimally.

Naive: nearest dasher to each restaurant. Problem: globally suboptimal.
  Order A assigns Dasher X (closest). Order B's only option was also Dasher X.
  Now Order B waits much longer.

Optimal: Hungarian Algorithm (bipartite matching, O(n³)):
  Build cost matrix: C[i][j] = cost of assigning dasher j to order i
  Cost = α * distance_to_restaurant + β * estimated_delivery_time + γ * dasher_utilization
  Solve for minimum total cost assignment.
  
  At 100 orders × 80 dashers: 100³ = 1M operations → < 10 ms. Fast enough.
  Run every 30 seconds to batch assignments (not per-order — batching is better globally).

Practical: at scale (10K orders × 5K dashers in a city), Hungarian is too slow.
  Use: greedy approximation with local search refinement.
  Or: LP relaxation (linear programming) solved with simplex method.
  
Real-time constraints:
  Food goes cold: max 5 min from "ready" to pickup → hard deadline
  Dasher at capacity: max 2 concurrent deliveries → constraint
  Dasher preferences: some dashers prefer short trips → soft constraint
```

---

## 9. Deep Dive: Engineering Trade-offs

### Race Conditions in Dispatch

```
Scenario: Two dispatch workers process different orders simultaneously.
  Both score Dasher X as the best candidate for their respective orders.
  Both attempt to assign Dasher X → one order gets the dasher, other waits.

Solution 1: Distributed Lock (Redis SETNX) ⭐
  Before sending dispatch request:
    SET dasher:lock:{dasher_id} {order_id} NX EX 30
    NX = only if not exists (atomic)
    EX = expires in 30 seconds (prevents deadlock on crash)
  
  If SET succeeds → we "own" this dasher → send dispatch request
  If SET fails → dasher already assigned → try next candidate
  On dasher accept/decline → release lock
  
  ✓ Simple, fast (< 1ms Redis call)
  ✗ False exclusion: dasher locked for 30s even if first order is declined
  Mitigation: 15-second accept window + immediate lock release on decline

Solution 2: Optimistic Concurrency in DB
  UPDATE dashers 
  SET current_order_count = current_order_count + 1, 
      version = version + 1
  WHERE dasher_id = ? AND current_order_count < 2 AND version = ?
  
  If rows_affected = 1 → success (dasher assigned)
  If rows_affected = 0 → concurrent assignment happened → retry with next dasher
  
  ✓ No external lock service
  ✗ Higher latency than Redis (DB round-trip)

Solution 3: Centralized Dispatch Queue per Zone ⭐⭐
  Partition city into zones (e.g., H3 resolution-7 hexagons)
  Each zone has ONE dispatch worker (single-writer)
  Worker consumes orders from Kafka partition for that zone
  Single-threaded per zone → NO race conditions by design
  
  ✓ Zero lock contention
  ✓ Can do batch optimization (collect orders for 5 seconds, then assign)
  ✗ Zone boundary: order near boundary might miss dashers in adjacent zone
  Mitigation: zones overlap by 1 km; dispatch worker checks neighboring zones

Recommended: Solution 3 for production (DoorDash's actual approach),
             Solution 1 for simpler deployments
```

### Order State Machine Failure Scenarios

```
Scenario 1: Restaurant tablet goes offline after CONFIRMED
  Detection: No heartbeat from tablet for 2 minutes
  Action:
    T+2min: Send SMS to restaurant phone number
    T+5min: Auto-call restaurant landline (IVR: "Press 1 to confirm order")
    T+8min: If still no response → auto-cancel order
    → Compensate: refund customer, release dasher assignment
    → Notify customer: "Restaurant unreachable, order cancelled. Here are similar restaurants."
  
  Edge case: Restaurant was preparing food but tablet died
    → When tablet reconnects, check if order was auto-cancelled
    → If food was prepared → restaurant eats the cost (SLA violation)
    → Platform may compensate restaurant for confirmed-then-cancelled orders

Scenario 2: Dasher app crashes during PICKED_UP
  Detection: No GPS heartbeat for 3 minutes
  Action:
    T+3min: Send push notification + SMS to dasher
    T+5min: Call dasher's phone
    T+8min: If unreachable → reassign to new dasher
    → Challenge: new dasher doesn't have the food!
    → If dasher had the food and went offline → customer gets refund
    → If dasher reconnects → resume delivery (update ETA)
  
  Prevention: "Proof of pickup" — dasher confirms pickup code from restaurant
    → System knows food was picked up → different handling than pre-pickup crash

Scenario 3: Payment capture fails after delivery
  Flow: Payment authorized at order placement, captured on delivery confirmation
  If capture fails (expired card, bank decline):
    → Retry capture 3 times with exponential backoff
    → If all retries fail → queue for manual payment recovery
    → Restaurant and dasher STILL get paid (platform absorbs the loss)
    → Never re-charge customer for same order (idempotency_key on capture)
    → Flag customer account for future orders (require upfront payment)

Scenario 4: Customer cancels after food is being prepared (PREPARING state)
  Cancellation policy based on state:
    PLACED (not confirmed):     Full refund, no penalty
    CONFIRMED (not preparing):  Full refund, restaurant gets small fee ($2)
    PREPARING:                  Partial refund (50%), restaurant keeps food cost
    READY / PICKED_UP:          No refund (food is ready, dasher is en route)
  
  Who pays for the food?
    Platform absorbs cost for CONFIRMED cancellations (acquisition cost)
    Customer pays for PREPARING+ cancellations (food already in progress)
    Restaurant always receives payment for food they've started preparing
```

### Real-Time Tracking Data Flow

```
Dasher App → Customer App: live location on map

  Dasher App                                              Customer App
      │                                                        │
      │ GPS every 4 seconds (adaptive — see below)             │
      │                                                        │
      ├──→ API Gateway                                         │
      │       │                                                │
      │    ┌──▼────────────┐                                   │
      │    │ Kafka topic:  │ partitioned by dasher_id          │
      │    │ dasher-        │ (ensures ordering per dasher)     │
      │    │ locations     │                                    │
      │    └──┬────────────┘                                   │
      │       │                                                │
      │    ┌──▼────────────┐                                   │
      │    │ Flink consumer│                                   │
      │    │ (or Go worker)│                                   │
      │    │               │                                   │
      │    │ 1. Update Redis GEOADD (for dispatch matching)   │
      │    │ 2. Map-snap GPS to nearest road segment          │
      │    │ 3. Kalman filter: smooth GPS jitter              │
      │    │ 4. Write to Cassandra (trip location trail)       │
      │    │ 5. Publish to Redis Pub/Sub channel:              │
      │    │    "tracking:{order_id}"                          │
      │    └──┬────────────┘                                   │
      │       │                                                │
      │    ┌──▼────────────┐                                   │
      │    │ Tracking      │ ← Customer's WebSocket connects   │
      │    │ Service       │    here when viewing order status  │
      │    │               │                                   │
      │    │ Subscribes to │                                   │
      │    │ Redis Pub/Sub │──── WebSocket push ──────────────→│
      │    │ "tracking:    │    { lat, lng, heading, eta }     │
      │    │  {order_id}"  │                                   │
      │    └───────────────┘                                   │

GPS optimization (adaptive frequency):
  Dasher stationary (speed=0):        Update every 30 seconds (save battery)
  Dasher walking to restaurant (<5km/h): Update every 10 seconds
  Dasher driving (>10 km/h):          Update every 4 seconds
  Dasher approaching customer (<500m): Update every 2 seconds (precise ETA)

GPS jitter handling (Kalman filter):
  Raw GPS has ±10-30m error. Without filtering:
    Map shows dasher "jumping" between buildings, crossing rivers
  
  Kalman filter: predict next position from velocity + heading,
    then correct with actual GPS reading. Weight prediction more
    when GPS signal is weak (indoors, urban canyon).
  
  Map snapping: after Kalman, snap to nearest road segment
    using a road network graph (OpenStreetMap / Google Roads API).
    Dasher always appears ON a road, moving smoothly.

ETA update during delivery:
  Not just "distance ÷ speed" — consider:
    Remaining route distance (via routing engine, not straight line)
    Real-time traffic on route segments
    Typical time to park + walk to door at this address
    Buffer: 2-3 min for elevator, apartment navigation
  Recalculated every 30 seconds as dasher moves
```

