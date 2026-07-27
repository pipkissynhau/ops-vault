# 16-Practical Operations on Deploying MySQL High-Availability Clusters: Kubernetes Deployment, Verification, Troubleshooting, and Basic Fault Experiment Based on Percona Operator

## Document Description
- Document Purpose: Practical operations on basic deployment of MySQL high-availability clusters in Kubernetes.
- Applicable Stage: After completing basic MySQL single-instance and master-slave replication setups, proceed to practice deploying high-availability clusters managed by the Operator.
- Objective of This Article: To be able to deploy a MySQL high-availability cluster based on Percona Operator step by step, and complete connection verification, data validation, and basic fault experiments.
- Note: More precisely, this is about deploying **Percona Operator for MySQL based on Percona XtraDB Cluster (PXC)**.
- Recommended Path: `04-Kubernetes/07-Application Deployment/03-Stateful Application Deployment/16-Practical Operations on Deploying MySQL High-Availability Clusters: Kubernetes Deployment, Verification, Troubleshooting, and Basic Fault Experiment Based on Percona Operator.md`

## Tags
#Kubernetes #MySQL #High-Availability #Operator #Percona #PXC #HAProxy #PVC #Secret #Stateful Application #Database Deployment #Application Deployment #Cloud-Native #Operation and Maintenance        enabled: true
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

### Key Points for Understanding This Values File

#### 1) Why is `pxc.size` still set to 3?
This is because the focus of this document is on building a high-availability cluster. It is not recommended to reduce the size of the PXC data layer to 1 or 2.

#### 2) Why is `haproxy.size` set to 2 instead of 1?
This was the first issue encountered during the experiment:

- Initially, when `haproxy.size` was set to 1, the Operator reported an error.
- The command `kubectl get pxc` also displayed an error.
- The logs clearly stated that:
  - `HAProxy size must be at least 2`
  - Unless `unsafeFlags.proxySize=true` is explicitly set.

Therefore, a safer value for experimentation is:

> **Set HAProxy to at least 2.**

#### 3) Why is `pxc.resources.limits.memory` set to 3Gi?
This was also one of the critical issues encountered during the experiment:

- Initially, when the memory limit for PXC was set too low, the pod `pxc-1` was scheduled to the node `k8s-node02`, causing it to frequently encounter OOMKills.
- The host machine's logs showed that the `mysqld` process was killed by the kernel's OOM Killer.
- Only after increasing the memory on `node02` from 2G to 8G did the system stabilize.

Therefore, while setting limits is important, they should be set reasonably and not too strictly.

#### 4) Why are components like `logcollector`, `pmm`, and `backup` disabled?
The current goal of this experiment is not to focus on backup, PMM monitoring, or log collection. Instead, the priority is to deploy the cluster and ensure that it can connect and function properly. Therefore, these additional components are temporarily disabled for stability reasons.

---

## Step 14: Install the Database Cluster

Here, the database cluster will be named `cluster1`:

    helm install cluster1 percona/pxc-db \
      -n mysql-ha \
      -f values-pxc.yaml

Explanation:

- `cluster1`: The name of the database cluster release.
- `percona/pxc-db`: The chart used to deploy the database cluster.
- `-f values-pxc.yaml`: Uses a custom parameter file for configuration.

---

## Step 15: Observe the Cluster Startup Process

### 1) Check the Pods
    kubectl get pods -n mysql-ha -o wide

You will gradually see the following components:

- The Operator Pod.
- The PXC data nodes.
- The HAProxy Pod.

### 2) Monitor the Status of PXC
    kubectl get pxc -n mysql-ha

This command is one of the most critical for observing the cluster's progress.

Expected outcome:

- Initially, it may show `error` or `initializing`.
- Eventually, it should transition to `ready`.

### 3) Verify the PVC Status
    kubectl get pvc -n mysql-ha

Expected outcome:

- The PXC-related PVCs should all be marked as `Bound`.

---

## Step 16: Key Issues Encountered During the Experiment

### Issue 1: `haproxy.size: 1` is blocked by the Operator's default security settings
Phenomenon:

- The command `kubectl get pxc -n mysql-ha` returns an error.
- No PXC or HAProxy Pods are created.
- Logs from both the Operator and `kubectl describe pxc` clearly indicate that:
  - `HAProxy size must be at least 2`.

Conclusion:

> **In this experimental environment, do not set `haproxy.size` to 1.**

---

### Issue 2: PXC Nodes Facing OOMKills
Phenomenon:

- The pod `pxc-1` repeatedly encounters OOMKills.
- The `kubectl describe pod` output shows `Reason: OOMKilled`.
- System logs on the host machine reveal that processes like `mysqld` and `pxc-entrypoint` are being killed by the kernel.

Key points for identification:

- This issue is not related to Helm or the probes themselves.
- It is caused by insufficient memory on the host machine, preventing the🔤 kubectl run -n mysql-ha -i --rm --tty percona-client-2 \
  --image=percona:8.0 \
  --restart=Never \
  -- mysql -h cluster1-pxc-db-haproxy.mysql-ha.svc.cluster.local -uroot -p '<root_password>'

Note:

- Replace `<root_password>` with the actual password you obtained earlier.
- If the `mysql` prompt does not appear immediately after the first execution, wait for a moment.
- When prompted to press Enter, just press Enter.

After a successful connection, you should see something similar to this:

- `Welcome to the MySQL monitor`
- `Server version: 8.4.7-7.1 Percona XtraDB Cluster`

This indicates that:

> **The HAProxy access link has been established.**

---

## Step 20: Insert Test Data

After a successful connection, execute the following SQL commands:

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

Expected results:

- The database is created successfully.
- The table is created successfully.
- The data is inserted successfully.
- The query returns the newly inserted data.

If all these steps are successful, it means that:

- The cluster is accessible.
- HAProxy is functioning correctly for forwarding requests.
- Data writes are working properly.

---

## Step 21: Conduct a Basic Fault Experiment

This experiment does not involve a full disaster recovery process but rather a basic test:

> **Delete one PXC data node Pod and observe the cluster's recovery, then verify that the data is still accessible.**

### 1) List all PXC Pods

    kubectl get pods -n mysql-ha

For example, you might see:

- `cluster1-pxc-db-pxc-0`
- `cluster1-pxc-db-pxc-1`
- `cluster1-pxc-db-pxc-2`

### 2) Delete one of the PXC Pods

For example:

    kubectl delete pod -n mysql-ha cluster1-pxc-db-pxc-1

Note:

- Only delete one Pod at a time.
- At this stage, we are only observing the recovery process.

### 3) Monitor the recovery process

    kubectl get pods -n mysql-ha -w
    kubectl get pxc -n mysql-ha -w

Expected results:

- The deleted Pod will be recreated.
- The cluster status will change.
- Eventually, the cluster will return to a stable state.

### 4) Reconnect and verify the data

Re-enter the temporary client:

    kubectl run -n mysql-ha -i --rm --tty percona-client-3 \
      --image=percona:8.0 \
      --restart=Never \
      -- mysql -h cluster1-pxc-db-haproxy.mysql-ha.svc.cluster.local -uroot -p '<root_password>'

Execute the following commands:

    USE mydb;
    SELECT * FROM extraordinary_gentlemen;

If the data is still available, it means that:

- The cluster has recovered after the Pod was deleted.
- HAProxy can still be used to access the database.
- The data remains accessible.

---

## Step 22: What Constitutes a Successful Outcome

A successful outcome of this experiment is not just about all Pods being in the `Running` state but also about meeting at least the following conditions:

### 1) Deployment Layer Success
- The Operator is functioning normally.
- All 3 PXC nodes are running correctly.
- Both HAProxy instances are working properly.
- `kubectl get pxc -n mysql-ha` shows `ready`.

### 2) Access Layer Success
- It is possible to connect to the database through `cluster1-pxc-db-haproxy`.

### 3) Read/Write Layer Success
- The database can be created, tables can be created, data can be inserted, and queries can be executed.

### 4) Recovery Layer Success
- After deleting one PXC Pod, it can be successfully reactivated.

### 5) Business Continuity after Recovery
- Reconnecting through HAProxy after recovery is successful.
- Previously written data can still be accessed.

If all these conditions are met, it means that:

> **The basic deployment, access, read/write operations, recovery process, and post-recovery data accessibility have all been successfully verified.**

---

## Step 23: Common Issues and Troubleshooting

### 1) `kubectl get pxc` takes a- Installing Percona Operator for MySQL using Helm  
  https://docs.percona.com/percona-operator-for-mysql/pxc/helm.html

- Quickstart for Percona Operator for MySQL  
  https://docs.percona.com/percona-operator-for-mysql/pxc/quickstart.html

- Connecting to the Database with Percona Operator for MySQL  
  https://docs.percona.com/percona-operator-for-mysql/pxc/connect.html

- Inserting Test Data Using Percona Operator for MySQL  
  https://docs.percona.com/percona-operator-for-mysql/pxc/data-insert.html

- Reference for Custom Resource Options of Percona Operator for MySQL  
  https://docs.percona.com/percona-operator-for-mysql/pxc/operator.html

- Default Values for the pxc-db Resource in Percona Helm Charts  
  https://github.com/percona/percona-helm-charts/blob/main/charts/pxc-db/values.yaml