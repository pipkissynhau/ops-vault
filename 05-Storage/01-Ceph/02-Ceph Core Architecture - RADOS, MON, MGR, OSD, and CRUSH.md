# Ceph Core Architecture: RADOS, MON, MGR, OSD, and CRUSH

Recommended Path: 05-Storage/01-Ceph/02-Ceph Core Architecture: RADOS, MON, MGR, OSD, and CRUSH.md

Tags: #Ceph #RADOS #MON #MGR #OSD #CRUSH #PG #Distributed Storage #SRE #Cloud-Native

---

## I. Document Overview

This article is the second in the advanced SRE storage module series for Ceph, focusing on its core architecture.

The previous article explained:

    - Ceph is a general-purpose distributed storage platform.
    - RADOS forms the foundation of Ceph.
    - Ceph provides three main services: RBD, CephFS, and RGW.

This article delves deeper into Ceph's core components:

    - RADOS
    - MON
    - MGR
    - OSD
    - CRUSH
    - Pool
    - PG
    - Cluster Map

Understanding these components will make it easier to learn about OSD management, Pool and PG operations, CRUSH rules, data recovery, and performance optimization.

---

## II. Ceph Core Architecture Overview

Ceph's overall architecture can be divided into three layers from top to bottom:

    **Layer 1: Client Access Layer**
    **Layer 2: Interface and Service Layer**
    **Layer 3: RADOS Distributed Storage Base**

Schematic Diagram:

    ┌──────────────────────────────────────────────┐
    │                  Application / Client                │
    │  VM / Kubernetes / Linux Host / S3 Client     │
    └──────────────────────────────────────────────┘
               │                 │                │
               v                 v                v
    ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
    │      RBD       │ │     CephFS     │ │      RGW       │
    │    Block Storage       │ │    File Storage     │ │  S3 Object Storage Gateway │
    └────────┬───────┘ └────────┬───────┘ └────────┬───────┘
             │                  │                  │
             └──────────────────┼──────────────────┘
                                v
    ┌──────────────────────────────────────────────┐
    │                    RADOS                     │
    │        Reliable Autonomic Distributed        │
    │                 Object Store                 │
    └──────────────────────────────────────────────┘
           │                │                 │
           v                v                 v
    ┌────────────┐   ┌────────────┐    ┌────────────┐
    │    OSD     │   │    OSD     │    │    OSD     │
    │  Store Data       │   │  Store Data   │    │  Store Data   │
    └────────────┘   └────────────┘    └────────────┘

**Control and Management Components:**

    ┌────────────┐
    │    MON     │  Maintains cluster status, Map, Quorum
    └────────────┘

    ┌────────────┐
    │    MGR     │  Manages, monitors, provides Dashboard, Prometheus, orchestration
    └────────────┘

    ┌────────────┐
    │   CRUSH    │  Determines data placement locations
    └────────────┘

**Key Concepts:**

    - RADOS is the foundation of Ceph.
    - OSDs store the actual data.
    - MON maintains the cluster's status.
    - MGR handles management and monitoring tasks.
    - CRUSH decides where data should be stored.
    - PG acts as a mapping layer between objects and OSDs.

---

## III. RADOS: Ceph's Distributed Storage Base

### 3.1 What is RADOS?

RADOS stands for:

    Reliable Autonomic Distributed Object Store

It can be understood as:

    Ceph's underlying distributed object storage system.

RADOS is responsible for:

- Storing objects
- Managing object copies
- Distributing objects across different OSDs
- Handling data recovery
- Supporting horizontal scaling
- Enabling self-healing in case of failures
- Providing a unified storage foundation for RBD, CephFS, and RGW

RADOS is not a file system directory that users see directly, nor is it a traditional block device. It serves as the central storage core within Ceph.

---

### 3.2 The Relationship between RADOS and RBD, CephFS, RGW

Ceph's three main usage scenarios—RBD for block storage, CephFS for file storage, and RGW for object storage—all rely on RADOS at their fundamental level.

- **Device Type**: HDD, SSD, etc.
- **Host**: The physical machine where the device is located.
- **Rack**: A group of hosts on the same physical rack.
- **Region**: A group of racks in the same data center or campus.
- **Pool**: A collection of OSDs that store objects together.
- **OSD Weight**: The relative importance of each OSD in data distribution.
- **Fault Domain**: Groups of OSDs that are distributed across different racks to prevent failures.

CRUSH Map uses this hierarchy and various rules to calculate the optimal location for storing objects on OSDs.

---

### 7.4 CRUSH 规则

Ceph provides several rules that influence how objects are placed:

1. **Pool Rules**: Define which OSDs belong to which pool.
2. **Replica Count**: Specify how many copies of an object should be stored.
3. **OSD Weight**: Determine the importance of each OSD in data distribution.
4. **Fault Domain**: Distribute objects across different racks to reduce the risk of data loss.
5. **Host and Rack Topology**: Consider the physical layout of hosts and racks when placing objects.

These rules work together to ensure that Ceph can store and retrieve data efficiently, while also providing fault tolerance and scalability.root default
      host ceph-node01
        osd.0
        osd.1
      host ceph-node02
        osd.2
        osd.3
      host ceph-node03
        osd.4
        osd.5

More complex production topologies may include:

    root default
      datacenter dc1
        rack rack01
          host ceph-node01
          host ceph-node02
        rack rack02
          host ceph-node03
          host ceph-node04

The CRUSH Map can represent the following components:

- Hosts
- Racks
- Data centers
- Disks
- Device types
- Fault domains

---

### 7.4 Fault Domains

Fault domains are used to ensure that replicas are distributed across different levels, preventing multiple copies from being affected by the same type of failure simultaneously.

Common fault domains include:

| Fault Domain | Meaning |
|---|---|
| osd | Replicas are distributed across different OSDs |
| host | Replicas are distributed across different hosts |
| rack | Replicas are distributed across different racks |
| datacenter | Replicas are distributed across different data centers |

In experimental environments, the "host" fault domain is often used to ensure that multiple copies of the same object are distributed across different hosts.

For example, if there are 3 replicas:

    Object A:
      replica 1 -> ceph-node01
      replica 2 -> ceph-node02
      replica 3 -> ceph-node03

This way, even if one host fails, other copies will still be available.

---

### 7.5 The Value of CRUSH

The value of CRUSH lies in its ability to:

- Avoid the need for a centralized object location table
- Support large-scale scaling
- Allow replicas to be placed according to fault domains
- Enable automatic data rebalancing
- Support rules based on different device types
- Help plan for high availability by considering hosts, racks, and data centers

Advanced SRE professionals should understand that Ceph's high availability is not solely determined by the number of replicas; where these replicas are placed is equally important. If all 3 replicas are on the same host, having more replicas will not prevent a host failure. CRUSH rules are therefore crucial for controlling the placement of replicas.

---

## VIII. Pool: Logical Storage Pools

### 8.1 What is a Pool?

A Pool is a logical storage pool in Ceph.

It can be thought of as:

- A logical container for a group of data.

Common Pool examples include:

    rbd
    cephfs_data
    cephfs_metadata
    .rgw.root
    default.rgw.buckets.data

Different services can use different Pools, and each Pool can have different settings, such as:

- Number of replicas
- Number of PGs (Placement Groups)
- CRUSH rules
- Storage type
- Quotas
- Application type
- Erasure code strategy

---

### 8.2 The Relationship between Pools and Services

For example:

    Kubernetes RBD PVCs are stored in the rbd Pool.
    CephFS file data is stored in the cephfs_data Pool.
    CephFS metadata is stored in the cephfs_metadata Pool.
    RGW Bucket data is stored in the rgw data Pool.

To view Pools, use the command:

    ceph osd pool ls

To get detailed information about a Pool, use:

    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> pg_num
    ceph osd pool get <pool-name> crush_rule

To check the capacity of a Pool, use:

    ceph df

---

### 8.3 Number of Replicas in a Pool

To view the number of replicas in a Pool, use:

    ceph osd pool get <pool-name> size

To set the number of replicas, use:

    ceph osd pool set <pool-name> size 3

This means that objects in this Pool will have 3 default replicas.

The minimum number of replicas can be configured using:

    min_size

To check this value, use:

    ceph osd pool get <pool-name> min_size

For example, setting both `size` and `min_size` to 3 ensures that at least 2 replicas are always available, providing data durability even in the event of a failure.

---

## IX. PG: The Intermediate Mapping Layer between Objects and OSDs

### 9.1 What is a PG?

A PG, or Placement Group, is a way to group objects within a Pool.

It acts as an intermediary between objects and OSDs.

The mapping relationship is:

    Object -> PG -> OSD Set

Why are PGs necessary? If each---

### 11.3 Changes After OSD Failure

If osd.2 fails:

    Original mapping of PG 1.23:
      osd.0
      osd.2
      osd.4

After failure:

    osd.2 is down

Ceph will select a new OSD based on the current OSD Map and CRUSH rules for recovery, for example:

    osd.5

Target after recovery:

    PG 1.23:
      osd.0
      osd.4
      osd.5

After recovery, the PG returns to:

    active+clean

---

## Chapter Twelve: High Availability Design in Ceph Architecture

### 12.1 MON High Availability

MON achieves high availability through Quorum.

Recommendation:

    Start with 3 MON nodes.

Example:

    ceph-node01
    ceph-node02
    ceph-node03

This ensures a majority is still available even if one MON node fails.

---

### 12.2 MGR High Availability

MGR uses an active/standby configuration.

Recommendation:

    Have at least 2 MGR instances.

If the active MGR fails, the standby can take over.

---

### 12.3 OSD High Availability

The high availability of OSDs depends on:

- Multiple OSDs
- Multiple nodes
- Appropriate number of replicas
- Reasonable CRUSH failure domains
- Sufficient remaining capacity
- Healthy network

OSDs are not designed for individual high availability but achieve data reliability through multiple copies and recovery mechanisms.

---

### 12.4 Data High Availability

Key elements for data high availability:

    Number of replicas
    min_size
    CRUSH failure domains
    OSD distribution
    Pool rules
    Recovery capability

Incorrect example:

    All 3 replicas are on the same host.

In this case, a host failure would cause all replicas to be unavailable simultaneously.

Correct example:

    The 3 replicas are distributed across 3 different hosts.

This ensures resilience against single-node failures.

---

## Chapter Thirteen: Component Deployment Recommendations in Experimental Environments

### 13.1 Three-Node Basic Experiment

Recommendation:

| Node | MON | MGR | OSD |
|---|---|---|---|
| ceph-node01 | Yes | Yes | Yes |
| ceph-node02 | Yes | Yes | Yes |
| ceph-node03 | Yes | Yes | Yes |

Advantages:

    Ensures a majority of MON nodes.
    Achieves multi-node OSD distribution.
    Facilitates understanding of CRUSH host failure domains.
    Allows for single-node failure experiments.

---

### 13.2 Four-Node Expansion Experiment

Additional node:

| Node | Role |
|---|---|
| ceph-node04 | OSD / Expansion / Fault Testing |

Uses:

- Adds new OSDs
- Expands cluster capacity
- Observes rebalancing processes
- Replaces faulty disks
- Simulates node decommissioning
- Tests recovery procedures

---

### 13.3 Client Nodes

Optional clients:

| Node | Purpose |
|---|---|
| ceph-client | For testing RBD, CephFS, and RGW |

Clients can be used for:

- Mounting RBD volumes
- Mounting CephFS filesystems
- Testing RGW using s3cmd/mc tools
- Verifying user permissions
- Assessing read/write performance

---

## Chapter Fourteen: Common Commands for Checking the Architecture

### 14.1 Viewing Cluster Status

    ceph -s

Key areas to monitor:

    health
    mon
    mgr
    osd
    pgs
    usage
    io

---

### 14.2 Viewing Service Processes

    ceph orch ps

Filter by component:

    ceph orch ps | grep mon
    ceph orch ps | grep mgr
    ceph orch ps | grep osd

---

### 14.3 Viewing Hosts

    ceph orch host ls

---

### 14.4 Viewing OSD Tree

    ceph osd tree

---

### 14.5 Viewing CRUSH Tree

    ceph osd crush tree

---

### 14.6 Viewing Pools

    ceph osd pool ls
    ceph df

---

### 14.7 Viewing PGs

    ceph pg stat
    ceph health detail

---

### 14.8 Viewing MGR Services

    ceph mgr services

---

## Chapter Fifteen: Common Architecture Issues and Troubleshooting Directions

### 15.1 MON Not Within Quorum

Symptom:

    ceph -s shows an abnormal mon quorum status.

Troubleshooting:

    ceph quorum_status --format json-pretty
    ceph mon stat
    ceph health detail

Common causes:

-In a Ceph cluster, it is the OSDs that actually store data, with each OSD typically corresponding to an independent disk. The MON is responsible for maintaining the cluster's status and various types of maps, such as the OSD Map, PG Map, and CRUSH Map, ensuring consistency through Quorum. The MGR handles management, monitoring, dashboard functionality, Prometheus integration, and CephADM orchestration tasks.

The placement of data is determined by the CRUSH algorithm, which takes into account factors like the cluster topology, OSD weights, pool rules, and fault domains to decide on which OSDs an object should be stored on. Objects are not directly mapped to OSDs but first to PGs, which are then allocated across a set of OSDs by CRUSH.

From an operational perspective, it is crucial to understand concepts such as OSDs, pools, PGs, CRUSH, replicas, fault domains, recovery processes, and backfilling. In production environments, one must not only ensure that the HEALTH_OK status is maintained but also pay attention to whether replicas are distributed reasonably, whether capacity levels are safe, whether PGs are clean, whether OSDs are balanced, and whether recovery activities affect business operations.