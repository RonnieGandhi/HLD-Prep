# 23. Design a Ticketing System (BookMyShow / TicketMaster)

---

## 1. Functional Requirements (FR)

- **Browse events**: Search and browse movies, concerts, sports events by location, date, genre
- **View available seats**: Display seat map with real-time availability (available, reserved, booked)
- **Select seats**: User selects specific seats (for assigned seating) or quantity (for general admission)
- **Temporary hold**: Selected seats are temporarily held (5-10 minutes) while user completes checkout
- **Book & Pay**: Complete booking with payment
- **Booking confirmation**: E-ticket (QR code) via email/app
- **Cancel/Refund**: Cancel booking and process refund based on cancellation policy
- **Waiting list**: If show is sold out, join a waiting list

---

## 2. Non-Functional Requirements (NFRs)

- **Strong Consistency**: Two users must NOT book the same seat (the #1 constraint)
- **High Availability**: 99.99%
- **Low Latency**: Seat availability loads in < 200 ms
- **Handle Traffic Spikes**: Popular events (Taylor Swift, Avengers premiere) → 1M+ users hitting the system simultaneously
- **Fair Booking**: First-come-first-served; prevent bots from grabbing all tickets
- **Idempotent**: No double-booking, no double-charging
- **Scalability**: 1000+ concurrent events, 10M+ bookings/day during peak

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active events at any time | 50K |
| Seats per event (avg) | 500 |
| Total bookable seats | 25M |
| Bookings / day | 5M |
| Peak bookings / sec (popular event on-sale) | 50K |
| Seat checks / sec (availability) | 500K |
| Temporary holds active at peak | 1M |

---

## 4. High-Level Design (HLD)

```
┌──────────────────┐
│      Client      │
│ (Browser / App)  │
└────────┬─────────┘
         │
┌────────▼─────────┐
│   CDN (Static)   │  ← Seat map SVGs, event posters, JS/CSS
└────────┬─────────┘
         │
┌────────▼─────────┐
│   API Gateway    │  ← Rate limiting, JWT auth, bot detection, virtual queue gate
└────────┬─────────┘
         │
         ├───────────────┬──────────────────┬──────────────────┬───────────────────┐
         │               │                  │                  │                   │
┌────────▼────────┐ ┌────▼──────────┐ ┌─────▼──────────┐ ┌────▼──────────┐ ┌──────▼──────┐
│ Event / Search  │ │   Booking     │ │   Payment      │ │   Virtual     │ │ Notification│
│ Service         │ │   Service     │ │   Service      │ │   Queue Svc   │ │ Service     │
│                 │ │ (Hold + Book) │ │                │ │               │ │(Email/SMS/  │
│                 │ │               │ │                │ │               │ │ Push)       │
└────────┬────────┘ └────┬──┬──────┘ └────┬───────────┘ └────┬──────────┘ └─────────────┘
         │               │  │              │                  │
┌────────▼────────┐ ┌────▼──▼──────┐ ┌────▼───────────┐ ┌────▼──────────┐
│ Elasticsearch   │ │   Redis      │ │   Payment      │ │   Redis       │
│ (Event search   │ │ Cluster      │ │   Gateway      │ │ (Queue ZSET)  │
│  index)         │ │ (Seat locks, │ │ (Stripe/       │ └───────────────┘
└─────────────────┘ │  avail cache,│ │  RazorPay)     │
                    │  hold timers)│ └────────────────┘
┌─────────────────┐ └──────┬──────┘
│ MySQL           │        │
│ (Events, Venues,│ ┌──────▼──────┐
│  Show Catalog)  │ │ PostgreSQL  │  ← Source of truth for bookings & seat status
└─────────────────┘ │ (Bookings,  │
                    │  Seats,     │
                    │  Payments,  │
                    │  Audit Log) │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Kafka    │  ← booking-events, payment-events, notification-events
                    └──────┬──────┘
                           │
           ┌───────────────┼────────────────┐
           │               │                │
    ┌──────▼──────┐ ┌──────▼──────┐  ┌──────▼──────┐
    │ Hold Expiry │ │ Analytics   │  │ Waitlist    ��
    │ Worker      │ │ Consumer    │  │ Consumer    │
    │ (cleanup)   │ │(ClickHouse) │  │             │
    └─────────────┘ └─────────────┘  └─────────────┘

┌─────────────────────┐
│ Seat Map Service    │  ← WebSocket / SSE for real-time seat status push
│ (Real-time updates) │
└─────────────────────┘
```

### Complete Booking Flow — Step-by-Step Sequence

```
User                 API GW           VirtualQ         BookingSvc       Redis            PostgreSQL       PaymentSvc       NotifSvc
 │                     │                 │                │               │                  │                │               │
 │─── Browse Event ───▶│                 │                │               │                  │                │               │
 │◀── Event Details ───│                 │                │               │                  │                │               │
 │                     │                 │                │               │                  │                │               │
 │─── "Book Tickets" ─▶│── isHotEvent? ─▶│                │               │                  │                │               │
 │                     │                 │── enqueue ────▶│(Redis ZADD)   │                  │                │               │
 │◀── "Queue pos #42" ─│◀────────────────│                │               │                  │                │               │
 │     ...wait...      │                 │                │               │                  │                │               │
 │◀── "Your turn!" ────│◀─ issue JWT ────│                │               │                  │                │               │
 │                     │                 │                │               │                  │                │               │
 │─── GET /seats ─────▶│────────────────────────────────▶│── SMEMBERS ──▶│                  │                │               │
 │◀── seat map ────────│◀────────────────────────────────│◀─ avail set ──│                  │                │               │
 │                     │                 │                │               │                  │                │               │
 │─── POST /hold ─────▶│─── validate JWT ──────────────▶│               │                  │                │               │
 │                     │                 │                │── SET NX EX ─▶│                  │                │               │
 │                     │                 │                │◀── OK ────────│                  │                │               │
 │                     │                 │                │── UPDATE seat status ──────────▶│                │               │
 │◀── hold confirmed ──│◀────────────────────────────────│               │                  │                │               │
 │                     │                 │                │               │                  │                │               │
 │─── POST /confirm ──▶│────────────────────────────────▶│               │                  │                │               │
 │                     │                 │                │── verify hold──▶               │                │               │
 │                     │                 │                │────────────────── auth payment ─────────────────▶│               │
 │                     │                 │                │◀─────────────────── payment OK ──────────────────│               │
 │                     │                 │                │── BEGIN TXN ───────────────────▶│                │               │
 │                     │                 │                │   UPDATE seat status='booked'  │                │               │
 │                     │                 │                │   INSERT booking               │                │               │
 │                     │                 │                │   INSERT audit_log             │                │               │
 │                     │                 │                │── COMMIT ──────────────────────▶│                │               │
 │                     │                 │                │── DEL redis lock ──────────────▶│                │               │
 │                     │                 │                │── publish Kafka ───────────────────────────────────────────────▶│
 │◀── booking + tickets│◀────────────────────────────────│               │                  │                │               │
 │                     │                 │                │               │                  │                │       email QR│
 │◀──── e-ticket email ───────────────────────────────────────────────────────────────────────────────────────────────────────│
```

### Component Deep Dive

#### API Gateway — First Line of Defense
- **Bot detection**: Fingerprint client (TLS fingerprint, JS challenge, device fingerprint). Block known headless browsers (Puppeteer, Selenium)
- **Rate limiting**: Token bucket per IP (100 req/min), per user (50 req/min for booking endpoints)
- **Virtual queue gate**: For hot events, redirect all booking attempts through Virtual Queue Service instead of directly to Booking Service
- **JWT validation**: Verify booking token issued by Virtual Queue (contains event_id, user_id, expiry)
- **Technology**: Kong / AWS API Gateway / Envoy

#### Event / Search Service
- **Catalog management**: CRUD for events, venues, shows, pricing tiers
- **Search**: Elasticsearch index on event title, category, city, venue, artist name, tags
  - Filters: category, city, date range, price range, availability
  - Sort: relevance, date, popularity, price
  - Autocomplete on event/artist name
- **Caching**: Redis caches hot event metadata (poster, description, show times) with TTL=5min
- **Why MySQL for catalog**: Relational data with clear schema (event → shows → venue → sections → seats). Strong consistency for admin operations. Read replicas for search traffic

#### Booking Service — The Heart of the System

The booking service manages the **complete seat lifecycle**:

```
SEAT STATES:
  ┌───────────┐    hold acquired     ┌───────────┐    payment OK       ┌───────────┐
  │ AVAILABLE │ ──────────────────▶  │   HELD    │ ──────────────────▶ │  BOOKED   │
  └───────────┘                      └─────┬─────┘                     └─────┬─────┘
       ▲                                   │                                 │
       │              hold expired /       │                          cancel/refund
       │              payment failed       │                                 │
       └───────────────────────────────────┘                                 │
       ▲                                                                     │
       └─────────────────────────────────────────────────────────────────────┘
```

**Hold Acquisition — Multi-Seat Atomic Hold (Lua Script)**:
When a user selects multiple seats (e.g., 4 tickets), ALL must be held atomically. If any one fails, none should be held:

```lua
-- Redis Lua script: atomic multi-seat hold
-- KEYS = list of seat lock keys
-- ARGV[1] = user_id, ARGV[2] = TTL in seconds

-- Phase 1: Check all seats are available
for i, key in ipairs(KEYS) do
    if redis.call('EXISTS', key) == 1 then
        return {0, key}  -- FAIL: seat already held, return which seat
    end
end

-- Phase 2: All available → lock all atomically
for i, key in ipairs(KEYS) do
    redis.call('SET', key, ARGV[1], 'EX', ARGV[2])
end

return {1, 'OK'}  -- SUCCESS: all seats held
```

**Why Lua script?** Redis executes Lua scripts atomically (single-threaded). No other command can interleave between the EXISTS checks and SET commands. This guarantees either ALL seats are held or NONE are — critical for group bookings.

**Booking Confirmation — Database Transaction**:
```sql
BEGIN TRANSACTION;
  -- 1. Verify hold is still valid and belongs to this user
  SELECT hold_id, user_id, seat_ids, hold_expires_at 
  FROM seat_holds 
  WHERE hold_id = $1 AND user_id = $2 AND hold_expires_at > NOW()
  FOR UPDATE;
  
  -- 2. Verify payment was authorized
  -- (payment_id verified via Payment Service call before this txn)
  
  -- 3. Update all seats to BOOKED
  UPDATE seats 
  SET status = 'booked', booked_by = $2, booking_id = $3 
  WHERE show_id = $4 AND seat_id = ANY($5) AND status = 'held' AND held_by = $2;
  
  -- 4. Verify all seats were updated (row count = expected seat count)
  -- If mismatch → ROLLBACK (hold expired mid-transaction)
  
  -- 5. Insert booking record
  INSERT INTO bookings (booking_id, user_id, show_id, seat_ids, total_amount, payment_id, status, booked_at)
  VALUES ($3, $2, $4, $5, $6, $7, 'confirmed', NOW());
  
  -- 6. Insert audit log
  INSERT INTO booking_audit_log (booking_id, action, actor, details, created_at)
  VALUES ($3, 'CONFIRMED', $2, '{"seats": ...}', NOW());
  
  -- 7. Decrement available seat count
  UPDATE shows SET available_seats = available_seats - $8 WHERE show_id = $4;
  
COMMIT;

-- 8. Post-commit: Delete Redis hold keys, publish Kafka event
```

#### The Core Challenge: Seat Booking Without Double-Booking

**Approach 1: Pessimistic Locking (Database Lock)** ⭐

```sql
BEGIN TRANSACTION;
  -- Lock the seat row
  SELECT * FROM seats WHERE event_id = ? AND seat_id = ? FOR UPDATE;
  
  -- Check availability
  IF seat.status = 'available' THEN
    UPDATE seats SET status = 'held', held_by = ?, held_until = NOW() + INTERVAL '10 MINUTES';
    COMMIT;
    RETURN SUCCESS;
  ELSE
    ROLLBACK;
    RETURN SEAT_UNAVAILABLE;
  END IF;
```

- **Pros**: Guaranteed no double booking, simple
- **Cons**: High lock contention for popular events (thousands hitting same rows)

**Approach 2: Optimistic Locking (Version-Based)**

```sql
-- Step 1: Read seat with version
SELECT seat_id, status, version FROM seats WHERE event_id = ? AND seat_id = ?;

-- Step 2: Update only if version matches
UPDATE seats SET status = 'held', held_by = ?, version = version + 1, held_until = NOW() + '10 min'
WHERE event_id = ? AND seat_id = ? AND version = ? AND status = 'available';

-- If rows_affected = 0 → someone else got it → retry with different seat or return error
```

- **Pros**: No locks held, better for read-heavy scenarios
- **Cons**: High retry rate during hot events

**Approach 3: Redis Atomic Lock** ⭐ (recommended for high-traffic events)

```
-- Atomically try to hold a seat
SET seat:{event_id}:{seat_id}:lock {user_id} NX EX 600
  NX = only if not exists
  EX 600 = expire in 10 minutes (auto-release hold)

If SET returns OK → seat is held for this user
If SET returns nil → seat already held by someone else
```

- **Pros**: Sub-millisecond, handles 100K+ ops/sec, auto-expire releases holds
- **Cons**: Redis data must be reconciled with DB on success

**Recommended Hybrid**:
1. Use **Redis** for fast seat locking (hold phase)
2. On checkout completion, write booking to **PostgreSQL** (source of truth)
3. If hold expires (user didn't pay), Redis key auto-expires → seat becomes available

#### Virtual Queue Service — Handling 1M Simultaneous Users (For Hot Events)

The virtual queue is the **most critical component for hot events**. Without it, 1M users simultaneously hitting the Booking Service would crash it.

**Architecture**:
```
┌─────────┐     ┌───────────────┐     ┌──────────────┐     ┌──────────────┐
│  Users  │────▶│ API Gateway   │────▶│ Virtual Queue│────▶│ Booking Svc  │
│ (1M)    │     │ (routes to Q) │     │ Service      │     │ (100 users/s)│
└─────────┘     └───────────────┘     └──────┬───────┘     └──────────────┘
                                             ��
                                      ┌──────▼───────┐
                                      │   Redis      │
                                      │ (ZSET queue) │
                                      └──────────────┘
```

**Detailed Flow**:

1. **Event admin marks event as "hot"** (or auto-detected: pre-registration count > threshold)
2. **On-sale time arrives** → API Gateway switches from direct booking to queue mode
3. **User clicks "Book Now"**:
   ```
   POST /api/v1/queue/join
   { "event_id": "evt-uuid", "user_id": "usr-uuid" }
   
   Redis: ZADD queue:{event_id} {timestamp_with_microseconds} {user_id}
   
   Response: 200 OK
   {
     "queue_position": 4523,
     "estimated_wait_seconds": 180,
     "status_poll_url": "/api/v1/queue/status?token=abc"
   }
   ```
4. **Client polls queue status** every 5 seconds (or receives update via SSE/WebSocket):
   ```
   GET /api/v1/queue/status?token=abc
   Response: { "position": 3201, "estimated_wait_seconds": 120 }
   ```
5. **Queue Drainer Worker** (runs continuously):
   ```python
   while True:
       # Pop next batch of users whose turn it is
       users = redis.zpopmin(f"queue:{event_id}", count=BATCH_SIZE)  # e.g., 10 users
       
       for user_id, score in users:
           # Issue a signed booking token (JWT)
           token = jwt.encode({
               "user_id": user_id,
               "event_id": event_id,
               "issued_at": now(),
               "expires_at": now() + timedelta(minutes=10),
               "max_tickets": 4
           }, SECRET_KEY)
           
           # Notify user their turn has arrived
           push_notification(user_id, {"status": "your_turn", "booking_token": token})
       
       # Control the drain rate: e.g., 100 users/sec
       sleep(BATCH_SIZE / DRAIN_RATE)
   ```
6. **User receives token** → selects seats → holds → pays → books (all within 10-minute token window)
7. **Token validation** at every booking endpoint: reject if expired, already used, or wrong event

**Key Design Decisions**:
- **Drain rate**: Tuned per event based on Booking Service capacity (e.g., 100 users/sec)
- **Position estimation**: `wait_time = position / drain_rate`
- **Fairness**: ZADD score = timestamp → FIFO ordering. For lottery: score = random()
- **Abandonment**: If user doesn't act within 10-min window, token expires, slot wasted (acceptable)
- **Scaling**: Queue is in Redis (single sorted set per event) — handles millions of entries easily. Queue Drainer is a simple stateless worker (can have hot standby)

#### Seat Map Service — Real-Time Availability Push

**Problem**: 10,000 users viewing the same show's seat map. As seats get held/booked, all users need live updates.

**Architecture**:
```
Booking Service ──(seat status changed)──▶ Kafka ──▶ Seat Map Consumer ──▶ WebSocket Server ──▶ All connected clients
```

**Detailed Flow**:
1. When a seat status changes (available → held, held → booked, held → available), Booking Service publishes event to Kafka topic `seat-status-changes`
2. Seat Map Consumer subscribes to the topic, filters by show_id
3. For each change, broadcasts to all WebSocket clients subscribed to that show_id:
   ```json
   {
     "type": "seat_update",
     "show_id": "show-uuid",
     "updates": [
       {"seat_id": "P-A1", "status": "held"},
       {"seat_id": "P-A2", "status": "held"}
     ]
   }
   ```
4. Client UI immediately updates seat color (green → yellow)

**WebSocket Connection Management**:
- Each WebSocket server maintains: `Map<show_id, Set<WebSocket connections>>`
- On connect: client sends `{"subscribe": "show-uuid"}` → added to the set
- On disconnect / page leave → removed from set
- **Scaling**: Multiple WebSocket server instances. Use Redis Pub/Sub to fan out Kafka events to all WS instances:
  ```
  Kafka → WS Server 1 → its local clients
                          Redis Pub/Sub channel: seat-updates:{show_id}
  Kafka → WS Server 2 → its local clients
  ```

**Fallback**: For clients that don't support WebSocket, use SSE (Server-Sent Events) or short polling (every 3 seconds)

#### Payment Service — Two-Phase Payment with Idempotency

**Why two-phase?** Hold the funds first (authorize), confirm the charge after booking is persisted (capture). If booking fails, release the authorization — no money moves.

**Flow**:
```
1. User clicks "Confirm Booking"
2. Booking Service calls Payment Service:
     POST /internal/payments/authorize
     {
       "idempotency_key": "hold-uuid",      ← reuse hold_id as idempotency key
       "amount": 1000,
       "currency": "INR",
       "payment_method_id": "pm-uuid",
       "metadata": {"booking_id": "book-uuid", "show_id": "..."}
     }
3. Payment Service calls Payment Gateway (Stripe/RazorPay):
     → Authorize $1000 on card
     ← Authorization ID returned
4. Booking Service persists booking in DB (within transaction)
5. Booking Service calls Payment Service:
     POST /internal/payments/capture
     { "authorization_id": "auth-uuid" }
6. If step 4 fails → Booking Service calls:
     POST /internal/payments/void
     { "authorization_id": "auth-uuid" }
     → Funds released back to user's card
```

**Idempotency**: If step 2 is retried (network timeout), same `idempotency_key` returns the same authorization — no double-charge.

#### Hold Expiry Worker — Background Cleanup

Holds that expire (user abandoned checkout) must be cleaned up:

```python
# Runs every 30 seconds
def cleanup_expired_holds():
    # 1. Redis handles its own expiry (TTL on SET keys)
    #    But we also need to clean up the DB
    
    # 2. Find expired holds in PostgreSQL
    expired = db.query("""
        SELECT hold_id, show_id, seat_ids 
        FROM seat_holds 
        WHERE status = 'active' AND hold_expires_at < NOW()
    """)
    
    for hold in expired:
        db.begin_transaction()
        
        # Mark hold as expired
        db.execute("UPDATE seat_holds SET status = 'expired' WHERE hold_id = %s", hold.hold_id)
        
        # Return seats to available
        db.execute("""
            UPDATE seats SET status = 'available', held_by = NULL, held_until = NULL 
            WHERE show_id = %s AND seat_id = ANY(%s) AND status = 'held'
        """, hold.show_id, hold.seat_ids)
        
        # Increment available seat count
        db.execute("""
            UPDATE shows SET available_seats = available_seats + %s 
            WHERE show_id = %s
        """, len(hold.seat_ids), hold.show_id)
        
        db.commit()
        
        # Publish seat-available event (for waitlist and seat map updates)
        kafka.produce('seat-status-changes', {
            'show_id': hold.show_id,
            'seats': hold.seat_ids,
            'status': 'available',
            'reason': 'hold_expired'
        })
```

**Why both Redis TTL and DB cleanup?**
- Redis TTL ensures the lock key disappears instantly (fast re-availability for new users)
- DB cleanup ensures the source of truth is consistent (handles edge cases where Redis and DB diverge)

#### Waitlist Consumer

Listens on Kafka for `seat-status-changes` where `reason = 'hold_expired'` or `reason = 'booking_cancelled'`:

```python
def on_seats_available(event):
    show_id = event['show_id']
    available_count = len(event['seats'])
    
    # Pop top N users from waitlist
    waiters = db.query("""
        SELECT user_id, requested_count 
        FROM waitlist 
        WHERE show_id = %s AND status = 'waiting' 
        ORDER BY created_at ASC 
        LIMIT 10
    """, show_id)
    
    for waiter in waiters:
        if waiter.requested_count <= available_count:
            # Notify user: "Seats are now available! You have 5 minutes to book."
            send_notification(waiter.user_id, {
                "type": "waitlist_available",
                "show_id": show_id,
                "available": available_count,
                "expires_in": 300
            })
            db.execute("UPDATE waitlist SET status = 'notified', notified_at = NOW() WHERE ...")
            break
```

---

## 5. APIs

### Search Events
```http
GET /api/v1/events?city=mumbai&category=movie&date=2026-03-13&q=avengers&sort=popularity&page=1&limit=20

Response: 200 OK
{
  "events": [
    {
      "event_id": "evt-uuid",
      "title": "Avengers: Secret Wars",
      "category": "movie",
      "venue": {"name": "PVR IMAX", "city": "Mumbai"},
      "poster_url": "https://cdn.example.com/posters/avengers.jpg",
      "start_date": "2026-03-13",
      "price_range": {"min": 200, "max": 800},
      "availability": "available",
      "shows_count": 5
    }
  ],
  "total": 3,
  "pagination": {"page": 1, "total_pages": 1}
}
```

### Get Event Details + Show Times
```http
GET /api/v1/events/{event_id}

Response: 200 OK
{
  "event_id": "evt-uuid",
  "title": "Avengers: Secret Wars",
  "description": "...",
  "category": "movie",
  "venue": {
    "venue_id": "ven-uuid",
    "name": "PVR IMAX",
    "address": "Phoenix Mills, Mumbai",
    "city": "Mumbai",
    "capacity": 350
  },
  "shows": [
    {
      "show_id": "show-uuid-1",
      "show_time": "2026-03-13T14:00:00+05:30",
      "available_seats": 125,
      "total_seats": 350,
      "status": "on_sale",
      "sections": [
        {"name": "Premium", "price": 800, "available": 20, "total": 50},
        {"name": "Executive", "price": 500, "available": 45, "total": 100},
        {"name": "Normal", "price": 200, "available": 60, "total": 200}
      ]
    },
    {
      "show_id": "show-uuid-2",
      "show_time": "2026-03-13T18:00:00+05:30",
      "available_seats": 0,
      "total_seats": 350,
      "status": "sold_out",
      "waitlist_enabled": true
    }
  ]
}
```

### Get Seat Map (Real-Time)
```http
GET /api/v1/events/{event_id}/shows/{show_id}/seats

Response: 200 OK
{
  "show_id": "show-uuid",
  "venue": {"name": "PVR IMAX", "layout_svg_url": "https://cdn.example.com/layouts/pvr-imax.svg"},
  "sections": [
    {
      "name": "Premium",
      "price": 800,
      "rows": [
        {
          "row": "A",
          "seats": [
            {"seat_id": "P-A1", "number": 1, "status": "available"},
            {"seat_id": "P-A2", "number": 2, "status": "held"},
            {"seat_id": "P-A3", "number": 3, "status": "booked"},
            {"seat_id": "P-A4", "number": 4, "status": "available"},
            ...
          ]
        },
        {"row": "B", "seats": [...]}
      ]
    }
  ],
  "websocket_url": "wss://ws.example.com/seats/show-uuid"  // for live updates
}
```

### Join Virtual Queue (Hot Events)
```http
POST /api/v1/queue/join
Authorization: Bearer <user_token>
{
  "event_id": "evt-uuid",
  "show_id": "show-uuid"
}

Response: 200 OK
{
  "queue_id": "q-uuid",
  "position": 4523,
  "estimated_wait_seconds": 180,
  "poll_url": "/api/v1/queue/status/q-uuid"
}
```

### Poll Queue Status
```http
GET /api/v1/queue/status/{queue_id}

Response (waiting): 200 OK
{
  "status": "waiting",
  "position": 2100,
  "estimated_wait_seconds": 85
}

Response (your turn): 200 OK
{
  "status": "your_turn",
  "booking_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_expires_at": "2026-03-13T10:10:00Z",
  "max_tickets": 4
}
```

### Hold Seats
```http
POST /api/v1/bookings/hold
Authorization: Bearer <user_token>
{
  "show_id": "show-uuid",
  "seat_ids": ["P-A1", "P-A4"],
  "booking_token": "eyJhbGciOiJIUzI1NiIs..."   // from virtual queue (hot events)
}

Response: 200 OK
{
  "hold_id": "hold-uuid",
  "seats": [
    {"seat_id": "P-A1", "section": "Premium", "row": "A", "number": 1, "price": 800},
    {"seat_id": "P-A4", "section": "Premium", "row": "A", "number": 4, "price": 800}
  ],
  "hold_expires_at": "2026-03-13T10:10:00Z",
  "subtotal": 1600,
  "convenience_fee": 50,
  "tax": 82,
  "total_amount": 1732
}

Error Response: 409 Conflict
{
  "error": "SEAT_UNAVAILABLE",
  "unavailable_seats": ["P-A1"],
  "message": "Some selected seats are no longer available."
}
```

### Confirm Booking (Pay & Book)
```http
POST /api/v1/bookings/confirm
Authorization: Bearer <user_token>
Idempotency-Key: idem-uuid-12345
{
  "hold_id": "hold-uuid",
  "payment_method_id": "pm-uuid",
  "promo_code": "FIRST50"          // optional
}

Response: 201 Created
{
  "booking_id": "booking-uuid",
  "status": "confirmed",
  "show": {"title": "Avengers: Secret Wars", "show_time": "2026-03-13T14:00:00+05:30", "venue": "PVR IMAX"},
  "tickets": [
    {"ticket_id": "tkt-uuid-1", "seat": "P-A1", "section": "Premium", "row": "A", "number": 1, "qr_code_url": "https://cdn.example.com/qr/tkt-uuid-1.png"},
    {"ticket_id": "tkt-uuid-2", "seat": "P-A4", "section": "Premium", "row": "A", "number": 4, "qr_code_url": "https://cdn.example.com/qr/tkt-uuid-2.png"}
  ],
  "payment": {
    "payment_id": "pay-uuid",
    "amount": 1732,
    "currency": "INR",
    "method": "VISA ****4242"
  },
  "confirmation_email_sent": true
}
```

### Cancel Booking
```http
POST /api/v1/bookings/{booking_id}/cancel
Authorization: Bearer <user_token>
{
  "reason": "change_of_plans"
}

Response: 200 OK
{
  "booking_id": "booking-uuid",
  "status": "cancelled",
  "refund": {
    "refund_id": "ref-uuid",
    "amount": 1500,
    "deduction": 232,
    "deduction_reason": "Cancellation fee (10%) + convenience fee non-refundable",
    "refund_to": "VISA ****4242",
    "estimated_refund_date": "2026-03-20"
  }
}
```

### Join Waitlist
```http
POST /api/v1/waitlist
Authorization: Bearer <user_token>
{
  "show_id": "show-uuid",
  "requested_seats": 2,
  "preferred_section": "Premium"
}

Response: 201 Created
{
  "waitlist_id": "wl-uuid",
  "position": 12,
  "status": "waiting",
  "notification_channels": ["push", "email"]
}
```

### Get User's Bookings
```http
GET /api/v1/users/me/bookings?status=confirmed&page=1

Response: 200 OK
{
  "bookings": [
    {
      "booking_id": "booking-uuid",
      "event_title": "Avengers: Secret Wars",
      "show_time": "2026-03-13T14:00:00+05:30",
      "venue": "PVR IMAX, Mumbai",
      "seats": ["P-A1", "P-A4"],
      "total_amount": 1732,
      "status": "confirmed",
      "booked_at": "2026-03-12T10:00:00Z"
    }
  ]
}
```

---

## 6. Data Model

### Why PostgreSQL as Source of Truth?
- **ACID transactions** are mandatory: seat hold, booking, and payment MUST be atomic
- Strong consistency prevents double-booking at the DB level (unique constraints)
- Rich querying for reporting, analytics, admin dashboards
- JSONB support for flexible metadata without sacrificing relational integrity
- Row-level locking (`SELECT ... FOR UPDATE`) for pessimistic concurrency control

### PostgreSQL — Venues & Sections (Relatively Static Data)

```sql
CREATE TABLE venues (
    venue_id        UUID PRIMARY KEY,
    name            VARCHAR(256) NOT NULL,
    address         TEXT,
    city            VARCHAR(128),
    state           VARCHAR(64),
    country         VARCHAR(64),
    capacity        INT,
    layout_svg_url  TEXT,           -- SVG seat map template
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE venue_sections (
    section_id      UUID PRIMARY KEY,
    venue_id        UUID REFERENCES venues,
    name            VARCHAR(64),        -- 'Premium', 'Executive', 'Normal', 'Balcony'
    total_seats     INT,
    sort_order      INT,                -- display order
    UNIQUE (venue_id, name)
);

CREATE TABLE venue_seat_templates (
    venue_id        UUID,
    section_id      UUID,
    seat_id         VARCHAR(16),        -- 'P-A1' (section prefix + row + number)
    row             VARCHAR(4),
    number          INT,
    x_pos           INT,                -- pixel position for seat map rendering
    y_pos           INT,
    seat_type       VARCHAR(20) DEFAULT 'standard',  -- 'standard', 'wheelchair', 'companion'
    PRIMARY KEY (venue_id, seat_id)
);
```

### MySQL — Events & Shows (Read-Heavy Catalog)

```sql
CREATE TABLE events (
    event_id        UUID PRIMARY KEY,
    title           VARCHAR(256) NOT NULL,
    category        VARCHAR(64),        -- 'movie', 'concert', 'sports', 'comedy', 'theatre'
    description     TEXT,
    venue_id        UUID NOT NULL,
    city            VARCHAR(128),
    start_date      DATE,
    end_date        DATE,
    poster_url      TEXT,
    banner_url      TEXT,
    artist_name     VARCHAR(256),
    language        VARCHAR(32),
    duration_min    INT,
    age_rating      VARCHAR(10),        -- 'U', 'UA', 'A', 'PG-13', 'R'
    is_hot          BOOLEAN DEFAULT FALSE,  -- triggers virtual queue
    status          VARCHAR(20) DEFAULT 'upcoming',  -- 'upcoming', 'on_sale', 'sold_out', 'completed', 'cancelled'
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_city_category (city, category),
    INDEX idx_status (status),
    INDEX idx_start_date (start_date),
    FULLTEXT idx_search (title, artist_name, description)
);

CREATE TABLE shows (
    show_id         UUID PRIMARY KEY,
    event_id        UUID NOT NULL,      -- FK to events
    venue_id        UUID NOT NULL,
    show_time       TIMESTAMP NOT NULL,
    total_seats     INT NOT NULL,
    available_seats INT NOT NULL,
    status          VARCHAR(20) DEFAULT 'on_sale',   -- 'on_sale', 'sold_out', 'cancelled', 'completed'
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_event (event_id),
    INDEX idx_show_time (show_time)
);

CREATE TABLE show_pricing (
    show_id         UUID,
    section_id      UUID,
    section_name    VARCHAR(64),
    base_price      DECIMAL(10,2) NOT NULL,
    current_price   DECIMAL(10,2) NOT NULL,  -- dynamic pricing may differ
    available_seats INT,
    total_seats     INT,
    PRIMARY KEY (show_id, section_id)
);
```

**Why MySQL for Events and PostgreSQL for Bookings?**
- Events/Shows are read-heavy catalog data (millions of reads, few writes). MySQL with read replicas handles this excellently
- Bookings require ACID transactions, row-level locking, and complex constraints. PostgreSQL excels here
- Alternative: Use a single PostgreSQL instance for both if simplicity is preferred

### PostgreSQL — Seats (Per-Show Instance)

```sql
CREATE TABLE seats (
    seat_id         VARCHAR(16),
    show_id         UUID NOT NULL,
    section_id      UUID,
    section_name    VARCHAR(64),
    row             VARCHAR(4),
    number          INT,
    price           DECIMAL(10,2),
    status          VARCHAR(12) NOT NULL DEFAULT 'available',
                    -- 'available', 'held', 'booked'
    held_by         UUID,               -- user_id who holds
    held_until      TIMESTAMP,
    booked_by       UUID,               -- user_id who booked
    booking_id      UUID,
    version         INT DEFAULT 0,      -- for optimistic locking
    updated_at      TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (show_id, seat_id),
    INDEX idx_status (show_id, status),
    INDEX idx_held_until (held_until) WHERE status = 'held',  -- partial index for cleanup
    CONSTRAINT chk_status CHECK (status IN ('available', 'held', 'booked')),
    CONSTRAINT uq_booked UNIQUE (show_id, seat_id, booking_id)  -- prevents double-booking
);
```

### PostgreSQL — Seat Holds (Explicit Hold Tracking)

```sql
CREATE TABLE seat_holds (
    hold_id         UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    show_id         UUID NOT NULL,
    seat_ids        TEXT[] NOT NULL,         -- array of seat_ids
    seat_count      INT NOT NULL,
    total_amount    DECIMAL(10,2),
    status          VARCHAR(20) DEFAULT 'active',  -- 'active', 'converted', 'expired', 'released'
    hold_expires_at TIMESTAMP NOT NULL,
    booking_token   TEXT,                    -- JWT from virtual queue (if applicable)
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_user (user_id),
    INDEX idx_expiry (status, hold_expires_at) WHERE status = 'active'
);
```

### PostgreSQL — Bookings (Confirmed Transactions)

```sql
CREATE TABLE bookings (
    booking_id      UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    show_id         UUID NOT NULL,
    event_id        UUID NOT NULL,
    hold_id         UUID,                   -- reference to the original hold
    seat_ids        TEXT[] NOT NULL,
    seat_count      INT NOT NULL,
    subtotal        DECIMAL(10,2) NOT NULL,
    convenience_fee DECIMAL(10,2) DEFAULT 0,
    discount        DECIMAL(10,2) DEFAULT 0,
    tax             DECIMAL(10,2) DEFAULT 0,
    total_amount    DECIMAL(10,2) NOT NULL,
    promo_code      VARCHAR(32),
    payment_id      UUID,
    payment_method  VARCHAR(32),
    status          VARCHAR(20) DEFAULT 'confirmed',
                    -- 'confirmed', 'cancelled', 'refunded', 'checked_in'
    cancellation_reason TEXT,
    refund_id       UUID,
    refund_amount   DECIMAL(10,2),
    booked_at       TIMESTAMP DEFAULT NOW(),
    cancelled_at    TIMESTAMP,
    idempotency_key VARCHAR(128) UNIQUE,    -- prevents duplicate bookings
    INDEX idx_user (user_id, booked_at DESC),
    INDEX idx_show (show_id),
    INDEX idx_event (event_id),
    INDEX idx_status (status)
);
```

### PostgreSQL — Tickets (Issued per Seat)

```sql
CREATE TABLE tickets (
    ticket_id       UUID PRIMARY KEY,
    booking_id      UUID NOT NULL REFERENCES bookings,
    show_id         UUID NOT NULL,
    seat_id         VARCHAR(16) NOT NULL,
    section_name    VARCHAR(64),
    row             VARCHAR(4),
    seat_number     INT,
    price           DECIMAL(10,2),
    qr_code_data    TEXT NOT NULL,           -- encrypted payload for QR generation
    qr_code_url     TEXT,                    -- CDN URL to pre-generated QR image
    status          VARCHAR(20) DEFAULT 'active',  -- 'active', 'cancelled', 'used'
    scanned_at      TIMESTAMP,              -- when QR was scanned at venue entry
    created_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE (show_id, seat_id, status) WHERE status = 'active'  -- one active ticket per seat
);
```

### PostgreSQL — Booking Audit Log (Immutable)

```sql
CREATE TABLE booking_audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    booking_id      UUID,
    hold_id         UUID,
    action          VARCHAR(32) NOT NULL,
                    -- 'HOLD_CREATED', 'HOLD_EXPIRED', 'HOLD_RELEASED',
                    -- 'PAYMENT_AUTHORIZED', 'PAYMENT_CAPTURED', 'PAYMENT_FAILED',
                    -- 'BOOKING_CONFIRMED', 'BOOKING_CANCELLED', 'REFUND_INITIATED',
                    -- 'REFUND_COMPLETED', 'TICKET_SCANNED'
    actor           VARCHAR(128),           -- user_id, 'system', 'admin'
    details         JSONB,                  -- action-specific data
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);
-- No UPDATE or DELETE on this table — append-only for compliance
```

### PostgreSQL — Waitlist

```sql
CREATE TABLE waitlist (
    waitlist_id     UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    show_id         UUID NOT NULL,
    requested_seats INT NOT NULL,
    preferred_section VARCHAR(64),
    status          VARCHAR(20) DEFAULT 'waiting',  -- 'waiting', 'notified', 'fulfilled', 'expired'
    notified_at     TIMESTAMP,
    expires_at      TIMESTAMP,             -- auto-expire when show starts
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_show_status (show_id, status, created_at),
    UNIQUE (user_id, show_id)              -- one waitlist entry per user per show
);
```

### PostgreSQL — Payments

```sql
CREATE TABLE payments (
    payment_id      UUID PRIMARY KEY,
    booking_id      UUID REFERENCES bookings,
    user_id         UUID NOT NULL,
    amount          DECIMAL(10,2) NOT NULL,
    currency        VARCHAR(3) DEFAULT 'INR',
    payment_method  VARCHAR(32),            -- 'credit_card', 'debit_card', 'upi', 'wallet'
    gateway         VARCHAR(32),            -- 'stripe', 'razorpay'
    gateway_txn_id  VARCHAR(128),           -- PSP's transaction reference
    authorization_id VARCHAR(128),
    status          VARCHAR(20),            -- 'authorized', 'captured', 'voided', 'refunded'
    idempotency_key VARCHAR(128) UNIQUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    captured_at     TIMESTAMP,
    INDEX idx_booking (booking_id)
);
```

### Redis — Seat Locks, Queue, and Caches

```
# ---- Seat Holds (auto-expires) ----
Key:    seat:{show_id}:{seat_id}:lock
Value:  {user_id}:{hold_id}
TTL:    600 (10 minutes)
# SET NX EX — atomic hold acquisition

# ---- Virtual Queue ----
Key:    queue:{event_id}:{show_id}
Type:   Sorted Set
Members: user_id
Scores:  join_timestamp_microseconds   (FIFO) or random() (lottery)

# ---- Active Queue Token Tracker ----
Key:    queue:token:{event_id}:{user_id}
Value:  JWT booking token
TTL:    600 (10 minutes — token validity window)

# ---- Available Seat Count (fast check) ----
Key:    seats:available:{show_id}
Value:  integer (DECR on hold, INCR on expiry)
TTL:    none (maintained by booking service)

# ---- Available Seat Set (for seat map) ----
Key:    seats:avail:set:{show_id}
Type:   Set
Members: seat_ids that are available
# SREM on hold, SADD on expiry

# ---- Event Metadata Cache ----
Key:    event:meta:{event_id}
Value:  Hash {title, category, venue, poster_url, ...}
TTL:    300 (5 minutes)

# ---- Show Pricing Cache ----
Key:    show:pricing:{show_id}
Value:  JSON string with section prices
TTL:    60 (1 minute — dynamic pricing may change)
```

### Kafka Topics

```
Topic: seat-status-changes        (partitioned by show_id)
  Purpose: Real-time seat map updates, waitlist triggers
  Message: { show_id, seat_ids[], new_status, reason, timestamp }

Topic: booking-events              (partitioned by booking_id)
  Purpose: Analytics, notification triggers, audit
  Message: { booking_id, event_type, user_id, show_id, seats, amount, timestamp }
  event_types: booking.confirmed, booking.cancelled, booking.refunded

Topic: payment-events              (partitioned by payment_id)
  Purpose: Payment reconciliation, accounting
  Message: { payment_id, booking_id, status, amount, gateway_txn_id, timestamp }

Topic: notification-events         (partitioned by user_id)
  Purpose: Email/SMS/Push notification dispatch
  Message: { user_id, type, template, vars, channels[], timestamp }
  types: booking_confirmation, cancellation_confirmation, waitlist_available, queue_turn
```

### Elasticsearch — Event Search Index

```json
{
  "event_id": "evt-uuid",
  "title": "Avengers: Secret Wars",
  "title_suggest": { "input": ["Avengers", "Secret Wars", "Avengers Secret Wars"] },
  "category": "movie",
  "city": "Mumbai",
  "venue_name": "PVR IMAX",
  "artist_name": null,
  "language": "English",
  "age_rating": "UA",
  "start_date": "2026-03-13",
  "end_date": "2026-03-27",
  "price_min": 200,
  "price_max": 800,
  "status": "on_sale",
  "tags": ["marvel", "superhero", "imax", "3d"],
  "popularity_score": 9500,
  "has_available_shows": true,
  "location": { "lat": 18.9944, "lon": 72.8265 },
  "created_at": "2026-03-01T00:00:00Z"
}
```

**Index settings**: Autocomplete via `completion` suggester on `title_suggest`, geo-distance queries on `location` for "events near me".

---

## 7. Fault Tolerance

### General Techniques

| Technique | Application |
|---|---|
| **DB replication** | PostgreSQL synchronous standby → zero booking data loss on primary failure |
| **Redis Cluster** | 3 masters + 3 replicas for seat locks; auto-failover if a master dies |
| **Kafka durability** | RF=3, min.insync.replicas=2; no event lost |
| **Idempotency keys** | On booking confirmation API → prevents double-charge on retry |
| **Circuit breaker** | Between Booking Service → Payment Gateway; if PSP is down, don't hold seats forever |
| **Health checks** | LB actively monitors all service instances every 5 seconds |
| **Multi-AZ deployment** | All critical services deployed across 3 availability zones |

### Problem-Specific Fault Tolerance

#### 1. Double-Booking Prevention (The #1 Guarantee)
Multiple layers of defense:
```
Layer 1: Redis SET NX EX       → fast, atomic, prevents concurrent holds
Layer 2: DB optimistic lock    → version column check on UPDATE
Layer 3: DB unique constraint  → UNIQUE (show_id, seat_id) WHERE status='booked'
Layer 4: Application check     → verify hold ownership before booking
```
If Redis is down, fall back to pessimistic DB locking (`SELECT ... FOR UPDATE`). Slower but still correct.

#### 2. Hold Not Released (User Abandons Checkout)
- **Primary**: Redis TTL auto-expires the lock key → seat becomes holdable by others immediately
- **Secondary**: Hold Expiry Worker scans PostgreSQL every 30 seconds for `seat_holds WHERE status='active' AND hold_expires_at < NOW()` → releases seats in DB
- **Tertiary**: Seat map shows "held" seats with a countdown timer on the client → if timer expires, client UI flips seat back to green (eventual client-side refresh)
- **Edge case**: If Hold Expiry Worker is down → holds accumulate → monitor alert on `active_holds_past_expiry` metric → manual intervention within minutes

#### 3. Payment Succeeds but Booking DB Write Fails
```
Scenario:
  1. Payment authorized ✓
  2. Payment captured ✓  
  3. DB INSERT booking → FAILS (DB timeout, disk full, etc.)
  
Without handling: User is charged but has no booking!

Solution (Saga with compensation):
  1. Booking Service detects DB failure
  2. Immediately calls Payment Service: POST /payments/{id}/refund
  3. Releases seat holds in Redis (DEL keys)
  4. Returns error to user: "Booking failed, payment will be refunded"
  5. Audit log records the failure for investigation
  
  Additionally:
  - Background reconciliation job: compare payments DB vs bookings DB every hour
  - Any payment without a matching booking → auto-refund + alert
```

#### 4. Payment Gateway Timeout (No Response)
```
Scenario: 
  Booking Service sends authorize request to Stripe → no response after 10 seconds

Solution:
  1. DO NOT assume failure (payment may have succeeded!)
  2. Mark booking as 'payment_pending'
  3. Background worker polls Payment Gateway for status every 30 seconds
  4. If authorized → proceed with booking
  5. If declined → release hold, notify user
  6. If still unknown after 5 minutes → release hold, log for manual review
  
  Key principle: NEVER release funds you haven't confirmed were charged
```

#### 5. Redis Crash (All Holds Lost)
```
Impact: All in-memory seat locks disappear. 
  - Seats that were held become "available" again (Redis keys gone)
  - Users in the middle of checkout will fail on "confirm" (hold verification fails)

Mitigation:
  - Redis Cluster with replicas → survives individual node failure
  - If entire Redis cluster fails:
    1. Booking Service detects Redis unavailability (circuit breaker trips)
    2. Falls back to pessimistic DB locking for new holds (slower but functional)
    3. Existing holds: DB still has seat_holds table → serve from DB
    4. Users in checkout can still complete IF their hold is in the DB
  - Recovery: When Redis comes back, rebuild lock keys from active holds in DB
```

#### 6. Hot Event Overload (Traffic Spike)
```
Without virtual queue:
  1M users → API Gateway → Booking Service → 50K req/sec → DB overwhelmed → cascading failure

With virtual queue:
  1M users → API Gateway → Virtual Queue (Redis ZADD = trivial) → controlled drain at 100 users/sec → Booking Service handles easily

Additional protections:
  - API Gateway: adaptive rate limiting (tighten limits when load increases)
  - Booking Service: auto-scaling (Kubernetes HPA on CPU/request rate)
  - PostgreSQL: read replicas for seat map queries; primary only for writes
  - CDN: cache static assets (posters, seat map SVGs, JS/CSS) → reduce origin load by 90%
```

#### 7. Race Condition: Hold Expires During Payment
```
Timeline:
  T=0:    User holds seats (TTL = 10 min)
  T=9:50: User clicks "Pay" (10 seconds left)
  T=10:   Hold expires in Redis → another user holds the same seats
  T=10:05: Payment authorized for original user
  T=10:10: Original user tries to book → seats are held by someone else!

Solution:
  1. On "Pay" click, EXTEND the Redis TTL by 2 minutes (grace period):
     SET seat:{show_id}:{seat_id}:lock {user_id} XX EX 720
     (XX = only if exists, i.e., only if we still hold it)
  2. If XX fails (lock already expired) → abort checkout immediately, refund payment auth
  3. Display countdown timer on UI with buffer: show "9:30 remaining" when actual TTL is 10:00
     This encourages users to complete early and reduces edge cases
```

#### 8. Network Partition Between Services
```
Scenario: Booking Service can reach Redis but not PostgreSQL

Solution:
  - Hold acquisition still works (Redis only) → user can select and hold seats
  - Booking confirmation fails → return "Service temporarily unavailable, your seats are held for 10 more minutes"
  - User retries when DB is back → hold is still valid → booking succeeds
  - If hold expires during outage → user loses hold but gets a clear error
```

### Preventing Scalper Bots — Deep Dive

| Defense | How It Works | Effectiveness |
|---|---|---|
| **CAPTCHA (reCAPTCHA v3)** | Score-based challenge before queue entry; invisible for normal users | High (blocks simple bots) |
| **Rate limiting** | Max 4 tickets per user per event per phone number | Medium (bots use multiple accounts) |
| **Browser fingerprinting** | Canvas fingerprint, WebGL hash, font enumeration — detect headless browsers | High (hard to spoof) |
| **TLS fingerprinting (JA3)** | Distinguish real browsers from scripts by TLS handshake signature | High (very hard to spoof) |
| **Queue randomization** | For VIP events: random queue position instead of FIFO | Eliminates first-mover advantage |
| **Verified fan program** | Pre-register with ID verification; priority queue for verified fans | Very high (identity-based) |
| **Device binding** | One device per booking session (device ID cookie + fingerprint) | Medium-High |
| **Phone OTP** | Require OTP verification before checkout | High (requires real phone number per booking) |
| **Dynamic pricing** | Higher prices in first minutes (scalpers need low prices for profit margin) | Economic deterrent |
| **Post-purchase verification** | Require ID at venue entry matching the booking name | Eliminates scalper resale profit |

---

## 8. Additional Considerations

### Contiguous Seat Selection Algorithm — Complete Implementation

When user requests "4 seats together", the system should find the best contiguous group:

```python
def find_best_contiguous_seats(show_id: str, section: str, count: int) -> List[Seat]:
    """
    Find 'count' contiguous available seats, prioritizing:
    1. Center seats (better view)
    2. Lower row numbers (closer to screen/stage)
    3. Fewest gaps (aesthetic seating)
    """
    rows = db.query("""
        SELECT row, seat_id, number, status 
        FROM seats 
        WHERE show_id = %s AND section_name = %s 
        ORDER BY row ASC, number ASC
    """, show_id, section)
    
    candidates = []
    
    for row_label, seats_in_row in group_by_row(rows):
        # Find all contiguous runs of available seats
        current_run = []
        for seat in seats_in_row:
            if seat.status == 'available':
                current_run.append(seat)
            else:
                if len(current_run) >= count:
                    candidates.append(current_run[:])
                current_run = []
        if len(current_run) >= count:
            candidates.append(current_run[:])
    
    if not candidates:
        return []  # No contiguous block available
    
    # Score each candidate: lower score = better
    def score(run):
        # Take the first 'count' seats from this run
        group = run[:count]
        row_ord = ord(group[0].row)  # 'A'=65 (front), 'Z'=90 (back) → prefer front
        center = max(s.number for s in seats_in_row) / 2  # middle seat number
        avg_pos = sum(s.number for s in group) / count
        center_distance = abs(center - avg_pos)
        return (row_ord * 100) + center_distance  # weight row > center proximity
    
    best_run = min(candidates, key=score)
    return best_run[:count]
```

**"Best Available" button**: Calls this algorithm → auto-selects optimal seats → user can override.

### Dynamic Pricing Engine

```python
def calculate_dynamic_price(show_id: str, section: str, base_price: float) -> float:
    show = get_show(show_id)
    section_info = get_section_info(show_id, section)
    
    # Factor 1: Demand (fill rate)
    fill_rate = 1 - (section_info.available / section_info.total)
    demand_multiplier = 1.0
    if fill_rate > 0.9:
        demand_multiplier = 1.5     # > 90% full → 50% premium
    elif fill_rate > 0.7:
        demand_multiplier = 1.2     # > 70% full → 20% premium
    elif fill_rate < 0.3:
        demand_multiplier = 0.9     # < 30% full → 10% discount
    
    # Factor 2: Time to show
    hours_to_show = (show.show_time - now()).total_seconds() / 3600
    time_multiplier = 1.0
    if hours_to_show < 2:
        time_multiplier = 0.7      # Last-minute discount to fill seats
    elif hours_to_show < 6:
        time_multiplier = 0.85     # Moderate discount
    
    # Factor 3: Day of week
    day_multiplier = 1.2 if show.show_time.weekday() in (4, 5, 6) else 1.0  # Weekend premium
    
    # Floor and ceiling
    price = base_price * demand_multiplier * time_multiplier * day_multiplier
    price = max(price, base_price * 0.6)   # Never below 60% of base
    price = min(price, base_price * 2.0)   # Never above 200% of base
    
    return round(price, 2)
```

### Cancellation Policy Engine

```python
CANCELLATION_POLICIES = {
    'movie': [
        {'hours_before': 24, 'refund_pct': 100},   # Full refund > 24h before
        {'hours_before': 4,  'refund_pct': 75},     # 75% refund 4-24h before
        {'hours_before': 2,  'refund_pct': 50},     # 50% refund 2-4h before
        {'hours_before': 0,  'refund_pct': 0},      # No refund < 2h before
    ],
    'concert': [
        {'hours_before': 72, 'refund_pct': 100},
        {'hours_before': 24, 'refund_pct': 50},
        {'hours_before': 0,  'refund_pct': 0},
    ],
}

def calculate_refund(booking, show) -> dict:
    hours_remaining = (show.show_time - now()).total_seconds() / 3600
    policy = CANCELLATION_POLICIES.get(show.category, CANCELLATION_POLICIES['movie'])
    
    refund_pct = 0
    for tier in policy:
        if hours_remaining >= tier['hours_before']:
            refund_pct = tier['refund_pct']
            break
    
    refundable = booking.subtotal * (refund_pct / 100)
    # Convenience fee is NEVER refundable
    deductions = booking.total_amount - refundable
    
    return {
        'refund_amount': refundable,
        'deduction': deductions,
        'refund_pct': refund_pct,
        'policy_applied': f"{refund_pct}% refund for cancellation >{hours_remaining:.0f}h before show"
    }
```

### QR Ticket Generation & Verification

**QR Code Content** (signed, tamper-proof):
```python
import jwt, qrcode

def generate_ticket_qr(ticket):
    payload = {
        "ticket_id": str(ticket.ticket_id),
        "booking_id": str(ticket.booking_id),
        "show_id": str(ticket.show_id),
        "seat_id": ticket.seat_id,
        "user_hash": sha256(ticket.user_id + SECRET_SALT),  # privacy: don't expose user_id
        "issued_at": int(time.time()),
        "exp": int(ticket.show_time.timestamp()) + 7200,    # valid until 2h after show start
    }
    qr_data = jwt.encode(payload, QR_SIGNING_KEY, algorithm="RS256")  # RSA for offline verification
    
    qr_img = qrcode.make(qr_data)
    # Upload to CDN, return URL
    return upload_to_cdn(qr_img, f"qr/{ticket.ticket_id}.png")
```

**Venue Entry Scanning** (offline-capable):
```python
def verify_ticket_qr(qr_data: str) -> VerificationResult:
    try:
        payload = jwt.decode(qr_data, QR_PUBLIC_KEY, algorithms=["RS256"])
    except jwt.ExpiredSignatureError:
        return VerificationResult(valid=False, reason="Ticket expired")
    except jwt.InvalidTokenError:
        return VerificationResult(valid=False, reason="Invalid ticket")
    
    # Check if already scanned (online check preferred; offline fallback to local cache)
    if is_already_scanned(payload["ticket_id"]):
        return VerificationResult(valid=False, reason="Ticket already used")
    
    # Mark as scanned
    mark_scanned(payload["ticket_id"])
    
    return VerificationResult(
        valid=True,
        seat=payload["seat_id"],
        ticket_id=payload["ticket_id"]
    )
```

**Why RSA (asymmetric) signing?**
- Venue scanners only need the **public key** to verify tickets
- **Private key** stays on the server → scanners can't forge tickets
- Works **offline**: scanner verifies JWT signature without calling the server
- Online check for "already scanned" is preferred but not required (local bloom filter as fallback)

### Multi-Format Ticket Delivery
| Format | Delivery | Offline | Use Case |
|---|---|---|---|
| **In-app QR** | Instant | Yes (cached) | Primary — fastest entry |
| **Apple Wallet** | Push to Wallet | Yes | iPhone users — always accessible |
| **Google Pay** | Push to Google Wallet | Yes | Android users |
| **PDF e-ticket** | Email attachment | Yes (if downloaded) | Backup — works on any device |
| **SMS link** | SMS with deep link | No (needs internet) | Fallback for basic phones |

### Waitlist — Advanced Implementation

**Priority-based waitlist** (not just FIFO):
```python
WAITLIST_PRIORITY_WEIGHTS = {
    'premium_subscriber': 3.0,     # Premium app subscribers
    'loyalty_gold': 2.0,           # Gold loyalty tier
    'loyalty_silver': 1.5,
    'regular': 1.0,
}

def waitlist_score(user, created_at):
    priority = WAITLIST_PRIORITY_WEIGHTS.get(user.tier, 1.0)
    # Score = priority / time_waiting → higher priority + longer wait = higher score
    return priority * 1000000 - created_at.timestamp()
```

### Monitoring & Alerting

| Metric | Target | Alert Threshold |
|---|---|---|
| Booking success rate | > 99% | < 95% |
| Seat hold acquisition latency (p99) | < 100 ms | > 500 ms |
| Payment authorization latency (p99) | < 2 s | > 5 s |
| Expired holds not cleaned up | 0 | > 100 for > 5 min |
| Redis memory usage | < 70% | > 85% |
| Virtual queue drain rate vs target | ±5% | > 20% deviation |
| Double-booking incidents | 0 | > 0 (CRITICAL — page on-call) |
| Kafka consumer lag (seat-status-changes) | < 1000 msgs | > 10000 msgs |
| WebSocket connections per show | < 50K | > 100K (need more WS servers) |
| Cancellation / refund success rate | > 99.9% | < 99% |

### Caching Strategy

```
Layer 1: CDN                      → Event posters, seat map SVGs, static assets (TTL: 1 day)
Layer 2: Redis (metadata cache)   → Event details, show times, pricing (TTL: 5 min)
Layer 3: Redis (seat status)      → Available seats set, counts (real-time, no TTL)
Layer 4: MySQL read replicas      → Event catalog queries
Layer 5: PostgreSQL primary       → Booking writes (source of truth, never cached)
```

### Scaling for Global Events (Multi-Region)

For a global concert (e.g., Taylor Swift world tour):
- **Regional deployments**: US, EU, Asia each have independent Booking Service + DB
- **Event catalog**: Replicated globally via MySQL replication or Elasticsearch cross-cluster
- **Bookings are region-local**: A show in London is managed entirely by the EU region
- **User accounts**: Global user service with cross-region replication
- **CDN**: Posters, assets served from nearest edge worldwide
- **Virtual queue**: Queue per show (region-local Redis) — no cross-region coordination needed
