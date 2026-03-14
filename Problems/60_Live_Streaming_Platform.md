# 60. Design a Live Streaming Platform like Twitch

---

## 1. Functional Requirements (FR)

- **Go live**: Streamers broadcast live video/audio from OBS, mobile app, or webcam
- **Watch live**: Viewers watch live streams with minimal delay (< 5 seconds glass-to-glass latency)
- **Live chat**: Real-time chat alongside the stream (thousands of messages/sec for popular streams)
- **Stream discovery**: Browse by category/game, recommended streams, search
- **Follow/Subscribe**: Follow streamers for notifications; paid subscriptions for perks
- **VOD**: Automatically save past broadcasts for on-demand viewing
- **Clips**: Viewers create short clips (30-60 sec) from live streams
- **Emotes**: Custom emoji/emotes per channel (subscriber-only emotes)
- **Stream quality**: Adaptive bitrate — viewer selects or auto-adjusts quality
- **Raids/Hosts**: Streamer redirects their audience to another channel
- **Moderation**: Chat moderation tools (ban, timeout, slow mode, subscriber-only chat)
- **Monetization**: Subscriptions, bits/donations, ads

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Glass-to-glass latency < 5 seconds (standard); < 2 seconds (low-latency mode)
- **Scale**: Support 100K+ concurrent streams; 50M+ concurrent viewers globally
- **Reliability**: Stream must not drop — even momentary interruption loses viewers
- **Chat Performance**: Handle 100K+ messages/sec across popular channels
- **Availability**: 99.99%
- **Global Distribution**: Low-latency viewing from any country
- **Cost Efficient**: Video bandwidth is the #1 cost center; optimize CDN usage
- **DVR**: Viewers can rewind live streams up to 2 hours

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Concurrent streamers | 100K |
| Concurrent viewers | 50M |
| Avg viewers per stream | 500 (power law: top 1% have 100K+) |
| Ingest bandwidth | 100K streams × 6 Mbps = 600 Gbps |
| Egress bandwidth | 50M viewers × 4 Mbps avg = 200 Tbps |
| Chat messages / sec | 500K (global); top stream: 50K/sec |
| VOD storage / day | 100K streams × 4 hrs avg × 2 GB/hr = 800 TB |
| Latency target | < 5 seconds (standard HLS); < 2 seconds (LL-HLS) |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐                              ┌──────────────┐
│  Streamer    │                              │  Viewer      │
│  (OBS/App)   │                              │  (Browser/   │
│              │                              │   App)       │
└──────┬───────┘                              └──────┬───────┘
       │ RTMP/SRT                                    │ HLS/LL-HLS
       │                                             │
┌──────▼───────────┐                    ┌────────────▼────────┐
│  Ingest Edge     │                    │        CDN          │
│  Servers         │                    │   (Edge Servers     │
│  (Regional PoPs) │                    │    200+ PoPs)       │
│                  │                    │                     │
│  RTMP → Decode   │                    │  LL-HLS segments    │
│  → Re-encode     │                    │  cached at edge     │
└──────┬───────────┘                    └────────────┬────────┘
       │                                             │ (cache miss)
       │ Internal transport                          │
       │ (SRT / RIST)                                │
┌──────▼───────────────────────────────────────┐     │
│            Transcoding Cluster               │     │
│                                              │     │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌───────┐│     │
│  │ 1080p  │ │  720p  │ │  480p  │ │ Audio ││     │
│  │ 60fps  │ │ 30fps  │ │ 30fps  │ │ only  ││     │
│  └────┬───┘ └────┬───┘ └────┬───┘ └───┬───┘│     │
│       │          │          │          │     │     │
│  ┌────▼──────────▼──────────▼──────────▼───┐│     │
│  │  HLS Packager (generate .ts segments    ││     │
│  │  + .m3u8 playlists every 2 seconds)     ││     │
│  └─────────────────────┬───────────────────┘│     │
└────────────────────────┼────────────────────┘     │
                         │                           │
                  ┌──────▼───────┐                   │
                  │  Origin      │◀──────────────────┘
                  │  Storage     │
                  │  (S3 + Origin│
                  │   Shield)    │
                  └──────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
   ┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────┐
   │ Chat        │ │ Stream    │ │ VOD       │
   │ Service     │ │ Metadata  │ │ Service   │
   │ (WebSocket  │ │ Service   │ │ (Archive  │
   │  cluster)   │ │ (MySQL)   │ │  past     │
   │             │ │           │ │  streams) │
   └─────────────┘ └───────────┘ └───────────┘
```

### Component Deep Dive

#### Ingest — Receiving the Live Stream

```
Streamer uses OBS Studio to broadcast:
  Protocol: RTMP (Real-Time Messaging Protocol) to ingest server
  Stream key: unique per channel (acts as authentication)
  Settings: 1080p 60fps, 6 Mbps bitrate, x264 encoder, keyframe interval 2s

RTMP Ingest Server:
  Receives RTMP stream → decodes → validates:
    - Stream key valid? (Redis lookup)
    - Bitrate within allowed limits? (prevent abuse)
    - Correct keyframe interval? (must be 2s for HLS segmentation)
  
  If valid → forward raw frames to transcoding cluster

RTMP vs SRT vs WebRTC for ingest:

RTMP:
  ✓ Universal: every streaming software supports it
  ✓ Mature, well-understood
  ✗ TCP-based → higher latency over long distances
  ✗ No built-in encryption (must use RTMPS = RTMP + TLS)
  ✗ Adobe Flash origin → technically deprecated

SRT (Secure Reliable Transport) ⭐:
  ✓ UDP-based → lower latency than RTMP
  ✓ Built-in encryption (AES-128/256)
  ✓ Forward error correction (handles packet loss gracefully)
  ✓ Designed for unreliable networks (mobile, satellite)
  ✗ Less universal than RTMP (growing adoption)

WebRTC (for browser-based streaming):
  ✓ No software needed — stream directly from browser
  ✓ Ultra-low latency (< 500 ms)
  ✗ Complex to manage at scale (STUN/TURN servers)
  ✗ Quality/bitrate constraints more limited than OBS

Twitch uses: RTMP for ingest (compatibility) + internal SRT transport
Future: migrating to SRT or QUIC-based protocols for better latency
```

#### Real-Time Transcoding — The Hardest Part

```
Unlike VOD transcoding (can take minutes), live transcoding must be REAL-TIME.
Each second of video must be transcoded in < 1 second (otherwise latency accumulates).

Pipeline per stream:
  1. Receive raw frames from ingest (1080p 60fps = 60 frames/sec)
  2. Transcode to multiple qualities IN PARALLEL:
     
     Quality    Resolution   FPS    Bitrate   Encoder
     Source     1080p        60     6 Mbps    (passthrough, for subscribers)
     High       720p        60     3 Mbps    x264/NVENC
     Medium     480p        30     1.5 Mbps  x264/NVENC
     Low        360p        30     0.8 Mbps  x264
     Audio only  —          —      128 Kbps  AAC
  
  3. Each quality output → HLS packager

Real-time constraint:
  1 second of 1080p 60fps → must encode 60 frames across 4 qualities
  x264 (CPU): one 1080p frame in ~5 ms → 60 frames = 300 ms ✓
  But 4 qualities: 300 ms × 4 = 1.2 seconds ✗ (sequential won't work!)
  
  Solution: Parallel encoding
    Each quality on its own CPU core / GPU stream
    GPU (NVENC): one frame in ~1 ms → 4 qualities × 60 frames = 240 ms total ✓
    
  Hardware allocation per stream:
    CPU encoding: 8-16 cores per stream (c6i.4xlarge = 16 vCPUs = 1-2 streams)
    GPU encoding: 1 GPU can handle 4-8 streams (NVENC has dedicated encoding ASIC)
    
  100K concurrent streams:
    GPU approach: 100K / 6 streams per GPU = 17K GPUs
    CPU approach: 100K × 8 cores = 800K cores → 50K servers
    
    GPU is 3× more cost-effective for live encoding

HLS Segment Generation:
  Every 2 seconds: cut the encoded stream into a segment
    segment_000001.ts (2 seconds of 720p video)
    segment_000002.ts
    ...
  
  Update the live playlist:
    #EXTM3U
    #EXT-X-TARGETDURATION:2
    #EXT-X-MEDIA-SEQUENCE:145
    #EXTINF:2.000,
    segment_000145.ts
    #EXTINF:2.000,
    segment_000146.ts
    #EXTINF:2.000,
    segment_000147.ts    ← latest segment
  
  Sliding window: playlist always contains last 3-5 segments
  Client polls playlist every 1-2 seconds → discovers new segments → downloads
```

#### Low-Latency HLS (LL-HLS) — Getting Below 3 Seconds

```
Standard HLS latency breakdown:
  Encoder buffer: 2 seconds (keyframe interval)
  Segment duration: 6 seconds (standard)
  Player buffer: 3 segments = 18 seconds
  CDN propagation: 1 second
  Total: ~25-30 seconds 😱

How to reduce:

1. Shorter segments (2 seconds instead of 6):
   Reduces segment wait time from 6s → 2s
   But: more HTTP requests, less CDN cache efficiency

2. LL-HLS with Partial Segments ⭐:
   Instead of waiting for full 2-second segment:
   Push 200ms "partial" segments
   
   Playlist:
   #EXT-X-PART:DURATION=0.2,URI="segment_147_part0.ts"
   #EXT-X-PART:DURATION=0.2,URI="segment_147_part1.ts"
   #EXT-X-PART:DURATION=0.2,URI="segment_147_part2.ts"
   ...
   
   Client downloads partial segments as they're produced
   Latency: encoder_buffer (2s) + partial (0.2s) + CDN (0.5s) + player (0.5s) = ~3 seconds

3. HTTP/2 Server Push or Chunked Transfer:
   Server pushes new partials to client without client polling
   Eliminates polling interval latency

4. Preload hints:
   #EXT-X-PRELOAD-HINT:TYPE=PART,URI="segment_147_part4.ts"
   Client pre-connects and waits for the NEXT partial before it's ready
   Segment arrives the instant it's produced

Latency comparison:
  Standard HLS:  25-30 seconds
  Short segments: 6-10 seconds
  LL-HLS:        2-5 seconds ⭐
  WebRTC:        < 1 second (but doesn't scale to millions of viewers)

Twitch uses: Custom low-latency protocol based on CMAF with chunked transfer
  Achieves ~2-3 seconds glass-to-glass for most streams
```

#### Chat System — 50K Messages/Second Per Channel

```
Popular streamer with 200K viewers. Chat moves FAST.

Architecture:
  Client → WebSocket connection → Chat Gateway → Kafka → Chat Processor → Fan-out

Chat Gateway (WebSocket servers):
  Each server: 100K WebSocket connections
  50M viewers → 500 WS servers
  
  Viewer sends message → WS server validates → publishes to Kafka
  Kafka topic: chat-messages, key = channel_id

Chat Processor (Flink):
  Consumes from Kafka → processes:
    1. Rate limit: max 1 message per 1.5 seconds per user (Twitch's actual limit)
    2. Spam filter: ML model or regex-based content filter
    3. Banned words: per-channel ban list
    4. Subscriber/follower check: some channels require subscription to chat
    5. Emote parsing: replace :pogchamp: with emote URL

Fan-out — The Scaling Challenge:
  200K viewers need to see each message → 200K WebSocket pushes per message
  At 50K messages/sec: 200K × 50K = 10 BILLION pushes/sec ← IMPOSSIBLE!

Solution: Sampling + batching

  1. Message batching:
     Don't send each message individually
     Batch messages per 100ms window → send batch to client
     50K msgs/sec → 500 msgs per 100ms window → 1 batch per 100ms
     Client renders batch smoothly

  2. Message sampling for massive channels:
     Channel with > 10K msgs/sec → viewers can't read them all anyway
     Show only 20 msgs/sec to each viewer (randomly sampled)
     "Highlighted" messages (subscriptions, donations) always shown
     
     Result: 200K viewers × 20 msgs/sec = 4M pushes/sec ← manageable!

  3. Channel-based pub/sub:
     Each WS server subscribes to Redis Pub/Sub for channels its clients watch
     Redis Pub/Sub: 1 publish → fan-out to all subscribing servers
     
     200K viewers across 500 servers → ~400 viewers per server per channel
     1 Redis publish → 500 server deliveries → each delivers to ~400 local clients
     
     Redis handles: 50K publishes/sec per channel (within Redis capability)

  4. Sharding hot channels:
     Top 10 channels: dedicated chat infrastructure
     Separate Kafka partitions, separate WS server pool
     Prevents one viral stream from affecting all chat
```

#### VOD — Automatic Stream Archive

```
While streaming, simultaneously save the stream for VOD:

During live stream:
  HLS segments (already transcoded) → copy to S3 "vod-archive" bucket
  Ordered: channel_id/stream_id/segment_000001.ts, segment_000002.ts, ...
  
  After stream ends:
  1. Generate complete HLS manifest (all segments, not sliding window)
  2. Generate DASH manifest
  3. Generate thumbnail (from key moments: peak viewer count frame)
  4. If stream was 4+ hours → split into chapters
  5. Run content moderation (async, can be after stream ends)
  6. Make available for on-demand playback

  Storage: 100K streams × 4 hrs × 2 GB/hr = 800 TB/day
  Retention: 60 days (Twitch standard for partners)
  After 60 days: auto-delete (unless streamer saves as "Highlight")
  
  Cost optimization:
    First 7 days: S3 Standard (frequently accessed)
    7-60 days: S3 Infrequent Access (50% cheaper)
    Saved highlights: S3 Standard (permanent)

Clips:
  Viewer clicks "Clip" → capture last 60 seconds of stream
  Backend:
    1. Find the last 30 HLS segments (30 × 2s = 60s)
    2. Concatenate into a single MP4 file
    3. Transcode to clip format (720p, H.264, 30fps)
    4. Store permanently in S3
    5. Return clip URL to viewer
  
  Latency: 5-10 seconds (fast enough for viral moments)
```

---

## 5. APIs

### Start Stream
```http
POST /api/v1/streams/start
{
  "channel_id": "ch-uuid",
  "title": "Friday Night Gaming",
  "category": "Fortnite",
  "tags": ["English", "Competitive"],
  "language": "en"
}
Response: 200 OK
{
  "stream_id": "stream-uuid",
  "ingest_url": "rtmp://ingest-us-east.example.com/live",
  "stream_key": "live_sk_abc123def456",
  "recommended_settings": {
    "resolution": "1920x1080",
    "fps": 60,
    "bitrate": "6000 kbps",
    "keyframe_interval": 2,
    "encoder": "x264",
    "rate_control": "CBR"
  }
}
```

### Get Live Stream (Viewer)
```http
GET /api/v1/streams/{channel_id}/live
Response: 200 OK
{
  "stream_id": "stream-uuid",
  "channel": {"name": "Ninja", "avatar": "..."},
  "title": "Friday Night Gaming",
  "viewer_count": 145230,
  "started_at": "2025-03-14T20:00:00Z",
  "manifest_url": "https://cdn.example.com/live/stream-uuid/master.m3u8",
  "chat_websocket": "wss://chat.example.com/ws/ch-uuid"
}
```

### Send Chat Message
```json
// WebSocket message
{
  "type": "chat_message",
  "channel_id": "ch-uuid",
  "content": "PogChamp that was insane!",
  "emotes": [{"name": "PogChamp", "position": [0, 9]}]
}

// Server response (broadcast to channel)
{
  "type": "chat_message",
  "id": "msg-uuid",
  "user": {"name": "viewer123", "color": "#FF0000", "badges": ["subscriber"]},
  "content": "PogChamp that was insane!",
  "emotes": [{"id": "88", "name": "PogChamp", "url": "https://..."}],
  "timestamp": "2025-03-14T20:15:23Z"
}
```

### Create Clip
```http
POST /api/v1/clips
{
  "stream_id": "stream-uuid",
  "title": "Insane play!",
  "duration_seconds": 30,
  "offset_seconds": -30
}
Response: 201 Created
{
  "clip_id": "clip-uuid",
  "url": "https://cdn.example.com/clips/clip-uuid.mp4",
  "status": "processing",
  "estimated_ready": "2025-03-14T20:16:00Z"
}
```

### Browse Streams
```http
GET /api/v1/streams?category=gaming&sort=viewers_desc&limit=20
Response: 200 OK
{
  "streams": [
    {"stream_id": "...", "channel": "Ninja", "title": "...",
     "viewer_count": 145230, "thumbnail": "https://...",
     "category": "Fortnite", "tags": ["English"]}
  ]
}
```

---

## 6. Data Model

### MySQL — Stream & Channel Metadata

```sql
CREATE TABLE channels (
    channel_id      UUID PRIMARY KEY,
    user_id         UUID NOT NULL UNIQUE,
    name            VARCHAR(50) UNIQUE NOT NULL,
    display_name    VARCHAR(50),
    description     TEXT,
    avatar_url      TEXT,
    banner_url      TEXT,
    stream_key      VARCHAR(64) NOT NULL,
    follower_count  INT DEFAULT 0,
    subscriber_count INT DEFAULT 0,
    partner         BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP
);

CREATE TABLE streams (
    stream_id       UUID PRIMARY KEY,
    channel_id      UUID NOT NULL,
    title           VARCHAR(255),
    category_id     INT,
    language        CHAR(2),
    status          ENUM('live', 'ended') DEFAULT 'live',
    viewer_count    INT DEFAULT 0,         -- periodically updated
    peak_viewers    INT DEFAULT 0,
    started_at      TIMESTAMP,
    ended_at        TIMESTAMP,
    vod_url         TEXT,
    duration_seconds INT,
    INDEX idx_channel (channel_id, started_at DESC),
    INDEX idx_category_live (category_id, status, viewer_count DESC),
    INDEX idx_live_viewers (status, viewer_count DESC)
);

CREATE TABLE clips (
    clip_id         UUID PRIMARY KEY,
    stream_id       UUID NOT NULL,
    channel_id      UUID NOT NULL,
    creator_id      UUID NOT NULL,
    title           VARCHAR(100),
    url             TEXT,
    duration_seconds SMALLINT,
    view_count      INT DEFAULT 0,
    created_at      TIMESTAMP,
    INDEX idx_stream (stream_id, view_count DESC),
    INDEX idx_channel (channel_id, created_at DESC)
);
```

### Redis — Live State

```
# Live stream state
stream:{channel_id}    → Hash { stream_id, title, category, started_at, 
                                 ingest_server, manifest_url, status }
TTL: none (deleted when stream ends)

# Viewer count (approximate, updated every 10 seconds)
viewers:{stream_id}    → INT (INCR/DECR on connect/disconnect)

# Stream key validation (ingest auth)
stream_key:{key}       → channel_id
TTL: none

# Chat rate limiting
chat_rate:{user_id}:{channel_id}  → INT (INCR, check < 1 per 1.5 sec)
TTL: 3

# Chat banned users per channel
chat_banned:{channel_id}  → SET of user_ids

# Top live streams by category (sorted set)
live_streams:{category}  → Sorted Set { channel_id: viewer_count }
live_streams:all         → Sorted Set { channel_id: viewer_count }
```

### Kafka Topics

```
Topic: stream-events         (stream started, ended, title changed)
Topic: chat-messages          (partitioned by channel_id for ordering)
Topic: viewer-events          (join, leave — for viewer count)
Topic: clip-requests          (async clip generation)
Topic: moderation-events      (ban, timeout, message delete)
```

### S3 — Video Storage

```
Bucket: live-segments (short retention, 48 hours)
  /{stream_id}/720p/segment_000001.ts
  /{stream_id}/720p/segment_000002.ts
  /{stream_id}/1080p/segment_000001.ts
  /{stream_id}/master.m3u8

Bucket: vod-archive (60-day retention)
  /{stream_id}/vod/master.m3u8
  /{stream_id}/vod/720p/playlist.m3u8
  /{stream_id}/vod/720p/segment_000001.ts
  ...

Bucket: clips (permanent)
  /{clip_id}/clip.mp4
  /{clip_id}/thumbnail.jpg
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Ingest server crash** | Streamer's software auto-reconnects to backup ingest server; 2-5 sec gap |
| **Transcoder crash** | Hot standby transcoder per stream; failover in < 1 second |
| **CDN edge failure** | CDN auto-routes to next nearest PoP; viewer sees brief rebuffer |
| **Origin storage failure** | S3 11 nines durability; origin shield with multiple backends |
| **Chat server crash** | WS reconnect → new server; message history replayed from Kafka |
| **Viewer count drift** | Periodic reconciliation: count actual WS connections per stream |
| **Stream key leaked** | Streamer can regenerate stream key instantly; invalidate old key |

### Specific: Handling a Stream That Goes Viral (1K → 500K Viewers in Minutes)

```
Scenario: Small streamer suddenly goes viral. 1K to 500K viewers in 5 minutes.

Challenge 1: CDN cache
  Before viral: segments cached at 2-3 edge PoPs
  After viral: viewers from 100+ PoPs → massive origin pull
  
  Solution:
  a. Origin shield: intermediate cache absorbs 90% of edge misses
  b. CDN pre-push: when viewer count > 50K, proactively push segments to all PoPs
  c. Multi-origin: segments replicated to 3 origin servers for redundancy

Challenge 2: Transcoding
  Stream was on a shared transcoder (handling 6 streams)
  500K viewers → this stream is critical; can't share resources
  
  Solution:
  a. Migrate to dedicated transcoder (seamless, < 1 second gap)
  b. Add more quality options (was only 720p → add 1080p, 480p for more ABR options)

Challenge 3: Chat
  1K viewers → ~10 msgs/sec (one WS server handles it)
  500K viewers → ~5K msgs/sec in channel + fan-out to 500K viewers
  
  Solution:
  a. Migrate channel to dedicated chat cluster
  b. Enable message sampling (show 20 msgs/sec to each viewer)
  c. Add slow mode (1 message per 5 seconds per user)
  
Challenge 4: Viewer count accuracy
  At 500K viewers, exact count is expensive to maintain
  Solution: approximate count with periodic reconciliation
  Show: "502K viewers" (not "502,347")
```

### Specific: Stream Latency Optimization

```
End-to-end latency budget:

Streamer's encoder → buffer → RTMP send:     500 ms
Network to ingest server:                     200 ms (regional PoP)
Ingest → Transcode:                           200 ms (decode + encode 1 segment)
Segment packaging (LL-HLS partial):           200 ms (partial segment duration)
Segment → Origin storage:                     100 ms
Origin → CDN edge:                            300 ms
CDN edge → Viewer download:                   200 ms
Player buffer:                                500 ms
Player decode + render:                       100 ms
─────────────────────────────────────────────────────
Total:                                       2,300 ms ≈ 2.3 seconds ⭐

Each step optimized:
  - Encoder: keyframe every 2 seconds, low-latency preset
  - Ingest PoP: regionally close to streamer
  - Transcode: GPU encoding (1 ms per frame)
  - LL-HLS: 200 ms partial segments (not 2-second full segments)
  - CDN: pre-connect viewers to edge, HTTP/2 multiplexing
  - Player: aggressive buffering (accept occasional rebuffer for lower latency)

Trade-off: Latency vs Reliability
  Lower latency = smaller player buffer = more frequent rebuffering
  Higher latency = larger buffer = smooth playback
  
  Let viewer choose:
    "Normal latency" (5 sec): smooth, no rebuffering
    "Low latency" (2-3 sec): occasional rebuffer during network dips
```

---

## 8. Deep Dive: Engineering Trade-offs

### HLS vs DASH vs WebRTC for Live Delivery

```
At scale (50M viewers), only HTTP-based protocols work:

HLS:
  ✓ Universal (iOS, Android, all browsers)
  ✓ Works with standard CDN (HTTP segments)
  ✓ LL-HLS brings latency to 2-5 seconds
  ✗ Latency higher than WebRTC

DASH:
  ✓ Open standard (no Apple dependency)
  ✓ LL-DASH similar to LL-HLS in latency
  ✗ iOS Safari support requires CMAF
  
  Industry moving to CMAF: unified format for both HLS and DASH

WebRTC:
  ✓ Ultra-low latency (< 500 ms)
  ✗ Peer-to-peer → doesn't scale to millions of viewers
  ✗ No CDN support (custom infrastructure needed)
  ✗ No DRM
  ✗ Each viewer needs a direct connection → 500K connections per stream
  
  Use case: Interactive streams (live auction, gaming with audience), < 1000 viewers

Twitch's actual approach:
  Custom protocol based on CMAF + chunked transfer encoding
  Segments pushed to CDN edge immediately as generated
  Player connects to nearest edge → receives chunks in real-time
  Achieves ~2-3 seconds end-to-end
```

### Viewer Count: Why It's Harder Than You Think

```
"Show how many people are watching" — sounds simple, right?

Challenge: 50M concurrent viewers across 500 WebSocket servers
  Each server knows its local connection count
  Global count = sum of all servers ← how to aggregate?

Approach 1: Central counter (Redis INCR/DECR)
  On WS connect: INCR viewers:{stream_id}
  On WS disconnect: DECR viewers:{stream_id}
  
  Problem: 50M viewers connecting/disconnecting → millions of Redis ops/sec
  More problems:
    - WebSocket server crashes → no DECR → count inflates
    - Network glitch → reconnect = INCR without DECR → count inflates
    - Eventually, count drifts significantly from reality

Approach 2: Periodic sweep ⭐ (Twitch's approach)
  Each WS server reports its per-stream count every 10 seconds:
    "Server 42: {stream-abc: 5230, stream-def: 120, stream-ghi: 8900}"
  
  Aggregator service sums all server reports:
    viewers[stream-abc] = sum(all server reports for stream-abc)
  
  Store in Redis: viewers:{stream_id} = aggregated count
  
  ✓ Self-correcting: if server crashes, it stops reporting → count drops
  ✓ No per-connection Redis operation
  ✗ 10-second staleness (acceptable — "145K viewers" doesn't need to be exact)

Approach 3: HyperLogLog (for unique viewer count, not concurrent)
  Redis: PFADD unique_viewers:{stream_id} {user_id}
  Count: PFCOUNT unique_viewers:{stream_id}
  Memory: 12 KB per stream (regardless of viewer count!)
  Accuracy: ±0.81% error
  Use for: "total unique viewers this stream" (not concurrent)
```

### Cost Analysis: Bandwidth Is Everything

```
50M concurrent viewers × 4 Mbps avg = 200 Tbps egress

CDN bandwidth cost: $0.02-0.08/GB (varies by provider and volume)
At $0.03/GB:
  200 Tbps × 3600 sec = 720 PB/hour
  720 PB × $0.03/GB = $21.6M per hour 😱
  
This is why Twitch operates at a loss on bandwidth alone.

Cost optimization strategies:

1. ABR pushes lower qualities:
   Default to 720p (3 Mbps) instead of 1080p (6 Mbps) for viewers
   Only show 1080p to subscribers → 50% bandwidth savings for free viewers
   
2. Multi-CDN:
   Use 3-4 CDN providers → negotiate volume discounts
   Route viewers to cheapest CDN with acceptable quality
   Savings: 20-40% vs single CDN

3. Own CDN (Twitch/Amazon):
   Build edge servers in major ISP facilities (peering agreements)
   Serve 60-70% of traffic from own infrastructure
   Savings: 50-70% vs commercial CDN

4. P2P CDN (experimental):
   Viewers relay segments to nearby viewers (WebRTC mesh)
   Each viewer becomes a mini-CDN node
   Reduces origin/edge bandwidth by 30-50%
   Challenges: NAT traversal, quality consistency, privacy

5. Codec efficiency:
   HEVC/AV1 → 30-50% less bandwidth for same quality
   Requires client support (growing but not universal)
```

### Transcoding: Dedicated Per-Stream vs Shared Pool

```
Dedicated (one transcoder per stream):
  ✓ Isolated: one stream's issues don't affect others
  ✓ Predictable latency (no resource contention)
  ✗ Expensive: 100K streams → 100K transcoders
  ✗ Under-utilized: most streams are 720p 30fps (need only 20% of a server)

Shared pool (multiple streams per server):
  ✓ Cost-efficient: 6-8 streams per GPU server
  ✓ Better resource utilization
  ✗ Noisy neighbor: one 4K 60fps stream hogs resources → affects others
  ✗ More complex scheduling

Hybrid approach ⭐ (Twitch):
  Tier 1 (partners, > 1000 avg viewers): Dedicated transcoder
    ~5K streams → 5K dedicated instances
    Premium: multiple quality options (160p to 1080p60)
  
  Tier 2 (affiliates, 10-1000 viewers): Shared pool (4 streams per instance)
    ~45K streams → ~12K instances
    Standard: 3 quality options (480p, 720p, source)
  
  Tier 3 (everyone else, < 10 viewers): Source quality only (no transcoding!)
    ~50K streams → 0 transcoding cost!
    Viewer gets whatever quality the streamer sends
    Saves: 50% of total transcoding cost
```



