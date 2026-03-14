# 74. Design a Fraud Detection System

---

## 1. Functional Requirements (FR)

- **Real-time scoring**: Score every transaction/event for fraud risk in < 100 ms
- **Rule engine**: Configurable rules (velocity checks, amount limits, geo-anomalies)
- **ML models**: Machine learning models for pattern detection (supervised + unsupervised)
- **Case management**: Queue suspicious events for human analyst review
- **Block/allow decisions**: Auto-block high-risk, auto-allow low-risk, manual review for medium
- **Feature store**: Real-time and historical features (user behavior, device fingerprint, transaction patterns)
- **Feedback loop**: Analyst decisions feed back into ML model training
- **Multi-channel**: Detect fraud across payments, account creation, login, promo abuse

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Fraud decision in < 100 ms (in transaction critical path)
- **High Recall**: Catch > 95% of fraud (false negatives are costly)
- **Acceptable Precision**: False positive rate < 5% (blocking legitimate users is costly too)
- **Scale**: 50K+ transactions/sec scoring
- **Availability**: 99.99% — fraud system down means either blocking all or allowing all
- **Adaptability**: New fraud patterns detected and rules deployed within hours

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Transactions scored / sec | 50K |
| ML features per transaction | ~200 |
| Feature computation latency | < 30 ms |
| Model inference latency | < 20 ms |
| Rule evaluation latency | < 10 ms |
| Total fraud decision latency | < 100 ms |
| Fraud rate | ~0.5% of transactions |
| Manual review queue | ~50K cases/day |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    REAL-TIME SCORING PATH (< 100 ms)                   │
│                                                                        │
│  Transaction --> API Gateway --> Fraud Scoring Service                  │
│                                       |                                │
│           +---------------------------+---------------------------+    │
│           |                           |                           |    │
│    +------v------+            +-------v-------+           +-------v--+ │
│    | Feature     |            | Rule Engine   |           | ML Model | │
│    | Service     |            | (in-memory,   |           | Service  | │
│    | (compute    |            |  configurable |           | (in-proc | │
│    |  200 feat-  |            |  JSON rules)  |           |  XGBoost | │
│    |  ures in    |            |               |           |  + Auto- | │
│    |  < 30 ms)   |            | - impossible  |           |  encoder)| │
│    +------+------+            |   travel      |           +-------+--+ │
│           |                   | - velocity    |                   |    │
│    +------v------+            |   breach      |                   |    │
│    | Redis       |            | - amount      |                   |    │
│    | (real-time  |            |   threshold   |                   |    │
│    |  features:  |            +-------+-------+                   |    │
│    |  velocity,  |                    |                           |    │
│    |  device     |            +-------v---------------------------v--+ │
│    |  history,   |            | Decision Aggregator                  | │
│    |  geo)       |            | final = 0.5*xgb + 0.3*ae + 0.2*rules| │
│    |             |            |                                      | │
│    | ClickHouse  |            | < 0.3: ALLOW   0.3-0.7: REVIEW      | │
│    | (historical |            | > 0.7: BLOCK                        | │
│    |  features:  |            +------+------+------+---------+-------+ │
│    |  30d avg,   |                   |      |      |                  │
│    |  patterns)  |                   |      |      |                  │
│    +-------------+                   v      v      v                  │
│                               ALLOW  REVIEW  BLOCK                    │
└────────────────────────────────────|──────|──────|─────────────────────┘
                                     |      |      |
┌────────────────────────────────────|──────|──────|─────────────────────┐
│                          ASYNC LAYER      |      |                    │
│                                     |     |      |                    │
│                              +------v-----v------v------+             │
│                              |         Kafka            |             │
│                              | (fraud-scoring-events)   |             │
│                              +------+------+------+-----+             │
│                                     |      |      |                   │
│                 +-------------------+      |      +--------+          │
│                 |                          |               |          │
│          +------v------+           +------v------+  +------v------+   │
│          | Case Mgmt   |           | Feature     |  | Analytics   |   │
│          | Service     |           | Updater     |  | (ClickHouse |   │
│          | (analyst    |           | (Flink:     |  |  model perf,|   │
│          |  dashboard, |           |  update     |  |  precision/ |   │
│          |  review     |           |  real-time  |  |  recall per |   │
│          |  queue,     |           |  features   |  |  category)  |   │
│          |  decisions) |           |  in Redis)  |  +-------------+   │
│          +------+------+           +-------------+                    │
│                 |                                                      │
│          +------v------+           +--------------------+             │
│          | Feedback    |           | Model Training     |             │
│          | Pipeline    |---------->| Pipeline (weekly)  |             │
│          | (analyst    |           | - Spark feature    |             │
│          |  decisions  |           |   engineering      |             │
│          |  -> labeled |           | - XGBoost retrain  |             │
│          |  training   |           | - A/B test new vs  |             │
│          |  data)      |           |   old model        |             │
│          +-------------+           | - Auto-deploy if   |             │
│                                    |   metrics improve  |             │
│                                    +--------------------+             │
│                                                                       │
│  DATA STORES:                                                         │
│  +----------+ +--------+ +-------+ +-----------+ +--------+          │
│  |PostgreSQL| | Redis  | | Kafka | | ClickHouse| | S3     |          │
│  |(rules,   | |(real-  | |(all   | |(historical| |(model  |          │
│  | cases,   | | time   | |events)| | events,   | | artif- |          │
│  | decisions| | feat-  | |       | | features) | | acts)  |          │
│  | feedback)| | ures)  | |       | |           | |        |          │
│  +----------+ +--------+ +-------+ +-----------+ +--------+          │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Feature Computation — Real-Time + Historical

```
200 features per transaction, computed in < 30 ms:

Real-time features (Redis, < 5 ms):
  - transaction_count_last_1h: INCR + TTL key per user
  - transaction_amount_last_24h: sorted set with timestamp scores
  - unique_merchants_last_7d: HyperLogLog per user
  - failed_transactions_last_1h: counter
  - device_fingerprint_seen_before: SET membership
  - ip_address_country: GeoIP lookup
  - time_since_last_transaction: timestamp diff

Historical features (pre-computed, ClickHouse -> Redis cache):
  - avg_transaction_amount_30d: pre-computed daily
  - typical_transaction_hour: histogram of user's transaction times
  - typical_merchant_categories: user's usual spending categories
  - account_age_days: user creation date
  - lifetime_transaction_count: aggregate

Derived features (computed at scoring time):
  - amount_deviation: (current_amount - avg_amount_30d) / stddev_amount_30d
  - geo_velocity: distance_from_last_txn / time_since_last_txn
    (If user was in NYC 10 min ago and now in London -> impossible -> fraud)
  - is_new_device: device not in user's device history
  - is_new_merchant: merchant not in user's history
  - hour_anomaly: abs(current_hour - typical_hour) / 12

Feature store architecture:
  Flink: consumes transaction events -> updates real-time features in Redis
  Spark (nightly): computes historical features -> stores in Redis/feature table
  Scoring time: Feature Service reads from Redis -> constructs 200-dim feature vector
```

#### ML Model Architecture

```
Ensemble of multiple models:

1. XGBoost (primary, supervised):
   Trained on labeled data: { features, is_fraud: 0/1 }
   Labels from: confirmed fraud (chargebacks), analyst decisions
   Features: all 200 features
   Output: P(fraud) 0.0-1.0
   Strengths: fast inference (< 5 ms), handles tabular data well
   Retrained: weekly on rolling 6-month window

2. Autoencoder (anomaly detection, unsupervised):
   Trained on legitimate transactions only
   Learns: "normal" transaction pattern
   High reconstruction error = anomaly = potential fraud
   Catches: NEW fraud patterns not in labeled training data
   
3. Graph Neural Network (network analysis):
   Model relationships: user -> device -> IP -> merchant
   Detect fraud rings: cluster of accounts sharing devices/IPs
   Expensive: run offline, flag suspicious clusters for enhanced scrutiny

Scoring:
  final_score = 0.5 * xgboost_score + 0.3 * autoencoder_anomaly + 0.2 * graph_risk
  
  score < 0.3: ALLOW (auto-approve)
  score 0.3-0.7: REVIEW (queue for analyst)
  score > 0.7: BLOCK (auto-decline)

Model serving:
  Models loaded in-memory in Fraud Service (no external ML serving call)
  XGBoost inference: < 1 ms. Autoencoder: < 5 ms. Total: < 10 ms.
```

#### Rule Engine — Fast, Configurable

```
Rules run IN ADDITION to ML model. Rules catch known patterns immediately.

Example rules (JSON-configurable, no code deploy needed):

{
  "rule_id": "R001",
  "name": "high_amount_new_account",
  "condition": "transaction.amount > 500 AND user.account_age_days < 7",
  "action": "review",
  "priority": 10
},
{
  "rule_id": "R002",
  "name": "impossible_travel",
  "condition": "geo_velocity_kmh > 1000",
  "action": "block",
  "priority": 1
},
{
  "rule_id": "R003",
  "name": "velocity_breach",
  "condition": "transaction_count_last_1h > 20",
  "action": "block",
  "priority": 2
}

Rule evaluation: < 10 ms (rules evaluated in-memory, priority order, short-circuit on block)
Rules override ML when action = "block" (safety net for known fraud patterns)
```

---

## 5. APIs

```http
POST /api/v1/fraud/score
{
  "event_type": "payment",
  "transaction_id": "txn-uuid",
  "user_id": "user-uuid",
  "amount": 599.99,
  "merchant_id": "m-uuid",
  "device_fingerprint": "fp-abc",
  "ip_address": "203.0.113.42",
  "timestamp": "2026-03-14T11:00:00Z"
}
Response: 200 OK (< 100 ms)
{
  "decision": "allow",
  "score": 0.15,
  "risk_factors": ["new_device"],
  "rule_triggers": [],
  "request_id": "req-uuid"
}

POST /api/v1/fraud/feedback  (analyst decision)
{ "case_id": "case-uuid", "decision": "fraud_confirmed", "analyst_id": "..." }
```

---

## 6. Data Model

### Redis — Real-Time Features
```
txn_count_1h:{user_id}          -> INT, TTL 3600
txn_amount_24h:{user_id}        -> Sorted Set { txn_id: amount }, TTL 86400
device_history:{user_id}        -> SET of fingerprints
last_location:{user_id}         -> Hash { lat, lng, timestamp }
ip_reputation:{ip}              -> FLOAT (risk score from threat intel feed)
```

### PostgreSQL — Rules & Cases
```sql
CREATE TABLE fraud_rules (
    rule_id VARCHAR(20) PRIMARY KEY, name VARCHAR(100),
    condition TEXT NOT NULL, action ENUM('allow','review','block'),
    priority INT, active BOOLEAN DEFAULT TRUE
);

CREATE TABLE fraud_cases (
    case_id UUID PRIMARY KEY, event_type VARCHAR(20),
    transaction_id UUID, user_id UUID, score DECIMAL(4,3),
    decision ENUM('pending','fraud_confirmed','legitimate','escalated'),
    auto_decision VARCHAR(10), analyst_id UUID,
    created_at TIMESTAMPTZ DEFAULT NOW(), resolved_at TIMESTAMPTZ,
    INDEX idx_pending (decision) WHERE decision = 'pending'
);
```

### ClickHouse — Historical Events
```sql
CREATE TABLE fraud_events (
    transaction_id UUID, user_id UUID, amount Float32,
    score Float32, decision String, features String,
    timestamp DateTime
) ENGINE = MergeTree() ORDER BY (user_id, timestamp);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Fraud service down** | Fail-open for low-value txns (allow with async review); fail-close for high-value (decline) |
| **Feature store lag** | Use stale features with reduced confidence; increase review threshold |
| **ML model error** | Rule engine as safety net; circuit breaker on model; fallback to rules-only |
| **False positive spike** | Monitor auto-block rate; alert if > 2x normal; auto-switch to review mode |
| **Feedback loop delay** | Weekly model retrain; interim: update rules for new patterns within hours |

---

## 8. Deep Dive

### Fail-Open vs Fail-Close

```
Fraud service unavailable. Two options:

Fail-Open (allow all): Revenue preserved. But fraud losses increase during outage.
  Use for: low-value transactions (< $50), known good users (long history)

Fail-Close (block all): No fraud. But ALL legitimate transactions blocked.
  Use for: high-value transactions (> $500), new accounts

Hybrid (recommended):
  score_available = false
  if transaction.amount < 50 AND user.account_age > 90 days:
    allow (low risk, fail-open)
  elif transaction.amount > 500:
    decline (high risk, fail-close)
  else:
    allow + queue for async review (medium risk)
```

### Why Both Rules AND ML

```
Rules: deterministic, explainable, instant deployment for known patterns.
  "Block all transactions from sanctioned countries" -> rule, not ML.

ML: catches novel patterns, handles complex feature interactions.
  "User's spending pattern changed subtly over 3 weeks" -> ML, not rules.

Together: Rules catch known fraud immediately. ML catches new fraud patterns.
  Rules: high precision for specific patterns. ML: high recall for general fraud.
  Defense in depth: if ML misses it, rules may catch it (and vice versa).
```

---

## 9. Deep Dive: Engineering Trade-offs

### Feature Freshness & Staleness Handling

```
Real-time features (Redis, Flink-updated) can become stale if pipeline lags.

Detection: Every feature hash includes _updated_at timestamp
  At scoring time:
    feature_age = now() - features._updated_at
    if feature_age > 5 minutes for velocity features → STALE
    if feature_age > 24 hours for historical features → STALE (nightly job late)

Handling stale features:
  Strategy 1: Conservative bias
    If velocity features are stale → add +0.15 to fraud score
    Rationale: "I can't see recent activity → assume higher risk"
    A legitimate user with stale features gets flagged for REVIEW (not BLOCK)
    
  Strategy 2: Default substitution
    Replace stale feature with population median
    avg_transaction_amount_30d → unknown → use $75 (population median)
    Reduces model confidence but doesn't block legitimate users
    
  Strategy 3: Feature importance gating
    If top-5 most important features are stale → route to REVIEW (skip ML)
    If only minor features are stale → score normally (minor accuracy loss)

Missing features (cold start — new user):
  Account < 24 hours old → ALL historical features are zero/missing
  Solution: Use population-level defaults + new_account flag
    new_account is itself a strong fraud signal (most fraud uses new accounts)
    Lower REVIEW threshold: score > 0.2 → REVIEW (vs 0.3 for established users)

Monitoring dashboard (ClickHouse):
  "% of predictions with stale features" — target < 2%
  "% of predictions with missing features" — target < 5%
  "Feature freshness p99" — target < 60 seconds for real-time features
  Alert if any metric exceeds threshold → investigate Flink pipeline health
```

### Model Version Management During Canary Deploys

```
Two models always loaded in memory on every Fraud Service instance:
  Champion: current production model (v12, trained March 7)
  Challenger: candidate model (v13, trained March 14)

Dual scoring (every transaction scored by BOTH models):
  Transaction arrives → score with champion → score with challenger
  Only champion's decision is enforced (allow/review/block)
  Both scores logged to ClickHouse for comparison

  Why dual scoring?
    Head-to-head comparison on exact same traffic → no confounding factors
    Can compare precision/recall offline without any user impact
    Cost: ~2× CPU for scoring (~10ms × 2 = 20ms)
    Since scoring is < 20ms and total budget is 100ms → acceptable

Canary deployment stages:
  Week 1: Shadow mode
    100% champion enforced, challenger logged only
    Compare: AUC, precision@0.3, recall, false positive rate
    
  Week 2: 5% canary (if Week 1 metrics look good)
    95% traffic: champion enforced
    5% traffic: challenger enforced
    Monitor real-world impact: block rate, review queue size, fraud chargebacks
    
  Week 3: 50% split (if Week 2 metrics look good)
    50/50 split for statistical significance
    
  Week 4: 100% promotion (if all metrics pass)
    Challenger becomes new champion
    Old champion becomes rollback model
    
  Promotion criteria (ALL must pass):
    Fraud detection rate ≥ champion (no recall regression)
    False positive rate ≤ champion + 0.5% (acceptable precision loss)
    Latency p99 ≤ champion + 2ms
    No category-specific regression (e.g., payment fraud up but promo fraud down → investigate)

Rollback:
  Config flag: "active_model_version" in Redis
  Flip from "v13" to "v12" → takes effect within 1 second
  No deployment needed — both models already in memory
  Rollback trigger: automated if fraud loss exceeds 120% of champion's loss rate
```

### Graph Neural Network (GNN) — Production Architecture

```
GNN does NOT run in the real-time scoring path (too expensive at < 100ms).
It runs as an OFFLINE batch pipeline that produces per-user risk scores.

Nightly pipeline:
  1. Build transaction graph (Spark, 2 hours):
     Nodes: users, devices, IPs, merchants, shipping addresses
     Edges: user → device (logged in from), user → IP (transacted from),
            user → merchant (purchased at), user → address (shipped to)
     Edge weight: number of transactions in last 90 days
     Graph size: ~500M nodes, ~2B edges
  
  2. Community detection — Louvain algorithm (Spark GraphX, 1 hour):
     Detect clusters of tightly-connected nodes
     Output: cluster_id for each node
     
     Example fraud ring detected:
       Cluster #4271:
         50 user accounts
         Sharing 3 device fingerprints
         Sharing 2 IP addresses
         All transacting with merchant M_789
         Cluster fraud rate: 35% (vs 0.5% baseline)
         → Flag ALL 50 accounts as graph_risk = HIGH
  
  3. Score clusters (Python, 30 min):
     cluster_fraud_rate = confirmed_fraud_in_cluster / total_in_cluster
     If cluster_fraud_rate > 10% → all members flagged HIGH
     If cluster_fraud_rate > 5% → all members flagged MEDIUM
     
     Per-user features computed from graph:
       graph_risk_score: 0.0-1.0 (based on cluster fraud rate)
       shared_device_count: how many other users share this device
       cluster_size: how many accounts in the cluster
       hop_distance_to_known_fraud: shortest path to a confirmed fraudster
  
  4. Write to Redis (bulk, 15 min):
     HSET graph:risk:{user_id} score 0.85 cluster_id 4271 ...
     TTL: 24 hours (refreshed nightly)
  
  At scoring time (real-time):
     graph_risk = HGET graph:risk:{user_id} score → 0.85 (< 1ms lookup)
     This is just another feature in the 200-feature vector
     No GNN inference at runtime — only a pre-computed lookup

  Cost: ~4 hours nightly compute (Spark cluster)
  Value: catches fraud rings that individual-transaction models miss
    Without GNN: each account in the ring looks low-risk individually
    With GNN: shared devices/IPs reveal coordinated fraud
```

### Fail-Open vs Fail-Close: Threshold-Based Decision

```
Fraud Service is DOWN. What do we do?

Simplistic: Fail-open (allow all) or Fail-close (block all)
  Both are bad — either fraud spikes or revenue drops to zero

Threshold-based failover (recommended):

  if fraud_service_available:
      score = fraud_service.score(transaction)
      decision = apply_thresholds(score)
  else:
      # Graceful degradation based on transaction risk profile
      if transaction.amount < $50 AND user.account_age > 90 days:
          decision = ALLOW  # Low risk, high confidence user
          # Async: queue for post-hoc fraud review when service recovers
      elif transaction.amount > $500 OR user.account_age < 7 days:
          decision = DECLINE  # High risk, protect the platform
      else:
          decision = ALLOW  # Medium risk, allow with async review
          # Apply basic rule checks in-process (no ML):
          if user.country != card.issuing_country:
              decision = DECLINE
          if transaction_count_last_hour > 10:  # simple velocity check
              decision = DECLINE

  Revenue impact analysis:
    Average fraud service downtime: 5 min/month
    During 5 min downtime with threshold-based failover:
      ~1,500 transactions processed (5K/sec × 5 min sample)
      ~750 low-risk auto-allowed → 0 expected fraud
      ~250 high-risk auto-declined → $12,500 lost revenue
      ~500 medium-risk auto-allowed → ~$250 fraud loss (0.1% fraud rate)
      Total impact: ~$12,750 vs $0 loss if system stays up
      
    Compare: fail-close (block all) → $75,000 lost revenue in 5 min
    Compare: fail-open (allow all) → $3,750 fraud loss in 5 min
    Threshold-based: best of both worlds
```

### Weekly vs Daily Model Retraining

```
Weekly retraining (current design):
  ✓ Stable: model has time to be evaluated before next version
  ✓ Sufficient for most fraud patterns (evolve over weeks, not hours)
  ✗ New fraud campaign on Monday → model doesn't learn until next Monday
  ✗ 7-day latency to adapt to new patterns

Daily retraining:
  ✓ 7× faster adaptation to new patterns
  ✓ Captures weekend vs weekday differences better
  ✗ More infrastructure: daily training pipeline, daily validation
  ✗ Risk of overfitting to noise (one bad day pollutes model)
  ✗ Need robust automated validation (can't manually review daily)

Hybrid approach (recommended):
  Base model: retrained weekly (stable foundation)
  Rule engine: updated within HOURS for known new patterns
    → Analyst spots new pattern → creates rule → deployed in 30 min
    → Rule catches new fraud immediately while model takes a week to learn
  
  Example timeline:
    Monday 10 AM: New fraud pattern detected (fake refunds on gift cards)
    Monday 11 AM: Analyst creates rule: "block refund if gift card + amount > $200"
    Monday 11:30 AM: Rule deployed → pattern blocked immediately
    Following Monday: Model retrained with labeled examples → catches variations too
    Monday+2 weeks: Rule can be retired (model handles it natively)

  Rules are fast first-responders. ML is the long-term defense.
```

