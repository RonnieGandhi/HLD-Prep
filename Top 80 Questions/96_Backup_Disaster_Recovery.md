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

