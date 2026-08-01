# Ceph CRUSH Rules: Fault Domains, Data Placement, and Rack-Level High Availability

Recommended path: 05-Storage/01-Ceph/08-Ceph CRUSH Rules: Fault Domains, Data Placement, and Rack-Level High Availability.md

Tags: #Ceph #CRUSH #FaultField #HighAvailable #OSD #Pool #PG #DataDistribution #It'sHighlyAvailable. #SRE #DistributedStorage #AdvancedSre

---

## I. Document Overview

This is the eighth article in the Ceph advanced SRE storage module series, focusing on CRUSH rules, fault domains, data placement, and rack-level high availability design.

Previously learned:

- Ceph core architecture
- RADOS, MON, MGR, OSD
- Pools and PGs
- Replication size
- min_size
- OSD addition, removal, replacement, and expansion
- PG status and data recovery basics

This article answers a key question:

    How does Ceph decide which OSDs to place data replicas on?

Ceph does not randomly write data to certain disks, but instead uses the CRUSH algorithm to calculate data placement based on the CRUSH Map, pool-bound CRUSH rules, OSD weights, device types, fault domains, and replication size.

Understanding CRUSH is essential to truly comprehend:

- Why size=3 doesn't necessarily mean safety
- Why replicas must be distributed across hosts/racks
- Why adding an OSD triggers Rebalance
- Why changing CRUSH rules causes data migration
- Why fault domain planning is mandatory in production
- How to upgrade from single-node HA to rack-level HA
- Why CRUSH changes are high-risk storage modifications

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand the role and basic principles of CRUSH.
2. View the current CRUSH Tree.
3. View the default CRUSH Rule.
4. Understand the hierarchy of root, rack, host, and OSD.
5. Understand the role of fault domains.
6. Create a host-level CRUSH Rule.
7. Create a pool using a specified CRUSH Rule.
8. View the CRUSH Rule used by a pool.
9. Use ceph pg map to verify OSD distribution for PGs.
10. Build a logical rack topology.
11. Create a rack-level CRUSH Rule.
12. Verify if replicas are distributed across racks.
13. Understand the cause of PG undersized due to faulty domains.
14. Understand the value of device class for HDD/SSD/NVMe tiering.
15. Clarify the boundaries of high-risk operations like modifying CRUSH Rules, moving hosts, and adjusting weights.

---

## III. Experiment Environment

This article continues using the Ceph module experiment environment.

| IP | Hostname | Role | Notes |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON/MGR/OSD | Ceph Node 1 |
| 10.0.0.32 | ceph-node02 | MON/MGR/OSD | Ceph Node 2 |
| 10.0.0.33 | ceph-node03 | MON/MGR/OSD | Ceph Node 3 |
| 10.0.0.34 | ceph-node04 | OSD/Expansion/Fault Simulation | Optional |

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Prerequisites:

    Ceph cluster has been initialized via cephadm.
    At least 3 OSDs.
    OSDs distributed across ceph-node01, ceph-node02, ceph-node03.
    MON quorum is normal.
    MGR is normal.
    ceph -s can be executed normally.
    PGs should be active+clean as much as possible.

---

## IV. Pre-Operation Checks

CRUSH-related operations may trigger data remapping and migration. It's recommended to perform checks before any experiment.

### 4.1 Check Cluster Status

    ceph -s

Ideal status:

    health: HEALTH_OK

At least confirm:

    mon is normal
    mgr is normal
    osd up/in
    pgs are active+clean

If there are already HEALTH_ERR issues, it's not recommended to proceed with CRUSH experiments.

---

### 4.2 Check Health Details

    ceph health detail

If any of the following anomalies exist, they need to be addressed or recorded:

- OSD down
- PG degraded
- PG undersized
- nearfull/full
- MON quorum issues
- MGR issues

---

### 4.3 Check OSD Distribution

    ceph osd tree

Key confirmations:

- Are OSDs up?
- Are OSDs in?
- Are OSDs distributed across different hosts?
- Are there any OSDs down?
- Is there a host without OSDs?
- Are OSD weights significantly abnormal?

---

### 4.4 Check CRUSH Tree

    ceph osd crush tree

Key confirmations:

- Root name
- Host hierarchy
- Are OSDs correctly assigned to hosts?
- Is there an existing rack hierarchy?
- Are OSD classes correct?
- Are weights aligned with expectations?

---

### 4.5 Check Pool and PG Status

    ceph osd pool ls
    ceph pg stat
    ceph osd pool autoscale-status

Confirm:

    No significant backfill/recovery activity currently.
    No unexplained remapped PGs.
    No long-term PG anomalies.

---

## V. Why CRUSH is Needed

### 5.1 Without CRUSH

If a distributed storage system needs to maintain a centralized location table for each object, for example:

    object-a -> osd.0, osd.2, osd.4
    object-b -> osd.1, osd.3, osd.5
    object-c -> osd.0, osd.3, osd.4

This approach would cause problems:

- Central metadata service under heavy pressure
- Mapping tables grow with more objects
- Clients need to rely on central queries to access data
- Cluster expansion requires massive updates to mapping relationships
- Central metadata service may become a bottleneck
- High maintenance costs in large-scale clusters

Ceph's design philosophy is:

    Not maintaining a centralized location table for each object.
    Clients and OSDs calculate where objects should be placed based on the CRUSH Map.

- Where should data be placed on which OSDs
- How should replicas be distributed across fault domains
- How does data rebalance after adding new OSDs
- How does Ceph choose new locations for data after an OSD failure
- How to allocate data by disk weight
- How to differentiate between HDD / SSD / NVMe
- How to plan high availability by host, rack, and data center

CRUSH's core value:

    Decentralized computation of data location.
    Placement of replicas according to topology and fault domains.
    Support for large-scale expansion.
    Support for data redistribution after node, disk, and rack changes.

---

## SixI don't know.CRUSH Core Concepts

### 6.1 What is CRUSH

CRUSH full name:

    Controlled Replication Under Scalable Hashing

It can be understood as:

    Ceph's data placement algorithm.

CRUSH calculates data location based on the following information:

- CRUSH Map
- Pool's corresponding CRUSH Rule
- Object / PG ID
- OSD weight
- OSD status
- Bucket hierarchy
- Fault domain
- Device type
- Replica size

Simplified chain:

    Object
      |
      v
    PG
      |
      v
    CRUSH Rule
      |
      v
    CRUSH Map
      |
      v
    OSD Set

---

### 6.2 What is CRUSH Map

CRUSH Map describes the storage topology and data placement rules of a Ceph cluster.

It contains:

- OSD list
- OSD weight
- Host hierarchy
- Rack hierarchy
- Root hierarchy
- Device category
- Bucket structure
- CRUSH Rule
- Data placement policy

You can understand CRUSH Map as:

    The storage topology map of a Ceph cluster.

---

### 6.3 What is CRUSH Rule

CRUSH Rule is the data placement rule used by a Pool.

It determines:

    How the replicas of data in this Pool are selected for OSDs.

Simple understanding:

    CRUSH Map is the map.
    CRUSH Rule is the rule for navigating the map.
    After a Pool binds to a CRUSH Rule, data will be placed according to this rule.

---

### 6.4 What is a Fault Domain

A fault domain represents:

    A boundary of resources that may be affected by a single failure.

Common fault domains:

| Fault Domain | Example Failure |
|---|---|
| osd | Single disk failure |
| host | Single server downtime |
| rack | Single rack power outage or switch failure |
| room | Single room area failure |
| datacenter | Entire data center failure |

Ceph's CRUSH Rule can specify fault domains.

For example:

    host fault domain: replicas should be spread across different hosts
    rack fault domain: replicas should be spread across different racks

---

## SevenI don't know.CRUSH Topology

### 7.1 Default host-level topology

Common CRUSH topology in experimental environments:

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

Schematic diagram:

    ┌──────────────────────────────┐
    │          root default         │
    └──────────────┬───────────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
       v           v           v
    host01      host02      host03
       │           │           │
    osd.0       osd.2       osd.4
    osd.1       osd.3       osd.5

Under the host fault domain, replicas will be spread across different hosts.

---

### 7.2 Rack-level topology

Production environments may have multiple racks:

    root default
      rack rack01
        host ceph-node01
          osd.0
          osd.1
        host ceph-node02
          osd.2
          osd.3
      rack rack02
        host ceph-node03
          osd.4
          osd.5
        host ceph-node04
          osd.6
          osd.7
      rack rack03
        host ceph-node05
          osd.8
          osd.9
        host ceph-node06
          osd.10
          osd.11

The goal of rack-level high availability:

    A rack failure should not affect all replicas.

---

### 7.3 Bucket Hierarchy

Common bucket types in CRUSH:

| Type | Meaning |
|---|---|
| osd | Single OSD |
| host | Single host |
| rack | Rack |
| row | Cabinet row |
| room | Room |
| datacenter | Data center |
| root | Root level |

Common hierarchy in experiments:

    root -> host -> osd

Production hierarchy may be:

    root -> rack -> host -> osd

More complex production hierarchy may be:

    root -> datacenter -> room -> rack -> host -> osd

---

## EightI don't know.Relationship Between Replica Count and Fault Domain

### 8.1 size only represents replica count

The size of a Pool represents the number of replicas.

For example:

    size = 3

Means:

    Each object has 3 replicas under normal conditions.

But size=3 does not guarantee protection against host or rack failures.

---

### 8.2 Fault domain determines where replicas are placed

Incorrect example:

    Object A:
      Replica 1 -> ceph-node01 / osd.0
      Replica 2 -> ceph-node01 / osd.1
      Replica 3 -> ceph-node01 / osd.2

Although size=3, all replicas are on the same host.

If ceph-node01 fails, all three replicas become unavailable simultaneously.

Correct example:

    Object A:
      Replica 1 -> ceph-node01 / osd.0
      Replica 2 -> ceph-node02 / osd.2
      Replica 3 -> ceph-node03 / osd.4

This configuration can withstand single-host failures.

---

### 8.3 The number of failure domains must meet the replica count

If:

    size = 3
    failure domain = host

At least need:

    3 available hosts

If:

    size = 3
    failure domain = rack

At least need:

    3 available racks

Otherwise may occur:

- PG undersized
- PG degraded
- Unable to place sufficient replicas
- Long-term HEALTH_WARN

Core principle:

    The replica count cannot exceed the number of available failure domains.

---

## Nine. Experiment Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | View default CRUSH Tree and CRUSH Rule | Low |
| Experiment 2 | Validate default host-level replica distribution | Low |
| Experiment 3 | Create host-level CRUSH Rule and test Pool | Medium |
| Experiment 4 | View PG to OSD mapping | Low |
| Experiment 5 | Build logical rack topology | High |
| Experiment 6 | Create rack-level CRUSH Rule and test Pool | High |
| Experiment 7 | Validate rack-level replica distribution | Medium-High |
| Experiment 8 | Simulate insufficient failure domains causing PG undersized | High |
| Experiment 9 | View and understand device class | Low |
| Experiment 10 | Clean up test Pool and experiment rules | High |

Notes:

    Viewing-type experiments carry low risk.
    Creating test Pools carries medium risk.
    Modifying CRUSH topology, moving hosts, switching Pool CRUSH Rules are high-risk operations.
    Do not directly apply experimental commands to production environments.

---

## Ten. Experiment 1: View Default CRUSH Tree and CRUSH Rule

### 10.1 View CRUSH Tree

    ceph osd crush tree

Expected example:

    ID  CLASS  WEIGHT   TYPE NAME
    -1         3.00000  root default
    -3         1.00000      host ceph-node01
     0    hdd  1.00000          osd.0
    -5         1.00000      host ceph-node02
     1    hdd  1.00000          osd.1
    -7         1.00000      host ceph-node03
     2    hdd  1.00000          osd.2

Key observations:

- Is root default?
- Is there a host level?
- Are OSDs under corresponding hosts?
- Is OSD class hdd/ssd/nvme?
- Is OSD weight normal?

---

### 10.2 View OSD Tree

    ceph osd tree

OSD Tree is more suitable for daily viewing:

- OSD status up/down
- Is OSD in/out?
- Host of OSD
- OSD weight
- reweight

---

### 10.3 View CRUSH Rule List

    ceph osd crush rule ls

Common output:

    replicated_rule

---

### 10.4 View Default CRUSH Rule

    ceph osd crush rule dump replicated_rule

Focus on:

    chooseleaf firstn 0 type host

If you see:

    type host

It indicates the default rule uses host-level failure domains.

---

## Eleven. Experiment 2: Validate Default host-level Replica Distribution

### 11.1 Create Test Pool

Create Pool:

    ceph osd pool create crush-default-demo 32

Enable application type:

    ceph osd pool application enable crush-default-demo rbd

Set replica count:

    ceph osd pool set crush-default-demo size 3
    ceph osd pool set crush-default-demo min_size 2

View Pool's CRUSH Rule:

    ceph osd pool get crush-default-demo crush_rule

Expected:

    crush_rule: replicated_rule

---

### 11.2 Create RBD Image to Generate Data

    rbd create demo-image --size 1G -p crush-default-demo

View:

    rbd ls -p crush-default-demo
    ceph -s
    ceph pg stat

---

### 11.3 View Pool ID

    ceph osd lspools

Example:

    2 crush-default-demo

Assume Pool ID is 2.

---

### 11.4 View PG Mapping

View several PGs:

    ceph pg map 2.0
    ceph pg map 2.1
    ceph pg map 2.2

Output example:

    pg 2.0 maps to up [0,1,2] acting [0,1,2]

Meaning:

| Field | Description |
|---|---|
| up | Set of target OSDs calculated by CRUSH |
| acting | Set of OSDs currently hosting the PG |

Combine with OSD Tree to verify:

    ceph osd tree

Confirm:

    Are osd.0, osd.1, osd.2 distributed across different hosts?

---

### 11.5 Experiment Conclusion

If the Pool uses host-level failure domains, and there are 3 hosts with size=3, then each PG's 3 replicas will be distributed across different hosts as much as possible.

This demonstrates:

size determines the number of replicas.
CRUSH Rule determines the distribution location of replicas.

---

## TwelveI don't know.Experiment Three: Creating a host-level CRUSH Rule and Testing Pool

### 12.1 Creating a host-level rule

Create a clear host-level rule:

    ceph osd crush rule create-replicated replicated-host default host

Check:

    ceph osd crush rule ls

Check rule details:

    ceph osd crush rule dump replicated-host

Expected to see:

    type host

---

### 12.2 Creating a Pool Using This Rule

    ceph osd pool create crush-host-demo 32 replicated replicated-host

Enable application type:

    ceph osd pool application enable crush-host-demo rbd

Set replica count:

    ceph osd pool set crush-host-demo size 3
    ceph osd pool set crush-host-demo min_size 2

Check:

    ceph osd pool get crush-host-demo crush_rule
    ceph osd pool get crush-host-demo size
    ceph osd pool get crush-host-demo min_size

Expected:

    crush_rule: replicated-host
    size: 3
    min_size: 2

---

### 12.3 Creating RBD Image

    rbd create demo-image --size 1G -p crush-host-demo

Check:

    rbd info crush-host-demo/demo-image

---

### 12.4 Verifying PG Distribution

Check Pool ID:

    ceph osd lspools

Assume Pool ID is 3.

Check PG:

    ceph pg map 3.0
    ceph pg map 3.1
    ceph pg map 3.2

Combine with:

    ceph osd tree

Confirm if replicas are distributed across different hosts.

---

## ThirteenI don't know.Experiment Four: Viewing PG to OSD Mapping Details

### 13.1 Viewing PG List

    ceph pg dump pgs_brief

Filter for a specific Pool ID:

    ceph pg dump pgs_brief | grep '^3\.'

---

### 13.2 Viewing PG map

    ceph pg map 3.0

Focus on:

- up set
- acting set
- primary OSD

---

### 13.3 Viewing PG query

    ceph pg 3.0 query

Focus on:

- state
- up
- acting
- primary
- recovery_state
- blocked_by

Note:

    The ceph pg query output is lengthy, suitable for deep troubleshooting.
    For daily data location understanding, prefer using ceph pg map.

---

## FourteenI don't know.Experiment Five: Building Logical rack Topology

### 14.1 High-risk Warning

This experiment will modify the CRUSH topology.

Only recommended to execute in a test environment.

In production environments:

    rack must correspond to real physical racks.
    Modifying CRUSH topology may cause data migration.
    Should not arbitrarily move hosts.
    Must have change evaluation and rollback plans before modification.

---

### 14.2 Logical rack Planning

In the experiment, place 3 nodes into 3 logical racks:

| Rack | Node |
|---|---|
| rack01 | ceph-node01 |
| rack02 | ceph-node02 |
| rack03 | ceph-node03 |

Target structure:

    root default
      rack rack01
        host ceph-node01
      rack rack02
        host ceph-node02
      rack rack03
        host ceph-node03

Note:

    This is only a logical rack for the experiment.
    In production environments, rack should correspond to real physical racks, power domains, or switch domains.

---

### 14.3 Record Current CRUSH Tree Before Operation

Execute:

    ceph osd crush tree

Recommend saving output:

    ceph osd crush tree > /root/crush-tree-before-rack.txt

View:

    cat /root/crush-tree-before-rack.txt

---

### 14.4 Creating rack bucket

    ceph osd crush add-bucket rack01 rack
    ceph osd crush add-bucket rack02 rack
    ceph osd crush add-bucket rack03 rack

Check:

    ceph osd crush tree

---

### 14.5 Move rack under root default

    ceph osd crush move rack01 root=default
    ceph osd crush move rack02 root=default
    ceph osd crush move rack03 root=default

Check:

    ceph osd crush tree

---

### 14.6 Move host under corresponding rack

    ceph osd crush move ceph-node01 rack=rack01
    ceph osd crush move ceph-node02 rack=rack02
    ceph osd crush move ceph-node03 rack=rack03

Check:

    ceph osd crush tree

Expected structure:

    root default
      rack rack01
        host ceph-node01
      rack rack02
        host ceph-node02
      rack rack03
        host ceph-node03

---

### 14.7 Observing Cluster Status

After modifying the CRUSH topology, observe:

    ceph -s
    ceph pg stat
    ceph osd df

If only moving host level, it may trigger remap or backfill.

Continuously observe:

    watch -n 5 'ceph -s'

Goal:

    Eventually recover active+clean

---

## FifteenI don't know.Experiment Six: Creating a Rack-Level CRUSH Rule and Testing Pool

### 15.1 Creating a Rack-Level Rule

    ceph osd crush rule create-replicated replicated-rack default rack

Check:

    ceph osd crush rule ls

Check details:

    ceph osd crush rule dump replicated-rack

Expected to see:

    type rack

---

### 15.2 Creating a Pool Using the Rack Rule

    ceph osd pool create crush-rack-demo 32 replicated replicated-rack

Enable application type:

    ceph osd pool application enable crush-rack-demo rbd

Set replicas:

    ceph osd pool set crush-rack-demo size 3
    ceph osd pool set crush-rack-demo min_size 2

Check:

    ceph osd pool get crush-rack-demo crush_rule
    ceph osd pool get crush-rack-demo size
    ceph osd pool get crush-rack-demo min_size

Expected:

    crush_rule: replicated-rack
    size: 3
    min_size: 2

---

### 15.3 Creating RBD Image

    rbd create demo-image --size 1G -p crush-rack-demo

Check:

    rbd info crush-rack-demo/demo-image

---

### 15.4 Observing Cluster Status

    ceph -s
    ceph pg stat

If the number of racks meets the replica count, eventually recover:

    active+clean

If the number of racks is insufficient, may appear:

    undersized
    degraded

---

## SixteenI don't know.Experiment Seven: Verifying Rack-Level Replica Distribution

### 16.1 Check Pool ID

    ceph osd lspools

Assume:

    The Pool ID of crush-rack-demo is 4

---

### 16.2 Check PG Mapping

    ceph pg map 4.0
    ceph pg map 4.1
    ceph pg map 4.2

Example output:

    pg 4.0 maps to up [0,1,2] acting [0,1,2]

---

### 16.3 Verifying Rack with CRUSH Tree

Check:

    ceph osd crush tree

Confirm:

    osd.0 belongs to rack01
    osd.1 belongs to rack02
    osd.2 belongs to rack03

If a PG's acting set is distributed across three different racks, it indicates the rack-level failure domain is effective.

---

## SeventeenI don't know.Experiment Eight: Simulating Failure Domain Insufficiency Leading to PG undersized

### 17.1 Experiment Description

This experiment is only for understanding the phenomenon of insufficient failure domains.

High-risk warning:

    Not recommended for production environments.
    Not recommended for existing business pools.
    Only allowed to use test pools.
    Suggest ensuring no business data before operation.

---

### 17.2 Typical Error Scenario

For example:

    Pool size = 3
    CRUSH Rule failure domain = rack
    But only 2 racks have available OSDs

Ceph cannot find 3 different racks to place 3 replicas.

May appear:

    PG undersized
    PG degraded
    HEALTH_WARN

---

### 17.3 Troubleshooting Commands

Check cluster:

    ceph -s

Check health details:

    ceph health detail

Check pool replica count:

    ceph osd pool get crush-rack-demo size

Check CRUSH Rule used by pool:

    ceph osd pool get crush-rack-demo crush_rule

Check rule details:

    ceph osd crush rule dump replicated-rack

Check CRUSH Tree:

    ceph osd crush tree

Judge:

    Whether the number of available racks is less than size.

---

### 17.4 Handling Methods

Optional handling methods:

1. Add sufficient racks and OSDs.
2. Lower the failure domain level, e.g., from rack back to host.
3. Temporarily lower size, suitable for test environments, cautious in production.
4. Repair the faulty rack or OSD.
5. Check if the CRUSH topology is configured incorrectly.

Production recommendations:

    Prioritize fixing the topology or supplementing the failure domain.
    Not recommended to blindly lower replica count to eliminate warnings.

---

## EighteenI don't know.Experiment Nine: Viewing and Understanding Device Class

### 18.1 Viewing Device Class

    ceph osd crush class ls

Possible output:

    hdd
    ssd
    nvme

Check CRUSH Tree:

    ceph osd crush tree

Can see the CLASS field:

    osd.0 hdd
    osd.1 hdd
    osd.2 ssd

---

### 18.2 Why Need Device Class

Different disks have different performance.

| Device Type | Characteristics | Typical Use |
|---|---|---|
| HDD | Large capacity, low cost, weak random IO | Cold data, object storage |
| SSD | Better performance, moderate cost | General block storage, file storage |
| NVMe | Strong performance, high cost | High-performance databases, critical business |

Different pools can use different device classes through CRUSH rules.

Example: /think

Database RBD Pool uses SSD.
Cold data object Pool uses HDD.
High-performance business Pool uses NVMe.

---

### 18.3 Setting Device Class

If an OSD is identified incorrectly, it can be manually set.

Example:

    ceph osd crush set-device-class ssd osd.0

Check:

    ceph osd crush tree

Remove class:

    ceph osd crush rm-device-class osd.0

High-risk warning:

    If there are existing CRUSH Rules based on device class, modifying the class may affect data placement.
    Production environment must be assessed before operation.

---

### 18.4 Creating a CRUSH Rule Example Based on SSD

If there is an ssd class, you can create:

    ceph osd crush rule create-replicated replicated-ssd default host ssd

Check:

    ceph osd crush rule dump replicated-ssd

Create Pool:

    ceph osd pool create rbd-ssd-demo 32 replicated replicated-ssd
    ceph osd pool application enable rbd-ssd-demo rbd

Note:

    This example is only suitable for environments with real SSD OSDs.
    If the experimental environment has only HDD, this step is not needed.

---

## NineteenI don't know.Experiment Ten: Cleaning Test Resources

### 19.1 Deleting Test RBD Image

Check:

    rbd ls -p crush-default-demo
    rbd ls -p crush-host-demo
    rbd ls -p crush-rack-demo

Delete:

    rbd rm crush-default-demo/demo-image
    rbd rm crush-host-demo/demo-image
    rbd rm crush-rack-demo/demo-image

If a Pool does not exist or has no images, you can ignore the corresponding command.

---

### 19.2 Deleting Test Pool

Enable Pool deletion:

    ceph config set mon mon_allow_pool_delete true

Delete:

    ceph osd pool rm crush-default-demo crush-default-demo --yes-i-really-really-mean-it
    ceph osd pool rm crush-host-demo crush-host-demo --yes-i-really-really-mean-it
    ceph osd pool rm crush-rack-demo crush-rack-demo --yes-i-really-really-mean-it

Disable Pool deletion:

    ceph config set mon mon_allow_pool_delete false

Check:

    ceph osd pool ls

High-risk warning:

    Deleting a Pool will delete all data in it.
    Production environment must be backed up and approved before operation.

---

### 19.3 Deleting Test CRUSH Rule

Before deletion, confirm that no Pool is using the rule:

    ceph osd pool ls detail | grep crush_rule

Delete test rules:

    ceph osd crush rule rm replicated-host
    ceph osd crush rule rm replicated-rack

Check:

    ceph osd crush rule ls

If deletion fails, it usually means the rule is still in use by a Pool.

---

### 19.4 Whether to Restore Rack Topology

If it's just an experimental environment, you can keep the rack logical topology for future rack-level failure domain experiments.

If you want to restore to the default host structure, you need to carefully move hosts back to root default.

Example:

    ceph osd crush move ceph-node01 root=default
    ceph osd crush move ceph-node02 root=default
    ceph osd crush move ceph-node03 root=default

Delete empty racks:

    ceph osd crush remove rack01
    ceph osd crush remove rack02
    ceph osd crush remove rack03

Check:

    ceph osd crush tree
    ceph -s

High-risk warning:

    Moving hosts will change the CRUSH topology.
    It may trigger data remap.
    Do not execute in production environments.
    Even in experimental environments, it's recommended to restore topology after cleaning test Pools.

---

## TwentyI don't know.Common Faults and Troubleshooting

### 20.1 PG Undersized

Phenomenon:

    ceph -s shows undersized

Common causes:

- Pool size exceeds the number of available failure domains
- Insufficient OSDs
- CRUSH Rule failure domain level is too high
- OSD down
- OSD out
- No available OSDs in a rack / host

Troubleshoot:

    ceph health detail
    ceph osd tree
    ceph osd crush tree
    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> crush_rule
    ceph osd crush rule dump <rule-name>

Handling direction:

    Increase available failure domains.
    Restore failed OSDs.
    Fix CRUSH topology.
    Lower failure domain level, requires assessment.
    Adjust replica count, requires caution.

ceph osd df  
ceph osd tree  
ceph osd crush tree  
ceph pg stat  
ceph osd pool autoscale-status  

Do not rashly adjust reweight.

---

### 20.3 After Modifying CRUSH, Large Backfill  

**Phenomenon:**  

    ceph -s showsMass backfilling / remapped  

**Cause:**  

    CRUSH topology or rule changes caused data remapping.  

**Troubleshooting:**  

    ceph -s  
    ceph -w  
    ceph pg stat  
    ceph osd df  

**Handling:**  

    Monitor recovery progress.  
    Avoid further high-risk changes.  
    Assess business IO impact.  
    Control recovery parameters during off-peak hours if necessary.  

---

### 20.4 Pool Uses Incorrect CRUSH Rule  

**Check:**  

    ceph osd pool get <pool-name> crush_rule  

If the Pool is bound to an incorrect rule, evaluate and modify:  

    ceph osd pool set <pool-name> crush_rule <rule-name>  

**High-Risk Warning:**  

    Changing a Pool's crush_rule will cause data remapping.  
    Production environments must evaluate the impact.  

---

### 20.5 Host Moved to Wrong Rack  

**Check:**  

    ceph osd crush tree  

If the host is moved to the wrong rack, reposition it:  

    ceph osd crush move ceph-node01 rack=rack01  

Then observe:  

    ceph -s  
    ceph pg stat  

**Note:**  

    Moving a host may trigger remap.  
    Exercise caution in production environments.  

---

## Twenty-one, Production Environment Considerations  

### 21.1 Don't Only Look at Size  

In production, you cannot simply say:  

    Pool size=3, so it's safe.  

You must also confirm:  

- Are replicas distributed across different hosts  
- Are they across different racks  
- Are they across different power domains  
- Are they across different switches  
- Does the CRUSH Rule match the real physical topology  

---

### 21.2 CRUSH Topology Must Reflect Real Physical Environment  

If the real environment has racks, they should be reflected in CRUSH.  

If the real environment has different data centers, plan accordingly at the architecture level.  

**Wrong Practice:**  

    Real physical environment has racks, but all hosts are under default root in CRUSH.  

**Issues:**  

    Ceph doesn't know the real rack failure boundaries.  
    Replica distribution may not meet expectations.  

---

### 21.3 Modifying CRUSH Is a High-Risk Change  

Be cautious with the following operations:  

- Moving a host to a different rack  
- Changing a Pool's crush_rule  
- Modifying CRUSH weight  
- Deleting a bucket  
- Changing device class  
- Modifying failure domain  
- Modifying root structure  

Before changes, must:  

    ceph -s  
    ceph health detail  
    ceph osd tree  
    ceph osd crush tree  
    ceph osd df  
    ceph pg stat  

And confirm:  

- Cluster is currently healthy  
- Sufficient capacity  
- Business low-traffic period  
- Rollback plan  
- Monitoring observation  
- Change records  

---

### 21.4 Rack-Level High Availability Requires Sufficient Racks  

To achieve:  

    Rack-level failure domain  
    size = 3  

At least need:  

    3 racks  

Otherwise, it may not meet replica distribution requirements.  

---

### 21.5 Device Class Should Be Combined with Business Planning  

Do not simply mix all disks into one Pool.  

**Recommendation:**  

| Business | Recommended Devices |  
|---|---|  
| Database RBD | SSD / NVMe |  
| General File Sharing | SSD or HDD, depending on performance requirements |  
| Object Cold Data | HDD |  
| High-Performance Business | NVMe |  

Different Pool can be bound to different device classes via CRUSH Rule.  

---

## Twenty-two, Advanced SRE Methodology  

### 22.1 CRUSH Is One of Ceph's High Availability Core Components  

Advanced SREs assess Ceph's high availability, not just:  

    How many replicas.  

But also:  

    Where the replicas are placed.  

Correct analysis sequence:  

    Pool size  
      |  
      v  
    min_size  
      |  
      v  
    CRUSH Rule  
      |  
      v  
    Failure Domain  
      |  
      v  
    CRUSH Tree  
      |  
      v  
    OSD Distribution  
      |  
      v  
    Real Physical Topology  

---

### 22.2 Failure Domain Must Match Real Fault Boundaries  

If a rack shares:  

- The same switch  
- The same PDU  
- The same uplink link  
- The same cabinet power  

Then the rack is a failure domain.  

If CRUSH lacks rack hierarchy, Ceph cannot perceive this risk.  

---

### 22.3 Data Migration Is the Cost of CRUSH Changes  

Any CRUSH change isn't just a configuration change.  

It may bring:  

- Data remap  
- Backfill  
- Recovery  
- Network consumption  
- OSD IO consumption  
- Business performance degradation  
- PG status changes  

Thus, CRUSH changes in production should be treated as formal storage changes.  

---

### 22.4 Replica Safety Equals Replica Count Plus Replica Location  

You can understand it as:  

    Replica count solves the "how many copies" problem.  
    CRUSH failure domain solves the "where to place" problem.  

True data safety requires both to be reasonable.  

---

## Twenty-three, Interview Answer Approach  

If asked in an interview:  

    What is Ceph's CRUSH? Why is it important?  

You can answer:  

    CRUSH is Ceph's core component for data distribution. It determines how data is replicated across the cluster. By defining a hierarchy of failure domains (like racks, hosts, etc.), CRUSH ensures data redundancy and fault tolerance. A well-designed CRUSH configuration minimizes data loss risks and optimizes cluster performance.

# CRUSH is Ceph's data placement algorithm, full name Controlled Replication Under Scalable Hashing. It is responsible for mapping objects or PGs to a group of OSDs based on the CRUSH Map, Pool's CRUSH Rule, OSD weights, device types, and failure domains.
Ceph does not rely on a centralized metadata table to record where each object is stored, but instead uses CRUSH to let clients and OSDs calculate data locations based on the cluster topology, making it more suitable for large-scale distributed storage.
CRUSH is crucial because Ceph's high availability depends not only on the number of replicas but also on where the replicas are placed. For example, if a Pool is set to size=3 and all three replicas fall on the same host, the system will still have issues if the host fails. Through CRUSH Rules, you can specify failure domains like host, rack, and datacenter, distributing replicas across different failure boundaries.
In production environments, CRUSH Maps are designed based on real physical topologies, such as at least host-level failure domains, and multi-rack environments can achieve rack-level failure domains. Modifying CRUSH rules will cause data remapping, potentially triggering backfill and recovery, so changes must be performed during business low-peak periods after assessment.

---

## Twenty-Four, Summary of This Article

This article mainly organizes Ceph CRUSH rules and failure domain design:

1. CRUSH is Ceph's data placement algorithm.
2. CRUSH Map describes Ceph's storage topology.
3. CRUSH Rule determines how Pool data is placed across failure domains.
4. size indicates the number of replicas, and failure domains determine replica distribution locations.
5. size=3 does not necessarily mean safety; it must be checked whether replicas are spread across host or rack.
6. Host-level failure domains can withstand single host failures.
7. Rack-level failure domains can withstand single rack failures.
8. size=3 + rack-level failure domains require at least 3 racks.
9. device class can differentiate HDD, SSD, NVMe.
10. Modifying CRUSH rules may trigger large-scale data migration.
11. In production environments, CRUSH topology must reflect real physical failure boundaries as much as possible.
12. Advanced SREs managing Ceph must simultaneously consider Pool, PG, CRUSH Rule, CRUSH Tree, and real physical topology.
13. CRUSH changes are not ordinary configuration changes but storage changes that may affect data location and business performance.

---

## Twenty-Five, Reference Documents

Ceph CRUSH Map official documentation:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph CRUSH Rule documentation:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/#crush-rules

Ceph CRUSH Map editing:

    https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/

Ceph Placement Groups:

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph Pool management:

    https://docs.ceph.com/en/latest/rados/operations/pools/

Ceph Device Classes:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/#device-classes

Ceph Health Checks:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/