---
tags: [Kubernetes, Pod, Service, Network, Troubleshooting, Interview]
---

# Interview Question 20: Troubleshooting Pod Network Access and Service Relationships

## Explanation
Network issues with Pods are common in Kubernetes and can prevent services from being accessible despite appearing to be configured correctly.  
When troubleshooting, it is essential to understand the Pod network model, CNI plugins, and the relationship between Services and Endpoints.

### Key Concepts
- **Pod IP**: Each Pod has its own unique IP address.
- **Service ClusterIP / NodePort / LoadBalancer**: These provide access points for Pods.
- **Endpoints**: A list of Pod IPs behind a Service.
- **CNI Plugins**: Responsible for Pod network connectivity and cross-node communication.

## Steps to Troubleshoot

1. **Check Pod Network Connectivity**
```bash
kubectl exec <pod-name> -- ping <target-pod-ip>
kubectl exec <pod-name> -- curl http://<service-name>:<port>
```

2. **Verify Service and Endpoints**
```bash
kubectl get svc <service-name>
kubectl get endpoints <service-name>
```

3. **Check Node Network**
- View Node IP addresses and network plugin status:
```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
```

4. **Inspect kube-proxy and iptables/ipvs Rules**
```bash
kubectl exec <node> -- iptables -t nat -L
kubectl exec <node> -- ipvsadm -L
```

5. **Check Pod Logs and Application Status**
```bash
kubectl logs <pod-name>
```

## Key Points
- Communication between Pods and Services relies on **Endpoints** and **CNI networks**.
- kube-proxy is responsible for forwarding Service access requests.
- The troubleshooting sequence should be: internal Pod checks → Service configuration → Endpoints → Node network → kube-proxy → application logs.
- Understanding Pod IPs, Service types, and network plugins helps quickly identify the issue.

## Example Interview Answer
> “When encountering network access issues with a Pod, I would first verify if the Pod can communicate with the target Pod or Service using ping or curl.  
> Then, I would check the Service configuration and its associated Endpoints to ensure traffic is correctly directed.  
> Next, I would examine the node’s network settings and kube-proxy status to confirm there are no issues with cross-node communication. Finally, I would review the Pod logs and application status to determine if the service is functioning properly. By following this sequence, I can quickly identify whether the problem lies in the Pod’s network, Service configuration, or the application itself.”