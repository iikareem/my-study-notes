---
tags:
  - aws
  - certification
  - practice
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[AWS Review]]

# DVA-C02 — Quick Review Sheet

## Global Infrastructure
- Multi-AZ = survive AZ failure. Multi-region = survive region failure
- CloudFront = CDN at edge locations (~600+ globally)
- Global services: IAM, CloudFront, Route 53, Budgets
- Regional services: EC2, RDS, DynamoDB, Lambda, KMS, S3

## IAM
- Explicit **DENY** always beats ALLOW
- **Roles** for apps/services (temporary creds via STS)
- **Users** for humans (permanent creds + MFA)
- Both IAM policy + resource policy needed for cross-account
- Least privilege: specific actions + specific resource ARNs

## EC2
- On-Demand = flexible. Reserved = 30-72% off (steady). Spot = 70-90% off (interruptible)
- Stop → no compute charge, EBS kept. Terminate → EBS deleted by default
- Instance Store = ephemeral (lost on stop). EBS = persistent
- SG = stateful, allow only. NACL = stateless, allow + deny

## KMS
- Regional. Encrypt max 4KB. GenerateDataKey for >4KB
- Access = IAM policy + Key policy (BOTH)
- Customer managed = $1/month, full control, auto-rotation yearly

## DynamoDB
- RCU = 4KB/read (strong), WCU = 1KB/write. Transactions = 2x cost
- LSI = same PK, creation only, strong consistency
- GSI = diff PK, any time, eventual only
- DAX = microsecond reads (eventual only). TTL = free deletes

## Lambda
- Max 15 min, 10GB memory, 10GB /tmp
- Cold start → init outside handler. Provisioned concurrency = no cold starts
- In VPC → loses internet → needs NAT Gateway
- Reserved concurrency = hard limit. Error 429 = throttled

## VPC
- 5 IPs reserved per subnet (/24 = 251 usable)
- Gateway Endpoint for S3/DynamoDB = free
- NAT Gateway in public subnet for private internet access
- VPC Peering = not transitive
