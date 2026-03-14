# 67. Design a Flash Sale System

---

## 1. Functional Requirements (FR)

- **Scheduled sales**: Admin schedules flash sale with start/end time, SKUs, quantities, discounted prices
- **Countdown timer**: Show countdown to sale start; reveal items at exact start time
- **Atomic purchase**: Click Buy, atomically decrement stock, reserve for payment
- **Virtual queue**: Enqueue users with position if traffic exceeds capacity
- **Purchase limits**: Max 1-2 units per user per SKU (prevent scalpers)
- **Real-time stock display**: Show remaining stock count
- **Fairness**: First-come-first-served; no advantage from refreshing
- **Anti-bot**: Prevent automated bots from sniping all stock

---

## 2. Non-Functional Requirements (NFRs)

- **Extreme Throughput**: Handle 1M+ concurrent users at T=0
- **Low Latency**: Purchase decision in < 100 ms
- **Strong Consistency**: No overselling — if 1,000 units, exactly 1,000 orders max
- **Availability**: Graceful degradation under extreme load
- **Fairness**: FIFO ordering guaranteed
- **Idempotent**: Double-click must not create two orders

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Concurrent users at sale start | 1M+ |
| Purchase attempts / sec (T=0) | 500K |
| Items for sale | 1,000 - 10,000 units |
| Time to sell out | 5-30 seconds |
| Page load requests / sec | 2M (pre-sale refresh storm) |
| Bot traffic ratio | 30-50% of requests |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    PRE-SALE (T-24h to T-0)                             │
│                                                                        │
│  Admin Console --> Flash Sale Config Service --> PostgreSQL (sale defs) │
│                                                     |                  │
│                                              +------v------+          │
│                                              | Pre-warm     |          │
│                                              | Service      |          │
│                                              | - Pre-render |          │
│                                              |   static HTML|          │
│                                              | - Push to CDN|          │
│                                              | - Load stock |          │
│                                              |   into Redis |          │
│                                              | - Pre-scale  |          │
│                                              |   API servers|          │
│                                              +-------------+          │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    SALE ACTIVE (T=0 onwards)                           │
│                                                                        │
│  User Browser                                                          │
│       |                                                                │
│  +----v----------+                                                     │
│  | CDN (sale     | <-- Pre-rendered static HTML                        │
│  | page + JS     |     Countdown JS (client-side, no origin call)      │
│  | countdown)    |     Buy button revealed at T=0 by JS                │
│  +----+----------+     1M page loads -> zero origin traffic            │
│       |                                                                │
│       | (user clicks "Buy" at T=0)                                     │
│       |                                                                │
│  +----v----------+                                                     │
│  | WAF/CDN Edge  | <-- Bot detection layer                            │
│  | (Cloudflare)  |     Rate limit: 10 req/sec/IP                      │
│  |               |     JS challenge, JA3 fingerprint                   │
│  +----+----------+     Block known bot signatures                      │
│       |                                                                │
│  +----v----------+                                                     │
│  | Virtual Queue | <-- Controlled admission                            │
│  | Service       |     INCR queue_position -> show position            │
│  |               |     ZPOPMIN: admit 10K users/sec FIFO               │
│  | Redis:        |     Issue admission JWT (TTL 60s)                   │
│  |  sorted set   |     If stock gone -> broadcast SOLD_OUT             │
│  +----+----------+                                                     │
│       |                                                                │
│       | (admitted users with JWT)                                      │
│       |                                                                │
│  +----v----------+                                                     │
│  | Flash Sale    | <-- Purchase decision service                       │
│  | Service       |     Validate admission JWT                          │
│  | (stateless    |     Validate idempotency key                        │
│  |  API servers, |     Execute Redis Lua script (atomic):              │
│  |  pre-scaled)  |       - check user limit                            │
│  +----+----------+       - DECRBY stock (undo if < 0)                  │
│       |                  - INCRBY user count                           │
│       |                  Result: reserved | sold_out | limit_exceeded  │
│       |                                                                │
│  +----v----------+                                                     │
│  | Redis         | <-- Single source of truth during sale              │
│  | (flash_stock, |     All reads/writes via Lua script                 │
│  |  user_limits, |     WAIT 1 for replica sync (durability)            │
│  |  reservations)|     TTL 600 on reservations (10-min payment window) │
│  +----+----------+                                                     │
│       |                                                                │
│       | (on successful reservation)                                    │
│       |                                                                │
│  +----v----------+                                                     │
│  | Kafka         | <-- Decouple purchase from order creation           │
│  | (flash-order  |     Guaranteed delivery (at-least-once)             │
│  |  events)      |     Partitioned by sale_id                          │
│  +----+----------+                                                     │
│       |                                                                │
│  +----v----------+     +---------------+     +-------------------+     │
│  | Order Service |---->| Payment       |---->| Notification      |     │
│  | (create order,|     | Service       |     | Service           |     │
│  |  consume Kafka|     | (charge within|     | (confirmation     |     │
│  |  event,       |     |  10-min       |     |  email/push)      |     │
│  |  idempotent)  |     |  window)      |     |                   |     │
│  +----+----------+     +-------+-------+     +-------------------+     │
│       |                        |                                       │
│  +----v----------+     +-------v-------+                               │
│  | PostgreSQL    |     | On payment    |                               │
│  | (flash_sale_  |     | failure/      |                               │
│  |  orders,      |     | timeout:      |                               │
│  |  durable      |     | INCR stock    |                               │
│  |  records)     |     | back in Redis |                               │
│  +---------------+     +---------------+                               │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### The Critical Purchase Path — Redis Lua Script

The entire purchase decision must be atomic and < 1 ms:

```lua
-- Keys: flash_stock:{sale_id}:{sku_id}, user_limit:{sale_id}:{user_id}
-- Args: user_id, quantity

-- Step 1: Check per-user purchase limit
local user_purchased = redis.call('GET', KEYS[2])
if user_purchased and tonumber(user_purchased) >= 2 then
  return {0, 'LIMIT_EXCEEDED'}
end

-- Step 2: Atomic stock decrement
local remaining = redis.call('DECRBY', KEYS[1], ARGV[2])
if remaining < 0 then
  redis.call('INCRBY', KEYS[1], ARGV[2])  -- undo
  return {0, 'SOLD_OUT'}
end

-- Step 3: Record user purchase count
redis.call('INCRBY', KEYS[2], ARGV[2])
redis.call('EXPIRE', KEYS[2], 86400)

return {1, remaining}
```

**Why Lua script and not separate commands?** Entire script executes atomically in Redis. Separate DECR + check has a race window. 500K attempts/sec handled by single Redis node (single-threaded serialization).

#### Virtual Queue — Handling 1M Concurrent Users

Problem: 1M users hit Buy at T=0. Even if Redis can handle it, API servers and load balancers collapse under 500K concurrent TCP connections.

Solution: Virtual Queue (controlled admission)

```
T-5 min: Users "Enter Queue" early
  Position assigned: INCR queue_position:{sale_id} --> position 347,231
  User shown: "Your position: 347,231. Estimated wait: ~5 minutes"

T=0: Sale starts. Queue processes users FIFO.
  Gate rate: 10,000 users admitted per second (tunable)
  
  Admitted users:
  1. Receive short-lived JWT token (valid 60 seconds)
  2. Token authorizes call to purchase API
  3. Purchase API validates token --> runs Lua script on Redis
  
  Users not yet admitted:
  - See "Please wait..." with live position via WebSocket/SSE
  - Position updates every 5 seconds
  
  Stock gone:
  - Broadcast SOLD_OUT to ALL remaining queue members immediately
  - Don't make users wait if nothing left to buy

Queue implementation:
  Redis sorted set: ZADD queue:{sale_id} {timestamp} {user_id}
  Processing: ZPOPMIN queue:{sale_id} 10000  (pop 10K per second)
```

#### Anti-Bot Measures

```
Layer 1: CDN/WAF (Cloudflare, AWS WAF)
  - Rate limit per IP: max 10 req/sec
  - Known bot signatures blocked
  - JavaScript challenge (bots can't execute JS)
  - TLS fingerprinting (JA3 hash) -- flag non-browser clients

Layer 2: Queue Entry Validation
  - CAPTCHA at queue entry (invisible reCAPTCHA)
  - Device fingerprint (canvas hash, WebGL, screen resolution)
  - Account age check: accounts < 24 hours old --> blocked

Layer 3: Purchase Validation
  - One purchase per user_id (Redis user_limit)
  - One purchase per device_fingerprint
  - One purchase per payment method

Layer 4: Post-Purchase Fraud Detection
  - Multiple orders to same shipping address from different accounts --> cancel
  - Reseller pattern detection --> flag
```

---

## 5. APIs

### Enter Queue
```http
POST /api/v1/flash-sale/{sale_id}/enter-queue
{
  "captcha_token": "recaptcha-response-token",
  "device_fingerprint": "fp-hash-abc"
}
Response: 200 OK
{
  "queue_position": 12345,
  "estimated_wait_seconds": 120,
  "queue_token": "qt-uuid"
}
```

### Purchase (After Admitted)
```http
POST /api/v1/flash-sale/{sale_id}/purchase
Idempotency-Key: "purchase-user123-sale456"
Authorization: Bearer {admission_jwt}
{
  "sku_id": "SKU-FLASH-1",
  "quantity": 1
}
Response: 200 OK
{
  "status": "reserved",
  "reservation_token": "res-uuid",
  "payment_deadline": "2026-03-14T11:10:00Z",
  "remaining_stock": 423
}
OR { "status": "sold_out" }
OR { "status": "limit_exceeded" }
```

### Get Sale Status
```http
GET /api/v1/flash-sale/{sale_id}/status
Response: 200 OK
{
  "sale_id": "sale-456",
  "status": "active",
  "items": [
    {"sku_id": "SKU-FLASH-1", "name": "iPhone 16", "flash_price": 499.00,
     "original_price": 999.00, "total_stock": 1000, "remaining": 423}
  ]
}
```

---

## 6. Data Model

### Redis — Flash Sale State

```
flash_stock:{sale_id}:{sku_id}      --> INT (atomic DECR)
user_limit:{sale_id}:{user_id}      --> INT (max 2), TTL 86400
reservation:{token}                 --> JSON { user_id, sku_id, qty }, TTL 600
queue_position:{sale_id}            --> INT (INCR for each entrant)
queue:{sale_id}                     --> Sorted Set { user_id: timestamp }
admission:{sale_id}:{user_id}       --> "admitted", TTL 60
device:{sale_id}:{fingerprint}      --> user_id, TTL 86400
```

### PostgreSQL — Durable Records

```sql
CREATE TABLE flash_sales (
    sale_id         UUID PRIMARY KEY,
    name            VARCHAR(255),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    status          ENUM('scheduled','active','ended','cancelled'),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE flash_sale_items (
    sale_id         UUID NOT NULL,
    sku_id          VARCHAR(50) NOT NULL,
    flash_price     DECIMAL(10,2) NOT NULL,
    original_price  DECIMAL(10,2) NOT NULL,
    total_stock     INT NOT NULL,
    sold_count      INT DEFAULT 0,
    PRIMARY KEY (sale_id, sku_id)
);

CREATE TABLE flash_sale_orders (
    order_id        UUID PRIMARY KEY,
    sale_id         UUID NOT NULL,
    user_id         UUID NOT NULL,
    sku_id          VARCHAR(50) NOT NULL,
    quantity        INT NOT NULL,
    price           DECIMAL(10,2),
    status          ENUM('reserved','paid','cancelled','expired'),
    reservation_token VARCHAR(64),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_sale_user (sale_id, user_id)
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Redis crash mid-sale** | Redis Cluster + WAIT 1 for replica sync; safety buffer (load 990/1000); post-sale reconciliation |
| **Overselling** | Lua script is atomic; remaining < 0 check with undo via INCRBY |
| **Payment timeout** | Reservation TTL 10 min; expired reservations auto-INCR stock back |
| **Double purchase** | Idempotency key + user_limit in Lua script |
| **1M page loads** | CDN pre-rendered static pages; zero origin load until Buy click |
| **Bot sniping** | Multi-layer: CAPTCHA, device fingerprint, rate limit, account age |
| **Queue fairness** | Sorted set with timestamp score — strict FIFO ordering |

### Specific: Redis Data Loss Mid-Sale

If primary crashes, replica may miss last 1-2 sec of DECRs — up to 10-20 items oversold.

Mitigations:
1. **WAIT command**: `WAIT 1 0` after Lua script ensures at least 1 replica ACKs before responding (+1ms latency)
2. **Safety buffer**: For 1000-unit sale, load only 990 into Redis. Hold 10 as buffer for replication lag.
3. **Post-sale reconciliation**: Count confirmed orders in PostgreSQL. If orders > total_stock, cancel excess (last-in-first-cancelled). Notify affected users within minutes.

---

## 8. Deep Dive: Engineering Trade-offs

### Pre-Rendered Static Pages — Surviving the Traffic Spike

CDN serves static sale page to 1M users with zero origin load. JS countdown is client-side. At T=0 JS reveals Buy button. Only purchase API calls hit origin (gated by virtual queue to ~10K/sec). Without this: 1M simultaneous requests = origin crash.

### Countdown Precision

Fetch server time on page load once, compute client clock drift, adjust countdown. Server-side gate rejects any Buy request before `sale.start_time` regardless of client clock. Ultimate safeguard against clock skew.

### Queue vs No Queue

```
No Queue:
  500K req/sec all hit API + Redis directly.
  Redis handles it. BUT: API servers, load balancers, network with
  500K concurrent connections --> likely crash.
  Result: errors, unfair (network latency determines winners).

Virtual Queue (recommended):
  All users enqueued. Admitted at controlled rate (10K/sec).
  API servers handle 10K/sec comfortably.
  Result: fair (FIFO), stable (no crashes), users wait 30-60s in queue.
  Trade-off: slight wait for fairness and stability.
```

### Why Not Just Auto-Scale?

Auto-scaling takes 2-5 minutes to spin up new instances. Flash sale goes from 0 to 500K req/sec in < 1 second. By the time scaling kicks in, sale is OVER (sold out in 5-30 sec). Solution: Pre-provision servers 24h before + CDN for page loads + Queue for API gating + Redis for stock (single node, no scaling needed).

