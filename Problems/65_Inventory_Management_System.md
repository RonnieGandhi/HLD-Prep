# 65. Design an Inventory Management System

---

## 1. Functional Requirements (FR)

- **Track stock levels**: Real-time quantity tracking per SKU across warehouses, stores, and channels
- **Stock reservation**: Temporarily hold inventory for pending orders (soft lock) before payment confirmation
- **Stock decrement**: Atomically reduce stock on confirmed purchase; prevent overselling
- **Multi-warehouse**: Track inventory per warehouse/fulfillment center with inter-warehouse transfers
- **Replenishment alerts**: Notify when stock falls below reorder point (low-stock threshold)
- **Batch updates**: Bulk ingest from suppliers, warehouse scans, POS systems
- **Stock adjustments**: Manual adjustments for damage, theft, counting discrepancies
- **Inventory holds**: Reserve stock for flash sales, bundles, pre-orders before they go live
- **Multi-channel sync**: Sync available stock across website, mobile app, marketplace (Amazon, eBay), and physical stores
- **Audit trail**: Immutable log of every stock change with reason, actor, and timestamp

---

## 2. Non-Functional Requirements (NFRs)

- **Strong Consistency**: Stock counts MUST be accurate — overselling is unacceptable
- **Low Latency**: Stock check in < 10 ms; reservation in < 50 ms
- **High Throughput**: Handle 100K+ stock operations/sec during flash sales
- **Availability**: 99.99% — inventory service is in the critical path for every checkout
- **Durability**: Stock data never lost; every mutation is logged
- **Idempotency**: Duplicate requests (network retries) must not double-decrement stock
- **Scalability**: 100M+ SKUs across 1,000+ warehouses

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total SKUs | 100M |
| Warehouses / fulfillment centers | 1,000 |
| SKU-warehouse combinations | ~500M (not every SKU in every warehouse) |
| Stock check queries / sec | 200K (product page views trigger stock check) |
| Reservation requests / sec | 10K (add to cart / checkout) |
| Confirmed decrements / sec | 5K (orders placed) |
| Flash sale peak | 500K stock ops/sec for hot SKUs |
| Stock record size | ~200 bytes |
| Total data | 500M × 200B = 100 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  E-Commerce  │    │  Marketplace │    │  POS / Store │
│  Website     │    │  Channels    │    │  System      │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼───────┐
                     │  API Gateway │
                     └──────┬───────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                   │
  ┌──────▼──────┐   ┌──────▼──────┐   ┌────────▼────────┐
  │  Stock      │   │ Reservation │   │  Channel Sync   │
  │  Query      │   │ Service     │   │  Service        │
  │  Service    │   │ (Reserve /  │   │  (Publish avail │
  │  (Read)     │   │  Confirm /  │   │   to all        │
  │             │   │  Release)   │   │   channels)     │
  └──────┬──────┘   └──────┬──────┘   └────────┬────────┘
         │                  │                    │
    ┌────▼────┐      ┌─────▼─────┐              │
    │  Redis  │      │ PostgreSQL│◀─────────────┘
    │ (Stock  │◀────▶│ (Source   │
    │  Cache) │      │  of Truth)│
    └─────────┘      └─────┬─────┘
                           │
                    ┌──────▼───────┐
                    │    Kafka     │
                    │ (Stock change│
                    │  events)     │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                  │
  ┌──────▼──────┐  ┌──────▼──────┐  ┌────────▼────────┐
  │ Replenish-  │  │  Audit Log  │  │  Analytics /    │
  │ ment Alert  │  │  Writer     │  │  Forecasting    │
  │ Service     │  │ (ClickHouse)│  │  Service        │
  └─────────────┘  └─────────────┘  └─────────────────┘
```

### Component Deep Dive

#### Stock Reservation Flow — The Two-Phase Pattern

```
Why two phases? Because payment takes time (30 sec – 5 min).
Without reservation, two users could both see "1 in stock" and both try to buy.

Phase 1: RESERVE (at checkout initiation)
  User clicks "Buy Now" → Reserve 1 unit of SKU-123 at Warehouse-NYC

  BEGIN TRANSACTION;
    SELECT available, reserved FROM inventory 
    WHERE sku_id = 'SKU-123' AND warehouse_id = 'WH-NYC' FOR UPDATE;
    
    -- Check: available - reserved >= requested_quantity
    IF (available - reserved) >= 1 THEN
      UPDATE inventory SET reserved = reserved + 1 
      WHERE sku_id = 'SKU-123' AND warehouse_id = 'WH-NYC';
      
      INSERT INTO reservations (reservation_id, sku_id, warehouse_id, 
                                quantity, user_id, expires_at, status)
      VALUES (uuid, 'SKU-123', 'WH-NYC', 1, 'user-1', NOW() + '10 min', 'active');
    ELSE
      RAISE 'Insufficient stock';
    END IF;
  COMMIT;

  Reservation has a TTL (10 minutes). If user doesn't complete payment → auto-release.

Phase 2a: CONFIRM (payment successful)
  UPDATE inventory SET available = available - 1, reserved = reserved - 1
  WHERE sku_id = 'SKU-123' AND warehouse_id = 'WH-NYC';
  
  UPDATE reservations SET status = 'confirmed' WHERE reservation_id = ?;

Phase 2b: RELEASE (payment failed / timeout / user abandoned cart)
  UPDATE inventory SET reserved = reserved - 1
  WHERE sku_id = 'SKU-123' AND warehouse_id = 'WH-NYC';
  
  UPDATE reservations SET status = 'released' WHERE reservation_id = ?;

Key fields in inventory table:
  available:  total physical stock (what warehouse actually has)
  reserved:   temporarily held for pending orders
  sellable:   available - reserved (what new customers can buy)

Expired reservation cleanup:
  Cron job every 1 minute:
    SELECT * FROM reservations WHERE status = 'active' AND expires_at < NOW();
    For each expired: release back to available pool
  
  OR: Use Redis with TTL for reservations (automatic expiry)
    SET reservation:{reservation_id} {details} EX 600
    On expiry → Keyspace notification → trigger release
```

#### Flash Sale Concurrency — Redis-Based Stock Counter

```
Problem: Flash sale of 1,000 units. 500K users hit "Buy" simultaneously.
  PostgreSQL: SELECT ... FOR UPDATE → row-level lock → 500K waiting → timeout/crash

Solution: Use Redis as the fast path for stock decrements.

Pre-sale setup:
  SET flash_stock:{sku_id} 1000

On purchase attempt:
  local remaining = redis.call('DECR', 'flash_stock:SKU-FLASH-1')
  if remaining >= 0 then
    -- SUCCESS: user got one
    -- Async: write to PostgreSQL, create order
    return 'reserved'
  else
    -- SOLD OUT
    redis.call('INCR', 'flash_stock:SKU-FLASH-1')  -- undo the DECR
    return 'sold_out'
  end

Why this works:
  Redis DECR is atomic (single-threaded) → no race conditions
  500K DECR operations/sec → Redis handles easily (100K ops/sec per shard)
  After Redis confirms → async write to PostgreSQL (no lock contention)

Edge case: User gets Redis reservation but payment fails
  → Redis counter was decremented → stock is "lost"
  
  Solution: Reservation with TTL
    DECR returns remaining → if success → SET flash_reservation:{user_id}:{sku_id} 1 EX 300
    If payment fails/timeout → INCR flash_stock:{sku_id} (release)
    
  Cleanup job: every 30 sec, check for expired reservations → INCR stock back

Consistency between Redis and PostgreSQL:
  Redis is the hot path (speed). PostgreSQL is source of truth (durability).
  After Redis DECR → publish Kafka event → worker writes to PostgreSQL
  Periodic reconciliation: compare Redis counter with PostgreSQL available - reserved
  If mismatch → alert + auto-correct Redis from PostgreSQL
```

#### Multi-Warehouse Stock Allocation

```
User orders SKU-123. It's available in 3 warehouses:
  WH-NYC: 50 units (200 miles from user)
  WH-CHI: 30 units (700 miles)  
  WH-LAX: 100 units (2500 miles)

Which warehouse fulfills the order?

Allocation strategies:

1. Nearest warehouse (minimize shipping cost + time):
   Sort warehouses by distance to delivery address
   Pick closest with stock → WH-NYC ✓

2. Load-balanced (prevent one warehouse from depleting):
   Pick warehouse with highest stock level → WH-LAX

3. Cost-optimized (minimize total fulfillment cost):
   cost = shipping_cost + handling_cost + last_mile_cost
   Consider: shipping zone, carrier rates, warehouse labor cost
   Pick minimum cost

4. Hybrid ⭐ (Amazon's approach):
   score = w1 × (1 / distance) + w2 × stock_level + w3 × (1 / cost) 
           + w4 × delivery_speed_guarantee
   Pick highest score

Split shipment:
  Order has 3 items: SKU-A (only in WH-NYC), SKU-B (only in WH-LAX), SKU-C (both)
  → Split into 2 shipments: {SKU-A, SKU-C} from WH-NYC, {SKU-B} from WH-LAX
  → Minimize number of shipments while respecting stock constraints
  → NP-hard problem at scale → greedy heuristic: maximize items per shipment
```

---

## 5. APIs

### Check Stock
```http
GET /api/v1/inventory/{sku_id}/stock?warehouse_id=WH-NYC
Response: 200 OK
{
  "sku_id": "SKU-123",
  "warehouse_id": "WH-NYC",
  "available": 50,
  "reserved": 5,
  "sellable": 45,
  "low_stock": false,
  "reorder_point": 10
}
```

### Reserve Stock
```http
POST /api/v1/inventory/reserve
Idempotency-Key: "order-uuid-abc"
{
  "items": [
    {"sku_id": "SKU-123", "quantity": 2, "warehouse_id": "WH-NYC"}
  ],
  "ttl_seconds": 600,
  "user_id": "user-uuid"
}
Response: 200 OK
{
  "reservation_id": "res-uuid",
  "status": "reserved",
  "expires_at": "2026-03-14T11:10:00Z",
  "items": [{"sku_id": "SKU-123", "reserved_qty": 2, "warehouse_id": "WH-NYC"}]
}
```

### Confirm Reservation (After Payment)
```http
POST /api/v1/inventory/confirm
{
  "reservation_id": "res-uuid",
  "order_id": "order-uuid"
}
Response: 200 OK
{ "status": "confirmed" }
```

### Bulk Stock Update (Warehouse Ingest)
```http
POST /api/v1/inventory/bulk-update
{
  "warehouse_id": "WH-NYC",
  "updates": [
    {"sku_id": "SKU-123", "available": 200, "reason": "supplier_shipment"},
    {"sku_id": "SKU-456", "available": 0, "reason": "discontinued"}
  ]
}
Response: 200 OK
{ "updated": 2, "failed": 0 }
```

---

## 6. Data Model

### PostgreSQL — Source of Truth (Sharded by sku_id)

```sql
CREATE TABLE inventory (
    sku_id          VARCHAR(50) NOT NULL,
    warehouse_id    VARCHAR(50) NOT NULL,
    available       INT NOT NULL DEFAULT 0 CHECK (available >= 0),
    reserved        INT NOT NULL DEFAULT 0 CHECK (reserved >= 0),
    reorder_point   INT DEFAULT 10,
    max_stock       INT DEFAULT 1000,
    last_replenished TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (sku_id, warehouse_id),
    CHECK (reserved <= available)
);

CREATE TABLE reservations (
    reservation_id  UUID PRIMARY KEY,
    sku_id          VARCHAR(50) NOT NULL,
    warehouse_id    VARCHAR(50) NOT NULL,
    user_id         UUID NOT NULL,
    order_id        UUID,
    quantity        INT NOT NULL,
    status          ENUM('active', 'confirmed', 'released', 'expired') DEFAULT 'active',
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_status_expires (status, expires_at) WHERE status = 'active',
    INDEX idx_sku (sku_id, warehouse_id)
);

CREATE TABLE stock_audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    sku_id          VARCHAR(50) NOT NULL,
    warehouse_id    VARCHAR(50) NOT NULL,
    change_type     ENUM('reserve', 'confirm', 'release', 'adjust', 'replenish', 'return'),
    quantity_change INT NOT NULL,
    previous_available INT,
    new_available   INT,
    reason          TEXT,
    actor_id        VARCHAR(50),
    idempotency_key VARCHAR(64),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_sku_time (sku_id, created_at DESC)
);
```

### Redis — Hot Path Cache + Flash Sale

```
# Sellable stock cache (read path)
stock:{sku_id}:{warehouse_id}  → INT (sellable = available - reserved)
TTL: 60 seconds (refreshed from PostgreSQL)

# Flash sale atomic counters
flash_stock:{sale_id}:{sku_id}  → INT
No TTL (managed explicitly)

# Reservation TTL keys
reservation:{reservation_id}   → JSON { sku_id, warehouse_id, quantity, user_id }
TTL: 600 (10 minutes)

# Idempotency (prevent duplicate reserve/confirm)
idempotency:{key}  → response JSON
TTL: 86400
```

### Kafka Topics

```
Topic: stock-changes         (every mutation → consumed by channel sync, analytics, alerts)
Topic: reservation-events    (reserved, confirmed, released → consumed by order service)
Topic: low-stock-alerts      (sku dropped below reorder point → notify procurement)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Overselling** | Reservation pattern + PostgreSQL CHECK constraint + Redis DECR atomic |
| **Reservation leak (never confirmed/released)** | TTL-based expiry + cron cleanup job every 1 min |
| **Redis ↔ PostgreSQL divergence** | Periodic reconciliation (every 5 min); Redis is cache, PostgreSQL is truth |
| **Double decrement (retry)** | Idempotency key on every reserve/confirm request |
| **Warehouse system offline** | Queue updates in Kafka; apply when system recovers; use last-known stock |
| **Flash sale thundering herd** | Redis absorbs all traffic; PostgreSQL writes batched asynchronously |
| **Database failover** | PostgreSQL synchronous replication; promote standby on primary failure |

### Specific: Reconciliation Between Redis and PostgreSQL

```
Problem: Redis says 45 available, PostgreSQL says 43. Which is right?

Root causes:
  - Reservation confirmed in PostgreSQL but Redis update failed (network blip)
  - Redis key expired and was re-populated from stale PostgreSQL read
  - Flash sale Redis DECR succeeded but async PostgreSQL write was lost

Reconciliation job (runs every 5 minutes):
  For each SKU in Redis:
    pg_sellable = SELECT available - reserved FROM inventory WHERE sku_id = ?
    redis_sellable = GET stock:{sku_id}:{warehouse_id}
    
    IF abs(pg_sellable - redis_sellable) > 0:
      SET stock:{sku_id}:{warehouse_id} {pg_sellable}
      LOG warning: "Reconciled SKU-123: Redis had {redis_sellable}, PG has {pg_sellable}"
  
  For flash sales:
    Compare flash_stock:{sku_id} with PostgreSQL
    If mismatch > 5 units → alert ops (potential overselling)

PostgreSQL is ALWAYS the source of truth.
Redis is a performance optimization that may drift.
```

---

## 8. Deep Dive: Engineering Trade-offs

### PostgreSQL vs DynamoDB for Inventory

```
PostgreSQL ⭐:
  ✓ ACID transactions: critical for reserve → confirm atomicity
  ✓ CHECK constraints: available >= 0, reserved <= available
  ✓ SELECT ... FOR UPDATE: row-level locking for safe concurrent updates
  ✓ Rich queries for analytics: "total stock across all warehouses"
  ✗ Single-writer bottleneck for hot SKUs (millions of updates to same row)
  
  Scaling: shard by sku_id hash → each shard handles subset of SKUs
  Hot SKU (flash sale): Redis fronts the hot path; PostgreSQL is async

DynamoDB:
  ✓ Horizontal scalability (no sharding management)
  ✓ Conditional writes: UpdateExpression with ConditionExpression
    SET available = available - 1 WHERE available >= 1
  ✓ Auto-scaling for burst traffic
  ✗ No multi-row transactions (25-item limit)
  ✗ Eventual consistency by default (need strongly consistent reads)
  ✗ More expensive at high write throughput

Decision: PostgreSQL for most e-commerce (ACID is critical).
  DynamoDB if: AWS-native, massive scale (100M+ SKUs), can tolerate 25-item txn limit.
  Both: Redis in front for read-heavy + flash-sale write-heavy paths.
```

### Inventory: Push vs Pull for Channel Sync

```
Push (event-driven) ⭐:
  Stock changes → Kafka event → Channel Sync Service → update Amazon/eBay/Shopify
  Latency: 1-5 seconds from stock change to channel update
  ✓ Near real-time; prevents overselling across channels
  ✗ Requires reliable event delivery (Kafka at-least-once)

Pull (polling):
  Each channel polls our API every 30 seconds for stock updates
  ✓ Simple; channel controls frequency
  ✗ 30-second stale window → overselling risk
  ✗ Wasteful: 90% of polls return "no change"

Hybrid:
  Push for critical changes (stock < 10 → urgent)
  Pull as fallback reconciliation (every 5 minutes)
  
  For safety buffer: when stock < 5, set channel stock = 0
    Prevents overselling during push delivery lag
    "Phantom out-of-stock" is better than overselling
```



