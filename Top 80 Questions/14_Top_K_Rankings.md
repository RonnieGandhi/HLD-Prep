# 14. Design Top K Rankings System (App Store / Amazon Bestsellers)

---

## 1. Functional Requirements (FR)

- Compute and display top K items (K = 10, 50, 100) ranked by a metric (sales, downloads, ratings, revenue)
- Support multiple ranking dimensions: overall, by category, by time period (daily, weekly, monthly, all-time)
- Near real-time updates: rankings reflect recent activity within minutes
- Support historical rankings: "What was #1 last Tuesday?"
- Handle large scale: millions of items, billions of events (purchases, downloads)
- Expose ranked lists via API for clients

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Return top K list in < 50 ms
- **High Availability**: 99.99%
- **Scalability**: Process billions of ranking events per day
- **Accuracy**: Precise top K for small K (top 100); approximate OK for larger K
- **Freshness**: Rankings updated every 1-5 minutes
- **Read-Heavy**: Millions of users viewing rankings, few writing events

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total items | 5M |
| Events (purchases/downloads) / day | 1B |
| Events / sec | ~12K (peak 60K) |
| Event size | 100 bytes |
| Top K lists to maintain | 1000 (categories × time periods) |
| List storage | 1000 × 100 entries × 200B = 20 MB |
| Event storage / day | 1B × 100B = 100 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐     ┌──────────────┐
│  Purchase    │     │  Download    │
│  Service     │     │  Service     │
└──────┬───────┘     └──────┬───────┘
       │                     │
       └─────────┬───────────┘
                 │
          ┌──────▼───────┐
          │    Kafka      │  ← ranking-events topic
          │               │
          └──────┬────────┘
                 │
       ┌─────────┤
       │         │
┌──────▼──────┐  │
│ Apache Flink│  │  ← Real-time aggregation
│ (Stream     │  │
│ Processing) │  │
└──────┬──────┘  │
       │         │
┌──────▼──────┐  │
│   Redis     │  │  ← Sorted Sets for top K
│  (Ranking   │  │
│   Cache)    │  │
└──────┬──────┘  │
       │         │
┌──────▼──────┐  │  ┌──────────────┐
│  Ranking    │  │  │  Batch       │
│  Service    │  │  │  Pipeline    │  ← Hourly Spark job for recalibration
│  (API)      │  │  │  (Spark)     │
└─────────────┘  │  └──────┬───────┘
                 │         │
                 │  ┌──────▼──────┐
                 └──▶  Cassandra  │  ← Historical rankings + raw events
                    │  / ClickHouse│
                    └─────────────┘
```

### Component Deep Dive

#### Stream Processing (Apache Flink) — The Core Engine

**Approach 1: Exact Count (for small-medium scale)**
```
Flink Pipeline:
  Source(Kafka: ranking-events)
  → KeyBy(item_id)
  → Tumbling/Sliding Window(5 minutes)
  → Aggregate(count per item_id)
  → Top K (min-heap of size K)
  → Sink(Redis sorted set)
```

**Approach 2: Approximate Count (for massive scale) — Count-Min Sketch + Heap**

When you have millions of unique items, maintaining exact counts for all is expensive.

**Count-Min Sketch**:
- Probabilistic data structure: d hash functions × w counters (matrix)
- On event for item X: hash with each of d functions, increment d counters
- To query count of X: take minimum of d counter values
- **Space**: O(w × d) — typically 2 MB for < 0.1% error rate
- **Error**: Always overcounts (never undercounts); error bounded by ε = e/w

**Min-Heap for Top K**:
```java
// Maintain a min-heap of size K
// For each incoming event:
if heap.size < K:
    heap.add(item)
elif item.count > heap.peek().count:
    heap.poll()
    heap.add(item)
```

**Combined approach**:
1. Count-Min Sketch tracks approximate counts for ALL items (memory efficient)
2. Min-Heap of size K maintains the current top K items
3. When a new event arrives:
   - Increment item count in Count-Min Sketch
   - Check if item's new count qualifies it for top K heap
   - If yes → add/update in heap
4. Periodically flush heap to Redis

#### Time-Windowed Rankings

```
Rankings needed:
  - Hourly top K  → Sliding window, 1 hour, slide every 5 min
  - Daily top K   → Tumbling window, 24 hours
  - Weekly top K  → Tumbling window, 7 days
  - All-time top K → Cumulative counter (no window)
```

**Exponential Decay for "Trending"**:
Instead of fixed windows, use exponential time decay:
```python
score = Σ (event_value × e^(-λ × age_in_hours))
# λ = decay rate (e.g., 0.1 → half-life ≈ 7 hours)
# Recent events contribute more than old events
```

#### Redis (Ranking Cache)

- **Sorted Set** per ranking list:
  ```
  Key:    ranking:{category}:{period}
  Type:   Sorted Set
  Members: item_id
  Scores:  count / score
  ```
- `ZREVRANGE ranking:games:daily 0 9` → Top 10 games today
- `ZREVRANK ranking:overall:weekly item123` → What's item123's rank this week?
- Update atomically: `ZADD ranking:games:daily item123 15432`

#### Batch Pipeline (Spark) — Reconciliation
- Hourly/daily Spark job reads raw events from Cassandra/S3
- Computes exact rankings (no approximation)
- Reconciles with real-time rankings in Redis (corrects Count-Min Sketch drift)
- Writes historical rankings to Cassandra for "ranking at time T" queries

---

## 5. APIs

### Get Top K
```http
GET /api/v1/rankings?category=games&period=daily&limit=50
Response: 200 OK
{
  "category": "games",
  "period": "daily",
  "as_of": "2026-03-13T10:00:00Z",
  "rankings": [
    {"rank": 1, "item_id": "app123", "name": "Puzzle Master", "score": 152000, "change": "+2"},
    {"rank": 2, "item_id": "app456", "name": "Word Rush", "score": 148500, "change": "-1"},
    ...
  ]
}
```

### Get Item Rank
```http
GET /api/v1/rankings/item/{item_id}?category=games&period=weekly
Response: 200 OK
{
  "item_id": "app123",
  "ranks": {
    "overall": 15,
    "category_games": 3,
    "daily": 1,
    "weekly": 5
  }
}
```

### Get Historical Rankings
```http
GET /api/v1/rankings/history?category=games&date=2026-03-01&limit=10
```

---

## 6. Data Model

### Kafka Topic: `ranking-events`

```json
{
  "event_id": "uuid",
  "event_type": "purchase",
  "item_id": "app123",
  "category": "games",
  "timestamp": "2026-03-13T10:00:00Z",
  "value": 1,
  "metadata": {"country": "US", "price": 2.99}
}
```

### Redis — Ranked Lists

```
Key:    ranking:overall:daily
Type:   Sorted Set
Members: item_id
Scores:  event_count (or revenue, or weighted score)

Key:    ranking:games:weekly
Type:   Sorted Set
...
```

### Cassandra — Historical Rankings

```sql
CREATE TABLE historical_rankings (
    category    TEXT,
    period      TEXT,          -- 'daily', 'weekly'
    snapshot_date DATE,
    rank        INT,
    item_id     TEXT,
    score       BIGINT,
    PRIMARY KEY ((category, period), snapshot_date, rank)
) WITH CLUSTERING ORDER BY (snapshot_date DESC, rank ASC);
```

### Cassandra — Raw Events (for batch reconciliation)

```sql
CREATE TABLE ranking_events (
    event_date  DATE,
    event_hour  INT,
    event_id    UUID,
    item_id     TEXT,
    event_type  TEXT,
    category    TEXT,
    value       INT,
    timestamp   TIMESTAMP,
    PRIMARY KEY ((event_date, event_hour), event_id)
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Flink failure** | Checkpointing to S3 every 30s; restart from last checkpoint |
| **Redis data loss** | Batch pipeline can reconstruct rankings from raw events |
| **Count-Min Sketch drift** | Hourly reconciliation via Spark batch job |
| **Event loss** | Kafka with RF=3 ensures no events lost |
| **Stale rankings** | Serve from Redis (stale by at most window size); acceptable |

### Specific: Handling "Rank Manipulation"
- Fake purchases/downloads to boost rankings
- **Defenses**:
  - Only count verified purchases (not free downloads with refunds)
  - Device fingerprinting to detect bot farms
  - Velocity checks: sudden spike in downloads → flag for review
  - Exclude events from known fraud accounts

---

## 8. Additional Considerations

### Lambda Architecture (Real-time + Batch)
```
Speed Layer:  Flink (real-time, approximate) → Redis
Batch Layer:  Spark (exact, hourly) → Cassandra → Redis
Serving Layer: Redis (merged view)
```

### Multi-Dimensional Rankings
```
Dimensions:
  - Category (games, productivity, education, ...)
  - Geography (US, UK, India, ...)
  - Time period (hourly, daily, weekly, monthly, all-time)
  - Metric (downloads, revenue, ratings, active users)
  
Total ranking lists = categories × geographies × periods × metrics
e.g., 50 × 20 × 5 × 4 = 20,000 ranking lists
Each maintained as a Redis Sorted Set
```

### Heavy Hitters Problem
The Top-K problem is related to the "Heavy Hitters" problem in streaming algorithms:
- **Misra-Gries Algorithm**: Maintains at most K-1 candidates using O(K) space
- **Space-Saving Algorithm**: Keeps top K counters, replaces the minimum when a new item arrives
- **Lossy Counting**: Maintains approximate counts with guaranteed error bound

For system design interviews, **Count-Min Sketch + Min-Heap** is the most common and practical answer.

