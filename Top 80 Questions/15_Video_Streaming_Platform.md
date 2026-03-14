# 15. Design a Video Streaming Platform (YouTube / Netflix)

---

## 1. Functional Requirements (FR)

- **Upload videos**: Users upload videos with title, description, tags, thumbnails
- **Stream videos**: Adaptive bitrate streaming (adjust quality based on bandwidth)
- **Search videos**: By title, description, tags, channel
- **Recommendations**: Personalized "what to watch next"
- **Channels/Subscriptions**: Subscribe to channels, receive updates
- **Engagement**: Like, dislike, comment, share
- **Watch history** and resume playback
- **Live streaming** (optional: YouTube Live)
- **Monetization**: Ads, premium subscriptions

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99%
- **Low Startup Latency**: Video starts playing within 2 seconds
- **Smooth Playback**: No buffering under normal network conditions
- **Scalability**: 2B+ monthly active users, 500 hours of video uploaded per minute
- **Durability**: Uploaded videos must never be lost
- **Global**: Low latency streaming worldwide (CDN)
- **Cost Efficient**: Video storage and bandwidth are the biggest costs

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 800M |
| Videos watched / day | 5B |
| Avg video duration | 5 min |
| Streaming bandwidth / video | 5 Mbps (avg bitrate) |
| Concurrent viewers | 100M |
| Peak bandwidth | 100M × 5 Mbps = **500 Tbps** |
| Videos uploaded / day | 720K (500 hrs/min × 60 min × 24) |
| Avg original video size | 500 MB |
| Upload storage / day | 720K × 500 MB = 360 TB |
| Transcoded versions (6 resolutions) | 360 TB × 3 = ~1 PB/day |
| Total storage (existing) | ~1 EB (exabyte) |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐                    ┌──────────────┐
│  Upload      │                    │  Viewer      │
│  Client      │                    │  Client      │
└──────┬───────┘                    └──────┬───────┘
       │                                   │
┌──────▼───────┐                    ┌──────▼───────┐
│  Upload      │                    │     CDN      │  ← Edge servers worldwide
│  Service     │                    │  (Akamai/    │
│              │                    │  CloudFront) │
└──────┬───────┘                    └──────┬───────┘
       │                                   │ (cache miss)
┌──────▼───────┐                    ┌──────▼───────┐
│  Object      │                    │  Origin      │
│  Store (S3)  │                    │  Storage     │
│  (Original)  │                    │  (Transcoded)│
└──────┬───────┘                    └──────────────┘
       │
┌──────▼─────────────────────────────────────┐
│         Video Processing Pipeline          │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────┐│
│  │ Trans- │ │Thumbnail│ │Content │ │ DRM  ││
│  │ coding │ │ Gen    │ │Moderat.│ │Encrypt││
│  └────────┘ └────────┘ └────────┘ └──────┘│
└──────┬─────────────────────────────────────┘
       │
┌──────▼───────┐     ┌──────────────┐     ┌──────────────┐
│  Video       │     │  Search      │     │ Recommend.   │
│  Metadata DB │     │  Service     │     │ Service      │
│  (MySQL)     │     │ (Elasticsrch)│     │ (ML)         │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Component Deep Dive

#### Upload Flow
1. Client requests a pre-signed upload URL from Upload Service
2. Client uploads video directly to S3 (chunked upload for large files)
3. Upload Service creates a metadata record (status = "processing")
4. Publishes `video-uploaded` event to Kafka
5. Video Processing Pipeline picks up the event

#### Video Processing Pipeline — The Most Complex Part

**Step 1: Transcoding (Most Expensive)**
- Convert original video to multiple resolutions and bitrates:
  ```
  Resolution    Bitrate     Use Case
  240p          400 Kbps    Slow mobile connections
  360p          800 Kbps    Standard mobile
  480p          1.5 Mbps    WiFi / tablet
  720p          3 Mbps      HD
  1080p         5 Mbps      Full HD
  4K            20 Mbps     4K displays
  ```
- Codec: **H.264** (broad compatibility) or **H.265/HEVC** (50% better compression, less support) or **AV1** (royalty-free, best compression)
- **Parallel processing**: Split video into segments (e.g., 10-second chunks) → transcode each segment in parallel on different machines → reassemble
- **DAG (Directed Acyclic Graph) processing**: Tasks have dependencies
  ```
  Original Video → Split → [Transcode 240p, Transcode 360p, ..., Transcode 4K]
                          → Generate Thumbnails
                          → Extract Audio → Transcode Audio (AAC 128k, 256k)
                          → Content Moderation (NSFW, violence)
                          → Subtitle Extraction (speech-to-text)
                 → Merge all → Package (HLS/DASH) → Upload to Origin Storage
  ```

**Step 2: Adaptive Bitrate Streaming Packaging**

**HLS (HTTP Live Streaming)** — Apple standard, most widely used:
```
Master Playlist (manifest.m3u8):
  #EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=426x240
  240p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
  360p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
  720p/playlist.m3u8
  
Quality Playlist (720p/playlist.m3u8):
  #EXTINF:10.0,
  segment_001.ts
  #EXTINF:10.0,
  segment_002.ts
```

- Client downloads the master playlist → selects quality based on available bandwidth
- If bandwidth drops mid-stream → client switches to lower quality seamlessly
- Each segment is a small file (2-10 seconds) → independently cacheable by CDN

**DASH (Dynamic Adaptive Streaming over HTTP)** — open standard, uses `.mpd` manifest and `.m4s` segments

**Step 3: DRM Encryption**
- **Widevine** (Google/Android), **FairPlay** (Apple), **PlayReady** (Microsoft)
- Each segment encrypted with AES-128
- License server provides decryption keys to authenticated clients

#### CDN (Content Delivery Network)
- **Purpose**: Serve video segments from edge servers closest to the user
- **Cache strategy**: 
  - Popular videos (top 20%) cached at edge → serve 80% of traffic
  - Less popular → fetch from origin, cache with shorter TTL
  - Long tail → direct from origin, no caching
- **Global deployment**: 200+ edge locations (PoPs) worldwide
- **Origin shield**: Intermediate cache layer between CDN edge and origin storage → reduces origin load
- **Push vs Pull CDN**:
  - Popular content: Push to edge servers proactively
  - Long tail: Pull on first request

#### Recommendation Service
- **Collaborative filtering**: "Users who watched X also watched Y"
- **Content-based**: Similar genre, director, actors, tags
- **Deep learning**: Video embeddings (analyze visual/audio content)
- **Features**: Watch history, watch duration (did they finish?), likes, search history
- **Serving**: Pre-compute recommendations offline (Spark) → cache in Redis → serve in < 50 ms

---

## 5. APIs

### Upload Video
```http
POST /api/v1/videos/upload-url
Response: 200 OK
{
  "upload_url": "https://s3.amazonaws.com/uploads/...",
  "video_id": "video-uuid"
}

POST /api/v1/videos/{video_id}/metadata
{
  "title": "System Design in 10 Minutes",
  "description": "...",
  "tags": ["system design", "tutorial"],
  "category": "education",
  "visibility": "public"
}
```

### Stream Video
```http
GET /api/v1/videos/{video_id}/manifest
Response: 200 OK (redirects to CDN)
{
  "manifest_url": "https://cdn.example.com/videos/{video_id}/manifest.m3u8",
  "thumbnail_url": "https://cdn.example.com/videos/{video_id}/thumb.jpg"
}
```

### Search
```http
GET /api/v1/search?q=system+design&type=video&sort=relevance
```

### Get Recommendations
```http
GET /api/v1/recommendations?limit=20
```

---

## 6. Data Model

### MySQL — Video Metadata (Sharded by video_id)

```sql
CREATE TABLE videos (
    video_id        BIGINT PRIMARY KEY,
    channel_id      BIGINT NOT NULL,
    title           VARCHAR(100),
    description     TEXT,
    tags            JSON,
    category        VARCHAR(50),
    duration_sec    INT,
    status          ENUM('processing', 'ready', 'failed', 'removed'),
    visibility      ENUM('public', 'unlisted', 'private'),
    view_count      BIGINT DEFAULT 0,
    like_count      INT DEFAULT 0,
    dislike_count   INT DEFAULT 0,
    manifest_url    TEXT,
    thumbnail_url   TEXT,
    upload_date     TIMESTAMP,
    INDEX idx_channel (channel_id, upload_date DESC)
);
```

### S3 — Video Storage Structure

```
Bucket: video-originals
  /{video_id}/original.mp4

Bucket: video-transcoded
  /{video_id}/manifest.m3u8
  /{video_id}/240p/playlist.m3u8
  /{video_id}/240p/segment_001.ts
  /{video_id}/240p/segment_002.ts
  /{video_id}/720p/playlist.m3u8
  /{video_id}/720p/segment_001.ts
  /{video_id}/thumbnails/thumb_001.jpg
  /{video_id}/subtitles/en.vtt
```

### Cassandra — View Events (for analytics)

```sql
CREATE TABLE view_events (
    video_id        BIGINT,
    view_date       DATE,
    view_hour       INT,
    user_id         UUID,
    watch_duration  INT,
    quality         VARCHAR,
    device          VARCHAR,
    country         VARCHAR,
    PRIMARY KEY ((video_id, view_date), view_hour, user_id)
);
```

### Redis — View Counters + Hot Video Cache

```
Key:    views:{video_id}
Value:  counter (INCR)

Key:    video:meta:{video_id}
Value:  Hash { title, channel, manifest_url, thumbnail_url }
TTL:    3600
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Upload failure** | S3 multipart upload (resumable); client retries from last chunk |
| **Transcoding failure** | Retry failed segments; DLQ for persistent failures |
| **CDN edge failure** | CDN automatically routes to next closest PoP |
| **Origin failure** | S3 cross-region replication; CDN caches absorb the load |
| **Video corruption** | Checksum verification at each stage; re-transcode from original |
| **Popularity surge** | CDN pre-warming for predicted viral content; auto-scale origin |

### Specific: Handling a Viral Video
1. Video starts getting 10M views/minute
2. CDN edge caches fill up → 90% of requests served from edge
3. View counter: Don't write to DB for every view. Batch in memory → flush every 5 seconds
4. Comment section: Rate limit comments per user; paginate aggressively

---

## 8. Additional Considerations

### Video Segment Prefetching
- Client prefetches next 2-3 segments while playing current segment
- If user seeks to a new position → cancel prefetch, start buffering from seek point
- Adaptive: if bandwidth is high, prefetch more; if low, prefetch less

### Thumbnail Generation
- Extract frames at 10% intervals → pick the most "interesting" frame (highest entropy, face detection)
- Or: generate "video preview" (6-second animated summary) shown on hover

### Copyright / Content ID
- **Audio fingerprinting**: Match audio against copyrighted music database
- **Video fingerprinting**: Perceptual hashing to detect re-uploads of copyrighted content
- **Action**: Block upload, mute audio, add ads (revenue to copyright holder), or allow with claim

### Cost Optimization
- **Storage tiering**: Frequently accessed videos on SSD-backed S3; rarely viewed videos on S3 Glacier
- **Encoding optimization**: Only transcode to 4K if original is 4K; don't upscale
- **CDN cost**: Negotiate bandwidth tiers; use multiple CDN providers (multi-CDN) for cost and resilience
- **Keep original**: Always keep the original file — codecs improve, and you can re-transcode later

### Live Streaming Architecture
- **Ingest**: RTMP from broadcaster → transcoding server
- **Real-time transcoding**: Must be fast (< 1 second per segment)
- **Delivery**: HLS/DASH with very short segments (2 seconds) for low latency
- **Glass-to-glass latency**: Target < 5 seconds (standard HLS is 15-30s; use LL-HLS for < 3s)
- **DVR**: Store live segments for rewind/replay

---

## 9. Deep Dive: Engineering Trade-offs

### HLS vs DASH vs WebRTC: Choosing the Streaming Protocol

| Protocol | How | Latency | Browser Support | DRM | Best For |
|---|---|---|---|---|---|
| **HLS** ⭐ | Apple's protocol; `.m3u8` manifest + `.ts` segments | 15-30s (standard), 2-5s (LL-HLS) | All browsers + native iOS/Android | FairPlay (Apple) | VOD, live streaming (widest compatibility) |
| **DASH** | Open standard; `.mpd` manifest + `.m4s` segments | 15-30s (standard), 2-5s (LL-DASH) | All except iOS Safari natively | Widevine, PlayReady | VOD, live (open ecosystem) |
| **WebRTC** | Peer-to-peer real-time | < 500ms | All modern browsers | None (not designed for it) | Video calling, ultra-low latency live |
| **CMAF** | Common Media Application Format (unified HLS+DASH) | 2-5s | Broad | All major | Future standard unifying HLS and DASH |

**Why YouTube/Netflix choose HLS + DASH (not WebRTC)**:
1. WebRTC is peer-to-peer — doesn't scale to millions of simultaneous viewers
2. WebRTC has no DRM — can't protect copyrighted content
3. HLS/DASH work over standard HTTP → leverages existing CDN infrastructure
4. 2-5 second latency is acceptable for VOD and most live streaming

**When WebRTC is better**: Real-time interactive (video calls, live auction bidding, interactive gaming streams)

### Codec Selection: H.264 vs H.265 vs AV1

| Codec | Compression | Encoding Cost | Decoding (Device Support) | License |
|---|---|---|---|---|
| **H.264 (AVC)** | Baseline | 1× | Universal (100% of devices) | Licensed (but essentially free for web) |
| **H.265 (HEVC)** | 50% better than H.264 | 5-10× slower | Good (most modern devices, patchy browser support) | Expensive licensing (patent pools) |
| **VP9** | ~Same as H.265 | 5-10× | Good (Chrome, Android, no Safari) | Royalty-free (Google) |
| **AV1** ⭐ | 30% better than H.265 | 50-100× slower! | Growing (modern devices/browsers) | Royalty-free (Alliance for Open Media) |

**Netflix's actual approach**: Encode in multiple codecs per resolution. Serve the best codec the client supports.
```
Client supports AV1?  → Serve AV1 (smallest files, cheapest bandwidth)
Client supports VP9?  → Serve VP9
Fallback?             → Serve H.264 (universal)
```

**The AV1 trade-off**: 30% bandwidth savings = massive cost reduction at Netflix scale (petabytes/day). But encoding is 50-100× slower than H.264 → requires GPU-accelerated hardware encoders or pre-encode everything offline.

### Pre-Signed Upload URL: Why Upload Directly to S3?

```
Naive approach: Client → API Server → S3
  ✗ API server becomes bottleneck (proxying gigabytes of video)
  ✗ Double bandwidth cost (ingest + upload)
  ✗ API server needs massive storage/memory buffers

Direct upload: Client → Pre-signed S3 URL → S3 directly
  1. Client calls API: POST /upload/init → returns pre-signed S3 URL
  2. Client uploads directly to S3 (multipart upload for large files)
  3. S3 triggers event (Lambda/SNS) → starts processing pipeline
  
  ✓ API server handles only metadata (tiny)
  ✓ S3 handles the heavy lifting (built for this)
  ✓ Resumable (multipart upload — restart from last chunk)
  ✓ Globally distributed (S3 Transfer Acceleration)
```

### View Count: Why Not Just INCREMENT a Database Counter?

```
Naive: Each view → UPDATE videos SET view_count = view_count + 1
  At 1M concurrent viewers → 1M write transactions/sec on a single row
  → Database lock contention → CRUSH the DB

Actual approach (YouTube):
  1. Client sends "view" event → collected in memory buffer on API server
  2. Every 5 seconds: batch flush to Kafka topic "view-events"
  3. Kafka → Flink → aggregate per video per minute → write to Cassandra/ClickHouse
  4. For display: Redis counter (INCR, batch flushed) for approximate real-time count
  5. Exact counts reconciled hourly via batch job
  
  Why this works:
  - Kafka absorbs the write burst (append-only log, 1M msgs/sec easy)
  - Flink aggregates → reduces 1M events/sec to thousands of writes/sec
  - Redis INCR is atomic and fast for real-time display
  - View counts don't need to be exact in real-time (±0.1% error is fine)
```

### Storage Tiering: The 80/20 Rule for Video

```
Observation: 20% of videos account for 80% of views (power law)

Tier 1: Hot Storage (SSD-backed S3, CDN-cached)
  → Top 20% most-viewed videos
  → All videos uploaded in last 7 days
  → ~20% of total storage, serves ~80% of traffic
  → Cost: $$$

Tier 2: Warm Storage (Standard S3)
  → Videos with > 10 views/month
  → ~30% of storage
  → Cost: $$

Tier 3: Cold Storage (S3 Glacier / Deep Archive)
  → Videos with < 10 views/month and older than 1 year
  → ~50% of storage, serves < 1% of traffic
  → Cost: $
  → Access: 1-12 hour retrieval time (acceptable for rarely viewed content)

Auto-tiering:
  - Monitor access patterns per video
  - Move cold → warm if sudden interest (viral resurgence)
  - Move warm → cold after 90 days of low activity
  
At Netflix scale: This saves hundreds of millions of dollars per year in storage costs
```

### Why MySQL for Video Metadata (Not MongoDB/Cassandra)?

```
Video metadata access patterns:
  1. Lookup by video_id (point query)           → Any DB works
  2. Search by title/description                → Elasticsearch (not primary DB)  
  3. List videos by channel, sorted by date     → B-tree index on (channel_id, upload_date)
  4. Update view_count, like_count              → Atomic counter updates
  5. Complex admin queries (revenue reports)    → JOINs, aggregations
  
MySQL ✓:
  - #3 and #5 need relational queries → MySQL excels
  - Video metadata is small (< 1KB per video) even at 1B videos → 1TB total (fits)
  - Sharding by video_id for horizontal scaling
  - Read replicas for search/browse queries

Cassandra ✗:
  - No JOINs → complex admin queries become application-level
  - Counter columns have known issues (eventual consistency, not idempotent)
  
MongoDB:
  - Could work (flexible schema for varying video attributes)
  - But: no native JOIN support; admin reporting is weaker
  - Use it if schema varies significantly across video types
```
