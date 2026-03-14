# 16. Design a Distributed Stream Processing System (Kafka)

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

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: 1M+ messages/sec per broker (GBs per second)
- **Low Latency**: End-to-end < 10 ms (producer → consumer) for real-time use cases
- **Durability**: No message loss even during broker failures
- **Scalability**: Horizontal scaling — add brokers to increase throughput
- **Fault Tolerant**: Survive broker failures without data loss or service disruption
- **Ordering Guarantees**: Per-partition ordering
- **Retention**: Configurable retention (time-based or size-based)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Messages / sec (system-wide) | 10M |
| Avg message size | 1 KB |
| Throughput | 10 GB/sec |
| Retention | 7 days |
| Storage per day | 10 GB/s × 86400 = ~864 TB |
| Storage for 7 days | ~6 PB |
| Replication factor 3 | ~18 PB total storage |
| Brokers (10 TB each) | ~1800 brokers |
| Topics | 10,000 |
| Partitions | 500,000 |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Producer 1  │  │  Producer 2  │  │  Producer N  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                  │                 │
       └──────────────────┼─────────────────┘
                          │
                ┌─────────▼──────────┐
                │   Kafka Cluster    │
                │                    │
                │  ┌──────────────┐  │
                │  │   Broker 1   │  │     ┌──────────────┐
                │  │  P0(L) P3(F) │  │     │  ZooKeeper / │
                │  └──────────────┘  │     │  KRaft       │
                │  ┌──────────────┐  │     │  (Metadata)  │
                │  │   Broker 2   │  │◄───▶│              │
                │  │  P1(L) P0(F) │  │     └──────────────┘
                │  └──────────────┘  │
                │  ┌──────────────┐  │
                │  │   Broker 3   │  │
                │  │  P2(L) P1(F) │  │
                │  └──────────────┘  │
                │                    │
                └─────────┬──────────┘
                          │
       ┌──────────────────┼─────────────────┐
       │                  │                 │
┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
│ Consumer     │  │ Consumer     │  │ Consumer     │
│ Group A      │  │ Group A      │  │ Group B      │
│ (Instance 1) │  │ (Instance 2) │  │ (Instance 1) │
└──────────────┘  └──────────────┘  └──────────────┘

L = Leader, F = Follower (replica)
```

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
```

#### Broker — Storage Engine

**How messages are stored on disk**:
- Each partition is a directory on the broker's filesystem
- Inside: a sequence of **segment files** (default 1 GB each)
- Each segment has:
  - `.log` file: Actual messages (append-only)
  - `.index` file: Sparse index mapping offset → file position
  - `.timeindex` file: Sparse index mapping timestamp → offset

```
/data/user-events-0/
  00000000000000000000.log      (segment 1: offsets 0-999999)
  00000000000000000000.index
  00000000000001000000.log      (segment 2: offsets 1000000-1999999)
  00000000000001000000.index
```

**Why append-only is fast**:
- Sequential writes → fully utilizes disk bandwidth (no seeks)
- OS page cache: recently written data is already in memory → consumers reading recent messages hit page cache
- **Zero-copy transfer**: `sendfile()` system call → data goes from disk → kernel buffer → NIC socket directly (no user-space copy)

#### Replication — ISR (In-Sync Replicas)

Each partition has:
- **Leader**: Handles all reads and writes
- **Followers**: Replicate the leader's log

```
Partition 0:
  Leader:    Broker 1  (offset: 150)
  Follower:  Broker 2  (offset: 148)  ← In ISR (within lag threshold)
  Follower:  Broker 3  (offset: 150)  ← In ISR
```

**ISR (In-Sync Replica Set)**:
- Followers that are "caught up" with the leader (within `replica.lag.time.max.ms`)
- A message is "committed" only when ALL ISR replicas have it
- If `min.insync.replicas=2` and one ISR falls behind → writes are still accepted
- If ISR drops below `min.insync.replicas` → producer gets `NotEnoughReplicas` error

**Leader Election**: If leader dies, one of the ISR followers is elected as new leader (via ZooKeeper or KRaft)

#### Producer — Write Path

```
Producer.send(topic, key, value):
  1. Serialize key and value
  2. Determine partition: hash(key) % num_partitions
  3. Find leader broker for that partition (from metadata cache)
  4. Send to leader broker
  5. Leader appends to local log
  6. Followers fetch from leader and append to their logs
  7. Once all ISR replicas have the message → committed
  8. Leader sends ack to producer

Producer acks config:
  acks=0:  Fire and forget (fastest, may lose messages)
  acks=1:  Leader ack (moderate, loses if leader dies before replication)
  acks=all: All ISR ack (slowest, no data loss)
```

**Producer Batching**: Collect messages for 5-50ms → send as a batch → amortize network overhead. Configurable via `batch.size` and `linger.ms`

**Compression**: Batch compressed with `snappy`, `lz4`, or `zstd` → reduces network and storage by 3-5×

#### Consumer — Read Path

```
Consumer.poll():
  1. Send FetchRequest to leader of each assigned partition
  2. Include: partition, offset, max_bytes
  3. Leader reads from log (usually page cache hit) → zero-copy to socket
  4. Consumer receives messages, processes them
  5. Consumer commits offset (to __consumer_offsets topic)
```

**Consumer Groups**:
- Consumers with the same `group.id` form a group
- Each partition assigned to exactly ONE consumer in the group
- If consumers < partitions → some consumers handle multiple partitions
- If consumers > partitions → excess consumers are idle
- **Rebalancing**: When a consumer joins/leaves, partitions are redistributed (cooperative sticky assignment)

**Offset Management**:
- Consumer tracks its position (offset) in each partition
- Offset committed to a special internal topic: `__consumer_offsets`
- On restart, consumer reads last committed offset → resumes from there

#### ZooKeeper / KRaft (Metadata Management)

**ZooKeeper** (legacy):
- Stores: broker list, topic configs, partition-to-leader mapping, consumer group metadata
- Leader election for partitions
- **Problem**: ZooKeeper is a bottleneck for large clusters (100K+ partitions)

**KRaft** (Kafka Raft — replacement):
- Kafka's own Raft-based consensus for metadata
- Metadata stored in a special Kafka topic (`__cluster_metadata`)
- No external dependency
- Better scalability (millions of partitions)

---

## 5. APIs

### Producer API
```java
ProducerRecord<String, String> record = 
    new ProducerRecord<>("user-events", userId, jsonPayload);

producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        log.info("Sent to partition {} offset {}", 
                 metadata.partition(), metadata.offset());
    }
});
```

### Consumer API
```java
consumer.subscribe(List.of("user-events"));
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record.key(), record.value());
    }
    consumer.commitSync();
}
```

### Admin API
```java
admin.createTopics(List.of(
    new NewTopic("user-events", 12, (short) 3)  // 12 partitions, RF=3
));
```

---

## 6. Data Model

### Message Format (on disk)

```
┌──────────────────────────────────────────────┐
│ Offset (8 bytes)                             │
│ Message Size (4 bytes)                       │
│ CRC32 (4 bytes)                              │
│ Magic Byte (1 byte) — format version         │
│ Attributes (1 byte) — compression, timestamp │
│ Timestamp (8 bytes)                          │
│ Key Length (4 bytes)                          │
│ Key (variable)                               │
│ Value Length (4 bytes)                        │
│ Value (variable)                             │
│ Headers (variable) — key-value pairs         │
└──────────────────────────────────────────────┘
```

### Segment Index Entry

```
Offset      Position (file byte offset)
0           0
100         16384
200         32768
...
```
- Sparse index (every 100th offset) → binary search for lookup

### Consumer Offset Storage

```
Topic: __consumer_offsets (50 partitions)
Key:   {group_id, topic, partition}
Value: {offset, metadata, timestamp}
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Broker failure** | ISR follower elected as new leader; producer/consumer redirect automatically |
| **Message loss** | `acks=all` + `min.insync.replicas=2` → committed messages survive any single broker failure |
| **Consumer failure** | Consumer group rebalances; another consumer takes over the partition |
| **Disk failure** | Data replicated on 2 other brokers; replace disk and re-replicate |
| **Network partition** | ISR mechanism prevents split-brain; leader only accepts writes if enough ISR replicas |
| **Unclean leader election** | Disabled by default (`unclean.leader.election.enable=false`) — prevents out-of-sync replica from becoming leader (avoids data loss) |

### Exactly-Once Semantics (EOS)

**Problem**: Producer retries can cause duplicates; consumer crash after processing but before committing offset → reprocessing.

**Solution — Idempotent Producer**:
- Producer assigned a `ProducerID` and each message has a `SequenceNumber`
- Broker deduplicates: if it sees same PID + SeqNum, it skips the write
- Guarantees: no duplicates within a single partition

**Solution — Transactional Producer**:
- Atomic writes across multiple partitions and topics
- `producer.beginTransaction()` → send messages → `producer.commitTransaction()`
- If producer crashes before commit → transaction aborted, messages discarded

**Solution — Consumer Read-Committed**:
- Consumer configured with `isolation.level=read_committed`
- Only sees messages from committed transactions

---

## 8. Additional Considerations

### Kafka vs. Other Message Systems

| Feature | Kafka | RabbitMQ | AWS SQS |
|---|---|---|---|
| Model | Log (pull) | Queue (push) | Queue (pull) |
| Ordering | Per-partition | Per-queue | Best-effort |
| Throughput | 1M+ msg/s | 50K msg/s | Unlimited (managed) |
| Replay | Yes (offset seek) | No (consumed = gone) | No |
| Persistence | Disk (retention) | Memory + disk | Managed |
| Use case | Event streaming, analytics | Task queues, RPC | Simple async |

### Log Compaction
- Alternative to time-based retention
- Keep only the **latest value** for each key
- Use case: Changelog/CDC — topic represents current state of a table
- Background compaction thread scans old segments, removes superseded records

### Tiered Storage
- **Problem**: Storing months of data on broker SSDs is expensive
- **Solution**: Move old segments to object storage (S3)
- Recent data (hot): On local SSD (fast)
- Historical data (cold): On S3 (cheap, slightly slower)
- Transparent to consumers — they read from either tier

### Monitoring
- **Broker metrics**: Under-replicated partitions, request latency, disk usage, network I/O
- **Producer metrics**: Record send rate, request latency, retry rate
- **Consumer metrics**: Consumer lag (most important!), commit rate, rebalance frequency
- **Alert on**: Consumer lag > threshold, under-replicated partitions > 0, ISR shrink

