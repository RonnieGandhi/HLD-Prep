# 64. Design a Video Recommendation Engine

---

## 1. Functional Requirements (FR)

- **Personalized recommendations**: "Videos you might like" feed tailored to each user's watch history, likes, and preferences
- **"Up Next" recommendation**: After watching a video, suggest what to watch next (autoplay)
- **Homepage feed**: Curated mix of trending, personalized, and fresh content
- **Similar videos**: "Because you watched X" — find videos related to a specific video
- **Category/topic recommendations**: "Trending in Technology", "Popular in Music"
- **New user cold start**: Recommend popular/trending content for users with no history
- **Explain recommendations**: "Recommended because you watched System Design Interview"
- **Feedback loop**: Incorporate explicit (like/dislike) and implicit (watch time, skip) signals
- **Diversity**: Avoid filter bubble; include exploratory recommendations
- **Real-time updates**: Recommendation changes within minutes of new user behavior

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Recommendations served in < 200 ms
- **Scalability**: 1B+ users, 500M+ videos, 5B+ watches/day
- **Freshness**: Newly uploaded videos appear in recommendations within 1 hour
- **Quality**: Increase average watch time per session (the north star metric)
- **Diversity**: No more than 30% of recommendations from same creator/category
- **Fairness**: New creators' content gets a fair chance (exploration vs exploitation)
- **Availability**: 99.99% — recommendations are the main UI; failure = blank homepage

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 500M |
| Videos in catalog | 500M |
| Watches / day | 5B |
| Recommendation requests / sec | 200K (homepage loads, up-next) |
| User feature vector size | 256 floats = 1 KB |
| Video feature vector size | 256 floats = 1 KB |
| User features total | 1B × 1 KB = 1 TB |
| Video features total | 500M × 1 KB = 500 GB |
| Model inference latency budget | < 50 ms |
| Pre-computed recs cache | 500M users × 100 recs × 8B = 400 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────┐
│                    Recommendation System                      │
│                                                                │
│   ┌─────────────────────────────────────────────────────┐     │
│   │              Online Serving Layer                    │     │
│   │                                                       │     │
│   │  ┌───────────────┐    ┌───────────────┐              │     │
│   │  │  Candidate    │    │  Ranking      │              │     │
│   │  │  Generation   │───▶│  Model        │──▶ Results   │     │
│   │  │  (1000 → 100) │    │  (100 → 20)   │              │     │
│   │  └───────┬───────┘    └───────┬───────┘              │     │
│   │          │                     │                      │     │
│   │  ┌───────▼───────┐    ┌───────▼───────┐              │     │
│   │  │  Feature      │    │  Post-Rank    │              │     │
│   │  │  Store        │    │  Filters      │              │     │
│   │  │  (Redis)      │    │  (diversity,  │              │     │
│   │  │               │    │   freshness,  │              │     │
│   │  │  User + Video │    │   dedup)      │              │     │
│   │  │  features     │    │               │              │     │
│   │  └───────────────┘    └───────────────┘              │     │
│   └─────────────────────────────────────────────────────┘     │
│                                                                │
│   ┌─────────────────────────────────────────────────────┐     │
│   │              Offline Training Pipeline               │     │
│   │                                                       │     │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │     │
│   │  │  Watch   │  │  Feature │  │  Model   │           │     │
│   │  │  Events  │→ │  Engin-  │→ │  Training│           │     │
│   │  │  (Kafka  │  │  eering  │  │  (Spark +│           │     │
│   │  │   → S3)  │  │  (Spark) │  │   PyTorch│           │     │
│   │  │          │  │          │  │   / TF)  │           │     │
│   │  └──────────┘  └──────────┘  └──────────┘           │     │
│   │                                                       │     │
│   │  ┌──────────┐  ┌──────────┐                          │     │
│   │  │ Embedding│  │ ANN      │                          │     │
│   │  │ Genera-  │→ │ Index    │                          │     │
│   │  │ tion     │  │ (Faiss/  │                          │     │
│   │  │ (user +  │  │  Milvus) │                          │     │
│   │  │  video)  │  │          │                          │     │
│   │  └──────────┘  └──────────┘                          │     │
│   └─────────────────────────────────────────────────────┘     │
│                                                                │
│   ┌─────────────────────────────────────────────────────┐     │
│   │              Near-Real-Time Update Pipeline          │     │
│   │                                                       │     │
│   │  User watches video → Kafka → Flink →                │     │
│   │  Update user features in Redis (< 5 min latency)     │     │
│   │                                                       │     │
│   └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘

External Services:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Redis       │  │  Faiss/      │  │  ClickHouse  │  │  S3          │
│ (Feature     │  │  Milvus      │  │ (Training    │  │ (Model       │
│  Store,      │  │ (ANN Index   │  │  data,       │  │  artifacts,  │
│  Cached      │  │  for embed-  │  │  metrics)    │  │  embeddings) │
│  recs)       │  │  ding search)│  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Component Deep Dive

#### Two-Stage Architecture: Candidate Generation → Ranking

```
Why two stages? Because scoring ALL 500M videos for every user is impossible.

Stage 1: Candidate Generation (500M → 1000 candidates, < 10 ms)
  "Find ~1000 videos that MIGHT be relevant to this user"
  
  Uses: lightweight models + index lookups
  
  Sources (each produces candidates, merged):
  
  a. Collaborative Filtering (ALS matrix factorization):
     User embedding × Video embedding → similarity score
     ANN search: find 200 videos with embeddings closest to user embedding
     Captures: "users like you watched these videos"
     
  b. Content-Based:
     User's interest vector (topic preferences) × Video topic vector
     ANN search: find 200 videos with similar topics to user's interests
     Captures: "you like System Design → here are more SD videos"
  
  c. Recent interactions:
     User watched video X → find 200 similar videos (item-item similarity)
     Precomputed: similar_videos:{video_id} → top 100 similar video_ids
     Captures: "because you just watched X"
  
  d. Trending/Popular:
     Top 200 trending videos (updated every 15 min)
     Top 200 in user's preferred categories
     Captures: cold-start users, diverse/fresh content
  
  e. Subscriptions:
     Recent uploads from channels user subscribes to
     Captures: explicit user intent
  
  Merge: union of all sources → ~1000 unique candidates
  Dedup: remove already watched videos

Stage 2: Ranking Model (1000 → 20 videos, < 30 ms)
  "Score each candidate with a precise model. Return top 20."
  
  Model: Deep neural network (DNN) or gradient boosted trees
  
  Input features per (user, candidate_video) pair:
    User features: [age_bucket, country, language, device, time_of_day, 
                     watch_history_embedding, search_history_embedding,
                     preferred_categories, avg_watch_duration, 
                     activity_level, days_since_signup]
    
    Video features: [title_embedding, category, duration, upload_date,
                      channel_popularity, view_count, like_ratio,
                      thumbnail_ctr, avg_watch_percentage, 
                      content_quality_score]
    
    Interaction features: [user_watched_this_channel_before,
                            user_liked_similar_category,
                            time_since_video_uploaded,
                            geographic_relevance]
  
  Output: predicted engagement metrics
    P(click) — probability user will click
    P(watch) — probability user will watch significant portion
    E[watch_time] — expected watch time
    P(like) — probability of explicit positive feedback
  
  Combined score:
    score = w1 × P(click) × E[watch_time] + w2 × P(like) - w3 × P(dislike)
    
    Why not just P(click)?
      Clickbait: high P(click) but low watch_time = bad recommendation
      Optimizing for watch_time rewards genuinely engaging content
    
    YouTube's actual objective: maximize user session time
      score = Σ(expected_watch_time_per_video × P(watching_that_video))
```

#### Embedding-Based Retrieval — How ANN Search Works

```
User embeddings and video embeddings live in the same vector space (256-dim).
Similarity = dot product or cosine similarity.

Generating embeddings:

  User Embedding (Two-Tower Model):
    User tower: 
      Input: user_id one-hot → 64-dim user ID embedding
             + average of last 50 watched video embeddings
             + demographics features
             + temporal features
      Layers: 3 dense layers (512 → 256 → 256)
      Output: 256-dim user embedding
    
    Video tower:
      Input: video_id one-hot → 64-dim video ID embedding
             + title word2vec embedding (avg pooled)
             + category one-hot
             + channel embedding
             + video features (duration, popularity)
      Layers: 3 dense layers (512 → 256 → 256)
      Output: 256-dim video embedding
    
    Training objective:
      For (user, positive_video, negative_video) triplets:
      Maximize: dot(user_emb, positive_video_emb) - dot(user_emb, negative_video_emb)
      
      Positive: videos user watched for > 50% duration
      Negative: random videos user was shown but didn't click

ANN (Approximate Nearest Neighbor) Index:
  Problem: Finding top-200 from 500M vectors by dot product is O(500M × 256)
  
  Solution: FAISS (Facebook AI Similarity Search) or Milvus

  FAISS options:
    1. IVF (Inverted File Index):
       Cluster 500M vectors into 10,000 clusters (k-means)
       For query: find closest 50 clusters → search only those
       Reduces search space from 500M to ~2.5M
       Time: ~5 ms for top-200
    
    2. HNSW (Hierarchical Navigable Small World):
       Graph-based index, each node connected to neighbors
       Navigate graph from random entry point to query's neighborhood
       Time: ~1 ms for top-200
       Memory: 2× the vector data (~1 TB for 500M vectors)
    
    3. IVF + PQ (Product Quantization):
       Compress 256-dim vectors to 64 bytes (4× compression)
       IVF clustering + search compressed vectors
       Time: ~2 ms, Memory: 32 GB (fits in RAM!)
  
  For our scale:
    500M videos × 256 × 4 bytes = 512 GB (raw embeddings)
    With IVF-PQ: 500M × 64 bytes = 32 GB → fits in single machine!
    
    Use FAISS IVF-PQ for candidate generation:
      32 GB index → serves in < 5 ms per query → 200K QPS feasible
      Deploy 10 replicas for availability and load distribution
```

#### Near-Real-Time Feature Updates

```
User watches a video at 3:00 PM. By 3:05 PM, recommendations reflect this.

Pipeline:
  1. User finishes video → client sends watch event to Kafka
     { user_id, video_id, watch_duration, total_duration, liked, timestamp }
  
  2. Flink streaming job consumes events:
     a. Update user's recent watch history (last 50 videos):
        LPUSH user_watch_history:{user_id} video_id
        LTRIM user_watch_history:{user_id} 0 49
     
     b. Update user's interest vector:
        new_interest = 0.9 × old_interest + 0.1 × watched_video_category_vector
        Exponential moving average → recent interests weighted higher
     
     c. Update user embedding (lightweight update):
        user_emb = average(embeddings of last 50 watched videos)
        HSET user_features:{user_id} embedding {binary_vector}
     
     d. Update video stats:
        INCR video_stats:{video_id}:views
        HSET video_features:{video_id} avg_watch_pct {new_avg}
  
  3. Next recommendation request (< 5 minutes later):
     - Uses updated user embedding for candidate generation
     - Uses updated user features for ranking
     - Recommendations reflect the video just watched

Full model retraining (daily):
  Spark job: extract all watch events from last 30 days
  Train two-tower model on GPU cluster (4-8 hours)
  Generate new embeddings for ALL users and videos
  Build new FAISS index
  Swap: old model → new model (blue-green deployment)
  
  Why daily retrain if real-time updates exist?
    Real-time updates: lightweight approximations (moving averages)
    Full retrain: learns NEW patterns, corrects drift, incorporates new videos
    Both are needed: real-time for responsiveness, daily retrain for accuracy
```

#### Post-Ranking Filters — Ensuring Quality and Diversity

```
After ranking model outputs top 100 scored candidates:

1. Diversity filter:
   No more than 3 videos from same channel
   No more than 30% from same category
   Algorithm: Maximal Marginal Relevance (MMR)
     For each position i in the final list:
       next_video = argmax_v [ λ × score(v) - (1-λ) × max_j(similarity(v, selected_j)) ]
     λ = 0.7 (70% relevance, 30% diversity)
   
   Result: diverse recommendations without sacrificing too much relevance

2. Freshness boost:
   Videos uploaded in last 24 hours: score × 1.2
   Videos uploaded in last 7 days: score × 1.1
   Ensures new content gets exposure (important for creators)

3. Quality filter:
   Remove: videos with < 40% average watch percentage (likely clickbait)
   Remove: videos flagged by content moderation
   Remove: videos with high dislike ratio (> 30%)

4. Explored vs Exploited:
   90% of recommendations: exploitation (high predicted score)
   10% of recommendations: exploration (random high-quality videos)
   
   Why exploration?
   - Discover user interests model hasn't learned yet
   - Give new videos/creators a chance
   - Prevent filter bubble
   
   Epsilon-greedy: with probability ε=0.1, replace one recommendation with random explore

5. Business rules:
   Inject 1 ad-supported video per 10 recommendations
   Boost premium/original content if user is subscriber
   Suppress videos from blocked creators

6. Deduplication:
   Remove videos user already watched
   Remove videos too similar to recently watched (pHash similarity)
   Remove same video in different quality/reupload
```

---

## 5. APIs

### Get Home Feed Recommendations
```http
GET /api/v1/recommendations/home?limit=20&cursor={last}
Headers: X-User-Id: user-uuid, X-Context: { "device": "mobile", "time_zone": "America/New_York" }
Response: 200 OK
{
  "recommendations": [
    {
      "video_id": "vid-uuid",
      "title": "System Design: URL Shortener",
      "channel": "TechChannel",
      "thumbnail": "https://cdn.example.com/thumb/vid-uuid.webp",
      "duration": 1200,
      "view_count": 1200000,
      "published_at": "2025-03-10",
      "score": 0.95,
      "reason": "Because you watched 'System Design Interview Guide'",
      "source": "collaborative_filtering"
    }
  ]
}
```

### Get "Up Next" Recommendation
```http
GET /api/v1/recommendations/up-next?current_video={video_id}&watch_time=600&total_duration=1200
Response: 200 OK
{
  "up_next": {
    "video_id": "vid-next",
    "title": "System Design: API Rate Limiter",
    "reason": "Next in series",
    "autoplay_in_seconds": 5
  },
  "related": [...]
}
```

### Get Similar Videos
```http
GET /api/v1/recommendations/similar/{video_id}?limit=10
Response: 200 OK
{
  "similar": [
    {"video_id": "...", "title": "...", "similarity_score": 0.89, "reason": "Similar topic"}
  ]
}
```

### Send Feedback Signal
```http
POST /api/v1/recommendations/feedback
{
  "video_id": "vid-uuid",
  "action": "not_interested",
  "reason": "already_watched"
}
Response: 200 OK
```

---

## 6. Data Model

### Redis — Feature Store + Cached Recommendations

```
# User features (updated in near-real-time by Flink)
user_features:{user_id}  → Hash {
  embedding: binary(1024),           # 256 × 4 bytes
  preferred_categories: "tech,science,education",
  avg_watch_duration: 480,
  activity_level: "high",
  country: "US",
  language: "en",
  last_updated: 1710400000
}
TTL: none (always fresh via streaming updates)

# User's recent watch history (for real-time signal)
user_watch_history:{user_id}  → List [video_id_1, video_id_2, ..., video_id_50]

# Video features
video_features:{video_id}  → Hash {
  embedding: binary(1024),
  category: "technology",
  channel_id: "ch-uuid",
  duration: 1200,
  view_count: 1200000,
  like_ratio: 0.95,
  avg_watch_pct: 0.72,
  upload_date: "2025-03-10"
}
TTL: 86400

# Pre-computed similar videos (item-item)
similar:{video_id}  → List [vid_1, vid_2, ..., vid_100]
TTL: 86400

# Cached recommendations per user
recs:{user_id}:{context}  → List [vid_1, vid_2, ..., vid_100]
TTL: 300 (5 minutes — balance freshness vs compute)

# Trending videos (global + per category)
trending:global          → Sorted Set { video_id: trend_score }
trending:{category}      → Sorted Set { video_id: trend_score }
TTL: 900
```

### FAISS / Milvus — Vector Index

```
Index: user_to_video_retrieval
  Vectors: 500M video embeddings (256-dim, float32)
  Index type: IVF4096,PQ64  (4096 clusters, PQ compression to 64 bytes)
  Memory: ~32 GB
  Query: given user embedding → top-200 nearest video embeddings
  Latency: < 5 ms
  
  Deployment: 10 replicas behind load balancer
  Update: rebuild daily from new embeddings; hot-swap via blue-green

Index: video_to_video_similarity
  Same 500M embeddings
  Query: given video embedding → top-100 most similar videos
  Used for: "Similar videos" and "Up Next"
  Pre-computed: materialize top 100 similar per video → store in Redis
  Update: daily batch job
```

### ClickHouse — Training Data + Metrics

```sql
CREATE TABLE watch_events (
    user_id         UUID,
    video_id        UUID,
    watch_duration  UInt32,
    total_duration  UInt32,
    watch_pct       Float32,
    liked           Int8,            -- 1=like, -1=dislike, 0=neutral
    source          String,          -- "home", "search", "up_next", "similar"
    position        UInt8,           -- position in recommendation list
    platform        String,
    country         FixedString(2),
    event_date      Date MATERIALIZED toDate(timestamp),
    timestamp       DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (user_id, timestamp);

-- Training query: all user-video interactions for model training
-- Metric query: "what's the CTR at each recommendation position?"
-- A/B test query: "does model V2 increase watch_time vs V1?"
```

### Kafka Topics

```
Topic: watch-events          (user watch behavior → training data + real-time features)
Topic: recommendation-served  (what was shown → for CTR computation)
Topic: recommendation-clicked (what was clicked → for CTR computation)
Topic: video-published       (new video → needs embedding generation)
Topic: model-events          (model deployed, A/B test started/ended)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Feature store (Redis) down** | Serve from pre-computed cached recommendations; degrade to trending/popular |
| **FAISS index unavailable** | Fall back to pre-computed similar videos + trending; skip ANN retrieval |
| **Ranking model error** | Circuit breaker → serve candidates by score from candidate generation (skip ranking) |
| **Cold user (no history)** | Trending + popular + category-based recommendations |
| **Cold video (just uploaded)** | Use content-based features (title, description, category) for initial embedding |
| **Embedding drift** | Daily retrain corrects drift; monitor embedding quality metrics |
| **A/B test regression** | Auto-rollback if treatment decreases avg watch time by > 2% |
| **Thundering herd (homepage)** | Pre-compute recommendations for active users every 5 minutes |

### Specific: Cold Start for New Users

```
New user with zero watch history — what to recommend?

Onboarding signal:
  1. Ask interests during signup: "Select topics you're interested in"
     [Technology, Music, Gaming, Cooking, Sports, ...]
     → Initialize user interest vector from selected categories
  
  2. First session behavior:
     Watch history updates in real-time (Flink)
     After 3 videos → personalized candidates start appearing
     After 10 videos → good personalization
     After 30 videos → converged

Fallback (no onboarding data):
  Based on context signals:
    Country → popular videos in that country
    Device language → videos in that language
    Time of day → appropriate content (news in morning, entertainment at night)
    Referral source → if came from Google search about "Python tutorial" → tech content

Progressive personalization:
  Session 1: 80% popular/trending + 20% random explore
  Session 2: 60% popular + 30% based on session 1 + 10% explore
  Session 5: 40% popular + 50% personalized + 10% explore
  Session 10+: 10% popular + 80% personalized + 10% explore
```

### Specific: New Video Cold Start

```
Video uploaded 5 minutes ago. No watch data. How to recommend it?

1. Content-based embedding:
   Generate embedding from video metadata (title, description, tags, category)
   NLP model: BERT or sentence-transformers → 256-dim text embedding
   This is the "cold" video embedding — purely content-based

2. Visual embedding (optional):
   Extract frames → run through CNN (ResNet) → visual embedding
   Captures: visual style, content type (animation, talking head, outdoors)

3. Channel prior:
   If channel has 100K subscribers → video likely to be good
   Inherit channel's average engagement metrics as initial priors
   
   cold_video_score = channel_avg_ctr × channel_subscriber_count / normalization

4. Exploration allocation:
   Reserve 10% of recommendation slots for videos < 24 hours old
   New video gets shown to small audience → collect engagement data
   If engagement is good → model learns quickly → video enters regular recommendations
   
   Multi-armed bandit: Thompson Sampling
     Each new video = a bandit arm
     Prior: Beta(1, 1) (uniform — no opinion)
     Each impression: observe click/no-click → update posterior
     After 1000 impressions: posterior converges → we know the video's quality
     
     If high CTR → promote into main recommendations
     If low CTR → stop recommending (save impressions for better content)

5. Embedding warm-up:
   After 100 watches → retrain video embedding using collaborative signals
   After 1000 watches → video embedding is well-calibrated
   Intermediate: blend content embedding (cold) with collaborative embedding (warm)
     blended_emb = α × content_emb + (1-α) × collab_emb
     α starts at 1.0 (pure content), decreases to 0.2 as watch data accumulates
```

---

## 8. Deep Dive: Engineering Trade-offs

### Two-Tower vs Cross-Network: Model Architecture Choice

```
Two-Tower Model (used for candidate generation):
  User tower: input user features → user embedding
  Video tower: input video features → video embedding
  Score: dot_product(user_emb, video_emb)
  
  ✓ Offline computation: video embeddings computed once, stored
  ✓ ANN-searchable: user embedding as query → find nearest video embeddings
  ✓ Scales to billions of videos (ANN index)
  ✗ Limited interaction modeling (no cross-features between user and video)
  
  Use for: Stage 1 candidate generation (speed matters, 500M candidates)

Cross-Network (DCN / DeepFM / Wide&Deep):
  Input: concatenated user features + video features + cross-features
  Learns: complex interactions between user and video attributes
  Example cross-feature: user_prefers_short_videos × video_is_short → high score
  
  ✓ More accurate: models feature interactions
  ✓ Can use hundreds of features
  ✗ Cannot pre-compute: each (user, video) pair needs forward pass
  ✗ 1000 candidates × 1 forward pass each → 1000 inferences per request
  ✗ Can't do ANN search (score depends on both user AND video)
  
  Use for: Stage 2 ranking (accuracy matters, only 1000 candidates)

YouTube's actual architecture (published paper):
  Stage 1: Two-tower model → candidate retrieval from 500M videos
  Stage 2: Wide & Deep model → rank 1000 candidates → return top 20
  
  The two-stage approach is THE standard in industry.
  Netflix, TikTok, Instagram, Pinterest all use variations of this.
```

### Optimizing for Watch Time vs Click-Through Rate

```
Problem: What objective should the recommendation model optimize?

CTR (Click-Through Rate):
  Maximize P(user clicks on video)
  ✗ Clickbait wins: sensational thumbnails get clicks but low watch time
  ✗ User satisfaction decreases (tricked into clicking)
  ✗ Session time doesn't increase (users leave after short watch)

Watch Time:
  Maximize E[watch_time] per recommendation
  ✓ Rewards genuinely engaging content
  ✓ Session time increases (users stay on platform longer)
  ✗ Bias toward long videos (60-min video with 50% completion = 30 min > 
     5-min video with 100% completion = 5 min)
  ✗ Doesn't account for user satisfaction

Watch Time × Watch Percentage ⭐:
  Maximize E[watch_time × watch_percentage]
  Balances: long engaging videos AND short fully-watched videos
  A 5-min video watched to completion (score: 5 × 1.0 = 5) can score
  similarly to a 10-min video watched 50% (score: 10 × 0.5 = 5)

YouTube's actual objective (evolved over time):
  2012: Optimize for clicks (CTR) → clickbait problem exploded
  2016: Optimize for watch time → long video bias
  2019+: Multi-objective optimization:
    score = w1 × E[watch_time]
          + w2 × P(like)
          + w3 × P(share)
          - w4 × P(dislike)
          - w5 × P(clickbait_flag)
          + w6 × user_satisfaction_survey_prediction
  
  Weights tuned via A/B testing on long-term user retention
```

### Model Serving: Batch Pre-Computation vs Real-Time Inference

```
Batch pre-computation:
  Every 15 minutes: for each active user, compute top 100 recommendations
  Store in Redis: recs:{user_id} → [vid_1, vid_2, ..., vid_100]
  On request: read from Redis (< 1 ms)
  
  ✓ Ultra-fast serving (Redis read)
  ✓ No inference compute on request path
  ✗ Stale: up to 15 minutes old
  ✗ Massive compute: 500M users × every 15 min = 2B inference jobs/hour
  ✗ Most pre-computed recs never served (user doesn't open app)

Real-time inference:
  On request: run full pipeline (candidate gen → ranking → filters)
  ✓ Fresh: reflects user's latest behavior
  ✓ Efficient: only compute for active users
  ✗ Latency: 50-200 ms per request (needs fast infrastructure)
  ✗ Requires high-throughput model serving (200K QPS)

Hybrid approach ⭐ (YouTube/Netflix):
  1. Pre-compute "base" recommendations every 4 hours
     Store: recs_base:{user_id} → top 200 candidates
     Covers: offline users who open app → instant recommendations
  
  2. Real-time personalization on request:
     Read base candidates from Redis
     Fetch fresh user features (updated by Flink in real-time)
     Re-rank base candidates with latest user features (fast: 200 candidates, not 500M)
     Apply post-rank filters
     
     Latency: Redis read (1 ms) + re-rank 200 candidates (10 ms) + filters (2 ms) = 13 ms
  
  3. If base recommendations are stale (> 4 hours or user behavior changed significantly):
     Trigger full pipeline (candidate gen + ranking) → update cache
     User sees cached results while fresh recs compute in background
     
  Result: < 15 ms serving latency with near-real-time personalization
```

### Filter Bubble Prevention: Exploration Strategies

```
Without exploration: user watches tech videos → model shows only tech videos
  → user never discovers they might enjoy cooking videos
  → engagement plateaus → user eventually gets bored and churns

Exploration strategies:

1. ε-greedy: Replace 10% of recs with random high-quality content
   Simple but wastes explore budget on truly irrelevant content

2. Thompson Sampling ⭐:
   Model uncertainty: for each candidate, estimate P(engagement) AND uncertainty
   High uncertainty + decent expected score → prioritize (explore)
   Result: intelligent exploration of videos model is uncertain about

3. Contextual Bandits:
   Learn a policy: given user context → choose between exploit and explore
   Policy optimizes long-term engagement (not just immediate session)
   Can learn: "explore cooking content for tech users on weekends" (context-dependent)

4. Interest expansion:
   Track user's explicit interest graph
   Suggest content at the boundary of known interests:
     User likes: [Python, System Design, AWS]
     Boundary topics: [DevOps, Machine Learning, Rust]
     NOT: [Cooking, Fashion, Sports] (too far from interests)
   
   Gradual expansion: show ML content → if engaged → show more → expand further

5. Diversity constraints (hard rules):
   Max 3 videos from same channel
   Max 30% from same category
   At least 2 videos uploaded in last 24 hours
   At least 1 video from a new (< 1000 subscriber) channel

Measuring exploration effectiveness:
  Metric: "interest breadth" = number of distinct categories watched per week
  A/B test: model with exploration vs without
  Good exploration: interest breadth increases WITHOUT decreasing session time
  Bad exploration: interest breadth increases BUT session time drops (showing irrelevant content)
```



