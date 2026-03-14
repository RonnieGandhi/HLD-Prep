# 86. Design a Blob Storage System (like S3)

---

## 1. Functional Requirements (FR)

- Upload objects (blobs) of any size (1 byte to 5 TB) via PUT
- Download objects via GET with support for range requests (partial downloads)
- Delete objects (immediate metadata removal, lazy storage reclamation)
- List objects in a bucket with prefix filtering and pagination
- Multipart upload: upload large objects in parts, assemble on completion
- Versioning: maintain multiple versions of the same object key
- Object metadata: custom key-value headers, content-type, content-encoding
- Access control: per-bucket and per-object ACLs, IAM policies
- Pre-signed URLs: generate time-limited URLs for upload/download without credentials
- Lifecycle policies: auto-transition to cheaper storage tiers, auto-delete after N days

---

## 2. Non-Functional Requirements (NFRs)

- **Durability**: 99.999999999% (11 nines) — virtually never lose data
- **Availability**: 99.99% for reads, 99.9% for writes
- **Scalability**: Exabytes of storage, millions of objects per bucket
- **Throughput**: 100K+ requests/sec per bucket, multi-Gbps per object
- **Consistency**: Strong read-after-write consistency (S3 achieved this in 2020)
- **Low Latency**: First-byte in < 100ms for most objects
- **Cost Efficiency**: Tiered storage (hot, warm, cold, archive)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total stored objects | 100+ trillion (S3-scale) |
| Total storage | Exabytes |
| Requests / sec (global) | 100M+ |
| Avg object size | 100 KB (highly variable: 1B to 5TB) |
| Metadata per object | ~1 KB (key, size, checksum, ACL, custom headers) |
| Metadata storage | 100T objects × 1KB = 100 PB |
| Write throughput | 10M objects/sec |
| Replication factor | 3 (cross-AZ) + erasure coding for cold data |

---

## 4. High-Level Design (HLD)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                         │
│   SDKs (Python boto3, Java, Go), CLI, Console, REST API               │
└───────────────────────────┬────────────────────────────────────────────┘
                            │ HTTPS
┌───────────────────────────▼────────────────────────────────────────────┐
│                   API GATEWAY / LOAD BALANCER                          │
│  - TLS termination                                                     │
│  - Authentication (IAM signature verification: AWS Signature V4)       │
│  - Rate limiting per account                                           │
│  - Request routing by bucket name                                      │
│  - Large request handling (chunked transfer encoding)                  │
└───────────────────────────┬────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────────────────┐
│                   API SERVICE LAYER                                    │
│                                                                        │
│  ┌────────────────┐  ┌──────────────────┐  ┌────────────────────┐     │
│  │ Object Service │  │ Bucket Service   │  │ IAM / Auth Service │     │
│  │                │  │                  │  │                    │     │
│  │ - PUT object   │  │ - Create bucket  │  │ - Policy eval      │     │
│  │ - GET object   │  │ - List buckets   │  │ - ACL check        │     │
│  │ - DELETE object│  │ - Bucket policies│  │ - Pre-signed URL   │     │
│  │ - HEAD object  │  │ - Versioning cfg │  │   generation       │     │
│  │ - Multipart    │  │ - Lifecycle rules│  │ - Cross-account    │     │
│  │   upload       │  │ - CORS config    │  │   access           │     │
│  └───────┬────────┘  └────────┬─────────┘  └────────────────────┘     │
│          │                    │                                         │
└──────────│────────────────────│─────────────────────────────────────────┘
           │                    │
┌──────────│────────────────────│─────────────────────────────────────────┐
│     METADATA LAYER            │                                         │
│          │                    │                                         │
│  ┌───────▼────────────────────▼────────────────────────┐               │
│  │  Metadata Store (Distributed KV / Sharded DB)       │               │
│  │                                                      │               │
│  │  Key:   bucket_name + "/" + object_key               │               │
│  │  Value: {                                            │               │
│  │    object_id: UUID,                                  │               │
│  │    version_id: UUID,                                 │               │
│  │    size: 1048576,                                    │               │
│  │    content_type: "image/jpeg",                       │               │
│  │    checksum_sha256: "abc123...",                     │               │
│  │    storage_class: "STANDARD",                        │               │
│  │    data_locations: [                                  │               │
│  │      {node: "dn-17", volume: "vol-3", offset: 4096},│               │
│  │      {node: "dn-42", volume: "vol-1", offset: 8192},│               │
│  │      {node: "dn-63", volume: "vol-7", offset: 2048} │               │
│  │    ],                                                │               │
│  │    created_at: "2026-03-14T10:00:00Z",              │               │
│  │    custom_metadata: {"x-amz-meta-author": "alice"}  │               │
│  │  }                                                   │               │
│  │                                                      │               │
│  │  Implementation options:                              │               │
│  │  - DynamoDB / FoundationDB (S3's actual choice)      │               │
│  │  - TiKV + Raft                                       │               │
│  │  - CockroachDB (for strong consistency)              │               │
│  │  - Sharded by hash(bucket + key)                     │               │
│  └──────────────────────────────────────────────────────┘               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    DATA STORAGE LAYER                                    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────┐              │
│  │  Data Nodes (hundreds to thousands)                    │              │
│  │                                                        │              │
│  │  Each data node:                                       │              │
│  │  ┌──────────────────────────────────────┐              │              │
│  │  │  Volumes (logical disk groups)        │              │              │
│  │  │  - Volume 1: [block1][block2][block3] │              │              │
│  │  │  - Volume 2: [block1][block2]         │              │              │
│  │  │                                        │              │              │
│  │  │  Each block: ~64MB (like GFS/HDFS)    │              │              │
│  │  │  Small objects: packed into a single  │              │              │
│  │  │    block with other small objects     │              │              │
│  │  │  Large objects: split across multiple │              │              │
│  │  │    blocks                             │              │              │
│  │  └──────────────────────────────────────┘              │              │
│  │                                                        │              │
│  │  Data placement:                                       │              │
│  │  - 3 replicas across 3 Availability Zones             │              │
│  │  - Placement decided by Placement Service             │              │
│  │  - Rebalancing: background process moves data when     │              │
│  │    nodes added/removed                                 │              │
│  └───────────────────────────────────────────────────────┘              │
│                                                                         │
│  ┌───────────────────────────────────────────────────────┐              │
│  │  Erasure Coding (for cold/archive storage)            │              │
│  │                                                        │              │
│  │  Instead of 3× replication (3× storage cost):         │              │
│  │  Reed-Solomon (10, 4): split into 10 data + 4 parity  │              │
│  │  Storage overhead: 1.4× (vs 3× for replication)       │              │
│  │  Can tolerate loss of any 4 of 14 fragments           │              │
│  │  Trade-off: higher CPU for encode/decode, slower reads │              │
│  │                                                        │              │
│  │  Used for: Infrequent Access, Glacier, Archive tiers  │              │
│  └───────────────────────────────────────────────────────┘              │
│                                                                         │
│  ┌───────────────────────────────────────────────────────┐              │
│  │  Placement Service                                     │              │
│  │  - Tracks all data nodes + volumes + disk health       │              │
│  │  - Decides where to place new object replicas          │              │
│  │  - Ensures replicas span different AZs/racks           │              │
│  │  - Triggers rebalancing on node failure                │              │
│  │  - Consistent hashing ring or CRUSH-like algorithm     │              │
│  └───────────────────────────────────────────────────────┘              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                   BACKGROUND SERVICES                                    │
│                                                                         │
│  ┌────────────────┐  ┌──────────────────┐  ┌──────────────────────┐    │
│  │ Garbage         │  │ Lifecycle        │  │ Replication         │    │
│  │ Collector       │  │ Manager          │  │ Repair              │    │
│  │                 │  │                  │  │                     │    │
│  │ - Scan for      │  │ - Evaluate rules │  │ - Anti-entropy:     │    │
│  │   orphaned data │  │ - Transition to  │  │   compare checksums │    │
│  │   blocks        │  │   cheaper tiers  │  │   across replicas   │    │
│  │ - Delete data   │  │ - Delete expired │  │ - Rebuild lost      │    │
│  │   for deleted   │  │   objects        │  │   replicas (bit-rot │    │
│  │   objects       │  │ - Abort stale    │  │   or disk failure)  │    │
│  │ - Reclaim space │  │   multipart      │  │ - Background scrub  │    │
│  │   from old      │  │   uploads        │  │   (weekly)          │    │
│  │   versions      │  │                  │  │                     │    │
│  └────────────────┘  └──────────────────┘  └──────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Write Path (PUT Object)

```
1. Client → API Gateway → Object Service
2. Object Service authenticates + authorizes request
3. If object > 5GB → require multipart upload
4. Object Service asks Placement Service: "Where to store 3 replicas?"
   → Returns: [dn-17:vol-3, dn-42:vol-1, dn-63:vol-7]
5. Object Service streams data to primary data node (dn-17)
6. Primary replicates to dn-42 and dn-63 (chain replication or parallel)
7. After all 3 replicas written + checksums verified:
   a. Object Service writes metadata to Metadata Store
   b. Returns HTTP 200 to client
8. Metadata write is LAST step → ensures strong read-after-write:
   "If PUT succeeds, subsequent GET always returns new data"
```

### Read Path (GET Object)

```
1. Client → API Gateway → Object Service
2. Object Service queries Metadata Store with (bucket, key)
3. Gets data_locations: [{dn-17, vol-3, offset-4096}, ...]
4. Selects closest/healthiest data node
5. Streams data directly from data node to client
6. Verifies checksum on the fly
7. If checksum mismatch → try next replica + trigger repair

Optimization for small objects (< 256 KB):
  - Cache in Redis / in-memory of API servers
  - Metadata + data cached together

Optimization for large objects:
  - Range requests: GET with Range header → parallel download
  - Multi-connection download (like S3 Transfer Acceleration)
```

---

## 5. APIs

```
# Object operations
PUT    /{bucket}/{key}                    → Upload object
GET    /{bucket}/{key}                    → Download object
HEAD   /{bucket}/{key}                    → Get object metadata
DELETE /{bucket}/{key}                    → Delete object
GET    /{bucket}/{key}?versionId={v}      → Get specific version

# Multipart upload
POST   /{bucket}/{key}?uploads           → Initiate multipart upload
PUT    /{bucket}/{key}?partNumber={n}&uploadId={id}  → Upload part
POST   /{bucket}/{key}?uploadId={id}     → Complete multipart upload
DELETE /{bucket}/{key}?uploadId={id}     → Abort multipart upload

# Bucket operations
PUT    /{bucket}                          → Create bucket
DELETE /{bucket}                          → Delete bucket (must be empty)
GET    /{bucket}?list-type=2&prefix={p}&max-keys=1000  → List objects

# Pre-signed URLs
POST   /api/presign
  { "method": "PUT", "bucket": "my-bucket", "key": "photo.jpg", "expires": 3600 }
  → Returns: "https://s3.example.com/my-bucket/photo.jpg?X-Sig=xxx&X-Expires=3600"
```

---

## 6. Data Models

### Metadata Store (Sharded KV — FoundationDB / DynamoDB)

```
Primary key: (bucket_name, object_key, version_id)
  → Sorted: allows prefix listing, version ordering

{
  "bucket": "my-photos",
  "key": "2026/march/sunset.jpg",
  "version_id": "v_abc123",
  "is_latest": true,
  "size": 2457600,
  "etag": "\"d41d8cd98f00b204e9800998ecf8427e\"",
  "content_type": "image/jpeg",
  "storage_class": "STANDARD",
  "checksum_sha256": "abc123...",
  "data_locations": [
    {"node_id": "dn-17", "volume_id": "vol-3", "offset": 4096, "length": 2457600},
    {"node_id": "dn-42", "volume_id": "vol-1", "offset": 8192, "length": 2457600},
    {"node_id": "dn-63", "volume_id": "vol-7", "offset": 2048, "length": 2457600}
  ],
  "custom_metadata": {
    "x-amz-meta-photographer": "Alice",
    "x-amz-meta-location": "Big Sur"
  },
  "delete_marker": false,
  "created_at": "2026-03-14T10:00:00Z",
  "expires_at": null
}
```

### Data Node — Block Format

```
Volume file (append-only, like a log-structured file):

┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Block 1  │ Block 2  │ Block 3  │ Block 4  │  ...     │
│ (64 MB)  │ (64 MB)  │ (64 MB)  │ (64 MB)  │          │
└──────────┴──────────┴──────────┴──────────┴──────────┘

Within a block (for small objects, "packing"):
┌─────────────────────────────────────────────────────────┐
│ Object A (15 KB) │ Object B (32 KB) │ Object C (8 KB)  │
│ [header][data]   │ [header][data]   │ [header][data]   │
└─────────────────────────────────────────────────────────┘

Header: {object_id, size, checksum, offset_in_block}

For large objects: one object spans multiple blocks
  Block 1: [bytes 0 - 64MB]
  Block 2: [bytes 64MB - 128MB]
  ...
  Metadata stores list of (block_id, offset, length) per chunk
```

### Bucket Metadata (PostgreSQL or DynamoDB)

```sql
CREATE TABLE buckets (
    bucket_name     TEXT PRIMARY KEY,
    owner_id        UUID NOT NULL,
    region          TEXT NOT NULL,
    versioning      TEXT DEFAULT 'disabled', -- disabled|enabled|suspended
    storage_class   TEXT DEFAULT 'STANDARD',
    lifecycle_rules JSONB,
    cors_config     JSONB,
    policy          JSONB,   -- bucket-level IAM policy
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 7. Fault Tolerance & Deep Dives

### How 11-Nines Durability Is Achieved

```
11 nines = lose 1 object per 10 billion objects per year

Strategy stack:
1. 3× replication across AZs (day 1)
   - Probability of all 3 AZs failing simultaneously: ~10^-9
   
2. Checksums at every layer
   - Object-level SHA-256 on write
   - Block-level checksum on disk
   - Network checksum (TCP + TLS)
   - Verified on every read → detect bit-rot
   
3. Background scrubbing
   - Weekly: compare all replica checksums
   - Any mismatch → re-replicate from healthy replica
   
4. Rebuild on disk/node failure
   - When data node fails → Placement Service detects (heartbeat)
   - Triggers re-replication from surviving replicas to new nodes
   - Rebuild rate: throttled to avoid overloading cluster (100 MB/s per node)
   - Full rebuild of 10TB node: ~28 hours (with 100 nodes helping, < 1 hour)

5. Erasure coding for cold data
   - RS(10,4): lose 4 fragments, still recoverable
   - 1.4× overhead vs 3× for replication

6. Cross-region replication (optional)
   - Asynchronous replication to another region
   - Protects against entire region outage
```

### Strong Read-After-Write Consistency

```
Problem (pre-2020 S3): Eventual consistency for new object PUTs
  PUT photo.jpg → 200 OK
  GET photo.jpg → 404 (metadata not yet replicated!)

How S3 achieved strong consistency (2020):
  1. Metadata store uses a strongly consistent distributed database
  2. PUT writes data to storage nodes FIRST
  3. Then writes metadata (atomically, with version number)
  4. GET reads from metadata store (always returns latest version)
  5. If metadata says object exists → data must exist (data written first)
  
  This is "data-before-metadata" pattern:
  - Data orphan (data exists, no metadata) → GC cleans up
  - Metadata orphan (metadata exists, no data) → NEVER HAPPENS
```

### Multipart Upload Recovery

```
Problem: 5TB file upload fails at 90%

Multipart upload:
  1. Initiate: POST /bucket/key?uploads → returns upload_id
  2. Upload parts: PUT with partNumber=1,2,...,10000 (each 5MB-5GB)
  3. Each part is independently replicated and checksummed
  4. Complete: POST with list of {partNumber, ETag}
  5. Server assembles parts into final object (metadata only, no data copy!)

Failure recovery:
  - If part upload fails → retry just that part
  - If client crashes → resume by listing uploaded parts (GET ?uploadId=...)
  - Upload never completed → lifecycle rule aborts after 7 days
  - Parts stored temporarily → GC'd on abort or after timeout
```

### Handling Hot Objects

```
Problem: A viral image gets millions of downloads/sec

Solutions:
1. CDN caching (CloudFront / Fastly)
   - Most objects served from edge → never hits origin
   
2. Request coalescing at API layer
   - 1000 concurrent GETs for same object → 1 read from data node
   - Other 999 wait for first response, all served from cache

3. Read replicas expansion
   - Normally 3 replicas; for hot objects → dynamically add more
   - Load balancer distributes across expanded replica set

4. Pre-signed URL + CDN
   - Generate pre-signed URL pointing to CDN
   - CDN caches object → near-infinite read scalability
```

### Garbage Collection

```
When object deleted:
  1. Metadata: delete marker added (or metadata removed)
  2. Data: NOT immediately deleted (lazy GC)

Why lazy GC?
  - Immediate deletion: random I/O on data nodes, risky during heavy write load
  - Lazy: background process, optimized for batch deletion

GC process:
  1. Scanner reads metadata store: find all live object → data location mappings
  2. Scanner reads data nodes: find all stored blocks
  3. Blocks NOT referenced by any metadata → marked as garbage
  4. After grace period (48h safety): garbage blocks reclaimed
  5. For versioned buckets: only GC non-current versions past lifecycle policy

Similar to: JVM garbage collection, Git gc, LSM tree compaction
```

---

## 8. Additional Considerations

### Storage Tiers

```
| Tier | Use Case | Durability | Availability | Cost | Access Time |
|---|---|---|---|---|---|
| Standard | Frequently accessed | 11 nines | 99.99% | $$$ | < 100ms |
| Infrequent Access | Monthly access | 11 nines | 99.9% | $$ | < 100ms |
| One Zone IA | Non-critical, rare | 11 nines (1 AZ) | 99.5% | $ | < 100ms |
| Glacier | Quarterly access | 11 nines | 99.99% | ¢ | 1-5 min |
| Deep Archive | Yearly access | 11 nines | 99.99% | ¢/10 | 12 hours |

Lifecycle automation:
  Rule: "After 30 days → IA, after 90 days → Glacier, after 365 days → Deep Archive"
  Applied daily by Lifecycle Manager background process
```

### Event Notifications

```
S3 can emit events for:
  - ObjectCreated (PUT, POST, COPY, multipart complete)
  - ObjectRemoved (DELETE)
  - Lifecycle transitions

Events sent to:
  - Kafka / SQS → trigger downstream processing
  - Lambda / Cloud Functions → serverless processing

Use cases:
  - Image uploaded → trigger thumbnail generation
  - Log file uploaded → trigger ETL pipeline
  - Object deleted → update search index
```

### Performance: Small vs Large Objects

```
Small objects (< 256 KB):
  - Bottleneck: metadata operations (not data I/O)
  - Optimization: pack multiple small objects into single block
  - Pre-warm metadata cache for hot prefixes

Large objects (> 1 GB):
  - Bottleneck: network throughput
  - Optimization: multipart upload with parallel parts
  - Transfer Acceleration: use edge locations to route traffic on AWS backbone
  - Multi-connection download: split GET into range requests, download in parallel
```

