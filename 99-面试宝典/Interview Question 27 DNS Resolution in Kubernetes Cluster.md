---
title: "# Domain Name Resolution in K8s Clusters

## CoreDNS Configuration
- [ ] Ensure CoreDNS is installed and running in the cluster
- [ ] Configure CoreDNS to support DNS resolution for cluster services
- [ ] Set up DNS records for external services

## DNS Resolution Methods
| Method | Description | Notes |
|-------|-------------|-------|
| ClusterDNS | Resolves DNS within the cluster | Default Kubernetes DNS |
| ExternalDNS | Resolves DNS outside the cluster | Requires external DNS server |
| CustomDNS | Custom DNS configuration | Requires manual setup |

> [!note] Always verify DNS resolution with nslookup or dig commands after configuration"
tags:
  - The Book of Interviews
  - Kubernetes
  - K8s
  - Network
  - DNS
  - CoreDNS
  - ServiceDiscovery
  - Interview
category: "# Interview Bible /think"
topic: "# Kubernetes Network"
type: "# Interview Questions"
source: "# Interview Review"
created: 2026-03-18
updated: 2026-03-18
---

# Interview Question 27: DNS Resolution in K8s Clusters

## Question
How is DNS resolution performed in a K8s cluster?

## Core Answer
DNS resolution within a Kubernetes cluster is primarily handled by **CoreDNS**.

After a Pod starts, `kubelet` injects DNS configuration into the Pod. The `/etc/resolv.conf` inside the Pod typically points `nameserver` to the CoreDNS Service IP within the cluster.  
When a Pod accesses a Service domain, the request is first sent to CoreDNS, which then resolves it using Service, Endpoints, and EndpointSlice information from Kubernetes.

The standard Service domain format is generally:

```bash
service-name.namespace.svc.cluster.local
```

Example:

```bash
nginx.default.svc.cluster.local
```

If within the same namespace, a short domain name can be used directly, such as:

```bash
nginx
```

The system automatically completes the domain based on the search domain.

## Interview-Ready Version
> DNS resolution in Kubernetes clusters is primarily handled by CoreDNS.  
> Pods automatically receive DNS configuration upon startup, with the nameserver typically pointing to the CoreDNS Service IP.  
> When a Pod accesses a domain like `nginx.default.svc.cluster.local`, CoreDNS resolves it to the corresponding ClusterIP using Service and Endpoint information.  
> For Headless Services, it directly returns backend Pod IPs instead of ClusterIP.

## Key Knowledge Points

### 1. CoreDNS's Role
CoreDNS is the default DNS component in Kubernetes clusters, responsible for:

- Service discovery within the cluster
- Service domain resolution
- Some plugin-based extension capabilities

### 2. Where Does Pod's DNS Configuration Come From
After a Pod starts, `kubelet` writes DNS configuration into the Pod's `/etc/resolv.conf`.

Typical content resembles:

```bash
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Here, `nameserver` is typically the Service IP of CoreDNS.

### 3. DNS Resolution Rules
A common full domain name for a Service in Kubernetes is:

```bash
service-name.namespace.svc.cluster.local
```

Example:

```bash
mysql.default.svc.cluster.local
```

Where:
- `service-name`: Service name
- `namespace`: Namespace
- `svc`: Service identifier
- `cluster.local`: Default cluster domain

### 4. Why Can Service Names Be Used Directly in Same Namespace
Because the Pod's `/etc/resolv.conf` contains search domains:

```bash
default.svc.cluster.local
svc.cluster.local
cluster.local
```

So when accessing:

```bash
nginx
```

the system automatically completes it to:

```bash
nginx.default.svc.cluster.local
```

### 5. Differences Between Regular and Headless Services

#### Regular Service
- Resolves to **ClusterIP** by default

#### Headless Service
Configuration:

```yaml
clusterIP: None
```

Features:
- No ClusterIP is assigned
- Directly returns backend Pod IP lists
- Commonly used for StatefulSet, databases, and stateful service scenarios

## Common Follow-up Questions

### 1. How Does CoreDNS Know Which IP a Service Corresponds To?
CoreDNS obtains:
- Service
- Endpoints
- EndpointSlice
from the Kubernetes API, then generates resolution results based on these resources.

### 2. What Are the Signs of DNS Resolution Failure?
Common symptoms include:
- Pods cannot access each other via service names
- Applications report `Temporary failure in name resolution`
- Access works via IP but not via Service name
- Cross-namespace service calls fail

### 3. How to Troubleshoot K8s DNS Issues?

Check Pod DNS configuration first:

```bash
cat /etc/resolv.conf
```

Test resolution:

```bash
nslookup kubernetes.default
dig kubernetes.default.svc.cluster.local
```

Inspect CoreDNS:

```bash
kubectl get pods -n kube-system
kubectl get svc -n kube-system
kubectl logs -n kube-system <coredns-pod-name>
```

Check Service and backend endpoints:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslices
```

## One-Sentence Summary
DNS resolution in K8s clusters is primarily handled by CoreDNS. When a Pod accesses a Service domain, it first goes to CoreDNS, which resolves it to ClusterIP or Pod IP using Service and Endpoint information.

## What This Question Tests
- CoreDNS
- Service Discovery
- Source of Pod DNS configuration
- Domain completion mechanism
- Headless Service
- DNS troubleshooting approach

## Associated Tags
#TheBookOfInterviews #Kubernetes #K8s #Network #DNS #CoreDNS #ServiceDiscovery #Interview