---
tags: "[Kubernetes, Pod, Scheduling, Scheduler, Affinity, Interview]"
---

# Interview Question 15: Pod Scheduling Strategy

## Overview
Pod scheduling strategies determine which Node a Pod is scheduled to.  
Scheduling strategies can be based on **resource availability, node labels, affinity, taints/tolerations, and topology**, ensuring Pod high availability and meeting business requirements.

## Core Scheduling Strategies

1. **Resource Scheduling**  
   - Scheduler schedules Pods based on Node CPU, memory, and other resource conditions  
   - Ensures Node resources are sufficient  

2. **Node Selector**  
   - Simple label matching  
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

3. **Node Affinity**  
   - Advanced label matching, divided into **requiredDuringSchedulingIgnoredDuringExecution** (must be met) and **preferredDuringSchedulingIgnoredDuringExecution** (preferably met)  
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

4. **Pod Affinity / Anti-Affinity**  
   - Controls Pod deployment on same or different nodes/regions through label selection and topology constraints  

5. **Taints & Tolerations**  
   - Nodes add taints to control whether Pods can be scheduled to the node  
```yaml
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```

6. **Topology Spread Constraints**  
   - Controls Pod distribution across different zones, racks, and nodes to avoid single points of failure  

## Operation / View Commands

```bash
# View Pod Schedule events
kubectl describe pod <pod-name>

# View Node Label
kubectl get nodes --show-labels

# View Pod Compassion and stigma/Tolerance
kubectl get pod <pod-name> -o yaml
kubectl describe node <node-name>
```

## Key Takeaways

- Scheduling strategies ensure reasonable Pod allocation, sufficient resources, and high availability  
- Core mechanisms: resource scheduling, node selector, node affinity, pod affinity/anti-affinity, taint tolerations, and topology spread constraints  
- Commands can be combined to view scheduling events and node information, aiding in troubleshooting scheduling issues  

## Interview Answer Example

> "Kubernetes Pod scheduling strategy is the mechanism by which the Scheduler decides which node to deploy a Pod.  
> It considers node available resources, label matching via NodeSelector or Node Affinity, Pod affinity/anti-affinity rules, taints/tolerations, and topology spread constraints.  
> For example, if you want to deploy a Pod on SSD nodes, you can use NodeSelector or Node Affinity; if you want to ensure Pods are not on the same node or zone, you can use Pod Anti-Affinity or Topology Spread Constraints. Scheduling events can be viewed via `kubectl describe pod` to troubleshoot scheduling failures or resource imbalance issues."