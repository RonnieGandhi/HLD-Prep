# 73. Design a Digital Wallet System

---

## 1. Functional Requirements (FR)

- **Wallet balance**: Maintain a monetary balance per user
- **Top-up**: Add funds via bank transfer, credit card, or external payment
- **Send money (P2P)**: Transfer funds between wallet users instantly
- **Pay merchants**: Pay at checkout using wallet balance
- **Transaction history**: View all credits, debits, and transfers with details
- **Withdraw**: Cash out wallet balance to bank account
- **Multi-currency**: Hold balances in multiple currencies with conversion
- **Rewards/cashback**: Credit rewards directly to wallet
- **Spending limits**: Configurable daily/monthly transaction limits

---

## 2. Non-Functional Requirements (NFRs)

- **Strong Consistency (ACID)**: Balance must NEVER go negative; no double-spend
- **Exactly-once**: Each transaction processed exactly once (idempotent)
- **Low Latency**: P2P transfer completes in < 500 ms
- **High Availability**: 99.999% — wallet is the user's money
- **Auditability**: Every balance change has an immutable audit trail (double-entry ledger)
- **Security**: PCI DSS, encryption at rest/transit, fraud detection
- **Scale**: 100M+ wallets, 50M+ transactions/day

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total wallets | 100M |
| Active wallets (monthly) | 30M |
| Transactions / day | 50M |
| Transactions / sec | ~600 (peak 5K) |
| Ledger entries / day | 100M (double-entry: 2 per transaction) |
| Avg transaction size (metadata) | ~500 bytes |
| Ledger storage / day | 100M × 500B = 50 GB |
| Ledger storage / year | ~18 TB |
| Kafka events / day | 50M txn events + 50M notification events = 100M |
| Redis keys (active) | 30M balance cache + 5M idempotency + 30M daily_spent = ~65M |
| PostgreSQL rows (wallets table) | 100M (fits single shard; shard at 500M+) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         SERVING LAYER                                  │
│                                                                        │
│  Client --> API Gateway (auth + rate limit) --> Wallet Service          │
│                                                      |                 │
│             +----------------------------------------+----------+      │
│             |                |                |               |  |      │
│     +-------v------+ +------v-------+ +------v------+ +------v-+----+ │
│     | Transaction  | | Balance      | | History     | | Top-up /    | │
│     | Orchestrator | | Service      | | Service     | | Withdrawal  | │
│     | (P2P, pay,   | | (real-time   | | (paginated  | | Service     | │
│     |  debit/credit| |  balance     | |  txn list   | | (bank       | │
│     |  via double- | |  from cache  | |  from PG)   | |  integration| │
│     |  entry       | |  or PG)      | |             | |  async via  | │
│     |  ledger)     | |              | |             | |  Kafka)     | │
│     +------+-------+ +------+-------+ +------+------+ +------+------+ │
│            |                |                |               |         │
│     +------v-------+ +-----v--------+ +-----v------+       |         │
│     | PostgreSQL   | | Redis        | | PostgreSQL |       |         │
│     | (wallets,    | | (balance     | | (ledger    |       |         │
│     |  balances,   | |  cache,      | |  entries,  |       |         │
│     |  ledger)     | |  idempotency | |  transactions     |         │
│     | Sharded by   | |  keys,       | |  partitioned     |         │
│     | user_id      | |  daily_spent | |  by month)  |       |         │
│     +--------------+ +--------------+ +------+------+       |         │
│                                              |               |         │
│                                       +------v------+       |         │
│                                       |    Kafka    |<------+         │
│                                       | (txn-events)|                  │
│                                       +------+------+                  │
│                                              |                         │
│                        +---------------------+---------------------+   │
│                        |                     |                     |   │
│                 +------v------+       +------v------+       +------v-+ │
│                 | Notification|       | Fraud       |       |Analytics│
│                 | Service     |       | Detection   |       |Pipeline │
│                 | (email, SMS,|       | Service     |       |(Click-  │
│                 |  push for   |       | (velocity,  |       | House:  │
│                 |  every txn) |       |  geo-anomaly|       | txn     │
│                 +-------------+       |  ML scoring)|       | volume, │
│                                       +------+------+       | trends) │
│                                              |               +--------+│
│                                       +------v------+                  │
│                                       | KYC / AML   |                  │
│                                       | Service     |                  │
│                                       | (identity   |                  │
│                                       |  verify,    |                  │
│                                       |  sanctions  |                  │
│                                       |  screening) |                  │
│                                       +-------------+                  │
│                                                                        │
│  SETTLEMENT (daily batch):                                             │
│  +----------------+     +------------------+     +-----------------+   │
│  | Settlement     | --> | Bank Gateway     | --> | Reconciliation  |   │
│  | Service        |     | (ACH, SWIFT,     |     | Service         |   │
│  | (net positions |     |  SEPA transfers) |     | (match bank     |   │
│  |  per user)     |     +------------------+     |  confirmations  |   │
│  +----------------+                              |  with ledger)   |   │
│                                                  +-----------------+   │
└────────────────────────────────────────────────────────────────────────┘
```

### Complete P2P Transfer Flow — Step-by-Step Sequence

```
Alice (Client)      API Gateway      Txn Orchestrator    PostgreSQL         Redis            Kafka            Notification
  │                    │                   │                 │                 │                │                │
  │── POST /transfer ─▶│                   │                 │                 │                │                │
  │   Idempotency-Key  │── auth + rate ───▶│                 │                 │                │                │
  │   "txn-uuid-123"   │   limit check     │                 │                 │                │                │
  │                    │                   │── check idem ──▶│                 │                │                │
  │                    │                   │   key in Redis  │           ┌─────│                │                │
  │                    │                   │◀── not found ───│           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │── check daily ──│───────────│────▶│                │                │
  │                    │                   │   spending limit│           │     │                │                │
  │                    │                   │◀── under limit ─│───────────│─────│                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │── fraud check ──│── inline: velocity, geo, amount  │                │
  │                    │                   │◀── PASS ────────│           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │── BEGIN TXN ───▶│           │     │                │                │
  │                    │                   │   SELECT balance│           │     │                │                │
  │                    │                   │   FROM wallets  │           │     │                │                │
  │                    │                   │   WHERE alice   │           │     │                │                │
  │                    │                   │   FOR UPDATE    │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │   balance=$100  │           │     │                │                │
  │                    │                   │   $100 >= $50 ✓ │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │   UPDATE alice  │           │     │                │                │
  │                    │                   │   balance = $50 │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │   UPDATE bob    │           │     │                │                │
  │                    │                   │   balance += $50│           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │   INSERT ledger │           │     │                │                │
  │                    │                   │   (alice DEBIT) │           │     │                │                │
  │                    │                   │   INSERT ledger │           │     │                │                │
  │                    │                   │   (bob CREDIT)  │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │   INSERT txn    │           │     │                │                │
  │                    │                   │   record        │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │   INSERT outbox │           │     │                │                │
  │                    │                   │   event         │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │── COMMIT ──────▶│           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │── cache update ─│───────────│────▶│                │                │
  │                    │                   │   SET balance:  │           │     │ (invalidate    │                │
  │                    │                   │   alice = $50   │           │     │  alice + bob)  │                │
  │                    │                   │   SET balance:  │           │     │                │                │
  │                    │                   │   bob = $150    │           │     │                │                │
  │                    │                   │   INCRBY daily_ │           │     │                │                │
  │                    │                   │   spent:alice 50│           │     │                │                │
  │                    │                   │   SET idem key  │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │◀── 200 OK ─────────│◀── response ──────│                 │           │     │                │                │
  │   {txn-id,         │                   │                 │           │     │                │                │
  │    balance: $50}   │                   │                 │           │     │                │                │
  │                    │                   │                 │           │     │                │                │
  │                    │                   │ (async) outbox ─│───────────│─────│───────────────▶│                │
  │                    │                   │ worker polls +  │           │     │  txn-events    │── push notif ─▶│
  │                    │                   │ publishes       │           │     │  topic         │  "Alice sent   │
  │                    │                   │                 │           │     │                │   you $50"     │
```

**Key invariants maintained throughout:**
1. Alice's row locked with `FOR UPDATE` — no concurrent spend can proceed
2. Both balance updates + both ledger entries + transaction record + outbox event = single atomic COMMIT
3. If anything fails before COMMIT → full rollback, Alice keeps $100
4. Idempotency key prevents duplicate processing on retry
5. Kafka event is guaranteed via transactional outbox (not direct publish)

### Component Deep Dive

#### Double-Entry Ledger — The Foundation

```
Every transaction creates TWO ledger entries:
  Debit from one account, Credit to another. Sum of all entries = 0.

P2P Transfer: Alice sends $50 to Bob
  Entry 1: Alice's wallet DEBIT  -$50
  Entry 2: Bob's wallet   CREDIT +$50
  Sum: 0 ✓

Top-up: Alice adds $100 from bank
  Entry 1: Alice's wallet  CREDIT +$100
  Entry 2: External_bank   DEBIT  -$100
  Sum: 0 ✓

Merchant Payment: Alice pays $25 to Merchant M
  Entry 1: Alice's wallet      DEBIT  -$25
  Entry 2: Merchant M's wallet CREDIT +$24.25
  Entry 3: Platform fee account CREDIT +$0.75 (3% commission)
  Sum: 0 ✓

Withdrawal: Alice withdraws $200 to bank
  Entry 1: Alice's wallet       DEBIT  -$200
  Entry 2: Bank_settlement acct CREDIT +$200
  Sum: 0 ✓

Why double-entry?
  1. Self-validating: SUM(all entries) must always = 0
  2. Complete audit trail: every dollar has a source and destination
  3. Regulatory requirement for financial systems
  4. Reconciliation: if SUM != 0, something is wrong -> alert immediately
  5. Revenue tracking: platform fee account balance = total revenue
```

#### Atomic Balance Update — Preventing Double-Spend

```
Critical: Two concurrent requests to spend Alice's last $50

Request 1: Pay merchant $50
Request 2: Send $50 to Bob (simultaneous)

Without protection: both read balance=50, both deduct, balance=-50 (INVALID)

Solution: PostgreSQL serializable transaction + row lock:

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SELECT balance FROM wallets WHERE user_id = 'alice' FOR UPDATE;
  -- balance = 50
  IF balance >= 50 THEN
    UPDATE wallets SET balance = balance - 50 WHERE user_id = 'alice';
    INSERT INTO ledger_entries (wallet_id, type, amount, ...) VALUES (...);
  ELSE
    RAISE 'Insufficient funds';
  END IF;
COMMIT;

FOR UPDATE locks Alice's row. Second transaction waits.
When first commits (balance=0), second reads balance=0 -> "Insufficient funds".

Why SERIALIZABLE and not just FOR UPDATE?
  FOR UPDATE prevents concurrent reads of stale balance.
  SERIALIZABLE prevents phantom reads (edge case with complex queries).
  For single-row updates, FOR UPDATE is sufficient and faster.
  Use SERIALIZABLE only for multi-row financial transactions.

Alternative: Optimistic approach (less contention at low conflict rates)
  UPDATE wallets SET balance = balance - 50 
  WHERE user_id = 'alice' AND balance >= 50;
  -- rows_affected = 1 → success
  -- rows_affected = 0 → insufficient funds (no lock held)
  
  ✓ No explicit lock, higher throughput under low contention
  ✓ CHECK(balance >= 0) as safety net
  ✗ Doesn't work when you need to read balance for business logic before update
  ✓ Best for simple debit operations
```

#### Idempotent Transactions

```
User clicks "Send $50" but network timeout. Client retries. Without idempotency: $100 sent.

Every API call includes idempotency_key (generated client-side per action):

  POST /api/v1/wallet/transfer
  Idempotency-Key: "txn-uuid-abc-123"
  { "to_user_id": "bob", "amount": 50.00 }

Server:
  1. Check: SELECT * FROM transactions WHERE idempotency_key = 'txn-uuid-abc-123'
     If exists -> return cached result (already processed)
  2. If not exists -> process transfer -> store result with idempotency_key
  3. Redis: SET idempotency:{key} {result} EX 86400 (fast lookup for retries)

Result: identical request always returns same response, processes only once.
```

#### Top-Up Flow — External Money In

```
User adds $100 from their bank account to wallet.

Two phases (because external bank transfers are async):

Phase 1: Initiate (synchronous, < 500ms)
  1. User selects "Add $100 from Chase Bank"
  2. API creates pending transaction: status = 'pending'
  3. Call Payment Gateway (Stripe/Plaid) to initiate ACH pull
  4. Return to user: "Top-up initiated, funds available shortly"

  Option A — Instant credit (PayPal, Venmo approach) ⭐:
    Credit wallet immediately, before bank confirms.
    Risk: bank transfer may fail (insufficient funds, closed account).
    Mitigation: limit instant credit to $500 for new users, $5000 for verified.
    If bank rejects → debit wallet (create reversal entry) + notify user.
    If balance insufficient for reversal → wallet goes negative → frozen.

  Option B — Credit on confirmation (conservative):
    Wallet balance updated ONLY after bank confirms (1-3 business days for ACH).
    User sees "pending" balance.
    No risk, but poor UX (user has to wait).

  Most wallets use Option A with risk limits.

Phase 2: Bank Confirmation (async, hours to days)
  1. Bank gateway (Stripe) sends webhook: "ACH transfer successful"
  2. Update transaction status: pending → completed
  3. If rejected: create reversal ledger entry, debit wallet, notify user

Credit card top-up:
  Instant: card authorization → capture → credit wallet immediately.
  Fees: 2-3% card processing fee (passed to user or absorbed by platform).
  Fraud risk: stolen card used to top up → send money out → chargeback.
  Mitigation: hold period for new cards, velocity checks, 3DS verification.
```

#### Withdrawal Flow — Money Out

```
User withdraws $200 from wallet to bank account.

1. Validate: balance >= $200, wallet not frozen, KYC verified
2. BEGIN TXN:
   UPDATE wallets SET balance = balance - 200 WHERE user_id = ? AND balance >= 200;
   INSERT ledger_entry (DEBIT -$200 from user's wallet)
   INSERT ledger_entry (CREDIT +$200 to external_bank_settlements)
   INSERT transaction (type='withdrawal', status='pending')
   INSERT outbox event
   COMMIT;
3. Wallet balance immediately reduced (prevents double-withdrawal)
4. Async: Settlement service batches withdrawals → ACH/SWIFT to bank
5. Bank confirms → update transaction status to 'completed'
6. Bank rejects → reverse: credit wallet back, notify user

Timing:
  ACH (US): 1-3 business days
  SEPA (EU): 1 business day
  Instant (UK Faster Payments, India UPI): seconds to minutes
  Wire (SWIFT): 1-5 business days, higher fees

Anti-abuse:
  New accounts: 7-day hold before first withdrawal (prevent stolen-card-to-withdrawal fraud)
  Max 3 withdrawals per day
  Amount > $10,000: trigger CTR (Currency Transaction Report) for FinCEN
```

#### Fraud Detection Service — Real-Time + Batch

```
Two-tier fraud detection:

Tier 1: Real-time (inline, < 50ms, on every transaction):
  Rule-based checks before transaction is processed:
  
  1. Velocity: > 5 transactions in 1 minute → BLOCK
  2. Amount anomaly: transaction > 5× user's 30-day average → FLAG
  3. Geo-anomaly: transaction from new country within 1 hour of last → FLAG
  4. Device fingerprint: new device + high-value transaction → require 2FA
  5. Recipient risk: sending to newly created wallet → require SMS confirm
  6. Time anomaly: transaction at 3 AM local time (unusual for user) → FLAG
  
  Implementation: Redis-backed rule engine
    Sliding window counters per user (ZADD + ZRANGEBYSCORE)
    Device fingerprint cache (SET device:{user_id}:{fingerprint})
    
  Actions: ALLOW, FLAG (proceed but alert), STEP_UP (require 2FA), BLOCK

Tier 2: Batch (async, hourly/daily):
  ML model + graph analysis on full transaction history:
  
  1. Network analysis: detect mule accounts (receive from many, send to few)
  2. Circular transfer detection (described in section 9)
  3. Behavioral profiling: deviation from established patterns
  4. Cross-user correlation: multiple wallets from same device/IP
  
  Technology: Spark batch jobs reading from Kafka → ClickHouse for analysis
  Output: risk scores per user, updated daily
  High-risk users → manual review queue for compliance team
```

#### Multi-Currency Wallet

```
User can hold balances in multiple currencies: USD, EUR, GBP, INR, etc.

Data model:
  wallet_balances table has ONE row per (user_id, currency) pair.
  Alice has: {USD: $500, EUR: €200, GBP: £100}

Cross-currency transfer: Alice sends $100 to Bob (who wants EUR)
  1. Get FX rate: 1 USD = 0.92 EUR (from FX rate service, cached 30s)
  2. Debit Alice: $100 USD
  3. Credit Bob: €92 EUR
  4. Ledger entries include FX rate used and both currency amounts
  5. FX spread/fee: 0.5-2% markup on mid-market rate (platform revenue)

FX Rate Service:
  Sources: Open Exchange Rates API, XE, or bank feed
  Cache in Redis: fx_rate:{from}:{to} → rate, TTL 30 seconds
  Fallback: if rate API is down, use last known rate + widen spread to 3%
  
  Edge case: rate changes between quote and execution
    Show user the quoted rate + guarantee it for 30 seconds
    If execution happens within 30s → honor quoted rate
    If expired → re-quote with current rate
```

#### Settlement and Reconciliation Service

```
Daily batch process to settle wallet ↔ bank movements:

1. Net Position Calculation (daily, 2 AM):
   For each user:
     total_top_ups_today = SUM(top-up amounts confirmed by bank)
     total_withdrawals_today = SUM(withdrawal amounts to process)
     net = total_top_ups - total_withdrawals
   
   For the platform:
     total_merchant_payouts = SUM(merchant wallet balances to settle)
     total_fees_collected = SUM(transaction fees, FX spreads)

2. Bank Settlement:
   Batch file → ACH processor (Stripe, Plaid, or direct bank connection)
   ACH files submitted in NACHA format (US) or SEPA XML (EU)
   Contains: [{account_number, routing_number, amount, direction}]

3. Reconciliation (daily, 6 AM):
   Match bank confirmations against our ledger:
   
   For each settlement item:
     bank_confirmed_amount == our_ledger_amount?
     If YES → mark as reconciled
     If NO → create discrepancy record → alert finance team
   
   Global check: SUM(all ledger debits) == SUM(all ledger credits)?
     If not balanced → CRITICAL ALERT → halt operations until resolved
   
   Redis vs PostgreSQL balance reconciliation:
     For each active user: compare Redis cached balance with PostgreSQL balance
     Discrepancies → invalidate Redis cache, log mismatch, alert

4. Merchant Settlement:
   Merchants paid out weekly or daily (configurable)
   net_payout = SUM(payments received) - SUM(refunds) - platform_commission
   Platform commission: 1.5-3% per transaction
```

---

## 5. APIs

```http
POST /api/v1/wallet/top-up
Idempotency-Key: "topup-uuid"
{ "amount": 100.00, "currency": "USD", "source": "bank_transfer", "source_ref": "bank-txn-id" }
→ 200 { "transaction_id": "txn-uuid", "new_balance": 150.00, "status": "pending" }

POST /api/v1/wallet/transfer
Idempotency-Key: "transfer-uuid"
{ "to_user_id": "bob-uuid", "amount": 50.00, "currency": "USD", "note": "Lunch" }
→ 200 { "transaction_id": "txn-uuid", "new_balance": 100.00 }

POST /api/v1/wallet/pay
Idempotency-Key: "pay-uuid"
{ "merchant_id": "m-uuid", "amount": 25.00, "currency": "USD", "order_ref": "order-123" }
→ 200 { "transaction_id": "txn-uuid", "new_balance": 75.00 }

POST /api/v1/wallet/withdraw
Idempotency-Key: "withdraw-uuid"
{ "amount": 200.00, "currency": "USD", "bank_account_id": "ba-uuid" }
→ 200 { "transaction_id": "txn-uuid", "new_balance": 75.00, "status": "pending", "estimated_arrival": "2026-04-19" }

POST /api/v1/wallet/refund
Idempotency-Key: "refund-uuid"
{ "original_transaction_id": "txn-uuid", "amount": 25.00, "reason": "product_return" }
→ 200 { "refund_id": "ref-uuid", "new_balance": 100.00 }

GET /api/v1/wallet/balance
→ { "balances": [{"currency": "USD", "available": 75.00, "pending": 0.00}] }

GET /api/v1/wallet/transactions?limit=20&cursor=...
→ { "transactions": [{ "id": "txn-uuid", "type": "debit", "amount": 25.00, 
    "counterparty": "merchant-name", "status": "completed", "created_at": "..." }],
    "next_cursor": "..." }
```

---

## 6. Data Model

### PostgreSQL — Source of Truth

```sql
CREATE TABLE wallets (
    wallet_id       UUID PRIMARY KEY,
    user_id         UUID UNIQUE NOT NULL,
    status          ENUM('active','frozen','closed','pending_kyc') NOT NULL DEFAULT 'active',
    kyc_level       ENUM('none','basic','verified','enhanced') DEFAULT 'none',
    daily_limit     DECIMAL(10,2) DEFAULT 5000.00,
    monthly_limit   DECIMAL(12,2) DEFAULT 50000.00,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE wallet_balances (
    wallet_id       UUID NOT NULL REFERENCES wallets,
    currency        CHAR(3) NOT NULL,
    balance         DECIMAL(15,2) NOT NULL DEFAULT 0 CHECK (balance >= 0),
    pending_in      DECIMAL(15,2) DEFAULT 0,   -- top-ups awaiting bank confirmation
    pending_out     DECIMAL(15,2) DEFAULT 0,   -- withdrawals in transit
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (wallet_id, currency)
);

CREATE TABLE ledger_entries (
    entry_id        BIGSERIAL PRIMARY KEY,
    transaction_id  UUID NOT NULL,
    wallet_id       UUID NOT NULL,
    currency        CHAR(3) NOT NULL,
    entry_type      ENUM('debit','credit') NOT NULL,
    amount          DECIMAL(15,2) NOT NULL,
    balance_after   DECIMAL(15,2) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_wallet (wallet_id, created_at DESC),
    INDEX idx_transaction (transaction_id)
) PARTITION BY RANGE (created_at);
-- Partition monthly for fast queries and archival

CREATE TABLE transactions (
    transaction_id  UUID PRIMARY KEY,
    idempotency_key VARCHAR(64) UNIQUE,
    type            ENUM('topup','transfer','payment','withdrawal','reward','refund') NOT NULL,
    from_wallet_id  UUID,
    to_wallet_id    UUID,
    amount          DECIMAL(15,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    fx_rate         DECIMAL(12,6),             -- if cross-currency
    fx_target_currency CHAR(3),
    fx_target_amount DECIMAL(15,2),
    status          ENUM('pending','completed','failed','reversed') NOT NULL,
    failure_reason  TEXT,
    metadata        JSONB,                     -- order_ref, note, merchant details
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_from (from_wallet_id, created_at DESC),
    INDEX idx_to (to_wallet_id, created_at DESC),
    INDEX idx_status (status, created_at)
);

-- Transactional outbox for Kafka event publishing
CREATE TABLE outbox (
    outbox_id       BIGSERIAL PRIMARY KEY,
    aggregate_type  VARCHAR(50) NOT NULL,      -- 'transaction', 'wallet'
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,      -- 'transfer.completed', 'topup.pending'
    payload         JSONB NOT NULL,
    published       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_unpublished (published, created_at) WHERE published = FALSE
);
```

### Redis
```
balance:{user_id}:{currency}  → DECIMAL (cached balance), TTL 60s
idempotency:{key}             → JSON (cached response), TTL 86400s
daily_spent:{user_id}         → DECIMAL (INCRBY on each spend), TTL resets at midnight
fraud:velocity:{user_id}      → ZSET (timestamps of recent txns), TTL 300s
fraud:device:{user_id}        → SET of known device fingerprints
session:{token}               → JSON (user session data), TTL 3600s
```

### Kafka Topics
```
Topic: txn-events              (every completed/failed transaction — for notification, analytics, fraud)
Topic: topup-events            (top-up initiated/confirmed/failed — for settlement processing)
Topic: withdrawal-events       (withdrawal requests — for batch settlement)
Topic: fraud-alerts            (flagged transactions — for compliance review queue)
Topic: audit-trail             (every state change — immutable regulatory log)
  Retention: 7 years (regulatory). Tiered storage: hot Kafka → cold S3.
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Double-spend** | SELECT FOR UPDATE row lock; CHECK(balance >= 0) constraint as safety net |
| **Duplicate transaction** | Idempotency key with UNIQUE constraint + Redis fast-path check |
| **Ledger inconsistency** | Double-entry: SUM(all entries) must = 0; hourly automated reconciliation |
| **DB failover** | Synchronous replication to standby; zero data loss on failover |
| **Frozen wallet** | Admin can freeze wallet (fraud); all operations return 403; requires manual unfreeze |
| **Balance cache stale** | Redis cache invalidated on every write; TTL 60s as fallback; reconciliation job catches drift |
| **Kafka publish failure** | Transactional outbox pattern: event stored in DB atomically with txn; polled and published async |
| **Bank gateway timeout** | Mark top-up/withdrawal as 'pending'; reconciliation job resolves based on bank webhook or polling |
| **Cross-shard P2P transfer** | Both wallets on different shards: use saga with compensating transaction (debit → credit; if credit fails → reverse debit) |

### Specific: Partial Failure in P2P Transfer

```
Debit Alice succeeds, but credit Bob fails (DB crash mid-transaction):

SAME-SHARD (Alice and Bob on same PostgreSQL shard):
  PostgreSQL transaction wraps BOTH operations:
    BEGIN; UPDATE alice balance; INSERT ledger; UPDATE bob balance; INSERT ledger; COMMIT;
    If any step fails → entire transaction rolls back. Alice keeps her money.
  
  This is the happy path — single DB transaction guarantees atomicity.

CROSS-SHARD (Alice on shard 1, Bob on shard 2):
  Cannot use single DB transaction across shards. Use saga pattern:
  
  Step 1: Debit Alice (shard 1)
    BEGIN; UPDATE alice balance -= 50; INSERT ledger; INSERT txn status='debit_done'; COMMIT;
  
  Step 2: Credit Bob (shard 2)
    BEGIN; UPDATE bob balance += 50; INSERT ledger; UPDATE txn status='completed'; COMMIT;
  
  If Step 2 fails:
    Compensating transaction on shard 1:
    BEGIN; UPDATE alice balance += 50; INSERT reversal ledger; UPDATE txn status='reversed'; COMMIT;
    Notify Alice: "Transfer failed, funds returned."
  
  Saga orchestrator retries Step 2 up to 3 times before compensating.
  Idempotency key on each step ensures no double-credit on retry.

But what if commit succeeds but Kafka event publish fails?
  Use transactional outbox pattern:
    INSERT event INTO outbox_table (within same DB transaction)
    Background worker polls outbox → publishes to Kafka → marks as published
  Guarantees: DB commit and event publish are atomic (via outbox).
```

---

## 8. Additional Considerations

### Regulatory Compliance (KYC / AML / CTR)
- **KYC (Know Your Customer)**: Identity verification required before enabling wallet
  - Level 1 (basic): email + phone → $500/day limit
  - Level 2 (verified): government ID + selfie → $5,000/day limit
  - Level 3 (enhanced): address proof + income verification → $50,000/day limit
- **AML (Anti-Money Laundering)**: Monitor transactions for suspicious patterns
  - SAR (Suspicious Activity Report): filed with FinCEN for flagged accounts
  - Sanctions screening: every transaction checked against OFAC/EU sanctions lists
  - PEP screening: Politically Exposed Persons require enhanced due diligence
- **CTR (Currency Transaction Report)**: Required for cash transactions > $10,000/day
- **Data retention**: All transaction data retained 7 years minimum (BSA requirement)
- **GDPR/CCPA**: Right to data export, right to deletion (except data required for regulatory retention)
- **Audit logs**: Immutable (append-only ledger_entries table, no UPDATE/DELETE allowed)
- **Licensing**: Money transmitter license required per US state; e-money license for EU (EMD2)

### Merchant Integration
- **Merchant wallets**: Separate wallet type with different limits and fee structure
- **Payment SDK**: JavaScript/mobile SDK for merchants to accept wallet payments at checkout
- **QR code payments**: Generate/scan QR code containing payment details (amount, merchant_id)
- **Webhooks**: Notify merchant of payment status changes (payment.completed, payment.refunded)
- **Payout schedule**: Daily or weekly settlement to merchant's bank account
- **Platform commission**: 1.5-3% per transaction; configurable per merchant tier

### Rewards and Cashback
- **Cashback rules engine**: Configurable rules per merchant/category (e.g., 5% cashback on food delivery)
- **Reward crediting**: Async Kafka consumer listens to `txn-events` → evaluates rules → credits wallet
- **Reward wallet**: Separate balance bucket (reward_balance) — can only be spent, not withdrawn
- **Expiry**: Rewards expire after 90 days (background job scans and debits expired rewards)
- **Cap**: Max $50 cashback per month per user (prevent abuse)

### Notifications
- **Every transaction**: Push notification + in-app notification for both sender and receiver
- **Channels**: Push (FCM/APNs), SMS (for OTP and high-value alerts), email (receipts)
- **Templates**: "You sent $50 to Bob", "You received $50 from Alice", "Your withdrawal of $200 is processing"
- **Low-balance alert**: Notify when balance drops below user-configured threshold
- **Security alerts**: Login from new device, password change, large transaction

### Monitoring & Alerting
- **Ledger balance invariant**: SUM(all debits) == SUM(all credits) — checked hourly; CRITICAL alert if violated
- **Transaction success rate**: Alert if drops below 99.5%
- **Latency P99**: Alert if P2P transfer > 2 seconds
- **Fraud flag rate**: Alert if > 1% of transactions flagged
- **Redis ↔ PostgreSQL drift**: Alert if > 0.1% of cached balances don't match DB
- **Settlement reconciliation**: Alert on any unmatched bank confirmations

---

## 9. Deep Dive: Engineering Trade-offs

### Why PostgreSQL (Not DynamoDB/Cassandra) for Wallets

```
Financial data DEMANDS:
  1. ACID transactions: debit + credit MUST be atomic (one transaction, one commit)
  2. Strong consistency: balance is either $50 or $100, never "eventually $50"
  3. CHECK constraints: balance >= 0 enforced at DB level (last line of defense)
  4. Complex queries: "all failed transactions > $1000 in last 7 days" (compliance)
  5. Foreign keys: transaction → wallet → user → ledger entries
  6. Row-level locking: SELECT FOR UPDATE for concurrent balance operations

PostgreSQL ✓:
  - ACID with serializable isolation for critical paths
  - CHECK(balance >= 0): database rejects any update that would violate this
  - Multi-row atomic: debit Alice + credit Bob + 2 ledger entries + outbox = 1 commit
  - Rich query support for compliance and reconciliation
  - JSONB for flexible metadata without sacrificing relational integrity
  - Partitioning: ledger_entries partitioned by month for fast queries + easy archival

DynamoDB:
  - Has transactions (up to 100 items in TransactWriteItems)
  - But: 100-item limit could be restrictive for complex multi-step operations
  - No CHECK constraints → application must enforce balance >= 0
  - No JOINs → reconciliation queries become application-level nightmares
  - ✓ Auto-scaling with zero ops overhead
  - Could work for simpler wallet (single-currency, no cross-shard transfers)

Cassandra ✗ (critically inappropriate):
  - No multi-row transactions → CANNOT atomically debit + credit
  - Eventual consistency → two reads might show different balances
  - No CHECK constraints → no DB-level guard against negative balance
  → Financial regulators would reject this architecture

At 50M txn/day (~600 TPS, peak 5K):
  Single PostgreSQL primary handles this easily (PostgreSQL does 10K+ TPS for row updates).
  Shard by user_id hash for 10x scale (64 shards → 160 TPS/shard avg).
  Read replicas for balance queries and transaction history.
```

### Sharding Strategy: When and How to Shard Wallets

```
Single PostgreSQL primary handles 600 TPS (avg) easily.
But at 10x growth (6,000 avg TPS, 50K peak) → need to shard.

Shard key: user_id (hash-based)
  shard_id = hash(user_id) % num_shards
  
  Why user_id?
    ✓ All of a user's data (wallet, ledger, transactions) on same shard
    ✓ P2P transfer between users on SAME shard = single DB transaction (atomic)
    ✓ Balance check + debit = single-shard operation (fast)
    ✗ P2P transfer between users on DIFFERENT shards = saga pattern (more complex)

Cross-shard transfer frequency:
  With 64 shards, probability that Alice and Bob are on same shard = 1/64 ≈ 1.5%
  So ~98.5% of P2P transfers are cross-shard → saga pattern is the common case.
  
  This is acceptable because:
  1. Saga adds ~50ms latency (two sequential DB calls) — well within 500ms SLA
  2. Compensation (rollback) is rare — only on DB failure during step 2
  3. Idempotency keys make retries safe

Alternative: Shard by wallet_id pairs (co-locate frequent transfer partners)
  ✗ Transfer patterns change over time
  ✗ Hot-spot risk (popular merchants would overload one shard)
  ✗ Complex shard assignment logic
  → Not recommended. Hash(user_id) is simpler and more predictable.

Shard topology:
  64 shards × (1 primary + 2 replicas) = 192 PostgreSQL instances
  Each shard handles: ~100M / 64 ≈ 1.5M wallets
  Each shard TPS: ~600 / 64 ≈ 10 avg (peak 80) → very comfortable
```

### Spending Limits and Velocity Checks

```
Daily spending limit per user (configurable):
  Default: $5,000/day. Premium: $50,000/day.

Implementation:
  Redis: INCRBY daily_spent:{user_id} {amount}
  Redis: EXPIRE daily_spent:{user_id} {seconds_until_midnight}
  
  Before processing transaction:
    current_spent = GET daily_spent:{user_id} or 0
    if current_spent + txn_amount > user.daily_limit:
      return ERROR "Daily spending limit exceeded"
  
  After successful transaction:
    INCRBY daily_spent:{user_id} {amount}

  Why Redis and not PostgreSQL?
    Checking limit is on the hot path (every transaction).
    Redis: < 1 ms. PostgreSQL SUM query: 10-50 ms.
    Reconcile daily: compare Redis total with PostgreSQL SUM(debits).

  Edge case: Redis crashes, daily_spent key lost
    User could exceed limit for the remainder of the day.
    Mitigation: on Redis miss, fall back to PostgreSQL SUM query (slow but correct).
    Also: hard PostgreSQL CHECK on monthly_limit as ultimate safety net.

Velocity checks (fraud-adjacent):
  > 5 transactions in 1 minute: flag for review
  > $1000 single transaction (new account): require additional verification
  Transfer to new recipient > $500: SMS/email confirmation required
```

### Edge Case: Circular Transfers and Money Laundering

```
Alice sends $1000 to Bob. Bob sends $1000 to Charlie. Charlie sends $1000 to Alice.
Net effect: $0 moved. But each transfer generated $3000 in volume.
Purpose: laundering money or exploiting cashback/rewards.

Detection:
  1. Graph analysis on transfer network:
     Build directed graph of transfers over 24-hour window
     Detect cycles: A → B → C → A (cycle detection via DFS)
     Flag if cycle total > $500

  2. Recipient diversity check:
     User sends money to 10 new recipients in 24 hours → suspicious
     Normal users: 1-3 unique recipients per week

  3. Round-trip detection:
     If A sends $X to B, and B sends $X (±5%) back to A within 24 hours → flag
     Even without cycle: simple round-trip is suspicious

  4. Structuring detection:
     Multiple transactions just below $10,000 threshold (avoiding CTR reporting)
     e.g., 5 × $9,500 deposits in one week = $47,500 → flag as structuring
     This is a federal crime (31 USC § 5324) even if the underlying money is clean.

  Action: freeze wallet pending manual review if any detection fires.
  Technology: Apache Flink streaming job on txn-events Kafka topic for real-time.
  Batch: daily Spark job builds full transfer graph → Neo4j for graph queries.
```

### Idempotency: Why It's the Most Critical Pattern for Wallets

```
Problem:
  Client sends "Pay $50" → server processes → response lost in network.
  Client retries → without idempotency → user charged $100.
  
  This is not hypothetical — mobile networks have 1-5% request failure rate.
  At 50M transactions/day × 2% retry rate = 1M retries/day.
  Without idempotency: 1M potential double-charges per day.

Implementation — three layers of defense:

Layer 1: Redis fast-path (< 1ms)
  On request: GET idempotency:{key}
  If found → return cached response immediately (no DB hit)
  TTL: 24 hours (client should generate new key for genuinely new transactions)

Layer 2: PostgreSQL UNIQUE constraint
  transactions.idempotency_key has UNIQUE constraint.
  If Redis misses (key expired or Redis down):
    INSERT INTO transactions (..., idempotency_key) VALUES (...);
    If UNIQUE violation → SELECT existing transaction → return cached result.
  This is the safety net if Redis is unavailable.

Layer 3: Application-level check within transaction
  BEGIN;
    SELECT * FROM transactions WHERE idempotency_key = ? FOR UPDATE;
    If found → ROLLBACK, return cached result.
    If not → proceed with transaction.
  COMMIT;
  
  The FOR UPDATE prevents two concurrent retries from both proceeding
  (one acquires lock, the other waits and then sees the existing row).

Why client-generated keys (not server-generated)?
  Server-generated: server returns ID in first response — but if response is lost,
  client doesn't have the ID for retry → can't deduplicate.
  Client-generated: client creates UUID before first attempt → same key on retry.
```

### Transactional Outbox vs Direct Kafka Publish

```
Problem: Transaction committed to PostgreSQL, but Kafka publish fails.
  Result: money moved, but notification/fraud-check/analytics never fires.

Direct Kafka publish (naive approach):
  1. BEGIN; UPDATE balance; INSERT ledger; COMMIT;
  2. kafka.send("txn-events", event)  // ← this can fail!
  
  If step 2 fails: transaction is committed but downstream systems don't know.
  If step 2 succeeds but step 1 rolls back: event published for non-existent txn.
  ✗ No atomicity between DB commit and event publish.

Transactional Outbox ⭐ (recommended):
  1. BEGIN;
       UPDATE balance; INSERT ledger; INSERT INTO outbox (event_payload);
     COMMIT;  // all-or-nothing, including the outbox row
  2. Background poller: SELECT * FROM outbox WHERE published = FALSE ORDER BY created_at;
  3. For each: kafka.send(event) → on success: UPDATE outbox SET published = TRUE;
  
  ✓ Atomicity: outbox row committed with the transaction
  ✓ At-least-once delivery: poller retries until published = TRUE
  ✓ Ordering: outbox rows polled in created_at order
  ✗ Latency: slight delay (poller interval, typically 100ms-1s)
  ✗ Poller adds DB load (mitigated: index on published=FALSE)

Alternative: CDC (Change Data Capture) via Debezium
  Debezium reads PostgreSQL WAL → publishes changes to Kafka automatically.
  ✓ No outbox table needed
  ✓ Near-zero latency (streams from WAL in real-time)
  ✗ More infrastructure to manage (Debezium connector, Kafka Connect)
  ✗ Captures ALL changes (need filtering for relevant events)
  
Decision: Transactional outbox for simplicity. Migrate to CDC at scale.
```

