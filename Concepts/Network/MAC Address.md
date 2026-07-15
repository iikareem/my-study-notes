---
tags:
  - network
---

### **MAC Address**

- **What is it?** A Media Access Control (MAC) address is a unique identifier assigned to a network interface card (NIC) for devices connected to a network. It operates at the **Data Link Layer** (Layer 2) of the OSI model.
- **Format:** A 48-bit address, typically written as six pairs of hexadecimal digits (e.g., 00:1A:2B:3C:4D:5E).
- **Purpose:** **Used for local network communication to identify devices within the same network** segment (e.g., Ethernet or Wi-Fi). It’s like a device’s hardware “fingerprint.”
- **Key Point:** **MAC addresses are fixed by the manufacturer and used for local traffic, not routable across the internet.**

HUB VS Switch

|Feature|**Hub**|**Switch**|
|---|---|---|
|📌 Role|Basic device to **connect devices** in a network|Smarter device to **connect and manage traffic** between devices|
|🧠 Intelligence|❌ Not smart – just **forwards everything**|✅ Smart – **sends data only to the right device**|
|📤 Traffic|**Broadcasts** data to **all devices**|**Directs** data to **specific device** only|
|🚀 Speed|Slower, more **collisions**|Faster, fewer collisions, better **performance**|
|🏠 Use Case|Rarely used today (old tech)|Commonly used in **modern networks**|
## 🧠 What is “Intelligence” in Networking Devices?

### 🔌 **Hub – No Intelligence**

- **Doesn't care who the data is for.**
    
- Just **forwards** the incoming data to **all connected devices**, hoping the right one receives it.
    
- Like a **loudspeaker** in a room – everyone hears everything.
    
- ❌ **No decision-making**, no memory.

### 🔀 **Switch – Has Intelligence**

- **Learns** which device is connected to which port by reading their **MAC addresses**.
    
- Stores this in a **MAC address table**.
    
- When data comes in, the switch checks this table and sends the data **only to the correct port**.
    
- ✅ **Makes decisions**, **remembers** device locations.

Both **hub** and **switch** are used to connect devices **within a local area network (LAN)** — like in your home, office, or school network.