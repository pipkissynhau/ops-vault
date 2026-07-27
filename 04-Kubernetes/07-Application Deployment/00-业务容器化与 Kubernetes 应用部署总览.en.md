# 00-Business Containerization and Kubernetes Application Deployment Overview

## Document Description
- Document Position: Main overview of Kubernetes application deployment
- Applicable Phase: Starting point for learning business containerization / Entry to the application deployment system
- Recommended Path: `04-Kubernetes/07-Application Deployment/00-Business Containerization and Kubernetes Application Deployment Overview.md`

## Tags
#Kubernetes #Application Deployment #Business Containerization #Containerization #Cloud-Native #Ops #Interview Notes #Deployment #StatefulSet #Service #Ingress #ConfigMap #Secret #PVC #Probe

---

## I. Why Learn Business Containerization

Many Ops engineers already possess the following capabilities:

- They can set up or maintain Kubernetes clusters.
- They know how to use common resource objects.
- They can handle basic network, storage, monitoring, and logging issues.
- They have some understanding of the control plane, node components, scheduling, and service exposure.

However, in actual job roles and interviews, companies often look beyond just "the ability to maintain clusters" and also focus on:

- How to turn a business program into an image.
- How to deploy a business system on Kubernetes.
- How to design configurations, secrets, storage, probes, and service exposure.
- Why a business Pod fails to start, restarts frequently, or is inaccessible.
- How to implement common middleware in Kubernetes.
- How to quickly diagnose and recover from failures.

Therefore, business containerization is a crucial step in moving from "platform Ops capabilities" to "business implementation capabilities."

---

## II. What is Business Containerization

Business containerization essentially involves encapsulating business programs that originally run on physical machines, virtual machines, or ordinary Linux processes into standardized, deliverable, migratable, and schedulable container units.

Common examples include:

- Nginx static websites
- Java Web services
- Python Web services
- Go binary services
- Middleware such as Redis, MySQL, Nacos, MinIO
- Scheduled task programs
- Internal tools and interface services

After business containerization, applications typically have the following characteristics:

- More unified running environments.
- More standardized startup methods.
- Easier dependency management.
- Consistent delivery methods.
- Better suitability for unified scheduling and management by Kubernetes.

You can think of it as:

**First, turn the business program into a standard image, and then use Kubernetes to deploy and run that image.**

---

## III. The Complete Process of Moving a Business from Program to Kubernetes

For a business to move from its development stage to stable operation on Kubernetes, it usually goes through the following processes.

### 1. Prepare the Business Program
The business program may be:

- Static page files.
- Java JAR packages.
- Python Web projects.
- Go compiled products.
- Nginx configuration and site content.
- Service programs that rely on databases or caches.

### 2. Write the Image Build File
You typically need to write a Dockerfile to define:

- The base image.
- The working directory.
- Dependency installation.
- The path to copy business files.
- The startup command.
- Environment variables.
- External exposure ports.

### 3. Build the Image and Push It to a Repository
After building the image, it is usually pushed to an image repository, such as:

- Harbor.
- Docker Hub.
- Alibaba Cloud Image Repository.
- Huawei Cloud SWR.
- Internal enterprise image repositories.

### 4. Write the Kubernetes Deployment Manifest
When submitting the image to Kubernetes, you typically need to write:

- Deployment/StatefulSet configurations.
- Service definitions.
- ConfigMap/Secret settings.
- Ingress rules.
- PVC/StorageClass specifications.
- Probe configurations.
- requests/limits settings.

### 5. Kubernetes Scheduling and Execution
Kubernetes performs the following actions based on the desired state:

- Pulls the image.
- Creates Pods.
- Schedules them onto nodes.
- Injects configuration.
- Mounts storage.
- Executes health checks.
- Exposes service access points.
- Rebuilds in case of exceptions.
- Performs rolling updates during upgrades.

### 6. Ops Monitoring and Fault Diagnosis
After the business goes live, it is necessary to continuously monitor:

- Whether the application starts successfully.
- Whether probes pass.
- Whether services are accessible.
- Whether logs are being output normally.
- Whether data is being persisted.
- Whether configurations take effect.
- Whether updates go smoothly.
- How quickly faults can be located.

This process represents the core mainline of "business containerization and Kubernetes application deployment."

---

## IV. Why Kubernetes Is Suitable for Running Containerized Businesses

The core value of Kubernetes is not just "running containers" but "unifying the management of the application lifecycle."

It provides key capabilities including:

### 1. Unified Scheduling
Pods can be scheduled to appropriate nodes based on available resources.

### 2. Declarative Management
Users only need to describe the desired state, and Kubernetes will### 3. Service is Present but Unreachable
Common causes:

- Selector mismatch
- Pod not Ready
- Endpoints not generated
- Container port does not match Service configuration
- Application process is not listening on the target port

### 4. Domain Name Access Failure
Common causes:

- CoreDNS resolution issue
- Ingress rule error
- Ingress Controller malfunction
- Service backend unavailable
- Layer-7 forwarding path mismatch

### 5. Data Not Persisted
Common causes:

- PVC not mounted
- Mount directory error
- StorageClass configuration incorrect
- Application writes data to temporary container layer

### 6. Configuration Not Taking Effect
Common causes:

- ConfigMap not correctly mounted
- Incorrect environment variable names
- Wrong configuration file path
- Application does not read the mount path
- Application does not reload configurations

---

## Section X: Basic Troubleshooting Approaches for Application Deployment

In subsequent learning, you should gradually establish a fixed troubleshooting process rather than trying various solutions at random when problems arise.

### 1. Check Resource Status First
Common inspection points:

- Is the Pod Running?
- Is the container Ready?
- Does the number of replicas match expectations?
- Is the PVC Bound?
- Are Service/Endpoints functioning normally?

### 2. Examine Event Information Next
Events can often quickly reveal issues, such as:

- FailedScheduling
- FailedMount
- ImagePullBackOff
- CrashLoopBackOff

### 3. Review Logs Then
It is important to distinguish between:

- Container startup logs
- Application runtime logs
- Ingress Controller logs
- DNS component logs
- Storage or middleware-specific logs

### 4. Verify Configuration Relationships
Focus on checking:

- Images and tags
- Startup commands
- Ports
- Probes
- Selectors
- PVC mount paths
- Configuration file paths
- Domain names and routing rules

### 5. Finally, Look Inside the Application
If there are no obvious errors at the Kubernetes object level, you need to inspect the application itself:

- Has the process actually started?
- Is the service port really listening?
- Are dependent services accessible?
- Is the configuration syntax correct?
- Does the application itself produce any exceptions?

---

## Section XI: Recommended Learning Path for Further Study

It is suggested to progress along the path from "simple to complex, from stateless to stateful, and from successful deployment to troubleshooting."

### Phase 1: Basics of Images and Containers
Recommended learning topics:

- Basic concepts of images, containers, and repositories
- Fundamentals of Dockerfile
- CMD and ENTRYPOINT
- Environment variables
- Log output standards
- Image building and pushing

### Phase 2: Stateless Application Deployment
Recommended learning topics:

- Deployment
- Service
- Ingress
- ConfigMap
- Secret
- Basics of Probes
- Nginx static page deployment
- Simple web service deployment

### Phase 3: Stateful Application Deployment
Recommended learning topics:

- StatefulSet
- Headless Service
- PVC
- StorageClass
- Deployment strategies for components like Redis, MySQL, MinIO, Nacos

### Phase 4: Application Deployment Troubleshooting
Recommended learning topics:

- Issues with image pulling
- Container startup failures
- Probe failures
- Service unreachability
- Ingress 404/502 errors
- DNS resolution problems
- PVC Pending issues
- Configuration mount failures

### Phase 5: Standardized Deployment Practices
Recommended learning topics:

- Helm
- values.yaml files
- Image tag management
- Release and update strategies
- Rollback procedures
- Integration with CI/CD pipelines

---

## Section XII: Common Extended Interview Questions

After completing this main learning path, you will likely encounter many interview questions.

Common questions include:

- How is a business system transformed into a container image?
- What is the difference between Deployment and StatefulSet in Kubernetes?
- How are configurations injected into applications in Kubernetes?
- How to troubleshoot issues where Service is functioning but the business is not?
- What are the steps for investigating Ingress 404/502 errors?
- Why is it recommended to output container logs to stdout/stderr?
- From which perspectives should you check if a Pod keeps restarting?
- How are stateful middleware deployed in Kubernetes?
- What scenarios are ConfigMap and Secret suitable for respectively?
- What problems can arise from improperly configured probes?

---

## Section XIII: Recommended Follow-up Documentation for This Topic

It is suggested to continue writing the following documents after this overview:

1. Comprehensive Overview of Images and Containers
2. Basics of Dockerfile and Image Building Process
3. Fundamentals of Stateless Application Deployment
4. Interaction between Deployment, Service, and Ingress
5. Application Configuration and Secret Management: ConfigMap vs Secret
6. Application Health Check Mechanisms: liveness, readiness, startupProbe
7. Stateful