---
tags:
  - aws
  - certification
  - lambda
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[Lambda Hints]]

Let’s dive into **AWS Lambda** from zero, focusing specifically on how it works and the exact scenarios, configurations, and mechanics that show up on the Developer Associate exam.

## 1. What is AWS Lambda? (The Big Picture)

In traditional computing (like EC2), you rent a virtual server. You pay for it 24/7, even if no one is using your app, and you have to manage the OS, security patches, and scaling.

**AWS Lambda is Serverless Functions-as-a-Service (FaaS).** * You upload your code (Python, Node.js, Java, Go, C#, etc.).

- You define what triggers it (e.g., an API request, a file upload to S3, a schedule).
    
- Lambda runs your code **only when triggered**, scales automatically from zero to thousands of concurrent requests, and shuts down immediately after.
    
- **The Billing Model:** You only pay for the number of requests and the _exact millisecond_ your code executes. If your code isn't running, your bill is $\$0$.
    

## 2. Core Execution Mechanics (Exam Essentials)

To write good code and pass the exam, you must understand how AWS spins up your Lambda function behind the scenes.

### Cold Starts vs. Warm Starts

When a trigger fires and no instance of your function is idling, AWS has to allocate a container, download your code, and start the runtime environment. This delay is called a **Cold Start**.

- **Optimization Trick (Execution Context Reuse):** Anything declared _outside_ the main Lambda handler function (like database connections or AWS SDK clients) stays alive in memory for subsequent invocations (**Warm Starts**).
    
- _Exam Tip:_ If a question asks how to optimize database performance or reduce initialization time in Lambda, the answer is always **initialize your SDK clients/DB connections outside the handler function**.
    

### Memory and CPU Allocation

When configuring a Lambda function, you only have a slider for **Memory** (from 128 MB to 10,240 MB).

- _Exam Tip:_ **You cannot manually adjust CPU.** AWS automatically allocates proportional CPU power based on the amount of memory you choose. If your code is CPU-heavy (like video processing), you make it faster by increasing the _Memory_ slider.
    

### Timeout

A Lambda function can only run for a maximum of **15 minutes (900 seconds)** per invocation. If your function hits 15 minutes, AWS forcefully kills it.

- _Exam Tip:_ Lambda is _not_ for long-running background jobs or continuous processing. For anything longer than 15 minutes, use **AWS Fargate** or **EC2**.
    

## 3. Concurrency (Guaranteed Exam Topic)

Concurrency is the number of in-flight requests your AWS Lambda function is handling at any given moment. By default, an AWS account has a regional limit of **1,000 concurrent executions** shared across all functions.

You must understand the two ways to manage this:

### 1. Reserved Concurrency

This guarantees a maximum number of concurrent instances for a specific function.

- **Why use it?** 1. **To act as a hard ceiling (Throttling):** If your function talks to a legacy database that crashes if it gets more than 50 connections, you set the function's Reserved Concurrency to 50.
    
    2. **To protect a function:** It ensures other noisy functions in your account can't eat up your 1,000 regional quota, leaving your critical function starved.
    

### 2. Provisioned Concurrency

This initializes a requested number of execution environments _before_ traffic arrives.

- **Why use it?** It completely **eliminates Cold Starts**. Your code is already downloaded, initialized, and sitting warm, ready to execute instantly. Use this for highly latency-sensitive applications (like retail checkout APIs).
    

## 4. Lambda Networking & VPCs

By default, Lambda functions run in a secure, AWS-managed VPC with internet access, but **they cannot see your private resources** (like an RDS database or ElastiCache cluster inside your custom VPC).

### Attaching Lambda to a VPC

To allow Lambda to talk to your private database, you must configure the function with your **VPC Subnets** and a **Security Group**.

- **How it works:** AWS attaches a Hyperplane Elastic Network Interface (ENI) to your private subnet, allowing the Lambda function to securely route traffic inside your VPC.
    
- _Exam Trap #1:_ When attached to a VPC, Lambda **loses its default access to the public internet**. If it needs to call an external API (like Stripe), you _must_ route its outbound traffic through a **NAT Gateway** placed in a public subnet.
    
- _Exam Trap #2:_ Placing a Lambda function directly inside a public subnet does _not_ give it a public IP or internet access. It must always use a private subnet + NAT Gateway for internet routing.
    

## 5. Deployment & Configuration

### Environment Variables

You can store key-value pairs inside your function configuration (e.g., `DB_HOST = mydb.xyz.com`).

- They can be encrypted using AWS **KMS**.
    
- _Exam Tip:_ Never hardcode passwords or API keys in your Lambda code. Use environment variables, or better yet, pull them from **Secrets Manager**.
    

### Deployment Packages

You can bundle your code in two ways:

1. **ZIP file archive:** Max size is 50 MB compressed, 250 MB uncompressed.
    
2. **Container Image:** You can package your Lambda function as a Docker image up to **10 GB** in size and host it on Amazon ECR (Elastic Container Registry).
    

## High-Level Exam Cheat Sheet

- **Error Code 429:** Means "Too Many Requests" (You are being throttled because you hit your concurrency limit).
    
- **Dead Letter Queues (DLQ):** If an _asynchronous_ invocation fails twice, Lambda can send the failed event payload to an **SQS queue** or **SNS topic** so you can debug it later.
    
- **Execution Context:** Write your code to utilize `/tmp` directory space (up to 10 GB) for temporary caching between warm starts.
    
- **IAM Role:** Every Lambda function _must_ have an **Execution Role** (IAM Role) that grants it permission to interact with other AWS services (e.g., permission to write logs to CloudWatch).
    

Would you like to try a few **Lambda-specific practice questions** to see how AWS tests this, or are you ready to move on to the next big serverless service, **DynamoDB**?
