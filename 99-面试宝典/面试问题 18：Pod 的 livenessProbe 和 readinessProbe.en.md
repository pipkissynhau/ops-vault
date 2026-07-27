---
tags: [Kubernetes, Pod, livenessProbe, readinessProbe, Interview]
---

# Interview Question 18: Kubernetes’ livenessProbe and readinessProbe

## Explanation
Kubernetes provides two types of health check mechanisms:

1. **livenessProbe**: Checks whether a container is running.
   - If the probe fails, kubelet will restart the container.
   - This is used to recover from hung or deadlocked containers.

2. **readinessProbe**: Checks whether a container is ready to receive requests.
   - If the probe fails, the Pod will be removed from the Service’s Endpoints.
   - This helps with traffic control and ensures that requests are not sent to unavailable containers.

### Probe Methods
- HTTP GET: Requests the container’s HTTP interface.
- TCP Socket: Checks if the port is accessible.
- Exec: Executes a command to check its exit status.

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

## Commands for Operation/Viewing

```bash
# View the Pod’s probe status
kubectl describe pod probe-demo

# View the Service Endpoints where this Pod is listed
kubectl get endpoints
```

## Key Points Summary

- **livenessProbe**: Ensures the container is running; failures trigger a restart.
- **readinessProbe**: Ensures the service is available; failures remove the Pod from service endpoints.
- Probes can use HTTP, TCP, or Exec commands.
- Setting appropriate initial delays and periodic checks helps prevent false positives/failures.

## Sample Interview Answer

> “The livenessProbe verifies if a container is alive by checking its HTTP response. If it fails, kubelet restarts the container to fix issues like deadlocks. The readinessProbe ensures that a container is ready to receive requests by checking its HTTP response. If it fails, the Pod is removed from the Service’s endpoints, so traffic isn’t sent to an unavailable instance. These probes can use HTTP, TCP, or command execution, and their settings (initialDelaySeconds and periodSeconds) help avoid misjudgments. Proper configuration of these probes enhances application stability and availability.”