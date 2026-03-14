# 97. Design an Ad Click Prediction System

---

## 1. Functional Requirements (FR)

- Predict probability a user will click an ad (CTR prediction) in < 10ms
- Feature engineering from user profile, ad creative, context (page, time, device)
- Model training pipeline: daily retraining on latest click data
- Online learning: model adapts to recent patterns within hours
- A/B testing: compare model versions on live traffic
- Feature store: consistent features between training and serving
- Calibration: predicted probabilities must match actual click rates

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: < 10ms p99 for prediction (in the ad auction critical path)
- **High Throughput**: 1M+ predictions/sec
- **Freshness**: Model reflects behavior from last 24h
- **Accuracy**: 0.1% CTR improvement = millions in revenue

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Predictions / sec | 10M auctions × 50 ads each | 500M feature lookups, 50M predictions |
| Feature lookup latency budget | | < 2ms |
| Model inference latency budget | | < 5ms |
| Features per prediction | | ~100–200 |
| Training data / day | All impression+click logs | ~10 TB |
| Model size | LightGBM / DNN | 100 MB – 2 GB |
| Feature store (online) | 1B users × 2 KB features | ~2 TB (Redis cluster) |
| Feature store (offline) | Historical, Parquet on S3 | ~100 TB |
| Model retraining | | Daily (batch) + hourly (online) |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────┐
│  REAL-TIME SERVING (< 10ms per prediction)                       │
│                                                                   │
│  Ad Request → Feature Retrieval (Redis, < 2ms)                   │
│             → Feature Assembly (user + ad + context features)     │
│             → Model Inference (ONNX/TensorRT, < 5ms)             │
│             → Return P(click) score                               │
│                                                                   │
│  Model: Gradient Boosted Trees (LightGBM) or Deep & Cross Network│
│  Features (~100):                                                 │
│    User: age, gender, interests, past CTR, session depth          │
│    Ad: category, advertiser, creative size, historical CTR        │
│    Context: page category, time of day, device, geo               │
│    Cross: user_interest × ad_category (interaction features)      │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  FEATURE STORE                                                    │
│                                                                   │
│  Offline (Batch):                                                 │
│  - Spark/Hive: compute user-level features daily                 │
│  - Historical CTR per user-ad_category, click patterns            │
│  - Write to Redis / DynamoDB                                      │
│                                                                   │
│  Online (Real-time):                                              │
│  - Flink: streaming features (session clicks, recency)           │
│  - Write to Redis (TTL: 1 hour)                                  │
│                                                                   │
│  Training ↔ Serving consistency:                                  │
│  - Same feature pipeline code for training and serving            │
│  - Feature store ensures no training-serving skew                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  TRAINING PIPELINE (Daily)                                        │
│                                                                   │
│  1. Click logs (Kafka → S3) → training data                     │
│  2. Label: clicked=1, not_clicked=0 (with attribution window)    │
│  3. Feature engineering (Spark): join click logs with features   │
│  4. Train model (LightGBM / DNN)                                 │
│  5. Calibration: isotonic regression to match predicted vs actual │
│  6. Evaluation: AUC, LogLoss, calibration plot                   │
│  7. A/B deploy: shadow mode → 1% canary → 50% → 100%           │
│                                                                   │
│  Challenge: Class imbalance (CTR ~1%, 99% negative)              │
│  Solution: Negative downsampling + weight correction              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Key Design Decisions & Deep Dives

### Model Choice
```
LightGBM (gradient boosted trees):
  ✅ Fast inference (< 1ms), interpretable, handles sparse features
  ❌ Can't learn complex interactions automatically

Deep & Cross Network (DCN):
  ✅ Automatically learns feature interactions (crossing layers)
  ❌ Slower inference (~5ms), needs GPU for serving

Practice: Two-stage
  Stage 1: LightGBM for candidate scoring (fast, high recall)
  Stage 2: DNN for final ranking (slow but accurate, only top 50 candidates)
```

### Calibration (Critical for Bidding)
```
Why: If model says P(click)=0.05 but actual CTR is 0.03 → overbid by 67%
  → Advertiser overspends, exchange loses trust

How: 
  Isotonic regression: maps raw scores to calibrated probabilities
  Platt scaling: logistic regression on model outputs

Monitoring:
  Expected calibration error (ECE) < 0.01
  Bucket predictions into deciles → compare predicted vs actual CTR
```

### Position Bias Correction
```
Problem: Ads in position 1 get clicked more regardless of relevance
Solution: Train on "position-aware" features, but serve WITHOUT position
  Training: include position as feature → model learns position bias
  Serving: set position=1 for all → model predicts "click if shown in position 1"
  
Alternative: IPW (Inverse Propensity Weighting)
  Weight each sample by 1/P(position), reducing position bias
```

---

## 6. APIs

```
# Real-time prediction (called by ad exchange in auction path)
POST /api/predict
{
  "user_id": "u_abc123",
  "ad_candidates": ["ad_001", "ad_002", "ad_003"],
  "context": { "page_url": "...", "device": "mobile", "geo": "US" }
}
→ Response:
{
  "predictions": [
    {"ad_id": "ad_001", "p_click": 0.032, "calibrated": true},
    {"ad_id": "ad_002", "p_click": 0.018, "calibrated": true},
    {"ad_id": "ad_003", "p_click": 0.045, "calibrated": true}
  ],
  "latency_ms": 8
}

# Model management
POST /api/models/deploy      → Deploy new model version (canary)
GET  /api/models/active       → Current model version + metrics
POST /api/models/rollback     → Revert to previous model

# Feature management
GET  /api/features/online?user_id=...  → Retrieve user features
POST /api/features/backfill           → Trigger offline→online sync
```

---

## 7. Data Models

### Redis (Online Feature Store)

```
HSET user:features:{user_id}
  avg_ctr_7d "0.032"
  click_count_30d "145"
  top_category "electronics"
  session_depth "3"
  last_click_hours "2.5"
  _updated_at "2026-03-14T10:05:00Z"
EXPIRE user:features:{user_id} 86400

HSET ad:features:{ad_id}
  historical_ctr "0.025"
  category "travel"
  creative_size "300x250"
  advertiser_id "adv_789"
```

### S3 / Parquet (Offline Feature Store + Training Data)

```
Training data schema (one row per impression):
  user_id, ad_id, timestamp,
  user_features: [avg_ctr_7d, click_count_30d, ...],
  ad_features: [historical_ctr, category, ...],
  context_features: [page_category, time_of_day, device, ...],
  label: clicked (0 or 1),
  position: 3  (for position bias correction)

Partitioned by: date
Format: Parquet (columnar, efficient for Spark training)
Retention: 90 days of impression logs
```

### Kafka (Click Stream)

```
Topic: ad-impressions (partitioned by user_id, RF=3)
  { "user_id": "u_abc", "ad_id": "ad_001", "timestamp": "...",
    "position": 2, "p_click": 0.032, "clicked": false }

Topic: ad-clicks (partitioned by user_id, RF=3)
  { "user_id": "u_abc", "ad_id": "ad_001", "click_timestamp": "..." }

→ Flink joins impressions + clicks → training labels
→ ClickHouse for analytics dashboards
```

---

## 8. Fault Tolerance

| Concern | Solution |
|---|---|
| **Model serving failure** | Fall back to simpler model (logistic regression) or last-known-good; both always loaded in memory |
| **Feature store unavailable** | Use default features (population medians); reduces accuracy, not availability |
| **Stale features** | TTL enforcement; `_updated_at` check; degrade gracefully with confidence reduction |
| **Training failure** | Don't deploy; keep serving current model; alert ML team |
| **Canary deployment** | New model serves 5% traffic first → monitor AUC, calibration → promote or rollback |
| **Redis shard failure** | Redis Cluster auto-failover to replica; missing features for ~1% of users during failover (~30s) |
| **Flink pipeline lag** | Session features stale; batch features (24h) still valid; increase review threshold |
| **Impression-click join failure** | Late clicks arriving after attribution window (30 min); extend window to 60 min with watermark; accept ~0.1% label noise |

### Model Redundancy

```
Two models always warm in memory on every serving host:
  Champion (current production model, v47)
  Challenger (candidate model, v48, or last-known-good v46)

Routing: config flag determines which model is active
  Normal: champion serves 100% traffic
  Canary: champion 95%, challenger 5%
  Rollback: instant config flip → challenger becomes active, <60 seconds

Both models score EVERY request (dual scoring):
  Active model's score → used for auction
  Inactive model's score → logged for offline comparison
  
  Why dual scoring?
    Enables head-to-head comparison on same traffic
    Detects regression BEFORE promoting challenger
    Cost: ~2× inference compute (acceptable given <5ms per model)
```

---

## 9. Scalability: Feature Store at Scale

```
Scale: 1B users × 2 KB features = 2 TB online feature store

Redis Cluster architecture:
  2 TB / 64 GB per node = 32 primary shards + 32 replicas = 64 nodes
  Sharding: CRC16(user_id) % 16384 → slot → shard
  Each prediction: 2 parallel Redis calls (user features + ad features) < 2ms
  
Read path (per prediction):
  1. Hash user_id → shard A → HGETALL user:features:{user_id}  (0.5 ms)
  2. Hash ad_id → shard B → HGETALL ad:features:{ad_id}        (0.5 ms)
  3. Context features computed in-process (no Redis call)        (0.1 ms)
  4. Assemble 200-dim feature vector                            (0.2 ms)
  Total: < 2ms (parallel Redis calls)

Write path (feature updates):
  Batch (nightly Spark): 1B users × 2KB = 2 TB written to Redis
    Pipeline: Spark → Redis PIPELINE (batch 1000 HSETs per pipeline)
    Duration: ~2 hours (parallel across shards)
    During write: reads still served from old values (eventually consistent)
  
  Streaming (Flink): 500M updates/day = ~6K updates/sec to Redis
    Flink keyBy(user_id) → ensures single writer per user per subtask
    Write: HSET user:features:{user_id} session_depth 5
  
Hot-key problem:
  Viral user (celebrity, influencer) → millions of ad impressions
  → Flink tries to update their features thousands of times per second
  → One Redis shard becomes hot
  
  Solution: Rate-limit feature updates per user
    If user_id seen > 100 times in last minute → skip update (sample 1-in-10)
    Features like "session_depth" change slowly; exact count doesn't matter
    Monitor: track p99 latency per Redis shard; alert if >2ms

Feature TTL strategy:
  session_depth, last_click_hours:        TTL 1 hour  (volatile)
  avg_ctr_7d, click_count_30d:            TTL 24 hours (refresh daily)
  top_category, lifetime_engagement:      TTL 7 days  (stable)
  _updated_at:                            No TTL (always present)
  
  On TTL expiry: feature returns nil → Feature Service uses population default
  Monitor: "% predictions with missing features" dashboard (target < 5%)
```

### Race Conditions in Online Feature Updates

```
Race 1: Flink writes feature while prediction reads it
  Flink: HSET user:features:u_abc avg_ctr_7d 0.035
  Prediction: HGETALL user:features:u_abc (concurrent)
  
  Could prediction see partial hash? NO — Redis single-threaded.
  HSET and HGETALL are serialized. Prediction sees either old or new values.
  
  But: multi-field update risk
    Flink updates avg_ctr_7d, then click_count_30d (two commands)
    Between the two: prediction reads → sees new avg_ctr but old count
    → Inconsistent feature vector (avg/count mismatch)
    
    Solution: MULTI/EXEC transaction for multi-field updates
      MULTI
      HSET user:features:u_abc avg_ctr_7d 0.035
      HSET user:features:u_abc click_count_30d 420
      HSET user:features:u_abc _updated_at 1710400000
      EXEC
    → Atomic: prediction sees all-old or all-new

Race 2: Two Flink workers updating same user
  Kafka partition 1: impression event for u_abc → Worker A
  Kafka partition 2: click event for u_abc → Worker B
  Both try to update user:features:u_abc simultaneously
  
  Solution: Partition Kafka by user_id
    All events for u_abc go to same partition → same Flink subtask
    Single writer per user → no concurrent updates
    
  If partitioning by user_id isn't possible (different topics):
    Use Redis Lua script for read-modify-write atomically:
      local count = redis.call('HINCRBY', KEYS[1], 'click_count_30d', 1)
      local impressions = redis.call('HGET', KEYS[1], 'impression_count_30d')
      local ctr = count / tonumber(impressions)
      redis.call('HSET', KEYS[1], 'avg_ctr_7d', tostring(ctr))
```

---

## 10. Deep Dive: Engineering Trade-offs

### LightGBM vs Deep Neural Network

```
LightGBM (gradient boosted trees):
  Inference latency: < 1 ms (single CPU core)
  Training: 2 hours on 1B samples (16-core machine)
  Interpretability: feature importance, SHAP values
  Feature engineering: manual (need to create interaction features)
  AUC: 0.785 (typical for CTR prediction)
  
  Best for: Candidate scoring (fast, high recall)

Deep & Cross Network (DCN-v2):
  Inference latency: 3-5 ms (GPU-accelerated)
  Training: 12 hours on 1B samples (8 GPUs)
  Interpretability: opaque (black box)
  Feature engineering: automatic (crossing layers learn interactions)
  AUC: 0.795 (+1% over LightGBM → millions in revenue)
  
  Best for: Final ranking (accurate, only top 50 candidates)

Two-stage production setup:
  Stage 1: LightGBM scores 200 ad candidates in 1ms → top 50
  Stage 2: DCN-v2 re-ranks top 50 in 5ms → final ranking
  Total: 6ms (within 10ms budget)
```

### Negative Downsampling: Critical for Class Imbalance

```
Problem: CTR ~1% → 99% negative samples
  Training on full data: model sees 99 negatives per positive
  Gradient dominated by negatives → model predicts 0 for everything

Solution: Downsample negatives
  Keep ALL positives (clicks)
  Keep 1-in-10 negatives (random sample)
  Effective ratio: 1:10 instead of 1:99
  
  BUT: Model now thinks CTR = 1/(1+10) = 9.1% instead of 1%
  Calibration correction:
    p_calibrated = p_model × sampling_rate / (p_model × sampling_rate + (1 - p_model))
    If p_model = 0.091, sampling_rate = 0.1:
    p_calibrated = 0.091 × 0.1 / (0.091 × 0.1 + 0.909) = 0.0099 ≈ 1% ✓

  Aggressive downsampling (1:3 ratio):
    ✓ 33× less training data → 33× faster training
    ✗ Losing rare negative patterns (unusual non-clicks)
    ✗ Higher variance in model predictions

  Conservative downsampling (1:30 ratio):
    ✓ More negative diversity preserved
    ✗ Training still slow, minor speedup
  
  Industry standard: 1:10 to 1:20 ratio
```

### Online Learning vs Daily Batch Retraining

```
Daily Batch Retraining:
  Train on yesterday's data → deploy today
  Data freshness: 24-48 hours behind
  ✓ Stable: well-tested before deploy
  ✓ Simple: standard ML pipeline
  ✗ Slow to adapt: new ad campaign → stale predictions for 1 day
  ✗ Weekend/holiday patterns missed until Monday retrain

Online Learning (incremental updates):
  Mini-batch updates every 1 hour on last hour's data
  Data freshness: 1-2 hours behind
  ✓ Adapts to trends within hours (Black Friday surge)
  ✓ New ad creatives get accurate predictions faster
  ✗ Risk of catastrophic forgetting (model forgets old patterns)
  ✗ Concept drift: model chases noise if update rate too high
  ✗ Engineering complexity: need streaming training infrastructure

Hybrid (recommended):
  Base model: daily batch retrain (foundation, stable)
  Delta model: hourly online updates (captures recent trends)
  Final score = 0.7 × base_score + 0.3 × delta_score
  Weekly: merge delta into base → fresh start for delta
  
  If delta model degrades → coefficient automatically reduces to 0
  → Graceful fallback to base model
```

---

## 11. Why This Problem Matters

```
At Google/Meta scale:
  0.1% CTR improvement → $100M+ annual revenue increase
  10ms latency increase → 0.5% fewer ads served → $50M loss
  
This is where ML meets systems design:
  Feature store design, model serving infra, real-time pipelines,
  A/B testing, calibration — all in one problem
```

