# System Design: Top 80 Questions

A comprehensive collection of **67 system design interview questions** with detailed solutions covering high-level architecture, capacity estimation, trade-offs, and implementation strategies.

---

## 📚 Table of Contents

1. [Overview](#overview)
2. [Problem Classification](#problem-classification)
3. [Problems by Category](#problems-by-category)
4. [Problems by Complexity](#problems-by-complexity)
5. [Key Concepts by Problem](#key-concepts-by-problem)
6. [How to Use This Guide](#how-to-use-this-guide)

---

## Overview

This is a curated selection of system design problems commonly asked in senior engineer interviews at tech companies like Google, Amazon, Meta, Microsoft, and others. Each problem includes:

- **Functional & Non-Functional Requirements**: What must the system do, and what qualities matter
- **Capacity Estimations**: Back-of-the-envelope calculations
- **High-Level Design (HLD)**: Architecture diagrams and component breakdown
- **Deep Dives**: Database choices, caching strategies, scaling techniques
- **Trade-offs**: When to use which approach and why
- **Real-world Considerations**: Failure handling, monitoring, deployment

---

## Problem Classification

Problems are grouped into **8 primary categories** based on their core domain:

### 🌐 **Web Services & APIs** (7 problems)
Short-lived requests, HTTP APIs, CRUD operations

### 🏠 **Feed & Social** (9 problems)
Real-time updates, ranking, personalization, graph-based queries

### 🎯 **Search & Discovery** (6 problems)
Indexing, ranking, inverted indexes, type-ahead

### 💰 **Commerce & Marketplace** (5 problems)
Transactions, inventory, payments, seller/buyer interactions

### 🗺️ **Location & Mapping** (4 problems)
Geospatial data, proximity queries, real-time tracking

### 📊 **Analytics & Data** (9 problems)
Stream processing, aggregations, time-series, metrics

### 🎬 **Media & Streaming** (7 problems)
Video/audio transcoding, CDNs, real-time delivery

### 🛠️ **Infrastructure & Systems** (10 problems)
Distributed systems, databases, caching, queues, load balancing

---

## Problems by Category

### 🌐 Web Services & APIs

| # | Problem | Complexity |
|---|---------|-----------|
| 1 | [URL Shortener (TinyURL)](#01-url-shortener) | **⭐⭐ Easy** |
| 2 | [API Rate Limiter](#02-api-rate-limiter) | **⭐⭐ Easy** |
| 26 | [Pastebin](#26-pastebin) | **⭐⭐⭐ Medium** |
| 24 | [Payment Gateway](#24-payment-gateway) | **⭐⭐⭐ Medium** |
| 23 | [Ticketing System](#23-ticketing-system) | **⭐⭐⭐⭐ Hard** |
| 25 | [Dropbox / Google Drive](#25-dropbox--google-drive) | **⭐⭐⭐⭐ Hard** |
| 65 | [Inventory Management System](#65-inventory-management-system) | **⭐⭐⭐⭐ Hard** |

### 🏠 Feed & Social

| # | Problem | Complexity |
|---|---------|-----------|
| 6 | [News Feed System (Facebook/Instagram)](#06-news-feed-system) | **⭐⭐⭐ Medium** |
| 8 | [Twitter Timeline](#08-twitter-timeline) | **⭐⭐⭐ Medium** |
| 9 | [Instagram](#09-instagram) | **⭐⭐⭐ Medium** |
| 34 | [Live Likes & Reactions](#34-live-likes--reactions) | **⭐⭐⭐⭐ Hard** |
| 35 | [Live Comments System](#35-live-comments-system) | **⭐⭐⭐⭐ Hard** |
| 44 | [TikTok](#44-tiktok) | **⭐⭐⭐⭐ Hard** |
| 48 | [Ephemeral Stories](#48-ephemeral-stories) | **⭐⭐⭐ Medium** |
| 49 | [User Presence System](#49-user-presence-system) | **⭐⭐⭐ Medium** |
| 51 | [Follower Following System](#51-follower-following-system) | **⭐⭐⭐ Medium** |

### 🎯 Search & Discovery

| # | Problem | Complexity |
|---|---------|-----------|
| 11 | [Typeahead / Autocomplete](#11-typeahead-autocomplete) | **⭐⭐⭐ Medium** |
| 12 | [Search Engine](#12-search-engine) | **⭐⭐⭐⭐ Hard** |
| 13 | [Web Crawler](#13-web-crawler) | **⭐⭐⭐⭐ Hard** |
| 42 | [Trending Topics](#42-trending-topics) | **⭐⭐⭐ Medium** |
| 43 | [Top K Shared Articles](#43-top-k-shared-articles) | **⭐⭐⭐⭐ Hard** |
| 14 | [Top K Rankings](#14-top-k-rankings) | **⭐⭐⭐⭐ Hard** |

### 💰 Commerce & Marketplace

| # | Problem | Complexity |
|---|---------|-----------|
| 21 | [Food Delivery](#21-food-delivery) | **⭐⭐⭐⭐ Hard** |
| 22 | [Ecommerce Platform](#22-ecommerce-platform) | **⭐⭐⭐⭐ Hard** |
| 66 | [Shopping Cart System](#66-shopping-cart-system) | **⭐⭐⭐ Medium** |
| 67 | [Flash Sale System](#67-flash-sale-system) | **⭐⭐⭐⭐ Hard** |
| 45 | [Tinder](#45-tinder) | **⭐⭐⭐⭐ Hard** |

### 🗺️ Location & Mapping

| # | Problem | Complexity |
|---|---------|-----------|
| 20 | [Proximity Server](#20-proximity-server) | **⭐⭐⭐ Medium** |
| 19 | [Ride Hailing (Uber)](#19-ride-hailing-uber) | **⭐⭐⭐⭐ Hard** |
| 53 | [Map Rendering & Navigation](#53-map-rendering--navigation) | **⭐⭐⭐⭐ Hard** |
| 54 | [Geofencing Service](#54-geofencing-service) | **⭐⭐⭐ Medium** |
| 55 | [Foursquare Checkins](#55-foursquare-checkins) | **⭐⭐⭐ Medium** |
| 56 | [ETA Calculation Service](#56-eta-calculation-service) | **⭐⭐⭐⭐ Hard** |
| 57 | [Real-time Vehicle Tracking](#57-realtime-vehicle-tracking) | **⭐⭐⭐ Medium** |
| 58 | [Bike Sharing System](#58-bike-sharing-system) | **⭐⭐⭐⭐ Hard** |

### 📊 Analytics & Data

| # | Problem | Complexity |
|---|---------|-----------|
| 3 | [Key-Value Store](#03-key-value-store) | **⭐⭐⭐ Medium** |
| 16 | [Distributed Stream Processing](#16-distributed-stream-processing) | **⭐⭐⭐⭐ Hard** |
| 32 | [Distributed Tracing](#32-distributed-tracing) | **⭐⭐⭐⭐ Hard** |
| 37 | [User Analytics Pipeline](#37-user-analytics-pipeline) | **⭐⭐⭐⭐ Hard** |
| 38 | [Distributed Metrics Aggregation](#38-distributed-metrics-aggregation) | **⭐⭐⭐ Medium** |
| 39 | [Realtime Dashboard & Metrics](#39-realtime-dashboard--metrics) | **⭐⭐⭐ Medium** |
| 40 | [Log Aggregation & Search](#40-log-aggregation--search) | **⭐⭐⭐⭐ Hard** |
| 41 | [Like Count (High Profile)](#41-like-count-high-profile) | **⭐⭐⭐⭐ Hard** |

### 🎬 Media & Streaming

| # | Problem | Complexity |
|---|---------|-----------|
| 15 | [Video Streaming Platform](#15-video-streaming-platform) | **⭐⭐⭐⭐ Hard** |
| 17 | [CDN](#17-cdn) | **⭐⭐⭐⭐ Hard** |
| 18 | [Music Streaming](#18-music-streaming) | **⭐⭐⭐ Medium** |
| 59 | [Video Transcoding Pipeline](#59-video-transcoding-pipeline) | **⭐⭐⭐⭐ Hard** |
| 60 | [Live Streaming Platform](#60-live-streaming-platform) | **⭐⭐⭐⭐ Hard** |
| 61 | [Image Processing Pipeline](#61-image-processing-pipeline) | **⭐⭐⭐⭐ Hard** |
| 62 | [Thumbnail Generation Service](#62-thumbnail-generation-service) | **⭐⭐⭐ Medium** |
| 63 | [Podcast Delivery Platform](#63-podcast-delivery-platform) | **⭐⭐⭐⭐ Hard** |
| 64 | [Video Recommendation Engine](#64-video-recommendation-engine) | **⭐⭐⭐⭐ Hard** |

### 🛠️ Infrastructure & Systems

| # | Problem | Complexity |
|---|---------|-----------|
| 4 | [Unique ID Generator](#04-unique-id-generator) | **⭐⭐ Easy** |
| 5 | [Notification System](#05-notification-system) | **⭐⭐⭐ Medium** |
| 7 | [Real-Time Chat](#07-real-time-chat) | **⭐⭐⭐⭐ Hard** |
| 10 | [Reddit Discussion Forum](#10-reddit-discussion-forum) | **⭐⭐⭐⭐ Hard** |
| 27 | [Distributed Cache](#27-distributed-cache) | **⭐⭐⭐⭐ Hard** |
| 28 | [Distributed Job Scheduler](#28-distributed-job-scheduler) | **⭐⭐⭐⭐ Hard** |
| 29 | [Distributed Lock Manager](#29-distributed-lock-manager) | **⭐⭐⭐ Medium** |
| 30 | [Load Balancer](#30-load-balancer) | **⭐⭐⭐ Medium** |
| 31 | [Distributed Queue](#31-distributed-queue) | **⭐⭐⭐⭐ Hard** |
| 33 | [Event Sourcing System](#33-event-sourcing-system) | **⭐⭐⭐⭐ Hard** |
| 36 | [On-Call Escalation System](#36-on-call-escalation-system) | **⭐⭐⭐ Medium** |
| 46 | [Quora](#46-quora) | **⭐⭐⭐⭐ Hard** |
| 47 | [Recommendation System](#47-recommendation-system) | **⭐⭐⭐⭐ Hard** |
| 50 | [Social Graph Store](#50-social-graph-store) | **⭐⭐⭐⭐ Hard** |
| 52 | [Mentions & Tagging System](#52-mentions--tagging-system) | **⭐⭐⭐ Medium** |

---

## Problems by Complexity

### ⭐⭐ Easy (Start Here!)

Great for beginners. Focus on core concepts: single-machine design, basic scaling, back-of-envelope math.

| # | Problem | Category |
|---|---------|----------|
| 1 | [URL Shortener (TinyURL)](#01-url-shortener) | Web Services |
| 2 | [API Rate Limiter](#02-api-rate-limiter) | Web Services |
| 4 | [Unique ID Generator](#04-unique-id-generator) | Infrastructure |

**What you'll learn:**
- Basic cache design (Redis)
- Database indexing & queries
- Back-of-the-envelope calculations
- Load balancing basics
- TTL-based expiration

---

### ⭐⭐⭐ Medium (Build Confidence)

Intermediate problems. Introduce distributed systems, caching, basic data structures, eventual consistency.

| # | Problem | Category | Key Concepts |
|---|---------|----------|--------------|
| 3 | [Key-Value Store](#03-key-value-store) | Analytics | In-memory storage, partitioning |
| 5 | [Notification System](#05-notification-system) | Infrastructure | Message queues, asynchronous processing |
| 6 | [News Feed System](#06-news-feed-system) | Feed & Social | Fan-out, cache invalidation |
| 8 | [Twitter Timeline](#08-twitter-timeline) | Feed & Social | Timeline ranking, eventual consistency |
| 9 | [Instagram](#09-instagram) | Feed & Social | Feed generation, photo storage |
| 11 | [Typeahead / Autocomplete](#11-typeahead-autocomplete) | Search | Trie data structures, prefix search |
| 18 | [Music Streaming](#18-music-streaming) | Media | CDN, streaming protocols |
| 20 | [Proximity Server](#20-proximity-server) | Location | Geohashing, QuadTree |
| 24 | [Payment Gateway](#24-payment-gateway) | Commerce | Transaction safety, idempotency |
| 26 | [Pastebin](#26-pastebin) | Web Services | Object storage, access control |
| 29 | [Distributed Lock Manager](#29-distributed-lock-manager) | Infrastructure | Consensus, distributed locks |
| 30 | [Load Balancer](#30-load-balancer) | Infrastructure | L4/L7 balancing, health checks |
| 36 | [On-Call Escalation System](#36-on-call-escalation-system) | Infrastructure | State machines, priority queues |
| 38 | [Distributed Metrics Aggregation](#38-distributed-metrics-aggregation) | Analytics | Time-series, aggregation |
| 39 | [Realtime Dashboard & Metrics](#39-realtime-dashboard--metrics) | Analytics | Real-time updates, WebSockets |
| 42 | [Trending Topics](#42-trending-topics) | Search | Counting, time-window bucketing |
| 48 | [Ephemeral Stories](#48-ephemeral-stories) | Feed & Social | Time-based deletion, caching |
| 49 | [User Presence System](#49-user-presence-system) | Feed & Social | Heartbeats, status updates |
| 51 | [Follower Following System](#51-follower-following-system) | Feed & Social | Graph design, denormalization |
| 52 | [Mentions & Tagging System](#52-mentions--tagging-system) | Infrastructure | Indexing, @-mention resolution |
| 54 | [Geofencing Service](#54-geofencing-service) | Location | Geo-indexes, event streaming |
| 55 | [Foursquare Checkins](#55-foursquare-checkins) | Location | Time-series, analytics |
| 57 | [Real-time Vehicle Tracking](#57-realtime-vehicle-tracking) | Location | Real-time updates, compression |
| 62 | [Thumbnail Generation Service](#62-thumbnail-generation-service) | Media | Async processing, image handling |
| 66 | [Shopping Cart System](#66-shopping-cart-system) | Commerce | Caching, consistency |

**What you'll learn:**
- Distributed caching (Redis)
- Message queues (Kafka, RabbitMQ)
- Data structure design choices
- Real-time update patterns
- Geospatial indexing

---

### ⭐⭐⭐⭐ Hard (Master Level)

Advanced problems. Heavy on trade-offs, distributed algorithms, consistency models, complex architectures.

| # | Problem | Category | Key Concepts |
|---|---------|----------|--------------|
| 7 | [Real-Time Chat](#07-real-time-chat) | Infrastructure | WebSockets, message ordering |
| 10 | [Reddit Discussion Forum](#10-reddit-discussion-forum) | Infrastructure | Nested comments, tree traversal |
| 12 | [Search Engine](#12-search-engine) | Search | Inverted indexes, ranking, distributed search |
| 13 | [Web Crawler](#13-web-crawler) | Search | BFS/DFS, politeness, robots.txt |
| 14 | [Top K Rankings](#14-top-k-rankings) | Search | Heap algorithms, sharding |
| 15 | [Video Streaming Platform](#15-video-streaming-platform) | Media | Adaptive bitrate, buffering, CDN |
| 16 | [Distributed Stream Processing](#16-distributed-stream-processing) | Analytics | Stream joins, windowing, stateful processing |
| 17 | [CDN](#17-cdn) | Media | Edge caching, geo-routing, cache coherence |
| 19 | [Ride Hailing (Uber)](#19-ride-hailing-uber) | Location | Matching, ranking, real-time updates |
| 21 | [Food Delivery](#21-food-delivery) | Commerce | Multi-party matching, delivery optimization |
| 22 | [Ecommerce Platform](#22-ecommerce-platform) | Commerce | Inventory consistency, payment integration |
| 23 | [Ticketing System](#23-ticketing-system) | Web Services | Race conditions, database transactions |
| 25 | [Dropbox / Google Drive](#25-dropbox--google-drive) | Web Services | File sync, versioning, delta encoding |
| 27 | [Distributed Cache](#27-distributed-cache) | Infrastructure | Consistent hashing, eviction, replication |
| 28 | [Distributed Job Scheduler](#28-distributed-job-scheduler) | Infrastructure | Scheduling, fault tolerance, resource allocation |
| 31 | [Distributed Queue](#31-distributed-queue) | Infrastructure | Message ordering, acknowledgment, dead-letter queues |
| 32 | [Distributed Tracing](#32-distributed-tracing) | Analytics | Span collection, distributed context, sampling |
| 33 | [Event Sourcing System](#33-event-sourcing-system) | Infrastructure | Event logs, CQRS, temporal queries |
| 34 | [Live Likes & Reactions](#34-live-likes--reactions) | Feed & Social | High-throughput counters, aggregation |
| 35 | [Live Comments System](#35-live-comments-system) | Feed & Social | Message ordering, notification storm |
| 37 | [User Analytics Pipeline](#37-user-analytics-pipeline) | Analytics | ETL, batch + real-time, data warehouse |
| 40 | [Log Aggregation & Search](#40-log-aggregation--search) | Analytics | Full-text search, distributed logging, indexing |
| 41 | [Like Count (High Profile)](#41-like-count-high-profile) | Analytics | Eventually consistent counters, race conditions |
| 43 | [Top K Shared Articles](#43-top-k-shared-articles) | Search | Leaderboards, time-decay ranking |
| 44 | [TikTok](#44-tiktok) | Feed & Social | Complex ranking, real-time engagement, graph |
| 45 | [Tinder](#45-tinder) | Commerce | Matching algorithms, recommendation, real-time |
| 46 | [Quora](#46-quora) | Infrastructure | Content discovery, ranking, user engagement |
| 47 | [Recommendation System](#47-recommendation-system) | Infrastructure | ML embeddings, candidate generation, ranking |
| 50 | [Social Graph Store](#50-social-graph-store) | Infrastructure | Graph databases, replication, query patterns |
| 53 | [Map Rendering & Navigation](#53-map-rendering--navigation) | Location | Tile generation, geo-partitioning, routing |
| 56 | [ETA Calculation Service](#56-eta-calculation-service) | Location | ML predictions, feature engineering, real-time |
| 58 | [Bike Sharing System](#58-bike-sharing-system) | Location | Rebalancing, demand forecasting, transactions |
| 59 | [Video Transcoding Pipeline](#59-video-transcoding-pipeline) | Media | Job distribution, resource pooling, monitoring |
| 60 | [Live Streaming Platform](#60-live-streaming-platform) | Media | Multi-bitrate, low latency, real-time synchronization |
| 61 | [Image Processing Pipeline](#61-image-processing-pipeline) | Media | Async job queue, resizing, format conversion |
| 63 | [Podcast Delivery Platform](#63-podcast-delivery-platform) | Media | CDN, caching, subscription, progress tracking |
| 64 | [Video Recommendation Engine](#64-video-recommendation-engine) | Media | ML pipeline, feature store, online serving |
| 65 | [Inventory Management System](#65-inventory-management-system) | Commerce | Stock consistency, reservations, distributed locks |
| 67 | [Flash Sale System](#67-flash-sale-system) | Commerce | Rate limiting, inventory depletion, fairness |

**What you'll learn:**
- Distributed consensus & transactions
- Complex caching strategies
- Event-driven architectures
- Machine learning pipelines
- Multi-region deployment
- Advanced database design (sharding, replication)
- Real-time systems at scale
- Trade-offs between consistency, availability, partition tolerance

---

## Key Concepts by Problem

### Concepts Grid

Below is a map of which problems emphasize which key concepts:

| Concept | Problems | Difficulty |
|---------|----------|-----------|
| **Caching (Redis/Memcached)** | 1, 6, 8, 9, 11, 18, 20, 27, 39, 49 | Easy - Hard |
| **Distributed Systems** | 7, 13, 16, 27, 28, 29, 31, 32, 33, 40, 50 | Medium - Hard |
| **Database Design** | 3, 6, 8, 22, 25, 46, 50 | Medium - Hard |
| **Real-Time Updates** | 7, 34, 35, 39, 44, 49, 57, 60 | Medium - Hard |
| **Search & Indexing** | 11, 12, 13, 40, 42, 43, 47 | Medium - Hard |
| **Geospatial** | 20, 53, 54, 55, 56, 57, 58 | Medium - Hard |
| **Machine Learning** | 11, 42, 44, 45, 47, 64 | Medium - Hard |
| **Message Queues** | 5, 16, 31, 37, 59, 61 | Medium - Hard |
| **Stream Processing** | 16, 37, 39, 40, 41, 42 | Hard |
| **High Availability** | 1, 15, 17, 22, 25, 27, 60 | Easy - Hard |
| **Consistency & Transactions** | 22, 23, 24, 25, 33, 65, 67 | Medium - Hard |

---

## How to Use This Guide

### 🎯 For Interview Preparation

**Week 1-2: Easy Problems**
- Start with problems 1, 2, 4
- Focus on back-of-the-envelope math
- Draw simple architectures
- Practice explaining trade-offs

**Week 3-4: Medium Problems**
- Pick 3-5 from the Medium category
- Focus on database choices
- Understand cache invalidation
- Learn real-time patterns

**Week 5-8: Hard Problems**
- Tackle 4-6 hard problems
- Deep dive into one category (e.g., Feed & Social)
- Study distributed algorithms
- Practice handling follow-up questions

### 📖 For System Design Learning

1. **Read the functional & non-functional requirements first**
   - Understand constraints before designing

2. **Do capacity estimation yourself before reading**
   - Build intuition for scale

3. **Study the HLD diagram**
   - Trace a request through the system

4. **Deep dive into critical components**
   - Database choice, cache strategy, consistency model

5. **Think through edge cases**
   - What breaks under load? How do you recover?

### 🔄 Study Patterns by Concept

Group problems by concept you want to master:

**Want to master caching?** → Study: 1, 6, 8, 9, 27
**Want to master real-time?** → Study: 7, 34, 35, 39, 44, 60
**Want to master distributed systems?** → Study: 27, 28, 31, 32, 33, 50
**Want to master search?** → Study: 11, 12, 13, 40, 42, 43

---

## Problem Quick Links

### Web Services & APIs
- [01 URL Shortener](#01-url-shortener)
- [02 API Rate Limiter](#02-api-rate-limiter)
- [24 Payment Gateway](#24-payment-gateway)
- [25 Dropbox / Google Drive](#25-dropbox--google-drive)
- [26 Pastebin](#26-pastebin)
- [23 Ticketing System](#23-ticketing-system)
- [65 Inventory Management System](#65-inventory-management-system)

### Feed & Social
- [06 News Feed System](#06-news-feed-system)
- [08 Twitter Timeline](#08-twitter-timeline)
- [09 Instagram](#09-instagram)
- [34 Live Likes & Reactions](#34-live-likes--reactions)
- [35 Live Comments System](#35-live-comments-system)
- [44 TikTok](#44-tiktok)
- [48 Ephemeral Stories](#48-ephemeral-stories)
- [49 User Presence System](#49-user-presence-system)
- [51 Follower Following System](#51-follower-following-system)

### Search & Discovery
- [11 Typeahead / Autocomplete](#11-typeahead-autocomplete)
- [12 Search Engine](#12-search-engine)
- [13 Web Crawler](#13-web-crawler)
- [14 Top K Rankings](#14-top-k-rankings)
- [42 Trending Topics](#42-trending-topics)
- [43 Top K Shared Articles](#43-top-k-shared-articles)
- [46 Quora](#46-quora)
- [47 Recommendation System](#47-recommendation-system)

### Commerce & Marketplace
- [19 Ride Hailing (Uber)](#19-ride-hailing-uber)
- [21 Food Delivery](#21-food-delivery)
- [22 Ecommerce Platform](#22-ecommerce-platform)
- [45 Tinder](#45-tinder)
- [66 Shopping Cart System](#66-shopping-cart-system)
- [67 Flash Sale System](#67-flash-sale-system)

### Location & Mapping
- [20 Proximity Server](#20-proximity-server)
- [53 Map Rendering & Navigation](#53-map-rendering--navigation)
- [54 Geofencing Service](#54-geofencing-service)
- [55 Foursquare Checkins](#55-foursquare-checkins)
- [56 ETA Calculation Service](#56-eta-calculation-service)
- [57 Real-time Vehicle Tracking](#57-realtime-vehicle-tracking)
- [58 Bike Sharing System](#58-bike-sharing-system)

### Analytics & Data
- [03 Key-Value Store](#03-key-value-store)
- [16 Distributed Stream Processing](#16-distributed-stream-processing)
- [32 Distributed Tracing](#32-distributed-tracing)
- [37 User Analytics Pipeline](#37-user-analytics-pipeline)
- [38 Distributed Metrics Aggregation](#38-distributed-metrics-aggregation)
- [39 Realtime Dashboard & Metrics](#39-realtime-dashboard--metrics)
- [40 Log Aggregation & Search](#40-log-aggregation--search)
- [41 Like Count (High Profile)](#41-like-count-high-profile)

### Media & Streaming
- [15 Video Streaming Platform](#15-video-streaming-platform)
- [17 CDN](#17-cdn)
- [18 Music Streaming](#18-music-streaming)
- [59 Video Transcoding Pipeline](#59-video-transcoding-pipeline)
- [60 Live Streaming Platform](#60-live-streaming-platform)
- [61 Image Processing Pipeline](#61-image-processing-pipeline)
- [62 Thumbnail Generation Service](#62-thumbnail-generation-service)
- [63 Podcast Delivery Platform](#63-podcast-delivery-platform)
- [64 Video Recommendation Engine](#64-video-recommendation-engine)

### Infrastructure & Systems
- [04 Unique ID Generator](#04-unique-id-generator)
- [05 Notification System](#05-notification-system)
- [07 Real-Time Chat](#07-real-time-chat)
- [10 Reddit Discussion Forum](#10-reddit-discussion-forum)
- [27 Distributed Cache](#27-distributed-cache)
- [28 Distributed Job Scheduler](#28-distributed-job-scheduler)
- [29 Distributed Lock Manager](#29-distributed-lock-manager)
- [30 Load Balancer](#30-load-balancer)
- [31 Distributed Queue](#31-distributed-queue)
- [33 Event Sourcing System](#33-event-sourcing-system)
- [36 On-Call Escalation System](#36-on-call-escalation-system)
- [50 Social Graph Store](#50-social-graph-store)
- [52 Mentions & Tagging System](#52-mentions--tagging-system)

---

## 📊 Statistics

| Metric | Value |
|--------|-------|
| **Total Problems** | 67 |
| **Easy Problems** | 3 (4%) |
| **Medium Problems** | 25 (37%) |
| **Hard Problems** | 39 (58%) |
| **Categories** | 8 |
| **Most Common Concepts** | Caching, Distributed Systems, Real-Time Updates |

---

## 🎓 Recommended Reading Order

### Option 1: By Difficulty (Beginner Friendly)
1. Start: 1, 2, 4
2. Ramp: 3, 5, 6, 11, 20
3. Master: 27, 28, 31, 32, 33

### Option 2: By Category (Deep Dive)
- **Pick a category** (e.g., Feed & Social)
- **Solve all problems in order of difficulty**
- **Compare design patterns between similar problems**

### Option 3: By Interview Target (Company-Focused)
- **Google/Meta** → Feed & Social + Search: 6, 8, 9, 11, 12, 13, 42, 43, 47
- **Amazon/Uber** → Commerce & Location: 19, 21, 22, 45, 56, 58
- **Microsoft/Apple** → Infrastructure: 27, 28, 31, 32, 33, 50
- **Bloomberg/Stripe** → Web Services & Analytics: 1, 2, 24, 37, 40, 41

---

**Happy learning! 🚀**

Each problem file contains detailed sections on functional requirements, capacity estimation, high-level design, deep dives, and trade-offs. Use this README as your navigation guide.
