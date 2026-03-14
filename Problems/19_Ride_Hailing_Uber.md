---

## 1. Functional Requirements (FR)

- **Rider**: Request a ride by specifying pickup and dropoff locations
- **Driver**: Go online/offline, accept/decline ride requests, navigate to pickup
- **Matching**: Match riders with nearby available drivers in real-time
- **ETA**: Show estimated time of arrival for pickup and trip
- **Pricing**: Dynamic pricing (surge pricing) based on supply-demand
- **Real-time tracking**: Both rider and driver see each other's live location on a map
- **Trip lifecycle**: Request → Match → Pickup → In-Trip → Dropoff → Payment → Rating
- **Payment**: Charge rider, pay driver (commission model)
- **Rating**: Mutual rating (rider rates driver, driver rates rider)
- **Ride history**: View past trips, receipts

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Match rider with driver in < 5 seconds
- **High Availability**: 99.99% — downtime means stranded riders
- **Real-time**: Location updates every 3-5 seconds from all active drivers
- **Scalability**: 100M+ riders, 5M+ drivers, 20M+ rides/day
- **Consistency**: Ride-to-driver matching must be exactly one-to-one (no double booking)
- **Geo-distributed**: Multi-city, multi-country
- **Fault Tolerant**: A matched ride must never be lost

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active drivers at any time | 2M |
| Location updates / sec | 2M ÷ 4s = 500K |
| Rides / day | 20M |
| Rides / sec | ~230 (peak 1,000) |
| Location update size | 100 bytes |
| Location data / day | 500K × 100B × 86400 = ~4 TB |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐
│  Rider App   │    │  Driver App  │
└──────┬───────┘    └──────┬───────┘
       │                    │
       │ REST/WS           │ WS (location stream)
       │                    │
┌──────▼────────────────────▼───────┐
│           API Gateway             │
└──────┬────────────────────┬───────┘
       │                    │
       │              ┌─────▼──────────┐
       │              │ Location       │
       │              │ Service        │  ← Ingests 500K loc updates/sec
       │              └─────┬──────────┘
       │                    │
       │              ┌─────▼──────────┐
       │              │ Location       │
       │              │ Index          │  ← Geospatial index (in-memory)
       │              │ (QuadTree /    │
       │              │  GeoHash)      │
       │              └────────────────┘
       │
┌──────▼───────┐    ┌──────────────┐    ┌──────────────┐
│  Ride        │    │  Matching    │    │  Pricing     │
│  Service     │◄──▶│  Service     │◄──▶│  Service     │
│ (Trip CRUD)  │    │ (Driver      │    │ (Surge,      │
│              │    │  Selection)  │    │  ETA, Fare)  │
└──────┬───────┘    └──────────────┘    └──────────────┘
       │
┌──────▼───────┐    ┌──────────────┐    ┌──────────────┐
│  Payment     │    │  Notification│    │  Map / Route │
│  Service     │    │  Service     │    │  Service     │
└──────────────┘    └──────────────┘    │  (OSRM/Google│
                                        │   Maps API)  │
                                        └──────────────┘

Data Stores:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │   Kafka      │
│ (Trips, Users│  │ (Driver Loc, │  │ (Events,     │
│  Payments)   │  │  Sessions)   │  │  Location)   │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Location Service — Handling 500K Location Updates/Sec

**Ingestion**:
1. Driver app sends GPS coordinates every 4 seconds via WebSocket
2. Location Service receives and updates the **in-memory geospatial index**
3. Also publishes to Kafka `driver-location` topic (for analytics and trip tracking)

**Geospatial Index — How to Find Nearby Drivers**:

**Option 1: GeoHash + Redis**
```
GeoHash: Encode lat/lng into a string. Nearby points share a prefix.
  37.7749, -122.4194 → "9q8yyk"
  37.7751, -122.4190 → "9q8yym"  (nearby → shares prefix "9q8yy")

Redis:
  GEOADD drivers:available {lng} {lat} {driver_id}
  GEORADIUS drivers:available {lng} {lat} 5 km COUNT 20
```
- **Pros**: Simple, Redis handles it natively, O(log N + M)
- **Cons**: Redis single-threaded bottleneck for 500K updates/sec

**Option 2: QuadTree** ⭐ (recommended for Uber-scale)
- Recursively subdivide 2D space into 4 quadrants
- Each leaf node contains drivers in that area (max ~100 drivers per leaf)
- **Search**: Start from root, navigate to leaf containing rider's location, expand to neighboring cells
- **Update**: Remove driver from old cell, add to new cell
- **In-memory**: Entire QuadTree fits in memory (~2M drivers × 64 bytes = 128 MB)
- **Sharding**: Partition the map into regions (cities); each region has its own QuadTree on a dedicated server

**Option 3: S2 Geometry (Google's approach)**
- Maps earth's surface onto a sphere, then to a cube, then to 6 squares
- Each cell has a unique 64-bit Cell ID
- Hierarchical: larger cells contain smaller cells
- Uber actually uses H3 (Hexagonal Hierarchical Spatial Index)

**H3 (Uber's Actual Index)**:
- Hexagonal grid system
- Each hex cell has a unique ID at various resolutions (0-15)
- Resolution 9 ≈ 0.1 km² per cell
- Finding neighbors is O(1) — hexagons tile evenly
- Index: `Map<H3CellId, List<DriverId>>`

#### Matching Service — The Core Algorithm

**Step 1**: Rider requests a ride at location (lat, lng)

**Step 2**: Find nearby available drivers (within 5 km radius)
- Query geospatial index → returns 20 candidate drivers with distances

**Step 3**: Rank candidates by:
- Distance to rider (shorter is better)
- ETA to rider (accounts for traffic, not just straight-line distance)
- Driver rating
- Acceptance rate
- Vehicle type match (UberX, UberXL, Black)

**Step 4**: Send ride request to top-ranked driver

**Step 5**: Driver has 15 seconds to accept/decline
- If declined or timed out → send to next driver in ranked list
- If accepted → create trip, notify rider

**Consistency**: Use distributed lock on driver_id to prevent double-matching:
```
Redis: SET driver:lock:{driver_id} {trip_id} NX EX 30
  NX = only set if not exists (atomic)
  EX = expire in 30 seconds
```

#### Pricing Service — Surge Pricing

**Surge multiplier** based on local supply-demand:
```python
surge = demand_in_area / supply_in_area

if surge > 1.0:
    # Cap at 8x; round to nearest 0.1x
    surge = min(surge, 8.0)
    fare = base_fare * surge
```

**Implementation**:
- Divide city into hexagonal cells (H3 resolution 7)
- Count: riders requesting rides in each cell (demand)
- Count: available drivers in each cell (supply)
- Compute surge per cell every 30 seconds
- Store in Redis: `surge:{city}:{h3_cell}` → multiplier

**Fare Calculation**:
```
fare = base_fare 
     + (per_minute_rate × trip_duration_minutes)
     + (per_mile_rate × trip_distance_miles)
     + booking_fee
     + tolls
fare = fare × surge_multiplier
fare = max(fare, minimum_fare)
```

#### ETA Service
- Uses routing engine (OSRM, Google Maps Directions API, or Uber's in-house)
- **Pre-computation**: For each driver candidate, compute ETA to rider (road distance, not straight line)
- **Real-time traffic**: Aggregate driver speed data across road segments → build live traffic model
- ETA for trip: route from pickup to dropoff considering current traffic

---

## 5. APIs

### Request Ride
```http
POST /api/v1/rides
{
  "rider_id": "user-uuid",
  "pickup": {"lat": 37.7749, "lng": -122.4194, "address": "..."},
  "dropoff": {"lat": 37.7849, "lng": -122.4094, "address": "..."},
  "ride_type": "UberX"
}
Response: 201 Created
{
  "ride_id": "ride-uuid",
  "status": "matching",
  "estimated_fare": {"min": 12.50, "max": 16.00, "surge": 1.2},
  "estimated_pickup_eta": "4 min"
}
```

### Driver Accept/Decline
```http
POST /api/v1/rides/{ride_id}/accept
POST /api/v1/rides/{ride_id}/decline
```

### Update Driver Location (WebSocket)
```json
{
  "type": "location_update",
  "driver_id": "driver-uuid",
  "lat": 37.7752,
  "lng": -122.4190,
  "heading": 45,
  "speed": 25,
  "timestamp": 1710320000
}
```

### Get Ride Status
```http
GET /api/v1/rides/{ride_id}
Response: 200 OK
{
  "ride_id": "...",
  "status": "in_trip",
  "driver": {"name": "...", "vehicle": "...", "rating": 4.9, "location": {...}},
  "pickup": {...},
  "dropoff": {...},
  "eta_to_dropoff": "12 min"
}
```

---

## 6. Data Model

### PostgreSQL — Trips (ACID required for financial data)

```sql
CREATE TABLE trips (
    trip_id         UUID PRIMARY KEY,
    rider_id        UUID NOT NULL,
    driver_id       UUID,
    status          ENUM('matching','driver_assigned','en_route_pickup',
                         'arrived','in_trip','completed','cancelled'),
    ride_type       VARCHAR(20),
    pickup_lat      DECIMAL(10,7),
    pickup_lng      DECIMAL(10,7),
    dropoff_lat     DECIMAL(10,7),
    dropoff_lng     DECIMAL(10,7),
    pickup_address  TEXT,
    dropoff_address TEXT,
    surge_multiplier DECIMAL(3,1),
    estimated_fare  DECIMAL(10,2),
    actual_fare     DECIMAL(10,2),
    distance_miles  DECIMAL(8,2),
    duration_minutes DECIMAL(8,2),
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    created_at      TIMESTAMP,
    INDEX idx_rider (rider_id, created_at DESC),
    INDEX idx_driver (driver_id, created_at DESC)
);
```

### Redis — Driver Availability & Location

```
# Geospatial index
GEOADD drivers:available:{city} {lng} {lat} {driver_id}

# Driver state
Key:    driver:state:{driver_id}
Value:  Hash { status: "available|on_trip|offline", trip_id, lat, lng, heading, updated_at }

# Surge pricing
Key:    surge:{city}:{h3_cell_id}
Value:  1.5
TTL:    60 seconds (refreshed every 30s)

# Driver lock (prevent double-matching)
Key:    driver:lock:{driver_id}
Value:  trip_id
TTL:    30 seconds
```

### Kafka Topics

```
Topic: driver-locations       (high throughput, partitioned by driver_id)
Topic: trip-events            (lifecycle events: created, matched, started, completed)
Topic: surge-updates          (pricing updates per cell)
```

### Cassandra — Location History (for trip reconstruction)

```sql
CREATE TABLE trip_location_trail (
    trip_id     UUID,
    timestamp   TIMESTAMP,
    lat         DECIMAL,
    lng         DECIMAL,
    speed       FLOAT,
    PRIMARY KEY (trip_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp ASC)
  AND default_time_to_live = 2592000;  -- 30 days
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Matching service failure** | Retry matching; rider sees "looking for driver..." until timeout |
| **Location service lag** | Stale driver locations (within 30s) are still usable for matching |
| **Payment failure** | Retry with idempotency key; fallback to post-trip charging |
| **Driver app crash** | Trip state persisted in PostgreSQL; driver reconnects and resumes |
| **Network partition** | Driver/rider continue trip offline; sync state when reconnected |
| **Double matching** | Redis distributed lock with NX flag ensures atomic driver assignment |

### Specific: What Happens if a Matched Driver Goes Offline?
1. Driver doesn't send heartbeat for 60 seconds
2. System detects driver offline → trip status = "driver_unavailable"
3. Auto-reassign: Re-enter matching for rider with priority boost
4. Notify rider: "Finding a new driver..."
5. Original driver penalized (lower acceptance rate score)

---

## 8. Additional Considerations

### Geofencing
- Define virtual boundaries (airports, stadiums, city limits)
- When driver enters a geofence → auto-queue for airport rides
- When rider requests from a geofence → apply special pricing/rules
- H3 hexagons make geofence membership checks efficient

### Ride Pooling (UberPool / Share)
- Match multiple riders with similar routes
- **Algorithm**: For each new request, check if existing in-progress rides have a detour < 5 minutes
- Computationally expensive: requires real-time route comparison
- Riders get discounted fare; driver gets paid for the full route

### Safety Features
- Trip sharing: Rider shares trip status with trusted contacts
- Emergency button: Direct call to emergency services with live location
- Driver identity verification: Periodic selfie check
- Route deviation alerts: If driver deviates significantly from planned route

### Analytics & Monitoring
- Real-time dashboards: rides in progress, available drivers per city, surge heatmap
- Demand forecasting: ML model predicts demand per area per hour → proactively incentivize drivers to position there
- Trip completion rate, average wait time, cancellation rate

---

## 9. Deep Dive: Engineering Trade-offs

### Geospatial Index: QuadTree vs GeoHash vs H3 vs S2

| Index | How It Works | Pros | Cons | Used By |
|---|---|---|---|---|
| **GeoHash** | Encode lat/lng to string; nearby points share prefix | Simple, works with any DB index | Edge cases at cell boundaries; rectangular cells | Redis (GEOADD), Elasticsearch |
| **QuadTree** | Recursive 2D subdivision into 4 quadrants | Dynamic (subdivides dense areas more), in-memory efficient | Requires in-memory maintenance; not natively supported by DBs | Yelp, Foursquare |
| **H3** ⭐ | Hierarchical hexagonal grid; each hex has unique ID | Uniform distance properties (hexagons have consistent neighbors); O(1) neighbor lookup; 15 resolution levels | Newer, smaller ecosystem; slight distortion near poles | **Uber (actual)** |
| **S2 Geometry** | Hilbert curve maps sphere to cells; 64-bit cell IDs | Mathematically precise; excellent covering algorithms; handles sphere correctly | Complex implementation; Google-centric ecosystem | **Google Maps**, **Foursquare** |

**Why Uber chose H3 specifically**:
1. **Hexagons tile uniformly** — unlike squares, every neighbor is equidistant from the center. This matters for "find drivers within 2 km" (circles approximate better with hexagons)
2. **Multi-resolution**: Resolution 9 (~174m edge) for precise matching, Resolution 7 (~1.2km edge) for surge pricing zones
3. **O(1) neighbor finding**: `h3.k_ring(cell, k=1)` returns all 6 immediate neighbors instantly
4. **Hierarchical containment**: A resolution-7 cell contains exactly 7 resolution-9 cells → natural aggregation for analytics

### Why WebSocket for Driver Location (Not HTTP Polling)?

```
HTTP Polling (every 4 seconds):
  Each poll = new TCP connection + TLS handshake + HTTP headers
  Per driver: 4 KB overhead × 15 polls/min = 60 KB/min wasted bandwidth
  For 2M drivers: 120 GB/min wasted bandwidth
  
  ✗ Massive bandwidth waste
  ✗ Higher latency (poll interval + network)
  ✗ Server overhead: 2M × 15 = 30M HTTP connections/min

WebSocket (persistent connection) ⭐:
  One connection setup → keep alive with heartbeat
  Each location update: ~100 bytes (raw payload)
  Per driver: 100B × 15/min = 1.5 KB/min (40× less!)
  For 2M drivers: 3 GB/min total
  
  ✓ 40× less bandwidth
  ✓ Real-time (no polling interval delay)
  ✓ Bi-directional (server can push ride requests TO driver)
  ✗ Sticky sessions needed (connection state is on one server)
  ✗ More complex connection management (reconnect logic, load balancing)

Decision: WebSocket. The bandwidth savings alone justify it. Plus, we need 
server-push for ride request delivery to drivers.
```

### Matching Algorithm: Greedy vs Batch Optimization

```
Greedy Matching (assign immediately):
  1. Rider requests ride
  2. Find nearest available driver
  3. Assign immediately
  
  ✓ Lowest latency for the rider (< 5 seconds to match)
  ✗ Globally suboptimal: nearby driver might be better for another rider who requests 10 seconds later
  ✗ Example: Driver D is 2 min from Rider A and 1 min from Rider B. 
    Greedy assigns D to A (who requested first). Now B waits 8 min for a farther driver.

Batch Optimization (collect and optimize):
  1. Collect all ride requests in a 5-second window
  2. Collect all available drivers
  3. Solve assignment problem: minimize total wait time across ALL riders
  4. Algorithm: Hungarian algorithm or min-cost max-flow
  
  ✓ Globally optimal (minimizes total/average wait time across riders)
  ✗ Higher latency (5-second batching window)
  ✗ Computationally expensive (O(n³) for Hungarian algorithm)
  ✗ Complex to implement and debug

Uber's actual approach: Hybrid
  - Low demand: Greedy (fast, good enough)
  - High demand: Batch optimization with 2-second windows
  - The batch window is dynamically adjusted based on request rate
```

### Why PostgreSQL for Trips (Not Cassandra/DynamoDB)?

```
Trip data requires:
  1. ACID transactions (trip state machine transitions must be atomic)
  2. Complex queries (admin: "all trips in city X on date Y with fare > $50")
  3. Foreign key relationships (trip → rider, trip → driver, trip → payment)
  4. Updates (trip status changes 6+ times during lifecycle)
  
PostgreSQL ✓:
  - ACID transactions for state machine transitions
  - Rich query support (JOINs, aggregations, window functions)
  - JSON support for flexible metadata
  - Row-level locking for concurrent updates
  - Sharded by city or region for horizontal scaling

Cassandra ✗:
  - No transactions → can't atomically update trip status + payment status
  - No JOINs → can't easily query "trip with rider and driver details"
  - Updates are actually new rows (immutable model) → complicates state machine
  
DynamoDB ✗:
  - 25-item transaction limit → fine for single trip but limits complex operations
  - Query flexibility limited to partition key + sort key
  - Vendor lock-in

For analytics: Trip events published to Kafka → ClickHouse for OLAP queries.
PostgreSQL serves the OLTP path (real-time trip management).
```

### Surge Pricing: Ethical and Technical Considerations

```
Technical implementation:
  demand = count_ride_requests(cell, last_5_minutes)
  supply = count_available_drivers(cell)
  surge = max(1.0, min(demand / supply, 8.0))  # Cap at 8×

Ethical issues:
  - Surge during emergencies (natural disasters, terrorist attacks)
    → Solution: Auto-cap surge at 1.0 during declared emergencies
  - Perception of price gouging
    → Solution: Show surge multiplier BEFORE booking; let user choose to wait
  
Alternatives to multiplicative surge:
  - Time-based premium: "Your ride will cost $5 extra during peak hours" (flat surcharge)
  - Wait-time option: "You can wait 10 minutes for a non-surge ride"
  - Surge income sharing: Extra surge revenue donated to charity (Lyft tried this)

Why surge pricing is essential for the system to work:
  1. Incentivizes drivers to go to high-demand areas (supply response)
  2. Reduces demand (some riders choose to wait) → balances supply/demand
  3. Without surge, high-demand periods have NO available drivers → worse user experience
```

### Driver Location Update: Accuracy vs Battery Life vs Bandwidth

```
Update every 1 second:
  ✓ Very accurate tracking
  ✗ Battery drain: GPS polling at 1Hz consumes ~15% battery/hour
  ✗ Bandwidth: 100B × 3600 = 360 KB/hour

Update every 4 seconds ⭐ (Uber's actual interval):
  ✓ Good balance: position error ~10-40m (acceptable for matching)
  ✓ Battery: ~5% per hour (acceptable)
  ✓ Bandwidth: 90 KB/hour

Update every 30 seconds:
  ✓ Minimal battery/bandwidth
  ✗ Position error up to 500m → very poor matching quality
  
Adaptive approach (even better):
  - Stationary (speed = 0): Update every 30 seconds
  - Moving slowly (< 10 km/h): Update every 10 seconds
  - Moving normally (> 10 km/h): Update every 4 seconds
  - In an active ride: Update every 2 seconds (rider wants accurate tracking)
```
