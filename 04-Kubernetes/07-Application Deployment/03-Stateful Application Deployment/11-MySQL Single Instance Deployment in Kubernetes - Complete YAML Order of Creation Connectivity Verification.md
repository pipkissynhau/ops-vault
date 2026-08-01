# 11-MySQL Single Instance Deployment in Kubernetes: Complete YAML, Creation Order, and Connectivity Verification

## Document Purpose
- Document Purpose: A basic hands-on example of deploying a single MySQL instance in Kubernetes
- Applicable Stage: After understanding the overview of MySQL stateful deployment, entering the most basic and implementable single-instance deployment practice
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/11-MySQL Single example in Kubernetes Deployment in progress: complete YAML, creation order and connectivity validation`

## Tags
#Kubernetes #MySQL #StatefulSet #PVC #ConfigMap #Secret #Service #ApplyWithStatus #DatabaseDeployment #ApplyDeployment #OperationalContainerization #Clouds. #Transport

---

## One: Hands-on Objectives

This article aims to complete the most basic MySQL single-instance deployment and verify the following:

- MySQL Pod can start normally
- root password injected via Secret
- MySQL configuration mounted via ConfigMap
- Data directory persisted via PVC
- Business can access MySQL via Service
- Data remains after Pod recreation

This article focuses on establishing a minimal, understandable, executable, and verifiable MySQL base deployment model, without involving more complex topics like master-slave replication, high availability, or automatic failover.

---

## Two: Experiment Object Relationship Diagram

This article will create the following objects:

- `mysql-secret`
- `mysql-config`
- `mysql`
- `mysql` StatefulSet
- `mysql-data-mysql-0` PVC (generated automatically by `volumeClaimTemplates`)

Overall relationship:

    Secret(mysql-secret)
            |
            v
    StatefulSet(mysql)
       |        \
       |         \
       |          -> ConfigMap(mysql-config)
       |
       -> volumeClaimTemplates -> PVC(mysql-data-mysql-0)
       |
       -> Pod(mysql-0)
       |
       -> Service(mysql:3306)

Other cluster business accesses via:

    mysql:3306

to this database instance.

---

## Three: Preparation Directory Structure

It is recommended to create a separate directory to save this article's YAML:

    mkdir -p mysql-single
    cd mysql-single

Recommended file split:

    01-secret.yaml
    02-configmap.yaml
    03-service.yaml
    04-statefulset.yaml

This split makes the structure clearer, and the boundaries are more distinct when modifying configurations, passwords, networks, and workloads later.

---

## Four: Write Secret: Save root Password

### File: `01-secret.yaml`

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
    type: Opaque
    stringData:
      MYSQL_ROOT_PASSWORD: StrongPass123!

### Explanation

This Secret only does one thing:

- Save root password

Here we use `stringData`, which facilitates direct plaintext testing values.  
In real environments, passwords should be generated and managed through standardized processes, and it's not recommended to use the fixed password from teaching examples long-term.

---

## Five: Write ConfigMap: Save MySQL Configuration

### File: `02-configmap.yaml`

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: mysql-config
    data:
      my.cnf: |
        [mysqld]
        character-set-server=utf8mb4
        collation-server=utf8mb4_general_ci
        max_connections=200
        default-authentication-plugin=mysql_native_password

### Explanation

This ConfigMap is used to inject a basic configuration file into MySQL.

The parameters in this example have the following effects:

- `character-set-server=utf8mb4`  
  Set the default character set for the server.

- `collation-server=utf8mb4_general_ci`  
  Set the default collation.

- `max_connections=200`  
  Set the maximum number of connections.

- `default-authentication-plugin=mysql_native_password`  
  Compatible with some old clients or old business connection methods.

This article does not expand on MySQL parameter tuning, focusing instead on establishing the habit of "externalizing configuration" in deployment.

---

## Six: Write Service: Provide Stable Access Entry

### File: `03-service.yaml`

    apiVersion: v1
    kind: Service
    metadata:
      name: mysql
    spec:
      selector:
        app: mysql
      ports:
        - port: 3306
          targetPort: 3306

### Explanation

The purpose of this Service is:

- Provide a stable access entry for MySQL
- Shield Pod IP changes
- Allow cluster business to access the database via service name

This does not explicitly write `type`, defaulting to `ClusterIP`.  
Therefore, this MySQL service is mainly used for:

- Cluster business access
- Cluster testing access
- Maintenance temporary troubleshooting

---

## Seven: Write StatefulSet: Run MySQL Instance

### File: `04-statefulset.yaml`

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
            - name: mysql-config
              mountPath: /etc/mysql/conf.d
          readinessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 60
            periodSeconds: 20
      volumes:
        - name: mysql-config
          configMap:
            name: mysql-config
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi

---

## Eight. Understanding This StatefulSet Segment by Segment

This YAML isn't simply running a container, but expressing a complete database runtime model.

### 1. `serviceName: mysql`

    serviceName: mysql

This indicates this StatefulSet is paired with a Service named `mysql`.  
In this single-instance model, the focus is establishing a stable access entry point, not member discovery.

### 2. `replicas: 1`

    replicas: 1

Indicates only one MySQL instance is running.

### 3. `selector` and `template.labels`

    selector:
      matchLabels:
        app: mysql

    template:
      metadata:
        labels:
          app: mysql

These two sections must remain consistent.  
Otherwise, the Service may fail to select Pods, and the StatefulSet may experience ownership relationship issues.

### 4. Root password comes from Secret

    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: MYSQL_ROOT_PASSWORD

Indicates the container reads the root password from `mysql-secret` during startup.

### 5. Data directory mounted to persistent volume

    volumeMounts:
      - name: mysql-data
        mountPath: /var/lib/mysql

Indicates MySQL's critical data directory uses a persistent volume, not the container's temporary layer.

### 6. Configuration directory mounted ConfigMap

    volumeMounts:
      - name: mysql-config
        mountPath: /etc/mysql/conf.d

Indicates external configuration is injected into MySQL's common configuration directory via ConfigMap.

### 7. Probes

    readinessProbe:
      tcpSocket:
        port: 3306

    livenessProbe:
      tcpSocket:
        port: 3306

Here we use the most basic TCP probe for teaching and basic verification.  
It can only indicate whether the port is open, not fully equivalent to the database being truly available.  
We'll refine probe understanding in the advanced section later.

### 8. `volumeClaimTemplates`

    volumeClaimTemplates:
      - metadata:
          name: mysql-data

Indicates the StatefulSet will automatically generate a corresponding PVC for this instance.  
The final common result is generating a PVC name like:

    mysql-data-mysql-0

---

## Nine. Creation Order

Create objects in this order:

### 1. Create Secret

    kubectl apply -f 01-secret.yaml

### 2. Create ConfigMap

    kubectl apply -f 02-configmap.yaml

### 3. Create Service

kubectl apply -f 03-service.yaml

### 4. Create StatefulSet

    kubectl apply -f 04-statefulset.yaml

---

## 10. Basic Check Commands

After object creation, first check the overall status.

### 1. View Core Objects

    kubectl get secret,configmap,svc,sts,pod,pvc

### 2. Focus on StatefulSet

    kubectl get sts mysql
    kubectl describe sts mysql

### 3. View Pod

    kubectl get pod -l app=mysql -o wide
    kubectl describe pod mysql-0

### 4. View PVC

    kubectl get pvc
    kubectl describe pvc mysql-data-mysql-0

### 5. View Service

    kubectl get svc mysql
    kubectl describe svc mysql

---

## 11. Expected Status Explanation

If deployment is successful, you should typically see the following results.

### 1. Pod Status
Should see:

- `mysql-0` is `Running`
- `READY` eventually becomes `1/1`

### 2. PVC Status
Should see:

- `mysql-data-mysql-0` is `Bound`

### 3. Service Status
Should see:

- `mysql` Service has been created
- Port is `3306`

### 4. StatefulSet Status
Should see:

- `READY` is `1/1`

---

## 12. Check if Configuration Files are Mounted Successfully

Enter the MySQL container:

    kubectl exec -it mysql-0 -- sh

After entering, execute:

    ls /etc/mysql/conf.d
    cat /etc/mysql/conf.d/my.cnf

If mounted successfully, you should see the content defined in the ConfigMap `my.cnf`.

---

## 13. Check if Data Directory is Mounted Successfully

Still inside the container, execute:

    ls /var/lib/mysql

If it's the first initialization, you'll typically see MySQL initialization data files and system library directories.

You can also exit the container and check the mount information:

    kubectl describe pod mysql-0

In the `Mounts` section, confirm:

- `/var/lib/mysql`
- `/etc/mysql/conf.d`

Both paths should be correctly mounted.

---

## 14. Connectivity Verification: Cluster Internal Client Access MySQL

To verify Service and database connectivity, start a temporary MySQL client Pod within the cluster.

### 1. Start Temporary Client

    kubectl run mysql-client --rm -it --image=mysql:8.0 -- bash

### 2. Connect to MySQL in the Client Container

    mysql -h mysql -uroot -p

Enter password:

    StrongPass123!

If connection is successful, you should enter the MySQL command line.

### 3. Simple Verification

After entering MySQL, execute:

    show databases;

If you can see the default database list, it indicates:

- Service is normal
- MySQL is listening on 3306
- Root password injection is normal
- Client can access the database through Service

---

## 15. Write Test Data and Verify Persistence

### 1. Create Test Database in MySQL Client

    create database appdb;

### 2. Use Test Database

    use appdb;

### 3. Create Test Table

    create table t1 (
      id int primary key auto_increment,
      name varchar(50) not null
    );

### 4. Insert Test Data

    insert into t1(name) values ('mysql-on-k8s');

### 5. Query Test Data

    select * from t1;

If you can see the written results, it indicates normal database read/write functionality.

---

## 16. Verify Data Persistence After Pod Rebuild

This is one of the most important verifications in this article.

### 1. Delete MySQL Pod

    kubectl delete pod mysql-0

### 2. Wait for StatefulSet to Automatically Restart New Pod

    kubectl get pod -w

Wait until the new `mysql-0` enters `Running` / `Ready` status.

### 3. Re-enter Client to Connect MySQL

    kubectl run mysql-client --rm -it --image=mysql:8.0 -- bash

After entering, execute:

    mysql -h mysql -uroot -p

After entering password, execute:

    use appdb;
    select * from t1;

### 4. Expected Results
If data still exists, it indicates:

- PVC is working properly
- Data directory is not lost when Pod is deleted
- The MySQL instance has the most basic stateful deployment characteristics

---

## 17. Common Issue 1: PVC Always Pending

### Phenomenon
Execute:

    kubectl get pvc

You see PVC status always as `Pending`.

### Common Causes
- Default `StorageClass` does not exist
- Storage provider is not installed
- Requested capacity is unreasonable
- Underlying storage is unavailable

### Troubleshooting Commands

    kubectl get storageclass
    kubectl describe pvc mysql-data-mysql-0

### Resolution Directions
- Confirm if available `StorageClass` exists in the cluster
- Confirm if dynamic provisioning is supported in the current environment
- Explicitly specify `storageClassName` if necessary

---

## 18. Common Issue 2: Pod Fails to Start or Reboots Repeatedly

### Phenomenon
Execute:

    kubectl get pod

You see `CrashLoopBackOff`, `Error`, or `Init` anomalies.

### Troubleshooting Commands

    kubectl logs mysql-0
    kubectl describe pod mysql-0

### Common Causes
- Secret key name mismatch
- Configuration file format error
- Data directory permission or content anomaly
- MySQL image startup parameter incompatibility

### Key Checks
- `MYSQL_ROOT_PASSWORD` whether successfully injected
- `/etc/mysql/conf.d/my.cnf` whether syntax issues exist
- `/var/lib/mysql` whether correctly mounted

---

## 19. Common Issue Three: Service Exists but Client Cannot Connect

### Phenomenon
`kubectl get svc mysql` is normal, but `mysql -h mysql -uroot -p` cannot connect.

### Troubleshooting Directions

#### 1. Check if Service selector matches Pod labels

    kubectl get pod --show-labels
    kubectl get svc mysql -o yaml

Confirm:
- Service selector is `app: mysql`
- Pod labels are `app: mysql`

#### 2. Check if MySQL Pod is Ready

    kubectl get pod mysql-0

#### 3. Check if container 3306 is listening normally

    kubectl exec -it mysql-0 -- sh

After entering, you can check MySQL process and port status.

---

## 20. Common Issue Four: ConfigMap Changes Not Taking Effect

### Cause
MySQL reads configuration files typically during startup.  
Even if ConfigMap content is updated, the MySQL process won't automatically reload all configurations.

### Resolution
Usually need to restart Pod to let MySQL reload configuration:

    kubectl delete pod mysql-0

### Note
Before restarting, confirm configuration file content is correct to avoid new Pod startup failure due to incorrect configuration.

---

## 21. Common Issue Five: Mistakenly Exposing Database as Ordinary Business Service

### Explanation
MySQL should generally not be exposed to external use via NodePort / LoadBalancer by default.  
More common practices are:

- Cluster internal business accesses via ClusterIP Service
- Management or debugging scenarios use port forwarding

### Temporary Port Forwarding Example

    kubectl port-forward svc/mysql 3306:3306

Then access from local machine:

    mysql -h 127.0.0.1 -P 3306 -uroot -p

This method is more suitable for learning environments and temporary operations verification.

---

## 22. Boundary of This Hands-on Model

This model only solves the most basic single-instance deployment issues, and does not cover the following:

- Master-slave replication
- Read-write separation
- Automatic failover
- Multi-replica high availability
- Backup and recovery system
- Operator management
- Complex probes and parameter tuning

These topics should be expanded after a clear understanding of the current basic model.

---

## 23. Stage Summary

This MySQL single-instance hands-on exercise verifies the most basic database deployment model:

- StatefulSet runs the instance
- PVC handles data persistence
- ConfigMap injects configuration
- Secret manages passwords
- Service provides stable access entry

Through this hands-on exercise, you should at least master the following key points:

- MySQL data directory must be persistent
- Database configuration should be externalized as much as possible
- Passwords should not be written in plaintext in workload objects
- Business access should use Service rather than Pod IP
- Verify data remains after Pod reconstruction

Although this model is basic, it already demonstrates the minimal runnable form of a typical stateful application in Kubernetes.

---

## 24. Keyword Mnemonics

- StatefulSet: MySQL's stateful hosting object
- PVC: MySQL data persistence object
- ConfigMap: MySQL configuration externalization object
- Secret: MySQL password management object
- Service: MySQL stable access entry
- `/var/lib/mysql`: MySQL common data directory
- `/etc/mysql/conf.d`: MySQL common configuration directory
- `mysql:3306`: Common access method within cluster

---

## 25. Operations Extension Understanding

Deploying MySQL single-instance in Kubernetes is a typical middleware containerization exercise.

The key to such deployment isn't the number of objects, but whether object boundaries are clear:

- Data belongs to data
- Configuration belongs to configuration
- Keys belong to keys
- Access belongs to access
- Workloads belong to workloads

Once object responsibilities are clear, subsequent learning will be easier:

- Initialization and recovery
- Probes
- Backup and recovery
- Helm
- Operator
- More complex database models

Will form stable understanding rather than just "knowing how to write a YAML".

---

## References
- Kubernetes StatefulSet official documentation
- Kubernetes Persistent Volumes official documentation
- Kubernetes ConfigMap official documentation
- Kubernetes Secret official documentation
- MySQL Official Docker Image documentation

---

## Next Day Suggestions
Next post suggestion: organize

[[12-Advanced Understanding of MySQL in Kubernetes - Initialization Probes Backup Recovery and Supplement on Delivery Methods]]