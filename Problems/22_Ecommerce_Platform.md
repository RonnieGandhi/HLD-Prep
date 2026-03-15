# 22. Design an E-Commerce Platform (Amazon / Flipkart)

---

## 1. Functional Requirements (FR)

- **Product Catalog**: Browse, search, filter products with details, images, reviews
- **Shopping Cart**: Add/remove items, persist across sessions
- **Checkout & Payment**: Place orders with multiple payment methods
- **Order Management**: Track order status (placed → shipped → delivered → returned)
- **Inventory Management**: Real-time stock tracking, prevent overselling
- **Search**: Full-text search with filters (category, price range, rating, brand)
- **Recommendations**: "Customers who bought this also bought...", "Frequently bought together"
- **Reviews & Ratings**: Write/read product reviews
- **Seller Management**: Sellers list products, manage inventory, view sales dashboards
- **Notifications**: Order confirmation, shipping updates, delivery alerts

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99% — downtime = lost revenue
- **Low Latency**: Search results < 200 ms, page loads < 500 ms
- **Scalability**: 500M+ products, 100M+ DAU, 10M+ orders/day
- **Strong Consistency**: Inventory and payments MUST be strongly consistent
- **Eventual Consistency**: Product reviews, recommendations can be eventually consistent
- **Flash Sale Support**: Handle 100× traffic spikes during sales events (Prime Day, Black Friday)
- **Idempotent Payments**: No double-charging

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Products | 500M |
| DAU | 100M |
| Search QPS | 200K |
| Orders / day | 10M |
| Orders / sec | ~115 (peak 10K during flash sales) |
| Product page views / day | 5B |
| Cart operations / day | 500M |
| Avg order value | $50 |
| Daily GMV | $500M |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │
│ (Web/Mobile) │
└──────┬───────┘
       │
┌──────▼───────┐
│    CDN       │  ← Product images, static assets, cached pages
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │  ← Auth, rate limiting, routing
└──────┬───────┘
       │
       ├───────────┬───────────┬───────────┬───────────┬──────────┐
       │           │           │           │           │          │
┌──────▼──┐ ┌─────▼────┐ ┌────▼────┐ ┌────▼───┐ ┌────▼───┐ ┌───▼──────┐
│Product  │ │ Search   │ │ Cart   │ │ User   │ │Recommend│ │Notif     │
│Catalog  │ │ Service  │ │ Service│ │Service │ │ation   │ │Service   │
│Service  │ │          │ │        │ │        │ │Service │ │(email,   │
│         │ │          │ │        │ │        │ │(collab │ │ push,    │
│         │ │          │ │        │ │        │ │filter, │ │ SMS)     │
│         │ │          │ │        │ │        │ │content)│ │          │
└────┬────┘ └────┬─────┘ └───┬────┘ └────┬───┘ └───┬────┘ └──────────┘
     │           │           │           │         │
┌────▼────┐ ┌────▼────┐ ┌───▼───┐  ┌────▼────┐ ┌──▼──────┐
│MongoDB/ │ │Elastic  │ │ Redis │  │PostgreSQL│ │ ML      │
│DynamoDB │ │Search   │ │(cart  │  │(Users,   │ │Pipeline │
│(Catalog)│ │(full-   │ │ data, │  │ Addresses│ │(Spark)  │
│         │ │text +   │ │ TTL   │  │ Prefs)   │ │         │
│         │ │facets)  │ │ 30d)  │  └──────────┘ └─────────┘
└─────────┘ └─────────┘ └───────┘

┌────────────────────────────────────────────────────────────────────┐
│              CHECKOUT / ORDER FLOW (Saga Orchestrator)             │
│                                                                    │
│  Client places order → Order Service (orchestrator)                │
│                              │                                     │
│     Step 1                   │              Step 2                  │
│  ┌──────────────────┐        │        ┌──────────────────┐         │
│  │ Inventory Service│◄───────┼───────▶│ Payment Service  │         │
│  │ (reserve stock:  │        │        │ (authorize charge│         │
│  │  SELECT...FOR    │        │        │  via PSP:        │         │
│  │  UPDATE, atomic  │        │        │  Stripe/Adyen)   │         │
│  │  decrement)      │        │        │                  │         │
│  │  Compensate:     │        │        │  Compensate:     │         │
│  │   release stock  │        │        │   void auth      │         │
│  └────────┬─────────┘        │        └────────┬─────────┘         │
│           │                  │                 │                    │
│           │            ┌─────▼─────┐           │                   │
│           │            │ Order     │           │                   │
│           └───────────▶│ Service   │◄──────────┘                   │
│                        │ (PostgreSQL│                               │
│                        │  orders,   │     Step 3                    │
│                        │  state     │  ┌──────────────────┐        │
│                        │  machine)  │─▶│ Notification     │        │
│                        └─────┬──────┘  │ (order confirm,  │        │
│                              │         │  shipping update) │        │
│                              ▼         └──────────────────┘        │
│                        ┌────────────┐                              │
│                        │ Inventory  │                              │
│                        │ (confirm:  │                              │
│                        │  reserved→ │                              │
│                        │  committed)│                              │
│                        └────────────┘                              │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                  ASYNC EVENT PIPELINE                               │
│                                                                    │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────┐           │
│  │  Kafka   │────▶│ Consumers:   │────▶│ Downstream:  │           │
│  │(order-   │     │ • ES indexer │     │ • ClickHouse │           │
│  │ events,  │     │ • Analytics  │     │   (analytics)│           │
│  │ product- │     │ • Recommend  │     │ • Data Lake  │           │
│  │ events,  │     │   pipeline   │     │   (S3)       │           │
│  │ inventory│     │ • Fraud check│     │              │           │
│  │ -events) │     │ • Notify     │     │              │           │
│  └──────────┘     └──────────────┘     └──────────────┘           │
└────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Product Catalog Service
- **Why MongoDB/DynamoDB**: Products have highly variable schemas (clothing has size/color; electronics has specs; books have ISBN). Document DB handles schema flexibility well
- **Caching**: Hot products in Redis (product pages accessed 1000s of times per minute)
- **Product data**: title, description, images (CDN URLs), price, seller, attributes, category tree

#### Search Service (Elasticsearch)
- **Index**: Product title, description, brand, category, tags
- **Filters** (aggregations): category facets, price range, rating, availability, brand, seller
- **Ranking**: BM25 text relevance × sales velocity × rating × sponsored boost
- **Autocomplete**: Prefix matching on product names and brands

#### Cart Service
- **Logged-in users**: Cart stored in **Redis** (fast, persistent with AOF)
  - Key: `cart:{user_id}`, Value: Hash of `{item_id → {quantity, price, seller_id}}`
  - TTL: 30 days
- **Guest users**: Cart stored in browser localStorage; merged on login
- **Price consistency**: Cart stores the price at add-time; on checkout, re-validate current prices
- **Stock validation**: On checkout, verify items are still in stock

#### Inventory Service — Critical for Correctness

**Problem**: 100 users buy the last item simultaneously → overselling

**Solution**: Pessimistic locking with database transactions
```sql
BEGIN TRANSACTION;
  SELECT quantity FROM inventory WHERE product_id = ? AND seller_id = ? FOR UPDATE;
  -- Check: quantity >= requested_quantity
  UPDATE inventory SET quantity = quantity - ? WHERE product_id = ? AND seller_id = ?;
COMMIT;
```

**For flash sales (extreme concurrency)**:
- Pre-load stock into **Redis**: `DECR inventory:{product_id}`
- Redis DECR is atomic → no race conditions
- If DECR result < 0 → sold out (INCR to restore)
- Async: Kafka event to reconcile Redis with PostgreSQL

#### Order Service — Saga-Based Orchestration

Order placement involves multiple services:
```
1. Validate Cart       → Cart Service
2. Reserve Inventory   → Inventory Service (lock stock)
3. Calculate Total     → Pricing Service (apply coupons, tax, shipping)
4. Process Payment     → Payment Service (authorize)
5. Create Order        → Order Service (persist order)
6. Confirm Inventory   → Inventory Service (deduct stock)
7. Notify              → Notification Service (email, push)

If step 4 fails → Compensate: Release Inventory (step 2)
If step 5 fails → Compensate: Refund Payment + Release Inventory
```

**Saga Orchestrator** (Order Service is the orchestrator): coordinates steps, handles failures with compensating transactions.

#### Payment Service
- Integrates with payment gateways (Stripe, PayPal, RazorPay)
- **Idempotency**: Every payment attempt has a unique `idempotency_key` → retry-safe
- **Two-phase**: Authorize on order placement → Capture on shipment confirmation
- **Refunds**: Automated for cancellations; manual review for disputes

#### User Service
- Manages user accounts, addresses, preferences, payment methods
- **PostgreSQL** (relational data with ACID — address linked to user, payment methods linked to user)
- **APIs**: Register, login, manage addresses (add/edit/delete), manage saved payment methods
- On checkout: validate shipping address, select default payment method
- **Guest checkout**: Creates a temporary user record; prompts to create full account after order

#### Recommendation Service
- Powers "Customers who bought this also bought...", "Frequently bought together", "Based on your browsing"
- **Collaborative filtering** (batch Spark job): Mine co-purchase patterns from order history → store item-to-item associations
- **Content-based**: Similar products by attributes (brand, category, price range, product embedding similarity)
- **Session-based**: Real-time recommendations from current session click/view history (lightweight model, served inline)
- **Serving**: Pre-computed recommendations stored in Redis → served in < 50ms
- **ML Pipeline (Spark)**: Runs daily, processes order data and clickstream → updates recommendation models → materializes top-50 recommendations per product and per user into Redis

#### Notification Service
- Sends transactional notifications triggered by order events (via Kafka consumer)
- **Channels**: Email (SES/SendGrid), push notifications (FCM/APNs), SMS (Twilio)
- **Events handled**:
  - `order.placed` → order confirmation email + push
  - `order.shipped` → shipping notification with tracking link
  - `order.delivered` → delivery confirmation + review prompt
  - `order.cancelled` → cancellation + refund confirmation
  - `payment.failed` → retry prompt
- **Template engine**: Per-channel templates with dynamic variables (order details, tracking URL, customer name)
- **Preference management**: Users can opt in/out of marketing notifications; transactional notifications always sent

#### Async Event Pipeline (Kafka)
The backbone connecting all services asynchronously:

- **Topics**: `order-events`, `product-events`, `inventory-events`, `user-events`, `click-events`
- **Consumers**:
  - **ES Indexer**: Product creates/updates → update Elasticsearch index (< 2 second lag)
  - **Analytics Pipeline**: Order and click events → ClickHouse for business intelligence dashboards
  - **Recommendation Pipeline**: Order completion events → trigger Spark job to update co-purchase associations
  - **Fraud Detection**: Order events → ML model scores transaction risk → flag for review or auto-cancel
  - **Notification Service**: Order state transitions → trigger appropriate notification channel
  - **Data Lake (S3)**: All events archived for compliance, ML training, and historical analysis
- **Why Kafka over direct service-to-service calls?** Decouples services — if Notification Service is down, orders still process; notifications are delivered when service recovers (Kafka retains events)

---

## 5. APIs

### Search Products
```http
GET /api/v1/products/search?q=wireless+headphones&category=electronics&price_min=20&price_max=200&sort=relevance&page=1
```

### Get Product
```http
GET /api/v1/products/{product_id}
Response: 200 OK
{
  "product_id": "...",
  "title": "Sony WH-1000XM5",
  "description": "...",
  "price": 349.99,
  "discount_price": 279.99,
  "images": ["https://cdn.example.com/..."],
  "rating": 4.7,
  "review_count": 12543,
  "seller": {"id": "...", "name": "..."},
  "in_stock": true,
  "delivery_estimate": "Mar 15 - Mar 17",
  "attributes": {"color": "Black", "connectivity": "Bluetooth 5.2", ...}
}
```

### Cart Operations
```http
POST /api/v1/cart/items
{ "product_id": "...", "quantity": 1, "seller_id": "..." }

PUT /api/v1/cart/items/{item_id}
{ "quantity": 3 }

DELETE /api/v1/cart/items/{item_id}

GET /api/v1/cart
```

### Place Order
```http
POST /api/v1/orders
{
  "shipping_address_id": "addr-uuid",
  "payment_method_id": "pm-uuid",
  "promo_code": "SAVE10"
}
Response: 201 Created
{
  "order_id": "order-uuid",
  "status": "placed",
  "estimated_delivery": "2026-03-17",
  "total": 299.99
}
```

---

## 6. Data Model

### MongoDB — Product Catalog

```json
{
  "_id": "product-uuid",
  "title": "Sony WH-1000XM5 Headphones",
  "description": "Industry-leading noise canceling...",
  "category_path": ["Electronics", "Audio", "Headphones", "Over-Ear"],
  "brand": "Sony",
  "seller_id": "seller-uuid",
  "price": 349.99,
  "discount_price": 279.99,
  "currency": "USD",
  "images": ["url1", "url2"],
  "attributes": {
    "color": "Black",
    "connectivity": "Bluetooth 5.2",
    "battery_life": "30 hours",
    "weight": "250g"
  },
  "tags": ["noise-canceling", "wireless", "premium"],
  "rating": 4.7,
  "review_count": 12543,
  "sales_count": 50000,
  "is_active": true,
  "created_at": "2026-01-15T00:00:00Z"
}
```

### PostgreSQL — Orders (ACID required)

```sql
CREATE TABLE orders (
    order_id            UUID PRIMARY KEY,
    user_id             UUID NOT NULL,
    status              VARCHAR(20),
    shipping_address    JSONB,
    subtotal            DECIMAL(12,2),
    shipping_fee        DECIMAL(10,2),
    tax                 DECIMAL(10,2),
    discount            DECIMAL(10,2),
    total               DECIMAL(12,2),
    payment_method      VARCHAR(20),
    payment_status      VARCHAR(20),
    placed_at           TIMESTAMP,
    shipped_at          TIMESTAMP,
    delivered_at        TIMESTAMP,
    INDEX idx_user (user_id, placed_at DESC)
);

CREATE TABLE order_items (
    id              UUID PRIMARY KEY,
    order_id        UUID REFERENCES orders,
    product_id      UUID,
    seller_id       UUID,
    product_name    VARCHAR(256),
    quantity        INT,
    unit_price      DECIMAL(10,2),
    status          VARCHAR(20)   -- per-item status for multi-seller orders
);
```

### PostgreSQL — Inventory

```sql
CREATE TABLE inventory (
    product_id      UUID,
    seller_id       UUID,
    warehouse_id    UUID,
    quantity         INT NOT NULL CHECK (quantity >= 0),
    reserved        INT DEFAULT 0,     -- reserved during checkout
    updated_at      TIMESTAMP,
    PRIMARY KEY (product_id, seller_id, warehouse_id)
);
```

### Redis — Cart & Flash Sale Inventory

```
# Cart
Key:    cart:{user_id}
Value:  Hash { product_id → JSON{quantity, price, seller_id} }
TTL:    2592000 (30 days)

# Flash sale inventory (atomic operations)
Key:    flash:stock:{product_id}
Value:  integer (remaining stock)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Overselling** | Database-level `CHECK (quantity >= 0)` + Redis atomic DECR for flash sales |
| **Order saga failure** | Compensating transactions roll back in reverse order |
| **Payment failure** | Retry with idempotency key; if persistent → cancel order, release inventory |
| **Inventory sync (Redis ↔ DB)** | Kafka event on every inventory change; periodic reconciliation job |
| **Search index lag** | Acceptable for new products (appear within minutes) |
| **Flash sale stampede** | Redis handles stock check; rate limit orders per user; queue overflow to Kafka |

### Flash Sale Architecture
1. **Pre-warming**: Load limited stock (e.g., 1000 units) into Redis
2. **Rate limiting**: Max 1 purchase per user per product
3. **Queue-based**: Frontend shows "You're in the queue" → background processes orders sequentially
4. **Redis atomic**: `DECR flash:stock:{product_id}` → if result >= 0, proceed; else, sold out
5. **Async order processing**: Order details queued in Kafka → order workers process

---

## 8. Additional Considerations

### Multi-Seller / Marketplace Model
- One order can contain items from multiple sellers
- Each seller's items are shipped independently → order has multiple shipments
- Payment split: Platform takes 15-30% commission; rest goes to seller
- Seller dashboard shows their orders, revenue, inventory

### Warehouse & Fulfillment
- Multiple warehouses per region → route order to nearest warehouse with stock
- If product in multiple warehouses → ship from closest to customer
- Fulfillment service tracks: pick → pack → ship → in-transit → delivered

### Price and Promotion Engine
- Dynamic pricing: Competitor monitoring, demand-based pricing
- Coupon types: flat discount, percentage, free shipping, buy-one-get-one
- Stack rules: "Only one coupon per order" or "coupon + sale price = max discount of 30%"
- Promo validation on checkout: verify eligibility, expiry, usage limits

### Product Recommendations
- **Collaborative filtering**: "Bought together" associations mined from order data
- **Content-based**: Similar products by attributes (brand, category, price range)
- **Session-based**: "Based on your recent browsing" using session click history
- Pre-computed by Spark batch jobs → stored in Redis → served in < 50 ms

### Monitoring
- Conversion funnel: Browse → Add to Cart → Checkout → Payment → Delivery
- Cart abandonment rate (target: minimize)
- Inventory accuracy metrics
- Payment success rate
- Search zero-result queries (signals missing products)

---

## 9. Deep Dive: Engineering Trade-offs

### Why MongoDB/DynamoDB for Product Catalog (Not MySQL)?

```
Product data is inherently heterogeneous:
  - A T-shirt has: size, color, fabric, fit
  - A laptop has: RAM, CPU, screen_size, GPU, battery_life
  - A book has: ISBN, author, publisher, pages, language
  
MySQL approach:
  Option A: One giant table with 200+ nullable columns → sparse, ugly, slow
  Option B: EAV (Entity-Attribute-Value) pattern → terrible query performance, 
            no type safety, complex JOINs for every attribute
  Option C: JSON column → better, but indexing JSON is limited in MySQL
  
MongoDB ⭐ / DynamoDB:
  - Each product is a document with ONLY the attributes it needs
  - No schema migration when adding a new product category
  - Rich querying on nested attributes: db.products.find({"attributes.RAM": "16GB"})
  - Secondary indexes on frequently filtered attributes

  // T-shirt document
  { "product_id": "...", "title": "...", "category": "clothing",
    "attributes": { "size": ["S","M","L","XL"], "color": "Blue", "fabric": "Cotton" }}
  
  // Laptop document  
  { "product_id": "...", "title": "...", "category": "electronics",
    "attributes": { "RAM": "16GB", "CPU": "M3 Pro", "screen": "14-inch", "GPU": "10-core" }}

When MySQL IS needed:
  - Orders (ACID transactions, relational: order → items → payments)
  - Inventory (strict consistency, CHECK constraints)
  - User accounts (relational: user → addresses → payment methods)
  → Use MySQL/PostgreSQL for transactional data, MongoDB for catalog
```

### Inventory Management: The Hardest Problem in E-Commerce

```
The overselling problem:
  100 users click "Buy" simultaneously for the last item in stock
  Without proper handling → 100 orders created, 99 customers disappointed

Approach 1: Database Lock (Pessimistic)
  BEGIN; SELECT quantity FROM inventory WHERE product_id=? FOR UPDATE;
  -- check quantity >= 1
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id=?;
  COMMIT;
  
  ✓ Guaranteed correctness
  ✗ Lock contention: 100 concurrent requests serialize on one row
  ✗ DB connection pool exhaustion under high load
  ✗ Deadlocks possible with multi-item orders

Approach 2: Atomic Decrement (Optimistic) ⭐
  UPDATE inventory SET quantity = quantity - 1 
  WHERE product_id = ? AND quantity >= 1;
  -- If rows_affected = 1 → success
  -- If rows_affected = 0 → out of stock
  
  ✓ No explicit lock (DB handles atomically)
  ✓ Higher throughput than pessimistic
  ✓ No deadlocks
  ✗ Still hits DB for every request

Approach 3: Redis Pre-Decrement (Flash Sales) ⭐⭐
  Pre-load stock into Redis: SET stock:{product_id} 1000
  On purchase: result = DECR stock:{product_id}
  If result >= 0 → proceed with order (async write to DB)
  If result < 0 → INCR stock:{product_id}; return "sold out"
  
  ✓ Sub-millisecond response
  ✓ Handles 100K+ concurrent requests
  ✓ Perfect for flash sales / limited drops
  ✗ Redis and DB can diverge → need reconciliation
  ✗ If Redis crashes before DB write → sold items "reappear"
  
  Reconciliation: 
    Kafka event on every Redis DECR → async DB update
    Periodic job: compare Redis stock vs DB stock → fix discrepancies

Recommended: Approach 2 for normal operations. Approach 3 for flash sales.
```

### Saga Pattern vs 2PC for Distributed Transactions

```
Order placement spans multiple services:
  1. Inventory Service (reserve stock)
  2. Payment Service (charge customer)
  3. Order Service (create order)
  4. Notification Service (send confirmation)

Two-Phase Commit (2PC):
  Coordinator asks all participants: "Can you commit?"
  All say YES → Coordinator: "COMMIT"
  Any says NO → Coordinator: "ABORT"
  
  ✗ Blocking: if coordinator crashes after "prepare", ALL participants hang
  ✗ High latency (multiple round trips)
  ✗ Tight coupling between services
  ✗ Not practical for microservices across different databases

Saga Pattern ⭐ (Recommended):
  Each step has a COMPENSATING action for rollback:
  
  Step 1: Reserve Inventory      ← Compensate: Release Inventory
  Step 2: Authorize Payment      ← Compensate: Void Authorization  
  Step 3: Create Order           ← Compensate: Cancel Order
  Step 4: Send Notification      ← (no compensation needed)
  
  If Step 2 fails:
    → Execute Compensate Step 1 (release inventory)
    → Return error to user
  
  Orchestration (centralized coordinator) vs Choreography (event-driven):
  
  Orchestration ⭐ (recommended for order flow):
    Order Service is the orchestrator
    Calls each service sequentially, handles failures
    ✓ Clear flow, easy to debug, centralized error handling
    ✗ Orchestrator is a single point of coordination
  
  Choreography (event-driven):
    Each service publishes events, next service reacts
    ✓ Loosely coupled
    ✗ Hard to trace the full flow, "spaghetti events"
    ✗ Harder to implement compensation
```

### Search: Elasticsearch vs Database Full-Text Search

```
MySQL FULLTEXT index:
  ✓ Simple, no additional infrastructure
  ✗ Limited relevance scoring
  ✗ No faceted search (filter by brand AND price AND rating simultaneously)
  ✗ No fuzzy matching ("iphne" → "iPhone")
  ✗ No autocomplete suggestions
  ✗ Becomes slow with 500M+ products

Elasticsearch ⭐:
  ✓ BM25 relevance scoring (better than TFIDF)
  ✓ Faceted search: aggregations on brand, category, price range, rating
  ✓ Fuzzy matching, synonyms, stemming ("running shoes" matches "run shoe")
  ✓ Autocomplete via completion suggester
  ✓ Horizontally scalable (sharding across nodes)
  ✓ Near real-time indexing (< 1 second from DB write to searchable)
  ✗ Additional infrastructure cost and complexity
  ✗ Not a source of truth (must sync from primary DB)
  
  Sync pattern: 
    Product created/updated in MongoDB → publish to Kafka → 
    ES consumer reads event → indexes in Elasticsearch
    Lag: typically < 2 seconds
```

### Cart Design: Redis vs Database vs Client-Side

```
Client-Side (localStorage):
  ✓ Zero server load
  ✓ Works offline
  ✗ Cart lost on device switch or browser clear
  ✗ No cross-device sync
  ✗ Can be manipulated (price tampering)
  Best for: Guest users, lightweight apps

Database (PostgreSQL/MySQL):
  ✓ Persistent across sessions and devices
  ✓ Reliable, ACID
  ✗ DB hit on every cart operation (add/remove/update)
  ✗ Overkill for ephemeral data (most carts are abandoned)
  Best for: When cart data feeds into analytics or ML

Redis ⭐ (Recommended):
  ✓ Sub-millisecond operations
  ✓ Persistent with AOF (survives restart)
  ✓ TTL: auto-expire abandoned carts after 30 days
  ✓ Hash data structure: HSET cart:{user_id} {product_id} {quantity}
  ✗ If Redis loses data → cart is lost (acceptable: user re-adds)
  Best for: Logged-in users at scale

Hybrid (Amazon's approach):
  - Guest: localStorage → merged into Redis on login
  - Logged-in: Redis (primary) + periodic backup to DB
  - Checkout: Re-validate all prices and stock from source of truth
```

### Why Separate Read and Write Databases (CQRS)?

```
Without CQRS:
  One DB handles both product browsing (reads) and order processing (writes)
  Problem: Black Friday → orders spike → DB under write pressure → 
           product pages slow down → conversion drops

With CQRS (Command Query Responsibility Segregation):
  Write Model: PostgreSQL for orders, inventory (optimized for transactions)
  Read Model: Elasticsearch for search, Redis for product cache, 
              MongoDB read replicas for catalog browsing
  
  Sync: Write events → Kafka → update read models asynchronously
  
  ✓ Reads and writes scale independently
  ✓ Each model optimized for its access pattern
  ✓ Write DB under heavy load doesn't affect read performance
  ✗ Eventual consistency between write and read models
  ✗ More complex architecture
  
  The consistency gap:
    Product price updated → Kafka → Elasticsearch updated (1-2 second delay)
    During this gap, a user might see an old price
    Mitigation: On checkout, ALWAYS re-fetch price from write DB (source of truth)
```
