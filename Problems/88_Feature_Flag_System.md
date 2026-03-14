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

