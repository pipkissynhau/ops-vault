---
title: Domain Name Resolution in K8s Clusters
tags:
  - Interview Guide
  - Kubernetes
  - K8s
  - Networking
  - DNS
  - CoreDNS
  - ServiceDiscovery
  - Interview Questions
category: Interview Guide
topic: Kubernetes Networking
type: Interview Question
source: Interview Review
created: 2026-03-18
updated: 2026-03-18
---

# Interview Question 27: Domain Name Resolution in K8s Clusters

## Question
How is domain name resolution handled within a K8s cluster?

## Core Answer
Domain name resolution within a Kubernetes cluster is primarily managed by **CoreDNS**.

After a Pod starts, `kubelet` injects DNS configuration into the Pod. The `/etc/resolv.conf` file inside the Pod usually sets the `nameserver` to the IP address of the CoreDNS Service in the cluster.  
When a Pod accesses a Service domain name, the request is first sent to CoreDNS, which then resolves it based on the information about Services, Endpoints, and EndpointSlice in Kubernetes.

The standard format for a Service domain name is:

```bash
service-name.namespace.svc.cluster.local
```

For example:

```bash
nginx.default.svc.cluster.local
```

If they are within the same namespace, a shorter domain name like `nginx` can also be used, and the system will automatically complete it with the full domain.

## Interview Answer Example
> Domain name resolution in K8s clusters is mainly handled by CoreDNS.  
> After a Pod starts, DNS configuration is automatically injected, with `nameserver` typically set to CoreDNS’s Service IP.  
> When a Pod accesses a domain like `nginx.default.svc.cluster.local`, CoreDNS resolves it into the corresponding ClusterIP based on Service and Endpoint information.  
> For Headless Services, the backend Pod IP is returned directly instead of the ClusterIP.

## Key Knowledge Points

### 1. Role of CoreDNS
CoreDNS is the default DNS component in Kubernetes clusters and is responsible for:

- Service discovery within the cluster
- Domain name resolution for Services
- Extending resolution capabilities through plugins

### 2. Where Pod DNS Configuration Comes From
After a Pod starts, `kubelet` writes DNS configuration into the Pod’s `/etc/resolv.conf` file.

A typical example might look like this:

```bash
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The `nameserver` here is usually the IP address of CoreDNS’s Service.

### 3. Domain Name Resolution Rules
The complete domain name format for a Kubernetes Service is:

```bash
service-name.namespace.svc.cluster.local
```

For example:

```bash
mysql.default.svc.cluster.local
```

Where:

- `service-name`: The service name
- `namespace`: The namespace
- `svc`: The service identifier
- `cluster.local`: The default cluster domain

### 4. Why Can Service Names Be Used Directly in the Same Namespace
Because the Pod’s `/etc/resolv.conf` file includes search domains like:

```bash
default.svc.cluster.local
svc.cluster.local
cluster.local
```

When a user accesses `nginx`, the system attempts to automatically complete it to `nginx.default.svc.cluster.local`.

### 5. Differences in Resolution Between Regular Services and Headless Services

#### Regular Services
- Usually resolve to **ClusterIP**

#### Headless Services
Configuration:

```yaml
clusterIP: None
```

Characteristics:
- No ClusterIP is assigned
- Directly returns the list of backend Pod IPs
- Commonly used in StatefulSet, database, and stateful service scenarios

## Common Follow-up Questions

### 1. How Does CoreDNS Determine Which IP Corresponds to a Service?
CoreDNS obtains information about Services, Endpoints, and EndpointSlice through the Kubernetes API and uses this data to generate resolution results.

### 2. What Are Some Possible Issues When DNS Resolution Fails?
Common issues include:

- Inability to access services by name between Pods
- Applications reporting `Temporary failure in name resolution`
- Ability to access services by IP but not by name
- Abnormalities in cross-namespace service calls

### 3. How to Troubleshoot K8s DNS Issues?
Start by checking the Pod’s DNS configuration:

```bash
cat /etc/resolv.conf
```

Test resolution:

```bash
nslookup kubernetes.default
dig kubernetes.default.svc.cluster.local
```

Check CoreDNS:

```bash
kubectl get pods -n kube-system
kubectl get svc -n kube-system
kubectl logs -n kube-system <coredns-pod-name>
```

Verify Services and backend Endpoints:

```bash
kubectl get svc