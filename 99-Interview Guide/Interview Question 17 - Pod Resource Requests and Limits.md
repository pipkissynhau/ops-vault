---
tags: [Kubernetes, Pod, Resources, Requests, Limits, Interview]
---

# Interview Question 17: Pod Resource Requests and Limits

## Explanation
Kubernetes allows setting **resource requests** and **resource limits** for Pods or containers to ensure proper scheduling and performance:

- **Requests**: These determine which nodes a Pod can be scheduled on, ensuring that the node has sufficient resources.
- **Limits**: They define the upper limit of resources a container can consume during runtime, preventing excessive usage that might affect other Pods.

### Functions
1. **Requests** → Used as a basis for scheduling.
2. **Limits** → Serve as operational constraints.
3. Setting appropriate Requests and Limits helps with Pod scheduling and maintains cluster stability.

## Configuration Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: my-app
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"
        cpu: "1"
```

## Commands for Operations/Viewing

```bash
# View Pod resource requests and limits
kubectl describe pod resource-demo

# Check node resource allocation
kubectl describe node <node-name>
```

## Key Points Summary

- Requests determine which nodes a Pod can be scheduled on.
- Limits prevent containers from consuming excessive resources.
- Setting reasonable Requests and Limits ensures stable cluster operation and avoids issues like OOM or CPU contention.

## Sample Interview Answer

> "In Kubernetes, Pods or containers can have resource requests and limits set. Requests help the scheduler decide where to place a Pod based on available resources, while limits ensure that a container doesn’t consume too many resources. For example, if a container requests 500m of CPU and 256Mi of memory but has a limit of 1 CPU and 512Mi of memory, the scheduler will ensure that the node has at least enough of these resources available, and the container won’t exceed its limits during runtime. This helps maintain efficient resource allocation and cluster stability."