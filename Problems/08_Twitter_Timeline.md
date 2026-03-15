# 8. Design a Timeline and Tweet Service (Twitter)

---

## 1. Functional Requirements (FR)

- **Post a tweet**: Text (280 chars), images, videos, links
- **User timeline**: View a user's own tweets (profile page)
- **Home timeline (feed)**: Aggregated tweets from followed users, ranked
- **Follow / Unfollow** users
- **Like, Retweet, Reply, Quote Tweet**
- **Search tweets** by keyword, hashtag, user
- **Trending topics**: Real-time trending hashtags and topics
- **Notifications**: Mentions, likes, retweets, new followers

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Home timeline loads in < 200 ms
- **High Availability**: 99.99%
- **Scalability**: 400M+ DAU, 500M tweets/day
- **Eventual Consistency**: Acceptable for timeline (few seconds delay OK)
- **Read-Heavy**: 100× more reads than writes
- **Real-time**: Trending topics and search index updated within seconds

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 400M |
| Tweets / day | 500M |
| Tweets / sec | ~6,000 (peak ~30K) |
| Home timeline reads / day | 10B (25 views per user/day) |
| Reads / sec | ~115K (peak ~500K) |
| Avg tweet size (metadata) | 1 KB |
| Storage / day (tweets only) | 500M × 1 KB = 500 GB |
| Media storage / day | 50M media tweets × 2 MB = 100 TB |
| Fan-out: avg followers = 200 | 500M × 200 = 100B fan-out writes/day |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │
│ (Web/Mobile) │
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │  ← Auth (JWT), rate limit, routing
└──────┬───────┘
       │
       ├──────────────────────────────────────────────────────┐
       │ WRITE PATH (post tweet)              READ PATH      │
       │                                      (get timeline) │
┌──────▼──────┐                         ┌────────────────────▼──┐
│ Tweet       │                         │  Timeline Service     │
│ Service     │                         │                       │
│             │                         │  1. Fetch pre-computed│
│ • Validate  │                         │     feed from Redis   │
│   (280 char,│                         │     (fan-out-on-write)│
│   content   │                         │  2. Fetch celebrity   │
│   mod)      │                         │     tweets on-the-fly │
│ • Store in  │                         │     (fan-out-on-read) │
│   MySQL     │                         │  3. Merge + Rank via  │
│ • Media →   │                         │     ML scoring        │
│   S3 + CDN  │                         │  4. Hydrate tweet IDs │
│ • Publish   │                         │     with full objects  │
│   event     │                         └──────┬──────┬─────────┘
└──────┬──────┘                                │      │
       │                                 ┌─────▼───┐  │
┌──────▼──────┐                          │  Redis  │  │
│    Kafka    │                          │(Timeline│  │
│(tweet-events│                          │ Cache   │  │
│ topic)      │                          │ ZSET per│  │
└──────┬──────┘                          │ user)   │  │
       │                                 └─────────┘  │
       ├──────────────────────────┐                   │
       │                         │              ┌─────▼──────┐
┌──────▼──────┐           ┌──────▼──────┐       │ ML Ranking │
│ Fan-Out     │           │ Social Graph│       │ Service    │
│ Service     │           │ Service     │       │            │
│             │           │             │       │ Score =    │
│ For each    │◄─────────▶│ Redis:      │       │ recency +  │
│ tweet:      │ get        │ followers:  │       │ engagement+│
│             │ followers  │ {uid} → SET │       │ affinity + │
│ If < 10K:   │           │             │       │ content    │
│  → ZADD to  │           │ MySQL:      │       │ relevance  │
│    each      │           │ durable     │       └────────────┘
│    follower's│           │ follow      │
│    Redis feed│           │ graph       │
│             │           └─────────────┘
│ If ≥ 10K:   │
│  → skip     │
│  (celebrity, │
│   pull on   │
│   read)     │
└──────┬──────┘
       │
       │     ┌──────────────────────────────────────────────────┐
       │     │           ASYNC CONSUMERS (Kafka)                │
       │     │                                                  │
       ├────▶│  ┌──────────────┐  ┌──────────────┐             │
       │     │  │ Search       │  │ Trending     │             │
       │     │  │ Indexer      │  │ Service      │             │
       │     │  │              │  │              │             │
       │     │  │ → Elastic    │  │ Flink:       │             │
       │     │  │   Search     │  │ • Extract    │             │
       │     │  │ (full-text,  │  │   hashtags   │             │
       │     │  │  hashtags,   │  │ • 5-min      │             │
       │     │  │  mentions)   │  │   sliding    │             │
       │     │  │              │  │   window     │             │
       │     │  │              │  │ • Velocity   │             │
       │     │  │              │  │   scoring    │             │
       │     │  │              │  │ • Top-K →    │             │
       │     │  │              │  │   Redis      │             │
       │     │  └──────────────┘  └──────────────┘             │
       │     │                                                  │
       │     │  ┌──────────────┐  ┌──────────────┐             │
       │     │  │ Notification │  │ Analytics    │             │
       │     │  │ Service      │  │ Pipeline     │             │
       │     │  │              │  │              │             │
       │     │  │ • "@mention" │  │ Flink →      │             │
       │     │  │   push notif │  │ ClickHouse   │             │
       │     │  │ • "X liked"  │  │              │             │
       │     │  │ • "X retweeted"│ │ Engagement  │             │
       │     │  │ • APNs/FCM   │  │ metrics,    │             │
       │     │  │              │  │ creator dash │             │
       │     │  └──────────────┘  └──────────────┘             │
       │     └──────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│                       DATA LAYER                                 │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ MySQL /      │  │    Redis     │  │   Cassandra          │   │
│  │ PostgreSQL   │  │              │  │   (User Timelines)   │   │
│  │              │  │ • Home       │  │                      │   │
│  │ • Tweets     │  │   timeline   │  │ PK: user_id          │   │
│  │ • Users      │  │   per user   │  │ CK: tweet_id (desc)  │   │
│  │ • Follows    │  │   (ZSET)     │  │                      │   │
│  │              │  │ • Follower   │  │ For user profile      │   │
│  │ Sharded by   │  │   SETs       │  │ "show me my tweets"  │   │
│  │ user_id      │  │ • Trending   │  │                      │   │
│  │              │  │   (ZSET)     │  │                      │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Hybrid Fan-Out Strategy (Same as News Feed — Critical for Twitter)

Twitter's core architectural challenge:

| User Type | Followers | Strategy | Reason |
|---|---|---|---|
| Normal (99.9%) | < 10K | **Fan-out on Write** | Pre-compute timeline at write time; reads are instant |
| Celebrity (0.1%) | > 10K | **Fan-out on Read** | Writing to 50M timelines is too slow |

**Implementation**:
1. User A tweets → Tweet Service publishes to Kafka
2. Fan-out Service consumes event, checks follower count
3. If < 10K followers → fetch follower list → write tweet_id to each follower's Redis timeline
4. If ≥ 10K followers → skip fan-out; store tweet in author's timeline only
5. When User B reads home timeline:
   - Fetch pre-computed timeline from Redis (fan-out-on-write results)
   - Separately fetch latest tweets from any celebrities User B follows (fan-out-on-read)
   - Merge and rank

### Component Deep Dive

#### Tweet Service
- Validates tweet (280 char limit, content moderation)
- Stores tweet in MySQL/Cassandra
- Uploads media to S3 + CDN
- Publishes to Kafka `tweet-events` topic

#### Timeline Service
- Serves home timeline and user timeline requests
- **Home timeline**: Reads from Redis Sorted Set (pre-computed) + celebrity merge
- **User timeline**: Direct Cassandra query `WHERE user_id = X ORDER BY created_at DESC`
- Enriches tweet IDs with full tweet data (from Tweet Cache or DB)

#### Fan-Out Service
- Kafka consumer group processing `tweet-events`
- For each tweet: queries Social Graph for follower list → writes tweet_id to each follower's Redis timeline
- **Performance**: With Kafka parallelism, can process 100B fan-out writes/day
- **Handles**: New tweet, delete tweet (remove from timelines), retweet

#### Search Service + Elasticsearch
- **Indexing**: Consumes from Kafka → indexes tweets in Elasticsearch in near real-time
- **Schema**: `{tweet_id, user_id, content, hashtags[], mentions[], created_at, engagement_score}`
- **Query**: Full-text search + filters (hashtag, user, date range)
- **Relevance**: BM25 text relevance + recency boost + engagement boost

#### Trending Service + Apache Flink
- **Purpose**: Identify trending hashtags and topics in real-time
- **How Flink processes trends**:
  1. Consume tweet events from Kafka
  2. Extract hashtags and keywords
  3. Sliding window aggregation (count per 5-minute window, sliding by 1 minute)
  4. Calculate velocity: `trend_score = current_window_count / previous_window_count`
  5. Top-K: Maintain a min-heap of top 50 trending topics
  6. Output to Redis sorted set for serving
- **Heavy Hitters / Count-Min Sketch**: For approximate counting of very high-cardinality hashtags with bounded memory

#### Social Graph Service
- Stores follow/following relationships
- **Redis**: For fast lookups during fan-out (`followers:{user_id}` → SET)
- **MySQL**: Durable storage for relationships

#### Notification Service
- Consumes events from Kafka (`tweet-events`, `like-events`, `retweet-events`, `follow-events`)
- **Triggers**:
  - `@mention` in a tweet → push notification to mentioned user
  - Like/retweet → notification to tweet author (batched: "Alice and 15 others liked your tweet")
  - New follower → "X started following you"
  - Reply → notification to parent tweet author
- **Channels**: APNs (iOS), FCM (Android), in-app notification tab, email digest (daily/weekly)
- **Batching**: Collapse multiple similar events to avoid notification spam (e.g., 50 likes → 1 push)

#### ML Ranking Service
- Timeline is NOT purely reverse-chronological — it's ranked by relevance
- **Features**: tweet recency, engagement velocity (likes/retweets per minute), user affinity (how often you interact with author), content type (image/video/text), author verification status
- **Model**: Lightweight gradient boosted tree for online scoring (< 10ms per candidate set)
- **Fallback**: If ranking service is unavailable → return reverse-chronological feed

---

## 5. APIs

### Post Tweet
```http
POST /api/v1/tweets
{
  "content": "Hello Twitter! #systemdesign",
  "media_ids": ["img-uuid"],
  "reply_to": null
}
Response: 201 Created
{
  "tweet_id": "snowflake-id",
  "created_at": "2026-03-13T10:00:00Z"
}
```

### Get Home Timeline
```http
GET /api/v1/timeline/home?cursor={last_tweet_id}&limit=20
Response: 200 OK
{
  "tweets": [...],
  "next_cursor": "1234567890"
}
```

### Get User Timeline
```http
GET /api/v1/users/{user_id}/tweets?cursor={last_tweet_id}&limit=20
```

### Search
```http
GET /api/v1/search?q=%23systemdesign&type=recent&cursor=...
```

### Like / Retweet
```http
POST /api/v1/tweets/{tweet_id}/like
POST /api/v1/tweets/{tweet_id}/retweet
```

### Get Trending
```http
GET /api/v1/trends?location=US
Response: 200 OK
{
  "trends": [
    {"hashtag": "#SystemDesign", "tweet_count": 125000, "rank": 1},
    ...
  ]
}
```

---

## 6. Data Model

### MySQL (Sharded by tweet_id) — Tweets

```sql
CREATE TABLE tweets (
    tweet_id    BIGINT PRIMARY KEY,  -- Snowflake ID
    user_id     BIGINT NOT NULL,
    content     VARCHAR(280),
    media_urls  JSON,
    reply_to    BIGINT,              -- null if not a reply
    retweet_of  BIGINT,              -- null if not a retweet
    like_count  INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    reply_count INT DEFAULT 0,
    created_at  TIMESTAMP,
    is_deleted  BOOLEAN DEFAULT FALSE,
    INDEX idx_user (user_id, created_at DESC)
);
```

### Cassandra — User Timeline

```sql
CREATE TABLE user_timeline (
    user_id     UUID,
    tweet_id    BIGINT,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
```

### Redis — Home Timeline Cache

```
Key:    timeline:home:{user_id}
Type:   Sorted Set
Members: tweet_id
Scores:  timestamp
Max:     800 entries
TTL:     48 hours
```

### Redis — Trending Topics

```
Key:    trending:{country_code}
Type:   Sorted Set
Members: hashtag
Scores:  trend_score
```

### Elasticsearch — Tweet Search Index

```json
{
  "tweet_id": "snowflake-id",
  "user_id": "user-uuid",
  "username": "johndoe",
  "content": "Hello Twitter! #systemdesign",
  "hashtags": ["systemdesign"],
  "mentions": [],
  "created_at": "2026-03-13T10:00:00Z",
  "engagement_score": 150,
  "language": "en"
}
```

### MySQL — Social Graph

```sql
CREATE TABLE follows (
    follower_id  BIGINT,
    followee_id  BIGINT,
    created_at   TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id, follower_id)
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Tweet never lost** | Written to MySQL/Cassandra with replication before ack |
| **Fan-out lag** | Kafka buffers events; fan-out workers catch up after recovery |
| **Redis timeline loss** | Reconstructable from DB (fan-out service can rebuild) |
| **Celebrity fan-out storm** | Hybrid model avoids this entirely |
| **Search index lag** | Consumers process Kafka with at-least-once semantics; Elasticsearch is eventually consistent |
| **Trending accuracy** | Flink checkpointing ensures exactly-once processing; Count-Min Sketch allows bounded error |
| **Delete propagation** | Tweet deletion published to Kafka → fan-out service removes from timelines + search index marked as deleted |
| **Hot tweet (viral)** | Cache full tweet object in distributed Redis; use read replicas |

### Specific: Handling Tweet Deletion
1. Mark tweet as `is_deleted = true` in DB (soft delete)
2. Publish `tweet-deleted` event to Kafka
3. Fan-out service removes tweet_id from all followers' Redis timelines (async, eventual)
4. Search service removes from Elasticsearch index
5. Client-side: on receiving deleted tweet_id, removes from UI

---

## 8. Additional Considerations

### Trending Algorithm Deep Dive
- **Not just "most tweets"** — that would always show popular permanent hashtags
- **Velocity-based**: `score = (count_in_current_window - count_in_previous_window) / count_in_previous_window`
- Topics with sudden spikes rank higher than consistently popular ones
- Filter out spam/bot-generated trends
- Geo-specific trends (per country/city)

### Count-Min Sketch for Trending
- Memory-efficient probabilistic data structure
- Uses d hash functions mapping to w counters (d × w matrix)
- On increment: hash hashtag with each of d functions, increment corresponding counters
- On query: take minimum of d counter values (overcounts, never undercounts)
- Space: ~1 MB can track millions of unique hashtags with < 1% error

### Real-Time Feed Updates
- Use Server-Sent Events (SSE) for "new tweets" indicator
- Don't push every tweet in real-time (bandwidth waste)
- Instead: send "12 new tweets" notification, user pulls to refresh

### Tweet Thread / Conversation View
- Recursive data model: `reply_to` forms a tree
- To render a thread: fetch root tweet, then all replies recursively
- Optimize with a `conversation_id` field → fetch all tweets in a conversation in one query

### Content Moderation
- Pre-publish: ML classifier checks for hate speech, spam, misinformation
- Post-publish: User reports → moderation queue → human review
- Automated actions: Shadow ban (reduce visibility), warning labels, account suspension

---

## 9. Deep Dive: Engineering Trade-offs

### Home Timeline Assembly — The Merge Walkthrough

```
User B opens home feed. B follows 300 normal users + 5 celebrities.

Step 1: Fetch pre-computed timeline from Redis (1 ms)
  ZREVRANGEBYSCORE timeline:home:{B} +inf -inf LIMIT 0 200
  → Returns ~200 tweet_ids from fan-out-on-write results
  These are tweets from B's 300 normal followees, already placed here
  at write time by the Fan-out Service.

Step 2: Fetch celebrity tweets (5 parallel Redis calls, 2 ms)
  For each celebrity B follows:
    ZREVRANGEBYSCORE timeline:user:{celeb_id} {now} {now - 24h} LIMIT 0 20
  → Returns ~100 recent tweet_ids from 5 celebrities
  These were NOT pre-computed into B's timeline (fan-out-on-read)

Step 3: Merge and Rank (5 ms in-process)
  Combine 200 + 100 = 300 candidate tweet_ids
  
  For each candidate, compute ranking score:
    score = α × recency + β × engagement + γ × affinity + δ × content_type
    
    recency:      decay function → tweets from 5 min ago > 5 hours ago
    engagement:   like_count + 2 × retweet_count + 3 × reply_count
    affinity:     how often does B interact with this author?
                  (likes, replies, profile views in last 30 days)
    content_type: photos > text > links (engagement data shows this)
  
  Sort by score → top 20 for the first page
  
  Reverse-chronological as fallback:
    If user has algorithmic timeline disabled → sort by timestamp only
    (Twitter's "Latest Tweets" toggle)

Step 4: Hydrate tweet objects (3 ms)
  For each of top 20 tweet_ids:
    Fetch full tweet from Tweet Cache (Redis hash):
      HGETALL tweet:{tweet_id}
    If cache miss → fetch from Cassandra, populate cache
  
  Attach: author profile (from User Cache), engagement counts,
          whether B has liked/retweeted it, conversation thread context

Step 5: Return to client (total: ~12 ms)
  {
    "tweets": [ fully hydrated tweet objects ],
    "next_cursor": "tweet_id_of_200th_item"
  }

Why the merge doesn't bottleneck:
  Pre-computed (step 1): O(1) Redis read for the common case
  Celebrity fetch (step 2): only 5 calls, not 5 million
  Ranking (step 3): 300 candidates × simple arithmetic = microseconds
  The genius: 99.9% of the work (fan-out to 300 followees) was done at WRITE time
```

### The Celebrity Threshold — 10K Followers, Why?

```
Fan-out-on-write cost per tweet:
  1 tweet × N followers = N Redis ZADD operations
  
  N = 100 followers:   100 ZADDs → < 1 ms → trivially fast
  N = 10,000:          10K ZADDs → ~10 ms → acceptable
  N = 100,000:         100K ZADDs → ~100 ms → starting to hurt
  N = 50,000,000:      50M ZADDs → ~50 seconds → IMPOSSIBLE per tweet
  
  Celebrity tweets: 10-50 per day × 50M ZADDs = 500M-2.5B writes/day from ONE user
  That's more than the entire non-celebrity fan-out combined.

Threshold choice:
  At 10K: fan-out cost is ~10 ms → acceptable with batching
  Below 10K: 99.9% of users, and their fan-out completes in < 10 ms
  Above 10K: ~0.1% of users, but they generate 80%+ of fan-out writes
  
  Dynamic threshold: can be tuned per system load
    During low traffic (2 AM): raise to 50K (pre-compute more)
    During peak (Super Bowl): lower to 5K (reduce fan-out load)

Fallback if fan-out is slow:
  Fan-out workers lag behind (spike in tweets)
  User opens feed → pre-computed timeline is stale (missing recent tweets)
  Solution: Timeline Service detects staleness (last_updated > 30s ago)
    → falls back to fan-out-on-read for ALL followees (not just celebrities)
    → slower but always fresh
    → cached result used for subsequent reads until fan-out catches up
```

### Delete Propagation — The Consistency Challenge

```
User tweets → fan-out to 5000 followers → user deletes tweet

Deletion must undo the fan-out:
  1. Mark tweet as deleted in DB (soft delete, instant)
  2. Publish tweet-deleted event to Kafka
  3. Fan-out Service: for each of 5000 followers:
       ZREM timeline:home:{follower} {tweet_id}
     This takes ~5 ms (same speed as fan-out-on-write)
  4. Remove from Elasticsearch search index
  
Problem: between steps 1 and 3, some followers may still SEE the tweet
  They fetch their timeline → tweet_id is there → hydrate from cache → 
  cache still has the tweet? Maybe.
  
  Solution: Tweet Cache check
    During hydration (Step 4 of timeline assembly):
    Fetch tweet from cache → check is_deleted flag → if deleted, skip
    Even if timeline still has the tweet_id, it's filtered out at read time
    
    Belt and suspenders: deletion in timeline (eventual) + filtering at read (immediate)

Celebrity tweet deletion:
  Celebrity deletes tweet → no fan-out to undo (was never written to followers)
  Just mark as deleted in DB → when followers do fan-out-on-read,
  the celebrity's timeline no longer includes it → naturally disappears
```

