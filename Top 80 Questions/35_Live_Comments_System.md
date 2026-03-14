# 35. Design a Live Comments System (like Facebook Live / YouTube Live)

---

## 1. Functional Requirements (FR)

- **Post comments**: Users post real-time comments on live videos/streams
- **Real-time delivery**: Comments appear for all viewers within 1 second
- **Pinned comments**: Host/moderator can pin a comment to top
- **Reply threads**: Comments can have short reply chains
- **Moderation**: Filter spam, profanity, block users
- **Comment rate throttling**: During viral events, show sampled comments (can't display 50K comments/sec)
- **Historical comments**: After stream ends, comments persist and are scrollable

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Comment visible to all viewers within 1 second
- **High Throughput**: Handle 100K+ comments/sec on viral streams
- **Scalability**: Support 10M concurrent viewers on a single stream
- **Availability**: 99.99%
- **Ordering**: Comments displayed in roughly chronological order
- **Content filtering**: Profanity/spam filtered in < 50 ms before broadcast

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Concurrent live streams | 50K |
| Peak viewers on one stream | 10M |
| Comments / sec (one hot stream) | 100K |
| Comments / sec (global) | 500K |
| Avg comment size | 200 bytes |
| Throughput | 100 MB/s |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│  Viewers (10M concurrent on one stream)                              │
│  ┌──────┐ ┌──────┐ ┌──────┐              ┌──────┐                  │
│  │ V1   │ │ V2   │ │ V3   │    ...       │ V_10M│                  │
│  └──┬───┘ └──┬───┘ └──┬───┘              └──┬───┘                  │
└─────┼────────┼────────┼──────────────────────┼──────────────────────┘
      │        │        │                      │
      │ WS     │ WS     │ WS                  │ WS
      ▼        ▼        ▼                      ▼
┌──────────────────────────────────────────────────────────────────────┐
│              WebSocket Gateway Cluster                               │
│                                                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ WS Server 1│  │ WS Server 2│  │ WS Server 3│  │WS Server 100 │  │
│  │ 100K conns │  │ 100K conns │  │ 100K conns │  │ 100K conns   │  │
│  │            │  │            │  │            │  │              │  │
│  │ Subscribes │  │ Subscribes │  │ Subscribes │  │ Subscribes   │  │
│  │ to Redis   │  │ to Redis   │  │ to Redis   │  │ to Redis     │  │
│  │ Pub/Sub:   │  │ Pub/Sub:   │  │ Pub/Sub:   │  │ Pub/Sub:     │  │
│  │ stream:    │  │ stream:    │  │ stream:    │  │ stream:      │  │
│  │ {stream_id}│  │ {stream_id}│  │ {stream_id}│  │ {stream_id}  │  │
│  │            │  │            │  │            │  │              │  │
│  │ Local fan- │  │ Local fan- │  │ Local fan- │  │ Local fan-   │  │
│  │ out to its │  │ out to its │  │ out to its │  │ out to its   │  │
│  │ 100K conns │  │ 100K conns │  │ 100K conns │  │ 100K conns   │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────────┘  │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │ ▲
                          Comment  │ │ Sampled comments
                          submit   │ │ via Redis Pub/Sub
                                   ▼ │
┌──────────────────────────────────────────────────────────────────────┐
│                    Comment Processing Pipeline                        │
│                                                                       │
│  ┌───────────────┐     ┌────────────────┐     ┌─────────────────┐   │
│  │ Comment       │     │  Moderation    │     │ Comment         │   │
│  │ Service       │────▶│  Service       │────▶│ Sampler &       │   │
│  │               │     │                │     │ Broadcaster     │   │
│  │ • Assign ID   │     │ • Profanity    │     │                 │   │
│  │ • Validate    │     │   dictionary   │     │ • Aggregate     │   │
│  │   length/rate │     │ • ML spam      │     │   500ms window  │   │
│  │ • Check ban   │     │   classifier   │     │ • Sample top 50 │   │
│  │   list (Redis)│     │ • User shadow  │     │   comments/sec  │   │
│  │               │     │   ban check    │     │ • Publish to    │   │
│  │               │     │ • < 50ms p99   │     │   Redis Pub/Sub │   │
│  └───────┬───────┘     └────────────────┘     └────────┬────────┘   │
│          │                                              │           │
│    ┌─────▼─────┐                                       │           │
│    │   Kafka   │                                       │           │
│    │ (comments)│                                       │           │
│    └─────┬─────┘                                       │           │
│          │                                              │           │
│    ┌─────▼──────────┐                                  │           │
│    │  Cassandra     │  ← ALL comments persisted        │           │
│    │  (Persistent   │    (not just sampled ones)       │           │
│    │   Store)       │                                  │           │
│    └────────────────┘                                  │           │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive
│Redis│  │ Kafka │
│    │  │       │
└─┬──┘  └───┬───┘
  │         │
  │    ┌────▼────┐
  │    │Cassandra│ ← Persistent comment store
  │    └─────────┘
  │
┌─▼──────────┐
│  Fan-Out   │ ← Broadcast to all viewers of a stream
│  Service   │
└────────────┘
```

### Component Deep Dive

#### Comment Flow

```
1. User types comment → sends via WebSocket
2. Comment Service receives → assigns Snowflake ID
3. Moderation Service checks (async, < 50ms):
   - Profanity filter (dictionary + ML classifier)
   - Spam detection (repeated text, links, rate)
   - User ban check (Redis SET of banned users per stream)
4. If approved → publish to Redis Pub/Sub channel: stream:{stream_id}:comments
5. Fan-Out Service delivers to all WebSocket connections subscribed to this stream
6. Async: Write to Kafka → Cassandra for persistence
```

#### Scaling to 10M Viewers — The Fan-Out Challenge

```
Problem: 10M viewers × 100 comments/sec = 1B messages/sec (impossible per-message fan-out)

Solution: Hierarchical Fan-Out

Level 1: Comment Service → Redis Pub/Sub channel per stream
Level 2: Multiple WebSocket servers subscribe to that channel
Level 3: Each WS server fans out to its local clients (e.g., 100K connections each)

10M viewers / 100K per server = 100 WebSocket servers
Each server receives ~100 comments/sec from Redis Pub/Sub → fans out to 100K local connections
Total: 100 × 100 = 10K messages between Redis and WS servers (manageable)
```

#### Comment Sampling for Viral Streams

```
At 100K comments/sec, displaying ALL comments is:
  1. Visually overwhelming (unreadable)
  2. Technically expensive (bandwidth to 10M clients)

Solution: Server-side sampling
  - Take a uniform random sample of ~50 comments/sec
  - Always include: host comments, pinned, verified users, comments with many likes
  - Fan out only the sampled set
  
  Client receives ~50 comments/sec → smooth scroll, readable
  Stored comments: ALL 100K/sec persisted to Cassandra for post-stream replay
```

---

## 5. APIs

### Post Comment (WebSocket)
```json
{"type": "comment", "stream_id": "live-123", "text": "Amazing performance! 🔥"}
// Server ACK: {"type": "comment_ack", "comment_id": "cmt-uuid", "status": "published"}
```

### Receive Comments (WebSocket — server push)
```json
{"type": "new_comments", "stream_id": "live-123", "comments": [
  {"id": "cmt-1", "user": {"name": "Alice", "avatar": "..."}, "text": "Wow!", "ts": 1710320000},
  {"id": "cmt-2", "user": {"name": "Bob"}, "text": "Let's go!", "ts": 1710320001}
]}
```

### Get Historical Comments (REST — after stream ends)
```http
GET /api/v1/streams/{stream_id}/comments?before={cursor}&limit=50
```

---

## 6. Data Model

### Cassandra — Comment Storage
```sql
CREATE TABLE comments (
    stream_id   UUID,
    comment_id  BIGINT,    -- Snowflake ID (time-ordered)
    user_id     UUID,
    text        TEXT,
    is_pinned   BOOLEAN,
    is_deleted  BOOLEAN,
    created_at  TIMESTAMP,
    PRIMARY KEY (stream_id, comment_id)
) WITH CLUSTERING ORDER BY (comment_id DESC)
  AND default_time_to_live = 7776000;  -- 90 days
```

### Redis
```
# Pub/Sub channel for live comment broadcast
Channel: stream:{stream_id}:comments

# Banned users per stream (moderation)
Key:    banned:{stream_id}
Type:   SET
Members: user_ids

# Recent comments buffer (for late joiners)
Key:    recent_comments:{stream_id}
Type:   LIST (capped at 100 items)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **WS server crash** | Client reconnects to another server; fetches missed comments from recent buffer |
| **Moderation service down** | Fallback: publish comment with warning flag; async moderation later |
| **Comment flood** | Rate limit: 1 comment/sec per user; sampling for display |
| **Redis Pub/Sub missed messages** | Acceptable for live — missed comment is transient. Historical replay from Cassandra |
| **Offensive content slip-through** | Report button → manual review queue; retroactive deletion |

### Race Conditions and Scale Challenges

#### 1. Late Joiner Problem — Missing Historical Context

```
User joins a live stream at T=30 minutes.
They see no comments because they missed everything before.

Solution: Recent comments buffer
  Redis LIST: recent_comments:{stream_id} (capped at 100 items)
  Every comment also LPUSHed to this list (LTRIM to keep 100 max)
  
  When user connects via WebSocket:
    1. Fetch last 100 comments from Redis LIST → send immediately
    2. Subscribe to Redis Pub/Sub for new comments
    3. Client merges: no gap, no duplicates (dedup by comment_id)
    
  Race: What if a comment is published between step 1 and step 2?
    → Client may receive it in BOTH the buffer and the subscription
    → Client-side dedup by comment_id handles this
```

#### 2. Moderation Pipeline Race — Published Before Moderated

```
Approach 1: Synchronous moderation (block until reviewed)
  Comment → Moderation → if approved → publish
  ✓ No bad content ever displayed
  ✗ Adds 50-100ms latency to every comment
  ✗ If moderation service is slow/down → ALL comments blocked

Approach 2: Publish first, moderate async ⭐
  Comment → publish immediately → moderate in background
  If flagged → remove from display within 2 seconds
  ✓ Zero added latency for legitimate comments
  ✗ Bad content visible for up to 2 seconds
  
  Mitigation: 
    • Pre-filter with a fast dictionary check (< 1ms) before publish
    • ML classifier runs async, catches sophisticated spam
    • Shadow ban: User sees their own comment, nobody else does
      (reduces complaints, user doesn't know they're banned)

Approach 3: Hybrid — fast filter + async deep analysis
  Comment → dictionary filter (1ms) → if clean → publish + async ML
  If dictionary catches it → block immediately
  If ML catches it (async) → remove retroactively
  
  This is what Facebook Live and YouTube Live actually use.
```

#### 3. Comment Ordering Under Load

```
At 100K comments/sec, comments arrive at different servers at different times.
  Comment A sent at T=1.000 from California
  Comment B sent at T=1.001 from Tokyo
  Comment B arrives at server before Comment A (lower network latency)
  
  Result: Comment B displayed before Comment A → out of order

  Why this is acceptable for live comments:
    - At 50 comments/sec being displayed (sampled), nobody notices ±100ms ordering
    - Comments are ephemeral — they scroll off screen in 3-5 seconds
    - This is NOT a chat (where A→B ordering matters for conversation)
  
  If ordering matters (e.g., host Q&A):
    Use Snowflake IDs (server timestamp) as sort key
    Client sorts by Snowflake ID before displaying
    Maximum skew: ~10ms (NTP sync between servers)
```

#### 4. WebSocket Server Failure — 100K Users Drop

```
WS Server 3 crashes → 100K users lose their WebSocket connection
  
  Recovery:
    1. Client detects connection drop (WebSocket onclose event)
    2. Client reconnects with exponential backoff (100ms, 200ms, 400ms...)
    3. Client sends last_seen_comment_id to new WS server
    4. New WS server:
       a) Fetch comments from Redis LIST where id > last_seen_comment_id
       b) Send missed comments to client
       c) Subscribe client to Redis Pub/Sub
    5. Seamless recovery — user sees no gap in comments
    
  Thundering herd risk: 100K clients all reconnect simultaneously
    Mitigation: Jittered backoff (random 0-500ms before reconnect)
    Load balancer distributes across surviving WS servers
```

---

## 8. Deep Dive: Engineering Trade-offs

### Redis Pub/Sub vs Kafka for Live Comment Broadcast

```
Redis Pub/Sub:
  ✓ Ultra-low latency (< 1 ms)
  ✓ Simple publish/subscribe model
  ✗ Fire-and-forget (no persistence, no replay)
  ✗ If subscriber is slow/offline → messages lost
  Best for: Real-time ephemeral broadcast (live comments, reactions)

Kafka:
  ✓ Durable, replayable
  ✓ Consumer groups for parallel processing
  ✗ Higher latency (5-10 ms)
  ✗ Overhead for ephemeral data
  Best for: Persistent processing (analytics, moderation pipeline, storage)

Use BOTH:
  Redis Pub/Sub for real-time broadcast to viewers (speed)
  Kafka for durable pipeline to Cassandra + moderation (reliability)
```

