---
tags:
  - aws
  - certification
  - network
  - vpc
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[VPC]]

Congrats on completing the VPC basics in Adrian Cantrill’s course! Below is a concise recap of the most important and mandatory concepts for AWS Virtual Private Cloud (VPC) to help you review. I’ll focus on the core elements you need to understand for foundational knowledge, assuming you’re preparing for something like the AWS Solutions Architect Associate certification or practical application.

### Key VPC Concepts
A VPC is a logically isolated section of the AWS cloud where you can launch resources in a virtual network that you define. Think of it as your private network in the cloud.

#### 1. **VPC Fundamentals**
- **Definition**: A VPC is a virtual network within a specific AWS region where you control IP addressing, subnets, routing, and network access.
- **CIDR Block**: 
  - You assign a private IP address range to your VPC using CIDR notation (e.g., `10.0.0.0/16`), which defines the size of your network (up to 65,536 IP addresses for /16).
  - **Mandatory**: Choose a non-overlapping CIDR block to avoid conflicts with other networks (e.g., on-premises or other VPCs).
- **Region-Specific**: A VPC is scoped to one AWS region but can span multiple Availability Zones (AZs).

#### 2. **Subnets**
- **Definition**: Subnets are subdivisions of the VPC’s CIDR block, each mapped to a specific Availability Zone.
- **Types**:
  - **Public Subnet**: Has a route to the internet via an Internet Gateway (IGW) and is typically used for resources like web servers.
  - **Private Subnet**: No direct internet access; used for resources like databases that don’t need public exposure.
- **Mandatory**:
  - Assign a smaller CIDR block to each subnet (e.g., `10.0.1.0/24` for 256 IPs).
  - Ensure subnets don’t overlap within the VPC.
  - Place subnets in different AZs for high availability.

#### 3. **Internet Gateway (IGW)**
- **Purpose**: Connects your VPC to the internet.
- **Mandatory**:
  - Attach an IGW to your VPC (one per VPC).
  - Public subnets need a route to the IGW in their route table (e.g., `0.0.0.0/0` -> IGW) for internet access.
  - Resources in public subnets need a public IP (auto-assigned or Elastic IP) to communicate with the internet.

#### 4. **Route Tables**
- **Definition**: Controls traffic routing within the VPC and to external destinations.
- **Mandatory**:
  - Each subnet is associated with one route table.
  - Default route table allows intra-VPC traffic (local route: `10.0.0.0/16` -> local).
  - For public subnets, add a route to the IGW (`0.0.0.0/0` -> IGW).
  - Private subnets typically don’t have a direct internet route unless using a NAT device.

#### 5. **Network Access Control Lists (NACLs)**
- **Purpose**: Stateless firewall at the subnet level, controlling inbound and outbound traffic.
- **Mandatory**:
  - NACLs are rule-based (evaluated in order, lowest rule number first).
  - Allow or deny traffic based on protocol, port, and source/destination IP.
  - Default NACL allows all traffic; custom NACLs start with a `DENY ALL` rule unless explicitly allowed.
  - **Key**: Both inbound and outbound rules must allow traffic for communication (stateless).

#### 6. **Security Groups**
- **Purpose**: Stateful firewall at the instance level (e.g., EC2 instances).
- **Mandatory**:
  - Only allow rules (no explicit deny).
  - Automatically allows return traffic (stateful).
  - Apply to specific resources (e.g., EC2, RDS) and control traffic based on protocol, port, and source/destination.
  - Example: Allow HTTP (port 80) from `0.0.0.0/0` for a web server.

#### 7. **NAT Gateway/Instance (Optional for Private Subnets)**
- **Purpose**: Allows private subnet resources to access the internet (e.g., for software updates) without being publicly accessible.
- **Mandatory for Private Subnets with Internet Needs**:
  - Place NAT Gateway in a public subnet.
  - Update private subnet route table to route internet-bound traffic (`0.0.0.0/0`) to the NAT Gateway.
  - NAT does not allow inbound internet connections to private resources.

#### 8. **VPC Default Settings**
- **Default VPC**: AWS provides a default VPC in each region with preconfigured settings:
  - CIDR: `172.31.0.0/16`.
  - Public subnets in each AZ with an IGW and default route table.
  - Default NACL and security group.
- **Mandatory**: Understand you can modify or delete the default VPC, but it’s better to create custom VPCs for production to avoid misconfigurations.

#### 9. **Key Best Practices**
- **Plan CIDR Carefully**: Avoid overlap with other VPCs or on-premises networks.
- **Use Multiple AZs**: Deploy subnets across AZs for high availability.
- **Segregate Resources**: Use public subnets for internet-facing resources and private subnets for internal resources.
- **Layer Security**: Combine NACLs (subnet-level) and Security Groups (instance-level) for defense-in-depth.
- **Tagging**: Tag VPCs, subnets, and resources for organization and management.

### Quick Review Checklist
- **VPC**: Define CIDR block (e.g., `10.0.0.0/16`), region-specific.
- **Subnets**: Create public/private subnets, assign to AZs, use non-overlapping CIDRs.
- **IGW**: Attach to VPC, configure public subnet route tables.
- **Route Tables**: Local route for VPC-internal traffic; IGW for public subnets; NAT for private subnets (if needed).
- **NACLs**: Stateless, subnet-level, allow/deny rules.
- **Security Groups**: Stateful, instance-level, allow-only rules.
- **NAT Gateway**: For private subnet internet access (outbound only).
- **Default VPC**: Preconfigured but customize for production.

### Practical Example
- **Scenario**: Launch a web server (public) and database (private).
  - VPC: `10.0.0.0/16`.
  - Public Subnet: `10.0.1.0/24` (AZ1), route to IGW, EC2 with public IP, Security Group allows HTTP (80).
  - Private Subnet: `10.0.2.0/24` (AZ1), no IGW route, RDS instance, Security Group allows MySQL (3306) from EC2.
  - NAT Gateway (in public subnet): For database updates, private subnet routes `0.0.0.0/0` to NAT.
  - NACLs: Allow HTTP (80) to public subnet, MySQL (3306) to private subnet.

### Tips for Retention
- **Visualize**: Sketch a VPC diagram (subnets, IGW, NAT, route tables).
- **Hands-On**: Create a VPC in the AWS Console to reinforce concepts.
- **Focus Areas**: CIDR sizing, public vs. private subnets, and security (NACLs vs. Security Groups).
- **Common Exam Questions**: Subnet routing, NACL rule ordering, Security Group behavior, default VPC settings.

If you want, I can provide a simple VPC setup example with a chart to visualize the architecture (e.g., subnets and routes) or dive deeper into any specific topic. Let me know!
