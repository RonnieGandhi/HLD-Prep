# 77. Design a Distributed Ledger or Banking Ledger System

---

## 1. Functional Requirements (FR)

- **Double-entry bookkeeping**: Every transaction creates debit and credit entries that sum to zero
- **Account management**: Create accounts (checking, savings, loan, revenue, expense)
- **Post transactions**: Record financial transactions atomically
- **Balance inquiry**: Real-time balance for any account
- **Statement generation**: Account statement for any date range
- **Reconciliation**: Verify all entries balance (total debits = total credits)
- **Multi-currency**: Support transactions in multiple currencies with exchange rates
- **Immutable audit trail**: Entries cannot be modified or deleted, only corrected via reversals

---

## 2. Non-Functional Requirements (NFRs)

- **ACID Compliance**: Every transaction is atomic, consistent, isolated, durable
- **Immutability**: Ledger entries are append-only; no UPDATE or DELETE ever
- **Exactly-once**: No duplicate postings; idempotent by design
- **Auditability**: Complete history for regulatory compliance (7+ years retention)
- **Scale**: 1B+ ledger entries/day, 100M+ accounts
- **Low Latency**: Balance check < 10 ms; posting < 100 ms

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Accounts | 100M |
| Ledger entries / day | 1B (2 entries per transaction x 500M transactions) |
| Balance queries / sec | 100K |
| Posting requests / sec | 12K |
| Ledger entry size | ~500 bytes |
| Storage / day | 500 GB |
| Storage / year | ~180 TB |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         REAL-TIME POSTING                              │
│                                                                        │
│  Client --> API Gateway --> Ledger Service (stateless)                  │
│                                  |                                     │
│         +------------------------+------------------------+            │
│         |                        |                        |            │
│  +------v-------+        +-------v-------+        +-------v------+    │
│  | Transaction  |        | Balance       |        | Statement    |    │
│  | Posting      |        | Service       |        | Service      |    │
│  | Engine       |        |               |        |              |    │
│  | - validate   |        | - real-time   |        | - date range |    │
│  |   entries    |        |   balance     |        |   query      |    │
│  |   sum = 0    |        |   from cache  |        | - PDF/CSV    |    │
│  | - check      |        |   or PG       |        |   generation |    │
│  |   idempotency|        |               |        | - async via  |    │
│  | - execute in |        |               |        |   S3         |    │
│  |   ACID txn   |        |               |        |              |    │
│  +------+-------+        +-------+-------+        +-------+------+    │
│         |                        |                        |            │
│  +------v------------------------------------------------v------+    │
│  |                    PostgreSQL (Primary)                        |    │
│  | ledger_entries (partitioned monthly)                           |    │
│  | account_balances (running balance)                             |    │
│  | transactions (idempotency key UNIQUE)                          |    │
│  | accounts (checking, savings, loan, revenue, expense)           |    │
│  +------+---------------------------+----------------------------+    │
│         |                           |                                  │
│  +------v-------+           +-------v-------+                         │
│  | PostgreSQL   |           |    Redis      |                         │
│  | (Sync        |           | - balance     |                         │
│  |  Replica     |           |   cache       |                         │
│  |  for reads   |           | - idempotency |                         │
│  |  + standby)  |           |   keys        |                         │
│  +--------------+           | - rate limits |                         │
│                             +---------------+                         │
│                                                                        │
│  +------v-------+     +----------------+     +-------------------+    │
│  |    Kafka     |     | Notification   |     | ClickHouse        |    │
│  | (ledger-     |---->| Service        |     | (analytics:       |    │
│  |  events)     |     | (txn receipts, |     |  daily volumes,   |    │
│  |              |---->|  low balance   |     |  account trends,  |    │
│  |              |     |  alerts)       |     |  anomaly detect)  |    │
│  +--------------+     +----------------+     +-------------------+    │
│                                                                        │
│  BATCH PROCESSING (nightly):                                           │
│  +------------------+     +--------------------+     +--------------+ │
│  | Reconciliation   |     | Interest           |     | Archival     | │
│  | Service          |     | Accrual Engine     |     | Service      | │
│  | (recompute all   |     | (compute daily     |     | (move old    | │
│  |  balances from   |     |  interest for      |     |  partitions  | │
│  |  entries;        |     |  savings accounts; |     |  to S3       | │
│  |  compare with    |     |  post interest     |     |  Glacier)    | │
│  |  running balance;|     |  entries to ledger)|     |              | │
│  |  alert on diff)  |     |                    |     |              | │
│  +------------------+     +--------------------+     +--------------+ │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Double-Entry Transaction Posting

```
Every financial event creates balanced entries:

Transfer $500 from Account A to Account B:

  Entry 1: { account: A, type: DEBIT,  amount: 500.00, tx_id: "txn-123" }
  Entry 2: { account: B, type: CREDIT, amount: 500.00, tx_id: "txn-123" }
  
  SUM = -500 + 500 = 0 ✓

Interest accrual on savings account:
  Entry 1: { account: interest_expense, type: DEBIT,  amount: 12.50 }
  Entry 2: { account: savings_acct,     type: CREDIT, amount: 12.50 }

Fee charge:
  Entry 1: { account: customer_checking, type: DEBIT,  amount: 5.00 }
  Entry 2: { account: fee_revenue,       type: CREDIT, amount: 5.00 }

Atomic posting (PostgreSQL transaction):
  BEGIN;
    -- Validate: sum of entries for this transaction = 0
    -- Insert all entries
    INSERT INTO ledger_entries (entry_id, transaction_id, account_id,
      entry_type, amount, currency, posted_at) VALUES
      (uuid1, 'txn-123', 'acct-A', 'debit', 500.00, 'USD', NOW()),
      (uuid2, 'txn-123', 'acct-B', 'credit', 500.00, 'USD', NOW());
    
    -- Update running balances
    UPDATE account_balances SET balance = balance - 500.00 WHERE account_id = 'acct-A';
    UPDATE account_balances SET balance = balance + 500.00 WHERE account_id = 'acct-B';
    
    -- Validate: no account goes below minimum (e.g., 0 for checking)
    SELECT balance FROM account_balances WHERE account_id = 'acct-A';
    -- If balance < 0 AND account type requires non-negative: ROLLBACK
  COMMIT;

Key invariants:
  1. SUM(all debit amounts) = SUM(all credit amounts) globally (always)
  2. Account balance = SUM(credits) - SUM(debits) for that account
  3. No entry is ever modified or deleted (append-only)
```

#### Balance Computation — Running Balance vs Calculated

```
Approach 1: Calculated balance (sum all entries):
  SELECT SUM(CASE WHEN entry_type='credit' THEN amount ELSE -amount END)
  FROM ledger_entries WHERE account_id = 'acct-A'
  
  Problem: scanning millions of entries per query -> slow (seconds for old accounts)

Approach 2: Running balance (maintained in separate table):
  account_balances: { account_id, balance, last_updated }
  Updated atomically with each posting (within same transaction)
  
  Balance query: SELECT balance FROM account_balances WHERE account_id = ?
  O(1) lookup, < 1 ms.
  
  Cached in Redis: balance:{account_id} -> DECIMAL, TTL 30s
  Invalidated on every posting.

Approach 3: Periodic checkpoint + delta:
  checkpoint_balances: { account_id, balance, checkpoint_date }
  Updated nightly via batch computation.
  
  Current balance = checkpoint_balance + SUM(entries since checkpoint)
  Limits scan to recent entries only.
  
  Used for: reconciliation, disaster recovery.

We use Approach 2 (running balance) for real-time + Approach 3 for reconciliation.
```

#### Immutability — How to "Fix" Errors

```
Entry is wrong. Cannot UPDATE or DELETE. How to correct?

Reversal: post a new entry that reverses the original.

Original (wrong): Charged $50 fee instead of $5
  Entry 1: { acct: customer, DEBIT, $50, reason: "monthly fee" }
  Entry 2: { acct: fee_rev,  CREDIT, $50 }

Reversal:
  Entry 3: { acct: customer, CREDIT, $50, reason: "reversal of entry 1", reverses: entry_1 }
  Entry 4: { acct: fee_rev,  DEBIT, $50, reverses: entry_2 }

Correct posting:
  Entry 5: { acct: customer, DEBIT, $5, reason: "monthly fee (corrected)" }
  Entry 6: { acct: fee_rev,  CREDIT, $5 }

Net effect: customer charged $5 (correct). Full audit trail preserved.
Regulators can see: original error, reversal, and correction.
```

---

## 5. APIs

```http
POST /api/v1/ledger/post
Idempotency-Key: "txn-uuid-123"
{
  "transaction_id": "txn-123",
  "description": "Transfer A to B",
  "entries": [
    { "account_id": "acct-A", "type": "debit", "amount": 500.00, "currency": "USD" },
    { "account_id": "acct-B", "type": "credit", "amount": 500.00, "currency": "USD" }
  ]
}
-> 200 { "transaction_id": "txn-123", "posted_at": "...", "status": "posted" }

GET /api/v1/accounts/{account_id}/balance
-> { "account_id": "acct-A", "balance": 2450.00, "currency": "USD", "as_of": "..." }

GET /api/v1/accounts/{account_id}/statement?from=2026-03-01&to=2026-03-14
-> { "entries": [...], "opening_balance": 2950.00, "closing_balance": 2450.00 }
```

---

## 6. Data Model

### PostgreSQL — Ledger (Partitioned by Month)

```sql
CREATE TABLE ledger_entries (
    entry_id        UUID PRIMARY KEY,
    transaction_id  UUID NOT NULL,
    account_id      UUID NOT NULL,
    entry_type      ENUM('debit','credit') NOT NULL,
    amount          DECIMAL(18,2) NOT NULL CHECK (amount > 0),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    balance_after   DECIMAL(18,2) NOT NULL,
    description     TEXT,
    reverses_entry  UUID, -- points to original entry if this is a reversal
    idempotency_key VARCHAR(64),
    posted_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    INDEX idx_account (account_id, posted_at DESC),
    INDEX idx_transaction (transaction_id)
) PARTITION BY RANGE (posted_at);

-- Monthly partitions for efficient querying and archival
CREATE TABLE ledger_entries_2026_03 PARTITION OF ledger_entries
  FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

CREATE TABLE account_balances (
    account_id UUID PRIMARY KEY,
    balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    currency CHAR(3) DEFAULT 'USD',
    last_transaction_id UUID,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Global constraint: sum of all entries must be 0
-- Verified by nightly reconciliation job, not per-transaction (too expensive)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Partial posting** | All entries in single DB transaction; all-or-nothing |
| **Duplicate posting** | Idempotency key with UNIQUE constraint |
| **Balance drift** | Nightly reconciliation: recompute all balances from entries, compare with running balances |
| **Data loss** | Synchronous replication; WAL archiving to S3; point-in-time recovery |
| **Immutability violation** | No UPDATE/DELETE permissions on ledger_entries; DB user has INSERT-only privilege |
| **Regulatory audit** | 7-year retention; partitioned tables with cold storage (S3 Glacier for old partitions) |

---

## 8. Deep Dive

### Why PostgreSQL (Not Blockchain/DynamoDB)

```
Blockchain: decentralized trust, but orders of magnitude slower (7-30 TPS for Bitcoin).
  Banking ledger doesn't need decentralized trust — the bank IS the trusted authority.
  PostgreSQL: 12K TPS easily. Blockchain: overkill and too slow.

DynamoDB: no multi-row transactions. Can't atomically post debit + credit entries.
  DynamoDB transactions: 25-item limit, eventually consistent reads by default.
  Financial ledger REQUIRES strong consistency and multi-row atomic writes.

PostgreSQL: ACID, serializable isolation, CHECK constraints, partitioning for time-series,
  rich SQL for statements and reconciliation queries. The right tool for this job.
```

### Decimal Precision: Why DECIMAL(18,2), Not FLOAT

```
FLOAT/DOUBLE: binary floating point. 0.1 + 0.2 = 0.30000000000000004.
  Accumulated over millions of transactions: significant errors.
  Unacceptable for financial systems.

DECIMAL(18,2): exact decimal arithmetic. 0.1 + 0.2 = 0.3. Always.
  18 digits total, 2 after decimal point.
  Max value: 9,999,999,999,999,999.99 ($10 quadrillion).
  Sufficient for any banking system.

In application code: use BigDecimal (Java), Decimal (Python), not float/double.
```

### End-of-Day Processing — Nightly Batch

```
Banks close their books every day. Nightly batch processes:

1. Interest accrual:
   For each savings account:
     daily_interest = balance * annual_rate / 365
     Post entries: DEBIT interest_expense, CREDIT customer_savings
   
   For each loan account:
     daily_interest = outstanding_balance * annual_rate / 365
     Post entries: DEBIT customer_loan, CREDIT interest_income

2. Reconciliation:
   For each account:
     recomputed_balance = SUM(credits) - SUM(debits) from ledger_entries
     if recomputed_balance != running_balance in account_balances:
       ALERT: "Account {account_id} has balance mismatch of {delta}"
       DO NOT auto-correct (human investigation required for financial systems)
   
   Global check: SUM(all entries) == 0 (double-entry invariant)
   If not zero: CRITICAL ALERT -> halt all processing until resolved

3. Statement generation:
   For each account: generate monthly statement (1st of each month)
   SELECT * FROM ledger_entries WHERE account_id = ? 
     AND posted_at BETWEEN ? AND ?
   ORDER BY posted_at
   
   Render PDF -> upload to S3 -> notify customer

4. Archival:
   Partitions older than 2 years: move to cold storage (S3 Glacier)
   Create: CREATE TABLE ledger_entries_2024_01 ... TABLESPACE cold_storage
   Keep hot: last 2 years on fast NVMe storage

Processing window: 11 PM - 6 AM (7 hours)
  1B entries/day -> reconciliation takes ~30 minutes (parallel per account shard)
  Interest accrual for 100M accounts: ~2 hours
```

### Multi-Entity Journal Entries

```
Complex transactions may involve more than 2 accounts:

Loan payment with interest:
  Customer pays $1,000 monthly payment:
    $200 goes to interest -> CREDIT interest_income
    $800 goes to principal -> CREDIT loan_receivable
    $1,000 from customer -> DEBIT customer_checking
  
  Entry 1: { account: customer_checking,   DEBIT,  $1,000 }
  Entry 2: { account: interest_income,     CREDIT, $200 }
  Entry 3: { account: loan_receivable,     CREDIT, $800 }
  
  SUM: -1000 + 200 + 800 = 0 ✓ (balanced)

Validation before posting:
  def validate_journal_entry(entries):
      total = sum(e.amount if e.type == 'credit' else -e.amount for e in entries)
      if total != 0:
          raise UnbalancedEntryError(f"Entries sum to {total}, expected 0")
      if len(entries) < 2:
          raise InvalidEntryError("Journal entry needs at least 2 entries")
      if len(set(e.currency for e in entries)) > 1:
          raise MixedCurrencyError("All entries must be in same currency")
  
  This validation runs BEFORE the PostgreSQL transaction opens.
  Defense in depth: PostgreSQL trigger also validates sum = 0 per transaction_id.
```
