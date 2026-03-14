# 36. Design an On-Call Escalation System (like PagerDuty / OpsGenie)

---

## 1. Functional Requirements (FR)

- **Alert ingestion**: Receive alerts from monitoring systems (Prometheus, Datadog, CloudWatch)
- **On-call schedules**: Define rotation schedules (weekly, daily) with primary and secondary
- **Escalation policies**: If primary doesn't acknowledge within N minutes → escalate to secondary → then to manager
- **Multi-channel notification**: Alert via push, SMS, phone call, email, Slack
- **Acknowledge / Resolve**: On-call engineer ACKs alert (stops escalation) or resolves it
- **Alert grouping**: Group related alerts (same service, same root cause) to prevent noise
- **Maintenance windows**: Suppress alerts during planned maintenance
- **Incident management**: Create incidents from alerts, assign severity, track resolution timeline

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-High Availability**: 99.999% — if PagerDuty is down, nobody gets alerted
- **Low Latency**: Alert → notification in < 30 seconds
- **Reliable Delivery**: Notification MUST reach the on-call person (multi-channel fallback)
- **Scalability**: Handle 100K+ alerts/min during cascading failures
- **Exactly-Once Alerting**: Don't alert the same person multiple times for the same incident
- **Audit Trail**: Complete log of who was alerted, when they ACKed, escalation timeline

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Alerts / day | 10M |
| Alerts / sec | ~115 (peak 10K during cascading failures) |
| Active on-call schedules | 50K (teams) |
| Notifications / day | 5M (alerts × channels) |
| Escalations / day | 500K |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│              Alert Sources (External Monitoring Systems)              │
│                                                                       │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────────────┐  │
│  │Prometheus │  │ Datadog   │  │CloudWatch │  │Custom Webhooks   │  │
│  │AlertMgr   │  │ Monitors  │  │ Alarms    │  │(HTTP POST)       │  │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └────────┬─────────┘  │
└────────┼───────────────┼───────────────┼─────────────────┼────────────┘
         │  webhook      │  webhook      │  webhook        │
         ▼               ▼               ▼                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Alert Ingestion Service                            │
│  (Stateless, auto-scaled, behind API Gateway)                        │
│                                                                       │
│  1. Authenticate source (API key per integration)                    │
│  2. Normalize to internal schema (different sources have             │
│     different formats: Prometheus labels, Datadog tags, etc.)         │
│  3. Dedup check: SETNX alert:dedup:{dedup_key} (Redis, TTL=24h)    │
│     If exists → merge into existing alert (increment occurrence count)│
│  4. Persist to PostgreSQL (status='triggered')                       │
│  5. Publish to Kafka topic: "alert-events"                           │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                        ┌──────▼──────┐
                        │    Kafka    │
                        │alert-events │
                        └──────┬──────┘
                               │
              ┌────────────────┼─────────────────┐
              │                │                  │
     ┌────────▼────────┐ ┌────▼──────────┐ ┌─────▼───────────┐
     │  Alert Grouper  │ │  Alert Router │ │ Analytics       │
     │  & Correlator   │ │               │ │ Consumer        │
     │                 │ │  1. Lookup    │ │                 │
     │  Group by:      │ │     team by   │ │  Write to       │
     │  • alert_name   │ │     service   │ │  ClickHouse     │
     │  • labels       │ │     routing   │ │  for reporting  │
     │  • 5-min window │ │     rules     │ │                 │
     │                 │ │  2. Lookup    │ │  MTTA, MTTR,    │
     │  If grouped:    │ │     on-call   │ │  alert volume,  │
     │  → merge into   │ │     schedule  │ │  escalation     │
     │    one incident │ │  3. Find who  │ │  rate            │
     │                 │ │     to page   │ │                 │
     └────────┬────────┘ └────┬──────────┘ └─────────────────┘
              │               │
              └───────┬───────┘
                      │
         ┌────────────▼────────────────────────────────────┐
         │           Escalation Engine                      │
         │  (Stateful — distributed across N instances     │
         │   with leader election per alert)                │
         │                                                  │
         │  State Machine per alert:                        │
         │  ┌──────────┐  timeout  ┌───────────┐           │
         │  │TRIGGERED │──────────▶│ ESCALATED │           │
         │  │(Level 0) │  5 min    │ (Level 1) │           │
         │  └────┬─────┘           └─────┬─────┘           │
         │       │ ACK                    │ ACK             │
         │  ┌────▼─────┐           ┌─────▼─────┐           │
         │  │  ACKED   │           │  ACKED    │           │
         │  └────┬─────┘           └─────┬─────┘           │
         │       │ resolve                │ resolve         │
         │  ┌────▼─────┐           ┌─────▼─────┐           │
         │  │ RESOLVED │           │ RESOLVED  │           │
         │  └──────────┘           └───────────┘           │
         │                                                  │
         │  Timer backing: Redis Sorted Set                 │
         │  ZADD escalation_timers {fire_epoch} {alert:lvl} │
         └─────────────────┬────────────────────────────────┘
                           │
              ┌────────────▼────────────────────┐
              │     Notification Router          │
              │                                  │
              │  Route by channel + priority:    │
              │  ┌───────────────────────────┐   │
              │  │ Push (APNs/FCM)  ← first  │   │
              │  │ SMS (Twilio)     ← backup │   │
              │  │ Phone Call (Twilio Voice)  │   │
              │  │ Email (SES)     ← non-urg.│   │
              │  │ Slack (@channel) ← team   │   │
              │  └───────────────────────────┘   │
              │                                  │
              │  Multi-channel strategy:         │
              │  Level 0: push + SMS             │
              │  Level 1: push + SMS + call      │
              │  Level 2: call (must confirm)     │
              │  Level 3: call + SMS + email      │
              └────────────────┬─────────────────┘
                               │
              ┌────────────────▼────────────────┐
              │      PostgreSQL (Primary)       │
              │                                  │
              │  • alerts table                  │
              │  • escalation_policies           │
              │  • on_call_schedules             │
              │  • notification_log              │
              │  • incidents                     │
              │  • audit_trail (immutable)       │
              └────────────────┬────────────────┘
                               │ sync replication
              ┌────────────────▼────────────────┐
              │   PostgreSQL (Standby — DR)      │
              │   Multi-region for 99.999%       │
              └─────────────────────────────────┘
```

### Component Deep Dive

#### Escalation Engine — The Core State Machine

```
Alert received for team "backend-infra":

Escalation Policy:
  Level 0: Primary on-call (Alice) — notify via push + SMS. Wait 5 min.
  Level 1: Secondary on-call (Bob) — notify via push + SMS + phone call. Wait 10 min.
  Level 2: Engineering Manager (Carol) — notify via phone call. Wait 15 min.
  Level 3: VP of Engineering (Dave) — notify via phone call + SMS.

Timeline:
  T=0:00  Alert arrives → notify Alice (push + SMS)
  T=5:00  No ACK from Alice → escalate to Level 1 → notify Bob
  T=15:00 No ACK from Bob → escalate to Level 2 → notify Carol
  T=30:00 No ACK from Carol → escalate to Level 3 → notify Dave
  
  At any point: if someone ACKs → stop escalation timer
```

**Timer Implementation**:
```
Redis Sorted Set: ZADD escalation_timers {fire_time} {alert_id}:{level}
  
Worker loop (every 10 seconds):
  due = ZRANGEBYSCORE escalation_timers 0 {now}
  for each due item:
    escalate(alert_id, level)
    ZREM escalation_timers {item}
    if next_level exists:
      ZADD escalation_timers {now + next_timeout} {alert_id}:{next_level}
```

#### Alert Grouping (Noise Reduction)

```
During a database outage:
  Alert 1: "DB connection timeout on service A" at T=0
  Alert 2: "DB connection timeout on service B" at T=2s
  Alert 3: "DB connection timeout on service C" at T=5s
  ... 500 more alerts in next minute

Without grouping: On-call gets 500 pages → alert fatigue → ignores real issues
With grouping: Group by {alert_name + labels} within a 5-minute window
  → One incident: "DB connection timeout (503 occurrences across 50 services)"
  → One page, one ACK resolves all
```

---

## 5. APIs

### Ingest Alert
```http
POST /api/v1/alerts
{
  "source": "prometheus",
  "alert_name": "HighCPUUsage",
  "severity": "critical",
  "service": "payment-service",
  "labels": {"env": "production", "cluster": "us-east-1"},
  "description": "CPU usage > 90% for 5 minutes",
  "dedup_key": "high_cpu_payment_us_east"
}
```

### Acknowledge Alert
```http
POST /api/v1/alerts/{alert_id}/acknowledge
{ "acknowledged_by": "alice@company.com" }
```

### Get On-Call Schedule
```http
GET /api/v1/schedules/{team_id}/on-call?at=2026-03-14T10:00:00Z
→ { "primary": "alice@company.com", "secondary": "bob@company.com" }
```

---

## 6. Data Model

### PostgreSQL

```sql
CREATE TABLE alerts (
    alert_id        UUID PRIMARY KEY,
    dedup_key       VARCHAR(256),
    alert_name      VARCHAR(256),
    severity        VARCHAR(20),
    service         VARCHAR(128),
    labels          JSONB,
    description     TEXT,
    status          VARCHAR(20) DEFAULT 'triggered', -- triggered, acknowledged, resolved
    escalation_level INT DEFAULT 0,
    team_id         UUID,
    acknowledged_by VARCHAR(256),
    acknowledged_at TIMESTAMP,
    resolved_at     TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE (dedup_key) WHERE status != 'resolved'  -- prevent duplicate active alerts
);

CREATE TABLE escalation_policies (
    policy_id       UUID PRIMARY KEY,
    team_id         UUID,
    levels          JSONB,  -- [{level: 0, targets: ["primary"], channels: ["push","sms"], timeout_min: 5}, ...]
);

CREATE TABLE on_call_schedules (
    schedule_id     UUID PRIMARY KEY,
    team_id         UUID,
    rotation_type   VARCHAR(20),  -- weekly, daily, custom
    participants    JSONB,        -- [{user_id, start, end}, ...]
    timezone        VARCHAR(64)
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Escalation engine failure** | Multiple instances with leader election. Timer state in Redis (persistent) |
| **Notification delivery failure** | Multi-channel: if push fails → SMS → phone call. Retry each channel 3× |
| **Alert storm** | Rate limiting + grouping. Max 10 pages/hour per person |
| **Schedule gap (nobody on-call)** | System detects gap → alerts the team admin; fallback to manager |
| **Timer drift** | Redis sorted set is clock-based; NTP sync across instances |
| **System-wide outage** | Multi-region deployment. If primary region down, secondary takes over |

### Race Conditions and Failure Challenges

#### 1. ACK Race — Two Responders ACK Simultaneously

```
Alice (primary) and Bob (secondary) both receive the alert.
Alice clicks ACK at T=5:00.000 and Bob clicks ACK at T=5:00.050.

Without coordination:
  Both ACKs hit the database → both succeed → both think they own it
  Alice starts debugging, Bob starts debugging → wasted effort

Solution: Atomic ACK with CAS
  UPDATE alerts SET status = 'acknowledged', acknowledged_by = 'alice'
  WHERE alert_id = ? AND status = 'triggered';
  
  rows_affected = 1 for Alice (first ACK wins)
  rows_affected = 0 for Bob (already acknowledged)
  → Bob receives "Already acknowledged by Alice" response
  → Escalation timer cancelled
```

#### 2. Alert Storm During Cascading Failure

```
Database goes down → 500 services start failing → each generates alerts
  → 500 alerts/sec flood the system → 500 pages to on-call → ALERT FATIGUE

Solution: Multi-level grouping
  
  Level 1: Dedup by dedup_key (exact match)
    Same alert firing repeatedly → only one active alert
  
  Level 2: Group by {alert_name, severity} within 5-minute window
    "DB connection timeout" × 50 services → one incident
    
  Level 3: Group by root cause (correlation)
    If alert "DB down" and alert "connection timeout" fire together →
    correlate as one incident (DB is the root cause)
    
  Level 4: Rate limiting per on-call person
    Max 10 pages/hour. After 10 → aggregate into hourly summary
    Exception: P0/critical alerts always delivered immediately
```

#### 3. Schedule Gap — Nobody On-Call

```
Friday 6pm: Alice's on-call shift ends
Monday 9am: Bob's on-call shift starts
Saturday 3am: Alert fires → WHO gets paged?

Detection: Schedule validation job runs daily
  For each team: scan next 7 days for gaps
  If gap detected → notify team admin
  
Fallback chain:
  1. Check if there's a secondary on-call → page them
  2. Check engineering manager for the team → page them
  3. Last resort: page the VP of Engineering
  4. Never: let an alert go unnoticed
```

#### 4. Timer Precision — Escalation Fires Too Early or Too Late

```
Escalation timeout: 5 minutes. Timer stored in Redis sorted set.
Redis sorted set is checked every 10 seconds.

Worst case: Timer fires at 5 minutes + 10 seconds (one check interval late)
  → Is 10 seconds acceptable? For on-call alerting, YES.
  → If stricter needed: reduce check interval to 1 second (more Redis load)

But what if the timer worker crashes?
  Timer state is in Redis (persistent with AOF)
  New worker starts → reads all due timers → fires them
  Potential double-fire: Timer fires, worker crashes before removing from set
  → Next worker fires the same timer again → duplicate notification
  
  Solution: Idempotent notifications. Each notification has a unique ID.
  Before sending: check if notification already sent (Redis SET with NX)
```

---

## 8. Deep Dive: Engineering Trade-offs

### Timer Implementation: Redis Sorted Set vs DB Polling vs Kafka Delayed Messages

```
Redis Sorted Set ⭐ (recommended):
  ZADD with fire_time as score → ZRANGEBYSCORE for due timers
  ✓ O(log N) insert, O(1) pop
  ✓ Sub-second precision
  ✗ Volatile (need persistence config)

DB Polling:
  SELECT * FROM alerts WHERE next_escalation_at < NOW() AND status = 'triggered'
  ✓ Durable
  ✗ Polling interval = latency floor (poll every 30s = up to 30s delay)
  ✗ DB load from frequent polling

Kafka Delayed Messages:
  Publish to delay topic with headers indicating fire time
  ✗ Kafka doesn't natively support delayed delivery
  ✗ Complex workaround with multiple delay buckets

Best: Redis Sorted Set with periodic checkpointing to PostgreSQL
```

### Phone Call ACK — The Most Reliable Channel

```
Push notification: User's phone is on silent → MISSED
SMS: User's phone is in airplane mode → MISSED (delivered later)
Phone call: RINGS until answered (even on silent on most phones)

Phone call implementation (Twilio Voice):
  1. Initiate call via Twilio API
  2. Twilio calls the on-call engineer
  3. Twilio plays TTS (Text-to-Speech):
     "Alert: Payment service error rate above 5%. 
      Press 1 to acknowledge. Press 2 to escalate."
  4. Engineer presses 1 → Twilio webhook → our API → ACK alert
  5. If no answer / voicemail → auto-escalate to next level
  6. If engineer presses 2 → manual escalate to next level
  
  Retry: If call fails (busy, network error) → retry after 60 seconds
  Max retries: 3 per level before auto-escalating
  
  Why phone calls are critical:
    PagerDuty data shows phone calls have 95%+ ACK rate vs 70% for push
    At 3am, phone call is the ONLY thing that wakes most people up
```

### Schedule Override and Swap Handling

```
Scenario: Alice is primary on-call but has a doctor appointment Tuesday 2-4pm.
  She creates an override: "Bob covers for me Tue 2-4pm"

Data model:
  on_call_schedules: Weekly rotation (Alice Mon-Fri, Bob Fri-Mon)
  overrides: [{user: "bob", start: "Tue 14:00", end: "Tue 16:00", covers_for: "alice"}]

Resolution order (who is on-call RIGHT NOW?):
  1. Check overrides (highest priority) → if override exists, use it
  2. Check schedule rotations → use the scheduled person
  3. Fallback to engineering manager
  
  Override types:
    • Temporary override: one-time (doctor appointment)
    • Recurring override: every Tuesday (standing meeting)
    • Swap: Alice and Bob agree to swap shifts (both schedules adjusted)
```

### Reliability: Why 99.999% Availability Matters

```
99.99% = 52 minutes downtime/year → 1 minute of missed pages could cost millions
99.999% = 5 minutes downtime/year → PagerDuty's actual SLA target

How to achieve 99.999%:
  1. Multi-region active-active deployment (us-east + eu-west)
  2. If primary region is unreachable → secondary takes over within 30 seconds
  3. Health checks: each region monitors the other
  4. Independent notification providers:
     Push: APNs + FCM (different infrastructure)
     SMS: Twilio + MessageBird (different providers, different telco routes)
     Call: Twilio + Vonage (different providers)
  5. If ALL digital channels fail → send email (always works, just slower)
  
  Chaos engineering: Regularly simulate:
    • Provider outage (Twilio down → failover to MessageBird)
    • Region failure (us-east down → eu-west handles all alerts)
    • Timer worker crash (new worker picks up pending timers)
```

