---
title: "# etcd Backup and Recovery Principles and Practical Operations (Kubernetes Runbook)"
tags:
  - Kubernetes
  - K8s
  - etcd
  - Backup
  - Restore
  - Transport
  - Runbook
category: Kubernetes
topic: "# Cluster Management"
type: "# Learning + Hands-on Practice"
created: 2026-04-20
updated: 2026-04-20
---

# etcd Backup and Recovery Principles and Practical Guide (Kubernetes)

## Document Notes

This document is a **production-grade Runbook** for etcd backup and recovery, used for:

- Backup before upgrades
- Fault recovery
- Data rollback

Goals:

- Understand etcd's role
- Master backup and recovery procedures
- Execute recovery directly during failures

---

# I. etcd's Role in Kubernetes

👉 etcd is the core data store of Kubernetes

Stored content:

- Pod / Deployment / Service
- ConfigMap / Secret
- Node / Namespace
- Scheduling information

---

### One-line Conclusion

👉 **etcd loss = cluster state loss**

---

# II. The Essence of Backup

etcd backup is not exporting YAML, but:

👉 **Taking a consistent snapshot of etcd data**

---

# III. Confirm etcd Deployment Method (Mandatory)

## kubeadm Cluster

    cat /etc/kubernetes/manifests/etcd.yaml

Confirm:

- etcd is a static Pod
- data-dir path (default /var/lib/etcd)

---

## View etcd Pod

    kubectl -n kube-system get pods | grep etcd

---

# IV. Backup Operations (Executable Steps)

## 1. Set Environment Variables

    export ETCDCTL_API=3

---

## 2. Execute Backup

    etcdctl snapshot save /backup/etcd-$(date +%F).db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key

---

## 3. Verify Backup

    etcdctl snapshot status /backup/etcd-*.db

---

## 4. Recommended Strategy

- Daily scheduled backup
- Keep multiple copies
- Offsite storage

---

# V. Recovery Principles (Core)

Recovering etcd essentially means:

👉 **Rebuilding the data directory using the snapshot and having etcd use this directory to start**

---

# VI. Recovery Operations (Complete Process)

⚠️ Must be executed on the control plane node

---

## 1. Stop kubelet

    systemctl stop kubelet

---

## 2. Backup Original Data (Prevent accidental operations)

    mv /var/lib/etcd /var/lib/etcd.bak

---

## 3. Restore Snapshot

    etcdctl snapshot restore /backup/etcd-xxx.db \
      --data-dir=/var/lib/etcd-restore

---

## 4. Modify etcd Configuration

    vi /etc/kubernetes/manifests/etcd.yaml

Find:

    --data-dir=/var/lib/etcd

Change to:

    --data-dir=/var/lib/etcd-restore

---

## 5. Start kubelet

    systemctl start kubelet

---

## 6. Verify etcd

    kubectl get nodes
    kubectl get pods -A

---

# VII. Common Post-Recovery Phenomena

After recovery:

- Cluster state reverts to the backup time point
- Newly created resources disappear

👉 Essentially it's "time travel back"

---

# VIII. Common Fault Handling (Core)

## ❌ 1. etcd Won't Start

    crictl ps -a | grep etcd
    crictl logs <container-id>

Check:

- Whether data-dir is correct
- Permission issues

---

## ❌ 2. API Server Won't Start

    crictl logs <apiserver-id>

Causes:

- etcd unavailable
- Certificate issues

---

## ❌ 3. kubectl Can't Connect

    kubectl get nodes

Error:

- x509 error
- connection refused

Resolution:

- Check apiserver
- Check kubeconfig

---

## ❌ 4. Node NotReady

    kubectl get nodes

Causes:

- kubelet not recovered
- Certificate issues

---

## ❌ 5. All Pods Are Down

    kubectl get pods -A

Causes:

- Data rollback
- Controller rebuild

---

# IX. Production Notes

- Must back up etcd before upgrades
- Do not overwrite original data directory
- Recovery must stop kubelet first
- Multi-control plane requires synchronized processing
- Must pre-exercise recovery procedures

---

# X. Prohibited Operations (Important)

❌ No backup and then upgrade  
❌ Directly overwrite /var/lib/etcd  
❌ Recover without stopping kubelet  
❌ First-time recovery in production environment

---

# XI. Rollback Capability Explanation

👉 etcd snapshot is the only reliable rollback method

Applicable to:

- Upgrade failure
- Configuration misoperation
- Cluster anomalies

---

# XII. One-line Summary

👉 etcd backup is a snapshot, recovery is using the snapshot to rebuild the data directory and return the cluster to a historical state.

---

# XIII. Reference Links

- Kubernetes etcd Management  
  https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/

- etcd Recovery Documentation  
  https://etcd.io/docs/v3.5/op-guide/recovery/

- etcdctl Usage  
  https://etcd.io/docs/v3.5/dev-guide/interacting_v3/