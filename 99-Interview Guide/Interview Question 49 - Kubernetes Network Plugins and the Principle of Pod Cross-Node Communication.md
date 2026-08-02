---
title: Interview Question 29: Kubernetes Network Plugins and the Principle of Pod Cross-Node Communication
tags:
  - Interview Guide
  - Kubernetes
  - K8s
  - Network
  - CNI
  - Flannel
  - Calico
  - VXLAN
  - BGP
  - Interview Questions
category: Interview Guide
topic: Kubernetes Networking
type: Interview Question
source: Interview Review
created: 2026-04-20
updated: 2026-04-20
---

---

# Interview Question 49: Kubernetes Network Plugins and the Principle of Pod Cross-Node Communication

## Question

How do Pods in Kubernetes access the node network?  
How do Pods on different nodes communicate across nodes?  
What are the differences between Flannel and Calico?

---

## Core Answer

In Kubernetes, Pods have an independent network namespace by default and do not directly share the host machine's network stack.  
After a Pod is created, it needs to access the node network through a **CNI plugin**.

Typical practices include:

- Creating an independent network namespace for the Pod
- Generating a pair of veth pairs
- Placing one end inside the Pod as the Pod's `eth0`
- Leaving the other end on the node side, connecting it to a bridge, routing device, or tunneling device

This way, Pod traffic can first reach the node and then be forwarded to other Pods, Services, or external networks.

Pod cross-node communication also relies on CNI plugins, which differ in their implementation:

- **Flannel**: Typically uses **VXLAN** to create an Overlay network.
- **Calico**: Usually employs **BGP** for routing to achieve layer 3 forwarding.

---

## Interview Answer Version

> In Kubernetes, Pods have an independent network namespace and do not use the host machine's network directly.  
> After a Pod starts, it typically uses CNI to create a veth pair to connect to the node network. One end of the pair becomes the Pod's `eth0`, while the other end is connected to the node's network devices.  
> Communication between Pods on the same node usually occurs through bridges or routing, while cross-node communication is handled by network plugins.  
> Flannel typically uses VXLAN for encapsulation and forwarding, while Calico uses BGP for route propagation.

---

## I. Why Do Pods Need to Access the Node Network

Although Pods have their own:

- IP address
- Network card
- Route table
- DNS configuration

These are all within the Pod's own network namespace.  
Without a mechanism to connect the Pod to the node network, data packets sent by the Pod cannot enter the host machine's network stack and therefore cannot access other Pods, Services, or external networks.

So, it's more accurate to say that:

> **Pods need to use network devices and kernel forwarding mechanisms to access the node network.**

---

## II. How Do Pods Access the Node Network

### 1. Network Namespace

Linux provides network namespaces to isolate network environments.  
In Kubernetes, Pods have an independent network namespace by default.

This means that within a Pod:

- The Pod sees its own network card
- IP address
- Route table
- `/etc/resolv.conf`

It does not directly use the host machine's network.

---

### 2. veth Pair

To establish a connection between a Pod and a node, a pair of **veth pairs** is usually created.

You can think of them as two ends of a “virtual network cable”:

- One end enters the Pod and becomes the `eth0`
- The other end remains on the node side

Data packets sent by the Pod exit through the veth on the Pod side, enter the veth on the node side, and then are processed by the node's network.

---

### 3. Node-Side Network Devices

The veth on the node side is usually not used alone but is connected to some kind of node network infrastructure, such as:

- A bridge (e.g., `cni0`)
- Routing devices
- VXLAN devices
- Virtual interfaces and routing systems created by Calico

This part of the process is handled by specific **CNI plugins**.

---

## III. What Is CNI

CNI stands for **Container Network Interface**.

It is a set of standards for container network interfaces that handle tasks such as:

- Assigning IPs to Pods
- Creating network devices
- Connecting Pods to the node network
- Configuring networking capabilities like bridges, routes, and tunnels

Kubernetes itself does not directly implement Pod networking but relies on CNI plugins to provide these functionalities.

Common CNI plugins include:

- Flannel
- Calico
- Cilium

---

## IV. The Relationship Between Pods and the Host Machine's Network

### Default Situation

By default:

- Pods have an independent- Accessing the node network via CNI and veth pairs
- Not directly sharing the host machine's network stack

Only `hostNetwork` is shared with the host machine.

---

### 2. Why can Pods access the external internet?

Because Pod traffic first enters the node network stack and then is forwarded out through routing and NAT.

---

### 3. Why are Flannel, Calico, and BGP mentioned in interviews?

Because they relate not to "how Docker configures networks" but rather to:

> **How Kubernetes implements cross-node Pod networking.**

---

### 4. What is the fundamental difference between Flannel and Calico?

The direct answer is:

**Flannel primarily uses an overlay approach, encapsulating packets using VXLAN before sending them; Calico mainly relies on routing, utilizing BGP to distribute Pod network segment routes.**

---

## Twelve, Why is this question prone to misinterpretation?

In interviews, it's easy to confuse these two aspects:

### First aspect: How do Pods connect to the node network?
Keywords:

- netns
- veth pair
- bridge
- CNI

### Second aspect: How do Pods communicate across nodes?
Keywords:

- Flannel
- Calico
- VXLAN
- BGP

If these two aspects are not distinguished, it's easy to answer from the perspective of "usage layers" such as Docker networks or host networks, which does not address the interviewer's real question about the "implementation layer."

---

## Thirteen, In one sentence

In Kubernetes, Pods have independent network namespaces by default and connect to the node network through CNI and veth pairs. Communication between Pods on the same node typically occurs via bridges or routing, while cross-node communication is implemented using network plugins—Flannel often encapsulates packets with VXLAN, and Calico uses BGP for routing.

---

## Fourteen, What does this question assess?

- Basic knowledge of Linux networking
- Understanding of network namespaces
- Knowledge of veth pairs
- The role of CNI
- The Pod network model
- Differences between same-node and cross-node communication
- The implementation mechanisms of Flannel and Calico
- The fundamental concepts of VXLAN and BGP

---

## Related Tags
#InterviewTips #Kubernetes #K8s #Network #CNI #Flannel #Calico #VXLAN #BGP #InterviewQuestions