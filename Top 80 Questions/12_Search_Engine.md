# 12. Design a Search Engine (Google)

---

## 1. Functional Requirements (FR)

- Accept a text query and return a ranked list of relevant web pages
- Support keyword matching, phrase matching, and boolean queries
- Rank results by relevance (PageRank + text relevance + freshness + personalization)
- Display result snippets (title, URL, description with highlighted keywords)
- Support autocomplete/typeahead (see #11)
- Support image, video, and news search (vertical search)
- Spell correction: "systm desgn" → "Did you mean: system design?"
- Knowledge panels for entities (people, places, companies)
- Pagination of results

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Results in < 500 ms (ideally < 200 ms)
- **High Availability**: 99.99%
- **Scalability**: 100B+ indexed pages, 10B+ queries/day
- **Freshness**: New/updated pages indexed within minutes (for news) to hours
- **Relevance**: Results must be useful and accurate (quality is king)
- **Spam Resistant**: Resist SEO manipulation and web spam

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Indexed pages | 100B |
| Avg page size (compressed) | 100 KB |
| Raw index size | 100B × 100 KB = 10 PB |
| Inverted index size | ~10% of raw = **1 PB** |
| Queries / day | 10B |
| Queries / sec | ~115K (peak ~500K) |
| Index servers (1TB per server) | ~1,000 index servers |
| Crawl rate | 1B pages/day |

---

## 4. High-Level Design (HLD)

```
                              ┌──────────────┐
                              │   User       │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │ Query Service│  ← Parse, spell-check, understand intent
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Index       │  ← Distributed inverted index
                              │  Service     │
                              │  (Sharded)   │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Ranking     │  ← PageRank + TF-IDF + ML
                              │  Service     │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Snippet     │  ← Generate result snippets
                              │  Generator   │
                              └──────────────┘

═══════════════ Offline: Crawl & Index Pipeline ═══════════════

┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Web     │──▶│  URL     │──▶│  Crawler │──▶│  Content │
│  (WWW)   │   │ Frontier │   │  Workers │   │  Store   │
└──────────┘   └──────────┘   └──────────┘   │ (GFS/S3) │
                                              └────┬─────┘
                                                   │
                                    ┌──────────────┼──────────────┐
                                    │              │              │
                              ┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐
                              │ Parser / │  │ PageRank │  │ Index    │
                              │ Extractor│  │ Computer │  │ Builder  │
                              └──────────┘  └──────────┘  └────┬─────┘
                                                               │
                                                        ┌──────▼──────┐
                                                        │  Inverted   │
                                                        │  Index      │
                                                        │  (Sharded)  │
                                                        └─────────────┘
```

### Component Deep Dive

#### Query Service (Query Understanding)
- **Query parsing**: Tokenize, lowercase, remove stop words, stem/lemmatize
- **Spell correction**: Edit distance + language model. "systm" → candidate corrections → pick highest probability correction
- **Query expansion**: "NYC restaurants" → also search "New York City restaurants"
- **Intent detection**: Is this a navigational query ("facebook login"), informational ("how to cook pasta"), or transactional ("buy iPhone 15")?
- **Synonym handling**: "car" → also search "automobile", "vehicle"

#### Inverted Index — Core Data Structure

An inverted index maps **terms → list of documents containing that term**.

```
"system"     → [doc1:3, doc5:1, doc15:7, doc22:2, ...]   (doc_id:term_frequency)
"design"     → [doc1:2, doc3:5, doc15:4, ...]
"interview"  → [doc1:1, doc15:3, doc42:2, ...]
```

**Index Structure per term (posting list)**:
```
{
  term: "system",
  document_frequency: 50000000,     // how many docs contain this term
  posting_list: [
    {doc_id: 1, tf: 3, positions: [15, 42, 78]},
    {doc_id: 5, tf: 1, positions: [201]},
    ...
  ]
}
```

**Index Sharding**:
- **Term-based sharding**: Each shard holds a subset of terms (e.g., A-M, N-Z)
  - Con: Query for "system design" hits two shards → inter-shard communication
- **Document-based sharding** ⭐: Each shard holds all terms for a subset of documents
  - Query is broadcast to ALL shards → each shard returns local top-K → merge results
  - Google uses this approach (called "index tiers")

**Index Compression**:
- Posting lists are compressed using **Variable-Byte Encoding** or **PForDelta**
- Doc IDs are delta-encoded: [1, 5, 15, 22] → [1, 4, 10, 7] (smaller values compress better)
- Reduces index size by 5-10×

#### Ranking Service — The Scoring Pipeline

**Stage 1: Initial Retrieval** (cheap, coarse)
- Boolean matching: Find documents containing ALL query terms (AND query)
- Use inverted index to intersect posting lists efficiently
- Return candidate set (e.g., top 10,000 documents)

**Stage 2: Scoring** (TF-IDF / BM25)

**BM25 (Best Matching 25)** — industry standard text relevance:
```
score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D|/avgdl))

Where:
  qi = ith query term
  f(qi, D) = term frequency of qi in document D
  |D| = document length
  avgdl = average document length across corpus
  k1 = 1.2 (saturation parameter)
  b = 0.75 (length normalization)
  IDF(qi) = log((N - n(qi) + 0.5) / (n(qi) + 0.5))
  N = total documents, n(qi) = documents containing qi
```

**Stage 3: PageRank (Link Analysis)**

PageRank measures page importance based on incoming links:
```
PR(A) = (1 - d) + d × Σ (PR(Ti) / C(Ti))

Where:
  d = damping factor (0.85)
  Ti = pages linking to A
  C(Ti) = number of outgoing links from Ti
```
- Computed offline via iterative MapReduce (20-50 iterations until convergence)
- Stored per document, used as a static quality signal

**Stage 4: ML Re-ranking** (expensive, precise)
- Learning-to-Rank model (LambdaMART, neural models)
- Features: BM25 score, PageRank, click-through rate, freshness, domain authority, user location
- Re-ranks top 1,000 candidates from previous stages
- Returns top 10 for display

#### Snippet Generator
- For each result, generate a relevant snippet showing query terms in context
- Find the passage in the document with highest query term density
- Highlight matching terms in bold
- Truncate to ~160 characters

---

## 5. APIs

### Search
```http
GET /api/v1/search?q=system+design+interview&page=1&lang=en&country=US
Response: 200 OK
{
  "query": "system design interview",
  "spell_correction": null,
  "results_count": 125000000,
  "results": [
    {
      "title": "System Design Interview Guide - ByteByteGo",
      "url": "https://bytebytego.com/system-design",
      "snippet": "A comprehensive guide to <b>system design interview</b> preparation...",
      "favicon": "https://bytebytego.com/favicon.ico",
      "cached_url": "...",
      "rank": 1
    },
    ...
  ],
  "related_searches": ["system design interview questions", "system design primer"],
  "knowledge_panel": null,
  "pagination": {"page": 1, "total_pages": 100}
}
```

---

## 6. Data Model

### Inverted Index (SSTable-like format on disk)

```
Term Dictionary (in-memory):
  "system" → {offset: 0x4A2F, doc_freq: 50000000}
  "design" → {offset: 0x8B1C, doc_freq: 35000000}

Posting List (on disk, compressed):
  Offset 0x4A2F:
    [doc_id_delta: 1, tf: 3, positions: [15, 42, 78]]
    [doc_id_delta: 4, tf: 1, positions: [201]]
    ...
```

### Document Store (Bigtable / GFS)

```
Row Key: doc_id (hash of URL)
Column Families:
  content:   {title, body_text, meta_description, language}
  metadata:  {url, domain, crawl_date, content_hash, robots_directives}
  links:     {outgoing_urls[], incoming_count}
  scores:    {pagerank, spam_score, domain_authority}
```

### URL Frontier (for Crawler — see #13)

```
Priority Queue:
  {url, priority, last_crawled, crawl_interval, domain}
```

### PageRank Store

```
Key:    doc_id
Value:  {pagerank_score: 0.00042, last_computed: timestamp}
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Index server failure** | Each index shard replicated 3×; query routed to healthy replica |
| **Index corruption** | Checksum verification; rebuild from content store |
| **Query overload** | Circuit breaker + graceful degradation (skip ML re-ranking, serve BM25 only) |
| **Stale index** | Incremental index updates; full rebuild weekly |
| **Crawler politeness** | Respect robots.txt, rate limit per domain (see #13) |

### Specific: Index Serving Architecture
- **Index Tiers**: 
  - Tier 0: Most important pages (top 1B) — always searched
  - Tier 1: Important pages (top 10B) — searched if Tier 0 insufficient
  - Tier 2: Long tail (100B) — searched only for rare queries
- Tiering reduces latency: most queries are answered by Tier 0 alone

---

## 8. Additional Considerations

### Index Update Strategy
- **Full rebuild**: MapReduce job rebuilds entire index weekly (handles deletions, re-scoring)
- **Incremental updates**: Real-time pipeline adds new/updated pages to a "supplement index"
- Query searches both main index + supplement index; results merged
- Supplement index is periodically merged into main index

### Anti-Spam / Web Spam Detection
- **Link spam**: Detect link farms (clusters of pages linking to each other artificially)
- **Content spam**: Keyword stuffing, hidden text, cloaking detection
- **Click spam**: Click-through rate manipulation detection
- **Signals**: Domain age, SSL certificate, content uniqueness, link velocity

### Query Result Caching
- Cache top results for popular queries in Redis/Memcached
- 25% of queries are repeated within an hour → high cache hit ratio
- Cache key: normalized query + location + language
- TTL: 1 hour for most queries, 5 min for news queries

### Semantic Search
- Beyond keyword matching: understand query meaning
- **BERT / Transformer models**: Encode query and documents into embedding vectors
- **Vector similarity search** (ANN: Approximate Nearest Neighbor) using FAISS or ScaNN
- Hybrid: BM25 keyword matching + semantic similarity → combined score

