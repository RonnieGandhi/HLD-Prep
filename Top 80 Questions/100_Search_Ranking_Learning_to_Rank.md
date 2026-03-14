# 100. Design a Search Ranking System (Learning to Rank)

---

## 1. Functional Requirements (FR)

- Given a search query, retrieve candidate documents and rank them by relevance
- Multi-stage ranking pipeline: retrieval → coarse ranking → fine ranking → re-ranking
- Feature extraction: query features, document features, query-document interaction features
- Support multiple ranking objectives: relevance, freshness, diversity, personalization
- Online A/B testing of ranking models
- Click-through feedback loop: use user clicks to improve ranking
- Support for different verticals: web, images, videos, news, shopping

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Full ranking pipeline < 200ms (retrieval to final result page)
- **High Throughput**: 100K+ queries/sec
- **Freshness**: Model updated daily; index updated in near-real-time
- **Quality**: NDCG@10 as primary offline metric; click-through rate online

---

## 3. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────┐
│  MULTI-STAGE RANKING PIPELINE                                     │
│                                                                   │
│  Query: "best noise cancelling headphones"                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐         │
│  │  Stage 1: RETRIEVAL (< 50ms)                        │         │
│  │  Candidates: 10M → 10,000                           │         │
│  │                                                      │         │
│  │  Methods (combined):                                  │         │
│  │  - Inverted index (BM25): keyword match              │         │
│  │  - Embedding retrieval (ANN): semantic match          │         │
│  │    Query embedding → FAISS/ScaNN → top-K similar docs│         │
│  │  - Boolean filters: in-stock, price range, category  │         │
│  │                                                      │         │
│  │  Key insight: SPEED over ACCURACY                    │         │
│  │  Use simple, fast models to cast a wide net          │         │
│  └──────────────────────┬──────────────────────────────┘         │
│                         │ 10,000 candidates                       │
│  ┌──────────────────────▼──────────────────────────────┐         │
│  │  Stage 2: COARSE RANKING (< 50ms)                    │         │
│  │  Candidates: 10,000 → 500                            │         │
│  │                                                      │         │
│  │  Model: Lightweight (logistic regression or small    │         │
│  │         gradient boosted tree)                       │         │
│  │  Features (~20): BM25 score, query-doc embedding     │         │
│  │    similarity, document popularity, freshness         │         │
│  │  Inference: batched, < 5µs per document              │         │
│  └──────────────────────┬──────────────────────────────┘         │
│                         │ 500 candidates                          │
│  ┌──────────────────────▼──────────────────────────────┐         │
│  │  Stage 3: FINE RANKING (< 100ms)                     │         │
│  │  Candidates: 500 → 50                                │         │
│  │                                                      │         │
│  │  Model: Heavy (deep neural network, BERT-based)     │         │
│  │  Features (~200):                                    │         │
│  │    Query: intent, entity recognition, query length   │         │
│  │    Document: title quality, content depth, authority  │         │
│  │    Interaction: BERT(query, doc_title) cross-encoder │         │
│  │    User: personalization (search history, location)  │         │
│  │    Context: time of day, device, locale              │         │
│  │                                                      │         │
│  │  LTR Models:                                         │         │
│  │  - Pointwise: predict relevance score per doc       │         │
│  │  - Pairwise: predict which doc is better (RankNet)  │         │
│  │  - Listwise: optimize NDCG directly (LambdaMART,    │         │
│  │    LambdaRank) ← BEST for search ranking            │         │
│  └──────────────────────┬──────────────────────────────┘         │
│                         │ 50 candidates                           │
│  ┌──────────────────────▼──────────────────────────────┐         │
│  │  Stage 4: RE-RANKING & BLENDING (< 20ms)             │         │
│  │  Final: 50 → 10 (page 1 results)                    │         │
│  │                                                      │         │
│  │  - Diversity: MMR (Maximal Marginal Relevance)       │         │
│  │    Don't show 10 results from same domain            │         │
│  │  - Freshness boost: recent news/articles ranked up   │         │
│  │  - Business rules: sponsored results, legal removal  │         │
│  │  - De-duplication: near-duplicate detection (SimHash)│         │
│  │  - NSFW/safety filtering                             │         │
│  └─────────────────────────────────────────────────────┘         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Learning to Rank (LTR) Deep Dive

```
Pointwise: Treat ranking as regression/classification
  Input: (query, document) → Output: relevance score (0-4)
  Loss: MSE or cross-entropy
  Problem: Doesn't optimize for ranking order directly

Pairwise: Predict which of two documents is more relevant
  Input: (query, doc_A, doc_B) → Output: P(A better than B)
  Loss: hinge loss or cross-entropy on pairs
  Example: RankNet, RankSVM
  Better than pointwise: learns relative ordering

Listwise: Optimize ranking metric directly
  Input: (query, [doc_1, doc_2, ..., doc_n]) → Output: ranked list
  Loss: approximate NDCG/MAP gradient
  Example: LambdaMART (gradient boosted trees with lambda gradients)
  Best for search: directly optimizes what you measure

LambdaMART (industry standard):
  1. Compute "lambda gradients" that push relevant docs up, irrelevant down
  2. Lambda = change in NDCG if two documents swapped
  3. Train gradient boosted tree using these lambdas as gradients
  4. Used by: Bing, Yahoo, many production search engines
```

### Relevance Labels & Training Data

```
Sources of training data:

1. Human judgments (gold standard, expensive):
   - Raters judge (query, document) pairs on 0-4 scale
   - 0: irrelevant, 1: slightly relevant, 2: relevant, 3: highly relevant, 4: perfect
   - Expensive: $5-10 per judgment, need ~5 raters per pair

2. Click data (cheap, biased):
   - Click = positive signal (but position-biased!)
   - Skip = weak negative signal
   - Long dwell time after click = strong positive
   - Quick back-click = negative (pogo-sticking)
   
   De-biasing techniques:
   - Position bias correction (clicks on pos 1 don't mean pos 1 is best)
   - Inverse propensity scoring (IPS)
   - Randomized experiments (show random ordering to measure true relevance)

3. Engagement metrics (implicit feedback):
   - Time on page, scroll depth, share, bookmark
   - Weighted combination → pseudo-relevance label
```

---

## 4. Key Features for Search Ranking

```
Query Features:
  - Query length (tokens)
  - Query intent (navigational / informational / transactional)
  - Named entity type (person, product, location)
  - Query frequency (popular vs rare)
  - Spell-corrected query

Document Features:
  - PageRank / domain authority
  - Content freshness (last updated)
  - Document length, reading level
  - Historical CTR for this document
  - Spam score

Query-Document Interaction Features:
  - BM25 score (title, body, anchor text)
  - TF-IDF cosine similarity
  - BERT cross-encoder score (query × title)
  - Query terms in title/URL/headings (binary)
  - Embedding cosine similarity (bi-encoder)

User Features (personalization):
  - Search history (recent queries)
  - Click history (preferred domains)
  - Location (for local results)
  - Language preference
  - Device type
```

---

## 5. Evaluation Metrics

```
Offline Metrics:
  NDCG@K (Normalized Discounted Cumulative Gain):
    - Measures quality of ranking at top K positions
    - Accounts for graded relevance (0-4) and position discount
    - NDCG@10 is THE primary metric for search ranking
  
  MAP (Mean Average Precision):
    - For binary relevance (relevant / not relevant)
    - Average precision across all relevant documents
  
  MRR (Mean Reciprocal Rank):
    - Position of FIRST relevant result
    - Good for navigational queries (one right answer)

Online Metrics:
  - CTR@1 (click-through rate on position 1)
  - Abandonment rate (searches with no clicks)
  - Time-to-first-click
  - Pogo-sticking rate (click → quick back → different click)
  - Sessions per search (fewer = better results)
```

---

## 6. Fault Tolerance

```
- Model failure: fall back to BM25 ranking (keyword-only, no ML)
- Feature store unavailable: serve with default/missing features (accuracy degrades gracefully)
- Latency spike: skip fine ranking stage, serve coarse ranking results
  (500 candidates with lightweight scoring > timeout with no results)
- Index failure: serve from replica index; rebuild from source data
```

---

## 7. Additional Considerations

### Semantic Search vs Keyword Search

```
Keyword (BM25): "noise cancelling headphones" → must contain these words
  ✅ Exact match, fast, interpretable
  ❌ Misses synonyms ("ANC earbuds"), context

Semantic (Embedding): query → vector → ANN search over document vectors
  ✅ Understands meaning, handles synonyms, multilingual
  ❌ Slower, needs training data, opaque

Hybrid (Best practice):
  score = α × BM25_score + β × embedding_similarity
  Tune α, β with LTR model — both are features for the ranker
```

### Why This Matters in Interviews

```
Search ranking demonstrates understanding of:
  1. ML systems design (training + serving)
  2. Latency budgets and multi-stage pipelines
  3. Feature engineering and feature stores
  4. Evaluation metrics (NDCG, MAP)
  5. Online/offline evaluation gap
  6. Scale: must score 10K+ documents in < 200ms
  
Commonly asked at: Google, Microsoft (Bing), Amazon, Pinterest, LinkedIn
```

