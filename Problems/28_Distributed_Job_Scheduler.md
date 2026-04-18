# 28. Design a Distributed Job Scheduler

---

## 1. Functional Requirements (FR)

- **Submit jobs**: Users/services submit jobs to be executed at a specific time or on a recurring schedule (cron-like)
- **One-time jobs**: Execute once at a specific time (e.g., "send email at 3pm tomorrow")
- **Recurring jobs**: Execute on a schedule (e.g., "every 5 minutes", "daily at midnight", cron expressions)
- **Job priorities**: Support priority levels (critical, high, normal, low)
- **Job dependencies**: Job B runs only after Job A completes (DAG execution)
- **Retry on failure**: Configurable retry policy (max retries, backoff strategy)
- **Job status**: Track status (pending, scheduled, running, completed, failed, cancelled)
- **Job cancellation**: Cancel a pending or scheduled job
- **Exactly-once execution**: A job must execute exactly once per scheduled time (no duplicates, no misses)

---

## 2. Non-Functional Requirements (NFRs)

- **Reliability**: Jobs must never be lost; scheduled jobs must execute even if nodes fail
- **Scalability**: Support millions of scheduled jobs, thousands executing concurrently
- **Low Latency**: Jobs execute within 1 second of their scheduled time
- **High Availability**: 99.99% — scheduler failure means jobs don't run
- **Fault Tolerant**: Survive node failures, network partitions
- **Idempotent Execution**: Same job running twice should produce the same result (or be deduplicated)
- **Ordered**: Respect job dependencies and priorities

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total scheduled jobs | 100M |
| Jobs executing / minute | 100K |
| Jobs executing / sec | ~1,700 |
| Avg job duration | 30 seconds |
| Concurrent running jobs | ~50K |
| Job metadata size | 1 KB |
| Storage | 100M × 1 KB = 100 GB |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CLIENTS (Services / Admin UI)                      │
│                                                                       │
│  POST /jobs (one-time)      POST /jobs/recurring (cron)              │
│  GET /jobs/{id} (status)    DELETE /jobs/{id} (cancel)               │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
                         ┌──────▼──────┐
                         │ API Gateway │
                         └──────┬──────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────┐
│                      JOB SCHEDULER SERVICE                            │
│                                                                       │
│  ┌─────────────────┐  ┌────────────────────────────────────────────┐  │
│  │ Job Store       │  │  Schedule Manager (Leader-elected)         │  │
│  │ (CRUD API)      │  │                                            │  │
│  │                 │  │  ┌──────────────────────────────────────┐  │  │
│  │ • Validate job  │  │  │  Timer Bucket Scanner (every 1 sec) │  │  │
│  │ • Write to PG   │  │  │                                      │  │  │
│  │ • Insert into   │  │  │  1. Scan current + previous bucket: │  │  │
│  │   timer bucket  │  │  │     ZRANGEBYSCORE timer:bucket:      │  │  │
│  │ • Compute next  │  │  │       {minute} 0 {now}              │  │  │
│  │   bucket for    │  │  │  2. For each due job:                │  │  │
│  │   cron jobs     │  │  │     ZREM (atomic claim) →            │  │  │
│  │ • Return job_id │  │  │     if success → dispatch            │  │  │
│  │                 │  │  │  3. Acquire dispatch lock:            │  │  │
│  │                 │  │  │     SET job:dispatch:{id} NX EX 60   │  │  │
│  │                 │  │  │  4. Update PG: status=dispatched     │  │  │
│  │                 │  │  │  5. Publish to priority Kafka topic  │  │  │
│  │                 │  │  └──────────────────────────────────────┘  │  │
│  │                 │  │                                            │  │
│  │                 │  │  ┌──────────────────────────────────────┐  │  │
│  │                 │  │  │  Recurring Job Scheduler             │  │  │
│  │                 │  │  │                                      │  │  │
│  │                 │  │  │  On job completion:                  │  │  │
│  │                 │  │  │  1. Parse cron_expression            │  │  │
│  │                 │  │  │  2. Compute next_execute_at          │  │  │
│  │                 │  │  │  3. Insert new job into PG           │  │  │
│  │                 │  │  │  4. ZADD into future timer bucket    │  │  │
│  │                 │  │  └──────────────────────────────────────┘  │  │
│  │                 │  │                                            │  │
│  │                 │  │  ┌──────────────────────────────────────┐  │  │
│  │                 │  │  │  DAG Dependency Resolver             │  │  │
│  │                 │  │  │                                      │  │  │
│  │                 │  │  │  On job completion:                  │  │  │
│  │                 │  │  │  1. Find dependent jobs              │  │  │
│  │                 │  │  │  2. DECR deps:{child_job_id}         │  │  │
│  │                 │  │  │  3. If counter = 0 → all deps met   │  │  │
│  │                 │  │  │     → dispatch child job             │  │  │
│  └─────────────────┘  └────────────────────────────────────────────┘  │
│                                                                       │
└──────┬──────────────────────┬───────────────────────┬─────────────────┘
       │                      │                       │
┌──────▼──────┐        ┌──────▼──────┐         ┌──────▼──────┐
│ PostgreSQL  │        │   Redis     │         │  ZooKeeper  │
│             │        │   Cluster   │         │  / etcd     │
│ • jobs table│        │             │         │             │
│ • recurring │        │ Timer       │         │ Leader      │
│   _jobs     │        │ Buckets:    │         │ election    │
│ • job_      │        │  ZSET per   │         │ for Schedule│
│   history   │        │  minute     │         │ Manager     │
│ • job_deps  │        │             │         │             │
│   (DAG      │        │ Exec Locks: │         │ Partition   │
│    edges)   │        │  SET NX EX  │         │ assignment  │
│             │        │             │         │ for multi-  │
│ Cold store: │        │ Dep Counters│         │ scheduler   │
│ jobs > 24h  │        │  deps:{id}  │         │             │
│ away        │        │             │         │             │
│             │        │ Rate Limits:│         │             │
│             │        │  rate_limit:│         │             │
│             │        │  {job_type} │         │             │
└─────────────┘        └─────────────┘         └─────────────┘
                              │
                              │ Hot/Cold migration:
                              │ Daily job moves tomorrow's
                              │ jobs from PG → Redis buckets
                              │
┌─────────────────────────────▼─────────────────────────────────────────┐
│                    KAFKA (Priority-Based Dispatch)                     │
│                                                                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────────┐ │
│  │jobs.critical │ │jobs.high     │ │jobs.normal   │ │jobs.low     │ │
│  │(priority 1)  │ │(priority 2)  │ │(priority 3)  │ │(priority 4) │ │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬──────┘ │
│         │                │                │                │        │
│  Dedicated          Shared pool      Shared pool      Idle-only    │
│  workers            workers          workers          workers      │
└─────────┼────────────────┼────────────────┼────────────────┼────────┘
          │                │                │                │
┌─────────▼────────────────▼────────────────▼────────────────▼────────┐
│                      WORKER POOL (Executors)                         │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  Worker Execution Lifecycle (per job):                         │   │
│  │                                                                │   │
│  │  1. Consume job from Kafka                                    │   │
│  │  2. Acquire execution lock:                                   │   │
│  │     SET job:exec:{id} {worker_id} NX EX {timeout×2}          │   │
│  │     → if fail → skip (another worker has it)                  │   │
│  │  3. Idempotency check: SELECT status FROM jobs WHERE id = ?   │   │
│  │     → if 'completed' → skip (already done)                    │   │
│  │  4. UPDATE jobs SET status='running', started_at=NOW()        │   │
│  │  5. EXECUTE: call target service with job payload             │   │
│  │  6. On SUCCESS:                                               │   │
│  │     → UPDATE status='completed', result={...}                 │   │
│  │     → Release lock: DEL job:exec:{id}                        │   │
│  │     → If callback_url: POST result to callback                │   │
│  │     → Trigger: recurring scheduler + dependency resolver      │   │
│  │  7. On FAILURE:                                               │   │
│  │     → retries < max_retries?                                  │   │
│  │       YES → compute next_retry (exponential backoff + jitter) │   │
│  │            → ZADD retry into future timer bucket              │   │
│  │            → UPDATE status='retry_pending'                    │   │
│  │       NO  → UPDATE status='failed'                            │   │
│  │            → Publish to jobs.dead-letter (DLQ)                │   │
│  │            → Alert if critical job                            │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌─────────────────┐   ┌──────────────────────────────────────────┐   │
│  │ Worker Pool 1   │   │  Dead Letter Queue Consumer             │   │
│  │ (critical jobs) │   │  • Log failed jobs for investigation    │   │
│  ├─────────────────┤   │  • Alert on-call if critical            │   │
│  │ Worker Pool 2   │   │  • Admin can manually retry from DLQ   │   │
│  │ (normal + high) │   │  • Metrics: DLQ depth, failure reasons  │   │
│  ├─────────────────┤   └──────────────────────────────────────────┘   │
│  │ Worker Pool 3   │                                                  │
│  │ (low priority,  │   ┌──────────────────────────────────────────┐   │
│  │  idle-only)     │   │  Callback Service (async)                │   │
│  └─────────────────┘   │  • POST job result to callback_url      │   │
│                        │  • Retry 3× with exponential backoff    │   │
│                        │  • If callback fails → log, don't retry │   │
│                        │    the job itself                        │   │
│                        └──────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Job Store (CRUD API)
- Entry point for all job submissions and status queries
- **On job submission**:
  1. Validate payload: job_type registered, execute_at in the future, payload matches schema
  2. Check idempotency: if `idempotency_key` already exists in PG → return existing job
  3. Write to PostgreSQL: `INSERT INTO jobs (status='scheduled', execute_at=...)`
  4. Compute timer bucket: `bucket_key = floor(execute_at / 60)`
  5. Insert into Redis timer bucket: `ZADD timer:bucket:{bucket_key} {exact_timestamp} {job_id}`
  6. For recurring jobs: also write to `recurring_jobs` table, compute `next_execute_at` from cron
  7. Return `job_id` to client
- **On job cancellation**:
  1. Update PG: `status = 'cancelled'`
  2. Remove from Redis timer bucket: `ZREM timer:bucket:{bucket_key} {job_id}`
  3. If job is already `dispatched` or `running` → cancellation is best-effort (worker checks status before execution)

#### Schedule Manager — The Core Scheduling Engine

**Leader Election**: Only ONE schedule manager is active at a time (via ZooKeeper/etcd). Standby instances watch the leader node — if leader dies, a standby is promoted within ~5 seconds.

**The Timer Problem**: How to efficiently trigger millions of jobs at their exact scheduled times?

**Approach 1: Polling Database** (naive)
- Every second, query DB: `SELECT * FROM jobs WHERE execute_at <= NOW() AND status = 'pending'`
- **Problem**: Expensive query on large table; doesn't scale

**Approach 2: Delay Queue (Redis Sorted Set)** ⭐
- Store upcoming jobs in a Redis Sorted Set with score = execute_at (Unix timestamp)
- Timer thread polls every second:
  ```
  ZRANGEBYSCORE job_queue 0 {current_timestamp} LIMIT 0 100
  ```
- For each due job → remove from sorted set (ZREM) → publish to Kafka for execution
- **Pros**: O(log N) insert, O(log N + M) range query, handles millions of jobs
- **Cons**: Redis memory limits, single-node bottleneck

**Approach 3: Time-Bucket Approach** ⭐ (recommended for massive scale)
- Divide time into 1-minute buckets
- Each bucket is a sorted set in Redis: `timer:bucket:{minute_timestamp}` → job_ids scored by exact second
- Schedule Manager processes the current minute's bucket
- Jobs within the bucket are sorted by exact second
- **Advantages**: 
  - Only one bucket active at a time → predictable load
  - Past buckets can be cleaned up
  - Easy to shard: different scheduler instances handle different time ranges

```
Bucket: 2026-03-13T10:00  → [job_1, job_2, job_3]
Bucket: 2026-03-13T10:01  → [job_4, job_5]
Bucket: 2026-03-13T10:02  → [job_6, job_7, job_8, job_9]
```

**Approach 4: Hierarchical Timing Wheel** (used by Kafka internally)
- Multi-level timing wheel: seconds → minutes → hours → days
- Each level is a circular array
- O(1) insert and O(1) expiration (amortized)
- Handles millions of timers with minimal memory

#### Timer Bucket Scanner (inside Schedule Manager)
The core loop that fires due jobs — runs every 1 second on the leader:

```
while leader:
    now = current_timestamp()
    current_bucket = floor(now / 60)
    prev_bucket = current_bucket - 1  # catch boundary stragglers
    
    for bucket in [prev_bucket, current_bucket]:
        due_jobs = ZRANGEBYSCORE timer:bucket:{bucket} 0 {now} LIMIT 0 100
        
        for job_id in due_jobs:
            # Atomic claim — only ONE scanner gets it
            claimed = ZREM timer:bucket:{bucket} {job_id}
            if claimed == 0:
                continue  # another scanner took it
            
            # Belt-and-suspenders dispatch lock
            locked = SET job:dispatch:{job_id} 1 NX EX 60
            if not locked:
                continue  # already dispatched
            
            # Update durable state
            UPDATE jobs SET status='dispatched' WHERE job_id = {job_id}
            
            # Dispatch to appropriate priority topic
            priority = lookup_priority(job_id)
            kafka.publish(topic=f"jobs.{priority}", payload=job_payload)
    
    sleep(1 second)
```

**Why scan previous bucket too?** A job scheduled at 15:00:59.9 lands in the 15:00 bucket. If the scanner finishes the 15:00 bucket at 15:00:59.5 (before the job is due), it moves to the 15:01 bucket. Without scanning the previous bucket, this job is never fired.

#### Job Dispatcher (Priority-Based Routing)
- Receives due jobs from the Timer Bucket Scanner
- Publishes job execution request to Kafka, routing to the correct priority topic
- **Priority handling**: Separate Kafka topics per priority level
  - `jobs.critical` → dedicated high-priority worker pool (always has spare capacity)
  - `jobs.high` → shared pool (processed before normal)
  - `jobs.normal` → shared pool (default)
  - `jobs.low` → low-priority pool (only processed when higher-priority topics are empty)
- **Partition key**: `job_type` or `job_id` — ensures ordering within a job type if needed

#### Worker Pool (Executors)
Kafka consumer groups that execute jobs. The full worker execution lifecycle:

1. **Consume** job message from Kafka priority topic
2. **Acquire execution lock**: `SET job:exec:{job_id} {worker_id} NX EX {timeout×2}`
   - If lock already held → skip (another worker has it, Kafka rebalance delivered duplicate)
3. **Idempotency check**: `SELECT status FROM jobs WHERE job_id = ?`
   - If status is `completed` → skip (already done, lock was stale)
4. **Update status**: `UPDATE jobs SET status='running', worker_id=?, started_at=NOW()`
5. **Execute**: Call target service with job payload (HTTP, gRPC, script execution, etc.)
6. **On success**:
   - `UPDATE jobs SET status='completed', result=?, completed_at=NOW()`
   - Release lock: `DEL job:exec:{job_id}`
   - If `callback_url` configured → hand off to Callback Service
   - Trigger Recurring Job Scheduler (compute next fire) and DAG Dependency Resolver (unblock children)
   - Write to `job_history` for audit
7. **On failure**:
   - If `retry_count < max_retries` → compute next retry time (exponential backoff + jitter) → `ZADD` into a future timer bucket → `UPDATE status='retry_pending', next_retry_at=?`
   - If `retry_count >= max_retries` → `UPDATE status='failed'` → publish to `jobs.dead-letter` → alert if job is critical priority

#### Recurring Job Scheduler
Triggered after a recurring job completes (success or permanent failure):

1. Look up `recurring_jobs` table by `recurring_id`
2. Parse `cron_expression` with timezone awareness (e.g., `croniter` library)
   - Handle DST transitions: "daily at 2 AM" — what if 2 AM doesn't exist? (spring forward) → fire at 3 AM
   - What if 2 AM happens twice? (fall back) → fire on FIRST occurrence only
3. Compute `next_execute_at`
4. Insert new job into PostgreSQL: `INSERT INTO jobs (status='scheduled', execute_at=next_execute_at, ...)`
5. If `next_execute_at` is within 24 hours → `ZADD` into Redis timer bucket (hot path)
6. If `next_execute_at` is > 24 hours away → stays in PG only (cold path, migrated later)
7. Update `recurring_jobs`: `last_executed = NOW(), next_execute_at = computed`

**Guard against overlapping executions**: If a recurring job takes longer than its interval (e.g., 5-min job on a 3-min schedule), don't stack up overlaps:
```
Before scheduling next: check if previous instance is still 'running'
If running → skip this occurrence, log warning, alert if consecutive skips > 3
```

#### DAG Dependency Resolver
Manages job dependency graphs (Job C depends on Job A and Job B):

```
Data model (PostgreSQL):
  CREATE TABLE job_deps (
      parent_job_id   UUID,
      child_job_id    UUID,
      PRIMARY KEY (parent_job_id, child_job_id)
  );

On job submission with depends_on = [job_A, job_B]:
  1. Insert edges: (job_A → job_C), (job_B → job_C)
  2. Set dependency counter in Redis: SET deps:{job_C} 2
  3. Set job_C status = 'waiting_deps' (not yet in timer bucket)

On job_A completion:
  1. Query: SELECT child_job_id FROM job_deps WHERE parent_job_id = job_A
     → finds job_C
  2. DECR deps:{job_C} → returns 1 (one dep remaining)
  3. 1 > 0 → not ready yet, do nothing

On job_B completion:
  1. Query: SELECT child_job_id FROM job_deps WHERE parent_job_id = job_B
     → finds job_C
  2. DECR deps:{job_C} → returns 0 (all deps met!)
  3. 0 = 0 → dispatch job_C:
     - Update status = 'scheduled'
     - ZADD into immediate timer bucket (execute_at = NOW)
     - Job_C will be picked up by Scanner within 1 second

Cycle detection: On submission, run topological sort on the DAG
  If cycle detected → reject with 400 Bad Request
  Cycle check: O(V + E) DFS — only at submission time, not on hot path
```

#### Dead Letter Queue (DLQ) Consumer
Handles jobs that exhausted all retries:

- Consumes from `jobs.dead-letter` Kafka topic
- **Actions per failed job**:
  1. Log full job details (payload, all attempt errors, worker IDs) to `job_history`
  2. If priority = critical → page on-call engineer immediately (PagerDuty/OpsGenie)
  3. If priority = high → create incident ticket (Jira/Linear)
  4. If priority = normal/low → log and increment failure metric
- **Admin UI integration**: DLQ dashboard shows failed jobs with ability to:
  - Inspect: view payload, all retry errors, timestamps
  - Retry: one-click re-enqueue back to the normal pipeline
  - Discard: acknowledge and remove from DLQ
- **Metrics**: DLQ depth, DLQ growth rate, top failing job_types, mean time in DLQ

#### Callback Service
Delivers job results to the submitting service asynchronously:

- Triggered by worker on job completion → publishes callback request to internal Kafka topic
- Callback worker picks up and sends:
  ```
  POST {callback_url}
  Content-Type: application/json
  X-Job-Id: job-uuid
  X-Job-Status: completed
  
  { "job_id": "...", "status": "completed", "result": {...}, "completed_at": "..." }
  ```
- **Retry policy**: 3 attempts with exponential backoff (1s, 5s, 25s)
- **If callback fails after retries**: Log failure, do NOT retry or re-execute the job itself. The job execution is complete — callback delivery is best-effort.
- **Idempotency**: Callback includes `X-Job-Id` header → receiving service can deduplicate
- **Timeout**: 10 second HTTP timeout per callback attempt

#### Hot/Cold Job Migration
Manages the split between Redis (hot, fast) and PostgreSQL (cold, durable):

```
Hot path (Redis): Jobs executing within the next 24 hours
  - Stored in timer buckets → scanned every second
  - Fast: O(log N) operations, sub-millisecond

Cold path (PostgreSQL): Jobs scheduled > 24 hours from now
  - Only stored in the jobs table, NOT in Redis
  - No timer scanning overhead for far-future jobs

Migration job (runs every hour):
  SELECT job_id, execute_at FROM jobs
  WHERE status = 'scheduled'
    AND execute_at BETWEEN NOW() AND NOW() + INTERVAL '25 hours'
    AND NOT EXISTS (check if already in Redis)
  
  For each: ZADD timer:bucket:{computed_minute} {exact_ts} {job_id}
  
  Why 25 hours? 1-hour overlap ensures no job falls through the gap.
  Idempotent: ZADD with same score+member is a no-op.

Benefit: 100M total jobs but only ~2M in Redis at any time (24-hour window)
  → Redis memory: 2M × 200 bytes = 400 MB (very manageable)
  vs 100M in Redis = 20 GB (expensive, wasteful)
```

#### Exactly-Once Execution
- **Challenge**: If scheduler crashes after dispatching but before marking job as dispatched, it might dispatch again on recovery
- **Solution — Three Layers of Protection**:
  1. **Atomic claim (ZREM)**: Only one scanner claims a job from the timer bucket — Redis single-threaded guarantees this
  2. **Dispatch lock (SET NX)**: Even if ZREM races (near-impossible), the dispatch lock prevents duplicate publishing to Kafka
  3. **Execution lock + idempotency check**: Worker acquires its own lock AND checks DB status before executing — catches any duplicate that slipped through layers 1 and 2
- **Fencing token**: Each dispatch includes a monotonically increasing sequence number. Worker accepts only if token ≥ last seen token for that job_id (prevents stale dispatch from a partitioned scheduler)

---

## 5. APIs

### Submit One-Time Job
```http
POST /api/v1/jobs
{
  "job_type": "send_email",
  "execute_at": "2026-03-14T15:00:00Z",
  "priority": "normal",
  "payload": {
    "to": "user@example.com",
    "template": "welcome_email",
    "vars": {"name": "Alice"}
  },
  "retry_policy": {
    "max_retries": 3,
    "backoff": "exponential",
    "initial_delay_sec": 60
  },
  "callback_url": "https://myservice.com/job-callback"
}
Response: 201 Created
{
  "job_id": "job-uuid",
  "status": "scheduled",
  "execute_at": "2026-03-14T15:00:00Z"
}
```

### Submit Recurring Job
```http
POST /api/v1/jobs/recurring
{
  "job_type": "generate_report",
  "cron_expression": "0 0 * * *",    // daily at midnight
  "timezone": "America/New_York",
  "payload": {"report_type": "daily_sales"},
  "priority": "high"
}
```

### Get Job Status
```http
GET /api/v1/jobs/{job_id}
Response: 200 OK
{
  "job_id": "job-uuid",
  "status": "completed",
  "execute_at": "2026-03-14T15:00:00Z",
  "started_at": "2026-03-14T15:00:01Z",
  "completed_at": "2026-03-14T15:00:05Z",
  "retries": 0,
  "result": {"email_sent": true}
}
```

### Cancel Job
```http
DELETE /api/v1/jobs/{job_id}
Response: 200 OK
{ "status": "cancelled" }
```

---

## 6. Data Model

### PostgreSQL — Jobs

```sql
CREATE TABLE jobs (
    job_id          UUID PRIMARY KEY,
    job_type        VARCHAR(64) NOT NULL,
    status          VARCHAR(20) NOT NULL,    -- scheduled, dispatched, running, completed, failed, cancelled
    priority        SMALLINT DEFAULT 3,      -- 1=critical, 2=high, 3=normal, 4=low
    payload         JSONB,
    execute_at      TIMESTAMP NOT NULL,
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    retry_count     INT DEFAULT 0,
    max_retries     INT DEFAULT 3,
    backoff_strategy VARCHAR(20),
    next_retry_at   TIMESTAMP,
    result          JSONB,
    error_message   TEXT,
    callback_url    TEXT,
    created_by      VARCHAR(128),
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    INDEX idx_execute (status, execute_at),
    INDEX idx_type (job_type, status),
    INDEX idx_creator (created_by, created_at DESC)
);

CREATE TABLE recurring_jobs (
    recurring_id    UUID PRIMARY KEY,
    job_type        VARCHAR(64),
    cron_expression VARCHAR(64),
    timezone        VARCHAR(64),
    payload         JSONB,
    priority        SMALLINT,
    is_active       BOOLEAN DEFAULT TRUE,
    last_executed   TIMESTAMP,
    next_execute_at TIMESTAMP,
    created_at      TIMESTAMP
);
```

### PostgreSQL — Job Execution History & Dependencies

```sql
CREATE TABLE job_history (
    execution_id    UUID PRIMARY KEY,
    job_id          UUID,
    status          VARCHAR(20),
    worker_id       VARCHAR(128),
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    duration_ms     INT,
    error           TEXT,
    attempt_number  INT
);

CREATE TABLE job_deps (
    parent_job_id   UUID NOT NULL,
    child_job_id    UUID NOT NULL,
    PRIMARY KEY (parent_job_id, child_job_id),
    INDEX idx_child (child_job_id)
);
```

### Redis — Timer Buckets, Locks & Dependency Counters

```
# Time bucket (jobs due in this minute)
Key:    timer:bucket:{minute_epoch}
Type:   Sorted Set
Members: job_id
Scores:  exact_timestamp_seconds

# Job dispatch lock (scanner → Kafka)
Key:    job:dispatch:{job_id}
Value:  1
TTL:    60 seconds

# Job execution lock (worker holds during execution)
Key:    job:exec:{job_id}
Value:  {worker_id}
TTL:    job_timeout_seconds × 2

# Job execution lock (legacy alias)
Key:    job:lock:{job_id}
Value:  {worker_id}
TTL:    job_timeout_seconds × 2

# Recurring job next fire time
Key:    recurring:next:{recurring_id}
Value:  next_execute_timestamp

# DAG dependency counter (decremented on each parent completion)
Key:    deps:{child_job_id}
Value:  integer (number of remaining parent jobs)
TTL:    none (deleted when counter reaches 0)

# Rate limit per job type (token bucket)
Key:    rate_limit:{job_type}
Value:  integer (remaining tokens)
TTL:    auto-refills via Lua script
```

### Kafka Topics

```
Topic: jobs.critical    (priority 1 — dedicated workers)
Topic: jobs.high        (priority 2)
Topic: jobs.normal      (priority 3)
Topic: jobs.low         (priority 4)
Topic: jobs.dead-letter (failed after all retries)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Scheduler crash** | Leader election via ZooKeeper. Standby scheduler takes over in < 5 seconds |
| **Worker crash mid-execution** | Job lock expires → job re-dispatched to another worker |
| **Missed job (scheduler was down)** | On recovery, scan for jobs with `execute_at < NOW() AND status = 'scheduled'` → dispatch immediately |
| **Duplicate execution** | Distributed lock + idempotent job handlers + status check before execution |
| **Kafka unavailable** | Buffer jobs in local queue + retry. If prolonged → direct DB polling as fallback |
| **Clock skew** | Use NTP; timer buckets have 1-minute granularity → tolerant of small skew |

### Specific: Job Failure Handling

```
Retry Strategy:
  Attempt 1: Immediate (or after initial_delay)
  Attempt 2: initial_delay × 2 = 2 min
  Attempt 3: initial_delay × 4 = 4 min
  Attempt 4: Give up → Dead Letter Queue

Backoff types:
  - Fixed: Same delay every time
  - Exponential: delay × 2^attempt
  - Exponential with jitter: delay × 2^attempt + random(0, delay)
```

---

## 8. Additional Considerations

### Job Dependencies (DAG Execution)
See **DAG Dependency Resolver** in Component Deep Dive above for full implementation. Summary:
```
Job A ──┬──▶ Job C ──▶ Job E
        │
Job B ──┘

Redis dependency counter: SET deps:{job_C} 2
On each parent completion: DECR deps:{job_C}
When counter = 0 → all parents done → dispatch child
Cycle detection via topological sort at submission time
```

### Rate Limiting Job Execution
- Some jobs shouldn't exceed a certain rate (e.g., API calls to external service)
- Use token bucket per job_type: `rate_limit:{job_type}` → max 10 executions/second
- Worker checks rate limit before executing; if exceeded → delay and retry

### Comparison with Existing Systems

| System | Type | Use Case |
|---|---|---|
| **Cron** | Single machine | Simple scheduled tasks |
| **Celery** | Task queue (Python) | Async task execution |
| **Airflow** | DAG scheduler | Data pipeline orchestration |
| **Quartz** | Java scheduler | JVM-based scheduling |
| **Temporal** | Workflow engine | Complex, long-running workflows |
| **This design** | Distributed job scheduler | General-purpose, highly available |

### Monitoring & Alerting
- **Job execution success rate** (alert if < 99%)
- **Job execution latency** (time between scheduled and actual execution)
- **Queue depth** (number of pending jobs)
- **Worker utilization** (idle vs busy)
- **Dead letter queue size** (alert if growing)
- **Missed jobs** (scheduled time passed without execution — most critical alert)

---

## 9. Deep Dive: Engineering Trade-offs

### End-to-End Job Execution Flow

```
Job: "Send welcome email to user-123 at 2026-03-14T15:00:00Z"

T-60min (submission):
  1. Client POST /api/v1/jobs → API Gateway → Job Service
  2. Job Service validates: payload, execute_at in future, idempotency_key unique
  3. Write to PostgreSQL: INSERT INTO jobs (status='scheduled', execute_at=...)
  4. Compute timer bucket: bucket = floor(execute_at / 60) = 2026-03-14T15:00
  5. Insert into Redis sorted set:
     ZADD timer:bucket:1710428400 1710428400 "job-uuid"
  6. Return 201 Created to client

T-0 (execution time arrives):
  7. Schedule Manager (leader) polls current bucket every 1 second:
     ZRANGEBYSCORE timer:bucket:1710428400 0 {now} LIMIT 0 100
  8. Found "job-uuid" → atomically remove from sorted set:
     ZREM timer:bucket:1710428400 "job-uuid"
     (If ZREM returns 0 → another scheduler already took it → skip)
  9. Acquire distributed lock: SET job:dispatch:job-uuid 1 NX EX 60
  10. Update PostgreSQL: status = 'dispatched'
  11. Publish to Kafka topic 'jobs.normal' with job payload

T+0.1s (worker picks up):
  12. Worker consumes from Kafka
  13. Worker acquires execution lock: SET job:exec:job-uuid {worker-id} NX EX 300
  14. Worker checks DB: status should be 'dispatched' (idempotency check)
  15. Update status = 'running', started_at = now()
  16. Execute: call email service POST /api/send-email
  17. On success: update status = 'completed', completed_at = now()
  18. Release lock: DEL job:exec:job-uuid
  19. If callback_url configured: POST result to callback URL

Total latency: execute_at to job start: ~100-500ms
```

### Race Conditions in Job Scheduling

```
Race 1: Duplicate Dispatch (two schedulers grab same job)

  Scheduler A and B are both active (split-brain during leader election,
  or intentional multi-scheduler for throughput):
  
  Scheduler A: ZRANGEBYSCORE → finds job-123
  Scheduler B: ZRANGEBYSCORE → finds job-123 (same scan)
  
  Both try to dispatch → duplicate execution!
  
  Solution: Atomic claim with ZREM
    result = ZREM timer:bucket:X job-123
    If result = 1 → I claimed it → dispatch
    If result = 0 → someone else claimed it → skip
    
    ZREM is atomic in Redis → only ONE scheduler succeeds
    No distributed lock needed for this step
  
  Additional safety: distributed lock on dispatch
    SET job:dispatch:job-123 1 NX EX 60
    Even if ZREM race happens (extremely unlikely), lock prevents double dispatch

Race 2: Timer Bucket Boundary Problem

  Job scheduled at 15:00:59.900 (near bucket boundary)
  Bucket for 15:00 scanned at 15:00:59.500 → job NOT YET DUE
  Bucket for 15:01 scanned at 15:01:00.500 → but job is in 15:00 bucket!
  
  Job falls through the crack — never executed!
  
  Solution: Scanner always checks CURRENT bucket AND PREVIOUS bucket
    current_bucket = floor(now / 60)
    prev_bucket = current_bucket - 60
    Scan both: timer:bucket:{current} AND timer:bucket:{prev}
    
    This ensures no job is missed at bucket boundaries
    Previous bucket may have stragglers that just became due

Race 3: Worker Crash After Processing But Before Status Update

  Worker executes email → success
  Worker crashes before UPDATE status = 'completed'
  Worker lock expires → job re-dispatched → email sent AGAIN
  
  Solution: Idempotent execution
    Email service deduplicates by idempotency_key
    Worker sends: POST /send-email with Idempotency-Key: job-uuid
    If already sent → email service returns 200 (cached response)
    
  Alternative: Two-phase completion
    Worker writes to outbox table (same DB as status update, atomic)
    Background process reads outbox → calls email service
    If email service fails → retry from outbox
    If worker crashes → outbox entry not written → job re-dispatched → correct
```

### Scaling Timer Buckets

```
Problem: 100M scheduled jobs → Redis sorted set becomes massive
  Single ZRANGEBYSCORE on 100M entries → slow

Sharding strategy:
  Shard timer buckets by time range across multiple Redis instances:
    Shard 0: buckets for minutes 0-14 of each hour
    Shard 1: buckets for minutes 15-29
    Shard 2: buckets for minutes 30-44
    Shard 3: buckets for minutes 45-59
  
  Each shard holds ~25M jobs → manageable
  At any given minute, only ONE shard is actively scanned
  
  Alternative: Shard by job_id hash
    ZADD timer:bucket:{minute}:{hash(job_id) % 16} {ts} {job_id}
    16 sub-buckets per minute → parallelizable scanning
    Each scanner instance handles a subset of sub-buckets

Multi-scheduler architecture:
  Run N scheduler instances, each responsible for a subset of buckets
  Partition assignment via consistent hashing or ZooKeeper
  Each scheduler: claim a time range → scan only its buckets
  Leader handles partition assignment + rebalancing on scheduler failure
```

### Timer Approaches: Sorted Set vs Timing Wheel vs DB Polling

```
Redis Sorted Set (this design):
  Insert: O(log N) — ZADD
  Scan due: O(log N + M) — ZRANGEBYSCORE for M due items
  Memory: all jobs in Redis RAM
  ✓ Simple, fast, well-understood
  ✗ Memory-bound: 100M × 200 bytes = 20 GB in Redis
  ✗ Redis becomes SPOF for scheduling
  Best for: < 50M scheduled jobs

Hierarchical Timing Wheel (Kafka, Netty):
  Structure: circular array at each level (seconds, minutes, hours)
  Insert: O(1) — place in correct wheel slot
  Fire: O(1) amortized — current slot fires, cascade from higher wheels
  ✓ O(1) operations, extremely efficient
  ✗ Complex to implement correctly
  ✗ Must be in-process (not easily distributed)
  Best for: In-process timers (network timeouts, connection keepalive)

Database Polling (simplest):
  SELECT * FROM jobs WHERE execute_at <= NOW() AND status = 'scheduled'
  LIMIT 100 FOR UPDATE SKIP LOCKED
  ✓ No Redis dependency
  ✓ PostgreSQL handles durability and locking
  ✗ Polling every second on large table → index pressure
  ✗ Higher latency (up to 1 second polling interval)
  Best for: < 1M jobs, or as fallback when Redis is down

Hybrid (recommended at scale):
  Redis sorted set for next-24-hour jobs (hot jobs, fast scan)
  PostgreSQL for jobs > 24 hours away (cold jobs, large volume)
  Daily migration: move tomorrow's jobs from PG → Redis
  Fallback: if Redis down → DB polling with SKIP LOCKED
```

### At-Most-Once vs At-Least-Once vs Exactly-Once Execution

```
At-Most-Once:
  Dispatch job, delete from timer, don't track completion
  If worker crashes → job lost (never retried)
  ✓ Simple
  ✗ Silent job loss — unacceptable for important work
  Use case: Non-critical analytics, optional notifications

At-Least-Once (this design):
  Dispatch job, track status, retry on failure/timeout
  If worker crashes after execution but before ACK → re-executed
  ✓ No job loss
  ✗ May execute twice → workers MUST be idempotent
  Use case: Most jobs (email, webhook, data processing)

Exactly-Once (hardest):
  Requires transactional processing:
    1. Read job from Kafka in a transaction
    2. Execute job
    3. Commit Kafka offset + write result to DB atomically
  Only possible if job result and offset commit are in same DB
  ✓ Perfect deduplication
  ✗ Complex, tight coupling between queue and result store
  Use case: Financial operations, inventory updates
  
  Practical exactly-once: at-least-once + idempotent workers
    Worker checks: "have I processed job-uuid before?"
    If yes → return cached result
    If no → process, store result with job-uuid as key
    This is simpler than transactional exactly-once and works in practice
```

