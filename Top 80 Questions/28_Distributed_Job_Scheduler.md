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
┌──────────────┐
│  Clients     │  ← Services submitting jobs via API
│ (Services)   │
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │
└──────┬───────┘
       │
┌──────▼───────────────────────────────────────┐
│            Job Scheduler Service              │
│                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Job     │  │ Schedule │  │  Job     │   │
│  │  Store   │  │ Manager  │  │  Dispatch│   │
│  │  (CRUD)  │  │ (Timer)  │  │  er     │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼──────────────┼─────────────┼──────────┘
        │              │             │
┌───────▼──────┐ ┌─────▼──────┐ ┌───▼──────────┐
│  PostgreSQL  │ │  Redis     │ │   Kafka      │
│  (Job Meta,  │ │ (Timer     │ │ (Job Queue)  │
│   History)   │ │  Bucket,   │ │              │
│              │ │  Locks)    │ │              │
└──────────────┘ └────────────┘ └───┬──────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
             ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
             │  Worker     │ │  Worker     │ │  Worker     │
             │  Pool 1     │ │  Pool 2     │ │  Pool N     │
             │ (Executors) │ │ (Executors) │ │ (Executors) │
             └─────────────┘ └─────────────┘ └─────────────┘

┌──────────────┐
│  ZooKeeper / │  ← Leader election for schedule managers
│  etcd        │
└──────────────┘
```

### Component Deep Dive

#### Schedule Manager — The Core Scheduling Engine

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
- Each bucket is a list in Redis: `bucket:{minute_timestamp}` → list of job_ids
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

#### Job Dispatcher
- Receives due jobs from Schedule Manager
- Publishes job execution request to Kafka (partitioned by job_type or priority)
- **Priority handling**: Separate Kafka topics per priority
  - `jobs.critical` → dedicated high-priority worker pool
  - `jobs.high` → standard pool
  - `jobs.normal` → standard pool
  - `jobs.low` → low-priority pool (processed when others are idle)

#### Worker Pool (Executors)
- Kafka consumers that execute jobs
- Each worker:
  1. Consume job from Kafka
  2. Set a distributed lock: `SET job:lock:{job_id} {worker_id} NX EX {timeout}`
  3. Execute the job (call the target service, run the script, etc.)
  4. Update job status in DB: `completed` or `failed`
  5. If failed → check retry policy → re-enqueue with backoff
  6. Release the lock

#### Recurring Job Handling
- On completion of a recurring job:
  1. Compute next execution time from cron expression
  2. Create a new job entry with the next execution time
  3. Insert into the timer (Redis sorted set or time bucket)
- **Library**: Parse cron expressions with a library (e.g., `croniter` in Python)
- **Example**: `"0 */5 * * *"` → every 5 minutes

#### Exactly-Once Execution
- **Challenge**: If scheduler crashes after dispatching but before marking job as dispatched, it might dispatch again on recovery
- **Solution**:
  1. **Distributed lock**: Before dispatching, acquire lock on job_id (Redis SET NX)
  2. **Idempotent execution**: Workers check if job already executed (DB status check)
  3. **Fencing token**: Each dispatch includes a monotonically increasing token. Worker accepts only if token matches expected

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

### PostgreSQL — Job Execution History

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
```

### Redis — Timer Buckets & Locks

```
# Time bucket (jobs due in this minute)
Key:    timer:bucket:{minute_epoch}
Type:   Sorted Set
Members: job_id
Scores:  exact_timestamp_seconds

# Job execution lock
Key:    job:lock:{job_id}
Value:  {worker_id}
TTL:    job_timeout_seconds × 2

# Recurring job next fire time
Key:    recurring:next:{recurring_id}
Value:  next_execute_timestamp
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
```
Job A ──┬──▶ Job C ──▶ Job E
        │
Job B ──┘

Implementation:
- Job C has: depends_on = [Job A, Job B]
- When Job A completes → check if all dependencies of Job C are complete
- If yes → schedule Job C for execution
- Use a "dependency counter" in Redis: DECR deps:{job_C} for each completed dependency
- When counter reaches 0 → dispatch Job C
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

