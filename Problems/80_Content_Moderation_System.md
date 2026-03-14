# 80. Design a Content Moderation System

---

## 1. Functional Requirements (FR)

- **Multi-modal moderation**: Moderate text, images, video, and audio content
- **Real-time scoring**: Score content for violations before or immediately after publishing
- **Policy engine**: Configurable rules per content type, region, and community standards
- **Violation categories**: Hate speech, nudity/NSFW, violence/gore, spam, misinformation, harassment, copyright, child safety (CSAM)
- **Action framework**: Auto-remove (high confidence), auto-flag for review (medium), allow (low risk)
- **Human review queue**: Prioritized queue for flagged content with analyst tooling
- **Appeals**: Users can appeal moderation decisions; routed to senior reviewers
- **User reporting**: Users report content; reports feed into moderation pipeline
- **Audit trail**: Every moderation decision logged with reason, model version, reviewer
- **Feedback loop**: Reviewer decisions retrain ML models

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Pre-publish moderation in < 500 ms (text); < 5 seconds (image); < 30 seconds (video)
- **High Recall**: Catch > 99.5% of CSAM, > 95% of hate speech (false negatives are catastrophic)
- **Acceptable Precision**: False positive rate < 5% (over-moderation hurts user engagement)
- **Scale**: 500M+ posts/day (text), 100M+ images/day, 10M+ videos/day
- **Availability**: 99.99% — moderation failure means either everything blocked or everything published
- **Regional Compliance**: Different rules per country (EU Digital Services Act, India IT rules, US Section 230)
- **Reviewer Wellbeing**: Limit exposure of harmful content to human reviewers; rotate, counsel

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Text posts / day | 500M |
| Images / day | 100M |
| Videos / day | 10M |
| Text moderation / sec | ~6K |
| Image moderation / sec | ~1.2K |
| Video moderation / sec | ~120 |
| Human review cases / day | 5M (~1% of total) |
| Human reviewers | ~15K (handling ~300 cases/day each) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    CONTENT INGESTION                                   │
│                                                                        │
│  Post/Comment/Story/Message Upload                                     │
│       |                                                                │
│  +----v-----------+                                                    │
│  | Content        |  (text extracted, media URLs resolved,             │
│  | Pre-Processor  |   user trust score fetched from Redis,             │
│  | Service        |   content hash computed for dedup)                 │
│  +----+-----------+                                                    │
│       |                                                                │
│  +----v-----------+                                                    │
│  |    Kafka       | (moderation-requests topic,                        │
│  |                |  partitioned by content_type)                      │
│  +----+-----------+                                                    │
└───────|────────────────────────────────────────────────────────────────┘
        |
┌───────|────────────────────────────────────────────────────────────────┐
│       |           MODERATION PIPELINE                                  │
│       |                                                                │
│  +----v-----------+                                                    │
│  | Router         | (route to appropriate moderation pipeline          │
│  | Service        |  based on content_type: text/image/video/audio)    │
│  +----+-----------+                                                    │
│       |                                                                │
│  +----+--------------------+--------------------+                      │
│  |                         |                    |                      │
│  |   TEXT PIPELINE         |  IMAGE PIPELINE    |  VIDEO PIPELINE      │
│  |                         |                    |                      │
│  | +--v-----------+  +----v-----------+  +----v-----------+           │
│  | | Keyword/     |  | PhotoDNA /     |  | Key Frame      |           │
│  | | Bloom Filter |  | pHash Match    |  | Extraction     |           │
│  | | (< 1ms)      |  | (CSAM check,  |  | (every 2s +    |           │
│  | +--v-----------+  |  < 10ms,       |  |  scene change) |           │
│  |    |              |  MANDATORY)     |  +----v-----------+           │
│  | +--v-----------+  +----v-----------+       |                       │
│  | | BERT/        |  | EfficientNet  |  +----v-----------+           │
│  | | DistilBERT   |  | (nudity,      |  | Image pipeline |           │
│  | | Multi-label  |  |  violence,    |  | per frame      |           │
│  | | (hate, spam, |  |  weapons,     |  +----v-----------+           │
│  | |  sexual,     |  |  hate symbols)|       |                       │
│  | |  violence,   |  +----v-----------+  +----v-----------+           │
│  | |  self_harm)  |       |              | Audio Pipeline |           │
│  | +--v-----------+  +----v-----------+  | (Whisper STT   |           │
│  |    |              | OCR (text in   |  |  -> text pipe, |           │
│  |    |              |  image ->      |  |  audio event   |           │
│  |    |              |  text pipeline)|  |  classifier)   |           │
│  |    |              +----v-----------+  +----v-----------+           │
│  |    |                   |                   |                       │
│  | +--v-----------+       |                   |                       │
│  | | LLM (border- |       |                   |                       │
│  | |  line only,  |       |                   |                       │
│  | |  score 0.3-  |       |                   |                       │
│  | |  0.7, < 2s)  |       |                   |                       │
│  | +--------------+       |                   |                       │
│  +----+--------------------+-------------------+                      │
│       |                                                                │
│  +----v-----------+                                                    │
│  | Score          | (combine all model outputs into                    │
│  | Aggregator     |  per-category violation scores)                    │
│  +----+-----------+                                                    │
│       |                                                                │
│  +----v-----------+                                                    │
│  | Policy Engine  | (apply region + content_type + user_age            │
│  | (configurable  |  specific thresholds from policy table)            │
│  |  rules per     |                                                    │
│  |  region)       |                                                    │
│  +----+-----------+                                                    │
│       |                                                                │
└───────|────────────────────────────────────────────────────────────────┘
        |
┌───────|────────────────────────────────────────────────────────────────┐
│       |              ACTION & FEEDBACK LAYER                           │
│       |                                                                │
│  +----+-------------+-------------------+                              │
│  |                   |                   |                              │
│  v                   v                   v                              │
│  AUTO-ALLOW       FLAG FOR REVIEW     AUTO-REMOVE                      │
│  (score < 0.3)    (0.3 - 0.9)        (score > 0.9)                    │
│  |                   |                   |                              │
│  |              +----v-----------+       |                              │
│  |              | Human Review   |  +----v-----------+                 │
│  |              | Queue (Redis   |  | Content        |                 │
│  |              |  sorted set,   |  | Removal        |                 │
│  |              |  priority =    |  | Service        |                 │
│  |              |  severity ×    |  | (hide content, |                 │
│  |              |  reach ×       |  |  increment     |                 │
│  |              |  time_sens.)   |  |  user strikes, |                 │
│  |              +----+-----------+  |  notify user)  |                 │
│  |                   |              +----+-----------+                 │
│  |              +----v-----------+       |                              │
│  |              | Analyst        |       |                              │
│  |              | Dashboard      |       |                              │
│  |              | (review tool,  |       |                              │
│  |              |  context,      |       |                              │
│  |              |  precedents,   |       |                              │
│  |              |  one-click     |       |                              │
│  |              |  actions)      |       |                              │
│  |              +----+-----------+       |                              │
│  |                   |                   |                              │
│  |              +----v-----------+       |                              │
│  |              | Decision:      |       |                              │
│  |              | approve/remove/|       |                              │
│  |              | escalate/warn  |       |                              │
│  |              +----+-----------+       |                              │
│  |                   |                   |                              │
│  +-------------------+-------------------+                              │
│                      |                                                  │
│               +------v------+                                          │
│               |    Kafka    | (moderation-decisions topic)              │
│               +------+------+                                          │
│                      |                                                  │
│    +-----------------+------------------+-------------------+          │
│    |                 |                  |                   |          │
│ +--v------+   +------v------+   +------v------+   +-------v------+   │
│ |User     |   |Model        |   |Strike       |   |Analytics     |   │
│ |Notif    |   |Feedback     |   |Service      |   |(ClickHouse:  |   │
│ |(removal |   |Pipeline     |   |(progressive |   | precision,   |   │
│ | reason, |   |(reviewer    |   | enforcement:|   | recall,      |   │
│ | appeal  |   | decisions   |   | warn->ban)  |   | review time, |   │
│ | link)   |   | -> retrain  |   |             |   | model drift) |   │
│ +---------+   | ML weekly)  |   +-------------+   +--------------+   │
│               +-------------+                                          │
│                                                                        │
│  DATA STORES:                                                          │
│  +-------------+ +--------+ +-------+ +-----------+ +--------+       │
│  | PostgreSQL  | | Redis  | | Kafka | | ClickHouse| | S3     |       │
│  | (decisions, | | (review| | (all  | | (analytics| | (content|      │
│  |  policies,  | |  queue,| | events| |  model    | |  for   |       │
│  |  strikes,   | |  trust | | async)| |  metrics) | |  review)|      │
│  |  appeals)   | |  score)| |       | |           | |        |       │
│  +-------------+ +--------+ +-------+ +-----------+ +--------+       │
└────────────────────────────────────────────────────────────────────────┘
```
                 +-------------+    | (0.3-0.9)   |    +------+------+
                                    +------+------+           |
                                           |                  v
                                    +------v------+    Notification
                                    | Human       |    to User
                                    | Review      |
                                    | Queue       |
                                    +------+------+
                                           |
                                    +------v------+
                                    | Decision    | --> Feedback --> Model Retraining
                                    | (approve/   |
                                    |  remove/    |
                                    |  escalate)  |
                                    +-------------+

Data Stores:
  PostgreSQL (moderation decisions, appeals, policies)
  Redis (rate limiting, user trust scores, model results cache)
  Kafka (content events, moderation events)
  S3 (content storage for review)
  ClickHouse (moderation analytics, model performance metrics)
```

### Component Deep Dive

#### Text Moderation Pipeline

```
Input: text content (post, comment, message, username, bio)

Layer 1: Keyword/Regex Filter (< 1 ms)
  Bloom filter + Aho-Corasick for known bad terms
  Catches: obvious slurs, known spam phrases, phone numbers in comments
  Fast but easily bypassed ("h8te" instead of "hate")

Layer 2: ML Text Classifier (< 50 ms)
  Model: Fine-tuned BERT or distilled model (DistilBERT for speed)
  Multi-label classification:
    P(hate_speech), P(harassment), P(spam), P(sexual), 
    P(violence), P(self_harm), P(misinformation)
  
  Handles: context, sarcasm (partially), obfuscation ("h@te", "k1ll")
  Training data: millions of labeled examples from human reviewers
  
  Inference: batch of 32 texts on GPU in ~50 ms total = 1.5 ms per text
  At 6K texts/sec: need 6000/640 = ~10 GPU instances (batch of 32, 50ms per batch)

Layer 3: LLM-based Analysis (for borderline cases only, < 2 sec)
  Only invoked if Layer 2 score is 0.3-0.7 (borderline)
  Prompt: "Analyze the following text for policy violations. 
           Consider context, intent, and severity. Explain your reasoning."
  
  LLM provides: nuanced analysis + explanation (useful for appeals)
  Expensive: ~$0.01 per text → only 10% of traffic → $50K/day
  
  Why not LLM for everything?
    $0.01 × 500M texts/day = $5M/day → way too expensive
    But for 50M borderline cases: $500K/day → acceptable for accuracy gain

Layer 4: Context Enrichment
  User's history: is this user a repeat offender?
  Conversation context: reply to what? (reply saying "kill it" to a cooking post = OK)
  Community norms: what's acceptable in r/MMA vs r/parenting?
```

#### Image Moderation Pipeline

```
Input: uploaded image

Step 1: Hash matching — PhotoDNA / pHash (< 10 ms)
  Compare image hash against known illegal content database (CSAM, terrorism)
  PhotoDNA: Microsoft's perceptual hash, robust against resizing/cropping
  NCMEC database: known CSAM hashes (legally required to check)
  
  Match found → IMMEDIATE removal + report to NCMEC (legal requirement in US)
  
  This is non-negotiable. Every platform must do this.

Step 2: ML Classification (< 200 ms)
  Model: EfficientNet or ResNet-50 fine-tuned on moderation data
  Multi-label output:
    P(nudity), P(violence), P(hate_symbol), P(drugs), P(weapons),
    P(gore), P(spam_text_in_image), P(minor_present)
  
  If P(minor_present) > 0.5 AND P(nudity) > 0.5 → IMMEDIATE escalation to CSAM team
  
  Inference: ~20 ms per image on GPU (batched)
  At 1.2K images/sec: need ~30 GPU instances

Step 3: OCR + Text Moderation (for text in images)
  Extract text from image using Tesseract/PaddleOCR
  Run text through text moderation pipeline
  Catches: hate speech in memes, spam overlays, phone numbers in profile pics

Step 4: Object Detection
  YOLO/Faster-RCNN for specific object detection:
    Weapons, drug paraphernalia, flags/symbols associated with extremism
  
  Combined with context: image of gun in news article vs gun in threatening post
```

#### Video Moderation Pipeline

```
Video is the hardest: long, multi-modal (visual + audio + text)

Approach: Sample + Classify (not every frame)

Step 1: Extract key frames (every 2 seconds) + audio track
  10-minute video → 300 frames + audio file
  Much cheaper than analyzing every frame (18,000 at 30fps)

Step 2: Run image moderation on each key frame
  300 frames × 20 ms = 6 seconds (parallelized across GPU batch)
  If ANY frame has high violation score → flag entire video

Step 3: Audio moderation
  Speech-to-text (Whisper) → text moderation pipeline
  Audio classification: gunshots, screams, hate speech audio
  Music copyright detection: audio fingerprint matching (like Shazam)

Step 4: Temporal context
  Single violent frame in a news report: likely OK (news context)
  30 seconds of continuous violence: likely violating
  Temporal analysis: how sustained is the violation?

Optimization — Prioritized Scanning:
  Don't process all 300 frames equally:
    First pass: sample 10 evenly-spaced frames → quick assessment
    If all 10 are clean: mark as likely OK → low-priority full scan later
    If any flagged: immediately scan all 300 frames → real-time decision
  
  Result: 90% of videos need only 10 frames analyzed (fast, cheap)
  Only 10% need full 300-frame scan
```

#### Policy Engine — Regional Compliance

```
Same content may be legal in one country and illegal in another:

  Nazi symbols: illegal in Germany, legal (but policy-violating) in US
  Blasphemy: illegal in Pakistan, legal in US/EU
  Lèse-majesté: illegal in Thailand (criticism of monarchy)
  Nudity: more accepted in EU than US for art/breastfeeding
  Political content: restricted in China, free in US

Implementation:
  Policy rules are configurable per:
    - Country/region
    - Content type (post, ad, story, message)
    - User age (under 18 → stricter rules)
    - Community type (default vs adult-marked)

  policy_rules table:
    { rule_id, region, content_type, violation_category, 
      auto_remove_threshold, review_threshold, enabled }

  Example:
    { region: "DE", category: "hate_symbol", auto_remove_threshold: 0.5, ... }
    { region: "US", category: "hate_symbol", auto_remove_threshold: 0.8, ... }

  On moderation decision:
    1. Determine user's region and content type
    2. Load applicable policy rules
    3. Apply model scores against regional thresholds
    4. Make decision based on MOST RESTRICTIVE applicable rule
    
  "Geo-restricted removal": content removed in Germany but visible in US
    Implementation: content has visibility_restrictions JSONB field
    CDN/API checks viewer's region against restrictions before serving
```

#### Human Review Queue — Prioritization

```
5M flagged items/day. 15K reviewers. Not all items are equal priority.

Priority scoring:
  priority = severity_weight × reach × time_sensitivity

  severity_weight:
    CSAM: 1000 (immediate, legal obligation)
    Violence/threats: 100
    Hate speech: 50
    Nudity: 30
    Spam: 10
    Minor policy violation: 5

  reach:
    Viral post (1M+ views): 10x multiplier
    Regular post: 1x
    Private message: 0.5x

  time_sensitivity:
    Content going viral RIGHT NOW: 10x
    Posted 2 hours ago, steady: 1x
    Posted 3 days ago: 0.5x

Queue implementation:
  Redis sorted set: ZADD review_queue {priority_score} {content_id}
  Reviewer dequeues: ZPOPMAX review_queue

  Specialization:
    CSAM team: dedicated, specially trained, mandatory counseling
    Hate speech team: language-specific expertise
    Copyright team: legal training
    General team: spam, nudity, misc violations

Reviewer tooling:
  - Show content with model's prediction + confidence
  - Show similar past decisions (precedent)
  - One-click actions: remove, approve, escalate, warning
  - Context: user's history, post's reach, community norms
  - Time limit: 30-60 seconds per decision (to maintain throughput)
```

---

## 5. APIs

### Score Content (Pre-Publish)
```http
POST /api/v1/moderation/score
{
  "content_id": "post-uuid",
  "content_type": "text",
  "text": "This is the post content...",
  "user_id": "user-uuid",
  "region": "US",
  "community_id": "comm-uuid"
}
Response: 200 OK (< 500 ms)
{
  "decision": "allow",
  "scores": {
    "hate_speech": 0.05,
    "spam": 0.02,
    "violence": 0.01,
    "sexual": 0.03
  },
  "flags": [],
  "review_required": false
}
```

### Report Content (User Report)
```http
POST /api/v1/moderation/report
{
  "content_id": "post-uuid",
  "reporter_id": "user-uuid",
  "reason": "hate_speech",
  "details": "This post contains racial slurs"
}
Response: 200 OK
{ "report_id": "rpt-uuid", "status": "received" }
```

### Review Decision (Analyst)
```http
POST /api/v1/moderation/review/{content_id}/decide
{
  "reviewer_id": "reviewer-uuid",
  "decision": "remove",
  "violation_category": "hate_speech",
  "notes": "Clear racial slur in context of harassment"
}
Response: 200 OK
{ "decision_id": "dec-uuid", "action_taken": "removed" }
```

### Appeal
```http
POST /api/v1/moderation/appeal
{
  "content_id": "post-uuid",
  "user_id": "user-uuid",
  "reason": "This is a news quote, not hate speech"
}
Response: 200 OK
{ "appeal_id": "app-uuid", "status": "under_review" }
```

---

## 6. Data Model

### PostgreSQL — Moderation Decisions & Policies

```sql
CREATE TABLE moderation_decisions (
    decision_id     UUID PRIMARY KEY,
    content_id      UUID NOT NULL,
    content_type    ENUM('text','image','video','audio','profile') NOT NULL,
    user_id         UUID NOT NULL,
    auto_scores     JSONB,                    -- ML model output scores
    auto_decision   ENUM('allow','review','remove'),
    final_decision  ENUM('allow','remove','restrict','warning'),
    violation_category VARCHAR(50),
    reviewer_id     UUID,
    reviewer_notes  TEXT,
    policy_version  VARCHAR(20),
    model_version   VARCHAR(20),
    region          VARCHAR(10),
    appealed        BOOLEAN DEFAULT FALSE,
    appeal_decision ENUM('upheld','overturned'),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    decided_at      TIMESTAMPTZ,
    INDEX idx_content (content_id),
    INDEX idx_user (user_id, created_at DESC),
    INDEX idx_review_pending (auto_decision) WHERE final_decision IS NULL
);

CREATE TABLE moderation_policies (
    policy_id       UUID PRIMARY KEY,
    region          VARCHAR(10) NOT NULL,
    content_type    VARCHAR(20) NOT NULL,
    violation_category VARCHAR(50) NOT NULL,
    auto_remove_threshold DECIMAL(4,3) DEFAULT 0.90,
    review_threshold DECIMAL(4,3) DEFAULT 0.30,
    enabled         BOOLEAN DEFAULT TRUE,
    effective_from  TIMESTAMPTZ,
    effective_to    TIMESTAMPTZ,
    UNIQUE (region, content_type, violation_category, effective_from)
);

CREATE TABLE user_reports (
    report_id       UUID PRIMARY KEY,
    content_id      UUID NOT NULL,
    reporter_id     UUID NOT NULL,
    reason          VARCHAR(50),
    details         TEXT,
    status          ENUM('received','reviewed','actioned','dismissed'),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_content (content_id),
    INDEX idx_reporter (reporter_id, created_at DESC)
);

CREATE TABLE user_strikes (
    strike_id       UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    violation_category VARCHAR(50),
    decision_id     UUID REFERENCES moderation_decisions(decision_id),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_user (user_id, created_at DESC)
);
```

### Redis — Real-Time State

```
# Human review queue (priority sorted set)
review_queue:{category}     --> Sorted Set { content_id: priority_score }

# User trust score (repeat offenders get lower trust = stricter moderation)
user_trust:{user_id}        --> FLOAT (0.0 = untrusted, 1.0 = highly trusted)
TTL: 86400

# User strike count (for escalating penalties)
strikes:{user_id}           --> INT
TTL: none (managed by expiry logic in application)

# Report dedup (prevent spam reports from same user)
reported:{reporter_id}:{content_id}  --> "1"
TTL: 86400

# Model result cache (avoid re-scoring same content)
mod_score:{content_hash}    --> JSON (scores)
TTL: 3600
```

### Kafka Topics

```
Topic: content-created          (new content → trigger moderation pipeline)
Topic: moderation-decisions     (decision made → update content visibility, notify user)
Topic: moderation-feedback      (reviewer decisions → feed ML training pipeline)
Topic: user-reports             (user reports → boost priority in review queue)
Topic: strike-events            (strikes issued → consumed by account service for bans)
```

### ClickHouse — Analytics

```sql
CREATE TABLE moderation_analytics (
    content_id UUID, content_type String, 
    auto_decision String, final_decision String,
    violation_category String, region String,
    model_version String, model_scores String,
    reviewer_id UUID, review_duration_sec UInt32,
    timestamp DateTime, date Date MATERIALIZED toDate(timestamp)
) ENGINE = MergeTree() PARTITION BY toYYYYMM(timestamp)
  ORDER BY (violation_category, timestamp);

-- Queries:
-- "What's our false positive rate for hate speech this week?"
-- "How long does the average review take per category?"
-- "Which model version performs best for NSFW detection?"
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Moderation service down** | Fail-close for new accounts (block content until moderated); fail-open for trusted users (publish, async review) |
| **ML model degradation** | Monitor precision/recall daily; auto-rollback to previous model if metrics drop > 5% |
| **Review queue backlog** | Auto-adjust thresholds: increase auto-remove threshold from 0.9 to 0.85 during backlog; hire surge reviewers |
| **False positive spike** | Circuit breaker: if auto-remove rate > 2x normal, switch all decisions to "review" |
| **Hash database unavailable** | Continue with ML-only; CSAM hash matching is critical — maintain local cache + replicas |
| **Regional policy update** | Version policies with effective dates; hot-reload from PostgreSQL every 60 seconds |
| **Reviewer burnout** | Rotate reviewers across categories; limit harmful content exposure to 4 hours/day; mandatory counseling |

### Specific: Handling Viral Harmful Content

```
Scenario: Terrorist attack live-streamed. Video going viral in real-time.
  15 minutes: video shared 500K times. Reuploads happening faster than removal.

Response playbook:
  1. CSAM/terrorism hash match → immediate auto-removal + hash added to blocklist
  2. Hash all known copies → perceptual hash (survives re-encoding, cropping)
  3. Block ALL uploads that match hash within 10 seconds of upload
  4. ML model: flag visually similar content even if hash doesn't match
  5. Keyword filters: block titles/descriptions referencing the event
  6. Rate limit new account uploads (prevent burner accounts)
  7. Human review team: "war room" mode — all hands on this event

  Result: New Zealand Christchurch attack (2019): Facebook removed 1.5M copies 
  in first 24 hours. 80% caught by automated systems before any human saw them.

Prevention: shared industry hash databases
  - GIFCT (Global Internet Forum to Counter Terrorism): shared hash database
  - PhotoDNA / NCMEC: CSAM hash sharing
  - Platforms contribute hashes → all platforms can block known content
```

---

## 8. Deep Dive: Engineering Trade-offs

### Pre-Publish vs Post-Publish Moderation

```
Pre-publish (moderate BEFORE content is visible):
  ✓ Harmful content never seen by any user
  ✓ Better user experience (no exposure to violations)
  ✗ Adds latency to publishing (500ms text, 5s image, 30s video)
  ✗ Over-moderation blocks legitimate content before anyone sees it
  ✗ At 500M posts/day, pre-publish for everything is expensive
  
  Use for: high-risk content types (ads, verified account posts with large reach)

Post-publish (publish immediately, moderate async):
  ✓ Zero latency for publishing (instant gratification)
  ✓ Only moderate content that gets engagement (ignore 0-view posts)
  ✗ Harmful content visible for seconds-to-minutes before removal
  ✗ Viral harmful content can spread before caught
  
  Use for: low-risk users with trust score > 0.8, text-only posts

Hybrid (recommended):
  New users / low trust score: pre-publish moderation (stricter)
  Established users / high trust: post-publish moderation (faster UX)
  
  Trust score factors:
    account_age, verification_status, past_violations, 
    community_standing, content_type
  
  Score 0.0-0.3: pre-publish everything
  Score 0.3-0.7: pre-publish images/videos, post-publish text
  Score 0.7-1.0: post-publish everything (fast async review)
```

### Model Accuracy vs Coverage: The Precision-Recall Trade-off

```
Setting auto-remove threshold:

Threshold = 0.95 (very high confidence required):
  Precision: 99% (almost no false positives)
  Recall: 70% (misses 30% of violations)
  Result: few legitimate posts wrongly removed, but lots of harmful content stays up
  More items need human review → larger review team needed

Threshold = 0.70 (moderate confidence):
  Precision: 85% (15% false positive rate)
  Recall: 95% (catches almost everything)
  Result: catches most violations, but 15% of auto-removed content was actually fine
  Fewer items for human review, but more appeals from wrongly-removed users

Different thresholds per category:
  CSAM: threshold = 0.50 (maximize recall at all costs — legal obligation)
  Spam: threshold = 0.80 (some false positives acceptable — users don't appeal spam removal)
  Hate speech: threshold = 0.90 (high bar — context matters, false positives are controversial)
  Nudity: threshold = 0.85 (reasonably confident; artistic nudity is edge case)
  
  Principle: the higher the harm of a false negative, the lower the threshold should be.
```

### Cost of Moderation at Scale

```
ML inference costs (per day):
  Text: 500M × $0.0001 = $50K
  Images: 100M × $0.001 = $100K
  Videos: 10M × $0.01 = $100K
  LLM (borderline only, 10% of text): 50M × $0.01 = $500K
  Total ML: ~$750K/day = ~$274M/year

Human review costs:
  15K reviewers × $15/hour avg (global) × 8 hours = $1.8M/day = ~$660M/year
  (Facebook/Meta reportedly spends ~$1B/year on content moderation)

Cost optimization:
  1. Trust-based routing: skip moderation for trusted users (reduces volume 50%)
  2. Efficient models: distilled models (DistilBERT) are 4x faster than BERT, 97% accuracy
  3. Tiered approach: cheap keyword filter catches 20%, ML catches 79%, LLM for 1%
  4. Only moderate content that gets views: post with 0 views → don't waste GPU time
  5. Hash matching is extremely cheap: catches reuploads of known-bad content for ~$0

Optimized cost: ~$150M-200M/year for a major platform (vs $900M+ without optimization)
```

### User Strike System — Progressive Enforcement

```
First violation: Warning (educational)
Second violation: Content removed + 24-hour posting restriction
Third violation: 7-day account suspension
Fourth violation: 30-day suspension
Fifth violation: Permanent ban (with appeal)

Strike expiry: strikes expire after 6 months of clean behavior.
  Incentivizes reform: user can "earn back" trust over time.

Severity override:
  CSAM: immediate permanent ban (no warnings)
  Terrorism: immediate suspension + law enforcement referral
  Doxxing: immediate 30-day suspension (minimum)

Implementation:
  On moderation decision "remove":
    INCR strikes:{user_id}
    INSERT INTO user_strikes (...)
    
    strikes = GET strikes:{user_id}
    penalty = PENALTY_SCHEDULE[min(strikes, 5)]
    
    if penalty.type == "suspension":
      UPDATE users SET suspended_until = NOW() + penalty.duration
      Publish Kafka: user-suspended event
    
    Notify user: "Your post was removed for {reason}. This is strike {n}."
```

