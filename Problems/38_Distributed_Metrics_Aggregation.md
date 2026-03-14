# 38. Design a Distributed Metrics Aggregation System

---

## 1. Functional Requirements (FR)

- **Ingest metrics**: Receive time-series metrics (CPU, memory, latency, custom counters) from thousands of hosts
- **Aggregate**: Sum, avg, p50/p95/p99, min, max across dimensions (host, service, region)
- **Downsample**: Auto-downsample old data (1s → 1m → 1h → 1d resolution over time)
- **Query**: "Average CPU across region=us-east for last 6 hours at 1-minute granularity"
- **Alerting integration**: Feed metrics to alerting rules
- **Dashboarding**: Low-latency queries for Grafana-style dashboards

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: 10M+ data points/sec ingested
- **Low Latency Queries**: Dashboard queries in < 500 ms
- **Long Retention**: Raw data 30 days, downsampled data 2 years
- **Scalability**: Thousands of metrics × thousands of hosts = billions of time series
- **High Availability**: 99.99% — metrics drive alerting
- **Cardinality Handling**: Support high-cardinality labels without explosion

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Hosts reporting | 500K |
| Metrics per host | 200 |
| Total unique time series | 100M |
| Data points / sec | 10M |
| Data point size | 16 bytes (timestamp + value) |
| Ingestion throughput | 160 MB/s |
| Raw storage / day | 13.8 TB |
| With downsampling (30d raw + 2yr aggregated) | ~500 TB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Metric Sources (500K Hosts)                      │
│                                                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ Host Agent │  │ Host Agent │  │ K8s Pods   │  │ Cloud        │  │
│  │ (StatsD /  │  │ (Prom      │  │ (OTel      │  │ Services     │  │
│  │  OTel)     │  │  Exporter) │  │  sidecar)  │  │ (CloudWatch) │  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └──────┬───────┘  │
└────────┼────────────────┼────────────────┼───────────────┼──────────┘
         │  Push           │  Pull          │  Push         │  Pull
         ▼                 ▼                ▼               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Distributor / Ingestion Gateway                    │
│  (Stateless, horizontally scaled)                                    │
│                                                                       │
│  1. Validate: reject malformed metrics, enforce label limits          │
│  2. Rate limit: per-tenant quota (max series, max samples/sec)       │
│  3. Hash: consistent hash(metric_labels) → select target Ingester   │
│  4. Replicate: write to N ingesters (RF=3) for durability            │
│  5. HA dedup: if two distributors write same sample → dedup at query │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
┌────────▼────────┐ ┌───────▼───────┐ ┌───────▼───────┐
│  Ingester 1     │ │  Ingester 2   │ │  Ingester N   │
│                 │ │               │ │               │
│  In-memory:     │ │               │ │               │
│  ┌────────────┐ │ │               │ │               │
│  │ Active     │ │ │  Same         │ │  Same         │
│  │ Series Map │ │ │  structure    │ │  structure    │
│  │ (hash →    │ │ │               │ │               │
│  │  time-ser.)│ │ │               │ │               │
│  ├────────────┤ │ │               │ │               │
│  │ WAL        │ │ │               │ │               │
│  │ (Write-    │ │ │               │ │               │
│  │  Ahead Log)│ │ │               │ │               │
│  ├────────────┤ │ │               │ │               │
│  │ 2-hour     │ │ │               │ │               │
│  │ chunks     │ │ │               │ │               │
│  │ (in-mem)   │ │ │               │ │               │
│  └─────┬──────┘ │ │               │ │               │
│        │ flush  │ │               │ │               │
│        │ every  │ │               │ │               │
│        │ 2 hours│ │               │ │               │
└────────┼────────┘ └───────┬───────┘ └───────┬───────┘
         │                  │                  │
         ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  Object Store (S3 / GCS)                             │
│                                                                       │
│  Block format (Prometheus TSDB):                                     │
│  ┌────────────────────┐                                              │
│  │ Block: 2h window   │  Each block:                                 │
│  │ ├── chunks/        │  • chunks: compressed time-series data       │
│  │ │   ├── 000001     │  • index: inverted index (label → series)    │
│  │ │   └── 000002     │  • meta.json: block metadata                 │
│  │ ├── index          │                                              │
│  │ └── meta.json      │  Stored as immutable objects in S3           │
│  └────────────────────┘                                              │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼───────┐ ┌──▼──────────┐  │
     │  Compactor     │ │  Querier    │  │
     │                │ │             │  │
     │  Merges small  │ │  Receives   │  │
     │  blocks into   │ │  PromQL     │  │
     │  larger ones   │ │  query      │  │
     │                │ │             │  │
     │  Downsamples:  │ │  Fan-out:   │  │
     │  raw → 5m avg  │ │  query ALL  │  │
     │  5m → 1h avg   │ │  ingesters  │  │
     │  1h → 1d avg   │ │  (in-mem)   │  │
     │                │ │  + S3 blocks│  │
     │  Dedup:        │ │             │  │
     │  remove HA     │ │  Merge &    │  │
     │  duplicates    │ │  dedup      │  │
     └────────────────┘ │  results    │  │
                        └──────┬──────┘  │
                        ┌──────▼──────┐  │
                        │  Query      │  │
                        │  Frontend   │  │
                        │ (Caching)   │  │
                        └──────┬──────┘  │
                        ┌──────▼──────┐  │
                        │  Grafana    │  │
                        └─────────────┘  │
                                         │
                        ┌────────────────▼┐
                        │  Store Gateway  │ ← Serves older blocks from S3
                        │  (caches block  │   without loading into Ingesters
                        │   index in mem) │
                        └─────────────────┘
```

This architecture follows the **Mimir / Cortex / Thanos** pattern — the industry standard for scalable Prometheus-compatible metrics.

### Component Deep Dive

#### Push vs Pull Ingestion

| Model | How | Pros | Cons |
|---|---|---|---|
| **Pull (Prometheus)** | Central server scrapes targets every 15s | Server controls pace; discovers targets via service discovery | Doesn't scale well to millions of targets; need federation |
| **Push (StatsD/OTEL)** ⭐ | Agents push metrics to gateway | Scales to millions of agents; works for ephemeral workloads | Must handle ingestion spikes; agents need gateway address |

**Recommendation**: Push for large-scale (>100K hosts). Pull for smaller deployments or Kubernetes-native (Prometheus + federation).

#### Time-Series Database — Why Specialized?

```
Regular DB (PostgreSQL):
  INSERT INTO metrics (ts, host, metric, value) VALUES (now(), 'h1', 'cpu', 85.2);
  At 10M inserts/sec → PostgreSQL dies
  Query "avg cpu for last 6 hours across 1000 hosts" → full table scan → minutes

Time-Series DB (VictoriaMetrics / InfluxDB / Mimir):
  Optimized for:
  1. High write throughput (append-only, LSM-tree / columnar)
  2. Time-range queries (data organized by time)
  3. Compression (delta-of-delta for timestamps, XOR for values → 10× compression)
  4. Downsampling (automatic resolution reduction for old data)
  
  Result: 10M inserts/sec, sub-second queries on billions of data points
```

#### Downsampling Strategy

```
Resolution tiers:
  Raw (10s interval):   retained for 30 days    → full precision
  1-minute avg:         retained for 6 months   → 6× reduction
  1-hour avg:           retained for 2 years    → 360× reduction
  1-day avg:            retained for 5 years    → 8,640× reduction

Background downsampler:
  Every hour: aggregate raw data from last hour → 1-min resolution
  Every day: aggregate 1-min data from last day → 1-hour resolution
  
  Storage savings: 30 days raw (13.8 TB/day × 30 = 414 TB)
                   + 6 months 1-min (414 TB / 6 = 69 TB)
                   + 2 years 1-hour (69 TB / 60 = 1.15 TB)
                   Total: ~485 TB (vs 10 PB without downsampling)
```

---

## 5. APIs

### Write Metrics (Prometheus Remote Write)
```http
POST /api/v1/write
Content-Type: application/x-protobuf
Body: TimeSeries { labels: [{name:"__name__", value:"cpu_usage"}, {name:"host", value:"h1"}], samples: [{timestamp: 1710320000, value: 85.2}] }
```

### Query (PromQL)
```http
GET /api/v1/query?query=avg(cpu_usage{region="us-east"})&time=1710320000
GET /api/v1/query_range?query=rate(http_requests_total[5m])&start=1710316400&end=1710320000&step=60
```

---

## 6. Data Model

### Time-Series Storage Format

```
Series: cpu_usage{host="h1", region="us-east", env="prod"}
  
Stored as:
  Series ID: hash(sorted_labels) = 0xABCD1234
  
  Chunk (2-hour block):
    Timestamps: [T0, T0+10s, T0+20s, ...] → delta-of-delta encoded
    Values:     [85.2, 85.3, 84.9, ...] → XOR float encoding
    
  Compression ratio: ~1.5 bytes per data point (vs 16 bytes raw)
```

### Label Index (Inverted Index for Query)
```
Label "region=us-east" → Series IDs: [0xABCD1234, 0xDEF56789, ...]
Label "host=h1" → Series IDs: [0xABCD1234, ...]

Query: cpu_usage{region="us-east", host="h1"}
  → Intersect postings lists → Series ID 0xABCD1234
  → Fetch chunks for that series in the queried time range
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Ingestion spike** | Kafka buffers; auto-scale ingestion workers |
| **TSDB node failure** | Replicated storage (Mimir uses object store + replicated ingesters) |
| **Data loss** | Write-ahead log on ingesters; replay on restart |
| **High cardinality** | Limit labels (reject metrics with >100K unique series per metric name) |
| **Query overload** | Query concurrency limits; query timeout (30s max); caching layer |
| **Clock skew** | Accept data within ±5 minute window; reject future timestamps |

### Race Conditions and Scale Challenges

#### 1. Cardinality Explosion — The #1 Operational Problem

```
Scenario: Developer adds a label "request_id" to their HTTP metrics
  http_requests_total{service="api", method="GET", request_id="uuid-1"} 1
  http_requests_total{service="api", method="GET", request_id="uuid-2"} 1
  ...
  
  Each unique request_id creates a NEW time series.
  10K requests/sec × 86400 sec/day = 864M unique time series per day
  
  Impact:
    - Ingester memory: Each series kept in-memory → OOM kill
    - Index size: Inverted index grows unbounded → slow queries
    - Storage: Millions of tiny time series with 1 data point each
    - Query: "avg(http_requests_total{service='api'})" scans 864M series → TIMEOUT
  
  Solutions:
    1. Label cardinality limits ⭐: Reject series if a metric has > 10K unique values for any label
       Return HTTP 429 with error: "cardinality limit exceeded for label request_id"
    
    2. Relabeling at ingestion: Drop or hash high-cardinality labels
       metric_relabel_configs:
         - source_labels: [request_id]
           action: drop  # Remove label entirely
    
    3. Active series limit per tenant: Max 1M active series per tenant
       If exceeded → oldest series evicted or new series rejected
    
    4. Monitoring: Dashboard showing top 10 metrics by cardinality
       Alert if any metric exceeds 50K series → investigate immediately
```

#### 2. Aggregation Across Distributed Nodes — Split-Brain Metrics

```
Metric: http_requests_total{service="api"}
  Node A (us-east) sees: 5000 requests/min
  Node B (us-west) sees: 3000 requests/min
  
  Query: "total http requests across all regions"
  Answer: 8000/min (must aggregate across nodes)
  
  Challenge: If nodes have different retention or different scrape intervals,
  aggregation gives inconsistent results.
  
  Solution: Federated querying (Thanos / Mimir)
    Querier contacts ALL storage nodes → merges results → deduplicates
    Dedup by: if two series have same labels + same timestamp → keep one
    
    But what if us-east is 30 seconds behind us-west?
    → Querier applies time alignment: round to 1-minute boundaries
    → Slight inaccuracy but globally consistent view
```

#### 3. Scrape Interval vs Event Frequency — Aliasing Problem

```
Prometheus scrapes every 15 seconds.
A spike lasts 5 seconds (between scrapes) → INVISIBLE!

  Actual CPU:  |    ___
               |   /   \
               |__/     \__________
               0   5   10  15  20  (seconds)
  
  Scraped at T=0 and T=15:
               |                    
               |__________________
               0        15         (seconds)
  
  The spike is completely missed! This is signal aliasing.
  
  Solutions:
    1. Push-based metrics for critical signals (StatsD → Kafka → immediate ingestion)
    2. Shorter scrape interval (5s instead of 15s → 3× more data, 3× more storage)
    3. Summary/Histogram: Service pre-aggregates over the scrape interval
       http_request_duration_seconds{quantile="0.99"} already reflects intra-interval data
    4. Recording rules: Pre-compute expensive queries and store results as new metrics
```

#### 4. Metric Staleness — Phantom Time Series

```
A pod is killed → it stops reporting metrics.
But the last data point is still in storage: cpu_usage{pod="pod-abc"} = 0.85
  
  Query: "avg(cpu_usage)" still includes the dead pod's last value
  → The average is wrong because it includes a stale data point
  
  Prometheus solution: Staleness marker
    If no new sample received for 5 minutes → series is "stale"
    Stale series excluded from queries
    
  Challenge: If scrape interval is 15s and pod dies right after scrape,
  staleness kicks in after 5 minutes → 5 minutes of stale data in queries
  
  Mitigation: Use shorter staleness timeout for ephemeral workloads (K8s pods)
  Or: Service sends an explicit "going down" metric on SIGTERM
```

---

## 8. Deep Dive: Engineering Trade-offs

### VictoriaMetrics vs Prometheus vs InfluxDB vs Mimir

| Feature | Prometheus | VictoriaMetrics ⭐ | InfluxDB | Mimir (Grafana) |
|---|---|---|---|---|
| **Scalability** | Single node | Clustered | Clustered (paid) | Horizontally scalable |
| **Ingestion** | Pull only | Push + Pull | Push | Push (Prometheus remote write) |
| **Storage** | Local disk | Local + S3 | Local | Object store (S3) |
| **Query language** | PromQL | MetricsQL (PromQL superset) | InfluxQL / Flux | PromQL |
| **Cost** | Free | Free (open-source) | Free/Paid | Free |
| **Operations** | Simple | Simple | Moderate | Complex (microservices) |

**Choose VictoriaMetrics** for high-performance, low-ops self-hosted metrics.
**Choose Mimir** for massive scale with S3 backend and Grafana ecosystem.
**Choose Prometheus** for small-medium Kubernetes deployments.

### Percentile Calculation at Scale — The Hard Problem

```
"What is the p99 latency of our API across all servers?"

Why this is hard:
  Server A has latency distribution: [10, 12, 15, 20, 25, 100, 500] ms
  Server B has latency distribution: [8, 11, 14, 18, 22, 90, 450] ms
  
  You CANNOT compute global p99 by averaging p99 of each server!
    Server A p99 = 500ms, Server B p99 = 450ms
    Average p99 = 475ms → WRONG
    True global p99 requires merging ALL individual data points → 
      sort all 14 values, pick the 99th percentile
  
  At 100K requests/sec across 1000 servers → cannot store all individual values.

Solutions:
  1. Histogram buckets ⭐ (Prometheus approach):
     Pre-defined buckets: [5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 5s]
     Count how many requests fell into each bucket
     Merge across servers: SUM the counts per bucket
     Estimate percentile: linear interpolation between buckets
     
     Error: ~5% depending on bucket granularity
     More buckets = more accurate = more storage
  
  2. T-Digest (approximate):
     Streaming algorithm that maintains a compact summary
     Mergeable: combine T-Digests from multiple servers
     Accuracy: ~1% error for extreme percentiles (p99, p999)
     Used by: Elasticsearch, Apache Spark
  
  3. DDSketch (Datadog approach):
     Deterministic error bound (e.g., ±1%)
     Mergeable across any number of sources
     Constant memory regardless of data volume

Recommendation: Histogram buckets for Prometheus-compatible systems.
T-Digest or DDSketch for custom aggregation pipelines.
```

### Write-Ahead Log (WAL) — Ingester Crash Recovery

```
Ingester receives 10M data points/sec, holds 2 hours in memory.
  If Ingester crashes → up to 2 hours of data LOST (in-memory only)

WAL (Write-Ahead Log) ⭐:
  Every sample received → immediately appended to WAL on disk
  WAL is sequential write → 100+ MB/s easily, negligible latency overhead
  
  On crash recovery:
    1. New Ingester starts
    2. Reads WAL from disk → replays all samples into memory
    3. Resumes normal operation
    4. Old WAL segments are deleted after successful flush to S3
  
  WAL segment rotation: New segment every 128 MB or 2 minutes
  WAL retention: Keep segments until the corresponding block is flushed to S3
  
  Recovery time: Replaying a 2-hour WAL takes ~30 seconds (sequential reads)
  Data loss window: 0 (WAL captures everything between flushes)
```

### Multi-Tenancy — Isolating Metrics Across Teams

```
Problem: Team A's runaway cardinality explosion affects Team B's query performance.
  All teams share the same Ingester pool → one bad tenant = all tenants slow.

Solutions (Mimir approach):
  1. Per-tenant rate limits:
     Max samples/sec: 100K per tenant
     Max active series: 1M per tenant
     Max labels per series: 30
     Max label value length: 2048 characters
  
  2. Tenant-aware sharding:
     Distributor hashes (tenant_id, metric_labels) → Ingester
     Different tenants' data on different Ingesters (when possible)
     Query isolation: tenant A's slow query doesn't block tenant B
  
  3. Per-tenant usage tracking:
     Store: samples_ingested, active_series, query_count per tenant
     Bill: based on active series × retention period
     Alert: if tenant approaches limits → notify team before rejection
  
  4. Query cost estimation:
     Before executing: estimate series to scan
     If estimate > 10M series → reject with error
     Prevents accidental "SELECT * FROM all_metrics" queries
```

