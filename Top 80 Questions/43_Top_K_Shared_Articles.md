# 43. Design Top K Most Shared Articles

---

## 1. Functional Requirements (FR)

- **Track shares**: Record every time an article/URL is shared (social media, messaging, email)
- **Top-K ranking**: Return the K most shared articles in a given time window (1h, 24h, 7d, all-time)
- **Real-time**: Rankings update within 1-5 minutes of new shares
- **Category filtering**: Top K per category (tech, sports, politics)
- **Regional**: Top K per country/region
- **Trending detection**: Highlight articles with fastest share velocity (not just highest absolute count)

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: 100K+ share events/sec
- **Low Latency**: Top-K query responds in < 50 ms
- **Approximate OK**: Exact ranking not required; approximate within 5% is acceptable
- **Scalability**: Handle billions of articles, millions of shares/day
- **Availability**: 99.99%

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Share events / day | 1B |
| Share events / sec | ~12K (peak 100K) |
| Unique articles shared / day | 10M |
| Top-K returned | K = 100 typically |
| Share event size | 100 bytes |
| Time windows | 1h, 24h, 7d, 30d |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Share Event Sources                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │ Social   │  │ Messaging│  │ Email    │  │ Copy-Link        │    │
│  │ Share    │  │ Share    │  │ Share    │  │ (in-app)         │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘    │
└───────┼──────────────┼──────────────┼─────────────────┼─────────────┘
        │              │              │                  │
        ▼              ▼              ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Kafka (share-events topic)                        │
│  Key: article_id    Value: {user_id, platform, region, timestamp}   │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼────────┐ ┌──▼──────────┐ ┌▼──────────────┐
     │  Window Counter │ │  Top-K      │ │ Persistence   │
     │  (Flink)        │ │  Aggregator │ │ Writer        │
     │                 │ │  (Flink)    │ │               │
     │  Maintains:     │ │             │ │ Batch write   │
     │  per (article,  │ │ Uses Space- │ │ to ClickHouse │
     │   region,       │ │ Saving algo │ │ every 10s     │
     │   category):    │ │ to maintain │ │               │
     │                 │ │ top 1000    │ │               │
     │  count_1h       │ │ per window  │ │               │
     │  count_24h      │ │ per region  │ │               │
     │  count_7d       │ │             │ │               │
     │                 │ │ Outputs to  │ │               │
     │  Using sliding  │ │ Redis every │ │               │
     │  windows with   │ │ 60 seconds  │ │               │
     │  panes          │ │             │ │               │
     └────────┬────────┘ └──────┬──────┘ └───────┬───────┘
              │                 │                 │
              └────────┬────────┘                 │
                       │                          │
              ┌────────▼────────┐        ┌───────▼───────┐
              │     Redis       │        │  ClickHouse   │
              │                 │        │               │
              │ Sorted Sets:    │        │ Share events  │
              │ topk:1h:US      │        │ for ad-hoc    │
              │ topk:24h:global │        │ queries and   │
              │ topk:7d:sports  │        │ backfill      │
              │                 │        │               │
              │ Score = count   │        │               │
              │ Member = article│        │               │
              └────────┬────────┘        └───────────────┘
                       │
              ┌────────▼────────┐
              │   Top-K API     │
              │                 │
              │ ZREVRANGE       │
              │  topk:24h:US   │
              │  0 99           │
              │  WITHSCORES     │
              └─────────────────┘
```

### Component Deep Dive

#### Multi-Window Counting with Panes

```
Problem: Maintaining exact counts for 1h, 24h, 7d windows simultaneously

Naive: Store every individual event → COUNT on each query → too slow

Pane-based approach ⭐:
  Divide time into 5-minute "panes"
  Each pane stores: count per article in that 5-min period
  
  1-hour count = sum of last 12 panes
  24-hour count = sum of last 288 panes
  7-day count = sum of last 2016 panes
  
  Every 5 minutes:
    - New pane created, oldest pane (outside window) dropped
    - Recompute top-K by summing relevant panes
  
  Memory: 10M articles × 4 bytes × 2016 panes = 80 GB → too much!
  
  Optimization: Only track articles with > 10 shares in any pane
    → ~100K active articles × 2016 panes = 800 MB → manageable
    Articles with < 10 shares can't be in top-K anyway
```

#### Approximate Top-K — Space-Saving Algorithm

```
Maintain a fixed-size map of K=1000 entries: {article_id → count}

On each share event:
  If article already in map → increment count
  If article not in map AND map has < K entries → add with count=1
  If article not in map AND map is full:
    Find entry with MINIMUM count (e.g., article-xyz with count=50)
    Replace it: article_id = new_article, count = 50 + 1
    (We "inherit" the evicted entry's count as an overestimate)

Guarantee: The true top-K is always contained in the result
Error bound: overestimate ≤ total_events / K
  With 1B events and K=1000: max overestimate = 1M
  For the top article with 10M shares → error < 10% → acceptable

Memory: O(K) = O(1000) regardless of how many articles exist
```

---

## 5. APIs

```http
GET /api/v1/top-articles?window=24h&region=US&category=tech&limit=50
→ 200 OK
{
  "articles": [
    {"article_id": "art-123", "url": "https://...", "title": "...", 
     "share_count": 152340, "rank": 1},
    ...
  ],
  "window": "24h",
  "computed_at": "2026-03-14T10:05:00Z"
}
```

---

## 6. Data Model

### Redis
```
Key:    topk:{window}:{region}:{category}
Type:   Sorted Set
Score:  share_count
Member: article_id
Ops:    ZREVRANGE for top-K, ZADD for updates
TTL:    Refreshed every 60s by Flink

Key:    article_meta:{article_id}
Type:   Hash
Fields: url, title, image_url, publisher, category
```

### ClickHouse
```sql
CREATE TABLE share_events (
    article_id   String,
    user_id      String,
    platform     LowCardinality(String),
    region       LowCardinality(String),
    category     LowCardinality(String),
    shared_at    DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(shared_at)
ORDER BY (region, category, shared_at);

-- Materialized view for pre-aggregation
CREATE MATERIALIZED VIEW share_counts_hourly
ENGINE = SummingMergeTree()
ORDER BY (article_id, region, category, hour)
AS SELECT
    article_id, region, category,
    toStartOfHour(shared_at) AS hour,
    count() AS share_count
FROM share_events
GROUP BY article_id, region, category, hour;
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Flink failure** | Checkpoint to S3 every 60s; resume from checkpoint |
| **Redis loss** | Flink recomputes full top-K within 60 seconds from pane state |
| **Late events** | Flink watermark allows 5-min late arrivals; beyond that → counted in next window |
| **Article metadata missing** | Async enrichment; display URL if title unavailable |
| **Spam shares** | Rate limit per user (max 100 shares/hour); bot detection |

---

## 8. Deep Dive: Engineering Trade-offs

### Exact vs Approximate Top-K

```
Exact: Maintain count for ALL articles → sort → take top K
  Memory: O(N) where N = unique articles (10M+)
  Sort: O(N log N) every update cycle
  ✓ Perfectly accurate
  ✗ Expensive at scale

Approximate (Space-Saving) ⭐:
  Memory: O(K) where K = 1000
  Update: O(1) per event
  ✓ Bounded memory, fast
  ✗ Overestimates counts for lower-ranked items
  ✗ Rarely: true top-K item evicted if it goes cold then hot again

Hybrid:
  Tier 1: Count-Min Sketch filters topics with > threshold mentions
  Tier 2: Exact HashMap for promoted topics (~10K active)
  Tier 3: Sort top 10K → take top 100
  → Exact for top-100, approximate for detection
```

### MapReduce vs Stream Processing for Top-K

```
Batch (Spark MapReduce):
  Every hour: scan all events in last 24h → aggregate → rank
  Latency: minutes to hours
  ✓ Exact results, handles late data perfectly
  ✗ Stale (up to 1 hour old)

Stream (Flink) ⭐:
  Continuous processing: each event immediately updates counters
  Latency: seconds
  ✓ Near real-time trending
  ✗ Late events may be missed or counted in wrong window
  ✗ State management complexity

Best: Flink for real-time (good enough accuracy) + hourly Spark job to reconcile exact counts
  Flink results shown to users; Spark results used for analytics and ground truth.
```

