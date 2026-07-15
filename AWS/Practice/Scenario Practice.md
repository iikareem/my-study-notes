---
tags:
  - aws
  - certification
  - ec2
  - iam
  - kms
  - practice
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[Mixed Services Practice]] · [[Practice Hints]]

# AWS Developer Associate Exam Questions

## Real Scenario-Based Questions from Global Infrastructure, IAM, EC2, EBS & KMS

---

## QUESTIONS SECTION

### (Do NOT scroll to answers until you attempt!)

---

## GLOBAL INFRASTRUCTURE QUESTIONS

### Question 1: High Availability Design

A development team is designing a critical web application that must survive the failure of an entire Availability Zone. The application uses RDS database and serves content with CloudFront.

Which of the following architecture changes would BEST ensure the application remains available if one AZ fails?

A) Replicate the RDS database to another region using read replicas  
B) Deploy RDS in Multi-AZ configuration with automatic failover in the same region  
C) Place all EC2 instances in a single AZ and use CloudFront for redundancy  
D) Use a single RDS instance with frequent manual backups to another AZ

---

### Question 2: CloudFront vs Multi-Region

Your application has users distributed globally - 40% in Asia, 35% in Europe, 25% in North America. Users report slow response times when accessing static images and HTML content.

What is the MOST cost-effective solution to improve performance without requiring application changes?

A) Deploy your application in 3 regions and use Route 53 with failover routing  
B) Use CloudFront with S3 as the origin to cache content at edge locations  
C) Migrate database to a global service and replicate data across regions  
D) Increase EC2 instance size in the primary region to handle more requests

---

### Question 3: Region Selection for Compliance

Your company has strict data residency requirements - all customer data MUST remain within the European Union and cannot be replicated to other regions.

Which AWS service feature would you use to ensure compliance?

A) Enable CloudFront in EU regions only  
B) Set S3 bucket region to eu-west-1 and configure bucket policies to prevent replication  
C) Use RDS read replicas in the same region for backup  
D) Deploy all infrastructure to a single Availability Zone

---

### Question 4: AZ Failure Scenario

Your development team wants to design an application that automatically handles the failure of one Availability Zone without any manual intervention or data loss.

Which combination would BEST achieve this?

A) EC2 in single AZ + manual backup to another AZ  
B) Auto Scaling Group spanning multiple AZs + Load Balancer + RDS Multi-AZ  
C) All services in one AZ with CloudFront caching as redundancy  
D) Read replica database in another region with manual failover process

---

### Question 5: Global Service vs Regional Service

A developer is setting up an AWS account and needs to know which services require region selection during initial setup.

Which of the following is a GLOBAL service that does NOT require region selection?

A) EC2 instance creation  
B) RDS database creation  
C) IAM user creation  
D) S3 bucket creation

---

## IAM QUESTIONS

### Question 6: EC2 Credential Management (CRITICAL)

A company deploys a web application on an EC2 instance that needs to read files from an S3 bucket. A junior developer suggests embedding AWS access keys (Access Key ID and Secret Access Key) directly in the application code.

Why is this approach problematic, and what's the correct solution?

A) Access keys embedded in code cannot be rotated; use root user credentials instead  
B) Access keys expire after 24 hours; implement a credential refresh mechanism  
C) Keys are vulnerable if code is compromised; use IAM role with instance profile instead  
D) Access keys cannot be used from EC2; only temporary credentials work

---

### Question 7: Policy Evaluation Logic

A developer has the following IAM permissions:

- Identity-based policy: Allows s3:GetObject on any S3 bucket
- S3 bucket policy: Denies s3:GetObject from all principals

What happens when the developer tries to get an object?

A) Request is allowed because identity-based policy allows it  
B) Request is denied because explicit DENY in bucket policy takes precedence  
C) Request is allowed with a warning in CloudTrail  
D) Access depends on which policy was created last

---

### Question 8: IAM Group Management

A company has 200 developers who need similar S3 permissions. Currently, the same inline policy is attached to each developer's user account individually.

What's the MOST efficient way to manage these permissions going forward?

A) Create a Lambda function that automatically attaches inline policies to new users  
B) Create a single IAM group, attach a managed policy to it, and add developers to the group  
C) Use CloudFront to restrict S3 access to authorized users  
D) Create 200 customer managed policies, one for each developer

---

### Question 9: Cross-Account Access

Company A needs to grant a developer from Company B (different AWS account) permission to read data from an S3 bucket in Company A's account.

Which is the CORRECT approach?

A) Create an IAM user in Company A and share the password with Company B developer  
B) Add Company B's AWS account ID to S3 bucket ACL and allow List/Read actions  
C) Create an IAM role in Company A with S3 read permissions and trust relationship to Company B  
D) Grant root user permissions from Company B to Company A's S3 bucket

---

### Question 10: Root User Security

A developer just created a new AWS account and needs to configure it for production use.

What MUST be done first?

A) Create IAM users for all developers to use for daily work  
B) Enable Multi-Factor Authentication (MFA) on the root account  
C) Grant root user password to the lead developer for account management  
D) Delete the root user and create an administrative IAM user instead

---

### Question 11: Policy Variables and Dynamic Permissions

A company wants to restrict S3 bucket access so each developer can only access their own folder (e.g., /dev-alice/, /dev-bob/).

How would you implement this with minimal policy overhead?

A) Create individual inline policy for each developer specifying their folder  
B) Use a single policy with variable `${aws:username}` in the resource ARN  
C) Create a Lambda function to validate folder access per request  
D) Use CloudFront to restrict access to specific prefixes

---

### Question 12: Temporary Credentials

A mobile application needs to access AWS resources without storing permanent AWS credentials.

Which AWS service should you use?

A) Create IAM users with access keys and embed them in the mobile app  
B) Use Amazon Cognito for web identity federation to get temporary credentials  
C) Store AWS credentials in the mobile app using encryption  
D) Create a separate AWS account for each mobile user

---

### Question 13: MFA Requirement

A company wants to ensure that sensitive operations (like deleting database snapshots) require MFA authentication.

How would you enforce this in an IAM policy?

A) Add a condition checking `"aws:MultiFactorAuthPresent": "true"`  
B) Create an IAM group that requires MFA and add users to it  
C) Enable MFA on all EC2 instances  
D) Use Security Groups to restrict access based on MFA status

---

## EC2 QUESTIONS

### Question 14: Instance Lifecycle and Data Loss

A developer stops an EC2 instance that has an attached EBS volume and a mounted instance store.

What happens to the data on each storage type?

A) EBS data is lost, instance store data is retained  
B) EBS data is retained, instance store data is lost  
C) Both are lost  
D) Both are retained

---

### Question 15: Pricing Model Selection

A company runs a batch processing job that:

- Runs for exactly 2 hours each day
- Can tolerate being interrupted
- Must complete within the day
- Cost optimization is critical

What is the BEST pricing option?

A) On-Demand instances  
B) Reserved Instances (1-year commitment)  
C) Spot instances with on-demand as fallback  
D) Dedicated hosts

---

### Question 16: Elastic IP and Restart

A web server application requires a consistent IP address that remains the same even when the EC2 instance is stopped and restarted.

What should the developer use?

A) The default public IP provided by AWS  
B) Elastic IP (EIP) associated with the instance  
C) Security Group with port forwarding  
D) Application load balancer

---

### Question 17: Security Group Behavior

A developer creates a security group with the following inbound rule:

- Protocol: TCP
- Port: 443
- Source: 0.0.0.0/0

Which statement is CORRECT?

A) Only HTTPS traffic on port 443 is allowed in, but outbound traffic is blocked  
B) HTTPS traffic on port 443 is allowed in, and return traffic is automatically allowed out  
C) Traffic is allowed bidirectionally on all protocols  
D) Traffic is allowed only from localhost

---

### Question 18: Instance Type for Database Server

A company needs to run a MySQL database server that requires:

- High CPU performance
- Large amount of RAM
- Consistent network performance

Which instance family is BEST suited?

A) T3 (burstable general purpose)  
B) C5 (compute optimized)  
C) R5 (memory optimized)  
D) I3 (storage optimized)

---

### Question 19: Persistent Static IP

A company has an application running on EC2. The application needs to maintain the same public IP address across multiple stop/start cycles.

If the developer fails to allocate an Elastic IP, what will happen?

A) AWS will automatically assign the same public IP on restart  
B) AWS will assign a different public IP on restart (old one is released)  
C) The instance will have no public IP  
D) The instance will retain the old IP for 24 hours

---

### Question 20: High Availability with Multiple AZs

A critical application must survive failure of a single Availability Zone. Currently, all EC2 instances are in us-east-1a.

What architectural change is needed?

A) Deploy another instance in the same AZ for redundancy  
B) Use CloudFront to cache all application content  
C) Deploy instances across multiple AZs (us-east-1a, us-east-1b, us-east-1c) with a load balancer  
D) Deploy to another region entirely

---

### Question 21: Instance Metadata for Credential Retrieval

An EC2 instance with an attached IAM role needs to retrieve temporary credentials.

Where would the application find these credentials?

A) In the .aws/credentials file on the instance  
B) From the EC2 instance metadata service at http://169.254.169.254/  
C) In the IAM role policy document  
D) From an environment variable set by AWS

---

## EBS QUESTIONS

### Question 22: EBS Volume Encryption (VERY IMPORTANT)

A developer has an existing unencrypted EBS volume attached to an EC2 instance. The security team requires all data to be encrypted with KMS.

What is the CORRECT procedure?

A) Use the ModifyVolume API to enable encryption on the existing volume  
B) Create snapshot of current volume → Create encrypted volume from snapshot → Detach old → Attach new  
C) Delete the volume and create a new encrypted one (data loss acceptable for this scenario)  
D) Use AWS backup to copy the volume with encryption enabled

---

### Question 23: Volume Type Selection for Database

A MySQL production database is experiencing slow query response times. Performance monitoring shows low CPU usage but high disk I/O wait times.

The volume is currently gp2. What should be upgraded?

A) Instance type to C5 (more CPU)  
B) EBS volume to io1 or io2 (higher IOPS)  
C) Network interface to higher bandwidth  
D) Root volume size to larger capacity

---

### Question 24: Snapshot and Incremental Storage

A company creates daily snapshots of a 100GB EBS volume. The first snapshot takes 2 hours.

How long will subsequent snapshots take (assuming similar data changes)?

A) 2 hours (full copy each time)  
B) Much faster, only changed blocks are stored incrementally  
C) Same time because AWS must verify all data  
D) Cannot be determined without knowing change rate

---

### Question 25: IOPS vs Throughput Understanding

An application performs the following operations:

- Many small random database queries
- Each query reads 4KB of data
- Processes 5,000 queries per second

Is the bottleneck IOPS or Throughput?

A) IOPS (5,000 operations/second is critical)  
B) Throughput (5,000 × 4KB = 20MB/s)  
C) Network bandwidth  
D) Instance CPU

---

### Question 26: Big Data Processing Volume Type

A data engineering team runs Hadoop jobs that read large sequential files from EBS and write results back.

Which volume type provides the BEST price/performance?

A) gp3 (general purpose)  
B) io2 (high IOPS)  
C) st1 (throughput optimized)  
D) sc1 (cold storage)

---

### Question 27: Instance Store vs EBS

A developer is choosing between EBS and instance store for temporary cached data that:

- Must be available for the instance lifetime
- Can be recreated if lost
- Needs maximum performance

Which is BEST?

A) EBS gp3 for reliability and performance  
B) Instance store (ephemeral) for maximum speed  
C) EBS with snapshots for backup  
D) CloudFront cache

---

### Question 28: EBS Optimization

A company runs critical financial transaction processing on EC2 with io2 EBS volumes.

What should be enabled to guarantee dedicated throughput to EBS?

A) EBS acceleration  
B) EBS encryption  
C) EBS optimization (on the instance)  
D) EBS fast snapshots

---

### Question 29: Cross-AZ Data Replication

An application requires high availability and data durability across Availability Zones.

Using EBS alone, how can you replicate data from us-east-1a to us-east-1b?

A) EBS automatically replicates across AZs  
B) Use EBS snapshots and create volume in different AZ  
C) Attach same EBS volume to instances in multiple AZs  
D) Configure EBS mirroring in storage settings

---

## KMS QUESTIONS

### Question 30: AWS Managed vs Customer Managed Keys (CRITICAL)

A company is configuring S3 encryption. They need:

- Encryption with audit trail
- Full control over key rotation
- Specific audit requirements for compliance

Should they use AWS Managed or Customer Managed KMS keys?

A) AWS Managed keys (simpler, no cost)  
B) Customer Managed keys (full control, audit capability, compliance)  
C) No encryption (compliance doesn't require it)  
D) Application-level encryption instead of KMS

---

### Question 31: KMS Key Access Control

An EC2 instance with IAM role needs to decrypt data encrypted with a customer managed KMS key.

The EC2 instance role has IAM policy allowing "kms:Decrypt", but the operation still fails with "Access Denied".

What's the MOST likely cause?

A) The EC2 instance is in wrong AZ  
B) The KMS key policy doesn't grant the EC2 role permission  
C) The IAM policy is attached to wrong user  
D) KMS doesn't support EC2 access

---

### Question 32: KMS Key Rotation

A customer managed KMS key is configured with automatic annual rotation.

After 1 year, what happens to data encrypted with the old key?

A) Data becomes unreadable and must be re-encrypted  
B) Data remains readable with old key, new data uses new key (transparent)  
C) All old data is automatically re-encrypted with new key  
D) Rotation fails if old data exists

---

### Question 33: Encrypting Existing EBS Volume

A developer needs to encrypt an existing unencrypted EBS volume.

Which KMS operation is used during the snapshot → encrypted volume process?

A) kms:Encrypt (encrypt the EBS volume directly)  
B) kms:GenerateDataKey (generate key for snapshot)  
C) kms:CreateGrant (allow snapshot service to use key)  
D) Both A and B are used

---

### Question 34: Data Key Generation for Application

An application needs to encrypt 10MB of sensitive data before storing it in a database.

What KMS operation should be used?

A) kms:Encrypt (max 4KB plaintext)  
B) kms:GenerateDataKey (returns plaintext + encrypted key)  
C) kms:GenerateDataKeyWithoutPlaintext (encrypted key only)  
D) kms:CreateGrant (long-term key usage)

---

### Question 35: KMS and S3 Integration

A developer configures an S3 bucket with SSE-KMS encryption using a customer managed key.

What do the bucket owner and the developer need for access?

A) S3 bucket policy granting access only  
B) IAM policy granting s3:GetObject only  
C) Both S3 bucket IAM policy AND KMS key policy must grant permissions  
D) Root account credentials

---

### Question 36: Cross-Account KMS Access

Company A has a customer managed KMS key. Company B needs to decrypt data encrypted with this key.

What is required in Company A's KMS key policy?

A) Add Company B's AWS account number as principal  
B) Add Company B's root user ARN as principal  
C) Add Company B's IAM role ARN as principal with kms:Decrypt action  
D) Share KMS key material with Company B

---

### Question 37: HTTPS Enforcement for Encrypted Data

A company encrypts data in S3 with SSE-KMS. They want to ensure data is never transmitted unencrypted over HTTP.

What bucket policy condition should be added?

A) Condition: "IpAddress": {"aws:SourceIp": ["10.0.0.0/8"]}  
B) Condition: "Bool": {"aws:SecureTransport": "false"} → Effect: Deny  
C) Condition: "StringEquals": {"s3:encryption": "AES256"}  
D) Add Security Group rule blocking port 80

---

### Question 38: KMS Key Policy Syntax

A developer writes a KMS key policy to allow an EC2 role to decrypt:

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789:role/EC2Role"
  },
  "Action": "kms:Decrypt",
  "Resource": "*"
}
```

Is this policy correct?

A) Yes, it allows EC2Role to decrypt with any KMS key  
B) No, "Resource" should specify a key ARN, not "*"  
C) No, "Principal" should be a service principal  
D) No, only root account can create key policies

---

### Question 39: Secrets Manager vs KMS Direct Encryption

A developer needs to store database passwords securely in AWS.

What's the BEST approach?

A) Encrypt passwords using kms:Encrypt and store encrypted value in database  
B) Use AWS Secrets Manager (handles KMS encryption internally)  
C) Store passwords in plaintext in Systems Manager Parameter Store  
D) Hardcode passwords in application configuration

---

### Question 40: KMS Regional Limitation

A company has users in multiple regions accessing the same encrypted data in S3.

An error occurs: "KMS key not found in us-west-2 region"

What's happening?

A) KMS is global but the key was created in us-east-1 region  
B) The data was encrypted with a key that doesn't exist in us-west-2  
C) KMS doesn't support multi-region data access  
D) S3 bucket is in wrong region

---

---

---

---

---

## ANSWER KEY & EXPLANATIONS

### Global Infrastructure Answers

**Question 1: High Availability Design** **Correct Answer: B**

**Explanation:**

- A) Cross-region read replicas are for DR, not AZ failure - too slow for automatic failover
- **B) CORRECT** - RDS Multi-AZ provides automatic failover to standby replica in different AZ within same region (low latency)
- C) Incorrect - Single AZ is not HA; CloudFront doesn't provide database redundancy
- D) Incorrect - Manual backups don't provide automatic failover

**Key Learning:** For AZ failure within same region → Multi-AZ. For region failure → cross-region replication.

---

**Question 2: CloudFront vs Multi-Region** **Correct Answer: B**

**Explanation:**

- A) Multi-region with failover is expensive and adds complexity for static content
- **B) CORRECT** - CloudFront caches at 500+ edge locations globally. Content served from nearest edge = lowest latency. Most cost-effective for static content (images, HTML)
- C) Global databases are expensive and complex for static content
- D) Bigger instances don't help with geographical latency

**Key Learning:** For static content served globally → CloudFront. For application logic across regions → multi-region deployment.

---

**Question 3: Region Selection for Compliance** **Correct Answer: B**

**Explanation:**

- A) CloudFront caches globally, violating residency requirement
- **B) CORRECT** - Set bucket region to eu-west-1 and bucket policy denies cross-region replication (aws:SecureTransport condition, deny GetObject from non-EU IPs)
- C) Read replicas in same region don't prevent data leaving region
- D) Single AZ doesn't prevent region replication settings

**Key Learning:** Data residency is controlled at bucket/service level, not AZ level.

---

**Question 4: AZ Failure Scenario** **Correct Answer: B**

**Explanation:**

- A) Manual process doesn't meet "without manual intervention" requirement
- **B) CORRECT** - Auto Scaling Group spans AZs (instances in 1a, 1b, 1c), Load Balancer distributes traffic, RDS Multi-AZ provides database failover. Completely automatic with no data loss
- C) Single AZ fails if that AZ goes down
- D) Manual failover is not automatic

**Key Learning:** True HA = ASG + ELB + Multi-AZ database.

---

**Question 5: Global Service vs Regional Service** **Correct Answer: C**

**Explanation:**

- A) EC2 is regional (select region/AZ)
- B) RDS is regional (select region)
- **C) CORRECT** - IAM is global, no region selection needed. One IAM account spans all regions
- D) S3 is regional (bucket created in specific region)

**Key Learning:** IAM, CloudFront, Route 53 = Global. Everything else = Regional.

---

### IAM Answers

**Question 6: EC2 Credential Management** **Correct Answer: C**

**Explanation:**

- A) Incorrect reasoning - root credentials are worse, not better
- B) Incorrect - access keys don't expire by default
- **C) CORRECT** - Hardcoded credentials are security risk. If code is exposed (GitHub, etc.), attacker has AWS access. IAM role + instance profile provides temporary credentials automatically rotated by AWS
- D) Incorrect - access keys work fine from EC2

**Key Learning:** NEVER hardcode credentials. EC2 → Use IAM roles. Applications → Use roles. Humans → Use IAM users (for learning only).

---

**Question 7: Policy Evaluation Logic** **Correct Answer: B**

**Explanation:**

- A) Incorrect - DENY overrides ALLOW
- **B) CORRECT** - In IAM policy evaluation, explicit DENY always takes precedence. Even if identity-based allows, explicit DENY in bucket policy blocks access
- C) Incorrect - warnings don't override denial
- D) Incorrect - DENY always wins regardless of order

**Key Learning:** DENY > ALLOW. One explicit deny anywhere = Access denied.

---

**Question 8: IAM Group Management** **Correct Answer: B**

**Explanation:**

- A) Lambda automation is unnecessary for this task
- **B) CORRECT** - Create Developers group, attach managed policy (e.g., AmazonS3ReadOnlyAccess or custom managed policy), add 200 users to group. Much easier than 200 inline policies
- C) CloudFront is for CDN, not IAM management
- D) 200 customer managed policies is unmaintainable

**Key Learning:** Use groups for permission management. One policy × 200 users = much easier than 200 inline policies.

---

**Question 9: Cross-Account Access** **Correct Answer: C**

**Explanation:**

- A) Sharing passwords is security anti-pattern
- B) S3 ACLs are legacy, not recommended for cross-account
- **C) CORRECT** - Create role in Company A with trust policy allowing Company B account principal, attach S3 read policy to role. Company B developer assumes role in Company A, gets temporary credentials
- D) Sharing root access is extremely dangerous

**Key Learning:** Cross-account access = Assume role pattern with trust relationship.

I choose wrong answer
In AWS cross-account access, the resource-owning account creates an IAM role with permissions and trust policy, while the other account temporarily assumes that role using STS.

---

**Question 10: Root User Security** **Correct Answer: B**

**Explanation:**

- A) Correct but not FIRST priority - security comes before convenience
- **B) CORRECT** - First thing: enable MFA on root account. Root account should be protected (emergency-only access)
- C) Incorrect - never share root credentials
- D) Incorrect - root account cannot be deleted, only managed

**Key Learning:** Root user = Enable MFA immediately. Then create IAM users for daily work. Root = Emergency only.

---

**Question 11: Policy Variables and Dynamic Permissions** **Correct Answer: B**

**Explanation:**

- A) Scales poorly (need to update policy when developers change)
- **B) CORRECT** - Use policy variable ${aws:username}:
    
    ```json
    "Resource": "arn:aws:s3:::bucket/dev-${aws:username}/*"
    ```
    
    Each developer can only access their folder (/dev-alice/, /dev-bob/)
- C) Lambda for access control is overengineering
- D) CloudFront doesn't provide this level of granularity

**Key Learning:** Policy variables allow dynamic, scalable permission grants.

---

**Question 12: Temporary Credentials** **Correct Answer: B**

**Explanation:**

- A) Hardcoding credentials in mobile app is security risk
- **B) CORRECT** - Amazon Cognito provides web identity federation. Mobile user logs in with Google/Facebook → Cognito exchanges credentials for temporary AWS credentials (STS tokens) → No permanent credentials stored
- C) Encrypting credentials in app is still risky
- D) Creating AWS account per mobile user is not scalable

**Key Learning:** Mobile apps → Cognito for temporary credentials. Never hardcode AWS credentials in apps.

---

**Question 13: MFA Requirement** **Correct Answer: A**

**Explanation:**

- **A) CORRECT** - Add condition to policy:
    
    ```json
    "Condition": {  "Bool": {"aws:MultiFactorAuthPresent": "true"}}
    ```
    
    User can only perform action if MFA authenticated
- B) Groups don't enforce MFA; policies do
- C) EC2 instances don't need MFA for data deletion
- D) Security groups don't control MFA

**Key Learning:** Use policy conditions to enforce MFA for sensitive operations.

I choose wrong

---

### EC2 Answers

**Question 14: Instance Lifecycle and Data Loss** **Correct Answer: B**

**Explanation:**

- A) Opposite of correct answer
- **B) CORRECT** - EBS is persistent (survives stop/start/reboot). Instance store (ephemeral) is lost on stop/terminate. When instance stops, instance store data is gone but EBS remains
- C) Incorrect - EBS is persistent
- D) Incorrect - instance store is ephemeral

**Key Learning:** EBS = persistent. Instance Store = ephemeral. Only use instance store for temporary data (cache, staging).

---

**Question 15: Pricing Model Selection** **Correct Answer: C**

**Explanation:**

- A) On-Demand is most expensive, not cost-optimized
- B) Reserved instances for 2 hours/day = 730 hours/year = expensive commitment
- **C) CORRECT** - Spot instances are 70-90% cheaper. 2-hour job can tolerate interruption (can retry if interrupted). Use spot for cost optimization with on-demand fallback for reliability
- D) Dedicated hosts are most expensive

**Key Learning:** Spot = cheapest for fault-tolerant workloads. Reserved = commitment for steady workloads. On-Demand = flexibility for unpredictable.

---

**Question 16: Elastic IP and Restart** **Correct Answer: B**

**Explanation:**

- A) Public IP changes on stop/start (not persistent)
- **B) CORRECT** - Elastic IP is static, remains associated through stop/start/reboot cycles
- C) Security groups are firewalls, not IP management
- D) ALB is for load balancing across multiple instances

**Key Learning:** Need persistent IP → Elastic IP. Temporary IP OK → default public IP.

---

**Question 17: Security Group Behavior** **Correct Answer: B**

**Explanation:**

- A) Incorrect - outbound traffic is allowed by default (open)
- **B) CORRECT** - Security groups are stateful. If inbound allowed, return traffic automatically allowed outbound. Allows HTTPS in, responses automatically allowed out
- C) All protocols not allowed (only HTTPS on 443)
- D) Source 0.0.0.0/0 means anywhere, not localhost

**Key Learning:** Security groups are stateful (inbound/outbound matched). Default = allow all outbound, deny all inbound.

---

**Question 18: Instance Type for Database Server** **Correct Answer: C**

**Explanation:**

- A) T3 is burstable, good for small workloads, not database
- B) C5 is compute optimized (high CPU, less memory) - database needs more RAM than CPU
- **C) CORRECT** - R5 (memory optimized) provides large memory pools with good CPU. Perfect for databases that need lots of RAM
- D) I3 is storage optimized (high I/O), not general database

**Key Learning:** T=burstable/small. C=compute-heavy. R=memory-heavy (databases). I=storage-heavy/NoSQL.

---

**Question 19: Persistent Static IP** **Correct Answer: B**

**Explanation:**

- A) Incorrect - AWS doesn't guarantee same public IP
- **B) CORRECT** - Without Elastic IP, AWS releases old public IP on stop and assigns new random public IP on restart. Application loses connectivity
- C) Incorrect - instance gets new public IP if in public subnet
- D) Incorrect - no 24-hour persistence of public IP

**Key Learning:** Public IP is ephemeral (changes on stop). EIP is persistent (static, reassignable).

---

**Question 20: High Availability with Multiple AZs** **Correct Answer: C**

**Explanation:**

- A) Multiple instances in same AZ don't protect against AZ failure
- B) CloudFront caches but doesn't provide HA for application
- **C) CORRECT** - Deploy instances across multiple AZs (1a, 1b, 1c) with load balancer distributing traffic. If one AZ fails, others handle traffic
- D) Different region is overengineering for AZ failure (more expensive)

**Key Learning:** AZ failure resilience = ASG + ELB across multiple AZs. Regional resilience = multi-region.

---

**Question 21: Instance Metadata for Credential Retrieval** **Correct Answer: B**

**Explanation:**

- A) .aws/credentials file is on user's local machine, not EC2
- **B) CORRECT** - EC2 instance metadata service at http://169.254.169.254/latest/meta-data/iam/security-credentials/{role-name}/ returns temporary credentials from attached IAM role
- C) Policy document doesn't contain credentials
- D) Credentials could be in environment variable, but metadata service is official method

**Key Learning:** EC2 gets credentials from metadata service. Applications should use SDK which automatically calls this endpoint.

---

### EBS Answers

**Question 22: EBS Volume Encryption** **Correct Answer: B**

**Explanation:**

- A) ModifyVolume cannot encrypt existing unencrypted volume
- **B) CORRECT** - Only way to encrypt existing volume:
    1. Create snapshot of unencrypted volume
    2. Create new encrypted volume from snapshot (specify KMS key)
    3. Detach old unencrypted volume
    4. Attach new encrypted volume
- C) Data loss not acceptable
- D) AWS backup is not the standard method

**Key Learning:** Cannot encrypt existing EBS volume directly. Must snapshot → create encrypted volume → attach.

---

**Question 23: Volume Type Selection for Database** **Correct Answer: B**

**Explanation:**

- A) CPU usage is low, so CPU not bottleneck
- **B) CORRECT** - High I/O wait time = disk I/O bottleneck. gp2 max IOPS = 16,000. Database needs io1/io2 (up to 64,000 IOPS) for better random access performance
- C) Network bandwidth unlikely bottleneck for single instance
- D) Larger volume doesn't increase IOPS

**Key Learning:** High I/O wait = upgrade to io1/io2. High CPU wait = upgrade instance type.

---

**Question 24: Snapshot and Incremental Storage** **Correct Answer: B**

**Explanation:**

- A) Subsequent snapshots are NOT full copies
- **B) CORRECT** - First snapshot captures all 100GB. Second snapshot only stores changed blocks (incremental). Much faster (minutes instead of hours). Saves storage space significantly
- C) No verification delay
- D) Change rate determines how much faster (but always faster than full copy)

**Key Learning:** Snapshots are incremental. Only changed blocks stored. First snapshot = slow, subsequent = fast.

---

**Question 25: IOPS vs Throughput Understanding** **Correct Answer: A**

**Explanation:**

- **A) CORRECT** - 5,000 operations/second = IOPS bottleneck. Database needs 5,000 IOPS minimum. Each operation is small (4KB), so throughput = 5,000 × 4KB = 20MB/s, which is well within any volume's throughput
- B) Incomplete answer - IOPS is the limiting factor here
- C) EC2 instance network can handle this
- D) CPU not mentioned as bottleneck

**Key Learning:** Random small operations = IOPS bottleneck. Sequential large reads = Throughput bottleneck.

---

**Question 26: Big Data Processing Volume Type** **Correct Answer: C**

**Explanation:**

- A) gp3 is okay but not optimized for throughput
- B) io2 is expensive for sequential workloads (unnecessary IOPS)
- **C) CORRECT** - st1 (throughput optimized) is designed for big data: 500 MB/s throughput, lower cost than gp3 for sequential access
- D) sc1 is for infrequent access (archive), too slow for Hadoop

**Key Learning:** Hadoop/data warehouse = st1. OLTP databases = io1/io2. General = gp3.

---

**Question 27: Instance Store vs EBS** **Correct Answer: B**

**Explanation:**

- A) EBS is reliable but slower than instance store
- **B) CORRECT** - Instance store (ephemeral NVMe SSD) provides maximum performance for temporary cache that can be recreated. Data loss on instance stop is acceptable for cache (can rebuild from source)
- C) EBS with snapshots is overkill for temporary data
- D) CloudFront is for HTTP content, not application cache

**Key Learning:** Instance store = fastest but ephemeral. Use for cache, temp data, staging. EBS = persistent but slower.

---

**Question 28: EBS Optimization** **Correct Answer: C**

**Explanation:**

- A) EBS acceleration is not standard term
- B) Encryption doesn't affect throughput optimization
- **C) CORRECT** - EBS optimization on instance provides dedicated bandwidth between instance and EBS. Prevents EBS traffic competing with network traffic. Critical for high-performance databases
- D) Fast snapshots is not a feature

**Key Learning:** Production databases with io2 → Enable EBS-optimized instances for guaranteed throughput.


EBS optimization provides dedicated bandwidth between EC2 and EBS volumes for more consistent and higher storage performance.

---

**Question 29: Cross-AZ Data Replication** **Correct Answer: B**

**Explanation:**

- A) EBS only replicates within single AZ automatically
- **B) CORRECT** - EBS cannot be directly attached across AZs. To replicate: Create snapshot (stored in S3 across AZs) → Create volume in us-east-1b from snapshot → Attach to instance in 1b
- C) Same volume cannot attach to multiple instances
- D) No EBS mirroring feature

**Key Learning:** EBS is single-AZ. For HA across AZs, use Multi-AZ database or snapshot → new volume in different AZ.

---

### KMS Answers

**Question 30: AWS Managed vs Customer Managed Keys** **Correct Answer: B**

**Explanation:**

- A) AWS Managed keys don't provide audit trail or control
- **B) CORRECT** - Customer Managed keys provide:
    - Full control over key rotation (automatic or manual)
    - Key policy for fine-grained access control
    - CloudTrail audit of all KMS operations
    - Meets compliance requirements
- C) Compliance typically requires encryption
- D) Application-level encryption is more complex

**Key Learning:** Compliance/audit requirements → Customer Managed KMS. General encryption → AWS Managed.

---

**Question 31: KMS Key Access Control** **Correct Answer: B**

**Explanation:**

- A) AZ doesn't matter for KMS (KMS is regional)
- **B) CORRECT** - Access to KMS key requires BOTH:
    1. IAM policy on user/role allowing kms:Decrypt ✓
    2. KMS key policy allowing the principal ✗ If "Access Denied" despite IAM policy, key policy is missing the EC2 role as principal
- C) Role policy is attached correctly
- D) KMS fully supports EC2 access via roles

**Key Learning:** KMS access = IAM policy + Key policy (BOTH required). Missing either = Access denied.

---

**Question 32: KMS Key Rotation** **Correct Answer: B**

**Explanation:**

- A) Incorrect - old data remains readable
- **B) CORRECT** - Automatic key rotation:
    - New backing key created annually (same key ID)
    - Old backing key retained for decryption
    - Old data decrypts with old key (transparent)
    - New data encrypts with new key
    - No user action required, no re-encryption needed
- C) Automatic rotation doesn't re-encrypt old data
- D) Rotation succeeds regardless of existing data

**Key Learning:** Automatic rotation is transparent. Old data stays readable with old key, new data uses new key.

---

**Question 33: Encrypting Existing EBS Volume** **Correct Answer: D**

**Explanation:**

- A) Encrypt is used but not the primary operation
- B) GenerateDataKey is not used in this workflow
- **C) CreateGrant might be used but not primary
- **D) CORRECT** - Process uses both:
    1. Create snapshot (data is read from unencrypted volume)
    2. When creating encrypted volume from snapshot, KMS GenerateDataKey creates data encryption key
    3. Data from snapshot encrypted with the generated key
    4. Encrypted data stored in new volume

Actually, looking more carefully: The snapshot service itself doesn't use explicit KMS calls on behalf of user. The user (through AWS console/CLI) creates the encrypted volume, which triggers KMS operations.

**REVISED: The correct answer is actually B** - When creating encrypted volume from snapshot, AWS uses kms:GenerateDataKey (or similar) to create the data key for encrypting the copied data.

Let me reconsider: In real AWS flow:

1. Snapshot of unencrypted volume (no KMS needed)
2. CreateVolume with Encrypted=true and KmsKeyId specified (this requires kms:Decrypt or kms:GenerateDataKey depending on implementation)

The answer should be **B - kms:GenerateDataKey** for encrypting the snapshot data into encrypted volume.

**Key Learning:** Encrypting snapshot data uses KMS GenerateDataKey to encrypt the volume.

### Summary

- **`kms:Encrypt`** means: _"Here is my data, KMS. You encrypt it for me inside your walls."_ (Fails because the data is too big).
    
- **`kms:GenerateDataKey`** means: _"KMS, give me a temporary key so I can encrypt the data myself out here."_ (This is what actually happens).

---

**Question 34: Data Key Generation for Application** **Correct Answer: B**

**Explanation:**

- A) kms:Encrypt max plaintext = 4KB (10MB exceeds limit)
- **B) CORRECT** - kms:GenerateDataKey:
    - Returns plaintext data key (use to encrypt data)
    - Returns encrypted data key (store with encrypted data)
    - Workflow: decrypt encrypted key with KMS → get plaintext → decrypt data
    - No plaintext size limit
- C) GenerateDataKeyWithoutPlaintext used when key generated for others
- D) CreateGrant is for long-term permissions, not data encryption

**Key Learning:**

- Data ≤ 4KB → kms:Encrypt
- Data > 4KB → kms:GenerateDataKey
- Data for others → kms:GenerateDataKeyWithoutPlaintext

---

**Question 35: KMS and S3 Integration** **Correct Answer: C**

**Explanation:**

- A) S3 bucket policy alone is insufficient
- B) IAM policy alone is insufficient
- **C) CORRECT** - Both required:
    1. S3 bucket IAM policy (or public access blocked, principal can access)
    2. IAM policy granting s3:GetObject on bucket
    3. KMS key policy granting kms:Decrypt to user Missing any layer = Access denied
- D) Don't use root credentials

**Key Learning:** SSE-KMS requires access at multiple layers: S3 bucket + IAM + KMS key.

---

**Question 36: Cross-Account KMS Access** **Correct Answer: C**

**Explanation:**

- A) Account number alone insufficient (no action specified)
- B) Root user is dangerous, should use specific role
- **C) CORRECT** - KMS key policy must include:
    
    ```json
    "Principal": {  "AWS": "arn:aws:iam::CompanyB:role/SpecificRole"},"Action": ["kms:Decrypt", "kms:DescribeKey"],"Resource": "*"
    ```
    
    Specific role ARN from Company B with explicit actions
- D) Never share key material

**Key Learning:** Cross-account KMS = Key policy + Trust relationship. Add specific role ARN, not account or root.

---

**Question 37: HTTPS Enforcement for Encrypted Data** **Correct Answer: B**

**Explanation:**

- A) IP restrictions don't enforce HTTPS
- **B) CORRECT** - Deny policy condition:
    
    ```json
    {  "Effect": "Deny",  "Principal": "*",  "Action": "s3:*",  "Resource": "arn:aws:s3:::bucket/*",  "Condition": {    "Bool": {"aws:SecureTransport": "false"}  }}
    ```
    
    This denies any request not using HTTPS/TLS
- C) Encryption setting doesn't enforce transport security
- D) Security groups don't control S3 access (S3 is not VPC resource)

**Key Learning:** Enforce HTTPS with `"aws:SecureTransport": "false" → Deny` policy condition.

---

**Question 38: KMS Key Policy Syntax** **Correct Answer: B**

**Explanation:**

- A) Incorrect - "Resource": "*" is problematic
- **B) CORRECT** - While "Resource": "*" is allowed, best practice is to specify key ARN:
    
    ```json
    "Resource": "arn:aws:kms:us-east-1:123456789:key/12345-key-id"
    ```
    
    However, if this exact policy works in exam context, "Resource": "*" in key policy is acceptable (key policy is key-specific anyway). But specific ARN is better practice
- C) EC2 is service principal, but role principal is correct for EC2 instances
- D) Key policy can be created by key owner/admin

**Actually, upon reflection: "Resource": "*" in key policy is acceptable because key policy is inherently scoped to the key. The answer is technically A), but B) is a better practice.**

For exam purposes: **Most likely answer is B) - specify resource ARN is more secure/correct**.

**Key Learning:** Key policies should specify key ARN, not "*".

---

**Question 39: Secrets Manager vs KMS Direct Encryption** **Correct Answer: B**

**Explanation:**

- A) Manual KMS encryption works but complex, no rotation
- **B) CORRECT** - AWS Secrets Manager:
    - Stores encrypted secrets (uses KMS internally)
    - Automatic rotation of database credentials
    - Audit trail (CloudTrail)
    - Built-in for common databases
    - Much simpler than manual KMS encryption
- C) Plaintext storage is insecure
- D) Hardcoded passwords are most dangerous

**Key Learning:** Secrets Manager >> Manual KMS for secrets. Use for database passwords, API keys, certificates.

---

**Question 40: KMS Regional Limitation** **Correct Answer: B**

**Explanation:**

- **B) CORRECT** - KMS is regional. If key was created in us-east-1, it doesn't exist in us-west-2. When accessing encrypted S3 object from us-west-2, decrypt fails because key not available in that region
    - Solution: Create key in us-west-2 OR replicate key from us-east-1 to us-west-2
- A) KMS is not global despite being regional
- C) Multi-region access works if key replicated
- D) S3 bucket region independent of KMS region

**Key Learning:** KMS is regional. Multi-region access requires key replication or separate keys per region.

---

---

## SUMMARY STATISTICS

**By Service:**

- Global Infrastructure: 5 questions
- IAM: 8 questions
- EC2: 7 questions
- EBS: 8 questions
- KMS: 12 questions
- **Total: 40 real exam-style questions**

**By Difficulty:**

- Easy (definitions): ~8 questions
- Medium (scenarios): ~20 questions
- Hard (complex scenarios): ~12 questions

**Most Tested Concepts:**

1. IAM policy evaluation logic (explicit DENY) - Question 7
2. EC2 to role for credentials - Question 6
3. EBS volume encryption procedure - Question 22
4. KMS key policy + IAM policy - Question 31
5. RDS Multi-AZ for AZ failure - Question 1

---

## STUDY TIPS

1. **Favorite Exam Pattern**: "What happens if X fails?" → Usually answer is multi-AZ or replication
    
2. **Red Flags on Exam**:
    ;
    - "Hardcoded credentials" → Role is answer
    - "Cannot decrypt" → Check both IAM + key policy
    - "Existing unencrypted EBS" → Snapshot → encrypted volume
    - "Sensitive operation" → Add MFA condition
    - "Global distribution" → CloudFront
3. **Confidence Builders**:
    
    - Memorize: Explicit DENY always wins
    - Memorize: EC2 gets credentials from metadata service
    - Memorize: EBS single-AZ → snapshot for cross-AZ
    - Memorize: KMS access = IAM + Key policy
    - Memorize: gp3 is default choice
4. **Common Wrong Answers**:
    
    - Using public IP thinking it persists (use EIP)
    - Thinking instance store persists (it doesn't)
    - Assuming AWS Managed keys have full audit (use Customer Managed)
    - Forgetting key policy when only IAM policy specified
    - Assuming CloudFront alone provides HA for compute

---

## PRACTICE STRATEGY

**Day 1-2**: Answer all 40 questions, check answers only **Day 3**: Review all INCORRECT answers, understand why **Day 4**: Answer same 40 again, should score 80%+ **Day 5**: Do in 90 minutes (simulate exam time pressure) **Before Exam**: Review incorrectly answered questions

Good luck on your exam! 🚀
