# 24. Design a Payment Gateway (Handling ACID Transactions)

---

## 1. Functional Requirements (FR)

- **Process payments**: Accept payments via credit/debit cards, bank transfers, digital wallets (PayPal, Apple Pay, Google Pay)
- **Authorize & Capture**: Two-phase payment — authorize (hold funds) → capture (charge) — or single-step direct charge
- **Refunds**: Full and partial refunds
- **Recurring payments**: Subscriptions, auto-debit
- **Multi-currency**: Accept and settle in multiple currencies
- **Tokenization**: Store card details securely as tokens (PCI DSS compliance)
- **Webhooks**: Notify merchants of payment status changes asynchronously
- **Retry failed payments**: Automatic retry for transient failures
- **Ledger**: Double-entry bookkeeping for every transaction
- **Merchant dashboard**: View transactions, settlements, chargebacks, analytics

---

## 2. Non-Functional Requirements (NFRs)

- **Strong Consistency (ACID)**: Every payment must be exactly-once — no double charges, no lost payments
- **High Availability**: 99.999% (five nines) — downtime = lost revenue for all merchants
- **Low Latency**: Payment authorization in < 2 seconds
- **Security**: PCI DSS Level 1 compliance, encryption at rest and in transit
- **Idempotency**: Same payment request submitted multiple times results in only one charge
- **Auditability**: Every state change logged immutably for regulatory compliance
- **Scalability**: Process 10,000+ transactions per second
- **Fault Tolerant**: Survive datacenter failures without data loss

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Transactions / day | 500M |
| Transactions / sec | ~6,000 (peak 20K) |
| Avg transaction size | 1 KB (metadata) |
| Ledger entries / day | 1B (2 entries per txn: debit + credit) |
| Storage / day | 1B × 500 bytes = 500 GB |
| Storage / year | ~180 TB |
| Active merchants | 5M |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│  Merchant    │
│  Application │
└──────┬───────┘
       │ HTTPS (TLS 1.3)
┌──────▼───────┐
│ API Gateway  │  ��� Rate limiting, API key auth, idempotency check
└──────┬───────┘
       │
┌──────▼───────────────────────────────────┐
│            Payment Service               │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐│
│  │ Payment  │  │  Fraud   │  │ Routing │││
│  │ Orchestr.│  │  Engine  │  │ Engine  │││
│  └────┬─────┘  └────┬─────┘  └────┬────┘│
│       │              │              │     │
└───────┼──────────────┼──────────────┼─────┘
        │              │              │
┌───────▼──────┐       │       ┌──────▼──────┐
│  Tokenization│       │       │  Payment    │
│  Service     │       │       │  Processor  │
│  (Vault)     │       │       │  Connector  │
└──────────────┘       │       └──────┬──────┘
                       │              │
                ┌──────▼──────┐       │
                │  Risk &     │  ┌────▼──────────────┐
                │  Fraud DB   │  │ External PSPs     │
                └─────────────┘  │ (Visa, MC, Stripe │
                                 │  Adyen, PayPal)   │
                                 └───────────────────┘
        │
┌───────▼──────────────────────────────┐
│         Ledger & Settlement          │
│  ┌──────────┐  ┌──────────────────┐  │
│  │  Ledger  │  │  Settlement      │  │
│  │  Service │  │  Service (T+1/T+2│  │
│  └────┬─────┘  └────┬─────────────┘  │
│       │              │               │
│  ┌────▼─────┐  ┌─────▼────┐         │
│  │PostgreSQL│  │ Batch    │         │
│  │(Ledger DB│  │ Payout   │         │
│  │ Immutable│  │ System   │         │
│  └──────────┘  └──────────┘         │
└──────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐
│    Kafka     │  │  Webhook     │
│ (Payment     │  │  Delivery    │
│  Events)     │  │  Service     │
└──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Payment Orchestrator — The Core State Machine

A payment goes through a well-defined state machine:

```
                    ┌───────────┐
                    │  CREATED  │  ← Merchant submits payment request
                    └─────┬─────┘
                          │ fraud check
                    ┌─────▼─────┐
               ┌────│ VALIDATED │  ← Fraud engine approves
               │    └─────┬─────┘
               │          │ send to PSP
     ┌─────────▼──┐  ┌────▼──────┐
     │  REJECTED  │  │AUTHORIZED │  ← Card network approves, funds held
     │ (fraud/    │  └─────┬─────┘
     │  declined) │        │ capture (immediate or later)
     └────────────┘  ┌─────▼─────┐
                     │ CAPTURED  │  ← Funds transferred
                     └─────┬─────┘
                           │
                     ┌─────▼─────┐
                     │ SETTLED   │  ← Funds in merchant's account
                     └─────┬─────┘
                           │ (optional)
                     ┌─────▼─────┐
                     │ REFUNDED  │  ← Full or partial refund
                     └───────────┘
```

- Each state transition is a **database transaction** with an audit log entry
- **Idempotency**: Every API call includes an `idempotency_key`. Before processing, check if this key was already processed → return cached result

#### Idempotency — The Most Critical Design Decision

```
Payment Request Flow:
1. Receive request with idempotency_key
2. BEGIN TRANSACTION
3. SELECT * FROM idempotency_keys WHERE key = ? FOR UPDATE
4. If exists → return cached_response (already processed)
5. If not exists → INSERT into idempotency_keys
6. Process payment
7. UPDATE idempotency_keys SET response = ?, status = 'completed'
8. COMMIT TRANSACTION
```

**Why this matters**: If a merchant's server crashes after sending the payment request but before receiving the response, it will retry. Without idempotency, the customer gets double-charged.

#### Tokenization Service (Card Vault)

- **Purpose**: Store sensitive card data (PAN, CVV) in an isolated, PCI-compliant vault
- **How**: 
  1. Merchant sends raw card data to Tokenization Service
  2. Service encrypts card data with AES-256, stores in HSM-backed vault
  3. Returns a non-reversible token: `tok_4242424242424242` 
  4. Subsequent payments use the token — no raw card data leaves the vault
- **PCI DSS**: Only the vault touches raw card data. All other services only see tokens
- **HSM (Hardware Security Module)**: Physical device that stores encryption keys — keys never leave the HSM

#### Fraud Detection Engine

- **Real-time checks** (< 100 ms):
  - Velocity checks: > 5 transactions in 1 minute from same card → flag
  - Geolocation mismatch: card issued in US, transaction from Nigeria → flag
  - Amount anomaly: Transaction 10× larger than user's average → flag
  - Known fraudulent BINs, IPs, devices
- **ML model**: Trained on historical fraud data
  - Features: amount, merchant category, time of day, device fingerprint, card age, transaction frequency
  - Output: fraud probability (0-1). If > threshold → reject or 3D Secure challenge
- **Rules engine**: Configurable rules per merchant (e.g., "block transactions > $10,000 without 3DS")

#### Payment Processor Connector (PSP Router)

- **Smart routing**: Choose the optimal PSP (Payment Service Provider) for each transaction
  - Route by: card network (Visa → PSP A, Mastercard → PSP B), geography, cost, success rate
  - **Failover**: If PSP A is down → automatically route to PSP B
- **Retry logic**: 
  - Soft decline (insufficient funds, temporary hold) → retry after 24 hours
  - Hard decline (stolen card, invalid number) → do not retry
- **Connectors**: Standardized interface; each PSP has an adapter implementing it

#### Ledger Service — Double-Entry Bookkeeping

Every financial transaction creates TWO ledger entries (debits and credits must balance):

```
Payment of $100:
  DEBIT   customer_account    $100  (money leaves customer)
  CREDIT  merchant_account    $100  (money enters merchant)
  CREDIT  platform_fee        $3    (our commission)
  DEBIT   merchant_account    $3    (adjusted merchant amount)

Refund of $100:
  DEBIT   merchant_account    $100  (money leaves merchant)
  CREDIT  customer_account    $100  (money returns to customer)
```

- **Immutable**: Ledger entries are append-only (never updated or deleted)
- **Reconciliation**: Daily batch job verifies: sum of all debits = sum of all credits (if not → alert)

#### Settlement Service

- **T+1 or T+2 settlement**: Aggregate captured payments per merchant → batch payout
- **Net settlement**: `payout = sum(captures) - sum(refunds) - sum(fees)`
- **Batch processing**: Daily Spark job computes settlement amounts → initiates bank transfers via ACH/SWIFT

---

## 5. APIs

### Create Payment
```http
POST /api/v1/payments
Idempotency-Key: idem-uuid-12345
Authorization: Bearer <merchant_api_key>

{
  "amount": 10000,           // in smallest currency unit (cents)
  "currency": "USD",
  "payment_method": "tok_4242424242424242",
  "capture": true,           // false for auth-only
  "description": "Order #12345",
  "metadata": {"order_id": "ORD-12345"},
  "return_url": "https://merchant.com/payment/complete"
}

Response: 200 OK
{
  "payment_id": "pay_uuid",
  "status": "captured",      // or "authorized"
  "amount": 10000,
  "currency": "USD",
  "payment_method": "tok_4242...",
  "created_at": "2026-03-13T10:00:00Z"
}
```

### Capture (for auth-only payments)
```http
POST /api/v1/payments/{payment_id}/capture
{ "amount": 10000 }    // can capture less than authorized (partial capture)
```

### Refund
```http
POST /api/v1/payments/{payment_id}/refund
Idempotency-Key: refund-idem-uuid
{ "amount": 5000, "reason": "customer_request" }
```

### Webhook (sent to merchant)
```http
POST https://merchant.com/webhooks/payment
{
  "event": "payment.captured",
  "payment_id": "pay_uuid",
  "amount": 10000,
  "currency": "USD",
  "timestamp": "2026-03-13T10:00:01Z"
}
```

---

## 6. Data Model

### PostgreSQL — Payments (ACID, the most critical table)

```sql
CREATE TABLE payments (
    payment_id      UUID PRIMARY KEY,
    merchant_id     UUID NOT NULL,
    amount          BIGINT NOT NULL,          -- in cents
    currency        VARCHAR(3) NOT NULL,
    status          VARCHAR(20) NOT NULL,     -- created, authorized, captured, refunded, failed
    payment_method  VARCHAR(64),              -- token reference
    capture_method  VARCHAR(20),              -- 'automatic' or 'manual'
    description     TEXT,
    metadata        JSONB,
    psp_reference   VARCHAR(128),             -- PSP's transaction ID
    psp_name        VARCHAR(64),              -- which PSP processed it
    failure_reason  TEXT,
    idempotency_key VARCHAR(128) UNIQUE,
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL,
    INDEX idx_merchant (merchant_id, created_at DESC),
    INDEX idx_status (status)
);
```

### PostgreSQL — Idempotency Keys

```sql
CREATE TABLE idempotency_keys (
    key             VARCHAR(128) PRIMARY KEY,
    merchant_id     UUID NOT NULL,
    request_hash    VARCHAR(64),      -- hash of request body
    response_code   INT,
    response_body   JSONB,
    created_at      TIMESTAMP,
    INDEX idx_created (created_at)    -- for cleanup of old keys
);
-- TTL: Background job deletes keys older than 24 hours
```

### PostgreSQL — Ledger (Append-Only, Immutable)

```sql
CREATE TABLE ledger_entries (
    entry_id        BIGSERIAL PRIMARY KEY,
    payment_id      UUID NOT NULL,
    account_id      UUID NOT NULL,
    entry_type      ENUM('debit', 'credit'),
    amount          BIGINT NOT NULL,
    currency        VARCHAR(3),
    balance_after   BIGINT,           -- running balance
    description     TEXT,
    created_at      TIMESTAMP NOT NULL,
    INDEX idx_account (account_id, created_at),
    INDEX idx_payment (payment_id)
);

-- Constraint: sum of debits = sum of credits per payment_id
```

### PostgreSQL — Audit Log (Every State Change)

```sql
CREATE TABLE payment_audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    payment_id      UUID NOT NULL,
    old_status      VARCHAR(20),
    new_status      VARCHAR(20),
    actor           VARCHAR(64),      -- 'system', 'merchant', 'psp'
    details         JSONB,
    created_at      TIMESTAMP NOT NULL
);
```

### Kafka Topics

```
Topic: payment-events       (status changes for downstream consumers)
Topic: webhook-delivery      (outbound webhook messages)
Topic: settlement-events     (captured payments for settlement processing)
```

---

## 7. Fault Tolerance

### General ACID Guarantees
| Property | How Ensured |
|---|---|
| **Atomicity** | PostgreSQL transactions — all or nothing |
| **Consistency** | DB constraints (CHECK, UNIQUE, FK) + application-level invariants |
| **Isolation** | Serializable or Read Committed isolation level |
| **Durability** | WAL (Write-Ahead Log) + synchronous replication to standby |

### Problem-Specific Fault Tolerance

1. **Network Failure Between Us and PSP**
   - We sent auth request to Visa but didn't get a response
   - **Solution**: Mark payment as `pending`. Background reconciliation job queries PSP for the status every minute. PSP APIs are idempotent → safe to re-query
   - **Never assume failure = declined**. The charge may have gone through

2. **Our Service Crashes After PSP Approved**
   - PSP authorized the payment, but we crash before recording it
   - **Solution**: PSP sends a webhook with the authorization result. Webhook handler checks if we have the payment recorded; if not, creates it. Reconciliation job catches anything missed

3. **Double Charge Prevention**
   - Merchant retries the same payment
   - **Solution**: Idempotency key. Second request returns the cached response from the first successful attempt

4. **Partial Failure in Multi-Step Payment**
   - Auth succeeded, but capture failed
   - **Solution**: Auth has an expiry (7-30 days). If not captured, funds are automatically released by the card network. Background job detects stuck payments and alerts

5. **Database Failover**
   - Primary DB goes down
   - **Solution**: Synchronous replication to standby. Promote standby to primary. Zero data loss guaranteed with synchronous replication (at cost of higher write latency)

6. **Webhook Delivery Failure**
   - Merchant's server is down when we try to deliver a webhook
   - **Solution**: Retry with exponential backoff (1s, 2s, 4s, 8s, ... up to 24h). After 24 hours of failures → mark as failed, alert merchant. Merchants can also poll via API for status

---

## 8. Additional Considerations

### PCI DSS Compliance
- Cardholder data (PAN, CVV) only in the tokenization vault
- All data encrypted at rest (AES-256) and in transit (TLS 1.3)
- Network segmentation: vault is in an isolated network
- Quarterly vulnerability scans, annual penetration testing
- Access logging for all vault operations

### Reconciliation
- **Internal reconciliation**: Verify our ledger matches our payment records (every hour)
- **External reconciliation**: Match our records with PSP settlement files (daily)
- **Bank reconciliation**: Match our bank statement with expected settlements (daily)
- Discrepancies → alert finance team for manual resolution

### Multi-Currency
- Accept payment in customer's currency (presentment currency)
- Settle to merchant in their currency (settlement currency)
- FX conversion at time of capture using real-time exchange rates
- FX markup: 1-3% fee on cross-currency transactions

### 3D Secure (3DS) Authentication
- Additional cardholder verification (OTP, biometric) for fraud prevention
- **3DS 2.0**: Frictionless flow for low-risk transactions (no redirect)
- Required by regulation in EU (PSD2 Strong Customer Authentication)

### Monitoring & Alerting
- Payment success rate per PSP (alert if drops below 95%)
- Average authorization latency (alert if > 3s)
- Fraud rate (alert if > 0.1% of transactions)
- Reconciliation discrepancy rate
- Webhook delivery success rate

---

## 9. Deep Dive: Engineering Trade-offs

### Why PostgreSQL (Not Cassandra/DynamoDB) for Payments?

```
Payment data DEMANDS:
  1. ACID transactions (atomic: debit + credit MUST happen together)
  2. Strong consistency (a payment is either completed or not — NEVER "eventually")
  3. Complex queries (finance team: "all failed payments > $1000 in last 7 days")
  4. Foreign keys (payment → booking → user → refund → ledger entries)
  5. CHECK constraints (amount > 0, status IN ('authorized', 'captured', ...))
  6. Audit requirements (triggers, row-level security, immutable audit tables)

PostgreSQL ✓:
  - ACID with serializable isolation for critical paths
  - Rich query support for finance/compliance
  - Row-level locking (SELECT ... FOR UPDATE) for concurrent payment updates
  - JSONB for flexible metadata without sacrificing relational integrity
  - Extensions: pg_audit for audit logging, pgcrypto for encryption

Cassandra ✗ (critically inappropriate):
  - No transactions across tables → can't atomically update payment + ledger
  - No constraints → application must enforce every invariant
  - No JOINs → reconciliation queries become application-level nightmares
  - Eventual consistency → two concurrent reads might see different payment states
  → Financial regulators would reject this architecture
  
DynamoDB:
  - Has transactions (up to 25 items)
  - But: limited query flexibility, no JOINs, vendor lock-in
  - Could work for small-scale payment systems
  - Not recommended for full-featured payment gateway

For analytics/reporting: Replicate to ClickHouse via Kafka (OLAP queries on 
payment data without loading the OLTP database).
```

### Authorize-Then-Capture vs Direct Charge: Why Two-Phase?

```
Direct Charge (single step):
  Customer clicks "Pay" → money moves immediately
  ✓ Simple
  ✗ If the order/booking fails AFTER payment → must refund (costs money, takes days)
  ✗ Card network charges refund fees to merchant
  ✗ High refund rate → card network flags merchant (can lose processing ability)

Authorize-Then-Capture (two-phase) ⭐:
  Step 1: Authorize → card network HOLDS funds (no money moves yet)
  Step 2: Capture → money actually moves (after business confirms the order)
  
  If order fails between auth and capture → VOID the authorization
  → No money moved, no refund needed, no fees, no penalties
  
  ✓ Zero-cost cancellation before capture
  ✓ Flexibility: authorize for $100, capture for $85 (partial capture for partial fulfillment)
  ✓ Fraud window: additional checks between auth and capture
  ✗ Authorization has an expiry (7-30 days depending on card network)
  ✗ More complex state machine
  
  Use cases:
    Hotels: Authorize at check-in → capture at checkout (actual charges may differ)
    E-commerce: Authorize at order → capture at shipment
    Ride-sharing: Authorize estimated fare → capture actual fare
```

### Idempotency: The Most Critical Design Pattern in Payments

```
Problem: 
  Client sends payment request → server processes it → response lost in network
  Client retries → without idempotency → customer charged TWICE
  
  This is NOT hypothetical — network failures happen constantly at scale.

Implementation layers:

Layer 1: Idempotency Key in API
  POST /payments { idempotency_key: "idem-uuid-123", ... }
  
  Server flow:
    1. BEGIN TRANSACTION
    2. SELECT * FROM idempotency_keys WHERE key = 'idem-uuid-123' FOR UPDATE
    3. If found AND status = 'completed' → return cached_response (no reprocessing)
    4. If found AND status = 'processing' → return 409 (in progress)
    5. If not found → INSERT with status = 'processing'
    6. Process payment
    7. UPDATE idempotency_keys SET status = 'completed', response = {...}
    8. COMMIT
  
  The FOR UPDATE lock prevents two concurrent retries from both processing.

Layer 2: PSP-Side Idempotency
  Most PSPs (Stripe, Adyen) support idempotency keys natively.
  Send our payment_id as their idempotency key → they deduplicate on their side.

Layer 3: Database Constraints
  UNIQUE constraint on (merchant_id, idempotency_key) → DB rejects duplicate inserts
  This is the last line of defense if application logic has a bug.
```

### Synchronous vs Asynchronous Payment Processing

```
Synchronous (blocking):
  Client → Server → PSP → Server → Client
  Total time: 1-5 seconds (depending on PSP + card network response time)
  
  ✓ Simple: client gets immediate yes/no
  ✗ Holds HTTP connection for 5 seconds
  ✗ Timeouts: if PSP takes > 30s, client's request times out
  ✗ Under load: many held connections exhaust server resources

Asynchronous (non-blocking):
  Client → Server: "Payment accepted, processing..."
  Server → PSP (async)
  PSP → Webhook → Server: "Payment authorized"
  Server → Client (push notification or polling)
  
  ✓ Server immediately frees the connection
  ✓ Handles slow PSPs gracefully
  ✗ More complex UX (client must poll or receive push update)
  ✗ More complex error handling

Hybrid ⭐ (Stripe's approach):
  Try synchronous with a 10-second timeout
  If PSP responds within 10s → return result immediately
  If timeout → return "pending" → continue async → notify via webhook
  
  Client experience: 95% of payments get instant response.
  The 5% that timeout get a "processing" status with webhook follow-up.
```

### Double-Entry Bookkeeping: Why It's Non-Negotiable

```
Without double-entry:
  Payment table shows: "$100 from customer to merchant"
  How do you answer: "What's the platform's total revenue today?"
  You'd need to scan all payments, calculate commissions... error-prone.

With double-entry ledger:
  Every payment creates BALANCED entries:
  
  DEBIT   Customer Wallet     $100    (money leaves customer)
  CREDIT  Platform Revenue    $3      (our commission)
  CREDIT  Merchant Account    $97     (merchant's share)
  
  Invariant: Sum of all DEBITs = Sum of all CREDITs (ALWAYS)
  
  Now: "Platform revenue today" = SELECT SUM(amount) FROM ledger 
       WHERE account = 'platform_revenue' AND date = today
  
  If the invariant is EVER violated → CRITICAL ALERT → something is wrong
  This is how all banks, exchanges, and financial systems work.
  
  Why immutable (append-only):
    A correction is a NEW entry (reversal), not an UPDATE to the old entry
    Example: Refund creates a reversal entry, not an edit of the original
    This provides a complete, auditable history of every penny movement
```

### Smart Routing: Choosing the Right PSP Per Transaction

```
Route based on:
  1. Card type: Visa → PSP A (better Visa rates), Mastercard → PSP B
  2. Geography: US transactions → Stripe, EU → Adyen, India → RazorPay
  3. Cost: PSP A charges 2.5%, PSP B charges 2.1% → prefer PSP B
  4. Success rate: PSP A has 98% auth rate for this card type, PSP B has 94% → prefer A
  5. Availability: PSP A is degraded → route to PSP B
  6. Fraud risk: High-risk transaction → PSP with better fraud detection

Routing algorithm:
  score(psp) = w1 × success_rate(psp, card_type) 
             + w2 × (1 / cost(psp)) 
             + w3 × availability(psp)
             + w4 × latency(psp)
  
  Route to: argmax(score(psp) for all psp in available_psps)
  
  Fallback cascade:
    PSP A fails → retry on PSP B → retry on PSP C → give up → alert

This optimization can improve overall authorization rates by 2-5%, 
which at scale means millions in additional revenue.
```
