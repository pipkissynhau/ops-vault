---
tags: "[Kubernetes, Pod, Service, network, troubleshooting, interview]"
---

# Interview Question 20: Troubleshooting Pod Network Access and Service Relationships

## Overview
Pod network anomalies are common issues in Kubernetes that may cause Service to function normally but business access to fail.  
Troubleshooting requires understanding the Pod network model, CNI plugins, and the relationship between Service and Endpoints.

### Core Concepts
- **Pod IP**: Each Pod has an independent IP  
- **Service ClusterIP / NodePort / LoadBalancer**: Provide access entry points for Pods  
- **Endpoints**: List of Pod IPs behind a Service  
- **CNI Plugin**: Responsible for Pod network connectivity and cross-node communication  

## Operational Methods / Troubleshooting Steps

1. **Check Pod Network Connectivity**  
```bash
kubectl exec <pod-name> -- ping <target-pod-ip>
kubectl exec <pod-name> -- curl http://<service-name>:<port>
```

2. **Check Service and Endpoints**  
```bash
kubectl get svc <service-name>
kubectl get endpoints <service-name>
```

3. **Check Network of Pod's Host Node**  
- View Node IP and network plugin status  
```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
```

4. **Inspect kube-proxy and iptables/ipvs rules**  
```bash
kubectl exec <node> -- iptables -t nat -L
kubectl exec <node> -- ipvsadm -L
```

5. **Check Pod logs and container application status**  
```bash
kubectl logs <pod-name>
```

## Key Takeaways

- Communication between Pod and Service depends on **Endpoints** and **CNI network**  
- kube-proxy handles traffic forwarding for Service access  
- Troubleshooting order: Pod internal → Service configuration → Endpoints → Node network → kube-proxy → Application logs  
- Understanding Pod IP, Service types, and network plugins helps quickly identify issues  

## Interview Answer Example

> "When encountering Pod network access anomalies, I would first verify if the target Pod or Service can be accessed internally by the Pod, using ping or curl tests.  
> Then I would check Service configuration and corresponding Endpoints to confirm if traffic is correctly mapped to Pods.  
> Next, I would examine node network and kube-proxy status to ensure cross-node communication and iptables/ipvs rules are correct. Finally, I would check application logs to confirm service normalcy. This sequential approach helps quickly identify issues in Pod network, Service configuration, or the application itself."