# 1. Design a URL Shortener (TinyURL / Bit.ly)

---

## 1. Functional Requirements (FR)

- Given a long URL, generate a unique, short URL (e.g., `https://short.ly/xK9b2`)
- Given a short URL, redirect the user to the original long URL (HTTP 301/302)
- Users can optionally provide a custom alias for their short URL
- Short URLs have a configurable expiration (default: 5 years)
- Analytics: track total clicks, geographic data, referrer, device info
- Users can delete their own short URLs
- Duplicate long URLs by the same user return the existing short URL

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability** (99.99%): Redirects must always work
- **Low Latency**: Redirect in < 10 ms p99
- **Read-Heavy**: Read:Write ratio ≈ 100:1
- **Scalability**: Billions of stored URLs, 100K+ reads/sec
- **Durability**: Once created, a URL must never be lost before its expiry
- **Uniqueness**: No two different long URLs should get the same short code (collision-free)
- **Eventual Consistency acceptable** for analytics; strong consistency for URL creation

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| New URLs / month | Given | 100M |
| Redirects / month | 100:1 read:write | 10B |
| Writes / sec | 100M / (30 × 86400) | ~38 writes/s |
| Reads / sec | 10B / (30 × 86400) | ~3,800 reads/s (peak 5×: ~19K) |
| Record size | short_code + long_url + metadata | ~500 bytes |
| Storage / year | 100M × 12 × 500B | ~600 GB |
| 5-year storage | | ~3 TB |
| Cache size (20% hot) | 0.2 × daily_reads × 500B = 0.2 × 333M × 500B | ~33 GB |

**Short Code Length**: Base62 encoding (a-z, A-Z, 0-9).  
- 6 chars → 62^6 ≈ 56.8 billion (sufficient)  
- 7 chars → 62^7 ≈ 3.5 trillion (future-proof)  
- **Choose 7 characters** for safety margin.

---

## 4. High-Level Design (HLD)

```
                              ┌─────────────┐
                              │  Clients    │
                              │ (Browser/App)│
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │    CDN      │  ← Cache 301 redirects at edge
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │   DNS +     │
                              │  Global LB  │  ← GeoDNS routes to nearest DC
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │ API Gateway │  ← Rate limiting, auth, SSL termination
                              └──────┬──────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
       ┌──────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
       │ URL Write   │       │ URL Read    │       │ Analytics   │
       │ Service     │       │ Service     │       │ Service     │
       └──────┬──────┘       └──────┬──────┘       └──────┬──────┘
              │                      │                      │
       ┌──────▼──────┐              │               ┌──────▼──────┐
       │   Key Gen   │              │               │   Kafka     │
       │   Service   │              │               │ (click-     │
       │   (KGS)     │              │               │  events)    │
       └──────┬──────┘       ┌──────▼──────┐       └──────┬──────┘
              │              │    Redis    │               │
              │              │   Cluster   │       ┌──────▼──────┐
              │              │   (Cache)   │       │ Apache Flink│
              │              └──────┬──────┘       │ (Stream     │
              │                     │               │ Processing) │
              └─────────┬──────────┘               └──────┬──────┘
                        │                                  │
                 ┌──────▼──────┐                   ┌──────▼──────┐
                 │  Cassandra  │                   │ ClickHouse  │
                 │  (URL Store)│                   │ (Analytics) │
                 └─────────────┘                   └─────────────┘
```

### Component Deep Dive

#### API Gateway
- **Why**: Centralized entry for rate limiting (token bucket per API key), JWT auth, SSL termination, request routing
- **How**: NGINX / Kong / AWS API Gateway. Routes `/api/v1/urls` to write service, `/{short_code}` to read service
- **Fault Tolerance**: Multiple instances behind an L4 load balancer; stateless, so horizontally scalable

#### Key Generation Service (KGS)
- **Why**: Pre-generating keys avoids real-time collision checks, which are expensive at scale
- **How it works**:
  1. Offline process generates all possible 7-char Base62 keys and stores them in a key-DB (two tables: `unused_keys` and `used_keys`)
  2. Each app server requests a **batch** (e.g., 1,000 keys) from KGS
  3. KGS atomically moves keys from `unused` → `used` and hands them to the requesting server
  4. App server serves keys from its in-memory batch (no DB hit per request)
  5. When a server dies, its unused in-memory keys are "lost" — acceptable since we have trillions of keys
- **Coordination**: ZooKeeper or etcd assigns non-overlapping ranges to KGS instances
- **Fault Tolerance**: Multiple KGS replicas with leader election; if leader fails, follower takes over with a fresh range

#### URL Write Service
- **Flow**:
  1. Receive long URL + optional custom alias
  2. If custom alias → check uniqueness via Bloom filter (fast negative) then DB conditional write
  3. If no alias → pop a key from the local KGS batch
  4. Write `{short_code → long_url, user_id, expiry}` to Cassandra
  5. Populate Redis cache
  6. Return short URL

#### URL Read Service (Redirect Service)
- **Flow**:
  1. Receive GET `/{short_code}`
  2. Check Redis cache → if hit, redirect immediately
  3. If cache miss → query Cassandra → populate cache → redirect
  4. Async: publish click event to Kafka
- **301 vs 302**:
  - **301** (Permanent Redirect): Browser caches it → fewer hits to our servers, but we lose per-click analytics
  - **302** (Temporary Redirect): Every click hits our servers → accurate analytics
  - **Recommendation**: Use **302** if analytics are important; 301 for pure shortening

#### Redis Cluster (Cache Layer)
- **Why Redis**: In-memory, sub-millisecond latency, supports TTL natively
- **Strategy**: Cache-aside pattern (read-through). LRU eviction policy
- **Data**: `key=url:{short_code}`, `value=long_url`, `TTL=24h`
- **Size**: ~33 GB fits in a single large Redis instance or a small cluster
- **Fault Tolerance**: Redis Cluster mode (6 nodes = 3 masters + 3 replicas). On master failure, replica auto-promotes

#### Cassandra (URL Store)
- **Why Cassandra**:
  - Single-key lookups (partition key = short_code) → O(1) with Cassandra
  - Massive write throughput (LSM-tree based)
  - Built-in TTL support → expired URLs auto-deleted by compaction
  - Multi-datacenter replication out of the box
  - No complex joins or transactions needed
- **Replication**: RF=3, consistency level QUORUM for writes, ONE for reads (fast reads, durable writes)
- **Compaction**: Leveled Compaction Strategy for read-heavy workload

#### Kafka (Click Event Stream)
- **Why Kafka**: Decouple click ingestion from analytics processing; buffer against traffic spikes
- **Topic**: `click-events`, partitioned by `short_code` (ensures all clicks for a URL go to same partition for ordered processing)
- **Replication**: RF=3, `min.insync.replicas=2`
- **Retention**: 7 days (enough time for Flink to process)

#### Apache Flink (Stream Processing)
- **Why Flink**: Real-time windowed aggregations (clicks per minute, per hour, per day)
- **Processing**: Consumes from Kafka, aggregates by short_code + time window + geo + device, writes to ClickHouse
- **Checkpointing**: Exactly-once semantics via Flink's checkpoint mechanism

#### ClickHouse (Analytics Store)
- **Why ClickHouse**: Columnar database optimized for OLAP queries (sum, count, group by) on billions of rows
- **Queries**: "Top 10 URLs by clicks today", "clicks by country for URL X"

---

## 5. APIs

### Create Short URL
```http
POST /api/v1/urls
Authorization: Bearer <token>
Content-Type: application/json

{
  "long_url": "https://example.com/some/very/long/path?query=param",
  "custom_alias": "my-brand",          // optional
  "expires_at": "2031-03-13T00:00:00Z" // optional
}

Response: 201 Created
{
  "short_url": "https://short.ly/xK9b2",
  "long_url": "https://example.com/some/very/long/path?query=param",
  "short_code": "xK9b2",
  "expires_at": "2031-03-13T00:00:00Z",
  "created_at": "2026-03-13T10:00:00Z"
}
```

### Redirect
```http
GET /{short_code}

Response: 302 Found
Location: https://example.com/some/very/long/path?query=param
```

### Delete URL
```http
DELETE /api/v1/urls/{short_code}
Authorization: Bearer <token>

Response: 204 No Content
```

### Get Analytics
```http
GET /api/v1/urls/{short_code}/analytics?period=7d
Authorization: Bearer <token>

Response: 200 OK
{
  "short_code": "xK9b2",
  "total_clicks": 152437,
  "clicks_by_day": [
    {"date": "2026-03-12", "count": 1234},
    ...
  ],
  "top_countries": [
    {"country": "US", "count": 50234},
    ...
  ],
  "top_referrers": [
    {"referrer": "twitter.com", "count": 30211},
    ...
  ]
}
```

---

## 6. Data Model

### Cassandra — URL Table

```sql
CREATE TABLE url_mappings (
    short_code  TEXT,          -- Partition key
    long_url    TEXT,
    user_id     UUID,
    created_at  TIMESTAMP,
    expires_at  TIMESTAMP,
    is_custom   BOOLEAN,
    PRIMARY KEY (short_code)
) WITH default_time_to_live = 157680000;  -- 5 years in seconds
```

**Why Cassandra over MySQL?**
- No joins needed — pure key-value lookups
- Horizontal scaling by adding nodes (consistent hashing)
- Built-in TTL auto-deletes expired rows
- Multi-DC replication for global availability

### Cassandra — User URLs Table (for user's dashboard)

```sql
CREATE TABLE user_urls (
    user_id     UUID,
    created_at  TIMESTAMP,
    short_code  TEXT,
    long_url    TEXT,
    PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Redis Cache

```
Key:    url:xK9b2
Value:  "https://example.com/some/very/long/path?query=param"
TTL:    86400 (24 hours)
```

### Kafka Topic: `click-events`

```json
{
  "event_id": "uuid-v4",
  "short_code": "xK9b2",
  "timestamp": "2026-03-13T10:30:00Z",
  "ip": "203.0.113.42",
  "user_agent": "Mozilla/5.0...",
  "referrer": "https://twitter.com/post/123",
  "country": "US",
  "city": "San Francisco",
  "device_type": "mobile",
  "os": "iOS"
}
```

### ClickHouse — Analytics Table

```sql
CREATE TABLE url_analytics (
    short_code  String,
    event_date  Date,
    hour        UInt8,
    country     LowCardinality(String),
    referrer    String,
    device_type LowCardinality(String),
    click_count UInt64
) ENGINE = SummingMergeTree()
ORDER BY (short_code, event_date, hour, country);
```

### KGS — Key Store (PostgreSQL)

```sql
CREATE TABLE keys (
    key_value   CHAR(7) PRIMARY KEY,
    status      ENUM('unused', 'assigned', 'used'),
    assigned_to VARCHAR(64),  -- server instance ID
    assigned_at TIMESTAMP
);
```

---

## 7. Fault Tolerance

### General Techniques
| Technique | Application |
|---|---|
| **Replication** | Cassandra RF=3, Kafka RF=3, Redis Cluster 3M+3R |
| **Health Checks** | LB actively health-checks all service instances |
| **Circuit Breaker** | Hystrix/Resilience4j between services (e.g., Write Service → KGS) |
| **Retry with Backoff** | On transient failures (DB timeout, network blip) |
| **Idempotency** | Same long URL + user → returns existing short URL (no duplicates) |
| **Graceful Degradation** | If analytics pipeline is down, redirects still work |

### Problem-Specific Fault Tolerance

1. **KGS Failure / Key Exhaustion**
   - Multiple KGS instances with pre-assigned non-overlapping key ranges
   - Each app server pre-fetches a batch of 1,000 keys → can survive KGS downtime for thousands of requests
   - If a server crashes, its unused keys are lost — acceptable (trillions of total keys)
   - Monitor key pool size; alert when < 10% remaining

2. **Cache Stampede (Thundering Herd)**
   - When a hot URL's cache expires, thousands of requests simultaneously hit DB
   - **Solution**: Use **request coalescing** (singleflight pattern). Only one thread fetches from DB; others wait for the result
   - Also: set cache TTL with jitter (e.g., 24h ± 2h) to avoid synchronized expiration

3. **Hot URL (Celebrity Problem)**
   - A viral URL gets millions of redirects per second
   - **Solution**: Replicate the hot key across multiple Redis shards (read replicas). Use local in-process cache (Caffeine/Guava) with 60s TTL as L1 cache

4. **Custom Alias Race Condition**
   - Two users simultaneously request the same custom alias
   - **Solution**: Cassandra Lightweight Transaction (LWT) — `INSERT IF NOT EXISTS`. Exactly one succeeds; the other gets an error

5. **Datacenter Failure**
   - Cassandra multi-DC replication ensures reads can be served from surviving DC
   - GeoDNS automatically routes traffic away from the failed DC

---

## 8. Additional Considerations

### URL Normalization
Before storing, normalize the long URL to avoid duplicates:
- Lowercase the scheme and host (`HTTP://Example.COM` → `http://example.com`)
- Remove default ports (`:80` for HTTP, `:443` for HTTPS)
- Sort query parameters alphabetically
- Remove trailing slashes
- Decode unnecessary percent-encoding

### Abuse Prevention
- **Rate limiting**: Token bucket per API key (100 URLs/hour for free tier)
- **URL Blacklist**: Integrate with Google Safe Browsing API to reject malicious URLs
- **CAPTCHA**: For anonymous URL creation
- **Spam detection**: Check if long URL domain is in known spam/phishing databases

### Bloom Filter for Duplicate Detection
- Before checking DB for custom alias uniqueness, check a distributed Bloom filter
- False positive rate ~1% means 99% of non-existent aliases are rejected without DB hit
- Bloom filter size: ~1 GB for 1 billion entries with 1% FPR

### Monitoring & Alerting
- **Redirect latency** (p50, p95, p99)
- **Cache hit ratio** (target > 80%)
- **KGS key pool remaining**
- **Kafka consumer lag** (analytics pipeline health)
- **Error rates** (4xx, 5xx by endpoint)

### GDPR / Privacy
- Allow users to delete URLs and all associated analytics data
- Anonymize IP addresses in analytics after 30 days
- Provide data export functionality

---

## 9. Deep Dive: Engineering Trade-offs

### Why KGS Over Hashing? (The Key Decision)

| Approach | How | Pros | Cons |
|---|---|---|---|
| **MD5/SHA256 hash → Base62** | Hash the long URL, take first 7 chars | Simple, deterministic, dedup-friendly | Collisions (birthday paradox: at ~100M URLs, collision probability becomes non-trivial for 7 chars of MD5). Must handle collision retry loop |
| **Counter-based (auto-increment)** | Central DB counter → Base62 encode | Guaranteed unique, simple | Single point of failure; counter is sequential → predictable URLs; single-writer bottleneck |
| **KGS (pre-generated keys)** ⭐ | Offline batch generate keys; app servers fetch batches | Zero collision risk; no runtime computation; non-sequential (unpredictable); no SPOF if multiple KGS | Wasted keys on server crash; requires KGS infrastructure; finite key pool (but 3.5T is effectively infinite) |
| **UUID → Base62** | Generate UUID v4, encode | Simple, decentralized | 128-bit UUID → 22 Base62 chars (too long for a "short" URL) |
| **Snowflake ID → Base62** | Time-ordered ID | Unique, sortable | 64-bit → 11 Base62 chars (still longish); exposes creation time |

**Why KGS wins for URL shortener specifically**: The primary goal is a *short* URL (7 chars). KGS gives you exactly 7 chars with zero collision overhead. The key waste on crash is negligible (losing 1000 out of 3.5T keys is nothing). The offline generation amortizes all uniqueness-checking cost to batch time.

**When would you choose hashing instead?** If you need content-addressable dedup — i.e., same long URL always maps to the same short URL globally. Hash-based approach gives this for free. KGS does not (you'd need a separate dedup lookup table).

### Why Cassandra Over MySQL/PostgreSQL/DynamoDB?

| DB | Strengths for This Use Case | Weaknesses |
|---|---|---|
| **Cassandra** ⭐ | Masterless (no SPOF), linear horizontal scaling, tunable consistency, built-in TTL for auto-expiry, multi-DC replication out of the box | No transactions, no joins (not needed here), operational complexity |
| **DynamoDB** | Managed, auto-scaling, single-digit ms latency, built-in TTL | Vendor lock-in (AWS), cost at very high scale, throughput provisioning complexity |
| **MySQL** | Familiar, ACID, strong consistency | Single-writer primary → write bottleneck at scale, manual sharding required, no built-in TTL |
| **PostgreSQL** | Same as MySQL + better JSON support | Same scaling limitations as MySQL |

**The deciding factors**:
1. **Access pattern is pure key-value lookup** (short_code → long_url). No joins, no range queries, no transactions needed. Cassandra and DynamoDB excel here; MySQL is overkill
2. **Write volume** (~38 writes/sec avg but burst to 1000+): Not extreme, but Cassandra's LSM-tree write path handles bursts gracefully
3. **TTL auto-expiry**: Cassandra's built-in TTL deletes expired rows during compaction — zero application-level cleanup. MySQL would need a cron job
4. **Multi-DC**: Cassandra's multi-datacenter replication is native and well-proven. MySQL multi-DC requires tools like Vitess/ProxySQL and is operationally complex
5. **When to choose DynamoDB**: If you're AWS-native and want zero operational overhead. Cost-effective up to ~1B records; beyond that, Cassandra can be cheaper

### 301 vs 302: The Subtle but Important Choice

```
301 (Permanent Redirect):
  Browser caches the redirect → next visit goes directly to long URL
  ✓ Faster for repeat users (no server hit)
  ✓ Less server load
  ✗ No per-click analytics (browser skips us)
  ✗ Can't change the target URL later (browser cached the old one)
  ✗ CDN/proxy caches it aggressively → hard to invalidate

302 (Temporary Redirect):
  Browser does NOT cache → every visit hits our server
  ✓ Full per-click analytics
  ✓ Can change target URL anytime
  ✓ Full control over every redirect
  ✗ Higher server load
  ✗ Slightly slower for users (extra hop every time)

307 (Temporary Redirect — preserves HTTP method):
  Like 302, but guarantees POST stays POST
  Only matters for non-GET requests (rare for URL shorteners)
```

**Recommendation**: Default to **302** (analytics are valuable). Offer 301 as an opt-in for high-volume, no-analytics use cases. This is what Bit.ly does.

### Read Path Optimization — The Real Bottleneck

At 19K reads/sec peak, the read path is the bottleneck. Here's how each layer helps:

```
Layer 1: CDN (CloudFront/Fastly)
  → For 301 redirects, the CDN caches the Location header
  → Cache key: short URL path
  → Hit ratio: ~30% (many URLs are one-time use)
  → Latency: ~5ms (edge)

Layer 2: Local In-Process Cache (Caffeine/Guava)
  → Each app server caches ~10K hottest URLs
  → Hit ratio: ~20-30%
  → Latency: ~0.01ms (memory access)
  → TTL: 60 seconds (stale reads OK for redirects)

Layer 3: Redis Cluster (Distributed Cache)
  → Caches all recently accessed URLs
  → Hit ratio: ~90% of remaining requests
  → Latency: ~0.5ms (network + memory)
  → TTL: 24h with jitter

Layer 4: Cassandra (Persistent Store)
  → Only ~5% of requests reach here (cold URLs)
  → Latency: ~2-5ms (SSD + network)
  → Consistency: LOCAL_ONE for reads (fastest)
```

**Effective latency**: 0.3 × 5ms + 0.25 × 0.01ms + 0.4 × 0.5ms + 0.05 × 3ms = **~1.9ms average**. Well under the 10ms p99 target.

### Base62 vs Base58 vs Base64

| Encoding | Characters | URL-Safe? | Ambiguity? |
|---|---|---|---|
| **Base62** | a-z, A-Z, 0-9 | ✓ | 0/O and l/1 look similar |
| **Base58** | Base62 minus 0, O, l, I | ✓ | No ambiguous characters (Bitcoin uses this) |
| **Base64** | Base62 + `/` + `+` | ✗ (need URL encoding) | N/A |

**Recommendation**: **Base58** if users might type URLs manually. **Base62** if URLs are only shared as links (more combinations per character).
