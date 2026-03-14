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

### A/B Testing — How to Validate Model Changes

```
Every recommendation model change MUST be A/B tested before full rollout.

Setup:
  Control group (50%): current production model (v14)
  Treatment group (50%): new candidate model (v15)
  
  Users deterministically assigned: hash(user_id) % 100 < 50 → control
  Assignment is STICKY (same user always in same group for experiment duration)

Key metrics:
  1. Primary: Watch time / session (did users watch MORE content?)
  2. Secondary: CTR (click-through rate on recommendations)
  3. Guardrail: Retention (did users come back next day?)
  4. Counter-metric: Diversity (did we accidentally create filter bubble?)
  
Duration: Minimum 7 days (to capture weekly seasonality)
Statistical significance: p < 0.05, minimum detectable effect = 0.5%

Infrastructure:
  Experiment config service → decides which model version serves each user
  Feature flags: {experiment_id: "rec_v15", model_version: "v15", traffic: 50%}
  Metrics pipeline: Flink aggregates per-experiment engagement metrics
  Dashboard: Real-time experiment results (like Optimizely/LaunchDarkly)

Common pitfall: "Engagement trap"
  Model optimizes for clicks → shows clickbait → high CTR but low satisfaction
  Solution: Optimize for COMPLETION rate, not just clicks
  "Did the user actually enjoy the content?" > "Did the user click?"
```

### Model Serving Infrastructure — GPU Inference at Scale

```
Ranking 5K candidates per request requires DNN inference.
At 100K requests/sec → 500M (user, item) pair scores per second.

Architecture:
  Model: TensorFlow SavedModel / PyTorch TorchScript
  Serving: TensorFlow Serving / Triton Inference Server
  Hardware: GPU instances (NVIDIA A100) for batch inference
  
  Batching ⭐: Don't score one (user, item) pair at a time.
    Batch 5K items into ONE forward pass through the DNN.
    GPU parallelism: 5K scores in ~50ms (vs 5K × 1ms = 5s sequential)
  
  Model loading: Keep model in GPU memory (hot), reload on new version deploy
  Canary deploy: New model version serves 1% traffic for 1 hour
    If metrics OK → ramp to 10% → 50% → 100%
    If metrics degrade → auto-rollback to previous version
  
  Fallback: If GPU serving is down → fallback to CPU-based simple model
    (e.g., popularity-based ranking, no personalization)
    Degraded but functional > completely broken

  Latency budget:
    Feature fetch: 20ms (Redis)
    Candidate generation: 30ms (ANN search)
    Ranking inference: 50ms (GPU batch)
    Re-ranking: 10ms (business rules)
    Total: ~110ms (well within 200ms target)
```

### Real-Time Feature Pipeline — Closing the Feedback Loop

```
User watches a horror movie at 9 PM. 
  At 9:01 PM, recommendations should start showing more horror.
  But the batch pipeline only updates every 30 minutes!

Real-time feature update path:
  User action → Kafka (user-events) → Flink →
    1. Update user embedding: append latest item to watch history
       → recompute user embedding (lightweight, < 5ms)
       → write to Redis: user:{uid}:embedding
    2. Update engagement counters:
       INCR user:{uid}:genre_watch_count:horror
    3. Update session-level features:
       LPUSH user:{uid}:session_history {item_id}
  
  Next recommendation request (at 9:01 PM):
    Read updated embedding → ANN search returns horror-adjacent candidates
    Ranking model uses updated genre counts → horror movies score higher
    → User sees horror recommendations within 60 seconds of watching

  This is the "real-time feature" approach — no model retraining needed.
  The same model, fed with updated features, produces different results.

Feature freshness tiers:
  Real-time (< 1 min): last watched, session history, time of day
  Near-real-time (< 30 min): updated embeddings, genre preferences
  Batch (daily): user demographics, long-term preferences, social graph
```

### Feedback Loops and Position Bias — The Hidden Dangers

```
Problem 1: Position bias
  Items shown at position 1 get 10× more clicks than position 5.
  Model trained on click data → learns "position 1 items are good"
  → Ranks same items at top forever → no exploration of new content.
  
  Solution: Remove position feature during training.
  At training time: the model should learn ITEM quality, not position.
  At serving time: items ranked by quality, position is just display order.

Problem 2: Popularity feedback loop
  Popular items → recommended more → more engagement → even more popular
  New/niche items → never recommended → no engagement → stay buried
  
  Solution: Exploration budget (10% random, underexposed items)
  + Position de-biasing: Train model with inverse propensity weighting
    Weight each training example by 1/P(shown) 
    → Under-exposed items get higher weight → model learns their true quality

Problem 3: "Rabbit hole" effect
  User watches one conspiracy theory video → algorithm feeds more → radicalization
  
  Solution: Content diversity enforcement in re-ranking
    Max 3 videos from same category in a row
    After 5 videos in same category → inject content from other categories
    Hard cap: no single category > 40% of any user's feed
```

