# 72. Design a Surge Pricing System like Uber or Lyft

---

## 1. Functional Requirements (FR)

- **Dynamic pricing**: Adjust ride prices based on real-time supply (drivers) / demand (ride requests)
- **Geospatial granularity**: Different surge multipliers per geographic zone (H3 hexagons)
- **Real-time computation**: Recalculate surge every 60 seconds
- **Surge display**: Show multiplier before booking confirmation
- **Surge caps**: Maximum 8x; emergency caps (1x during disasters)
- **Driver incentives**: Show high-surge zone heatmap to attract supply
- **Predictive surge**: Forecast upcoming demand spikes using ML
- **Smooth transitions**: Gradual ramp up/down to avoid oscillation

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Surge lookup per zone in < 10 ms
- **Freshness**: Current conditions reflected within 2 minutes
- **Scale**: 500K active zones, 100K surge lookups/sec, 2M driver GPS/sec
- **Availability**: 99.99% — in critical path for every ride request

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Geographic zones (H3 res 7) | ~10M globally |
| Active zones (with activity) | ~500K |
| Recalculation frequency | Every 60 seconds |
| Ride requests / sec | 100K |
| Driver location updates / sec | 2M |
| Surge lookups / sec | 100K |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                     DATA INGESTION LAYER                               │
│                                                                        │
│  Driver App                    Rider App                               │
│  (GPS every 4s)                (ride request)                          │
│       |                              |                                 │
│  +----v---------+             +------v---------+                       │
│  | Location     |             | Ride Request   |                       │
│  | Service      |             | Service        |                       │
│  | (validate,   |             | (validate,     |                       │
│  |  H3 zone     |             |  geocode,      |                       │
│  |  assignment)  |            |  H3 zone)      |                       │
│  +----+---------+             +------+---------+                       │
│       |                              |                                 │
│  +----v-----------+           +------v-----------+                     │
│  | Kafka          |           | Kafka            |                     │
│  | driver-        |           | ride-requests    |                     │
│  | locations      |           | (partitioned by  |                     │
│  | (partitioned   |           |  H3 zone prefix) |                     │
│  |  by zone)      |           +------+-----------+                     │
│  +----+-----------+                  |                                 │
│       |                              |                                 │
└───────|──────────────────────────────|─────────────────────────────────┘
        |                              |
┌───────|──────────────────────────────|─────────────────────────────────┐
│       |    SURGE COMPUTATION LAYER   |                                 │
│       |                              |                                 │
│  +----v------------------------------v----+                            │
│  |         Apache Flink Pipeline          |                            │
│  |                                        |                            │
│  |  1. Sliding Window (5 min, slide 1 min)|                            │
│  |     - Count ride requests per zone     |                            │
│  |     - Count available drivers per zone |                            │
│  |                                        |                            │
│  |  2. Compute demand/supply ratio        |                            │
│  |     - demand = requests / window_min   |                            │
│  |     - supply = drivers / window_min    |                            │
│  |                                        |                            │
│  |  3. Apply surge function               |                            │
│  |     - Piecewise linear mapping         |                            │
│  |     - Apply cap from surge_cap:{city}  |                            │
│  |                                        |                            │
│  |  4. Exponential smoothing              |                            │
│  |     - 0.7 * previous + 0.3 * new      |                            │
│  |     - Read previous from Redis         |                            │
│  |                                        |                            │
│  |  5. Neighbor blending                  |                            │
│  |     - 0.6 * zone + 0.4 * avg(6 nbrs)  |                            │
│  |     - Lookup neighbor zone IDs via H3  |                            │
│  |                                        |                            │
│  |  6. Write to Redis + ClickHouse        |                            │
│  +----+-----------------------------------+                            │
│       |                                                                │
└───────|────────────────────────────────────────────────────────────────┘
        |
┌───────|────────────────────────────────────────────────────────────────┐
│       |              SERVING LAYER                                     │
│       |                                                                │
│  +----v---------+     +----------------+     +-------------------+     │
│  |    Redis     |     | Surge API      |     | Driver Incentive  |     │
│  | surge:{zone} |<----| Service        |     | Service           |     │
│  | = multiplier |     | (lat,lng -> H3 |     | (push high-surge  |     │
│  | TTL: 120s    |     |  -> Redis GET  |     |  zones to nearby  |     │
│  |              |     |  -> return     |     |  drivers via push)|     │
│  | surge_cap:   |     |  multiplier)   |     |                   |     │
│  | {city} = cap |     +-------+--------+     +--------+----------+     │
│  +--------------+             |                       |                │
│                               v                       v                │
│                        Rider App              Driver App               │
│                        (shows surge           (heatmap of              │
│                         before booking)        high-demand zones)      │
│                                                                        │
│  +----------------+     +--------------------+                         │
│  | ClickHouse     |     | Predictive Surge   |                         │
│  | (surge history |     | Service (XGBoost)  |                         │
│  |  for analytics |     | - historical demand|                         │
│  |  and ML        |     | - weather API      |                         │
│  |  training)     |     | - events calendar  |                         │
│  +----------------+     | - real-time trend  |                         │
│                         | -> 15-min forecast |                         │
│                         +--------------------+                         │
│                                                                        │
│  +--------------------+                                                │
│  | Monitoring +       |                                                │
│  | Alerting Service   |                                                │
│  | - surge > 5x for   |                                                │
│  |   > 30 min: alert  |                                                │
│  | - supply = 0 in    |                                                │
│  |   zone: alert      |                                                │
│  | - emergency detect |                                                │
│  |   (demand > 10x    |                                                │
│  |   + news API)      |                                                │
│  +--------------------+                                                │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Surge Calculation Algorithm

```
For each zone (H3 cell, resolution 7 ~ 5.16 km^2), every 60 seconds:

  demand = count(ride_requests in zone, last 5 minutes) / 5  (per-minute rate)
  supply = count(available_drivers in zone, last 5 minutes) / 5
  
  ratio = demand / max(supply, 1)
  
  surge = piecewise_function(ratio):
    ratio <= 1.0:  1.0 (no surge)
    ratio 1.0-1.5: 1.0 + (ratio - 1.0) * 0.5
    ratio 1.5-3.0: 1.25 + (ratio - 1.5) * 1.0
    ratio > 3.0:   min(ratio, 8.0)

  Smoothing (prevent oscillation):
    smoothed = 0.7 * previous_surge + 0.3 * calculated_surge
    
    Without smoothing:
      T=0: high demand -> surge 3x -> riders cancel -> demand drops -> surge 1x
      T=1: riders see 1x -> rush back -> surge 3x -> cycle repeats
    With smoothing: gradual change over 3-5 minutes. Stable UX.
```

#### Why H3 Hexagons (Not Grid Squares)

```
Grid squares: corner cells have different distances from center than edges.
H3 hexagons: all 6 neighbors equidistant. Better circle approximation.
  Resolution 7 (~5 km^2): surge zones
  Resolution 9 (~175m): precise driver matching

Boundary blending:
  rider_surge = 0.6 * zone_surge + 0.4 * avg(neighbor_surges)
  Prevents hard surge boundaries (crossing one street changes price dramatically).
```

#### Predictive Surge (ML)

```
Features: historical demand (same hour/day/week), weather, events (concert end time),
real-time trend (demand increasing?), time of day.

Model: XGBoost per-zone. Prediction horizon: 15-30 min.
Use case: "Zone X will have 3x surge in 15 min" -> show drivers incentive to head there.
Result: supply arrives BEFORE demand spike -> surge is lower -> better UX for everyone.
```

---

## 5. APIs

```http
GET /api/v1/surge?lat=37.7749&lng=-122.4194
Response: 200 OK
{
  "zone_id": "872830926cfffff",
  "surge_multiplier": 2.3,
  "estimated_fare": { "base": 15.00, "surged": 34.50 },
  "message": "Prices are 2.3x due to high demand",
  "updated_at": "2026-03-14T11:00:00Z"
}

GET /api/v1/surge/heatmap?ne_lat=37.82&ne_lng=-122.35&sw_lat=37.70&sw_lng=-122.52
Response: 200 OK
{ "zones": [{ "zone_id": "...", "surge": 2.3, "center": [37.78, -122.41] }, ...] }
```

---

## 6. Data Model

### Redis — Current Surge

```
surge:{zone_id}  --> Hash { multiplier: 2.3, demand: 45, supply: 20, updated_at: ts }
TTL: 120 seconds (stale if not refreshed)

surge_cap:{city}  --> FLOAT (admin override, e.g., 1.0 during emergency)
No TTL (manually removed)
```

### ClickHouse — Historical Surge

```sql
CREATE TABLE surge_history (
    zone_id String, multiplier Float32, demand UInt32, supply UInt32,
    timestamp DateTime, date Date MATERIALIZED toDate(timestamp)
) ENGINE = MergeTree() PARTITION BY toYYYYMM(timestamp)
  ORDER BY (zone_id, timestamp);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Surge service down** | Default to 1.0x — underprice rather than block rides |
| **Stale surge data** | TTL 120s; if expired, use historical pattern or 1.0x |
| **Location data lag** | Use last known positions; degrade gracefully |
| **Emergency events** | Admin override: SET surge_cap:{city} 1.0 — instant |
| **Oscillation** | Exponential smoothing prevents wild swings |

---

## 8. Deep Dive: Engineering Trade-offs

### Ethical Considerations

```
Surge during emergencies (hurricane, attack):
  Auto-detect: demand > 10x normal AND news API reports emergency -> cap at 1.0x
  Manual override: ops can cap any city instantly
  Regulatory: some cities mandate caps (NYC: 2.5x during emergencies)

Transparency:
  Show surge BEFORE booking (rider chooses to accept or wait)
  "Wait and save" option: "Surge likely to decrease in ~10 minutes"
  Fare estimate with surge shown prominently (no surprise at end)

Why surge is necessary:
  1. Incentivizes drivers to high-demand areas (supply response)
  2. Reduces demand (riders who can wait, do wait)
  3. Without surge: high-demand periods have ZERO drivers -> worse for everyone
```

### Granularity: Zone Size Trade-off

```
Large zones (10 km^2):
  ✓ More data points per zone -> more accurate demand/supply estimate
  ✗ Masks hyperlocal demand (airport vs nearby residential)

Small zones (0.5 km^2):
  ✓ Precise surge reflecting local conditions
  ✗ Fewer data points -> noisy, unreliable estimates
  ✗ "Surge boundary" problem: crossing one street changes price

Sweet spot: H3 resolution 7 (~5 km^2) with neighbor blending.
For airports/stadiums: use resolution 8 (~1 km^2) custom zones.
```

### Flink Window Semantics — Why Sliding Windows

```
Tumbling window (5 min): counts reset every 5 min.
  At minute 4:59 -> high surge. At minute 5:00 -> counter resets to 0 -> surge drops to 1x.
  Sudden drops at window boundaries = bad UX.

Sliding window (5 min window, 1 min slide):
  At any given second, the window covers the LAST 5 minutes.
  Every 1 minute, Flink re-evaluates with updated counts.
  No sudden resets. Smooth, continuous surge updates.

Implementation in Flink:
  DataStream<RideRequest> requests = ...
  requests
    .keyBy(event -> event.zoneId)        // partition by H3 zone
    .window(SlidingEventTimeWindows.of(   // 5-min window
        Time.minutes(5), Time.minutes(1))) // slide every 1 min
    .aggregate(new DemandSupplyAggregator())
    .map(new SurgeCalculator())           // apply piecewise function + smoothing
    .addSink(new RedisSink());            // write to Redis

Watermark strategy:
  Allow 10-second out-of-orderness for late GPS events.
  Events arriving > 10 seconds late are dropped (acceptable for surge accuracy).
```

### Driver Supply Response — Feedback Loop

```
Surge creates a feedback loop:

  High demand -> surge rises -> drivers see high-surge zone on heatmap
  -> drivers drive to that zone -> supply increases -> surge decreases

Measurement: "supply elasticity to surge"
  How many extra drivers appear per 1x increase in surge?
  Typical: 1.5x surge attracts 30% more drivers within 10 minutes
  3.0x surge attracts 100% more drivers within 15 minutes

Driver incentive push notification:
  When zone surge > 2.0x AND duration > 3 minutes:
    Push to drivers within 10 km:
    "High demand in Downtown! Earn 2.3x fares. Estimated $45 for next ride."
    
  Only push if driver is online, not in a ride, and hasn't been pushed in last 15 min
  (prevent notification fatigue)

Surge decay on supply arrival:
  As drivers arrive -> supply increases -> smoothed surge decreases
  Important: decay must be gradual (smoothing factor 0.7)
  If decay is too fast: drivers arrive, surge drops, drivers leave, surge rises again
  (ping-pong effect)
```

