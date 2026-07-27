# Day1-Phase 1-Overall Understanding of Data Center Networks

## Obsidian Tags
#Network #Huawei Digital Network #Data Center Network #Underlay #Overlay #OSPF #BGP #VXLAN #Technical Enhancement #Knowledge Base

---

## 1. Learning Topics

As the first day of a 40-day network technology enhancement program, this section focuses on establishing an overall framework for understanding data center networks.

Today, we will not delve into specific protocol details or perform extensive command and configuration exercises. Instead, we will address the following fundamental questions:

- What are the differences in focus between data center networks and traditional enterprise networks?
- What problems do layer 2 and layer 3 networks solve respectively?
- What is Underlay?
- What is Overlay?
- Where do OSPF, BGP, VXLAN, and SDN roughly fall within the network hierarchy?
- Why is this sequence of learning arranged for the next 40 days?

The goal of this section is not to memorize details but to create a “master map” that will guide our subsequent studies.

---

## 2. Why Establish an Overall Understanding First

Network knowledge can easily become fragmented.

If we start directly with a particular protocol, such as OSPF, BGP, or VXLAN, we may encounter the following issues:

- We only remember the name of the protocol but do not understand its role in the overall network.
- We can interpret individual configuration commands but fail to see how they address specific network layers.
- When learning VXLAN, we may not understand why Underlay connectivity is essential beforehand.
- When studying route introduction, we might not grasp why business routes cannot be freely propagated within a data center.
- When learning about SDN, we might mistakenly consider it a single product rather than a higher-level control and orchestration mechanism.

Therefore, the first task is to clarify the hierarchical relationships within the network before delving into specific protocols.

---

## 3. Core Focus Areas of Data Center Networks

Compared to traditional small-scale networks, data center networks place more emphasis on the following aspects:

### 3.1 Scale
The number of devices, business instances, tenants, and network segments is usually much larger, requiring the network to have excellent scalability.

### 3.2 Stability
Business systems are concentrated in these networks, so network issues can directly affect the accessibility of numerous applications. Therefore, higher availability and faster fault recovery times are essential.

### 3.3 Three-Layer Trend
To reduce the problems associated with large broadcast domains, convergence delays, and operational complexity caused by layer 2 networking, data center networks increasingly adopt three-layer interconnection.

### 3.4 Automation and Orchestration
As business scales expand, it becomes impractical to rely solely on manual configuration of individual devices. Controllers, platforms, and unified policy distribution become increasingly crucial.

### 3.5 Multi-Tenant and Virtualization
Cloud platforms, virtualization, and container technologies create demands for network isolation, address reuse, and cross-host communication. This is the main reason why technologies like VXLAN, EVPN, and SDN have emerged.

---

## 4. Differences Between Layer 2 and Layer 3 Networks

To understand data center networks, it is essential to distinguish between layer 2 and layer 3 networks.

### 4.1 Problems Solved by Layer 2 Networks
Layer 2 networks are primarily responsible for Ethernet forwarding within the same broadcast domain. Switches use MAC address tables to forward packets.

Typical characteristics of layer 2 networks include:

- Focus on MAC addresses.
- Hosts within the same VLAN can communicate directly at layer 2.
- Broadcast packets are spread throughout the same broadcast domain.
- Common technologies include VLAN, Access, Trunk, MAC address tables, and ARP.

### 4.2 Problems Solved by Layer 3 Networks
Layer 3 networks are mainly responsible for forwarding between different network segments. Devices use routing tables to determine the next hop for packets.

Typical characteristics of layer 3 networks include:

- Focus on IP addresses and routing tables.
- Gateways are required to forward packets between different network segments.
- More suitable for large-scale interconnection.
- Common technologies include static routing, default routing, OSPF, BGP, etc.

### 4.3 Why Data Centers Are Moving Towards Three-Layer Networks
If a network remains heavily based on layer 2 expansion for an extended period, it may face the following issues:

- Excessively large broadcast domains.
- Wider impact of faults.
- More complex loop control mechanisms.
- Increased costs associated with convergence and troubleshooting.

Therefore, modern data center networks typically strive to achieve three-layer interconnection at the underlying level, ensuring stable and scalable large-scale connectivity.

---

## 5. What is Underlay

Underlay can be understood as the “underlying physical network that provides the foundation.”

It typically has the following characteristics:

- It consists of switches, routers, layer 3 interfaces, routing protocols, etc- https://www.rfc-editor.org/rfc/rfc2328

### BGP
- https://www.rfc-editor.org/rfc/rfc4271

### VXLAN
- https://www.rfC-editor.org/rfc/rfc7348

### EVPN
- https://www.rfC-editor.org/rfc/rfc7432

---

## 13. Today's Summary

The focus of this section is not on the specific details of various protocols, but rather on establishing an overall network framework.

The key points that need to be clarified on the first day are as follows:

- Data center networks generally place greater emphasis on scale, stability, layer 3 functionality, and orchestration capabilities.
- Layer 2 networks are used to handle forwarding within the same broadcast domain, while layer 3 networks address cross-segment forwarding.
- The Underlay represents the underlying physical network, whereas the Overlay is the logical network built upon it.
- Protocols such as OSPF, BGP, VXLAN, and SDN operate at different layers and should not be considered in a simplistic parallel manner.
- The subsequent learning sequence should follow the logic of "starting with the underlying layers, then moving on to connections, control mechanisms, overlays, and finally completing the overall picture."

---

## 14. Tomorrow's Plan

The plan for tomorrow is to study the basics of VLANs, focusing on the following topics:

- The role of VLANs
- The concept of broadcast domains
- The basic logic behind layer 2 isolation
- The relationship between VLANs and subsequent concepts such as VXLAN and VNI.