---
tags:
  - api-gateway
  - aws
  - certification
  - ec2
  - iam
  - kms
  - lambda
  - network
  - practice
  - security
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[Scenario Practice]] · [[Practice Hints]]

# AWS Certified Developer – Associate (DVA-C02)

## Practice Exam — 40 Questions

> **Instructions:** Choose the **best** answer for each question. Answers are listed at the bottom of this document. Do not scroll ahead until you have attempted all questions.

---

**Question 1**

A developer is building a serverless application. The function needs to process records from a stream and must retry failed records without losing them. The function should not process the same record more than once on success. Which configuration should the developer use?

A. Configure the function with an SQS queue as the event source and set `MaximumRetryAttempts` to 0  
B. Configure the function with a Kinesis Data Stream as the event source and enable a destination for failed-record routing  
C. Configure the function with an SNS topic as the event source and use dead-letter queues  
D. Configure the function with an S3 event notification and use S3 versioning for replay

---

**Question 2**

A company stores sensitive customer data in a database. A developer needs to ensure all data at rest is encrypted and that the company controls the key rotation schedule. The developer wants the least operational overhead. Which solution meets these requirements?

A. Enable server-side encryption using AWS-managed keys (SSE-AWS)  
B. Enable server-side encryption using customer-managed keys (SSE-KMS) with automatic rotation enabled in AWS KMS  
C. Implement client-side encryption with keys stored in AWS Secrets Manager  
D. Use SSE-S3 and schedule manual key rotation every 90 days x

---

**Question 3**

A developer is designing a RESTful API that must throttle requests per user and cache responses for 5 minutes. The API backend is hosted on EC2 instances. Which combination of features should the developer configure?

A. Enable API Gateway usage plans with API keys; set cache TTL to 300 seconds  
B. Use an Application Load Balancer with sticky sessions; configure CloudFront with a 300-second TTL  
C. Enable API Gateway resource policies; use ElastiCache in front of EC2  
D. Configure API Gateway stage-level caching only, without usage plans

---

**Question 4**

An application running on EC2 instances reads configuration values from environment variables set at launch time. The security team requires that database passwords never be stored in plaintext. What is the recommended approach?

A. Store passwords as EC2 user data and retrieve them at runtime using the instance metadata service  
B. Store passwords in AWS Systems Manager Parameter Store as SecureString parameters and retrieve them at application startup  
C. Hardcode the passwords in the application binary and encrypt the binary with KMS  
D. Store passwords in an S3 bucket with server-side encryption and grant EC2 instances read access

---

**Question 5**

A developer creates a Lambda function that must access an RDS database inside a private VPC subnet. The function cannot reach the internet and must also call the AWS SSM API. What should the developer do?

A. Place the Lambda function inside the VPC and create a VPC endpoint for SSM  
B. Place the Lambda function outside the VPC; connect to RDS via a public endpoint  
C. Add a NAT Gateway to the private subnet and configure the Lambda function's security group  x
D. Enable VPC peering between the Lambda execution environment and the VPC

---

**Question 6**

A DynamoDB table is experiencing throttling on writes during peak hours. The table uses provisioned capacity mode with 500 WCUs. The traffic pattern is unpredictable. What is the MOST cost-effective solution?

A. Increase provisioned WCUs to 5,000 permanently  
B. Enable DynamoDB Auto Scaling with target utilization of 70%  
C. Switch the table to on-demand capacity mode  
D. Implement SQS to buffer writes and keep the current WCU setting

---

**Question 7**

A company uses IAM roles to grant EC2 instances access to an S3 bucket. A developer notices that instances in a different AWS account also need access to the same bucket. What is the correct approach?

A. Create an IAM user in the bucket-owner account and share the access keys with the other account's team  
B. Add a bucket policy that grants access to the IAM role ARN from the external account  
C. Enable S3 Transfer Acceleration and configure the external account's CIDR range  
D. Use VPC peering and grant the peer VPC's security group read access to the bucket

---

**Question 8**

A Lambda function is invoked synchronously by API Gateway. The function sometimes takes more than 30 seconds to complete, causing timeout errors. The business logic cannot be optimized further. What should the developer do to resolve this?

A. Increase the Lambda timeout to 15 minutes in the function configuration  
B. Refactor the API to use an asynchronous pattern: API Gateway → SQS → Lambda, and poll for results  
C. Deploy the function on EC2 with a larger instance type  
D. Increase the API Gateway integration timeout to 60 seconds

---

**Question 9**

A developer needs to give a third-party auditor temporary read-only access to specific AWS resources without creating permanent IAM credentials. What is the best approach?

A. Create a new IAM user, attach a read-only policy, and delete the user after the audit  
B. Create a cross-account IAM role with a read-only policy and provide the auditor with a link to assume the role  
C. Generate a pre-signed S3 URL for each resource the auditor needs  
D. Share the root account credentials with the auditor and revoke after the audit

---

**Question 10**

A developer deploys a new version of a Lambda function. Immediately after deployment, a small percentage of production traffic should be routed to the new version while the rest continues to use the stable version. Which feature supports this?

A. Lambda layers  
B. Lambda aliases with weighted routing  
C. Lambda environment variables  
D. Lambda reserved concurrency

---

**Question 11**

An application stores session data in a DynamoDB table. A developer wants to ensure that a write operation only succeeds if a specific attribute value has not changed since the last read. Which DynamoDB feature should the developer use?

A. DynamoDB Streams  
B. Conditional expressions  
C. Atomic counters  
D. Batch write operations

---

**Question 12**

A developer is setting up a VPC for a new application. The application tier must communicate with the database tier, but the database tier must never be directly reachable from the internet. What VPC design achieves this?

A. Place both tiers in a public subnet; use Security Groups to restrict access  
B. Place the application tier in a public subnet and the database tier in a private subnet; use a NAT Gateway for outbound database updates  
C. Place both tiers in a private subnet and use an internet gateway for all traffic  
D. Place the database in a public subnet but disable DNS resolution

---

**Question 13**

A developer needs to call AWS APIs from code running on an EC2 instance without embedding long-term credentials. What is the recommended approach?

A. Store access keys in the `~/.aws/credentials` file on the instance  
B. Attach an IAM role to the EC2 instance and retrieve credentials from the instance metadata service  
C. Use environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set via user data  
D. Embed credentials in the application source code and rotate them monthly

---

**Question 14**

A developer configures an API Gateway REST API with Lambda proxy integration. The Lambda function returns the following object:

```json
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{\"message\": \"success\"}"
}
```

Callers receive a 502 error instead. What is the most likely cause?

A. The Lambda function execution role does not have permission to invoke API Gateway  
B. The `body` field must be an object, not a stringified JSON string  
C. API Gateway requires the `isBase64Encoded` field in every response  
D. The Lambda function is returning a response, but the integration type is misconfigured as HTTP instead of Lambda proxy

---

**Question 15**

A developer wants to scan a DynamoDB table but only retrieve items where the `status` attribute equals `"active"` and the `age` attribute is greater than 30. Which DynamoDB operation and parameters should be used?

A. `Query` with a `KeyConditionExpression`  
B. `Scan` with a `FilterExpression`  
C. `GetItem` with a `ProjectionExpression`  
D. `Query` with a `FilterExpression` on the partition key

---

**Question 16**

An application deployed across three AWS Regions needs a single global endpoint with automatic failover and low latency routing for TCP traffic on port 443. Which AWS service meets this requirement?

A. Amazon CloudFront  
B. AWS Global Accelerator  
C. Amazon Route 53 with latency-based routing  
D. Elastic Load Balancing with cross-zone load balancing

---

**Question 17**

A developer is encrypting data with a KMS symmetric key. The application encrypts and decrypts millions of small messages per second. Direct KMS API calls are too slow and costly. What approach should the developer use?

A. Use the KMS `GenerateDataKey` API to obtain a plaintext data key, encrypt data locally, and discard the plaintext key after use  
B. Cache the KMS CMK ARN in memory and call KMS for every encryption operation  
C. Switch to an asymmetric KMS key pair for faster encryption  
D. Store the KMS key material in DynamoDB for direct retrieval

---

**Question 18**

A developer is designing the IAM strategy for a new service. The service has three components: an API layer, a processing layer, and a storage layer. Each component needs different AWS permissions. What is the best practice?

A. Create one IAM user with all required permissions and share credentials across all components  
B. Create a single IAM role with a union of all permissions and attach it to each component  
C. Create a separate IAM role for each component with only the permissions it needs (least privilege)  
D. Use the root account for all components to avoid permission issues

---

**Question 19**

A developer notices that a Lambda function is being throttled because it exceeds the account's concurrency limit. Other critical Lambda functions in the same region are also affected. What should the developer do?

A. Deploy the throttled function in a different AWS Region  
B. Set reserved concurrency on the throttled function to cap its usage and protect other functions  
C. Increase the Lambda function's memory to reduce execution duration  
D. Switch the invocation type from asynchronous to synchronous

---

**Question 20**

A DynamoDB table has `UserID` (partition key) and `Timestamp` (sort key). A developer needs to retrieve all items for a specific `UserID` where `Timestamp` is between two values. Which API call should be used?

A. `Scan` with a `FilterExpression` on `UserID` and `Timestamp`  
B. `Query` with `KeyConditionExpression` specifying `UserID` and a range condition on `Timestamp`  
C. `GetItem` with a composite key specifying both `UserID` and a timestamp range  
D. `BatchGetItem` with a list of all timestamps

---

**Question 21**

A developer is building a multi-tier application. The web tier on EC2 must communicate with an RDS database but must not be exposed to the internet. The web tier itself must accept HTTPS traffic. Which architecture is correct?

A. Web tier in a private subnet behind an internet-facing ALB; RDS in a private subnet with no public access  
B. Web tier and RDS both in a public subnet; use Security Groups to restrict RDS access  
C. Web tier in a public subnet with a direct connection to RDS in the same subnet  
D. Web tier behind a NAT Gateway; RDS in a public subnet

---

**Question 22**

An API Gateway API is publicly accessible. The developer needs to allow only a specific set of IP addresses to call the API. Which approach should the developer use?

A. Configure a WAF Web ACL with an IP set condition and attach it to the API Gateway stage  
B. Use an API Gateway usage plan to restrict IP addresses  
C. Modify the Lambda function to validate the source IP from the event object  
D. Enable API Gateway mutual TLS (mTLS) authentication

---

**Question 23**

A developer needs to ensure that items in a DynamoDB table are automatically deleted after 24 hours. What is the most efficient solution?

A. Write a Lambda function triggered every hour to scan and delete expired items  
B. Enable DynamoDB Time to Live (TTL) and set the expiry attribute value to the current epoch time plus 86400 seconds  
C. Configure a DynamoDB Stream and process deletions via Lambda  
D. Use a DynamoDB conditional write to reject items older than 24 hours

---

**Question 24**

A developer is troubleshooting an EC2 instance that cannot reach the internet. The instance is in a public subnet with an assigned public IP. What should the developer check FIRST?

A. Whether the instance has an IAM role attached  
B. Whether the VPC has an Internet Gateway and the subnet's route table includes a route to it  
C. Whether the instance is using enhanced networking  
D. Whether CloudTrail logging is enabled in the VPC

---

**Question 25**

A developer wants to grant a Lambda function permission to read from an encrypted DynamoDB table using a KMS customer-managed key. What permissions are required?

A. `kms:Decrypt` on the KMS key and `dynamodb:GetItem` on the table  
B. `kms:GenerateDataKey` on the KMS key and `dynamodb:PutItem` on the table  
C. `kms:Encrypt` on the KMS key and `dynamodb:Scan` on the table  
D. Only `dynamodb:GetItem` on the table; KMS permissions are not required for reads

---

**Question 26 **
Wrong for me

A company uses API Gateway with a custom domain name. The developer needs to route v1 API traffic to one Lambda function and v2 API traffic to another. How should the developer configure this?

A. Create two API Gateway APIs and use base path mappings (`/v1` and `/v2`) on the custom domain  
B. Create one API with two Lambda functions configured in the same stage  
C. Use Lambda aliases to differentiate between v1 and v2 in a single API  
D. Create two custom domain names, one per version

---

**Question 27**

WRONG

A Lambda function is failing intermittently with a `TooManyRequestsException`. The function is invoked asynchronously. Which built-in mechanism will automatically retry the invocation?

A. AWS automatically retries asynchronous Lambda invocations up to 2 times with delays between attempts  
B. AWS automatically retries synchronous Lambda invocations up to 3 times  
C. The caller must implement retry logic; Lambda does not retry asynchronous invocations  
D. Lambda retries only if a dead-letter queue is configured

---

**Question 28**

A developer needs to store a secret API key so that it is automatically rotated every 30 days and can be accessed securely by a Lambda function. Which service should be used?

A. AWS Systems Manager Parameter Store (Standard tier)  
B. AWS Secrets Manager with automatic rotation configured  
C. AWS KMS with an imported key scheduled for deletion every 30 days  
D. Amazon S3 with versioning and lifecycle rules

---

**Question 29**

A developer is using the AWS SDK and notices that the SDK is making requests to the EC2 metadata endpoint `169.254.169.254` to obtain credentials. The application is not running on EC2. Why might this happen?

A. The SDK defaults to the instance metadata endpoint when no other credential provider resolves credentials first  
B. The SDK always uses the metadata endpoint regardless of configuration  
C. This occurs only when the application is running inside a Docker container on EC2  
D. The SDK is incorrectly configured and should be reinstalled

---

**Question 30**

A developer wants to ensure that two DynamoDB write operations either both succeed or both fail — similar to a transaction. Which DynamoDB feature supports this?

A. DynamoDB Streams with Lambda for compensation logic  
B. `TransactWriteItems` API  
C. Batch write with error handling  
D. Conditional writes on each individual item

---

**Question 31**

A developer needs to expose a WebSocket API for a real-time chat application. The backend logic runs in Lambda. Which AWS service natively supports WebSocket APIs with Lambda integration?

A. Application Load Balancer  
B. Amazon CloudFront  
C. Amazon API Gateway (WebSocket API)  
D. AWS AppSync

---

**Question 32**

An EC2 instance in a private subnet needs to download software packages from the internet without being directly reachable from the internet. What is the correct solution?

A. Assign a public IP to the instance and open port 80 in the security group  
B. Deploy a NAT Gateway in a public subnet and update the private subnet route table to route `0.0.0.0/0` to the NAT Gateway  
C. Create a VPN connection from the private subnet to the internet  
D. Enable VPC Flow Logs and use Route 53 for DNS resolution

---

**Question 33**

A developer is creating a KMS key policy. The policy must allow the key to be used for encryption and decryption by an EC2 role named `AppRole`, while ensuring the account root can administer the key. Which statement is correct?

A. KMS key policies are not required if IAM policies already grant access  
B. The key policy must explicitly grant `kms:Encrypt` and `kms:Decrypt` to `AppRole`; without an explicit key policy statement, IAM policies alone are insufficient  
C. Attaching the `AmazonEC2FullAccess` policy to `AppRole` automatically grants KMS access  
D. Only the root user can use KMS keys; role-based access is not supported

---

**Question 34**

A developer is building an application that needs to query DynamoDB items by an attribute that is NOT the primary key (e.g., `email`). Queries must be fast and consistent. What should the developer do?

A. Perform a `Scan` with a `FilterExpression` on `email`  
B. Create a Global Secondary Index (GSI) with `email` as the partition key  
C. Use a `BatchGetItem` with a list of all known emails  
D. Store a duplicate table with `email` as the primary key and keep them in sync manually

---

**Question 35**

A developer is configuring IAM for a new application. The application runs on Lambda and must read from an S3 bucket and write logs to CloudWatch. What is the MINIMUM set of permissions required?

A. `AmazonS3FullAccess`, `CloudWatchFullAccess`  
B. `s3:GetObject` on the specific bucket, `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`  
C. `AWSLambdaFullAccess` — this includes all required permissions  
D. `s3:*`, `logs:*` — wildcard actions to avoid permission errors

---

**Question 36**

A company is deploying its application across multiple AWS Regions for disaster recovery. Users should always be routed to the nearest healthy Region. Which combination of services provides this?

A. Route 53 with latency-based routing and health checks + EC2 in multiple Regions  
B. CloudFront with a single-origin pointing to one Region  
C. An Application Load Balancer with cross-zone load balancing  
D. Elastic Beanstalk with a multi-AZ configuration in one Region

---

**Question 37**

A developer enables DynamoDB Streams on a table. A Lambda function processes stream records. The developer notices that some records are processed multiple times. What is the most likely cause and solution?

A. Lambda does not support DynamoDB Streams; use Kinesis instead  
B. Lambda functions reading from streams have at-least-once delivery semantics; the function must be idempotent to handle duplicate records  
C. Duplicate processing occurs because multiple Lambda functions are attached to the same stream; remove the extra functions  
D. DynamoDB Streams only guarantee exactly-once delivery if `BisectBatchOnFunctionError` is enabled

---

**Question 38**

A developer deploys a Lambda function inside a VPC. The function must call an external HTTPS endpoint on the public internet. After deployment, all external calls time out. What is the MOST likely cause?

A. Lambda functions inside a VPC cannot make HTTPS calls  
B. The function's security group does not allow outbound HTTPS traffic  
C. The VPC lacks a NAT Gateway, and the private subnet has no route to the internet  
D. The Lambda execution role lacks the `ec2:DescribeNetworkInterfaces` permission

---

**Question 39**

A developer creates a REST API in API Gateway. The API should return a custom error message when the Lambda integration returns an error. What is the correct way to achieve this?

A. Configure a Lambda Authorizer to intercept errors before they reach the client  
B. Use Gateway Responses to customize error messages for integration errors  
C. Add a try/catch block in the Lambda function and return a 200 status with an error body  
D. Enable detailed CloudWatch logging and configure an alarm to notify clients

---

**Question 40**

A developer needs to allow users to upload files directly to S3 from a browser without routing the upload through the application server. The upload must be authenticated and expire after 15 minutes. Which mechanism should be used?

A. Configure an S3 bucket policy to allow public uploads  
B. Generate a pre-signed URL using the IAM role's credentials and return it to the client; set expiration to 900 seconds  
C. Use S3 Transfer Acceleration with API Gateway as a proxy  
D. Enable CORS on the S3 bucket and provide the bucket URL directly to the browser

---

---

## ✅ Answer Key

| #   | Answer | Explanation                                                                                                                                                                                                    |
| --- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **B**  | Kinesis event sources with Lambda support bisect-on-error and failure destinations to route failed records without reprocessing successful ones.                                                               |
| 2   | **B**  | SSE-KMS with a customer-managed key (CMK) gives you control over rotation schedules while AWS manages the infrastructure.                                                                                      |
| 3   | **A**  | API Gateway usage plans + API keys enable per-client throttling; stage caching with a 300s TTL handles response caching.                                                                                       |
| 4   | **B**  | SSM Parameter Store SecureString parameters are encrypted at rest and retrieved at runtime — no plaintext storage.                                                                                             |
| 5   | **A**  | Lambda inside the VPC can reach private RDS; a VPC endpoint for SSM avoids the need for internet access to reach the SSM API.                                                                                  |
| 6   | **C**  | On-demand mode handles unpredictable traffic without over-provisioning and is more cost-effective than permanently high provisioned WCUs.                                                                      |
| 7   | **B**  | S3 bucket policies support cross-account access by specifying IAM principal ARNs from other accounts.                                                                                                          |
| 8   | **B**  | **API Gateway has a maximum 29-second integration timeout. Async pattern (SQS + Lambda + polling) bypasses this constraint.**                                                                                  |
| 9   | **B**  | IAM cross-account roles with temporary credentials via STS AssumeRole are the recommended way to grant temporary access without permanent credentials.                                                         |
| 10  | **B**  | Lambda aliases support weighted traffic shifting, sending a defined percentage of traffic to a new version.                                                                                                    |
| 11  | **B**  | DynamoDB conditional expressions allow optimistic locking — the write fails if the attribute has changed since the last read.                                                                                  |
| 12  | **B**  | Public subnet for the app tier (internet-facing), private subnet for the database with no route from the internet.                                                                                             |
| 13  | **B**  | IAM roles attached to EC2 instances provide rotating temporary credentials via the instance metadata service (IMDS) — no static keys needed.                                                                   |
| 14  | **D**  | With Lambda proxy integration the response format must be valid (statusCode, headers, body as string). A 502 indicates the response format is malformed or the integration type is wrong.                      |
| 15  | **B**  | `Scan` with a `FilterExpression` works when there's no key-based access pattern; `Query` requires a known partition key value.                                                                                 |
| 16  | **B**  | AWS Global Accelerator provides a static anycast IP with TCP/UDP support and automatic failover across Regions. CloudFront is HTTP/HTTPS only.                                                                 |
| 17  | **A**  | Envelope encryption: generate a data key, encrypt data locally, store the encrypted key. This minimizes KMS API calls.                                                                                         |
| 18  | **C**  | Least-privilege IAM roles per component limits blast radius and follows AWS security best practices.                                                                                                           |
| 19  | **B**  | Reserved concurrency limits a function's maximum concurrency, protecting other functions in the account from being starved.                                                                                    |
| 20  | **B**  | `Query` with a `KeyConditionExpression` using the partition key (`UserID`) and a `BETWEEN` condition on the sort key (`Timestamp`) is the most efficient approach.                                             |
| 21  | **A**  | Internet-facing ALB in a public subnet handles HTTPS; web tier EC2 and RDS in private subnets are not directly reachable from the internet.                                                                    |
| 22  | **A**  | AWS WAF IP set conditions attached to an API Gateway stage are the standard way to allowlist/blocklist IPs.                                                                                                    |
| 23  | **B**  | TTL is a native DynamoDB feature that automatically deletes items when the epoch time attribute value is in the past — no extra code needed.                                                                   |
| 24  | **B**  | An Internet Gateway must be attached to the VPC and the subnet's route table must have a `0.0.0.0/0` route pointing to it.                                                                                     |
| 25  | **A**  | Reading encrypted DynamoDB data requires `dynamodb:GetItem` (or read equivalent) on the table AND `kms:Decrypt` on the CMK used to encrypt it.                                                                 |
| 26  | **A**  | Base path mappings on a custom domain allow routing `/v1` and `/v2` to separate API Gateway APIs or stages.                                                                                                    |
| 27  | **A**  | AWS Lambda retries failed asynchronous invocations automatically up to 2 additional times (3 total attempts).                                                                                                  |
| 28  | **B**  | Secrets Manager is designed for secret storage with built-in automatic rotation — Parameter Store Standard does not support automatic rotation.                                                                |
| 29  | **A**  | The AWS SDK credential provider chain tries several sources in order; the instance metadata endpoint is a fallback that is reached when earlier providers (env vars, shared credentials file) fail to resolve. |
| 30  | **B**  | `TransactWriteItems` provides ACID transactions across multiple DynamoDB items — all succeed or all fail.                                                                                                      |
| 31  | **C**  | API Gateway natively supports WebSocket APIs with Lambda backend integration, managing connection state.                                                                                                       |
| 32  | **B**  | A NAT Gateway in a public subnet allows instances in private subnets to initiate outbound internet connections while remaining unreachable from the internet.                                                  |
| 33  | **B**  | KMS key policies must explicitly grant usage permissions; IAM policies alone cannot grant access to a KMS key unless the key policy also allows it.                                                            |
| 34  | **B**  | A GSI with `email` as the partition key enables efficient `Query` operations on a non-primary-key attribute.                                                                                                   |
| 35  | **B**  | Least-privilege means scoping S3 access to `s3:GetObject` on the specific bucket and CloudWatch Logs permissions to the three required actions.                                                                |
| 36  | **A**  | Route 53 latency-based routing + health checks directs users to the lowest-latency healthy Region automatically.                                                                                               |
| 37  | **B**  | DynamoDB Streams with Lambda guarantee at-least-once delivery. Idempotent processing logic (e.g., checking if a record was already processed) is required to handle duplicates safely.                         |
| 38  | **C**  | Lambda functions in a VPC need a NAT Gateway in a public subnet and a route in the private subnet's route table to reach the internet.                                                                         |
| 39  | **B**  | Gateway Responses in API Gateway let you customize the response body, headers, and status code for integration errors and other error types.                                                                   |
| 40  | **B**  | S3 pre-signed URLs generated server-side allow authenticated, time-limited direct uploads from browsers without exposing credentials or routing through the server.                                            |

Q1- Don't worry if this feels a bit overwhelming right now. AWS serverless architectures have specific patterns for handling data streams and errors. Let's break down exactly _why_ B is the right choice, and give you a general strategy to ace these kinds of architectural questions.

## Why B is the Correct Choice

To understand why B works, we need to look at how AWS Lambda interacts with **Kinesis Data Streams** and how it handles errors.

### 1. The Core Requirement: Stream Processing

The question specifically mentions processing records from a **stream**. Kinesis is AWS's premier real-time data streaming service.

### 2. The Problem: "The Line Leader" Dilemma

By default, when a Lambda function reads from a Kinesis stream, it processes records in batches. If a single record in that batch fails, Lambda will retry the _entire batch_ over and over again. Because it retries continuously, the stream gets blocked (this is called a "poison pill" record).

### 3. The Solution: Failed-Record Routing (Bisect and Drop)

To prevent a broken record from blocking the stream forever, AWS allows you to configure a **Destination for failed-record routing** (sometimes called On-Failure Destinations).

When configured:

- If a record keeps failing after a certain number of retries, Lambda will pluck that specific failed record out of the stream.
    
- It sends the metadata about the failure to a destination (like an SQS queue or SNS topic) for you to fix later.
    
- **Crucially**, the stream keeps moving forward, and successful records are only processed once.
    

## Why the Other Options are Wrong

Elimination is your best friend in AWS exams. Let's look at why the others fail the criteria:

- **A is wrong:** SQS is a _queue_, not a _stream_. Furthermore, setting `MaximumRetryAttempts` to 0 means it wouldn't retry at all, violating the requirement to "retry failed records."
    
- **C is wrong:** SNS is a pub/sub messaging service, not a stream. Also, Lambda cannot use an SNS topic as a direct stream-like event source in this context.
    
- **D is wrong:** S3 event notifications trigger on file uploads/deletions. While S3 versioning keeps old files, it doesn't inherently manage retry logic or prevent duplicate processing for stream records.
    

## A General Guide to Answering Stream & Queue Questions

When you see questions about processing data in AWS serverless, look for these specific keywords to instantly narrow down your choices:

### 1. Identify the Source Type (Stream vs. Message)

- **If the keyword is "Stream" or "Order matters":** Look for **Kinesis Data Streams** or **DynamoDB Streams**.
    
- **If the keyword is "Decouple", "Asynchronous", or "Fan-out":** Look for **SQS** (queues) or **SNS** (pub/sub).
    

### 2. Look for Error Handling Mechanism Match

- **For SQS/SNS:** The standard way to handle unprocessable messages is a **Dead-Letter Queue (DLQ)**.
    
- **For Kinesis/DynamoDB Streams:** Standard DLQs don't work the same way because the data stays in the stream. Instead, you use **On-Failure Destinations** (Failed-record routing) or "Bisect batch on function error."
    

### 3. Check for Idempotency (Processing Only Once)

The question states the function shouldn't process the same record twice on success. In Kinesis, because Lambda automatically manages the "pointer" (checkpoint) in the stream, once a batch succeeds, Lambda moves to the next set of data. It won't repeat successful records unless the whole batch fails—which is exactly what "failed-record routing" prevents by removing the bad apple.

Does the difference between how a stream (Kinesis) and a queue (SQS) handles retries make sense now, or would you like to dive deeper into one of them?



q5- 
An **AWS VPC Endpoint** (Virtual Private Cloud Endpoint) is essentially a secure, private doorway that connects your private AWS resources to other AWS services without ever sending data over the public internet.

To understand why it's so important, we have to look at the problem it solves.

## The Problem: The "Public Internet" Detour

Imagine you have a private database running on an EC2 instance inside your VPC. You want this instance to save some files into an **Amazon S3 bucket** or send data to **Kinesis**.

By default, even though S3 and Kinesis are AWS services, they live outside your private VPC on public IP addresses. To talk to them, your private EC2 instance would normally have to:

1. Route data through a **NAT Gateway**.
    
2. Go out through an **Internet Gateway** onto the public internet.
    
3. Travel across the internet to reach the public S3 or Kinesis endpoint.
    

This approach has major downsides: it exposes your traffic to the public internet, introduces latency, and **NAT Gateways charge you for every gigabyte of data processed**, which can get incredibly expensive.

## The Solution: VPC Endpoints

A VPC Endpoint cuts out the middleman. It creates a **direct, private connection** from your VPC to the supported AWS service. Your data stays entirely within the secure AWS network, never touching the public internet.

### The Two Main Types of VPC Endpoints

When setting these up, you will choose between two primary types depending on the service you are trying to reach:

#### 1. Interface Endpoints (Powered by AWS PrivateLink)

- **How it works:** It places a private IP address (an Elastic Network Interface, or ENI) right inside your VPC's subnet. To your EC2 instance, the AWS service looks like it's just another local device in the same room.
    
- **Cost:** You pay a small hourly fee for the endpoint, plus data processing fees.
    
- **Supported Services:** Used for almost all AWS services (e.g., Kinesis, SageMaker, Athena, EC2 API) as well as third-party SaaS applications.
    

#### 2. Gateway Endpoints

- **How it works:** Instead of putting an IP address in your subnet, it acts as a target entry in your VPC’s **Route Table**. When your EC2 instance tries to talk to S3, the route table instantly redirects that traffic through the gateway endpoint.
    
- **Cost:** **100% Free.** * **Supported Services:** Only available for two services: **Amazon S3** and **DynamoDB**.
    

## Quick Summary: Why Use a VPC Endpoint?

- **Security:** Data never traverses the public internet, reducing your architecture's attack surface.
    
- **Cost Savings:** For S3 and DynamoDB, using a Gateway Endpoint eliminates NAT Gateway data processing charges completely.
    
- **Performance:** Traffic routes directly within the AWS network, often resulting in lower, more predictable latency.
    

Are you looking to set up an endpoint for a specific service like S3 or Kinesis, or are you studying how they fit into VPC security for an exam?



q10- You are actually spot on with your thinking! The deployment strategy you are describing **is** called a **Canary deployment**.

The reason the answer is **B** is because the question is asking for the specific **AWS Lambda feature** used to build and support that canary deployment.

Let’s break down what Lambda Aliases and Weighted Routing are, and how they make canary deployments work.

## Understanding Lambda Versions and Aliases

To understand aliases, we first need to look at how Lambda handles code updates.

### 1. Lambda Versions

Every time you publish code in AWS Lambda, AWS creates a frozen, immutable snapshot of your function called a **Version** (e.g., Version 1, Version 2). You cannot change a version once it's made.

### 2. Lambda Aliases (The "Pointers")

An **Alias** is like a nickname or a pointer that points to a specific version.

- For example, you can create an alias named `PROD`.
    
- Today, `PROD` points to Version 1 (the stable version).
    
- Your API Gateway or application is configured to talk only to the `PROD` alias.
    

## What is Weighted Routing?

Normally, an alias points to exactly _one_ version. However, AWS Lambda allows you to configure an alias to point to **two different versions at the same time** and assign a "weight" (a percentage) to each. This is **Weighted Routing**.

### How it solves the question:

When you deploy your new code (Version 2), instead of shifting 100% of your users to it immediately, you configure your `PROD` alias like this:

- **Version 1 (Stable):** 90% of traffic
    
- **Version 2 (New):** 10% of traffic
    

Lambda will automatically and randomly route 10% of incoming requests to your new version. If you see errors spike in your logs, you can instantly flip the weight back to 100% on Version 1. If everything looks great, you can slowly roll it up to 100% on Version 2.

**This is exactly how a Canary Deployment is executed under the hood in AWS Lambda.**

## Why the Other Options are Incorrect

- **A. Lambda Layers:** These are used to pull in shared code or external libraries (like an SDK or a common utility function) so your deployment package is smaller. They have nothing to do with traffic routing.
    
- **C. Lambda Environment Variables:** These are key-value pairs used to pass configuration settings (like database passwords or API keys) to your function dynamically without changing the code.
    
- **D. Lambda Reserved Concurrency:** This guarantees a maximum number of concurrent instances your function can scale to. It's used to prevent a function from scaling out of control or to ensure it always has capacity, not for traffic splitting.
    

## Summary for Your Exam

Whenever an AWS exam question mentions **gradually shifting traffic**, **routing a percentage of traffic**, or **canary/blue-green deployments** specifically within **AWS Lambda**, look for **Aliases** and **Weighted Routing**.

Do you want to look at how AWS Developer tools like AWS CodeDeploy can automate this shifting process for you, or does this clear up the feature vs. strategy confusion?

q11- 
In Amazon DynamoDB, a **Conditional Expression** is an "if-then" check that you attach to a write operation (like putting, updating, or deleting an item).

By default, DynamoDB writes are **unconditional**—if you tell DynamoDB to put an item with `User: "John"`, it will overwrite whatever was there before without asking questions. A conditional expression tells DynamoDB: _"Only perform this write **if** a specific condition is met. If it isn't met, cancel the write and throw an error."_

## Why Do We Need Them? (The Real-World Problem)

Imagine you are building a ticket booking system, and there is only **1 ticket left** for a concert.

1. User A and User B click "Buy" at the exact same millisecond.
    
2. Both of their apps read the database and see: `AvailableTickets: 1`.
    
3. User A's app sends an update: `Set AvailableTickets = 0`.
    
4. A fraction of a second later, User B's app sends the exact same update: `Set AvailableTickets = 0`.
    

Without conditional expressions, User B would successfully overwrite User A's purchase, and you would end up selling the same ticket twice!

## How It Works with Conditional Expressions

With conditional expressions, you don't just say "Set tickets to 0." You say:

> _"Set `AvailableTickets = 0` **ONLY IF** `AvailableTickets` is currently equal to 1."_

- **User A's request arrives:** Is `AvailableTickets == 1`? Yes. The write succeeds. The value becomes `0`.
    
- **User B's request arrives:** Is `AvailableTickets == 1`? **No** (it's now 0). The write **fails**. DynamoDB rejects the operation and returns a `ConditionalCheckFailedException`. User B gets a friendly message saying "Sorry, sold out!"
    

## Common Use Cases and Syntax Examples

Here are the most common ways you will see conditional expressions used in development and on AWS exams:

### 1. Preventing Overwrites (`attribute_not_exists`)

When a new user signs up, you want to make sure their username isn't already taken.

- **Expression:** `attribute_not_exists(Username)`
    
- **Meaning:** Only create this item if the `Username` primary key does not already exist in the table.
    

### 2. Checking Values (`=`, `<`, `>`, `contains`)

Ensuring a user has enough balance before a purchase.

- **Expression:** `AccountBalance >= :cost`
    
- **Meaning:** Only deduct the money if their current balance is greater than or equal to the cost of the item.
    

### 3. Verifying Status or Types (`attribute_exists`, `attribute_type`)

Only allowing an order to be canceled if it hasn't shipped yet.

- **Expression:** `OrderStatus = :pending`
    
- **Meaning:** If the status has already changed to "Shipped", the cancellation request fails.
    

## Key Exam Takeaways

- **Idempotency & Concurrency:** Conditional expressions are AWS's primary tool for handling optimistic concurrency control (preventing users from overwriting each other's data).
    
- **Cost Efficiency:** If a conditional expression fails, **you do not pay for the write operation operation unit (WCU)**. However, you do still pay a tiny amount for the read unit (RCU) required to check the item.
    
- **Atomic Operations:** The check and the write happen as a single, atomic step inside DynamoDB. There is zero chance of another request sneaking in between the check and the write.
    

Does this make sense? We can look at how to handle the `ConditionalCheckFailedException` in your code if you'd like!
