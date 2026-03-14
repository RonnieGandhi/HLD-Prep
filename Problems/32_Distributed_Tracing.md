# 32. Design a Distributed Tracing System (like Jaeger / Zipkin)

---

## 1. Functional Requirements (FR)

- **Instrument services**: Inject trace context (trace_id, span_id, parent_span_id) into all inter-service calls
- **Collect spans**: Each service emits spans representing a unit of work (start time, duration, metadata)
- **Correlate traces**: Reconstruct the full request flow across microservices using trace_id
- **Visualize traces**: Waterfall/timeline view of spans showing latency breakdown
- **Search traces**: By trace_id, service name, operation, duration, status, tags
- **Service dependency graph**: Auto-discover and visualize service-to-service dependencies
- **Alerting**: Alert on latency anomalies (p99 > threshold), error rate spikes

---

## 2. Non-Functional Requirements (NFRs)

- **Low Overhead**: Tracing must add < 1% CPU and < 5% latency overhead to instrumented services
- **High Throughput**: Ingest 1M+ spans/sec (every RPC generates a span)
- **Sampling**: Support adaptive sampling (not every request needs full tracing)
- **Scalability**: Handle thousands of microservices, billions of spans/day
- **Durability**: Traces retained for 7-14 days for debugging
- **Low Latency Query**: Search traces by ID in < 1 second
- **Open Standards**: OpenTelemetry compatible (vendor-neutral)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Microservices | 2,000 |
| Requests / sec (system-wide) | 500K |
| Avg spans per trace | 10 |
| Total spans / sec | 5M (before sampling) |
| Sampling rate | 10% (1 in 10 traces) |
| Sampled spans / sec | 500K |
| Avg span size | 500 bytes |
| Ingestion throughput | 250 MB/s |
| Storage / day | 21 TB |
| Retention (14 days) | ~300 TB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Instrumented Services (Data Plane)                  │
│                                                                        │
│  ┌──────────────┐         ┌──────────────┐       ┌──────────────┐    │
│  │  Service A   │  HTTP   │  Service B   │ gRPC  │  Service C   │    │
│  │  (API GW)    │────────▶│  (User Svc)  │──────▶│  (DB Proxy)  │    │
│  │              │         │              │       │              │    │
│  │ ┌──────────┐ │         │ ┌──────────┐ │       │ ┌──────────┐ │    │
│  │ │OTel SDK  │ │         │ │OTel SDK  │ │       │ │OTel SDK  │ │    │
│  │ │          │ │         │ │          │ │       │ │          │ │    │
│  │ │• Create  │ │         │ │• Extract │ │       │ │• Extract │ │    │
│  │ │  root    │ │         │ │  context │ │       │ │  context │ │    │
│  │ │  span    │ │         │ │• Create  │ │       │ │• Create  │ │    │
│  │ │• Inject  │ │         │ │  child   │ │       │ │  child   │ │    │
│  │ │  context │ │         │ │  span    │ │       │ │  span    │ │    │
│  │ │• Export  │ │         │ │• Add tags│ │       │ │• Add db  │ │    │
│  │ │  to agent│ │         │ │• Export  │ │       │ │  tags    │ │    │
│  │ └────┬─────┘ │         │ └────┬─────┘ │       │ └────┬─────┘ │    │
│  └──────┼───────┘         └──────┼───────┘       └──────┼───────┘    │
│         │  UDP/gRPC              │                       │           │
└─────────┼────────────────────────┼───────────────────────┼───────────┘
          │                        │                       │
          ▼                        ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Agent Layer (per-host sidecar)                     │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ OTel Agent   │  │ OTel Agent   │  │ OTel Agent   │              │
│  │ (Host 1)     │  │ (Host 2)     │  │ (Host 3)     │              │
│  │              │  │              │  │              │              │
│  │ • Batch spans│  │ • Batch spans│  │ • Batch spans│              │
│  │   (5s / 500) │  │   (5s / 500) │  │   (5s / 500) │              │
│  │ • Head-based │  │ • Head-based │  │ • Head-based │              │
│  │   sampling   │  │   sampling   │  │   sampling   │              │
│  │ • Compress   │  │ • Compress   │  │ • Compress   │              │
│  │ • Retry queue│  │ • Retry queue│  │ • Retry queue│              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
└─────────┼──────────────────┼──────────────────┼─────────────────────┘
          │   gRPC/OTLP      │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Collector Tier (Centralized, Auto-scaled)               │
│                                                                      │
│  ┌─────────���────────────────────────────────────────────┐           │
│  │          OTel Collector Pool (stateless)              │           │
│  │                                                       │           │
│  │  Receive → Tail-Based Sampling → Export               │           │
│  │           ┌─────────────────┐                         │           │
│  │           │ Sampling Buffer │ ← Hold spans 30s        │           │
│  │           │ (in-memory or   │   waiting for trace     │           │
│  │           │  Redis-backed)  │   completion            │           │
│  │           └─────────┬───────┘                         │           │
│  │                     │ Decision:                       │           │
│  │                     │ • Any span has error? → KEEP    │           │
│  │                     │ • Duration > p99? → KEEP        │           │
│  │                     │ • Random 10%? → KEEP            │           │
│  │                     │ • Otherwise → DROP              │           │
│  └─────────────────────┼────────────────────────────────┘           │
└─────────────────────────┼────────────────────────────────────────────┘
                          │
                   ┌──────▼──────┐
                   │    Kafka    │  ← Durable buffer between collection and storage
                   │ Topics:     │
                   │ • spans-raw │
                   │ • spans-    │
                   │   sampled   │
                   └──────┬──────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
  ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
  │  Span       │  │ Dependency  │  │  Metrics   │
  │  Ingester   │  │ Graph       │  │  Aggregator│
  │             │  │ Builder     │  │  (Flink)   │
  │ Batch writes│  │             │  │            │
  │ to storage  │  │ Extracts    │  │ • p50/p99  │
  │ (bulk index)│  │ parent→child│  │   latency  │
  │             │  │ relationships│ │ • Error    │
  │             │  │             │  │   rates    │
  └──────┬──────┘  └──────┬──────┘  └─────┬──────┘
         │                │                │
  ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
  │             │  │   Redis     │  │ ClickHouse │
  │ Hot Store   │  │ (Adj. List) │  │ (Metrics   │
  │(Elasticsearch│ │             │  │  OLAP)     │
  │ last 48h)   │  │ Neo4j (deep │  │            │
  │             │  │  analysis)  │  │            │
  │ Cold Store  │  │             │  │            │
  │ (S3 Parquet │  │             │  │            │
  │  + Bloom    │  │             │  │            │
  │  filter idx)│  │             │  │            │
  └──────┬──────┘  └─────────────┘  └────────────┘
         │
  ┌──────▼──────┐
  │  Query API  │  ← Federates across hot + cold storage
  │  Service    │     Uses Bloom filter to check if trace_id is in cold
  └──────┬──────┘
         │
  ┌──────▼──────┐
  │  Trace UI   │  ← Jaeger UI / Grafana Tempo
  │             │     Waterfall view, dependency graph, service map
  └─────────────┘
```

### Component Deep Dive

#### Trace Context Propagation

The foundation of distributed tracing: passing context across service boundaries.

```
Service A receives HTTP request:
  → SDK generates: trace_id = "abc123", span_id = "span-A", parent_id = null
  
Service A calls Service B via HTTP:
  Headers injected:
    traceparent: 00-abc123-span-A-01
    tracestate: vendor=value
  
Service B receives the call:
  → SDK extracts trace_id = "abc123", creates span_id = "span-B", parent_id = "span-A"
  
Service B calls Service C via gRPC:
  → Same trace_id propagated via gRPC metadata

Result: All spans share trace_id = "abc123" → can be correlated
```

**W3C Trace Context Standard** (traceparent header):
```
traceparent: {version}-{trace-id}-{parent-span-id}-{trace-flags}
Example:     00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

#### Sampling Strategies

At 5M spans/sec, storing everything is too expensive. Sampling reduces volume:

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Head-based** (probabilistic) | Decide at trace start: sample 10% | Simple, low overhead | May miss interesting traces |
| **Tail-based** ⭐ | Collect all spans, decide AFTER trace completes | Can sample 100% of errors/slow traces | Requires buffering all spans temporarily |
| **Adaptive** | Adjust rate per service/endpoint based on traffic | Ensures coverage of low-traffic endpoints | Complex to implement |
| **Always-on for errors** | Sample 100% of traces with errors | Never miss failures | Error-heavy services generate more data |

**Recommended**: Head-based 10% + always-on for errors + tail-based for slow traces (p99 > 2×median).

#### Span Ingester → Storage

```
Span received → validate → enrich (add service name, env, version) → write to storage

Storage Options:

Elasticsearch ⭐ (Jaeger default):
  ✓ Full-text search on span tags/logs
  ✓ Rich query language
  ✗ Expensive at high volume
  ✗ Index management complexity
  
Cassandra (Jaeger alternative):
  ✓ High write throughput
  ✓ Natural TTL for retention
  ✗ Limited query flexibility (must query by trace_id or service name)
  
ClickHouse (Grafana Tempo / SigNoz):
  ✓ Columnar — excellent for aggregation queries
  ✓ 10× less storage than Elasticsearch (compression)
  ✓ Fast range scans
  ✗ Less mature for tracing use case

Object Store + Index (Grafana Tempo approach) ⭐:
  Spans stored in S3/GCS as compressed blocks
  Index in Redis/Bloom filters for trace_id lookup
  ✓ 100× cheaper than Elasticsearch
  ✓ Infinite retention at low cost
  ✗ Query latency higher (must fetch from object store)
```

---

## 5. APIs

### Ingest Span (Collector)
```http
POST /api/v1/spans
Content-Type: application/json
[
  {
    "trace_id": "abc123",
    "span_id": "span-B",
    "parent_span_id": "span-A",
    "operation_name": "GET /api/users",
    "service_name": "user-service",
    "start_time": 1710320000000,
    "duration_us": 12500,
    "status": "OK",
    "tags": {"http.method": "GET", "http.status_code": 200, "db.type": "postgresql"},
    "logs": [{"timestamp": 1710320000005, "message": "cache miss, querying DB"}]
  }
]
```

### Query Traces
```http
GET /api/v1/traces?service=user-service&operation=GET+/api/users&minDuration=100ms&maxDuration=5s&limit=20&start=1710320000&end=1710406400

GET /api/v1/traces/{trace_id}
```

### Service Dependencies
```http
GET /api/v1/dependencies?start=1710320000&end=1710406400
Response: [{"parent": "api-gateway", "child": "user-service", "call_count": 150000}]
```

---

## 6. Data Model

### Span Schema (Elasticsearch / Cassandra)

```json
{
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "b7ad6b7169203331",
  "parent_span_id": "3a2fb4a1b3c4d5e6",
  "operation_name": "GET /api/users/{id}",
  "service_name": "user-service",
  "service_version": "v2.3.1",
  "environment": "production",
  "start_time_us": 1710320000000000,
  "duration_us": 12500,
  "status_code": "OK",
  "tags": {
    "http.method": "GET",
    "http.status_code": 200,
    "http.url": "/api/users/123",
    "db.type": "postgresql",
    "db.statement": "SELECT * FROM users WHERE id = $1",
    "peer.service": "postgres-primary",
    "component": "spring-web"
  },
  "logs": [
    {"timestamp_us": 1710320000005000, "event": "cache_miss"},
    {"timestamp_us": 1710320000008000, "event": "db_query_complete", "rows": 1}
  ],
  "process": {
    "hostname": "user-svc-pod-abc123",
    "ip": "10.0.5.42"
  }
}
```

### Cassandra — Span Storage

```sql
CREATE TABLE traces (
    trace_id        TEXT,
    span_id         TEXT,
    parent_span_id  TEXT,
    operation_name  TEXT,
    service_name    TEXT,
    start_time      TIMESTAMP,
    duration        BIGINT,
    tags            MAP<TEXT, TEXT>,
    PRIMARY KEY (trace_id, span_id)
) WITH default_time_to_live = 1209600;  -- 14 days

CREATE TABLE service_operations (
    service_name    TEXT,
    start_time      TIMESTAMP,
    trace_id        TEXT,
    duration        BIGINT,
    PRIMARY KEY ((service_name), start_time, trace_id)
) WITH CLUSTERING ORDER BY (start_time DESC);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Span loss during ingestion** | Kafka buffer between collector and ingester; retry on failure |
| **Collector overload** | Local agent batches spans; Kafka absorbs bursts |
| **Storage failure** | Elasticsearch replicas / Cassandra RF=3 |
| **SDK failure** | Tracing SDK must NEVER crash the application; all errors swallowed |
| **Clock skew** | Spans may have inconsistent timestamps across services. UI sorts by causal order (parent before child), not just timestamp |
| **High cardinality tags** | Limit tag key/value cardinality; reject unbounded values (e.g., user_id as tag) |

### Problem-Specific Challenges

#### 1. Clock Skew Across Services — Span Ordering Nightmare

```
Service A is on Host 1 (clock = 10:00:00.000)
Service B is on Host 2 (clock = 10:00:00.150 — 150ms AHEAD)

Service A calls Service B at T=10:00:00.100 (A's clock)
Service B receives at T=10:00:00.250 (B's clock — 150ms skew)
Service B responds at T=10:00:00.300 (B's clock)
Service A receives at T=10:00:00.200 (A's clock)

Problem: B's span STARTS at 10:00:00.250 but A's span ENDS at 10:00:00.200
  → Child span appears to START 50ms AFTER parent span ENDS
  → Waterfall chart looks broken (negative gap)

Solutions:
  1. UI adjustment: Use parent-child relationships (not timestamps) for ordering
     Display spans in causal order, adjust positions to avoid visual overlap
  
  2. Relative timestamps: Store duration (not absolute start/end) for child spans
     Child.start = parent.timestamp + network_delay (estimated)
  
  3. Clock sync: Enforce NTP across all hosts (target: < 10ms skew)
     Google uses TrueTime (GPS + atomic clocks) for < 7ms uncertainty
  
  4. Hybrid Logical Clocks (HLC): Track causal order even with clock skew
     Used by CockroachDB, can be adapted for trace ordering
```

#### 2. Trace Assembly — When Is a Trace "Complete"?

```
Problem: Spans arrive asynchronously from different services.
  Span A arrives at T=0, Span B at T=2s, Span C at T=5s
  When can we say the trace is complete and make a sampling decision?

Challenge: There's no "trace complete" signal. We must infer completion.

Solutions:
  1. Timeout-based ⭐: If no new spans arrive for 30 seconds → trace is "complete"
     ✓ Simple
     ✗ 30-second delay before sampling decision
     ✗ Long-running traces (e.g., async workflows) may time out prematurely
  
  2. Root span completion: When the root span (parent_id = null) finishes → trace is done
     ✓ Accurate for synchronous call chains
     ✗ Root span may complete before async child spans
     ✗ Root span might arrive AFTER child spans (network delay)
  
  3. Expected span count: Root span includes "expected_children" count
     ✓ Precise
     ✗ Requires instrumentation to predict child count (hard in dynamic systems)
  
  Practical approach: Timeout (30s) + root span heuristic. Accept that some
  traces may be sampled/discarded with incomplete span sets.
```

#### 3. Cardinality Explosion — The Silent Killer

```
Every unique label combination creates a new "series" in storage.
  
  Good labels (low cardinality):
    service_name: 200 unique values ✓
    http_method: 5 unique values ✓
    environment: 3 unique values ✓
    
  Dangerous labels (high cardinality):
    user_id: 100M unique values ✗ → 100M series!
    request_id: infinite ✗
    session_id: infinite ✗
    full_url (with query params): infinite ✗
  
  Impact of 100M series:
    Elasticsearch: 100M documents × 500 bytes = 50 GB per hour of data
    Index size: 2× data = 100 GB
    Query "errors for service X" scans 100M docs → TIMEOUT
  
  Solutions:
    1. Drop high-cardinality tags at collector (before storage)
    2. Move high-cardinality data to span logs (not indexed tags)
    3. Set hard limits: max 50 tag keys per span, max 256 chars per value
    4. Alert on cardinality: if unique values for a tag > 10K → auto-drop
```

#### 4. SDK Overhead — The Observer Effect

```
Tracing adds overhead to every instrumented service:
  
  Per-span cost:
    CPU: 2-5 μs to create a span (context allocation, ID generation)
    Memory: ~200 bytes per span in-flight
    Network: 500 bytes per span exported
  
  For a service handling 10K requests/sec with 3 spans per request:
    CPU: 30K × 5 μs = 150 ms/sec of CPU time (~15% of one core)
    Memory: 30K × 200 bytes = 6 MB in-flight spans
    Network: 30K × 500 bytes = 15 MB/sec exported to agent
  
  Mitigations:
    1. Sampling at SDK level: Head-based 10% → 90% less overhead
    2. Async export: SDK batches spans and exports in background thread
    3. Agent as sidecar: SDK sends UDP (fire-and-forget) to local agent
       → Zero blocking in the application hot path
    4. Noop spans: If not sampled, span operations are no-ops (< 1 μs overhead)
    5. Context propagation cost: ~100 ns to inject/extract headers (negligible)
  
  Golden rule: Tracing must NEVER be in the critical path.
  If the tracing backend is down, the application must continue unaffected.
  SDK must swallow ALL exceptions from tracing code.
```

---

## 8. Deep Dive: Engineering Trade-offs

### Why Kafka Between Collector and Storage?

```
Without Kafka (direct write):
  Collector → Elasticsearch directly
  ✗ Spike in traffic → Elasticsearch overwhelmed → span loss
  ✗ ES maintenance (rolling restart) → all spans lost during downtime

With Kafka buffer ⭐:
  Collector → Kafka → Ingester → Elasticsearch
  ✓ Kafka absorbs traffic spikes (2M spans/sec → Kafka handles easily)
  ✓ ES downtime → spans queue in Kafka → replayed when ES is back
  ✓ Multiple consumers: one writes to ES, another to ClickHouse, another builds dependency graphs
  ✗ Additional infrastructure
  ✗ 5-10ms added latency (acceptable for tracing)
```

### Head-Based vs Tail-Based Sampling

```
Head-based (decide at entry point):
  API Gateway receives request → random(0,1) < 0.1 → SAMPLE
  trace_id carries the sampling decision → all downstream services respect it
  
  ✓ Simple, low overhead
  ✗ A 10% sample rate means 90% of errors are MISSED
  
Tail-based (decide after trace completes):
  ALL spans buffered in collector for ~30 seconds
  After trace completes: check if any span has error OR duration > threshold
  If yes → keep entire trace. If no → discard or downsample.
  
  ✓ 100% of error traces captured
  ✓ 100% of slow traces captured
  ✗ Requires buffering ALL spans temporarily (memory-intensive)
  ✗ Must wait for trace "completion" (how long to wait? Use timeout)
  
  Implementation: OpenTelemetry Collector with tail-sampling processor
```

