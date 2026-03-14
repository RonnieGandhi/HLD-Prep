# 41. Design Like Count for High Profile Users

---

## 1. Functional Requirements (FR)

- **Like/Unlike**: Users can like/unlike posts, photos, videos
- **Display count**: Show total like count on each piece of content
- **Check if liked**: Show whether the current user has already liked a post
- **Celebrity scale**: Handle posts with 100M+ likes (Ronaldo, Taylor Swift, MrBeast)
- **Recent likers**: "Liked by Alice, Bob, and 2.3M others"
- **Like notifications**: Notify content owner of new likes (batched/grouped for celebrities)

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-High Throughput**: 1M+ likes/sec on a viral celebrity post
- **Low Latency**: Like action completes in < 100 ms
- **Eventual Consistency**: Count can lag by a few seconds; approximate OK for display
- **Availability**: 99.99% — likes are a core engagement feature
- **Idempotent**: Rapid double-tap doesn't create duplicate likes
- **Scalability**: Support 1B+ users, posts with 100M+ likes

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total likes / day | 5B |
| Likes / sec (avg) | ~58K |
| Peak likes / sec (viral post) | 1M+ |
| Avg post likes | 50 |
| Celebrity post likes | 10M - 100M |
| Like record size | 32 bytes (user_id + post_id + timestamp) |
| Storage / day | 5B × 32B = 160 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Client Layer                                │
│  User taps ❤️ → optimistic UI update (instant) → send to server     │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                         ┌──────▼──────┐
                         │ API Gateway │
                         │ Rate limit: │
                         │ 10 likes/sec│
                         │ per user    │
                         └──────┬──────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│                         Like Service                                 │
│  (Stateless, horizontally scaled)                                    │
│                                                                       │
│  POST /like:                                                         │
│  1. Validate: user exists, post exists                               │
│  2. Dedup check (Redis): SISMEMBER liked:{post_id} {user_id}        │
│     → If already liked → return current count (idempotent)           │
│  3. For NORMAL posts (< 10K likes):                                  │
│     → Direct DB write + Redis INCR                                   │
│  4. For HOT posts (celebrity, trending):                             │
│     → Buffer in Redis only → async batch write to DB                 │
│  5. Publish event to Kafka: {user_id, post_id, action: "like"}      │
│  6. Return updated count                                             │
└──────┬────────────────────────┬──────────────────────────────────────┘
       │                        │
  ┌────▼──────────┐      ┌──────▼──────┐
  │    Redis      │      │   Kafka     │
  │   Cluster     │      │ (like-events│
  │               │      │  topic)     │
  │ Data:         │      └──────┬──────┘
  │ ┌───────────┐ │             │
  │ │like_count:│ │      ┌──────▼──────────────────────┐
  │ │{post_id}  │ │      │  Like Event Consumers        │
  │ │= integer  │ │      │                              │
  │ ├───────────┤ │      │  Consumer 1: DB Writer        │
  │ │liked:     │ │      │  → Batch INSERT into          │
  │ │{post_id}  │ │      │    Cassandra every 5 seconds  │
  │ │= SET of   │ │      │                              │
  │ │ user_ids  │ │      │  Consumer 2: Count Syncer     │
  │ │(or Bloom) │ │      │  → Periodic reconcile Redis   │
  │ ├───────────┤ │      │    count with Cassandra count  │
  │ │hot_posts  │ │      │                              │
  │ │= SET of   │ │      │  Consumer 3: Notification     │
  │ │ post_ids  │ │      │  → Batch: "Alice and 1,523    │
  │ │ currently │ │      │    others liked your post"    │
  │ │ viral     │ │      │                              │
  │ └───────────┘ │      │  Consumer 4: Analytics        │
  └───────────────┘      │  → ClickHouse for reporting   │
                         └──────────────────────────────┘
       │
  ┌────▼──────────┐
  │  Cassandra    │  ← Source of truth for individual likes
  │               │
  │  likes_by_post│  ← "Who liked post X?"
  │  likes_by_user│  ← "What did user Y like?"
  │  like_counts  │  ← Materialized count per post
  └───────────────┘
```

### Component Deep Dive

#### The Hot Counter Problem — Why Normal DBs Fail

```
Celebrity posts "Ronaldo scores goal" → 1M likes in 60 seconds

Naive approach:
  UPDATE posts SET like_count = like_count + 1 WHERE post_id = ?;
  
  At 1M/sec → 1M write transactions on a SINGLE ROW
  → Row-level lock contention → DB thread pool exhausted
  → All other queries on the DB slow down → cascading failure
  
  Even Cassandra counters struggle: counter updates require read-before-write
  internally, and at 1M/sec on one partition → hot partition → node overwhelmed
```

#### Solution: Tiered Architecture for Normal vs Hot Posts

```
Tier 1 — Normal Posts (< 10K likes):
  Like → Redis SADD + INCR → synchronous Cassandra write
  Simple, consistent, low volume per post
  
Tier 2 — Hot Posts (10K+ likes, detected via velocity):
  Like → Redis SADD + INCR only → Kafka event
  Cassandra batch write every 5 seconds (1 write per batch, not 1M)
  
Hot post detection:
  Track like velocity per post in Redis:
    INCR like_velocity:{post_id}:{minute}
    If velocity > 1000/min → add to hot_posts SET → switch to Tier 2
  
  When velocity drops below 100/min → remove from hot_posts → back to Tier 1

Why this works:
  Redis handles 500K+ INCR/sec on a single key (in-memory, atomic)
  Cassandra batch: 1 write every 5s vs 5M individual writes → 5M× reduction
  Count accuracy: Redis is always current; Cassandra lags by up to 5 seconds
  Display shows Redis count (instant) → nobody notices 5-second lag
```

#### "Liked by" Deduplication — One Like Per User

```
Challenge: At 1M likes/sec, how to ensure user can't like twice?

Redis SET approach:
  SADD liked:{post_id} {user_id}
  Returns 1 → new like → INCR count
  Returns 0 → already liked → no-op (idempotent)
  
  Problem: Post with 100M likes → 100M user_ids in SET
  Memory: 100M × 16 bytes (UUID) = 1.6 GB for ONE post's dedup set
  
Bloom Filter approach ⭐ (for mega-posts):
  Space: 100M entries × 10 bits = 125 MB (vs 1.6 GB)
  False positive rate: ~1% (1 in 100 new likes incorrectly seen as duplicate)
  → 1% of legitimate likes silently dropped → acceptable
  
  BF.ADD liked:{post_id} {user_id}
  BF.EXISTS liked:{post_id} {user_id}
  
Hybrid ⭐⭐:
  Posts with < 100K likes → Redis SET (exact, 1.6 MB)
  Posts with 100K+ likes → Bloom Filter (approximate, saves 90% memory)
  Cassandra is always the source of truth for exact dedup
```

#### Sharded Counters — The Alternative Approach

```
Instead of one counter, split into N sub-counters:

  like_count:{post_id}:0 = 15234
  like_count:{post_id}:1 = 15189
  like_count:{post_id}:2 = 15301
  ...
  like_count:{post_id}:7 = 15122

  Write: INCR like_count:{post_id}:{random(0..7)}
  Read:  SUM(like_count:{post_id}:0 through :7) = 121,234

  Each shard handles 1/8 of write traffic → 8× more throughput
  
  Read cost: 8 Redis GETs instead of 1 → MGET in 1 round trip
  
  When to use:
    Standard INCR: up to ~500K ops/sec per key (single Redis node)
    Sharded (8 shards): up to ~4M ops/sec per key
    Beyond that: in-memory batching at the service layer
```

---

## 5. APIs

### Like / Unlike
```http
POST /api/v1/likes
{ "post_id": "post-uuid", "action": "like" }
→ 200 OK { "liked": true, "like_count": 1523401 }

POST /api/v1/likes
{ "post_id": "post-uuid", "action": "unlike" }
→ 200 OK { "liked": false, "like_count": 1523400 }
```

### Check If Liked
```http
GET /api/v1/likes/check?post_id=post-uuid
→ 200 OK { "liked": true }
```

### Get Like Count (Batch)
```http
POST /api/v1/likes/counts
{ "post_ids": ["post-1", "post-2", "post-3"] }
→ 200 OK { "counts": {"post-1": 1523401, "post-2": 42, "post-3": 89234} }
```

### Get Recent Likers
```http
GET /api/v1/likes/recent?post_id=post-uuid&limit=5
→ 200 OK { "users": [{"id": "...", "name": "Alice"}, ...], "total_count": 1523401 }
```

---

## 6. Data Model

### Redis — Real-Time (Primary Read Path)
```
# Like count (instant reads)
Key:    like_count:{post_id}
Type:   Integer
Ops:    INCR / DECR / GET

# Dedup set (normal posts)
Key:    liked:{post_id}
Type:   SET
Members: user_ids
TTL:    None (permanent for active posts)

# Bloom filter (mega-posts, 100K+ likes)
Key:    bf:liked:{post_id}
Type:   Bloom Filter (RedisBloom module)
Config: capacity=100M, error_rate=0.01

# Hot post detection
Key:    like_velocity:{post_id}:{minute}
Type:   Integer (INCR, TTL=120s)
```

### Cassandra — Source of Truth
```sql
-- Who liked a specific post? (paginated)
CREATE TABLE likes_by_post (
    post_id     UUID,
    liked_at    TIMESTAMP,
    user_id     UUID,
    PRIMARY KEY (post_id, liked_at, user_id)
) WITH CLUSTERING ORDER BY (liked_at DESC)
  AND default_time_to_live = 0;

-- What posts did a user like? (for profile page)
CREATE TABLE likes_by_user (
    user_id     UUID,
    liked_at    TIMESTAMP,
    post_id     UUID,
    PRIMARY KEY (user_id, liked_at)
) WITH CLUSTERING ORDER BY (liked_at DESC);

-- Materialized like count (reconciled from likes_by_post)
CREATE TABLE like_counts (
    post_id     UUID PRIMARY KEY,
    count       COUNTER
);
```

**Why Cassandra?**
- Write-heavy workload (5B writes/day) → Cassandra excels at writes
- Partition by post_id → each post's likes on one partition
- Time-ordered clustering → "recent likers" is a simple range scan
- Counter type for aggregated counts
- TTL not needed (likes are permanent)

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Redis crash** | Redis Cluster with replicas. Rebuild counts from Cassandra if needed |
| **Double-like** | Redis SET/Bloom dedup + Cassandra PRIMARY KEY dedup (upsert) |
| **Count drift (Redis vs Cassandra)** | Reconciliation job every hour: read Cassandra count → fix Redis if diverged |
| **Hot post overloads Redis** | Sharded counters or in-memory batching at service layer |
| **Kafka consumer lag** | Cassandra writes may lag; Redis is always current for display |
| **Unlike race with like** | Redis SREM + DECR is atomic per command; Cassandra delete by PK |

### Race Conditions

#### 1. Like + Unlike in Quick Succession

```
T=0ms:   User likes post → SADD returns 1 → INCR count → Kafka event "like"
T=50ms:  User unlikes → SREM returns 1 → DECR count → Kafka event "unlike"

If Kafka events arrive out of order at Cassandra writer:
  "unlike" processed first → DELETE (no-op, row doesn't exist)
  "like" processed second → INSERT row
  → User is now shown as "liked" even though they unliked!

Solution: Include timestamp in Kafka event. Cassandra writer:
  On "like": INSERT IF NOT EXISTS (or check timestamp)
  On "unlike": DELETE only if existing row timestamp < unlike timestamp
  → Last-write-wins with timestamp-based resolution
```

#### 2. Counter Reconciliation Race

```
Redis count: 1,523,401
Cassandra count query: SELECT COUNT(*) FROM likes_by_post WHERE post_id = ?
  → Returns 1,523,398 (3 events still in Kafka, not yet written)

If reconciliation blindly sets Redis = Cassandra count:
  Redis goes from 1,523,401 → 1,523,398 → looks like 3 likes disappeared
  Then Kafka consumer writes 3 more → Cassandra becomes 1,523,401
  But Redis is stuck at 1,523,398

Solution: Reconciliation should only fix LARGE discrepancies (> 100)
  Small discrepancies are likely due to Kafka pipeline lag.
  Or: use Cassandra COUNTER (atomic increment, not count query)
```

---

## 8. Additional Considerations

### Count Display Formatting
```
< 1,000:     exact ("842")
1K - 999K:   "15.2K"
1M - 999M:   "1.5M"
1B+:         "1.2B"

At this level of rounding, ±100 count error is completely invisible.
This is why eventual consistency is perfectly acceptable for like counts.
```

### Celebrity Notification Batching
```
Ronaldo gets 1M likes/min. Don't send 1M push notifications.

Batching strategy:
  First 10 likes: individual notifications ("Alice liked your post")
  10-100: batch every 5 min ("42 people liked your post")
  100-1000: batch every 15 min
  1000+: batch every hour ("1.2M people liked your post in the last hour")
  
  + "Top liker" highlight: "Messi and 1.2M others liked your post"
    (Show verified/famous accounts first)
```

---

## 9. Deep Dive: Engineering Trade-offs

### Redis SET vs Bloom Filter vs Cassandra-Only Dedup

| Approach | Memory per 1M likes | Accuracy | Latency | Best For |
|---|---|---|---|---|
| **Redis SET** | 16 MB | 100% exact | < 1 ms | Posts with < 100K likes |
| **Bloom Filter** | 1.2 MB | ~99% (1% false positive) | < 1 ms | Mega-posts (100K+) |
| **Cassandra-only** | 0 (disk) | 100% exact | ~5 ms | When memory is constrained |
| **Hybrid** ⭐ | SET for small + BF for large | 99-100% | < 1 ms | Production systems at scale |

### Why Not PostgreSQL for Likes?

```
PostgreSQL:
  INSERT INTO likes (post_id, user_id) VALUES (?, ?);
  SELECT COUNT(*) FROM likes WHERE post_id = ?;
  
  At 1M likes/sec:
    INSERT → row-level lock on unique index → lock contention → ~10K/sec max
    COUNT(*) on 100M rows → sequential scan → 30+ seconds
  
  ✗ Cannot handle write throughput for hot posts
  ✗ COUNT(*) is O(N) — catastrophically slow for large like counts
  
Cassandra:
  ✓ Writes are O(1) (append-only LSM tree, no read-before-write for non-counter)
  ✓ Partition by post_id → each post's writes go to one node
  ✓ COUNTER type for O(1) count reads
  ✗ No transactions (OK for likes — eventual consistency is fine)
  ✗ Counter updates are slower than Redis INCR

Best: Redis for real-time count + Cassandra for persistence
```

