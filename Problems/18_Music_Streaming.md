# 18. Design a Music Streaming Service (Spotify)

---

## 1. Functional Requirements (FR)

- **Stream music**: Play songs on-demand with continuous playback
- **Search**: Search by song, artist, album, genre, lyrics
- **Playlists**: Create, edit, share, follow playlists (personal + editorial + algorithmic)
- **Library**: Save songs, albums, artists to personal library
- **Recommendations**: Discover Weekly, Daily Mix, Release Radar (personalized)
- **Social**: Follow friends, see what they're listening to, collaborative playlists
- **Offline mode**: Download songs for offline playback
- **Podcasts**: Stream and download podcast episodes
- **Queue**: Manage playback queue, shuffle, repeat
- **Cross-device**: Seamlessly switch playback between devices (Spotify Connect)

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Song starts playing in < 200 ms (buffer first 5 seconds)
- **High Availability**: 99.99%
- **Scalability**: 500M+ users, 100M+ songs
- **Smooth Playback**: No buffering interruptions under normal network conditions
- **Content Protection**: DRM to prevent unauthorized copying
- **Global**: Low latency worldwide via CDN
- **Personalization**: Real-time recommendation updates

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total users | 500M |
| DAU | 200M |
| Total songs | 100M |
| Avg song size | 5 MB (compressed @ 256 kbps) |
| Total music storage | 100M × 5 MB = 500 TB |
| Concurrent listeners | 50M |
| Bandwidth (256 kbps per listener) | 50M × 256 Kbps = 12.8 Tbps |
| Songs played / day | 2B (10 per user) |
| Streams / sec | ~23K |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Client (Mobile / Desktop / Web)                    │
│                                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Audio Player │  │ Browse /    │  │ Playlist    │  │ Spotify     │  │
│  │              │  │ Search      │  │ Manager     │  │ Connect     │  │
│  │ • Prefetch   │  │             │  │             │  │ (Device     │  │
│  │   next track │  │ • Autocmplt │  │ • Collab    │  │  Switch)    │  │
│  │ • Adaptive   │  │ • Fuzzy     │  │   playlists │  │             │  │
│  │   bitrate    │  │   search    │  │ • Offline   │  │             │  │
│  │ • Gapless    │  │             │  │   downloads │  │             │  │
│  │   playback   │  │             │  │             │  │             │  │
│  └──────┬───────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
└─────────┼──────────────────┼──────────────────┼──────────────┼─────────┘
          │ HTTP Range       │ HTTPS            │ HTTPS        │ WS/MQTT
          │ Requests         │                  │              │
   ┌──────▼───────┐   ┌─────▼──────────────────▼──────────────▼────────┐
   │     CDN      │   │              API Gateway                        │
   │ (Audio Edge  │   │  Auth (JWT), rate limiting, geo-routing         │
   │  Servers)    │   └──────┬───────────┬──────────┬──────────┬───────┘
   └──────┬───────┘          │           │          │          │
          │                  │           │          │          │
   ┌──────▼───────┐   ┌──────▼───┐ ┌────▼────┐ ┌──▼──────┐ ┌─▼──────────┐
   │ Object Store │   │ Music    │ │ Search  │ │Playlist │ │ Device     │
   │ (S3)        │   │ Catalog  │ │ Service │ │ Service │ │ Session    │
   │              │   │ Service  │ │         │ │         │ │ Service    │
   │ /track_id/   │   └────┬─────┘ └────┬────┘ └───┬─────┘ │(Spotify   │
   │  24.ogg     │        │           │          │       │ Connect)   │
   │  96.ogg     │   ┌────▼─────┐ ┌───▼────┐    │       └────┬───────┘
   │  160.ogg    │   │ MySQL    │ │Elastic │    │            │
   │  320.ogg    │   │(Metadata)│ │Search  │    │       ┌────▼───────┐
   └──────────────┘   └──────────┘ └────────┘    │       │ Redis      │
                                                 │       │(Session +  │
                                            ┌────▼────┐  │ Device Map)│
                                            │Cassandra│  └────────────┘
                                            │(Playlist│
                                            │ + Listen│
                                            │ History)│
                                            └─────────┘
┌────────────────────────────────────────────────────────────────────────┐
│                   RECOMMENDATION PIPELINE                              │
│                                                                        │
│  ┌──────────┐     ┌───────────┐     ┌──────────┐     ┌─────────────┐  │
│  │ Spark    │────▶│ Model     │────▶│ Redis    │────▶│ Recommend   │  │
│  │(Batch ML:│     │ Serving   │     │(Reco     │     │ Service     │  │
│  │ Collab   │     │(embed-    │     │ Cache:   │     │ (serves     │  │
│  │ Filter + │     │ ding ANN) │     │ discover │     │  /recommend │  │
│  │ Audio    │     │           │     │ _weekly,  │     │  endpoint)  │  │
│  │ Embed)   │     │           │     │ daily_mix)│     │             │  │
│  └──────────┘     └───────────┘     └──────────┘     └─────────────┘  │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                   PLAYBACK & ROYALTY PIPELINE                          │
│                                                                        │
│  Client ──playback report──▶ API ──▶ Kafka (playback-events)          │
│                                          │                             │
│                                    ┌─────▼──────┐                      │
│                                    │ Flink      │                      │
│                                    │(Dedup,     │                      │
│                                    │ validate   │                      │
│                                    │ ≥30s,      │                      │
│                                    │ fraud      │                      │
│                                    │ filter)    │                      │
│                                    └─────┬──────┘                      │
│                                          │                             │
│                              ┌───────────┼───────────┐                 │
│                              │           │           │                 │
│                        ┌─────▼───┐ ┌─────▼───┐ ┌────▼──────┐         │
│                        │ PostgreSQL│ │ClickHouse│ │ Royalty   │         │
│                        │(Stream   │ │(Listening│ │ Ledger DB │         │
│                        │ Counts)  │ │Analytics)│ │(Payments) │         │
│                        └──────────┘ └──────────┘ └───────────┘         │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Audio Streaming Architecture
1. Client requests song → API returns song metadata + CDN URL
2. Client downloads audio from CDN in chunks (HTTP range requests)
3. **Gapless playback**: Client prefetches next song's first 5 seconds while current song plays
4. **Crossfade**: Overlap end of current song with start of next song
5. **Audio format**: OGG Vorbis (free) at multiple bitrates:
   - 24 kbps (Low — mobile data saver)
   - 96 kbps (Normal)  
   - 160 kbps (High)
   - 320 kbps (Very High — premium only)
6. **Normalization**: ReplayGain / loudness normalization so songs play at consistent volume

#### Music Catalog Service
- Master database of all songs, albums, artists, metadata
- **Data source**: Record labels provide metadata via ingestion pipelines
- **Relationships**: Song → Album → Artist, Song → Genre, Song ↔ Song (features/remixes)
- **MySQL** sharded by artist_id or song_id

#### Search Service (Elasticsearch)
- Full-text search across songs, artists, albums, playlists, podcasts
- **Features**: Fuzzy matching ("bettles" → "Beatles"), autocomplete, did-you-mean
- **Index fields**: title, artist_name, album_name, genre, lyrics (if available), popularity_score
- **Ranking**: BM25 text relevance × popularity × recency

#### Playlist Service
- **Personal playlists**: User-created, stored in Cassandra
- **Collaborative playlists**: Multiple users can add/remove songs
- **Algorithmic playlists**: Generated by recommendation engine
  - **Discover Weekly**: 30 songs, refreshed every Monday
  - **Daily Mix**: 6 mixes based on listening clusters
  - **Release Radar**: New releases from followed artists + algorithmic picks

#### Recommendation Engine — Deep Dive

**Collaborative Filtering**:
- User-user: "Users with similar listening habits liked song X"
- Item-item: "Users who listened to song A also listened to song B"
- Matrix factorization (ALS — Alternating Least Squares) on user-song interaction matrix

**Content-Based Filtering**:
- Audio features: tempo, key, energy, danceability, acousticness (extracted via audio analysis ML models)
- Genre, artist similarity

**Spotify's Actual Approach** (simplified):
1. **Audio embeddings**: CNN analyzes raw audio → 128-dim embedding vector
2. **NLP on playlists**: Word2Vec-like model trained on playlist track sequences (playlist = "sentence", song = "word")
3. **Graph-based**: Artist/song knowledge graph
4. **Bandit exploration**: Deliberately recommend new/unfamiliar music to explore user preferences

**Serving**: Pre-compute top 100 recommendations per user (Spark batch job → Redis cache)

#### Spotify Connect (Cross-Device)
- Each device registers with the Device Session Service
- User can see all active devices and transfer playback
- **How**: Current device sends "transfer" command to server → server notifies target device → target device starts streaming from the same position
- Uses MQTT or WebSocket for real-time device communication

#### Device Session Service
- Tracks which devices a user has and which one is currently active
- **Redis storage**:
  - `session:{user_id}:active` → `{device_id}` (currently playing device)
  - `session:{user_id}:devices` → SET of `{device_id: {type, name, last_seen}}`
- On playback transfer: update `active` key → push notification to target device via MQTT/WebSocket
- On device disconnect (no heartbeat for 60s): remove from devices set
- Ensures only ONE device plays at a time (free tier restriction): check `active` device before allowing playback start

#### Playback & Royalty Pipeline
Tracks every stream for analytics, billing, and royalty payments to rights holders:

1. **Client reports playback**: `POST /api/v1/playback/report` with track_id, duration_listened_ms, completed, quality, device_id
2. **API Server** publishes to **Kafka** topic `playback-events` (partitioned by user_id)
3. **Flink streaming job** processes events:
   - **Deduplication**: Same user + same track + same timestamp → drop duplicate (client retry)
   - **Validation**: Duration ≥ 30 seconds to count as a "stream" (industry standard for royalty)
   - **Fraud filtering**: Bot detection — inhuman patterns (1000 plays/hour, zero skip rate, same track on loop from fresh accounts)
   - **Enrichment**: Join with track metadata (artist_id, label_id, territory)
4. **Outputs** (three sinks):
   - **PostgreSQL** (`stream_counts`): Increment per-track, per-artist, per-album counters (used for charts, popularity scores)
   - **ClickHouse** (`listening_analytics`): Full event details for analytics queries (per-country, per-device, per-hour breakdowns, user engagement metrics)
   - **Royalty Ledger DB**: Per-track stream count per territory per billing period → feeds monthly royalty calculation
5. **Monthly royalty calculation** (batch Spark job):
   - Total revenue pool for the month (subscription fees + ad revenue)
   - Per-track share = (track's streams / total platform streams) × revenue pool
   - Split per contractual agreement: label gets X%, artist gets Y%, songwriter gets Z%
   - Output: Payment records sent to payment service for disbursement

---

## 5. APIs

### Get Song / Stream
```http
GET /api/v1/tracks/{track_id}
Response: 200 OK
{
  "track_id": "track-uuid",
  "title": "Bohemian Rhapsody",
  "artist": {"id": "...", "name": "Queen"},
  "album": {"id": "...", "name": "A Night at the Opera", "cover_url": "..."},
  "duration_ms": 354000,
  "stream_url": "https://cdn.spotify.com/audio/{track_id}/320.ogg",
  "preview_url": "https://cdn.spotify.com/preview/{track_id}.mp3"
}
```

### Search
```http
GET /api/v1/search?q=bohemian+rhapsody&type=track,artist,album&limit=10
```

### Playlist Operations
```http
POST /api/v1/playlists
{ "name": "My Playlist", "description": "...", "public": true }

POST /api/v1/playlists/{playlist_id}/tracks
{ "track_ids": ["track-1", "track-2"], "position": 0 }

GET /api/v1/playlists/{playlist_id}/tracks?offset=0&limit=50
```

### Get Recommendations
```http
GET /api/v1/recommendations?seed_tracks=track-1,track-2&seed_artists=artist-1&limit=30
```

### Report Playback (for analytics + billing)
```http
POST /api/v1/playback/report
{
  "track_id": "track-uuid",
  "duration_listened_ms": 200000,
  "completed": false,
  "device_id": "device-uuid",
  "quality": "320kbps"
}
```

---

## 6. Data Model

### MySQL — Song Metadata

```sql
CREATE TABLE tracks (
    track_id        BIGINT PRIMARY KEY,
    title           VARCHAR(256),
    artist_id       BIGINT,
    album_id        BIGINT,
    duration_ms     INT,
    genre           VARCHAR(64),
    release_date    DATE,
    popularity      INT,          -- 0-100
    explicit        BOOLEAN,
    audio_features  JSON,         -- tempo, key, energy, etc.
    cdn_path        VARCHAR(512),
    created_at      TIMESTAMP,
    INDEX idx_artist (artist_id),
    INDEX idx_album (album_id),
    FULLTEXT idx_title (title)
);

CREATE TABLE artists (
    artist_id       BIGINT PRIMARY KEY,
    name            VARCHAR(256),
    bio             TEXT,
    image_url       TEXT,
    follower_count  BIGINT,
    genre           VARCHAR(64)
);

CREATE TABLE albums (
    album_id        BIGINT PRIMARY KEY,
    title           VARCHAR(256),
    artist_id       BIGINT,
    cover_url       TEXT,
    release_date    DATE,
    track_count     INT
);
```

### Cassandra — User Library & Listen History

```sql
CREATE TABLE user_library (
    user_id     UUID,
    item_type   TEXT,        -- 'track', 'album', 'artist', 'playlist'
    item_id     BIGINT,
    added_at    TIMESTAMP,
    PRIMARY KEY (user_id, item_type, added_at)
) WITH CLUSTERING ORDER BY (item_type ASC, added_at DESC);

CREATE TABLE listen_history (
    user_id         UUID,
    listened_at     TIMESTAMP,
    track_id        BIGINT,
    duration_ms     INT,
    completed       BOOLEAN,
    context_type    TEXT,     -- 'playlist', 'album', 'radio', 'search'
    context_id      TEXT,
    PRIMARY KEY (user_id, listened_at)
) WITH CLUSTERING ORDER BY (listened_at DESC)
  AND default_time_to_live = 7776000;  -- 90 days
```

### Cassandra — Playlists

```sql
CREATE TABLE playlists (
    playlist_id     UUID PRIMARY KEY,
    owner_id        UUID,
    name            TEXT,
    description     TEXT,
    is_public       BOOLEAN,
    follower_count  INT,
    track_count     INT,
    cover_url       TEXT,
    created_at      TIMESTAMP
);

CREATE TABLE playlist_tracks (
    playlist_id     UUID,
    position        INT,
    track_id        BIGINT,
    added_by        UUID,
    added_at        TIMESTAMP,
    PRIMARY KEY (playlist_id, position)
);
```

### S3 — Audio Files

```
Bucket: spotify-audio
Path:   /{track_id}/
Files:  24.ogg, 96.ogg, 160.ogg, 320.ogg
        preview.mp3 (30-second preview)
```

### Redis — Recommendation Cache

```
Key:    reco:{user_id}:discover_weekly
Value:  List of track_ids
TTL:    604800 (7 days, until next Monday)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **CDN failure** | Multi-CDN setup; fallback to alternate CDN or direct-from-origin |
| **Playback interruption** | Client buffers 30+ seconds ahead; can survive short outages |
| **Metadata DB failure** | MySQL read replicas; client caches metadata locally |
| **Recommendation service down** | Serve pre-cached recommendations from Redis; fallback to popularity-based |
| **Audio file corruption** | Checksum verification; original master files always preserved |

### Specific: Seamless Playback During Issues
- Client downloads 30-60 seconds of audio ahead of playback position
- If CDN is slow, quality auto-downgrades (320k → 160k → 96k)
- If CDN is unreachable, play from local cache/offline downloads
- "Offline mode" allows downloaded content to play without any network

---

## 8. Additional Considerations

### Audio Fingerprinting (Content ID)
- When labels upload new music, generate audio fingerprint (Chromaprint / Shazam-like)
- Detect duplicates, covers, unauthorized uploads
- Also used for: "What song is this?" feature

### Royalty Payment System
- Every stream > 30 seconds counts for royalty calculation
- **Pro-rata model**: Pool all subscription revenue → distribute based on share of total streams
- Requires accurate stream counting → stream reports are critical financial data
- **Data pipeline**: Listen events → Kafka → Flink aggregation → royalty DB → payment system

### Audio Quality and Encoding
- **Codec comparison**: OGG Vorbis (Spotify) vs AAC (Apple Music) vs FLAC (lossless)
- Spotify HiFi (lossless): FLAC at 1411 kbps → 4× bandwidth increase
- **Normalization**: Apply ReplayGain to ensure consistent volume across tracks

### Offline Downloads
- Encrypted audio files stored on device
- License check every 30 days (must connect to internet)
- DRM: Widevine (Android) / FairPlay (iOS) — prevent extraction of audio files
- Storage management: Auto-remove downloaded songs not played in 30 days

### Queue and Playback State
- User's current queue, playback position, shuffle state stored server-side
- Synced across devices in near real-time
- When switching devices (Spotify Connect), transfer the entire playback state

---

## 9. Deep Dive: Engineering Trade-offs

### Audio Streaming — How Chunks Are Delivered

```
Client requests: GET /audio/{track_id}/320.ogg
But NOT as a single giant download — uses HTTP Range Requests:

  Client                              CDN Edge
    │                                    │
    │ GET /audio/track-42/320.ogg        │
    │ Range: bytes=0-262143              │  ← First 256 KB chunk
    │───────────────────────────────────→│
    │                                    │
    │ 206 Partial Content                │
    │ Content-Range: bytes 0-262143/5242880
    │←───────────────────────────────────│
    │                                    │
    │ (decode + start playing after ~1s) │
    │                                    │
    │ Range: bytes=262144-524287         │  ← Second chunk (prefetch)
    │───────────────────────────────────→│
    │                                    │
    │ ...continues until song ends...    │

Why range requests (not full download)?
  1. Instant playback: start playing after first 256 KB (< 1 second at 256 kbps)
  2. Seek support: user scrubs to 3:00 → request bytes at offset for 3:00
     No need to download 0:00-3:00 first
  3. Bandwidth savings: user skips after 30 seconds → only downloaded 30s of audio
  4. Resumable: if connection drops, resume from last byte received

Prefetch strategy:
  While playing chunk N, request chunk N+1 in background
  Buffer target: 30-60 seconds ahead of playback position
  If buffer drops below 10 seconds → reduce quality (320 → 160 kbps)
```

### Adaptive Bitrate for Audio

```
Unlike video (which uses HLS/DASH manifests), audio streaming uses
simpler adaptive bitrate based on network probing:

  Client measures: download_time for each chunk
  Estimate bandwidth: chunk_size / download_time

  bandwidth > 500 kbps → stream 320 kbps (very high)
  bandwidth 200-500 kbps → stream 160 kbps (high)
  bandwidth 100-200 kbps → stream 96 kbps (normal)
  bandwidth < 100 kbps → stream 24 kbps (low, or pause to buffer)

Quality switch happens at chunk boundaries (not mid-chunk):
  Playing 320k chunk → bandwidth drops → next chunk requested at 160k
  Decoder handles codec switch (OGG Vorbis at any bitrate → same codec)
  User hears brief quality change but no interruption

Pre-encoded files:
  Each track stored as 4 separate files on S3:
    {track_id}/24.ogg, 96.ogg, 160.ogg, 320.ogg
  CDN caches all quality levels (most traffic hits 160 and 320)
  
  Why not dynamic transcoding?
    Audio transcoding is cheap (~0.1s per track) but:
    - Adds latency to first chunk
    - CDN can't cache dynamically generated content efficiently
    - Pre-encoded: S3 storage is cheaper than real-time compute
```

### Shuffle Algorithm — Why Naive Random Is Wrong

```
Naive random shuffle (Fisher-Yates):
  Randomly permute the queue → play in that order
  
  Problem: pure random can produce "clumpy" sequences
    12 songs, 4 by Artist A, 4 by Artist B, 4 by Artist C
    Random shuffle might produce: A, A, A, B, C, B, A, C, C, B, B, C
    → Three Artist A songs in a row → feels "not shuffled" to humans
  
  Humans expect "random" to mean "evenly spread" (not truly random)

Spotify's dithered shuffle algorithm:
  1. Group songs by artist
  2. Place each artist's songs evenly spaced across the queue:
     Artist A (4 songs): positions 0, 3, 6, 9
     Artist B (4 songs): positions 1, 4, 7, 10
     Artist C (4 songs): positions 2, 5, 8, 11
  3. Add small random jitter to each position (±1 position)
  4. Sort by jittered position → final shuffle order
  
  Result: A, B, C, A, B, C, A, B, C, A, B, C (with slight variation)
  → No artist clumping → feels "more random" to users
  
  Also spreads by: genre, tempo, mood
    Avoid: two slow ballads back-to-back
    Avoid: three hip-hop tracks then three classical
    
  This is why Spotify shuffle "sounds right" while true random doesn't.
```

### Collaborative Playlist — Concurrency Challenges

```
Two users edit the same playlist simultaneously:

  User A: adds "Song X" at position 3
  User B: removes song at position 2

  Without coordination:
    Original: [S1, S2, S3, S4]
    User A sees: [S1, S2, S3, S4] → inserts at 3 → [S1, S2, Song X, S3, S4]
    User B sees: [S1, S2, S3, S4] → removes at 2 → [S1, S3, S4]
    
    Server receives both → which one wins? Depends on order.
    If A first: [S1, S2, Song X, S3, S4] → B removes pos 2 → removes Song X (WRONG!)
    B intended to remove S2, not Song X.

Solution: Operate on track IDs, not positions

  User A: INSERT track_id="song-x" AFTER track_id="s2"
  User B: DELETE track_id="s2"
  
  Regardless of order:
    Result: [S1, Song X, S3, S4] (S2 removed, Song X inserted after where S2 was)
    OR: [S1, S3, S4, Song X] (Song X appended since its anchor S2 was deleted)
  
  Implementation in Cassandra:
    playlist_tracks keyed by (playlist_id, position)
    Each modification: read current state → compute new state → batch write
    Optimistic concurrency: IF version = expected_version (lightweight transaction)
    On conflict: re-read, re-apply, retry

  For real-time sync (multiple users viewing playlist):
    WebSocket: server pushes playlist diff to all connected viewers
    Clients apply diff to local state → instant UI update
```

### Stream Counting for Royalty Payments

```
A "stream" counts for royalties ONLY if:
  1. Song played for ≥ 30 seconds
  2. User is authenticated (not anonymous/preview)
  3. Not detected as bot/fraud playback

This is critical financial data — accuracy directly affects artist payments.

Data flow:
  Client sends playback report every 30 seconds:
    POST /playback/report
    { track_id, duration_listened_ms, quality, device_id, session_id }
  
  API Gateway → Kafka topic: playback-events (partitioned by user_id)
  
  Flink streaming job:
    1. Deduplicate: same (user_id, track_id, session_id) within 5 min → count once
    2. Validate: duration ≥ 30,000 ms
    3. Fraud detection:
       - Same user playing 500+ tracks/hour → bot
       - Same track on repeat 100+ times → fraud farm
       - Device fingerprint → known bot device → reject
    4. Aggregate: per-track daily stream count → write to PostgreSQL
    5. Publish: validated stream events → Kafka: royalty-events
  
  Nightly batch job:
    1. Read daily aggregates from PostgreSQL
    2. Compute per-artist, per-label stream share
    3. Pro-rata calculation:
       artist_payment = (artist_streams / total_streams) × total_revenue_pool
    4. Write to royalty ledger → finance team processes payouts

Race condition in deduplication:
  User plays song on phone → playback report sent
  User switches to laptop (Spotify Connect) → same song continues
  Laptop also sends playback report → same (user, track, session)
  
  Flink deduplication window: 5-minute session window per (user, track)
  First report counted, second suppressed
  Key: session_id changes on device switch → separate sessions → both count
  (This is correct: user deliberately chose to listen on two devices)
```

