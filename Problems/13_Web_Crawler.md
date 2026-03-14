# 13. Design a Web Crawler (Googlebot)

---

## 1. Functional Requirements (FR)

- Crawl the web starting from a set of seed URLs
- Discover new URLs by extracting links from crawled pages
- Download and store web page content for indexing
- Respect `robots.txt` directives (politeness)
- Handle URL deduplication (don't crawl the same page twice)
- Support recrawling to detect updated content
- Prioritize important/popular pages for crawling first
- Handle different content types (HTML, PDF, images, etc.)

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: Crawl 1 billion pages per day
- **Politeness**: Don't overload any single web server (rate limit per domain)
- **Robustness**: Handle spider traps, infinite loops, malformed HTML, server errors
- **Scalability**: Horizontally scalable — add machines to crawl faster
- **Freshness**: Re-crawl pages based on change frequency
- **Extensible**: Easy to add new modules (content extraction, language detection, etc.)
- **Fault Tolerant**: A single node failure should not lose crawl progress

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Pages to crawl | 15B (entire web, roughly) |
| Target crawl rate | 1B pages/day |
| Pages / sec | ~11,500 |
| Avg page size | 100 KB |
| Download / day | 1B × 100 KB = 100 TB |
| Download bandwidth | ~10 Gbps |
| URL frontier size | 10B URLs × 100 bytes = 1 TB |
| Content storage / month | 3 PB |
| Crawl workers | ~1000 machines (each handles ~12 pages/sec) |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│  Seed URLs   │
└──────┬───────┘
       │
┌──────▼───────────────────────────────────────────┐
│                 URL Frontier                      │
│  (Priority Queue + Politeness Queue per domain)   │
└──────┬───────────────────────────────────────────┘
       │
┌──────▼───────┐
│  DNS Resolver│  ← Local DNS cache
└──────┬───────┘
       │
┌──────▼───────┐
│  Fetcher     │  ← HTTP downloader (with timeout, redirect handling)
│  (Workers)   │
└──────┬───────┘
       │
┌──────▼───────┐
│  Content     │  ← Duplicate? Already seen? (content fingerprint check)
│  Dedup       │
└──────┬───────┘
       │
┌──────▼───────┐
│  Content     │  ← Store raw HTML in distributed file system
│  Store       │
│  (S3 / GFS)  │
└──────┬───────┘
       │
┌──────▼───────┐
│  Parser /    │  ← Extract text, links, metadata
│  Extractor   │
└──────┬───────┘
       │
       ├─────────────────────┐
       │                     │
┌──────▼───────┐     ┌──────▼───────┐
│  URL Filter  │     │  Index       │
│  + Dedup     │     │  Pipeline    │  ← Send to search engine indexer
│  (Bloom      │     └──────────────┘
│   Filter)    │
└──────┬───────┘
       │
       │ (new URLs discovered)
       │
┌──────▼───────┐
│  URL Frontier│  ← Back to the beginning (cycle)
└──────────────┘
```

### Component Deep Dive

#### URL Frontier — The Most Complex Component

The URL frontier is NOT a simple queue. It has two sub-systems:

**1. Priority Queue (Front Queue)**:
- Determines WHICH URLs to crawl first
- Priority based on:
  - PageRank of the page (higher = more important)
  - Change frequency (pages that change often should be recrawled sooner)
  - Domain authority
  - Freshness requirement (news sites = high priority)
- Implementation: Multiple priority queues (P0, P1, P2, ..., PN)
  - `Priority = f(PageRank, freshness_score, domain_authority)`
  - A prioritizer assigns each URL to the appropriate queue

**2. Politeness Queue (Back Queue)**:
- Ensures we don't overwhelm any single domain
- One FIFO queue per domain (e.g., queue for `example.com`, queue for `wikipedia.org`)
- A rate limiter enforces: max 1 request per domain per second (or per `Crawl-delay` in robots.txt)
- Implementation:
  - Hash table: `domain → FIFO queue`
  - Worker threads pull from different domain queues (round-robin)
  - After fetching from domain X, put domain X's queue on cooldown for N seconds

```
Front (Priority) Queues:        Back (Politeness) Queues:
┌─────────┐                     ┌──────────────────────┐
│ P0 (VIP)│ ────────────────▶   │ queue: example.com   │
│ P1 (High│     Routing by      │ queue: wikipedia.org │
│ P2 (Med)│     domain          │ queue: reddit.com    │
│ P3 (Low)│                     │ ...                  │
└─────────┘                     └──────────────────────┘
```

#### Fetcher (HTTP Downloader)
- **Connection pooling**: Reuse HTTP connections to the same host
- **Timeouts**: Connect timeout = 5s, read timeout = 30s
- **Redirect handling**: Follow up to 5 redirects (301, 302)
- **robots.txt compliance**:
  1. Before crawling any page on a domain, fetch and cache `robots.txt`
  2. Parse directives: `Disallow`, `Allow`, `Crawl-delay`, `Sitemap`
  3. Cache robots.txt per domain with TTL = 24 hours
- **User-Agent**: Identify as `Googlebot/2.1` (or custom bot name)
- **Content types**: Accept HTML, PDF, DOC (reject binary, video, etc.)

#### Content Deduplication
- **Problem**: Many pages have identical or near-identical content (mirrors, syndication, plagiarism)
- **Exact dedup**: MD5/SHA-256 hash of page content → check against seen hashes
- **Near-dedup**: **SimHash** or **MinHash** algorithm
  
**SimHash Algorithm**:
1. Extract features (word n-grams) from page
2. Hash each feature to a 64-bit value
3. For each bit position: if feature hash bit = 1, add weight; if 0, subtract weight
4. Final SimHash = sign of each bit position (positive → 1, negative → 0)
5. Two pages are near-duplicates if Hamming distance of their SimHashes ≤ 3

- **Storage**: Bloom filter for exact hashes (set membership), SimHash table for near-dupes

#### URL Deduplication (Seen URLs)
- **Problem**: The same URL can be discovered from multiple pages
- **Solution**: Bloom filter with 100 billion entries
  - Size: ~10 GB for 1% false positive rate (10 bits per element)
  - Before adding a URL to the frontier, check the Bloom filter
  - False positives mean we skip some URLs (acceptable — better than re-crawling)
- **URL normalization** before dedup:
  - Lowercase scheme and host
  - Remove fragment (`#section`)
  - Remove default ports
  - Sort query parameters
  - Resolve relative paths (`/a/b/../c` → `/a/c`)
  - Remove tracking parameters (`utm_source`, `fbclid`)

#### DNS Resolver
- DNS lookups are slow (50-200ms) and become a bottleneck at scale
- **Solution**: Local DNS cache + custom DNS resolver
- Pre-resolve DNS for URLs in the frontier (batch DNS lookups)
- Cache DNS results with TTL

#### Parser / Link Extractor
- Parse HTML using robust parser (handles malformed HTML)
- Extract:
  - Links (`<a href>`, `<link>`, `<script src>`)
  - Text content (for indexing)
  - Metadata (`<title>`, `<meta>`, Open Graph tags)
  - `<base>` tag for resolving relative URLs
- Handle JavaScript-rendered pages:
  - Option 1: Headless browser rendering (Puppeteer/Playwright) — expensive
  - Option 2: Only crawl server-rendered HTML — cheaper, misses SPA content

---

## 5. APIs

A web crawler is primarily a batch system, but has operational APIs:

### Add Seed URLs
```http
POST /api/v1/crawler/seeds
{
  "urls": ["https://example.com", "https://news.ycombinator.com"],
  "priority": "high"
}
```

### Get Crawl Status
```http
GET /api/v1/crawler/status
Response: 200 OK
{
  "pages_crawled_today": 892345678,
  "pages_in_frontier": 5234567890,
  "crawl_rate_per_sec": 11500,
  "active_workers": 980,
  "errors_today": 12345
}
```

### Block Domain
```http
POST /api/v1/crawler/block
{
  "domain": "spam-site.com",
  "reason": "spam"
}
```

---

## 6. Data Model

### URL Frontier Entry

```
{
  url:              "https://example.com/page/123",
  normalized_url:   "https://example.com/page/123",
  domain:           "example.com",
  priority:         2,
  discovered_at:    "2026-03-13T10:00:00Z",
  last_crawled_at:  "2026-03-12T08:00:00Z",
  recrawl_interval: 86400,         // seconds
  retries:          0,
  depth:            3,              // hops from seed URL
  referrer_url:     "https://example.com/"
}
```

### Content Store (S3 / GFS)

```
Bucket: crawler-content
Key:    {content_hash}
Value:  {
          url: "https://example.com/page/123",
          content_type: "text/html",
          status_code: 200,
          headers: {...},
          body: "<html>...",
          crawled_at: "2026-03-13T10:00:00Z",
          content_hash: "sha256:...",
          simhash: "0xA3F2B1C4D5E6F7A8"
        }
```

### Robots.txt Cache (Redis)

```
Key:    robots:{domain}
Value:  {
          rules: [
            {user_agent: "*", disallow: ["/admin", "/private"]},
            {user_agent: "Googlebot", allow: ["/api"]}
          ],
          crawl_delay: 2,
          sitemaps: ["https://example.com/sitemap.xml"],
          fetched_at: "2026-03-13T00:00:00Z"
        }
TTL:    86400 (24 hours)
```

### Bloom Filter — Seen URLs

```
Type:       Distributed Bloom Filter (Redis-backed or custom)
Capacity:   100 billion URLs
FPR:        1%
Size:       ~120 GB (10 bits per element × 100B)
Hash functions: 7
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Worker crash** | URL remains in frontier (not removed until crawl confirmed). Reassigned to another worker |
| **DNS failure** | Retry with exponential backoff; fallback to secondary DNS |
| **HTTP timeout** | Retry up to 3 times with backoff; then mark URL as failed and deprioritize |
| **Content store failure** | S3 with 11 nines durability + cross-region replication |
| **Frontier data loss** | Checkpoint frontier to disk periodically; rebuild from content store's crawled URLs |
| **Bloom filter loss** | Rebuild from frontier + content store URLs (takes hours but possible) |

### Spider Traps and Infinite Loops
- **Trap types**: Calendar URLs (`/calendar?date=2026-03-13`, `/calendar?date=2026-03-14`, ...), URL with session IDs, paginated URLs with no end
- **Defenses**:
  - Max URL depth: Stop crawling after depth 15 from seed
  - Max URLs per domain: Cap at 1M per domain per crawl cycle
  - URL pattern detection: If > 1000 URLs match same URL pattern, stop following
  - Maximum page size: Skip pages > 10 MB
  - Content hash dedup: If same content at different URLs → stop following that path

---

## 8. Additional Considerations

### Recrawl Strategy
Not all pages need the same recrawl frequency:
- **News sites**: Every 15 minutes
- **E-commerce (prices)**: Every few hours
- **Wikipedia/docs**: Every few days
- **Static pages**: Weekly or monthly

**Adaptive recrawl**: Track how often a page changes → adjust recrawl interval dynamically
```
if page_changed:
    recrawl_interval = max(recrawl_interval / 2, MIN_INTERVAL)
else:
    recrawl_interval = min(recrawl_interval * 2, MAX_INTERVAL)
```

### Distributed Architecture
- **Master-Worker**: Master assigns URL batches to workers; workers fetch and report results
- **Alternative (Masterless)**: Each worker manages its own set of domains (consistent hashing on domain name)
  - Advantage: No single point of failure
  - Workers coordinate via message queue (Kafka)

### Legal and Ethical Considerations
- **robots.txt**: MUST be respected
- **Crawl-delay**: Honor the specified delay between requests
- **noindex meta tag**: Don't index pages with `<meta name="robots" content="noindex">`
- **Terms of service**: Some sites prohibit crawling beyond robots.txt
- **Personal data**: Don't store PII discovered during crawling (GDPR)

### Sitemap Processing
- Parse `sitemap.xml` for a curated list of a site's pages
- Priority and changefreq hints from sitemap help with scheduling
- Sitemaps can reference other sitemaps (sitemap index files)

---

## 9. Deep Dive: Engineering Trade-offs

### URL Frontier: Priority + Politeness Architecture

```
The URL Frontier is NOT a simple queue. It's a two-tier system:

┌─────────────────────────────────────────────────────────┐
│                    FRONT QUEUES (Priority)                │
│                                                           │
│  Queue P1 (critical):  [google.com/news, bbc.com/latest] │
│  Queue P2 (high):      [wikipedia.org/..., nytimes.com/] │
│  Queue P3 (normal):    [example.com/about, blog.io/post] │
│  Queue P4 (low):       [random-site.xyz/page42]          │
│                                                           │
│  Priority assignment:                                     │
│    PageRank of domain → high PR = high priority           │
│    Freshness need → news sites = high priority            │
│    Sitemap priority hint → 0.8 = high, 0.2 = low         │
│    Previous change rate → frequently changing = higher    │
│                                                           │
│  Selection: weighted random from front queues             │
│    P1: 40% chance, P2: 30%, P3: 20%, P4: 10%            │
└───────────────────┬─────────────────────────────────────┘
                    │ URL selected by priority
┌───────────────────▼─────────────────────────────────────┐
│                    BACK QUEUES (Politeness)               │
│                                                           │
│  One queue PER DOMAIN (ensures per-domain rate limiting)  │
│                                                           │
│  Queue [google.com]:     [url1, url2, url3]              │
│    last_fetch: 10:00:05, crawl_delay: 2s                 │
│    → next fetch allowed: 10:00:07                        │
│                                                           │
│  Queue [wikipedia.org]:  [url4, url5]                    │
│    last_fetch: 10:00:03, crawl_delay: 5s                 │
│    → next fetch allowed: 10:00:08                        │
│                                                           │
│  Queue [tiny-blog.com]:  [url6]                          │
│    last_fetch: 10:00:01, crawl_delay: 10s                │
│    → next fetch allowed: 10:00:11                        │
│                                                           │
│  Worker selects: domain whose next_fetch_time <= now()    │
│  → Dequeue one URL from that domain's back queue          │
│  → Fetch → update last_fetch timestamp                    │
└─────────────────────────────────────────────────────────┘

Why two tiers?
  Front queues: ensure important pages are crawled first (priority)
  Back queues: ensure we don't hammer any single domain (politeness)
  
  Without back queues: high-priority domain gets 1000 requests/second
  → Gets banned, IP blacklisted, violates robots.txt crawl-delay
  
  Without front queues: crawl random pages regardless of importance
  → Waste bandwidth on low-value pages while important pages wait
```

### Crawl Execution Flow — One URL End-to-End

```
URL: "https://example.com/products/shoes"

Step 1: Dequeue from frontier (0.1 ms)
  Back queue [example.com] is eligible (last_fetch was 3s ago, delay = 2s)
  Pop URL from queue

Step 2: DNS resolution (1 ms — cached)
  Check local DNS cache → hit: example.com → 93.184.216.34
  Cache miss: query DNS resolver → cache for TTL (300s typical)
  Optimization: batch DNS prefetch for all URLs in a domain's back queue

Step 3: Robots.txt check (0.1 ms — cached)
  Fetch from Redis: robots:{example.com}
  Check if /products/shoes is allowed for our User-Agent
  If disallowed → skip URL, log reason, dequeue next

Step 4: HTTP fetch (200 ms — network)
  Open connection (reuse connection pool for same domain)
  Send: GET /products/shoes HTTP/1.1
        Host: example.com
        User-Agent: MyCrawler/1.0 (+https://mycrawler.com/about)
  Handle redirects: follow up to 5 redirects (301, 302, 307)
  Timeout: 30 seconds (some servers are slow)
  Max response size: 10 MB (drop larger responses)

Step 5: Content deduplication (1 ms)
  Compute SimHash of page content
  Check SimHash against seen-pages Bloom filter
  If near-duplicate (Hamming distance < 3) → skip (already crawled similar page)
  If new → add to Bloom filter, continue

Step 6: Parse and extract (5 ms)
  Parse HTML → extract:
    - Title, meta description, canonical URL
    - All <a href="..."> links → new URLs for frontier
    - Structured data (JSON-LD, microdata)
    - Language detection (for language-specific index)

Step 7: Store content (10 ms)
  Write to S3: { url, content, headers, crawled_at, content_hash }
  Key: content_hash (deduplication at storage level)

Step 8: Process extracted URLs (1 ms)
  For each extracted URL:
    Normalize: lowercase, remove fragment, resolve relative
    Check Bloom filter: already seen?
    If new → add to frontier with priority score
    
  Update domain crawl stats:
    INCR crawled:{domain} — track total pages per domain
    If crawled > 1M → stop crawling this domain (trap protection)

Step 9: Update back queue timestamp
  Set last_fetch_time = now() for domain example.com
  Next URL from this domain eligible after crawl_delay

Total time per URL: ~220 ms (dominated by network fetch)
Throughput per worker: ~4.5 URLs/sec
1000 workers × 4.5 = ~4,500 URLs/sec ≈ ~390M pages/day
Target: 1B pages/day → need ~2,500 workers
```

### Distributed Coordination: Domain-Sharded Architecture

```
Problem: 1000 crawler workers fetching URLs from the same frontier
  → Multiple workers might fetch from the same domain simultaneously
  → Violates politeness (combined rate exceeds crawl-delay)

Solution: Shard domains across workers using consistent hashing

  domain_shard = hash(domain) % num_workers
  
  Worker 0: responsible for domains hashing to 0
    [google.com, stackoverflow.com, ...]
  Worker 1: responsible for domains hashing to 1
    [wikipedia.org, amazon.com, ...]
  ...
  Worker 999: responsible for domains hashing to 999
  
  Each worker maintains its OWN back queues for its assigned domains
  → No coordination needed for politeness (single writer per domain)
  → No distributed locks for rate limiting

Feeding URLs to workers:
  Kafka topic: discovered-urls (partitioned by domain hash)
  When any worker discovers new URLs:
    Publish to Kafka with key = domain
    Kafka routes to correct partition → correct worker consumes
  
  Worker consumes URLs for its assigned domains
  → Checks Bloom filter (shared Redis, or local + periodic sync)
  → Adds to local frontier if new

Rebalancing on worker failure:
  Worker 42 dies → its domains temporarily uncrawled
  Kafka consumer group rebalances → another worker inherits partition 42
  New worker loads domain state (last_fetch times) from Redis
  Resumes crawling within ~30 seconds

Advantages over centralized frontier:
  ✓ No single point of failure (no master)
  ✓ Linear scalability (add workers → more domains handled)
  ✓ Politeness guaranteed by design (single writer per domain)
  ✗ Hot domains (google.com) assigned to one worker → uneven load
  Mitigation: split hot domains into sub-crawlers (by URL path prefix)
```

### BFS vs DFS vs Priority-Based Crawling

```
BFS (Breadth-First):
  Crawl all pages at depth 1, then depth 2, then depth 3...
  ✓ Discovers important pages early (homepage → main sections)
  ✗ May waste bandwidth on low-value pages at same depth
  ✗ Doesn't account for page importance
  
DFS (Depth-First):
  Follow links deep into a site before backtracking
  ✓ Less memory (only current path in stack)
  ✗ Gets trapped in deep page hierarchies
  ✗ May never reach important pages on other domains
  
Priority-Based (this design) ⭐:
  Score each URL by importance, crawl highest-scored first
  Score = α × PageRank(domain) + β × depth_penalty + γ × freshness_need
  
  ✓ Most important pages crawled first
  ✓ Adaptive: can reprioritize based on discovery
  ✗ More complex (priority queue management)
  
  Practical: hybrid approach
    Start with BFS from seeds (discover site structure)
    Switch to priority-based after initial crawl (focus on important pages)
    Use DFS within a domain (follow sitemap structure)
```

### URL Normalization: Why It Matters

```
The same page accessible via many URLs:
  https://example.com/page
  https://Example.COM/page          → lowercase host
  https://example.com/page/         → trailing slash
  https://example.com/page?a=1&b=2  → query param order
  https://example.com/page?b=2&a=1  → same params, different order
  http://example.com/page           → different scheme
  https://example.com/./page/../page → path normalization

Without normalization: crawl same page 7 times → wasted bandwidth
With normalization: all resolve to https://example.com/page

Normalization rules (applied before Bloom filter check):
  1. Lowercase scheme and host
  2. Remove default port (:80 for HTTP, :443 for HTTPS)
  3. Remove trailing slash (unless root /)
  4. Sort query parameters alphabetically
  5. Remove tracking parameters (utm_source, fbclid, gclid)
  6. Resolve path (remove /. and /..)
  7. Decode unreserved percent-encoded characters (%41 → A)
  8. Prefer HTTPS canonical URL (follow <link rel="canonical">)
```

