# 51. Design a Follower/Following System

---

## 1. Functional Requirements (FR)

- **Follow/Unfollow**: Follow a user, unfollow a user (directed edge)
- **Follower list**: Get all followers of a user (paginated, sorted by recency)
- **Following list**: Get all users someone follows (paginated)
- **Follower/following count**: Display counts on profile
- **Follow suggestions**: "People you may know" based on mutual connections
- **Follow status check**: "Does Alice follow Bob?" — fast lookup
- **Fan-out on write**: When user posts, deliver to followers' feeds
- **Notifications**: "Alice started following you"

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Follow/unfollow < 100 ms; list queries < 50 ms
- **Scale**: Handle users with 100M+ followers (celebrity accounts)
- **Consistency**: Follow action immediately visible to the actor
- **Eventual Consistency**: Follower count can lag by seconds
- **Availability**: 99.99%
- **Hot spot handling**: Celebrity follow events (100K follows/sec on one user)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total users | 1B |
| Avg followers per user | 200 |
| Total follow edges | 200B |
| Follow/unfollow events / day | 200M |
| Follow events / sec | ~2.3K (peak 50K during celebrity events) |
| Follow check queries / sec | 500K |
| Edge record size | 24 bytes (2 user_ids + timestamp) |
| Edge storage | 200B x 24 bytes = ~4.8 TB |

---

## 4. High-Level Design (HLD)

```
+--------------------------------------------------------------------+
|                          API Layer                                  |
|  POST /follow    DELETE /unfollow    GET /followers    GET /following|
|  GET /mutual-friends    GET /suggestions                            |
+----------------------------+---------------------------------------+
                             |
                      +------v------+
                      | Follow      |
                      | Service     |
                      | (Stateless) |
                      +------+------+
                             |
        +--------------------+--------------------+
        |                    |                    |
  +-----v------+     +------v-------+    +-------v------+
  |  Write     |     |  Read Path   |    | Count        |
  |  Path      |     |              |    | Service      |
  |            |     | Redis cache  |    |              |
  | 1.Validate |     | for hot users|    | Redis INCR/  |
  | 2.Write to |     |              |    | DECR atomic  |
  |  Cassandra |     | Cassandra    |    |              |
  |  (both     |     | for cold     |    | Hourly       |
  |   tables)  |     | users        |    | reconcile    |
  | 3.Update   |     |              |    | with DB      |
  |  Redis     |     |              |    |              |
  | 4.INCR/DECR|     |              |    |              |
  |  count     |     |              |    |              |
  | 5.Kafka    |     |              |    |              |
  |  event     |     |              |    |              |
  +-----+------+     +--------------+    +--------------+
        |
   +----v----------------------------------------------+
   |              Kafka (follow-events)                 |
   |                                                    |
   |  Consumers:                                        |
   |  1. Feed Fan-Out: deliver new posts to followers   |
   |  2. Notification: "Alice followed you"             |
   |  3. Follow Suggestion: update mutual friend graphs |
   |  4. Analytics: follow/unfollow trends              |
   +----------------------------------------------------+

+--------------------------------------------------------------------+
|                    Storage Architecture                              |
|                                                                      |
|  +------------------+  +------------------+  +--------------------+ |
|  |    Cassandra      |  |     Redis         |  |    MySQL (counts)  | |
|  |                   |  |                   |  |                    | |
|  |  following table  |  |  following:{uid}  |  |  user_counts table | |
|  |  PK: (user_id,    |  |  = Sorted Set     |  |  follower_count    | |
|  |      follows)     |  |  (hot users only) |  |  following_count   | |
|  |                   |  |                   |  |                    | |
|  |  followers table  |  |  followers:{uid}  |  |  Source of truth   | |
|  |  PK: (user_id,    |  |  = Sorted Set     |  |  for counts        | |
|  |      bucket,      |  |  (hot users only) |  |                    | |
|  |      follower)    |  |                   |  |                    | |
|  |                   |  |  is_following:     |  |                    | |
|  |  is_following     |  |  {uid}:{target}   |  |                    | |
|  |  PK: (follower,   |  |  = "1" (TTL 24h)  |  |                    | |
|  |      followee)    |  |                   |  |                    | |
|  +------------------+  +------------------+  +--------------------+ |
+--------------------------------------------------------------------+
```

### Component Deep Dive

#### Write Path — Follow Action

```
Alice follows Bob:

1. Validate: Alice not blocked by Bob, Bob exists, not already following
   Redis check: EXISTS is_following:alice:bob  (< 1ms)
   If exists -> return "already following" (idempotent)

2. Write to Cassandra (both tables in BATCH):
   INSERT INTO following (user_id, follows, created_at) VALUES (alice, bob, now());
   INSERT INTO followers (user_id, follower, created_at) VALUES (bob, alice, now());

3. Update Redis cache:
   ZADD following:alice {timestamp} bob
   ZADD followers:bob {timestamp} alice
   SETEX is_following:alice:bob "1" 86400

4. Update counts:
   INCR following_count:alice
   INCR follower_count:bob

5. Publish Kafka event:
   { "type": "follow", "follower": "alice", "followee": "bob", "ts": ... }
```

#### Celebrity Hot Partition — 100M Followers

```
Taylor Swift has 100M followers.
followers:taylorswift partition in Cassandra = 100M rows -> HUGE partition!

Problems:
  1. Single Cassandra partition max practical size ~100MB, 100M rows far exceeds
  2. COUNT(*) on 100M rows = timeout
  3. Redis sorted set with 100M members = 1.6 GB for one key

Solutions:

  1. Bucketed partitioning:
     PRIMARY KEY ((user_id, bucket), follower_id)
     bucket = follower_id % 100  ->  100 sub-partitions
     Each sub-partition: ~1M rows -> manageable
     
     Query followers: parallel read from all 100 buckets -> merge client-side
     Follow/unfollow: write to bucket = follower_id % 100 (deterministic)

  2. Count stored separately (never computed by scanning):
     Redis: INCR/DECR on follow/unfollow -> O(1)
     Reconcile hourly: COUNT from Cassandra -> fix Redis if drifted

  3. Don't cache celebrity follower lists in Redis:
     Only cache following lists (< 10K for most users -> small)
     For "who follows Taylor Swift": paginated Cassandra reads
     For "does Alice follow Taylor Swift": is_following key (O(1))
```

#### Follow Suggestions — "People You May Know"

```
Algorithm: Mutual friends / friends-of-friends

For Alice:
  1. Get Alice's following list: [Bob, Carol, Dave]
  2. For each: get THEIR following lists:
     Bob follows: [Eve, Frank, Grace]
     Carol follows: [Eve, Hannah, Ivan]
     Dave follows: [Eve, Frank]
  3. Aggregate: Eve appears 3 times, Frank 2, Grace 1, Hannah 1, Ivan 1
  4. Filter: remove already-followed, blocked users
  5. Rank by frequency -> Eve (3 mutual), Frank (2) -> suggest Eve first

At scale (offline computation):
  Spark MapReduce: for each user -> emit (user, friend_of_friend, count)
  Store results in Redis: suggestions:{uid} = [user_ids]
  Refresh daily. Pre-computed -> instant serving.
```

---

## 5. APIs

```http
POST /api/v1/users/{uid}/follow
  -> 200 OK { "following": true }

DELETE /api/v1/users/{uid}/follow
  -> 200 OK { "following": false }

GET /api/v1/users/{uid}/followers?cursor={last_id}&limit=50
  -> { "followers": [{id, name, avatar}, ...], "total": 15234, "cursor": "..." }

GET /api/v1/users/{uid}/following?cursor={last_id}&limit=50
  -> { "following": [...], "total": 342, "cursor": "..." }

GET /api/v1/users/{uid}/is-following/{target_uid}
  -> { "following": true }

GET /api/v1/users/{uid}/mutual-friends?with={other_uid}
  -> { "mutual": [{id, name}, ...], "count": 12 }

GET /api/v1/users/{uid}/suggestions?limit=20
  -> { "suggestions": [{id, name, mutual_count: 12}, ...] }
```

---

## 6. Data Model

### Cassandra — Source of Truth

```sql
-- "Who does this user follow?" (paginated, most-recent-first)
CREATE TABLE following (
    user_id    UUID,
    follows    UUID,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, follows)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- "Who follows this user?" (bucketed for celebrities)
CREATE TABLE followers (
    user_id    UUID,
    bucket     INT,
    follower   UUID,
    created_at TIMESTAMP,
    PRIMARY KEY ((user_id, bucket), created_at, follower)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Fast "does A follow B?" check
CREATE TABLE is_following (
    follower_id UUID,
    followee_id UUID,
    PRIMARY KEY (follower_id, followee_id)
);
```

**Why Cassandra?**
- Write-heavy workload (200M writes/day) — Cassandra excels
- Bucketed partitioning handles 100M-follower celebrities
- Cell-level timestamps give natural Last-Writer-Wins conflict resolution
- Tunable consistency: CL=ONE for reads, CL=QUORUM for writes
- No TTL needed (follows are permanent unlike stories)

### Redis — Cache Layer

```
following:{uid}       -> Sorted Set (score=timestamp, member=user_id)
followers:{uid}       -> Sorted Set (non-celebrity users only, < 100K)
is_following:{a}:{b}  -> "1" (TTL: 24h, refreshed on access)
follower_count:{uid}  -> Integer (INCR/DECR atomic)
following_count:{uid} -> Integer
suggestions:{uid}     -> List of user_ids (pre-computed, TTL: 24h)
```

---

## 7. Fault Tolerance & Race Conditions

| Concern | Solution |
|---|---|
| **Dual table write** | Cassandra BATCH ensures both tables updated atomically |
| **Count drift** | Hourly reconciliation job: count edges in Cassandra -> fix Redis |
| **Cache invalidation** | On follow/unfollow: update Redis + Kafka event for downstream |
| **Celebrity hot partition** | Bucketed partitioning (100 buckets per celebrity) |
| **Follow spam** | Rate limit: max 100 follows/hour per user |
| **Redis failure** | Fallback to Cassandra direct reads; rebuild cache from DB |

### Race Condition: Follow + Unfollow Rapid Succession

```
T=0:    Alice follows Bob   -> Cassandra INSERT with ts=0
T=50ms: Alice unfollows Bob -> Cassandra DELETE with ts=50

If Cassandra writes arrive out of order at replicas:
  INSERT (ts=0) arrives at replica 1 first
  DELETE (ts=50) arrives at replica 2 first
  
  Cassandra resolves via cell-level timestamps:
  DELETE at ts=50 > INSERT at ts=0 -> DELETE wins -> CORRECT!
  
  This is Cassandra's natural Last-Writer-Wins (LWW) behavior.

Redis: ZADD + ZREM are idempotent -> correct regardless of order
Count: INCR then DECR -> net effect = 0 -> correct
```

### Race Condition: Concurrent Follow of Same User

```
Two users (Alice and Carol) both follow Bob at the same time.

Cassandra: Two independent INSERTs into followers table
  INSERT (bob, bucket_alice, alice, ts1) -> partition (bob, 17)
  INSERT (bob, bucket_carol, carol, ts2) -> partition (bob, 42)
  
  Different partitions -> zero contention -> both succeed. No race.

Redis: Two ZADD operations on followers:bob
  ZADD is atomic per command -> both added correctly
  
Count: Two concurrent INCR on follower_count:bob
  Redis INCR is atomic -> 2 increments -> correct count
```

---

## 8. Deep Dive: Engineering Trade-offs

### Feed Fan-Out — The Critical Integration

```
When Alice posts content -> deliver to all her followers' feeds

Normal users (< 10K followers): Fan-out on WRITE
  On post -> Kafka event -> Feed Worker -> write post_id to each
  follower's feed in Redis (LPUSH feed:{follower_id} post_id)
  Latency: < 5 seconds for all 10K followers
  Cost: 10K Redis writes per post (cheap)

Celebrities (> 100K followers): Fan-out on READ
  Don't write to 100M feeds -> 100M Redis writes per post = EXPENSIVE
  Instead: when a user opens their feed:
    1. Read pre-computed feed (posts from normal users they follow)
    2. Read latest posts from followed celebrities (pulled at read time)
    3. Merge, rank by timestamp, return top 50
  Cost: N celebrity queries per feed read (N = followed celebrities, typically < 50)

Hybrid model (Twitter/Instagram's approach):
  Follower count < threshold (e.g., 50K) -> fan-out on write
  Follower count >= threshold -> fan-out on read
  Threshold is configurable; some systems use 10K, others 100K.
```

### Graph DB (Neo4j) vs Cassandra + Redis

| Feature | Neo4j / Graph DB | Cassandra + Redis (TAO pattern) |
|---|---|---|
| **1-hop queries** | Fast | Fastest (Redis cache < 1ms) |
| **Multi-hop (friend-of-friend)** | Native O(edges traversed) | App-level joins / batch Spark |
| **Write throughput** | Moderate | Very high (Cassandra) |
| **Scale** | ~10B edges (single cluster) | 500B+ edges (horizontal sharding) |
| **Consistency** | ACID transactions | Eventual (cache), tunable (Cassandra) |
| **Operations** | Complex clustering | Well-understood at scale |
| **Mutual friends** | Cypher query (built-in) | Redis ZINTERSTORE or app-level |

**Recommendation**: Cassandra + Redis (TAO pattern) for social networks at scale.
Graph DB only if multi-hop traversals are a core product feature (LinkedIn-style).

### Mutual Friends Computation

```
"Alice and Bob have 12 mutual friends"

Approach 1: Redis Set Intersection
  ZINTERSTORE temp_result followers:alice followers:bob
  ZCARD temp_result -> 12
  Redis does this in-memory in < 5ms for sets up to 10K members
  
  For celebrities (100M followers): intersection is too expensive
  Solution: Only intersect with YOUR friend list (small, ~500)
    "3 friends you follow also follow Taylor Swift"
    Intersect following:alice (500) with followers:taylor (skip, too large)
    Instead: for each of Alice's 500 following -> check is_following:{friend}:taylor
    500 Redis lookups with pipeline -> < 10ms

Approach 2: Pre-compute (for profile display)
  On follow event: check if new followee is followed by any mutual friend
  Store mutual_count in a separate table
  Update incrementally (not recomputed from scratch)
```

