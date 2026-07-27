# Kubernetes IKM Evaluation Questions (Final Review of Question 33: Questions + Answers + Explanations)
#tags: #Kubernetes #Interview #Cloud-Native #CKA

---

## 1. How does Kubernetes implement service discovery?
Answer:
- DNS
- Environment Variables

Explanation:
Kubernetes defaults to using CoreDNS for service discovery and injects the Service's environment variables when a Pod starts.
In production environments, DNS is recommended because it supports dynamic updates.

---

## 2. On what conditions can Ingress route traffic?
Answer:
- Domain name (Host)
- URL path (Path)

Explanation:
Ingress is a layer-7 (HTTP) routing component that only supports forwarding traffic based on host and path; it does not support protocol or method-based routing.

---

## 3. On which nodes are Pods created?
Answer:
- Nodes selected by the Scheduler

Explanation:
The Scheduler selects the most suitable node using filters and scores, and then kubelet creates the Pod on that node.

---

## 4. What is the role of the Kubernetes Scheduler?
Answer:
- To schedule Pods onto Nodes

Explanation:
The Scheduler is only responsible for "selecting nodes"; it does not actually run containers; this task is performed by kubelet.

---

## 5. What is the purpose of the kubectl expose command?
Answer:
- To create a Service

Explanation:
kubectl expose creates a Service based on existing resources (Pods or Deployments) to expose them externally.

---

## 6. What are the characteristics of a Secret?
Answer:
- Base64 encoding

Explanation:
By default, a Secret is only base64-encoded; it is not encrypted. For true security, etcd encryption or external key management must be enabled.

---

## 7. How does a Service select Pods?
Answer:
- Label selector

Explanation:
A Service matches Pods based on their label selectors and then generates Endpoints, which kube-proxy uses to forward traffic.

---

## 8. How can different environments be distinguished from each other?
Answer:
- Labels
- Namespaces
- Different clusters

Explanation:
Namespaces are used for small-scale deployments, while multiple clusters are used for large-scale setups. Labels provide logical identification between resources across environments.

---

## 9. Which resource provides the identity for a Pod?
Answer:
- ServiceAccount

Explanation:
By default, a Pod is bound to a ServiceAccount, which allows it to access the API Server or other cloud resources.

---

## 10. What tools are available for locally installing Kubernetes?
Answer:
- kubeadm
- minikube

Explanation:
kubeadm is used in production environments, while minikube is suitable for development and testing purposes.

---

## 11. How does NodePort achieve high availability?
Answer:
- External LoadBalancer
- Keepalived (VIP)

Explanation:
NodePort itself does not provide high availability; additional mechanisms such as an external LoadBalancer or Keepalived are required to ensure it is highly available.

---

## 12. What are the limitations of multi-cloud migration?
Answer:
- Storage
- Images

Explanation:
Different cloud providers have different storage interfaces, and image repositories may not be accessible across clouds.

---

## 13. Which of the following correctly describes Taint/Toleration?
Answer:
- PreferNoSchedule is valid.
- Toleration includes key/operator/effect.

Explanation:
Taints affect Nodes, while tolerations affect Pods.

---

## 14. What is the DNS component in Kubernetes?
Answer:
- CoreDNS

Explanation:
CoreDNS is responsible for resolving domain names for Services within the cluster.

---

## 15. What is the role of kube-proxy?
Answer:
- To forward traffic

Explanation:
kube-proxy uses iptables or ipvs to forward traffic between Services and Pods.

---

## 16. What are the different types of Services in Kubernetes?
Answer:
- ClusterIP
- NodePort
- LoadBalancer

Explanation:
ClusterIP is for internal access, NodePort is for local node ports, and LoadBalancer provides cloud-based load balancing.

---

## 17. What is an Endpoint?
Answer:
- A list of Pod IP addresses

Explanation:
An Endpoint represents the actual Pods behind a Service, providing their IP addresses for external access.

---

## 18. How does an Ingress Controller expose services?
Answer:
- LoadBalancer
- NodePort + External LB

Explanation:
An Ingress Controller is essentially another type of Service that manages traffic routing.

---

## 19. What are the common reasons for failed node scaling in Kubernetes?
Answer:
- Insufficient permissions
- Incorrect cloud credentials

Explanation:
The Cluster Autoscaler relies on cloud APIs, so any issues with these credentials or permissions can cause scaling failures.

---

## 20. How can etcd be restored?
Answer:
- Restore from a snapshot
- Force a new cluster creation

Explanation:
etcd is critical to the cluster's stability,