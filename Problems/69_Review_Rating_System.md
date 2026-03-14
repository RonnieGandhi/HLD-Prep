# 69. Design a Review and Rating System

---

## 1. Functional Requirements (FR)

- **Write reviews**: Text reviews with 1-5 star rating for products/services
- **Photo/video reviews**: Attach media to reviews
- **Helpful votes**: Other users vote reviews as "helpful" or "not helpful"
- **Verified purchase badge**: Mark reviews from actual buyers
- **Aggregate ratings**: Average rating, rating distribution histogram, total count per product
- **Sort/filter reviews**: By recency, helpfulness, rating, verified purchase
- **Seller responses**: Sellers can respond publicly to reviews
- **Edit/delete own reviews**: Users manage their own reviews
- **Anti-abuse**: Detect and filter fake/spam reviews via ML + rules
- **Review summary**: AI-generated summary of common themes across reviews

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Aggregate ratings in < 50 ms (shown on every product page)
- **Eventual Consistency**: New review reflected in aggregates within 5 minutes
- **Scale**: 500M+ products, 5B+ reviews, 200K new reviews/day
- **Availability**: 99.99% read; 99.9% write
- **Fraud Resistant**: Detect and suppress fake reviews within hours

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total products | 500M |
| Total reviews | 5B |
| New reviews / day | 200K |
| Review reads / sec | 100K |
| Aggregate rating reads / sec | 500K |
| Total review storage | 5 TB text + 100 TB media |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    WRITE PATH (new review)                             │
│                                                                        │
│  Client --> API Gateway --> Review Service                              │
│                                  |                                     │
│         +------------------------+---------------------------+         │
│         |                        |                           |         │
│  +------v-------+         +------v-------+           +-------v------+  │
│  | Review       |         | Purchase     |           | Photo/Video  |  │
│  | Validator    |         | Verification |           | Upload       |  │
│  | (rate limit, |         | Service      |           | Service      |  │
│  |  text check, |         | (check if    |           | (S3 + CDN,   |  │
│  |  duplicate   |         |  user bought |           |  resize,     |  │
│  |  detection)  |         |  product)    |           |  moderate)   |  │
│  +------+-------+         +------+-------+           +-------+------+  │
│         |                        |                           |         │
│  +------v------------------------------------------------v--v------+  │
│  |                    PostgreSQL                                    |  │
│  | reviews (UNIQUE product_id + user_id)                            |  │
│  | product_ratings (pre-computed aggregates)                        |  │
│  | review_votes, seller_responses                                   |  │
│  +------+---------------------------+-------------------------------+  │
│         |                           |                                  │
│  +------v-------+           +-------v-------+                         │
│  |    Kafka     |           |    Redis      |                         │
│  | (review      |           | - rating:{id} |                         │
│  |  events)     |           |   aggregate   |                         │
│  +------+-------+           |   cache       |                         │
│         |                   | - vote dedup  |                         │
│         |                   +---------------+                         │
│    +----+----------+----------+----------+                            │
│    |    |          |          |          |                            │
│ +--v--+ +v------+ +v-------+ +v--------+ +v---------+               │
│ |Aggr-| |Fraud  | |Elastic | |AI       | |Notif     |               │
│ |egate| |Detect | |search  | |Summary  | |Service   |               │
│ |Worker| |Service| |Index   | |Generator| |(notify   |               │
│ |(incr | |(ML:   | |(full-  | |(weekly  | | seller   |               │
│ | avg, | | XGB,  | | text   | | batch,  | | of new   |               │
│ | hist)| | NLP)  | | review | | LLM per | | reviews) |               │
│ +------+ +-------+ | search)| | product)| +---------+               │
│                     +--------+ +---------+                            │
└────────────────────────────────────────────────────────────────────────┘

READ PATH (product page):
  Client --> CDN --> API Gateway --> Review Service
                                         |
                              +----------+----------+
                              |                     |
                        +-----v------+        +-----v------+
                        |   Redis    |        | PostgreSQL |
                        | (aggregate |        | (paginated |
                        |  cache,    |        |  reviews,  |
                        |  99% hit   |        |  sorted by |
                        |  rate)     |        |  Wilson    |
                        +------------+        |  score)    |
                                              +------------+
```

### Component Deep Dive

#### Aggregate Rating — Pre-Computed, Not Calculated

```
Problem: Product has 50,000 reviews. Computing AVG on every request kills the DB.

Solution: Pre-computed aggregates in PostgreSQL + Redis cache.

PostgreSQL table:
  product_ratings: { product_id, avg_rating, total_reviews, star_1..star_5, updated_at }

On new review (via Kafka consumer):
  UPDATE product_ratings SET
    star_{rating} = star_{rating} + 1,
    total_reviews = total_reviews + 1,
    avg_rating = (avg_rating * (total_reviews - 1) + {new_rating}) / total_reviews
  WHERE product_id = ?;
  
  Invalidate Redis: DEL rating:{product_id}

Redis (500K reads/sec):
  rating:{product_id} --> Hash { avg, total, s1, s2, s3, s4, s5 }
  TTL: 300 seconds. Cache miss --> read PostgreSQL --> populate.

Why not COUNT/AVG on reviews table?
  50K rows scan per query x 500K QPS = DB death.
  Pre-computed: O(1) lookup, < 1 ms.
```

#### Bayesian Average — Not Simple Average

```
Product A: 1 review, 5.0 avg. Product B: 10K reviews, 4.5 avg.
Simple average ranks A higher. That's statistically wrong.

Bayesian Average (what Amazon/IMDB actually use):
  bayesian_avg = (C * m + sum_of_ratings) / (C + total_reviews)
  
  C = 25 (minimum reviews to trust)
  m = 3.7 (global average rating)
  
  Product A: (25*3.7 + 5) / (25+1) = 3.75
  Product B: (25*3.7 + 45000) / (25+10000) = 4.498
  
  Product B correctly ranks higher. As review count grows,
  bayesian_avg converges to true average.
```

#### Fake Review Detection

```
Signals for ML fraud detection:

Behavioral:
  - Account age < 7 days + review posted -> suspicious
  - 50 reviews in 1 day -> bot
  - All 5-star or all 1-star -> biased
  - Copy-pasted text across products

Purchase verification:
  - No purchase record -> unverified (lower weight in aggregates)
  - Purchased and returned same day -> suspicious

Text analysis (NLP):
  - Generic text: "Great product! Highly recommended!" (no specifics)
  - Sentiment mismatch: positive text + 1 star
  - Language similarity between reviews from same IP

Network analysis:
  - Multiple reviews from same IP/device for same product
  - Review rings: group of accounts all reviewing same products
  
Timing:
  - Product gets 500 5-star reviews in 1 hour (normally 5/day) -> burst

ML Model (XGBoost):
  Features: all signals above. Output: P(fake) score.
  P(fake) > 0.8: auto-suppress + queue for human review
  P(fake) 0.5-0.8: flag, reduced weight in aggregate
  P(fake) < 0.5: show normally
  
  Suppressed reviews excluded from aggregate rating.
  Nightly recomputation corrects for retroactively suppressed reviews.
```

---

## 5. APIs

### Write Review
```http
POST /api/v1/products/{product_id}/reviews
{
  "rating": 4,
  "title": "Great quality, slow shipping",
  "body": "The product quality is excellent but shipping took 2 weeks...",
  "photos": ["https://cdn.../photo1.jpg"]
}
Response: 201 Created
{ "review_id": "rev-uuid", "status": "published" }
```

### Get Reviews for Product
```http
GET /api/v1/products/{product_id}/reviews?sort=helpful&rating=4&verified=true&page=1&limit=10
Response: 200 OK
{
  "aggregate": {
    "avg_rating": 4.3, "total": 50000,
    "distribution": {"5": 30000, "4": 10000, "3": 5000, "2": 3000, "1": 2000}
  },
  "reviews": [
    { "review_id": "rev-uuid", "user": {"name": "John D.", "verified": true},
      "rating": 4, "title": "Great quality", "body": "...",
      "helpful_count": 234, "photos": [...], "created_at": "2026-03-10",
      "seller_response": {"body": "Thank you for your feedback!", "responded_at": "2026-03-11"} }
  ]
}
```

### Vote Helpful
```http
POST /api/v1/reviews/{review_id}/helpful
{ "helpful": true }
Response: 200 OK
{ "helpful_count": 235 }
```

---

## 6. Data Model

### PostgreSQL — Reviews

```sql
CREATE TABLE reviews (
    review_id       UUID PRIMARY KEY,
    product_id      VARCHAR(50) NOT NULL,
    user_id         UUID NOT NULL,
    order_id        UUID,
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title           VARCHAR(200),
    body            TEXT,
    photos          JSONB,
    verified_purchase BOOLEAN DEFAULT FALSE,
    helpful_count   INT DEFAULT 0,
    unhelpful_count INT DEFAULT 0,
    status          ENUM('published','suppressed','pending_review','deleted') DEFAULT 'published',
    fraud_score     DECIMAL(4,3),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ,
    UNIQUE (product_id, user_id),  -- one review per user per product
    INDEX idx_product (product_id, status, created_at DESC),
    INDEX idx_product_helpful (product_id, status, helpful_count DESC),
    INDEX idx_user (user_id, created_at DESC)
);

CREATE TABLE product_ratings (
    product_id      VARCHAR(50) PRIMARY KEY,
    avg_rating      DECIMAL(3,2),
    total_reviews   INT DEFAULT 0,
    star_1 INT DEFAULT 0, star_2 INT DEFAULT 0, star_3 INT DEFAULT 0,
    star_4 INT DEFAULT 0, star_5 INT DEFAULT 0,
    updated_at      TIMESTAMPTZ
);

CREATE TABLE review_votes (
    user_id         UUID NOT NULL,
    review_id       UUID NOT NULL,
    helpful         BOOLEAN NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, review_id)
);

CREATE TABLE seller_responses (
    response_id     UUID PRIMARY KEY,
    review_id       UUID NOT NULL UNIQUE,
    seller_id       UUID NOT NULL,
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis
```
# Aggregate ratings cache
rating:{product_id}            --> Hash { avg, total, s1, s2, s3, s4, s5 }
TTL: 300

# Helpful vote dedup
voted:{user_id}:{review_id}    --> "1"
TTL: 86400

# Review count per user per day (rate limit)
review_rate:{user_id}:{date}   --> INT
TTL: 86400
```

### Elasticsearch — Review Search
```json
{
  "review_id": "rev-uuid", "product_id": "prod-123",
  "rating": 4, "title": "Great quality",
  "body": "The product quality is excellent but shipping was slow...",
  "verified": true, "helpful_count": 234, "created_at": "2026-03-10"
}
// Queries: "battery life" within reviews of product X
// Filter by: rating >= 4, verified only, sort by helpful
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Review written but aggregate not updated** | Kafka at-least-once delivery to aggregation worker; idempotent UPDATE |
| **Duplicate review** | UNIQUE(product_id, user_id) constraint prevents duplicates |
| **Rating cache stale** | TTL 5 min; invalidated on write; worst case 5-min lag |
| **Fake review flood** | Rate limit per user (max 5 reviews/day); async ML detection |
| **Vote spam** | One vote per user per review (DB primary key constraint + Redis dedup) |
| **Aggregate drift** | Nightly batch: recompute all aggregates from reviews table; overwrite running values |

### Specific: Aggregate Rating Drift Over Time

```
Incremental average accumulates floating-point errors after millions of updates.
Running avg after 50K updates: 4.29999999987. Actual avg: 4.30000000000.

Fix: Nightly full recomputation batch job:
  SELECT product_id, AVG(rating), COUNT(*),
         SUM(CASE WHEN rating=1 THEN 1 ELSE 0 END) as s1, ...
  FROM reviews WHERE status = 'published'
  GROUP BY product_id;

Overwrite product_ratings with exact values.
Also corrects for reviews suppressed/deleted since last recomputation.
```

---

## 8. Deep Dive: Engineering Trade-offs

### Review Ordering: Wilson Score Interval

```
Problem: Sort by helpful_count is biased toward old reviews (more time to accumulate votes).

Wilson Score balances helpfulness RATIO with confidence interval:
  wilson = (p + z^2/(2n) - z*sqrt((p(1-p) + z^2/(4n))/n)) / (1 + z^2/n)
  
  p = helpful_votes / total_votes, n = total_votes, z = 1.96 (95% confidence)
  
  Review A: 10/10 helpful -> wilson = 0.72
  Review B: 100/200 helpful -> wilson = 0.46
  
  10/10 correctly ranks higher despite fewer total votes.
  This is what Reddit uses for comment ranking. Amazon uses a similar approach.
```

### AI Review Summaries

```
Instead of reading 50K reviews, show:
  "Customers frequently mention excellent battery life (45%), lightweight design (38%).
   Common complaints: slow charging (12%), screen glare (8%)."

Implementation:
  1. Weekly batch per product (products with > 50 reviews)
  2. Collect all review texts -> topic extraction (BERT NER / LDA)
  3. Identify top 5 positive and top 3 negative themes
  4. Generate summary using LLM (GPT-4 / Claude)
  5. Cache in product_review_summary table + Redis
  
  Cost: ~$0.01 per summary. Products with > 50 reviews: ~10M.
  Total: $100K one-time. Refresh weekly for active products (~$20K/week).
```

