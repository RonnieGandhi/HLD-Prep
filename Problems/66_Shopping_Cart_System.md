# 66. Design a Shopping Cart System

---

## 1. Functional Requirements (FR)

- **Add / remove items**: Add products with quantity; update quantity; remove items
- **Persist cart across sessions**: Logged-in user's cart survives app close, device switch
- **Guest cart**: Visitors can add items without login; merge cart on sign-in
- **Price & availability validation**: Re-validate prices and stock at checkout (cart may sit for days)
- **Saved for later**: Move items from cart to "Save for Later" wishlist
- **Cart expiry**: Auto-remove items after configurable TTL (e.g., 30 days)
- **Promotions**: Apply coupons, show discounted prices, bundle discounts
- **Multi-seller**: Items from different sellers in one cart; show per-seller subtotals
- **Shipping estimation**: Show estimated delivery date and shipping cost per item
- **Cart sharing**: Share cart via link (gift registries, wishlists)

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Cart operations (add/remove/update) in < 50 ms
- **High Availability**: 99.99% — cart failure = lost revenue
- **Consistency**: Cart state must be consistent across devices in real-time
- **Scale**: 100M+ DAU, 500M+ cart operations/day
- **Durability**: Cart data survives Redis restart (AOF persistence)
- **Session affinity not required**: Any API server can serve any cart (stateless)
- **Idempotent**: Duplicate "add to cart" requests don't double-add

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU with active carts | 50M |
| Cart operations / day | 500M |
| Cart operations / sec | ~6K (peak 50K during sales) |
| Avg items per cart | 5 |
| Cart data per user | ~2 KB |
| Total cart data | 50M × 2 KB = 100 GB |
| Guest carts / day | 20M (many abandoned) |
| Cart-to-order conversion | ~10% |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐
│  Web Client  │    │  Mobile App  │
└──────┬───────┘    └──────┬───────┘
       │                    │
       └─────────┬──────────┘
                 │
          ┌──────▼───────┐
          │  API Gateway │
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │  Cart        │
          │  Service     │
          │              │
          │  - CRUD ops  │
          │  - Validation│
          │  - Merge     │
          │  - Promo     │
          └──────┬───────┘
                 │
     ┌───────────┼───────────┐
     │           │           │
┌────▼────┐ ┌───▼────┐ ┌────▼────────┐
│  Redis  │ │ Product│ │ Inventory   │
│ (Cart   │ │ Catalog│ │ Service     │
│  Store) │ │ Service│ │ (Stock      │
│         │ │(prices)│ │  check)     │
└────┬────┘ └────────┘ └─────────────┘
     │
     │ (Async backup for durability)
┌────▼────┐
│PostgreSQL│
│(Cart     │
│ backup / │
│ analytics│
└──────────┘
```

### Component Deep Dive

#### Cart Storage: Why Redis

```
Cart access patterns:
  - Read cart: on every page (show cart icon count, cart page)
  - Write cart: add/remove/update items
  - TTL: carts expire after 30 days of inactivity
  - Size: ~2 KB per cart (5 items × ~400 bytes each)
  - Total: 50M active carts × 2 KB = 100 GB → fits in Redis Cluster

Redis data structure per cart:

Option 1: Hash (one hash per cart) ⭐
  Key: cart:{user_id}
  Field: sku_id
  Value: JSON { quantity, price_at_add, seller_id, added_at }
  
  HSET cart:user-123 SKU-456 '{"qty":2,"price":29.99,"seller":"s-1","added":"..."}'
  HGET cart:user-123 SKU-456
  HDEL cart:user-123 SKU-456
  HGETALL cart:user-123  → entire cart
  HLEN cart:user-123     → item count
  
  ✓ O(1) add/remove/update per item
  ✓ O(N) to fetch entire cart (N = items, typically 5-10)
  ✓ Natural per-item granularity

Option 2: Sorted Set (for ordering by add time)
  Key: cart:{user_id}
  Score: timestamp
  Member: JSON { sku_id, quantity, price }
  
  ✗ Harder to update specific item
  ✗ Members must be unique → can't have same JSON twice

Decision: Hash is the right structure for shopping carts.

TTL management:
  EXPIRE cart:user-123 2592000  (30 days)
  On every cart operation: refresh the TTL (touch)
  Inactive for 30 days → auto-deleted
```

#### Guest Cart → Login Merge

```
Scenario:
  1. Guest adds 3 items to cart (stored by session_id)
     cart:guest-session-abc → { SKU-1: qty 1, SKU-2: qty 2, SKU-3: qty 1 }
  
  2. Guest logs in as user-123
     Existing cart: cart:user-123 → { SKU-2: qty 1, SKU-4: qty 3 }
  
  3. Merge strategy:
     For items in BOTH carts (SKU-2):
       Option A: Keep higher quantity → qty = max(2, 1) = 2
       Option B: Sum quantities → qty = 2 + 1 = 3
       Option C: Keep guest cart quantity (most recent intent) → qty = 2
       
       Amazon uses Option A (max). Most user-friendly.
     
     For items only in guest cart: add to user cart
     For items only in user cart: keep
     
     Result: cart:user-123 → { SKU-1: qty 1, SKU-2: qty 2, SKU-3: qty 1, SKU-4: qty 3 }

  4. Delete guest cart: DEL cart:guest-session-abc

Implementation:
  def merge_carts(user_id, guest_session_id):
      guest_cart = redis.hgetall(f"cart:guest-{guest_session_id}")
      user_cart = redis.hgetall(f"cart:{user_id}")
      
      for sku_id, guest_item in guest_cart.items():
          if sku_id in user_cart:
              user_qty = user_cart[sku_id].quantity
              guest_qty = guest_item.quantity
              merged_qty = max(user_qty, guest_qty)
              redis.hset(f"cart:{user_id}", sku_id, {**guest_item, quantity: merged_qty})
          else:
              redis.hset(f"cart:{user_id}", sku_id, guest_item)
      
      redis.delete(f"cart:guest-{guest_session_id}")

Race condition: user logs in on two devices simultaneously
  Both trigger merge → double merge?
  Solution: Lua script in Redis (atomic merge):
    EVAL merge_script cart:user-123 cart:guest-abc
    Lua script runs atomically → no race condition
```

#### Price Validation at Checkout

```
Problem: User adds item at $29.99. Price changes to $34.99 while cart sits for 3 days.
  Should user pay $29.99 or $34.99?

Common approaches:

1. Always current price ⭐ (Amazon):
   Cart stores sku_id + quantity only (not price)
   On cart view: fetch current price from Product Catalog → display
   On checkout: use current price
   
   ✓ Simple, always accurate
   ✗ User surprised by price changes
   Mitigation: show "Price changed since added to cart" warning

2. Price at add time (locked price):
   Cart stores price snapshot at add time
   On checkout: honor that price
   
   ✗ Seller can't adjust prices (stuck at old price)
   ✗ Abuse: add items at sale price, buy weeks later at regular price
   
3. Hybrid ⭐⭐:
   Store price at add time in cart
   On checkout: compare with current price
   If current <= cart price → use current (customer wins)
   If current > cart price → show warning, use current
   If current > cart price × 1.1 → show prominent alert
   
   This is what most large retailers use.

Implementation:
  On cart page load:
    cart_items = redis.hgetall("cart:user-123")
    current_prices = catalog_service.batch_get_prices([sku_ids])
    
    for item in cart_items:
      item.current_price = current_prices[item.sku_id]
      item.price_changed = (item.current_price != item.price_at_add)
      item.in_stock = inventory_service.check_stock(item.sku_id)
    
    return cart with annotations
```

---

## 5. APIs

### Add to Cart
```http
POST /api/v1/cart/items
{
  "sku_id": "SKU-456",
  "quantity": 2,
  "seller_id": "seller-1"
}
Response: 200 OK
{
  "cart_id": "cart:user-123",
  "item_count": 5,
  "item": {"sku_id": "SKU-456", "quantity": 2, "price": 29.99, "seller_id": "seller-1"}
}
```

### Get Cart
```http
GET /api/v1/cart
Response: 200 OK
{
  "items": [
    {"sku_id": "SKU-456", "name": "Wireless Mouse", "quantity": 2,
     "price_at_add": 29.99, "current_price": 29.99, "price_changed": false,
     "in_stock": true, "seller": "TechStore", "image": "https://cdn..."}
  ],
  "subtotal": 59.98,
  "item_count": 5,
  "promotions_applied": [{"code": "SAVE10", "discount": -5.99}],
  "estimated_total": 53.99
}
```

### Update Item Quantity
```http
PUT /api/v1/cart/items/{sku_id}
{ "quantity": 3 }
Response: 200 OK
```

### Remove Item
```http
DELETE /api/v1/cart/items/{sku_id}
Response: 200 OK
```

### Merge Guest Cart
```http
POST /api/v1/cart/merge
{ "guest_session_id": "session-abc" }
Response: 200 OK
{ "merged_items": 3, "conflicts_resolved": 1 }
```

---

## 6. Data Model

### Redis — Primary Cart Store

```
# User cart (Hash)
cart:{user_id}  → Hash {
  SKU-456: '{"qty":2,"price_at_add":29.99,"seller_id":"s-1","added_at":"2026-03-14T10:00:00Z"}',
  SKU-789: '{"qty":1,"price_at_add":14.99,"seller_id":"s-2","added_at":"2026-03-14T10:05:00Z"}'
}
TTL: 2592000 (30 days, refreshed on activity)

# Guest cart
cart:guest-{session_id}  → Hash (same structure)
TTL: 86400 (1 day — shorter for guests)

# Saved for later
saved:{user_id}  → Hash { SKU-123: '{"saved_at":"...","price":29.99}' }
TTL: 7776000 (90 days)

# Cart item count (for badge display — avoid HLEN on every page)
cart_count:{user_id}  → INT
TTL: 2592000
```

### PostgreSQL — Backup + Analytics

```sql
CREATE TABLE cart_snapshots (
    user_id         UUID NOT NULL,
    sku_id          VARCHAR(50) NOT NULL,
    quantity        INT NOT NULL,
    price_at_add    DECIMAL(10,2),
    seller_id       VARCHAR(50),
    added_at        TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, sku_id)
);

-- Async sync from Redis → PostgreSQL every 5 minutes
-- Used for: analytics (abandonment analysis), Redis disaster recovery
```

### Kafka Topics

```
Topic: cart-events  (add, remove, update, checkout, abandon)
  → consumed by: analytics, recommendation engine ("frequently added together")
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Redis node failure** | Redis Cluster (6+ nodes); AOF persistence; data replicated across shards |
| **Redis total failure** | Fall back to PostgreSQL backup (slightly stale); rebuild Redis from PG |
| **Cart data loss** | Async backup to PostgreSQL every 5 min; client-side localStorage as last resort |
| **Duplicate add-to-cart** | Idempotent: HSET is naturally idempotent (same key → overwrite, not duplicate) |
| **Price discrepancy** | Always validate at checkout; show warnings on cart page |
| **Race condition (merge)** | Redis Lua script for atomic merge; no concurrent merge possible |
| **Abandoned cart recovery** | Kafka event on cart inactivity > 1 hour → trigger email reminder |

---

## 8. Deep Dive: Engineering Trade-offs

### Redis vs DynamoDB vs Session Storage for Cart

```
Redis ⭐ (recommended):
  ✓ Sub-millisecond latency
  ✓ Hash data structure is perfect for cart operations
  ✓ TTL for automatic expiry
  ✓ Lua scripting for atomic operations (merge, validate)
  ✗ Memory-only (need AOF/RDB for persistence)
  ✗ Cost: RAM is expensive at 100 GB scale ($$$)

DynamoDB:
  ✓ Durable (replicated, persistent)
  ✓ Pay-per-request pricing (good for variable load)
  ✓ Auto-scaling
  ✗ Higher latency (5-10 ms vs < 1 ms Redis)
  ✗ No TTL granularity per hash field (only per item)
  ✗ Item size limit 400 KB (fine for carts, but constraint)

Server-side session (e.g., PostgreSQL):
  ✓ Durable, ACID
  ✗ Much higher latency (10-50 ms)
  ✗ Cart reads are very frequent → DB bottleneck
  ✗ Not designed for key-value access patterns

Client-side (localStorage):
  ✓ Zero server cost for guest carts
  ✓ Works offline
  ✗ Not synced across devices
  ✗ Lost on browser clear
  ✗ No server-side analytics

Best practice:
  Guest carts: localStorage (client) + async backup to Redis on significant changes
  Logged-in carts: Redis (primary) + async backup to PostgreSQL
  This minimizes Redis memory usage (no guest cart storage) while keeping fast access
```

### Cart Abandonment: Why It Matters and How to Handle

```
Industry average: 70% of carts are abandoned (never checked out)

Detection:
  Cart has items + no checkout activity for 1 hour → "abandoned"
  
  Kafka event: { user_id, cart_items, total_value, abandoned_at }
  → Triggers:
    1. Email reminder (1 hour): "You left items in your cart"
    2. Push notification (4 hours): "Your cart is waiting"
    3. Email with discount (24 hours): "10% off your cart — use code COMEBACK10"
    4. Final reminder (72 hours): "Items may sell out soon"
  
  Suppress if:
    - User opted out of marketing
    - Cart value < $10 (not worth the email cost)
    - User has completed a purchase since abandonment

Analytics:
  Track: abandonment rate per step (cart → shipping → payment → confirm)
  Identify: at which step users drop off most
  A/B test: different cart UX → measure conversion rate
```



