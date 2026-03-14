# 4. Design a Unique ID Generator (Twitter Snowflake)

---

## 1. Functional Requirements (FR)

- Generate globally unique IDs across a distributed system
- IDs must be **64-bit integers** (fit in a long)
- IDs must be **roughly sortable by time** (IDs generated later are larger)
- Generate IDs with extremely high throughput (100K+ IDs/sec per node)
- No coordination between nodes required for ID generation
- IDs should not expose sensitive information (e.g., exact count of objects)

---

## 2. Non-Functional Requirements (NFRs)

- **Uniqueness**: No two IDs are ever the same across all nodes and all time
- **High Availability**: ID generation must never block (no single point of failure)
- **Low Latency**: < 1 ms per ID generation (ideally sub-microsecond)
- **Scalability**: Support 1000+ nodes generating IDs concurrently
- **Time-Ordered**: IDs generated at time T1 < IDs generated at time T2 (within same node, guaranteed; across nodes, approximately)
- **Compact**: 64-bit (not UUIDs which are 128-bit)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total ID generation rate | 10M IDs/sec (system-wide) |
| Number of nodes | 1024 (10 bits) |
| IDs per node per ms | ~10K |
| Sequence bits (12) | 4096 per millisecond per node |
| Epoch lifespan (41 bits) | 2^41 ms ≈ 69.7 years |
| Storage per ID | 8 bytes |

---

## 4. High-Level Design (HLD)

### Approaches Compared

| Approach | Pros | Cons |
|---|---|---|
| **UUID (v4)** | No coordination, simple | 128 bits, not sortable, poor index performance |
| **DB Auto-Increment** | Simple, ordered | Single point of failure, not scalable |
| **DB with Ranges** | Scalable | Requires coordination to assign ranges |
| **Twitter Snowflake** ⭐ | 64-bit, sortable, no coordination, fast | Requires clock synchronization |
| **MongoDB ObjectId** | 96-bit, sortable | Larger than 64-bit |

### Snowflake ID Structure (64 bits)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0|                    Timestamp (41 bits)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Datacenter ID   |   Machine ID    |    Sequence Number    |
|     (5 bits)        |   (5 bits)      |    (12 bits)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Bit 0:        Sign bit (always 0 for positive)
Bits 1-41:    Timestamp in milliseconds since custom epoch
Bits 42-46:   Datacenter ID (5 bits = 32 datacenters)
Bits 47-51:   Machine/Worker ID (5 bits = 32 machines per DC)
Bits 52-63:   Sequence number (12 bits = 4096 per ms per machine)
```

### Architecture

```
                         ┌──────────────────────────┐
                         │     ZooKeeper / etcd     │
                         │  (Worker ID Assignment)  │
                         └──────────┬───────────────┘
                                    │ one-time registration
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
     ┌──────▼──────┐        ┌──────▼──────┐        ┌──────▼──────┐
     │ ID Gen Node │        │ ID Gen Node │        │ ID Gen Node │
     │ DC=0, W=0   │        │ DC=0, W=1   │        │ DC=1, W=0   │
     │             │        │             │        │             │
     │ ┌─────────┐ │        │ ┌─────────┐ │        │ ┌─────────┐ │
     │ │Snowflake│ │        │ │Snowflake│ │        │ │Snowflake│ │
     │ │ Engine  │ │        │ │ Engine  │ │        │ │ Engine  │ │
     │ └─────────┘ │        │ └─────────┘ │        │ └─────────┘ │
     └──────┬──────┘        └──────┬──────┘        └──────┬──────┘
            │                      │                       │
     ┌──────▼──────────────────────▼───────────────────────▼──────┐
     │                    Application Services                     │
     │         (call local ID generator — no network hop)          │
     └─────────────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Snowflake Engine (In-Process Library)
- **Why in-process**: Zero network latency. Each app server has the Snowflake library embedded
- **How it works**:
  1. Get current timestamp in milliseconds since custom epoch (e.g., 2020-01-01)
  2. If same millisecond as last ID → increment sequence counter
  3. If sequence overflows (> 4095) → **wait** until next millisecond
  4. If timestamp moved backward (clock skew) → **reject** or wait
  5. Combine: `(timestamp << 22) | (datacenter_id << 17) | (worker_id << 12) | sequence`

#### ZooKeeper / etcd (Worker ID Assignment)
- **Why**: Need to assign unique `(datacenter_id, worker_id)` pairs to each node
- **How**: 
  - On startup, each node creates an ephemeral sequential znode in ZooKeeper
  - The znode's sequence number becomes the worker ID
  - If the node dies, the ephemeral znode is deleted → worker ID can be reused
- **Alternative**: Store worker ID mappings in a simple DB or config file (less dynamic but simpler)

#### NTP Synchronization
- **Why**: Snowflake depends on clock timestamps. Clocks must be reasonably synchronized
- **How**: All servers run NTP daemon synced to the same NTP server pool
- **Tolerance**: Clocks within 10ms of each other is acceptable (IDs are time-sorted within a node, approximately sorted across nodes)

---

## 5. APIs

```java
// Core API — typically an in-process library call
long nextId()

// Decompose an ID back to its components (for debugging)
IdComponents parse(long id)
// Returns: {timestamp, datacenterId, workerId, sequence}

// Batch generation for efficiency
long[] nextIds(int count)
```

### Snowflake Implementation (Java-like pseudocode)

```java
class SnowflakeIdGenerator {
    private final long epoch = 1577836800000L; // 2020-01-01 UTC
    private final long datacenterIdBits = 5L;
    private final long workerIdBits = 5L;
    private final long sequenceBits = 12L;
    
    private final long maxDatacenterId = ~(-1L << datacenterIdBits); // 31
    private final long maxWorkerId = ~(-1L << workerIdBits);         // 31
    private final long maxSequence = ~(-1L << sequenceBits);         // 4095
    
    private final long workerIdShift = sequenceBits;                 // 12
    private final long datacenterIdShift = sequenceBits + workerIdBits; // 17
    private final long timestampShift = sequenceBits + workerIdBits + datacenterIdBits; // 22
    
    private final long datacenterId;
    private final long workerId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;
    
    public synchronized long nextId() {
        long timestamp = currentTimeMillis();
        
        // Clock moved backwards — refuse to generate
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                "Clock moved backwards by " + (lastTimestamp - timestamp) + "ms");
        }
        
        if (timestamp == lastTimestamp) {
            // Same millisecond — increment sequence
            sequence = (sequence + 1) & maxSequence;
            if (sequence == 0) {
                // Sequence exhausted — wait for next millisecond
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            // New millisecond — reset sequence
            sequence = 0L;
        }
        
        lastTimestamp = timestamp;
        
        return ((timestamp - epoch) << timestampShift)
             | (datacenterId << datacenterIdShift)
             | (workerId << workerIdShift)
             | sequence;
    }
    
    private long waitNextMillis(long lastTs) {
        long ts = currentTimeMillis();
        while (ts <= lastTs) {
            ts = currentTimeMillis();
        }
        return ts;
    }
}
```

---

## 6. Data Model

### ID Bit Layout

| Field | Bits | Range | Purpose |
|---|---|---|---|
| Sign | 1 | 0 | Always positive |
| Timestamp | 41 | 0 – 2,199,023,255,551 ms (~69 years) | Time ordering |
| Datacenter ID | 5 | 0 – 31 | 32 data centers |
| Worker ID | 5 | 0 – 31 | 32 workers per DC |
| Sequence | 12 | 0 – 4095 | 4096 IDs per ms per worker |

### Custom Epoch Choice
```
Standard Epoch:  1970-01-01 (Unix)
Custom Epoch:    2020-01-01 00:00:00 UTC = 1577836800000 ms

Why custom? Extends the 41-bit lifespan. Starting from 2020 gives us until ~2089.
Starting from 1970 would only last until ~2039.
```

### Worker ID Assignment Table (ZooKeeper / DB)

```sql
CREATE TABLE worker_assignments (
    datacenter_id  SMALLINT NOT NULL,
    worker_id      SMALLINT NOT NULL,
    hostname       VARCHAR(256),
    assigned_at    TIMESTAMP,
    heartbeat_at   TIMESTAMP,
    PRIMARY KEY (datacenter_id, worker_id)
);
```

---

## 7. Fault Tolerance

### General
| Issue | Solution |
|---|---|
| **Single node failure** | No impact on other nodes — each generates IDs independently |
| **No SPOF** | ID generation is decentralized; no central service needed at runtime |
| **Worker ID reuse** | ZooKeeper ephemeral nodes auto-cleanup; new instance gets a fresh ID |

### Problem-Specific Fault Tolerance

1. **Clock Backward Jump (NTP Adjustment)**
   - NTP can adjust clock backward (e.g., -5ms correction)
   - **If small (< 5ms)**: Wait until clock catches up to `lastTimestamp`
   - **If large (> 5ms)**: Throw exception, alert ops team, refuse to generate IDs until clock is fixed
   - **Prevention**: Configure NTP for slew mode (gradually adjust, never jump)

2. **Sequence Exhaustion (> 4096 IDs in 1ms)**
   - **Solution**: Spin-wait until the next millisecond (waitNextMillis)
   - In practice, 4096/ms = 4M IDs/sec per node — rarely a bottleneck
   - If needed: Use 13 sequence bits (8192/ms) by reducing worker bits

3. **Worker ID Exhaustion**
   - 10 bits = 1024 unique workers (5 DC bits × 5 worker bits)
   - **If not enough**: Reduce timestamp bits by 1 (still ~34 years) → use 11 bits for worker (2048)
   - Or: Dynamic worker ID assignment with lease renewal

4. **ZooKeeper Failure**
   - Worker IDs are assigned once on startup and cached locally
   - ZooKeeper being down doesn't affect running generators
   - Only affects new node registration

5. **Duplicate IDs After Crash/Restart**
   - Node crashes, restarts within the same millisecond with sequence=0
   - **Solution**: On startup, wait 1ms or persist lastTimestamp to disk and read on startup

---

## 8. Additional Considerations

### Alternative Bit Allocations

| Variant | Timestamp | DC | Worker | Sequence | Trade-off |
|---|---|---|---|---|---|
| **Standard Snowflake** | 41 | 5 | 5 | 12 | Balanced |
| **High Throughput** | 39 | 4 | 5 | 15 | 32K IDs/ms, 17 year lifespan |
| **Many Workers** | 41 | 0 | 12 | 10 | 4096 workers, 1K IDs/ms |
| **Long Lifespan** | 43 | 5 | 5 | 10 | 278 years, 1K IDs/ms |

### Comparison with Other Approaches

#### UUID v4
- 128 bits, truly random, no ordering
- B-tree index performance is poor (random inserts)
- Good when ordering doesn't matter and 128 bits is acceptable

#### ULID (Universally Unique Lexicographically Sortable Identifier)
- 128 bits: 48-bit timestamp + 80-bit randomness
- Lexicographically sortable (string comparison works)
- No worker coordination needed
- Collision probability: negligible for < 2^80 IDs per millisecond

#### Database Ticket Server (Flickr approach)
- Two MySQL servers, one generates odd IDs, one generates even
- Simple but limited scalability, requires network round-trip
- Use `auto_increment_increment=2` and `auto_increment_offset=1/2`

### ID as Database Primary Key
- Snowflake IDs are monotonically increasing within a node → excellent B-tree insert performance
- Across nodes, IDs are roughly time-ordered → much better than random UUIDs
- 64-bit fits in a `BIGINT` column → half the storage of UUID

### Embedding Metadata in IDs
Some systems embed business information:
```
| Timestamp | Service Type (4 bits) | Shard ID (8 bits) | Sequence |
```
This allows routing a request to the correct shard just by looking at the ID — no separate lookup needed.

### Monitoring
- Track IDs generated per second per node
- Alert if clock skew detected (lastTimestamp > currentTimestamp)
- Monitor sequence utilization (if consistently hitting 4095, need more capacity)
- Dashboard showing worker ID assignments across datacenters

---

## 9. Deep Dive: Engineering Trade-offs

### Why Snowflake Over UUID? The Critical Trade-off

```
UUID v4 (128-bit random):
  ID: 550e8400-e29b-41d4-a716-446655440000
  
  ✓ Zero coordination (generate anywhere, anytime)
  ✓ Collision probability: ~10^-18 even at 1B IDs (effectively zero)
  ✗ 128 bits → 16 bytes per ID (vs 8 bytes for Snowflake)
  ✗ NOT sortable by time → terrible as DB primary key:
    - Random UUIDs cause B-tree page splits on INSERT
    - Index fragmentation → 5-10× worse write performance vs sequential IDs
    - No natural ordering for "get recent records"
  ✗ 36-character string representation (ugly in URLs, logs)

Snowflake (64-bit, time-ordered):
  ID: 1541815603606036480
  
  ✓ 64 bits → 8 bytes → fits in BIGINT column (half UUID's storage)
  ✓ Time-sortable → EXCELLENT B-tree performance (sequential inserts)
  ✓ Embeds timestamp → "when was this created?" from ID alone
  ✓ Compact: 19-digit decimal, 11-char Base62
  ✗ Requires worker ID coordination (ZooKeeper or similar)
  ✗ Clock-dependent (must handle clock skew)
  ✗ 4096 IDs/ms/worker limit (enough for most, but a ceiling)

Benchmark (PostgreSQL, 1M INSERTs):
  UUID v4 as PK:      ~45 seconds (random inserts, page splits)
  Snowflake as PK:    ~12 seconds (sequential inserts, no splits)
  Auto-increment:     ~10 seconds (perfectly sequential, but centralized)

Use Snowflake when: Performance matters, time-ordering is useful, 
                     and you can coordinate worker IDs
Use UUID when: Simplicity > performance, or in serverless/stateless 
               environments where worker ID coordination is impractical
```

### Clock Skew: The Hardest Problem in Snowflake

```
Problem:
  Snowflake depends on wall clock for the timestamp component.
  If the clock jumps BACKWARD (NTP correction, VM migration, leap second):
  
  T=1000: Generate ID with timestamp 1000
  T=1001: Clock jumps back to 999
  T=999:  Generate ID with timestamp 999 → DUPLICATE possible!
           Also: IDs are now OUT OF ORDER (999 < 1000)

Solutions:

1. Wait/Block ⭐ (Twitter's original approach):
   if currentTimestamp < lastTimestamp:
       sleep(lastTimestamp - currentTimestamp)  // wait for clock to catch up
   Pro: Simple, guaranteed correctness
   Con: Blocks ID generation during skew (could be seconds)

2. Throw Exception:
   if currentTimestamp < lastTimestamp:
       throw ClockMovedBackwardException()
   Pro: Caller decides how to handle
   Con: Caller must implement retry logic

3. Logical Clock Fallback:
   if currentTimestamp < lastTimestamp:
       use lastTimestamp, increment sequence
   Pro: No blocking, no exceptions
   Con: Timestamp in ID is slightly wrong (acceptable for ordering purposes)

4. Hybrid Logical Clock (HLC):
   Combine wall clock with a logical counter
   When clock moves backward, increment the logical counter instead
   HLC provides causal ordering even with clock skew
   Used by CockroachDB, TiDB
   
Prevention:
  - Use hardware clock sources (GPS, PTP) instead of NTP
  - Monitor clock drift with alerting (alert if drift > 100ms)
  - AWS: use ClockBound service for bounded clock uncertainty
```

### Worker ID Assignment: Coordination Strategies

```
Option 1: ZooKeeper / etcd (Persistent Coordination)
  Each worker acquires a unique sequential ID from ZK
  ✓ Guaranteed unique across restarts
  ✗ Dependency on ZK (SPOF if ZK cluster fails)
  ✗ Operational complexity of maintaining ZK
  Used by: Twitter Snowflake (original)

Option 2: Database Sequence
  Each worker claims an ID from a DB table at startup
  ✓ Simple, uses existing infrastructure
  ✗ DB is a bottleneck at startup (many workers starting at once)
  
Option 3: MAC Address / IP Hash
  worker_id = hash(mac_address) % max_workers
  ✓ No external coordination
  ✗ MAC collisions possible (VMs, containers)
  ✗ Worker ID changes if machine moves

Option 4: Kubernetes Pod Ordinal
  In StatefulSet: pod-0, pod-1, pod-2 → worker_id = ordinal
  ✓ Natural for K8s environments
  ✓ No external service needed
  ✗ Only works with StatefulSets
  Used by: Many modern microservice deployments

Option 5: Pre-assigned Range (Instagram approach)
  Each shard has a fixed worker_id embedded in its configuration
  ✓ No runtime coordination
  ✗ Static, doesn't scale dynamically
```
