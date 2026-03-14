# 94. Design a Leaderboard System

---

## 1. Functional Requirements (FR)

- Real-time global leaderboard showing top-K users by score
- User can query their own rank among millions of players
- Update scores in real time (increment, set, decay)
- Multiple leaderboards: daily, weekly, seasonal, all-time, per-game-mode
- Friends leaderboard: rank among friends only
- Historical leaderboards: view past week/season final standings
- Tie-breaking: same score → earlier achievement time ranks higher
- Percentile ranking: "You are in the top 5% of all players"

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Top-K query < 10ms; rank lookup < 50ms
- **High Write Throughput**: 100K score updates/sec during peak
- **Consistency**: Eventual consistency OK (1-2 second delay acceptable)
- **Scalability**: 100M+ players per leaderboard
- **Availability**: 99.99% — leaderboard is core game feature

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Total players | | 100M |
| Active players per leaderboard | | 10M |
| Score updates / sec (peak) | | 100K |
| Top-K queries / sec | | 50K |
| Rank lookup queries / sec | | 200K |
| Leaderboards (total) | games × modes × timeframes | ~1,000 |
| Memory per leaderboard (Redis) | 10M entries × 150 bytes | ~1.5 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                    │
│  Game Client / Web Dashboard / Mobile App                         │
└───────────────────────────┬───────────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────────┐
│                    API GATEWAY                                     │
│  Rate limiting, auth, request routing                             │
└───────────────────────────┬───────────────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────────┐
│                 LEADERBOARD SERVICE                                │
│  ┌──────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Score Writer     │  │  Rank Reader     │  │  History        │  │
│  │  - Validate score │  │  - Top K          │  │  Service        │  │
│  │  - Update Redis   │  │  - User rank      │  │  - Snapshot     │  │
│  │  - Publish event  │  │  - Around user    │  │    daily top-K  │  │
│  │    to Kafka       │  │  - Friends LB     │  │  - Store to PG  │  │
│  └────────┬─────────┘  └────────┬──────────┘  └────────┬────────┘  │
│           │                     │                       │          │
└───────────│─────────────────────│───────────────────────│──────────┘
            │                     │                       │
   ┌────────▼─────────────────────▼───────┐    ┌──────────▼──────────┐
   │  Redis Cluster                        │    │  PostgreSQL         │
   │  (Primary Data Store)                 │    │  (Historical)       │
   │                                       │    │                     │
   │  Sorted Set per leaderboard:          │    │  - Daily snapshots  │
   │  ZADD lb:weekly:game1 {score} {uid}   │    │    of top 10K       │
   │  ZREVRANK lb:weekly:game1 {uid}       │    │  - Season archives  │
   │  ZREVRANGE lb:weekly:game1 0 99       │    │  - Player history   │
   │  ZINCRBY lb:weekly:game1 {delta} {uid}│    │                     │
   │                                       │    └─────────────────────┘
   │  All ops: O(log N), N = 10M           │
   │  Memory: ~1.5 GB per 10M-entry LB    │    ┌─────────────────────┐
   │  Total: 1000 LBs × 1.5 GB = 1.5 TB   │    │  Kafka              │
   │  (Redis Cluster across nodes)         │    │  - Score events     │
   │                                       │    │  - For analytics    │
   └───────────────────────────────────────┘    │  - Rebuild trigger  │
                                                └─────────────────────┘
```

### Component Deep Dive

#### Redis Sorted Set (Core Data Structure)

**Why Redis Sorted Set is THE answer for leaderboards:**

```
ZADD lb:weekly {score} {user_id}           → O(log N) insert/update
ZREVRANK lb:weekly {user_id}               → O(log N) get user's rank
ZREVRANGE lb:weekly 0 99 WITHSCORES        → O(log N + K) top K
ZINCRBY lb:weekly {delta} {user_id}        → O(log N) atomic increment
ZREVRANGE lb:weekly rank-5 rank+5          → O(log N + K) neighbors

Internal: Skip List — balanced probabilistic data structure
  10M members: ~20 operations per query (log₂(10M) ≈ 23)
  Each operation: ~1µs → total: ~20µs per query
  Memory: ~1.5 GB for 10M entries (key + score + overhead)
```

#### Tie-Breaking

```
Problem: Two users have score=1000 → who ranks higher?
Rule: Earlier achievement = higher rank

Implementation: Encode timestamp into the score
  score_key = score × 10^10 + (MAX_TIMESTAMP - actual_timestamp)
  
  Example:
    User A: score=1000 at t=100  → key = 1000_0000000000 + (9999999999 - 100) = 10009999999899
    User B: score=1000 at t=200  → key = 1000_0000000000 + (9999999999 - 200) = 10009999999799
    
    User A's key > User B's key → User A ranks higher ✓
    (Earlier timestamp → larger key → higher rank)

Redis: ZADD lb:weekly 10009999999899 "userA"
  Sorted by score descending → correct ranking with tie-break
```

#### Scaling Beyond 100M Members

```
Approach 1: Single Redis Sorted Set (recommended up to ~100M)
  100M × 150 bytes = 15 GB → fits in single large Redis instance
  All operations still O(log N) ≈ O(27)

Approach 2: Sharded by Score Range (for > 100M)
  Shard 0: scores 0-999        → Redis instance 0
  Shard 1: scores 1000-1999    → Redis instance 1
  ...
  Shard N: scores 9000-9999    → Redis instance N
  
  Top-K: query highest shard → get top from that shard
  Rank: sum counts in all higher shards + ZREVRANK within shard
  Rebalancing: if shard grows too large, split score range

Approach 3: Fenwick Tree / Binary Indexed Tree (for percentile rank)
  Score range [0, MAX_SCORE] → array of counts
  rank(score) = prefix_sum(MAX_SCORE) - prefix_sum(score)
  O(log S) where S = max score range, regardless of user count
  Memory: 1M scores × 8 bytes = 8 MB (tiny!)
  Best for: "what percentile am I in?" at billion scale
```

#### Friends Leaderboard

```
Problem: Show user's rank among their friends (not global)

Solution 1 (small friend list, < 500):
  ZINTERSTORE lb:friends:{uid} 2 lb:weekly friends:{uid} WEIGHTS 1 0
  → Intersect global leaderboard with friend set → ranked friend list
  Cost: O(N × log(N)) where N = friend count

Solution 2 (on-demand, for large friend lists):
  1. Fetch all friend IDs from social graph
  2. ZSCORE lb:weekly {friend_id} for each friend (pipelined)
  3. Sort in application → return ranked list
  Cost: O(F) where F = friend count, all pipelined to Redis

Recommendation: Solution 2 (simpler, no temp keys, pipeline makes it fast)
```

---

## 5. APIs

```
# Score updates
POST /api/leaderboard/{lb_id}/score
{
  "user_id": "u_123",
  "score_delta": 50        // relative increment
  // OR "score": 1250      // absolute set
}

# Read operations
GET /api/leaderboard/{lb_id}/top?k=100            → Top K players
GET /api/leaderboard/{lb_id}/rank/{user_id}        → User's rank + score
GET /api/leaderboard/{lb_id}/around/{user_id}?n=10 → 10 above + 10 below
GET /api/leaderboard/{lb_id}/friends/{user_id}     → Rank among friends
GET /api/leaderboard/{lb_id}/percentile/{user_id}  → "Top 5%" info

# Historical
GET /api/leaderboard/{lb_id}/history?date=2026-03-01  → Archived standings
```

---

## 6. Data Models

### Redis (Active Leaderboard)

```
# Primary leaderboard (Sorted Set)
ZADD lb:weekly:game1 {score_with_tiebreak} {user_id}

# Metadata per user (Hash, for display in leaderboard)
HSET user:display:{user_id} name "Alice" avatar_url "..." level 42

# Leaderboard metadata
HSET lb:meta:weekly:game1 start_date "2026-03-10" end_date "2026-03-17" status "active"
```

### PostgreSQL (Historical Snapshots)

**Why PostgreSQL?** Relational storage for historical archives, easy querying by date/season.

```sql
CREATE TABLE leaderboard_snapshots (
    leaderboard_id TEXT,
    snapshot_date  DATE,
    rank           INT,
    user_id        UUID,
    score          BIGINT,
    PRIMARY KEY (leaderboard_id, snapshot_date, rank)
);

-- Snapshot top 10,000 daily at midnight
-- Full seasonal archive at season end
```

### Kafka (Score Events)

```
Topic: score-events
  Key: user_id
  Value: { "lb_id": "weekly:game1", "user_id": "u_123", "delta": 50, 
           "new_score": 1250, "timestamp": "..." }
  
Used for: analytics, anti-cheat detection, rebuild leaderboard if Redis lost
```

---

## 7. Fault Tolerance

| Technique | Application |
|---|---|
| Redis persistence | RDB snapshots + AOF → recover on restart |
| Redis Cluster | 3 masters + 3 replicas; auto-failover |
| Rebuild from Kafka | If Redis lost → replay score events → rebuild leaderboard |
| Read replicas | Separate read replicas for top-K (heavy read load) |
| Leaderboard rotation | Weekly reset → archive to PostgreSQL → DEL key |

### Problem-Specific

**Cache Stampede on Leaderboard**: Many users request top-K simultaneously.
  Solution: Read from Redis replicas (eventual consistency OK). If Redis master fails → replica promotes in ~seconds.

**Score Tampering**: Server validates scores (never trust client-submitted scores). Anti-cheat: anomaly detection (score increase too fast → flag + review).

**Leaderboard Rotation (Race Condition)**:
```
Problem: At midnight, rotate weekly leaderboard
  1. Snapshot current leaderboard → PostgreSQL
  2. Delete old leaderboard key
  3. Start accepting scores for new week
  
  Race: User submits score BETWEEN step 1 and 2 → score lost
  
Solution: Use key with time bucket
  Current week: lb:weekly:2026-W11
  New week:     lb:weekly:2026-W12
  Switch read pointer at midnight → no deletion race
  Delete old key after 1 hour grace period
```

---

## 8. Additional Considerations

### Percentile Rank at Scale

```
Problem: User wants to know "I'm in top 5%" (not just rank #50,000)

Approach 1: ZCARD + ZREVRANK
  percentile = (1 - ZREVRANK/ZCARD) × 100
  Exact, but requires two Redis calls

Approach 2: Approximate with histogram (for billion-scale)
  Maintain 1000 score buckets, each tracking count
  percentile = sum(buckets above user's score) / total_users
  O(1) lookup, periodically refreshed (every 1 minute)
  Sufficient accuracy for "top 5%" display
```

### Score Decay (Keeping Leaderboards Fresh)

```
Problem: Inactive players sit at top forever → leaderboard feels stale

Solution: Time-based decay
  Option A: Separate leaderboards with time windows (weekly resets)
  Option B: Daily decay job: ZINCRBY lb:active {-decay_amount} {user_id}
  Option C: Recency-weighted score: score × decay_factor^(days_since_last_play)
  
  Applied via background Flink job processing score events
```

### Deep Dive: Why Redis Sorted Set Wins

```
| Solution | Top-K | User Rank | Update | Memory | Complexity |
|---|---|---|---|---|---|
| MySQL ORDER BY score | O(N log N) | O(N) | O(log N) | Disk | High |
| Redis Sorted Set ⭐ | O(log N + K) | O(log N) | O(log N) | RAM | Low |
| Fenwick Tree | O(S) | O(log S) | O(log S) | RAM | Medium |
| Pre-computed ranks | O(1) | O(1) | O(N) recompute | RAM | High |

Redis Sorted Set is THE go-to answer because:
  1. O(log N) for ALL operations — no trade-offs
  2. Built-in ZREVRANK for instant rank lookup
  3. ZREVRANGE for top-K in one command
  4. ZINCRBY for atomic score updates
  5. Fits in memory for realistic user counts (100M = 15 GB)
  6. Zero custom data structure code — use Redis as-is
  
Only consider Fenwick Tree when:
  - Need percentile at billion scale
  - Score range is bounded (e.g., 0-1,000,000)
  - Don't need top-K with user details (just rank number)
```

