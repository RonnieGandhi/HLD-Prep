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

