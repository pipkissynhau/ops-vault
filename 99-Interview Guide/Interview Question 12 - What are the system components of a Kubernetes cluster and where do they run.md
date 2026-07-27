---
tags: [Kubernetes, System Components, ControlPlane, Node, Interview]
---

# Interview Question 12: What are the system components of a Kubernetes cluster and where do they run?

## Explanation
A Kubernetes cluster consists of **Control Plane (Master) components** and **Node (Worker) components**, each performing different functions:

| Component | Function | Location |
|------------|-----------|-------------|
| **API Server** | Receives and processes kubectl/REST requests; exposes the Kubernetes API | Control Plane |
| **etcd** | Stores cluster state and resource objects | Control Plane |
| **Controller Manager** | Manages replication, nodes, services, and other control loops | Control Plane |
| **Scheduler** | Assigns Pods to appropriate nodes | Control Plane |
| **kubelet** | Node agent that creates and manages Pods | Node |
| **kube-proxy** | Provides Service network proxy and load balancing | Node |
| **Container Runtime** (Docker, containerd, CRI-O) | Runs containers | Node |
| **CoreDNS / kube-dns** | Provides internal DNS services for the cluster | Control Plane / Pod (scalable) |

## Commands for Operation/Viewing

```bash
# Check the status of Control Plane components
kubectl get pods -n kube-system

# View the status of kubelet and kube-proxy on a Node
systemctl status kubelet
kubectl get pods -n kube-system -o wide
```

## Key Points Summary

1. **Control Plane**: API Server, etcd, Scheduler, Controller Manager, DNS, etc.
2. **Node / Worker**: kubelet, kube-proxy, Container Runtime.
3. Understanding the responsibilities and locations of these components helps in troubleshooting cluster issues and optimizing architecture.

## Example Answer for Interview

> “The system components of a Kubernetes cluster are divided into Control Plane and Node components. The Control Plane includes the API Server (which provides REST APIs), etcd (for storing cluster state), Controller Manager (for managing replication, nodes, etc.), Scheduler (for Pod scheduling), and CoreDNS (for internal DNS). On the Nodes, kubelet manages Pods, kube-proxy handles Service networking, and the Container Runtime runs containers. Knowing where these components are located and what they do allows us to quickly identify and resolve potential problems in the cluster, such as scheduling failures or service access issues.”