# 3. Design a Key-Value Store

---

## 1. Functional Requirements (FR)

- `put(key, value)` — Store a key-value pair; overwrite if key exists
- `get(key)` → value — Retrieve the value for a given key; return null if not found
- `delete(key)` — Remove a key-value pair
- Support keys up to 256 bytes, values up to 10 KB
- Support versioning / conflict resolution (optional: last-write-wins or vector clocks)
- Support TTL-based expiration on keys
- Support range queries on keys (if ordered store)

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: AP system (prefer availability over consistency) — always writable
- **Tunable Consistency**: Allow clients to choose (eventual, quorum, strong)
- **Low Latency**: < 10 ms p99 for reads and writes
- **Scalability**: Horizontal scaling — add nodes to increase capacity linearly
- **Durability**: Data must survive node failures (replicated)
- **Partition Tolerant**: System continues operating during network partitions
- **Automatic failure detection and recovery**

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total key-value pairs | 1 billion |
| Average key size | 100 bytes |
| Average value size | 5 KB |
| Storage per entry | ~5.1 KB |
| Total data | 1B × 5.1 KB ≈ **5 TB** |
| Replication factor 3 | **15 TB** total |
| Writes/sec | 50K |
| Reads/sec | 200K |
| Nodes (if 1TB per node) | 15 nodes |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                            CLIENT                                      │
│                                                                        │
│  put(key, value, consistency=QUORUM)                                  │
│  get(key, consistency=QUORUM)                                         │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│                    CLIENT LIBRARY / COORDINATOR                        │
│                                                                        │
│  1. Hash key → position on consistent hash ring                       │
│     slot = hash(key) % 2^128                                          │
│  2. Walk clockwise → find first node (= coordinator)                  │
│  3. Coordinator handles request:                                       │
│                                                                        │
│  WRITE PATH:                          READ PATH:                       │
│  ┌─────────────────────────────┐      ┌─────────────────────────────┐  │
│  │ 1. Append to local WAL     │      │ 1. Send read to N replicas  │  │
│  │    (fsync for durability)   │      │ 2. Wait for R responses     │  │
│  │ 2. Write to MemTable       │      │ 3. Return highest timestamp │  │
│  │    (in-memory skip list)   │      │ 4. If versions differ →     │  │
│  │ 3. Forward to N-1 replicas │      │    READ REPAIR: send latest │  │
│  │ 4. Wait for W acks         │      │    to stale replicas        │  │
│  │ 5. Return success to client│      │                             │  │
│  │                            │      │ Read from each node:        │  │
│  │ If a replica is DOWN:      │      │  a. Check MemTable (newest) │  │
│  │  → Hinted Handoff: write   │      │  b. Check Bloom Filters     │  │
│  │    to substitute node with │      │     per SSTable             │  │
│  │    hint = {target, data}   │      │  c. If BF says "maybe" →   │  │
│  │  → When target recovers,  │      │     read SSTable index →    │  │
│  │    forward hinted data     │      │     read data block         │  │
│  └─────────────────────────────┘      │  d. Return first match     │  │
│                                       └─────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENT HASH RING                                 │
│                                                                        │
│           vnode_A3        vnode_B1                                     │
│              ╲              ╱                                          │
│    vnode_C2 ── ○────○────○ ── vnode_A1                                │
│              ╱   RING      ╲                                          │
│    vnode_B3 ── ○────○────○ ── vnode_C1                                │
│              ╲              ╱                                          │
│           vnode_A2        vnode_B2                                     │
│                                                                        │
│  Each physical node → 150-200 virtual nodes on ring                   │
│  Key hashes to a position → walk clockwise to first vnode             │
│  Next N-1 DISTINCT physical nodes = replicas                          │
│  Adding a node: only ~1/N of keys migrate (minimal disruption)        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    STORAGE NODE (each of N nodes)                       │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  MemTable (In-Memory)                                            │  │
│  │  • Sorted skip list (O(log n) insert/lookup)                    │  │
│  │  • Threshold: 64 MB → flush to disk as immutable SSTable         │  │
│  │  • Concurrent reads during flush (swap to new MemTable)          │  │
│  └──────────────────────┬───────────────────────────────────────────┘  │
│                         │ flush (when 64 MB)                           │
│  ┌──────────────────────▼───────────────────────────────────────────┐  │
│  │  WAL (Write-Ahead Log)                                           │  │
│  │  • Append-only file, fsynced on every write (durability)        │  │
│  │  • On crash: replay WAL to reconstruct MemTable                 │  │
│  │  • Format: | CRC | timestamp | key_len | val_len | key | value | │  │
│  │  • Truncated after MemTable flush completes                     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                         │                                              │
│  ┌──────────────────────▼───────────────────────────────────────────┐  │
│  │  SSTables (Sorted String Tables — Immutable on Disk)             │  │
│  │                                                                  │  │
│  │  Level 0: [SST-1] [SST-2] [SST-3]  ← freshly flushed, may      │  │
│  │                                       overlap in key ranges      │  │
│  │  Level 1: [SST-A─────────] [SST-B─────────]  ← compacted,      │  │
│  │                                                 non-overlapping  │  │
│  │  Level 2: [SST-X───────────────] [SST-Y───────────────]         │  │
│  │                                                                  │  │
��  │  Each SSTable contains:                                          │  │
│  │   • Data blocks (sorted key-value pairs, compressed)            │  │
│  │   • Sparse index (key → block offset)                           │  │
│  │   • Bloom filter (false positive ~1%, eliminates 99% disk reads)│  │
│  │   • Footer (metadata, compression codec, checksum)              │  │
│  │                                                                  │  │
│  │  Compaction (background):                                        │  │
│  │   • Size-tiered: merge similarly-sized SSTables (write-optimized)│  │
│  │   • Leveled: non-overlapping levels, 10× size growth per level  │  │
│  │     (read-optimized)                                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Merkle Tree (Anti-Entropy)                                      │  │
│  │                                                                  │  │
│  │  • One per key range owned by this node                         │  │
│  │  • Leaf = hash(key + value)                                      │  │
│  │  • Parent = hash(left_child + right_child)                      │  │
│  │  • Periodically compare root hash with replica's root hash      │  │
│  │  • If roots differ → walk tree to find divergent leaves         │  │
│  │  • Sync only divergent keys (bandwidth-efficient repair)         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                    CLUSTER MANAGEMENT                                   │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Gossip Protocol (Decentralized Failure Detection)               │  │
│  │                                                                  │  │
│  │  Every 1 second, each node:                                     │  │
│  │   1. Pick a random peer                                          │  │
│  │   2. Exchange membership state:                                  │  │
│  │      {node_id, status, heartbeat_counter, token_ranges, load}   │  │
│  │   3. Merge: take max(heartbeat_counter) per node                │  │
│  │                                                                  │  │
│  │  Failure detection: Phi Accrual Failure Detector                │  │
│  │   • Track heartbeat inter-arrival times (mean + stddev)          │  │
│  │   • phi = -log10(P(no heartbeat for this long))                 │  │
│  │   • phi > 8 → declare node SUSPECT → after grace period → DEAD  │  │
│  │   • Advantage over fixed timeout: adapts to network conditions   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Hinted Handoff (Temporary Failure Handling)                     │  │
│  │                                                                  │  │
│  │  Node B is down. Write for key K (replicas = A, B, C):          │  │
│  │   1. Coordinator writes to A and C (both alive)                 │  │
│  │   2. For B's copy: write to substitute node D with hint:        │  │
│  │      hint = {target: B, key: K, value: V, timestamp: T}        │  │
│  │   3. When B recovers (detected via gossip):                     │  │
│  │      D forwards all hinted data to B                            │  │
│  │   4. D deletes the hints after B acknowledges                   │  │
│  │                                                                  │  │
│  │  Hint TTL: 3 hours (if B doesn't recover, hints are dropped    │  │
│  │  and full repair via Merkle trees is needed)                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Conflict Resolution                                             │  │
│  │                                                                  │  │
│  │  Option 1 — Vector Clocks (Dynamo):                             │  │
│  │   {node_A: 3, node_B: 1} vs {node_A: 2, node_C: 1}            │  │
│  │   Neither dominates → CONFLICT → return both to client          │  │
│  │   Client merges (application-aware resolution)                  │  │
│  │                                                                  │  │
│  │  Option 2 — Last-Write-Wins (Cassandra):                       │  │
│  │   Compare timestamps → highest wins → simpler but may lose data │  │
│  │   Use Hybrid Logical Clocks to avoid wall-clock skew issues     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### Core Architecture Components

#### Consistent Hashing with Virtual Nodes
- **Why**: Distributes data evenly across nodes; adding/removing nodes only affects neighboring keys
- **How**:
  - Hash ring with 2^128 positions (MD5/SHA-256 of key)
  - Each physical node owns multiple virtual nodes (150-200 vnodes)
  - Key is placed on the ring → walk clockwise to find the first node → that's the coordinator
  - Next N-1 distinct physical nodes clockwise are the replicas
- **Why Virtual Nodes**: Without vnodes, data distribution is uneven. Vnodes ensure uniform distribution and smooth rebalancing when nodes join/leave

#### Storage Engine — LSM Tree (Log-Structured Merge Tree)
**Why LSM over B-Tree**: Optimized for write-heavy workloads (sequential disk writes)

**Write Path**:
1. Write to **WAL** (Write-Ahead Log) — append-only file, fsynced on every write for durability
   - Format per entry: `| CRC (4B) | Timestamp (8B) | Key_Len (4B) | Val_Len (4B) | Key | Value |`
   - On crash: replay WAL entries to reconstruct the MemTable (no data loss)
   - Truncated after MemTable successfully flushes to SSTable
2. Write to **MemTable** — in-memory sorted skip list (O(log n) insert/lookup)
3. When MemTable reaches threshold (~64 MB) → freeze current MemTable (still readable), create a new empty MemTable for new writes → flush frozen MemTable to disk as an immutable **SSTable**
4. Background **compaction** merges SSTables to reduce read amplification

**Read Path**:
1. Check **MemTable** (most recent data, in-memory, fastest)
2. Check **Bloom Filter** for each SSTable (probabilistic: "definitely not here" or "maybe here" — eliminates ~99% of unnecessary disk reads)
3. If Bloom filter says "maybe" → check SSTable's sparse index → locate data block → read and decompress
4. Check SSTables from newest to oldest (Level 0 first, then Level 1, etc.)
5. Return first match found

**SSTable Internal Structure**:
- **Data blocks**: Sorted key-value pairs, compressed (LZ4/Snappy)
- **Sparse index**: Every Nth key → block offset (binary search to find the right block)
- **Bloom filter**: Per-SSTable, ~1% false positive rate with 10 bits per entry
- **Footer**: Metadata, compression codec, format version, checksum

**SSTable Levels**:
- **Level 0**: Freshly flushed from MemTable — key ranges MAY overlap (multiple SSTables can contain the same key)
- **Level 1+**: Compacted — key ranges are non-overlapping within a level (each key exists in at most one SSTable per level)
- Each level is ~10× the size of the previous level

**Compaction Strategies**:
- **Size-Tiered Compaction (STCS)**: Merge similarly-sized SSTables when count exceeds threshold. Good for write-heavy workloads. Trade-off: high space amplification (up to 2× during compaction)
- **Leveled Compaction (LCS)**: Promote SSTables from Level N to Level N+1, re-splitting to maintain non-overlapping ranges. Good for read-heavy workloads. Trade-off: high write amplification (rewrites entire level on merge)

#### Replication
- **Replication Factor (N)**: Configurable, default 3
- **Replication Strategy**: 
  - Coordinator writes to N replicas
  - Quorum-based: W + R > N ensures consistency
    - W=2, R=2, N=3 → strong consistency
    - W=1, R=1 → eventual consistency (fast)
    - W=3, R=1 → write-heavy consistency

#### Gossip Protocol (Failure Detection + Membership)
- **Why**: Decentralized — no single point of failure (unlike Raft/Paxos leader)
- **How**: Every node periodically (every 1 second) picks a random node and exchanges membership state
- **Failure detection**: Uses **Phi Accrual Failure Detector** — calculates a suspicion level (phi) based on heartbeat intervals. Declares node dead when phi exceeds threshold (e.g., 8)
- **Information propagated**: Node liveness, token ranges, load metrics
- **Convergence**: With N nodes, information reaches all nodes in O(log N) gossip rounds (~10 rounds for 1000 nodes)

#### Hinted Handoff (Temporary Failure Handling)
- **When**: A write's designated replica is DOWN but the write must still succeed (AP system)
- **How**:
  1. Coordinator detects that replica B is unreachable
  2. Instead of failing the write, coordinator writes B's copy to substitute node D
  3. D stores the data with a hint: `{target: B, key: K, value: V, timestamp: T}`
  4. When B recovers (detected via gossip heartbeat resuming), D forwards all hinted data to B
  5. D deletes the hints after B acknowledges receipt
- **Hint TTL**: 3 hours — if B doesn't recover within this window, hints are dropped. Full repair via Merkle trees is then needed to restore consistency.
- **Sloppy quorum interaction**: During partitions, writes can go to ANY W available nodes (not necessarily the "correct" replicas). This is what makes the system AP — always writable.

#### Anti-Entropy Repair (Merkle Trees)
- **Purpose**: Detect and repair data inconsistencies between replicas that hinted handoff couldn't fix (e.g., node was down longer than hint TTL)
- **How**:
  1. Each node maintains a Merkle tree per key range it owns
  2. Leaf node = hash(key + value) for each key-value pair
  3. Parent node = hash(left_child + right_child)
  4. Periodically (e.g., every hour), two replica nodes compare their Merkle tree root hashes
  5. If roots match → replicas are identical, done
  6. If roots differ → recursively walk down the tree comparing children until divergent leaf nodes are found
  7. Only the divergent keys are synchronized between replicas
- **Efficiency**: For 1 million keys, a Merkle tree has ~20 levels. If only 10 keys differ, you compare ~200 hashes (not 1 million) — O(log N × diff_count) bandwidth.
- **Rebuild**: Merkle tree is rebuilt after compaction (SSTables changed)

#### Conflict Resolution
- **When**: Two clients write to the same key concurrently (or during a network partition)
- **Option 1 — Vector Clocks (Amazon Dynamo)**:
  - Each version carries a vector clock: `{node_A: 3, node_B: 1}`
  - On read, compare vector clocks: if one dominates (all entries ≥), take the dominant one
  - If neither dominates (concurrent) → **conflict** → return BOTH versions to the client → client merges (application-aware resolution, e.g., union of shopping cart items)
  - **Problem**: Vector clock size grows with number of writers — mitigate by truncating oldest entries when clock exceeds N entries
- **Option 2 — Last-Write-Wins (Cassandra)**:
  - Compare timestamps → highest wins → other version silently discarded
  - Simpler but can **lose data** — concurrent writes, one is dropped
  - Use **Hybrid Logical Clocks (HLC)** instead of wall clock to avoid clock skew issues (HLC = max(wall_clock, last_HLC) + 1)

#### Coordinator Node
- **Role**: The node that receives the client request (determined by consistent hashing of the key)
- **Write coordination**:
  1. Append to local WAL (fsync for durability — survives crashes)
  2. Write to local MemTable (in-memory, fast)
  3. Forward write to N-1 replica nodes in parallel
  4. Wait for W total acknowledgments (including own)
  5. Return success to client
  - **If a replica is DOWN**: Coordinator uses **Hinted Handoff** — writes the data to a substitute node with a hint `{target: dead_node, key, value, timestamp}`. When the target recovers, the substitute forwards the hinted data.
- **Read coordination**:
  1. Send read request to all N replica nodes in parallel
  2. Wait for R responses
  3. Return the value with the highest timestamp (or latest vector clock) to the client
  4. **Read Repair**: If the R responses contain different versions, the coordinator sends the latest version to the stale replicas in the background — eventually converges all replicas
  - **Per-node read flow** (what each replica does on receiving a read):
    - a. Check MemTable (newest data, in-memory)
    - b. Check Bloom Filter for each SSTable — if BF says "definitely not here," skip that SSTable entirely (saves disk I/O)
    - c. If BF says "maybe" → read SSTable's sparse index → find the data block offset → read and decompress the data block
    - d. Return the first match found (newest SSTable first)

---

## 5. APIs

```
put(key: bytes, value: bytes, context: Context) → void
    context contains: version info (vector clock), consistency level

get(key: bytes, consistency: Level) → {value: bytes, context: Context}
    Returns value + metadata for conflict resolution

delete(key: bytes, context: Context) → void
    Actually writes a tombstone (deleted marker)
```

### Internal APIs
```
replicate(key, value, timestamp, vector_clock) → ack
    Node-to-node replication

handoff(key_range, data) → ack
    Transfer data during node join/leave

repair(key_range) → void
    Anti-entropy repair: Merkle tree comparison
```

---

## 6. Data Model

### On-Disk SSTable Format

```
┌───────────────────────────────────────────┐
│ Data Block 1                              │
│  key1 | timestamp | value1                │
│  key2 | timestamp | value2                │
│  ...                                      │
├───────────────────────────────────────────┤
│ Data Block 2                              │
│  ...                                      │
├───────────────────────────────────────────┤
│ Index Block (sparse index)                │
│  key1 → offset_block1                     │
│  key100 → offset_block2                   │
├───────────────────────────────────────────┤
│ Bloom Filter                              │
├───────────────────────────────────────────┤
│ Footer (metadata, version, compression)   │
└───────────────────────────────────────────┘
```

### In-Memory MemTable
```
Data Structure: Skip List (O(log n) insert, lookup, range scan)
Max Size: 64 MB
On flush: Becomes immutable, new MemTable created
```

### WAL Entry Format
```
| CRC (4B) | Timestamp (8B) | Key Length (4B) | Value Length (4B) | Key | Value |
```

### Vector Clock (for conflict resolution)
```json
{
  "key": "user:123:profile",
  "value": "{name: 'Alice'}",
  "vector_clock": {
    "node_A": 3,
    "node_B": 1,
    "node_C": 2
  },
  "timestamp": 1710320000
}
```

**Conflict Resolution**:
- If one vector clock dominates the other → no conflict, take the dominant one
- If concurrent (neither dominates) → return both to client, let application resolve (Amazon Dynamo approach)
- Alternative: Last-Write-Wins (LWW) using timestamps — simpler but can lose data

---

## 7. Fault Tolerance

### Core Mechanisms

| Mechanism | How It Works |
|---|---|
| **Replication** | Each key stored on N=3 nodes. Survives up to N-1 simultaneous failures |
| **WAL** | Every write is first logged to WAL before MemTable. On crash, replay WAL to recover MemTable |
| **Hinted Handoff** | If a replica is down during write, coordinator stores a "hint" locally. When the replica recovers, the hint is forwarded |
| **Anti-Entropy (Merkle Trees)** | Background process compares Merkle trees between replicas to detect and repair inconsistencies |
| **Read Repair** | During a read, if replicas return different versions, coordinator sends the latest version to stale replicas |

### Problem-Specific Fault Tolerance

1. **Node Failure (Temporary)**
   - Hinted Handoff ensures writes are not lost
   - Gossip protocol detects failure and informs all nodes
   - Reads still succeed from remaining replicas (if R ≤ remaining replicas)

2. **Node Failure (Permanent)**
   - Gossip declares node permanently dead after timeout
   - Token reassignment: Virtual nodes of dead node are distributed to remaining nodes
   - Data re-replicated from surviving replicas to maintain RF=3

3. **Network Partition**
   - AP system: Both sides of the partition continue accepting reads and writes
   - Conflict resolution via vector clocks when partition heals
   - Sloppy quorum: Write to any W available nodes (not necessarily the "correct" replicas)

4. **Data Corruption**
   - CRC checksums on every SSTable data block and WAL entry
   - Detected during reads → repair from replica

5. **Compaction Storms**
   - Too many SSTables → background compaction consumes all I/O
   - **Mitigation**: Rate-limit compaction I/O, prioritize user reads/writes, use leveled compaction

---

## 8. Additional Considerations

### Merkle Trees for Anti-Entropy
- Each node maintains a Merkle tree per key range
- Leaf = hash of a key-value pair
- Parent = hash of children
- Two nodes compare root hashes → if different, recursively compare children to find divergent keys
- Only divergent keys are synchronized → efficient bandwidth usage

### Bloom Filters
- One Bloom filter per SSTable
- Before reading an SSTable from disk, check its Bloom filter
- False positive rate ~1% with 10 bits per entry
- Eliminates 99% of unnecessary disk reads

### Tombstones and Garbage Collection
- Deletes write a tombstone (marker) with a timestamp
- Tombstones must persist for a grace period (e.g., 10 days) so all replicas learn about the delete
- After grace period, compaction removes the tombstone

### Hot Keys
- If one key is extremely popular → one node becomes a bottleneck
- **Solution**: Read from all replicas (load balance reads), add a cache layer in front

### Tunable Consistency Examples
| Config | Behavior | Use Case |
|---|---|---|
| W=1, R=1 | Fastest, eventual consistency | Session data, click tracking |
| W=2, R=2, N=3 | Strong consistency | User profiles, inventory |
| W=3, R=1 | Strong write, fast read | Write-once, read-many |
| W=1, R=3 | Fast write, strong read | Logging with reliable reads |

---

## 9. Deep Dive: Engineering Trade-offs

### Why LSM-Tree Over B-Tree?

| Factor | LSM-Tree (This Design) | B-Tree (MySQL/PostgreSQL) |
|---|---|---|
| **Write performance** | O(1) amortized (sequential append) | O(log N) per write (random I/O) |
| **Read performance** | O(N) worst case (check multiple SSTables) | O(log N) guaranteed |
| **Write amplification** | High (data rewritten during compaction) | Low (in-place updates) |
| **Read amplification** | High (may check multiple levels) | Low (single tree traversal) |
| **Space amplification** | Moderate (old versions during compaction) | Low (in-place updates) |
| **Disk I/O pattern** | Sequential (HDD-friendly, SSD-excellent) | Random (SSD preferred) |

**Why LSM wins for a KV store**: KV stores are typically write-heavy (ingesting events, session data, caches). The sequential write pattern of LSM trees achieves 10-100× higher write throughput compared to B-trees. The read penalty is mitigated by Bloom filters (eliminating 99% of false lookups) and the MemTable (most-recent data served from memory).

**When B-Tree is better**: Read-heavy OLTP workloads with point queries and range scans where predictable read latency matters more than write throughput (e.g., relational databases).

### CAP Theorem: Why AP Over CP?

```
This design chooses AP (Availability + Partition Tolerance):
  → During a network partition, BOTH sides continue serving reads AND writes
  → Conflicts are resolved after partition heals (vector clocks / LWW)

Alternative CP choice (e.g., etcd, ZooKeeper):
  → During partition, the minority side REJECTS writes
  → Guarantees no conflicting writes, but reduces availability

Why AP for a general-purpose KV store?
  1. Most KV store use cases tolerate stale reads for seconds
  2. Shopping cart, session data, user preferences → availability > consistency
  3. Downtime costs more than temporary inconsistency
  4. Conflict resolution (vector clocks) handles the rare actual conflicts
  
When CP is better:
  → Configuration management (etcd), leader election, distributed locks
  → Financial transactions, inventory counters
  → Any case where "two different values for the same key" is unacceptable
```

### Vector Clocks vs Last-Write-Wins: The Conflict Resolution Trade-off

```
Vector Clocks (Dynamo approach):
  ✓ No data loss — concurrent writes are BOTH preserved
  ✓ Client chooses how to merge (application-aware resolution)
  ✗ Vector clock size grows with number of writers (unbounded in theory)
  ✗ Client must handle merge complexity
  ✗ Garbage collection of old clock entries is tricky
  
  Mitigation: Truncate vector clock when it exceeds N entries (e.g., 10)
              → Lose some causal tracking but bounds size
  
Last-Write-Wins (Cassandra approach):
  ✓ Simple — highest timestamp wins, no client merge logic
  ✓ Fixed-size metadata (just a timestamp)
  ✗ LOSES data — concurrent writes, one is silently discarded
  ✗ Depends on synchronized clocks (NTP)
  ✗ Wall clock issues: clock skew can cause "future" writes to always win
  
  Mitigation: Use Hybrid Logical Clocks (HLC) instead of wall clock
              → Combines wall clock + logical counter for causality without clock skew issues
```

**Recommendation**: For interview, discuss vector clocks (shows depth). In production, most systems use LWW (Cassandra) or CRDTs (Riak) because vector clocks' operational complexity is high.

### Consistent Hashing: Why Virtual Nodes Are Essential

```
Without virtual nodes (3 physical nodes on the ring):
  Node A: owns 60% of keys (unlucky hash distribution)
  Node B: owns 25% of keys
  Node C: owns 15% of keys
  → Massively unbalanced! Node A is overloaded.

With virtual nodes (each node has 200 vnodes):
  Node A: 200 points on ring → ~33.3% of keys
  Node B: 200 points on ring → ~33.3% of keys  
  Node C: 200 points on ring → ~33.3% of keys
  → Nearly perfectly balanced (law of large numbers)

Additional benefit — heterogeneous hardware:
  Beefy server with 32GB RAM → assign 300 vnodes
  Small server with 8GB RAM → assign 75 vnodes
  → Capacity-proportional data distribution
  
When a node DIES:
  Without vnodes: ALL its data shifts to ONE neighbor → overload
  With vnodes: Its 200 vnodes are spread across the ring → 
               load spreads evenly across ALL surviving nodes
```

### Compaction Strategy: Size-Tiered vs Leveled

| Factor | Size-Tiered (STCS) | Leveled (LCS) |
|---|---|---|
| **Write amplification** | Low (compact only similarly-sized files) | High (rewrite entire level on promotion) |
| **Read amplification** | High (many overlapping SSTables to check) | Low (no overlap within a level) |
| **Space amplification** | High (up to 2× during compaction) | Low (~10% overhead) |
| **Best for** | Write-heavy, space-insensitive | Read-heavy, space-sensitive |

**Decision**: Start with STCS (better write performance). If read latency becomes a problem (too many SSTables to check), switch to LCS. Many production deployments use STCS for recent data (high write rate) and LCS for older levels (optimize reads).

### Sloppy Quorum vs Strict Quorum

```
Strict Quorum:
  Write MUST go to the designated N replicas for this key
  If 2 of 3 replicas are down → write FAILS
  → Higher consistency, lower availability
  
Sloppy Quorum (this design):
  Write goes to ANY W available nodes (even if they're not the "correct" replicas)
  Temporarily stored on substitute nodes → "hinted handoff" when original recovers
  → Higher availability, but data temporarily on wrong nodes
  
Trade-off:
  Sloppy quorum during partition:
    Client writes to nodes X, Y (substitutes) instead of A, B (correct replicas)
    Read from node C (the only correct replica online) → MISS (data on X, Y!)
    → Consistency violation until hinted handoff completes
    
  This is acceptable for AP systems where availability > consistency
```
