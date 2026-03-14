# 99. Design a Shared Calendar System (like Google Calendar)

---

## 1. Functional Requirements (FR)

- Create, update, delete events (title, time, location, description, recurrence)
- Recurring events: daily, weekly, monthly, yearly, custom (RFC 5545 RRULE)
- Invite attendees: send invitations, track RSVP (accept/decline/tentative)
- Free/busy lookup: check availability of multiple people across calendars
- Calendar sharing: view-only or edit access to entire calendars
- Reminders and notifications (push, email) before events
- Time zone handling: events stored in UTC, displayed in user's timezone
- Room/resource booking: find and reserve conference rooms
- Calendar sync: CalDAV/iCal standard for external calendar apps
- Multiple calendars per user (personal, work, holidays)

---

## 2. Non-Functional Requirements (NFRs)

- **Consistency**: No double-booking of rooms (strong consistency for resources)
- **Availability**: 99.99% — calendar is always-on productivity tool
- **Low Latency**: Calendar view load < 200ms, free/busy query < 500ms
- **Sync**: Changes propagated to all devices within 5 seconds
- **Scalability**: 1B+ users, 10B+ events

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Users | 1B |
| Events per user (next 12 months) | 500 (including recurring expansions) |
| Total events | 500B (with recurring expansions) |
| Stored events (without expansion) | 50B |
| Calendar views / sec | 100K |
| Event creates / sec | 10K |
| Free/busy queries / sec | 50K |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                        │
│  Web / Mobile / Desktop / External CalDAV Clients (Outlook, Apple)    │
└───────────────────────────┬────────────────────────────────────────────┘
                            │  REST + WebSocket push + CalDAV sync
┌───────────────────────────▼────────────────────────────────────────────┐
│                    API GATEWAY                                         │
│  Auth (OAuth), rate limiting, CalDAV protocol translation             │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┬────────────────┐
         │                  │                  │                │
┌────────▼──────┐  ┌────────▼──────┐  ┌────────▼──────┐ ┌──────▼───────┐
│ Event Service │  │ Free/Busy     │  │ Notification  │ │ Room Booking │
│               │  │ Service       │  │ Service       │ │ Service      │
│ - CRUD events │  │               │  │               │ │              │
│ - RRULE       │  │ - Aggregate   │  │ - Reminders   │ │ - Available  │
│   expansion   │  │   across      │  │   (15m/1h/1d) │ │   room query │
│ - Attendee    │  │   calendars   │  │ - Invite RSVP │ │ - Capacity   │
│   management  │  │ - Bitmask     │  │ - Push/Email  │ │   check      │
│ - TZ / DST    │  │   free/busy   │  │ - iCal attach │ │ - Conflict   │
│   handling    │  │ - Slot suggest │  │               │ │   prevention │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘ └──────┬───────┘
        │                  │                  │                │
   ┌────▼──────────────────▼──────────────────▼────────────────▼────────┐
   │                        DATA LAYER                                  │
   │                                                                    │
   │ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
   │ │ PostgreSQL   │  │ Redis        │  │ Kafka        │              │
   │ │              │  │              │  │              │              │
   │ │ - events     │  │ - Free/busy  │  │ - Event      │              │
   │ │ - attendees  │  │   bitmask    │  │   change     │              │
   │ │ - recurrence │  │   per user   │  │   stream     │              │
   │ │   exceptions │  │   per day    │  │ - Notif      │              │
   │ │ - calendars  │  │ - Room avail │  │   delivery   │              │
   │ │ - rooms      │  │   cache      │  │ - CalDAV     │              │
   │ │              │  │ - Expanded   │  │   sync push  │              │
   │ │ EXCLUDE      │  │   events     │  │              │              │
   │ │ constraint   │  │   cache (30d)│  │              │              │
   │ │ for rooms    │  │              │  │              │              │
   │ └──────────────┘  └──────────────┘  └──────────────┘              │
   │                                                                    │
   └────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Event Service — RRULE Expansion
- **Why expand at query time?** "Daily standup for 2 years" = 730 rows if stored. Store ONE row with RRULE, expand when rendering calendar view
- **Expansion cache**: Redis caches expanded events for next 30 days per user. Invalidated on event change
- **Exception handling**: "Edit this occurrence" stores override; "Edit this and following" splits RRULE into two series

#### Free/Busy Service — Bitmask Algorithm
- **Why bitmask?** O(1) availability check via bitwise AND for any group of users
- **Pre-compute daily bitmask** per user: 8 hours × 4 slots/15min = 32 bits. Bit=1 means busy
- **Group query**: AND all bitmasks → 0 bits = everyone free → O(1) per time slot
- **Invalidation**: On event create/update → recompute bitmask for affected user+day

#### Room Booking — PostgreSQL Exclusion Constraint
- **Why exclusion constraint?** Database-level prevention of overlapping time ranges for same room — impossible to bypass from application code
- **How**: `EXCLUDE USING gist (room_id WITH =, tsrange(start_time, end_time) WITH &&)` — GiST index on time ranges

### Recurring Events: Storage vs Expansion

```
Wrong approach: Store every occurrence of "daily standup for 2 years" = 730 rows
Right approach: Store ONE row with recurrence rule (RRULE)

RRULE:FREQ=WEEKLY;BYDAY=MO,WE,FR;UNTIL=20261231
→ Expands at query time to show individual occurrences in the calendar view

Exception handling:
  - "Edit this occurrence": store exception (overrides that single instance)
  - "Edit this and following": split into two recurrence rules
  - "Delete this occurrence": store exclusion date (EXDATE)

Storage: 1 event row + exceptions
Query: expand RRULE in [view_start, view_end] range at read time
Cache: pre-expand next 30 days in Redis for fast calendar view
```

### Free/Busy Query (Key Algorithm)

```
Input: "Find when Alice, Bob, and Carol are all free next Tuesday 9am-5pm"

1. Fetch all events for each person on Tuesday (from cache or DB)
2. Build busy intervals per person:
   Alice: [9:00-10:00, 11:00-12:00, 14:00-15:00]
   Bob:   [9:30-10:30, 13:00-14:00]
   Carol: [10:00-11:30]
3. Merge all busy intervals → union: [9:00-12:00, 13:00-15:00]
4. Invert → free slots: [12:00-13:00, 15:00-17:00]
5. Filter by desired meeting duration (e.g., 30 min)
6. Return suggested slots

Optimization: Pre-compute daily free/busy bitmask (one bit per 15-min slot)
  8 hours × 4 slots/hr = 32 bits per day per person
  AND all bitmasks → free slots in O(1) bitwise operation
```

### Room Booking (Race Condition)

```sql
-- Atomic room reservation (prevent double-booking)
BEGIN;
SELECT COUNT(*) FROM events
WHERE room_id = 'conf-room-A'
  AND date = '2026-03-14'
  AND (start_time < '11:00' AND end_time > '10:00')  -- overlap check
FOR UPDATE;

-- If count = 0, room is free
INSERT INTO events (room_id, start_time, end_time, ...) VALUES (...);
COMMIT;

-- Alternative: Use a constraint
-- EXCLUDE USING gist (room_id WITH =, tsrange(start_time, end_time) WITH &&)
-- PostgreSQL exclusion constraint prevents overlapping ranges automatically!
```

---

## 5. APIs

```
POST   /api/events              → Create event (with recurrence, attendees)
GET    /api/events?start=...&end=...&calendar_id=...  → List events in range
PUT    /api/events/{id}          → Update event (this/all/following)
DELETE /api/events/{id}          → Delete event (this/all/following)
POST   /api/events/{id}/rsvp     → Accept/decline/tentative
GET    /api/freebusy             → Query free/busy for list of users
POST   /api/rooms/search         → Find available rooms for time slot
GET    /api/calendars            → List user's calendars
POST   /api/calendars/{id}/share → Share calendar with user/group
```

---

## 6. Data Models

### PostgreSQL

```sql
CREATE TABLE events (
    event_id       UUID PRIMARY KEY,
    calendar_id    UUID NOT NULL,
    creator_id     UUID NOT NULL,
    title          TEXT, description TEXT, location TEXT,
    start_time     TIMESTAMPTZ NOT NULL,
    end_time       TIMESTAMPTZ NOT NULL,
    timezone       TEXT DEFAULT 'UTC',
    is_all_day     BOOLEAN DEFAULT FALSE,
    recurrence     TEXT,  -- RRULE string (null for non-recurring)
    room_id        UUID,
    status         TEXT DEFAULT 'confirmed',
    visibility     TEXT DEFAULT 'default',  -- public|private|default
    created_at     TIMESTAMPTZ DEFAULT NOW(),
    updated_at     TIMESTAMPTZ DEFAULT NOW(),
    -- Exclusion constraint for room double-booking prevention
    EXCLUDE USING gist (room_id WITH =, tsrange(start_time, end_time) WITH &&)
        WHERE (room_id IS NOT NULL)
);

CREATE TABLE event_attendees (
    event_id     UUID REFERENCES events(event_id),
    user_id      UUID,
    email        TEXT,
    rsvp_status  TEXT DEFAULT 'needs-action',  -- accepted|declined|tentative
    role         TEXT DEFAULT 'attendee',  -- organizer|attendee|optional
    PRIMARY KEY (event_id, user_id)
);

CREATE TABLE recurrence_exceptions (
    event_id           UUID REFERENCES events(event_id),
    original_start     TIMESTAMPTZ,  -- which occurrence is modified
    modified_event_id  UUID,         -- replacement event (null if deleted)
    PRIMARY KEY (event_id, original_start)
);
```

---

## 7. Fault Tolerance & Deep Dives

### Room Double-Booking Prevention
```
PostgreSQL exclusion constraint → database-level guarantee
Even if two concurrent transactions try to book overlapping times:
  Transaction 1: INSERT event for Room-A, 10:00-11:00 → succeeds
  Transaction 2: INSERT event for Room-A, 10:30-11:30 → FAILS (exclusion violation)
No application-level locking needed — the database enforces it
```

### Time Zone + DST Deep Dive (The Hardest Part)
```
Problem: "9 AM every Monday" in New York
  Summer (EDT, UTC-4): 9 AM local = 13:00 UTC
  Winter (EST, UTC-5): 9 AM local = 14:00 UTC
  
Storage: Store RRULE + original timezone ("America/New_York")
Expansion: At query time, expand RRULE using IANA tz database
  March 9, 2026 (spring forward): 2 AM → 3 AM (no 2:30 AM exists)
    Event at 2:30 AM → skip? Move to 3:00 AM? (RFC 5545 says skip or adjust)
  Nov 1, 2026 (fall back): 1 AM occurs TWICE
    Event at 1:30 AM → which one? (use wall clock: first occurrence)

NEVER store computed UTC offsets in the RRULE itself
  Offsets change when DST rules change (governments update DST dates)
  Must re-expand using latest IANA tzdata at render time
```

### CalDAV Sync Protocol
```
Client A (desktop) edits event → server stores change → version incremented
Client B (phone) sends: GET /calendars/cal1?sync-token=42
  Server returns: events changed since token 42 → Client B updates local
  Efficient: only sends diffs, not full calendar

Conflict: Two devices edit same event offline
  Last-write-wins with server timestamp
  Alternative: merge non-conflicting fields (title change + time change = merge both)
```

### Invitation Delivery
```
Kafka topic: calendar-notifications
  Guaranteed delivery: RSVP emails, reminder notifications
  Retry with backoff: 1min, 5min, 30min
  iCalendar attachment (.ics): standard format for calendar invites
  All major email clients auto-detect and offer "Add to Calendar"
```

---

## 8. Key Differences from Other Systems

```
vs Ticketing (#23): Calendar manages TIME SLOTS not seats; recurring events are unique
vs Notification (#05): Calendar is the SOURCE of scheduled notifications
vs Job Scheduler (#28): Similar recurrence handling (RRULE ≈ cron), but calendar is user-facing

Unique challenges:
- RRULE expansion with exceptions is complex (RFC 5545 spec is 150+ pages)
- Time zone + DST: "9 AM every Monday" means different UTC times in summer vs winter
- Free/busy across organizations: privacy (show busy, not event details)
- Room booking: exclusion constraints prevent overlapping reservations at DB level
```

---

## 9. Deep Dive: Engineering Trade-offs

### Scalability: Sharding Calendar Data

```
1B users, 50B events → need horizontal scaling

Shard by calendar_id (≈ user_id for primary calendar):
  events table:          shard by calendar_id
  event_attendees table: shard by event_id (co-located with event)
  recurrence_exceptions: shard by event_id (co-located with event)
  
  1B users / 256 shards = ~4M users per shard
  ~200M events per shard → manageable with PostgreSQL + partitioning

  Single-user operations (most common):
    "Show my calendar for this week" → single shard query ✓
    "Create event on my calendar" → single shard write ✓
  
Cross-shard challenges:

  1. Free/busy query for 10 attendees across 10 shards:
     Naive: 10 DB queries to 10 different shards → 10 round-trips → slow
     Solution: Pre-computed free/busy bitmasks in Redis (already in design)
       Key: freebusy:{user_id}:{date}
       Value: 32-bit bitmask (one bit per 15-min slot, 8 hours)
       Free/busy query: 10 Redis GETS → bitwise AND → O(1) merge
       No cross-shard DB queries needed ✓

  2. Shared calendar (team calendar with 50 members):
     All 50 members need to see updates instantly
     Event created on shard A → notifications to users on shards B, C, D...
     Solution: Kafka topic calendar-changes → each shard's consumer
       processes events for its users → push via WebSocket
     Eventual consistency: updates visible within 2-5 seconds

  3. Room booking across organizations:
     Rooms are shared resources, not user-owned
     Solution: Separate rooms shard (low cardinality: ~100K rooms total)
       OR: single PostgreSQL instance for rooms (with exclusion constraints)
       Room booking: write to rooms DB + write to user's shard
       Two-phase: rooms DB first (authoritative), user shard second
```

### Offline Sync Conflict Resolution

```
Scenario: User edits event on phone (offline), then edits same event on laptop.
  Phone reconnects → server has a newer version from laptop.

Strategy: Field-level merge with last-write-wins per field

  Phone edit (offline, T=10:00): Changed title to "Team Standup v2"
  Laptop edit (online, T=10:05): Changed time to 10:30 AM
  
  Phone reconnects at T=10:15:
    Server compares field-by-field:
    - title: phone="Team Standup v2" (T=10:00), server="Team Standup" (T=9:00)
      → Phone is newer → accept phone's title ✓
    - time: phone=10:00 AM (T=9:00, unchanged), server=10:30 AM (T=10:05)
      → Server is newer → keep server's time ✓
    - Result: title="Team Standup v2", time=10:30 AM (merged!)

Same-field conflict:
  Phone: changed time to 11:00 AM (T=10:00)
  Laptop: changed time to 10:30 AM (T=10:05)
  
  Resolution: Last-write-wins by timestamp → laptop's 10:30 AM wins
  Notification: "Your offline edit to meeting time was overridden. 
    You set 11:00 AM, but [Laptop] changed it to 10:30 AM at 10:05."
  User can re-edit if needed.

Deletion conflict:
  Phone (offline): Deletes the event
  Laptop (online): Adds an attendee to the event
  
  Option A: Delete wins (simpler, data loss risk)
    Event is deleted. Attendee addition is discarded.
    Notify laptop user: "This event was deleted from [Phone]"
  
  Option B: Preserve with changes (user-friendlier)
    Keep the event (un-delete), apply attendee addition
    Mark as "conflict resolved — event restored"
    This is Google Calendar's approach for most cases

Recurring event conflict:
  Phone (offline): Edits "this occurrence" (March 20 standup → 11 AM)
  Laptop (online): Edits "this and all following" (all standups → 10:30 AM)
  
  Resolution:
    1. Server applies laptop's "all following" first (bulk rule change)
    2. Server applies phone's single exception on top (March 20 → 11 AM)
    3. Result: all standups move to 10:30 AM, EXCEPT March 20 which is 11 AM
    4. The exception is stored in recurrence_exceptions table

Sync protocol (CalDAV sync-token):
  Client stores last_sync_token per calendar
  On reconnect: GET /calendars/cal1?sync-token=42
  Server returns: all changes since token 42 (efficient delta sync)
  Client applies changes, resolves conflicts locally, pushes pending offline edits
  Server resolves any remaining conflicts server-side
```

### RRULE Expansion at Read-Time vs Materialized Occurrences

```
Option 1: Expand at read time (this design) ⭐
  Store: 1 row with RRULE "FREQ=WEEKLY;BYDAY=MO,WE,FR"
  Query: expand to individual occurrences when rendering calendar view
  
  ✓ Storage efficient: 1 row instead of 365+ rows per recurring event
  ✓ "Edit all future" is a single row update
  ✓ RRULE change is instant (no bulk update of 365 rows)
  ✗ CPU cost at read time (RRULE expansion library, ~1ms per event)
  ✗ Complex query: "all events on March 20" requires expanding ALL
    recurring events to check if they fall on that date
  
  Mitigation: Redis cache of expanded events for next 30 days
    Invalidation: on event create/update → recompute affected days
    Cache hit: skip expansion entirely → O(1) calendar view

Option 2: Materialized occurrences
  Store: 365 rows for "daily standup for 1 year"
  Query: simple SELECT WHERE date = 'March 20'
  
  ✓ Simple queries (no expansion logic)
  ✓ Per-occurrence modifications are just row updates
  ✗ Storage explosion: 500B+ rows for all users' recurring events
  ✗ "Edit all future" requires updating 200+ rows atomically
  ✗ RRULE change (e.g., cancel recurring) → bulk delete
  
  Used by: Outlook (internally materializes for performance)

Recommendation: Expand at read time + aggressive caching
  The storage savings (100×) and write simplicity outweigh 
  the read-time expansion cost (cached and amortized)
```

### Bitmask Granularity: 15-min vs Per-Minute

```
15-minute slots (this design):
  8-hour workday × 4 slots/hour = 32 bits per day per user
  1B users × 365 days × 4 bytes = 1.46 TB for full year
  Bitwise AND for group query: single CPU instruction
  ✓ Fixed 32-bit integer — CPU-native operations
  ✓ Sufficient for meeting scheduling (meetings rarely start at :07)
  ✗ Loses precision: 15-min meeting at 10:00-10:15 blocks entire 10:00-10:15 slot
  
Per-minute slots:
  8-hour workday × 60 slots/hour = 480 bits per day per user
  1B users × 365 days × 60 bytes = 22 TB for full year (15× more)
  Bitwise AND: 8 × 64-bit operations (still fast, but more memory)
  ✓ Precise availability for any duration
  ✗ 15× more storage and memory
  ✗ Diminishing returns — nobody schedules 7-minute meetings
  
5-minute slots (compromise):
  8 hours × 12 slots/hour = 96 bits per day per user
  Fits in 2 × 64-bit integers
  ✓ Precise enough for most use cases
  ✓ 3× more storage than 15-min, 5× less than per-minute

Recommendation: 15-minute slots (Google Calendar's approach)
  Standard meeting durations (15, 30, 45, 60 min) align perfectly
  For sub-15-min events: round up to nearest slot (conservative)
```

### PostgreSQL Exclusion Constraint vs Application-Level Locking

```
Exclusion Constraint (this design):
  EXCLUDE USING gist (room_id WITH =, tsrange(start_time, end_time) WITH &&)
  
  ✓ Database enforces invariant — impossible to bypass from any client
  ✓ No application code needed for conflict detection
  ✓ Works even if a bug in application logic skips the check
  ✗ PostgreSQL-specific (not portable to MySQL, DynamoDB)
  ✗ GiST index overhead: slower INSERTs (~2× vs plain B-tree)
  ✗ Error handling: application must catch exclusion violation exception

Application-Level Locking:
  BEGIN; SELECT ... FOR UPDATE WHERE room_id AND date_range overlap; 
  if count = 0 → INSERT; COMMIT;
  
  ✓ Works with any database
  ✓ More flexible (can add custom logic: "allow overlap if same organizer")
  ✗ Bug risk: if application forgets the check → double booking
  ✗ Race condition if isolation level is wrong (need SERIALIZABLE or careful FOR UPDATE)
  
Recommendation: Use exclusion constraint as safety net + application check
  Application does the overlap check (good UX, return "room busy" message)
  Exclusion constraint catches any bugs (database-level guarantee)
  Belt AND suspenders approach for critical data integrity
```

