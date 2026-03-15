# Consistency Models, Consensus & Distributed Coordination

Core concept tested in HLD rounds: "What consistency does your system provide? How do you handle network partitions? Why not just use strong consistency everywhere?"

---

## 1. CAP Theorem — The Foundation

```
You can have at most TWO of three:
  Consistency (C): Every read returns the latest write
  Availability (A): Every request gets a response (no errors/timeouts)
  Partition tolerance (P): System works despite network partitions

Since network partitions WILL happen in distributed systems:
  You really choose between CP and AP

CP (Consistent + Partition-tolerant):
  During partition: refuse requests to avoid stale reads
  Examples: ZooKeeper, etcd, HBase, Spanner, CockroachDB
  Use for: leader election, distributed locks, financial balances

AP (Available + Partition-tolerant):
  During partition: serve requests with possibly stale data
  Examples: Cassandra, DynamoDB, DNS, CDN caches
  Use for: social feeds, product catalogs, user presence

In practice: Most systems are neither pure CP nor pure AP
  They offer TUNABLE consistency (Cassandra: W + R > N = CP, W=1 R=1 = AP)
```

---

## 2. Consistency Models — Spectrum from Weakest to Strongest

```
Weakest ─────────────────────────────────────────────── Strongest

Eventual     Causal      Read-Your-   Sequential   Linearizable
Consistency  Consistency  Writes       Consistency  (Strong)

                         "I see my     "Everyone    "Single global
"All replicas  "If A      own writes   sees ops     ordering, as if
 converge      caused B,   immediately" in the       there's one copy
 eventually"   everyone               same order"   of the data"
               sees A                
               before B"
```

### Eventual Consistency

```
Guarantee: If no new writes, all replicas will EVENTUALLY have the same value.
No guarantee about WHEN or what you read in the meantime.

Example:
  T=0: Write "Bob" to replica A
  T=1: Read from replica B → might get "Alice" (old value) or "Bob" (new value)
  T=5: Read from replica B → "Bob" (converged)

Used by: DNS, CDN caches, Cassandra (W=1, R=1), DynamoDB (default reads)
Acceptable for: User profiles, product catalog, social feed, recommendation lists

NOT acceptable for: Bank balance, inventory count, distributed lock state
```

### Causal Consistency

```
Guarantee: If event A CAUSED event B, everyone sees A before B.
Concurrent events (no causal relation) can be seen in any order.

Example:
  Alice posts: "I got promoted!" (event A)
  Alice comments on her own post: "Thanks everyone!" (event B, caused by A)
  
  Causal consistency guarantees: everyone sees post before the comment
  Without causal consistency: some users might see the comment before the post

How: Attach logical timestamps (Lamport/vector clocks) to events
  Each event carries the causal dependencies it builds upon
  Replica delays delivery of event B until it has seen event A

Used by: MongoDB (causal sessions), COPS
```

### Read-Your-Writes Consistency

```
Guarantee: After you write, YOUR subsequent reads will see that write.
Other users may still see stale data.

Example:
  User updates profile name to "Bob"
  User refreshes page → must see "Bob" (not old "Alice")
  
  Another user viewing the profile might still see "Alice" for a few seconds

Implementation: Route user's reads to the primary (or to a replica that is caught up
to the user's last write position via LSN tracking)

Used by: Most production systems as a minimum guarantee
```

### Linearizability (Strong Consistency)

```
Guarantee: All operations appear to execute atomically at some point between
invocation and completion. Everyone sees the SAME ORDER.

Behaves as if there is a single copy of the data.
Once a write is acknowledged, all subsequent reads (from any client) see it.

Example:
  T=0: Client A writes balance = 100
  T=1: Client A gets ACK
  T=2: Client B reads balance → MUST return 100 (not stale value)

Cost: Every write requires consensus (Raft/Paxos) → high latency
  Cross-region linearizability: ~100-300ms per write (speed of light limitation)

Used by: ZooKeeper, etcd, Spanner (TrueTime), CockroachDB
Use for: Leader election, distributed locks, compare-and-swap operations
```

---

## 3. Consensus Algorithms — How Nodes Agree

### Raft ⭐ (Most Common in Practice)

```
Purpose: Ensure all nodes agree on the same sequence of operations

Roles:
  Leader: accepts all writes, replicates to followers
  Follower: replicates from leader, participates in elections
  Candidate: node trying to become leader (during election)

Normal operation:
  1. Client sends write to Leader
  2. Leader appends to its log
  3. Leader sends AppendEntries RPC to all followers
  4. Followers append to their logs → ACK
  5. When MAJORITY (N/2 + 1) ACKs → Leader commits entry
  6. Leader applies to state machine → responds to client
  7. Followers learn of commit → apply to their state machines

Leader election:
  1. Follower doesn't hear from leader for election_timeout (150-300ms random)
  2. Becomes Candidate, increments term, votes for itself
  3. Sends RequestVote to all nodes
  4. Wins if gets MAJORITY of votes
  5. Becomes Leader, starts sending heartbeats
  
  Split-vote prevention: randomized election timeout
  Term number: monotonically increasing epoch — stale leaders rejected

Safety guarantees:
  • At most ONE leader per term
  • Leader has ALL committed entries (election restriction)
  • Committed entries are never overwritten

Used by: etcd, CockroachDB, TiKV, Consul, MongoDB (similar protocol)
Practical limits: 3 or 5 nodes (7 max). Latency = 1 network round-trip for majority.
```

### Paxos (Theoretical Foundation)

```
Same purpose as Raft but harder to understand and implement.
Paxos predates Raft and is the theoretical foundation.

Roles: Proposer, Acceptor, Learner

Phase 1 (Prepare):
  Proposer picks a proposal number N
  Sends PREPARE(N) to majority of acceptors
  Acceptors promise: "I won't accept proposals < N"
  Acceptors respond with any previously accepted value

Phase 2 (Accept):
  If proposer gets majority promises:
    Sends ACCEPT(N, value) to acceptors
    Acceptors: if no higher promise → accept
    If majority accept → value is CHOSEN

In practice: Multi-Paxos optimizes for steady-state (skip Phase 1 when leader is stable)
  → Essentially becomes Raft-like

Used by: Google Spanner (internally), Google Chubby
In interviews: Say "Raft" — equivalent for practical purposes, easier to explain
```

---

## 4. ZooKeeper / etcd — Distributed Coordination

```
These services provide strongly consistent coordination primitives:

  • Linearizable key-value store
  • Leader election
  • Distributed locks
  • Watch notifications (notify when a key changes)
  • Ephemeral nodes (disappear when client disconnects — presence detection)
  • Sequential nodes (auto-incrementing, useful for queue ordering)

Use for:
  ✓ Service discovery (which instances are alive?)
  ✓ Leader election (who is the primary DB / scheduler?)
  ✓ Distributed locks (only one worker processes this job)
  ✓ Configuration management (feature flags, cluster config)
  ✓ Cluster membership (which nodes are in the Kafka cluster?)

Do NOT use for:
  ✗ High-throughput data storage (ZK handles ~10K writes/sec, not 100K)
  ✗ Large values (ZK max node size: 1 MB)
  ✗ Application data (it's for metadata/coordination only)

ZooKeeper vs etcd vs Consul:
  ZooKeeper: Tree structure (znodes), Java, used by Kafka/HBase/Hadoop
  etcd: Flat key-value, Go, used by Kubernetes/CockroachDB
  Consul: Service mesh + KV store, Go, used for service discovery
  
  All three provide equivalent consensus guarantees (Raft/ZAB)
  Choose based on ecosystem fit
```

### Leader Election Pattern

```
How to elect a leader using etcd:

  1. All candidates try to create the same key with a lease:
     etcdctl put /leader/scheduler "node-A" --lease=abc123
     
  2. Only ONE succeeds (atomic compare-and-swap)
     Winner: "I am the leader" → starts doing leader work
     Losers: watch the key → wait for it to disappear
     
  3. Leader must keep renewing the lease (heartbeat every 10s)
     etcdctl lease keep-alive abc123
     
  4. If leader crashes → lease expires → key disappears
     → One of the watchers acquires the key → becomes new leader

This is how Patroni (PostgreSQL HA), Kafka controller, and scheduler
leaders are elected in production systems.
```

### Distributed Lock Pattern

```
Using etcd/ZooKeeper for distributed locks:

  ACQUIRE:
    Create ephemeral key: /locks/resource-X = "owner-A" (with TTL / session)
    If key already exists → lock is held → wait (watch for deletion)
    
  RELEASE:
    Delete the key: /locks/resource-X
    
  AUTO-RELEASE:
    If lock holder crashes → session ends → ephemeral node disappears
    → No stale locks (unlike Redis where TTL may be too long or too short)

  ✓ Stronger than Redis locks (consensus-backed, no split-brain)
  ✗ Higher latency (~10ms vs ~1ms for Redis)
  ✗ Lower throughput (not for high-frequency locking)

When to use ZK/etcd locks vs Redis locks:
  Redis locks: High frequency, best-effort mutual exclusion (rate limiters, caching)
  ZK/etcd locks: Critical sections where correctness matters (leader election, 
    schema migrations, one-time job execution)
```

---

## 5. Distributed Transactions

### Two-Phase Commit (2PC)

```
Coordinator asks participants: "Can you commit?"

Phase 1 (Prepare/Vote):
  Coordinator → all participants: PREPARE
  Each participant: write to local WAL, lock resources → vote YES or NO
  
Phase 2 (Commit/Abort):
  If ALL voted YES → Coordinator: COMMIT → participants commit
  If ANY voted NO  → Coordinator: ABORT → participants rollback

Problems:
  ✗ Blocking: If coordinator crashes after PREPARE but before COMMIT/ABORT,
    participants are STUCK — holding locks, waiting for a decision that never comes
  ✗ Slow: 2 network round-trips minimum
  ✗ Availability: ALL participants must be up
  
  Rarely used directly in microservices. Used internally by databases (XA transactions).
```

### Saga Pattern ⭐ (Preferred for Microservices)

```
Break a distributed transaction into a sequence of local transactions,
each with a compensating action for rollback.

Place Order Saga:
  Step 1: Order Service    → CREATE ORDER (status: pending)
  Step 2: Inventory Service → RESERVE STOCK
  Step 3: Payment Service   → CHARGE CARD
  Step 4: Order Service    → CONFIRM ORDER (status: confirmed)

If Step 3 fails (payment declined):
  Compensate Step 2: RELEASE STOCK
  Compensate Step 1: CANCEL ORDER (status: cancelled)

Orchestration (central coordinator) ⭐:
  Saga Orchestrator (service or Temporal workflow) tells each service what to do next
  ✓ Clear flow, easy to debug and monitor
  ✓ Centralized error handling and compensation
  ✗ Single point of coordination (mitigate: make orchestrator stateless + durable queue)
  
Choreography (event-driven):
  Each service publishes events → next service reacts
  Order Created event → Inventory Service reserves → Stock Reserved event 
    → Payment Service charges → Payment Charged event → Order confirmed
  ✓ Fully decoupled, no central coordinator
  ✗ Hard to trace end-to-end flow, implicit dependencies
  ✗ Difficult to add compensating actions retroactively

Production choice: 
  Orchestration for complex multi-step business flows (order placement, onboarding)
  Choreography for simple 2-3 step flows (user signup → send welcome email)
  
Tools: Temporal (recommended), Cadence, AWS Step Functions, Netflix Conductor
```

### Idempotency — Essential for Both Patterns

```
In distributed systems, messages can be delivered MORE THAN ONCE (retry on timeout).
Every service MUST be idempotent:

  "Charging the card" called twice with same idempotency_key → only charges once

  Implementation:
    CREATE TABLE processed_events (
      event_id UUID PRIMARY KEY,
      processed_at TIMESTAMP
    );
    
    BEGIN TRANSACTION;
      INSERT INTO processed_events (event_id) VALUES ('evt-123')
        ON CONFLICT DO NOTHING;
      -- If inserted (new): process the event
      -- If conflict (duplicate): skip processing
    COMMIT;
```

---

## 6. Clocks in Distributed Systems

### Why Clocks Matter

```
Problem: No global clock in distributed systems
  Node A's clock: 10:00:00.000
  Node B's clock: 10:00:00.150  (150ms ahead due to NTP drift)
  
  If both write simultaneously, who wrote "first"?
  Using wall clock → B appears to have written later (even if A was truly later)
  → Last-Write-Wins with wall clocks can silently lose data
```

### Logical Clocks

```
Lamport Clock:
  Each event gets a counter: max(local_counter, received_counter) + 1
  Establishes a total order of events
  ✗ Cannot distinguish concurrent events from causal ones
  (If Lamport(A) < Lamport(B), it does NOT mean A happened before B)

Vector Clock:
  Each node maintains a vector: [node_A: 3, node_B: 1, node_C: 2]
  Increment own entry on each local event
  On receive: take element-wise max + increment own
  
  Comparison:
    [3, 1, 2] vs [2, 2, 1]:
    Neither dominates (3>2 but 1<2) → CONCURRENT events (conflict!)
    [3, 1, 2] vs [3, 0, 1]:
    First dominates (all entries ≥) → first HAPPENED AFTER second (causal)
  
  ✓ Detects concurrent vs causal events (Lamport can't)
  ✗ Vector grows with number of writers (mitigate: cap + truncate oldest)
  Used by: Amazon Dynamo, Riak

Hybrid Logical Clock (HLC):
  Combines wall clock with logical counter
  HLC = max(wall_clock, last_HLC) + 1
  
  ✓ Close to wall clock (human-readable timestamps)
  ✓ Causal ordering guaranteed
  ✓ Fixed size (unlike vector clocks that grow with nodes)
  Used by: CockroachDB, MongoDB, YugabyteDB
```

### Google TrueTime (Spanner's Secret Weapon)

```
GPS receivers + atomic clocks in every Google datacenter
Provide bounded uncertainty: TrueTime.now() → [earliest, latest]
  Uncertainty typically 1-7ms

Spanner's commit protocol:
  1. Assign commit timestamp = TrueTime.now().latest
  2. WAIT until TrueTime.now().earliest > commit_timestamp
     (wait out the uncertainty window — typically 7ms)
  3. Now it's guaranteed: this transaction's timestamp is in the past
     for ALL nodes globally
  
  → Linearizable reads without coordination (just check local TrueTime)
  → The only system with externally consistent distributed transactions

  ✗ Requires GPS/atomic clock hardware (Google-only infrastructure)
  ✗ 7ms commit latency floor (uncertainty window)
```

---

## 7. Consistency in Interviews — What to Say

```
Framework for answering "What consistency does your system need?":

1. Identify the data and its criticality
2. Choose the WEAKEST consistency that meets the requirement
3. Justify why stronger consistency is too expensive or unnecessary

Examples:

Social media feed: Eventual consistency
  "A user's feed being 2 seconds behind is fine. Linearizability would 
   require consensus on every write — adding 100ms+ latency for no 
   user-visible benefit at the scale of 100K writes/sec."

E-commerce inventory: Strong consistency on the hot path
  "We can't oversell. The seat/stock reservation uses SELECT FOR UPDATE
   on a single shard (serialized writes). Read-heavy product pages use 
   eventually consistent cached data."

Chat message ordering: Causal consistency
  "Messages in a conversation must appear in causal order (reply after 
   original). But messages across different conversations can be eventually 
   consistent — no user expects global ordering."

Bank transfer: Saga with idempotent compensation
  "Cross-account transfers use the Saga pattern: debit account A → credit 
   account B. If credit fails, compensate by re-crediting A. Each step is 
   idempotent (idempotency_key = transfer_id). Within a single account, 
   balance reads are linearizable (single-leader, synchronous replication)."

User session: Read-your-writes
  "After login, the user must see their own session. We achieve this by 
   routing session reads to the primary for 10 seconds after write. Other 
   services seeing stale session data for 1-2 seconds is acceptable."

Configuration / Feature flags: Linearizable
  "Feature flag changes must be immediately visible to all nodes — otherwise 
   we could have half the fleet on the old behavior and half on the new, 
   causing inconsistency. We store flags in etcd (Raft consensus)."
```
