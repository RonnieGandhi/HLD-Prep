# 98. Design an ML Feature Store

---

## 1. Functional Requirements (FR)

- Register feature definitions (name, type, entity, description, owner, SLA)
- Ingest features from batch (Spark/Hive daily) and streaming (Flink/Kafka real-time) sources
- Serve features at low latency for online inference (< 5ms p99)
- Serve features in batch for model training (point-in-time correct joins)
- Feature versioning: track schema changes, backward compatibility
- Feature sharing: discover and reuse features across teams
- Point-in-time correctness: training data must reflect features AS THEY WERE at prediction time (no data leakage)
- Feature monitoring: drift detection, freshness alerts, missing value alerts
- Feature lineage: trace from raw data source → transformation → feature

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency (Online)**: < 5ms p99 for feature retrieval during inference
- **High Throughput (Online)**: 1M+ feature lookups/sec
- **Training Correctness**: Point-in-time joins must be exact (no future data leakage)
- **Freshness**: Online features reflect latest state (< 1 minute for streaming features)
- **Consistency**: Online and offline features must compute identically (no training-serving skew)
- **Scalability**: 10K+ feature definitions, 1B+ entity rows

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Feature definitions | 10,000 |
| Entities (users, items, etc.) | 1B |
| Features per entity | 50-200 |
| Online feature retrievals / sec | 1M |
| Feature vector size per entity | ~2 KB (200 features × 10 bytes avg) |
| Online store size | 1B entities × 2 KB = 2 TB (fits in Redis cluster) |
| Offline store size | 2 TB × 365 days × versions = ~100 TB (S3/Parquet) |
| Batch ingestion (daily) | 1B entities × 2 KB = 2 TB |
| Streaming ingestion | 100K events/sec → feature updates |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    DATA SOURCES                                        │
│                                                                        │
│  Batch: Data Warehouse (Hive/BigQuery/Snowflake) → Spark jobs         │
│  Streaming: Kafka topics → Flink/Spark Streaming                      │
│  Request-time: computed on-the-fly during inference (e.g., time of day)│
└───────────────────────┬───────────────────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────────────────┐
│                TRANSFORMATION LAYER                                    │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Feature Transformation Engine                             │        │
│  │                                                            │        │
│  │  SAME CODE runs for both batch and streaming:              │        │
│  │  (This is the key to avoiding training-serving skew)       │        │
│  │                                                            │        │
│  │  @feature(entity="user", name="user_avg_purchase_30d")     │        │
│  │  def avg_purchase_30d(transactions):                       │        │
│  │      return transactions                                   │        │
│  │          .filter(last_30_days)                             │        │
│  │          .agg(avg("amount"))                               │        │
│  │                                                            │        │
│  │  Batch mode: Spark processes full history daily            │        │
│  │  Streaming mode: Flink maintains sliding window            │        │
│  │  Both produce IDENTICAL results for same input             │        │
│  └──────────────┬─────────────────────┬───────────────────────┘        │
│                 │                     │                                 │
│        (materialized to)     (materialized to)                        │
│                 │                     │                                 │
└─────────────────│─────────────────────│────────────────────────────────┘
                  │                     │
┌─────────────────│─────────────────────│────────────────────────────────┐
│          STORAGE LAYER                │                                 │
│                 │                     │                                 │
│  ┌──────────────▼──────────┐  ┌──────▼──────────────────────┐         │
│  │  OFFLINE STORE          │  │  ONLINE STORE               │         │
│  │  (S3 / Parquet / Delta) │  │  (Redis / DynamoDB)         │         │
│  │                         │  │                              │         │
│  │  For training:          │  │  For serving:                │         │
│  │  - Historical feature   │  │  - Latest feature values     │         │
│  │    values with timestamps│  │  - Key: entity_id           │         │
│  │  - Partitioned by date  │  │  - Value: feature vector     │         │
│  │  - Enables point-in-time│  │  - TTL for freshness         │         │
│  │    correct joins        │  │  - Sub-5ms retrieval         │         │
│  │  - Columnar format for  │  │                              │         │
│  │    efficient scans      │  │  Key design:                 │         │
│  │                         │  │  HSET user:123               │         │
│  │  Schema:                │  │    avg_purchase_30d "45.20"  │         │
│  │  entity_id | timestamp  │  │    click_rate_7d "0.032"     │         │
│  │  | feature_1 | feat_2.. │  │    last_login_hours "2.5"    │         │
│  └─────────────────────────┘  └──────────────────────────────┘         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                SERVING LAYER                                           │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Feature Serving API                                       │        │
│  │                                                            │        │
│  │  Online: GET /features/user/123?features=avg_purchase,ctr  │        │
│  │    → Redis lookup → return feature vector in < 5ms         │        │
│  │    → Batch multiple entities: POST with entity_id list     │        │
│  │                                                            │        │
│  │  Training: get_historical_features(entity_df, features,    │        │
│  │                                     timestamps)            │        │
│  │    → Point-in-time join against offline store               │        │
│  │    → Returns: training dataset with correct historical      │        │
│  │      feature values (no future leakage)                    │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ┌───────────────��────────────────────────────────────────────┐        │
│  │  Feature Registry (Metadata Service)                       │        │
│  │                                                            │        │
│  │  PostgreSQL:                                               │        │
│  │  - Feature name, type, entity, description                 │        │
│  │  - Owner team, SLA, freshness requirement                  │        │
│  │  - Transformation code (versioned)                         │        │
│  │  - Data source lineage                                     │        │
│  │  - Schema version history                                  │        │
│  │  - Access control (who can read/write)                     │        │
│  │                                                            │        │
│  │  Discovery UI:                                              │        │
│  │  - Search features by name, entity, description            │        │
│  │  - View feature stats (distribution, null rate, freshness) │        │
│  │  - "Add to my model" workflow                              │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                MONITORING LAYER                                        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Feature Monitoring                                        │        │
│  │                                                            │        │
│  │  Freshness: Is the feature up to date?                    │        │
│  │  - Track last_updated_at per feature per entity            │        │
│  │  - Alert if stale > SLA (e.g., > 1 hour for streaming)    │        │
│  │                                                            │        │
│  │  Drift: Has feature distribution changed?                 │        │
│  │  - Compare current distribution vs training distribution   │        │
│  │  - KL divergence, PSI (Population Stability Index)         │        │
│  │  - Alert if drift > threshold → model may need retraining │        │
│  │                                                            │        │
│  │  Quality: Are features correct?                           │        │
│  │  - Null rate, out-of-range values, type mismatches         │        │
│  │  - Compare online vs offline values (should match)         │        │
│  │  - Training-serving skew detection                         │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 5. APIs

```
# Feature definition
POST /api/features
{
  "name": "user_avg_purchase_30d",
  "entity": "user",
  "value_type": "FLOAT",
  "description": "Average purchase amount in last 30 days",
  "source": "transactions_table",
  "transformation": "avg(amount) WHERE timestamp > now() - 30d",
  "freshness_sla": "1h",
  "owner": "ml-team"
}

# Online serving
GET /api/features/online
  ?entity=user
  &entity_id=123,456,789
  &features=avg_purchase_30d,click_rate_7d,last_login_hours

Response:
{
  "features": [
    {"entity_id": "123", "avg_purchase_30d": 45.20, "click_rate_7d": 0.032, "last_login_hours": 2.5},
    {"entity_id": "456", "avg_purchase_30d": 12.80, "click_rate_7d": 0.015, "last_login_hours": 48.0}
  ]
}

# Training data generation (batch)
POST /api/features/historical
{
  "entity": "user",
  "entity_ids_with_timestamps": [
    {"entity_id": "123", "timestamp": "2026-03-01T10:00:00Z"},
    {"entity_id": "456", "timestamp": "2026-03-05T14:00:00Z"}
  ],
  "features": ["avg_purchase_30d", "click_rate_7d"]
}
→ Returns: feature values AS THEY WERE at each timestamp (point-in-time correct)
```

---

## 6. Data Models

### Online Store (Redis)

```
# Key: {entity_type}:{entity_id}
# Value: Hash of feature_name → feature_value

HSET user:123
  avg_purchase_30d "45.20"
  click_rate_7d "0.032"
  last_login_hours "2.5"
  favorite_category "electronics"
  account_age_days "730"
  _updated_at "2026-03-14T10:05:00Z"

# TTL per entity (stale data auto-expires)
EXPIRE user:123 86400  # 24h TTL
```

### Offline Store (S3 / Delta Lake / Parquet)

```
Path: s3://feature-store/user/avg_purchase_30d/

Partitioned by date:
  date=2026-03-14/
    part-00000.parquet
    part-00001.parquet

Schema:
  entity_id:   STRING
  timestamp:   TIMESTAMP
  value:       FLOAT
  created_at:  TIMESTAMP

Point-in-time join:
  For training example at (user_123, 2026-03-10T14:00):
    Find latest feature value WHERE timestamp <= 2026-03-10T14:00
    → Returns feature from 2026-03-10 batch (not 2026-03-14!)
    This prevents future data leakage
```

### Feature Registry (PostgreSQL)

```sql
CREATE TABLE features (
    feature_id    UUID PRIMARY KEY,
    name          TEXT UNIQUE NOT NULL,
    entity_type   TEXT NOT NULL,     -- user, item, session
    value_type    TEXT NOT NULL,     -- FLOAT, INT, STRING, EMBEDDING
    description   TEXT,
    transformation_code TEXT,        -- versioned transformation logic
    source_table  TEXT,
    freshness_sla INTERVAL,
    owner_team    TEXT,
    schema_version INT DEFAULT 1,
    created_at    TIMESTAMPTZ DEFAULT NOW(),
    deprecated    BOOLEAN DEFAULT FALSE
);

CREATE TABLE feature_usage (
    feature_id   UUID REFERENCES features(feature_id),
    model_id     UUID,
    model_name   TEXT,
    first_used   TIMESTAMPTZ,
    last_used    TIMESTAMPTZ
);
```

---

## 7. Fault Tolerance & Deep Dives

### Training-Serving Skew (The #1 Problem)

```
What it is:
  Features computed differently in training vs serving → model performs worse in production

Example:
  Training: avg_purchase computed in Spark with pandas (Python float64)
  Serving: avg_purchase computed in Java (Java double, different rounding)
  Result: 0.1% difference → model accuracy degrades

Prevention:
  1. Single transformation definition: same code for batch AND streaming
     (Feast, Tecton, and Feathr all enforce this)
  2. Validation job: compute features both ways, compare → alert on divergence
  3. Feature logging: log online features at serving time → use THESE for training
     (guarantees zero skew, but requires logging infrastructure)
```

### Point-in-Time Correctness (Training Data Leakage)

```
Wrong approach:
  Join training data (March 1 prediction) with current features (March 14)
  → Model sees "future" information → performs unrealistically well in training
  → Fails in production (no future data available at prediction time)

Correct approach:
  For training example at time T:
    feature_value = latest(feature WHERE timestamp <= T)
  
  This requires the offline store to maintain historical feature values
  with timestamps — not just the current value

Implementation:
  1. Store features as (entity_id, timestamp, value) triples
  2. Training join: AS OF JOIN on timestamp
  3. Delta Lake / Apache Iceberg support time-travel queries natively
```

### Online Store Failure

```
Redis cluster failure → feature retrieval fails → inference fails

Mitigations:
1. Redis Cluster with replicas (6-node: 3 masters + 3 replicas)
2. Local feature cache on inference servers (LRU, 5-min TTL)
3. Default values: if Redis unavailable, use population median for each feature
   (degrades accuracy, preserves availability)
4. Feature importance: top 10 features provide 80% of model accuracy
   → cache only critical features locally → tolerate missing low-importance features
```

---

## 8. Additional Considerations

### Popular Feature Store Systems

```
| System | Type | Key Trait |
|---|---|---|
| Feast | Open source | Most popular OSS, supports Redis/DynamoDB online, S3/BigQuery offline |
| Tecton | SaaS | Founded by Uber Michelangelo team, strong streaming support |
| Hopsworks | Open source | Feature pipeline as first-class citizen |
| Feathr (LinkedIn) | Open source | Production-proven at LinkedIn scale |
| Vertex AI Feature Store | GCP managed | Integrated with Vertex AI training/serving |
| SageMaker Feature Store | AWS managed | Integrated with SageMaker |
```

### When You Need a Feature Store (vs. Not)

```
DON'T need:
  - < 10 features, 1 model, 1 team → inline feature computation is fine
  - Batch-only ML (no real-time serving) → just use SQL views

DO need:
  - Real-time serving (< 10ms features for online prediction)
  - Multiple teams sharing features (reuse, discovery)
  - Training-serving skew is causing production issues
  - Point-in-time correctness matters (any time-series ML)
  - Feature freshness matters (streaming features)
```

### Embedding Features

```
Modern ML uses embedding vectors (e.g., 128-dim user embedding from a recommendation model)

Storage: Redis doesn't natively handle vectors well
  Option A: Store as serialized blob in Redis (fast, but no vector search)
  Option B: Dedicated vector store (Pinecone, Milvus) for ANN queries
  Option C: Feature store + vector index sidecar

Serving: Return pre-computed embedding as part of feature vector
  → Model uses embedding directly (no online computation needed)
  → Embedding recomputed daily in batch
```

