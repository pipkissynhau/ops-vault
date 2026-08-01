---
title: "# The Role of kube-proxy

kube-proxy is a component in Kubernetes used for service discovery and traffic forwarding. It ensures that network traffic is correctly routed to the appropriate pods based on the service configuration.

- **Traffic Forwarding**: kube-proxy forwards traffic to the correct pods based on the service rules.
- **Load Balancing**: It provides load balancing capabilities to distribute traffic across multiple pods.
- **Service Discovery**: kube-proxy enables pods to discover and communicate with services within the cluster.

> [!note]
> kube-proxy operates at the network layer and can use different modes such as iptables or IPVS to manage traffic."
tags:
  - The Book of Interviews
  - Kubernetes
  - K8s
  - Network
  - Service
  - kube-proxy
  - iptables
  - IPVS
  - Interview
category: "# Interview Compendium"
topic: "# Kubernetes Networking"
type: "Interview Questions /think"
source: "# Interview Review"
created: 2026-03-18
updated: 2026-03-18
---

# Interview Question 28: What is the role of kube-proxy?

## Question
What is the role of kube-proxy?

## Core Answer
`kube-proxy` is a network proxy component running on each Node, primarily responsible for implementing **Kubernetes Service traffic forwarding**.

It listens to changes in Service and Endpoint / EndpointSlice in `apiserver`, then maintains corresponding network forwarding rules on the node.  
These rules are typically implemented through:

- `iptables`
- `IPVS`

When clients access a Service's `ClusterIP` or `NodePort`, requests are forwarded to a healthy backend Pod according to these rules.

## Interview Answer Version
> kube-proxy is a network proxy component running on each node, primarily responsible for implementing Kubernetes Service traffic forwarding.  
> It listens to changes in Service and Endpoint in apiserver, then generates iptables or IPVS rules on the node to forward traffic to backend Pods.  
> Service itself is an abstraction, and the actual forwarding of ClusterIP or NodePort requests to specific Pods is mainly handled by kube-proxy.

## Essential Understanding
You can think of it this way:

- **Service** is an abstract entry point
- **kube-proxy** is responsible for forwarding traffic accessing this entry point to backend Pods

You can also remember it as:

- **CoreDNS** solves "finding who"
- **kube-proxy** solves "how to forward"

## kube-proxy's Working Mechanism

### 1. Monitoring Resource Changes
kube-proxy continuously monitors:

- Service
- Endpoints
- EndpointSlice

When these resources change, for example:

- A new Service is created
- A new Pod is added
- A Pod is deleted
- A backend Pod becomes unhealthy

kube-proxy synchronizes and updates the local traffic forwarding rules.

### 2. Maintaining Forwarding Rules on the Node
kube-proxy itself does not forward traffic hop-by-hop, but rather:

- Generates rules based on cluster state
- Deploys rules to the Linux kernel network stack
- The kernel ultimately executes the forwarding

### 3. Implementing Load Distribution from Service to Pod
When accessing:

- ClusterIP
- NodePort

Requests are forwarded to a healthy backend Pod according to the rules.

## Common kube-proxy Modes

### 1. userspace
An early mode with poor performance; it's rarely used now.

### 2. iptables
Implements Service-to-Pod traffic forwarding through Linux `iptables` rules.

Features:

- Simple implementation
- Widespread use
- Rules can proliferate with large scale

### 3. IPVS
Implements four-layer load balancing through Linux `IPVS`.

Features:

- Better performance
- Stronger forwarding table capabilities
- More suitable for large-scale clusters

## Common Follow-up Questions

### 1. What's the difference between kube-proxy and CoreDNS?
#### CoreDNS
Responsible for:
- Domain name resolution
- Mapping Service names to ClusterIP

#### kube-proxy
Responsible for:
- Traffic forwarding
- Forwarding ClusterIP / NodePort to backend Pods

One-sentence distinction:
> CoreDNS handles "resolving names", kube-proxy handles "forwarding traffic".

### 2. What's the relationship between kube-proxy and NodePort?
NodePort is also one of the important capabilities implemented by kube-proxy.

When creating a NodePort Service:
- kube-proxy establishes rules on each node
- Accessing `NodeIP:NodePort`
- Traffic is forwarded to the corresponding Service, then to backend Pods

### 3. Is kube-proxy a load balancer?
It can be said to have **four-layer forwarding and basic load distribution capabilities**, but it's not equivalent to traditional seven-layer load balancers.

It primarily handles:
- IP
- Port
- Forwarding from Service to Pod

It does not process complex seven-layer semantics, such as HTTP routing rules.

### 4. What are the signs of kube-proxy issues?
Common symptoms include:
- ClusterIP of Service is inaccessible
- NodePort is unreachable
- Pod direct access works, but Service is unreachable
- DNS resolution works, but traffic forwarding fails

## Troubleshooting Approach

Check kube-proxy:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <kube-proxy-pod-name>
```

Check Service and endpoints:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```

Check iptables rules:

```bash
iptables -t nat -L -n
```

If using IPVS mode:

```bash
ipvsadm -Ln
```

## One-sentence Summary
kube-proxy's role is to generate iptables or IPVS forwarding rules on each node based on changes in Service and Endpoint, forwarding traffic to backend Pods.

## What This Question Tests
- Service implementation principles
- ClusterIP / NodePort forwarding
- Relationship between Service and Pod
- iptables / IPVS
- Kubernetes network troubleshooting approach

## Associated Tags
#TheBookOfInterviews #Kubernetes #K8s #Network #Service #kube-proxy #iptables #IPVS #Interview