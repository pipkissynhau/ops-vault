---
title: Kubernetes Network Plugins and Underlying Principles
tags:
  - Interview Guide
  - Kubernetes
  - K8s
  - Networking
  - CNI
  - Calico
  - Flannel
  - Cilium
  - eBPF
  - NetworkPolicy
  - Interview Questions
category: Interview Guide
topic: Kubernetes Networking
type: Interview Question
source: Interview Review
created: 2026-03-18
updated: 2026-03-18
---

# Interview Question 29: Kubernetes Network Plugins and Underlying Principles

## Question
What are the Kubernetes network plugins, and what are their underlying principles?

## Key Answer
Kubernetes itself only defines the network model but does not directly implement Pod network communication.  
The **CNI (Container Network Interface) plugins** are responsible for configuring the network for Pods.

Common network plugins include:

- Flannel
- Calico
- Cilium
- Weave Net
- Canal
- Cloud vendor-provided CNIs

## Interview Answer Example
> Kubernetes itself does not handle the specific implementation of Pod networks; this task is performed by CNI plugins such as Flannel, Calico, and Cilium.  
> These plugins primarily accomplish the following: assigning IP addresses to Pods, creating network interfaces, connecting Pods to the host network, enabling cross-node communication, and supporting NetworkPolicy and service forwarding optimization in certain scenarios.  
> For example, Flannel uses VXLAN for overlay networking; Calico focuses on layer 3 routing and policy control; Cilium leverages eBPF for enhanced performance and visibility, even allowing it to replace kube-proxy.

## The Essence of Kubernetes Network Plugins

CNI plugins essentially address the following issues:

### 1. Assigning IP Addresses to Pods
This is often referred to as **IPAM (IP Address Management)**.  
Each Pod requires a unique IP address when it starts up.

### 2. Connecting Pods to the Host Network
Pods operate within their own independent **network namespace**.  
CNI plugins create network devices to connect Pods to the host network.

Common methods include:

- veth pair
- bridge
- route

### 3. Enabling Cross-Node Communication
When two Pods are on different nodes, they need a way to communicate with each other.  
This is achieved through various techniques:

#### Overlay Networks
Examples include:

- VXLAN
- IP-in-IP

Advantages:
- Lower requirements for the underlying network
- Easy to deploy
- Additional encapsulation overhead

#### Underlay / Layer 3 Routing
Examples include:

- BGP
- Direct routing

Advantages:
- More direct paths
- Better performance
- Higher demands on the underlying network

### 4. Supporting Network Policies and Service Traffic Optimization
Advanced plugins also provide features such as:

- NetworkPolicy for controlling network traffic
- eBPF for enhanced data plane functionality
- More efficient Service forwarding
- Improved visibility and debugability

## Key Technical Concepts

### 1. Network Namespace
Each Pod has its own independent network namespace.

### 2. veth Pair
A **veth pair** consists of two virtual network interfaces:

- One within the Pod's namespace
- The other on the host side

They are used to connect the Pod's network to the host network.

### 3. bridge / route / overlay
Different CNI plugins use various methods to achieve these functions:

- bridge
- route
- VXLAN
- IPIP
- BGP

### 4. iptables / eBPF
Network policy control and service forwarding optimization often rely on:

- iptables
- conntrack
- eBPF

## Characteristics of Common Plugins

## 1. Flannel

### Characteristics
- Lightweight
- Simple
- Often used in beginner scenarios

### Principle
Flannel typically assigns a Pod network segment to each node and uses VXLAN or similar overlay techniques to connect Pods across nodes.

### Suitable for
- Quickly setting up basic networks
- Scenarios where NetworkPolicy is not a critical requirement

## 2. Calico

### Characteristics
- Widely used in enterprise environments
- Supports NetworkPolicy
- Offers multiple routing modes (BGP, IPIP, VXLAN)

### Principle
Calico can use layer 3 routing to forward Pod networks directly or employ IPIP/VXLAN for encapsulation.

It not only provides connectivity but also emphasizes:

- Routing capabilities
- Policy control
- Stability in production environments

### Suitable for
- Production scenarios with strict network policy and stability requirements

## 3. Cilium

### Characteristics
- Based on eBPF
- High performance
- Strong visibility
- Can replace kube-proxy

### Principle
Cilium offloads some network forwarding, policy control, and load balancing logic to the Linux kernel's eBPF framework,