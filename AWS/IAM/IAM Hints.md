---
tags:
  - aws
  - certification
  - iam
  - practice
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[IAM]]

# DVA-C02 — IAM: Exam Hints

## Policy Evaluation Logic
1. Default = **implicit DENY**
2. Explicit **DENY** always wins (overrides any ALLOW)
3. If no DENY and there's an ALLOW → access is allowed
4. If no DENY and no ALLOW → access is denied

## Root User
- Enable **MFA** immediately on root account
- Only use root for: initial account setup, billing, close account, change email
- NEVER use root for daily work
- NEVER share root credentials

## Users vs Roles
| Users | Roles |
|---|---|
| Permanent credentials (access key + secret) | Temporary credentials (STS, auto-rotated) |
| For humans | For applications/services |
| Can't be used by EC2 easily | EC2 assumes role via Instance Profile |

**Rule of thumb:** EC2/Lambda → always use **roles**. Humans → use **users** with MFA.

## Policies
- Identity-based policy = "what can this user/role do?" (attached to IAM identity)
- Resource-based policy = "who can access this resource?" (attached to resource like S3 bucket)
- **Both policies must allow** for cross-account access to work

## Cross-Account Access
1. Account A creates an IAM role with permissions and a **trust policy** allowing Account B
2. Account B user assumes that role via STS AssumeRole
3. After assuming, **only the role's policies apply** — the user's original policies no longer matter

## Common Exam Traps
- ❌ Hardcoding credentials in code → ✅ Use IAM roles for EC2/Lambda
- ❌ Forgetting DENY always wins → If bucket policy denies and IAM allows → DENIED
- ❌ Using IAM users for applications → ✅ Use roles for temporary credentials
- ❌ Missing key policy → KMS access requires BOTH IAM policy AND key policy
- ❌ Giving `"Action": "*"` → violates least privilege principle

## Key Numbers to Memorize
- Max 2 access keys per IAM user
- STS session duration: 15 min to 12 hours (default 1 hour)
- Max 10 policies per role/user
- Managed policy max size: 10 KB
