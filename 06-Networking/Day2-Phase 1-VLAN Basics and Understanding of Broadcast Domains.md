# Day2-Phase 1-VLAN Basics and Understanding of Broadcast Domains

## Obsidian Tags
#Network #Huawei Datacom #Data Center Networking #VLAN #Broadcast Domain #Layer 2 Networking #Technical Enhancement #Knowledge Base

---

## 1. Learning Topic

This is the second day of the 40-day network technology enhancement program, focusing on VLAN basics and understanding of broadcast domains.

Today, we will not delve into complex configurations or advanced features but instead establish the following fundamental understandings:

- What is a VLAN?
- What is a broadcast domain?
- Why do switch networks need VLANs?
- What problems do VLANs mainly solve in actual networks?
- The relationship between VLANs, service isolation, access ports, and uplink ports.
- The connection between VLANs and subsequent studies on VXLAN/VNI.

The purpose of this section is to review the basics of layer 2 isolation, laying the foundation for later topics such as Access/Trunk/Hybrid ports, MAC address tables, ARP, and VXLAN.

---

## 2. Why Are VLANs Needed?

If all interfaces in a switch network are part of the same layer 2 broadcast domain, it will lead to the following issues:

- All hosts are within the same broadcast range, causing broadcast traffic to spread throughout the entire network.
- There is a lack of isolation between different services, departments, or environments.
- If one area encounters problems, the impact can easily expand.
- Network management and permission boundaries become unclear.
- It becomes more difficult to locate faults and allocate resources.

Therefore, switch networks typically do not maintain a state where "the entire network belongs to one layer 2 domain" for long periods. Instead, VLANs are used to logically divide the layer 2 network.

The core function of VLANs can be summarized as follows:

**On the same set of physical switch devices, multiple isolated logical layer 2 networks are created.**

---

## 3. What Is a VLAN?

A VLAN, or Virtual Local Area Network, is essentially a logical division of an existing physical switch network. It does not create a new physical network but divides interfaces that would otherwise be part of the same layer 2 broadcast domain into separate ones.

You can understand VLANs from the following perspectives:

### 3.1 From the Perspective of Isolation
By default, different VLANs cannot communicate directly at the layer 2 level.  
In other words, even if two hosts are connected to the same switch, if they belong to different VLANs, they are not in the same layer 2 broadcast domain.

### 3.2 From the Perspective of Broadcast Domains
A VLAN usually corresponds to one layer 2 broadcast domain.  
Broadcast packets generally only travel within their respective VLANs and will not uncontrollably spread to other VLANs.

### 3.3 From the Perspective of Management
VLANs allow logical isolation boundaries to be established on the same set of physical switch devices for different services, users, areas, or environments.

---

## 4. What Is a Broadcast Domain?

To understand VLANs, it is essential to first understand broadcast domains.

### 4.1 The Basic Meaning of Broadcast Packets
Broadcast packets are messages whose destination address is "all hosts within this network."  
In layer 2 networks, typical broadcast traffic is received by all devices within the same broadcast domain.

### 4.2 The Meaning of a Broadcast Domain
A broadcast domain can be understood as:

**The layer 2 range within which a broadcast packet can travel.**

If the entire switch network is not divided into VLANs, then theoretically, the broadcast domain would continue to expand indefinitely.

### 4.3 Problems Caused by Large Broadcast Domains
Large broadcast domains often lead to:

- An increased range of broadcast traffic.
- Unnecessary hosts receiving broadcast packets.
- Increased network noise.
- A wider impact when faults occur.
- Unclear management boundaries.

Therefore, one of the significant benefits of VLANs is that they allow large broadcast domains to be divided into smaller, more manageable ones.

---

## 5. The Core Value of VLANs

VLANs are not just "test knowledge points" but fundamental layer 2 capabilities in actual networks. Their core value lies in the following aspects:

### 5.1 Layer 2 Isolation
By default, different VLANs cannot communicate directly at the layer 2 level, which provides basic isolation between services.

For example:

- Separating the office network from the server network.
- Isolating the production environment from the testing environment.
- Keeping the management network separate from the service network.
- Separating access networks for different departments.

### 5.2 Reducing Broadcast Domains
Dividing a large layer 2 network into multiple VLANs limits broadcast traffic to within each VLAN, reducing its impact range.

### 5.3 Improving Network Manageability
VLANs help clearly define:

- Which interfaces belong to which service.
- Which users access which network- It is understood that cross-VLAN communication is related to subsequent layer 3 forwarding processes.
- It is also recognized that VLANs serve as a fundamental prerequisite for learning about Access, Trunk, VLANIF, and VXLAN in the future.

If you can currently only grasp the basic concepts and understand their general functions, that is also considered a normal state.

---

## 14. Related External Links

### Huawei Official Documentation
- https://support.huawei.com/enterprise/zh/index.html
- https://info.support.huawei.com/enterprise/zh/doc/

### Network Basics References
- https://www.rfc-editor.org/
- https://www.cisco.com/c/en/us/solutions/small-business/resource-center/networking/what-is-a-vlan.html
- https://www.cloudflare.com/learning/network-layer/what-is-a-vlan/

### VXLAN
- https://www.rfC-editor.org/rfc/rfc7348

---

## 15. Today's Summary

The focus of today's learning was to establish a basic understanding of VLANs and broadcast domains.

The key points that need to be clearly understood include:

- VLANs are a fundamental mechanism for dividing multiple logical layer 2 networks on the same physical switching network.
- A significant role of VLANs is to segment broadcast domains.
- By default, different VLANs cannot communicate directly at the layer 2 level.
- VLANs are commonly used for isolating services, defining network boundaries, and implementing management controls.
- VLANs remain an essential foundation for learning about port types, layer 3 gateways, and VXLAN in the future.

---

## 16. Tomorrow's Plan

Tomorrow, we will focus on learning about Access, Trunk, and Hybrid port types. The key areas to cover include:

- The functions of Access ports.
- The roles of Trunk ports.
- A basic understanding of Hybrid ports.
- Typical usage scenarios for access ports and uplink ports.
- The relationship between VLANs and different port types.