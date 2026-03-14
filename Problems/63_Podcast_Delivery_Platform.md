# 63. Design a Podcast Delivery Platform

---

## 1. Functional Requirements (FR)

- **Upload podcasts**: Creators upload audio episodes with metadata (title, description, show notes, chapters)
- **Streaming playback**: Stream episodes with seeking, speed control (0.5×–3×), skip silence
- **Downloading**: Offline download for listening without internet
- **RSS feed**: Generate and serve RSS/Atom feeds for distribution to Apple Podcasts, Spotify, etc.
- **Show management**: Create shows (series), manage episodes, schedule future releases
- **Discovery**: Browse by category, charts (top podcasts), search, recommendations
- **Subscriptions**: Users subscribe to shows; new episodes appear in their feed
- **Playback state**: Sync playback position across devices (resume where you left off)
- **Analytics**: Download counts, listener demographics, retention graphs per episode
- **Monetization**: Dynamic ad insertion (pre-roll, mid-roll, post-roll), premium subscriptions

---

## 2. Non-Functional Requirements (NFRs)

- **Playback Start**: Audio begins playing within 2 seconds
- **Availability**: 99.99% — listeners can always play episodes
- **Durability**: Audio files never lost
- **Scalability**: 5M+ podcast shows, 100M+ episodes, 100M+ active listeners/month
- **Global**: Low-latency playback worldwide via CDN
- **Bandwidth Efficiency**: Adaptive bitrate; compressed audio formats (Opus, AAC)
- **Analytics Accuracy**: Download/listen counts accurate within ±1%
- **RSS Compliance**: Valid RSS 2.0 with iTunes podcast extensions

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total shows | 5M |
| Total episodes | 100M |
| New episodes / day | 100K |
| Avg episode duration | 45 minutes |
| Avg episode file size | 50 MB (128 kbps MP3) |
| Upload storage / day | 100K × 50 MB = 5 TB |
| Total storage | 100M × 50 MB = 5 PB |
| Daily active listeners | 30M |
| Concurrent streams | 5M |
| Stream bandwidth | 5M × 128 kbps = 640 Gbps |
| Downloads / day | 500M (including RSS aggregators) |
| Download bandwidth | 500M × 50 MB = 25 PB / day |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐
│  Podcast     │    │  Listener    │
│  Creator     │    │  App         │
│  Dashboard   │    │              │
└──────┬───────┘    └──────┬───────┘
       │                    │
┌──────▼───────┐    ┌──────▼───────┐
│  Creator     │    │   CDN        │ ← Audio served from edge
│  Service     │    │  (CloudFront │
│  (Upload,    │    │   / Akamai)  │
│   Manage)    │    └──────┬───────┘
└──────┬───────┘           │ (cache miss)
       │            ┌──────▼───────┐
       │            │  Audio       │
       │            │  Origin      │
       │            │  (S3)        │
       │            └──────────────┘
       │
┌──────▼─────────────────────────────────────────┐
│                Core Services                    │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Catalog  │  │ Feed     │  │ Subscription │ │
│  │ Service  │  │ Service  │  │ Service      │ │
│  │ (Shows,  │  │ (RSS     │  │ (User subs,  │ │
│  │ Episodes)│  │  gen +   │  │  new episode │ │
│  │          │  │  serve)  │  │  delivery)   │ │
│  └──────────┘  └──────────┘  └──────────────┘ │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Playback │  │ Search   │  │ Analytics    │ │
│  │ Sync     │  │ Service  │  │ Service      │ │
│  │ Service  │  │ (Elastic │  │ (ClickHouse) │ │
│  │ (position│  │  search) │  │              │ │
│  │  sync)   │  │          │  │              │ │
│  └──────────┘  └──────────┘  └──────────────┘ │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  Ad Insertion Service (VAST/VMAP)        │  │
│  │  Dynamic ad stitching per listener       │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘

Data Stores:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │ Elasticsearch│  │  ClickHouse  │
│ (Shows, Eps, │  │ (Playback   │  │ (Search)     │  │ (Analytics)  │
│  Users, Subs)│  │  state,     │  │              │  │              │
│              │  │  feed cache, │  │              │  │              │
│              │  │  charts)    │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Audio Processing Pipeline

```
Creator uploads raw audio file (WAV, FLAC, MP3, M4A, OGG):

Step 1: Validate + Probe
  FFprobe: extract duration, sample rate, channels, bitrate, codec
  Validate: audio only (no video), duration < 12 hours, file < 2 GB

Step 2: Normalize Audio
  - Loudness normalization: target -16 LUFS (loudness standard for podcasts)
    ffmpeg -i input.wav -af "loudnorm=I=-16:TP=-1.5:LRA=11" normalized.wav
    Why: different creators record at different volumes → consistent listening experience
  
  - Silence trimming: remove > 3 seconds of silence at start/end
  - Noise reduction (optional): spectral subtraction for background noise

Step 3: Transcode to Multiple Formats/Bitrates
  
  Format     Bitrate    Use Case              File Size (1hr)
  MP3        128 kbps   Universal (RSS)       57 MB
  MP3         64 kbps   Low bandwidth         29 MB
  AAC        128 kbps   iOS/Android native    57 MB (better quality than MP3)
  AAC         64 kbps   Mobile data saver     29 MB
  Opus        48 kbps   Best quality/size     22 MB ⭐ (50% smaller than MP3!)
  
  Why multiple formats:
  - MP3 128k: RSS feeds, compatibility (every device plays MP3)
  - AAC 128k: native mobile apps (better quality at same bitrate)
  - Opus 48k: modern apps (incredible compression, 50% less bandwidth)
  - MP3 64k: data-saver mode, emerging markets

Step 4: Chapter Markers
  If creator provides chapter markers (or we detect them from show notes):
  Embed in MP3 ID3 tags / M4A chapters
  
  Chapters enable: "Skip to Chapter 3: Interview with guest"
  Format: { title, start_time, end_time, optional_url, optional_image }

Step 5: Generate Waveform
  Visualize audio waveform for seek bar (like SoundCloud)
  Compute: RMS amplitude per 100ms window → JSON array
  Client renders waveform from data (no image needed)
  
  waveform = [0.1, 0.3, 0.5, 0.8, 0.7, 0.4, ...]  // 600 data points per minute
  File: ~50 KB JSON per episode (compressed)

Step 6: Speech-to-Text Transcription
  Generate transcript for:
    - Search indexing (find episodes by spoken words)
    - Accessibility (hearing impaired)
    - Show notes auto-generation
  
  Tool: Whisper (OpenAI) or cloud STT (Google, AWS Transcribe)
  Cost: ~$0.006 per minute of audio
  45-min episode: $0.27 per episode × 100K episodes/day = $27K/day
  
  Optimization: Only transcribe for shows with > 100 subscribers (covers 90% of listeners)
```

#### RSS Feed Service — The Core Distribution Mechanism

```
Podcasts are distributed via RSS. Every podcast app (Apple Podcasts, Spotify,
Overcast, Pocket Casts) polls RSS feeds to discover new episodes.

RSS feed structure:
  <?xml version="1.0" encoding="UTF-8"?>
  <rss version="2.0" xmlns:itunes="..." xmlns:podcast="...">
    <channel>
      <title>My Podcast Show</title>
      <link>https://example.com/shows/my-podcast</link>
      <itunes:author>John Doe</itunes:author>
      <itunes:category text="Technology"/>
      <itunes:image href="https://cdn.example.com/art/show-123.jpg"/>
      
      <item>
        <title>Episode 42: System Design</title>
        <enclosure url="https://cdn.example.com/audio/ep-42.mp3" 
                   length="57000000" type="audio/mpeg"/>
        <pubDate>Fri, 14 Mar 2025 08:00:00 GMT</pubDate>
        <itunes:duration>3600</itunes:duration>
        <description>In this episode we discuss...</description>
        <podcast:chapters url="https://cdn.example.com/chapters/ep-42.json"/>
      </item>
      ...
    </channel>
  </rss>

Feed serving at scale:
  5M shows × polled every 15-30 minutes by 10+ aggregators = ~30M feed requests/hour
  
  Strategy:
  1. Pre-generate RSS XML for each show → store in S3
  2. Serve via CDN with 15-minute TTL
  3. On new episode publish: regenerate feed XML → invalidate CDN cache
  4. Conditional requests: ETag/If-Modified-Since → 304 Not Modified (saves bandwidth)
  
  Feed size: ~10 KB per show (last 50 episodes)
  Total: 5M × 10 KB = 50 GB (easily fits in CDN)

RSS feed URL:
  https://feeds.example.com/shows/{show_id}/rss
  Stable URL — never changes (aggregators subscribe to this URL permanently)
```

#### Playback Sync — Resume Across Devices

```
User listens on phone during commute → switches to laptop at desk → resumes from same point

Sync mechanism:
  Client reports playback position every 30 seconds:
    POST /api/v1/playback/progress
    { episode_id, position_seconds, speed, timestamp }
  
  On opening an episode:
    GET /api/v1/playback/progress/{episode_id}
    Response: { position_seconds: 1847, speed: 1.5, updated_at: "..." }
    Player seeks to 1847 seconds → user resumes seamlessly

Storage: Redis ⭐ (fast, simple, TTL-based cleanup)
  Key: playback:{user_id}:{episode_id}
  Value: Hash { position, speed, duration, updated_at }
  TTL: 90 days (auto-cleanup old progress)

Scale: 30M DAU × ~3 episodes/day × update every 30 sec = ~100K writes/sec to Redis
  Easily handled by Redis Cluster (10 shards)

Conflict resolution (rare but possible):
  User plays on phone AND laptop simultaneously
  → Last-write-wins (most recent timestamp)
  → Acceptable: user explicitly resumed on one device

Queue state:
  In addition to per-episode progress, sync the episode queue:
  Key: queue:{user_id}
  Value: ordered list of episode_ids (next up, in progress)
  Sync on app open → update queue from server
```

#### Dynamic Ad Insertion (DAI) — The Revenue Engine

```
Instead of baking ads into audio (static), insert ads at playback time (dynamic):

Benefits:
  - Different listeners hear different ads (targeted)
  - Update/replace ads without re-uploading episode
  - Track ad impressions per listener (better analytics)
  - Fill unsold ad slots with house ads or silence removal

Implementation:

Server-Side Ad Insertion (SSAI) ⭐:
  1. Creator marks ad break points in episode:
     { "breaks": [{"position": 0, "type": "pre-roll"}, 
                   {"position": 1200, "type": "mid-roll"}, 
                   {"position": 3500, "type": "post-roll"}] }
  
  2. On playback request:
     a. Ad Decision Service selects ads based on:
        - Listener demographics (age, location, interests)
        - Show category
        - Time of day
        - Ad campaign targeting rules
        - Frequency cap (don't show same ad twice in 24 hours)
     
     b. Audio Stitching Service:
        - Read original audio file from S3
        - At each break point: splice in selected ad audio
        - Output: continuous audio stream with ads inserted
        - Client is unaware ads are dynamically inserted
  
  3. Implementation options:
     a. Real-time stitching (on each request):
        ✓ Every listener gets personalized ads
        ✗ CPU-intensive; requires audio processing per request
        
     b. Segmented approach (HLS-like for audio):
        Split episode into segments (before ad, ad, after ad)
        Generate playlist: [segment_pre, ad_1, segment_mid, ad_2, segment_post]
        Client plays segments in order → seamless playback
        ✓ Pre-encoded segments → no real-time processing
        ✓ CDN-cacheable segments (ads cached separately from content)
        ✗ Seek across ad boundaries requires playlist management

Ad Tracking:
  Client reports: { "ad_id": "ad-123", "event": "impression|start|25%|50%|75%|complete" }
  → Kafka → ClickHouse → Ad analytics dashboard
  
  Attribution: listener heard ad for Product X → did they visit Product X's website?
  Tracking pixel / promo code in ad → measure conversion
```

---

## 5. APIs

### Upload Episode
```http
POST /api/v1/shows/{show_id}/episodes
{
  "title": "Episode 42: System Design",
  "description": "In this episode...",
  "audio_file_key": "uploads/ep-42-raw.wav",
  "publish_at": "2025-03-14T08:00:00Z",
  "season": 3,
  "episode_number": 42,
  "explicit": false,
  "chapters": [
    {"title": "Introduction", "start": 0},
    {"title": "Main Topic", "start": 180},
    {"title": "Interview", "start": 1200}
  ],
  "ad_breaks": [
    {"position": 0, "type": "pre_roll", "max_duration": 30},
    {"position": 1200, "type": "mid_roll", "max_duration": 60}
  ]
}
Response: 201 Created
{
  "episode_id": "ep-uuid",
  "status": "processing",
  "estimated_ready": "2025-03-14T07:55:00Z"
}
```

### Stream Episode
```http
GET /api/v1/episodes/{episode_id}/stream?format=aac&quality=128k
Response: 302 Redirect
  Location: https://cdn.example.com/audio/ep-uuid/aac_128k.m4a
  
  Or with ad insertion:
  Location: https://cdn.example.com/dai/ep-uuid/playlist.m3u8
```

### Get User's Feed (New Episodes from Subscriptions)
```http
GET /api/v1/feed?limit=20&cursor={last}
Response: 200 OK
{
  "episodes": [
    {
      "episode_id": "ep-uuid",
      "show": {"id": "show-uuid", "title": "Tech Talk", "art": "..."},
      "title": "Episode 42: System Design",
      "duration_seconds": 2700,
      "published_at": "2025-03-14T08:00:00Z",
      "progress": {"position": 1847, "percent": 68},
      "stream_url": "https://cdn.example.com/audio/ep-uuid/aac_128k.m4a",
      "download_url": "https://cdn.example.com/audio/ep-uuid/mp3_128k.mp3",
      "file_size_bytes": 32400000
    }
  ]
}
```

### Update Playback Position
```http
POST /api/v1/playback/progress
{
  "episode_id": "ep-uuid",
  "position_seconds": 1847,
  "speed": 1.5,
  "duration_seconds": 2700
}
Response: 200 OK
```

### Get RSS Feed
```http
GET /feeds/{show_id}/rss
Response: 200 OK
Content-Type: application/rss+xml
(RSS XML with iTunes extensions)
```

---

## 6. Data Model

### PostgreSQL — Core Data

```sql
CREATE TABLE shows (
    show_id         UUID PRIMARY KEY,
    creator_id      UUID NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(50),
    subcategory     VARCHAR(50),
    language        CHAR(5),
    artwork_url     TEXT,
    website_url     TEXT,
    rss_feed_url    TEXT NOT NULL,          -- public feed URL (stable, permanent)
    explicit        BOOLEAN DEFAULT FALSE,
    subscriber_count INT DEFAULT 0,
    total_episodes  INT DEFAULT 0,
    status          ENUM('active', 'paused', 'archived') DEFAULT 'active',
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    INDEX idx_category (category, subscriber_count DESC),
    INDEX idx_creator (creator_id)
);

CREATE TABLE episodes (
    episode_id      UUID PRIMARY KEY,
    show_id         UUID NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    show_notes      TEXT,
    season          SMALLINT,
    episode_number  SMALLINT,
    duration_seconds INT,
    audio_url_mp3   TEXT,                  -- CDN URL for MP3
    audio_url_aac   TEXT,                  -- CDN URL for AAC
    audio_url_opus  TEXT,                  -- CDN URL for Opus
    original_s3_key TEXT,
    file_size_bytes INT,
    chapters        JSONB,
    ad_breaks       JSONB,
    transcript_url  TEXT,
    waveform_url    TEXT,
    explicit        BOOLEAN DEFAULT FALSE,
    status          ENUM('draft','processing','scheduled','published','archived'),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMP,
    INDEX idx_show (show_id, published_at DESC),
    INDEX idx_published (status, published_at DESC)
);

CREATE TABLE subscriptions (
    user_id         UUID NOT NULL,
    show_id         UUID NOT NULL,
    subscribed_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notifications   BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (user_id, show_id),
    INDEX idx_show (show_id)               -- "who subscribes to this show"
);
```

### Redis — Playback + Caching

```
# Playback progress
playback:{user_id}:{episode_id}  → Hash { position, speed, updated_at }
TTL: 7776000 (90 days)

# User's episode queue
queue:{user_id}  → List of episode_ids (ordered: next up first)

# RSS feed cache
rss:{show_id}  → String (RSS XML blob)
TTL: 900 (15 minutes)

# Podcast charts (sorted sets)
charts:top:{category}    → Sorted Set { show_id: score }
charts:trending          → Sorted Set { show_id: growth_score }
charts:new_noteworthy    → Sorted Set { show_id: score }

# Episode download counter (batch flushed to ClickHouse)
downloads:{episode_id}:{date}  → INT (INCR)
TTL: 172800 (2 days, flushed daily to analytics DB)
```

### S3 — Audio Storage

```
Bucket: podcast-originals (cross-region replicated, permanent)
  /{show_id}/{episode_id}/original.wav

Bucket: podcast-processed (CDN-served)
  /{show_id}/{episode_id}/mp3_128k.mp3
  /{show_id}/{episode_id}/mp3_64k.mp3
  /{show_id}/{episode_id}/aac_128k.m4a
  /{show_id}/{episode_id}/aac_64k.m4a
  /{show_id}/{episode_id}/opus_48k.ogg
  /{show_id}/{episode_id}/waveform.json
  /{show_id}/{episode_id}/transcript.json
  /{show_id}/{episode_id}/chapters.json

Bucket: podcast-artwork
  /{show_id}/artwork_3000x3000.jpg
  /{show_id}/artwork_600x600.jpg
  /{show_id}/artwork_300x300.jpg
```

### Kafka Topics

```
Topic: episode-published     (trigger RSS regeneration, subscriber notifications)
Topic: playback-events       (play, pause, seek, complete — for analytics)
Topic: download-events       (download started — for analytics + ad tracking)
Topic: ad-events             (ad impression, start, complete — for ad analytics)
```

### ClickHouse — Analytics

```sql
CREATE TABLE episode_plays (
    episode_id      UUID,
    show_id         UUID,
    user_id         UUID,
    event_type      Enum8('play'=0,'pause'=1,'seek'=2,'complete'=3,'download'=4),
    position_seconds UInt32,
    duration_seconds UInt32,
    speed           Float32,
    platform        Enum8('ios'=0,'android'=1,'web'=2,'rss'=3),
    country         FixedString(2),
    city            String,
    event_date      Date MATERIALIZED toDate(timestamp),
    timestamp       DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (show_id, episode_id, timestamp);

-- "Retention curve: what % of listeners reach 25%, 50%, 75%, 100%?"
-- "Downloads by country for Episode 42"
-- "Average listen duration for my show"
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Audio file corruption** | Checksum verification after upload; re-upload from creator if corrupt |
| **CDN failure** | Multi-CDN (CloudFront + Akamai); DNS failover in < 30 seconds |
| **RSS feed stale** | Max TTL 15 min; manual cache purge on publish; ETag for conditional requests |
| **Playback sync loss** | Client buffers progress locally → retry sync when online |
| **Ad insertion failure** | Serve episode WITHOUT ads (degrade gracefully; better than no audio) |
| **Processing pipeline failure** | Retry 3× from Kafka; DLQ for persistent failures; alert creator |
| **Download counter loss** | Redis AOF + batch flush to ClickHouse every hour; ClickHouse is source of truth |

### Specific: RSS Polling Storm

```
Problem: Apple Podcasts polls all 5M feeds every 15-30 minutes
  Plus: Spotify, Google, Overcast, Pocket Casts, Castbox, ...
  Total: 5M feeds × 10 aggregators × 4 polls/hour = 200M requests/hour = 55K req/sec

All aggregators poll on the hour (synchronized peak):
  Minute 0: 200M requests hit feed service simultaneously → thundering herd!

Solutions:
  1. CDN caching (primary):
     RSS feeds served from CDN edge
     TTL: 15 minutes
     55K req/sec → 95% CDN hit rate → only 2,750 origin requests/sec
  
  2. Conditional requests:
     Aggregators send: If-None-Match: "etag-abc"
     If feed unchanged → 304 Not Modified (no body transferred)
     90% of polls → feed hasn't changed → 304 saves 90% bandwidth
  
  3. WebSub (PubSubHubbub):
     Instead of polling, push new episodes to aggregators in real-time
     Creator publishes episode → our server notifies aggregator's webhook
     Aggregator doesn't need to poll → eliminates 99% of feed requests
     Apple Podcasts + Spotify support WebSub
  
  4. Rate limiting per aggregator:
     Each aggregator gets max 10 req/sec
     Prevents one misbehaving bot from overwhelming feed service
```

### Specific: Download Counting Accuracy (IAB Standard)

```
Podcasting uses downloads as the primary metric (not streams, unlike music).
The IAB (Interactive Advertising Bureau) defines standards:

IAB Podcast Measurement Guidelines:
  - A "download" = one complete or substantial file transfer
  - Filter: bots, duplicate requests (same IP + User-Agent within 24 hours)
  - Filter: byte-range requests that don't cover substantial content
  - Filter: < 60 seconds between requests from same IP (pre-fetching)

Implementation:
  1. Log every download request:
     { episode_id, ip, user_agent, bytes_served, timestamp, country }
  
  2. Deduplication (in Flink stream processor):
     Key: SHA256(ip + user_agent + episode_id)
     Window: 24 hours
     If duplicate key seen → don't count
  
  3. Bot filtering:
     Maintain bot User-Agent list (IAB provides one)
     Filter: if UA matches bot → don't count
  
  4. Byte-range filtering:
     If request is for bytes 0-1000 (metadata only) → don't count
     If total bytes served < 50% of episode → don't count
  
  5. Store verified downloads in ClickHouse:
     Certified IAB-compliant download count = revenue basis for ads
     
  Accuracy matters: advertisers pay per 1000 downloads (CPM)
  Overcounting = fraud; undercounting = lost revenue
```

---

## 8. Deep Dive: Engineering Trade-offs

### Audio Codec Choice: MP3 vs AAC vs Opus

```
MP3 (MPEG Layer 3):
  ✓ Universal: every device, browser, car stereo plays MP3
  ✓ RSS standard (all podcast apps expect MP3 in enclosure tags)
  ✗ Old codec (1993): inferior compression to modern alternatives
  ✗ 128 kbps MP3 = audible artifacts on complex audio
  
  When to use: RSS feeds, maximum compatibility

AAC (Advanced Audio Coding):
  ✓ Better quality than MP3 at same bitrate (30% more efficient)
  ✓ Native support on iOS/Android/modern browsers
  ✗ Some older devices don't support
  ✗ Patent licensing (expired in many jurisdictions)
  
  When to use: Native mobile apps (guaranteed device support)

Opus ⭐:
  ✓ Best compression: 48 kbps Opus ≈ 128 kbps MP3 quality
  ✓ Royalty-free
  ✓ Excellent for speech (specifically designed for voice communication)
  ✓ Low latency (important for live podcast streaming)
  ✗ Limited support in podcast apps (growing but not universal)
  ✗ Not supported in RSS enclosure standard (yet)
  
  When to use: Native apps where you control the player; data-saver mode

Strategy:
  Store: MP3 128k (RSS/universal) + AAC 128k (quality) + Opus 48k (bandwidth)
  Serve: best format client supports
  RSS feed: always references MP3 (compatibility)
  Native app: negotiates best format (Opus → AAC → MP3 fallback)
```

### Silence Detection and Skip: More Complex Than It Sounds

```
Feature: "Skip silence" — automatically skip parts with no speech

Detection:
  Analyze audio in 50ms windows
  Compute RMS (root mean square) energy per window
  If RMS < threshold for > 500ms → "silence"
  
  Threshold: depends on recording environment
    Studio recording: -40 dBFS is silence
    Noisy recording: -30 dBFS might have ambient noise as "silence"
  
  Adaptive threshold:
    Compute noise floor from first 2 seconds of episode (usually quiet)
    Set threshold = noise_floor + 6 dB

Two implementation approaches:

1. Client-side skip:
   Provide silence map to client: [(start1, end1), (start2, end2), ...]
   Client skips these segments during playback
   ✓ No server processing per listen
   ✓ User controls sensitivity
   ✗ Can't actually remove silence from file (seeking still works)
   
   Silence map generation: async, during audio processing pipeline
   Store: JSON file alongside audio → client downloads once

2. Server-side processing:
   Remove silence segments from audio file → create "compact" version
   ✓ Smaller file → less bandwidth
   ✗ Need to store additional version
   ✗ Chapter markers/ad breaks need timestamp adjustment
   
   Result: 45-min podcast with "um"s and pauses → 38 minutes
   ~15% duration reduction typical for interview-style podcasts

Recommended: Client-side skip (flexible, no extra storage)
  With pre-computed silence map for instant silence detection
```

### Podcast Discovery: Charts and Recommendations

```
Charts (Top Podcasts):

Ranking algorithm:
  score = w1 × new_subscribers_7d 
        + w2 × downloads_7d 
        + w3 × listener_retention 
        + w4 × review_score 
        + w5 × velocity (growth rate)

  velocity = (downloads_this_week - downloads_last_week) / downloads_last_week
  
  Weight velocity highly → "trending" shows appear on charts
  This is similar to how Apple Podcasts and Spotify rank shows

Computation:
  Daily Spark batch job:
    SELECT show_id, 
           COUNT(*) as downloads_7d,
           COUNT(DISTINCT user_id) as subscribers
    FROM episode_plays
    WHERE timestamp > now() - INTERVAL 7 DAY
    GROUP BY show_id
  
  Compute scores → store in Redis sorted set
  charts:top:{category} → Sorted Set { show_id: score }

Recommendations:
  Collaborative filtering:
    "Users who subscribe to Show A also subscribe to Show B"
    User-show interaction matrix → matrix factorization → similar shows
  
  Content-based:
    Show embeddings from: title, description, transcript text (NLP)
    Similar shows = cosine similarity of embeddings
  
  Hybrid serving:
    1. Get user's subscriptions
    2. For each subscription: find similar shows (pre-computed)
    3. Filter: already subscribed, recently dismissed
    4. Rank by similarity score × show quality score
    5. Cache per user in Redis (TTL: 24 hours)
```



