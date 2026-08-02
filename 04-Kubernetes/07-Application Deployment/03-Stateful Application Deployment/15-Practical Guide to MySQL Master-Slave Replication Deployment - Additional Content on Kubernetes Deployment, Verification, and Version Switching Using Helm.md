# 15-Practical Guide to MySQL Master-Slave Replication Deployment: Additional Content on Kubernetes Deployment, Verification, and Version Switching Using Helm

## Document Description
- Document Purpose: Practical guidance on basic MySQL master-slave replication deployment in Kubernetes using Helm.
- Applicable Phase: After completing the basic deployment of StatefulSet, PVC, Service, and a single MySQL instance, this guide helps you practice master-slave replication deployment with Helm.
- Goal: To enable you to deploy MySQL master-slave replication step by step and complete basic verification, troubleshooting, and version switching experiments.
- Note: This guide focuses on "implementable experiments" rather than a complete production environment solution. However, we will try to organize parameters and verification processes in a way that resembles real-world delivery scenarios as much as possible.
- Recommended Reading Path: `04-Kubernetes/07-Application Deployment/03-Stateful Application Deployment/15-Practical Guide to MySQL Master-Slave Replication Deployment: Additional Content on Kubernetes Deployment, Verification, and Version Switching Using Helm.md`

## Tags
#Kubernetes #MySQL #Master-Slave Replication #Helm #Bitnami #StatefulSet #PVC #Service #Stateful Application #Database Deployment #Application Deployment #Cloud-Native #Operation and Maintenance

---

## I. Why Use Helm for MySQL Master-Slave Deployment

In previous sections, we have learned:

- Why stateful applications require attention to identity management, storage, service discovery, and startup order.
- How to deploy a single MySQL instance in Kubernetes using StatefulSet, PVC, ConfigMap, Secret, and Service.
- Which scenarios are suitable for each of the three delivery methods: manual YAML writing, Helm, and Operators.

Moving forward, it would be impractical to manually assemble all components such as "master database + slave database + Secret + Service + persistent volumes + initialization logic" again.

The reasons are straightforward:

- Master-slave replication is more complex than a single instance.
- There are more objects involved.
- More parameters need to be configured.
- It's easier to encounter issues during initialization, password setting, replication account configuration, and role assignment.
- At this stage, our goal is no longer just to understand the object relationships but to actually deploy master-slave replication and verify whether synchronization works properly.

Therefore, in this guide, we will prioritize using Helm Charts instead of writing all YAML code manually.

The key advantage of this approach is not to skip the basics but to:

> Focus on "architecture, parameters, verification, and troubleshooting" in a way that resembles real-world delivery scenarios.

---

## II. Why Choose the Bitnami MySQL Chart

In this guide, we will use the Bitnami MySQL Helm Chart.

There are several main reasons for this choice:

### 1) It Supports the Replication Architecture
The official Bitnami MySQL Chart provides support for the replication architecture. You can switch to the replication mode by adjusting parameters, rather than only supporting a single instance.

### 2) It Encapsulates Many Basic Components
For example:

- StatefulSet
- Service
- Secret
- Application for persistent volumes
- Probes
- Initialization parameters

These components are not insignificant but have already been standardized within the Chart.

### 3) It Aligns with Real-World Standardized Delivery Practices
In actual work, many middleware components are not manually configured using YAML from scratch every time. Instead, we usually:

- Use existing Charts.
- Override default values.
- Perform installation, updates, and rollbacks.
- Adjust parameters according to the environment.

Therefore, the core learning objective of this guide is not to "manually create all resources" but to:

> Understand how Helm Charts organize a set of resources and why we need to use a `values` file to override default behaviors.

---

## III. First, Understand: What Are the Relationships Between Helm Repositories, Charts, and values.yaml Files

Many people tend to stop at the following steps when using Helm to deploy databases for the first time:

- `helm repo add`
- `helm install`
- Deployment completed.

However, if you only stay at this level, it will be easy to get confused when you need to change parameters later.

So let's clarify these concepts first.

### 1) What Is a Helm Repository
A Helm repository can be understood as:

> A place where many Chart packages are stored.

For example:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Essentially, this command adds the Bitnami Chart repository to your local Helm client and synchronizes its index.

---

### 2) What Is a Chart
A Chart can be understood as:

> A set of Kubernetes resource packages that have been templated and parameterized.

For example, the `bitnami/mysql` Chart essentially contains a set of templates that will eventually generate:

- StatefulSet
- Service
## Step 2: Checking Basic Information of the MySQL Chart

Before actually installing it, it is recommended to first check the basic information of this chart.

```bash
helm show chart bitnami/mysql
```

You can also view the default values:

```bash
helm show values bitnami/mysql
```

This command is very important.

Its purpose is not just to “look around”, but to help you understand:

> What parameters are provided by default in the chart. The `values-replication.yaml` file you will create later will actually select key items from these default parameters for customization.

It is recommended to observe at least the following categories of parameters:

- `architecture`
- `image`
- `auth`
- `primary`
- `secondary`
- `persistence`
- `resources`

---

## Step 3: Creating a Namespace

It is suggested to create a separate namespace instead of using the `default` one.

```bash
kubectl create namespace mysql-replication
```

Check if it was created successfully:

```bash
kubectl get ns
```

---

## Step 4: Writing values-replication.yaml

Below is an example of `values-replication.yaml` suitable for this learning phase.

The goal of this configuration is:

- To use the replication architecture
- To deploy 1 primary database and 1 secondary database
- To enable persistence for both the primary and secondary databases
- To explicitly set the root password and the replication user account
- To define basic resource requests and limits for both the primary and secondary databases

File name: `values-replication.yaml`

```yaml
architecture: replication

image:
  registry: docker.io
  repository: bitnamilegacy/mysql
  tag: 9.4.0-debian-12-r1

auth:
  rootPassword: "RootPass123!"
  replicationUser: "repl_user"
  replicationPassword: "ReplPass123!"

primary:
  persistence:
    enabled: true
    size: 8Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi

secondary:
  replicaCount: 1
  persistence:
    enabled: true
    size: 8Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
```

---

## Step 12: What This values File Controls

This section is crucial; you shouldn’t just copy the configuration without understanding it.

### 1) `architecture: replication`
This setting indicates:

- That this installation is not a standalone instance, but rather a MySQL replication architecture.

In other words, it tells the chart:

> I want to set up a primary-secondary replication deployment, not a single-instance setup.

---

### 2) `image`
This section is used to explicitly override the image address.

In this experiment, we are specifying:

- `repository: bitnamilegacy/mysql`
- `tag: 9.4.0-debian-12-r1`

Because in actual testing, using the default public image provided by the chart sometimes resulted in errors such as:

- `ErrImagePull`
- `ImagePullBackOff`
- `docker.io/bitnami/mysql:9.4.0-debian-12-r1: not found`

This means that in our current experimental environment:

> If we don’t explicitly specify the image address, Helm may successfully install it, but the Pod might not be able to pull the image correctly.

Therefore, by specifying the image address in the `values` file, we can ensure that the image is pulled correctly and avoid potential issues.

---

### 3) `auth`
This section defines the authentication and replication-related accounts:

- `rootPassword`: The root password
- `replicationUser`: The replication user account
- `replicationPassword`: The password for the replication user account

The benefits of specifying these values explicitly are:

- You know which account is used for primary-secondary replication.
- You can easily verify the passwords when needed.
- It avoids relying entirely on randomly generated or default settings.

---

### 4) `primary`
This section defines the parameters for the primary database.

It mainly controls:

- Whether persistence is enabled for the primary database.
- The size of the volume used by the primary database.
- The resource requests and limits for the primary database.

---

### 5) `secondary`
This section defines the parameters for the secondary database.

The most important setting here is:

- `replicaCount: 1`

This indicates that only one secondary database will be deployed initially. If you later want to add more secondary databases, you can adjust this value accordingly.

---

## Step 5: Installing MySQL```bash
kubectl get pods -n mysql-replication
```Common causes:

- No default StorageClass set
- Dynamic provider exceptions
- Insufficient storage capacity

---

### 2) Pod keeps crashing in a LoopBackOff state or is running but shows `0/1`
Priority checks:

```bash
kubectl logs -n mysql-replication <pod-name>
kubectl describe pod -n mysql-replication <pod-name>
```

Common causes:

- Abnormal password parameters
- Storage mounting issues
- Insufficient resources
- Incompatibility between old data volumes and new versions
- Probe failures

---

### 3) Image not found / `ErrImagePull` / `ImagePullBackOff`
This is a problem that has actually been encountered in current experiments.

Typical symptoms:

- Helm installation is successful
- Pod fails to start
- When checking with `kubectl describe pod`, the following errors are displayed:
  - `ErrImagePull`
  - `ImagePullBackOff`
  - `Not found`

In this case, priority checks should be:

```bash
kubectl describe pod -n mysql-replication <pod-name>
```

If it is found that the default image cannot be pulled, then the `image.repository` and `image.tag` fields in the values file need to be explicitly overridden.

Do not rely solely on whether `helm install` was successful.

---

### 4) Secondary database has not synchronized data from the primary database
Priority checks:

- Whether both the primary and secondary databases are ready
- Whether writes to the primary database are actually successful
- Whether the connection to the secondary database is through a secondary Service
- Whether the replication user and password in the values file are correct
- Whether `SHOW REPLICA STATUS\G` returns normal results
- Whether there are any error messages related to replication initialization in the Pod logs

---

### 5) The Service name does not match what was expected
Do not memorize the name. Instead, directly check:

```bash
kubectl get svc -n mysql-replication
```

The resource names rendered by the Helm Chart may vary depending on the release name and the logic of the Chart. Therefore, the safest approach is to use `kubectl get svc` to verify the actual name.

---

### 6) Not knowing what other adjustable parameters are available in the Chart's default values
Directly check:

```bash
helm show values bitnami/mysql
```

This is more reliable than searching through various second-hand blogs.

---

## Chapter 23: Step 12: Clearing up the Environment

After completing the experiment, if it is necessary to clean up, you can execute the following commands:

```bash
helm uninstall mysql-rep -n mysql-replication
```

If you also want to delete the namespace, use:

```bash
kubectl delete namespace mysql-replication
```

If you are clearing up an independent experiment using version 8.0, use:

```bash
helm uninstall mysql-repl-80 -n mysql-replication-80
kubectl delete namespace mysql-replication-80
```

Notes:

- Whether to keep the PVC after deleting the release depends on your actual resource recycling strategy.
- If it is an experimental environment, make sure you have confirmed whether you still need to retain the test data before deletion.

---

## Chapter 24: What You Have Really Learned from This Article

The most important thing after completing this article is not to “memorize a single `helm install` command”, but to understand the following concepts:

### 1) MySQL master-slave replication does not necessarily require writing YAML manually
In scenarios that are closer to real-world deployment, it can be set up using Helm Charts.

### 2) `values.yaml` is the core control point for Helm deployments
You are not creating a Chart from scratch; instead, you are overriding the default behavior of the Chart.

### 3) When deploying master-slave replication, the focus is not just on whether the Pod starts successfully
Other aspects that need attention include:

- Whether the PVC is functioning properly
- Whether the Service is working correctly
- Whether the primary database can write data
- Whether the secondary database can read the replicated data
- Whether `SHOW REPLICA STATUS\G` shows healthy status

### 4) The most recommended way to view passwords is through Secrets and jsonpath
There is no need to read the entire YAML file every time, and relying on logs is not recommended either.

### 5) Different versions of databases can be installed by overriding `image.repository` and `image.tag`
For example, versions 9.4 or 8.0, but note that when switching between major database versions, you cannot simply reuse old PVCs.

### 6) For stateful applications like databases, version switching is not just about changing the image
You also need to consider:

- Whether the old data directories can still be used
- Whether new PVCs are required
- Whether a new release