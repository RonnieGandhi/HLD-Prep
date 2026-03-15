# 9. Design Instagram (Photo Sharing + Social Features)

---

## 1. Functional Requirements (FR)

- **Upload photos/videos** with captions, tags, location, filters
- **News feed**: Personalized feed of posts from followed users
- **Stories**: Ephemeral content that disappears after 24 hours
- **Follow / Unfollow** users
- **Like, Comment, Save** posts
- **Explore page**: Discover new content based on interests
- **Direct Messaging** (DMs)
- **User profiles** with grid of posts, follower/following counts
- **Hashtags** and location-based discovery
- **Notifications** for likes, comments, follows, mentions

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99%
- **Low Latency**: Feed loads in < 200 ms, image renders in < 500 ms
- **Scalability**: 2B+ registered users, 500M+ DAU
- **Durability**: Uploaded photos/videos must never be lost
- **Read-Heavy**: 100:1 read to write ratio
- **Eventual Consistency**: OK for feed, likes count, follower count
- **Global**: CDN for media delivery worldwide

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 500M |
| Photo uploads / day | 100M |
| Avg photo size (original) | 3 MB |
| Photo storage / day | 100M × 3 MB = 300 TB |
| Photo storage / year | ~110 PB |
| Resized versions per photo | 4 (thumbnail, small, medium, large) |
| Total storage with resizes | 300 TB × 2 = 600 TB/day |
| Feed reads / day | 5B |
| Stories uploads / day | 500M |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │
│  (iOS/Android│
│   /Web)      │
└──────┬───────┘
       │
┌──────▼───────┐
│    CDN       │  ← Serves images, videos, stories (edge-cached)
│ (CloudFront) │
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │  ← Auth (JWT), rate limiting, geo-routing
└──────┬───────┘
       │
       ├──────────┬──────────┬──────────┬──────────┬──────────┐
       │          │          │          │          │          │
┌──────▼──┐ ┌────▼────┐ ┌───▼───┐ ┌───▼───┐ ┌───▼───┐ ┌───▼────┐
│ Post    │ │ Feed    │ │Story  │ │Search │ │Explore│ │Social  │
│ Service │ │ Service │ │Service│ │Service│ │Service│ │Service │
│         │ │         │ │       │ │       │ │       │ │(Follow,│
│ Upload  │ │ Hybrid  │ │24h TTL│ │Users, │ │Content│ │ Like,  │
│ + meta  │ │ fan-out │ │ ephem.│ │ Tags, │ │ reco  │ │ Comment│
│ + event │ │ + rank  │ │       │ │ Locs  │ │ ML    │ │ Notif) │
└────┬────┘ └────┬────┘ └───┬───┘ └───┬───┘ └───┬───┘ └───┬────┘
     │           │          │         │         │         │
     │      ┌────▼────┐     │    ┌────▼───┐  ┌──▼──────┐ │
     │      │ Redis   │     │    │Elastic │  │ ML      │ │
     │      │(Feed    │     │    │Search  │  │ Ranking │ │
     │      │ Cache   │     │    │(users, │  │ Service │ │
     │      │ per user│     │    │ tags,  │  │(collab  │ │
     │      │ ZSET)   │     │    │ places)│  │ filter +│ │
     │      └─────────┘     │    └────────┘  │ content)│ │
     │                      │                └─────────┘ │
     │                      │                            │
┌────▼──────────────────────▼────────────────────────────▼────┐
│                          Kafka                               │
│  (post-events, story-events, like-events, follow-events,    │
│   comment-events, notification-events)                       │
└────┬────────────────┬──────────────────┬────────────────┬───┘
     │                │                  │                │
┌────▼────┐    ┌──────▼───────┐   ┌──────▼──────┐  ┌─────▼──────┐
│Fan-Out  │    │ Media        │   │ Notification│  │ Analytics  │
│Service  │    │ Processing   │   │ Service     │  │ Pipeline   │
│         │    │ Pipeline     │   │             │  │            │
│ Normal  │    │              │   │ • Push      │  │ Flink →    │
│ user:   │    │ • Resize 4   │   │   (APNs/   │  │ ClickHouse │
│ push to │    │   sizes      │   │   FCM)     │  │            │
│ follower│    │ • Blurhash   │   │ • "X liked │  │ Engagement │
│ feeds   │    │ • EXIF strip │   │   your     │  │ metrics,   │
│         │    │ • Content    │   │   photo"   │  │ creator    │
│ Celeb:  │    │   moderation │   │ • "X       │  │ analytics  │
│ skip    │    │   (NSFW ML)  │   │   started  │  │            │
│ (pull   │    │ • Video:     │   │   following│  │            │
│  on     │    │   transcode  │   │   you"     │  │            │
│  read)  │    │   HLS multi- │   │ • Batch:   │  │            │
│         │    │   bitrate    │   │   "and 42  │  │            │
│         │    │ • CDN warm   │   │   others"  │  │            │
└────┬────┘    └──────┬───────┘   └─────────────┘  └────────────┘
     │                │
┌────▼────────────────▼──────────────────────────────────────────┐
│                       DATA LAYER                                │
│                                                                  │
│ ┌──────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│ │ Cassandra /  │  │   S3     │  │  Redis   │  │ MySQL        │ │
│ │ DynamoDB     │  │ (Object  │  │          │  │ (Vitess)     │ │
│ │              │  │  Storage)│  │ • Feed   │  │              │ │
│ │ • Posts      │  │          │  │   cache  │  │ • Users      │ │
│ │ • Feeds      │  │ • Images │  │ • Like   │  │ • Follows    │ │
│ │ • Stories    │  │   (4 sizes│  │   counts │  │ • Blocks     │ │
│ │ • Comments   │  │   per    │  │ • Story  │  │ • Auth       │ │
│ │ • User feed  │  │   image) │  │   tray   │  │ • Settings   │ │
│ │   timelines  │  │ • Videos │  │ • Session│  │              │ │
│ │              │  │   (HLS)  │  │ • Explore│  │ Source of    │ │
│ │ Write-       │  │ • Story  │  │   cache  │  │ truth for    │ │
│ │ optimized    │  │   media  │  │          │  │ social graph │ │
│ └──────────────┘  └──────────┘  └──────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Post Service
- Handles photo/video upload and post creation
- **Upload Flow**:
  1. Client requests a pre-signed S3 URL from Post Service
  2. Client uploads media directly to S3 (avoids routing through our servers)
  3. Client sends post metadata (caption, tags, location) to Post Service
  4. Post Service writes to Posts DB
  5. Publishes `post-created` event to Kafka

#### Media Processing Pipeline
- **Triggered by**: S3 event notification or Kafka event
- **Processing steps**:
  1. **Resize**: Generate 4 versions (150×150 thumbnail, 320×320, 640×640, 1080×1080)
  2. **Apply filters**: If user selected a filter, apply server-side (or client-side pre-upload)
  3. **Generate blurhash**: Low-res placeholder for progressive loading
  4. **Extract EXIF**: GPS coordinates, camera info (strip sensitive data)
  5. **Content moderation**: NSFW detection, violence detection (ML model)
  6. **Video processing**: Transcode to multiple resolutions (360p, 720p, 1080p), generate thumbnails
- **Technology**: AWS Lambda for image processing, FFmpeg for video, GPU instances for ML
- **Output**: Multiple resized images stored back in S3, CDN URLs generated

#### Feed Service (Hybrid Fan-Out)
- Same hybrid approach as Twitter/Facebook (see News Feed design)
- Normal users: fan-out on write to Redis
- Celebrities: fan-out on read
- **Feed ranking**: ML model considers:
  - Relationship strength (how often you interact with the poster)
  - Post recency
  - Engagement velocity (how fast it's getting likes)
  - Content type (photos vs. videos vs. carousels)
  - User's historical preferences

#### Story Service
- **24-hour ephemeral content**
- **Storage**: S3 with TTL (lifecycle policy deletes after 24 hours)
- **Serving**: Stories for users you follow are pre-fetched
- **Data model**: Cassandra with TTL = 86400 seconds
- **Viewing**: Stories are displayed in a horizontal carousel, ordered by recency and relationship strength
- **Story tray**: Pre-computed list of users who have active stories → stored in Redis

#### Explore Service
- **Purpose**: Surface interesting content from users you don't follow
- **How**: 
  1. Collaborative filtering: "Users similar to you liked these posts"
  2. Content-based: Posts with hashtags/locations matching your interests
  3. Engagement signals: High like velocity posts
- **Implementation**: 
  - Offline: Spark job computes candidate posts per interest cluster
  - Online: ML ranking model scores candidates for each user
  - Cached in Redis with TTL

#### Social Service (Follow, Like, Comment)
- **Follow**: Write to MySQL (source of truth for social graph) + update Redis follower sets + publish `follow-event` to Kafka
- **Like**: Atomic check-and-set in Redis (`SADD liked:{post_id} {user_id}`) + increment counter (`INCR like_count:{post_id}`) + Kafka event for async DB write + notification
- **Comment**: Write to Cassandra (partition key = post_id, clustering = comment_id for time ordering) + Kafka event → Notification Service
- **Dedup**: Redis SET for `liked:{post_id}` prevents double-likes; idempotent on retry
- **Anti-spam**: Rate limit comments (max 20/min per user), ML-based spam classifier on comment text

#### Notification Service
- Consumes events from Kafka (`like-events`, `follow-events`, `comment-events`, `mention-events`)
- **Channels**: APNs (iOS push), FCM (Android push), in-app notification feed
- **Batching**: Multiple likes on the same post → collapse into "Alice and 42 others liked your photo" (not 43 separate pushes)
- **Notification feed**: Stored in Cassandra (partition key = user_id, ordered by timestamp) — the "Activity" tab
- **Rate limiting**: Max 1 push per post per 5-minute window for the same event type

#### Search Service (Elasticsearch)
- **Indexed entities**: Users (username, full name, bio), Hashtags (#sunset → post count, trending score), Locations (place name, coordinates)
- **Features**: Prefix autocomplete on usernames, fuzzy matching, trending hashtags
- **Ranking**: For user search — verified badge boost + follower count + mutual followers. For hashtag search — post count + trending velocity
- **Sync**: User profile changes → Kafka CDC → Elasticsearch consumer updates index
- **Note**: Post content (captions) are NOT full-text searchable (Instagram design choice — discovery is via hashtags and Explore, not text search)

#### Analytics Pipeline
- Consumes all Kafka event streams → Flink aggregation → ClickHouse
- **Metrics**: Post engagement rates, story completion rates, follower growth, creator analytics dashboard
- **Used by**: Explore ranking model training, content moderation signal enrichment, ad targeting

---

## 5. APIs

### Upload Photo
```http
POST /api/v1/posts
{
  "media_ids": ["media-uuid-1"],
  "caption": "Beautiful sunset! #nature",
  "location": {"lat": 37.7749, "lng": -122.4194, "name": "San Francisco"},
  "tagged_users": ["user-456"],
  "filter": "clarendon"
}
Response: 201 Created
```

### Get Pre-signed Upload URL
```http
GET /api/v1/media/upload-url?type=image&content_type=image/jpeg
Response: 200 OK
{
  "upload_url": "https://s3.amazonaws.com/instagram-media/...",
  "media_id": "media-uuid-1"
}
```

### Get Feed
```http
GET /api/v1/feed?cursor={post_id}&limit=10
```

### Get Stories
```http
GET /api/v1/stories/feed
Response: 200 OK
{
  "story_trays": [
    {
      "user": {"user_id": "...", "username": "...", "avatar_url": "..."},
      "stories": [
        {"story_id": "...", "media_url": "...", "created_at": "...", "expires_at": "..."}
      ]
    }
  ]
}
```

### Post Story
```http
POST /api/v1/stories
{
  "media_id": "media-uuid",
  "stickers": [...],
  "music_id": "..."
}
```

### Like / Comment
```http
POST /api/v1/posts/{post_id}/like
POST /api/v1/posts/{post_id}/comments
{ "text": "Amazing photo!" }
```

---

## 6. Data Model

### Cassandra — Posts

```sql
CREATE TABLE posts (
    post_id         BIGINT,         -- Snowflake ID
    user_id         UUID,
    caption         TEXT,
    media_urls      LIST<TEXT>,     -- CDN URLs for different sizes
    location        TEXT,
    hashtags        SET<TEXT>,
    tagged_users    SET<UUID>,
    like_count      COUNTER,
    comment_count   COUNTER,
    created_at      TIMESTAMP,
    PRIMARY KEY (post_id)
);

-- User's own posts (profile grid)
CREATE TABLE user_posts (
    user_id     UUID,
    post_id     BIGINT,
    media_thumb TEXT,              -- thumbnail URL for grid
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);
```

### Cassandra — Stories (with TTL)

```sql
CREATE TABLE stories (
    user_id     UUID,
    story_id    BIGINT,
    media_url   TEXT,
    media_type  VARCHAR,          -- image, video
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, story_id)
) WITH CLUSTERING ORDER BY (story_id DESC)
  AND default_time_to_live = 86400;   -- Auto-delete after 24 hours
```

### Redis — Feed Cache

```
Key:    feed:{user_id}
Type:   Sorted Set
Members: post_id
Scores:  ranking_score (not just timestamp — ML-ranked)
Max:     500 entries
```

### Redis — Story Tray

```
Key:    story_tray:{user_id}
Type:   Sorted Set
Members: poster_user_id
Scores:  latest_story_timestamp
TTL:     3600 (refresh hourly)
```

### MySQL — Users & Social Graph

```sql
CREATE TABLE users (
    user_id       UUID PRIMARY KEY,
    username      VARCHAR(30) UNIQUE,
    display_name  VARCHAR(64),
    bio           TEXT,
    avatar_url    TEXT,
    post_count    INT DEFAULT 0,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    is_private    BOOLEAN DEFAULT FALSE,
    created_at    TIMESTAMP
);

CREATE TABLE follows (
    follower_id  UUID,
    followee_id  UUID,
    status       ENUM('active', 'pending'),  -- pending for private accounts
    created_at   TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);
```

### S3 — Media Storage Structure

```
Bucket: instagram-media
Path:   /{user_id}/{year}/{month}/{media_id}/
Files:  original.jpg
        thumb_150.jpg
        small_320.jpg
        medium_640.jpg
        large_1080.jpg
        blurhash.txt
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Media loss** | S3 with 11 nines durability, cross-region replication |
| **Upload failure** | Client retries with idempotent media_id; S3 multipart upload for large files |
| **Feed cache loss** | Rebuild from DB; degrade to reverse-chronological (no ranking) |
| **Story expiration accuracy** | Cassandra TTL handles deletion; client also checks `expires_at` |
| **CDN cache invalidation** | On post delete, purge CDN cache by URL pattern |
| **Media processing failure** | DLQ for failed processing jobs; retry with different worker |

### Specific: Handling Image Upload Failures
1. Client uploads to S3 using multipart upload (resumable)
2. If upload fails mid-way, client retries from last successful part
3. If post metadata is saved but media processing fails → post is in "processing" state
4. Background retry processes failed media
5. After 3 failures → notify user to re-upload

---

## 8. Additional Considerations

### Progressive Image Loading
1. First: Show blurhash placeholder (tiny hash → instant blur preview)
2. Then: Load thumbnail (150px)
3. Then: Load appropriate resolution based on device/viewport
4. Uses `srcset` and `sizes` attributes for responsive images

### Content Moderation Pipeline
```
Upload → NSFW Detection (ML) → Hate Speech (NLP) → 
  → Score > threshold → Auto-reject + notify user
  → Score borderline → Queue for human review
  → Score OK → Publish
```

### Hashtag and Location Pages
- Hashtag page: Elasticsearch query for posts with specific hashtag
- Location page: Geospatial query (PostGIS or Elasticsearch geo_point)
- Both sorted by recency or "top" (engagement-based)

### Private Accounts
- Follow requires approval (status = 'pending')
- Posts only visible to approved followers
- Feed fan-out only to approved followers
- Explore page excludes private account posts

### Instagram Reels (Video Feed)
- Separate vertical video feed (like TikTok)
- Video transcoded to HLS/DASH adaptive streaming
- Recommendation engine: engagement-based + content understanding (video embeddings)
- Pre-fetch next 3 reels for smooth scrolling experience

### Feed Ranking Model — ML Deep Dive

```
Instagram's feed is NOT chronological. It's ranked by predicted engagement.

Features fed to the ranking model:
  User-author affinity:
    - interaction_score: how often user likes/comments author's posts (decay over time)
    - profile_visit_count: how often user visits author's profile
    - DM frequency: users who DM each other see each other's posts first
  
  Post features:
    - age_minutes: exponential decay (post from 1 hr ago >> post from 24 hrs ago)
    - content_type: photo / video / carousel (video gets ~1.3x implicit boost)
    - engagement_velocity: likes_in_first_30_min / impressions_in_first_30_min
    - caption_length, hashtag_count, has_location
  
  Context features:
    - time_of_day (user's local time), day_of_week
    - session_number_today: 1st open = best content; 5th open = deeper inventory
    - network_quality: on slow network, deprioritize video

Model: Multi-task learning — predict P(like), P(comment), P(save), P(share), P(dwell_time > 3s)
  Final score = w1*P(like) + w2*P(comment) + w3*P(save) + w4*P(share) + w5*P(dwell)
  Weights: save and share are weighted highest (stronger engagement signals than likes)

Serving: candidate generation (500 posts from fan-out cache) → ML ranking → top 50 served
Latency budget: < 100 ms for scoring 500 candidates (batched inference, ONNX on CPU)
Diversity injection: after ranking, ensure no 3 consecutive posts from same author
```

### Race Condition: Post Visible Before Media Processed

```
Problem: User creates post → metadata saved to DB → fan-out starts → followers see post
  But media is still processing (resize, filter, moderation). Followers see broken image.

Solution: Two-phase publishing
  Phase 1: Upload media + process (resize, moderate). Post status = "processing"
  Phase 2: Only after ALL media variants ready → set status = "published" → start fan-out
  
  Client shows spinner until post is "published" (typically 3-8 seconds).
  Fan-out never triggers for "processing" posts → followers never see broken images.
```

