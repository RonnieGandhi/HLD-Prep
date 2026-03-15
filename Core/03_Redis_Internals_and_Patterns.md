# Redis Internals, Data Structures & System Design Patterns

Core concept tested in HLD rounds: "Why did you choose Redis here? What data structure would you use? What happens when Redis runs out of memory?"

---

## 1. Why Redis — When and Why to Choose It

```
Redis is an in-memory data structure store. Choose it when you need:

✓ Sub-millisecond latency (in-memory, no disk I/O on reads)
✓ Rich data structures (not just key-value — sorted sets, streams, etc.)
✓ Atomic operations (single-threaded command execution)
✓ Pub/Sub or Streams for real-time messaging
✓ TTL-based expiry (sessions, caches, rate limiters)

✗ DON'T use for: Primary source of truth (data loss on crash possible)
✗ DON'T use for: Data larger than available RAM
✗ DON'T use for: Complex queries (no JOINs, no SQL, limited secondary indexes)
```

---

## 2. Data Structures — When to Use Each

### String

```
SET key value [EX seconds] [NX]
GET key
INCR key          ← atomic increment
MGET key1 key2    ← batch get

Use cases:
  • Cache (simple key→value): SET user:42 '{"name":"Alice"}'
  • Counters: INCR page_views:homepage
  • Distributed lock: SET lock:order:123 owner_id NX EX 30
  • Rate limiter: INCR ratelimit:user:42:minute EX 60
  • Session store: SET session:abc123 '{"user_id":42}' EX 1800

Size: Max 512 MB per string value
Memory: ~56 bytes overhead per key (dict entry + SDS header)
```

### Hash

```
HSET user:42 name "Alice" age 30 city "NYC"
HGET user:42 name          → "Alice"
HGETALL user:42            → {name: "Alice", age: "30", city: "NYC"}
HINCRBY user:42 age 1      ← atomic field increment

Use cases:
  • Object storage (user profile, product details) — more memory-efficient than
    storing as JSON string because fields are individually addressable
  • Partial updates without reading entire object: HSET user:42 city "SF"
  • Rate limiter with multiple windows: HSET ratelimit:user:42 minute 5 hour 100
  
Memory optimization:
  If hash has < 128 fields AND all values < 64 bytes:
    Redis uses ziplist encoding (compact, contiguous memory)
  Otherwise: hashtable encoding (standard, more memory)
  
  Configure thresholds:
    hash-max-ziplist-entries 128
    hash-max-ziplist-value 64
```

### List

```
LPUSH queue:emails "msg1"          ← push left
RPOP queue:emails                  ← pop right (FIFO queue)
BRPOP queue:emails 30              ← blocking pop (wait up to 30s)
LRANGE feed:user:42 0 19           ← get first 20 items

Use cases:
  • Message queue (simple, but prefer Streams or Kafka for production)
  • News feed: LPUSH feed:user:42 post_id (latest first)
    LTRIM feed:user:42 0 499  ← keep only latest 500
  • Activity log: append events, read recent N
  
Internal: Quicklist (linked list of ziplists) — good for both ends access
Max size: 2^32 - 1 elements
```

### Set

```
SADD liked:post:100 user_42 user_55 user_99
SISMEMBER liked:post:100 user_42     → 1 (true)
SCARD liked:post:100                 → 3 (count)
SMEMBERS liked:post:100              → [user_42, user_55, user_99]
SINTER friends:alice friends:bob     → mutual friends

Use cases:
  • Tracking unique items: who liked a post, who's online, unique visitors
  • Deduplication: "has user already voted on this?"
  • Set operations: mutual friends (SINTER), recommended friends (SDIFF)
  • Tagging: SADD tags:post:100 "redis" "database" "nosql"
  
Memory: Each member stored as a hashtable entry (~56 bytes overhead per member)
  For small sets (< 128 members, all < 64 bytes): intset or ziplist encoding
```

### Sorted Set (ZSet) ⭐ Most Powerful Structure

```
ZADD leaderboard 1500 "alice" 1200 "bob" 1800 "charlie"
ZREVRANGE leaderboard 0 9 WITHSCORES     → top 10 players with scores
ZRANK leaderboard "alice"                → rank (0-based)
ZRANGEBYSCORE leaderboard 1000 2000      → scores between 1000-2000
ZINCRBY leaderboard 50 "alice"           ← atomic score increment
ZREVRANGEBYSCORE feed:user:42 +inf -inf LIMIT 0 20  → latest 20 feed items

Use cases:
  • Leaderboard / ranking: ZINCRBY leaderboard score player
  • Rate limiter (sliding window):
      ZADD ratelimit:user:42 {timestamp} {request_id}
      ZREMRANGEBYSCORE ratelimit:user:42 0 {now - window_size}
      ZCARD ratelimit:user:42 → count in window
  • Delayed job queue: ZADD delayed_jobs {execute_at_timestamp} {job_id}
      Poll: ZRANGEBYSCORE delayed_jobs 0 {now} LIMIT 0 10
  • News feed cache: ZADD feed:user:42 {post_timestamp} {post_id}
  • Priority queue: score = priority
  • Time-series (recent events): score = timestamp

Internal: Skip list + hash table
  Skip list: O(log N) insert, delete, range query
  Hash table: O(1) score lookup by member
  
  Combined: O(log N) for rank/range operations, O(1) for score lookup
```

### HyperLogLog (Probabilistic Cardinality)

```
PFADD unique_visitors:2026-03-15 "user_42" "user_55" "user_42"
PFCOUNT unique_visitors:2026-03-15    → ~2 (deduplicated count)
PFMERGE unique_visitors:week unique_visitors:2026-03-10 ... unique_visitors:2026-03-16

Use cases:
  • Count unique visitors/IPs/users with O(1) memory
  • Union cardinality across time windows (daily → weekly → monthly)

Memory: Fixed 12 KB per HyperLogLog regardless of cardinality
Accuracy: ±0.81% standard error
Trade-off: Can't retrieve individual members — only the count

When to use vs Set:
  1M unique users/day:
    SET: 1M × ~60 bytes = 60 MB per day
    HyperLogLog: 12 KB per day (5000× less memory!)
  If you need "was user X in the set?" → use Set
  If you only need "how many unique users?" → use HyperLogLog
```

### Bloom Filter (Redis Module: RedisBloom)

```
BF.ADD seen_urls "https://example.com/page1"
BF.EXISTS seen_urls "https://example.com/page1"  → 1 (probably yes)
BF.EXISTS seen_urls "https://example.com/page2"  → 0 (definitely no)

Use cases:
  • Cache penetration prevention: "Does this key exist in DB?"
    If Bloom says NO → definitely doesn't exist → return 404 (skip DB)
    If Bloom says YES → might exist → check cache/DB
  • Web crawler: "Have I already crawled this URL?"
  • Username availability: "Is this username taken?"

Properties:
  False positive: possible (says "yes" but key doesn't exist) — rate configurable
  False negative: IMPOSSIBLE (if it says "no", key definitely doesn't exist)
  Memory: ~1.2 GB for 1 billion items at 1% false positive rate
  Cannot delete items (use Cuckoo Filter if deletion needed)
```

### Geospatial

```
GEOADD restaurants -122.4194 37.7749 "joes_pizza"
GEOADD restaurants -122.4089 37.7844 "bobs_burgers"
GEORADIUS restaurants -122.42 37.78 5 km WITHCOORD WITHDIST COUNT 20 ASC

Use cases:
  • Nearby search: find restaurants/drivers/users within radius
  • Distance calculation: GEODIST restaurants "joes_pizza" "bobs_burgers" km

Internal: Stored as a Sorted Set with GeoHash as the score
  → All Sorted Set operations work (ZRANGE, ZRANK, etc.)
```

### Streams (Kafka-Like Log)

```
XADD orders * user_id 42 item "widget" quantity 1
  → Returns stream ID: "1710320000000-0"

XREAD COUNT 10 BLOCK 5000 STREAMS orders $
  → Blocking read (consumer waits for new entries)

XREADGROUP GROUP order_processors consumer_1 COUNT 10 STREAMS orders >
  → Consumer group read (each message delivered to ONE consumer in group)

XACK orders order_processors "1710320000000-0"
  → Acknowledge processing complete

Use cases:
  • Event log / message queue (persistent, replayable)
  • Consumer groups (like Kafka consumer groups — fan-out with exactly-once-per-group delivery)
  • Activity stream / audit log

vs Pub/Sub:
  Pub/Sub: fire-and-forget (offline subscribers miss messages)
  Streams: persistent, replayable, consumer groups, acknowledgment

vs Kafka:
  Streams: simpler, lower throughput (~100K msg/sec), single-node or cluster
  Kafka: distributed, higher throughput (~1M msg/sec), built for streaming at scale
  Use Streams for: lightweight event processing, microservice communication
  Use Kafka for: high-throughput event pipelines, analytics, CDC
```

---

## 3. Persistence — RDB vs AOF

### RDB (Snapshotting)

```
Periodic point-in-time snapshots to disk:
  save 900 1        # Snapshot if ≥1 key changed in 900 seconds
  save 300 10       # Snapshot if ≥10 keys changed in 300 seconds
  save 60 10000     # Snapshot if ≥10000 keys changed in 60 seconds

How: fork() → child process writes entire dataset to dump.rdb
  Parent continues serving (copy-on-write for modified pages)

✓ Compact file (good for backups, DR)
✓ Fast restart (load entire dataset from single file)
✗ Data loss: up to the last snapshot interval (minutes of data)
✗ fork() can be slow for large datasets (copy page tables)
```

### AOF (Append-Only File)

```
Every write operation appended to a log file:
  appendonly yes
  appendfsync everysec    # fsync every 1 second (recommended)
  # appendfsync always    # fsync on every write (safest but slowest)
  # appendfsync no        # let OS decide when to fsync (fastest, risky)

AOF Rewrite:
  AOF file grows over time (even for same key updated 1000 times)
  Background rewrite: compact AOF by reading current dataset state
  Example: 1000 INCRs on key "counter" → rewritten as single SET counter 1000

✓ Minimal data loss (at most 1 second with everysec)
✓ Append-only → corruption-resistant (partial write = just truncate)
✗ Larger file than RDB
✗ Slower restart (replay all commands)
```

### Hybrid (RDB + AOF) ⭐ Production Recommendation

```
Use both:
  aof-use-rdb-preamble yes
  
  AOF file starts with RDB snapshot (fast load)
  Followed by AOF entries since snapshot (minimal data loss)
  
  → Fast restart (RDB) + minimal data loss (AOF)
```

---

## 4. Redis Cluster vs Sentinel

### Redis Sentinel (HA for Single-Shard)

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Sentinel │     │ Sentinel │     │ Sentinel │
│    1     │     │    2     │     │    3     │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │  monitor        │               │
┌────▼─────┐     ┌─────▼────┐    ┌─────▼────┐
│  Master  │────►│ Replica  │    │ Replica  │
└──────────┘     └──────────┘    └──────────┘

Role: Automatic failover for a single master
  • Monitor: Sentinels check master health
  • Notification: Alert if master fails
  • Failover: Promote a replica to master (majority vote among sentinels)
  • Configuration: Clients query sentinel for current master address

When to use: Dataset fits in single node's memory
  Single master handles all writes → write bottleneck at scale
```

### Redis Cluster (Sharded, Multi-Master)

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Master A │     │ Master B │     │ Master C │
│ slots    │     │ slots    │     │ slots    │
│ 0-5460   │     │ 5461-    │     │ 10923-   │
│          │     │ 10922    │     │ 16383    │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
┌────▼─────┐     ┌────▼─────┐     ┌────▼─────┐
│ Replica  │     │ Replica  │     │ Replica  │
│    A1    │     │    B1    │     │    C1    │
└──────────┘     └──────────┘     └──────────┘

Role: Horizontal scaling (data partitioned across masters)
  • 16,384 hash slots distributed across masters
  • Key routing: slot = CRC16(key) % 16384
  • Each master handles reads + writes for its slots
  • MOVED redirect: if key sent to wrong node → client redirects
  
When to use: Dataset exceeds single node's memory
  OR: need more write throughput (multiple masters)
  
Limitations:
  • Multi-key operations ONLY if all keys on same slot
    → Use hash tags: {user:42}:profile and {user:42}:settings → same slot
  • No multi-database support (only DB 0)
  • Cluster management overhead
```

---

## 5. Common System Design Patterns with Redis

### Distributed Lock

```
ACQUIRE:
  SET lock:resource_name {owner_id} NX EX 30
  NX = only if not exists (atomic)
  EX = 30 second expiry (auto-release if holder crashes)

RELEASE (Lua script — atomic check-and-delete):
  if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  end
  return 0

Why Lua? Without it: GET + DEL is NOT atomic → race condition
  Thread A: GET lock → "my_id" (matches!) 
  Thread B: <lock expires, B acquires lock>
  Thread A: DEL lock → deletes B's lock!
```

### Rate Limiter (Sliding Window)

```
Sorted Set approach:
  Key: ratelimit:{user_id}:{endpoint}
  Score: request timestamp (ms)
  Member: unique request ID

  On request:
    now = current_time_ms()
    window_start = now - 60000  # 1 minute window
    
    MULTI
      ZREMRANGEBYSCORE key 0 {window_start}   # remove old entries
      ZADD key {now} {request_id}             # add current request
      ZCARD key                                # count in window
    EXEC
    
    if count > limit → reject (429 Too Many Requests)

  Memory: Each entry ~50 bytes. 1M users × 100 requests/min = ~5 GB
```

### Session Store

```
Key: session:{session_id}
Value: Hash { user_id, role, csrf_token, last_active }
TTL: 30 minutes (refreshed on every request)

HSET session:abc123 user_id 42 role "admin" last_active 1710320000
EXPIRE session:abc123 1800

On request: HGETALL session:abc123 → if empty → session expired → redirect to login
On activity: EXPIRE session:abc123 1800 → refresh TTL
```

### Pub/Sub (Real-Time Notifications)

```
Publisher: PUBLISH channel:user:42 '{"type":"new_message","from":"alice"}'
Subscriber: SUBSCRIBE channel:user:42 → receives messages in real-time

Use cases:
  • Chat message delivery between servers
  • Real-time notifications
  • Cache invalidation broadcasts

⚠ Limitations:
  • Fire-and-forget: offline subscribers MISS messages (not persisted)
  • No acknowledgment: no delivery guarantee
  • No consumer groups: every subscriber gets every message
  → For durable messaging: use Redis Streams or Kafka
```

---

## 6. Memory Management & Eviction

### What Happens When Redis Runs Out of Memory?

```
Behavior depends on maxmemory-policy:

noeviction (default):
  Returns OOM error on writes. Reads still work.
  → Safest for critical data, but writes fail!

allkeys-lru:
  Evict least-recently-used key (approximated: sample 5 random keys, evict oldest)
  → Best for cache use case (general purpose)

volatile-lru:
  Evict least-recently-used key ONLY among keys with TTL set
  → Keeps permanent keys safe, evicts only expiring keys

allkeys-lfu:
  Evict least-frequently-used key (Redis 4.0+)
  → Better than LRU for skewed access patterns (keeps hot keys)

volatile-ttl:
  Evict keys with shortest remaining TTL first
  → Good when you want to evict "about to expire anyway" keys first

volatile-random / allkeys-random:
  Random eviction. Rarely used.

Production recommendation:
  Cache: allkeys-lfu (best hit rate for hot/cold workloads)
  Session store: volatile-lru (only evict sessions, not config data)
  Mixed: volatile-lru with important keys having no TTL (never evicted)
```

### Memory Optimization Tips

```
1. Use Hash for small objects (ziplist encoding saves ~10× memory vs individual keys)
   Instead of: SET user:42:name "Alice", SET user:42:age 30
   Use:        HSET user:42 name "Alice" age 30

2. Use short key names in production:
   Instead of: user_profile:user_id:42:settings:notification_preferences
   Use:        up:42:s:np

3. Use integer IDs as SET/ZSET members (intset encoding)
   Instead of: SADD followers:42 "user_abc_def_ghi"
   Use:        SADD followers:42 123456789

4. Set maxmemory to 80% of available RAM (leave room for fork + OS)

5. Monitor: INFO memory → used_memory, used_memory_rss, mem_fragmentation_ratio
   If fragmentation_ratio > 1.5 → restart Redis to defragment
   Or use: activedefrag yes (Redis 4.0+)
```

---

## 7. Lua Scripting — Atomic Multi-Step Operations

```
Why Lua in Redis?
  Redis is single-threaded → one command at a time
  But multi-step operations (check-then-set) are NOT atomic across commands
  Lua script executes atomically — no other command can interleave

Example: Atomic rate limiter
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])
  
  redis.call("ZREMRANGEBYSCORE", key, 0, now - window)
  local count = redis.call("ZCARD", key)
  
  if count < limit then
    redis.call("ZADD", key, now, now .. ":" .. math.random())
    redis.call("EXPIRE", key, window / 1000)
    return 1  -- allowed
  else
    return 0  -- rate limited
  end

EVALSHA {sha} 1 ratelimit:user:42 100 60000 1710320000000

✓ Atomic: entire script runs without interleaving
✓ Low latency: script executes server-side (no network round-trips between steps)
✗ Long-running scripts block ALL Redis operations (single-threaded)
  → Keep scripts short (< 5ms). Never do I/O in Lua scripts.
```

---

## 8. Redis in System Design — Decision Cheat Sheet

```
Need                          → Redis Solution
─────────────────────────────────────────────────────
Simple cache                  → String with TTL
Object cache                  → Hash with TTL
Counting (exact)              → String INCR
Counting (approximate unique) → HyperLogLog
Leaderboard / ranking         → Sorted Set
Queue (simple)                → List (LPUSH/BRPOP)
Queue (persistent, groups)    → Streams
Pub/Sub (fire-and-forget)     → Pub/Sub
Distributed lock              → String SET NX EX + Lua DEL
Rate limiter                  → Sorted Set (sliding window) or String INCR (fixed window)
Session store                 → Hash with EXPIRE
Geospatial search             → Geospatial (GEOADD/GEORADIUS)
Non-existent key guard        → Bloom Filter
Feed / timeline cache         → Sorted Set (score = timestamp)
Online/presence tracking      → String with TTL (heartbeat refreshes)
Feature flags                 → Hash (flag_name → enabled/disabled)
```

