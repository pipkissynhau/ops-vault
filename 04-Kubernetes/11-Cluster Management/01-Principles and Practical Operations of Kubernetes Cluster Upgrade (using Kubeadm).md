---
title: Principles and Practical Operations of Kubernetes Cluster Upgrade (using Kubeadm)
tags:
  - Kubernetes
  - K8s
  - Cluster Upgrade
  - kubeadm
  - Operations and Maintenance
  - Study Notes
  - Runbook
category: Kubernetes
topic: Cluster Management
type: Learning + Practical Operation
created: 2026-04-20
updated: 2026-04-20
---

# Principles and Practical Operations of Kubernetes Cluster Upgrade (using Kubeadm)

## Document Description

This document is a "combination of learning and practical production operations", aiming to:

- Understand the upgrade principles
- Master the executable steps
- Be able to troubleshoot problems
- Be applicable in a production environment

---

# I. Why Is an Upgrade Needed?

Reasons for upgrading a Kubernetes cluster include:

- Fixing security vulnerabilities
- Obtaining new features
- Improving stability
- Avoiding version expiration (since official support periods are relatively short)

---

# II. The Essence of Upgrade

The essence of a Kubernetes upgrade is:

👉 Upgrading the control plane and node components in phases while ensuring uninterrupted business operations

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

## 1. Do Not Upgrade Across Versions

    1.28 → 1.29 → 1.30

---

## 2. Prioritize the Control Plane

    Control Plane → Worker Node

---

## 3. Rollout Upgrade

👉 Only operate on one node at a time

---

## 4. Ensure High Availability of Business Services

- Deployments should have ≥2 replicas
- Services should provide load balancing

---

# IV. Pre-upgrade Checks (Mandatory)

## 1. Cluster Status

    kubectl get nodes
    kubectl get pods -A

Requirements:

- All nodes must be in the "Ready" state
- There should be no abnormal Pods

---

## 2. Pending Pods

    kubectl get pods -A | grep Pending

👉 The number must be 0

---

## 3. Replica Check

    kubectl get deploy -A

👉 Core business services should have ≥2 replicas

---

## 4. View the Upgrade Path

    kubeadm upgrade plan

---

## 5. etcd Backup (Mandatory)

    ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F).db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key

---

# V. Control Plane Upgrade (First Master Node)

## 1. Upgrade Kubeadm

    apt-mark unhold kubeadm
    apt-get update
    apt-get install -y kubeadm=1.xx.x-00
    apt-mark hold kubeadm

---

## 2. Execute the Upgrade

    kubeadm upgrade apply v1.xx.x

---

## 3. Upgrade kubelet / kubectl

    apt-mark unhold kubelet kubectl
    apt-get install -y kubelet=1.xx.x-00 kubectl=1.xx.x-00
    apt-mark hold kubelet kubectl
    systemctl restart kubelet

---

## 4. Verify the Control Plane

    kubectl get pods -n kube-system

Focus on checking:

- kube-apiserver
- kube-controller-manager
- kube-scheduler

---

# VI. Upgrade of Multiple Control Plane Nodes (High Availability)

Execute one by one:

    kubeadm upgrade node
    systemctl restart kubelet

---

# VII. Worker Node Upgrade (One by One)

## 1. Drain the Node

    kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

---

## 2. Upgrade Kubeadm

    apt-get install -y kubeadm=1.xx.x-00
    kubeadm upgrade node

---

## 3. Upgrade kubelet

    apt-get install -y kubelet=1.xx.x-00 kubectl=1.xx.x-00
    systemctl restart kubelet

---

## 4. Restore Scheduling

    kubectl uncordon <node-name>

---

# VIII. Post-upgrade Verification (Mandatory)

## 1. Node Status

    kubectl get nodes

---

## 2. Pod Status

    kubectl get pods -A

---

## 3. Business Verification

- Check if service interfaces are accessible normally
