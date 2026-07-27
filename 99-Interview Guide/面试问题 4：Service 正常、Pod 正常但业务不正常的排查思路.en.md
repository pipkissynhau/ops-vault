---
tags: [Kubernetes, Service, Pod, Troubleshooting, Interview]
---

# Interview Question 4: Troubleshooting Steps When the Service and Pod Are Normal but the Business Is Not

## Explanation
Sometimes, even though both the Service and Pod in Kubernetes are in normal status, the business still encounters issues.  
This situation usually involves problems with networking, ports, service discovery, or internal application issues, requiring a systematic approach to troubleshooting.

## Steps to Follow

```bash
# 1. Check the internal service of the Pod
kubectl exec -it <pod> -- /bin/bash
curl http://localhost:<port>

# 2. View the Service configuration
kubectl get svc <service-name> -o yaml

# 3. Examine the Endpoints
kubectl get endpoints <service-name>

# 4. Verify network connectivity
kubectl exec <pod> -- ping <target-pod-ip>

# 5. Review application logs
kubectl logs <pod>
```

## Key Points
1. Normal status of Pod and Service does not guarantee normal business functionality.  
2. Check networking, ports, service discovery, and application logs for issues.  
3. Systematic troubleshooting order: internal Pod services -> Service configuration -> Endpoints -> network connectivity -> logs.

## Example Interview Answer
> "When encountering a situation where both the Service and Pod are normal but the business is not functioning correctly, I would first verify if the internal services of the Pod are accessible. For example, I would use curl to test local ports.  
> Next, I would check whether the Service configuration is correct and whether the Endpoints are set up properly to ensure traffic can be routed correctly.  
> If everything looks fine, I would then check the network connectivity between Pods to confirm they can communicate with each other. Finally, I would review the application logs to identify any potential errors or issues. By following these steps, I can effectively troubleshoot and determine the root cause of the business failure."