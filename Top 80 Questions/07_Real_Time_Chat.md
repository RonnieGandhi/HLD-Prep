# 7. Design a Real-Time Chat Application (WhatsApp / Slack)

---

## 1. Functional Requirements (FR)

- **1:1 messaging**: Send and receive text messages between two users in real-time
- **Group messaging**: Support group chats with up to 256 members (WhatsApp) or thousands (Slack channels)
- **Online/Offline presence**: Show whether a user is online, offline, or last seen
- **Read receipts**: Show sent ✓, delivered ✓✓, and read (blue ✓✓) status
- **Media sharing**: Send images, videos, documents, voice messages
- **Message history**: Persist messages and allow history retrieval with pagination
- **Push notifications**: Notify offline users of new messages
- **End-to-end encryption** (for WhatsApp-style): Messages encrypted on device, server cannot read content

---

## 2. Non-Functional Requirements (NFRs)

- **Ultra-Low Latency**: Message delivered in < 100 ms between online users
- **High Availability**: 99.99% uptime — chat is a core communication tool
- **Message Ordering**: Messages within a conversation must appear in correct order
- **Durability**: Messages must never be lost (stored reliably)
- **Scalability**: Support 1B+ users, 100M+ concurrent connections
- **Consistency**: Messages must be delivered at least once (at-least-once, idempotent on client)
- **Offline Support**: Messages sent to offline users are delivered when they come back online

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total users | 2B |
| DAU | 500M |
| Concurrent connections | 100M |
| Messages / day | 50B (~100 msgs per user/day) |
| Messages / sec | ~580K |
| Avg message size | 200 bytes (text) |
| Storage / day | 50B × 200B = 10 TB |
| Storage / year | ~3.6 PB |
| Media messages (10%) | 5B/day × 500KB avg = 2.5 PB/day |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐
│   Client A   │    │   Client B   │
│  (Mobile/Web)│    │  (Mobile/Web)│
└──────┬───────┘    └──────┬───────┘
       │ WebSocket          │ WebSocket
       │                    │
┌──────▼───────┐    ┌──────▼───────┐
│  Chat Server │    │  Chat Server │
│  (WS Handler)│    │  (WS Handler)│
│  Instance 1  │    │  Instance 2  │
└─��────┬───────┘    └──────┬───────┘
       │                    │
       ├────────────────────┤
       │                    │
┌──────▼────────────────────▼───────┐
│         Message Router            │  ← Routes between chat servers
│      (Kafka / Redis Pub-Sub)      │
└──────┬────────────────────┬───────┘
       │                    │
┌──────▼───────┐    ┌──────▼───────┐
│  Session     │    │  Message     │
│  Service     │    │  Service     │
│ (who's on    │    │ (persist &   │
│  which server)│   │  retrieve)   │
└──────┬───────┘    └──────┬───────┘
       │                    │
┌──────▼───────┐    ┌──────▼───────┐
│    Redis     │    │  Cassandra   │
│ (Session +   │    │  (Message    │
│  Presence)   │    │   Store)     │
└──────────────┘    └──────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Media Service│    │  Group       │    │  Push Notif  │
│ (S3 + CDN)  │    │  Service     │    │  Service     │
└──────────────┘    └──────────────┘    └──────────────┘
```

### Component Deep Dive

#### WebSocket Connection & Chat Servers
- **Why WebSocket**: HTTP is request-response (half-duplex). WebSocket provides full-duplex, persistent connection — essential for real-time messaging
- **How**:
  1. Client opens WebSocket connection to a chat server (via load balancer)
  2. Connection is kept alive with heartbeats (every 30 seconds)
  3. Messages are sent/received over this connection in real-time
  4. If connection drops, client reconnects with exponential backoff
- **Scaling**: Each chat server handles ~100K concurrent WebSocket connections
  - 100M concurrent users / 100K per server = **1,000 chat servers**
- **Sticky sessions**: A user's WebSocket must remain on the same server for the duration of the connection. Use consistent hashing on `user_id` at the L4 load balancer
- **Connection state**: Each server maintains an in-memory map: `user_id → WebSocket connection`

#### Session Service + Redis
- **Purpose**: Track which chat server each user is connected to
- **Data**: `session:{user_id}` → `{server_id, connected_at, last_active}`
- **Why Redis**: In-memory, sub-ms lookups, supports TTL for auto-cleanup
- **Flow**: When user connects, chat server registers in Redis. When user disconnects, entry is removed

#### Message Router (Cross-Server Communication)
- **Problem**: User A is on Chat Server 1, User B is on Chat Server 2. How does the message get from Server 1 to Server 2?
- **Solution 1 — Redis Pub/Sub**:
  - Each chat server subscribes to a Redis channel named after its server_id
  - To route a message: look up recipient's server_id in Session Service → publish to that server's channel
  - **Pros**: Simple, low latency
  - **Cons**: Redis Pub/Sub is fire-and-forget (no persistence). If the subscriber is slow, messages can be lost
- **Solution 2 — Kafka** (recommended for durability):
  - Topic per chat server or per user-range
  - Each chat server is a consumer for its assigned partitions
  - **Pros**: Durable, replayable, handles backpressure
  - **Cons**: Slightly higher latency (~5-10ms) vs Redis Pub/Sub (~1ms)
- **Hybrid**: Use Redis Pub/Sub for online→online delivery (fast path). Use Kafka as durable fallback for offline users and reliability

#### Message Service
- **Purpose**: Persist messages, retrieve chat history
- **Write flow**:
  1. Receive message from chat server
  2. Assign a monotonically increasing `message_id` per conversation (using Snowflake or per-conversation counter)
  3. Write to Cassandra
  4. Update conversation's `last_message` in cache
- **Read flow**: Paginated retrieval: `SELECT * FROM messages WHERE conversation_id = ? AND message_id < ? ORDER BY message_id DESC LIMIT 20`

#### Cassandra (Message Store)
- **Why Cassandra**:
  - Write-heavy workload (580K writes/sec)
  - Natural time-series access pattern (messages by conversation, ordered by time)
  - Horizontal scaling, multi-DC replication
  - No complex joins needed
- **Partition key**: `conversation_id` (all messages for a chat are co-located)
- **Clustering key**: `message_id` (ordered retrieval)

#### Presence / Online Status
- **How**: 
  - When user connects → set `presence:{user_id}` = "online" in Redis (with TTL 60s)
  - Heartbeat every 30s resets the TTL
  - If no heartbeat → key expires → user is "offline"
  - Last seen = `last_active` timestamp from session entry
- **Fan-out**: When user's presence changes, notify their contacts who are online
  - For users with many contacts, this is expensive → batch and throttle presence updates
  - Only show presence for active conversations (Slack approach)

#### Group Service
- **Manages**: Group creation, membership, admin roles
- **Group message delivery**:
  1. Sender sends message to group
  2. Group Service fetches member list
  3. For each member: look up session → route message to their chat server
  4. Store one copy of the message in DB (not N copies)
  - **Optimization**: For large groups (1000+ members), use Kafka topic per group. Members' chat servers subscribe to the group topic

#### Media Service
- **Flow**: 
  1. Client uploads media to Media Service (HTTP, not WebSocket — WebSocket is for text)
  2. Media Service stores in S3, generates a CDN URL
  3. Client sends a message with the media URL (not the binary) over WebSocket
  4. Recipient fetches media from CDN

#### End-to-End Encryption (E2EE)
- **Signal Protocol** (used by WhatsApp):
  - Each user generates a key pair (public + private)
  - Public key registered with the server
  - Messages encrypted with recipient's public key on sender's device
  - Server stores encrypted ciphertext — cannot read the content
  - Key exchange uses X3DH (Extended Triple Diffie-Hellman)
  - Forward secrecy via Double Ratchet algorithm

---

## 5. APIs

### Send Message
```
WebSocket Frame:
{
  "type": "send_message",
  "message_id": "client-generated-uuid",
  "conversation_id": "conv-uuid",
  "content": "Hey, how are you?",
  "content_type": "text",
  "timestamp": 1710320000000
}

Server Acknowledgment:
{
  "type": "ack",
  "message_id": "client-generated-uuid",
  "server_message_id": "snowflake-id",
  "status": "sent",
  "timestamp": 1710320000050
}
```

### Receive Message
```
WebSocket Frame (pushed to recipient):
{
  "type": "new_message",
  "message_id": "snowflake-id",
  "conversation_id": "conv-uuid",
  "sender_id": "user-123",
  "content": "Hey, how are you?",
  "content_type": "text",
  "timestamp": 1710320000050
}
```

### Message Status Updates
```
WebSocket Frame:
{
  "type": "message_status",
  "message_id": "snowflake-id",
  "status": "delivered" | "read",
  "timestamp": 1710320001000
}
```

### REST APIs (for non-real-time operations)

```http
GET /api/v1/conversations/{conv_id}/messages?before={message_id}&limit=50
GET /api/v1/conversations
POST /api/v1/groups
POST /api/v1/groups/{group_id}/members
```

---

## 6. Data Model

### Cassandra — Messages

```sql
CREATE TABLE messages (
    conversation_id   UUID,
    message_id        BIGINT,       -- Snowflake ID (time-ordered)
    sender_id         UUID,
    content           TEXT,         -- encrypted ciphertext
    content_type      VARCHAR,      -- text, image, video, voice
    media_url         TEXT,
    created_at        TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

### Cassandra — Conversations per User

```sql
CREATE TABLE user_conversations (
    user_id             UUID,
    last_message_at     TIMESTAMP,
    conversation_id     UUID,
    conversation_type   VARCHAR,     -- '1:1' or 'group'
    last_message_preview TEXT,
    unread_count        INT,
    PRIMARY KEY (user_id, last_message_at)
) WITH CLUSTERING ORDER BY (last_message_at DESC);
```

### Redis — Session & Presence

```
# Session (which server is user connected to)
Key:    session:{user_id}
Value:  Hash { server_id, connected_at, device_type }
TTL:    300 (5 min, refreshed by heartbeat)

# Presence
Key:    presence:{user_id}
Value:  "online"
TTL:    60 (refreshed by heartbeat every 30s)

# Last Seen (persisted separately)
Key:    last_seen:{user_id}
Value:  epoch_timestamp
```

### MySQL — User & Group Metadata

```sql
CREATE TABLE users (
    user_id     UUID PRIMARY KEY,
    phone       VARCHAR(20) UNIQUE,
    name        VARCHAR(128),
    avatar_url  TEXT,
    public_key  BLOB,           -- for E2EE
    created_at  TIMESTAMP
);

CREATE TABLE groups (
    group_id    UUID PRIMARY KEY,
    name        VARCHAR(256),
    avatar_url  TEXT,
    created_by  UUID,
    max_members INT DEFAULT 256,
    created_at  TIMESTAMP
);

CREATE TABLE group_members (
    group_id    UUID,
    user_id     UUID,
    role        ENUM('admin', 'member'),
    joined_at   TIMESTAMP,
    PRIMARY KEY (group_id, user_id)
);
```

### Kafka Topics

```
Topic: messages          (partitioned by conversation_id)
Topic: presence-updates  (partitioned by user_id)
Topic: push-notifications
```

---

## 7. Fault Tolerance

### General
| Technique | Application |
|---|---|
| **Message persistence before ack** | Message written to Cassandra before server sends "sent" ack to sender |
| **Client-side retry** | Client retries sending if no ack received within timeout |
| **Idempotent writes** | `message_id` generated by client → duplicate sends are idempotent |
| **Multi-DC replication** | Cassandra + Redis replicated across data centers |

### Problem-Specific

1. **Chat Server Crashes (User Loses WebSocket)**
   - Client detects connection loss, reconnects to another chat server
   - New server registers new session in Redis
   - Client sends "sync" request: "Give me all messages after message_id X"
   - Server fetches from Cassandra and delivers missed messages
   - **No messages lost**: All messages are persisted in Cassandra before ack

2. **Message Ordering**
   - Scenario: Two rapid messages arrive at different servers out of order
   - **Solution**: 
     - Messages have Snowflake IDs (time-ordered within sender)
     - Client sorts by message_id before displaying
     - Server stores with clustering by message_id → reads are ordered
   
3. **Offline Message Delivery**
   - User B is offline when User A sends a message
   - Message is persisted in Cassandra
   - `unread_count` incremented in `user_conversations`
   - Push notification sent via APNs/FCM
   - When User B comes online → sync: fetch all messages with `message_id > last_seen_message_id`

4. **Group Message Fan-Out**
   - 256-member group: message must be delivered to 256 users
   - **Solution**: Don't fan out to all 256 as individual messages. Store once in DB. Each online member's chat server pulls/receives the message. Offline members sync on reconnect

5. **Presence Thundering Herd**
   - User with 10,000 contacts comes online → must notify all 10K contacts
   - **Solution**: 
     - Only notify contacts who have an active conversation open with this user
     - Batch presence updates (send every 5 seconds, not instantly)
     - For contacts not in the active view, fetch presence lazily when they open the chat

6. **Split Brain (Network Partition Between DCs)**
   - Users in DC-A and DC-B both send messages to the same conversation
   - **Solution**: Cassandra's eventual consistency + LWW (Last-Write-Wins) with Snowflake timestamps. Both messages are preserved (no conflict for chat — append-only model)

---

## 8. Additional Considerations

### Message Sync Protocol
- Each client tracks `last_synced_message_id` per conversation
- On reconnect: `GET /messages?conversation_id=X&after=last_synced_message_id`
- For first-time device: full sync of recent N messages per conversation

### Typing Indicators
- User A starts typing → send `{"type": "typing", "conversation_id": "...", "user_id": "..."}` over WebSocket
- Ephemeral: not persisted, only forwarded to online participants
- Auto-expires after 5 seconds of inactivity

### Message Search
- Full-text search on messages using **Elasticsearch**
- Index: messages by `user_id` + `conversation_id` + `content`
- Note: With E2EE, server can't index encrypted content. Search must happen client-side (WhatsApp approach)

### Multi-Device Support
- One user on phone + tablet + web simultaneously
- Session Service stores multiple sessions per user: `sessions:{user_id}` → SET of `{server_id, device_id}`
- Messages delivered to ALL active devices
- Read receipts synced across devices

### Rate Limiting
- Max 100 messages per minute per user (prevent spam bots)
- Max 256 members per group (WhatsApp) or configurable (Slack)
- Max file size: 100 MB

### Data Retention
- Store messages for 90 days (WhatsApp doesn't store after delivery for E2EE)
- Or indefinitely (Slack enterprise) with archival to cold storage (S3 Glacier)

---

## 9. Deep Dive: Engineering Trade-offs

### WebSocket vs Long Polling vs SSE: Why WebSocket?

| Protocol | How | Latency | Bi-directional | Connection Cost | Best For |
|---|---|---|---|---|---|
| **WebSocket** ⭐ | Persistent TCP connection, full-duplex | < 100ms | ✓ (both client→server and server→client) | 1 TCP connection per client | Chat, gaming, collaborative editing |
| **Long Polling** | Client sends request, server holds until data available, then responds | 100-500ms | ✗ (simulated by new request each time) | New HTTP request per message received | Fallback when WebSocket not available |
| **SSE (Server-Sent Events)** | Persistent HTTP connection, server→client only | < 100ms | ✗ (server→client only; client uses POST for sending) | 1 HTTP connection per client | News feeds, notifications (read-only) |
| **HTTP Polling** | Client polls every N seconds | N seconds (interval) | ✗ | New HTTP request every N seconds | Not suitable for chat |

**Why WebSocket is ideal for chat**:
1. Bi-directional: Client sends messages AND receives messages on the same connection
2. Low latency: No HTTP overhead per message (WebSocket frames are tiny: 2-14 bytes header vs 200+ bytes for HTTP headers)
3. Efficient: 500M users × 1 WebSocket each = 500M connections. With HTTP polling every 1s, that's 500M HTTP requests/sec (30× the load)

**When Long Polling is acceptable**: As a WebSocket fallback behind corporate proxies that block WebSocket upgrades. Implement auto-detection: try WebSocket → if fails, fall back to Long Polling.

### Why Cassandra for Messages (Not MySQL/PostgreSQL/MongoDB)?

```
Message access patterns:
  1. Write a message (append-only, VERY frequent)
  2. Read recent messages for a conversation (time-range query, VERY frequent)
  3. Search messages (not on Cassandra — use Elasticsearch)
  4. Never: JOINs, transactions, or complex queries on messages

Cassandra ✓:
  - Partition key = conversation_id → all messages for a conversation on same node
  - Clustering key = message_id (time-sorted) → efficient range reads
  - LSM-tree storage → optimized for high write throughput (50K msgs/sec easily)
  - TTL support → auto-expire messages after retention period
  - Linear horizontal scaling → add nodes as message volume grows
  - Multi-DC replication built-in → low latency reads globally

MySQL/PostgreSQL ✗:
  - Single-writer primary → write bottleneck at scale
  - Sharding required (by conversation_id) — Cassandra gives this natively
  - No built-in TTL → need cron jobs for data cleanup
  
MongoDB:
  - Could work (document model, sharding, TTL indexes)
  - But: Cassandra's time-series data model (partition + clustering key) is a 
    more natural fit for "messages ordered by time within a conversation"
  - MongoDB's WiredTiger engine handles writes well but Cassandra's LSM-tree 
    is better for append-heavy workloads
```

### E2EE: Why Signal Protocol (Not PGP/TLS)?

```
Option 1: TLS only (transport encryption)
  ✓ Simple — HTTPS encrypts in transit
  ✗ Server sees plaintext → compromised server exposes all messages
  ✗ Government subpoena → server can hand over messages

Option 2: PGP (Pretty Good Privacy)
  ✓ End-to-end encrypted
  ✗ No forward secrecy (one key compromise exposes ALL past messages)
  ✗ Manual key management (user must manage public/private keys)
  ✗ No key ratcheting → same key for all messages in a session

Option 3: Signal Protocol ⭐ (Double Ratchet)
  ✓ End-to-end encrypted
  ✓ Forward secrecy (new ephemeral key per message → past messages safe even if key leaked)
  ✓ Post-compromise security (future messages safe after device recovers from compromise)
  ✓ Transparent key exchange (Diffie-Hellman, no manual key sharing)
  ✗ Complex implementation
  ✗ Server cannot index/search message content (search must be client-side)

Why Signal Protocol wins:
  - Forward secrecy is critical: leaked key only compromises ONE message, not all history
  - Double Ratchet derives a new key per message → each message has a unique encryption key
  - Used by: WhatsApp, Signal, Facebook Messenger (secret conversations), Skype
```

### Message Ordering: The Distributed Systems Challenge

```
Problem: User A sends "Hello" then "How are you?" from two different devices.
  Due to network delays, "How are you?" might arrive at the server BEFORE "Hello"
  
If we use server timestamp → messages appear out of order

Solutions:
  1. Client-generated timestamp → but clocks are not synchronized across devices
  
  2. Snowflake ID ⭐ (recommended):
     - Server assigns a monotonically increasing ID on receipt
     - Within a single server: guaranteed ordered
     - Across servers: approximately ordered (Snowflake embeds timestamp)
     - Per-conversation ordering: route all messages for a conversation to 
       the same Kafka partition (partition key = conversation_id) → strict FIFO
  
  3. Lamport Clock / Vector Clock:
     - Tracks causal ordering (A happened before B)
     - Overkill for chat — Snowflake + Kafka partition ordering is sufficient
  
  4. Client sequence number:
     - Each client maintains a per-conversation sequence counter
     - Server detects gaps: "Got seq 5, missing seq 4" → requests retransmission
     - Useful for offline-first sync
```

### Connection Management: The WebSocket Server Scaling Problem

```
Problem: 500M WebSocket connections. Each connection is stateful (tied to one server).
  If a server dies, all its connections are lost.

Solution 1: Sticky Load Balancing
  - L4 LB hashes client IP → always routes to same WS server
  - ✓ Simple
  - ✗ Uneven distribution; server failure drops all its clients

Solution 2: Connection Registry (Redis) ⭐
  - Redis stores: user_id → {ws_server_id, device_id}
  - When message needs to be delivered:
    1. Lookup recipient's WS server from Redis
    2. Send message to that server via internal RPC
    3. Server pushes to client over existing WebSocket
  - ✓ Decoupled: any server can route to any other
  - ✓ Server failure: client reconnects to any server, Redis updated
  - ✗ Extra hop (Redis lookup + internal RPC)
  
Solution 3: Pub/Sub Fan-Out
  - Each WS server subscribes to user-specific Redis Pub/Sub channels
  - Message for user X → PUBLISH to channel "user:X" → reaches the right server
  - ✓ Simpler than direct RPC routing
  - ✗ Redis Pub/Sub doesn't persist → if server is down, message lost from this channel
  - → Combine with message store (Cassandra) for durability

Production approach:
  Redis for connection registry + internal gRPC for message delivery + 
  Cassandra for persistence + Kafka for async processing
```

### Group Chat: Small Groups vs Large Channels

```
Small Groups (< 256 members, WhatsApp-style):
  - Store members list in a Redis SET
  - Message delivery: iterate members, send to each connected user
  - Fan-out is small (< 256) → done synchronously
  - Every member gets every message (no threading)

Large Channels (1000+ members, Slack/Discord-style):
  - Message delivery is fan-out-on-read (too expensive to push to 100K members)
  - Channel messages stored in Cassandra
  - Client polls for new messages or subscribes to channel WebSocket room
  - Unread count tracked per user per channel (Redis counter)
  - Threading: messages have parent_id → tree structure (like Reddit comments)
  
  Why fan-out-on-read for large channels:
    10K members × 100 messages/hour = 1M deliveries/hour per channel
    With 1000 active channels = 1B deliveries/hour → infeasible with fan-out-on-write
    Instead: each member fetches new messages when they open the channel
```
