# 89. Design a Real-Time Bidding System (Ad Tech)

---

## 1. Functional Requirements (FR)

- When a user loads a webpage with ad slots, conduct a real-time auction in < 100ms
- Send bid requests to multiple demand-side platforms (DSPs) simultaneously
- Each DSP evaluates the user and returns a bid price + ad creative
- Select the highest bidder, render their ad, record the impression
- Support multiple ad formats: display, video, native, rich media
- Frequency capping: limit how many times a user sees the same ad
- Budget management: stop bidding when advertiser daily/lifetime budget exhausted
- Win notification: notify winning DSP so they can track spend
- Click and conversion tracking with attribution
- Fraud detection: filter bot traffic, invalid clicks, ad stacking

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: End-to-end auction in < 100ms (bid request → ad rendered)
- **Massive Throughput**: 10M+ auctions/sec globally
- **High Availability**: 99.99% — failed auction = lost revenue
- **Consistency**: Budget enforcement must prevent overspend (eventual OK within ~5%)
- **Geographic Distribution**: Edge servers in every major market
- **Auditability**: Full trail for every auction (regulatory compliance)

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Auctions / sec | | 10M |
| DSPs per auction | avg | 5–20 |
| Bid requests / sec (outbound) | 10M × 10 | 100M |
| Bid response time SLA | | < 50ms |
| Win notifications / sec | ~50% fill rate | 5M |
| Impressions / day | 10M × 86400 × 50% | 432B |
| Click-through rate | 0.1% | 432M clicks/day |
| Auction log size / day | 10M/sec × 1KB × 86400 | ~864 TB/day |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PUBLISHER SIDE                                      │
│                                                                        │
│  User loads webpage → Ad tag (JavaScript) fires                       │
│  → Sends ad request to SSP (Supply-Side Platform)                     │
│  Contains: user_id/cookie, page_url, ad_slot_size, geo, device        │
└───────────────────────┬───────────────────────────────────────────────┘
                        │ HTTP/2 (< 10ms to nearest edge)
┌───────────────────────▼───────────────────────────────────────────────┐
│               SSP / AD EXCHANGE (Our System)                           │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  Ad Request Handler (Edge Server, Stateless)                 │      │
│  │  1. Parse ad request, extract user signals                   │      │
│  │  2. Cookie sync: map publisher cookie → internal user ID     │      │
│  │  3. Enrich with user profile (Redis lookup, < 2ms)           │      │
│  │  4. Check floor price for publisher                          │      │
│  │  5. Select eligible DSPs (targeting match, budget check)     │      │
│  │  6. Fan-out bid requests to DSPs (parallel HTTP/2, 50ms TO)  │      │
│  │  7. Collect responses, run auction                           │      │
│  │  8. Return winning ad creative to publisher                  │      │
│  └────────┬───────────────────────────────────┬─────────────────┘      │
│           │                                   │                        │
│  ┌────────▼─────────────┐        ┌────────────▼──────────────┐        │
│  │  User Data Service   │        │  Bid Fan-Out Engine        │        │
│  │  (Redis Cluster)     │        │                            │        │
│  │                      │        │  Parallel HTTP/2 to DSPs:  │        │
│  │  - User segments     │        │  ┌─────┐ ┌─────┐ ┌─────┐  │        │
│  │  - Interest profile  │        │  │DSP 1│ │DSP 2│ │DSP 3│  │        │
│  │  - Frequency caps    │        │  │ $2.5│ │ $3.1│ │ $1.8│  │        │
│  │  - Cookie map        │        │  └──┬──┘ └──┬──┘ └──┬──┘  │        │
│  │  Lookup: < 2ms       │        │     └───────┼───────┘      │        │
│  └──────────────────────┘        │      ┌──────▼──────┐       │        │
│                                  │      │  Auction    │       │        │
│                                  │      │  Engine     │       │        │
│                                  │      │ 2nd price:  │       │        │
│                                  │      │ Winner=DSP2 │       │        │
│                                  │      │ Price=$2.51 │       │        │
│                                  │      └──────┬──────┘       │        │
│                                  └─────────────│──────────────┘        │
│                                                │                       │
│  ┌─────────────────────────────────────────────▼────────────────┐      │
│  │  Post-Auction Processing (async where possible)              │      │
│  │  1. Send win notification to DSP 2 (async HTTP)             │      │
│  │  2. Update frequency cap (Redis INCR, per user+campaign)    │      │
│  │  3. Update budget consumed (Redis + async reconcile to DB)  │      │
│  │  4. Log auction to Kafka (billing, analytics, fraud)        │      │
│  │  5. Return ad markup (HTML/JS) to publisher                 │      │
│  └─────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐      │
│  │  Async Pipeline (Kafka → Flink → ClickHouse)                │      │
│  │                                                              │      │
│  │  Kafka topics:                                               │      │
│  │  - auction-logs: every auction (win/loss/no-bid per DSP)    │      │
│  │  - impressions: rendered ads (billing event)                 │      │
│  │  - clicks: user clicked ad (attribution)                    │      │
│  │  - conversions: user completed action (purchase, signup)     │      │
│  │                                                              │      │
│  │  Flink jobs:                                                 │      │
│  │  - Budget tracking: real-time spend per campaign             │      │
│  │  - Fraud detection: bot patterns, click farms, IVT scoring  │      │
│  │  - Billing aggregation: hourly rollups by advertiser         │      │
│  │  - CTR/CVR computation: for model training data              │      │
│  │                                                              │      │
│  │  ClickHouse: OLAP for analytics dashboard                   │      │
│  │  - Spend by campaign, CTR by segment, fill rate by pub      │      │
│  └─────────────────────────────────────────────────────────────┘      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Auction Engine
- **Second-Price**: Bids $3.10, $2.50, $1.80 → Winner=$3.10 bidder pays $2.51 (2nd+$0.01). Incentive: bid true value
- **First-Price** (industry standard since 2019): Winner pays their bid. Challenge: bid shading (DSPs use ML to determine optimal bid)
- **Header Bidding**: Publisher sends to multiple SSPs simultaneously → each runs internal auction → publisher's ad server picks overall winner

#### Latency Budget Breakdown

```
Total budget: 100ms (page load to ad rendered)
  Ad request to SSP edge:     5ms
  User data lookup (Redis):   2ms
  Bid request to DSPs:        5ms (parallel HTTP/2)
  DSP processing + response:  30ms (DSP timeout: 50ms)
  Auction logic:              1ms
  Win notification (async):   0ms
  Ad markup to publisher:     5ms
  Ad render in browser:       ~50ms (client-side)
  Server-side total: ~48ms → within 100ms budget
```

#### Budget Enforcement at Scale

```
Problem: 10M auctions/sec, budget check must prevent overspend
  Multiple edge servers deducting from same budget simultaneously

Solution: Tiered budget enforcement
  Tier 1 (hot path, Redis):
    Each edge server gets a "budget slice" from central budget
    Check local slice only → no cross-server coordination
    When slice exhausted → request new slice from central
  
  Tier 2 (near real-time, Flink):
    Aggregate all win notifications per campaign
    If spend approaches limit → signal "stop bidding" to edge servers
    Latency: ~5 seconds behind real-time
  
  Tier 3 (daily reconciliation):
    ClickHouse aggregation → reconcile against DSP reports
    Handle discrepancies (different counting methodologies)

  Overspend tolerance: ~2-5% (industry standard, contractually agreed)
```

---

## 5. APIs

### OpenRTB Bid Request (Industry Standard)

```json
POST /bid (sent to each DSP, parallel)
{
  "id": "auction-uuid-123",
  "imp": [{
    "id": "1",
    "banner": {"w": 300, "h": 250},
    "bidfloor": 0.50,
    "bidfloorcur": "USD"
  }],
  "site": { "domain": "example.com", "page": "https://example.com/article/123" },
  "user": { "id": "user-cookie-hash", "geo": {"country": "US", "city": "San Francisco"} },
  "device": { "ua": "Mozilla/5.0...", "ip": "203.0.113.42", "os": "iOS" },
  "tmax": 50
}
```

### Bid Response (from DSP)

```json
{
  "seatbid": [{ "bid": [{
    "id": "bid-456", "impid": "1", "price": 3.10,
    "adm": "<div>...ad creative HTML...</div>",
    "nurl": "https://dsp.example.com/win?price=${AUCTION_PRICE}",
    "crid": "creative-789"
  }] }]
}
```

### Internal APIs

```
GET  /api/campaigns/{id}/budget    → Current spend vs limit
POST /api/campaigns/{id}/pause     → Pause bidding for campaign
GET  /api/analytics/spend?campaign=...&date=...  → Spend analytics
```

---

## 6. Data Models

### Redis (Hot Path — Low Latency)

```
# User profile (bid request enrichment)
HSET user:{cookie_hash} segments "sports,tech" age_range "25-34"  TTL: 30d

# Frequency cap (per user per campaign)
INCR freq:{user_id}:{campaign_id}:{date}  TTL: 24h
  Check: GET < max_freq → allow bid

# Budget tracking (per campaign, real-time)
HSET budget:{campaign_id} daily_spent 4523.50 daily_limit 5000.00
  If daily_spent >= daily_limit → don't bid for this campaign
```

### ClickHouse (Analytics — Columnar)

**Why ClickHouse?** Columnar storage, 100× faster than MySQL for OLAP aggregations on billions of rows.

```sql
CREATE TABLE auction_logs (
    auction_id    UUID,
    timestamp     DateTime,
    publisher_id  UInt64,
    ad_slot_size  String,
    user_id       String,
    country       LowCardinality(String),
    device_type   LowCardinality(String),
    num_bids      UInt8,
    winning_dsp   LowCardinality(String),
    winning_price Float64,
    second_price  Float64,
    floor_price   Float64,
    campaign_id   UInt64,
    is_click      UInt8,
    is_conversion UInt8
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (publisher_id, timestamp);
```

### Kafka Topics

```
auction-logs:   partitioned by publisher_id, RF=3, retention 7d
impressions:    partitioned by campaign_id, RF=3, retention 30d
clicks:         partitioned by campaign_id, RF=3, retention 90d
```

---

## 7. Fault Tolerance

| Technique | Application |
|---|---|
| DSP timeout (50ms) | If DSP doesn't respond → excluded from auction |
| No-bid fallback | If no DSPs respond → serve house ad or empty slot |
| Kafka RF=3 | Auction logs, impressions survive broker failure |
| Redis Cluster | User data + budget survives node failure |
| Edge redundancy | Multiple edge servers per region; LB health checks |
| Budget reconciliation | Daily reconcile catches any tracking drift |

### Problem-Specific

**DSP Failure**: If DSP consistently times out → circuit breaker stops sending bid requests for 60s → retry probe.

**Budget Race Condition**: Solved by budget slicing (see above). Edge servers independently check their allocated slice → no coordination needed in hot path.

**Fraud Detection**: 
- Real-time (in auction, <5ms): IP blocklist, bot UA detection, IVT score
- Near real-time (Flink): CTR anomalies (CTR>10% = suspicious), click timing (<100ms = bot), geo anomalies
- Batch (daily): Click farm patterns, publisher traffic quality scoring

**Auction Log Durability**: Async log to Kafka. If Kafka slow → buffer locally, retry. Auction result never blocked by logging.

---

## 8. Additional Considerations

### How a DSP Bidder Works (< 50ms)
```
1. Cookie sync: map SSP user ID → DSP user profile (2ms)
2. Campaign matching: which campaigns target this user? (5ms)
3. Bid price prediction: ML model → optimal bid (10ms)
4. Budget check: can this campaign afford to bid? (2ms)
5. Frequency check: has user seen this ad too many times? (2ms)
6. Select creative: pick best ad for user/context (2ms)
7. Return bid response (1ms)
```

### Why This Problem Is Unique in System Design
```
- Most extreme latency: 100ms total, 50ms DSP SLA
- Highest throughput: 10M auctions/sec (exceeds most systems' read QPS)
- Budget = real money: 0.1% overspend on $100M = $100K loss
- Adversarial: fraud actors gaming the system
- Privacy: GDPR/CCPA consent, data deletion, cookieless tracking trends
```

