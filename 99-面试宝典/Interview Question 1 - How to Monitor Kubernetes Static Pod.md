---
tags:
  - Kubernetes
  - StaticPod
  - Monitor
  - Interviews
---

# Interview Question 1: How to Monitor Kubernetes Static Pods

## Explanation
Static Pods are directly managed by **kubelet**, not dynamically scheduled through the API Server.  
Monitoring approaches differ from those for Pods managed by Deployments/ReplicaSets.

## Operation Steps

```bash
# View Static Pod manifest
ls /etc/kubernetes/manifests

# View kubelet Log
journalctl -u kubelet -f

# View Static Pod Yes. API Server Registration status
kubectl get pod -n kube-system
```

## Monitoring Methods

1. **Host-level Monitoring**: NodeExporter / cAdvisor

2. **kubelet metrics**:

```yaml
scrape_configs:
  - job_name: 'kubelet-cadvisor'
    scheme: https
    tls_config:
      insecure_skip_verify: true
    metrics_path: /metrics/cadvisor
    static_configs:
      - targets:
          - <node1-ip>:10250
          - <node2-ip>:10250
```

3. **Log Collection**: Filebeat / Fluentd collect `/var/log/pods/`, send to Elasticsearch / Loki

## Key Points Summary

- Static Pod manifest is located in `/etc/kubernetes/manifests`  
- kubelet registers with API Server, viewable via `kubectl get pod`  
- Monitoring methods: host metrics / kubelet metrics / log collection

## Interview Answer Example

> "Static Pods are directly managed by kubelet, with files typically located in `/etc/kubernetes/manifests`. Although they lack ReplicaSet or Deployment controllers, kubelet registers them with API Server, so we can check their status via `kubectl get pod -n kube-system`.  
> Common methods to monitor static Pods include: first, host-level monitoring using NodeExporter or cAdvisor to obtain CPU, memory metrics; second, scraping Pod container metrics via kubelet's metrics; third, collecting logs using Filebeat or Fluentd to gather logs from `/var/log/pods/`, sending them to Elasticsearch or Loki. Combining these methods allows comprehensive monitoring of static Pod status and performance."