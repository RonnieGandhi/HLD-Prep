# Core Concepts — System Design Fundamentals

Deep-dive topics frequently tested in HLD interview rounds. Technology/pattern-focused (not problem-specific).

---

## Files

| # | File | Topic |
|---|---|---|
| 1 | `01_Database_Replication.md` | Replication topologies, sync/async/semi-sync, replication lag, failover, split-brain, CDC, outbox pattern |
| 2 | `02_Cache_Failures_and_Resilience.md` | Caching strategies, thundering herd, stampede, cache down fallbacks, circuit breaker, invalidation, double-delete |
| 3 | `03_Redis_Internals_and_Patterns.md` | Redis data structures (String, Hash, Set, ZSet, HyperLogLog, Bloom, Streams), persistence, Cluster vs Sentinel, Lua, eviction |
| 4 | `04_SQL_Scaling_and_Sharding.md` | Read replicas, connection pooling, partitioning, sharding, cross-shard problems, Vitess/Citus/CockroachDB, query optimization |
| 5 | `05_Kafka_Event_Streaming.md` | Kafka architecture, producer/consumer guarantees, exactly-once, partitioning, consumer groups, rebalancing, failure scenarios |
| 6 | `06_Consistency_Models_and_Consensus.md` | CAP theorem, consistency spectrum, Raft/Paxos, ZooKeeper/etcd, 2PC vs Saga, clocks (Lamport/Vector/HLC/TrueTime) |
