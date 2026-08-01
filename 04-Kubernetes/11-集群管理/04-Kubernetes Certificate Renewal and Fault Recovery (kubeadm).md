---
title: "# Kubernetes Certificate Renewal and Fault Recovery (kubeadm Runbook)"
tags:
  - Kubernetes
  - K8s
  - Certificate
  - TLS
  - kubeadm
  - Transport
  - Runbook
category: Kubernetes
topic: "# Cluster Management"
type: "# Learning + Hands-on Practice"
created: 2026-04-20
updated: 2026-04-20
---

# Kubernetes Certificate Renewal and Fault Recovery (kubeadm)

## Document Overview

This document is a **production-grade Runbook** for Kubernetes certificate management, used for:

- Certificate renewal
- Certificate expiration fault recovery
- Handling API Server unavailability

Goals:

- Understand the certificate system
- Master renewal operations
- Handle "cluster-wide outage" scenarios

---

# I. Why Kubernetes Needs Certificates

All Kubernetes components communicate via HTTPS:

- kube-apiserver
- kubelet
- controller-manager
- scheduler
- etcd
- kubectl

---

### Essence

👉 **Identity authentication + Data encryption**

---

# II. Certificate Locations (Must Know)

## Control Plane Certificates

    /etc/kubernetes/pki/

---

## etcd Certificates

    /etc/kubernetes/pki/etcd/

---

## kubeconfig

    /etc/kubernetes/admin.conf

---

# III. Typical Symptoms of Certificate Expiration

## kubectl Becomes Unusable

    kubectl get nodes

Error:

- x509: certificate has expired

---

## API Server Unavailability

- connection refused
- TLS error

---

## Node NotReady

    kubectl get nodes

---

## kubelet Errors

    journalctl -u kubelet -f

---

# IV. Certificate Check (Executable)

    kubeadm certs check-expiration

Check:

- Expiration times of all certificates
- Remaining validity period

---

# V. Certificate Renewal (Normal Scenario)

## 1. Renew All Certificates

    kubeadm certs renew all

---

## 2. Restart kubelet (Critical)

    systemctl restart kubelet

---

## 3. Update kubeconfig

    cp /etc/kubernetes/admin.conf ~/.kube/config

---

## 4. Verification

    kubectl get nodes
    kubectl get pods -A

---

# VI. Why Restart kubelet

Because:

👉 Components do not automatically reload after certificate updates

---

Control Plane Components:

- Static Pods (managed by kubelet)

👉 Restarting kubelet = Restarting the control plane

---

# VII. Recovery After Certificate Expiration (Critical)

⚠️ This is the most important section

---

## Scenario: kubectl is no longer usable

---

## 1. Execute on the control plane node

    kubeadm certs renew all

---

## 2. Restart kubelet

    systemctl restart kubelet

---

## 3. Wait for control plane recovery

    crictl ps

Check:

- apiserver
- controller
- scheduler

---

## 4. Restore kubectl

    export KUBECONFIG=/etc/kubernetes/admin.conf

Or:

    cp /etc/kubernetes/admin.conf ~/.kube/config

---

# VIII. API Server Won't Start (Severe Failure)

## 1. Check Containers

    crictl ps -a | grep apiserver

---

## 2. Check Logs

    crictl logs <container-id>

---

## Common Causes

- Certificate mismatch
- CA issues
- etcd unavailable

---

# IX. kubelet Certificate Issues

## Check kubelet Status

    systemctl status kubelet

---

## Check Logs

    journalctl -u kubelet -f

---

## Common Problems

- Client cert expired
- Unable to connect to apiserver

---

## Resolution

    rm -f /var/lib/kubelet/pki/*
    systemctl restart kubelet

---

# X. Multi-Control Plane Notes

- Renew on each master
- Or synchronize certificate directories

---

# XI. Production Notes

- Regularly check certificates (monthly)
- Renew in advance (don't wait until expiration)
- Keep CA files (extremely important)
- Check certificates during upgrades

---

# XII. Prohibited Actions

❌ Delete CA  
❌ Do not restart kubelet  
❌ Modify certificate paths  
❌ Test directly in production

---

# XIII. Recovery Capability Explanation

Certificate issues fall under:

👉 **Control plane-level failure**

Impact:

- kubectl unavailable
- API Server unavailable
- All nodes abnormal

---

# XIV. One-Sentence Summary

👉 Kubernetes certificate renewal essentially involves reissuing component certificates and restarting the control plane to activate them.

---

# XV. Reference Links

- Official certificate documentation  
  https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/

- TLS mechanism  
  https://kubernetes.io/docs/concepts/cluster-administration/certificates/

- kubeadm certs  
  https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/