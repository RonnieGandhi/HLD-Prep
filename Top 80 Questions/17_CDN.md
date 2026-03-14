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

