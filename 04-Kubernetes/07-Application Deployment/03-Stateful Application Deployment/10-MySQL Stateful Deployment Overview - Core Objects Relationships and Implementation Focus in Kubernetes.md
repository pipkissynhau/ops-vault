# 10-MySQL Stateful Deployment Overview: Core Objects, Relationships, and Implementation Focus in Kubernetes

## Document Purpose
- Document Positioning: Overview of MySQL stateful deployment in Kubernetes
- Applicable Stage: After completing persistent storage, StatefulSet, Headless Service, service discovery, and initialization basics, entering specific middleware deployment understanding
- Recommended Path: `04-Kubernetes/07-Apply deployment/03-Apply deployment in status/10-MySQL Overview with status:Kubernetes Core audience, relationship and focus`

## Tags
#Kubernetes #MySQL #StatefulSet #PVC #ConfigMap #Secret #Service #ApplyWithStatus #DatabaseDeployment #ApplyDeployment #OperationalContainerization #Clouds. #Transport

---

## I. Why Learn MySQL Deployment Logic in Kubernetes

Previous content has completed the foundation for stateful application deployment, including:

- Persistent storage
- StatefulSet
- Headless Service
- Service discovery
- Startup order and initialization

These foundations clearly indicate:

> **Database applications cannot be simply deployed as stateless services.**

MySQL is a typical stateful component with the following characteristics:

- Strong dependency on persistent data
- Clear data directory
- Initialization logic on startup
- Configuration typically needs to be externalized
- Sensitive information like passwords needs separate management
- Requires stable access entry

Therefore, the focus of MySQL deployment is not just "can the container start", but:

- Can data be retained long-term
- Can configuration be independently maintained
- Can passwords be properly managed
- Can business access be stable
- Can the database recover as expected after Pod reconstruction

---

## II. MySQL Deployment Focuses on a Group of Objects Rather Than a Single YAML

Deploying MySQL in Kubernetes typically doesn't rely on a single object.

A basic single-instance MySQL model usually involves at least these objects:

- StatefulSet
- PVC
- ConfigMap
- Secret
- Service

These objects solve different problems:

| Object | Primary Responsibility |
|---|---|
| StatefulSet | Hosts MySQL instance itself |
| PVC | Provides persistent data volume |
| ConfigMap | Hosts MySQL configuration files |
| Secret | Hosts database passwords and sensitive information |
| Service | Provides stable access entry |

From deployment model perspective, MySQL's basic implementation can be summarized as:

> **StatefulSet runs the database, PVC stores data, ConfigMap hosts external configuration, Secret manages passwords, and Service provides access entry.**

---

## III. Why MySQL is More Suitable for StatefulSet Rather Than Deployment

Although MySQL can be containerized, it's not suitable for simple Deployment approach.

### Deployment Suitable Scenarios
Deployment is more suitable for:

- Stateless services
- Multi-replica replaceable
- No data identity differences between replicas
- More focus on replica count and rolling updates

### MySQL Closer Scenarios
MySQL is closer to:

- Fixed data directory
- Still wants to reconnect to original data after Pod reconstruction
- Instance identity and data relationship is more important
- May expand to master-slave, master-standby models later

### StatefulSet Adaptation Points
StatefulSet is more suitable for MySQL, mainly because:

- Better expresses stable identity
- Better expresses stable volume relationships
- Easier to support stateful application lifecycle
- Easier to transition from single instance to multi-member model

### Basic Understanding
Deployment emphasizes:

> "How many replicas are needed"

StatefulSet emphasizes:

> "What is this instance, where is its data, will it still be the original instance after reconstruction"

This is exactly what database applications care about most.

---

## IV. Why MySQL's Data Directory is the First Priority in Deployment

MySQL's core asset isn't the image, but the data.

### Why Data Directory is More Important Than Image
Restarting a MySQL container isn't difficult, but if database data isn't preserved, the instance has no continuity for business.

### Typical MySQL Data Directory
In MySQL official image, common data directory is:

    /var/lib/mysql

This means deployment must prioritize:

- Whether this directory is mounted to persistent volume
- Whether data remains after Pod deletion
- Whether it can reconnect to original data after Pod reconstruction

### What Happens Without Persistence
If MySQL data directory isn't correctly mounted to PVC, the result is typically:

- Data loss after Pod deletion
- Restarted as a new instance
- Initialization process repeats
- Business data can't persist

### Operation Focus
When deploying MySQL, the first thing to confirm isn't replica count, but:

> **Where exactly is the data directory located.**

---

## V. What Problem Does PVC Solve in MySQL

PVC's value in MySQL deployment isn't "mounting an additional volume", but decoupling database data from container lifecycle.

### PVC's Role
In MySQL deployment, PVC is typically used for:

- Storing database files
- Storing system tables
- Storing business data
- Letting Pod reconstruction still read original data

### Common Mounting Method
For example:

    volumeMounts:
      - name: mysql-data
        mountPath: /var/lib/mysql

Corresponding PVC or `volumeClaimTemplates`.

### What Does This Mean
This means MySQL's written data won't stay only in container's temporary layer, but will be written to persistent volume.

### Operation Focus
For MySQL and similar databases, PVC isn't an optional enhancement, but a fundamental part of deployment model.

volumeMounts:
  - name: mysql-config
    mountPath: /etc/mysql/conf.d

### Operational Understanding Focus
MySQL configuration is better to follow:

> **Use as generic an image as possible, and externalize configuration as much as possible.**

---

## VII. What Role Do Secrets Typically Play in MySQL

Sensitive information often exists in MySQL deployments, with passwords being the most common.

### Common Sensitive Information Includes
- root password
- business database account passwords
- authentication information used in initialization scripts
- passwords for subsequent replication or read-only accounts

### Why Use Secrets
Because these information, if written directly in YAML in plain text, would bring obvious risks:

- Configuration leakage
- Git repository leakage
- Visibility to multiple people
- Difficulty in rotation later

### Typical Use of Secrets
For example:

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
    type: Opaque
    stringData:
      MYSQL_ROOT_PASSWORD: StrongPass123!

Then reference through environment variables:

    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: MYSQL_ROOT_PASSWORD

### Operational Understanding Focus
MySQL passwords should be handled in separate layers from regular configuration:

- Configuration goes to ConfigMap
- Passwords go to Secret

---

## VIII. What Role Does Service Play in MySQL

After MySQL deployment, business typically needs a stable entry point to access the database.

### Why Not Directly Access Pod IP
Because Pod IP is not a stable identifier.  
Pods may change IP after recreation, and business should not rely on this temporary address.

### Role of Service
Service mainly handles:

- Providing a stable service name
- Shielding Pod IP changes
- Letting business access MySQL through a fixed entry point

### Typical Service Example

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

### Common Access Methods from Business Side
If in the same namespace, business typically accesses via:

    mysql:3306

If across namespaces, you can also use the full service name, for example:

    mysql.default.svc.cluster.local:3306

### Operational Understanding Focus
For business, a more reasonable access method is to connect to Service rather than directly connect to the database Pod.

---

## IX. Why MySQL Initialization in Kubernetes Needs Special Attention

MySQL is not an application that "just needs the main process to start up".

### Common Behaviors on First Start
When the data directory is empty, MySQL typically executes initialization logic, such as:

- Initialize system tables
- Create root password
- Establish basic runtime state
- Execute default initialization process

### Common Behaviors on Subsequent Restarts
If the data directory already contains valid data, MySQL should behave as:

- Read existing data
- Recover startup based on existing data
- Not re-initialize the database directory

### What Does This Mean
This means MySQL's startup process typically implicitly includes a judgment:

> **Whether it is the first initialization or recovery based on existing data.**

### Operational Understanding Focus
When deploying MySQL, you should not only check if the Pod is Running, but also pay attention to:

- Whether the data directory is empty
- Whether initialization was correctly executed
- Whether duplicate initialization was mistakenly triggered

---

## X. Why MySQL Cannot Just Check "Pod Is Successfully Started"

The operational status of database applications cannot be simply equated to container status.

### What Can Pod Running Only Indicate
Pod Running can only indicate:
- The container process has started
- The container is not immediately exiting

### But What Else Involves Database Availability
Database availability may also involve:
- Whether initialization is complete
- Whether configuration is effective
- Whether the port is truly accepting connections
- Whether the data directory is correctly mounted
- Whether passwords are correctly injected
- Whether business can actually connect

### Common Verification Methods
For example:

    kubectl get pod
    kubectl get pvc
    kubectl get svc
    kubectl exec -it mysql-0 -- sh

Or test connection via mysql client within the cluster:

    mysql -h mysql -uroot -p

### Operational Understanding Focus
The criteria for judging MySQL deployment completion should not be:

> "Pod is running"

But should at least include:

- Pod is normal
- PVC is normal
- Service is normal
- Configuration is normal
- Data directory is normal
- Connection test is normal

---

## XI. Several Layers to Consider When Deploying MySQL

To avoid understanding MySQL deployment as a single resource object problem, it can be split into the following layers.

### 1. Container Layer
Includes:
- Image
- Start parameters
- Container port
- Environment variables

### 2. Data Layer
Includes:
- Data directory
- PVC
- StorageClass
- Persistent volume binding

### 3. Configuration Layer
Includes:
- my.cnf
- Custom parameters
- External configuration via ConfigMap

### 4. Security Layer
Includes:
- root password
- Business account passwords
- Secret management

### 5. Network Layer
Includes:
- Service exposure
- Cluster internal access method
- Whether external access is allowed

### 6. Lifecycle Layer
Includes:
- First initialization
- Restart recovery
- Configuration change
- Subsequent upgrades and maintenance

### Operational Understanding Focus
MySQL deployment is a multi-layered combination problem, not a StatefulSet YAML problem.

---

## XII. A Basic MySQL Object Combination Diagram

The following provides a minimal combination diagram for educational purposes to understand object relationships.

### 1. Secret

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
    type: Opaque
    stringData:
      MYSQL_ROOT_PASSWORD: StrongPass123!

### 2. ConfigMap

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

### 3. Service

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

### 4. StatefulSet

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

### This group of objects expresses what
This group of objects expresses:

- MySQL runs via StatefulSet
- Root password comes from Secret
- Configuration comes from ConfigMap
- Data directory comes from PVC
- Business accesses MySQL via Service

---

## Thirteen, What deployment model is more suitable to master first in this stage

Currently, it is more suitable to first master:

> **The basic deployment model of a single MySQL instance.**

### Why start with a single instance first
Because a single instance already covers these core knowledge points:

- Why StatefulSet is more suitable for databases than Deployment
- How PVC carries the database data directory
- How ConfigMap injects configuration
- How Secret manages passwords
- How Service provides a stable access entry
- How MySQL distinguishes between initial setup and recovery startup

### Why not start with master-slave and high availability immediately
Because master-slave and high availability would introduce additional elements:

- Multiple member identities
- Member discovery
- Master-slave relationship establishment
- Initialization and joining logic
- More complex probes and service design

These contents can be expanded after a clear understanding of the single-instance model.

---

## Fourteen, Learning focus of this article

The focus of this article is not to directly obtain a production-grade MySQL solution, but to first establish the following understandings:

### 1. MySQL is a typical stateful application
Its core is data, configuration, passwords, and initialization, rather than just the container process.

### 2. MySQL deployment depends on multiple objects working together
A single StatefulSet is insufficient to express the complete deployment model.

### 3. The data directory must be prioritized for persistence
PVC is one of the fundamental objects in MySQL deployment.

### 4. Configuration and passwords should be managed in layers
ConfigMap manages configuration, Secret manages sensitive information.

### 5. Business access should go through Service
Rather than directly accessing the database Pod IP.

### 6. Initialization and recovery should be distinguished
Database initial startup and subsequent restarts are not the same type of behavior.

---

## Fifteen, Stage summary

MySQL deployment in Kubernetes can be seen as a specific implementation of the general methodology for stateful applications.

In the MySQL deployment process, the most important thing is not "whether you can write a StatefulSet," but whether you can correctly combine the following dimensions:

- Stateful carrying model
- Persistent storage
- Configuration externalization
- Password management
- Stable access entry
- Initialization and recovery logic

From a deployment perspective, a basic usable MySQL model typically includes at least:

- StatefulSet
- PVC
- ConfigMap
- Secret
- Service

These objects collectively form the most basic database runtime unit.

As long as this layer of understanding is clear, subsequent progress into:

- MySQL single-instance deployment practice
- Connectivity verification
- Initialization and probes
- Helm / Operator and other higher-level delivery methods

Will be smoother.

---

## Sixteen, Keyword quick notes

- MySQL: Typical stateful database
- StatefulSet: Stateful carrying object
- PVC: Data persistence object
- ConfigMap: Configuration externalization object
- Secret: Sensitive information management object
- Service: Stable access entry
- `/var/lib/mysql`: MySQL common data directory
- Initialization: Establishing the database's basic state during first startup
- Recovery: Restarting based on existing data directory

## 17. Operational Extension Understanding

MySQL deployment in Kubernetes is a typical middleware containerization exercise.

From the object level perspective, we see:

- StatefulSet
- PVC
- ConfigMap
- Secret
- Service

From the system level perspective, we truly build:

- A database instance that can persistently save data
- A configurable runtime configuration set
- A group of independently managed sensitive information
- A stable service entry point accessible by business applications

Therefore, the value of MySQL deployment is not just learning a database example, but applying the previously established stateful application methodology to a concrete, understandable, and operable object.

---

## References
- Kubernetes StatefulSet Official Documentation
- Kubernetes Persistent Volumes Official Documentation
- Kubernetes ConfigMap Official Documentation
- Kubernetes Secret Official Documentation
- MySQL Official Docker Image Documentation

---

## Next Day's Recommendation
The next article suggests organizing:

[[11-MySQL Single Instance Deployment in Kubernetes - Complete YAML Order of Creation Connectivity Verification]]