---
tags:
  - aws
  - certification
  - ec2
  - practice
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[EC2]] · [[EC2 EBS and KMS]]

# DVA-C02 — EC2: Exam Hints

## Pricing Models
| Model | Discount | Use Case |
|---|---|---|
| On-Demand | 0% | Unpredictable workloads, short-term |
| Reserved Instances | 30-72% (1 or 3 yr) | Steady, predictable production |
| Spot Instances | 70-90% | Batch, fault-tolerant, can be interrupted |
| Savings Plans | 30-72% (1 or 3 yr) | Flexible across instance types/regions |

**Exam pattern:** "Cost optimization for steady workload" → Reserved Instances or Savings Plans. "Cost optimization, can tolerate interruption" → Spot.

## Instance Types
- **T series** = burstable (small workloads, CPU credits)
- **M series** = general purpose (web servers, balanced)
- **C series** = compute optimized (batch, encoding, HPC)
- **R series** = memory optimized (databases, Redis, SAP)
- **I series** = storage optimized (NoSQL, NVMe SSD)
- **P/G series** = GPU (ML, graphics)

Naming: `m5.large` → family (m) + generation (5) + size (large)

## Instance Lifecycle
- **Stop** → EBS kept, no compute charge, public IP changes (unless Elastic IP)
- **Terminate** → EBS deleted by default (unless "Delete on termination" unchecked), permanent
- **Hibernate** → RAM saved to EBS, resumes faster than cold boot
- **Reboot** → keeps public IP, no data loss

## Security Groups (SG) vs Network ACLs (NACL)
| SG | NACL |
|---|---|
| Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Allow rules only | Allow AND Deny rules |
| Instance level | Subnet level |
| Default: deny inbound, allow outbound | Default: allow all in/out |

## Elastic IP
- Static public IP that persists through stop/start
- Costs money if not attached to a running instance
- Use when you need a consistent public IP

## IAM Roles for EC2 (CRITICAL)
- Never put access keys in EC2 — use IAM role + Instance Profile
- EC2 gets temporary credentials from **metadata service** at `http://169.254.169.254/latest/meta-data/iam/security-credentials/{role-name}`
- AWS SDK automatically calls this endpoint

## Exam Trap: Instance Store vs EBS
- **Instance Store**: physically attached SSD, extremely fast, **data LOST on stop/terminate**
- **EBS**: network-attached, **persistent through stop/start**, survives termination if configured
- If exam says "temporary cache, max performance, can be recreated" → Instance Store
