---
tags: "[Kubernetes, System Components, ControlPlane, Node, Interview]"
---

# Interview Question 12: What are the system components of a Kubernetes cluster and where are they located?

## Explanation
A Kubernetes cluster consists of **Control Plane (Master) components** and **Node (Worker) components**, each responsible for different functions:

| Component | Function | Location |
|----------|---------|----------|
| **API Server** | Receives and processes kubectl/REST requests, exposes Kubernetes API | Control Plane |
| **etcd** | Stores cluster state and resource objects | Control Plane |
| **Controller Manager** | Manages replica, node, service, and other control loops | Control Plane |
| **Scheduler** | Responsible for scheduling Pods to appropriate nodes | Control Plane |
| **kubelet** | Node agent, creates and manages Pods | Node |
| **kube-proxy** | Provides Service network proxy and load balancing | Node |
| **Container Runtime** (Docker, containerd, CRI-O) | Runs containers | Node |
| **CoreDNS / kube-dns** | Provides internal DNS service for the cluster | Control Plane / Pod (schedulable) |

## Operations / View Commands

```bash
# View Control Plane Component Operation Status
kubectl get pods -n kube-system

# View Node Let's go. kubelet and kube-proxy Status
systemctl status kubelet
kubectl get pods -n kube-system -o wide
```

## Key Points Summary

1. **Control Plane**: API Server, etcd, Scheduler, Controller Manager, DNS, etc.  
2. **Node / Worker**: kubelet, kube-proxy, Container Runtime  
3. Understanding component responsibilities and locations helps troubleshoot cluster issues and optimize architecture  

## Interview Answer Example

> "A Kubernetes cluster's system components are divided into Control Plane and Node components.  
> The Control Plane includes API Server (provides REST API), etcd (stores cluster state), Controller Manager (manages replicas, nodes, services, etc. control loops), Scheduler (Pod scheduling), and CoreDNS (internal cluster DNS).  
> Nodes run kubelet (node agent, manages Pods), kube-proxy (Service network proxy), and Container Runtime (container runtime).  
> Understanding these components and their locations helps quickly diagnose cluster issues, such as Pod scheduling failures typically relate to Scheduler or kubelet, and Service access anomalies may involve kube-proxy or CoreDNS."