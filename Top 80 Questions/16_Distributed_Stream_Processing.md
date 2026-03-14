# 16. Design a Distributed Stream Processing System (Apache Flink)

---

## 1. Functional Requirements (FR)

- Process unbounded (infinite) event streams in real-time with sub-second latency
- Support **windowed aggregations**: tumbling, sliding, session, and global windows
- Handle **event-time semantics** with watermarks for out-of-order data
- Provide **exactly-once processing** guarantees even across failures
- Support **stateful operators**: keyed state (per-key counters, aggregates, ML features) persisted durably
- Stream-to-stream **joins**: join two event streams on a key within a time window
- Stream-to-table **enrichment joins**: enrich streaming events with slowly-changing dimension data
- SQL interface for analysts (Flink SQL) alongside programmatic DataStream API for engineers
- Connectors to sources (Kafka, Kinesis, files) and sinks (Kafka, Elasticsearch, PostgreSQL, S3, ClickHouse)
- Savepoints: manually triggered consistent snapshots for version upgrades and job migration
- Backpressure handling: slow operators should not cause data loss

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Event-to-output in < 100ms (p99) for real-time use cases
- **High Throughput**: 10M+ events/sec per job; cluster handles 100M+ events/sec
- **Exactly-Once**: No duplicates, no data loss — even during TaskManager crashes
- **Scalability**: Horizontal scaling — add TaskManagers to increase parallelism
- **Fault Tolerance**: Automatic recovery from TaskManager failures within seconds
- **Stateful at Scale**: Support TB-scale keyed state per job (via RocksDB)
- **Elastic Scaling**: Scale up/down without losing state (reactive mode)
- **Backpressure Handling**: Credit-based flow control — no data loss under slow consumers

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Events ingested / sec | | 50M |
| Avg event size | | 500 bytes |
| Ingestion throughput | 50M × 500B | 25 GB/sec |
| Stateful jobs | | 500 |
| Avg keyed state per job | | 100 GB |
| Total state | 500 × 100 GB | 50 TB |
| Checkpoints (every 1 min) | 50 TB incremental | ~500 GB incremental per checkpoint cycle |
| Checkpoint storage | S3 | ~10 TB (retained checkpoints) |
| TaskManagers | | 2,000 (8 cores, 32 GB each) |
| JobManagers | | 3 (HA quorum) |
| Network (inter-operator shuffle) | | ~10 GB/sec cluster-wide |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                         │
│                                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Kafka       │  │  Kinesis     │  │  Files (S3)  │  │  CDC         │       │
│  │  Topics      │  │  Streams     │  │  (bounded)   │  │  (Debezium)  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         └──────────────────┴─────────────────┴─────────────────┘               │
│                                      │                                         │
└──────────────────────────────────────│─────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────│─────────────────────────────────────────┐
│              FLINK CLUSTER            │                                        │
│                                       │                                        │
│  ┌────────────────────────────────────▼──────────────────────────────────────┐  │
│  │                    JOBMANAGER (Master)                                    │  │
│  │                                                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐      │  │
│  │  │  Active JobManager (Leader — elected via ZK or K8s HA)         │      │  │
│  │  │                                                                 │      │  │
│  │  │  Responsibilities:                                              │      │  │
│  │  │  ┌──────────────────┐  ┌───────────────────┐  ┌─────────────┐ │      │  │
│  │  │  │ Job Graph        │  │ Checkpoint        │  │ Resource    │ │      │  │
│  │  │  │ Compiler         │  │ Coordinator       │  │ Manager     │ │      │  │
│  │  │  │                  │  │                   │  │             │ │      │  │
│  │  │  │ - Parse user job │  │ - Trigger barrier │  │ - Request   │ │      │  │
│  │  │  │ - Optimize DAG   │  │   injection       │  │   slots from│ │      │  │
│  │  │  │ - Assign operator│  │ - Collect acks    │  │   TaskMgrs  │ │      │  │
│  │  │  │   parallelism    │  │ - Commit to S3    │  │ - Track slot│ │      │  │
│  │  │  │ - Schedule tasks │  │ - Trigger restore │  │   allocation│ │      │  │
│  │  │  │   to task slots  │  │   on failure      │  │ - Auto-scale│ │      │  │
│  │  │  └──────────────────┘  └───────────────────┘  └─────────────┘ │      │  │
│  │  │                                                                 │      │  │
│  │  │  ┌──────────────────┐  ┌───────────────────┐                   │      │  │
│  │  │  │ Dispatcher       │  │ REST API          │                   │      │  │
│  │  │  │ - Accept job     │  │ - Submit/cancel   │                   │      │  │
│  │  │  │   submissions    │  │   jobs            │                   │      │  │
│  │  │  │ - Maintain job   │  │ - View metrics    │                   │      │  │
│  │  │  │   history        │  │ - Trigger         │                   │      │  │
│  │  │  │                  │  │   savepoints      │                   │      │  │
│  │  │  └──────────────────┘  └───────────────────┘                   │      │  │
│  │  └─────────────────────────────────────────────────────────────────┘      │  │
│  │                                                                          │  │
│  │  Standby JobManagers: JM-2, JM-3 (ZK leader election for HA)           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                    TASKMANAGERS (Workers)                                │  │
│  │                                                                          │  │
│  │  ┌─────────────────────────────────────┐  ┌──────────────────────────┐  │  │
│  │  │  TaskManager 1 (4 task slots)       │  │  TaskManager 2           │  │  │
│  │  │                                     │  │  (4 task slots)          │  │  │
│  │  │  ┌──────────┐  ┌──────────┐         │  │                          │  │  │
│  │  │  │ Slot 1   │  │ Slot 2   │         │  │  ┌──────────┐           │  │  │
│  │  │  │ ┌──────┐ │  │ ┌──────┐ │         │  │  │ Slot 1   │           │  │  │
│  │  │  │ │Source│ │  │ │Source│ │         │  │  │ ┌──────┐ │           │  │  │
│  │  │  │ │ (P0) │ │  │ │ (P1) │ │         │  │  │ │Source│ │           │  │  │
│  │  │  │ └──┬───┘ │  │ └──┬───┘ │         │  │  │ │ (P2) │ │           │  │  │
│  │  │  │    │     │  │    │     │         │  │  │ └──┬───┘ │           │  │  │
│  │  │  │ ┌──▼───┐ │  │ ┌──▼───┐ │         │  │  │    │     │           │  │  │
│  │  │  │ │Map   │ │  │ │Map   │ │         │  │  │ ┌──▼───┐ │           │  │  │
│  │  │  │ │Filter│ │  │ │Filter│ │         │  │  │ │Map   │ │           │  │  │
│  │  │  │ └──┬───┘ │  │ └──┬───┘ │         │  │  │ │Filter│ │           │  │  │
│  │  │  │    │     │  │    │     │         │  │  │ └──┬───┘ │           │  │  │
│  │  │  │ ┌──▼───┐ │  │ ┌──▼───┐ │         │  │  │ ┌──▼───┐ │           │  │  │
│  │  │  │ │KeyBy │ │  │ │KeyBy │ │         │  │  │ │KeyBy │ │           │  │  │
│  │  │  │ │Window│ │  │ │Window│ │         │  │  │ │Window│ │           │  │  │
│  │  │  │ │Agg   │ │  │ │Agg   │ │         │  │  │ │Agg   │ │           │  │  │
│  │  │  │ └──┬───┘ │  │ └──┬───┘ │         │  │  │ └──┬───┘ │           │  │  │
│  │  │  │    │     │  │    │     │         │  │  │    │     │           │  │  │
│  │  │  │ ┌──▼───┐ │  │ ┌──▼───┐ │         │  │  │ ┌──▼───┐ │           │  │  │
│  │  │  │ │ Sink │ │  │ │ Sink │ │         │  │  │ │ Sink │ │           │  │  │
│  │  │  │ │(Kafka│ │  │ │(ES)  │ │         │  │  │ │(PG)  │ │           │  │  │
│  │  │  │ └──────┘ │  │ └──────┘ │         │  │  │ └──────┘ │           │  │  │
│  │  │  └──────────┘  └──────────┘         │  │  └──────────┘           │  │  │
│  │  │                                     │  │                          │  │  │
│  │  │  ┌──────────────────────────────┐   │  │  ┌──────────────┐       │  │  │
│  │  │  │  State Backend (RocksDB)    │   │  │  │ State Backend│       │  │  │
│  │  │  │  - Keyed state per operator │   │  │  │ (RocksDB)    │       │  │  │
│  │  │  │  - Spills to local SSD      │   │  │  │              │       │  │  │
│  │  │  │  - Incremental checkpoints  │   │  │  │              │       │  │  │
│  │  │  └──────────────────────────────┘   │  │  └──────────────┘       │  │  │
│  │  │                                     │  │                          │  │  │
│  │  │  Network: credit-based flow ctrl    │  │  Network buffers        │  │  │
│  │  │  Managed memory: off-heap for sort  │  │                          │  │  │
│  │  └─────────────────────────────────────┘  └──────────────────────────┘  │  │
│  │                                                                          │  │
│  │  ... TaskManager 3 ... TaskManager N (2000 total) ...                   │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                    CHECKPOINT STORAGE (S3 / HDFS)                        │  │
│  │                                                                          │  │
│  │  /checkpoints/job-abc123/                                                │  │
│  │    chk-42/                                                               │  │
│  │      _metadata           (checkpoint metadata — which operators, offsets) │  │
│  │      taskmanager-1/      (state handle files — RocksDB SST files)        │  │
│  │      taskmanager-2/                                                      │  │
│  │    chk-43/                                                               │  │
│  │      _metadata                                                           │  │
│  │      taskmanager-1/  (incremental: only NEW SST files since chk-42)      │  │
│  │                                                                          │  │
│  │  Retained checkpoints: last 3 (configurable)                             │  │
│  │  Savepoints: /savepoints/job-abc123/sp-2026-03-14/                       │  │
│  │    (manually triggered, portable across job versions)                     │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────│─────────────────────────────────────────┐
│                           DATA SINKS  │                                        │
│                                       │                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌▼─────────────┐  ┌──────────────┐       │
│  │  Kafka       │  │ Elasticsearch│  │  PostgreSQL  │  │  ClickHouse  │       │
│  │  (results)   │  │  (search)    │  │  (serving DB)│  │  (analytics) │       │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                                                │
│  ┌──────────────┐  ┌──────────────┐                                            │
│  │  S3 / Parquet│  │  Redis       │                                            │
│  │  (data lake) │  │  (cache)     │                                            │
│  └──────────────┘  └──────────────┘                                            │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### JobManager — The Brain

```
The JobManager is the control plane for a Flink cluster.

Key sub-components:
  1. Dispatcher: Accepts job submissions (JAR + job graph), starts JobMasters
  2. JobMaster (one per job): Manages the lifecycle of a single job
     - Compiles logical DAG → physical execution graph
     - Determines parallelism per operator
     - Requests task slots from ResourceManager
     - Deploys tasks to TaskManagers
     - Coordinates checkpointing
  3. ResourceManager: Manages TaskManager slots
     - In YARN mode: requests new containers from YARN
     - In K8s mode: creates new TaskManager pods
     - In standalone mode: uses pre-started TaskManagers
  4. Checkpoint Coordinator: Triggers periodic checkpoints, collects acknowledgments

HA mode:
  Multiple JobManagers deployed; ZooKeeper or K8s leader election
  Active JM dies → standby JM takes over → recovers from latest checkpoint
  Job metadata persisted in ZK/ConfigMap → not lost on JM crash
```

#### TaskManager — The Muscle

```
Each TaskManager is a JVM process with:
  - N task slots (typically = number of CPU cores)
  - Each slot runs a pipeline of operators (subtask chain)
  - Managed memory: off-heap memory pool for sorting, hashing, caching
  - Network buffers: credit-based flow control between operators
  - State backend: RocksDB (local SSD) for large state; HashMap for small state

Task slot isolation:
  - Slots share JVM but have isolated memory regions
  - One slot = one thread of parallelism
  - Multiple operators from SAME job can share a slot (operator chaining)
  - Operators from DIFFERENT jobs → different slots (resource isolation)
```

#### Operator Chaining (Key Optimization)

```
Without chaining:
  Source → [network] → Map → [network] → Filter → [network] → Sink
  Every arrow = network serialization/deserialization = expensive

With chaining:
  [Source → Map → Filter] → [network] → [Sink]
  Chained operators run in SAME thread, pass objects by reference
  → 10× throughput improvement for narrow operators

Chaining broken at:
  - keyBy() — requires network shuffle (hash partition by key)
  - rebalance() — round-robin redistribution
  - Different parallelism settings
```

---

### Core Concepts Deep Dive

#### Windowing (The Heart of Stream Processing)

```
TUMBLING WINDOW (fixed-size, non-overlapping):
  [0-5min] [5-10min] [10-15min] ...
  Each event belongs to exactly ONE window
  Use case: "Total orders per 5-minute interval"

  events.keyBy(e -> e.userId)
        .window(TumblingEventTimeWindows.of(Time.minutes(5)))
        .sum("amount");

SLIDING WINDOW (fixed-size, overlapping):
  [0-10min] [5-15min] [10-20min] ...  (size=10min, slide=5min)
  Each event belongs to MULTIPLE windows (size/slide = 2 windows)
  Use case: "Moving average of CPU usage over last 10 minutes, updated every 5 minutes"

SESSION WINDOW (gap-based, dynamic):
  Events clustered by activity; window closes after inactivity gap
  User A: [click, click, click] <5min gap> [click, click] <5min gap> [click]
  → 3 session windows
  Use case: "Session duration per user" (web analytics)

  events.keyBy(e -> e.userId)
        .window(EventTimeSessionWindows.withGap(Time.minutes(5)))
        .aggregate(new SessionDurationAggregator());

GLOBAL WINDOW (no time boundary):
  All events for a key in one window; fires on custom trigger
  Use case: "Emit result after every 100 events" (count-based trigger)
```

#### Event Time vs Processing Time vs Ingestion Time

```
PROCESSING TIME: wall clock of the machine processing the event
  ✅ Simple, no watermarks needed
  ❌ Non-deterministic: reprocessing gives different results
  ❌ Out-of-order events → wrong window assignment

EVENT TIME: timestamp embedded in the event by the producer
  ✅ Deterministic: reprocessing gives SAME results
  ✅ Handles out-of-order events correctly
  ❌ Requires watermarks (complexity)
  ❌ Needs to handle late data

INGESTION TIME: timestamp assigned when event enters Flink
  Middle ground: deterministic within Flink, but not across replays

RECOMMENDATION: Always use EVENT TIME for production systems
  Processing time only for debugging or when event timestamps are unavailable
```

#### Watermarks & Late Data (The Hardest Concept)

```
Problem:
  Events arrive out of order due to network delays, buffering, etc.
  Event with timestamp 10:05 might arrive AFTER event with timestamp 10:08
  How does Flink know when it's "safe" to close the 10:00-10:05 window?

Watermark = "assertion that no more events with timestamp ≤ W will arrive"
  Watermark W=10:05 → all events with t ≤ 10:05 have been seen
  → 10:00-10:05 window can fire

Watermark generation:
  Periodic: emit watermark = max_event_time - max_out_of_orderness
    WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5))
    → If latest event is t=10:10, watermark = 10:05
    → 5-second tolerance for out-of-order events

  Punctuated: emit watermark based on special events (e.g., heartbeat)

Late data (event arrives AFTER watermark has passed its timestamp):
  Option 1: DROP (default) — late events are lost
  Option 2: ALLOWED LATENESS — keep window open for additional time
    .window(TumblingWindows.of(5min))
    .allowedLateness(Time.minutes(1))
    → Window fires at watermark, but accepts late events for 1 more minute
    → Each late event triggers an UPDATED result
    
  Option 3: SIDE OUTPUT — route late events to a separate stream
    .sideOutputLateData(lateOutputTag)
    → Process late events differently (e.g., write to error topic)

The trade-off:
  Small max_out_of_orderness → low latency but drops more late events
  Large max_out_of_orderness → higher latency but fewer drops
  Typically 5-30 seconds for real-time; minutes for batch-like jobs
```

#### State Management (What Makes Flink Special)

```
KEYED STATE (per-key, per-operator):
  ValueState<T>:     single value per key  (e.g., last seen location)
  ListState<T>:      list of values per key (e.g., recent events)
  MapState<K,V>:     map per key (e.g., feature vector)
  ReducingState<T>:  pre-aggregated value (e.g., running sum)
  AggregatingState:  custom accumulator

Example: Count events per user in last 1 hour
  ValueState<Long> count;
  
  processElement(Event e) {
      count.update(count.value() + 1);
  }

STATE BACKENDS:

  HashMapStateBackend (in-memory):
    ✅ Fastest (HashMap in JVM heap)
    ❌ Limited by heap size (typically ≤ 10 GB)
    ❌ Full checkpoint (serialize entire HashMap to S3)
    Use for: small state, low-latency requirements

  EmbeddedRocksDBStateBackend (on-disk):
    ✅ Supports TB-scale state (spills to local SSD)
    ✅ Incremental checkpoints (only changed SST files to S3)
    ❌ ~10× slower per-access than HashMap (disk I/O + serialization)
    Use for: large state (counters for millions of keys, ML features)

  Production recommendation: ALWAYS use RocksDB
    - State will grow over time; you don't want to hit OOM in production
    - Incremental checkpoints: 100 GB state → ~5 GB per checkpoint (not 100 GB)
    - Tune: state.backend.rocksdb.block.cache-size = 256MB (per slot)
```

#### Checkpointing — Exactly-Once Guarantee (Deep Dive)

```
Flink's core innovation: Chandy-Lamport distributed snapshot algorithm (adapted)

How it works:
  1. Checkpoint Coordinator (in JobManager) triggers checkpoint N
  2. Sends BARRIER-N to all source operators
  3. Sources inject barrier into their output streams:
     
     [event] [event] [BARRIER-N] [event] [event] [BARRIER-N+1] ...
     
  4. When operator receives barrier from ALL input channels:
     a. Snapshot its state to S3 (async)
     b. Forward barrier downstream
     c. Continue processing (no pause!)
  
  5. When ALL operators have acknowledged → checkpoint N is COMPLETE
  6. Checkpoint N is the latest "consistent global snapshot"

Barrier alignment (exactly-once):
  Operator with 2 inputs:
    Input A: [event] [BARRIER-N] [event] ...
    Input B: [event] [event] [event] [BARRIER-N] ...
    
    Barrier arrives on A first → PAUSE reading from A, buffer events
    Continue reading from B until BARRIER-N arrives on B
    Both barriers received → snapshot state → forward barrier → resume A
    
    This ensures state snapshot is EXACTLY at the barrier boundary
    No event is counted in both checkpoint N-1 and checkpoint N

Unaligned checkpoints (Flink 1.11+):
  DON'T pause/buffer when barriers are skewed
  Instead: include in-flight data (buffered events) in the snapshot
  ✅ No processing pause → better latency under backpressure
  ❌ Larger checkpoint size (includes buffered data)

Checkpoint vs Savepoint:
  Checkpoint: automatic, periodic, for failure recovery, may not be portable
  Savepoint: manual, for job upgrades/migration, always portable
  
  Use savepoint when: changing job code, rescaling parallelism, migrating clusters
  Use checkpoint when: automatic failure recovery
```

---

## 5. APIs

### DataStream API (Java)

```java
// Fraud detection: flag users with >5 failed payments in 10 minutes
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStateBackend(new EmbeddedRocksDBStateBackend());
env.enableCheckpointing(60_000, CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30_000);

DataStream<PaymentEvent> events = env
    .addSource(new FlinkKafkaConsumer<>("payment-events", schema, kafkaProps))
    .assignTimestampsAndWatermarks(
        WatermarkStrategy.<PaymentEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
            .withTimestampAssigner((event, ts) -> event.getTimestamp())
    );

DataStream<FraudAlert> alerts = events
    .filter(e -> e.getStatus().equals("FAILED"))
    .keyBy(PaymentEvent::getUserId)
    .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(1)))
    .aggregate(new CountAggregator())
    .filter(count -> count.getValue() > 5)
    .map(count -> new FraudAlert(count.getUserId(), count.getValue()));

alerts.addSink(new FlinkKafkaProducer<>("fraud-alerts", alertSchema, kafkaProps,
    FlinkKafkaProducer.Semantic.EXACTLY_ONCE));

env.execute("Fraud Detection Job");
```

### Flink SQL

```sql
-- Same fraud detection in SQL
CREATE TABLE payment_events (
    user_id STRING,
    amount DECIMAL(10,2),
    status STRING,
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH ('connector' = 'kafka', 'topic' = 'payment-events', ...);

SELECT user_id, COUNT(*) AS fail_count
FROM payment_events
WHERE status = 'FAILED'
GROUP BY user_id, HOP(event_time, INTERVAL '1' MINUTE, INTERVAL '10' MINUTE)
HAVING COUNT(*) > 5;
```

### Job Management REST API

```
POST /jars/upload                       → Upload job JAR
POST /jars/{jar-id}/run                 → Submit job
GET  /jobs                              → List all jobs
GET  /jobs/{job-id}                     → Job status + metrics
POST /jobs/{job-id}/savepoints          → Trigger savepoint
PATCH /jobs/{job-id}                    → Cancel job
GET  /jobs/{job-id}/checkpoints         → Checkpoint history
GET  /taskmanagers                      → List TaskManagers + resources
```

---

## 6. Data Models

### State Backend — RocksDB Internal Structure

```
Per TaskManager, per keyed operator:

/tmp/flink-state/job-abc123/op-window-agg/
  ├── 000042.sst    (Sorted String Table — immutable, sorted key-value pairs)
  ├── 000043.sst
  ├── 000044.sst
  ├── MANIFEST      (tracks which SST files are current)
  ├── WAL           (write-ahead log for crash recovery)
  └── OPTIONS       (RocksDB configuration)

Key format:   [key_group (2 bytes)] [key_namespace] [user_key]
Value format: [serialized state value]

Key groups: Flink pre-assigns key ranges to task slots
  Slot 0: key_groups 0-31
  Slot 1: key_groups 32-63
  → Enables rescaling: redistribute key groups across slots
```

### Checkpoint Metadata

```json
{
  "checkpoint_id": 42,
  "job_id": "abc123",
  "timestamp": "2026-03-14T10:01:00Z",
  "duration_ms": 3500,
  "state_size_bytes": 5368709120,
  "is_incremental": true,
  "operator_states": [
    {
      "operator_id": "source-kafka",
      "subtask_states": [
        {"subtask": 0, "offset": {"partition-0": 5000150, "partition-1": 4200300}},
        {"subtask": 1, "offset": {"partition-2": 3100200, "partition-3": 2800100}}
      ]
    },
    {
      "operator_id": "window-aggregate",
      "subtask_states": [
        {"subtask": 0, "state_handle": "s3://checkpoints/job-abc/chk-42/tm-1/sst-files/"},
        {"subtask": 1, "state_handle": "s3://checkpoints/job-abc/chk-42/tm-2/sst-files/"}
      ]
    }
  ]
}
```

### Kafka Source Offset Tracking

```
Flink does NOT use Kafka's __consumer_offsets topic for exactly-once.

Instead:
  Kafka offsets stored IN Flink's checkpoint state
  On checkpoint: record {partition → offset} as part of operator state
  On recovery: seek Kafka consumer to checkpointed offsets
  → Offsets and operator state are atomically consistent
  → This is HOW exactly-once works end-to-end
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **TaskManager crash** | JobManager detects (heartbeat), restores job from latest checkpoint, redeploys tasks to surviving/new TMs |
| **JobManager crash** | Standby JM elected via ZK/K8s; recovers job graph + latest checkpoint from persistent storage |
| **State loss** | State is checkpointed to S3; on recovery, RocksDB state restored from checkpoint files |
| **Slow operator (backpressure)** | Credit-based flow control: upstream pauses sending when downstream buffers full → no data loss |
| **Out-of-order events** | Watermarks + allowed lateness → window fires correctly despite late arrivals |
| **Kafka offset drift** | Offsets stored IN checkpoint (not Kafka) → exactly-once from source to sink |
| **Checkpoint failure** | Retry; if checkpoint keeps failing (e.g., S3 down) → job continues but cannot recover from NEW failures |
| **Skewed keys** | Hot key → one subtask overloaded; mitigate with `.rebalance()` before keyBy or salted keys |

### Recovery Flow (Detailed)

```
1. TaskManager-3 crashes (OOM, hardware failure, etc.)
2. JobManager detects missing heartbeat (~30 seconds by default, configurable)
3. JobManager cancels ALL running tasks for the affected job
4. JobManager requests new task slots from ResourceManager
   (K8s: spin up new TM pod; YARN: request new container)
5. JobManager reads latest completed checkpoint metadata from S3
6. JobManager deploys ALL operators to task slots (potentially different TMs)
7. Each operator restores its state:
   - Source: seek Kafka to checkpointed offsets
   - Stateful operators: download RocksDB SST files from S3, rebuild state
   - Sink: (depends on sink — see "exactly-once sinks" below)
8. Processing resumes from checkpoint position

Recovery time:
  State download: 100 GB state ÷ 1 GB/s network = ~100 seconds
  Typical: 30-120 seconds total (dominated by state restore + container startup)
  
  Optimization: Local recovery (Flink 1.13+)
    TaskManager stores state locally on SSD (in addition to S3 checkpoint)
    If SAME TaskManager slot is re-used → skip S3 download → recover in seconds
```

### Exactly-Once Sinks (The Last Mile)

```
Problem: Flink processes event exactly once internally, but what about the SINK?
  If sink writes to Kafka/PG and then crashes before checkpoint → duplicate write on recovery

Solution depends on sink type:

  Kafka Sink: Two-Phase Commit
    1. On checkpoint, Flink pre-commits Kafka transaction (writes visible only to read_committed consumers)
    2. On checkpoint complete → Flink commits Kafka transaction → writes visible
    3. On failure → Flink aborts Kafka transaction → writes rolled back
    → True end-to-end exactly-once

  JDBC/PG Sink: Idempotent writes
    Use upsert (INSERT ON CONFLICT UPDATE) with deterministic primary key
    On recovery, re-processing produces same writes → idempotent → no duplicates

  S3/File Sink: Two-phase commit via rename
    1. Write to temp file: s3://output/.tmp/part-0
    2. On checkpoint complete → rename to final: s3://output/part-0
    3. On failure → temp files cleaned up → no duplicates
    
  Redis Sink: Idempotent SET operations (natural exactly-once)
  
  Elasticsearch Sink: Use document ID = deterministic hash of key + window
```

---

## 8. Additional Considerations & Deep Dives

### Flink vs Spark Structured Streaming vs Kafka Streams

| Aspect | Flink | Spark Structured Streaming | Kafka Streams |
|---|---|---|---|
| Processing model | True streaming (event-at-a-time) | Micro-batch (100ms+ intervals) | True streaming (event-at-a-time) |
| Latency | < 100ms | > 100ms (batch interval) | < 10ms |
| State management | RocksDB (TB-scale, incremental checkpoints) | State store (limited) | RocksDB (embedded) |
| Exactly-once | Barrier-based (Chandy-Lamport) | Micro-batch atomicity | Kafka transactions |
| Windowing | Rich (tumbling, sliding, session, custom) | Tumbling, sliding (limited session) | Tumbling, sliding, session |
| Event time | First-class (watermarks, late data) | Supported (watermarks) | Supported |
| Deployment | Standalone, YARN, K8s | Spark cluster (YARN, K8s) | Embedded in app (no cluster) |
| SQL support | Flink SQL (full SQL) | Spark SQL (mature) | KSQL / ksqlDB |
| Best for | Complex event processing, large state | Batch + stream unification | Simple stream processing inside services |
| Cluster overhead | Full cluster (JMs + TMs) | Full Spark cluster | None (library) |

```
When to choose Flink:
  - Sub-second latency required
  - Large state (GB-TB per operator)
  - Complex event processing (CEP)
  - Session windows needed
  - Late data handling is critical

When to choose Spark Streaming:
  - Already have Spark ecosystem
  - Batch + stream unification needed
  - Latency > 1 second is acceptable

When to choose Kafka Streams:
  - Simple transformations (filter, map, aggregate)
  - Want embedded processing (no separate cluster)
  - State fits in single machine
  - Operational simplicity is priority
```

### Backpressure — Credit-Based Flow Control

```
Problem: Operator B is slower than Operator A → what happens?

Push-based systems (Spark micro-batch): buffer grows → OOM
TCP-based: backpressure propagates via TCP but slow and imprecise

Flink's credit-based system:
  Each downstream operator advertises CREDITS (available buffer space) to upstream
  Upstream sends data ONLY if it has credits
  No credits → upstream stops sending → backpressure propagates instantly
  
  Visualization in Flink Web UI:
    Source: 100% busy (back-pressured — waiting for credits)
    Map: 50% busy (OK)
    Window Agg: 100% busy (← THE BOTTLENECK)
    Sink: 20% busy
    
  Fix the bottleneck: increase parallelism of Window Agg, optimize aggregation logic,
  or increase resources (memory/CPU) for that operator
```

### Rescaling (Changing Parallelism Without Data Loss)

```
Scenario: Job running with parallelism=8, need to scale to parallelism=16

1. Trigger savepoint: POST /jobs/{id}/savepoints
2. Cancel job
3. Modify job: set parallelism=16
4. Submit job from savepoint: POST /jars/{id}/run?savepointPath=s3://...

How state is redistributed:
  Flink uses KEY GROUPS (128 by default)
  
  Parallelism=8: each subtask owns 16 key groups
    Subtask 0: key groups 0-15
    Subtask 1: key groups 16-31
    ...
  
  Parallelism=16: each subtask owns 8 key groups
    Subtask 0: key groups 0-7   (subset of old subtask 0's state)
    Subtask 1: key groups 8-15  (other subset of old subtask 0's state)
    ...

  Key groups are the unit of state redistribution
  max_parallelism (key groups) is set at job creation and CANNOT change
  → Always set max_parallelism high enough (e.g., 128 or 256)
```

### Production Monitoring

```
CRITICAL metrics:
  - checkpointDuration: if growing → state too large or S3 slow
  - lastCheckpointSize: trending up → state leak?
  - numRecordsInPerSecond / numRecordsOutPerSecond: throughput health
  - currentInputWatermark: if stuck → source is not producing events
  - busyTimeMsPerSecond per operator: >900 = saturated
  - backPressuredTimeMsPerSecond: >0 = bottleneck detected

WARNING metrics:
  - numLateRecordsDropped: late data being discarded
  - numberOfFailedCheckpoints: checkpoint health
  - fullRestarts: job crash + recovery count
  - managed memory usage: approaching limit?

Alert rules:
  checkpoint_duration_p99 > 2 × checkpoint_interval → danger
  consumer_lag (Kafka source) growing → processing can't keep up
  watermark_lag > 5 minutes → source delay or stalled partition
```

### Common Anti-Patterns

```
1. Using processing time when event time is available
   → Non-deterministic results, wrong window assignment for late events

2. Not setting max_parallelism
   → Defaults to 128; if you later need parallelism=200, you can't rescale

3. Storing large objects in ValueState (instead of MapState)
   → Every access deserializes the ENTIRE object (use MapState for maps)

4. Not enabling incremental checkpoints with RocksDB
   → 100 GB state → 100 GB checkpoint every minute → S3 bill explodes
   → Incremental: only 1-5 GB of changed SST files per checkpoint

5. KeyBy on high-cardinality field without understanding skew
   → One key has 90% of events → one subtask does 90% of work → bottleneck
   → Fix: pre-aggregate before keyBy, or salt the key

6. Ignoring backpressure
   → "Job is running fine" but sink is 100% busy → latency is minutes, not seconds
   → Always check Flink Web UI backpressure tab
```
