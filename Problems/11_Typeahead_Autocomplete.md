# 11. Design Google Typeahead / Autocomplete

---

## 1. Functional Requirements (FR)

- As the user types, suggest top 5-10 matching search queries in real-time
- Suggestions ranked by popularity (search frequency)
- Support prefix matching (typing "sys" → "system design", "system architecture")
- Update suggestions based on new/trending search data
- Handle multi-language queries
- Personalized suggestions (based on user's search history — optional)

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: Suggestions must appear within 50-100 ms
- **High Availability**: 99.99%
- **Scalability**: Handle 100K+ autocomplete requests/sec
- **Freshness**: New trending terms appear within minutes to hours
- **Consistency**: Eventual consistency acceptable (slightly stale suggestions OK)
- **Fault Tolerant**: System degrades gracefully (show slightly stale suggestions)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 500M |
| Searches / day | 5B |
| Autocomplete requests / search | 6 (avg 6 keystrokes before selecting) |
| Autocomplete requests / day | 30B |
| Autocomplete requests / sec | ~350K |
| Unique query terms | 1B |
| Avg query length | 20 characters |
| Trie storage (with counts) | ~50 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │  ← Debounce: only send request after 100ms of no typing
└──────┬───────┘
       │
┌──────▼───────┐
│    CDN       │  ← Cache popular prefixes at edge
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │
└──────┬───────┘
       │
┌──────▼───────────────┐
│  Autocomplete        │
│  Service             │  ← In-memory Trie lookup
│  (Stateless Servers) │
└──────┬───────────────┘
       │
┌──────▼──────┐
│  Trie       │  ← Pre-built, periodically refreshed
│  (In-Memory)│
└─────────────┘

════════════════════ Offline Data Pipeline ════════════════════

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Search Logs │     │   Kafka      │     │  Flink /     │
│  (raw query  │────▶│ (search-     │────▶│  Spark       │
│   events)    │     │  events)     │     │ (Aggregate)  │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                          ┌───────▼───────┐
                                          │  Aggregated   │
                                          │  Query DB     │
                                          │ (query→count) │
                                          └───────┬───────┘
                                                  │
                                          ┌───────▼───────┐
                                          │  Trie Builder │
                                          │  (Offline)    │
                                          └───────┬───────┘
                                                  │
                                          ┌───────▼───────┐
                                          │  Trie Blob    │
                                          │  (S3 / HDFS)  │  ← Serialized trie
                                          └───────────────┘
                                                  │
                                         (pulled by autocomplete
                                          servers on refresh)
```

### Component Deep Dive

#### Trie Data Structure — The Core Algorithm

**Standard Trie**:
```
Root
├── s
│   ├── y
│   │   ├── s
│   │   │   ├── t
│   │   │   │   ├── e
│   │   │   │   │   ├── m (★ "system" count=50000)
│   │   │   │   │   │   ├── d → e → s → i → g → n (★ "system design" count=20000)
│   │   │   │   │   │   └── a → r → c → h (★ "system architecture" count=5000)
```

**Optimization 1: Store Top-K results at each node**:
- At each trie node, pre-compute and cache the top 10 most popular completions
- When user types "sys" → go to node 's' → 'y' → 's' → return pre-cached top 10
- Avoids traversing the entire subtree at query time → **O(prefix_length)** lookup

```
Node 'sys':
  top_10: [
    ("system design", 20000),
    ("system architecture", 5000),
    ("system requirements", 3000),
    ...
  ]
```

**Optimization 2: Compressed Trie (Radix Tree / Patricia Trie)**:
- Merge nodes with single children: `s → y → s → t → e → m` becomes `"system"` as one node
- Reduces memory by 50-70%

**Optimization 3: Trie Serialization**:
- Serialize trie as a flat byte array for compact storage and fast loading
- Use memory-mapped files for instant startup

#### Offline Data Pipeline
- **Search Logs**: Every search query logged with timestamp, user_id (anonymized), location
- **Kafka**: Buffers search events
- **Flink/Spark Aggregation**:
  1. Count query frequency in sliding windows (hourly, daily, weekly)
  2. Apply decay function: `score = Σ (count_i × decay^(age_in_hours))`
     - Recent queries weigh more than old ones
  3. Filter: Remove queries with < 5 occurrences (noise), profanity, PII
  4. Output: `{query_string → popularity_score}` to Aggregated Query DB
- **Trie Builder**: 
  1. Read aggregated query-score pairs
  2. Build trie in memory
  3. At each node, compute top-K descendants
  4. Serialize to binary blob → upload to S3
  5. **Frequency**: Rebuild every 15 minutes (trending) or hourly (stable)

#### Autocomplete Service
- **Stateless**: Each server loads the trie blob into memory on startup
- **Trie refresh**: New trie blob published to S3 → servers poll every N minutes → atomically swap old trie with new trie (double-buffering)
- **Lookup**: `trie.search(prefix)` → returns pre-cached top 10 suggestions in O(prefix_length)
- **Scaling**: Horizontally scale by adding servers (each holds a full copy of the trie)
  - For very large tries: shard by first character(s) of the query

#### CDN Caching
- Popular prefixes ("how to", "what is", "why") are cached at CDN edge
- TTL: 1 hour (popularity is relatively stable for common prefixes)
- Cache key: `autocomplete:{prefix}`
- **Impact**: 30-50% of autocomplete requests served from CDN

#### Client-Side Optimizations
- **Debouncing**: Only send request after 100-200 ms of no typing (reduces 70% of requests)
- **Local cache**: Cache previous responses. "syst" → "syste" → reuse results from "syst" client-side
- **Prefetch**: After showing suggestions for "sys", prefetch "syst", "sysa", etc. in background

---

## 5. APIs

### Autocomplete
```http
GET /api/v1/autocomplete?q=system+des&limit=10&lang=en
Response: 200 OK
{
  "suggestions": [
    {"query": "system design", "score": 20000},
    {"query": "system design interview", "score": 15000},
    {"query": "system design primer", "score": 8000},
    {"query": "system design patterns", "score": 5000},
    {"query": "system design basics", "score": 3000}
  ]
}
```

### Log Search (Internal — for data pipeline)
```http
POST /api/v1/search-log (internal)
{
  "query": "system design",
  "user_id": "anon-hash",
  "timestamp": "2026-03-13T10:00:00Z",
  "location": "US"
}
```

---

## 6. Data Model

### In-Memory Trie Node

```java
class TrieNode {
    Map<Character, TrieNode> children;   // or array[26] for ASCII lowercase
    boolean isEnd;                        // marks a complete query
    List<Pair<String, Long>> topK;       // pre-cached top-K suggestions with scores
}
```

### Aggregated Query DB (Cassandra / DynamoDB)

```sql
-- Stores aggregated query frequencies
Key:    query_string (partition key)
Value:  {
          hourly_count: 500,
          daily_count: 5000,
          weekly_count: 25000,
          score: 15234.5,         -- weighted score with decay
          last_updated: timestamp
        }
```

### Kafka Topic: `search-events`

```json
{
  "query": "system design",
  "user_id": "anon-hash-uuid",
  "timestamp": "2026-03-13T10:00:00Z",
  "country": "US",
  "language": "en",
  "result_count": 1250000
}
```

### Trie Blob (Serialized Binary Format)

```
Header: { version, node_count, total_queries, built_at }
Nodes:  [ { char, is_end, child_count, child_offsets[], topK[] } ]
Strings: [ offset → query_string ]
```
- Stored in S3, ~5-50 GB depending on query corpus
- Memory-mapped for fast loading

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Trie server crash** | Stateless; LB routes to healthy instances. New instance loads trie from S3 |
| **Trie build failure** | Servers keep serving old trie; alert ops. Fallback: serve last known good trie |
| **Stale suggestions** | Acceptable — users get suggestions from 15-60 min ago |
| **S3 unavailable** | Trie blob cached locally on each server; survives S3 outage |
| **Data pipeline lag** | Trending topics appear with slight delay; fallback to existing trie data |

### Specific: Trie Hot-Swap Without Downtime
1. New trie blob built and uploaded to S3
2. Each server has two trie slots (A and B) — double buffering
3. Server loads new trie into inactive slot (B) while serving from active slot (A)
4. Atomic pointer swap: active = B. Old trie (A) is garbage collected
5. Zero downtime, zero latency spike during refresh

---

## 8. Additional Considerations

### Handling Trending / Breaking News
- Standard trie rebuild (hourly) is too slow for breaking news
- **Solution**: Maintain a small "trending overlay" trie updated every 5 minutes
- At query time: merge results from main trie + trending trie
- Trending trie is built from Flink real-time stream (last 1-hour window)

### Personalized Suggestions
- Augment global suggestions with user's own search history
- Client stores recent 100 queries locally → searched client-side first
- Merge: Show 2-3 personalized + 7-8 global suggestions
- **Privacy**: Personalized suggestions computed client-side (no server-side history needed)

### Multi-Language Support
- Separate trie per language (detected from user's locale or keyboard input)
- CJK (Chinese, Japanese, Korean): Tokenization differs — use n-gram based approach instead of character trie

### Handling Offensive / Sensitive Queries
- Blocklist of offensive terms — filtered out of suggestions
- PII detection: Don't suggest queries containing email addresses, phone numbers, SSNs
- Legal/DMCA: Remove specific queries upon legal request

### Sampling for Scale
- Don't log/count every single search query
- Sample 1 in 10 (or 1 in 100) queries for frequency estimation
- Multiply counts accordingly — statistically valid for large-scale trends
- Reduces data pipeline cost by 10-100×

### Spell Correction Integration
- If no suggestions found for a prefix, try spelling correction
- Use edit distance (Levenshtein) to find closest matching prefix in trie
- "systm desi" → "system desi" → suggestions for "system design"

