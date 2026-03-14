# 42. Design a Trending Topics System

---

## 1. Functional Requirements (FR)

- **Detect trends**: Identify topics/hashtags/keywords with abnormal surge in volume
- **Real-time**: Trends update every 1-5 minutes
- **Ranking**: Show Top 10/20/50 trending topics, ranked by velocity of growth
- **Personalization**: Regional/country-specific trends, optionally personalized
- **Context**: Show why it's trending ("50K tweets about #WorldCup")
- **Trend lifecycle**: Detect start, peak, and decay of trends
- **Filter abuse**: Prevent spam/manipulation from gaming trends

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Trend detection within 1-5 minutes of surge start
- **High Throughput**: Process 500K+ events/sec (tweets, posts, searches)
- **Scalability**: Handle billions of events/day across 100+ countries
- **Accuracy**: Distinguish genuine trends from spam/bot campaigns
- **Availability**: 99.99%

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Events / sec (tweets + searches + posts) | 500K |
| Unique topics / hour | 10M |
| Trending topics displayed | Top 20 per region |
| Regions | 200+ countries |
| Topic velocity window | 5-minute sliding window |
| Storage for trend history | 10 GB/day |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Event Sources                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    │
│  │ Tweets   │  │ Searches │  │ Posts    │  │ News articles    │    │
│  │ (text +  │  │ (queries)│  │ (social) │  │ (RSS/webhooks)   │    │
│  │ hashtags)│  │          │  │          │  │                  │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘    │
└───────┼──────────────┼──────────────┼─────────────────┼─────────────┘
        ▼              ▼              ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│               Kafka (raw-events, partitioned by region)              │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼────────┐ ┌──▼──────────┐  │
     │  Topic Extractor│ │ Spam/Bot    │  │
     │  (Flink)        │ │ Filter      │  │
     │                 │ │             │  │
     │  1. Extract     │ │ • Account   │  │
     │     hashtags    │ │   age check │  │
     │  2. NER/NLP     │ │ • Bot score │  │
     │     entities    │ │   ML model  │  │
     │  3. Keywords    │ │ • Content   │  │
     │  4. Normalize   │ │   similarity│  │
     │     (lowercase, │ │   (SimHash) │  │
     │      lemmatize) │ │ • Velocity  │  │
     └────────┬────────┘ │   anomaly   │  │
              │          └──────┬──────┘  │
              └────────┬────────┘         │
                       ▼                  │
              ┌───────────────────────────▼────────────────────────────┐
              │           Trend Detection Engine (Flink)               │
              │                                                        │
              │  Per (topic, region) — keyed stream:                   │
              │                                                        │
              │  1. Count in sliding 5-min window (slide=1 min)       │
              │     V_current = count(topic, region, last 5 min)      │
              │                                                        │
              │  2. Load baseline from state                           │
              │     V_baseline = avg(same hour, same day-of-week,     │
              │                  past 4 weeks)                         │
              │                                                        │
              │  3. Compute velocity (anomaly score):                  │
              │     velocity = (V_current - V_baseline)                │
              │               / max(V_baseline, 10)                   │
              │     Threshold: velocity > 2.0 → TRENDING              │
              │                                                        │
              │  4. Rank by velocity (NOT absolute volume)             │
              │     Niche topic 10→5000 (500×) ranks HIGHER           │
              │     than popular topic 100K→200K (2×)                 │
              │                                                        │
              │  5. Spam penalty: if >30% from new accounts → ×0.2   │
              │                                                        │
              │  → Output: Top 50 per region, emitted every 60s       │
              └────────────────────────┬──────────────────────────────┘
                                       │
                          ┌────────────▼────────────┐
                          │     Redis (Hot Store)    │
                          │                          │
                          │  trending:{region}       │
                          │  = Sorted Set (velocity) │
                          │                          │
                          │  topic_ctx:{topic}       │
                          │  = Hash {count, started, │
                          │    category, samples}    │
                          └────────────┬─────────────┘
                                       │
                    ┌──────────────────▼──────────┐
                    │       Trend API Service      │
                    │  GET /trends?region=US       │
                    │  → ZREVRANGE trending:US 0 19│
                    └──────────────────┬──────────┘
                                       │
                    ┌──────────────────▼──────────┐
                    │   ClickHouse (Trend History) │
                    │   Archive + analytics        │
                    └─────────────────────────────┘
```

### Component Deep Dive

#### Velocity Over Volume — The Core Algorithm

```
Why velocity, not raw count?

  "Taylor Swift" always has 500K mentions/hour → NOT trending (normal)
  "#ObscureEvent" normally has 10/hour, today has 5000 → TRENDING!

Formula:
  velocity = (V_current - V_baseline) / max(V_baseline, min_floor=10)
  
  Example:
    Topic A: V_current=200K, V_baseline=100K → velocity = 1.0  (2× normal)
    Topic B: V_current=5000,  V_baseline=10  → velocity = 499  (500× normal!)
    
  Topic B ranks higher despite lower absolute count.
  This surfaces genuinely surprising/newsworthy events.

Decay: After spike, V_baseline catches up → velocity drops naturally
  Lifecycle: Emerging → Peak → Decaying → Normal
```

#### Spam & Manipulation Detection

```
Signals computed per topic in the Flink pipeline:

  1. Account age: >30% from accounts <7 days old → score ×0.2
  2. Account diversity: >50% from <100 unique accounts → suppress
  3. Content similarity (SimHash): >60% near-duplicate text → suppress
  4. Velocity shape: organic = gradual ramp; bots = instant step function
  5. Geographic anomaly: claimed region doesn't match account timezones
```

#### Baseline Management

```
Seasonality model: day-of-week × hour-of-day

  Redis key: baseline:{topic}:{day_of_week}:{hour}
  Updated weekly by Spark batch job:
    avg(volume at this day+hour over past 4 weeks)
  
  Example: "#MondayMotivation"
    Monday 8am baseline: 50K (always spikes)
    Tuesday 8am baseline: 2K
    → Monday 55K → velocity=0.1 → NOT trending (normal Monday)
    → Tuesday 20K → velocity=9.0 → TRENDING (unusual for Tuesday!)

Cold-start (new topic, no history): min_floor=10 as baseline
```

---

## 5. APIs

### Get Trending Topics
```http
GET /api/v1/trends?region=US&count=20
→ 200 OK
{
  "trends": [
    { "rank": 1, "topic": "#WorldCup2026", "volume": "2.3M",
      "velocity": 15.2, "category": "Sports",
      "started_at": "2026-03-14T08:00:00Z",
      "context": "Quarter-finals underway" },
    ...
  ],
  "as_of": "2026-03-14T10:05:00Z"
}
```

---

## 6. Data Model

### Redis
```
trending:{region}                    → Sorted Set (score=velocity, member=topic)
topic_ctx:{topic}                    → Hash {volume, started_at, category, samples}
baseline:{topic}:{day_of_week}:{hour} → Integer (rolling avg)
```

### Flink State
```
Per (topic, region):
  current_window_count:  int
  ring_buffer[12]:       int[] (last 12 five-min windows)
  last_emit_ts:          long
Checkpoint: S3, every 60s
```

### ClickHouse
```sql
CREATE TABLE trend_snapshots (
    region      LowCardinality(String),
    topic       String,
    velocity    Float64,
    volume      UInt64,
    rank        UInt16,
    snapshot_at DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(snapshot_at)
ORDER BY (region, snapshot_at, rank);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Flink failure** | Checkpoint to S3 every 60s; resume from last checkpoint |
| **Redis loss** | Flink recomputes full top-K within 60 seconds |
| **Kafka lag** | Trends delayed proportionally; alert if lag > 2 min |
| **Spam spike** | Inline bot filter penalizes before scoring |
| **Topic explosion** | Only track topics with >100 mentions/hour; prune below |

### Race Conditions

#### Window Boundary Splitting
```
Tumbling window: [10:00, 10:05), [10:05, 10:10)
Spike at 10:04→10:06 → each window sees only HALF

Fix: Sliding window (5 min wide, 1 min slide) ⭐
  At T=10:06: window [10:01, 10:06] captures both halves
  Cost: 5× more state (5 overlapping windows). Worth it.
```

#### Hot Key Problem
```
Super Bowl → US region has 10× more events → consumer partition lags

Fix: Sub-partition hot regions (US-east, US-west, US-central)
  Merge in a second Flink stage.
  Or: key by (region, topic_hash % 100) → 100 sub-partitions
```

#### Stale Trend — Topic That Should Have Decayed But Didn't

```
"#MondayMotivation" spikes every Monday morning (expected).
But if baseline hasn't been updated in 2 weeks (Spark job failed):
  → Old baseline is too low → appears as trending every Monday → wrong!

Detection: Monitor baseline freshness
  If baseline is > 14 days old → use conservative fallback:
    fallback_baseline = max(V_baseline, V_current × 0.5)
    This prevents anything from appearing as more than 2× "normal"

Fix: Alert on Spark baseline job failures; auto-retry with backoff
```

#### Flink Checkpoint Recovery — Events Replayed Twice

```
Flink checkpoints at T=10:00 (offset=1000)
Processes events 1001-1500
Crashes at T=10:03 before next checkpoint

Recovery: restart from offset 1000 → events 1001-1500 replayed!
  → Counts for topics in those events are DOUBLED

Solution: Flink's exactly-once guarantee with Kafka transactions
  Flink writes to Kafka output topic + commits checkpoint atomically
  If crash → events re-consumed but duplicates detected via transaction
  
  For Redis output: idempotent ZADD (same score overwritten, not doubled)
  For counting: use Flink's managed state (restored from checkpoint, not Redis)
```

#### Trend Suppression — Sensitive Content

```
Breaking news: mass shooting → topic trends instantly
Problem: Surfacing this as "trending" feels exploitative

Content sensitivity pipeline:
  1. Keyword blocklist: certain terms auto-suppressed from trending
  2. Sentiment analysis: if topic sentiment is overwhelmingly negative
     AND contains sensitive keywords → flag for review
  3. Human review: on-call Trust & Safety team can manually suppress trends
  4. Editorial override: trending list can be manually curated during crises
  
Twitter's approach: "Trending with context"
  Instead of just "#shooting" → show "Mass shooting at X: law enforcement responding"
  Adds responsible context rather than raw hashtag
```
  Or: key by (region, topic_hash % 100) → 100 sub-partitions
```

---

## 8. Deep Dive: Engineering Trade-offs

### Exact Counting vs Count-Min Sketch

```
Exact HashMap: O(N) memory, N=10M unique topics → 2 GB per worker
Count-Min Sketch: Fixed 200 KB, approximate (overestimates only)

Hybrid "Sketch + Promote" ⭐:
  1. CMS for ALL topics (200 KB)
  2. If CMS estimate > threshold → promote to exact HashMap
  3. Only ~10K promoted topics tracked exactly
  4. Top-K from exact counters → accurate ranking
  Memory: 2.2 MB total vs 2 GB → 1000× reduction
```

### Flink vs Spark Streaming vs Kafka Streams

| Feature | Flink ⭐ | Spark Streaming | Kafka Streams |
|---|---|---|---|
| **Latency** | True streaming (ms) | Micro-batch (seconds) | True streaming (ms) |
| **Windowing** | Rich (sliding, session) | Basic | Basic |
| **State** | RocksDB, checkpointed | Spark state | RocksDB |
| **Best for** | Complex event processing | Batch+stream hybrid | Simple pipelines |

Choose Flink: needs sliding windows, low latency, complex per-key state.

### Personalized Trends

```
Standard trends: Top 20 for a region (same for everyone in US)
Personalized trends: Mix in topics relevant to the user's interests

Implementation:
  User interests = topics they follow + topics they engage with
  For each trending topic:
    relevance = cosine_similarity(topic_embedding, user_interest_embedding)
  Final_rank = w1 × velocity + w2 × relevance
  
  Example: User follows #Tech and #AI
    #WorldCup trending (velocity=15, relevance=0.1) → final=1.6
    #GPT5Release trending (velocity=5, relevance=0.9) → final=2.3
    → User sees #GPT5Release higher (personalized!)

  Pre-computed: user interest embeddings updated daily via Spark
  Real-time: re-rank top 50 regional trends using user embedding (< 10ms)
```

### Trend Category Auto-Classification

```
Categorize trending topics automatically: Sports, Politics, Entertainment, Tech, etc.

Approach:
  1. Named Entity Recognition (NER): "Taylor Swift" → entity type = PERSON
  2. Topic model: LDA / BERTopic on sample tweets mentioning the topic
  3. Knowledge graph lookup: "Taylor Swift" → Wikidata → category = musician
  4. Hashtag taxonomy: #WorldCup → manually curated mapping → Sports
  
  Fallback: If no category detected → "Trending" (uncategorized)
  
  Used for: filtering ("show only Sports trends"), display context
```

### Multi-Language Trend Merging

```
Same event trending in multiple languages:
  "#CopaDelMundo" (Spanish), "#WorldCup" (English), "#ワールドカップ" (Japanese)

Should these merge into ONE trend or stay separate?

Approach: Entity-based merging
  1. NER extracts entities: all three map to entity "FIFA World Cup 2026"
  2. Merge counts across language variants
  3. Display: use the user's language variant but aggregate volume

Without merging: Same event appears 3 times in trending → redundant
With merging: One entry showing "2.3M tweets" (across all languages) → cleaner

Implementation: entity_id as the trend key, not the raw hashtag text
  Flink: group by entity_id (resolved via a pre-built entity dictionary)
```

