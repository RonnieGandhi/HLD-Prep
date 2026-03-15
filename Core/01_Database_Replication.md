# Database Replication — SQL & NoSQL Deep Dive

Core concept tested in HLD rounds: "How does your database stay consistent across replicas? What happens when a replica falls behind? What happens during a network partition?"

---

## 1. Why Replicate?

| Goal | How Replication Helps |
|---|---|
| **High Availability** | If primary dies, a replica takes over (failover) |
| **Read Scalability** | Distribute reads across replicas (read replicas) |
| **Disaster Recovery** | Replicas in another region survive full DC failure |
| **Low Latency** | Serve reads from the geographically closest replica |

---

## 2. Replication Topologies

### Single-Leader (Primary-Replica)

```
Writes ──► [Primary] ──replicate──► [Replica 1]
                      ──replicate──► [Replica 2]
                      ──replicate──► [Replica 3]

Reads ◄── [Primary] or [Replica 1] or [Replica 2] or [Replica 3]
```

- All writes go to ONE primary
- Replicas receive a replication stream (WAL-based or logical)
- **Used by**: MySQL, PostgreSQL, MongoDB (replica sets), Redis (master-replica)
- **Pros**: Simple consistency model (single source of truth), no write conflicts
- **Cons**: Write bottleneck on primary, failover complexity

### Multi-Leader (Master-Master)

```
[Leader A] ◄──replicate──► [Leader B]
     │                          │
Writes from             Writes from
Region US               Region EU
```

- Multiple nodes accept writes (typically one per region)
- Each leader replicates its writes to other leaders
- **Used by**: MySQL Galera Cluster, CockroachDB, Cassandra (all nodes are leaders), DynamoDB Global Tables
- **Pros**: Local writes everywhere (low latency), no single write bottleneck
- **Cons**: Write conflicts are INEVITABLE — need conflict resolution strategy

**Conflict Resolution Strategies**:
```
Scenario: User updates profile name on US leader AND EU leader simultaneously

1. Last-Write-Wins (LWW):
   Compare timestamps → latest wins → earlier is silently lost
   ✓ Simple, no application changes
   ✗ Data loss (silent!) — unacceptable for critical data
   Used by: Cassandra, DynamoDB

2. Custom Merge Function (Application-Level):
   Both values returned to application → app decides
   Example: Shopping cart → union of items (no loss)
   Example: Counter → sum the increments from both sides
   ✓ No data loss
   ✗ Complex, application must handle merges
   Used by: Amazon Dynamo (vector clocks + app merge)

3. CRDTs (Conflict-free Replicated Data Types):
   Data structures mathematically guaranteed to converge
   Examples: G-Counter, PN-Counter, OR-Set, LWW-Register
   ✓ Automatic convergence, no coordination needed
   ✗ Limited to specific data types, memory overhead
   Used by: Redis CRDT module, Riak

4. Operational Transform / Event Sourcing:
   Don't merge state — merge operations
   "Add item X" and "Add item Y" → apply both → no conflict
   ✓ Rich conflict resolution
   ✗ Complex implementation
   Used by: Google Docs (OT), Event-sourced systems
```

### Leaderless (Peer-to-Peer)

```
Client ──write──► [Node A] + [Node B] + [Node C]
       ──read───► [Node A] + [Node B] + [Node C]
                  (quorum: W + R > N)
```

- No designated leader — any node accepts reads AND writes
- Client writes to W nodes, reads from R nodes, requires W + R > N for consistency
- **Used by**: Cassandra, DynamoDB, Riak, Voldemort
- **Pros**: No failover needed (no leader to fail), always writable (AP)
- **Cons**: Quorum overhead, conflict resolution complexity, no ordering guarantees

---

## 3. Sync vs Async vs Semi-Sync Replication

### Synchronous Replication

```
Client ──write──► Primary ──replicate──► Replica (waits for ACK)
                                                    │
                                              ACK back to Primary
                                                    │
                                         Primary ACKs to Client
```

- Primary waits for ALL replicas to confirm before ACKing to client
- **Guarantee**: If primary ACKs, data is on ALL replicas (zero data loss)
- **Trade-off**: High write latency (limited by slowest replica), reduced availability (any replica down = writes blocked)
- **When to use**: Financial systems, payment ledgers — where data loss is unacceptable
- **Examples**: PostgreSQL synchronous_commit = on, MySQL Group Replication (single-primary mode)

### Asynchronous Replication

```
Client ──write──► Primary ──ACK to client (immediately)
                      │
                      └──replicate──► Replica (eventually)
```

- Primary ACKs to client immediately after local write
- Replication happens in background
- **Guarantee**: None — if primary crashes before replicating, that write is LOST
- **Trade-off**: Low write latency, high availability, but potential data loss
- **Replication lag**: Replicas may be seconds or minutes behind (especially under load)
- **When to use**: Most systems where occasional data loss of last few seconds is acceptable
- **Examples**: MySQL default, PostgreSQL default, MongoDB default, Redis

### Semi-Synchronous Replication

```
Client ──write──► Primary ──replicate──► Replica 1 (waits for ACK from AT LEAST ONE)
                                    └──► Replica 2 (async, don't wait)
                                    └──► Replica 3 (async, don't wait)
                                              │
                                     Replica 1 ACKs ← Primary ACKs to Client
```

- Primary waits for at least ONE replica to confirm before ACKing
- Other replicas receive data asynchronously
- **Guarantee**: Data survives primary failure if at least one replica got it
- **Trade-off**: Middle ground — slightly higher latency than async, much better durability
- **When to use**: Most production systems (good balance of safety and performance)
- **Examples**: MySQL semi-sync replication (`rpl_semi_sync_master_wait_for_slave_count = 1`)

### Comparison

| Property | Sync | Semi-Sync | Async |
|---|---|---|---|
| Write latency | High (wait for ALL) | Medium (wait for ONE) | Low (no wait) |
| Data loss on primary crash | Zero | Near-zero (1 replica has it) | Possible (up to lag) |
| Availability | Low (any replica down blocks) | Good (need ≥1 replica up) | High (no dependency) |
| Replication lag | Zero | Near-zero on 1 replica | Seconds to minutes |
| Use case | Financial ledgers | Most prod systems | Analytics replicas |

---

## 4. Replication Mechanisms

### Physical (WAL-Based) Replication

```
Primary writes WAL (Write-Ahead Log) entry:
  LSN: 0x3A8F | Table: users | Op: UPDATE | Row: id=42 | old: {name: "Alice"} | new: {name: "Bob"}

Primary streams WAL bytes to replicas
Replicas apply WAL entries to their local storage
```

- Replicate the raw WAL (byte-level changes)
- **Pros**: Exact byte-for-byte copy, supports all operations (DDL, functions, etc.)
- **Cons**: Replicas must be SAME database version, SAME OS, SAME architecture. Can't replicate to different DB engines.
- **Used by**: PostgreSQL (streaming replication), MySQL (binary log with ROW format)

### Logical Replication

```
Primary writes logical change:
  INSERT INTO users (id, name) VALUES (42, 'Bob')
  -- or --
  {table: "users", op: "INSERT", data: {id: 42, name: "Bob"}}

Logical decoder converts WAL → logical changes
Stream logical changes to replicas
```

- Replicate logical changes (row-level insert/update/delete events)
- **Pros**: Cross-version replication, cross-engine replication, selective table replication, can transform data in transit
- **Cons**: More overhead (parsing), DDL not always supported, doesn't capture sequences/large objects automatically
- **Used by**: PostgreSQL logical replication, MySQL binlog (ROW format), Debezium (CDC)

### Statement-Based Replication (Legacy — Mostly Deprecated)

```
Primary executes: UPDATE users SET last_login = NOW() WHERE id = 42
Replica receives and re-executes the same SQL statement

Problem: NOW() returns different time on replica (clock skew) → data divergence!
```

- **Mostly deprecated**: MySQL moved to ROW-based binlog, PostgreSQL never used statement-based

---

## 5. Replication Lag — The Fundamental Challenge

### What Is Replication Lag?

```
Timeline:
  T=0:     Primary writes row X
  T=0.001: Primary ACKs to client
  T=0.5:   Replica receives row X (lag = 500ms)

  If client reads from Replica at T=0.3: row X is NOT there yet!
  → Stale read (read-your-writes violation)
```

### Read-Your-Writes Consistency

**Problem**: User writes data, then reads it back from a replica that hasn't caught up yet → user thinks their write was lost.

**Solutions**:
```
1. Route reads-after-writes to Primary:
   After writing, set a cookie: "read-from-primary-for-next-10s"
   API layer checks cookie → routes to primary if set
   ✓ Simple
   ✗ Doesn't work cross-device

2. Track replication position (LSN/GTID):
   On write: record the WAL position (LSN) that the write produced
   On read: pass the LSN → replica checks if it's caught up to that LSN
   If not caught up → wait (bounded) or redirect to primary
   ✓ Precise, works across devices if LSN is stored per user
   Used by: Aurora (cluster endpoint tracks LSN per connection)

3. Causal consistency with logical timestamps:
   Each write gets a logical timestamp (Hybrid Logical Clock)
   Client tracks "last-write-timestamp"
   Read request includes "at-least-timestamp" → replica waits until caught up
   ✓ Works across services
   Used by: MongoDB (causal sessions), CockroachDB

4. Optimistic UI:
   After write, show optimistic update locally (don't re-read from server)
   Eventually consistent read catches up
   ✓ Best UX, simplest
   ✗ Doesn't solve programmatic reads (API-to-API)
```

### Monotonic Read Consistency

**Problem**: User reads from Replica A (up to date), then Replica B (behind) → sees data go BACKWARD in time.

**Solution**: Sticky reads — hash user_id → always same replica. If that replica fails → fall back to primary.

---

## 6. Failover — What Happens When Primary Dies

### Automated Failover Steps

```
1. DETECT failure:
   Health checks fail for N consecutive checks (e.g., 3 × 5s = 15s)
   → Declare primary DEAD

2. ELECT new primary:
   Choose the replica with the LEAST replication lag (most up-to-date)
   
   PostgreSQL: Patroni/Stolon orchestrator
   MySQL: Orchestrator / MHA picks the most advanced replica
   MongoDB: Built-in Raft-like election (majority vote)

3. PROMOTE replica:
   New primary starts accepting writes
   Other replicas reconfigure to replicate from new primary

4. UPDATE routing:
   DNS/VIP failover — connection pools reconnect to new primary
   
5. RECONFIGURE old primary (when it comes back):
   Rejoin as a replica → replicate from new primary
   DANGER: If old primary had un-replicated writes → those writes are LOST
```

### Split-Brain Prevention

```
Split-Brain: Network partition → BOTH old and new primary accept writes

Prevention:
1. Fencing (STONITH):
   Before promoting, FORCE-STOP the old primary (IPMI reset, cloud API force-stop)
   
2. Lease-based leadership:
   Primary holds a 30s lease in ZooKeeper/etcd
   If can't renew (partitioned) → stop accepting writes
   New primary elected only after lease expires
   
3. Quorum-based (odd number of nodes):
   Only the partition with MAJORITY can elect a primary
   Minority partition steps down
   Used by: MongoDB, etcd, CockroachDB
```

---

## 7. SQL-Specific Replication

### PostgreSQL Streaming Replication

```
Primary:
  wal_level = replica
  max_wal_senders = 10
  synchronous_standby_names = ''  # '' = async, 'replica1' = sync

Replication Slot:
  Primary retains WAL segments until replica confirms receipt
  Prevents replica from falling irretrievably behind
  DANGER: If replica goes down for days, WAL accumulates → disk full on primary
  → Set max_slot_wal_keep_size to cap retained WAL
```

### MySQL Binary Log Replication

```
GTID (Global Transaction ID):
  Each transaction: server_uuid:sequence_number
  Example: 3E11FA47-71CA:42
  
  Replica tracks: "I've applied all GTIDs up to X"
  On failover: new primary says "I have all GTIDs up to Y"
  Replica connects: "Send me everything after GTID X"
  → No positional confusion (unlike old-style binlog file:position)
```

---

## 8. NoSQL-Specific Replication

### Cassandra (Leaderless, Tunable Consistency)

```
Replication Factor = 3 (RF=3): Each key stored on 3 nodes

Write: W=QUORUM (2 of 3 must ACK)
Read:  R=QUORUM (2 of 3 must respond) → read repair if responses differ

Strong consistency: W + R > N → guaranteed overlap
  W=2, R=2, N=3: 2+2=4 > 3 ✓
  W=1, R=1, N=3: 1+1=2 < 3 ✗ (may read stale)
```

### MongoDB (Single-Leader, Raft-like Election)

```
Replica Set (3 members, odd for majority):
  Primary: accepts writes
  Secondary 1, 2: replicate from primary (async default)

  w: "majority" → ACK after majority confirms (safe)
  Read preference: primary | secondary | nearest
  Failover: 5-15 seconds (automatic election)
```

### DynamoDB (Global Tables — Multi-Region Multi-Master)

```
Full multi-master: writes accepted in ANY region
Conflict resolution: Last-Writer-Wins (wall clock timestamp)
Replication lag: < 1 second cross-region
Caveat: LWW → concurrent writes to same item → one silently lost
```

---

## 9. Change Data Capture (CDC) — Replication Beyond the Database

```
CDC captures every row-level change from the database's transaction log
and streams it to external consumers.

Primary DB ──CDC──► Kafka ──► Elasticsearch (search index)
                          ──► Redis (cache invalidation)
                          ──► Data Warehouse (analytics)
                          ──► Another DB (cross-system sync)

Best tool: Debezium (reads WAL/binlog, emits to Kafka)
  ✓ No impact on DB performance (reads existing log)
  ✓ Captures ALL changes including direct SQL
  ✓ Preserves order and transaction boundaries
```

### Transactional Outbox Pattern

```
Problem: Write to DB + publish event to Kafka must be atomic.
  If DB write succeeds but Kafka fails → inconsistency.

Solution: Write event to "outbox" table in SAME DB transaction.
  Debezium reads outbox → publishes to Kafka → deletes from outbox.

  BEGIN TRANSACTION;
    INSERT INTO orders (...);
    INSERT INTO outbox (event_type, payload) VALUES ('order_created', '{...}');
  COMMIT;

  ✓ Atomic: data change + event always consistent
  ✓ No distributed transaction needed
```

---

## 10. Decision Framework

```
Q1: Can I tolerate data loss on primary failure?
  YES → Async replication
  NO  → Semi-sync or sync

Q2: Do I need writes from multiple regions?
  YES → Multi-leader (Cassandra, DynamoDB Global Tables, CockroachDB)
  NO  → Single-leader (simpler, no conflicts)

Q3: How do I handle conflicts in multi-leader?
  Shopping cart → CRDTs or merge function (union)
  User profile → LWW (latest wins)
  Financial balance → single-leader only (no conflicts allowed)

Q4: What read consistency do I need?
  "Latest data" → Read from primary or quorum reads
  "OK if stale" → Read from any replica
  "Read my writes" → Sticky sessions or LSN tracking

Q5: How many replicas?
  3 minimum (survive 1 failure)
  5 for critical (survive 2 failures)
  Cross-region for DR (adds 50-200ms latency)
```

