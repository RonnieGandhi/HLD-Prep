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

