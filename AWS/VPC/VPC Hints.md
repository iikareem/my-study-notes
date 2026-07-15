---
tags:
  - aws
  - certification
  - network
  - practice
  - vpc
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[VPC]] · [[VPC Quiz]]

# DVA-C02 — VPC: Exam Hints

## Security Groups (SG) vs Network ACLs (NACL)

| SG | NACL |
|---|---|
| **Stateful**: if inbound allowed, response outbound is auto-allowed | **Stateless**: must explicitly allow both inbound AND outbound |
| Allow rules only (no Deny) | Allow AND Deny rules (rule numbers evaluated in order) |
| Instance level (attached to ENI) | Subnet level (applies to all instances in subnet) |
| Default: deny all inbound, allow all outbound | Default NACL: allow all inbound + outbound |
| Changes take effect immediately | Changes take effect immediately |

**Exam trap:** If inbound web traffic works but responses don't return → check if the issue is NACL (stateless, missing ephemeral port allow outbound) vs SG (stateful, return traffic auto-allowed).

## Reserved IPs per Subnet
AWS reserves **5 IP addresses** in every subnet:
- `.0` — network address
- `.1` — VPC router
- `.2` — DNS server
- `.3` — future use
- `.255` — broadcast

A `/24` subnet (256 IPs) → only **251 usable** IPs.

## Internet Gateway (IGW)
- Makes a subnet **public**
- Must be **attached** to the VPC
- Route table in the public subnet must have `0.0.0.0/0 → igw-id`

## NAT Gateway
- Allows **private subnets** to access the internet (outbound only)
- Must be placed in a **public subnet**
- Requires an **Elastic IP**
- Private subnet route table: `0.0.0.0/0 → nat-gateway-id`
- AWS managed (auto-scaling, no patching needed) — higher cost than NAT Instance

## VPC Endpoints
| Gateway Endpoint | Interface Endpoint (PrivateLink) |
|---|---|
| **Free** | **Hourly cost** |
| Added to route table | Uses ENI with private IP |
| Only for **S3 and DynamoDB** | For most other AWS services |
| No extra data transfer cost | Data transfer charges apply |

**Exam scenario:** "EC2 in private subnet needs S3 access without internet" → **Gateway Endpoint for S3**

## VPC Peering
- Connects two VPCs via AWS private network
- **NOT transitive**: if A is peered with B, and B is peered with C, A cannot reach C through B
- No overlapping CIDR blocks allowed
- Route tables must be updated in both VPCs

## Lambda in VPC
- Lambda in a private subnet can't reach internet unless there's a **NAT Gateway**
- Lambda in a public subnet does NOT get internet access automatically (Lambda ENIs don't get public IPs)

## Exam Traps
- ❌ Trying to block a specific IP with Security Group → must use **NACL** (SGs only allow, not deny)
- ❌ Forgetting NACL is stateless → need both inbound AND outbound rules
- ❌ Using NAT Gateway for S3/DynamoDB → use **Gateway Endpoint** (cheaper, faster)
- ❌ Assuming VPC Peering is transitive → it's NOT
- ❌ Expecting 256 IPs from /24 → only 251 usable (AWS reserves 5)
