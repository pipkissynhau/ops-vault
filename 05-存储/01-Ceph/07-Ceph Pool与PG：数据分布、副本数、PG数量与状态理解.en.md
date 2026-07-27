# Ceph Pools and Placement Groups: Understanding Data Distribution, Number of Replicas, and Status of Placement Groups

Recommended Path: 05-Storage/01-Ceph/07-Ceph Pools and Placement Groups: Understanding Data Distribution, Number of Replicas, and Status of Placement Groups.md

Tags: #Ceph #Pool #PG #PlacementGroup #Number of Replicas #CRUSH #OSD #Data Distribution #SRE #Distributed Storage #Advanced SRE

---

## I. Document Overview

This article is the seventh in the Ceph Advanced SRE storage module series, focusing on the theory, practical operations, and troubleshooting methods related to Pools and Placement Groups.

Previous topics covered include:

- Ceph Architecture
- RADOS, MON, MGR, OSD, CRUSH
- Three Types of Storage: RBD, CephFS, RGW
- Ceph Cluster Planning
- cephadm Cluster Initialization
- Adding, Removing, Replacing, and Expanding OSD Disks

Starting from this article, you will need to understand the core mechanisms behind Ceph's data distribution:

    Client
      |
      v
    Pool
      |
      v
    Object
      |
      v
    Placement Group
      |
      v
    CRUSH
      |
      v
    OSD Set

Where:

- A Pool is a logical storage container that defines the boundaries for business data and storage policies.
- A Placement Group serves as an intermediate mapping layer between objects and OSDs.
- CRUSH determines which OSDs will store the data based on the rules defined by the Placement Group and Pool, as well as fault domains.
- OSDs are the actual locations where data is stored.

This article not only explains these concepts but also includes practical experiments to demonstrate how to:

- Create a Pool
- Set the number of replicas
- Configure min_size
- Enable application-type settings
- Create an RBD Image
- Write test objects
- Check the status of Placement Groups
- View the mapping between Placement Groups and OSDs
- Adjust the number of replicas and observe cluster changes
- Review the recommendations provided by the PG Autoscaler
- Clean up experimental resources
- Troubleshoot common Pool/Placement Group-related issues

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the role and common parameters of Pools.
2. Comprehend the function and mapping relationship of Placement Groups.
3. Create a test Pool and enable application-type settings.
4. Configure the size and min_size of a Pool.
5. Use Rados to write and read test objects.
6. Create an RBD Image to verify the Pool's functionality.
7. Check the status of Placement Groups and their mapping to OSDs.
8. Use the `ceph pg map` command to understand the mapping between Placement Groups and OSDs.
9. Use `ceph osd pool autoscale-status` to view the recommendations from the PG Autoscaler.
10. Identify various states of Placement Groups, such as active+clean, degraded, undersized, stale, inconsistent, etc.
11. Troubleshoot issues like POOL_APP_NOT_ENABLED, PG degraded, PG undersized, PG stuck, etc.
12. Be aware of the potential risks associated with high-risk operations such as deleting Pools, adjusting Placement Group configurations, or modifying the number of replicas.

---

## III. Experimental Environment

The same experimental environment used in previous Ceph modules will be utilized for this article.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Testing (optional) |

Primary experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Prerequisites:

- The Ceph cluster has been initialized using `cephadm`.
- There are at least 3 OSDs.
- These 3 OSDs are distributed across nodes ceph-node01, ceph-node02, and ceph-node03.
- The MON quorum is functioning correctly.
- The MGR is operational.
- The command `ceph -s` can be executed successfully.

---

## IV. Pre-Experiment Checks

Before starting experiments with Pools and Placement Groups, ensure the cluster is in a stable state first.

### 4.1 Checking the Cluster Status

    Run the command `ceph -s`.

Ideal status:

    health: HEALTH_OK

At minimum, you should confirm that:

- MON isAll PGs should ideally be in the active+clean state.

Example:

33 PGs: 33 active+clean

---

### 9.4 Viewing Health Details

ceph health detail

If there are any alerts related to pools or PGs, record them first.

---

## Section Ten: Experiment Two: Creating a Test Pool

### 10.1 Creating the rbd-demo Pool

Creating a test pool:

ceph osd pool create rbd-demo 32

Explanation:

rbd-demo is the name of the pool.
32 represents the number of PGs.
For small experimental environments, using 32 PGs is sufficient.

To check:

ceph osd pool ls
ceph osd pool get rbd-demo pg_num

Expected result:

pg_num: 32

---

### 10.2 Enabling the RBD Application Type

Ceph requires specifying the purpose of a pool.

To enable RBD:

ceph osd pool application enable rbd-demo rbd

To check:

ceph osd pool application get rbd-demo

Expected result:

{
    "rbd": {}
}

If the application type is not enabled, you may receive an error message like:

POOL_APP_NOT_ENABLED

---

### 10.3 Viewing Detailed Pool Parameters

ceph osd pool get rbd-demo all

Pay special attention to:

- size
- min_size
- pg_num
- pgp_num
- crush_rule
- pg_autoscale_mode
- application

---

## Section Eleven: Experiment Three: Setting the Number of Replicas (size and min_size)

### 11.1 Checking the Current Number of Replicas

ceph osd pool get rbd-demo size
ceph osd pool get rbd-demo min_size

---

### 11.2 Setting the Number of Replicas to 3

Set the size to 3:

ceph osd pool set rbd-demo size 3

Set the min_size to 2:

ceph osd pool set rbd-demo min_size 2

Check again:

ceph osd pool get rbd-demo size
ceph osd pool get rbd-demo min_size

Expected result:

size: 3
min_size: 2

---

### 11.3 The Meaning of size and min_size

size = 3

This means that under normal circumstances, each object will have 3 replicas.

min_size = 2

This means that IO operations are allowed only when at least 2 replicas are available.

Common production configurations:

size = 3
min_size = 2

Explanation:

The higher the size, the greater the redundancy, but the lower the available capacity.
If the min_size is too low, it may increase data risks.
If the min_size is too high, it may reduce business availability in case of failures.

---

## Section Twelve: Experiment Four: Writing and Reading Objects Using rados

### 12.1 Creating a Test File

echo "hello ceph pool and pg" > /tmp/ceph-pool-pg.txt

To check:

cat /tmp/ceph-pool-pg.txt

---

### 12.2 Writing an Object

rados -p rbd-demo put object-demo /tmp/ceph-pool-pg.txt

To list objects:

rados -p rbd-demo ls

Expected result:

object-demo

---

### 12.3 Reading an Object

rados -p rbd-demo get object-demo /tmp/ceph-pool-pg.out

To check:

cat /tmp/ceph-pool-pg.out

Expected result:

hello ceph pool and pg

---

### 12.4 Checking the Number of Pool Objects

ceph df

Observe whether the number of OBJECTS in rbd-demo has increased.

---

### 12.5 Deleting the Test Object

rados -p rbd-demo rm object-demo

To confirm:

rados -p rbd-demo ls

---

## Section Thirteen: Experiment Five: Creating an RBD Image to Verify the Pool

### 13.1 Creating an RBD Image

rbd create demo-image --size 10G -p rbd-demo

To check:

rbd ls -p rbd-demo

Expected result:

demo-image

---

### 13.2 Checking RBD Image Information

rbd info rbd-demo/demo-image

You should see the following information:

- size: 10 GiB
- object size
- features
- op_features
- flags

---

### 13.3 Checking the Cluster Status

ceph -s
ceph pg stat
ceph df

Expected result:

The PG should ultimately be in the active+clean state.
The number and capacity of objects in the rbd-demo pool should have changed.

---

### 13.4 Deleting the RBD Image

rbd rm rbd-demo/demo-image

To confirm:

rbd ls -p rbd-demo

---

### 16.2 Viewing the Autoscale Mode of a Specified Pool

    `ceph osd pool get rbd-demo pg_autoscale_mode`

Common Modes:

| Mode | Meaning |
|---|---|
| on | Automatic adjustment |
| warn | Only provides recommendations |
| off | Disabled |

---

### 16.3 Setting to Warn Mode

It is recommended to use the warn mode during experimentation and in the initial stages of production:

    `ceph osd pool set rbd-demo pg_autoscale_mode warn`

To check the status:

    `ceph osd pool autoscale-status`

---

### 16.4 Setting the target_size_ratio

If it is expected that a particular Pool will use 50% of the cluster's capacity, you can set the following value:

    `ceph osd pool set rbd-demo target_size_ratio 0.5`

To check the status:

    `ceph osd pool autoscale-status`

Note:

    `target_size_ratio` is not a quota.
    It serves merely as a reference for the PG Autoscaler's capacity planning.

---

### 16.5 Whether to Enable on Mode

You can try enabling on mode in a testing environment.

In a production environment, proceed with caution:

    `ceph osd pool set rbd-demo pg_autoscale_mode on`

Risks:

    Automatic adjustment of the number of PGs may trigger data remapping.
    Data remapping could lead to backfilling.
    Backfilling will consume additional network and OSD I/O resources.

Recommendation:

    In production, start by using the warn mode.
    Decide whether to adjust the settings based on the recommendations and within the designated change window.

---

## Chapter Seventeen: Experiment Nine: Clearing Test Pools

### 17.1 Deleting RBD Images

If an RBD Image was created earlier:

    `rbd ls -p rbd-demo`

To delete it:

    `rbd rm rbd-demo/demo-image`

To confirm deletion:

    `rbd ls -p rbd-demo`

---

### 17.2 Deleting Objects

To list objects:

    `rados -p rbd-demo ls`

To delete an object:

    `rados -p rbd-demo rm object-demo`

---

### 17.3 Enabling Pool Deletion

To check the current setting:

    `ceph config get mon mon_allow_pool_delete`

To temporarily enable it in a testing environment:

    `ceph config set mon mon_allow_pool_delete true`

---

### 17.4 Deleting a Pool

    `ceph osd pool rm rbd-demo rbd-demo --yes-i-really-really-mean-it`

To confirm deletion:

    `ceph osd pool ls`

---

### 17.5 Disabling Pool Deletion

    `ceph config set mon mon_allow_pool_delete false`

To confirm the change:

    `ceph config get mon mon_allow_pool_delete`

Production Reminder:

    Deleting a Pool is a high-risk operation.
    All data within the Pool will be permanently removed.
    In a production environment, ensure that backups have been made and that changes have been approved before proceeding.
    After deletion, make sure to disable `mon_allow_pool_delete` again.

---

## Chapter Eighteen: Experiment Ten: Troubleshooting Common Pool/PG Issues

### 18.1 POOL_APP_NOT_ENABLED

Symptom:

    The message "pool application not enabled" appears when using `ceph -s`.

Cause:

    The pool has not been configured to use an application type.

To check:

    `ceph health detail`

Solution:

If the pool is for RBD:

    `ceph osd pool application enable <pool-name> rbd`

If the pool is for CephFS:

    `ceph osd pool application enable <pool-name> cephfs`

If the pool is for RGW:

    `ceph osd pool application enable <pool-name> rgw`

To confirm the setting:

    `ceph osd pool application get <pool-name>`

---

### 18.2 PG Degraded

Symptom:

    The message "degraded" appears when using `ceph -s`.

To check:

    `ceph health detail`
    `ceph pg stat`
    `ceph osd tree`

Common Causes:

- One or more OSDs are down.
- An OSD is out of the cluster.
- The number of replicas is insufficient.
- The total number of OSDs is too low.
- CRUSH Rules are restricting performance.
- Recovery processes are in progress.

Action Plan:

    First, check if any OSDs are down.
    Verify the pool size and minimum size settings.
    Examine the CRUSH fault domains.
    Check whether recovery efforts are underway.
    Avoid directly repairing or deleting the PG until the issues have been resolved.

---

### 18.3 PG Undersized

Meaning:

    The current number of PG replicas is less than the```markdown
🔤 ceph osd pool set <pool-name> target_size_ratio 0.5

---

### 19.7 RADOS Testing

    rados -p <pool-name> put <object-name> <local-file>
    rados -p <pool-name> ls
    rados -p <pool-name> get <object-name> <local-file>
    rados -p <pool-name> rm <object-name>

---

## Section 20: Considerations for Production Environments

### 20.1 Avoid Creating Pools Indiscriminately

Creating a Pool increases management complexity. Before setting up a Pool in production, clarify the following:

- Pool name
- Business purpose
- Application type
- Number of replicas
- Minimum size
- Number of Placement Groups (PGs)
- CRUSH Rule
- Capacity planning
- Alarm policies
- Whether quota management is required
- Whether backup measures are needed

---

### 20.2 Do Not Delete Pools Arbitrarily

Deleting a Pool is a high-risk operation. In production, ensure the following before deletion:

- Verify that the business has been shut down.
- Ensure no PVCs, RBDs, CephFS volumes, or RGW objects are still in use.
- Complete data backup before deletion.
- Obtain necessary approval for changes.
- Set `mon_allow_pool_delete` to `off` after deletion.

---

### 20.3 Do Not Rely Solely on the `size` Parameter

Misunderstanding:

- A `size` of 3 always guarantees safety.

Correct understanding:

- A `size` of 3 means there are 3 replicas. However, safety also depends on the CRUSH fault domain configuration. If all 3 replicas reside on the same host, a host failure remains risky.

---

### 20.4 Adjust the Number of PGs with Caution

Changing the number of PGs may lead to:

- Remapping
- Backfilling
- Recovery processes
- Increased OSD I/O operations
- Higher network traffic
- Decreased business performance

In production, follow these steps:

- Consult the recommendations from the `pg_autoscaler`.
- Assess the business during off-peak hours.
- Adjust the number of PGs in stages.
- Continuously monitor `ceph -s` and `ceph pg stat` for any changes.

---

### 20.5 Manage Capacity Levels Ahead of Time

Ceph cannot be operated like a regular disk, where 95% usage is acceptable before intervention. The reasons are:

- Ceph requires additional space for recovery and rebalancing.
- If the cluster approaches full capacity, recovery and backfilling processes may fail.
- Writing operations to a Pool might encounter errors.

Recommendations:

- Start monitoring when the capacity reaches 70%.
- Plan capacity expansion or data cleanup at 80%.
- Consider it a high-risk situation above 85%.
- Immediate action is required if the capacity is near full or at full.

---

## Section 21: Advanced SRE Methodologies

### 21.1 Pools Represent Business and Policy Boundaries

Advanced SRE professionals consider more than just the name of a Pool when evaluating it. Important factors include:

- Who will use this Pool?
- What type of data will be stored in it?
- How many replicas are required?
- What is the minimum size?
- Is the number of PGs appropriate?
- Does the CRUSH Rule match the fault domain requirements?
- Are there any capacity quotas in place?
- Are monitoring and alarm mechanisms configured?
- Do backup and recovery strategies exist?

---

### 21.2 Placement Groups Serve as Indicators of Faults and Recovery Progress

Many Ceph failures are reflected in the status of Placement Groups. Common issues include:

- `active+degraded`
- `undersized`
- `stale`
- `inconsistent`
- `remapped`
- `backfilling`
- `recovering`

Understanding the status of PGs helps determine whether data is being recovered, whether there are insufficient OSDs or replicas, or whether there are network or OSD-related issues. It also aids in identifying if recovery processes are slow or stuck.

---

### 21.3 The Balance Between `size` and `min_size` Ensures Safety and Availability

- `size` indicates the desired number of replicas.
- `min_size` specifies the minimum number of replicas required to support I/O operations.

If `min_size` is too low, data may continue to be written during failures, increasing risk. If it is too high, data protection is enhanced, but business operations may be interrupted more easily in case of failures. In production, do not blindly reduce `min_size` just to ensure continued data writes.

---

## Section 22: Interview Answer Guidelines

If interviewed about Pools and Placement Groups in Ceph, you can answer as