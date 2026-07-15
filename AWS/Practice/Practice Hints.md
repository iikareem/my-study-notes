---
tags:
  - aws
  - certification
  - postgresql
  - practice
  - scaling
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[Mixed Services Practice]] · [[Scenario Practice]]

# DVA-C02 — How to Use These Practice Questions

## Study Strategy
1. **Attempt all 40 questions** before looking at answers
2. Mark your answers, then check the answer key
3. For every wrong answer: understand WHY the correct answer is right AND why your choice was wrong
4. Re-attempt the same 40 questions 2 days later — aim for 80%+
5. On exam day: quickly review only your previously wrong answers

## Most Repeated Patterns in These Questions
- Cross-account access → always uses **assume role with trust policy**, never share credentials
- EBS encryption → must **snapshot → create encrypted volume** (can't encrypt in-place)
- KMS access denied → check **BOTH IAM policy AND key policy**
- EC2 needs AWS access → **IAM role + instance profile**, never hardcoded keys
- Surviving AZ failure → **Multi-AZ (RDS) + ASG across AZs + ELB**
- Surviving region failure → **cross-region replication** (S3 CRR, Global Tables, RDS read replicas)
- Cost optimization → **Spot for fault-tolerant, Reserved/Savings Plans for steady, On-Demand for flexible**
