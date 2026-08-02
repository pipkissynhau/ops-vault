---
tags: "# Kubernetes, Pod, Resources, Requests, Limits, Interview"
---

# Interview Question 17: Pod Resource Requests & Limits (Resource Requests & Limits)

## Explanation
Kubernetes supports setting **resource requests (Requests)** and **resource limits (Limits)** for Pods or containers, ensuring scheduling and runtime performance:

- **Requests**: Used by the scheduler to determine which node a Pod can be placed on, ensuring the node has sufficient resources  
- **Limits**: The upper bound of resources for container runtime, preventing excessive consumption that could affect other Pods  

### Roles
1. Requests → Scheduling basis  
2. Limits → Runtime constraints  
3. Setting reasonable Requests/Limits helps with Pod scheduling and cluster stability  

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

## Operation / View Commands

```bash
# View Pod Resource requests and limitations
kubectl describe pod resource-demo

# View node allocation
kubectl describe node <node-name>
```

## Key Takeaways

- Requests determine scheduling nodes, Limits determine container resource upper bounds  
- Pods without set Requests/Limits may be scheduled on resource-insufficient nodes  
- Setting reasonable Requests/Limits helps maintain cluster stability, avoiding OOM or CPU contention  

## Interview Answer Example

> "In Kubernetes, Pods or containers can set resource requests and limits. Requests are used for scheduling; the scheduler uses Requests to determine if a node has sufficient resources. Limits are the upper bound of resources for container runtime, preventing Pods from consuming excessive resources that could affect other Pods.  
> For example, if a container requests 500m CPU and 256Mi memory, with limits of 1 CPU and 512Mi memory, the scheduler ensures the node has at least 500m CPU and 256Mi memory available, while the container's runtime will not exceed 1 CPU or 512Mi memory. This helps ensure reasonable resource allocation and stable operation of the cluster."