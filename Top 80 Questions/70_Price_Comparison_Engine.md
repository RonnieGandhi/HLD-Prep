# 70. Design a Price Comparison Engine

---

## 1. Functional Requirements (FR)

- **Aggregate prices**: Collect prices for the same product from multiple retailers/sellers
- **Product matching**: Identify the same product across different sites (different names/URLs)
- **Price tracking**: Track price history over time; show price trends and charts
- **Price alerts**: Notify users when price drops below their target
- **Search & browse**: Search products; filter by category, brand, price range
- **Best deal identification**: Show cheapest option with total cost (price + shipping + tax)
- **Coupon integration**: Show applicable coupons/deals alongside prices
- **Retailer ratings**: Show retailer reliability and shipping speed

---

## 2. Non-Functional Requirements (NFRs)

- **Freshness**: Prices updated at least every 6 hours; popular products every 1 hour
- **Scale**: 100M+ products, 50+ retailers, 1B+ price data points
- **Search Latency**: < 200 ms for product search with price comparison
- **Accuracy**: Prices must reflect actual retailer prices (stale prices erode trust)
- **Availability**: 99.9%

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Products tracked | 100M |
| Retailers | 50+ |
| Price data points / day | 500M (100M products x ~5 price updates avg) |
| Search queries / sec | 10K |
| Price alert checks / day | 50M |
| Price history storage | ~2 TB/year |

---

## 4. High-Level Design (HLD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DATA COLLECTION LAYER                            │
│                                                                         │
│  +-----------+    +-----------+    +-----------+    +-----------+       │
│  | Retailer  |    | Retailer  |    | Retailer  |    | RSS/Atom  |       │
│  | API       |    | Scraper   |    | Affiliate |    | Price     |       │
│  | Connector |    | (Playwright|   | Feed      |    | Feeds     |       │
│  | Service   |    |  + Proxy  |    | Processor |    | Parser    |       │
│  +-----+-----+    |  rotation)|    +-----+-----+    +-----+-----+       │
│        |           +-----+-----+         |                |             │
│        +----------------+----------------+----------------+             │
│                         |                                               │
│                  +------v------+                                        │
│                  |    Kafka    |  (raw-price-events topic)              │
│                  +------+------+                                        │
└─────────────────────────|───────────────────────────────────────────────┘
                          |
┌─────────────────────────|───────────────────────────────────────────────┐
│                  PROCESSING LAYER                                       │
│                         |                                               │
│                  +------v------+                                        │
│                  | Flink       |                                        │
│                  | Pipeline    |                                        │
│                  |             |                                        │
│                  | 1. Dedup    | (same retailer, same product, < 5 min) │
│                  | 2. Validate | (price > 0, currency correct,         │
│                  |             |  < 50% change from last known)         │
│                  | 3. Enrich   | (add product_id via matching engine)   │
│                  | 4. Publish  |                                        │
│                  +------+------+                                        │
│                         |                                               │
│          +--------------+--------------+                                │
│          |              |              |                                │
│   +------v------+ +----v-----+ +------v------+                         │
│   | Product     | | Price    | | Anomaly     |                         │
│   | Matching    | | Alert    | | Detector    |                         │
│   | Engine      | | Checker  | | (spike/drop |                         │
│   | (UPC, NLP,  | | (Flink   | |  > 50% from |                        │
│   |  Siamese    | |  joined  | |  rolling    |                         │
│   |  Network)   | |  with    | |  median)    |                         │
│   |             | |  alert   | |             |                         │
│   |             | |  rules)  | |             |                         │
│   +------+------+ +----+-----+ +------+------+                         │
│          |              |              |                                │
└──────────|──────────────|──────────────|────────────────────────────────┘
           |              |              |
┌──────────|──────────────|──────────────|────────────────────────────────┐
│                    DATA LAYER                                           │
│          |              |              |                                │
│   +------v------+ +----v-----+ +------v------+                         │
│   | PostgreSQL  | | Notif    | | ClickHouse  |                         │
│   | (Products,  | | Service  | | (Price      |                         │
│   |  Current    | | (Email,  | |  History,   |                         │
│   |  Prices,    | |  Push)   | |  Anomaly    |                         │
│   |  Alerts)    | +----------+ |  Log)       |                         │
│   +------+------+              +------+------+                         │
│          |                            |                                 │
│   +------v------+              +------v------+                         │
│   |   Redis     |              | S3          |                         │
│   | (price      |              | (scraped    |                         │
│   |  cache per  |              |  page HTML  |                         │
│   |  product,   |              |  snapshots  |                         │
│   |  search     |              |  for audit) |                         │
│   |  cache)     |              +-------------+                         │
│   +------+------+                                                      │
│          |                                                              │
│   +------v------+                                                      │
│   |Elasticsearch|                                                      │
│   | (product    |                                                      │
│   |  search +   |                                                      │
│   |  faceted    |                                                      │
│   |  filtering) |                                                      │
│   +-------------+                                                      │
└────────────────────────────────────────────────────────────────────────┘

SERVING LAYER:
  Client --> CDN (product pages) --> API Gateway --> Comparison Service
                                                        |
                                        +---------------+---------------+
                                        |               |               |
                                     Redis           PostgreSQL    Elasticsearch
                                   (price cache)    (current       (search + 
                                                     prices)        filters)
```

### Component Deep Dive

#### Product Matching — The Hardest Problem

```
Same product, different retailers:
  Amazon: "Apple AirPods Pro 2nd Gen USB-C" - $199.99
  BestBuy: "AirPods Pro (2nd generation) with USB-C" - $189.99
  Walmart: "Apple AirPods Pro 2 USB-C Charging" - $194.00

How to identify these as the SAME product?

Layer 1: UPC/EAN/GTIN matching (if available)
  Universal Product Code is globally unique per product
  If two listings share UPC -> same product (99.9% confidence)
  Problem: many listings don't include UPC

Layer 2: Brand + Model Number matching
  Extract brand ("Apple") + model ("AirPods Pro 2nd Gen") from title
  NER (Named Entity Recognition) to extract structured attributes
  Normalize: "2nd Gen" = "2nd generation" = "Gen 2"
  Match if brand + model match after normalization

Layer 3: ML-based fuzzy matching
  Features: title similarity (cosine of TF-IDF), category match,
    price proximity (same product should have similar price), image similarity
  Model: Siamese network or gradient boosted classifier
  Output: P(same_product) for each pair of listings
  Threshold: > 0.9 -> auto-match, 0.7-0.9 -> human review, < 0.7 -> different

Scaling: 100M products x 50 retailers = 5B potential pairs -> impossible to compare all
  Blocking: only compare products in same category + similar price range
  Reduces to ~1M comparisons -> tractable
```

#### Price Ingestion Pipeline

```
Three data sources:

1. Affiliate APIs (best quality):
   Amazon Product Advertising API, eBay API, etc.
   Structured data: price, availability, shipping, images
   Rate limited: ~1 request/sec per API key
   Coverage: major retailers only

2. Data Feeds (bulk):
   Retailers provide CSV/XML feeds daily with all products + prices
   Process: download feed -> parse -> match products -> update prices
   Coverage: retailers with affiliate programs

3. Web Scraping (fill gaps):
   Crawl retailer websites for products without API/feed
   Challenges: anti-bot measures, dynamic JS rendering, rate limiting
   Legal: must comply with robots.txt and terms of service
   Use: headless browser (Playwright) + residential proxy rotation
   
Pipeline:
  Source -> Kafka (raw price events) -> Flink (dedup, validate, match)
    -> PostgreSQL (current prices) + ClickHouse (price history)
  
  Validation:
  - Price is positive and reasonable (not $0.01 for a laptop)
  - Currency is correct
  - Product URL is still valid
  - Price change < 50% from previous (flag for review if larger)
```

#### Price Alert System

```
User sets alert: "Notify me when AirPods Pro drops below $180"

Naive: For each price update, check all alerts for that product
  500M price updates/day x avg 5 alerts per product = 2.5B checks/day -> feasible

Implementation:
  1. Alerts stored in PostgreSQL:
     alerts: { alert_id, user_id, product_id, target_price, active }
     Index on (product_id, active, target_price)

  2. Price update arrives for product X at $175:
     SELECT user_id FROM alerts 
     WHERE product_id = 'X' AND active = true AND target_price >= 175

  3. For each matching user: send push/email notification
     Mark alert as triggered: UPDATE alerts SET active = false

  Optimization: batch check
    Flink consumer: accumulate price updates for 1 minute
    Batch query: SELECT * FROM alerts WHERE product_id IN (updated_products)
                 AND target_price >= current_price AND active = true
    Send notifications in batch -> reduces DB round trips
```

---

## 5. APIs

```http
GET /api/v1/products/{product_id}/prices
--> { "product": { "name": "AirPods Pro 2", "image": "..." },
      "prices": [
        { "retailer": "Amazon", "price": 199.99, "shipping": "Free",
          "url": "https://...", "in_stock": true, "updated_at": "..." },
        { "retailer": "BestBuy", "price": 189.99, ... }
      ],
      "price_history": { "30d_low": 179.99, "30d_high": 249.99, "current_vs_avg": "-12%" } }

GET /api/v1/search?q=airpods+pro&sort=price_asc&min_price=100&max_price=300

POST /api/v1/alerts
{ "product_id": "prod-123", "target_price": 180.00 }
```

---

## 6. Data Model

### PostgreSQL — Products & Current Prices

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY, name TEXT, brand VARCHAR(100),
    category VARCHAR(100), upc VARCHAR(20), image_url TEXT,
    attributes JSONB, created_at TIMESTAMPTZ
);
CREATE TABLE current_prices (
    product_id UUID, retailer_id VARCHAR(50),
    price DECIMAL(10,2), currency CHAR(3), shipping_cost DECIMAL(10,2),
    in_stock BOOLEAN, url TEXT, updated_at TIMESTAMPTZ,
    PRIMARY KEY (product_id, retailer_id)
);
CREATE TABLE price_alerts (
    alert_id UUID PRIMARY KEY, user_id UUID, product_id UUID,
    target_price DECIMAL(10,2), active BOOLEAN DEFAULT TRUE,
    INDEX idx_product_active (product_id, active, target_price)
);
```

### ClickHouse — Price History
```sql
CREATE TABLE price_history (
    product_id UUID, retailer_id String, price Float32,
    in_stock UInt8, recorded_at DateTime, recorded_date Date
) ENGINE = MergeTree() PARTITION BY toYYYYMM(recorded_at)
  ORDER BY (product_id, retailer_id, recorded_at);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Stale prices** | TTL-based freshness; show "last updated X hours ago"; flag stale |
| **Retailer API down** | Serve last known price with staleness indicator; retry with backoff |
| **Wrong product match** | Human review queue for low-confidence matches; user "report wrong match" |
| **Price scraping blocked** | Proxy rotation, rate limiting, fallback to affiliate API |
| **Alert notification lost** | Kafka at-least-once; dedup by (alert_id, trigger_time) |

---

## 8. Deep Dive

### Total Cost Comparison (Not Just Price)

```
Price alone is misleading:
  Amazon: $189.99 + Free shipping + no tax in some states = $189.99
  SmallRetailer: $179.99 + $9.99 shipping + $14.40 tax = $204.38

Show total_cost = price + shipping + estimated_tax
Sort by total_cost, not just price. This is what smart shoppers actually care about.
Tax estimation: use user's zip code + product category + state tax rules.
```

### Web Scraper Architecture — Crawl Scheduling and Anti-Detection

```
Not all 100M products need the same crawl frequency:

Tier 1 (Hot products, top 1%): crawl every 1 hour
  Products with > 1000 daily views or active price alerts
  ~1M products x 24 crawls/day = 24M crawls/day

Tier 2 (Popular, top 10%): crawl every 6 hours
  ~10M products x 4 crawls/day = 40M crawls/day

Tier 3 (Long tail, 89%): crawl every 24-48 hours
  ~89M products x 0.5 crawls/day = 44.5M crawls/day

Total: ~108M crawls/day = ~1,250 crawls/sec

Scraper fleet:
  200 scraper workers (Kubernetes pods)
  Each worker: headless Chromium (Playwright) + residential proxy
  Rate limit: 1 request/sec/worker per retailer domain
  
  Anti-detection measures:
  - Rotate residential IPs (pool of 10K+ IPs)
  - Randomize User-Agent, viewport, timezone, language headers
  - Add random delays (2-5 sec) between requests
  - Execute JavaScript (for SPAs that render prices client-side)
  - CAPTCHA solving integration (2Captcha/hCaptcha solver as fallback)
  - Respect robots.txt and crawl-delay directives

  Monitoring:
  - Track success rate per retailer (alert if < 90%)
  - Track block rate (alert if > 5%)
  - Track price extraction accuracy (compare with known prices from APIs)

Crawl scheduler (PostgreSQL + Redis):
  crawl_queue table: { product_id, retailer_id, next_crawl_at, priority, last_status }
  Worker polls: SELECT product_id, url FROM crawl_queue 
                WHERE next_crawl_at <= NOW() ORDER BY priority DESC LIMIT 100
  After crawl: UPDATE next_crawl_at = NOW() + interval based on tier
```

### Edge Case: Price Manipulation Detection

```
Problem: Retailers may inflate prices before a "sale" to show larger discounts.
  Original price: $100. Retailer raises to $150, then "50% off" = $75.
  Misleading: actual discount from true price is only 25%.

Detection:
  1. Track rolling 90-day median price per product per retailer
  2. If current "original price" > 120% of 90-day median -> flag as potentially manipulated
  3. Show users: "Lowest price in 90 days: $75" and "Average price: $100"
  4. This is exactly what CamelCamelCamel does for Amazon prices

Storage: ClickHouse materialized view for 90-day rolling stats:
  SELECT product_id, retailer_id, 
         median(price) as median_90d, min(price) as min_90d
  FROM price_history 
  WHERE recorded_at > now() - INTERVAL 90 DAY
  GROUP BY product_id, retailer_id
```

