---
tags:
  - aws
  - certification
  - iam
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[IAM]] · [[Global Infrastructure]]

# AWS Developer Associate Exam: Global Infrastructure & IAM

## Complete Study Guide with Critical Concepts

---

## PART 1: AWS GLOBAL INFRASTRUCTURE

### 1.1 Core Concepts & Hierarchy

#### **AWS Regions**

- **Definition**: Geographically isolated areas containing multiple Availability Zones
- **Current Count**: 30+ regions worldwide (constantly expanding)
- **Key Points**:
    - Each region is completely independent
    - Data doesn't replicate across regions by default
    - Services available vary by region
    - Compliance & data sovereignty requirements dictate region choice
    - Cost varies by region

#### **Availability Zones (AZs)**

- **Definition**: Multiple isolated data centers within a region
- **Key Points**:
    - Minimum 2 AZs per region, typically 3
    - Connected by low-latency, high-bandwidth network
    - Failure in one AZ doesn't affect others
    - **Exam Tip**: Design for high availability = deploy across AZs

#### **Edge Locations & CloudFront**

- **Definition**: Caching nodes for CloudFront CDN
- **Count**: 500+ edge locations globally
- **Purpose**: Cache content closer to users for better performance
- **Key Difference from Regions**: Edge locations are READ-ONLY caches, not full AWS regions

#### **Local Zones**

- Extension of AWS regions closer to large cities
- Used for ultra-low latency applications
- Less services available than regions

#### **Wavelength Zones**

- 5G mobile edge computing
- Low-latency applications for 5G devices

---

### 1.2 Exam-Critical Infrastructure Concepts

#### **Multi-AZ Deployments (CRITICAL FOR EXAM)**

```
BEST PRACTICE ARCHITECTURE:
- RDS Multi-AZ: Primary + Standby replica (automatic failover)
- ELB: Automatically distributes across AZs
- ASG: Span across multiple AZs for redundancy
- EBS: Single AZ (use snapshots for cross-AZ)
```

**Key Question Format on Exam:**

> "You need to design a highly available database. What should you do?" **Answer**: RDS Multi-AZ deployment with automatic failover

#### **Auto Scaling Across AZs**

- Distribute EC2 instances across multiple AZs
- Use Application Load Balancer to distribute traffic
- Ensures application survives AZ failure

#### **Data Replication Strategy**

```
Same Region, Different AZ:
- Low latency, fast replication
- No data transfer charges
- Survives datacenter failure

Different Regions:
- High latency, slower replication
- Data transfer charges apply
- Survives region-wide disaster
```

---

### 1.3 Service Availability & Deployment Patterns

#### **Which Services are Region-Specific?**

- EC2, RDS, DynamoDB, S3 buckets (regional), Lambda
- **Exception**: S3 is regional but globally accessible

#### **Which Services are Global?**

- **IAM** (global, no region selection needed)
- **CloudFront** (global CDN)
- **Route 53** (global DNS)
- **CloudWatch** (has global dashboards)
- **Budgets & Billing** (global view)

#### **Common Deployment Patterns**

```
Pattern 1: Active-Active Multi-Region
- Read/write in multiple regions
- Use DynamoDB Global Tables or Aurora Global Database
- Highest availability but most complex

Pattern 2: Active-Passive Multi-Region
- Primary region handles all traffic
- Secondary region ready for failover
- Disaster recovery approach
- Use Route 53 health checks for failover

Pattern 3: Single Region + Edge Caching
- All infrastructure in one region
- CloudFront caches content globally
- Cost-effective for read-heavy workloads
```

---

### 1.4 Exam Tips & Scenarios

**Scenario 1**: "Your application must survive complete AZ failure"

- **Answer**: Multi-AZ deployment (ELB + ASG across AZs)

**Scenario 2**: "You need disaster recovery with data in multiple regions"

- **Answer**: Cross-region replication (S3 cross-region replication, RDS read replicas)

**Scenario 3**: "Users globally complain about slow response times"

- **Answer**: CloudFront (CDN) to cache content at edge locations

**Scenario 4**: "You need to serve content with lowest latency"

- **Answer**: CloudFront + S3 (origin)

---

## PART 2: IAM (IDENTITY & ACCESS MANAGEMENT)

### 2.1 IAM Fundamentals (CRITICAL)

#### **What is IAM?**

- Service to manage access to AWS resources
- **Global service** (no region selection needed)
- **Free to use**
- Controls **authentication** (who you are) and **authorization** (what you can do)

#### **IAM Core Components**

|Component|Purpose|Key Points|
|---|---|---|
|**Users**|Individual people/applications|Unique credentials, shouldn't share|
|**Groups**|Collections of users|Easier permission management|
|**Roles**|Set of permissions (identity)|Assumed by users/services, temporary credentials|
|**Policies**|Permissions (document)|JSON format, attach to users/groups/roles|

#### **Authentication vs Authorization**

```
Authentication (Who are you?):
- IAM User credentials (Access Key ID + Secret Access Key)
- MFA (Multi-Factor Authentication)
- Temporary credentials from STS (Security Token Service)

Authorization (What can you do?):
- IAM Policies (JSON documents)
- Resource-based policies
- ACLs
```

---

### 2.2 Users & Credentials (EXAM CRITICAL)

#### **IAM Users**

- Represent people or applications
- Have permanent credentials (Access Key ID + Secret Access Key)
- **NEVER** share credentials
- Can have multiple access keys
- Best practice: One user per person

#### **Access Keys Structure**

```
Access Key ID: AKIAIOSFODNN7EXAMPLE
Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

Used for:
- AWS CLI
- SDK calls
- API calls
- NOT for logging into AWS Console (use username/password)
```

#### **Root User (CRITICAL)**

```
❌ NEVER use for daily work
❌ NEVER share root credentials
✅ Use only for:
  - Initial AWS account setup
  - Billing management
  - Close account
  - Change email/root credentials
✅ Enable MFA on root account immediately
```

#### **MFA (Multi-Factor Authentication)**

- **Virtual MFA device**: Google Authenticator, Authy
- **Security Key**: U2F key (most secure)
- **Hardware MFA**: Physical device
- **EXAM TIP**: MFA is for console login security, not API calls

---

### 2.3 Policies & Permissions (VERY CRITICAL FOR EXAM)

#### **IAM Policy Structure**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow" or "Deny",
      "Principal": "arn:aws:iam::123456789:user/username",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

**Components Explained:**

- **Effect**: Allow or Deny (Deny takes precedence)
- **Principal**: WHO (used in resource-based policies)
- **Action**: WHAT (s3:GetObject, ec2:*)
- **Resource**: ON WHAT (arn:aws:s3:::bucket/*)
- **Condition**: WHEN (IP, time, MFA status, etc.)

#### **Policy Types**

|Type|Use Case|Example|
|---|---|---|
|**Managed Policies**|Reusable across users/groups/roles|AmazonEC2FullAccess, AmazonS3ReadOnlyAccess|
|**Inline Policies**|One-off policies|Custom policy attached to single user|
|**AWS Managed**|Pre-made by AWS|Ready to use, maintained by AWS|
|**Customer Managed**|Custom policies|Created by you, reusable|

#### **Policy Evaluation Logic (EXAM CRITICAL)**

```
Default: DENY (implicit deny)
    ↓
Explicit ALLOW in identity-based policy?
    ↓ YES
Continue checking...
    ↓ NO
Result: DENY
    ↓
Explicit DENY in any policy?
    ↓ YES
Result: DENY (Deny overrides Allow)
    ↓ NO
Explicit ALLOW in resource-based policy?
    ↓ YES
Result: ALLOW
    ↓ NO
Result: DENY

KEY PRINCIPLE: An explicit DENY always wins
```

#### **Least Privilege Principle**

- **CRITICAL for exam**: Give minimum permissions needed
- **Bad**: `"Action": "*"`, `"Resource": "*"`
- **Good**: `"Action": "s3:GetObject"`, `"Resource": "arn:aws:s3:::my-bucket/public/*"`

---

### 2.4 Roles (VERY IMPORTANT)

#### **What are IAM Roles?**

- Identity with permissions
- Temporary credentials (STS tokens)
- Assumed by:
    - EC2 instances
    - Lambda functions
    - Cross-account users
    - Federated users
    - AWS services

#### **Why Use Roles Instead of Users?**

```
DON'T: Embed credentials in EC2 instance
❌ Credentials in config files (security risk)
❌ Hard to rotate credentials
❌ If compromised, manual cleanup needed

DO: Use IAM Roles
✅ Credentials automatically rotated by STS
✅ No manual credential management
✅ Can assume role from EC2 instance metadata
✅ Can be revoked instantly
```

#### **Trust Policy (AssumeRolePolicyDocument)**

Defines WHO can assume the role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### **Instance Profile**

- Container for EC2 to assume role
- EC2 automatically gets temporary credentials
- Don't need to manage keys

---

### 2.5 Groups (Exam Frequency: Medium)

#### **What are IAM Groups?**

- Collection of users with same permissions
- Users inherit group permissions
- Simplifies permission management

#### **Best Practices**

```
✅ Group by job function:
  - Developers group
  - Admins group
  - QA group

✅ Attach policies to groups, not individual users
```

#### **Key Points**

- Groups cannot be nested (can't add group to group)
- Users can belong to multiple groups
- No default groups

---

### 2.6 Cross-Account Access (Exam Important)

#### **Scenario**: Developer in Account A needs to access resources in Account B

**Solution: Assume Role in Another Account**

Account A (Dev Account):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::222222222:role/CrossAccountRole"
  }]
}
```

Account B (Prod Account) - Trust Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111:user/developer"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

---

### 2.7 Temporary Security Credentials (STS)

#### **Security Token Service (STS)**

- Issues temporary credentials
- Used by:
    - Roles
    - Federated users
    - Cross-account access
    - Web identity federation

#### **Credentials Returned**

```
- AccessKeyId (short-lived)
- SecretAccessKey (short-lived)
- SessionToken (unique to temporary credentials)
- Expiration (default 1 hour)
```

#### **Key APIs**

- `sts:AssumeRole` - Assume role (cross-account)
- `sts:GetCallerIdentity` - Get current identity
- `sts:GetSessionToken` - MFA-protected access
- `sts:AssumeRoleWithWebIdentity` - Web identity federation (Google, Facebook login)

---

### 2.8 Federated Identity & Web Identity

#### **Federated Identity (Third-party authentication)**

- Users login with external provider
- External provider returns temporary AWS credentials
- **Use cases**: Mobile apps, web apps

#### **SAML Federation** (Enterprise)

- Integration with corporate directory (Active Directory)
- Users login to corporate portal
- Automatically get AWS credentials

#### **Web Identity Federation** (Public)

- Users login with Google, Facebook, etc.
- Cognito exchanges external credentials for AWS credentials
- Lambda, mobile apps access AWS directly

#### **Amazon Cognito**

- Managed service for user authentication
- Issues temporary AWS credentials
- Used for mobile/web applications

---

### 2.9 Policy Simulator & Best Practices

#### **Policy Simulator**

- Test IAM policies before applying
- Available in IAM console
- Try calling service with specific credentials to see if allowed

#### **Best Practices for Exam**

```
1. LEAST PRIVILEGE
   - Only grant needed permissions
   - Use specific resource ARNs, not "*"

2. MFA FOR PRIVILEGED USERS
   - Require MFA for sensitive operations
   - Use Condition: "Bool": {"aws:MultiFactorAuthPresent": "true"}

3. USE ROLES, NOT LONG-TERM CREDENTIALS
   - For applications/EC2 instances
   - Temporary credentials are more secure

4. ROTATE CREDENTIALS REGULARLY
   - Access keys every 90 days
   - Use IAM Access Analyzer to find unused credentials

5. MONITOR & AUDIT
   - Enable CloudTrail (logs API calls)
   - Use IAM Access Analyzer (find unused permissions)
   - Review permissions regularly

6. USE GROUPS FOR PERMISSION MANAGEMENT
   - Easier to manage 100 users in 5 groups than 100 individual policies

7. NEVER HARDCODE CREDENTIALS
   - No credentials in code
   - Use roles for EC2, Lambda
   - Use Secrets Manager for database passwords
```

---

### 2.10 Common Exam Scenarios

#### **Scenario 1: EC2 Instance Needs S3 Access**

```
❌ WRONG: Add AWS credentials to EC2 config
✅ RIGHT: Attach IAM role to EC2 instance
         EC2 automatically gets temporary credentials
```

#### **Scenario 2: Developer Needs Limited S3 Access**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::dev-bucket/user-${aws:username}/*"
  }]
}
```

- Uses policy variable `${aws:username}` for user-specific prefixes

#### **Scenario 3: Application Needs Cross-Account Access**

```
Use STS AssumeRole with:
- Cross-account role in target account
- Trust policy allowing source account
- Temporary credentials returned
```

#### **Scenario 4: External User Needs AWS Access**

```
Use Web Identity Federation:
1. User logs in with external provider (Google, etc.)
2. Cognito exchanges credentials for AWS temporary credentials
3. User gets temporary AWS access
4. No long-term AWS credentials needed
```

#### **Scenario 5: Sensitive Operation Requires MFA**

```json
{
  "Effect": "Allow",
  "Action": "iam:DeleteUser",
  "Resource": "*",
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```

---

## EXAM QUICK REFERENCE

### Global Infrastructure Must-Knows

- [ ] Regions are completely independent
- [ ] Minimum 2 AZs per region
- [ ] Multi-AZ = High Availability
- [ ] CloudFront = Global CDN (Edge Locations)
- [ ] IAM/Route53/CloudFront = Global services
- [ ] Design for AZ failure (use ASG + ELB across AZs)

### IAM Must-Knows

- [ ] IAM is global
- [ ] Root user = Emergency only
- [ ] Temporary credentials better than long-term
- [ ] Roles > Users for applications
- [ ] Explicit DENY always wins
- [ ] Least privilege principle
- [ ] MFA for privileged users
- [ ] Don't hardcode credentials
- [ ] Use policy variables for fine-grained access
- [ ] CloudTrail for audit

### Dangerous Mistakes on Exam

- ❌ Forgetting MFA on root account
- ❌ Using root user for daily work
- ❌ Giving overly broad permissions (Action: "*")
- ❌ Hardcoding credentials in applications
- ❌ Not designing for AZ failure
- ❌ Assuming data replicates across regions automatically
- ❌ Using users instead of roles for EC2/Lambda
- ❌ Not understanding policy evaluation logic

---

## Practice Questions to Study

### Global Infrastructure

1. **Q**: Your application experiences high latency in Asia. What should you add? **A**: CloudFront with origin in closest region
    
2. **Q**: You need to survive complete regional failure. What's the design? **A**: Multi-region active-active with DynamoDB Global Tables
    
3. **Q**: Where should you place cache for lowest latency? **A**: CloudFront Edge Locations
    

### IAM

1. **Q**: EC2 instance needs to read from S3. Best approach? **A**: Attach IAM role to EC2 instance (Instance Profile)
    
2. **Q**: You have 500 developers. How to manage permissions efficiently? **A**: Create groups by role, assign policies to groups
    
3. **Q**: What always takes precedence in IAM? **A**: Explicit DENY
    
4. **Q**: How to give user access only to their own S3 folder? **A**: Use policy variable `${aws:username}` in resource ARN
    
5. **Q**: How to let another AWS account access your resources? **A**: Create role in your account with trust policy for other account
    
6. **Q**: External user needs AWS access. How without AWS credentials? **A**: Web Identity Federation via Cognito
    

---

## Additional Resources for Study

- AWS IAM Documentation: https://docs.aws.amazon.com/iam/
- AWS Global Infrastructure: https://aws.amazon.com/about-aws/global-infrastructure/
- IAM Best Practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- AWS Whitepapers on Multi-Region Architecture
