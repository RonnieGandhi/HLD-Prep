# 58. Design a Bike Sharing System like Citi Bike

---

## 1. Functional Requirements (FR)

- **Station map**: Show nearby docking stations with available bikes and empty docks in real-time
- **Rent a bike**: Unlock a bike from a station using app/membership card
- **Return a bike**: Dock a bike at any station; end the trip
- **Pricing**: Time-based pricing (first 30 min free for members, then per-minute charges)
- **Membership**: Annual/monthly memberships and single-ride passes
- **Trip history**: View past rides with duration, distance, route, cost
- **Rebalancing alerts**: Notify operations when stations are too full or too empty
- **Station status**: Real-time dock availability (bikes available, docks available)
- **Reservation**: Reserve a bike at a station for 5 minutes (prevent someone else from taking it)
- **E-bike support**: Battery level tracking, pricing premium for e-bikes

---

## 2. Non-Functional Requirements (NFRs)

- **Consistency**: Bike checkout MUST be exactly-once (two users can't rent the same bike)
- **Low Latency**: Bike unlock in < 3 seconds after request
- **Real-time Station Data**: Availability updates within 10 seconds of change
- **Availability**: 99.9% — system downtime means stranded riders and locked bikes
- **Scalability**: 50K+ bikes, 5K+ stations, 500K+ daily trips
- **Durability**: Trip records for billing must never be lost
- **Fault Tolerant**: Bikes must be unlockable even during partial system outages

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total bikes | 50,000 |
| Total stations | 5,000 |
| Docks per station | 10-40 (avg 15) |
| Daily trips | 500K (peak: summer weekday) |
| Trips / sec | ~15 peak (rush hour: ~50) |
| Active rides concurrently | ~25,000 (at peak hour) |
| Station status updates / sec | ~100 (each dock change) |
| Bike heartbeat (e-bikes) | Every 30 sec = 50K/30 = ~1,700/sec |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Mobile App  │    │  Kiosk       │    │  RFID Card   │
│  (User)      │    │  (Station)   │    │  Tap         │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼───────┐
                     │  API Gateway │
                     └──────┬───────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                    │
 ┌──────▼──────┐    ┌──────▼──────┐    ┌────────▼────────┐
 │  Trip       │    │  Station    │    │  User/Member     │
 │  Service    │    │  Service    │    │  Service         │
 │  (Rent,     │    │  (Avail-    │    │  (Auth, billing, │
 │  Return,    │    │  ability,   │    │   membership)    │
 │  Pricing)   │    │  Status)    │    │                  │
 └──────┬──────┘    └──────┬──────┘    └─────────────────┘
        │                   │
 ┌──────▼──────┐    ┌──────▼──────┐
 │  Bike       │    │  Dock       │
 │  Service    │    │  Controller │ ← Communication with physical docks
 │  (Lock/     │    │  Service    │   (unlock command, dock sensor)
 │  Unlock,    │    │             │
 │  GPS, E-    │    │             │
 │  bike)      │    │             │
 └──────┬──────┘    └──────┬──────┘
        │                   │
        └───────────┬───────┘
                    │
             ┌──────▼───────┐
             │    Kafka     │
             │  (Trip events│
             │   station    │
             │   changes)   │
             └──────┬───────┘
                    │
     ┌──────────────┼──────────────┐
     │              │              │
┌────▼────┐  ┌─────▼─────┐  ┌────▼──────┐
│Rebalance│  │ Billing   │  │ Analytics │
│ Service │  │ Service   │  │ (Click-   │
│ (Fleet  │  │ (Charge   │  │  House)   │
│  ops)   │  │  user)    │  │           │
└─────────┘  └───────────┘  └───────────┘

Data Stores:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │  ClickHouse  │
│ (Trips, Users│  │ (Station     │  │ (Analytics,  │
│  Bikes, Stn) │  │  avail.,     │  │  demand      │
│              │  │  bike locks, │  │  forecasting)│
│              │  │  reservations│  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Bike Checkout (Rent) Flow — The Critical Path

```
User taps "Unlock Bike" at Station 42, Dock 7:

Step 1: Validate user
  a. Is user authenticated? (JWT token check)
  b. Has active membership or valid payment method?
  c. Any outstanding balance > $50? → Block rental
  d. Already has an active ride? (one bike at a time)

Step 2: Reserve the bike (prevent race condition)
  This is the critical consistency point!
  
  Two users tap "unlock" at the same dock within 1 second:
  
  Solution: Distributed lock on the bike
  
  Redis:
    SET bike_lock:{bike_id} {user_id} NX EX 30
    NX = only set if not exists → exactly one user wins
    EX = 30 second expiry (if unlock fails, lock auto-releases)
  
  If SET returns OK → proceed with unlock
  If SET returns nil → bike already being checked out → "Bike unavailable, try another dock"

Step 3: Unlock the bike
  a. Send unlock command to dock controller (IoT message via MQTT)
     Topic: docks/{station_id}/{dock_id}/command
     Payload: { "action": "unlock", "bike_id": "B-123" }
  
  b. Wait for dock ACK (dock confirms bike physically released)
     Timeout: 10 seconds
     If no ACK → retry once → if still no ACK → fail, release lock, refund
  
  c. Dock sensor confirms: bike removed from dock

Step 4: Start trip
  INSERT INTO trips (trip_id, user_id, bike_id, start_station_id, start_dock_id, 
                     started_at, status)
  VALUES (..., 'active')
  
  Publish: Kafka "trip-started" event

Step 5: Update station availability
  Redis: 
    DECR station:{station_id}:bikes_available
    INCR station:{station_id}:docks_available
  
  Publish: Kafka "station-change" event → pushed to all apps viewing this station

Total latency: 2-3 seconds (auth + lock + IoT unlock + DB write)
```

#### Bike Return (Dock) Flow

```
User docks bike at Station 55, Dock 12:

Step 1: Dock sensor detects bike inserted
  Dock controller publishes:
    MQTT topic: docks/{station_id}/{dock_id}/event
    Payload: { "event": "bike_docked", "bike_id": "B-123" }

Step 2: Trip Service processes return
  a. Find active trip for bike_id B-123
     SELECT * FROM trips WHERE bike_id = 'B-123' AND status = 'active'
  
  b. Complete the trip
     UPDATE trips SET 
       end_station_id = station_55,
       end_dock_id = dock_12,
       ended_at = NOW(),
       duration_minutes = EXTRACT(EPOCH FROM (NOW() - started_at)) / 60,
       status = 'completed'
     WHERE trip_id = ?
  
  c. Calculate fare
     See pricing section below

Step 3: Lock the bike
  Dock controller locks the bike mechanism
  Release Redis lock: DEL bike_lock:{bike_id}

Step 4: Update station availability
  Redis:
    INCR station:{station_55}:bikes_available
    DECR station:{station_55}:docks_available

Step 5: Billing
  Publish Kafka "trip-completed" event → Billing Service charges user
  
Step 6: (E-bikes) Start charging
  If e-bike → dock initiates battery charging
  Report battery level to Bike Service
```

#### Pricing Engine

```
Pricing tiers:

Member (annual/monthly):
  First 30 min: FREE (included in membership)
  31-60 min: $0.15/min
  61+ min: $0.30/min
  E-bike: $0.20/min from minute 1 (no free period)

Day Pass:
  First 30 min: $4.00 flat
  31-60 min: $0.20/min
  61+ min: $0.40/min

Single Ride:
  $1 unlock fee + $0.25/min

Implementation:
  def calculate_fare(trip):
      duration = trip.duration_minutes
      
      if trip.membership_type == 'annual':
          if trip.bike_type == 'ebike':
              return duration * 0.20
          elif duration <= 30:
              return 0
          elif duration <= 60:
              return (duration - 30) * 0.15
          else:
              return 30 * 0.15 + (duration - 60) * 0.30
      
      elif trip.membership_type == 'day_pass':
          if duration <= 30:
              return 4.00
          elif duration <= 60:
              return 4.00 + (duration - 30) * 0.20
          else:
              return 4.00 + 30 * 0.20 + (duration - 60) * 0.40
      
      else:  # single ride
          return 1.00 + duration * 0.25

Surge pricing (optional):
  During high-demand periods, no free minutes for members
  "Bike angels" program: reward users who return bikes to empty stations
```

#### Station Rebalancing — The Operational Challenge

```
Problem: Commuters ride from residential areas to business districts in the morning.
  By 9 AM: residential stations EMPTY, business stations FULL
  Nobody can rent from empty stations or return to full stations.

Rebalancing strategies:

1. Truck-based rebalancing (reactive):
   Monitor station fill levels
   When station < 20% full or > 80% full → alert rebalancing team
   Trucks redistribute bikes between stations
   
   Optimization: Solve vehicle routing problem (VRP)
   Given: set of "surplus" stations and "deficit" stations + truck fleet
   Minimize: total truck travel time while balancing all stations
   Algorithm: Capacitated VRP solver (Google OR-Tools)

2. Incentive-based rebalancing (proactive) ⭐:
   "Bike Angels" program (Citi Bike actually does this):
   Show users: "Return to Station X → earn 3 points"
   Points redeemable for membership credit
   
   Users naturally rebalance the system for us!

3. Predictive rebalancing:
   ML model predicts demand per station per hour
   Features: time_of_day, day_of_week, weather, events, historical patterns
   
   Pre-position bikes BEFORE demand spike:
   Forecast: "Station 42 will need 15 bikes by 8 AM, currently has 5"
   → Schedule truck at 6 AM to add 10 bikes

Rebalancing Service:
  Every 5 minutes:
    For each station:
      current = Redis GET station:{id}:bikes_available
      target = ML prediction for next hour
      delta = target - current
    
    Sort stations by abs(delta)
    Generate rebalancing plan (which trucks go where, how many bikes)
    Push plan to Operations Dashboard
```

#### Handling Full / Empty Stations — "Station Full" Problem

```
User arrives at destination station: ALL DOCKS FULL. Can't return bike!

Solutions:
  1. Extra 15 minutes free: app shows nearby stations with available docks
     "Station 55 is full. Station 57 (2 blocks away) has 5 docks available."
     Give user 15 extra free minutes to reach alternative station
  
  2. Virtual return: user marks "returned" at full station
     Bike stays in area (not docked); ops team collects it
     User not charged overtime
     Risk: bike theft → only allow for members with history
  
  3. Overflow dock: some stations have expandable overflow areas
     Not locked into dock, but GPS-tracked
  
  4. Push notification: "Station 55 is filling up — consider Station 57"
     Sent when user is 5 minutes away and station is > 80% full

Similar for empty stations:
  "Station 42 has no bikes. Station 40 (1 block away) has 3 bikes."
  Show alternatives sorted by distance
```

---

## 5. APIs

### Rent a Bike
```http
POST /api/v1/trips/start
{
  "station_id": "stn-42",
  "dock_id": "dock-7",
  "bike_type": "classic"    // or "ebike"
}
Response: 200 OK
{
  "trip_id": "trip-uuid",
  "bike_id": "B-123",
  "started_at": "2025-03-14T08:30:00Z",
  "pricing_plan": "annual_member",
  "free_minutes": 30,
  "unlock_status": "success"
}
```

### Return a Bike
```http
POST /api/v1/trips/end
{
  "trip_id": "trip-uuid",
  "station_id": "stn-55",
  "dock_id": "dock-12"
}
Response: 200 OK
{
  "trip_id": "trip-uuid",
  "duration_minutes": 22,
  "distance_km": 4.2,
  "fare": 0.00,
  "fare_breakdown": { "base": 0, "overage": 0, "ebike_premium": 0 },
  "message": "22 min ride — within your free 30 minutes!"
}
```

### Get Nearby Stations
```http
GET /api/v1/stations?lat=37.7749&lng=-122.4194&radius=1000&limit=10
Response: 200 OK
{
  "stations": [
    {
      "station_id": "stn-42",
      "name": "Market St & 3rd St",
      "lat": 37.7755, "lng": -122.4189,
      "distance_meters": 80,
      "total_docks": 20,
      "bikes_available": 12,
      "docks_available": 8,
      "ebikes_available": 3,
      "last_reported": "2025-03-14T08:29:55Z"
    }
  ]
}
```

### Reserve a Bike (Hold for 5 Minutes)
```http
POST /api/v1/reservations
{
  "station_id": "stn-42",
  "bike_type": "classic"
}
Response: 200 OK
{
  "reservation_id": "res-uuid",
  "station_id": "stn-42",
  "bike_id": "B-123",
  "expires_at": "2025-03-14T08:35:00Z"
}
```

### Station Status (Real-time)
```http
GET /api/v1/stations/{station_id}
WS: wss://bikeshare.example.com/ws/stations/{station_id}
→ Push: { "bikes_available": 11, "docks_available": 9, "ebikes": 2 }
```

---

## 6. Data Model

### PostgreSQL — Core Entities

```sql
-- Stations
CREATE TABLE stations (
    station_id      UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    lat             DECIMAL(10,7) NOT NULL,
    lng             DECIMAL(10,7) NOT NULL,
    total_docks     SMALLINT NOT NULL,
    address         TEXT,
    region          VARCHAR(50),
    status          ENUM('active', 'maintenance', 'closed') DEFAULT 'active',
    installed_at    DATE,
    geometry        GEOMETRY(Point, 4326),
    SPATIAL INDEX idx_location (geometry)
);

-- Bikes
CREATE TABLE bikes (
    bike_id         UUID PRIMARY KEY,
    bike_type       ENUM('classic', 'ebike') NOT NULL,
    status          ENUM('available', 'rented', 'maintenance', 'retired') DEFAULT 'available',
    current_station UUID REFERENCES stations(station_id),
    current_dock    SMALLINT,
    battery_level   DECIMAL(3,2),  -- for e-bikes (0.00-1.00)
    last_maintenance DATE,
    total_trips     INT DEFAULT 0,
    total_km        DECIMAL(10,2) DEFAULT 0,
    created_at      DATE
);

-- Trips
CREATE TABLE trips (
    trip_id           UUID PRIMARY KEY,
    user_id           UUID NOT NULL,
    bike_id           UUID NOT NULL,
    start_station_id  UUID NOT NULL,
    end_station_id    UUID,
    started_at        TIMESTAMPTZ NOT NULL,
    ended_at          TIMESTAMPTZ,
    duration_minutes  DECIMAL(8,2),
    distance_km       DECIMAL(8,2),
    fare_cents        INT,
    pricing_plan      VARCHAR(20),
    status            ENUM('active', 'completed', 'cancelled') DEFAULT 'active',
    route_polyline    TEXT,         -- encoded polyline from GPS trail
    INDEX idx_user (user_id, started_at DESC),
    INDEX idx_bike (bike_id, started_at DESC),
    INDEX idx_active (status) WHERE status = 'active',
    INDEX idx_station_time (start_station_id, started_at)
);

-- Users / Members
CREATE TABLE users (
    user_id         UUID PRIMARY KEY,
    email           VARCHAR(255) UNIQUE,
    name            VARCHAR(100),
    membership_type ENUM('annual', 'monthly', 'day_pass', 'none'),
    membership_expires DATE,
    payment_method_id VARCHAR(64),
    balance_cents   INT DEFAULT 0,
    status          ENUM('active', 'suspended', 'banned') DEFAULT 'active',
    created_at      TIMESTAMP
);

-- Reservations
CREATE TABLE reservations (
    reservation_id  UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    station_id      UUID NOT NULL,
    bike_id         UUID,
    expires_at      TIMESTAMPTZ NOT NULL,
    status          ENUM('active', 'used', 'expired', 'cancelled') DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_station_active (station_id, status) WHERE status = 'active'
);
```

### Redis — Real-time Station + Bike State

```
# Station real-time availability
station:{station_id}:bikes_available    → INT
station:{station_id}:docks_available    → INT
station:{station_id}:ebikes_available   → INT
station:{station_id}:last_updated       → TIMESTAMP

# Bike lock (for checkout race condition prevention)
bike_lock:{bike_id}  → user_id
TTL: 30 seconds

# Reservation hold
reservation:{station_id}:{bike_id}  → reservation_id
TTL: 300 seconds (5 min reservation window)

# Active ride per user (enforce one-ride-at-a-time)
active_ride:{user_id}  → trip_id
TTL: 86400 (24 hours max ride)

# E-bike battery levels
ebike_battery:{bike_id}  → FLOAT (0.0-1.0)
TTL: 120

# Station set for geo queries
stations:geo  → Sorted Set (GEOADD for nearby station queries)
```

### Kafka Topics

```
Topic: trip-events           (trip started, completed, cancelled)
Topic: station-changes       (dock count changes → consumed by app for real-time display)
Topic: bike-telemetry        (e-bike GPS, battery, diagnostics)
Topic: rebalancing-alerts    (station imbalance notifications)
```

### ClickHouse — Trip Analytics

```sql
CREATE TABLE trip_analytics (
    trip_id         UUID,
    user_id         UUID,
    bike_id         UUID,
    bike_type       Enum8('classic'=0, 'ebike'=1),
    start_station   UUID,
    end_station     UUID,
    started_at      DateTime,
    ended_at        DateTime,
    duration_min    Float32,
    distance_km     Float32,
    fare_cents      UInt32,
    day_of_week     UInt8,
    hour_of_day     UInt8,
    trip_date       Date MATERIALIZED toDate(started_at)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(started_at)
ORDER BY (start_station, started_at);

-- "Busiest stations this month"
-- "Average trip duration by hour of day"
-- "Revenue per station"
-- "Most common origin-destination pairs (for rebalancing)"
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Race condition on bike checkout** | Redis SET NX (atomic lock); only one user can lock a bike |
| **Dock controller offline** | Queue unlock command; retry when dock comes online; manual override at station kiosk |
| **Redis failure** | Fall back to PostgreSQL for availability counts (slower but correct); Redis Cluster for HA |
| **Payment failure** | Start ride anyway (pre-authorized card); charge asynchronously post-ride |
| **Trip not ended (bike abandoned)** | Auto-end trip after 24 hours; charge max fare; flag bike for ops |
| **Network outage at station** | Station kiosk has local unlock capability (cached member list); sync when online |
| **Duplicate trip start** | active_ride:{user_id} prevents multiple concurrent rides |

### Specific: Dock Sensor Failure — "Ghost Bikes"

```
Problem: Dock sensor fails. System thinks bike is docked (available), but dock is actually empty.
  User walks to station, sees "5 bikes available", but finds 0 physical bikes.

Detection:
  1. User reports "no bike at dock X" via app
  2. If 3 users report within 10 minutes → auto-mark dock as "sensor_fault"
  3. Periodic reconciliation: e-bikes report GPS location → if GPS says bike is 5 km away 
     but system says "docked at station X" → sensor fault detected
  
Mitigation:
  1. Show "reported" count: "5 bikes available (3 reported missing)"
  2. Under-report: Show "2-5 bikes available" instead of exact count
  3. Confidence score: stations with recent sensor faults get lower confidence
  4. Ops alert: dispatch technician to repair sensor

Opposite problem: "Phantom docks"
  Dock sensor says dock is full (bike present), but actually empty.
  User arrives with bike to return, but all "available" docks rejected.
  Solution: Kiosk override + maintenance alert
```

### Specific: Handling "Ride in Progress" Server Restart

```
Scenario: 25,000 active rides. Trip Service restarts.

State recovery:
  1. Active rides are in PostgreSQL: SELECT * FROM trips WHERE status = 'active'
  2. Redis active_ride:{user_id} rebuilt from DB on startup
  3. Ongoing rides continue ��� bikes don't need server to function (they're physically unlocked)
  4. Trip end (dock event) processed normally after restart

Risk: If both PostgreSQL and Redis lose data:
  Trip records lost → users not charged → bikes "stuck" in rented state
  
  Safety net:
  - PostgreSQL synchronous replication (zero data loss)
  - Daily reconciliation: physical dock inventory vs system records
  - E-bikes have GPS → can locate all bikes regardless of system state
```

---

## 8. Deep Dive: Engineering Trade-offs

### Reservation System: Should You Allow Reservations?

```
Pros:
  ✓ User walks to station knowing a bike is waiting
  ✓ Reduces frustration of "arrived but no bikes"
  ✓ Higher conversion: users commit to riding

Cons:
  ✗ Reduces effective supply: reserved bike can't be used by walk-up user
  ✗ No-shows: user reserves but doesn't show up → bike wasted for 5 minutes
  ✗ Complexity: reservation expires → need to release bike back
  ✗ Fairness: heavy app users advantage over casual/tourist users

Citi Bike's approach: NO reservations for regular bikes
  Reasoning: demand is high; reservations would reduce availability for others
  Exception: e-bikes can be reserved (premium feature for members)

If implementing:
  - Max 1 reservation per user at a time
  - 5-minute window (short to minimize waste)
  - If user doesn't pick up → reservation expires, bike released
  - Limit reservations to stations with ≥ 5 bikes (don't reserve the last bike)
  - Show reserved vs available counts separately
```

### Why PostgreSQL (Not DynamoDB or Cassandra) for Trips?

```
Trip data characteristics:
  1. ACID required: checkout = lock bike + start trip + update station (multi-table atomic)
  2. Billing queries: sum fares by user, date range, membership type
  3. Rebalancing analytics: COUNT trips per station pair per hour (JOINs + GROUP BY)
  4. Moderate scale: 500K trips/day = ~6 writes/sec (trivial for PostgreSQL)

PostgreSQL ✓:
  - ACID transactions for checkout flow
  - Rich analytics queries (JOINs, aggregations, window functions)
  - PostGIS for station geo queries
  - ~6 writes/sec is easily handled by single primary
  - Read replicas for dashboard/analytics queries
  
DynamoDB ✗:
  - 25-item transaction limit (fine for checkout but limits future complexity)
  - No JOINs → complex analytics need application-side joining
  - Cost: pay per read/write capacity (predictable but adds up)

Cassandra ✗:
  - Overkill for this scale (500K/day, not 500M/day)
  - No transactions → can't atomic checkout
  - No JOINs for analytics

At Citi Bike scale (50K bikes, 500K daily trips):
  PostgreSQL is the right choice. Not everything needs NoSQL.
  
If scaling to 50M bikes (global bike-sharing):
  Shard PostgreSQL by region (NYC shard, London shard, etc.)
  Or migrate trips to DynamoDB with analytics export to ClickHouse
```

### Physical Lock Mechanism: How Bike Unlocking Actually Works

```
The software system must interface with physical hardware. This is the most
failure-prone part of the system.

Communication chain:
  App → API Server → MQTT Broker → Station Controller → Dock Lock Motor

Station Controller:
  - Industrial-grade embedded computer at each station
  - Connected to internet via cellular (4G/5G) modem
  - Connected to each dock via RS-485 serial bus
  - Runs local firmware that can operate independently

Dock Lock:
  - Electromechanical solenoid lock
  - Sensor: detects bike present/absent (IR sensor + physical lever)
  - Lock motor: 12V DC, activated by controller command
  - Lock states: LOCKED (bike secured), UNLOCKED (bike free to pull out)

Unlock sequence:
  1. Controller receives MQTT command: { "unlock": true, "dock": 7 }
  2. Controller activates solenoid on dock 7 (200ms pulse)
  3. Lock disengages → bike is free to pull
  4. User has 30 seconds to pull bike from dock
  5. Sensor detects bike removed → controller reports "dock_empty"
  6. If user doesn't pull bike in 30 seconds → lock re-engages

Failure modes:
  a. Solenoid stuck: mechanical failure → bike can't be released
     → Show "This dock is unavailable, try another dock"
     → Alert maintenance
  
  b. Cellular outage at station: controller can't reach server
     → Fallback: member card stored locally on controller
     → Controller authorizes unlock from local member database
     → Syncs trip data when connectivity restored
  
  c. Power outage: entire station loses power
     → Battery backup (4-hour runtime) keeps controller + locks operational
     → Solar panels on station for trickle charging
     → Fail-secure: bikes remain locked (prevent theft)

This is why 99.9% (not 99.99%) availability:
  Physical hardware has higher failure rates than pure software systems.
  A stuck solenoid can't be fixed with a code deploy.
```

### Demand Prediction for Rebalancing: Features and Model

```
Input features for demand prediction model:
  1. Temporal: hour_of_day, day_of_week, month, is_holiday, is_weekend
  2. Weather: temperature, precipitation, wind_speed, feels_like_temp
  3. Events: nearby event at stadium/convention center (API integration)
  4. Historical: avg demand at this station at this time over last 4 weeks
  5. Spatial: subway station proximity, office density, residential density
  6. Trend: demand trend over last 3 hours (rising/falling)

Output: predicted trips starting FROM and ending AT each station, per hour

Model: XGBoost or LSTM
  XGBoost: simple, interpretable, works well with tabular features
  LSTM: captures sequential patterns (demand at 8 AM predicts demand at 9 AM)

Rebalancing optimization:
  Given predictions, solve assignment problem:
    surplus_stations = [stations where predicted_supply > predicted_demand]
    deficit_stations = [stations where predicted_demand > predicted_supply]
    trucks = [available rebalancing trucks with capacity 20 bikes]
    
    Minimize: total_truck_distance
    Subject to: each deficit station receives enough bikes
    
    This is a Capacitated Vehicle Routing Problem (CVRP)
    Solver: Google OR-Tools or custom branch-and-bound

Proactive vs reactive:
  Reactive: "Station 42 has 1 bike left → send truck now"
    Problem: by the time truck arrives, station is empty for 30 min
  
  Proactive ⭐: "Station 42 will have 1 bike left in 90 minutes → schedule truck now"
    Truck arrives before station empties → no user frustration
    Requires accurate demand prediction (±2 bikes per hour)
```



