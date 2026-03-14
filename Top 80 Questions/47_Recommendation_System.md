# 47. Design a Recommendation System (Netflix / TikTok Style)

---

## 1. Functional Requirements (FR)

- **Personalized recommendations**: Show content tailored to each user's taste
- **Multiple surfaces**: Home feed, "Because you watched X", "Trending", category pages
- **Real-time signals**: Incorporate recent user actions (watch, like, skip) within minutes
- **Cold start handling**: Recommendations for brand-new users with no history
- **Diversity**: Avoid filter bubbles; expose users to varied content
- **Explainability**: "Because you watched Stranger Things" or "Trending in your area"

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Recommendations served in < 200 ms
- **High Throughput**: Serve 100K+ recommendation requests/sec
- **Freshness**: New content surfaced within hours of upload
- **Scalability**: 500M+ users, 100M+ items (videos/movies/songs)
- **Offline + Online**: Batch training (daily) + real-time feature updates
- **A/B Testable**: Every model change tested via controlled experiments

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 300M |
| Rec requests / sec | 100K |
| Items in catalog | 100M |
| User-item interactions / day | 10B (views, likes, skips) |
| Model size | 10-50 GB (embedding tables) |
| Feature store entries | 300M users + 100M items |
| Candidate generation latency budget | < 50 ms |
| Ranking latency budget | < 100 ms |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Client Request: GET /recommendations             │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  Recommendation API Service                          │
│  (Orchestrates the pipeline, caches results)                         │
└───────┬──────────────────────────────────────────────────────────────┘
        │
        │  Step 1: Fetch user profile + recent activity
        ▼
┌───────────────────────────────────────┐
│         Feature Store (Redis)         │
│                                       │
│  user:{uid}:profile                   │
│  = {embedding_128d, interests,        │
│     country, age_group, language,     │
│     last_50_watched, last_10_liked}   │
│                                       │
│  item:{iid}:features                  │
│  = {embedding_128d, genre, tags,      │
│     avg_completion_rate, like_rate,    │
│     recency_score, popularity}        │
└───────────────────┬───────────────────┘
                    │
        ┌───────────▼───────────┐
        │                       │
        ▼                       ▼
┌───────────────────┐  ┌───────────────────────────────────────────┐
│ Candidate         │  │        Candidate Generation               │
│ Sources           │  │        (Recall Stage — ~50ms)             │
│                   │  │                                           │
│ 1. Collaborative  │  │  Each source returns ~3K candidates:      │
│    Filtering      │  │                                           │
│    (ANN search:   │  │  Source 1: "Users like you watched..."   │
│     user emb →    │  │  → ANN search in item embedding space   │
│     nearest item  │  │  → HNSW index (Faiss/Milvus/Pinecone)   │
│     embeddings)   │  │  → dot_product(user_emb, item_emb)      │
│                   │  │  → 3K items in ~10ms                     │
│ 2. Content-based  │  │                                           │
│    (similar to    │  │  Source 2: "Similar to recently watched" │
│     recently      │  │  → For each of last 5 watched items,     │
│     watched)      │  │    find 200 similar items (ANN search)   │
│                   │  │  → 1K items                              │
│ 3. Popular /      │  │                                           │
│    Trending       │  │  Source 3: "Trending in your region"     │
│    (by region)    │  │  → Pre-computed daily, cached in Redis   │
│                   │  │  → 500 items                             │
│ 4. Editor picks / │  │                                           │
│    curated lists  │  │  Source 4: "New releases in your genres" │
│                   │  │  → 500 items                             │
│ 5. Exploration    │  │                                           │
│    (random from   │  │  Source 5: Random exploration            │
│     underexposed  │  │  → 200 items from low-exposure pool     │
│     items)        │  │                                           │
└───────────────────┘  │  Union + dedup → ~5K unique candidates  │
                       └───────────────────┬───────────────────────┘
                                           │
                                           ▼
                       ┌───────────────────────────────────────────┐
                       │     Ranking Model (Precision — ~100ms)    │
                       │                                           │
                       │  Input per (user, item) pair:             │
                       │    user_features (64 dims)                │
                       │    item_features (64 dims)                │
                       │    context: {time_of_day, device,         │
                       │              day_of_week, session_depth}  │
                       │    cross_features: {user×item interaction │
                       │              history if any}              │
                       │                                           │
                       │  Model: Deep Neural Network (DNN)         │
                       │    or Transformer-based (BERT4Rec)        │
                       │                                           │
                       │  Output:                                  │
                       │    P(watch_complete)  × w1                │
                       │    P(like)            × w2                │
                       │    P(share)           × w3                │
                       │    P(subscribe)       × w4                │
                       │    Score = weighted sum                    │
                       │                                           │
                       │  GPU inference on TensorFlow Serving       │
                       │  Batch: score 5K items in one forward pass│
                       │  → Top 500 ranked items                   │
                       └───────────────────┬───────────────────────┘
                                           │
                                           ▼
                       ┌───────────────────────────────────────────┐
                       │     Re-Ranking (Business Rules — ~10ms)   │
                       │                                           │
                       │  1. De-duplicate creators (max 2 per)     │
                       │  2. Genre/category diversity              │
                       │     (not all horror, not all comedy)      │
                       │  3. Freshness boost (new items get        │
                       │     guaranteed exposure in top 50)        │
                       │  4. Already-watched filter                │
                       │     (Redis SET watched:{uid})             │
                       │  5. Brand safety / content policy         │
                       │  6. Sponsored content insertion (ad slots)│
                       │  7. A/B test variant selection             │
                       │                                           │
                       │  → Final 100 items for this session       │
                       └───────────────────┬───────────────────────┘
                                           │
                                           ▼
                       ┌───────────────────────────────────────────┐
                       │     Response Cache (Redis)                │
                       │  feed:{uid} = [item_ids] (TTL: 30 min)  │
                       │  Return first 20 to client                │
                       │  Subsequent pages: pop next 20 from cache │
                       └───────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                    Offline Training Pipeline                         │
│                                                                       │
│  Kafka (user-events) → Spark ETL → Feature Engineering               │
│  → Training Data (S3 Parquet) → Model Training (GPU cluster)        │
│  → Model Validation (offline metrics: NDCG, recall@K)               │
│  → Model Registry → Canary Deploy → TF Serving                      │
│                                                                       │
│  Daily: retrain embedding model (collaborative filtering)            │
│  Weekly: retrain ranking DNN                                         │
│  Real-time: update user features in Redis (via Flink from Kafka)    │
└──────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Cold Start — New Users and New Items

```
New user (no watch history):
  1. Demographics-based: recommend popular items for age group + region
  2. Onboarding survey: "What genres do you like?" → seed preferences
  3. Exploration-heavy: show diverse content, learn preferences from first 10 interactions
  4. Bandits (explore/exploit): Multi-Armed Bandit selects between diverse candidates
     Track which genres/types the user engages with → personalize within hours

New item (no engagement data):
  1. Content-based features: genre, tags, description, cast → predict similar items
  2. Creator signal: if creator's past items performed well → boost new item
  3. Guaranteed exposure: every new item gets shown to N users (e.g., 1000)
     Collect engagement data → model learns true quality → organic ranking kicks in
  4. Freshness boost in re-ranking: new items get score multiplier for first 48 hours
```

#### Embedding Generation — Two-Tower Model

```
Tower 1 (User encoder):
  Input: user watch history (sequence of item_ids), demographics, preferences
  Output: user_embedding (128-dim float vector)
  Architecture: Transformer encoder over watch sequence
  
Tower 2 (Item encoder):
  Input: item metadata (genre, tags, description), engagement stats
  Output: item_embedding (128-dim float vector)
  Architecture: MLP over concatenated features
  
Training: Contrastive learning
  Positive pairs: (user, items they watched to completion)
  Negative pairs: (user, random items they didn't watch)
  Loss: maximize dot_product for positives, minimize for negatives
  
Inference:
  User embedding: computed once per session (~1ms)
  Item embeddings: pre-computed for all 100M items
  ANN search: dot_product(user_emb, item_embs) → top 3K in ~10ms
  Index: HNSW (Hierarchical Navigable Small World) in Faiss/Milvus
```

---

## 5. APIs

### Get Recommendations
```http
GET /api/v1/recommendations?surface=home&count=20&cursor=0
→ 200 OK
{
  "items": [
    { "item_id": "movie-123", "title": "...", "score": 0.95,
      "reason": "Because you watched Stranger Things",
      "thumbnail": "https://cdn.../thumb.jpg" },
    ...
  ],
  "cursor": "20"
}
```

### Record User Event (for real-time feature updates)
```http
POST /api/v1/events
{ "user_id": "u1", "item_id": "movie-123", "event": "watch_complete", 
  "duration_sec": 3600, "timestamp": "..." }
→ 202 Accepted
```

---

## 6. Data Model

### Feature Store (Redis)
```
user:{uid}:embedding     → Binary (512 bytes = 128 floats × 4 bytes)
user:{uid}:history       → List (last 100 item_ids)
user:{uid}:profile       → Hash {country, age_group, language, signup_date}
item:{iid}:embedding     → Binary (512 bytes)
item:{iid}:stats         → Hash {completion_rate, like_rate, view_count}
watched:{uid}            → Set (item_ids watched, TTL: 30 days)
feed:{uid}               → List (pre-computed recommendations, TTL: 30 min)
```

### Model Artifacts (S3)
```
s3://models/collaborative-filtering/v42/
  ├── user_embeddings.npy     (300M × 128 × 4 = ~150 GB)
  ├── item_embeddings.npy     (100M × 128 × 4 = ~50 GB)
  └── model_metadata.json

s3://models/ranking-dnn/v15/
  ├── saved_model.pb
  └── model_config.json
```

### Vector Index (Faiss/Milvus)
```
HNSW index over 100M item embeddings (128-dim)
  Build time: ~2 hours on GPU
  Search time: ~5ms for top-3K nearest neighbors
  Memory: ~55 GB (embeddings + graph structure)
  Update: nightly rebuild or real-time incremental insert for new items
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Model serving failure** | Fallback to previous model version (blue-green deploy) |
| **Feature store down** | Serve from cached feed; degrade to popularity-based recs |
| **ANN index stale** | Rebuild nightly; new items get guaranteed exposure via exploration |
| **Training data poisoning** | Filter bot/spam interactions before training; outlier detection |
| **Cold start** | Demographics + onboarding survey + exploration; no empty feed ever |
| **Feedback loop (filter bubble)** | 10% of recommendations are random exploration items |

### Race Conditions

#### 1. Feature Staleness — User Just Watched a Horror Movie

```
User watches "The Conjuring" at T=0
Feature store updated at T=30s (Flink pipeline latency)
User refreshes feed at T=5s → features still show old interests

Result: Recommendations don't reflect the horror movie yet

Mitigations:
  1. Client-side context: Pass last_watched_item_id in request headers
     Ranking model uses this as real-time context feature (no pipeline delay)
  2. Session-level re-ranking: Boost items similar to recently watched
     Done in re-ranking stage, no model inference needed
  3. Accept 30-second delay: for most users, this is invisible
```

#### 2. Popularity Bias — Rich Get Richer

```
Popular items get shown more → get more clicks → get recommended more
Niche quality content never gets exposure → never gets clicks → stays buried

Solution: Exploration budget
  90% of feed: model-scored recommendations (exploitation)
  10% of feed: random items from underexposed pool (exploration)
  
  Underexposed pool: items with < 1000 views AND < 48 hours old
  After 1000 views: enough data to determine quality → organic ranking
  
  Thompson Sampling: Select exploration items using Bayesian bandits
    Each item has a Beta distribution of (successes, failures)
    Sample from each distribution → highest sample gets shown
    Balances exploration of uncertain items vs exploitation of known good ones
```

---

## 8. Deep Dive: Engineering Trade-offs

### Batch vs Real-Time Recommendations

```
Batch (pre-computed):
  Background job: for each active user, compute top 200 recs
  Store in Redis: feed:{uid} = [item_ids]
  ✓ Serves in < 10 ms (just Redis read)
  ✓ Model can be complex (minutes of compute per user)
  ✗ Stale: doesn't reflect user's last 30 min of activity
  ✗ Compute cost: 300M users × 30 min intervals = expensive

Real-time (on-demand):
  Each request → full pipeline: candidates → rank → re-rank
  ✓ Always fresh, reflects current context
  ✗ Latency: 200-500 ms
  ✗ Compute spike at peak hours

Hybrid ⭐ (Netflix/TikTok approach):
  Batch pre-computes candidate set (top 500) — updated every 30 min
  Real-time ranking re-scores candidates using latest features — per request
  Re-ranking applies business rules — per request
  
  Total latency: ~150 ms (no candidate generation, just ranking + re-ranking)
  Freshness: ranking uses real-time features (last watched, time of day)
```

### Collaborative Filtering vs Content-Based vs Hybrid

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Collaborative** | "Users like you liked X" | Discovers unexpected interests | Cold start; popularity bias |
| **Content-Based** | "Similar to what you liked" | No cold start for items; transparent | Limited serendipity; feature engineering |
| **Knowledge Graph** | Item relationships (actor, genre, director) | Rich semantics | Expensive to build; sparse |
| **Hybrid** ⭐ | Multiple candidate sources, unified ranker | Best of all worlds | System complexity |

**Netflix uses Hybrid**: Multiple candidate generators (collaborative, content, trending, editorial) → single DNN ranker → business rules re-ranker.

