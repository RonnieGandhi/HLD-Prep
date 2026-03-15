# 25. Design Dropbox / Google Drive (File Storage + Sync)

---

## 1. Functional Requirements (FR)

- **Upload files**: Upload files of any type and size (up to 50 GB)
- **Download files**: Download any stored file
- **Sync**: Automatically sync files across multiple devices (desktop, mobile, web)
- **File versioning**: Maintain version history; rollback to previous versions
- **Sharing**: Share files/folders with other users via link or email (view/edit permissions)
- **Offline access**: Work offline; sync changes when reconnected
- **Conflict resolution**: Handle simultaneous edits of the same file by multiple users
- **Notifications**: Notify users when shared files are modified
- **Folder structure**: Support hierarchical folder organization
- **Deduplication**: Don't store duplicate files/chunks multiple times

---

## 2. Non-Functional Requirements (NFRs)

- **Durability**: 99.999999999% (11 nines) — files must NEVER be lost
- **High Availability**: 99.99%
- **Consistency**: Strong consistency for metadata (file exists or doesn't); eventual consistency for sync propagation
- **Low Latency Sync**: Changes propagated to other devices within seconds
- **Bandwidth Efficiency**: Minimize data transfer (send only changed portions of files)
- **Scalability**: 500M+ users, billions of files, exabytes of storage
- **Security**: End-to-end encryption, access control

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Users | 500M |
| Avg files per user | 200 |
| Total files | 100B |
| Avg file size | 500 KB |
| Total storage | 100B × 500 KB = 50 PB |
| With replication (3×) | 150 PB |
| DAU | 100M |
| Files synced / day | 1B |
| Uploads / sec | ~12K |
| Reads / sec | ~50K |
| Avg file change (delta) | 50 KB (10% of file) |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │  Desktop / Mobile Sync Agent                               │        │
│  │                                                            │        │
│  │  • File watcher: detects local file changes (OS events)   │        │
│  │  • Chunker: CDC (content-defined chunking) with Rabin hash │        │
│  │  • Sync engine: compare local chunk list vs server metadata│        │
│  │  • Delta upload: only send new/changed chunks             │        │
│  │  • Dedup: skip chunks already in cloud (by SHA-256 hash)  │        │
│  │  • Offline queue: queue changes when disconnected          │        │
│  └──────────────────────┬─────────────────────────────────────┘        │
│                         │                                              │
└─────────────────────────│──────────────────────────────────────────────┘
                          │
           ┌──────────────┼──────────────────────────────┐
           │              │                              │
    Upload │       Metadata│changes               Receive│notifications
    chunks │              │                              │
           │              │                              │
┌──────────▼──┐   ┌──────▼───────┐          ┌───────────▼──────────────┐
│ Block Server│   │ API Gateway  │          │ Notification Service     │
│             │   └──────┬───────┘          │                          │
│ • Receive   │          │                  │ • WebSocket / Long Poll  │
│   chunk +   │   ┌──────▼───────┐          │   per connected client   │
│   SHA-256   │   │ Metadata     │          │ • Consume file-changed   │
│ • Verify    │   │ Service      │          │   events from Kafka      │
│   integrity │   │              │          │ • Push to all devices    │
│ • Compress  │   │ • File tree  │          │   + shared users         │
│   (LZ4)     │   │ • Chunk refs │          │ • Offline → query on     │
│ • Encrypt   │   │ • Versions   │          │   reconnect (since=ts)   │
│   (AES-256) │   │ • Sharing    │          └──────────────────────────┘
│ • Upload    │   │   permissions│
│   to S3     │   │ • On update: │
│             │   │   publish    │
│ • Dedup:    │   │   file-changed│
│   check     │   │   to Kafka   │
│   hash      │   │              │
│   first     │   └──────┬───────┘
└──────┬──────┘          │
       │          ┌──────▼───────────────────────────────────┐
┌──────▼──────┐   │  DATA LAYER                              │
│ Object Store│   │                                          │
│ (S3 / GCS)  │   │  ┌──────────────┐    ┌──────────────┐   │
│             │   │  │ PostgreSQL   │    │ Redis        │   │
│ /chunks/    │   │  │              │    │              │   │
│  {sha256}/  │   │  │ • files      │    │ • file meta  │   │
│  data       │   │  │ • versions   │    │ • user       │   │
│             │   │  │ • chunk_refs │    │   session    │   │
│ Content-    │   │  │ • shares     │    │ • dedup      │   │
│ addressed:  │   │  │ • namespaces │    │   bloom      │   │
│ same hash = │   │  │              │    │   filter     │   │
│ same chunk  │   │  │ Source of    │    │              │   │
│ (global     │   │  │ truth for    │    │              │   │
│  dedup)     │   │  │ all metadata │    │              │   │
└─────────────┘   │  └──────────────┘    └──────────────┘   │
                  │                                          │
                  │  ┌──────────────┐                        │
                  │  │ Kafka        │                        │
                  │  │              │                        │
                  │  │ file-changed │ → Notification Service │
                  │  │ events       │ → Sync workers         │
                  │  │              │ → Search index (ES)    │
                  │  └──────────────┘                        │
                  │                                          │
                  └──────────────────────────────────────────┘

CONFLICT RESOLUTION:
  Device A (offline edit)    Device B (online edit)
       │                          │
       │   both edit same file    │
       │                          │
       └───────────┬──────────────┘
                   │
            Server detects conflict
            (version mismatch on metadata update)
                   │
            Save both: report.pdf + report (Alice's conflicted copy).pdf
            User manually resolves
```

### Component Deep Dive

#### Chunking — The Key Innovation

**Why chunk files?**
1. **Delta sync**: If a 1 GB file changes slightly, only upload the changed chunks (not the entire file)
2. **Deduplication**: Identical chunks across files/users are stored only once
3. **Parallel upload**: Multiple chunks uploaded simultaneously
4. **Resumable upload**: If upload fails, resume from the last successful chunk

**Chunking Algorithm — Content-Defined Chunking (CDC)** ⭐:

Fixed-size chunks (e.g., 4 MB each) have a problem: inserting 1 byte at the beginning shifts all chunk boundaries → all chunks look "changed."

**Rolling hash (Rabin fingerprint)**:
```python
def chunk_file(data):
    chunks = []
    window_size = 48  # bytes
    min_chunk = 256 * 1024    # 256 KB minimum
    max_chunk = 8 * 1024 * 1024  # 8 MB maximum
    target_chunk = 4 * 1024 * 1024  # 4 MB target
    
    position = 0
    chunk_start = 0
    
    while position < len(data):
        hash = rabin_hash(data[position:position+window_size])
        
        chunk_size = position - chunk_start
        
        # Boundary detected when hash matches a pattern
        if (hash % target_chunk == 0 and chunk_size >= min_chunk) or chunk_size >= max_chunk:
            chunks.append(data[chunk_start:position])
            chunk_start = position
        
        position += 1
    
    # Last chunk
    chunks.append(data[chunk_start:])
    return chunks
```

- Content-defined boundaries are **stable**: inserting data only affects the chunk where the insertion happened
- Hash each chunk with SHA-256 → content-addressable storage
- **Dedup**: Before uploading, check if chunk hash already exists in storage → skip upload

#### Desktop Sync Agent (Client-Side)
- **File system watcher**: Monitors local folder for changes (inotify on Linux, FSEvents on macOS, ReadDirectoryChangesW on Windows)
- **On file change**:
  1. Chunk the file using CDC
  2. Hash each chunk (SHA-256)
  3. Compare with known chunk hashes → identify changed/new chunks
  4. Upload only new chunks to Block Server
  5. Send updated file metadata (chunk list, file size, modified time) to Metadata Service
- **On remote change notification**:
  1. Receive notification that a file was modified on another device
  2. Fetch new metadata from Metadata Service
  3. Compare chunk lists → download only missing/changed chunks
  4. Reconstruct file from chunks locally

#### Block Server
- **Purpose**: Handles chunk upload/download
- **Upload flow**:
  1. Receive chunk data + SHA-256 hash
  2. Verify hash (detect corruption)
  3. Compress chunk (LZ4 for speed, or zstd for better ratio)
  4. Encrypt chunk (AES-256, per-user key)
  5. Upload to object store (S3/GCS)
  6. Return chunk reference (hash + storage location)
- **Download flow**: Reverse — fetch from S3, decrypt, decompress, verify hash, return to client

#### Metadata Service
- **Manages**: File/folder tree, chunk references, sharing permissions, version history
- **Database**: PostgreSQL (ACID transactions for file operations)
- **Cache**: Redis (frequently accessed file metadata)
- **On file update**:
  1. Begin transaction
  2. Create new version entry (old version retained for history)
  3. Update file's chunk list
  4. Commit transaction
  5. Publish `file-changed` event to Kafka

#### Notification Service
- **Purpose**: Notify other devices and shared users when a file changes
- **WebSocket/Long Polling**: Each connected client maintains a persistent connection
- **Flow**: Kafka consumer receives `file-changed` event → look up all devices/users that should be notified → push notification via WebSocket
- **Offline devices**: When device reconnects, it queries Metadata Service for all changes since `last_sync_timestamp`

#### Conflict Resolution
**Scenario**: User A edits file on laptop (offline) while User B edits same file on desktop.

**Strategies**:
1. **Last-Writer-Wins (LWW)**: Latest timestamp wins; other version saved as "conflicted copy"
2. **Dropbox approach**: Save both versions — `report.docx` and `report (Alice's conflicted copy).docx`. Let user manually merge
3. **Operational Transform / CRDT**: For real-time collaborative editing (Google Docs) — merge concurrent edits automatically. Beyond scope for file storage, but relevant for collaborative editing

**Dropbox's actual approach**: Conflicted copies. Simple, no data loss, user resolves.

---

## 5. APIs

### Upload File
```http
POST /api/v1/files/upload/init
{
  "file_name": "report.pdf",
  "file_size": 10485760,
  "parent_folder_id": "folder-uuid",
  "chunk_hashes": ["sha256:abc...", "sha256:def...", "sha256:ghi..."]
}
Response: 200 OK
{
  "file_id": "file-uuid",
  "upload_id": "upload-uuid",
  "chunks_needed": ["sha256:def..."],    // only chunks not already stored (dedup!)
  "upload_urls": {
    "sha256:def...": "https://s3.presigned-url..."
  }
}
```

### Upload Chunk
```http
PUT {presigned_upload_url}
Content-Type: application/octet-stream
Body: [chunk binary data]
```

### Complete Upload
```http
POST /api/v1/files/upload/complete
{
  "upload_id": "upload-uuid",
  "chunk_hashes_ordered": ["sha256:abc...", "sha256:def...", "sha256:ghi..."]
}
```

### Download File
```http
GET /api/v1/files/{file_id}/download
Response: 200 OK
{
  "chunks": [
    {"hash": "sha256:abc...", "url": "https://s3.presigned...", "size": 4194304},
    {"hash": "sha256:def...", "url": "https://s3.presigned...", "size": 4194304},
    {"hash": "sha256:ghi...", "url": "https://s3.presigned...", "size": 2097152}
  ]
}
```

### Get File Metadata
```http
GET /api/v1/files/{file_id}
Response: 200 OK
{
  "file_id": "file-uuid",
  "name": "report.pdf",
  "size": 10485760,
  "modified_at": "2026-03-13T10:00:00Z",
  "version": 5,
  "shared_with": ["user-456"],
  "path": "/Documents/Work/report.pdf"
}
```

### Share File
```http
POST /api/v1/files/{file_id}/share
{
  "user_email": "alice@example.com",
  "permission": "edit"     // "view" or "edit"
}
```

### Get Changes Since
```http
GET /api/v1/sync/changes?since=1710320000&cursor=abc123
```

---

## 6. Data Model

### PostgreSQL — File Metadata

```sql
CREATE TABLE files (
    file_id         UUID PRIMARY KEY,
    owner_id        UUID NOT NULL,
    name            VARCHAR(256) NOT NULL,
    parent_id       UUID,                   -- parent folder
    is_folder       BOOLEAN DEFAULT FALSE,
    size            BIGINT,
    mime_type       VARCHAR(128),
    current_version INT DEFAULT 1,
    is_deleted      BOOLEAN DEFAULT FALSE,  -- soft delete
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP,
    UNIQUE (parent_id, name, owner_id),     -- no duplicate names in same folder
    INDEX idx_owner (owner_id),
    INDEX idx_parent (parent_id)
);
```

### PostgreSQL — File Versions

```sql
CREATE TABLE file_versions (
    version_id      UUID PRIMARY KEY,
    file_id         UUID REFERENCES files,
    version_number  INT,
    size            BIGINT,
    chunk_hashes    TEXT[],                  -- ordered list of chunk hashes
    modified_by     UUID,
    created_at      TIMESTAMP,
    UNIQUE (file_id, version_number)
);
```

### PostgreSQL — Chunks (Content-Addressable)

```sql
CREATE TABLE chunks (
    chunk_hash      VARCHAR(64) PRIMARY KEY, -- SHA-256
    size            INT,
    storage_path    TEXT,                     -- S3 key
    reference_count INT DEFAULT 1,           -- for garbage collection
    compressed_size INT,
    created_at      TIMESTAMP
);
```

### PostgreSQL — Sharing

```sql
CREATE TABLE sharing (
    share_id        UUID PRIMARY KEY,
    file_id         UUID REFERENCES files,
    shared_with     UUID,                    -- user_id
    permission      ENUM('view', 'edit'),
    shared_by       UUID,
    created_at      TIMESTAMP,
    UNIQUE (file_id, shared_with)
);
```

### S3 — Chunk Storage

```
Bucket: file-chunks
Key:    chunks/{sha256_hash_prefix_2chars}/{full_sha256_hash}
Value:  encrypted, compressed chunk binary data
```

### Kafka Topics

```
Topic: file-events      (created, updated, deleted, shared)
Topic: sync-events      (per-user sync notifications)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Chunk loss in S3** | S3 provides 11 nines durability. Cross-region replication for DR |
| **Metadata DB loss** | PostgreSQL with synchronous replication to standby. Daily backups to S3 |
| **Partial upload failure** | Resumable uploads — client retries only failed chunks |
| **Sync conflict** | Conflicted copy created; no data loss |
| **Client crash mid-sync** | On restart, client compares local state with server → resync delta |
| **Dedup reference counting** | Chunk's `reference_count` decremented on file delete. Chunk physically deleted only when count = 0 (background GC) |

### Specific: Data Integrity

1. **End-to-end checksums**: SHA-256 computed client-side before upload → verified server-side after upload → verified again on download
2. **Encryption**: 
   - Client-side encryption (optional, like Boxcryptor): User holds keys, server can't read files
   - Server-side encryption (default): AES-256 with per-user keys managed by KMS
3. **Immutable chunk storage**: Chunks are write-once, never modified → eliminates corruption risk

---

## 8. Additional Considerations

### Bandwidth Optimization
- **Delta sync**: Only upload changed chunks (CDC ensures minimal chunks change)
- **Compression**: LZ4 (fast) or zstd (better ratio) applied to each chunk
- **Deduplication**: Same file uploaded by 1000 users → stored once
  - Cross-user dedup: Hash-based. If chunk hash exists → skip upload
  - Storage savings: 30-50% in typical deployments

### Storage Tiering
- **Hot storage** (SSD-backed S3): Files accessed in last 30 days
- **Warm storage** (S3 Standard): Files accessed in last 90 days
- **Cold storage** (S3 Glacier): Files not accessed in 90+ days
- Automatic tiering based on last access timestamp

### Quota Management
- Free tier: 15 GB
- Paid tiers: 100 GB, 2 TB, unlimited
- Track usage: `user_storage_used = sum(unique chunks owned by user)`
- Shared files: storage counted against the file owner, not the viewer

### Team/Enterprise Features
- **Admin console**: Manage users, set policies, audit activity
- **Audit log**: Who accessed what file, when (compliance/GDPR)
- **Data Loss Prevention (DLP)**: Scan files for sensitive content (SSN, credit card numbers)
- **Legal hold**: Prevent deletion of files under legal investigation
- **Single Sign-On (SSO)**: SAML/OIDC integration

### Real-Time Collaborative Editing (Google Docs Extension)
- Beyond file-level sync: character-level real-time collaboration
- **Operational Transform (OT)** or **CRDT (Conflict-free Replicated Data Types)**
- OT: Central server transforms concurrent operations to maintain consistency
- CRDT: Each character has a unique ID, merge operations are commutative and idempotent → no central coordination needed

---

## 9. Deep Dive: Engineering Trade-offs

### Fixed-Size Chunking vs Content-Defined Chunking (CDC)

```
Fixed-Size Chunks (e.g., every 4 MB):
  File:    [AAAA][BBBB][CCCC][DDDD]
  Insert 1 byte at position 0:
  File:    [xAAA][ABBB][BCCC][CDDD][D...]
  
  Result: ALL 4 chunks changed! (boundaries shifted by 1 byte)
  Upload: Must upload all 4 chunks = 16 MB for a 1-byte change
  
Content-Defined Chunking (Rabin Fingerprint) ⭐:
  Boundaries determined by content (rolling hash matches a pattern)
  File:    [AAA..A][BBB..B][CC..C][DDD..D]
  Insert 1 byte at position 0:
  File:    [xAAA..A][BBB..B][CC..C][DDD..D]
  
  Result: Only chunk 1 changed! (content-defined boundaries are stable)
  Upload: Only 1 chunk = 4 MB for a 1-byte change
  
  How Rabin fingerprint finds boundaries:
    Slide a window across the file, compute rolling hash
    When hash % TARGET_SIZE == MAGIC_VALUE → mark a chunk boundary
    Boundaries naturally appear at the same content points regardless of insertions
    
  Min/max chunk size bounds prevent degenerate cases:
    min = 256 KB (avoid tiny chunks for highly repetitive content)
    max = 8 MB (cap for large non-matching regions)
```

**Savings in practice**: For a 100 MB file with a 1 KB edit, CDC uploads ~4 MB (one chunk) vs 100 MB (all chunks with fixed-size). **25× less bandwidth**.

### Why PostgreSQL for Metadata (Not MongoDB/Cassandra)?

```
File metadata access patterns:
  1. List files in a folder (parent_id = ?, ORDER BY name) → B-tree index
  2. Check if filename exists in folder (UNIQUE constraint on parent_id + name)
  3. Update file version (transaction: new version + update current_version)
  4. Share permissions (relational: file_id → user_id → permission)
  5. Get all files owned by user (for quota calculation)
  6. Get sharing tree (recursive: folder → subfolders → files)

PostgreSQL ✓:
  - #1, #2: B-tree indexes with composite keys
  - #3: ACID transactions for atomic version creation
  - #4: Relational JOINs for permission checks
  - #6: Recursive CTEs for folder tree traversal
  - UNIQUE constraints prevent duplicate filenames in same folder
  - JSON support for flexible file attributes

MongoDB:
  - Could work for basic CRUD
  - But: no real transactions (4.0+ has limited multi-doc txns)
  - No recursive queries → folder tree traversal requires application logic
  - No unique constraint across documents → must handle duplication in app

Cassandra:
  - Terrible fit: no JOINs, no transactions, no unique constraints
  - Folder listing would require denormalized materialized views
```

### Conflict Resolution: OT vs CRDT vs Last-Write-Wins

```
Last-Write-Wins (Dropbox approach for files) ⭐:
  - Simple: latest timestamp wins, loser saved as "conflicted copy"
  - ✓ No data loss (both versions preserved)
  - ✓ No complex merge logic
  - ✗ User must manually resolve conflicts
  - Best for: File-level sync (Dropbox, Google Drive file sync)

Operational Transform (Google Docs approach):
  - Server transforms concurrent operations to maintain consistency
  - Example: User A inserts "X" at position 5, User B deletes position 3
    → Server transforms A's operation: insert "X" at position 4 (adjusted for B's delete)
  - ✓ Real-time character-level collaboration
  - ✗ Requires a central server (single point of coordination)
  - ✗ Complex: O(n²) transformation complexity for n concurrent operations
  - Best for: Real-time collaborative text editing

CRDT (Figma approach):
  - Each character/element has a globally unique ID
  - Merge operations are commutative and idempotent
  - ✓ No central server needed (works peer-to-peer)
  - ✓ Mathematically guaranteed convergence
  - ✗ Higher memory overhead (IDs per character)
  - ✗ Deletion is complex ("tombstones" must persist)
  - Best for: Decentralized collaboration, offline-heavy workflows
```

### Deduplication: Within-User vs Cross-User

```
Within-User Dedup (safe, always do):
  User uploads the same file twice → stored once
  Chunk hash comparison → reference counting
  ✓ Always safe (user's own data)
  ✓ Significant savings (many duplicate files per user)

Cross-User Dedup (privacy-sensitive):
  Two users upload the exact same file → stored once
  ✓ Massive storage savings (30-50% at scale)
  ✗ Privacy concern: You can infer that another user has the same file
    Example: "I uploaded secret_project.pdf and it was instantly deduplicated 
    → someone else already has this exact file!"
  ✗ Legal: Different retention policies for different users complicate deletion
  
  Mitigation: 
    - Only dedup at the chunk level (not file level) → harder to infer
    - Convergent encryption: encrypt each chunk with hash(chunk_content) as key
      → Same content → same ciphertext → dedup works on encrypted data
      → But: this enables the "confirmation attack" described above
    
  Dropbox's actual approach: Cross-user dedup on chunk hashes. 
  Accept the minor privacy trade-off for 30%+ storage savings.
```

### Upload Strategy: Whole File vs Chunked vs Streaming

```
Whole File Upload:
  ✗ If 1 GB upload fails at 999 MB → restart from scratch
  ✗ Memory-intensive for large files
  ✗ Can't deduplicate or delta-sync

Chunked Upload (this design) ⭐:
  ✓ Resumable: if chunk 47 fails, restart only chunk 47
  ✓ Parallel: upload 4 chunks simultaneously
  ✓ Dedup: skip chunks that already exist on server
  ✓ Delta sync: only upload changed chunks
  
  Implementation:
    1. Client chunks file (CDC, 4 MB average)
    2. Client hashes each chunk (SHA-256)
    3. Client sends hash list to server: "Here are my 25 chunk hashes"
    4. Server responds: "I already have chunks 1-23. Upload only chunks 24, 25"
    5. Client uploads only the new chunks
    
  For a 100 MB file with 1 KB change: uploads ~4 MB (1 chunk) instead of 100 MB

Streaming Upload (for very large files):
  ✓ No need to hold entire file in memory
  ✗ More complex server-side reassembly
  ✗ Harder to implement dedup during upload
```
