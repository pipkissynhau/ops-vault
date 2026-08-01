# Ceph Basics: Why Distributed Storage is Needed and Ceph's Overall Architecture

# Ceph Basics: Why Distributed Storage is Needed and Ceph's Overall Architecture

Recommended path: 05-Storage/01-Ceph/01-Ceph Basics: Why Distributed Storage is Needed and Ceph's Overall Architecture.md

Tags: #Ceph #DistributedStorage #RADOS #OSD #MON #MGR #BlockStorage #FileStorage #ObjectStorage #SRE #Clouds. #Kubernetes

---

## I. Document Overview

This is the first foundational note of the Ceph advanced SRE storage module, focusing on answering three key questions:

1. Why is distributed storage needed?
2. What problems does Ceph solve?
3. How to understand Ceph's overall architecture?

This document does not directly enter installation and deployment, but instead establishes an overall understanding of Ceph first. Ceph's subsequent deployment, OSD management, Pool/PG, CRUSH, RBD, CephFS, RGW, Kubernetes CSI integration, fault diagnosis, and performance optimization will all be built upon this foundational knowledge.

Main thread of this document:

    Single-machine storage issues
        |
        v
    Value of distributed storage
        |
        v
    Ceph's overall positioning
        |
        v
    RADOS foundation
        |
        v
    Three capabilities: RBD / CephFS / RGW
        |
        v
    Ceph's SRE perspective on operational boundaries

---

## II. Why is Distributed Storage Needed

### 2.1 Common Issues with Single-Machine Storage

In traditional environments, business data is typically stored on local disks of a single server, for example:

    /data
    /var/lib/mysql
    /mnt/app-data
    /opt/application/upload

This approach is simple and direct, but the problems are equally obvious.

Common risks include:

- Data loss due to single-machine disk failure
- Business unavailability due to server failure
- Difficulties in expanding storage capacity after disk space is exhausted
- Inability to share data between multiple nodes
- Local disk performance and capacity limited by single-machine constraints
- Complex data migration during business relocation
- Kubernetes Pod drift results in local data becoming unavailable
- Backup and recovery depend on additional systems
- Inability to natively support multi-replica and automatic recovery

Typical scenarios:

    A MySQL instance runs on a single server with data directory at /data/mysql.
    If the server's disk fails, MySQL data may be lost directly.
    If the server crashes, business operations can only wait for server recovery.
    If migrating to another server, data needs to be copied, validated, and business switched.

---

### 2.2 What RAID Solves and What It Doesn't

Many traditional environments use RAID to improve single-machine disk reliability.

For example:

| RAID Type | Purpose |
|---|---|
| RAID1 | Mirroring, improves single-machine disk redundancy |
| RAID5 | Balances capacity and parity |
| RAID10 | Balances performance and redundancy |

RAID can solve some disk failure issues, but not all.

RAID cannot solve:

- Entire server failure
- Data center failure
- Horizontal scaling of storage nodes
- Multi-business shared storage
- Kubernetes dynamic volume provisioning
- Automatic cross-node data recovery
- Multi-replica distributed placement
- Unified block, file, object storage interfaces

You can understand it this way:

    RAID solves the problem of single-machine internal disk redundancy.
    Distributed storage solves the overall storage capability issues of multi-node, multi-disk, multi-replica, and multi-access interface.

---

### 2.3 What Distributed Storage Solves

The core idea of distributed storage is:

    Distribute data across multiple servers and disks, and improve capacity, reliability, and scalability through replication, erasure coding, scheduling algorithms, and health check mechanisms.

Distributed storage typically solves:

- Horizontal scaling
- High availability across multiple nodes
- Data replication
- Automatic fault recovery
- Unified capacity management
- Multi-client access
- Storage resource pooling
- Decoupling storage from compute
- Cloud platform and Kubernetes dynamic storage needs

Simple illustration:

    Traditional local storage:

        App -> Local disk

    Distributed storage:

        App -> Storage client -> Distributed storage cluster -> Multiple nodes and disks

---

## III. What is Ceph

Ceph is an open-source distributed storage system. Its core features are:

    A single underlying distributed storage cluster that provides block, file, and object storage capabilities.

Ceph can provide:

| Capability | Component / Interface | Typical Use Cases |
|---|---|---|
| Block Storage | RBD | Virtual machine cloud disks, Kubernetes PVC, database data disks |
| File Storage | CephFS | Shared file systems, multi-node shared read/write |
| Object Storage | RGW | S3-compatible object storage, images, backups, archives |
| Underlying Object Storage | RADOS | Ceph's unified distributed storage foundation |

You can understand it this way:

    Ceph is not simply a "cloud disk".
    Ceph is not simply an "object storage".
    Ceph is a general-purpose distributed storage platform.

---

## IV. Ceph Overall Architecture

### 4.1 Overall Architecture Diagram

Ceph's logical architecture can be simplified to: /think

```
┌───────────────────────────────────────────────┐
│                  Application / Client                 │
└───────────────────────────────────────────────┘
             │               │                │
             │               │                │
             v               v                v
┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│      RBD       │ │     CephFS     │ │      RGW       │
│    Block Storage       │ │    File Storage     │ │   Object Storage/S3   │
└────────┬───────┘ └────────┬───────┘ └────────┬───────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            v
┌───────────────────────────────────────────────┐
│                 RADOS Distributed Object Storage Foundation       │
└───────────────────────────────────────────────┘
             │                  │                  │
             v                  v                  v
┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│      OSD       │ │      OSD       │ │      OSD       │
│   Data Storage Process  │ │   Data Storage Process  │ │   Data Storage Process  │
└────────────────┘ └────────────────┘ └────────────────┘

The cluster state and management capabilities are maintained by the following components:

MON: Maintains cluster mapping and consistency
MGR: Management, monitoring, Dashboard, Prometheus, etc.
MDS: Provides metadata service for CephFS
RGW: Provides object storage S3/Swift API

---

### 4.2 Ceph's core is not "directories", but objects

Many people new to Ceph often misunderstand it using traditional file system thinking:

    Where is the file located?
    Which path is the disk mounted to?
    Is the data stored in /data on a specific machine?

But Ceph's underlying core is RADOS object storage.

In other words:

    Regardless of whether it's RBD, CephFS, or RGW on top, everything ultimately converts to RADOS objects and is distributed across multiple OSDs.

For example:

    A RBD cloud disk
        |
        v
    Split into multiple objects
        |
        v
    Distributed to multiple OSDs via Pool/PG/CRUSH
        |
        v
    Multi-replica written to different nodes or disks

This is why Ceph can simultaneously support block, file, and object interfaces.

---

## Five, What Problems Does Ceph Solve

### 5.1 Unified Storage Pool

Ceph organizes multiple disks across multiple servers into a unified storage pool.

Traditional approach:

    Server A has 2TB
    Server B has 2TB
    Server C has 2TB

Each server is used separately, leading to fragmented capacity.

Ceph approach:

    ceph-node01 /dev/sdb
    ceph-node02 /dev/sdb
    ceph-node03 /dev/sdb
        |
        v
    Unified Ceph Storage Cluster
        |
        v
    Pool / RBD / CephFS / RGW

---

### 5.2 Data Replication and Automatic Recovery

Ceph can protect data through replication mechanisms.

For example, if the Pool replica count is 3:

    size = 3

Means each piece of data has 3 replicas by default.

Illustration:

    Object A
      ├── Replica 1 -> ceph-node01 / OSD.0
      ├── Replica 2 -> ceph-node02 / OSD.2
      └── Replica 3 -> ceph-node03 / OSD.4

If ceph-node02's disk fails, Ceph will recover the replica on other OSDs based on cluster state and CRUSH rules.

This is one of Ceph's core values:

    It doesn't prevent failures, but detects, isolates, and recovers after failures occur.

---

### 5.3 Horizontal Scaling

Traditional single-machine disk expansion typically requires:

- Shutdown and add disks
- Expand filesystem
- Migrate data
- Adjust mount points
- Modify business configurations

Ceph's scaling approach is:

    Add new nodes
        |
        v
    Add new OSD disks
        |
        v
    Ceph detects new capacity
        |
        v
    CRUSH recalculates data distribution
        |
        v
    Data gradually Rebalances

This makes Ceph more suitable for scenarios with continuously growing capacity.

---

### 5.4 Support for Multiple Storage Interfaces

Ceph can provide multiple interfaces through a single cluster.

| Type | Ceph Capability | Analogy |
|---|---|---|
| Block Storage | RBD | Cloud Disk |
| File Storage | CephFS | NFS-like shared file system |
| Object Storage | RGW | S3/OSS-like object storage |

This is valuable for platform environments:

    Virtual machines need block storage
    Kubernetes needs PVC
    Multi-node applications need shared file systems
    Backup, images, archives need object storage

Ceph can provide these capabilities on a unified foundation.
```

- Object Storage
- Data Distribution
- Data Replication
- Data Recovery
- Cluster Expansion
- Cluster Self-Management

Can be understood as:

    RADOS is the foundation of Ceph.

RBD, CephFS, and RGW are all built on top of RADOS.

---

### 6.2 OSD

OSD, full name Object Storage Daemon.

OSD is the process in Ceph that actually stores data.

Each OSD typically corresponds to a disk or storage device.

OSD is responsible for:

- Storing objects
- Processing read/write requests
- Participating in replica replication
- Participating in recovery
- Reporting heartbeats
- Reporting capacity and status
- Synchronizing data with other OSDs

Common understanding:

    Without OSDs, there would be no actual data storage.
    Ceph's capacity mainly comes from OSDs.
    Ceph's performance also strongly depends on the number of OSDs, disk performance, and network capabilities.

---

### 6.3 MON

MON, full name Monitor.

MON is responsible for maintaining key mapping information of the Ceph cluster, such as:

- Monitor Map
- OSD Map
- PG Map
- CRUSH Map
- MDS Map

MON is not used for storing business data; it mainly maintains cluster status and consistency.

Production recommendations:

    MONs are typically deployed in an odd number, such as 3 or 5.
    In experimental environments, 3 MONs are commonly used.

Reason:

    MONs need to use a majority mechanism to ensure consistency.
    An even number of MONs has no obvious advantages and may complicate the arbitration design.

---

### 6.4 MGR

MGR, full name Manager.

MGR is responsible for management and monitoring capabilities of the Ceph cluster.

Common capabilities:

- Dashboard
- Prometheus Exporter
- Cluster metrics
- Orchestration interface
- Management modules
- Alert information

In cephadm deployments, MGR is also closely related to orchestration management capabilities.

Production recommendations:

    At least 2 MGR instances should be deployed, one active and one standby.

---

### 6.5 MDS

MDS, full name Metadata Server.

MDS mainly serves CephFS.

CephFS is a file system, which requires managing:

- Directories
- File names
- Inodes
- Permissions
- Metadata

These metadata are handled by MDS.

Note:

    If only RBD or RGW is used, MDS may not be needed.
    If CephFS is to be used, MDS must be deployed.

---

### 6.6 RGW

RGW, full name RADOS Gateway.

RGW provides object storage interfaces compatible with S3 / Swift API.

Typical uses:

- Image storage
- Backup archiving
- Object storage service
- Private S3
- Integration with applications via S3 SDK

RGW has some overlapping positioning with MinIO; when studying MinIO later, they can be compared:

    Ceph RGW: Built on Ceph RADOS, it is Ceph's object storage gateway.
    MinIO: A standalone object storage system focused on S3 object storage.

---

## Seven, Several Confusing Concepts in Ceph

### 7.1 Pool

Pool is a logical storage pool in Ceph.

It can be understood as:

    A logical container for a group of data.

Examples:

    rbd-pool
    cephfs-data
    cephfs-metadata
    rgw.buckets.data

Pools can be configured with:

- Replica count
- PG count
- CRUSH rules
- Application type
- Quotas
- Storage policies

---

### 7.2 PG

PG, full name Placement Group.

PG is a very important intermediate layer in Ceph.

Simple understanding:

    PG is the grouping unit between objects and OSDs.

Why is PG needed?

If each object directly maps to an OSD, the management cost would be very high. Ceph uses PG as an intermediate layer:

    Object -> PG -> OSD Set

This reduces the complexity of object scheduling.

Common states:

    active+clean
    active+degraded
    active+recovering
    peering
    stale
    undersized

A dedicated section on Pool and PG will be written later.

---

### 7.3 CRUSH

CRUSH is Ceph's data placement algorithm.

It determines:

    Which OSDs an object should be placed on.

CRUSH considers:

- OSD weights
- Hosts
- Racks
- Data centers
- Failure domains
- Pool rules
- Replica count

Value of CRUSH:

    No need for a centralized metadata service to record where each object is.
    Clients can calculate data locations based on the CRUSH Map.
    Data distribution can be recalculated during cluster expansion, scaling down, or failure recovery.

---

### 7.4 Object

Object is the basic unit of storage at the CephBottom.

From an upper-level perspective, it may appear as:

    A RBD block device
    A CephFS file
    A RGW object

But at the bottom level, it becomes RADOS objects.

---

### 7.5 Replica

Replica is a copy.

If the Pool's size is set to 3, it means each object has 3 replicas.

Example:

    ceph osd pool set rbd-pool size 3

This represents:

    Normally, each object in rbd-pool will retain 3 copies.

Note:

    The higher the replica count, the higher the data safety, but the lower the usable capacity.
    The lower the replica count, the higher the space utilization, but the higher the data risk.

---

## Eight, Three Main Usage Methods of Ceph

### 8.1 RBD: Block Storage

RBD, full name RADOS Block Device.

It provides block device capabilities.

Typical uses:

- Virtual machine cloud disks
- OpenStack Cinder backend
- Kubernetes PVC
- Database data disks
- Stateful application data disks

Illustration:

    Pod / VM / Host
        |
        v
    RBD Image
        |
        v
    RADOS Objects
        |
        v
    OSDs

RBD features:

- Supports snapshots
- Supports cloning
- Supports thin provisioning
- Suitable for single-client mounting
- Commonly used in Kubernetes ReadWriteOnce scenarios

---

### 8.2 CephFS: File Storage

CephFS is Ceph's file system capability.

Typical uses:

- Shared directories across multiple nodes
- Shared configuration files
- Shared log storage
- Shared files for AI/data processing
- Kubernetes ReadWriteMany scenarios

Illustration: /think

Multiple Clients  
|  
v  
CephFS  
|  
v  
MDS Manages Metadata  
|  
v  
RADOS Stores Data  

CephFS Features:  

- Provides POSIX filesystem semantics  
- Depends on MDS  
- Supports multi-client shared access  
- Suitable for shared file scenarios  

---  

### 8.3 RGW: Object Storage  

RGW is Ceph's object storage gateway.  

Typical Use Cases:  

- Images  
- Videos  
- Backups  
- Archives  
- S3-compatible applications  
- Private object storage service  

Diagram:  

    S3 Client / SDK / mc / s3cmd  
        |  
        v  
    RGW  
        |  
        v  
    RADOS  
        |  
        v  
    OSDs  

RGW Features:  

- Compatible with S3 API  
- Uses Bucket / Object model  
- Suitable for massive amounts of unstructured objects  
- Can serve as a private object storage solution  

---  

## Nine. Ceph and Kubernetes Relationship  

In Kubernetes, Ceph is commonly used as a persistent storage backend.  

Common Approaches:  

| Ceph Capability | Kubernetes Integration Method | Use Case |  
|---|---|---|  
| RBD | RBD CSI | Single Pod block storage, ReadWriteOnce |  
| CephFS | CephFS CSI | Multi-Pod shared file storage, ReadWriteMany |  
| RGW | S3 API | Applications access object storage via S3 SDK |  

Typical RBD CSI Workflow:  

    StorageClass  
        |  
        v  
    PVC  
        |  
        v  
    Ceph CSI Provisioner  
        |  
        v  
    Ceph RBD Image  
        |  
        v  
    Pod mounts block device  

Typical CephFS CSI Workflow:  

    StorageClass  
        |  
        v  
    PVC  
        |  
        v  
    CephFS CSI  
        |  
        v  
    CephFS Subvolume  
        |  
        v  
    Multi-Pod shared mount  

Note:  

    Kubernetes does not directly manage Ceph OSDs.  
    Kubernetes only uses Ceph's storage capabilities via CSI.  
    The Ceph cluster itself still requires independent planning, deployment, monitoring, and operations.  

---  

## Ten. Ceph Suitable Scenarios  

### 10.1 Suitable Scenarios  

Ceph is suitable for:  

- Private cloud storage foundation  
- OpenStack backend storage  
- Kubernetes persistent storage  
- Large-scale virtual machine cloud disks  
- Multi-tenant storage platform  
- Unified block, file, and object storage  
- Scalable storage cluster  
- Environments requiring multi-replica and fault recovery capabilities  
- Enterprises with professional operations capabilities  

---  

### 10.2 Unsuitable Scenarios  

Ceph may not be suitable for:  

- Extremely small-scale environments  
- Environments without dedicated maintenance  
- Simple storage needs with only one or two servers  
- Scenarios extremely sensitive to deployment complexity  
- Environments with poor network and disk performance  
- Small-scale scenarios requiring only simple S3 object storage  
- Environments unwilling to invest in monitoring, alerts, backups, and recovery drills  

If it's just small-scale object storage, MinIO may be simpler.  

If it's just Kubernetes small clusters needing block storage, Longhorn may be easier to implement.  

---  

## Eleven. Ceph Advantages  

Ceph's advantages include:  

### 11.1 Unified Storage Capabilities  

A single cluster can simultaneously provide:  

- RBD  
- CephFS  
- RGW  

This is highly valuable for platform environments.  

---  

### 11.2 Horizontal Scaling Capabilities  

Ceph can scale capacity and performance by adding nodes and OSDs.  

Scaling Method:  

    New Server  
        |  
        v  
    New Data Disk  
        |  
        v  
    Add OSD  
        |  
        v  
    Data automatically rebalances  

---  

### 11.3 Self-Recovery Capabilities  

When an OSD fails, Ceph can recover data based on replicas and CRUSH rules.  

Recovery Process Includes:  

- Detect OSD down  
- Mark objects as degraded  
- Select new OSD  
- Trigger recovery / backfill  
- Restore replica count  
- Return to active+clean  

---  

### 11.4 Decentralized Data Placement  

Ceph calculates data locations via CRUSH algorithm, avoiding traditional centralized metadata bottlenecks.  

This is also one of the key reasons Ceph can scale to large clusters.  

---  

### 11.5 Cloud-Native Ecosystem Support  

Ceph can integrate with Kubernetes via CSI.  

Common Projects:  

- ceph-csi  
- Rook Ceph  
- OpenStack  
- Proxmox VE  
- KubeVirt  

---  

## Twelve. Ceph Complexity  

Ceph is powerful but complex.  

Common Complexity Points Include:  

- Many components  
- Many states  
- Many concepts  
- High deployment requirements  
- Disk planning is important  
- Network planning is important  
- Understanding PG states is needed  
- Recovery process affects performance  
- Capacity water levels need strict monitoring  
- Fault handling cannot be done blindly  
- Upgrades require careful planning  

Ceph's Typical Risks:  

    Not afraid of single OSD failure, but afraid of capacity planning errors.  
    Not afraid of single node failure, but afraid of fault domain design errors.  
    Not afraid of recovery, but afraid of resource exhaustion during recovery causing business snowball.  
    Not afraid of alerts, but afraid of not understanding the meaning behind HEALTH_WARN.  

---  

## Thirteen. Ceph Basic Experiment Environment  

Subsequent Ceph experiments default to using:  

| IP | Hostname | Role |  
|---|---|---|  
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |  
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |  
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |  
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (optional) |  
| 10.0.0.35 | ceph-client | Client testing (optional) |  

Default System:  

    Ubuntu Server 22.04.5 LTS  

Additional Systems:  

    Rocky Linux 9  

Each storage node should prepare at least:  

    /dev/sda  System disk  
    /dev/sdb  Ceph OSD data disk  
    /dev/sdc  Ceph OSD data disk (optional)  

Network:  

    10.0.0.0/24  

Note: /think

Ceph experiments do not directly consume existing Kubernetes nodes 10.0.0.20/21/22.
Ceph uses independent VM/bare metal multi-node clusters, facilitating OSD, disk, node, recovery, and failure scenario exercises.

---

## 14. Preview of Basic Verification Commands

After deployment, common commands include:

Check cluster status:

    ceph -s

Check health details:

    ceph health detail

Check OSD tree:

    ceph osd tree

Check OSD capacity:

    ceph osd df

Check pools:

    ceph osd pool ls

Check PG status:

    ceph pg stat

Check orchestration services:

    ceph orch ps

Check host list:

    ceph orch host ls

Check MGR services:

    ceph mgr services

Check CephFS:

    ceph fs ls

Check RGW users:

    radosgw-admin user list

These commands will be explained in detail in corresponding chapters.

---

## 15. How to Learn Ceph from an Advanced SRE Perspective

Ceph learning should not stop at:

    cephadm bootstrap
    ceph -s
    ceph osd tree

Advanced SREs need to focus more on:

### 15.1 Data Security

Need to know:

- How many replicas are there
- Where replicas are distributed
- What fault domain is
- Whether data becomes degraded after OSD down
- Whether it can recover to active+clean
- What impact nearfull/full has

---

### 15.2 Fault Recovery

Need to know:

- How to determine OSD down
- How to replace a failed OSD disk
- How to check PG degraded status
- Why backfill is slow
- Whether recovery affects business
- What operations trigger large-scale rebalance

---

### 15.3 Performance Impact

Need to know:

- How disk types affect performance
- How network bandwidth impacts recovery
- How PG count affects the cluster
- The role of BlueStore
- Differences in client read/write modes
- Different performance bottlenecks of RBD, CephFS, and RGW

---

### 15.4 Production Boundaries

Need to know:

- Whether small-scale environments are really suitable for Ceph
- Whether there are enough nodes and disks
- Whether there is monitoring and alerting
- Whether someone can handle PG/OSD failures
- Whether recovery drills have been done
- Whether there are upgrade and rollback plans

---

## 16. Common Misconceptions

### 16.1 Misconception One: Ceph Equals Object Storage

Incorrect understanding:

    Ceph is just object storage.

Correct understanding:

    Ceph'sBottom is RADOS distributed object storage, but it can provide RBD block storage, CephFS file storage, and RGW object storage externally.

---

### 16.2 Misconception Two: Ceph Can Be Used in Production Immediately

Incorrect understanding:

    cephadm bootstrap succeeds and ceph -s shows HEALTH_OK, so it can be used in production.

Correct understanding:

    Production requires capacity planning, fault domain design, monitoring alerts, backup recovery, performance evaluation, failure drills, and upgrade strategies.

---

### 16.3 Misconception Three: More Replicas Are Better

Incorrect understanding:

    More replicas mean more safety, so higher is better.

Correct understanding:

    More replicas mean stronger data redundancy, but available capacity decreases, write costs increase, and recovery traffic also increases.
    In production, it needs to be comprehensively evaluated based on business importance, capacity costs, and fault domain design.

---

### 16.4 Misconception Four: Ceph Can Replace All Storage

Incorrect understanding:

    With Ceph, there's no need for other storage.

Correct understanding:

    Ceph is very powerful, but not suitable for all scenarios.
    For small-scale object storage, consider MinIO.
    For small Kubernetes block storage, consider Longhorn.
    For simple shared directories, consider NFS.
    The key is to choose based on scenarios, not to use Ceph for all problems.

---

## 17. Interview Answer Strategy

If asked in an interview:

    Why is Ceph needed? What is Ceph?

You can answer:

    Ceph is an open-source distributed storage system. Its core value is to organize multiple servers and disks into a unified distributed storage cluster, providingBottom object storage capabilities via RADOS, and offering RBD block storage, CephFS file storage, and RGW object storage on top of it.
    Compared to single-machine local storage, Ceph solves capacity horizontal scaling, node high availability, data replication, fault recovery, unified storage resource pools, and multiple storage interfaces.
    From an operations perspective, Ceph's focus isn't just installation, but understanding concepts like OSD, MON, MGR, Pool, PG, CRUSH, replicas, fault domains, backfill, and recovery.
    In production environments, using Ceph requires focusing on node planning, disk planning, network planning, capacity water levels, monitoring alerts, OSD fault recovery, PG anomaly handling, performance optimization, and upgrade risks.
    In Kubernetes, Ceph typically connects via Ceph CSI, with RBD suitable for providing block storage PVCs and CephFS suitable for shared file storage PVCs.

---

## 18. Summary of This Chapter

This chapter establishes a basic understanding of Ceph:

1. Single-machine storage has capacity, availability, and scalability issues.
2. RAID only solves single-machine disk redundancy, not distributed high availability.
3. Distributed storage improves reliability through multi-node, multi-disk, replication, and recovery mechanisms.
4. Ceph is a unified distributed storage platform.
5. Ceph's core is RADOS.
6. Ceph provides three capabilities: RBD, CephFS, and RGW.
7. OSD handles actual data storage, MON manages cluster status, MGR handles management monitoring, MDS serves CephFS, and RGW provides S3 object interfaces.
8. Ceph is powerful but complex; production environments must focus on capacity, fault domains, recovery, performance, security, and monitoring.
9. Advanced SREs learning Ceph should focus on understanding where data is, where replicas are, how to recover after failures, and what resources recovery consumes.

---

## 19. Reference Documents

Ceph official architecture documentation:

    https://docs.ceph.com/en/reef/architecture/

Ceph official technical introduction:

    https://ceph.io/en/discover/technology/

Ceph official RBD documentation:

    https://docs.ceph.com/en/reef/rbd/

Ceph official terminology table:

    https://docs.ceph.com/en/latest/glossary/

Ceph rados command documentation:

    https://docs.ceph.com/en/latest/man/8/rados/

Ceph official documentation homepage:

    https://docs.ceph.com/