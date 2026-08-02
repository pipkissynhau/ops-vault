---
title: Kubectl Drain Principle and Node Maintenance (Kubernetes Runbook)
tags:
  - Kubernetes
  - K8s
  - Node Management
  - drain
  - Scheduling
  - Operations
  - Runbook
category: Kubernetes
topic: Cluster Management
type: Learning + Practice
created: 2026-04-20
updated: 2026-04-20
---

# Kubectl Drain Principle and Node Maintenance (Kubernetes)

## Document Description

This document is a **production-grade Runbook + principle integration version** of kubectl drain, used for:

- Node maintenance
- Cluster upgrades
- Node decommissioning
- Fault troubleshooting

Objectives:

- Understand the underlying mechanism of drain
- Master the executable steps
- Be able to troubleshoot issues

---

# I. What is Drain

kubectl drain is used for:

👉 **Securely clearing business Pods from a node**

---

# II. The Essence of Drain

Drain performs two actions:

1. Mark the node as unschedulable (cordon)
2. Evict Pods from the node

---

# III. Drain Principle (Core Understanding)

## 1. Cordon

    kubectl cordon node-1

Function:

- No new Pods will be scheduled on this node
- Existing Pods are not affected

---

## 2. Eviction

Drain does not directly delete Pods; instead, it:

👉 Calls the Eviction API

---

# 3. Scheduler Intervention

After a Pod is evicted:

👉 The scheduler reschedules it to another node

---

# 4. Why Business Is Not Interrupted

Prerequisites:

- Deployment has at least 2 replicas
- Service uses load balancing

Process:

- Pods are evicted
- New Pods start on other nodes
- Service automatically directs traffic to the new pods

---

# IV. Standard Node Maintenance Process (Executable Steps)

## 1. Check Node Status

    kubectl get nodes

---

## 2. List Pods on the Node

    kubectl get pods -o wide | grep node-1

---

## 3. Execute Drain

    kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

---

## 4. Verify Pod Migration

    kubectl get pods -o wide

---

## 5. Perform Maintenance Operations

Examples:

- System upgrades
- kubelet updates
- Network repairs
- Hardware maintenance

---

## 6. Restore Scheduling

    kubectl uncordon node-1

---

## 7. Verify Restoration

    kubectl get nodes
    kubectl get pods -o wide

---

# V. Key Parameter Explanation

## --ignore-daemonsets

👉 Ignores DaemonSet Pods (they will not be evicted)

---

## --delete-emptydir-data

👉 Deletes data from emptyDir volumes

Risk:

- Data loss

---

## --force (Use with Caution)

👉 Forces the deletion of Pods

Risk:

- Business interruption

---

# VI. Which Pods Are Not Drained

## 1. DaemonSets

- Must exist on every node

---

## 2. Static Pods

- Managed by kubelet
- Not controlled by APIs

---

## 3. Controllerless Pods

👉 Will not be automatically restored after deletion

---

# VII. Common Fault Troubleshooting (Core)

## ❌ 1. Drain Stalls

    kubectl drain node-1

No response or stalls occur

---

### Troubleshooting

    kubectl get pdb -A
    kubectl describe pod <pod>

---

### Possible Causes

- PDB restrictions
- Insufficient replicas

---

## ❌ 2. Pods Remain Pending Scheduling

    kubectl describe pod <pod>

---

### Possible Causes

- Insufficient resources
- Node selector constraints

---

## ❌ 3. StatefulSet/Local Storage Issues

Scenarios:

- local PVs
- hostPath volumes

👉 Pods cannot be migrated

---

### Solutions

- Manually migrate data
- Pause maintenance operations

---

## ❌ 4. Single-Replica Services

👉 Service interruptions after draining

---

### Solution

- Increase the number of replicas in advance

---

## ❌ 5. Insufficient Node Resources

👉 Pods cannot be rescheduled

---

### Solution

    kubectl describe node <node>

---

# VIII. Production Best Practices

- Ensure at least 2 replicas for each resource
- Maintain only one node at a time
- Avoid peak business hours
- Check resources in advance
- Be cautious with local PVs

---

# IX. Prohibited Operations

❌ Drain without checking replica counts  
❌ Drain multiple nodes simultaneously  
❌ Forcefully delete critical Pods  
❌ Ignore PDB information  

---

# X. Relationship with Cluster Upgrades

Upgrade process:

1. Drain the node
2. Upgrade