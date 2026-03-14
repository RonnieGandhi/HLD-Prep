# 44. Design TikTok (Short Video Platform)

---

## 1. Functional Requirements (FR)

- **Upload short videos**: 15s to 10min videos with music, effects, filters
- **For You Page (FYP)**: AI-driven personalized infinite-scroll video feed
- **Social features**: Like, comment, share, follow, duet, stitch
- **Video creation tools**: In-app recording, editing, effects, music overlay
- **Search & Discover**: Search by hashtags, sounds, users, trending content
- **Notifications**: Likes, comments, new followers, trending
- **Creator analytics**: Views, likes, shares, audience demographics
- **Live streaming**: Real-time broadcast with gifts/donations
- **Monetization**: Creator fund, brand partnerships, in-app purchases

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency Feed**: FYP loads in < 500 ms with pre-fetched videos
- **High Throughput**: Serve 1B+ video plays/day
- **Global CDN**: Videos cached at edge worldwide, < 100 ms first byte
- **Recommendation Quality**: FYP must be highly engaging (core product differentiator)
- **Upload Processing**: Video available within 5 minutes of upload
- **Availability**: 99.99%
- **Scalability**: 1B+ MAU, 500M+ DAU

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 500M |
| Videos watched / user / day | 150 (avg session 30-60 min) |
| Total video plays / day | 75B |
| Video uploads / day | 10M |
| Avg video size (after encoding) | 15 MB (multi-bitrate) |
| Upload storage / day | 10M × 15 MB = 150 TB |
| CDN bandwidth | 75B plays × 3 MB avg = 225 PB/day |
| Feed requests / sec | ~870K |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Client (Mobile App)                           │
│                                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ Feed Player  │  │ Upload +     │  │ Social       │                │
│  │              │  │ Creation     │  │ (Like,       │                │
│  │ • Pre-buffer │  │ Tools        │  │  Comment,    │                │
│  │   next 3     │  │              │  │  Follow)     │                │
│  │   videos     │  │ • Record     │  │              │                │
│  │ • Adaptive   │  │ • Edit/Trim  │  │              │                │
│  │   bitrate    │  │ • Effects    │  │              │                │
│  │ • Swipe UX   │  │ • Music      │  │              │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
└─────────┼──────────────────┼──────────────────┼──────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        API Gateway / L7 LB                           │
│  Rate limiting, auth (JWT), routing, geo-routing                     │
└──────┬───────────────┬────────────────┬──────────────────────────────┘
       │               │                │
┌──────▼──────┐ ┌──────▼──────┐ ┌───────▼───────┐
│ Feed Service│ │Upload Service│ │Social Service │
└──────┬──────┘ └──────┬──────┘ └───────┬───────┘
       │               │                │
       │          ┌────▼──────────────────────────────────────────┐
       │          │         Video Upload & Processing Pipeline     │
       │          │                                                │
       │          │  1. Client uploads to S3 (pre-signed URL)      │
       │          │  2. Upload Service creates metadata in MySQL   │
       │          │  3. Kafka event: "video-uploaded"               │
       │          │  4. Transcoding Pipeline (GPU workers):        │
       │          │     • Transcode to HLS: 360p, 480p, 720p, 1080p│
       │          │     • Generate thumbnails (3 per video)         │
       │          │     • Extract audio fingerprint (music ID)     │
       │          │     • Content moderation (nudity, violence ML) │
       │          │     • Compute video embeddings (for rec system)│
       │          │  5. Output → S3 (HLS segments) + CDN warm-up  │
       │          │  6. Update metadata: status = "published"      │
       │          │  7. Feed to recommendation pipeline             │
       │          └────────────────────┬──────────────────────────┘
       │                               │
       │          ┌────────────────────▼──────────────────────────┐
       │          │    Recommendation System (ML Pipeline)         │
       │          │                                                │
       │          │  Feature Store (Redis):                        │
       │          │    user_features:{user_id}                     │
       │          │    = {watch_history_embedding, interests,      │
       │          │       location, age_group, language,            │
       │          │       time_of_day_preferences}                 │
       │          │                                                │
       │          │    video_features:{video_id}                   │
       │          │    = {content_embedding, hashtags, music,      │
       │          │       creator_id, completion_rate, like_rate,  │
       │          │       share_rate}                               │
       │          │                                                │
       │          │  Candidate Generation (recall):               │
       │          │    1. Collaborative filtering (users like you) │
       │          │    2. Content-based (videos similar to watched)│
       │          │    3. Trending/popular in region                │
       │          │    4. Creator-based (followed creators)         │
       │          │    → 10,000 candidates                         │
       │          │                                                │
       │          │  Ranking (precision):                          │
       │          │    Deep learning model (transformer/DNN)       │
       │          │    Input: user features + video features       │
       │          │    Output: P(watch_complete), P(like), P(share)│
       │          │    Score = weighted sum of predicted actions    │
       │          │    → Top 500 ranked videos                     │
       │          │                                                │
       │          │  Re-ranking (diversity):                       │
       │          │    • De-duplicate creators (max 2 per creator) │
       │          │    • Category diversity (not all dance videos)  │
       │          │    • Freshness boost (new videos get exposure)  │
       │          │    • Brand safety filter                        │
       │          │    → Final 200 videos for this session          │
       │          │                                                │
       │          │  Pre-compute: Every 30 min per active user     │
       │          │  Store in Redis: feed:{user_id} = [video_ids]  │
       │          └────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────────┐
│                         Feed Service                                 │
│                                                                       │
│  GET /feed:                                                          │
│  1. Read pre-computed feed from Redis: feed:{user_id}                │
│  2. If empty/expired → call Rec System synchronously (fallback)      │
│  3. Filter: remove already-watched videos (Redis SET watched:{uid}) │
│  4. Return next 10 videos with metadata + HLS URLs                   │
│  5. Client pre-buffers videos 2, 3, 4 while user watches video 1    │
│                                                                       │
│  Engagement tracking:                                                │
│  • Video start → Kafka event                                        │
│  • Watch duration → Kafka event (every 5 seconds heartbeat)          │
│  • Like/comment/share → Kafka event                                  │
│  • Swipe away (skip) → Kafka event                                   │
│  → All feed back into recommendation model training                  │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                      Storage Architecture                            │
│                                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ S3 + CDN    │  │   MySQL     │  │   Redis     │  │ Cassandra  │ │
│  │             │  │  (Vitess)   │  │   Cluster   │  │            │ │
│  │ Video files │  │             │  │             │  │ Comments,  │ │
│  │ HLS segments│  │ Users,      │  │ Feed cache, │  │ likes,     │ │
│  │ Thumbnails  │  │ Videos meta,│  │ Feature     │  │ user       │ │
│  │             │  │ Follows,    │  │ store,      │  │ activity   │ │
│  │ 225 PB/day  │  │ Music       │  │ Watched set │  │ (write-    │ │
│  │ served via  │  │             │  │             │  │  heavy)    │ │
│  │ CDN         │  │ Sharded by  │  │             │  │            │ │
│  │             │  │ user_id     │  │             │  │            │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. APIs

### Get Feed (For You Page)
```http
GET /api/v1/feed?count=10&cursor={last_video_id}
→ 200 OK
{
  "videos": [
    {
      "video_id": "v-uuid", "creator": {"id": "u1", "name": "Alice", "avatar": "..."},
      "description": "Dance challenge #fyp", "music": {"id": "m1", "title": "..."},
      "stats": {"plays": 1523000, "likes": 89200, "comments": 3400, "shares": 12300},
      "hls_url": "https://cdn.tiktok.com/v-uuid/playlist.m3u8",
      "thumbnail_url": "https://cdn.tiktok.com/v-uuid/thumb.jpg",
      "duration_sec": 45, "created_at": "2026-03-14T08:00:00Z"
    }, ...
  ],
  "cursor": "v-uuid-10"
}
```

### Upload Video
```http
POST /api/v1/videos/upload-url
→ 200 OK { "upload_url": "https://s3.../presigned", "video_id": "v-uuid" }

POST /api/v1/videos/{video_id}/publish
{ "description": "Dance challenge #fyp", "music_id": "m1", "privacy": "public" }
→ 202 Accepted { "status": "processing" }
```

---

## 6. Data Model

### MySQL (Vitess) — Video Metadata
```sql
CREATE TABLE videos (
    video_id        VARCHAR(36) PRIMARY KEY,
    creator_id      BIGINT NOT NULL,
    description     TEXT,
    music_id        BIGINT,
    duration_sec    SMALLINT,
    status          ENUM('processing','published','removed') DEFAULT 'processing',
    privacy         ENUM('public','private','friends') DEFAULT 'public',
    s3_key          VARCHAR(512),
    thumbnail_key   VARCHAR(512),
    view_count      BIGINT DEFAULT 0,
    like_count      BIGINT DEFAULT 0,
    comment_count   INT DEFAULT 0,
    share_count     INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_creator (creator_id, created_at DESC),
    INDEX idx_status_created (status, created_at DESC)
);
```

### Redis — Feed & Features
```
feed:{user_id}           → LIST of video_ids (pre-computed, 200 items)
watched:{user_id}        → SET of video_ids (TTL 30 days)
user_features:{user_id}  → Hash (embedding, interests, language)
video_features:{vid_id}  → Hash (embedding, completion_rate, like_rate)
```

### Cassandra — Engagement Data
```sql
CREATE TABLE video_likes (
    video_id    UUID,
    user_id     UUID,
    created_at  TIMESTAMP,
    PRIMARY KEY (video_id, user_id)
);

CREATE TABLE video_comments (
    video_id    UUID,
    comment_id  TIMEUUID,
    user_id     UUID,
    text        TEXT,
    PRIMARY KEY (video_id, comment_id)
) WITH CLUSTERING ORDER BY (comment_id DESC);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Transcoding failure** | Retry 3× with exponential backoff; dead-letter for manual review |
| **Recommendation cache miss** | Fallback: serve trending/popular videos in user's region |
| **CDN miss** | Origin pull from S3; CDN cache warm-up for predicted viral videos |
| **Feed service down** | Client caches last 50 videos locally; offline playback for downloaded |
| **Content moderation false negative** | User report → human review queue; takedown within 1 hour |

### Race Conditions

#### 1. Video Published Before Transcoding Complete
```
User publishes → metadata created → transcoding still running → 
another user searches and finds it → tries to play → 404!

Solution: Video status = "processing" until ALL transcode variants ready.
  Feed/search service only returns videos with status = "published".
  Client shows "Processing..." placeholder on creator's profile.
```

#### 2. View Count Accuracy Under Viral Load
```
Same approach as Like Count (file 41):
  Redis INCR for real-time display → Kafka → batch write to MySQL
  View count can lag by seconds — invisible at "1.5M views" display level
  
  Anti-fraud: Don't count repeated views from same user within 30 seconds
  Redis: SETEX view_dedup:{video_id}:{user_id} 1 30
```

---

## 8. Deep Dive: Engineering Trade-offs

### Recommendation: Two-Tower vs Single-Tower Model

```
Two-Tower (TikTok's actual approach) ⭐:
  Tower 1: User encoder → user embedding vector (128-dim)
  Tower 2: Video encoder → video embedding vector (128-dim)
  Score = dot_product(user_embedding, video_embedding)
  
  ✓ Video embeddings can be pre-computed (offline)
  ✓ User embedding computed once per session → fast inference
  ✓ Approximate Nearest Neighbor (ANN) search for candidate generation
  ✗ Can't capture cross-features (user×video interaction features)

Single-Tower (for re-ranking):
  Input: concat(user_features, video_features, context_features)
  → Deep neural network → P(engagement)
  
  ✓ Captures complex interactions
  ✗ Must run inference for EACH (user, video) pair → expensive
  ✗ Can't pre-compute → only used for top 500 candidates (not 10M)

TikTok's pipeline:
  Two-Tower (recall): 10M videos → 10K candidates (fast, pre-computed)
  Single-Tower (rank): 10K candidates → 500 ranked (precise, expensive)
  Business rules (re-rank): 500 → 200 final (diversity, safety)
```

### Pre-Computed vs On-Demand Feed Generation

```
Pre-computed ⭐ (TikTok's approach):
  Background job computes feed for each active user every 30 minutes
  Stores top 200 video_ids in Redis: feed:{user_id}
  
  ✓ Feed loads in < 50 ms (just Redis read)
  ✓ Recommendation model has time to run complex inference
  ✗ Stale: doesn't reflect last 30 minutes of activity
  ✗ Compute cost: 500M DAU × every 30 min = 1B feed generations/day
  
  Optimization: Only recompute for users who were active in last 30 min
  If user inactive for 1 hour → use last cached feed → still good enough
  
On-demand:
  Each feed request → call recommendation service → wait for inference
  
  ✓ Always fresh
  ✗ Latency: 200-500 ms for model inference → bad UX for first video
  ✗ Compute spike at peak hours
  
Best: Pre-computed feed + on-demand refresh when user exhausts cached feed
  Client pre-fetches next batch before current batch runs out.
```

### Content Moderation Pipeline — The Trust & Safety Layer

```
Every uploaded video MUST be moderated before reaching users.

Pipeline (in transcoding workers):
  Stage 1: Automated ML classifiers (< 5 seconds)
    • Nudity/sexual content detection (image classifier on keyframes)
    • Violence/gore detection
    • Hate speech detection (text classifier on captions + OCR on video text)
    • Copyright music detection (audio fingerprint → match database)
    • Spam/scam detection (metadata patterns)
    
    Result: confidence score [0, 1] per category
    
  Stage 2: Decision routing
    confidence > 0.95 → AUTO-REJECT (block immediately, notify creator)
    confidence 0.7-0.95 → HUMAN REVIEW QUEUE (hold, don't publish)
    confidence < 0.7 → AUTO-APPROVE (publish, but monitor)
    
  Stage 3: Human review (for borderline cases)
    Queue: 10K-50K videos/day need human review
    SLA: review within 2 hours
    Reviewers: trained content moderators with escalation to policy team
    
  Stage 4: Post-publish monitoring
    Published videos continuously monitored via user reports
    If video gets > 10 reports → auto-deprioritize in recommendations
    If > 50 reports → remove from recommendations, flag for review
    
  Appeals: Creator can appeal removal → second human review
  
  False positive rate target: < 1% (wrongly removed)
  False negative rate target: < 0.1% (harmful content reaching users)
```

### Engagement Signals — What the Algorithm REALLY Optimizes

```
TikTok's "secret sauce" is the engagement signal hierarchy:

  Signal 1: Watch completion rate (MOST important)
    Did the user watch the full video? 2nd loop? 3rd loop?
    Completion_rate = watch_time / video_duration
    If completion_rate > 1.0 → user LOVED it (rewatched)
    This signal is 10× more predictive than likes
    
  Signal 2: Share (very strong positive)
    Sharing = "I want my friends to see this"
    Strongest explicit signal of quality
    
  Signal 3: Comment (strong positive)
    Even negative comments = engagement
    Videos with high comment rate get boosted
    
  Signal 4: Like (moderate positive)
    Easy action, high frequency → somewhat noisy signal
    
  Signal 5: Follow after watching (strong positive)
    "This creator's content is consistently good"
    
  Signal 6: Skip / swipe away (negative)
    Watch < 3 seconds then swipe → strong negative signal
    
  Signal 7: "Not interested" (explicit negative)
    User explicitly marks → downweight similar content heavily

Training data:
  Every user session generates training examples:
    (user_features, video_features) → {completed, liked, shared, skipped}
  Model retrained daily with latest data → adapts to trends within 24 hours
```

### CDN Optimization — Serving 225 PB/Day

```
225 PB/day = 2.6 GB/sec average, peaks at 5+ GB/sec

Optimization strategies:

  1. HLS Adaptive Bitrate:
     360p (0.5 Mbps), 480p (1.5 Mbps), 720p (3 Mbps), 1080p (5 Mbps)
     Client starts at lowest → upgrades based on bandwidth
     Saves bandwidth: 60% of views are on mobile → 480p sufficient
     
  2. Predictive pre-warming:
     When video starts going viral (velocity > threshold):
       → Push to CDN edge nodes in likely regions BEFORE requests arrive
       → Avoid origin pull thundering herd
     
  3. Short video = small files = high cache hit rate:
     30-second video at 720p = ~5 MB
     CDN edge server with 1 TB cache → holds 200K videos
     Top 200K videos cover 80%+ of views → 80% cache hit rate
     
  4. Range requests for seek:
     HLS segments: 2-second chunks
     User scrolls to middle → only fetch that segment → low TTFB
```

