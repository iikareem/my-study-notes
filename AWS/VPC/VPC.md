---
tags:
  - aws
  - certification
  - network
  - vpc
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[VPC Quiz]] · [[VPC Hints]] · [[VPC Recap]]

Welcome to AWS VPC 101. Whether you are building a real-world architecture or cramming for an AWS certification (like the Solutions Architect Associate), VPC is the absolute bedrock of AWS networking.

Let’s strip away the jargon and break this down from absolute zero to exam-ready.

## 1. What is a VPC? (The Big Picture)

Imagine AWS is a massive, crowded public apartment building. A **VPC (Virtual Private Cloud)** is your own locked, private apartment inside that building.

You control who comes in, who goes out, and how the rooms inside are isolated from each other. It is a **logically isolated virtual network** dedicated to your AWS account.

### The Core Component: CIDR Blocks

When you create a VPC, you must give it a range of IP addresses. This is done using **CIDR (Classless Inter-Domain Routing)** notation.

- **Example:** `10.0.0.0/16`
    
- The `/16` means the first 16 bits are fixed. This gives you **65,536** available IP addresses to use inside your VPC.
    
- _Exam Tip:_ You cannot change the size of a VPC CIDR block after creation. You can only add secondary CIDR blocks.
    

## 2. Subnets (The Rooms in Your Apartment)

You don't just throw all your furniture into one giant room. You divide your VPC into smaller chunks called **Subnets**(Sub-networks).

- Subnets reside _inside_ a specific **Availability Zone (AZ)** (e.g., `us-east-1a`). They cannot span multiple AZs.
    
- You allocate a chunk of your VPC IP range to each subnet (e.g., `10.0.1.0/24`).
    

### The AWS IP "Tax" (Crucial for Exams)

For _every_ subnet you create, AWS reserves **5 IP addresses** for its own management purposes. If a subnet has a `/24` range (256 IPs), you only get **251** usable IPs.

> **The Reserved IPs:**
> 
> - `.0`: Network address
>     
> - `.1`: VPC Router
>     
> - `.2`: DNS (Amazon Provided DNS)
>     
> - `.3`: Future use
>     
> - `.255`: Network broadcast address
>     

### Public vs. Private Subnets

- **Public Subnet:** Connected to the internet. This is where you put public-facing things like web servers or Load Balancers.
    
- **Private Subnet:** Isolated from the internet. This is where you put backend databases or internal application servers.
    

## 3. How Traffic Flows: The Mandatory Components

A subnet doesn't just "become" public or private by magic. It requires specific networking components to dictate how traffic flows.

### Route Tables (The Traffic Cop)

Every VPC comes with a **Main Route Table**. Every subnet must be associated with a route table. It contains a set of rules (routes) that determine where network traffic is directed.

- By default, all subnets can talk to each other because of a local route rule (e.g., Target: `10.0.0.0/16` -> Local).
    

### Internet Gateway (IGW)

The IGW is the bridge between your VPC and the public internet.

- **To make a subnet PUBLIC:** You must create an IGW, attach it to your VPC, and add a route in that subnet's Route Table directing all internet traffic (`0.0.0.0/0`) to the IGW.
    

### NAT Gateway (Network Address Translation)

What if a database in a _private_ subnet needs to download a security patch from the internet, but you don't want hackers to be able to reach it? You use a **NAT Gateway**.

- **How it works:** It allows resources in a private subnet to initiate outbound connections to the internet, but prevents the internet from initiating inbound connections to them.
    
- _Exam Tip:_ NAT Gateways **must be placed in a PUBLIC subnet** and require an Elastic IP (static public IP). Your private subnet's route table will then point `0.0.0.0/0` traffic to the NAT Gateway.
    

## 4. VPC Security: The Two Firewalls

AWS protects your VPC using a multi-layered security approach. You **must** know the differences between these two for the exam.

|**Feature**|**Security Group (SG)**|**Network ACL (NACL)**|
|---|---|---|
|**Operates at**|Instance Level (EC2)|Subnet Level|
|**Statefulness**|**Stateful** (Inbound allowed? Outbound automatically allowed)|**Stateless** (Inbound allowed? You _must_ explicitly allow outbound)|
|**Rules Support**|Allow rules only|Allow AND Deny rules|
|**Evaluation**|Evaluates all rules before deciding|Evaluates rules in numerical order (lowest first)|
|**Default State**|Denies all inbound, allows all outbound|Default NACL allows _all_ traffic in/out|

[Image comparing Security Groups at the instance level and Network ACLs at the subnet level]

## 5. Connecting VPCs and External Networks

As architectures grow, you need to connect your VPC to other VPCs or corporate data centers.

- **VPC Peering:** Connects two VPCs directly using AWS's private network.
    
    - _Exam Tip:_ **No Transitive Peering!** If VPC A is peered with VPC B, and VPC B is peered with VPC C, VPC A _cannot_ talk to VPC C through B. You must create a direct peer between A and C.
        
- **Transit Gateway:** A central hub that connects thousands of VPCs and on-premises networks together. It solves the transitive peering mess by acting as a cloud router.
    
- **VPC Endpoints (PrivateLink):** Allows you to connect your private instances to AWS services (like S3 or DynamoDB) without using an internet gateway or NAT gateway. Traffic stays entirely within the AWS network.
    
    - _Interface Endpoints:_ Uses an Elastic Network Interface (ENI) with a private IP (costs money).
        
    - _Gateway Endpoints:_ Modifies your route table (free). **Only available for S3 and DynamoDB.**
        

## 6. High-Level Cheat Sheet for the Exam

When reading an exam question, look out for these triggers:

1. **High Availability:** Always deploy subnets in at least **two different Availability Zones**.
    
2. **NAT Gateway Failure:** If instances in a private subnet lose internet access, check if the NAT Gateway was placed in a _private_ subnet by mistake (it belongs in a public one), or if its Elastic IP is missing.
    
3. **S3/DynamoDB cost optimization:** Instead of using a NAT Gateway to access S3 from a private subnet, use a **Gateway VPC Endpoint** (it's free and faster).
    
4. **Blocking a specific IP address:** Security Groups cannot deny specific IPs (they only allow). You **must use a Network ACL (NACL)** to explicitly deny an IP address.
    

To understand how a **Route Table**, an **Internet Gateway (IGW)**, and a **Network ACL (NACL)** work together, let’s use a simple real-world analogy.

Imagine your VPC is a **highly secure corporate office building**:

- **The Internet Gateway (IGW)** is the **Front Door** of the building. It connects the inside of the building to the outside world.
    
- **The Route Table** is the **Hallway Signage / Directory**. It tells people which hallway to walk down to get to their destination.
    
- **The Network ACL (NACL)** is the **Security Guard at the entrance of a specific department floor**. They check a list to see who is allowed in or out of that floor.
    

## 1. Route Table (The GPS / Directory)

A Route Table contains a set of rules (called routes) that determine where network traffic from your subnet is directed. Every single subnet in your VPC must be associated with a Route Table.

- **What it does:** It looks at the destination IP address of a packet and says, _"To get there, you need to go this way."_
    
- **Example Rule:** If a server wants to talk to the internet (`0.0.0.0/0`), the Route Table points it toward the Internet Gateway.
    

## 2. Internet Gateway or IGW (The Door)

The IGW is a software component that connects your VPC to the public internet.

- **What it does:** It performs network address translation (NAT) for your public instances and allows traffic from the outside world to enter your VPC.
    
- **The Core Difference from a Route Table:** The IGW is the actual _exit/entrance gate_. The Route Table is just the _map_ that points traffic to that gate. Without an IGW, a Route Table has nowhere to send internet traffic. Without a Route Table pointing to it, the IGW sits there unused.
    

## 3. Network ACL or NACL (The Floor Security Guard)

A Network ACL (Access Control List) is an optional layer of security for your VPC that acts as a firewall for controlling traffic in and out of one or more **subnets**.

- **What it does:** It looks at traffic trying to enter or leave a subnet and either **Allows** or **Denies** it based on strict rules (e.g., _"Block traffic from IP address 192.168.1.50"_).
    
- **The Core Difference from Route Tables & IGWs:** * Route Tables and IGWs **do not care if traffic is safe or malicious**—their only job is to route it.
    
    - The NACL is purely about **security permissions**. It doesn't guide traffic; it decides if traffic has the right to pass through.
        

## Summary of Differences

|**Component**|**Primary Job**|**Analogy**|**Key Feature to Remember**|
|---|---|---|---|
|**Route Table**|**Direction:** Sends traffic to the correct destination.|Hallway Signpost|Contains destination routes (e.g., `0.0.0.0/0` -> IGW).|
|**Internet Gateway (IGW)**|**Connection:** Allows communication with the public internet.|The Main Door|Must be attached to the VPC to make any subnet "Public".|
|**Network ACL (NACL)**|**Protection:** Blocks or permits traffic at the subnet level.|Security Guard|**Stateless** (Must open both inbound and outbound doors explicitly). Supports Deny rules.|

### How they work together in a split second:

When a packet comes from the internet to your EC2 instance:

1. It enters through the **Internet Gateway**.
    
2. The VPC looks at the **Route Table** to see which subnet the packet should go to.
    
3. Before entering that subnet, the **Network ACL** checks its rules to see if that traffic is allowed in. If yes, it passes through to the instance.


