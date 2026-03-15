# 20. Design a Proximity Server (Yelp / Nearby Friends)

---

## 1. Functional Requirements (FR)

- **Search nearby places**: Given user's location and a radius, return nearby businesses/places (restaurants, gas stations, etc.)
- **Filter by category**: Filter results by type (restaurant, hospital, ATM, etc.)
- **Sort by**: Distance, rating, popularity, relevance
- **Business details**: Name, address, phone, hours, photos, reviews
- **Add/update business**: Business owners can add and update their listings
- **Reviews and ratings**: Users can write reviews and rate businesses
- **Nearby Friends** (variant): Show friends who are nearby in real-time

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Nearby search results in < 100 ms
- **High Availability**: 99.99%
- **Read-Heavy**: 100:1 read to write (business data changes infrequently)
- **Scalability**: 200M+ businesses, 500M+ DAU
- **Geospatial Efficiency**: Efficient spatial queries over large datasets
- **Real-time** (for Nearby Friends): Location updates within seconds

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total businesses | 200M |
| Business record size | 1 KB |
| Total business data | 200 GB |
| Search QPS | 100K |
| Business updates / day | 1M (low frequency) |
| Nearby Friends: Active users streaming location | 10M |
| Location updates / sec | 2.5M (10M users × 1 update per 4s) |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                           Clients                                     │
│                                                                       │
│  ┌─────────────────┐              ┌─────────────────┐                │
│  │ Search Nearby   │              │ Nearby Friends  │                │
│  │ Businesses      │              │ (Real-Time)     │                │
│  │ (REST)          │              │ (WebSocket)     │                │
│  └────────┬────────┘              └────────┬────────┘                │
└───────────┼────────────────────────────────┼─────────────────────────┘
            │                                │
┌───────────▼────────────────────────────────▼─────────────────────────┐
│                         API Gateway                                   │
│  Auth, rate limiting, geo-routing                                     │
└───────────┬────────────────────────────────┬─────────────────────────┘
            │                                │
┌───────────▼──────────┐          ┌──────────▼───────────────┐
│  Proximity Search    │          │  Nearby Friends          │
│  Service             │          │  Service (Real-Time)     │
│                      │          │                          │
│  1. Parse lat/lng    │          │  • WebSocket per user    │
│  2. Compute GeoHash  │          │  • Receive location      │
│  3. Query QuadTree   │          │    updates every 4s      │
│     or Redis Geo     │          │  • Query friends' locs   │
│  4. Filter by radius │          │  • Push nearby list      │
│  5. Enrich from cache│          │                          │
└───────┬──────┬───────┘          └──────┬─────┬─────────────┘
        │      │                         │     │
        │      │                    ┌────▼─────▼────┐
        │      │                    │  Redis Geo    │
        │      │                    │  (User Locs)  │
        │      │                    │               │
        │      │                    │  GEOADD       │
        │      │                    │  user_locations│
        │      │                    │  {lng} {lat}  │
        │      │                    │  {user_id}    │
        │      │                    └───────────────┘
        │      │
┌───────▼──────┴───────┐
│  Geospatial Index    │
│                      │
│  Option A: In-Memory │       ┌───────────────────┐
│  QuadTree Servers    │       │  Redis            │
│  (replicated, each   │       │  (Business Cache) │
│   holds full tree:   │       │                   │
│   200M biz × 32B     │       │  biz:{id} → Hash  │
│   = 6.4 GB per node) │       │  { name, category,│
│                      │       │    lat, lng,       │
│  Option B: Redis     │       │    rating, ... }   │
│  GeoIndex            │       │  TTL: 3600        │
│  GEOADD businesses:  │       └───────────────────┘
│  {category}          │
│  {lng} {lat} {biz_id}│
│  GEORADIUS 5km       │
└──────────┬───────────┘
           │
┌──────────▼───────────┐
│  MySQL               │
│  (Business Listings) │
│                      │
│  Source of truth for  │
│  all business data.  │
│  QuadTree rebuilt     │
│  from MySQL nightly.  │
│  Incremental updates │
│  via Kafka CDC.      │
└──────────────────────┘
```

### Core Algorithm: Geospatial Indexing — Deep Dive

#### Approach 1: GeoHash

**How GeoHash works**:
1. Divide the world into 2 halves (east/west) → bit 0 or 1
2. Divide each half vertically (north/south) → another bit
3. Repeat alternately (longitude, latitude) to desired precision
4. Encode the bit string as Base32 → "9q8yyk8yuv"

```
GeoHash Precision:
  Length 1: ~5,000 km cell
  Length 4: ~39 km cell
  Length 5: ~4.9 km cell  ← Good for nearby search
  Length 6: ~1.2 km cell  ← Good for walking distance
  Length 7: ~150 m cell
  Length 8: ~38 m cell
```

**Key property**: Nearby locations share a common GeoHash prefix. "9q8yy" and "9q8yz" are adjacent cells.

**Search algorithm**:
1. Compute user's GeoHash at precision 5 (4.9 km cells)
2. Find all businesses in user's cell: `WHERE geohash LIKE '9q8yy%'`
3. Also search the 8 neighboring cells (edge cases: user near cell boundary)
4. Filter by actual distance (GeoHash cells are rectangular approximations)
5. Sort by distance

**Pros**: Simple, works with standard B-tree indexes in any SQL database
**Cons**: Edge cases at cell boundaries, uneven cell sizes near poles

#### Approach 2: QuadTree ⭐ (recommended for in-memory)

**How QuadTree works**:
- Start with the entire world as the root node
- Recursively divide each node into 4 quadrants (NW, NE, SW, SE)
- Stop subdividing when a node has ≤ K businesses (e.g., K=100)
- Leaf nodes contain the actual business locations

```
Root (World)
├── NW (Americas)
│   ├── NW (Canada)
│   ├── NE (NE US) → [biz1, biz2, ..., biz87]  (leaf: 87 businesses)
│   ├── SW (SW US)
│   │   ├── NW (Bay Area) → [biz1, ..., biz95]  (leaf)
│   │   ├── NE (Central CA)
│   │   └── ...
│   └── SE (SE US)
├── NE (Europe/Asia)
└── ...
```

**Search algorithm**:
1. Start at root, traverse to leaf containing user's location
2. Collect all businesses in that leaf
3. If not enough results (radius extends beyond leaf) → check parent's sibling leaves
4. Continue expanding until enough results found or radius exceeded
5. Filter by actual distance

**Building QuadTree**: O(N log N), where N = number of businesses
**Search**: O(log N + M), where M = businesses in result
**Memory**: 200M businesses × 32 bytes = ~6.4 GB (fits in memory on a single server)
**Rebuild**: Every night (businesses don't change frequently). Incremental updates during the day

#### Approach 3: Google S2 Geometry

- Uses a space-filling curve (Hilbert curve) to map 2D space to 1D
- Each cell has a 64-bit Cell ID
- **Covering**: For a circular region, S2 finds the minimal set of cells that cover the area
- Query: `SELECT * FROM businesses WHERE s2_cell_id IN (cell1, cell2, ..., cellN)`
- **Pros**: More uniform cell sizes than GeoHash, efficient coverings
- **Used by**: Google Maps, Foursquare

### Component Deep Dive

#### Proximity Search Service
1. Receive query: `{lat, lng, radius, category, sort_by}`
2. Query geospatial index for businesses in radius
3. Filter by category
4. Rank by sort criteria (distance, rating, relevance)
5. Enrich with business details from MySQL cache
6. Return paginated results

#### Business Service
- CRUD operations for business listings
- On create/update → update MySQL → async update geospatial index
- Index rebuild is periodic (nightly) for full consistency
- Hot businesses cached in Redis

#### Kafka CDC — Incremental QuadTree Updates
- Full QuadTree rebuild from MySQL runs nightly (~5 minutes for 200M businesses)
- During the day, new/updated businesses can't wait until the nightly rebuild
- **Debezium CDC** on MySQL captures row-level changes → publishes to Kafka topic `business-changes`
- QuadTree server consumes the topic:
  - **Insert**: Add business to the appropriate leaf node; if leaf exceeds K=100, split into 4 quadrants
  - **Update** (location changed): Remove from old leaf, insert into new leaf
  - **Update** (metadata only): Update in-place (no tree restructuring)
  - **Delete/deactivate**: Remove from leaf node
- Same CDC stream also updates Redis business cache (`biz:{id}`) and Redis GeoIndex (`GEOADD`)
- Lag: business changes reflected in search within ~2 seconds

---

## 5. APIs

### Search Nearby
```http
GET /api/v1/search/nearby?lat=37.7749&lng=-122.4194&radius=5000&category=restaurant&sort=distance&limit=20
Response: 200 OK
{
  "businesses": [
    {
      "business_id": "biz-uuid",
      "name": "Joe's Pizza",
      "category": "restaurant",
      "address": "123 Main St, San Francisco, CA",
      "lat": 37.7751,
      "lng": -122.4190,
      "distance_meters": 45,
      "rating": 4.5,
      "review_count": 234,
      "price_level": "$$",
      "is_open": true,
      "photo_url": "..."
    }
  ],
  "total": 85
}
```

### Get Business Details
```http
GET /api/v1/businesses/{business_id}
```

### Add/Update Business
```http
POST /api/v1/businesses
PUT /api/v1/businesses/{business_id}
```

### Nearby Friends (Real-Time Variant)
```http
WebSocket /ws/v1/nearby-friends
{
  "type": "location_update",
  "lat": 37.7749,
  "lng": -122.4194
}

Server Push:
{
  "type": "nearby_friends",
  "friends": [
    {"user_id": "...", "name": "Alice", "distance_meters": 120, "last_updated": "10s ago"}
  ]
}
```

---

## 6. Data Model

### MySQL — Business Listings

```sql
CREATE TABLE businesses (
    business_id     BIGINT PRIMARY KEY,
    name            VARCHAR(256),
    category        VARCHAR(64),
    address         TEXT,
    city            VARCHAR(128),
    state           VARCHAR(64),
    country         VARCHAR(64),
    lat             DECIMAL(10,7),
    lng             DECIMAL(10,7),
    geohash         VARCHAR(12),
    phone           VARCHAR(20),
    website         TEXT,
    price_level     TINYINT,
    rating          DECIMAL(2,1),
    review_count    INT DEFAULT 0,
    hours           JSON,
    photos          JSON,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    INDEX idx_geohash (geohash),
    INDEX idx_category_geohash (category, geohash),
    INDEX idx_city (city)
);
```

### In-Memory QuadTree Node

```java
class QuadTreeNode {
    double x1, y1, x2, y2;    // bounding box
    List<Business> businesses; // non-null only for leaf nodes (max 100)
    QuadTreeNode NW, NE, SW, SE; // children (null for leaves)
}

class Business {
    long businessId;
    double lat, lng;
    String category;
    float rating;
}
```

### Redis — Business Cache & GeoIndex

```
# Business details cache
Key:    biz:{business_id}
Value:  Hash { name, category, lat, lng, rating, ... }
TTL:    3600

# Redis GeoIndex (alternative to in-memory QuadTree)
GEOADD businesses:restaurant {lng} {lat} {business_id}
GEORADIUS businesses:restaurant {lng} {lat} 5 km WITHCOORD WITHDIST COUNT 20 ASC
```

### Redis — Nearby Friends (Real-Time)

```
# User's current location
GEOADD user_locations {lng} {lat} {user_id}

# Query friends nearby
for friend_id in user.friends:
    GEODIST user_locations {user_id} {friend_id} m
    if distance < 5000:
        add to result
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **QuadTree server failure** | Multiple replicas; rebuild from MySQL in < 5 minutes |
| **MySQL read failure** | Read replicas in multiple AZs |
| **Stale geospatial index** | Incremental updates during day; full rebuild nightly |
| **Cache miss storm** | Request coalescing + stale-while-revalidate |

### Specific: GeoHash Boundary Problem
- A user at the edge of a GeoHash cell might miss businesses just across the border
- **Solution**: Always search the 8 neighboring cells in addition to the user's cell
- For QuadTree: expand search to sibling nodes when results near the boundary

---

## 8. Additional Considerations

### Scaling the QuadTree
- 200M businesses × 32 bytes ≈ 6.4 GB → fits on a single server
- For redundancy: replicate to multiple servers (read replicas)
- For extremely large datasets: shard by region (US, Europe, Asia)
- Each shard handles a geographic region independently

### Nearby Friends — Real-Time Architecture
- Different from business search (businesses are static; friends move)
- **Architecture**: 
  - Each user streams location every 4 seconds via WebSocket
  - Location stored in Redis Geo Set
  - When user opens "Nearby Friends": query Redis for each friend's distance
  - Optimization: Don't query all 1000 friends. Use GeoHash to pre-filter → only check friends in same/adjacent GeoHash cells

### Density-Based Search Radius
- In Manhattan (dense): default radius = 500 m
- In rural areas: default radius = 50 km
- Adaptive: start with small radius, expand until >= 10 results

### Business Hours and Open/Closed Status
- Store hours per day as JSON: `{"mon": {"open": "09:00", "close": "22:00"}, ...}`
- On search: filter by "open now" using user's timezone
- Timezone determined from user's location (timezone lookup table)

