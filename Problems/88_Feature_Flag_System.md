# 88. Design a Feature Flag System

---

## 1. Functional Requirements (FR)

- Create, update, and delete feature flags (boolean, string, number, JSON variants)
- Target flags by user ID, user attributes (country, plan, device), percentage rollout
- Gradual rollout: 1% → 5% → 25% → 50% → 100% (with rollback at any point)
- A/B testing integration: assign users to experiment variants deterministically
- Kill switch: instantly disable a feature globally in < 5 seconds
- Flag dependencies: flag B requires flag A to be enabled
- Audit log: who changed what flag, when, and why
- SDK support: server-side (Java, Go, Python) and client-side (JS, iOS, Android)
- Environment separation: dev, staging, prod with independent flag states
- Scheduled flags: auto-enable at a specific time (launch events)

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: Flag evaluation in < 1µs (local in-memory, no network call)
- **High Availability**: 99.999% — flag evaluation must never fail (default fallback)
- **Consistency**: Flag change propagated to all servers within 10 seconds
- **Scalability**: 10K+ flags, 1B+ evaluations/day
- **Resilience**: SDK works offline / when flag service is down (cached state)
- **Zero Performance Impact**: No measurable overhead in the hot path

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Feature flags (total) | | 10,000 |
| Active flags | | 2,000 |
| Flag evaluations / sec | 1B/day ÷ 86400 | ~12K/sec per instance × 100 instances |
| Flag changes / day | | ~50 |
| SDK instances | servers + clients | 100,000 |
| Flag definition payload | All flags compressed | ~50 KB |
| Streaming update bandwidth | 100K connections × 1 KB/min | ~1.7 MB/sec |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   MANAGEMENT PLANE                                     │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Admin Dashboard (React UI)                                  │       │
│  │  - Create/edit/delete flags                                  │       │
│  │  - Configure targeting rules + percentage rollout            │       │
│  │  - View flag status across environments                      │       │
│  │  - Audit log viewer (who changed what, when)                 │       │
│  │  - Kill switch button (one click → globally off)             │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │                                              │
│  ┌──────────────────────▼──────────────────────────────────────┐       │
│  │  Flag Management API Service (Stateless, behind LB)          │       │
│  │  - CRUD for flags and targeting rules                       │       │
│  │  - Validate rules (no circular dependencies)                │       │
│  │  - Write to PostgreSQL (source of truth)                    │       │
│  │  - Invalidate Redis cache                                   │       │
│  │  - Publish change event to Kafka                            │       │
│  │  - Write audit log entry                                    │       │
│  └──────────────────────┬──────────────────────────────────────┘       │
│                         │                                              │
└─────────────────────────│──────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────────┐
         │                │                    │
┌────────▼───────┐ ┌──────▼──────┐  ┌──────────▼───────────┐
│  PostgreSQL    │ │  Redis      │  │  Kafka               │
│  (Truth)       │ │  (Cache)    │  │  (Change Stream)     │
│                │ │             │  │                      │
│  - Flag defs   │ │  - Full flag│  │  Topic: flag-changes │
│  - Environments│ │    snapshot │  │  - Consumed by relay │
│  - Rules       │ │  - TTL: 5m  │  │    service           │
│  - Audit log   │ │             │  │                      │
└────────────────┘ └─────────────┘  └──────────┬───────────┘
                                               │
                          ┌────────────────────▼────────────────────────┐
                          │  FLAG RELAY SERVICE (Edge-deployed)          │
                          │                                              │
                          │  - SSE/WebSocket connections to all SDKs    │
                          │  - On flag change → push update instantly   │
                          │  - Full flag set on initial SDK connect     │
                          │  - Horizontally scaled (one per region)     │
                          │  - If SDK disconnects → SDK polls every 30s │
                          └────────────────────┬───────────────────────┘
                                               │
┌──────────────────────────────────────────────▼───────────────────────┐
│               APPLICATION SERVICES (SDK in-process)                   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐      │
│  │  Feature Flag SDK (in-process library)                      │      │
│  │                                                              │      │
│  │  ┌───────────────────────────────────────────────┐           │      │
│  │  │  In-Memory Flag Store                          │           │      │
│  │  │  - ConcurrentHashMap of all flag definitions   │           │      │
│  │  │  - Updated atomically via pointer swap         │           │      │
│  │  │  - Evaluation: ZERO I/O, pure memory           │           │      │
│  │  └──────────────────────┬────────────────────────┘           │      │
│  │                         │                                    │      │
│  │  ┌──────────────────────▼────────────────────────┐           │      │
│  │  │  Evaluation Engine                             │           │      │
│  │  │                                                │           │      │
│  │  │  evaluate("new_checkout", userCtx):            │           │      │
│  │  │    1. Kill switch OFF? → return default        │           │      │
│  │  │    2. User in allowlist? → return variant      │           │      │
│  │  │    3. Attribute rules match?                   │           │      │
│  │  │       country=US AND plan=premium → true       │           │      │
│  │  │    4. Percentage rollout:                      │           │      │
│  │  │       hash(flag_key + user_id) % 100           │           │      │
│  │  │       < rollout_pct? → true                    │           │      │
│  │  │    5. Return fallthrough variant                │           │      │
│  │  └────────────────────────────────────────────────┘           │      │
│  │                                                              │      │
│  │  ┌────────────────────────────────────────────────┐           │      │
│  │  │  Persistent Disk Cache                          │           │      │
│  │  │  - Survives restart; fallback when relay down  │           │      │
│  │  └────────────────────────────────────────────────┘           │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                       │
│  if (featureFlags.isEnabled("new_checkout", userCtx)) {               │
│      showNewCheckout();                                                │
│  } else { showLegacyCheckout(); }                                     │
└───────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Flag Relay Service
- **Why a dedicated relay?** Direct DB polling from 100K SDKs = 100K queries/sec → DB dies. Relay multiplexes: 1 Kafka consumer → 100K SSE pushes
- **Scaling**: 1 relay handles ~10K SSE connections. 10 relays per region
- **Regional deployment**: us-east, eu-west, ap-south → low latency push

#### Deterministic Percentage Rollout (Key Algorithm)

```
bucket = murmurhash3(flag_key + ":" + user_id) % 100

rollout_percent = 25 → bucket < 25 → ON

Monotonic increase:
  25% → 50%: users 0-24 STILL ON, users 25-49 NOW ON
  Nobody loses access — only gains

Why MurmurHash3: fast (~5ns), uniform distribution, deterministic
```

#### Multi-Variant Experiments

```
Flag: "checkout_layout"
  Variant A: "single_page" (33%), Variant B: "multi_step" (33%), Variant C: "wizard" (34%)

Mutual exclusion via experiment layers:
  Layer 1 (checkout): hash1(user_id) % 100 < 50
  Layer 2 (pricing): hash2(user_id) % 100 >= 50
  Different hash seed per layer ensures non-correlation
```

---

## 5. APIs

```
POST   /api/flags              → Create flag with targeting rules
GET    /api/flags               → List all flags (paginated)
GET    /api/flags/{key}         → Flag details with all environments
PUT    /api/flags/{key}         → Update flag (creates audit entry)
DELETE /api/flags/{key}         → Archive flag (soft delete)
POST   /api/flags/{key}/toggle  → Kill switch enable/disable
GET    /api/flags/{key}/audit   → Audit log with diffs
POST   /api/flags/{key}/schedule → Schedule future enable/disable

# SDK endpoints
GET    /api/sdk/flags?env=prod   → Full flag definitions
GET    /api/sdk/stream?env=prod  → SSE stream of changes
POST   /api/sdk/evaluate         → Server-side evaluation for client SDKs
```

---

## 6. Data Models

### PostgreSQL (Source of Truth)

**Why PostgreSQL?** ACID for metadata, JSONB for flexible rules, reliable for low-write workload (~50 writes/day).

```sql
CREATE TABLE feature_flags (
    flag_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_key    TEXT UNIQUE NOT NULL,
    name        TEXT NOT NULL,
    description TEXT,
    flag_type   TEXT NOT NULL CHECK (flag_type IN ('boolean','string','number','json')),
    created_by  UUID, created_at TIMESTAMPTZ DEFAULT NOW(),
    archived    BOOLEAN DEFAULT FALSE
);

CREATE TABLE flag_environments (
    flag_id         UUID REFERENCES feature_flags(flag_id),
    environment     TEXT NOT NULL,
    enabled         BOOLEAN DEFAULT FALSE,
    default_variant JSONB,
    fallthrough     JSONB,
    rules           JSONB,      -- ordered array of targeting rules
    version         BIGINT DEFAULT 1,
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_by      UUID,
    PRIMARY KEY (flag_id, environment)
);

CREATE TABLE flag_audit_log (
    audit_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_id    UUID NOT NULL,
    flag_key   TEXT NOT NULL,
    environment TEXT,
    action     TEXT NOT NULL,
    old_value  JSONB,
    new_value  JSONB,
    changed_by UUID NOT NULL,
    reason     TEXT,
    changed_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_audit_flag ON flag_audit_log(flag_id, changed_at DESC);
```

### Redis Cache

```
SET flags:production '{...all flags...}' EX 300
SET flag_version:production 42
```

### SDK In-Memory (Atomic Pointer Swap)

```java
class FlagStore {
    volatile Map<String, FlagDef> flags;  // immutable, replaced atomically
    FlagDef get(String key) { return flags.get(key); }  // O(1), zero locks
    void update(Map<String, FlagDef> n) { this.flags = Map.copyOf(n); }
}
```

---

## 7. Fault Tolerance

| Technique | Application |
|---|---|
| SDK disk cache | Survives restart; fallback when relay unreachable |
| Default values | Flag not found → hardcoded default |
| Streaming + polling | SSE primary, 30s poll backup, disk cache last resort |
| PostgreSQL replicas | Read replicas for API reads |
| Kafka RF=3 | Change events survive broker failure |
| Relay redundancy | Multiple instances per region; SDK reconnects on failure |

### Problem-Specific

**SDK Resilience**: Startup: disk cache → relay fetch → defaults. Runtime: streaming → polling → cache. Evaluation NEVER makes a network call.

**Concurrent Flag Updates**: Optimistic locking — `UPDATE ... WHERE version = $expected`. Conflict → 409, UI shows diff.

**Flag Debt**: Expiration dates with alerts, stale detection (100% for >30 days → prompt), CI code scanning (flag key not in codebase → auto-archive).

**Relay Failure**: SDKs detect disconnect → reconnect to another relay → send last_seen_version → receive delta. If ALL relays down → poll API. If API down → use cached flags.

---

## 8. Additional Considerations

### Client-Side vs Server-Side SDKs
- **Server-side**: Gets ALL flags, evaluates locally, secure (rules not exposed)
- **Client-side**: Gets ONLY evaluated results for current user (server evaluates)
- **Anti-flicker**: Block render until flags loaded (200ms timeout) or SSR with flags

### Performance: ~100ns per evaluation (10M evals/sec per instance)

### Why Not Config Files?
```
Config: deploy required (30 min), no targeting, no audit, no gradual rollout
Flags: change in 5 sec, targeted, audited, reversible, schedulable
Kill switch alone justifies the system for any org with >100 engineers
```

---

## 9. Deep Dive: Engineering Trade-offs

### Race Condition Analysis

```
Race 1: Flag update arrives DURING an evaluation

  Thread A: evaluating flag "new_checkout" for user X
    1. Read flag definition from in-memory ConcurrentHashMap
    2. Evaluate targeting rules
    3. Compute hash for percentage rollout
    4. Return result
  
  Thread B (SSE listener): new flag snapshot arrives
    1. Deserialize new flag definitions
    2. Build new immutable Map
    3. Atomic pointer swap: this.flags = newMap  (volatile write)
  
  Is this safe? YES.
    Thread A reads flags reference once at step 1 (volatile read)
    Even if Thread B swaps the pointer during steps 2-4,
    Thread A still holds a reference to the OLD map → completes consistently
    Next evaluation will see the NEW map
    
    Java volatile guarantees: write in Thread B happens-before
    any subsequent read in Thread A (JMM memory model)
    
    No partial state: Thread A sees ALL old flags or ALL new flags, never a mix

Race 2: Multi-flag dependency during update

  Flag B depends on Flag A: "B is enabled ONLY IF A is enabled"
  Update: disable Flag A
  
  Problem window (~10ms):
    T+0ms: SSE delivers new snapshot to SDK
    T+5ms: SDK starts building new map
    T+8ms: Evaluation request → reads OLD map → A=enabled, B=enabled ✓
    T+10ms: New map swapped (A=disabled, B=disabled) ✓
    
    No issue: atomic snapshot swap means both flags change together
    
  BUT: What if Flag A and Flag B are on DIFFERENT relay messages?
    T+0ms: Relay sends "Flag A disabled"
    T+5ms: SDK updates A → A=disabled, B still=enabled → INCONSISTENT!
    T+100ms: Relay sends "Flag B disabled"
    
    Solution: Always send FULL flag snapshot (not individual flag deltas)
    Trade-off: more bandwidth (~50 KB per update) vs consistency guarantee
    This is why LaunchDarkly and similar systems use full snapshot pushes

Race 3: Scheduled flag activation with clock skew

  Admin schedules: "Enable flag X at 9:00 AM EST"
  
  Scheduler runs on Server A (clock: 9:00:00.000)
  SDK on Server B (clock: 8:59:58.200 — 1.8s behind)
  SDK on Server C (clock: 9:00:01.500 — 1.5s ahead)
  
  Server C sees flag enabled 1.5s before Server A activates it? NO.
  Server B evaluates flag as disabled 1.8s after it should be on? YES.
  
  Solutions:
    1. NTP sync all servers to < 500ms accuracy (practical minimum)
    2. For time-critical activations: use server-side evaluation
       SDK calls API: "Is flag X enabled for user Y?"
       Server uses canonical server time → consistent across all clients
       Trade-off: 5ms latency per evaluation vs 0ms for local evaluation
    3. Pre-activate 5 seconds early, rely on relay propagation
       By T=0, all SDKs have received the update
       Trade-off: flag appears 5 seconds early for some users (acceptable)
```

### Consistency Model

```
This system provides BOUNDED EVENTUAL CONSISTENCY:
  After a flag change, all SDKs converge to the new state within ~10 seconds

  Timeline of a flag change:
    T+0s:     Admin clicks "Enable flag" in dashboard
    T+0.1s:   API writes to PostgreSQL (committed)
    T+0.2s:   API publishes to Kafka (flag-changes topic)
    T+0.5s:   Relay Service consumes from Kafka
    T+1s:     Relay pushes update via SSE to all connected SDKs
    T+5s:     All healthy SDKs have received update (SSE propagation)
    T+10s:    SDKs that were reconnecting have received update (poll fallback)
    T+30s:    SDKs with network issues have received update (poll retry)

  During the T+0 to T+10s window:
    Some users see old flag state, others see new
    This is acceptable for feature flags (not financial data)
    
  For kill switches requiring stronger consistency:
    Use server-side evaluation via API call
    API reads from Redis (refreshed within 1 second of DB write)
    Latency: ~5ms per evaluation
    Only use for critical flags (< 1% of all evaluations)

CAP analysis:
  Network partition between Relay and Kafka:
    SDKs can't receive updates → serve stale cached flags
    Choice: AVAILABILITY over consistency
    Alternative: refuse to evaluate → blocks all feature-flagged code paths
    → Clearly wrong: serving stale flags is better than crashing the app
    
  This is an AP system by design.
  Flags are NOT a source of truth for critical business logic.
  Critical decisions (payments, inventory) should NOT depend solely on flags.
```

### Full Snapshot Push vs Delta Updates

```
Full Snapshot (this design, LaunchDarkly approach):
  Every update: relay sends complete set of all flag definitions (~50 KB)
  SDK replaces entire in-memory map atomically
  
  ✓ Always consistent (all flags in sync)
  ✓ Recovery is trivial (missed an update? Full snapshot on reconnect)
  ✓ No ordering dependencies (no "apply delta 42 before delta 43")
  ✗ Bandwidth: 100K SDKs × 50 KB × 50 updates/day = 250 GB/day
  
Delta Updates (alternative):
  Only send the changed flag definition (~1 KB)
  SDK applies patch to in-memory map
  
  ✓ 50× less bandwidth per update
  ✗ Ordering matters: missed delta → state diverges
  ✗ Need sequence numbers and gap detection
  ✗ Recovery requires full snapshot anyway (bootstrap + catch-up)
  ✗ Multi-flag dependency race (see Race 2 above)
  
Recommendation: Full snapshot (simplicity + consistency > bandwidth)
  50 KB compressed to ~15 KB with gzip → 75 GB/day → trivial at scale
```

### In-Process SDK vs Remote Evaluation API

```
In-Process SDK (this design, recommended):
  Flag definitions downloaded to SDK → evaluated locally in memory
  Latency: ~100 ns (pure memory read, zero I/O)
  Availability: 100% (works offline, on crash, during network partition)
  
  ✗ Flag definitions exposed to client (security for server-side)
  ✗ SDK size (~500 KB library added to application)
  ✗ SDK must be updated when evaluation logic changes

Remote Evaluation API:
  Every evaluation → HTTP call to Flag Service
  Latency: 5-20 ms (network round-trip)
  Availability: depends on Flag Service uptime
  
  ✓ No SDK to maintain per language
  ✓ Flag definitions never leave the server (secure)
  ✗ 1M evaluations/day × 10ms = 2.8 hours of cumulative latency
  ✗ Network failure → flags stop working (unless cached)
  
  Used for: client-side SDKs (browser, mobile)
    Client can't be trusted with all flag definitions
    Server evaluates for the specific user → returns results only

Production pattern:
  Server-side: in-process SDK (Java, Go, Python) → 0ms latency
  Client-side: server evaluates on behalf of client → returns user's flags
    Initial page load: one batch API call for all flags → cache locally
    Updates: SSE stream of evaluated results (not raw definitions)
```

### Deterministic Hash vs Server-Assigned Cohort

```
Deterministic Hash (this design, MurmurHash3):
  bucket = hash(flag_key + ":" + user_id) % 100
  ✓ Stateless: any SDK instance computes same result
  ✓ No storage: cohort assignment not stored anywhere
  ✓ Monotonic rollout: 25% → 50% → users 0-24 stay ON, 25-49 added
  ✗ Can't manually override specific users' cohort (except via allowlist)
  ✗ Hash collisions: two flags may assign same users to treatment
    Mitigation: include flag_key in hash → independent assignments per flag

Server-Assigned Cohort:
  On first evaluation: server assigns user to cohort, stores in DB
  ✓ Full control: admin can move specific users between cohorts
  ✓ Stable assignment even if hash function changes
  ✓ Cross-flag coordination (ensure user in experiment A isn't in B)
  ✗ Requires DB lookup per new user → latency
  ✗ Storage: 1B users × 10K flags = 10T assignments → expensive
  ✗ Not stateless: SDK must query server for assignment

Recommendation: Deterministic hash for most flags
  Server-assigned only for formal A/B experiments requiring strict isolation
  Experiment platform (see #79) handles server-assigned cohorts separately
```

