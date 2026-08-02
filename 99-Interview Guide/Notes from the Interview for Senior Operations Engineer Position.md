# Notes from the Interview for Senior Operations Engineer Position
# Tags: #Interview #Operations #Senior Operations Engineer #Kubernetes #CICD #SRE #Cloud-Native #Observability #Automation #Fault Troubleshooting

---

## I. Core Profile of the Position

This position is essentially not about traditional "point-to-point operations and maintenance," but rather focuses on:

- Ensuring the stability of cloud-based production environments
- Operations and maintenance of Kubernetes platforms
- CI/CD and release governance
- Observability and alert systems
- Online fault emergency response and stability enhancement
- Automated operations and maintenance/platform capabilities
- Security governance and cross-team collaboration

### One-Sentence Explanation
This is a position that leans towards **cloud-native operations and maintenance/SRE/platform operations and maintenance/transition to operations and maintenance development**.

---

## II. Detailed Analysis of the Job Description

### 1) Responsible for the architecture and operations and maintenance of core business in Huawei Cloud, aiming to improve stability, availability, and scalability (SLA/SLO)

#### What Interviewers Want to Know
- Have you ever engaged in production environment operations and maintenance?
- Do you understand high availability, scalability, and stability?
- Are you familiar with SLA/SLO/SLI?
- Have you worked on capacity planning, redundancy, failover, monitoring, and alerts?

#### Answers You Should Give
- **SLA**: External service commitments, such as 99.9% availability.
- **SLO**: Internal goals, such as interface success rate, latency, and error rate targets.
- **SLI**: Specific measurement indicators, such as request success rate, P99 latency, CPU usage, etc.

#### Memorized Answers
I understand that the core of this position is not just to maintain machines but to ensure business continuity through stability measures.  
In a production environment, I would assess stability from several aspects:

1. **High availability of infrastructure**: Cloud hosts, load balancing, storage, network redundancy.
2. **High availability of the container platform**: Kubernetes control plane, node resource pools, number of workload replicas, affinity/anti-affinity settings.
3. **Controllable release process**: Canvassing, blue-green deployment, rollback, change auditing.
4. **Observability loop**: Metrics, logs, network links, alerts, duty response.
5. **Capacity and elasticity**: Capacity planning based on business peaks to avoid resource bottlenecks.

If I were to implement this, I would start by defining SLI indicators, classifying core services, setting up monitoring and alerts, controlling changes, and conducting fault reviews.

---

### 2) Responsible for the entire lifecycle management of Kubernetes clusters: planning, deployment, upgrading, scaling, fault troubleshooting, performance optimization, and security governance (whether using ACK or self-built solutions)

#### Frequently Asked Questions
- Have you built a K8s cluster?
- What lifecycle tasks have you handled?
- How do you perform upgrades?
- How do you scale up?
- How do you troubleshoot issues with failing Pods?
- If the Service is working but the business is not, how do you investigate?
- How do you ensure security in Kubernetes?

#### Structured Answers You Should Provide

### What Does the Entire Lifecycle of Kubernetes Include?
- **Planning**: Network solutions, container runtime, storage solutions, node specifications, namespace planning.
- **Deployment**: Master/worker deployment, network plugins, Ingress, monitoring, logging, storage access.
- **Daily Operations and Maintenance**: Node inspections, resource quotas, certificates, backups, monitoring, alerts.
- **Upgrading**: Version compatibility checks, canvassing upgrades, business verification, rollback plans.
- **Scaling**: Node expansion, resource pool adjustments, HPA/VPA/CA.
- **Fault Troubleshooting**: Node anomalies, Pod Pending/CrashLoopBackOff, DNS issues, storage mounting failures.
- **Security Governance**: RBAC, NetworkPolicy, image security, Secret management, audit logs.

### Memorized Answers
I understand that managing the entire lifecycle of Kubernetes involves more than just deploying clusters; it also includes continuous governance from early planning to post-release maintenance.  
If I were in charge, I would proceed as follows:

1. **Planning Phase**
   - Define the purpose of the cluster: production, testing, GPU, logging, or general-purpose.
   - Plan the network, storage, namespaces, resource quotas, and node roles.
   - Determine the image repository, Ingress, logging, monitoring, and other supporting components.

2. **Deployment Phase**
   - Install the container runtime and Kubernetes components.
   - Deploy CNI, CoreDNS, Ingress Controller, and monitoring logging systems.
   - Establish basic access controls, such as naming conventions, resource requests/limits, and probe configurations.

3. **Operations and Maintenance Phase**
   - Conduct regular node inspections, certificate checks, etcd backups, and cluster health checks.
   - Monitor issues such as Pod scheduling failures, node resource levels, disk inode problems### Ingress 是什么？
Ingress in Kubernetes serves as an entry point for managing external access to the cluster. It typically works together with an Ingress Controller, such as Nginx Ingress, to handle various functions including:

#### Functions
- Domain name routing
- HTTPS termination
- Path forwarding
- Load balancing
- Support for gradual feature deployment (grayscale release)