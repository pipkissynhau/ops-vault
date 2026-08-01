# 00 - Business Containerization and Kubernetes Application Deployment Overview

## Document Notes
- Document Position: Mainline Overview of Kubernetes Application Deployment
- Applicable Stage: Starting Point of Business Containerization Learning / Entry Point of Application Deployment System
- Recommended Path: `04-Kubernetes/07-Apply deployment/00-Operational containerization and Kubernetes Apply deployment overview.md`

## Tags
#Kubernetes #ApplyDeployment #OperationalContainerization #Containerization #Clouds. #Transport #InterviewNotes #Deployment #StatefulSet #Service #Ingress #ConfigMap #Secret #PVC #Probe

---

## I. Why Learn Business Containerization

Many operations engineers already have the following capabilities:

- Can set up or maintain Kubernetes clusters
- Can use common resource objects
- Can handle basic network, storage, monitoring, and log issues
- Understand control plane, node components, scheduling, and service exposure

However, in actual job roles and interviews, enterprises often not only focus on "whether you can maintain a cluster," but also continue to ask:

- How to turn a business program into an image
- How to deploy a business system to Kubernetes
- How to design configurations, secrets, storage, probes, and service exposure
- Why a business Pod can't start, restarts frequently, or has access issues
- How to implement common middleware in Kubernetes
- How to quickly troubleshoot and recover from failures

Therefore, business containerization is a critical step from "platform operations capability" to "business implementation capability."

---

## II. What is Business Containerization

Business containerization, essentially, is to encapsulate business programs originally running on physical machines, virtual machines, or ordinary Linux processes into a standardized, deliverable, migratable, and schedulable container runtime unit.

Common objects include:

- Nginx static site
- Java Web service
- Python Web service
- Go binary service
- Redis, MySQL, Nacos, MinIO, etc. middleware
- Scheduled task program
- Internal tools and API services

After business containerization, applications typically have the following characteristics:

- More uniform runtime environment
- More standardized startup method
- Easier dependency encapsulation
- More consistent delivery method
- More suitable for being managed by Kubernetes for unified scheduling and management

You can understand it as:

**First, convert the business program into a standard image, then hand the standard image over to Kubernetes for deployment and operation.**

---

## III. Complete Pipeline from Business to Kubernetes

A business from its development artifact to stable operation in Kubernetes typically goes through the following process.

### 1. Prepare the Business Program
The business program may be:

- Static page files
- Java JAR package
- Python Web project
- Go compiled output
- Nginx configuration and site content
- Service program dependent on database or cache

### 2. Write Image Build File
Usually need to write a Dockerfile to define:

- Base image
- Working directory
- Dependency installation
- Business file copy path
- Startup command
- Environment variables
- Exposed ports

### 3. Build Image and Push to Repository
After image building, it's generally needed to push to an image repository, such as:

- Harbor
- Docker Hub
- Alibaba Cloud Image Repository
- Huawei Cloud SWR
- Enterprise internal image repository

### 4. Write Kubernetes Deployment Manifest
When handing the image over to Kubernetes, usually need to write:

- Deployment / StatefulSet
- Service
- ConfigMap / Secret
- Ingress
- PVC / StorageClass
- Probe
- requests / limits

### 5. Scheduled and Run by Kubernetes
Kubernetes completes the following actions based on the desired state:

- Pull image
- Create Pod
- Schedule to node
- Inject configuration
- Mount storage
- Execute health check
- Expose service access entry
- Rebuild on exception
- Rolling update on upgrade

### 6. Operations Monitoring and Fault Diagnosis
After business deployment, continuous attention is needed:

- Whether the application successfully starts
- Whether probes pass
- Whether the service is accessible
- Whether logs output normally
- Whether data is persisted
- Whether configuration takes effect
- Whether updates are smooth
- Whether faults can be quickly located

This pipeline is the coreMain of "business containerization and Kubernetes application deployment."

---

## IV. Why Kubernetes is Suitable for Hosting Containerized Business

Kubernetes' core value is not just "running containers," but "unifying application lifecycle management."

It provides the following key capabilities:

### 1. Unified Scheduling
Schedule Pods to suitable nodes based on node resource conditions.

### 2. Declarative Management
Users only need to describe the desired state, and Kubernetes will continuously converge the actual state toward the desired state.

### 3. Automatic Recovery
Controllers can automatically restart new instances after business Pods fail.

### 4. Service Discovery
Provide stable access entry for Pods through Service and DNS.

### 5. Traffic Exposure
Provide cluster internal and external access capabilities through Service, Ingress, Gateway API, etc.

### 6. Configuration and Image Decoupling
Manage configurations through ConfigMap and Secret, avoiding hardcoding configurations into images.

### 7. Storage Abstraction
Manage persistent data through PV, PVC, StorageClass.

### 8. Deployment and Updates
Support rolling updates, rollback, scaling, and version switching.

---

## V. The Most Core Questions in Learning Business Containerization

When learning application deployment, it's recommended to always focus on the following questions.

### 1. How to Convert This Business into an Image
Focus on:

- What is the entry program
- What is the startup command
- Are dependencies complete
- Which port does the application listen on
- Where are logs output
- Where are configurations read from

### 2. Which Controller is Suitable for This Business
Common choices include:

- Deployment: Suitable for stateless business
- StatefulSet: Suitable for stateful business
- DaemonSet: Suitable for scenarios where one instance runs per node
- Job / CronJob: Suitable for one-time tasks or scheduled tasks

### 3. How to Manage This Business's Configuration
Focus on:

- Where to place ordinary configurations
- How to manage sensitive information
- Whether to use environment variables or file mounting
- How the application perceives configuration updates

### 4. Does This Business Need Persistence
Focus on:

- Whether data must be retained
- Whether data can still be used after Pod recreation
- Whether PVC is needed
- Whether shared storage or local volume is required

### 5. How to Provide External Access for This Business
Focus on:

- Whether only cluster internal access is needed, or external access
- How to expose through ClusterIP, NodePort, LoadBalancer, or Ingress
- Whether domain name and layer-7 routing are needed

### 6. How to Determine the Health Status of This Business
Focus on:

- Whether the application has truly started
- Whether it's ready to receive traffic
- Whether automatic restart is needed on failure
- How to design the Probe

### 7. How to Troubleshoot Issues with This Business
Focus on:

- Pod status
- Event information
- Container logs
- Configuration mounting
- Service and Endpoints
- Ingress forwarding
- Storage mounting
- Application internal errors

---

## Six. Common Core Objects in Application Deployment

### 1. Deployment
Applicable to stateless business scenarios, such as:

- Frontend services
- Web API services
- Nginx static sites
- General business services

Typical characteristics:

- Can run multiple replicas
- Instances typically have no local critical state
- Suitable for rolling updates and elastic scaling

### 2. StatefulSet
Applicable to stateful business scenarios, such as:

- MySQL
- Redis
- Nacos
- Zookeeper
- Kafka
- Services requiring stable identity and stable storage

Typical characteristics:

- Stable Pod name
- Stable network identity
- Stable storage volume binding
- Controllable startup and deletion order

### 3. Service
Used to provide a stable access entry for a group of Pods.  
Even if Pod IPs change, the Service maintains a stable service address.

### 4. Ingress
Used for HTTP/HTTPS seven-layer traffic access.  
Suitable for domain access, path forwarding, and unified entry management.

### 5. ConfigMap / Secret
Used for configuration injection and sensitive information management.

### 6. PVC / StorageClass
Used for applying and managing persistent storage.

### 7. Probe
Used for application health checks, including:

- livenessProbe
- readinessProbe
- startupProbe

---

## Seven. Differences Between Stateless and Stateful Applications

This is a high-frequency focus point in business containerization.

### 1. Stateless Application
Stateless applications typically have the following characteristics:

- Any instance can replace another
- Do not store critical business data locally
- Scaling is relatively simple
- More suitable for Deployment
- More suitable for multiple replicas and load balancing

Common examples:

- Nginx static pages
- Ordinary API services
- Frontend page services
- Gateway services

### 2. Stateful Application
Stateful applications typically have the following characteristics:

- Instances have identity differences
- Data needs to be persisted
- Instances may have master-slave, election, or cluster relationships
- More dependent on stable network identity and stable storage
- More suitable for StatefulSet

Common examples:

- MySQL
- Redis
- Nacos
- Elasticsearch
- Kafka

### 3. Learning Focus
In subsequent learning, you need to constantly train yourself to judge:

- Is this component stateful?
- Why is it suitable for Deployment or StatefulSet?
- Where is its core data stored?
- Is data retention required after Pod recreation?

---

## Eight. Typical Technical Points in Business Containerization

### 1. Image
Focus on how images are built, pulled, and version managed.

### 2. Configuration
Focus on whether configuration is decoupled from images and whether it supports environment differentiation.

### 3. Logs
Focus on whether logs are output to standard output and whether they are easy to collect.

### 4. Networking
Focus on listening ports, Service mapping, DNS resolution, and access paths.

### 5. Storage
Focus on whether data needs to be persisted and whether PVCs are successfully bound.

### 6. Health Checks
Focus on issues like slow application startup, unready interfaces, and probe misjudgment.

### 7. Resource Planning
Focus on whether requests/limits are reasonable to avoid OOMKilled or resource contention.

### 8. Deployment Updates
Focus on image tag management, rolling updates, configuration updates, and rollback.

---

## Nine. Common Fault Points in Application Deployment

From an operations and interview perspective, business containerization is not just about "being able to deploy," but more importantly about "knowing how to troubleshoot when deployment fails."

### 1. Pod Cannot Start
Common causes:

- Image pull failure
- Startup command error
- Environment variable error
- Configuration mount error
- Storage mount failure
- Application startup error

### 2. Pod Continuously Restarts
Common causes:

- Application exits abnormally after startup
- Probe configuration is unreasonable
- Port not correctly listened
- Dependent services unreachable
- Memory shortage leading to OOMKilled

### 3. Service Exists but Access Fails
Common causes:

- Selector mismatch
- Pod not Ready
- Endpoints not generated
- Container port mismatch with Service configuration
- Application process not actually listening on target port

### 4. Domain Access Failure
Common causes:

- CoreDNS resolution anomaly
- Ingress rule error
- Ingress Controller anomaly
- Backend Service unavailable
- Seven-layer forwarding path mismatch

### 5. Data Not Persisted
Common causes:

- PVC not mounted
- Mount directory error
- StorageClass configuration error
- Application writes data to container's temporary layer

### 6. Configuration Not Effective
Common causes:

- ConfigMap not correctly mounted
- Environment variable name error
- Configuration file path error
- Application not reading mounted path
- Application not reloading configuration

---

## Ten. Basic Troubleshooting Approach for Application Deployment

In subsequent learning, gradually form a fixed troubleshooting path rather than randomly trying solutions when encountering problems.

### 1. First Check Resource Status
Common checkpoints:

- Pod status: Running
- Container status: Ready
- Replica count matches expectations
- PVC status: Bound
- Service/Endpoints status: Normal

### 2. Then Review Event Information
Events often quickly expose issues, such as:

- FailedScheduling
- FailedMount
- ImagePullBackOff
- CrashLoopBackOff

### 3. Then Check Logs
Distinguish between:

- Container startup logs
- Application runtime logs
- Ingress Controller logs
- DNS component logs
- Storage or middleware logs

### 4. Then Verify Configuration Relationships
Focus on verifying:

- Image and tag
- Startup command
- Port
- Probe
- Selector
- PVC mount path
- Configuration file path
- Domain and routing rules

### 5. Finally Enter the Application Perspective
If there are no obvious errors at the Kubernetes object level, troubleshoot from the application itself:

- Is the process actually started?
- Is the service port actually listening?
- Are dependent services reachable?
- Is the configuration syntax correct?
- Does the application itself throw exceptions?

---

## Eleven. Recommended Learning Path for Subsequent Study

Recommend progressing along the path of "from simple to complex, from stateless to stateful, from successful deployment to troubleshooting."

### Phase One: Image and Container Basics
Recommended learning content:

- Basic concepts of images, containers, and repositories
- Dockerfile basics
- CMD vs. ENTRYPOINT
- Environment variables
- Log output standards
- Image building and pushing

### Phase Two: Stateless Application Deployment
Recommended learning content:

- Deployment
- Service
- Ingress
- ConfigMap
- Secret
- Basic Probe
- Nginx static page deployment
- Simple web service deployment

### Phase Three: Stateful Application Deployment
Recommended learning content:

- StatefulSet
- Headless Service
- PVC
- StorageClass
- Deployment strategies for components like Redis, MySQL, MinIO, Nacos

### Phase Four: Application Deployment Troubleshooting
Recommended learning content:

- Image pull failed
- Container startup failed
- Probe failed
- Service unreachable
- Ingress 404 / 502
- DNS resolution failed
- PVC Pending
- Configuration mount failed

### Fifth Stage: Deployment Standardization
Recommended learning content:

- Helm
- values.yaml
- Image tag management
- Release and update strategies
- Rollback approach
- Integration with CI/CD

---

## Twelve, Common Interview Follow-up Questions

After mastering this main line, you can typically cover many interview questions.

Common questions include:

- How to containerize a business system into an image
- What's the difference between Deployment and StatefulSet
- How to inject configuration into applications in Kubernetes
- How to troubleshoot when Service is normal but the business is not working
- What's the troubleshooting approach for Ingress 404 / 502
- Why is it recommended to output container logs to stdout / stderr
- What dimensions should be checked when a Pod keeps restarting
- How to deploy stateful middleware in Kubernetes
- What scenarios are suitable for ConfigMap and Secret respectively
- What issues can arise from unreasonable probe configuration

---

## Thirteen, Recommended Documents for This Topic

After this overview, it's recommended to continue writing the following documents:

1. Overview of Images and Containers Basics
2. Dockerfile Basics and Image Build Process
3. Basics of Deploying Stateless Applications
4. Synergy between Deployment, Service, and Ingress
5. Application Configuration and Key Management: ConfigMap and Secret
6. Application Health Checks: liveness, readiness, startupProbe
7. Stateful Application Deployment: StatefulSet, Headless Service, PVC
8. Overview of Application Deployment Troubleshooting
9. Practical Deployment of Common Middleware Containers: Nginx / MinIO / Redis / MySQL / Nacos

---

## Fourteen, Stage Summary

Business containerization isn't about learning a single Kubernetes object, but establishing a complete deployment pipeline for business:

- How to containerize a business application
- How to bring the image into Kubernetes
- How to inject configuration
- How to expose services
- How to persist data
- How to perform health checks
- How to troubleshoot when failures occur
- What deployment models should be chosen for business and middleware

For operations engineers, mastering this main line means evolving from "being able to maintain platforms" to "being able to support business implementation".

This is also the key stage where Kubernetes learning transitions from understanding resource objects to practical production application implementation.

---

## Fifteen, Keyword Mnemonics

- Business Containerization: Packaging business applications as standard container units
- Image: The result of packaging application runtime environment and program
- Container: The running instance of an image
- Deployment: Controller for stateless applications
- StatefulSet: Controller for stateful applications
- Service: Provides a stable access entry for Pods
- Ingress: Provides HTTP/HTTPS layer 7 traffic entry
- ConfigMap: General configuration management object
- Secret: Object for managing sensitive information
- PVC: Persistent storage request object
- Probe: Health check mechanism

---

## Sixteen, Operational Extended Understanding

From a production operations perspective, the focus of learning business containerization isn't about "remembering how many YAML fields", but forming the following capabilities:

- Ability to determine what deployment model a business is suitable for
- Ability to understand what configurations, ports, storage, and external components a business depends on
- Ability to connect deployment, exposure, probes, logs, and storage into a closed loop
- Ability to quickly identify whether the issue is at the image layer, Pod layer, Kubernetes object layer, network layer, storage layer, or application layer when an application fails

The more solid these capabilities are, the easier it is to evolve from basic operations roles toward cloud-native operations, SRE, DevOps, and platform engineering directions.

---

## References
- Kubernetes Official Documentation: https://kubernetes.io/docs/
- Kubernetes Concepts: https://kubernetes.io/docs/concepts/
- Kubernetes Workloads: https://kubernetes.io/docs/concepts/workloads/
- Kubernetes Services, Load Balancing, and Networking: https://kubernetes.io/docs/concepts/services-networking/
- Kubernetes Storage Concepts: https://kubernetes.io/docs/concepts/storage/
- Docker Docs: https://docs.docker.com/
- Helm Docs: https://helm.sh/docs/

---

## Next Day Recommendation
Next article suggestion to organize:

[[01-Image and Container Fundamentals - Images Containers Repositories Dockerfile]]