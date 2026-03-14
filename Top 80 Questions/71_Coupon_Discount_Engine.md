# 71. Design a Coupon and Discount Engine

---

## 1. Functional Requirements (FR)

- **Create coupons**: Percentage off, fixed amount, BOGO, free shipping, tiered discounts
- **Apply coupons**: Validate and apply coupon code at checkout
- **Auto-apply promotions**: Automatic discounts (site-wide sales, category discounts) without code
- **Stacking rules**: Define which coupons can combine (stackable vs exclusive)
- **Eligibility rules**: Target by user segment, purchase history, cart value, product category, first-time buyer
- **Usage limits**: Per-coupon limit (10,000 total), per-user limit (1 per customer), time-bound
- **Coupon generation**: Bulk generate unique codes for campaigns (100K codes)
- **Analytics**: Track redemption rates, revenue impact, popular coupons

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Coupon validation in < 50 ms (in checkout critical path)
- **Strong Consistency**: Usage count must be accurate — no over-redemption
- **Scale**: 100K+ active coupons, 10M+ redemptions/day
- **Availability**: 99.99% — coupon failure blocks checkout
- **Fraud Resistant**: Prevent coupon abuse (sharing, scripted redemption)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active coupons | 100K |
| Coupon validation / sec | 5K |
| Redemptions / day | 10M |
| Bulk code generation | Up to 1M codes per campaign |

---

## 4. High-Level Design (HLD)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SERVING PATH                                  │
│                                                                      │
│  Client (Checkout) --> API Gateway --> Coupon Service (stateless)    │
│                                            |                         │
│           +--------------------------------+------------------+      │
│           |                |                |                 |      │
│   +-------v------+ +------v-------+ +------v------+ +-------v----+ │
│   | Rule Engine  | | Redis        | | PostgreSQL  | | User       | │
│   | (in-process, | | (usage       | | (coupon     | | Segment    | │
│   |  evaluate    | |  counters,   | |  definitions| | Service    | │
│   |  conditions  | |  per-user    | |  redemption | | (check     | │
│   |  against     | |  limits,     | |  history)   | |  membership| │
│   |  cart JSON)  | |  coupon      | |             | |  first-time| │
│   |              | |  cache)      | |             | |  buyer)    | │
│   +--------------+ +--------------+ +------+------+ +------------+ │
│                                            |                         │
│                                     +------v------+                  │
│                                     |    Kafka    |                  │
│                                     | (coupon-    |                  │
│                                     |  events)    |                  │
│                                     +------+------+                  │
│                                            |                         │
│                          +-----------------+------------------+      │
│                          |                 |                  |      │
│                   +------v------+   +------v------+   +------v----+ │
│                   | Analytics   |   | Fraud       |   | Auto-Apply| │
│                   | Pipeline    |   | Detection   |   | Service   | │
│                   | (ClickHouse:|   | (multi-acct |   | (evaluate | │
│                   |  redemption |   |  same IP,   |   |  all auto-| │
│                   |  rates,     |   |  velocity,  |   |  promos   | │
│                   |  revenue    |   |  coupon     |   |  against  | │
│                   |  impact,    |   |  aggregator |   |  cart at  | │
│                   |  A/B test)  |   |  detection) |   |  page load| │
│                   +-------------+   +-------------+   +-----------+ │
└──────────────────────────────────────────────────────────────────────┘

ADMIN PATH:
  Admin Dashboard --> Coupon Admin Service --> PostgreSQL (CRUD coupons)
                                           --> Bulk Code Generator (async, Kafka)
                                           --> Campaign Analytics (ClickHouse)
```

### Component Deep Dive

#### Coupon Rule Engine

```
A coupon has complex eligibility rules. Use a rule engine pattern:

Coupon definition:
{
  "coupon_id": "SUMMER25",
  "type": "percentage",
  "value": 25,
  "conditions": {
    "min_cart_value": 100.00,
    "eligible_categories": ["electronics", "clothing"],
    "eligible_user_segments": ["premium_members"],
    "first_purchase_only": false,
    "max_uses_total": 10000,
    "max_uses_per_user": 1,
    "valid_from": "2026-03-01",
    "valid_to": "2026-03-31",
    "stackable": false,
    "excluded_skus": ["SKU-GIFT-CARD"]
  },
  "discount_cap": 50.00  // max $50 off regardless of cart size
}

Validation flow (must be fast — in checkout path):
  1. Lookup coupon: Redis cache or PostgreSQL
  2. Check validity period: valid_from <= NOW <= valid_to
  3. Check total usage: Redis INCR coupon_usage:{coupon_id} — if > max_uses -> reject
  4. Check per-user usage: Redis GET user_coupon:{user_id}:{coupon_id}
  5. Evaluate conditions against cart:
     - Cart total >= min_cart_value?
     - Cart contains eligible categories?
     - User belongs to eligible segment?
  6. If all pass -> calculate discount and return

Discount calculation:
  percentage: discount = min(cart_subtotal * value/100, discount_cap)
  fixed: discount = min(value, cart_subtotal)  // can't exceed cart
  bogo: discount = price of cheapest qualifying item
  tiered: spend $100 get $10 off, spend $200 get $30 off
  free_shipping: discount = shipping_cost
```

#### Stacking Rules — Which Coupons Combine?

```
User applies two coupons: "SUMMER25" (25% off) + "FREESHIP" (free shipping)

Stacking policies:
  1. No stacking: only one coupon per order (simplest, Amazon's approach)
  2. Category stacking: one coupon per category (percentage + shipping OK)
  3. Full stacking: any coupons can combine (rare, complex)

Implementation:
  Each coupon has: stackable = true/false, stack_group = "percentage" | "shipping" | "fixed"
  
  Rule: max one coupon per stack_group.
  "SUMMER25" (stack_group: "percentage") + "FREESHIP" (stack_group: "shipping") -> OK
  "SUMMER25" (percentage) + "FALL20" (percentage) -> REJECTED (same group)
  
  Application order matters:
    Option A: Apply percentage THEN fixed: $100 - 25% = $75 - $10 = $65
    Option B: Apply fixed THEN percentage: $100 - $10 = $90 - 25% = $67.50
    Standard: apply most beneficial order for the customer (or fixed order per policy)
```

#### Race Condition: Coupon Usage Limit

```
Coupon "FLASH50" has max_uses = 1000. 1,500 users try to use it simultaneously.

Without atomicity: 1,500 users all read count=999, all increment, all succeed -> oversold

Solution: Redis INCR with Lua script (same pattern as flash sale):

  local current = redis.call('INCR', 'coupon_usage:FLASH50')
  if current > 1000 then
    redis.call('DECR', 'coupon_usage:FLASH50')
    return 0  -- exceeded
  end
  return 1  -- success

If payment later fails -> DECR to release usage slot.
Per-user: SETNX user_coupon:{user_id}:{coupon_id} with TTL.
```

---

## 5. APIs

```http
POST /api/v1/coupons/validate
{ "coupon_code": "SUMMER25", "cart": { "items": [...], "subtotal": 150.00 }, "user_id": "..." }
--> 200 { "valid": true, "discount": 37.50, "type": "percentage", "message": "25% off (max $50)" }
 OR 200 { "valid": false, "reason": "Coupon expired" }

POST /api/v1/coupons/apply  (at checkout confirmation)
{ "coupon_code": "SUMMER25", "order_id": "order-uuid" }
--> 200 { "applied": true, "discount": 37.50 }

POST /api/v1/admin/coupons  (create coupon)
{ "code": "SUMMER25", "type": "percentage", "value": 25, "conditions": {...} }

POST /api/v1/admin/coupons/bulk-generate
{ "campaign_id": "camp-uuid", "prefix": "SUMMER", "count": 100000, "template": {...} }
--> 202 { "batch_id": "batch-uuid", "status": "generating" }
```

---

## 6. Data Model

### PostgreSQL

```sql
CREATE TABLE coupons (
    coupon_id VARCHAR(50) PRIMARY KEY, campaign_id UUID,
    type ENUM('percentage','fixed','bogo','free_shipping','tiered'),
    value DECIMAL(10,2), discount_cap DECIMAL(10,2),
    conditions JSONB, stackable BOOLEAN DEFAULT FALSE, stack_group VARCHAR(20),
    max_uses_total INT, max_uses_per_user INT DEFAULT 1,
    current_uses INT DEFAULT 0,
    valid_from TIMESTAMPTZ, valid_to TIMESTAMPTZ,
    active BOOLEAN DEFAULT TRUE, created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE coupon_redemptions (
    redemption_id UUID PRIMARY KEY, coupon_id VARCHAR(50),
    user_id UUID, order_id UUID, discount_amount DECIMAL(10,2),
    redeemed_at TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_coupon (coupon_id), INDEX idx_user_coupon (user_id, coupon_id)
);
```

### Redis
```
coupon:{code}                    --> JSON (coupon definition), TTL 3600
coupon_usage:{coupon_id}         --> INT (atomic INCR/DECR)
user_coupon:{user_id}:{coupon_id} --> "1", TTL 86400
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Over-redemption** | Redis atomic INCR; rollback on payment failure |
| **Coupon cache stale** | TTL + invalidate on coupon update |
| **Bulk generation failure** | Batch insert; resume from last generated; idempotent |
| **Abuse (shared codes)** | Per-user limits; device fingerprinting; account age checks |
| **Redis/PG divergence** | Periodic reconciliation; PG is source of truth |

---

## 8. Deep Dive

### Personalized Coupons vs Universal Codes

```
Universal: "SAVE20" — anyone can use. Easy to share virally but hard to control.
Unique per-user: "USR-A3F7K2" — generated per user. Prevents sharing.
  Generate: prefix + random chars. Store: coupon_id -> user_id mapping.
  Validate: check coupon belongs to requesting user.

For campaigns: generate 100K unique codes -> distribute via email/SMS.
Each code is single-use + tied to recipient. Prevents coupon aggregator sites.
```

### Auto-Apply Promotions — No Code Needed

```
Site-wide: "10% off everything this weekend"
Category: "Free shipping on electronics"
Cart-value: "$15 off orders over $100"

These are applied automatically — user doesn't enter a code.

Implementation:
  1. On cart page load / checkout initiation:
     Fetch all active auto-apply promotions from Redis cache:
       auto_promos:{current_date} --> List of auto-apply coupon definitions
       TTL: 300 (refreshed from PostgreSQL)
  
  2. Evaluate each promotion against current cart:
     For each promo in auto_promos:
       if evaluate_conditions(promo.conditions, cart, user):
         applicable_promos.append(promo)
  
  3. Apply the BEST applicable promotion (or stack if policy allows):
     Sort by discount_amount DESC -> pick highest
     Or: apply all stackable auto-promos

  4. Show on cart page: "Promotion applied: $15 off orders over $100 ✓"
     User sees discount without doing anything.

Edge case: auto-promo + manual coupon
  Policy options:
  a. Manual coupon replaces auto-promo (user chose this coupon specifically)
  b. Both stack (more customer-friendly)
  c. Apply whichever gives bigger discount (best of both)
  Standard: option (c) — show user the better deal.
```

### Coupon Abuse Detection

```
Common abuse patterns:

1. Code sharing on Reddit/coupon sites:
   Detection: > 1000 unique users redeem same code within 1 hour
   Action: auto-disable coupon; alert marketing team
   Prevention: use unique per-user codes instead of universal

2. Multi-account abuse:
   Same person creates 5 accounts to use "first-purchase" coupon 5x
   Detection: same device fingerprint / IP / payment method across accounts
   Action: flag accounts; revoke coupon benefit from secondary accounts

3. Automated redemption bots:
   Script tries thousands of coupon code variations: "SAVE01", "SAVE02", ...
   Detection: > 10 invalid coupon attempts per minute from same IP/session
   Action: CAPTCHA + rate limit (max 5 coupon attempts per checkout session)
   Redis: INCR coupon_attempts:{session_id}, TTL 600, max 5

4. Returning after coupon:
   Buy with 50% off coupon -> return item -> rebuy at full price with credit
   Net effect: user effectively gets a permanent discount
   Detection: track return rate per coupon; if > 30% return rate, flag coupon

Fraud score:
  Combine signals -> fraud_score per redemption
  High score -> delay coupon validation; require manual approval for high-value coupons
```

### Coupon Analytics — Measuring Revenue Impact

```
Key metrics per coupon (ClickHouse queries):

1. Redemption rate = redemptions / unique_views_of_coupon_field
2. Revenue impact = total_order_value_with_coupon - total_discount_given
3. Incremental revenue = orders_with_coupon - estimated_orders_without_coupon
   (A/B test: show coupon to 50% of users, compare conversion rates)
4. Average order value with coupon vs without
5. Customer acquisition cost (for first-purchase coupons):
   CAC = total_discount / new_customers_acquired

ClickHouse table:
  coupon_events: { coupon_id, event_type (view/apply/redeem/expire),
                   user_id, order_value, discount_amount, timestamp }
  
  Dashboard queries:
  - "Which coupons drive the most revenue?" (order_value - discount)
  - "Which coupons have highest ROI?" (incremental_revenue / discount)
  - "What's the average time from coupon view to redemption?"
```

