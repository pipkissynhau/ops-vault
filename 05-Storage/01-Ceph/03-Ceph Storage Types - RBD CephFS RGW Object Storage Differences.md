# Ceph Storage Types: Differences Between RBD, CephFS, and RGW Object Storage

Recommended path: 05-Storage/01-Ceph/03-Ceph Storage Types: RBD, CephFS, RGW Object Storage Differences.md

Tags: #Ceph #RBD #CephFS #RGW #BlockStorage #FileStorage #ObjectStorage #S3 #Kubernetes #SRE #DistributedStorage

---

## I. Document Overview

This is the third article in the Ceph Advanced SRE Storage module, focusing on Ceph's three primary storage capabilities:

- RBD: Block Storage
- CephFS: File Storage
- RGW: Object Storage

Ceph's core is RADOS. Regardless of whether upper-layer applications use RBD, CephFS, or RGW, all data will ultimately be stored as RADOS Objects on OSDs.

This document answers the following questions:

1. What are block storage, file storage, and object storage?
2. What scenarios are suitable for RBD, CephFS, and RGW?
3. What differences exist among them in architecture, access methods, Kubernetes integration, and operational complexity?
4. How to choose in production environments?
5. What should advanced SREs focus on when designing storage solutions?

---

## II. Why Ceph Provides Three Storage Types

Ceph's core capability comes from RADOS.

RADOS is Ceph's underlying distributed object storage system, responsible for:

- Data object storage
- Replication management
- Data recovery
- OSD management
- Pool management
- PG mapping
- CRUSH data placement

On top of RADOS, Ceph provides different access interfaces:

    ┌─────────────────────────────────────────────┐
    │                Application / Client                │
    └─────────────────────────────────────────────┘
             │                │                │
             v                v                v
    ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
    │      RBD       │ │     CephFS     │ │      RGW       │
    │    Block Storage       │ │    File Storage     │ │   Object Storage/S3   │
    └────────┬───────┘ └────────┬───────┘ └────────┬───────┘
             │                  │                  │
             └──────────────────┼──────────────────┘
                                v
    ┌─────────────────────────────────────────────┐
    │              RADOS Distributed Storage Foundation            │
    └─────────────────────────────────────────────┘
                                │
                                v
    ┌─────────────────────────────────────────────┐
    │                  OSD Cluster                    │
    └─────────────────────────────────────────────┘

This can be understood as:

    RADOS is the foundation.
    RBD, CephFS, and RGW are access methods tailored for different use cases.

---

## III. Overview of the Three Storage Types

| Storage Type | Ceph Component | Access Model | Typical Use Cases | Kubernetes Scenario |
|---|---|---|---|---|
| Block Storage | RBD | Block Device | Cloud disks, database disks, virtual machine disks | ReadWriteOnce PVC |
| File Storage | CephFS | File System | Shared directories, multi-node read/write | ReadWriteMany PVC |
| Object Storage | RGW | S3 / Swift API | Images, backups, archives, unstructured data | Applications access via S3 API |

Simple judgment criteria:

    Need to mount as a disk for a single host or Pod: Choose RBD.
    Need multiple nodes to share the same directory: Choose CephFS.
    Need to access objects via HTTP/S3 API: Choose RGW.

---

## IV. Block Storage: RBD

### 4.1 What is RBD

RBD, full name:

    RADOS Block Device

RBD is Ceph's block storage capability.

It abstracts Ceph cluster storage resources into a block device.

For clients, RBD resembles a cloud hard disk:

    /dev/rbd0

Applications can create file systems on it, such as:

    ext4
    xfs

Then mount it to a directory for use:

    /mnt/data

---

### 4.2 RBD Data Flow

Simplified RBD data flow:

    Application writes to file system
        |
        v
    File system writes to block device
        |
        v
    RBD Image
        |
        v
    RBD object slicing
        |
        v
    Pool / PG / CRUSH
        |
        v
    Multiple OSDs store object replicas

Diagram: /think

```
┌────────────────────┐
│       Application   │
└─────────┬──────────┘
          v
┌────────────────────┐
│  File system ext4/xfs │
└─────────┬──────────┘
          v
┌────────────────────┐
│    RBD Image        │
└─────────┬──────────┘
          v
┌────────────────────┐
│  RADOS Objects      │
└─────────┬──────────┘
          v
┌────────────────────┐
│      OSD Cluster    │
└────────────────────┘

---

### 4.3 When is RBD suitable?

RBD is suitable for:

- Virtual Machine Cloud Disk
- OpenStack Cinder backend
- KubeVirt virtual machine disk
- Kubernetes stateful application PVC
- Database data disk
- Single-node application persistent data disk
- Scenarios requiring Snapshot / Clone

Typical use cases:

- MySQL
- PostgreSQL
- Redis persistence
- MongoDB
- Elasticsearch single-instance data disk
- Harbor / GitLab and other stateful service data disks

---

### 4.4 When is RBD unsuitable?

RBD is unsuitable for:

- Multiple nodes simultaneously reading/writing the same file system directory
- Large number of small files shared access scenarios
- Scenarios requiring POSIX shared file system semantics
- Direct storage of images, backups, and archives via HTTP API

Reasons:

    RBD is a block device.
    Block devices are typically suitable for single client mounting.
    Multiple clients mounting the same regular file system may lead to data corruption.

---

### 4.5 RBD and Kubernetes

In Kubernetes, RBD is typically connected via Ceph CSI.

Typical workflow:

    StorageClass
        |
        v
    PVC
        |
        v
    Ceph RBD CSI
        |
        v
    Ceph RBD Image
        |
        v
    Pod mounted as a data volume

RBD typically corresponds to:

    ReadWriteOnce

That is, a PVC is typically mounted by a Pod on a single node.

Illustration:

    ┌────────────────────┐
    │       Pod           │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │        PVC          │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │     StorageClass    │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │     RBD CSI         │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │    Ceph RBD Image   │
    └────────────────────┘

---

### 4.6 Preview of Common RBD Commands

Check RBD Pool:

    ceph osd pool ls

Enable RBD application type:

    ceph osd pool application enable rbd rbd

Check RBD Image:

    rbd ls -p rbd

Create RBD Image:

    rbd create demo-image --size 10G -p rbd

Check RBD Image information:

    rbd info rbd/demo-image

Create snapshot:

    rbd snap create rbd/demo-image@snap01

Check snapshots:

    rbd snap ls rbd/demo-image

Delete snapshot:

    rbd snap rm rbd/demo-image@snap01

Delete RBD Image:

    rbd rm rbd/demo-image

---

## V. File Storage: CephFS

### 5.1 What is CephFS

CephFS is a distributed file system provided by Ceph.

It provides access similar to traditional file systems:

    Directory
    File
    Permissions
    inode
    Metadata
    Mount point

Clients can mount CephFS to a local directory:

    /mnt/cephfs

Then read and write files as with a regular file system.

---

### 5.2 Core Components of CephFS: MDS

CephFS requires MDS.

MDS, full name:

    Metadata Server

MDS is responsible for metadata management of CephFS, such as:

- Filename
- Directory structure
- inode
- Permissions
- File attributes
- File locks
- Metadata cache

CephFS data and metadata are typically stored separately:

| Type | Storage Location |
|---|---|
| File data | CephFS Data Pool |
| File metadata | CephFS Metadata Pool |
| Metadata service | MDS |

Illustration: /think

```
┌────────────────────┐
│     CephFS Client   │
└─────────┬──────────┘
          │
          v
┌────────────────────┐
│        MDS          │  Handles directory, filename, inode, etc. metadata
└─────────┬──────────┘
          │
          v
┌────────────────────┐
│     RADOS Pools     │
│ metadata + data     │
└─────────┬──────────┘
          │
          v
┌────────────────────┐
│       OSD Cluster   │
└────────────────────┘

---

### 5.3 Use Cases for CephFS

CephFS is suitable for:

- Multi-node shared directory
- Multi-Pod shared read/write
- ReadWriteMany PVC
- Shared configuration directory
- Shared upload directory
- AI / big data shared data directory
- Multi-client file access
- Replacing some NFS scenarios

Typical scenarios:

    Multiple Web Pods need to share an upload file directory.
    Multiple compute nodes need to access the same batch of training data.
    Multiple application instances need to share file-based data.

---

### 5.4 Scenarios Not Suitable for CephFS

CephFS may not be suitable for:

- High IOPS database primary data disk
- Small-scale simple shared directory where NFS is sufficient
- Scenarios with extremely sensitive metadata performance without tuning capabilities
- Environments without MDS operations experience
- Environments unwilling to monitor MDS status

Reasons:

    CephFS has an additional MDS component compared to RBD.
    File system metadata performance is closely related to MDS status.
    Large numbers of small files and high concurrency directory operations may put pressure on MDS.

---

### 5.5 CephFS and Kubernetes

In Kubernetes, CephFS is typically connected via CephFS CSI.

CephFS is more suitable for:

    ReadWriteMany

That is, multiple Pods can share the same PVC.

Typical workflow:

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
    Multiple Pods mounting the same shared directory

Illustration:

    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │    Pod A     │    │    Pod B     │    │    Pod C     │
    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
           │                   │                   │
           └───────────────────┼───────────────────┘
                               v
                    ┌────────────────────┐
                    │     CephFS PVC      │
                    └────────────────────┘

---

### 5.6 Preview of Common CephFS Commands

View file system:

    ceph fs ls

View CephFS status:

    ceph fs status

View MDS status:

    ceph mds stat

Create metadata pool:

    ceph osd pool create cephfs_metadata 32

Create data pool:

    ceph osd pool create cephfs_data 64

Create CephFS:

    ceph fs new cephfs cephfs_metadata cephfs_data

View mount information:

    ceph fs get cephfs

Client mount example:

    mount -t ceph ceph-node01,ceph-node02,ceph-node03:/ /mnt/cephfs -o name=admin,secretfile=/etc/ceph/admin.secret

---

## Six. Object Storage: RGW

### 6.1 What is RGW

RGW, full name:

    RADOS Gateway

RGW is the object storage gateway of Ceph.

It provides object storage interfaces compatible with S3 / Swift.

Users do not access RGW through block devices or file systems, but through HTTP API.

Common access methods:

- S3 SDK
- AWS CLI
- s3cmd
- mc
- Application object storage client

---

### 6.2 RGW Object Storage Model

Core model of object storage:

    User
      |
      v
    Bucket
      |
      v
    Object

Example:

    User: app-user
    Bucket: images
    Object: 2026/04/28/a.jpg

Object storage does not emphasize traditional directory structure.

Although object names can include `/`:

    logs/2026/04/app.log

This is essentially still an object name, not a true file system directory.

---

### 6.3 Use Cases for RGW

RGW is suitable for:

- Image storage
- Video storage
- Backup archiving
- Log archiving
- Object storage service
- Private S3
- Applications accessing files via S3 SDK
- Business compatible with cloud object storage interfaces

Typical scenarios:

    Application uploads images to object storage.
    Backup systems write backup files to S3.
    Log systems archive historical logs.
    User files, attachments, and static resources enter object storage.

---

### 6.4 Scenarios Not Suitable for RGW

RGW is not suitable for:

- Database data disks requiring block device semantics
- Shared directories requiring POSIX file system semantics
- Scenarios needing direct mount for local directory-like usage
- Scenarios with special dependencies on low-latency, strongly consistent file locks for single objects

Object storage is more suitable for:

    Writing objects
    Reading objects
    Deleting objects
    Listing objects
    Managing objects via API
```

---

### 6.5 Relationship Between RGW and MinIO

RGW and MinIO can both provide S3-compatible object storage capabilities.

Differences:

| Comparison Item | Ceph RGW | MinIO |
|---|---|---|
| Ecosystem Belonging | Part of the Ceph ecosystem | Independent object storage system |
| Underlying Storage | RADOS | MinIO's own Erasure Coding |
| Deployment Complexity | Higher, requires Ceph cluster | Relatively simpler |
| Functional Focus | Object interface within Ceph unified storage | Focused on S3 object storage |
| Operations Complexity | Depends on Ceph's overall operations capabilities | More focused on object storage operations |
| Suitable Scenarios | Existing Ceph cluster, needs object interface | Dedicated S3 object storage construction |

Simple understanding:

    If you already have Ceph and need object storage capabilities, consider RGW.
    If you just want to quickly build S3 object storage, MinIO is lighter and more direct.

---

### 6.6 Preview of RGW Common Commands

Check RGW service:

    ceph orch ps | grep rgw

Create RGW user:

    radosgw-admin user create --uid=app-user --display-name="App User"

Check user:

    radosgw-admin user info --uid=app-user

List users:

    radosgw-admin user list

List Buckets:

    radosgw-admin bucket list

Check Bucket information:

    radosgw-admin bucket stats --bucket=<bucket-name>

Check RGW-related Pools:

    ceph osd pool ls | grep rgw

---

## Seven, Core Differences Between RBD, CephFS, and RGW

### 7.1 Access Method Differences

| Type | Access Method |
|---|---|
| RBD | Block device |
| CephFS | File system mounting |
| RGW | HTTP / S3 API |

Corresponding understanding:

    RBD is like a cloud hard disk.
    CephFS is like a shared file system.
    RGW is like a private OSS / S3.

---

### 7.2 Data Model Differences

| Type | Data Model |
|---|---|
| RBD | Block |
| CephFS | Files and directories |
| RGW | Buckets and Objects |

---

### 7.3 Client Differences

| Type | Common Clients |
|---|---|
| RBD | Linux rbd, QEMU, Kubernetes CSI, OpenStack |
| CephFS | Linux kernel client, ceph-fuse, Kubernetes CSI |
| RGW | S3 SDK, awscli, s3cmd, mc, HTTP API |

---

### 7.4 Kubernetes Usage Differences

| Type | Kubernetes Usage | Access Mode |
|---|---|---|
| RBD | RBD CSI StorageClass | ReadWriteOnce |
| CephFS | CephFS CSI StorageClass | ReadWriteMany |
| RGW | Application accesses via S3 API | Not via PVC |

Note:

    RGW is typically not used via PVC.
    RGW is accessed by applications through S3 Endpoint in business code or configuration.

---

### 7.5 Operations Focus Differences

| Type | Main Focus Points |
|---|---|
| RBD | Image, Snapshot, Clone, Mapping, Lock, Single-client mounting |
| CephFS | MDS, metadata performance, multi-client sharing, directory operations |
| RGW | S3 users, Buckets, Policy, Endpoint, gateway high availability |

---

## Eight, How to Choose for Production Scenarios

### 8.1 Scenarios to Choose RBD

Choose RBD:

    Need a data volume similar to a cloud hard disk.

Suitable for:

- Database data disks
- Kubernetes stateful applications
- Virtual machine disks
- Single-instance application persistence
- Block devices needing snapshots and cloning

Example:

    MySQL Pod uses RBD PVC.
    KubeVirt virtual machines use RBD as the system disk.
    OpenStack virtual machine cloud disks use Ceph RBD as backend.

---

### 8.2 Scenarios to Choose CephFS

Choose CephFS:

    Need multiple clients to share the same file system directory.

Suitable for:

- Multiple Pods sharing an upload directory
- Multiple nodes sharing a dataset
- Shared file-based business
- ReadWriteMany PVC
- Some scenarios replacing NFS

Example:

    Multiple Web Pods share /uploads.
    Multiple AI tasks read the same dataset directory.
    Multiple application instances share a static file directory.

---

### 8.3 Scenarios to Choose RGW

Choose RGW:

    Need object storage and S3 API.

Suitable for:

- Images
- Videos
- Archives
- Backups
- Log archiving
- Application attachments
- Private object storage service

Example:

    Applications write uploaded files to S3 Bucket.
    Backup systems upload backup files to object storage.
    Logs are archived into object storage.

---

### 8.4 Simple Selection Mnemonic

You can judge as follows:

    Use like a disk: RBD
    Use like a shared directory: CephFS
    Use like OSS/S3: RGW

More specifically:

    Single Pod / Single VM mounting: RBD
    Multi-Pod shared read/write: CephFS
    Application accesses objects via HTTP API: RGW

---

## Nine, Architecture Topologies of the Three Types

### 9.1 RBD Topology

    ┌────────────────────┐
    │  VM / Pod / Host    │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │      RBD Image      │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │      RADOS Pool     │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │       OSD Cluster   │
    └────────────────────┘

---

### 9.2 CephFS Topology /think

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Client A   │   │   Client B   │   │   Client C   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          v
                ┌────────────────────┐
                │       CephFS        │
                └─────────┬──────────┘
                          v
                ┌────────────────────┐
                │        MDS          │
                └─────────┬──────────┘
                          v
                ┌────────────────────┐
                │   Metadata/Data Pool│
                └─────────┬──────────┘
                          v
                ┌────────────────────┐
                │       OSD Cluster      │
                └────────────────────┘

---

### 9.3 RGW Topology

    ┌────────────────────┐
    │   S3 Client / App   │
    └─────────┬──────────┘
              │ HTTP / HTTPS
              v
    ┌────────────────────┐
    │        RGW          │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │    RGW Pools        │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │      OSD Cluster       │
    └────────────────────┘

---

## TenI don't know.Three Storage Types' High Availability Focus

### 10.1 RBD High Availability Focus

RBD's high availability core is:

- Replica count of the Pool where RBD Image resides
- Pool's CRUSH Rule
- OSD distribution
- Client remount capability
- Kubernetes CSI recovery capability
- Pod rescheduling capability after node failure

Key points:

    RBD itself relies on Ceph's underlying replication to ensure data high availability.
    The business side also needs to consider whether applications can remount PVC after failure.

---

### 10.2 CephFS High Availability Focus

CephFS's high availability, besides the underlying OSD replication, also needs to focus on MDS.

Key points:

- MDS active / standby
- Replica count of Metadata Pool
- Replica count of Data Pool
- MDS failover
- Metadata pressure from large number of small files
- Consistency across multiple clients

Production recommendations:

    CephFS should at least configure standby MDS.
    Critical business needs to focus on MDS performance and metadata pool capacity.

---

### 10.3 RGW High Availability Focus

RGW's high availability focus:

- Multiple RGW instances
- Frontend load balancing
- HTTPS reverse proxy
- S3 Endpoint stability
- RGW Pool replica count
- User key security
- Bucket data distribution
- Performance differences between large and small objects

Typical production structure:

    S3 Client
        |
        v
    Nginx / LB / VIP
        |
        v
    Multiple RGW instances
        |
        v
    Ceph RADOS

---

## ElevenI don't know.Three Types' Performance Focus

### 11.1 RBD Performance Focus

RBD performance mainly focuses on:

- OSD disk performance
- Network latency
- Pool replica count
- Client IO mode
- RBD Image features
- BlueStore performance
- Database write mode
- Presence of slow OSD

Common commands:

    ceph osd perf
    ceph osd df
    ceph -s
    rbd perf image iostat

---

### 11.2 CephFS Performance Focus

CephFS performance mainly focuses on:

- MDS performance
- Metadata operation count
- Small file count
- Directory depth
- Data Pool performance
- Metadata Pool performance
- Client count
- MDS cache

Common commands:

    ceph fs status
    ceph mds stat
    ceph tell mds.* perf dump

---

### 11.3 RGW Performance Focus

RGW performance mainly focuses on:

- RGW instance count
- Reverse proxy performance
- HTTP connection count
- S3 request concurrency
- Small object count
- Large object upload/download
- RGW Pool performance
- OSD performance
- Network bandwidth

Common commands:

    ceph orch ps | grep rgw
    radosgw-admin bucket stats --bucket=<bucket-name>
    ceph -s
    ceph osd perf

---

## TwelveI don't know.Operational Complexity Comparison

| Type | Operational Complexity | Main Complexity Points |
|---|---|---|
| RBD | Medium | Image, snapshots, client mapping, CSI, locks |
| CephFS | Medium-High | MDS, metadata performance, multi-client sharing |
| RGW | Medium-High | S3 users, Bucket, Endpoint, gateway high availability, permissions |

From an operational perspective: /think
```

RBD is commonly used for block storage and is one of the most common usage methods of Ceph.
CephFS requires additional attention to MDS.
RGW requires additional attention to S3 API, users, Bucket, Endpoint, and reverse proxy.

---

## ThirteenI don't know.Experiment Environment Planning

Subsequent experiments still use an independent Ceph cluster:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation, optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Testing, optional |

Operating System:

    Ubuntu Server 22.04.5 LTS

Supplementary System:

    Rocky Linux 9

Experiment Boundary:

    Ceph modules are independent experiments, not dependent on MinIO, Longhorn, or RustFS.
    Ceph nodes do not directly occupy existing Kubernetes nodes.
    Kubernetes integration via CSI uses Ceph, and Ceph OSD is not directly deployed on K8s nodes.

---

## FourteenI don't know.Common Selection Errors

### 14.1 Using RBD as a Shared File System

Error:

    Multiple nodes mounting the same RBD and writing with ext4/xfs format.

Risk:

    File system damage
    Data inconsistency
    Application anomalies

Correct Approach:

    Single client block device usage for RBD.
    Multi-client shared directory usage for CephFS.

---

### 14.2 Using CephFS as a High-Performance Database Block Storage

Error:

    Placing the database's main data directory on CephFS just because it can be shared mounted.

Risk:

    Metadata and file system semantics may bring additional overhead.
    Databases are typically better suited for block devices.

Correct Approach:

    Prioritize RBD for the database's main data disk.
    Consider CephFS for shared file directories.

---

### 14.3 Mounting RGW as a Regular File System

Error:

    Believing that RGW Bucket is a directory that can be normally mounted.

Correct Understanding:

    RGW is an object storage system, accessed via S3 API.
    Object storage is not a POSIX file system.

---

### 14.4 Using Ceph for All Scenarios

Error:

    Using Ceph for all storage needs because Ceph is available.

Correct Understanding:

    Ceph is powerful but not suitable for all scenarios.
    Small object storage can consider MinIO.
    Small K8s block storage can consider Longhorn.
    Simple shared directories can consider NFS.
    Storage selection depends on business scenarios, scale, team capabilities, and operation costs.

---

## FifteenI don't know.Production Environment Selection Recommendations

### 15.1 Database Class Business

Priority Consideration:

    RBD

Reason:

    Databases typically require block device semantics.
    RBD can be used as a cloud disk.
    RBD supports snapshots, cloning, and CSI integration.

Note:

    Database performance strongly depends on OSD, network, disks, and pool policies.
    Performance testing must be done before deploying Ceph for production databases.

---

### 15.2 Shared Upload Directory

Priority Consideration:

    CephFS

Reason:

    Multiple Pods or nodes need to share a file directory.
    CephFS supports ReadWriteMany scenarios.

Note:

    Need to pay attention to MDS high availability and metadata performance.

---

### 15.3 Images, Attachments, Backup Archives

Priority Consideration:

    RGW or MinIO

If an existing Ceph cluster is available:

    RGW can be considered.

If only independent S3 object storage is needed:

    MinIO may be simpler.

---

### 15.4 Kubernetes PVC

Common Selection:

| PVC Requirement | Recommended |
|---|---|
| Single Pod Data Disk | RBD CSI |
| Multi-Pod Shared Directory | CephFS CSI |
| Application Object Storage | RGW S3 API, not via PVC |

---

## SixteenI don't know.Daily Troubleshooting Approach

### 16.1 RBD Issue Troubleshooting

Common Issues:

- PVC Pending
- Pod Mount Failure
- RBD Image Cannot Map
- RBD Image Locked
- Read/Write Performance Poor
- Snapshot Deletion Failure

Troubleshooting Entry:

    ceph -s
    rbd ls -p <pool>
    rbd info <pool>/<image>
    rbd status <pool>/<image>
    ceph osd pool get <pool> size
    ceph health detail

---

### 16.2 CephFS Issue Troubleshooting

Common Issues:

- CephFS Mount Failure
- Multi-Pod Shared Anomalies
- MDS Anomalies
- File Access Slow
- Performance Degradation from Large Number of Small Files

Troubleshooting Entry:

    ceph -s
    ceph fs status
    ceph mds stat
    ceph health detail
    ceph orch ps | grep mds

---

### 16.3 RGW Issue Troubleshooting

Common Issues:

- S3 Endpoint Unreachable
- AccessKey / SecretKey Error
- Bucket Permission Anomalies
- Upload/Download Slow
- RGW Service Anomalies
- Reverse Proxy Anomalies

Troubleshooting Entry:

    ceph -s
    ceph orch ps | grep rgw
    radosgw-admin user info --uid=<user>
    radosgw-admin bucket stats --bucket=<bucket>
    curl http://<rgw-endpoint>

---

## SeventeenI don't know.Interview Answer Approach

If asked in an interview:

    What are the differences between Ceph's RBD, CephFS, and RGW?

You can answer:

Ceph's underlying core is RADOS, which provides three common storage interfaces on top of RADOS: RBD, CephFS, and RGW.
RBD is block storage, similar to cloud disks, suitable for virtual machine disks, database data disks, and Kubernetes ReadWriteOnce PVCs. It is typically mounted as a block device for single client use.
CephFS is file storage, providing POSIX file system semantics, relying on MDS to manage metadata, suitable for multiple nodes or multiple Pods sharing the same directory, commonly used for ReadWriteMany PVCs in Kubernetes.
RGW is an object storage gateway, providing HTTP API compatible with S3, suitable for unstructured object data such as images, videos, backups, archives, and attachments.
In simple terms: use RBD like a disk, use CephFS like a shared directory, and use RGW like OSS/S3.
When selecting for production environments, consider access models, performance requirements, whether multiple clients share, whether S3 API is needed, team operations capabilities, and monitoring alert capabilities.

---

## Eighteen. Summary of This Article

This article mainly summarizes the three main storage types of Ceph:

1. RBD is block storage, suitable for cloud disks, databases, and Kubernetes ReadWriteOnce PVCs.
2. CephFS is file storage, suitable for shared directories across multiple nodes and Kubernetes ReadWriteMany PVCs.
3. RGW is an object storage gateway, suitable for S3 API, images, backups, archives, and other object scenarios.
4. All three rely on RADOS and OSD clusters at theBottom.
5. RBD focuses on Image, Snapshot, Clone, mounting, and locks.
6. CephFS focuses on MDS, metadata performance, and multi-client sharing.
7. RGW focuses on S3 users, Buckets, Endpoints, gateway high availability, and permissions.
8. Production selection cannot only consider Ceph's strong features but should choose the appropriate storage type based on business access models.
9. Advanced SREs must be able to explain the differences between the three and make reasonable selections based on business scenarios.

---

## Nineteen. Reference Documents

Ceph RBD Official Documentation:

    https://docs.ceph.com/en/reef/rbd/

CephFS Official Documentation:

    https://docs.ceph.com/en/reef/cephfs/

Ceph RGW Official Documentation:

    https://docs.ceph.com/en/reef/radosgw/

Ceph Architecture Documentation:

    https://docs.ceph.com/en/reef/architecture/

Ceph CSI GitHub:

    https://github.com/ceph/ceph-csi

Ceph Official Documentation Homepage:

    https://docs.ceph.com/