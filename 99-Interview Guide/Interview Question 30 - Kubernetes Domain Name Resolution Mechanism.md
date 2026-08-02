---
tags: [kubernetes, dns, service, coredns, interview frequently asked]
---

# Interview Question 30: Kubernetes Domain Name Resolution Mechanism

## 🧭 I. Key Concepts

Internal service communication in Kubernetes primarily relies on DNS (CoreDNS) for service discovery.

Pods do not need to remember IP addresses; instead, they are accessed using **Service names**.

---

## 🧱 II. DNS Components

- CoreDNS (default)
- kube-dns (historical versions)

Functions:
- Resolve Service names into ClusterIP addresses
- Support Pod DNS resolution

---

## 🌐 III. Service Domain Name Format

### 1️⃣ Fully Qualified Domain Name (FQDN)

<service-name>.<namespace>.svc.cluster.local

Example:

nginx.default.svc.cluster.local

---

### 2️⃣ Abbreviated Form (same namespace)

<service-name>

Example:

nginx

---

## 🔍 IV. Resolution Process

1. The Pod initiates a DNS query.
2. CoreDNS resolves the Service name into its corresponding ClusterIP address.
3. The result is returned to the Pod.
4. kube-proxy forwards the request to the Pod.

---

## 🔁 V. Service Types and Resolution

### ClusterIP
- Default type
- Returns a virtual IP address

---

### Headless Service

clusterIP: None

- Returns the Pod’s IP address
- Commonly used with StatefulSets

---

## 🧪 VI. Pod DNS Configuration

View configuration using:

cat /etc/resolv.conf

---

## ⚠️ VII. Common Issues

### DNS Not Working

Run the following command to check CoreDNS status:

kubectl get pods -n kube-system | grep coredns

---

### Service Not Working

Check service and endpoint configurations using:

kubectl get svc
kubectl get endpoints

---

## 🎯 VIII. Interview Summary

- CoreDNS is responsible for service discovery in Kubernetes.
- Services are accessed by their names, which are resolved into ClusterIP addresses.
- Headless Services return the Pod’s IP address directly.