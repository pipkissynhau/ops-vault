# Cloud Migration and Platform-Related Issues: Migration Methods, Mainstream Cloud Vendor Tools, Databases, and Kubernetes Basics

## Document Description
Compile common technical issues in cloud migration/cloud platform/operations roles, covering migration fundamentals, mainstream cloud vendor migration tool systems, database basics, and common Kubernetes principles, suitable for knowledge base documentation and pre-interview review.

## Tags
#CloudMigration #CloudPlatform #Transport #DevOps #Kubernetes #Redis #MongoDB #Database #InterviewSorting

---

## I. Migration Fundamentals

### 1. Differences Between Cold Migration and Hot Migration

#### 1) Cold Migration
Cold migration refers to stopping the source business first, then executing migration, and starting the business on the target side after migration completion.

#### 2) Hot Migration
Hot migration refers to performing full synchronization and incremental synchronization during business operation, and executing the final synchronization and switch during the cut-over window, aiming to minimize business interruption time.

#### 3) Core Differences
**Business Interruption**
- Cold Migration: Obvious business interruption during migration
- Hot Migration: Most migration process is online, with only short downtime during final cut-over

**Consistency Control**
- Cold Migration: Easier to ensure consistency
- Hot Migration: Need to handle full, incremental, and final consistency verification

**Implementation Complexity**
- Cold Migration: Relatively simple to implement
- Hot Migration: Requires more comprehensive synchronization, cut-over, and rollback design

**Applicable Scenarios**
- Cold Migration: Business that can tolerate downtime window, test environments, low-peak migration
- Hot Migration: Core business, production systems, scenarios sensitive to downtime

#### 4) One-Sentence Memory
- Cold Migration: Stop and Migrate
- Hot Migration: Online Synchronization, Final Cut-Over

---

### 2. What Are the Typical Stages of a Standard Migration Project

#### 1) Survey and Evaluation
- Inventory source resources
- Confirm operating system, architecture, disk, application dependencies, network dependencies
- Confirm migration method: P2V, V2V, database migration, object migration, etc.

#### 2) Target Environment Preparation
- Prepare cloud hosts, virtual machines, or target databases
- Prepare network, VPC, subnet, security group, routing, EIP
- Prepare target storage, image, target resource specifications

#### 3) Full Synchronization
- Initial replication of system, disk, or business data
- This is one of the longest stages in migration

#### 4) Incremental Synchronization
- Capture changes in data during migration
- Compress final downtime window through multiple incremental synchronizations

#### 5) Cut-Over Preparation
- Determine downtime window
- Notify business parties
- Prepare cut-over and rollback plans
- Confirm data verification methods and scope

#### 6) Final Cut-Over
- Stop source business or pause writes
- Execute final incremental synchronization
- Start business on target side
- Modify access entry, connection address, DNS, routing, or configuration

#### 7) Verification and Rollback Assurance
- Perform functional verification
- Perform data consistency verification
- Observe operational status
- Reserve rollback window

#### 8) One-Sentence Memory
- Survey → Preparation → Full → Incremental → Cut-Over → Verification → Rollback Assurance

---

### 3. How to Describe Migration Experience Without Using Cloud Vendor Tools Directly

Even without directly using public cloud standard migration tools, you can describe work content from the perspective of migration methodology.

#### 1) Core Content That Can Be Described
- Source resource inventory
- Network connectivity
- Full synchronization
- Incremental synchronization
- Target environment reconstruction
- Cut-over deployment
- Data consistency verification
- Rollback plan

#### 2) More Engineering Expression
Avoid only saying:
- Data synchronization
- Environment reconstruction
- Redeployment

Better expressions:
- Complete source and target resource inventory
- Establish migration chain and network connectivity
- Execute full and incremental synchronization
- Complete business reconstruction on target environment
- Complete final synchronization and switch during cut-over window
- Perform consistency verification on critical data
- Reserve rollback plan

#### 3) One-Sentence Memory
- Not using a specific tool doesn't mean no migration experience; migration essence remains full, incremental, cut-over, verification, and rollback

---

## II. Mainstream Cloud Vendor Migration Tool Systems

### 4. Alibaba Cloud Migration Tool System

#### 1) SMC (Server Migration Center)
**Positioning: Server Migration Tool**

**Main Uses**
- Migrate IDC physical machines to Alibaba Cloud
- Migrate VMware/KVM virtualization environments to Alibaba Cloud
- Migrate cloud hosts from other cloud vendors to Alibaba Cloud
- Supports P2V, V2V, cross-cloud migration

**Core Approach**
- Collect system and disk data from source
- Execute full replication first
- Execute incremental synchronization next
- Complete target ECS startup during cut-over window

**Applicable Scenarios**
- Traditional server cloud migration
- Whole machine migration
- Cross-cloud host migration

**Focus Points**
- Source system compatibility
- Disk size and target specifications
- Network bandwidth
- Final downtime window

---

#### 2) DTS (Data Transmission Service)
**Positioning: Database Migration / Data Synchronization / Data Subscription**

**Main Uses**
- Migrate MySQL, PostgreSQL, SQL Server, Oracle, etc. databases
- Homogeneous migration
- Heterogeneous migration
- Real-time synchronization
- Disaster recovery and active-active scenarios

**Core Approach**
- Full data migration
- Incremental synchronization based on logs
- Complete business connection switch during cut-over window

**Applicable Scenarios**
- Database cloud migration
- Cross-regional synchronization
- Disaster recovery construction
- Data synchronization chain in active-active architecture

**Focus Points**
- Database version compatibility
- Character set, collation
- Primary key, index, constraints
- Log retention and synchronization latency
- Final consistency verification

---

#### 3) CMH (Cloud Migration Hub)
**Positioning: Migration Management Platform**

**Main Functions**
- Unified management of migration projects
- Management and orchestration of multiple migration tasks
- Track migration process and status
- Provide visual migration management capabilities

**Applicable Scenarios**
- Batch migration of multiple systems and businesses
- Large-scale migration projects requiring unified management

---

#### 4) LHM (Lakehouse Migration)
**Positioning: Lakehouse / Data Platform Migration Tool**

**Main Uses**
- Migrate Hive, Spark, Flink, etc. big data platforms
- Migrate data lakes, data warehouses, lakehouse architecture

**Applicable Scenarios**
- Big data platform cloud migration
- Lakehouse architecture migration
- Data platform reconstruction or relocation

---

### 5. Tencent Cloud Migration Tool System

#### 1) Server Online Migration / Migration Center
**Positioning: Host Migration Tool**

**Main Uses**
- Migrate physical machines to CVM
- Migrate virtual machines to Tencent Cloud
- Migrate cloud hosts from other cloud vendors to Tencent Cloud

**Core Approach**
- Collect system and disk data from source
- Perform full synchronization
- Compress downtime window through incremental synchronization
- Complete cut-over and start target instance

**Main Features**
- Supports one-click migration via console
- Supports client-based import of migration sources
- Suitable for rapid cloud migration and medium-scale migrations

---

#### 2) DTS
**Positioning: Database Migration / Real-Time Synchronization Tool**

**Main Uses**
- Online database migration
- Real-time synchronization
-Alien. disaster recovery
- Data link synchronization

**Core Approach**
- Full data migration
- Incremental log synchronization
- Final consistency confirmation during cut-over window

---

#### 3) DBbridge
**Positioning: Privateized Database Migration and Synchronization Solution**

**Main Uses**
- Private deployment data migration or synchronization
- Homogeneous and heterogeneous database migration

**Notes**
- More suitable for private environments, complex network environments, or enterprise-level database migration scenarios
- Knowing its positioning is sufficient for interviews, no need to elaborate excessively

---

### 6. Huawei Cloud Migration Tool System /think

#### 1I'm not sure.SMSI'm sorry.Host Migration ServiceI'm not sure.
**Purpose**: Host whole machine migration tool

**Main Uses**
- P2V: Migrate physical machine to cloud host
- V2V: Migrate virtual machine to cloud host
- Cross-cloud migration to Huawei Cloud
- Whole system migration of traditional servers

**Core Approach**
- Install Agent or collection component on source
- Execute full system and disk data replication
- Compress downtime via incremental synchronization
- Final cutover and start target host

**Applicable Scenarios**
- Migrate traditional business to cloud
- Batch migration of enterprise hosts
- System-level migration projects

**Understanding of Rainbow**
- Rainbow can be understood as the migration solution or framework within Huawei's ecosystem
- In public product expressions, it's safer to directly refer to SMS
- Core migration methodology remains: full + incremental + cutover

---

#### 2I'm not sure.DRSI'm sorry.Data Replication ServiceI'm not sure.
**Purpose**: Database migration / real-time synchronization tool

**Main Uses**
- Online database migration
- Real-time synchronization
- Disaster recovery construction
- Data synchronization related to dual-live

**Core Approach**
- Full migration
- Incremental log synchronization
- Final business connection switch

---

#### 3I'm not sure.OMSI'm sorry.Object Migration ServiceI'm not sure.
**Purpose**: Object storage migration tool

**Main Uses**
- Migrate object storage or object-like data to Huawei Cloud object storage
- Suitable for file, object, and archive-type data migration

---

#### 4I'm not sure.CDMI'm sorry.Data Integration / Big Data Migration DirectionI'm not sure.
**Purpose**: Big data / data integration / data migration tool

**Main Uses**
- Migrate big data platforms
- Data synchronization and integration
- Data relocation

---

### 7. Baidu Smart Cloud Migration Toolset

#### 1I'm not sure.Server Migration Center
**Purpose**: Host migration tool

**Main Uses**
- Migrate physical machines, virtual machines to Baidu Cloud
- Cross-environment host migration

**Core Approach**
- Install Agent on source
- Execute full synchronization
- Perform multiple incremental synchronizations
- Final cutover and start target instance

---

#### 2I'm not sure.DTS
**Purpose**: Database migration / synchronization tool

**Main Uses**
- Migrate MySQL, PostgreSQL, SQL Server, Oracle, etc. databases
- Real-time synchronization
- Disaster recovery scenarios

---

#### 3I'm not sure.CMCI'm sorry.Cloud Migration CenterI'm not sure.
**Purpose**: One-stop migration management platform

**Main Functions**
- Manage migration projects
- Provide capabilities for research, planning, resource preparation, and verification
- Suitable for unified governance of complex migration projects

---

## Three. Common Principles of Migration Tools

### 8. Are the underlying approaches of migration tools from different cloud providers similar?

Most migration tools have similar methodologies, differing mainly in product packaging and ecosystem integration.

#### 1I'm not sure.Commonalities
**Source Collection**
- Install Agent
- Collect OS, disk, application, or database information

**Full Synchronization**
- First replicate base data to target

**Incremental Synchronization**
- Continuously synchronize data changed during migration
- Compress final downtime as much as possible

**Final Cutover**
- Stop business or pause writes
- Execute final synchronization
- Start business on target

**Validation and Rollback**
- Function verification
- Data consistency check
- Reserve rollback window

#### 2I'm not sure.Differences
Main differences between cloud providers generally include:
- Product packaging degree
- Supported data source types
- Support for complex networks and private environments
- Visualization level
- Integration with native cloud ecosystem

#### 3I'm not sure.One-sentence Memory
- Tool names differ, but migration methodologies are highly similar; differences mainly lie in product capability boundaries and cloud ecosystem integration

---

### 9. Why can't we simply say "migration is seconds-level"?

#### 1I'm not sure.More accurate understanding
- **Synchronization phase**: Near real-time via multiple incremental synchronizations
- **Final switch phase**: Usually still requires short downtime
- Therefore, we cannot simply say "seamless seconds-level migration"

#### 2I'm not sure.Common expressions
More cautious phrasing includes:
- Synchronization phase approaches real-time
- Cutover phase typically involves brief interruption
- Actual business impact mainly occurs during final cutover window

#### 3I'm not sure.One-sentence Memory
- Synchronization can be fast, but cutover usually still requires brief downtime

---

### 10. Essential Differences Between Host Migration and Database Migration

#### 1I'm not sure.Host Migration
- Migrate OS, system disk, data disk, application environment
- More focused on whole machine system relocation
- Common in P2V, V2V, cross-cloud whole machine migration

#### 2I'm not sure.Database Migration
- Migrate data structure, business data, incremental logs
- More focused on data layer migration
- Emphasizes online migration, log synchronization, and final consistency

#### 3I'm not sure.One-sentence Memory
- Host migration: Whole machine relocation
- Database migration: Data and log synchronization switch

---

## Four. Platform Tool Issues

### 11. What is Alibaba Cloud's CloudEffort?

CloudEffort is Alibaba Cloud's one-stop DevOps platform covering the entire process from R&D collaboration to continuous delivery.

#### Main Capabilities
- Project collaboration
- Code management
- Pipeline construction
- Continuous integration
- Continuous delivery
- Artifact management
- R&D efficiency analysis

#### One-sentence Memory
- CloudEffort = Alibaba Cloud's one-stop DevOps platform

---

### 12. How to answer "Have you used CloudEffort?"

If no direct experience, answer from platform positioning perspective:

- Though not deeply used in production
- But understand its capability boundaries and positioning
- Essentially an integrated platform for project collaboration, code hosting, CI/CD, and release processes

---

## Five. Redis-related Issues

### 13. Difference Between Redis Master-Slave and Cluster

#### 1I'm not sure.Master-Slave
- One master node with multiple slave nodes
- Data replicated from master to slaves
- Main purpose: High availability foundation, read-write separation

#### 2I'm not sure.Cluster
- Multiple master nodes with sharded storage
- Each master node responsible for part of slots
- Main purpose: Horizontal scaling, high availability, distributed storage

#### 3I'm not sure.Core Differences
- Master-slave doesn't solve single-machine capacity and write bottlenecks
- Cluster resolves capacity and concurrency issues via sharding
- Master-slave focuses on high availability and read-write separation
- Cluster focuses on high availability plus horizontal scaling

#### 4I'm not sure.One-sentence Memory
- Master-slave focuses on replication and read-write separation
- Cluster focuses on sharding and scaling

---

### 14. What is a Redis Hot Key?

#### 1I'm not sure.Core Definition
A hot key is a key with access frequency far exceeding the average, causing traffic concentration on a single node or shard.

#### 2I'm not sure.Potential Issues
- Node CPU elevation
- QPS elevation
- Response latency increase
- Hot shard
- Cache penetration after expiration

#### 3I'm not sure.Common Handling Methods
- Local caching
- Multi-level caching
- Hot key splitting
- Asynchronous refresh
- Logical expiration
- Concurrent rebuild control

#### 4I'm not sure.One-sentence Memory
- Hot key isn't about key size, but rather concentrated access

---

## Six. MongoDB-related Issues

### 15. What is MongoDB Sharding Cluster?

#### 1I'm not sure.Core Definition
MongoDB sharding cluster is primarily used for horizontal scaling in large data scenarios.

#### 2I'm not sure.Core Components
- **shard**: Actual data storage node
- **mongos**: Request router
- **config server**: Stores shard metadata

#### 3I'm not sure.Core Principles
- Distribute data across different shards via **shard key**
- Split data into chunks
- Balancer performs data balancing in the background

#### 4) Differences with Replica Sets
- **Replica Set**: Mainly solves high availability
- **Sharded Cluster**: Mainly solves horizontal scaling
- In production, each shard is typically itself a replica set

#### 5) One-sentence Memory
- Replica Set = High Availability
- Sharded Cluster = Horizontal Scaling
- Components = shard + mongos + config server

---

## Seven, Kubernetes Related Questions

### 16. What is the role of kube-proxy

kube-proxy's main role is to provide traffic forwarding capabilities for Service, forwarding requests accessing Service to backend Pods.

#### One-sentence Memory
- kube-proxy is responsible for traffic forwarding from Service to Pod

---

### 17. What are the differences between the two modes of kube-proxy

#### 1) iptables Mode
- Completes forwarding by maintaining a large number of iptables rules
- Simple implementation
- When Service, Endpoint numbers increase, rules grow, matching efficiency decreases

#### 2) IPVS Mode
- Utilizes Linux kernel's four-layer load balancing capabilities
- Maintains Service to backend Pod mapping relations in kernel
- Better performance and scalability, more suitable for large-scale scenarios

#### 3) More accurate understanding
It's not simply "more nodes" causing slowness, but:
- Cluster scale grows
- Service / Pod / Endpoint numbers increase
- Rule count per node rises
- iptables matching chains become longer
- Therefore performance degradation becomes more noticeable

#### 4) One-sentence Memory
- iptables = Rule chain forwarding
- IPVS = Kernel-level four-layer load balancing

---

## Eight, Related Topics That Can Be Extended

### 18. Migration direction can be further supplemented
- P2V / V2V / Cross-cloud migration
- Whole machine migration vs Database migration
- Incremental synchronization mechanism
- Cutover and rollback design
- Network migration and application dependencyCombination

### 19. Database direction can be further supplemented
- MySQL Master-Slave Replication
- binlog
- Redis Sentinel
- Redis Persistence
- MongoDB Replica Set

### 20. Kubernetes direction can be further supplemented
- Service / Endpoints / NodePort
- CoreDNS
- CNI
- Ingress
- kube-proxy forwarding chain

---

## Nine, Keyword Mnemonics

- Cold migration / Hot migration
- Full synchronization / Incremental synchronization / Cutover / Rollback
- SMC / DTS / CMH / LHM
- SMS / DRS / OMS / CDM
- CMC / Server Migration Center
- CloudEfficiency
- Redis Master-Slave / Cluster / Hot key
- MongoDB Sharded Cluster
- kube-proxy
- iptables / IPVS