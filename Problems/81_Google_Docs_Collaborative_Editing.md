# 81. Design Google Docs (Real-Time Collaborative Editing)

---

## 1. Functional Requirements (FR)

- Multiple users can simultaneously edit the same document in real time
- Each user sees a live cursor and selections of other collaborators
- Changes appear on all clients within ~100ms of being made
- Full version history with ability to view/restore any past version
- Commenting and suggestion mode (track changes)
- Offline editing with automatic conflict resolution on reconnect
- Rich text formatting: bold, italic, headings, lists, tables, images
- Permissions: owner, editor, commenter, viewer
- Document sharing via link with configurable access levels
- Export to PDF, DOCX, plain text

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Local keystroke reflected instantly; remote changes visible within 100–200ms
- **Consistency**: All clients must eventually converge to the same document state (strong eventual consistency)
- **Availability**: 99.99% — editing must work even if some servers are down
- **Scalability**: Support 100M+ documents, up to 200 concurrent editors per document
- **Durability**: Zero data loss — every accepted edit persisted
- **Conflict Resolution**: Concurrent edits must merge automatically without user intervention
- **Offline Support**: Edits queued locally and synced on reconnect
- **Security**: End-to-end encryption in transit, RBAC, audit logs

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total documents | 500M |
| DAU | 50M users |
| Concurrently active docs | 10M |
| Avg editors per active doc | 2–5 (up to 200 max) |
| Operations/sec (global) | 500K (keystrokes, format changes) |
| Avg document size | 50 KB (text) + 500 KB (images/embeds) |
| Storage (docs) | 500M × 550 KB = ~275 TB |
| Version history (30 days) | ~10× raw = 2.75 PB (with delta compression ~500 TB) |
| WebSocket connections | 20M concurrent |

---

## 4. High-Level Design (HLD)

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                   │
│  Browser / Mobile / Desktop App                                   │
│  ┌─────────────────────────────────────────────┐                  │
│  │  Local Document Model (CRDT / OT buffer)    │                  │
│  │  - Apply local edits instantly               │                  │
│  │  - Queue ops for server                      │                  │
│  │  - Apply remote ops from server              │                  │
│  │  - Offline queue (IndexedDB / SQLite)        │                  │
│  └──────────────────────┬──────────────────────┘                  │
│                         │ WebSocket (persistent)                  │
└─────────────────────────│─────────────────────────────────────────┘
                          │
┌─────────────────────────│─────────────────────────────────────────┐
│                  EDGE / GATEWAY LAYER                              │
│                         │                                          │
│              ┌──────────▼──────────┐                               │
│              │   WebSocket Gateway  │  (Sticky sessions per doc)   │
│              │   - Auth validation  │                               │
│              │   - Connection mgmt  │                               │
│              │   - Heartbeat/ping   │                               │
│              │   - Route to correct │                               │
│              │     collaboration    │                               │
│              │     server           │                               │
│              └──────────┬──────────┘                               │
│                         │                                          │
└─────────────────────────│──────────────────────────────────────────┘
                          │
┌─────────────────────────│──────────────────────────────────────────┐
│               COLLABORATION LAYER                                  │
│                         │                                          │
│    ┌────────────────────▼────────────────────┐                     │
│    │      Collaboration Server               │                     │
│    │      (One logical server per document)  │                     │
│    │                                          │                     │
│    │  ┌──────────────────────────────────┐    │                     │
│    │  │   OT Engine / CRDT Engine       │    │                     │
│    │  │   - Receive client ops           │    │                     │
│    │  │   - Transform against concurrent │    │                     │
│    │  │     ops (OT) or merge (CRDT)     │    │                     │
│    │  │   - Assign global sequence number│    │                     │
│    │  │   - Broadcast to all clients     │    │                     │
│    │  └──────────────────────────────────┘    │                     │
│    │                                          │                     │
│    │  ┌──────────────────────────────────┐    │                     │
│    │  │   Presence Manager               │    │                     │
│    │  │   - Cursor positions              │    │                     │
│    │  │   - Selection ranges              │    │                     │
│    │  │   - User colors                   │    │                     │
│    │  │   - Typing indicators             │    │                     │
│    │  └──────────────────────────────────┘    │                     │
│    │                                          │                     │
│    │  ┌──────────────────────────────────┐    │                     │
│    │  │   In-Memory Document State       │    │                     │
│    │  │   - Current doc content           │    │                     │
│    │  │   - Pending op queue              │    │                     │
│    │  │   - Connected client list         │    │                     │
│    │  └──────────────────────────────────┘    │                     │
│    └─────────────┬───────────┬───────────────┘                     │
│                  │           │                                      │
│       ┌──────────▼──┐  ┌────▼─────────┐                            │
│       │ Op Log      │  │  Snapshot    │                             │
│       │ Writer      │  │  Writer     │                              │
│       │ (every op)  │  │  (periodic) │                              │
│       └──────┬──────┘  └──────┬──────┘                              │
│              │                │                                      │
└──────────────│────────────────│──────────────────────────────────────┘
               │                │
┌──────────────│────────────────│──────────────────────────────────────┐
│           DATA LAYER          │                                      │
│              │                │                                      │
│    ┌─────────▼──────┐  ┌─────▼────────┐   ┌─────────────────┐      │
│    │  Operation Log │  │  Document    │   │  Blob Storage   │      │
│    │  (Cassandra /  │  │  Snapshots   │   │  (S3 / GCS)     │      │
│    │  ScyllaDB)     │  │  (PostgreSQL)│   │                 │      │
│    │                │  │              │   │  - Images        │      │
│    │  - doc_id      │  │  - doc_id    │   │  - Embedded files│      │
│    │  - seq_num     │  │  - version   │   │  - Export PDFs   │      │
│    │  - op_type     │  │  - content   │   │                 │      │
│    │  - op_data     │  │  - snapshot  │   └─────────────────┘      │
│    │  - user_id     │  │    at seq_n  │                             │
│    │  - timestamp   │  │  - metadata  │   ┌─────────────────┐      │
│    │                │  │              │   │  Redis           │      │
│    │  (append-only, │  │  (periodic   │   │  - Active doc    │      │
│    │   immutable)   │  │   checkpoint)│   │    sessions      │      │
│    └────────────────┘  └──────────────┘   │  - Presence      │      │
│                                           │  - Cursor cache  │      │
│    ┌────────────────┐                     │  - Permission    │      │
│    │  Metadata DB   │                     │    cache         │      │
│    │  (PostgreSQL)  │                     └─────────────────┘      │
│    │                │                                               │
│    │  - doc_id, title, owner             ┌─────────────────┐      │
│    │  - sharing permissions              │  Search Index    │      │
│    │  - created_at, updated_at           │  (Elasticsearch) │      │
│    │  - folder structure                 │  - Full-text     │      │
│    │  - comments, suggestions            │    search across │      │
│    └────────────────┘                    │    all user docs │      │
│                                          └─────────────────┘      │
└────────────────────────────────────────────────────────────────────┘
```

### Core Design Decision: OT vs CRDT

| Aspect | OT (Operational Transformation) | CRDT (Conflict-free Replicated Data Type) |
|---|---|---|
| **How it works** | Transform concurrent ops against each other relative to a shared server sequence | Each op carries unique IDs; merge is commutative, associative, idempotent |
| **Server required?** | Yes — central server assigns canonical order | No — can merge peer-to-peer |
| **Complexity** | Simpler ops, complex transform functions | Complex data structure, simple merge |
| **Latency** | Must wait for server ACK before confirming | Can apply locally, sync later |
| **Used by** | Google Docs (original), SharePoint | Figma, Yjs, Automerge, Apple Notes |
| **Offline support** | Harder (ops must be rebased) | Natural (merge on reconnect) |
| **Recommendation** | ✅ Use CRDT for new systems — better offline, simpler reasoning |

**Google Docs approach (OT)**:
1. Client generates operation (e.g., `insert('H', pos=5)`)
2. Client sends op to server, applies locally immediately
3. Server assigns sequence number, transforms against any concurrent ops
4. Server broadcasts transformed op to all other clients
5. Other clients transform incoming op against their pending local ops, then apply

**CRDT approach (recommended for new systems)**:
1. Each character has a unique ID: `(siteId, lamportTimestamp)`
2. Insert creates new ID between two existing IDs (fractional indexing)
3. Delete marks character as tombstone (never truly removed)
4. Merge: union of all character sets, ordered by ID — always converges
5. Compaction: periodic garbage collection of tombstones

### Collaboration Server: Why One Per Document

- Each active document is assigned to exactly ONE collaboration server
- This avoids distributed coordination for op ordering
- Server holds document state in memory for fast op processing
- If server dies → another server loads latest snapshot + replays op log
- Consistent hashing or a coordination service (ZooKeeper) assigns doc → server

### Document Session Lifecycle

```
1. User opens doc → WebSocket Gateway routes to assigned Collaboration Server
2. If doc not in memory → Load latest snapshot from PostgreSQL
                        → Replay ops from Cassandra since snapshot
                        → Build in-memory state
3. User types → Client generates op → Sends via WebSocket
4. Server transforms → Assigns seq_num → Persists to op log → Broadcasts
5. Periodically (every 100 ops or 30 sec) → Write snapshot to PostgreSQL
6. Last user leaves → After 5 min idle, evict doc from memory
```

---

## 5. APIs

### WebSocket Protocol (bidirectional)

```
// Client → Server
{
  "type": "operation",
  "doc_id": "d_abc123",
  "client_seq": 42,           // Client's local sequence
  "server_seq": 100,          // Last server seq the client has seen
  "ops": [
    { "type": "insert", "pos": 15, "content": "Hello", "attrs": {"bold": true} },
    { "type": "delete", "pos": 10, "len": 3 },
    { "type": "format", "pos": 5, "len": 10, "attrs": {"italic": true} }
  ]
}

// Server → Client (acknowledgement + broadcast)
{
  "type": "ack",
  "client_seq": 42,
  "server_seq": 105
}

{
  "type": "remote_op",
  "user_id": "u_xyz",
  "server_seq": 106,
  "ops": [...]
}

// Presence updates
{
  "type": "cursor",
  "user_id": "u_xyz",
  "cursor": { "pos": 42, "selection": { "start": 42, "end": 50 } },
  "color": "#FF5733"
}
```

### REST APIs

```
POST   /api/docs                         → Create new document
GET    /api/docs/{doc_id}                → Get document (latest snapshot + pending ops)
DELETE /api/docs/{doc_id}                → Delete document (soft delete)
GET    /api/docs/{doc_id}/history        → Get version history
GET    /api/docs/{doc_id}/version/{v}    → Get doc at specific version
POST   /api/docs/{doc_id}/restore/{v}   → Restore to version v
POST   /api/docs/{doc_id}/share         → Update sharing permissions
POST   /api/docs/{doc_id}/comment       → Add comment
POST   /api/docs/{doc_id}/export?fmt=pdf → Export document
GET    /api/docs?q=search_term          → Search across user's docs
```

---

## 6. Data Models

### Operation Log (Cassandra / ScyllaDB)

**Why Cassandra?** Append-only writes at massive scale, time-series-like access pattern (replay ops in order), wide-column for efficient range queries by doc_id + seq range.

```
Table: operation_log
  Partition Key: doc_id
  Clustering Key: seq_num (ASC)

  doc_id       TEXT      -- Document identifier
  seq_num      BIGINT    -- Server-assigned sequence number
  user_id      TEXT      -- Who made the edit
  op_type      TEXT      -- 'insert' | 'delete' | 'format' | 'compound'
  op_data      BLOB      -- Serialized operation (protobuf)
  timestamp    TIMESTAMP -- Wall clock time
  client_seq   INT       -- Client's local sequence for dedup
```

### Document Snapshots (PostgreSQL)

**Why PostgreSQL?** ACID for metadata, supports JSON for document content, reliable for periodic snapshots.

```sql
CREATE TABLE documents (
    doc_id          UUID PRIMARY KEY,
    title           TEXT,
    owner_id        UUID REFERENCES users(user_id),
    content_snapshot JSONB,       -- Full document state (CRDT/OT)
    snapshot_seq     BIGINT,      -- Seq number this snapshot is at
    created_at       TIMESTAMPTZ,
    updated_at       TIMESTAMPTZ,
    is_deleted       BOOLEAN DEFAULT FALSE
);

CREATE TABLE document_permissions (
    doc_id     UUID REFERENCES documents(doc_id),
    user_id    UUID,  -- NULL for link-sharing
    email      TEXT,  -- For pending invites
    role       TEXT,  -- 'owner' | 'editor' | 'commenter' | 'viewer'
    link_token TEXT,  -- For shareable links
    PRIMARY KEY (doc_id, user_id)
);

CREATE TABLE document_comments (
    comment_id   UUID PRIMARY KEY,
    doc_id       UUID,
    user_id      UUID,
    anchor_start INT,   -- Position in document
    anchor_end   INT,
    content      TEXT,
    parent_id    UUID,  -- For threaded replies
    resolved     BOOLEAN DEFAULT FALSE,
    created_at   TIMESTAMPTZ
);
```

### Redis

```
# Active document sessions
HSET doc:session:{doc_id} user:{user_id} '{"cursor":42,"color":"#FF5733","connected_at":"..."}'

# Permission cache (TTL 5 min)
HSET doc:perms:{doc_id} user:{user_id} "editor"

# Document-to-server mapping
SET doc:server:{doc_id} "collab-server-17" EX 300  # TTL for lease
```

### Elasticsearch

```json
{
  "doc_id": "d_abc123",
  "title": "Q3 Planning Doc",
  "owner_id": "u_xyz",
  "content_text": "Full plaintext extracted from document...",
  "shared_with": ["u_abc", "u_def"],
  "updated_at": "2026-03-14T10:00:00Z",
  "tags": ["planning", "engineering"]
}
```

---

## 7. Fault Tolerance & Deep Dives

### Collaboration Server Failure

```
Problem: Server holding doc X in memory crashes.
Solution:
1. WebSocket Gateway detects broken connection to collab server (heartbeat)
2. Gateway triggers failover: assigns doc to another collab server via ZooKeeper
3. New server loads latest snapshot from PostgreSQL
4. Replays all ops from Cassandra where seq_num > snapshot_seq
5. Clients reconnect (WebSocket auto-reconnect with exponential backoff)
6. Clients send their pending ops (buffered locally)
7. New server transforms and applies pending client ops

Recovery time: < 5 seconds (snapshot load + op replay)
```

### Conflict Resolution (OT Deep Dive)

```
Scenario: User A inserts "X" at position 5, User B deletes character at position 3.
Both ops created when doc state was at seq 100.

Without transformation:
  A's insert at pos 5 → correct
  B's delete at pos 3 → shifts A's insert to pos 4 → WRONG position

With OT:
  Server receives A's op first → applies insert("X", 5) → seq 101
  Server receives B's op (based on seq 100) → must transform:
    B's delete(3) vs A's insert(5): since 3 < 5, delete(3) stays at 3
    BUT A's insert(5) vs B's delete(3): since 5 > 3, adjust to insert(4)
    Server applies delete(3) → seq 102
  
  Server sends transformed ops to clients:
    Client A receives: delete(3) — adjusted because A already applied own insert
    Client B receives: insert("X", 4) — adjusted because B already applied own delete
  
  Both clients converge to same state ✓
```

### CRDT Deep Dive (Recommended Approach)

```
Using RGA (Replicated Growable Array) or Yjs-style CRDT:

Each character = (uniqueID, value, isDeleted, parentID)
UniqueID = (siteID, lamportClock) — globally unique, totally ordered

Insert "A" between "H" and "e" in "Hello":
  Site 1: insert(id=(1,5), value='A', after=id_of_H)
  Site 2 concurrently: insert(id=(2,5), value='B', after=id_of_H)
  
  Both arrive at all replicas. Merge rule:
  When two inserts have same parent, order by uniqueID:
    (1,5) < (2,5) by site_id → 'A' comes before 'B'
  Result: "HBAello" on ALL replicas — deterministic, no server needed

Tombstone GC:
  Deleted chars kept as tombstones (value=∅)
  Periodic compaction: if all replicas have seen delete, remove tombstone
  Use version vector to determine if safe to GC
```

### Offline Editing & Sync

```
1. User goes offline → all edits stored in IndexedDB/SQLite
2. Local CRDT state diverges from server
3. User comes back online:
   a. Client sends all buffered ops to server
   b. Server sends all ops that happened since client's last_seen_seq
   c. CRDT merge: both sides apply all ops — convergence guaranteed
   d. No manual conflict resolution needed (unlike Git)
4. Edge case: user offline for days
   → Client may be many snapshots behind
   → Server sends latest snapshot + ops since snapshot
   → Client rebuilds state and merges local ops
```

### Version History & Snapshotting

```
Strategy: periodic snapshots + op log between snapshots

- Snapshot every 100 ops or every 30 seconds (whichever first)
- Op log kept for 90 days (compliance)
- To view version at time T:
  1. Find latest snapshot before T
  2. Replay ops from snapshot to T
  3. Render document state

Optimization:
- Store snapshots at key moments (every 1000 ops) for fast history access
- Use delta compression between adjacent snapshots (~90% reduction)
- Timeline UI shows snapshots as "version markers"
```

### Scalability: Handling Hot Documents

```
Problem: A viral Google Doc with 200 concurrent editors

Solution:
1. Collaboration server for this doc runs on a beefy machine
2. Presence updates batched (every 100ms, not per keystroke)
3. Op batching: multiple chars typed quickly → single compound op
4. Cursor updates sent via unreliable UDP-like channel (loss acceptable)
5. If > 200 editors → switch to "view-only" for additional users with comment-only access
6. Regional edge servers can handle presence locally, relay only edit ops to central collab server
```

### Consistency Guarantees

```
- Causal consistency: If user A sees user B's edit, all future users see B's edit too
- Convergence: All clients eventually see the same document
- Intent preservation: An insert at "position 5" means between the same two characters,
  regardless of concurrent edits (OT/CRDT guarantee this)
- No lost updates: Every acknowledged op is in the op log (Cassandra RF=3)
- Exactly-once application: Client seq + server seq dedup prevents double-apply
```

---

## 8. Additional Considerations

### Rich Text & Embedded Objects
- Document model as a tree: Document → Paragraphs → Runs (text spans with formatting)
- Images/tables stored as references (blob URL in S3), not inline
- Embedded objects (charts, drawings) are separate CRDT documents linked to main doc

### Permission Model
- Hierarchical: folder permissions cascade to documents
- Link sharing: anyone with link can view/edit (configurable)
- Time-limited access: sharing links with expiry
- Audit log: who accessed/edited what, when (compliance)

### Search
- Full-text search via Elasticsearch
- Index updated asynchronously: snapshot → extract text → index
- Search across all docs user has access to (permission-filtered queries)

### Mobile & Low Bandwidth
- Delta sync: only send changed ops, not full document
- Binary protocol (protobuf) instead of JSON for WebSocket messages
- Adaptive quality: reduce presence updates on slow connections

### Export & Import
- Async job: snapshot → render to PDF/DOCX in background
- Import: parse DOCX/HTML → convert to internal CRDT representation
- Large documents: streaming export to avoid memory spikes

### Real-Time Presence at Scale
```
Problem: 200 users in a doc, each moving cursor → 200 updates × 200 recipients = 40K msgs/sec
Solution:
- Batch cursor updates: aggregate all cursor positions, broadcast every 100ms
- One broadcast message with all 200 cursor positions
- Reduce to 10 broadcasts/sec × 200 recipients = 2K msgs/sec (20× reduction)
```

### Why Not Just Use Git?
```
Git:  Manual merge, line-based, offline-first, explicit commits
Docs: Automatic merge, character-level, real-time, continuous save

Git doesn't work because:
1. Line-based merge → character-level edits cause constant conflicts
2. Manual conflict resolution → unacceptable UX for non-developers
3. No real-time visibility → can't see others typing
4. Commit-based → users expect continuous autosave
```

