---
title: "Kubernetes Cluster Upgrade Principles and Practical Operations (kubeadm)"
tags:
  - Kubernetes
  - K8s
  - Cluster upgrade
  - kubeadm
  - Transport
  - Learning notes
  - Runbook
category: Kubernetes
topic: "# Cluster Management /think"
type: "# Learning + Hands-on Practice"
created: 2026-04-20
updated: 2026-04-20
---

# Kubernetes Cluster Upgrade Principles and Practical Guide (kubeadm)

## Document Notes

This document is a "Learning + Production Practice Integration Version", aiming to:

- Understand upgrade principles
- Master executable steps
- Be able to troubleshoot issues
- Be applicable in production environments

---

# I. Why Upgrades Are Needed

Reasons for upgrading Kubernetes clusters:

- Fix security vulnerabilities
- Obtain new features
- Improve stability
- Avoid version expiration (official support cycle is relatively short)

---

# II. Essence of Upgrade

Kubernetes upgrade essentially means:

👉 Gradually upgrade control plane and node components while ensuring business continuity

Components involved:

Control Plane:

- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd

Nodes:

- kubelet
- kube-proxy

---

# III. Core Upgrade Principles

## 1. No Cross-Version Upgrades

    1.28 → 1.29 → 1.30

---

## 2. Prioritize Control Plane

    Control Plane → Worker Node

---

## 3. Rolling Upgrade

👉 Operate one node at a time

---

## 4. Business Must Have High Availability

- Deployment ≥ 2 replicas
- Service provides load balancing

---

# IV. Pre-Upgrade Checks (Mandatory)

## 1. Cluster Status

    kubectl get nodes
    kubectl get pods -A

Requirements:

- All nodes Ready
- No abnormal Pods

---

## 2. Pending Pods

    kubectl get pods -A | grep Pending

👉 Must be 0

---

## 3. Replica Checks

    kubectl get deploy -A

👉 Core business ≥ 2 replicas

---

## 4. Check Upgrade Path

    kubeadm upgrade plan

---

## 5. etcd Backup (Mandatory)

    ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F).db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key

---

# V. Control Plane Upgrade (First Master)

## 1. Upgrade kubeadm

    apt-mark unhold kubeadm
    apt-get update
    apt-get install -y kubeadm=1.xx.x-00
    apt-mark hold kubeadm

---

## 2. Execute Upgrade

    kubeadm upgrade apply v1.xx.x

---

## 3. Upgrade kubelet / kubectl

    apt-mark unhold kubelet kubectl
    apt-get install -y kubelet=1.xx.x-00 kubectl=1.xx.x-00
    apt-mark hold kubelet kubectl
    systemctl restart kubelet

---

## 4. Verify Control Plane

    kubectl get pods -n kube-system

Focus on checking:

- kube-apiserver
- kube-controller-manager
- kube-scheduler

---

# VI. Multi-Control Plane Node Upgrade (HA)

Execute sequentially:

    kubeadm upgrade node
    systemctl restart kubelet

---

# VII. Worker Node Upgrade (One by One)

## 1. Drain Node

    kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

---

## 2. Upgrade kubeadm

    apt-get install -y kubeadm=1.xx.x-00
    kubeadm upgrade node

---

## 3. Upgrade kubelet

    apt-get install -y kubelet=1.xx.x-00 kubectl=1.xx.x-00
    systemctl restart kubelet

---

## 4. Resume Scheduling

    kubectl uncordon <node-name>

---

# VIII. Post-Upgrade Verification (Mandatory)

## 1. Node Status

    kubectl get nodes

---

## 2. Pod Status

    kubectl get pods -A

---

## 3. Business Verification

- Service interface access is normal
- No errors
- Monitoring is normal

---

# IX. Common Fault Handling (Critical)

## ❌ 1. kubeadm upgrade Stuck

    journalctl -u kubelet -f

Check:

- kubelet logs
- Image pull

---

## ❌ 2. apiserver Won't Start

    crictl ps -a | grep apiserver
    crictl logs <container-id>

---

## ❌ 3. Drain Stuck

    kubectl get pdb -A
    kubectl describe pod <pod>

Reasons:

- PDB limits
- Scheduling failure

---

## ❌ 4. Pod Pending

    kubectl describe pod <pod>

Focus on:

- Resource insufficiency
- Node restrictions

---

## ❌ 5. kubelet Startup Failure

    journalctl -u kubelet -f

---

# X. Production Considerations

- Operate one node at a time
- Must back up etcd first
- Avoid business peak hours
- Do not skip versions
- Do not upgrade multiple nodes in parallel

---

# XI. Rollback Strategy (Critical)

If upgrade fails:

👉 Only reliable method:

- Stop the cluster
- Restore using etcd snapshot

---

# XII. Summary in One Sentence

👉 Cluster upgrade = Control plane upgrade first + Node-by-node upgrade + Drain to ensure business continuity + etcd as the fallback

---

# XIII. Reference Links

- kubeadm Upgrade Official Documentation  
  https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

- Upgrade Cluster  
  https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/

- Version Compatibility Strategy  
  https://kubernetes.io/releases/version-skew-policy/