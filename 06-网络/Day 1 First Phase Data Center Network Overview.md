# Day1 - Phase 1 - Overview of Data Center Networks

## Obsidian Tags
#Network #China #DataCentreNetwork #Underlay #Overlay #OSPF #BGP #VXLAN #TechnicalEnhancements #KnowledgeBase

---

## 1. Learning Topics

This article serves as the first content of the 40-day network technology reinforcement plan, focusing on establishing an overall framework for understanding data center networks.

This day does not delve into specific protocol details or extensive command configurations, but rather answers the following foundational questions first:

- What are the differences in focus between data center networks and traditional enterprise networks?
- What problems do Layer 2 and Layer 3 networks respectively solve?
- What is Underlay?
- What is Overlay?
- Where do OSPF, BGP, VXLAN, and SDN roughly fall in the network architecture?
- Why is the learning sequence for the next 40 days arranged this way?

The goal of this article is not to memorize details, but to first establish a "big picture map" that can span the subsequent learning.

---

## 2. Why Establish an Overall Understanding First

Network knowledge tends to fragment easily.

If you jump directly into a specific protocol, such as OSPF, BGP, or VXLAN from the start, you may encounter the following issues:

- Remembering only the protocol name without understanding its role in the overall network
- Understanding individual configuration fragments but not knowing which layer they address
- Not understanding why Underlay must be reachable when learning VXLAN
- Not grasping why business routes cannot be freely propagated in data centers when learning route import
- Misunderstanding SDN as a single product rather than a higher-level control and orchestration capability when learning SDN

Therefore, the task of Day 1 is not to deeply explore protocols, but to first clarify the hierarchical relationships in the network.

---

## 3. Core Focus of Data Center Networks

Compared to traditional small-scale networks, data center networks typically emphasize the following aspects more:

### 3.1 Scale
The number of devices, business instances, tenants, and network segments is usually larger, requiring better scalability.

### 3.2 Stability
Business systems are concentrated, and network issues often directly impact large-scale application access, thus requiring higher availability and faster fault convergence.

### 3.3 Three-Layer Trend
To reduce the broadcast domain expansion, convergence issues, and operational complexity caused by large Layer 2, data center networks generally lean toward three-layer interconnection.

### 3.4 Automation and Orchestration
As business scales grow, manual configuration per device is no longer suitable. Controllers, platforms, and unified policy deployment will become increasingly important.

### 3.5 Multi-Tenancy and Virtualization
Cloud platforms, virtualization, and container platforms bring requirements for network isolation, address reuse, and cross-host communication, which are also important backgrounds for the existence of technologies like VXLAN, EVPN, and SDN.

---

## 4. Differences Between Layer 2 and Layer 3 Networks

Understanding data center networks requires first distinguishing between Layer 2 and Layer 3.

### 4.1 Problems Solved by Layer 2 Networks
Layer 2 networks primarily handle Ethernet forwarding within the same broadcast domain. Switches forward packets based on MAC address tables.

Typical characteristics of Layer 2 networks:

- Focus on MAC addresses
- Hosts in the same VLAN can directly communicate at Layer 2
- Broadcast packets propagate within the same broadcast domain
- Common technologies include VLAN, Access, Trunk, MAC address tables, ARP, etc.

### 4.2 Problems Solved by Layer 3 Networks
Layer 3 networks primarily handle cross-subnet forwarding. Devices determine the next hop based on routing tables.

Typical characteristics of Layer 3 networks:

- Focus on IP addresses and routing tables
- Different subnets require a gateway for forwarding
- More suitable for large-scale interconnection
- Common technologies include static routing, default routing, OSPF, BGP, etc.

### 4.3 Why Data Centers Are Shifting Toward Layer 3
If the network remains in large-scale Layer 2 expansion for a long time, it will bring the following issues:

- Excessively large broadcast domains
- Expanded impact range of faults
- Complex loop control
- Increased convergence and troubleshooting costs

Therefore, modern data center networks typically strive to make the underlying interconnection as Layer 3 as possible, completing stable large-scale interconnection through Layer 3 networks.

---

## 5. What is Underlay

Underlay can be understood as the "underlying real carrier network."

It typically has the following characteristics:

- Composed of switches, routers, Layer 3 interfaces, routing protocols, etc.
- Responsible for basic IP reachability between devices
- Does not directly reflect the logical network structure of tenant business
- Is the prerequisite for Overlay networks to exist

In subsequent learning, Underlay is usually associated with the following content:

- Layer 3 interconnection
- Loopback reachability
- Dynamic routing protocols like OSPF / BGP
- Routing tables and next hops
- Underlying interconnectivity between devices

You can initially understand Underlay as a single sentence:

**Underlay is responsible for establishing the "path" between devices.**

---

## 6. What is Overlay

Overlay can be understood as a "logical network built on top of the underlying network."

It typically has the following characteristics:

- Relies on the IP reachability of the underlying Underlay
- Provides a logical Layer 2 or Layer 3 network structure for business
- Focuses more on tenant isolation, cross-host communication, and virtual network expansion
- Common technologies include VXLAN, and EVPN will be involved in large-scale scenarios later

You can initially understand Overlay as a single sentence:

**Overlay is responsible for organizing the network that business actually sees.**

---

## 7. Relationship Between Underlay and Overlay

These two concepts are key to the learningMain in subsequent sections.

### 7.1 Underlay Comes First, Then Overlay
Overlay is not a standalone entity independent of the underlying network.  
If the underlying devices cannot reach each other via IP, Overlay cannot function properly.

### 7.2 Underlay Focuses on "How to Reach"
For example:

- How devices communicate with each other
- How routing is learned
- Which path is the next hop
- Whether Loopback addresses are reachable

### 7.3 Overlay Focuses on "What the Business Network Looks Like"
For example:

- How to isolate different tenants
- How to carry the same logical network across Layer 3
- How virtual machines, containers, and servers communicate across nodes

### 7.4 A Common Misconception
It's easy to directly regard VXLAN, EVPN, and SDN as the "network itself," while ignoring the underlying Underlay.

In reality, when the underlying network is not reachable, Overlay will definitely be unstable.  
Therefore, the learning plan's sequence of first reinforcing basics, then OSPF/BGP, and then VXLAN/EVPN is logically consistent with the network structure.

---

## 8. Where Do OSPF, BGP, VXLAN, and SDN Roughly Fall

This section only establishes a rough sense of location, without delving into details.

### 8.1 OSPF
OSPF is a typical internal dynamic routing protocol, commonly used for propagating reachability in internal Layer 3 networks.

In this learning phase, you can initially understand OSPF as:

- One of the common protocols for Underlay in data centers
- Responsible for Layer 3 reachability at the bottom layer
- Focuses on solving internal routing learning and convergence issues

### 8.2 BGP
BGP is more suitable for large-scale network control, boundary propagation, and more complex path selection.

In this learning phase, you can initially understand BGP as:

- A more powerful routing control protocol
- Can be used for boundary interconnection or more advanced control scenarios within data centers
- Will later connect with EVPN and VXLAN

### 8.3 VXLAN
VXLAN is an Overlay technology, not a fundamental routing protocol.

In this learning phase, you can initially understand VXLAN as:

- Used to carry logical Layer 2 networks over Layer 3 networks
- Solves the limited scalability issue of traditional VLANs
- Relies on Underlay to provide underlying IP reachability

### 8.4 SDN
SDN operates at a higher level and does not directly replace OSPF, BGP, VXLAN.

In this learning phase, you can understand SDN as:

- Emphasizing centralized control, unified orchestration, and policy distribution
- Related to controllers, platforms, automation, tunnels, and service deployment capabilities
- An architectural approach for unified management and abstraction of network resources

---

## 9. Why the Learning Order Isn't Random

The learning sequence for this 40-day reinforcement plan is roughly as follows:

### Phase 1: Foundation Refresher
Return to VLAN, MAC, ARP, gateway, static routing, and basic troubleshooting.

The reason is that all subsequent intermediate and advanced content depends on these foundations.

### Phase 2: OSPF
First understand the common dynamic routing approaches in data center underlays.

The reason is that overlay technologies cannot be understood independently from the underlying Layer 3 interconnectivity.

### Phase 3: Route Import
Understand the relationships between different protocols.

The reason is that real networks often have multiple routing sources, requiring protocols to propagate and control appropriately.

### Phase 4: BGP
After understanding internal dynamic routing, move to stronger routing control protocols.

The reason is that BGP's positioning differs from OSPF, requiring further understanding on top of existing routing.

### Phase 5: VXLAN / EVPN
After clearly understanding the underlying reachability and routing propagation, move to overlay networks.

The reason is that this makes it easier to understand why VXLAN necessarily depends on underlay.

### Phase 6: SDN Consolidation
Finally return to the higher-level control and orchestration perspective.

The reason is that by first reinforcing the underlying protocols and forwarding logic, you'll better understand SDN without superficial product features.

---

## 10. Key Foundations to Establish at This Stage

Day 1 doesn't require mastery of details, but it's recommended to first establish the following judgments:

### 10.1 Layer 2 Issues and Layer 3 Issues Are Not the Same Thing
Layer 2 focuses on VLAN, MAC, ARP, and interface access.  
Layer 3 focuses on gateway, routing tables, next hops, and dynamic routing.

### 10.2 Underlay and Overlay Are Not the Same Thing
Underlay solves the underlying interconnectivity.  
Overlay solves the organization of logical networks.

### 10.3 OSPF, BGP, VXLAN Are Not Parallel Replacement Relationships
They are not just "multiple protocols" so simple, but rather located at different levels and positions in the network architecture.

### 10.4 SDN Is Not a Single Tunnel Technology
SDN leans more toward control and orchestration layers, and shouldn't be understood as a single product or tunnel itself.

---

## 11. Minimum Completion Standards After Today's Reading

After completing this article, you should at least reach the following state:

- Be able to explain the general differences between Layer 2 and Layer 3 networks
- Be able to explain the basic meanings of Underlay and Overlay
- Know the general positions of OSPF, BGP, VXLAN, and SDN in the overall framework
- Understand that the learning sequence follows "first foundations, then Underlay, then route import, then BGP, then VXLAN, and finally SDN"

If you can only reach "having an impression, being able to recognize, but not yet proficient," that's also a normal state.

---

## 12. Related External Links

### Huawei Official Documentation Entry
- https://support.huawei.com/enterprise/zh/index.html
- https://info.support.huawei.com/enterprise/zh/doc/

### Network Foundation References
- https://www.rfc-editor.org/
- https://www.cisco.com/c/en/us/solutions/enterprise-networks/what-is-networking.html
- https://www.ibm.com/think/topics/software-defined-networking

### OSPF
- https://www.rfc-editor.org/rfc/rfc2328

### BGP
- https://www.rfc-editor.org/rfc/rfc4271

### VXLAN
- https://www.rfc-editor.org/rfc/rfc7348

### EVPN
- https://www.rfc-editor.org/rfc/rfc7432

---

## 13. Summary for Today

The focus of this article isn't specific protocol details, but establishing the overall network framework.

The core content to clarify on Day 1 is as follows:

- Data center networks generally emphasize scalability, stability, Layer 3 capabilities, and orchestration
- Layer 2 networks solve forwarding within the same broadcast domain, while Layer 3 networks solve cross-subnet forwarding
- Underlay is the underlying real carrier network, while Overlay is the logical network built upon it
- OSPF, BGP, VXLAN, and SDN are located at different levels and cannot be simply viewed as parallel entities
- The learning sequence follows the logic of "first foundations, then interconnection, then control, then Overlay, and finally consolidation"

---

## 14. Tomorrow's Plan

Study VLAN basics, focusing on:

- The role of VLAN
- The concept of broadcast domains
- The basic logic of Layer 2 isolation
- The relationship between VLAN and subsequent VXLAN / VNI understanding

--- /think