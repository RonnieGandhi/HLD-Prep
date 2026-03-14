# 33. Design an Event Sourcing System

---

## 1. Functional Requirements (FR)

- **Append events**: Store all state changes as immutable, ordered events (not mutable state)
- **Rebuild state**: Reconstruct current entity state by replaying events from the beginning
- **Event replay**: Replay events to rebuild read models, fix bugs, or create new projections
- **Snapshots**: Periodically save materialized state to avoid replaying all events every time
- **Projections**: Build multiple read-optimized views from the same event stream
- **Temporal queries**: "What was the state of entity X at time T?"
- **Event versioning**: Handle schema evolution (new fields, renamed events)
- **Pub/Sub**: Downstream services subscribe to event streams reactively

---

## 2. Non-Functional Requirements (NFRs)

- **Durability**: Events are the source of truth — must NEVER be lost (11 nines)
- **Immutability**: Events are append-only; never updated or deleted
- **Strong ordering**: Events within an aggregate must be strictly ordered
- **High Throughput**: Support 100K+ events/sec writes
- **Low Latency Reads**: Projections served in < 10 ms
- **Scalability**: Billions of events across millions of aggregates
- **Schema Evolution**: Support backward/forward compatible event schema changes

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Events / day | 5B |
| Events / sec | ~58K (peak 200K) |
| Avg event size | 500 bytes |
| Write throughput | 29 MB/s avg, 100 MB/s peak |
| Storage / day | 2.5 TB |
| Storage / year | ~900 TB |
| Aggregates (entities) | 500M |
| Avg events per aggregate | 50 |

---

## 4. High-Level Design (HLD)

```
                    ┌──────────────────────────────────────┐
                    │         Write Path (Commands)         │
                    │                                       │
                    │  ┌──────────────┐                    │
                    │  │  Client      │                    │
                    │  │  POST /orders│                    │
                    │  └──────┬───────┘                    │
                    │         │                             │
                    │  ┌──────▼───────┐                    │
                    │  │ Command      │                    │
                    │  │ Validator    │                    │
                    │  │ (API Layer)  │                    │
                    │  └──────┬───────┘                    │
                    │         │                             │
                    │  ┌──────▼───────────────────────┐    │
                    │  │ Aggregate Root (Domain Logic) │    │
                    │  │                               │    │
                    │  │ 1. Load current state:        │    │
                    │  │    a) Load snapshot (v100)     │    │
                    │  │    b) Replay events v101-v105  │    │
                    │  │    c) Current state = v105     │    │
                    │  │                               │    │
                    │  │ 2. Validate business rules:    │    │
                    │  │    "Can this order be shipped?"│    │
                    │  │    assert(status == 'paid')    │    │
                    │  │                               │    │
                    │  │ 3. Generate new event:         │    │
                    │  │    OrderShipped{tracking:"X"}  │    │
                    │  └──────┬───────────────────────┘    │
                    │         │                             │
                    │  ┌──────▼─────────────────────────┐  │
                    │  │ Event Store (PostgreSQL)        │  │
                    │  │                                 │  │
                    │  │ INSERT INTO events              │  │
                    │  │ (stream_id, version, type, data)│  │
                    │  │ VALUES ('order-123', 106, ...)  │  │
                    │  │                                 │  │
                    │  │ Optimistic Concurrency:         │  │
                    │  │ UNIQUE(stream_id, version)      │  │
                    │  │ If version 106 exists → CONFLICT│  │
                    │  │ → Retry: re-load, re-validate   │  │
                    │  └──────┬─────────────────────────┘  │
                    └─────────┼─────────────────────────────┘
                              │
                              │ CDC (Change Data Capture) or
                              │ Outbox Pattern → publish to Kafka
                              ▼
                    ┌─────────────────────────────────────┐
                    │           Kafka Event Bus            │
                    │  Topic: domain-events                │
                    │  Partitioned by: aggregate_id        │
                    │  Retention: forever (compacted)      │
                    └─────────┬───────────────────────────┘
                              │
              ┌───────────────┼──────────────────┐
              │               │                  │
    ┌─────────▼────┐  ┌──────▼──────┐  ┌────────▼──────┐
    │ Projection   │  │ Projection  │  │ Downstream    │
    │ Builder:     │  │ Builder:    │  │ Services      │
    │ Order View   │  │ Analytics   │  │ Notifications │
    │              │  │             │  │ Billing       │
    │ Tracks:      │  │ Tracks:     │  │               │
    │ Kafka offset │  │ Kafka offset│  │               │
    └──────┬───────┘  └──────┬──────┘  └───────────────┘
           │                 │
    ┌──────▼───────┐  ┌──────▼──────┐
    │ PostgreSQL   │  │ ClickHouse  │
    │ (Read Model) │  │ (Analytics) │
    │ Denormalized │  │             │
    │ for fast     │  │             │
    │ lookups      │  │             │
    └──────┬───────┘  └─────────────┘
           │
    ┌──────▼───────┐         ┌──────────────┐
    │ Redis Cache  │         │  Read Path   │
    │ (TTL: 60s)   │◄────────│  GET /orders │
    └──────────────┘         └──────────────┘
```

### Component Deep Dive

#### Event Store — The Source of Truth

**Data Model**: Each aggregate (entity) has a stream of ordered events:
```
Stream ID: order-12345
  Version 1: {type: "OrderCreated", data: {items: [...], total: 100}, timestamp: ...}
  Version 2: {type: "PaymentReceived", data: {amount: 100, method: "card"}, timestamp: ...}
  Version 3: {type: "OrderShipped", data: {tracking: "UPS123"}, timestamp: ...}
```

**Optimistic Concurrency**:
```
When writing event to stream "order-12345":
  Expected version: 3 (client knows current version)
  New event version: 4
  
  IF current stream version ≠ 3 → CONFLICT (another writer modified it)
  → Client must re-read, re-validate business rules, retry
  
  This is CAS (Compare-And-Swap) at the stream level
  No locks needed, no deadlocks possible
```

**Why not a regular DB with UPDATE?**
```
Traditional CRUD:
  UPDATE orders SET status='shipped' WHERE id=12345;
  → Previous state LOST. Audit trail requires separate logging.
  → "Why did this order get cancelled?" → No history.

Event Sourcing:
  APPEND {OrderCancelled, reason: "customer_request", cancelled_by: "user-789"}
  → Complete history preserved. Every state transition recorded.
  → Time-travel: replay events up to any point to see past state.
  → Debug: exact sequence of operations that led to current state.
```

#### Snapshots — Performance Optimization

```
Problem: Order with 10,000 events → rebuilding state requires replaying all 10,000
Solution: Periodically save a snapshot (materialized state at version N)

Rebuilding with snapshot:
  1. Load latest snapshot (version 9,900, state = {...})
  2. Replay only events 9,901 to 10,000 (100 events instead of 10,000)
  3. Current state reconstructed in milliseconds

Snapshot strategy:
  - Every N events (e.g., every 100)
  - When event count exceeds threshold
  - On-demand when entity is frequently read
```

#### Projections — CQRS Read Models

```
Event Stream → Projection Builder → Read-Optimized Database

Example projections from the same "order" events:

Projection 1: Order Details (PostgreSQL)
  OrderCreated → INSERT INTO orders (id, items, total, status='created')
  PaymentReceived → UPDATE orders SET status='paid', payment_method=...
  OrderShipped → UPDATE orders SET status='shipped', tracking=...

Projection 2: Daily Revenue (ClickHouse)
  PaymentReceived → INSERT INTO revenue (date, amount, currency)

Projection 3: Customer Order Count (Redis)
  OrderCreated → INCR user:{user_id}:order_count

Each projection is independently rebuildable by replaying all events
```

---

## 5. APIs

### Command (Write)
```http
POST /api/v1/orders
{ "items": [{"product_id": "p1", "qty": 2}], "customer_id": "c123" }
→ Generates: OrderCreated event

POST /api/v1/orders/{order_id}/ship
{ "tracking_number": "UPS123" }
→ Generates: OrderShipped event (only if order status is 'paid')
```

### Query (Read — from projections)
```http
GET /api/v1/orders/{order_id}
→ Returns current state from read model (PostgreSQL projection)

GET /api/v1/orders/{order_id}/history
→ Returns all events for this order (from event store)

GET /api/v1/orders/{order_id}/state-at?timestamp=2026-01-15T00:00:00Z
→ Replays events up to given timestamp, returns state at that point
```

---

## 6. Data Model

### Event Store (PostgreSQL / EventStoreDB)

```sql
CREATE TABLE events (
    global_position  BIGSERIAL,           -- global ordering
    stream_id        VARCHAR(256) NOT NULL, -- e.g., "order-12345"
    stream_version   INT NOT NULL,         -- per-stream ordering
    event_type       VARCHAR(128) NOT NULL, -- e.g., "OrderCreated"
    data             JSONB NOT NULL,       -- event payload
    metadata         JSONB,               -- correlation_id, causation_id, user_id
    created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id, stream_version),
    UNIQUE (global_position)
);

CREATE TABLE snapshots (
    stream_id        VARCHAR(256),
    stream_version   INT,
    state            JSONB NOT NULL,
    created_at       TIMESTAMP,
    PRIMARY KEY (stream_id)
);
```

### Kafka — Event Bus
```
Topic: domain-events (partitioned by stream_id → ordering per aggregate)
Message: { stream_id, event_type, data, metadata, version, timestamp }
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Event loss** | PostgreSQL WAL + synchronous replication. Kafka RF=3 |
| **Projection failure** | Projection stores its last processed position. On restart, resumes from that position |
| **Projection corruption** | Delete projection, replay ALL events from event store to rebuild from scratch |
| **Concurrent writes** | Optimistic concurrency (expected version check) prevents conflicts |
| **Schema changes** | Event upcasters transform old event formats to new on read |
| **Event store unavailable** | Write-ahead to local WAL, sync when store recovers |

---

## 8. Additional Considerations

### Event Versioning / Schema Evolution
```
V1: OrderCreated { items: [...], total: 100 }
V2: OrderCreated { items: [...], subtotal: 100, tax: 8, total: 108 }

Strategy: Upcaster (transform V1 → V2 on read):
  if event.version == 1:
    event.data.subtotal = event.data.total
    event.data.tax = 0
  return event
```

### CQRS + Event Sourcing Pattern
- **CQRS** (Command Query Responsibility Segregation): Separate write and read models
- Write side: Event Store (append-only, normalized)
- Read side: Projections (denormalized, query-optimized)
- Sync: Events flow from write to read via Kafka/subscription

### When NOT to Use Event Sourcing
- Simple CRUD applications (overhead not justified)
- When audit trail is not valuable
- When schema changes frequently and upcasters become unmanageable
- When team lacks experience (steep learning curve)

---

## 9. Deep Dive: Engineering Trade-offs

### Event Store: PostgreSQL vs EventStoreDB vs Kafka

| Feature | PostgreSQL | EventStoreDB | Kafka |
|---|---|---|---|
| **Optimistic concurrency** | ✓ (UNIQUE constraint) | ✓ (built-in) | ✗ (no per-key versioning) |
| **Stream subscriptions** | Polling or LISTEN/NOTIFY | ✓ (native catch-up subscriptions) | ✓ (consumer groups) |
| **Ordering** | Per-stream + global position | Per-stream + global | Per-partition only |
| **Query flexibility** | ✓ (SQL) | Limited | ✗ (offset-based only) |
| **Scalability** | Sharding required | Clustering | Excellent (native partitioning) |
| **Maturity** | Very mature | Niche (Greg Young) | Very mature |

**Recommendation**: PostgreSQL for < 100K events/sec (simple, flexible). EventStoreDB for pure event sourcing workloads. Kafka as the event bus (not as the primary event store, because it lacks per-stream optimistic concurrency).

---

## Race Conditions and Concurrency Challenges

#### 1. Optimistic Concurrency Conflict — Two Writers Same Aggregate

```
Two users simultaneously try to update order-123 (current version: 5)

  Thread A: Load order-123 (version 5)
  Thread B: Load order-123 (version 5)
  Thread A: Validate → generate event → INSERT (stream='order-123', version=6, ...)
  Thread A: SUCCESS (version 6 written)
  Thread B: Validate → generate event → INSERT (stream='order-123', version=6, ...)
  Thread B: CONFLICT! (UNIQUE constraint on stream_id + version)
  
  Thread B must:
    1. Re-load order-123 (now at version 6, reflecting Thread A's change)
    2. Re-validate business rules (may no longer be valid!)
    3. If still valid → retry with version 7
    4. If invalid → return error to user
    
  Why optimistic concurrency is better than pessimistic locks:
    - No deadlocks (no locks held across requests)
    - High throughput for non-conflicting writes (most writes are to DIFFERENT aggregates)
    - Conflicts are rare (< 0.1% for most workloads)
    - When conflict occurs, cost is one retry (acceptable)
```

#### 2. Event Store → Kafka Delivery Gap (Dual Write Problem)

```
Problem: Writing to Event Store AND publishing to Kafka is a dual write.
  If one succeeds and the other fails → inconsistency.

  Scenario: Event written to PostgreSQL, but Kafka publish fails
  → Projections never updated → read model permanently stale

Solutions:

  1. Outbox Pattern ⭐:
     Write event + outbox entry in SAME database transaction:
       BEGIN;
         INSERT INTO events (stream_id, version, ...) VALUES (...);
         INSERT INTO outbox (event_id, topic, payload) VALUES (...);
       COMMIT;
     
     Separate "Outbox Relay" process polls outbox table → publishes to Kafka
     On success → delete from outbox (or mark as published)
     
     Guarantees: Exactly-once DB write + at-least-once Kafka publish
  
  2. CDC (Change Data Capture) ⭐:
     Use Debezium to capture PostgreSQL WAL changes
     Any INSERT to events table → automatically published to Kafka
     No application-level dual write needed
     
     Guarantees: Zero gap (WAL is the single source of truth)
  
  3. Event Store with built-in subscriptions:
     EventStoreDB has native catch-up subscriptions
     Consumers subscribe to streams directly, no Kafka needed
```

#### 3. Projection Lag — Stale Read Model

```
Write at T=0: Event "OrderShipped" written to event store
Kafka publish at T=0.05: Event arrives in Kafka topic
Projection builder at T=0.5: Reads event, updates read model

Query at T=0.1: GET /orders/123 → returns status="paid" (STALE!)
  The read model hasn't been updated yet. User sees old state for 400ms.

Mitigation strategies:
  1. Read-your-own-writes: After command, return the event directly
     (don't redirect to read model). Client uses the event data.
  
  2. Version check: Command returns new version number.
     Read API accepts ?min_version=106. If read model < 106 → wait or retry.
  
  3. Synchronous projection: For critical projections, update in-process
     (same transaction as event write). Slower but consistent.
  
  4. Accept eventual consistency: For dashboards and listings,
     0.5-second lag is invisible to users. Only matters for the writer.
```

#### 4. Snapshot Consistency — When Snapshots Lie

```
Snapshot at version 100: {status: "paid", items: [...], total: 100}
Events 101-105 played on top of snapshot to get current state.

Problem: If snapshot was corrupted or created from a bug:
  Snapshot says total=100, but replaying events 1-100 gives total=108
  → All subsequent state derived from bad snapshot is WRONG

Solutions:
  1. Validate snapshots: Periodically replay events from scratch and compare
     with snapshot. If different → delete snapshot and rebuild.
  
  2. Immutable events are the source of truth, NOT snapshots.
     Snapshot is just a performance optimization.
     When in doubt → delete all snapshots → rebuild from events.
  
  3. Snapshot versioning: Include snapshot schema version.
     If aggregate logic changes, invalidate old snapshots.
```
