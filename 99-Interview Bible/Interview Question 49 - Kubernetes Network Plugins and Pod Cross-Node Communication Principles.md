---
title: "# Interview Question 29: Kubernetes Networking Plugin and Pod Cross-Node Communication Principles"
tags:
  - The Book of Interviews
  - Kubernetes
  - K8s
  - Network
  - CNI
  - Flannel
  - Calico
  - VXLAN
  - BGP
  - Interview
category: "# Interview Bible"
topic: "# Kubernetes Network"
type: "# Interview Questions"
source: "# Interview Review"
created: 2026-04-20
updated: 2026-04-20
---

---

# Interview Question 49: Kubernetes Network Plugins and Pod Cross-Node Communication Principles

## Question

How does a Pod network connect to the node network in Kubernetes?  
How do Pods on different nodes achieve cross-node communication?  
What are the differences between Flannel and Calico?

---

## Core Answer

In Kubernetes, Pods have independent network namespaces by default and do not directly share the host's network stack.  
After creation, a Pod needs to connect to the node network through a **CNI plugin**.

The typical approach is:

- Create an independent network namespace for the Pod  
- Create a pair of veth interfaces  
- One end is placed inside the Pod as `eth0`  
- The other end remains on the node side, connected to a bridge, routing device, or tunnel device  

This allows Pod traffic to first reach the node, then be forwarded to other Pods, Services, or external networks.

Cross-node Pod communication also relies on CNI plugins, with different plugins having distinct approaches:

- **Flannel**: Typically uses **VXLAN** to implement an Overlay network  
- **Calico**: Typically uses **BGP** to distribute routes, achieving Layer 3 forwarding  

---

## Interview-Ready Version

> In Kubernetes, Pods have independent network namespaces and do not directly use the host's network.  
> After startup, Pods generally connect to the node network via CNI by creating a veth pair, with one end inside the Pod as eth0 and the other end connected to the node's network device.  
> Same-node Pod communication typically uses bridge or routing, while cross-node communication is handled by the network plugin.  
> Flannel usually uses VXLAN for encapsulation and forwarding, while Calico typically uses BGP for route propagation.

---

## One: Why Pods Need to Connect to Node Networks

Pods have their own:

- IP address  
- Network interface  
- Routing table  
- DNS configuration  

But all of these exist within the Pod's own network namespace.  
Without a mechanism to connect the Pod to the node network, packets sent by the Pod cannot enter the host's network stack and thus cannot access other Pods, Services, or external networks.

So more accurately, it's not simply "connecting the Pod and host network," but rather:

> **Pods need to connect to the node network path through network devices and kernel forwarding mechanisms.**

---

## Two: How Pods Connect to Node Networks

### 1. Network Namespace

Linux provides network namespaces to isolate network environments.  
In Kubernetes, Pods have independent network namespaces by default.

This means the Pod sees its own:

- Network interface  
- IP  
- Routing  
- `/etc/resolv.conf`  

It does not directly use the host's network.

---

### 2. veth Pair

To establish a connection between the Pod and the node, a pair of **veth interfaces** is typically created.

You can think of it as the two ends of a "virtual Ethernet cable":

- One end enters the Pod, serving as `eth0`  
- The other end remains on the node side  

Traffic from the Pod exits through the veth on the Pod side, enters the veth on the node side, and is then processed by the node's network.

---

### 3. Node-Side Network Device

The veth on the node side is usually not standalone but is connected to some node network infrastructure, such as:

- bridge (e.g., `cni0`)  
- Routing device  
- VXLAN device  
- Virtual interfaces and routing system created by Calico  

This work is handled by the specific **CNI plugin**.

---

## Three: What is CNI

CNI stands for **Container Network Interface**.

It is a standard interface for container networking, responsible for:

- Assigning IP addresses to Pods  
- Creating network devices  
- Connecting Pods to the node network  
- Configuring bridge, route, tunnel, and other network capabilities  

Kubernetes itself does not directly implement Pod networking but delegates this capability to network plugins via CNI.

Common network plugins include:

- Flannel  
- Calico  
- Cilium  

---

## Four: Relationship Between Pods and Host Networks

### Default Scenario

By default:

- Pods have independent network namespaces  
- Pods have their own Pod IP  
- Pods do not share the host's network stack  
- But Pods must connect to the node network via veth + CNI  

So the default relationship is:

> **Isolated but connected.**

---

### hostNetwork Mode

If a Pod is set with:

    hostNetwork: true  

Then the Pod directly uses the host's network stack.

At this point:

- There is no independent Pod network isolation  
- The Pod directly uses the host's IP and ports  
- Port conflicts are easy to occur  

So it's crucial to distinguish between:

- **Normal Pod**: Independent network space, connected to the node network via veth  
- **hostNetwork Pod**: Directly shares the host's network stack  

---

## Five: Same-Node Pod Communication

If two Pods are on the same node, the communication path is typically simple:

    PodA -> veth -> bridge/route -> veth -> PodB  

In other words:

- Both Pods are connected to the same node network  
- The node internally forwards traffic via bridge or routing  
- No need for cross-node network  
- Usually no additional encapsulation is required  

---

## Six: Why Cross-Node Communication is a Problem

If PodA is on Node1 and PodB is on Node2, Node1 must know:

- Whether the target Pod IP is local  
- If not, which node to forward the packet to  

This is the core issue of cross-node Pod communication:

> **How can Pod subnets on different nodes be mutually reachable.**

This is primarily resolved by network plugins like Flannel and Calico.

---

## Seven: Flannel's Implementation Approach

Flannel is a common and relatively easy-to-understand solution.

### Core Idea

Flannel leans more toward **Overlay networks**.

It assigns each node a Pod subnet, for example:

- Node1: 10.244.1.0/24  
- Node2: 10.244.2.0/24  

When a Pod on Node1 accesses a Pod on Node2:

1. Node1 discovers the target Pod IP is not in its local subnet  
2. Flannel encapsulates the original Pod packet  
3. Sends the encapsulated packet through the host's network to Node2  
4. Node2 receives it, decapsulates it  
5. Then forwards the original packet to the target Pod  

This approach commonly uses **VXLAN**.

---

### Understanding VXLAN

VXLAN can be simply understood as:

> **Encapsulating the original Pod packet with an outer network header, allowing it to travel through the host's network to the target node, then decapsulating it.**

So the best way to remember Flannel is:

> **Pod networks cannot directly cross nodes, so first encapsulate, then use the host network for transmission.**

---

### Flannel Features

Advantages: /think

- Intuitive implementation approach  
- Simple deployment  
- Does not require the underlying network to directly recognize Pod IPs  

Disadvantages:  

- Encapsulation and decapsulation overhead  
- Performance is typically worse than pure routing solutions  

---  

## Eight. Calico's Implementation Approach  

Calico's approach differs from Flannel.  

### Core Concept  

Calico leans more toward a **layer 3 routing solution**.  

It aims to inform the underlying network about:  

- Which Pod subnet resides on which node  
- Which route to take when accessing a target Pod subnet  

In other words:  

> **It does not encapsulate packets and send them, but instead directly tells the node "how to reach the target subnet."**  

---  

### BGP's Role in Calico  

BGP is essentially a routing protocol.  

In the Calico scenario, it can be simply understood as:  

> **Nodes exchange routing information about their responsible Pod subnets via BGP.**  

This allows each node to know:  

- Which Pod subnet belongs to which node  
- Who the next hop is for accessing this subnet  

Thus, data packets can be directly forwarded according to the routing table, without necessarily requiring encapsulation.  

---  

### Calico Features  

Advantages:  

- Better performance when no encapsulation is used  
- More direct routing logic  
- More suitable for scenarios with high network performance requirements  

Disadvantages:  

- Higher requirements for understanding networks and routing  
- Deployment and troubleshooting are typically more complex than simple Overlay solutions  

---  

## Nine. Core Differences Between Flannel and Calico  

### The Easiest-to-Remember Summary  

- **Flannel: Encapsulate and send**  
- **Calico: Directly tell you how to reach the destination**  

---  

### Comparison Summary  

#### Flannel  
- Biases toward Overlay  
- Common implementation: VXLAN  
- Nodes communicate via encapsulation  
- Underlying network does not need to recognize Pod IPs  

#### Calico  
- Biases toward layer 3 routing  
- Common implementation: BGP  
- Nodes communicate via routing propagation  
- Can avoid encapsulation  

---  

## Ten. Common Data Flow Paths  

### 1. Pod Accessing External Network  

    Pod -> eth0 -> veth -> Node network stack -> Routing/NAT -> External network  

If accessing outside the cluster, it typically involves:  

- iptables  
- SNAT / MASQUERADE  

---  

### 2. Pod Accessing Another Pod on the Same Node  

    PodA -> veth -> bridge/route -> veth -> PodB  

---  

### 3. Pod Accessing Another Pod on a Different Node (Flannel)  

    PodA -> veth -> Node1  
         -> VXLAN encapsulation  
         -> Host network  
         -> Node2 decapsulation  
         -> PodB  

---  

### 4. Pod Accessing Another Pod on a Different Node (Calico)  

    PodA -> veth -> Node1  
         -> Route lookup to Node2  
         -> Direct forwarding between nodes  
         -> PodB  

---  

## Eleven. Common Follow-up Questions  

### 1. Is it mandatory for Pods to share the host network?  

No.  

By default:  

- Pods have independent network namespaces  
- Access the node network via CNI and veth  
- Do not directly share the host network stack  

Only `hostNetwork` shares the host network.  

---  

### 2. Why can Pods access the external network?  

Because Pod traffic first enters the node network stack, then gets forwarded via routing and NAT.  

---  

### 3. Why is Flannel, Calico, and BGP mentioned in interviews?  

Because they correspond to **how Kubernetes implements cross-node Pod networking**, not **how Docker configures network**.  

> **Kubernetes cross-node Pod networking implementation.**  

---  

### 4. What is the fundamental difference between Flannel and Calico?  

You can directly answer:  

**Flannel primarily uses Overlay logic, encapsulating packets via VXLAN; Calico primarily uses routing logic, propagating Pod subnet routes via BGP.**  

---  

## Twelve. Why This Question Is Easy to Misanswer  

Because interviews often confuse two layers:  

### First Layer: How Pods Access the Node Network  
Keywords:  

- netns  
- veth pair  
- bridge  
- CNI  

### Second Layer: How Pods Communicate Across Nodes  
Keywords:  

- Flannel  
- Calico  
- VXLAN  
- BGP  

If you don't separate these two layers, you might answer from the "usage layer" (e.g., Docker network, host network), which mismatches the interview's focus on the "implementation layer."  

---  

## Thirteen. One-Sentence Summary  

In Kubernetes, Pods default to having independent network namespaces, accessing the node network via CNI and veth pairs; same-node Pod communication typically uses bridge or routing; cross-node communication is handled by network plugins, with Flannel commonly using VXLAN encapsulation and Calico using BGP route propagation.  

---  

## Fourteen. What Does This Question Test?  

- Linux networking basics  
- network namespace  
- veth pair  
- CNI's role  
- Pod networking model  
- Differences between same-node and cross-node communication  
- Implementation approaches of Flannel and Calico  
- Basic meanings of VXLAN and BGP  

---  

## Associated Tags  
#TheBookOfInterviews #Kubernetes #K8s #Network #CNI #Flannel #Calico #VXLAN #BGP #Interview