# Ceph Core Architecture: RADOS, MON, MGR, OSD, and CRUSH

Suggested path: 05-Storage/01-Ceph/02-Ceph Core Architecture: RADOS, MON, MGR, OSD, and CRUSH.md

Tags: #Ceph #RADOS #MON #MGR #OSD #CRUSH #PG #DistributedStorage #SRE #Clouds.

---

## I. Document Explanation

This is the second article of the Ceph advanced SRE storage module, focusing on Ceph's core architecture.

The previous article explained:

    Ceph is a general-purpose distributed storage platform.
    The core of Ceph is RADOS at the bottom layer.
    Ceph can provide three capabilities: RBD, CephFS, and RGW.

This article further focuses on Ceph's core components:

    RADOS
    MON
    MGR
    OSD
    CRUSH
    Pool
    PG
    Cluster Map

Understanding these components will make subsequent learning about OSD management, Pool and PG, CRUSH rules, data recovery, and performance optimization much easier.

---

## II. Overview of Ceph Core Architecture

Ceph's overall architecture can be divided into three layers from top to bottom:

    First layer: Client access layer
    Second layer: Interface and service layer
    Third layer: RADOS distributed storage foundation

Diagram:

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
    │  Storage Data   │   │  Storage Data   │    │  Storage Data   │
    └────────────┘   └────────────┘    └────────────┘

    Control and management components:

    ┌────────────┐
    │    MON     │  Maintains cluster state, Map, Quorum
    └────────────┘

    ┌────────────┐
    │    MGR     │  Management, monitoring, Dashboard, Prometheus, orchestration
    └────────────┘

    ┌────────────┐
    │   CRUSH    │  Calculates data placement location
    └────────────┘

Core understanding:

    RADOS is Ceph's foundation.
    OSD is where data is actually stored.
    MON maintains cluster state.
    MGR provides management and monitoring capabilities.
    CRUSH determines where data is placed.
    PG is the intermediate mapping layer between objects and OSDs.

---

## III. RADOS: Ceph's Distributed Storage Foundation

### 3.1 What is RADOS

RADOS, full name:

    Reliable Autonomic Distributed Object Store

Can be understood as:

    Ceph's underlying distributed object storage system.

RADOS is responsible for:

- Storing objects
- Managing object replicas
- Distributing objects to different OSDs
- Handling data recovery
- Supporting horizontal scaling
- Supporting self-healing
- Providing a unified storage foundation for RBD, CephFS, and RGW

RADOS is not a file system directory directly visible to users, nor is it a traditional block device.

It's more like Ceph's internal unified storage core.

---

### 3.2 Relationship Between RADOS and RBD, CephFS, RGW

Ceph's three common usage methods:

    RBD    Block Storage
    CephFS File Storage
    RGW    Object Storage

All of them ultimately rely on RADOS.

Diagram:

    RBD Image
        |
        v
    RADOS Objects
        |
        v
    OSD

    CephFS File
        |
        v
    RADOS Objects + MDS Metadata
        |
        v
    OSD

    RGW Object
        |
        v
    RADOS Objects
        |
        v
    OSD

Therefore, it can be understood as:

    RBD, CephFS, and RGW are different access interfaces of Ceph.
    RADOS is their common underlying data storage system.

---

### 3.3 Why RADOS is Important

If you only use RBD, CephFS, and RGW without understanding RADOS, it's difficult to truly troubleshoot Ceph issues.

For example:

- Why is an RBD volume degraded?
- Why is CephFS slowing down?
- Why is there an anomaly in RGW Bucket access?
- Why does an OSD down affect multiple services?
- Why is a PG always active+degraded?
- Why does business IO slow down during recovery?

These questions ultimately need to be traced back to the underlying RADOS:

    Where is the object?
    Which Pool does the object belong to?
    Which PG does the object map to?
    Which OSDs does the PG map to?
    Is the OSD currently healthy?
    Are there enough replicas?
    Is the CRUSH rule reasonable?

---

## FourI don't know.OSD: The Core Component That Actually Stores Data

### 4.1 What is OSD

OSD, full name:

    Object Storage Daemon

OSD is the daemon in Ceph that actually stores data.

Typically, one OSD corresponds to an independent data disk, for example:

    /dev/sdb -> osd.0
    /dev/sdc -> osd.1

In production, it's more recommended:

    One independent disk corresponds to one OSD.

In experimental environments, virtual disks can also be used as OSD data disks.

---

### 4.2 What Does OSD Handle

Main responsibilities of OSD:

- Store RADOS objects
- Process client read/write requests
- Handle replica replication
- Synchronize data with other OSDs
- Report heartbeats
- Report capacity
- Participate in data recovery
- Participate in Backfill
- Participate in Rebalance
- Execute Scrub / Deep Scrub
- Report status to MON

Simple understanding:

    OSD is the data node of Ceph.
    Ceph's capacity comes from OSD.
    Ceph's read/write performance strongly depends on the number of OSDs, disk performance, and network performance.

---

### 4.3 Relationship Between OSD and Disk

Typical relationship:

    ceph-node01:
      /dev/sdb -> osd.0
      /dev/sdc -> osd.1

    ceph-node02:
      /dev/sdb -> osd.2
      /dev/sdc -> osd.3

    ceph-node03:
      /dev/sdb -> osd.4
      /dev/sdc -> osd.5

Check OSD tree:

    ceph osd tree

Example output structure is similar to:

    ID  CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
    -1         6.00000  root default
    -3         2.00000      host ceph-node01
     0    hdd  1.00000          osd.0             up   1.00000  1.00000
     1    hdd  1.00000          osd.1             up   1.00000  1.00000
    -5         2.00000      host ceph-node02
     2    hdd  1.00000          osd.2             up   1.00000  1.00000
     3    hdd  1.00000          osd.3             up   1.00000  1.00000
    -7         2.00000      host ceph-node03
     4    hdd  1.00000          osd.4             up   1.00000  1.00000
     5    hdd  1.00000          osd.5             up   1.00000  1.00000

Focus on:

    Whether OSD is up
    Whether OSD is in
    Which hosts OSDs are distributed across
    Whether OSD weights are normal
    Whether there is an issue with OSDs concentrated on a single node

---

### 4.4 OSD's up and in

In Ceph, OSD has two important states:

    up / down
    in / out

Meaning:

| Status | Meaning |
|---|---|
| up | OSD process is running |
| down | OSD process is unavailable |
| in | OSD participates in data distribution |
| out | OSD no longer participates in new data distribution |

Common combinations:

| Status | Description |
|---|---|
| up + in | Normal state |
| down + in | OSD failure, but still in CRUSH data distribution, may trigger recovery |
| up + out | OSD is online but not participating in data distribution, common in maintenance or removal |
| down + out | OSD is offline and not participating in data distribution |

Check commands:

    ceph osd stat
    ceph osd tree
    ceph health detail

---

### 4.5 Impact of OSD Failure on the Cluster

If an OSD fails, Ceph will determine whether to recover replicas based on the replica count and CRUSH rules.

For example, if a Pool has a replica count of 3:

    size = 3

Normal object replicas:

    Object A:
      osd.0
      osd.2
      osd.4

If osd.2 fails:

    Object A:
      osd.0
      osd.2 down
      osd.4

The object may become:

    degraded

Ceph will attempt to recreate replicas on other OSDs until the target replica count is restored.

This process may trigger:

    Recovery
    Backfill
    Rebalance

These operations will consume disk IO, network bandwidth, and CPU.

---

## FiveI don't know.MON: Maintaining Cluster Status and Consistency

### 5.1 What is MON

MON, full name:

    Monitor

MON is the component in Ceph that maintains cluster status.

MON does not store business data; it is responsible for maintaining the key maps of the Ceph cluster.

Includes:

- Monitor Map
- OSD Map
- PG Map
- CRUSH Map
- MDS Map
- Manager Map
- Auth information

It can be understood as:

    MON is the status center and consistency control component of the Ceph cluster.

---

### 5.2 Why MON Needs an Odd Number

In production environments, MONs are typically deployed in an odd number:

3 MON  
5 MON  

Reason:  

MONs require the Quorum mechanism to ensure consistency.  

Quorum can be understood as the majority.  

For example, 3 MONs:  

- Allow 1 MON failure, remaining 2 can still form a majority.  

If only 1 MON:  

- Single point of failure is obvious.  

With 2 MONs:  

- After one failure, remaining one cannot form a majority, actual fault tolerance is poor.  

Recommended experiment:  

- 3 MONs  

Example nodes:  

- ceph-node01  
- ceph-node02  
- ceph-node03  

---  

### 5.3 Maps Maintained by MON  

The Maps maintained by MON are the foundation for Ceph to function normally.  

| Map | Function |  
|---|---|  
| Monitor Map | Records MON member information |  
| OSD Map | Records OSD status, weight, topology, etc. |  
| PG Map | Records PG status |  
| CRUSH Map | Records data placement rules and topology |  
| MDS Map | Records CephFS MDS status |  
| Manager Map | Records MGR status |  

Clients and OSDs rely on these Maps to determine cluster status and data location.  

---  

### 5.4 Checking MON Status  

Check cluster status:  

    ceph -s  

Check MON status:  

    ceph mon stat  

Check Quorum:  

    ceph quorum_status --format json-pretty  

Check MON dump:  

    ceph mon dump  

Focus on:  

- How many MONs are in quorum  
- Whether all MONs are online  
- Whether any MON has left quorum  
- Whether MON clocks are abnormal  
- Whether MON storage is full  

---  

### 5.5 Common MON Issues  

Common MON issues include:  

- MON not in quorum  
- MON disk space insufficient  
- MON time unsynchronized  
- MON network unreachable  
- MON data directory corrupted  
- MON certificate or authentication anomalies  
- Insufficient MON count  

Common troubleshooting directions:  

    ceph -s  
    ceph health detail  
    ceph quorum_status --format json-pretty  
    journalctl -u ceph-*.target  
    ceph orch ps | grep mon  

Time synchronization is particularly important.  

If node time drift is severe, it may affect MON quorum and authentication.  

---  

## SixI don't know.MGR: Management, Monitoring, and Expansion Capabilities  

### 6.1 What is MGR  

MGR, full name:  

    Manager  

MGR is the management component of Ceph, primarily responsible for cluster management, monitoring, and expansion modules.  

MGR typically provides:  

- Dashboard  
- Prometheus metrics  
- Orchestrator management capabilities  
- Telemetry  
- Crash module  
- Alert module  
- Device health  
- cephadm orchestration interface  

In modern Ceph clusters, MGR is a very important management component.  

---  

### 6.2 Difference Between MGR and MON  

MON and MGR are often confused.  

Simple distinction:  

| Component | Main Responsibilities |  
|---|---|  
| MON | Maintains cluster status and consistency |  
| MGR | Provides management, monitoring, and module expansion capabilities |  

It can be understood as:  

    MON is the core of status consistency.  
    MGR is the entry point for management and observation.  

MON anomalies will directly affect cluster status consistency.  

MGR anomalies typically affect Dashboard, Prometheus, ceph orch, etc. management capabilities.  

---  

### 6.3 MGR High Availability  

MGR typically has:  

    active MGR  
    standby MGR  

Check:  

    ceph mgr stat  

Example:  

    {  
        "epoch": 12,  
        "available": true,  
        "active_name": "ceph-node01",  
        "num_standby": 2  
    }  

Production recommendations:  

    Deploy at least 2 MGRs.  
    Recommended to deploy multiple MGR instances in a 3-node cluster.  

---  

### 6.4 Checking MGR Services  

Check MGR status:  

    ceph mgr stat  

Check enabled modules:  

    ceph mgr module ls  

Check service addresses:  

    ceph mgr services  

Common output:  

    {  
        "dashboard": "https://ceph-node01:8443/",  
        "prometheus": "http://ceph-node01:9283/"  
    }  

---  

### 6.5 Common MGR Issues  

Common issues:  

- Dashboard inaccessible  
- Prometheus metrics unavailable  
- ceph orch commands abnormal  
- MGR modules not enabled  
- No standby MGR  
- MGR process anomalies  

Troubleshooting:  

    ceph -s  
    ceph mgr stat  
    ceph mgr services  
    ceph mgr module ls  
    ceph orch ps | grep mgr  

---  

## SevenI don't know.CRUSH: Ceph's Data Placement Algorithm  

### 7.1 What is CRUSH  

CRUSH, full name:  

    Controlled Replication Under Scalable Hashing  

CRUSH is Ceph's data placement algorithm.  

It answers a critical question:  

    Which OSDs should an object be stored on?  

CRUSH is not a simple random algorithm. It calculates data placement location based on:  

- CRUSH Map  
- Pool rules  
- Replication count  
- OSD weights  
- Failure domain  
- Host topology  
- Rack topology  
- Device type  

---  

### 7.2 Why CRUSH is Needed  

Traditional storage systems may rely on centralized metadata services to record object locations.  

For example:  

    Object A -> Node1 Disk1  
    Object B -> Node2 Disk3  
    Object C -> Node5 Disk2  

If all object locations depend on a centralized table, scaling the cluster will bring:  

- Metadata bottleneck  
- Query bottleneck  
- Single point risk  
- Complex expansion  
- Complex migration  

Ceph uses CRUSH to solve this problem.  

Clients and OSDs can calculate data locations based on CRUSH Map, instead of querying a centralized object location table every time.  

---  

### 7.3 What is CRUSH Map  

CRUSH Map describes the physical and logical topology of the Ceph cluster.  

Typical hierarchy: /think

root default
  host ceph-node01
    osd.0
    osd.1
  host ceph-node02
    osd.2
    osd.3
  host ceph-node03
    osd.4
    osd.5

More complex production topologies may be:

  root default
    datacenter dc1
      rack rack01
        host ceph-node01
        host ceph-node02
      rack rack02
        host ceph-node03
        host ceph-node04

CRUSH Map can express:

- Host
- Rack
- Datacenter
- Disk
- Device type
- Failure domain

---

### 7.4 Failure Domain

Failure domain indicates:

    Where replicas should be spread across different levels to avoid multiple replicas being affected by the same type of failure simultaneously.

Common failure domains:

| Failure Domain | Meaning |
|---|---|
| osd | Replicas spread across different OSDs |
| host | Replicas spread across different hosts |
| rack | Replicas spread across different racks |
| datacenter | Replicas spread across different datacenters |

In experimental environments, it is generally used:

    host

To distribute multiple replicas of the same object across different hosts.

For example, with 3 replicas:

    Object A:
      replica 1 -> ceph-node01
      replica 2 -> ceph-node02
      replica 3 -> ceph-node03

This way, even if one host fails, there are still other replicas available.

---

### 7.5 Value of CRUSH

Value of CRUSH:

- Avoid centralized object location table
- Support large-scale expansion
- Support replica placement by failure domain
- Support automatic data rebalancing
- Support rules for different device types
- Support planning high availability by host, rack, and datacenter

Advanced SREs need to focus on understanding:

    Ceph's high availability is not only about replica count.
    Where replicas are placed is equally important.
    If 3 replicas are on the same host, higher replica count cannot protect against host failure.
    CRUSH rules are the core for controlling replica placement.

---

## VIII. Pool: Logical Storage Pool

### 8.1 What is a Pool

Pool is a logical storage pool in Ceph.

It can be understood as:

    A logical container for a group of data.

Common Pools:

    rbd
    cephfs_data
    cephfs_metadata
    .rgw.root
    default.rgw.buckets.data

Different businesses can use different Pools.

Different Pools can have different strategies set:

- Replica count
- PG count
- CRUSH Rule
- Storage type
- Quota
- Application type
- Erasure code strategy

---

### 8.2 Relationship Between Pool and Business

For example:

    Kubernetes RBD PVC -> rbd Pool
    CephFS file data -> cephfs_data Pool
    CephFS metadata -> cephfs_metadata Pool
    RGW Bucket data -> rgw data Pool

Check Pools:

    ceph osd pool ls

Check Pool details:

    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> pg_num
    ceph osd pool get <pool-name> crush_rule

Check capacity:

    ceph df

---

### 8.3 Pool Replica Count

Check Pool replica count:

    ceph osd pool get <pool-name> size

Set replica count:

    ceph osd pool set <pool-name> size 3

Meaning:

    size = 3 indicates that objects in this Pool have 3 replicas by default.

Minimum replica count:

    min_size

Check:

    ceph osd pool get <pool-name> min_size

Example:

    size = 3
    min_size = 2

Indicates:

    Normally retain 3 replicas.
    Continue providing write service as long as there are 2 replicas available.
    If below min_size, write may be stopped to protect data consistency.

---

## IX. PG: Intermediate Mapping Layer Between Object and OSD

### 9.1 What is PG

PG, full name:

    Placement Group

PG is a grouping of objects within a Pool.

It is between objects and OSDs.

Mapping relationship:

    Object -> PG -> OSD Set

Why is PG needed?

If each object directly maps to an OSD, it would cause excessive management overhead.

PG's role is:

    Group large numbers of objects for management, improving Ceph scheduling and recovery efficiency.

---

### 9.2 Object Write Path

Simplified write path:

    Client writes an object
        |
        v
    Calculate the PG that the object belongs to based on its name
        |
        v
    CRUSH calculates the OSD Set based on the PG
        |
        v
    Primary OSD handles the write
        |
        v
    Replica OSDs synchronize the write
        |
        v
    Write confirmation returns to the client

Illustration:

    Object A
      |
      v
    PG 1.23
      |
      v
    [osd.0, osd.2, osd.4]

---

### 9.3 PG Status

Common PG statuses:

| Status | Meaning |
|---|---|
| active+clean | Normal status |
| active+degraded | Missing replicas, but still serving |
| active+recovering | Recovering |
| active+backfilling | Backfilling data |
| peering | OSDs are negotiating PG status |
| stale | PG status has not been updated for a long time |
| undersized | Insufficient replica count |
| inconsistent | Data inconsistency |

Check PG status:

    ceph pg stat
    ceph health detail
    ceph pg dump_stuck

---

### 9.4 PG and Troubleshooting

Ceph troubleshooting often revolves around PGs.

For example: /think

HEALTH_WARN
Degraded data redundancy
pgs degraded
pgs undersized
pgs stuck inactive
pgs stale

Troubleshooting approach:

    1. ceph -s to check overall status.
    2. ceph health detail to check specific PG issues.
    3. ceph osd tree to check if OSD is down.
    4. ceph pg dump_stuck to check stuck PGs.
    5. Determine recovery direction based on OSD, Pool, and CRUSH rules.

---

## TenI don't know.Cluster Map: Core Information of Ceph Cluster Status

### 10.1 What is Cluster Map

Ceph has multiple Maps to describe cluster status.

Mainly includes:

| Map | Description |
|---|---|
| Monitor Map | MON member information |
| OSD Map | OSD status, weight, in/out status |
| PG Map | PG status |
| CRUSH Map | Topology and data placement rules |
| MDS Map | CephFS MDS status |
| Manager Map | MGR status |

These Maps are maintained by MON and used by clients and components.

---

### 10.2 Why Maps are Important

Ceph clients need to know:

    Which MONs are available
    Which OSDs are present
    Whether OSDs are up/in
    What CRUSH rules are in place
    Which PG objects should map to
    Which OSDs should map to PGs

This information comes from Cluster Map.

If Maps are inconsistent, the cluster cannot operate reliably.

---

### 10.3 Viewing Related Maps

View OSD Map:

    ceph osd dump

View CRUSH Map:

    ceph osd crush tree

View MON Map:

    ceph mon dump

View MDS Map:

    ceph mds stat

View MGR Map:

    ceph mgr dump

---

## ElevenI don't know.Ceph Data Write Process Simplified Understanding

### 11.1 RBD Write Example

Taking RBD as an example:

    Application writes to block device
        |
        v
    RBD client splits data into objects
        |
        v
    Objects enter specified Pool
        |
        v
    Objects map to PG
        |
        v
    CRUSH calculates OSD set based on PG
        |
        v
    Primary OSD receives write
        |
        v
    Replica OSDs synchronize write
        |
        v
    Return success after meeting write confirmation conditions

Illustration:

    App
     |
     v
    RBD Image
     |
     v
    Object A
     |
     v
    PG 1.23
     |
     v
    osd.0 primary
    osd.2 replica
    osd.4 replica

---

### 11.2 Primary OSD and Replica OSD

A PG corresponds to a group of OSDs.

One is the primary OSD, others are replica OSDs.

Writing process:

    Client -> Primary OSD -> Replica OSDs

Primary OSD coordinates the write.

Returns to client after sufficient replica confirmations.

---

### 11.3 Changes After OSD Failure

If osd.2 fails:

    PG 1.23 original mapping:
      osd.0
      osd.2
      osd.4

After failure:

    osd.2 down

Ceph selects new OSDs based on current OSD Map and CRUSH rules, e.g.:

    osd.5

Recovery goal:

    PG 1.23:
      osd.0
      osd.4
      osd.5

After recovery, PG returns to:

    active+clean

---

## TwelveI don't know.High Availability Design in Ceph Architecture

### 12.1 MON High Availability

MON achieves high availability through Quorum.

Recommendation:

    Start with 3 MONs.

Example:

    ceph-node01
    ceph-node02
    ceph-node03

Allows remaining in majority after one MON failure.

---

### 12.2 MGR High Availability

MGR uses active / standby.

Recommendation:

    At least 2 MGR instances.

If active MGR fails, standby can take over.

---

### 12.3 OSD High Availability

OSD high availability depends on:

- Multiple OSDs
- Multiple nodes
- Appropriate replica count
- Reasonable CRUSH failure domain
- Sufficient remaining capacity
- Healthy network

OSD itself isn't highly available, but data availability is achieved through multiple OSD replicas and recovery.

---

### 12.4 Data High Availability

Core elements of data high availability:

    Replica count
    min_size
    CRUSH failure domain
    OSD distribution
    Pool rules
    Recovery capability

Error example:

    3 replicas on the same host.

In this case, host failure would make all replicas unavailable.

Correct example:

    3 replicas on 3 different hosts.

This ensures resilience against single-node failures.

---

## ThirteenI don't know.Component Deployment Recommendations in Experimental Environment

### 13.1 Three-Node Basic Experiment

Recommended:

| Node | MON | MGR | OSD |
|---|---|---|---|
| ceph-node01 | Yes | Yes | Yes |
| ceph-node02 | Yes | Yes | Yes |
| ceph-node03 | Yes | Yes | Yes |

Advantages:

    Meets MON majority requirement.
    Meets OSD multi-node replica distribution.
    Helps understand CRUSH host failure domain.
    Enables single-node failure experiments.

---

### 13.2 Four-Node Extended Experiment

Extended node:

| Node | Role |
|---|---|
| ceph-node04 | OSD / Expansion / Fault Simulation |

Purpose:

- Add new OSDs
- Expand cluster capacity
- Observe Rebalance
- Replace faulty disks
- Simulate node offline
- Validate recovery process

---

### 13.3 Client Node

Optional client: /think

| Node | Purpose |
|---|---|
| ceph-client | RBD / CephFS / RGW Testing |

Client can be used for:

- Mount RBD
- Mount CephFS
- Use s3cmd / mc to test RGW
- Verify user permissions
- Verify read/write performance

---

## Fourteen. Daily Architecture Check Commands

### 14.1 Check Cluster Status

    ceph -s

Focus on:

    health
    mon
    mgr
    osd
    pgs
    usage
    io

---

### 14.2 Check Service Processes

    ceph orch ps

Filter by component:

    ceph orch ps | grep mon
    ceph orch ps | grep mgr
    ceph orch ps | grep osd

---

### 14.3 Check Hosts

    ceph orch host ls

---

### 14.4 Check OSD Tree

    ceph osd tree

---

### 14.5 Check CRUSH Tree

    ceph osd crush tree

---

### 14.6 Check Pools

    ceph osd pool ls
    ceph df

---

### 14.7 Check PGs

    ceph pg stat
    ceph health detail

---

### 14.8 Check MGR Services

    ceph mgr services

---

## Fifteen. Common Architecture Issues and Troubleshooting Directions

### 15.1 MON Not in Quorum

Phenomenon:

    ceph -s shows mon quorum abnormality

Troubleshooting:

    ceph quorum_status --format json-pretty
    ceph mon stat
    ceph health detail

Common causes:

- MON node network unavailability
- MON node time desynchronization
- MON data directory anomaly
- MON process anomaly
- Insufficient MON count

---

### 15.2 MGR Unavailable

Phenomenon:

    ceph -s shows mgr abnormality
    Dashboard inaccessible
    Prometheus metrics unavailable

Troubleshooting:

    ceph mgr stat
    ceph mgr services
    ceph mgr module ls
    ceph orch ps | grep mgr

---

### 15.3 OSD down

Phenomenon:

    ceph -s shows osd down
    PG degraded
    HEALTH_WARN

Troubleshooting:

    ceph osd tree
    ceph osd stat
    ceph health detail
    ceph orch ps | grep osd

Further troubleshooting:

    Disk damage
    OSD process running
    Network normal
    Node online
    Logs have BlueStore / IO errors

---

### 15.4 PG degraded

Phenomenon:

    pgs degraded
    Degraded data redundancy

Troubleshooting:

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree

Possible causes:

- OSD down
- OSD out
- Insufficient replicas
- CRUSH rule limitations
- Capacity insufficiency
- Recovery in progress

---

### 15.5 Slow Data Recovery

Possible causes:

- OSD disk slow
- Insufficient network bandwidth
- backfill parameter limitations
- recovery parameter limitations
- High cluster IO pressure
- Few OSDs
- Unreasonable PG count
- Some nodes becoming hotspots

Troubleshooting:

    ceph -s
    ceph osd perf
    ceph osd df
    ceph pg stat
    ceph health detail

---

## Sixteen. Advanced SRE Focus Points

### 16.1 Don't Only Check Component Status

Basic check:

    ceph -s HEALTH_OK

Advanced SRE also needs to check:

- Whether OSD distribution is reasonable
- Whether pool replica count is reasonable
- Whether CRUSH failure domain is reasonable
- Whether capacity water level is healthy
- Whether single-node capacity is too high
- Whether slow OSDs exist
- Whether PGs are long-term unclean
- Whether recovery tasks affect business
- Whether Dashboard / Prometheus is integrated with monitoring
- Whether fault drill plans are available

---

### 16.2 Architecture Design Matters More Than Commands

Many Ceph failures are not command-related but due to early architecture design issues.

For example:

| Issue | Root Cause |
|---|---|
| High data risk after OSD failure | Unreasonable replica count or failure domain design |
| Slow recovery process | Insufficient network and disk planning |
| Cluster quickly full | Insufficient capacity planning |
| Long-term PG anomalies | OSD, Pool, PG, CRUSH design issues |
| High single-node pressure | Unbalanced data distribution |
| Slow recovery after expansion | Few OSDs or insufficient network bandwidth |

---

### 16.3 Ceph Should Not Run Without Monitoring

Ceph must have monitoring and alerts.

At least monitor:

- Cluster health status
- OSD up/down
- OSD in/out
- PG clean/degraded
- Capacity usage rate
- nearfull/full
- slow ops
- recovery/backfill
- MON quorum
- MGR status
- Disk errors
- Network latency

---

## Seventeen. Production Notes

### 17.1 MON

Production recommendations:

    3 or 5 MONs.
    Not recommended to have 1 MON.
    Not recommended to have only 2 MONs.

Notes:

    MON node time synchronization must be stable.
    MON node disk space cannot be full.
    MON network must be stable.

---

### 17.2 MGR

Production recommendations:

    At least 2 MGRs.
    Dashboard and Prometheus modules must be included in monitoring.
    MGR anomalies may not affect data read/write but impact management and monitoring capabilities.

---

### 17.3 OSD

Production recommendations:

    Use independent disks as OSDs.
    Do not mix system disks with OSDs.
    Distribute OSDs across multiple nodes.
    Avoid concentrating all OSDs on few nodes.
    Regularly monitor OSD latency and disk errors.

---

### 17.4 CRUSH

Production recommendations:

Use at least host-level fault domains.  
In multi-rack environments, consider rack-level fault domains.  
Do not only focus on replica count; check if replicas are spread out.  
Evaluate data migration impact before modifying CRUSH rules.  

---

### 17.5 Pool and PG  

Production recommendations:  

    Do not create excessive Pools arbitrarily.  
    Keep PG count reasonable.  
    Set replica count and min_size carefully.  
    Confirm the CRUSH Rule used by the Pool before business goes live.  
    Important business Pools should clearly define capacity and alert thresholds.  

---

## EighteenI don't know.Interview Answer Structure  

If asked in an interview:  

    What is Ceph's core architecture?  

Answer:  

    Ceph's core is RADOS, the distributed object storage foundation. Upper-layer RBD block storage, CephFS file storage, and RGW object storage all ultimately rely on RADOS.  
    In a Ceph cluster, data is stored by OSDs, typically one OSD per independent disk. MON maintains cluster state and various Maps, such as OSD Map, PG Map, and CRUSH Map, and ensures consistency through Quorum. MGR handles management, monitoring, Dashboard, Prometheus, and cephadm orchestration.  
    Data placement is determined by the CRUSH algorithm, which calculates where objects should be placed based on cluster topology, OSD weights, Pool rules, and fault domains. Objects are not directly mapped to OSDs but first mapped to PGs, then CRUSH maps PGs to a group of OSDs.  
    From an operations perspective, understanding OSDs, Pools, PGs, CRUSH, replicas, fault domains, recovery, and backfill is key. In production, it's not enough to check HEALTH_OK; also monitor replica distribution, capacity water levels, PG status, OSD balance, and whether recovery affects business.  

---

## NineteenI don't know.Summary of This Article  

This article mainly explains Ceph's core architecture:  

1. RADOS is the distributed storage foundation of Ceph.  
2. RBD, CephFS, and RGW are all built on top of RADOS.  
3. OSDs are the core components that store data.  
4. MON maintains cluster state and various Maps, ensuring consistency through Quorum.  
5. MGR provides management, monitoring, Dashboard, Prometheus, and orchestration capabilities.  
6. CRUSH determines where objects or PGs should be placed on OSDs.  
7. Pools are logical storage pools that can set replica counts, PGs, CRUSH Rules, etc.  
8. PGs are the intermediate mapping layer between objects and OSDs.  
9. Ceph's high availability isn't just about replica count; it's also about whether replicas are reasonably distributed across fault domains.  
10. Advanced SREs need to understand Ceph from architecture design, fault domains, capacity, recovery, monitoring, and production boundaries.  

---

## TwentyI don't know.Reference Documents  

Ceph official architecture documentation:  

    https://docs.ceph.com/en/reef/architecture/  

Ceph Storage Cluster documentation:  

    https://docs.ceph.com/en/reef/rados/  

Ceph CRUSH Map documentation:  

    https://docs.ceph.com/en/latest/rados/operations/crush-map/  

Ceph Placement Groups documentation:  

    https://docs.ceph.com/en/reef/rados/operations/placement-groups/  

Ceph OSD Service documentation:  

    https://docs.ceph.com/en/reef/cephadm/services/osd/  

Ceph Health Checks documentation:  

    https://docs.ceph.com/en/reef/rados/operations/health-checks/  

Ceph Glossary:  

    https://docs.ceph.com/en/latest/glossary/