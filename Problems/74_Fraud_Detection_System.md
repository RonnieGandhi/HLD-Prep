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

