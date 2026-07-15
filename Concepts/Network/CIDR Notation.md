---
tags:
  - network
---

### **CIDR Notation**

- **What is it?** Classless Inter-Domain Routing (CIDR) notation is a compact way to represent an IP address and its associated **subnet mask**. It’s written as an IP address followed by a slash and a number (e.g., 192.168.1.0/24).
- **The Number (e.g., /24):** Indicates how many bits in the IP address are used for the **network portion**. The remaining bits are for **hosts**.
    - Example: /24 means 24 bits are for the network, leaving 8 bits for hosts (in IPv4, 32 bits total).
    - This translates to 2⁸ = 256 possible addresses in the subnet (including network and broadcast addresses).
- **Purpose:** CIDR simplifies IP address allocation and routing by allowing flexible subnet sizes, replacing older class-based addressing (Class A, B, C).

### **What is a Host and a Network?**

#### **Network:**

- A **network** is a group of devices that can communicate with each other directly (same subnet).
    
- Identified by the **network part** of the IP address.
    
- Example: In `192.168.1.0/24`, the network is `192.168.1.0`.
    

#### **Host:**

- A **host** is any device (computer, phone, server, etc.) that connects to the network.
    
- Each host gets a **unique IP address** within the subnet.

✅ **Given CIDR Notation (e.g., `192.168.10.0/24`) — What Can We Get?**

| What You Get                           | What It’s Called          | Why It’s Useful                                              |
| -------------------------------------- | ------------------------- | ------------------------------------------------------------ |
| `192.168.10.0`                         | **Network Address**       | Identifies the subnet. Routers use this to direct traffic.   |
| `/24` → 255.255.255.0                  | **Subnet Mask**           | Separates network and host portions of the IP.               |
| `192.168.10.1` to `192.168.10.254`     | **Usable Host Range**     | IPs you can assign to devices.                               |
| `192.168.10.255`                       | **Broadcast Address**     | Used to send a message to all hosts in the subnet.           |
| `2^(32 - 24) = 256` IPs                | **Total IP Addresses**    | Total number of IPs in the subnet (including reserved ones). |
| `256 - 2 = 254 usable IPs`             | **Usable Host Count**     | Number of devices you can connect.                           |
| Binary version of IP and mask          | **Binary Representation** | Helpful for subnetting, supernetting, or learning.           |
| Range: `192.168.10.0 - 192.168.10.255` | **IP Block**              | Defines the span of addresses inside the subnet.             |
## **Why Each Is Useful:**

- **Network Address**: Helps routers know which subnet a packet belongs to.
    
- **Subnet Mask**: Used by devices and routers to calculate what’s “local” and what’s not.
    
- **Usable Host IPs**: Lets you know how many devices (servers, laptops, etc.) you can assign addresses to.
    
- **Broadcast Address**: Useful in local networks for announcements or discovery messages (like ARP).
    
- **Total IPs**: Helps in planning your network size—how many you need.
    
- **CIDR Flexibility**: Lets you create just the right-sized subnet to avoid wasting IPs.