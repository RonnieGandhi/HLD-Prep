# 2. Design an API Rate Limiter

---

## 1. Functional Requirements (FR)

- Limit the number of requests a client can make to an API within a given time window
- Support multiple rate limiting rules (e.g., per user, per IP, per API key, per endpoint)
- Support configurable rules: X requests per Y seconds/minutes/hours
- Return meaningful error responses (HTTP 429) with retry-after header when limit is exceeded
- Support different tiers of rate limits (free, premium, enterprise)
- Provide a way to whitelist certain clients (internal services)
- Support both hard limits (reject) and soft limits (log warning, allow)
- Dashboard to view rate limiting metrics and adjust rules in real-time

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: Rate check must add < 1 ms overhead to each request
- **High Availability**: Rate limiter failure should not block legitimate traffic (fail-open)
- **Distributed**: Work correctly across multiple servers/data centers (global rate limiting)
- **Accuracy**: Minimal over-counting or under-counting in distributed setting
- **Scalability**: Handle millions of requests/sec across all clients
- **Fault Tolerant**: Gracefully degrade if the rate limiting backend is unavailable
- **Atomic Operations**: Increment + check must be atomic to prevent race conditions

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total API requests/sec | 1M |
| Unique clients | 10M |
| Rate limit rules | ~100 rules |
| Redis entry per client-rule | ~100 bytes |
| Memory for active clients (1M active) | 1M × 100 bytes × 5 rules = ~500 MB |
| Redis operations/sec | 2M (1 read + 1 write per request) |

Rate limiting metadata is tiny — the main concern is **throughput** (ops/sec), not storage.

---

## 4. High-Level Design (HLD)

```
                              ┌──────────────┐
                              │   Clients    │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │    L7 Load   │
                              │   Balancer   │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │ Rate Limiter │  ← Middleware / Sidecar
                              │  Middleware  │
                              └──────┬───────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
             ┌──────▼──────┐ ┌──────▼──────┐  ┌──────▼──────┐
             │ API Server  │ │ API Server  │  │ API Server  │
             │  Instance 1 │ │  Instance 2 │  │  Instance N │
             └─────────────┘ └─────────────┘  └─────────────┘
                    │                │                 │
                    └────────────────┼─────────────────┘
                                     │
                              ┌──────▼───────┐
                              │    Redis     │
                              │   Cluster    │  ← Centralized counters
                              │ (Rate State) │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Rules DB    │  ← Rate limit configurations
                              │ (MySQL/Cfg)  │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Rules       │
                              │  Config Svc  │  ← Admin dashboard to manage rules
                              └──────────────┘
```

### Component Deep Dive

#### Rate Limiter Middleware
- **Where it runs**: As a middleware in the API Gateway or as a sidecar proxy (e.g., Envoy filter)
- **Why middleware**: Every request passes through it before reaching the API server. Centralized enforcement
- **Flow**:
  1. Extract client identifier (API key, user ID, IP)
  2. Look up applicable rate limit rules (from local cache, refreshed every 30s from Rules DB)
  3. Check counter in Redis: `INCR` + `EXPIRE` atomically
  4. If under limit → allow, pass `X-RateLimit-Remaining` header
  5. If over limit → reject with `429 Too Many Requests` + `Retry-After` header

#### Redis Cluster (Rate State Store)
- **Why Redis**:
  - In-memory → sub-millisecond latency
  - Atomic operations (`INCR`, `EXPIRE`) → no race conditions
  - Built-in TTL → counters auto-expire
  - Single-threaded model → no locking needed for atomic ops
- **Deployment**: Redis Cluster with 6 nodes (3 masters + 3 replicas) for HA
- **Data Model**: Key = `ratelimit:{client_id}:{endpoint}:{window}`, Value = counter

#### Rules Configuration Service
- **Purpose**: Admin UI/API to create, update, delete rate limiting rules
- **Rules stored in**: MySQL (persistent) + cached in-memory on each API Gateway instance
- **Rule format**: YAML/JSON defining limits per client tier, per endpoint
- **Push-based updates**: When a rule changes, publish to a Kafka topic. All gateway instances subscribe and update their local cache

---

## 5. Rate Limiting Algorithms — Deep Dive

### Algorithm 1: Token Bucket ⭐ (Most Common)
**How it works**:
- Each client has a "bucket" with a max capacity of `N` tokens
- Tokens are added at a fixed rate of `R` tokens/sec
- Each request consumes 1 token
- If bucket is empty → reject

**Pros**: Allows bursts (up to bucket capacity), smooth rate limiting  
**Cons**: Two parameters to tune (bucket size + refill rate)

```
Redis Implementation:
  Key: ratelimit:token:{client_id}
  Value: {tokens: 8, last_refill: 1710320000}
  
  Lua Script (atomic):
    local tokens = get current tokens
    local elapsed = now - last_refill
    tokens = min(max_tokens, tokens + elapsed * refill_rate)
    if tokens >= 1 then
      tokens = tokens - 1
      return ALLOWED
    else
      return REJECTED
```

### Algorithm 2: Sliding Window Log
**How it works**:
- Store timestamp of every request in a sorted set
- On new request, remove all entries older than the window
- Count remaining entries; if < limit → allow

**Pros**: Exact, no boundary issues  
**Cons**: Memory-intensive (stores every timestamp)

```
Redis: ZSET per client
  ZREMRANGEBYSCORE key 0 (now - window)
  ZADD key now now
  ZCARD key → count
```

### Algorithm 3: Sliding Window Counter ⭐ (Best Balance)
**How it works**:
- Combine fixed window counters with weighted overlap
- `effective_count = prev_window_count × overlap_percentage + current_window_count`
- Example: window = 1 min, at 0:30 → effective = prev × 0.5 + current

**Pros**: Memory efficient (two counters), smooth, no boundary spikes  
**Cons**: Approximate (but very close to exact)

```
Redis:
  Key prev: ratelimit:{client}:{endpoint}:202603131000
  Key curr: ratelimit:{client}:{endpoint}:202603131001
  
  weight = (60 - seconds_into_current_window) / 60
  count = prev_count × weight + curr_count
```

### Algorithm 4: Fixed Window Counter
**How it works**:
- Counter per fixed time window (e.g., 0:00-1:00, 1:00-2:00)
- `INCR key; if count > limit → reject`

**Pros**: Simple, memory efficient  
**Cons**: Boundary problem — a burst at 0:59 and 1:01 allows 2× the limit in a 2-second span

### Algorithm 5: Leaky Bucket
**How it works**:
- Requests enter a FIFO queue (the "bucket")
- Processed at a fixed rate
- If queue is full → reject

**Pros**: Smooth output rate  
**Cons**: Burst traffic fills the queue; old requests may time out

### Recommendation
| Use Case | Algorithm |
|---|---|
| General API rate limiting | **Sliding Window Counter** |
| APIs that need burst tolerance | **Token Bucket** |
| Strict rate enforcement | **Sliding Window Log** |

---

## 6. APIs

### Check Rate Limit (Internal)
```
// Middleware call — not exposed externally
checkRateLimit(client_id, endpoint, timestamp) → {allowed: bool, remaining: int, retry_after: int}
```

### Get Rate Limit Status
```http
GET /api/v1/rate-limit/status
Authorization: Bearer <token>

Response: 200 OK
{
  "client_id": "api_key_123",
  "limits": [
    {
      "endpoint": "/api/v1/search",
      "limit": 100,
      "window": "1m",
      "remaining": 42,
      "resets_at": "2026-03-13T10:01:00Z"
    }
  ]
}
```

### Rate Limit Response Headers (on every API response)
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1710320460
```

### Rejected Response
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 30 seconds."
}
```

### Configure Rule (Admin)
```http
POST /api/v1/admin/rate-rules
{
  "rule_id": "search_free_tier",
  "client_tier": "free",
  "endpoint": "/api/v1/search",
  "max_requests": 100,
  "window_seconds": 60,
  "algorithm": "sliding_window_counter"
}
```

---

## 7. Data Model

### Redis — Rate Counters (Sliding Window Counter)

```
Key:    ratelimit:{client_id}:{endpoint}:{window_start_epoch}
Value:  integer (counter)
TTL:    2 × window_size (keep previous window for weighted calculation)

Example:
  ratelimit:user123:/api/search:1710320400 → 45
  ratelimit:user123:/api/search:1710320460 → 12
```

### Redis — Token Bucket State

```
Key:    ratelimit:token:{client_id}:{endpoint}
Value:  Hash { tokens: float, last_refill_ts: epoch }
TTL:    window_size + buffer
```

### MySQL — Rate Limit Rules

```sql
CREATE TABLE rate_limit_rules (
    rule_id         VARCHAR(64) PRIMARY KEY,
    client_tier     ENUM('free', 'premium', 'enterprise', 'internal'),
    endpoint_pattern VARCHAR(256),     -- regex: /api/v1/search.*
    max_requests    INT NOT NULL,
    window_seconds  INT NOT NULL,
    algorithm       ENUM('token_bucket', 'sliding_window_counter', 'sliding_window_log'),
    action          ENUM('reject', 'log_only', 'throttle'),
    enabled         BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    INDEX idx_tier_endpoint (client_tier, endpoint_pattern)
);
```

### MySQL — Client Tier Mapping

```sql
CREATE TABLE client_tiers (
    client_id   VARCHAR(128) PRIMARY KEY,
    tier        ENUM('free', 'premium', 'enterprise', 'internal'),
    custom_limits JSON,    -- override specific rules
    whitelisted  BOOLEAN DEFAULT FALSE,
    created_at   TIMESTAMP
);
```

---

## 8. Fault Tolerance

### General
| Technique | Details |
|---|---|
| **Fail-Open** | If Redis is unreachable, allow the request (don't block legitimate traffic). Log for monitoring |
| **Local Fallback** | Each API server maintains a local in-memory counter as fallback. Less accurate but functional |
| **Redis Cluster HA** | 3 masters + 3 replicas; automatic failover in < 15 seconds |
| **Circuit Breaker** | If Redis latency > 5ms for > 10 consecutive calls, trip the breaker → fall back to local counting |

### Problem-Specific Fault Tolerance

1. **Race Conditions in Distributed Setting**
   - Multiple API servers increment the same counter concurrently
   - **Solution**: Redis Lua scripts execute atomically (single-threaded Redis). The entire check-and-increment is one atomic operation
   - Alternative: Use Redis `MULTI/EXEC` transactions

2. **Clock Skew Across Servers**
   - Different servers have slightly different clocks → window boundaries differ
   - **Solution**: Use Redis server time (`TIME` command) as the authoritative clock, not local server time

3. **Redis Data Loss (Failover)**
   - When a Redis master fails, the replica may not have the latest writes
   - **Impact**: Some clients get a few extra requests through during failover
   - **Mitigation**: Acceptable trade-off (over-allow briefly vs. blocking legitimate traffic)

4. **Global Rate Limiting Across Data Centers**
   - A user with 100 req/min limit hitting both US and EU data centers could get 200 req/min
   - **Solutions**:
     - **Centralized Redis**: All DCs talk to one Redis (adds latency)
     - **Sync-based**: Each DC tracks locally, periodically syncs to a global counter via Kafka (slight over-allow during sync gap)
     - **Sticky routing**: GeoDNS routes a user always to the same DC (simplest, but less resilient)
   - **Recommendation**: Sticky routing for most cases; sync-based for strict global limits

5. **Hot Client (Single Heavy User)**
   - One client sending millions of requests → hot key in Redis
   - **Solution**: Shard by `client_id + endpoint` → distributes across Redis cluster. For extreme cases, use local counters with periodic sync

---

## 9. Additional Considerations

### Rate Limiting at Multiple Levels
```
Level 1: Network/IP level     → Nginx connection limits, iptables
Level 2: API Gateway level    → Per API key, per endpoint (this design)
Level 3: Application level    → Per user, per resource (e.g., max 10 posts/day)
Level 4: Infrastructure level → Circuit breaker, bulkheads
```

### Distributed Rate Limiting with Redis Lua Script (Token Bucket)
```lua
-- Atomic token bucket in Redis
local key = KEYS[1]
local max_tokens = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or max_tokens
local last_refill = tonumber(data[2]) or now

-- Refill tokens
local elapsed = now - last_refill
tokens = math.min(max_tokens, tokens + elapsed * refill_rate)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, max_tokens / refill_rate * 2)
    return {1, tokens}  -- allowed, remaining
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    local retry_after = (1 - tokens) / refill_rate
    return {0, retry_after}  -- rejected, retry_after seconds
end
```

### Monitoring & Alerting
- Total requests allowed vs. rejected (ratio should be healthy)
- P99 latency added by rate limiter (target < 1ms)
- Redis memory usage and operation throughput
- Alert when a client consistently hits their limit (potential abuse or need for tier upgrade)
- Dashboard showing top rate-limited clients and endpoints

---

## 10. Deep Dive: Engineering Trade-offs

### Where Should the Rate Limiter Live?

| Placement | Pros | Cons | When to Use |
|---|---|---|---|
| **API Gateway (centralized)** ⭐ | Single enforcement point; consistent; easy to manage rules | Gateway becomes bottleneck; adds latency to every request | External-facing APIs; coarse-grained limits |
| **Application middleware** | Access to user context (auth, session); fine-grained rules | Must be implemented in every service; inconsistent enforcement | Per-service, business-logic-aware limits |
| **Sidecar proxy (Envoy/Istio)** | Decoupled from application; works for any language | Operational complexity of service mesh; limited rule expressiveness | Microservice architectures; east-west traffic |
| **Client-side** | Reduces unnecessary requests | Easily bypassed; not trustworthy | As *advisory* companion to server-side enforcement |
| **Network layer (iptables, WAF)** | Extremely fast; kernel-level | Very coarse (IP-only); can't distinguish authenticated users | DDoS mitigation; IP-level blocking |

**Best practice**: Layer them. Network-level DDoS protection → API Gateway coarse limits → Application middleware fine-grained limits.

### Why Redis Specifically? Alternatives Compared

| Solution | Latency | Throughput | Atomicity | Persistence | Distributed |
|---|---|---|---|---|---|
| **Redis** ⭐ | < 0.5 ms | 1M+ ops/sec | Lua scripts = atomic | Optional (RDB/AOF) | Redis Cluster |
| **Memcached** | < 0.5 ms | 1M+ ops/sec | CAS only (no Lua) | None | Client-side sharding |
| **In-memory (local)** | < 0.01 ms | Unlimited | Thread-safe map | None | Not distributed |
| **Database (PostgreSQL)** | ~5 ms | ~10K ops/sec | Transactions | Yes | Single-writer |

**Why Redis wins**: (1) Lua scripts execute atomically — the entire "check + increment + expire" is one operation with zero race condition risk. Memcached can't do this. (2) Built-in TTL handles window expiry automatically. (3) Redis Cluster gives us distributed operation. (4) If Redis is down, we fail-open (allow requests) — cache loss is not catastrophic for rate limiting.

**When local in-memory is better**: If you only need per-server limits (not global), a thread-safe map with periodic cleanup is the fastest option and has zero network dependency. Example: Guava `RateLimiter` in Java.

### Fail-Open vs Fail-Closed: The Critical Design Decision

```
Fail-Open (allow requests when rate limiter is unavailable):
  ✓ Legitimate users are never blocked by infrastructure failures
  ✓ Higher availability for the overall system
  ✗ During outage, no rate limiting → abuse is possible
  ✗ Downstream services may be overwhelmed
  
  When to use: Most APIs — user experience > abuse prevention
  
Fail-Closed (reject requests when rate limiter is unavailable):
  ✓ Guaranteed rate enforcement even during failures
  ✗ Infrastructure failure blocks ALL users — outage
  ✗ Violates the "rate limiter should not reduce availability" principle
  
  When to use: Financial APIs, security-critical endpoints where abuse 
               would cause more damage than downtime
```

**Recommendation**: Fail-open for 95% of use cases. Pair with monitoring to detect and respond to abuse during limiter outages.

### Algorithm Selection Framework (Decision Tree)

```
Q: Do you need to allow short bursts?
  YES → Token Bucket (burst = bucket size)
  NO  → Fixed Window Counter (simplest)

Q: Do you need smooth, predictable rate enforcement?
  YES → Leaky Bucket (constant output rate)
  NO  → Token Bucket is fine

Q: Is the "boundary burst" problem unacceptable?
  (2× allowed rate at window boundary with Fixed Window)
  YES → Sliding Window Counter or Sliding Window Log
  NO  → Fixed Window Counter

Q: Is memory a concern? (millions of unique clients)
  YES → Sliding Window Counter (2 integers per client per window)
  NO  → Sliding Window Log (stores every timestamp — O(rate_limit) per client)
```

**Industry usage**: Stripe uses Token Bucket. GitHub uses Sliding Window. Cloudflare uses Fixed Window with sub-second windows. Google Cloud uses Token Bucket with configurable burst.

### The Distributed Rate Limiting Accuracy Problem

When you have N API servers and a centralized Redis:
```
Scenario: 100 req/min limit, 10 API servers

Ideal: All 10 servers check the same Redis counter → accurate global count

Reality:
  - Network latency to Redis: 0.5-2ms per check
  - Under heavy load: Redis becomes bottleneck
  - Optimization: batch increments locally, sync to Redis periodically

Trade-off spectrum:
  Accurate (every request checks Redis) ◄─────────► Fast (local counters, periodic sync)
  
  Stripe's approach: "Approximate" — each server tracks locally, 
  syncs to Redis every 100ms. May over-allow by up to 10% during sync gaps.
  This is acceptable because rate limits are already approximate by nature.
```

### Rate Limit Headers: The Full Standard (RFC 6585 + Draft)

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100            # max requests in window
RateLimit-Remaining: 42         # requests left in current window
RateLimit-Reset: 1710320460     # Unix timestamp when window resets
RateLimit-Policy: 100;w=60      # 100 per 60-second window

HTTP/1.1 429 Too Many Requests
Retry-After: 30                 # seconds until client should retry
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1710320460
```

Including these headers on EVERY response (not just 429) is a best practice — it lets well-behaved clients self-throttle before hitting the limit.
