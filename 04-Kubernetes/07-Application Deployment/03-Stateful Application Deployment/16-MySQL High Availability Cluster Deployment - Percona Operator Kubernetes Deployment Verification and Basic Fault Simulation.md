# 16-MySQL High Availability Cluster Deployment: Kubernetes Deployment, Verification, Troubleshooting, and Basic Fault Simulation Using Percona Operator

## Document Overview
- Document Purpose: Basic deployment of MySQL High Availability Cluster in Kubernetes
- Applicable Stage: After completing MySQL single instance and MySQL master-slave replication basics, entering Operator-managed high availability cluster deployment practice
- This Article's Goal: Be able to deploy a MySQL High Availability Cluster based on Percona Operator step by step, complete connection verification, data verification, and basic fault simulation
- Note: More accurately, it's deploying **Percona Operator for MySQL based on Percona XtraDB Cluster (PXC)**
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/16-MySQL High-availability cluster deployment exercises: based on Percona Operator Yes. Kubernetes Deployment, validation, pit-step correction and basic malfunction exercise.md`

## Tags
#Kubernetes #MySQL #HighAvailable #Operator #Percona #PXC #HAProxy #PVC #Secret #ApplyWithStatus #DatabaseDeployment #ApplyDeployment #Clouds. #Transport

---

## I. Why This Article Switches from Using Regular Helm Master-Slave to Operator

The previous article has already set up MySQL master-slave replication.  
But master-slave replication and high availability cluster are not the same level of issues.

Master-slave replication is more focused on:

- Whether replication relationship is established
- Master write, slave read
- Parameterized delivery
- Basic replication verification

High availability cluster is more focused on:

- Cluster members remaining healthy continuously
- Recovery capability after node failure
- Unified proxy layer access
- Lifecycle management
- Subsequent capabilities like backup, recovery, upgrade, scaling, and monitoring

Continuing to use "a Chart for master-slave" approach will make it increasingly difficult to manage complex lifecycle.  
So this article is more suitable for entering:

> **Operator-managed database cluster.**

---

## II. What Exactly Is Being Deployed in This Article

This article is not deploying "ordinary single-machine MySQL" or "traditional asynchronous master-slave".

This article deploys:

> **Percona Operator for MySQL based on Percona XtraDB Cluster (PXC)**

You can initially understand it as:

- Operator: Responsible for managing the database cluster as a Kubernetes native object continuously
- PXC: Responsible for the actual data node cluster
- HAProxy: Provides unified access entry
- PVC: Provides independent persistent volumes for each data node

So the overall model of this article is:

- One Operator
- One PXC data node set
- A group of HAProxy proxy nodes
- Several PVCs
- Unified orchestration by Operator through custom resources

This is no longer the same level of abstraction as "a StatefulSet running single-machine MySQL" from before.

---

## III. What Is the Fundamental Difference Between This Article and Article 15

### Article 15 Focus
- Master-slave replication
- Replication parameters
- Master write, slave read
- Helm values override
- Master-slave status verification

### Article 16 Focus
- Operator
- PXC cluster
- HAProxy entry
- `kubectl get pxc`'s `ready`
- Single-node failure recovery
- Data remains accessible after recovery

So do not simply understand this article as:

> "Switching to another Helm Chart for MySQL"

More accurately, this article is entering:

> **MySQL cluster in Kubernetes under Operator mode.**

---

## IV. First Understand the Complete Flow of This Article

This article ultimately aims to implement the following flow:

1. Add Percona Helm repository
2. Install Percona Operator
3. Check `percona/pxc-db` default values
4. Prepare a suitable `values-pxc.yaml` for current experiment environment
5. Install PXC cluster
6. Check if `kubectl get pxc` is `ready`
7. Get root password
8. Connect to database via HAProxy Service
9. Insert test data
10. Delete one PXC Pod and observe recovery
11. After recovery, reconnect and confirm data remains accessible

The core goal of this article is not to explain all production details at once, but:

> **First deploy a MySQL High Availability Cluster managed by Operator, and complete basic availability and recovery verification.**

---

## V. Prerequisites

Before starting, it's recommended to meet these conditions:

- Available Kubernetes cluster
- Helm v3 available
- At least one available StorageClass
- Ability to pull Percona images
- Cluster resources cannot be too small, especially memory cannot be too conservative
- At least 3 nodes available for scheduling PXC, or allow master node to temporarily participate in scheduling

### Why This Article Requires More Resources Than the Previous One
Percona's default values are not light:

- Default `pxc.size: 3`
- Default `haproxy.size: 3`
- Default also includes TLS, log collection, backup structure reservations

So if the experiment environment is resource-constrained, it's easy to encounter:

- `kubectl get pxc` not being `ready` for a long time
- Pod constantly `Pending`
- Pod `OOMKilled`
- Third PXC node cannot be scheduled

---

## VI. First Do Basic Checks

### 1) Check Node Status

    kubectl get nodes

Requirements:

- Node status is `Ready`

### 2) Check StorageClass

    kubectl get storageclass

Requirements:

- At least one available StorageClass
- Preferably has a default StorageClass
- If you specifically want to use a certain class, like `nfs-client`, explicitly write it in the values later

### 3) Check Helm Version

    helm version

### 4) Observe Node Resources

    kubectl top nodes

Note:

- Memory is very critical in this article
- If a PXC node continuously `OOMKilled`, don't just look at Pod, combine with node resources for judgment

---

## VII. Directory Structure Recommendation

It's recommended to create a separate directory to save the experiment files for this article.

    mysql-ha-percona/
    ├── values-pxc.yaml
    ├── 01-install-operator.txt
    ├── 02-install-cluster.txt
    ├── 03-verify.txt
    └── 04-cleanup.txt

The most critical one is:

- `values-pxc.yaml`

---

## VIII. Step 1: Add Percona Helm Repository

    helm repo add percona https://percona.github.io/percona-helm-charts/
    helm repo update

Check:

    helm repo list

---

## IX. Step 2: Create Namespace

    kubectl create namespace mysql-ha

Check:

    kubectl get ns

---

## X. Step 3: Install Percona Operator

First install the Operator:

    helm install my-op percona/pxc-operator -n mysql-ha

Check:

    kubectl get pods -n mysql-ha

Expected behavior:

- First see Operator-related Pod starting
- Operator Pod eventually enters `Running`

Notes:

- `my-op` is just the name of this Helm release for the Operator
- It is not the name of the database cluster itself

---

## XI. First review default values, don't install blindly

Before writing `values-pxc.yaml`, first review the default values:

    helm show values percona/pxc-db

Focus on:

- `pxc.size`
- `pxc.resources`
- `pxc.persistence`
- `haproxy.enabled`
- `haproxy.size`
- `haproxy.resources`
- `tls.enabled`
- `logcollector.enabled`
- `backup.enabled`
- `pmm.enabled`

### Key understandings from default values
Default values are not "minimal experimental configuration", but rather a more complete form:

- PXC defaults to 3 nodes
- HAProxy defaults to 3 nodes
- TLS defaults to enabled
- logcollector defaults to enabled
- backup defaults to enabled
- pmm defaults to reserved capabilities

Therefore:

> **Do not treat default values as a lightweight experimental template.**

---

## XII. Why this article still requires writing values-pxc.yaml manually

Same as the previous article:

> You are not rewriting the Chart, but overriding default parameters.

The purpose of writing values manually in this article is:

- Fix PXC node scale
- Compress HAProxy scale to an acceptable range for this experiment
- Explicitly set resource requests and limits for PXC and HAProxy
- Explicitly specify StorageClass
- Disable unnecessary components for this stage
- Clearly document the most important parameters for this experiment for easy reuse later

---

## XIII. Step 4: Write values-pxc.yaml (Final Operational Version)

This values file is the final version derived from this experiment's real-world issues:

    pxc:
      size: 3
      certManager: false
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 1000m
          memory: 3Gi
      persistence:
        enabled: true
        size: 8Gi
        storageClass: nfs-client

    haproxy:
      enabled: true
      size: 2
      resources:
        requests:
          cpu: 300m
          memory: 300Mi
        limits:
          cpu: 500m
          memory: 512Mi

    logcollector:
      enabled: false

    pmm:
      enabled: false

    backup:
      enabled: false

### Key understandings of this values file

#### 1) Why `pxc.size` still retains 3
Because this article aims to establish a high-availability cluster awareness.  
PXC data layer is not recommended to be scaled down to 1 or 2 directly.

#### 2) Why `haproxy.size` is set to 2 instead of 1
This is the first pitfall encountered in this hands-on experiment:

- Initially set `haproxy.size` to 1
- Operator immediately reported an error
- `kubectl get pxc` showed `error`
- Logs clearly indicated:
  - `HAProxy size must be at least 2`
  - Unless explicitly set `unsafeFlags.proxySize=true`

Therefore, the more stable experimental value currently is:

> **HAProxy should be set to at least 2.**

#### 3) Why `pxc.resources.limits.memory` is set to 3Gi
This is also a key pitfall encountered in this experiment:

- Initially set `limits.memory` to a relatively small value
- `pxc-1` was scheduled to `k8s-node02`
- Containers repeatedly `OOMKilled`
- Host logs showed `mysqld` being killed by the kernel OOM Killer
- After increasing `node02` memory from 2G to 8G, stability was finally achieved

Therefore, we retain `limits`, but need to provide more generous values, not overly aggressive.

#### 4) Why disable `logcollector`, `pmm`, and `backup`
Because this article's current goal is not:

- Backup specialty
- PMM monitoring specialty
- Log collection specialty

But rather:

- First deploy the cluster
- First complete connectivity and recovery verification

Therefore, disabling these additional components is more stable.

---

## XIV. Step 5: Install Database Cluster

Here define the database cluster release name as `cluster1`:

    helm install cluster1 percona/pxc-db \
      -n mysql-ha \
      -f values-pxc.yaml

Notes:

- `cluster1`: Database cluster release name
- `percona/pxc-db`: Database cluster Chart
- `-f values-pxc.yaml`: Use custom parameter file

---

## XV. Step 6: Observe Cluster Startup Process

### 1) Check Pods

    kubectl get pods -n mysql-ha -o wide

You will gradually see:

- Operator Pod
- PXC data nodes
- HAProxy Pod

### 2) Check PXC Status

    kubectl get pxc -n mysql-ha

This is one of the most critical observation commands in this article.

Expected behavior:

- Initially may show `error`, `initializing`
- Eventually needs to enter `ready`

### 3) Check PVC

kubectl get pvc -n mysql-ha

Expected Outcome:

- All PXC PVCs become `Bound`

---

## Sixteen, Several Critical Pitfalls I Actually Encountered

### Pitfall 1: `haproxy.size: 1` Will Be Blocked by Operator's Default Safety Value
Phenomenon:

- `kubectl get pxc -n mysql-ha` Displays `error`
- No real PXC / HAProxy Pod is created
- `kubectl describe pxc` and operator logs clearly indicate:
  - `HAProxy size must be at least 2`

Conclusion:

> **In the current experimental environment, do not compress `haproxy.size` to 1.**

---

### Pitfall 2: PXC Node `OOMKilled`
Phenomenon:

- `pxc-1` Repeatedly `OOMKilled`
- `kubectl describe pod` Shows `Reason: OOMKilled`
- System logs on the node show `mysqld`, `pxc-entrypoint`, etc. processes being killed by the kernel

Judgment Points:

- Not a Helm issue
- Not a probe itself issue
- Insufficient memory at the host level causing container startup failure

Handling:

- Increase PXC's memory limit
- More importantly, check if the node has sufficient memory
- In this experiment, it was finally stabilized after expanding `node02` from 2G to 8G

---

### Pitfall 3: Third PXC Node Always `Pending`
Phenomenon:

- `pxc-0` and `pxc-1` have already been deployed to two workers
- `pxc-2` keeps `Pending`

In `kubectl describe pod`, you'll see similar information:

- `1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }`
- `2 node(s) didn't match pod anti-affinity rules`

Meaning:

- The master node has `control-plane` taint, Pod cannot be scheduled
- Two workers already have one PXC each
- Default anti-affinity prevents the third PXC from being scheduled on the same host

There are three handling options:

1. Add another schedulable worker node  
2. Temporarily allow master to participate in scheduling  
3. Reduce anti-affinity constraints (not recommended as the first choice currently)

In this experiment, the adopted approach was:

> **Remove the taint from master, allowing `pxc-2` to be scheduled on `k8s-master`.**

---

### Pitfall 4: Temporary Client `kubectl run -it --rm` Is Not Immediate
Phenomenon:

- After executing `kubectl run -it --rm ...`, the terminal will have a period of "no response" waiting
- Actually, the temporary Pod is scheduling, pulling images, starting, and attaching

Handling:

- Wait a moment
- If prompted:
  - `If you don't see a command prompt, try pressing enter.`
  Press Enter
- Don't assume the command is wrong just because there's no response

---

### Pitfall 5: Temporary Client with Same Name Pod Already Exists
Phenomenon:

- `Error from server (AlreadyExists): pods "percona-client" already exists`

Handling:

- Delete the old Pod first
- Or use a different name each time, e.g., `percona-client-2`

---

### Pitfall 6: Deleting Namespace Stuck at `Terminating`
Phenomenon:

- The `mysql-ha` namespace can't be deleted
- Namespace status indicates:
  - `Some resources are remaining: perconaxtradbclusters.pxc.percona.com has 1 resource instances`
  - `Some content in the namespace has finalizers remaining: percona.com/delete-pxc-pods-in-order`

Root Cause:

- The PerconaXtraDBCluster CR still exists
- Its finalizer wasn't properly removed

Handling:

First patch the finalizer on the PXC resource:

    kubectl patch pxc cluster1-pxc-db -n mysql-ha --type=merge -p '{"metadata":{"finalizers":[]}}'

If it's still stuck, consider forcefully clearing the namespace's finalizer.

---

## Seventeen, Step 7: Check Service

Before connecting, check the actual Service name:

    kubectl get svc -n mysql-ha

Focus on observing:

- `cluster1-pxc-db-haproxy`
- `cluster1-pxc-db-haproxy-replicas`
- `cluster1-pxc-db-pxc`
- `cluster1-pxc-db-pxc-unready`

In the current experiment, the main access entry is:

> **cluster1-pxc-db-haproxy**

---

## Eighteen, Step 8: Retrieve Root Password

First list the Secrets:

    kubectl get secrets -n mysql-ha

In the current experiment, the one to focus on is:

- `cluster1-pxc-db-secrets`

Retrieve the root password:

    kubectl get secret cluster1-pxc-db-secrets \
      -n mysql-ha \
      --template='{{.data.root | base64decode}}{{"\n"}}'

Note:

- The result is a random strong password
- The password often contains special characters
- It's recommended to use single quotes when connecting

Example:

    mysql -h cluster1-pxc-db-haproxy.mysql-ha.svc.cluster.local -uroot -p'<root_password>'

---

## Nineteen, Step 9: Launch Temporary Client and Connect to Cluster

Recommended to directly launch the temporary client and connect in one step:

    kubectl run -n mysql-ha -i --rm --tty percona-client-2 \
      --image=percona:8.0 \
      --restart=Never \
      -- mysql -h cluster1-pxc-db-haproxy.mysql-ha.svc.cluster.local -uroot -p'<root_password>'

Note:

- Replace `<root_password>` with the real password you decrypted earlier
- If the `mysql>` doesn't appear immediately after the first execution, wait a moment
- When prompted to press Enter, press Enter

After a successful connection, you'll see similar output:

- `Welcome to the MySQL monitor`
- `Server version: 8.4.7-7.1 Percona XtraDB Cluster`

This indicates:

> **The HAProxy access chain has been successfully established.**

---

## Twenty, Step 10: Insert Test Data

After a successful connection, execute the following SQL:

    CREATE DATABASE mydb;
    USE mydb;

    CREATE TABLE extraordinary_gentlemen (
      id INT NOT NULL AUTO_INCREMENT,
      name VARCHAR(255) NOT NULL,
      occupation VARCHAR(255),
      PRIMARY KEY (id)
    );

INSERT INTO extraordinary_gentlemen (name, occupation)
VALUES ('Zhang San', 'SRE');

SELECT * FROM extraordinary_gentlemen;

Expected behavior:

- Database creation successful
- Table creation successful
- Insert successful
- Query returns the newly inserted data

If all these succeed, it indicates:

- The cluster is now connectable
- HAProxy is properly forwarding traffic
- Business write operations are fully connected

---

## 21. Step 11: Basic Fault Simulation

This section is not about full disaster recovery, but rather a basic simulation:

> **Delete one PXC data node Pod, observe cluster recovery, and verify data accessibility again.**

### 1) List PXC Pods first

    kubectl get pods -n mysql-ha

For example, you will see:

- `cluster1-pxc-db-pxc-0`
- `cluster1-pxc-db-pxc-1`
- `cluster1-pxc-db-pxc-2`

### 2) Delete one PXC Pod

For example:

    kubectl delete pod -n mysql-ha cluster1-pxc-db-pxc-1

Notes:

- Only delete one Pod
- Do not delete multiple consecutively
- This stage is only for basic recovery observation

### 3) Observe the recovery process

    kubectl get pods -n mysql-ha -w
    kubectl get pxc -n mysql-ha -w

Expected behavior:

- The deleted Pod will be recreated
- Cluster status will change
- Eventually return to a stable state

### 4) Reconnect and verify data

Re-enter the temporary client:

    kubectl run -n mysql-ha -i --rm --tty percona-client-3 \
      --image=percona:8.0 \
      --restart=Never \
      -- mysql -h cluster1-pxc-db-haproxy.mysql-ha.svc.cluster.local -uroot -p'<root_password>'

Execute:

    USE mydb;
    SELECT * FROM extraordinary_gentlemen;

If the data is still present, it indicates:

- The cluster has recovered after Pod deletion
- HAProxy entry is still accessible
- Data remains readable

---

## 22. What level of success is considered successful in this section

This section is considered successful not just by all Pods being `Running`, but by meeting at least the following conditions simultaneously:

### 1) Deployment layer success
- Operator is functioning normally
- PXC 3-node cluster is normal
- HAProxy 2 instances are normal
- `kubectl get pxc -n mysql-ha` shows `ready`

### 2) Access layer success
- Can connect to the database via `cluster1-pxc-db-haproxy`

### 3) Read/write layer success
- Can create databases
- Can create tables
- Can insert data
- Can query data

### 4) Recovery layer success
- After deleting a PXC Pod, it can be restarted automatically

### 5) Business continuity after recovery
- Successfully connect via HAProxy after recovery
- Previously written data is still queryable

If all these conditions are met, this section is not just "appearing to be deployed successfully," but:

> **Basic deployment, access, read/write, recovery, and post-recovery data access have all been verified.**

---

## 23. Common Issues Troubleshooting

### 1) `kubectl get pxc` is not `ready` for a long time
Check first:

    kubectl get pods -n mysql-ha
    kubectl describe pxc cluster1-pxc-db -n mysql-ha
    kubectl logs -n mysql-ha deployment/my-op-pxc-operator --tail=100

### 2) PXC Pod `OOMKilled`
Check first:

    kubectl describe pod -n mysql-ha <pxc-pod-name>
    kubectl top nodes

Key points to judge:

- Don't only look at Pods
- Combine with node physical memory
- Host machine is too small, adjusting values alone may not be sufficient

### 3) PXC Pod is always `Pending`
Check first:

    kubectl describe pod -n mysql-ha <pxc-pod-name>
    kubectl get pvc -n mysql-ha

Focus on:

- Whether PVC is bound
- Whether master has taint
- Whether anti-affinity rules are triggered

### 4) `haproxy.size: 1` causes cluster to fail directly
Check first:

    kubectl get pxc -n mysql-ha
    kubectl describe pxc cluster1-pxc-db -n mysql-ha
    kubectl logs -n mysql-ha deployment/my-op-pxc-operator --tail=100

### 5) Terminal is unresponsive when connecting to database
Don't immediately assume the command is wrong.  
First consider:

- Temporary client Pod is still starting
- Image is still pulling
- Attach is not yet complete

### 6) Root password is unreadable
The recommended way is to decode directly from Secret:

    kubectl get secret cluster1-pxc-db-secrets \
      -n mysql-ha \
      --template='{{.data.root | base64decode}}{{"\n"}}'

### 7) Deleting namespace is stuck
Check first:

    kubectl get pxc -n mysql-ha
    kubectl get namespace mysql-ha -o yaml

If you see finalizer-related prompts, prioritize patching the finalizer of PXC resources.

---

## 24. Step 12: Clean Up Environment

After the experiment, it's recommended to clean up in order:

### 1) Delete database cluster release first

    helm uninstall cluster1 -n mysql-ha

### 2) Then delete Operator release

    helm uninstall my-op -n mysql-ha

### 3) Finally delete the namespace

    kubectl delete namespace mysql-ha

### 4) Check for residual volumes if necessary

    kubectl get pvc -n mysql-ha
    kubectl get pv

Note: /think

- Percona CR with finalizer
- When deleting a namespace, do not wait indefinitely if it gets stuck
- Prioritize checking if there are any residual `PerconaXtraDBCluster` resources left

---

## 25. Supplementary Understanding: Why This Article is Closer to a High Availability Cluster

Because the core of Article 15 is:

- replication
- master writes, slave reads
- master-slave relationship verification

While the core of this article is:

- PXC data node cluster
- HAProxy unified access
- Operator continuous management
- Recovery after single-node deletion
- Data remains accessible after recovery

In other words:

### Article 15 solves
> How to deploy MySQL master-slave replication

### Article 16 solves
> How to deploy and verify basic recovery capabilities of a MySQL high availability cluster managed by Operator

---

## 26. Conclusion of This Article

This article has completed the deployment and verification experiment of a MySQL high availability cluster based on Percona Operator.

Currently verified:

- `kubectl get pxc -n mysql-ha` status is `ready`
- 3 PXC nodes and 2 HAProxy instances are all running normally
- Successfully connected to the database via HAProxy Service
- Completed database creation, table creation, data insertion, and query verification
- Completed observation of recovery after single-node Pod deletion
- Reconnected to the database after recovery and confirmed that original data remains accessible

This indicates:

> **This Percona Operator-based PXC high availability cluster has completed the minimum closed-loop verification of basic deployment, access, read/write, recovery, and business continuity after recovery.**

---

## Official Reference Links
- Percona Operator for MySQL based on PXC overview  
  https://docs.percona.com/percona-operator-for-mysql/pxc/index.html

- Percona Operator for MySQL using Helm installation  
  https://docs.percona.com/percona-operator-for-mysql/pxc/helm.html

- Percona Operator for MySQL Quickstart  
  https://docs.percona.com/percona-operator-for-mysql/pxc/quickstart.html

- Percona Operator for MySQL connecting to the database  
  https://docs.percona.com/percona-operator-for-mysql/pxc/connect.html

- Percona Operator for MySQL inserting test data  
  https://docs.percona.com/percona-operator-for-mysql/pxc/data-insert.html

- Percona Operator for MySQL Custom Resource options reference  
  https://docs.percona.com/percona-operator-for-mysql/pxc/operator.html

- Default values of pxc-db in Percona Helm Charts  
  https://github.com/percona/percona-helm-charts/blob/main/charts/pxc-db/values.yaml