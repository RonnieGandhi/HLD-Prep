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

```
- Model serving failure → fall back to simpler model (logistic regression) or last-known-good
- Feature store unavailable → use default features (reduces accuracy, not availability)
- Stale features → TTL enforcement, degrade gracefully
- Training failure → don't deploy; keep serving current model
- Canary deployment: new model serves 5% traffic first → monitor AUC, calibration → promote or rollback
```

---

## 9. Why This Problem Matters

```
At Google/Meta scale:
  0.1% CTR improvement → $100M+ annual revenue increase
  10ms latency increase → 0.5% fewer ads served → $50M loss
  
This is where ML meets systems design:
  Feature store design, model serving infra, real-time pipelines,
  A/B testing, calibration — all in one problem
```

