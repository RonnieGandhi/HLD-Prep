# 93. Design a Cryptocurrency Exchange

---

## 1. Functional Requirements (FR)

- Order placement: limit, market, stop-loss orders for crypto-to-crypto and crypto-to-fiat pairs
- Order matching engine: price-time priority (same as stock exchange)
- Wallet management: deposit, withdraw crypto (on-chain) and fiat (bank transfer)
- Hot wallet (online, for fast withdrawals) and cold wallet (offline, for security)
- Real-time order book and trade feed (WebSocket)
- Portfolio: view balances, P&L, transaction history
- KYC/AML: identity verification, transaction monitoring
- Multi-factor authentication (2FA, hardware keys)
- Trading fees with tiered pricing (maker/taker)
- Staking and lending features

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Order matching < 1ms; API response < 50ms
- **High Availability**: 99.99% (downtime during volatility = massive losses)
- **Security**: Cold wallet for 95% of assets; HSM for key management; no single point of compromise
- **Consistency**: Account balances must NEVER be wrong (double-spend prevention)
- **Auditability**: Every transaction traceable for regulatory compliance
- **Throughput**: 100K orders/sec peak (during market events)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Registered users | 50M |
| DAU | 5M |
| Trading pairs | 500 |
| Orders / sec (peak) | 100K |
| Trades / sec (peak) | 50K |
| WebSocket connections | 2M concurrent |
| Wallet transactions / day | 1M (deposits + withdrawals) |
| Hot wallet balance | 5% of total assets ($500M if $10B total) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    TRADING LAYER                                       │
│                                                                        │
│  Client (Web/Mobile/API)                                               │
│     │                                                                  │
│     ├── REST API (order placement, portfolio, history)                 │
│     └── WebSocket (order book, trades, ticker, user events)            │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  API Gateway                                               │        │
│  │  - Rate limiting, JWT auth, IP whitelisting (for API keys)│        │
│  │  - Request validation, HMAC signature verification        │        │
│  └──────────────────────┬────────────────────────────────────┘        │
│                         │                                              │
│  ┌──────────────────────▼────────────────────────────────────┐        │
│  │  Order Service                                             │        │
│  │  - Pre-trade balance check (Redis: available balance)     │        │
│  │  - Reserve funds (Redis DECR atomically)                  │        │
│  │  - Route to correct matching engine shard (by trading pair)│       │
│  └──────────────────────┬────────────────────────────────────┘        │
│                         │                                              │
│  ┌──────────────────────▼────────────────────────────────────┐        │
│  │  Matching Engine (one per trading pair, single-threaded)   │        │
│  │  - Price-time priority order book (same as #76)           │        │
│  │  - In-memory sorted data structures (TreeMap bids/asks)   │        │
│  │  - WAL for durability (persist before ACK)                │        │
│  │  - Produces: trades, order updates, book updates          │        │
│  └──────────────────────┬────────────────────────────────────┘        │
│                         │                                              │
│  ┌──────────────────────▼────────────────────────────────────┐        │
│  │  Post-Trade Processing                                     │        │
│  │  - Settlement: update balances (PostgreSQL + Redis)       │        │
│  │  - Fee calculation (maker/taker, tier-based)              │        │
│  │  - Trade event → Kafka → WebSocket broadcast             │        │
│  │  - Audit log (immutable, append-only)                     │        │
│  └───────────────────────────────────────────────────────────┘        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    WALLET LAYER (Critical Security)                     │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Hot Wallet Service (5% of assets, online)                 │        │
│  │  - Auto-process small withdrawals                         │        │
│  │  - HSM (Hardware Security Module) for signing transactions│        │
│  │  - Multi-sig: 2-of-3 signatures required                 │        │
│  │  - Per-user and global withdrawal limits                  │        │
│  │  - Anomaly detection triggers manual review               │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Cold Wallet (95% of assets, air-gapped)                  │        │
│  │  - Offline signing ceremony (multi-sig, 3-of-5)           │        │
│  │  - Geo-distributed key shards (3 continents)              │        │
│  │  - Scheduled replenishment: cold → hot (daily)            │        │
│  │  - Manual approval required for large movements           │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Blockchain Watchers (one per supported chain)             │        │
│  │  - Monitor deposit addresses for incoming transactions    │        │
│  │  - Wait for N confirmations before crediting:             │        │
│  │    Bitcoin: 3 confirmations (~30 min)                     │        │
│  │    Ethereum: 12 confirmations (~3 min)                    │        │
│  │  - Detect chain reorgs → reverse unconfirmed credits      │        │
│  │  - Gas optimization for batch withdrawals (ETH)           │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    SECURITY & COMPLIANCE LAYER                         │
│                                                                        │
│  KYC Service ──── Fraud Detection ──── AML Monitoring                  │
│  (identity         (withdrawal         (transaction                    │
│   verification)     anomalies)          pattern analysis)              │
│                                                                        │
│  Circuit Breaker: halt trading during suspected hack                  │
│  Proof of Reserves: periodic auditing (Merkle tree of balances)       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 5. APIs

```
POST /api/orders              → Place order {pair, side, type, quantity, price}
DELETE /api/orders/{id}       → Cancel order
GET /api/orders?status=open   → Open orders
GET /api/orderbook/{pair}     → Order book snapshot
GET /api/trades/{pair}        → Recent trades
GET /api/wallet/balances      → Portfolio balances
POST /api/wallet/withdraw     → Initiate withdrawal (2FA required)
GET /api/wallet/deposit_address/{coin}  → Get deposit address

# WebSocket streams
WS /ws/orderbook/{pair}       → Real-time order book
WS /ws/trades/{pair}          → Real-time trades
WS /ws/user                   → User order updates, balance changes
```

---

## 6. Data Models

### PostgreSQL (Ledger — Source of Truth)

**Why PostgreSQL?** ACID for financial data. DECIMAL(28,18) handles crypto precision (18 decimals for ETH). SELECT FOR UPDATE prevents double-spend race conditions.

```sql
CREATE TABLE balances (
    user_id   UUID,
    currency  TEXT,
    available DECIMAL(28,18) NOT NULL DEFAULT 0 CHECK (available >= 0),
    reserved  DECIMAL(28,18) NOT NULL DEFAULT 0 CHECK (reserved >= 0),
    PRIMARY KEY (user_id, currency)
);

CREATE TABLE orders (
    order_id    UUID PRIMARY KEY,
    user_id     UUID NOT NULL,
    pair        TEXT NOT NULL,        -- e.g., BTC_USDT
    side        TEXT NOT NULL,        -- buy|sell
    order_type  TEXT NOT NULL,        -- limit|market|stop_loss
    price       DECIMAL(28,18),      -- null for market orders
    quantity    DECIMAL(28,18) NOT NULL,
    filled_qty  DECIMAL(28,18) DEFAULT 0,
    status      TEXT DEFAULT 'open', -- open|partially_filled|filled|cancelled
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE trades (
    trade_id    UUID PRIMARY KEY,
    pair        TEXT NOT NULL,
    buyer_id    UUID NOT NULL,
    seller_id   UUID NOT NULL,
    buy_order   UUID REFERENCES orders(order_id),
    sell_order  UUID REFERENCES orders(order_id),
    price       DECIMAL(28,18) NOT NULL,
    quantity    DECIMAL(28,18) NOT NULL,
    buyer_fee   DECIMAL(28,18),
    seller_fee  DECIMAL(28,18),
    executed_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE withdrawals (
    withdrawal_id UUID PRIMARY KEY,
    user_id       UUID NOT NULL,
    currency      TEXT NOT NULL,
    amount        DECIMAL(28,18) NOT NULL,
    address       TEXT NOT NULL,       -- blockchain destination
    tx_hash       TEXT,                -- filled after broadcast
    status        TEXT DEFAULT 'pending', -- pending|processing|broadcast|confirmed|failed
    confirmations INT DEFAULT 0,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis (Hot Balances + Order Book Cache)

```
# Fast balance check (authoritative is PG, Redis is speed layer)
HSET balance:{user_id} BTC_available "1.234" BTC_reserved "0.5" ETH_available "10.0"

# Atomic reservation via Lua script:
-- KEYS[1] = balance:{user_id}, ARGS[1] = currency_available, ARGS[2] = amount
local avail = tonumber(redis.call('HGET', KEYS[1], ARGV[1]))
if avail >= tonumber(ARGV[2]) then
    redis.call('HINCRBYFLOAT', KEYS[1], ARGV[1], -tonumber(ARGV[2]))
    redis.call('HINCRBYFLOAT', KEYS[1], ARGV[3], tonumber(ARGV[2]))
    return 1
end
return 0

# Order book snapshot cache (per pair)
SET orderbook:BTC_USDT '{bids:[...],asks:[...]}' EX 1
```

---

## 7. Fault Tolerance & Deep Dives

### Component Deep Dive: Order Lifecycle

```
1. User submits order: POST /api/orders {pair:BTC_USDT, side:buy, price:50000, qty:0.5}
2. Order Service:
   a. Validate: user authenticated, pair exists, qty > min
   b. Reserve funds: Lua script in Redis → USDT_available -= 25000, USDT_reserved += 25000
   c. If Redis reserve fails → reject immediately (insufficient balance)
   d. Persist order (status=open) to PostgreSQL
   e. Persist reservation to PostgreSQL (BEGIN; SELECT FOR UPDATE; UPDATE; COMMIT)
   f. Route to matching engine for this pair

3. Matching Engine (single-threaded, in-memory per pair):
   a. Insert order into order book (TreeMap of price levels)
   b. Match against opposite side:
      - Buy at $50,000 vs best ask at $49,950 → MATCH at $49,950
   c. Generate trade event: {buyer, seller, price, qty}
   d. Write trade to WAL (append-only file) BEFORE returning
   e. Publish trade event to Kafka

4. Post-Trade Settlement (async from Kafka):
   a. Buyer: USDT_reserved -= 24975 (trade amount), BTC_available += 0.5
   b. Seller: BTC_reserved -= 0.5, USDT_available += 24975
   c. Deduct fees (maker 0.1%, taker 0.15%)
   d. Update order status (filled/partially_filled)
   e. All inside PostgreSQL transaction

5. WebSocket broadcast:
   a. Order book update → all subscribers of BTC_USDT
   b. Trade feed → public subscribers
   c. User events → buyer and seller individually
```

### Race Conditions

#### 1. Double-Spend (Same Balance Used Twice)

```
Problem: User has 1 BTC, submits two sell orders for 1 BTC simultaneously

Solution (Redis + PG two-layer):
  Redis Lua: atomic check-and-decrement (single-threaded, no race)
    First order: available=1 → reserve → available=0, reserved=1 ✓
    Second order: available=0 → reject ✗
  
  PostgreSQL: authoritative backup
    Periodic reconciliation: if Redis drifts from PG → PG wins
    On Redis failure → fall back to PG SELECT FOR UPDATE (slower but correct)
```

#### 2. Order Cancel During Matching

```
Problem: User cancels order while matching engine is processing it

Solution:
  Matching engine is single-threaded per pair → cancel queued behind match
  If order partially filled before cancel:
    - Fill what was already matched
    - Cancel remaining quantity
    - Return unreserved funds: reserved -= remaining_qty × price
  Cancel is idempotent: cancel already-filled order → no-op
```

#### 3. Withdrawal After Balance Used in Trade

```
Problem: User requests withdrawal of BTC while a trade settles that changes BTC balance

Solution:
  Withdrawal deducts from available (not reserved)
  Trade settlement deducts from reserved → different pool
  PostgreSQL transaction ensures atomicity:
    BEGIN;
    SELECT available FROM balances WHERE user_id=$1 AND currency='BTC' FOR UPDATE;
    IF available < withdrawal_amount THEN ROLLBACK;
    UPDATE balances SET available = available - $amount;
    INSERT INTO withdrawals (...);
    COMMIT;
```

### Matching Engine Deep Dive

```
Data structure: Two TreeMaps (Red-Black Trees) per pair
  Bids: sorted by price DESC (highest first), then time ASC
  Asks: sorted by price ASC (lowest first), then time ASC

Price-Time Priority:
  Same price → earlier order fills first (FIFO within price level)

Processing:
  New buy limit at $50,000:
    1. Check asks: best ask = $49,950 → match!
    2. Fill at $49,950 (price improvement for buyer)
    3. If buyer's qty > ask qty → continue matching next ask
    4. If buyer's qty satisfied → stop
    5. If no more matching asks → insert remaining into bid book

Single-threaded: ONE thread per trading pair
  Why? No locks needed → predictable µs latency
  One pair: BTC/USDT gets dedicated thread
  100 pairs: 100 threads (each independent)
  
WAL: Every match written to append-only file BEFORE publish
  On crash → replay WAL → rebuild order book state

Performance: 100K+ matches/sec per pair (in-memory, single-threaded)
  Reference: LMAX Disruptor achieves 6M orders/sec
```

### Deposit & Withdrawal Flow

```
DEPOSIT:
  1. User requests deposit address → system generates HD wallet address (unique per user per chain)
  2. Blockchain Watcher detects incoming transaction
  3. Wait for N confirmations (BTC:3, ETH:12, SOL:32)
  4. Credit user: UPDATE balances SET available = available + amount
  5. Edge case: chain reorg → reverse credit if tx disappears

WITHDRAWAL:
  1. User submits withdrawal (2FA required)
  2. Anomaly check: amount > daily limit? → flag for manual review
  3. Deduct from available balance (PG transaction)
  4. Hot wallet service signs transaction (HSM multi-sig 2-of-3)
  5. Broadcast to blockchain
  6. Blockchain Watcher monitors for confirmation
  7. Update status: broadcast → confirmed (after N confirmations)
  
  If hot wallet balance < withdrawal amount:
    Queue withdrawal → trigger cold→hot replenishment (manual ceremony)
    Notify user: "withdrawal processing, expected completion: 24h"
```

### Exchange Hack Prevention (Defense in Depth)

```
1. Hot wallet ≤ 5% of assets → limits blast radius of any breach
2. Multi-sig: hot=2-of-3, cold=3-of-5 → no single key compromise
3. HSM (Hardware Security Module) → keys never in software memory
4. Withdrawal anomaly detection → auto-freeze + human review
5. IP whitelisting for withdrawal addresses (user-configurable)
6. 24h withdrawal lock after password change
7. Circuit breaker: unusual trading volume/patterns → halt market
8. Proof of Reserves: Merkle tree of all balances → public audit
9. Bug bounty program → crowd-sourced vulnerability discovery
```

### Blockchain Reorg Handling

```
Problem: Chain reorganization can invalidate confirmed transactions

Bitcoin example:
  Block 800,000 confirms user deposit of 1 BTC
  User trades 1 BTC → 50,000 USDT
  Chain reorg: block 800,000 replaced → deposit NEVER happened
  User has 50,000 USDT but deposited nothing → exchange loses

Solution:
  1. Wait for N confirmations before crediting (BTC:3 ≈ 30min)
  2. Blockchain Watcher compares block hashes continuously
  3. If reorg detected deeper than N blocks → reverse pending credits
  4. NEVER let users trade unconfirmed deposits
  5. For high-value deposits: wait for more confirmations (BTC:6 for >$100K)
```

---

## 8. Additional Considerations

### Proof of Reserves
```
Merkle tree: each leaf = hash(user_id, balance)
Root published monthly
Users can verify their inclusion without revealing other users' balances
Third-party auditor verifies total on-chain assets ≥ tree total
```

### Market Manipulation Detection
```
Wash trading: same entity buying from themselves → detect via IP/device/timing correlation
Spoofing: large order placed and cancelled quickly → detect via order lifetime analysis
Layering: multiple orders at different prices to create false depth → pattern detection
Circuit breaker: price moves >10% in 5min → halt trading for 5min cooldown
```

### vs Stock Exchange (#76)
```
Stock exchange: regulated, T+2 settlement, central clearing house
Crypto exchange: self-custody, instant settlement, on-chain withdrawal
Stock: assets held by DTCC → no custody risk
Crypto: exchange holds private keys → hack risk → hot/cold wallet architecture
Stock: fiat-only, standard hours
Crypto: multi-currency, 24/7/365 operation
```

