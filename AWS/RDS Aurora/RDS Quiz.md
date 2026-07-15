---
tags:
  - aws
  - certification
  - postgresql
  - practice
  - rds
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[RDS Aurora]]

# AWS Certified Developer – Associate (DVA-C02)

## RDS Deep Dive — 25 Questions

> **Instructions:** Choose the **best** answer for each question. Answers and explanations are at the bottom. Do not scroll ahead until you have finished.

---

**Question 1**

A developer is building a web application that uses Amazon RDS MySQL. During peak hours, the application experiences high read latency. The write volume is normal. What is the MOST effective solution?

A. Upgrade the RDS instance to a larger instance type  
B. Enable Multi-AZ deployment on the RDS instance  
C. Create one or more RDS Read Replicas and direct read traffic to them  
D. Enable RDS automated backups with a 1-hour retention window

---

**Question 2**

A developer's application connects to an RDS database using hardcoded credentials in the application config file. The security team requires credentials to be rotated every 30 days without downtime. What is the BEST solution?

A. Update the config file manually every 30 days and redeploy the application  
B. Store credentials in AWS Secrets Manager and configure automatic rotation; update the application to retrieve credentials from Secrets Manager at runtime  
C. Store credentials in an S3 bucket with versioning enabled and rotate by uploading a new file  
D. Use IAM database authentication and rotate IAM user access keys every 30 days

---

**Question 3**

An application running on EC2 uses IAM database authentication to connect to an RDS MySQL instance. Which of the following is required for this to work?

A. The RDS instance must be in the same VPC as the EC2 instance  
B. The EC2 instance must have an IAM role with the `rds-db:connect` permission, and IAM authentication must be enabled on the RDS instance  
C. SSL must be disabled on the RDS connection for IAM authentication to function  
D. The database user must have the `SUPER` privilege in MySQL

---

**Question 4**

A company's RDS PostgreSQL database is in a private subnet. A developer needs to allow an application in a different VPC to connect to it. What is the correct approach?

A. Make the RDS instance publicly accessible and whitelist the application's IP  
B. Set up VPC peering between the two VPCs and update the security group to allow traffic from the peered VPC's CIDR  
C. Create a Read Replica in the application's VPC  
D. Enable RDS Multi-AZ and use the standby endpoint from the application's VPC

---

**Question 5**

A developer wants to minimize RDS downtime during a planned maintenance window for a production database. The application must continue serving traffic during the maintenance. Which feature should be enabled?

A. Automated backups  
B. Multi-AZ deployment — failover to the standby instance handles patching with minimal disruption  
C. Read Replicas — promote a replica to primary during maintenance  
D. Enable deletion protection to prevent unplanned maintenance

---

**Question 6**

An application inserts a record into an RDS table and immediately queries it. Occasionally the query returns no results. The application uses one RDS writer endpoint and one Read Replica. What is causing this behavior?

A. The RDS instance has a caching layer that delays visibility  
B. There is replication lag between the primary and the Read Replica; reads should be routed to the primary writer endpoint for strongly consistent data  
C. Automated backups are interfering with read operations on the replica  
D. The Read Replica is in a different Region and DNS propagation is causing the delay

---

**Question 7**

A developer needs to restore an RDS instance to a specific point 2 hours ago due to accidental data deletion. Automated backups are enabled with a 7-day retention period. What should the developer do?

A. Use the most recent snapshot and manually replay application logs  
B. Use the RDS Point-in-Time Restore (PITR) feature to restore the instance to 2 hours ago  
C. Contact AWS Support to recover the deleted data  
D. Restore from a Read Replica which maintains a copy of all historical data

---

**Question 8**

A developer is storing database credentials in AWS Secrets Manager. The Lambda function that retrieves credentials is invoked thousands of times per second. The developer is concerned about Secrets Manager API throttling. What is the BEST approach?

A. Store credentials in DynamoDB and cache them in Lambda memory  
B. Use the AWS Secrets Manager caching client library to cache the secret in the Lambda execution environment and refresh it only when it expires or rotates  
C. Increase the Lambda function timeout to give Secrets Manager time to respond  
D. Call Secrets Manager on every invocation — it is designed for high-frequency access

---

**Question 9**

A company runs an RDS MySQL database. The database is encrypted with a KMS customer-managed key (CMK). The team wants to share an encrypted snapshot with another AWS account. What must they do?

A. The snapshot cannot be shared across accounts when encrypted with a CMK  
B. Share the encrypted snapshot with the target account AND grant the target account permission to use the CMK used to encrypt the snapshot  
C. Re-encrypt the snapshot with an AWS-managed key before sharing  
D. Enable cross-Region replication before sharing the snapshot

---

**Question 10**

A developer is designing a connection strategy for a serverless application using many short-lived Lambda functions that connect to RDS PostgreSQL. The number of concurrent connections frequently exceeds the database's `max_connections` limit. What should the developer do?

A. Increase the `max_connections` parameter in the RDS parameter group  
B. Use Amazon RDS Proxy to pool and reuse database connections, reducing the total number of open connections  
C. Switch the database engine to Aurora Serverless  
D. Reduce Lambda concurrency using reserved concurrency to limit connections

---

**Question 11**

A developer needs to copy an RDS snapshot from `us-east-1` to `eu-west-1` for disaster recovery. The snapshot is encrypted. Which statement is TRUE?

A. Encrypted snapshots cannot be copied across Regions  
B. The snapshot must first be decrypted before it can be copied to another Region  
C. When copying an encrypted snapshot across Regions, a KMS key in the destination Region must be specified  
D. Cross-Region copy automatically uses the same KMS key from the source Region

---

**Question 12**

An Aurora MySQL cluster is deployed with one writer and two readers. A developer wants connections from the application to automatically be distributed across the readers. Which endpoint should the application use for read operations?

A. The cluster endpoint (writer endpoint)  
B. The instance endpoint of each reader, alternating in code  
C. The reader endpoint, which load-balances connections across all Aurora Replicas  
D. The custom endpoint configured for multi-region reads

---

**Question 13**

A developer's RDS application logs show intermittent `Too many connections` errors during load spikes. The RDS instance is a `db.t3.medium`. Which TWO actions will resolve this? (Choose TWO)

A. Enable Multi-AZ deployment  
B. Use RDS Proxy to pool database connections  
C. Upgrade to a larger instance type with higher `max_connections`  
D. Enable automatic backups  
E. Create a Read Replica

---

**Question 14**

A developer needs to enforce that all connections to an RDS MySQL instance use SSL/TLS. What is the correct way to enforce this at the database level?

A. Set the `rds.force_ssl` parameter to `1` in the RDS parameter group  
B. Enable Multi-AZ — SSL is automatically enforced on standby connections  
C. Configure a security group rule to block non-SSL traffic on port 3306  
D. Use IAM database authentication, which always uses SSL

---

**Question 15**

An application uses Amazon Aurora Serverless v2. The developer notices that the cluster scales up slowly during sudden traffic spikes, causing latency. What configuration change should the developer make?

A. Switch to provisioned Aurora with Read Replicas  
B. Increase the minimum Aurora Capacity Unit (ACU) setting so the cluster starts from a higher baseline  
C. Enable Multi-AZ on the Aurora Serverless cluster  
D. Enable Aurora Global Database to distribute load

---

**Question 16**

A developer accidently drops a table in an RDS PostgreSQL database. Automated backups are enabled. There is no Read Replica. Point-in-Time Restore is initiated. Which statement about the restored instance is CORRECT?

A. The restore overwrites the existing RDS instance  
B. Point-in-Time Restore always creates a NEW RDS instance; the application must be updated to point to the new endpoint  
C. The restore automatically modifies the original instance's connection string  
D. PITR restores only the dropped table, not the entire database

---

**Question 17**

A developer is using RDS MySQL and wants to capture all data changes (INSERT, UPDATE, DELETE) in real time to feed a downstream analytics pipeline. Which approach is recommended on RDS?

A. Enable RDS automated backups and parse the backup files  
B. Enable binary logging on the RDS instance and use AWS Database Migration Service (DMS) with Change Data Capture (CDC) to stream changes  
C. Use RDS event notifications via SNS to capture data changes  
D. Create a Lambda trigger on the RDS table using native MySQL triggers that call Lambda

---

**Question 18**

A developer is connecting to RDS from a Lambda function inside a VPC. The Lambda function times out when attempting to connect. The RDS instance is in the same VPC. What should the developer check FIRST?

A. Whether the Lambda function's execution role has the `rds:Connect` IAM permission  
B. Whether the RDS security group has an inbound rule allowing traffic on the database port from the Lambda function's security group  
C. Whether RDS Multi-AZ is enabled  
D. Whether the Lambda function has enough memory allocated

---

**Question 19**

A company uses Amazon Aurora Global Database spanning `us-east-1` (primary) and `eu-west-1` (secondary). A disaster causes the primary region to fail. What action should the developer take to restore write capability?

A. Reboot the Aurora primary cluster from the AWS console in `us-east-1`  
B. Perform a managed failover (or detach and promote) the secondary cluster in `eu-west-1` to become the new primary  
C. Enable Multi-AZ on the `eu-west-1` cluster to automatically promote it  
D. Restore from the most recent automated backup in `eu-west-1`

---

**Question 20**

A developer needs to migrate data from an on-premises Oracle database to Amazon RDS PostgreSQL with minimal downtime. Which AWS service is designed for this use case?

A. AWS DataSync  
B. AWS Snowball  
C. AWS Database Migration Service (DMS) with ongoing replication enabled  
D. AWS Glue

---

**Question 21**

An application connects to RDS using the writer endpoint of an Aurora cluster. After a failover event, the application cannot reconnect for several minutes. What should the developer change to reduce reconnection time?

A. Increase the application's connection timeout to 10 minutes  
B. Implement connection retry logic with exponential backoff and ensure the application re-resolves the DNS endpoint after a failure (does not cache the DNS IP)  
C. Switch from Aurora to RDS MySQL to avoid failover delays  
D. Use the instance endpoint instead of the cluster endpoint

---

**Question 22**

A developer wants to create a copy of a production RDS database for testing without impacting the production workload and without waiting for a full data copy. What is the FASTEST option available for Aurora?

A. Create a Read Replica and promote it to a standalone instance  
B. Restore from an automated backup to a new instance  
C. Use Aurora Fast Clone — it creates a copy-on-write clone instantly  
D. Use RDS Snapshot export to S3 and restore from there

---

**Question 23**

An application needs to write to an RDS database. The security team requires that the application never store static database passwords. Which solution meets this requirement?

A. Store the password in an environment variable in the Lambda function configuration  
B. Use IAM database authentication — the application generates a temporary authentication token using IAM credentials and connects without a static password  
C. Store the password in AWS Systems Manager Parameter Store as a plaintext string  
D. Embed the password in the application binary and obfuscate it with Base64 encoding

---

**Question 24**

A developer creates an RDS parameter group to change the `max_connections` value. After associating the new parameter group with the RDS instance, the change has not taken effect. What is the most likely reason?

A. Parameter groups can only be associated during instance creation  
B. The `max_connections` parameter is a static parameter that requires an instance reboot to take effect  
C. The parameter group was created in the wrong AWS Region  
D. RDS does not support modifying `max_connections` — it is calculated automatically

---

**Question 25**

A developer is building a multi-tenant SaaS application on RDS PostgreSQL. Each tenant's data must be isolated. The developer wants to use a single RDS instance to minimize costs. Which approach provides the BEST isolation with a single instance?

A. Use a separate database per tenant within the same RDS instance and enforce access via RDS security groups  
B. Use a separate schema per tenant and create a dedicated database user per tenant with access scoped only to their schema  
C. Store all tenant data in the same tables with a `tenant_id` column and rely on application-level filtering  
D. Enable RDS Multi-AZ and dedicate each AZ to a different tenant

---

---

## ✅ Answer Key

| #   | Answer   | Explanation                                                                                                                                                                         |
| --- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **C**    | Read Replicas offload read traffic from the primary instance. Multi-AZ is for HA/failover, not read scaling.                                                                        |
| 2   | **B**    | Secrets Manager is purpose-built for credentials with automatic rotation and runtime retrieval — no redeployment needed.                                                            |
| 3   | **B**    | IAM DB authentication requires `rds-db:connect` on the execution role and the feature enabled on the RDS instance. SSL is also required but not the key distinguishing factor here. |
| 4   | **B**    | VPC peering + updated security group rules is the correct network-level solution for cross-VPC private RDS access.                                                                  |
| 5   | **B**    | Multi-AZ patches the standby first, then fails over with minimal (~60 second) downtime, keeping the application running.                                                            |
| 6   | **B**    | Read Replicas have eventual consistency due to asynchronous replication lag. Write-then-immediate-read must use the primary endpoint.                                               |
| 7   | **B**    | PITR allows restoring to any second within the retention window using automated backups + transaction logs.                                                                         |
| 8   | **B**    | The Secrets Manager caching client (available for Python, Java, .NET) caches secrets in memory and only calls the API on rotation or TTL expiry — dramatically reducing API calls.  |
| 9   | **B**    | Cross-account encrypted snapshot sharing requires both snapshot sharing AND granting the target account `kms:Decrypt` / `kms:CreateGrant` on the CMK.                               |
| 10  | **B**    | RDS Proxy is specifically designed to pool connections for Lambda-to-RDS patterns, solving the connection exhaustion problem.                                                       |
| 11  | **C**    | Cross-Region snapshot copy of an encrypted snapshot requires specifying a KMS key in the destination Region — KMS keys are Region-specific.                                         |
| 12  | **C**    | Aurora's reader endpoint load-balances across all read replicas automatically.                                                                                                      |
| 13  | **B, C** | RDS Proxy pools connections (reduces count); a larger instance type has higher `max_connections`. Multi-AZ and backups don't help with connection limits.                           |
| 14  | **A**    | Setting `rds.force_ssl=1` in the parameter group forces all MySQL connections to use SSL at the engine level.                                                                       |
| 15  | **B**    | Raising the minimum ACU ensures Aurora Serverless v2 is already at a higher capacity baseline, reducing scale-up latency during spikes.                                             |
| 16  | **B**    | PITR always restores to a **new** RDS instance with a new endpoint. The original instance is untouched. The application connection string must be updated.                          |
| 17  | **B**    | DMS with CDC reads the MySQL binary log and streams changes in near-real-time — the standard AWS approach for RDS change data capture.                                              |
| 18  | **B**    | The most common cause of Lambda-to-RDS timeout in the same VPC is a missing inbound rule on the RDS security group for the Lambda security group.                                   |
| 19  | **B**    | Aurora Global Database failover requires manually detaching and promoting the secondary cluster — it does not happen automatically like Multi-AZ.                                   |
| 20  | **C**    | AWS DMS with ongoing replication (CDC) enables live migration with minimal downtime by continuously replicating changes until cutover.                                              |
| 21  | **B**    | Retry logic + DNS re-resolution is critical — Aurora cluster endpoints are DNS-based, and caching the IP causes reconnection failures after failover.                               |
| 22  | **C**    | Aurora Fast Clone creates a copy-on-write clone in seconds, regardless of database size — no full data copy is made at clone time.                                                  |
| 23  | **B**    | IAM database authentication generates a short-lived token (15-minute TTL) signed by IAM credentials — no static password is stored anywhere.                                        |
| 24  | **B**    | RDS parameters are either **dynamic** (apply immediately) or **static** (require reboot). `max_connections` on some engines is static and needs a reboot.                           |
| 25  | **B**    | Separate schemas + dedicated database users with schema-scoped permissions provide database-level isolation while sharing one RDS instance, balancing cost and security.            |
