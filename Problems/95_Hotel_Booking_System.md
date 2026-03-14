# 95. Design a Hotel Booking System

---

## 1. Functional Requirements (FR)

- Search hotels by location, dates, guests, price range, amenities, star rating
- View room types, photos, reviews, availability calendar
- Book room(s): select dates → reserve → pay → confirm
- Overbooking management: controlled overbooking with walking policy
- Cancellation with policy enforcement (free cancel before X days)
- Price management: dynamic pricing based on demand, season, events
- Loyalty program: points accrual and redemption
- Multi-property management for hotel chains
- Calendar-based inventory (each room-night is a separate unit)

---

## 2. Non-Functional Requirements (NFRs)

- **Consistency**: No double-booking of the same room-night (strong consistency on booking)
- **High Availability**: 99.99% — booking failures = lost revenue
- **Low Latency**: Search < 200ms, booking < 2s
- **Scalability**: 1M+ hotels, 100M+ room-nights of inventory

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Hotels | 1M |
| Rooms (total) | 50M |
| Bookable room-nights (next 365 days) | 50M × 365 = 18.25B |
| Searches / sec | 50K |
| Bookings / sec | 5K |
| Peak (holiday season) | 5× normal |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                        │
│  Web / Mobile App / OTA Partners (Booking.com, Expedia APIs)          │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│                    API GATEWAY                                         │
│  Rate limiting, JWT auth, partner API key validation                  │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┬─────────────────┐
         │                  │                  │                 │
┌────────▼──────┐  ┌────────▼──────┐  ┌────────▼──────┐ ┌───────▼──────┐
│ Search        │  │ Booking       │  │ Pricing       │ │ Notification │
│ Service       │  │ Service       │  │ Service       │ │ Service      │
│               │  │               │  │               │ │              │
│ - Geo+date    │  │ - Reserve     │  │ - Dynamic     │ │ - Confirm    │
│   queries     │  │ - Payment     │  │   pricing     │ │   emails     │
│ - Filters     │  │ - Confirm     │  │ - Demand/     │ │ - Cancel     │
│ - Sort/rank   │  │ - Cancel      │  │   season adj  │ │   alerts     │
│ - Availability│  │ - Race cond   │  │ - Rate parity │ │ - Reminders  │
│   check       │  │   prevention  │  │   enforcement │ │              │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘ └──────────────┘
        │                  │                  │
┌───────▼──────┐  ┌────────▼──────────────────▼──────────────────────┐
│Elasticsearch │  │  PostgreSQL (Primary DB)                         │
│              │  │                                                   │
│ - Hotel      │  │  - room_inventory (calendar-based, per room-night│
│   profiles   │  │  - reservations (booking records)                │
│ - Geo index  │  │  - hotels, room_types (master data)              │
│ - Amenities  │  │  - cancellation_policies                        │
│ - Reviews    │  │                                                   │
│ - Facets     │  │  SELECT ... FOR UPDATE for race condition prevent│
└──────────────┘  └──────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Redis        │  │ Kafka        │  │ Payment Svc  │
│              │  │              │  │ (Stripe/PSP) │
│ - Inventory  │  │ - Booking    │  │              │
│   cache      │  │   events     │  │ - Charge     │
│ - Price      │  │ - Search     │  │ - Refund     │
│   cache      │  │   analytics  │  │ - Webhook    │
│ - Session    │  │ - Pricing    │  │              │
│              │  │   updates    │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Search Service + Elasticsearch
- **Why Elasticsearch?** Geo queries (bounding box search by city/coordinates), full-text (hotel name, description), faceted filtering (star rating, amenities, price range), relevance scoring
- **Index design**: One document per hotel with nested room types; filter by date range availability via a separate availability check service
- **Search flow**: ES returns matching hotels → Availability Service validates room-night inventory for each → Pricing Service computes price → Return sorted results
- **Cache**: Redis caches popular searches ("NYC, next weekend, 2 guests") for 5 minutes

#### Booking Service (Critical Path)
- **Why PostgreSQL?** ACID transactions for inventory management. `SELECT FOR UPDATE` prevents double-booking race condition
- **Hold-and-confirm pattern**: Reserve inventory (status=PENDING, 15-min TTL) → charge payment → confirm (status=CONFIRMED). If payment fails → auto-release after 15 minutes via scheduled job
- **Multi-night atomicity**: Must lock ALL room-night rows in date range → if any night unavailable, entire booking fails

#### Pricing Service
- **Dynamic pricing**: Base price × demand_multiplier × season_factor × day_of_week_factor
- **Demand signals**: Booking velocity, search volume, competitor rates, inventory remaining percentage
- **Rate parity**: Contractually, direct booking price must match OTA price → centralized pricing source

### Search Flow

```
1. User searches "NYC, Dec 20-25, 2 guests"
2. Search Service queries Elasticsearch (geo + filters)
3. For top 50 results: check room_inventory in Redis cache
   (fallback to PostgreSQL for cache miss)
4. Pricing Service computes dynamic price per hotel per night
5. Return sorted results (by price, rating, or relevance)
```

### Booking Flow

```
1. User selects hotel + room type + dates
2. Booking Service: BEGIN PostgreSQL transaction
3. SELECT ... FOR UPDATE on room-night inventory rows
4. Check availability for ALL dates in range
5. Decrement booked_rooms for each room-night
6. Create reservation (status=PENDING)
7. COMMIT
8. Payment Service charges card
9. Update reservation status=CONFIRMED
10. Notification Service sends confirmation email
```

### Calendar-Based Inventory Model

```
Unlike ticketing (#23) where each seat is unique:
  Hotels have POOLED inventory: "5 Deluxe King rooms available on Dec 20"
  
Room-Night Inventory:
  hotel_id + room_type + date → available_count

  Hotel ABC, Deluxe King:
  Dec 20: 5 available (of 10 total)
  Dec 21: 3 available
  Dec 22: 0 available ← SOLD OUT
  Dec 23: 7 available
  
Booking "Dec 20-23" requires ALL 4 nights to have availability
  → Atomic decrement across all 4 dates in one transaction
```

### Overbooking Strategy

```
Airlines/Hotels intentionally overbook by 5-10% (data-driven):
  10 rooms, sell 11 reservations
  Historical no-show rate: 15% → expect 1-2 no-shows
  If all 11 show up → "walk" lowest-priority guest to partner hotel

Implementation:
  available_count can go negative (to overbooking limit)
  overbooking_limit = total_rooms × overbooking_factor (e.g., 1.1)
  
  Decision: ML model predicts no-show probability per booking
  - Business traveler, booked recently: low no-show risk
  - Leisure, booked 6 months ago, no prepayment: high no-show risk
```

---

## 5. APIs

```
GET  /api/hotels/search?location=NYC&checkin=2026-12-20&checkout=2026-12-25&guests=2
POST /api/bookings          → {hotel_id, room_type, checkin, checkout, guest_info, payment}
GET  /api/bookings/{id}     → Booking details
DELETE /api/bookings/{id}   → Cancel (applies cancellation policy)
GET  /api/hotels/{id}/availability?start=...&end=...  → Room availability calendar
```

---

## 6. Data Models

### PostgreSQL

```sql
CREATE TABLE room_inventory (
    hotel_id     UUID, room_type TEXT, date DATE,
    total_rooms  INT, booked_rooms INT, overbooking_limit INT,
    price_cents  INT,  -- dynamic price for this room-night
    PRIMARY KEY (hotel_id, room_type, date)
);

CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY,
    hotel_id UUID, room_type TEXT,
    guest_id UUID, checkin DATE, checkout DATE,
    status TEXT,  -- PENDING|CONFIRMED|CANCELLED|CHECKED_IN|COMPLETED|NO_SHOW
    total_price_cents INT, payment_id UUID,
    cancellation_policy TEXT,
    created_at TIMESTAMPTZ
);
```

### Booking Transaction (Race Condition Prevention)

```sql
BEGIN;
-- Lock all room-night rows for the date range
SELECT booked_rooms, total_rooms, overbooking_limit
FROM room_inventory
WHERE hotel_id = $1 AND room_type = $2 AND date BETWEEN $3 AND $4
FOR UPDATE;

-- Check ALL dates have availability
-- If any date has booked_rooms >= overbooking_limit → ROLLBACK

UPDATE room_inventory SET booked_rooms = booked_rooms + 1
WHERE hotel_id = $1 AND room_type = $2 AND date BETWEEN $3 AND $4;

INSERT INTO reservations (...) VALUES (...);
COMMIT;
```

---

## 7. Fault Tolerance

- **Payment failure after inventory reserved**: Hold for 15 min → auto-release if unpaid
- **Double-booking prevention**: `SELECT FOR UPDATE` + transaction isolation
- **Search-to-book staleness**: Availability shown in search may be stale by seconds → final check at booking time
- **Rate parity**: price must be consistent across OTAs (contractual) → centralized pricing service

---

## 8. Key Differences from Ticketing System (#23)

```
Ticketing: Each seat is unique → seat-level locking
Hotel: Pooled inventory → count-based → simpler locking
Ticketing: One event, one time → no date-range complexity
Hotel: Multi-night stays → must lock ALL dates atomically
Ticketing: No overbooking (each seat is physical)
Hotel: Controlled overbooking is standard industry practice
```

---

## 9. Deep Dive: Engineering Trade-offs

### Scalability: Sharding Strategy

```
1M hotels, 50M rooms, 18.25B room-nights → need horizontal scaling

Shard by hotel_id:
  room_inventory: Shard key = hotel_id
    All room-nights for a hotel co-located on one shard
    Booking transaction: hits ONE shard (single-shard ACID transaction)
    No distributed transactions needed for a single booking
  
  1M hotels / 16 shards = ~62,500 hotels per shard
  Each shard: ~1.14B room-night rows → manageable with PostgreSQL partitioning
  
  Within each shard: partition room_inventory by date range
    PARTITION BY RANGE (date)
    Each partition = 1 month of data
    Query "Dec 20-25" hits 1 partition (fast) not scanning full year

Search (cross-shard):
  "Find available hotels in NYC for Dec 20-25"
  
  Step 1: Elasticsearch returns matching hotels (geo + filters) — not sharded by hotel
  Step 2: For top 50 results, check availability:
    Group hotels by shard → fan-out availability queries (at most 16 parallel queries)
    Each shard: SELECT for those hotel_ids + date range → return availability
    Merge results → return to client with prices
  
  Optimization: Redis cache per hotel with availability bitmap
    Key: avail:{hotel_id}:{room_type}:{month}
    Value: bitmask (1 bit per day, 1=available, 0=booked)
    Lookup: bitwise AND across date range → instant availability check
    Invalidation: on booking/cancellation, update bitmask in Redis
    Cache hit: skip DB query entirely for search results

Reservation table: shard by reservation_id (or hotel_id for co-location)
  Guest lookups ("my reservations"): secondary index on guest_id
  Or: separate guest_reservations table sharded by guest_id (CQRS pattern)
```

### Race Condition Deep Dive: Isolation Levels

```
The booking transaction needs to prevent two users from simultaneously
booking the last room for the same night.

READ COMMITTED (PostgreSQL default):
  Transaction A: SELECT booked_rooms WHERE hotel=X, date='Dec 22' FOR UPDATE
    → sees booked_rooms = 9 (of 10 total) → 1 available
  Transaction B: also tries SELECT FOR UPDATE on SAME row
    → BLOCKED by A's row lock → waits
  Transaction A: UPDATE booked_rooms = 10, COMMIT
  Transaction B: unblocked, re-reads → sees booked_rooms = 10 → no availability → ROLLBACK
  
  ✓ Correct! FOR UPDATE provides row-level exclusive lock
  ✓ No double-booking possible for SAME rows

  BUT: What about phantom reads in date range?
  Transaction A: SELECT ... WHERE date BETWEEN 'Dec 20' AND 'Dec 25' FOR UPDATE
    → Locks rows for Dec 20-25
  Transaction B: INSERT a new inventory row for Dec 23 (admin adds rooms)
    → READ COMMITTED allows this INSERT (no gap lock)
    → Transaction A doesn't see the new row → stale data
  
  In practice: this is fine for hotel booking because:
    room_inventory rows are pre-created for all dates (365 days ahead)
    No INSERTs during booking — only UPDATEs to existing rows
    FOR UPDATE on existing rows is sufficient

SERIALIZABLE (strongest):
  Prevents ALL anomalies including phantoms
  Uses predicate locks (lock the RANGE, not just existing rows)
  
  ✓ Bulletproof correctness
  ✗ Higher abort rate under contention (serialization failures)
  ✗ Application must retry on serialization failure
  
  When to use: if room_inventory rows are NOT pre-created and
  the system dynamically inserts inventory → SERIALIZABLE prevents phantoms

Recommendation for hotel booking:
  Pre-create room_inventory rows for 365 days → use READ COMMITTED + FOR UPDATE
  The CHECK constraint (booked_rooms <= overbooking_limit) provides an
  additional database-level safety net even if application logic has a bug
```

### Pooled Inventory vs Named-Room Assignment

```
Pooled (this design):
  "5 Deluxe King rooms available on Dec 20" — any of the 5 rooms
  Booking decrements a counter, specific room assigned at check-in
  ✓ Simpler booking logic (count-based)
  ✓ Flexible room assignment (maintenance, upgrades, requests)
  ✗ Can't guarantee "same room for 3-night stay"
  ✗ Can't sell "room with ocean view" as distinct inventory

Named-Room Assignment:
  Room 401, Room 402, ... each tracked individually
  Booking locks a SPECIFIC room for the date range
  ✓ Guest preferences ("I want room 401 again")
  ✓ Distinct pricing for view/floor (Room 401 = $200, Room 405 = $250)
  ✗ More complex locking (must lock specific room across dates)
  ✗ Fragmentation: Room 401 free Dec 20-21, Room 402 free Dec 22-23
     → Neither can serve a Dec 20-23 booking even though hotel has availability

  Industry practice: Pooled inventory for booking, named assignment at check-in
    Exception: luxury hotels sell specific rooms (suites, penthouses)
```

### Eager Payment vs Lazy Payment

```
Eager (charge at booking):
  ✓ Revenue certainty — money collected immediately
  ✓ Reduces no-shows (financial commitment)
  ✗ Higher cancellation friction → lost bookings
  ✗ Refund processing cost for cancellations
  Best for: Non-refundable rates, flash sales, peak season

Lazy (charge at check-in or post-stay):
  ✓ Lower booking friction → higher conversion
  ✓ No refund processing for cancellations
  ✗ No-show risk (guest never shows, never pays)
  ✗ Card might be declined at check-in
  Best for: Flexible rates, business travel, loyalty members

Hybrid (industry standard):
  Authorization hold at booking ($1 or full amount) → validates card
  No-show fee: charge 1 night if guest doesn't cancel by deadline
  Capture: charge on check-in or check-out (depending on hotel policy)
  Non-refundable rate: charge immediately (lower price, no cancellation)
```

### Search Staleness vs Real-Time Availability

```
Problem: Search shows "5 rooms available" → user clicks → booking fails
  (someone else booked during the 30 seconds between search and click)

Option 1: Real-time availability on every search result
  For each of top 50 hotels: query room_inventory DB
  ✗ 50K searches/sec × 50 hotels = 2.5M DB queries/sec → DB dies

Option 2: Cached availability with staleness ⭐
  Redis bitmap per hotel, refreshed every 30 seconds
  Search uses cached availability — may be stale by up to 30s
  Final check at booking time (authoritative DB query with FOR UPDATE)
  If booking fails due to stale cache → show "Sorry, this room just sold out"
  Acceptable UX: < 1% of booking attempts fail due to staleness
  
  ✓ DB handles only actual booking attempts (5K/sec not 2.5M/sec)
  ✗ Occasional "phantom availability" shown to users

Option 3: Pessimistic display (show fewer rooms than available)
  Cache shows "available" only if real availability > 2
  The last 1-2 rooms are hidden from search, only shown on direct hotel page
  ✓ Fewer failed booking attempts
  ✗ Might undercount availability → lost revenue
```

