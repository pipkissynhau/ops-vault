---
tags: "[Kubernetes, Pod, livenessProbe, readinessProbe, Interview]"
---

# Interview Question 18: Pod's livenessProbe and readinessProbe

## Explanation
Kubernetes provides two health check mechanisms:

1. **livenessProbe**: Checks if the container is alive  
   - If the probe fails, kubelet will restart the container  
   - Used to recover containers that have crashed or deadlock  

2. **readinessProbe**: Checks if the container is ready to provide service  
   - If the probe fails, the Pod will be removed from the Service's Endpoints  
   - Used for traffic control, without affecting the container itself  

### Probe Methods
- HTTP GET: Access the container's HTTP interface  
- TCP Socket: Check if the port is reachable  
- Exec: Execute a command to check the result  

## Configuration Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: my-app
    image: nginx
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

## Operation / View Commands

```bash
# View Pod The state of detection.
kubectl describe pod probe-demo

# View Pod Location Service Endpoints
kubectl get endpoints
```

## Key Points Summary

- **livenessProbe**: Ensures container survival; restarts if failure occurs  
- **readinessProbe**: Ensures service availability; removes Pod from Service Endpoints if failure occurs  
- Can use HTTP, TCP, or Exec for probing  
- Set reasonable delays and intervals to avoid false positives  

## Interview Answer Example

> [!note] "livenessProbe is used to detect if a container is alive. If the probe fails, kubelet will restart the container to prevent deadlocked or crashed applications from affecting service. readinessProbe is used to detect if a container is ready to receive traffic. If it fails, the Pod will be removed from the Service's Endpoints, ensuring traffic only goes to available instances. Probing can be done via HTTP, TCP, or command execution, and initialDelaySeconds and periodSeconds control the probing timing. Proper configuration of probes enhances application stability and availability."