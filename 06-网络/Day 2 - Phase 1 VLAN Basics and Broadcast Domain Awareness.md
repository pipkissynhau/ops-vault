# Day2 - Phase 1: VLAN Basics and Broadcast Domain Awareness

## Obsidian Tags
#Network #China #DataCentreNetwork #VLAN #BroadcastingField #SecondTierNetwork #TechnicalEnhancements #KnowledgeBase

---

## 1. Learning Topics

This article covers the second day content of the 40-day network technology reinforcement plan, focusing on VLAN basics and broadcast domain awareness.

The key focus of this day is not to dive into complex configurations or advanced features, but to establish the following foundational understanding:

- What is VLAN
- What is a broadcast domain
- Why switch networks need VLAN
- What problems VLAN primarily solves in actual networks
- The relationship between VLAN and business isolation, access ports, and uplink ports
- The connection between VLAN and subsequent VXLAN / VNI learning

The purpose of this article is to complete the warming up of basic Layer 2 isolation, establishing prerequisite foundations for subsequent topics like Access / Trunk / Hybrid, MAC address tables, ARP, VXLAN, etc.

---

## 2. Why VLAN is Needed

If all interfaces in a switch network are in the same Layer 2 broadcast domain, it will lead to the following issues:

- All hosts are in the same broadcast range, causing broadcast traffic to spread across the entire network
- Lack of isolation between different businesses, departments, and environments
- When issues occur in a specific area, the impact scope may easily expand
- Network management and permission boundaries are unclear
- Difficulty in fault localization and resource allocation when performing subsequent operations

Therefore, switch networks typically do not maintain a "single Layer 2 domain" for long, but instead use VLAN to logically segment Layer 2 networks.

The core function of VLAN can be summarized as one sentence:

**On the same physical switch network, it divides into multiple logically isolated Layer 2 networks.**

---

## 3. What is VLAN

VLAN stands for Virtual Local Area Network, commonly known as a virtual local area network.

Its essence is not to create a new physical network, but to logically divide existing switch networks, separating interfaces that were originally in the same Layer 2 broadcast domain into different Layer 2 domains.

It can be understood from the following perspectives:

### 3.1 Understanding from Isolation Perspective
Different VLANs cannot communicate directly at Layer 2 by default.  
That is, even if two hosts are connected to the same switch, they cannot be in the same Layer 2 broadcast domain if they belong to different VLANs.

### 3.2 Understanding from Broadcast Domain Perspective
A VLAN typically corresponds to a Layer 2 broadcast domain.  
Broadcast packets generally propagate only within the same VLAN and do not spread uncontrollably to other VLANs.

### 3.3 Understanding from Management Perspective
VLAN allows logically isolated boundaries for different businesses, users, areas, and environments on the same physical switch equipment.

---

## 4. What is a Broadcast Domain

Understanding VLAN requires first understanding broadcast domains.

### 4.1 Basic Meaning of Broadcast Packets
Broadcast packets are packets with a target address directed at "all hosts in the local network."  
In Layer 2 networks, typical broadcast traffic is received by devices in the same broadcast domain.

### 4.2 Meaning of Broadcast Domain
A broadcast domain can be understood as:

**The Layer 2 range a broadcast packet can propagate to.**

If the entire switch network does not use VLANs, the broadcast domain theoretically expands continuously.

### 4.3 Problems with Large Broadcast Domains
A large broadcast domain typically means:

- Broadcast traffic range expands
- Non-essential hosts also receive broadcasts
- Network noise increases
- Fault impact scope expands
- Management boundaries are unclear

Therefore, one important value of VLAN is to divide large broadcast domains into multiple smaller, clearer broadcast domains.

---

## 5. Core Value of VLAN

VLAN is not just an "exam topic," but a fundamental Layer 2 capability in actual networks. Its core value mainly lies in the following aspects.

### 5.1 Layer 2 Isolation
Different VLANs cannot communicate directly at Layer 2 by default, enabling basic isolation between businesses.

Examples include:

- Isolation between management network and server network
- Isolation between production environment and test environment
- Isolation between management network and business network
- Isolation between terminals of different departments accessing the network

### 5.2 Reducing Broadcast Domains
After dividing a large Layer 2 network into multiple VLANs, broadcast traffic is limited to propagation within each VLAN, reducing the broadcast impact scope.

### 5.3 Improving Network Manageability
Through VLAN, it's clearer to define:

- Which interfaces belong to which business
- Which users access which network
- Which areas should be isolated by default

### 5.4 Establishing Foundation for Layer 3 Gateway Planning
When a VLAN needs to communicate externally, it usually configures a corresponding Layer 3 gateway interface, such as VLANIF.  
Therefore, VLAN is often the starting point for planning the boundary between Layer 2 and Layer 3.

---

## 6. Relationship Between VLAN and Physical Interfaces

VLAN is a logical division, not equal to "re-wiring."

This means:

- The physical switch remains the same device
- The interfaces remain the same interfaces
- The difference lies in which VLAN the interface is logically assigned to

From this perspective, the significance of VLAN is:

**Different ports of the same switch can logically belong to different networks.**

Example:

- Interface A belongs to VLAN 10
- Interface B belongs to VLAN 20
- Interface C belongs to VLAN 30

Although they may be on the same switch, because they are in different VLANs, they are logically divided into different Layer 2 networks.

---

## 7. Relationship Between VLAN and Business Isolation

In actual networks, VLAN often corresponds to a business boundary rather than just a technical number.

Common understanding methods are as follows:

### 7.1 Dividing by Business
Examples include:

- Management VLAN
- Server VLAN
- Office Network VLAN
- Database VLAN
- Storage VLAN

### 7.2 Dividing by Environment
Examples include:

- Development Environment VLAN
- Testing Environment VLAN
- Production Environment VLAN

### 7.3 Dividing by Area or Floor
Examples include:

- First Floor Office VLAN
- Second Floor Office VLAN
- Data Center Access VLAN

### 7.4 Dividing by Tenant or Resource Domain
In some cloud platforms, virtualization platforms, or large-scale networks, VLAN can also be one of the basic isolation methods for a resource domain.

Note that VLAN is only one method of Layer 2 isolation.  
In larger-scale, more complex scenarios, relying solely on VLAN is often insufficient, which is also an important background for the emergence of VXLAN.

---

## 8. Why Different VLANs Cannot Communicate Directly

This point needs to form a clear understanding first.

### 8.1 Different VLANs Belong to Different Broadcast Domains by Default
Since broadcast domains have been divided, Layer 2 learning and forwarding within the same VLAN will not naturally extend to other VLANs.

### 8.2 Layer 2 Switching Only Occurs Within the Same VLAN
Switches perform Layer 2 forwarding based on MAC address tables, distinguishing by VLAN.  
Therefore, hosts in different VLANs, even if connected to the same switch, will not automatically communicate just because they are physically close.

### 8.3 Cross-VLAN Communication Requires Layer 3 Devices
If a host in VLAN 10 needs to access a host in VLAN 20, it requires a Layer 3 gateway for cross-subnet forwarding.  
This content will be expanded in subsequent sections about VLANIF, Layer 3 interfaces, and gateways.

At this stage, only need to establish this foundational judgment:

**VLAN solves Layer 2 isolation, cross-VLAN communication belongs to Layer 3 forwarding issues.**

---

## 9. VLAN's Fundamental Role in Data Centers

Although modern data center networks have extensively used Layer 3 technologies, VXLAN, EVPN, etc., VLAN remains a very fundamental concept and has not become obsolete.

### 9.1 VLAN is Still Common on the Access Side
Servers, terminals, and some business access ports still may use VLAN as the basis for Layer 2 segmentation on the access side.

### 9.2 Management and Traditional Networks Still Reliant on VLAN
Management networks, office networks, some storage networks, and traditional business access networks still rely on VLAN as a foundational capability.

### 9.3 VXLAN Does Not Negate VLAN
Future learning will show:

- VLAN is more focused on access-side and local Layer 2 segmentation
- VXLAN is more focused on cross-Layer 3 support for large-scale logical networks

Therefore, when understanding VXLAN in the future, it should not be simply understood as "replacing VLAN", but rather as:

**VLAN is the foundation for traditional Layer 2 isolation, VXLAN is a larger-scale Overlay carrying technology.**

---

## 10. Relationship Between VLAN and Subsequent VXLAN / VNI

Today we'll only establish a basic understanding without going into depth.

### 10.1 Limitations of VLAN
VLAN is very important in traditional networks, but it has clear boundaries:

- Limited isolation scale
- More suitable for local Layer 2 segmentation
- Not suitable for directly supporting large-scale cross-Layer 3 logical Layer 2 expansion

### 10.2 Problems VXLAN Solves
When data centers or cloud platforms need:

- Multi-tenant isolation
- Large-scale network segment expansion
- Cross-Layer 3 support for logical Layer 2 networks

VXLAN will be introduced.

### 10.3 Current Only Need to Remember One Sentence
In the future, you can understand their relationship as:

**VLAN is the foundation unit for traditional Layer 2 isolation, VNI is the logical network identifier in the VXLAN world.**

Today, no details about VNI are required to be mastered. Just need to know that VLAN is not an outdated concept for future learning, but an important prerequisite for VXLAN learning.

---

## 11. Basic Understanding of VLAN in Huawei Devices

No actual operation is required today, but need to first recognize the basic forms of VLAN-related configuration and viewing in devices.

### 11.1 Common Configuration Forms
In Huawei devices, VLAN generally involves the following basic forms:

- `vlan batch ...`
- Adding an interface to a specific VLAN
- VLAN membership and permit settings under Access / Trunk / Hybrid ports

No specific port configuration details will be expanded on at this stage. These contents will be explained in the next section about port types.

### 11.2 Common Viewing Commands
When reading notes, it's recommended to first recognize the following commands:

- `display vlan`
- `display port vlan`
- `display current-configuration`

At this stage, only need to know:

- `display vlan` is used to view VLAN information
- VLAN and port types will be further associated during subsequent port learning

---

## 12. Common Misunderstandings at This Stage

### 12.1 Mistakenly Believing VLAN Equals Physical Isolation
VLAN is logical isolation, not rebuilding an independent physical network.

### 12.2 Mistakenly Believing All Hosts on the Same Switch Are Naturally Interconnected
Whether they can communicate depends on whether they are in the same VLAN and whether there is a Layer 3 forwarding path.

### 12.3 Mistakenly Believing Networks Are Definitely More Complex After VLAN Division
From a long-term management perspective, reasonable VLAN division actually helps control broadcast scope, clarify boundaries, and reduce the impact range of faults.

### 12.4 Mistakenly Believing VLAN Is No Longer Important After VXLAN Appears
One of the prerequisites for learning VXLAN is understanding the boundaries and limitations of traditional Layer 2 isolation methods like VLAN.

---

## 13. Minimum Completion Standards After Today's Reading

After completing this section, at least the following states should be achieved:

- Knowing VLAN is a logical Layer 2 isolation method
- Knowing a VLAN usually corresponds to a broadcast domain
- Knowing large broadcast domains can cause problems
- Knowing different VLANs cannot communicate directly at Layer 2 by default
- Knowing cross-VLAN communication belongs to subsequent Layer 3 forwarding issues
- Knowing VLAN remains an important prerequisite for future learning about Access / Trunk / VLANIF / VXLAN

If currently only reaching "seeing these concepts not being unfamiliar, knowing their general functions" is also a normal state.

---

## 14. Related External Links

### Huawei Official Documentation Entry
- https://support.huawei.com/enterprise/zh/index.html
- https://info.support.huawei.com/enterprise/zh/doc/

### Network Foundation References
- https://www.rfc-editor.org/
- https://www.cisco.com/c/en/us/solutions/small-business/resource-center/networking/what-is-a-vlan.html
- https://www.cloudflare.com/learning/network-layer/what-is-a-vlan/

### VXLAN
- https://www.rfc-editor.org/rfc/rfc7348

---

## 15. Summary for Today

The focus of this section is to establish a basic understanding of VLAN and broadcast domains.

The core content to clarify today is as follows:

- VLAN is the foundational capability for dividing multiple logical Layer 2 networks on the same physical switching network
- One important role of VLAN is to split broadcast domains
- Different VLANs cannot communicate directly at Layer 2 by default
- VLAN is commonly used for business isolation, network boundary division, and management control
- VLAN remains an important prerequisite for future learning about port types, Layer 3 gateways, and VXLAN

---

## 16. Plan for Tomorrow

Learn about Access / Trunk / Hybrid port types, focusing on:

- The role of Access ports
- The role of Trunk ports
- Basic understanding of Hybrid ports
- Typical use scenarios for access ports and uplink ports
- The relationship between VLAN and port types

---