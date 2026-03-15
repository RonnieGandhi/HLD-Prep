# 90. Design a Multiplayer Game Backend

---

## 1. Functional Requirements (FR)

- Real-time game state synchronization across all players (tick-based: 20-60 ticks/sec)
- Matchmaking: group players of similar skill into game sessions
- Lobby system: create/join rooms, invite friends, ready-up
- Player input processing with server-authoritative validation (anti-cheat)
- Game state persistence: save progress, inventory, stats
- Leaderboard: global and seasonal rankings
- In-game chat (text and voice)
- Replay system: record and playback full game sessions
- Spectator mode: watch live games with slight delay
- Cross-platform play (PC, console, mobile)

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: < 50ms round-trip for player actions (ideally < 20ms)
- **Tick Rate**: Server processes game state at 20-64 times per second
- **High Availability**: 99.9% — game server crash = game lost for all players
- **Consistency**: Server is authoritative — all clients see same game state
- **Scalability**: 10M+ concurrent players, 500K+ concurrent game sessions
- **Anti-Cheat**: Server validates all actions; client is untrusted

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Concurrent players | | 10M |
| Concurrent game sessions | 10M / 20 players avg | 500K |
| Tick rate | | 30 ticks/sec |
| State updates / sec per session | 30 × 20 players | 600 |
| Total state updates / sec | 500K × 600 | 300M |
| Network per player | up + down | ~50 KB/sec up + ~100 KB/sec down |
| Total bandwidth | 10M × 150 KB/sec | 1.5 TB/sec |
| Game servers needed | 500K / 10 sessions per server | ~50K |
| Match results / sec | 500K / 20 min avg game | ~420/sec |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GAME CLIENTS                                  │
│  PC / Console / Mobile                                                │
│  ┌──────────────────────────────────────────┐                         │
│  │  Client Game Engine                       │                         │
│  │  - Render game world                      │                         │
│  │  - Capture player input                   │                         │
│  │  - Client-side prediction                 │                         │
│  │  - Interpolation for remote players       │                         │
│  │  - Send inputs via UDP                    │                         │
│  │  - Receive state via UDP                  │                         │
│  └──────────────────────┬────────────────────┘                         │
│                         │ UDP (game) + TCP/WS (meta)                  │
└─────────────────────────│──────────────────────────────────────────────┘
                          │
┌─────────────────────────│──────────────────────────────────────────────┐
│                PLATFORM SERVICES LAYER                                 │
│                         │                                              │
│  ┌──────────────────────▼──────────────────────────────┐               │
│  │  Gateway / Router                                    │               │
│  │  - JWT auth, request routing, DDoS protection        │               │
│  └──────────────────────┬──────────────────────────────┘               │
│                         │                                              │
│  ┌──────────┐  ┌────────▼───┐  ┌────────────┐  ┌──────────────┐      │
│  │Matchmaker│  │ Lobby      │  │ Profile &  │  │ Leaderboard  │      │
│  │ Service  │  │ Service    │  │ Inventory  │  │ Service      │      │
│  │          │  │            │  │ Service    │  │              │      │
│  │- Glicko-2│  │- Room CRUD │  │            │  │- Redis Sorted│      │
│  │  rating  │  │- Ready up  │  │- PostgreSQL│  │  Sets        │      │
│  │- Queue   │  │- Invite    │  │  player DB │  │- ZREVRANK    │      │
│  │  by skill│  │  friends   │  │- Redis     │  │  for rank    │      │
│  │- Widen   │  │- Transfer  │  │  session   │  │- Per-season  │      │
│  │  search  │  │  to game   │  │  cache     │  │  leaderboard │      │
│  │  over    │  │  server    │  │            │  │              │      │
│  │  time    │  │            │  │            │  │              │      │
│  └────┬─────┘  └────────────┘  └────────────┘  └──────────────┘      │
│       │                                                                │
│       │  Matched → allocate game server                               │
│       │                                                                │
└───────│────────────────────────────────────────────────────────────────┘
        │
┌───────▼────────────────────────────────────────────────────────────────┐
│              GAME SERVER LAYER (Stateful, Per-Session)                  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  Dedicated Game Server (one per game session)                │      │
│  │                                                              │      │
│  │  Game Loop (30 ticks/sec = every 33ms):                      │      │
│  │  ┌───────────────────────────────────────────────────────┐   │      │
│  │  │  tick() {                                             │   │      │
│  │  │    1. Receive all player inputs since last tick        │   │      │
│  │  │    2. Validate inputs (anti-cheat):                    │   │      │
│  │  │       - Speed hack? (moved too far in one tick)       │   │      │
│  │  │       - Wall hack? (shot through solid wall)          │   │      │
│  │  │       - Aimbot? (statistical accuracy analysis)       │   │      │
│  │  │    3. Simulate game physics (movement, collisions)    │   │      │
│  │  │    4. Resolve actions (damage, pickups, scores)       │   │      │
│  │  │    5. Update authoritative game state                 │   │      │
│  │  │    6. Send delta-compressed state to all clients      │   │      │
│  │  │    7. Record tick for replay system                   │   │      │
│  │  │  }                                                    │   │      │
│  │  └─��─────────────────────────────────────────────────────┘   │      │
│  │                                                              │      │
│  │  Networking:                                                  │      │
│  │  - Input: UDP from clients (unreliable, fast)                │      │
│  │  - State: UDP to clients (delta compressed)                  │      │
│  │  - Reliable events: TCP/WS (kills, scores, chat)            │      │
│  │                                                              │      │
│  │  Anti-Cheat (server-authoritative):                           │      │
│  │  - Client sends "I pressed W" NOT "I'm at position X"       │      │
│  │  - Server simulates what pressing W does                     │      │
│  │  - Mismatch → reject and correct client                     │      │
│  │  - ✅ Prevents: speed hacks, teleporting, infinite health   │      │
│  │  - ❌ Harder: wallhack (only send visible player positions) │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  Orchestration: Agones (K8s-native game server orchestrator)          │
│  - Pool of "ready" servers (pre-warmed)                               │
│  - Matchmaker requests → Agones allocates → returns address          │
│  - Game ends → recycle server (terminate pod, start fresh)            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│              DATA & ASYNC LAYER                                        │
│                                                                        │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │ Kafka        │  │ PostgreSQL   │  │ Redis        │  │ S3        │  │
│  │ - Match      │  │ - Players    │  │ - Sessions   │  │ - Replays │  │
│  │   results    │  │ - Match      │  │ - Matchmaking│  │ - ~5 MB   │  │
│  │ - Events     │  │   history    │  │   queues     │  │   per 20  │  │
│  │ - Anti-cheat │  │ - Inventory  │  │ - Leaderboard│  │   min game│  │
│  │   reports    │  │ - Purchases  │  │ - Presence   │  │           │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Netcode: Client-Side Prediction

```
Problem: 50ms round-trip → 50ms delay between pressing "move" and seeing movement

Solution: Client-Side Prediction + Server Reconciliation
  1. Player presses "move forward"
  2. Client: immediately moves character locally (feels responsive)
  3. Client: sends input to server (timestamp + input)
  4. Server: simulates movement, sends authoritative position back
  5. Client: receives server position at time T
     - If matches prediction → great, no correction needed
     - If different → rewind to server state at T, replay all inputs since T
     - "Reconciliation" produces smooth correction without teleporting

  This is why multiplayer games feel responsive despite network latency
```

#### Netcode: Interpolation (for other players)

```
Problem: Server sends state at tick 100 and 101 (33ms apart at 30 ticks/sec)
  Between ticks, other players would "jump" from position to position

Solution: Render other players ~50ms in the past (one tick behind)
  At real time tick 101, render other players at tick 100
  Smooth interpolation between tick 99 and 100 positions
  Result: smooth movement despite discrete server ticks
  Cost: other players are always ~50ms "in the past"
```

#### Netcode: Lag Compensation (for shooting)

```
Problem: Player aims and shoots at an enemy on their screen
  But on the server, the enemy is 50ms ahead of where player sees them

Solution: Server-side rewind ("favor the shooter")
  1. Player shoots at tick 100, but sees world at tick 99 (interpolation)
  2. Server receives shot → rewinds game state to tick 99
  3. Checks if shot would hit at tick 99 → YES → register hit
  4. "You shot where the enemy WAS on your screen" = feels fair

  Trade-off: Shooter's experience is prioritized
  Side effect: Victim occasionally gets shot "around corners"
  (enemy peeked, saw you, shot — but you saw them duck behind wall)
```

#### Matchmaking: Glicko-2 Algorithm

```
Rating system (improvement over Elo):
  - Rating: 1500 (default) ± rating deviation (uncertainty)
  - New players: high deviation → bigger rating changes per game
  - Veteran players: low deviation → small rating changes
  - Volatility parameter: how erratic the player's performance is

Queue process:
  1. Player enters queue with (rating, deviation, region)
  2. Initially: search for exact match (±50 rating, same region)
  3. Over time: widen search (±100, ±200, then cross-region)
  4. Quality score: 1 - (avg_rating_diff / max_diff) × (1 - wait_penalty)
  5. Accept match when quality > threshold
  
Party matching:
  - Weighted average: highest-rated member weighted 1.5×
  - Prefer party vs party (avoid party vs solo players)
```

#### State Compression (Bandwidth Optimization)

```
Full state snapshot: 20 players × 100 bytes = 2 KB per tick × 30 ticks = 60 KB/sec

Delta compression: only send what changed since last acknowledged tick
  Player didn't move? Skip (0 bytes)
  Position changed? Send delta (2 bytes vs 12 bytes for full vector)
  Typical delta: 200 bytes vs 2 KB full state (10× reduction)

Priority-based updates:
  Nearby players: full update every tick (30/sec)
  Far players: reduced rate (10/sec)
  Off-screen: minimal updates (2/sec)
  Result: 5-10× bandwidth reduction for battle royale (100 players, huge map)
```

---

## 5. APIs

```
# Matchmaking
POST /api/matchmaking/queue       → Enter queue {rating, region, party_ids}
DELETE /api/matchmaking/queue     → Leave queue
GET /api/matchmaking/status       → Queue status, estimated wait time

# Lobby
POST /api/lobbies                 → Create lobby room
POST /api/lobbies/{id}/join       → Join lobby
POST /api/lobbies/{id}/ready      → Ready up
POST /api/lobbies/{id}/invite     → Invite friend
POST /api/lobbies/{id}/start      → Start game (host only)

# Game connection (returned after match found)
{ "game_server": "gs-us-east-42.game.com", "port": 27015, "token": "jwt..." }
→ Client connects via UDP to game server with token

# Profile & Stats
GET /api/players/{id}/stats       → Win rate, K/D, matches played
GET /api/players/{id}/inventory   → Cosmetics, unlocks

# Leaderboard
GET /api/leaderboard?season=s12&page=1    → Top players
GET /api/leaderboard/rank?player_id=123   → My rank + neighbors
```

---

## 6. Data Models

### PostgreSQL (Persistent Player Data)

**Why PostgreSQL?** ACID for player inventory/purchases, relational for match history joins.

```sql
CREATE TABLE players (
    player_id    UUID PRIMARY KEY,
    username     TEXT UNIQUE,
    rating       FLOAT DEFAULT 1500.0,
    rating_dev   FLOAT DEFAULT 350.0,
    volatility   FLOAT DEFAULT 0.06,
    matches_played INT DEFAULT 0,
    wins         INT DEFAULT 0,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE match_history (
    match_id     UUID,
    player_id    UUID,
    team         INT,
    result       TEXT CHECK (result IN ('win','loss','draw')),
    kills        INT DEFAULT 0,
    deaths       INT DEFAULT 0,
    score        INT DEFAULT 0,
    rating_change FLOAT,
    played_at    TIMESTAMPTZ,
    replay_url   TEXT,
    PRIMARY KEY (match_id, player_id)
);
CREATE INDEX idx_match_player ON match_history(player_id, played_at DESC);
```

### Redis

```
# Matchmaking queue (sorted set by rating for range matching)
ZADD mm:queue:{region} {rating} {player_id:metadata_json}

# Active game sessions
HSET game:session:{game_id} server "gs-42" players "[p1,p2,...]" started_at "..."

# Leaderboard (sorted set by rating)
ZADD leaderboard:season:s12 {rating} {player_id}
ZREVRANK leaderboard:season:s12 {player_id}  → player's rank
ZREVRANGE leaderboard:season:s12 0 99 WITHSCORES  → top 100

# Player presence
SET presence:{player_id} "in_game:gs-42" EX 300
```

### S3 (Replay Storage)

```
Path: s3://replays/{match_id}.bin
Format: compressed tick log (protobuf per tick, gzipped)
Size: ~5 MB per 20-min game (20 players × 30 ticks/sec × 33ms × 1200s)
Indexed by: match_id in PostgreSQL match_history.replay_url
```

---

## 7. Fault Tolerance

| Technique | Application |
|---|---|
| Agones orchestration | Pre-warmed server pool; auto-replace crashed pods |
| Health checks | Gateway monitors game servers; redirect on failure |
| UDP resilience | Lost packets → interpolation fills gaps (no TCP retransmit delay) |
| Kafka RF=3 | Match results, events survive broker failure |
| Redis Cluster | Session state, leaderboard survives node failure |

### Problem-Specific

#### Game Server Crash Recovery

```
Short matches (< 10 min): Game lost → don't count for ranking → notify players
Long/ranked matches:
  1. Checkpoint game state every 30s to Redis (serialized state)
  2. On crash → spin up new server → load checkpoint → players reconnect
  3. Resume from ~30 seconds ago (acceptable for most games)

Tournament matches:
  Shadow server receives all inputs in real-time
  On primary crash → promote shadow → seamless failover (<1s interruption)
  Cost: 2× game servers, only for high-stakes games
```

#### DDoS Protection

```
Problem: DDoS game server IP → ruins game for all players

Mitigations:
  1. Don't expose game server IPs directly to clients
     Client → relay proxy → proxy forwards to game server
     Game server IP changes per match
  2. Per-player rate limiting at proxy layer
  3. Validate player token in first UDP packet (reject unknown sources)
  4. Anycast network absorbs volumetric attacks
  5. Auto-migration: detect DDoS → move game to new server IP → notify clients
```

#### Anti-Cheat Deep Dive

```
Server-authoritative model prevents:
  ✅ Speed hacks: server validates max distance per tick
  ✅ Teleporting: server validates position continuity
  ✅ Infinite health: server tracks HP, not client
  ✅ Item duplication: server manages all inventory
  ✅ Wall shooting: server validates line-of-sight

Client-side cheats (harder to prevent):
  ❌ Wallhack: client needs enemy positions for interpolation
     Mitigation: only send positions of players within view/hearing range
  ❌ Aimbot: client auto-aims at heads
     Mitigation: statistical analysis (inhuman accuracy → flag → ban)
  ❌ ESP: highlighting enemies through walls
     Same mitigation as wallhack (don't send hidden player data)
```

---

## 8. Additional Considerations

### Game Server Orchestration with Agones
```
Agones (Kubernetes-native game server orchestrator):
  1. Maintain pool of "Ready" game servers (pre-warmed containers)
  2. Matchmaker calls Agones API: "allocate server in us-east"
  3. Agones marks Ready → Allocated → returns IP:port
  4. Players connect → game begins
  5. Game ends → server transitions to Shutdown → pod recycled

Auto-scaling:
  Monitor Ready server pool size → if < threshold → scale up deployment
  Pre-warm during peak hours (Friday evening, weekends)
```

### Replay System
```
Full tick log: record all player inputs + server state per tick
Storage: S3, ~5 MB per 20-min game (compressed protobuf)
Playback: load tick log → re-simulate in client → watch from any perspective
Spectator: subscribe to live tick stream with 15s delay (prevents ghosting)
```

### Why UDP Over TCP for Game State
```
TCP: Guarantees delivery + ordering → but if packet lost, everything waits (head-of-line blocking)
UDP: No guarantees → if packet lost, skip it, use next one
  For game state: getting tick 101 is more important than retransmitting tick 99
  Lost tick → interpolation fills the gap → player barely notices
  TCP retransmit delay (100ms+) → player sees freeze → terrible experience

Use TCP for: chat messages, inventory changes, kill feed (reliable events)
Use UDP for: position updates, state snapshots (real-time, loss-tolerant)
```

---

## 9. Deep Dive: Engineering Trade-offs

### The Game Loop — One Tick End-to-End (33ms Budget)

```
Server tick rate: 30 ticks/sec → 33ms per tick
20 players in a match, each sending ~3 inputs per tick

Tick 1042 processing:

  T=0ms: RECEIVE phase
    Read all pending UDP packets from player sockets
    Buffer: [(player_3, "move_forward, shoot"), (player_7, "jump"), ...]
    ~60 inputs accumulated (20 players × 3 avg)
    Late inputs (from tick 1041): process anyway (grace period = 1 tick)
    Very late inputs (tick 1039): DROP (too old, would break simulation)

  T=2ms: VALIDATE phase
    For each input:
      player_3 says "move_forward":
        Current position: (100, 50). Movement speed: 5 units/tick.
        New position: (105, 50). Valid? Check:
          ✓ Distance ≤ max_speed × dt (no speed hack)
          ✓ No collision with walls (server-side physics check)
          ✓ Player is alive (dead players can't move)
        → Accept. Update server state: player_3.pos = (105, 50)
      
      player_3 says "shoot":
        Gun cooldown: last shot was tick 1040 → 2 ticks ago → cooldown = 3 ticks
        → REJECT (cooldown not expired). Log: potential cheat attempt.
      
      player_7 says "jump":
        Is player_7 on ground? Yes → Accept. Apply physics: vertical velocity.

  T=8ms: SIMULATE phase
    Run physics simulation for all entities:
      Projectiles: advance positions, check hit detection
      Players: apply gravity, resolve collisions
      Explosions: calculate area damage
      Power-ups: check pickup proximity
    
    Hit detection: player_12's bullet → raycast → hits player_5 at (200, 80)
      player_5.hp -= 25. If hp <= 0 → kill event.

  T=15ms: SNAPSHOT phase
    Build game state snapshot for tick 1042:
      { tick: 1042, players: [{id:3, pos:(105,50), hp:100}, ...],
        projectiles: [...], events: ["player_5 hit by player_12"] }

  T=18ms: BROADCAST phase
    For each player, build personalized snapshot:
      Only include entities within their "area of interest" (anti-wallhack)
      Compress with delta encoding: only send CHANGES from tick 1041
        player_3: pos changed (100,50) → (105,50) → send delta (+5, 0)
        player_8: no change → skip (saves bandwidth)
    
    Serialize to protobuf → send UDP packet to each player
    ~20 packets × ~200 bytes each = 4 KB total outbound

  T=22ms: RECORD phase (async)
    Append tick 1042 inputs + state to replay buffer (in-memory ring buffer)
    Every 30 seconds: flush buffer to S3 (compressed)

  T=23ms: IDLE (10ms remaining until tick 1043)
    Wait or process admin tasks (kick, ban, settings change)

Total: ~23ms of 33ms budget used. 10ms headroom for spikes.
If a tick exceeds 33ms → game stutters → clients notice immediately.
```

### Client-Side Prediction + Server Reconciliation

```
The problem:
  Player presses "move right" at T=0
  Input sent to server (50ms network latency)
  Server processes at T=50ms, broadcasts result at T=55ms
  Client receives confirmation at T=105ms
  
  If client waits for server: 105ms delay between keypress and movement
  At 30 FPS, that's 3+ frames of input lag → feels sluggish, unplayable

Solution: Client-Side Prediction

  T=0ms:  Player presses "move right"
          Client IMMEDIATELY applies the input locally (prediction)
          Player sees character move right → feels instant
          Client stores: { input_seq: 42, input: "move_right", predicted_pos: (105, 50) }
          Client sends input to server: { seq: 42, input: "move_right" }

  T=50ms: Server receives input #42, validates, applies
          Server broadcasts: { tick: X, player_3: { pos: (105, 50), last_processed_input: 42 } }

  T=105ms: Client receives server state
           Check: server says pos=(105,50), my prediction was (105,50)
           MATCH! → no correction needed → smooth experience

  When prediction is WRONG (server rejected):
    Client predicted: pos = (105, 50) — moved right
    Server says: pos = (100, 50) — movement rejected (hit a wall server-side)
    
    Client must reconcile:
      1. Snap to server position: (100, 50) — abrupt, visually jarring
      2. OR: interpolate from current rendered position to server position over 100ms
         (smooth correction — player barely notices)
    
    Then: re-apply all inputs the server hasn't processed yet
      Client has inputs [42, 43, 44] pending
      Server processed up to 42 → re-simulate 43, 44 on top of server state
      New predicted position reflects server truth + unprocessed inputs

  This is why server-authoritative + client prediction gives BOTH:
    Responsiveness (instant feedback from prediction)
    Security (server validates everything, rejects cheats)
```

### Entity Interpolation — Making Other Players Move Smoothly

```
Problem:
  Server sends position updates at 30 ticks/sec (every 33ms)
  Client renders at 60 FPS (every 16ms)
  Between server ticks, what position do we render other players at?

Without interpolation:
  Tick 100: player_5 at (100, 50)
  Tick 101: player_5 at (105, 50)
  
  Client renders: (100,50) for 33ms → jumps to (105,50) → choppy movement

With interpolation:
  Client renders BETWEEN two known server states (100ms behind real-time):
  
  T=0ms (render):    show player_5 at (100, 50)     — tick 100
  T=11ms (render):   show player_5 at (101.7, 50)   — 1/3 between tick 100→101
  T=22ms (render):   show player_5 at (103.3, 50)   — 2/3 between tick 100→101
  T=33ms (render):   show player_5 at (105, 50)     — tick 101 (exact)
  
  Movement is smooth because we're always drawing between TWO known positions
  Trade-off: other players rendered ~100ms behind real-time (interpolation buffer)
  For a shooter: 100ms means aiming where player WAS, not where they ARE
  
Extrapolation (when next tick hasn't arrived yet):
  If tick 102 is LATE (packet loss), predict where player will be:
  Velocity = (105-100) / 33ms → extrapolate: (110, 50)
  Risk: if player changed direction, extrapolation is wrong → snap correction
  Extrapolation used only as fallback; interpolation is always preferred

Why render other players 100ms behind?
  Need TWO data points to interpolate → must buffer one tick
  Buffer = 2-3 ticks × 33ms = 66-100ms
  This is invisible to humans (reaction time is 200ms+)
  But for competitive games: 100ms = difference between hit and miss
  → Competitive games use tick rates of 64-128 (Valorant: 128-tick = 7.8ms)
```

### Matchmaking — How Skill-Based Matching Works

```
ELO / TrueSkill rating system:
  Each player has: μ (mean skill) and σ (uncertainty)
  New player: μ=1500, σ=350 (high uncertainty)
  After 50 games: μ=1720, σ=50 (well-calibrated)

Matching algorithm:
  1. Player joins queue → placed in Redis sorted set:
     ZADD mm:queue:us-east {μ} {player_id}
  
  2. Matchmaker (runs every 2 seconds):
     For each player in queue:
       Find players within μ ± search_radius (start: ±100)
       If enough found (e.g., 10 for a 5v5) → create match
       If not → widen search_radius by 50 → retry next cycle
     
     search_radius increases with wait time:
       0-30s: ±100 (tight, fair matches)
       30-60s: ±200 (wider, acceptable)
       60-120s: ±400 (any match better than no match)
       120s+: ±1000 (match anyone available)
  
  3. Match creation:
     Balance teams: assign players to Team A vs Team B
       Minimize: |avg_μ(Team A) - avg_μ(Team B)|
       Constraint: max individual μ difference within team < 300
     
     Select game server:
       Choose server in region closest to average player location
       Allocate via Agones API → get server IP:port
     
     Notify players: send match details via WebSocket
       { server: "gs-42.us-east.game.com:7777", team: "A", match_id: "..." }
  
  Why not just sort by rating and pair adjacent?
    Queue is dynamic: players join/leave constantly
    Need to consider: region (latency), party groups, role preferences
    Fairness metric: track post-match win rate per rating bracket
      If 1500-rated players win 60% against 1500-rated → rating is miscalibrated
```

