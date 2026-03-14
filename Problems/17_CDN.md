# 17. Design a Content Delivery Network (CDN)

---

## 1. Functional Requirements (FR)

- Cache and serve static content (images, videos, CSS, JS, fonts) from edge servers closest to the user
- Route user requests to the optimal edge server (lowest latency)
- If content not cached at edge, fetch from origin server ("cache miss")
- Support cache invalidation / purge on demand
- Support SSL/TLS termination at the edge
- Provide analytics: bandwidth usage, cache hit ratio, latency by region
- Support custom cache rules (TTL, cache key customization, query string handling)

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Serve content in < 50 ms from edge (vs 200+ ms from origin)
- **High Availability**: 99.99% — CDN failure degrades user experience globally
- **Massive Scale**: Serve 100+ Tbps of bandwidth globally
- **Global Coverage**: 200+ Points of Presence (PoPs) worldwide
- **Cache Efficiency**: > 90% cache hit ratio for popular content
- **Fault Tolerant**: Individual PoP failures should be transparent to users
- **DDoS Resilient**: Absorb volumetric attacks at the edge

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total PoPs | 300 |
| Servers per PoP | 50-500 (varies by PoP size) |
| Total edge servers | 50,000 |
| Peak bandwidth | 200 Tbps |
| Requests / sec | 50M (globally) |
| Cache storage per PoP | 100 TB |
| Total cached content | 300 × 100 TB = 30 PB |
| Origin requests (10% miss rate) | 5M/sec |

---

## 4. High-Level Design (HLD)

```
                         ┌──────────────┐
                         │    User      │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │   DNS        │  ← GeoDNS / Anycast routing
                         │   Resolver   │
                         └──────┬───────┘
                                │ (resolves to nearest PoP IP)
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
   ┌──────▼──────┐      ┌──────▼──────┐      ┌──────▼──────┐
   │  PoP: NYC   │      │  PoP: LDN   │      │  PoP: TKY   │
   │             │      │             │      │             │
   │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │
   │ │  Edge   │ │      │ │  Edge   │ │      │ │  Edge   │ │
   │ │ Servers │ │      │ │ Servers │ │      │ │ Servers │ │
   │ │ (Cache) │ │      │ │ (Cache) │ │      │ │ (Cache) │ │
   │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │
   └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │ (cache miss)
                         ┌──────▼───────┐
                         │ Origin Shield│  ← Mid-tier cache (reduces origin load)
                         │  (Regional)  │
                         └──────┬───────┘
                                │ (still a miss)
                         ┌──────▼───────┐
                         │   Origin     │  ← Customer's server or S3
                         │   Server     │
                         └──────────────┘
```

### Component Deep Dive

#### DNS-Based Routing (GeoDNS)
- **How**: User's DNS resolver sends query for `cdn.example.com`
- CDN's authoritative DNS server looks up the resolver's IP location
- Returns the IP address of the closest/fastest PoP
- **Pros**: Simple, widely supported
- **Cons**: DNS TTL caching means slow failover; resolver IP ≠ user IP (shared resolvers)

#### Anycast Routing (Alternative/Complementary)
- **How**: Multiple PoPs announce the **same IP address** via BGP
- Network routing (BGP) naturally sends packets to the closest PoP
- **Pros**: Instant failover (BGP reconverges in seconds), immune to DNS TTL issues
- **Cons**: No per-user control, relies on ISP routing tables
- **Real-world**: Cloudflare uses Anycast; AWS CloudFront uses GeoDNS

#### Edge Server (Cache Node)
- **Reverse proxy** (NGINX / custom): Handles TLS, request routing, caching
- **Cache storage**: 
  - **Memory** (RAM): Hot content, 64 GB per server
  - **SSD**: Warm content, 2-10 TB per server
  - **HDD**: Cold content (for very large PoPs), 50+ TB
- **Cache lookup**: Hash(cache_key) → check memory → check SSD → check HDD → cache miss
- **LRU eviction**: Least Recently Used items evicted when cache is full
- **Consistent hashing**: Within a PoP, requests are routed to specific servers based on content hash → avoids duplicate caching of the same content across servers

#### Origin Shield (Mid-Tier Cache)
- **Why**: Without it, 300 PoPs each cache-miss independently → origin gets hammered with 300 requests for the same content
- **How**: Intermediate cache between PoPs and origin. All PoPs in a region route cache misses through the shield
- **Benefit**: Origin sees 3-5 cache-miss requests instead of 300
- **Placement**: 3-5 regional shields (US-East, US-West, Europe, Asia)

#### Cache Key
```
Default cache key:  scheme + host + path + query_string
  https://cdn.example.com/images/logo.png?v=2

Customizable:
  - Ignore query string (for static assets)
  - Include cookies (for personalized content — careful, reduces hit ratio)
  - Include headers (Accept-Encoding, Accept-Language)
  - Include device type (mobile vs desktop)
```

#### Cache Control Headers
```http
Cache-Control: public, max-age=86400, s-maxage=604800
  public:    CDN can cache
  max-age:   Browser cache TTL (1 day)
  s-maxage:  CDN/shared cache TTL (7 days)

Cache-Control: private, no-store
  Don't cache at CDN (personalized content)

Vary: Accept-Encoding
  Cache different versions for gzip vs brotli
```

---

## 5. APIs

### Cache Purge
```http
POST /api/v1/purge
{
  "urls": ["https://cdn.example.com/images/logo.png"],
  "pattern": "https://cdn.example.com/css/*"
}
Response: 202 Accepted
{
  "purge_id": "purge-uuid",
  "status": "propagating",
  "estimated_completion": "30 seconds"
}
```

### Cache Warm (Pre-populate)
```http
POST /api/v1/warm
{
  "urls": ["https://cdn.example.com/videos/new-release.mp4"],
  "regions": ["us-east", "eu-west", "ap-south"]
}
```

### Get Analytics
```http
GET /api/v1/analytics?domain=cdn.example.com&period=24h
Response: 200 OK
{
  "total_requests": 50000000,
  "cache_hit_ratio": 0.93,
  "bandwidth_gb": 15000,
  "latency_p50_ms": 12,
  "latency_p99_ms": 45,
  "by_region": [...]
}
```

---

## 6. Data Model

### Edge Server Cache Entry

```
Cache Key:  "https://cdn.example.com/images/logo.png"
Metadata:
  content_type:   "image/png"
  content_length: 45678
  etag:           "abc123"
  last_modified:  "2026-03-13T00:00:00Z"
  cache_control:  "public, max-age=86400"
  expires_at:     "2026-03-14T00:00:00Z"
  created_at:     "2026-03-13T00:00:00Z"
  hit_count:      1523
  last_accessed:  "2026-03-13T10:30:00Z"
Body:
  [binary content stored in SSD/memory]
```

### DNS Routing Table

```
Region    PoP           IP Addresses         Health
US-East   NYC           [203.0.113.1, ...]   healthy
US-East   IAD           [203.0.113.5, ...]   healthy
EU-West   LDN           [198.51.100.1, ...]  healthy
AP-South  MUM           [192.0.2.1, ...]     degraded
```

### Purge Propagation

```
Kafka Topic: cache-purge
{
  "purge_id": "uuid",
  "pattern": "https://cdn.example.com/css/*",
  "initiated_at": "2026-03-13T10:00:00Z",
  "target_pops": ["all"]
}
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **PoP failure** | DNS/Anycast routes to next closest PoP. Health checks every 10s |
| **Edge server failure** | Load balancer within PoP routes to healthy servers; consistent hashing rebalances |
| **Origin failure** | Serve stale content from cache (`stale-while-revalidate`, `stale-if-error` directives) |
| **Cache stampede** | Request coalescing: only one request to origin; all other waiters served from the same response |
| **DDoS at edge** | Rate limiting, WAF rules, TCP SYN cookies, challenge pages (CAPTCHA) at edge |
| **Cable cut (region offline)** | Anycast reroutes globally; regional failover to adjacent PoPs |

### Specific: Cache Stampede / Thundering Herd
When a popular cached item expires, hundreds of concurrent requests trigger simultaneous origin fetches:
1. **Request coalescing**: First request triggers origin fetch; subsequent requests wait for the result
2. **Stale-while-revalidate**: Serve stale content while fetching fresh content in background
3. **Jittered TTL**: Add random ±10% jitter to TTL → different PoPs expire at different times

---

## 8. Additional Considerations

### Push vs Pull CDN

| Type | How | Best For |
|---|---|---|
| **Pull CDN** | Edge fetches from origin on first request (cache miss) | Dynamic sites, user-generated content |
| **Push CDN** | Origin proactively pushes content to edge servers | Known popular content (video, software updates) |

### Multi-CDN Strategy
- Use multiple CDN providers (Akamai + CloudFront + Fastly)
- **Benefits**: Redundancy, cost optimization, best performance per region
- **Real-time switching**: Traffic management layer routes to the best-performing CDN per user based on latency measurements

### Edge Computing
- Run application logic at the edge (Cloudflare Workers, AWS Lambda@Edge)
- Use cases: A/B testing, header manipulation, authentication, image resizing, personalization
- Reduces round trips to origin

### TLS at Edge
- SSL/TLS terminated at edge server (not origin)
- Certificate management: Automatic certificate provisioning (Let's Encrypt) or customer's certificate
- Edge-to-origin connection: Can be HTTP (for internal network) or HTTPS (for security)

### Image Optimization at Edge
- Automatic format conversion (WebP/AVIF for supported browsers)
- Responsive sizing (resize based on `Accept` header or query param)
- Quality adjustment based on network speed
- Reduces bandwidth by 30-50%

### Monitoring
- Real-time dashboards: requests/sec, bandwidth, cache hit ratio, error rate by PoP
- Alert on: cache hit ratio < 80%, origin error rate > 5%, latency p99 > 100ms
- Origin health monitoring: synthetic probes from each region

---

## 9. Deep Dive: Engineering Trade-offs

### Detailed Cache Miss Flow (Edge → Shield → Origin)

```
User requests: GET https://cdn.example.com/images/hero.jpg

Step 1: DNS Resolution (0 ms — cached from previous visit)
  cdn.example.com → 203.0.113.42 (NYC PoP IP, via GeoDNS/Anycast)

Step 2: TLS Handshake at Edge (0 ms — session resumed)
  Returning visitor: 0-RTT TLS 1.3 resumption (PSK from prior visit)
  New visitor: 1-RTT full TLS 1.3 handshake (~10ms within PoP)

Step 3: Edge Server Cache Lookup (NYC PoP)
  Cache key: "https://cdn.example.com/images/hero.jpg"
  
  3a. L1: Memory (RAM) lookup       → MISS (0.1 ms)
      64 GB RAM, LRU eviction, ~500K hot objects
  3b. L2: SSD lookup                 → MISS (1 ms)
      4 TB SSD, ~20M warm objects
  3c. L3: HDD lookup (large PoPs)   → MISS (5 ms)
      50 TB HDD, ~200M cold objects
  
  Total edge lookup: ~6 ms → CACHE MISS

Step 4: Consistent Hash Check (0.1 ms)
  hash("cdn.example.com/images/hero.jpg") % ring → Server 7
  Am I Server 7? YES → I'm the correct server for this content
  (If NO → redirect to Server 7 within PoP via internal LB — avoids
   N copies of same content across N servers in PoP)

Step 5: Origin Shield Fetch (15 ms RTT to US-East shield)
  Edge → Origin Shield (regional mid-tier cache)
  
  5a. Shield memory check        → MISS
  5b. Shield SSD check           → MISS
  
  Shield MISS: only 1 request goes to origin
  (If 50 PoPs miss simultaneously, shield collapses 50 → 1 origin request)

Step 6: Origin Fetch (80 ms RTT to origin in US-West)
  Shield → Origin Server (or S3 bucket)
  Origin generates response: 200 OK, image/jpeg, 150 KB
  Headers: Cache-Control: public, max-age=86400, s-maxage=604800

Step 7: Backfill — Response flows back through the chain
  Origin → Shield: caches on SSD (TTL 7 days from s-maxage)     (80 ms)
  Shield → Edge:   caches on SSD + RAM (TTL 7 days)              (15 ms)
  Edge → User:     serves response                                (5 ms)

Total cold-miss latency: ~6 + 15 + 80 + 5 = ~106 ms
Subsequent requests (cache hit): ~5 ms (RAM) or ~6 ms (SSD)

Performance impact: 20× latency difference between hit and miss
  This is why cache hit ratio (target >90%) is the most critical CDN metric
```

### TLS 1.3 Handshake at Edge (Why Edge Termination Matters)

```
Without CDN (TLS to origin, 200 ms away):
  Client                                 Origin (200ms RTT)
    │── ClientHello ──────────────────────→│  (200ms)
    │←── ServerHello + Certificate ────────│  (200ms)
    │── Finished ──────────────────────────→│  (200ms)
    │←── Application Data ─────────────────│  (200ms)
    Total: 2 RTT × 200ms = 400ms before first byte

With CDN (TLS at edge, 5 ms away):
  Client           Edge (5ms RTT)        Origin (if miss)
    │── ClientHello ──→│                    (5ms)
    │←── ServerHello ──│                    (5ms)
    │── Finished ──────→│                    (5ms)
    │←── Data (if hit) ─│                    (5ms)
    Total: 2 RTT × 5ms = 10ms (cache hit)
    
    TLS 1.3 with 0-RTT resumption (returning visitor):
    │── ClientHello + PSK + HTTP Request ──→│  (5ms)
    │←── ServerHello + Application Data ────│  (5ms)
    Total: 1 RTT × 5ms = 5ms!

Saving: 400ms → 10ms (or 5ms with 0-RTT) = 40-80× faster TLS
  OCSP stapling: edge server pre-fetches certificate revocation status
    → avoids client's separate OCSP check (saves another RTT)
```

### Consistent Hashing Within a PoP

```
Problem: NYC PoP has 100 edge servers. Without coordination,
  the same URL could be cached on ALL 100 servers = 100× wasted storage
  
Solution: Consistent hashing maps URLs to specific servers

  Hash Ring (virtual nodes):
  ┌───────────────────────────────────────────────────────────┐
  │  0°                                                  360° │
  │  │    S1   S3    S2   S1   S3   S2    S1   S3   S2   │   │
  │  │    ↓    ↓     ↓    ↓    ↓    ↓     ↓    ↓    ↓    │   │
  │  ├────●────●─────●────●────●────●─────●────●────●────┤   │
  │       ↑                                                    │
  │   hash("/images/hero.jpg") lands here → routed to S3      │
  └───────────────────────────────────────────────────────────┘

  Each physical server has 50+ virtual nodes on the ring
  hash(URL) → position on ring → clockwise to nearest server
  
  Result: Each URL is cached on exactly ONE server within the PoP
    100 servers, each caches 1% of URLs → 100× storage efficiency
    Cache hit ratio within PoP: if S3 has it, guaranteed hit

  Server failure (S3 dies):
    L7 LB detects via health check (every 5s)
    S3's virtual nodes removed from ring
    S3's URLs now map to next server clockwise (S2 or S1)
    Only ~1/N keys need to re-cache (not all keys)
    Temporary cache misses for S3's keys → backfill from shield/origin

  L7 Load Balancer implementation:
    Nginx/Envoy computes hash of request URL
    Routes to correct backend server based on ring position
    Configuration updates pushed via control plane on server add/remove
```

### Cache Invalidation Propagation Flow

```
Admin triggers: "Purge /css/main.css across all PoPs"

  T+0ms:    Admin API call → Purge Service
  T+5ms:    Purge Service validates credentials + rate limits
  T+10ms:   Write to Kafka topic: cache-purge
              { "purge_id": "uuid", 
                "pattern": "https://cdn.example.com/css/main.css",
                "type": "exact",    // or "glob" for wildcard
                "target": "all_pops" }
  
  T+50ms:   Each PoP's purge consumer reads from Kafka
            (300 PoPs, each with dedicated consumer)
  
  T+100ms:  Within each PoP, purge consumer fans out to all edge servers
            HTTP POST /internal/purge to each server
            
  T+200ms:  Each edge server:
            - Exact match: hash(key) → delete from memory + SSD index
            - Glob pattern: scan cache index for matches → delete all
            - ACK back to purge consumer
  
  T+500ms:  All PoPs report completion → API returns 202 Accepted
  
  Total propagation: < 1 second for exact URLs
                     < 5 seconds for glob patterns (index scan slower)
  
  Consistency window during purge:
    Between T+0 and T+500ms, some PoPs serve stale content
    In-flight requests started before purge → may receive stale response
    Mitigation: add Cache-Control: no-cache for critical updates
    OR: version URLs (/css/main.v42.css) → no purge needed (immutable URLs)

  Glob purge performance:
    Pattern "https://cdn.example.com/css/*" → must scan cache index
    100M cached objects per PoP → full scan takes 2-5 seconds
    Optimization: prefix tree index on cache keys → O(prefix_len) lookup
    Alternative: tag-based purge — tag objects at cache time, purge by tag
      tag "css-assets" → all CSS files → purge tag = O(1) lookup
```

### Cache Warming vs On-Demand: When to Pre-Populate

```
On-Demand (Pull, default):
  First user request triggers cache miss → origin fetch → cache fill
  ✓ Zero wasted cache space (only cache what's requested)
  ✗ First user in each PoP sees slow response (cache miss penalty)
  ✗ Flash crowds: viral content → all 300 PoPs miss simultaneously
      → origin gets 300 concurrent requests for same object

Pre-Warming (Push):
  Proactively push content to edge servers BEFORE users request it
  ✓ Zero cold-start latency for users
  ✓ Origin sees 1 request (from push system), not 300
  ✗ Wastes cache space if content isn't actually popular
  ✗ Requires knowing WHAT to warm in advance
  
  Best for:
    - Software update releases (known time, massive download)
    - Major product launches (viral landing page)
    - Live event thumbnails (Super Bowl halftime ads)
    - Video platform: push first 10 seconds of trending videos

Hybrid (recommended):
  Default: on-demand pull for all content
  Selective warm: push known-popular content 1 hour before expected traffic
  Request coalescing at origin shield: collapse concurrent misses into 1 fetch
```

