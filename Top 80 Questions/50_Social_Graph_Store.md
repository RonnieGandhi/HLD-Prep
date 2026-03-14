# 50. Design a Social Graph Store

---

## 1. Functional Requirements (FR)

- **Follow/Unfollow**: User A follows/unfollows User B (directed edge)
- **Friend request**: Mutual follow / bidirectional friendship (undirected edge)
- **Get followers**: List of users following User A
- **Get following**: List of users User A follows
- **Mutual friends**: "You and Alice have 12 mutual friends"
- **Friend-of-friend**: 2nd degree connections for recommendations
- **Graph queries**: Shortest path, connected components, influence scoring
- **Blocking**: Exclude blocked users from all graph queries

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Follower/following list in < 50 ms
- **High Write Throughput**: Millions of follow/unfollow per day
- **Scale**: 2B+ users, 500B+ edges (avg 250 connections per user)
- **Consistency**: Follow action must be immediately visible to the actor
- **Availability**: 99.99%
- **Traversal Performance**: 2-hop queries (friend-of-friend) in < 200 ms

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Users (nodes) | 2B |
| Edges (follow relationships) | 500B |
| Follow/unfollow ops / day | 500M |
| Follower list queries / sec | 100K |
| Edge record size | 32 bytes (2 user_ids + timestamp) |
| Total edge storage | 500B × 32 bytes = 16 TB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          API Layer                                   │
│  POST /follow    DELETE /unfollow    GET /followers    GET /following│
│  GET /mutual-friends    GET /friend-suggestions                      │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                     ┌──────▼──────┐
                     │ Social Graph│
                     │ Service     │
                     └──────┬──────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
  ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
  │  Primary    │   │  Graph      │   │  Cache      │
  │  Store      │   │  Index      │   │  Layer      │
  │             │   │             │   │             │
  │  MySQL      │   │  TAO /      │   │  Redis      │
  │  (Vitess)   │   │  Neo4j /    │   │             │
  │  or         │   │  Custom     │   │ followers:  │
  │  Cassandra  │   │  adjacency  │   │  {uid} =    │
  │             │   │  list store │   │  sorted set │
  │  Source of  │   │             │   │             │
  │  truth for  │   │  Optimized  │   │ following:  │
  │  edges      │   │  for graph  │   │  {uid} =    │
  │             │   │  traversals │   │  sorted set │
  └─────────────┘   └─────────────┘   └─────────────┘
```

### Component Deep Dive

#### Storage: Adjacency List vs Edge Table

```
Approach 1: Edge Table (relational) ⭐
  CREATE TABLE follows (
    follower_id  BIGINT,
    followee_id  BIGINT,
    created_at   TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
  );
  -- Index for reverse lookup
  CREATE INDEX idx_followee ON follows (followee_id, follower_id);
  
  "Who does Alice follow?" → SELECT followee_id FROM follows WHERE follower_id = alice
  "Who follows Alice?" → SELECT follower_id FROM follows WHERE followee_id = alice
  
  ✓ Simple, well-understood
  ✓ Strong consistency (RDBMS)
  ✗ 2-hop traversals require JOIN → expensive at scale
  ✗ Hot celebrities (100M followers) → massive partition

Approach 2: Adjacency List in Cassandra ⭐ (Facebook TAO-inspired)
  CREATE TABLE following (
    user_id     BIGINT,
    follows     BIGINT,
    followed_at TIMESTAMP,
    PRIMARY KEY (user_id, follows)
  );
  CREATE TABLE followers (
    user_id     BIGINT,
    follower    BIGINT,
    followed_at TIMESTAMP,
    PRIMARY KEY (user_id, follower)
  );
  
  Two tables: one partitioned by follower, one by followee
  Each direction is a single-partition read → fast
  
  ✓ Scales horizontally (Cassandra)
  ✓ Fast single-hop queries
  ✗ 2-hop still requires client-side join
  ✗ Dual writes (must update both tables atomically)

Approach 3: Graph DB (Neo4j, Neptune)
  (:User {id: "alice"})-[:FOLLOWS]->(:User {id: "bob"})
  
  ✓ Native graph traversals (2-hop, shortest path) in < 50ms
  ✓ Rich query language (Cypher)
  ✗ Harder to scale beyond 100B edges
  ✗ Operational complexity
  ✗ Not as fast for simple lookups as key-value
```

#### Facebook TAO — The Industry Standard

```
TAO (The Associations and Objects) — Facebook's custom graph store

Core abstraction:
  Objects: users, pages, groups, posts (nodes)
  Associations: follows, likes, friendships (edges)
  
  Each association has:
    id1 (source), id2 (target), atype (association type), time, data
  
  Stored in MySQL, cached aggressively in a distributed cache layer:
    Cache hit rate: > 99.8%
    Read from cache: < 1 ms
    Cache miss → MySQL: ~5 ms
  
  Write path:
    1. Write to MySQL (source of truth)
    2. Invalidate cache entries for both id1 and id2
    3. Async: update denormalized count tables (follower_count, etc.)
  
  Why not a graph DB?
    Facebook has 2B users × 250 avg connections = 500B edges
    No graph DB scales to this level
    TAO: MySQL + massive cache layer → custom-built for their scale
    
  Key insight: 99%+ of queries are single-hop (get followers, get following)
    Only friend suggestions need multi-hop → done offline in Spark
```

#### Mutual Friends — The Interview Favorite

```
"Alice and Bob have 12 mutual friends" — how to compute?

Naive: 
  A_friends = SELECT followee FROM follows WHERE follower = Alice  → 500 items
  B_friends = SELECT followee FROM follows WHERE follower = Bob    → 300 items
  mutual = INTERSECT(A_friends, B_friends)
  
  Time: O(A + B) with sorted lists or hash sets → fast for small friend lists
  At scale: Alice has 5000 friends, Bob has 3000 → 8000 items to intersect

Optimization: Redis sorted set intersection
  ZINTERSTORE mutual_temp followers:alice followers:bob
  ZCARD mutual_temp → count
  
  Redis does this in-memory in < 5 ms for sets up to 10K members

For celebrities (100M followers):
  Don't compute real-time. Pre-compute mutual count in batch.
  Or: show "You follow [3 friends who follow this celebrity]"
  → Only need intersection with YOUR friend list (small, ~500)
```

---

## 5. APIs

```http
POST /api/v1/follow
{ "target_user_id": "bob" }
→ 200 OK

DELETE /api/v1/follow
{ "target_user_id": "bob" }
→ 200 OK

GET /api/v1/users/{uid}/followers?cursor=...&limit=50
→ { "followers": [{id, name, avatar}, ...], "cursor": "..." }

GET /api/v1/users/{uid}/following?cursor=...&limit=50
→ { "following": [...], "cursor": "..." }

GET /api/v1/users/{uid}/mutual-friends?with=bob
→ { "mutual": [{id, name}, ...], "count": 12 }
```

---

## 6. Data Model

### MySQL/Cassandra — Source of Truth
```sql
CREATE TABLE follows (
    follower_id  BIGINT,
    followee_id  BIGINT,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_followee ON follows (followee_id, follower_id);

CREATE TABLE user_counts (
    user_id        BIGINT PRIMARY KEY,
    follower_count BIGINT DEFAULT 0,
    following_count BIGINT DEFAULT 0
);
```

### Redis — Cache Layer
```
followers:{uid}  → Sorted Set (score=timestamp, member=user_id)
following:{uid}  → Sorted Set (score=timestamp, member=user_id)
follow_count:{uid} → Hash {followers: 1523, following: 342}
```

---

## 7. Fault Tolerance & Race Conditions

| Concern | Solution |
|---|---|
| **Dual write** | Write both tables in same Cassandra batch or MySQL transaction |
| **Count drift** | Async count reconciliation job (hourly) from edge table |
| **Cache invalidation** | On follow/unfollow → invalidate both users' cache entries |
| **Celebrity hot partition** | Shard followers by (followee_id, follower_id_prefix) |

### Race: Follow + Unfollow in Quick Succession
```
T=0: Follow Bob → INSERT follows (alice, bob)
T=50ms: Unfollow Bob → DELETE follows (alice, bob)

If out of order at DB:
  DELETE arrives first (no-op, row doesn't exist)
  INSERT arrives second → alice follows bob (WRONG!)

Solution: Include timestamp, use LWW (Last-Writer-Wins):
  Write with timestamp → reject writes with older timestamps
  Or: Use Cassandra (naturally LWW with cell-level timestamps)
```

---

## 8. Deep Dive: Engineering Trade-offs

### Graph DB vs Relational + Cache

| Feature | Graph DB (Neo4j) | Relational + Cache (TAO) ⭐ |
|---|---|---|
| **1-hop queries** | Fast | Fastest (cache) |
| **Multi-hop** | Native, O(edges) | App-level joins |
| **Scale** | ~10B edges | 500B+ edges |
| **Consistency** | ACID | Eventual (cache) |
| **Operations** | Complex | Well-understood |

For social networks at scale: Relational + cache (TAO pattern).
For knowledge graphs / small-scale: Graph DB.

### Bidirectional Friendship vs Directed Follow

```
Directed follow (Twitter, Instagram):
  A follows B ≠ B follows A
  Store: one row per direction → follows(A, B)
  "A's following": SELECT * FROM follows WHERE follower = A
  "A's followers": SELECT * FROM follows WHERE followee = A
  Need index on BOTH columns

Bidirectional friendship (Facebook, LinkedIn):
  A friends B = B friends A (mutual)
  
  Option 1: Store BOTH directions
    INSERT (A, B) AND INSERT (B, A)
    ✓ Fast query either direction
    ✗ 2× storage, must keep in sync
    
  Option 2: Store one direction (min_id, max_id)
    INSERT (min(A,B), max(A,B))
    "A's friends": SELECT * WHERE user_1 = A OR user_2 = A
    ✗ OR query is slow (can't use single index)
    
  Option 3 ⭐: Separate friend_list table (denormalized)
    CREATE TABLE friend_list (
      user_id   BIGINT,
      friend_id BIGINT,
      PRIMARY KEY (user_id, friend_id)
    );
    INSERT both (A,B) and (B,A) in a transaction
    "A's friends": SELECT friend_id FROM friend_list WHERE user_id = A (fast!)

  Facebook uses Option 3 (via TAO associations stored in both directions).
```

### Degree of Separation — "How far is Alice from Bob?"

```
LinkedIn feature: "You are 2nd degree connected to Bob"

Algorithm: Bidirectional BFS (Breadth-First Search)
  Start BFS from Alice AND from Bob simultaneously
  When the two search frontiers MEET → that's the shortest path

  Alice's frontier: {Alice} → {A's friends} → {A's friends' friends}
  Bob's frontier:   {Bob}   → {B's friends} → {B's friends' friends}
  
  Intersection at depth 1+1 = 2 degrees

Why bidirectional?
  Unidirectional BFS at depth 3: visits ~200^3 = 8M nodes
  Bidirectional BFS at depth 3: visits ~2 × 200^1.5 = ~5.6K nodes
  → 1400× fewer nodes explored!

At scale (offline):
  Pre-compute for ALL user pairs → impossible (N² pairs)
  Pre-compute for top 10K influencers → store in Redis
  Real-time: bidirectional BFS with max depth 3, timeout 200ms
  If no path found within 200ms → return "3+ degrees" 

LinkedIn's approach:
  Pre-compute 1st and 2nd degree connections offline (Spark)
  Store in graph index: 2nd_degree:{uid} = [user_ids]
  Real-time check: is target_user in 2nd_degree:{uid}?
```

### Graph Partitioning — Sharding Edges Across Nodes

```
500B edges across N database nodes. How to shard?

Option 1: Hash partition by user_id
  User A's edges on shard = hash(A) % N
  ✓ Even distribution
  ✗ "Friends of A's friends" requires cross-shard queries
     (A's friends may be on different shards)

Option 2: Social-aware partitioning (METIS algorithm)
  Place users who are densely connected on the SAME shard
  ✓ Most graph traversals stay within one shard
  ✗ Complex to compute, must re-partition periodically
  ✗ Uneven shard sizes if communities have different sizes

Option 3 ⭐: Hash partition + replicated edge list
  Primary copy: hash(user_id) → shard
  For hot queries (mutual friends): read from cache (Redis)
  For multi-hop: fan-out queries to multiple shards in parallel
  
Facebook TAO: hash partition by user_id, 
  with aggressive caching (99.8% hit rate) → cross-shard reads are rare.
```

### Graph Analytics — PageRank and Influence Scoring

```
Problem: Who are the most influential users? (for suggestions, ranking)

PageRank algorithm (offline, Spark GraphX):
  Each user starts with score = 1/N
  Iteratively: distribute score equally to all outgoing edges
  After convergence: high-score users are "influential"
  
  Used for:
    • Follow suggestions (suggest high-PageRank users in your cluster)
    • Content ranking (posts from high-influence users rank higher)
    • Spam detection (spam accounts have abnormally low PageRank)

Storage: PageRank scores in Redis hash
  HSET influence_score {user_id} 0.00042
  Updated weekly via Spark batch job

Community detection (Louvain algorithm):
  Partition users into communities (clusters of densely connected users)
  Used for: content recommendations, friend suggestions, abuse detection
  (If 80% of a community's accounts are < 7 days old → bot ring)
```

