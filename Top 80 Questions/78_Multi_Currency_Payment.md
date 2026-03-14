# 78. Design a Multi-Currency Payment System

---

## 1. Functional Requirements (FR)

- **Accept payments in any currency**: Buyer pays in their local currency (EUR, GBP, JPY, INR...)
- **Settle in merchant's currency**: Convert and settle to merchant's preferred currency
- **Real-time exchange rates**: Use live FX rates for conversion at payment time
- **FX rate locking**: Lock exchange rate for a window (15-30 min) during checkout
- **Multi-currency wallets**: Users/merchants hold balances in multiple currencies
- **Cross-border transfers**: Send money internationally with transparent FX fees
- **FX markup/fee**: Configurable spread on exchange rates for revenue
- **Currency display**: Show prices in buyer's local currency across the platform

---

## 2. Non-Functional Requirements (NFRs)

- **Accuracy**: FX rates accurate to 6 decimal places; no rounding errors that benefit/harm users
- **Low Latency**: Currency conversion decision in < 50 ms
- **Consistency**: Locked FX rate honored even if market moves during checkout
- **Compliance**: Adhere to currency regulations per country (capital controls, sanctioned currencies)
- **Scale**: 10M+ cross-currency transactions/day
- **Availability**: 99.99%

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Supported currencies | 150+ |
| FX rate updates | Every 30 seconds from providers |
| Cross-currency transactions / day | 10M |
| FX rate lookups / sec | 50K |
| Locked rate records | 5M active at any time |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         REAL-TIME SERVING                              │
│                                                                        │
│  Buyer (EUR) --> API Gateway --> Payment Service                       │
│                                       |                                │
│            +--------------------------+---------------------------+    │
│            |                |                |                    |    │
│    +-------v------+ +------v-------+ +------v------+ +-----------v--+ │
│    | FX Service   | | Ledger       | | Payment     | | Compliance   | │
│    | (quote, lock,| | Service      | | Processor   | | Service      | │
│    |  convert)    | | (multi-ccy   | | (charge     | | (sanctions   | │
│    |              | |  double-     | |  buyer's    | |  check,      | │
│    |              | |  entry       | |  card in    | |  capital     | │
│    |              | |  posting)    | |  local ccy) | |  controls,   | │
│    +------+-------+ +------+-------+ +------+------+ |  country     | │
│           |                |                |         |  restrictions| │
│    +------v-------+ +------v-------+ +------v------+ +-----------+--+ │
│    | Redis        | | PostgreSQL   | | PSP Gateway |             |    │
│    | (live FX     | | (ledger      | | (Stripe,    |             |    │
│    |  rates,      | |  entries,    | |  Adyen —    |             |    │
│    |  rate locks, | |  balances,   | |  supports   |             |    │
│    |  conversion  | |  fx_locks,   | |  multi-ccy  |             |    │
│    |  cache)      | |  conversions)| |  natively)  |             |    │
│    +--------------+ +--------------+ +-------------+             |    │
│                                                                  |    │
│                                                           +------v--+ │
│                                                           | Kafka   | │
│                                                           | (payment| │
│                                                           |  events)| │
│                                                           +----+----+ │
│                                                                |      │
│                          +-------------------------------------+---+  │
│                          |                    |                     |  │
│                   +------v------+      +------v------+      +------v+ │
│                   | Notification|      | FX Risk     |      | Settle│ │
│                   | Service     |      | Engine      |      | ment  │ │
│                   | (receipt,   |      | (monitor net|      | Service│
│                   |  FX details)|      |  exposure   |      | (daily│ │
│                   +-------------+      |  per ccy,   |      |  batch│ │
│                                        |  auto-hedge |      |  pay- │ │
│                                        |  if > $1M)  |      |  outs)│ │
│                                        +-------------+      +-------+ │
│                                                                       │
│  FX RATE INGESTION (background):                                      │
│  +----------------+    +-----------+    +-----------+                  │
│  | Reuters API    |--->|           |--->|           |                  │
│  +----------------+    | Rate      |    |   Redis   |                  │
│  | ECB API        |--->| Ingestion |    | fx_rate:  |                  │
│  +----------------+    | Service   |--->| {base}:   |                  │
│  | Open Exchange  |--->| (poll 30s,|    | {quote}   |                  │
│  | Rates API      |    |  validate,|    |           |                  │
│  +----------------+    |  best bid)|    +-----------+                  │
│                        +-----------+                                   │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### FX Rate Management

```
Rate ingestion:
  Multiple FX rate providers (redundancy + best rate selection):
    - Reuters/Bloomberg: institutional rates
    - ECB (European Central Bank): reference rates
    - Open Exchange Rates API: retail rates
  
  Polling every 30 seconds. Store in Redis:
    fx_rate:{base}:{quote} -> { rate: 1.0845, provider: "reuters", updated_at: ts }
    e.g., fx_rate:EUR:USD -> 1.0845 (1 EUR = 1.0845 USD)

Rate with markup:
  Raw rate: 1.0845
  Platform markup: 2% (revenue)
  Buy rate (user buys USD with EUR): 1.0845 * 0.98 = 1.0628
  Sell rate (user sells USD for EUR): 1.0845 * 1.02 = 1.1062
  
  Spread: sell_rate - buy_rate = markup revenue

Rate triangulation:
  No direct rate for NGN->BRL?
  NGN -> USD -> BRL (via common intermediate currency)
  rate = ngn_usd_rate * usd_brl_rate
```

#### FX Rate Locking

```
Checkout flow with rate lock:

1. User sees price: "$100 USD" -> "Show in my currency" -> "€92.17 EUR"
   FX Service returns: rate = 1.0850, locked = false

2. User clicks "Pay €92.17":
   FX Service locks rate:
     INSERT INTO rate_locks (lock_id, from_ccy, to_ccy, rate, expires_at)
     VALUES ('lock-uuid', 'EUR', 'USD', 1.0850, NOW() + '15 min');
     
   Returns: lock_id to client

3. User completes payment within 15 min:
   Payment Service: validate lock_id is not expired
   Charge: €92.17 EUR -> convert at locked rate 1.0850 -> $100.00 USD to merchant
   
4. If user takes > 15 min:
   Lock expired -> re-quote with current rate
   New rate might be better or worse for user

Why lock rates?
  Without locking: user sees €92.17, takes 5 min to fill payment form,
  rate changes, charged €93.50. Bad UX. Possible complaints/chargebacks.
  
  With locking: platform absorbs FX risk for 15 min. At scale (10M txns/day),
  average FX movement in 15 min is < 0.05%. Risk is manageable.
```

#### Multi-Currency Ledger

```
Transaction: Buyer pays €92.17, Merchant receives $100.00

Ledger entries (4 entries, balanced per currency):

EUR ledger:
  Entry 1: { account: buyer_eur,     DEBIT,  €92.17 }
  Entry 2: { account: platform_eur,  CREDIT, €92.17 }

USD ledger:
  Entry 3: { account: platform_usd,  DEBIT, $100.00 }
  Entry 4: { account: merchant_usd, CREDIT, $100.00 }

FX conversion record:
  { from: €92.17 EUR, to: $100.00 USD, rate: 1.0850, lock_id: "lock-uuid" }

The platform's EUR account grows, USD account shrinks.
Platform periodically settles (sells EUR, buys USD in FX market) to rebalance.

Per-currency balancing:
  SUM(EUR debits) = SUM(EUR credits) ✓
  SUM(USD debits) = SUM(USD credits) ✓
  Cross-currency entries connected via FX conversion record.
```

---

## 5. APIs

```http
GET /api/v1/fx/quote?from=EUR&to=USD&amount=100
-> { "from": "EUR", "to": "USD", "amount": 100.00,
     "converted": 108.45, "rate": 1.0845, "markup": 0.02,
     "effective_rate": 1.0628, "valid_for_seconds": 30 }

POST /api/v1/fx/lock
{ "from_currency": "EUR", "to_currency": "USD", "rate": 1.0850, "ttl_seconds": 900 }
-> { "lock_id": "lock-uuid", "expires_at": "2026-03-14T11:15:00Z" }

POST /api/v1/payments/cross-currency
{
  "buyer_currency": "EUR", "buyer_amount": 92.17,
  "merchant_currency": "USD", "merchant_amount": 100.00,
  "fx_lock_id": "lock-uuid", "merchant_id": "m-uuid"
}
-> { "payment_id": "pay-uuid", "status": "completed",
     "fx_rate_used": 1.0850, "fx_fee": 1.84 }
```

---

## 6. Data Model

### Redis — Live Rates
```
fx_rate:{base}:{quote}  -> Hash { rate, bid, ask, provider, updated_at }
TTL: 60 (stale after 1 min without update)
```

### PostgreSQL — Locks & Ledger
```sql
CREATE TABLE rate_locks (
    lock_id UUID PRIMARY KEY, from_currency CHAR(3), to_currency CHAR(3),
    locked_rate DECIMAL(12,6), expires_at TIMESTAMPTZ, used BOOLEAN DEFAULT FALSE,
    payment_id UUID, created_at TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_expires (expires_at) WHERE used = FALSE
);

CREATE TABLE fx_conversions (
    conversion_id UUID PRIMARY KEY, payment_id UUID,
    from_currency CHAR(3), from_amount DECIMAL(18,2),
    to_currency CHAR(3), to_amount DECIMAL(18,2),
    rate_used DECIMAL(12,6), lock_id UUID,
    platform_fee DECIMAL(18,2), created_at TIMESTAMPTZ
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **FX provider down** | Multiple providers with fallback; use last known rate with staleness warning |
| **Rate lock expired mid-payment** | Extend lock by 5 min on active payment; or re-quote |
| **Rounding errors** | Use DECIMAL(18,6) for rates; banker's rounding; verify: from_amount * rate = to_amount ± 0.01 |
| **FX risk** | Platform treasury team hedges FX exposure daily; automated rebalancing |
| **Sanctioned currencies** | Country/currency blocklist enforced at API gateway level |

---

## 8. Deep Dive

### Settlement and Treasury

```
Platform accumulates various currencies throughout the day:
  EUR balance: +€500K, USD balance: -$542K, GBP balance: +£100K

Daily settlement:
  1. Calculate net position per currency
  2. Execute FX trades in wholesale market (better rates than retail)
  3. Transfer settled amounts to merchants' bank accounts
  4. T+1 or T+2 settlement (1-2 business days)

FX hedging:
  If platform knows it will receive €1M tomorrow (from European sales):
  Buy a EUR/USD forward contract today -> lock in today's rate
  Eliminates FX risk on the $1M settlement.
```

### Smallest Currency Units: Why Store Amounts in Minor Units

```
Different currencies have different decimal places:
  USD: 2 decimals ($1.00 = 100 cents)
  JPY: 0 decimals (¥100 = 100 yen, no fractional yen)
  BHD: 3 decimals (0.001 BHD = 1 fils)
  
Best practice: store amounts as integers in smallest unit.
  $100.00 -> 10000 (cents)
  ¥100 -> 100
  0.500 BHD -> 500 (fils)

Avoids all decimal arithmetic issues. Integer math is exact.
Display layer converts back to human-readable format with correct decimal places.
Currency metadata: { "USD": { "decimals": 2, "symbol": "$" }, "JPY": { "decimals": 0, ... } }
```

### FX Risk Management — Platform Exposure

```
Problem: Platform locks EUR/USD at 1.0850 for 15 min for buyer.
  If EUR/USD moves to 1.0750 in those 15 min, platform loses on the conversion.
  
  Per-transaction risk: small (< 0.1% typically)
  At 10M cross-currency txns/day x avg $50 = $500M daily volume
  0.1% adverse movement = $500K daily risk exposure

Mitigation:

1. Markup covers most risk:
   2% markup >> average 15-min FX movement (0.05%)
   Platform's spread is 40x the expected adverse movement
   Net: platform is profitable on FX in aggregate

2. Auto-hedging for large exposures:
   If net position in any currency exceeds $1M:
     Automatically execute FX hedge via wholesale market
     Forward contract: lock in the conversion rate
   
   FX Risk Engine (Flink):
     Consume payment events -> track net position per currency
     HSET fx_position:{currency} {net_amount}
     If abs(net_amount) > $1M:
       Trigger hedge order via FX broker API
       Log: hedged $X of {currency} exposure

3. Dynamic lock TTL based on volatility:
   Normal markets: lock TTL = 15 min
   High volatility (FX moves > 1% in 1 hour): lock TTL = 5 min
   Extreme volatility (FX moves > 3%): lock TTL = 2 min + larger markup
   
   Monitor: if rate at payment time differs from locked rate by > 1%,
   alert treasury team (potential loss on this transaction)
```

### Rate Staleness and Provider Failover

```
FX rate providers may fail or return stale data.

Multi-provider strategy:
  Priority order: Reuters (best quality) -> Bloomberg -> ECB -> OpenExchangeRates
  
  Every 30 seconds:
    1. Try primary provider (Reuters)
    2. If fails or response time > 2s: fall to Bloomberg
    3. If both fail: use ECB (updated hourly — less granular)
    4. If all fail: use last known rate with staleness flag
    
  Staleness rules:
    Rate < 1 min old: "live" (green)
    Rate 1-5 min old: "recent" (yellow) — acceptable
    Rate 5-30 min old: "stale" (orange) — show warning to user
    Rate > 30 min old: "expired" (red) — block new rate locks;
      serve with disclaimer "Exchange rate may have changed"
    
  Redis TTL management:
    fx_rate:{base}:{quote} — TTL 60s
    If not refreshed within 60s: key expires
    On read: if key missing, trigger immediate fetch from provider
    Serve stale value from fallback key:
      fx_rate_backup:{base}:{quote} — TTL 3600 (1 hour backup)

Provider disagreement:
  Reuters says EUR/USD = 1.0845
  Bloomberg says EUR/USD = 1.0852
  
  If difference > 0.5%: alert (someone may have a stale/wrong feed)
  Otherwise: use primary provider's rate
  For additional safety: use median of all available provider rates
```

### Payment Routing by Currency

```
Different PSPs have different strengths per currency/country:

  Stripe: excellent for USD, EUR, GBP. Reasonable for JPY, AUD.
  Adyen: strong in EU and Asia. Supports 150+ currencies natively.
  PayPal: good for cross-border consumer payments.
  Local PSPs: Razorpay (INR), iDEAL (NL), Alipay (CNY) — best local rates.

Smart routing:
  For INR payments: route to Razorpay (lower fees, higher approval rates)
  For EUR payments: route to Adyen (native EUR processing, no conversion)
  For USD payments: route to Stripe (lowest processing fee)
  
  Routing table (PostgreSQL):
    { currency: "INR", country: "IN", primary_psp: "razorpay", 
      fallback_psp: "stripe", fee_pct: 1.5 }
    { currency: "EUR", country: "DE", primary_psp: "adyen",
      fallback_psp: "stripe", fee_pct: 1.2 }
  
  Benefits:
    - Higher approval rates (local PSP understands local cards)
    - Lower fees (avoid double conversion)
    - Fewer chargebacks (local processing = local acquirer)

Fallback:
  If primary PSP returns error -> retry with fallback PSP
  If both fail -> return error to user
  Log routing decisions for fee optimization analytics
```

