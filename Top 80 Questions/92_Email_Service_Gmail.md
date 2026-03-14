# 92. Design an Email Service (like Gmail)

---

## 1. Functional Requirements (FR)

- Send and receive emails (SMTP) with attachments (up to 25 MB)
- Inbox, Sent, Drafts, Spam, Trash folders + custom labels/folders
- Full-text search across all emails (subject, body, sender, attachments)
- Conversation threading: group related emails into threads
- Spam filtering using ML + rule-based system
- Push notifications for new emails
- Rich text compose (HTML email) with inline images
- Contact management and autocomplete
- Filters and rules: auto-label, auto-archive, auto-forward
- Calendar integration (event invitations, RSVP)

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99% — email is mission-critical
- **Durability**: Zero email loss (once accepted for delivery)
- **Low Latency**: Email delivery < 5 seconds (within same system)
- **Search Performance**: Full-text search < 500ms across millions of emails
- **Scalability**: 1B+ users, 100B+ stored emails
- **Security**: TLS in transit, encryption at rest, phishing detection

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Users | 1B |
| Emails sent / day | 300B (50% spam) |
| Emails received per user / day | 50 (after spam filtering) |
| Avg email size | 50 KB (body) + 200 KB avg attachment |
| Storage per user | 15 GB |
| Total storage | 1B × 15 GB = 15 EB |
| Emails / sec (inbound) | 3.5M |
| Search queries / sec | 100K |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    EMAIL FLOW                                          │
│                                                                        │
│  SENDING:                                                              │
│  User composes → SMTP Service → Queue → Delivery Pipeline             │
│                                                                        │
│  RECEIVING:                                                            │
│  External SMTP → MX Records → Inbound SMTP → Spam Filter              │
│  → Store in Mailbox → Index → Push Notification                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  INBOUND PATH                                                          │
│                                                                        │
│  External MTA ──SMTP──► ┌─────────────────────┐                       │
│                          │  MX Gateway          │                       │
│                          │  (SMTP receiver)      │                       │
│                          │  - TLS/STARTTLS       │                       │
│                          │  - SPF/DKIM/DMARC     │                       │
│                          │    validation          │                       │
│                          │  - Rate limiting       │                       │
│                          │  - Connection-level    │                       │
│                          │    spam filtering      │                       │
│                          └──────────┬────────────┘                       │
│                                     │                                    │
│                          ┌──────────▼────────────┐                       │
│                          │  Spam & Phishing       │                       │
│                          │  Filter                │                       │
│                          │                        │                       │
│                          │  Layer 1: Rules        │                       │
│                          │  - Blocklists (IP/domain)│                     │
│                          │  - SPF/DKIM fail        │                       │
│                          │  - Known spam patterns  │                       │
│                          │                        │                       │
│                          │  Layer 2: ML Model     │                       │
│                          │  - NLP on subject+body │                       │
│                          │  - Sender reputation   │                       │
│                          │  - Link analysis       │                       │
│                          │  - Image analysis      │                       │
│                          │  - User feedback (not  │                       │
│                          │    spam / is spam)      │                       │
│                          │                        │                       │
│                          │  Result: HAM / SPAM    │                       │
│                          │  Score: 0.0-1.0        │                       │
│                          └──────────┬────────────┘                       │
│                                     │                                    │
│                          ┌──────────▼────────────┐                       │
│                          │  Mail Delivery Agent   │                       │
│                          │  - Store email blob    │                       │
│                          │    (Blob Store / S3)   │                       │
│                          │  - Store metadata      │                       │
│                          │    (Bigtable/Cassandra)│                       │
│                          │  - Index for search    │                       │
│                          │    (Elasticsearch)     │                       │
│                          │  - Apply user filters  │                       │
│                          │  - Push notification   │                       │
│                          └───────────────────────┘                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  OUTBOUND PATH                                                         │
│                                                                        │
│  Client ──REST/SMTP──► ┌──────────────────────┐                        │
│                         │  Submission Service   │                        │
│                         │  - Auth (OAuth 2.0)    │                        │
│                         │  - Compose validation  │                        │
│                         │  - Attachment virus scan│                       │
│                         │  - Rate limiting        │                        │
│                         └──────────┬─────────────┘                        │
│                                    │                                     │
│                         ┌──────────▼─────────────┐                        │
│                         │  Outbound Queue (Kafka) │                        │
│                         └──────────┬─────────────┘                        │
│                                    │                                     │
│                         ┌──────────▼─────────────┐                        │
│                         │  SMTP Sender            │                        │
│                         │  - DNS MX lookup         │                        │
│                         │  - DKIM signing          │                        │
│                         │  - TLS connection to     │                        │
│                         │    recipient MTA         │                        │
│                         │  - Retry with backoff    │                        │
│                         │    (up to 72h per RFC)   │                        │
│                         │  - Bounce handling       │                        │
│                         └────────────────────────┘                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  STORAGE LAYER                                                         │
│                                                                        │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐        │
│  │ Metadata DB    │  │ Blob Store      │  │ Search Index     │        │
│  │ (Bigtable/     │  │ (S3 / GCS)      │  │ (Elasticsearch)  │        │
│  │  Cassandra)    │  │                 │  │                  │        │
│  │                │  │ - Email bodies   │  │ - Subject, body  │        │
│  │ - user_id      │  │ - Attachments   │  │ - From, to       │        │
│  │ - email_id     │  │ - Inline images │  │ - Labels, dates  │        │
│  │ - subject      │  │ - Keyed by      │  │ - Attachment name│        │
│  │ - from/to      │  │   email_id      │  │ - Per-user index │        │
│  │ - labels/folder│  │                 │  │   partition      │        │
│  │ - thread_id    │  │ Deduplication:  │  │                  │        │
│  │ - snippet      │  │ same attachment │  │                  │        │
│  │ - is_read      │  │ sent to 100     │  │                  │        │
│  │ - timestamp    │  │ people → store  │  │                  │        │
│  │ - blob_ref     │  │ once (content   │  │                  │        │
│  │                │  │ hash dedup)     │  │                  │        │
│  └────────────────┘  └─────────────────┘  └──────────────────┘        │
│                                                                        │
│  ┌────────────────┐                                                    │
│  │ Redis Cache    │                                                    │
│  │ - Recent emails (inbox top 50)                                     │
│  │ - Unread count per label                                           │
│  │ - Contact autocomplete trie                                        │
│  └────────────────┘                                                    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 5. APIs

```
# Email operations
GET    /api/messages?label=INBOX&page_token=...    → List emails
GET    /api/messages/{id}                          → Get email (metadata + body)
POST   /api/messages/send                          → Send email
PUT    /api/messages/{id}/labels                   → Add/remove labels
PUT    /api/messages/{id}/read                     → Mark read/unread
DELETE /api/messages/{id}                          → Move to trash
POST   /api/messages/{id}/reply                    → Reply
POST   /api/messages/{id}/forward                  → Forward
GET    /api/threads/{thread_id}                    → Get conversation thread
GET    /api/messages/search?q=from:alice+subject:meeting  → Full-text search

# Draft and attachment
POST   /api/drafts                                 → Save draft
PUT    /api/drafts/{id}                            → Update draft
POST   /api/messages/{id}/attachments              → Upload attachment
GET    /api/messages/{id}/attachments/{att_id}     → Download attachment
```

---

## 6. Data Models

### Email Metadata (Bigtable / Cassandra)

**Why Bigtable/Cassandra?** Wide-column, partitioned by user_id, time-sorted, handles billions of rows.

```
Row key: user_id#reverse_timestamp#email_id
  (reverse timestamp: newest emails first → efficient inbox fetch)

Columns:
  subject, from, to[], cc[], bcc[], snippet (first 200 chars),
  thread_id, labels[] (INBOX, STARRED, IMPORTANT, custom),
  is_read, is_starred, has_attachment, body_blob_ref,
  attachment_refs[], size_bytes, received_at, internal_date
```

### Conversation Threading

```
Algorithm (Gmail-style):
  Thread ID = hash of normalized subject + participants
  OR use In-Reply-To and References headers (RFC 2822)
  
  New email with same subject + participants → added to existing thread
  Thread view: fetch all emails with same thread_id, sort by date
```

### Spam Filter ML Features

```
Text features: word/bigram frequencies, URL patterns, HTML ratio
Sender features: domain age, SPF/DKIM/DMARC, reputation score
Behavioral: sender-recipient history, user spam reports
Link features: URL shorteners, redirect chains, domain reputation
Image features: text-in-image ratio (spammers embed text as images)
Model: gradient boosted trees (LightGBM) + deep NLP (BERT fine-tuned)
Threshold: score > 0.7 → spam, 0.3-0.7 → maybe spam (user confirmation)
```

---

## 7. Fault Tolerance

### Email Delivery Guarantees
```
SMTP RFC 5321: once a server ACKs receipt, it MUST deliver or bounce
  - Accepted → persisted to durable queue (Kafka, RF=3) before ACK
  - If downstream fails → retry with exponential backoff (up to 72h)
  - After 72h → generate bounce (DSN) to sender
  - Never silently drop an email
```

### Data Loss Prevention
```
- Blob Store: 3× replication across AZs (11-nines durability)
- Metadata: Bigtable/Cassandra RF=3, quorum writes
- Search index: rebuilt from metadata + blob store (source of truth)
- Backup: incremental snapshots to cold storage (weekly)
```

---

## 8. Additional Considerations

### Email Send Flow (Deep Dive)

```
1. User clicks "Send" → POST /api/messages/send
2. Validate: recipients exist, attachment size < 25MB, rate limit check
3. Store email body + attachments to Blob Store (S3)
4. Store email metadata to Bigtable/Cassandra (sender's "Sent" label)
5. Enqueue to send queue (Kafka topic: outgoing-emails)
6. SMTP Sender Worker picks up from queue:
   a. DNS MX lookup: recipient domain → find receiving mail server
   b. Open TLS connection to receiving server (STARTTLS)
   c. Authenticate: sign with DKIM key for sender's domain
   d. Transmit email via SMTP protocol
   e. Receiving server ACKs → mark as delivered
   f. If rejected → generate bounce (DSN) → deliver to sender's inbox
7. If temporary failure (server busy, rate limited):
   → Retry with exponential backoff: 1min, 5min, 30min, 2h, 12h, 24h, 48h, 72h
   → After 72h of retries → permanent failure → bounce to sender

Optimization for internal emails (sender@gmail → recipient@gmail):
   Skip SMTP entirely → directly store in recipient's mailbox → 100ms delivery
```

### Incoming Email Flow (SMTP Receive)

```
1. External sender's MTA connects to our SMTP gateway (MX record)
2. SMTP handshake: EHLO, MAIL FROM, RCPT TO
3. Before accepting DATA:
   a. SPF check: is sender's IP authorized for their domain?
   b. Rate limiting: too many emails from this IP? → 421 temporary reject
   c. Recipient exists? → 550 user unknown if not
4. Accept DATA → email content streamed
5. DKIM verification: check cryptographic signature
6. DMARC evaluation: combine SPF + DKIM → pass/fail policy
7. Spam classification: ML model scores email (0-1)
   Score > 0.7 → spam folder; 0.3-0.7 → show warning; < 0.3 → inbox
8. Virus scan: check attachments for malware (ClamAV)
9. Store body to Blob Store, metadata to Bigtable
10. Index in Elasticsearch for search
11. Push notification to recipient (if enabled)
12. Return 250 OK to sender's MTA (MUST persist before ACK per RFC)
```

### Push vs Pull (IMAP IDLE vs Polling)

```
IMAP IDLE (push-like):
  Client sends IDLE command → server holds connection open
  New email arrives → server sends EXISTS notification → client fetches
  Efficient: no polling, sub-second notification
  Used by: Outlook, Thunderbird, Apple Mail

Google Sync (proprietary):
  Mobile clients use Google's push notification service
  FCM (Android) / APNs (iOS) for battery-efficient push
  No persistent IMAP connection → saves mobile battery

Web client:
  WebSocket for real-time inbox updates
  Server pushes new email metadata → client renders immediately
```

### SPF, DKIM, DMARC (Email Authentication)
```
SPF: DNS record listing authorized IPs for a domain
DKIM: Cryptographic signature in email header (proves domain sent it)
DMARC: Policy telling receivers what to do if SPF/DKIM fails (reject/quarantine)
All three: prevent email spoofing and phishing
```

### Why Email Architecture Is Unique
```
- Federated protocol: anyone can run an SMTP server (unlike WhatsApp)
- Store-and-forward: emails may bounce through multiple MTAs
- Spam: 50%+ of all email is spam → filtering is critical infrastructure
- Standards: 40+ years of RFCs → backward compatibility constraints
- No real-time: unlike chat, email is async (seconds to hours delivery)
- Delivery guarantee: RFC mandates no silent drops → bounce on failure
```

