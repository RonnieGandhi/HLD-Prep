# Cache Failure Patterns, Fallbacks & Resilience Techniques

Core concept tested in HLD rounds: "What happens when your cache goes down? How do you prevent a thundering herd from crushing your database?"

---

## 1. Caching Strategies — Foundation

### Cache-Aside (Lazy Loading) ⭐ Most Common

```
READ:
  1. Application checks cache → HIT? Return cached value
  2. MISS → read from DB → write to cache (with TTL) → return value

WRITE:
  1. Write to DB
  2. Invalidate cache (DEL key) — NOT update cache
     (next read will populate cache with fresh data)

Why invalidate, not update?
  • Double-write risk: if cache update fails but DB succeeded → stale forever
  • Race condition: two concurrent writes may update cache out of order
  • Invalidate is idempotent: DEL key is safe to retry
```

### Write-Through

```
WRITE:
  1. Write to cache
  2. Cache synchronously writes to DB
  3. ACK to client only after both succeed

READ:
  Always from cache (guaranteed fresh)

✓ Cache is always consistent with DB
✗ Write latency (two writes on every mutation)
✗ Cache fills with data that may never be read (wasteful)
Used by: DynamoDB Accelerator (DAX)
```

### Write-Behind (Write-Back)

```
WRITE:
  1. Write to cache → ACK to client immediately
  2. Cache asynchronously flushes to DB (batch, every N seconds)

✓ Extremely low write latency
✗ Data loss if cache crashes before flush
✗ Complex consistency guarantees
Used by: CPU caches, some write-heavy systems
```

---

## 2. Cache Failure Scenarios & Solutions

### Scenario 1: Cache Miss Storm (Thundering Herd)

```
Problem:
  Cache entry for popular key expires (TTL)
  100,000 concurrent requests arrive for that key
  ALL hit cache MISS simultaneously
  ALL query the DB simultaneously
  → DB overwhelmed → cascading failure

  Timeline:
    T=0:    cache:popular_key expires
    T=0.001: 100K requests arrive → cache MISS
    T=0.002: 100K SELECT queries hit DB
    T=0.1:   DB connection pool exhausted → errors cascade
```

**Solutions**:

```
1. Request Coalescing (Singleflight) ⭐:
   Only ONE of the 100K requests actually queries the DB
   The other 99,999 WAIT for that one result
   When result arrives → serve all 100K from that single DB read
   
   Implementation (Go singleflight / Java Guava cache loading):
     lock = acquire_lock("loading:popular_key")
     if lock acquired:
       value = db.query(key)
       cache.set(key, value, ttl=300)
       release_lock()
     else:
       wait_for_lock("loading:popular_key", timeout=5s)
       value = cache.get(key)  # now populated by the winner
   
   ✓ DB sees exactly 1 query instead of 100K
   ✗ Adds latency for the waiters (but better than DB crash)

2. Stale-While-Revalidate:
   Cache returns the STALE (expired) value immediately
   Simultaneously triggers a background refresh
   Next request gets the fresh value
   
   Implementation:
     cache.set(key, value, ttl=300, stale_ttl=600)
     # ttl=300: after 5min, value is "stale"
     # stale_ttl=600: stale value kept for 10min as fallback
     
     on GET:
       if fresh → return
       if stale → return stale AND trigger async refresh
       if gone  → synchronous DB read (normal miss)
   
   ✓ Zero-latency reads even on expiry
   ✓ No thundering herd (only 1 background refresh)
   ✗ Briefly serves stale data (acceptable for most use cases)
   Used by: CDNs (stale-while-revalidate header), Varnish

3. Pre-emptive Cache Warming:
   Refresh cache BEFORE TTL expires
   Background job refreshes keys at 80% of TTL
   
   Example: TTL = 5 minutes → refresh job runs at 4 minutes
   Cache never expires → no miss storm
   
   ✓ Prevents expiry entirely
   ✗ Extra load on DB (refreshing things that may not be needed)
   ✗ Doesn't help with first-time population (cold cache)

4. Randomized TTL (Jitter):
   Instead of all keys expiring at exactly TTL, add random jitter:
     ttl = base_ttl + random(0, base_ttl * 0.1)  # ±10% jitter
   
   Prevents synchronized mass expiry (many keys expiring at same second)
   ✓ Simple, prevents correlated expiry
   ✗ Doesn't help with single hot key
```

### Scenario 2: Cache Stampede (Hot Key Expiry)

```
Problem: One specific key is extremely hot (celebrity profile, trending topic)
  1M+ reads/sec on this single key
  Key expires → 1M requests hit DB for same key
```

**Solutions**:

```
1. Lock-Based Loading (Mutex):
   On cache miss: try to acquire a distributed lock
   Winner: queries DB, populates cache, releases lock
   Losers: spin-wait or return a fallback value
   
   SET cache_lock:hot_key 1 NX EX 10  # 10 second lock
   if acquired → query DB → set cache → DEL lock
   else → sleep(50ms) → retry cache.get()

2. Probabilistic Early Expiration (XFetch):
   Each cached value stores its TTL + a "should I refresh?" probability
   As the key approaches expiry, a randomly selected reader refreshes it
   
   Algorithm:
     remaining_ttl = expiry_time - now
     if remaining_ttl < random() * beta * compute_time:
       refresh()  # only ~1 reader does this
   
   ✓ No locks, no coordination
   ✓ Statistically only 1 reader refreshes
   ✗ Slightly complex probability tuning

3. Never-Expire + Async Refresh:
   Set TTL = INFINITE (no expiry)
   Use Kafka CDC or a periodic job to push updates to cache
   Cache always has data → never a miss storm
   
   ✓ Zero miss storms
   ✗ Memory usage (never evicted by TTL)
   ✗ Staleness until async update arrives
```

### Scenario 3: Full Cache Down (Redis Crash)

```
Problem: Redis cluster is completely unavailable
  ALL reads become cache misses → ALL go to DB
  DB was sized for 5% of traffic (cache hit rate = 95%)
  Now receives 100% of traffic → instant DB overload
```

**Solutions**:

```
1. Circuit Breaker Pattern ⭐:
   When cache errors exceed threshold → trip the circuit breaker
   While circuit is OPEN:
     • Don't attempt cache reads (fail fast, don't waste time on timeouts)
     • Route to DB with HEAVY rate limiting
     • Return degraded responses (cached local data, default values, "try later")
   Periodically probe cache (half-open state)
   When cache responds → close circuit → resume normal operation
   
   States:
     CLOSED → normal operation (try cache first)
     OPEN   → cache is down (skip cache, go to DB with limits, or serve stale)
     HALF-OPEN → testing if cache is back (send 1 probe request)

2. L1 In-Process Cache (Multi-Layer):
   Each app server maintains a small LOCAL cache (Caffeine/Guava):
     L1: In-process (100 MB, 30s TTL) → < 0.1ms
     L2: Redis (shared, large) → < 1ms
     L3: DB → 5-50ms
   
   If L2 (Redis) is down:
     L1 absorbs most hot-key reads
     Only L1 misses hit DB
     Even 50% L1 hit rate cuts DB load in half
   
   ✓ Automatic fallback, no special handling
   ✗ L1 staleness (30s TTL means stale for up to 30s)
   ✗ L1 is per-process → no dedup across servers

3. Request Rate Limiting / Shedding:
   If cache is down AND DB is overloaded:
     Accept only P% of traffic (start at 20%, increase as DB stabilizes)
     Reject rest with 503 + Retry-After header
     Priority: authenticated users > anonymous, paid > free
   
   ✓ Protects DB from total collapse
   ✗ Degrades user experience (503 errors)

4. Graceful Degradation Responses:
   For non-critical data: return a DEFAULT or CACHED response
   
   Examples:
     Product recommendations → return "Popular products" (pre-computed, stored locally)
     User feed → show "Trending" feed (pre-computed) instead of personalized
     Like count → show "1K+" instead of exact count
     User avatar → show default avatar
   
   ✓ User sees something (not an error page)
   ✗ Reduced functionality

5. Warm-Up Strategy After Cache Recovery:
   When Redis comes back online, don't let all traffic rush to it
   Gradually route traffic: 10% → 25% → 50% → 100%
   Meanwhile, a background job pre-populates hot keys:
     SELECT * FROM products WHERE view_count > 10000 → bulk SET into Redis
```

### Scenario 4: Cache Penetration (Non-Existent Keys)

```
Problem: Requests for keys that DON'T EXIST in DB either
  Cache miss → DB query → empty result → nothing to cache
  Next request → cache miss again → DB query again
  Attacker can exploit this: request random non-existent IDs → bypass cache → hammer DB
```

**Solutions**:

```
1. Cache Null Results:
   If DB returns empty → cache the NULL with a SHORT TTL
     cache.set("user:99999", NULL, ttl=60)
   Next request → cache HIT (null) → return 404 immediately
   
   ✓ Simple, effective
   ✗ Fills cache with null entries (use short TTL)

2. Bloom Filter Gate:
   Before checking cache or DB, check a Bloom filter:
     "Does this key POSSIBLY exist?"
     If Bloom says NO → key definitely doesn't exist → return 404 (skip cache + DB)
     If Bloom says MAYBE → proceed with cache/DB lookup
   
   Bloom filter size: 1B keys × 10 bits/key = ~1.2 GB (fits in memory)
   False positive rate: ~1% (acceptable)
   
   ✓ Blocks most non-existent key attacks
   ✗ Must keep Bloom filter in sync with DB (update on insert/delete)

3. Input Validation:
   Validate key format before lookup: is user_id a valid UUID? Is it within expected range?
   Reject obviously invalid keys at the API layer
   
   ✓ Zero cost
   ✗ Doesn't help with valid-format but non-existent keys
```

### Scenario 5: Cache Avalanche (Mass Expiry)

```
Problem: Many cache keys expire at the SAME TIME
  (e.g., bulk-loaded with same TTL, or cache restarted and refilled simultaneously)
  → Thousands of cache misses at once → DB spike
```

**Solutions**:

```
1. TTL Jitter (most important):
   ttl = base_ttl + random(0, base_ttl * 0.2)
   Spreads expirations across a time window
   
2. Staggered Warm-Up:
   After cache restart, don't load everything at once
   Load in batches: 1000 keys/second over 30 minutes
   
3. Multi-Layer Cache:
   L1 (local, 30s TTL) absorbs the first wave of misses
   Even if L2 (Redis) mass-expires, L1 still serves hot keys
```

---

## 3. Cache Invalidation — "The Two Hard Problems"

```
"There are only two hard things in Computer Science:
 cache invalidation and naming things." — Phil Karlton
```

### Invalidation Strategies

```
1. TTL-Based Expiry (Time-Based):
   Set TTL on every cache entry
   After TTL: entry is automatically deleted
   
   ✓ Simple, self-healing (stale data eventually disappears)
   ✗ Stale for up to TTL duration
   ✗ Choose TTL carefully:
     Too short → low hit rate, high DB load
     Too long → stale data, poor UX
   
   Typical TTLs:
     User profile: 5 minutes
     Product catalog: 1 hour
     Configuration: 5 minutes
     Static content: 24 hours
     Session data: 30 minutes

2. Event-Based Invalidation (Kafka CDC) ⭐:
   DB write → Debezium captures change → Kafka event → cache consumer DELetes key
   
   ✓ Near-real-time invalidation (< 2 second lag)
   ✓ Works even when the write came from a different service
   ✗ Requires CDC infrastructure (Debezium, Kafka)
   ✗ Eventual consistency (brief window of stale data)

3. Write-Through Invalidation:
   Application writes to DB AND deletes/updates cache in the same code path
   
   ✓ Immediate invalidation
   ✗ If cache DEL fails → stale forever (unless TTL saves you)
   ✗ Only works for writes going through YOUR application (not direct SQL, migrations)

4. Version-Based (Cache Busting):
   Include a version in the cache key: product:{id}:v{version}
   On update: increment version → new key → old key expires naturally
   
   ✓ No race conditions (new version is a new key)
   ✗ Wastes memory (old versions linger until TTL)
```

### The Double-Delete Problem

```
Problem: Race condition between cache invalidation and concurrent reads

Thread A (writer):                   Thread B (reader):
  1. Write to DB (new value)
                                      2. Read from cache → MISS
                                      3. Read from DB → gets OLD value (replication lag!)
  4. Delete from cache
                                      5. Write OLD value to cache
  → Cache now has STALE data that won't expire until TTL!

Solution: Delayed Double-Delete
  1. Delete from cache
  2. Write to DB
  3. Sleep(500ms)  ← wait for any in-flight reads to complete
  4. Delete from cache AGAIN
  
  The second delete catches any stale value written by step 5 above
  
  ✓ Handles the race condition
  ✗ Adds 500ms to write path (can be done async)
```

---

## 4. Cache Consistency Patterns

### Read-Heavy System (95% reads)

```
Pattern: Cache-aside + TTL + Event-based invalidation

Write: DB → Kafka CDC → cache DEL
Read: Cache → if miss → DB → cache SET (with TTL as safety net)

Consistency: Eventual (stale window = CDC lag, typically < 2s)
Acceptable for: product catalog, user profiles, social feeds
```

### Write-Heavy System (50%+ writes)

```
Pattern: Write-behind cache + periodic DB flush

Write: Cache (immediate ACK) → async batch write to DB every 5s
Read: Always from cache

Risk: Data loss if cache crashes before flush
Mitigation: Redis AOF (append-only file) with fsync=everysec
Acceptable for: view counters, analytics events, session data
```

### Strong Consistency Required

```
Pattern: Don't cache OR use read-through with write-through

Option A: No cache (read from DB directly)
  DB handles consistency → simple, correct
  Scale DB with read replicas + connection pooling

Option B: Distributed cache with invalidation lock
  On write: acquire lock → write DB → update cache → release lock
  On read: acquire shared lock → read cache (or DB on miss)
  
  ✓ Linearizable
  ✗ Lock contention kills throughput
  ✗ Distributed locks are complex and fragile

Acceptable for: Financial balances, inventory counts, booking systems
```

---

## 5. Production Resilience Checklist

```
□ Circuit breaker on cache client (trip on 5 consecutive failures)
□ Timeout on cache reads (2ms — if cache is slow, skip it and go to DB)
□ L1 in-process cache for hottest keys (Caffeine, 100MB, 30s TTL)
□ TTL jitter on all cache entries (±10-20%)
□ Request coalescing / singleflight for hot keys
□ Bloom filter for non-existent key protection
□ Cache null results with short TTL (60s)
□ Graceful degradation responses (defaults, popular items, "try later")
□ Cache warm-up procedure (batch pre-populate hot keys after restart)
□ Monitoring: hit rate, miss rate, latency p99, eviction rate, memory usage
□ Alert on: hit rate drop > 10%, cache latency p99 > 5ms, eviction spike
□ Gradual traffic ramp-up after cache recovery (10% → 25% → 50% → 100%)
```

