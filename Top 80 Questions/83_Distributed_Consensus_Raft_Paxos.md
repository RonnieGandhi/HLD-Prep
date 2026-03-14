# 83. Design a Distributed Consensus System (Raft / Paxos)

---

## 1. Functional Requirements (FR)

- Implement a replicated state machine across N nodes (typically 3, 5, or 7)
- Leader election: automatically elect a leader; re-elect on failure
- Log replication: leader replicates log entries to followers in order
- Linearizable reads and writes: clients see the most recent committed value
- Membership changes: add/remove nodes without downtime (joint consensus)
- Snapshot support: compact log by snapshotting state machine
- Client request forwarding: followers redirect to leader

---

## 2. Non-Functional Requirements (NFRs)

- **Safety**: Never return incorrect results, even during partitions (no split-brain)
- **Liveness**: System makes progress as long as majority of nodes are alive (N/2 + 1)
- **Latency**: Write committed in 1 round-trip (leader → majority followers)
- **Durability**: Committed entries never lost (persisted to stable storage before ACK)
- **Availability**: Tolerate (N-1)/2 node failures (e.g., 2 of 5)
- **Deterministic**: Same log → same state machine state on all nodes

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Cluster size | 3 (dev), 5 (prod), 7 (highly critical) |
| Writes / sec | 10K–100K (bottleneck: disk fsync on leader) |
| Read / sec | 100K+ (with read-only followers or lease-based reads) |
| Log entry size | ~100–500 bytes |
| Log growth | 100K entries/sec × 200 bytes = 20 MB/sec |
| Snapshot interval | Every 10K entries or 100 MB of log |
| Leader election time | 150–300ms (Raft election timeout) |
| Heartbeat interval | 50–100ms |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CLIENT                                        │
│  ┌─────────────────────────────────────────────────┐                 │
│  │  Client Library                                  │                 │
│  │  - Find leader (via any node or service discovery)│                │
│  │  - Send write to leader                          │                 │
│  │  - Retry on leader change                        │                 │
│  │  - Read from leader (linearizable) or follower   │                 │
│  │    (lease-based / stale reads)                   │                 │
│  └──────────────────────┬──────────────────────────┘                 │
└─────────────────────────│────────────────────────────────────────────┘
                          │
┌─────────────────────────│────────────────────────────────────────────┐
│                    RAFT CLUSTER (5 nodes)                             │
│                          │                                            │
│         ┌────────────────▼────────────────┐                           │
│         │          NODE 1 (LEADER)         │                           │
│         │                                  │                           │
│         │  ┌────────────────────────────┐  │                           │
│         │  │   Consensus Module         │  │                           │
│         │  │   - Current term: 42       │  │                           │
│         │  │   - State: LEADER          │  │                           │
│         │  │   - Voted for: self        │  │                           │
│         │  │   - Match index per follower│  │                           │
│         │  │   - Next index per follower │  │                           │
│         │  └─────────────┬──────────────┘  │                           │
│         │                │                  │                           │
│         │  ┌─────────────▼──────────────┐  │                           │
│         │  │   Log (append-only)         │  │                           │
│         │  │   [1: set x=1 | term 40]   │  │                           │
│         │  │   [2: set y=2 | term 40]   │  │                           │
│         │  │   [3: del x   | term 41]   │  │                           │
│         │  │   [4: set z=3 | term 42]   │  │ ← commit index = 3       │
│         │  │   [5: set w=4 | term 42]   │  │ ← uncommitted            │
│         │  └─────────────┬──────────────┘  │                           │
│         │                │                  │                           │
│         │  ┌─────────────▼──────────────┐  │                           │
│         │  │   State Machine             │  │                           │
│         │  │   (Key-Value Store,         │  │                           │
│         │  │    Config Store, etc.)      │  │                           │
│         │  │   Applied through index 3   │  │                           │
│         │  └────────────────────────────┘  │                           │
│         │                                  │                           │
│         │  ┌────────────────────────────┐  │                           │
│         │  │   Stable Storage (WAL)      │  │                           │
│         │  │   - Current term            │  │                           │
│         │  │   - Voted for               │  │                           │
│         │  │   - Log entries (fsync'd)   │  │                           │
│         │  └────────────────────────────┘  │                           │
│         └───────────┬──────────┬───────────┘                           │
│                     │          │                                        │
│      AppendEntries  │          │  AppendEntries                        │
│      (replicate)    │          │  (replicate)                          │
│                     │          │                                        │
│    ┌────────────────▼──┐   ┌──▼──────────────────┐                    │
│    │   NODE 2 (FOLLOWER)│   │  NODE 3 (FOLLOWER)  │                    │
│    │   Term: 42         │   │  Term: 42           │                    │
│    │   Log: [1..4]      │   │  Log: [1..3]        │                    │
│    │   State machine    │   │  State machine      │                    │
│    │    applied: 3      │   │   applied: 3        │                    │
│    └────────────────────┘   └─────────────────────┘                    │
│                                                                        │
│    ┌────────────────────┐   ┌─────────────────────┐                    │
│    │   NODE 4 (FOLLOWER)│   │  NODE 5 (FOLLOWER)  │                    │
│    │   Term: 42         │   │  Term: 42           │                    │
│    │   Log: [1..5]      │   │  Log: [1..4]        │                    │
│    │   State machine    │   │  State machine      │                    │
│    │    applied: 3      │   │   applied: 3        │                    │
│    └────────────────────┘   └─────────────────────┘                    │
│                                                                        │
│  Commit rule: entry committed when replicated to majority (3 of 5)    │
│  Entry 4: on nodes 1,2,4,5 → 4/5 = majority → COMMITTED              │
│  Entry 5: on nodes 1,4 → 2/5 = not majority → UNCOMMITTED            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Raft Algorithm Deep Dive

#### Leader Election

```
Normal operation:
  - Leader sends heartbeats (empty AppendEntries) every 100ms
  - Followers reset election timer on heartbeat receipt

Leader failure:
  1. Follower's election timer expires (random: 150-300ms)
  2. Follower increments term, transitions to CANDIDATE
  3. Votes for self, sends RequestVote to all peers
  4. Includes: candidate's term, last log index, last log term
  5. Other nodes vote YES if:
     a. Candidate's term >= voter's term
     b. Voter hasn't voted in this term yet
     c. Candidate's log is at least as up-to-date as voter's
        (compare last log term, then last log index)
  6. If candidate gets majority → becomes LEADER
  7. If another leader discovered (higher term) → revert to FOLLOWER
  8. If election timeout with no winner → increment term, try again

Election safety:
  - At most ONE leader per term (each node votes once per term)
  - Leader Completeness: elected leader has all committed entries
    (guaranteed by log comparison in step 5c)
```

#### Log Replication

```
1. Client sends write request to leader
2. Leader appends entry to local log: {term, index, command}
3. Leader sends AppendEntries RPC to all followers:
   {
     term: 42,
     leader_id: 1,
     prev_log_index: 4,      // index of entry before new ones
     prev_log_term: 42,      // term of entry at prev_log_index
     entries: [{term:42, cmd:"set w=4"}],
     leader_commit: 3        // leader's commit index
   }
4. Follower checks:
   a. Does my log have entry at prev_log_index with prev_log_term?
   b. If YES → append new entries, reply success
   c. If NO → reply failure (log inconsistency)
5. On failure: leader decrements nextIndex for that follower, retries
   (backtracks until logs match, then replays forward)
6. When majority reply success → leader commits entry, applies to state machine
7. Commit index propagated to followers in next heartbeat
```

#### Log Consistency Guarantee

```
If two logs contain an entry with the same index and term:
  1. The entries store the same command
  2. All preceding entries are identical

This is maintained by:
  - Leader creates at most one entry per index per term
  - AppendEntries consistency check (prev_log_index + prev_log_term)
  - If check fails → follower deletes conflicting entries and accepts leader's
```

---

## 5. APIs (RPC)

### Inter-Node RPCs (Raft Protocol)

```protobuf
service RaftNode {
  rpc AppendEntries(AppendEntriesRequest) returns (AppendEntriesResponse);
  rpc RequestVote(RequestVoteRequest) returns (RequestVoteResponse);
  rpc InstallSnapshot(InstallSnapshotRequest) returns (InstallSnapshotResponse);
}

message AppendEntriesRequest {
  uint64 term = 1;
  string leader_id = 2;
  uint64 prev_log_index = 3;
  uint64 prev_log_term = 4;
  repeated LogEntry entries = 5;
  uint64 leader_commit = 6;
}

message AppendEntriesResponse {
  uint64 term = 1;
  bool success = 2;
  uint64 match_index = 3;  // optimization: tell leader where logs match
}

message RequestVoteRequest {
  uint64 term = 1;
  string candidate_id = 2;
  uint64 last_log_index = 3;
  uint64 last_log_term = 4;
}

message LogEntry {
  uint64 term = 1;
  uint64 index = 2;
  bytes command = 3;  // serialized state machine command
  EntryType type = 4; // NORMAL, CONFIG_CHANGE, NOOP
}
```

### Client API

```
PUT    /api/kv/{key}           → Write key-value (linearizable)
GET    /api/kv/{key}           → Read key-value (linearizable or stale)
DELETE /api/kv/{key}           → Delete key
GET    /api/cluster/status     → Cluster health, leader info
POST   /api/cluster/add_node   → Add node (membership change)
POST   /api/cluster/remove_node → Remove node
```

---

## 6. Data Models

### WAL (Write-Ahead Log) — On-Disk Format

```
File: wal-segment-00042.log

Each entry (binary, protobuf-encoded):
┌──────────┬──────────┬──────────┬──────────────┬──────────┐
│ Length(4B)│ CRC32(4B)│ Term(8B) │ Index(8B)    │ Data(var)│
└──────────┴──────────┴──────────┴──────────────┴──────────┘

- CRC32 for corruption detection
- fsync after every append (safety) or batch fsync (performance)
- Segment rotation: new file every 64 MB
```

### State (Persisted to stable storage, survives restart)

```
{
  "current_term": 42,
  "voted_for": "node-1",       // null if not voted in current term
  "log": [...],                 // WAL
  "snapshot": {
    "last_included_index": 10000,
    "last_included_term": 38,
    "state_machine_data": <binary>
  }
}
```

### Snapshot Format

```
Snapshot is a serialized state machine at a point in time:
  - last_included_index: 10000
  - last_included_term: 38
  - data: serialized key-value map (or whatever the state machine holds)

After snapshot:
  - Delete log entries [1..10000]
  - Keep log entries [10001..current]
  
InstallSnapshot RPC: leader sends snapshot to slow followers
  - Sent in chunks (1 MB each)
  - Follower replaces its state machine + log with snapshot
```

---

## 7. Fault Tolerance & Deep Dives

### Split Brain Prevention

```
Scenario: Network partition splits 5-node cluster into [A, B, C] and [D, E]

Partition [A, B, C] (3 nodes = majority):
  - If leader is in this partition → continues operating normally
  - If leader was D or E → [A,B,C] elect new leader (they have majority)
  - All writes succeed (3/5 majority available)

Partition [D, E] (2 nodes = minority):
  - Cannot elect leader (need 3 votes, only have 2)
  - No writes accepted → clients get redirected/errors
  - Stale reads possible if explicitly allowed

When partition heals:
  - D and E discover higher term from [A,B,C]
  - Revert to followers, sync log from new leader
  - Any uncommitted entries on D/E are overwritten

KEY GUARANTEE: At most ONE leader at any time
  - Because you need majority to win election
  - Two majorities always overlap by at least one node
  - That node can only vote for one candidate per term
```

### Read Scalability

```
Problem: All reads go to leader → leader becomes bottleneck

Solutions (from weakest to strongest consistency):

1. Follower Reads (stale, simplest):
   - Read from any follower
   - May return stale data (follower behind leader)
   - Use case: monitoring dashboards, analytics

2. Read Index (linearizable, no disk I/O):
   - Client sends read to leader
   - Leader records current commit index
   - Leader confirms it's still leader (heartbeat round)
   - Once state machine applied through commit index → respond
   - No log append needed for reads → faster than writes

3. Lease-Based Reads (linearizable, lowest latency):
   - Leader holds a lease (e.g., heartbeat interval × factor)
   - During lease, leader knows no other leader can exist
   - Serve reads locally without heartbeat confirmation
   - Risk: clock skew → lease may overlap with new leader
   - Mitigation: use monotonic clock, conservative lease duration

4. Follower Reads with Read Index:
   - Follower asks leader for current commit index
   - Follower waits until it has applied through that index
   - Then serves the read locally
   - Distributes read load while maintaining linearizability
```

### Membership Changes (Joint Consensus)

```
Problem: Adding/removing nodes risks split-brain during transition

Wrong approach: directly switch from 3-node to 4-node
  - During transition, both old config (2/3 majority) and new config (3/4 majority) 
    could independently elect leaders

Raft solution: Joint Consensus (two-phase)
  Phase 1: Leader logs C_old,new configuration entry
    - Both old AND new configs must agree (majority of old AND majority of new)
    - This entry commits under joint rules
  Phase 2: Leader logs C_new configuration entry
    - Now only new config rules apply
    - Old nodes not in new config step down

Single-server changes (simpler, used in practice):
  - Only add or remove ONE node at a time
  - 3→4→5 or 5→4→3 (never 3→5 directly)
  - Safe because adding one node can't create disjoint majorities
  - Used by etcd, CockroachDB
```

### Liveness vs Safety

```
Safety (always guaranteed):
  - No two leaders in the same term
  - Committed entries are never lost
  - State machines apply same entries in same order

Liveness (guaranteed if):
  - Majority of nodes are running
  - Network eventually delivers messages
  - broadcastTime << electionTimeout << MTBF
    (e.g., 10ms << 200ms << months)
  
If election timeout too short:
  - Frequent unnecessary elections (leader can't heartbeat fast enough)
  
If election timeout too long:
  - Slow recovery after leader failure
```

### Raft vs Paxos vs ZAB Comparison

| Aspect | Raft | Multi-Paxos | ZAB (ZooKeeper) |
|---|---|---|---|
| **Understandability** | ✅ Designed to be simple | ❌ Notoriously complex | Medium |
| **Leader** | Strong leader | Proposer (weak leader) | Leader |
| **Log** | Contiguous, no gaps | Can have gaps, fill later | Contiguous |
| **Membership change** | Joint consensus | Complex | Atomic broadcast |
| **Used by** | etcd, CockroachDB, TiKV | Cassandra (Paxos for LWT), Spanner | ZooKeeper, Kafka (KRaft) |
| **Rounds for commit** | 1 (AppendEntries) | 2 (prepare + accept) unless leader | 1 (proposal) |

---

## 8. Additional Considerations

### Performance Optimizations

```
1. Batching: Buffer multiple client requests, replicate as one AppendEntries
   - Amortizes fsync cost across many entries
   - 10× throughput improvement

2. Pipeline: Send AppendEntries for index N+1 before N is acknowledged
   - Reduces latency from N round-trips to ~1

3. Parallel AppendEntries: Send to all followers simultaneously
   - Commit as soon as any majority responds

4. Pre-vote: Before starting election, candidate asks "would you vote for me?"
   - Prevents disruptive elections from partitioned nodes
   - Node that was partitioned doesn't bump everyone's term on rejoin

5. Read optimization: Lease-based reads (see above)
```

### Where Raft/Paxos Is Used in System Design

```
Almost every distributed system uses consensus internally:
  - etcd (Kubernetes config store): Raft
  - ZooKeeper (Kafka, HBase coordination): ZAB ≈ Paxos
  - CockroachDB: Multi-Raft (one Raft group per range)
  - TiKV: Multi-Raft
  - Google Spanner: Multi-Paxos
  - Kafka (KRaft mode): Raft replacing ZooKeeper
  - Consul: Raft
  - MongoDB (replication): Raft-like protocol

When to mention Raft in interviews:
  - "We use etcd for configuration/leader election" (Raft underneath)
  - "ZooKeeper for distributed coordination" (ZAB ≈ Paxos)
  - "CockroachDB for globally distributed SQL" (Multi-Raft)
  - Any time you need "exactly one leader" or "consistent metadata"
```

### Multi-Raft

```
Single Raft group: all data on 3-5 nodes → limited by single node's disk/CPU

Multi-Raft (CockroachDB, TiKV):
  - Data split into ranges/regions (e.g., key range [a-f] → one Raft group)
  - Each range has its own Raft group (3-5 replicas)
  - Hundreds of Raft groups per node
  - Each group can have a different leader
  
Benefits:
  - Parallel writes across ranges
  - Better load distribution
  - Individual ranges can be moved between nodes

Challenge:
  - Cross-range transactions need 2PC on top of Raft
  - Raft group management (creation, splitting, merging)
  - Heartbeat overhead: N groups × heartbeat_interval
    → Coalesced heartbeats (one message carries heartbeats for all groups)
```

