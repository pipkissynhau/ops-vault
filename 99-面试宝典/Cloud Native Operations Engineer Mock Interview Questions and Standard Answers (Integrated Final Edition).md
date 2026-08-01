---
tags:
  - Interviews
  - Transport
  - Clouds.
  - Kubernetes
  - Mixed clouds
  - Observable
  - Prometheus
  - CI-CD
  - Helm
  - Ceph
  - GPU
  - Fault check.
---

# Cloud-Native Operations Engineer Mock Interview Questions and Standard Answers (Integrated Final Version)

> Applicable Directions: Cloud-Native Operations / Kubernetes Operations / Hybrid Cloud Operations / Entry/Mid-Level DevOps / Platform Operations  
> Usage Method: Do not memorize word-for-word, focus on memorizing "answer structure + keywords + your own real experiences"

---

# I. Self-Introduction

## 1. Please make a self-introduction

### Standard Answer

Hello, I'm SYQ, currently working in the field of cloud-native operations with about 6 years of operational experience.

In my early career, I worked at an integration and video conferencing manufacturer, participating in projects like the "Snow Bright Project", responsible for the operation of approximately 60,000 surveillance channels and 90 servers across six counties and one district. This phase laid a solid foundation in Linux, networking, and middleware.

In recent times, I've been working at theInformatization subsidiary of China Nuclear Group, mainly responsible for the operation of cloud hosts in the company's internal hybrid cloud platform, including virtual machine resource management, daily inspections, fault handling, and resource scheduling.

On this basis, I also participated in the construction of a core cloud data center, building a Kubernetes cluster from scratch and deploying a high-availability bastion host system based on it.

In terms of architecture, the cluster adopted a 3 Master + 2 Node configuration, with the control plane achieving high availability through HAProxy + Keepalived, providing a unified access entry. etcd was deployed in a three-node distributed manner to ensure data strong consistency and reliability.

In terms of storage, a dedicated Ceph cluster was connected, achieving dynamic storage provisioning through StorageClass + PVC to meet the business's persistent storage needs.

In terms of application deployment, Helm was used to manage applications, achieving parameterization and version control, while Deployment + Service + Ingress built a complete service access chain.

In terms of operations, Kubeboard was introduced for visualization management, and I have practical experience in troubleshooting Pod, Service, and network chain issues during daily operations.

Additionally, I've practiced the full CI/CD process from code submission to image building, pushing, and Kubernetes deployment.

I'm currently seeking a position in the cloud-native or cloud platform operations field, aiming to continue deepening my expertise in Kubernetes and automation operations.

### What Interviewers Want to Hear
- Who you are
- Whether your experience is genuine
- Your main direction
- Where you want to go in the future

---

# II. Kubernetes Basics and High Availability

## 2. How is your Kubernetes cluster built?

### Standard Answer

We built our Kubernetes cluster based on kubeadm.

The overall process is generally divided into several steps:

Step 1: Basic environment preparation, including installing containerd, kubelet, kubeadm on all nodes, disabling swap, configuring hostnames and networks, etc.

Step 2: Execute kubeadm init on the first Master node to initialize the cluster, while specifying the Pod network segment.

Step 3: Configure kubectl and install a CNI network plugin like Calico to ensure Pod-to-Pod network connectivity.

Step 4: Join other Master nodes and Worker nodes to the cluster via kubeadm join, where Master nodes are configured for control plane high availability.

Step 5: Optimize the control plane for high availability, such as using HAProxy + Keepalived to provide a unified entry point.

Finally, verify the cluster status, including node status, Pod status, and whether CoreDNS is running normally.

---

## 3. Why is HAProxy + Keepalived placed in front of Kubernetes control plane?

### Standard Answer

Because kube-apiserver is the unified entry point for the entire cluster control plane, and kubectl, controller-manager, scheduler, and many other components depend on it.

If there's only a single Master node, the entire cluster management plane would become unavailable if that node fails.

Therefore, a load balancer like HAProxy is typically placed in front of multiple Master nodes to forward requests to backend APIservers; Keepalived is used to provide a floating VIP for unified external access.

The significance of this approach is:
- Avoid API Server single point failure
- Provide a unified external access entry
- Automatically switch if a Master node fails

---

## 4. What does etcd do in Kubernetes?

### Standard Answer

etcd is a distributed key-value store for Kubernetes, primarily used to store core metadata for the entire cluster.

For example, objects like Pod, Node, Service, Deployment, ConfigMap, and Secret are ultimately stored in etcd.

You can simply understand it as:
- API Server is the unified entry point
- etcd is the core data storage

If etcd fails, cluster state read/write operations will be severely affected, and in extreme cases, the entire control plane will fail.

### Common Follow-up Questions
#### Why is etcd typically an odd number of nodes?
Because etcd uses the Raft consensus protocol, an odd number of nodes makes it easier to form a majority for voting, improving availability and avoiding resource waste.

---

## 5. What is the Pod creation process?

### Standard Answer
I generally understand the Pod creation process as follows:

Step 1: Request submission  
Users create a Pod via kubectl apply or a controller (like Deployment), sending the request to kube-apiserver.

Step 2: API Server processing  
API Server performs authentication, authorization, and admission control, writing the Pod object to etcd after processing.

Step 3: Scheduling phase  
Scheduler watches API Server for unscheduled Pods, selects suitable Nodes based on resources, labels, taints, etc., and writes the scheduling result back to API Server.

Step 4: Node processing  
The kubelet on the target node watches API Server for new Pods assigned to this node, then calls the container runtime (like containerd) to create containers.

Step 5: Network configuration  
During container creation, the CNI plugin assigns an IP and configures the network for the Pod.

Step 6: Service integration  
kube-proxy updates iptables or ipvs rules to ensure Service can correctly access the Pod.

Finally, the Pod enters the Running state.

---

## 6. Why does kube-apiserver need caching?

### Standard Answer

kube-apiserver needs caching mainly to reduce direct access pressure on etcd.

Because Kubernetes is a high-frequency read scenario, many components like kubelet, controller, scheduler frequently access API Server. If all requests directly hit etcd, it would lead to high QPS on etcd, affecting performance and stability.

Therefore, API Server internally implements a watch cache, watching data changes from etcd and caching them in memory. Most read requests can directly return from the cache instead of accessing etcd every time.

Kubernetes also uses the watch mechanism, allowing clients to receive resource changes via long connections, further reducing pressure on etcd.

**etcd is responsible for persistence, API Server handles caching and external services.**

---

# III. Kubernetes Troubleshooting

## 7. How to troubleshoot a Pod that won't start?

### Standard Answer
I generally follow this approach for troubleshooting:

Step 1: Check Pod status:
- Pending
- CrashLoopBackOff
- ImagePullBackOff
- ErrImagePull
- OOMKilled

Step 2: Check describe:
- `kubectl describe pod`
Focus on Events, which usually show scheduling failures, image pull failures, mount failures, or probe failures.

Step 3: Check logs:
- `kubectl logs`
If the Pod has restarted, also check `--previous` for the previous logs.

Step 4: Check nodes and resources:
- Whether nodes are Ready
- CPU / memory insufficiency
- Disk full
- Presence of taints, affinity, or unreasonable resource limits

Step 5: Check external dependencies:
- Missing ConfigMap / Secret
- PVC binding success
- Reachability of image registry
- Abnormality in dependent databases, Redis, or registry centers

---

## 8. How to troubleshoot if Service and Pod are normal but business access is abnormal?

### Standard Answer

If Service and Pod are normal but business access is abnormal, I will troubleshoot layer by layer:

Step 1: Entry layer  
First check if DNS resolves correctly and if requests reach the load balancer, such as ELB or Ingress.  
Use nslookup / dig to verify domain resolution, and use curl or browser to confirm if requests reach the entry point.

Step 2: Ingress / Gateway layer  
If there is an Ingress, check if routing rules are correct and if they match the corresponding Service.  
Also check Ingress Controller logs, such as nginx ingress.

Step 3: Service layer  
Check if Service's selector correctly matches Pod.  
Use `kubectl get endpoints` to verify if backend Pod IPs are registered normally.  
Also confirm Service type (ClusterIP / NodePort / LoadBalancer) matches the access path.

Step 4: Pod layer  
Enter the Pod and use curl localhost or service port to test, confirming if the application is truly available.  
Because Pod Running doesn't mean business is normal.

Step 5: Network layer  
Troubleshoot kube-proxy, and check if iptables / ipvs forwarding tables are correct.  
If there is NetworkPolicy, confirm if traffic is restricted.

Step 6: Application layer  
Check Pod logs `kubectl logs` for errors.  
Also verify if Readiness / Liveness Probe configurations are reasonable to avoid incorrect traffic forwarding.

Overall approach:  
**Traffic entry → Service forwarding → Pod → Application**  
Troubleshoot layer by layer to locate the issue.

---

## 9. How to troubleshoot if Service access times out?

### Standard Answer

If Service access times out, I will troubleshoot layer by layer along the network chain:

Step 1: Client side  
First confirm if requests are sent normally, using curl / telnet to test port connectivity, and use dig / nslookup to check DNS resolution.

Step 2: Entry layer (Ingress / LB)  
If accessed via domain, check if Ingress rules are correct and if the load balancer forwards to the corresponding Service.

Step 3: Service layer  
Check if Service's selector correctly matches Pod, and use `kubectl get endpoints` to confirm if backend Pod IPs are registered normally.

Step 4: kube-proxy / forwarding  
Check if kube-proxy on nodes is normal, and if iptables / ipvs forwarding tables are correct, as Service traffic forwarding depends on this layer.

Step 5: Pod layer  
Check if Pod is running normally, and enter the Pod to use curl localhost to test if the application is accessible.

Step 6: Network layer  
Check if CNI network is normal, confirm Pod IP reachability, and check for NetworkPolicy restrictions.

Step 7: Logs and events  
Finally check Pod, Ingress, and Service logs and events to locate the specific error cause.

---

## 10. How to troubleshoot if business QPS is normal, Pod is normal, but users report slowness?

### Standard Answer

If business QPS is normal, Pod is normal, but users report slowness, I will troubleshoot latency layer by layer:

Step 1: Confirm if it's a general issue  
Use Prometheus / Grafana to check interface latency, such as P95 / P99, to confirm if it's overall slow or only some requests are slow.

Step 2: Entry layer  
Check Nginx Ingress or gateway access logs, focusing on upstream_response_time and request_time to determine if the gateway is slow or the backend is slow.

Step 3: Pod / Application layer  
Enter the Pod and use curl to test interface response time.  
If the Pod internal access is also slow, it indicates an internal application issue.

Step 4: Application internal  
Check for:
- Slow SQL
- Slower third-party API calls
- Thread pool / connection pool exhaustion

Step 5: Database / dependent services  
Check database slow query logs to confirm query performance issues.  
Also check Redis / MQ for delays.

Step 6: Resource layer  
Check for CPU, memory, IO, and network bottlenecks.  
For example, high IO wait even with low CPU can cause slow responses.

Step 7: Network layer  
Check for network latency and packet loss, such as cross-node or cross-availability zone access.

Overall goal:  
**Identify which layer has slowed down, rather than just checking if the service is alive or not.**

---

# IV. Prometheus and Observability

## 11. What does Prometheus mainly monitor?

### Standard Answer
Prometheus mainly monitors hosts, containers, Kubernetes cluster objects, and business basic metrics.

Examples include:
- Host layer: CPU, memory, disk, load, network traffic
- K8s layer: Node status, Pod status, Deployment replica count, container restart count
- Resource layer: CPU / Memory usage rate, Request / Limit
- Business layer: Application online status, interface latency, error rate

Common components include:
- Node Exporter
- kube-state-metrics
- Prometheus
- Grafana
- Alertmanager

---

## 12. How to set Prometheus alert thresholds?

### Standard Answer

I generally consider Prometheus alerts from three aspects: "threshold design + alert flow + notification strategy".

# One: Alert Threshold Design  
Thresholds are not fixed, generally there are three ways:  
1) Based on experience, for example CPU, memory usage exceeds 80%  
2) Based on business metrics, for example interface latency, error rate (such as 5xx proportion)  
3) Based on historical data, using PromQL to do dynamic threshold, for example year-over-year, month-over-month or anomaly detection  

Example:  
- CPU usage > 80% for 5 minutes  
- Interface error rate > 5%  

Second: Alert Trigger Mechanism  
Prometheus will periodically calculate based on rule files' PromQL expressions, and generate alert when conditions are met, then send to Alertmanager.  

Third: Alertmanager Processing  
Alertmanager mainly responsible for:  
- Alert grouping (grouping)  
- Alert suppression (inhibit)  
- Alert severity (severity)  
- Alert routing (route)  

Fourth: Notification Methods  
- webhook (integrate with DingTalk / Enterprise WeChat)  
- Email  
- SMS (via third-party)  

Fifth: Alert Optimization  
- Set for time, avoid false alarms from transient jitter  
- Reasonable severity classification  
- Control alert quantity, avoid alert storm  

Overall chain is:  
**Prometheus → Alert Rule → Alertmanager → Notification Channel**  

---

## 13. What Does Alertmanager Do?  

### Standard Answer  

Alertmanager is the notification and management component in Prometheus alerting system.  

Prometheus is responsible for discovering issues according to rules, while Alertmanager is responsible for:  
- Alert deduplication  
- Alert grouping  
- Alert suppression  
- Alert routing  
- Alert notification  

One-sentence summary:  
**Prometheus is responsible for discovering alerts, Alertmanager is responsible for sending alerts in an orderly manner.**  

---

## 14. How Does Prometheus Monitor Network Devices?  

### Standard Answer  

Prometheus monitors network devices, generally through SNMP protocol.  

Because most network devices, such as switches, routers, firewalls, do not directly expose Prometheus metrics, so need to use snmp_exporter as intermediate conversion.  

Overall process is:  

1) Enable SNMP (v2c or v3) on network devices  
2) snmp_exporter pulls device data via SNMP OID  
3) snmp_exporter converts data into Prometheus metrics format  
4) Prometheus periodically scrapes snmp_exporter's /metrics interface  

Common metrics include:  
- Interface traffic  
- Interface status  
- Packet loss rate  
- CPU / Memory  
- Port error rate  

If there are many devices, generally manage through bulk configuration or auto discovery.  

---

## 15. What Is Prometheus Service Dynamic Discovery?  

### Standard Answer  

Service dynamic discovery refers to the system's ability to automatically discover service instance changes and dynamically update configurations.  

In cloud-native environments, since Pod IP and instance numbers are dynamically changing, it must rely on service discovery mechanisms.  

In Prometheus, it is mainly implemented through service discovery, common methods include:  
- Kubernetes service discovery  
- file_sd  
- Cloud vendors or registry, such as Consul  

Prometheus will periodically refresh target lists and process through relabel_configs.  

This can realize automatic monitoring of instance online and offline removal, reduce manual maintenance cost.  

---

# Five, CI/CD and Deployment Pipeline  

## 16. What Is Your Understanding of the Complete CI/CD Process?  

### Standard Answer  

A complete CI/CD process, I generally design according to the following chain:  

First step: Code submission (CI trigger)  
Developers submit code to GitLab repository, trigger GitLab CI pipeline via webhook, defined by `.gitlab-ci.yml`.  

Second step: CI stage (build and test)  
GitLab Runner executes pipeline, mainly includes:  
- Code compilation  
- Unit test  
- Build Docker image  

Third step: Image push  
Push built image to Harbor or other image repository, usually add tag, such as commit id or version.  

Fourth step: CD stage (deployment)  
Through pipeline or CD tool, such as kubectl / helm, deploy new image to Kubernetes.  

Fifth step: Release strategy  
[Question 41: Difference between K8s Rolling Update, Blue-Green Release, and Gray Release Notes]  
Can combine:  
- Rolling Update  
- Canary release  
- Blue-green release  

Sixth step: Rollback mechanism  
If new version has issues:  
- Can quickly rollback via `kubectl rollout undo`  
- Or redeploy old version image  

Overall chain is:  
**Code submission → CI build → Image repository → CD deployment → K8s running**  

In production environment, I also consider:  
[Question 42: Difference between Kaniko, Docker-in-Docker, and Buildah Notes]  
- Use kaniko to avoid Docker in Docker security issues  
- Do cache optimization to reduce build time  
- Add failure retry and notification mechanism  

---

## 17. How Do You Understand GitLab CI and Jenkins?  

### Standard Answer  
Both are essentially pipeline platforms, but with slightly different focuses.  

Jenkins is older, more flexible, with rich plugin ecosystem, suitable for complex custom workflows, but usually higher maintenance cost.  

GitLab CI is more tightly integrated with code repository, usually defined via `.gitlab-ci.yml`, configuration is more code-based, better for version management.  

I understand the core difference is not the tool name, but:  
- Whether the build, artifact, deployment, and rollback process is normalized  
- Whether the workflow is automated and traceable  

---

## 18. How Would You Answer If the Interviewer Asks You to Explain `.gitlab-ci.yml` On the Spot?  

### Standard Answer  
I can write a basic structure, such as defining stages and job.  

Example:  

```yaml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t myapp:latest .
    - docker push myapp:latest

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=myapp:latest
```  

But specific details I usually modify based on existing templates, since different projects have different build methods, such as Java, Node, Go.  

I'm more familiar with overall process design and deployment pipeline.  

I once set up a simple flow:  
After code submission to GitLab, trigger pipeline,  
via GitLab Runner execute build tasks,  
use docker build to build image and push to Harbor,  
then deploy to Kubernetes cluster via kubectl or helm.  

The flow is fully runnable.  

CI/CD YAML syntax I have used in practice, but won't memorize itI mean it..  
Generally design pipeline based on business process, specific syntax refer to official documentation or existing templates for adjustment.  

I focus more on:  

- What each stage does  
- How to build image  
- How to deploy to K8s  
- How to handle failures  

---

## 19. What Are Harbor and Nexus Used For?  

### Standard Answer

# Harbor is primarily a private image repository used to manage Docker images, supporting capabilities such as project isolation, image version management, and access control.

Nexus is more of a general-purpose artifact repository, capable of managing dependencies like Maven, npm, and yum, and can also serve as an internal dependency source.

In a containerized release pipeline, the common division of labor is:

- Nexus: Manages dependencies and artifact sources
- Harbor: Manages container images
- CI tools: Handles build processes
- Kubernetes: Manages deployment and runtime

---

# SixI don't know.Helm Basics

## 20. What is Helm?

### Standard Answer

Helm is a package management tool for Kubernetes, capable of packaging a set of Kubernetes resource files into a Chart for convenient unified installation, upgrades, rollbacks, and parameterized management.

You can think of Helm as:  
**The yum/apt of Kubernetes**  
Or understand it as:  
**A templating way to manage K8s application deployments**

Without Helm, deploying an application might require manually maintaining multiple YAML files for Deployment, Service, Ingress, ConfigMap, Secret, etc.  
With Helm, you can consolidate these into a Chart and pass parameters uniformly through values.yaml.

---

## 21. What is the value of Helm?

### Standard Answer

Helm's core value includes several aspects:

- **Parameterized deployment**: Only modify values.yaml for different environments, avoiding duplicating many YAML files
- **Version management**: Supports release concepts, facilitating upgrades and rollbacks
- **Strong reusability**: The same template can deploy multiple environments
- **High deployment efficiency**: Install an entire application with a single command
- **Suitable for team collaboration**: Standardizes application delivery methods

---

## 22. What are the core concepts of Helm?

### Standard Answer

Remember these key concepts:

- **Chart**: Application package containing a set of templates and configurations
- **Release**: The running instance generated when a Chart is installed in a cluster
- **values.yaml**: The parameter file for a Chart
- **templates/**: Directory for Kubernetes resource templates
- **helm install**: Install an application
- **helm upgrade**: Upgrade an application
- **helm rollback**: Roll back an application

---

## 23. How to answer when asked about using Helm in an interview?

### Standard Answer

In my projects, I mainly use Helm to manage application deployments on Kubernetes.

My understanding of Helm is that it's suitable for managing applications composed of multiple resource objects, such as Deployment, Service, Ingress, ConfigMap, etc., which can be uniformly templated.

In actual use, Helm's value mainly lies in:

- Different environments are distinguished through values.yaml
- Helm upgrade can be used for application upgrades
- Rollback is supported when issues occur
- More standardized than directly maintaining large volumes of YAML files

I tend to use Helm from an operations and delivery perspective rather than developing complex Charts myself.

---

# SevenI don't know.Ceph Basics

## 24. What is Ceph?

### Standard Answer

Ceph is an open-source distributed storage system that provides object storage, block storage, and file storage capabilities.

In Kubernetes scenarios, the most common use is to use Ceph as a backend storage to provide persistent volumes for containers.

You can initially understand Ceph as:  
**A distributed storage cluster that organizes disks across multiple machines to provide unified storage capabilities.**

---

## 25. Why is Ceph suitable as a backend storage for Kubernetes?

### Standard Answer

Because Kubernetes itself is dynamically scheduled, Pods can migrate to different nodes, and if there are persistence requirements, it cannot rely solely on local disks.

Ceph's advantages include:

- Distributed storage with scalable capacity
- Data multi-replication for high reliability
- Support for dynamic provisioning
- Good integration with Kubernetes via CSI

Therefore, the common solution in K8s is:  
**Ceph + StorageClass + PVC**  
To achieve dynamic storage allocation.

---

## 26. What are the core components of Ceph?

### Standard Answer

Remember these most core components first:

### 1I'm not sure.MONI'm sorry.MonitorI'm not sure.

Responsible for maintaining cluster status and metadata, similar to one of the "brains".  
It doesn't store real data but is responsible for perceiving cluster health and member information.

### 2I'm not sure.OSDI'm sorry.Object Storage DaemonI'm not sure.

This is the component that actually stores data.  
Each OSD typically corresponds to a disk or storage instance, responsible for data read/write, replication, and recovery.

### 3I'm not sure.MGRI'm sorry.ManagerI'm not sure.

Responsible for supplementing monitoring and management capabilities, providing status display and some extension modules.

### 4I'm not sure.MDSI'm sorry.Metadata ServerI'm not sure.

Mainly used for CephFS, if only using RBD block storage, MDS may not be core.

---

## 27. What are the three common storage interfaces of Ceph?

### Standard Answer

Ceph commonly has three external capabilities:

### 1I'm not sure.RBDI'm sorry.Block StorageI'm not sure.

Most commonly used in Kubernetes.  
Can provide block devices for Pods, often as the backend for PVC.

### 2I'm not sure.CephFSI'm sorry.File StorageI'm not sure.

Provides shared file system capabilities, suitable for multi-instance shared read/write scenarios.

### 3I'm not sure.RGWI'm sorry.Object StorageI'm not sure.

Provides an object storage interface similar to S3.

---

## 28. Why won't Ceph data be lost easily?

### Standard Answer

Because Ceph is a distributed storage system, it typically uses multi-replication or erasure coding to ensure data reliability.

The most common is multi-replication mechanism, such as 3 replicas:

- One primary data
- Two replicas

This way, even if a disk or machine fails, data can still be recovered from other replicas.

---

## 29. How is Ceph typically used in Kubernetes?

### Standard Answer

In Kubernetes, Ceph is typically accessed via CSI.

The basic workflow is:

- Administrators first prepare storage pools in Ceph
- Define StorageClass in Kubernetes
- Business creates PVC
- CSI dynamically requests storage from Ceph based on PVC
- Pods use persistent volumes after mounting PVC

You can simply remember it as:

**Ceph provides backend storage capabilities, and Kubernetes dynamically requests through StorageClass + PVC.**

---

## 30. How to answer confidently if asked about Ceph knowledge in an interview?

### Standard Answer

My understanding of Ceph currently focuses on the layer of Kubernetes persistent storage integration.

I know Ceph is a distributed storage system with core components including MON, OSD, MGR, and common external capabilities like RBD, CephFS, and RGW.

In Kubernetes scenarios, I focus on understanding how Ceph serves as a backend storage, combined with StorageClass and PVC to provide dynamic persistent volumes.

If asked about very low-level Ceph operations, such as CRUSH map, PG tuning, and fault recovery details, I currently don't have deep production maintenance experience. However, I understand the overall architecture and the K8s integration path.

### Key Point

Don't claim to be "proficient in Ceph".  
You're better suited to say:

- I understand the basic principles  
- I know the K8s integration path  
- Deep-level operations are still being supplemented  

---

# VIII. Kubernetes GPU Node Scheduling

## 31. How does GPU scheduling work in Kubernetes?

### Standard Answer

GPU scheduling in Kubernetes essentially ensures workloads requiring GPU resources only run on nodes with GPU resources, and the scheduler allocates them based on resource declarations.

Unlike regular CPU and memory, GPU is typically a special hardware resource. By default, Kubernetes cannot directly use it. Usually, you need to first install the corresponding drivers, container runtime support, and NVIDIA Device Plugin for Kubernetes to recognize GPU resources.

The general approach is usually:

Step 1: Install GPU drivers on GPU nodes  
For example, NVIDIA drivers, nvidia-container-toolkit, etc.

Step 2: Deploy Device Plugin in the cluster  
The most common is the NVIDIA-provided device plugin, which registers GPU resources on the node to Kubernetes.

Step 3: Declare GPU resource requests in the Pod  
For example, declaring a need for 1 GPU in resource limits.

Step 4: The scheduler schedules the Pod to a node with available GPU resources.

One-sentence summary:  
**First, let the cluster recognize GPU resources, then let the business declare GPU needs, and finally let the scheduler place the Pod on the appropriate GPU node.**

---

## 32. How does Kubernetes recognize if a node has GPU?

### Standard Answer

Kubernetes itself does not directly recognize GPU. It typically relies on GPU vendor-provided Device Plugins.

Taking NVIDIA as an example:

- First install NVIDIA drivers on the node  
- Install container runtime support for GPU  
- Deploy nvidia device plugin  

After deployment, the device plugin registers GPU resources to the node status, and Kubernetes can see similar entries:

- `nvidia.com/gpu: 1`  
- `nvidia.com/gpu: 4`  

The scheduler then knows which GPU resources are available on this node.

---

## 33. How does a Pod request GPU?

### Standard Answer

A Pod requests GPU by declaring extended resources in resource limits, for example:

resources:  
  limits:  
    nvidia.com/gpu: 1  

This indicates the container needs 1 GPU.

GPU and regular resources have a common characteristic:

- Usually only declared in limits  
- Requests and limits are generally consistent  
- GPU resources typically cannot be oversubscribed like CPU  

After the scheduler sees this declaration, it will schedule the Pod to a node with sufficient GPU resources.

---

## 34. How to ensure GPU tasks only run on designated GPU nodes?

### Standard Answer

Common approaches are three:

### 1) Directly rely on GPU resource declarations

If the Pod declares `nvidia.com/gpu`, the scheduler will naturally only choose nodes with GPU resources.

### 2) Combine with nodeSelector or nodeAffinity

For example, label GPU nodes:

gpu=true  

Then configure in the Pod:

nodeSelector:  
  gpu: "true"  

Or use more flexible nodeAffinity.

### 3) Combine with taint / toleration

To prevent regular workloads from scheduling to GPU nodes, you can taint GPU nodes, for example:

kubectl taint nodes gpu-node dedicated=gpu:NoSchedule  

Only Pods with corresponding tolerations can run on these nodes.

In production, it's common to use:  
**Resource declaration + label affinity + taint toleration**  
together, to avoid wasting GPU resources.

---

## 35. Why do GPU nodes often need taints?

### Standard Answer

Because GPU nodes are usually expensive and resources are scarce. Without restrictions, regular workloads might be scheduled there, causing resource waste.

In production, common practice is to add taints to GPU nodes to prevent default workloads from scheduling there, allowing only workloads explicitly declaring GPU needs or with tolerations to use these nodes.

Benefits include:

- Preventing regular workloads from occupying GPU nodes  
- Ensuring high-value resources are reserved for actual needs  
- Improving cluster resource utilization and controllability  

---

## 36. How to troubleshoot GPU Pod scheduling failures?

### Standard Answer

If a GPU Pod scheduling fails, I generally troubleshoot in the following order:

Step 1: Check Pod Events  
Use:

- `kubectl describe pod`  
    Focus on whether it shows:
- `Insufficient nvidia.com/gpu`
- Node mismatch  
- No toleration  
- Affinity conditions not met  

Step 2: Check if the node actually recognizes GPU  
Use:

- `kubectl describe node`  
    Confirm if the node resources include:
- `nvidia.com/gpu`  

Step 3: Check if the device plugin is working  
Verify if the NVIDIA device plugin Pod is running normally, and check for errors in the logs.

Step 4: Check if GPU drivers are working  
Confirm that NVIDIA drivers and container runtime support are correctly installed on the node.

Step 5: Check scheduling constraints  
Verify if you've written:

- nodeSelector  
- nodeAffinity  
- taints / tolerations  
    Whether these conditions have excluded the node.

Step 6: Check if resources are fully occupied  
For example, if a node has 4 GPUs but they're all occupied by other Pods, new Pods will also fail to schedule.

---

## 37. How to answer "Do you know Kubernetes GPU scheduling" in an interview?

### Standard Answer

My understanding of Kubernetes GPU scheduling mainly lies in the scheduling pipeline and resource management layer.

I know GPU nodes need to first install drivers and device plugins to let Kubernetes recognize `nvidia.com/gpu` such extended resources. After business Pods declare GPU needs via resource limits, the scheduler will place them on suitable GPU nodes.

In production management, I also understand that labels, affinities, taints, and tolerations are typically combined to prevent regular workloads from occupying GPU nodes.

If asked about very low-level GPU driver compatibility, CUDA version matrices, or AI framework-side issues, I currently don't have deep production experience. However, I understand the Kubernetes-layer scheduling mechanisms and troubleshooting approaches.

### Key Point

Don't claim to be "proficient in AI platforms".  
A more stable statement is:

- I understand the GPU node scheduling mechanism  
- I know the core points of resource declaration, device plugin, labels, and taints  
- More foundational AI training framework experience is still being accumulated  

---

# IX. Hybrid Cloud and Troubleshooting  

## 38. What is your understanding of hybrid cloud operations?  

### Standard Answer  

Hybrid cloud operations involve maintaining both private and public cloud resources simultaneously, ensuring their coordination in networking, business, security, permissions, monitoring, and fault handling.  

The challenge isn't just the volume of resources, but:  

- Complex network interoperability  
- Unclear responsibility boundaries  
- Longer problem traceability chains  
- Significant differences between on-cloud and on-premises components  

Thus, the core of hybrid cloud operations isn't merely "maintaining resources"—it's more importantly:  
**Cross-environment coordination, complex chain troubleshooting, and stability assurance.**  

---

## 39. What are cloud connections and peering connections used for?  

### Standard Answer  

They essentially solve network interoperability issues.  

- **Peering Connection**: Better suited for point-to-point communication between two VPCs  
- **Cloud Connection**: Better suited for unified interconnection across multiple regions and network instances  

From an operations perspective, focus should be on:  

- Whether routing is correct  
- Whether IP ranges conflict  
- Whether security groups / ACLs / firewalls are open  
- Whether latency and link stability meet business requirements  

---

## 40. How do you troubleshoot hybrid cloud access anomalies?  

### Standard Answer  

I generally troubleshoot in the order of "from near to far, from simple to complex":  

1. Confirm the phenomenon: Is it completely unreachable, intermittent timeouts, or partial business anomalies?  
2. Check the business itself: Processes, ports, resources, and logs  
3. Check local network and security policies: Routing, firewalls, security groups, DNS  
4. Check cross-cloud links: Cloud connections, peering connections, load balancers, WAF, cloud firewalls  
5. Capture packets if necessary to confirm where the three-way handshake fails  

---

# X. Must-Remember Interview Expression Principles  

## 1. Don't say "participated" when you mean "didn't do it"  

You can say:  

- I was responsible for the operations support portion  
- I handled the implementation and troubleshooting phases  
- I led the user-side issue resolution process  

## 2. Don't overstate your Ceph expertise  

A safer approach is:  

- I understand Ceph's basic architecture  
- I know its role in Kubernetes persistence  
- More foundational Ceph operations are still being built  

## 3. Don't overstate GitLab CI YAML skills  

A better approach is:  

- I've written basic pipelines  
- I can understand the full release chain  
- Complex YAMLs are usually adjusted based on templates and documentation  

## 4. Don't claim GPU expertise as AI platform expertise  

A safer approach is:  

- I understand Kubernetes GPU scheduling mechanisms  
- I know device plugin, resource declaration, node labels, and taints  
- More foundational CUDA / AI framework experience is still being accumulated  

## 5. Answers should have layers  

Prioritize this structure:  

- Start with background  
- Then explain the chain  
- Then highlight key points  
- Finally discuss risks and experience  

---

# XI. Memorization Version  

## What is Helm  

Helm is a package management tool for Kubernetes, used to template, parameterize, and versionize a group of K8s resources for convenient installation, upgrades, and rollbacks.  

## What is Ceph  

Ceph is a distributed storage system, commonly used as backend persistent storage for Kubernetes, with common components including MON, OSD, MGR.  

## What is GPU scheduling  

K8s GPU scheduling is the process of first having the cluster identify GPU resources via device plugin, then letting Pods declare requirements via `nvidia.com/gpu`, and finally scheduling to appropriate GPU nodes.  

## Pod Creation Process  

Request enters API Server → Written to etcd → Scheduler schedules → kubelet creates containers → CNI configures network → kube-proxy integrates with Service.  

## Service is normal but business is unreachable  

Troubleshoot layer by layer: entry point, Ingress, Service, Endpoints, Pod, kube-proxy, and application logs.  

## Prometheus Alerts  

Prometheus identifies issues based on rules, while Alertmanager handles grouping, suppression, routing, and notifications.  

## CI/CD Process  

Code commit → Pipeline triggered → Build and test → Image pushed to Harbor → Helm / kubectl deployment → Release validation → Rollback.  

## GPU Pod Scheduling Failure  

First check Pod events, then check if nodes have `nvidia.com/gpu`, then check device plugin, drivers, scheduling constraints, and resource saturation.