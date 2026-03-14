# 55. Design Foursquare (Check-ins and Recommendations)

---

## 1. Functional Requirements (FR)

- **Check-in**: Users check in at venues (restaurants, bars, parks) with optional text/photo
- **Venue discovery**: Search nearby venues by category, distance, rating
- **Recommendations**: Personalized venue suggestions based on user history, preferences, location, time of day
- **Venue pages**: Rich profiles with photos, tips, hours, menu, ratings
- **Tips**: Users leave short tips/reviews for venues
- **Lists**: Curated venue lists ("Best Brunch in NYC", "My Favorites")
- **Friends**: Social layer — see where friends checked in, friend recommendations
- **Mayorships**: Gamification — most frequent check-in at a venue earns "Mayor" status
- **Trending**: Show what's trending nearby right now
- **Explore feed**: Discovery feed of nearby popular/new venues

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Nearby venue search in < 200 ms; recommendations in < 500 ms
- **Location Accuracy**: Venue results relevant to user's exact location (not 10 km away)
- **Scalability**: 100M+ registered venues, 50M+ MAU, 10M+ check-ins/day
- **Freshness**: New venues appear in search within minutes; check-in counts update in real-time
- **Personalization**: Recommendations improve with user history
- **Availability**: 99.99%
- **Privacy**: Users control check-in visibility (public, friends-only, private)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total venues | 100M |
| DAU | 15M |
| Check-ins / day | 10M |
| Check-ins / sec | ~115 (peak 500) |
| Venue search queries / sec | 10K |
| Recommendation queries / sec | 5K |
| Tips / day | 2M |
| Average venue data size | 2 KB |
| Total venue data | 100M × 2 KB = 200 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│  Mobile App  │
└──────┬───────┘
       │
┌──────▼───────────────────────────────┐
│            API Gateway               │
└──────┬───────┬───────┬───────┬───────┘
       │       │       │       │
┌──────▼──┐ ┌──▼─────┐ ┌▼──────┐ ┌▼───────────┐
│ Check-in│ │ Venue  │ │Search │ │Recommend.  │
│ Service │ │ Service│ │Service│ │Service     │
│         │ │ (CRUD) │ │(Nearby│ │(Personal-  │
│         │ │        │ │ + text│ │ ized venue │
│         │ │        │ │search)│ │ discovery) │
└────┬────┘ └───┬────┘ └──┬───┘ └─────┬──────┘
     │          │          │           │
     │     ┌────▼────┐  ┌──▼────────┐  │
     │     │PostgreSQL│  │Elastic-   │  │
     │     │+ PostGIS │  │search     │  │
     │     │(Venues,  │  │(Full-text │  │
     │     │ Users)   │  │+ geo)     │  │
     │     └─────────┘  └───────────┘  │
     │                                  │
┌────▼────────────────────────────────┐ │
│              Kafka                   │ │
│  (check-in events, venue updates)   │ │
└────┬───────────┬────────────────────┘ │
     │           │                      │
┌────▼────┐ ┌───▼──────┐ ┌────────────▼┐
│ Feed    │ │ Mayor-   │ │  ML Feature │
│ Service │ │ ship     │ │  Pipeline   │
│ (Friend │ │ Service  │ │  (Spark →   │
│  check- │ │ (Track   │ │   Redis for │
│  ins)   │ │  counts) │ │   rec model)│
└────┬────┘ └──────────┘ └─────────────┘
     │
┌────▼────┐
│ Redis   │
│ (Feed   │
│  cache, │
│  mayor, │
│  trending│
│  venue   │
│  scores) │
└─────────┘
```

### Component Deep Dive

#### Venue Discovery — Nearby Search

```
User opens app at (37.7749, -122.4194) and wants "Italian restaurants within 2 km"

Two approaches:

Approach 1: PostGIS spatial query
  SELECT venue_id, name, category, rating, 
         ST_Distance(location, ST_MakePoint(-122.4194, 37.7749)::geography) as dist
  FROM venues
  WHERE category = 'italian_restaurant'
    AND ST_DWithin(location, ST_MakePoint(-122.4194, 37.7749)::geography, 2000)
  ORDER BY dist
  LIMIT 20;
  
  ✓ Accurate, supports complex filters
  ✗ DB query per request → may not handle 10K QPS

Approach 2: Elasticsearch geo query ⭐ (recommended for search)
  {
    "query": {
      "bool": {
        "must": [
          {"term": {"category": "italian_restaurant"}},
          {"range": {"rating": {"gte": 4.0}}}
        ],
        "filter": {
          "geo_distance": {
            "distance": "2km",
            "location": {"lat": 37.7749, "lon": -122.4194}
          }
        }
      }
    },
    "sort": [
      {"_geo_distance": {"location": {"lat": 37.7749, "lon": -122.4194}, "order": "asc"}}
    ]
  }
  
  ✓ Full-text + geo + filtering in one query
  ✓ Handles 10K+ QPS with proper cluster sizing
  ✓ Text search: "best pizza near me" → text relevance + geo distance

Recommended: Elasticsearch for user-facing search; PostGIS for admin/analytics queries
```

#### Check-in Flow

```
User checks in at "Tartine Bakery":

1. Client sends: POST /api/v1/checkins
   { venue_id, lat, lng, text, photo_url, visibility }

2. Check-in Service:
   a. Validate venue exists
   b. Verify user location is within 200m of venue (anti-fraud)
      Distance = haversine(user_lat/lng, venue_lat/lng) < 200m
   c. Create check-in record in PostgreSQL
   d. Publish to Kafka "check-in-events" topic

3. Downstream consumers:
   a. Feed Service: Add to friends' feeds
   b. Mayorship Service: Increment user's check-in count for this venue
   c. Venue Stats: Update venue's check-in count, popularity score
   d. Recommendation Pipeline: Update user's preference profile
   e. Trending Service: Update trending venues for this area

Anti-fraud: Location validation
  Problem: Users fake GPS to check in at venues they haven't visited
  
  Defenses:
  - Distance check: user GPS must be within 200m of venue
  - Velocity check: can't check in at venues 100 km apart within 10 minutes
  - Rate limit: max 10 check-ins per hour (prevents scripted fraud)
  - Device fingerprinting: flag suspicious devices
  - Pattern detection: checking into 50 venues in one day is suspicious
```

#### Recommendation Service — Personalized Venue Discovery

```
"Explore" feature: "Places you might like near you right now"

Recommendation signals:

1. User Profile Vector (built from check-in history):
   {
     "cuisine_preferences": {"italian": 0.8, "japanese": 0.7, "mexican": 0.5},
     "price_preference": "moderate",  // $$ 
     "typical_visit_times": {"lunch": 0.6, "dinner": 0.3, "brunch": 0.1},
     "avg_rating_threshold": 4.0,
     "categories": {"restaurant": 0.7, "cafe": 0.2, "bar": 0.1}
   }

2. Collaborative Filtering:
   "Users similar to you also liked these venues"
   Similarity: users who checked into same venues → matrix factorization
   
3. Context-Aware Features:
   - Time of day: 8 AM → suggest cafes; 7 PM → suggest restaurants
   - Day of week: Saturday brunch vs Tuesday lunch
   - Weather: rainy → suggest indoor venues
   - Past velocity: traveling → suggest tourist attractions; local → suggest hidden gems

4. Venue Quality Signals:
   - Rating (weighted by recency)
   - Check-in count (popularity)
   - Tip sentiment (NLP on tips)
   - Photo count and quality
   - Hours (only show open venues)

ML Model Architecture:
  Offline: Spark job computes user embeddings and venue embeddings nightly
    User embedding: 128-dimensional vector based on check-in history
    Venue embedding: 128-dimensional vector based on features + collaborative filtering
    
  Online serving:
    1. User opens Explore → send (user_id, lat, lng, time) to Recommendation Service
    2. Candidate generation: 
       Find venues within 5 km → filter by open hours → ~500 candidates
    3. Scoring:
       For each candidate: score = dot_product(user_embedding, venue_embedding) 
                                  × recency_boost × distance_decay × context_multiplier
    4. Rank by score → return top 20
    5. Cache recommendations in Redis (TTL: 15 minutes, invalidate on location change)

  Latency budget: 500 ms total
    Candidate retrieval: 50 ms (ES geo query)
    Embedding lookup: 10 ms (Redis)
    Scoring + ranking: 40 ms (in-memory)
    Total: ~100 ms (well within budget)
```

#### Mayorship — Gamification Logic

```
"Mayor" = user with most check-ins at a venue in the last 60 days

Data structure (Redis sorted set per venue):
  Key: mayor:{venue_id}
  Members: user_ids
  Scores: check-in count in last 60 days

On check-in:
  ZINCRBY mayor:{venue_id} 1 {user_id}
  
  Current mayor = ZREVRANGE mayor:{venue_id} 0 0 WITHSCORES
  
  If the check-in user's score > current mayor's score:
    → New mayor! Send notification to both old and new mayor
    → Update venue page

Expiry of old check-ins (sliding window):
  Problem: Redis sorted set counts don't decay over time
  
  Solution 1: Daily cron job
    For each venue: recalculate check-in counts for last 60 days from PostgreSQL
    Update Redis sorted set scores
    
  Solution 2: Sliding window with timestamps
    Store individual check-in timestamps, not counts
    ZADD mayor:{venue_id} {timestamp} {user_id}:{checkin_id}
    Periodically ZREMRANGEBYSCORE to remove entries > 60 days old
    Count per user: ZRANGEBYSCORE + count by user_id prefix
    
    Trade-off: More memory but real-time accuracy
  
  Recommended: Solution 1 (daily recalc) — mayorship doesn't need second-level accuracy
```

---

## 5. APIs

### Check In
```http
POST /api/v1/checkins
{
  "venue_id": "v-uuid",
  "lat": 37.7749,
  "lng": -122.4194,
  "shout": "Best croissants ever! 🥐",
  "photo_url": "https://cdn.example.com/photos/abc.jpg",
  "visibility": "friends"
}
Response: 201 Created
{
  "checkin_id": "ci-uuid",
  "venue": {"name": "Tartine Bakery", "category": "Bakery"},
  "points_earned": 5,
  "is_mayor": false,
  "mayor": {"user_id": "u-mayor", "name": "Alice", "checkin_count": 42}
}
```

### Search Nearby Venues
```http
GET /api/v1/venues/search?q=pizza&lat=37.7749&lng=-122.4194&radius=2000&category=restaurant&min_rating=4.0&limit=20
Response: 200 OK
{
  "venues": [
    {"venue_id": "v-uuid", "name": "Tony's Pizza", "category": "Pizza",
     "location": {"lat": 37.776, "lng": -122.418}, "distance_meters": 350,
     "rating": 4.7, "checkin_count": 12500, "tips_count": 450,
     "price_tier": 2, "hours": {"open": true, "closes_at": "22:00"}}
  ]
}
```

### Get Explore Recommendations
```http
GET /api/v1/explore?lat=37.7749&lng=-122.4194&limit=20&time=2025-03-14T19:00:00Z
Response: 200 OK
{
  "recommendations": [
    {"venue_id": "v-uuid", "name": "Nopa", "category": "American",
     "reason": "Popular with people who like the places you visit",
     "score": 0.92, "distance_meters": 1200, "rating": 4.6,
     "friend_checkins": 3}
  ]
}
```

### Add Tip
```http
POST /api/v1/venues/{venue_id}/tips
{
  "text": "Try the almond croissant — it's life-changing!",
  "photo_url": "https://cdn.example.com/photos/tip.jpg"
}
Response: 201 Created
{ "tip_id": "tip-uuid" }
```

### Get Friends' Recent Check-ins
```http
GET /api/v1/feed/friends?limit=20&cursor={last}
Response: 200 OK
{
  "checkins": [
    {"checkin_id": "ci-uuid", "user": {"name": "Bob", "avatar": "..."},
     "venue": {"name": "Blue Bottle Coffee"}, "shout": "Morning fuel ☕",
     "created_at": "2025-03-14T08:15:00Z"}
  ]
}
```

---

## 6. Data Model

### PostgreSQL + PostGIS — Core Data

```sql
-- Venues
CREATE TABLE venues (
    venue_id        UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    category_id     INT NOT NULL,
    subcategory     VARCHAR(100),
    location        GEOMETRY(Point, 4326) NOT NULL,
    lat             DECIMAL(10,7),
    lng             DECIMAL(10,7),
    address         TEXT,
    city            VARCHAR(100),
    country_code    CHAR(2),
    phone           VARCHAR(20),
    website         TEXT,
    price_tier      SMALLINT,           -- 1-4 ($-$$$$)
    rating          DECIMAL(2,1),
    rating_count    INT DEFAULT 0,
    checkin_count   INT DEFAULT 0,
    tip_count       INT DEFAULT 0,
    photo_count     INT DEFAULT 0,
    hours           JSONB,
    verified        BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    SPATIAL INDEX idx_location (location),
    INDEX idx_category_city (category_id, city),
    INDEX idx_name_trgm USING gin (name gin_trgm_ops)  -- fuzzy text search
);

-- Check-ins
CREATE TABLE checkins (
    checkin_id      UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    venue_id        UUID NOT NULL,
    lat             DECIMAL(10,7),
    lng             DECIMAL(10,7),
    shout           TEXT,
    photo_url       TEXT,
    visibility      ENUM('public', 'friends', 'private') DEFAULT 'friends',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id, created_at DESC),
    INDEX idx_venue (venue_id, created_at DESC),
    INDEX idx_venue_user (venue_id, user_id, created_at DESC)  -- for mayorship
);

-- Tips
CREATE TABLE tips (
    tip_id          UUID PRIMARY KEY,
    venue_id        UUID NOT NULL,
    user_id         UUID NOT NULL,
    text            TEXT NOT NULL,
    photo_url       TEXT,
    upvote_count    INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_venue (venue_id, upvote_count DESC)
);
```

### Elasticsearch — Venue Search Index

```json
{
  "mappings": {
    "properties": {
      "venue_id": {"type": "keyword"},
      "name": {"type": "text", "analyzer": "standard", "fields": {"autocomplete": {"type": "search_as_you_type"}}},
      "category": {"type": "keyword"},
      "location": {"type": "geo_point"},
      "rating": {"type": "float"},
      "checkin_count": {"type": "integer"},
      "price_tier": {"type": "integer"},
      "hours": {"type": "object"},
      "city": {"type": "keyword"},
      "tags": {"type": "keyword"}
    }
  }
}
```

### Redis — Caches and Real-time Data

```
# Mayorship per venue (sorted set: user → check-in count)
mayor:{venue_id}         → Sorted Set { user_id: count }

# Venue hot cache
venue:{venue_id}         → Hash { name, category, rating, lat, lng, hours }
TTL: 3600

# User recommendation cache
recs:{user_id}:{geohash} → List of venue_ids (pre-computed recommendations)
TTL: 900 (15 minutes)

# Trending venues per city
trending:{city}          → Sorted Set { venue_id: check-in count in last hour }
TTL: 3600

# User embeddings for recommendations
user_emb:{user_id}       → Binary (128-dim float32 vector = 512 bytes)
TTL: 86400

# Friends feed cache
friend_feed:{user_id}    → List of recent friend check-in IDs
TTL: 300
```

### Kafka Topics

```
Topic: checkin-events        (partitioned by user_id)
Topic: venue-updates         (rating changes, new tips, new photos)
Topic: trending-updates      (aggregated venue popularity snapshots)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Check-in location fraud** | Haversine distance check + velocity check + rate limiting |
| **Elasticsearch down** | Fallback to PostGIS for search (degraded but functional) |
| **Recommendation service down** | Serve cached recs from Redis; fallback to popularity-based (non-personalized) |
| **Mayorship race condition** | Redis ZINCRBY is atomic; concurrent check-ins handled correctly |
| **Venue data inconsistency** | Elasticsearch synced from PostgreSQL via CDC (Debezium) |
| **Photo upload failure** | Pre-signed S3 URL; retry from client; check-in created without photo |
| **Feed staleness** | Cache TTL + publish-subscribe for real-time updates to online friends |

### Specific: Check-in Count Consistency

```
Problem: Venue's checkin_count in PostgreSQL vs actual check-in rows can diverge

Scenario:
  T=0: checkin_count = 1000
  T=1: User A checks in → INSERT checkin → UPDATE venue SET checkin_count = checkin_count + 1
  T=1: User B checks in → INSERT checkin → UPDATE venue SET checkin_count = checkin_count + 1
  
  If both transactions read count=1000 before the other commits:
    Both write count=1001 → actual count should be 1002!
  
Solution: PostgreSQL handles this correctly with row-level locking.
  UPDATE venues SET checkin_count = checkin_count + 1 WHERE venue_id = ?
  This is an atomic increment — PostgreSQL serializes concurrent updates to the same row.
  
  But at very high scale (10K check-ins/sec to one viral venue):
    Row lock contention → performance degradation
  
  Better: Don't update count synchronously. Use eventual consistency:
    1. Check-in insert → Kafka event
    2. Counter worker: batch count updates per venue every 5 seconds
    3. UPDATE venues SET checkin_count = checkin_count + {batch_delta}
    4. One update per 5 seconds per venue instead of one per check-in
```

---

## 8. Deep Dive: Engineering Trade-offs

### Venue Deduplication — The Hardest Problem

```
Problem: Multiple sources submit the same venue with slightly different info
  Source 1: "Tartine Bakery" at (37.7614, -122.4241)
  Source 2: "Tartine bakery & cafe" at (37.7615, -122.4239)
  User submission: "tartine" at (37.7613, -122.4242)
  
  Are these the same venue? How to detect and merge?

Dedup pipeline:
  1. Normalize: lowercase, remove punctuation, standardize abbreviations
     "tartine bakery" and "tartine bakery cafe" and "tartine"
  
  2. Geo-cluster: group venues within 50m of each other
  
  3. Name similarity: Jaccard similarity or edit distance on normalized names
     "tartine bakery" vs "tartine bakery cafe" → Jaccard = 2/3 = 0.67 → likely match
  
  4. Category match: both are "bakery" → strong signal
  
  5. ML model: combine features → predict P(same_venue)
     Features: name_similarity, geo_distance, category_match, phone_match, website_match
     Threshold: P > 0.85 → auto-merge
     P between 0.5-0.85 → human review queue
     P < 0.5 → separate venues

  6. Merge strategy: keep the record with most data; combine unique attributes
```

### Time-Aware Recommendations: Why Time of Day Matters

```
Same user, same location, different times → completely different recommendations:

8:00 AM Tuesday:
  Boost: cafes (+3×), breakfast spots (+2×)
  Suppress: bars (-5×), nightclubs (-10×)
  Features: "quick", "takeout", "coffee"

12:30 PM Tuesday:
  Boost: restaurants (+3×), lunch specials (+2×)
  Suppress: bars (-3×), late-night spots (-5×)
  Features: "lunch", "fast-casual", "affordable"

9:00 PM Friday:
  Boost: bars (+3×), upscale restaurants (+2×), live music (+2×)
  Suppress: breakfast spots (-5×)
  Features: "dinner", "cocktails", "date night"

Implementation:
  24-hour cycle encoded as features in recommendation model:
    time_features = [sin(2π × hour/24), cos(2π × hour/24)]  // circular encoding
    day_features = [is_weekday, is_weekend]
  
  Model learns time-dependent preferences from check-in history:
    "User A checks into cafes at 8 AM and bars at 9 PM"
    → weight cafe features high in morning, bar features high in evening
```

### Privacy-Preserving Location History

```
Foursquare knows every venue a user has ever visited.
This is extremely sensitive data.

Privacy measures:
  1. Data minimization: store venue_id, not raw GPS coordinates
     (venue location is public; user's exact GPS within the venue is not)
  
  2. Check-in aging: after 2 years, remove specific timestamps
     Keep: "User visited Venue X approximately 15 times"
     Delete: exact dates and times
  
  3. Recommendation model: user embeddings are irreversible
     Can't reconstruct check-in history from embedding vector
  
  4. Friend visibility: check-in location shared only with approved friends
     Server-side filtering: never send private check-ins to unauthorized users
  
  5. Export/delete: GDPR compliance → user can download all data and request deletion
     Deletion cascades: checkins, tips, ratings, embeddings, feeds
```



