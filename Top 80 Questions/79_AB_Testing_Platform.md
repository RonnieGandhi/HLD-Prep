# 79. Design an A/B Testing and Experimentation Platform

---

## 1. Functional Requirements (FR)

- **Create experiments**: Define experiment with name, hypothesis, variants (control + treatments), and traffic allocation
- **User assignment**: Deterministically assign users to experiment variants (consistent hashing)
- **Feature flags**: Toggle features on/off; gradual rollout (1% -> 10% -> 50% -> 100%)
- **Metric tracking**: Track conversion rates, revenue, engagement metrics per variant
- **Statistical analysis**: Compute p-value, confidence interval, statistical significance
- **Mutual exclusion**: Prevent conflicting experiments from overlapping (same user in two related experiments)
- **Experiment lifecycle**: Draft -> Running -> Paused -> Completed -> Archived
- **Guardrail metrics**: Auto-stop experiment if key metrics (crash rate, latency) degrade
- **Segmentation**: Run experiments on specific user segments (country, platform, account age)

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Variant assignment in < 5 ms (in every API request path)
- **Consistency**: Same user always sees same variant (deterministic assignment)
- **Scale**: 1000+ concurrent experiments, 500M+ users, 50B+ events/day
- **Statistical Rigor**: Correct p-values; account for multiple comparisons
- **No Impact on Production**: Experiment infrastructure must not slow down main services
- **Availability**: 99.99% for assignment; analytics can tolerate minutes of lag

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Concurrent experiments | 1,000 |
| Users | 500M |
| Variant assignment calls / sec | 500K |
| Experiment events (impressions, clicks) / day | 50B |
| Event data storage / day | 5 TB |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                    ASSIGNMENT PATH (< 5 ms)                            │
│                                                                        │
│  Client / Server --> Experiment SDK (embedded library)                  │
│                          |                                             │
│                   +------v------+                                      │
│                   | Assignment  |                                      │
│                   | Engine      |                                      │
│                   | (in-process |                                      │
│                   |  hash-based |                                      │
│                   |  assignment)|                                      │
│                   +------+------+                                      │
│                          |                                             │
│               +----------+----------+                                  │
│               |                     |                                  │
│        +------v------+       +------v------+                           │
│        | Local Cache |       | Redis       |                           │
│        | (experiment |       | (experiment |                           │
│        |  configs,   |       |  configs,   |                           │
│        |  refreshed  |       |  layer      |                           │
│        |  every 60s) |       |  assignments)|                          │
│        +-------------+       +------+------+                           │
│                                     |                                  │
│                              +------v------+                           │
│                              | PostgreSQL  |                           │
│                              | (experiment |                           │
│                              |  definitions|                           │
│                              |  variants,  |                           │
│                              |  layers,    |                           │
│                              |  lifecycle) |                           │
│                              +-------------+                           │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    EVENT PIPELINE                                      │
│                                                                        │
│  SDK (client/server) --> event { experiment_id, variant, user_id,      │
│                                  event_type, value, timestamp }        │
│          |                                                             │
│   +------v-------+                                                     │
│   |    Kafka     |  (experiment-events topic,                          │
│   |              |   partitioned by experiment_id)                      │
│   +------+-------+                                                     │
│          |                                                             │
│   +------v----------------------------------------------------+       │
│   |                 Flink Pipeline                             |       │
│   |                                                            |       │
│   |  1. Real-time metric aggregation per (experiment, variant) |       │
│   |     - impressions, conversions, revenue (tumbling 1-min)   |       │
│   |  2. Guardrail metric monitoring                            |       │
│   |     - crash_rate, p99_latency, error_rate per variant      |       │
│   |     - If degradation > 2 stddev -> auto-pause experiment   |       │
│   |  3. Sample Ratio Mismatch (SRM) detection                  |       │
│   |     - Expected: 50/50 split. Actual: 48/52 -> alert        |       │
│   |     - Chi-squared test on assignment counts                |       │
│   +------+----------------------------------------------------+       │
│          |                                                             │
│   +------v-------+     +-------------------+     +-----------------+  │
│   | ClickHouse   |     | Statistics Engine |     | Experiment      |  │
│   | (raw events, |---->| (batch, hourly)   |---->| Dashboard       |  │
│   |  aggregated  |     |                   |     | (React UI)      |  │
│   |  metrics per |     | - p-value         |     |                 |  │
│   |  experiment/ |     | - confidence      |     | - lift per      |  │
│   |  variant/    |     |   interval        |     |   variant       |  │
│   |  day)        |     | - power analysis  |     | - significance  |  │
│   |              |     | - Bayesian        |     | - guardrails    |  │
│   |              |     |   posterior       |     | - SRM warnings  |  │
│   +--------------+     | - sequential      |     | - recommended   |  │
│                        |   testing bounds  |     |   action        |  │
│                        +-------------------+     +-----------------+  │
└────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Deterministic User Assignment

```
User must ALWAYS see the same variant. No randomness per request.

Algorithm: hash-based assignment

  def get_variant(user_id, experiment_id, variants, traffic_percent):
      # Deterministic hash
      hash_input = f"{experiment_id}:{user_id}"
      hash_value = murmurhash3(hash_input) % 10000  # 0-9999
      
      # Traffic allocation: only include traffic_percent of users
      if hash_value >= traffic_percent * 100:
          return None  # user not in experiment
      
      # Assign to variant proportionally
      # variants = [("control", 50), ("treatment_a", 25), ("treatment_b", 25)]
      cumulative = 0
      for variant_name, weight in variants:
          cumulative += weight * 100  # scale to 0-10000
          if hash_value < cumulative:
              return variant_name
      
      return variants[-1][0]  # fallback

Properties:
  - Deterministic: same user_id + experiment_id -> same variant, always
  - Uniform: murmurhash3 is uniformly distributed -> fair variant split
  - Independent: adding/removing one experiment doesn't change other assignments
    (because experiment_id is part of hash input)
  - Fast: murmurhash3 is < 100 ns

Why NOT random assignment?
  Random: user sees control on Monday, treatment on Tuesday.
  Pollutes data (same user contributes to both groups).
  Hash-based: user ALWAYS in same group. Clean data.
```

#### Mutual Exclusion (Experiment Layers)

```
Problem: Experiment A tests checkout button color. Experiment B tests checkout page layout.
If same user is in both: which caused the conversion improvement?

Solution: Experiment Layers (Google's Overlapping Experiment Infrastructure)

  Layer 1 (UI experiments):
    Experiment A: button color (control: blue, treatment: green)
    Experiment B: header layout (control: v1, treatment: v2)
    Within a layer: user is in AT MOST one experiment
    
  Layer 2 (Backend experiments):
    Experiment C: recommendation algorithm (control: v1, treatment: v2)
    Independent of Layer 1
    
  Cross-layer: user CAN be in A AND C simultaneously (no conflict)
  Same-layer: user is in A OR B, never both

Implementation:
  Each experiment belongs to a layer.
  Assignment within a layer: hash(user_id + layer_id) % total_traffic
  Each experiment in the layer "owns" a non-overlapping range of hash values.
  
  Layer 1 (10000 hash buckets):
    Experiment A: buckets 0-2999 (30% of traffic)
    Experiment B: buckets 3000-5999 (30% of traffic)
    Unallocated: buckets 6000-9999 (40% available for new experiments)
```

#### Statistical Analysis Engine

```
For each experiment, compute:

1. Conversion rate per variant:
   control_rate = control_conversions / control_impressions
   treatment_rate = treatment_conversions / treatment_impressions

2. Relative lift:
   lift = (treatment_rate - control_rate) / control_rate

3. Statistical significance (two-proportion z-test):
   p_pooled = total_conversions / total_impressions
   se = sqrt(p_pooled * (1 - p_pooled) * (1/n_control + 1/n_treatment))
   z = (treatment_rate - control_rate) / se
   p_value = 2 * (1 - norm_cdf(abs(z)))

4. Confidence interval (95%):
   CI = (treatment_rate - control_rate) ± 1.96 * se

5. Sample size / power:
   Minimum sample per variant for 80% power:
   n = (z_alpha + z_beta)^2 * (p1*(1-p1) + p2*(1-p2)) / (p1 - p2)^2
   Experiment should not conclude until minimum sample reached.

6. Multiple comparison correction:
   With 3+ variants: Bonferroni correction (divide alpha by number of comparisons)
   Or: Benjamini-Hochberg procedure (controls false discovery rate)

7. Sequential testing (optional, for early stopping):
   Instead of fixed sample size: use sequential analysis
   Can stop experiment early if result is clearly significant
   Method: O'Brien-Fleming boundaries or Bayesian posterior probability

Guardrail metrics:
  Track key health metrics alongside experiment metrics:
    - App crash rate, page load time, error rate
  If ANY guardrail metric degrades > 2 standard deviations: AUTO-PAUSE experiment
  Alert experiment owner.
```

---

## 5. APIs

```http
POST /api/v1/experiments
{
  "name": "checkout_button_color",
  "hypothesis": "Green button increases conversion by 5%",
  "layer": "checkout_ui",
  "traffic_percent": 20,
  "variants": [
    { "name": "control", "weight": 50, "config": { "button_color": "blue" } },
    { "name": "treatment", "weight": 50, "config": { "button_color": "green" } }
  ],
  "primary_metric": "checkout_conversion_rate",
  "guardrail_metrics": ["crash_rate", "p95_latency"]
}

GET /api/v1/experiments/{experiment_id}/assignment?user_id=user-uuid
-> { "variant": "treatment", "config": { "button_color": "green" } }

POST /api/v1/experiments/{experiment_id}/events
{ "user_id": "user-uuid", "event_type": "conversion", "value": 1, "timestamp": "..." }

GET /api/v1/experiments/{experiment_id}/results
-> {
  "status": "running", "days_running": 7,
  "variants": {
    "control": { "impressions": 50000, "conversions": 2500, "rate": 0.0500 },
    "treatment": { "impressions": 49800, "conversions": 2750, "rate": 0.0552 }
  },
  "lift": 0.104, "p_value": 0.0023, "significant": true,
  "confidence_interval": [0.032, 0.176],
  "recommended_action": "Ship treatment (statistically significant improvement)"
}
```

---

## 6. Data Model

### PostgreSQL — Experiment Configuration

```sql
CREATE TABLE experiments (
    experiment_id UUID PRIMARY KEY, name VARCHAR(100), hypothesis TEXT,
    layer VARCHAR(50), traffic_percent DECIMAL(5,2),
    primary_metric VARCHAR(100), guardrail_metrics JSONB,
    status ENUM('draft','running','paused','completed','archived'),
    started_at TIMESTAMPTZ, ended_at TIMESTAMPTZ,
    owner VARCHAR(100), created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE experiment_variants (
    variant_id UUID PRIMARY KEY, experiment_id UUID NOT NULL,
    name VARCHAR(50), weight INT, config JSONB
);
```

### Redis — Fast Assignment
```
exp_config:{experiment_id}  -> JSON (experiment config for fast assignment)
TTL: 60 (refreshed from PostgreSQL)

active_experiments:{layer}  -> LIST of experiment_ids
TTL: 60
```

### ClickHouse — Event Analytics
```sql
CREATE TABLE experiment_events (
    experiment_id UUID, variant String, user_id UUID,
    event_type String, value Float64,
    timestamp DateTime, date Date MATERIALIZED toDate(timestamp)
) ENGINE = MergeTree() PARTITION BY toYYYYMM(timestamp)
  ORDER BY (experiment_id, variant, timestamp);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Assignment service down** | SDK caches last assignment locally; fall back to control variant |
| **Event pipeline lag** | ClickHouse backfill from Kafka replay; results delayed but not lost |
| **Experiment degrades metrics** | Guardrail auto-pause; manual kill switch |
| **Hash collision causing uneven split** | Chi-squared test on assignment counts; alert if > 2% deviation |
| **Novelty effect** | Run experiments for minimum 2 weeks; track metrics over time, not just cumulative |

---

## 8. Deep Dive

### Peeking Problem

```
Problem: checking p-value daily and stopping when it's < 0.05.
  This inflates false positive rate from 5% to 20-30%!
  
Why? Statistical tests assume you look at data ONCE at predetermined sample size.
  Multiple looks -> multiple chances to see "significant" result by chance.

Solution:
  1. Pre-determine sample size. Don't look at results until reached.
  2. Sequential testing with adjusted boundaries (O'Brien-Fleming).
     Allows periodic checking with correct false positive rate.
  3. Bayesian approach: compute posterior probability continuously.
     P(treatment > control | data). No "stopping too early" problem.
```

### Feature Flags vs A/B Tests

```
Feature flag: binary (on/off). Used for gradual rollout, kill switches.
  "Enable new checkout flow for 10% of users, then 50%, then 100%"
  No statistical analysis needed. Just monitoring.

A/B test: controlled experiment with statistical rigor.
  "Does new checkout flow improve conversion?" with p-value.
  Requires control group, sufficient sample size, statistical analysis.

In practice: same infrastructure serves both.
  Feature flag = experiment with 1 variant + 100% traffic allocation + no metrics.
  A/B test = experiment with 2+ variants + metrics + statistical analysis.
```

### Sample Ratio Mismatch (SRM) Detection

```
Experiment configured for 50/50 split. After 1 week:
  Control: 502,000 users. Treatment: 498,000 users.
  Is this a problem?

Chi-squared test:
  Expected: 500,000 each. Observed: 502,000 / 498,000.
  chi2 = (502000-500000)^2/500000 + (498000-500000)^2/500000 = 16.0
  p-value < 0.001 -> SIGNIFICANT MISMATCH

Causes of SRM:
  1. Bot traffic filtered differently per variant (bots don't execute JS -> no events)
  2. Treatment causes more app crashes -> users never log the impression event
  3. Redirect-based experiment: one variant has slower redirect -> more users abandon
  4. Hash function isn't uniformly distributed for this population

Impact: SRM invalidates the experiment. Results cannot be trusted.
  Any observed metric difference might be due to the assignment bias, not the treatment.

Action: investigate root cause. Fix. Re-run experiment.
Always monitor SRM automatically and alert if p < 0.01.
```

### Network Effects — When User Independence Breaks

```
A/B tests assume: user A's behavior is independent of user B's assignment.
This breaks with network effects:

Example: Testing a new messaging feature.
  User A (treatment) has new feature. User A messages User B (control).
  User B's experience is affected by A's treatment -> independence violated.

Solutions:
  1. Cluster randomization: assign entire friend clusters to same variant
     All friends of User A -> same variant
     Expensive: requires graph analysis at assignment time

  2. Geo-based randomization: assign entire cities to variants
     City A gets treatment, City B gets control
     Less statistical power (fewer independent units) but clean isolation

  3. Time-based (switchback): alternate treatment by time period
     Week 1: treatment. Week 2: control. Week 3: treatment.
     Measures aggregate effect on entire population.
     Problem: carry-over effects between periods.

For social platforms: cluster or geo randomization is necessary for chat/feed experiments.
For independent features (checkout UI): standard user-level randomization is fine.
```

