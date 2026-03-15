# 10. Design a Threaded Discussion Forum (Reddit)

---

## 1. Functional Requirements (FR)

- **Subreddits**: Create and join topic-based communities
- **Posts**: Submit text posts, links, images, videos to a subreddit
- **Comments**: Threaded/nested comments on posts (tree structure)
- **Voting**: Upvote/downvote on posts and comments
- **Ranking**: Posts ranked by Hot, New, Top, Rising, Controversial
- **User Karma**: Aggregated score from upvotes/downvotes received
- **Home feed**: Aggregated posts from subscribed subreddits
- **Search**: Search posts, comments, subreddits
- **Moderation**: Subreddit moderators can remove posts/comments, ban users
- **Awards / Gilding**: Give awards to posts/comments (premium feature)

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99%
- **Low Latency**: Front page loads in < 200 ms; comment thread loads in < 500 ms
- **Scalability**: 50M+ DAU, millions of posts/day
- **Read-Heavy**: 100:1 read to write ratio
- **Eventual Consistency**: Votes and karma can be slightly stale
- **Deep Comment Threads**: Support threads with 10,000+ comments, nested 20+ levels deep

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 50M |
| Posts / day | 5M |
| Comments / day | 50M |
| Votes / day | 500M |
| Votes / sec | ~6,000 (peak 30K) |
| Avg post size | 2 KB |
| Avg comment size | 500 bytes |
| Storage / day | (5M × 2KB) + (50M × 500B) = 35 GB |
| Hot post cache | 100K posts × 10 KB = 1 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │
│ (Web/Mobile) │
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │  ← Auth, rate limiting, bot detection
└──────┬───────┘
       │
       ├───────────┬───────────┬───────────┬──────────┬──────────┐
       │           │           │           │          │          │
┌──────▼──┐ ┌─────▼────┐ ┌────▼────┐ ┌────▼───┐ ┌───▼────┐ ┌──▼───────┐
│ Post    │ │ Comment  │ │  Vote   │ │ Feed   │ │Search  │ │Subreddit │
│ Service │ │ Service  │ │ Service │ │ Service│ │Service │ │ Service  │
│         │ │          │ │         │ │        │ │        │ │          │
│ • CRUD  │ │ • Nested │ │ • Dedup │ │ • Home │ │ • Full │ │ • Create │
│ • Media │ │   tree   │ │   check │ │   feed │ │   text │ │ • Join / │
│   upload│ │   (mat.  │ │   Redis │ │   merge│ │   posts│ │   leave  │
│ → S3    │ │   path)  │ │ • Kafka │ │   from │ │ • Sub  │ │ • Rules  │
│ • Flair │ │ • Sort:  │ │   buffer│ │   subs │ │   name │ │ • Mods   │
│ • Publish│ │  best,   │ │ • Async │ │ • Rank │ │   search│ │ • Ban   │
│   event │ │  top,    │ │   DB    │ │   by   │ │        │ │   users  │
│         │ │  new,    │ │   write │ │   hot / │ │        │ │          │
│         │ │  controv.│ │         │ │   top   │ │        │ │          │
└────┬────┘ └────┬─────┘ └────┬────┘ └────┬───┘ └───┬────┘ └──────────┘
     │           │            │            │         │
     │           │            │            │    ┌────▼────┐
     │           │            │            │    │Elastic  │
     │           │            │       ┌────▼──┐ │Search   │
     │           │            │       │Redis  │ └─────────┘
     │           │            │       │(Ranked│
     │           │        ┌───▼────┐  │ Feeds │
     │           │        │ Redis  │  │per sub│
     │           │        │(Vote   │  │+ home)│
     │           │        │ Counts │  └───────┘
     │           │        │+ User  │
     │           │        │ Vote   │
     │           │        │ State) │
     │           │        └────────┘
     │           │            │
┌────▼───────────▼────────────▼────────────────────────────┐
│                        Kafka                              │
│  (post-events, comment-events, vote-events)              │
└────┬───────────────────────┬─────────────────────────────┘
     │                       │
┌────▼────┐           ┌──────▼──────┐
│ Ranking │           │  Moderation │
│ Worker  │           │  Pipeline   │
│         │           │             │
│ Consume │           │ • Spam ML   │
│ vote +  │           │ • Toxicity  │
│ post    │           │ • Report    │
│ events  │           │   queue     │
│ → re-   │           │ • Auto-     │
│ compute │           │   remove    │
│ hot/top │           └─────────────┘
│ scores  │
│ → update│
│ Redis   │
│ sorted  │
│ sets    │
└────┬────┘
     │
┌────▼────────────────────────────────────────────────────┐
│                     DATA LAYER                           │
│                                                           │
│ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│ │ PostgreSQL   │  │    Redis     │  │ Elasticsearch│    │
│ │              │  │              │  │              │    │
│ │ • posts      │  │ • vote counts│  │ • Post full  │    │
│ │ • comments   │  │   per post/  │  │   text index │    │
│ │   (mat. path)│  │   comment    │  │ • Subreddit  │    │
│ │ • votes      │  │ • user vote  │  │   name index │    │
│ │ • subreddits │  │   state per  │  │              │    │
│ │ • users      │  │   post       │  │              │    │
│ │ • memberships│  │ • ranked     │  │              │    │
│ │              │  │   feeds (ZSET│  │              │    │
│ │ Source of    │  │   per sub    │  │              │    │
│ │ truth        │  │   by hot/top/│  │              │    │
│ │              │  │   new)       │  │              │    │
│ └──────────────┘  └──────────────┘  └──────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Post Service
- **Create post**: Validate content (size limits, flair required for some subs), upload media to S3 if present, write to PostgreSQL, publish `post-created` event to Kafka
- **Post types**: Text, link, image, video, poll — each stored with type-specific metadata
- **Flair**: Per-subreddit flair system (tags like "Discussion", "Meme", "Question") — enforced by Post Service
- **Soft delete**: Deleted posts keep the entry (for comment tree integrity) but content is replaced with "[deleted]"

#### Subreddit Service
- **Manages**: Subreddit creation, membership (join/leave), rules, moderator assignments, banned users
- **Membership**: `user_subreddits` table (user_id, subreddit_id) — used by Feed Service to build home feed
- **Access control**: Public, restricted (anyone can view, only approved can post), private (invite-only)
- **Moderator actions**: Remove posts/comments, ban users, pin posts, configure automod rules

#### Comment Tree Service — The Key Challenge
Reddit's nested comment tree is the most complex data model:

**Approach 1: Adjacency List** (simple but slow for deep nesting)
```sql
comments (comment_id, post_id, parent_comment_id, ...)
-- To get full tree: recursive query or multiple DB round trips
```

**Approach 2: Materialized Path** ⭐ (recommended)
```sql
comments (comment_id, post_id, path, ...)
-- path = "001/003/007/015" (ancestors from root)
-- To get all descendants: WHERE path LIKE '001/003/%'
-- Sorting by path gives pre-order traversal (natural comment display order)
```

**Approach 3: Nested Set Model** (fast reads, slow writes)
```sql
comments (comment_id, post_id, lft, rgt, ...)
-- All descendants: WHERE lft > parent.lft AND rgt < parent.rgt
-- But inserting a comment requires updating lft/rgt of many rows
```

**Approach 4: Closure Table** (flexible but storage-heavy)
```sql
comment_tree (ancestor_id, descendant_id, depth)
-- One row per ancestor-descendant pair
-- Fast queries but O(depth) inserts
```

**Recommendation**: **Materialized Path** for Reddit-like system. Efficient reads, reasonable writes, and supports sorting.

#### Vote Service
- **Challenge**: 500M votes/day = high write throughput on potentially hot rows
- **Architecture**:
  1. Vote request → check Redis if user already voted on this post/comment
  2. If new vote or changed vote → publish to Kafka `vote-events`
  3. Kafka consumer updates PostgreSQL (source of truth) and Redis (cached counts)
  4. Redis stores: `vote_count:{entity_type}:{entity_id}` as atomic counter
  5. Redis stores: `user_votes:{user_id}:{post_id}` → direction (up/down/none)
- **Why not direct DB write**: Hot posts get thousands of concurrent votes → DB contention. Kafka buffers and Redis absorbs reads

#### Ranking Algorithms — Deep Dive

**Hot Ranking (Reddit's actual algorithm)**:
```python
import math
from datetime import datetime

def hot_score(ups, downs, created_at):
    score = ups - downs
    order = math.log10(max(abs(score), 1))
    sign = 1 if score > 0 else -1 if score < 0 else 0
    
    # Epoch: Dec 8, 2005 (Reddit's birthday)
    epoch = datetime(2005, 12, 8, 7, 46, 43)
    seconds = (created_at - epoch).total_seconds()
    
    return round(sign * order + seconds / 45000, 7)
```
- **Key insight**: Time dominates. A post from 12.5 hours ago needs 10× the votes to rank equally with a new post
- The log scale means going from 10→100 votes has same effect as 100→1000

**Wilson Score (for "Best" comment ranking)**:
```python
import math

def wilson_score(ups, downs, confidence=0.95):
    n = ups + downs
    if n == 0:
        return 0
    z = 1.96  # 95% confidence
    p = ups / n
    
    return (p + z*z/(2*n) - z * math.sqrt((p*(1-p) + z*z/(4*n)) / n)) / (1 + z*z/n)
```
- Handles the "1 upvote, 0 downvote" vs "100 upvote, 1 downvote" problem
- Lower bound of confidence interval → more data = more confident

**Controversial**:
```python
def controversial_score(ups, downs):
    if ups <= 0 or downs <= 0:
        return 0
    magnitude = ups + downs
    balance = min(ups, downs) / max(ups, downs)  # closer to 1 = more controversial
    return magnitude * balance
```

#### Feed Service
- Home feed = aggregation from subscribed subreddits
- **Pre-computation**: Background job periodically computes top posts per subreddit → stores in Redis
- **On request**: Merge top posts from subscribed subreddits, interleave by score
- **Popular/All**: Global feed → single pre-computed ranked list in Redis

#### Search Service (Elasticsearch)
- **Indexed**: Post title, body text, subreddit name, author username
- **Features**: Full-text search with BM25 ranking, filter by subreddit/time range/post type, sort by relevance or recency
- **Sync**: Post creates/updates → Kafka → Elasticsearch consumer indexes in near real-time

#### Moderation Pipeline
- Consumes post and comment events from Kafka for automated moderation:
  - **Spam ML model**: Detect spam posts (link farms, repetitive content, new account patterns)
  - **Toxicity classifier**: Flag toxic/hateful content for review or auto-removal
  - **AutoMod rules**: Per-subreddit regex rules (e.g., "remove posts with banned keywords", "require minimum karma to post")
  - **Report queue**: User reports → prioritized queue for human moderators
  - **Auto-removal**: Content matching high-confidence spam/toxicity → removed immediately, mod notified

---

## 5. APIs

### Create Post
```http
POST /api/v1/subreddits/{subreddit_name}/posts
{
  "title": "System Design Tips",
  "content": "Here are my top tips...",
  "type": "text",
  "flair": "Discussion"
}
```

### Get Post with Comments
```http
GET /api/v1/posts/{post_id}?sort=best&depth=5&limit=200
Response:
{
  "post": { ... },
  "comments": [
    {
      "comment_id": "...",
      "content": "Great post!",
      "score": 150,
      "depth": 0,
      "replies": [
        {
          "comment_id": "...",
          "content": "Agreed!",
          "score": 42,
          "depth": 1,
          "replies": [...]
        }
      ]
    }
  ],
  "has_more_comments": true
}
```

### Vote
```http
POST /api/v1/vote
{
  "entity_type": "post",
  "entity_id": "post-uuid",
  "direction": 1    // 1 = upvote, -1 = downvote, 0 = remove vote
}
```

### Get Subreddit Feed
```http
GET /api/v1/subreddits/{name}/posts?sort=hot&cursor=...&limit=25
```

### Add Comment
```http
POST /api/v1/posts/{post_id}/comments
{
  "parent_comment_id": "comment-uuid",  // null for top-level
  "content": "This is a great point!"
}
```

---

## 6. Data Model

### PostgreSQL — Posts (Sharded by post_id)

```sql
CREATE TABLE posts (
    post_id         BIGINT PRIMARY KEY,
    subreddit_id    INT NOT NULL,
    user_id         BIGINT NOT NULL,
    title           VARCHAR(300),
    content         TEXT,
    url             TEXT,
    media_url       TEXT,
    post_type       ENUM('text', 'link', 'image', 'video'),
    score           INT DEFAULT 0,
    upvotes         INT DEFAULT 0,
    downvotes       INT DEFAULT 0,
    comment_count   INT DEFAULT 0,
    hot_score       FLOAT,
    is_locked       BOOLEAN DEFAULT FALSE,
    is_removed      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP,
    INDEX idx_subreddit_hot (subreddit_id, hot_score DESC),
    INDEX idx_subreddit_new (subreddit_id, created_at DESC),
    INDEX idx_subreddit_top (subreddit_id, score DESC)
);
```

### PostgreSQL — Comments (Materialized Path)

```sql
CREATE TABLE comments (
    comment_id      BIGINT PRIMARY KEY,
    post_id         BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    parent_id       BIGINT,                    -- null for top-level
    path            VARCHAR(1024),             -- "001.003.007.015"
    depth           SMALLINT,
    content         TEXT,
    score           INT DEFAULT 0,
    upvotes         INT DEFAULT 0,
    downvotes       INT DEFAULT 0,
    is_removed      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP,
    INDEX idx_post_path (post_id, path),
    INDEX idx_post_score (post_id, depth, score DESC)
);
```

**Querying comment tree**:
```sql
-- Get all top-level comments sorted by "best"
SELECT * FROM comments 
WHERE post_id = ? AND depth = 0 
ORDER BY wilson_score DESC LIMIT 25;

-- Get all replies to a comment
SELECT * FROM comments 
WHERE post_id = ? AND path LIKE '001.003.%' 
ORDER BY path;
```

### Redis — Vote Counts & User Votes

```
# Score cache
Key:    score:post:{post_id}
Value:  Hash { upvotes: 150, downvotes: 3, hot_score: 123456.789 }

# User's vote on a post (to show UI state and prevent double-voting)
Key:    uv:{user_id}:{post_id}
Value:  1 | -1 | (deleted if removed)
TTL:    30 days
```

### Redis — Ranked Feeds

```
# Subreddit hot feed (pre-computed)
Key:    feed:sub:{subreddit_id}:hot
Type:   Sorted Set
Members: post_id
Scores:  hot_score

# Home feed for user
Key:    feed:home:{user_id}
Type:   Sorted Set
Members: post_id
Scores:  hot_score
TTL:    300 (5 min, rebuilt by background job)
```

### Kafka Topics

```
Topic: post-events       (new post, edit, delete)
Topic: comment-events    (new comment, edit, delete)
Topic: vote-events       (upvote, downvote, remove)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Vote count accuracy** | Kafka ensures no vote is lost; Redis is periodically reconciled with DB |
| **Comment tree corruption** | Materialized path is append-only (new comments add to path); no destructive updates |
| **Hot score staleness** | Background job recomputes hot scores every 5 minutes; Kafka consumers update in near real-time for popular posts |
| **DB hot spot (popular post)** | Read replicas for post reads; Redis caches for vote counts |
| **Ranking computation failure** | Stale rankings served from Redis (still functional, just not perfectly fresh) |

### Specific: Handling Vote Manipulation
- **Challenge**: Bots mass-upvoting/downvoting
- **Solutions**:
  - One vote per user per entity (enforced at DB level with unique constraint)
  - Rate limiting votes per user (max 100 votes/minute)
  - "Vote fuzzing": Display slightly randomized scores to confuse bots
  - Shadow banning: Bot's votes are accepted but not counted
  - IP-based detection: Multiple accounts voting from same IP

---

## 8. Additional Considerations

### "More Comments" / Lazy Loading
- For posts with 10K+ comments, don't load all at once
- Load top 200 top-level comments sorted by "best"
- Each comment shows reply count; clicking "load replies" fetches children
- Use cursor-based pagination for deep threads

### Subreddit Moderation
- Moderators can: remove posts/comments, ban users, set rules, configure automod
- AutoModerator: Rule-based system (regex patterns, account age requirements, karma thresholds)
- Moderation queue: Reported content queued for moderator review

### Cross-posting
- A post can appear in multiple subreddits (cross-post)
- Stored as a separate post with `crosspost_parent_id` reference
- Votes are independent per crosspost

### Caching Strategy
```
L1: Client-side cache (browser/app)     — 60s TTL
L2: CDN cache (for popular subreddits)  — 30s TTL
L3: Redis (ranked feeds, vote counts)   — 5 min TTL
L4: PostgreSQL (source of truth)
```

