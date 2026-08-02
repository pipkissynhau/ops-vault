# Compilation of Cloud Migration and Platform-related Issues: Migration Methods, Mainstream Cloud Vendor Tools, Database and Kubernetes Basics

## Document Description
This document compiles common technical issues encountered in cloud migration/cloud platform/ops positions, covering basic migration methodologies, mainstream cloud vendor migration tool suites, database fundamentals, and common Kubernetes principles. It is suitable for knowledge base development and pre-interview review.

## Tags
#CloudMigration #CloudPlatform #Ops #DevOps #Kubernetes #Redis #MongoDB #Database #InterviewQuestionCompilation

---

## I. Basic Migration Issues

### 1. Differences between Cold Migration and Hot Migration

#### 1) Cold Migration
Cold migration involves stopping the source service first, then performing the migration, and finally starting the service on the target side after completion.

#### 2) Hot Migration
Hot migration means performing full and incremental syncs during the service operation, with the final sync and switch occurring during a dedicated window to minimize service interruptions.

#### 3) Key Differences
**Service Interruption**
- Cold Migration: Obvious service disruption during migration.
- Hot Migration: Service remains online except for a brief shutdown at the end of the switch.

**Consistency Control**
- Cold Migration: Easier to ensure consistency.
- Hot Migration: Requires handling full, incremental, and final consistency checks.

**Implementation Complexity**
- Cold Migration: Relatively simple to implement.
- Hot Migration: Requires more comprehensive synchronization, switching, and recovery designs.

**Applicable Scenarios**
- Cold Migration: Suitable for services that can tolerate downtime windows, testing environments, or off-peak migrations.
- Hot Migration: Critical services, production systems, or scenarios sensitive to downtime.

#### 4) One-sentence Memory Aid
- Cold Migration: Shutdown-based migration.
- Hot Migration: Online sync with final switch.

---

### 2. Phases of a Standard Migration Project

#### 1) Research and Evaluation
- Identify source resources.
- Confirm operating system, architecture, disks, application dependencies, and network connections.
- Determine the migration method (e.g., P2V, V2V, database migration, object migration).

#### 2) Target Environment Preparation
- Prepare cloud hosts, virtual machines, or target databases.
- Set up networks, VPCs, subnets, security groups, routing, and EIPs.
- Ensure target storage, images, and resource specifications are ready.

#### 3) Full Sync
- Initially copy system, disk, or business data. This is often the most time-consuming phase of migration.

#### 4) Incremental Sync
- Capture changes that occur during the migration process. Multiple incremental syncs help reduce the final downtime window.

#### 5) Switch Preparation
- Determine the downtime window.
- Notify relevant parties.
- Prepare switch and recovery plans.
- Confirm data validation methods and scope.

#### 6) Final Switch
- Stop the source service or suspend writes.
- Perform the last incremental sync.
- Start the service on the target side.
- Update access addresses, connection details, DNS, routing, or configurations.

#### 7) Verification and Recovery Assurance
- Verify functionality and data consistency.
- Monitor system performance.
- Maintain a recovery plan in case of issues.

#### 8) One-sentence Memory Aid
- Research → Preparation → Full Sync → Incremental Sync → Switch → Verification → Recovery Assurance

---

### 3. Describing Migration Experience Without Using Specific Cloud Vendor Tools

Even if you haven’t used specific public cloud migration tools, you can still describe your work from a methodological perspective.

#### 1) Key Points to Include
- Source resource identification.
- Network setup.
- Full data synchronization.
- Incremental data syncs.
- Target environment reconfiguration.
- Switch and go-live procedures.
- Data consistency verification.
- Recovery plan preparation.

#### 2) More Technical Expressions
Instead of simply saying:
- Data sync.
- Environment rebuilding.
- Re-deployment,

you could use more precise terms like:
- Conducting a thorough analysis of source resources.
- Establishing a stable migration network connection.
- Performing full and incremental data replication.
- Reconstructing the service environment on the target side.
- Completing the final synchronization and switch within the designated window.
- Verifying data integrity for critical components.
- Preparing a contingency plan for potential issues.

#### 3) One-sentence Memory Aid
- Even without specific tools, the fundamental steps of migration remain the same: full sync, incremental updates, switch, verification, and recovery.

---

## II. Mainstream Cloud Vendor Migration Tool Suites

### 4. Alibaba Cloud Migration Tool Suite

#### 1) SMC (Server Migration Center)
**Purpose**: Server migration tool.

**Main Uses**:
- Migrating IDC physical machines to Alibaba Cloud.
- Moving virtualization environments like VMware/KVM to Alibaba Cloud.
- Transferring servers from other cloud providers to Alibaba Cloud.
- Supports P2V, V2V, and cross-cloud migrations.

**### 10. The Essential Differences Between Host Migration and Database Migration

#### 1) Host Migration
- Moves the operating system, system disk, data disk, and application environment.
- Focuses on the entire system relocation.
- Commonly seen in P2V, V2V, and cross-cloud whole-machine migrations.

#### 2) Database Migration
- Transfers data structures, business data, and incremental logs.
- Concentrates on the data layer migration.
- Places greater emphasis on online migration, log synchronization, and eventual consistency.

#### 3) One-sentence Memory Aid
- Host migration: Relocation of the entire system.
- Database migration: Synchronization and switching of data and logs.

---

## IV. Platform Tool Issues

### 11. What is Alibaba Cloud YunXiao?

YunXiao is a one-stop DevOps platform from Alibaba Cloud that covers the entire process from development collaboration to continuous delivery.

#### Main Capabilities
- Project collaboration
- Code management
- Pipeline construction
- Continuous integration
- Continuous delivery
- Product management
- Development efficiency analysis

#### One-sentence Memory Aid
- YunXiao = Alibaba Cloud's one-stop DevOps platform.

---

### 12. How to Answer “Have You Used YunXiao?”

If you don't have direct experience using it, you can answer from the perspective of its platform role:

- Although I haven't used it extensively in production,
- I am aware of its capabilities and scope.
- Essentially, it is an integrated platform for project collaboration, code hosting, CI/CD, and release processes.

---

## V. Redis-related Issues

### 13. The Differences Between Redis Master-Slave and Cluster

#### 1) Master-Slave
- One master node manages multiple slave nodes.
- Data is replicated from the master to the slaves.
- Main functions: High availability foundation, read-write separation.

#### 2) Cluster
- Multiple master nodes store data in shards.
- Each master node is responsible for a specific set of slots.
- Main functions: Horizontal scaling, high availability, distributed storage.

#### 3) Core Differences
- Master-slave architecture addresses single-machine capacity and write bottlenecks.
- Cluster architecture uses sharding to handle capacity and concurrency issues.
- Master-slave focuses on high availability and read-write separation.
- Cluster emphasizes high availability and horizontal scaling.

#### 4) One-sentence Memory Aid
- Master-slave is about replication and read-write separation.
- Cluster is about sharding and scaling.

---

### 14. What Are Redis Hot Keys?

#### 1) Core Definition
Hot keys are those that are accessed much more frequently than average, causing traffic to concentrate on a single node or shard.

#### 2) Possible Issues
- Increased CPU usage on the node.
- Higher QPS (Queries Per Second).
- Increased response latency.
- Creation of hot spots within shards.
- Potential cache breakdown after key expiration.

#### 3) Common Solutions
- Local caching.
- Multi-level caching.
- Splitting hot keys across different nodes.
- Asynchronous data refreshing.
- Implementing logical expiration mechanisms.
- Controlling concurrent rebuilding processes.

#### 4) One-sentence Memory Aid
- Hot keys are not defined by their size but by their extremely high access frequency.

---

## VI. MongoDB-related Issues

### 15. What Is a MongoDB Sharded Cluster?

#### 1) Core Definition
MongoDB sharded clusters are primarily used for horizontal scaling in scenarios involving large amounts of data.

#### 2) Key Components
- **Shard**: The actual node that stores data.
- **Mongos**: The request router.
- **Config Server**: Stores cluster metadata.

#### 3) Core Principle
- Data is distributed across different shards using a **shard key**.
- Data is divided into chunks, and the balancer automatically manages data distribution in the background.

#### 4) Differences from Replicas Sets
- **Replicas Sets**: Focus on high availability.
- **Sharded Clusters**: Aim at horizontal scaling.
- In production environments, each shard typically also serves as a replicas set.

#### 5) One-sentence Memory Aid
- Replicas Sets = High availability.
- Sharded Clusters = Horizontal scaling.
- Key components include shards, mongos, and config server.

---

## VII. Kubernetes-related Issues

### 16. What Is the Role of kube-proxy?

The main function of kube-proxy is to forward traffic to Service instances by directing requests intended for Services to their corresponding Pods.

#### 1-sentence Memory Aid
- kube-proxy is responsible for forwarding traffic from Services to Pods.

---

### 17. What Are the Differences Between the Two Modes of kube-proxy?

#### 1) iptables Mode
- Uses a large number of iptables rules to perform traffic forwarding.
- Simple implementation.
- However, as the number of Services and Endpoints increases, the number of rules