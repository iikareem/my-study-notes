---
tags:
  - aws
  - certification
  - postgresql
  - practice
  - rds
  - replication
  - scaling
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[RDS Aurora]] · [[RDS Aurora Exam Guide]]

# RDS & Aurora — Exam Cheatsheet

> Only what matters most for the exam. Bold = high-confidence exam topic.

---

## RDS — Core

- **Managed service** — AWS handles OS patching, backups, hardware. You cannot SSH in.
- **Multi-AZ** — Synchronous replication to standby in another AZ. Standby is **NOT readable**. ~60–120s failover. For **disaster recovery only**.
- **Read Replicas** — Async replication. Up to **5**. Can be cross-region. For **read scaling**. May have lag.
- **Automated backups** — Retention 0–35 days. Point-in-time restore to any second. Stored in S3 (not visible to you).
- **Manual snapshots** — Don't expire. Can copy across regions/accounts.
- **Storage types**: gp2 (IOPS tied to size), **gp3** (decoupled IOPS/size — recommended), io1/io2 (provisioned IOPS)
- **Storage Auto Scaling** — Automatically grows when low on space. Size only, not IOPS.
- **Encryption at rest** — Via KMS. Must enable at **creation time**. To encrypt unencrypted DB → snapshot → copy encrypted → restore.
- **IAM DB Auth** — Temporary token (15 min). No passwords in code. Supported for MySQL & PostgreSQL.
- **RDS Proxy** — Connection pooling. **Essential for Lambda** (prevent connection exhaustion). Handles failover transparently.
- **Parameter Groups** — Engine config. Static = needs restart. Dynamic = applies immediately.
- **Maintenance** — Minor version upgrades can be auto-applied. Major version upgrades are **manual**.
- **No Multi-AZ for read replicas** — Replicas can be in different AZs but replication is async.

---

## RDS vs Aurora — Decision

| Factor | Choose RDS | Choose Aurora |
|---|---|---|
| Engine needed | Oracle, SQL Server | MySQL or PostgreSQL |
| Cost-sensitive, small workload | ✅ Cheaper | ❌ ~20% more |
| HA with fast failover | ❌ 60–120s | ✅ ~30s |
| Many read replicas | ❌ Max 5 | ✅ Max 15 |
| Multi-region | ❌ Limited | ✅ Global DB (sub-second) |
| Variable traffic | ❌ | ✅ Serverless v2 |
| Storage auto-scale | ❌ Manual/provisioned | ✅ Automatic up to 128TB |

---

## Aurora — Critical

- **Compute/storage separation** — Unlike RDS (which uses EBS per instance).
- **6 copies across 3 AZs** — Write quorum = 4/6. Read quorum = 3/6.
- **Automatic storage** — Starts at 10GB, grows in 10GB increments, max **128TB**.
- **Up to 15 Aurora Replicas** — Shared storage (no data copy). ~**<100ms lag**. Replicas ARE readable and serve as failover targets.
- **Failover <30 seconds** — Promotes highest-priority replica.
- **Endpoints**:
  - **Cluster endpoint** → Always points to writer (use for writes + read-your-writes)
  - **Reader endpoint** → Round-robin across replicas (use for read traffic)
  - **Instance endpoint** → Specific instance (rarely used in apps)
  - **Custom endpoint** → Route to specific subset of replicas
- **Crash recovery in seconds** — No redo log replay (storage layer is always consistent).
- **Self-healing storage** — Automatically repairs failed storage nodes.

---

## Aurora Serverless

| | v1 | v2 |
|---|---|---|
| Production-ready | ❌ No | ✅ Yes |
| Scales to zero | ✅ Yes | ❌ No (min 0.5 ACU) |
| Scale granularity | Discrete ACU steps | Fine-grained (0.5 ACU) |
| Scale speed | 15–30s (connections may drop) | Near-instant |
| Cold start | ✅ Yes (if at zero) | ❌ No |
| **Exam verdict** | Know for dev/test, intermittent workloads | Know for production, unpredictable traffic |

---

## Aurora Global Database

- **Primary region** (write) + up to **5 secondary regions** (read-only)
- Storage-level replication — **<1 second lag** (typically 100–400ms)
- **Planned failover** = zero data loss (RPO=0). **Unplanned** = up to ~1s data loss.
- Use for: global low-latency reads, disaster recovery across regions

---

## RDS Multi-AZ vs Read Replicas (Exam Trap)

| | Multi-AZ | Read Replica |
|---|---|---|
| Purpose | HA / disaster recovery | Read scaling |
| Replication | **Synchronous** | **Asynchronous** |
| Standby readable? | **No** | **Yes** |
| Failover | Automatic | Manual (promote to primary) |
| Cross-region? | No (same region) | Yes |

---

## Aurora Backtrack (MySQL Only)

- "Rewind" the DB to a previous point in time **without restoring from backup**
- Configurable backtrack window
- **NOT** the same as point-in-time restore
- Aurora MySQL **only**, not available for Aurora PostgreSQL

---

## Security — Quick Hits

- **VPC isolation** → Place DB in private subnets. Use security groups to restrict access.
- **Public accessibility** → Never enable in production.
- **Encryption at rest** → Must enable at creation. Workaround: snapshot → copy encrypted → restore.
- **Encryption in transit** → TLS/SSL. AWS certificate bundle.
- **IAM DB Auth** → Token valid 15 min. No passwords. MySQL & PostgreSQL only.
- **Secrets Manager** → Auto-rotate DB credentials.

---

## Limits to Memorize

| Item | Limit |
|---|---|
| RDS Read Replicas | 5 |
| Aurora Replicas | 15 |
| Automated backup retention | 35 days |
| Aurora max storage | 128 TB |
| Aurora Global DB secondary regions | 5 |
| RDS Proxy (default) | 1 per DB instance |

---

## Common Exam Scenarios

1. **Need HA with zero data loss?** → RDS Multi-AZ (sync replication, RPO=0)
2. **Reduce read load on primary?** → Read Replicas (RDS) or Aurora Replicas
3. **Lambda connecting to DB?** → **RDS Proxy** (prevents connection storm)
4. **Cross-region DR?** → Cross-region Read Replica (RDS) or Aurora Global Database
5. **Unpredictable traffic, want to scale to zero?** → Aurora Serverless v1
6. **Production variable traffic, no cold starts?** → Aurora Serverless v2
7. **Encrypt existing unencrypted DB?** → Snapshot → copy encrypted → restore
8. **Connect without passwords?** → IAM Database Authentication (temporary tokens)
9. **Fast failover needed?** → Aurora (faster than RDS Multi-AZ)
10. **Multiple regions with sub-second replication?** → Aurora Global Database
