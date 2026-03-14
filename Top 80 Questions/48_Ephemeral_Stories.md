# 48. Design a Status Update Service with Ephemeral Content (Stories)

---

## 1. Functional Requirements (FR)

- **Post stories**: Upload photo/video stories that auto-delete after 24 hours
- **View stories**: Users see stories from people they follow in a tray/carousel at the top
- **Story ring**: Colored ring indicator shows unviewed stories
- **View tracking**: Track who viewed your story; show viewer list to author
- **Story reactions**: Reply to a story (sends a DM) or react with emoji
- **Close friends**: Post stories visible only to a selected "close friends" list
- **Highlights**: Pin select stories to profile permanently (opt out of deletion)
- **Story ordering**: Show stories from closest friends first, then by recency

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Story tray loads in < 300 ms; story content in < 500 ms
- **Auto-Deletion**: Stories MUST disappear after exactly 24 hours (hard TTL)
- **High Throughput**: 500M+ stories/day, 10B+ story views/day
- **Availability**: 99.99%
- **CDN**: Media served from edge; < 100 ms first byte
- **Consistency**: View tracking must be accurate (no missed views)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 500M |
| Stories posted / day | 500M |
| Story views / day | 10B |
| Avg story media size | 2 MB |
| Upload storage / day | 500M × 2 MB = 1 PB (but deleted after 24h!) |
| Peak concurrent story views / sec | 200K |
| Unique viewers per story (avg) | 50 |
| Storage steady state (24h window) | ~1 PB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Client (Mobile App)                         │
│                                                                       │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│  │  Story Tray     │  │  Camera /    │  │  Story Viewer        │    │
│  │  (Horizontal    │  │  Upload      │  │  (Full-screen,       │    │
│  │   scroll at top)│  │              │  │   auto-advance)      │    │
│  │                 │  │  Capture →   │  │                      │    │
│  │  Shows ring     │  │  Edit →      │  │  Pre-fetch next 3    │    │
│  │  indicators:    │  │  Audience →  │  │  stories while       │    │
│  │  🔵 = unviewed  │  │  Publish     │  │  viewing current     │    │
│  │  ⚪ = all viewed │  │              │  │                      │    │
│  └────────┬────────┘  └──────┬───────┘  └────────┬─────────────┘    │
└───────────┼──────────────────┼────────────────────┼─────────────────┘
            │                  │                    │
            ▼                  ▼                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         API Gateway                                  │
└───────┬──────────────┬───────────────────┬───────────────────────────┘
        │              │                   │
┌───────▼──────┐ ┌─────▼───────┐   ┌──────▼──────┐
│ Story Tray   │ │ Upload      │   │ View        │
│ Service      │ │ Service     │   │ Tracking    │
│              │ │             │   │ Service     │
│ "Whose stories│ │             │   │             │
│  should I    │ │             │   │ Record who  │
│  show?"      │ │             │   │ viewed what │
└───────┬──────┘ └─────┬───────┘   └──────┬──────┘
        │              │                   │
        │         ┌────▼──────────────────────────────────────────┐
        │         │         Story Upload Pipeline                 │
        │         │                                                │
        │         │  1. Client uploads media → S3 (pre-signed URL)│
        │         │  2. Create metadata in Cassandra:              │
        │         │     status = "processing"                      │
        │         │  3. Kafka event → Media Processing Workers:    │
        │         │     • Resize (multiple resolutions)            │
        │         │     • Generate thumbnail                       │
        │         │     • Content moderation (nudity/violence ML)  │
        │         │  4. Upload processed media → S3 + CDN warm-up │
        │         │  5. Update metadata: status = "active"         │
        │         │  6. Set TTL: auto-delete at posted_at + 24h   │
        │         │  7. Fan-out to followers' story trays (Redis)  │
        │         └──────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────────────┐
│                    Story Tray Generation                             │
│                                                                       │
│  For user U requesting their story tray:                             │
│                                                                       │
│  1. Get list of users U follows (Social Graph Service)              │
│     → [Alice, Bob, Carol, ...] (1000 followings max)                │
│                                                                       │
│  2. For each followed user, check if they have active stories:      │
│     Redis: SISMEMBER active_stories {followed_user_id}              │
│     → Filter to only users with active stories                      │
│     → [Alice, Carol] (150 have stories)                             │
│                                                                       │
│  3. For each, check if U has viewed ALL their stories:              │
│     Redis: story_viewed:{U}:{author_id} vs story_count:{author_id} │
│     → Unviewed: ring = 🔵, Viewed: ring = ⚪                        │
│                                                                       │
│  4. Rank:                                                            │
│     a. Unviewed stories first                                       │
│     b. Among unviewed: closest friends first (interaction score)     │
│     c. Among viewed: recency (most recently posted first)           │
│                                                                       │
│  5. Cache result in Redis: tray:{U} = [author_ids] (TTL: 5 min)   │
│                                                                       │
│  6. Return first 20 story authors with metadata                     │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                      Storage Architecture                            │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ S3 + CDN     │  │  Cassandra   │  │   Redis      │               │
│  │              │  │              │  │              │               │
│  │ Media files  │  │ Story meta:  │  │ active_stories│              │
│  │ (auto-delete │  │ story_id,    │  │  = SET of     │              │
│  │  via S3      │  │ author_id,   │  │  user_ids with│              │
│  │  lifecycle   │  │ media_url,   │  │  active stories│             │
│  │  policy,     │  │ audience,    │  │               │              │
│  │  24h expiry) │  │ posted_at,   │  │ story_viewed: │              │
│  │              │  │ expires_at   │  │  {viewer}:{author}│          │
│  │              │  │              │  │  = SET of      │              │
│  │              │  │ TTL = 24h    │  │  viewed story_ │              │
│  │              │  │ (auto-delete)│  │  ids           │              │
│  │              │  │              │  │               │              │
│  │              │  │ Story viewers│  │ tray:{uid}    │              │
│  │              │  │ (write-heavy)│  │  = cached tray │              │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└──────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### 24-Hour Auto-Deletion — The Key Challenge

```
Stories MUST disappear after exactly 24 hours. Zero tolerance.

Approach 1: Background deletion job (cron)
  Every minute: DELETE FROM stories WHERE expires_at < NOW()
  ✗ Up to 60-second delay after expiry
  ✗ At 500M stories/day → scanning is expensive
  ✗ Media files in S3 need separate cleanup

Approach 2: TTL-based deletion ⭐ (recommended)
  Cassandra: INSERT ... USING TTL 86400 (24 hours)
    → Cassandra automatically deletes the row after TTL expires
    → Zero application-level cleanup needed
  
  S3: Lifecycle policy on the bucket
    → Objects in /stories/ prefix expire after 24 hours
    → S3 handles deletion automatically
  
  Redis: EXPIRE key 86400
    → All story-related keys auto-expire
  
  Result: Three layers of automatic cleanup, no cron jobs needed.
  
  Edge case: "Highlights" — user pins a story to their profile
    → Copy story metadata + media to a permanent table/bucket
    → Original story still expires; highlight copy is permanent
```

#### View Tracking — High-Volume Writes

```
Every time a user views a story → write a view record:
  {viewer_id, story_id, viewed_at}

At 10B views/day = ~115K writes/sec → needs high write throughput

Cassandra table:
  CREATE TABLE story_viewers (
      story_id    UUID,
      viewer_id   UUID,
      viewed_at   TIMESTAMP,
      PRIMARY KEY (story_id, viewer_id)  -- dedup: one view per user
  ) WITH default_time_to_live = 86400;   -- auto-delete after 24h

Redis for fast check (has user viewed this story?):
  SADD story_viewed:{viewer_id}:{author_id} {story_id}
  EXPIRE story_viewed:{viewer_id}:{author_id} 86400

View count per story:
  INCR story_views:{story_id}
  EXPIRE story_views:{story_id} 86400
```

---

## 5. APIs

### Post Story
```http
POST /api/v1/stories
{ "media_url": "s3://...", "audience": "everyone" | "close_friends",
  "stickers": [...], "mentions": ["@alice"] }
→ 201 Created { "story_id": "s-uuid", "expires_at": "2026-03-15T10:00:00Z" }
```

### Get Story Tray
```http
GET /api/v1/stories/tray
→ 200 OK
{ "tray": [
    { "user_id": "u-alice", "username": "alice", "avatar": "...",
      "has_unviewed": true, "story_count": 3, "latest_at": "..." },
    ...
]}
```

### View Story (Record + Fetch Content)
```http
GET /api/v1/stories/{story_id}
→ 200 OK { "story_id": "...", "media_url": "https://cdn.../...",
           "posted_at": "...", "view_count": 342 }
// Also records view event server-side
```

### Get Story Viewers (Author Only)
```http
GET /api/v1/stories/{story_id}/viewers?cursor=0&limit=50
→ 200 OK { "viewers": [{"user_id": "...", "name": "...", "viewed_at": "..."}] }
```

---

## 6. Data Model

### Cassandra — Story Metadata + Viewers
```sql
CREATE TABLE stories (
    author_id   UUID,
    story_id    TIMEUUID,
    media_url   TEXT,
    media_type  TEXT,        -- image, video
    audience    TEXT,        -- everyone, close_friends
    posted_at   TIMESTAMP,
    PRIMARY KEY (author_id, story_id)
) WITH CLUSTERING ORDER BY (story_id DESC)
  AND default_time_to_live = 86400;

CREATE TABLE story_viewers (
    story_id    UUID,
    viewer_id   UUID,
    viewed_at   TIMESTAMP,
    PRIMARY KEY (story_id, viewer_id)
) WITH default_time_to_live = 86400;
```

### Redis
```
active_stories              → SET of user_ids who have active stories
tray:{user_id}              → List of author_ids (cached story tray, TTL: 5 min)
story_viewed:{viewer}:{author} → SET of viewed story_ids (TTL: 24h)
story_views:{story_id}      → Integer (view count, TTL: 24h)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Story not deleted after 24h** | Triple TTL: Cassandra TTL + S3 lifecycle + Redis EXPIRE |
| **View tracking loss** | Kafka buffer before Cassandra write; at-least-once delivery |
| **Media processing failure** | Retry 3×; if still failing → mark story as "processing_failed" |
| **Story tray staleness** | Cache TTL = 5 min; manual refresh on pull-to-refresh |
| **Close friends privacy leak** | Audience check at story-fetch time, not just at tray level |

### Race Conditions

#### Story Expires While User Is Viewing

```
User opens story tray at T=23:59:55 (story expires at T+0:00:00)
User starts viewing story at T=23:59:58
Story TTL expires at T=0:00:00 → Cassandra deletes row

If user taps "viewers list" at T=0:00:03 → story_id not found → 404!

Solution: Soft expiry window
  Display until: expires_at + 5 min (grace period for active viewers)
  TTL in Cassandra: 86400 + 300 (24h + 5 min)
  UI shows "Expired" badge during grace period
  API returns story content during grace period but hides from tray
```

#### Fan-Out for Story Tray Updates

```
Alice posts a story → need to update story trays for all her followers

Pull model (lazy) ⭐:
  Don't fan-out. On tray request, check each followed user.
  For 1000 followings: 1000 Redis SISMEMBER checks → ~5 ms with pipeline
  
Push model (eager):
  On story post → write to all followers' tray lists in Redis
  For a user with 1M followers → 1M Redis writes → expensive
  
Hybrid:
  Push for regular users (< 10K followers) → fast tray loads
  Pull for celebrities (> 10K followers) → avoid expensive fan-out
```

---

## 8. Deep Dive: Engineering Trade-offs

### Cassandra TTL vs Application-Level Deletion

```
Cassandra TTL ⭐:
  ✓ Zero application code for deletion
  ✓ Handles tombstone cleanup automatically (compaction)
  ✗ Tombstones accumulate → read performance degrades until compaction
  ✗ Cannot "undo" a TTL (must re-insert to keep)
  
  Tombstone issue: After 500M stories expire/day, Cassandra creates
  500M tombstones. Until compaction runs → range queries slow down.
  
  Mitigation: 
    - Time-Window Compaction Strategy (TWCS) ⭐
    - Each SSTable covers a time window (1 hour)
    - When entire window expires → drop entire SSTable (no tombstones!)
    - Perfect for TTL-heavy workloads like stories

Application-level deletion:
  ✓ Full control over when and how
  ✗ Must implement cron job, handle failures, maintain consistency
  ✗ More operational burden

For stories: Cassandra TTL + TWCS compaction is the best fit.
```

