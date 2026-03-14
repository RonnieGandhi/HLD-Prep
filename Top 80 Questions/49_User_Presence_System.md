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
┌────────────────────────────────────────────────────────────────┐
│                   Client (Mobile/Desktop)                      │
│  • App open → "ONLINE"   • Every 30s → heartbeat              │
│  • App close → "OFFLINE" • No heartbeat 60s → server expires  │
│  • 5 min idle → "AWAY"   • Receive updates via WebSocket      │
└───────────────────────┬────────────────────────────────────────┘
                        │ WebSocket
                 ┌──────▼──────┐
                 │  WebSocket  │ 200M connections across 2000 servers
                 │  Gateway    │
                 └──────┬──────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
  ┌──────▼─────┐ ┌──────▼─────┐ ┌─────▼──────────┐
  │ Heartbeat  │ │ Presence   │ │ Subscription   │
  │ Processor  │ │ Query Svc  │ │ Manager        │
  │            │ │            │ │                │
  │ 6.7M/sec   │ │ "Is Alice  │ │ Only track     │
  │ SET w/ TTL │ │  online?"  │ │ friends with   │
  │ in Redis   │ │ Read Redis │ │ chat/list open │
  └──────┬─────┘ └──────┬─────┘ └────────┬────────┘
         │              │                │
         └──────┬───────┘                │
                ▼                        ▼
         ┌───────────────────────────────────────┐
         │              Redis Cluster             │
         │                                        │
         │  presence:{uid}:{device}               │
         │  = "online"|"away"  TTL: 60s           │
         │                                        │
         │  presence:{uid}                        │
         │  = Hash {status, last_active, device}  │
         │  TTL: 60s                              │
         │                                        │
         │  Pub/Sub: presence_changes:{uid}       │
         └──────────────────┬────────────────────┘
                            │ on status change
         ┌──────────────────▼────────────────────┐
         │         Fan-Out Service                │
         │                                        │
         │  1. Get friend list                    │
         │  2. Filter: only ONLINE friends        │
         │  3. Filter: only friends with chat/    │
         │     contact list open (subscribed)     │
         │     → typically ~15 friends             │
         │  4. Push via their WS servers          │
         │  5. Batch updates every 5s             │
         └────────────────────────────────────────┘
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
```

---

## 8. Deep Dive: Engineering Trade-offs

| Approach | Latency | Scale | Complexity |
|---|---|---|---|
| **Redis + TTL** ⭐ | < 1 ms | 200M+ | Low |
| **Custom gossip** | < 0.1 ms | Billions | Very high |
| **XMPP** | 1-5 ms | Millions | Moderate |

Redis for most; custom for WhatsApp-scale (2B users).
