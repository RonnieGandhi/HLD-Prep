# HLD Prep — 100 System Design Problems

> **For senior / staff engineers** who already know the fundamentals but need to sharpen depth, nail tricky trade-offs, and plug gaps that cost rounds.

The 100 problems are split into **three progressive batches**:

| Batch | Problems | Goal |
|-------|----------|------|
| **🔴 Batch 1 — Must Do** | 30 | High ROI. Covers the widest spread of categories, each one chosen because it teaches a **non-obvious depth concept** that senior/staff interviews specifically probe. Completing just this batch gives you strong coverage across all major domains. |
| **🟡 Batch 2 — Expand** | 30 | Broadens your arsenal. Fills remaining categories, introduces niche domains (fintech, ad-tech, ML systems), and adds problems where the **differentiator is a specific technique** (CRDT, circuit breaker, consensus). |
| **🟢 Batch 3 — Good to Have** | 40 | Lower marginal ROI. Either simpler problems you can reason through in an interview without deep prep, or very specialized systems. Study if time permits — these won't be your weak link. |

Within each batch, problems are ordered so that **earlier ones build foundations that later ones reference**.

---

## 🔴 Batch 1 — Must Do (30 Problems)

*Goal: maximum topic coverage × maximum depth per problem. Every one of these has a concept that regularly trips up senior candidates.*

| # | Problem | Category | Complexity | Key Concepts / Why It Trips People Up |
|---|----------|----------|------------|---------------------------------------|
| 1 | [URL Shortener](Top%2080%20Questions/01_URL_Shortener.md) | Web Services | ⭐⭐⭐ Medium | Key generation service (KGS), base62 vs hashing, 301 vs 302, cache-aside with Redis, Cassandra partitioning — the "simple" problem where interviewers go **very deep** on trade-offs |
| 2 | [API Rate Limiter](Top%2080%20Questions/02_API_Rate_Limiter.md) | Infrastructure | ⭐⭐⭐ Medium | Token bucket vs sliding window log vs sliding window counter, distributed rate limiting with Redis + Lua, race conditions in `GET-then-SET`, rate limiting at API gateway vs application layer |
| 3 | [Distributed Cache (Redis)](Top%2080%20Questions/27_Distributed_Cache.md) | Infrastructure | ⭐⭐⭐⭐ Hard | Consistent hashing with virtual nodes, eviction policies (LRU/LFU internals), cache stampede / thundering herd, write-through vs write-behind vs write-around, hot key mitigation |
| 4 | [Distributed Worker Queue (RabbitMQ / SQS)](Top%2080%20Questions/31_1_Distributed_Worker_Queue) | Infrastructure | ⭐⭐⭐⭐ Hard | Partitioning & consumer groups, exactly-once semantics, ISR replication, offset management, dead-letter queues, backpressure — **foundational** for half the other problems |
| 5 | [News Feed System](Top%2080%20Questions/06_News_Feed_System.md) | Feed & Social | ⭐⭐⭐⭐ Hard | Fan-out on write vs read vs **hybrid**, celebrity problem, feed ranking with ML, precomputed timelines in Redis, sharding social graph |
| 6 | [Real-Time Chat (WhatsApp)](Top%2080%20Questions/07_Real_Time_Chat.md) | Messaging | ⭐⭐⭐⭐ Hard | WebSocket connection management at scale, message ordering guarantees, read receipts, group chat fan-out, offline message queue, end-to-end encryption architecture |
| 7 | [Notification System](Top%2080%20Questions/05_Notification_System.md) | Infrastructure | ⭐⭐⭐ Medium | Multi-channel delivery (push/SMS/email), priority queues, deduplication, rate limiting per user, template rendering, delivery guarantees vs best-effort |
| 8 | [Unique ID Generator](Top%2080%20Questions/04_Unique_ID_Generator.md) | Infrastructure | ⭐⭐⭐ Medium | Snowflake ID, UUID trade-offs, clock skew handling, database ticket servers, k-sortable IDs — tiny problem but the **depth of clock reasoning** catches people |
| 9 | [Key-Value Store](Top%2080%20Questions/03_Key_Value_Store.md) | Storage | ⭐⭐⭐⭐ Hard | LSM trees vs B-trees, WAL, compaction strategies, consistent hashing, vector clocks for conflict resolution, tunable consistency (quorum reads/writes) |
| 10 | [Search Engine](Top%2080%20Questions/12_Search_Engine.md) | Search | ⭐⭐⭐⭐⭐ Very Hard | Inverted index construction, TF-IDF / BM25 scoring, index sharding & replication, query parsing & spell correction, ranking pipeline (retrieval → re-ranking), snippet generation |
| 11 | [Typeahead / Autocomplete](Top%2080%20Questions/11_Typeahead_Autocomplete.md) | Search | ⭐⭐⭐ Medium | Trie with frequency, precomputed top-K per prefix, data collection pipeline, sampling for scale, prefix-based sharding, Zookeeper for trie updates |
| 12 | [Web Crawler](Top%2080%20Questions/13_Web_Crawler.md)| Data Infra | ⭐⭐⭐⭐ Hard | BFS with politeness, URL frontier design, robots.txt, duplicate detection (simhash/minhash), DNS resolution caching, distributed coordination |
| 13 | [Ride Hailing (Uber)](Top%2080%20Questions/19_Ride_Hailing_Uber.md) | Location & Matching | ⭐⭐⭐⭐⭐ Very Hard | Geospatial indexing (S2/H3 cells), real-time supply/demand matching, dispatch algorithm, ETA prediction, surge pricing trigger, trip state machine |
| 14 | [Distributed Job Scheduler](Top%2080%20Questions/28_Distributed_Job_Scheduler.md) | Infrastructure | ⭐⭐⭐⭐ Hard | Exactly-once execution, task deduplication, priority scheduling, cron expression parsing, fault-tolerant task pickup with distributed locks, job DAGs |
| 15 | [Payment Gateway](Top%2080%20Questions/24_Payment_Gateway.md) | Fintech | ⭐⭐⭐⭐ Hard | Idempotency keys, two-phase payment (authorize → capture), PSP integration, reconciliation, ledger design, PCI compliance, handling partial failures |
| 16 | [Video Streaming Platform (YouTube)](Top%2080%20Questions/15_Video_Streaming_Platform.md) | Media | ⭐⭐⭐⭐ Hard | Adaptive bitrate streaming (HLS/DASH), video transcoding pipeline, CDN edge caching, pre-signed URLs, chunked upload with resume, view count aggregation |
| 17 | [CDN](Top%2080%20Questions/17_CDN.md) | Infrastructure | ⭐⭐⭐⭐ Hard | PoP architecture, cache hierarchies (edge → shield → origin), cache invalidation strategies, consistent hashing for origin shielding, TLS termination, geo-routing |
| 18 | [Dropbox / Google Drive](Top%2080%20Questions/25_Dropbox_Google_Drive.md) | Storage | ⭐⭐⭐⭐⭐ Very Hard | File chunking & deduplication, delta sync (rsync), conflict resolution, block-level storage, metadata DB design, notification of changes across devices |
| 19 | [Distributed Tracing (Jaeger)](Top%2080%20Questions/32_Distributed_Tracing.md) | Observability | ⭐⭐⭐⭐ Hard | Span propagation (context injection), sampling strategies (head vs tail), trace assembly from distributed spans, storage in columnar DBs, trace ID correlation |
| 20 | [Log Aggregation & Search (ELK)](Top%2080%20Questions/40_Log_Aggregation_Search.md) | Observability | ⭐⭐⭐⭐ Hard | Agent → collector → indexer pipeline, structured logging, index rotation & retention, full-text search at scale, hot-warm-cold tiering, log-based alerting |
| 21 | [Event Sourcing System](Top%2080%20Questions/33_Event_Sourcing_System.md) | Architecture | ⭐⭐⭐⭐ Hard | Event log as source of truth, CQRS pattern, projections/materialized views, snapshot optimization, saga pattern for distributed transactions, temporal queries |
| 22 | [Ecommerce Platform](Top%2080%20Questions/22_Ecommerce_Platform.md) | Commerce | ⭐⭐⭐⭐⭐ Very Hard | Catalog service, cart ↔ inventory reservation, order state machine, search with faceted filtering, payment orchestration, seller/buyer split architecture |
| 23 | [Ticketing System (BookMyShow)](Top%2080%20Questions/23_Ticketing_System.md) | Commerce | ⭐⭐⭐⭐ Hard | Seat locking with TTL, distributed transactions for booking, double-booking prevention, pessimistic vs optimistic locking, waiting queue fairness |
| 24 | [Distributed Lock Manager](Top%2080%20Questions/29_Distributed_Lock_Manager.md) | Infrastructure | ⭐⭐⭐⭐ Hard | Redlock algorithm, fencing tokens, lease-based locks, ZooKeeper vs etcd for coordination, lock contention & fairness, clock drift issues |
| 25 | [Google Docs (Collaborative Editing)](Top%2080%20Questions/81_Google_Docs_Collaborative_Editing.md) | Real-time Collab | ⭐⭐⭐⭐⭐ Very Hard | OT (Operational Transformation) vs CRDT, cursor position sync, version vectors, WebSocket session management, conflict-free merges, undo/redo in collaborative context |
| 26 | [Fraud Detection System](Top%2080%20Questions/74_Fraud_Detection_System.md) | ML / Security | ⭐⭐⭐⭐ Hard | Real-time feature computation, rule engine + ML model ensemble, graph-based fraud rings, false positive management, model retraining pipeline, PII handling |
| 27 | [Stock Exchange Matching Engine](Top%2080%20Questions/76_Stock_Exchange_Matching_Engine.md) | Fintech | ⭐⭐⭐⭐⭐ Very Hard | Order book data structure (price-time priority), matching algorithm, lock-free queues, nanosecond latency, market data dissemination, regulatory audit trail |
| 28 | [Recommendation System](Top%2080%20Questions/47_Recommendation_System.md) | ML Systems | ⭐⭐⭐⭐ Hard | Two-tower model, candidate generation (ANN search), ranking stage, feature store, cold start, exploration vs exploitation, A/B testing integration |
| 29 | [Distributed Consensus (Raft/Paxos)](Top%2080%20Questions/83_Distributed_Consensus_Raft_Paxos.md) | Infrastructure | ⭐⭐⭐⭐⭐ Very Hard | Leader election, log replication, safety guarantees, membership changes, split-brain prevention — **the theoretical backbone** interviewers expect staff engineers to articulate |
| 30 | [Load Balancer](Top%2080%20Questions/30_Load_Balancer.md) | Infrastructure | ⭐⭐⭐ Medium | L4 vs L7, consistent hashing, health checks, connection draining, sticky sessions, global server load balancing (GSLB), direct server return |

**After Batch 1 you'll have covered:** caching, queues, feeds, chat, search, geo, payments, storage, streaming, observability, event sourcing, consensus, collaborative editing, ML systems, fintech, and core infra patterns.

---

## 🟡 Batch 2 — Expand (30 Problems)

*Goal: fill category gaps, introduce specialized domains, add problems where a single technique is the make-or-break differentiator.*

| # | Problem| Category | Complexity | Key Concepts / Why It's Worth Studying |
|---|--------|----------|------------|----------------------------------------|
| 1 | [Twitter Timeline](Top%2080%20Questions/08_Twitter_Timeline.md) | Feed & Social | ⭐⭐⭐ Medium | Fan-out trade-offs from a different angle than News Feed, timeline merging, list-based feeds, retweet mechanics |
| 2 | [Instagram](Top%2080%20Questions/09_Instagram.md) | Feed & Social | ⭐⭐⭐ Medium | Photo upload pipeline, CDN + S3 storage, explore/discovery feed, story rendering, hashtag search |
| 3 | [TikTok](Top%2080%20Questions/44_TikTok.md) | Feed & Social | ⭐⭐⭐⭐ Hard | Interest graph vs social graph, engagement-loop optimization, cold-start exploration, short-video pipeline, content safety at upload |
| 4 | [Food Delivery](Top%2080%20Questions/21_Food_Delivery.md) | Marketplace | ⭐⭐⭐⭐ Hard | Three-sided marketplace (user, restaurant, driver), order dispatch, real-time tracking, kitchen prep time estimation, batching orders |
| 5 | [Flash Sale System](Top%2080%20Questions/67_Flash_Sale_System.md) | Commerce | ⭐⭐⭐⭐ Hard | Inventory pre-warming, request queuing, atomic decrement, fairness under extreme write contention, graceful degradation |
| 6 | [Distributed Stream Processing (Flink)](Top%2080%20Questions/16_Distributed_Stream_Processing.md) | Data Infra | ⭐⭐⭐⭐ Hard | Windowing (tumbling, sliding, session), watermarks & late data, exactly-once in Flink/Spark, stateful operators, checkpointing |
| 7 | [Trending Topics](Top%2080%20Questions/42_Trending_Topics.md) | Analytics | ⭐⭐⭐ Medium | Sliding window counting, exponential decay, Count-Min Sketch, heavy hitters algorithm, spam/bot filtering |
| 8 | [Top K Rankings](Top%2080%20Questions/14_Top_K_Rankings.md) | Analytics | ⭐⭐⭐⭐ Hard | Min-heap per partition, map-reduce merge, approximate algorithms, time-decay scoring, multi-level aggregation |
| 9 | [User Analytics Pipeline](Top%2080%20Questions/37_User_Analytics_Pipeline.md) | Data Infra | ⭐⭐⭐⭐ Hard | Lambda vs Kappa architecture, event schema evolution, data warehouse star schema, ETL vs ELT, data lake partitioning |
| 10 | [Live Streaming Platform (Twitch)](Top%2080%20Questions/60_Live_Streaming_Platform.md) | Media | ⭐⭐⭐⭐ Hard | Ingest → transcode → distribute pipeline, WebRTC vs RTMP, low-latency HLS, chat overlay at scale, DVR functionality |
| 11 | [Video Transcoding Pipeline](Top%2080%20Questions/59_Video_Transcoding_Pipeline.md) | Media | ⭐⭐⭐⭐ Hard | DAG-based job orchestration, GPU worker pools, codec selection, chunk-level parallelism, retry & poison pill handling |
| 12 | [Map Rendering & Navigation](Top%2080%20Questions/53_Map_Rendering_Navigation.md) | Geo | ⭐⭐⭐⭐ Hard | Tile pyramid (zoom levels), vector vs raster tiles, Dijkstra/A* with contraction hierarchies, offline maps, turn-by-turn state machine |
| 13 | [Proximity Server (Yelp)](Top%2080%20Questions/20_Proximity_Server.md) | Geo | ⭐⭐⭐ Medium | Geohash vs QuadTree vs S2, spatial indexing in Redis, radius search, ranking by distance + relevance, geo-sharding |
| 14 | [Real-time Vehicle Tracking](Top%2080%20Questions/57_Realtime_Vehicle_Tracking.md) | Geo / IoT | ⭐⭐⭐ Medium | High-frequency location ingestion, geospatial pub/sub, trajectory compression, map matching, historical replay |
| 15 | [Blob Storage (S3)](Top%2080%20Questions/86_Blob_Storage_S3.md) | Storage | ⭐⭐⭐⭐ Hard | Object metadata store, erasure coding vs replication, multipart upload, garbage collection, pre-signed URLs, consistency model (strong read-after-write) |
| 16 | [Time-Series Database](Top%2080%20Questions/85_Time_Series_Database.md) | Storage | ⭐⭐⭐⭐ Hard | Write-optimized storage (LSM/columnar), downsampling, rollup aggregation, retention policies, tag-based indexing, out-of-order write handling |
| 17 | [Video Conferencing (Zoom)](Top%2080%20Questions/82_Video_Conferencing_Zoom.md) | Real-time | ⭐⭐⭐⭐⭐ Very Hard | SFU vs MCU media servers, WebRTC data channels, bandwidth estimation (REMB/TWCC), simulcast/SVC layers, screen sharing, recording pipeline |
| 18 | [Circuit Breaker](Top%2080%20Questions/84_Circuit_Breaker.md) | Resilience | ⭐⭐⭐ Medium | State machine (closed → open → half-open), failure rate thresholds, bulkhead pattern, fallback strategies, integration with service mesh |
| 19 | [Service Discovery](Top%2080%20Questions/87_Service_Discovery.md) | Infrastructure | ⭐⭐⭐ Medium | Client-side vs server-side discovery, health check mechanisms, DNS-based vs registry (Consul/etcd), self-registration, stale entry eviction |
| 20 | [Feature Flag System](Top%2080%20Questions/88_Feature_Flag_System.md) | DevOps | ⭐⭐⭐ Medium | Flag evaluation engine, percentage rollouts, user segmentation, kill switches, flag dependency graph, audit trail |
| 21 | [A/B Testing Platform](Top%2080%20Questions/79_AB_Testing_Platform.md) | ML / Product | ⭐⭐⭐⭐ Hard | Experiment assignment (deterministic hashing), metric pipeline, statistical significance engine, interaction effects, guardrail metrics, ramp-up |
| 22 | [Content Moderation System](Top%2080%20Questions/80_Content_Moderation_System.md) | Trust & Safety | ⭐⭐⭐⭐ Hard | Multi-modal detection (text, image, video), ML classifier + human review queue, appeal flow, false positive handling, latency vs accuracy trade-off |
| 23 | [Auth System (OAuth/SSO)](Top%2080%20Questions/75_Auth_System_OAuth_SSO.md) | Security | ⭐⭐⭐⭐ Hard | OAuth 2.0 flows (authorization code, PKCE), JWT lifecycle, refresh token rotation, RBAC vs ABAC, session management, token revocation |
| 24 | [Banking Ledger System](Top%2080%20Questions/77_Banking_Ledger_System.md) | Fintech | ⭐⭐⭐⭐⭐ Very Hard | Double-entry bookkeeping, immutable append-only ledger, ACID at scale, balance snapshot optimization, regulatory audit, cross-ledger settlements |
| 25 | [Reddit Discussion Forum](Top%2080%20Questions/10_Reddit_Discussion_Forum.md) | Social | ⭐⭐⭐⭐ Hard | Nested comment trees (materialized path vs adjacency list vs closure table), hot/best/controversial ranking, subreddit isolation, vote aggregation |
| 26 | [Social Graph Store](Top%2080%20Questions/50_Social_Graph_Store.md) | Storage | ⭐⭐⭐⭐ Hard | Graph DB vs adjacency list in relational, fan-out queries (friends-of-friends), graph partitioning (edge-cut vs vertex-cut), bidirectional edges |
| 27 | [Multiplayer Game Backend](Top%2080%20Questions/90_Multiplayer_Game_Backend.md) | Real-time | ⭐⭐⭐⭐⭐ Very Hard | Game loop tick rate, client-side prediction, server reconciliation, lag compensation, spatial partitioning (interest management), UDP vs TCP |
| 28 | [Ad Click Prediction](Top%2080%20Questions/97_Ad_Click_Prediction.md) | ML Systems | ⭐⭐⭐⭐ Hard | Feature engineering at serving time, model serving latency, click-through rate prediction, bid optimization, feedback loop, online learning |
| 29 | [Realtime Bidding System (Ad Tech)](Top%2080%20Questions/89_Realtime_Bidding_System.md) | Ad Tech | ⭐⭐⭐⭐ Hard | 100ms bid deadline, bid request/response protocol (OpenRTB), budget pacing, frequency capping, auction types (first-price vs second-price) |
| 30 | [P2P File Transfer (BitTorrent)](Top%2080%20Questions/91_P2P_File_Transfer_BitTorrent.md) | Distributed | ⭐⭐⭐⭐ Hard | Piece selection (rarest first), peer discovery (DHT/tracker), tit-for-tat incentive, NAT traversal (STUN/TURN), swarm management |

**After Batch 2 you'll have added:** multi-sided marketplaces, flash sales, stream processing, advanced geo/navigation, blob storage, TSDB, video conferencing, resilience patterns, A/B testing, auth, fintech ledgers, ad tech, P2P, game backends.

---

## 🟢 Batch 3 — Good to Have (40 Problems)

*Goal: fill remaining gaps. These are either simpler variations of Batch 1/2 problems, domain-specific, or problems most senior engineers can reason through without dedicated prep.*

| # | Problem | Category | Complexity | Key Concepts / Notes |
|---|---------|----------|------------|----------------------|
| 1 | [Pastebin](Top%2080%20Questions/26_Pastebin.md) | Web Services | ⭐⭐ Easy | Simpler variant of URL Shortener — blob storage, TTL, access control. Good warm-up. |
| 2 | [Music Streaming (Spotify)](Top%2080%20Questions/18_Music_Streaming.md) | Media | ⭐⭐⭐ Medium | Audio streaming protocols, playlist service, offline sync, codec selection, CDN delivery |
| 3 | [Podcast Delivery Platform](Top%2080%20Questions/63_Podcast_Delivery_Platform.md) | Media | ⭐⭐⭐ Medium | RSS ingestion, audio processing, CDN distribution, subscription management, download tracking |
| 4 | [Thumbnail Generation Service](Top%2080%20Questions/62_Thumbnail_Generation_Service.md) | Media | ⭐⭐⭐ Medium | Async job queue, image resizing, format conversion, CDN caching, idempotent generation |
| 5 | [Image Processing Pipeline](Top%2080%20Questions/61_Image_Processing_Pipeline.md) | Media | ⭐⭐⭐ Medium | Multi-step pipeline, worker pools, retry logic, output storage, webhook callbacks |
| 6 | [Ephemeral Stories (Instagram/Snapchat)](Top%2080%20Questions/48_Ephemeral_Stories.md) | Feed & Social | ⭐⭐⭐ Medium | 24-hour TTL, viewer list tracking, story ring ordering, CDN pre-warming, soft delete |
| 7 | [User Presence System](Top%2080%20Questions/49_User_Presence_System.md) | Messaging | ⭐⭐⭐ Medium | Heartbeat mechanism, WebSocket state, presence fan-out to friends, coalescing updates |
| 8 | [Follower/Following System](Top%2080%20Questions/51_Follower_Following_System.md) | Social | ⭐⭐⭐ Medium | Denormalized counters, fan-out implications, asymmetric follows, count consistency |
| 9 | [Mentions & Tagging System](Top%2080%20Questions/52_Mentions_Tagging_System.md) | Social | ⭐⭐⭐ Medium | @-mention parsing & indexing, notification trigger, privacy checks, reverse index |
| 10 | [Live Likes & Reactions](Top%2080%20Questions/34_Live_Likes_Reactions.md) | Real-time | ⭐⭐⭐⭐ Hard | High-throughput write aggregation, buffered counters, WebSocket broadcast, animation sync |
| 11 | [Live Comments System](Top%2080%20Questions/35_Live_Comments_System.md) | Real-time | ⭐⭐⭐⭐ Hard | Ordered message delivery, comment fan-out, profanity filtering, rate limiting per user |
| 12 | [Like Count — High Profile](Top%2080%20Questions/41_Like_Count_High_Profile.md) | Analytics | ⭐⭐⭐⭐ Hard | Eventual consistency for counters, sharded counters, read-your-writes for liker, cache warming |
| 13 | [Top K Shared Articles](Top%2080%20Questions/43_Top_K_Shared_Articles.md) | Analytics | ⭐⭐⭐ Medium | Lossy counting, time-window leaderboards, map-reduce aggregation, approximation algorithms |
| 14 | [Distributed Metrics Aggregation](Top%2080%20Questions/38_Distributed_Metrics_Aggregation.md) | Observability | ⭐⭐⭐ Medium | StatsD-style push model, pre-aggregation at agents, rollup storage, percentile computation |
| 15 | [Realtime Dashboard & Metrics](Top%2080%20Questions/39_Realtime_Dashboard_Metrics.md) | Observability | ⭐⭐⭐ Medium | WebSocket push, materialized view refresh, time-range queries, downsampling for display |
| 16 | [On-Call Escalation System (PagerDuty)](Top%2080%20Questions/36_OnCall_Escalation_System.md) | DevOps | ⭐⭐⭐ Medium | Escalation policy state machine, schedule management, acknowledgment timeout, multi-channel alert |
| 17 | [Geofencing Service](Top%2080%20Questions/54_Geofencing_Service.md) | Geo | ⭐⭐⭐ Medium | Point-in-polygon tests, geofence event triggers, R-tree indexing, batch vs streaming detection |
| 18 | [Foursquare Checkins](Top%2080%20Questions/55_Foursquare_Checkins.md) | Geo / Social | ⭐⭐⭐ Medium | Location verification, venue search, check-in feed, gamification (badges), privacy controls |
| 19 | [ETA Calculation Service](Top%2080%20Questions/56_ETA_Calculation_Service.md) | Geo | ⭐⭐⭐⭐ Hard | Real-time traffic data ingestion, graph-based routing with live weights, ML-based prediction, caching route segments |
| 20 | [Bike Sharing System](Top%2080%20Questions/58_Bike_Sharing_System.md) | Marketplace | ⭐⭐⭐ Medium | Station inventory, rebalancing algorithm, trip lifecycle, pricing engine, IoT lock integration |
| 21 | [Tinder](Top%2080%20Questions/45_Tinder.md) | Matching | ⭐⭐⭐⭐ Hard | Recommendation + geo filtering, swipe queue pre-computation, mutual match detection, Elo/Glicko scoring |
| 22 | [Quora](Top%2080%20Questions/46_Quora.md) | Social / Search | ⭐⭐⭐ Medium | Question deduplication, answer ranking, topic graph, expertise scoring, knowledge base search |
| 23 | [Video Recommendation Engine](Top%2080%20Questions/64_Video_Recommendation_Engine.md) | ML Systems | ⭐⭐⭐⭐ Hard | Two-stage retrieval + ranking (variant of Batch 1 #28), user/item embeddings, real-time feature updates, diversity injection |
| 24 | [Shopping Cart System](Top%2080%20Questions/66_Shopping_Cart_System.md) | Commerce | ⭐⭐⭐ Medium | Guest vs logged-in cart merge, cart ↔ inventory sync, pricing at checkout, session management |
| 25 | [Inventory Management System](Top%2080%20Questions/65_Inventory_Management_System.md) | Commerce | ⭐⭐⭐⭐ Hard | Reserved vs available stock, warehouse distribution, eventual consistency, oversell protection |
| 26 | [Order Management System](Top%2080%20Questions/68_Order_Management_System.md) | Commerce | ⭐⭐⭐ Medium | Order state machine, fulfillment orchestration, return/refund flow, event-driven status updates |
| 27 | [Review & Rating System](Top%2080%20Questions/69_Review_Rating_System.md) | Commerce | ⭐⭐⭐ Medium | Aggregated rating computation, spam detection, verified purchase, helpful vote ranking |
| 28 | [Price Comparison Engine](Top%2080%20Questions/70_Price_Comparison_Engine.md) | Commerce | ⭐⭐⭐ Medium | Web scraping pipeline, price normalization, product matching (entity resolution), alerting |
| 29 | [Coupon & Discount Engine](Top%2080%20Questions/71_Coupon_Discount_Engine.md) | Commerce | ⭐⭐⭐ Medium | Rule engine, stacking policies, redemption limits (atomic counter), coupon code generation |
| 30 | [Surge Pricing System](Top%2080%20Questions/72_Surge_Pricing_System.md) | Marketplace | ⭐⭐⭐⭐ Hard | Real-time supply/demand ratio, dynamic multiplier, zone-level pricing, price smoothing, passenger communication |
| 31 | [Digital Wallet System](Top%2080%20Questions/73_Digital_Wallet_System.md) | Fintech | ⭐⭐⭐⭐ Hard | Balance consistency (serializable transactions), top-up/withdraw flows, P2P transfer, fraud checks, KYC integration |
| 32 | [Multi-Currency Payment](Top%2080%20Questions/78_Multi_Currency_Payment.md) | Fintech | ⭐⭐⭐ Medium | FX rate service, settlement currency, rounding rules, multi-PSP routing, cross-border compliance |
| 33 | [Email Service (Gmail)](Top%2080%20Questions/92_Email_Service_Gmail.md) | Messaging | ⭐⭐⭐⭐ Hard | SMTP relay, mailbox storage, search indexing, spam filtering (Bayesian + ML), threading, attachment storage |
| 34 | [Cryptocurrency Exchange](Top%2080%20Questions/93_Cryptocurrency_Exchange.md) | Fintech | ⭐⭐⭐⭐ Hard | Variant of stock exchange + wallet — hot/cold wallet, blockchain confirmation, order matching |
| 35 | [Leaderboard System](Top%2080%20Questions/94_Leaderboard_System.md) | Analytics | ⭐⭐⭐ Medium | Redis sorted sets, rank queries, sharded leaderboards, time-scoped boards, tie-breaking |
| 36 | [Hotel Booking System](Top%2080%20Questions/95_Hotel_Booking_System.md) | Commerce | ⭐⭐⭐ Medium | Variant of ticketing — room availability calendar, overbooking policy, cancellation, price optimization |
| 37 | [Backup & Disaster Recovery](Top%2080%20Questions/96_Backup_Disaster_Recovery.md) | Infrastructure | ⭐⭐⭐ Medium | RPO/RTO definitions, incremental backups, cross-region replication, failover orchestration, DR drills |
| 38 | [ML Feature Store](Top%2080%20Questions/98_ML_Feature_Store.md) | ML Infra | ⭐⭐⭐⭐ Hard | Online vs offline store, feature freshness, point-in-time correctness, feature serving latency, backfill |
| 39 | [Shared Calendar System (Google Calendar)](Top%2080%20Questions/99_Shared_Calendar_System.md) | Collaboration | ⭐⭐⭐ Medium | Recurring event expansion, timezone handling, free/busy lookup, invitation RSVP, CalDAV sync |
| 40 | [Search Ranking (Learning to Rank)](Top%2080%20Questions/100_Search_Ranking_Learning_to_Rank.md) | ML Systems | ⭐⭐⭐⭐ Hard | Pointwise/pairwise/listwise ranking, feature engineering for search, NDCG evaluation, online/offline metrics |

---

## 📊 Coverage Summary

| Dimension | Batch 1 (30) | + Batch 2 (60) | + Batch 3 (100) |
|-----------|:------------:|:--------------:|:---------------:|
| Core Infra (cache, queue, LB, locks) | ✅ | ✅ | ✅ |
| Feed & Social | ✅ | ✅✅ | ✅✅✅ |
| Search & Discovery | ✅ | ✅✅ | ✅✅✅ |
| Storage Systems | ✅ | ✅✅ | ✅✅ |
| Payments & Fintech | ✅ | ✅✅ | ✅✅✅ |
| Real-time & Messaging | ✅ | ✅✅ | ✅✅✅ |
| Media & Streaming | ✅ | ✅✅ | ✅✅✅ |
| Geo & Location | ✅ | ✅✅ | ✅✅✅ |
| Observability | ✅ | ✅ | ✅✅ |
| Commerce | ✅ | ✅✅ | ✅✅✅ |
| ML / Data Systems | ✅ | ✅✅ | ✅✅✅ |
| Security / Auth | — | ✅ | ✅ |
| Resilience Patterns | — | ✅ | ✅✅ |
| Collaboration / Productivity | ✅ | ✅ | ✅✅ |
| Ad Tech | — | ✅✅ | ✅✅ |
| DevOps / Platform | — | ✅ | ✅✅ |

---

## 🎯 How to Use This

1. **Start with Batch 1 in order** — each problem builds on concepts from earlier ones. If short on time, even the first 15 give you excellent coverage.

2. **For each problem, focus on:**
   - Requirements scoping (interviewers test if you ask the right clarifying questions)
   - Back-of-the-envelope math (don't skip this — staff rounds always probe it)
   - One or two **deep dives** — the "Key Concepts" column tells you exactly what the interviewer will zoom into
   - Trade-offs — never present one solution, always compare alternatives

3. **If you only have 2 weeks:** Batch 1 problems 1–15 + skim the rest of Batch 1.
4. **If you have 4 weeks:** All of Batch 1 + Batch 2 problems 1–15.
5. **If you have 6+ weeks:** Batch 1 + Batch 2 fully, cherry-pick from Batch 3.

---

| Complexity | Count | Distribution |
|------------|-------|-------------|
| ⭐⭐ Easy | 1 | 1% |
| ⭐⭐⭐ Medium | 36 | 36% |
| ⭐⭐⭐⭐ Hard | 48 | 48% |
| ⭐⭐⭐⭐⭐ Very Hard | 15 | 15% |
| **Total** | **100** | |
