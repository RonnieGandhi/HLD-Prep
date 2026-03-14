# 31.2 Design a Distributed Message Broker (Kafka-style)

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

---

## 8. Additional Considerations & Deep Dives

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
