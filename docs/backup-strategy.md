# Backup Strategy

**Status:** Draft — Open for discussion

## Overview

This document outlines backup considerations for the OSDU platform. Final decisions are pending.

## Components Requiring Backup

| Component | Data Type | Size Estimate | Criticality | Notes |
|-----------|-----------|---------------|-------------|-------|
| PostgreSQL (OSDU) | OSDU metadata, partitions | 5-20GB | High | Core platform data |
| PostgreSQL (Keycloak) | Users, realms, clients | 100MB-1GB | High | Auth configuration |
| Elasticsearch | Search indices, logs | 50-200GB | Medium | Can rebuild from source if needed |
| MinIO | Files, records, schemas | 100GB+ | High | Primary object storage |
| K8s Resources | Secrets, ConfigMaps, CRDs | Small | High | Platform configuration |
| Host Configs | Nginx, WireGuard, systemd | Small | Medium | Infrastructure config |

## Backup Architecture Options

### Option A: Local + Offsite Sync

```
┌────────────────────────────────────────────────────────────────┐
│                      OSDU Server                               │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Backup Jobs (CronJobs)                 │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │  pg_dump    │  │  ES Snapshot│  │  mc mirror  │       │  │
│  │  │  (daily)    │  │  (daily)    │  │  (daily)    │       │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │  │
│  │         │                │                │              │  │
│  │         └────────────────┼────────────────┘              │  │
│  │                          │                               │  │
│  │                          ▼                               │  │
│  │              ┌───────────────────────┐                   │  │
│  │              │  Local Staging        │                   │  │
│  │              │  /data/backups/       │                   │  │
│  │              └───────────┬───────────┘                   │  │
│  │                          │                               │  │
│  └──────────────────────────┼───────────────────────────────┘  │
│                             │                                  │
│                             │ restic / rclone                  │
│                             │ (encrypted, incremental)         │
│                             ▼                                  │
│               ┌─────────────────────────────┐                  │
│               │     Offsite Storage         │                  │
│               │  (Hetzner Box / B2 / S3)    │                  │
│               └─────────────────────────────┘                  │
└────────────────────────────────────────────────────────────────┘
```

**Pros:** Two-tier protection, fast local restore, encrypted offsite
**Cons:** More complex, uses local disk space

### Option B: Direct to Offsite

```
┌────────────────────────────────────────────────────────────────┐
│                      OSDU Server                               │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Backup Jobs                            │  │
│  │                                                          │  │
│  │  pg_dump ──┐                                             │  │
│  │            │                                             │  │
│  │  ES API ───┼───▶ Direct upload to S3/B2                  │  │
│  │            │                                             │  │
│  │  mc mirror─┘                                             │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                             │                                  │
│                             ▼                                  │
│               ┌─────────────────────────────────────────────┐  │
│               │           Offsite Storage                   │  │
│               └─────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

**Pros:** Simpler, no local storage needed
**Cons:** Slower restore, network dependent

### Option C: Velero (Kubernetes-Native)

```
┌────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                        │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      Velero                              │  │
│  │                                                          │  │
│  │  - Backs up K8s resources (YAML)                         │  │
│  │  - Snapshots PVCs                                        │  │
│  │  - Scheduled backups                                     │  │
│  │  - Disaster recovery                                     │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                             │                                  │
│                             ▼                                  │
│               ┌─────────────────────────────────────────────┐  │
│               │        S3-Compatible Storage                │  │
│               └─────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

**Pros:** K8s-native, handles PVCs, easy restore
**Cons:** Doesn't handle application-consistent DB backups well

## Backup Tools Under Consideration

### Database Backup

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| `pg_dump` | PostgreSQL logical backup | Simple, portable | Slower for large DBs |
| `pg_basebackup` | PostgreSQL physical backup | Fast, PITR capable | Less portable |
| `pgBackRest` | PostgreSQL backup management | Full/incremental, parallel | More complex setup |

### Elasticsearch Backup

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| Snapshot API | Native ES snapshots | Built-in, incremental | Requires shared storage |
| Curator | Snapshot management | Automated retention | Additional tool |
| elasticdump | Index export | Simple, portable | Slower, not incremental |

### Object Storage (MinIO)

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| `mc mirror` | MinIO client sync | Native, simple | Full sync each time |
| `rclone` | Multi-cloud sync | Flexible, many backends | External tool |
| MinIO replication | Built-in replication | Real-time | Needs second MinIO |

### Offsite Sync

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| `restic` | Encrypted backup | Dedup, encrypted, incremental | Learning curve |
| `rclone` | Cloud sync | Simple, many backends | No deduplication |
| `borgbackup` | Dedup backup | Efficient, mature | SFTP only (no S3) |

### Kubernetes Resources

| Tool | Use Case | Pros | Cons |
|------|----------|------|------|
| `velero` | Full cluster backup | K8s-native, PVC snapshots | Complex |
| `kubectl` export | Resource export | Simple | Manual, no PVCs |
| GitOps (ArgoCD) | Config as code | Declarative, versioned | Doesn't backup data |

## Offsite Storage Options

| Provider | Type | Cost | Pros | Cons |
|----------|------|------|------|------|
| Hetzner Storage Box | SFTP/SMB/S3 | €3.50/mo (100GB) | Same DC, fast, cheap | Same provider as server |
| Backblaze B2 | S3-compatible | ~$5/TB/mo | Very cheap, truly offsite | Egress costs |
| Wasabi | S3-compatible | $7/TB/mo | No egress fees | Higher base cost |
| AWS S3 | S3 | ~$23/TB/mo | Reliable, many regions | Expensive |
| Another Hetzner server | Full VM | €4+/mo | Full control | Overkill for backups |

## Recovery Considerations

### Recovery Point Objective (RPO)

*How much data loss is acceptable?*

| RPO | Backup Frequency | Use Case |
|-----|------------------|----------|
| 24 hours | Daily | Dev/learning environments |
| 1 hour | Hourly | Production |
| Minutes | Continuous/streaming | Critical production |

### Recovery Time Objective (RTO)

*How long to restore service?*

| RTO | Strategy | Complexity |
|-----|----------|------------|
| Days | Manual restore from backup | Simple |
| Hours | Scripted restore | Medium |
| Minutes | Hot standby, automated failover | Complex |

## Proposed Backup Schedule (Draft)

```
┌─────────────────────────────────────────────────────────────┐
│                    Daily Backup Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  02:00  PostgreSQL dump (OSDU + Keycloak)                   │
│           └─→ /data/backups/postgres/                       │
│                                                             │
│  02:30  Elasticsearch snapshot                              │
│           └─→ /data/backups/elasticsearch/                  │
│                                                             │
│  03:00  MinIO sync (local staging)                          │
│           └─→ /data/backups/minio/                          │
│                                                             │
│  04:00  K8s secrets/configs export                          │
│           └─→ /data/backups/kubernetes/                     │
│                                                             │
│  05:00  Offsite sync (restic/rclone)                        │
│           └─→ Remote storage                                │
│                                                             │
│  05:30  Cleanup old local backups                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Retention Policy Options

### Local Retention

| Option | Duration | Storage Impact |
|--------|----------|----------------|
| Minimal | 3 days | ~15GB |
| Standard | 7 days | ~35GB |
| Extended | 14 days | ~70GB |

### Offsite Retention (with deduplication)

| Option | Policy | Estimated Storage |
|--------|--------|-------------------|
| Minimal | 7 daily | ~50GB |
| Standard | 30 daily, 4 weekly | ~75GB |
| Extended | 30 daily, 12 weekly, 12 monthly | ~100GB |

## Estimated Storage Requirements

| Component | Daily Backup Size | Monthly (deduplicated) |
|-----------|-------------------|------------------------|
| PostgreSQL | ~1GB | ~5GB |
| Elasticsearch | ~5GB | ~20GB |
| MinIO | Varies | ~50-100GB |
| K8s exports | ~10MB | ~50MB |
| **Total** | ~6-10GB | **~75-125GB** |

## Monitoring & Alerting Considerations

- Backup job success/failure notifications
- Storage space monitoring
- Backup age verification (stale backup detection)
- Periodic restore testing

## Open Questions

### 1. Recovery Targets
- What RPO is acceptable for a learning environment?
- What RTO is acceptable?
- Is 24-hour data loss tolerable?

### 2. Offsite Storage
- Hetzner Storage Box (same provider, fast) vs Backblaze B2 (truly offsite, cheaper)?
- Is geographic separation important?
- Budget constraints?

### 3. Automation Level
- Fully automated with monitoring?
- Manual/semi-automated acceptable for learning?
- Alert on failures?

### 4. Disaster Recovery Scope
- Just data restoration?
- Full server rebuild automation?
- Infrastructure as Code for full recovery?

### 5. Testing Strategy
- How often to test restores?
- Automated restore testing?
- Separate test environment for restore validation?

### 6. Encryption
- Encrypt backups at rest?
- Key management strategy?
- Where to store encryption keys?

### 7. Compliance
- Any data retention requirements?
- Audit logging for backup access?
- Geographic data residency concerns?

## Next Steps

1. Decide on RPO/RTO targets
2. Choose offsite storage provider
3. Select backup tools for each component
4. Define retention policies
5. Implement and test backup jobs
6. Set up monitoring and alerting
7. Document restore procedures
8. Schedule periodic restore tests

---

*This document will be updated once decisions are made on the open questions.*
