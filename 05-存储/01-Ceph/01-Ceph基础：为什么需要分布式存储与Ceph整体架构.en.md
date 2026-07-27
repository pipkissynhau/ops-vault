# Ceph Basics: Why Distributed Storage is Needed and the Overall Architecture of Ceph

# Ceph Basics: Why Distributed Storage is Needed and the Overall Architecture of Ceph

Recommended Path: 05-Storage/01-Ceph/01-Ceph Basics: Why Distributed Storage is Needed and the Overall Architecture of Ceph.md

Tags: #Ceph #Distributed Storage #RADOS #OSD #MON #MGR #Block Storage #File Storage #Object Storage #SRE #Cloud-Native #Kubernetes

---

## I. Document Overview

This article serves as the first foundational note in the advanced SRE storage module for Ceph, focusing on answering three key questions:

1. Why is distributed storage necessary?
2. What problems does Ceph address?
3. How should one understand the overall architecture of Ceph?

This article does not delve directly into installation and deployment but rather establishes a comprehensive understanding of Ceph first. Subsequent topics such as Ceph deployment, OSD management, Pool/PG, CRUSH, RBD, CephFS, RGW, Kubernetes CSI integration, troubleshooting, and performance optimization will all be built upon this foundation.

The main thread of this article is:

    Issues with Single-Server Storage
        |
        v
    The Value of Distributed Storage
        |
        v
    Ceph's Overall Positioning
        |
        v
    The RADOS Foundation
        |
        v
    The Three Capabilities of RBD / CephFS / RGW
        |
        v
    The Operational Boundaries of Ceph from an Advanced SRE Perspective

---

## II. Why Distributed Storage is Needed

### 2.1 Common Issues with Single-Server Storage

In traditional environments, business data is usually stored on the local disks of a single server, for example:

    /data
    /var/lib/mysql
    /mnt/app-data
    /opt/application/upload

This approach is simple and direct but comes with significant drawbacks.

Common risks include:

- Data loss due to single-disk failure
- Business disruption caused by server downtime
- Difficulty in scaling up when disk space runs out
- Inability to share data across multiple nodes
- Limited performance and capacity of local disks
- Complexity in data migration during business relocations
- Local data becoming unavailable after Kubernetes Pods move around
- Dependence on additional systems for backup and recovery
- Lack of built-in support for multi-replica mechanisms and automatic recovery

Typical scenarios:

    MySQL runs on a single server with its data directory located at /data/mysql.
    If this server's disk fails, the MySQL data may be lost immediately.
    If the server crashes, business operations must wait until it restarts.
    Moving data to another server requires additional steps such as copying, verifying, and switching services.

---

### 2.2 What RAID Solves, and What It Doesn't

Many traditional environments use RAID to enhance the reliability of single-disk systems.

For example:

| RAID Type | Function |
|-------------|-----------|
| RAID1      | Provides disk redundancy |
| RAID5      | Balances capacity and error checking |
| RAID10     | Combines performance and redundancy |

RAID can help address some disk failure issues but not all of them.

RAID cannot solve:

- Total server failures
- Outages in the data center
- Horizontal expansion of storage nodes
- Sharing storage resources across multiple services
- Dynamic volume provisioning in Kubernetes
- Automatic cross-node data recovery
- Distributed placement of multiple replicas
- Providing a unified interface for block, file, and object storage

In other words:

    RAID addresses internal disk redundancy within a single machine.
    Distributed storage solves the broader problems related to managing multiple nodes, disks, replicas, and access interfaces across a network.

---

### 2.3 What Problems Distributed Storage Solves

The core idea of distributed storage is to:

    Distribute data across multiple servers and disks, and use mechanisms such as replication, erasure coding, scheduling algorithms, and health checks to enhance capacity, reliability, and scalability.

Distributed storage typically addresses the following issues:

- Horizontal scaling
- High availability across multiple nodes
- Data replication
- Automatic fault recovery
- Unified management of storage capacity
- Support for multi-client access
- Pooling of storage resources
- Decoupling storage from computing resources
- Meeting the dynamic storage needs of cloud platforms and Kubernetes

Simplified illustration:

    Traditional Local Storage:

        App -> Local Disk

    Distributed Storage:

        App -> Storage Client -> Distributed Storage Cluster -> Multiple Nodes and Disks

---

## III. What is Ceph

Ceph is an open-source distributed storage system. Its key features include:

    It consists of a底层 distributed storage cluster that simultaneously provides block storage, file storage, and object storage capabilities.

Ceph can offer:

| Capability | Component/Interface | Typical Use Cases |
|-------------|-----------------|--------------------------|
| Block Storage | RBD      └── Copy 3 -> ceph-node03 / OSD.4

If the disk of ceph-node02 is damaged, Ceph will restore the copy on another OSD based on the cluster status and CRUSH rules.

This is one of the core values of Ceph:

    It's not about preventing failures, but about being able to detect, isolate, and recover from them after they occur.

---

### 5.3 Horizontal Scaling

Traditional single-machine disk scaling usually requires:

- Shutting down the machine to add disks
- Expanding the file system
- Moving data
- Adjusting mount points
- Modifying service configurations

Ceph's approach to scaling is:

    Add new nodes
        |
        v
    Add new OSD disks
        |
        v
    Ceph detects the new capacity
        |
        v
    CRUSH recalculates data distribution
        |
        v
    Data is gradually rebalanced

This makes Ceph more suitable for scenarios where capacity needs to grow continuously.

---

### 5.4 Support for Multiple Storage Interfaces

A single Ceph cluster can provide multiple types of storage interfaces:

| Type | Ceph Capability | Equivalent |
|---|---|---|
| Block Storage | RBD | Cloud block storage |
| File Storage | CephFS | NFS-like shared file systems |
| Object Storage | RGW | S3/OSS-style object storage |

This is very valuable for platform-based environments:

    Virtual machines need block storage
    Kubernetes requires PVCs
    Multi-node applications need shared file systems
    Backup, images, and archives require object storage

Ceph can provide all these capabilities on a unified foundation.

---

## Six, Overview of Ceph's Core Components

### 6.1 RADOS

RADOS is Ceph's underlying distributed object storage system.

It handles:

- Object storage
- Data distribution
- Data replication
- Data recovery
- Cluster expansion
- Cluster self-management

In other words:

    RADOS is the foundation of Ceph.

RBD, CephFS, and RGW are all built on top of RADOS.

---

### 6.2 OSD

OSD stands for Object Storage Daemon.

OSDs are the processes in Ceph that actually store data.

Each OSD typically corresponds to a disk or storage device.

OSDs are responsible for:

- Storing objects
- Handling read and write requests
- Participating in copy replication
- Participating in recovery
- Reporting heartbeats
- Reporting capacity and status
- Synchronizing data with other OSDs

A key point to understand:

    Without OSDs, there would be no actual data storage.
    Ceph's total capacity comes primarily from its OSDs.
    Ceph's performance also heavily depends on the number of OSDs, their disk performance, and network capabilities.

---

### 6.3 MON

MON stands for Monitor.

MON is responsible for maintaining critical mapping information in the Ceph cluster, such as:

- Monitor Map
- OSD Map
- PG Map
- CRUSH Map
- MDS Map

MON is not used to store business data; it primarily ensures the cluster's status and consistency.

Production recommendations:

    MONs are usually deployed in odd numbers, such as 3 or 5.
    In experimental environments, 3 MONs are commonly used.

The reason for this is that MONs need to use a majority voting mechanism to ensure consistency. An even number of MONs does not offer any clear advantages and may make the arbitration design more complex.

---

### 6.4 MGR

MGR stands for Manager.

MGR is responsible for managing and monitoring the Ceph cluster.

Common capabilities include:

- Dashboard
- Prometheus Exporter
- Cluster metrics
- Orchestration interfaces
- Management modules
- Alerting systems

In the cephadm deployment, MGR also plays a crucial role in orchestration and management.

Production recommendations:

    At least 2 instances of MGR should be deployed, one active and one standby.

---

### 6.5 MDS

MDS stands for Metadata Server.

MDS primarily serves CephFS.

Since CephFS is a file system, it needs to manage:

- Directories
- File names
- Inodes
- Permissions
- Metadata

All this metadata is managed by MDS.

Note:

    If only RBD or RGW are used, MDS may not be necessary.
    However, if CephFS is required, MDS must be deployed.

---

### 6.6 RGW

RGW stands for RADOS Gateway.

RGW provides an object storage interface that is compatible with S3/Swift APIs.

Typical uses include:

- Image storage
- Backup and archiving
- Object storage services
- Private S3 solutions
- Connecting applications through S3 SDKs

RGW shares some similarities with MinIO, but they serve different purposes. A comparison can help### 参考文档

- [Ceph 官方文档](https://docs.ceph.com/en/latest/)
- [Kubernetes CSI 文档](https://kubernetes.io/docs/concepts/storage/csi/)
- [MinIO 文档](https://minio.io/docs/)
- [Longhorn 文档](https://longhorn.io/docs/)Ceph Official Architecture Documentation:

https://docs.ceph.com/en/reef/architecture/

Ceph Official Technical Introduction:

https://ceph.io/en/discover/technology/

Ceph Official RBD Documentation:

https://docs.ceph.com/en/reef/rbd/

Ceph Official Glossary:

https://docs.ceph.com/en/latest/glossary/

Ceph Rados Command Documentation:

https://docs.ceph.com/en/latest/man/8/rados/

Ceph Official Documentation Home Page:

https://docs.ceph.com/