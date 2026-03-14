# 101. Design a Change Data Capture (CDC) Pipeline

---

## 1. Functional Requirements (FR)

- Capture every INSERT, UPDATE, DELETE from source databases in real-time
- Propagate changes to downstream consumers (Kafka, Elasticsearch, data warehouse, caches) with < 5 second latency
- Maintain strict ordering of changes per row/primary key
- Support initial snapshot (full table copy) + ongoing incremental streaming
- Schema evolution: handle column adds, renames, type changes without pipeline restart
- Multi-source support: PostgreSQL, MySQL, MongoDB, SQL Server, Oracle
- Multi-sink support: Kafka topics, Elasticsearch, S3/Parquet, Redis, ClickHouse
- Exactly-once delivery guarantee (no duplicates, no missed changes)
- Filtering: capture only specific tables/columns (reduce noise)
- Transformations: rename fields, mask PII, enrich with lookup data in-flight

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Database change → downstream consumer in < 5 seconds (p99)
- **High Throughput**: 100K+ changes/sec from a single source database
- **Zero Impact on Source**: No performance degradation of the production database
- **Exactly-Once**: No duplicates even during connector restarts or Kafka broker failures
- **Durability**: No change event lost — every committed transaction must be captured
- **Scalability**: Support 1000+ source tables across 100+ databases
- **Schema Resilient**: Handle DDL changes (ALTER TABLE) without manual intervention
- **Ordering**: Changes to the same row are delivered in commit order

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Source databases | | 100 |
| Tables per database | | 50 |
| Total tables | 100 × 50 | 5,000 |
| Changes / sec (all sources) | | 500K |
| Avg change event size | | 500 bytes |
| Throughput | 500K × 500B | 250 MB/sec |
| Kafka retention (7 days) | 250 MB/s × 86400 × 7 | ~150 TB |
| Kafka topics (one per table) | | 5,000 |
| Debezium connectors | one per database | 100 |
| Kafka Connect workers | 100 connectors / 10 per worker | 10 |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         SOURCE DATABASES                                   │
│                                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ PostgreSQL   │  │ MySQL        │  │ MongoDB      │  │ SQL Server   │   │
│  │ (Primary)    │  │ (Primary)    │  │ (Primary)    │  │ (Primary)    │   │
│  │              │  │              │  │              │  │              │   │
│  │ WAL (Write   │  │ Binlog       │  │ Oplog        │  │ CT / CDC     │   │
│  │ Ahead Log)   │  │ (row-based)  │  │ (change      │  │ Tables       │   │
│  │              │  │              │  │  stream)     │  │              │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                  │                  │                  │          │
│    Logical                Binary             Change            Change      │
│    Replication             Log               Stream            Tracking    │
│    Slot                    Position           Resume            LSN         │
│         │                  │                  │                  │          │
└─────────│──────────────────│──────────────────│──────────────────│──────────┘
          │                  │                  │                  │
┌─────────▼──────────────────▼──────────────────▼──────────────────▼──────────┐
│                    CAPTURE LAYER (Debezium + Kafka Connect)                  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Kafka Connect Cluster (Distributed Mode)                              │  │
│  │                                                                        │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │  │
│  │  │ Worker 1       │  │ Worker 2       │  │ Worker 3       │           │  │
│  │  │                │  │                │  │                │           │  │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │           │  │
│  │  │ │Debezium PG │ │  │ │Debezium    │ │  │ │Debezium    │ │           │  │
│  │  │ │Connector-1 │ │  │ │MySQL       │ │  │ │MongoDB     │ │           │  │
│  │  │ │            │ │  │ │Connector-1 │ │  │ │Connector-1 │ │           │  │
│  │  │ │- Read WAL  │ │  │ │- Read      │ │  │ │- Read      │ │           │  │
│  │  │ │- Convert   │ │  │ │  binlog    │ │  │ │  oplog     │ │           │  │
│  │  │ │  to change │ │  │ │- Convert   │ │  │ │- Convert   │ │           │  │
│  │  │ │  events    │ │  │ │  to change │ │  │ │  to change │ │           │  │
│  │  │ │- Produce   │ │  │ │  events    │ │  │ │  events    │ │           │  │
│  │  │ │  to Kafka  │ │  │ │- Produce   │ │  │ │- Produce   │ │           │  │
│  │  │ │            │ │  │ │  to Kafka  │ │  │ │  to Kafka  │ │           │  │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │           │  │
│  │  │                │  │                │  │                │           │  │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │                │           │  │
│  │  │ │Debezium PG │ │  │ │ES Sink     │ │  │                │           │  │
│  │  │ │Connector-2 │ │  │ │Connector   │ │  │                │           │  │
│  │  │ └────────────┘ │  │ └────────────┘ │  │                │           │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘           │  │
│  │                                                                        │  │
│  │  Offset storage: Kafka internal topic (__connect_offsets)              │  │
│  │  Config storage: Kafka internal topic (__connect_configs)              │  │
│  │  Status storage: Kafka internal topic (__connect_status)               │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────┬───────────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────────┐
│                    STREAMING BACKBONE (Kafka)                                 │
│                                                                              │
│  Topic naming: {server_name}.{schema}.{table_name}                          │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ Kafka Cluster (RF=3, min.ISR=2)                                      │    │
���  │                                                                      │    │
│  │ Topic: pg-server1.public.orders        (partitioned by PK)          │    │
│  │ Topic: pg-server1.public.users         (partitioned by PK)          │    │
│  │ Topic: mysql-server2.ecommerce.products                             │    │
│  │ Topic: mongo-server3.analytics.events                               │    │
│  │ ...                                                                  │    │
│  │                                                                      │    │
│  │ Schema Registry (Confluent):                                         │    │
│  │ - Stores Avro/Protobuf schemas per topic                            │    │
│  │ - Backward compatibility check on schema change                     │    │
│  │ - Schema ID embedded in each message → compact serialization        │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────┬───────────────────────────────────────┘
                                       │
┌──────────────────────────────────────▼───────────────────────────────────────┐
│                    CONSUME & PROCESS LAYER                                    │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Flink Job    │  │ ES Sink      │  │ S3 Sink      │  │ Redis Cache  │     │
│  │ (Stream      │  │ Connector    │  │ Connector    │  │ Invalidator  │     │
│  │ Processing)  │  │              │  │              │  │              │     │
│  │              │  │ - Index into │  │ - Write      │  │ - On change  │     │
│  │ - Aggregate  │  │   ES for     │  │   Parquet    │  │   event →    │     │
│  │ - Enrich     │  │   search     │  │   to S3 for  │  │   DEL key    │     │
│  │ - Alert      │  │ - Upsert by  │  │   data lake  │  │   from Redis │     │
│  │ - Real-time  │  │   doc ID     │  │ - Partitioned│  │ - Cache-     │     │
│  │   analytics  │  │              │  │   by date    │  │   aside      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐                                          │
│  │ ClickHouse   │  │ Downstream   │                                          │
│  │ Materialized │  │ Microservice │                                          │
│  │ View         │  │ (consume &   │                                          │
│  │              │  │  react)      │                                          │
│  └──────────────┘  └──────────────┘                                          │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### How Log-Based CDC Works (PostgreSQL Example)

```
PostgreSQL Write-Ahead Log (WAL):
  Every transaction → WAL entry before data pages modified
  WAL is the source of truth for replication (streaming replication uses it)

Logical Replication (PostgreSQL 10+):
  1. Create PUBLICATION: CREATE PUBLICATION my_pub FOR TABLE orders, users;
  2. Debezium creates a REPLICATION SLOT:
     pg_create_logical_replication_slot('debezium_slot', 'pgoutput');
  3. Debezium reads decoded WAL changes via replication protocol
  4. Each change decoded into: {table, op, before, after, source_metadata}

Replication slot guarantees:
  PostgreSQL retains WAL segments until Debezium confirms consumption
  → No changes are ever lost (even if Debezium is down for hours)
  → BUT: if Debezium is down too long → WAL accumulates → disk fills up
  → Monitor: pg_replication_slots → confirmed_flush_lsn lag

Zero impact on production:
  Debezium reads the WAL (which is written anyway for crash recovery)
  No extra queries to the database
  No triggers, no polling
  CPU impact: < 1% (WAL decode is cheap)
```

#### Initial Snapshot + Streaming (The Bootstrap Problem)

```
Problem: When a CDC connector starts, the table already has 100M rows.
  You need: full initial snapshot + ongoing changes with NO GAPS.

Debezium Snapshot Flow:
  1. LOCK table (briefly) to get a consistent snapshot position
     PostgreSQL: BEGIN; SET TRANSACTION SNAPSHOT; → get LSN
     MySQL: FLUSH TABLES WITH READ LOCK → get binlog position
  2. Read entire table with SELECT * (chunked by PK for large tables)
     Each row emitted as a READ event (op='r')
  3. UNLOCK table (total lock time: < 1 second for most DBs)
  4. Switch to streaming mode: read WAL/binlog from snapshot position
  
  No gap between snapshot and streaming → every change captured exactly once

For very large tables (billions of rows):
  Incremental snapshot (Debezium 1.6+):
  - No global table lock
  - Reads table in chunks (WHERE id BETWEEN 1 AND 10000)
  - Interleaves snapshot reads with streaming reads
  - Deduplicates overlapping events by LSN comparison
  - Can take hours but never locks the table
```

#### Change Event Format (Debezium Envelope)

```json
{
  "schema": {...},  // Schema Registry reference
  "payload": {
    "before": {                          // Previous row state (null for INSERT)
      "id": 42,
      "status": "pending",
      "amount": 99.99,
      "updated_at": "2026-03-14T09:00:00Z"
    },
    "after": {                           // New row state (null for DELETE)
      "id": 42,
      "status": "completed",
      "amount": 99.99,
      "updated_at": "2026-03-14T10:00:00Z"
    },
    "source": {                          // Source metadata
      "version": "2.5.0",
      "connector": "postgresql",
      "name": "pg-server1",
      "db": "ecommerce",
      "schema": "public",
      "table": "orders",
      "txId": 12345678,
      "lsn": 987654321,
      "ts_ms": 1710403200000           // Transaction commit timestamp
    },
    "op": "u",                          // c=create, u=update, d=delete, r=read (snapshot)
    "ts_ms": 1710403200050             // Debezium processing timestamp
  }
}
```

---

## 5. APIs

### Kafka Connect REST API (Manage Connectors)

```
# Source connectors (Debezium)
POST   /connectors                    → Create new Debezium connector
GET    /connectors                    → List all connectors
GET    /connectors/{name}/status      → Connector + task status
PUT    /connectors/{name}/config      → Update connector config
POST   /connectors/{name}/restart     → Restart connector
DELETE /connectors/{name}             → Delete connector
PAUSE  /connectors/{name}/pause       → Pause (stop reading WAL)
RESUME /connectors/{name}/resume      → Resume from last position

# Example: Create PostgreSQL CDC connector
POST /connectors
{
  "name": "pg-ecommerce-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "pg-primary.internal",
    "database.port": "5432",
    "database.user": "debezium",
    "database.dbname": "ecommerce",
    "database.server.name": "pg-server1",
    "table.include.list": "public.orders,public.users,public.products",
    "column.exclude.list": "public.users.ssn,public.users.password_hash",
    "slot.name": "debezium_ecommerce",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
    "snapshot.mode": "initial",
    "transforms": "route",
    "transforms.route.type": "io.debezium.transforms.ByLogicalTableRouter",
    "transforms.route.topic.regex": "(.*)\\.public\\.(.*)",
    "transforms.route.topic.replacement": "cdc.$2"
  }
}
```

---

## 6. Data Models

### Kafka Topic Layout

```
Topic per table (recommended):
  pg-server1.public.orders     → partitioned by PK hash (order_id)
  pg-server1.public.users      → partitioned by PK hash (user_id)
  pg-server1.public.products   → partitioned by PK hash (product_id)

Key: serialized primary key (ensures same row → same partition → ordering)
Value: Debezium change event envelope (before, after, source, op)

Why partition by PK:
  All changes to order_id=42 go to same partition → strict ordering
  Consumer reading partition sees INSERT → UPDATE → UPDATE → DELETE in correct order
```

### Connect Offset Storage

```
Topic: __connect_offsets (25 partitions, compacted)
Key:   ["pg-ecommerce-connector", {"server": "pg-server1"}]
Value: {"lsn": 987654321, "txId": 12345678, "ts_sec": 1710403200}

On restart:
  Connector reads last stored offset → resumes WAL reading from that LSN
  No events missed, no events duplicated
```

### Schema Registry

```
Subject naming: {topic-name}-value
  pg-server1.public.orders-value → schema v1, v2, v3 ...

Compatibility modes:
  BACKWARD: new schema can read old data (add optional fields)
  FORWARD: old schema can read new data (remove optional fields)
  FULL: both directions (safest for CDC — ALTER TABLE ADD COLUMN is backward compatible)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Debezium crash** | Restart connector → reads last committed offset from __connect_offsets → resumes from WAL position |
| **Source DB failure** | Debezium reconnects automatically; WAL retained until consumed (replication slot) |
| **WAL accumulation** | Monitor slot lag; alert if > 1 hour; auto-drop abandoned slots |
| **Kafka broker failure** | RF=3, min.ISR=2; change events survive any single broker failure |
| **Schema change (DDL)** | Debezium detects DDL → updates schema in Schema Registry → new schema version → backward compatible |
| **Duplicate events** | Kafka key = PK → log compaction retains latest; consumers use upsert/idempotent writes |
| **Ordering violation** | Partition by PK → all changes for same row in same partition → strict order |
| **Connect worker failure** | Kafka Connect distributed mode → task rebalanced to surviving worker |

### Problem-Specific Deep Dives

#### Slot WAL Accumulation (The #1 Operational Risk)

```
Problem: Debezium is down for 6 hours → PostgreSQL retains 6 hours of WAL → disk fills up
  → PostgreSQL runs out of disk → PRODUCTION DATABASE DOWN

Mitigations:
  1. Monitor: pg_stat_replication_slots → confirmed_flush_lsn vs pg_current_wal_lsn()
     Alert if lag > 1 GB or 1 hour
  2. max_slot_wal_keep_size = 100GB (PG 13+) → auto-invalidate slot before disk fills
     Debezium detects invalidation → triggers new snapshot automatically
  3. Heartbeat table: Debezium writes to a heartbeat table every 30s
     → Forces WAL position to advance even if no user data changes
     → Prevents "stuck slot" problem on quiet tables
  4. PagerDuty alert if connector status != RUNNING for > 5 minutes
```

#### Schema Evolution Without Downtime

```
Scenario: DBA runs ALTER TABLE orders ADD COLUMN discount DECIMAL(10,2);

What happens:
  1. Debezium reads DDL from WAL
  2. Debezium generates new schema (v2) with discount field added
  3. Schema Registry: compatibility check (BACKWARD) → v2 is compatible → registered
  4. New events have discount field; old events do not
  5. Consumers using schema v2 → new field is null for old events
  6. No pipeline restart needed

Problematic DDL (breaks compatibility):
  ALTER TABLE orders DROP COLUMN status;  → BACKWARD incompatible
  Schema Registry REJECTS schema v2 → Debezium pauses with error
  Fix: use FORWARD or NONE compatibility, or do two-phase migration:
    Phase 1: Stop consumers from using the field
    Phase 2: Drop column → register schema → consumers already adapted
```

#### Exactly-Once End-to-End

```
Source → Kafka: Debezium + Kafka acks=all → at-least-once (idempotent via PK key)
Kafka → Sink:
  Elasticsearch: upsert by doc_id = PK → naturally idempotent
  PostgreSQL: INSERT ON CONFLICT (pk) DO UPDATE → idempotent
  S3: Flink exactly-once file sink (two-phase commit)
  Redis: SET key value → idempotent (last write wins)

Net effect: exactly-once semantics without distributed transactions
  The pattern: at-least-once delivery + idempotent consumers = exactly-once
```

---

## 8. Additional Considerations

### CDC vs Dual Writes vs Polling

```
| Approach | How | Pros | Cons |
|---|---|---|---|
| CDC (log-based) ⭐ | Read database WAL | Zero impact, complete, ordered, real-time | Operational complexity (slots, Debezium) |
| Dual writes | App writes to DB + Kafka | Simple | Race condition: DB write succeeds, Kafka fails → inconsistency |
| Outbox pattern | App writes to outbox table → CDC on outbox | Transactional consistency | Extra table, need to manage outbox cleanup |
| Polling (query-based) | SELECT WHERE updated_at > last_poll | Simple, no infrastructure | Misses deletes, not real-time, adds DB load |

CDC wins for:
  - Real-time data sync (< 5 seconds)
  - Search index updates (DB → Elasticsearch)
  - Cache invalidation (DB → Redis DELETE)
  - Analytics pipeline (DB → data warehouse)
  - Event-driven microservices (DB → Kafka → services)
  - CQRS read model updates
```

### Outbox Pattern (When Pure CDC Isn't Enough)

```
Problem: Application needs to emit a DOMAIN EVENT (not just a row change)
  CDC captures: "order.status changed from 'pending' to 'shipped'"
  But you need: "OrderShipped event with tracking_number, carrier, estimated_delivery"

Solution: Outbox Pattern
  1. In SAME database transaction:
     UPDATE orders SET status='shipped' WHERE id=42;
     INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
       VALUES ('order', '42', 'OrderShipped', '{"tracking":"1Z999"}');
  2. Debezium CDC on outbox table → rich domain events to Kafka
  3. Background cleaner: DELETE FROM outbox WHERE created_at < NOW() - INTERVAL '1 hour';

  Transactional guarantee: event is produced IF AND ONLY IF the business action committed
  This solves the dual-write consistency problem
```

### Multi-Region CDC

```
Primary DB (us-east) → Debezium (us-east) → Kafka (us-east) → MirrorMaker → Kafka (eu-west)
                                                                                    ↓
                                                              EU consumers (Elasticsearch, cache, etc.)

Latency: DB change → EU consumer = ~200ms (WAL read + Kafka + cross-region)
```

### Why This Problem Matters for Senior/Staff Engineers

```
CDC is THE pattern that enables:
  1. Microservice decomposition (shared DB → event-driven)
  2. CQRS (write model → read model sync)
  3. Real-time analytics (OLTP → OLAP without ETL)
  4. Search index freshness (DB → Elasticsearch in seconds)
  5. Cache coherence (DB → Redis invalidation)
  6. Data mesh (domain teams own their data as events)

Every senior engineer should be able to explain:
  - Why log-based CDC is superior to polling and dual writes
  - How replication slots work and their operational risks
  - The outbox pattern and when to use it
  - Schema evolution in a CDC pipeline
  - Exactly-once via idempotent consumers (not distributed transactions)
```

