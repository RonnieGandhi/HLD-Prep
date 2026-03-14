# 56. Design an ETA Calculation Service

---

## 1. Functional Requirements (FR)

- **Point-to-point ETA**: Given origin and destination, estimate travel time considering current traffic
- **Multi-stop ETA**: ETA for routes with multiple waypoints (delivery routes)
- **ETA for pickup**: How long until a driver/courier reaches a pickup point (live tracking + prediction)
- **ETA for trip**: Predicted trip duration from pickup to dropoff
- **Historical ETA**: "How long does this trip usually take on Tuesday at 8 AM?"
- **Batch ETA**: Compute ETAs for N origin-destination pairs efficiently (e.g., matching drivers to riders)
- **ETA updates**: Continuously update ETA as trip progresses and conditions change
- **Multi-modal ETA**: Support driving, walking, cycling, transit modes

---

## 2. Non-Functional Requirements (NFRs)

- **Accuracy**: ETA within ±10% of actual travel time (key business metric)
- **Low Latency**: Single ETA response in < 100 ms; batch of 20 ETAs in < 500 ms
- **High Throughput**: 500K ETA requests/sec at peak (matching systems query ETA for every candidate driver)
- **Freshness**: Traffic conditions reflected in ETA within 60 seconds
- **Availability**: 99.99% — ETA is in the critical path for ride-hailing/delivery matching
- **Graceful degradation**: If real-time traffic unavailable, fall back to historical patterns
- **Global**: Support all major cities with varying road network characteristics

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| ETA requests / sec | 500K peak |
| Single ETA latency target | < 100 ms |
| Batch ETA (20 pairs) target | < 500 ms |
| Active road segments tracked | 50M globally |
| Traffic updates / sec | 10M GPS data points |
| Road graph size (in-memory) | ~300 GB globally |
| ETA model features | ~50 per query |
| ETA cache hit rate | ~30% (routes repeat) |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Rider App   │    │ Matching Svc │    │ Delivery Svc │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼───────┐
                     │  ETA Service │ ← Unified API
                     │  (API layer) │
                     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
       ┌──────▼──────┐ ┌───▼───────┐ ┌───▼──────────┐
       │  Route      │ │ Traffic   │ │ ML ETA       │
       │  Engine     │ │ Service   │ │ Prediction   │
       │  (OSRM/     │ │ (Live     │ │ Model        │
       │   Valhalla) │ │  segment  │ │ (Correction  │
       │             │ │  speeds)  │ │  layer)      │
       └──────┬──────┘ └───┬───────┘ └───┬──────────┘
              │             │             │
       ┌──────▼──────┐ ┌───▼───────┐ ┌───▼──────────┐
       │  Road Graph │ │ Redis     │ │ Feature      │
       │  (In-memory │ │ (Live     │ │ Store        │
       │   CH graph) │ │  traffic  │ │ (Redis/      │
       │             │ │  per seg) │ │  ClickHouse) │
       └─────────────┘ └───┬───────┘ └──────────────┘
                            │
                     ┌──────▼───────┐
                     │   Kafka      │
                     │ (GPS traces, │
                     │  traffic     │
                     │  aggregation)│
                     └──────┬───────┘
                            │
                     ┌──────▼───────┐
                     │  Traffic     │
                     │  Aggregation │
                     │  (Flink)     │
                     └──────────────┘
```

### Component Deep Dive

#### ETA Calculation Pipeline — Three-Layer Architecture

```
Layer 1: Route Engine ETA (physics-based)
  Input: origin (lat, lng), destination (lat, lng)
  Process:
    1. Find route via Contraction Hierarchies (< 1 ms)
    2. Route = list of road segments with distances
    3. For each segment: time = distance / speed
       Speed = min(speed_limit, current_traffic_speed)
    4. Sum segment times = raw ETA
    
  Result: "Route is 15.2 km, raw ETA = 22 minutes"
  
  Limitation: Doesn't account for:
    - Turn delays (left turn at busy intersection = 2 min wait)
    - Traffic signal timing
    - Construction zones
    - Time of day patterns (traffic may get worse during the trip)

Layer 2: Traffic-Adjusted ETA
  For each road segment in the route:
    traffic_speed = Redis GET traffic:{segment_id} → current avg speed
    If no traffic data: use historical avg speed for this segment at this time of day
    
    segment_time = segment_length / traffic_speed
    
  Additional adjustments:
    - Turn penalties: add 15-60 seconds per major turn (based on turn type and intersection complexity)
    - Traffic signal density: add avg wait time per signal
    - Construction/closure: add detour time
    
  Result: "Traffic-adjusted ETA = 28 minutes"

Layer 3: ML Correction Model ⭐ (the secret sauce)
  Even Layer 2 has systematic errors:
    - Segment speed data is aggregate (avg of all vehicles, not YOUR path)
    - Doesn't predict future traffic changes during the trip
    - Doesn't account for micro-delays (parking, passenger loading)
  
  ML model: Predict actual_time / route_engine_time ratio
  
  Features:
    1. Route features: distance, segment count, highway_pct, urban_pct
    2. Traffic features: congestion_level_avg, congestion_variance, trend (improving/worsening)
    3. Temporal: hour_of_day, day_of_week, is_holiday, minutes_to_rush_hour
    4. Weather: rain, snow, visibility
    5. Historical: median actual time for similar trips at this time of day
    6. Spatial: origin/destination neighborhood characteristics
    7. Trip-specific: pickup_time_estimate (for ride-hailing)
  
  Model: Gradient Boosted Trees (XGBoost/LightGBM)
    - Why not deep learning? GBT works well with tabular features; faster inference
    - Training data: millions of completed trips with (predicted_eta, actual_time) pairs
    - Output: correction_factor (e.g., 1.15 → actual time is 15% longer than route engine says)
    
  Final ETA = route_engine_ETA × correction_factor
  Result: "ML-corrected ETA = 32 minutes" (route engine was optimistic by 15%)
```

#### Traffic Aggregation — Converting GPS Traces to Segment Speeds

```
Pipeline:

1. GPS Ingestion (10M points/sec):
   Kafka topic: "gps-traces"
   Message: { device_id, lat, lng, speed, heading, timestamp }

2. Map Matching (Flink):
   GPS point → which road segment is this device on?
   HMM map matching (see Q53 for details)
   Output: { segment_id, speed, timestamp }

3. Segment Speed Aggregation (Flink, 60-second tumbling window):
   For each segment_id in the window:
     speeds = [all reported speeds]
     avg_speed = median(speeds)  // median more robust to outliers than mean
     sample_count = len(speeds)
     confidence = min(1.0, sample_count / 10)  // need ≥10 samples for high confidence
   
   Output: { segment_id, avg_speed, sample_count, confidence, window_end }

4. Store in Redis:
   HSET traffic:{segment_id} speed {avg_speed} confidence {confidence} updated_at {timestamp}
   EXPIRE traffic:{segment_id} 120  // stale after 2 minutes without update

5. Fallback hierarchy:
   a. Real-time traffic (last 60 sec): confidence HIGH
   b. Recent traffic (last 5 min): confidence MEDIUM  
   c. Historical pattern (same day/time, last 4 weeks average): confidence LOW
   d. Speed limit × 0.7 (default): confidence VERY LOW

Problem: Sparse coverage
  Rural roads: maybe 0 GPS traces in the last hour
  Solution: Historical patterns fill the gaps
  
  ClickHouse: Pre-computed historical speeds
  SELECT avg(speed) FROM segment_speeds_historical
  WHERE segment_id = ? AND day_of_week = ? AND hour = ?
  GROUP BY segment_id
  
  Updated weekly via Spark batch job processing all GPS traces
```

#### Batch ETA — Matching System Integration

```
The matching system (ride-hailing) needs ETAs for 20 candidate drivers to 1 rider.

Naive: 20 sequential ETA API calls → 20 × 100 ms = 2 seconds. Too slow!

Batch optimization:

1. Many-to-One Routing:
   All 20 origins to 1 destination = reverse Dijkstra from destination
   Start at destination → expand outward → find all 20 origins
   Much faster than 20 separate queries because search space is shared!
   
   Time: ~5 ms for 20 origins (vs 20 ms for 20 separate queries)

2. ETA Matrix (pre-computation):
   Divide city into H3 cells (resolution 9 ≈ 175m)
   Pre-compute ETA between ALL cell pairs (within 30 km)
   
   Number of cells per city: ~50K
   Cell pairs within 30 km: ~500M
   Storage: 500M × 4 bytes = 2 GB per city
   
   Lookup: O(1) — just a table lookup!
   Accuracy: ±2-3 minutes (coarse, but good enough for initial matching)
   
   Refinement: After initial matching narrows to top 3 drivers,
   compute exact ETA with full route + traffic for final selection.

3. Traffic-speed matrix:
   Instead of per-segment traffic, pre-aggregate traffic speed by H3 cell
   traffic_cell:{h3_cell_id} → avg_speed_in_cell
   Coarser but enables very fast batch ETA estimation
   
   cell_eta = haversine_distance(cell_A, cell_B) / avg_speed_in_path_cells
```

#### Live ETA Updates During Navigation

```
User is in a ride. ETA was 32 minutes. 15 minutes in, how to update?

Update triggers:
  1. Every 60 seconds (periodic)
  2. When traffic conditions change significantly on remaining route
  3. When driver deviates from planned route

Update calculation:
  remaining_distance = total_route_distance - distance_traveled
  remaining_route = segments from current_position to destination
  
  For remaining segments:
    ETA = Σ(segment_length / current_traffic_speed) × ml_correction
  
  Smoothing: Don't show erratic ETA changes
    raw_eta_update = 17 minutes
    previous_eta = 18 minutes
    displayed_eta = 0.7 × previous_eta + 0.3 × raw_eta_update = 17.7 → show "18 min"
    
    Only show change when delta > 2 minutes or > 10% change
    
  Why smoothing matters:
    GPS jitter → small route changes → ETA fluctuates ±1 minute
    Without smoothing: "17 min" → "18 min" → "17 min" → "19 min" → confusing
    With smoothing: "18 min" (stable for 2 minutes) → "17 min" → user trusts it
```

---

## 5. APIs

### Single ETA
```http
GET /api/v1/eta?origin_lat=37.7749&origin_lng=-122.4194&dest_lat=37.3382&dest_lng=-121.8863&mode=driving&departure_time=now
Response: 200 OK
{
  "eta_seconds": 1920,
  "eta_display": "32 min",
  "distance_meters": 72400,
  "confidence": 0.85,
  "traffic_level": "moderate",
  "route_summary": "US-101 S",
  "breakdown": {
    "route_engine_eta": 1680,
    "traffic_adjustment": 180,
    "ml_correction": 60
  }
}
```

### Batch ETA (Many Origins → One Destination)
```http
POST /api/v1/eta/batch
{
  "origins": [
    {"lat": 37.78, "lng": -122.41, "id": "driver-1"},
    {"lat": 37.77, "lng": -122.43, "id": "driver-2"},
    {"lat": 37.76, "lng": -122.40, "id": "driver-3"}
  ],
  "destination": {"lat": 37.7749, "lng": -122.4194},
  "mode": "driving"
}
Response: 200 OK
{
  "etas": [
    {"id": "driver-1", "eta_seconds": 240, "distance_meters": 1200},
    {"id": "driver-2", "eta_seconds": 480, "distance_meters": 2100},
    {"id": "driver-3", "eta_seconds": 360, "distance_meters": 1800}
  ]
}
```

### ETA Matrix (Cell-to-Cell Pre-computed)
```http
GET /api/v1/eta/matrix?origin_h3=892830926cfffff&dest_h3=89283092e3fffff
Response: 200 OK
{
  "origin_cell": "892830926cfffff",
  "dest_cell": "89283092e3fffff",
  "eta_seconds": 600,
  "distance_meters": 5200,
  "source": "precomputed",
  "confidence": 0.7,
  "last_updated": "2025-03-14T10:00:00Z"
}
```

### Live ETA Update (WebSocket during navigation)
```json
// Server pushes to client every 60 seconds:
{
  "type": "eta_update",
  "session_id": "nav-uuid",
  "remaining_eta_seconds": 1080,
  "remaining_distance_meters": 35200,
  "arrival_time": "2025-03-14T10:50:00Z",
  "traffic_ahead": "moderate",
  "route_changed": false
}
```

---

## 6. Data Model

### Redis — Live Traffic + ETA Cache

```
# Per-segment live traffic
traffic:{segment_id}     → Hash { speed: 45.2, confidence: 0.9, updated_at: 1710400000 }
TTL: 120 seconds

# Pre-computed ETA matrix (H3 cell pairs)
eta_matrix:{origin_h3}:{dest_h3} → INT (eta_seconds)
TTL: 300 seconds (refreshed every 5 minutes with current traffic)

# ETA cache (exact origin-destination pairs that repeat)
eta_cache:{origin_geohash6}:{dest_geohash6}:{mode}:{hour} → INT (eta_seconds)
TTL: 300

# Historical segment speeds (by time slot)
hist_speed:{segment_id}:{day_of_week}:{hour} → FLOAT (avg speed)
No TTL (updated weekly)
```

### ClickHouse — Historical Trip Data (ML Training + Analytics)

```sql
CREATE TABLE completed_trips (
    trip_id         UUID,
    origin_lat      Float64,
    origin_lng      Float64,
    dest_lat        Float64,
    dest_lng        Float64,
    origin_h3       UInt64,
    dest_h3         UInt64,
    distance_meters UInt32,
    predicted_eta   UInt32,
    actual_duration UInt32,
    departure_hour  UInt8,
    day_of_week     UInt8,
    is_holiday      UInt8,
    weather         Enum8('clear'=0,'rain'=1,'snow'=2,'fog'=3),
    route_segments  Array(UInt64),
    mode            Enum8('driving'=0,'walking'=1,'cycling'=2),
    trip_date       Date MATERIALIZED toDate(created_at),
    created_at      DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (origin_h3, dest_h3, departure_hour, day_of_week);

-- Training query: get all trips between these cells to train ML model
-- Analytics: "What's the average prediction error this week?"
```

### Kafka Topics

```
Topic: gps-traces             (10M msgs/sec, partitioned by geohash region)
Topic: segment-traffic        (aggregated speeds, partitioned by segment_id)
Topic: eta-predictions        (prediction logs for ML model monitoring)
Topic: trip-completed         (actual vs predicted for model retraining)
```

### ML Model Store

```
Model: gradient_boosted_trees_v23.pkl
  - Trained on 50M trips from last 6 months
  - Features: 50 input features
  - Output: correction_factor (float)
  - Serving: loaded in ETA Service memory (model size ~50 MB)
  - Inference time: < 1 ms per prediction
  - Retrained weekly; A/B tested before deployment

Feature Store (Redis):
  feature:{origin_h3}:{hour}:{dow}  → Hash { 
    hist_median_speed, hist_p25_speed, hist_p75_speed,
    avg_trip_duration, trip_count, congestion_probability 
  }
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Traffic service down** | Fall back to historical speed patterns (ClickHouse → Redis backup) |
| **ML model error** | Circuit breaker: if model latency > 50 ms or error rate > 5%, bypass ML layer; use route engine ETA only |
| **Route engine crash** | Multiple replicas; if all down, use pre-computed ETA matrix (coarser but available) |
| **Stale traffic data** | TTL on Redis keys; if traffic data > 2 min old, use historical + confidence=LOW flag |
| **GPS data pipeline lag** | Flink checkpointing; if lag > 5 min, switch to historical patterns globally |
| **ETA cache thundering herd** | Singleflight: if 100 concurrent requests for same ETA, only compute once; others wait |

### Specific: Model Monitoring and Rollback

```
Problem: New ML model deployed, but ETA accuracy degrades in certain cities

Monitoring:
  For each ETA prediction, log: (predicted_eta, actual_time) when trip completes
  Kafka → ClickHouse: compute MAPE (Mean Absolute Percentage Error) per city per hour
  
  Alert if MAPE > 15% for any city (normal is 8-10%)
  
  Dashboard:
  SELECT city, 
         avg(abs(predicted_eta - actual_duration) / actual_duration) * 100 as mape
  FROM completed_trips
  WHERE created_at > now() - INTERVAL 1 HOUR
  GROUP BY city
  ORDER BY mape DESC

Rollback:
  Model versions stored in S3: model_v22.pkl, model_v23.pkl
  If v23 has high MAPE → revert to v22 (< 1 minute, just reload model in memory)
  
  A/B testing:
  Route 5% of traffic to new model → compare MAPE
  If better → gradual rollout (5% → 25% → 50% → 100%)
  If worse → auto-rollback after 1 hour of elevated MAPE
```

### Specific: ETA for Trips Crossing Time Boundaries

```
Problem: Trip starts at 5:30 PM (pre-rush), arrives at 6:30 PM (peak rush)
  Route engine uses CURRENT traffic speed for ALL segments
  But traffic will be WORSE on segments reached later!

Solution: Time-dependent routing
  For each segment along the route:
    estimated_arrival_at_segment = departure_time + sum(previous_segment_times)
    traffic_speed = predicted_speed(segment_id, estimated_arrival_time)
    segment_time = segment_length / traffic_speed
  
  predicted_speed(segment, time):
    If time is within 60 minutes → use real-time traffic + trend
    If time is 1-3 hours ahead → blend real-time (30%) + historical (70%)
    If time is 3+ hours ahead → use historical patterns only
  
  Result: ETA accounts for traffic getting worse (or better) during the trip
  
  Complexity: O(N × T) where N = segments, T = time predictions per segment
  In practice: T=1 (use predicted speed at arrival time) → still O(N)
```

---

## 8. Deep Dive: Engineering Trade-offs

### Haversine Distance vs Road Distance vs Routing ETA

```
For different use cases, different precision levels are needed:

1. Haversine (straight-line distance):
   Formula: Great circle distance on Earth's surface
   Compute time: O(1), < 1 μs
   Use case: Initial filtering ("find drivers within 5 km")
   Accuracy: Can be 50-200% off from road distance (rivers, highways, one-way streets)
   
2. Road distance (shortest path distance):
   Method: Contraction Hierarchies shortest path
   Compute time: < 1 ms
   Use case: Ranking candidates after initial filter
   Accuracy: Distance is correct, but time estimate is rough (assumes speed limit)

3. Routing ETA (full traffic-aware):
   Method: CH + traffic data + ML correction
   Compute time: 10-50 ms
   Use case: Final ETA shown to user; matching decision
   Accuracy: ±10% of actual time

Layered approach in matching:
  Step 1: Haversine filter → 1000 drivers → 100 within 5 km (< 1 ms)
  Step 2: Road distance rank → top 20 drivers (< 5 ms)
  Step 3: Full routing ETA → top 5 drivers (< 50 ms)
  Step 4: Select best driver (< 1 ms)
  Total: < 60 ms for finding optimal match from 1000 candidates
```

### Why XGBoost for ETA (Not Deep Learning)?

```
XGBoost / LightGBM advantages for ETA:
  1. Tabular data: ETA features are structured (numbers, categories)
     → GBT excels on tabular data (often beats DL on Kaggle for structured data)
  
  2. Inference speed: < 1 ms per prediction
     DL models: 5-50 ms (even with GPU)
     At 500K QPS: ms matters
  
  3. Interpretability: Feature importance is transparent
     "hour_of_day is the most important feature for NYC"
     → helps debug when ETA is wrong in specific conditions
  
  4. Training speed: GBT trains in minutes on 50M samples
     DL: hours to days
  
  5. Robust to missing features: If weather data is unavailable, GBT handles it gracefully
     DL: may need special handling for missing inputs

When DL IS better:
  - Sequence modeling: predict ETA from GPS trace sequence (LSTM/Transformer)
    "Given the driver's last 50 GPS points, predict remaining time"
    Captures driving behavior, stop patterns, traffic signals
  - Graph neural networks: model the road network structure
    Capture spatial dependencies (congestion on one road affects neighbors)
  
  Google Maps uses a hybrid: GBT for base ETA + GNN for spatial traffic prediction
```

### ETA Percentiles: Not Just Average, But Confidence Interval

```
User cares about: "Will I make it to my meeting at 10 AM?"

Showing single ETA = 28 minutes doesn't capture uncertainty.
Better: "28 minutes (25-35 min range)"

Implementation:
  ML model predicts distribution, not just point estimate:
  
  Quantile regression:
    Model 1 (p10): "90% chance you'll take MORE than 25 minutes"
    Model 2 (p50): "50% chance you'll take MORE than 28 minutes" (median)
    Model 3 (p90): "90% chance you'll arrive within 35 minutes"
  
  Or: Predict mean and variance → assume normal distribution
    P10 = mean - 1.28 × std
    P90 = mean + 1.28 × std

Use cases:
  - Rider ETA display: show P50 (median) → "28 min"
  - Delivery promise: use P90 → "Arrives by 10:35 AM" (90% confidence)
  - Matching: use P50 for ranking, but flag if P90 is too high
  - Surge pricing: wide confidence interval → uncertain demand → less aggressive surge

Factors that increase uncertainty:
  - Rain/snow: adds 20-50% variance
  - Rush hour start/end: traffic transitions are unpredictable
  - Accidents ahead: unknown delay duration
  - Route through downtown vs highway: downtown has more variability
```

### Cold Start: ETA in a New City with No Historical Data

```
Problem: Launching in a new city. No GPS traces, no completed trips, no historical patterns.

Bootstrapping strategy:
  1. Use speed limits as baseline (publicly available road data from OpenStreetMap)
     ETA = Σ(segment_length / speed_limit × 0.7)  // 0.7 = typical speed/limit ratio
     Accuracy: ±30-40% (rough but functional)
  
  2. Import historical traffic from third-party (HERE, TomTom)
     Provides time-of-day speed profiles per road segment
     Accuracy: ±15-20%
  
  3. Seed with partner driver data
     First 1,000 drivers in the city → collect GPS traces aggressively
     After 1 week: enough data for basic traffic model
     After 1 month: ML model has enough training data for city-specific correction
  
  4. Transfer learning from similar cities
     Train ML model on NYC → apply to London (similar density, driving patterns)
     Fine-tune with local data as it accumulates
     
  5. Progressive accuracy improvement
     Week 1: ±30% (speed limit based)
     Month 1: ±15% (traffic patterns emerging)
     Month 6: ±10% (full ML model trained on local data)
     Year 1: ±8% (mature, matches established cities)
```



