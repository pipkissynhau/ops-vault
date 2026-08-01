# 15-MySQL Master-Slave Replication Deployment Practice: Helm-Based Kubernetes Deployment, Verification, and Version Switching Supplement

## Document Notes
- Document Focus: Basic deployment practice of MySQL master-slave replication in Kubernetes
- Applicable Stage: After completing StatefulSet, PVC, Service, and single-instance MySQL deployment foundations, proceed to Helm-based master-slave replication deployment practice
- Objective: Be able to deploy MySQL master-slave replication step-by-step, complete basic verification, fundamental troubleshooting, and version switching experiments
- Notes: This document focuses on "practical experimentation", not equivalent to a complete production environment solution, but will organize parameters and verification processes as close to real delivery as possible
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/15-MySQL Main deployment from copy: based on Helm Yes. Kubernetes Deployment, validation and version switching supplement.md`

## Tags
#Kubernetes #MySQL #MainFromCopy #Helm #Bitnami #StatefulSet #PVC #Service #ApplyWithStatus #DatabaseDeployment #ApplyDeployment #Clouds. #Transport

---

## I. Why Use Helm for MySQL Master-Slave Deployment

In previous content, we've already learned:

- Why stateful applications need to focus on identity, storage, service discovery, and startup order
- How to implement MySQL single-instance in Kubernetes through StatefulSet, PVC, ConfigMap, Secret, Service
- When to use handwritten YAML, Helm, and Operator delivery methods

Continuing forward at this stage would be unsuitable for manually assembling all "master node + slave node + Secret + Service + persistent volume + initialization logic" components.

The reason is straightforward:

- Master-slave replication is more complex than single-instance
- More objects involved
- More parameters required
- Prone to pitfalls in initialization, passwords, replication accounts, and role division
- Current stage's goal is no longer just "understanding object relationships", but "first deploying and verifying master-slave replication synchronization"

Therefore, this document prioritizes using Helm Chart rather than manually writing all YAML.

The key focus of this approach isn't to skip fundamentals, but:

> To use a delivery method closer to actual implementation, focusing attention on "architecture, parameters, verification, and troubleshooting".

---

## II. Why Choose Bitnami MySQL Chart Here

The MySQL Helm Chart used in this document is from Bitnami.

The main reasons for this choice are:

### 1) It natively supports replication architecture
Bitnami's official MySQL Chart provides master-slave replication architecture capabilities, allowing parameter-based switching to replication mode rather than only supporting single-instance.

### 2) It has already encapsulated many foundational objects
For example:

- StatefulSet
- Service
- Secret
- Persistent volume claims
- Probes
- Initialization parameters

These are not unimportant, but are already standardized in the Chart.

### 3) It aligns more closely with standardized delivery methods in real environments
In actual work, many middleware solutions aren't written from scratch every time, but rather:

- Use existing Charts
- Override values
- Install, upgrade, rollback
- Adjust parameters based on environment

Therefore, this document's core learning objective isn't "manually creating all resources", but:

> To understand how Helm Charts organize a set of resources and why overriding default behaviors through values files is necessary.

---

## III. First Understand: What Is the Relationship Between Helm Repository, Chart, and values.yaml

Many people when first using Helm to deploy databases tend to stop at:

- `helm repo add`
- `helm install`
- Deployment completed

But if only stopping at this level, parameter adjustments later can easily become confusing.

So let's clarify these concepts first.

### 1) What Is a Helm Repository
A Helm repository can be understood as:

> A place storing many Chart packages.

For example:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Essentially, it's adding the Bitnami Chart repository to the local Helm client and synchronizing the index.

---

### 2) What Is a Chart
A Chart can be understood as:

> A set of Kubernetes resource packages that are templated and parameterized.

For example `bitnami/mysql` this Chart, essentially it's a set of templates that will eventually render:

- StatefulSet
- Service
- Secret
- PVC
- Other auxiliary resources

In other words, a Chart isn't a single YAML file, but a "template package that can generate resources based on parameters".

---

### 3) What Is values.yaml
`values.yaml` can be understood as:

> The default parameter configuration file for this template.

Chart authors will first provide a set of default values.  
You can deploy without passing parameters.  
But default values are usually only suitable for "general scenarios" or "quick experience", not necessarily suitable for current experimental goals.

---

### 4) Why Can't We Directly Use Default values for Deployment
Because default values, while enabling Chart startup, often have these issues:

- Architecture may not be the desired master-slave mode
- Replica count may not meet experimental needs
- Passwords may not be the ones you want to explicitly manage
- Resource limits may not suit the current cluster
- Storage size may not be appropriate
- Some behaviors you may want to explicitly define rather than relying on default values

Therefore, this document requires writing a `values-replication.yaml`, which isn't "overkill", but:

> Clearly tells Helm: This deployment isn't the default MySQL, but a MySQL master-slave replication environment I can understand, control, and verify.

---

## IV. Three Typical Issues Encountered in This Experiment

This document supplements records of three typical issues encountered in practical operations.

### 1) Chart Installation Success Doesn't Guarantee Pod Startup Success
In experiments, after `helm install` succeeds, Pods may still fail to start due to image pull failures.  
That is:

- Helm installation success: Indicates the Chart template has successfully rendered and submitted to Kubernetes
- Pod startup success: Still depends on image pullability, PVC binding, and probe success factors

### 2) Viewing passwords, it's recommended to check Secret rather than the entire YAML or logs
In experiments, the most reliable method isn't checking logs, but:

- First check what you've written in the values file
- Then use `kubectl get secret ... -o jsonpath=... | base64 -d` to precisely extract the currently effective password

### 3) Database major version switching cannot directly reuse old PVC
In experiments, after running a newer version successfully, attempting to switch to 8.0 on the original PVC resulted in:

- `0/1 Running`
- Reboots
- Incompatible data directory with version

This indicates:

> For stateful applications like databases, switching between major versions cannot simply reuse old data volumes.

A more stable approach is:

- New namespace
- New release
- New PVC

---

## V. What Is the Experimental Objective of This Document

This document's practical objective isn't to pursue "complete production high availability", but to first complete the following main workflow:

1. Add Bitnami Helm Repository
2. View Default Information for MySQL Chart
3. Customize a `values-replication.yaml`
4. Deploy MySQL Master-Slave Replication Based on This values File
5. Confirm Pod, PVC, and Service Status
6. Check Current Password
7. Write Test Data to the Master Database
8. Verify Replication Results in the Slave Database
9. Check Replication Status
10. Further Supplement: How to Specify Installation of 8.0 Version

The most important part of this document is:

> First, deploy the master-slave replication and truly verify that the master-slave chain is established.

---

## Six. Prerequisites

Before starting, it's recommended to meet the following conditions:

- You have an available Kubernetes cluster
- Helm is installed
- The cluster has a default StorageClass, or you know exactly which StorageClass to use
- At least basic PVC dynamic provisioning capability
- Access to Helm repository and image repository, or you've already prepared the relevant chart/images in advance

It's recommended to perform the following checks first.

### 1) Check Node Status

```bash
kubectl get nodes
```

Expected Results:

- All node statuses are `Ready`

---

### 2) Check Default StorageClass

```bash
kubectl get storageclass
```

Expected Results:

- At least one available StorageClass exists
- It's better to have a default StorageClass (usually with name ending in `(default)`)

If there's no available StorageClass, the PVCs in the following sections of this document may likely always `Pending`.

---

### 3) Check Helm Version

```bash
helm version
```

---

## Seven. Directory Structure Recommendation

To make subsequent management easier, it's recommended to create a separate directory to save the content of this experiment.

```text
mysql-replication-helm/
├── values-replication.yaml
├── values-replication-80.yaml
├── 01-install.txt
├── 02-verify.txt
└── 03-cleanup.txt
```

Notes:

- `values-replication.yaml`: Default version master-slave experiment parameters
- `values-replication-80.yaml`: 8.0 independent experiment parameters
- Other files are not mandatory, but they're used to separate installation, verification, and cleanup commands

---

## Eight. Step 1: Add Helm Repository and Update Index

First, add the Bitnami repository to your local Helm.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Notes:

- `helm repo add`: Add repository
- `helm repo update`: Update local index

You can continue to execute:

```bash
helm repo list
```

Confirm whether the repository already exists.

---

## Nine. Step 2: View MySQL Chart Basic Information

Before actual installation, it's recommended to first check the basic information of this Chart.

```bash
helm show chart bitnami/mysql
```

You can continue to view the default values:

```bash
helm show values bitnami/mysql
```

This command is very important.

Its purpose isn't just "to see what's happening," but to help build awareness:

> What parameters does the Chart actually provide by default, and the `values-replication.yaml` you write later is essentially selecting key items from these default parameters for override.

It's recommended to at least observe the following types of parameters:

- `architecture`
- `image`
- `auth`
- `primary`
- `secondary`
- `persistence`
- `resources`

---

## Ten. Step 3: Create Namespace

It's recommended to create a separate namespace, don't directly place it in `default`.

```bash
kubectl create namespace mysql-replication
```

Check:

```bash
kubectl get ns
```

---

## Eleven. Step 4: Write values-replication.yaml

Below is an example of a `values-replication.yaml` suitable for the current learning stage.

The goal of this configuration is:

- Use replication architecture
- Deploy 1 master + 1 slave
- Enable persistence for both master and slave
- Explicitly set root password and replication account
- Set basic resource requests and limits for master and slave

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

## Twelve. What This values File Controls

This section is very critical; don't just copy the configuration blindly.

### 1) `architecture: replication`
This item indicates:

- This installation is not standalone
- It's a MySQL replication architecture

In other words, it tells the Chart:

> I want a master-slave replication deployment, not a single instance.

---

### 2) `image`
This section is used to explicitly override the image address.

In this experiment, we explicitly write:

- `repository: bitnamilegacy/mysql`
- `tag: 9.4.0-debian-12-r1`

Because in actual testing, using the Chart's default public image directly resulted in:

- `ErrImagePull`
- `ImagePullBackOff`
- `docker.io/bitnami/mysql:9.4.0-debian-12-r1: not found`

That is, in the current experimental environment:

> If you don't explicitly override the image address, Helm installation may succeed, but the Pod might not be able to pull the image normally.

Therefore, explicitly writing the image address in the values file can avoid fully relying on default values for image behavior.

---

### 3) `auth`
This section defines authentication and replication-related accounts:

- `rootPassword`: root password
- `replicationUser`: replication user
- `replicationPassword`: replication user password

The benefit of explicitly writing these is:

- You know which account the master-slave replication depends on
- You can clearly know the password when verifying later
- Avoid relying entirely on random generation or default behavior

---

### 4) `primary`
This section defines parameters for the master database.

Here, it mainly controls:

- Whether persistence is enabled for the master
- Master volume size
- Resource requests and limits for the master

---

### 5) `secondary`
This section defines parameters for the slave database.

The most important thing here is:

- `replicaCount: 1`

This indicates deploying one slave first.  
If you want to expand to two slaves later, you can adjust this value.

---

## Thirteen. Step 5: Install MySQL Master-Slave Replication

After confirming that `values-replication.yaml` is written, start the installation.

```bash
helm install mysql-repl bitnami/mysql \
  -n mysql-replication \
  -f values-replication.yaml
```

Notes:

- `mysql-repl`: The name of this release
- `bitnami/mysql`: The Chart used
- `-n mysql-replication`: Install to the specified namespace
- `-f values-replication.yaml`: Use the custom parameters file

If you need to re-override parameters later, you can use `helm upgrade`.

---

## Fourteen. Step 6: Check Installation Results

### 1) Check Helm Release

```bash
helm list -n mysql-replication
```

Notes:

- `helm list` contains `APP VERSION`, which is the Chart's built-in appVersion display
- It may not equal the actual container image version you end up with

In other words:

> If you later manually change image.tag to 8.0, `helm list` may still display the Chart's default appVersion information.

Therefore, to confirm the actual running version, a more stable way is:

- Check the actual image in the Pod
- Or enter MySQL and execute `SELECT VERSION();`

---

### 2) Check Pod

```bash
kubectl get pods -n mysql-replication -o wide
```

Expected Phenomenon:

- Should be able to see the master database Pod
- Should be able to see the replica database Pod
- The status should eventually become `Running`
- The Ready column should be `1/1`

---

### 3) Check PVC

```bash
kubectl get pvc -n mysql-replication
```

Expected Phenomenon:

- Master database PVC is `Bound`
- Replica database PVC is `Bound`

If the PVC remains `Pending`, prioritize checking the StorageClass.

---

### 4) Check Service

```bash
kubectl get svc -n mysql-replication
```

You will typically see Services related to the master and replica databases.  
The specific names may vary with Chart versions, but they can generally be inferred from the release name.

---

## FifteenI don't know.Step 7: Check the Generated Secret and View the Root Password

Often you want to confirm whether the Chart has created the password as a Secret.

First list the Secrets:

```bash
kubectl get secret -n mysql-replication
```

A more recommended way to view the root password is:

```bash
kubectl get secret mysql-repl -n mysql-replication -o jsonpath='{.data.mysql-root-password}' | base64 -d; echo
```

Check the replication password:

```bash
kubectl get secret mysql-repl -n mysql-replication -o jsonpath='{.data.mysql-replication-password}' | base64 -d; echo
```

Check the password for regular business users:

```bash
kubectl get secret mysql-repl -n mysql-replication -o jsonpath='{.data.mysql-password}' | base64 -d; echo
```

### Why This Method is Recommended
Because this method has several advantages:

- Directly read the actual effective passwords in the current cluster
- No need to review the entire YAML
- More stable than checking logs
- Can be reused when checking other Secrets in the future

If it's just an experimental environment, you can also directly use the root password defined in the values file.  
But from an operations perspective, **Secret + jsonpath + base64 -d** is the more recommended method.

---

## SixteenI don't know.Step 8: Check Pod Logs to Confirm Normal Master-Slave Initialization

### 1) Check Master Database Logs

```bash
kubectl logs -n mysql-replication <primary-pod-name>
```

### 2) Check Replica Database Logs

```bash
kubectl logs -n mysql-replication <secondary-pod-name>
```

You should focus on:

- Whether there are obvious initialization failures
- Whether there are authentication errors
- Whether there are errors related to replication users
- Whether there are errors related to volume mounting
- Whether there are errors in pulling the image

If you're unsure about the Pod name, you can first check:

```bash
kubectl get pods -n mysql-replication
```

---

## SeventeenI don't know.Step 9: Enter the Master Database to Write Test Data

Now start the actual master-slave verification.

### 1) First Start a Temporary MySQL Client Pod

In the current experimental environment, it's recommended that the client image source matches the server image source.

```bash
kubectl run mysql-repl-client \
  -n mysql-replication \
  --rm -it \
  --restart='Never' \
  --image docker.io/bitnamilegacy/mysql:9.4.0-debian-12-r1 \
  --command -- bash
```

---

### 2) Connect to the Master Database

First confirm the master database Service name:

```bash
kubectl get svc -n mysql-replication
```

The master database Service name is typically similar to:

- `mysql-repl-primary`

Then the connection method can be similar to:

```bash
mysql -h mysql-repl-primary.mysql-replication.svc.cluster.local -uroot -p
```

Enter the root password set in the values file earlier.

---

### 3) Create Test Data in the Master Database

After successful connection, execute:

```sql
CREATE DATABASE repl_test;
USE repl_test;
CREATE TABLE t1 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50)
);
INSERT INTO t1 (name) VALUES ('test_from_primary');
SELECT * FROM t1;
```

If the result is normal, it indicates that the master database write was successful.

---

## EighteenI don't know.Step 10: Enter the Replica Database to Verify Replication Results

After exiting the current MySQL session, connect to the replica database.

Similarly, first confirm the replica database Service name:

```bash
kubectl get svc -n mysql-replication
```

If the replica database Service name is similar to:

- `mysql-repl-secondary`

Then you can try connecting:

```bash
mysql -h mysql-repl-secondary.mysql-replication.svc.cluster.local -uroot -p
```

After connecting to the replica database, execute:

```sql
SHOW DATABASES;
USE repl_test;
SELECT * FROM t1;
```

If you can find:

- Database `repl_test`
- Table `t1`
- Record `test_from_primary`

in the replica database, it indicates that the master-slave replication has basically been established.

---

## NineteenI don't know.Step 11: Further Check Replication Status

If you want to more clearly observe the replication status, execute in the replica database:

```sql
SHOW REPLICA STATUS\G
```

### Note
In the current experiment, under MySQL 9.4 / 8.0 new semantics:

- The old `SHOW SLAVE STATUS\G` is no longer recommended
- `SHOW REPLICA STATUS\G` is more recommended

Focus on these fields:

- `Replica_IO_Running: Yes`
- `Replica_SQL_Running: Yes`
- `Seconds_Behind_Source: 0`
- `Last_IO_Errno: 0`
- `Last_SQL_Errno: 0`

If these fields are normal, it indicates:

- The replica database I/O thread is normal
- The replica database SQL thread is normal
- There is no obvious replication delay currently
- No I/O errors
- No SQL execution errors

---

## TwentyI don't know.How to Specify Installation of 8.0 Version

This section is an additional focus point added based on practical operations in this article.

### 1) Core Idea for Specifying the Version
If you want to install 8.0, the core is not changing the Helm chart version, but:

- Override `image.repository`
- Override `image.tag`

In other words:

> Still use the same Chart, but let it pull different versions of MySQL images.

For example:

File name: `values-replication-80.yaml`

```yaml
architecture: replication

image:
  registry: docker.io
  repository: bitnamilegacy/mysql
  tag: 8.0.40-debian-12-r4

auth:
  rootPassword: "RootPass80!"
  replicationUser: "repl_user"
  replicationPassword: "ReplPass80!"

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

### 2) Why It's Not Recommended to Switch to 8.0 Directly on the Original 9.4 PVC
In actual experiments, directly using the PVC that originally ran 9.4 for 8.0 will result in:

- Pods start but `0/1 Running`
- Container restarts
- Readiness probe fails
- Essentially, incompatibility between data directory and major version

Therefore, a more stable approach is:

- New namespace
- New release
- New PVC

For example:

```bash
kubectl create namespace mysql-replication-80

helm install mysql-repl-80 bitnami/mysql \
  -n mysql-replication-80 \
  -f values-replication-80.yaml
```

---

### 3) How to Confirm Whether the Current Instance is Running 8.0
Don't just check:

```bash
helm list -n mysql-replication-80
```

A more stable method is:

#### Check the Actual Image of the Pod
```bash
kubectl get pod mysql-repl-80-primary-0 -n mysql-replication-80 -o jsonpath='{.spec.containers[0].image}'; echo
kubectl get pod mysql-repl-80-secondary-0 -n mysql-replication-80 -o jsonpath='{.spec.containers[0].image}'; echo
```

#### Or Enter the Database to Directly Check the Version
First start the client:

```bash
kubectl run mysql-repl-80-client \
  --rm --tty -i \
  --restart='Never' \
  --image docker.io/bitnamilegacy/mysql:8.0.40-debian-12-r4 \
  --namespace mysql-replication-80 \
  --command -- bash
```

Connect to the master database:

```bash
mysql -h mysql-repl-80-primary.mysql-replication-80.svc.cluster.local -uroot -p'RootPass80!'
```

Query the version:

```sql
SELECT VERSION();
```

If it returns `8.0.40`, it indicates that the current instance is running 8.0.

---

### 4) How to Verify 8.0 Master-Slave Replication
In the 8.0 master database, execute:

```sql
CREATE DATABASE repl_test_80;
USE repl_test_80;
CREATE TABLE t1 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50)
);
INSERT INTO t1 (name) VALUES ('test_from_primary_80');
SELECT * FROM t1;
```

Then connect to the 8.0 replica database and execute:

```sql
SHOW DATABASES;
USE repl_test_80;
SELECT * FROM t1;
SHOW REPLICA STATUS\G
```

If the replica database can find:

- `repl_test_80`
- `test_from_primary_80`

And `SHOW REPLICA STATUS\G` is normal, it indicates that the 8.0 master-slave replication has been successfully established.

---

## Twenty-OneI don't know.Why This Article Doesn't Dive Deeper into Replication Parameters

Because the core goal at this stage is:

> First deploy and verify the master-slave replication.

Rather than immediately entering:

- GTID deep configuration
- Semi-synchronous replication
- Read-only control strategy
- Delayed replication
- Multi-level replication
- Read-write separation proxy
- Cross-availability zone replication governance
- Database version upgrade strategy details

These contents are indeed more aligned with production environments, but if we pack them all in now, it would disrupt the main thread.

Therefore, this article first focuses on:

- Chart understanding
- Values override
- Helm installation
- Master-slave verification
- Password checking
- Specifying version installation
- Basic troubleshooting

---

## Twenty-TwoI don't know.Common Issue Troubleshooting

### 1) Pod Stays in Pending State
Prioritize checking:

```bash
kubectl get pvc -n mysql-replication
kubectl get storageclass
kubectl describe pvc -n mysql-replication <pvc-name>
```

Common causes:

- No default StorageClass
- Dynamic provisioner anomaly
- Storage capacity insufficient

--- /think

### 2I'm not sure.Pod Stuck in CrashLoopBackOff or Running but `0/1`
Priority Checks:

```bash
kubectl logs -n mysql-replication <pod-name>
kubectl describe pod -n mysql-replication <pod-name>
```

Common Causes:

- Abnormal password parameters
- Storage mount anomalies
- Insufficient resources
- Incompatibility between old data volumes and new versions
- Probe failure

---

### 3I'm not sure.Image Not Found / `ErrImagePull` / `ImagePullBackOff`
This is a real issue we've encountered in current experiments.

Typical Symptoms:

- Helm installation succeeds
- Pod fails to start
- In `kubectl describe pod` you see:
  - `ErrImagePull`
  - `ImagePullBackOff`
  - `not found`

In such cases, prioritize checking:

```bash
kubectl describe pod -n mysql-replication <pod-name>
```

If the default image cannot be pulled, explicitly override in values file:

- `image.repository`
- `image.tag`

Don't only check whether `helm install` succeeds.

---

### 4I'm not sure.Slave Not Syncing Data from Master
Priority Checks:

- Whether both master and slave are Ready
- Whether master writes are truly successful
- Whether the slave is connected to the correct secondary Service
- Whether replication user and password in values file are correct
- Whether `SHOW REPLICA STATUS\G` is normal
- Whether Pod logs show replication initialization errors

---

### 5I'm not sure.Service Name Mismatch
Don't memorize names blindly.  
Directly check:

```bash
kubectl get svc -n mysql-replication
```

Helm Chart rendered resource names relate to release name and Chart logic.  
Thus, the safest approach is not to guess, but to use `kubectl get svc` to see actual names.

---

### 6I'm not sure.Unknown Parameters in Chart Default Values
Directly check:

```bash
helm show values bitnami/mysql
```

This is more reliable than searching for second-hand blog posts.

---

## 23. Step 12: Clean Up Environment

After experiment completion, if cleanup is needed, execute:

```bash
helm uninstall mysql-repl -n mysql-replication
```

If deleting namespace is also required:

```bash
kubectl delete namespace mysql-replication
```

If cleaning up 8.0 independent experiment:

```bash
helm uninstall mysql-repl-80 -n mysql-replication-80
kubectl delete namespace mysql-replication-80
```

Notes:

- Whether PVCs are retained after release deletion depends on actual resource recycling policies
- If in experimental environment, confirm whether you still need to preserve test data before deletion

---

## 24. What Did We Truly Learn from This Article

After completing this article, the most important thing is not "remembering a helm install command", but understanding these concepts:

### 1I'm not sure.MySQL Master-Slave Replication Doesn't Necessarily Require Manual YAML
In more realistic delivery scenarios, it can be deployed via Helm Chart.

### 2I'm not sure.values.yaml is the Core Control Entry for Helm Deployments
You're not modifying the Chart from scratch, but overriding its default behavior.

### 3I'm not sure.When Deploying Master-Slave, Focus Isn't Only on Pod Startup
Also check:

- PVC status
- Service status
- Master write capability
- Slave read replication results
- Whether `SHOW REPLICA STATUS\G` is healthy

### 4I'm not sure.Recommended Way to View Passwords is Secret + jsonpath
Avoid checking entire YAML files every time, and don't prioritize relying on logs.

### 5I'm not sure.Different Versions Can Be Installed by Overriding image.repository / image.tag
For example 9.4, 8.0, etc., but database major version switching cannot directly reuse old PVCs.

### 6I'm not sure.For Stateful Applications Like Databases, Version Switching Isn't Just Image Replacement
Must first consider:

- Whether old data directories can continue to be used
- Whether new PVC is needed
- Whether a new release should be created for isolated experimentation

---

## 25. Relationship Between This Article and the Next Article 16

This article solves:

> How to deploy MySQL master-slave replication using Helm, and how to view passwords, specify versions, and avoid image and old PVC pitfalls.

The next article 16 is more suitable for:

> How to deploy a more high-availability governed MySQL cluster using Operator.

In other words, this article is "Master-Slave Replication Delivery Practice", while the next article is "High-Availability Cluster Governance Introduction".

---

## 26. Supplementary Notes: This Article Is Closer to Production but Not a Complete Production Solution

Need to clarify:

- This article is already closer to real environments than hand-written single instances
- But it's still primarily used for experiments, learning, and establishing deployment understanding of master-slave replication
- True production requires further supplementation:
  - Monitoring and alerts
  - Backup and recovery
  - Security and permission control
  - Resource capacity planning
  - Node affinity / anti-affinity
  - Upgrade and rollback strategies
  - Failover strategies
  - Read-write separation access methods
  - Image and chart internal repositoryization

Thus, the most accurate positioning is:

> First deploy MySQL master-slave replication in a way closer to real delivery, and understand the role of Helm values, password viewing methods, version specification methods, and major version switching considerations.

---

## Official Reference Links
- Bitnami MySQL Helm Chart README  
  https://github.com/bitnami/charts/blob/main/bitnami/mysql/README.md

- Bitnami MySQL Chart Default Values  
  https://github.com/bitnami/charts/blob/main/bitnami/mysql/values.yaml

- Helm Official Documentation  
  https://helm.sh/docs/

- Helm `helm repo add` Documentation  
  https://helm.sh/docs/helm/helm_repo_add/

- Kubernetes Persistent Volumes Official Documentation  
  https://kubernetes.io/docs/concepts/storage/persistent-volumes/

- Kubernetes StorageClass Official Documentation  
  https://kubernetes.io/docs/concepts/storage/storage-classes/