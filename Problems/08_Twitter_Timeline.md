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
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │
└──────┬───────┘
       │
       ├────────────────────┬───────────────────┬─────────────────┐
       │                    │                   │                 │
┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐  ┌──────▼──────┐
│ Tweet       │     │ Timeline    │     │  Search     │  │  Trending   │
│ Service     │     │ Service     │     │  Service    │  │  Service    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘  └──────┬──────┘
       │                    │                   │                 │
┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐  ┌──────▼──────┐
│   Kafka     │     │  Redis      │     │Elasticsearch│  │ Apache      │
│(tweet-events│     │  (Timeline  │     │  (Search    │  │ Flink       │
│  topic)     │     │   Cache)    │     │   Index)    │  │ (Streaming) │
└──────┬──────┘     └─────────────┘     └─────────────┘  └──────┬──────┘
       │                                                        │
       ├──────────────────────────────────────┐          ┌──────▼──────┐
       │                                      │          │   Redis     │
┌──────▼──────┐                        ┌──────▼──────┐   │ (Trending  │
│  Fan-out    │                        │ Social Graph│   │  Cache)    │
│  Service    │                        │  Service    │   └─────────────┘
└──────┬──────┘                        └─────────────┘
       │
┌──────▼──────┐
│  MySQL /    │
│  Cassandra  │
│ (Tweet Store│
│  + Timeline)│
└─────────────┘
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

