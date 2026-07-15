---
tags:
  - aws
  - certification
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[Region]] · [[Global Infrastructure]]

High Availability (HA), Fault Tolerance (FT), and Disaster Recovery (DR) are distinct but related IT strategies for ensuring system reliability and business continuity. Here’s a clear breakdown to address your confusion:

- **High Availability (HA)**: HA focuses on minimizing downtime by ensuring systems are operational as much as possible, typically targeting 99.999% uptime ("five nines," about 5.26 minutes of downtime per year). It uses redundancy, like duplicate servers or load balancers, to handle failures of individual components. If a server fails, HA systems automatically redirect workloads to a backup, resulting in minimal downtime (seconds to minutes). For example, in VMware, HA restarts a failed virtual machine (VM) on another host in a cluster. HA is cost-effective but allows brief interruptions.[](https://www.lunavi.com/blog/high-availability-vs-fault-tolerance-vs-disaster-recovery)[](https://www.nakivo.com/blog/disaster-recovery-vs-high-availability-vs-fault-tolerance/)

- **Fault Tolerance (FT)**: FT goes further, aiming for zero downtime by maintaining continuous operation even during failures. It achieves this through real-time replication, where a secondary system (e.g., a shadow VM) mirrors the primary system, executing the same instructions simultaneously. If the primary fails, the secondary takes over instantly with no data loss or interruption. FT is ideal for mission-critical applications like financial or healthcare systems but is more complex and costly due to the need for synchronized, redundant systems. In VMware, FT keeps VM copies on separate hosts.[](https://www.lunavi.com/blog/high-availability-vs-fault-tolerance-vs-disaster-recovery)[](https://www.linkedin.com/pulse/vmware-ha-vs-ft-eng-ali-m-alribi)

- **Disaster Recovery (DR)**: DR focuses on restoring systems after major disruptions, like natural disasters or cyberattacks, that render entire infrastructures unusable. It involves policies, tools, and procedures to recover data and functionality, often using off-site backups or secondary data centers. DR accepts some data loss and longer recovery times (based on Recovery Time Objective and Recovery Point Objective) compared to HA and FT. For example, VMware’s vSphere Replication can restore VMs at a secondary site. DR is essential for large-scale failures but isn’t designed for real-time continuity.[](https://www.nakivo.com/blog/disaster-recovery-vs-high-availability-vs-fault-tolerance/)[](https://www.cbtnuggets.com/blog/technology/system-admin/vmware-high-availability-vs-fault-tolerance-vs-disaster-recovery)

**Key Differences**:
- **Purpose**: HA prevents downtime from component failures; FT ensures zero downtime for critical systems; DR recovers systems after catastrophic events.
- **Downtime**: HA has minimal downtime (seconds/minutes); FT has none; DR has longer recovery times (hours/days).
- **Cost/Complexity**: HA is moderately costly; FT is highly expensive and complex; DR varies based on recovery site setup (hot, warm, or cold sites).
- **Use Case**: HA for general business applications; FT for critical systems like banking; DR for large-scale disruptions.

**Overlap**: HA and FT can reduce the need for DR by handling smaller failures, but DR is a backup for major disasters. They complement each other for comprehensive resilience.[](https://www.tierpoint.com/blog/high-availability-vs-disaster-recovery/)

If you need a visual representation or specific examples (e.g., in VMware or AWS), let me know, and I can provide a chart or dive deeper into a platform-specific explanation!
