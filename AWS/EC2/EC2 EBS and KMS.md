---
tags:
  - aws
  - certification
  - ec2
  - kms
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[EC2]] · [[EC2 Hints]] · [[KMS]]

# AWS Developer Associate Exam: EC2 (Elastic Compute Cloud)

## Complete Study Guide with Critical Concepts

---

## PART 1: EC2 FUNDAMENTALS

### 1.1 What is EC2?

**Definition**: Elastic Compute Cloud - Virtual machines in the AWS cloud

**Key Characteristics**:

- **Resizable**: Scale up/down compute capacity
- **On-Demand**: Pay only for what you use
- **Secure**: VPC, Security Groups, IAM roles
- **Reliable**: Run on AWS infrastructure
- **Elastic**: Auto Scaling capabilities

**Pricing Models** (EXAM CRITICAL):

```
1. ON-DEMAND (Default)
   - Pay per hour/second (Linux/Unix after 60 seconds)
   - Highest price, maximum flexibility
   - Use: Unpredictable workloads, short-term needs
   - Exam Tip: When you don't know pricing, answer is ON-DEMAND

2. RESERVED INSTANCES (RI) - COMMITMENT
   - Commit 1 or 3 years, get 30-70% discount
   - Must pick: Instance type, region, OS
   - Unused RIs = wasted money
   - Use: Stable, predictable workloads
   - Can convert between instance types (Convertible RI)

3. SPOT INSTANCES - CHEAPEST
   - 70-90% discount off On-Demand
   - AWS can terminate anytime (2-minute notice)
   - Bid for spare capacity
   - Use: Fault-tolerant workloads (batch jobs, data processing)
   - NOT for critical production

4. DEDICATED HOSTS
   - Physical server just for you
   - Highest price
   - Use: Licensing requirements, compliance

5. SAVINGS PLANS
   - Flexible commitment (1 or 3 years)
   - Can change instance type, region, OS
   - 30-70% discount
   - More flexible than Reserved Instances
```

**Exam Question Pattern**:

> "You need cost optimization for predictable workload" **Answer**: Reserved Instances or Savings Plans
> 
> "You have flexible workload, any instance type OK" **Answer**: Savings Plans (more flexible than RI)

---

### 1.2 Instance Types (EXAM IMPORTANT)

#### **General Purpose (T/M series)**

- **T3/T4g**: Burstable performance, small workloads
    - Good for development, low-traffic apps
    - CPU credits system (accumulate when idle)
    - **Exam Tip**: When you need small, burstable → T series
- **M5/M6**: Balanced compute, memory, networking
    - Most popular for general applications
    - Web servers, small databases

**When to use**: Web servers, small apps, development

#### **Compute Optimized (C series)**

- **C5/C6**: High CPU performance
- More CPU than memory
- Use: Batch processing, HPC, gaming servers, video encoding
- **Exam Tip**: High compute, low memory → C series

#### **Memory Optimized (R/X series)**

- **R5/R6**: Large memory pools, good CPU
- Use: In-memory databases (Redis), data analytics, SAP
- **Exam Tip**: Lots of memory → R series

#### **Storage Optimized (I/D/H series)**

- **I3/I4**: NVMe SSD storage, high I/O
- Use: NoSQL databases, data warehousing
- **Exam Tip**: High I/O, fast storage → I series

#### **GPU Accelerated (P/G/F series)**

- **P3**: ML training
- **G4**: Graphics-intensive, ML inference
- **F1**: FPGA for custom hardware
- Use: Machine learning, video encoding, graphics

**Naming Convention**:

```
m5.large
├─ m = instance family (General Purpose)
├─ 5 = generation (newer = better)
└─ large = size (small, medium, large, xlarge, 2xlarge...)
```

---

### 1.3 Instance Lifecycle (EXAM CRITICAL)

#### **States**

```
pending ──→ running ──→ stopping ──→ stopped ──→ terminated
  (0-1 min)   (stable)    (brief)     (stable)   (permanent)
  
  └─────────────────────────────────────────────────────┘
                     rebooting
                    (few seconds)
```

**Key Points**:

- **Stopped Instance**: EBS volumes retained, elastic IPs retained
- **Terminated Instance**: EBS volumes deleted (unless root is set to persist)
- **Reboot**: Doesn't stop instance, keeps public IP
- **Exam Tip**: Data loss only happens on TERMINATION

#### **Stopping vs Terminating**

```
STOPPED:
✅ Don't pay for compute
✅ EBS volumes kept
✅ Elastic IP retained
✅ Can restart later

TERMINATED:
❌ Can't restart
❌ EBS deleted (unless configured otherwise)
❌ Elastic IP released (unless reassigned)
❌ Permanent deletion
```

---

### 1.4 Security Groups & Network ACLs

#### **Security Groups** (Firewall for EC2)

- **Stateful**: If outbound allowed, inbound response allowed
- **Default**: Allow all outbound, deny all inbound
- Changes take effect immediately
- Can attach multiple security groups to one instance

**Inbound Rules Example**:

```
Protocol: TCP
Port Range: 80 (HTTP)
Source: 0.0.0.0/0 (anywhere)
```

**Key Points**:

- Allow rules only (can't deny explicitly)
- DENY uses Network ACLs instead
- Default security group allows traffic between members
- **Exam Tip**: "Allow SSH from office" = Security Group rule

#### **Network ACLs** (Optional Layer)

- **Stateless**: Must explicitly allow return traffic
- Subnet level (not instance level)
- Support both Allow and Deny rules
- Rarely tested on Developer Associate

**When to Use**:

```
Security Group: Most cases (application firewall)
Network ACL: Extra security layer (rarely needed)
```

---

### 1.5 Elastic IP & Network Interfaces

#### **Elastic IP (EIP)**

- Static public IP address
- Reassignable to different instances
- Associated with VPC
- Pay hourly if not assigned to running instance
- **Exam Tip**: Use EIP when you need persistent IP

#### **ENI (Elastic Network Interface)**

- Network card for EC2
- Can have multiple ENIs per instance
- Each ENI can have multiple IP addresses
- **Exam Scenario**: Primary ENI + secondary ENI for multiple IPs

#### **Private IP vs Public IP**

```
PRIVATE IP (172.31.x.x or 10.x.x.x):
- Always assigned
- For internal communication within VPC
- Not internet-routable
- Persists through stop/start

PUBLIC IP (random):
- Assigned only if public subnet + public IP setting
- Changes on stop/start
- Not recommended for persistence → use EIP instead
- Internet-routable

ELASTIC IP (static):
- User-chosen
- Persists through stop/start
- Associated with private IP
- Can be reassigned
```

---

## PART 2: EBS (ELASTIC BLOCK STORE)

### 2.1 EBS Fundamentals

**Definition**: Persistent block storage volumes for EC2 instances

**Key Characteristics**:

- **Persistent**: Data survives instance stop/start
- **Network Attached**: Separate from instance
- **Encryption**: Can be encrypted at rest
- **Snapshots**: Point-in-time backups
- **Single AZ**: Cannot span across AZs directly

**Relationship to EC2**:

```
EC2 Instance (in us-east-1a) ──→ EBS Volume (in us-east-1a)
                                  (same AZ only)
```

---

### 2.2 EBS Volume Types (VERY IMPORTANT FOR EXAM)

#### **SSD Volumes (Fast, For IOPS)**

**General Purpose: gp3 (LATEST, EXAM FOCUS)**

```
Performance:
- 3,000 IOPS baseline (no extra cost)
- Up to 16,000 IOPS (additional cost)
- 125 MB/s to 1,000 MB/s throughput
- Cost: Moderate

Use Cases:
- General workloads
- MySQL, PostgreSQL
- Small databases
- Most common choice

Exam Tip: When unsure between gp2/gp3 → gp3 is newer, better
```

**General Purpose: gp2 (OLDER, LEGACY)**

```
Performance:
- Max 16,000 IOPS (burst)
- Performance tied to volume size (3 IOPS per GB)
- Smaller volume = worse performance

Use Cases:
- Existing applications
- Legacy deployments
- Cost-sensitive (slightly cheaper than gp3)

Exam Tip: gp3 supersedes gp2, but both on exam
```

**High Performance: io1/io2 (Database)**

```
Performance:
- Up to 64,000 IOPS (io1) or 64,000 IOPS (io2)
- io2 has higher durability (99.999% vs 99.99%)
- 6.5 GB/s throughput

Cost: Expensive (pay per provisioned IOPS)

Use Cases:
- Transactional databases (MySQL, Oracle, SQL Server)
- NoSQL databases (MongoDB, Cassandra)
- High-I/O applications
- Production databases

Exam Tip: Slow database queries? → io1/io2 for better IOPS
```

#### **HDD Volumes (Slow, For Throughput)**

**Throughput Optimized: st1**

```
Performance:
- 500 IOPS max
- Up to 500 MB/s throughput
- Fast sequential reads/writes

Cost: Cheaper than SSD

Use Cases:
- Big data workloads
- Hadoop, data warehousing
- Log processing
- Sequential, not random access
```

**Cold Storage: sc1**

```
Performance:
- 250 IOPS max
- 250 MB/s throughput
- Cheapest option

Use Cases:
- Archive storage
- Cold backup storage
- Infrequent access
```

#### **Quick Decision Tree for Exam**

```
Need database performance?
├─ YES → io1/io2
└─ NO → gp3 (general purpose, most common)

Need maximum throughput (streaming)?
├─ YES → st1 (throughput optimized)
└─ NO → gp3

Need cheapest archive storage?
├─ YES → sc1
└─ NO → gp3

Exam Pattern: If unsure and no special requirement → gp3
```

---

### 2.3 EBS Features & Operations (CRITICAL)

#### **Volume Encryption**

- **At-Rest Encryption**: Data encrypted on disk using KMS
- **In-Transit Encryption**: Data encrypted between EC2 and EBS
- Enabled at creation time
- **Important**: Cannot encrypt existing volume (must snapshot)

**Process to Encrypt Existing Volume**:

```
1. Create snapshot of unencrypted volume
2. Create encrypted volume from snapshot
3. Attach encrypted volume to instance
4. Detach old unencrypted volume
```

**Exam Question**:

> "How do you encrypt an existing unencrypted EBS volume?" **Answer**: Create snapshot → Create encrypted volume from snapshot → Attach to instance

#### **Snapshots**

- **Point-in-time backup** of EBS volume
- Stored in S3 (but user doesn't see it)
- Incremental (only changed blocks stored)
- Can create AMI from snapshot
- Can share snapshots (cross-account)
- **Cost**: Pay for storage, not per snapshot

**Key Timing**:

- First snapshot: Full copy
- Subsequent snapshots: Only changed blocks
- **Exam Tip**: Snapshots are cheaper than full copies

#### **Creating Image from Snapshot**

```
EBS Volume → Snapshot → AMI (reusable template)
                          ↓
                    Launch new EC2 instances
```

#### **EBS Optimization**

- Dedicated throughput between EC2 and EBS
- Not all instance types support it
- Reduces contention with network traffic
- **Exam Tip**: Production databases → use EBS-optimized instances

#### **RAID on EBS** (Not common for exam, but know basics)

```
RAID 0 (Striping): 
- Higher performance, lower redundancy
- Lose one volume = lose everything

RAID 1 (Mirroring):
- Higher redundancy, lower performance
- Duplicate writes to both volumes

Typically not needed because:
- EBS already redundant within AZ
- Better to use Multi-AZ database
```

---

### 2.4 EBS vs Instance Store Storage

#### **Instance Store (Ephemeral Storage)**

```
Characteristics:
- Physical storage on host machine
- Data LOST when instance stops/terminates
- Very fast (NVMe SSD)
- No additional cost (included)
- Cannot be detached

Use Cases:
- Temporary data
- Cache
- Staging
- NOT for persistent data

Exam Tip: Instance Store = LOST on stop. EBS = KEPT on stop
```

#### **EBS vs Instance Store Comparison**

```
                EBS             Instance Store
Persistence    YES (stop/start) NO (lost on stop)
Detachable     YES             NO
Speed          Fast            Very fast (NVMe)
Cost           Pay per GB      Included
Use Case       Persistent data Temporary data
```

---

### 2.5 Throughput vs IOPS (EXAM IMPORTANT)

#### **Understanding the Difference**

```
IOPS (Input/Output Operations Per Second):
- Number of operations per second
- Important for: Random access, databases
- Measured in: IOPS
- Example: 10,000 IOPS = 10,000 operations/sec

THROUGHPUT (MB/s):
- Amount of data transferred per second
- Important for: Sequential access, streaming
- Measured in: MB/s
- Example: 500 MB/s = 500 megabytes/sec
```

#### **Real-World Analogy**

```
IOPS = Number of cars crossing intersection per minute
THROUGHPUT = Total weight of cargo delivered per minute

Database = Many small operations (IOPS matters)
Video streaming = Large sequential reads (Throughput matters)
```

#### **Which Volume Type for Which Workload?**

```
Need lots of random operations? (Database)
├─ YES, IOPS critical → gp3 or io1/io2
└─ NO, Sequential OK → st1 (throughput optimized)

Database running slow?
├─ Check IOPS → upgrade to io1/io2
└─ Check throughput → might be network, not EBS

Streaming/Big Data running slow?
├─ Check throughput → upgrade to st1
└─ Check IOPS → st1 has enough for sequential
```

---

## PART 3: KMS (KEY MANAGEMENT SERVICE)

### 3.1 KMS Fundamentals

**Definition**: AWS service to create and manage encryption keys

**Key Concepts**:

- **Managed encryption** for AWS services
- **KMS Keys** replace older "CMK" (Customer Master Key) terminology
- Control who can use keys (IAM policies)
- **Regional** service (different keys per region)
- Integrated with CloudTrail for audit

**When to Use KMS**:

```
✅ Encrypting EBS volumes
✅ Encrypting S3 objects
✅ Encrypting RDS databases
✅ Encrypting secrets/credentials
✅ Application-level encryption

❌ Don't use for: Just hashing passwords (use bcrypt/Argon2)
```

---

### 3.2 KMS Keys vs AWS Managed Keys

#### **Types of KMS Keys**

**AWS Managed Keys (Default)**

```
Who manages?
├─ AWS manages creation, rotation, deletion
├─ You cannot delete
├─ Free to use (within AWS service)

Key Naming:
├─ aws/s3 (for S3 encryption)
├─ aws/ebs (for EBS encryption)
├─ aws/rds (for RDS encryption)

Use: Default encryption, low security needs
Exam Tip: "By default, S3 uses aws/s3 managed key"
```

**Customer Managed Keys (Custom)**

```
Who manages?
├─ You create and manage
├─ You pay per key
├─ You can delete (and enable deletion, 7-30 day wait)
├─ You control key policy

Control:
├─ Full control over key rotation
├─ Control who can use key (key policy)
├─ Can replicate to other regions

Use: Production, compliance, multi-team environments
Exam Tip: When you need full control → Customer Managed Key
```

**Key Differences Summary**:

```
                    AWS Managed    Customer Managed
Cost                Free           $1/month
Control             Limited        Full
Rotation            Automatic      Manual or Automatic
Deletion            No             Yes (7-30 day wait)
Use Cases           Development    Production
Compliance          Basic          Full auditing
Key Policy          No             Yes
```

---

### 3.3 KMS Key Rotation (EXAM IMPORTANT)

#### **Automatic Rotation (Customer Managed Keys)**

```
Enabled by default for customer managed keys:
- New backing key created annually
- Old keys kept for decryption
- No action needed from you
- Audit in CloudTrail

Scenario: You encrypt file with key on Jan 1
         On Jan 1 next year, new key created
         Old data still decrypts with old key
         New data encrypts with new key
```

#### **Manual Rotation**

```
When needed:
- Create new KMS key
- Update application to use new key
- Keep old key for decryption of old data

Difference from automatic:
- Automatic: Same key ID, internal backing key changes
- Manual: Different key ID, need to update app

Exam Tip: Automatic rotation is default and recommended
```

---

### 3.4 Key Policy & Access Control

#### **KMS Key Policy (Trust Policy for Keys)**

- **JSON document** defining who can use the key
- Similar to IAM policy but for keys
- Must explicitly grant permissions
- **Root principle**: Principal needs:
    1. IAM permission (via role/user policy)
    2. Key policy grant (via key policy)

**Example: Allow EC2 Role to Use Key**

```json
{
  "Sid": "Allow EC2 Role to use key",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789:role/EC2Role"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

#### **Key IAM Policy (User Side)**

```json
{
  "Sid": "Allow user to encrypt with key",
  "Effect": "Allow",
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "arn:aws:kms:us-east-1:123456789:key/12345-key-id"
}
```

#### **Both Policies Required**

```
User wants to decrypt with KMS key:

1. User's IAM policy must allow:
   "Action": "kms:Decrypt"

2. Key policy must allow:
   "Principal": "arn:aws:iam::123456789:user/john"
   "Action": "kms:Decrypt"

If EITHER is missing → Access Denied
```

---

### 3.5 KMS Operations (EXAM CRITICAL)

#### **Key Operations**

**Encrypt / Decrypt**

```
Encrypt(Plaintext, KeyId) → Ciphertext
Decrypt(Ciphertext) → Plaintext

Max plaintext size: 4 KB
Larger data: Use data key (see below)
```

**GenerateDataKey**

```
Purpose: Encrypt large data (>4KB)
Returns:
- Plaintext data key (for immediate use)
- Encrypted data key (for storage)

Workflow:
1. Call GenerateDataKey
2. Get plaintext key + encrypted key
3. Use plaintext key to encrypt your data
4. Discard plaintext key
5. Store encrypted data + encrypted key together
6. Later: Decrypt encrypted key with KMS → get plaintext → decrypt data

Exam Tip: For encrypting large objects → GenerateDataKey
```

**GenerateDataKeyWithoutPlaintext**

```
Purpose: Same as GenerateDataKey, but don't need plaintext immediately

Returns: Only encrypted data key

Use when: You generate key to send someone else
They'll call Decrypt on encrypted key to get plaintext
```

**Decrypt**

```
Works on:
- Encrypted ciphertext (from Encrypt)
- Encrypted data key (from GenerateDataKey)

Returns: Plaintext

What you need:
- Access to KMS key (IAM policy + key policy)
- Ciphertext/encrypted data key
```

---

### 3.6 Encryption in AWS Services

#### **S3 Encryption Options**

```
1. Server-Side Encryption (SSE):
   
   SSE-S3 (Default):
   ├─ S3 manages keys (aws/s3)
   ├─ Free
   ├─ No key policy needed
   ├─ Good for: General use
   
   SSE-KMS:
   ├─ KMS manages keys (customer managed or AWS managed)
   ├─ You control access
   ├─ Audit with CloudTrail
   ├─ Good for: Compliance, multi-team
   
   SSE-C:
   ├─ You provide key in each request
   ├─ AWS doesn't store key
   ├─ You manage key rotation
   ├─ Good for: Maximum control, external systems

2. Client-Side Encryption:
   ├─ Encrypt before sending to S3
   ├─ S3 never sees plaintext
   ├─ You manage encryption
   ├─ Most secure but complex
```

**Exam Question Pattern**:

> "You need encryption in S3 with full audit trail" **Answer**: SSE-KMS with customer managed key

#### **EBS Encryption**

```
Creation:
- Enable encryption during volume creation
- Select KMS key (default aws/ebs or custom)
- All data encrypted at rest

Access:
- EC2 instance needs IAM role with kms:Decrypt
- Instance needs key policy grant
- Automatic after that

Changing Encryption:
- Cannot modify existing volume
- Create snapshot + encrypted volume from it
```

#### **RDS Encryption**

```
Enable at Creation:
- During RDS instance creation
- Select KMS key
- All data at rest encrypted

Multi-AZ:
- If Multi-AZ enabled, standby encrypted with same key
- Failover still works

Backup:
- Automated backups encrypted with same key
- Manual snapshots also encrypted
```

---

### 3.7 Encryption in Transit

#### **HTTPS/TLS**

```
When data moves between:
- Client to AWS service → Use HTTPS
- Service to service → Use VPC endpoints + TLS

How to Enforce:
- Bucket policy requiring aws:SecureTransport
- Security group allowing only 443 (HTTPS)
```

**S3 Bucket Policy Example**:

```json
{
  "Sid": "Deny unencrypted connections",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::bucket/*",
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

---

### 3.8 Key Rotation Best Practices

#### **Rotation Strategy**

```
AWS Managed Keys (aws/s3, aws/ebs):
├─ Automatic rotation (annual)
├─ You can't disable
├─ Recommended practice

Customer Managed Keys:
├─ Default: Automatic (annual)
├─ Can be enabled/disabled
├─ Can manually rotate if needed

Manual Rotation Process:
1. Create new key
2. Update application to use new key
3. Keep old key for decryption
4. No data re-encryption needed (old data stays readable)
```

#### **When to Rotate Keys**

```
✅ Automatically (annual) - default
✅ Manually when:
   - Suspected compromise
   - Compliance requirement
   - Personnel change (developer left)
   - Key material exposed

❌ Don't delete old keys (needed for decryption)
```

---

### 3.9 Common Exam Scenarios

#### **Scenario 1: Encrypt EBS Volume**

```
Question: "Need to encrypt existing EBS volume with KMS"

Answer Process:
1. Create snapshot of unencrypted volume
2. Create new encrypted EBS volume from snapshot
   - Select customer managed KMS key
3. Attach new volume to EC2 instance
4. Verify EC2 instance has IAM role with kms:Decrypt
5. Verify KMS key policy allows EC2 role to decrypt

Exam Tip: Cannot directly encrypt, must use snapshot
```

#### **Scenario 2: S3 with KMS Encryption**

```
Question: "Configure S3 with KMS encryption and audit trail"

Answer:
1. Create customer managed KMS key
2. Create S3 bucket
3. Configure bucket default encryption → SSE-KMS → select key
4. Enable CloudTrail for KMS API calls
5. Bucket policy requires HTTPS

IAM role for application:
- kms:Decrypt
- s3:GetObject
KMS key policy:
- Allow the role principal
```

#### **Scenario 3: Cross-Account Encryption**

```
Question: "Account A needs to decrypt data encrypted by Account B"

Answer:
1. Account B creates customer managed key
2. Account B adds Account A principal to key policy:
   "Principal": "arn:aws:iam::AccountA:role/MyRole"
   "Action": ["kms:Decrypt", "kms:GenerateDataKey"]
3. Account A role has IAM policy allowing kms:Decrypt
4. Account A can now decrypt data

Key: BOTH key policy AND IAM policy needed
```

#### **Scenario 4: Encrypting Sensitive Data in Application**

```
Question: "Application needs to store encrypted secrets (passwords, API keys)"

Answer: Use AWS Secrets Manager or Parameter Store (with KMS encryption)

Secrets Manager:
- Automatic rotation support
- Requires: KMS key + IAM role with kms:GenerateDataKey

Parameter Store:
- Free tier (10 parameters)
- Secure String type = KMS encrypted
- Requires: KMS key + IAM role with kms:Decrypt
```

---

## EXAM QUICK REFERENCE

### EC2 Must-Knows

- [ ] On-Demand = most flexible, highest price
- [ ] Reserved Instances = 1/3 year commitment, 30-70% discount
- [ ] Spot = cheapest (70-90% off), can be terminated
- [ ] T-series = burstable, small workloads
- [ ] C-series = compute optimized
- [ ] R-series = memory optimized
- [ ] Stopped instance = EBS kept, no compute charge
- [ ] Terminated = permanent deletion, EBS deleted
- [ ] Security Group = stateful, allow rules only
- [ ] Elastic IP = static public IP, reassignable

### EBS Must-Knows

- [ ] Persistent (survives stop/start)
- [ ] Single AZ only (cannot span AZs)
- [ ] gp3 = default general purpose, replaces gp2
- [ ] io1/io2 = databases requiring high IOPS
- [ ] st1 = throughput optimized for big data
- [ ] sc1 = cold storage, cheapest
- [ ] Snapshots = incremental backups to S3
- [ ] Encrypt existing volume = snapshot → encrypted volume
- [ ] Instance Store = ephemeral, lost on stop
- [ ] EBS vs Instance Store = persistent vs temporary
- [ ] IOPS = random operations, Throughput = sequential
- [ ] EBS-optimized instances for production databases

### KMS Must-Knows

- [ ] AWS Managed Keys = free, limited control
- [ ] Customer Managed Keys = you control, costs $1/month
- [ ] Key rotation = automatic (annual) by default
- [ ] Encrypt = works on plaintext up to 4KB
- [ ] GenerateDataKey = for encrypting large data (>4KB)
- [ ] Key Policy + IAM Policy both needed for access
- [ ] S3 SSE-KMS = encryption with custom keys + audit
- [ ] EBS encryption = select KMS key during creation
- [ ] Cannot encrypt existing volume → snapshot method
- [ ] CloudTrail = audit KMS API calls
- [ ] Regional service = different keys per region

### Common Mistakes on Exam

- ❌ Using instance public IP as permanent (should use EIP)
- ❌ Assuming terminated instance data persists (it doesn't)
- ❌ Choosing wrong volume type (gp2 instead of gp3)
- ❌ Forgetting EC2 role needs kms:Decrypt for encryption
- ❌ Not understanding snapshot is incremental
- ❌ Using AWS Managed Key when full control needed
- ❌ Forgetting both key policy AND IAM policy needed
- ❌ Assuming IOPS and throughput are same thing
- ❌ Not encrypting in transit (HTTP instead of HTTPS)
- ❌ Trying to directly encrypt existing EBS volume

---

## Practice Questions

### EC2

1. **Q**: You need highly available web application that survives AZ failure. **A**: Use ASG + ELB across multiple AZs with on-demand instances
    
2. **Q**: Cost optimization for steady 24/7 production workload. **A**: Reserved Instances (1-3 year) for 30-70% discount
    
3. **Q**: Batch processing job that can be interrupted. **A**: Spot Instances for 70-90% cost savings
    
4. **Q**: Application needs persistent static IP even after stop/start. **A**: Elastic IP
    
5. **Q**: Database queries are slow, volume is gp2. **A**: Upgrade to io1/io2 for better IOPS
    

### EBS

1. **Q**: Encrypting existing EBS volume, how? **A**: Create snapshot → Create encrypted volume from snapshot → Attach
    
2. **Q**: Database needs 50,000 IOPS consistently. **A**: io1 or io2 volume type
    
3. **Q**: Big data streaming at 400 MB/s needed. **A**: st1 (throughput optimized)
    
4. **Q**: Need backup of EBS volume, what's cheapest? **A**: Snapshot (incremental, stored in S3)
    

### KMS

1. **Q**: Need encryption with full audit trail for S3. **A**: SSE-KMS with customer managed key
    
2. **Q**: EC2 instance can't decrypt with KMS key. **A**: Check both: EC2 role IAM policy + KMS key policy
    
3. **Q**: Want KMS key automatically rotated annually. **A**: Automatic key rotation enabled by default
    
4. **Q**: Encrypt data >4KB with KMS. **A**: Use GenerateDataKey (not Encrypt)
    
5. **Q**: Application needs encrypted secrets (passwords). **A**: Use AWS Secrets Manager with KMS key
    

---

## Additional Resources for Study

- AWS EC2 Documentation: https://docs.aws.amazon.com/ec2/
- AWS EBS Documentation: https://docs.aws.amazon.com/ebs/
- AWS KMS Documentation: https://docs.aws.amazon.com/kms/
- EC2 Pricing: https://aws.amazon.com/ec2/pricing/
- EBS Pricing: https://aws.amazon.com/ebs/pricing/
