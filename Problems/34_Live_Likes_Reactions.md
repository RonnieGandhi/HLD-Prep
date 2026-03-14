# 34. Design Live Likes and Reactions System

---

## 1. Functional Requirements (FR)

- **React to content**: Users can like/react (❤️ 😂 😮 😢 😡) to posts, videos, live streams in real-time
- **Real-time count**: Display live reaction count that updates instantly for all viewers
- **Reaction animation**: Show floating reaction emojis on live streams (like Facebook Live)
- **Toggle reaction**: User can change/remove their reaction
- **Aggregate counts**: Show total counts per reaction type (1.2K ❤️, 300 😂)
- **Deduplicate**: One reaction per user per content item

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: Reaction visible to all viewers within 1 second
- **High Throughput**: Handle 1M+ reactions/sec during viral events (Super Bowl, concert)
- **Eventual Consistency**: Exact count can lag by a few seconds; approximate is fine for display
- **Availability**: 99.99% — reactions are a core engagement feature
- **Scalability**: Support 100M concurrent users on a single live event
- **Idempotent**: Duplicate clicks don't double-count

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Peak reactions / sec (viral live event) | 1M |
| Avg reactions / sec | 50K |
| Reaction record size | 50 bytes (user_id + content_id + type) |
| Write throughput | 50 MB/s peak |
| Active live events | 10K |
| Reactions per event (avg) | 100K |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────┐
│                        Client Layer                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Viewer 1 │  │ Viewer 2 │  │ Viewer 3 │  │ Viewer N │        │
│  │ (WS conn)│  │ (WS conn)│  │ (WS conn)│  │ (WS conn)│        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
└───────┼──────────────┼──────────────┼──────────────┼─────────────┘
        │  HTTP POST   │              │              │ WebSocket
        │  (reaction)  │              │              │ (receive updates)
        ▼              ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────┐
│                     API Gateway / L7 LB                          │
│  • Rate limit: 5 reactions/sec per user (sliding window)        │
│  • Auth: JWT validation                                          │
│  • Route: POST → Reaction Service; WS → WebSocket Gateway       │
└───────┬──────────────────────────────────────┬───────────────────┘
        │                                       │
┌───────▼──────────────┐              ┌─────────▼──────────────┐
│   Reaction Service   │              │  WebSocket Gateway      │
│   (Stateless Pool)   │              │  (Stateful — per conn)  │
│                      │              │                         │
│  1. Validate request │              │  • 100K conns/server    │
│  2. Dedup check      │              │  • Subscribe to Redis   │
│     (Redis SISMEMBER)│              │    Pub/Sub channel per  │
│  3. Increment count  │              │    content_id           │
│     (Redis HINCRBY)  │              │  • Push batched deltas  │
│  4. Publish to Kafka │              │    to clients every     │
│  5. Return ACK       │              │    500ms                │
└───────┬──────────────┘              └───────┬─────────────────┘
        │                                      │
  ┌─────┴──────┐                              │
  │            │                              │
┌─▼────┐   ┌──▼──────┐                ┌──────▼──────────────┐
│Redis │   │  Kafka  │                │   Redis Pub/Sub     │
│Cluster│   │         │                │                     │
│      │   │ Topic:  │                │ Channel per content:│
│ Data:│   │reactions│                │ reactions:{cid}     │
│ ─────│   │         │                │                     │
│ Hash:│   │Partition│                │ Aggregator publishes│
│ count│   │  by     │                │ batched deltas every│
│ ─────│   │content_ │                │ 500ms               │
│ Set: │   │  id     │                │                     │
│ dedup│   │         │                └─────────────────────┘
└──┬───┘   └──┬──────┘                         ▲
   │          │                                 │
   │     ┌────▼─────────────────────┐          │
   │     │ Reaction Aggregator      │          │
   │     │ (Flink / Custom Worker)  │──────────┘
   │     │                          │  publishes batched deltas
   │     │ • Window: 500ms tumbling │  to Redis Pub/Sub
   │     │ • Batch by content_id    │
   │     │ • Sample for animations  │
   │     │ • Write to Cassandra     │
   │     │   (batch INSERT every 5s)│
   │     └────┬─────────────────────┘
   │          │
   │     ┌────▼─────────────┐
   │     │   Cassandra      │
   │     │  (Persistent)    │
   │     │                  │
   │     │  user_reactions  │ ← individual reactions (dedup by PK)
   │     │  reaction_counts │ ← aggregate counters
   │     └──────────────────┘
   │
   └──── Redis also serves GET /counts (read path)
```

### Component Deep Dive

#### The Hot Counter Problem

**Naive approach**: `UPDATE reactions SET count = count + 1 WHERE content_id = ?`
- At 1M reactions/sec → 1M write transactions on a single row → DB melts

**Solution: Redis + Batched Writes** ⭐:
```
Layer 1 — Redis (real-time, approximate):
  HINCRBY reaction_count:{content_id} heart 1
  HINCRBY reaction_count:{content_id} laugh 1
  
  Reads: HGETALL reaction_count:{content_id} → {heart: 15234, laugh: 3421}
  
  Redis handles 1M+ INCR/sec easily (single-threaded atomic operations)

Layer 2 — Cassandra (persistent, exact):
  Kafka consumer batches reactions every 5 seconds
  Batch write to Cassandra: INSERT INTO reactions (content_id, user_id, type, timestamp)
  Update aggregate: UPDATE reaction_counts SET count = count + batch_size
  
  5-second eventual consistency is invisible to users
```

#### Deduplication (One Reaction Per User)

```
Fast dedup (Redis):
  SADD reacted:{content_id} {user_id}
  If SADD returns 0 → already reacted → update type (toggle) instead of increment

Persistent dedup (Cassandra):
  PRIMARY KEY (content_id, user_id) → natural dedup on upsert
```

#### Real-Time Broadcast

```
For live streams with millions of viewers:
  Don't broadcast every individual reaction → 1M messages/sec is too much

Instead: Batch + Sample
  Every 500ms, aggregate reactions received in that window:
    {heart: +342, laugh: +89, wow: +23}
  Broadcast ONE message with deltas to all WebSocket clients
  
  Client-side: Smoothly animate counter incrementing by the delta
  Client-side: Show random sample of 5 floating emoji animations (not all 342)
```

---

## 5. APIs

### React to Content
```http
POST /api/v1/reactions
{ "content_id": "post-123", "type": "heart" }
→ 200 OK { "current_counts": {"heart": 15235, "laugh": 3421} }
```

### Remove Reaction
```http
DELETE /api/v1/reactions?content_id=post-123
→ 200 OK
```

### Get Reaction Counts
```http
GET /api/v1/reactions/counts?content_id=post-123
→ { "heart": 15234, "laugh": 3421, "wow": 892, "total": 19547 }
```

### WebSocket (Live Stream Reactions)
```json
// Server pushes every 500ms:
{ "type": "reaction_update", "content_id": "live-456", 
  "deltas": {"heart": 342, "laugh": 89}, "total": 1523400 }
```

---

## 6. Data Model

### Redis — Real-Time Counts
```
Key:    reaction_count:{content_id}
Type:   Hash
Fields: heart=15234, laugh=3421, wow=892

Key:    reacted:{content_id}
Type:   Set
Members: user_ids who reacted (for dedup)
TTL:    24h for live events (cleanup after event ends)
```

### Cassandra — Persistent Reactions
```sql
CREATE TABLE user_reactions (
    content_id  UUID,
    user_id     UUID,
    type        TEXT,       -- heart, laugh, wow, sad, angry
    created_at  TIMESTAMP,
    PRIMARY KEY (content_id, user_id)
);

CREATE TABLE reaction_counts (
    content_id  UUID PRIMARY KEY,
    heart       COUNTER,
    laugh       COUNTER,
    wow         COUNTER,
    sad         COUNTER,
    angry       COUNTER,
    total       COUNTER
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Redis crash** | Redis Cluster with replicas. Rebuild counts from Cassandra if needed |
| **Double-counting** | Redis SET dedup + Cassandra upsert (PRIMARY KEY dedup) |
| **Kafka consumer lag** | Cassandra counts may lag; Redis is always current for display |
| **Hot content** | Shard Redis by content_id across cluster; use local in-memory aggregation |
| **Celebrity post** (1M reactions/sec on one post) | Redis HINCRBY is O(1) atomic; single key handles millions of INCRs/sec |

### Race Conditions and Scale Challenges

#### 1. Toggle Reaction Race — Like Then Unlike in Quick Succession

```
User rapidly clicks: Like → Unlike → Like (within 100ms)

Without careful handling:
  T=0ms:   LIKE received → SADD reacted:{cid} user-1 → returns 1 (new) → HINCRBY heart +1
  T=30ms:  UNLIKE received → SREM reacted:{cid} user-1 → returns 1 → HINCRBY heart -1
  T=60ms:  LIKE received → SADD reacted:{cid} user-1 → returns 1 (new) → HINCRBY heart +1
  
  Final: heart count is +1 (correct), user has reacted (correct) ✓

But what if they arrive OUT OF ORDER at the server?
  T=0ms:   UNLIKE arrives first → SREM user-1 → returns 0 (wasn't in set!) → no decrement
  T=30ms:  LIKE arrives → SADD user-1 → returns 1 → HINCRBY +1
  T=60ms:  LIKE (second) arrives → SADD user-1 → returns 0 (already in) → no increment
  
  Final: heart count is +1, but user only clicked like once after un-liking. 
  This is actually correct! Redis SET handles idempotency naturally.

Real problem: Cassandra and Redis disagree on state
  Redis says: user reacted (SET has user-1)
  Cassandra (delayed by 5 seconds): hasn't processed the toggle yet
  → Temporary inconsistency, but self-heals when Kafka consumer catches up
  → ACCEPTABLE for a reaction system
```

#### 2. Hot Key — Viral Content on Single Redis Node

```
Problem: content_id "super-bowl-2026" → all 1M reactions/sec hit ONE Redis key
  → One Redis node handles all traffic → potential bottleneck

  Redis handles ~500K ops/sec on a single node. At 1M → risk of saturation.

Solutions (layered):
  1. In-memory aggregation at Reaction Service layer:
     Each service instance buffers reactions for 100ms
     Then sends a SINGLE HINCRBY with batch count (+342)
     20 service instances × 1 batch/100ms = 200 Redis ops/sec (trivial!)
     
  2. Sharded counters:
     Split one counter into N sub-counters:
     reaction_count:{cid}:0, reaction_count:{cid}:1, ..., reaction_count:{cid}:7
     Each reaction randomly goes to one sub-counter
     Read: SUM across all 8 sub-counters
     Trade-off: Reads are 8× slower, but writes scale 8×

  3. Local Redis proxy with write-behind:
     Each service host has a local Redis that accepts writes
     Background process syncs to central Redis every 500ms
     Read from central Redis (eventual consistency)
```

#### 3. Dedup Set Memory — Million Users Reacting

```
Problem: SADD reacted:{content_id} stores ALL user_ids who reacted
  For a viral post: 50M unique reactors × 16 bytes per user_id = 800 MB
  For one post! If there are 10 such posts → 8 GB just for dedup sets

Solutions:
  1. Bloom filter instead of SET:
     Space: ~15 bits per element (vs 128 bits for UUID)
     50M entries × 15 bits = 93 MB (vs 800 MB)
     Trade-off: ~1% false positive (a new reaction is incorrectly seen as duplicate)
     → 1% of reactions silently dropped → ACCEPTABLE for a reaction system

  2. TTL-based cleanup:
     SET TTL = 24 hours for live events
     After event ends, dedup set auto-expires
     New reactions to old content → re-check in Cassandra (slower, but rare)

  3. Move to Cassandra dedup only:
     Don't store dedup in Redis for non-live content
     Check Cassandra: SELECT * FROM user_reactions WHERE content_id=? AND user_id=?
     Slower (~5ms) but uses no Redis memory
     Use Redis dedup only for actively live streaming content
```

---

## 8. Additional Considerations

### Count Display Formatting
- < 1000: exact ("842")
- 1K-999K: rounded ("15.2K")
- 1M+: rounded ("1.5M")
- This means slight count inaccuracies are invisible to users → eventual consistency is perfectly fine

---

## 9. Deep Dive: Engineering Trade-offs

### Why Not Write Every Reaction to the Database?

```
1M reactions/sec × DB write = DB dies

Instead: Tiered write path
  Tier 1: Redis (in-memory, instant)     → all reads come from here
  Tier 2: Kafka (durable buffer)         → absorbs write burst  
  Tier 3: Cassandra (persistent store)   → batch writes every 5 seconds

If Redis has the count and Cassandra has the individual reactions,
we get the best of both worlds: speed + durability.
```

### Counter Approaches: Redis INCR vs Cassandra Counter vs Flink Aggregation

| Approach | Latency | Accuracy | Durability | Best For |
|---|---|---|---|---|
| **Redis HINCRBY** ⭐ | < 1 ms | Exact in Redis | Volatile | Real-time display |
| **Cassandra COUNTER** | ~5 ms | Exact | Durable | Persistent aggregate |
| **Flink window aggregation** | 1-5 sec window | Exact within window | Durable output | Complex aggregations (by region, time) |
| **HyperLogLog (approx)** | < 1 ms | ~0.81% error | Volatile | Unique reactor count (not total reactions) |

