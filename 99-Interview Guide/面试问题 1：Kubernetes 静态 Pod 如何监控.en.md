---
tags:
  - Kubernetes
  - Static Pods
  - Monitoring
  - Interviews
---

# Interview Question 1: How to Monitor Kubernetes Static Pods

## Explanation
Static Pods are directly managed by the **kubelet** and are not dynamically scheduled through the API Server.  
The monitoring methods differ from those used for Pods managed by regular Deployments/ReplicaSets.

## Steps to Follow

```bash
# View the static Pod manifest
ls /etc/kubernetes/manifests

# Check the kubelet logs
journalctl -u kubelet -f

# Verify the registration status of the static Pod in the API Server
kubectl get pod -n kube-system
```

## Monitoring Approaches

1. **Host-level monitoring**: Use NodeExporter or cAdvisor to collect metrics such as CPU and memory usage.

2. **kubelet metrics**:

```yaml
scrape_configs:
  - job_name: 'kubelet-cadvisor'
    scheme: https
    tls_config:
      insecure_skip_verify: true
    metrics_path: /metrics/cadvisor
    staticConfigs:
      - targets:
          - <node1-ip>:10250
          - <node2-ip>:10250
```

3. **Log collection**: Use Filebeat or Fluentd to collect logs from `/var/log/pods/` and send them to Elasticsearch or Loki for analysis.

## Key Points Summary

- The static Pod manifest is located in `/etc/kubernetes/manifests`.
- Kubelets register static Pods with the API Server, which can be checked using `kubectl get pod -n kube-system`.
- Common monitoring methods include host-level metrics, kubelet-provided metrics, and log collection.

## Sample Interview Answer

> “Static Pods are directly managed by the kubelet, and their manifests are typically found in `/etc/kubernetes/manifests`. Although they are not controlled by Deployments or ReplicaSets, kubelets still register them with the API Server. You can use `kubectl get pod -n kube-system` to check their status.
> There are several ways to monitor static Pods: First, use tools like NodeExporter or cAdvisor to collect host-level metrics. Second, leverage kubelet’s built-in metrics to gather container-specific data. Third, collect logs from `/var/log/pods/` and analyze them using Elasticsearch or Loki. By combining these methods, you can get a comprehensive view of the status and performance of static Pods.”