---
tags:
  - etcd
  - Kubernetes
  - Distribution
  - Cache
  - Interviews
  - High Available
---

# Interview Question 10: etcd Distributed and API Server Write Cache Layer

## Explanation
etcd is Kubernetes' core strong-consistency distributed storage, used to store the entire cluster's state information, including:  
- **Pod, Deployment, Service, etc. object definitions**  
- **ConfigMap, Secret, Namespace, etc. metadata**  
- **Cluster node information, scheduling status, role assignments**  

When API Server writes to etcd, it passes through **local cache (watch cache)** to accelerate reads and reduce etcd pressure.

## Operation Method / Process

1. **API Server Write Process**:  
   - User submits resource YAML → `kubectl apply`  
   - API Server receives request → writes to etcd  
   - Local watch cache is updated for fast querying and event notifications  

2. **etcd Distributed Deployment**:  
   - **Multi-node cluster**: Typically 3~5 nodes forming a Raft cluster  
   - **Raft protocol**: Ensures strong consistency, Leader node receives write requests, Follower nodes synchronize data  
   - **High availability**: Cluster remains readable/writable even if any node fails, as long as more than half the nodes are alive  
   - **Deployment method**: Can run etcd nodes via bare metal, virtual machine, or container (static Pod)  

3. **Optimization Recommendations**:  
   - Avoid frequent writes to small objects; batch operations are preferred  
   - Adjust API Server cache size to optimize read performance  
   - Monitor etcd response time, disk I/O, and node health  

## Key Points Summary

- etcd stores all cluster state data for Kubernetes (object definitions, metadata, node information, etc.)  
- Multi-node etcd clusters use Raft protocol to ensure strong consistency and high availability  
- API Server has internal caching (watch cache) to accelerate reads and reduce etcd pressure  
- Understanding distributed deployment and caching mechanisms helps troubleshoot performance and availability issues  

## Interview Answer Example

> "etcd is Kubernetes' core storage, saving all cluster state information, including Pod, Deployment, Service object definitions, ConfigMap, Secret metadata, and cluster node and scheduling status.  
> We typically deploy etcd as a 3~5 node Raft cluster. Each write is received by the Leader node, with Follower nodes synchronizing data to ensure strong consistency and high availability.  
> API Server also has a watch cache internally for faster reads and event notifications, reducing etcd pressure. Understanding these concepts helps quickly identify root causes during cluster failures or performance issues."