---
tags:
  - aws
  - certification
  - postgresql
  - practice
  - scaling
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[Global Infrastructure]]

# DVA-C02 — Global Infrastructure: Exam Hints

## Regions
- Each region is completely independent — data does NOT replicate across regions by default
- You must explicitly enable cross-region replication (S3 CRR, DynamoDB Global Tables, RDS read replicas)
- Service availability varies by region — not all services exist in every region
- Cost varies by region

## Availability Zones
- Minimum **2 AZs** per region, typically 3
- Multi-AZ = High Availability (survives one AZ going down)
- ELB + ASG across AZs = standard HA pattern for compute
- RDS Multi-AZ = synchronous standby in another AZ, automatic failover (~60-120 sec)
- EBS is single-AZ only — use snapshots to move data across AZs
- **AZ IDs are randomized per account** — us-east-1a in your account is not the same physical rack as us-east-1a in another account

## Edge Locations & CloudFront
- Edge locations are NOT regions — they are read-only caches (~600+ globally)
- CloudFront caches content at edge locations for lower latency
- **Lambda@Edge** runs code at edge locations
- PriceClass_100 = cheapest (covers 90%+ of traffic), PriceClass_All = all edge locations (most expensive)

## Global vs Regional Services (COMMON EXAM TRAP)
| Global (no region) | Regional (pick a region) |
|---|---|
| IAM | EC2, RDS, DynamoDB |
| CloudFront | Lambda |
| Route 53 | S3 (regional but globally accessible) |
| Budgets/Billing | KMS, EBS, VPC |

## Exam Scenarios
- "Survive AZ failure" → Multi-AZ deployment
- "Survive region failure" → Multi-region (Global Tables, S3 CRR, RDS cross-region read replicas)
- "Global users slow" → CloudFront CDN
- "Data must stay in EU" → deploy in eu-west-1/eu-central-1, disable cross-region replication, add bucket policy to block non-EU access
