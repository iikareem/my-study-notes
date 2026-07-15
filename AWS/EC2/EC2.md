---
tags:
  - aws
  - certification
  - ec2
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[EC2 Hints]] · [[EC2 EBS and KMS]]

# EC2

Let's dive into **Amazon Elastic Compute Cloud (EC2)**. EC2 is the backbone of AWS compute infrastructure. This guide covers everything from absolute zero up to the latest architectures and exact exam traps you will face.

## Part 1: AWS EC2 from Zero (The Fundamentals)

### What is EC2?

Amazon EC2 provides **resizable virtual servers** in the cloud, known as **instances**. Instead of buying physical hardware, you rent virtual capacity. You get full administrative (`root` or `Administrator`) access, allowing you to configure operating systems, install software, and scale resources up or down at will.

### The Standard Components of an EC2 Instance

When you launch an EC2 instance, you aren't just spinning up a CPU; you are stitching together a mini-virtual data center:

- **AMI (Amazon Machine Image):** The template/blueprint of your server. It specifies the Operating System (Linux, Windows), pre-installed software, and access permissions.
    
- **Instance Type:** Determines the hardware profile (how many vCPUs, how much RAM).
    
- **Storage:** Every server needs a hard drive. EC2 utilizes **EBS (Elastic Block Store)** or **Instance Store**.
    
- **Networking:** The instance is assigned to a **VPC (Virtual Private Cloud)**, a specific subnet, and given an IP address.
    
- **Security:** **Security Groups** act as a virtual firewall controlling inbound and outbound traffic.
    

## Part 2: Decoding Instance Types

AWS categorizes its virtual hardware using an alphanumeric system. It's incredibly easy to decipher once you break it down:

Using an example like `m8g.large`:

- **m:** The **Instance Family** (General Purpose).
    
- **8:** The **Generation** (8th generation).
    
- **g:** **Attributes/Processor flag** (Built on AWS Graviton, an ARM-based processor).
    
- **large:** The **Size** (Determines the actual slice of CPU, RAM, and network bandwidth).
    

### The Big Five Families (Memory Hook: **FIGHT-C**)

- **F** - **Fast/Accelerated Computing (P6, G6):** Graphics, heavy AI/ML training, and 3D rendering.
    
- **I** - **I/O Storage Optimized (I5, I7i):** Fast NVMe local storage for massive NoSQL databases or data warehousing.
    
- **G** - **General Purpose (M8a, M8g, T3):** Balanced CPU, memory, and networking. Great for web servers and small code repositories.
    
- **H / C** - **Compute Optimized (C8a, C8i):** Heavy CPU focus. Think batch processing, media encoding, high-performance web servers.
    
- **T** - **Memory Optimized (R8a, R8g, X3):** Huge RAM-to-CPU ratio. Built for in-memory databases (Redis, SAP HANA) and big data processing.
    

## Part 3: Instance Storage: EBS vs. Instance Store

This distinction is tested heavily on every technical AWS exam.

### 1. Amazon EBS (Elastic Block Store)

- A network-attached virtual hard drive.
    
- **Persistence:** If the EC2 instance is stopped or terminated, the data on the EBS volume survives (provided you uncheck "Delete on termination").
    
- You can snapshot EBS volumes to back them up to S3.
    

### 2. EC2 Instance Store

- Physical SSD drives physically attached to the host computer hosting your virtual instance.
    
- **Ephemeral Storage:** The storage is blazing fast with ultra-low latency, but **if the instance is stopped or terminated, you lose ALL data permanently.** Data survives a standard OS reboot, but _not_ a hardware stop.
    

## Part 4: How You Pay for EC2 (The Buying Options)

How you purchase EC2 instances will dictate up to 20% of your score on the architecture exams.

|**Purchase Option**|**Cost Profile**|**Best Use Case**|
|---|---|---|
|**On-Demand**|Fixed price per second/hour. No commitment.|Short-term, unpredictable workloads; dev/test environments.|
|**Savings Plans / Reserved Instances**|Up to **72% discount** for committing to a 1 or 3-year term.|Consistent, predictable, baseline production traffic.|
|**Spot Instances**|Up to **90% discount** by bidding on spare AWS capacity.|Fault-tolerant, stateless, or batch workloads. **AWS can terminate with a 2-minute warning.**|
|**Dedicated Hosts**|Physical servers fully dedicated to your use.|Strict corporate compliance or complex per-socket software licensing (BYOL).|

## Part 5: Exam Tips, Tricks, and "Must-Knows"

### 1. User Data vs. Metadata (Do Not Confuse These!)

- **User Data:** A script or set of commands passed to the instance that **runs exactly once during the very first boot cycle**. Used to bootstrap the server (e.g., `yum install -y httpd`).
    
- **Metadata:** Information _about_ the running instance (its private IP, instance-id, security group names). Accessible from _inside_ the instance by querying the local link-local address: `http://169.254.169.254/latest/meta-data/`.
    

### 2. Security Groups are Stateful, Network ACLs are Stateless

- **Security Groups** operate at the **instance level**. If you open an inbound port (e.g., Allow port 80), the return outbound traffic is **automatically allowed**, regardless of outbound rules.
    
- **NACLs** operate at the **subnet level**. If you allow inbound traffic on port 80, you _must_ also explicitly allow outbound traffic back out on the ephemeral ports.
    

### 3. Placement Groups

When you need to specify how instances are laid out on the physical datacenter racks, use Placement Groups:

- **Cluster:** Packs instances close together inside the same Availability Zone. Used for High-Performance Computing (HPC) requiring low-latency, high-throughput networking.
    
- **Spread:** Ensures instances are placed on completely separate physical server racks with independent power and networking. Max capacity is 7 instances per AZ. Used for highly critical, redundant apps.
    
- **Partition:** Divides the group into logical segments. Instances in one partition do not share hardware with instances in another partition. Great for distributed topologies like Hadoop or Cassandra.
    

### 4. Spot Instance Resiliency Architectures

- **Exam Scenario:** A question asks you to design a highly cost-efficient backend batch processing system that can tolerate interruptions.
    
- **The Answer:** Use an **Auto Scaling Group (ASG) combined with a mix of Spot and On-Demand instances**. Always pair Spot instances with a stateless architecture so that if AWS issues a 2-minute reclamation notice, no user data is corrupted or lost.
    

### 5. Hibernate vs. Stop vs. Terminate

- **Stop:** The instance shuts down. You stop paying for compute, but you **keep paying** for the attached EBS storage volume.
    
- **Terminate:** The virtual instance is deleted forever. The attached EBS volume is usually deleted alongside it unless configured otherwise.
    
- **Hibernate:** The instance saves its RAM contents directly onto the root EBS volume. When you start it back up, it resumes exactly where it left off without going through a full OS boot process.
    

Are there specific infrastructure components—like configuring an Auto Scaling Group or attaching Elastic Load Balancers—that you'd like to sketch out next?



An **Amazon EC2 Auto Scaling Group (ASG)** is a collection of EC2 instances managed as a logical grouping to ensure you have the correct number of servers running to handle the load of your application.

Instead of manually launching a new server when your website gets busy or deleting a server when traffic drops, an ASG automates the entire lifecycle based on parameters and rules you define.

## The Core Concept: Elasticity & High Availability

An ASG solves two massive problems in cloud architecture:

1. **High Availability (Self-Healing):** If an EC2 instance crashes or the physical hardware hosting it fails, the ASG detects this, terminates the unhealthy instance, and immediately launches a brand-new one to replace it.
    
2. **Cost Optimization & Scalability:** It scales out (adds more instances) when traffic spikes to prevent application downtime, and scales in (removes instances) when traffic is low to save you money.
    

## How an ASG Works: The 3 Core Components

To configure an Auto Scaling Group, you must define three main pieces of information:

### 1. What to Launch (Launch Template or Launch Configuration)

You cannot just tell an ASG to "create a server." You must give it a blueprint.

- A **Launch Template** specifies exactly how new instances should be configured: which AMI to use, what instance type (e.g., `t3.micro`), which Security Groups to attach, User Data scripts, and IAM roles.
    
- _Note:_ "Launch Configurations" are the legacy version; always use Launch Templates in modern architecture.
    

### 2. Where to Launch (The Network Settings)

You must define which **VPC** and which specific **subnets** the ASG is allowed to deploy instances into.

- **Exam Best Practice:** Always select subnets spread across multiple Availability Zones (AZs). The ASG will automatically balance your instances across these AZs to ensure that if one entire datacenter goes offline, your application stays up.
    

### 3. How Many to Launch (The Size Limits)

An ASG operates within strict boundaries defined by three configuration settings:

- **Minimum Size:** The absolute fewest number of instances that must be running. Even if traffic drops to zero, the ASG will never scale below this number.
    
- **Maximum Size:** The absolute limit of instances the ASG can create. This acts as a budget cap to prevent runaway costs if your application experiences a massive spike (or a DDoS attack).
    
- **Desired Capacity:** The target number of instances you want running _right now_. The ASG will actively work to maintain this number.
    

## Dynamic Scaling Policies (How it decides to scale)

How does the ASG know _when_ to add or remove servers? You assign a **Scaling Policy**:

- **Target Tracking Scaling:** The simplest and most common method. You choose a metric (like average CPU utilization) and a target value (like 60%). The ASG will constantly add or remove instances to keep the overall fleet average at 60% CPU.
    
- **Step Scaling:** Scales based on defined thresholds. For example: "If CPU is between 50% and 70%, add 1 instance. If CPU is over 70%, add 3 instances."
    
- **Scheduled Scaling:** Scales based on predictable, known time patterns. If you run an online school and traffic spikes every Monday morning at 9:00 AM, you can tell the ASG to proactively scale out before the rush hits.
    
- **Predictive Scaling:** Uses machine learning to analyze your historical traffic patterns and forecasts future traffic, automatically scheduling scaling actions ahead of time.
    

## Exam Tips, Tricks, and "Must-Knows"

Auto Scaling Groups appear in almost every AWS infrastructure question. Memorize these scenarios:

### 1. The ASG + ELB (Elastic Load Balancer) Relationship

- **Exam Scenario:** You have an ASG, but users are complaining that they are hitting broken or unresponsive web servers, and the ASG isn't replacing them.
    
- **The Solution:** By default, an ASG only performs **EC2 Health Checks** (checking if the virtual machine hardware is powered on). If your web server software (like Apache or Nginx) crashes, the VM is technically "healthy" to EC2. You must change the ASG health check type to **ELB (Elastic Load Balancer) Health Checks**. This way, if the load balancer fails to get a successful HTTP response from the instance, it tells the ASG to terminate and replace it.
    

### 2. Cooldown Periods

- **Exam Scenario:** A sudden traffic spike triggers the ASG to add a new server. Five seconds later, before the new server has even finished booting up, the ASG triggers _another_ scale-out event, accidentally spinning up way too many servers.
    
- **The Solution:** Understand the **Scaling Cooldown Period** (Default: 300 seconds). This locks the ASG from making any further scaling decisions for a few minutes, giving newly launched instances time to boot up, start handling traffic, and stabilize the application metrics.
    

### 3. Termination Policies (How it deletes instances)

When an ASG scales _in_ (reduces servers), how does it decide which instance to kill first?

1. It identifies the Availability Zone with the **most instances** (to keep things balanced).
    
2. If multiple instances match, it looks for the one configured with the **oldest Launch Template**.
    
3. If they are identical, it looks for the instance **closest to its next billing hour**.
    

- _Tip:_ You can override this by using a custom Termination Policy or setting up **Instance Scale-In Protection** to prevent critical instances from being terminated.
    

### 4. Lifecycle Hooks

- **Exam Scenario:** When a server is scaling in (terminating), you need to extract its log files and upload them to S3 before the drive vanishes forever.
    
- **The Solution:** Use **ASG Lifecycle Hooks**. This allows you to pause the launch or termination process (putting the instance into a `Pending:Wait` or `Terminating:Wait` state) so you can run custom scripts or AWS Lambda functions before the instance transitions fully.
    

How is your study plan going? Since we've covered KMS, EC2, Cognito, and Auto Scaling, would you like to explore how they connect together—such as looking at an architecture for a highly available web application?
