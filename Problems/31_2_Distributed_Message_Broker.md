# 31.2 Design a Distributed Message Broker (Kafka-style)

> **Key distinction from Worker Queue (31_1)**: A message broker / event log **retains messages after consumption** (append-only log), uses **offset-based tracking** (consumer tracks position, not broker), supports **replay** (seek to any offset), and is **pull-based** (consumer controls pace). Worker queues (RabbitMQ/SQS) delete messages after ACK — see `31_1_Distributed_Worker_Queue.md`.

---

## 1. Functional Requirements (FR)

- **Publish**: Producers publish messages to named topics
- **Subscribe**: Consumers subscribe to topics and receive messages in order
- **Persistence**: Messages are durably stored on disk for a configurable retention period
- **Consumer groups**: Multiple consumers in a group share the load; each message consumed by exactly one consumer in the group
- **Ordering**: Messages within a partition are strictly ordered (FIFO)
- **At-least-once, at-most-once, exactly-once** delivery semantics
- **Replay**: Consumers can re-read old messages by seeking to an offset
- **Partitioning**: Topics split into partitions for parallelism
- **Log compaction**: Retain only the latest value per key (changelog / CDC use case)
- **Schema evolution**: Support backward/forward compatible schema changes (via Schema Registry)

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: 1M+ messages/sec per broker (GBs per second)
- **Low Latency**: End-to-end < 10 ms (producer → consumer) for real-time use cases
- **Durability**: No message loss even during broker failures (acks=all + ISR)
- **Scalability**: Horizontal scaling — add brokers to increase throughput linearly
- **Fault Tolerant**: Survive broker, rack, and even AZ failures without data loss
- **Ordering Guarantees**: Per-partition strict ordering
- **Retention**: Configurable (time-based, size-based, or compaction)
- **Backpressure**: Gracefully handle slow consumers without affecting producers or other consumers
- **Multi-Tenancy**: Quotas per client to prevent noisy-neighbor problems

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Messages / sec (system-wide) | | 10M |
| Avg message size | | 1 KB |
| Throughput (write) | 10M × 1 KB | 10 GB/sec |
| Throughput (read) | 3 consumer groups × 10 GB/s | 30 GB/sec |
| Retention | | 7 days |
| Storage per day (raw) | 10 GB/s × 86400 | ~864 TB |
| Storage for 7 days | 864 × 7 | ~6 PB |
| Replication factor 3 | 6 PB × 3 | ~18 PB total storage |
| Brokers (12 TB usable each) | 18 PB / 12 TB | ~1500 brokers |
| Topics | | 10,000 |
| Partitions (total) | | 500,000 |
| Network per broker (write+read+repl) | | ~40 Gbps peak |

### Why These Numbers Matter

```
Producer network: 10 GB/s ingress across cluster
Replication network: 10 GB/s × 2 (RF=3 means 2 copies) = 20 GB/s intra-cluster
Consumer network: 30 GB/s egress (multiple consumer groups)
Total cluster I/O: ~60 GB/s → requires 25GbE or 100GbE NICs

Per broker (1500 brokers):
  Write: ~7 MB/s per broker (easily handled)
  Replicate: ~14 MB/s
  Serve reads: ~20 MB/s
  Disk: sequential writes at ~200 MB/s per SSD → comfortable headroom
```

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                              PRODUCER LAYER                                    │
│                                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Producer A  │  │  Producer B  │  │  Producer C  │  │  Producer N  │       │
│  │              │  │              │  │              │  │              │       │
│  │  Partitioner │  │  Partitioner │  │  Partitioner │  │  Partitioner │       ���
│  │  Serializer  │  │  Serializer  │  │  Serializer  │  │  Serializer  │       │
│  │  Batch buffer│  │  Batch buffer│  │  Batch buffer│  │  Batch buffer│       │
│  │  Compressor  │  │  Compressor  │  │  Compressor  │  │  Compressor  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                  │                 │                  │               │
│         └──────────┬───────┘                 └────────┬─────────┘               │
│                    │    Direct to partition leader    │                         │
│                    │    (no proxy / gateway)          │                         │
└────────────────────│─────────────────────────────────│─────────────────────────┘
                     │                                  │
┌────────────────────│──────────────────────────────────│─────────────────────────┐
│                    │     KAFKA CLUSTER                │                         │
│                    │                                  │                         │
│  ┌─────────────────▼──────────────────────────────────▼──────────────────────┐  │
│  │                     CONTROLLER (KRaft Quorum)                            │  │
│  │                                                                          │  │
│  │  Active Controller (Leader of __cluster_metadata partition)              │  │
│  │  ┌────────────────────────────────────────────────────────────────┐      │  │
│  │  │  Metadata:                                                     │      │  │
│  │  │  - Topic → partition list                                      │      │  │
│  │  │  - Partition → {leader_broker, ISR[], replicas[], epoch}       │      │  │
│  │  │  - Broker → {host, port, rack, status, capacity}              │      │  │
│  │  │  - Consumer group → {members, partition assignments}           │      │  │
│  │  │                                                                │      │  │
│  │  │  Responsibilities:                                             │      │  │
│  │  │  - Leader election when broker fails                           │      │  │
│  │  │  - Partition reassignment (on broker add/remove)               │      │  │
│  │  │  - ISR updates (add/remove followers from ISR)                 │      │  │
│  │  │  - Topic creation/deletion                                     │      │  │
│  │  │  - Quota enforcement decisions                                 │      │  │
│  │  └────────────────────────────────────────────────────────────────┘      │  │
│  │                                                                          │  │
│  │  Standby Controllers: Broker-2, Broker-5 (Raft followers, hot standby)  │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌───── RACK A (AZ-1) ─────┐  ┌───── RACK B (AZ-2) ─────┐  ┌── RACK C (AZ-3)│
│  │                          │  │                          │  │                 │
│  │  ┌────────────────────┐  │  │  ┌────────────────────┐  │  │  ┌─────────────│
│  │  │    Broker 1        │  │  │  │    Broker 2        │  │  │  │  Broker 3   │
│  │  │                    │  │  │  │                    │  │  │  │             │
│  │  │  Partition 0 (L)   │  │  │  │  Partition 0 (F)   │  │  │  │  Part 0 (F)│
│  │  │  Partition 3 (F)   │  │  │  │  Partition 1 (L)   │  │  │  │  Part 1 (F)│
│  │  │  Partition 5 (L)   │  │  │  │  Partition 4 (L)   │  │  │  │  Part 2 (L)│
│  │  │  Partition 7 (F)   │  │  │  │  Partition 6 (F)   │  │  │  │  Part 3 (L)│
│  │  │                    │  │  │  │                    │  │  │  │  Part 5 (F)│
│  │  │  ┌──────────────┐  │  │  │  ┌──────────────┐    │  │  │  │  Part 7 (L)│
│  │  │  │ Segment Files│  │  │  │  │ Segment Files│    │  │  │  │             │
│  │  │  │ .log .idx    │  │  │  │  │ .log .idx    │    │  │  │  │ ┌─────────┐│
│  │  │  │ .timeindex   │  │  │  │  │ .timeindex   │    │  │  │  │ │Segments ││
│  │  │  │              │  │  │  │  │              │    │  │  │  │ │.log .idx││
│  │  │  │ Page Cache   │  │  │  │  │ Page Cache   │    │  │  │  │ │         ││
│  │  │  └──────────────┘  │  │  │  └──────────────┘    │  │  │  │             │
│  │  │                    │  │  │                    │  │  │  │             │
│  │  │  Network: 25 GbE   │  │  │  Network: 25 GbE   │  │  │  │  25 GbE    │
│  │  │  Disk: 12× NVMe   │  │  │  Disk: 12× NVMe   │  │  │  │  12× NVMe  │
│  │  │  RAM: 256 GB       │  │  │  RAM: 256 GB       │  │  │  │  256 GB    │
│  │  │  (mostly pagecache)│  │  │  (mostly pagecache)│  │  │  │             │
│  │  └────────────────────┘  │  │  └────────────────────┘  │  │  └─────────────│
│  │                          │  │                          │  │                 │
│  │  ┌────────────────────┐  │  │  ┌────────────────────┐  │  │  ┌─────────────│
│  │  │    Broker 4        │  │  │  │    Broker 5        │  │  │  │  Broker 6   │
│  │  │    ...             │  │  │  │    ...             │  │  │  │  ...        │
│  │  └────────────────────┘  │  │  └────────────────────┘  │  │  └─────────────│
│  └──────────────────────────┘  └──────────────────────────┘  └─────────────────│
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Internal Topics (managed by Kafka)                                      │  │
│  │                                                                          │  │
│  │  __consumer_offsets (50 partitions) — Consumer group offset tracking     │  │
│  │  __transaction_state (50 partitions) — Transaction coordinator state     │  │
│  │  __cluster_metadata (1 partition) — KRaft metadata log                  │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
                     │
┌────────────────────│───────────────────────────────────────────────────────────┐
│                    │     CONSUMER LAYER                                        │
│                    │                                                           │
│  ┌─────────────────▼──────────────────────────────────────────────────────┐    │
│  │  Consumer Group A (3 consumers, processing order-events, 6 partitions)│    │
│  │                                                                        │    │
│  │  Consumer-A1: P0, P1  ←── Each consumer owns specific partitions      │    │
│  │  Consumer-A2: P2, P3  ←── Partition ownership is exclusive             │    │
│  │  Consumer-A3: P4, P5  ←── Rebalanced if consumer joins/leaves         │    │
│  │                                                                        │    │
│  │  Offset tracking: each consumer commits (partition, offset) to         │    │
│  │  __consumer_offsets topic periodically or after processing batch       │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │  Consumer Group B (independent — reads SAME topic from offset 0)      │    │
│  │  Consumer Group C (Kafka Streams app — read + transform + write)      │    │
│  │  Consumer Group D (Flink job — real-time aggregation)                  │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘

                          OPTIONAL ECOSYSTEM
┌────────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Schema       │  │ Kafka        │  │ MirrorMaker2 │  │ Kafka        │       │
│  │ Registry     │  │ Connect      │  │ (Cross-DC    │  │ REST Proxy   │       │
│  │              │  │              │  │  replication) │  │              │       │
│  │ - Avro/Proto │  │ - Source:    │  │              │  │ - HTTP →     │       │
│  │   schemas    │  │   MySQL CDC  │  │ - Active-    │  │   Kafka for  │       │
│  │ - Compat     │  │   S3, etc    │  │   passive    │  │   non-JVM    │       │
│  │   checks     │  │ - Sink:      │  │ - Active-    │  │   clients    │       │
│  │ - Versioning │  │   ES, HDFS   │  │   active     │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

### Core Architecture Deep Dive

#### Topics and Partitions
- A **topic** is a logical category of messages (e.g., `user-events`, `order-events`)
- Each topic is split into **partitions** (e.g., 12 partitions)
- Partitions enable **parallelism**: each partition can be read by one consumer in a group
- **Partition assignment**: Producer specifies partition key → hash(key) % num_partitions
- Messages within a partition are **strictly ordered** by offset (monotonically increasing integer)

```
Topic: user-events (3 partitions)

Partition 0: [msg0, msg1, msg2, msg3, ...]  → Offset 0, 1, 2, 3
Partition 1: [msg0, msg1, msg2, ...]         → Offset 0, 1, 2
Partition 2: [msg0, msg1, ...]               → Offset 0, 1

Key insight: ordering is ONLY within a partition, NOT across partitions.
  All events for user_id=123 go to same partition → ordered.
  Events across user_id=123 and user_id=456 have NO ordering guarantee.
```

#### Broker — Storage Engine

**How messages are stored on disk**:
- Each partition is a directory on the broker's filesystem
- Inside: a sequence of **segment files** (default 1 GB each)
- Each segment has:
  - `.log` file: Actual messages (append-only, immutable once rolled)
  - `.index` file: Sparse index mapping offset → file byte position
  - `.timeindex` file: Sparse index mapping timestamp → offset

```
/data/user-events-0/
  00000000000000000000.log      (segment 1: offsets 0 – 999,999)
  00000000000000000000.index    (sparse: every 4KB of data → one entry)
  00000000000000000000.timeindex
  00000000000001000000.log      (segment 2: offsets 1,000,000 – 1,999,999)
  00000000000001000000.index
  leader-epoch-checkpoint        (tracks leader changes for truncation)
```

**Why append-only is fast** (the #1 interview talking point):
```
1. Sequential writes → 600 MB/s on SSD (vs 10 MB/s random writes)
   Hard drives: sequential = 100 MB/s, random = 0.1 MB/s (1000× difference!)

2. OS page cache: Kafka's JVM heap is small (~6 GB)
   Data lives in OS page cache (remaining ~250 GB of RAM)
   Recent writes (last few seconds) are already in page cache
   → Consumer reading recent data hits page cache → zero disk I/O

3. Zero-copy transfer: sendfile() system call
   Traditional: disk → kernel buffer → user buffer → socket buffer → NIC (4 copies)
   Zero-copy:   disk → kernel buffer → NIC socket  (2 copies, no user-space copy)
   → Saves CPU and memory bandwidth → 2-3× throughput improvement

4. Batched I/O: Multiple messages per write → amortize fsync overhead
   Single write → flushes batch of 100s of messages
```

**Message Lookup by Offset**:
```
Consumer requests offset 1,500,042:
1. Binary search segment files by name → find segment starting at 1,000,000
2. Open 00000000000001000000.index → binary search for ≤ 1,500,042
   → Index entry: offset 1,500,000 → file position 483,200
3. Seek to position 483,200 in .log file
4. Scan forward 42 messages → found offset 1,500,042
Total: ~2 disk seeks + short scan → < 1ms from page cache
```

#### Replication — ISR (In-Sync Replicas)

Each partition has:
- **Leader**: Handles all reads and writes for that partition
- **Followers**: Continuously fetch from leader and append to their local log

```
Partition 0 (Topic: order-events):
  Leader:    Broker 1 (Rack A)  offset: 5,000,150   ← Serves all clients
  Follower:  Broker 4 (Rack B)  offset: 5,000,148   ← In ISR (2 behind, within threshold)
  Follower:  Broker 7 (Rack C)  offset: 5,000,150   ← In ISR (fully caught up)

  ISR = {Broker 1, Broker 4, Broker 7}

  High Watermark (HW) = min(ISR offsets) = 5,000,148
  → Consumers can only read up to HW (5,000,148)
  → Messages 5,000,149 and 5,000,150 are "uncommitted" (not yet replicated everywhere)
```

**ISR (In-Sync Replica Set)** — the core of Kafka's fault tolerance:
```
Followers that are "caught up" with the leader
"Caught up" = fetched within replica.lag.time.max.ms (default 30s)

If follower falls behind > 30s → removed from ISR (becomes "out of sync")
If follower catches up → re-added to ISR

Key configs:
  replication.factor=3         → 3 copies of each partition
  min.insync.replicas=2        → at least 2 replicas must ACK before commit
  acks=all                     → producer waits for ALL ISR replicas

Safety guarantee with (RF=3, min.ISR=2, acks=all):
  Can tolerate 1 broker failure with zero data loss
  If 2 brokers fail → partition becomes unavailable (won't accept writes)
  → Choose: availability (allow writes with 1 replica) vs durability (block writes)
```

**Leader Election** (on broker failure):
```
1. Controller detects broker failure (heartbeat timeout ~10s)
2. Controller selects new leader from ISR for each affected partition
3. Controller updates __cluster_metadata log
4. New leader starts serving reads and writes
5. Producers/consumers get metadata refresh → redirect to new leader

Election time: ~5-10 seconds (dominated by failure detection, not election itself)
During election: partition is UNAVAILABLE for reads/writes → brief outage
```

#### Rack-Aware Replication

```
Why: If leader and all followers are on the same rack/AZ → rack failure = total data loss

Solution: rack-aware replica assignment

  broker.rack=us-east-1a  (Broker 1, 4)
  broker.rack=us-east-1b  (Broker 2, 5)
  broker.rack=us-east-1c  (Broker 3, 6)

  Partition 0 replicas placed on: Broker 1 (1a), Broker 2 (1b), Broker 3 (1c)
  → Survives any single AZ failure

  Without rack-awareness: replicas might all land on Broker 1, 4 (same rack)
  → Rack failure = ALL replicas lost → unrecoverable data loss
```

#### Producer — Write Path (Detailed)

```
Producer.send(topic="orders", key="user_123", value=orderJson):

  ┌─ PRODUCER (Client-side) ──────────────────────────────────────┐
  │                                                                │
  │  1. Serialize: key → bytes, value → bytes (Avro/JSON/Proto)  │
  │  2. Partition: hash(key) % numPartitions → partition 7        │
  │  3. Batch: add to in-memory batch buffer for partition 7      │
  │     (accumulate until batch.size=16KB or linger.ms=5ms)       │
  │  4. Compress: compress batch with lz4 (3-5× reduction)       │
  │  5. Send: TCP request to leader of partition 7 (Broker 2)    │
  │                                                                │
  └────────────────────────────┬───────────────────────────────────┘
                               │
  ┌────────────────────────────▼───────────────────────────────────┐
  │  BROKER (Leader of partition 7)                                │
  │                                                                │
  │  6. Validate: CRC check, authorize producer, check quota      │
  │  7. Assign offsets: next sequential offsets for each message  │
  │  8. Append to active segment's .log file (sequential write)  │
  │  9. Update .index and .timeindex entries                      │
  │  10. Wait for followers to replicate (if acks=all):           │
  │      Follower on Broker 5 fetches → appends → ACKs           │
  │      Follower on Broker 3 fetches → appends → ACKs           │
  │  11. Advance High Watermark                                   │
  │  12. Send ProduceResponse to producer (success + offset)      │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘

Producer acks config (THE key durability knob):
  acks=0:  Fire and forget. Fastest. May lose messages silently.
  acks=1:  Leader ACK only. Fast. Loses if leader dies before replication.
  acks=all: ALL ISR ACK. Slowest (~5ms extra). No data loss with min.ISR≥2.
```

**Producer Partitioner Strategies**:
```
1. Key-based (default): hash(key) % numPartitions
   ✅ Same key → same partition → ordering guaranteed
   ❌ Hot key → hot partition (see "Hot Partition" section below)

2. Round-robin (no key): distribute evenly across partitions
   ✅ Even load distribution
   ❌ No ordering for any entity

3. Sticky partitioner (Kafka 2.4+): batch to random partition, stick until batch full
   ✅ Maximizes batch size → better compression, fewer requests
   ❌ No ordering guarantee (but often not needed for key-less messages)

4. Custom partitioner: application-specific logic
   Example: route by tenant_id to dedicated partitions for isolation
```

#### Consumer — Read Path (Detailed)

```
Consumer.poll(timeout=100ms):

  ┌─ CONSUMER (Client-side) ──────────────────────────────────────┐
  │                                                                │
  │  1. For each assigned partition, send FetchRequest to leader: │
  │     {partition: 7, offset: 5000100, max_bytes: 1MB}           │
  │  2. Leader reads from log:                                     │
  │     - Recent data → page cache HIT (zero disk I/O)            │
  │     - Old data → disk read (sequential, still fast)            │
  │  3. Leader sends response via zero-copy (sendfile)            │
  │  4. Consumer deserializes messages                             │
  │  5. Consumer processes messages (application logic)            │
  │  6. Consumer commits offset:                                   │
  │     Option A: auto-commit (every 5s) — at-least-once          │
  │     Option B: manual commit after processing — at-least-once  │
  │     Option C: commit BEFORE processing — at-most-once         │
  │                                                                │
  └────────────────────────────────────────────────────────────────┘
```

#### Consumer Group Rebalancing (Deep Dive)

```
Trigger: Consumer joins, leaves, crashes, or new partitions added

EAGER REBALANCING (legacy — "Stop the World"):
  1. All consumers in group stop processing and revoke ALL partitions
  2. Group coordinator assigns partitions from scratch
  3. All consumers resume with new assignment
  
  Problem: During rebalance (10-30 seconds), NO messages processed
  → "Rebalance storm": frequent consumer restarts → cascading pauses

COOPERATIVE INCREMENTAL REBALANCING (modern — preferred):
  1. Group coordinator computes new assignment
  2. Only AFFECTED consumers revoke their moved partitions
  3. Unaffected consumers CONTINUE processing (no pause)
  4. Revoked partitions assigned to new owners
  
  Result: < 1 second pause for affected partitions only
  Non-affected consumers: zero downtime
  Config: partition.assignment.strategy=CooperativeStickyAssignor

STATIC GROUP MEMBERSHIP (for Kubernetes):
  Each consumer has a fixed group.instance.id
  Consumer restart → same partitions assigned (no rebalance at all)
  → Critical for stateful consumers (Kafka Streams, Flink)
  → Prevents unnecessary rebalancing on rolling deployments
```

#### ZooKeeper / KRaft (Metadata Management)

**ZooKeeper** (legacy, being removed):
- Stores: broker registry, topic configs, partition-to-leader mapping, ACLs
- Handles leader election for partitions
- **Problem**: ZooKeeper is a bottleneck for large clusters (100K+ partitions)
- **Problem**: Separate operational burden (deploy, monitor, scale ZK independently)

**KRaft** (Kafka Raft — the replacement, production-ready since Kafka 3.3):
```
- Kafka's own Raft-based consensus layer for metadata
- Metadata stored in internal topic: __cluster_metadata
- Controller quorum: 3 or 5 brokers elected as controllers
  Active controller = Raft leader → handles all metadata changes
  Standby controllers = Raft followers → take over on failure
  
Benefits over ZooKeeper:
  1. No external dependency → simpler operations
  2. Metadata changes are faster (single round-trip vs ZK multi-step)
  3. Scales to millions of partitions (ZK struggled at ~200K)
  4. Metadata is a Kafka log → can snapshot + replay (familiar model)
```

---

## 5. APIs

### Producer API
```java
// Synchronous send (wait for ack)
ProducerRecord<String, String> record = 
    new ProducerRecord<>("order-events", orderId, orderJson);
RecordMetadata meta = producer.send(record).get();  // blocks until ack
log.info("Sent to partition {} at offset {}", meta.partition(), meta.offset());

// Asynchronous send (non-blocking, callback on completion)
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        log.error("Send failed: {}", exception.getMessage());
        // Retry logic or dead-letter handling
    }
});

// Transactional send (exactly-once across topics)
producer.beginTransaction();
producer.send(new ProducerRecord<>("orders", key, value));
producer.send(new ProducerRecord<>("inventory", key, inventoryUpdate));
producer.sendOffsetsToTransaction(offsets, consumerGroupId);  // atomic with consume
producer.commitTransaction();
```

### Consumer API
```java
consumer.subscribe(List.of("order-events"));  // subscribe by topic name
// OR: consumer.assign(List.of(new TopicPartition("orders", 0)));  // manual partition

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processOrder(record.key(), record.value(), record.offset());
    }
    consumer.commitSync();  // commit after successful processing
}

// Seek to specific offset (replay from a point)
consumer.seek(new TopicPartition("orders", 0), 5000000L);

// Seek to timestamp
consumer.offsetsForTimes(Map.of(tp, targetTimestamp));
```

### Admin API
```java
// Create topic with rack-aware placement
admin.createTopics(List.of(
    new NewTopic("order-events", 12, (short) 3)  // 12 partitions, RF=3
        .configs(Map.of(
            "retention.ms", "604800000",      // 7 days
            "min.insync.replicas", "2",
            "compression.type", "lz4"
        ))
));

// Reassign partitions (for broker decommission / rebalancing)
admin.alterPartitionReassignments(reassignments);

// Describe cluster
admin.describeCluster();  // brokers, controller, cluster ID
```

---

## 6. Data Model

### Message Format — Record Batch (on disk)

```
┌────────────────────────────────────────────────────────────────┐
│  Record Batch Header (61 bytes fixed)                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Base Offset (8 bytes) — first offset in this batch       │  │
│  │ Batch Length (4 bytes)                                    │  │
│  │ Partition Leader Epoch (4 bytes)                          │  │
│  │ Magic (1 byte) — record format version (currently 2)     │  │
│  │ CRC32 (4 bytes) — checksum of remaining bytes            │  │
│  │ Attributes (2 bytes) — compression, timestamp type,      │  │
│  │                        transactional, control batch       │  │
│  │ Last Offset Delta (4 bytes)                               │  │
│  │ Base Timestamp (8 bytes)                                  │  │
│  │ Max Timestamp (8 bytes)                                   │  │
│  │ Producer ID (8 bytes) — for idempotent/transactional      │  │
│  │ Producer Epoch (2 bytes)                                  │  │
│  │ Base Sequence (4 bytes) — for dedup                       │  │
│  │ Records Count (4 bytes)                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  Records (variable, compressed as a batch)                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Record 0:                                                 │  │
│  │   Length (varint), Attributes (1), Timestamp Delta (varint)│  │
│  │   Offset Delta (varint), Key Length (varint), Key (bytes) │  │
│  │   Value Length (varint), Value (bytes)                     │  │
│  │   Headers Count (varint), Headers[] (key-value pairs)     │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Record 1: ...                                             │  │
│  │ Record 2: ...                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

Key optimizations:
  - Offsets stored as DELTAS from base offset (varint = smaller)
  - Timestamps stored as DELTAS from base timestamp (varint)
  - Entire batch compressed as ONE unit (better ratio than per-message)
  - Producer ID + Sequence enable idempotent dedup at broker
```

### Segment Index Entry (Sparse)

```
Offset      Position (file byte offset)
0           0
4096        32768        ← Entry every ~4 KB of log data
8192        65536
12288       98304
...

Lookup offset 10000:
  Binary search index → nearest ≤ entry = 8192 at position 65536
  Seek to 65536, scan forward → find 10000
  Average scan: 2 KB (half of 4 KB interval)
```

### Consumer Offset Storage

```
Topic: __consumer_offsets (50 partitions, compacted)
Partition key: hash(group_id) % 50

Key:   {group_id: "order-processor", topic: "orders", partition: 7}
Value: {offset: 5000150, metadata: "", commit_timestamp: 1710403200000}

Compacted: only latest offset per (group, topic, partition) retained
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Broker failure** | ISR follower elected as new leader; producer/consumer redirect automatically via metadata refresh |
| **Message loss** | `acks=all` + `min.insync.replicas=2` → committed messages survive any single broker failure |
| **Consumer failure** | Consumer group rebalances; another consumer takes over the failed consumer's partitions |
| **Disk failure** | Data replicated on 2 other brokers in different racks; replace disk and re-replicate |
| **Rack/AZ failure** | Rack-aware replication ensures replicas across racks; survive full AZ outage |
| **Network partition** | ISR mechanism: follower removed from ISR if can't reach leader; prevents stale reads |
| **Split-brain** | KRaft quorum or ZK ensemble ensures single controller; fencing via leader epochs |
| **Unclean leader election** | `unclean.leader.election.enable=false` → only ISR members become leader (no data loss) |
| **Producer duplicate** | Idempotent producer: ProducerID + SequenceNumber → broker deduplicates retried sends |
| **Slow consumer** | Consumer lag grows but doesn't affect producers or other consumers (pull model) |

### Exactly-Once Semantics (EOS) — Deep Dive

```
THREE levels of delivery guarantees:

AT-MOST-ONCE (acks=0 or commit-before-process):
  Producer: fire and forget → message may be lost if broker crashes
  Consumer: commit offset BEFORE processing → crash after commit = skipped message
  Use case: Metrics, logs (tolerable loss)

AT-LEAST-ONCE (acks=all + commit-after-process):
  Producer: retry on failure → may produce duplicate if original ACK was lost
  Consumer: commit offset AFTER processing → crash before commit = reprocess
  Use case: Most applications (with idempotent downstream)

EXACTLY-ONCE (EOS):
  Producer side: Idempotent producer (PID + sequence number)
    → Broker detects retried message → dedup → no duplicate in partition
  
  Consumer side: Transactional consume-transform-produce
    → Read from input topic + write to output topic + commit input offsets
       ALL in one atomic transaction
    → If crash mid-transaction → entire transaction rolled back
    → Consumer with isolation.level=read_committed → only sees committed data

  Kafka Streams achieves EOS automatically:
    input → process → output + offset commit → all transactional
```

### Controlled Shutdown (Broker Maintenance)

```
Problem: Taking a broker offline for maintenance → partition leaders move → brief unavailability

Controlled shutdown sequence:
1. Admin signals broker to shut down gracefully
2. Broker tells controller: "I'm leaving"
3. Controller moves ALL leadership off this broker BEFORE it stops:
   Partition 7 leader: Broker 3 → Broker 5 (within ISR)
   Partition 12 leader: Broker 3 → Broker 1 (within ISR)
4. Once all leaders moved → broker shuts down
5. Zero downtime: leadership transferred BEFORE broker stops

Uncontrolled shutdown (crash):
1. Controller detects missed heartbeat (~10s)
2. Controller moves leaders after detection → 10-15 second outage per partition
```

### Broker Decommission and Partition Reassignment

```
Scenario: Remove Broker 3 from cluster (hardware refresh)

1. Generate reassignment plan:
   kafka-reassign-partitions --generate → suggests new replica placement
   
2. Execute reassignment:
   Broker 3's replicas copied to new brokers (background, throttled to avoid network saturation)
   e.g., Partition 7 replicas: [Broker 3, 5, 1] → [Broker 4, 5, 1]
   
3. Copy phase:
   New replica (Broker 4) fetches data from leader (may take hours for large partitions)
   Throttled to limit.bytes.per.second to avoid saturating network
   
4. Completion:
   Once Broker 4 catches up → added to ISR → Broker 3 removed from replica set
   Controller updates metadata
   
5. Decommission:
   Broker 3 has no replicas → safe to shut down
```

### Race Condition Deep Dives

#### 1. Producer Retry Duplicate — The Idempotent Producer ⭐

```
Scenario: Producer sends message to leader. Leader writes it. ACK is LOST in network.
Producer doesn't get ACK → retries → Leader receives AGAIN → writes DUPLICATE.

Without idempotent producer:
  Offset 42: {order-123, "payment_confirmed"}    ← original
  Offset 43: {order-123, "payment_confirmed"}    ← DUPLICATE!
  Consumer processes both → double payment!

With idempotent producer ⭐ (enable.idempotence=true, default since Kafka 3.0):
  Producer sends: PID=7, Sequence=15, message={order-123, ...}
  Broker receives: writes to log, ACK lost
  Producer retries: PID=7, Sequence=15 (same sequence)
  Broker checks: (PID=7, Partition=0, Sequence=15) already seen → DEDUP → return existing offset

  Implementation:
    Broker maintains per partition: Map<PID, last_5_sequence_numbers> in memory
    This map is persisted in .snapshot files alongside segment files
    If incoming_seq ≤ stored_last_seq → duplicate → discard + return success
    If incoming_seq > stored_last_seq + 1 → out of order → reject (OutOfOrderSequence)
    
  Window: broker tracks last 5 ProducerIDs' sequences per partition
  PID is assigned by the transaction coordinator (or randomly for non-transactional)
```

#### 2. Consumer Offset Commit Race — The "At-Least-Once" Gap ⭐

```
Scenario: Consumer reads message at offset 50, processes it, then commits.

  T=0:   Consumer reads offset 50
  T=0.1: Consumer processes message (e.g., writes to DB)
  T=0.2: Consumer CRASHES before commitSync()
  T=1.0: Consumer restarts, resumes from last committed offset = 49
  T=1.1: Consumer re-reads offset 50 → processes AGAIN → duplicate DB write!

This is the fundamental at-least-once gap. Every Kafka consumer must handle it.

Solutions:
  1. Idempotent consumer ⭐ (most common):
     Consumer writes to DB with idempotency key = (topic, partition, offset)
     INSERT INTO results (id, data) VALUES ('orders-7-50', ...) ON CONFLICT DO NOTHING
     Duplicate reprocessing → DB rejects → no side effect
  
  2. Exactly-once with Kafka Transactions:
     Consumer reads → BEGIN TXN → produce to output topic + commit input offset → COMMIT TXN
     Both succeed atomically or both fail
     Requires: transactional.id on producer, isolation.level=read_committed on downstream consumer
  
  3. Outbox pattern:
     Consumer writes result + consumed offset to same DB in one DB transaction
     On restart, read last processed offset from DB (not from __consumer_offsets)
     Seek to that offset → no duplicates

  4. Auto-commit (enable.auto.commit=true, every 5s):
     Even WORSE: commit happens on timer, not tied to processing
     Crash between auto-commit and processing → message SKIPPED (at-most-once!)
     Crash after processing but before auto-commit → message REPROCESSED
     → Manual commit is almost always preferred for important data
```

#### 3. ISR Shrinkage — The Data Loss Window ⭐

```
Partition with ISR = {Broker1 (leader), Broker2, Broker3}
Config: acks=all, min.insync.replicas=2, replication.factor=3

T=0:   All healthy. ISR = {B1, B2, B3}. acks=all means all 3 ACK.
T=10s: Broker3 hits GC pause, falls behind > replica.lag.time.max.ms
T=10s: Controller removes B3 from ISR. ISR = {B1, B2}
T=10s: acks=all now means acks={B1, B2} only ← reduced redundancy!
T=15s: Broker1 (leader) crashes. B2 has all committed data → becomes leader.
T=15s: B3 is behind → when it recovers, it truncates to High Watermark and re-fetches.

  → No data loss in this scenario (B2 had everything committed).

BUT if min.insync.replicas=1 (dangerous config):
T=10s: ISR shrinks to {B1, B2}
T=12s: B2 also falls out of ISR. ISR = {B1} (just the leader!)
T=12s: acks=all = acks={B1} only = effectively acks=1 ← DANGER
T=13s: B1 crashes → messages since B2 fell out are LOST (only existed on B1)

Prevention:
  min.insync.replicas=2 ⭐ → if ISR drops below 2, broker REJECTS writes
  (NotEnoughReplicasException) → unavailable but NO DATA LOSS
  
  Config recommendation for durability:
    acks=all + min.insync.replicas=2 + replication.factor=3
    → Tolerates 1 broker failure with zero data loss
    → 2 failures → writes rejected (unavailable, not lossy)
```

#### 4. Leader Epoch Fencing — Preventing Split-Brain Writes ⭐

```
Problem: Old leader doesn't know it's been replaced → accepts stale writes.

Scenario (without fencing):
  T=0:   B1 is leader for partition 0, epoch=5
  T=1:   Network partition isolates B1 from controller and followers
  T=10s: Controller detects B1 is dead → elects B2 as new leader, epoch=6
  T=11s: B1's network recovers. B1 still thinks it's leader (epoch=5)
  T=11s: Producer (with stale metadata) sends to B1 → B1 accepts!
  T=11s: B2 (actual leader) also accepts writes → DIVERGED LOGS!

Solution — Leader Epoch (fencing token):
  Every leadership change increments the epoch number
  
  T=0:   B1 is leader, epoch=5. All writes include epoch=5.
  T=10s: Controller elects B2, epoch=6.
  T=11s: B1's network recovers:
         - Followers fetch from B2 now (epoch=6) → B1 gets no fetch requests
         - B1 sends metadata request → learns new epoch=6 → steps down
         - Producer sends to B1 with epoch=5 → B1 checks: my epoch < current → REJECT
         - Producer gets error → refreshes metadata → discovers B2 is leader → redirects

  Follower truncation on leader change:
    B1 (old leader) may have messages at offset 5,000,150 that were never committed
    New leader B2's log ends at 5,000,148
    B1 becomes follower → fetches from B2 → B2 says "epoch 6 starts at offset 5,000,148"
    B1 truncates offsets 5,000,149-150 → re-fetches from B2 → logs converge

  This is why unclean.leader.election.enable=false is critical:
    If enabled → out-of-ISR replica can become leader → LOSES committed messages
    If disabled → only ISR members become leader → may be unavailable but never lossy
```

#### 5. Consumer Rebalance — The Stop-the-World Problem

```
Consumer Group with 3 consumers, 6 partitions. Consumer B crashes.

EAGER rebalance (legacy):
  T=0:    Consumer B crashes
  T=0.1s: Coordinator detects missed heartbeat (session.timeout.ms=10s default)
  T=10s:  Coordinator marks B as dead → triggers rebalance
  T=10s:  ALL consumers (A and C too!) STOP processing + revoke ALL partitions
  T=10.5s: A and C re-join group, coordinator reassigns
  T=11s:  A → {P0,P1,P2}, C → {P3,P4,P5}
  T=11s:  Consumers resume
  
  ⚠️ TOTAL PAUSE: ~11 seconds. ALL partitions stopped.
  At 500K msgs/sec → 5.5M messages delayed during rebalance

COOPERATIVE incremental rebalance ⭐ (CooperativeStickyAssignor):
  T=0:    Consumer B crashes
  T=10s:  Coordinator detects B is dead
  T=10s:  Coordinator sends: "revoke ONLY B's partitions (P2, P3)"
  T=10s:  Consumer A keeps processing P0, P1 (NEVER STOPPED!)
  T=10s:  Consumer C keeps processing P4, P5 (NEVER STOPPED!)
  T=10.5s: B's partitions reassigned: A → {P0,P1,P2}, C → {P3,P4,P5}
  
  ⚠️ Only B's 2 partitions paused for ~0.5 seconds. A and C NEVER stopped.

STATIC GROUP MEMBERSHIP (group.instance.id) ⭐:
  Consumer B restarts (rolling deploy) with same instance ID
  → Coordinator recognizes it → assigns SAME partitions → NO rebalance at all
  → Session timeout extended (session.timeout.ms=300s for static members)
  → Perfect for Kubernetes rolling deployments
```

#### 6. Follower Fetch Protocol — How Replication Actually Works

```
Followers do NOT receive pushes from the leader. They PULL (fetch).

Fetch Loop (on each follower):
  while (true) {
    FetchRequest to leader: {partition: 0, fetch_offset: 5000148, max_bytes: 1MB}
    Leader responds: {messages from offset 5000148..5000200, leader_epoch: 5, HW: 5000145}
    Follower appends messages to local log
    Follower advances local High Watermark to min(leader_HW, local_log_end)
    Repeat immediately (no delay for real-time replication)
  }

Why pull, not push?
  1. Follower controls pace → natural backpressure
  2. Simpler: leader doesn't track each follower's state
  3. Follower crash → just stops fetching → leader doesn't care
  4. Follower recovery → starts fetching from last offset → catches up automatically

Replication latency:
  Network RTT between brokers: ~0.5ms (same DC)
  Fetch interval: continuous (next fetch immediately after previous completes)
  Typical replication lag: 1-5ms (sub-millisecond for colocated brokers)
  ISR threshold: 30 seconds (VERY generous → covers GC pauses, temp issues)
```

---

## 8. Additional Considerations & Deep Dives

### High Watermark (HW) and Log End Offset (LEO) — The Commit Model ⭐

```
Every partition has TWO key offset markers:

  Log End Offset (LEO): The offset of the NEXT message to be written
    → Leader's LEO advances on every produce
    → Follower's LEO advances as it replicates

  High Watermark (HW): The offset up to which ALL ISR replicas have replicated
    → HW = min(LEO of all ISR members)
    → Consumers can ONLY read up to HW (committed data only)
    → Messages between HW and LEO are "uncommitted" — may be lost on failure

  Example:
    Leader  LEO=150, HW=148
    Follower A LEO=150
    Follower B LEO=148  ← slowest ISR member
    
    HW = min(150, 150, 148) = 148
    Consumer sees offsets 0..147 (up to HW-1)
    Offsets 148, 149 exist on leader but are NOT yet committed

  Why this matters:
    If leader crashes BEFORE HW advances to 150:
      New leader (Follower B) only has up to 148
      Offsets 148-149 are LOST (they were uncommitted)
      → This is expected and correct behavior with acks=1
      → With acks=all, producer only gets ACK after HW advances → no surprise loss

  HW propagation:
    Follower sends FetchRequest with its LEO to leader
    Leader uses follower LEOs to compute HW
    Leader sends HW back in FetchResponse
    Follower updates its local HW = min(leader_HW, own_LEO)
    
    This means HW propagation has a ONE-FETCH-ROUND-TRIP delay
    → Brief window where follower's HW lags leader's HW
    → Not a problem: consumers read from leader, which has the latest HW
```

### Hot Partition Problem

```
Problem: Producer key distribution is skewed
  90% of orders are from top 1% of users
  hash("celebrity_user") % 12 = partition 7 → partition 7 gets 100× more writes
  → Broker hosting partition 7's leader is overloaded

Detection:
  Monitor bytes-in per partition → alert if any partition > 5× average

Solutions:
1. Application-level: add random suffix to key
   Key: "user_123_" + random(0,9) → spreads across 10 partitions
   Trade-off: lose ordering for that key (usually acceptable for hot keys)

2. Salted partition key:
   if (isHotKey(key)) partition = hash(key + timestamp % 10) % numPartitions
   else partition = hash(key) % numPartitions

3. More partitions: increase from 12 → 120 → reduces per-partition skew
   Trade-off: more metadata, more file handles, longer rebalances

4. Dedicated topic for hot entities (extreme case):
   Topic "orders-celebrity" with 50 partitions, separate consumer group
```

### Kafka vs. Other Message Systems

| Feature | Kafka | RabbitMQ | AWS SQS | Pulsar |
|---|---|---|---|---|
| Model | Append-only log (pull) | Queue (push) | Queue (pull) | Log (pull/push) |
| Ordering | Per-partition | Per-queue | FIFO queues only | Per-partition |
| Throughput | 1M+ msg/s/broker | 50K msg/s | Unlimited (managed) | 1M+ msg/s |
| Replay | Yes (offset seek) | No (consumed = gone) | No | Yes (cursor) |
| Persistence | Disk (retention) | Memory + disk | Managed | BookKeeper |
| Multi-tenancy | Weak (quotas) | Fair (vhosts) | Good (per-queue) | Strong (native) |
| Tiered storage | Plugin (newer) | No | N/A | Native (BookKeeper) |
| Best for | Event streaming, CDC, analytics | Task queues, RPC | Simple async, serverless | Multi-tenant streaming |

### Log Compaction (Deep Dive)

```
Normal retention: delete segments older than 7 days (regardless of content)
Compaction: keep ONLY the latest value per key (forever, or until explicit delete)

Before compaction (segment contains):
  offset 100: key=user_1, value={name:"Alice"}
  offset 101: key=user_2, value={name:"Bob"}
  offset 102: key=user_1, value={name:"Alice V2"}
  offset 103: key=user_1, value=null              ← tombstone (delete marker)

After compaction:
  offset 101: key=user_2, value={name:"Bob"}       ← latest for user_2
  offset 103: key=user_1, value=null                ← kept for delete.retention.ms

Use cases:
  - CDC: topic represents current state of a database table
  - Event sourcing: latest state per entity
  - KTable in Kafka Streams
  - __consumer_offsets topic (compacted: latest offset per group)
```

### Tiered Storage (Solving the Cost Problem)

```
Problem: 7 days × 864 TB/day × RF=3 = 18 PB on broker NVMe SSDs → EXPENSIVE

Solution: Tiered storage (Kafka 3.6+)
  Hot tier (local NVMe): Last 4-24 hours of data → ~200 TB
  Cold tier (S3/GCS): Remaining retention period → 17.8 PB at S3 prices

  How it works:
  1. Segments older than local.retention.ms → uploaded to S3
  2. Local copy deleted → frees broker disk
  3. Consumer requesting old offset → transparent fetch from S3
  4. Slightly higher latency for cold reads (S3 ~50ms vs local ~1ms)

  Cost impact:
  NVMe SSD: ~$100/TB/month → 18 PB = $1.8M/month
  S3:       ~$23/TB/month  → 17.8 PB = $409K/month + 200 TB SSD = $20K
  Savings: ~75% reduction in storage cost
```

### Multi-Datacenter Replication

```
Problem: Single-cluster Kafka in one region → region failure = total outage

Solution: MirrorMaker 2 (Kafka Connect-based cross-cluster replication)

  Cluster A (us-east) ←── MirrorMaker 2 ──→ Cluster B (eu-west)

  Active-Passive:
    Cluster A: all producers write here
    Cluster B: replica (read-only, for DR)
    Failover: redirect producers to Cluster B
    RPO: seconds (replication lag)

  Active-Active:
    Both clusters accept writes (different topics or partitioned by region)
    MirrorMaker replicates each direction
    Challenge: avoid infinite replication loops (topic name prefixing)
      us-east.orders → replicated to eu-west as us-east.orders (prefixed)
      eu-west.orders → replicated to us-east as eu-west.orders

  Offset translation:
    Offset 5000 in Cluster A ≠ offset 5000 in Cluster B
    MirrorMaker maintains offset mapping: checkpoints topic
    On failover → consumer reads mapping → seeks to correct offset in Cluster B
```

### Backpressure and Quota Management

```
Problem: One tenant producing 10× expected volume → starves other tenants

Solution: Kafka Quotas (per client-id or per user)
  Producer quota: max 50 MB/s per client-id
    If exceeded → broker throttles (delays response) → producer slows down naturally
  
  Consumer quota: max 100 MB/s per client-id
    If exceeded → broker delays fetch response
  
  Request quota: max 50% of broker request handler threads per client-id

  NOT a hard reject — gentle throttling via delayed responses
  → Producer/consumer backs off naturally without errors

Consumer backpressure (architectural):
  Kafka is pull-based → slow consumer just polls less frequently
  Consumer lag increases but producers and other consumers unaffected
  This is a FUNDAMENTAL advantage over push-based systems (RabbitMQ)
  → No broker memory pressure from slow consumers (messages on disk)
```

### Monitoring — What to Alert On

```
CRITICAL alerts:
  - Under-replicated partitions > 0 (data at risk)
  - Offline partitions > 0 (partition unavailable)
  - ISR shrink rate > threshold (replicas falling behind)
  - Consumer lag > threshold (processing falling behind)

WARNING alerts:
  - Disk usage > 80% on any broker
  - Request latency p99 > 100ms
  - Network utilization > 70%
  - Controller election (unexpected leader change)
  - Consumer group rebalance frequency > threshold
  - Unclean leader election count > 0

Dashboards:
  - Messages in / out per second (per topic, per broker)
  - Bytes in / out (network utilization)
  - Consumer lag per group per partition (THE most important metric)
  - Request handler idle ratio (< 30% = overloaded)
  - Log flush latency (disk I/O health)
  - Partition count per broker (balance check)
```

### Performance Tuning Cheat Sheet (Senior/Staff Level)

```
Producer tuning:
  batch.size=65536 (64 KB)     → larger batches = higher throughput
  linger.ms=10                  → wait 10ms to fill batch
  compression.type=lz4          → best throughput/compression trade-off
  buffer.memory=67108864 (64MB) → buffer for async sends
  acks=all                      → durability (always for important data)

Consumer tuning:
  fetch.min.bytes=1048576 (1 MB)     → wait for 1 MB before returning
  fetch.max.wait.ms=500              → max wait for min.bytes
  max.poll.records=500               → messages per poll() call
  max.poll.interval.ms=300000 (5min) → max time between polls before considered dead

Broker tuning:
  num.io.threads=16                  → disk I/O threads (match # of disks)
  num.network.threads=8              → network threads (match # of NICs)
  socket.send.buffer.bytes=1048576   → 1 MB socket buffer
  log.segment.bytes=1073741824 (1GB) → segment size
  log.retention.hours=168 (7 days)   → retention period
  num.partitions=12                  → default partitions per topic

Golden rule: partitions per broker < 4000 (beyond = high metadata overhead)
Golden rule: partition count = max(producer_throughput / single_partition_throughput, consumer_count)
```

### Partition Count Selection — The #1 Operational Decision

```
Choosing the right partition count is IRREVERSIBLE (can increase, but NEVER decrease).

Too few partitions (e.g., 1):
  → Max 1 consumer per group → throughput bottleneck
  → All data on one broker → hot spot
  → Single point of failure for that topic

Too many partitions (e.g., 50,000 per topic):
  → Each partition = open file handles (.log + .index + .timeindex) → FD exhaustion
  → Each partition leader election takes time → slow recovery on broker crash
  → More metadata in KRaft → memory pressure on controllers
  → Consumer rebalance scans all partitions → slow rebalance
  → More end-to-end latency (more partitions = more replication work)

Sizing formula:
  partitions = max(
    target_throughput / throughput_per_partition,
    max_consumer_count_per_group
  )
  
  Example: 
    Target: 100 MB/s for topic. Single partition: ~10 MB/s throughput.
    → 10 partitions minimum for throughput.
    But we want 20 consumers for processing → need 20 partitions.
    → Choose 20 partitions.

  Industry defaults: 6-64 partitions per topic
  LinkedIn (Kafka creators): typically 8-32 partitions per topic
  Cluster limit: ~200K partitions with ZooKeeper, millions with KRaft
  Per-broker guideline: < 4,000 partitions (leader + follower combined)

  ⚠️ Increasing partitions: allowed (kafka-topics --alter --partitions 24)
     BUT: existing keys will remap to different partitions → breaks ordering
     New partition has no historical data → consumers see incomplete history
  
  ⚠️ Decreasing partitions: IMPOSSIBLE. Must recreate topic.
     → Always start slightly higher than you think you need.
```

### Schema Registry — Schema Evolution Without Breaking Consumers ⭐

```
Problem: Producer changes message format → consumer crashes (can't deserialize)
  V1: {user_id, name, email}
  V2: {user_id, name, email, phone}    ← New field added
  V3: {user_id, full_name, email, phone} ← Field renamed (name → full_name)

  Old consumers reading V2/V3 messages → unknown fields → crash or data loss?

Solution: Schema Registry (Confluent or Apicurio)

  Architecture:
    Producer → serialize with schema (Avro/Protobuf/JSON Schema)
           → register schema in Schema Registry (HTTP API)
           → send (schema_id + binary data) to Kafka
    
    Consumer → read (schema_id + binary data) from Kafka
            → fetch schema by ID from Schema Registry (cached)
            → deserialize with schema → process

  Message on the wire:
  ┌──────────┬──────────┬────────────────────┐
  │ Magic (1)│Schema ID │ Avro-encoded data  │
  │  0x00    │ (4 bytes)│ (no schema overhead)│
  └──────────┴──────────┴────────────────────┘
  
  Schema is stored ONCE in registry, referenced by ID in every message
  → Massive space savings vs embedding schema in every message (JSON overhead)

Compatibility modes (THE interview question):
┌───────────────┬──────────────────────────────────────────────────────┐
│ BACKWARD ⭐   │ New schema can READ data written by old schema       │
│ (default)     │ Allowed: add field with default, remove field        │
│               │ Blocked: add required field without default          │
│               │ Use: most common. Deploy consumers first, then prods │
├───────────────┼──────────────────────────────────────────────────────┤
│ FORWARD       │ Old schema can READ data written by new schema       │
│               │ Allowed: remove field, add optional field            │
│               │ Blocked: add required field                          │
│               │ Use: deploy producers first, then consumers          │
├───────────────┼──────────────────────────────────────────────────────┤
│ FULL          │ Both backward AND forward compatible                 │
│               │ Only: add/remove OPTIONAL fields with defaults       │
│               │ Use: strictest, safest for independent deployments   │
├───────────────┼──────────────────────────────────────────────────────┤
│ NONE          │ No compatibility check — anything goes               │
│               │ Use: development only, never production              │
└───────────────┴──────────────────────────────────────────────────────┘

Avro vs Protobuf vs JSON Schema:
  Avro:       Compact binary, schema evolution built-in, Kafka's native choice
  Protobuf:   Compact binary, strong typing, popular in gRPC ecosystems
  JSON Schema: Human-readable, larger on wire, easier debugging
  
  Recommendation: Avro for Kafka (best ecosystem support), Protobuf if already using gRPC
```

### Kafka Connect — Scalable Data Integration

```
Problem: Every team writes custom producers/consumers to move data in/out of Kafka
  → Duplicated effort, inconsistent error handling, no monitoring

Solution: Kafka Connect — a framework for scalable, fault-tolerant data pipelines

  Source Connectors (external → Kafka):
    - Debezium (MySQL/Postgres CDC → Kafka) ⭐ most popular
    - JDBC Source (poll-based database → Kafka)
    - S3 Source, Kinesis Source, MQ Source

  Sink Connectors (Kafka → external):
    - Elasticsearch Sink (search indexing)
    - S3 Sink (data lake / archival)
    - HDFS Sink (Hadoop)
    - JDBC Sink (database)
    - BigQuery / Snowflake Sink

  Architecture:
    ┌─────────────────────────────────────────────────────┐
    │  Kafka Connect Cluster (3+ worker nodes)             │
    │                                                      │
    │  Worker 1: [Debezium-MySQL task 0] [S3-Sink task 0] │
    │  Worker 2: [Debezium-MySQL task 1] [ES-Sink task 0] │
    │  Worker 3: [Debezium-MySQL task 2] [ES-Sink task 1] │
    │                                                      │
    │  Connector config stored in: connect-configs topic   │
    │  Connector offsets stored in: connect-offsets topic   │
    │  Connector status stored in: connect-status topic     │
    └─────────────────────────────────────────────────────┘

  Key properties:
    - Distributed mode: tasks balanced across workers, auto-rebalance on failure
    - Exactly-once (Kafka 3.3+): source connectors can use transactions
    - Schema integration: auto-registers schemas in Schema Registry
    - Transforms (SMTs): lightweight message transforms (rename fields, route, filter)
    - Dead letter queue: failed records sent to DLQ topic (not block pipeline)

  Why not just write a consumer?
    Connect handles: offset management, parallelism, fault tolerance, monitoring,
    schema evolution, exactly-once — all out of the box. Writing a robust CDC
    consumer from scratch takes months. Debezium connector takes 10 minutes to configure.
```

### Kafka Streams vs. Flink — Stream Processing Comparison

```
Problem: Data in Kafka needs real-time transformation, aggregation, joins

Kafka Streams (library, runs in your JVM):
  ✓ No separate cluster needed — just a Java library in your app
  ✓ Exactly-once semantics (built on Kafka transactions)
  ✓ State stores (RocksDB-backed, changelog topic for recovery)
  ✓ Simple deployment (just deploy your app)
  ✗ JVM only
  ✗ Scaling = more app instances (each gets partitions)
  ✗ Limited windowing compared to Flink

  Example:
    StreamsBuilder builder = new StreamsBuilder();
    builder.stream("orders")
           .filter((k, v) -> v.amount > 100)
           .groupByKey()
           .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
           .count()
           .toStream()
           .to("high-value-order-counts");

Flink (separate cluster):
  ✓ Rich windowing, complex event processing
  ✓ Multi-language (Java, Scala, Python, SQL)
  ✓ Handles out-of-order events (watermarks)
  ✓ Can read from Kafka AND other sources
  ✗ Separate cluster to deploy and manage
  ✗ More complex operations
  ✗ Checkpointing overhead

  Choose Kafka Streams when: simple transforms, Kafka-to-Kafka, small team
  Choose Flink when: complex analytics, multi-source, large-scale aggregation
```

### Event Log vs Worker Queue: The Fundamental Design Choice

```
┌──────────────────────┬───────────────────────┬────────────────────────┐
│                      │ Event Log (Kafka) ⭐   │ Worker Queue           │
│                      │                        │ (RabbitMQ/SQS)         │
├──────────────────────┼───────────────────────┼────────────────────────┤
│ Mental model         │ Immutable event ledger │ Task inbox             │
│ Message after ACK    │ RETAINED ⭐            │ DELETED                │
│ Consumer state       │ Consumer tracks offset │ Broker tracks per-msg  │
│ Delivery model       │ Pull (consumer polls)  │ Push (broker → worker) │
│ Routing              │ Partition key only     │ Rich (exchanges) ⭐     │
│ Replay               │ Seek to any offset ⭐  │ Not possible           │
│ Ordering             │ Per-partition FIFO     │ Per-queue FIFO         │
│ Consumer parallelism │ Limited by partitions  │ Add workers freely ⭐   │
│ Storage growth       │ Unbounded (retention)  │ Bounded (transient)    │
│ Backpressure         │ Natural (pull-based) ⭐ │ Credit-based (push)    │
│ Multi-consumer       │ Consumer groups (free) │ Fanout exchange (copy) │
│ Use case             │ Event sourcing, CDC,   │ Job processing, RPC,   │
│                      │ analytics, streaming   │ task queues             │
│ Analogy              │ Git commit log         │ Email inbox            │
└──────────────────────┴───────────────────────┴────────────────────────┘

When interviewer says "Design a Message Broker / Event Streaming":
  → Think Kafka: append-only log, partitions, offsets, ISR, consumer groups, replay
  
When interviewer says "Design a Worker Queue / Task Queue":
  → Think RabbitMQ/SQS: delete on ACK, visibility timeout, exchanges, DLQ, priority
  → See 31_1_Distributed_Worker_Queue.md
```

### Message Delivery Semantics — Complete Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Delivery Guarantee Matrix                        │
├─────────────────┬────────────────┬────────────────┬────────────────────┤
│                 │ At-Most-Once   │ At-Least-Once  │ Exactly-Once       │
├─────────────────┼────────────────┼────────────────┼────────────────────┤
│ Producer        │ acks=0         │ acks=all       │ acks=all           │
│                 │ no retry       │ retries=MAX    │ enable.idempotence │
│                 │                │                │ transactional.id   │
├─────────────────┼────────────────┼────────────────┼────────────────────┤
│ Consumer        │ commit BEFORE  │ commit AFTER   │ read_committed +   │
│                 │ processing     │ processing     │ transactional      │
│                 │                │                │ offset commit      │
├─────────────────┼────────────────┼────────────────┼────────────────────┤
│ Data Loss?      │ YES            │ NO             │ NO                 │
│ Duplicates?     │ NO             │ YES            │ NO                 │
│ Use Case        │ Metrics, logs  │ Most workloads │ Financial, orders  │
│ Perf impact     │ Fastest        │ Fast           │ ~10-20% slower     │
└─────────────────┴────────────────┴────────────────┴────────────────────┘

Exactly-once end-to-end requires ALL THREE:
  1. Idempotent producer (PID + sequence → broker dedup)
  2. Transactional produce (atomic multi-partition writes)
  3. Consumer with read_committed (skip uncommitted/aborted messages)

Most teams choose at-least-once + idempotent consumer (simpler, nearly as safe)
```

