# 27. Design a Distributed Cache (Redis / Memcached)

---

## 1. Functional Requirements (FR)

- `PUT(key, value, TTL)` — Store a key-value pair with optional time-to-live
- `GET(key)` → value — Retrieve value by key; return null if not found or expired
- `DELETE(key)` — Remove a key-value pair
- Support **eviction policies**: LRU, LFU, FIFO, Random
- Support **data structures** beyond strings: hashes, lists, sets, sorted sets (Redis-like)
- Support **TTL/expiry** on keys
- Support **atomic operations**: increment, compare-and-swap
- Distributed across multiple nodes for scalability and availability

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: < 1 ms p99 for read/write operations
- **High Throughput**: 1M+ operations/sec per node
- **High Availability**: Cache should survive node failures without total loss
- **Scalability**: Horizontally scalable — add nodes to increase capacity
- **Consistency**: Eventual consistency across replicas (cache is inherently best-effort)
- **Memory Efficient**: Maximize useful data per GB of RAM
- **Partition Tolerant**: Continue operating during network partitions

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total data cached | 10 TB |
| Avg key size | 100 bytes |
| Avg value size | 1 KB |
| Entry overhead (metadata) | 100 bytes per entry |
| Entries per node (64 GB RAM) | ~50M entries |
| Nodes needed | 10 TB / 64 GB = ~160 nodes (without replication) |
| With replication (2×) | ~320 nodes |
| Operations / sec (total) | 100M |
| Operations / sec / node | ~600K |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Application Servers                                │
│                                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  L1: In-Process Cache (Caffeine / Guava)                         │ │
│  │  ~100 MB per server, < 0.1 ms, hottest keys only                 │ │
│  │  TTL: 30s (short, to limit staleness)                            │ │
│  └────────────────────────────┬──────────────────────────────────────┘ │
│                               │ L1 miss                               │
│  ┌────────────────────────────▼──────────────────────────────────────┐ │
│  │  Cache Client Library (embedded in each app server)              │ │
│  │  • Consistent hashing: CRC16(key) % 16384 → hash slot → node    │ │
│  │  • Connection pooling: persistent TCP to each cache node         │ │
│  │  • Retry + circuit breaker per node                              │ │
│  │  • Compression for values > 1 KB                                 │ │
│  └────────────────────────────┬──────────────────────────────────────┘ │
└───────────────────────────────┼────────────────────────────────────────┘
                                │ direct connection to responsible node
                                │
┌───────────────────────────────▼────────────────────────────────────────┐
│            L2: Distributed Cache Cluster (Redis Cluster)              │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  16,384 Hash Slots distributed across masters                   │  │
│  │                                                                  │  │
│  │  slot = CRC16(key) % 16384                                      │  │
│  │                                                                  │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │  │
│  │  │  Master A      │  │  Master B      │  │  Master C      │     │  │
│  │  │  Slots 0-5460  │  │  Slots 5461-   │  │  Slots 10923-  │     │  │
│  │  │                │  │  10922          │  │  16383         │     │  │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │     │  │
│  │  │ │ Hash Map   │ │  │ │ Hash Map   │ │  │ │ Hash Map   │ │     │  │
│  │  │ │ (in-memory)│ │  │ │            │ │  │ │            │ │     │  │
│  │  │ ├────────────┤ │  │ ├────────────┤ │  │ ├────────────┤ │     │  │
│  │  │ │ Eviction   │ │  │ │ Eviction   │ │  │ │ Eviction   │ │     │  │
│  │  │ │ (LRU/LFU)  │ │  │ │ (LRU/LFU)  │ │  │ │ (LRU/LFU)  │ │     │  │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │     │  │
│  │  │       │        │  │       │        │  │       │        │     │  │
│  │  │  async repl.   │  │  async repl.   │  │  async repl.   │     │  │
│  │  │       │        │  │       │        │  │       │        │     │  │
│  │  │ ┌─────▼──────┐ │  │ ┌─────▼──────┐ │  │ ┌─────▼──────┐ │     │  │
│  │  │ │ Replica A' │ │  │ │ Replica B' │ │  │ │ Replica C' │ │     │  │
│  │  │ │ (failover) │ │  │ │ (failover) │ │  │ │ (failover) │ │     │  │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │     │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘     │  │
│  │                                                                  │  │
│  │  MOVED redirect: if client hits wrong node, node responds with  │  │
│  │  MOVED {slot} {correct_host}:{port} → client updates slot map   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  On cache miss → App reads from Primary DB → writes back to cache     │
└────────────────────────────────────────────────────────────────────────┘
                                │
                         ┌──────▼──────┐
                         │ Primary DB  │  ← Source of truth
                         │ (PostgreSQL │     Cache is a read-through
                         │  / MySQL)   │     acceleration layer
                         └─────────────┘
```

### Component Deep Dive

#### L1: In-Process Cache (Caffeine / Guava)
- Local cache within each application server's JVM — no network hop, < 0.1ms latency
- **Size**: ~100 MB per server (limited by JVM heap — don't starve the application)
- **TTL**: Short (30 seconds) to limit staleness — L1 has no cache invalidation from Redis
- **Eviction**: Window TinyLFU (Caffeine) — near-optimal hit rates, better than LRU for skewed workloads
- **Use case**: Hottest keys only — configuration data, user session, frequently accessed product data
- **Trade-off**: Each app server has its own L1 → after an update, different servers may return different values for up to 30 seconds (TTL). Acceptable for most read-heavy workloads; NOT suitable for data requiring strong consistency

```
Request flow:
  1. Check L1 (in-process) → hit? Return immediately (< 0.1ms)
  2. L1 miss → Check L2 (Redis) → hit? Return + populate L1 (< 1ms)
  3. L2 miss → Read from Primary DB → populate L2 + L1 (5-50ms)
```

#### Cache Client Library
Embedded in each application server — handles routing, connections, and resilience:

- **Hash slot routing**: Compute `CRC16(key) % 16384` → look up slot-to-node mapping → send directly to the responsible master node. No proxy overhead.
- **Slot map cache**: Client stores the full slot→node mapping locally. Refreshed on MOVED redirects or periodically (every 60 seconds).
- **Connection pooling**: Persistent TCP connections to each cache node (pool of 10-20 connections per node). Avoids TCP handshake overhead per request.
- **Pipelining**: Batch multiple commands into a single network round-trip (e.g., fetch 50 keys in one pipeline call → 50× less latency than sequential GETs)
- **Compression**: For values > 1 KB, compress with LZ4 before storing (reduces memory usage and network bandwidth). Transparent to the caller.
- **Circuit breaker**: Per-node circuit breaker — if a node fails 5 consecutive health checks, stop sending requests to it for 30 seconds (fallback to DB or return stale L1 data). Prevents cascading failures when a cache node is slow/down.
- **Retry with timeout**: 2ms timeout per request + 1 retry on failure. Don't let a slow cache node block the application thread (cache miss to DB is better than hanging).

#### Consistent Hashing — Data Distribution

**Why not simple modulo hashing?**
- `node = hash(key) % N` → when N changes (node added/removed), almost ALL keys are remapped → massive cache miss storm

**Consistent Hashing**:
- Hash ring: Both keys and nodes are hashed to positions on a circular ring (0 to 2^32-1)
- Key assigned to the first node found clockwise from its hash position
- **Adding a node**: Only keys between the new node and its predecessor are remapped (~1/N of keys)
- **Removing a node**: Only that node's keys move to the next node clockwise

**Virtual Nodes**:
- Each physical node has 100-200 virtual nodes spread across the ring
- Ensures even distribution (without vnodes, distribution is uneven)
- On node failure, load is spread across many surviving nodes (not just one neighbor)

```
Ring positions:
  vnode_A1: 1000    vnode_B1: 2500    vnode_A2: 5000
  vnode_C1: 6000    vnode_B2: 7500    vnode_C2: 9000
  
Key "user:123" hashes to 3000 → assigned to vnode_A2 (position 5000)
```

#### In-Memory Data Structure — Hash Map

**Core storage**: Hash table (like Java HashMap or C++ unordered_map)
- **Hash function**: MurmurHash3 or xxHash (fast, low collision)
- **Collision resolution**: Chaining (linked list per bucket) or open addressing
- **Load factor**: Rehash when load factor > 0.75

**Memory layout optimization**:
- Avoid per-entry heap allocations (fragmentation)
- Use **slab allocator** (Memcached approach): Pre-allocate slabs of fixed sizes (64B, 128B, 256B, 512B, 1KB, ...). Each value stored in the smallest slab that fits
- **Jemalloc** (Redis approach): Advanced memory allocator that reduces fragmentation

#### Eviction Policies — Deep Dive

**LRU (Least Recently Used)** ⭐:
- Evict the entry that hasn't been accessed for the longest time
- Implementation: Hash map + Doubly linked list
  - On GET: Move entry to head of list (most recently used)
  - On eviction: Remove entry from tail (least recently used)
  - O(1) for both operations

```
Most Recent ←→ ... ←→ ... ←→ Least Recent (evict this)
  HEAD                           TAIL
```

**Approximated LRU (Redis approach)**:
- True LRU requires maintaining a linked list → memory overhead
- Redis samples 5 random keys → evicts the one with the oldest access time
- Nearly as effective as true LRU with less memory overhead

**LFU (Least Frequently Used)**:
- Evict the entry with the lowest access count
- Better for workloads with varying popularity (some keys are consistently hot)
- Implementation: Morris counter (approximate frequency) with decay over time
- Redis uses LFU with logarithmic counter + decay

**TTL-Based Eviction**:
- **Lazy expiration**: Check TTL on access; if expired, delete and return miss
- **Active expiration**: Background thread periodically samples keys → deletes expired ones
- Redis uses both: lazy + periodic sampling (10 times/sec, sample 20 keys)

#### Replication

**Redis Cluster Replication**:
- Each master node has 1+ replica nodes
- Replication is **asynchronous** (eventual consistency)
- Writes go to master → asynchronously replicated to replicas
- If master fails → replica promoted to master (automatic failover)

**Write flow**: Client → Master → ACK to client → async replicate to replicas
**Read flow**: Reads from master (default) or replicas (for read scaling, with READONLY command)

**Trade-off**: Async replication means a write ACKed by master might be lost if master crashes before replicating. Acceptable for cache (data can be re-fetched from source of truth).

#### Cluster Architecture (Redis Cluster Style)

- **Hash slots**: 16,384 hash slots distributed across masters
- `slot = CRC16(key) % 16384`
- Each master owns a range of slots (e.g., Master A: 0-5460, Master B: 5461-10922, Master C: 10923-16383)
- Client library knows the slot→master mapping → sends directly to correct master
- **MOVED redirect**: If client sends to wrong node, node responds with `MOVED` → client updates its mapping

---

## 5. APIs

### Core Operations
```
GET(key) → value | null
PUT(key, value, ttl_seconds) → OK
DELETE(key) → OK | NOT_FOUND

// Atomic operations
INCR(key) → new_value
DECR(key) → new_value
CAS(key, expected_value, new_value) → OK | CONFLICT
```

### Data Structure Operations (Redis-like)
```
// Hash
HSET(key, field, value)
HGET(key, field) → value
HGETALL(key) → {field: value, ...}

// List
LPUSH(key, value)
RPOP(key) → value
LRANGE(key, start, stop) → [values]

// Set
SADD(key, member)
SMEMBERS(key) → [members]
SISMEMBER(key, member) → bool

// Sorted Set
ZADD(key, score, member)
ZRANGE(key, start, stop) → [members]
ZRANGEBYSCORE(key, min, max) → [members]
```

---

## 6. Data Model

### In-Memory Entry Structure

```c
struct CacheEntry {
    char *key;              // pointer to key string
    char *value;            // pointer to value (or embedded for small values)
    uint32_t key_length;    // 4 bytes
    uint32_t value_length;  // 4 bytes
    uint64_t expiry;        // 8 bytes (Unix timestamp, 0 = no expiry)
    uint64_t last_access;   // 8 bytes (for LRU)
    uint8_t  lfu_counter;   // 1 byte (logarithmic frequency counter)
    struct CacheEntry *prev; // 8 bytes (LRU linked list)
    struct CacheEntry *next; // 8 bytes (LRU linked list)
    // Total overhead per entry: ~50 bytes (excluding key and value)
};
```

### Slab Allocator (Memcached)

```
Slab Classes:
  Class 1:  64-byte chunks   → for values ≤ 64 bytes
  Class 2:  128-byte chunks  → for values ≤ 128 bytes
  Class 3:  256-byte chunks
  Class 4:  512-byte chunks
  Class 5:  1 KB chunks
  ...
  Class 20: 1 MB chunks

Each slab page = 1 MB, divided into chunks of the class size
```

### Cluster Slot Mapping

```
Slot Range    Master Node    Replica Node
0 - 5460      Node A         Node A'
5461 - 10922  Node B         Node B'
10923 - 16383 Node C         Node C'
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Node failure** | Replica promoted to master; client library redirected |
| **Data loss on failure** | Acceptable — cache is a secondary store. Data can be re-populated from primary DB |
| **Cache stampede** | Singleflight pattern: one thread fetches from DB; others wait |
| **Hot key** | Local in-process cache (L1) in front of distributed cache (L2) |
| **Network partition** | Nodes on both sides continue serving. Inconsistency resolved when partition heals |
| **Thundering herd on cold start** | Warm-up script pre-populates cache from DB before accepting traffic |

### Cache Invalidation Strategies

| Strategy | How | When to Use |
|---|---|---|
| **TTL-based** | Key auto-expires after N seconds | Most common; simple |
| **Write-through** | Update cache on every DB write | Strong consistency needed |
| **Write-behind** | Update cache first, async write to DB | Write-heavy, tolerance for stale DB |
| **Cache-aside** ⭐ | App reads from cache; on miss, reads from DB and populates cache | Default strategy |
| **Pub/Sub invalidation** | DB change → publish event → all cache nodes invalidate the key | Multi-node, near-real-time consistency |

---

## 8. Additional Considerations

### Cache Warming Strategies
- **On deploy**: Pre-populate cache with top 1000 most accessed keys from access logs
- **Lazy warming**: Let cache populate naturally from misses (cold start may cause latency spike)
- **Hybrid**: Warm critical keys proactively; let long tail populate lazily

### Monitoring
- **Hit ratio** (target > 90%): `hits / (hits + misses)`
- **Memory usage**: Per node, per slab class
- **Eviction rate**: High eviction rate → need more memory or better TTLs
- **Latency percentiles**: p50, p95, p99 per operation type
- **Connection count**: Per client, per node
- **Replication lag**: Delay between master write and replica receiving it

### Redis vs. Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | Hash, List, Set, Sorted Set, Stream | Only strings |
| Persistence | RDB + AOF | None |
| Replication | Built-in (async) | None (client-side) |
| Clustering | Redis Cluster (hash slots) | Client-side consistent hashing |
| Threading | Single-threaded (6.0+ has I/O threads) | Multi-threaded |
| Memory efficiency | Higher overhead per key | Lower overhead (slab allocator) |
| Use case | Feature-rich caching, pub/sub, leaderboards | Simple high-throughput caching |

### Multi-Level Caching
```
L1: In-process cache (Caffeine/Guava)    → ~1 ms, 100 MB
L2: Distributed cache (Redis)             → ~1-5 ms, 100 GB
L3: CDN cache                             → ~10-50 ms
L4: Database                              → ~10-100 ms
```

### Cache Penetration, Breakdown, and Avalanche

| Problem | Description | Solution |
|---|---|---|
| **Penetration** | Query for key that will NEVER exist → always hits DB | Bloom filter; cache null result with short TTL |
| **Breakdown** | Hot key expires → thousands of requests simultaneously hit DB | Mutex lock (singleflight); never expire hot keys |
| **Avalanche** | Many keys expire at the same time → massive DB load | Jittered TTL (random ±10%); staggered cache warming |

---

## 9. Deep Dive: Engineering Trade-offs

### Write-Through vs Write-Behind vs Cache-Aside: The Critical Pattern Choice

```
Cache-Aside (Lazy Loading) ⭐ (Most Common):
  Read:  App checks cache → miss → read DB → write to cache → return
  Write: App writes to DB → invalidate cache (or do nothing)
  
  ✓ Simple to implement
  ✓ Only requested data is cached (no unnecessary data in cache)
  ✓ Cache failure doesn't prevent reads (just slower — goes to DB)
  ✗ First request is always a cache miss (cold start)
  ✗ Data can become stale if DB is updated without invalidating cache
  Best for: General purpose caching (most applications)

Write-Through:
  Write: App writes to cache → cache synchronously writes to DB → return
  Read:  Always from cache (guaranteed fresh)
  
  ✓ Cache always has the latest data
  ✓ Reads are always fast
  ✗ Every write has double latency (cache + DB)
  ✗ Newly written data may never be read → wasted cache space
  Best for: Read-heavy workloads where consistency is critical

Write-Behind (Write-Back):
  Write: App writes to cache → return immediately → cache async writes to DB
  
  ✓ Lowest write latency (just cache write)
  ✓ Can batch DB writes (1000 cache writes → 1 bulk DB insert)
  ✗ DATA LOSS risk: if cache crashes before async write → data lost forever
  ✗ Complex failure handling
  Best for: Write-heavy workloads where some data loss is acceptable (metrics, counters)

Read-Through:
  Read:  App reads from cache → cache auto-fetches from DB on miss → returns
  (Like cache-aside but cache handles the DB fetch, not the application)
  
  ✓ Application code is simpler
  ✗ Cache must know how to talk to DB (coupling)
```

### Why Single-Threaded Redis Outperforms Multi-Threaded Alternatives

```
Intuition: "Multi-threaded must be faster than single-threaded!"
Reality: For in-memory key-value operations, single-threaded is BETTER.

Why:
  1. No lock contention: Multi-threaded maps need locks (mutex, CAS). 
     Lock contention at 1M ops/sec creates massive overhead.
     Redis: zero locks, zero contention.
  
  2. Memory operations are fast: A GET/SET in memory takes ~100 nanoseconds.
     Thread context switching takes ~1,000 nanoseconds.
     At Redis's workload, context switching costs MORE than the actual work.
  
  3. Network I/O is the bottleneck, not CPU:
     Parsing a network packet takes ~1 microsecond.
     The in-memory operation takes ~100 nanoseconds.
     CPU is idle 90% of the time, waiting for network.
     
     Redis 6.0+ solution: Keep single-threaded for data operations,
     but use multiple I/O threads for network reading/writing.
     This gives the benefits of multi-threading WHERE IT MATTERS
     without any locking on data operations.

When multi-threading helps (Memcached advantage):
  - Very large values (> 1 KB): CPU time for serialization/copying becomes significant
  - Very high connection count (> 100K): I/O thread pool handles more connections
  - Simple GET/SET only: No complex data structures → locking is simpler
```

### Consistent Hashing: Why It's Non-Negotiable

```
Without consistent hashing (simple modulo):
  node = hash(key) % N
  
  N = 3 servers: key "user:123" → hash=7 → 7 % 3 = 1 → Server 1
  
  Add a 4th server (N = 4):
  key "user:123" → hash=7 → 7 % 4 = 3 → Server 3 (MOVED!)
  
  Result: ~75% of all keys are remapped → MASSIVE cache miss storm
  All keys hit the database simultaneously → database crashes
  
With consistent hashing:
  Adding Server 4 only moves ~1/N = 25% of keys
  The other 75% stay on their current servers → no cache miss
  
  Cost of getting this wrong:
    1M keys × 75% remapped × $0.001 per DB query = $750 in DB load
    ... and that's for a small system. At 1B keys, it's catastrophic.
```

### Eviction Policy: LRU vs LFU — When Each Wins

```
LRU (Least Recently Used):
  Evict the key that hasn't been accessed for the longest time
  Assumption: "Recently accessed keys will be accessed again soon"
  
  ✓ Works well for temporal locality (web sessions, recent pages)
  ✗ One-time scan pollutes cache: scanning 1M keys once pushes out hot data
  ✗ Doesn't consider frequency (a key accessed 1000×/day but not in last second 
     gets evicted over a key accessed once 0.5 seconds ago)

LFU (Least Frequently Used):
  Evict the key with the lowest access count
  Assumption: "Frequently accessed keys are more important"
  
  ✓ Better for skewed workloads (80/20 rule: 20% of keys get 80% of requests)
  ✓ One-time scans don't pollute cache
  ✗ "Frequency aging" problem: a key popular last month but not now stays forever
  ✗ New keys start with low frequency → evicted immediately before proving value
  
  Redis's LFU solution:
    - Logarithmic counter (8-bit) → reduces by 50% every 10 minutes
    - New keys start with count = 5 (not 0) → get a fair chance
    - This handles both the aging and cold-start problems

Recommendation:
  - General purpose: LRU (simpler, good enough for most workloads)
  - Mixed traffic with hot spots: LFU (better hit ratio, 5-10% improvement)
  - Redis default: allkeys-lru (Redis lets you configure this easily)
```

### Cache Warming: Cold Start Problem

```
Scenario: Deploy new cache cluster → all empty → 100% miss rate → 
          all traffic hits DB → DB overwhelmed → cascading failure

Solution 1: Lazy warming (do nothing)
  Let misses populate the cache organically
  ✗ 5-30 minutes of high DB load during warmup
  ✗ Latency spike visible to users
  
Solution 2: Pre-warming ⭐
  Before routing traffic to new cache:
  1. Query access logs: identify top 10K most-accessed keys
  2. Fetch those keys from DB → populate cache
  3. Route traffic to the new cache (70%+ hit rate from minute 1)
  
Solution 3: Cache replication
  Instead of cold start, replicate from existing cache cluster
  Redis: SLAVEOF → full sync → promote to master
  ✓ Zero cold start
  ✗ Requires existing cache to be available

Solution 4: Gradual traffic shift
  Route 10% traffic to new cache → warm up → increase to 25% → 50% → 100%
  ✓ Controlled load on DB
  ✗ Slower rollout
```

### The Thundering Herd / Cache Stampede — Deep Dive

```
Timeline:
  T=0:     Hot key "product:123" cached (TTL = 60s)
  T=0-59:  1000 requests/sec → all served from cache ✓
  T=60:    Key expires
  T=60.001: 1000 requests arrive simultaneously → ALL miss cache → ALL hit DB
  T=60.050: DB overwhelmed, response time spikes to 5 seconds
  T=60.100: DB connection pool exhausted → errors for all users
  T=65:    One response comes back → cache populated → cache works again
  
  The damage: 5 seconds of downtime from ONE key expiring.

Solutions:

1. Singleflight / Request Coalescing ⭐
   First request: acquires a mutex lock on the key → fetches from DB → populates cache
   Remaining 999 requests: wait for the lock → get the cached result
   Only 1 DB query instead of 1000.
   
   Implementation (Go singleflight pattern):
     mutex = redis.SET("lock:product:123", "1", NX, EX, 5)
     if mutex acquired:
       data = db.fetch("product:123")
       redis.SET("product:123", data, EX, 60)
       return data
     else:
       sleep(50ms)
       return redis.GET("product:123")  // populated by the lock holder

2. Stale-While-Revalidate
   Cache returns STALE data immediately while fetching fresh data in background
   User gets fast (slightly stale) response; cache refreshes asynchronously
   
3. Jittered TTL
   Instead of TTL = 60s for all keys, use TTL = 60 ± random(0, 10)s
   Keys expire at different times → no synchronized stampede
   
4. Never Expire + Background Refresh
   Set no TTL. Background worker refreshes popular keys every 30 seconds.
   ✗ Stale data for up to 30 seconds
   ✓ Zero cache misses ever
```
