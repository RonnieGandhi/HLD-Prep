# 53. Design a Map Rendering and Navigation System like Google Maps

---

## 1. Functional Requirements (FR)

- **Map rendering**: Display interactive, zoomable, pannable maps with satellite/terrain/traffic layers
- **Geocoding**: Convert addresses to coordinates and vice versa (reverse geocoding)
- **Route planning**: Calculate optimal routes between origin and destination (driving, walking, cycling, transit)
- **Turn-by-turn navigation**: Real-time guided navigation with voice prompts, lane guidance
- **Live traffic**: Show real-time traffic conditions; reroute when congestion detected
- **Search places**: Search for POIs (restaurants, gas stations, hospitals) by name or category
- **ETA estimation**: Accurate estimated time of arrival considering traffic, road type, speed limits
- **Offline maps**: Download map regions for use without internet
- **Street View**: 360° panoramic imagery at street level
- **User contributions**: Report road closures, accidents, speed traps

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Map tiles render in < 200 ms; route calculations in < 1 second
- **High Availability**: 99.99% — navigation failure while driving is dangerous
- **Scalability**: 1B+ MAU, 50M+ active navigation sessions concurrently
- **Accuracy**: Routes must reflect real road conditions; ETA within ±10% of actual
- **Freshness**: Traffic data updated every 30–60 seconds
- **Bandwidth Efficiency**: Minimize data transfer for mobile users (vector tiles vs raster)
- **Global Coverage**: Support every country, including local address formats and driving rules
- **Offline Capable**: Core navigation must work without connectivity

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 300M |
| Map tile requests / sec | ~2M (multiple tiles per pan/zoom) |
| Route calculation requests / sec | 100K |
| Active navigation sessions | 50M concurrently |
| Traffic data points / sec | 10M (aggregated from active drivers) |
| Map data total size | ~20 TB (vector data, POIs, road graph) |
| Tile cache (all zoom levels) | ~500 TB (pre-rendered + vector) |
| Road graph (edges + nodes) | ~2B nodes, ~5B edges globally |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│  Mobile /    │
│  Web Client  │
└──────┬───────┘
       │
┌──────▼───────────────────────────────────┐
│              API Gateway / CDN           │
│  (Map tiles served from CDN edge)        │
└──────┬──────────────────┬────────────────┘
       │                  │
       │            ┌─────▼──────────┐
       │            │  Tile Service  │ ← Serves pre-rendered / dynamic map tiles
       │            │  (Vector/Raster│
       │            │   Tile Server) │
       │            └─────┬──────────┘
       │                  │
       │            ┌─────▼──────────┐
       │            │  Tile Storage  │ ← S3 + CDN caching
       │            │  (Pre-computed │
       │            │   tile pyramid)│
       │            └────────────────┘
       │
┌──────▼───────┐    ┌──────────────┐    ┌──────────────┐
│  Routing     │    │  Geocoding   │    │  Search /    │
│  Service     │    │  Service     │    │  POI Service │
│  (OSRM/     │    │  (Address ↔  │    │  (Elastic-   │
│  Valhalla)   │    │  Coordinates)│    │  search)     │
└──────┬───────┘    └──────────────┘    └──────────────┘
       │
┌──────▼───────┐    ┌──────────────┐    ┌──────────────┐
│  Navigation  │    │  Traffic     │    │  ETA         │
│  Service     │    │  Service     │    │  Service     │
│  (Turn-by-   │    │  (Real-time  │    │  (ML-based   │
│   turn)      │    │   aggregation│    │   prediction)│
└──────────────┘    └──────┬───────┘    └──────────────┘
                           │
                    ┌──────▼───────┐
                    │  Traffic     │
                    │  Data Store  │
                    │  (Time-series│
                    │   per road   │
                    │   segment)   │
                    └──────────────┘

Data Stores:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │   Kafka      │  │ Elasticsearch│
│  + PostGIS   │  │ (Tile cache, │  │ (Location    │  │ (POI search) │
│ (Road graph, │  │  traffic     │  │  telemetry,  │  │              │
│  POIs, geo)  │  │  segments)   │  │  traffic)    │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Map Tile System — How Maps Are Rendered

```
The world map is divided into a tile pyramid:

Zoom Level 0: Entire world = 1 tile (256×256 px)
Zoom Level 1: 4 tiles (2×2 grid)
Zoom Level 2: 16 tiles (4×4 grid)
...
Zoom Level 18: ~69 billion tiles (street level)
Zoom Level 22: Used for building-level detail in some areas

Each tile is addressed as: (zoom, x, y) → e.g., /tiles/14/8190/5456.png

Tile Addressing — Slippy Map Convention:
  URL: https://tiles.example.com/{z}/{x}/{y}.{format}
  z = zoom level (0–22)
  x = column (0 to 2^z − 1)
  y = row (0 to 2^z − 1)

Total tiles across all zoom levels ≈ Σ(4^z) for z=0..22 → ~5.6 trillion tiles
Obviously, NOT all pre-rendered. Strategy:

Pre-render:
  - Zoom 0–10: All tiles (~1.4M tiles) — global view
  - Zoom 11–14: Populated areas (~500M tiles) — city view
  - Zoom 15–18: Dense urban areas on-demand — street view

On-demand rendering:
  - Zoom 15+ for rural areas: render on request, cache with TTL
  - Avoids rendering billions of ocean/desert tiles nobody views
```

**Raster vs Vector Tiles — Why Google Maps Moved to Vector:**

```
Raster Tiles (traditional):
  - Pre-rendered PNG/JPEG images
  - Client just displays images (simple)
  - Each zoom level = separate set of images
  - Rotation: looks pixelated (image doesn't adapt)
  - Styling: requires re-rendering all tiles
  - Size: ~20-50 KB per tile
  
Vector Tiles (modern) ⭐:
  - Tile contains raw geometry data (roads, buildings, labels as vectors)
  - Client renders locally (using GPU / OpenGL / WebGL)
  - Rotation: smooth (vectors re-render at any angle)
  - Styling: client-side style sheets — change colors/themes without re-fetching
  - Size: ~5-15 KB per tile (compressed protobuf — smaller than raster!)
  - Labels: rendered client-side → proper orientation even when map rotated
  - 3D buildings: natural extension (extrude polygons)
  - Offline: store vector tiles + style → full offline map

Format: Mapbox Vector Tiles (MVT) — protobuf-encoded geometry
  Each tile contains layers: roads, buildings, water, land_use, POIs
  Client applies a style sheet (like CSS for maps) to render

Why Google moved to vector:
  1. 10× less bandwidth (critical for mobile)
  2. Smooth rotation and 3D (key for navigation)
  3. Dynamic styling (night mode, traffic overlay — no server re-render)
  4. Offline maps become feasible (vector data is compact)
```

#### Routing Engine — How Route Calculation Works

```
The road network is a weighted directed graph:
  - Nodes: intersections, road endpoints
  - Edges: road segments between nodes
  - Weights: travel time (distance / speed_limit × traffic_factor)

Scale: ~2 billion nodes, ~5 billion edges globally

Naive Dijkstra:
  O(E log V) → 5 billion × log(2 billion) ≈ 155 billion operations
  Minutes per query. Unacceptable.

Optimized Algorithms (what Google actually uses):

1. Contraction Hierarchies (CH) ⭐:
   - Pre-processing: Create "shortcut" edges that bypass intermediate nodes
   - Identify "unimportant" nodes (dead-end streets, residential)
   - Add shortcut edges between their neighbors
   - Result: hierarchical graph where highways are at top, local streets at bottom
   
   Query: Bidirectional Dijkstra on contracted graph
   - Forward search from origin (going UP the hierarchy)
   - Backward search from destination (going UP the hierarchy)
   - Meet in the middle at a highway-level node
   - Query time: < 1 ms for continental routes!
   
   Trade-off: 
   - Pre-processing: hours to days for full graph
   - Doesn't handle real-time traffic well (edge weights change)
   
2. Customizable Route Planning (CRP) — Microsoft's approach:
   - Partition road graph into cells
   - Pre-compute clique (boundary-to-boundary) distances within each cell
   - Live traffic: only re-compute affected cells (not entire graph)
   - Supports real-time weight updates → better for traffic-aware routing

3. A* with ALT (A*, Landmarks, Triangle inequality):
   - A* heuristic: pre-compute distances from ~20 "landmark" nodes
   - Use triangle inequality for admissible heuristic
   - 10-50× faster than plain Dijkstra
   - Supports dynamic edge weights (traffic)

Google's actual approach: Multi-modal + traffic-aware CRP variant
  - Pre-compute hierarchy for static road network
  - Overlay real-time traffic data on edge weights
  - Re-compute affected portions when traffic changes
  - Alternative routes: compute K-shortest paths, rank by different criteria
```

#### Traffic Service — Real-Time Traffic Aggregation

```
Data Sources:
  1. GPS traces from Google Maps users (phones reporting speed + location)
  2. Connected vehicles (OBD data)
  3. Historical patterns (rush hour, weekday vs weekend)
  4. Reported incidents (accidents, road closures, construction)

Processing Pipeline:
  Phone → sends (lat, lng, speed, heading, timestamp) every 5 seconds
  → Kafka topic: "traffic-telemetry"
  → Flink streaming job:
     a. Map-match GPS point to nearest road segment (snap to road)
     b. Aggregate speed per road segment per 60-second window
     c. Compare current speed to free-flow speed → congestion level
     d. Emit: (road_segment_id, avg_speed, congestion_level, timestamp)
  → Redis: traffic:{segment_id} → { speed, congestion, updated_at }
  → Tile Service: regenerate traffic overlay tiles every 60 seconds

Congestion Levels:
  Free flow: speed > 80% of speed_limit → GREEN
  Moderate:  speed 40-80% → YELLOW/ORANGE
  Heavy:     speed 10-40% → RED
  Stopped:   speed < 10% → DARK RED

Map Matching (GPS → Road Segment):
  GPS is noisy (±10m accuracy). Must "snap" to correct road.
  Algorithm: Hidden Markov Model (HMM)-based map matching
    States: candidate road segments near GPS point
    Transitions: probability of moving between segments
    Observations: distance from GPS point to road segment
    Viterbi algorithm: find most likely sequence of road segments
    
  Why HMM and not just "nearest road"?
    GPS point near a highway overpass could match to highway OR service road below
    HMM considers the sequence of points → picks the coherent path
```

#### Navigation Service — Turn-by-Turn Guidance

```
Once route is computed (list of road segments):

1. Generate maneuvers:
   For each junction along the route:
   - Determine turn type: left, right, slight_left, u_turn, merge, exit, roundabout
   - Calculate distance to next maneuver
   - Generate instruction: "In 200 meters, turn left onto Oak Street"
   - Include lane guidance: "Use left 2 lanes to turn left"

2. Real-time position tracking:
   Client sends GPS position every 1-2 seconds
   Server (or client-side) matches position to route:
   - On route: show current progress, distance/time to next maneuver
   - Off route: trigger re-routing (compute new route from current position)
   
3. Re-routing:
   If user deviates > 50m from route OR traffic changes significantly:
   - Compute new route from current position to destination
   - Must complete in < 2 seconds (latency-critical!)
   - Compare new route vs current remaining route → only reroute if saves > 5 minutes
   
4. Client-side vs Server-side navigation:
   Google Maps: Hybrid
   - Initial route computed server-side (better traffic data, more compute)
   - Maneuver progression tracked client-side (works offline)
   - Periodic server check for traffic updates (every 60 seconds)
   - Rerouting: try client-side first (offline graph), fallback to server
```

---

## 5. APIs

### Get Map Tiles
```http
GET /tiles/{z}/{x}/{y}.mvt
  z = zoom level, x = column, y = row
  Response: Binary protobuf (vector tile)
  Headers: Cache-Control: public, max-age=86400
  Served via CDN (95%+ cache hit rate at popular zoom levels)
```

### Geocode (Address → Coordinates)
```http
GET /api/v1/geocode?address=1600+Amphitheatre+Parkway+Mountain+View
Response: 200 OK
{
  "results": [
    {
      "formatted_address": "1600 Amphitheatre Pkwy, Mountain View, CA 94043",
      "geometry": {"lat": 37.4220, "lng": -122.0841},
      "place_id": "ChIJj61dQgK6j4AR4GeTYWZsKWw",
      "types": ["street_address"]
    }
  ]
}
```

### Get Route
```http
POST /api/v1/routes
{
  "origin": {"lat": 37.7749, "lng": -122.4194},
  "destination": {"lat": 37.3382, "lng": -121.8863},
  "mode": "driving",
  "departure_time": "2025-03-14T08:00:00Z",
  "alternatives": true,
  "avoid": ["tolls", "ferries"]
}
Response: 200 OK
{
  "routes": [
    {
      "route_id": "r-uuid",
      "summary": "US-101 S",
      "distance_meters": 72400,
      "duration_seconds": 3420,
      "duration_in_traffic_seconds": 4200,
      "polyline": "encoded_polyline_string",
      "legs": [...],
      "maneuvers": [
        {"type": "turn_left", "instruction": "Turn left onto Market St",
         "distance_meters": 800, "duration_seconds": 120,
         "start_location": {"lat": 37.7749, "lng": -122.4194},
         "lane_guidance": ["left", "left", "straight"]}
      ]
    }
  ]
}
```

### Start Navigation Session
```http
POST /api/v1/navigation/start
{
  "route_id": "r-uuid",
  "current_location": {"lat": 37.7749, "lng": -122.4194}
}
Response: 200 OK
{
  "session_id": "nav-uuid",
  "websocket_url": "wss://nav.example.com/session/nav-uuid"
}
```

### Search POIs
```http
GET /api/v1/places/search?q=gas+station&lat=37.77&lng=-122.41&radius=5000
Response: 200 OK
{
  "results": [
    {"place_id": "p-uuid", "name": "Shell Gas Station", "lat": 37.78, "lng": -122.40,
     "rating": 4.2, "distance_meters": 850, "open_now": true}
  ]
}
```

---

## 6. Data Model

### PostgreSQL + PostGIS — Road Network Graph

```sql
-- Road segments (edges of the graph)
CREATE TABLE road_segments (
    segment_id      BIGINT PRIMARY KEY,
    start_node_id   BIGINT NOT NULL,
    end_node_id     BIGINT NOT NULL,
    road_name       TEXT,
    road_class      ENUM('motorway','trunk','primary','secondary',
                         'tertiary','residential','service'),
    length_meters   FLOAT,
    speed_limit_kmh INT,
    one_way         BOOLEAN DEFAULT FALSE,
    toll            BOOLEAN DEFAULT FALSE,
    geometry        GEOMETRY(LineString, 4326),
    country_code    CHAR(2),
    updated_at      TIMESTAMP,
    INDEX idx_start_node (start_node_id),
    INDEX idx_end_node (end_node_id),
    SPATIAL INDEX idx_geometry (geometry)
);

-- Nodes (intersections)
CREATE TABLE road_nodes (
    node_id         BIGINT PRIMARY KEY,
    lat             DECIMAL(10,7),
    lng             DECIMAL(10,7),
    geometry        GEOMETRY(Point, 4326),
    SPATIAL INDEX idx_geometry (geometry)
);

-- Places of Interest
CREATE TABLE places (
    place_id        VARCHAR(64) PRIMARY KEY,
    name            TEXT NOT NULL,
    category        VARCHAR(50),
    lat             DECIMAL(10,7),
    lng             DECIMAL(10,7),
    address         TEXT,
    phone           VARCHAR(20),
    rating          DECIMAL(2,1),
    review_count    INT,
    hours           JSONB,
    geometry        GEOMETRY(Point, 4326),
    SPATIAL INDEX idx_geometry (geometry)
);
```

**Why PostGIS?**
- Native spatial indexing (R-tree) for "find all roads within bounding box"
- Spatial functions: ST_DWithin, ST_Intersects, ST_Distance for geospatial queries
- Road graph queries benefit from relational model (JOIN start/end nodes)
- Industry standard for geographic data

### Redis — Traffic & Tile Cache

```
# Real-time traffic per road segment
traffic:{segment_id}  → Hash { avg_speed, congestion_level, sample_count, updated_at }
TTL: 120 seconds (stale if not refreshed)

# Hot tile cache
tile:{z}:{x}:{y}     → Binary (protobuf vector tile)
TTL: 3600 (zoom 0-10), 300 (zoom 11-14), 60 (zoom 15+, traffic changes)

# Geocoding cache (popular addresses)
geocode:{address_hash} → JSON { lat, lng, formatted_address }
TTL: 86400
```

### Routing Engine — In-Memory Graph (Contraction Hierarchies)

```
The routing engine loads the entire road graph into memory:

Memory per node: 16 bytes (id, lat, lng, level)
Memory per edge: 32 bytes (from, to, weight, flags, shortcut_target)
2B nodes × 16B = 32 GB
5B edges × 32B = 160 GB
Contraction shortcuts: ~3B additional edges × 32B = 96 GB
Total: ~288 GB → fits in a high-memory server (512 GB RAM)

Sharding by region:
  Partition world into ~20 regions (continents/sub-continents)
  Each region: 50-150 GB in-memory graph
  Cross-region queries: handled by stitching at border nodes
  
Replication: 3 replicas per region for availability + load balancing
```

### Kafka Topics

```
Topic: traffic-telemetry     (GPS traces from users, partitioned by geohash region)
Topic: traffic-aggregated    (processed road segment speeds, consumed by tile service)
Topic: route-requests        (async route calculation for batch/analytics)
Topic: incident-reports      (user-reported road events)
```

### Elasticsearch — POI Search

```
Index: places
{
  "name": "Starbucks",
  "category": "cafe",
  "location": { "lat": 37.7749, "lon": -122.4194 },  // geo_point
  "address": "123 Market St, San Francisco, CA",
  "rating": 4.3,
  "open_hours": {...}
}

Queries:
  - Text search + geo filter: "starbucks within 5km of (lat, lng)"
  - Category filter + geo sort: "restaurants sorted by distance"
  - Autocomplete: prefix query on name field
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Tile server failure** | CDN absorbs 95%+ of tile requests; origin behind ALB with auto-scaling |
| **Routing engine crash** | Stateless; multiple replicas per region; request retried on another replica |
| **Traffic data lag** | Fall back to historical patterns (same day/time last week) |
| **GPS signal loss** | Client-side dead reckoning (speed + heading from last known position) |
| **Off-route detection** | Client matches position to route locally; reroute even when offline |
| **CDN cache miss storm** | Origin shield (intermediate cache layer) prevents thundering herd |
| **Map data corruption** | Version-controlled map data; rollback to last known good version |
| **Cross-region routing** | Border nodes replicated on both region shards; stitching layer |

### Specific: Navigation During Network Outage

```
Scenario: User is navigating, enters a tunnel (no signal for 5 minutes)

1. Client has full route pre-loaded (maneuver list + polyline)
2. Client continues GPS tracking (GPS works without cellular)
3. Client-side position matching to polyline continues working
4. Turn-by-turn instructions continue (pre-computed)
5. No traffic updates → user follows original route (acceptable)
6. When signal returns:
   a. Client sends buffered GPS traces to server
   b. Server checks if traffic conditions changed significantly
   c. If better route exists → push reroute suggestion
   d. If not → continue current route

Key design decision: Navigation must be primarily client-side
  Server is for: initial route, traffic updates, rerouting suggestions
  Client is for: position tracking, maneuver triggering, voice guidance
```

---

## 8. Deep Dive: Engineering Trade-offs

### Tile Rendering: Pre-Render All vs On-Demand vs Hybrid

```
Pre-render all tiles:
  Zoom 0-18 for the whole world ≈ 70 billion tiles
  At 10 KB/tile = 700 TB of storage
  Rendering time: weeks on a cluster
  ✓ Instant response (no compute at request time)
  ✗ Massive storage; most tiles never requested (ocean, desert)
  ✗ Any map data update → re-render affected tiles

On-demand rendering:
  Render tile when first requested, cache result
  ✓ Only renders tiles people actually view
  ✓ Always up-to-date
  ✗ First request for a tile is slow (100-500 ms to render)
  ✗ Cold start after cache flush is painful

Hybrid approach ⭐ (Google/Mapbox):
  Pre-render: Zoom 0-12 globally (~16M tiles) — takes hours
  Pre-render: Zoom 13-18 for major cities (~1B tiles)
  On-demand: Everything else, cached with TTL
  
  Cache hierarchy:
    Client-side cache → CDN edge → CDN origin shield → Tile server
    
  Invalidation: Map data update → invalidate affected tiles in CDN
    Tile server stores: tile → map_data_version
    If data version changes → tile regenerated on next request
```

### Map Matching: Nearest Road vs HMM — When "Closest" Is Wrong

```
Problem: GPS says user is at point P. Which road are they on?

Scenario: Highway overpass
  Highway segment (elevated): 10m away from GPS point
  Service road (ground level): 8m away from GPS point
  
  Nearest road algorithm: picks service road (wrong!)
  HMM algorithm: considers last 10 GPS points → user was traveling at 
    100 km/h → clearly on highway, not service road

HMM Map Matching:
  States: candidate road segments within 50m of each GPS point
  Emission probability: P(GPS_point | road_segment) ~ exp(-distance²)
  Transition probability: P(segment_i → segment_j) ~ exp(-route_distance)
  Viterbi algorithm: find most likely sequence of segments
  
  Handles:
  - Parallel roads (highway + service road)
  - GPS drift in urban canyons (tall buildings reflect GPS signals)
  - Tunnel entry/exit (GPS gaps)
  - U-turns and loops

Computational cost:
  Per GPS point: ~10 candidate segments × 10 previous candidates = 100 transitions
  For 60 points (1 minute at 1 Hz): 6,000 computations → < 1 ms
```

### Offline Maps: What Gets Downloaded?

```
User downloads "San Francisco" for offline use:

Vector tiles: Zoom 0-16 for the SF bounding box
  ~ 50K tiles × 10 KB avg = 500 MB
  
Road graph: All road segments in the bounding box
  ~ 200K segments × 200 bytes = 40 MB
  
POIs: All indexed places in the area
  ~ 100K places × 500 bytes = 50 MB
  
Routing graph (Contraction Hierarchies):
  Pre-computed CH for the region = 100 MB
  
Style sheet: Map rendering rules = 2 MB
  
Total: ~700 MB per major city

Offline capabilities:
  ✓ Map viewing (pan, zoom, rotate)
  ✓ Route calculation (using local CH graph)
  ✓ POI search (local index)
  ✓ Turn-by-turn navigation
  ✗ Live traffic (requires network)
  ✗ Real-time rerouting for traffic (but route deviation rerouting works)
  
Update strategy:
  Delta updates: download only changed tiles/segments since last sync
  Typical weekly delta: 10-50 MB (vs 700 MB full download)
```

### Why Separate Routing Engine From PostGIS (Not Just pgRouting)?

```
pgRouting (PostGIS extension):
  - Runs Dijkstra/A* directly on PostgreSQL road_segments table
  - Query: SELECT * FROM pgr_dijkstra('SELECT ...', start_node, end_node)
  ✓ Simple to set up
  ✗ Each query reads from disk (even with caching, it's I/O bound)
  ✗ No contraction hierarchies → continental routes take seconds
  ✗ Can't handle 100K route requests/sec

Dedicated in-memory routing engine (OSRM, Valhalla, GraphHopper):
  - Load entire graph into RAM (288 GB for global)
  - Pre-compute contraction hierarchies
  - Route query: < 1 ms for continental, < 10 ms with traffic
  ✓ 100-1000× faster than pgRouting
  ✓ Supports CH, CRP, ALT algorithms
  ✗ High memory requirements (512 GB RAM servers)
  ✗ Graph updates require re-loading (10-30 min for a region)

Decision: Use dedicated routing engine for real-time queries
  PostGIS for: map data storage, spatial queries, tile generation
  Routing engine for: route calculation, ETA estimation
  
  Separation of concerns: 
    PostGIS = source of truth for road network
    Routing engine = optimized read-only copy for fast queries
    Pipeline: PostGIS → export graph → build CH → load into routing engine
    Frequency: re-build every 6 hours (or on significant map data changes)
```

### Polyline Encoding: Reducing Route Data Size

```
A route from SF to LA: ~600 km, ~5,000 coordinate pairs

Raw JSON:
  [{"lat": 37.774900, "lng": -122.419400}, {"lat": 37.774950, ...}, ...]
  5,000 × 40 bytes = 200 KB

Google's Encoded Polyline Algorithm:
  - Encode deltas between consecutive points
  - Most deltas are tiny (0.00005°) → encode in 1-2 bytes
  - Base64-like encoding
  - Result: ~15 KB for same route (13× compression)

Example:
  Original: [(38.5, -120.2), (40.7, -120.95), (43.252, -126.453)]
  Encoded: "_p~iF~ps|U_ulLnnqC_mqNvxq`@"

Why this matters:
  - Route response over cellular: 15 KB vs 200 KB
  - Cached routes in memory: 13× more routes per GB
  - Client rendering: decode is O(n), fast
```



