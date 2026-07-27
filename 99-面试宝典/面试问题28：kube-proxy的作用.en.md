---
title: The Role of kube-proxy
tags:
  - Interview Guide
  - Kubernetes
  - K8s
  - Networking
  - Service
  - kube-proxy
  - iptables
  - IPVS
  - Interview Questions
category: Interview Guide
topic: Kubernetes Networking
type: Interview Question
source: Interview Review
created: 2026-03-18
updated: 2026-03-18
---

# Interview Question 28: The Role of kube-proxy

## Question
What is the role of kube-proxy?

## Core Answer
`kube-proxy` is a network proxy component that runs on each Node, primarily responsible for **forwarding traffic related to Kubernetes Services**.

It monitors changes in Services and Endpoints / EndpointSlice within the `apiserver`, and then maintains corresponding network forwarding rules on the node.  
These rules are usually implemented using:

- `iptables`
- `IPVS`

When a client accesses a Service's `ClusterIP` or `NodePort`, the request is forwarded to an available backend Pod based on these rules.

## Interview Answer Version
> kube-proxy is a network proxy component that runs on each node, mainly responsible for forwarding traffic related to Kubernetes Services.  
> It monitors changes in Services and Endpoints within the apiserver, and then generates iptables or IPVS rules on the node to forward requests to the corresponding backend Pods.  
> Service itself is merely an abstraction; it is kube-proxy that actually directs requests made to ClusterIP or NodePort to the appropriate Pods.

## Fundamental Understanding
You can think of it this way:

- **Service** acts as an abstract entry point.
- **kube-proxy** is responsible for forwarding traffic arriving at this entry point to the backend Pods.

Alternatively, you can say:

- **CoreDNS** handles “finding the right target”.
- **kube-proxy** handles “how to route the traffic there”.

## How kube-proxy Works

### 1. Monitoring Resource Changes
kube-proxy continuously monitors changes in:

- Services
- Endpoints
- EndpointSlice

When these resources change—for example, a new Service is created, a Pod is added, or a backend Pod becomes unhealthy—kube-proxy updates its local forwarding rules accordingly.

### 2. Maintaining Forwarding Rules on the Node
kube-proxy does not forward traffic at every hop itself. Instead, it:

- Generates rules based on the cluster's current state.
- Applies these rules to the Linux kernel's network stack.
- The kernel then handles the actual forwarding.

### 3. Implementing Load Distribution for Services to Pods
Requests made to:

- `ClusterIP`
- `NodePort`

are forwarded to a healthy backend Pod according to the established rules.

## Common Modes of kube-proxy

### 1. userspace
An early mode with poor performance, now rarely used.

### 2. iptables
Uses Linux `iptables` rules to forward Service traffic to Pods.

Advantages:

- Simple implementation.
- Widely adopted.
- Can lead to numerous rules in large-scale clusters.

### 3. IPVS
Uses Linux `IPVS` for layer-4 load balancing.

Advantages:

- Better performance.
- More powerful rule management.
- Suitable for larger clusters.

## Common Follow-up Questions

### 1. What is the difference between kube-proxy and CoreDNS?
#### CoreDNS
Responsible for:
- Domain name resolution.
- Mapping Service names to ClusterIPs.

#### kube-proxy
Responsible for:
- Traffic forwarding.
- Forwarding requests from ClusterIPs/NodePorts to backend Pods.

In simple terms:

> CoreDNS resolves names, while kube-proxy forwards traffic based on those resolutions.

### 2. What is the relationship between kube-proxy and NodePort?
NodePort is another key capability implemented by kube-proxy.

When a NodePort Service is created, kube-proxy sets up rules on each node. Accessing `NodeIP:NodePort` directs traffic to the corresponding Service and then to the backend Pod.

### 3. Is kube-proxy considered a load balancer?
It has **layer-4 forwarding and basic load distribution capabilities**, but it is not equivalent to traditional layer-7 load balancers. It primarily handles IP, Port, and Service-to-Pod forwarding without handling complex layer-7 routing rules.

### 4. What are common issues with kube-proxy?
Common problems include:

- Inaccessibility of a Service's ClusterIP.
- Issues with NodePort connectivity.
- A Pod being directly reachable but the Service not functioning.
- Normal DNS resolution but failed traffic forwarding.

## Troubleshooting Steps
To diagnose issues with kube-proxy, you can check:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <kube-proxy-pod-name>
```

Check Services and endpoints:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```

View iptables rules:

```bash
iptables