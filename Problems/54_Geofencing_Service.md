# 54. Design a Geofencing Service

---

## 1. Functional Requirements (FR)

- **Create geofences**: Define virtual geographic boundaries (polygon, circle, or rectangle) with associated metadata and rules
- **Real-time trigger**: Detect when a device/user enters, exits, or dwells within a geofence
- **Event notifications**: Fire callbacks/webhooks/push notifications on geofence triggers
- **Bulk management**: Support millions of active geofences simultaneously
- **Geofence types**: Static (store boundary), dynamic (moving delivery zone), time-based (active only during hours)
- **Dwell detection**: Trigger after a user stays inside a geofence for N seconds (not just passing through)
- **Geofence groups**: Group fences by category (stores, warehouses, competitor locations)
- **Analytics**: Track entry/exit counts, dwell time, popular times per geofence
- **Multi-tenant**: Support multiple clients (apps) each with their own geofences

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Geofence check (point-in-polygon) in < 10 ms per device update
- **Real-time**: Event triggered within 5 seconds of fence crossing
- **Scale**: 100M+ active geofences, 10M+ devices sending location updates
- **Throughput**: 5M location updates/sec at peak
- **Accuracy**: Handle geofences as small as 50m radius
- **Availability**: 99.99% — geofencing is critical for delivery, security, fleet management
- **Durability**: No trigger events lost (at-least-once delivery)
- **Battery Aware**: Minimize GPS polling frequency on mobile devices

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active geofences | 100M |
| Active devices | 10M concurrently |
| Location updates / sec | 5M peak |
| Avg geofence polygon vertices | 6 |
| Geofence storage per fence | ~500 bytes |
| Total geofence data | 100M × 500B = 50 GB |
| Trigger events / sec | ~50K (0.1% of location updates trigger) |
| Event notification delivery | < 5 seconds |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Mobile App  │    │  IoT Devices │    │ Fleet GPS    │
│  (SDK)       │    │  (Trackers)  │    │  Trackers    │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼───────┐
                     │  API Gateway │
                     │  / LB        │
                     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
       ┌──────▼──────┐ ┌───▼───────┐ ┌───▼──────────┐
       │  Location   │ │ Geofence  │ │ Geofence     │
       │  Ingestion  │ │ Mgmt      │ │ Query        │
       │  Service    │ │ Service   │ │ Service      │
       │  (Kafka     │ │ (CRUD)    │ │ (Point-in-   │
       │   producer) │ │           │ │  fence check)│
       └──────┬──────┘ └───┬───────┘ └───┬──────────┘
              │             │             │
       ┌──────▼──────┐     │      ┌──────▼───────┐
       │    Kafka    │     │      │  Geofence    │
       │  (location  │     │      │  Spatial     │
       │   updates)  │     │      │  Index       │
       └──────┬──────┘     │      │  (In-memory  │
              │             │      │   R-tree/    │
       ┌──────▼──────┐     │      │   Quadtree)  │
       │  Geofence   │     │      └──────────────┘
       │  Evaluation │     │
       │  Engine     │─────┘
       │  (Flink /   │
       │   Workers)  │
       └──────┬──────┘
              │
       ┌──────▼──────┐    ┌──────────────┐
       │    Kafka    │    │  Trigger     │
       │  (trigger   │───▶│  Delivery    │
       │   events)   │    │  Service     │
       └─────────────┘    │ (Webhooks,   │
                          │  Push, SMS)  │
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │  Event       │
                          │  Store       │
                          │  (ClickHouse)│
                          └──────────────┘
```

### Component Deep Dive

#### Geofence Spatial Index — How to Check Millions of Fences Efficiently

```
Problem: Device sends location (lat, lng). Which of 100M geofences contain this point?

Brute force: Check every geofence → 100M × point-in-polygon test = WAY too slow

Solution: Two-phase spatial filtering

Phase 1: Coarse filtering with spatial index (R-tree or Geohash grid)
  - Each geofence has a bounding box (MBR — minimum bounding rectangle)
  - R-tree indexes these bounding boxes
  - Query: "Which bounding boxes contain (lat, lng)?"
  - Result: ~10-50 candidate geofences (from 100M → 50 — huge reduction!)
  - Time: O(log N) where N = number of geofences

Phase 2: Precise point-in-polygon test
  - For each candidate geofence, run ray-casting algorithm
  - Ray from point to infinity; count intersections with polygon edges
  - Odd intersections → inside; Even → outside
  - Time: O(V) per polygon where V = number of vertices
  - For 50 candidates × 6 vertices avg = 300 operations → microseconds

Total: < 1 ms per location update for point-in-fence check

Index choice comparison:

R-tree ⭐:
  - Balanced tree of bounding rectangles
  - Excellent for overlapping polygons (geofences can overlap)
  - Query: O(log N + K) where K = result count
  - Memory: ~100 bytes per geofence → 100M × 100B = 10 GB
  - Used by PostGIS, standard for spatial databases

Geohash Grid:
  - Divide world into grid cells (e.g., geohash precision 6 ≈ 1.2km × 0.6km)
  - Each cell has a list of geofences whose bounding box intersects it
  - Lookup: compute geohash of point → check all fences in that cell + neighbors
  - Simpler than R-tree, works well with Redis/distributed cache
  - Trade-off: cells near geofence boundaries need to check neighboring cells

Quadtree:
  - Recursively subdivide space; each leaf contains geofences in that area
  - Better for non-uniform distribution (dense cities + sparse rural)
  - Dynamic: adapts to geofence density
```

#### Device State Machine — Tracking Enter/Exit/Dwell

```
Each (device, geofence) pair has a state machine:

     ┌──────────┐    point inside     ┌──────────┐
     │ OUTSIDE  │───────────────────▶ │ ENTERED  │
     │          │                     │ (start   │
     │          │◀───────────────────│  dwell   │
     └──────────┘    point outside    │  timer)  │
                                      └────┬─────┘
                                           │ dwell timer > threshold
                                           │
                                      ┌────▼─────┐
                                      │ DWELLING │
                                      │          │
                                      └────┬─────┘
                                           │ point outside
                                      ┌────▼─────┐
                                      │ EXITED   │
                                      │ (emit    │
                                      │  exit    │
                                      │  event)  │
                                      └──────────┘

State storage (Redis):
  Key: device_fence:{device_id}:{fence_id}
  Value: Hash { state, entered_at, last_seen_at }
  TTL: 3600 (cleanup stale entries)

Problem: 10M devices × potentially thousands of fences each = billions of state entries

Optimization: Only track state for fences the device is near
  When device location update arrives:
  1. Spatial query → find ~10 candidate fences near the device
  2. Only maintain state for these 10 fences
  3. If device moves away → state expires via TTL
  
  Active state entries: 10M devices × ~5 active fences = 50M entries
  Redis memory: 50M × 200 bytes = 10 GB → feasible
```

#### Geofence Evaluation Engine — Processing 5M Updates/Sec

```
Architecture: Flink streaming application

Kafka Topic: location-updates
  Key: device_id
  Value: { device_id, lat, lng, accuracy, timestamp, app_id }
  Partitions: 256 (by device_id hash)

Flink Job:
  1. Consume location updates
  2. For each update:
     a. Load device's app_id → determine which tenant's geofences to check
     b. Query spatial index (R-tree served via sidecar or gRPC service)
     c. Get candidate geofences (10-50)
     d. Point-in-polygon check for each candidate
     e. Compare with previous state (Redis lookup)
     f. If state changed (OUTSIDE → ENTERED, DWELLING → EXITED):
        - Emit trigger event to Kafka "trigger-events" topic
        - Update state in Redis
  3. Parallelism: 256 partitions × Flink parallelism → handles 5M/sec

Scaling strategy:
  Partition location updates by geohash region (not just device_id)
  Each Flink task manager handles one geographic region
  Spatial index is sharded by region → each worker loads only its region's fences
  
  Region 1 (NYC):   2M fences, 500K devices → 1 task manager
  Region 2 (LA):    1.5M fences, 400K devices → 1 task manager
  Region 3 (Rural): 100K fences, 50K devices → shared task manager
```

#### Dwell Detection — Preventing False Triggers

```
Problem: User drives through a geofence at highway speed. Should NOT trigger "entered" event.

Dwell detection: Only trigger if user stays inside for > N seconds.

Implementation:
  On ENTER: Start a dwell timer (e.g., 60 seconds)
  Timer stored in Redis with TTL:
    SET dwell_timer:{device}:{fence} 1 EX 60
  
  On each subsequent location update inside the fence:
    Check if timer exists → if yes, still waiting
    If timer expired AND still inside → trigger DWELL event
  
  On EXIT before timer expires:
    Delete timer → no event triggered (pass-through ignored)

Alternative: Sliding window
  Track last N location updates for the device
  If > 80% of updates in last 60 seconds are inside the fence → trigger
  Handles GPS jitter near fence boundaries (oscillating in/out)

GPS Jitter Handling:
  User standing still near fence boundary → GPS oscillates ±10m
  Without hysteresis: ENTER, EXIT, ENTER, EXIT → notification spam
  
  Solution: Hysteresis buffer
    Enter threshold: must be > 10m inside the fence boundary
    Exit threshold: must be > 10m outside the fence boundary
    Dead zone: within 10m of boundary → maintain current state
```

---

## 5. APIs

### Create Geofence
```http
POST /api/v1/geofences
{
  "name": "Downtown Store #42",
  "type": "polygon",
  "coordinates": [
    {"lat": 37.7749, "lng": -122.4194},
    {"lat": 37.7759, "lng": -122.4184},
    {"lat": 37.7739, "lng": -122.4174},
    {"lat": 37.7729, "lng": -122.4184}
  ],
  "radius_meters": null,
  "triggers": ["enter", "exit", "dwell"],
  "dwell_time_seconds": 120,
  "metadata": {"store_id": "s-42", "category": "retail"},
  "active_hours": {"start": "08:00", "end": "22:00", "timezone": "America/New_York"},
  "webhook_url": "https://client.example.com/geofence-events",
  "group_id": "retail-stores"
}
Response: 201 Created
{ "fence_id": "fence-uuid", "status": "active" }
```

### Batch Location Update
```http
POST /api/v1/locations/batch
{
  "device_id": "device-uuid",
  "locations": [
    {"lat": 37.7749, "lng": -122.4194, "accuracy": 10, "timestamp": 1710400000},
    {"lat": 37.7751, "lng": -122.4190, "accuracy": 8, "timestamp": 1710400005}
  ]
}
Response: 200 OK
```

### Get Geofence Events
```http
GET /api/v1/geofences/{fence_id}/events?start=2025-03-14T00:00:00Z&end=2025-03-14T23:59:59Z
Response: 200 OK
{
  "events": [
    {"event_id": "e-uuid", "device_id": "d-uuid", "type": "enter",
     "timestamp": "2025-03-14T10:23:45Z", "lat": 37.7749, "lng": -122.4194,
     "dwell_seconds": null},
    {"event_id": "e-uuid2", "device_id": "d-uuid", "type": "dwell",
     "timestamp": "2025-03-14T10:25:45Z", "dwell_seconds": 120}
  ]
}
```

### Query Active Devices in Geofence
```http
GET /api/v1/geofences/{fence_id}/devices?status=inside
Response: 200 OK
{
  "devices": [
    {"device_id": "d-uuid", "entered_at": "2025-03-14T10:23:45Z",
     "current_location": {"lat": 37.7750, "lng": -122.4192}}
  ],
  "count": 42
}
```

---

## 6. Data Model

### PostgreSQL + PostGIS — Geofence Definitions

```sql
CREATE TABLE geofences (
    fence_id        UUID PRIMARY KEY,
    app_id          UUID NOT NULL,          -- multi-tenant
    name            VARCHAR(255),
    fence_type      ENUM('polygon', 'circle') NOT NULL,
    geometry        GEOMETRY(Polygon, 4326),  -- for polygons
    center_lat      DECIMAL(10,7),            -- for circles
    center_lng      DECIMAL(10,7),
    radius_meters   INT,
    triggers        JSONB,                    -- ["enter", "exit", "dwell"]
    dwell_seconds   INT DEFAULT 0,
    metadata        JSONB,
    group_id        UUID,
    webhook_url     TEXT,
    active          BOOLEAN DEFAULT TRUE,
    active_hours    JSONB,
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    SPATIAL INDEX idx_geometry (geometry),
    INDEX idx_app (app_id, active),
    INDEX idx_group (group_id)
);
```

**Why PostGIS for storage (not just in-memory)?**
- Durable storage — geofences must survive restarts
- Complex spatial queries for management (find all fences intersecting a region)
- Backup and recovery
- In-memory R-tree is built FROM PostGIS data at startup and periodically refreshed

### Redis — Device State + Dwell Timers

```
# Device-to-fence state
device_fence:{device_id}:{fence_id} → Hash { state, entered_at, last_lat, last_lng }
TTL: 3600

# Dwell timer (existence = timer active)
dwell_timer:{device_id}:{fence_id} → "1"
TTL: dwell_seconds configured for the fence

# Device last known location
device_loc:{device_id} → Hash { lat, lng, accuracy, timestamp }
TTL: 300

# Geofence active device count (approximate)
fence_device_count:{fence_id} → INT (INCR on enter, DECR on exit)
```

### Kafka Topics

```
Topic: location-updates
  Key: device_id
  Value: { device_id, app_id, lat, lng, accuracy, timestamp }
  Partitions: 256
  Retention: 24 hours

Topic: geofence-triggers
  Key: fence_id
  Value: { event_id, device_id, fence_id, event_type, timestamp, lat, lng, dwell_seconds }
  Partitions: 64
  Retention: 7 days

Topic: geofence-changes (CDC for fence CRUD — notify evaluation engine)
  Key: fence_id
  Value: { action: "create|update|delete", fence_data }
```

### ClickHouse — Event Analytics

```sql
CREATE TABLE geofence_events (
    event_id        UUID,
    fence_id        UUID,
    device_id       UUID,
    app_id          UUID,
    event_type      Enum8('enter'=1, 'exit'=2, 'dwell'=3),
    timestamp       DateTime,
    lat             Float64,
    lng             Float64,
    dwell_seconds   UInt32,
    event_date      Date MATERIALIZED toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (app_id, fence_id, timestamp);

-- Analytics queries:
-- "How many unique visitors entered Store #42 this week?"
-- "Average dwell time at all retail stores in NYC"
-- "Peak hours for fence entries across all locations"
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Evaluation engine failure** | Flink checkpointing; Kafka consumer group rebalance picks up from offset |
| **Spatial index crash** | Rebuild from PostGIS (takes 2-3 minutes for 100M fences); replicas serve during rebuild |
| **Redis state loss** | Redis Cluster with AOF persistence; worst case: re-derive state from recent location updates |
| **Webhook delivery failure** | Exponential backoff retry (max 5 attempts); DLQ for persistent failures |
| **Duplicate triggers** | Idempotent event processing: deduplicate by (device_id, fence_id, event_type, 5-min window) |
| **GPS accuracy poor** | Ignore updates with accuracy > 100m; use accuracy circle in point-in-polygon (expanded boundary) |
| **Geofence update while evaluating** | Double-buffer spatial index: build new version → atomic swap |

### Specific: Preventing Duplicate Enter/Exit Events

```
Scenario: Network hiccup causes Flink to replay some location updates

Without dedup:
  Update 1: device enters fence → ENTER event
  (replay)
  Update 1 again: device inside fence → no state change (state already ENTERED) → OK
  
State machine naturally deduplicates because transitions are idempotent:
  OUTSIDE + "point inside" → ENTERED (emit event)
  ENTERED + "point inside" → ENTERED (NO event, already entered)
  
BUT: If Redis state was lost between original and replay:
  Update 1 replay: state not found → treat as OUTSIDE → ENTERED (duplicate event!)

Solution: 
  1. Write trigger events to ClickHouse with (device_id, fence_id, event_type, timestamp)
  2. Before emitting, check ClickHouse: "was enter event for this device+fence emitted in last 5 min?"
  3. If yes → skip (deduplicate)
  4. Alternative: Bloom filter per (device, fence) pair with 5-min window for fast dedup
```

### Specific: Geofence Update During Active Sessions

```
Scenario: Admin shrinks geofence boundary. Device was inside, now outside new boundary.

Naive: No immediate effect → device state shows "inside" but it's actually outside now

Solution: 
  On geofence update:
  1. Publish to Kafka "geofence-changes" topic
  2. Evaluation engine receives change
  3. Re-evaluate ALL devices currently "inside" the old fence
  4. Devices now outside the new boundary → emit EXIT event
  5. Update spatial index with new geometry (atomic swap)
  
  This "reconciliation" ensures consistency after fence modification.
```

---

## 8. Deep Dive: Engineering Trade-offs

### Client-Side vs Server-Side Geofencing

```
Client-Side (iOS/Android native geofencing APIs):
  iOS: CLLocationManager.startMonitoring(for: region) — max 20 regions
  Android: GeofencingClient.addGeofences() — max 100 geofences
  
  ✓ Works offline (no server needed)
  ✓ Battery optimized by OS (uses cell towers + WiFi, not just GPS)
  ✓ Instant triggers (no network latency)
  ✗ Limited to 20-100 geofences per device
  ✗ Can't handle millions of fences (our use case)
  ✗ No server-side analytics or cross-device coordination

Server-Side (our design):
  ✓ Unlimited geofences per device
  ✓ Centralized analytics and webhook delivery
  ✓ Cross-device coordination (e.g., "any driver entered the zone")
  ✗ Requires constant location updates (battery drain)
  ✗ Network latency (5-second trigger delay)
  ✗ Requires cellular connectivity

Hybrid Approach ⭐:
  1. Server manages all 100M geofences
  2. For each device, server identifies the 20 nearest/most-relevant fences
  3. Push these 20 to the client's native geofencing API
  4. Client triggers instantly (offline-capable)
  5. Client also sends location updates to server for fences beyond the 20
  6. As device moves → server updates the "top 20" pushed to client
  
  Result: Best of both worlds
    - Top 20 fences: instant, battery-efficient, offline
    - Remaining fences: server-side evaluation with 5-second latency
```

### Point-in-Polygon: Ray Casting vs Winding Number

```
Ray Casting Algorithm:
  Cast a ray from point to infinity (usually along X axis)
  Count intersections with polygon edges
  Odd count → inside; Even → outside
  
  ✓ Simple, fast: O(V) per polygon
  ✓ Works for all simple polygons (convex and concave)
  ✗ Edge cases: point exactly on boundary, ray passing through vertex

Winding Number Algorithm:
  Count how many times the polygon winds around the point
  Non-zero winding number → inside
  
  ✓ Numerically more robust (handles boundary cases better)
  ✓ Works for self-intersecting polygons
  ✗ Slightly more complex implementation

For geofencing: Ray Casting is sufficient and faster.
  Geofences are typically simple polygons (4-8 vertices, convex or mildly concave).
  Edge case of exactly-on-boundary is handled by hysteresis buffer anyway.
```

### Battery Optimization: Adaptive Location Update Frequency

```
Constant 5-second GPS polling:
  Battery impact: ~8% per hour → phone dead in 12 hours
  Unacceptable for consumer apps

Adaptive strategy:
  1. Far from any geofence (> 5 km from nearest): 
     → Update every 5 minutes (cell tower location, ~500m accuracy)
     → Battery: negligible
  
  2. Approaching a geofence (1-5 km):
     → Update every 60 seconds (WiFi-assisted, ~50m accuracy)
     → Battery: ~1% per hour
  
  3. Near a geofence boundary (< 1 km):
     → Update every 10 seconds (GPS, ~10m accuracy)
     → Battery: ~3% per hour
  
  4. Inside a geofence (dwell tracking):
     → Update every 30 seconds (to confirm still inside)
     → Battery: ~2% per hour

How does the client know which mode to use?
  Server pushes "awareness regions" to client:
    "Your nearest geofence is at (lat, lng), 3.2 km away"
    Client uses distance to nearest fence → select polling frequency
    
  Update awareness regions on each location report to server.
```

### Geofence Evaluation: In-Line vs Async

```
In-Line (synchronous):
  Location update → immediately check geofences → return result in response
  
  ✓ Lowest latency (result in same request)
  ✗ Increases API response time (need spatial query in request path)
  ✗ Can't handle 5M updates/sec if each needs spatial query + state check

Async (Kafka + Flink) ⭐:
  Location update → write to Kafka → return 200 immediately
  Flink reads from Kafka → evaluates geofences → emits trigger events
  
  ✓ Decouples ingestion from evaluation (each scales independently)
  ✓ Handles 5M updates/sec (Kafka can absorb this easily)
  ✓ Backpressure: if evaluation is slow, Kafka buffers (no data loss)
  ✗ Additional latency (1-5 seconds from update to trigger)
  
  For most use cases (marketing, analytics, fleet management): 5-second delay is fine.
  For safety-critical (perimeter breach detection): use in-line path for high-priority fences.
```



