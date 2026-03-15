# SQL Scaling — Sharding, Partitioning, Read Replicas & Beyond

Core concept tested in HLD rounds: "Your single PostgreSQL can handle 10K writes/sec. Your system needs 100K. How do you scale?"

---

## 1. Vertical Scaling (Scale Up) — The Starting Point

```
Before distributing data, try scaling the single machine:

CPU:    16 cores → 64 cores
RAM:    64 GB → 512 GB (entire working set in memory)
Disk:   HDD → NVMe SSD (100× IOPS improvement)
Network: 1 Gbps → 25 Gbps

PostgreSQL on a beefy machine (r6g.16xlarge — 512 GB RAM, 64 vCPUs):
  Handles: ~50K simple queries/sec, ~10K complex queries/sec
  Stores: ~2 TB comfortably with good indexing

When this is enough: most startups, up to ~10M users
When this fails: write throughput exceeds single-machine limits,
  data exceeds single-machine storage, or HA requires multi-node
```

---

## 2. Read Replicas — Scale Reads

```
                      ┌─────────────┐
  Writes ────────────►│   Primary   │
                      └──────┬──────┘
                             │ async replication
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │Replica 1│   │Replica 2│   │Replica 3│
        └─────────┘   └─────────┘   └─────────┘
              ▲              ▲              ▲
              └──────────────┼──────────────┘
  Reads ─────────────────────┘
```

- **All writes → primary**. Reads distributed across replicas.
- **Scales reads linearly**: 5 replicas = 5× read throughput
- **Does NOT scale writes** (still bottlenecked on primary)

**Connection Routing**:
```
Application-level routing:
  write_db = connect("primary:5432")
  read_db = connect("replica:5432")  # or use a read-only endpoint

  def get_user(user_id):
      return read_db.query("SELECT * FROM users WHERE id = ?", user_id)
  
  def update_user(user_id, name):
      write_db.query("UPDATE users SET name = ? WHERE id = ?", name, user_id)

Proxy-level routing (PgBouncer, ProxySQL):
  Proxy intercepts queries, routes SELECTs to replicas, writes to primary
  ✓ Transparent to application
  ✗ Must correctly identify read-only queries (can misroute)

Cloud managed:
  AWS RDS: reader endpoint automatically load-balances across replicas
  Aurora: single cluster endpoint with read/write splitting
```

**Replication Lag Handling**:
```
Problem: User updates profile → reads from replica → sees OLD data

Solution 1: Read-after-write routing
  After a write, route that user's reads to primary for N seconds

Solution 2: RETURNING clause
  UPDATE users SET name = 'Bob' WHERE id = 42 RETURNING *;
  → Application already has the updated row, no need to re-read

Solution 3: Version tracking
  On write: record WAL LSN. On read: check replica has reached that LSN.
```

---

## 3. Connection Pooling — The Hidden Bottleneck

```
Problem: PostgreSQL creates a new OS PROCESS per connection
  Each process: ~10 MB RAM overhead
  Max connections: typically 200-500 (kernel/memory limited)
  
  100 app servers × 20 connections each = 2,000 connections
  But PostgreSQL max_connections = 500 → connection refused!

Solution: Connection Pooler (PgBouncer / PgPool)

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ App Svr 1│  │ App Svr 2│  │ App Svr 3│
  │ 20 conns │  │ 20 conns │  │ 20 conns │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       └──────────────┼──────────────┘
                      │ 60 connections
                ┌─────▼──────┐
                │  PgBouncer │  ← Multiplexes 60 app conns into 20 DB conns
                └─────┬──────┘
                      │ 20 connections
                ┌─────▼──────┐
                │ PostgreSQL │
                └────────────┘

Modes:
  Session pooling: 1 app connection → 1 DB connection for session lifetime
    (no saving, but preserves session state)
  
  Transaction pooling ⭐: DB connection shared between app connections
    Borrow on BEGIN, return on COMMIT
    ✓ Massive savings: 2000 app conns → 50 DB conns
    ✗ Can't use session-level features (LISTEN/NOTIFY, prepared statements)
  
  Statement pooling: DB connection shared per statement
    ✓ Maximum multiplexing
    ✗ Breaks transactions (each statement may run on different connection)
```

---

## 4. Table Partitioning — Scale Within a Single Database

### Horizontal Partitioning (Range / List / Hash)

```
Split a large table into smaller physical tables (partitions) — same schema.
PostgreSQL handles routing transparently.

CREATE TABLE orders (
    order_id    BIGSERIAL,
    user_id     INT,
    created_at  TIMESTAMP,
    total       DECIMAL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... one partition per month

Query: SELECT * FROM orders WHERE created_at = '2026-02-15'
  → PostgreSQL automatically routes to orders_2026_02 (partition pruning)
  → Only scans 1 partition, not the full table

Benefits:
  ✓ Queries touching one partition are fast (smaller index, less data)
  ✓ DROP old partitions instantly (instead of DELETE with vacuum)
  ✓ Each partition can be on a different tablespace (hot/cold storage)
  ✓ Parallel query across partitions
  
Limitations:
  ✗ Still single machine (doesn't scale writes to multiple machines)
  ✗ Cross-partition queries may be slower (scan all partitions)
  ✗ Foreign keys to partitioned tables are limited
```

**Partition Strategies**:
```
Range (by time) ⭐:
  Best for: Time-series data, logs, orders
  Partition key: created_at (monthly/weekly/daily)
  Benefit: Old data → drop partition (instant purge)

List (by category):
  Best for: Multi-tenant, geo-specific data
  Partition key: country, tenant_id
  Benefit: Per-tenant partition → easy tenant isolation

Hash (by ID):
  Best for: Uniform distribution without natural range
  Partition key: user_id, order_id
  Benefit: Even distribution across N partitions
  
  CREATE TABLE users PARTITION BY HASH (user_id);
  CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
  CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
  ...
```

---

## 5. Sharding — Scale Writes Across Machines

### What Is Sharding?

```
Partitioning across MULTIPLE database servers.
Each server (shard) holds a subset of the data.

  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │
  │ users    │  │ users    │  │ users    │  │ users    │
  │ 0-25M    │  │ 25M-50M  │  │ 50M-75M  │  │ 75M-100M │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘

Each shard is an independent PostgreSQL/MySQL instance with replicas.
Application or proxy routes queries to the correct shard.
```

### Shard Key Selection — The Most Critical Decision

```
The shard key determines which shard each row lives on.
  shard_id = hash(shard_key) % num_shards
  
GOOD shard keys:
  user_id   → user's data is co-located → most queries hit 1 shard
  tenant_id → multi-tenant SaaS → full tenant isolation
  order_id  → if orders are independent (but JOIN with user requires cross-shard)

BAD shard keys:
  created_at → recent data piles up on one shard (hot shard)
  country    → uneven distribution (US shard 10× bigger than Iceland)
  email      → no relation to access patterns

The rule: Shard key = the entity that MOST queries filter by
  If 90% of queries are "get user's data" → shard by user_id
  If 90% of queries are "get tenant's data" → shard by tenant_id
```

### Cross-Shard Operations — The Pain of Sharding

```
Problem 1: JOINs across shards
  SELECT * FROM orders JOIN users ON orders.user_id = users.id
  If orders and users are on different shards → can't do SQL JOIN
  
  Solutions:
    a. Shard both tables by same key (user_id) → co-located JOIN
    b. Denormalize: store user_name in orders table (duplicate data)
    c. Application-level join: fetch from both shards, merge in code

Problem 2: Aggregations across all shards
  SELECT COUNT(*) FROM orders WHERE status = 'completed'
  Must query ALL shards and sum results (scatter-gather)
  
  Solution: Maintain aggregates in a separate analytics store (ClickHouse)

Problem 3: Unique constraints across shards
  UNIQUE(email) across all users → can't enforce per-shard
  
  Solutions:
    a. Global lookup table: email → shard_id (check before insert)
    b. Assign email shard by hash(email) → uniqueness check within shard
    c. Use a distributed unique index service

Problem 4: Transactions across shards
  "Transfer $100 from user A (shard 1) to user B (shard 3)"
  Can't use a single database transaction
  
  Solutions:
    a. Two-Phase Commit (2PC): Coordinator asks both shards to prepare → commit
       ✗ Slow, blocks on any failure, rarely used
    b. Saga pattern: Debit A → Credit B. If Credit fails → compensate (re-credit A)
       ✓ Eventually consistent, no global locks
    c. Avoid cross-shard transactions by design (co-locate related data)

Problem 5: Resharding (adding/removing shards)
  Adding shard 4 to a 3-shard cluster:
    hash(key) % 3 ≠ hash(key) % 4 → most keys need to move!
  
  Solutions:
    a. Consistent hashing: only ~1/N of keys move
    b. Logical sharding: 1024 logical shards mapped to N physical shards
       Adding a physical shard: move some logical shards (no rehashing)
    c. Vitess: handles resharding transparently (split/merge shards)
```

---

## 6. Sharding Solutions

### Application-Level Sharding (DIY)

```
Application contains shard routing logic:

def get_shard(user_id):
    shard_id = user_id % NUM_SHARDS
    return shard_connections[shard_id]

def get_user(user_id):
    shard = get_shard(user_id)
    return shard.query("SELECT * FROM users WHERE id = ?", user_id)

✓ Full control
✗ Every query must include shard routing
✗ Schema migrations must run on ALL shards
✗ Resharding is manual and risky
```

### Vitess (MySQL Sharding Layer) ⭐

```
┌──────────┐
│   App    │ ← standard MySQL protocol
└────┬─────┘
     │
┌────▼─────┐
│  VTGate  │ ← query router (understands shard topology)
└────┬─────┘
     │
┌────┼─────────────┐
│    │              │
▼    ▼              ▼
┌────────┐  ┌────────┐  ┌────────┐
│VTTablet│  │VTTablet│  │VTTablet│ ← each manages one MySQL shard
│(Shard 0)│  │(Shard 1)│  │(Shard 2)│
└────────┘  └────────┘  └────────┘

Features:
  • Transparent sharding (app sees one "database")
  • Online resharding (split/merge shards with zero downtime)
  • Connection pooling built-in
  • Schema migrations across all shards
  • Query routing and scatter-gather for cross-shard queries

Used by: YouTube, Slack, GitHub, Pinterest, Square
```

### Citus (PostgreSQL Sharding Extension)

```
Citus extends PostgreSQL with distributed table capabilities:

SELECT create_distributed_table('orders', 'user_id');
  → Shards orders table across worker nodes by user_id

-- This query automatically routes to correct shard:
SELECT * FROM orders WHERE user_id = 42;

-- This scatter-gathers across all shards:
SELECT status, COUNT(*) FROM orders GROUP BY status;

Features:
  • Transparent to PostgreSQL clients (standard SQL)
  • Co-location: tables sharded by same key → local JOINs
  • Reference tables: small tables replicated to all shards (e.g., countries)
  • Real-time analytics: parallel query execution across shards

Used by: Microsoft (Azure Cosmos DB for PostgreSQL), Algolia
```

### CockroachDB / YugabyteDB (Distributed SQL — NewSQL)

```
Fully distributed SQL database — sharding is built into the database itself.
No external sharding layer needed.

CockroachDB:
  • Automatic range-based sharding (data split into ~64 MB ranges)
  • Raft consensus per range (strong consistency)
  • Distributed transactions (serializable isolation)
  • Multi-region with automatic replica placement
  • PostgreSQL wire-compatible

  ✓ "Just works" — no manual sharding logic
  ✓ Strong consistency + distributed transactions
  ✗ Higher write latency (consensus overhead, ~5-10ms vs ~1ms for single-node PG)
  ✗ Less mature ecosystem than PostgreSQL/MySQL

When to use:
  • Global apps needing multi-region strong consistency
  • When you want SQL semantics without manual sharding
  • When data exceeds single-node and you can tolerate slightly higher latency
```

---

## 7. Query Optimization — Before You Scale

```
Often the bottleneck is bad queries, not insufficient hardware.

1. EXPLAIN ANALYZE every slow query:
   EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
   SELECT * FROM orders WHERE user_id = 42 AND created_at > '2026-01-01';
   
   Look for:
     Seq Scan → missing index (should be Index Scan or Index Only Scan)
     Nested Loop on large tables → consider hash join or add indexes
     Sort → consider index with matching order
     Rows estimated vs actual → stale statistics → ANALYZE table

2. Index strategy:
   CREATE INDEX idx_orders_user_date ON orders (user_id, created_at DESC);
   
   Composite index order matters:
     WHERE user_id = ? AND created_at > ? → (user_id, created_at) ✓
     WHERE created_at > ? AND user_id = ? → same index works ✓
     WHERE created_at > ? → (user_id, created_at) ✗ (can't skip first column)
   
   Covering index (includes all selected columns):
     CREATE INDEX idx_orders_cover ON orders (user_id) INCLUDE (total, status);
     → Index-only scan (no table lookup needed)

3. Pagination — offset vs cursor:
   OFFSET 10000 LIMIT 20  ← scans and discards 10,000 rows! O(offset)
   
   Cursor-based (keyset) pagination ⭐:
   SELECT * FROM orders 
   WHERE user_id = 42 AND order_id < {last_seen_order_id}
   ORDER BY order_id DESC LIMIT 20;
   → Always O(limit) regardless of page depth

4. Denormalization for read performance:
   Instead of JOIN on every read:
     Store user_name directly in orders table
     Update on user name change (rare) via CDC/trigger
     
   ✓ Eliminates JOIN on hot read path
   ✗ Data duplication, eventual consistency on updates

5. Materialized Views for complex aggregations:
   CREATE MATERIALIZED VIEW daily_revenue AS
     SELECT DATE(created_at), SUM(total) FROM orders GROUP BY 1;
   
   REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
   → Pre-computed, refreshed periodically or on schedule
```

---

## 8. Failover & High Availability

### Quorum Commits (Synchronous Replication)

```
For critical data (payments, balances): don't lose a single transaction

PostgreSQL:
  synchronous_standby_names = 'ANY 1 (replica1, replica2, replica3)'
  → Primary waits for AT LEAST 1 replica to confirm WAL receipt before ACKing
  
  "ANY 2 (...)" → wait for 2 replicas (stronger safety, higher latency)

MySQL Group Replication (single-primary mode):
  Paxos-based consensus — transaction committed only when majority ACKs
  Automatic failover on primary failure

Trade-off:
  Quorum commits add ~1-5ms latency to writes (network round-trip to replica)
  But guarantee zero data loss on primary failure
```

### Patroni (PostgreSQL HA) ⭐

```
┌──────────┐
│  Patroni │ ← HA controller per node
│  + PG    │
└────┬─────┘
     │ register leadership
┌────▼─────┐
│  etcd /  │ ← distributed consensus store (leadership lock)
│  ZooKeeper│
└──────────┘

Patroni manages:
  • Leader election via etcd/ZooKeeper distributed lock
  • Automatic failover (promote most up-to-date replica)
  • Replica configuration (add/remove replicas)
  • Switchover (planned failover with zero data loss)
  • Fencing: old primary demoted to replica (prevents split-brain)

Production setup:
  3 PostgreSQL nodes (1 primary + 2 replicas)
  3 etcd nodes (consensus quorum)
  HAProxy or PgBouncer for connection routing
  Failover time: 10-30 seconds
```

---

## 9. Scaling Decision Tree

```
Step 1: Is the bottleneck reads or writes?
  Reads → Add read replicas (easiest)
  Writes → Continue to step 2

Step 2: Can you optimize queries/indexes?
  Often 10× improvement from proper indexing and query rewrite
  Run EXPLAIN ANALYZE on top 20 slowest queries

Step 3: Can you vertically scale?
  Bigger instance: more CPU, RAM, NVMe
  Handles up to ~50K queries/sec for simple queries

Step 4: Can you partition within the same DB?
  Time-based partitioning for time-series data
  Reduces per-query scan size significantly

Step 5: Do you need to shard (distribute writes)?
  < 100M rows → probably don't need sharding
  > 1B rows or > 50K writes/sec → consider sharding
  
  Options:
    Vitess (MySQL) → proven at YouTube/Slack scale
    Citus (PostgreSQL) → extension, PostgreSQL-compatible
    CockroachDB → built-in distribution, strong consistency
    Application-level → full control, most effort

Step 6: Do you actually need SQL?
  If access pattern is simple key-value → DynamoDB/Cassandra scales horizontally
  If access pattern needs JOINs/transactions → stick with SQL + sharding
  Hybrid: SQL for transactional data, NoSQL for high-throughput reads
```

