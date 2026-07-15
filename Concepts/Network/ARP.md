---
tags:
  - network
---

## 🔎 What is ARP?

**ARP (Address Resolution Protocol)** is a protocol used to **find the MAC address** of a device when you only know its **IP address**.

---

## 🧠 Why Do We Need ARP?

In a local network:

- Devices communicate using **MAC addresses** (Layer 2).
    
- But applications and users usually use **IP addresses** (Layer 3).
    

So, when one device wants to send data to an IP address, it needs to find out:  
**"What is the MAC address of the device with this IP?"**

That’s where **ARP** comes in.

---

## 🛠️ What ARP Does (Simple Steps):

1. Your device wants to talk to IP `192.168.1.10`.
    
2. It checks its ARP table to see if it knows the MAC address for that IP.
    
3. If **not found**, it sends a **broadcast message**:
    
    > "Who has IP `192.168.1.10`? Tell me your MAC!"
    
4. The device with that IP replies:
    
    > "I have it. My MAC is `AA:BB:CC:DD:EE:FF`"
    
5. Your device stores this in its **ARP cache** and uses it to send data.
    

---

## 📦 Summary:

|Feature|Description|
|---|---|
|**Full Name**|Address Resolution Protocol|
|**Purpose**|Maps **IP addresses → MAC addresses**|
|**Used In**|Local networks (LAN)|
|**Type**|Layer 2/3 (Data Link / Network layer connector)|
|**Example**|"I want to send data to IP 192.168.1.10 – what's the MAC?"|

---

Let me know if you want to see how this works with a diagram or a real example from your system’s ARP cache (`arp -a` command)!

## When Two Devices in the Same Wi-Fi Network Communicate:

### 🔹 Step 1: **Use ARP (Layer 3 → Layer 2)**

- Device A knows Device B’s **IP address**.
    
- It sends an **ARP request** to get Device B’s **MAC address**.
    
- Now it can build a data packet to send.
    

> 📌 **ARP = Find out: “What is the MAC address of this IP?”**

---

### 🔹 Step 2: **Router acts like a switch (Layer 2)**

- Your **Wi-Fi router**, acting like a **switch**, forwards the data **based on MAC addresses**.
    
- It **doesn't need to route**, since both devices are in the **same subnet**.
    
- The router just says:
    
    > “Oh, I see this MAC is connected via Wi-Fi — I’ll forward it there.”
    

---

## 🧠 So in simple terms:

> **ARP** happens first to get the MAC address,  
> then the **router (as a switch)** forwards the data to the right device.

✅ All of this happens **inside your LAN**, quickly and efficiently.