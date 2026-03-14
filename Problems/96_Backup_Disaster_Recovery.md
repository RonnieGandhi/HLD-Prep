# 96. Design a Backup and Disaster Recovery System

---

## 1. Functional Requirements (FR)

- Full backups: complete copy of all data (databases, object storage, config)
- Incremental backups: only changes since last backup (space-efficient)
- Point-in-time recovery (PITR): restore to any second within retention window
- Cross-region replication: backups stored in geographically separate region
- Backup scheduling: configurable policies (hourly incremental, daily full, weekly archive)
- Restore testing: automated periodic restore verification (backup is useless if restore fails)
- Multi-tier storage: recent backups on fast storage, older on cold/archive (S3 Glacier)
- Backup encryption at rest (AES-256) and in transit (TLS)
- RPO (Recovery Point Objective) and RTO (Recovery Time Objective) enforcement
- Disaster recovery runbook: automated failover to DR region

---

## 2. Non-Functional Requirements (NFRs)

- **RPO**: < 1 minute for critical data (max data loss on failure)
- **RTO**: < 15 minutes for critical services (max time to recover)
- **Durability**: 11 nines for backup data (same as S3)
- **Consistency**: Backup must represent a consistent point-in-time snapshot
- **Encryption**: All backups encrypted; key management via KMS
- **Compliance**: Retention policies per regulation (GDPR: right to delete from backups)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Total production data | 500 TB |
| Daily change rate | 5% → 25 TB/day incremental |
| Full backup size | 500 TB (compressed: ~200 TB) |
| Full backup frequency | Weekly |
| Incremental backup frequency | Hourly |
| Retention | 30 days hot, 1 year warm, 7 years cold |
| Total backup storage | ~5 PB (with retention + dedup) |
| Restore throughput needed (RTO=15min) | 500 TB / 15 min = 556 GB/sec |

---

## 4. High-Level Design (HLD)

```
┌─────────────────────────────────────────────────────────────────┐
│                   PRIMARY REGION (us-east-1)                     │
│                                                                   │
│  Production Systems:                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │PostgreSQL│ │Cassandra │ │Redis     │ │S3 (Blobs)│            │
│  │Cluster   │ │Cluster   │ │Cluster   │ │          │            │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘            │
│       │            │            │            │                    │
│  ┌────▼────────────▼────────────▼────────────▼──────────────┐    │
│  │  Backup Agent (per data source)                           │    │
│  │                                                            │    │
│  │  PostgreSQL: pg_basebackup (full) + WAL archiving (PITR) │    │
│  │  Cassandra: sstable snapshot + incremental                │    │
│  │  Redis: RDB snapshot + AOF streaming                      │    │
│  │  S3: Cross-region replication (built-in)                  │    │
│  │                                                            │    │
│  │  Compression: LZ4 (fast) for hot, ZSTD (high ratio) cold │    │
│  │  Deduplication: block-level dedup (same block → store once)│   │
│  │  Encryption: AES-256-GCM with per-backup key from KMS    │    │
│  └──────────────────────┬────────────────────────────────────┘    │
│                         │                                          │
│  ┌──────────────────────▼────────────────────────────────────┐    │
│  │  Backup Storage (Primary Region)                           │    │
│  │  Hot tier (S3 Standard): last 7 days                      │    │
│  │  Warm tier (S3 IA): 7-30 days                             │    │
│  │  Lifecycle rule auto-transitions                           │    │
│  └──────────────────────┬────────────────────────────────────┘    │
│                         │ Cross-region replication                  │
└─────────────────────────│──────────────────────────────────────────┘
                          │
┌─────────────────────────▼──────────────────────────────────────────┐
│                   DR REGION (us-west-2)                              │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  Backup Storage (DR Region)                                │      │
│  │  Mirror of primary region backups                         │      │
│  │  Cold tier (S3 Glacier): 30 days - 7 years               │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  DR Standby (Warm Standby or Pilot Light)                  │      │
│  │                                                            │      │
│  │  Pilot Light: minimal infra (DB replicas, no app servers)  │      │
│  │  Warm Standby: reduced-capacity app + DB (handles 10%)    │      │
│  │  Hot Standby: full capacity, active-active                │      │
│  │                                                            │      │
│  │  On failover:                                              │      │
│  │  1. DNS failover (Route 53 health check triggers)         │      │
│  │  2. Scale up DR compute (auto-scaling)                    │      │
│  │  3. Promote DB replica to primary                         │      │
│  │  4. Verify data consistency                               │      │
│  │  5. Route traffic to DR region                            │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  BACKUP CONTROL PLANE                                                 │
│                                                                       │
│  ┌────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │ Scheduler       │  │ Monitoring       │  │ Restore Testing     │  │
│  │                 │  │                  │  │                      │  │
│  │ - Cron-based    │  │ - Backup success │  │ - Weekly: restore   │  │
│  │   backup jobs   │  │   /failure alerts│  │   random backup to  │  │
│  │ - Hourly incr   │  │ - Backup size    │  │   isolated env      │  │
│  │ - Daily full    │  │   trend (detect  │  │ - Run sanity checks │  │
│  │ - Weekly archive│  │   corruption)    │  │   (row counts,      │  │
│  │ - Retention     │  │ - RPO compliance │  │   checksums)        │  │
│  │   enforcement   │  │   (was backup    │  │ - Measure actual    │  │
│  │                 │  │   taken on time?)│  │   RTO (time to      │  │
│  │                 │  │ - Storage costs  │  │   restore)          │  │
│  └─────────────────┘  └──────────────────┘  └──────────────────────┘  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### RPO vs RTO

```
RPO (Recovery Point Objective): How much data can you afford to LOSE?
  RPO = 1 hour → you backup every hour → lose at most 1 hour of data
  RPO = 0 → synchronous replication (zero data loss, highest cost)

RTO (Recovery Time Objective): How quickly must you be back ONLINE?
  RTO = 4 hours → can restore from backups (slow but cheap)
  RTO = 15 min → need warm/hot standby (fast but expensive)
  RTO ≈ 0 → active-active multi-region (fastest, most expensive)

Cost/Complexity:
  RPO↓ + RTO↓ = $$$$ (synchronous replication + hot standby)
  RPO↑ + RTO↑ = $     (daily backups to cold storage)
```

### DR Strategy Tiers

```
| Strategy | RPO | RTO | Cost | Description |
|---|---|---|---|---|
| Backup & Restore | Hours | Hours | $ | Restore from S3 on demand |
| Pilot Light | Minutes | 30 min | $$ | Minimal infra, scale on failover |
| Warm Standby | Seconds | 15 min | $$$ | Reduced capacity, always running |
| Active-Active | 0 | ~0 | $$$$ | Both regions serve traffic |
```

---

## 5. APIs

```
POST /api/backups/trigger        → Trigger manual backup {source, type}
GET  /api/backups                → List backups with status
GET  /api/backups/{id}/status    → Backup job status
POST /api/restore                → Initiate restore {backup_id, target}
POST /api/dr/failover            → Initiate DR failover (manual trigger)
GET  /api/dr/status              → DR region health, replication lag
```

---

## 6. Data Models

### PostgreSQL (Backup Metadata — Control Plane)

```sql
CREATE TABLE backup_jobs (
    backup_id      UUID PRIMARY KEY,
    source_system  TEXT NOT NULL,     -- 'postgresql-primary', 'cassandra-cluster', 'redis-cluster'
    backup_type    TEXT NOT NULL,     -- 'full', 'incremental', 'wal_archive', 'snapshot'
    status         TEXT DEFAULT 'running',  -- running|completed|failed|validating
    storage_path   TEXT,              -- s3://backups/pg/2026-03-14/full.tar.gz
    size_bytes     BIGINT,
    checksum       TEXT,              -- SHA-256 of backup data
    started_at     TIMESTAMPTZ DEFAULT NOW(),
    completed_at   TIMESTAMPTZ,
    retention_tier TEXT DEFAULT 'hot', -- hot|warm|cold|archive
    expires_at     TIMESTAMPTZ,
    encrypted      BOOLEAN DEFAULT TRUE,
    kms_key_id     TEXT               -- KMS key used for encryption
);

CREATE TABLE restore_jobs (
    restore_id     UUID PRIMARY KEY,
    backup_id      UUID REFERENCES backup_jobs(backup_id),
    target_system  TEXT NOT NULL,
    restore_point  TIMESTAMPTZ,       -- for PITR
    status         TEXT DEFAULT 'running',
    started_at     TIMESTAMPTZ DEFAULT NOW(),
    completed_at   TIMESTAMPTZ,
    validated      BOOLEAN DEFAULT FALSE
);

CREATE TABLE dr_regions (
    region_id       TEXT PRIMARY KEY,
    role            TEXT NOT NULL,     -- primary|dr
    replication_lag INTERVAL,
    last_heartbeat  TIMESTAMPTZ,
    status          TEXT DEFAULT 'healthy'
);
```

### S3 Backup Layout

```
s3://backups/
├── postgresql/
│   ├── 2026-03-14/
│   │   ├── full/base.tar.gz.enc           (weekly full backup)
│   │   ├── wal/000000010000001A000000FF   (continuous WAL archive)
│   │   └── manifest.json                  (backup metadata + checksum)
│   └── lifecycle.json                     (hot→warm→cold transitions)
├── cassandra/
│   ├── 2026-03-14/
│   │   ├── keyspace_orders/sstable-*.gz
│   │   └── incremental/                   (since last snapshot)
├── redis/
│   └── 2026-03-14/rdb-snapshot.rdb.enc
└── config/
    └── 2026-03-14/etcd-snapshot.db        (service configs, feature flags)
```

---

## 7. Key Design Decisions

### PITR (Point-in-Time Recovery) for PostgreSQL
```
1. Base backup: full copy of data directory (pg_basebackup)
2. WAL archiving: continuously ship WAL segments to S3
3. Restore to time T:
   a. Restore latest base backup before T
   b. Replay WAL segments up to time T
   c. Result: exact database state at time T (second-level precision)
```

### Backup Consistency
```
Problem: Backing up a running database → inconsistent snapshot

Solutions:
- Database-native: pg_dump --serializable (consistent snapshot)
- Filesystem: LVM snapshot → backup from frozen snapshot
- Cloud: EBS snapshot (crash-consistent, fine for most DBs with WAL)
- Application-level: quiesce writes → snapshot → resume (brief downtime)
```

---

## 8. Fault Tolerance & Deep Dives

### Automated Failover Runbook

```
Trigger: Primary region health check fails for > 60 seconds

Automated sequence (one-click or auto-trigger):
  T=0s:   Health check failure detected (Route53 health checker)
  T=5s:   Alert PagerDuty + Slack channel
  T=10s:  Fence old primary (revoke DNS, block writes via security group)
  T=15s:  Promote DR PostgreSQL replica to primary
          → pg_promote() on standby, < 5 seconds
  T=20s:  Update service discovery (Consul/etcd) → DR endpoints
  T=25s:  Scale DR auto-scaling groups to full capacity
  T=30s:  DNS failover: Route53 switches to DR region
  T=60s:  Traffic flowing to DR region
  T=120s: Run smoke tests (health endpoints, sample queries)
  T=180s: Declare DR active; page on-call to confirm

Total RTO: ~3 minutes (automated) vs 30+ minutes (manual)

CRITICAL: Fence old primary BEFORE promoting DR
  If old primary comes back online → split-brain → data corruption
  Fencing: revoke IAM roles, block network, shut down DB processes
```

### Restore Testing (The Most Overlooked Step)

```
"A backup that has never been tested is not a backup"

Weekly automated restore test:
1. Pick random recent backup
2. Restore to isolated environment (separate VPC)
3. Run validation checks:
   a. Row counts match expected (within 0.1%)
   b. Checksums of critical tables match
   c. Application can connect and query
   d. Measure actual restore time (compare to RTO target)
4. Report results → dashboard + alert on failure
5. Tear down isolated environment

If restore test fails → P1 alert to on-call backup engineer
```

### Immutable Backups (Ransomware Protection)

```
Problem: Ransomware encrypts production data AND deletes backups

Solution: S3 Object Lock (WORM — Write Once Read Many)
  aws s3api put-object-lock-configuration --bucket backups \
    --object-lock-configuration '{"ObjectLockEnabled":"Enabled",
    "Rule":{"DefaultRetention":{"Mode":"COMPLIANCE","Days":30}}}'

  COMPLIANCE mode: NOBODY can delete (not even root/admin) for 30 days
  GOVERNANCE mode: Admin with special permission can override

  Combined with:
  - Cross-account replication: backups in separate AWS account
  - MFA delete: require MFA token to delete any backup
  - Separate KMS keys: backup encryption keys in different account
```

### Crypto-Shredding (GDPR Right to Erasure from Backups)

```
Problem: User requests data deletion, but their data exists in 90 days of backups

Traditional: Restore each backup, delete user data, re-backup → IMPRACTICAL

Crypto-shredding:
  1. Each user's data encrypted with per-user key before backup
  2. User deletion request → delete their encryption key from KMS
  3. Backup data still exists but is unreadable (key is gone)
  4. Effectively deleted without modifying backup files
  5. Immutable backups + GDPR compliance = both satisfied
```

---

## 9. Additional Considerations

- **Cost optimization**: Dedup + compression + lifecycle policies reduce storage 5-10×. S3 Intelligent-Tiering auto-moves to cheapest tier
- **Chaos engineering**: Regularly test DR failover (GameDay exercises) — Netflix, Google do this quarterly
- **Multi-database consistency**: If backup PG at T=100 and Cassandra at T=105 → inconsistent. Solution: coordinated snapshot across systems (or accept small inconsistency window)
- **Backup bandwidth**: 500TB backup over network = days. Use incremental (25TB/day) + local snapshot + async replication
- **Monitoring**: Track backup success rate, backup size trends (sudden growth = anomaly), replication lag, last successful restore test date

---

## 10. Deep Dive: Engineering Trade-offs

### RPO vs RTO: The Fundamental Trade-off

```
RPO = how much data you can afford to lose
RTO = how long you can be down

RPO 0 (zero data loss):
  Requires: synchronous replication to DR site
  Sync replication: primary WAITS for DR to confirm write before ACK to client
  Latency cost: +20-100ms per write (cross-region network RTT)
  ✗ 2-5× slower writes than async
  ✓ Zero data loss guaranteed
  Use for: Financial transactions, ledger systems, payment data

RPO 1 minute:
  Requires: async replication + WAL shipping every minute
  Primary writes freely → ships WAL logs to DR every 60 seconds
  On failover: lose up to 60 seconds of committed transactions
  ✓ Minimal write performance impact
  ✗ 1 minute of data loss on catastrophic failure
  Use for: E-commerce orders, user data, content

RPO 1 hour:
  Requires: hourly backups to S3 (snapshot + incremental)
  Cheapest approach — no replication, just scheduled backups
  On failover: restore from last backup → 1 hour of data loss
  ✓ Simple, cheap
  ✗ Significant data loss + slow restore (RTO = hours)
  Use for: Analytics, non-critical data, development environments

RTO → cost relationship:
  RTO = 4 hours: weekly full + daily incremental. Restore from backup.
    Cost: ~$500/month (just S3 storage)
  RTO = 15 minutes: hot standby + automated failover
    Cost: ~$10,000/month (duplicate infrastructure, always running)
  RTO = 30 seconds: active-active multi-region
    Cost: ~$50,000/month (full infrastructure in 2+ regions)
  
  Each 10× improvement in RTO → ~5-10× cost increase
```

### Split-Brain Prevention During Failover

```
The most dangerous failure mode in DR:

  Both primary and DR think they are the active writer
  → Two databases accepting writes independently
  → Data diverges → reconciliation is extremely hard (or impossible)
  → Financial data corruption → regulatory and legal exposure

How split-brain happens:
  Primary: us-east-1 (healthy, accepting writes)
  DR:      us-west-2 (standby, read-only)
  
  Monitoring detects: "primary appears down" (false alarm — network issue)
  Automated failover promotes DR to primary
  Old primary is actually fine → now BOTH accept writes → SPLIT BRAIN
  
Prevention strategies:

  1. Fencing (STONITH — Shoot The Other Node In The Head):
     Before promoting DR:
       a. Revoke primary's IAM write permissions
       b. Modify security group: block all client traffic to primary
       c. Shut down primary DB process via out-of-band management (IPMI/ILO)
     Only AFTER fencing succeeds → promote DR
     If fencing fails → DO NOT promote (manual intervention required)
  
  2. Witness/Quorum:
     Deploy a "witness" node in a third region (e.g., us-central-1)
     Failover decision requires agreement from 2-of-3: primary, DR, witness
     If primary is truly down: DR + witness agree → promote
     If network partition: primary + witness agree → primary stays active
     Prevents: monitoring false alarms from triggering split-brain
  
  3. Lease-based leadership:
     Primary holds a "leadership lease" in a distributed lock (DynamoDB/etcd)
     Lease expires every 30 seconds → primary must renew
     If primary can't renew (network partition) → lease expires → safe to promote
     Primary detects lease loss → SELF-FENCES (stops accepting writes)
     
     This is how AWS RDS multi-AZ failover works internally

  4. Epoch-based writes:
     Every write includes an epoch number
     On failover: new primary increments epoch (42 → 43)
     Old primary still at epoch 42
     Storage layer rejects writes with epoch < current (42 < 43 → rejected)
     Even if old primary briefly accepts writes → they're discarded downstream
```

### Failback: Returning to Primary After DR

```
DR failover succeeded. DR is now the active primary.
Original primary region is repaired. How to fail BACK?

Step 1: Rebuild primary as a replica (NOT as primary)
  Point repaired primary at DR as its replication source
  Full base backup from DR → restore on primary
  Begin streaming replication: DR → primary
  Wait for replication lag to reach 0

Step 2: Validate primary replica
  Compare row counts, checksums, query results
  Run application smoke tests against primary replica (read-only)
  Confirm: primary has ALL data that DR accumulated during outage

Step 3: Planned failover (during maintenance window)
  Announce maintenance window (e.g., 2 minutes)
  Stop writes to DR (brief downtime)
  Wait for primary to catch up (last few transactions)
  Promote primary back to leader
  Point all traffic to primary
  DR becomes standby again

Step 4: Post-failback monitoring
  Watch for: replication lag, error rates, latency
  Keep DR hot for 48 hours (quick re-failover if primary has issues)

Duration: Step 1 takes hours (full sync), Steps 2-4 take minutes
Total failback time: 2-12 hours (depending on data volume)

Danger: If you fail back too quickly without full sync:
  Primary missed transactions that happened on DR during the outage
  Those transactions are LOST when traffic switches back to primary
  → Always verify full sync before failback
```

### Active-Active Multi-Region: The Holy Grail (and Its Costs)

```
Instead of active-passive, run BOTH regions as active simultaneously:
  Users in US → us-east-1 (reads AND writes)
  Users in EU → eu-west-1 (reads AND writes)
  
  Both regions replicate to each other (bidirectional)
  If either region fails → other handles ALL traffic

Why it's hard:
  
  Conflict resolution:
    User updates profile in us-east at T=100, and in eu-west at T=101
    Both regions accepted the write → which one wins?
    
    Strategies:
      Last-write-wins (LWW): higher timestamp wins → simple but lossy
      Application-level merge: merge non-conflicting fields → complex
      CRDTs: data structures that auto-merge → limited to specific types
      Region-owned data: user's primary region handles writes → no conflicts
  
  Referential integrity:
    Order created in us-east references user created in eu-west
    Replication lag: user hasn't replicated to us-east yet → foreign key violation
    
    Solution: use global IDs (no foreign keys), eventual consistency accepted
    OR: route user + all related entities to same region (partition by user)

  When active-active makes sense:
    ✓ Read-heavy workloads (global CDN-like caching, no write conflicts)
    ✓ Partition-able data (each user assigned to home region)
    ✓ CRDT-compatible operations (counters, sets, append-only logs)
    
  When active-passive is better:
    ✓ Strong consistency required (financial, inventory)
    ✓ Complex transactions spanning multiple entities
    ✓ Write-heavy workloads (replication lag causes conflicts)
    ✓ Simpler operations (most organizations)
```

