# OSPFv2 Network Types & Control Plane Analysis: Multi-Access Broadcast vs. Point-to-Point

![Platform](https://img.shields.io/badge/Platform-EVE--NG-blue)
![Vendor](https://img.shields.io/badge/Vendor-Cisco%20IOS--XE-orange)
![Protocol](https://img.shields.io/badge/Protocol-OSPFv2%20%7C%20Network%20Types-green)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

This repository contains the full architecture, Cisco IOS-XE configurations, protocol behavioral analysis, and Wireshark control-plane validation tasks for studying OSPFv2 behavior across **Multi-Access Broadcast** and **Point-to-Point (P2P)** network types within a single OSPF Area 0 topology.

---

## 📋 Overview & Protocol Behavioral Objectives

This laboratory is specifically designed for deep-dive protocol analysis comparing OSPF operating characteristics across different physical/logical link types:

* **Broadcast Multi-Access Domain (`192.168.2.0/24`):**
  * **Election Dynamics:** Study of Designated Router (DR) and Backup Designated Router (BDR) election algorithm based on Priority and Router-ID.
  * **Multicast Destination Grouping:** Packet capture analysis contrasting `224.0.0.5` (**AllSPFRouters**) and `224.0.0.6` (**AllDRouters**).
  * **LSA Abstraction:** Generation of **Type 2 Network LSAs** by the DR to represent pseudonode adjacency graphs.
  * **State Machine Boundaries:** Observation of `2-WAY` neighbor states between DROTHER routers versus `FULL` adjacencies formed exclusively with the DR and BDR.
* **Point-to-Point Domain (`192.168.3.0/24` & `192.168.4.0/24`):**
  * **DR/BDR Bypass:** Complete suppression of election timers (40-second Wait timer eliminated during startup).
  * **Database Efficiency:** Suppression of Type 2 Network LSAs; point-to-point links are directly represented as subnets/links within Type 1 Router LSAs.
  * **Direct Adjacency:** Direct transition into `FULL` state across inter-router transit links.

---

## 📐 Network Topology

```text
                                                       +------------------+
                                                       |        R2        | (192.168.3.0/24)
                                                 +---->|  10.255.255.2/32 +-------------------+
                                                 |     +------------------+                   |
                                                 |                                            v
+------------------+     +-------+     +----+    |     +------------------+            +---------------+     +-------+     +------------------+
|    Linux PC1     |     | Switch|     | R1 |    |     |        R3        |            |      R4       |     | Switch|     |    Linux PC2     |
|  192.168.1.2/24  |<--->|   1   |<--->|    |<---+---->|  10.255.255.3/32 +----------->| 10.255.255.4  |<--->|   3   |<--->|  192.168.5.2/24  |
+------------------+     +-------+     +----+    |     +------------------+            +---------------+     +-------+     +------------------+
                                  (192.168.2.0)  |       (192.168.4.0/24)
                                  [Switch2 Segment]
                                                 |     +------------------+
                                                 +---->|        R5        |
                                                       |  10.255.255.5/32 | [DROTHER / Priority 0]
                                                       +------------------+ (No link to R4)
```

## 🏷️ Addressing Matrix & OSPF Interface Roles

| Node | Interface | IP Address / Mask | OSPF Network Type | Priority | Designated Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R1** | `Loopback0`<br>`Gi0/0`<br>`Gi0/1` | `10.255.255.1/32`<br>`192.168.1.1/24`<br>`192.168.2.1/24` | Loopback<br>Broadcast (Passive)<br>Broadcast | 1<br>N/A<br>1 | RID<br>Access Stub<br>DR / BDR Candidate |
| **R2** | `Loopback0`<br>`Gi0/0`<br>`Gi0/1` | `10.255.255.2/32`<br>`192.168.2.2/24`<br>`192.168.3.2/24` | Loopback<br>Broadcast<br>Point-to-Point | 1<br>1<br>N/A | RID<br>DR / BDR Candidate<br>P2P Transit Neighbor |
| **R3** | `Loopback0`<br>`Gi0/0`<br>`Gi0/1` | `10.255.255.3/32`<br>`192.168.2.3/24`<br>`192.168.4.3/24` | Loopback<br>Broadcast<br>Point-to-Point | 1<br>1<br>N/A | RID<br>DR / BDR Candidate<br>P2P Transit Neighbor |
| **R5** | `Loopback0`<br>`Gi0/0` | `10.255.255.5/32`<br>`192.168.2.5/24` | Loopback<br>Broadcast | 1<br>**0** | RID<br>**DROTHER** (Elections Disabled) |
| **R4** | `Loopback0`<br>`Gi0/0`<br>`Gi0/1`<br>`Gi0/2` | `10.255.255.4/32`<br>`192.168.3.4/24`<br>`192.168.4.4/24`<br>`192.168.5.1/24` | Loopback<br>Point-to-Point<br>Point-to-Point<br>Broadcast (Passive) | N/A<br>N/A<br>N/A<br>N/A | RID<br>P2P Transit Neighbor<br>P2P Transit Neighbor<br>Access Stub |


## 🧪 Deep-Dive Protocol Behavioral Comparison

| Feature / Behavior | Broadcast Multi-Access | Point-to-Point (P2P) |
| :--- | :--- | :--- |
| **DR/BDR Election** | Enabled (40-sec Wait Timer) | Disabled (Instant Formation) |
| **Multicast Addresses Used** | `224.0.0.5` (All) & `224.0.0.6` (DR) | `224.0.0.5` Only |
| **Type 2 Network LSA Generated?** | Yes (Generated by DR) | No |
| **Neighbor Adjacency States** | `FULL` (DR/BDR), `2WAY` (DROTHERs) | Always `FULL` |
| **Mask Requirement for Adjacency** | Must match exactly | Subnet mask can differ (IOS) |

### Key Behavioral Analysis Points:

1. **Multicast Packet Flow on `192.168.2.0/24` (Broadcast):**
   * **DROTHERs** (e.g., R5) send Link State Updates (LSU) and Link State Acknowledgments (LSAck) directly to **`224.0.0.6`** (AllDRouters).
   * The **DR** receives these updates, processes them, and floods the LSU out to the rest of the segment using **`224.0.0.5`** (AllSPFRouters).

2. **LSA Type 2 Elimination on P2P Links:**
   * On interfaces `Gi0/1` of R2/R3 and `Gi0/0`/`Gi0/1` of R4, `ip ospf network point-to-point` suppresses Type 2 LSA generation.
   * This reduces overall Link-State Database (LSDB) memory footprint and SPF calculations domain-wide.

3. **Multi-Path Routing (2-Way ECMP):**
   * Traffic traveling from PC1 (`192.168.1.2`) to PC2 (`192.168.5.2`) is load-balanced equally over **R2** and **R3**.
   * R5 remains an edge node on `Switch2` without forwarding capabilities toward R4 due to the missing link (`192.168.6.0/24`).
