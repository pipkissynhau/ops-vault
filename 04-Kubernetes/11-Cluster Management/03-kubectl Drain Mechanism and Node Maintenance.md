---
title: "# kubectl drain Principles and Node Maintenance (Kubernetes Runbook)

## kubectl drain Principles

kubectl drain is a command used to safely evict all pods from a node before performing maintenance operations. It follows these key principles:

- [ ] Ensures all pods are gracefully terminated
- [ ] Prevents new pods from being scheduled on the node
- [ ] Maintains application availability during the process
- [ ] Provides options for different eviction policies

## Node Maintenance

When performing node maintenance, follow these steps:

1. **Preparation**
   - [ ] Check node status with `kubectl get nodes`
   - [ ] Identify critical workloads
   - [ ] Plan for potential downtime

2. **Drain the Node**
   ```bash
   kubectl drain <node-name> --ignore-daemonsets --delete-local-data
   ```

3. **Maintenance Operations**
   - [ ] Perform hardware maintenance
   - [ ] Update software
   - [ ] Replace components

4. **Reschedule Workloads**
   - [ ] Use `kubectl uncordon <node-name>` to re-enable scheduling
   - [ ] Monitor pod status with `kubectl get pods -o wide`

> [!note] Always verify the node's role in the cluster before draining. Critical nodes (like control plane nodes) require special handling."
tags:
  - Kubernetes
  - K8s
  - Node Management
  - drain
  - Movement control
  - Transport
  - Runbook
category: Kubernetes
topic: "# Cluster Management"
type: "# Learning + Hands-on Practice"
created: 2026-04-20
updated: 2026-04-20
---

# kubectl drain Principles and Node Maintenance (Kubernetes)

## Document Notes

This document is a **production-grade Runbook + Principle Integration Version** of kubectl drain, used for:

- Node maintenance
- Cluster upgrades
- Node decommissioning
- Fault handling

Goals:

- Understand the underlying mechanism of drain
- Master executable steps
- Troubleshoot issues when problems occur

---

# I. What is drain

kubectl drain is used for:

👉 **Securely empty business Pods on a node**

---

# II. Essence of drain

drain does two things:

1. Mark node as unschedulable (cordon)
2. Evict Pods (eviction)

---

# III. drain Principles (Core Understanding)

## 1. cordon

    kubectl cordon node-1

Purpose:

- No longer schedule new Pods
- Does not affect existing Pods

---

## 2. eviction (Eviction)

drain does not directly delete Pods, but instead:

👉 Calls Eviction API

---

## 3. Scheduler Involvement

After Pods are evicted:

👉 Scheduler reschedules Pods to other nodes

---

## 4. Why Business is Not Interrupted

Prerequisites:

- Deployment ≥ 2 replicas
- Service load balancing

Process:

- Pods are evicted
- New Pods start on other nodes
- Service automatically switches traffic

---

# IV. Standard Node Maintenance Procedure (Executable)

## 1. Check node status

    kubectl get nodes

---

## 2. Check Pods on node

    kubectl get pods -o wide | grep node-1

---

## 3. Execute drain

    kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

---

## 4. Verify Pod migration

    kubectl get pods -o wide

---

## 5. Execute maintenance operations

Examples:

- System upgrade
- kubelet upgrade
- Network repair
- Hardware maintenance

---

## 6. Restore scheduling

    kubectl uncordon node-1

---

## 7. Verify restoration

    kubectl get nodes
    kubectl get pods -o wide

---

# V. Key Parameter Explanations

## --ignore-daemonsets

👉 Ignore DaemonSet Pods (will not be evicted)

---

## --delete-emptydir-data

👉 Delete emptyDir data

Risks:

- Data loss

---

## --force (Use with caution)

👉 Force delete Pods

Risks:

- Business interruption

---

# VI. Which Pods Will Not Be Drained

## 1. DaemonSet

- Must exist on each node

---

## 2. Static Pods

- Managed by kubelet
- Not controlled by API

---

## 3. Orphaned Pods

👉 Deleted Pods will not automatically recover

---

# VII. Common Fault Handling (Core)

## ❌ 1. Drain Stuck

    kubectl drain node-1

No response or stuck

---

### Troubleshooting

    kubectl get pdb -A
    kubectl describe pod <pod>

---

### Causes

- PDB limits
- Insufficient replicas

---

## ❌ 2. Pod Cannot Schedule (Pending)

    kubectl describe pod <pod>

---

### Causes

- Resource insufficiency
- Node selector restrictions

---

## ❌ 3. StatefulSet / Local Storage Issues

Scenario:

- local PV
- hostPath

👉 Pods cannot migrate

---

### Resolution

- Manually migrate data
- Or pause maintenance

---

## ❌ 4. Single-Replica Service

👉 Service interruption after drain

---

### Resolution

- Expand replicas in advance

---

## ❌ 5. Node Resource Insufficiency

👉 Pods cannot reschedule

---

### Resolution

    kubectl describe node <node>

---

# VIII. Production Notes

- Must ensure replicas ≥ 2
- Maintain one node at a time
- Avoid business peak hours
- Pre-check resources
- Note local PV

---

# IX. Prohibited Actions

❌ Drain without checking replicas  
❌ Parallel drain multiple nodes  
❌ Force delete critical Pods  
❌ Ignore PDB

---

# X. Relationship with Cluster Upgrade

Upgrade process:

1. Drain node
2. Upgrade node
3. Uncordon node

👉 Drain is the core step of upgrade

---

# XI. One-Sentence Summary

👉 kubectl drain essentially uses the eviction mechanism to smoothly migrate Pods from a node to other nodes, enabling safe maintenance.

---

# XII. Reference Links

- drain official documentation  
  https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/

- Eviction API  
  https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/

- PodDisruptionBudget  
  https://kubernetes.io/docs/concepts/workloads/pods/disruptions/