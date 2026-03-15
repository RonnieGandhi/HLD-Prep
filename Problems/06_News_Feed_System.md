# 6. Design a News Feed System (Facebook / Instagram)

---

## 1. Functional Requirements (FR)

- User can create a post (text, image, video, link)
- User can follow/unfollow other users (or friend them)
- News feed displays posts from followed users/friends, ranked by relevance
- Feed supports infinite scroll / pagination
- Real-time or near-real-time feed updates when a followed user posts
- Support reactions (like, love, etc.) and comments on posts
- Support re-sharing / reposting

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Feed loads in < 200 ms (p99)
- **High Availability**: 99.99% uptime
- **Scalability**: 500M+ DAU, each with a personalized feed
- **Eventual Consistency**: It's OK if a new post appears in followers' feeds within a few seconds
- **Read-Heavy**: Read:Write ratio ~1000:1 (users read feed far more than they post)
- **Ranking**: Feed should be ranked (not purely chronological) for engagement

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 500M |
| Posts / day | 100M (avg 0.2 posts per user/day) |
| Feed reads / day | 5B (avg 10 feed views per user/day) |
| Avg followers per user | 200 |
| Avg post size (metadata) | 1 KB |
| Feed size per user | 500 posts × 1 KB = 500 KB |
| Fan-out writes/day (push model, avg) | 100M × 200 = 20B writes |
| Fan-out writes/sec | ~230K |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │
│ (App/Web)    │
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │  ← Auth, rate limit, routing
└──────┬───────┘
       │
       ├──────────────────────────────────────────────────────┐
       │ WRITE PATH (create post)               READ PATH    │
       │                                        (get feed)   │
┌──────▼───────┐                          ┌──────────────────▼──┐
│  Post Service│                          │   Feed Service      │
│              │                          │                     │
│ • Validate   │                          │  1. Fetch from Redis│
│ • Store post │                          │     (pre-computed)  │
│ • Upload     │                          │  2. Fetch celebrity │
│   media → S3 │                          │     posts (on-read) │
│ • Publish    │                          │  3. Merge + Rank    │
│   event      │                          │  4. Hydrate post    │
│              │                          │     objects         │
└──────┬───────┘                          └──────┬──────┬──────┘
       │                                         │      │
┌──────▼───────┐                          ┌──────▼──┐   │
│  Post DB     │                          │  Feed   │   │
│  (MySQL      │                          │  Cache  │   │
│   sharded    │                          │ (Redis) │   │
│   by user_id)│                          │ Sorted  │   │
└──────┬───────┘                          │ Sets per│   │
       │                                  │ user    │   │
┌──────▼───────┐                          └─────────┘   │
│    Kafka     │                                        │
│ (new-posts)  │                          ┌─────────────▼──┐
└──────┬───────┘                          │ Ranking Service│
       │                                  │ (ML model)     │
┌──────▼───────────────────┐              │                │
│  Fan-Out Service         │              │ Score = α×recency│
│                          │              │  + β×engagement │
│  For each post:          │              │  + γ×affinity   │
│  1. Query Social Graph   │              │  + δ×content    │
│     for follower list    │              │                │
│  2. If followers < 10K:  │              │ Re-rank top 500│
│     → ZADD to each       │              │ → return top 20│
│       follower's Redis   │              └────────────────┘
│       feed (fan-out on   │
│       write) ⭐           │
│  3. If followers ≥ 10K:  │
│     → Skip (celebrity).  │
│       Feed Service does  │
│       fan-out on read    │
│       at query time      │
└──────┬───────────────────┘
       │
┌──────▼───────┐
│ Social Graph │  ← followers:{user_id} → SET of follower_ids
│ Service      │  ← MySQL (durable) + Redis (fast lookups)
└──────────────┘
```

### The Two Approaches: Fan-Out on Write vs. Fan-Out on Read

#### Fan-Out on Write (Push Model)
- **How**: When a user creates a post, immediately write the post ID to every follower's feed cache
- **Pros**: Feed reads are fast (pre-computed), O(1) read
- **Cons**: Expensive for users with millions of followers (celebrity problem). Wastes writes for inactive users
- **Best for**: Normal users with < 10K followers

#### Fan-Out on Read (Pull Model)
- **How**: When a user requests their feed, fetch latest posts from all followed users at query time
- **Pros**: No wasted writes, handles celebrities naturally
- **Cons**: Slow feed reads (must query many users' post timelines and merge)
- **Best for**: Celebrity accounts

#### Hybrid Approach ⭐ (Recommended)
- **Normal users (< 10K followers)**: Fan-out on write → push post ID to followers' feed caches
- **Celebrities (> 10K followers)**: Fan-out on read → when a user opens their feed, fetch celebrity posts on-the-fly and merge with pre-computed feed
- **Classification**: A background job periodically classifies users as "normal" or "celebrity" based on follower count

### Component Deep Dive

#### Post Service
- Handles post creation (text, media references, privacy settings)
- Stores posts in Posts DB (MySQL sharded by user_id)
- Uploads media to Object Store (S3) via CDN
- Publishes `new-post` event to Kafka

#### Fan-Out Service
- Consumes `new-post` events from Kafka
- For each new post:
  1. Fetch the poster's follower list from Social Graph Service
  2. If poster is "normal" user → push post_id to each follower's feed in Redis
  3. If poster is "celebrity" → skip fan-out (will be fetched on read)
- **Scaling**: Kafka consumer group with many workers; each worker handles fan-out for assigned partitions
- **Rate**: For a post by a user with 200 followers, fan-out is 200 Redis writes — trivial. For 10K followers, still manageable

#### Social Graph Service
- Stores follower/following relationships
- **Storage**: Adjacency list in Redis (for fast lookups) backed by MySQL/Cassandra
  - `following:{user_id}` → SET of user_ids
  - `followers:{user_id}` → SET of user_ids
- For large followers lists (celebrity): paginated retrieval from Cassandra

#### Feed Service
- Handles feed read requests
- **Flow**:
  1. Fetch pre-computed feed from Redis (list of post_ids)
  2. Fetch celebrity posts separately (pull model) and merge
  3. Pass merged list to Ranking Service
  4. Fetch full post objects for top N ranked posts from Post DB/Cache
  5. Return enriched feed to client

#### Ranking Service
- **Input**: List of candidate post_ids with features
- **ML Model Features**: Affinity with poster, post age, post type (photo vs. text), engagement signals (likes, comments), user's past interactions
- **Output**: Ranked list of post_ids
- **Implementation**: Lightweight inference service (TensorFlow Serving / ONNX Runtime)
- **Fallback**: If ranking service is down, return reverse-chronological feed

#### Feed Cache (Redis)
- **Data Structure**: Sorted Set per user → `feed:{user_id}` with score = timestamp
- **Max size**: Keep latest 500 post_ids per user (~500 × 16 bytes = 8 KB per user)
- **Eviction**: When list exceeds 500, remove oldest entries
- **Read**: `ZREVRANGEBYSCORE feed:{user_id} +inf -inf LIMIT 0 20` → latest 20 posts

---

## 5. APIs

### Create Post
```http
POST /api/v1/posts
Authorization: Bearer <token>
{
  "content": "Hello world!",
  "media_ids": ["img-uuid-1"],
  "privacy": "public"
}
Response: 201 Created
{ "post_id": "post-uuid", "created_at": "..." }
```

### Get News Feed
```http
GET /api/v1/feed?page_size=20&cursor=<last_post_timestamp>
Authorization: Bearer <token>

Response: 200 OK
{
  "posts": [
    {
      "post_id": "...",
      "author": {"user_id": "...", "name": "...", "avatar_url": "..."},
      "content": "Hello world!",
      "media": [{"url": "...", "type": "image"}],
      "reactions_count": 42,
      "comments_count": 7,
      "created_at": "2026-03-13T10:00:00Z"
    }
  ],
  "next_cursor": "1710320400000"
}
```

### Follow/Unfollow
```http
POST /api/v1/users/{user_id}/follow
DELETE /api/v1/users/{user_id}/follow
```

---

## 6. Data Model

### MySQL (Sharded by user_id) — Posts

```sql
CREATE TABLE posts (
    post_id     BIGINT PRIMARY KEY,  -- Snowflake ID
    user_id     BIGINT NOT NULL,
    content     TEXT,
    media_urls  JSON,
    privacy     ENUM('public', 'friends', 'private'),
    created_at  TIMESTAMP,
    INDEX idx_user_created (user_id, created_at DESC)
);
```

### Cassandra — User Timeline (author's own posts)

```sql
CREATE TABLE user_timeline (
    user_id     UUID,
    post_id     BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Redis — Pre-computed Feed

```
Key:    feed:{user_id}
Type:   Sorted Set
Members: post_id (as string)
Scores:  timestamp (epoch ms)
Max:     500 entries (ZREMRANGEBYRANK to trim)
```

### Redis / Cassandra — Social Graph

```
Redis:
  Key: following:{user_id}  → SET of followed user_ids
  Key: followers:{user_id}  → SET of follower user_ids

Cassandra (backup/overflow):
  CREATE TABLE followers (
      user_id      UUID,
      follower_id  UUID,
      followed_at  TIMESTAMP,
      PRIMARY KEY (user_id, follower_id)
  );
```

### Kafka Topic: `new-posts`

```json
{
  "post_id": "post-snowflake-id",
  "user_id": "author-uuid",
  "content_preview": "Hello world!",
  "created_at": "2026-03-13T10:00:00Z",
  "is_celebrity": false,
  "follower_count": 200
}
```

---

## 7. Fault Tolerance

### General
| Technique | Application |
|---|---|
| **Kafka durability** | Post events never lost (RF=3) |
| **Redis persistence** | RDB snapshots + AOF; Redis Cluster for HA |
| **DB replication** | MySQL read replicas, Cassandra RF=3 |
| **Cache-aside fallback** | If Redis feed is empty, reconstruct from Cassandra |

### Problem-Specific

1. **Celebrity Fan-Out Storm**
   - A celebrity with 50M followers posts → 50M writes would take minutes
   - **Solution**: Hybrid model — celebrities use fan-out on read, not write. No storm

2. **Feed Cache Miss (New User or Cold Start)**
   - A user who hasn't opened the app in months has an empty feed cache
   - **Solution**: Fall back to fan-out on read: fetch last 500 posts from all followed users' timelines, merge, rank, cache the result

3. **Stale Feed after Unfollow**
   - User unfollows someone but their posts are still in the feed cache
   - **Solution**: Lazy deletion — when feed is fetched, filter out posts from unfollowed users. Periodically rebuild feed cache in background

4. **Post Deletion**
   - Author deletes a post, but it's in millions of feed caches
   - **Solution**: Don't remove from feed caches (too expensive). When feed is served, check post validity. If deleted, skip it. Eventually, old posts age out of the feed cache

5. **Ranking Service Failure**
   - **Solution**: Graceful degradation — serve reverse-chronological feed (no ranking). User still gets a feed, just less personalized

---

## 8. Additional Considerations

### Cursor-Based Pagination
- Don't use `OFFSET` (performance degrades with depth)
- Use cursor = last post's timestamp: `WHERE created_at < cursor ORDER BY created_at DESC LIMIT 20`
- For ranked feeds: cursor = `{score}_{post_id}` to handle ties

### Content Moderation
- Before publishing a post, run through a content moderation pipeline (text: ML classifier, images: NSFW detection)
- Async: post is published immediately, moderation runs in background. If flagged → hide post

### Push Notifications for Feed
- When a close friend posts, send a push notification
- Don't notify for every post — use a "close friends" heuristic based on interaction frequency

### Feed Freshness
- Use a WebSocket or SSE (Server-Sent Events) to push "new posts available" indicator to the client
- Client can then pull the latest feed

---

## 9. Deep Dive: Engineering Trade-offs

### The Fan-Out Decision: The Most Important Architectural Choice

This is the single most asked follow-up question in a news feed design interview.

```
Fan-Out on Write (Push):
  Post Event → for each follower → write post_id to their feed cache
  
  Cost per post = O(followers_count) WRITES
  Cost per feed read = O(1) READ (pre-computed)
  
  ✓ Read is instant (pre-computed feed)
  ✓ Simple read path
  ✗ Write amplification (celebrity with 50M followers → 50M writes)
  ✗ Wasteful for inactive users (compute feeds they never read)
  ✗ Latency for the post to appear in ALL followers' feeds

Fan-Out on Read (Pull):
  Feed Request → fetch latest posts from each followed user → merge & rank
  
  Cost per post = O(1) WRITE (just store the post)
  Cost per feed read = O(following_count) READS + MERGE
  
  ✓ No write amplification
  ✓ No wasted work for inactive users
  ✗ Slow reads (must query N users' timelines and merge)
  ✗ Latency spikes if user follows many accounts

Hybrid (Facebook/Twitter actual approach) ⭐:
  - For 99.9% of users (< 10K followers): Fan-out on Write
  - For 0.1% celebrities (> 10K followers): Fan-out on Read
  
  Why 10K threshold?
    10K × 500M posts/day × 0.2 posts/user = 1B fan-out writes/day (manageable)
    50M × 500M posts/day × 0.001 posts/celebrity = impossible fan-out
    
  On feed read:
    1. Read pre-computed feed from Redis (fan-out-on-write part)
    2. Fetch latest posts from followed celebrities (fan-out-on-read part)  
    3. Merge + rank → return
```

### Why Redis Sorted Set for Feed Cache (Not a List)?

```
Option 1: Redis LIST (LPUSH + LRANGE)
  ✓ Simple append
  ✗ No dedup — if fan-out retries, duplicates appear
  ✗ Cannot efficiently remove a specific post (deletion/moderation)
  ✗ No scoring for ranked feeds

Option 2: Redis SORTED SET (ZADD + ZREVRANGE) ⭐
  ✓ Natural ordering by score (timestamp or ranking score)
  ✓ Dedup built-in (same post_id is a single member)
  ✓ Efficient removal: ZREM post_id
  ✓ Efficient trimming: ZREMRANGEBYRANK to keep top 500
  ✓ Range queries: ZREVRANGEBYSCORE for cursor-based pagination
  ✗ Slightly more memory than LIST (~2× per entry due to skip list structure)

Decision: SORTED SET. The memory overhead is negligible compared to the operational 
benefits (dedup, deletion, ranking). At 500 entries × 8 bytes per member ≈ 4 KB per 
user feed — trivial.
```

### Why MySQL for Posts, Cassandra for Timeline, Redis for Feed?

Each data store is chosen for its access pattern:

| Data | Access Pattern | Best DB | Why |
|---|---|---|---|
| **Posts** (content) | Write once, read by ID, update (edit/delete) | **MySQL (sharded)** | Relational (post → user, post → media), ACID for edits/deletes, familiar query patterns |
| **User Timeline** (author's posts) | Append, read by user_id sorted by time | **Cassandra** | Time-series write pattern, partition by user_id, high write throughput, no joins needed |
| **Home Feed** (pre-computed) | Write many (fan-out), read by user_id | **Redis** | In-memory for instant reads, sorted set for ranking, TTL for auto-cleanup |
| **Social Graph** | Read heavy, set operations (mutual friends) | **Redis + Cassandra** | Redis for fast lookups during fan-out, Cassandra for durability |

**Why not just use one DB for everything?**
- MySQL can't handle 100B fan-out writes/day (write bottleneck)
- Cassandra can't serve ranked feeds in < 200ms (no in-memory sorted access)
- Redis can't store 10B posts durably (volatile, expensive at that scale)
- **Polyglot persistence** — each DB for what it does best

### Ranking Model: Chronological vs ML-Ranked

```
Chronological (Twitter's original approach):
  ✓ Simple, deterministic, transparent to users
  ✓ No cold-start problem
  ✗ Noisy — high-volume posters dominate
  ✗ Important posts from close friends get buried
  ✗ Lower engagement (users miss relevant content)

ML-Ranked (Facebook/Instagram approach) ⭐:
  ✓ Higher engagement (shows content you care about)
  ✓ Handles information overload
  ✗ "Filter bubble" — may limit exposure to diverse content
  ✗ Complex to build and maintain
  ✗ Users sometimes complain about non-chronological order

Ranking Features:
  - Affinity: How often you interact with this user (likes, comments, messages)
  - Post age: Newer posts scored higher (time decay)
  - Content type: Does this user prefer photos over text?
  - Engagement velocity: How fast is this post getting likes/comments?
  - Author engagement: Does the author post high-quality content?
  - User feedback signals: Did user hide similar posts before?

Model Architecture:
  - Candidate generation: ~1000 posts from followed users
  - Scoring: Lightweight model (logistic regression or small neural net) scores each
  - Re-ranking: Apply diversity rules (don't show 5 posts from same user in a row)
  - Returns top ~50 scored posts for initial feed load
```

### The Delete Propagation Problem (Why "Lazy Delete" is Necessary)

```
Scenario: User deletes a post. That post_id exists in 200K followers' feed caches.

Option 1: Eager Delete (remove from all caches immediately)
  → 200K Redis ZREM operations
  → Takes minutes to propagate
  → If Kafka is backed up, caches are inconsistent for hours
  → Cost: O(followers) per delete

Option 2: Lazy Delete (check validity at read time) ⭐
  → Post marked as deleted in Posts DB (instant, O(1))
  → When feed is served, batch-lookup all post_ids → filter out deleted ones
  → Deleted post_id eventually ages out of feed caches (ZREMRANGEBYRANK keeps only 500)
  → Cost: O(1) per delete + small overhead per feed read
  → Trade-off: Deleted post might appear briefly for some users (< 1 second if using cache-aside)

Production approach: Lazy delete + async background cleanup
  → Publish "post-deleted" event to Kafka
  → Background consumer eventually removes from affected feeds (best-effort)
  → Feed serving always double-checks post validity as safety net
```
