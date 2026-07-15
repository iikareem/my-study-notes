---
tags:
  - aws
  - certification
  - kms
  - practice
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[KMS]]

# DVA-C02 — KMS: Exam Hints

## Key Types
| Type | Cost | Control | Use Case |
|---|---|---|---|
| AWS Owned | Free | None | Internal AWS use, no CloudTrail |
| AWS Managed | Free (API only) | Limited | Default encryption (aws/s3, aws/ebs) |
| Customer Managed | $1/month/key | Full | Production, compliance, audit |

## Envelope Encryption
- KMS Encrypt only works on data up to **4 KB**
- For data >4 KB → use **GenerateDataKey**
- GenerateDataKey returns: plaintext data key + encrypted data key
  - Use plaintext key to encrypt your data, then discard it
  - Store encrypted data key alongside your encrypted data
  - To decrypt: send encrypted key to KMS → get plaintext → decrypt data

## Access Control — BOTH Required
1. **IAM policy** on the user/role (allows kms:Decrypt, etc.)
2. **Key policy** on the KMS key (allows the principal)
- If either is missing → **Access Denied**

## KMS is Regional
- A KMS key created in us-east-1 CANNOT be used in us-west-2
- For cross-region: use **Multi-Region Keys** (same key material, same key ID across regions)
- Or: create separate keys per region

## Key Rotation
- AWS Managed keys = automatic (every 3 years, can't disable)
- Customer Managed keys = automatic every **1 year** (can enable/disable)
- Old backing key retained — old data stays readable, new data uses new key

## Exam Scenarios
- "Encrypt existing EBS volume" → Snapshot → Create encrypted volume from snapshot
- "Encrypt >4KB in app" → GenerateDataKey
- "S3 encryption with audit trail" → SSE-KMS with customer managed key
- "EC2 can't decrypt using KMS" → Check BOTH: EC2 role IAM policy + KMS key policy
- "Throttling on KMS" → Use S3 Bucket Keys or Data Key Caching (don't just increase limits)
