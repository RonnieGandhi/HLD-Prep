# 85. Design a Time-Series Database

---

## 1. Functional Requirements (FR)

- Ingest time-stamped data points at high throughput (metrics, IoT, financial ticks)
- Write format: `(metric_name, tags, timestamp, value)` — e.g., `cpu.usage{host=web-01, dc=us-east} 1710410700 72.5`
- Query by metric name + tag filters + time range (e.g., "CPU usage for host=web-01 in last 1 hour")
- Aggregation queries: avg, sum, min, max, percentile over time windows (1m, 5m, 1h, 1d)
- Downsampling: automatically roll up high-resolution data to lower resolution for older data
- Retention policies: auto-delete data older than configured retention (7d raw, 90d hourly, 2y daily)
- Tag-based indexing: efficiently filter by any combination of tags
- Support rate, derivative, moving average, and other time-series functions
- Alerting integration: trigger alerts based on threshold queries

---

## 2. Non-Functional Requirements (NFRs)

- **High Write Throughput**: 10M+ data points/sec ingestion
- **Low Query Latency**: < 100ms for recent data queries; < 5s for large historical queries
- **Storage Efficiency**: 2–4 bytes per data point after compression (vs 16 bytes raw)
- **High Availability**: 99.99% for writes (metrics loss is critical)
- **Horizontal Scalability**: add nodes for more write throughput and storage
- **Durability**: no data loss for committed writes
- **Write-Optimized**: 95% writes, 5% reads (typical monitoring workload)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Data points / sec (ingestion) | 10M |
| Unique time series (cardinality) | 10M (unique metric+tag combinations) |
| Avg tags per series | 5–10 |
| Raw data point size | 16 bytes (8B timestamp + 8B value) |
| Compressed data point | 2–4 bytes (Gorilla/delta-delta compression) |
| Raw ingestion bandwidth | 10M × 16B = 160 MB/sec |
| Compressed storage / day | 10M × 4B × 86400 = ~3.5 TB/day |
| Compressed storage / year (with downsampling) | ~200 TB |
| Query rate | 10K queries/sec |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                                    │
│  Prometheus agents, Telegraf, StatsD, IoT devices, App metrics        │
│  Write protocol: HTTP, gRPC, UDP (StatsD), Prometheus remote_write    │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│                     INGESTION LAYER                                    │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Write Gateway (Stateless, horizontally scalable)           │       │
│  │  - Parse incoming data points                               │       │
│  │  - Validate metric names, tag cardinality                   │       │
│  │  - Assign to partition (by metric + tags hash)              │       │
│  │  - Write to WAL (local disk)                                │       │
│  │  - Batch and forward to storage nodes                       │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │                                              │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │  Kafka (Optional — Buffer & Replication)                    │       │
│  │  Topic: metrics-raw (partitioned by metric hash)            │       │
│  │  - Decouples ingestion from storage                         │       │
│  │  - Replay capability for reprocessing                       │       │
│  │  - Retention: 24h                                           │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │                                              │
└─────────────────────────│──────────────────────────────────────────────┘
                          │
┌─────────────────────────│──────────────────────────────────────────────┐
│                  STORAGE ENGINE LAYER                                   │
│                         │                                              │
│    ┌────────────────────▼─────────────────────┐                        │
│    │  Ingester / Storage Node                 │                        │
│    │  (Multiple nodes, each owns a shard)     │                        │
│    │                                          │                        │
│    │  ┌──────────────────────────────────┐    │                        │
│    │  │  In-Memory Buffer (Write Buffer) │    │                        │
│    │  │  - Current 2-hour block           │    │                        │
│    │  │  - Head chunk per series          │    │                        │
│    │  │  - Gorilla-compressed in memory   │    │                        │
│    │  │  - WAL for durability             │    │                        │
│    │  └──────────────┬───────────────────┘    │                        │
│    │                 │ (every 2 hours)         │                        │
│    │  ┌──────────────▼───────────────────┐    │                        │
│    │  │  Block Compactor                  │    │                        │
│    │  │  - Flush memory → immutable block │    │                        │
│    │  │  - Compact: merge small blocks    │    │                        │
│    │  │    into larger blocks             │    │                        │
│    │  │  - Create index for block         │    │                        │
│    │  └──────────────┬───────────────────┘    │                        │
│    │                 │                         │                        │
│    │  ┌──────────────▼───────────────────┐    │                        │
│    │  │  Block Storage (Local Disk/S3)    │    │                        │
│    │  │                                  │    │                        │
│    │  │  Block structure:                │    │                        │
│    │  │  /block-001/                     │    │                        │
│    │  │    meta.json    (time range,     │    │                        │
│    │  │                  series count)   │    │                        │
│    │  │    index         (inverted index │    │                        │
│    │  │                  label → series) │    │                        │
│    │  │    chunks/       (compressed     │    │                        │
│    │  │                  time-value data)│    │                        │
│    │  │    tombstones    (deletions)     │    │                        │
│    │  └──────────────────────────────────┘    │                        │
│    └──────────────────────────────────────────┘                        │
│                                                                        │
│    ┌──────────────────────────────────────────┐                        │
│    │  Downsampler (Background Process)        │                        │
│    │  - Read raw 15s data → compute 1min agg  │                        │
│    │  - Read 1min data → compute 1hour agg    │                        │
│    │  - Read 1hour data → compute 1day agg    │                        │
│    │  - Store downsampled data as separate    │                        │
│    │    time series                           │                        │
│    └──────────────────────────────────────────┘                        │
│                                                                        │
│    ┌──────────────────────────────────────────┐                        │
│    │  Retention Manager                       │                        │
│    │  - Delete blocks older than retention    │                        │
│    │  - Raw (15s): 7 days                     │                        │
│    │  - 1-min: 30 days                        │                        │
│    │  - 1-hour: 1 year                        │                        │
│    │  - 1-day: forever                        │                        │
│    └──────────────────────────────────────────┘                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                       QUERY LAYER                                      │
│                                                                        │
│    ┌──────────────────────────────────────────┐                        │
│    │  Query Frontend                          │                        │
│    │  - Parse PromQL / InfluxQL / SQL         │                        │
│    │  - Query splitting (time range → blocks) │                        │
│    │  - Results cache (Redis, query → result) │                        │
│    │  - Query deduplication                   │                        │
│    └──────────────────────┬───────────────────┘                        │
│                           │                                            │
│    ┌──────────────────────▼───────────────────┐                        │
│    │  Query Engine (Distributed)              │                        │
│    │  - Scatter query to relevant shards      │                        │
│    │  - Each shard scans local blocks         │                        │
│    │  - Merge results from all shards         │                        │
│    │  - Apply aggregation functions            │                        │
│    │  - Stream large results                   │                        │
│    └──────────────────────────────────────────┘                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Gorilla Compression (Facebook's Time-Series Compression)

```
Key insight: consecutive data points in a time series are similar

Timestamp compression (Delta-of-Delta):
  t0 = 1710410700                 → store full (64 bits)
  t1 = 1710410715 → delta = 15   → store delta (variable bits)
  t2 = 1710410730 → delta = 15, delta-of-delta = 0 → store 1 bit (same!)
  t3 = 1710410745 → delta = 15, delta-of-delta = 0 → store 1 bit
  
  Most points: 1 bit per timestamp (when scrape interval is regular)

Value compression (XOR):
  v0 = 72.5  → store full (64 bits)
  v1 = 72.8  → XOR with v0, store only changed bits
  v2 = 72.8  → XOR with v1 = 0 → store 1 bit (same value!)
  
  For slowly changing metrics: ~2 bits per value

Result: 16 bytes raw → 1.37 bytes average (12× compression)
         This is why TSDBs can handle 10M points/sec on modest hardware
```

### Block-Based Storage (Prometheus TSDB Model)

```
Time axis divided into blocks:
  
  |----2h----|----2h----|----2h----|----2h----|
  | Block 1  | Block 2  | Block 3  | HEAD     |
  | (immut)  | (immut)  | (immut)  | (memory) |
  
  HEAD block: in-memory, accepts writes
  After 2h: HEAD → immutable block on disk → new HEAD created
  
  Compaction: merge adjacent blocks
    Block 1 + Block 2 → Block A (4h)
    Block A + Block 3 → Block B (6h)
    ... up to max block duration (e.g., 48h)
  
  Benefits of block design:
  1. Writes go to single in-memory block → fast
  2. Immutable blocks → easy to cache, replicate, backup
  3. Deletion: delete entire block (no random deletes)
  4. Each block has its own index → parallel scan
```

---

## 5. APIs

### Write API

```
POST /api/v1/write
Content-Type: application/x-protobuf  (or text/plain for Prometheus format)

# Prometheus exposition format:
cpu_usage{host="web-01", dc="us-east", env="prod"} 72.5 1710410700
memory_used_bytes{host="web-01", dc="us-east"} 8589934592 1710410700
http_requests_total{method="GET", path="/api", status="200"} 15234 1710410700

# Batch: thousands of samples per request
```

### Query API (PromQL-style)

```
GET /api/v1/query_range
  ?query=avg(cpu_usage{dc="us-east"}) by (host)
  &start=1710324300
  &end=1710410700
  &step=60

Response:
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {"host": "web-01"},
        "values": [[1710324300, "72.5"], [1710324360, "73.1"], ...]
      },
      {
        "metric": {"host": "web-02"},
        "values": [[1710324300, "65.2"], [1710324360, "64.8"], ...]
      }
    ]
  }
}
```

### Admin APIs

```
POST /api/v1/admin/retention   → Configure retention policies
POST /api/v1/admin/downsample  → Configure downsampling rules
GET  /api/v1/admin/status      → Storage node health, disk usage
POST /api/v1/admin/compact     → Trigger manual compaction
DELETE /api/v1/admin/series     → Delete series matching selector
```

---

## 6. Data Models

### On-Disk Block Format

```
block-ulid/
├── meta.json
│   {
│     "ulid": "01F8VYXK5...",
│     "minTime": 1710324300000,
│     "maxTime": 1710331500000,
│     "stats": { "numSamples": 5000000, "numSeries": 100000 },
│     "compaction": { "level": 1, "sources": ["01F8VYX..."] }
│   }
│
├── index
│   Inverted index:  label_name=label_value → sorted list of series IDs
│   Posting lists:   host="web-01" → [series_1, series_5, series_42]
│   Series metadata: series_id → {metric_name, labels, chunk_refs}
│
├── chunks/
│   000001  (max 512MB per chunk file)
│   000002
│   Each chunk: Gorilla-compressed (timestamp[], value[]) for one series
│
└── tombstones
    Records of deleted series/time ranges (applied during compaction)
```

### In-Memory (HEAD Block)

```
HashMap<SeriesID, HeadChunk>

HeadChunk:
  labels: {__name__="cpu_usage", host="web-01", dc="us-east"}
  samples: Gorilla-compressed buffer (appends only)
  min_time: 1710410700
  max_time: 1710410715
  num_samples: 2

Inverted Index (in-memory):
  label_name=label_value → BitSet of series IDs
  Fast intersection for multi-label queries:
    host="web-01" AND dc="us-east" → bitwise AND of posting lists
```

### WAL (Write-Ahead Log)

```
WAL segments (sequential files, append-only):
  segment-00001:
    [record_type=SERIES, series_id=1, labels={...}]
    [record_type=SAMPLES, series_id=1, t=1710410700, v=72.5]
    [record_type=SAMPLES, series_id=1, t=1710410715, v=73.1]
    [record_type=SAMPLES, series_id=2, t=1710410700, v=8589934592]
    ...
  
  On restart: replay WAL → rebuild HEAD block
  WAL truncated: after HEAD block is flushed to persistent block
  Checkpoint: periodic snapshot of in-memory state → truncate older WAL segments
```

### Downsampled Data

```
Raw (15s resolution):     cpu_usage{host="web-01"} → [72.5, 73.1, 72.8, ...]
1-min aggregation:        cpu_usage:1m_avg{host="web-01"} → [72.8]
                          cpu_usage:1m_max{host="web-01"} → [73.1]
                          cpu_usage:1m_min{host="web-01"} → [72.5]
                          cpu_usage:1m_count{host="web-01"} → [4]
1-hour aggregation:       cpu_usage:1h_avg{host="web-01"} → [71.2]

Query engine automatically selects resolution based on query time range:
  Last 1h → raw 15s data
  Last 24h → 1-min data
  Last 30d → 1-hour data
  Last 1y → 1-day data
```

---

## 7. Fault Tolerance & Deep Dives

### High Cardinality Problem

```
Problem: Tags with unbounded values (user_id, request_id, IP address)
  user_id has 100M unique values → 100M unique time series
  Memory: 100M × 1KB overhead = 100 GB just for index
  Index lookup slows down for all queries

Solutions:
1. Cardinality limits: reject metrics with > 100K unique tag values
2. Pre-aggregation at ingestion: aggregate per-user metrics into percentiles
3. Separate high-cardinality data into dedicated store (ClickHouse/Druid)
4. Tag value allowlisting: only index known tag values, hash others

Detection:
  - Monitor unique series count per metric
  - Alert when cardinality growth rate > threshold
  - Dashboard showing top-K highest cardinality metrics
```

### Write Path Durability

```
1. Client sends batch of samples → Write Gateway
2. Write Gateway appends to local WAL (fsync) → ACK to client
3. WAL entries batched → sent to storage node(s)
4. Storage node appends to its WAL + in-memory HEAD
5. If storage node dies before flush → replay WAL on restart

Replication for HA:
  Option A: Write to N replicas independently (Prometheus federation)
    - Simple, but slight gaps if one replica misses scrape
  Option B: Kafka buffer → multiple consumers (ingester replicas)
    - Better: no data loss, replay capability
  Option C: Storage-level replication (like Cortex/Thanos using object storage)
    - Blocks uploaded to S3 → infinitely durable + queryable
```

### Query Performance Optimization

```
1. Block pruning: skip blocks whose time range doesn't overlap query
2. Series filtering: use inverted index to find matching series
   before reading any chunks
3. Chunk caching: LRU cache of recently accessed chunks in memory
4. Query splitting: 30-day query → 15 × 2-day sub-queries → parallel
5. Step alignment: align query step to storage resolution
   (don't query 15s resolution when step is 1h)
6. Results caching: cache query results in Redis (keyed by query + time range)
7. Materialized views: pre-compute common aggregations

Benchmark: 1M series, 6h query, 1m step
  Without optimization: 30 seconds
  With all optimizations: 200ms
```

### Multi-Tenancy & Isolation

```
Problem: SaaS TSDB (like Datadog, Grafana Cloud) serving 1000s of tenants

Isolation strategies:
1. Tenant ID as top-level label: __tenant__="acme-corp"
   - Simple, but noisy neighbor: one tenant's high cardinality affects all
2. Per-tenant ingester: dedicated in-memory buffer per tenant
   - Better isolation, but more resource overhead
3. Per-tenant storage: separate blocks/S3 prefixes per tenant
   - Best isolation, easy billing, but complex routing

Rate limiting:
  - Per-tenant write rate limit (samples/sec)
  - Per-tenant series cardinality limit
  - Per-tenant query rate limit and query timeout
```

### Comparison: TSDB vs General-Purpose DB

```
Why not just use PostgreSQL for time-series data?

PostgreSQL:
  - Row-based storage → 100+ bytes per data point
  - B-tree index → write amplification
  - No native compression for time-series patterns
  - No automatic downsampling or retention
  - 10K writes/sec max (with index maintenance)

TSDB:
  - Column/chunk-based storage → 2-4 bytes per data point (50× less)
  - Append-only writes → no index maintenance during write
  - Gorilla compression → 12× compression
  - Built-in downsampling and retention
  - 10M writes/sec

When PostgreSQL IS appropriate:
  - Low cardinality + low volume (< 1K writes/sec)
  - Need joins with relational data
  - TimescaleDB extension bridges the gap
```

---

## 8. Additional Considerations

### Prominent TSDB Architectures

```
Prometheus (single-node):
  - Pull-based scraping, local block storage
  - Great for single-cluster monitoring
  - Not horizontally scalable

Cortex / Mimir (distributed Prometheus):
  - Ingesters with in-memory chunks → flush to S3
  - Queriers read from ingesters + S3
  - Compactor merges blocks in S3
  - Horizontally scalable

InfluxDB:
  - TSM engine (like LSM tree optimized for time-series)
  - InfluxQL or Flux query language
  - Built-in retention policies and continuous queries

TimescaleDB:
  - PostgreSQL extension with hypertables (auto-partitioned by time)
  - Full SQL support
  - Best when you need SQL joins + time-series

ClickHouse (column store):
  - Not pure TSDB but excellent for time-series analytics
  - MergeTree engine with time-based partitioning
  - SQL interface, great for ad-hoc queries
```

### Out-of-Order Writes

```
Problem: Data arrives out of order (e.g., batch upload, delayed agents)

Solutions:
1. Reject out-of-order (Prometheus default until v2.39):
   - Simple, but loses data from delayed sources
   
2. Accept with limited out-of-order window (Prometheus v2.39+):
   - Accept writes within 1h of HEAD max time
   - Buffer out-of-order samples in separate memory structure
   - Merge during compaction

3. Full out-of-order support (InfluxDB, VictoriaMetrics):
   - Accept any timestamp
   - More complex merging during compaction
   - Higher memory usage

Recommendation: Allow configurable out-of-order window (e.g., 30 min)
```

### Exemplars and Traces Correlation

```
Modern TSDBs support exemplars: link metric data points to trace IDs

cpu_usage{host="web-01"} 95.0 1710410700 # trace_id=abc123

When user sees CPU spike on dashboard:
  → Click on data point
  → Follow trace_id link to distributed tracing system
  → See exact request causing the spike

This bridges metrics ↔ traces observability
```

