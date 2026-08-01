---
tags: "[Kubernetes, Service, Pod, Troubleshooting, Interview]"
---

# Interview Question 4: Troubleshooting Approach When Service and Pod Are Normal but Business Is Not

## Explanation
Sometimes in Kubernetes, both Service and Pod status are normal, but the business still encounters issues.  
This typically involves network, port, service discovery, or internal application problems, requiring systematic troubleshooting.

## Operation Steps

```bash
# 1. Inspection Pod Internal operations
kubectl exec -it <pod> -- /bin/bash
curl http://localhost:<port>

# 2. View Service Configure
kubectl get svc <service-name> -o yaml

# 3. View Endpoints
kubectl get endpoints <service-name>

# 4. Network connectivity
kubectl exec <pod> -- ping <target-pod-ip>

# 5. View Business Log
kubectl logs <pod>
```

## Key Takeaways

1. Pod and Service status normal ≠ business normal  
2. Check network, ports, service discovery, and application logs  
3. Systematic troubleshooting order: Pod internally -> Service configuration -> Endpoints -> Network -> Logs  

## Interview Answer Example

> "When encountering situations where Service and Pod are both normal but the business is abnormal, I would first check if the service inside the Pod is available, such as testing local ports with curl.  
> Then verify the Service configuration correctness and ensure Endpoints match the Pod, confirming traffic can route normally.  
> If everything is normal, check network connectivity to ensure Pods can communicate with each other. Finally, review application logs to find potential errors. Through this process, I can identify the root cause of the business abnormality." /think