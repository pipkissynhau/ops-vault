---
title: Principles and Practical Operations of etcd Backup and Recovery (Kubernetes Runbook)
tags:
  - Kubernetes
  - K8s
  - etcd
  - backup
  - recovery
  - operations
  - Runbook
category: Kubernetes
topic: Cluster Management
type: Learning + Practice
created: 2026-04-20
updated: 2026-04-20
---

# Principles and Practical Operations of etcd Backup and Recovery (Kubernetes)

## Documentation Notes

This document is a **production-level Runbook** for etcd backup and recovery, intended for:

- Pre-upgrade backups
- Disaster recovery
- Data rollback

Objectives:

- Understand the role of etcd
- Master the backup and recovery process
- Be able to perform recovery operations in case of failures

---

# I. The Role of etcd in Kubernetes

👉 etcd is the core data storage system in Kubernetes.

Stored content includes:

- Pods / Deployments / Services
- ConfigMaps / Secrets
- Nodes / Namespaces
- Scheduling information

---

### One-sentence Conclusion

👉 **Loss of etcd = Loss of cluster state**

---

# II. The Nature of Backup

Etcd backup does not involve exporting YAML files; instead, it involves:

👉 **Taking a consistent snapshot of the etcd data**

---

# III. Confirming the etcd Deployment Method (Required)

## kubeadm Cluster

    cat /etc/kubernetes/manifests/etcd.yaml

Verify:

- etcd is configured as a static Pod
- The path for the data directory (default: /var/lib/etcd)

---

## Checking the etcd Pod

    kubectl -n kube-system get pods | grep etcd

---

# IV. Backup Procedures (Step-by-step Instructions)

## 1. Set Environment Variables

    export ETCDCTL_API=3

---

## 2. Execute the Backup

    etcdctl snapshot save /backup/etcd-$(date +%F).db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key

---

## 3. Verify the Backup

    etcdctl snapshot status /backup/etcd-*.db

---

## 4. Recommended Practices

- Perform regular daily backups
- Keep multiple backup copies
- Store backups in a remote location

---

# V. Recovery Principles (Core)

The essence of recovering etcd data is:

👉 **Using the snapshot to rebuild the data directory and restart etcd using this new directory**

---

# VI. Complete Recovery Process

⚠️ These steps must be performed on control plane nodes.

## 1. Stop kubelet

    systemctl stop kubelet

---

## 2. Back up the original data (to prevent accidental overwrites)

    mv /var/lib/etcd /var/lib/etcd.bak

---

## 3. Restore the snapshot

    etcdctl snapshot restore /backup/etcd-xxx.db \
      --data-dir=/var/lib/etcd-restore

---

## 4. Modify the etcd Configuration File

    vi /etc/kubernetes/manifests/etcd.yaml

Find and change:

    --data-dir=/var/lib/etcd

to:

    --data-dir=/var/lib/etcd-restore

---

## 5. Restart kubelet

    systemctl start kubelet

---

## 6. Verify etcd Status

    kubectl get nodes
    kubectl get pods -A

---

# VII. Common Post-Recovery Phenomena

After recovery:

- The cluster will revert to the state at the time of the backup
- Any newly created resources will be lost

👉 Essentially, it's like “time rewinding”.

---

# VIII. Common Fault Resolution (Key Points)

## ❌ 1. etcd Fails to Start

    crictl ps -a | grep etcd
    crictl logs <container-id>

Check:

- Whether the data directory path is correct
- Permission issues

---

## ❌ 2. API Server Fails to Start

    crictl logs <apiserver-id>

Possible causes:

- etcd is unavailable
- Certificate issues

---

## ❌ 3. kubectl Cannot Connect

    kubectl get nodes

Possible errors:

- X509 certificate error
- Connection refused

Resolution:

- Check the apiserver
- Verify the kubeconfig file

---

## ❌ 4. Node NotReady

    kubectl get nodes

Possible causes:

- kubelet has not been restored
- Certificate issues

---

## ❌ 5. All Pods Fail to Start