# 31. Design a Distributed Worker Queue (like RabbitMQ / SQS)

> **Key distinction from Kafka/Message Broker (31_2)**: A worker queue **deletes messages after ACK**, has **no offsets/replay**, supports **push-based delivery**, **per-message routing**, **visibility timeouts**, and **priority queues**. Kafka is an append-only log; this is a traditional message queue.

---

## 1. Functional Requirements (FR)

- **Enqueue messages**: Producers send messages to named queues
- **Dequeue & process**: Workers receive messages, process them, and ACK — message is **deleted** after ACK
- **At-least-once delivery**: Unacknowledged messages are redelivered (visibility timeout)
- **Visibility timeout**: After delivery, message is hidden (not deleted) for a configurable window; if no ACK → becomes visible again
- **Dead Letter Queue (DLQ)**: Messages that fail N times are moved to a DLQ
- **Delay queues**: Messages become visible only after a configurable delay
- **Priority queues**: Higher-priority messages dequeued first
- **Message TTL**: Messages expire if not consumed within a time window
- **Fan-out (pub/sub)**: One published message copied to multiple queues via exchange/topic routing
- **Routing**: Flexible message routing — direct, topic (wildcard), fanout, headers-based (RabbitMQ exchanges)
- **Request-Reply (RPC)**: Correlation ID + reply-to queue for synchronous-over-async patterns
- **FIFO ordering**: Optional strict ordering per queue (with deduplication)
- **Batch operations**: Send/receive/delete messages in batches

---

## 2. Non-Functional Requirements (NFRs)

- **Durability**: Messages must survive broker failures (persisted to disk, replicated)
- **High Availability**: 99.99% — queue unavailability blocks entire pipelines
- **Throughput**: ~50K–100K msg/sec per node (RabbitMQ); effectively unlimited horizontal with SQS
- **Low Latency**: < 5 ms p99 for in-memory queues (RabbitMQ), < 20 ms for SQS
- **Scalability**: Add nodes to increase queue capacity and consumer parallelism
- **Message deduplication** (optional): Exactly-once via dedup ID within a window
- **Backpressure**: Credit-based flow control (push model) or queue depth limits

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Messages / day | 5B |
| Messages / sec | ~58K (peak 200K) |
| Avg message size | 2 KB |
| Throughput | 116 MB/s avg, 400 MB/s peak |
| Avg time-in-queue | 50 ms (real-time) to 5 min (batch) |
| Active queues | 50K |
| Workers (consumers) | 200K |
| Avg visibility timeout | 30 seconds |
| In-flight messages (unACKed) | 58K × 30s = ~1.7M |
| Messages needing persistence | Only unprocessed messages (not retained after ACK) |
| Storage (7-day TTL worst case) | 5B × 2 KB = 10 TB (transient — most deleted within seconds) |

> **Key difference from Kafka**: Storage is transient. Messages are deleted after ACK, so steady-state storage is small (just in-flight + pending messages), not petabytes of retained logs.

---

## 4. High-Level Design (HLD)

```
                    ┌─────────────────────────────────────────────┐
                    │            Producer Pool                     │
                    │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐          │
                    │  │P1   │ │P2   │ │P3   │ │P_N  │          │
                    │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘          │
                    └─────┼──────┼──────┼──────┼──────────────────┘
                          │      │      │      │
                          │   AMQP / HTTPS / gRPC
                          │      │      │      │
                    ┌─────▼──────▼──────▼──────▼──────────────────┐
                    │          Exchange / Router Layer              │
                    │                                               │
                    │  ┌────────────┐  ┌────────────┐              │
                    │  │  Direct    │  │  Fanout    │              │
                    │  │  Exchange  │  │  Exchange  │              │
                    │  └─────┬──┬──┘  └──┬──┬──┬──┘              │
                    │        │  │        │  │  │                   │
                    │  Routing key       Copies to all bound queues│
                    │  matching          (broadcast)               │
                    │                                               │
                    │  ┌────────────┐  ┌────────────┐              │
                    │  │   Topic    │  │  Headers   │              │
                    │  │  Exchange  │  │  Exchange  │              │
                    │  │ *.order.#  │  │ x-type=pdf │              │
                    │  └────────────┘  └────────────┘              │
                    └─────────┬────────────┬───────────────────────┘
                              │            │
              ┌───────────────▼────────────▼──────────────────────┐
              │              Queue Store (Broker Cluster)          │
              │                                                    │
              │  ┌──────────────┐ ┌──────────────┐ ┌───────────┐ │
              │  │  Queue:      │ │  Queue:      │ │  Queue:   │ │
              │  │  "orders"    │ │  "emails"    │ │  "images" │ │
              │  │              │ │              │ │           │ │
              │  │ ┌──────────┐ │ │ ┌──────────┐ │ │┌─────────┐│ │
              │  │ │Ready     │ │ │ │Ready     │ │ ││Ready    ││ │
              │  │ │Messages  │ │ │ │Messages  │ │ ││Messages ││ │
              │  │ │[M1,M2,M5]│ │ │ │[M3,M7]  │ │ ││[M4,M8]  ││ │
              │  │ ├──────────┤ │ │ ├──────────┤ │ │├─────────┤│ │
              │  │ │Unacked   │ │ │ │Unacked   │ │ ││Unacked  ││ │
              │  │ │(In-flight│ │ │ │{M6→W2,  │ │ ││{M9→W5}  ││ │
              │  │ │{M4→W1}  │ │ │ │ M8→W3}  │ │ ││         ││ │
              │  │ ├──────────┤ │ │ ├──────────┤ │ │├─────────┤│ │
              │  │ │Delayed   │ │ │ │DLQ       │ │ ││Priority ││ │
              │  │ │(Timer)   │ │ │ │[M_fail]  │ │ ││Heap     ││ │
              │  │ └──────────┘ │ │ └──────────┘ │ │└─────────┘│ │
              │  └──────────────┘ └──────────────┘ └───────────┘ │
              │                                                    │
              │  Mirrored / Replicated across nodes (Quorum Qs)   │
              └────────────────────┬───────────────────────────────┘
                                   │
                                   │  Push (prefetch) / Pull
                                   │
              ┌────────────────────▼───────────────────────────────┐
              │              Worker Pool (Consumers)                │
              │                                                     │
              │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │
              │  │Worker 1 │  │Worker 2 │  │Worker 3 │            │
              │  │         │  │         │  │         │            │
              │  │Process  │  │Process  │  │Process  │            │
              │  │→ ACK    │  │→ ACK    │  │→ NACK   │            │
              │  │→ Delete │  │→ Delete │  │→ Requeue│            │
              │  └─────────┘  └─────────┘  └─────────┘            │
              └────────────────────────────────────────────────────┘
```

### Component Deep Dive

#### Exchange / Router — Message Routing (RabbitMQ model) ⭐

```
Producers NEVER send directly to a queue. They send to an Exchange.
The Exchange routes messages to queues based on Bindings + Routing Key.

Exchange Types:
┌──────────┬──────────────────────────────────────────────────────────┐
│ Direct   │ Route to queue whose binding key EXACTLY matches        │
│          │ routing key. (1-to-1 routing)                           │
│          │ Ex: routing_key="order.created" → queue "order-process" │
├──────────┼──────────────────────────────────────────────────────────┤
│ Fanout   │ Route to ALL bound queues (ignore routing key)          │
│          │ Ex: "order.created" → queue "email" + "analytics" +     │
│          │     "inventory" (broadcast / pub-sub)                   │
├──────────┼──────────────────────────────────────────────────────────┤
│ Topic    │ Wildcard matching: * = one word, # = zero or more       │
│          │ Ex: "order.*.shipped" matches "order.us.shipped"        │
│          │ Binding "order.#" matches "order.us.west.returned"      │
├──────────┼──────────────────────────────────────────────────────────┤
│ Headers  │ Route based on message header attributes (not routing   │
│          │ key). "x-match=all" or "x-match=any"                   │
└──────────┴──────────────────────────────────────────────────────────┘

This is fundamentally different from Kafka's partition-key routing.
RabbitMQ exchanges give rich, content-based routing.
```

#### Queue — The Core Data Structure ⭐

```
Unlike Kafka (append-only log), a worker queue manages per-message state:

Message States:
  ┌─────────┐    Dequeue    ┌────────────┐    ACK     ┌─────────┐
  │  READY  │──────────────→│ IN-FLIGHT  │───────────→│ DELETED │
  │         │               │ (invisible)│            │         │
  └─────────┘               └─────┬──────┘            └─────────┘
       ▲                          │
       │     Visibility timeout   │ NACK / timeout
       │     expires              │
       └──────────────────────────┘
                                  │ After N retries
                                  ▼
                            ┌──────────┐
                            │   DLQ    │
                            └──────────┘

Internal data structures per queue:
  • Ready list:     Ordered list of messages available for delivery (FIFO or priority heap)
  • Unacked map:    Map<delivery_tag → {message, consumer, expiry_time}> 
  • Delayed set:    Sorted set by delivery_time (timer wheel or min-heap)
  • DLQ:            Separate queue for poison messages
```

#### Storage Engine — NOT an Append-Only Log ⭐

```
Worker queues need RANDOM DELETES (after ACK), not just appends.
This changes the storage design fundamentally vs Kafka.

RabbitMQ (Classic Queues):
  • In-memory queue backed by disk overflow ("paging")
  • Messages start in RAM → if queue grows too large → page to disk
  • ACK → delete from both memory and disk index
  • Problem: Random deletes → disk fragmentation → GC required
  
RabbitMQ (Quorum Queues) ⭐:
  • Raft-based replicated log (WAL) per queue
  • Messages appended to WAL, delivered to consumers
  • ACKs recorded in WAL → compaction removes ACKed messages
  • Better durability than classic queues
  • WAL segments cleaned up after all messages in segment are ACKed

SQS-style (Distributed Hash + Storage):
  • Messages stored in distributed storage (DynamoDB-like)
  • Each message has: {msg_id, queue_id, body, visibility_deadline, receive_count}
  • ReceiveMessage: SELECT WHERE queue_id=X AND visibility_deadline < now LIMIT 10
  • ACK (DeleteMessage): DELETE WHERE msg_id=Y
  • Visibility timeout expires: message becomes visible again automatically

  Storage layout:
  ┌──────────┬──────────┬────────┬─────────────────┬───────────┐
  │ msg_id   │ queue_id │ body   │ visible_after   │ recv_count│
  │ uuid-1   │ orders   │ {...}  │ 2025-01-01T00:00│ 0         │ ← ready
  │ uuid-2   │ orders   │ {...}  │ 2025-01-01T00:35│ 1         │ ← in-flight
  │ uuid-3   │ orders   │ {...}  │ 2025-01-01T00:00│ 3         │ ← ready (failed 3x)
  └──────────┴──────────┴────────┴─────────────────┴───────────┘
```

#### Visibility Timeout — The Core Worker Queue Mechanism ⭐

```
This is the defining feature that separates worker queues from event logs.

Flow:
  1. Worker calls ReceiveMessage → broker picks message M1
  2. M1's visibility_deadline = now + visibility_timeout (e.g., 30s)
  3. M1 is INVISIBLE to other workers for 30 seconds
  4. Worker processes M1 successfully → calls ACK/DeleteMessage → M1 deleted forever
  
  If worker crashes:
  5. 30 seconds pass, no ACK received
  6. M1's visibility_deadline expires → M1 becomes VISIBLE again
  7. Another worker picks up M1 → processes it → ACK → deleted

Why not just delete on receive?
  → If worker crashes after receiving, message is LOST
  → Visibility timeout gives at-least-once without coordinator tracking

Tuning:
  Too short (5s):  Worker might still be processing → message redelivered → DUPLICATE
  Too long (5min): Worker crashes → 5 min delay before retry → slow recovery
  
  Best practice: Set to 2-3× expected processing time
  SQS also supports ChangeMessageVisibility (extend the timeout mid-processing)
```

#### Message Lifecycle (Worker Queue)

```
1. Producer sends message to exchange with routing key
2. Exchange routes message to matching queue(s) based on bindings
3. Broker persists message to queue (disk + memory)
4. Broker ACKs producer ("confirmed" / "publisher confirm")
5. Broker pushes message to consumer (or consumer polls)
6. Message becomes IN-FLIGHT (invisible to other consumers)
7. Consumer processes message
8. Consumer sends ACK → message PERMANENTLY DELETED ⭐
   OR Consumer sends NACK → message requeued (back to ready)
   OR Visibility timeout expires → message becomes visible again
9. After N failures → message moved to DLQ
```

#### Replication — Quorum Queues (RabbitMQ) ⭐

```
Classic Mirrored Queues (legacy, deprecated):
  Primary + N mirrors. ALL mirrors replicate synchronously.
  ✗ Slow (write amplification to all mirrors)
  ✗ Network partition → split-brain → message duplication/loss
  ✗ Adding mirror = full queue sync (blocks queue)

Quorum Queues ⭐ (RabbitMQ 3.8+):
  Raft consensus (majority-based replication)
  
  Queue "orders" replicated across 3 nodes:
    Node A: Leader  (accepts writes + reads)
    Node B: Follower (Raft voter)
    Node C: Follower (Raft voter)
  
  Write path:
    Producer → Leader → replicate to majority (2 of 3) → ACK producer
    If Leader dies → Raft elects new leader from followers
  
  ✓ No split-brain (Raft guarantees single leader)
  ✓ Faster than mirroring (only needs majority, not all)
  ✓ Better availability (survives minority failures)
  ✗ Higher memory usage (Raft log + in-memory queue state)

SQS replication:
  Fully managed — messages stored redundantly across 3 AZs
  Invisible to the user — you just get 99.999999999% durability
```

#### Push vs Pull — Worker Queue is Push-First ⭐

```
RabbitMQ (Push model ⭐):
  Consumer registers with broker: "I want messages from queue X"
  Broker PUSHES messages to consumer as they arrive
  Consumer sets prefetch_count (e.g., 10) — broker sends up to 10 unACKed messages
  
  Advantage: Lowest latency (no polling interval)
  Backpressure: Consumer stops ACKing → broker stops sending (credit-based flow control)

  basic_consume(queue, callback, prefetch_count=10)
  → broker pushes messages → callback(message) → ack(delivery_tag)

SQS (Long-polling):
  Consumer calls ReceiveMessage with WaitTimeSeconds (up to 20s)
  If messages available → return immediately
  If no messages → hold connection open up to 20s → return when message arrives
  
  Not true push, but long-polling reduces empty responses
  More scalable for serverless/Lambda patterns

Compare to Kafka (Pull):
  Consumer polls at its own pace with offset tracking
  → Better for high-throughput replay scenarios
  → Worse latency than push (poll interval delay)
```

---

## 5. APIs

### Producer (SQS-style)
```
SendMessage(queue_url, body, delay_seconds, message_attributes, dedup_id, group_id)
  → {message_id, md5_of_body}

SendMessageBatch(queue_url, entries[{id, body, delay, attrs}])
  → {successful[], failed[]}
```

### Producer (RabbitMQ-style / AMQP)
```
basic_publish(exchange, routing_key, body, properties={
  delivery_mode: 2,         // persistent
  priority: 5,              // 0-9
  expiration: "60000",      // TTL in ms
  correlation_id: "abc",    // for RPC
  reply_to: "reply-queue",  // for RPC
  headers: {x-delay: 5000}  // plugin headers
})
```

### Consumer (SQS-style)
```
ReceiveMessage(queue_url, max_messages=10, wait_time=20, visibility_timeout=30)
  → messages[{message_id, receipt_handle, body, attributes}]

DeleteMessage(queue_url, receipt_handle)    // ACK — permanently remove

ChangeMessageVisibility(queue_url, receipt_handle, new_timeout)  // extend processing time
```

### Consumer (RabbitMQ-style / AMQP)
```
basic_consume(queue, on_message_callback, auto_ack=false, prefetch_count=10)

// In callback:
basic_ack(delivery_tag)                    // success — delete message
basic_nack(delivery_tag, requeue=true)     // failure — put back in queue
basic_reject(delivery_tag, requeue=false)  // failure — discard or DLQ
```

### Admin
```
CreateQueue(name, attributes={
  visibility_timeout: 30,
  message_retention: 345600,      // 4 days
  max_message_size: 262144,       // 256 KB
  delay_seconds: 0,
  dead_letter_queue: "orders-dlq",
  max_receive_count: 5,           // DLQ threshold
  fifo_queue: true,               // SQS FIFO
  content_based_dedup: true
})

DeleteQueue(queue_url)
PurgeQueue(queue_url)              // delete all messages
GetQueueAttributes(queue_url)      // depth, in-flight count, etc.
ListQueues(prefix)
```

---

## 6. Data Model

### Message (On Disk / In Storage)

```
┌──────────────────────────────────────────────────────┐
│ message_id          UUID                              │ globally unique
│ queue_id            VARCHAR                           │ which queue
│ body                BLOB (up to 256 KB)               │ message payload
│ attributes          MAP<string, string>                │ user metadata
│ system_attributes:                                    │
│   sent_timestamp    BIGINT                            │ when produced
│   visible_after     BIGINT                            │ visibility deadline
│   receive_count     INT                               │ delivery attempts
│   first_receive_ts  BIGINT                            │ when first delivered
│   sequence_number   BIGINT (FIFO only)                │ ordering within group
│   message_group_id  VARCHAR (FIFO only)               │ partition key for ordering
│   dedup_id          VARCHAR (FIFO only)               │ deduplication window key
│ receipt_handle      VARCHAR                           │ opaque token for ACK/NACK
│ priority            INT (0-9)                         │ for priority queues
│ expiration          BIGINT                            │ message TTL
│ routing_key         VARCHAR (AMQP)                    │ exchange routing
│ correlation_id      VARCHAR                           │ RPC correlation
│ reply_to            VARCHAR                           │ RPC reply queue
└──────────────────────────────────────────────────────┘
```

### Queue Metadata (Control Plane)

```
/queues/{queue_name}/config → {
  visibility_timeout, max_receive_count, dlq_target,
  retention_period, delay_seconds, max_message_size,
  fifo: bool, content_based_dedup: bool
}

/queues/{queue_name}/stats → {
  messages_ready, messages_in_flight, messages_delayed,
  oldest_message_age, consumers_connected
}

/queues/{queue_name}/bindings → [
  {exchange: "order-events", routing_key: "order.created"},
  {exchange: "order-events", routing_key: "order.cancelled"}
]
```

### Consumer State (Broker-Tracked)

```
Unlike Kafka (consumer tracks its own offset), the BROKER tracks per-message state:

In-Flight Tracking:
  Map<receipt_handle → {
    message_id,
    consumer_id,
    delivery_time,
    visibility_deadline,
    delivery_attempt
  }>

  Timer thread scans for visibility_deadline < now() → requeue message
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Broker failure** | Quorum queue: Raft leader election from followers. SQS: multi-AZ redundancy transparent to user |
| **Message loss** | Publisher confirms (RabbitMQ) / durable queues. Message persisted + replicated BEFORE producer ACK |
| **Consumer failure** | Visibility timeout expires → message becomes visible → another worker picks it up |
| **Poison message** | DLQ after N receive_count. Prevents one bad message from blocking the queue forever |
| **Network partition** | Quorum queues: minority partition becomes read-only, majority continues. Pause-minority mode in RabbitMQ |
| **Duplicate delivery** | At-least-once is default. Consumer-side idempotency (dedup key) for exactly-once |
| **Queue overflow** | Lazy queues (RabbitMQ): page to disk aggressively. SQS: effectively unlimited depth |
| **Slow consumer** | Prefetch limit (push model) or long-poll timeout (pull model). Alert on queue depth growth |

### Publisher Confirms — Ensuring No Message Loss ⭐

```
Fire-and-forget (default):
  Producer sends → no confirmation → message might be lost if broker crashes

Publisher Confirms (RabbitMQ) / SendMessage response (SQS) ⭐:
  Producer sends → broker persists to disk + replicates → sends CONFIRM back
  Producer only considers message "sent" after receiving confirm
  
  If no confirm within timeout → producer retries
  
  RabbitMQ implementation:
    channel.confirm_select()              // enable confirms
    channel.basic_publish(msg)
    channel.wait_for_confirms(timeout=5)  // block until broker confirms
    
    Or async: publisher_confirm_callback(ack/nack, delivery_tag)
    
  Batched confirms:
    Publish 100 messages → wait_for_confirms() → all 100 confirmed at once
    More efficient than confirming each message individually
```

### Race Condition Deep Dives

#### 1. Visibility Timeout Race — Double Processing ⭐

```
Scenario: Worker A receives message M1 with visibility_timeout = 30s.
Worker A processes M1 but takes 35 seconds (slow DB call).

  T=0s:   Worker A receives M1. visibility_deadline = T+30s
  T=30s:  M1's deadline expires. M1 becomes VISIBLE again.
  T=31s:  Worker B receives M1 (it looks like a new message)
  T=35s:  Worker A finishes processing M1. Calls DeleteMessage(receipt_handle_A)
  T=36s:  Worker B finishes processing M1. Calls DeleteMessage(receipt_handle_B)
  
  Result: M1 processed TWICE! Potential double side effects.

Solutions:
  1. ChangeMessageVisibility ⭐: Worker A extends timeout mid-processing
     "I'm still working, give me 30 more seconds"
     Worker A: heartbeat thread extends visibility every 20s while processing
     
  2. Idempotent consumers ⭐:
     Consumer writes result with idempotency key = message_id
     Second processing attempt → ON CONFLICT DO NOTHING / conditional write
     
  3. Receipt handle invalidation (SQS behavior):
     When M1 is redelivered to Worker B, Worker B gets a NEW receipt_handle
     Worker A's old receipt_handle becomes INVALID
     Worker A's DeleteMessage fails (stale handle) → Worker A knows it lost the message
     But Worker A's side effects already happened → still need idempotency!
```

#### 2. Consumer Prefetch Starvation

```
Scenario: RabbitMQ push model with prefetch_count=100.
Consumer A is slow. It has 100 unACKed messages.

  Broker has 500 messages in queue.
  Consumer A: 100 messages in-flight (at capacity, broker stops pushing)
  Consumer B: 0 messages (idle! broker gave everything to A first)
  
  Problem: Consumer B starves because A hogged all the prefetched messages.

Solution: prefetch_count should be SMALL ⭐
  prefetch_count=1:  Perfect fairness, but high latency (1 message at a time)
  prefetch_count=10: Good balance (fair distribution, reasonable batching)
  prefetch_count=100: Only if consumers are fast and uniform

  RabbitMQ also supports global prefetch (shared across all consumers on a channel)
  vs per-consumer prefetch (each consumer gets its own limit)
```

#### 3. FIFO Queue Head-of-Line Blocking

```
SQS FIFO queues use message_group_id for ordered processing.
All messages with same group_id are delivered IN ORDER to ONE consumer.

Scenario: group_id="user-123" has messages [M1, M2, M3, M4]
  M1 delivered to Worker A → Worker A processing (M2, M3, M4 BLOCKED)
  Worker A is slow → entire group "user-123" is stuck!
  
  Meanwhile, group_id="user-456" messages flow freely to other workers.

  This is HEAD-OF-LINE BLOCKING per message group.

Solutions:
  1. Small message groups ⭐: Use fine-grained group IDs
     Don't use group_id="all-orders" (one big group = no parallelism)
     Use group_id="order-{order_id}" (each order is independent)
     
  2. Short visibility timeout: If worker is stuck, message returns quickly
  
  3. Monitor per-group depth: Alert if one group's depth grows disproportionately
```

#### 4. Redelivery Storm After Broker Restart

```
Scenario: Broker crashes and restarts. 10,000 messages were in-flight (unACKed).

  On restart: ALL 10,000 messages become visible simultaneously
  (their visibility deadlines have passed during downtime)
  
  Workers suddenly flooded with 10,000 messages at once → CPU/memory spike
  Workers might crash → messages go back to queue → vicious cycle

Solutions:
  1. Gradual redelivery ⭐: Broker doesn't make all messages visible at once
     Stagger: make 100 visible per second over 100 seconds
     
  2. Consumer rate limiting: Workers limit their own receive rate
     Even if queue has 10K messages, receive at most 50/sec per worker
     
  3. Auto-scaling: CloudWatch/metrics trigger → scale up workers when queue depth spikes
```

### Scale Challenges

#### Scaling a Single Queue to High Throughput

```
Problem: A single queue is a single ordered data structure → hard to parallelize.

RabbitMQ approach:
  Single queue = single Erlang process = single CPU core
  Max throughput per queue: ~30-50K msg/sec
  Scale by: Creating multiple queues + sharding messages across them
  
  Sharded queues:
    Producer hashes message key → picks queue shard
    "orders-0", "orders-1", "orders-2", ... "orders-15"
    Each shard is an independent queue on potentially different nodes
    ✓ 16 shards × 50K = 800K msg/sec
    ✗ No global ordering across shards
    ✗ Client must handle sharding logic

SQS approach:
  Single standard queue = virtually unlimited throughput (internally sharded)
  SQS manages sharding invisibly — you just send/receive
  FIFO queue: 300 msg/sec per group_id, 3000 msg/sec per queue (with batching)
  
  How SQS scales internally:
    Messages distributed across multiple internal partitions
    ReceiveMessage samples from random partitions → eventually consistent visibility
    This is why SQS standard queue does NOT guarantee FIFO (messages from different 
    partitions may arrive out of order)
```

#### Queue Depth Monitoring and Auto-scaling

```
Key metrics to monitor:
  • ApproximateNumberOfMessagesVisible (queue depth)
  • ApproximateNumberOfMessagesNotVisible (in-flight)
  • ApproximateAgeOfOldestMessage
  • NumberOfMessagesSent (enqueue rate)
  • NumberOfMessagesReceived (dequeue rate)

Auto-scaling formula:
  desired_workers = queue_depth / (processing_rate_per_worker × acceptable_drain_time)
  
  Example: 10,000 messages, each worker processes 100/min, want drained in 5 min
  desired_workers = 10,000 / (100 × 5) = 20 workers
  
  SQS + Lambda: Automatic scaling — Lambda spawns new invocations per message
  SQS + ECS/K8s: KEDA or custom HPA based on queue depth metric
```

---

## 8. Additional Considerations

### Delay Queue Implementation

```
Approach 1: Per-message delay (SQS DelaySeconds, RabbitMQ x-delay header)
  Message M1 sent with delay=60s → M1 not visible until now()+60s
  Implementation: Store M1 with visible_after = now()+60s
  Timer/scheduler thread: periodically scan for visible_after < now() → make visible
  
  SQS: DelaySeconds per message (0-900s) or default queue delay
  RabbitMQ: rabbitmq-delayed-message-exchange plugin (stores in Mnesia, timer-based)

Approach 2: Timer wheel (in-broker, high performance) ⭐
  Hierarchical timing wheel:
    Wheel 1 (seconds): 60 slots (0-59s)
    Wheel 2 (minutes): 60 slots (0-59min)
    Wheel 3 (hours):   24 slots (0-23h)
  
  Insert delay=90s → Wheel 2, slot 1 (1 minute), sub-slot 30s
  Timer ticks every second → when slot fires → promote to inner wheel or deliver
  
  O(1) insert and O(1) fire — much better than sorted set for high volume

Approach 3: Sorted set (Redis or DB-backed)
  ZADD delayed_msgs {delivery_timestamp} {message_id}
  Worker polls: ZRANGEBYSCORE delayed_msgs -inf {now} LIMIT 100
  Move due messages to the ready queue
  Simple but requires polling → slight latency
```

### Priority Queue Implementation

```
RabbitMQ supports 0-255 priority levels (recommend 1-10).

Internal structure: One ready sub-queue per priority level
  Priority 9 (highest): [M1, M4]
  Priority 5 (medium):  [M2, M7, M8]
  Priority 1 (lowest):  [M3, M5, M6]

  Dequeue: Always drain highest non-empty priority first
  
  ⚠️ Starvation: Low-priority messages may NEVER be consumed if high-priority
  messages keep arriving. Solution: Aging — increase priority of old messages.

SQS does NOT support priority queues natively.
  Workaround: Separate queues per priority + weighted polling
    "orders-high" → poll 80% of the time
    "orders-low"  → poll 20% of the time
```

### Request-Reply Pattern (RPC over Queue)

```
Client → Request Queue → Server processes → Reply Queue → Client

  Client:
    1. Create exclusive reply queue: "reply-{uuid}"
    2. Publish to "rpc-queue" with:
       correlation_id: "req-123"
       reply_to: "reply-{uuid}"
    3. Consume from "reply-{uuid}", wait for correlation_id match
    
  Server:
    1. Consume from "rpc-queue"
    2. Process request
    3. Publish response to message.reply_to with same correlation_id

  Timeout: Client waits max 5s for reply → if no reply → error

  This is synchronous-over-async: client blocks waiting for response.
  Used by: Microservice RPC, distributed task results (Celery)
```

### Message Deduplication (Exactly-Once Producing)

```
SQS FIFO:
  Producer sends with dedup_id="order-123-payment"
  SQS stores dedup_id for 5-minute window
  If same dedup_id arrives within 5 min → silently dropped (no duplicate)
  
  Content-based dedup (alternative):
  SQS hashes message body → uses hash as dedup_id automatically

RabbitMQ:
  No built-in dedup. Options:
  1. Publisher confirms + idempotent consumer (most common)
  2. Dedup plugin: maintains a cache of recent message IDs
  3. Application-level: check DB before processing
```

---

## 9. Deep Dive: Engineering Trade-offs

### RabbitMQ vs SQS vs Kafka — When to Choose What

| Feature | RabbitMQ | AWS SQS | Kafka |
|---|---|---|---|
| **Model** | Broker (push, AMQP) | Managed queue (poll) | Distributed log (pull) |
| **Message fate** | **Deleted after ACK** ⭐ | **Deleted after ACK** ⭐ | **Retained (log)** |
| **Throughput** | ~50K msg/sec/node | ~3K/sec/queue (standard unlimited) | 1M+ msg/sec |
| **Latency** | < 1ms (in-memory) | 1-10ms | 2-10ms |
| **Ordering** | Per-queue FIFO | FIFO queues (per group) | Per-partition FIFO |
| **Routing** | Exchanges (rich) ⭐ | None | Partition key only |
| **Priority** | ✓ (0-255) ⭐ | ✗ | ✗ |
| **Delay** | Plugin-based | ✓ (0-15min) | ✗ (workaround only) |
| **Replay** | ✗ (deleted on ACK) | ✗ (deleted on ACK) | ✓ (seek to offset) ⭐ |
| **Visibility timeout** | Basic (prefetch + NACK) | ✓ (first-class) ⭐ | N/A (offset-based) |
| **Exactly-once** | ✗ (at-least-once) | ✓ (FIFO dedup) | ✓ (transactional) |
| **Operations** | Moderate (Erlang cluster) | Zero (managed) ⭐ | Complex (brokers, ZK) |
| **Best for** | Complex routing, RPC, low latency | Simple decoupling, serverless | Event streaming, replay |

### Classic Queue vs Quorum Queue vs Stream (RabbitMQ)

```
Classic Queue (legacy):
  Single leader, optional mirrors (synchronous replication to ALL mirrors)
  ✓ Fastest (single-node, in-memory)
  ✗ Not safe: mirror sync can block queue, split-brain possible
  ✗ Deprecated for most use cases

Quorum Queue ⭐ (RabbitMQ 3.8+):
  Raft-based. Majority replication. Leader + N followers.
  ✓ Safe: no split-brain, tolerates minority failures
  ✓ Automatic leader election
  ✓ Poison message handling (delivery_limit built-in)
  ✗ ~20% slower than classic (Raft overhead)
  ✗ No priority queue support
  ✗ No lazy queue (all in memory + WAL)
  Best for: Most production workloads

Stream (RabbitMQ 3.9+):
  Append-only log (Kafka-like!) within RabbitMQ
  ✓ Replay, multiple consumers, high throughput
  ✓ Use RabbitMQ for both queue AND stream workloads
  ✗ No routing/exchange semantics
  ✗ Not a traditional queue (offset-based like Kafka)
  Best for: Event streaming when you already have RabbitMQ
```

### SQS Standard vs FIFO Queue

```
Standard Queue:
  ✓ Nearly unlimited throughput
  ✓ At-least-once delivery
  ✗ Best-effort ordering (NOT strict FIFO)
  ✗ Occasional duplicates
  
  Why not FIFO? SQS standard is internally sharded across many servers.
  Messages on different shards may be delivered out of order.
  Two ReceiveMessage calls may hit different shards.

FIFO Queue:
  ✓ Strict ordering per message_group_id
  ✓ Exactly-once processing (dedup within 5-min window)
  ✗ 300 messages/sec per group, 3000/sec per queue (with batching)
  ✗ Higher latency (ordering requires coordination)
  
  How ordering works:
    Messages with same group_id always delivered in order
    Different group_ids can be processed in parallel
    One in-flight message per group_id → head-of-line blocking within group

  Choose Standard for: High throughput, order doesn't matter
  Choose FIFO for: Financial transactions, user-action ordering
```

### Worker Queue vs Event Log: Fundamental Design Choice

```
┌──────────────────────┬───────────────────────┬────────────────────────┐
│                      │ Worker Queue           │ Event Log (Kafka)      │
│                      │ (RabbitMQ/SQS)         │                        │
├──────────────────────┼───────────────────────┼────────────────────────┤
│ Mental model         │ Task list / inbox      │ Immutable event ledger │
│ Message after ACK    │ DELETED ⭐             │ RETAINED ⭐            │
│ Consumer state       │ Broker tracks          │ Consumer tracks offset │
│ Delivery model       │ Push (broker → worker) │ Pull (consumer → log)  │
│ Routing              │ Rich (exchanges)       │ Partition key only     │
│ Replay               │ Not possible           │ Seek to any offset     │
│ Ordering             │ Per-queue FIFO         │ Per-partition FIFO     │
│ Consumer parallelism │ Add workers freely ⭐  │ Limited by partitions  │
│ Storage growth       │ Bounded (transient)    │ Unbounded (retention)  │
│ Use case             │ Job processing, RPC    │ Event sourcing, CDC    │
│ Analogy              │ Email inbox            │ Git commit log         │
└──────────────────────┴───────────────────────┴────────────────────────┘

When the interviewer says "Design a Worker Queue":
  → Think RabbitMQ/SQS: delete on ACK, visibility timeout, exchanges, DLQ, priority
  → Do NOT talk about offsets, partitions, ISR, consumer groups, log compaction

When the interviewer says "Design a Message Broker / Event Streaming":
  → Think Kafka: append-only log, partitions, offsets, consumer groups, replay
  → See 31_2_Distributed_Message_Broker.md
```

### Exactly-Once Processing in Worker Queues

```
At-least-once is the DEFAULT for all worker queues.
True exactly-once requires consumer-side cooperation:

Pattern 1: Idempotency key ⭐
  Consumer checks: "Have I processed message_id X before?"
  INSERT INTO results (msg_id, result) VALUES (X, ...) ON CONFLICT DO NOTHING
  If duplicate delivery → DB rejects → no side effect

Pattern 2: SQS FIFO dedup
  Producer attaches dedup_id to each message
  SQS drops duplicate sends within 5-minute window
  + Idempotent consumer = end-to-end exactly-once

Pattern 3: Transactional outbox
  Consumer: BEGIN TXN → write result + mark msg_id as processed → COMMIT
  If crash before commit → both rolled back → message retried → safe
  If crash after commit → message already marked → skip on retry
```

### Credit-Based Flow Control (RabbitMQ Backpressure)

```
Problem: If broker pushes messages faster than consumer can process,
consumer's memory fills up → crash.

Credit system:
  Consumer connects → broker grants N credits (e.g., 10)
  Each message pushed = 1 credit consumed
  When consumer ACKs → broker replenishes credits
  When credits = 0 → broker STOPS pushing to that consumer
  
  This is TCP-like flow control at the application level.
  
  tcp_back_pressure: If socket send buffer is full, broker stops reading from queue
  memory_alarm: If broker memory > threshold (40% default), BLOCK all publishers
  disk_alarm: If free disk < threshold, BLOCK all publishers
  
  These are RabbitMQ's multi-level backpressure mechanisms:
    Level 1: Per-consumer prefetch (credit-based)
    Level 2: Per-connection TCP backpressure
    Level 3: Broker-wide memory alarm (block publishers)
    Level 4: Broker-wide disk alarm (block publishers)
```
