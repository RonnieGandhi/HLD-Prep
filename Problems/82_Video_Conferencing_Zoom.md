# 82. Design a Video Conferencing System (like Zoom)

---

## 1. Functional Requirements (FR)

- One-on-one and group video/audio calls (up to 1,000 participants)
- Screen sharing with annotation support
- Real-time chat during calls (text, file sharing)
- Meeting scheduling with calendar integration
- Meeting recording with cloud storage and playback
- Virtual backgrounds, noise cancellation
- Breakout rooms for large meetings
- Waiting room with host admission control
- Raise hand, reactions, polls during meetings
- Join via browser (WebRTC), desktop app, or phone (PSTN dial-in)
- End-to-end encryption for 1:1 calls

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: < 150ms glass-to-glass (camera capture to display) for acceptable experience; < 400ms tolerable
- **High Availability**: 99.99% — outages during calls are catastrophic
- **Scalability**: 100M+ concurrent users, 10M+ concurrent meetings
- **Adaptive Quality**: Gracefully degrade on poor networks (lower resolution, audio-only fallback)
- **Global**: Edge servers in 50+ regions to minimize round-trip
- **Reliability**: No dropped frames under normal conditions; reconnect within 2 seconds on network blip
- **Security**: E2E encryption for 1:1, TLS for group calls, SOC 2 compliance

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Concurrent meetings | 10M |
| Avg participants per meeting | 5 |
| Concurrent streams | 50M |
| Bandwidth per participant | 2 Mbps down + 1.5 Mbps up (720p) |
| Total bandwidth | 50M × 3.5 Mbps = 175 Pbps (!!) → Edge distribution critical |
| Audio-only bandwidth | 50 kbps per participant |
| Recording storage / day | 5M recorded meetings × 30 min × 500 MB/hr = 1.25 PB/day |
| Signaling messages / sec | 10M meetings × 2 signals/sec = 20M/sec |
| Chat messages / sec | 500K |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                            CLIENTS                                    │
│   Desktop App / Browser (WebRTC) / Mobile App / PSTN Phone           │
│                                                                       │
│   ┌───────────────────────────────────────────┐                       │
│   │  Media Engine (per client)                │                       │
│   │  - Camera/mic capture                     │                       │
│   │  - VP8/VP9/H.264/AV1 encode               │                       │
│   │  - Opus audio encode                      │                       │
│   │  - Jitter buffer, FEC, NACK               │                       │
│   │  - Adaptive bitrate (sender-side)         │                       │
│   │  - Echo cancellation, noise suppression   │                       │
│   │  - Virtual background (local ML)          │                       │
│   └──────────────────────┬────────────────────┘                       │
│                          │ SRTP/DTLS (encrypted media)                │
│                          │ + WebSocket (signaling)                     │
└──────────────────────────│────────────────────────────────────────────┘
                           │
┌──────────────────────────│────────────────────────────────────────────┐
│              SIGNALING LAYER                                          │
│                          │                                            │
│              ┌───────────▼───────────┐                                │
│              │  Signaling Server     │  (Stateless, horizontally      │
│              │                       │   scalable)                     │
│              │  - SDP exchange       │                                │
│              │    (offer/answer)     │                                │
│              │  - ICE candidate      │                                │
│              │    exchange           │                                │
│              │  - Room management    │                                │
│              │  - Participant join/  │                                │
│              │    leave events       │                                │
│              │  - Chat relay         │                                │
│              │  - Meeting controls   │                                │
│              │    (mute, kick, etc.) │                                │
│              └───────────┬───────────┘                                │
│                          │                                            │
└──────────────────────────│────────────────────────────────────────────┘
                           │
┌──────────────────────────│────────────────────────────────────────────┐
│              MEDIA LAYER  (Global Edge Network)                       │
│                          │                                            │
│    ┌─────────────────────▼─────────────────────┐                      │
│    │          SFU (Selective Forwarding Unit)   │                      │
│    │          One SFU cluster per meeting       │                      │
│    │                                            │                      │
│    │  How SFU works:                            │                      │
│    │  - Each participant sends ONE stream to SFU│                      │
│    │  - SFU selectively forwards streams to     │                      │
│    │    each participant based on:              │                      │
│    │    · Layout (who's visible)                │                      │
│    │    · Bandwidth (send lower quality to       │                      │
│    │      constrained clients)                  │                      │
│    │    · Speaking (active speaker gets HD,      │                      │
│    │      others get thumbnail)                 │                      │
│    │  - Simulcast: sender encodes 3 qualities   │                      │
│    │    (1080p, 720p, 180p) → SFU picks per     │                      │
│    │    receiver                                │                      │
│    │  - SVC: Scalable Video Coding — single     │                      │
│    │    layered stream, SFU drops layers         │                      │
│    └──────────┬──────────────────┬──────────────┘                      │
│               │                  │                                      │
│    ┌──────────▼──────┐  ┌───────▼──────────┐                           │
│    │  TURN/STUN      │  │  Cascaded SFU    │                           │
│    │  Server         │  │  (for multi-     │                           │
│    │                 │  │   region)         │                           │
│    │  - NAT traversal│  │                  │                           │
│    │  - Relay for    │  │  SFU in US-East  │                           │
│    │    symmetric NAT│  │  ←── relay ──►   │                           │
│    │  - Allocated per│  │  SFU in EU-West  │                           │
│    │    edge region  │  │                  │                           │
│    └─────────────────┘  │  Only 1 copy of  │                           │
│                         │  each stream     │                           │
│                         │  crosses regions │                           │
│                         └──────────────────┘                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    BACKEND SERVICES                                    │
│                                                                        │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Meeting    │  │ Recording   │  │ Auth &       │  │ Analytics    │  │
│  │ Service    │  │ Service     │  │ Identity     │  │ Service      │  │
│  │            │  │             │  │              │  │              │  │
│  │ - Create   │  │ - Composite │  │ - SSO/OAuth  │  │ - Call       │  │
│  │   meeting  │  │   recording │  │ - JWT tokens │  │   quality    │  │
│  │ - Schedule │  │   (mix all  │  │ - E2E key    │  │   metrics    │  │
│  │ - Join/    │  │   streams)  │  │   exchange   │  │ - Usage      │  │
│  │   leave    │  │ - Store to  │  │ - Meeting    │  │   stats      │  │
│  │ - Controls │  │   S3/GCS    │  │   passwords  │  │ - Network    │  │
│  │ - Breakout │  │ - Transcode │  │ - Waiting    │  │   quality    │  │
│  │   rooms    │  │   to MP4    │  │   room auth  │  │   per region │  │
│  └─────┬──────┘  └──────┬──────┘  └──────────────┘  └──────┬───────┘  │
│        │                │                                    │          │
│  ┌─────▼────────────────▼────────────────────────────────────▼───────┐  │
│  │                     DATA LAYER                                    │  │
│  │                                                                   │  │
│  │  PostgreSQL          Redis              S3 / GCS      Kafka       │  │
│  │  - Meetings          - Active rooms     - Recordings  - Call      │  │
│  │  - Users             - Participants     - Shared      quality     │  │
│  │  - Schedules         - TURN allocations   files       events     │  │
│  │  - Chat history      - Rate limits                    - Chat      │  │
│  │  - Billing           - Session tokens                   events    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### Why SFU Over MCU and Mesh?

| Topology | How It Works | Pros | Cons | Use When |
|---|---|---|---|---|
| **Mesh** | Each participant sends to every other (P2P) | No server cost, lowest latency for 1:1 | N² streams, unscalable past 4-5 users | 1:1 calls |
| **MCU** | Server decodes all, composites into one stream, re-encodes | Client uploads 1, downloads 1 stream | Massive server CPU (decode+encode), no per-client quality | Legacy telephony |
| **SFU** ✅ | Server forwards streams without decoding | Low server CPU, per-client quality selection, simulcast | More downstream bandwidth than MCU | 2+ participant calls |

**Zoom/Google Meet/Teams all use SFU** because:
1. Server doesn't decode → 10× less CPU than MCU
2. Simulcast allows per-client quality adaptation
3. Can selectively forward only visible/speaking participants

### Simulcast & SVC

```
Simulcast (used by most):
  Sender encodes 3 independent streams:
    High:   1080p @ 2.5 Mbps
    Medium:  720p @ 1.0 Mbps  
    Low:     180p @ 150 kbps
  
  SFU selects per receiver:
    - Active speaker → receives High from speaker, Low from others
    - Gallery view → receives Medium from all
    - Mobile on 3G → receives Low from all

SVC (Scalable Video Coding):
  Sender encodes single layered stream:
    Base layer:  180p @ 150 kbps  (always sent)
    Layer 1:     + 360p @ 500 kbps
    Layer 2:     + 720p @ 1.0 Mbps
    Layer 3:     + 1080p @ 2.5 Mbps
  
  SFU drops upper layers for constrained receivers
  Advantage: single encode, seamless quality transitions
  Used by: Google Meet (VP9-SVC), Zoom (newer versions)
```

---

## 5. APIs

### REST APIs

```
POST   /api/meetings                        → Create/schedule meeting
GET    /api/meetings/{meeting_id}           → Get meeting details
PUT    /api/meetings/{meeting_id}           → Update meeting settings
DELETE /api/meetings/{meeting_id}           → Cancel meeting
POST   /api/meetings/{meeting_id}/join      → Join meeting (get SFU endpoint)
POST   /api/meetings/{meeting_id}/leave     → Leave meeting
POST   /api/meetings/{meeting_id}/record    → Start/stop recording
GET    /api/meetings/{meeting_id}/recording → Get recording URL
POST   /api/meetings/{meeting_id}/breakout  → Create breakout rooms
GET    /api/meetings/{meeting_id}/chat      → Get chat history
```

### WebSocket Signaling Protocol

```json
// Join meeting
{ "type": "join", "meeting_id": "m_123", "token": "jwt...", "media": {"audio": true, "video": true} }

// SDP Offer/Answer (WebRTC handshake)
{ "type": "offer", "sdp": "v=0\r\n...", "target": "sfu" }
{ "type": "answer", "sdp": "v=0\r\n...", "from": "sfu" }

// ICE candidate exchange
{ "type": "ice_candidate", "candidate": "candidate:... udp 2113937151 ...", "sdpMid": "0" }

// Meeting controls
{ "type": "mute", "target_user": "u_456", "media": "audio" }
{ "type": "raise_hand" }
{ "type": "reaction", "emoji": "👍" }
{ "type": "screen_share_start", "stream_id": "screen_1" }

// Participant events (server → client)
{ "type": "participant_joined", "user": {"id": "u_789", "name": "Alice"} }
{ "type": "participant_left", "user_id": "u_789" }
{ "type": "active_speaker", "user_id": "u_456" }
{ "type": "quality_update", "max_quality": "720p", "reason": "bandwidth" }
```

---

## 6. Data Models

### PostgreSQL

```sql
CREATE TABLE meetings (
    meeting_id      UUID PRIMARY KEY,
    host_id         UUID NOT NULL,
    title           TEXT,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    status          TEXT DEFAULT 'scheduled',  -- scheduled|active|ended
    password        TEXT,
    settings        JSONB,  -- {waiting_room, mute_on_join, allow_recording, e2e_encryption}
    max_participants INT DEFAULT 100,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE meeting_participants (
    meeting_id   UUID REFERENCES meetings(meeting_id),
    user_id      UUID,
    display_name TEXT,
    join_time    TIMESTAMPTZ,
    leave_time   TIMESTAMPTZ,
    role         TEXT DEFAULT 'participant',  -- host|co-host|participant
    device_type  TEXT,  -- desktop|mobile|phone
    PRIMARY KEY (meeting_id, user_id, join_time)
);

CREATE TABLE meeting_recordings (
    recording_id UUID PRIMARY KEY,
    meeting_id   UUID,
    storage_url  TEXT,   -- S3 URL
    duration_sec INT,
    size_bytes   BIGINT,
    format       TEXT,   -- mp4|webm
    status       TEXT,   -- recording|processing|ready|failed
    created_at   TIMESTAMPTZ
);
```

### Redis

```
# Active meeting state
HSET meeting:active:{meeting_id} 
    host "u_123"
    sfu_endpoint "sfu-us-east-3.zoom.us:443"
    participant_count 15
    started_at "2026-03-14T10:00:00Z"

# Participant list (sorted set by join time)
ZADD meeting:participants:{meeting_id} {join_timestamp} {user_id}

# TURN allocation cache
SET turn:alloc:{user_id}:{meeting_id} "turn-server-17:3478" EX 3600

# Rate limiting (meeting joins per minute)
INCR rate:join:{meeting_id} 
EXPIRE rate:join:{meeting_id} 60
```

### Kafka Topics

```
Topic: meeting-events (partitioned by meeting_id)
  - meeting.created, meeting.started, meeting.ended
  - participant.joined, participant.left

Topic: call-quality-metrics (partitioned by user_id)
  - {meeting_id, user_id, timestamp, packet_loss, jitter, rtt, bitrate, fps, resolution}
  
Topic: chat-messages (partitioned by meeting_id)
  - {meeting_id, user_id, message, timestamp}
```

---

## 7. Fault Tolerance & Deep Dives

### SFU Failure Recovery

```
Problem: SFU server handling a meeting crashes mid-call

Solution: SFU Cascading + Fast Failover
1. Each meeting has a primary SFU and a hot-standby SFU
2. Hot-standby receives participant list + ICE candidates from signaling layer
3. On primary failure:
   a. Signaling server detects (heartbeat miss, < 3 sec)
   b. Signals all clients: "reconnect to SFU-backup"
   c. Clients perform ICE restart (new DTLS handshake, ~1-2 sec)
   d. Media resumes on backup SFU
4. User experience: 2-3 second freeze, then auto-recovery

Optimization: "Warm" SFU
  - Standby SFU already has ICE candidates cached
  - On failover, skip full ICE negotiation → reconnect in < 1 sec
```

### Network Quality Adaptation

```
Client continuously monitors:
  - Packet loss rate (via RTCP receiver reports)
  - Round-trip time (via STUN binding requests)
  - Available bandwidth (via REMB / Transport-CC)

Adaptation strategy:
  Loss < 2%  → Full quality (1080p, 30fps)
  Loss 2-5%  → Reduce to 720p
  Loss 5-10% → Reduce to 360p, 15fps
  Loss > 10% → Audio-only mode, disable video
  
FEC (Forward Error Correction):
  - Send redundant packets (e.g., 10% overhead)
  - Receiver recovers lost packets without retransmission
  - Critical for audio (any gap is audible)

NACK (Negative Acknowledgement):
  - Client detects missing packet (sequence gap)
  - Requests retransmission from SFU
  - SFU retransmits from its jitter buffer
  - Only useful if RTT < 100ms (otherwise too late)
```

### Large Meeting Optimization (1000+ participants)

```
Problem: 1000 participants in a webinar

Solution: Tiered architecture
  Tier 1: Speakers (5-10) → Full SFU mesh, send+receive video
  Tier 2: Active participants → Receive video, can unmute
  Tier 3: Viewers → Receive-only via CDN (HLS/DASH, 5-10s delay)

For Tier 3 (viewers):
  - SFU composites speaker streams into single output
  - Transcodes to HLS → pushes to CDN
  - Viewers watch via regular video player
  - Chat/reactions still via WebSocket (real-time)
  
  This reduces from 1000 WebRTC connections to ~10 WebRTC + 990 CDN viewers
```

### Multi-Region Cascaded SFU

```
Meeting with participants in US-East, EU-West, APAC:

Without cascading:
  All 30 participants connect to one SFU in US-East
  EU and APAC users have 200ms+ RTT → bad experience

With cascading:
  SFU-US-East ← 10 US participants
  SFU-EU-West ← 10 EU participants  
  SFU-APAC   ← 10 APAC participants
  
  SFU-to-SFU relay: each stream crosses region ONCE
  Each SFU fan-outs locally
  
  Bandwidth: 3 × (10 streams × 1 Mbps) cross-region = 30 Mbps inter-region
  vs. 30 × (29 streams × 1 Mbps) = 870 Mbps if all direct → 29× saving
```

### End-to-End Encryption (E2E)

```
Challenge: SFU needs to read RTP headers for routing but shouldn't decrypt media

Solution: Insertable Streams API (SFrame)
1. Client encrypts media payload before encoding into RTP
2. RTP headers remain unencrypted → SFU can route
3. Media payload encrypted with per-meeting key
4. Key exchange: Diffie-Hellman via signaling channel
5. SFU cannot decrypt content — true E2E

Limitation:
  - Only works for small meetings (key management)
  - Recording impossible without a trusted recorder
  - Server-side features (noise cancellation, virtual BG) must be client-side
```

### Recording Architecture

```
1. Recording bot joins as invisible participant on SFU
2. Bot receives all streams (audio + video)
3. Bot performs real-time compositing:
   - Active speaker layout or gallery view
   - Mixes audio streams
   - Renders to single 1080p video
4. Writes chunks to temporary storage (local SSD)
5. On meeting end:
   - Upload chunks to S3/GCS
   - Trigger transcoding pipeline (MP4, subtitles, chapters)
   - Generate searchable transcript (speech-to-text)
   - Send notification with recording link

For large meetings: composite on GPU-accelerated servers (NVIDIA T4)
Storage: 1080p @ 30fps → ~500 MB/hour after H.264 encoding
```

---

## 8. Additional Considerations

### Noise Cancellation & Virtual Background
- Run ML models locally on client (RNNoise for audio, MediaPipe for segmentation)
- Client-side processing: no server involvement, privacy preserved
- GPU acceleration via WebGPU/WebGL in browser

### PSTN Integration
- SIP trunk gateway converts phone calls to WebRTC
- PSTN users: audio-only, DTMF for controls (mute = *6)
- Dial-in numbers per country, toll-free options

### Scalability of Signaling Layer
- Signaling servers are stateless — horizontal scaling behind load balancer
- Redis Pub/Sub for cross-server meeting room state
- Each meeting's signaling pinned to one server via consistent hashing (or any server can handle with Redis backing)

### Why WebRTC Specifically?
```
WebRTC provides:
1. NAT traversal (ICE/STUN/TURN) — works behind firewalls
2. Encrypted media (SRTP/DTLS) — built-in security  
3. Adaptive bitrate — bandwidth estimation built-in
4. Codec negotiation — H.264/VP8/VP9/AV1 via SDP
5. Browser-native — no plugins needed
6. Sub-second latency — unlike HLS/DASH (5-30s delay)

Alternative: Custom UDP protocol (Zoom's original approach)
  - Even lower latency
  - Full control over congestion control
  - But: doesn't work in browsers without plugin, firewall issues
  - Zoom now also supports WebRTC for browser clients
```

### Meeting Quality Monitoring
```
Every 5 seconds, each client reports:
  - Packet loss (%), jitter (ms), RTT (ms)
  - Current resolution sent/received
  - CPU usage, encode time
  - Available bandwidth (estimated)

Backend aggregates in ClickHouse:
  - Real-time dashboard: meetings with quality issues
  - Alerts: if > 20% of participants have poor quality
  - Regional analysis: identify bad PoPs or ISP issues
  - Historical analysis: 95th percentile quality by region
```

