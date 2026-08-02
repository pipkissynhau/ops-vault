---
title: "# Kubernetes Network Plugins and Underlying Principles"
tags:
  - The Book of Interviews
  - Kubernetes
  - K8s
  - Network
  - CNI
  - Calico
  - Flannel
  - Cilium
  - eBPF
  - NetworkPolicy
  - Interview
category: "# Interview Bible /think"
topic: "# Kubernetes Network

## Network Policy Overview
- [ ] Unconfigured network policy
- [x] Default Deny All policy
- [ ] Custom network policy

| Policy Type | Description | Default Behavior |
|-------------|-------------|------------------|
| Default Deny All | Blocks all traffic | Enabled by default |
| Default Allow All | Allows all traffic | Disabled by default |

> [!note] Network policies are applied at the pod level
> [!warning] Misconfigured policies can cause service disruptions

--- 

### Network Policy Configuration
- [ ] Create policy using `kubectl apply -f policy.yaml`
- [ ] Define ingress/egress rules
- [ ] Specify source/destination pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-policy
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

> [!tip] Use `kubectl get networkpolicy` to verify configuration

--- 

### Troubleshooting
- [ ] Check policy conflicts with `kubectl describe networkpolicy`
- [ ] Validate pod IP ranges
- [ ] Review kube-proxy logs

> [!important] Always test policies in a staging environment before production deployment"
type: "Interview Questions /think"
source: "Interview Debrief"
created: 2026-03-18
updated: 2026-03-18
---

# Interview Question 29: Kubernetes Network Plugins and Underlying Principles

## Question
What are Kubernetes network plugins? What is the underlying principle?

## Core Answer
Kubernetes itself only defines the network model and does not directly implement Pod network communication.  
The **CNI (Container Network Interface) plugin** is truly responsible for configuring the network for Pods.

Common network plugins include:

- Flannel
- Calico
- Cilium
- Weave Net
- Canal
- Cloud provider's built-in CNI

## Interview-Friendly Version
> Kubernetes itself does not handle the specific implementation of Pod networking; the CNI network plugin is responsible for this. Common plugins include Flannel, Calico, Cilium, etc.  
> They primarily do the following at the underlying level: assign IP addresses to Pods, create network interfaces, connect Pods to the host network, enable cross-node communication, and in some scenarios support NetworkPolicy and service forwarding optimization.  
> Flannel is more focused on simplicity, typically using overlay technologies like VXLAN for cross-node communication; Calico emphasizes Layer 3 routing and network policy control; Cilium leverages eBPF for stronger performance and observability, and can even replace kube-proxy.

## Essence of Kubernetes Network Plugins

CNI plugins fundamentally solve the following issues:

### 1. Assign IP to Pod
This is typically referred to as:

- IPAM (IP Address Management)

Each Pod needs an independent IP upon startup.

### 2. Connect Pod to Host Network
Pods run in independent network namespaces.  
CNI plugins need to create network devices to connect Pods to the host network.

Common approaches:

- veth pair
- bridge
- route

### 3. Enable Cross-Node Pod Communication
If two Pods are on different nodes, the cross-node network connectivity issue must be resolved.

Common implementation approaches are divided into two categories:

#### Overlay Network
Examples:

- VXLAN
- IP-in-IP

Features:
- Low requirement for underlying network
- Easy deployment
- Additional encapsulation overhead

#### Underlay / Layer 3 Routing
Examples:

- BGP
- Direct routing

Features:
- More direct path
- Better performance
- Higher requirement for underlying network environment

### 4. Support for Network Policy and Service Traffic Optimization
Some advanced plugins also support:

- NetworkPolicy
- eBPF data plane
- Higher-performance Service forwarding
- Enhanced observability

## Core Technical Principles

### 1. Network Namespace
Each Pod has its own independent network namespace.

### 2. veth pair
A veth pair is a pair of virtual network interfaces:

- One end in the Pod
- One end on the host

Used to connect Pod network to the host network.

### 3. bridge / route / overlay
Different CNI plugins implement these in various ways:

- bridge
- route
- VXLAN
- IPIP
- BGP

### 4. iptables / eBPF
Network policy control, service forwarding optimization, etc., often rely on:

- iptables
- conntrack
- eBPF

## Characteristics of Common Plugins

## 1. Flannel

### Features
- Lightweight
- Simple
- Commonly used in introductory environments

### Principle
Flannel typically:

- Assigns a Pod subnet to each node
- Uses overlay technologies like VXLAN to connect Pod networks across nodes

### Suitable Scenarios
- Rapid setup of basic networking
- Scenarios with low NetworkPolicy requirements

## 2. Calico

### Features
- Very common in enterprises
- Supports NetworkPolicy
- Supports BGP, IPIP, VXLAN, etc.

### Principle
Calico can:

- Directly forward Pod network via Layer 3 routing
- Also use IPIP/VXLAN for encapsulation

It not only solves networking issues but also emphasizes:

- Routing capabilities
- Policy control
- Controllability in production environments

### Suitable Scenarios
- Production environments
- Scenarios with high requirements for network policy and stability

## 3. Cilium

### Features
- Based on eBPF
- Strong performance
- Strong observability
- Can replace kube-proxy

### Principle
Cilium moves part of the network forwarding, policy control, and service load balancing logic into the Linux kernel's eBPF to reduce reliance on iptables.

### Suitable Scenarios
- Pursuit of high performance
- Need for stronger network observability
- Scenarios wishing to leverage eBPF capabilities

## Common Follow-up Questions

### 1. What is a veth pair?
A veth pair is a pair of virtual network interfaces, one end in the Pod's network namespace, the other on the host side, used to connect Pod and host networks.

### 2. What is an overlay network?
Overlay is a virtual network built on top of the existing underlying network, with VXLAN being a common technology.

Advantages:
- Low requirement for underlying network
- Easy deployment

Disadvantages:
- Additional encapsulation overhead

### 3. What is the difference between Flannel and Calico?

#### Flannel
- More focused on simple networking
- Common overlay solution

#### Calico
- Supports NetworkPolicy in addition to networking
- Emphasizes Layer 3 routing and production capabilities

### 4. Why is Cilium gaining popularity now?
Because it leverages eBPF to move network forwarding, policy control, and observability capabilities into the kernel, offering stronger performance and functionality.

## One-Sentence Summary
Kubernetes network plugins are essentially CNI plugins, responsible for assigning IPs to Pods, creating network interfaces, enabling cross-node communication, and providing network policy and high-performance data plane capabilities in some scenarios.

## What This Question Tests
- Understanding of CNI fundamentals
- Pod network attachment process
- IPAM
- veth pair
- Overlay vs. Underlay
- Differences between Flannel/Calico/Cilium
- NetworkPolicy
- eBPF

## Associated Tags
#TheBookOfInterviews #Kubernetes #K8s #Network #CNI #Calico #Flannel #Cilium #eBPF #NetworkPolicy #Interview