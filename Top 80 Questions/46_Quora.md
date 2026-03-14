# 46. Design Quora (Q&A Platform)

---

## 1. Functional Requirements (FR)

- **Ask questions**: Users post questions (with topics/tags)
- **Answer questions**: Users write answers; multiple answers per question
- **Upvote/Downvote**: Vote on answers (and questions) to surface quality
- **Follow topics/questions**: Get notified when new answers are posted
- **Feed**: Personalized home feed of questions and answers from followed topics/users
- **Search**: Full-text search across questions and answers
- **Spaces** (communities): Topic-based groups with moderation
- **Request answers**: Request a specific user to answer a question
- **Editing**: Collaborative editing with revision history
- **Content moderation**: Detect spam, hate speech, low-quality content

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Feed and search in < 500 ms
- **SEO Friendly**: Questions/answers are the core value — must be crawlable and indexable
- **Scalability**: 300M+ MAU, 500M+ questions
- **Quality Ranking**: Best answers surfaced at top (not just newest)
- **Availability**: 99.99%
- **Write Volume**: Moderate (10K answers/min) but read-heavy (1M reads/min)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| DAU | 100M |
| Questions asked / day | 500K |
| Answers posted / day | 2M |
| Votes / day | 50M |
| Feed views / sec | ~30K |
| Search queries / sec | ~10K |
| Avg answer size | 2 KB |
| Total content storage | 1 TB questions + 4 TB answers |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Clients                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │
│  │ Web (SSR)│  │ Mobile   │  │ Google   │  ← SEO: Server-side     │
│  │          │  │ App      │  │ Crawler  │    rendered HTML for     │
│  │          │  │          │  │          │    search indexing        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                          │
└───────┼──────────────┼──────────────┼───────────────────────────────┘
        │              │              │
        ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  API Gateway + CDN (Cloudflare/CloudFront)                           │
│  • Cache question/answer pages at edge (TTL: 5 min)                 │
│  • SSR for SEO (pre-render full HTML for crawlers)                  │
│  • Rate limiting                                                     │
└───────┬──────────────┬────────────────┬──────────────────────────────┘
        │              │                │
┌───────▼──────┐ ┌─────▼─────��┐ ┌──────▼──────┐
│ Question     │ │ Feed       │ │ Search      │
│ & Answer     │ │ Service    │ │ Service     │
│ Service      │ │            │ │             │
└───────┬──────┘ └─────┬──────┘ └──────┬──────┘
        │              │                │
        │              │                │
   ┌────▼────┐    ┌────▼────┐     ┌────▼────────┐
   │ MySQL   │    │ Redis   │     │Elasticsearch│
   │ (Vitess)│    │ (Feed   │     │ (Full-text  │
   │         │    │  cache, │     │  search)    │
   │ Q & A   │    │  vote   │     │             │
   │ Users   │    │  counts)│     │ Q & A index │
   │ Votes   │    │         │     │             │
   │ Follows │    │         │     │             │
   └─────────┘    └─────────┘     └─────────────┘
        │
   ┌────▼────────────────────────────────────────┐
   │           Answer Ranking Service             │
   │                                              │
   │  Score(answer) =                             │
   │    w1 × upvotes                              │
   │    - w2 × downvotes                          │
   │    + w3 × author_credibility                 │
   │    + w4 × answer_length_score                │
   │    + w5 × recency_decay                      │
   │    + w6 × has_references                     │
   │    - w7 × spam_probability                   │
   │                                              │
   │  Wilson Score Interval for confidence:       │
   │  With few votes → rank lower (uncertain)     │
   │  With many votes → rank matches true quality │
   │                                              │
   │  Periodically recompute for top questions    │
   │  Cache ranked answer order in Redis          │
   └──────────────────────────────────────────────┘
```

### Component Deep Dive

#### Answer Ranking — Wilson Score vs Simple Vote Count

```
Simple upvote - downvote:
  Answer A: 100 up, 2 down → score = 98
  Answer B: 5 up, 0 down → score = 5
  A ranks higher. Seems correct.

But what about:
  Answer C: 1 up, 0 down → score = 1
  Answer D: 500 up, 400 down → score = 100
  D ranks higher, but it's controversial (44% downvote rate!)
  C has 100% approval but only 1 vote → low confidence

Wilson Score Interval ⭐ (Reddit's approach):
  Considers BOTH the ratio of upvotes AND the sample size.
  Low confidence (few votes) → lower bound of score is low → ranks lower
  High confidence (many votes, high upvote ratio) → ranks higher
  
  Formula (lower bound of 95% confidence interval):
  score = (p + z²/2n - z√(p(1-p)/n + z²/4n²)) / (1 + z²/n)
  where p = upvotes/total, n = total votes, z = 1.96 (95% CI)
  
  This naturally handles:
    New answers (low n → low score → needs to "prove itself")
    Controversial answers (low p → low score even with many votes)
    Universally liked answers (high p, high n → highest score)
```

#### Feed Generation — Hybrid Push/Pull

```
Sources for a user's feed:
  1. New answers to questions they follow
  2. New questions in topics they follow
  3. Activity from users they follow
  4. Trending/recommended content
  
For normal users (pull):
  On feed load → query for recent activity from followed entities
  Merge, rank, return top 50
  Cache in Redis: feed:{user_id} (TTL: 5 min)

For prolific authors (fan-out):
  When a popular author writes an answer:
  → Don't fan out to 10M followers (too expensive)
  → Instead: add to trending/recommended pool
  → Followers discover it via periodic feed refresh
```

---

## 5. APIs

### Ask Question
```http
POST /api/v1/questions
{ "title": "What is the best database for analytics?", 
  "details": "I'm comparing ClickHouse, BigQuery, and Redshift...",
  "topics": ["databases", "analytics", "data-engineering"] }
→ 201 Created { "question_id": "q-uuid", "url": "/q/what-is-the-best-database-for-analytics" }
```

### Post Answer
```http
POST /api/v1/questions/{question_id}/answers
{ "body": "I'd recommend ClickHouse for self-hosted and BigQuery for managed..." }
→ 201 Created { "answer_id": "a-uuid" }
```

### Vote
```http
POST /api/v1/votes
{ "target_type": "answer", "target_id": "a-uuid", "vote": "up" }
→ 200 OK { "upvotes": 153, "downvotes": 4 }
```

---

## 6. Data Model

### MySQL (Vitess) — Core Data
```sql
CREATE TABLE questions (
    question_id     BIGINT PRIMARY KEY AUTO_INCREMENT,
    author_id       BIGINT NOT NULL,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    slug            VARCHAR(500) UNIQUE,     -- SEO-friendly URL
    view_count      INT DEFAULT 0,
    answer_count    INT DEFAULT 0,
    follow_count    INT DEFAULT 0,
    status          ENUM('open','closed','merged') DEFAULT 'open',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP,
    FULLTEXT INDEX idx_search (title, body),
    INDEX idx_author (author_id)
);

CREATE TABLE answers (
    answer_id       BIGINT PRIMARY KEY AUTO_INCREMENT,
    question_id     BIGINT NOT NULL,
    author_id       BIGINT NOT NULL,
    body            TEXT NOT NULL,
    upvotes         INT DEFAULT 0,
    downvotes       INT DEFAULT 0,
    wilson_score    FLOAT DEFAULT 0,
    is_accepted     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_question_score (question_id, wilson_score DESC),
    INDEX idx_author (author_id)
);

CREATE TABLE votes (
    user_id         BIGINT,
    target_type     ENUM('question','answer'),
    target_id       BIGINT,
    vote_type       ENUM('up','down'),
    created_at      TIMESTAMP,
    PRIMARY KEY (user_id, target_type, target_id)  -- one vote per user per target
);
```

### Redis
```
vote_count:{type}:{id}    → Hash {up: 153, down: 4}
answer_order:{question_id} → Sorted Set (score = wilson_score, member = answer_id)
feed:{user_id}            → List of content_ids (TTL: 5 min)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Vote manipulation** | Rate limit + only count votes from accounts > 7 days old |
| **Duplicate votes** | PRIMARY KEY (user_id, target_type, target_id) prevents duplicates |
| **Search index lag** | Elasticsearch updated via CDC (Debezium); lag < 30 seconds |
| **SEO staleness** | CDN cache TTL = 5 min; instant purge on content update |
| **Answer quality** | ML spam classifier + community flagging + moderator review queue |

### Race Conditions

#### Vote Count vs Wilson Score Drift
```
Multiple users vote simultaneously:
  T=0: upvotes=100, downvotes=4
  Thread A: reads (100,4) → votes up → writes upvotes=101
  Thread B: reads (100,4) → votes up → writes upvotes=101 (LOST UPDATE!)

Solution: Atomic increment
  UPDATE answers SET upvotes = upvotes + 1 WHERE answer_id = ?;
  Then: async job recomputes wilson_score from (upvotes, downvotes)
  Wilson score recomputation batched every 30 seconds (not per vote)
```

---

## 8. Deep Dive: Engineering Trade-offs

### SEO: Server-Side Rendering vs Client-Side

```
CSR (Single Page App):
  Google crawler loads page → sees empty div → JavaScript runs → content appears
  ✗ Google may not execute JS properly → pages not indexed → zero SEO traffic
  
SSR ⭐ (Quora's approach):
  Server renders full HTML with question/answer content
  Crawler sees complete content → indexes properly → SEO traffic
  
  Implementation: Next.js / Nuxt.js → render on server → send HTML
  Cache: Full HTML cached at CDN edge (CloudFront) → TTL 5 min
  Dynamic content (vote counts): client-side hydration after page load
  
  Result: Quora gets 50%+ of traffic from Google → SSR is non-negotiable
```

### MySQL vs MongoDB for Q&A Data

```
MySQL ⭐ (Quora's actual choice):
  ✓ Relational: question → answers → votes (natural joins)
  ✓ Strong consistency for vote counts
  ✓ FULLTEXT index for basic search
  ✓ Vitess for horizontal sharding
  ✗ Schema changes require migrations

MongoDB:
  ✓ Flexible schema (answers can have varying metadata)
  ✓ Embed answers inside question document (one read)
  ✗ Embedded answers grow unbounded (16 MB document limit!)
  ✗ No joins → vote aggregation is application-level
  
For Q&A: MySQL wins. The data is inherently relational, 
consistency matters, and Vitess solves scaling.
```

