---
tags:
  - aws
  - certification
  - network
  - vpc
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[VPC]] · [[VPC Hints]]

Here is a set of 10 exam-style practice questions designed to mimic the difficulty, structure, and scenarios you will encounter on the **AWS Certified Developer - Associate (DVA-C02)** exam regarding VPC and networking.

Try to answer them on your own, and then check the **Answer Key & Explanations** provided at the bottom!

### **Practice Questions**

#### **Question 1**

An application hosted on an Amazon EC2 instance in a private subnet needs to securely fetch configuration files from an Amazon S3 bucket. The traffic must not traverse the public internet, and you want to minimize data transfer costs. How should a Developer configure this?

- A) Set up a NAT Gateway in a public subnet and route S3 traffic through it.
    
- B) Create an Interface VPC Endpoint for Amazon S3 and update the subnet's route table.
    
- C) Create a Gateway VPC Endpoint for Amazon S3 and associate it with the private subnet's route table.
    
- D) Attach an Internet Gateway to the VPC and assign a public IP address to the EC2 instance.
    

#### **Question 2**

A Developer has deployed an AWS Lambda function that needs to query an Amazon RDS MySQL database located within a private subnet of a VPC. After configuring the Lambda function to access the VPC subnets and security groups, the function fails to connect to the database, timing out. What is the most likely cause of this issue?

- A) The Lambda function requires an Internet Gateway attached to its private subnet.
    
- B) The database's Security Group does not allow inbound traffic from the Lambda function's Security Group.
    
- C) Lambda functions cannot be deployed inside a VPC if they need to access RDS.
    
- D) The Lambda function's IAM execution role is missing the `AmazonS3ReadOnlyAccess` policy.
    

#### **Question 3**

You are troubleshooting a connectivity issue where an EC2 instance in a public subnet cannot receive HTTP traffic on port 80 from the internet. You have verified that the Internet Gateway is attached, the Route Table has a path (`0.0.0.0/0 -> IGW`), and the Network ACL allows inbound port 80 traffic. The Security Group attached to the instance has an inbound rule allowing port 80 from `0.0.0.0/0`, but no outbound rules. Why is the traffic failing?

- A) Security Groups are stateless; you must explicitly add an outbound rule for ephemeral ports to allow the response traffic.
    
- B) Security Groups are stateful; the missing outbound rule is causing the response traffic to be blocked.
    
- C) Security Groups are stateful; they automatically allow return traffic, so the issue must be that the Network ACL is missing an outbound rule for return traffic.
    
- D) The Internet Gateway requires an outbound rule to allow traffic back to the internet.
    

#### **Question 4**

An application components tier consists of EC2 instances across two Availability Zones in private subnets. A Developer needs to allow these instances to download software updates from the internet while ensuring that no external internet entities can initiate a connection to the instances. Which configuration meets these requirements with the least operational overhead?

- A) Deploy a NAT Instance in each private subnet and disable the Source/Destination check.
    
- B) Create a NAT Gateway in a public subnet and add a route in the private subnets' route table pointing `0.0.0.0/0` to the NAT Gateway.
    
- C) Attach an Internet Gateway to the VPC and configure a Network ACL to block all inbound traffic.
    
- D) Deploy a Transit Gateway in a public subnet and peer it with the private subnets.
    

#### **Question 5**

A company has two VPCs: VPC A (Application tier) and VPC B (Analytics tier). A Developer establishes a VPC Peering connection between VPC A and VPC B. Instances in VPC A can successfully ping instances in VPC B, but they cannot access a shared database in VPC C, which already has an active, working VPC Peering connection with VPC B. What is the reason for this?

- A) VPC Peering does not support TCP traffic, only ICMP (ping).
    
- B) VPC Peering is non-transitive; VPC A cannot route traffic through VPC B to reach VPC C.
    
- C) The Security Group of the database in VPC C must implicitly trust VPC A.
    
- D) You cannot have more than one VPC Peering connection attached to a single VPC.
    

#### **Question 6**

When launching an EC2 instance in a newly created custom VPC subnet, a Developer notices that the instance is not assigned a public IPv4 address, preventing remote SSH access. What is the quickest way to ensure that future instances launched in this specific subnet automatically receive a public IP address?

- A) Modify the subnet settings and enable the "Auto-assign public IPv4 address" attribute.
    
- B) Attach a secondary CIDR block to the VPC dedicated to public addresses.
    
- C) Recreate the VPC because default subnet attributes cannot be altered after creation.
    
- D) Create a Main Route Table and assign it as the primary directory for the subnet.
    

#### **Question 7**

A security audit reveals that malicious traffic is originating from a specific external IP address (`203.0.113.50`) and targeting web servers inside a public subnet. A Developer is tasked with immediately blocking all incoming and outgoing traffic from this specific IP address. Which action should the Developer take?

- A) Add a Deny rule for `203.0.113.50/32` in the Security Group attached to the web servers.
    
- B) Remove the `0.0.0.0/0` route from the Route Table to disconnect the subnet temporarily.
    
- C) Add an Inbound Deny rule and an Outbound Deny rule for `203.0.113.50/32` in the Subnet's Network ACL (NACL) with a low rule number.
    
- D) Configure an IAM Policy to deny requests coming from the malicious IP address.
    

#### **Question 8**

A Developer is setting up a subnet with a CIDR block of `10.0.1.0/24`. The Developer intends to deploy a high-availability cluster that requires exactly 254 usable IP addresses inside this single subnet. Will this setup work?

- A) Yes, a `/24` CIDR block provides exactly 256 addresses, leaving room for 254 hosts.
    
- B) No, AWS reserves 2 IP addresses per subnet (Network and Broadcast), leaving only 254, but the router takes one more.
    
- C) No, AWS reserves 5 IP addresses in every subnet for management purposes, meaning only 251 IP addresses are usable.
    
- D) Yes, because AWS dynamically allocates addresses from the adjacent subnets if a shortage occurs.
    

#### **Question 9**

An AWS Lambda function is configured to run inside a custom VPC so it can securely access an Amazon ElastiCache cluster. The Lambda function now needs to call an external third-party payment gateway API via the public internet. What infrastructure components are required to allow the Lambda function to access the internet?

- A) The Lambda function must be assigned a public IP directly within its configuration.
    
- B) The Lambda function must be placed in a public subnet with a route to an Internet Gateway.
    
- C) The Lambda function must be placed in a private subnet with a route to a NAT Gateway located in a public subnet.
    
- D) Lambda functions cannot access both internal VPC resources and the public internet at the same time.
    

#### **Question 10**

A Developer creates a new custom VPC and an Internet Gateway (IGW). The Developer attaches the IGW to the VPC and creates a public subnet. However, EC2 instances launched inside this subnet still cannot reach the internet. Which mandatory step did the Developer forget?

- A) Enabling VPC Flow Logs to authorize outward routing.
    
- B) Creating a route in the subnet's Route Table that directs `0.0.0.0/0` traffic to the Internet Gateway.
    
- C) Associating the Internet Gateway directly with the EC2 instances' Elastic Network Interfaces (ENIs).
    
- D) Modifying the Main Network ACL to permit traffic to pass to the IGW explicitly.
    

### **Answer Key & Explanations**

#### **1. Correct Answer: C**

- **Explanation:** A **Gateway VPC Endpoint** allows private resources to connect to Amazon S3 or DynamoDB without leaving the AWS private network. It does not cost anything extra and modifies the route table directly. While an Interface Endpoint (Option B) or NAT Gateway (Option A) would work, they incur hourly charges and data processing fees, making Gateway Endpoints the most cost-effective and recommended method for S3.
    

#### **2. Correct Answer: B**

- **Explanation:** When an application tier (like Lambda) tries to talk to a database tier (like RDS) inside a VPC, the most common cause of a timeout is a firewall restriction. The RDS Security Group must explicitly permit inbound traffic from the Lambda function's Security Group or its private IP range on the database port (e.g., 3306 for MySQL).
    

#### **3. Correct Answer: C**

- **Explanation:** Security Groups are **stateful**—if traffic is allowed inbound, the return outbound traffic is automatically allowed regardless of outbound rules. Network ACLs, however, are **stateless**. Because the question specifies that the NACL allows inbound traffic but says nothing about outbound, the return traffic is getting blocked by the NACL's default outbound rules or lack of ephemeral port allocation.
    

#### **4. Correct Answer: B**

- **Explanation:** A **NAT Gateway** is a managed AWS service designed precisely for this use case: allowing outbound-only internet access for private resources while blocking incoming connection attempts. NAT Instances (Option A) require significant operational overhead (patching, scaling, scripts).
    

#### **5. Correct Answer: B**

- **Explanation:** AWS **VPC Peering does not support transitive routing**. If VPC A is peered with VPC B, and VPC B is peered with VPC C, traffic from A cannot "hop" through B to get to C. To fix this, a direct peering connection must be built between VPC A and VPC C, or you must switch to an AWS Transit Gateway.
    

#### **6. Correct Answer: A**

- **Explanation:** Subnets created manually inside a custom VPC have the "Auto-assign public IPv4 address" setting turned **off** by default. To change this behavior for all future instances deployed in that specific subnet, you simply check this box in the subnet settings.
    

#### **7. Correct Answer: C**

- **Explanation:** Security Groups **only support Allow rules**; you cannot explicitly Deny an IP address using a Security Group. To block a specific malicious IP address, you must use a **Network ACL (NACL)**, which supports both Allow and Deny rules, and evaluate it with a low rule number so it executes before any broader "Allow" rules.
    

#### **8. Correct Answer: C**

- **Explanation:** This is a classic AWS trivia question. A `/24` network has 256 total IP addresses ($2^{(32-24)} = 2^8 = 256$). However, AWS reserves **5 IP addresses** in every single subnet (`.0`, `.1`, `.2`, `.3`, and `.255`), leaving exactly **251**usable IP addresses. Therefore, a cluster requiring 254 hosts will fail due to insufficient IPs.
    

#### **9. Correct Answer: C**

- **Explanation:** When a Lambda function is attached to a VPC, it loses its default internet access. To regain internet access while staying connected to VPC resources (like ElastiCache), the Lambda function **must run in a private subnet** and route its internet-bound traffic through a **NAT Gateway** residing in a public subnet. (Note: Placing a Lambda function directly in a public subnet does _not_ grant it a public IP or internet access).
    

#### **10. Correct Answer: B**

- **Explanation:** Simply creating and attaching an Internet Gateway does not automatically route traffic to it. You must explicitly update the **Route Table** associated with your public subnet to include a default route (`0.0.0.0/0`) pointing to the Internet Gateway (`igw-xxxxxx`) as the target.
