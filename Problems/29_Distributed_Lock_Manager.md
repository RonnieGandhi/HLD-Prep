# 29. Design a Distributed Lock Manager

---

## 1. Functional Requirements (FR)

- **Acquire lock**: A process acquires a named lock with a timeout (lease duration)
- **Release lock**: The lock holder explicitly releases the lock
- **Auto-release**: Lock automatically released after TTL expires (prevents deadlocks from crashed holders)
- **Mutual exclusion**: At most one process holds the lock at any time
- **Reentrant locks** (optional): Same process can acquire the same lock multiple times
- **Read-Write locks** (optional): Multiple readers OR one writer
- **Try-lock**: Non-blocking attempt to acquire; return immediately if unavailable
- **Lock metadata**: See who holds the lock, when it was acquired, when it expires

---

## 2. Non-Functional Requirements (NFRs)

- **Correctness**: The lock MUST provide mutual exclusion — this is the primary requirement
- **High Availability**: Lock service must be highly available (SPOF means entire system blocks)
- **Low Latency**: Lock acquisition in < 5 ms p99
- **Fault Tolerant**: Survive node failures without leaving orphaned (permanently held) locks
- **Deadlock Prevention**: Auto-release via TTL prevents permanent deadlocks
- **Fairness**: Optional — FIFO ordering for lock waiters

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Active locks | 1M |
| Lock operations / sec | 100K |
| Avg lock hold time | 5 seconds |
| Lock metadata size | 200 bytes |
| Total memory | 1M × 200B = 200 MB |

Distributed locks are lightweight — the challenge is **correctness**, not scale.

---

## 4. High-Level Design (HLD)

### Approach 1: Redis-Based Lock (Single Instance — Simple)

```
Client A                    Redis                     Client B
   │                          │                          │
   │── SET lock:X A NX EX 30 ─▶│                          │
   │◀── OK ────────────────────│                          │
   │                          │                          │
   │   (Client A holds lock)  │── SET lock:X B NX EX 30 ─▶│
   │                          │◀── nil (lock busy) ───────│
   │                          │                          │
   │── DEL lock:X ────────────▶│                          │
   │                          │                          │
```

**Redis SET NX EX (The Simplest Lock)**:
```
ACQUIRE:  SET lock:{name} {owner_id} NX EX {ttl_seconds}
  NX = only set if Not eXists (atomic)
  EX = expire after ttl_seconds
  
  If returns "OK" → lock acquired
  If returns nil → lock held by someone else

RELEASE:  (must be atomic — use Lua script)
  if redis.call("GET", key) == owner_id then
      return redis.call("DEL", key)
  else
      return 0  -- not the lock holder; don't delete!
  end
```

**Why `owner_id` matters**: Without it, Client A could accidentally release Client B's lock:
1. Client A acquires lock (TTL=30s)
2. Client A's operation takes 35 seconds (lock auto-expires at 30s)
3. Client B acquires the lock at 31s
4. Client A finishes and calls DEL → deletes Client B's lock!
5. **Solution**: Only delete if current value matches your owner_id

**Problem with single Redis**: If Redis crashes, all locks are lost. Redis replication is async → failover can grant the same lock to two clients.

### Approach 2: Redlock Algorithm (Distributed Redis — Safer) ⭐

Martin Kleppmann criticized Redlock, but it's still widely used and good enough for many use cases.

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Redis 1  │  │ Redis 2  │  │ Redis 3  │  │ Redis 4  │  │ Redis 5  │
│(Independ)│  │(Independ)│  │(Independ)│  │(Independ)│  │(Independ)│
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

**Redlock Algorithm**:
1. Get current time T1
2. Try to acquire lock on ALL 5 independent Redis instances (same key, same owner_id, same TTL)
3. Set a small per-instance timeout (e.g., 5-50ms) to avoid blocking on crashed instances
4. Get current time T2. Elapsed = T2 - T1
5. Lock is acquired if:
   - Acquired on **majority** (≥ 3 out of 5) instances
   - Elapsed time < TTL (lock hasn't already expired)
6. Effective lock validity = TTL - elapsed time
7. If lock NOT acquired → release on ALL instances (cleanup)

**Properties**:
- Survives up to 2 out of 5 Redis instances failing
- No single point of failure
- Clock drift tolerance is bounded

### Approach 3: ZooKeeper-Based Lock (Strongest Correctness) ⭐

```
ZooKeeper Ensemble (3 or 5 nodes, Raft/ZAB consensus)

Lock path: /locks/resource-name/

Client A creates: /locks/resource-name/lock-0000000001 (ephemeral sequential)
Client B creates: /locks/resource-name/lock-0000000002 (ephemeral sequential)
Client C creates: /locks/resource-name/lock-0000000003 (ephemeral sequential)

Lowest sequence number holds the lock → Client A
```

**Algorithm**:
1. Create an ephemeral sequential znode under `/locks/{resource}/`
2. Get all children of `/locks/{resource}/`
3. If your znode has the lowest sequence number → you hold the lock
4. If not → set a **watch** on the znode with the next-lower sequence number
5. When that znode is deleted → you're notified → recheck (you might now be lowest)
6. To release: delete your znode
7. If client crashes: ephemeral znode auto-deleted → lock auto-released

**Why ephemeral sequential?**
- **Ephemeral**: Deleted when client's session ends (crash protection)
- **Sequential**: ZooKeeper assigns a monotonically increasing sequence number → natural ordering → fairness (FIFO)

**Why watch on predecessor (not all children)?**
- Avoids **herd effect**: If 100 clients are waiting, and the lock holder releases, only the NEXT client is notified (not all 100)

### Approach 4: etcd-Based Lock (Modern Alternative to ZooKeeper)

```go
// etcd distributed lock using lease
lease, _ := client.Grant(ctx, 30)  // 30-second TTL lease
_, err := client.Put(ctx, "/locks/resource", "owner-id", clientv3.WithLease(lease.ID))
// Keep lease alive with KeepAlive
// On crash, lease expires → lock released
```

- Uses **Raft consensus** — linearizable writes
- Lease-based TTL — auto-release on failure
- **etcd** is simpler than ZooKeeper (gRPC API, no JVM dependency)

---

## 5. APIs

### Acquire Lock
```
acquire(lock_name: string, owner_id: string, ttl_ms: int) → {acquired: bool, lock_token: string, expires_at: timestamp}

// With blocking wait
acquire_blocking(lock_name, owner_id, ttl_ms, wait_timeout_ms) → same

// Non-blocking try
try_acquire(lock_name, owner_id, ttl_ms) → {acquired: bool} (returns immediately)
```

### Release Lock
```
release(lock_name: string, lock_token: string) → {released: bool}
// lock_token prevents accidental release by non-holder
```

### Extend Lock (Renew TTL)
```
extend(lock_name: string, lock_token: string, additional_ttl_ms: int) → {extended: bool, new_expires_at: timestamp}
```

### Get Lock Info
```
info(lock_name: string) → {held: bool, owner: string, acquired_at: timestamp, expires_at: timestamp}
```

---

## 6. Data Model

### Redis Lock Entry

```
Key:    lock:{resource_name}
Value:  {owner_id}:{uuid_token}    // identifies who holds it
TTL:    30 seconds (configurable)
```

### ZooKeeper Lock Structure

```
/locks/
  /resource-A/
    /lock-0000000001    (data: {owner: "service-A-pod-3", created: "..."})
    /lock-0000000002    (data: {owner: "service-B-pod-1", created: "..."})
  /resource-B/
    /lock-0000000001    (data: {owner: "service-C-pod-2", created: "..."})
```

### Fencing Token
```
On lock acquisition, increment a global counter:
  fencing_token = INCR global:fence:{lock_name}

When making the protected operation (e.g., writing to DB):
  Include fencing_token in the request
  Storage server rejects requests with fencing_token < last_seen_token
  
This prevents the "zombie lock holder" problem (see below)
```

---

## 7. Fault Tolerance

### The Martin Kleppmann Problem (Zombie Lock Holders)

**Scenario**:
1. Client A acquires lock (TTL=30s)
2. Client A starts a long GC pause (60 seconds!)
3. Lock auto-expires at 30s
4. Client B acquires the lock
5. Client A wakes up from GC pause, thinks it still holds the lock
6. **Both A and B operate on the shared resource → DATA CORRUPTION**

**Solution: Fencing Tokens**

```
Timeline:
  T=0:   Client A acquires lock, gets fencing_token = 33
  T=30:  Lock expires
  T=31:  Client B acquires lock, gets fencing_token = 34
  T=60:  Client A wakes up, sends write with fencing_token = 33
  
  Storage server: "fencing_token 33 < last_seen 34 → REJECT"
```

- Every lock acquisition increments a monotonic counter (the fencing token)
- The protected resource (DB, API) checks the fencing token
- Rejects operations from stale lock holders

| Concern | Solution |
|---|---|
| **Lock holder crashes** | TTL auto-expires the lock; ZK ephemeral node auto-deletes |
| **Network partition** | Majority quorum (Redlock/ZK/etcd) ensures only one side can acquire |
| **Clock drift (Redlock)** | Use bounded clock drift assumption; set TTL conservatively |
| **GC pause extends past TTL** | Fencing tokens prevent stale holders from causing damage |
| **Split brain (two holders)** | Consensus-based systems (ZK, etcd) prevent this; Redlock has a theoretical gap |
| **Deadlock** | TTL-based auto-release prevents permanent deadlocks |

### Choosing the Right Approach

| Scenario | Recommendation |
|---|---|
| **Efficiency lock** (cache dedup, non-critical) | Single Redis `SET NX EX` — simple, fast, good enough |
| **Correctness lock** (billing, inventory) | ZooKeeper or etcd — consensus-backed, linearizable |
| **Middle ground** (important but not financial) | Redlock (5 independent Redis) |

---

## 8. Additional Considerations

### Lock Renewal / Heartbeat Pattern
- For long-running operations, the lock TTL might expire before the operation completes
- **Solution**: Lock holder runs a background thread that renews the lock every TTL/3
  ```
  Lock acquired with TTL = 30s
  Background thread: Every 10s, extend lock by 30s
  If holder crashes → thread stops → lock expires naturally
  ```
- This is how Redisson (Java Redis client) implements its `RLock`

### Read-Write Locks
- **Read lock**: Multiple readers allowed simultaneously
- **Write lock**: Exclusive — no readers or other writers
- **ZooKeeper implementation**:
  - Read locks create `/locks/resource/read-0000000001`
  - Write locks create `/locks/resource/write-0000000001`
  - Read lock acquired if no write locks with lower sequence exist
  - Write lock acquired if no locks with lower sequence exist

### Distributed Semaphore
- Generalization: Allow N concurrent holders (not just 1)
- ZooKeeper: Allow lock if count of existing children < N
- Redis: `DECR semaphore:{name}; if result >= 0 → acquired`

### Advisory vs. Mandatory Locks
- **Advisory**: Lock is cooperative — processes must voluntarily check the lock. If a buggy process ignores the lock, it bypasses mutual exclusion
- **Mandatory**: Enforced by the system (e.g., DB row locks). Cannot be bypassed
- Distributed locks are almost always **advisory** — correctness depends on all participants respecting the lock protocol

### Comparison Summary

| Feature | Redis SET NX | Redlock | ZooKeeper | etcd |
|---|---|---|---|---|
| Consistency | Weak (async replication) | Stronger (majority) | Strong (ZAB consensus) | Strong (Raft) |
| Availability | High (single node fast) | High (survives minority failure) | High (survives minority) | High |
| Latency | < 1 ms | ~5-10 ms | ~5-20 ms | ~5-10 ms |
| Complexity | Very simple | Moderate | Complex (JVM, session mgmt) | Moderate |
| Fairness | No | No | Yes (sequential znodes) | Yes (lease revision) |
| Auto-release | TTL | TTL | Ephemeral node + session | Lease TTL |
| Fencing token | Manual | Manual | Built-in (zxid) | Built-in (revision) |

---

## 9. Deep Dive: Engineering Trade-offs

### Redlock Algorithm — Step-by-Step Walkthrough

```
Setup: 5 independent Redis instances (NOT replicas — fully independent)
  Redis-1 (us-east-1a), Redis-2 (us-east-1b), Redis-3 (us-west-2a),
  Redis-4 (eu-west-1a), Redis-5 (ap-south-1a)

Lock acquisition for resource "inventory:item-42":

  T=0ms:    Client generates: 
              lock_key = "lock:inventory:item-42"
              owner_id = UUID "abc-123" (unique per attempt)
              ttl = 10 seconds
              
  T=0ms:    Record start_time = now()
  
  T=0ms:    Send SET lock_key abc-123 NX PX 10000 to ALL 5 Redis instances
            (in parallel, not sequential — critical for latency)
  
  T=3ms:    Redis-1 responds: OK (lock acquired)
  T=4ms:    Redis-2 responds: OK (lock acquired)
  T=5ms:    Redis-3 responds: nil (lock held by someone else)
  T=6ms:    Redis-4 responds: OK (lock acquired)
  T=50ms:   Redis-5: timeout (network delay)
  
  T=50ms:   Count successes: 3 out of 5 → majority (≥ N/2+1 = 3) ✓
  
  T=50ms:   Compute validity time:
              elapsed = now() - start_time = 50ms
              validity = ttl - elapsed - clock_drift_bound
              validity = 10000ms - 50ms - 20ms = 9930ms
              (clock_drift_bound ≈ ttl × 0.01 × 2 = 200ms is conservative)
              validity > 0 → LOCK ACQUIRED ✓
  
  Lock is valid for next 9930ms. Client must complete work within this time.

Lock release:
  Send Lua script to ALL 5 Redis instances (even ones that failed):
    if redis.call("GET", key) == owner_id then
        return redis.call("DEL", key)
    end
  This ensures we only release OUR lock (not someone else's)

Why ALL 5 on release? If we only release the 3 that succeeded,
  and Redis-3 (which was held by someone else) later expires,
  we leave stale lock state. Always clean up everywhere.
```

### Redlock Failure Scenarios

```
Scenario 1: Client crashes while holding lock
  Client acquires lock (3/5 Redis instances)
  Client crashes → lock expires after TTL (10 seconds)
  Other clients can acquire after TTL
  ✓ TTL prevents permanent deadlock

Scenario 2: Redis instance crashes while holding lock
  Client holds lock on Redis-1, 2, 4 (3/5)
  Redis-2 crashes → restarts → lock state LOST
  Another client tries: acquires on Redis-2 (fresh), 3, 5 → 3/5 ✓
  TWO clients hold the "lock" simultaneously → VIOLATION!
  
  Solution: Delayed restart
    Redis-2 must wait at least TTL seconds before accepting new locks
    After restart, all locks that existed before crash have expired
    → No conflict possible
    
    Implementation: Redis persistence (AOF with fsync=always) preserves locks
    OR: just wait TTL after restart before serving lock requests

Scenario 3: Clock drift
  Client A acquires lock at T=0, TTL=10s
  Redis-1's clock runs fast → lock expires at T=8 (not T=10)
  Client B acquires at T=9 → overlap with Client A's expected validity
  
  Solution: Include clock drift in validity calculation
    validity = ttl - elapsed - (ttl × clock_drift_rate)
    clock_drift_rate ≈ 0.01 (1% — typical for NTP-synced servers)
    For 10s TTL: validity reduced by 100ms for safety margin
    
  Martin Kleppmann's criticism: clock drift is UNBOUNDED in practice
    GC pauses, VM live migration, NTP jumps → clocks can jump seconds
    → Redlock's safety depends on bounded clock drift → not provable
    → Use fencing tokens for true safety (see §7)

Scenario 4: Network partition during lock hold
  Client holds lock on Redis-1, 2, 4
  Network partition: Client can reach Redis-1, 2 but NOT Redis-3, 4, 5
  Lock expires on Redis-4 (unreachable, so client can't renew)
  Another client acquires on Redis-3, 4, 5 → 3/5
  Both clients think they hold the lock!
  
  Solution: Client must validate lock before critical operation
    Before protected write: check lock still held (re-read from majority)
    OR: use fencing tokens (storage server validates token monotonicity)
```

### ZooKeeper Lock — Why It's Stronger (and Slower)

```
ZooKeeper lock acquisition:

  1. Client creates EPHEMERAL + SEQUENTIAL znode:
     CREATE /locks/resource-A/lock- (ephemeral, sequential)
     → ZK creates: /locks/resource-A/lock-0000000007
     
  2. Client lists children of /locks/resource-A/:
     [lock-0000000003, lock-0000000005, lock-0000000007]
     
  3. Is my node the LOWEST numbered? 
     NO (lock-0000000003 is lower) → I don't have the lock
     
  4. Set a WATCH on the node just before mine (lock-0000000005):
     "Notify me when lock-0000000005 is deleted"
     
  5. Wait for notification...
     
  6. lock-0000000005 deleted → re-check: am I lowest now?
     [lock-0000000003, lock-0000000007]
     NO → watch lock-0000000003
     
  7. lock-0000000003 deleted → I am lowest! → LOCK ACQUIRED

Why this is correct:
  - ZAB consensus: CREATE is linearizable → total order guaranteed
  - Ephemeral: if client crashes → session expires → znode deleted → lock released
  - No TTL needed: session heartbeat (not wall clock) determines liveness
  - No clock drift problem: ZK uses logical ordering, not timestamps
  
Why only watch PREDECESSOR (not all children):
  "Herd effect": if all waiters watch the lock holder,
    when lock is released → ALL waiters wake up → ALL try to acquire
    → thundering herd
  
  Watching only predecessor: 
    lock released → only next waiter notified → O(1) notification
    This is the "scalable lock" pattern from the ZooKeeper recipes

Performance:
  Acquire: 2 ZK operations (create + getChildren) → ~15-20ms
  Release: 1 ZK operation (delete) → ~5-10ms
  Compare: Redis SET NX → ~0.5ms
  
  ZK is 20-40× slower but provides linearizable correctness
```

### Lock Renewal: The Watchdog Pattern

```
Problem: Long-running operation exceeds lock TTL

  Client acquires lock (TTL=30s)
  Operation takes 45 seconds
  At T=30s: lock expires → another client acquires → DATA CORRUPTION

Solution: Background renewal thread (watchdog)

  main_thread:
    lock = acquire("resource-X", ttl=30s)
    watchdog = start_renewal_thread(lock, renewal_interval=10s)
    try:
        do_critical_work()  // may take 45 seconds
    finally:
        watchdog.stop()
        lock.release()
  
  renewal_thread (runs every 10 seconds):
    while not stopped:
        sleep(10 seconds)
        success = extend_lock(lock.key, lock.owner_id, new_ttl=30s)
        if not success:
            // Lock was lost (expired between renewals, or stolen)
            signal_main_thread_to_abort()
            return
  
  Lua script for safe renewal:
    if redis.call("GET", key) == owner_id then
        redis.call("PEXPIRE", key, new_ttl)
        return 1
    else
        return 0  -- lock lost, don't extend someone else's lock
    end

  This is exactly how Redisson (Java) implements its RLock:
    Default: TTL=30s, renewal every 10s (TTL/3)
    If client JVM crashes → renewal stops → lock expires in ≤30s

Edge case: GC pause during renewal
  T=0:   Lock acquired, TTL=30s
  T=10:  Renewal succeeds, TTL reset to 30s
  T=15:  JVM enters full GC pause (stop-the-world)
  T=45:  GC completes (30-second pause!)
         Renewal thread wakes up → tries to renew → lock has EXPIRED
         Another client already holds the lock
         Renewal fails → signal abort to main thread
  T=45:  But main thread also just woke from GC → it already executed
         part of the critical section with an expired lock → DATA CORRUPTION
  
  This is the Martin Kleppmann problem. Solution: FENCING TOKENS.
  The renewal watchdog catches MOST cases but not GC pauses.
  For true correctness: always use fencing tokens with the storage layer.
```

### When to Use Each Lock Approach

```
Decision tree:

  Is correctness critical (financial, inventory)?
    YES → Use ZooKeeper or etcd (consensus-backed, linearizable)
           + ALWAYS use fencing tokens on the storage layer
    NO → Continue below
  
  Is latency critical (< 1ms)?
    YES → Use single Redis SET NX
           Accept: lock may be unsafe during Redis failover (~seconds)
           Ensure: operations are idempotent (so duplicate execution is harmless)
    NO → Continue below
  
  Do you need to survive Redis failover?
    YES → Use Redlock (5 independent Redis instances)
           Accept: theoretical gap during clock drift (see Kleppmann criticism)
           Mitigate: use fencing tokens for critical operations
    NO → Single Redis SET NX is sufficient
  
  Summary:
    "This is just to avoid redundant work" → Single Redis, no fencing
    "This protects a database write" → Redlock + fencing token
    "This involves money" → ZooKeeper/etcd + fencing token, always
```

