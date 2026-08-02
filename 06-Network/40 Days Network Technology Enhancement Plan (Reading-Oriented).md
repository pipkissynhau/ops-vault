# 40-Day Network Technology Reinforcement Plan (Reading-Focused)

## Obsidian Tags
#Network #China #DataCentreNetwork #TechnicalEnhancements #VLAN #OSPF #BGP #VXLAN #RouteIntroduction #SDN #KnowledgeBase

---

## Document Positioning

This plan targets reading-focused reinforcement for data center network technology.

Applicable scenarios include:

- Current phase focuses on reading notes, system restoration, and foundational understanding
- No actual operation is assumed as a prerequisite
- Notes retain Huawei common commands, configuration skeletons, and troubleshooting approaches
- Goal is to restore and strengthen the following capabilities:
  - Basic network awareness
  - Understanding of data center Underlay / Overlay
  - OSPF / BGP / VXLAN core capabilities
  - Understanding of route introduction and data center route propagation
  - Systematic understanding of SDN

---

## Learning Objectives

After 40 days, the expected state should include:

- Basic network knowledge fully warmed up
- Ability to understand VLAN / Trunk / Gateway / Routing Table / ARP / MAC related content
- Ability to understand basic OSPF / BGP configuration and common viewing commands
- Ability to comprehend route introduction and data center route propagation control logic
- Ability to clearly understand the relationship between VXLAN, EVPN, Underlay, and Overlay
- Ability to place existing SDN product experience within a more standardized network framework for understanding

---

## Phase Division

### Phase 1: Foundation Warming (Day1–Day10)
### Phase 2: OSPF Mainline (Day11–Day18)
### Phase 3: Route Introduction Mainline (Day19–Day24)
### Phase 4: BGP Mainline (Day25–Day31)
### Phase 5: VXLAN / EVPN Mainline (Day32–Day37)
### Phase 6: SDN and Systematic Consolidation (Day38–Day40)

---

# Phase 1: Foundation Warming (Day1–Day10)

## Day1: Overall Understanding of Data Center Networks
Learning Tasks:
- Difference between Layer 2 and Layer 3 networks
- Concept of Underlay
- Concept of Overlay
- Basic trend of three-tiered data center networks

Learning Objectives:
- Establish the overall framework for the next 40 days of learning
- Clarify the approximate hierarchical position of OSPF / BGP / VXLAN / SDN

Next Day Plan:
- Study VLAN basics and broadcast domain concepts

---

## Day2: VLAN Basics
Learning Tasks:
- Purpose of VLAN
- Concept of broadcast domain
- Necessity of VLAN isolation in switching networks
- Typical uses of VLAN in business networks

Learning Objectives:
- Understand VLAN as the foundational capability for Layer 2 isolation
- Establish prerequisite understanding for VXLAN and VNI

Next Day Plan:
- Study Access / Trunk / Hybrid port types

---

## Day3: Access / Trunk / Hybrid Port Types
Learning Tasks:
- Purpose of Access ports
- Purpose of Trunk ports
- Basic understanding of Hybrid ports
- Typical scenarios for uplink and server connection ports on switches

Learning Objectives:
- Understand switch port roles
- Recognize common locations for VLAN-related issues in business communication failures

Next Day Plan:
- Study MAC address table and Layer 2 forwarding process

---

## Day4: MAC Address Table and Layer 2 Forwarding
Learning Tasks:
- Purpose of MAC addresses
- Basic process of MAC address learning on switches
- Basic Layer 2 switching forwarding logic
- Basic understanding of unknown unicast, broadcast, and flooding

Learning Objectives:
- Understand the flow of Layer 2 packets in switching networks
- Establish foundation for understanding BUM traffic in VXLAN

Next Day Plan:
- Study ARP and IP-to-MAC resolution process

---

## Day5: ARP Basics
Learning Tasks:
- Purpose of ARP
- Reason for ARP resolution before Layer 3 forwarding
- Basic understanding of ARP tables
- Common manifestations of ARP anomalies

Learning Objectives:
- Understand the dependency of IP communication on ARP
- Establish foundational understanding of the interrelation between Layer 2 and Layer 3 issues

Next Day Plan:
- Study VLANIF / Layer 3 interfaces / gateway basics

---

## Day6: VLANIF, Layer 3 Interfaces, and Gateway
Learning Tasks:
- Purpose of Layer 3 interfaces
- Purpose of VLANIF
- Purpose of gateway
- Reason for cross-subnet communication through gateway

Learning Objectives:
- Understand the transition logic from Layer 2 switching to Layer 3 forwarding
- Prepare for subsequent OSPF and static routing studies

Next Day Plan:
- Study static routing and default routing basics

---

## Day7: Static Routing and Default Routing
Learning Tasks:
- Definition of static routing
- Definition of default routing
- Concept of next-hop
- Basic way routing tables guide forwarding

Learning Objectives:
- Understand routing tables as the core basis for network forwarding
- Establish foundation for subsequent route introduction studies

Next Day Plan:
- Study routing table viewing and basic interpretation

---

## Day8: Routing Table Basic Interpretation
Learning Tasks:
- Meaning of common fields in routing tables
- Differences between direct, static, and dynamic routes
- Basic methods to determine path reachability from routing tables
- Position and role of default routes in routing tables

Learning Objectives:
- Develop basic reading capability for routing tables
- Understand why routing tables are prioritized in troubleshooting

Next Day Plan:
- Study basic troubleshooting approaches for Layer 2 and Layer 3 issues

---

## Day9: Basic Troubleshooting for Layer 2 and Layer 3 Issues
Learning Tasks:
- Basic judgment of interface up/down status
- Typical phenomena of VLAN errors
- Problem directions corresponding to MAC not learned, ARP not reachable, and route missing
- Basic methods to distinguish Layer 2 vs. Layer 3 issues

Learning Objectives:
- Establish basic troubleshooting sequence
- Build methodological foundation for subsequent protocol troubleshooting

Next Day Plan:
- Study Huawei basic viewing command quick reference

---

## Day10: Huawei Basic Viewing Command Warming
Learning Tasks:
- `display interface brief`
- `display vlan`
- `display mac-address`
- `display arp`
- `display ip interface brief`
- `display ip routing-table`

Learning Objectives:
- Restore command recognition capability
- Prepare for subsequent OSPF / BGP / VXLAN reading

Next Day Plan:
- Begin entering OSPF mainline

---

# Phase 2: OSPF Mainline (Day11–Day18)

## Day11: Purpose and Positioning of OSPF
Learning Tasks:
- Limitations of static routing
- Role of OSPF as an IGP
- Why OSPF is suitable for internal networks
- Common positions of OSPF in data center internals

Learning Objectives:
- Clarify that OSPF is one of the common foundational protocols for Underlay

Next Day Plan:
- Study OSPF core terminology

---

## Day12: OSPF Core Concepts
Learning Tasks:
- Router ID
- Area
- Neighbor and adjacency
- LSDB
- LSA
- SPF
- Cost

Learning Objectives:
- Establish OSPF terminology system
- Understand the basic composition structure of OSPF

Next Day Plan:
- Study OSPF neighbor establishment process

---

## Day13: OSPF Neighbor Relationships and State Machine
Learning Tasks:
- Down / Init / 2-Way / ExStart / Exchange / Loading / Full
- Difference between Neighbor and Adjacency
- Why neighbor states may not reach Full

Learning Objectives:
- Understand the OSPF neighbor establishment process
- Build foundational knowledge for OSPF troubleshooting

Next Day Plan:
- Learn about OSPF LSA, LSDB, and SPF Calculation

---

## Day14: LSA, LSDB, and SPF Calculation
Learning Tasks:
- Basic concepts of Link State
- Role of LSDB
- Basic process of SPF
- Core logic of OSPF as a link-state protocol

Learning Objectives:
- Understand the fundamental principles of OSPF's shortest path calculation

Next Day Plan:
- Learn about OSPF Area and Area Division

---

## Day15: OSPF Area Basics
Learning Tasks:
- Backbone Area
- Regular Area
- Reasons for dividing Areas
- Basic concepts of Stub / NSSA

Learning Objectives:
- Establish foundational awareness of OSPF's scalable design

Next Day Plan:
- Learn about OSPF basic configuration structure

---

## Day16: OSPF Basic Configuration Structure (Huawei Direction)
Learning Tasks:
- `ospf 1`
- `area 0`
- `network ...`
- Basic approach to declaring interfaces into OSPF
- Configuration structure reading

Learning Objectives:
- Understand the basic configuration structure of OSPF

Next Day Plan:
- Learn about OSPF viewing commands and interpretation

---

## Day17: Common OSPF Viewing Commands (Huawei Direction)
Learning Tasks:
- `display ospf peer`
- `display ospf interface`
- `display ospf routing`
- `display ip routing-table`

Learning Objectives:
- Clarify main viewing entry points for OSPF troubleshooting

Next Day Plan:
- Learn about common OSPF faults

---

## Day18: Common OSPF Faults and Troubleshooting Order
Learning Tasks:
- Neighbor not forming
- Area inconsistency
- Authentication inconsistency
- MTU inconsistency
- Router ID conflict
- Routes not entering routing table

Learning Objectives:
- Establish OSPF troubleshooting mainline:
  - Check neighbors first
  - Then check interfaces
  - Finally check routes

Next Day Plan:
- Begin entering the routing introduction mainline

---

# Phase 3: Routing Introduction Mainline (Day19–Day24)

## Day19: Routing Introduction Basics
Learning Tasks:
- Definition of routing introduction
- Necessity of routing introduction
- Why routing protocols cannot natively share all routes
- Relationship between routing introduction and routing distribution

Learning Objectives:
- Understand routing introduction as protocol interconnection capability, not an isolated command

Next Day Plan:
- Learn about routing introduction in OSPF

---

## Day20: Routing Introduction in OSPF
Learning Tasks:
- Introducing directly connected routes into OSPF
- Introducing static routes into OSPF
- `import-route direct`
- `import-route static`
- Basic impact of metric

Learning Objectives:
- Understand the sources of non-OSPF routes in OSPF

Next Day Plan:
- Learn about routing distribution and introduction in BGP

---

## Day21: Routing Distribution and Introduction Basics in BGP
Learning Tasks:
- `network ...`
- `import-route direct`
- `import-route static`
- `import-route ospf`
- Basic distinction between `network` and `import-route`

Learning Objectives:
- Establish preliminary understanding of BGP route sources

Next Day Plan:
- Learn about routing introduction boundaries in data centers

---

## Day22: Routing Introduction in Data Centers
Learning Tasks:
- Difference between Underlay routing and business routing
- Reasons not to arbitrarily introduce routes
- Basic boundary of default route distribution
- Risks of bidirectional introduction

Learning Objectives:
- Establish awareness of controlling the scope of routing introduction

Next Day Plan:
- Learn about route-policy / filtering basics

---

## Day23: Routing Policy and Filtering Basics
Learning Tasks:
- Role of route-policy
- Role of prefix-list
- Importance of route filtering
- Basic understanding of route tagging and loop risks

Learning Objectives:
- Clarify that routing introduction is not just a command issue, but a control issue

Next Day Plan:
- Learn about common routing introduction faults and design pitfalls

---

## Day24: Common Routing Introduction Faults and Pitfalls
Learning Tasks:
- Route table inflation
- Route leakage
- Suboptimal paths
- Loop caused by bidirectional introduction
- Uncontrolled default route propagation

Learning Objectives:
- Establish systematic understanding of routing introduction risks

Next Day Plan:
- Begin entering the BGP mainline

---

# Phase 4: BGP Mainline (Day25–Day31)

## Day25: Role and Positioning of BGP
Learning Tasks:
- Boundary of OSPF
- Why BGP is suitable for large-scale networks
- Role of BGP in data centers and edge networks
- Difference in positioning between BGP and IGP

Learning Objectives:
- Understand that BGP is more than just another dynamic routing protocol

Next Day Plan:
- Learn about BGP core concepts

---

## Day26: BGP Core Concepts
Learning Tasks:
- AS
- EBGP / IBGP
- Peer
- Update
- Next-Hop
- Path Attribute

Learning Objectives:
- Establish foundational terminology for BGP

Next Day Plan:
- Learn about BGP neighbor establishment and basic configuration structure

---

## Day27: BGP Neighbor Establishment and Basic Configuration Structure
Learning Tasks:
- `bgp <as-number>`
- `peer x.x.x.x as-number ...`
- Basic conditions for neighbor establishment
- Basic understanding of Loopback neighbor establishment

Learning Objectives:
- Understand the basic configuration structure of BGP

Next Day Plan:
- Learn about BGP route distribution and propagation

---

## Day28: BGP Route Distribution and Propagation
Learning Tasks:
- `network ...`
- Basic methods of route distribution
- Process of peer learning routes
- Common reasons why peer fails to learn routes when local routes exist

Learning Objectives:
- Establish awareness that "neighbor establishment is just a prerequisite, propagation is the key"

Next Day Plan:
- Learn about BGP route attributes

---

## Day29: BGP Route Attributes
Learning Tasks:
- AS Path
- Local Preference
- MED
- Next-Hop
- Origin

Learning Objectives:
- Understand the basic influencing factors of BGP route selection

Next Day Plan:
- Learn about BGP viewing commands and interpretation

---

## Day30: Common BGP Viewing Commands (Huawei Direction)
Learning Tasks:
- `display bgp peer`
- `display bgp routing-table`
- `display ip routing-table`
- Basic interpretation of peer status, route status, and next-hop

Learning Objectives:
- Establish basic interpretation capabilities for BGP

Next Day Plan:
- Learn about common BGP faults and troubleshooting order

---

## Day31: Common BGP Faults and Troubleshooting Order
Learning Tasks:
- Neighbor not forming
- Routes not learned
- Next-hop unreachable
- Policy filtering
- Routes exist but business not reachable

Learning Objectives:
- Establish BGP troubleshooting mainline:
  - First check peer
  - Then check routing
  - Then check next-hop
  - Finally check policies

Next Day Plan:
- Start entering VXLAN mainline

---

# Phase 5: VXLAN / EVPN Mainline (Day32–Day37)

## Day32: Background and Positioning of VXLAN
Learning Tasks:
- Limitations of VLAN
- Source of demand for Layer 2 across Layer 3
- Demand for multi-tenancy and large-scale virtual networks
- VXLAN as Overlay positioning

Learning Objectives:
- Clarify that VXLAN is Overlay technology, not simply replacing VLAN

Next Day Plan:
- Learn core terminology of VXLAN

---

## Day33: Core Concepts of VXLAN
Learning Tasks:
- Overlay / Underlay
- VTEP
- VNI
- Tunnel
- MAC-in-UDP
- Basic understanding of BUM traffic

Learning Objectives:
- Establish key terminology system of VXLAN

Next Day Plan:
- Learn VXLAN traffic encapsulation and forwarding process

---

## Day34: VXLAN Traffic Process
Learning Tasks:
- Local host sends Layer 2 frame
- Local VTEP encapsulates
- Underlay forwards based on outer IP
- Remote VTEP decapsulates
- Target host receives packet

Learning Objectives:
- Mentally connect the complete VXLAN forwarding chain

Next Day Plan:
- Learn basic configuration framework of VXLAN

---

## Day35: Basic Configuration Framework of VXLAN (Huawei Direction)
Learning Tasks:
- Underlay first connected
- VTEP address
- VLAN and VNI mapping
- Relationship with remote VTEP
- Verification approach

Learning Objectives:
- Understand VXLAN configuration sequence and dependencies

Next Day Plan:
- Learn common VXLAN faults and troubleshooting order

---

## Day36: Common VXLAN Faults and Troubleshooting Order
Learning Tasks:
- VTEP not connected
- VNI mapping error
- VLAN interface error
- Underlay normal but Overlay not connected
- MAC learning anomaly

Learning Objectives:
- Establish VXLAN troubleshooting mainline:
  - First check Underlay
  - Then check mapping
  - Then check VTEP
  - Finally check forwarding table

Next Day Plan:
- Learn relationship between EVPN and VXLAN

---

## Day37: Relationship between EVPN and VXLAN
Learning Tasks:
- VXLAN as data plane
- EVPN / BGP as control plane
- Reasons for common EVPN + VXLAN in production
- Scaling boundaries of static VXLAN

Learning Objectives:
- Establish further understanding of data center Overlay architecture

Next Day Plan:
- Start entering SDN and systematized consolidation phase

---

# Phase 6: SDN and Systematized Consolidation (Day38–Day40)

## Day38: Revisiting SDN from Existing Product Experience
Learning Tasks:
- Controller
- Network management platform
- Tunnel
- Encryption
- NAT plane
- Business release

Learning Objectives:
- Translate existing product experience into standard network structure understanding

Next Day Plan:
- Learn relationship between SDN and OSPF / BGP / VXLAN system

---

## Day39: Relationship between SDN and Data Center Network System
Learning Tasks:
- Role of SDN control layer
- Functions of OSPF / BGP / VXLAN
- Reasons for vendors integrating these capabilities
- SDN as higher-level control and orchestration, not replacing protocols

Learning Objectives:
- Integrate previous learning into overall network architecture understanding

Next Day Plan:
- Conduct 40-day comprehensive review

---

## Day40: 40-Day Comprehensive Review
Learning Tasks:
- Review basic network mainline
- Review OSPF mainline
- Review routing introduction mainline
- Review BGP mainline
- Review VXLAN / EVPN mainline
- Review SDN system understanding

Learning Objectives:
- Form complete knowledge map
- Establish entry point for next stage deep learning

Next Day Plan:
- Next stage optional directions:
  - OSPF advancement
  - BGP advancement
  - VXLAN / EVPN advancement
  - Huawei command topic compilation
  - Data center network fault case topic

---

## Recommended External Links

### Huawei Official Documentation Entry
- https://support.huawei.com/enterprise/zh/index.html
- https://info.support.huawei.com/enterprise/zh/doc/

### OSPF
- https://www.rfc-editor.org/rfc/rfc2328

### BGP
- https://www.rfc-editor.org/rfc/rfc4271

### VXLAN
- https://www.rfc-editor.org/rfc/rfc7348

### EVPN
- https://www.rfc-editor.org/rfc/rfc7432

### SDN
- https://opennetworking.org/sdn-definition/

---

## Usage Suggestions

Recommended way to use this plan:

- Read one day's content daily
- Not aim for complete memorization
- Prioritize "understanding, identification, and connection"
- Focus on understanding command purpose when reading commands
- Focus on structure and sequence when reading configurations

---

## Summary

This plan is used to re-systematize, structure, and knowledge-base existing but partially rusty data center network knowledge, providing a stable foundation for subsequent deep learning in directions like OSPF, BGP, VXLAN, EVPN, and SDN.