# 01-Cloud Migration Tool Systems Overview: Host Migration, Database Migration, Object Storage Migration, and Comparison with Major Cloud Providers

## Document Description
This document outlines the tool systems of major domestic cloud providers in areas such as host migration, database migration, object storage migration, and big data/migration management platforms. It summarizes the common principles, applicable scenarios, and differences among these tools, making it useful for knowledge base development and preparation before interviews.

## Tags
#Cloud Migration #Alibaba Cloud #Tencent Cloud #Huawei Cloud #Baidu Smart Cloud #Host Migration #Database Migration #Object Storage #OSS

---

## I. General Overview

Major domestic cloud providers generally offer comprehensive migration tool suites that cover the following areas:

1. **Host Migration**
   - Used for migrating physical machines, virtual machines, and other cloud hosts to the cloud.
   - Common scenarios include P2V, V2V, and cross-cloud migrations.

2. **Database Migration**
   - Handles migrations between homogeneous or heterogeneous databases, as well as real-time synchronization, disaster recovery, and dual-active setups.
   - Common applications include database migration to the cloud, cross-regional synchronization, and disaster recovery setup.

3. **Object Storage Migration**
   - Allows for migrating object storage, files, and archived data to the cloud.
   - Typical use cases include moving local files or third-party object stores to cloud services.

4. **Big Data / Lakehouse Migration**
   - Designed for migrating platforms like Hive, Spark, Flink, data warehouses, and data lakes.
   - Common scenarios involve migrating big data systems to the cloud or reorganizing lakehouse architectures.

5. **Migration Management Platforms**
   - Provide unified management of migration tasks, projects, resource assessment, and process orchestration.
   - Suitable for large-scale migration projects, batch migrations, and multi-system governance.

---

## II. Alibaba Cloud Migration Tool System

### 1. Host Migration: SMC (Server Migration Center)
**Purpose:** Server migration platform

#### Main Uses
- Migrate IDC physical machines to Alibaba Cloud.
- Move virtualization environments like VMware/KVM to Alibaba Cloud.
- Transfer cloud hosts from other providers to Alibaba Cloud.
- Supports P2V, V2V, and cross-cloud migrations.

#### Core Mechanism
- Collects source system and disk data.
- Performs full data replication.
- Compresses the downtime window through incremental synchronization.
- Starts instances on the target ECS after completion.

#### Applicable Scenarios
- Migrating traditional servers to the cloud.
- IDC-to-cloud migrations.
- Cross-cloud whole-machine migrations.

#### Features
- Mature tool system.
- Comprehensive documentation and ecosystem.
- Suitable for complex whole-machine migration scenarios.

---

### 2. Database Migration: DTS (Data Transmission Service)
**Purpose:** Database migration/synchronization/subscription platform

#### Main Uses
- Migrates databases such as MySQL, PostgreSQL, SQL Server, and Oracle.
- Supports homogeneous and heterogeneous migrations.
- Handles real-time synchronization, disaster recovery, and dual-active data links.

#### Core Mechanism
- Full data migration.
- Incremental synchronization based on log analysis.
- Switches business connections during the cut-over window.

#### Applicable Scenarios
- Database migration to the cloud.
- Cross-regional data synchronization.
- Offsite disaster recovery.
- Data link synchronization.

#### Key Considerations
- Database version compatibility.
- Character set and sorting rules.
- Primary keys, indexes, and constraints.
- Log retention and synchronization latency.
- Final consistency verification.

---

### 3. Object Storage Migration: OSSImport / OSS Toolchain
**Purpose:** Object storage migration tool

#### Main Uses
- Migrates local file systems to OSS.
- Transfers data from third-party object stores to OSS.
- Moves static resources, attachments, and archived files to the cloud.

#### Core Mechanism
- Scans source objects or files.
- Performs full data replication.
- Executes incremental synchronization for live writes.
- Verifies object quantities, sizes, and hash values.
- Switches access domains, CDN, or application configurations after completion.

#### Common Methods
- OSSImport.
- ossutil.
- OSS SDK.
- Custom scripts or batch processing tools.

#### Applicable Scenarios
- Migrating file data to the cloud.
- Transferring data from other cloud object stores to OSS.
- Moving business static resources.

---

### 4. Migration Management Platform: CMH (Cloud Migration Hub)
**Purpose:** Migration project management and orchestration platform

#### Main Functions
- Unified management of migration tasks.
- Tracking multiple migration tools and projects.
- Providing visual migration governance capabilities.

#### Applicable Scenarios
- Multi-system, multi-batch migrations.
- Large-scale projects requiring unified migration management.

---

### 5. Lakehouse / Big Data Migration: LHM (Lakehouse Migration)
**Purpose:** Lakehouse/data platform migration tool

#### Main Uses
- Migrates data platforms such as Hive, Spark,---

### 4. Core Processes of OSS Migration

#### 1) Source Evaluation
- Assess the total amount of files.
- Evaluate the overall capacity.
- Determine the proportion of large files and numerous small files.
- Examine the directory hierarchy and naming rules.
- Verify any requirements for retaining permissions, timestamps, or metadata.

#### 2) Target Preparation
- Create a Bucket.
- Plan the storage type.
- Configure access permissions.
- Set up lifecycle policies.
- Enable cross-domain, acceleration, version control, and other features.

#### 3) Full Migration
- Initially transfer all historical object data to the target.
- This is usually the most time-consuming phase.

#### 4) Incremental Migration
- Synchronize newly added or changed objects after the full migration.
- Commonly used when the business continues to write data online.

#### 5) Verification and Switching
- Verify the number of objects.
- Check the object sizes.
- Validate the hash values of key files.
- Ensure that access paths and application references are correct.
- Switch the download/access address to the target object storage.

#### One-sentence Memory Aid
- Evaluation → Preparation → Full Migration → Incremental Migration → Verification → Switching

---

### 5. Key Points to Consider in OSS Migration

#### Data Consistency
Ensure that before and after migration:
- The number of files remains the same.
- File sizes are consistent.
- The hash values of key files match.

#### Handling Numerous Small Files
Numerous small files can be more problematic than large files for reasons such as:
- Higher number of requests.
- Greater overhead for metadata processing.
- Potential lower migration efficiency.

#### Incremental Synchronization Strategy
If the business continues to write data to object storage, consider:
- The frequency of incremental synchronization.
- The final window before stopping writes.
- Whether dual writing or gradual switching is necessary.

#### Access Address Switching
After migration, handle tasks such as:
- Domain name changes.
- CDN origin-pull adjustments.
- Application configuration modifications.
- Adaptation of permissions and signature methods.

#### Permissions and Metadata
In some cases, it is essential to retain:
- Content-Type.
- Cache-Control.
- File timestamps.
- ACL permissions.
- Custom metadata.

#### One-sentence Memory Aid
- OSS migration involves more than just copying files; it also requires verification, incremental processing, access switching, and management of metadata and permissions.

---

### 6. Challenges in OSS Migration

#### Large Data Volumes
If the total data volume is huge, the migration process will take a long time and require high bandwidth.

#### Numerous Small Files
A large number of small files can reduce migration efficiency and increase the complexity of task management.

#### Continuous Business Writing
If new files are continuously added at the source, it is necessary to design mechanisms for incremental synchronization and gradual switching.

#### Access Address Switching
Migration completion does not mean the end of business operations; smooth switching of application access points must also be addressed.

#### Verification Costs
Post-migration verification can be time-consuming, so it is important to plan accordingly.

---

### 7. How to Answer “How Do I Perform OSS Migration” in an Interview

#### Standard Answer Direction
OSS migration essentially involves the transfer of object data. The process typically begins with evaluating the source data's scale, number of files, file types, and directory structure. Then, a Bucket is created on the target, and permissions and storage policies are set up. During migration, full data transfer is performed first, and if the business continues to write data online, an incremental synchronization mechanism is implemented. After migration, consistency checks are conducted based on the number of objects, file sizes, hash values, etc. Finally, access domain names, CDN origin-pull settings, or application configurations are switched to complete the business transition.

#### One-sentence Version
- The key steps in OSS migration include: full data transfer, incremental synchronization, data verification, and access switching.

---

### 8. Comparing OSS Migration and Database Migration

It is not appropriate to simply say which one is easier; rather, their focus areas differ:

- Database migration emphasizes transactions, consistency, log synchronization, and business write transitions.
- OSS migration focuses on object replication, metadata retention, handling numerous small files, and access address switching.

Generally speaking:
- In terms of principle, OSS migration is more straightforward than database migration.
- However, challenges such as numerous small files, continuous data writing, and permission/cDN adjustments can significantly increase the complexity in both cases.

#### One-sentence Memory Aid
- While OSS migration may seem simpler, its actual implementation involves complexities related to incremental processing, verification, and access switching.

---

## VII. Comparison of Major Cloud Provider Tools

### 1. Host Migration Tools

| Cloud Provider | Tool          | Main Purpose                    | Typical Use Cases                |
|-----------------|---------------|-----------------------------------|--------------------------------------------|
| Alibaba Cloud | SMC            | Complete host migration           | IDC-to-cloud migration, cross-cloud migration |
