# Kubernetes IKM Evaluation Questions (33-Question Ultimate Review: Questions + Answers + Analysis)
#tags: #Kubernetes #Interviews #Clouds. #CKA

---

## 1. How does Kubernetes implement service discovery?
Answer:
- DNS
- Environment Variables

Explanation:
Kubernetes provides service discovery by default through CoreDNS, and injects Service environment variables when Pod starts.
In production environments, DNS is recommended as it supports dynamic updates.

---

## 2. On which conditions can Ingress route traffic?
Answer:
- Host (domain name)
- Path (URL path)

Explanation:
Ingress is a seven-layer (HTTP) routing component, only supports forwarding based on host and path, does not support protocol or method.

---

## 3. On which nodes are Pods created?
Answer:
- Node selected by Scheduler

Explanation:
Scheduler selects the most suitable node through filter and score, then kubelet creates Pod on that node.

---

## 4. What is the role of Kubernetes Scheduler?
Answer:
- Schedule Pod to Node

Explanation:
Scheduler is only responsible for "selecting node", not running containers. Actual container execution is handled by kubelet.

---

## 5. What is the purpose of kubectl expose command?
Answer:
- Create Service

Explanation:
kubectl expose creates Service based on existing resources (Pod/Deployment), used to expose services externally.

---

## 6. What are the characteristics of Secret?
Answer:
- Base64 encoding

Explanation:
Secret is only base64 encoded by default, not encrypted. True security requires enabling etcd encryption or external key management.

---

## 7. How does Service select Pod?
Answer:
- Label selector

Explanation:
Service matches Pod label through selector, generates Endpoint, and forwards traffic via kube-proxy.

---

## 8. How to distinguish different environments?
Answer:
- Labels
- Namespaces
- Different clusters

Explanation:
Use namespace for small-scale, multi-cluster for large-scale, label for logical identification.

---

## 9. Which resource provides Pod identity?
Answer:
- ServiceAccount

Explanation:
Pod is bound to ServiceAccount by default, used for accessing API Server or cloud resources.

---

## 10. Local Kubernetes tool installation?
Answer:
- kubeadm
- minikube

Explanation:
kubeadm is for production, minikube is for development/testing.

---

## 11. How to achieve high availability with NodePort?
Answer:
- External LoadBalancer
- Keepalived (VIP)

Explanation:
NodePort itself lacks high availability, requires additional entry points.

---

## 12. What are the limitations of multi-cloud migration?
Answer:
- Storage
- Image

Explanation:
Different cloud vendors have different storage interfaces, image repositories may be unreachable.

---

## 13. Correct description of Taint/Toleration?
Answer:
- PreferNoSchedule is valid
- Toleration contains key/operator/effect

Explanation:
Taint acts on Node, Toleration acts on Pod.

---

## 14. What is the DNS component?
Answer:
- CoreDNS

Explanation:
CoreDNS handles domain resolution for cluster Services.

---

## 15. What is the role of kube-proxy?
Answer:
- Forward traffic

Explanation:
Achieves traffic forwarding from Service to Pod via iptables or ipvs.

---

## 16. What are the types of Service?
Answer:
- ClusterIP
- NodePort
- LoadBalancer

Explanation:
ClusterIP for internal access, NodePort for node port, LoadBalancer for cloud load balancing.

---

## 17. What is Endpoint?
Answer:
- List of Pod IPs

Explanation:
Endpoint is the collection of actual Pod IPs behind Service.

---

## 18. How to expose Ingress Controller?
Answer:
- LoadBalancer
- NodePort + External LB

Explanation:
Ingress Controller is essentially a Service.

---

## 19. Why does Node expansion fail?
Answer:
- Insufficient permissions
- Cloud credential error

Explanation:
Cluster Autoscaler depends on cloud API.

---

## 20. How to recover etcd?
Answer:
- Snapshot recovery
- Force-new-cluster

Explanation:
etcd is the core of the cluster, recovery requires caution.

---

## 21. High availability solution for Control Plane?
Answer:
- LoadBalancer
- VIP

Explanation:
Ensure apiserver is accessible.

---

## 22. How to troubleshoot Pod Pending?
Answer:
- describe pod
- events
- Resources

Explanation:
Usually related to scheduling or resource issues.

---

## 23. What is the purpose of ConfigMap?
Answer:
- Configuration management

Explanation:
Used to store non-sensitive information.

---

## 24. PV vs PVC?
Answer:
- PV: Resource
- PVC: Request

Explanation:
PVC requests PV.

---

## 25. What is the result of Retain strategy?
Answer:
- PV Released
- Data preserved

Explanation:
Data is not automatically deleted.

---

## 26. What is the role of StorageClass?
Answer:
- Dynamic storage

Explanation:
Automatically creates PV.

---

## 27. What is the role of etcd?
Answer:
- Store cluster state

Explanation:
All configurations are stored in etcd.

---

## 28. What are the control plane components?
Answer:
- apiserver
- scheduler
- controller-manager

Explanation:
Core three components.

---

## 29. What is the role of controller-manager?
Answer:
- Maintain desired state

Explanation:
Such as replica count control.

---

## 30. What is the role of kubelet?
Answer:
- Run Pod

Explanation:
Execution component on node.

---

## 31. What is the role of RBAC?
Answer:
- Permission control

Explanation:
Based on role-based authorization.

---

## 32. What is the role of ServiceAccount?
Answer:
- Pod identity

Explanation:
Used for API authentication.

---

## 33. EKS IAM mapping?
Answer:
- OIDC + IRSA

Analysis:
Implement Pod access to AWS resources.

---

## ⭐ Summary (Interview Must-Know)

Kubernetes core five components:

- Scheduler
- Network (Service / Ingress)
- Storage (PV / PVC)
- State (etcd)
- Permissions (RBAC)

---