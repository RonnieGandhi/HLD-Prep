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

Why double-entry?
  1. Self-validating: SUM(all entries) must always = 0
  2. Complete audit trail: every dollar has a source and destination
  3. Regulatory requirement for financial systems
  4. Reconciliation: if SUM != 0, something is wrong -> alert immediately
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

---

## 5. APIs

```http
POST /api/v1/wallet/top-up
Idempotency-Key: "topup-uuid"
{ "amount": 100.00, "source": "bank_transfer", "source_ref": "bank-txn-id" }
-> 200 { "transaction_id": "txn-uuid", "new_balance": 150.00 }

POST /api/v1/wallet/transfer
Idempotency-Key: "transfer-uuid"
{ "to_user_id": "bob-uuid", "amount": 50.00, "note": "Lunch" }
-> 200 { "transaction_id": "txn-uuid", "new_balance": 100.00 }

POST /api/v1/wallet/pay
Idempotency-Key: "pay-uuid"
{ "merchant_id": "m-uuid", "amount": 25.00, "order_ref": "order-123" }
-> 200 { "transaction_id": "txn-uuid", "new_balance": 75.00 }

GET /api/v1/wallet/balance
-> { "balance": 75.00, "currency": "USD", "pending": 0.00 }

GET /api/v1/wallet/transactions?limit=20&cursor=...
-> { "transactions": [{ "id": "txn-uuid", "type": "debit", "amount": 25.00, ... }] }
```

---

## 6. Data Model

### PostgreSQL — Source of Truth

```sql
CREATE TABLE wallets (
    wallet_id UUID PRIMARY KEY, user_id UUID UNIQUE NOT NULL,
    balance DECIMAL(15,2) NOT NULL DEFAULT 0 CHECK (balance >= 0),
    currency CHAR(3) DEFAULT 'USD', status ENUM('active','frozen','closed'),
    daily_limit DECIMAL(10,2) DEFAULT 5000.00,
    created_at TIMESTAMPTZ DEFAULT NOW(), updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ledger_entries (
    entry_id BIGSERIAL PRIMARY KEY,
    transaction_id UUID NOT NULL,
    wallet_id UUID NOT NULL,
    entry_type ENUM('debit','credit') NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_wallet (wallet_id, created_at DESC),
    INDEX idx_transaction (transaction_id)
);

CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY,
    idempotency_key VARCHAR(64) UNIQUE,
    type ENUM('topup','transfer','payment','withdrawal','reward'),
    from_wallet_id UUID, to_wallet_id UUID,
    amount DECIMAL(15,2) NOT NULL,
    status ENUM('pending','completed','failed','reversed'),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis
```
balance:{user_id}       -> DECIMAL (cached balance for fast reads), TTL 60
idempotency:{key}       -> JSON (cached response), TTL 86400
daily_spent:{user_id}   -> DECIMAL (INCRBY on each spend, TTL resets at midnight)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Double-spend** | SELECT FOR UPDATE row lock; CHECK(balance >= 0) |
| **Duplicate transaction** | Idempotency key with UNIQUE constraint |
| **Ledger inconsistency** | Double-entry: SUM(all entries) must = 0; nightly reconciliation |
| **DB failover** | Synchronous replication; zero data loss on failover |
| **Frozen wallet** | Admin can freeze wallet (fraud); all operations return 403 |
| **Balance cache stale** | Redis cache TTL 60s; invalidate on every write |

### Specific: Partial Failure in P2P Transfer

```
Debit Alice succeeds, but credit Bob fails (DB crash mid-transaction):

PostgreSQL transaction wraps BOTH operations:
  BEGIN; UPDATE alice balance; INSERT ledger; UPDATE bob balance; INSERT ledger; COMMIT;
  If any step fails -> entire transaction rolls back. Alice keeps her money.

But what if commit succeeds but Kafka event publish fails?
  Use transactional outbox pattern:
    INSERT event INTO outbox_table (within same DB transaction)
    Background worker polls outbox -> publishes to Kafka -> marks as published
  Guarantees: DB commit and event publish are atomic (via outbox).
```

---

## 8. Deep Dive

### Why PostgreSQL (Not DynamoDB/Cassandra) for Wallets

ACID transactions are non-negotiable for financial data. PostgreSQL provides serializable isolation, CHECK constraints (balance >= 0), and multi-row atomic updates (debit + credit in one transaction). NoSQL databases cannot guarantee these properties. At 50M txn/day (~600/sec), a single PostgreSQL primary handles this easily. Shard by user_id for 10x scale.

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
     Detect cycles: A -> B -> C -> A (cycle detection via DFS)
     Flag if cycle total > $500

  2. Recipient diversity check:
     User sends money to 10 new recipients in 24 hours -> suspicious
     Normal users: 1-3 unique recipients per week

  3. Round-trip detection:
     If A sends $X to B, and B sends $X (±5%) back to A within 24 hours -> flag
     Even without cycle: simple round-trip is suspicious

  Action: freeze wallet pending manual review if any detection fires.
```

### Regulatory Compliance

KYC verification required before enabling wallet. Transaction limits (daily/monthly). AML monitoring: flag large/unusual transfers. Data retention: 7 years minimum. Right to data export (GDPR). Audit logs immutable (append-only ledger_entries table, no UPDATE/DELETE).

