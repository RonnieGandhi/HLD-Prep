# Kafka & Event Streaming — Deep Dive

Core concept tested in HLD rounds: "Why Kafka and not a simple queue? How do you guarantee exactly-once processing? What happens when a consumer falls behind?"

---

## 1. Kafka Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     KAFKA CLUSTER (3+ Brokers)                   │
│                                                                   │
│  Topic: order-events (3 partitions, RF=3)                        │
│                                                                   │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │ Partition 0    │  │ Partition 1    │  │ Partition 2    │     │
│  │ Leader: B1     │  │ Leader: B2     │  │ Leader: B3     │     │
│  │ Replicas: B2,B3│  │ Replicas: B1,B3│  │ Replicas: B1,B2│     │
│  │                │  │                │  │                │     │
│  │ [0] order_1    │  │ [0] order_2    │  │ [0] order_3    │     │
│  │ [1] order_4    │  │ [1] order_5    │  │ [1] order_6    │     │
│  │ [2] order_7    │  │ [2] order_8    │  │ [2] order_9    │     │
│  │ ...            │  │ ...            │  │ ...            │     │
│  └────────────────┘  └────────────────┘  └────────────────┘     │
│                                                                   │
│  ZooKeeper / KRaft: broker metadata, partition leadership        │
└──────────────────────────────────────────────────────────────────┘

Producer: writes to a topic (key → partition via hash)
Consumer Group: each partition assigned to exactly ONE consumer in the group
  → Parallelism = number of partitions
  → Adding consumers beyond partition count = idle consumers
```

### Key Concepts

```
Topic: Named stream of events (like a table, but append-only)
Partition: Ordered, immutable sequence of events within a topic
  → Partition is the unit of parallelism
Offset: Sequential ID per message within a partition (0, 1, 2, ...)
Consumer Group: Set of consumers that cooperatively consume a topic
  Each partition → exactly one consumer in the group
  Each consumer → zero or more partitions
Replication Factor: Number of copies of each partition across brokers
  RF=3: data on 3 brokers. Survives loss of 2 brokers.

Message Key:
  Messages with the SAME key always go to the SAME partition
  → Ordering guaranteed per key
  → Example: key=user_id ensures all events for a user are in order
  
  No key: round-robin distribution (no ordering guarantee)
```

---

## 2. Why Kafka Over a Simple Queue (RabbitMQ/SQS)?

```
Feature                    Kafka                    RabbitMQ/SQS
──────────────────────────────────────────────────────────────────
Message retention          Time-based (7 days)      Until consumed (deleted)
Replay                     ✓ (seek to any offset)   ✗ (consumed = gone)
Consumer groups            ✓ (multiple independent)  Limited
Ordering                   Per-partition             Per-queue
Throughput                 1M+ msg/sec              10K-50K msg/sec
Consumer model             Pull (consumer pulls)     Push (broker pushes)
Use case                   Event streaming, log      Task queue, work
                           aggregation, CDC          distribution, RPC

Choose Kafka when:
  • Multiple consumers need the same events (fan-out to search, analytics, cache)
  • You need replay (reprocess events after a bug fix)
  • High throughput (100K+ msg/sec)
  • Event sourcing / CQRS

Choose RabbitMQ/SQS when:
  • Simple task queue (one consumer per message)
  • Need routing complexity (exchange → queues by pattern)
  • Low throughput, simpler operations
  • Request-reply pattern
```

---

## 3. Producer Guarantees

### Acknowledgment Modes

```
acks=0 (fire-and-forget):
  Producer doesn't wait for any acknowledgment
  ✓ Lowest latency
  ✗ Messages can be lost (broker crash before write)
  Use for: metrics, analytics (loss acceptable)

acks=1 (leader acknowledgment):
  Producer waits for partition leader to confirm write
  ✓ Good balance of safety and performance
  ✗ If leader crashes before replicating → message lost
  Use for: most applications

acks=all (all replicas acknowledge) ⭐:
  Producer waits for ALL in-sync replicas (ISR) to confirm
  ✓ No data loss (if min.insync.replicas ≥ 2)
  ✗ Higher latency (wait for slowest replica)
  Use for: financial events, critical data
  
  Must configure together:
    acks = all
    min.insync.replicas = 2  (at least 2 replicas must be in sync)
    → If only 1 replica in sync → producer gets error (better than losing data)
```

### Idempotent Producer (Exactly-Once Semantics on Produce Side)

```
Problem: Network retry → duplicate messages
  Producer sends message → broker writes it → ACK gets lost → producer retries
  → Same message written TWICE

Solution: enable.idempotence = true
  Producer assigns a sequence number to each message
  Broker tracks: {producer_id, partition} → last_sequence_number
  If duplicate sequence received → broker ignores it (idempotent)
  
  ✓ Exactly-once per partition (no duplicates)
  ✓ Automatic (no application code changes)
```

---

## 4. Consumer Guarantees

### Offset Management

```
Each consumer tracks which messages it has processed via "committed offset":

  Partition 0: [0][1][2][3][4][5][6][7][8][9]
                              ▲           ▲
                     committed offset   latest offset
                     (processed up to)  (last written)

  Committed offset = 4 means: messages 0-4 are processed
  On restart: consumer resumes from offset 5

Commit strategies:
  Auto-commit (enable.auto.commit = true):
    Offsets committed every 5 seconds automatically
    ✗ If consumer crashes between commit intervals → reprocess messages
    ✗ If consumer commits before processing → message lost
    
  Manual commit ⭐:
    Consumer explicitly commits after successful processing
    
    while true:
      records = consumer.poll(100ms)
      for record in records:
        process(record)          # process first
      consumer.commitSync()      # then commit
    
    ✗ If crash after process but before commit → reprocess on restart
    → At-least-once delivery (idempotent processing handles duplicates)
```

### At-Least-Once vs At-Most-Once vs Exactly-Once

```
At-Most-Once:
  Commit offset BEFORE processing
  If crash during processing → message skipped (lost)
  ✓ No duplicates
  ✗ Message loss possible
  Use for: non-critical analytics, metrics

At-Least-Once ⭐ (most common):
  Commit offset AFTER processing
  If crash after processing but before commit → reprocess on restart
  ✓ No message loss
  ✗ Duplicates possible → make processing idempotent
  Use for: most applications (with idempotent consumers)

Exactly-Once (Kafka Transactions):
  Producer + consumer in same Kafka transaction
  read-process-write pattern within Kafka (e.g., Kafka Streams)
  
  ✓ No loss, no duplicates within Kafka
  ✗ Only works for Kafka-to-Kafka pipelines
  ✗ Higher latency (transaction overhead)
  Use for: stream processing (Kafka Streams, ksqlDB)

For Kafka-to-external-system (DB, API):
  Exactly-once is NOT possible with Kafka alone
  → Use at-least-once + idempotent processing:
    Insert with ON CONFLICT DO NOTHING (dedup by event_id)
    Or: track processed event_ids in a dedup table
```

---

## 5. Partition Strategy

### How Many Partitions?

```
Rule of thumb:
  target_throughput / per_consumer_throughput = min_partitions
  
  Example: 100K msg/sec target, each consumer handles 10K msg/sec
  → 10 partitions minimum
  
  Add headroom: 10 × 1.5 = 15 partitions
  
  Too few partitions → can't scale consumers
  Too many partitions → more overhead (leader elections, rebalancing)
  
  Practical guidance:
    Low throughput topic (< 10K msg/sec): 3-6 partitions
    Medium throughput (10K-100K msg/sec): 12-30 partitions
    High throughput (100K+ msg/sec): 30-100 partitions
    
  Can increase partitions later but CANNOT decrease
  Increasing partitions breaks key-based ordering (keys redistribute)
```

### Partition Key Selection

```
Key determines which partition a message goes to:
  partition = hash(key) % num_partitions

GOOD keys:
  user_id → all events for a user in order
  order_id → all events for an order in order
  device_id → all telemetry from a device in order

BAD keys:
  null (no key) → round-robin, no ordering
  timestamp → all messages in same millisecond to same partition (hot partition)
  country → "US" partition gets 50% of traffic (hot partition)

Hot Partition Problem:
  One partition gets disproportionate traffic → one consumer overloaded
  Solution: Add randomness to key: key = user_id + random(0, 5)
    → Spreads user's events across 5 partitions
    ✗ Loses per-user ordering (acceptable if ordering within user not needed)
```

---

## 6. Consumer Group Rebalancing

```
When a consumer joins/leaves/crashes, partitions are reassigned:

Before (3 consumers, 6 partitions):
  C1: [P0, P1]
  C2: [P2, P3]
  C3: [P4, P5]

C3 crashes → rebalance:
  C1: [P0, P1, P4]
  C2: [P2, P3, P5]

Problem: During rebalance, ALL consumers STOP processing ("stop-the-world")
  Duration: seconds to minutes (depends on group size)
  
  → Messages pile up during rebalance → latency spike

Solutions:
  1. Cooperative Rebalancing (Incremental):
     Only affected partitions are revoked/assigned
     Other consumers continue processing uninterrupted
     partition.assignment.strategy = CooperativeStickyAssignor

  2. Static Group Membership:
     group.instance.id = "consumer-1" (unique per consumer)
     On transient failure: consumer rejoins without triggering full rebalance
     session.timeout.ms = 300000 (5 min) → wait longer before declaring dead

  3. Avoid frequent scaling:
     Don't auto-scale consumers aggressively
     Size consumer group for peak + headroom
```

---

## 7. Kafka Failure Scenarios

### Broker Failure

```
Broker B2 crashes:
  Partitions where B2 was leader → leadership moves to ISR member on another broker
  Failover time: < 10 seconds (ZooKeeper/KRaft detects and elects new leader)
  Producers/consumers briefly see errors → retry → reconnect to new leader
  
  No data loss if: acks=all + min.insync.replicas ≥ 2
```

### Consumer Lag (Consumer Falls Behind)

```
Consumer can't keep up with producer rate:
  Offset falls further behind → "consumer lag" grows
  Eventually: messages expire (retention period) before consumed → DATA LOSS

Detection:
  Monitor: kafka.consumer.lag (offset gap per partition)
  Alert if: lag > threshold (e.g., > 100K messages or > 5 minutes behind)

Solutions:
  1. Scale consumers (add more to the consumer group, up to partition count)
  2. Increase consumer throughput (batch processing, faster processing logic)
  3. Increase retention.ms (give more buffer time)
  4. Backpressure: if downstream is slow, buffer in Kafka (that's what it's for)
```

### Message Too Large

```
Default max message size: 1 MB
  Broker: message.max.bytes = 1048576
  Producer: max.request.size = 1048576

If you need larger messages:
  Option 1: Increase limits (up to ~10 MB practical max)
  Option 2 ⭐: Claim-check pattern
    Store large payload in S3/blob store
    Send S3 reference in Kafka message
    Consumer fetches from S3 when processing
    ✓ Keeps Kafka lean
    ✓ No message size issues
```

---

## 8. Common Kafka Patterns in System Design

### Event-Driven Microservices

```
Order Service → Kafka (order-events) → Inventory Service (reserve stock)
                                     → Payment Service (charge card)
                                     → Notification Service (send email)
                                     → Analytics Service (track metrics)

Each downstream service is a separate consumer group:
  → Each gets ALL events independently
  → Services can be added/removed without changing Order Service
  → Replay: new Search Service can consume from beginning to build its index
```

### CQRS (Command Query Responsibility Segregation)

```
Write Path: API → PostgreSQL (source of truth, normalized)
  ↓ CDC (Debezium)
  ↓ Kafka (change events)
  ↓
Read Path: Kafka Consumer → Elasticsearch (denormalized, search-optimized)
                          → Redis (cached, fast reads)
                          → ClickHouse (analytics-optimized)

Each read store is optimized for its specific query pattern
Kafka ensures all read stores eventually converge with write store
```

### Dead Letter Queue (DLQ)

```
Consumer can't process a message (malformed data, downstream failure):

  while true:
    record = consumer.poll()
    try:
      process(record)
      consumer.commit()
    except RetriableError:
      retry(record, max_attempts=3)
    except NonRetriableError:
      publish_to_dlq(record)  # send to dead-letter topic
      consumer.commit()       # don't block the partition

DLQ topic: order-events-dlq
  Monitored by ops team
  Manually inspected and reprocessed after fixing the issue
  
  ✓ Bad messages don't block good ones
  ✓ No data loss (bad messages preserved for investigation)
```

---

## 9. Kafka vs Alternatives — Decision Guide

```
Need                              → Technology
──────────────────────────────────────────────────
High-throughput event streaming    → Kafka
Simple task queue                  → SQS / RabbitMQ
Real-time pub/sub (low latency)   → Redis Pub/Sub
Exactly-once stream processing    → Kafka Streams / Flink
Serverless event routing          → AWS EventBridge
IoT device messaging              → MQTT (Mosquitto)
Cross-region event replication    → Kafka MirrorMaker 2

Kafka sizing rules of thumb:
  3 brokers minimum (production)
  Replication factor = 3
  Retention = 7 days (default, increase for replay needs)
  Disk: plan for 2× retention × daily_ingestion_rate
  Memory: 6 GB heap per broker + OS page cache (the more RAM, the better reads)
```

