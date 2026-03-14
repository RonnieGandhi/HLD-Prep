# 5. Design a Notification System (Push, Email, SMS)

---

## 1. Functional Requirements (FR)

- Send notifications via multiple channels: **Push (iOS/Android/Web)**, **Email**, **SMS**
- Support both **real-time** and **scheduled** notifications
- Support **user preferences**: users choose which channels they want, per notification type
- Support **template-based** notifications with variable substitution
- Support **bulk/batch** notifications (e.g., marketing campaign to 10M users)
- Track notification delivery status: sent, delivered, read, failed, bounced
- Rate limit notifications per user to prevent spam
- Support notification grouping/bundling (e.g., "5 people liked your photo")
- Priority levels: critical (immediately), high, normal, low (batched)

---

## 2. Non-Functional Requirements (NFRs)

- **High Throughput**: Send 1M+ notifications per minute
- **Low Latency**: Real-time notifications delivered within 1-2 seconds
- **Reliability**: No notification should be lost (at-least-once delivery)
- **Scalability**: Handle 100M+ users, billions of notifications/day
- **Fault Tolerant**: If one channel (e.g., SMS provider) is down, other channels still work
- **Ordered**: Notifications for a user should arrive in chronological order (best-effort)
- **Idempotent**: Same notification should not be sent twice to the same user
- **Extensible**: Easy to add new channels (WhatsApp, Slack, etc.)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 100M |
| Notifications / day | 1B (10 per user/day average) |
| Notifications / sec | ~12K (peak 5×: ~60K) |
| Push notifications | 60% → 600M/day |
| Emails | 30% → 300M/day |
| SMS | 10% → 100M/day |
| Notification record size | ~500 bytes |
| Storage / day | 1B × 500B = 500 GB |
| Storage / year | ~180 TB |

---

## 4. High-Level Design (HLD)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Service A  │     │  Service B  │     │  Scheduler  │
│  (e.g., SN) │     │  (e.g., Order)│    │  (Cron/     │
└──────┬──────┘     └──────┬──────┘     │  Campaigns) │
       │                    │            └──────┬──────┘
       └────────────────────┼───────────────────┘
                            │
                     ┌──────▼───────┐
                     │ Notification │
                     │   Service   │   ← Validates, enriches, fans out
                     │   (API)     │
                     └──────┬──────┘
                            │
                     ┌──────▼───────┐
                     │ User Pref &  │   ← Check: does user want this channel?
                     │ Template Svc │
                     └──────┬──────┘
                            │
                     ┌──────▼───────┐
                     │    Kafka     │   ← Durable message queue
                     │  (by channel)│
                     │              │
                     │ Topics:      │
                     │  push_notif  │
                     │  email_notif │
                     │  sms_notif   │
                     └──┬──────┬──┬─┘
                        │      │  │
          ┌─────────────┘      │  └─────────────┐
          │                    │                 │
   ┌──────▼──────┐     ┌──────▼──────┐   ┌──────▼──────┐
   │ Push Worker │     │ Email Worker│   │ SMS Worker  │
   │ Pool        │     │ Pool        │   │ Pool        │
   └──────┬──────┘     └──────┬──────┘   └──────┬──────┘
          │                    │                 │
   ┌──────▼──────┐     ┌──────▼──────┐   ┌──────▼──────┐
   │   APNs     │     │  SendGrid   │   │  Twilio     │
   │   FCM      │     │  SES        │   │  Nexmo      │
   │  (Push)    │     │  (Email)    │   │  (SMS)      │
   └─────────────┘     └─────────────┘   └─────────────┘
          │                    │                 │
          └────────────────────┼─────────────────┘
                               │
                     ┌─────────▼─────────┐
                     │ Delivery Tracker  │  ← Callback/webhook from providers
                     │ (Status Updates)  │
                     └─────────┬─────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
         ┌──────▼──────┐ ┌────▼────┐  ┌──────▼──────┐
         │ Cassandra   │ │  Redis  │  │ ClickHouse  │
         │ (Notif Log) │ │ (Dedup) │  │ (Analytics) │
         └─────────────┘ └─────────┘  └─────────────┘
```

### Component Deep Dive

#### Notification Service (API Layer)
- **Purpose**: Entry point for all notification requests from internal services
- **Responsibilities**:
  1. Validate request (required fields, valid user IDs)
  2. Check user preferences (does user want push? email?)
  3. Apply rate limiting (max 10 push notifications/hour per user)
  4. Check quiet hours (don't send at 3am unless critical)
  5. Render template with variables
  6. Fan out to appropriate Kafka topics per channel
- **Idempotency**: Each notification has a `request_id`. Check Redis dedup cache before processing
- **Why separate service**: Decouples notification logic from business services. Any service just sends a notification request; this service handles all complexity

#### User Preferences & Template Service
- **User Preferences**: Stored in a user preferences DB (PostgreSQL)
  - Per notification type (e.g., "marketing", "order_update", "social")
  - Per channel (push: yes, email: yes, SMS: no)
  - Quiet hours (10pm - 8am)
  - Language preference
- **Templates**: Stored in a template DB with version control
  - Support variable substitution: `"Hello {{user_name}}, your order {{order_id}} has shipped"`
  - Templates per channel (push is short, email is HTML, SMS is 160 chars)
  - A/B testing support for template variants

#### Kafka (Message Queue)
- **Why Kafka over RabbitMQ/SQS**:
  - Massive throughput (millions of messages/sec)
  - Durable — messages persisted to disk with replication
  - Consumer groups — easy to scale workers independently per channel
  - Replay capability — if a worker has a bug, fix it and replay
- **Topics**: Separate topic per channel (`push_notifications`, `email_notifications`, `sms_notifications`)
  - Allows independent scaling of each channel's worker pool
  - If email provider is slow, email queue backs up without affecting push
- **Partitioning**: By `user_id` → ensures notifications for the same user are processed in order
- **Config**: RF=3, `min.insync.replicas=2`, retention = 7 days

#### Push Worker Pool
- **APNs (Apple Push Notification Service)**: For iOS devices. HTTP/2 persistent connections. Must handle token invalidation (user uninstalled app)
- **FCM (Firebase Cloud Messaging)**: For Android and web. REST API. Supports topic messaging for broadcast
- **Flow**:
  1. Consume from `push_notifications` topic
  2. Look up device tokens for user from Device Token DB
  3. Send to APNs/FCM
  4. Handle response: success → update status; invalid token → remove token; rate limited → retry with backoff
- **Connection pooling**: Maintain persistent HTTP/2 connections to APNs/FCM (connection setup is expensive)

#### Email Worker Pool
- **Providers**: SendGrid, AWS SES, Mailgun (use multiple for redundancy)
- **Flow**:
  1. Consume from `email_notifications` topic
  2. Render HTML template
  3. Send via primary provider (SendGrid)
  4. If primary fails → failover to secondary (SES)
  5. Track bounce/complaint callbacks via webhooks
- **Considerations**: SPF, DKIM, DMARC for deliverability. Warm up IPs for bulk sends

#### SMS Worker Pool
- **Providers**: Twilio, Nexmo/Vonage (use multiple; some are better in certain regions)
- **Flow**: Similar to email worker
- **Considerations**: SMS costs money ($0.01-0.05 per SMS). Apply strict rate limiting. Support country-specific routing (cheapest provider per country)

#### Delivery Tracker
- **Purpose**: Receive delivery receipts from providers (webhooks/callbacks)
- **Tracks**: `QUEUED → SENT → DELIVERED → READ → FAILED → BOUNCED`
- **Webhooks**: Providers call back with delivery status updates
- **Stores**: Status updates in Cassandra notification log

#### Redis (Deduplication Cache)
- **Purpose**: Prevent duplicate notifications
- **How**: `SET notification:{request_id} 1 EX 86400 NX` — if key exists, it's a duplicate
- **Also used for**: Rate limiting counters per user per channel

---

## 5. APIs

### Send Notification
```http
POST /api/v1/notifications
Authorization: Bearer <service_token>

{
  "request_id": "uuid-v4",        // idempotency key
  "user_ids": ["user123", "user456"],
  "notification_type": "order_shipped",
  "priority": "high",             // critical, high, normal, low
  "channels": ["push", "email"],  // optional; if omitted, use user prefs
  "template_id": "order_shipped_v2",
  "template_vars": {
    "order_id": "ORD-12345",
    "tracking_url": "https://track.ly/abc"
  },
  "scheduled_at": null,           // null = immediate
  "metadata": {
    "campaign_id": "spring_sale_2026"
  }
}

Response: 202 Accepted
{
  "notification_id": "notif-uuid",
  "status": "queued",
  "channels_targeted": ["push", "email"]
}
```

### Get Notification Status
```http
GET /api/v1/notifications/{notification_id}

Response: 200 OK
{
  "notification_id": "notif-uuid",
  "user_id": "user123",
  "channels": {
    "push": {"status": "delivered", "delivered_at": "..."},
    "email": {"status": "sent", "sent_at": "..."}
  }
}
```

### Update User Preferences
```http
PUT /api/v1/users/{user_id}/notification-preferences
{
  "channels": {
    "push": true,
    "email": true,
    "sms": false
  },
  "quiet_hours": {
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/New_York"
  },
  "notification_types": {
    "marketing": {"push": false, "email": true},
    "social": {"push": true, "email": false},
    "order_updates": {"push": true, "email": true, "sms": true}
  }
}
```

### Get User's Notification History
```http
GET /api/v1/users/{user_id}/notifications?page=1&limit=20

Response: 200 OK
{
  "notifications": [
    {
      "notification_id": "...",
      "type": "order_shipped",
      "title": "Your order has shipped!",
      "body": "Order ORD-12345 is on its way.",
      "channel": "push",
      "status": "read",
      "created_at": "2026-03-13T10:00:00Z"
    }
  ],
  "pagination": {"page": 1, "total": 150}
}
```

---

## 6. Data Model

### PostgreSQL — User Preferences

```sql
CREATE TABLE user_notification_preferences (
    user_id             UUID PRIMARY KEY,
    push_enabled        BOOLEAN DEFAULT TRUE,
    email_enabled       BOOLEAN DEFAULT TRUE,
    sms_enabled         BOOLEAN DEFAULT FALSE,
    quiet_hours_start   TIME,
    quiet_hours_end     TIME,
    timezone            VARCHAR(64),
    language            VARCHAR(10) DEFAULT 'en',
    updated_at          TIMESTAMP
);

CREATE TABLE user_type_preferences (
    user_id             UUID,
    notification_type   VARCHAR(64),
    push_enabled        BOOLEAN DEFAULT TRUE,
    email_enabled       BOOLEAN DEFAULT TRUE,
    sms_enabled         BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (user_id, notification_type)
);
```

**Why PostgreSQL**: Relational data with clear schema, strong consistency needed for preferences.

### Cassandra — Notification Log

```sql
CREATE TABLE notification_log (
    user_id           UUID,
    created_at        TIMESTAMP,
    notification_id   UUID,
    notification_type VARCHAR,
    channel           VARCHAR,
    title             TEXT,
    body              TEXT,
    status            VARCHAR,   -- queued, sent, delivered, read, failed
    metadata          MAP<TEXT, TEXT>,
    PRIMARY KEY (user_id, created_at, notification_id)
) WITH CLUSTERING ORDER BY (created_at DESC)
  AND default_time_to_live = 7776000;  -- 90 days retention
```

**Why Cassandra**: High write throughput (billions of notifications), time-series access pattern (user's recent notifications), TTL support.

### Redis — Dedup & Rate Limiting

```
# Deduplication
Key:    notif:dedup:{request_id}
Value:  1
TTL:    86400 (24 hours)

# Rate limiting (per user per channel)
Key:    notif:rate:{user_id}:{channel}:{hour}
Value:  counter (INCR)
TTL:    3600
```

### Redis — Device Tokens

```
Key:    device_tokens:{user_id}
Value:  SET of {device_token, platform, app_version, last_active}
```

Alternatively, store in PostgreSQL if you need complex queries.

### Kafka Topics

```
Topic: push_notifications    (partitioned by user_id)
Topic: email_notifications   (partitioned by user_id)
Topic: sms_notifications     (partitioned by user_id)
Topic: notification_status   (delivery status callbacks)

Message Schema (push_notifications):
{
  "notification_id": "uuid",
  "user_id": "user123",
  "title": "Your order has shipped!",
  "body": "Order ORD-12345 is on its way.",
  "data": {"order_id": "ORD-12345", "deep_link": "app://orders/12345"},
  "priority": "high",
  "created_at": "2026-03-13T10:00:00Z"
}
```

### MySQL — Notification Templates

```sql
CREATE TABLE notification_templates (
    template_id     VARCHAR(128) PRIMARY KEY,
    version         INT,
    channel         ENUM('push', 'email', 'sms'),
    subject         TEXT,           -- for email
    title           TEXT,           -- for push
    body_template   TEXT,           -- "Hello {{user_name}}, ..."
    html_template   TEXT,           -- for email HTML
    language        VARCHAR(10),
    active          BOOLEAN,
    created_at      TIMESTAMP,
    UNIQUE KEY (template_id, version, channel, language)
);
```

---

## 7. Fault Tolerance

### General
| Technique | Application |
|---|---|
| **Kafka durability** | RF=3, min.insync.replicas=2. Notifications survive broker failures |
| **Consumer group rebalancing** | If a push worker dies, Kafka rebalances partitions to surviving workers |
| **Retry with exponential backoff** | On provider errors (APNs, SendGrid timeouts) |
| **Dead Letter Queue (DLQ)** | After N retries, move to DLQ for manual investigation |
| **Idempotent processing** | Dedup by notification_id in Redis before sending |
| **Circuit breaker** | Per provider — if Twilio is down, stop sending to it, alert ops |

### Problem-Specific Fault Tolerance

1. **Provider Outage (e.g., SendGrid is down)**
   - Circuit breaker trips after 5 consecutive failures
   - Automatic failover to secondary provider (AWS SES)
   - Provider abstraction layer makes switching transparent
   - Queue keeps growing → process backlog when provider recovers

2. **Push Token Invalidation**
   - APNs/FCM return "invalid token" (user uninstalled app)
   - Worker removes the invalid token from device token store
   - If all tokens invalid → can't send push; fall back to email/SMS based on preferences

3. **Duplicate Notifications**
   - Kafka consumer commits offset after processing → if crash before commit, message re-processed
   - **Solution**: Check Redis dedup cache (`notification_id`) before calling provider
   - Also: providers themselves deduplicate (APNs has `apns-collapse-id`)

4. **Notification Storm (Bulk Campaign)**
   - Marketing sends a campaign to 50M users at once
   - **Solution**: 
     - Separate Kafka topic/partition for bulk vs. transactional notifications
     - Bulk notifications are throttled (processed at controlled rate)
     - Transactional notifications (order confirmation) always prioritized

5. **User Device Offline**
   - Push notification is sent to APNs/FCM but device is offline
   - APNs/FCM handle this — they store the notification and deliver when device comes online
   - We track status as "sent" (not "delivered") until device acks

---

## 8. Additional Considerations

### Notification Grouping / Bundling
- Instead of "User A liked your photo", "User B liked your photo" × 50 times
- Group into: "User A, User B, and 48 others liked your photo"
- **Implementation**: Hold notifications in a buffer (Redis sorted set by user_id) for 5 minutes. If multiple notifications of same type arrive, merge them. Timer triggers the bundled notification

### Priority Queue Implementation
```
Kafka topics by priority:
  notifications_critical  → immediate processing, dedicated worker pool
  notifications_high      → normal processing
  notifications_normal    → best-effort
  notifications_low       → batched processing (hourly digest)
```

### Analytics
- Track delivery rate per channel, per provider
- Track open rates, click-through rates for emails
- Track notification-to-action conversion
- Store in ClickHouse for dashboarding

### Quiet Hours / Timezone Handling
- Store user's timezone in preferences
- Before sending, check if current time in user's timezone is within quiet hours
- If yes → schedule for delivery at quiet hours end (unless priority = critical)
- Use a **scheduled notification queue** backed by a distributed scheduler

### Unsubscribe / Compliance
- Every email must have an unsubscribe link (CAN-SPAM / GDPR)
- SMS requires opt-in (TCPA compliance)
- Push can be disabled at OS level (handle gracefully)
- Maintain a suppression list (bounced emails, unsubscribed users)

---

## 9. Deep Dive: Engineering Trade-offs

### Push vs Pull for Notification Delivery

```
Push (Server-initiated) ⭐:
  Server pushes notification to client via APNs/FCM/WebSocket
  ✓ Real-time delivery (< 1 second)
  ✓ No wasted bandwidth (only sent when there's something to send)
  ✗ Requires persistent connection or OS-level push service
  ✗ APNs/FCM can throttle or drop notifications under load
  Best for: Real-time alerts, chat messages, critical notifications

Pull (Client-initiated):
  Client polls server: "Any new notifications?"
  ✓ Simple server-side implementation
  ✓ Client controls frequency
  ✗ Wastes bandwidth (most polls return nothing)
  ✗ Latency = poll interval (if polling every 30s, avg delay = 15s)
  Best for: Non-real-time (email digests, weekly summaries)

Hybrid:
  Push a lightweight "you have notifications" signal → client pulls full details
  ✓ Push is tiny (no payload, just a trigger)
  ✓ Client fetches exactly what it needs
  Used by: Instagram, Twitter (push wake-up → pull feed)
```

### Why Kafka Per Channel (Not a Single Topic)?

```
Single topic approach:
  All notifications → one "notifications" topic → one consumer group
  
  Problem: SMS delivery takes 2s; push takes 50ms; email takes 500ms
  A burst of 100K emails BLOCKS push notifications waiting in the same queue
  
Per-channel topics ⭐:
  notifications_push   → fast consumers (50ms avg)
  notifications_email  → medium consumers (500ms avg)
  notifications_sms    → slow consumers (2s avg)
  
  Each channel scales independently:
    Push: 20 consumers (high volume, fast)
    Email: 50 consumers (high volume, medium speed)
    SMS: 10 consumers (lower volume, slow provider)
  
  Bonus: If email provider is down, only email topic backs up.
         Push and SMS continue normally. Circuit breaker per channel.
```

### At-Least-Once vs Exactly-Once vs At-Most-Once

```
At-Most-Once:
  Send and forget. If delivery fails, don't retry.
  ✓ No duplicates ever
  ✗ Notifications can be lost
  Use for: Marketing, non-critical (better to miss than annoy with duplicates)

At-Least-Once ⭐ (Recommended for most):
  Retry on failure. May result in duplicates.
  ✓ No notification is ever lost
  ✗ User might get "You have a new message" twice
  Mitigation: Client-side deduplication by notification_id
  Use for: Transactional (order confirmation, OTP, alerts)

Exactly-Once:
  Guarantee each notification delivered exactly once.
  ✗ Extremely hard in distributed systems (requires idempotency + dedup)
  ✗ Higher latency (need to check dedup before every delivery)
  Implementation: 
    Store notification_id in Redis SET → before sending, check if already sent
    TTL = 24 hours (dedup window)
  Use for: Financial alerts, critical one-time codes
```

### Provider Failover Strategy

```
For each channel, maintain primary and fallback providers:
  Email:  Primary: AWS SES  → Fallback: SendGrid → Fallback: Mailgun
  SMS:    Primary: Twilio   → Fallback: AWS SNS  → Fallback: MessageBird
  Push:   Primary: APNs/FCM → (no fallback — platform-specific)

Failover logic:
  try:
    primary_provider.send(notification)
  except ProviderError, Timeout:
    circuit_breaker.record_failure(primary)
    if circuit_breaker.is_open(primary):
      fallback_provider.send(notification)
    else:
      retry(primary, max_retries=2, backoff=exponential)

Circuit breaker thresholds:
  - Open after 5 consecutive failures OR > 50% error rate in last 60s
  - Half-open after 30 seconds → try 1 request → if success, close
  - Closed (healthy) → normal operation
  
Why not always use the cheapest provider?
  - Reliability varies (provider-specific outages)
  - Deliverability differs (SES has better inbox placement than some)
  - Regional performance (Twilio better in US, MessageBird better in EU)
  - Cost optimization: Route 80% through cheapest, 20% through most reliable
```

### Template Engine: Server-Side vs Client-Side Rendering

```
Server-side rendering (compile template + data → final content):
  Template: "Hi {{name}}, your order #{{order_id}} is confirmed!"
  Data: {name: "Alice", order_id: "12345"}
  Output: "Hi Alice, your order #12345 is confirmed!"
  
  ✓ Works for all channels (email, SMS, push — all get final content)
  ✓ Consistent rendering regardless of client
  ✗ Template changes require redeployment (unless stored in DB)
  
  Best practice: Store templates in DB (versioned), compile at runtime
  Cache compiled templates in Redis (TTL = 5 min)

Client-side rendering:
  Send template + data separately → client renders
  ✓ Reduces payload size (template cached on client)
  ✗ Only works for push/in-app (not email/SMS)
  ✗ Client must handle rendering logic
```
