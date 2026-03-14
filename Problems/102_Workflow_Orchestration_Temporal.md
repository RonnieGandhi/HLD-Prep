# 102. Design a Workflow Orchestration Engine (like Temporal / Cadence)

---

## 1. Functional Requirements (FR)

- Execute **long-running, multi-step workflows** that can span seconds to months
- **Durable execution**: workflow state survives process crashes, server restarts, and deployments — code resumes exactly where it left off
- **Activity tasks**: individual units of work (API call, DB write, file upload) executed by workers
- **Timers and sleep**: `workflow.sleep(Duration.ofDays(30))` — workflow pauses for 30 days, resumes automatically
- **Retry policies**: automatic retry of failed activities with configurable backoff, max attempts, non-retryable errors
- **Saga pattern**: compensating transactions for distributed rollback (e.g., cancel hotel if flight booking fails)
- **Child workflows**: compose workflows hierarchically (order workflow → payment sub-workflow → shipping sub-workflow)
- **Signals and queries**: send events TO a running workflow (signal) or read its state (query) without affecting execution
- **Cron/scheduled workflows**: periodic execution with guaranteed exactly-once per schedule
- **Versioning**: update workflow code without breaking in-flight executions
- **Visibility**: search, filter, and monitor running/completed workflows by custom attributes

---

## 2. Non-Functional Requirements (NFRs)

- **Durability**: Zero state loss — workflow state survives any infrastructure failure
- **Scalability**: 100K+ concurrent workflows, 10K+ workflow starts/sec
- **Low Latency**: Activity dispatch < 50ms; workflow decision < 10ms
- **Exactly-Once Execution**: Each activity/workflow step executed exactly once (even with retries and crashes)
- **High Availability**: 99.99% — workflow engine unavailability blocks all business processes
- **Multi-Tenancy**: Namespace isolation for teams; quota per namespace
- **Observability**: Full audit trail of every workflow state transition

---

## 3. Capacity Estimations

| Metric | Calculation | Value |
|---|---|---|
| Concurrent workflows | | 100K |
| Workflow starts / sec | | 10K |
| Activity dispatches / sec | | 50K |
| Avg activities per workflow | | 10 |
| Avg workflow duration | | 5 minutes (some: months) |
| Event history per workflow | avg 50 events × 2 KB | 100 KB |
| Total history storage | 10K starts/sec × 86400 × 100 KB | ~86 TB/day |
| History retention | 30 days | ~2.5 PB |
| Workflow workers | | 500 |
| Activity workers | | 2,000 |
| History service nodes | | 20 (sharded by workflow ID) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT APPLICATIONS                                  │
│                                                                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │  Order Service   │  │  Payment Service │  │  Onboarding Svc  │             │
│  │                  │  │                  │  │                  │             │
│  │  Start workflow: │  │  Start workflow: │  │  Start workflow: │             │
│  │  ProcessOrder    │  │  ChargePayment   │  │  UserOnboarding  │             │
│  │  (orderId,       │  │  (paymentId,     │  │  (userId,        │             │
│  │   items, addr)   │  │   amount, card)  │  │   email)         │             │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘             │
│           │                     │                      │                       │
│           └─────────────────────┼──────────────────────┘                       │
│                                 │                                              │
└─────────────────────────────────│──────────────────────────────────────────────┘
                                  │ gRPC / SDK
┌─────────────────────────────────│──────────────────────────────────────────────┐
│                    TEMPORAL SERVER                                              │
│                                 │                                              │
│  ┌──────────────────────────────▼──────────────────────────────────────────┐   │
│  │                    FRONTEND SERVICE                                      │   │
│  │                                                                          │   │
│  │  - gRPC API gateway (all client requests)                               │   │
│  │  - Request validation, auth, rate limiting                              │   │
│  │  - Routes to correct History Service shard by workflow ID               │   │
│  │  - Stateless, horizontally scalable (behind LB)                         │   │
│  └──────────────────────────────┬──────────────────────────────────────────┘   │
│                                 │                                              │
│  ┌──────────────────────────────▼──────────────────────────────────────────┐   │
│  │                    HISTORY SERVICE (Sharded)                             │   │
│  │                                                                          │   │
│  │  Shard 0          Shard 1          Shard 2          Shard N             │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐      │   │
│  │  │            │   │            │   │            │   │            │      │   │
│  │  │ Workflow   │   │ Workflow   │   │ Workflow   │   │ Workflow   │      │   │
│  │  │ State      │   │ State      │   │ State      │   │ State      │      │   │
│  │  │ Machine    │   │ Machine    │   │ Machine    │   │ Machine    │      │   │
│  │  │            │   │            │   │            │   │            │      │   │
│  │  │ - Event    │   │ - Event    │   │ - Event    │   │ - Event    │      │   │
│  │  │   sourced  │   │   sourced  │   │   sourced  │   │   sourced  │      │   │
│  │  │ - Append   │   │ - Append   │   │ - Append   │   │ - Append   │      │   │
│  │  │   events   │   │   events   │   │   events   │   │   events   │      │   │
│  │  │ - Schedule │   │ - Schedule │   │ - Schedule │   │ - Schedule │      │   │
│  │  │   timers   │   │   timers   │   │   timers   │   │   timers   │      │   │
│  │  │ - Dispatch │   │ - Dispatch │   │ - Dispatch │   │ - Dispatch │      │   │
│  │  │   tasks    │   │   tasks    │   │   tasks    │   │   tasks    │      │   │
│  │  └────────────┘   └────────────┘   └────────────┘   └────────────┘      │   │
│  │                                                                          │   │
│  │  Shard assignment: hash(namespace + workflowId) % numShards             │   │
│  │  Each shard: owns and manages a subset of workflows                     │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                    MATCHING SERVICE                                       │   │
│  │                                                                          │   │
│  │  Task Queues (one per workflow type + activity type):                    │   │
│  │                                                                          │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐             │   │
│  │  │ workflow-task:  │  │ activity-task: │  │ activity-task: │             │   │
│  │  │ ProcessOrder   │  │ ChargeCard     │  │ SendEmail      │             │   │
│  │  │                │  │                │  │                │             │   │
│  │  │ [wf-exec-1]    │  │ [charge-42]    │  │ [email-99]     │             │   │
│  │  │ [wf-exec-7]    │  │ [charge-43]    │  │ [email-100]    │             │   │
│  │  │ [wf-exec-12]   │  │                │  │ [email-101]    │             │   │
│  │  └────────┬───────┘  └────────┬───────┘  └────────┬───────┘             │   │
│  │           │                   │                    │                     │   │
│  │  Workers poll tasks from matching service via long-poll gRPC            │   │
│  └───────────│───────────────────│────────────────────│─────────────────────┘   │
│              │                   │                    │                         │
│  ┌───────────▼───────────────────▼────────────────────▼─────────────────────┐   │
│  │                    PERSISTENCE (Cassandra / PostgreSQL / MySQL)           │   │
│  │                                                                          │   │
│  │  Tables:                                                                 │   │
│  │  - executions:     (namespace, workflow_id, run_id) → current state     │   │
│  │  - history_events: (namespace, workflow_id, run_id, event_id) → event   │   │
│  │  - visibility:     (namespace, workflow_type, status, start_time, ...)   │   │
│  │  - timers:         (visibility_ts, namespace, workflow_id) → fire time  │   │
│  │  - tasks:          (namespace, task_queue, task_id) → pending task      │   │
│  │                                                                          │   │
│  │  Cassandra recommended for scale (100K+ wf/sec)                         │   │
│  │  PostgreSQL sufficient for < 10K wf/sec                                 │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                    VISIBILITY (Elasticsearch)                            │   │
│  │                                                                          │   │
│  │  Search running/completed workflows by:                                  │   │
│  │  - workflow type, status, start time, close time                        │   │
│  │  - Custom search attributes (order_id, customer_tier, error_code)       │   │
│  │  - Full-text search on workflow memo                                     │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
                    │                   │                    │
┌───────────────────│───────────────────│────────────────────│────────────────────┐
│                   │      WORKER FLEET │                    │                    │
│                   │                   │                    │                    │
│  ┌────────────────▼──┐  ┌─────────────▼──┐  ┌─────────────▼──┐                │
│  │ Workflow Worker   │  │ Activity Worker│  │ Activity Worker│                │
│  │ (decision maker)  │  │ (task executor)│  │ (task executor)│                │
│  │                   │  │                │  │                │                │
│  │ - Polls workflow  │  │ - Polls        │  │ - Polls        │                │
│  │   task queue      │  │   activity     │  │   activity     │                │
│  │ - Replays event   │  │   task queue   │  │   task queue   │                │
│  │   history to      │  │ - Executes     │  │ - Executes     │                │
│  │   rebuild state   │  │   activity code│  │   activity code│                │
│  │ - Executes        │  │   (API call,   │  │   (DB write,   │                │
│  │   workflow code   │  │    DB query)   │  │    file upload)│                │
│  │ - Emits commands: │  │ - Returns      │  │ - Returns      │                │
│  │   ScheduleActivity│  │   result to    │  │   result to    │                │
│  │   StartTimer      │  │   server       │  │   server       │                │
│  │   StartChild      │  │ - Heartbeats   │  │ - Heartbeats   │                │
│  │   CompleteWorkflow│  │   for long     │  │   for long     │                │
│  │                   │  │   tasks        │  │   tasks        │                │
│  └───────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                                │
│  Workers are STATELESS — all state is in the Temporal server                  │
│  Workers can be scaled independently per task queue                           │
│  Workflow workers: CPU-bound (replay history)                                 │
│  Activity workers: I/O-bound (external calls)                                 │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### How Durable Execution Works (The Core Innovation)

```
Traditional approach (stateless service):
  function processOrder(order) {
    payment = chargeCard(order)         // ← crashes here
    shipment = createShipment(order)    // never reached
    sendConfirmation(order)             // never reached
  }
  // On crash: partial state, manual recovery needed

Temporal approach (durable execution):
  function processOrder(order) {
    payment = workflow.executeActivity(ChargeCard, order)     // Step 1
    shipment = workflow.executeActivity(CreateShipment, order) // Step 2
    workflow.executeActivity(SendConfirmation, order)          // Step 3
  }
  // On crash after step 1: Temporal replays history, skips ChargeCard (already done),
  // resumes at step 2 (CreateShipment)

HOW THIS WORKS INTERNALLY:
  1. Workflow worker polls workflow task queue → receives ProcessOrder task
  2. Worker replays event history:
     Event 1: WorkflowExecutionStarted (input: order)
     Event 2: ActivityTaskScheduled (ChargeCard)
     Event 3: ActivityTaskCompleted (result: {payment_id: "pay_123"})
     ← History ends here — means ChargeCard is done but CreateShipment hasn't started
  3. Worker re-executes workflow code from the beginning:
     chargeCard(order) → ALREADY IN HISTORY → return cached result "pay_123"
     createShipment(order) → NOT IN HISTORY → this is a NEW command
  4. Worker returns command: ScheduleActivityTask(CreateShipment, order)
  5. Server appends: Event 4: ActivityTaskScheduled (CreateShipment)
  6. Activity worker picks up CreateShipment, executes, returns result
  7. Server appends: Event 5: ActivityTaskCompleted (result: {shipment_id: "ship_456"})
  8. Workflow worker gets new task → replays all 5 events → reaches SendConfirmation
  9. Cycle continues until workflow completes

CRITICAL RULE: Workflow code must be DETERMINISTIC
  No random(), no System.currentTimeMillis(), no direct I/O
  All side effects must go through activities
  Temporal SDK provides: workflow.now(), workflow.random(), etc. (deterministic replicas)
```

#### Event History (Event-Sourced State)

```
Every workflow execution is an append-only event log:

  Event 1:  WorkflowExecutionStarted     {input: {orderId: "42", items: [...]}}
  Event 2:  WorkflowTaskScheduled        {}
  Event 3:  WorkflowTaskCompleted        {commands: [ScheduleActivity(ChargeCard)]}
  Event 4:  ActivityTaskScheduled        {activityType: "ChargeCard", input: {...}}
  Event 5:  ActivityTaskStarted          {workerId: "worker-7", attempt: 1}
  Event 6:  ActivityTaskCompleted        {result: {paymentId: "pay_123"}}
  Event 7:  WorkflowTaskScheduled        {}
  Event 8:  WorkflowTaskCompleted        {commands: [ScheduleActivity(CreateShipment)]}
  Event 9:  TimerStarted                 {duration: 30 days, timerId: "1"}
  ...
  Event 42: TimerFired                   {timerId: "1"}
  Event 43: WorkflowTaskCompleted        {commands: [CompleteWorkflow]}
  Event 44: WorkflowExecutionCompleted   {result: {status: "success"}}

This log IS the workflow state. No separate database of "current state."
To know what step the workflow is on → replay all events.
```

#### Saga Pattern (Distributed Compensation)

```
Problem: Book a trip = Flight + Hotel + Car Rental (3 services, no distributed TX)
  If car rental fails after flight and hotel are booked → must undo

Temporal Saga:
  function bookTrip(trip) {
    flight = workflow.executeActivity(BookFlight, trip)
    try {
      hotel = workflow.executeActivity(BookHotel, trip)
    } catch (e) {
      workflow.executeActivity(CancelFlight, flight)  // compensate
      throw e
    }
    try {
      car = workflow.executeActivity(RentCar, trip)
    } catch (e) {
      workflow.executeActivity(CancelHotel, hotel)    // compensate
      workflow.executeActivity(CancelFlight, flight)  // compensate
      throw e
    }
    return {flight, hotel, car}
  }

  Why this is better than choreography (event-driven):
    - Compensation logic is in ONE place (not scattered across services)
    - Easy to reason about (sequential code, not event chains)
    - Temporal guarantees compensation runs even if workflow worker crashes
    - Full visibility: see exactly which step failed and which compensations ran
```

---

## 5. APIs

### Temporal Client SDK (Go example)

```go
// Start a workflow
we, err := client.ExecuteWorkflow(ctx, client.StartWorkflowOptions{
    ID:        "order-42",
    TaskQueue: "order-processing",
    RetryPolicy: &temporal.RetryPolicy{
        MaximumAttempts: 3,
    },
}, ProcessOrder, orderInput)

// Query workflow state (synchronous, doesn't affect execution)
var status OrderStatus
err = client.QueryWorkflow(ctx, "order-42", "", "getStatus", &status)

// Signal a running workflow (send event)
err = client.SignalWorkflow(ctx, "order-42", "", "cancelOrder", cancelReason)

// Wait for workflow result
var result OrderResult
err = we.Get(ctx, &result)
```

### Temporal Server gRPC API

```
StartWorkflowExecution          → Start new workflow
SignalWorkflowExecution         → Send signal to running workflow
QueryWorkflow                   → Query running workflow state
TerminateWorkflowExecution      → Force-terminate workflow
RequestCancelWorkflowExecution  → Request graceful cancellation
ListWorkflowExecutions          → Search workflows (via visibility)
GetWorkflowExecutionHistory     → Full event history
PollWorkflowTaskQueue           → Worker long-poll for workflow tasks
PollActivityTaskQueue           → Worker long-poll for activity tasks
RespondWorkflowTaskCompleted    → Worker returns workflow decision
RespondActivityTaskCompleted    → Worker returns activity result
RespondActivityTaskFailed       → Worker reports activity failure
RecordActivityTaskHeartbeat     → Worker heartbeat for long activities
```

---

## 6. Data Models

### Cassandra (History Store — Recommended for Scale)

```sql
-- Execution state (current mutable state)
CREATE TABLE executions (
    namespace_id   UUID,
    workflow_id    TEXT,
    run_id         UUID,
    state          INT,           -- running, completed, failed, cancelled, terminated
    next_event_id  BIGINT,        -- next sequence number
    workflow_type  TEXT,
    task_queue     TEXT,
    start_time     TIMESTAMP,
    close_time     TIMESTAMP,
    memo           BLOB,          -- user-defined searchable metadata
    PRIMARY KEY ((namespace_id, workflow_id), run_id)
);

-- Event history (append-only, event-sourced)
CREATE TABLE history_events (
    namespace_id   UUID,
    workflow_id    TEXT,
    run_id         UUID,
    event_id       BIGINT,        -- monotonically increasing per workflow
    event_type     INT,           -- WorkflowStarted, ActivityCompleted, TimerFired, etc.
    timestamp      TIMESTAMP,
    data           BLOB,          -- serialized event payload (protobuf)
    PRIMARY KEY ((namespace_id, workflow_id, run_id), event_id)
) WITH CLUSTERING ORDER BY (event_id ASC);

-- Pending tasks (consumed by workers via long-poll)
CREATE TABLE tasks (
    namespace_id   UUID,
    task_queue     TEXT,
    task_type      INT,           -- workflow_task or activity_task
    task_id        BIGINT,
    workflow_id    TEXT,
    run_id         UUID,
    scheduled_time TIMESTAMP,
    PRIMARY KEY ((namespace_id, task_queue, task_type), task_id)
);

-- Timers (scanned by timer processor)
CREATE TABLE timers (
    namespace_id   UUID,
    fire_time      TIMESTAMP,
    workflow_id    TEXT,
    run_id         UUID,
    timer_id       TEXT,
    PRIMARY KEY ((namespace_id), fire_time, workflow_id, timer_id)
) WITH CLUSTERING ORDER BY (fire_time ASC);
```

### Elasticsearch (Visibility — Search & Filter)

```json
{
  "namespace": "production",
  "workflow_id": "order-42",
  "run_id": "abc-123-def",
  "workflow_type": "ProcessOrder",
  "status": "Running",
  "start_time": "2026-03-14T10:00:00Z",
  "task_queue": "order-processing",
  "custom_attributes": {
    "customer_id": "cust-789",
    "order_total": 299.99,
    "customer_tier": "premium"
  }
}

// Query: Find all failed premium orders in last 24h
// WorkflowType = 'ProcessOrder' AND Status = 'Failed'
//   AND CustomAttributes.customer_tier = 'premium'
//   AND StartTime > now() - 24h
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Workflow worker crash** | Workflow task times out → server re-dispatches → new worker replays history and resumes |
| **Activity worker crash** | Activity task times out (heartbeat or schedule-to-close) → server retries per retry policy |
| **Server node crash** | History shards rebalanced to surviving nodes; all state in Cassandra (durable) |
| **Database unavailable** | Server queues writes; client retries with backoff; no state lost |
| **Long activity timeout** | Activity heartbeat: worker sends heartbeat every N seconds → server detects dead worker if heartbeat stops |
| **Poison pill workflow** | Workflow error → retry with backoff → max retries → fail → dead-letter handling |
| **Versioning during deployment** | Workflow.getVersion() → run old code for in-flight workflows, new code for new ones |
| **Timer persistence** | Timer stored in DB; timer processor scans for due timers → fires even if server restarted |

### Exactly-Once Activity Execution

```
Problem: Activity worker completes work (charges credit card) but crashes before ACK
  Server retries → credit card charged TWICE

Solution (Temporal's approach):
  Temporal guarantees at-most-once DISPATCH (via unique activity task token)
  Activity itself should be IDEMPOTENT:
    Use idempotency key: chargeCard(paymentId="pay_42", amount=99.99)
    Payment gateway: if pay_42 already processed → return cached result
  
  Combined: at-most-once dispatch + idempotent activity = exactly-once effect

For non-idempotent activities:
  Use activity heartbeat + short timeout
  If heartbeat stops → assume worker dead → retry
  If worker was actually alive (network partition) → detect via task token mismatch → cancel duplicate
```

### Workflow Versioning (Zero-Downtime Deploys)

```
Problem: 10,000 workflows running processOrder v1.
  Deploy v2 (added new step). How to handle in-flight v1 workflows?

Solution: Temporal's getVersion()
  function processOrder(order) {
    payment = executeActivity(ChargeCard, order)
    
    version = workflow.getVersion("add-fraud-check", 1, 2)  // min=1, max=2
    if (version >= 2) {
      executeActivity(FraudCheck, order)  // NEW step in v2
    }
    
    shipment = executeActivity(CreateShipment, order)
  }

  In-flight v1 workflows: getVersion returns 1 → skip FraudCheck → continue as before
  New workflows: getVersion returns 2 → run FraudCheck → new behavior
  
  Eventually all v1 workflows complete → remove version check in v3 (cleanup)
```

---

## 8. Additional Considerations

### When to Use Temporal vs Other Approaches

```
| Approach | Use When | Don't Use When |
|---|---|---|
| Temporal/Cadence | Multi-step, long-running, needs retry/compensation | Simple request-response |
| Kafka + Consumers | Event-driven, fan-out, decoupled services | Need request-response or saga |
| Step Functions | AWS-native, < 25K workflows/sec, simple DAGs | High throughput, complex logic |
| Database + cron | Simple periodic jobs, < 100 jobs | Complex state, error handling |
| Choreography (events) | Loose coupling, simple flows | Complex compensation, visibility |

Temporal shines when:
  1. Flow has > 3 steps across services
  2. Steps can fail and need retry/compensation
  3. Flow spans minutes to months (timers, human approval)
  4. You need visibility into every step of every execution
  5. You want workflow logic in CODE, not YAML/JSON (Step Functions)
```

### Production Patterns

```
1. Order Processing: validate → reserve inventory → charge → ship → notify
   On failure at any step → compensate all previous steps (saga)

2. User Onboarding: create account → send welcome email →
   wait 3 days → send tutorial → wait 7 days → send offer
   Entire flow is ONE workflow with timers

3. Subscription Billing: charge monthly →
   if failed → retry 3 times over 7 days → if still failed → cancel subscription
   Handles the ENTIRE lifecycle in code

4. Data Pipeline: extract → transform → validate → load → notify
   If transform fails → retry → if still fails → alert on-call → wait for signal → resume

5. Human-in-the-Loop: submit expense → wait for manager approval (signal) →
   if approved → reimburse → if rejected → notify employee
   Timer: if no approval in 7 days → escalate to VP
```

### Temporal vs Saga with Events (Choreography vs Orchestration)

```
CHOREOGRAPHY (event-driven):
  OrderService → emits OrderCreated
  PaymentService → listens → charges card → emits PaymentCompleted
  ShipmentService → listens → creates shipment → emits ShipmentCreated
  NotificationService → listens → sends email

  Problems:
    - Hard to see the full flow (logic scattered across services)
    - Compensation: who owns the "undo" logic? Every service must handle it
    - Debugging: correlate events across services with correlation IDs
    - Ordering: what if ShipmentCreated arrives before PaymentCompleted?

ORCHESTRATION (Temporal):
  OrderWorkflow:
    payment = executeActivity(ChargeCard)
    shipment = executeActivity(CreateShipment)
    executeActivity(SendConfirmation)

  Advantages:
    - Flow visible in ONE place (the workflow code)
    - Compensation = simple try/catch in the workflow
    - Full history in Temporal UI (every step, timing, inputs/outputs)
    - Ordering guaranteed by sequential code execution
    - Testing: unit test the workflow like any function

RECOMMENDATION: Use orchestration (Temporal) for CRITICAL business flows
  Use choreography for DECOUPLED notifications, analytics, side effects
```

### Why This Problem Matters

```
Temporal/Cadence is used by:
  - Uber: trip lifecycle, driver onboarding, pricing
  - Netflix: media encoding pipeline, content delivery
  - Stripe: payment processing, refund orchestration
  - Snap: ad delivery, content moderation
  - Coinbase: crypto transactions, compliance workflows

For senior/staff engineers:
  - Understand durable execution model (event-sourced replay)
  - Know when to use orchestration vs choreography
  - Saga pattern implementation (compensation != rollback)
  - Workflow versioning for zero-downtime deploys
  - Activity idempotency design
  - Timer management at scale (millions of pending timers)
```

