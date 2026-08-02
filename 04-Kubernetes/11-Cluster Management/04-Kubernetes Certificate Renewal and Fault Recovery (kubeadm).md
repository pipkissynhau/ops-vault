---
title: Kubernetes Certificate Renewal and Fault Recovery (Kubeadm Runbook)
tags:
  - Kubernetes
  - K8s
  - Certificates
  - TLS
  - kubeadm
  - Operations
  - Runbook
category: Kubernetes
topic: Cluster Management
type: Learning + Practice
created: 2026-04-20
updated: 2026-04-20
---

# Kubernetes Certificate Renewal and Fault Recovery (Kubeadm)

## Document Description

This document is a **production-grade Runbook** for Kubernetes certificate management, used for:

- Certificate renewal
- Fault recovery in case of certificate expiration
- Handling API Server unavailability

Objectives:

- Understand the certificate system
- Master the renewal process
- Be able to handle scenarios where the entire cluster becomes unavailable

---

# I. Why Do Kubernetes Need Certificates?

All components in Kubernetes communicate via HTTPS:

- kube-apiserver
- kubelet
- controller-manager
- scheduler
- etcd
- kubectl

---

### Essence

👉 **Authentication + Data Encryption**

---

# II. Where Are the Certificates Located (Must Know)

## Control Plane Certificates

    /etc/kubernetes/pki/

---

## etcd Certificates

    /etc/kubernetes/pki/etcd/

---

## kubeconfig

    /etc/kubernetes/admin.conf

---

# III. Typical Symptoms of Expired Certificates

## kubectl Becomes Unusable

    Trying to run `kubectl get nodes` results in:

    - x509: certificate has expired

---

## API Server Unavailability

- Connection refused
- TLS errors

---

## Node NotReady

    `kubectl get nodes` returns incorrect results

---

## kubelet Reports Errors

    Check `journalctl -u kubelet -f` for details

---

# IV. Certificate Verification (Executable)

    Run `kubeadm certs check-expiration` to see:

- Expiration dates of all certificates
- Remaining validity periods

---

# V. Certificate Renewal (Under Normal Circumstances)

## 1. Renew All Certificates

    Use `kubeadm certs renew all`

---

## 2. Restart kubelet (Critical)

    Execute `systemctl restart kubelet`

---

## 3. Update kubeconfig

    Copy `/etc/kubernetes/admin.conf` to `~/.kube/config`

---

## 4. Verification

    Run `kubectl get nodes` and `kubectl get pods -A` to confirm renewal success

---

# VI. Why Restart kubelet?

Because:

👉 Components do not automatically reload updated certificates

---

For control plane components:

- They are managed by static Pods managed by kubelet

👉 Restarting kubelet means restarting the entire control plane

---

# VII. Recovery After Certificate Expiration (Critical)

⚠️ This is the most important step

---

## Scenario: kubectl Is Already Unusable

---

## 1. Execute on the Control Plane Node

    Run `kubeadm certs renew all`

---

## 2. Restart kubelet

    Execute `systemctl restart kubelet`

---

## 3. Wait for the Control Plane to Recover

    Use `crictl ps` to check if:

- apiserver
- controller
- scheduler

are running normally

---

## 4. Restore kubectl Access

    Set `KUBECONFIG` environment variable to `/etc/kubernetes/admin.conf` or copy it to `~/.kube/config`

---

# VIII. API Server Fails to Start (Severe Fault)

## 1. Check Containers

    Use `crictl ps -a | grep apiserver` to identify affected containers

---

## 2. Check Logs

    View logs of the affected container using `crictl logs <container-id>`

---

## Common Causes

- Mismatched certificates
- CA issues
- etcd unavailability

---

# IX. kubelet Certificate Issues

## Check kubelet Status

    Execute `systemctl status kubelet`

---

## Check Logs

    Use `journalctl -u kubelet -f` to diagnose problems

---

## Common Problems

- Client certificate expiration
- Inability to connect to the apiserver

---

## Solutions

    Remove outdated certificates from `/var/lib/kubelet/pki/*` and restart `kubelet`

---

# X. Multi-Control Plane Considerations

- Renew certificates for each master node
- Alternatively, synchronize certificate directories across nodes

---

# XI. Production Best Practices

- Regularly check certificates (monthly)
- Renew them in advance to avoid delays
- Keep CA files safe (extremely important)
- Verify certificates before any upgrades

---

# XII. Prohibited Operations

❌ Never delete the CA files
❌ Never skip restarting kubelet
❌ Never modify certificate paths
❌ Never test these procedures in a production environment

---

# XIII