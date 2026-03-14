# 31. Design a Distributed Queue (like RabbitMQ / SQS)

---

## 1. Functional Requirements (FR)

- **Produce messages**: Publishers send messages to named queues/topics
- **Consume messages**: Consumers pull or receive pushed messages from queues
- **At-least-once delivery**: Every message delivered at least once (duplicates possible, not losses)
- **Ordering**: FIFO ordering within a partition/queue
- **Acknowledgment**: Consumers explicitly ACK messages; unACKed messages are redelivered
- **Dead Letter Queue (DLQ)**: Messages that fail N times are moved to a DLQ
- **Delay queues**: Schedule message delivery after a configurable delay
- **Fan-out**: One message delivered to multiple consumer groups (pub/sub)
- **Priority queues**: Higher-priority messages consumed first
- **Message TTL**: Messages expire if not consumed within a time window

---

## 2. Non-Functional Requirements (NFRs)

- **Durability**: Messages must survive broker failures (persisted to disk, replicated)
- **High Availability**: 99.99% — queue unavailability blocks entire pipelines
- **High Throughput**: 1M+ messages/sec across the cluster
- **Low Latency**: End-to-end < 10 ms p99 for in-memory queues, < 50 ms for durable
- **Scalability**: Horizontally scalable — add brokers to increase throughput
- **Exactly-once processing** (optional): Via consumer-side idempotency
- **Backpressure**: Handle slow consumers without losing messages

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Messages / day | 50B |
| Messages / sec | ~580K (peak 2M) |
| Avg message size | 1 KB |
| Throughput | 580 MB/s avg, 2 GB/s peak |
| Retention | 7 days |
| Storage | 50B × 1 KB × 7 = 350 TB |
| With replication (RF=3) | ~1 PB |
| Active queues/topics | 100K |
| Consumer groups | 500K |

---

## 4. High-Level Design (HLD)

```
                    ┌─────────────────────────────────────────┐
                    │           Producer Pool                  │
                    │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      │
                    │  │P1   │ │P2   │ │P3   │ │P_N  │      │
                    │  │     │ │     │ │     │ │     │      │
                    │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘      │
                    └─────┼──────┼──────┼──────┼──────────────┘
                          │      │      │      │
                          │  Partition routing (key hash % N)
                          │      │      │      │
┌─────────────────────────▼──────▼──────▼──────▼──────────────────┐
│                     Broker Cluster                                │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐ │
│  │    Broker 1      │  │    Broker 2      │  │   Broker 3     │ │
│  │                  │  │                  │  │                │ │
│  │ ┌──────────────┐ │  │ ┌──────────────┐ │  │ ┌────────────┐ │ │
│  │ │ Topic:orders │ │  │ │ Topic:orders │ │  │ │Topic:orders│ │ │
│  │ │ Partition 0  │ │  │ │ Partition 1  │ │  │ │Partition 2 │ │ │
│  │ │ ★ LEADER     │ │  │ │ ★ LEADER     │ │  │ │★ LEADER    │ │ │
│  │ ├──────────────┤ │  │ ├──────────────┤ │  │ ├────────────┤ │ │
│  │ │ Topic:orders │ │  │ │ Topic:orders │ │  │ │Topic:orders│ │ │
│  │ │ Partition 2  │ │  │ │ Partition 0  │ │  │ │Partition 1 │ │ │
│  │ │ ○ FOLLOWER   │ │  │ │ ○ FOLLOWER   │ │  │ │○ FOLLOWER  │ │ │
│  │ ├──────────────┤ │  │ ├──────────────┤ │  │ ├────────────┤ │ │
│  │ │ Commit Log   │ │  │ │ Commit Log   │ │  │ │Commit Log  │ │ │
│  │ │┌────────────┐│ │  │ │┌────────────┐│ │  │ │┌──────────┐│ │ │
│  │ ││SegFile.log ││ │  │ ││SegFile.log ││ │  │ ││SegFile   ││ │ │
│  │ ││SegFile.idx ││ │  │ ││SegFile.idx ││ │  │ ││.log .idx ││ │ │
│  │ ││SegFile.ts  ││ │  │ ││SegFile.ts  ││ │  │ ││.ts       ││ │ │
│  │ │└────────────┘│ │  │ │└────────────┘│ │  │ │└──────────┘│ │ │
│  │ │ Page Cache   │ │  │ │ Page Cache   │ │  │ │Page Cache  │ │ │
│  │ └──────────────┘ │  │ └──────────────┘ │  │ └────────────┘ │ │
│  └──────────────────┘  └──────────────────┘  └────────────────┘ │
│                                                                   │
│  Internal Replication Protocol (Fetch requests between brokers)   │
└───────────────────────────┬───────────────────────────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │  Controller / Coordinator  │
              │  (KRaft / ZooKeeper)       │
              │                            │
              │  Responsibilities:         │
              │  • Partition → Leader map  │
              │  • ISR list maintenance    │
              │  • Leader election on fail │
              │  • Topic/Partition config  │
              │  • Broker liveness (heart.)│
              │  • Consumer group coords.  │
              └─────────────┬──────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                    Consumer Group Layer                           │
│                                                                   │
│  Consumer Group: "order-processor"                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │Consumer 1│  │Consumer 2│  │Consumer 3│                       │
│  │P0 ← Reads│  │P1 ← Reads│  │P2 ← Reads│                       │
│  │          │  │          │  │          │                       │
│  │Offset: 42│  │Offset: 87│  │Offset: 33│                       │
│  └──────────┘  └──────────┘  └──────────┘                       │
│                                                                   │
│  Consumer Group: "analytics-pipeline" (independent offsets)       │
│  ┌──────────┐  ┌──────────┐                                     │
│  │Consumer A│  │Consumer B│                                     │
│  │P0,P1     │  │P2        │                                     │
│  └──────────┘  └──────────┘                                     │
└──────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Broker — The Core Processing Unit

Each broker manages a subset of queues/partitions:

**Storage Engine — Append-Only Commit Log** ⭐:
```
Queue "orders" Partition 0:
  Segment File: 00000000000000000000.log
  ┌──────┬──────────┬───────────┬──────┐
  │Offset│Timestamp │Key        │Value │
  │  0   │171032000 │order-123  │{...} │
  │  1   │171032001 │order-124  │{...} │
  │  2   │171032002 │order-125  │{...} │
  └──────┴──────────┴───────────┴──────┘
  
  Segment File: 00000000000000050000.log  (new segment after 50K msgs or 1GB)
  Index File:   00000000000000000000.index (offset → file position mapping)
```

- **Why append-only log?** Sequential disk writes are 100× faster than random writes. Even on HDD, sequential I/O achieves 200+ MB/s
- **Segment rotation**: When a segment reaches size limit (1 GB) or age limit (7 days), create a new segment. Old segments are deleted after retention period
- **Index files**: Sparse index (every 4 KB) maps offset → byte position in log file. Binary search for O(log N) lookup
- **Page cache**: OS page cache acts as a natural read cache. Recent messages served from memory, old messages from disk

**Zero-Copy Transfer**:
```
Traditional: Disk → Kernel Buffer → User Buffer → Socket Buffer → NIC
Zero-Copy:   Disk → Kernel Buffer ──────────────→ NIC (via sendfile())

Result: 4× less data copying, 2× less context switches
Kafka and modern queues use this for consumer reads
```

#### Message Lifecycle

```
1. Producer sends message to broker
2. Broker appends to commit log (sequential write to disk)
3. Broker replicates to follower replicas (async or sync)
4. Broker ACKs producer (after replication, configurable)
5. Consumer polls for new messages (long-polling)
6. Broker sends messages from commit log (zero-copy read)
7. Consumer processes message
8. Consumer commits offset (ACK)
9. If no ACK within timeout → message redelivered to another consumer
```

#### Replication — Leader-Follower Per Partition

```
Partition "orders-0":
  Leader:    Broker 1  (accepts reads + writes)
  Follower:  Broker 2  (replicates from leader)
  Follower:  Broker 3  (replicates from leader)

ISR (In-Sync Replicas): {Broker 1, Broker 2, Broker 3}
  A replica is "in-sync" if it's within max.lag (e.g., 10 seconds) of the leader

Write Durability (acks config):
  acks=0:  Don't wait for any ACK (fastest, data loss possible)
  acks=1:  Wait for leader ACK only (leader writes to disk)
  acks=all: Wait for ALL ISR replicas to ACK (strongest, slowest)
```

#### Consumer Groups — Parallel Processing

```
Topic "orders" with 6 partitions:
Consumer Group "order-processor":
  Consumer A: reads from partitions 0, 1
  Consumer B: reads from partitions 2, 3
  Consumer C: reads from partitions 4, 5

If Consumer B dies:
  Rebalance → Consumer A: 0, 1, 2  |  Consumer C: 3, 4, 5

Key insight: Max parallelism = number of partitions
  10 consumers but 6 partitions → 4 consumers sit idle
  Solution: Choose partition count carefully (default: 12-64 for most topics)
```

#### Dead Letter Queue (DLQ)

```
Message processing fails → retry (up to N times with backoff)
After N retries → move message to DLQ topic: "orders.dlq"

DLQ allows:
  - Manual inspection of failed messages
  - Fix the bug → replay DLQ messages back to original topic
  - Alert on DLQ growth rate (indicates systematic failures)
```

#### Delay Queue Implementation

```
Approach 1: Per-delay-bucket topics
  Topic: "delayed-5s", "delayed-30s", "delayed-5m", "delayed-1h"
  Worker reads from each topic, holds messages until delay expires, 
  then publishes to target topic
  
Approach 2: Timer wheel (in-broker)
  Store delayed messages in a hierarchical timing wheel
  Timer fires at the right moment → message becomes visible
  Used by: RabbitMQ (delayed message plugin), AWS SQS delay queues

Approach 3: Sorted Set in Redis
  ZADD delayed_msgs {delivery_timestamp} {message_id}
  Worker polls: ZRANGEBYSCORE delayed_msgs 0 {now}
  Move due messages to the actual queue
```

---

## 5. APIs

### Producer
```
publish(topic, key, value, headers, partition) → {offset, partition, timestamp}
publish_batch(topic, messages[]) → {offsets[]}
```

### Consumer
```
subscribe(topics[], consumer_group)
poll(timeout_ms) → messages[]
commit(offsets)      // manual commit
seek(partition, offset)  // replay from specific position
```

### Admin
```
create_topic(name, partitions, replication_factor, config)
delete_topic(name)
describe_topic(name) → {partitions, replicas, ISR, offsets}
list_consumer_groups() → groups[]
get_consumer_lag(group, topic) → {partition: lag}
```

---

## 6. Data Model

### On-Disk — Commit Log Segment

```
Message Format:
┌──────────────────────────────────────────────┐
│ Offset (8 bytes)                              │ monotonically increasing
│ Message Size (4 bytes)                        │
│ CRC32 (4 bytes)                               │ integrity check
│ Magic (1 byte)                                │ format version
│ Compression (1 byte)                          │ none/gzip/snappy/lz4/zstd
│ Timestamp (8 bytes)                           │
│ Key Length (4 bytes) + Key (variable)         │
│ Value Length (4 bytes) + Value (variable)     │
│ Headers Length (4 bytes) + Headers (variable) │
└──────────────────────────────────────────────┘
```

### Metadata Store (ZooKeeper / etcd)

```
/brokers/ids/{broker_id} → {host, port, rack}
/topics/{topic}/partitions/{partition}/state → {leader, isr[], epoch}
/consumers/{group}/offsets/{topic}/{partition} → {offset, timestamp}
```

### Consumer Offset Storage

```
Option 1: ZooKeeper (Kafka legacy) — ZK becomes bottleneck
Option 2: Internal topic "__consumer_offsets" ⭐ (Kafka modern)
  - Compacted topic: only latest offset per consumer-group-partition retained
  - Replicated for durability
  - Consumers commit offsets by producing to this topic
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Broker failure** | Leader election from ISR; followers become new leader. Producers/consumers discover new leader from metadata |
| **Message loss** | acks=all ensures all ISR replicas have the message before ACK |
| **Consumer failure** | Consumer group rebalance redistributes partitions to surviving consumers |
| **Network partition** | Min ISR config prevents writes when too few replicas are available (no split-brain writes) |
| **Disk failure** | Replicas on different brokers/racks/AZs. Failed broker's data rebuilt from replicas |
| **Poison message** | DLQ after N retries prevents one bad message from blocking the queue |
| **Consumer lag** | Monitor lag per consumer group; alert when lag exceeds threshold; auto-scale consumers |

### Exactly-Once Semantics

```
Producer side (idempotent producer):
  Each producer has a unique PID (Producer ID)
  Each message has a sequence number per partition
  Broker deduplicates: if (PID, partition, sequence) already seen → discard
  
Consumer side (transactional processing):
  Read message → process → write result + commit offset in ONE transaction
  If consumer crashes before commit → message reprocessed → result is idempotent
  Requires: transactional producer API + read_committed isolation level
```

### Race Condition Deep Dives

#### 1. Consumer Rebalance — The Stop-the-World Problem

```
Scenario: Consumer Group with 3 consumers, 6 partitions.
  Consumer B crashes → triggers rebalance.

EAGER rebalance (legacy):
  T=0:    Consumer B crashes
  T=0.1s: Coordinator detects heartbeat timeout (session.timeout.ms)
  T=0.1s: Coordinator sends "REVOKE ALL" to Consumer A and C
  T=0.1s: ALL consumers STOP processing (even healthy ones!)
  T=0.2s: Consumers A and C re-join consumer group
  T=0.5s: Coordinator assigns: A → {P0,P1,P2}, C → {P3,P4,P5}
  T=0.5s: Consumers resume processing
  
  ⚠️ DOWNTIME: 0.5 seconds of ZERO processing across ALL partitions
  At 500K msgs/sec → 250K messages delayed during rebalance

COOPERATIVE (incremental) rebalance ⭐:
  T=0:    Consumer B crashes
  T=0.1s: Coordinator detects heartbeat timeout
  T=0.1s: Coordinator sends "REVOKE P2,P3 from B only" 
  T=0.1s: Consumer A keeps processing P0,P1 (never stopped!)
  T=0.1s: Consumer C keeps processing P4,P5 (never stopped!)
  T=0.2s: Coordinator assigns B's partitions: A gets P2, C gets P3
  
  ⚠️ Only B's partitions paused. A and C NEVER stopped.
  Downtime: 0.2 seconds, only for 2 out of 6 partitions
```

#### 2. Duplicate Message on Producer Retry

```
Producer sends message → Broker receives, writes to log, but ACK is LOST in network
Producer doesn't get ACK → retries → Broker receives AGAIN → writes DUPLICATE

Without idempotent producer:
  Offset 42: {order-123, "payment_confirmed"}    ← original
  Offset 43: {order-123, "payment_confirmed"}    ← DUPLICATE!
  
  Consumer processes both → double payment!

With idempotent producer ⭐:
  Producer sends: PID=7, Sequence=15, message={order-123, ...}
  Broker receives: check (PID=7, Partition=0, Sequence=15) — already seen!
  Broker returns: ACK with existing offset (no duplicate write)
  
  Implementation:
    Broker maintains: Map<(PID, Partition), max_sequence> in memory
    If incoming_seq <= stored_max_seq → duplicate → discard
    This map is persisted in the partition log (.snapshot file)
```

#### 3. Offset Commit Race — The "At-Least-Once" Gap

```
Scenario: Consumer reads message at offset 50, processes it, then commits.

RISK: Consumer crashes AFTER processing but BEFORE committing offset.
  T=0:   Consumer reads offset 50
  T=0.1: Consumer processes message (e.g., inserts into DB)
  T=0.2: Consumer CRASHES before commit()
  T=1.0: Consumer restarts, resumes from last committed offset = 49
  T=1.1: Consumer re-reads offset 50 → processes AGAIN → duplicate DB insert

Solutions:
  1. Idempotent consumer ⭐: Consumer writes to DB with idempotency key (message offset)
     INSERT INTO processed (id, data) VALUES (50, ...) ON CONFLICT DO NOTHING
  
  2. Exactly-once with transactions:
     Consumer reads msg → BEGIN TXN → write result + commit offset → END TXN
     Both succeed or both fail (requires Kafka Transactions API)
  
  3. Outbox pattern:
     Consumer writes result + offset to same DB in one transaction
     On restart, read last committed offset from DB, not Kafka
```

#### 4. ISR Shrinkage — Data Loss Window

```
Partition with ISR = {Broker1 (leader), Broker2, Broker3}

T=0:   Broker3 slows down (GC pause, disk I/O)
T=10s: Broker3 falls out of ISR. ISR = {Broker1, Broker2}
T=10s: acks=all now means acks={Broker1, Broker2} ← only 2 replicas!
T=15s: Broker1 (leader) crashes. Only Broker2 has all data.
T=15s: Broker2 becomes leader. Broker3 is behind → data gap.

If min.insync.replicas=2 and ISR drops to {Broker1} only:
  Broker REJECTS writes (NotEnoughReplicas exception)
  → No data loss but availability reduced (unavailable for writes)
  
Config recommendation for durability:
  acks=all + min.insync.replicas=2 + replication.factor=3
  → Guarantees at least 2 copies exist before ACK
  → Survives 1 broker failure without data loss
  → If 2 brokers fail → writes rejected (unavailable, but no loss)
```

### Scale Challenges

#### Partition Count Selection

```
Too few partitions (e.g., 1):
  → Max 1 consumer → throughput bottleneck (single-threaded consumption)
  → All data on one broker → hot spot

Too many partitions (e.g., 10,000 per topic):
  → Each partition = 1 open file handle → FD exhaustion
  → Each partition leader election takes time → slow recovery
  → More ZooKeeper/KRaft metadata → memory pressure
  → Consumer rebalance scans all partitions → slow rebalance

Formula for partition count:
  partitions = max(
    throughput_target / throughput_per_partition,
    max_consumer_count
  )
  
  Example: 100 MB/s target, 10 MB/s per partition → 10 partitions minimum
  But if we want 20 consumers → need 20 partitions

  Industry defaults: 6-64 partitions per topic
  Kafka limit: ~200K partitions per cluster (KRaft handles better than ZooKeeper)
```

#### Log Compaction — Keeping Only Latest Value Per Key

```
Normal retention: Delete entire segment after 7 days
Log compaction: Keep ONLY the latest message per key (forever)

Use case: Changelog / state store
  Key: user-123, Value: {name: "Alice", email: "alice@example.com"}  (v1)
  Key: user-123, Value: {name: "Alice Smith", ...}                    (v2)
  Key: user-456, Value: {name: "Bob", ...}                           (v1)
  
  After compaction:
  Key: user-123, Value: {name: "Alice Smith", ...}  (only latest v2 kept)
  Key: user-456, Value: {name: "Bob", ...}          (latest v1 kept)

  Tombstone: Key: user-123, Value: null → deletes the key after grace period

Used by: __consumer_offsets topic, Kafka Streams state stores, CDC pipelines
```

---

## 8. Additional Considerations

### Push vs Pull Consumers

| Model | How | Pros | Cons |
|---|---|---|---|
| **Push** (RabbitMQ) | Broker pushes to consumer | Lower latency; immediate delivery | Broker must track consumer pace; backpressure complexity |
| **Pull** (Kafka) ⭐ | Consumer polls broker | Consumer controls pace; batching; backpressure natural | Slight latency (poll interval); long-polling mitigates this |

### Backpressure Handling
- **Pull model**: Consumer simply polls slower → backpressure is automatic
- **Push model**: Broker must implement flow control (credit-based: "you can send me 100 more messages")
- **Queue depth monitoring**: Alert when queue depth > threshold → scale consumers

### Message Ordering Guarantees
```
Global ordering: Only possible with 1 partition → limits throughput to 1 consumer
Partition ordering: Messages with same key → same partition → ordered within partition
  Use: key = user_id → all events for a user are ordered
  
Caution: Consumer rebalance can temporarily break ordering
  Mitigation: Cooperative rebalance (incremental, not stop-the-world)
```

---

## 9. Deep Dive: Engineering Trade-offs

### Kafka vs RabbitMQ vs SQS: When to Choose What

| Feature | Kafka | RabbitMQ | AWS SQS |
|---|---|---|---|
| **Model** | Distributed log (pull) | Message broker (push) | Managed queue (pull) |
| **Throughput** | 1M+ msg/sec | ~50K msg/sec | ~3K msg/sec per queue |
| **Ordering** | Per-partition FIFO | Per-queue FIFO | FIFO queues (limited) |
| **Retention** | Configurable (days/weeks) | Until consumed | 14 days max |
| **Replay** | ✓ (seek to any offset) | ✗ (message deleted after ACK) | ✗ |
| **Exactly-once** | ✓ (transactional API) | ✗ (at-least-once) | ✗ (at-least-once) |
| **Routing** | Partition key | Exchange types (direct, fanout, topic, headers) | None |
| **Operations** | Complex (ZK/KRaft, brokers) | Moderate | Zero (managed) |

**Choose Kafka when**: High throughput, event streaming, replay needed, log compaction
**Choose RabbitMQ when**: Complex routing, priority queues, request-reply patterns, lower throughput
**Choose SQS when**: Simple decoupling, zero-ops, AWS-native, moderate throughput

### Append-Only Log vs Traditional Queue: The Fundamental Design Choice

```
Traditional Queue (RabbitMQ):
  Message consumed → DELETED from queue
  ✓ Simple mental model
  ✗ No replay (message gone forever after ACK)
  ✗ Multiple consumers need multiple copies
  ✗ Random deletes → disk fragmentation

Append-Only Log (Kafka) ⭐:
  Message consumed → offset advances, message STAYS
  ✓ Replay: reset offset to any point in time
  ✓ Multiple consumer groups read independently (no duplication)
  ✓ Sequential I/O → 10× higher throughput
  ✓ Natural event sourcing / audit log
  ✗ Storage grows with retention (must eventually delete old segments)
  ✗ No per-message deletion (can't remove a single "bad" message)
```

### In-Memory vs Disk-Backed: Latency vs Durability

```
In-memory queue (Redis, ZeroMQ):
  Latency: < 1 ms
  Durability: NONE (unless persisted)
  Throughput: Very high
  Use for: Inter-process communication, real-time signaling

Disk-backed with page cache (Kafka):
  Latency: 2-10 ms
  Durability: Survives process crash (data on disk)
  Throughput: High (sequential I/O + page cache)
  Use for: Event streaming, reliable messaging

Fully durable (acks=all, fsync):
  Latency: 10-50 ms
  Durability: Survives machine crash
  Throughput: Lower (must fsync to disk)
  Use for: Financial transactions, critical events
```

### ZooKeeper vs KRaft: Why Kafka Is Removing ZooKeeper

```
ZooKeeper (legacy):
  Separate cluster (3-5 ZK nodes) manages all Kafka metadata
  ✗ Operational burden: maintain two distributed systems
  ✗ ZK bottleneck: metadata operations serialized through ZK leader
  ✗ Partition limit: ZK struggles above ~200K partitions per cluster
  ✗ Slow controller failover: new controller must reload ALL metadata from ZK (minutes)
  ✗ ZK data model doesn't match Kafka's needs (ephemeral znodes, watches)

KRaft ⭐ (Kafka 3.3+, production-ready):
  Metadata managed by Kafka brokers themselves using Raft consensus
  ✓ No external dependency (single system to operate)
  ✓ Metadata in Raft log: faster reads, faster failover
  ✓ Scales to millions of partitions per cluster
  ✓ Controller failover in seconds (not minutes)
  ✓ Simpler operational model
  
  How KRaft works:
    3-5 brokers designated as "controllers" (voters in Raft quorum)
    One controller is the active controller (Raft leader)
    All metadata changes are committed to the metadata log (Raft)
    Non-controller brokers receive metadata updates via the log
```

### Producer Batching and Compression — The Throughput Multiplier

```
Without batching:
  1 message = 1 network round-trip = 1 disk write
  At 1 KB/msg → 1M msgs/sec = 1M network calls = IMPOSSIBLE

With batching ⭐:
  Producer buffers messages locally for up to linger.ms (e.g., 5ms)
  Or until batch reaches batch.size (e.g., 64 KB)
  Then sends ONE network request with entire batch

  Compression on the batch (snappy/lz4/zstd):
    64 KB batch → compressed to ~15 KB → 4× bandwidth reduction
    Compression happens once at producer, stored compressed on broker,
    decompressed at consumer → broker does zero compression work

  Impact:
    Without batching: 1M separate requests → overwhelm network
    With batching: 1M messages in ~15K batches → 15K network calls → easy
    
  Throughput improvement: 10-50× with batching + compression
```

### Message Delivery Semantics — A Complete Framework

```
┌────��─────────────────────────────────────────────────────────────────┐
│                        Delivery Guarantee Matrix                      │
├─────────────────┬────────────────┬────────────────┬──────────────────┤
│                 │ At-Most-Once   │ At-Least-Once  │ Exactly-Once     │
├─────────────────┼────────────────┼────────────────┼──────────────────┤
│ Producer        │ acks=0         │ acks=all       │ acks=all         │
│                 │ no retry       │ retries=MAX    │ enable.idempot.  │
│                 │                │                │ transactional.id │
├─────────────────┼────────────────┼────────────────┼──────────────────┤
│ Consumer        │ commit before  │ commit after   │ read_committed + │
│                 │ processing     │ processing     │ transactional    │
│                 │                │                │ offset commit    │
├─────────────────┼────────────────┼────────────────┼──────────────────┤
│ Data Loss?      │ YES            │ NO             │ NO               │
│ Duplicates?     │ NO             │ YES            │ NO               │
│ Use Case        │ Metrics/logs   │ Most workloads │ Financial/orders │
│ Performance     │ Fastest        │ Fast           │ Slowest (~20%)   │
└─────────────────┴────────────────┴────────────────┴──────────────────┘
```

