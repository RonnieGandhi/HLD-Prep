# 49. Design a User Presence System (Online/Offline Status)

---

## 1. Functional Requirements (FR)

- **Show online status**: Green dot / "Online" for active users
- **Show last seen**: "Last seen 5 minutes ago" for offline users
- **Real-time updates**: Status changes propagate to friends within 5 seconds
- **Typing indicator**: "Alice is typing..." in chat
- **Privacy controls**: Hide online status from all or specific users
- **Multi-device**: Online on any device → online; all offline → offline
- **Away/Idle**: Mark "Away" after 5 min inactivity

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Status propagated in < 5 seconds
- **Scale**: 500M+ concurrent users; ~500 friends each
- **Bandwidth Efficient**: Don't flood network
- **Eventual Consistency**: 5-second lag acceptable
- **Availability**: 99.99%

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Concurrent online users | 200M |
| Heartbeat interval | 30 seconds |
| Heartbeats / sec | ~6.7M |
| Status changes / sec | ~2M |
| Avg friends per user | 500 |
| Fan-out (optimized) | ~15 per change (subscription-based) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Client (Mobile/Desktop/Web)                          │
│                                                                        │
│  Heartbeat Logic:                                                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ • App open → send {type: "status", status: "online"}            │  │
│  │ • Every 30s (desktop) / 60s (mobile) → ping frame heartbeat    │  │
│  │ • App background → send {type: "status", status: "away"}       │  │
│  │ • App close → send {type: "status", status: "offline"}         │  │
│  │ • Force kill → no event sent → server TTL handles it (60-120s) │  │
│  │ • 5 min no touch/scroll → client sends "away" automatically    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  Receive Presence Updates:                                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ • Only subscribe for friends currently visible on screen        │  │
│  │   (chat list, conversation header, friend list)                 │  │
│  │ • On navigate to chat with Bob → subscribe to Bob's presence   │  │
│  │ • On navigate away → unsubscribe from Bob                      │  │
│  │ • Updates received via existing WebSocket connection            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────┬────────────────────────────────────────────────┘
                        │ WebSocket (persistent connection)
                        ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  WebSocket Gateway Cluster                             │
│                                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       ┌──────────┐        │
│  │ WS Srv 1 │  │ WS Srv 2 │  │ WS Srv 3 │  ...  │ WS Srv N │        │
│  │ 100K conn│  │ 100K conn│  │ 100K conn│       │ 100K conn│        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       └────┬─────┘        │
│       │              │              │                   │              │
│  Each server maintains local connection map:                          │
│  { user_id → WebSocket_conn_reference }                              │
│                                                                        │
│  Connection routing (WHERE is user connected?):                       │
│  Redis Hash: ws_location:{user_id} → "ws-server-3:port"             │
│  Set on connect, deleted on disconnect                                │
│  Used by Fan-Out Service to route presence pushes                    │
└───────────────────────┬────────────────────────────────────────────────┘
                        │
         ┌──────────────┼──────────────────────┐
         │              │                      │
  ┌──────▼──────┐ ┌─────▼───────┐  ┌──────────▼───────────┐
  │ Heartbeat   │ │ Presence    │  │ Subscription         │
  │ Processor   │ │ Query       │  │ Manager              │
  │             │ │ Service     │  │                      │
  │ Receives    │ │             │  │ Manages who is       │
  │ 6.7M pings │ │ "Is Alice   │  │ subscribed to whose  │
  │ /sec       │ │  online?"   │  │ presence changes     │
  │             │ │             │  │                      │
  │ For each:  │ │ Batch API:  ���  │ On subscribe:        │
  │ SET presence│ │ check 50    │  │ SADD presence_subs:  │
  │ :{uid}:    │ │ user_ids in │  │   {target} {subber}  │
  │ {device}   │ │ one call    │  │                      │
  │ "online"   │ │ (pipeline)  │  │ On unsubscribe:      │
  │ EX 60      │ │             │  │ SREM                 │
  │             │ │             │  │                      │
  │ Detect     │ │ Privacy     │  │ On target status     │
  │ status     │ │ filter:     │  │ change → notify only │
  │ CHANGE:    │ │ check perms │  │ subscribers          │
  │ was offline│ │ before      │  │                      │
  │ now online │ │ returning   │  │ Cleanup: unsubscribe │
  │ → Kafka    │ │ presence    │  │ on WS disconnect     │
  │ event      │ │             │  │                      │
  └──────┬─────┘ └─────────────┘  └──────────────────────┘
         │
  ┌──────▼───────────────────────────────────────────────────────┐
  │                    Redis Cluster                              │
  │                                                               │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │ Presence State (per user per device):                    │ │
  │  │   presence:{uid}:mobile  = "online"   TTL: 120s         │ │
  │  │   presence:{uid}:desktop = "online"   TTL: 60s          │ │
  │  │   presence:{uid}:web     = "away"     TTL: 60s          │ │
  │  └─────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │ Aggregated Status:                                       │ │
  │  │   presence_agg:{uid} = Hash {                            │ │
  │  │     status: "online",                                    │ │
  │  │     last_active: 1710412800,                             │ │
  │  │     devices: "mobile,desktop"                            │ │
  │  │   }  TTL: 120s                                           │ │
  │  └─────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │ Subscription Sets:                                       │ │
  │  │   presence_subs:{uid} = SET of user_ids wanting updates │ │
  │  │   Example: presence_subs:alice = {bob, carol}            │ │
  │  │   (Bob and Carol have Alice's chat open)                 │ │
  │  └─────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │ WebSocket Location Routing:                              │ │
  │  │   ws_location:{uid} = "ws-server-3:8080"                │ │
  │  │   (Which WS server holds this user's connection)         │ │
  │  └─────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │ Typing Indicator (ephemeral):                            │ │
  │  │   typing:{conversation_id}:{uid} = "1"  TTL: 5s         │ │
  │  │   Auto-expires if user stops typing                      │ │
  │  └─────────────────────────────────────────────────────────┘ │
  └──────────────────────┬────────────────────────────────────────┘
                         │
  ┌──────────────────────▼────────────────────────────────────────┐
  │              Fan-Out / Notification Service                    │
  │                                                                │
  │  Triggered on status CHANGE (online↔offline, online↔away):   │
  │                                                                │
  │  1. Read subscribers: SMEMBERS presence_subs:{uid}            │
  │     → [bob, carol] (only users with chat/list open)           │
  │                                                                │
  │  2. For each subscriber:                                      │
  │     a. Read ws_location:{subscriber} → "ws-server-7:8080"    │
  │     b. Send inter-server message to ws-server-7:              │
  │        {type: "presence_update", user: uid, status: "offline"}│
  │     c. ws-server-7 pushes to subscriber's WebSocket           │
  │                                                                │
  │  3. Batching optimization:                                    │
  │     Collect all presence changes in 3-second window           │
  │     Send ONE message per subscriber with all changes:          │
  │     {updates: [{uid: "alice", status: "offline"},              │
  │                {uid: "dave", status: "online"}]}              │
  │                                                                │
  │  Inter-server communication:                                   │
  │     Option A: Redis Pub/Sub (channel per WS server)           │
  │     Option B: gRPC between WS servers (lower latency)         │
  │     Option C: Shared message bus (Kafka — but adds latency)   │
  │     Recommended: Redis Pub/Sub ⭐ (already in stack)          │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │              Cassandra (Persistent Last Seen)                   │
  │                                                                │
  │  On going offline → write last_active to Cassandra:           │
  │    INSERT INTO last_seen (user_id, last_active, device)       │
  │    VALUES (uid, now(), 'mobile');                              │
  │                                                                │
  │  Query: when user opens chat with someone who is offline      │
  │    SELECT last_active FROM last_seen WHERE user_id = ?        │
  │    → "Last seen today at 2:30 PM"                             │
  └────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Heartbeat + TTL — Why Not WebSocket State?

```
WebSocket connection as presence:
  Brief network glitch → WS disconnects → reconnects in 2s
  → User flickers "offline" → "online" → terrible UX

Heartbeat + TTL ⭐:
  Client: heartbeat every 30s
  Server: SET presence:{uid} "online" EX 60
  No heartbeat for 60s → key expires → user offline
  
  ✓ Tolerates brief disconnects (still "online" up to 60s)
  ✓ Force-kill app → TTL handles cleanup automatically
  ✓ No cron jobs needed

Multi-device:
  presence:u1:mobile  → TTL 120s (battery-friendly)
  presence:u1:desktop → TTL 60s
  Online if ANY device key exists. Offline only when ALL expire.
```

#### Fan-Out: Avoiding 1B Updates/sec

```
Naive: 2M changes/sec × 500 friends = 1B pushes/sec → IMPOSSIBLE

Optimizations:
  Layer 1: Only notify ONLINE friends → ~100 → 200M/sec
  Layer 2: Only friends with chat/list OPEN → ~15 → 30M/sec ⭐
  Layer 3: Batch every 5s: {online: [u1], offline: [u2]}
  Layer 4: Pull on open + push while open + unsubscribe on navigate away

This is WhatsApp/Instagram's actual approach.
```

#### Last Seen Persistence

```
Redis key expires → last_active timestamp GONE

Solution: On going offline → persist to Cassandra
  CREATE TABLE last_seen (
    user_id UUID PRIMARY KEY, last_active TIMESTAMP, device TEXT
  );

Query:
  Redis key exists → "Online" (or "Away")
  Redis key absent → Cassandra lookup → "Last seen at {time}"
```

---

## 5. APIs

### Heartbeat (WebSocket)
```json
{"type": "heartbeat", "device": "mobile"}
→ {"type": "heartbeat_ack"}
```

### Query Presence (Batch)
```http
POST /api/v1/presence/batch
{ "user_ids": ["u1", "u2", "u3"] }
→ { "u1": {"status": "online"}, "u2": {"status": "offline", "last_seen": "..."} }
```

### Subscribe to Updates (WebSocket)
```json
{"type": "subscribe_presence", "user_ids": ["u1", "u2"]}
// Server pushes: {"type": "presence_update", "user_id": "u1", "status": "offline"}
```

---

## 6. Data Model

### Redis
```
presence:{uid}:{device}  → "online"|"away" (TTL: 60s mobile:120s)
presence:{uid}           → Hash {status, last_active} (TTL: 60s)
presence_subs:{uid}      → SET of subscriber user_ids
```

### Cassandra
```sql
CREATE TABLE last_seen (user_id UUID PRIMARY KEY, last_active TIMESTAMP, device TEXT);
```

---

## 7. Fault Tolerance & Race Conditions

| Concern | Solution |
|---|---|
| **WS crash** | Client reconnects + heartbeat → status recovers in < 60s |
| **Redis down** | Users show "unknown" status; degrade gracefully |
| **Force-kill app** | No OFFLINE sent → TTL expires in 60s → correct |
| **Multi-device conflict** | ANY device online → user online |
| **Privacy** | Check settings before returning presence |

### Key Race: Multi-Device Offline Detection
```
Phone goes offline. Desktop still on.
presence:u1:mobile expires → but presence:u1:desktop still alive
Must check ALL device keys before declaring offline.
Use SADD user_devices:{uid} to track active devices.

Implementation (Lua script for atomicity):
  local mobile = redis.call('EXISTS', 'presence:' .. uid .. ':mobile')
  local desktop = redis.call('EXISTS', 'presence:' .. uid .. ':desktop')
  local web = redis.call('EXISTS', 'presence:' .. uid .. ':web')
  if mobile + desktop + web == 0 then
    redis.call('DEL', 'presence_agg:' .. uid)
    return 'offline'
  end
  return 'online'

This runs atomically — no race between checking devices.
```

### Race: Heartbeat Arrives After TTL Expires

```
Mobile heartbeat interval: 60s. TTL: 120s.

T=0:    Heartbeat → SET presence:u1:mobile "online" EX 120
T=60:   Heartbeat → refresh TTL
T=120:  Network hiccup → heartbeat delayed
T=121:  TTL expires → user appears offline to friends!
T=125:  Delayed heartbeat arrives → user back "online"

Result: 4-second false offline blip. Friends see flicker.

Mitigation:
  1. TTL = 2.5× heartbeat interval (60s heartbeat → 150s TTL)
     Allows missing ONE heartbeat without going offline
  2. Grace period: Don't broadcast "offline" immediately on TTL expiry
     Wait 10 seconds → if heartbeat arrives → cancel the broadcast
     If no heartbeat → THEN broadcast offline to subscribers
```

---

## 8. Deep Dive: Engineering Trade-offs

### Typing Indicator — The Ephemeral Signal

```
"Alice is typing..." shown in chat conversation with Bob

Implementation:
  Client-side: On each keystroke (throttled to 1 event per 3 seconds):
    Send via WebSocket: {type: "typing", conversation_id: "conv-123"}
  
  Server:
    SET typing:{conv-123}:{alice} "1" EX 5  (auto-expires in 5 seconds)
    Route to Bob's WS server → push to Bob's WebSocket
  
  Client (Bob's app):
    On receiving typing event → show "Alice is typing..."
    If no new typing event in 5 seconds → hide indicator
    (Server TTL + client-side timeout = double safety)

Why NOT persist typing state?
  Typing is ultra-ephemeral (3-5 seconds relevant)
  No need to store in DB or even Kafka
  Direct WS-to-WS routing via Redis Pub/Sub is sufficient
  If message is lost → user just doesn't see indicator for 3 seconds → no harm

Scaling concern:
  In a group chat with 500 members, 10 typing simultaneously:
  10 typing × 500 recipients × 1 event/3s = 1666 messages/sec for ONE group
  
  Optimization: Only send typing indicator to members with chat window OPEN
  Typically 5-10 members actively viewing → 10 × 10 = 100 messages/sec → trivial
```

### Privacy Model — Fine-Grained Control

```
User privacy settings stored in Redis:
  HSET privacy:{uid} presence "everyone"|"friends"|"nobody"|"custom"
  HSET privacy:{uid} last_seen "everyone"|"friends"|"nobody"
  HSET privacy:{uid} typing "everyone"|"friends"

Custom blocklist:
  SADD presence_hidden:{uid} {blocked_user_id_1} {blocked_user_id_2}

Query flow:
  Bob requests Alice's presence:
  1. Check privacy:{alice} presence setting
  2. If "friends" → verify Bob is in Alice's friend list
  3. If "custom" → check SISMEMBER presence_hidden:{alice} {bob}
  4. If allowed → return real status
  5. If blocked → return {"status": "unavailable"} (hide completely)

Important: Privacy check happens at QUERY time AND PUSH time
  Don't push Alice's status to Bob if Bob is on Alice's hidden list
  Check at fan-out: filter subscribers through privacy settings
```

### WebSocket Connection Routing — Finding Where a User Is Connected

```
Problem: 200M users across 2000 WS servers.
  To push a presence update to Bob → which of 2000 servers has Bob's connection?

Solution: Connection registry in Redis
  On WS connect:
    HSET ws_location:{uid} server "ws-server-42" device "mobile"
    EXPIRE ws_location:{uid} 300  (5 min, refreshed with heartbeats)
  
  On WS disconnect:
    HDEL ws_location:{uid}
  
  To route a message to Bob:
    server = HGET ws_location:{bob} server → "ws-server-42"
    Send to ws-server-42 via Redis Pub/Sub channel: ws_inbox:ws-server-42
    ws-server-42 receives → looks up local connection map → pushes to Bob's WS

Multi-device:
  Bob on mobile (ws-server-12) AND desktop (ws-server-42):
  ws_location:{bob}:mobile  → "ws-server-12"
  ws_location:{bob}:desktop → "ws-server-42"
  Route message to BOTH servers → both devices get the update
```

### Redis vs Custom In-Memory vs XMPP

| Approach | Latency | Scale | Complexity | Best For |
|---|---|---|---|---|
| **Redis + TTL** ⭐ | < 1 ms | 200M+ with cluster | Low | Most apps |
| **Custom gossip protocol** | < 0.1 ms | Billions | Very high | WhatsApp (2B users) |
| **XMPP (ejabberd)** | 1-5 ms | Millions | Moderate | Small-medium IM |
| **Firebase Presence** | 5-50 ms | Millions | Zero (managed) | Prototypes, small apps |

Redis for most production systems. Custom only at extreme WhatsApp/WeChat scale.

### Status Display Logic — Client-Side Formatting

```
Given: status field + last_active timestamp

Rendering rules:
  status == "online"  → "Online" (green dot)
  status == "away"    → "Away" (yellow dot)
  status == "offline" → format_last_seen(last_active):
    < 1 min ago    → "Last seen just now"
    < 60 min ago   → "Last seen {N} minutes ago"
    Today          → "Last seen today at 2:30 PM"
    Yesterday      → "Last seen yesterday at 10:15 PM"
    < 7 days       → "Last seen Monday at 3:00 PM"
    > 7 days       → "Last seen Mar 7"
    No data        → "Last seen a long time ago"

Why client-side formatting?
  Server returns raw timestamp → client formats based on locale and timezone
  Avoids server computing relative times for millions of queries
```
