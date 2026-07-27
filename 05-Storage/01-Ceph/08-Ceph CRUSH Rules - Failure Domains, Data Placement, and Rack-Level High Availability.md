# Ceph CRUSH Rules: Failure Domains, Data Placement, and Rack-Level High Availability

Recommended Path: 05-Storage/01-Ceph/08-Ceph CRUSH Rules: Failure Domains, Data Placement, and Rack-Level High Availability.md

Tags: #Ceph #CRUSH #Failure Domain #High Availability #OSD #Pool #PG #Data Distribution #Rack-Level High Availability #SRE #Distributed Storage #Advanced SRE

---

## I. Document Overview

This article is the eighth in the Ceph Advanced SRE storage series, focusing on CRUSH rules, failure domains, data placement, and rack-level high availability design.

Previously learned topics include:

- Ceph's core architecture
- RADOS, MON, MGR, OSD
- Pools and PGs
- Number of replicas (size) and minimum size (min_size)
- Adding, removing, replacing, and scaling OSDs
- PG status and basic data recovery

This article addresses a key question:

    How does Ceph determine which OSDs to store data copies on?

Ceph does not randomly assign data to disks but uses the CRUSH algorithm, based on the CRUSH Map, Pool-bound CRUSH Rules, OSD weights, device types, failure domains, and the number of replicas, to calculate the data placement.

Understanding CRUSH is essential to grasp:

- Why setting size=3 does not necessarily ensure security
- Why replicas must be distributed across hosts/racks
- Why adding new OSDs triggers rebalancing
- Why modifying CRUSH rules causes data migration
- Why failure domains are necessary in production environments
- How to upgrade from single-node high availability to rack-level high availability
- Why changing CRUSH rules is considered a high-risk storage operation

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the function and basic principles of CRUSH.
2. View the current CRUSH Tree.
3. Check the default CRUSH Rules.
4. Comprehend the hierarchical relationships between root, rack, host, and OSD.
5. Recognize the role of failure domains.
6. Create host-level CRUSH Rules.
7. Establish Pools using specified CRUSH Rules.
8. Verify the CRUSH Rules used by a Pool.
9. Use `ceph pg map` to inspect PG OSD distribution.
10. Construct logical rack topologies.
11. Create rack-level CRUSH Rules.
12. Confirm that replicas are distributed across racks.
13. Understand why incorrect failure domains can lead to undersized PGS.
14. Appreciate the value of device class in distinguishing between HDD, SSD, and NVMe.
15. Be aware of the boundaries for high-risk operations such as modifying CRUSH Rules, moving hosts, or adjusting weights.

---

## III. Experimental Environment

The same Ceph module experimental environment is used throughout this article.

| IP | Host Name | Role | Description |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON/MGR/OSD | Ceph Node 1 |
| 10.0.0.32 | ceph-node02 | MON/MGR/OSD | Ceph Node 2 |
| 10.0.0.33 | ceph-node03 | MON/MGR/OSD | Ceph Node 3 |
| 10.0.0.34 | ceph-node04 | OSD/Expansion/Failure Testing | Optional |

Primary experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Prerequisites:

- The Ceph cluster has been initialized using `cephadm`.
- There are at least 3 OSDs.
- OSDs are distributed across ceph-node01, ceph-node02, and ceph-node03.
- MON quorum is functioning properly.
- MGR is operational.
- The command `ceph -s` can be executed successfully.
- PGS should preferably be in an active+clean state.

---

## IV. Pre-Operation Checks

CRUSH-related operations may trigger data remapping and migration. It is recommended to perform checks before starting any experiments.

### 4.1 Checking the Cluster Status

    `ceph -s`

Ideal status:

    health: HEALTH_OK

At minimum, you should confirm:

- MON is functioning normally.
- MGR is operational.
- All OSDs are up/in.
- PGSes are in an active+clean state.

If the current status shows HEALTH_ERR, it is not advisable to proceed with CRUSH experiments.

---

### 4.2 Checking Health Details

    `ceph health detail`

Any of the following issues requires immediate attention or documentation:

- OSDs down
- Degraded PGSes
- Undersized PGSes
- Near-full/full status                   │
       ┌───────────┼───────────┐
       │           │           │
       v           v           v
    host01      host02      host03
       │           │           │
    osd.0       osd.2       osd.4
    osd.1       osd.3       osd.5

In the event of a host failure, replicas are distributed across different hosts as much as possible.

---

### 7.2 Rack-Level Topology

Production environments may consist of multiple racks:

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

The goal of rack-level high availability is:

    When one rack fails, it does not affect all replicas.

---

### 7.3 Bucket Level

Common bucket types in CRUSH include:

| Type | Meaning |
|---|---|
| osd | Single OSD |
| host | Single host |
| rack | Rack |
| row | Cabinet row |
| room | Room |
| datacenter | Data center |
| root | Root level |

The experimental level is generally:

    root -> host -> osd

The production level may be:

    root -> rack -> host -> osd

More complex production levels could include:

    root -> datacenter -> room -> rack -> host -> osd

---

## VIII. The Relationship Between the Number of Replicas and Failure Domains

### 8.1 Size Only Represents the Number of Replicas

The size of a Pool indicates the number of replicas.

For example:

    size = 3

This means that:

    Normally, each object has 3 replicas.

However, a size of 3 does not necessarily mean that the system can withstand host or rack failures.

---

### 8.2 Failure Domains Determine Where Replicas Are Placed

Incorrect Example:

    Object A:
      Replica 1 -> ceph-node01 / osd.0
      Replica 2 -> ceph-node01 / osd.1
      Replica 3 -> ceph-node01 / osd.2

Although the size is 3, all replicas are on the same host.

If ceph-node01 fails, all three replicas will be unavailable.

Correct Example:

    Object A:
      Replica 1 -> ceph-node01 / osd.0
      Replica 2 -> ceph-node02 / osd.2
      Replica 3 -> ceph-node03 / osd.4

This way, a single host failure will not affect all replicas.

---

### 8.3 The Number of Failure Domains Must Match the Number of Replicas

If:

    size = 3
    failure domain = host

Then at least:

    3 available hosts are required.

If:

    size = 3
    failure domain = rack

Then at least:

    3 available racks are needed.

Otherwise, issues such as:

- PG undersizing
- PG degradation
- Inability to place enough replicas
- Persistent HEALTH_WARN messages

may occur.

Core principle:

    The number of replicas must not exceed the number of available failure domains.

---

## IX. Experimental Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | View the default CRUSH Tree and CRUSH Rule | Low |
| Experiment 2 | Verify the default host-level replica distribution | Low |
| Experiment 3 | Create a host-level CRUSH Rule and test the Pool | Medium |
| Experiment 4 | Check the mapping from PG to OSD | Low |
| Experiment 5 | Construct a logical rack topology | High |
| Experiment 6 | Create a rack-level CRUSH Rule and test the Pool | High |
| Experiment 7 | Verify the rack-level replica distribution | Medium to High |
| Experiment 8 | Simulate insufficient failure domains causing PG undersizing | High |
| Experiment 9 | Understand device classes | Low |
| Experiment 10 | Clean up test pools and experimental rules | High |

Note:

    Experiments that involve simply viewing data have a low risk level.
    Creating test pools carries medium-risk operations.
    Modifying CRUSH topologies, moving hosts, or changing Pool CRUSH Rules are considered high-risk tasks.
    Do not directly apply experimental commands in a production environment.

### 12.1 Creating Host-Level Rules

Create a specific host-level rule:

    ceph osd crush rule create-replicated replicated-host default host

View the rules:

    ceph osd crush rule ls

View detailed information about the rule:

    ceph osd crush rule dump replicated-host

The expected output should be:

    type host

---

### 12.2 Creating a Pool That Uses This Rule

    ceph osd pool create crush-host-demo 32 replicated replicated-host

Enable the application type for this pool:

    ceph osd pool application enable crush-host-demo rbd

Set the number of replicas:

    ceph osd pool set crush-host-demo size 3
    ceph osd pool set crush-host-demo min_size 2

View the settings of the pool:

    ceph osd pool get crush-host-demo crush_rule
    ceph osd pool get crush-host-demo size
    ceph osd pool get crush-host-demo min_size

The expected results are:

    crush_rule: replicated-host
    size: 3
    min_size: 2

---

### 12.3 Creating an RBD Image

    rbd create demo-image --size 1G -p crush-host-demo

View information about the created RBD image:

    rbd info crush-host-demo/demo-image

---

### 12.4 Verifying PG Distribution

View the Pool ID:

    ceph osd lspools

Assume the Pool ID is 3.

View the PG mapping:

    ceph pg map 3.0
    ceph pg map 3.1
    ceph pg map 3.2

Combine this with the CRUSH Tree information to confirm that the replicas are distributed across different hosts.

---

## Chapter Thirteen: Experiment Four: Viewing Details of PG to OSD Mapping

### 13.1 Viewing the List of PGs

    ceph pg dump pgs_brief

Filter for a specific Pool ID:

    ceph pg dump pgs_brief | grep '^3\.'

---

### 13.2 Viewing the PG Map

    ceph pg map 3.0

Pay attention to the following fields:

- up set
- acting set
- primary OSD

---

### 13.3 Viewing the PG Query

    ceph pg 3.0 query

Focus on the following fields:

- state
- up
- acting
- primary
- recovery_state
- blocked_by

Note:

    The output of ceph pg query can be quite long and is suitable for in-depth troubleshooting.
    For daily understanding of data locations, it is recommended to use ceph pg map.

---

## Chapter Fourteen: Experiment Five: Building a Logical Rack Topology

### 14.1 High-Risk Warning

This experiment will modify the CRUSH topology.

It is only recommended to perform this in a testing environment.

In a production environment:

    The racks must correspond to actual physical racks.
    Modifying the CRUSH topology may cause data migration.
    Do not move hosts arbitrarily.
    A change assessment and rollback plan must be prepared before making any modifications.

---

### 14.2 Planning Logical Racks

In this experiment, three nodes will be placed in three different logical racks:

| Rack | Node |
|---|---|
| rack01 | ceph-node01 |
| rack02 | ceph-node02 |
| rack03 | ceph-node03 |

The target structure is:

    root default
      rack rack01
        host ceph-node01
      rack rack02
        host ceph-node02
      rack rack03
        host ceph-node03

Note:

    These are just logical racks for the experiment.
    In a production environment, racks should correspond to actual physical racks, power domains, or switch domains.

---

### 14.3 Recording the Current CRUSH Tree Before Making Changes

Execute the following command:

    ceph osd crush tree

It is recommended to save the output:

    ceph osd crush tree > /root/crush-tree-before-rack.txt

View the saved file:

    cat /root/crush-tree-before-rack.txt

---

### 14.4 Creating Rack Buckets

    ceph osd crush add-bucket rack01 rack
    ceph osd crush add-bucket rack02 rack
    ceph osd crush add-bucket rack03 rack

View the updated CRUSH Tree:

    ceph osd crush tree

---

### 14.5 Moving Racks Under root default

    ceph osd crush move rack01 root=default
    ceph osd crush move rack02 root=default
    ceph osd crush move rack03 root=default

View the final CRUSH Tree:

    ceph osd crush tree

---

###osd.1 is located in rack02.
osd.2 is located in rack03.

If the acting set of a PG is distributed across three different racks, it indicates that the rack-level fault domain setting is effective.

---

## Section Seventeen: Experiment Eight: Simulating Insufficient Fault Domains Leading to an Under-sized PG

### 17.1 Experimental Description

This experiment is intended solely for understanding the phenomenon of insufficient fault domains.

High-risk Warning:

- It is not recommended to perform this in a production environment.
- Do not attempt this on existing service pools. Only use test pools.
- Ensure that no service data is present before proceeding.

---

### 17.2 Typical Error Scenarios

For example:

- Pool size = 3
- CRUSH Rule fault domain setting = rack
- However, only 2 racks have available OSDs

In this case, Ceph will be unable to find three different racks to place the three replicas.

Possible outcomes include:

- The PG becomes under-sized.
- The PG degrades in performance.
- A HEALTH_WARN is generated.

---

### 17.3 Diagnostic Commands

To check the cluster:

    ceph -s

For detailed health information:

    ceph health detail

To view the number of replicas per pool:

    ceph osd pool get crush-rack-demo size

To inspect the CRUSH Rule being used by the pool:

    ceph osd pool get crush-rack-demo crush_rule

To view the details of the rule:

    ceph osd crush rule dump replicated-rack

To examine the CRUSH Tree:

    ceph osd crush tree

To determine if the number of available racks is less than the specified size:

---

### 17.4 Resolution Methods

Possible solutions include:

1. Add sufficient racks and OSDs.
2. Reduce the level of the fault domain, for example, changing it from rack to host.
3. Temporarily reduce the pool size; this is acceptable in a test environment but should be done with caution in production.
4. Repair any faulty racks or OSDs.
5. Verify if there are any configuration errors in the CRUSH topology.

Production recommendations:

- Prioritize fixing the topology or ensuring that the fault domains are adequate.
- Avoid blindly reducing the number of replicas just to eliminate alerts.

---

## Section Eighteen: Experiment Nine: Viewing and Understanding Device Classes

### 18.1 Viewing Device Classes

    ceph osd crush class ls

Possible output:

    hdd
    ssd
    nvme

To view the CRUSH Tree:

    ceph osd crush tree

The CLASS field can be observed here:

    osd.0 hdd
    osd.1 hdd
    osd.2 ssd

---

### 18.2 Why Device Classes Are Needed

Different types of disks have varying performance characteristics.

| Device Type | Characteristics | Typical Uses |
|---|---|---|
| HDD | Large capacity, low cost, poor random I/O performance | Cold data storage, object storage |
| SSD | Higher performance, moderate cost | General block storage, file storage |
| NVMe | Extremely high performance, high cost | High-performance databases, critical applications |

By using CRUSH Rules, different pools can be configured to use different device classes.

Example:

- The RBD pool for the database uses SSDs.
- The cold data object pool uses HDDs.
- The high-performance business pool uses NVMe.

---

### 18.3 Setting Device Classes

If an OSD is incorrectly identified, it can be manually adjusted.

Example:

    ceph osd crush set-device-class ssd osd.0

To verify the setting:

    ceph osd crush tree

To remove a class setting:

    ceph osd crush rm-device-class osd.0

High-risk Warning:

- Modifying the device class for an existing CRUSH Rule may affect where data is stored.
- Such changes must be carefully evaluated before applying in a production environment.

---

### 18.4 Example of Creating a CRUSH Rule Based on SSDs

If the ssd class is available, you can create a rule like this:

    ceph osd crush rule create-replicated replicated-ssd default host ssd

To verify the rule:

    ceph osd crush rule dump replicated-ssd

To create a pool:

    ceph osd pool create rbd-ssd-demo 32 replicated replicated-ssd
    ceph osd pool application enable rbd-ssd-demo rbd

Note:

- This example is only suitable for environments where SSD OSDs are actually present.
- If the entire test environment consists of HDDs, this step is unnecessary.

---

## Section Nineteen: Experiment Ten: Clearing Up Test Resources

### 19.1 Deleting Test RBD Images

To list existing images:

    rbd ls -p crush-default### 20.3 Significant Backfill After Modifying CRUSH

**Phenomenon:**  
The `ceph -s` command shows a large amount of backfilling and remapping.

**Reason:**  
Changes in the CRUSH topology or rules cause data to be remapped.

**Troubleshooting:**  
Use the following commands for diagnosis:  
`ceph -s`, `ceph -w`, `ceph pg stat`, `ceph osd df`.  

**Action:**  
- Monitor the recovery progress.  
- Avoid making further high-risk changes.  
- Evaluate whether business I/O is affected.  
- If necessary, adjust recovery parameters during off-peak hours.

---

### 20.4 Using the Wrong CRUSH Rule for a Pool

**Check:**  
Run `ceph osd pool get <pool-name> crush_rule`.  

If an incorrect rule is found, modify it as follows:  
`ceph osd pool set <pool-name> crush_rule <rule-name>`.  

**High-Risk Warning:**  
Changing the `crush_rule` of a pool will cause data to be remapped. This must be carefully evaluated in a production environment.

---

### 20.5 Moving a Host to the Wrong Rack

**Check:**  
View the `ceph osd crush tree`.  

If a host is in the wrong rack, move it using:  
`ceph osd crush move ceph-node01 rack=rack01`.  

Then monitor changes with:  
`ceph -s`, `ceph pg stat`.  

**Note:**  
Moving a host may trigger data remapping. Proceed with caution in a production environment.

---

## Section 21: Best Practices for Production Environments

### 21.1 Don’t Rely Solely on Pool Size

In production, it’s not enough to consider only the `pool size`. You also need to verify:  
- Whether replicas are distributed across different hosts.  
- If they are spread across different racks.  
- Whether they are located in distinct power domains.  
- Whether they are connected through different switches.  
- Whether the CRUSH rules match the actual physical topology.

---

### 21.2 The CRUSH Topology Must Reflect the Physical Environment

If the physical environment includes multiple racks, these should be reflected in the CRUSH configuration. Similarly, if there are separate data centers, this should be taken into account at the architectural level.  

**Incorrect Practice:**  
If the physical environment has different racks but the CRUSH configuration groups all replicas on the same host under `default root`, Ceph will not be able to detect rack-level failures, leading to potential issues with replica distribution.

---

### 21.3 Modifying CRUSH Represents High-Risk Changes

The following operations require extreme caution:  
- Moving a host to a different rack.  
- Changing the `crush_rule` of a pool.  
- Adjusting CRUSH weights.  
- Deleting buckets.  
- Modifying device classes.  
- Changing failure domains.  
- Altering the root structure.  

Before making any changes, perform the following checks:  
`ceph -s`, `ceph health detail`, `ceph osd tree`, `ceph osd crush tree`, `ceph osd df`, `ceph pg stat`.  

Also confirm:  
- The cluster is in good health.  
- There is sufficient capacity available.  
- The operation can be performed during off-peak hours.  
- You have a plan for rollback in case of issues.  
- You have adequate monitoring in place.  
- All changes are properly documented.

---

### 21.4 Rack-Level High Availability Requires Sufficient Racks

To achieve rack-level failure domains and a `pool size` of 3, at least 3 racks are required. Otherwise, the desired replica distribution cannot be ensured.

---

### 21.5 Device Classes Should Be Selected Based on Business Requirements

Don’t simply put all disks in the same pool. Instead, assign different device classes based on the specific use cases:  
- **Database RBD**: SSDs/NVMe.  
- **General File Sharing**: SSDs or HDDs, depending on performance requirements.  
- **Cold Object Data**: HDDs.  
- **High-Performance Applications**: NVMe.  

You can use CRUSH rules to bind different device classes to specific pools.

---

## Section 22: Advanced SRE Methods

### 22.1 CRUSH Is a Key Factor in Ceph’s High Availability

Advanced SRE professionals consider more than just the number of replicas when evaluating Ceph’s high availability. They also focus on where these replicas are located.  

**Correct Analysis Order:**  
- `pool size` → `min_size` → `CRUSH Rule` → `Failure Domain` → `CRUSH Tree` → `OSD Distribution` → Actual Physical Topology.

---

### 22.2 Failure Domains Must Match the Actual Fault