# 40-Day Network Technology Enhancement Plan (Reading-oriented)

## Obsidian Tags
#Network #Huawei Datacom #Data Center Networking #Technical Enhancement #VLAN #OSPF #BGP #VXLAN #Route Introduction #SDN #Knowledge Base

---

## Document Purpose

This plan is designed for reading-based technical enhancement in the field of data center networking.

Applicable scenarios include:

- Currently focusing on reading notes, restoring knowledge systems, and enhancing foundational understanding.
- Not requiring practical operations at this stage.
- Including common Huawei commands, configuration frameworks, and troubleshooting approaches in the notes.
- The goal is to restore and strengthen the following abilities:
  - Basic network awareness
  - Understanding of data center Underlay/Overlay networks
  - Core skills in OSPF/BGP/VXLAN
  - Knowledge of route introduction and routing propagation in data centers
  - Systematic understanding of SDN

---

## Overall Learning Objectives

After 40 days, the following goals are expected to be achieved:

- Revisit basic network knowledge
- Understand concepts related to VLANs, trunks, gateways, route tables, ARP, and MAC addresses.
- Be able to configure and check basic OSPF/BGP commands.
- Comprehend the principles of route introduction and routing propagation in data centers.
- Clearly understand the relationships between VXLAN, EVPN, Underlay, and Overlay.
- Apply existing SDN product experience within a more standardized network framework.

---

## Phase Division

### Phase 1: Basic Review (Day1–Day10)
### Phase 2: OSPF Core Skills (Day11–Day18)
### Phase 3: Route Introduction Basics (Day19–Day24)
### Phase 4: BGP Core Skills (Day25–Day31)
### Phase 5: VXLAN/EVPN Core Skills (Day32–Day37)
### Phase 6: SDN and Overall Integration (Day38–Day40)

---

# Phase 1: Basic Review (Day1–Day10)

## Day1: Overall Understanding of Data Center Networking
Learning tasks:
- Differences between layer 2 and layer 3 networks
- Concepts of Underlay and Overlay
- General trends in data center network layering

Learning objectives:
- Establish a framework for the subsequent 40 days of learning.
- Clarify the general hierarchy of OSPF/BGP/VXLAN/SDN.

Next day's plan:
- Learn about VLAN basics and broadcast domains.

---

## Day2: VLAN Basics
Learning tasks:
- Functions of VLANs
- Concept of broadcast domains
- Importance of VLAN isolation in switching networks
- Typical uses of VLANs in service networks

Learning objectives:
- Understand that VLANs are fundamental for layer 2 isolation.
- Lay the foundation for understanding VXLAN and VNI later on.

Next day's plan:
- Learn about Access/Trunk/Hybrid port types.

---

## Day3: Access/Trunk/Hybrid Port Types
Learning tasks:
- Functions of Access ports
- Functions of Trunk ports
- Basic concepts of Hybrid ports
- Typical scenarios for connecting switches to servers

Learning objectives:
- Understand the roles of switch interfaces.
- Identify common issues with VLAN connectivity in service networks.

Next day's plan:
- Learn about MAC address tables and layer 2 forwarding processes.

---

## Day4: MAC Address Tables and Layer 2 Forwarding
Learning tasks:
- Functions of MAC addresses
- How switches learn MAC addresses
- Basic logic of layer 2 switching
- Understanding of unknown unicast, broadcast, and flood packets

Learning objectives:
- Comprehend how layer 2 packets are forwarded in switching networks.
- Lay the foundation for understanding BUM traffic in VXLAN.

Next day's plan:
- Learn about ARP and the process of resolving IP addresses to MAC addresses.

---

## Day5: ARP Basics
Learning tasks:
- Functions of ARP
- Reason for performing ARP resolution before layer 3 forwarding
- Basic concepts of the ARP table
- Common manifestations of ARP anomalies

Learning objectives:
- Understand the interdependence between IP communication and ARP.
- Establish a connection between layer 2 and layer 3 issues.

Next day's plan:
- Learn about VLANIF, layer 3 interfaces, and gateways.

---

## Day6: VLANIF, Layer 3 Interfaces, and Gateways
Learning tasks:
- Functions of layer 3 interfaces
- Functions of VLANIF
- Roles of gateways
- Reason why cross-segment communication goes through gateways

Learning objectives:
- Understand the transition from layer 2 switching to layer 3 forwarding.
- Prepare for learning OSPF and static routing later on.

Next day's plan:
- Learn about static routing and default routes.

---

## Day7: Static Routing and Default Routes
Learning tasks:
- Definition of static routing
- Definition of default routes
- Concept of the next hop
- Basic### The Role of Route-Policy
### The Function of Prefix-List
### The Importance of Route Filtering
### Basic Understanding of Routing Tags and Loop Risks

**Learning Objectives:**
- Understand that route introduction is not just a matter of commands but also involves control.

**Plan for Tomorrow:**
- Study common faults and design misconceptions in route introduction.

---

## Day24: Common Faults and Misconceptions in Route Introduction
**Learning Tasks:**
- Route table expansion
- Route leakage
- Suboptimal paths
- Loops caused by bidirectional introduction
- Uncontrolled propagation of default routes

**Learning Objectives:**
- Develop a systematic understanding of the risks associated with route introduction.

**Plan for Tomorrow:**
- Begin to explore the main aspects of BGP.

---

# Phase 4: BGP Mainline (Day25–Day31)

## Day25: The Role and Positioning of BGP
**Learning Tasks:**
- The role of OSPF at borders
- Reasons why BGP is suitable for large-scale networks
- The role of BGP in data centers and border networks
- Differences in positioning between BGP and IGP

**Learning Objectives:**
- Understand that BGP is more than just another dynamic routing protocol.

**Plan for Tomorrow:**
- Study the core concepts of BGP.

---

## Day26: Core Concepts of BGP
**Learning Tasks:**
- AS
- EBGP / IBGP
- Peer
- Update
- Next-Hop
- Path Attribute

**Learning Objectives:**
- Establish a basic terminology system for BGP.

**Plan for Tomorrow:**
- Study the establishment of BGP neighbors and the basic configuration framework.

---

## Day27: Establishment of BGP Neighbors and Basic Configuration Framework
**Learning Tasks:**
- `bgp <as-number>`
- `peer x.x.x.x as-number ...`
- Basic conditions for establishing neighbors
- Basic understanding of using Loopback for neighbor establishment

**Learning Objectives:**
- Understand the basic configuration structure of BGP.

**Plan for Tomorrow:**
- Study route publication and propagation in BGP.

---

## Day28: Route Publication and Propagation in BGP
**Learning Tasks:**
- `network ...`
- Basic methods of route publication
- The process by which peers learn routes
- Common reasons why a route exists locally but is not learned by the peer

**Learning Objectives:**
- Establish the understanding that "establishing neighbors is just the prerequisite; propagation is the key."

**Plan for Tomorrow:**
- Study BGP route attributes.

---

## Day29: BGP Route Attributes
**Learning Tasks:**
- AS Path
- Local Preference
- MED
- Next-Hop
- Origin

**Learning Objectives:**
- Understand the basic factors that influence BGP routing decisions.

**Plan for Tomorrow:**
- Study BGP viewing commands and how to interpret them.

---

## Day30: Common BGP Viewing Commands (Huawei Perspective)
**Learning Tasks:**
- `display bgp peer`
- `display bgp routing-table`
- `display ip routing-table`
- Basic interpretation of peer status, route status, and next-hop

**Learning Objectives:**
- Develop basic skills in interpreting BGP information.

**Plan for Tomorrow:**
- Study common faults in BGP and the order of troubleshooting them.

---

## Day31: Common Faults in BGP and the Order of Troubleshooting
**Learning Tasks:**
- Failure to establish neighbors
- Routes not being learned
- Next-hop unreachable
- Policy filtering issues
- Routes existing but services not functioning

**Learning Objectives:**
- Establish a troubleshooting approach for BGP:
  - Check peers first
  - Then check routes
  - Next, check next-hop
  - Finally, check policies

**Plan for Tomorrow:**
- Begin to explore VXLAN mainline.

---

# Phase 5: VXLAN / EVPN Mainline (Day32–Day37)

## Day32: The Background and Positioning of VXLAN
**Learning Tasks:**
- Limitations of VLAN
- Reasons for the need to cross layers between layer 2 and layer 3
- Requirements for multi-tenancy and large-scale virtual networks
- The positioning of VXLAN as an overlay technology

**Learning Objectives:**
- Understand that VXLAN is an overlay technology, not a simple replacement for VLAN.

**Plan for Tomorrow:**
- Study the core terminology of VXLAN.

---

## Day33: Core Concepts of VXLAN
**Learning Tasks:**
- Overlay / Underlay
- VTEP
- VNI
- Tunnel
- MAC-in-UDP
- Basic understanding of BUM traffic

**Learning Objectives:**
- Establish a key terminology system for VXLAN.

**Plan for Tomorrow:**
- Study the process