# 76. Design a Stock Exchange Matching Engine

---

## 1. Functional Requirements (FR)

- **Order types**: Limit orders, market orders, stop orders, stop-limit orders, IOC (Immediate-or-Cancel), FOK (Fill-or-Kill), GTC (Good-Till-Cancelled)
- **Order matching**: Match buy and sell orders by price-time priority (FIFO within same price)
- **Order book**: Maintain real-time order book per instrument (bids and asks sorted by price)
- **Trade execution**: Execute matched orders; generate trade records with unique trade IDs
- **Order lifecycle**: New, acknowledged, partially filled, filled, cancelled, expired, rejected
- **Market data**: Publish real-time L1 (BBO), L2 (depth), L3 (full order book), last trade, VWAP, volume
- **Order modification**: Amend quantity (reduce only) or price (loses time priority on price change)
- **Cancel**: Cancel resting orders; cancel-on-disconnect for algo traders
- **Opening/closing auctions**: Batch matching at market open and close (call auction)
- **Multiple instruments**: Support 10,000+ tradeable instruments (stocks, ETFs, options)
- **Pre-trade risk checks**: Position limits, order rate limits, fat-finger checks

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: Order-to-acknowledgement in < 10 microseconds (not milliseconds!)
- **Deterministic**: Same input sequence always produces same output (for regulatory replay)
- **Throughput**: 1M+ orders/sec across all instruments; 10K+/sec per hot instrument
- **Fairness**: Strict price-time priority (regulatory requirement — FIFO at each price level)
- **Durability**: Every order and trade persisted to WAL before acknowledgement for regulatory compliance
- **Availability**: 99.99% during market hours (9:30 AM – 4:00 PM ET)
- **Consistency**: No phantom fills, no double fills, no lost orders — deterministic execution

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Instruments | 10,000 |
| Orders / sec (peak) | 1M across all instruments |
| Orders per instrument / sec | 100 avg, 10K for hot stocks (AAPL, TSLA) |
| Trades / sec | 100K |
| Cancels / sec | 500K (algo traders cancel more than they trade) |
| Order book depth | ~1,000 price levels per side per instrument |
| Market data updates / sec | 5M (quotes, trades broadcast to all subscribers) |
| WAL write throughput | 1M events/sec x ~200 bytes = 200 MB/sec |
| Daily trade records | ~500M |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        ORDER ENTRY LAYER                                   │
│                                                                            │
│  Broker/Algo ──► FIX Gateway          Retail App ──► REST/WS Gateway      │
│                    │                                     │                  │
│                    └──────────────┬───────────────────────┘                  │
│                                  │                                          │
│                           +──────v──────+                                   │
│                           │   Order     │                                   │
│                           │   Router    │                                   │
│                           │  (validate  │                                   │
│                           │   format,   │                                   │
│                           │   auth,     │                                   │
│                           │   normalize │                                   │
│                           │   to internal│                                  │
│                           │   format)   │                                   │
│                           +──────+──────+                                   │
│                                  │                                          │
│                           +──────v──────+                                   │
│                           │  Pre-Trade  │                                   │
│                           │  Risk       │                                   │
│                           │  Engine     │                                   │
│                           │             │                                   │
│                           │  - Fat-finger│ (qty > 100K or price > 2x last) │
│                           │  - Position  │ (net exposure < limit per firm) │
│                           │    limits    │                                  │
│                           │  - Order rate│ (< 5K orders/sec per firm)      │
│                           │    throttle  │                                  │
│                           │  - Self-trade│ (prevent same firm buy+sell)    │
│                           │    prevention│                                  │
│                           │  - Kill      │                                  │
│                           │    switch    │ (emergency halt per firm)        │
│                           +──────+──────+                                   │
│                                  │                                          │
│                           REJECT │ (if risk check fails → immediate NAK)   │
│                                  │                                          │
└──────────────────────────────────│──────────────────────────────────────────┘
                                   │
┌──────────────────────────────────│──────────────────────────────────────────┐
│                    CORE MATCHING LAYER                                      │
│                                  │                                          │
│                           +──────v──────+                                   │
│                           │  Sequencer  │ (single-threaded per partition)   │
│                           │             │                                   │
│                           │  Assign     │                                   │
│                           │  seq_num    │                                   │
│                           │  (monotonic │                                   │
│                           │   counter)  │                                   │
│                           │             │                                   │
│                           │  Write to   │                                   │
│                           │  WAL (NVMe) │                                   │
│                           │  + replicate│                                   │
│                           │  to standby │                                   │
│                           +──────+──────+                                   │
│                                  │                                          │
│            ┌─────────────────────┼─────────────────────┐                    │
│            │                     │                     │                    │
│     +──────v──────+       +──────v──────+       +──────v──────+            │
│     │  Matching   │       │  Matching   │       │  Matching   │            │
│     │  Engine     │       │  Engine     │       │  Engine     │            │
│     │  Partition 1│       │  Partition 2│       │  Partition N│            │
│     │             │       │             │       │             │            │
│     │  AAPL, MSFT │       │  GOOGL,AMZN │       │  SPY, QQQ  │            │
│     │  TSLA, META │       │  NVDA, NFLX │       │  IWM, ...  │            │
│     │             │       │             │       │             │            │
│     │  Single-    │       │  Single-    │       │  Single-    │            │
│     │  threaded   │       │  threaded   │       │  threaded   │            │
│     │  per        │       │  per        │       │  per        │            │
│     │  instrument │       │  instrument │       │  instrument │            │
│     │             │       │             │       │             │            │
│     │  Ring Buffer│       │  Ring Buffer│       │  Ring Buffer│            │
│     │  (LMAX      │       │  (LMAX      │       │   Disruptor)│            │
│     │   Disruptor)│       │   Disruptor)│       │   Disruptor)│            │
│     +──────+──────+       +──────+──────+       +──────+──────+            │
│            │                     │                     │                    │
│            └─────────────────────┼─────────────────────┘                    │
│                                  │                                          │
│                    Outputs per matched event:                               │
│                    1. Execution Report (ACK/fill/cancel confirm)            │
│                    2. Trade Record (buyer, seller, price, qty)              │
│                    3. Order Book Delta (price level change)                 │
│                                  │                                          │
└──────────────────────────────────│──────────────────────────────────────────┘
                                   │
┌──────────────────────────────────│──────────────────────────────────────────┐
│                    OUTPUT / DOWNSTREAM LAYER                                │
│                                  │                                          │
│         ┌────────────────────────┼────────────────────┐                     │
│         │                        │                    │                     │
│  +──────v──────+          +──────v──────+      +──────v──────+             │
│  │  Execution  │          │  Market     │      │  Trade      │             │
│  │  Report     │          │  Data       │      │  Store      │             │
│  │  Publisher  │          │  Publisher  │      │  (Write-    │             │
│  │             │          │             │      │   Behind)   │             │
│  │  FIX/WS    │          │  Multicast  │      │             │             │
│  │  back to   │          │  UDP to all │      │  Batch WAL  │             │
│  │  client    │          │  subscribers│      │  entries to │             │
│  │  (< 1μs)   │          │             │      │  PostgreSQL │             │
│  +─────────────+          │  L1: BBO    │      │  every 1s   │             │
│                           │  L2: 10-deep│      │             │             │
│                           │  L3: Full   │      │  Async:     │             │
│                           │    book     │      │  non-critical│            │
│                           │  Trades tape│      │  path       │             │
│                           +──────+──────+      +──────+──────+             │
│                                  │                    │                     │
│                           +──────v──────+      +──────v──────+             │
│                           │  Kafka      │      │  PostgreSQL │             │
│                           │  (market-   │      │  (orders,   │             │
│                           │   data for  │      │   trades,   │             │
│                           │   analytics,│      │   audit log)│             │
│                           │   compliance│      │             │             │
│                           │   tape)     │      │             │             │
│                           +─────────────+      +─────────────+             │
│                                                                            │
│  +─────────────+          +─────────────+      +─────────────+             │
│  │  Clearing   │          │  Surveillance│     │  Drop Copy  │             │
│  │  Service    │          │  Engine     │      │  Service    │             │
│  │  (net       │          │  (detect    │      │  (real-time │             │
│  │   positions,│          │   wash      │      │   trade copy│             │
│  │   margin    │          │   trading,  │      │   to brokers│             │
│  │   calc,     │          │   spoofing, │      │   for their │             │
│  │   settlement│          │   layering) │      │   records)  │             │
│  │   T+1)      │          │             │      │             │             │
│  +─────────────+          +─────────────+      +─────────────+             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Complete Order-to-Trade Flow — Step-by-Step Sequence

```
Broker           FIX GW         Order Router     Risk Engine      Sequencer        Matching Engine    Exec Reporter    Mkt Data Pub
  │                │                │                │                │                │                │                │
  │── New Order ──▶│                │                │                │                │                │                │
  │  (FIX MsgType  │                │                │                │                │                │                │
  │   =D, AAPL,    │                │                │                │                │                │                │
  │   Buy 100@     │                │                │                │                │                │                │
  │   $150.10)     │                │                │                │                │                │                │
  │                │── parse FIX ──▶│                │                │                │                │                │
  │                │   normalize    │                │                │                │                │                │
  │                │   to internal  │                │                │                │                │                │
  │                │   format       │── validate ───▶│                │                │                │                │
  │                │                │   auth, format │                │                │                │                │
  │                │                │                │── risk checks: │                │                │                │
  │                │                │                │   kill switch? │                │                │                │
  │                │                │                │   fat-finger?  │                │                │                │
  │                │                │                │   position ok? │                │                │                │
  │                │                │                │   rate limit?  │                │                │                │
  │                │                │                │   STP preview? │                │                │                │
  │                │                │                │   (~500ns)     │                │                │                │
  │                │                │                │                │                │                │                │
  │                │                │  ◀── PASS ─────│                │                │                │                │
  │                │                │                │                │                │                │                │
  │                │                │────────────────────────────────▶│                │                │                │
  │                │                │                │                │── assign       │                │                │
  │                │                │                │                │   seq_num=4523 │                │                │
  │                │                │                │                │── write WAL    │                │                │
  │                │                │                │                │   (NVMe ~1μs)  │                │                │
  │                │                │                │                │── replicate    │                │                │
  │                │                │                │                │   to standby   │                │                │
  │                │                │                │                │── publish to ─▶│                │                │
  │                │                │                │                │   ring buffer  │── lookup AAPL  │                │
  │                │                │                │                │                │   order book   │                │
  │                │                │                │                │                │── best ask =   │                │
  │                │                │                │                │                │   $150.10 (100)│                │
  │                │                │                │                │                │── MATCH! fill  │                │
  │                │                │                │                │                │   100@$150.10  │                │
  │                │                │                │                │                │── update book  │                │
  │                │                │                │                │                │── check stops  │                │
  │                │                │                │                │                │── check LULD   │                │
  │                │                │                │                │                │                │                │
  │                │                │                │                │                │── exec report ▶│                │
  │                │                │                │                │                │   (fill 100@   │── FIX 8       │
  │                │                │                │                │                │    $150.10)    │   ExecType=F  │
  │◀───────────────│◀───────────────│────────────────│────────────────│────────────────│◀───────────────│   to buyer    │
  │  Fill confirm  │                │                │                │                │                │   + seller    │
  │  (< 10μs       │                │                │                │                │                │                │
  │   end-to-end)  │                │                │                │── trade rec ──▶│   (async)      │                │
  │                │                │                │                │                │── book delta ──│───────────────▶│
  │                │                │                │                │                │                │   multicast   │
  │                │                │                │                │                │                │   UDP to all  │
  │                │                │                │                │                │                │   subscribers │
  │                │                │  ◀── position update ◀─────────│                │                │                │
  │                │                │                │  (async: +100  │                │                │                │
  │                │                │                │   AAPL for     │                │                │                │
  │                │                │                │   buying firm) │                │                │                │
```

**Critical path latency budget** (total < 10μs):
| Step | Latency |
|---|---|
| FIX parse + normalize | ~1-2 μs |
| Risk checks (shared memory) | ~0.5 μs |
| Sequencer (assign + NVMe WAL write) | ~1-2 μs |
| Ring buffer publish + consume | ~0.1 μs |
| Matching engine (lookup + match) | ~0.5-1 μs |
| Execution report publish | ~0.5-1 μs |
| FIX response serialize + send | ~1-2 μs |
| **Total** | **~5-9 μs** |

### Component Deep Dive

#### Matching Algorithm — Price-Time Priority

```
Order Book for stock AAPL:

BUY (Bids)                          SELL (Asks)
Price    Qty    Time                 Price    Qty    Time
$150.10  100    09:30:01.001        $150.15  200    09:30:01.002
$150.10  50     09:30:01.005        $150.20  300    09:30:01.003
$150.05  200    09:30:01.003        $150.25  100    09:30:01.001
$150.00  500    09:30:00.999

New SELL limit order arrives: sell 120 shares at $150.10

Matching process:
1. Best bid = $150.10. Sell price ($150.10) <= best bid ($150.10) -> MATCH!
2. First bid at $150.10: 100 shares (time: 09:30:01.001) -> fill 100 shares
   Trade: 100 shares @ $150.10
   Remaining sell quantity: 120 - 100 = 20
3. Second bid at $150.10: 50 shares (time: 09:30:01.005) -> fill 20 shares
   Trade: 20 shares @ $150.10
   Remaining sell quantity: 0. Order fully filled.
   Second bid partially filled: 50 - 20 = 30 remaining.

Updated Order Book:
BUY (Bids)                          SELL (Asks)
$150.10  30     09:30:01.005        $150.15  200    09:30:01.002
$150.05  200    09:30:01.003        $150.20  300    09:30:01.003
$150.00  500    09:30:00.999        $150.25  100    09:30:01.001

Data structure for order book:
  Bids: sorted map (price DESC -> doubly-linked list of orders FIFO)
  Asks: sorted map (price ASC -> doubly-linked list of orders FIFO)
  orders: HashMap<OrderId, pointer-to-node-in-linked-list>
  
  Lookup best bid/ask: O(1) with cached pointer to top of tree
  Insert order at existing price: O(1) append to linked list
  Insert order at new price: O(log P) where P = distinct price levels
  Match: O(1) amortized (consume from head of linked list at top price)
  Cancel by order_id: O(1) — HashMap lookup + doubly-linked-list remove
  
  Why linked list (not deque/array)?
    Cancel is the most frequent operation (algos cancel 90% of orders).
    Doubly-linked list: O(1) removal by pointer. Array: O(n) shift.
    At 500K cancels/sec, this difference is critical.

  Price representation: integer ticks (not floating point).
    $150.10 = 15010 (in cents). Integer math: no floating-point rounding errors.
    All exchanges use integer price internally. Display layer converts back.
```

#### Sequencer — Total Ordering of Events

```
Why total ordering? Fairness. The order that arrives first MUST be processed first.

Problem: Multiple gateway servers receive orders concurrently.
Without total ordering, two orders arriving "simultaneously" could be processed
in different orders depending on which gateway → race condition → unfair.

Sequencer implementation:
  All gateways ──► Sequencer (single thread, single machine per partition)
  
  For each incoming order:
    1. Assign seq_num = atomic_counter++ (monotonically increasing, no gaps)
    2. Write { seq_num, order_data } to WAL on local NVMe SSD (~1μs write)
    3. Replicate WAL entry to hot standby (async, < 10μs)
    4. Publish to ring buffer for matching engine consumption
  
  Throughput: ~10M seq assignments/sec (counter increment + NVMe write)
  Latency: ~1-2 μs per assignment (dominated by NVMe fsync)
  
  Why NVMe and not regular SSD?
    Regular SSD fsync: 50-200 μs. NVMe: 5-20 μs.
    At exchange latencies, the storage write IS the bottleneck.
    Some exchanges use battery-backed RAM for 0.5 μs writes.

Partitioning:
  Not all 10,000 instruments share one sequencer — bottleneck.
  Partition instruments into N groups (e.g., 64 partitions).
  Each partition has its own sequencer + matching engine threads.
  Partition assignment: hash(instrument_id) % 64
  Within a partition: total ordering guaranteed.
  Cross-partition: no ordering guarantee needed (different instruments are independent).

Fault tolerance:
  Primary sequencer + hot standby (same machine rack, different power supply)
  WAL replicated synchronously (or within 10μs async with bounded loss)
  On primary failure: standby takes over, continues from last seq_num
  Gap detection: if client sees seq_num jump, it replays from WAL
```

#### Why Single-Threaded Matching (Not Multi-Threaded)

```
Counter-intuitive: the most performance-critical financial system runs SINGLE-THREADED.

Why?
  1. No locking overhead:
     Concurrent access to order book requires locks or CAS operations.
     Lock contention at 10K orders/sec for one stock = massive overhead.
     Single-threaded: zero lock overhead, deterministic execution.
     Measured: single-threaded matching is 3-10x faster than lock-based concurrent.
  
  2. Deterministic replay:
     Same input sequence → same output, always.
     Multi-threaded: order of execution is non-deterministic → can't replay.
     Regulatory requirement: must be able to replay any trading day exactly.
     SEC Rule 613 (Consolidated Audit Trail): exchanges must reconstruct events.
  
  3. CPU cache efficiency:
     Hot order book data fits in L1/L2 cache (~256 KB for top 100 price levels).
     Single thread: no cache invalidation from other threads → max cache hit rate.
     Measured: 99%+ L1 cache hit rate for matching operations.
  
  4. Good enough throughput:
     Single core processes 500K+ operations/sec per instrument.
     No single instrument has > 10K orders/sec → single thread is 50x sufficient.
  
  Per-instrument parallelism:
    Each instrument has its own matching engine thread on its own core.
    Core pinning: Thread for AAPL always runs on core 4 (no context switches).
    10,000 instruments across 64 partitions, each partition on dedicated hardware.
    NUMA-aware: each partition's memory allocated on same NUMA node as its core.
```

#### Pre-Trade Risk Engine — Sub-Microsecond Checks

```
Every order passes through risk checks BEFORE reaching the sequencer.
If any check fails → immediate reject (never touches order book).

Check 1: Fat-Finger Detection
  if order.price > 2x last_traded_price → REJECT
  if order.quantity > firm_max_single_order (e.g., 100,000 shares) → REJECT
  if order.notional_value > $10M → REJECT (require manual approval)

Check 2: Position Limit
  Firm's net position in AAPL: +50,000 shares. Limit: 100,000.
  New buy order for 60,000 would bring position to 110,000 > limit → REJECT
  
  Implementation: maintain per-firm per-instrument position counter in shared memory.
  Updated on every fill. Checked on every new order. < 100 ns lookup.

Check 3: Order Rate Throttle
  Max 5,000 orders/sec per firm per instrument (prevent algo runaway).
  Token bucket per (firm_id, instrument_id). Refill rate: 5,000/sec.
  If bucket empty → THROTTLE (reject with "rate limit exceeded")

Check 4: Self-Trade Prevention (STP)
  Firm A has resting buy at $150.10. Firm A sends sell at $150.10.
  Without STP: Firm A trades with itself (wash trading — illegal).
  STP modes:
    Cancel newest: cancel the incoming order
    Cancel oldest: cancel the resting order
    Cancel both: cancel both
    Decrement and Cancel: reduce both quantities by the overlap; cancel remainder
  Configured per-firm.

Check 5: Kill Switch
  Ops can instantly disable a firm: all orders rejected, all resting orders cancelled.
  Triggered: algo malfunction, suspected fraud, margin breach.
  Implementation: per-firm boolean flag in shared memory. Checked first (fastest).

All checks combined: < 500 ns total (shared memory lookups, no network).
```

#### Opening and Closing Auctions

```
Regular trading (continuous matching): orders matched immediately as they arrive.
Auctions (call matching): orders collected, then matched at a single clearing price.

Opening Auction (9:00 AM – 9:30 AM):
  Phase 1 (Pre-open, 9:00-9:28): accept orders, display indicative price
  Phase 2 (Random close, 9:28-9:30): end time randomized within 2-min window
    Why random? Prevent last-second order stuffing (gaming the auction).
  Phase 3 (Match): find price that maximizes volume
  
  Clearing price algorithm:
    For each possible price P:
      buy_volume(P) = SUM(qty) of all buy orders with price >= P
      sell_volume(P) = SUM(qty) of all sell orders with price <= P
      matched_volume(P) = MIN(buy_volume, sell_volume)
    
    Clearing price = P that maximizes matched_volume(P)
    Tie-breaking: minimize imbalance (|buy_volume - sell_volume|)
    Second tie-break: minimize distance from reference price (yesterday's close)
    
    All matched orders execute at the SINGLE clearing price.
    Example: 500K shares matched at $150.05 → opening price = $150.05

Closing Auction (3:50 PM – 4:00 PM):
  Same mechanism. Sets the official closing price (used for index calculations, ETF NAV).
  Closing auction typically handles 10-20% of daily volume (institutional rebalancing).

Why auctions?
  1. Price discovery: single price reflects aggregate supply/demand (fairer than continuous)
  2. Reduced volatility: no price swings from sequential individual orders
  3. Liquidity concentration: everyone trades at one price → better fills for large orders
```

#### Stop Order Triggering — Event-Driven

```
Stop orders are NOT in the order book. They are conditional orders stored separately.

Stop Buy at $155: "When market price reaches $155, submit a market buy order"
Stop-Limit Sell at $145 limit $144: "When price drops to $145, submit limit sell at $144"

Implementation:
  Stop order store: HashMap<InstrumentId, TreeMap<trigger_price, List<StopOrder>>>
  
  After EVERY trade:
    trade_price = $155.00
    
    Check stop buys: any stop buy with trigger_price <= 155.00?
      Yes → convert to market/limit order → inject into matching engine
    
    Check stop sells: any stop sell with trigger_price >= 155.00?
      (This would be unusual; stop sells trigger when price DROPS)
      For stop sells: check if trade_price <= trigger_price
    
  Edge case: cascading triggers
    Stop sells at $145 trigger → new sells hit book → price drops to $140
    → more stop sells at $140 trigger → cascading "stop loss avalanche"
    
    Solution: LULD circuit breaker halts the instrument if price moves > 5% in 5 min.
    This breaks the cascade and allows human intervention.
```

#### IOC, FOK, and GTC — Time-in-Force Handling

```
Time-in-Force determines what happens to the UNFILLED portion of an order:

DAY (default):
  Order rests in book until end of trading day.
  At 4:00 PM ET → all DAY orders automatically cancelled.
  Implementation: end-of-day sweep iterates all order books, removes DAY orders.

GTC (Good-Till-Cancelled):
  Order persists across trading days until explicitly cancelled.
  Survives overnight. Re-loaded on next market open from persistent store.
  Risk: stale GTC orders from weeks ago can match at unexpected prices.
  Some exchanges cap GTC at 90 days to prevent this.

IOC (Immediate-or-Cancel):
  Execute whatever quantity matches IMMEDIATELY, cancel the rest.
  Never rests in the book. Zero book impact if no match.
  
  Example: IOC Buy 500 shares at $150.10
    200 shares available at $150.10 → fill 200, cancel remaining 300.
    Execution report: Partial Fill 200, then Cancelled 300.
  
  Used by: algo traders who want to "take" liquidity without showing intent.

FOK (Fill-or-Kill):
  ENTIRE order must fill immediately, or ENTIRE order is cancelled.
  All-or-nothing. Even more restrictive than IOC.
  
  Example: FOK Buy 500 shares at $150.10
    Only 200 shares available → REJECT (cancel entire order, fill nothing).
    500 shares available → fill all 500.
  
  Implementation: before matching, check if total available qty at eligible 
  prices >= order.qty. If not → reject without touching book.
  Used by: institutional block traders who need exact fill sizes.

  Matching engine implementation:
    match(order):
      if order.tif == IOC:
        result = match_aggressive(order)  // match what's available
        if order.remaining_qty > 0:
          cancel(order, reason="IOC_UNFILLED")
      
      elif order.tif == FOK:
        available = calculate_fillable_qty(order)
        if available < order.qty:
          reject(order, reason="FOK_INSUFFICIENT_LIQUIDITY")
        else:
          match_aggressive(order)  // guaranteed full fill
      
      elif order.tif == DAY or order.tif == GTC:
        result = match_aggressive(order)
        if order.remaining_qty > 0:
          insert_into_book(order)  // rest unfilled portion
```

#### Cancel-on-Disconnect (CoD) — Algo Safety Mechanism

```
Problem: An algo trading firm's network connection drops.
Their 10,000 resting orders are still in the book — but the algo can't manage them.
If the market moves, those stale orders will fill at disadvantageous prices.

Solution: Cancel-on-Disconnect (CoD)
  When a FIX session disconnects (heartbeat timeout, TCP RST, etc.):
    1. Gateway detects session loss (FIX heartbeat missed for 3 consecutive intervals)
    2. Gateway sends mass cancel for ALL orders associated with that session
    3. Matching engine cancels all resting orders for that firm + session_id
    4. Execution reports (cancel confirms) are queued for when session reconnects
  
  Implementation:
    Session state: HashMap<SessionId, Set<OrderId>>
    On disconnect: for each order_id in session_orders:
      send_cancel(order_id, reason="CANCEL_ON_DISCONNECT")
    
    Time to cancel all orders: ~1ms for 10,000 orders (10 cancels × 100ns each)
  
  Configuration per firm:
    CoD enabled: yes/no (some firms prefer orders to persist through brief disconnects)
    Grace period: 0-30 seconds (time to reconnect before mass cancel triggers)
    Scope: all orders, or only orders entered in current session
  
  Edge case: firm reconnects DURING mass cancel
    Some orders already cancelled, some not yet processed.
    Solution: firm must reconcile state on reconnect by requesting full order status.
    Matching engine provides "mass order status" query for this purpose.
```

#### Surveillance Engine — Market Abuse Detection

```
Consumes ALL order and trade events from Kafka (audit-trail topic).
Runs asynchronous pattern detection — NOT in the matching critical path.

Pattern 1: Spoofing / Layering Detection
  Spoofer places large visible orders to fake demand, then cancels before execution.
  
  Detection algorithm:
    For each firm, per instrument, in rolling 10-second windows:
      large_orders = orders with qty > 10x average order size
      cancel_rate = cancelled_large_orders / total_large_orders
      cancel_speed = avg time between place and cancel of large orders
      
      if cancel_rate > 90% AND cancel_speed < 1 second AND 
         price_moved_in_favor(firm's executions on opposite side):
        → ALERT: suspected spoofing
  
  Example: Firm places 50,000 share buy order (visible), price ticks up, 
  firm sells 5,000 shares at higher price, then cancels the 50,000 buy.
  Pattern: inflate price → profit from opposite side → cancel inflating order.

Pattern 2: Wash Trading Detection
  Same entity trading with itself to create illusion of volume.
  Even with STP enabled, can happen across sub-accounts or affiliated firms.
  
  Detection: graph analysis of trade counterparties.
    Build graph: nodes = firm sub-accounts, edges = trades between them.
    If a cluster of accounts repeatedly trade with each other and are 
    controlled by same beneficial owner → ALERT.
  
  Also: statistical detection — if two accounts always appear as buyer/seller 
  of the same trades at suspiciously correlated times.

Pattern 3: Momentum Ignition
  Rapid submission of orders designed to trigger other algo's stop orders or 
  momentum strategies, then profiting from the resulting price move.
  
  Detection: unusual order submission rate from one firm immediately preceding 
  a rapid price move, followed by that firm's executions on the opposite side.

Pattern 4: Quote Stuffing
  Submitting and cancelling thousands of orders per second to slow down competitors.
  
  Detection: order-to-trade ratio > 100:1 per firm per instrument per minute.
  Some exchanges charge fees for excessive cancellation rates to discourage this.

Technology: Apache Flink or Spark Streaming consuming from Kafka.
  Complex event processing (CEP) rules engine for real-time alerting.
  Historical analysis: batch Spark jobs on full day's data for deeper patterns.
  Alerts → compliance team dashboard + regulatory reporting (FINRA, SEC).
```

#### Drop Copy Service — Real-Time Trade Reporting

```
Every broker connected to the exchange needs a real-time copy of their trades 
and order acknowledgements — separate from the primary execution report feed.

Why separate from execution reports?
  1. Execution reports go to the TRADING session (the algo or trader).
  2. Drop copy goes to the COMPLIANCE/RISK session (back office, middle office).
  3. If the trading session disconnects, drop copy still delivers to compliance.
  4. Multiple compliance systems can each receive their own drop copy.

Implementation:
  Matching engine outputs → Kafka "trade-reports" topic (keyed by firm_id)
  Drop Copy Service: per-firm Kafka consumer → FIX drop copy session to firm's 
  back-office system.
  
  Guaranteed delivery: Kafka offset tracking ensures no trade is missed.
  If firm's drop copy connection is down → messages buffered in Kafka → 
  delivered on reconnect (replay from last committed offset).

Content per drop copy message:
  - Trade ID, instrument, side, price, quantity, timestamp
  - Order ID, original order details
  - Counterparty firm ID (anonymized — identified as "Market" per exchange rules)
  - Sequence number for gap detection

Regulatory requirement: exchanges MUST provide drop copy capability.
  FINRA Rule 7440: firms must have real-time trade reporting capability.
```

---

## 5. APIs

### FIX Protocol (Institutional / Algo)
```
FIX 4.4 / 5.0 — Binary protocol for sub-millisecond communication.
  New Order: MsgType=D, ClOrdID, Symbol, Side, OrdType, Price, OrderQty, TimeInForce
  Cancel: MsgType=F, OrigClOrdID
  Amend: MsgType=G, OrigClOrdID, Price (optional), OrderQty (optional)
  
  Execution Report (response): MsgType=8
    ExecType=0 (New), ExecType=F (Fill), ExecType=4 (Cancelled), ExecType=8 (Rejected)
    
  Market Data: MsgType=W (snapshot), MsgType=X (incremental)

FIX is the industry standard. All brokers and exchanges speak FIX.
Binary FIX (SBE encoding): ~10x faster than text FIX.
```

### REST/WebSocket (Retail)
```http
POST /api/v1/orders
{
  "instrument": "AAPL", "side": "buy", "type": "limit",
  "price": 150.10, "quantity": 100, "time_in_force": "day"
}
→ 201 { "order_id": "ord-uuid", "status": "new", "seq_num": 4523781, "timestamp": "..." }

POST /api/v1/orders
{
  "instrument": "AAPL", "side": "sell", "type": "stop_limit",
  "stop_price": 145.00, "limit_price": 144.00, "quantity": 100, "time_in_force": "gtc"
}
→ 201 { "order_id": "ord-uuid", "status": "new", "trigger_type": "stop" }

PUT /api/v1/orders/{order_id}
{ "quantity": 50 }  // reduce quantity only (increase not allowed)
→ 200 { "status": "amended", "new_quantity": 50 }

DELETE /api/v1/orders/{order_id}
→ 200 { "status": "cancelled", "remaining_qty": 50, "filled_qty": 50 }

GET /api/v1/orderbook/AAPL?depth=10
→ {
    "bids": [{"price":150.10,"qty":130,"orders":3}, {"price":150.05,"qty":200,"orders":1}, ...],
    "asks": [{"price":150.15,"qty":200,"orders":1}, ...],
    "last_trade": {"price":150.10,"qty":100,"time":"09:30:01.523"},
    "stats": {"volume":1234567, "vwap": 150.08, "high": 151.20, "low": 149.50, "open": 150.05}
  }

WS: /ws/market-data/AAPL
→ Stream of events:
  { "type":"trade", "price":150.10, "qty":100, "seq":123456 }
  { "type":"bbo", "bid":150.10, "bid_qty":30, "ask":150.15, "ask_qty":200, "seq":123457 }
  { "type":"depth_update", "side":"bid", "price":150.05, "new_qty":0, "action":"delete" }
```

---

## 6. Data Model

### In-Memory — Order Book (per instrument, hot path)

```
OrderBook {
  instrument_id: String
  bids: TreeMap<PriceTicks, DoublyLinkedList<Order>>   // price DESC
  asks: TreeMap<PriceTicks, DoublyLinkedList<Order>>    // price ASC
  best_bid: PriceTicks   // cached, updated on match/cancel
  best_ask: PriceTicks   // cached
  orders: HashMap<OrderId, NodePointer>   // O(1) cancel by order_id
  stop_buys: TreeMap<TriggerPrice, List<StopOrder>>
  stop_sells: TreeMap<TriggerPrice, List<StopOrder>>
  last_trade_price: PriceTicks
  total_volume: long
  vwap_numerator: long   // sum(price * qty) for VWAP calculation
  state: ENUM(pre_open, auction, continuous, halted, closed)
}

Order {
  order_id: UUID
  firm_id: String       // broker/firm identifier
  side: BUY | SELL
  price: PriceTicks     // integer (e.g., 15010 = $150.10)
  remaining_qty: int
  original_qty: int
  time_in_force: DAY | GTC | IOC | FOK
  seq_num: long
  timestamp_ns: long    // nanosecond precision
}

// PriceTicks: integer cents. $150.10 = 15010.
// No floating-point anywhere in matching engine.
```

### PostgreSQL — Durable Store (async write-behind, NOT in critical path)

```sql
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY,
    instrument      VARCHAR(10) NOT NULL,
    firm_id         VARCHAR(20) NOT NULL,
    user_id         UUID,
    side            ENUM('buy','sell') NOT NULL,
    order_type      ENUM('limit','market','stop','stop_limit','ioc','fok') NOT NULL,
    price           BIGINT,     -- integer ticks (cents)
    stop_price      BIGINT,     -- for stop orders
    original_qty    INT NOT NULL,
    filled_qty      INT DEFAULT 0,
    remaining_qty   INT NOT NULL,
    status          ENUM('new','partial','filled','cancelled','expired','rejected') NOT NULL,
    time_in_force   ENUM('day','gtc','ioc','fok') NOT NULL,
    seq_num         BIGINT UNIQUE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ,
    cancel_reason   VARCHAR(100),
    INDEX idx_instrument (instrument, created_at),
    INDEX idx_firm (firm_id, created_at),
    INDEX idx_seq (seq_num)
) PARTITION BY RANGE (created_at);

CREATE TABLE trades (
    trade_id        UUID PRIMARY KEY,
    instrument      VARCHAR(10) NOT NULL,
    buy_order_id    UUID NOT NULL,
    sell_order_id   UUID NOT NULL,
    buy_firm_id     VARCHAR(20),
    sell_firm_id    VARCHAR(20),
    price           BIGINT NOT NULL,     -- integer ticks
    quantity        INT NOT NULL,
    seq_num         BIGINT NOT NULL,
    executed_at     TIMESTAMPTZ NOT NULL,
    INDEX idx_instrument_time (instrument, executed_at),
    INDEX idx_buy_order (buy_order_id),
    INDEX idx_sell_order (sell_order_id)
) PARTITION BY RANGE (executed_at);

-- Partitioned by day for easy archival and fast queries
CREATE TABLE orders_2026_03_14 PARTITION OF orders
  FOR VALUES FROM ('2026-03-14') TO ('2026-03-15');
CREATE TABLE trades_2026_03_14 PARTITION OF trades
  FOR VALUES FROM ('2026-03-14') TO ('2026-03-15');
```

### Kafka — Async Downstream Distribution

```
Topic: market-data-raw         (every order book change, trade — for analytics/compliance)
Topic: trade-reports           (executed trades — for clearing, settlement, drop copy)
Topic: audit-trail             (every order event with timestamps — regulatory requirement)
  Retention: 7 years (regulatory). Tiered storage: recent in Kafka, old in S3.

Kafka is NEVER in the critical (matching) path.
It's used for async fan-out to downstream systems only.
```

### Shared Memory — Risk Engine State

```
// Shared between gateway threads (lockless read, atomic update)
FirmRiskState {
  firm_id: String
  position: AtomicLong per instrument     // net shares (+buy, -sell)
  order_count_window: AtomicLong          // orders in current second
  kill_switch: AtomicBoolean              // emergency halt
  max_position: long                      // configured limit
  max_order_rate: int                     // orders/sec limit
}
// Updated on every fill (atomic add). Checked on every new order (atomic read).
// Shared memory: no serialization, no network — sub-100ns access.
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Matching engine crash** | Replay from sequencer WAL; rebuild order book deterministically; recovery < 1 second |
| **Sequencer failure** | Hot standby with replicated WAL; automatic failover in < 1 sec; bounded event loss (~10μs of events) |
| **Network partition** | Reject orders during partition (safety over availability during market hours) — CP, not AP |
| **Duplicate orders** | Seq_num dedup; each order processed exactly once; ClOrdID dedup at gateway level |
| **Market data loss** | Clients request snapshot + subscribe to stream; gap detection via seq_num; automatic re-sync |
| **Split-brain (two sequencers active)** | Fencing token: new sequencer epoch_id embedded in seq_num; old sequencer's messages rejected |
| **Clock skew** | NTP is insufficient (ms accuracy). Use PTP (Precision Time Protocol) for μs accuracy. GPS-synchronized clocks on all servers |
| **Data center failover** | Active-passive DC pair; manual failover by ops during market hours (too risky for automatic); DR site replays WAL |
| **Algo gone rogue** | Kill switch per firm; circuit breaker per instrument (LULD); order rate throttle |

### Specific: Deterministic Order Book Recovery

```
Matching engine crashes at 10:30:15 AM. How to recover?

1. Sequencer WAL contains: every order, cancel, and amend event with seq_num
   WAL persisted to local NVMe SSD + replicated to standby server.

2. Recovery process:
   a. Start fresh matching engine with empty order books
   b. Replay ALL events from WAL starting from market open (9:30:00 AM)
   c. Each event replayed in strict seq_num order
   d. Same events → same state (deterministic guarantee)
   e. Order book fully reconstructed to exact pre-crash state

3. Recovery time:
   1 hour of events ≈ 3.6M events (at 1K/sec per instrument)
   Replay speed: 10M events/sec (pure in-memory, no I/O, no network)
   Recovery time: < 1 second

4. Gap detection:
   After recovery, matching engine announces its last processed seq_num.
   If any client's last seen seq_num > engine's → client detects gap → reconnects.

5. Snapshot optimization:
   Periodically (every 5 minutes): snapshot entire order book state to disk.
   On recovery: load snapshot + replay only events SINCE snapshot.
   Reduces replay from 6.5 hours (full day) to < 5 minutes of events.

Why deterministic replay is non-negotiable:
  Regulatory requirement: exchange must prove any historical state can be reconstructed.
  Audit question: "What was the order book state at 10:15:23.456789 AM?"
  Answer: replay WAL to that exact seq_num → exact state reproduced.
  SEC Rule 613 (Consolidated Audit Trail) mandates this capability.
```

---

## 8. Additional Considerations

### Regulatory Compliance
- **SEC Rule 613 (Consolidated Audit Trail)**: Must reconstruct any historical order book state from audit trail. Every order event (new, amend, cancel, fill) logged with nanosecond timestamp, participant ID, and sequence number
- **Reg NMS (National Market System)**: Orders must be routed to the exchange with the best price (NBBO — National Best Bid/Offer). Exchange must publish quotes to the SIP (Securities Information Processor) within microseconds
- **MiFID II (EU)**: Clock synchronization to 100μs granularity. Best execution reporting. Transaction reporting to regulators within T+1
- **Market Access Rule (15c3-5)**: Brokers must have pre-trade risk controls before orders reach the exchange. Our pre-trade risk engine enforces this on behalf of member firms
- **Audit retention**: All order and trade data retained for minimum 7 years (SEC) or 5 years (FINRA). Tiered storage: hot (NVMe) → warm (SSD) → cold (S3/Glacier)

### Co-Location and Fairness
- Trading firms co-locate servers in the same data center as the exchange
- **Equal-length cables**: NYSE and NASDAQ provide equal-length network cables to all co-located firms — prevents any firm from having a physical proximity advantage
- **Randomized processing**: Some exchanges (IEX) add a 350μs "speed bump" delay to all incoming orders to neutralize latency arbitrage
- **Fee structure**: Co-location fees range from $5K–$20K/month per rack. Market data fees: $10K–$50K/month for full depth feed

### Dark Pools and Alternative Execution
- **Dark pool**: Off-exchange venue where orders are hidden (no visible order book)
- Used by institutional investors to execute large block orders without moving the market
- Matching logic similar to lit exchange but: no pre-trade transparency, mid-point matching common
- Our exchange may operate a dark pool alongside the lit market — separate matching engine, shared instrument definitions

### Multi-Asset Class Support
- Equities (stocks, ETFs): price-time priority, penny tick increments
- Options: additional complexity — strike, expiry, put/call, implied volatility, complex multi-leg orders (spreads, strangles)
- Futures: similar to equities but with expiration, daily mark-to-market, margin requirements
- Each asset class may have different matching rules, tick sizes, and circuit breaker thresholds
- Shared infrastructure: gateway, sequencer, risk engine. Separate: matching engines per asset class

### Market Making and Liquidity Programs
- **Designated Market Makers (DMMs)**: Obligated to maintain continuous two-sided quotes (bid + ask) with minimum size and maximum spread
- In return: fee rebates, priority in matching (some exchanges give DMMs last-look or size priority)
- **Maker-taker fee model**: Liquidity providers (resting orders) receive rebate (~$0.002/share). Liquidity takers (crossing orders) pay fee (~$0.003/share)
- **Inverted model (BATS BYX)**: Takers receive rebate, makers pay — attracts different order flow

### Monitoring & Alerting
- **Latency percentiles**: P50, P99, P99.9 for order-to-ack, match-to-trade-report, market-data-publish
- **Throughput dashboards**: Orders/sec, trades/sec, cancels/sec — per instrument and aggregate
- **Queue depth**: Sequencer input queue depth, matching engine ring buffer consumer lag
- **Risk alerts**: Firm position breaches, order rate violations, kill switch activations
- **System health**: CPU utilization per pinned core (should be < 30% to avoid thermal throttling), NVMe write latency, memory allocation rate (should be zero on hot path)
- **Market quality metrics**: Bid-ask spread, order book depth, trade-through rate (orders executing at worse-than-NBBO prices)

### End-of-Day and Corporate Actions
- **End-of-day**: Cancel all DAY orders at market close. GTC orders persist overnight. Publish official closing prices (from closing auction)
- **Corporate actions**: Stock splits, dividends, mergers require adjusting open orders
  - Stock split 2:1: all resting order prices halved, quantities doubled
  - Ex-dividend date: adjust open limit orders by dividend amount
  - Applied overnight between close and next open — matching engine offline during adjustment
- **Symbology changes**: Ticker changes (e.g., FB → META) require updating instrument definitions, order book migration

---

## 9. Deep Dive: Engineering Trade-offs

### LMAX Disruptor Pattern

```
The gold standard for exchange matching engines.
Single-threaded event processing with lock-free ring buffer.

Ring buffer: pre-allocated circular array (power of 2 size, e.g., 2^20 = 1M slots)
  Each slot: pre-allocated event object (no GC, no allocation on hot path)
  
  Producer (sequencer): writes events to next slot
    Slot index = seq_num & (buffer_size - 1)  // bitwise AND — faster than modulo
    Memory barrier ensures visibility to consumer
  
  Consumer (matching engine): reads events in order
    Spins on sequence counter — no blocking, no context switches
    Batch processing: reads all available events at once
  
  No allocation, no GC, no locks, no system calls → deterministic microsecond latency

LMAX achieves: 6M transactions/sec with < 1 microsecond latency.
Java with mechanical sympathy (CPU cache-aware programming):
  - Object pooling: reuse event objects, never allocate
  - Cache-line padding: prevent false sharing between threads
  - Busy-spin wait: never yield to OS scheduler
  - No frameworks, no reflection, no virtual dispatch in hot path
  - JIT-compiled tight loops: CPU branch prediction hits 99%+
```

### Why Not Kafka for Order Sequencing?

```
Kafka latency: 2-5 ms per message (99th percentile).
Exchange requirement: < 10 microseconds.
Kafka is 500-5000x too slow for the matching hot path.

Why Kafka is slow (for this use case):
  1. Network round-trip: producer → broker → consumer = 0.5-1ms minimum
  2. Replication: wait for ISR (in-sync replicas) ACK = 1-3ms
  3. Batching: Kafka batches messages for throughput, adding latency
  4. Page cache: relies on OS page cache — unpredictable μs spikes

Where Kafka IS used in exchange architecture:
  - Trade reporting to clearing house (async, 100ms OK)
  - Market data distribution to analytics (async, seconds OK)
  - Regulatory audit trail (async, minutes OK)
  - NOT in the order-to-match critical path
```

### Circuit Breakers — Market Halts

```
Flash crash: price drops 10% in 5 seconds due to algorithmic trading error.

Market-wide circuit breakers (SEC-mandated):
  Level 1: S&P 500 drops 7%  → 15-minute trading halt (all instruments)
  Level 2: S&P 500 drops 13% → 15-minute halt
  Level 3: S&P 500 drops 20% → market closed for the day

Per-instrument circuit breaker (LULD — Limit Up-Limit Down):
  Price bands calculated every 5 minutes:
    Reference price = average trade price over last 5 min
    Band width = 5% for large-cap, 10% for small-cap, 20% for ETFs
    Upper band = reference * (1 + width)
    Lower band = reference * (1 - width)
  
  If trade would execute outside bands:
    1. Instrument enters "Limit State" — only orders that would improve price accepted
    2. If Limit State persists for 15 seconds → 5-minute trading halt
    3. After halt: re-open with auction (batch matching to find clearing price)
  
  Implementation in matching engine:
    after_trade(trade):
      if trade.price > upper_band or trade.price < lower_band:
        set_instrument_state(LIMIT_STATE)
        start_timer(15_seconds)
    
    on_timer_expire():
      if still_in_limit_state:
        set_instrument_state(HALTED)
        start_timer(5_minutes)
    
    on_halt_timer_expire():
      set_instrument_state(AUCTION)
      run_opening_auction()
      set_instrument_state(CONTINUOUS)

  Real-world example: August 24, 2015 "Flash Crash"
    LULD triggered on 1,200+ stocks. Trading halted 1,200+ times in one day.
    System must handle mass-halt scenario without performance degradation.
```

### Market Order Edge Case — Sweeping the Book

```
Problem: Market order "Buy 10,000 shares at market" when book is thin:
  Best ask: $150.00 (100 shares)
  Next: $155.00 (200 shares)
  Next: $160.00 (300 shares)
  Next: $200.00 (1,000 shares) ← stale resting order from last week
  
  Market order sweeps all levels, paying up to $200/share for last fills.
  User expected ~$150 but paid average ~$175. Terrible outcome.

Protection mechanisms:
  1. Market order price collar:
     mid = ($150.00 + $150.15) / 2 = $150.075
     collar = mid * 1.05 = $157.58 (5% from mid)
     Stop execution at $157.58 → remaining quantity rests as limit order at $157.58
  
  2. Synthetic limit:
     Convert all market orders to aggressive limit orders internally:
     "Buy at market" → "Buy at $157.58 limit" (5% above current mid)
     This is what most real exchanges do (NASDAQ, NYSE).
  
  3. Maximum notional value per market order:
     If order.qty * best_ask > $1M → REJECT (require explicit limit order)
  
  4. Minimum book depth requirement:
     If ask side has < 5 price levels → reject market orders for this instrument
     Prevents execution in extremely illiquid conditions
```

### Order Cancel and Amend Race Conditions

```
Race condition 1: Cancel arrives WHILE matching is executing a fill

  T=0: Resting buy order #123 at $150.10 for 100 shares
  T=1: Sell order arrives → matching engine starts filling buy #123
  T=2: Cancel request for order #123 arrives (nanoseconds later)
  
  Single-threaded matching engine resolves this trivially:
    Events processed in seq_num order. If fill's seq_num < cancel's seq_num:
    Fill executes first → order partially/fully filled → cancel either:
      - Cancels remaining qty (if partially filled)
      - Gets "too late to cancel" response (if already fully filled)
  
  In multi-threaded systems: this is a nightmare (locks, CAS, undo logic).
  Single-threaded: no race. Events serialized by design.

Race condition 2: Amend price while order is being matched

  T=0: Resting buy #123 at $150.10. User amends price to $149.00.
  T=1: Sell at $150.10 arrives → would match old price but not new price.
  
  Resolution: amend processed in seq_num order.
    If amend seq_num < sell seq_num: amend applies first, sell doesn't match.
    If sell seq_num < amend seq_num: sell matches at old price, amend fails ("already filled").
  
  Important: amending price = cancel old + insert new.
    New order loses time priority (goes to back of queue at new price level).
    Amending quantity down: order keeps its position (no new insertion needed).

Race condition 3: Duplicate cancel (network retry)
  Client sends cancel. Network timeout. Client resends cancel.
  Second cancel arrives → order already cancelled → return "already cancelled" (idempotent).
  Matching engine checks: orders HashMap lookup → not found → NOOP.
```

### Self-Trade Prevention (STP) — Regulatory Requirement

```
Firm XYZ has a market-making algorithm and a separate alpha-seeking algorithm.
Both may have orders in the same book. If they match → "wash trading" (illegal).

STP modes (configured per firm):
  1. Cancel Newest (CN): incoming order cancelled if it would cross same firm's resting order
  2. Cancel Oldest (CO): resting order cancelled to avoid self-trade
  3. Cancel Both (CB): both orders cancelled
  4. Decrement and Cancel (DC): reduce both quantities by the overlap; cancel remainder

Implementation in matching loop:
  while matching:
    if incoming.firm_id == resting.firm_id:
      apply_stp_mode(incoming, resting, firm.stp_mode)
      skip this match
    else:
      execute_trade(incoming, resting)

  STP is checked INSIDE the matching loop — not as a pre-check.
  Why? Because the same-firm resting order may be at the best price,
  and you need to skip it and match against the NEXT resting order at that price.
```

### Market Data Distribution — Multicast vs Unicast vs Kafka

```
5M market data updates/sec. 10,000+ subscribers. How to deliver efficiently?

Unicast TCP (to each subscriber):
  5M × 10,000 = 50B messages/sec. Impossible.
  ✗ Does not scale with subscriber count
  ✗ Each subscriber adds server-side load
  ✓ Reliable delivery (TCP guarantees order)

Kafka (fan-out):
  5M messages/sec → Kafka → 10,000 consumers
  ✗ Latency: 2-5ms per message (too slow for co-located HFT firms)
  ✗ Consumer coordination overhead
  ✓ Built-in replication and retention
  ✓ Acceptable for non-latency-sensitive consumers (analytics, compliance)

Multicast UDP ⭐ (what real exchanges use):
  Exchange sends ONE copy per update to multicast group address.
  Network switches replicate to all subscribers.
  
  Groups:
    224.1.1.1: AAPL + MSFT + TSLA (partition 1 instruments)
    224.1.1.2: GOOGL + AMZN + NVDA (partition 2)
    Subscribers join only the groups they care about.
  
  Reliability:
    UDP is unreliable (packets can be lost). Solutions:
    1. Sequence numbers: subscriber detects gap → requests retransmission
    2. Retransmission server: subscriber sends NACK → server unicasts missed messages
    3. Snapshot service: subscriber can request full order book snapshot at any time
       Snapshot + replay from last snapshot seq_num = guaranteed consistency

  Wire format: SBE (Simple Binary Encoding) or ITCH protocol
    ITCH message: ~20 bytes for a quote update
    5M × 20 bytes = 100 MB/sec = 800 Mbps — fits in 1 Gbps link easily

  Co-location:
    Trading firms place servers in same data center as exchange (< 1 mile of fiber).
    Market data latency: 5-20 μs from matching engine to co-located subscriber.
    Remote subscribers (over WAN): 1-50 ms latency (depends on distance).

Decision: Multicast UDP for co-located low-latency consumers. Kafka for async 
downstream (analytics, compliance, drop copy). Both running in parallel.
```

### Why Java/C++ (Not Go/Rust/Python) for the Matching Engine?

```
Python:
  ✗ GIL prevents true parallelism (irrelevant for single-threaded, but GC pauses)
  ✗ Interpreted: 100x slower than compiled for tight loops
  ✗ No control over memory layout, allocation, or GC timing
  ✗ No exchange in the world uses Python for matching

Go:
  ✗ GC pauses: Go's GC targets < 500μs pause time — UNACCEPTABLE at μs latency
  ✗ No control over object layout (runtime decides)
  ✗ goroutine scheduling adds non-deterministic latency
  ✓ Great for gateway/risk services (not latency-critical path)

Rust:
  ✓ Zero-cost abstractions, no GC, deterministic memory management
  ✓ Ownership model prevents data races at compile time
  ✗ Smaller ecosystem for financial systems (fewer battle-tested libraries)
  ✗ Hiring pool is smaller than Java/C++ in finance
  ✓ Growing adoption: some new exchanges (e.g., Serum/Solana DEX) use Rust

C++ ⭐ (most exchanges):
  ✓ Zero overhead: no GC, manual memory management, inline assembly possible
  ✓ Full control over cache layout (struct packing, alignment, placement new)
  ✓ Kernel bypass (DPDK, RDMA) for sub-μs networking
  ✓ Used by: NYSE (Pillar), NASDAQ (INET), LSE (Millennium), CME (Globex)
  ✗ Memory safety issues (buffer overflows, use-after-free)
  ✗ Complex build systems, slow compilation

Java ⭐ (LMAX, some exchanges):
  ✓ JIT compilation: hot loops optimized to near-C++ performance
  ✓ LMAX Disruptor pattern achieves < 1μs latency in Java
  ✓ Easier to develop and maintain than C++
  ✗ GC pauses — mitigated by: zero-allocation hot path, off-heap memory,
    Azul Zing JVM (pauseless GC), or GraalVM
  ✓ Used by: LMAX Exchange, some prop trading firms

Decision: C++ for the matching engine hot path (sequencer + matching + market data).
Java for gateway services, risk engine, and trade store.
Python for analytics, surveillance, and batch reporting.
```

### Clearing and Settlement — T+1

```
After a trade executes, it must be "cleared" and "settled":

Clearing (same day):
  1. Net positions: Firm A bought 5000 AAPL and sold 3000 AAPL → net: +2000
  2. Calculate margin: Firm A must deposit margin for its net +2000 position
  3. Risk checks: if margin insufficient → margin call → forced liquidation

Settlement (T+1, next business day):
  1. Firm A actually receives 2000 AAPL shares in their depository account
  2. Firm A's cash is debited: 2000 × trade_price
  3. Counterparty receives cash, delivers shares
  
  Central Counterparty (CCP) like DTCC/NSCC:
    Steps between buyer and seller.
    If one side defaults → CCP guarantees the trade (reduces counterparty risk).

Clearing service in our architecture:
  Consumes trade events from Kafka → accumulates net positions per firm
  End of day: submit clearing file to CCP → CCP confirms
  T+1: settlement instructions sent to depository
  
This is the ONLY part of the exchange that doesn't need microsecond latency.
Batch processing at end of day is fine.
```

### Order Book Data Structure: TreeMap vs Array vs Skip List

```
TreeMap<Price, LinkedList<Order>> (Red-Black Tree):
  Insert at new price: O(log P) where P = distinct price levels
  Best price lookup: O(1) with cached min/max pointer
  Iterate price levels: O(1) per level (in-order traversal)
  Memory: Pointer-heavy → poor cache locality
  ✓ Handles arbitrary price levels efficiently
  ✓ Standard library available in Java/C++
  ✗ O(log P) insert at new price — matters when P is large

Array indexed by price (direct-mapped):
  Price range: $0.01 to $10,000.00 = 1,000,000 price levels (cents)
  Array[price_in_cents] → LinkedList<Order>
  Insert/lookup: O(1) always (direct index)
  Best price: scan from top → O(P) worst case; O(1) with cached pointer
  ✓ O(1) everything with cached best pointer
  ✓ Excellent cache locality (contiguous memory)
  ✗ Wastes memory for sparse price ranges (most slots empty)
  ✗ Price range must be bounded
  ✓ Used by some HFT firms for instruments with known price ranges

Skip List:
  Probabilistic balanced structure
  Insert/lookup: O(log P) expected
  ✓ Lock-free implementations exist (ConcurrentSkipListMap)
  ✗ Not needed since matching is single-threaded
  ✗ More pointer chasing → worse cache locality than array

Recommended ⭐: TreeMap for general case (handles any instrument).
Array for hot instruments with bounded price ranges (AAPL, SPY) where
every nanosecond matters.
```

### FIX Protocol vs Proprietary Binary: Gateway Protocol Choice

```
FIX (Financial Information eXchange) — text-based:
  "8=FIX.4.4|35=D|49=BROKER1|55=AAPL|54=1|38=100|44=15010|..."
  ✓ Industry standard — every broker speaks FIX
  ✓ Human-readable for debugging
  ✗ Parsing overhead: string splitting, field lookup by tag
  ✗ Message size: ~200-500 bytes for a new order
  ✗ Latency: 5-20μs to parse a FIX message

FIX with SBE (Simple Binary Encoding):
  Binary encoding of FIX messages — same semantics, binary wire format
  ✓ Parse in < 1μs (direct memory access, no string parsing)
  ✓ Message size: ~50-100 bytes (4-5x smaller)
  ✓ Compatible with FIX semantics (easy migration)
  ✗ Not human-readable on the wire

Proprietary binary (exchange-specific):
  Custom binary protocol optimized for the exchange's specific needs
  ✓ Absolute minimum latency (< 0.5μs parse time)
  ✓ Minimum message size
  ✗ Every client must implement a custom adapter
  ✗ Maintenance burden: protocol changes require all clients to update
  ✓ Used by: CME (iLink), NASDAQ (OUCH), NYSE (Pillar Gateway)

Decision: FIX/SBE for standard client-facing gateway. Proprietary binary
for co-located ultra-low-latency clients willing to implement custom adapters.
Both gateways normalize to the same internal format before hitting the sequencer.
```

### Matching Priority Models: Price-Time vs Pro-Rata vs Size-Time

```
Price-Time Priority ⭐ (most equity exchanges — NYSE, NASDAQ):
  1. Best price wins (highest bid, lowest ask)
  2. Within same price: first order in time wins (FIFO)
  
  ✓ Simple, fair, transparent
  ✓ Rewards speed → incentivizes tighter quotes
  ✓ Standard for equities — regulatory expectation
  ✗ Favors speed → arms race for microseconds (co-location, FPGA)
  ✗ Small retail orders can't compete with HFT for time priority

Pro-Rata (some options exchanges — CBOE, CME for futures):
  1. Best price wins
  2. Within same price: allocated proportionally to order size
  
  Example: 3 orders resting at $150.10:
    Order A: 100 shares, Order B: 300 shares, Order C: 600 shares
    Incoming sell matches 500 shares at $150.10
    Allocation: A gets 50, B gets 150, C gets 300 (proportional to size)
  
  ✓ Rewards SIZE (quoting large) rather than speed
  ✓ Reduces latency arms race
  ✗ Incentivizes oversized quotes (quote more than you want to fill)
  ✗ Fragmentation: multiple small fills per order
  ✗ More complex implementation (calculate proportional allocation per match)

Size-Time Priority (some options exchanges):
  1. Best price
  2. Largest order first (not proportional — winner-take-all by size)
  3. Ties in size broken by time
  
  ✓ Strongly rewards providing deep liquidity
  ✗ Small market makers disadvantaged

Price-Time with DMM Parity (NYSE for listed stocks):
  1. Best price
  2. DMM (Designated Market Maker) gets priority at each price level
  3. Remaining allocated FIFO among other orders
  
  ✓ Incentivizes DMMs to provide continuous quotes (reduces spreads)
  ✗ Creates two classes of market participants (DMM vs everyone else)

Our design uses Price-Time Priority:
  - Standard for equities
  - Simplest to implement and reason about
  - Regulatory default expectation
  - Single-threaded FIFO matching naturally implements this
  
  Pro-rata would be needed if designing an options or futures exchange.
```

### NVMe WAL vs Battery-Backed RAM vs Persistent Memory: Storage for Sequencer

```
The sequencer's WAL write is the single biggest latency bottleneck.
Every order MUST be persisted before acknowledgement (regulatory requirement).

Regular SSD (SATA):
  fsync latency: 50-200 μs
  ✗ Way too slow for exchange use
  ✗ Would add 50-200μs to every order

NVMe SSD ⭐ (most exchanges):
  fsync latency: 5-20 μs (Intel Optane: 7-10 μs)
  Throughput: 500K+ IOPS for 4KB writes
  ✓ Good balance of cost, performance, and durability
  ✓ Commodity hardware, widely available
  ✗ Still 5-20μs per write — dominates the latency budget

Battery-Backed RAM (BBRAM / NVDIMM):
  Write latency: 0.1-0.5 μs (100-500 nanoseconds!)
  Data survives power loss (battery keeps RAM powered for ~72 hours)
  ✓ 10-100x faster than NVMe
  ✓ Used by top-tier exchanges for absolute minimum latency
  ✗ Expensive ($$$)
  ✗ Capacity limited (typically 128-512 GB per DIMM)
  ✗ Battery maintenance required
  ✗ If battery fails during power outage → data loss

Intel Optane Persistent Memory (PMEM):
  Write latency: 0.3-1 μs
  Byte-addressable (no block I/O overhead)
  ✓ Faster than NVMe, cheaper than BBRAM
  ✗ Intel discontinued Optane line (2022) — limited future availability
  ✗ Requires special programming model (libpmem)

Decision: NVMe (Intel Optane SSD) for most deployments.
BBRAM for the top 5-10 hottest instrument partitions where every μs matters.
The replication to standby provides the durability guarantee; the local WAL 
provides regulatory compliance for "order received" timestamping.
```

