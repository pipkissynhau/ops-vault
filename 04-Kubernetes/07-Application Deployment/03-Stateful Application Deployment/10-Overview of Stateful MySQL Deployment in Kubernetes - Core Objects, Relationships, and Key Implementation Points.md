# 10-Overview of Stateful MySQL Deployment in Kubernetes: Core Objects, Relationships, and Key Implementation Points

## Document Description
- Documentation Purpose: An overview of stateful MySQL deployment in Kubernetes.
- Applicable Phase: After establishing persistent storage, StatefulSet, Headless Service, service discovery, and basic initialization, this document guides you through understanding specific middleware deployments.
- Recommended Path: `04-Kubernetes/07-Application Deployment/03-Stateful Application Deployment/10-Overview of Stateful MySQL Deployment in Kubernetes: Core Objects, Relationships, and Key Implementation Points`

## Tags
#Kubernetes #MySQL #StatefulSet #PVC #ConfigMap #Secret #Service #Stateful Applications #Database Deployment #Application Deployment #Business Containerization #Cloud Native #Ops

---

## I. Why Learn About MySQL Deployment in Kubernetes

The previous content has laid the foundation for stateful application deployment, including:

- Persistent storage
- StatefulSet
- Headless Service
- Service discovery
- Startup order and initialization

These basics clearly illustrate one thing:

> **Database applications cannot be deployed simply as stateless services.**

MySQL is a typical stateful component with the following characteristics:

- Strong dependence on persistent data
- Clear data directories
- Initialization logic during startup
- Configuration usually requires external management
- Sensitive information such as passwords needs to be managed separately
- Must be accessible through stable interfaces for business use

Therefore, the focus of MySQL deployment is not just whether the container can start, but rather:

- Whether data can be retained long-term
- Whether configuration can be independently maintained
- Whether passwords can be managed securely
- Whether businesses can access the database stably
- Whether the database can be restored as expected after Pod reconstruction

---

## II. MySQL Deployment Depends on a Set of Objects Working Together, Not Just a Single YAML File

Deploying MySQL in Kubernetes typically involves more than one object.

A basic single-instance MySQL model usually involves at least the following objects:

- StatefulSet
- PVC
- ConfigMap
- Secret
- Service

These objects address different needs:

| Object | Main Function |
|---|---|
| StatefulSet | Hosts the MySQL instance itself |
| PVC | Provides persistent data volumes |
| ConfigMap | Stores MySQL configuration files |
| Secret | Holds sensitive information like database passwords |
| Service | Provides a stable access point |

From a deployment perspective, the basic approach for implementing MySQL can be summarized as:

> **StatefulSet runs the database, PVC stores data, ConfigMap manages external configurations, Secret handles passwords, and Service provides access.**

---

## III. Why StatefulSet Is More Suitable for MySQL Than Deployment

Although MySQL can also be containerized, it is not suitable for a straightforward Deployment approach.

### Scenarios More Suitable for Deployment
Deployment is better suited for:

- Stateless services
- Multiple replaceable replicas
- No data identity differences between replicas
- Focusing more on the number of replicas and rolling updates

### Scenarios More Suitable for MySQL
MySQL is more appropriate for:

- Applications with fixed data directories
- Cases where data needs to be retained after Pod reconstruction
- Situations where instance identity and data relationships are crucial
- Potential expansion to more complex models like master-slave or primary-backup setups

### Adaptability of StatefulSet
StatefulSet is more suitable for MySQL because it:

- Better expresses stable identity
- Better maintains stable volume relationships
- More easily manages the lifecycle of stateful applications
- Facilitates transition from single-instance to multi-member models

### A Fundamental Understanding
Deployment emphasizes:

> “How many replicas are needed”

StatefulSet emphasizes:

> “Who this instance is, where its data is located, and whether it remains the same after reconstruction”

These are precisely the concerns for database applications.

---

## IV. Why MySQL’s Data Directory Is the Top Priority in Deployment

The core asset of MySQL is not the image but the data.

### Why the Data Directory Is More Important Than the Image
Restarting a MySQL container is not difficult, but if the data is not preserved, the instance will lack continuity for the business.

### Typical MySQL Data Directory
In official MySQL images, the common data directory is:

    /var/lib/mysql

This means that when deploying MySQL, the following must be prioritized:

- Whether this directory is mounted on a persistent volume
- Whether data can be retained after Pod deletion
- Whether data can be restored after Pod reconstruction

### Consequences of Not Using Persistence
If MySQL’s data directory is not correctly mounted on a PVC, the typical outcomes are:

- Data loss after Pod deletion
- The instance is treated as new upon restart
- Repeated initialization processes
- Inability to maintain business data continuity

### Key Points for Ops Professionals
When deploying MySQL, the first thing to confirm is not the number of replicas but:

> **Where the data directory is located.**

---

## V. What Problem Does PVC Solve### Common Behaviors During Subsequent Reboots
If valid data already exists in the data directory, MySQL should behave as follows:

- Read the existing data.
- Resume startup based on the existing data.
- Do not reinitialize the database directory.

### What Does This Mean?
This means that MySQL's startup process usually involves an implicit determination of whether it is:

> **performing initial initialization for the first time or resuming from existing data.**

### Key Points for Operations and Maintenance
When deploying MySQL, it is important to not only check whether the Pod is Running but also pay attention to the following:

- Whether the data directory is empty.
- Whether initialization was successfully executed.
- Whether repeated initialization was accidentally triggered.

---

## X. Why Can’t We Rely Solely on “Whether the Pod Has Successfully Started” to Assess MySQL’s Status
The running status of database applications cannot be simply equated with that of containers.

### What Does “Pod Running” Indicate?
“Pod Running” only indicates that:

- The container process has started.
- The container is not exiting immediately at present.

### But What Else Is Needed for a Database to Be Available?
For a database to be available, the following may also be necessary:

- Whether initialization is complete.
- Whether configurations have taken effect.
- Whether ports are actually accessible for connections.
- Whether the data directory is correctly mounted.
- Whether passwords have been correctly set.
- Whether the business logic can actually communicate with the database.

### Common Verification Methods
For example:

    kubectl get pod
    kubectl get pvc
    kubectl get svc
    kubectl exec -it mysql-0 -- sh

Or test the connection within the cluster using the mysql client:

    mysql -h mysql -uroot -p

### Key Points for Operations and Maintenance
The criteria for determining that MySQL deployment is complete should not be merely “the Pod exists”, but should at least include:

- The Pod is functioning normally.
- The PVC is normal.
- The Service is normal.
- Configurations are correct.
- The data directory is in good order.
- Connection tests are successful.

---

## XI. Several Layers to Consider When Deploying MySQL
To avoid misunderstanding MySQL deployment as a matter involving only a single resource object, it can be broken down into the following layers:

### 1. Container Layer
This includes:
- Image.
- Startup parameters.
- Container ports.
- Environment variables.

### 2. Data Layer
This includes:
- Data directory.
- PVC.
- StorageClass.
- Persistence volume binding.

### 3. Configuration Layer
This includes:
- my.cnf file.
- Custom parameters.
- ConfigMap for external configuration.

### 4. Security Layer
This includes:
- root password.
- Business account passwords.
- Secret management.

### 5. Network Layer
This includes:
- Service exposure.
- Access methods within the cluster.
- Whether external access is allowed.

### 6. Lifecycle Layer
This includes:
- Initial initialization.
- Restart and recovery.
- Configuration changes.
- Subsequent upgrades and maintenance.

### Key Points for Operations and Maintenance
MySQL deployment involves a multi-layer combination, rather than simply dealing with a StatefulSet YAML file.

---

## XII. A Basic Illustration of MySQL Object Composition
The following is a simplified illustrative example for educational purposes, to help understand the relationships between these objects.

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
                - name:- A database instance that can continuously store data
- A set of adjustable operating configurations
- A group of sensitive information that is independently managed
- A service entrance that can be stably accessed by business applications

Therefore, the value of deploying MySQL lies not just in learning about a particular database example, but in applying the established stateful application methodology to a tangible and practical object that can be easily understood and implemented.

---

## References
- Kubernetes StatefulSet official documentation
- Kubernetes Persistent Volumes official documentation
- Kubernetes ConfigMap official documentation
- Kubernetes Secret official documentation
- MySQL Official Docker Image documentation

---

## Next Steps
Suggested next topic:

[[11-Practical Deployment of a Single MySQL Instance in Kubernetes: Complete YAML Configuration, Creation Order, and Connectivity Verification]]