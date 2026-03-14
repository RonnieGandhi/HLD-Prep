# 45. Design Tinder (Matching System)

---

## 1. Functional Requirements (FR)

- **Profile creation**: Photos, bio, age, gender, preferences (age range, distance, gender)
- **Discovery (Swiping)**: Show nearby profiles one at a time; user swipes right (like) or left (pass)
- **Matching**: When both users swipe right on each other → MATCH → enable chat
- **Geo-based discovery**: Only show users within configured radius (e.g., 50 km)
- **Chat**: Matched users can text message each other
- **Super Like**: Special like that notifies the other user immediately
- **Undo**: Undo last swipe (premium feature)
- **Boost**: Temporarily increase visibility (premium)
- **Block/Report**: Safety features

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Swipe deck loads in < 200 ms
- **Geo-Accuracy**: Distance calculation accurate within 1 km
- **Consistency**: Mutual match must be EXACTLY consistent (no one-sided matches)
- **Scalability**: 75M+ MAU, 2B+ swipes/day
- **Privacy**: Location never exposed to other users (only distance shown)
- **Freshness**: New users/profile updates reflected in discovery within minutes

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 25M |
| Swipes / day | 2B |
| Swipes / sec | ~23K (peak 100K) |
| Matches / day | 30M |
| Avg profiles in deck | 200/session |
| Profile size | 5 KB (metadata) + 5 MB (photos) |
| Geo-query fan-out | ~10K profiles per 50 km radius in dense city |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Client (Mobile App)                         │
│                                                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ Swipe Deck │  │ Match      │  │ Chat       │  │ Profile      │  │
│  │ (cards UI) │  │ Animation  │  │ (WebSocket)│  │ Editor       │  │
│  │            │  │ "It's a    │  │            │  │              │  │
│  │ Pre-fetch  │  │  Match!"   │  │            │  │              │  │
│  │ next 20    │  │            │  │            │  │              │  │
│  └─────┬──────┘  └────────────┘  └─────┬──────┘  └──────┬───────┘  │
└────────┼───────────────────────────────┼──────────────────┼──────────┘
         │                               │                  │
         ▼                               ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         API Gateway                                  │
└──────┬────────────────┬───────────────┬──────────────────────────────┘
       │                │               │
┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
│ Discovery   │  │ Match       │  │ Chat       │
│ Service     │  │ Service     │  │ Service    │
└──────┬──────┘  └──────┬──────┘  └────────────┘
       │                │
       │         ┌──────▼──────────────────────────────────┐
       │         │        Match Detection Engine            │
       │         │                                          │
       │         │  On swipe-right from User A on User B:   │
       │         │                                          │
       │         │  1. Record: SADD swiped_right:{A} {B}   │
       │         │  2. Check: SISMEMBER swiped_right:{B} {A}│
       │         │     → If B already swiped right on A:   │
       │         │       ★ MATCH! Insert into matches table│
       │         │       → Notify both A and B             │
       │         │       → Enable chat between A and B     │
       │         │     → If not: no match (yet)            │
       │         │                                          │
       │         │  Race condition protection:              │
       │         │  Use Redis transaction (MULTI/EXEC)      │
       │         │  or Lua script for atomic check+set      │
       │         └──────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────────┐
│                    Discovery Service (The Core)                      │
│                                                                       │
│  Input: User A requests next 20 profiles                             │
│                                                                       │
│  Pipeline:                                                           │
│  1. GEO QUERY: Find users within A's distance preference             │
│     → Geohash/H3 index on user locations                            │
│     → Redis GeoSet: GEORADIUS users:{geohash_prefix} lat lng 50 km  │
│     → Returns ~10K candidate user_ids                                │
│                                                                       │
│  2. FILTER: Remove incompatible profiles                             │
│     → Age range mismatch                                             │
│     → Gender preference mismatch                                     │
│     → Already swiped (Redis SET: swiped:{A})                        │
│     → Blocked users                                                  │
│     → Inactive users (last active > 7 days)                          │
│     → ~2K candidates remain                                          │
│                                                                       │
│  3. RANK: Score remaining candidates                                 │
│     → Elo/Desirability score (based on swipe-right rate received)   │
│     → Profile completeness (more photos = higher rank)               │
│     → Activity recency (recently active = higher rank)               │
│     → Mutual interest signals (shared interests/music)               │
│     → Boost status (paid boost = temporary rank increase)            │
│     → ML model: P(A swipes right on B)                              │
│                                                                       │
│  4. SELECT: Take top 200, shuffle slightly for variety               │
│     → Cache in Redis: deck:{A} = [user_ids]                         │
│     → Return first 20 to client                                     │
│                                                                       │
│  5. SUBSEQUENT REQUESTS: Pop next 20 from cached deck               │
│     → When deck empty → regenerate (step 1-4)                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                      Storage Architecture                            │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │   MySQL      │  │   Redis      │  │   S3 + CDN   │               │
│  │  (Vitess)    │  │   Cluster    │  │              │               │
│  │              │  │              │  │  Profile     │               │
│  │  users       │  │  Location    │  │  photos      │               │
│  │  matches     │  │  GeoSet      │  │              │               │
│  │  preferences │  │  swiped:{uid}│  │              │               │
│  │  reports     │  │  deck:{uid}  │  │              │               │
│  │              │  │  swiped_right│  │              │               │
│  │  Sharded by  │  │  :{uid} SET  │  │              │               │
│  │  user_id     │  │              │  │              │               │
│  └��─────────────┘  └──────────────┘  └──────────────┘               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. APIs

### Get Discovery Deck
```http
GET /api/v1/discovery?count=20
→ 200 OK
{ "profiles": [
    { "user_id": "u-uuid", "name": "Alice", "age": 28,
      "photos": ["url1", "url2"], "bio": "Love hiking...",
      "distance_km": 5.2, "common_interests": ["hiking", "photography"] },
    ...
]}
```

### Swipe
```http
POST /api/v1/swipes
{ "target_user_id": "u-uuid", "action": "right" }
→ 200 OK { "match": true, "match_id": "m-uuid" }  // or "match": false
```

### Get Matches
```http
GET /api/v1/matches?cursor={last_match_id}&limit=20
→ 200 OK { "matches": [{ "match_id": "m-uuid", "user": {...}, "matched_at": "..." }] }
```

---

## 6. Data Model

### MySQL — Core Data
```sql
CREATE TABLE users (
    user_id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    name            VARCHAR(100),
    birth_date      DATE,
    gender          ENUM('M','F','NB'),
    bio             TEXT,
    photo_urls      JSON,
    last_active     TIMESTAMP,
    latitude        DECIMAL(10,7),
    longitude       DECIMAL(10,7),
    elo_score       FLOAT DEFAULT 1000,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE preferences (
    user_id         BIGINT PRIMARY KEY,
    gender_pref     SET('M','F','NB'),
    age_min         TINYINT,
    age_max         TINYINT,
    distance_km     SMALLINT DEFAULT 50,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE matches (
    match_id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id_1       BIGINT,
    user_id_2       BIGINT,
    matched_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY (user_id_1, user_id_2),
    INDEX idx_user1 (user_id_1, matched_at DESC),
    INDEX idx_user2 (user_id_2, matched_at DESC)
);
```

### Redis — Location & Swipe State
```
# Geospatial index for proximity queries
Key:    users:geo
Type:   GeoSet
Ops:    GEOADD users:geo {lng} {lat} {user_id}
Query:  GEORADIUS users:geo {lng} {lat} 50 km COUNT 10000

# Swipe history (who has this user swiped on?)
Key:    swiped:{user_id}
Type:   SET (all swiped user_ids — left or right)
Purpose: Filter already-seen profiles from discovery

# Right-swipe tracking (for match detection)
Key:    swiped_right:{user_id}
Type:   SET (user_ids this user swiped right on)

# Pre-computed discovery deck
Key:    deck:{user_id}
Type:   LIST (ordered user_ids for next swipe session)
TTL:    1 hour
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Match consistency** | Redis Lua script ensures atomic check-and-set for mutual match |
| **Location staleness** | Update location only when app is in foreground; TTL of 24 hours |
| **Swipe history too large** | Bloom filter for "already swiped" check (false positive = user re-shown, not harmful) |
| **Redis GeoSet loss** | Rebuild from MySQL user locations on startup |
| **Unfair ranking** | Elo score decay for inactive users; reset periodically |

### Race Conditions

#### 1. Simultaneous Mutual Swipe — Double Match

```
A swipes right on B at T=0.000
B swipes right on A at T=0.001

Without protection: Both threads check → both see no prior right-swipe → 
both record right-swipe → both detect match → TWO match records created!

Solution: Redis Lua script (atomic):
  local already_liked = redis.call('SISMEMBER', 'swiped_right:'..B, A)
  redis.call('SADD', 'swiped_right:'..A, B)
  if already_liked == 1 then
    return 1  -- MATCH
  end
  return 0  -- no match yet

This is atomic (Redis is single-threaded) → no race condition possible.
Match record written to MySQL idempotently:
  INSERT INTO matches (user_id_1, user_id_2) VALUES (min(A,B), max(A,B))
  ON DUPLICATE KEY IGNORE
  
  Always store (smaller_id, larger_id) → prevents duplicate match records.
```

#### 2. Stale Location — User Moves to New City

```
Alice was in New York → Tinder shows NY profiles
Alice flies to London → still showing NY profiles!

Solution: Update location on app open + every 30 minutes while active
  GEOADD users:geo {new_lng} {new_lat} {user_id}
  Invalidate cached deck: DEL deck:{user_id}
  → Next swipe request regenerates deck with London profiles
```

---

## 8. Deep Dive: Engineering Trade-offs

### Elo Score vs ML Ranking

```
Elo Score (Tinder's original approach):
  Each user has an Elo rating (like chess)
  If a high-Elo user swipes right on you → your Elo increases more
  If a low-Elo user swipes left → less impact
  Show users with similar Elo scores to each other
  
  ✓ Simple, computationally cheap
  ✗ Reduces to "attractiveness ranking" → ethical concerns
  ✗ New users with no swipe data → cold start
  ✗ Doesn't account for individual preferences
  
  Tinder deprecated Elo in 2019.

ML-based scoring ⭐ (current approach):
  Features: profile completeness, photo quality, bio length,
            shared interests, response rate in chat, activity level
  Model: predicts P(mutual match) for each pair
  
  ✓ Considers compatibility, not just attractiveness
  ✓ Handles cold start (content-based features for new users)
  ✗ More compute-intensive (inference per candidate pair)
  ✗ Feedback loop: model trained on past matches → may reinforce biases

Best: ML model for ranking + diversity injection to expose users to
  profiles they wouldn't normally see (exploration vs exploitation)
```

### Geohash vs H3 vs R-tree for Proximity

```
Redis GeoSet (Geohash internally) ⭐:
  GEORADIUS: find all users within radius
  ✓ Built into Redis, zero additional infrastructure
  ✓ Fast: O(N) where N = users in the area
  ✗ Accuracy decreases near poles (Geohash distortion)
  ✗ No polygon queries (only radius)
  Best for: Simple radius-based proximity (exactly Tinder's use case)

H3 (Uber's hexagonal grid):
  ✓ Uniform cell sizes worldwide (no pole distortion)
  ✓ Hierarchical (zoom levels for different granularities)
  ✗ Not built into Redis (application-level conversion needed)
  Best for: Ride-hailing, delivery (non-uniform coverage needs)

PostGIS / R-tree:
  ✓ Arbitrary polygon queries
  ✓ Most accurate spatial operations
  ✗ Slower than Redis for simple proximity
  ✗ Harder to scale horizontally
  Best for: Complex geo queries (geofencing, polygon search)

For Tinder: Redis GeoSet is the simplest and fastest choice.
```

### Already-Swiped Tracking — Bloom Filter for Memory Efficiency

```
Problem: User has swiped on 50K profiles over 6 months.
  swiped:{user_id} SET in Redis = 50K × 16 bytes = 800 KB per user
  200M DAU × 800 KB = 160 TB just for swipe dedup → TOO EXPENSIVE

Bloom filter approach ⭐:
  BF per user: 50K entries, 0.1% false positive rate → 72 KB per user
  200M DAU × 72 KB = 14 TB → 10× reduction

  False positive impact: BF says "already swiped" but actually not →
    User never sees that profile → missed opportunity, but harmless
    At 0.1% rate → 1 in 1000 profiles incorrectly filtered → acceptable

  Implementation:
    BF.ADD swiped_bf:{user_id} {target_user_id}
    BF.EXISTS swiped_bf:{user_id} {candidate_id}
    → If exists → filter out from deck
    → If not exists → show in deck (guaranteed correct)

  Cassandra stores exact swipe history for auditing / undo feature.
  Bloom filter is a READ optimization, not the source of truth.
```

### Profile Boost — Temporarily Increasing Visibility

```
Premium feature: "Boost" places your profile at the top of nearby users' decks

Implementation:
  On boost activation:
    SET boost:{user_id} {expiry_timestamp} EX 1800  (30-minute boost)
    ZADD boosted_users:{geohash_prefix} {score=999} {user_id}

  During deck generation for nearby users:
    1. First: pull boosted users in this geo area (ZREVRANGE boosted_users:...)
    2. Then: normal ranked candidates
    3. Mix: 1 boosted profile per 5 normal profiles
    
  Revenue model: ~$5 per boost → at 10M boosts/month = $50M/month
  
  Anti-abuse: Max 1 boost per 12 hours, no stacking.
  Fairness: If too many boosts in one area → dilute effect (each boost gets
  fewer guaranteed views → prevents boost-only decks)
```

### Safety — Photo Verification and Reporting

```
Problem: Catfishing (fake photos), harassment, underage users

Photo verification:
  1. User takes a real-time selfie matching a specific pose
  2. ML model compares selfie to profile photos (face matching)
  3. If match confidence > 0.9 → verified badge on profile
  4. Not face recognition (privacy) → just "does this face match the photos?"

Reporting flow:
  User reports another user → {reporter, reported, reason, evidence}
  1. Auto-actions based on report count:
     5 reports → profile temporarily hidden from discovery
     10 reports → account suspended pending human review
  2. Severe reports (harassment, threats) → immediate suspension
  3. Human review SLA: 24 hours
  4. Repeat offenders → permanent ban + device fingerprint ban

Underage detection:
  Age from profile (self-reported) → if < 18 → cannot create account
  ML age estimation on profile photos → flag if estimated < 18
  Flagged accounts require ID verification before publishing profile
```

