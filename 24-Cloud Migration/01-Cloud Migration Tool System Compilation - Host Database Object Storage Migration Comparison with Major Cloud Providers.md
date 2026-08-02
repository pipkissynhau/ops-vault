# 01-Cloud Migration Toolset Organization: Host Migration, Database Migration, Object Storage Migration, and Comparison with Main Cloud Providers

## Document Summary
Organize the migration toolsets of domestic mainstream cloud providers in areas such as host migration, database migration, object storage migration, and big data/migration management platforms, and summarize the common principles, applicable scenarios, and differences of various migration tools. Suitable for knowledge baseDeposition and interview preparation.

## Tags
#CloudMigration #Alexander! #It'sACloud. #♪TheClouds♪ #100DegreesOfIntelligentCloud #HostMigration #DatabaseMigration #ObjectStorage #OSS

---

## I. Overview

Domestic mainstream cloud providers generally offer relatively complete migration toolsets, usually covering the following capabilities:

1. **Host Migration**
   - Used for migrating physical servers, virtual machines, and cloud hosts from other cloud platforms to the cloud
   - Common scenarios: P2V, V2V, cross-cloud migration

2. **Database Migration**
   - Used for homogeneous or heterogeneous database migration, real-time synchronization, disaster recovery, and active-active configurations
   - Common scenarios: Database cloud migration, cross-regional synchronization, disaster recovery construction

3. **Object Storage Migration**
   - Used for migrating object storage, files, and archival data to the cloud
   - Common scenarios: Local file migration, third-party object storage migration

4. **Big Data / Lakehouse Migration**
   - Used for migrating Hive, Spark, Flink, data warehouses, and data lake platforms
   - Common scenarios: Big data platform cloud migration, lakehouse architecture migration

5. **Migration Management Platform**
   - Used for unified management of migration tasks, migration projects, resource assessment, and workflow orchestration
   - Common scenarios: Large-scale migration projects, batch migration, multi-system unified governance

---

## II. Alibaba Cloud Migration Toolset

### 1. Host Migration: SMC (Server Migration Center)
**Positioning: Server Migration Platform**

#### Main Purposes
- Migrate IDC physical servers to Alibaba Cloud
- Migrate VMware/KVM virtualization environments to Alibaba Cloud
- Migrate cloud hosts from other cloud providers to Alibaba Cloud
- Supports P2V, V2V, cross-cloud migration

#### Core Approach
- Collect source system and disk data
- Execute full replication
- Compress final downtime window through incremental synchronization
- Start instance on target ECS after final cutover

#### Applicable Scenarios
- Traditional server full migration to cloud
- IDC migration to cloud
- Cross-cloud full migration

#### Features
- Mature toolset
- Complete documentation and ecosystem
- Suitable for complex full migration scenarios

---

### 2. Database Migration: DTS (Data Transmission Service)
**Positioning: Database Migration/Synchronization/Subscription Platform**

#### Main Purposes
- Migrate MySQL, PostgreSQL, SQL Server, Oracle, etc.
- Homogeneous and heterogeneous database migration
- Real-time synchronization, disaster recovery, and active-active data pipelines

#### Core Approach
- Full migration
- Incremental synchronization based on log parsing
- Switch business connections during final cutover window

#### Applicable Scenarios
- Database cloud migration
- Cross-regional synchronization
-Alien. disaster recovery
- Data pipeline synchronization

#### Focus Areas
- Database version compatibility
- Character set, collation
- Primary keys, indexes, constraints
- Log retention and synchronization latency
- Final consistency verification

---

### 3. Object Storage Migration: OSSImport / OSS Toolchain
**Positioning: Object Storage Migration Tool**

#### Main Purposes
- Migrate local file systems to OSS
- Migrate third-party object storage to OSS
- Migrate static resources, attachments, archival files to cloud
- Migrate images, videos, logs, and backup data

#### Core Approach
- Scan source objects or files
- Execute full migration
- Perform incremental synchronization for online write scenarios
- Complete object count, size, and hash value verification
- Switch access domain names, CDN, or application configurations finally

#### Common Methods
- OSSImport
- ossutil
- OSS SDK
- Self-developed scripts or batch processing tools

#### Applicable Scenarios
- File-type data cloud migration
- Migrating object storage from other clouds to OSS
- Business static resource migration

---

### 4. Migration Management Platform: CMH (Cloud Migration Hub)
**Positioning: Migration Project Management and Orchestration Platform**

#### Main Functions
- Unified management of migration tasks
- Track multiple migration tools and projects
- Provide visual migration governance capabilities

#### Applicable Scenarios
- Multi-system, multi-batch migration
- Large projects requiring unified migration governance

---

### 5. Lakehouse/Big Data Migration: LHM (Lakehouse Migration)
**Positioning: Lakehouse/Data Platform Migration Tool**

#### Main Purposes
- Migrate Hive, Spark, Flink, and other data platforms
- Migrate data lakes, data warehouses, and lakehouse architecture

#### Applicable Scenarios
- Big data platform cloud migration
- Lakehouse architecture migration
- Data platform reconstruction or relocation

---

## III. Tencent Cloud Migration Toolset

### 1. Host Migration: Server Online Migration / Migration Center
**Positioning: Host Migration Tool**

#### Main Purposes
- Migrate physical servers to CVM
- Migrate virtual machines to Tencent Cloud
- Migrate cloud hosts from other clouds to Tencent Cloud

#### Core Approach
- Collect source system and disk data
- Full data transfer
- Incremental synchronization
- Start instance on target end after final cutover

#### Main Features
- Supports one-click migration via console
- Supports importing migration sources via client
- Focuses on quick, lightweight experience

#### Applicable Scenarios
- Medium-scale full migration
- Rapid cloud migration projects
- Cross-environment host migration

---

### 2. Database Migration: DTS
**Positioning: Database Migration/Data Synchronization Platform**

#### Main Purposes
- Online database migration
- Real-time synchronization
-Alien. disaster recovery
- Data pipeline replication

#### Core Approach
- Full migration
- Incremental log synchronization
- Switch business connections finally

---

### 3. Object Storage Migration: COS Migration / COS Toolchain
**Positioning: Object Storage Migration Tool**

#### Main Purposes
- Migrate local files to COS
- Migrate third-party object storage to COS
- Migrate massive static resources and object files

#### Core Approach
- Scan source objects
- Full replication
- Incremental synchronization
- Object verification
- Switch domain names, access entry, or CDN configurations

#### Common Methods
- COS Migration Tool
- COSCMD
- COS SDK
- Self-developed synchronization scripts

---

### 4. Private Database Migration: DBbridge
**Positioning: Private Database Migration and Synchronization Solution**

#### Main Purposes
- Migrate and synchronize databases in private environments
- Homogeneous or heterogeneous database migration

#### Notes
- More suitable for private environments, complex network environments, or enterprise-level database migration scenarios
- Knowing its positioning is sufficient, no need for detailed expansion

---

## IV. Huawei Cloud Migration Toolset

### 1. Host Migration: SMS (Server Migration Service)
**Positioning: Full System Migration Tool**

#### Main Purposes
- P2V: Migrate physical servers to cloud hosts
- V2V: Migrate virtual machines to cloud hosts
- Cross-cloud migration to Huawei Cloud
- Traditional server system-level migration

#### Core Approach
- Install Agent on source end
- Collect system and disk data
- Perform full replication
- Compress downtime window through incremental synchronization
- Start target host after final cutover

#### Applicable Scenarios
- Enterprise-level full migration to cloud
- IDC server relocation
- Batch host migration

#### Notes on Rainbow
- Rainbow is more appropriately understood as the migration solution or migration framework terminology used within Huawei's system
- In public product expressions, the safer approach is to directly refer to SMS
- The core migration methodology remains: full + incremental + cutover

---

### 2. Database Migration: DRS (Data Replication Service)
**Positioning: Database migration / real-time synchronization tool**

#### Main Uses
- Online database migration
- Real-time synchronization
- Disaster recovery
- Dual-live related scenarios

#### Core Approach
- Full migration
- Incremental log synchronization
- Final business connection switch

---

### 3. Object Storage Migration: OMS (Object Migration Service)
**Positioning: Object storage migration tool**

#### Main Uses
- Object storage data migration
- Migration of file and archive-type data to Huawei Cloud object storage

#### Core Approach
- Scan source objects
- Execute full replication
- Perform incremental synchronization for online changes
- Complete object consistency verification
- Switch access entry

---

### 4. Big Data / Data Integration Migration: CDM
**Positioning: Data integration / big data migration tool**

#### Main Uses
- Data integration
- Big data platform migration
- Data migration and transformation

---

## Five. Baidu Smart Cloud Migration Toolset

### 1. Host Migration: Server Migration Center
**Positioning: Host migration tool**

#### Main Uses
- Migrate physical servers to Baidu Cloud
- Migrate virtual machines to Baidu Cloud
- Cross-environment host migration

#### Core Approach
- Install Agent on source
- Full synchronization
- Incremental synchronization
- Final cutover and start target instance

---

### 2. Database Migration: DTS
**Positioning: Database migration / synchronization tool**

#### Main Uses
- Migration of MySQL, PostgreSQL, SQL Server, Oracle, etc. databases
- Real-time synchronization
- Disaster recovery scenarios

---

### 3. Object Storage Migration: Object Storage Toolchain / Platform Capabilities
**Positioning: Object storage migration capabilities**

#### Main Uses
- Migrate local files, object data to Baidu Object Storage
- Migration of files, images, static resources, archive-type data

#### Notes
- Typically used in conjunction with object storage toolchain, SDK or migration platform
- In interviews, it can be understood as "object migration capabilities", no need to memorize excessive product details

---

### 4. Migration Management Platform: CMC (Cloud Migration Center)
**Positioning: One-stop migration management platform**

#### Main Functions
- Manage migration projects
- Support survey, planning, resource preparation, verification processes
- Suitable for unified governance of complex migration projects

---

## Six. Object Storage Migration Issues

### 1. What is OSS Migration

OSS migration essentially involves migrating data from local file systems, third-party object storage, or other cloud vendors' object storage to the target object storage platform, such as migrating to Alibaba Cloud OSS.

#### Common Scenarios
- Migrate local NAS / file servers to OSS
- Migrate other cloud vendors' object storage to OSS
- Migrate IDC static resources to cloud
- Migrate massive images, videos, attachments, backup files to object storage

#### One-sentence Memory
- OSS migration essentially involves object data migration, not whole machine migration, nor database migration

---

### 2. Differences Between OSS Migration and Host/Database Migration

#### Differences with Host Migration
- Host migration migrates OS, system disk, data disk and application environment
- OSS migration migrates object data itself, such as images, videos, attachments, archive files

#### Differences with Database Migration
- Database migration emphasizes table structure, business data, incremental logs and consistency switch
- OSS migration focuses more on file/object replication, directory mapping, verification and incremental synchronization

#### One-sentence Memory
- Host migration: Migrate system
- Database migration: Migrate data and logs
- OSS migration: Migrate objects and files

---

### 3. Common Tools and Methods for OSS Migration

#### Alibaba Cloud
- OSSImport
- ossutil
- OSS SDK
- Self-developed scripts

#### Tencent Cloud
- COS Migration Tool
- COSCMD
- COS SDK

#### Huawei Cloud
- OMS

#### Baidu Smart Cloud
- Object Storage Toolchain
- SDK
- Migration platform capabilities

#### One-sentence Memory
- Object storage migration generally has official migration tools + SDK/command-line tools

---

### 4. Core Process of OSS Migration

#### 1) Source Evaluation
- Evaluate total file count
- Evaluate total capacity
- Evaluate ratio of large files and massive small files
- Evaluate directory hierarchy and naming rules
- Confirm requirements for permissions, timestamps, and metadata retention

#### 2) Target Preparation
- Create Bucket
- Plan storage type
- Configure access permissions
- Configure lifecycle policies
- Configure cross-domain, acceleration, version control capabilities

#### 3) Full Migration
- Migrate all historical object data to target
- Usually the most time-consuming phase

#### 4) Incremental Migration
- Synchronize newly added or changed objects after full migration
- Common in scenarios with ongoing business writes

#### 5) Verification and Switch
- Verify object count
- Verify object size
- Verify critical file hash values
- Validate access paths and application references
- Switch download/access address to target object storage

#### One-sentence Memory
- Evaluation → Preparation → Full → Incremental → Verification → Switch

---

### 5. What to Focus on in OSS Migration

#### Data Consistency
Confirm consistency before and after migration:
- File count consistency
- File size consistency
- Critical file hash value consistency

#### Massive Small Files Issue
Massive small files are typically more problematic than large files, due to:
- High request frequency
- High metadata processing overhead
- Potentially lower migration efficiency

#### Incremental Synchronization Strategy
If business continues to write to object storage, consider:
- Incremental synchronization frequency
- Final stop-write window
- Need for dual-write or gradual switch

#### Access Address Switch
After migration, typically handle:
- Domain name switch
- CDN source adjustment
- Application configuration modification
- Permission and signature method adaptation

#### Permissions and Metadata
Some scenarios require retention of:
- Content-Type
- Cache-Control
- File timestamp
- ACL permissions
- Custom metadata

#### One-sentence Memory
- OSS migration isn't just file copying, but also considers verification, incremental, access switching, metadata and permissions

---

### 6. Challenges in OSS Migration

#### Large Data Volume
If total volume is huge, migration duration will be long, requiring high bandwidth.

#### Massive Small Files
Massive small files can reduce migration efficiency and increase task management complexity.

#### Ongoing Business Writes
If source continues to add files, need to design incremental synchronization and final switch plan.

#### Access Path Switch
Migration completion doesn't mean business ends, need to consider smooth access entry switching.

#### Verification Cost
Post-migration verification may take long time, need to plan verification strategy in advance.

---

### 7. How to Answer "How to Do OSS Migration" in Interviews

#### Standard Answer Direction
OSS migration essentially belongs to object data migration. Typically, first evaluate source data scale, file count, file types and directory structure, then create Bucket on target and complete permission and storage strategy preparation. During migration implementation, usually start with full migration, and if business is still writing, need to design incremental synchronization mechanism. After migration, verify consistency from object count, file size, hash values, etc., and finally switch access domain name, CDN source or application configuration to complete business switch.

#### One-Sentence Version
- The key to OSS migration is: full migration, incremental synchronization, data verification, and access switching

---

### 8. Comparing OSS Migration with Database Migration: Which is Simpler

You cannot simply say which is simpler; the focus differs:

- Database migration emphasizes transactions, consistency, log synchronization, and business write switching
- OSS migration emphasizes object replication, metadata preservation, handling massive small files, and access address switching

Generally:
- From a principle perspective, OSS migration is more intuitive than database migration
- However, if encountering massive small files, continuous writing, permissions, and CDN switching, complexity will significantly increase

#### One-Sentence Memory
- OSS migration seems simple, but the true complexity lies in incremental synchronization, verification, and access switching

---

## Seven. Comparison of Main Cloud Provider Tools

### 1. Host Migration Comparison

| Cloud Provider | Tool | Main Focus | Typical Scenarios |
|---|---|---|---|
| Alibaba Cloud | SMC | Whole Machine Migration | IDC to Cloud, Cross-Cloud Migration |
| Tencent Cloud | Server Online Migration / Migration Center | Whole Machine Migration | Quick Cloud Migration, Small-to-Medium Scale Migration |
| Huawei Cloud | SMS | Whole Machine Migration | Enterprise-Level Host Migration, Batch Migration |
| Baidu Intelligent Cloud | Server Migration Center | Whole Machine Migration | Cross-Environment Host Relocation |

### 2. Database Migration Comparison

| Cloud Provider | Tool | Main Focus | Typical Scenarios |
|---|---|---|---|
| Alibaba Cloud | DTS | Database Migration/Synchronization | Cloud Migration, Disaster Recovery, Active-Active |
| Tencent Cloud | DTS | Database Migration/Synchronization | Online Migration, Synchronization |
| Huawei Cloud | DRS | Database Migration/Synchronization | Online Migration, Disaster Recovery |
| Baidu Intelligent Cloud | DTS | Database Migration/Synchronization | Cloud Migration, Real-Time Synchronization |

### 3. Object Storage Migration Comparison

| Cloud Provider | Tool/Capability | Main Focus | Typical Scenarios |
|---|---|---|---|
| Alibaba Cloud | OSSImport / ossutil / OSS SDK | Object Migration | Local Files, Third-Party Object Storage to OSS Migration |
| Tencent Cloud | COS Migration / COSCMD / COS SDK | Object Migration | Local Files, Object Data to COS Migration |
| Huawei Cloud | OMS | Object Migration | Object, File, Archive Data Migration |
| Baidu Intelligent Cloud | Object Storage Toolchain / Platform Capabilities | Object Migration | File and Object Data Relocation |

### 4. Management Platform Comparison

| Cloud Provider | Tool | Main Focus |
|---|---|---|
| Alibaba Cloud | CMH | Migration Project Management and Orchestration |
| Huawei Cloud | MgC (Migration Center Perspective) | Migration Process Management and Governance |
| Baidu Intelligent Cloud | CMC | One-Stop Migration Management |
| Tencent Cloud | Migration Center / Platform Entry | Productized Migration Entry |

---

## Eight. Common Principles and Key Understandings

### 1. Common Methodology of Main Migration Tools
Most migration tools essentially follow a similar process:

1. **Source End Collection**
   - Install Agent
   - Retrieve system, disk, application, database, or object information

2. **Full Synchronization**
   - Initial replication of base data

3. **Incremental Synchronization**
   - Continuous synchronization of changed data
   - Minimize final downtime window

4. **Final Cutover**
   - Stop business or pause writes
   - Execute final synchronization
   - Start business or switch access entry on target

5. **Verification and Rollback**
   - Conduct functional verification
   - Validate data consistency
   - Maintain rollback window

---

### 2. Differences Between Host Migration, Database Migration, and Object Storage Migration

#### Host Migration
- Migrates OS, system disk, data disk, application environment
- Closer to whole machine relocation

#### Database Migration
- Migrates table structure, business data, incremental logs
- Emphasizes online migration and consistency switching

#### Object Storage Migration
- Migrates images, videos, attachments, archive files, etc.
- Emphasizes object replication, metadata preservation, verification, and access switching

#### One-Sentence Memory
- Host Migration: Migrate System
- Database Migration: Migrate Data and Logs
- Object Migration: Migrate Objects and Files

---

### 3. Why You Can't Simply Say "Migration is Seconds"

A more accurate statement should be:

- **Synchronization phase** can approach real-time through multiple incremental synchronizations
- **Final switching phase** typically still requires short downtime or temporary access switching
- Therefore, you cannot simply say "seconds-level seamless migration"

#### One-Sentence Memory
- Synchronization can be fast, but switching usually still requires short downtime

---

## Nine. Review Suggestions

### 1. Key Tools to Remember
- Alibaba Cloud: SMC / DTS / OSSImport / CMH / LHM
- Tencent Cloud: Server Online Migration / DTS / COS Migration / DBbridge
- Huawei Cloud: SMS / DRS / OMS / CDM
- Baidu Intelligent Cloud: Server Migration Center / DTS / CMC

### 2. Key Methodologies to Remember
- Full Synchronization
- Incremental Synchronization
- Final Cutover
- Data Verification
- Rollback Plan
- Access Entry Switching

### 3. Safe Expression in Interviews
- Avoid over-explaining the underlying implementation of each tool
- Focus on clearly explaining "Tool Focus + Applicable Scenarios + Common Migration Principles"
- Avoid easily claiming migration is "completely zero downtime" or "unified seconds-level"

---

## Ten. Keyword Quick Notes

- P2V
- V2V
- Cross-Cloud Migration
- Whole Machine Migration
- Database Online Migration
- Object Storage Migration
- Full Synchronization
- Incremental Synchronization
- Cutover
- Rollback
- Access Switching
- SMC
- DTS
- OSSImport
- SMS
- DRS
- OMS
- CDM
- CMC
- CMH
- LHM