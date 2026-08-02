# Ceph Storage Types: Differences Between RBD, CephFS, and RGW Object Storage

Recommended Path: 05-Storage/01-Ceph/03-Ceph Storage Types: Differences Between RBD, CephFS, and RGW Object Storage.md

Tags: #Ceph #RBD #CephFS #RGW #Block Storage #File Storage #Object Storage #S3 #Kubernetes #SRE #Distributed Storage

---

## I. Document Overview

This article is the third in the advanced SRE storage module series for Ceph, focusing on the three primary storage services provided by Ceph:

- RBD: Block Storage
- CephFS: File Storage
- RGW: Object Storage

At its core, Ceph relies on RADOS. Regardless of whether the upper-layer service uses RBD, CephFS, or RGW, all data is ultimately stored as RADOS Objects on OSDs.

This article addresses the following key questions:

1. What are block storage, file storage, and object storage?
2. In what scenarios are RBD, CephFS, and RGW suitable?
3. How do these three differ in terms of architecture, access methods, Kubernetes integration, and operational complexity?
4. How should one choose between them in a production environment?
5. What considerations should advanced SRE professionals take when designing storage solutions?

---

## II. Why Can Ceph Offer Three Types of Storage

Ceph's core capability stems from RADOS.

RADOS is Ceph's underlying distributed object storage system, responsible for:

- Storing data objects
- Managing replicas
- Ensuring data recovery
- Managing OSDs
- Managing pools
- Mapping PGs
- Determining the placement of CRUSH data

On top of RADOS, Ceph provides various access interfaces:

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
    │              RADOS Distributed Storage Base            │
    └─────────────────────────────────────────────┘
                                │
                                v
    ┌─────────────────────────────────────────────┐
    │                  OSD Cluster                    │
    └─────────────────────────────────────────────┘

In other words:

    RADOS serves as the foundation.
    RBD, CephFS, and RGW are different access methods designed for various use cases.

---

## III. Overview of the Three Storage Types

| Storage Type | Ceph Component | Access Model | Typical Uses | Kubernetes Scenarios |
|---|---|---|---|---|
| Block Storage | RBD | Block Device | Cloud disks, database disks, virtual machine disks | ReadWriteOnce PVCs |
| File Storage | CephFS | File System | Shared directories, multi-node read/write | ReadWriteMany PVCs |
| Object Storage | RGW | S3 / Swift APIs | Images, backups, archives, unstructured data | Applications access via S3 APIs |

Quick Guidance:

- Choose RBD if you need a disk-like resource mounted to a single host or Pod.
- Select CephFS if multiple nodes need to share the same directory.
- Use RGW for accessing objects through HTTP/S3 APIs.

---

## IV. Block Storage: RBD

### 4.1 What is RBD

RBD stands for:

    RADOS Block Device

It is Ceph's block storage service that allows the storage resources within a Ceph cluster to be accessed as a single block device.

For clients, RBD behaves like a cloud disk:

    /dev/rbd0

Applications can create file systems on it, such as:

    ext4
    xfs

and then mount them for use:

    /mnt/data

---

### 4.2 RBD Data Flow

The simplified data flow of RBD is as follows:

    Application writes to the file system
        |
        v
    File system writes to the block device
        |
        v
    RBD Image
        |
        v
    RBD objects are divided into chunks
        |
        v
    These chunks are stored across multiple OSDs

Diagram:

    ┌────────────────────┐
    rbd create demo-image --size 10G -p rbd

View RBD Image information:

    rbd info rbd/demo-image

Create a snapshot:

    rbd snap create rbd/demo-image@snap01

View the snapshot:

    rbd snap ls rbd/demo-image

Delete the snapshot:

    rbd snap rm rbd/demo-image@snap01

Delete the RBD Image:

    rbd rm rbd/demo-image

---

## Section 5: File Storage: CephFS

### 5.1 What is CephFS

CephFS is a distributed file system provided by Ceph.

It offers access methods similar to traditional file systems, including:

- Directories
- Files
- Permissions
- Inodes
- Metadata
- Mount points

Clients can mount CephFS to a local directory, such as `/mnt/cephfs`, and then read and write files just like using a regular file system.

---

### 5.2 The Core Component of CephFS: MDS

CephFS relies on MDS (Metadata Server).

MDS is responsible for managing the metadata of CephFS, including:

- File names
- Directory structures
- Inodes
- Permissions
- File attributes
- File locks
- Metadata caching

In CephFS, data and metadata are typically stored separately:

| Type | Storage Location |
|---|---|
| File Data | CephFS Data Pool |
| File Metadata | CephFS Metadata Pool |
| Metadata Service | MDS |

Schematic representation:

    ┌────────────────────┐
    │     CephFS Client   │
    └─────────┬──────────┘
              │
              v
    ┌────────────────────┐
    │        MDS          │  Handles metadata such as directories, file names, and inodes
    └─────────┬──────────┘
              │
              v
    ┌────────────────────┐
    │     RADOS Pools     │
    │ metadata + data     │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │       OSD Cluster      │
    └────────────────────┘

---

### 5.3 Scenarios Suitable for CephFS

CephFS is suitable for:

- Sharing directories across multiple nodes
- Shared read and write operations among multiple Pods
- Use with ReadWriteMany PVCs
- Sharing configuration directories
- Sharing upload directories
- Sharing data for AI and big data applications
- Multi-client file access
- Replacing certain NFS use cases

Typical scenarios include:

- Multiple Web Pods needing to share an upload directory.
- Multiple computing nodes requiring access to the same training datasets.
- Multiple application instances sharing file-based data.

---

### 5.4 Scenarios Unsuitable for CephFS

CephFS may not be suitable for:

- Database primary data disks that require extremely high IOPS.
- Small-scale, simple shared directories where NFS is already sufficient.
- Scenarios where metadata performance is critical and there is no ability to optimize it.
- Environments without experience in managing MDS.
- Environments where monitoring MDS status is not preferred.

The reason for this is that CephFS includes an additional MDS component compared to RBD, and the performance of the file system's metadata is closely related to the status of MDS. Additionally, handling a large number of small files or high-concurrency directory operations can put significant strain on MDS.

---

### 5.5 CephFS and Kubernetes

In Kubernetes, CephFS is typically integrated through CephFS CSI (Ceph File System Interface).

CephFS is particularly suitable for scenarios involving multiple Pods sharing the same PVC.

Typical setup process:

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
    Multiple Pods mount to the same shared directory

Schematic representation:

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

View the file system:

    ceph fs ls

Check the status of CephFS:

       ceph osd pool ls | grep rgw

---

## Section 7: Key Differences Between RBD, CephFS, and RGW

### 7.1 Differences in Access Methods

| Type          | Access Method         |
|---------------|---------------------------|
| RBD           | Block Device              |
| CephFS        | File System Mounting     |
| RGW           | HTTP / S3 API            |

Corresponding interpretations:

    RBD is similar to cloud block storage.
    CephFS functions like a shared file system.
    RGW operates like a private OSS/S3 service.

---

### 7.2 Differences in Data Models

| Type          | Data Model                |
|---------------|---------------------------|
| RBD           | Block                     |
| CephFS        | Files and Directories     |
| RGW           | Bucket and Object         |

---

### 7.3 Differences in Clients

| Type          | Common Clients             |
|---------------|-------------------------------|
| RBD           | Linux rbd, QEMU, Kubernetes CSI, OpenStack |
| CephFS        | Linux kernel client, ceph-fuse, Kubernetes CSI |
| RGW           | S3 SDK, awscli, s3cmd, mc, HTTP API |

---

### 7.4 Differences in Usage with Kubernetes

| Type          | How to Use in Kubernetes | Access Mode                |
|---------------|---------------------------------|-------------------------|
| RBD           | RBD CSI StorageClass         | ReadWriteOnce            |
| CephFS        | CephFS CSI StorageClass         | ReadWriteMany            |
| RGW           | Accessed through S3 API directly | Does not use PVCs          |

Note:

    RGW is typically not used via PVCs.
    It is accessed directly through business code or configuration using the S3 Endpoint.

---

### 7.5 Differences in Operational Concerns

| Type          | Main Operational Considerations |
|---------------|-------------------------------------------|
| RBD           | Image management, snapshots, cloning, mapping, locking, single-client mounting |
| CephFS        | MDS performance, metadata handling, multi-client sharing, directory operations |
| RGW           | S3 user management, Bucket configuration, Policy settings, Endpoint reliability, gateway high availability |

---

## Section 8: How to Choose Between the Three Types in Production Scenarios

### 8.1 Scenarios for Choosing RBD

Choose RBD when:

    You need a data volume similar to cloud block storage.

Suitable for:

- Database data disks
- Stateful applications in Kubernetes
- Virtual machine disks
- Persistence for single-instance applications
- Block devices that require snapshots and cloning

Examples:

    MySQL Pods use RBD PVCs.
    KubeVirt virtual machines use RBD as system disks.
    OpenStack virtual machine cloud disks use Ceph RBD as backend.

---

### 8.2 Scenarios for Choosing CephFS

Choose CephFS when:

    You need multiple clients to share the same file system directory.

Suitable for:

- Multiple Pods sharing an upload directory
- Multi-node data sets shared among clients
- File-based services that require shared access
- ReadWriteMany PVCs
- Situations where NFS can be partially replaced

Examples:

    Multiple Web Pods share the /uploads directory.
    Multiple AI tasks read from the same data set directory.
    Multiple application instances share a static file directory.

---

### 8.3 Scenarios for Choosing RGW

Choose RGW when:

    You need object storage capabilities and access through the S3 API.

Suitable for:

- Storing images, videos, and archives
- Performing backup and logging tasks
- Using it as application attachments
- Providing a private object storage service

Examples:

    Applications write uploaded files to S3 Buckets.
    Backup systems upload backup files to object storage.
    Log archives are stored in object storage.

---

### 8.4 Simple Selection Guidelines

You can make the choice based on the following criteria:

    Use it like a disk: RBD
    Use it like a shared directory: CephFS
    Use it like OSS/S3: RGW

More specifically:

    For single Pod/single VM mounting: RBD
    For multi-Pod shared read/write: CephFS
    For applications that store objects through HTTP API: RGW

---

## Section 9: Architecture Topologies of the Three Types

### 9.1 RBD Topology

    ┌────────────────────┐
    │ VM / Pod / Host     │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │      RBD Image       │
    └─────────┬──────────┘
              v
    ┌────────────────────┐
    │      RADOS Pool        │
    └─────────┬──────────┘
### rbd perf image iostat

---

### 11.2 CephFS Performance Focus Areas

The main performance considerations for CephFS include:

- MDS performance
- Number of metadata operations
- Number of small files
- Directory depth
- Data Pool performance
- Metadata Pool performance
- Number of clients
- MDS cache

Common commands:

    ceph fs status
    ceph mds stat
    ceph tell mds.* perf dump

---

### 11.3 RGW Performance Focus Areas

The main performance considerations for RGW include:

- Number of RGW instances
- Reverse proxy performance
- HTTP connections
- S3 request concurrency
- Number of small objects
- Large object upload and download
- RGW Pool performance
- OSD performance
- Network bandwidth

Common commands:

    ceph orch ps | grep rgw
    radosgw-admin bucket stats --bucket=<bucket-name>
    ceph -s
    ceph osd perf

---

## XII. Comparison of Operational Complexity

| Type | Operational Complexity | Main Challenges |
|---|---|---|
| RBD | Medium | Image management, snapshots, client mapping, CSI, locking |
| CephFS | Medium to High | MDS performance, metadata handling, multi-client sharing |
| RGW | Medium to High | S3 API management, user accounts, Buckets, Endpoints, high availability of gateways, permissions |

From an operational perspective:

    RBD is commonly used for block storage and is one of the most prevalent applications of Ceph.
    With CephFS, additional attention should be paid to MDS performance.
    For RGW, managing S3 APIs, users, Buckets, Endpoints, and ensuring high availability of gateways is crucial.

---

## XIII. Experimental Environment Setup

For subsequent experiments, an independent Ceph cluster will continue to be used:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Scaling / Fault tolerance testing, optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW testing, optional |

Operating System:

    Ubuntu Server 22.04.5 LTS

Additional Systems:

    Rocky Linux 9

Experimental Constraints:

    Ceph modules will be tested independently without relying on MinIO, Longhorn, or RustFS.
    Ceph nodes will not directly occupy existing Kubernetes nodes.
    The Kubernetes integration will use CSI to interact with Ceph, and Ceph OSDs will not be deployed directly on Kubernetes nodes.

---

## XIV. Common Misconceptions

### 14.1 Using RBD as a Shared File System

Misconception:

    Multiple nodes mount the same RBD volume simultaneously and write data to it using conventional ext4/xfs methods.

Risks:

    File system corruption
    Data inconsistency
    Application failures

Correct approach:

    Use RBD for block device storage on a single client.
    Use CephFS for shared directories among multiple clients.

---

### 14.2 Using CephFS as High-Performance Block Storage for Databases

Misconception:

    Storing database primary data in CephFS just because it supports shared mounting.

Risks:

    Additional overhead due to metadata and file system semantics.
    Databases generally perform better on block devices.

Correct approach:

    Prioritize RBD for database primary data disks.
    Consider using CephFS for shared file directories.

---

### 14.3 Mounting RGW as a Regular File System

Misconception:

    Assuming that an RGW Bucket can be mounted directly like a regular directory.

Correct understanding:

    RGW is an object storage system that uses the S3 API for access.
    Object storage does not follow POSIX file system semantics.

---

### 14.4 Using Ceph for All Storage Scenarios

Misconception:

    Assuming that Ceph can meet all storage needs without considering other options.

Correct understanding:

    Although Ceph is powerful, it may not be the best choice for every scenario.
    For small object storage, MinIO might be more suitable.
    For small-scale Kubernetes block storage, Longhorn could be a better option.
    For simple shared directories, NFS could suffice.
    The choice of storage solution should depend on specific business requirements, scale, team capabilities, and operational costs.

---

## XV. Production-Grade Selection Recommendations

### 15.1 Database-Related Applications

Recommended Choice:

    Rhttps://github.com/ceph/ceph-csi

Official Ceph Documentation Homepage:

https://docs.ceph.com/