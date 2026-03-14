# System Design Interview — 80 Problems with Solutions

A complete system design interview preparation guide with **80 detailed solutions**, each covering FR/NFR, capacity estimation, HLD architecture, APIs, data models, fault tolerance, and senior/staff-level deep dives.

---

## How to Read This Guide

Every question below is tagged with:

- **Difficulty**: ⭐ Easy · ⭐⭐ Medium · ⭐⭐⭐ Hard
- **Priority**: 🔴 Must-Do · 🟡 Should-Do · 🟢 Bonus
- **Core Concepts** it teaches

> **The 🔴 Must-Do set (25 problems) covers every major system design concept.** If you're short on time, doing only the 🔴 problems is sufficient. 🟡 Should-Do problems (25) reinforce concepts with different problem shapes. 🟢 Bonus problems (30) are for completeness or niche domains.

### Preparation Strategy

| Time Available | What to Cover |
|---|---|
| **1 week** | All 🔴 Must-Do (25 problems) — covers every concept |
| **2-3 weeks** | 🔴 + 🟡 Should-Do (50 problems) — strong across all categories |
| **4+ weeks** | All 80 — complete mastery, niche domain depth |

---

## Concept Coverage Map

These are the core concepts that appear in system design interviews. The 🔴 Must-Do set ensures you hit every one.

| Concept | Covered By (🔴 Must-Do) |
|---|---|
| Hashing, key generation, encoding | 01 URL Shortener, 04 Unique ID |
| Rate limiting, throttling | 02 Rate Limiter |
| Caching (Redis, CDN, cache-aside) | 01, 06, 17, 27 |
| SQL vs NoSQL, schema design | 03 Key-Value Store, 22 E-commerce |
| Message queues, async processing | 05 Notification, 31 Queue |
| Fan-out on write/read, feeds | 06 News Feed |
| WebSockets, real-time communication | 07 Chat |
| Search, inverted index, ranking | 12 Search Engine |
| Web crawling, BFS, politeness | 13 Web Crawler |
| Streaming, adaptive bitrate, CDN | 15 Video Streaming, 17 CDN |
| Stream processing, Flink/Kafka | 16 Stream Processing |
| Geospatial (H3, QuadTree, GeoHash) | 19 Uber, 20 Proximity |
| ACID transactions, payments | 24 Payment Gateway |
| File sync, chunking, delta | 25 Dropbox |
| Distributed cache, consistent hashing | 27 Distributed Cache |
| Job scheduling, cron at scale | 28 Job Scheduler |
| Distributed locks, consensus | 29 Lock Manager |
| Load balancing (L4/L7) | 30 Load Balancer |
| Exactly-once, ordering, DLQ | 31 Distributed Queue |
| Race conditions, seat booking | 23 Ticketing System |
| Counters at scale, sharding | 41 Like Count |
| ML ranking, recommendations | 47 Recommendation |
| Event sourcing, CQRS | 33 Event Sourcing |
| Fraud detection, ML pipelines | 74 Fraud Detection |

---

## All 80 Problems — Grouped by Category

### 🏗️ Core Infrastructure & Distributed Systems

These are the building blocks. Almost every other system uses these patterns.

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 01 | [URL Shortener](Top%2080%20Questions/01_URL_Shortener.md) | ⭐ | 🔴 Must-Do | Hashing, Base62, KGS, Redis cache, Cassandra, read-heavy system |
| 02 | [API Rate Limiter](Top%2080%20Questions/02_API_Rate_Limiter.md) | ⭐ | 🔴 Must-Do | Token bucket, sliding window, Redis Lua, distributed rate limiting |
| 03 | [Key-Value Store](Top%2080%20Questions/03_Key_Value_Store.md) | ⭐⭐ | 🔴 Must-Do | LSM tree, WAL, compaction, replication, consistent hashing |
| 04 | [Unique ID Generator](Top%2080%20Questions/04_Unique_ID_Generator.md) | ⭐ | 🔴 Must-Do | Snowflake, UUID, clock skew, k-sortable IDs |
| 26 | [Pastebin](Top%2080%20Questions/26_Pastebin.md) | ⭐ | 🟢 Bonus | S3 storage, key generation — simpler version of URL Shortener |
| 27 | [Distributed Cache](Top%2080%20Questions/27_Distributed_Cache.md) | ⭐⭐⭐ | 🔴 Must-Do | Consistent hashing, eviction policies, cache-aside/write-through, replication |
| 28 | [Distributed Job Scheduler](Top%2080%20Questions/28_Distributed_Job_Scheduler.md) | ⭐⭐⭐ | 🔴 Must-Do | Cron at scale, task queues, idempotency, worker pools, dead-letter |
| 29 | [Distributed Lock Manager](Top%2080%20Questions/29_Distributed_Lock_Manager.md) | ⭐⭐ | 🔴 Must-Do | Redlock, ZooKeeper, fencing tokens, consensus, TTL |
| 30 | [Load Balancer](Top%2080%20Questions/30_Load_Balancer.md) | ⭐⭐ | 🔴 Must-Do | L4/L7, round-robin, least-connections, health checks, sticky sessions |
| 31 | [Distributed Queue](Top%2080%20Questions/31_Distributed_Queue.md) | ⭐⭐⭐ | 🔴 Must-Do | Ordering guarantees, at-least-once/exactly-once, partitioning, DLQ, backpressure |
| 33 | [Event Sourcing System](Top%2080%20Questions/33_Event_Sourcing_System.md) | ⭐⭐⭐ | 🔴 Must-Do | Event log, CQRS, snapshots, temporal queries, replay |

> **After these 11 problems**, you understand: caching, hashing, queues, scheduling, locking, load balancing, event-driven architecture, and all fundamental distributed systems patterns.

---

### 📱 Social, Feed & Real-Time

The most commonly asked category. Feeds, chat, and social features appear in ~40% of interviews.

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 06 | [News Feed System](Top%2080%20Questions/06_News_Feed_System.md) | ⭐⭐ | 🔴 Must-Do | Fan-out on write vs read, feed ranking, cache invalidation, hybrid approach |
| 07 | [Real-Time Chat](Top%2080%20Questions/07_Real_Time_Chat.md) | ⭐⭐⭐ | 🔴 Must-Do | WebSockets, message ordering, delivery receipts, presence, group chat |
| 05 | [Notification System](Top%2080%20Questions/05_Notification_System.md) | ⭐⭐ | 🔴 Must-Do | Multi-channel (push/email/SMS), priority queues, templating, rate limiting |
| 08 | [Twitter Timeline](Top%2080%20Questions/08_Twitter_Timeline.md) | ⭐⭐ | 🟡 Should-Do | Same as News Feed but with trending, Count-Min Sketch — reinforces fan-out |
| 09 | [Instagram](Top%2080%20Questions/09_Instagram.md) | ⭐⭐ | 🟡 Should-Do | Photo upload pipeline, CDN, media processing, feed ranking ML |
| 10 | [Reddit Discussion Forum](Top%2080%20Questions/10_Reddit_Discussion_Forum.md) | ⭐⭐⭐ | 🟡 Should-Do | Threaded comments (tree), voting, hot/best ranking algorithms |
| 44 | [TikTok](Top%2080%20Questions/44_TikTok.md) | ⭐⭐⭐ | 🟡 Should-Do | Interest-graph feed (not social-graph), video pipeline, cold-start |
| 46 | [Quora](Top%2080%20Questions/46_Quora.md) | ⭐⭐⭐ | 🟢 Bonus | Q&A ranking, Wilson score, SEO, collaborative editing |
| 34 | [Live Likes & Reactions](Top%2080%20Questions/34_Live_Likes_Reactions.md) | ⭐⭐ | 🟡 Should-Do | High-throughput counters, WebSocket broadcast, batching |
| 35 | [Live Comments System](Top%2080%20Questions/35_Live_Comments_System.md) | ⭐⭐ | 🟢 Bonus | Ordered real-time stream, fan-out to viewers — similar to 34 |
| 41 | [Like Count for High Profile](Top%2080%20Questions/41_Like_Count_High_Profile.md) | ⭐⭐⭐ | 🔴 Must-Do | Sharded counters, Bloom filter dedup, eventual consistency, Redis vs Cassandra |
| 48 | [Ephemeral Stories](Top%2080%20Questions/48_Ephemeral_Stories.md) | ⭐⭐ | 🟢 Bonus | TTL-based deletion, story ring, view tracking — subset of Instagram |
| 49 | [User Presence System](Top%2080%20Questions/49_User_Presence_System.md) | ⭐⭐ | 🟡 Should-Do | Heartbeat, last-seen, fan-out presence to friends, Redis pub/sub |
| 50 | [Social Graph Store](Top%2080%20Questions/50_Social_Graph_Store.md) | ⭐⭐⭐ | 🟡 Should-Do | Adjacency list, graph traversal, mutual friends, TAO-style cache |
| 51 | [Follower/Following System](Top%2080%20Questions/51_Follower_Following_System.md) | ⭐⭐ | 🟢 Bonus | Celebrity hot-spot, counter sync — subset of Social Graph |
| 52 | [Mentions & Tagging](Top%2080%20Questions/52_Mentions_Tagging_System.md) | ⭐⭐ | 🟢 Bonus | @-mention parsing, autocomplete, notification fan-out |

> **After 06, 07, 05, 41** you've covered fan-out, real-time communication, notifications, and counters at scale. The rest reinforce with variations.

---

### 🔍 Search, Ranking & Discovery

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 11 | [Typeahead / Autocomplete](Top%2080%20Questions/11_Typeahead_Autocomplete.md) | ⭐⭐ | 🟡 Should-Do | Trie, prefix search, top-K per node, hot-swap serving |
| 12 | [Search Engine](Top%2080%20Questions/12_Search_Engine.md) | ⭐⭐⭐ | 🔴 Must-Do | Inverted index, BM25, PageRank, distributed index sharding, tiered serving |
| 13 | [Web Crawler](Top%2080%20Questions/13_Web_Crawler.md) | ⭐⭐⭐ | 🔴 Must-Do | BFS/DFS, URL frontier, politeness, dedup (Bloom filter), robots.txt |
| 14 | [Top K Rankings](Top%2080%20Questions/14_Top_K_Rankings.md) | ⭐⭐⭐ | 🟡 Should-Do | Min-heap, approximate counting, Redis sorted sets, time-window ranking |
| 42 | [Trending Topics](Top%2080%20Questions/42_Trending_Topics.md) | ⭐⭐ | 🟡 Should-Do | Velocity-based scoring, Count-Min Sketch, sliding window, Flink |
| 43 | [Top K Shared Articles](Top%2080%20Questions/43_Top_K_Shared_Articles.md) | ⭐⭐⭐ | 🟢 Bonus | Time-decay leaderboard — similar to 14 + 42 combined |
| 47 | [Recommendation System](Top%2080%20Questions/47_Recommendation_System.md) | ⭐⭐⭐ | 🔴 Must-Do | Collaborative filtering, embeddings, ANN (FAISS), candidate gen + ranking, feature store |

> **12, 13, 47** cover the three pillars: indexing/search, crawling/data ingestion, and ML-based ranking. 11, 14, 42 are good reinforcement.

---

### 🛒 E-Commerce, Payments & Transactions

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 22 | [E-Commerce Platform](Top%2080%20Questions/22_Ecommerce_Platform.md) | ⭐⭐⭐ | 🟡 Should-Do | Catalog, cart, checkout, search, microservice orchestration |
| 23 | [Ticketing System](Top%2080%20Questions/23_Ticketing_System.md) | ⭐⭐⭐ | 🔴 Must-Do | Seat reservation race condition, SELECT FOR UPDATE, hold-and-confirm, TTL release |
| 24 | [Payment Gateway](Top%2080%20Questions/24_Payment_Gateway.md) | ⭐⭐⭐ | 🔴 Must-Do | ACID, idempotency keys, double-entry ledger, PSP integration, reconciliation |
| 65 | [Inventory Management](Top%2080%20Questions/65_Inventory_Management_System.md) | ⭐⭐⭐ | 🟡 Should-Do | Two-phase reserve/confirm, multi-warehouse, Redis+PG sync |
| 66 | [Shopping Cart System](Top%2080%20Questions/66_Shopping_Cart_System.md) | ⭐⭐ | 🟢 Bonus | Redis Hash, guest-to-login merge, price validation — simpler subset |
| 67 | [Flash Sale System](Top%2080%20Questions/67_Flash_Sale_System.md) | ⭐⭐⭐ | 🟡 Should-Do | Redis Lua atomic scripts, virtual queue, anti-bot, extreme concurrency |
| 68 | [Order Management System](Top%2080%20Questions/68_Order_Management_System.md) | ⭐⭐⭐ | 🟡 Should-Do | SAGA pattern, state machine, compensation, split shipments |
| 69 | [Review & Rating System](Top%2080%20Questions/69_Review_Rating_System.md) | ⭐⭐ | 🟢 Bonus | Pre-computed aggregates, Bayesian average, Wilson score, fake detection |
| 70 | [Price Comparison Engine](Top%2080%20Questions/70_Price_Comparison_Engine.md) | ⭐⭐ | 🟢 Bonus | Product matching (NLP), scraper fleet, price history |
| 71 | [Coupon & Discount Engine](Top%2080%20Questions/71_Coupon_Discount_Engine.md) | ⭐⭐ | 🟢 Bonus | Rule engine, stacking, Redis atomic usage limits |

> **23, 24** are the must-dos — they teach ACID transactions, race conditions, and idempotency. 67 is great for extreme concurrency. The rest are domain-specific variations.

---

### 🗺️ Location, Geo & Ride-Hailing

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 19 | [Ride-Hailing (Uber)](Top%2080%20Questions/19_Ride_Hailing_Uber.md) | ⭐⭐⭐ | 🔴 Must-Do | Driver matching, geospatial index, ETA, dispatch, real-time tracking |
| 20 | [Proximity Server](Top%2080%20Questions/20_Proximity_Server.md) | ⭐⭐ | 🔴 Must-Do | GeoHash, QuadTree, R-tree, nearby search, boundary problem |
| 21 | [Food Delivery](Top%2080%20Questions/21_Food_Delivery.md) | ⭐⭐⭐ | 🟡 Should-Do | Multi-party (customer/restaurant/driver), SAGA, dispatch matching |
| 45 | [Tinder](Top%2080%20Questions/45_Tinder.md) | ⭐⭐⭐ | 🟡 Should-Do | Geospatial + recommendation hybrid, swipe queue, mutual match |
| 53 | [Map Rendering & Navigation](Top%2080%20Questions/53_Map_Rendering_Navigation.md) | ⭐⭐⭐ | 🟢 Bonus | Tile pyramid, Dijkstra/A*, road graph, vector tiles |
| 54 | [Geofencing Service](Top%2080%20Questions/54_Geofencing_Service.md) | ⭐⭐ | 🟢 Bonus | Point-in-polygon, R-tree, enter/exit events — subset of 20 |
| 55 | [Foursquare (Check-ins)](Top%2080%20Questions/55_Foursquare_Checkins.md) | ⭐⭐ | 🟢 Bonus | Venue discovery, mayorship, recommendation — variation of 20 |
| 56 | [ETA Calculation](Top%2080%20Questions/56_ETA_Calculation_Service.md) | ⭐⭐⭐ | 🟢 Bonus | ML-based ETA, historical traffic, graph algorithms |
| 57 | [Real-Time Vehicle Tracking](Top%2080%20Questions/57_Realtime_Vehicle_Tracking.md) | ⭐⭐ | 🟢 Bonus | High-frequency GPS ingestion, Kafka, time-series — subset of 19 |
| 58 | [Bike Sharing System](Top%2080%20Questions/58_Bike_Sharing_System.md) | ⭐⭐⭐ | 🟢 Bonus | Dock availability, rebalancing, demand forecasting |
| 72 | [Surge Pricing System](Top%2080%20Questions/72_Surge_Pricing_System.md) | ⭐⭐⭐ | 🟢 Bonus | H3 hexagons, supply/demand ratio, Flink window, smoothing |

> **19, 20** cover all geospatial fundamentals. 21 and 45 add multi-party coordination. 53-58, 72 are niche unless interviewing for a mapping/logistics company.

---

### 🎬 Media, Streaming & Content

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 15 | [Video Streaming (YouTube/Netflix)](Top%2080%20Questions/15_Video_Streaming_Platform.md) | ⭐⭐⭐ | 🔴 Must-Do | HLS/DASH, adaptive bitrate, transcoding, CDN, DRM |
| 17 | [Content Delivery Network](Top%2080%20Questions/17_CDN.md) | ⭐⭐⭐ | 🔴 Must-Do | Edge caching, PoP, cache invalidation, origin shield, Anycast |
| 18 | [Music Streaming (Spotify)](Top%2080%20Questions/18_Music_Streaming.md) | ⭐⭐ | 🟡 Should-Do | Audio codecs, offline mode, queue sync, royalty tracking |
| 25 | [Dropbox / Google Drive](Top%2080%20Questions/25_Dropbox_Google_Drive.md) | ⭐⭐⭐ | 🔴 Must-Do | Chunking, delta sync, conflict resolution, dedup, metadata DB |
| 59 | [Video Transcoding Pipeline](Top%2080%20Questions/59_Video_Transcoding_Pipeline.md) | ⭐⭐⭐ | 🟡 Should-Do | DAG orchestration, worker pools, GPU vs CPU, codec ladder |
| 60 | [Live Streaming (Twitch)](Top%2080%20Questions/60_Live_Streaming_Platform.md) | ⭐⭐⭐ | 🟡 Should-Do | RTMP ingest, LL-HLS, real-time transcoding, chat integration |
| 61 | [Image Processing Pipeline](Top%2080%20Questions/61_Image_Processing_Pipeline.md) | ⭐⭐ | 🟢 Bonus | Resize, format conversion, ML inference — simpler version of 59 |
| 62 | [Thumbnail Generation](Top%2080%20Questions/62_Thumbnail_Generation_Service.md) | ⭐⭐ | 🟢 Bonus | ML frame selection, async workers — subset of 59 |
| 63 | [Podcast Delivery Platform](Top%2080%20Questions/63_Podcast_Delivery_Platform.md) | ⭐⭐ | 🟢 Bonus | RSS ingestion, episode delivery, progress sync — simpler Spotify |
| 64 | [Video Recommendation Engine](Top%2080%20Questions/64_Video_Recommendation_Engine.md) | ⭐⭐⭐ | 🟢 Bonus | Two-tower model, FAISS, feature store — variation of 47 |

> **15, 17, 25** are must-dos. 59, 60 add depth on transcoding and live delivery. 61-64 are variations unless you're targeting a media company.

---

### 📊 Analytics, Observability & Data Pipelines

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 16 | [Distributed Stream Processing](Top%2080%20Questions/16_Distributed_Stream_Processing.md) | ⭐⭐⭐ | 🔴 Must-Do | Kafka internals, partitioning, consumer groups, exactly-once, windowing |
| 32 | [Distributed Tracing](Top%2080%20Questions/32_Distributed_Tracing.md) | ⭐⭐⭐ | 🟡 Should-Do | Span propagation, head-based vs tail-based sampling |
| 36 | [On-Call Escalation System](Top%2080%20Questions/36_OnCall_Escalation_System.md) | ⭐⭐ | 🟢 Bonus | State machine, escalation chains, schedule rotation |
| 37 | [User Analytics Pipeline](Top%2080%20Questions/37_User_Analytics_Pipeline.md) | ⭐⭐⭐ | 🟡 Should-Do | Lambda architecture, Spark batch + Flink real-time, ClickHouse |
| 38 | [Distributed Metrics Aggregation](Top%2080%20Questions/38_Distributed_Metrics_Aggregation.md) | ⭐⭐ | 🟢 Bonus | Time-series roll-up, Prometheus, pre-aggregation — overlaps with 39 |
| 39 | [Real-Time Dashboard](Top%2080%20Questions/39_Realtime_Dashboard_Metrics.md) | ⭐⭐ | 🟢 Bonus | WebSocket push, materialized views — consumer of 37/38 |
| 40 | [Log Aggregation & Search](Top%2080%20Questions/40_Log_Aggregation_Search.md) | ⭐⭐⭐ | 🟡 Should-Do | ELK stack, log shipping, index lifecycle, hot/warm/cold |
| 79 | [A/B Testing Platform](Top%2080%20Questions/79_AB_Testing_Platform.md) | ⭐⭐⭐ | 🟡 Should-Do | Deterministic hashing, experiment layers, SRM detection, sequential testing |

> **16** is the must-do — Kafka internals come up everywhere. 37, 40, 32 are strong should-dos for observability depth. 38, 39, 36 are overlapping variations.

---

### 💰 FinTech, Payments & Security

| # | Problem | Difficulty | Priority | Core Concepts Learned |
|---|---|---|---|---|
| 73 | [Digital Wallet System](Top%2080%20Questions/73_Digital_Wallet_System.md) | ⭐⭐⭐ | 🟡 Should-Do | Double-entry ledger, SELECT FOR UPDATE, idempotent transfers, KYC |
| 74 | [Fraud Detection System](Top%2080%20Questions/74_Fraud_Detection_System.md) | ⭐⭐⭐ | 🔴 Must-Do | Real-time feature store, ML ensemble, rule engine, fail-open/fail-close |
| 75 | [Auth System (OAuth 2.0 / SSO)](Top%2080%20Questions/75_Auth_System_OAuth_SSO.md) | ⭐⭐⭐ | 🟡 Should-Do | JWT, refresh token rotation, PKCE, RBAC, credential stuffing defense |
| 76 | [Stock Exchange Matching Engine](Top%2080%20Questions/76_Stock_Exchange_Matching_Engine.md) | ⭐⭐⭐ | 🟢 Bonus | Price-time priority, single-threaded matching, LMAX Disruptor, µs latency |
| 77 | [Banking Ledger System](Top%2080%20Questions/77_Banking_Ledger_System.md) | ⭐⭐⭐ | 🟢 Bonus | Double-entry, immutable append-only, reversals, reconciliation |
| 78 | [Multi-Currency Payment](Top%2080%20Questions/78_Multi_Currency_Payment.md) | ⭐⭐⭐ | 🟢 Bonus | FX rate locking, multi-currency ledger, payment routing |
| 80 | [Content Moderation System](Top%2080%20Questions/80_Content_Moderation_System.md) | ⭐⭐⭐ | 🟡 Should-Do | Multi-modal ML pipeline, policy engine, human review queue, trust & safety |

> **74** is the must-do — real-time ML scoring is a hot interview topic. 73, 75, 80 are strong should-dos. 76-78 are niche (FinTech/exchange-specific).

---

## 🔴 Must-Do List — 25 Problems That Cover Everything

If you prepare only these, you'll have covered every major concept asked in system design interviews.

| # | Problem | Category | What It Uniquely Teaches |
|---|---|---|---|
| 01 | URL Shortener | Infrastructure | Hashing, Base62, KGS, read-heavy design |
| 02 | Rate Limiter | Infrastructure | Token bucket, sliding window, Redis Lua |
| 03 | Key-Value Store | Infrastructure | LSM tree, WAL, replication, partitioning |
| 04 | Unique ID Generator | Infrastructure | Snowflake, clock skew, k-sortable |
| 05 | Notification System | Social/Real-Time | Multi-channel delivery, priority queues, templating |
| 06 | News Feed System | Social/Real-Time | Fan-out on write/read, hybrid model, feed ranking |
| 07 | Real-Time Chat | Social/Real-Time | WebSocket, message ordering, delivery receipts |
| 12 | Search Engine | Search | Inverted index, BM25, PageRank, distributed search |
| 13 | Web Crawler | Search | URL frontier, politeness, Bloom filter dedup |
| 15 | Video Streaming | Media | HLS/DASH, adaptive bitrate, DRM |
| 16 | Stream Processing | Analytics | Kafka internals, partitioning, windowing, exactly-once |
| 17 | CDN | Media | Edge caching, PoP, Anycast, cache invalidation |
| 19 | Ride-Hailing (Uber) | Geo/Location | Geospatial matching, real-time tracking, dispatch |
| 20 | Proximity Server | Geo/Location | GeoHash, QuadTree, nearby search |
| 23 | Ticketing System | E-Commerce | Race conditions, seat hold, TTL release |
| 24 | Payment Gateway | E-Commerce | ACID, idempotency, double-entry ledger |
| 25 | Dropbox / Google Drive | Media | File chunking, delta sync, conflict resolution |
| 27 | Distributed Cache | Infrastructure | Consistent hashing, eviction, replication |
| 28 | Job Scheduler | Infrastructure | Cron at scale, worker pools, idempotency |
| 29 | Lock Manager | Infrastructure | Redlock, ZooKeeper, fencing tokens |
| 30 | Load Balancer | Infrastructure | L4/L7, algorithms, health checks |
| 31 | Distributed Queue | Infrastructure | Ordering, DLQ, exactly-once, backpressure |
| 33 | Event Sourcing | Infrastructure | Event log, CQRS, snapshots, replay |
| 41 | Like Count (High Profile) | Social | Sharded counters, Bloom filter, eventual consistency |
| 47 | Recommendation System | Search/ML | Embeddings, ANN, candidate gen + ranking, feature store |
| 74 | Fraud Detection | FinTech/ML | Real-time ML scoring, feature store, rule engine |

### Concepts Fully Covered by the Must-Do Set

✅ Caching (Redis, CDN, cache-aside, write-through)
✅ Database choices (SQL vs NoSQL, when to use each)
✅ Message queues & async processing (Kafka, SQS)
✅ Real-time communication (WebSockets, SSE)
✅ Search & indexing (inverted index, trie, Elasticsearch)
✅ Geospatial (GeoHash, QuadTree, H3)
✅ Consistency models (strong, eventual, causal)
✅ Distributed coordination (locks, consensus, leader election)
✅ Stream processing (Flink, windowing, exactly-once)
✅ Rate limiting & throttling
✅ Fan-out patterns (write vs read)
✅ File storage & sync (chunking, delta)
✅ Video/audio streaming (HLS, ABR, CDN)
✅ ML ranking & recommendations
✅ ACID transactions & idempotency
✅ Race condition handling
✅ Event sourcing & CQRS
✅ Counter sharding at scale
✅ Load balancing (L4/L7)
✅ ID generation (Snowflake)

---

## 🟡 Should-Do List — 25 More for Depth

These reinforce concepts from the must-do set with different problem shapes and add a few new patterns.

| # | Problem | What It Adds Beyond Must-Do Set |
|---|---|---|
| 08 | Twitter Timeline | Trending (Count-Min Sketch), celebrity fan-out |
| 09 | Instagram | Photo processing pipeline, CDN for media, feed ranking ML |
| 10 | Reddit | Threaded comments (tree), hot/best ranking algorithms |
| 11 | Typeahead | Trie data structure, prefix top-K, hot-swap |
| 14 | Top K Rankings | Min-heap, approximate counting at scale |
| 18 | Music Streaming | Audio codecs, offline mode, royalty tracking |
| 21 | Food Delivery | Multi-party SAGA (customer/restaurant/driver), dispatch matching |
| 22 | E-Commerce | Full microservice orchestration, catalog + search + cart + checkout |
| 32 | Distributed Tracing | Span propagation, head-based vs tail-based sampling |
| 34 | Live Likes & Reactions | High-throughput WebSocket broadcast, counter batching |
| 37 | User Analytics Pipeline | Lambda architecture, batch + stream, data warehouse |
| 40 | Log Aggregation | ELK stack, index lifecycle management, hot/warm/cold |
| 42 | Trending Topics | Velocity-based scoring, sliding window aggregation |
| 44 | TikTok | Interest-graph (not social-graph) feed, content embeddings |
| 45 | Tinder | Geo + recommendation hybrid, swipe queue, mutual match race condition |
| 49 | User Presence | Heartbeat protocol, fan-out to friends, status aggregation |
| 50 | Social Graph Store | TAO-style cache, graph traversal, mutual friends |
| 59 | Video Transcoding | DAG pipeline orchestration, GPU worker pools |
| 60 | Live Streaming | RTMP→HLS, low-latency live, real-time transcoding |
| 65 | Inventory Management | Two-phase stock reservation, multi-warehouse allocation |
| 67 | Flash Sale System | Redis Lua scripts, virtual queue, extreme concurrency |
| 68 | Order Management | SAGA with compensation, state machine, carrier integration |
| 73 | Digital Wallet | Double-entry ledger, spending limits, P2P transfers |
| 75 | Auth (OAuth/SSO) | JWT, PKCE, refresh rotation, credential stuffing defense |
| 79 | A/B Testing | Deterministic hashing, experiment layers, SRM detection |
| 80 | Content Moderation | Multi-modal ML pipeline (text/image/video), policy engine |

---

## 🟢 Bonus — 30 Remaining for Completeness

These are niche, domain-specific, or overlapping with problems you've already studied. Useful for targeting specific companies or for comprehensive mastery.

| # | Problem | Why It's Bonus |
|---|---|---|
| 26 | Pastebin | Simpler version of URL Shortener |
| 35 | Live Comments | Similar to Live Likes (34) |
| 36 | On-Call Escalation | Niche (PagerDuty-style), state machine already covered |
| 38 | Metrics Aggregation | Overlaps with 37 (Analytics Pipeline) |
| 39 | Real-Time Dashboard | Consumer of 37/38, WebSocket already covered |
| 43 | Top K Shared Articles | Combines 14 (Top K) + 42 (Trending) |
| 46 | Quora | Q&A variant of Reddit (10), Wilson score |
| 48 | Ephemeral Stories | Subset of Instagram (09) with TTL |
| 51 | Follower/Following | Subset of Social Graph (50) |
| 52 | Mentions & Tagging | Autocomplete + notification fan-out, both covered |
| 53 | Map Rendering | Niche (Google Maps), tile pyramid, A* routing |
| 54 | Geofencing | Subset of Proximity Server (20) |
| 55 | Foursquare | Variation of Proximity (20) + Social |
| 56 | ETA Calculation | Niche (ML + graph), subset of Uber (19) |
| 57 | Vehicle Tracking | GPS ingestion — subset of Uber (19) |
| 58 | Bike Sharing | Niche (rebalancing, dock availability) |
| 61 | Image Processing | Simpler version of Video Transcoding (59) |
| 62 | Thumbnail Generation | Subset of Image Processing (61) |
| 63 | Podcast Delivery | Simpler version of Music Streaming (18) |
| 64 | Video Recommendation | Variation of Recommendation System (47) |
| 66 | Shopping Cart | Simpler subset of E-Commerce (22) |
| 69 | Review & Rating | Bayesian average, fake detection — standalone |
| 70 | Price Comparison | NLP product matching, scraper architecture |
| 71 | Coupon Engine | Rule engine, stacking — domain-specific |
| 72 | Surge Pricing | H3 hexagons, supply/demand — niche (Uber-specific) |
| 76 | Stock Exchange | Ultra-low latency, LMAX Disruptor — niche (FinTech) |
| 77 | Banking Ledger | Double-entry, reconciliation — niche (FinTech) |
| 78 | Multi-Currency Payment | FX locking, payment routing — niche (FinTech) |

---

## Study Paths by Company Type

| Company Type | Focus On | Problems |
|---|---|---|
| **FAANG / Big Tech** | Core infra + social + search | 🔴 All Must-Do + 08, 09, 10, 44, 50 |
| **Fintech (Stripe, Square)** | Payments + transactions + fraud | 23, 24, 67, 73, 74, 76, 77, 78 |
| **Ride-Hailing / Logistics** | Geo + real-time + matching | 19, 20, 21, 45, 53, 56, 57, 72 |
| **Streaming / Media** | Video + CDN + transcoding | 15, 17, 18, 25, 59, 60, 64 |
| **E-Commerce** | Cart + inventory + orders | 22, 23, 24, 65, 66, 67, 68, 69, 71 |
| **Observability / DevTools** | Logs + metrics + tracing | 16, 32, 37, 38, 39, 40, 79 |
| **Social / Consumer** | Feed + chat + presence + graph | 06, 07, 08, 09, 34, 41, 44, 49, 50, 80 |

---

## 📚 Reference Books

Available in the `Books/` folder:
- System Design Interview – Alex Xu Vol 1 & Vol 2
- ByteByteGo Big Archive 2023 & 2024
- Grok System Design Interview

---

*80 problems · Structured solutions · Staff-engineer depth · Good luck! 🚀*
