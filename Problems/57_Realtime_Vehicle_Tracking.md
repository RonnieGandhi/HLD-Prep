# 57. Design a Real-time Vehicle Tracking System

---

## 1. Functional Requirements (FR)

- **Live tracking**: Display real-time location of vehicles on a map (fleet management, delivery tracking)
- **Location ingestion**: Ingest GPS coordinates from thousands/millions of vehicles at configurable intervals
- **Trip tracking**: Track active trips with start, waypoints, and end; record full route trail
- **Geofence alerts**: Trigger alerts when vehicles enter/exit defined zones
- **Historical playback**: Replay a vehicle's route over any past time period
- **Speed/idle alerts**: Detect speeding, excessive idling, harsh braking events
- **Fleet dashboard**: Real-time overview of entire fleet — active, idle, offline vehicles
- **ETA for deliveries**: Show customer the live position and ETA of their delivery
- **Multi-tenant**: Support multiple fleet operators on the same platform

---

## 2. Non-Functional Requirements (NFRs)

- **Real-time**: Location updates visible on dashboard within 3 seconds of device report
- **High Throughput**: Handle 5M+ location updates/sec across all fleets
- **Scalability**: Support 10M+ simultaneously tracked vehicles
- **Durability**: No location data point lost (required for compliance, insurance, disputes)
- **Low Latency**: Map updates in < 3 seconds; dashboard aggregations in < 500 ms
- **Availability**: 99.99%
- **Data Retention**: Raw location trails retained for 90 days; aggregated data for 2+ years
- **Bandwidth Efficiency**: Minimize cellular data usage for vehicle trackers

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active vehicles | 10M |
| Avg location update interval | 5 seconds |
| Location updates / sec | 2M |
| Location point size | 100 bytes |
| Raw data / day | 2M × 100B × 86400 = ~17 TB |
| Concurrent dashboard viewers | 500K |
| WebSocket connections | 500K (dashboard) + 10M (vehicle connections) |
| Historical queries / sec | 10K |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Vehicle GPS │    │  OBD-II      │    │  Mobile      │
│  Tracker     │    │  Device      │    │  Driver App  │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                    │                    │
       │  MQTT/TCP          │  MQTT              │  WebSocket
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼───────────────────┐
                     │   Connection Gateway      │
                     │   (MQTT Broker / WS       │
                     │    Server Cluster)         │
                     │   - Protocol translation  │
                     │   - Auth + device registry│
                     └──────┬───────────────────┘
                            │
                     ┌──────▼───────┐
                     │    Kafka     │ ← Core message bus
                     │  (location   │   Partitioned by vehicle_id
                     │   updates)   │
                     └──────┬───────┘
                            │
          ┌─────────────────┼─────────────────────────┐
          │                 │                          │
   ┌──────▼──────┐   ┌─────▼──────┐   ┌──────────────▼───────┐
   │ Location    │   │ Alert      │   │ Trail Writer         │
   │ Processor   │   │ Engine     │   │ (Location History)   │
   │ (Flink)     │   │ (Flink)    │   │                      │
   │             │   │            │   │ Batch write to       │
   │ Updates     │   │ Speed,     │   │ TimescaleDB /        │
   │ Redis with  │   │ geofence,  │   │ Cassandra            │
   │ latest loc  │   │ idle       │   │                      │
   └──────┬──────┘   └─────┬──────┘   └──────────────────────┘
          │                 │
   ┌──────▼──────┐   ┌─────▼──────┐
   │   Redis     │   │ Notification│
   │ (Latest     │   │ Service    │
   │  vehicle    │   │ (Push,     │
   │  positions) │   │  Webhook)  │
   └──────┬──────┘   └────────────┘
          │
   ┌──────▼──────────────────────────┐
   │   Dashboard WebSocket Server    │
   │   (Push latest positions to     │
   │    connected dashboard clients) │
   └─────────────────────────────────┘
```

### Component Deep Dive

#### Connection Gateway — Handling 10M Persistent Connections

```
Vehicle devices connect via MQTT (lightweight IoT protocol) or raw TCP.
Dashboard users connect via WebSocket.

MQTT vs HTTP for vehicle GPS:
  HTTP (polling or POST):
    Each update = new TCP connection + TLS handshake
    Per vehicle: 4 KB overhead × 12 updates/min = 48 KB/min wasted
    For 10M vehicles: 480 TB/day wasted bandwidth
    
  MQTT ⭐:
    Persistent connection with QoS levels
    Per update: ~20 bytes header + 100 bytes payload = 120 bytes
    For 10M vehicles: 120B × 12/min × 10M = 86 GB/day (5500× less!)
    QoS 1: at-least-once delivery (no data loss)
    Keep-alive: heartbeat every 60 seconds
    
  Why MQTT is perfect for vehicle tracking:
    1. Minimal bandwidth (cellular data is expensive per vehicle)
    2. Built-in reconnection logic (vehicles enter tunnels, rural areas)
    3. QoS guarantees (critical for compliance — every point must arrive)
    4. Last Will Testament: broker detects vehicle offline → alert fleet manager

MQTT Broker Cluster:
  Technology: EMQX or VerneMQ (horizontally scalable MQTT brokers)
  Each broker handles ~200K connections
  10M vehicles → 50 broker instances
  
  Clustering:
    Shared subscription: distribute incoming messages across Kafka producers
    Session persistence: vehicle reconnects to any broker, resumes from last ACK
    
  Bridge to Kafka:
    MQTT broker → Kafka producer (bridge)
    Each location update published to Kafka topic: "vehicle-locations"
    Key: vehicle_id → ensures ordering per vehicle
```

#### Location Processor — Updating Latest Position in Redis

```
Flink streaming job consuming from Kafka "vehicle-locations":

For each location update:
  1. Validate: reasonable lat/lng range, timestamp not in future, speed < 300 km/h
  2. De-duplicate: if same timestamp as last update → skip (at-least-once produces dupes)
  3. Enrich: attach fleet_id, vehicle_type from device registry (cached in Flink state)
  4. Update Redis:
     HSET vehicle:{vehicle_id} lat {lat} lng {lng} speed {speed} heading {heading} 
                                ts {timestamp} status {moving|idle|parked}
     EXPIRE vehicle:{vehicle_id} 300  // 5-min TTL — offline detection
  5. Publish to Redis Pub/Sub channel: "vehicle_updates:{fleet_id}"
     → Dashboard WebSocket server subscribes to fleet channels
     → Pushes updates to connected dashboard clients

Movement Status Detection:
  speed > 5 km/h → "moving"
  speed < 5 km/h for < 5 min → "idle"  (temporary stop: traffic light, loading)
  speed < 2 km/h for > 5 min → "parked"
  No update for > 5 min → "offline" (TTL expiry in Redis)

Optimization — Location Compression:
  Don't store every update if vehicle hasn't moved significantly:
  
  Douglas-Peucker algorithm for trail compression:
    If new point is < 10m from straight line between last two stored points → skip
    Reduces storage by 50-70% for highway driving (straight roads)
    Preserves turns and stops (important waypoints)
    
  But: for real-time display, always update Redis (even if not persisting to DB)
```

#### Trail Writer — Persisting Location History

```
Storage requirements:
  - 2M updates/sec → 17 TB/day raw
  - Must query by: (vehicle_id, time_range) for route playback
  - Must retain 90 days → 1.5 PB total
  
Storage choice: TimescaleDB (PostgreSQL extension for time-series) ⭐

Why TimescaleDB over alternatives:
  
  TimescaleDB:
    ✓ Time-based partitioning (automatic chunking by time)
    ✓ Compression: 10-20× on time-series data → 1.5 PB → 100 TB after compression
    ✓ SQL queries (familiar, powerful: JOINs, window functions)
    ✓ Continuous aggregates (pre-compute hourly summaries)
    ✓ Native PostGIS integration (spatial queries on trails)
    ✗ Write throughput: ~500K inserts/sec per node (need sharding for 2M/sec)
  
  Cassandra:
    ✓ Linear write scalability (easy to handle 2M/sec)
    ✓ Partition by vehicle_id, cluster by timestamp → fast range queries
    ✗ No spatial queries
    ✗ No JOINs (harder for analytics)
    ✗ Less compression on irregular data patterns
  
  ClickHouse:
    ✓ Excellent compression (10-30×)
    ✓ Blazing fast analytics queries
    ✗ Not optimized for point queries (single vehicle trail)
    ✗ Inserts should be batched (not individual row inserts)

Decision: 
  Primary (hot, 90 days): TimescaleDB (sharded by vehicle_id hash, 4 nodes)
  Analytics (cold, 2+ years): ClickHouse (aggregated hourly/daily summaries)
  
Batch writing:
  Flink accumulates updates → batch write to TimescaleDB every 5 seconds
  Each batch: 2M/sec × 5s = 10M rows → 4 shards = 2.5M rows per shard
  TimescaleDB handles 2.5M bulk insert in < 1 second per shard
```

#### Dashboard — Real-Time Fleet View

```
Fleet manager opens dashboard → sees all 5,000 vehicles on a map

Connection flow:
  1. Dashboard client authenticates → get fleet_id
  2. WebSocket connection to Dashboard WS Server
  3. Server subscribes to Redis Pub/Sub channel: "vehicle_updates:{fleet_id}"
  4. Initial load: SCAN Redis for all vehicle:{vehicle_id} where fleet_id matches
     → send all current positions to client
  5. Ongoing: Redis Pub/Sub pushes updates → server forwards to client via WebSocket

Scaling WebSocket servers:
  Each server: handles ~50K WS connections
  500K dashboard users → 10 WS servers
  Sticky sessions: user connects to one server for duration
  
  If server goes down:
    Client reconnects to another server (via load balancer)
    New server re-subscribes to Redis Pub/Sub for the client's fleet
    Client requests full state refresh → server sends all current positions
    Seamless recovery in < 5 seconds

Map tile optimization:
  Dashboard doesn't need updates for vehicles outside the current viewport
  Client sends viewport bounds to server:
    { "viewport": { "ne_lat": 37.82, "ne_lng": -122.35, "sw_lat": 37.70, "sw_lng": -122.52 } }
  Server filters: only send updates for vehicles within viewport
  Result: 5000-vehicle fleet, viewport shows 200 → 96% bandwidth reduction

Clustering at low zoom:
  Zoom level 8 (city view): 5000 vehicles → show 50 clusters with counts
  Client-side clustering (Supercluster library) or server-side pre-aggregation
  As user zooms in → show individual vehicle icons
```

---

## 5. APIs

### Send Location Update (Vehicle → Server)
```
MQTT Topic: vehicles/{vehicle_id}/location
QoS: 1 (at-least-once)
Payload (protobuf, compressed):
{
  "vehicle_id": "v-uuid",
  "lat": 37.7749,
  "lng": -122.4194,
  "altitude": 15.2,
  "speed": 45.3,
  "heading": 180,
  "accuracy": 5,
  "timestamp": 1710400000,
  "odometer": 45230.5,
  "fuel_level": 0.65,
  "engine_status": "on",
  "events": ["harsh_brake"]
}
```

### Get Vehicle Current Location
```http
GET /api/v1/vehicles/{vehicle_id}/location
Response: 200 OK
{
  "vehicle_id": "v-uuid",
  "lat": 37.7749, "lng": -122.4194,
  "speed": 45.3, "heading": 180,
  "status": "moving",
  "last_updated": "2025-03-14T10:23:45Z",
  "driver": {"name": "John", "phone": "+1..."}
}
```

### Get Vehicle Route History
```http
GET /api/v1/vehicles/{vehicle_id}/trail?start=2025-03-14T08:00:00Z&end=2025-03-14T18:00:00Z&simplify=true
Response: 200 OK
{
  "vehicle_id": "v-uuid",
  "trail": [
    {"lat": 37.7749, "lng": -122.4194, "speed": 0, "ts": "2025-03-14T08:00:00Z"},
    {"lat": 37.7760, "lng": -122.4180, "speed": 35, "ts": "2025-03-14T08:05:00Z"},
    ...
  ],
  "total_distance_km": 145.3,
  "total_duration_hours": 8.5,
  "stops": [
    {"lat": 37.78, "lng": -122.41, "arrived": "10:30", "departed": "11:15", "duration_min": 45}
  ]
}
```

### Fleet Overview
```http
GET /api/v1/fleets/{fleet_id}/overview
Response: 200 OK
{
  "fleet_id": "f-uuid",
  "total_vehicles": 5000,
  "moving": 3200,
  "idle": 800,
  "parked": 700,
  "offline": 300,
  "alerts_active": 12,
  "vehicles_speeding": 5
}
```

### Subscribe to Fleet Updates (WebSocket)
```
WS: wss://tracking.example.com/ws/fleet/{fleet_id}
→ Server sends: { "type": "location_update", "vehicle_id": "v-uuid", "lat": ..., "lng": ..., "speed": ... }
→ Server sends: { "type": "alert", "vehicle_id": "v-uuid", "alert_type": "speeding", "speed": 120, "limit": 80 }
→ Server sends: { "type": "status_change", "vehicle_id": "v-uuid", "old": "moving", "new": "idle" }
```

---

## 6. Data Model

### Redis — Latest Vehicle State

```
# Per-vehicle latest position
vehicle:{vehicle_id}  → Hash {
  lat, lng, speed, heading, altitude, accuracy,
  status (moving|idle|parked),
  fleet_id, driver_id, vehicle_type,
  fuel_level, odometer,
  last_updated
}
TTL: 300 (offline detection)

# Fleet vehicle set (for fleet queries)
fleet_vehicles:{fleet_id}  → SET of vehicle_ids

# Fleet stats (atomic counters)
fleet_stats:{fleet_id}:moving  → INT
fleet_stats:{fleet_id}:idle    → INT
fleet_stats:{fleet_id}:parked  → INT
fleet_stats:{fleet_id}:offline → INT

# Active alerts per vehicle
alerts:{vehicle_id}  → LIST of active alert JSONs
```

### TimescaleDB — Location Trail (Hot Storage, 90 Days)

```sql
CREATE TABLE location_points (
    vehicle_id      UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    lat             DOUBLE PRECISION,
    lng             DOUBLE PRECISION,
    speed           REAL,
    heading         SMALLINT,
    altitude        REAL,
    accuracy        REAL,
    status          TEXT,
    odometer        REAL,
    fuel_level      REAL,
    events          JSONB
);

SELECT create_hypertable('location_points', 'timestamp', 
    chunk_time_interval => INTERVAL '1 day',
    partitioning_column => 'vehicle_id',
    number_partitions => 4);

-- Enable compression (10-20× ratio)
ALTER TABLE location_points SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'vehicle_id',
    timescaledb.compress_orderby = 'timestamp DESC'
);

-- Auto-compress chunks older than 7 days
SELECT add_compression_policy('location_points', INTERVAL '7 days');

-- Auto-drop chunks older than 90 days
SELECT add_retention_policy('location_points', INTERVAL '90 days');

-- Continuous aggregates: hourly summary per vehicle
CREATE MATERIALIZED VIEW vehicle_hourly_summary
WITH (timescaledb.continuous) AS
SELECT vehicle_id,
       time_bucket('1 hour', timestamp) AS hour,
       avg(speed) as avg_speed,
       max(speed) as max_speed,
       count(*) as point_count,
       ST_Length(ST_MakeLine(ST_MakePoint(lng, lat) ORDER BY timestamp)::geography) / 1000 as distance_km
FROM location_points
GROUP BY vehicle_id, time_bucket('1 hour', timestamp);
```

### Kafka Topics

```
Topic: vehicle-locations
  Key: vehicle_id
  Value: protobuf { vehicle_id, lat, lng, speed, heading, timestamp, events[] }
  Partitions: 512 (for 2M/sec throughput)
  Retention: 48 hours
  Compaction: disabled (need every point)

Topic: vehicle-alerts
  Key: vehicle_id
  Value: { vehicle_id, alert_type, details, timestamp }
  Partitions: 64
  Retention: 7 days

Topic: vehicle-status-changes
  Key: vehicle_id
  Value: { vehicle_id, old_status, new_status, timestamp }
  Partitions: 64
```

### ClickHouse — Cold Analytics (2+ Years)

```sql
CREATE TABLE vehicle_daily_summary (
    vehicle_id      UUID,
    date            Date,
    fleet_id        UUID,
    total_distance_km  Float32,
    total_drive_hours  Float32,
    total_idle_hours   Float32,
    max_speed_kmh      Float32,
    avg_speed_kmh      Float32,
    fuel_consumed      Float32,
    alerts_count       UInt16,
    stops_count        UInt16,
    geofence_events    UInt16
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (fleet_id, vehicle_id, date);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **MQTT broker failure** | Cluster with session handoff; vehicles reconnect to another broker; QoS 1 ensures no message loss |
| **Kafka lag** | Consumer group rebalancing; if lag > 30 sec, alert ops; auto-scale Flink parallelism |
| **Redis failure** | Redis Cluster (6+ nodes); AOF persistence; if down, dashboard shows "updating..." |
| **TimescaleDB write failure** | Kafka retains data (48h); replay failed writes after DB recovery |
| **Vehicle goes offline** | TTL-based detection in Redis; alert fleet manager; last known position preserved |
| **WebSocket server crash** | Client reconnect to new server; full state refresh from Redis |
| **GPS drift/spoofing** | Validate: speed vs distance between points; reject impossible movements |
| **Data center outage** | Multi-region Kafka replication; vehicles buffer locally during outage |

### Specific: Handling Vehicle GPS Blackout (Tunnel, Rural Area)

```
Scenario: Delivery truck enters a tunnel. No GPS for 10 minutes.

Vehicle-side:
  1. GPS device detects signal loss
  2. Continues recording timestamps (no coordinates)
  3. When GPS recovers, device has buffered timestamps with gaps
  4. Device sends batch of stored points + gap indicator to server

Server-side:
  1. Detect gap: no updates from vehicle for 5+ minutes
  2. Mark vehicle status = "GPS_LOST" (not offline — MQTT connection may still be alive)
  3. Dashboard shows: vehicle icon with "?" and last known position
  4. When GPS resumes:
     a. Receive batch of new points
     b. Interpolate the gap (if start and end points known):
        - Straight-line interpolation (simple)
        - Or: map-matched interpolation using road network (better for compliance)
     c. Store interpolated points with "estimated" flag
     d. Update trail for historical playback

Edge case: Vehicle GPS dies permanently
  OBD-II fallback: use cellular signal + cell tower triangulation
  Accuracy: ~200m (much worse than GPS ~5m, but shows general area)
  Alert fleet manager: "Vehicle V-123 GPS failure — needs maintenance"
```

### Specific: Out-of-Order GPS Points

```
Problem: Points arrive out of order due to network latency, retransmissions

  Point A (ts=100) → arrives at server at T=102
  Point B (ts=105) → arrives at server at T=106
  Point C (ts=103) → arrives at server at T=110 (delayed!)

For Redis (latest position):
  Only update if incoming timestamp > stored timestamp
  EVALSHA script:
    local current_ts = redis.call('HGET', key, 'ts')
    if not current_ts or tonumber(incoming_ts) > tonumber(current_ts) then
      redis.call('HMSET', key, ...)
    end
  
  Ensures Redis always shows the LATEST position, not the most recently arrived

For TimescaleDB (trail):
  Insert all points regardless of order
  Query with ORDER BY timestamp → returns correct chronological order
  Idempotency: ON CONFLICT (vehicle_id, timestamp) DO NOTHING → prevents duplicates

For speed alerts:
  Must compare consecutive points in timestamp order, not arrival order
  Flink: use event-time processing with watermarks
    Watermark = "all events with timestamp < T have arrived"
    Process speed calculation only after watermark advances past the point
    Late arrivals (after watermark) → still insert to DB, but don't re-trigger alerts
```

---

## 8. Deep Dive: Engineering Trade-offs

### MQTT vs WebSocket vs gRPC for Vehicle Communication

```
MQTT ⭐ (recommended):
  Protocol: Pub/sub over TCP
  Overhead: 2-4 bytes per message header (minimal!)
  QoS levels: 0 (fire-forget), 1 (at-least-once), 2 (exactly-once)
  Features: Last Will (offline detection), retained messages, session persistence
  Best for: IoT devices with constrained bandwidth/power
  Ecosystem: Mature IoT ecosystem, hardware GPS tracker support

WebSocket:
  Protocol: Bidirectional over HTTP upgrade
  Overhead: 2-14 bytes per frame
  QoS: None built-in (must implement retries in application)
  Best for: Browser-based clients, apps that need bidirectional communication
  Use in our system: Dashboard connections (browser-based)

gRPC:
  Protocol: HTTP/2 based, protobuf serialization
  Overhead: HTTP/2 framing + protobuf (efficient but more than MQTT)
  Features: Bidirectional streaming, strong typing, load balancing
  Best for: Service-to-service communication; mobile apps with gRPC support
  
Decision:
  Vehicles → Server: MQTT (minimal bandwidth, QoS, offline detection)
  Dashboard → Server: WebSocket (browser native, bidirectional)
  Internal services: gRPC (type safety, streaming)
```

### Location Storage: Raw Points vs Compressed Trails

```
Raw points:
  Store every GPS update as individual row
  100 bytes × 12 points/min × 1440 min/day × 10M vehicles = 17 TB/day
  Query: SELECT * WHERE vehicle_id = ? AND ts BETWEEN ? AND ?
  ✓ Maximum fidelity
  ✗ Massive storage

Compressed trail approaches:

1. Douglas-Peucker simplification:
   Remove points that don't add significant shape to the route
   Highway driving (straight): 70% reduction
   City driving (many turns): 30% reduction
   Average: 50% reduction → 8.5 TB/day
   
2. Delta encoding:
   Store first point fully: (lat=37.7749, lng=-122.4194)
   Subsequent: delta from previous (+0.0002, -0.0001)
   Deltas are tiny → compress well with LZ4/ZSTD
   10-15× compression on top of TimescaleDB native compression
   
3. Segment-based storage:
   Instead of individual points, store route segments:
   { vehicle_id, start_ts, end_ts, polyline: "encoded_polyline", avg_speed, distance }
   One row per 5-minute segment instead of 60 individual points
   12× fewer rows → faster queries

Hybrid approach ⭐:
  - Real-time (last 24h): raw points in TimescaleDB (need full fidelity for alerts)
  - Recent (1-90 days): compressed with TimescaleDB native compression (10× reduction)
  - Cold (90+ days): aggregated segments in ClickHouse (100× reduction from raw)
  
  Total storage:
    Hot: 17 TB × 1 day = 17 TB
    Warm: 17 TB × 89 days × 0.1 (compressed) = 151 TB
    Cold: daily summaries, negligible
    Total: ~170 TB (vs 1.5 PB uncompressed)
```

### Scalability: 10M Vehicles at 5-Second Intervals

```
2M location updates/sec is serious throughput. How to scale each component:

1. MQTT Broker Cluster:
   Each EMQX node: 200K connections, 100K msgs/sec
   10M connections: 50 nodes
   2M msgs/sec: 20+ nodes (connection and throughput have different scaling)
   → 50 MQTT broker nodes

2. Kafka:
   2M msgs/sec × 120 bytes = 240 MB/sec
   Each Kafka broker: 200 MB/sec write throughput
   512 partitions across 6 brokers → handles 2M msgs/sec easily
   Replication factor 3 → 18 brokers total

3. Flink (Location Processor):
   Each task slot: 10K msgs/sec (with Redis writes)
   2M / 10K = 200 task slots → ~25 TaskManagers (8 slots each)
   
4. Redis:
   2M HSET/sec → each Redis shard handles 100K writes/sec
   20 Redis shards (Redis Cluster)
   10M vehicle keys × 500 bytes = 5 GB memory (easily fits in 20 shards)

5. TimescaleDB:
   2M inserts/sec → batch into 5-second windows → 10M rows per batch
   4 shards × 2.5M rows per batch → each shard handles 500K inserts/5sec
   With prepared statements and COPY protocol: achievable
   
6. Dashboard WebSocket Servers:
   500K connections → 10 servers (50K connections each)
   Each server subscribes to Redis Pub/Sub for relevant fleets
   Fan-out: each vehicle update → push to ~3 dashboard viewers (avg)
   = 6M pushes/sec across 10 servers = 600K pushes/sec per server → manageable
```



