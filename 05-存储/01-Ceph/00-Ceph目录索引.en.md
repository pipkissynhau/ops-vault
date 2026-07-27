# Ceph Directory Index

Recommended Path: 05-Storage/01-Ceph/00-Ceph Directory Index.md

Tags: #Ceph #Distributed Storage #Block Storage #File Storage #Object Storage #SRE #Cloud-Native #Kubernetes

---

## I. Document Description

This document serves as a directory index for the advanced SRE storage module of Ceph, providing information on the learning objectives, experimental environment, note structure, reading sequence, experimental boundaries, and production methodology of the Ceph module.

Ceph is a general-purpose distributed storage platform that can offer the following on the same distributed storage infrastructure:

- Block Storage: RBD
- File Storage: CephFS
- Object Storage: RGW, compatible with S3 interfaces
- Underlying Distributed Object Storage: RADOS

The ultimate goal of this module is not just to learn how to install Ceph but to understand its architecture, deployment, operation and maintenance, fault recovery, performance optimization, security governance, and integration with Kubernetes from an advanced SRE perspective.

---

## II. Module Positioning

Ceph is the most core and complex module within the entire storage topic.

The current storage topic includes:

    05-Storage/
    ├── 01-Ceph
    ├── 02-MinIO
    ├── 03-Longhorn
    └── 04-RustFS

Among them, Ceph is positioned as the:

**Distributed Storage Foundation**

Its relationship with other modules can be understood as follows:

| Module | Main Role | Relationship with Ceph |
|---|---|---|
| Ceph | General-purpose Distributed Storage Platform | Provides the most comprehensive underlying capabilities, including block, file, and object storage |
| Longhorn | Kubernetes Cloud-Native Block Storage | More lightweight, designed for Kubernetes scenarios; understanding Ceph helps in grasping concepts related to replication and recovery |
| MinIO | S3 Object Storage | Focuses on object storage; shares some conceptual similarities with Ceph RGW |
| RustFS | New S3-compatible Object Storage | Can be used for comparative study alongside MinIO and Ceph RGW |

The recommended learning sequence is:

    1. Ceph
    2. Longhorn
    3. MinIO
    4. RustFS

The reason for this sequence is that Ceph covers core concepts such as replication, fault domains, data distribution, block storage, file storage, object storage, and recovery mechanisms in distributed storage. Mastering Ceph first makes it easier to understand other related modules.

---

## III. Learning Objectives

Upon completing the Ceph module, you should possess the following capabilities:

### 3.1 Architectural Understanding

You need to understand:

- Why Ceph is a distributed storage solution
- How Ceph can provide block, file, and object storage simultaneously
- What RADOS is
- The roles of MON, MGR, OSD, MDS, and RGW
- The functions of Pool, PG, and CRUSH
- How data is distributed across multiple OSDs
- The relationship between the number of replicas, fault domains, and data security
- Under what scenarios Backfill, Recovery, and Rebalance occur

---

### 3.2 Deployment Planning Skills

You need to master:

- Planning for multi-node Ceph clusters
- Selection of operating systems
- Disk planning
- Network planning
- Configuration of hostnames and DNS/hosts files
- Planning for MON/MGR/OSD nodes
- The cephadm deployment process
- Configuration of domestic software repositories
- Installation methods for Ubuntu 22.04.5 LTS and Rocky Linux 9
- Precautions regarding firewalls and SELinux

---

### 3.3 Operation and Maintenance Management Skills

You need to be able to:

- Check the health status of the cluster
- Monitor the status of OSDs
- Track the status of Pools
- Review the status of PGs
- Analyze capacity usage
- Manage the addition, removal, and replacement of OSDs
- Adjust Pool configurations and the number of replicas
- Manage RBD Images
- Administer the CephFS file system
- Handle RGW users and Buckets
- Interpret Dashboard and Prometheus metrics

---

### 3.4 Fault Diagnosis Skills

You need to be proficient in:

- Common causes for HEALTH_WARN alerts
- Troubleshooting OSD Down events
- Identifying issues with OSDs being Full or Nearfull
- Resolving PG stuck situations
- Dealing with degraded PGs
- Investigating slow operations
- Diagnosing Ceph failures caused by network issues
- Managing recovery processes due to disk abnormalities
- Understanding reasons for slow Backfill/Recovery operations
- Troubleshooting MDS anomalies
- Analyzing RGW access problems

---

### 3.5 Kubernetes Integration Skills

You need to know how to:

- Use Ceph RBD🔤 OSD recovery, backfilling, and rebalancing generate significant amounts of network traffic. If this traffic shares the same network as business access, it may affect client I/O operations.

---

## Section 5: Port Planning

Common ports used in Ceph include:

| Component | Port | Description |
|---|---|---|
| MON | 3300 | Messenger v2 |
| MON | 6789 | Messenger v1, compatible with legacy protocols |
| MGR Dashboard | 8443 | Dashboard HTTPS; specific configuration may vary |
| MGR Prometheus | 9283 | Prometheus metrics port |
| OSD | 6800-7300 | Range of OSD service ports |
| RGW | 7480 | Default RGW port; varies based on deployment |
| RGW Reverse Proxy | 80 / 443 | External object storage access; optional |
| SSH | 22 | Used by cephadm for node addition and management |

In a testing environment, you can temporarily disable or allow these firewall rules. In a production environment, only necessary ports should be exposed in accordance with the principle of minimal exposure. It is not recommended to disable security measures indiscriminately.

---

## Section 6: Domestic Software Source Strategies

For Ceph installations, it is advisable to use domestic sources to avoid slow downloads, installation failures, or unstable version availability when relying on foreign repositories.

### 6.1 Ubuntu Basic Sources

Ubuntu 22.04 should use the Alibaba Cloud Ubuntu repository:

    https://mirrors.aliyun.com/ubuntu/

Specific source configuration for `jammy` will be covered in subsequent deployment guides.

---

### 6.2 Rocky Linux Basic Sources

Rocky Linux 9 uses the Alibaba Cloud Rocky Linux repository:

    https://mirrors.aliyun.com/rockylinux/

Instructions on replacing the repository and using `dnf makecache` will also be provided in later deployment guides.

---

### 6.3 Ceph Software Source

Ceph utilizes the Alibaba Cloud Ceph mirror repository:

    https://mirrors.aliyun.com/ceph/

Source configuration guidelines:

Replace `download.ceph.com` mentioned in the official documentation with `mirrors.aliyun.com/ceph`.

Subsequent Ceph deployment guides will cover:

- Ubuntu apt source for Ceph
- Rocky Linux 9 dnf/yum source for Ceph

---

## Section 7: Structure of Ceph Module Notes

There are a total of 22 Ceph module notes:

    00-Ceph Directory Index.md
    01-Ceph Basics: Why Distributed Storage is Needed and Ceph's Overall Architecture.md
    02-Ceph Core Architecture: RADOS, MON, MGR, OSD, and CRUSH.md
    03-Ceph Storage Types: Differences Between RBD, CephFS, and RGW Object Storage.md
    04-Ceph Cluster Deployment Planning: Node, Disk, Network, and Fault Domain Design.md
    05-Ceph Cluster Deployment Practice: Basic cephadm Installation and Cluster Initialization.md
    06-Ceph OSD Management: Adding, Removing, Replacing Disks, and Capacity Expansion.md
    07-Ceph Pools and PGs: Understanding Data Distribution, Number of Replicas, and PG Status.md
    08-Ceph CRUSH Rules: Fault Domains, Data Placement, and Rack-Level High Availability.md
    09-Ceph RBD Block Storage: Images, Snapshots, Clones, and Common Operations.md
    10-CephFS File Storage: MDS, File System Creation, and Mounting Practices.md
    11-Ceph RGW Object Storage: S3 Compatibility, Users, Buckets, and Access Authentication.md
    12-Ceph Integration with Kubernetes: RBD CSI Dynamic Volume Provisioning Practice.md
    13-Ceph Integration with Kubernetes: CephFS CSI File Sharing Storage Practice.md
    14-Ceph Daily Operations: Common Commands for Checking Cluster Status, Capacity, OSDs, Pools, and PGs.md
    15-Ceph Troubleshooting: HEALTH_WARN, OSD Down, PG Issues, and Recovery Methods.md
    16-Ceph Data Recovery: OSD Replacement, PG Restoration, Backfilling, and Rebalancing md
    17-Ceph Performance Optimization: Disks, Networks, Pools, PGs, Clients, and BlueStore.md
    18-Ceph Security and Permissions: cephx, Users, Keyring, and Minimum Permission Control.md
    19-Ceph Monitoring and Alerts: Dashboard, Prometheus Metrics, and Capacity Alerts.md
    20-Ceph Production Practices: Capacity Planning, Fault Drills, Upgrades, and Operational Boundaries.md
    99-Ceph Phase Summary: From Architectural Understanding to Production/dev/sdc

It is not recommended to use the system disk directory to simulate a production OSD.

In a testing environment, additional virtual disks can be mounted via virtual machines, but each OSD should still correspond to an independent disk for ease of subsequent demonstrations:

- Adding an OSD
- Removing an OSD
- Replacing an OSD
- Disk failure
- Data recovery
- Backfilling
- Rebalancing

---

### 10.4 Using Domestic Repositories

For all installation notes related to Ceph modules, domestic repositories should be prioritized:

- Alibaba Cloud Ubuntu repository
- Alibaba Cloud Rocky Linux repository
- Alibaba Cloud Ceph repository

At the same time, the official documentation should also be referenced for easy verification of versions and parameters later on.

---

## Chapter Eleven: Core Topology Diagram of Ceph Modules

Test topology:

    ┌───────────────────────────────────────────────┐
    │                 10.0.0.0/24                    │
    └───────────────────────────────────────────────┘

        ┌──────────────────────┐
        │ ceph-node01          │
        │ 10.0.0.31            │
        │ MON / MGR / OSD      │
        │ /dev/sdb /dev/sdc    │
        └──────────┬───────────┘
                   │
        ┌──────────┼───────────┐
        │          │           │
        v          v           v
    ┌──────────────────────┐ ┌──────────────────────┐
    │ ceph-node02          │ │ ceph-node03          │
    │ 10.0.0.32            │ │ 10.0.0.33            │
    │ MON / MGR / OSD      │ │ MON / MGR / OSD      │
    │ /dev/sdb /dev/sdc    │ │ /dev/sdb /dev/sdc    │
    └──────────────────────┘ └──────────────────────┘
                   │
                   v
        ┌──────────────────────┐
        │ ceph-node04          │
        │ 10.0.0.34            │
        │ OSD / Expansion / Fault Simulation │
        │ /dev/sdb /dev/sdc    │
        └──────────────────────┘

        ┌──────────────────────┐
        │ ceph-client          │
        │ 10.0.0.35            │
        │ RBD / CephFS / RGW Testing│
        └──────────────────────┘

---

## Chapter Twelve: Ceph and Kubernetes Integration Topology

When a Ceph cluster is integrated with a Kubernetes cluster, the logical relationship is as follows:

    ┌─────────────────────────────┐
    │        Kubernetes Cluster       │
    │ 10.0.0.20 / 21 / 22         │
    │                             │
    │ StorageClass                │
    │ PVC                         │
    │ Pod                         │
    │ Ceph CSI                    │
    └──────────────┬──────────────┘
                   │
                   │ RBD CSI / CephFS CSI
                   v
    ┌─────────────────────────────┐
    │          Ceph Cluster           │
    │ 10.0.0.31 / 32 / 33 / 34    │
    │                             │
    │ MON / MGR / OSD             │
    │ Pool / RBD / CephFS         │
    └─────────────────────────────┘

Explanation:

- Kubernetes does not directly manage Ceph OSDs.
- Kubernetes utilizes Ceph's storage capabilities through the CSI mechanism.
- RBD is suitable for block storage that is exclusively used by a single Pod.
- CephFS is ideal for file storage shared among multiple Pods.

---

## Chapter Thirteen: Index of Common Commands

Detailed instructions will be provided in subsequent chapters, but here are the main command entries listed:

### 13.1 Cluster Status

    ceph -s
    ceph status
    ceph health detail

---

### 13.2 Nodes and Services

    ceph orch host ls
    ceph orch ps
    ceph orch ls

---

### 13.3 OSDs

    ceph osd tree
    ceph osd stat
    ceph osd df
    ceph osd status

---

### 13.4 Pools

    ceph osd pool ls
    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> pgOn top of RADOS, Ceph offers three common storage interfaces: RBD provides block storage, CephFS provides file storage, and RGW provides object storage compatible with S3. From an operational perspective, the core of Ceph lies not just in its installation but in understanding concepts such as OSDs, Pools, PGs, CRUSH, replication, fault domains, Backfill, and Recovery. In production environments, it is essential to focus on high availability of nodes, disk planning, network bandwidth, capacity management, OSD failure recovery, PG anomalies, monitoring and alerting systems, as well as upgrade risks.

When using Ceph in Kubernetes, it is typically integrated through the Ceph CSI interface. RBD is suitable for providing exclusive block storage for a single Pod, while CephFS is ideal for shared file storage across multiple Pods.

---

## Chapter Seventeen: References

Ceph Official Documentation:

    https://docs.ceph.com/

Ceph Official Architecture Documentation:

    https://docs.ceph.com/en/latest/architecture/

Cephadm Deployment Documentation:

    https://docs.ceph.com/en/latest/cephadm/install/

Ceph Hardware Recommendations:

    https://docs.ceph.com/en/reef/start/hardware-recommendations/

Ceph Official Website:

    https://ceph.io/

Aliyun Ubuntu Mirrors:

    https://developer.aliyun.com/mirror/ubuntu

Aliyun Rocky Linux Mirrors:

    https://developer.aliyun.com/mirror/rockylinux

Aliyun Ceph Mirrors:

    https://developer.aliyun.com/mirror/ceph