# 40. Design a Log Aggregation and Search System (like Splunk / ELK)

---

## 1. Functional Requirements (FR)

- **Collect logs**: Ingest logs from thousands of services, servers, containers, cloud resources
- **Structured and unstructured**: Handle JSON, syslog, plain text, multi-line stack traces
- **Search**: Full-text search across all logs with < 5 second latency
- **Filter**: By service, severity, time range, hostname, custom fields
- **Live tail**: Stream new logs matching a filter in real-time
- **Dashboards**: Visualize log volume, error rates, trends
- **Alerting**: Alert when error log rate exceeds threshold or specific patterns appear
- **Retention**: Configurable per log source (7 days to 1 year)

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: Ingest 1M+ log lines/sec
- **Low Latency Search**: Query results in < 5 seconds for last 24 hours
- **Scalability**: Petabytes of logs, thousands of data sources
- **Durability**: Ingested logs must not be lost
- **Cost Efficient**: Storage costs are the dominant expense — compress and tier aggressively
- **High Availability**: 99.99%

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Log lines / sec | 1M |
| Avg log line size | 500 bytes |
| Ingestion throughput | 500 MB/s |
| Storage / day (raw) | 43 TB |
| With compression (~10×) | 4.3 TB/day |
| 30-day retention | ~130 TB |
| 1-year (with tiering) | ~400 TB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│               Log Sources (Thousands of Hosts/Containers)            │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Application  │  │ Infrastructure│  │ Cloud Native │               │
│  │ (stdout/file)│  │ (syslog,     │  │ (K8s pods,   │               │
│  │              │  │  kernel,     │  │  Lambda,     │               │
│  │ JSON / plain │  │  systemd)    │  │  CloudWatch) │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                  │                  │                       │
│  ┌──────▼──────────────────▼──────────────────▼───────────────────┐  │
│  │              Log Agent (per-host sidecar)                      │  │
│  │              Vector / Filebeat / Fluentd                       │  │
│  │                                                                │  │
│  │  Pipeline:                                                     │  │
│  │  1. Tail log files / capture stdout                           │  │
│  │  2. Multiline aggregation (stack traces → 1 entry)            │  │
│  │  3. Parse: JSON auto-detect, grok regex, custom parsers       │  │
│  │  4. Enrich: add hostname, service_name, env, cluster          │  │
│  │  5. Rate limit: max 1000 lines/sec per source                 │  │
│  │  6. Buffer: in-memory (1000 events) + disk spillover (1 GB)   │  │
│  │  7. Compress: snappy/lz4 before sending                       │  │
│  │  8. Send to Kafka with retry + backoff                        │  │
│  └──────────────────────────┬─────────────────────────────────────┘  │
└─────────────────────────────┼────────────────────────────────────────┘
                              │ gRPC / HTTP (compressed batches)
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Kafka Cluster                                │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Topic: logs-raw  (RF=3, partitioned by service_name hash)     │  │
│  │ Retention: 48 hours (buffer for reprocessing)                 │  │
│  └────────────────────────────┬───────────────────────────────────┘  │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                  │
     ┌────────▼────────┐ ┌─────▼──────────┐ ┌─────▼──────────┐
     │ Log Processor   │ │ Alert          │ │ Live Tail      │
     │ (Vector /       │ │ Evaluator      │ │ Service        │
     │  Logstash)      │ │                │ │                │
     │                 │ │ Pattern match: │ │ Kafka consumer │
     │ 1. Deep parse   │ │ • "OOM killed" │ │ per active     │
     │ 2. GeoIP lookup │ │ • "disk full"  │ │ tail session   │
     │ 3. PII redact   │ │ • error rate   │ │                │
     │    (mask emails,│ │   > threshold  │ │ Filter + push  │
     │     SSN, etc.)  │ │                │ │ via WebSocket  │
     │ 4. Route by     │ │ → PagerDuty /  │ │ to client      │
     │    severity     │ │   Slack alert  │ │                │
     └────────┬────────┘ └────────────────┘ └────────────────┘
              │
              │ Bulk index (batch 5000 docs or 5 seconds)
              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Storage Layer (Tiered)                            │
│                                                                       │
│  ┌─────────────────────────────────────┐                             │
│  │  HOT TIER (0-48 hours)              │                             │
│  │  Elasticsearch / OpenSearch         │                             │
│  │                                     │                             │
│  │  • Full inverted index on all fields│                             │
│  │  • SSD-backed for fast queries      │                             │
│  │  • 5 shards × 1 replica per index  │                             │
│  │  • Index per day: logs-2026.03.14  │                             │
│  │  • Rolling index with ILM           │                             │
│  └────────────────┬────────────────────┘                             │
│                   │ ILM rollover after 48h                           │
│  ┌────────────────▼────────────────────┐                             │
│  │  WARM TIER (2-30 days)              │                             │
│  │  Elasticsearch (HDD) or S3 Parquet  │                             │
│  │                                     │                             │
│  │  • Force merge: 5 shards → 1 shard │                             │
│  │  • Read-only, compressed            │                             │
│  │  • Slower queries but still indexed │                             │
│  └────────────────┬────────────────────┘                             │
│                   │ ILM after 30 days                                │
│  ┌────────────────▼────────────────────┐                             │
│  │  COLD TIER (30-365 days)            │                             │
│  │  S3 / GCS (object store)            │                             │
│  │                                     │                             │
│  │  • Compressed Parquet files          │                             │
│  │  • Query via Athena / Presto        │                             │
│  │  • Cost: $0.023/GB/month            │                             │
│  │  • Access time: 2-10 seconds        │                             │
│  └────────────────┬────────────────────┘                             │
│                   │ Lifecycle after 365 days                         │
│  ┌────────────────▼────────────────────┐                             │
│  │  ARCHIVE (365+ days / compliance)   │                             │
│  │  S3 Glacier Deep Archive            │                             │
│  │  Cost: $0.00099/GB/month            │                             │
│  │  Retrieval: 12-48 hours             │                             │
│  └─────────────────────────────────────┘                             │
└──────────────────────────────────────────────────────────────────────┘
              │
     ┌────────▼───────────────────────────────────────┐
     │           Query Federation Layer                │
     │                                                 │
     │  Query: "errors from payment-service last 7d"  │
     │                                                 │
     │  1. Last 48h → query Elasticsearch (fast)      │
     │  2. 2-7 days → query warm ES / S3 Parquet      │
     │  3. Merge results, sort by timestamp            │
     │  4. Return to UI                                │
     └────────┬───────────────────────────────────────┘
              │
     ┌────────▼────────┐
     │   Kibana /      │
     │   Grafana UI    │
     │                 │
     │  • Search bar   │
     │  • Log stream   │
     │  • Dashboards   │
     │  • Live tail    │
     └─────────────────┘
```

### Component Deep Dive

#### Log Collection Agents

**Why agents on every host?**
- Applications write logs to stdout/files → agent tails and ships
- Agent handles: batching, compression, backpressure, retry, buffering
- Agent options: **Filebeat** (lightweight), **Fluentd** (plugin ecosystem), **Vector** (high performance, Rust)

**Log Pipeline**:
```
Application → stdout → Container runtime captures → Agent tails →
  → Parse (JSON/regex) → Enrich (add hostname, service name, env) →
  → Buffer locally (in case Kafka is slow) → Send to Kafka
```

#### Log Parsing and Transformation

```
Raw log: "2026-03-14T10:00:00.123Z ERROR [user-service] Failed to process payment: timeout after 5000ms"

After parsing:
{
  "timestamp": "2026-03-14T10:00:00.123Z",
  "level": "ERROR",
  "service": "user-service",
  "message": "Failed to process payment: timeout after 5000ms",
  "host": "user-svc-pod-abc",
  "env": "production",
  "cluster": "us-east-1"
}

Parsing approaches:
  - JSON logs: zero-cost parsing (already structured) ⭐
  - Grok patterns: regex-based parsing for unstructured logs
  - Multiline: Java stack traces → combine into single log entry
```

#### Storage: Elasticsearch vs Loki vs ClickHouse

| Feature | Elasticsearch | Grafana Loki ⭐ | ClickHouse |
|---|---|---|---|
| **Index model** | Full-text inverted index on ALL fields | Index only labels (not log content) | Columnar with full-text index |
| **Storage cost** | High (index = 1.5× raw data) | Low (just compressed chunks) | Medium |
| **Query speed** | Fast (indexed) | Slower for full-text (grep-like) | Fast for structured queries |
| **Best for** | Search-heavy, ad-hoc queries | Large volume, label-based filtering | Structured logs, SQL queries |
| **Operations** | Complex (JVM, cluster management) | Simple | Moderate |

**Recommendation**: 
- **Loki** for cost-efficient, label-based log aggregation (Kubernetes-native)
- **Elasticsearch** for full-text search on unstructured logs (Splunk alternative)
- **ClickHouse** for structured/semi-structured logs with SQL queries

#### Storage Tiering

```
Hot tier (0-48 hours): Elasticsearch / Loki local storage
  → Full indexing, fastest queries
  → SSD-backed

Warm tier (2-30 days): S3 with compressed Parquet
  → Loki/Elasticsearch can query S3 directly (slower)
  → Cost: ~$0.023/GB/month (vs $0.10/GB on SSD)

Cold tier (30+ days): S3 Glacier
  → Compliance/audit purposes only
  → Cost: ~$0.004/GB/month
  → Access time: minutes to hours
```

---

## 5. APIs

### Search Logs
```http
POST /api/v1/logs/search
{
  "query": "level:ERROR AND service:payment-service AND message:*timeout*",
  "from": "2026-03-14T04:00:00Z",
  "to": "2026-03-14T10:00:00Z",
  "limit": 100,
  "sort": "timestamp:desc"
}
```

### Live Tail (WebSocket)
```json
// Client subscribes:
{"type": "subscribe", "filter": "service=payment-service AND level=ERROR"}

// Server pushes matching logs in real-time:
{"type": "log", "data": {"timestamp": "...", "message": "...", "level": "ERROR"}}
```

---

## 6. Data Model

### Elasticsearch Index
```json
{
  "mappings": {
    "properties": {
      "timestamp": {"type": "date"},
      "level": {"type": "keyword"},
      "service": {"type": "keyword"},
      "host": {"type": "keyword"},
      "env": {"type": "keyword"},
      "message": {"type": "text", "analyzer": "standard"},
      "trace_id": {"type": "keyword"},
      "custom_fields": {"type": "object", "dynamic": true}
    }
  },
  "settings": {
    "index.number_of_shards": 5,
    "index.number_of_replicas": 1,
    "index.lifecycle.name": "logs-policy"    // ILM: hot → warm → cold → delete
  }
}
```

### Index Lifecycle Management (ILM)
```
Hot:  0-2 days  → 5 shards, 1 replica, SSD, force merge at rollover
Warm: 2-30 days → shrink to 1 shard, read-only, HDD, compressed
Cold: 30-90 days → freeze index, S3 snapshot
Delete: > 90 days → delete index
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Agent failure** | Agent buffers to local disk; retry on restart |
| **Kafka down** | Agent buffers locally (up to 1 GB); retries with backoff |
| **Elasticsearch overload** | Kafka absorbs bursts; auto-scale indexer workers |
| **Index corruption** | Elasticsearch replicas; daily S3 snapshots |
| **Query overload** | Query timeout (30s); rate limiting per user |
| **Disk full** | ILM auto-deletes oldest indices; alert on disk usage > 80% |

### Race Conditions and Scale Challenges

#### 1. Log Volume Spike — Cascading Failure Floods Logs

```
Normal: 500K log lines/sec
Error loop: Service encounters an error → logs it → retries → logs again → repeat
  → 50M log lines/sec from ONE service (100× normal)

Impact:
  - Kafka topic fills up → other services' logs delayed
  - Elasticsearch indexing falls behind → hours of query lag
  - Disk fills up → older logs deleted prematurely

Solutions:
  1. Per-service rate limiting at the agent level:
     Agent config: max 1000 log lines/sec per service
     Excess logs sampled (keep every 100th) or dropped with a counter
     "Dropped 49,000 messages in last minute from service X"
  
  2. Kafka per-service topic or quotas:
     Each service gets a Kafka quota (bytes/sec limit)
     If exceeded → producer is throttled (backpressure to application)
  
  3. Circuit breaker in application:
     If logging fails N times → stop logging for 30 seconds
     Prevents the logging subsystem from making the outage worse
  
  4. Tiered ingestion:
     Priority 1: ERROR and CRITICAL logs → always ingested
     Priority 2: WARN logs → ingested if capacity permits
     Priority 3: INFO/DEBUG → dropped during spikes
```

#### 2. Multi-Line Log Parsing — Stack Traces

```
Java exception:
  2026-03-14 10:00:00 ERROR com.app.Service - Payment failed
  java.lang.NullPointerException: null
      at com.app.PaymentService.process(PaymentService.java:42)
      at com.app.OrderController.checkout(OrderController.java:15)
      at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
      at java.lang.Thread.run(Thread.java:748)

Without multiline handling: 6 separate log entries, each incomplete
With multiline handling: 1 log entry with the full stack trace

Implementation:
  Agent rule: "Lines starting with whitespace or 'at' are continuations"
  Pattern: negate: true, match: after, pattern: '^\s|^Caused by|^at '
  
  Timeout: After 500ms without a continuation line → flush the multiline buffer
  
  Race condition: If a non-continuation log arrives between stack trace lines
    (e.g., another thread logs simultaneously) → stack trace split incorrectly
  
  Solution: Log with unique correlation ID per thread. Agent reassembles by correlation ID.
  Or: Use structured logging (JSON) where each log is one line → no multiline issue ⭐
```

#### 3. Index Management — The Elasticsearch Scale Problem

```
Daily index rollover: logs-2026.03.14, logs-2026.03.15, etc.

At 4.3 TB/day compressed:
  One day = one index with 5 shards × 2 replicas = 10 shards
  30 days = 300 shards
  
  Elasticsearch cluster: 30 nodes × 10 shards/node = OK
  But if we have 50 services with separate indices:
  50 services × 30 days × 10 shards = 15,000 shards → PROBLEM
  
  Elasticsearch rule of thumb: max ~1000 shards per node
  30 nodes × 1000 = 30,000 max → 15,000 is borderline
  
  Solutions:
    1. Data streams (ES 7.9+): Auto-manage rollover, ILM, and deletion
    2. Merge small indices: Don't create per-service indices unless high volume
       Use one "logs" data stream with service as a field (filtered at query time)
    3. Force merge cold indices: Reduce 5 shards → 1 shard after rollover
    4. Shrink replicas: cold indices → 0 replicas (saved in S3 snapshots instead)
```

#### 4. Live Tail Scaling — Streaming Matching Logs to Users

```
User wants: "Show me all ERROR logs from payment-service in real-time"

Implementation:
  Option 1: Query Elasticsearch repeatedly (poll every 1 second)
    ✗ Inefficient: each poll is a full query
    ✗ 1-second delay minimum
    ✗ 100 users tailing → 100 queries/second → ES load
  
  Option 2: Kafka consumer with server-side filter ⭐
    Server creates a temporary Kafka consumer
    Reads from the "logs" topic in real-time
    Applies the filter (service=payment-service AND level=ERROR)
    Streams matching logs to the user's WebSocket
    
    Challenge: Kafka topic has ALL logs (1M lines/sec)
      Reading and filtering 1M lines/sec per tailing user is expensive
    
    Solution: Pre-filtered Kafka topics
      Topic: logs-payment-service-error (low volume, ~100 lines/sec)
      Created automatically by the log router based on common patterns
      
    Or: Use Kafka Streams/Flink to create a filtered stream per active tail session
      (but this doesn't scale to hundreds of concurrent tail sessions)
    
    Practical limit: ~50 concurrent live tail sessions
    Beyond that: Use pre-filtered topics or accept polling-based approach
```

---

## 8. Deep Dive: Engineering Trade-offs

### Indexing Everything vs Indexing Labels Only

```
Full indexing (Elasticsearch):
  Every word in every log line is indexed (inverted index)
  Query: "timeout in payment processing" → instant results
  ✓ Fast ad-hoc search
  ✗ Index is 1-2× the size of raw data (doubles storage cost)
  ✗ Index build is CPU-intensive (limits ingestion throughput)

Label-only indexing (Loki):
  Only metadata labels are indexed (service, level, env, host)
  Log content is stored as compressed chunks, searched via brute-force grep
  Query: {service="payment"} |= "timeout" → grep through payment service chunks
  ✓ 10× less storage (no content index)
  ✓ 10× higher ingestion throughput
  ✗ Full-text search is slower (must scan chunks)
  ✗ Not suitable for ad-hoc search across all services

Decision: If you know WHICH service to search → Loki is 10× cheaper.
          If you need "search everywhere for this error" → Elasticsearch.
```

### PII Redaction — Preventing Sensitive Data in Logs

```
Problem: Developers accidentally log sensitive data:
  log.info("User registered: email={}, ssn={}", user.email, user.ssn)
  → PII (Personally Identifiable Information) now in your log system
  → GDPR violation: PII in logs means user data deletion requests 
    require scanning ALL logs (100+ TB) — nearly impossible

Solutions (defense in depth):

  Layer 1: Code-level prevention (best)
    Use structured logging with allowlisted fields:
      log.info("User registered", userId=user.id)  // Only log non-PII IDs
    Linting rules: fail CI if log statement contains "email", "ssn", "password"
    
  Layer 2: Agent-level redaction
    Vector/Fluentd transforms:
      If field matches email regex → replace with [REDACTED_EMAIL]
      If field matches SSN pattern → replace with ***-**-****
      If field matches credit card → replace with ****-****-****-{last4}
    
    Regex patterns for common PII:
      Email:  [a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
      SSN:    \d{3}-\d{2}-\d{4}
      CC:     \d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}
    
  Layer 3: Tokenization
    Replace PII with a reversible token:
      "email=alice@company.com" → "email=tok_a1b2c3d4"
    Token mapping stored in a separate vault (only accessible by compliance team)
    Log system never sees real PII

  Cost of getting this wrong:
    GDPR fine: up to 4% of global annual revenue
    CCPA fine: $7,500 per intentional violation
```

### Structured vs Unstructured Logging — Why JSON Wins

```
Unstructured:
  2026-03-14 10:00:00 ERROR [payment-service] User 12345 payment failed: timeout after 5000ms
  
  To extract fields: regex parsing (fragile, slow, breaks on format changes)
  Grok pattern: %{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:service}\] %{GREEDYDATA:message}

Structured (JSON) ⭐:
  {"timestamp":"2026-03-14T10:00:00Z","level":"ERROR","service":"payment-service",
   "user_id":"12345","message":"payment failed","error":"timeout","duration_ms":5000,
   "trace_id":"abc123"}
  
  ✓ Zero parsing overhead (fields already extracted)
  ✓ Typed fields (duration_ms is a number, not a string)
  ✓ No regex maintenance
  ✓ Consistent schema → better Elasticsearch mapping
  ✓ trace_id enables log-to-trace correlation
  ✗ Slightly larger per-line (field names repeated)
  ✗ Less human-readable in raw form
  
  Compression eliminates the size difference (field names deduplicated by compressor)
  
  Industry trend: ALL modern systems use structured JSON logging.
  Libraries: logback-json (Java), structlog (Python), zap (Go), winston (Node.js)
```

### Log-Trace-Metric Correlation — The Observability Triangle

```
User reports: "My payment failed"

Without correlation:
  1. Search logs for "payment failed" → find log line
  2. Log line has no trace_id → which request was it?
  3. Manually search metrics for payment service error rate → find the spike
  4. No way to connect the log, the trace, and the metric → painful debugging

With correlation ⭐:
  Every log line includes trace_id and span_id:
    {"level":"ERROR","message":"payment timeout","trace_id":"abc123","span_id":"def456"}
  
  1. Find the error log → extract trace_id
  2. Click trace_id → opens Jaeger with full request trace
  3. See exactly which service/DB call was slow
  4. Click "View metrics" → see the service's error rate dashboard
  
  Implementation:
    Log SDK auto-injects trace_id from OpenTelemetry context:
      logger.with(traceId=span.getTraceId()).error("payment timeout")
    
    Elasticsearch: trace_id is a keyword field → instant lookup
    Dashboard: Grafana links from logs panel → traces panel → metrics panel

This is the "three pillars of observability" — logs, traces, metrics — all connected.
```

### Cost Optimization — The Biggest Challenge at Scale

```
At 43 TB/day raw:
  Elasticsearch: ~$0.10/GB on SSD → $4,300/day → $1.57M/year just for hot storage!
  
  Cost breakdown (typical):
    Hot tier (Elasticsearch, 2 days):  43 TB × 2 × $0.10 = $8,600
    Warm tier (ES HDD, 28 days):       43 TB × 28 × $0.03 = $36,120
    Cold tier (S3, 335 days):           43 TB × 335 × $0.023 = $331,385
    Total annual: ~$376K (with this tiering) vs $1.57M (all in hot)
    
    Savings: 76% cost reduction from tiering alone
    
  Additional optimizations:
    1. Compression: gzip/zstd on warm/cold → 10× compression → 10× less storage
    2. Sampling: Keep 100% of ERROR/WARN, sample 10% of INFO/DEBUG
       → 70% volume reduction (most logs are INFO)
    3. Drop noisy logs: Health check logs, heartbeats → filter at agent level
       → 20-30% volume reduction
    4. Field pruning: Don't index fields you never search
       → 30-50% index size reduction
    
    After all optimizations: ~$100K/year (vs $1.57M naive approach)
    94% cost reduction through engineering!
```

