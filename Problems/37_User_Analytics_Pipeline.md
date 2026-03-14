# 37. Design a User Analytics Pipeline (like Google Analytics)

---

## 1. Functional Requirements (FR)

- **Event tracking**: Capture page views, clicks, scrolls, custom events from web/mobile
- **Real-time dashboard**: Show active users, events/sec, top pages in real-time
- **Historical analytics**: Queries like "page views last 7 days, grouped by country"
- **Funnels**: Track conversion funnels (e.g., landing → signup → purchase)
- **Cohort analysis**: Retention by signup week, behavior by user segment
- **Custom dimensions**: Track arbitrary key-value metadata per event
- **Session tracking**: Group events into sessions (30-min inactivity timeout)

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: 1M+ events/sec from millions of websites/apps
- **Low Latency Ingestion**: Events ingested in < 5 seconds
- **Query Performance**: Dashboard queries in < 2 seconds
- **Scalability**: Handle 100B+ events/day
- **Data Retention**: Raw events for 90 days, aggregated data for 2 years
- **Fault Tolerant**: No event loss; at-least-once delivery

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Events / day | 100B |
| Events / sec | ~1.2M |
| Avg event size | 500 bytes |
| Ingestion throughput | 600 MB/s |
| Storage / day (raw) | 50 TB |
| Storage / 90 days | 4.5 PB |
| Unique users tracked | 1B |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Client Instrumentation Layer                      │
│                                                                       │
│  ┌───────────────────┐   ┌───────────────────┐                       │
│  │   Web SDK (JS)    │   │  Mobile SDK       │                       │
│  │                   │   │  (iOS/Android)     │                       │
│  │ • Auto-capture:   │   │                    │                       │
│  │   page views,     │   │ • Lifecycle events │                       │
│  │   clicks, scroll  │   │ • Screen views     │                       │
│  │ • Batching:       │   │ • Offline queue    │                       │
│  │   buffer 5s or    │   │   (SQLite)         │                       │
│  │   20 events       │   │ • Flush on app     │                       │
│  │ • sendBeacon()    │   │   backgrounding    │                       │
│  │   on page unload  │   │                    │                       │
│  │ • localStorage    │   │                    │                       │
│  │   fallback queue  │   │                    │                       │
│  └────────┬──────────┘   └────────┬───────────┘                       │
└───────────┼────────────────────────┼──────────────────────────────────┘
            │   HTTPS POST /collect  │
            ▼                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│              Edge Collector Cluster (Multi-Region)                    │
│                                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │ Collector   │  │ Collector   │  │ Collector   │                  │
│  │ us-east     │  │ eu-west     │  │ ap-south    │                  │
│  │             │  │             │  │             │                  │
│  │ Processing: │  │             │  │             │                  │
│  │ 1. Validate │  │ Same flow   │  │ Same flow   │                  │
│  │    schema   │  │             │  │             │                  │
│  │ 2. GeoIP    │  │             │  │             │                  │
│  │    lookup   │  │             │  │             │                  │
│  │    (MaxMind │  │             │  │             │                  │
│  │    in-mem)  │  │             │  │             │                  │
│  │ 3. UA parse │  │             │  │             │                  │
│  │ 4. Bot      │  │             │  │             │                  │
│  │    filter   │  │             │  │             │                  │
│  │ 5. PII      │  │             │  │             │                  │
│  │    scrub    │  │             │  │             │                  │
│  │ 6. Publish  │  │             │  │             │                  │
│  │    to Kafka │  │             │  │             │                  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                  │
└─────────┼────────────────┼────────────────┼──────────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Kafka Cluster (Central)                            │
│                                                                       │
│  Topics:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ raw-events          (partitioned by site_id, RF=3)             │ │
│  │ enriched-events     (after processing, ready for storage)       │ │
│  │ session-events      (session-stitched events)                   │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
┌────────▼────────┐ ┌───────▼───────┐ ┌───────▼───────┐
│  Real-Time      │ │  Batch ETL    │ │  Alert        │
│  Pipeline       │ │  Pipeline     │ │  Pipeline     │
│  (Flink)        │ │  (Spark)      │ │  (Flink)      │
│                 │ │               │ │               │
│  Aggregations:  │ │  Hourly jobs: │ │  Rules:       │
│  • Active users │ │  • Session    │ │  • Error rate │
│    (1-min win.) │ │    reconstr.  │ │    > 5%       │
│  • Events/sec   │ │  • Funnel     │ │  • Zero       │
│    (10s slide)  │ │    analysis   │ │    events for │
│  • Top pages    │ │  • Cohort     │ │    > 10 min   │
│    (1-min win.) │ │    retention  │ │  • Anomaly    │
│  • Referrer     │ │  • Pre-agg    │ │    detection  │
│    tracking     │ │    for OLAP   │ │               │
│                 │ │  • S3 Parquet │ │  → PagerDuty  │
│                 │ │    archival   │ │  → Slack      │
└────────┬────────┘ └───────┬───────┘ └───────────────┘
         │                  │
┌────────▼────────┐ ┌───────▼───────┐
│     Redis       │ │  ClickHouse   │
│  (Real-Time)    │ │  (Historical  │
│                 │ │   OLAP)       │
│  • active_users │ │               │
│  • top_pages    │ │  Materialized │
│  • events/sec   │ │  views for    │
│  • HyperLogLog  │ │  common       │
│    for uniques  │ │  dashboard    │
│                 │ │  queries      │
└────────┬────────┘ └───────┬───────┘
         │                  │
         └────────┬─────────┘
           ┌──────▼──────┐
           │  Query      │ ← Federates real-time (Redis) + historical (ClickHouse)
           │  Service    │
           └──────┬──────┘
           ┌──────▼──────┐
           │  Dashboard  │ ← React SPA
           │  UI         │    Auto-refresh: 5s for real-time, 1min for historical
           └─────────────┘
```

### Component Deep Dive

#### Event Collector (Edge Ingestion)

**Client SDK** sends batched events:
```json
POST /collect
{
  "site_id": "site-123",
  "client_id": "anon-uuid",      // browser fingerprint / cookie
  "session_id": "sess-uuid",
  "events": [
    {"type": "pageview", "url": "/pricing", "ts": 1710320000, "referrer": "google.com"},
    {"type": "click", "element": "#signup-btn", "ts": 1710320005},
    {"type": "custom", "name": "video_play", "props": {"video_id": "v123"}, "ts": 1710320010}
  ]
}
```

**Collector enrichment**:
- GeoIP lookup (MaxMind DB — in-memory): IP → country, city, region
- User-Agent parsing: browser, OS, device type
- Session stitching: group events with same session_id
- Bot filtering: reject known bot user agents

**Scaling**: Collectors are stateless → auto-scale horizontally behind a global LB. Deploy at edge (multi-region) for low latency ingestion.

#### Real-Time Pipeline (Apache Flink)

```
Flink consumes from Kafka → windowed aggregations:

1. Active Users (1-min tumbling window):
   Count distinct client_ids per site_id per minute
   Write to Redis: SETEX active_users:{site_id} 120 {count}

2. Events per second (10-sec sliding window):
   Count events per site_id → Redis time-series

3. Top Pages (1-min window):
   Count pageviews per URL per site_id → Redis sorted set
   ZADD top_pages:{site_id} {count} {url}
```

#### Batch Pipeline (Apache Spark)

```
Daily/hourly Spark jobs:
1. Read raw events from Kafka → S3 (Parquet, partitioned by date + site_id)
2. Session reconstruction: group events by session_id, compute session duration
3. Funnel analysis: for each funnel definition, count users at each step
4. Pre-aggregate: pageviews by day × country × device → materialized views in ClickHouse
5. Cohort analysis: retention by signup week
```

#### ClickHouse — OLAP Query Engine

**Why ClickHouse**: Columnar storage, vectorized execution, 10-100× faster than PostgreSQL for analytical queries. Handles billions of rows with sub-second query times.

```sql
CREATE TABLE events (
    event_date    Date,
    site_id       String,
    session_id    String,
    client_id     String,
    event_type    LowCardinality(String),
    url           String,
    referrer      String,
    country       LowCardinality(String),
    device_type   LowCardinality(String),
    browser       LowCardinality(String),
    custom_props  Map(String, String),
    timestamp     DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY (event_date, site_id)
ORDER BY (site_id, event_date, session_id, timestamp);
```

---

## 5. APIs

### Collect Events
```http
POST /collect
Content-Type: application/json
{ "site_id": "...", "events": [...] }
→ 204 No Content (fire-and-forget, beacon API compatible)
```

### Query Dashboard
```http
GET /api/v1/analytics/{site_id}/realtime
→ { "active_users": 1523, "events_per_sec": 342, "top_pages": [...] }

GET /api/v1/analytics/{site_id}/report?metric=pageviews&from=2026-03-01&to=2026-03-14&group_by=country
→ { "data": [{"country": "US", "pageviews": 1523000}, ...] }
```

---

## 6. Data Model

### Raw Event (Kafka → S3 Parquet)
```json
{ "event_id": "uuid", "site_id": "s123", "session_id": "sess-abc",
  "client_id": "anon-xyz", "event_type": "pageview", "url": "/pricing",
  "referrer": "google.com", "country": "US", "device": "mobile",
  "browser": "Chrome", "timestamp": 1710320000000,
  "custom": {"plan": "pro", "source": "ad_campaign_123"} }
```

### Redis (Real-Time)
```
active_users:{site_id}        → integer (TTL 120s)
events_per_sec:{site_id}      → time-series (last 60 data points)
top_pages:{site_id}           → sorted set (top 100 URLs)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Event loss** | Kafka RF=3; collector returns 204 only after Kafka ACK |
| **Flink failure** | Flink checkpointing to S3 every 60s; resume from checkpoint |
| **ClickHouse failure** | Replicated tables (ReplicatedMergeTree); multi-AZ |
| **Client offline** | SDK buffers events in localStorage; sends on reconnect |
| **Bot traffic** | Filter known bots; rate limit per IP; CAPTCHA for suspicious patterns |
| **Data skew** | Partition Kafka by site_id; ClickHouse partitioned by date + site |

### Race Conditions and Scale Challenges

#### 1. Session Stitching — When Does a Session End?

```
Problem: Events arrive from the same user but we need to group them into sessions.
  Session = sequence of events with no gap > 30 minutes.

Challenges:
  - Events arrive out of order (mobile user on spotty connection)
  - User has multiple tabs/devices → separate sessions or one?
  - Events delayed by 10 minutes (device was offline) → same session?

Implementation (Flink session windows):
  Flink session window with 30-minute gap:
    Events for same client_id are grouped until 30-min silence
    Watermark: allow 5-minute late arrivals before closing window
  
  Race condition: Event arrives 6 minutes late (after watermark)
    → Falls into a new session → session count inflated
    
  Mitigation:
    - Accept ~1% session count inflation as a tradeoff
    - Or: Increase watermark to 30 minutes (but delays session metrics by 30 min)
    - Batch correction: Spark job re-stitches sessions hourly using complete data
```

#### 2. Beacon API — Fire-and-Forget Reliability

```
Client SDK uses navigator.sendBeacon() for page unload events:
  window.addEventListener('beforeunload', () => {
    navigator.sendBeacon('/collect', JSON.stringify(events));
  });

Problems:
  1. Beacon has no response → client doesn't know if it was received
  2. Ad blockers may block the /collect endpoint
  3. Browser may drop the beacon if it's too large (64 KB limit)
  4. Beacon is POST-only, fire-and-forget → at-most-once delivery
  
Mitigations:
  1. Batch events regularly (every 5s) via XHR, not just on unload
  2. Use pixel tracking (GET /pixel.gif?events=...) as fallback for ad blockers
  3. Keep beacon payload small (< 10 KB, just the events since last batch)
  4. Accept ~2-5% event loss (industry standard for web analytics)
  5. Client-side localStorage queue: buffer events → send on next page load if beacon failed
```

#### 3. Active Users Double-Counting

```
"1,523 users currently on your site" — how to count accurately?

Problem: User has 3 browser tabs open → 3 client_ids? 1 active user?
  If using random client_id per tab → 3× overcounting
  If using cookie-based client_id → correct (1 user)
  If user clears cookies → new client_id → counted as new user
  
Redis HyperLogLog for approximate unique counting:
  PFADD active_users:{site_id}:{minute} {client_id}
  PFCOUNT active_users:{site_id}:{minute} → ~1523 (0.81% error)
  
  Memory: 12 KB per HyperLogLog (regardless of cardinality!)
  50K sites × 12 KB = 600 MB total for all sites → trivial
  
  Trade-off: 0.81% error on unique count
    For 1,523 actual users → reported as 1,511-1,535
    → Perfectly acceptable for analytics dashboards
```

#### 4. Late-Arriving Events — Time Window Accuracy

```
Dashboard shows: "Page views in the last hour"
  Query: SELECT COUNT(*) FROM events WHERE timestamp > now() - 1h

But events from 45 minutes ago are still arriving (mobile user was offline)
  → The "last hour" count keeps changing as late events arrive
  
  Two approaches:
    1. Processing time: Count when event ARRIVES at server
       ✓ Stable (count doesn't change retroactively)
       ✗ Inaccurate (event from 45 min ago counted as "now")
    
    2. Event time ⭐: Count by the timestamp in the event
       ✓ Accurate (event is counted in the correct time window)
       ✗ Count for "last hour" keeps changing for ~5 minutes
       
  Industry standard: Event time with a 5-minute "settling window"
    "Numbers for the current hour are approximate. Final numbers available after 5 minutes."
```

---

## 8. Deep Dive: Engineering Trade-offs

### ClickHouse vs Druid vs BigQuery vs TimescaleDB

| Feature | ClickHouse ⭐ | Druid | BigQuery | TimescaleDB |
|---|---|---|---|---|
| **Query speed** | Sub-second on billions | Sub-second (pre-agg) | Seconds (serverless) | Seconds on millions |
| **Ingestion** | 1M+ rows/sec | 1M+ rows/sec | Streaming insert | 100K rows/sec |
| **Cost** | Low (self-hosted) | Moderate | High (pay-per-query) | Low |
| **Real-time** | Near real-time | Real-time | Minutes delay | Real-time |
| **SQL support** | Full SQL | Limited | Full SQL | Full SQL |
| **Operations** | Moderate | Complex | Zero (managed) | Easy |

**Choose ClickHouse** for self-hosted, high-volume analytics with full SQL.
**Choose BigQuery** for managed, pay-per-query with no ops.
**Choose Druid** for true real-time with sub-second queries on streaming data.

### Funnel Analysis — Implementation Deep Dive

```
Funnel: Landing Page → Sign Up → Add Payment → First Purchase

Raw events for user "client-xyz":
  T=0:   pageview /landing
  T=30:  pageview /signup  
  T=60:  custom event "signup_complete"
  T=120: pageview /add-payment
  T=180: custom event "payment_added"
  T=300: custom event "purchase_complete"

This user completed the entire funnel ✓.

Implementation (Spark batch job):

  Step 1: Group events by client_id, sort by timestamp
  Step 2: For each user, check if funnel steps occurred IN ORDER:
    - Did they visit /landing? YES
    - After that, did they trigger signup_complete? YES
    - After that, payment_added? YES
    - After that, purchase_complete? YES
  Step 3: Count users at each step
  
  Result:
    Step 1 (Landing):   100,000 users (100%)
    Step 2 (Sign Up):    25,000 users (25%)
    Step 3 (Payment):    10,000 users (10%)
    Step 4 (Purchase):    5,000 users (5%)
    
  Drop-off insights:
    75% drop off at signup → simplify signup form
    60% drop off at payment → add more payment options

  ClickHouse implementation:
    SELECT 
      countIf(step >= 1) AS landing,
      countIf(step >= 2) AS signup,
      countIf(step >= 3) AS payment,
      countIf(step >= 4) AS purchase
    FROM (
      SELECT client_id, 
        windowFunnel(86400)(timestamp, 
          url = '/landing',
          event = 'signup_complete',
          event = 'payment_added',
          event = 'purchase_complete'
        ) AS step
      FROM events
      WHERE timestamp > now() - INTERVAL 7 DAY
      GROUP BY client_id
    );
    
  windowFunnel() is a ClickHouse-native function for this exact use case.
```

### Privacy, GDPR, and Cookie-Less Tracking

```
GDPR / CCPA requirements:
  1. Consent: Must get explicit consent before tracking (EU users)
  2. Right to deletion: User requests deletion → purge ALL their events
  3. Data minimization: Don't collect more than necessary
  4. PII scrubbing: IP addresses, emails must be anonymized or not stored

Cookie-less tracking (post-iOS 14, post-3rd-party-cookie deprecation):
  
  Old way: Set a 3rd-party cookie → tracks user across sites → privacy violation
  
  New approaches:
    1. First-party cookie: Set cookie on YOUR domain → tracks user within your site only
       ✓ Still works, privacy-respecting
       ✗ Lost when user clears cookies or switches browsers
    
    2. Fingerprinting: Combine browser properties (screen size, timezone, 
       fonts, WebGL renderer) → pseudo-unique identifier
       ✗ Ethically questionable, browsers are blocking this
    
    3. Server-side tracking ⭐: 
       Collect events server-side (not in browser)
       User logs in → server tracks events with user_id
       No cookies needed, no ad blockers, full control
       ✗ Only works for logged-in users
    
    4. Aggregate-only analytics (Plausible/Fathom approach):
       No user-level tracking at all
       Count page views, not users
       ✓ No consent needed (no personal data collected)
       ✗ No funnels, no retention analysis, no user journeys

Deletion pipeline (Right to Erasure):
  User requests deletion → publish to Kafka topic "deletion-requests"
  → Batch job: DELETE FROM events WHERE client_id = ?
  → Delete from ClickHouse, S3 Parquet files, Redis caches
  → Challenge: Deleting from immutable Parquet files = rewrite entire partition
  → Solution: Mark as deleted (soft delete), physical deletion in weekly compaction job
```

