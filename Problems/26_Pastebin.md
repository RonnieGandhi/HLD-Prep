# 26. Design Pastebin

---

## 1. Functional Requirements (FR)

- **Create paste**: User submits text content, gets a unique short URL (e.g., `pastebin.com/abc123`)
- **Read paste**: Anyone with the URL can view the paste content
- **Expiration**: Pastes expire after a configurable duration (10 min, 1 hour, 1 day, 1 week, never)
- **Syntax highlighting**: Support multiple programming languages for code display
- **Private pastes**: Only accessible via the exact URL (not listed publicly)
- **User accounts** (optional): Registered users can manage, edit, and delete their pastes
- **Raw view**: View paste as plain text (no formatting)
- **Paste size limit**: Max 10 MB per paste

---

## 2. Non-Functional Requirements (NFRs)

- **High Availability**: 99.99% — reads are critical
- **Low Latency**: Paste loads in < 100 ms
- **Read-Heavy**: Read:Write ratio ≈ 5:1
- **Scalability**: Support millions of pastes, 10K+ reads/sec
- **Durability**: Pastes must not be lost before their expiry
- **Unique URLs**: No collisions on paste identifiers

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| New pastes / day | 1M |
| Reads / day | 5M |
| Writes / sec | ~12 |
| Reads / sec | ~58 (peak 300) |
| Avg paste size | 10 KB |
| Storage / day | 1M × 10 KB = 10 GB |
| Storage / year | 3.6 TB |
| Retention (avg 6 months) | ~1.8 TB active |
| Metadata per paste | 500 bytes |
| Metadata storage | 1M × 500B × 365 = ~180 GB/year |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│    Client    │
│ (Browser/API)│
└──────┬───────┘
       │
┌──────▼───────┐
│ API Gateway  │  ← Rate limiting, CAPTCHA for anonymous users
└──────┬───────┘
       │
       ├────────────────────────────┐
       │                            │
┌──────▼──────┐             ┌──────▼──────┐
│ Write       │             │ Read        │
│ Service     │             │ Service     │
└──────┬──────┘             └──────┬──────┘
       │                           │
       │                    ┌──────▼──────┐
       │                    │    Redis    │  ← Cache hot pastes
       │                    │   (Cache)   │
       │                    └──────┬──────┘
       │                           │ (cache miss)
┌──────▼──────────────────────────▼──────┐
│              Object Store (S3)          │  ← Paste content
└──────┬──────────────────────────────────┘
       │
┌──────▼──────┐
│  Metadata   │  ← Paste metadata (URL, expiry, user, language)
│  DB (MySQL) │
└──────┬──────┘
       │
┌──────▼──────┐
│ Key Gen     │  ← Pre-generated unique short codes
│ Service     │
└─────────────┘

┌──────────────┐
│ Cleanup      │  ← Background job: delete expired pastes
│ Worker       │
└──────────────┘
```

### Component Deep Dive

#### Key Generation — Same as URL Shortener

**Option 1: Base62 Encoding of Auto-Increment ID**
- MySQL auto-increment → Base62 encode → 6-8 char string
- Pros: Simple, guaranteed unique
- Cons: Sequential IDs are predictable (security concern for private pastes)

**Option 2: Pre-Generated Key Service (KGS)** ⭐
- Offline: generate all possible 8-character Base62 keys (62^8 ≈ 218 trillion)
- Store in a key pool DB with two tables: `unused_keys` and `used_keys`
- On paste creation: pop a key from `unused_keys` atomically
- Pros: No collision checking, very fast, non-sequential
- Cons: Slightly more complex setup

**Option 3: MD5/SHA-256 Hash of Content**
- Hash the paste content → take first 8 characters
- Pros: Same content → same URL (deduplication)
- Cons: Collision risk, different metadata for same content creates issues

**Recommendation**: KGS for unique, non-predictable keys. 8 characters of Base62 gives 218 trillion keys — more than enough.

#### Write Service
1. Receive paste content + expiration + language
2. Get a unique key from KGS
3. Store paste content in S3: `s3://pastes/{key_prefix}/{key}`
4. Store metadata in MySQL: key, user_id, expiry, language, size, created_at
5. Return URL: `https://pastebin.com/{key}`

#### Read Service
1. Receive `GET /{key}`
2. Check Redis cache → if hit, return
3. If miss → check MySQL for metadata (exists? expired?)
4. If valid → fetch content from S3
5. Populate Redis cache (TTL = min(paste expiry, 1 hour))
6. Return content with syntax highlighting

#### Content Storage: S3 vs Database
- **Why S3 and not DB**: 
  - Paste content can be up to 10 MB — too large for relational DB rows
  - S3 is optimized for object storage, 11-nines durability, cheap
  - DB stores only metadata (small, indexed, queryable)
- **Compression**: Compress text content with zstd before storing in S3 (3-10× savings for code/text)

#### Cleanup Worker
- Runs every hour
- Query MySQL: `SELECT key FROM pastes WHERE expires_at < NOW()`
- Delete content from S3
- Delete metadata from MySQL
- Invalidate Redis cache
- Alternative: MySQL events + TTL in Cassandra if using Cassandra

---

## 5. APIs

### Create Paste
```http
POST /api/v1/pastes
{
  "content": "print('Hello, World!')",
  "language": "python",
  "expiry": "1d",           // 10m, 1h, 1d, 1w, never
  "is_private": false,
  "title": "My first paste"  // optional
}
Response: 201 Created
{
  "key": "abc12345",
  "url": "https://pastebin.com/abc12345",
  "raw_url": "https://pastebin.com/raw/abc12345",
  "expires_at": "2026-03-14T10:00:00Z"
}
```

### Read Paste
```http
GET /api/v1/pastes/{key}
Response: 200 OK
{
  "key": "abc12345",
  "content": "print('Hello, World!')",
  "language": "python",
  "title": "My first paste",
  "created_at": "2026-03-13T10:00:00Z",
  "expires_at": "2026-03-14T10:00:00Z",
  "views": 42
}
```

### Raw Content
```http
GET /raw/{key}
Response: 200 OK
Content-Type: text/plain
print('Hello, World!')
```

### Delete Paste (for authenticated users)
```http
DELETE /api/v1/pastes/{key}
Authorization: Bearer <token>
Response: 204 No Content
```

---

## 6. Data Model

### MySQL — Paste Metadata

```sql
CREATE TABLE pastes (
    paste_key       CHAR(8) PRIMARY KEY,
    user_id         UUID,                    -- null for anonymous
    title           VARCHAR(256),
    language        VARCHAR(32) DEFAULT 'text',
    content_size    INT,
    s3_path         VARCHAR(256),
    is_private      BOOLEAN DEFAULT FALSE,
    view_count      INT DEFAULT 0,
    expires_at      TIMESTAMP,               -- null = never
    created_at      TIMESTAMP NOT NULL,
    INDEX idx_user (user_id, created_at DESC),
    INDEX idx_expiry (expires_at)
);
```

### S3 — Paste Content

```
Bucket: pastebin-content
Key:    pastes/{first_2_chars_of_key}/{paste_key}
Value:  compressed text content
Metadata: content-type, content-encoding (zstd)
```

**Why prefix with first 2 chars?** S3 partitions by key prefix for performance. Distributing keys across prefixes avoids hot partitions.

### Redis Cache

```
Key:    paste:{key}
Value:  Hash { content, language, title, expires_at, view_count }
TTL:    min(paste_expiry_remaining, 3600)
```

### KGS — Key Pool

```sql
CREATE TABLE unused_keys (
    key_value   CHAR(8) PRIMARY KEY
);

CREATE TABLE used_keys (
    key_value   CHAR(8) PRIMARY KEY,
    used_at     TIMESTAMP
);
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **S3 unavailable** | Multi-region S3 replication; serve from cache if possible |
| **MySQL failure** | Read replicas + standby for failover |
| **KGS failure** | Each app server caches a batch of 1000 keys locally |
| **Expired paste served** | Redis TTL alignment + MySQL check on cache miss |
| **Content corruption** | Checksum stored in metadata; verified on read |
| **Duplicate key** | KGS guarantees uniqueness; DB primary key constraint as safety net |

---

## 8. Additional Considerations

### Abuse Prevention
- **Rate limiting**: Anonymous users: 10 pastes/hour; registered: 100/hour
- **CAPTCHA**: For anonymous paste creation
- **Content scanning**: ML-based detection of malware, phishing, illegal content
- **Size limit**: 10 MB per paste, 512 KB for anonymous users
- **Spam detection**: Flag pastes with known spam patterns

### Analytics
- Track view count per paste (async increment via Kafka → batch update DB)
- Popular pastes dashboard (sorted by view count)
- Language distribution analytics

### Syntax Highlighting
- Server-side: Use Pygments (Python) or highlight.js rendering
- Client-side: Send raw content + language → client renders with highlight.js
- **Recommendation**: Client-side rendering (reduces server load, better UX)

### API for Programmatic Access
- CLI tools: `cat script.py | curl -X POST pastebin.com/api/pastes -d @-`
- SDKs for popular languages
- Authenticated API with API keys for higher rate limits

### Burn After Reading
- Special paste type: deleted after first read
- Implementation: After serving the content, immediately delete from S3 + MySQL
- Use Redis lock to ensure only one reader gets the content

