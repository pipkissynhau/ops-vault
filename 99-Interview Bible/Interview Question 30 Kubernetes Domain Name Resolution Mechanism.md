---
tags: "[kubernetes, dns, service, coredns, Interview High Frequency]"
---

# Interview Question 30: Kubernetes DNS Resolution Mechanism

## 🧭 One: Core Concepts

Kubernetes internal service communication primarily relies on DNS (CoreDNS) for service discovery.

Pods do not need to remember IPs; they access services via **Service names**.

---

## 🧱 Two: DNS Components

- CoreDNS (default)
- kube-dns (historical version)

Function:
- Resolve Service names → ClusterIP
- Support Pod DNS resolution

---

## 🌐 Three: Service Domain Format

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

## 🔍 Four: Resolution Process

1. Pod initiates DNS query  
2. CoreDNS resolves the Service  
3. Returns ClusterIP  
4. kube-proxy forwards to Pod  

---

## 🔁 Five: Service Types and Resolution

### ClusterIP
- Default
- Returns virtual IP

---

### Headless Service

clusterIP: None

- Returns Pod IP
- Commonly used for StatefulSet

---

## 🧪 Six: Pod DNS Configuration

cat /etc/resolv.conf

---

## ⚠️ Seven: Common Issues

### DNS Not Working

kubectl get pods -n kube-system | grep coredns

---

### Service Not Working

kubectl get svc  
kubectl get endpoints  

---

## 🎯 Eight: Interview Summary

- CoreDNS implements service discovery  
- Service → ClusterIP → Pod  
- Headless returns Pod IP