# Ceph Directory Index

Recommended path: 05-Storage/01-Ceph/00-Ceph Directory Index.md

Tags: #Ceph #DistributedStorage #BlockStorage #FileStorage #ObjectStorage #SRE #Clouds. #Kubernetes

---

## I. Document Explanation

This document serves as a directory index for the Ceph Advanced SRE Storage Module, explaining the learning objectives, experimental environment, note structure, reading order, experimental boundaries, and production methodology for the Ceph module.

Ceph is a general-purpose distributed storage platform that can provide:

- Block storage: RBD
- File storage: CephFS
- Object storage: RGW (S3 interface compatible)
- Underlying distributed object storage: RADOS

This module does not aim to "install Ceph" as its final goal, but rather to understand Ceph's architecture, deployment, operations, fault recovery, performance optimization, security governance, and Kubernetes integration from an advanced SRE perspective.

---

## II. Module Positioning

Ceph is the most core and complex module in the entire storage topic.

Current storage topics include:

    05-Storage/
    ├── 01-Ceph
    ├── 02-MinIO
    ├── 03-Longhorn
    └── 04-RustFS

Ceph's positioning is:

    Distributed storage foundation

The relationship between this module and others can be understood as:

| Module | Main Positioning | Relationship with Ceph |
|---|---|---|
| Ceph | General-purpose distributed storage platform | Most completeBottom capability, includes block, file, object |
| Longhorn | Kubernetes cloud-native block storage | Lighter, more K8s-focused, easier to understand replication and recovery after understanding Ceph |
| MinIO | S3 object storage | Object storage-focused, has some conceptual parallels with Ceph RGW |
| RustFS | New S3-compatible object storage | Can be compared and understood with MinIO and Ceph RGW |

Recommended learning order:

    1. Ceph
    2. Longhorn
    3. MinIO
    4. RustFS

Reasons:

    Ceph covers core concepts in distributed storage such as replication, fault domain, data distribution, block storage, file storage, object storage, and recovery mechanisms.
    After mastering Ceph, learning Longhorn, MinIO, and RustFS will be easier to build a systematic understanding.

---

## III. Learning Objectives

After completing the Ceph module, you should possess the following capabilities:

### 3.1 Architecture Understanding Ability

Need to understand:

- Why Ceph is a distributed storage system
- Why Ceph can provide block, file, and object storage simultaneously
- What is RADOS
- What responsibilities do MON, MGR, OSD, MDS, RGW have
- What are the roles of Pool, PG, CRUSH
- How data is distributed across multiple OSDs
- The relationship between replica count, fault domain, and data security
- When Backfill, Recovery, Rebalance occur

---

### 3.2 Deployment Planning Ability

Need to master:

- Multi-node Ceph cluster planning
- Operating system selection
- Disk planning
- Network planning
- Hostname and DNS / hosts planning
- MON / MGR / OSD node planning
- cephadm deployment process
- Domestic software source configuration
- Ubuntu 22.04.5 LTS installation method
- Rocky Linux 9 installation method
- Firewall and SELinux considerations

---

### 3.3 Operations Management Ability

Need to master:

- Checking cluster health status
- Checking OSD status
- Checking Pool status
- Checking PG status
- Checking capacity usage
- Managing OSD addition, removal, replacement
- Managing Pool and replica count
- Managing RBD Image
- Managing CephFS file system
- Managing RGW users and Bucket
- Checking Dashboard and Prometheus metrics

---

### 3.4 Fault Diagnosis Ability

Need to master:

- Common causes of HEALTH_WARN
- OSD Down troubleshooting
- OSD Full / Nearfull troubleshooting
- PG stuck troubleshooting
- PG degraded troubleshooting
- Slow ops troubleshooting
- Ceph failures caused by network anomalies
- Recovery process caused by disk anomalies
- Reasons for slow Backfill / Recovery
- MDS anomaly troubleshooting
- RGW access anomaly troubleshooting

---

### 3.5 Kubernetes Integration Ability

Need to master:

- Ceph RBD CSI dynamic volume provisioning
- CephFS CSI file sharing storage
- StorageClass configuration
- Secret configuration
- PVC / PV creation
- Pod mounting verification
- Use case differences between RBD and CephFS
- Troubleshooting methods for using Ceph in Kubernetes

---

### 3.6 Advanced SRE Methodology

Need to consistently apply the following methodologies:

- Node high availability
- Data replication
- Fault domain design
- OSD replacement
- Replica recovery
- Backfill and Rebalance control
- Capacity planning
- Performance optimization
- Security authentication
- Principle of least privilege
- Monitoring and alerts
- Upgrade and rollback risks
- Production boundary identification

---

## IV. Experimental Environment Overview

### 4.1 Subnet Planning

This document uses a virtual machine experiment subnet:

    10.0.0.0/24

Existing environment:

| Address | Hostname | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab / Jenkins / Harbor / Image transit |
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Ceph experiments do not directly use existing K8s nodes to avoid affecting the existing Kubernetes environment.

---

### 4.2 Ceph Node Planning

Ceph main experiment node planning is as follows:

| IP | Hostname | Role | Notes |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD | Initial bootstrap node |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD | Storage node |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD | Storage node |
| 10.0.0.34 | ceph-node04 | OSD (optional) | Expansion, fault simulation node |
| 10.0.0.35 | ceph-client | Client (optional) | RBD / CephFS / RGW testing |

Minimum experimental recommendation:

    3 Ceph nodes, each node with at least 1 system disk + 1 independent data disk.

Recommended experiment: /think

3 Ceph nodes, each with 1 system disk + 2 independent data disks.

Expansion experiment:

    4 Ceph nodes for demonstrating OSD expansion, node scaling, and fault recovery.

---

### 4.3 Operating System Planning

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Supplementary production reference system:

    Rocky Linux 9

Reasons:

    Ubuntu 22.04.5 LTS remains consistent with the current experiment environment, suitable for quick setup and reproducibility.
    Rocky Linux 9 belongs to the RHEL family, commonly used in production environments, requiring additional notes on dnf/yum sources, firewalld, SELinux, Podman, etc.

Subsequent Ceph deployment sections will cover:

| System | Package Manager | Key Points |
|---|---|---|
| Ubuntu 22.04.5 LTS | apt | Aliyun Ubuntu source, Aliyun Ceph source, cephadm |
| Rocky Linux 9 | dnf/yum | Aliyun Rocky source, Aliyun Ceph source, Podman, firewalld, SELinux |

---

### 4.4 Basic Hostname Planning

All Ceph nodes are recommended to configure `/etc/hosts` uniformly:

    10.0.0.31 ceph-node01
    10.0.0.32 ceph-node02
    10.0.0.33 ceph-node03
    10.0.0.34 ceph-node04
    10.0.0.35 ceph-client

Subsequent deployments should prioritize using hostnames instead of hardcoding IPs.

---

### 4.5 Disk Planning

Each Ceph node is recommended to prepare at least:

| Disk | Purpose |
|---|---|
| /dev/sda | Operating system disk |
| /dev/sdb | Ceph OSD data disk |
| /dev/sdc | Ceph OSD data disk (optional) |

Experimental environment example:

    ceph-node01:
      /dev/sda  System disk
      /dev/sdb  OSD.0
      /dev/sdc  OSD.1, optional

    ceph-node02:
      /dev/sda  System disk
      /dev/sdb  OSD.2
      /dev/sdc  OSD.3, optional

    ceph-node03:
      /dev/sda  System disk
      /dev/sdb  OSD.4
      /dev/sdc  OSD.5, optional

Notes:

    Ceph OSD recommends using independent raw disks.
    It is not recommended to mix system disks with OSD data disks.
    Virtual disks can be used in experimental environments, but production environments require careful planning of disk types, capacity, IOPS, fault domains, and replacement procedures.

---

### 4.6 Network Planning

Ceph commonly has the following network types:

| Network Type | Description |
|---|---|
| Public Network | Client access to Ceph, MON communication, cluster basic communication |
| Cluster Network | OSD communication for replication, recovery, Backfill |

Experimental environment can initially use a single network interface:

    10.0.0.0/24

Production environment recommends separation:

    Public Network: Client access, MON, MGR, RGW
    Cluster Network: OSD replication, recovery, Backfill

Reasons:

    OSD recovery, Backfill, and Rebalance generate significant network traffic.
    Sharing the network with business access may affect client IO.

---

## Five. Port Planning

Common Ceph ports are as follows:

| Component | Port | Description |
|---|---|---|
| MON | 3300 | Messenger v2 |
| MON | 6789 | Messenger v1, old protocol compatibility |
| MGR Dashboard | 8443 | Dashboard HTTPS, specific to configuration |
| MGR Prometheus | 9283 | Prometheus metrics port |
| OSD | 6800-7300 | OSD service port range |
| RGW | 7480 | RGW default port, varies by deployment configuration |
| RGW Reverse Proxy | 80 / 443 | External object storage entry point, optional |
| SSH | 22 | Used by cephadm for node addition and management |

Experimental environments can initially disable or allow firewall rules.

Production environments need to expose ports according to the minimal exposure principle, and it is not recommended to disable security policies indiscriminately.

---

## Six. Domestic Software Source Strategy

Ceph modules require using domestic sources to avoid slow installation, failure, or unstable version retrieval caused by direct access to foreign sources.

### 6.1 Ubuntu Base Source

Ubuntu 22.04 uses Aliyun Ubuntu source:

    https://mirrors.aliyun.com/ubuntu/

Subsequent deployment sections will write `jammy` related source configurations.

---

### 6.2 Rocky Linux Base Source

Rocky Linux 9 uses Aliyun Rocky Linux source:

    https://mirrors.aliyun.com/rockylinux/

Subsequent deployment sections will write Rocky 9's repo replacement method and `dnf makecache`.

---

### 6.3 Ceph Software Source

Ceph uses Aliyun Ceph mirror source:

    https://mirrors.aliyun.com/ceph/

Source configuration principles:

    Replace the official documentation's download.ceph.com with mirrors.aliyun.com/ceph.

Subsequent Ceph deployment sections will cover separately:

    Ubuntu apt Ceph source
    Rocky Linux 9 dnf/yum Ceph source

---

## Seven. Ceph Module Notes Structure

The Ceph module has 22 articles: /think

## Eight, Recommended Reading Order

### First Stage: Understanding What Ceph Is

Recommended Reading:

    01-CephFoundation: Why needs distributed storage andCephOverall structure.md
    02-CephCore structure:RADOSI don't know.MONI don't know.MGRI don't know.OSDandCRUSH.md
    03-CephStorage type:RBDI don't know.CephFSI don't know.RGWObject Storage Distinction.md

Goals:

    First understand Ceph's overall positioning, core components, and three storage interfaces.
    Don't rush deployment; first establish the architecture relationships.

---

### Second Stage: Deployment and Cluster Initialization

Recommended Reading:

    04-CephCluster deployment planning: nodes, diskettes, network and fault field design.md
    05-CephCluster deployment practices:cephadmBasic installation and cluster initialization.md

Goals:

    Be able to complete a basic Ceph cluster deployment in a VM / bare-metal multi-node environment.
    Clearly understand the differences in source configuration and installation between Ubuntu and Rocky Linux 9.

---

### Third Stage: OSD, Pool, PG, CRUSH

Recommended Reading:

    06-Ceph OSDManagement: Add, remove, replace and expand disks.md
    07-Ceph PoolandPGData distribution, number of copies,PGQuantity and status understanding.md
    08-Ceph CRUSHRules: failed domain, data placement and shelf level available.md

Goals:

    Understand how Ceph data is written to disks, how it is distributed, and how it ensures high availability of replicas.
    Be able to interpret OSD Tree, Pool, PG status and CRUSH rules.

---

### Fourth Stage: Three Storage Interface Practices

Recommended Reading:

    09-Ceph RBDBlock storage:ImageI don't know.SnapshotI don't know.CloneWith common operations.md
    10-CephFSFile storage:MDS, Document System Creation and Mounting Practice.md
    11-Ceph RGWObject storage:S3Compatibility, users,BucketCheck with Access.md

Goals:

    Be able to independently use Ceph's block, file, and object storage capabilities.
    Understand the usage scenario differences between RBD, CephFS, and RGW.

---

### Fifth Stage: Kubernetes Integration

Recommended Reading:

    12-CephandKubernetesMatch:RBD CSIDynamic volume supply practice.md
    13-CephandKubernetesMatch:CephFS CSIFile-sharing storage practices.md

Goals:

    Master Ceph's basic methods as a persistent storage backend in Kubernetes.
    Understand the differences between RBD CSI and CephFS CSI.

---

### Sixth Stage: Operations, Troubleshooting, and Recovery

Recommended Reading:

    14-CephDaily operations: cluster status, capacity,OSDI don't know.PoolandPGCommon commands.md
    15-CephFault check:HEALTH_WARNI don't know.OSD DownI don't know.PGUnusual and restored.md
    16-CephData recovery:OSDReplace,PGRecovery,BackfillandRebalanceUnderstood..md

Goals:

    Be able to interpret Ceph cluster health status.
    Be able to handle common OSD, PG, capacity, and recovery-related failures.
    Understand the risks and performance impacts during data recovery.

---

### Seventh Stage: Performance, Security, Monitoring, and Production Practices

Recommended Reading:

    17-CephPerformance optimization: disks, networks,PoolI don't know.PGClients andBlueStore.md
    18-CephSecurity and authority:cephxUser,KeyringWith minimum permission control.md
    19-CephWatch and report:DashboardI don't know.PrometheusIndicators and capacity alerts.md
    20-CephProduction practices: capacity planning, malfunctioning exercises, upgrading and transport boundary.md
    99-CephPhase summary: from structural understanding to production transport.md

Goals:

    Elevate from "able to deploy" to "able to operate, troubleshoot, optimize, and assess production risks."

---

## Nine, Uniform Structure for Each Note

Subsequent notes should follow a uniform structure:

    # Title

    Recommended path: xxx

    Tags: xxx

    ---

    ## One, Document Description

    ## Two, Why This Capability Is Needed

    ## Three, Core Concepts

    ## Four, Architecture or Working Principles

    ## Five, Experiment Environment Planning

    ## Six, Operation Steps or Deployment Steps

    ## Seven, Verification Methods

    ## Eight, Daily Maintenance Commands

    ## Nine, Common Faults and Troubleshooting

    ## Ten, Production Environment Considerations

    ## Eleven, Interview Answering Approach

    ## Twelve, Reference Documents

Different topics can be slightly adjusted, but overall maintain:

    Clear theory
    Clear architecture
    Executable commands
    Reproducible experiments
    Troubleshooting aligned with production
    Clear production boundaries

---

## Ten, Ceph Module Experiment Principles

### 10.1 Module Independence

Ceph modules must be independent systems.

Do not rely on MinIO, Longhorn, or RustFS experiment environments.

Ceph module experiments should be completed independently based on the following nodes:

    ceph-node01
    ceph-node02
    ceph-node03
    ceph-node04, optional
    ceph-client, optional

---

### 10.2 Do Not Mix With Kubernetes Nodes

Ceph experiments should not directly use existing Kubernetes nodes:

    10.0.0.20
    10.0.0.21
    10.0.0.22

Reasons:

    Ceph experiments involve OSD disks, network recovery, node failures, capacity stress testing, and service restarts.
    Mixing with Kubernetes nodes may affect the existing K8s learning environment.

---

### 10.3 Prioritize Independent Data Disks

OSD experiments should prioritize independent data disks:

    /dev/sdb
    /dev/sdc

It is not recommended to simulate production OSDs using system disk directories.

Experimental environments can use virtual machines with additional mounted virtual disks, but each OSD should still correspond to an independent disk for easier demonstration:

- OSD Addition
- OSD Removal
- OSD Replacement
- Disk Failure
- Data Recovery
- Backfill
- Rebalance

---

### 10.4 Use Domestic Sources

All installation-related notes for the Ceph module should prioritize domestic sources:

- Alibaba Cloud Ubuntu Source
- Alibaba Cloud Rocky Linux Source
- Alibaba Cloud Ceph Source

Official documentation should also be retained in reference materials for version and parameter verification.

---

## ElevenI don't know.Ceph Module Core Topology

Experiment topology:

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
        │ RBD / CephFS / RGW Testing │
        └──────────────────────┘

---

## TwelveI don't know.Ceph and Kubernetes Integration Topology

When a Ceph cluster integrates with a Kubernetes cluster, the logical relationship is as follows:

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

Notes:

    Kubernetes does not directly manage Ceph OSDs.
    Kubernetes uses CSI to leverage Ceph's storage capabilities.
    RBD is suitable for single Pod exclusive block storage.
    CephFS is suitable for multi-Pod shared file storage.

---

## ThirteenI don't know.Common Command Index

Each subsequent section will expand on these, but here we only list core command entries.

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

### 13.3 OSD

# 13. Ceph Commands

## 13.1 OSD

```
ceph osd tree
ceph osd stat
ceph osd df
ceph osd status
```

---

### 13.4 Pool

```
ceph osd pool ls
ceph osd pool get <pool-name> size
ceph osd pool get <pool-name> pg_num
ceph osd pool stats
```

---

### 13.5 PG

```
ceph pg stat
ceph pg dump
ceph pg dump_stuck
ceph health detail
```

---

### 13.6 RBD

```
rbd ls -p <pool-name>
rbd create <image-name> --size 10G -p <pool-name>
rbd info <pool-name>/<image-name>
rbd snap create <pool-name>/<image-name>@<snap-name>
```

---

### 13.7 CephFS

```
ceph fs ls
ceph fs status
ceph mds stat
```

---

### 13.8 RGW

```
radosgw-admin user list
radosgw-admin user info --uid=<user-name>
radosgw-admin bucket list
```

---

### 13.9 Dashboard

```
ceph mgr services
ceph dashboard set-login-credentials <user> <password>
```

---

## 14. Production Environment Focus Points

When Ceph enters production, the following aspects need special attention:

### 14.1 Node High Availability

At least consider:

- MON deployed with odd number of nodes
- MGR with at least 2 instances
- OSD distributed across multiple nodes
- CRUSH failure domain not concentrated on the same node
- Critical business Pool replica count not too low

---

### 14.2 Data Security

Need to consider:

- Replica count
- Erasure coding
- Pool planning
- OSD failure recovery
- Snapshot strategy
- Backup strategy
- Cross-cluster replication (scenario-dependent)

---

### 14.3 Performance

Need to consider:

- Disk type
- Network bandwidth
- OSD count
- PG count
- BlueStore parameters
- Client IO type
- Small file / large file scenarios
- Usage differences between RBD / CephFS / RGW

---

### 14.4 Operation Risk

Need to consider:

- Batch OSD failure
- Cluster nearfull/full
- Long-term degraded PG
- Recovery affecting business IO
- Network jitter causing slow ops
- Upgrade failure
- Accidental Pool deletion
- Incorrect CRUSH configuration
- Incorrect permission configuration

---

## 15. Learning Recommendations

Don't try to memorize all Ceph commands at the beginning of learning.

Recommended learning path:

```
1. Understand Ceph's overall architecture first.
2. Deploy a minimal 3-node cluster.
3. Study OSD, Pool, PG, CRUSH.
4. Practice RBD, CephFS, RGW separately.
5. Implement Kubernetes CSI integration.
6. Finally handle fault recovery, performance, security, and production planning.
```

The key focus of Ceph learning is not "command quantity", but understanding:

```
Where data is stored
Where replicas are located
How to recover after failure
What resources recovery consumes
How to avoid one operation affecting the entire cluster
How to determine if the issue is disk, network, OSD, PG, Pool, or client-related
```

---

## 16. Interview Answer Structure

If asked:

```
How do you understand Ceph?
```

You can answer:

```
Ceph is a distributed storage platform with RADOS as its core. It manages data through multiple OSDs, maintains cluster state and consistency via MON, provides management and monitoring capabilities through MGR, and uses CRUSH algorithm to determine OSD distribution.
RADOS provides three common storage interfaces: RBD for block storage, CephFS for file storage, and RGW for S3-compatible object storage.
From an operations perspective, the core of Ceph isn't just installation, but understanding concepts like OSD, Pool, PG, CRUSH, replicas, failure domains, Backfill, and Recovery. Production environments need focus on node high availability, disk planning, network bandwidth, capacity water levels, OSD failure recovery, PG anomalies, monitoring alerts, and upgrade risks.
When using Ceph with Kubernetes, it's typically integrated via Ceph CSI. RBD suits single Pod exclusive block storage, while CephFS is suitable for multi-Pod shared file storage.
```

---

## 17. Reference Documents

Ceph official documentation:

```
https://docs.ceph.com/
```

Ceph official architecture documentation:

```
https://docs.ceph.com/en/latest/architecture/
```

Cephadm deployment documentation:

```
https://docs.ceph.com/en/latest/cephadm/install/
```

Ceph hardware recommendations:

```
https://docs.ceph.com/en/reef/start/hardware-recommendations/
```

Ceph official website:

```
https://ceph.io/
```

Aliyun Ubuntu image:

```
https://developer.aliyun.com/mirror/ubuntu
```

Aliyun Rocky Linux image:

```
https://developer.aliyun.com/mirror/rockylinux
```

Aliyun Ceph image:

```
https://developer.aliyun.com/mirror/ceph
```