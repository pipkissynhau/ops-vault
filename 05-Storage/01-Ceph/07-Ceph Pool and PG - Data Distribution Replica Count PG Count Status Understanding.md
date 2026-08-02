# Ceph Pool and PG: Data Distribution, Replication, PG Count, and Status Understanding

Recommended path: 05-Storage/01-Ceph/07-Ceph Pool and PG: Data Distribution, Replication, PG Count, and Status Understanding.md

Tags: #Ceph #Pool #PG #PlacementGroup #NumberOfCopies #CRUSH #OSD #DataDistribution #SRE #DistributedStorage #AdvancedSre

---

## I. Document Overview

This is the seventh article in the Ceph advanced SRE storage module, focusing on the theory, practical operations, and troubleshooting methods for Pools and PGs.

Previously completed:

- Ceph basic architecture
- RADOS, MON, MGR, OSD, CRUSH
- RBD, CephFS, RGW three storage types
- Ceph cluster planning
- cephadm cluster initialization
- OSD disk addition, removal, replacement, and expansion

From this article onward, we need to understand the core data distribution chain in Ceph:

    Client
      |
      v
    Pool
      |
      v
    Object
      |
      v
    PG
      |
      v
    CRUSH
      |
      v
    OSD Set

Where:

- Pool is a logical storage pool, the boundary for business data and storage policies.
- PG is the intermediate mapping layer between Object and OSD.
- CRUSH determines where data is ultimately placed on OSDs based on PG, Pool rules, and failure domains.
- OSD is where data is actually stored.

This article not only explains concepts but also completes experiments:

- Creating a Pool
- Setting replication factor
- Setting min_size
- Enabling application type
- Creating RBD Image
- Writing test objects
- Checking PG status
- Checking PG to OSD mapping
- Adjusting replication factor and observing cluster changes
- Checking PG Autoscaler recommendations
- Cleaning up experimental resources
- Troubleshooting common Pool/PG alerts

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the role of Pools and common parameters.
2. Understand the role of PGs and their mapping relationships.
3. Create a test Pool and enable application type.
4. Set Pool size and min_size.
5. Use rados to write and read test objects.
6. Create RBD Image to verify Pool availability.
7. Check PG status and which OSDs the PG maps to.
8. Use ceph pg map to understand PG to OSD mapping.
9. Use ceph osd pool autoscale-status to check PG Autoscaler recommendations.
10. Identify PG statuses like active+clean, degraded, undersized, stale, inconsistent.
11. Troubleshoot issues like POOL_APP_NOT_ENABLED, PG degraded, PG undersized, PG stuck.
12. Clarify the boundaries for high-risk operations like Pool deletion, PG adjustment, and replication factor changes.

---

## III. Experimental Environment

This article continues using the Ceph module experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Prerequisites:

    Ceph cluster has been initialized with cephadm.
    At least 3 OSDs.
    3 OSDs distributed across ceph-node01, ceph-node02, ceph-node03.
    MON quorum is normal.
    MGR is normal.
    ceph -s can be executed normally.

---

## IV. Pre-Operation Checks

Before performing Pool and PG experiments, confirm the cluster status.

### 4.1 Check Cluster Status

    ceph -s

Ideal status:

    health: HEALTH_OK

At least confirm:

    mon is normal
    mgr is normal
    osd up/in
    pgs active+clean

If there are existing HEALTH_ERR, it's not recommended to proceed with Pool/PG adjustment experiments.

---

### 4.2 Check Health Details

    ceph health detail

If there are no anomalies, there may be no output or it will show healthy.

If there are anomalies, they need to be addressed or recorded first.

---

### 4.3 Check OSD Status

    ceph osd tree

Expected:

    All OSDs are up.
    OSDs are distributed across different hosts.
    No abnormal down OSDs.

---

### 4.4 Check PG Summary

    ceph pg stat

Expected similar to:

    1 pgs: 1 active+clean

Or:

    64 pgs: 64 active+clean

Key points:

    All PGs should be active+clean.

---

### 4.5 Check Capacity

    ceph df
    ceph osd df

Confirm:

    Cluster is not nearfull.
    OSD capacity is sufficient.
    No OSDs with obvious anomalies.

---

## V. What is a Pool

A Pool is a logical storage pool in Ceph.

It can be understood as:

    A collection of data with the same storage policy.

A Pool can be configured with:

- Replication factor size
- Minimum replication factor min_size
- Number of PGs pg_num
- CRUSH Rule
- Application type application
- Quota quota
- Erasure code policy
- Device category rules
- Automatic PG scaling policy

Common Pool examples:

    rbd
    cephfs_data
    cephfs_metadata
    .rgw.root
    default.rgw.buckets.data
    test-pool

Core role of a Pool:

    Isolate different business data.
    Set different storage policies for different businesses.
    Allow RBD, CephFS, RGW to use different data pools.
    Facilitate capacity, permissions, failure domain, and performance management.

---

## VI. What is a PG

PG stands for:

    Placement Group

PG is the intermediate mapping layer between Object and OSD.

In Ceph, the data mapping relationship is:

Object -> PG -> OSD Set

That is to say:

    Object does not directly map to OSD.
    Object first maps to PG.
    PG then maps to a group of OSDs via CRUSH.

Why PG is needed:

- Avoid direct OSD mapping for each object
- Reduce scheduling complexity for large numbers of objects
- Facilitate recovery and rebalancing
- Simplify state management
- Enable better data distribution control

Simple understanding:

    Pool is a logical pool.
    PG is a grouping within the pool.
    OSD is the final storage location.

---

## Seven. Pool and PG Relationship

A Pool contains multiple PGs.

Illustration:

    Pool: rbd-demo
      |
      ├── PG 1.0 -> osd.0, osd.1, osd.2
      ├── PG 1.1 -> osd.1, osd.2, osd.0
      ├── PG 1.2 -> osd.2, osd.0, osd.1
      └── PG 1.3 -> osd.0, osd.2, osd.1

When writing objects:

    Object A -> PG 1.0 -> osd.0, osd.1, osd.2
    Object B -> PG 1.2 -> osd.2, osd.0, osd.1

If Pool's size=3, each PG typically maps to 3 OSDs.

One is the primary OSD, the others are replica OSDs.

---

## Eight. Experiment Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | View existing Pools and PGs | Low |
| Experiment 2 | Create test Pool | Medium |
| Experiment 3 | Set size and min_size | Medium |
| Experiment 4 | Use rados to write and read objects | Low |
| Experiment 5 | Create RBD Image to verify Pool | Low |
| Experiment 6 | View PG to OSD mapping | Low |
| Experiment 7 | Adjust replica count to observe state changes | Medium-High |
| Experiment 8 | View PG Autoscaler recommendations | Low |
| Experiment 9 | Delete test Pool | High |
| Experiment 10 | Simulate common Pool/PG alarm troubleshooting | Medium-High |

High-risk warning:

    Deleting a Pool is a high-risk operation.
    Adjusting replica count may trigger recovery and data redistribution.
    Adjusting PG count may trigger remap and backfill.
    Do not directly use experimental commands in production environments.

---

## Nine. Experiment 1: View Existing Pools and PGs

### 9.1 View Pool List

    ceph osd pool ls

View detailed information:

    ceph osd pool ls detail

View Pool ID:

    ceph osd lspools

Expected example:

    1 .mgr

If there are existing RBD, CephFS, RGW, you may see more Pools.

---

### 9.2 View Cluster Capacity and Pool Capacity

    ceph df

Key fields:

| Field | Description |
|---|---|
| RAW STORAGE | Cluster raw capacity |
| POOLS | Pool-level capacity |
| STORED | Actual stored data |
| OBJECTS | Number of objects |
| USED | Used capacity |
| %USED | Usage rate |
| MAX AVAIL | Current theoretical available capacity for the Pool |

---

### 9.3 View PG Summary

    ceph pg stat

Expected:

    All PGs should be active+clean.

Example:

    33 pgs: 33 active+clean

---

### 9.4 View Health Details

    ceph health detail

If there are Pool or PG-related alarms, record them first.

---

## Ten. Experiment 2: Create Test Pool

### 10.1 Create rbd-demo Pool

Create test Pool:

    ceph osd pool create rbd-demo 32

Note:

    rbd-demo is the Pool name.
    32 is the pg_num.
    32 PGs are sufficient for small experimental environments.

View:

    ceph osd pool ls
    ceph osd pool get rbd-demo pg_num

Expected:

    pg_num: 32

---

### 10.2 Enable RBD Application Type

Ceph requires Pool to declare its purpose.

Enable RBD:

    ceph osd pool application enable rbd-demo rbd

View:

    ceph osd pool application get rbd-demo

Expected:

    {
        "rbd": {}
    }

If application type is not enabled, you may get:

    POOL_APP_NOT_ENABLED

---

### 10.3 View Pool Detailed Parameters

    ceph osd pool get rbd-demo all

Focus on:

- size
- min_size
- pg_num
- pgp_num
- crush_rule
- pg_autoscale_mode
- application

---

## Eleven. Experiment 3: Set Replica Count size and min_size

### 11.1 View Current Replica Count

    ceph osd pool get rbd-demo size
    ceph osd pool get rbd-demo min_size

---

### 11.2 Set Replica Count to 3

    ceph osd pool set rbd-demo size 3

Set min_size to 2:

    ceph osd pool set rbd-demo min_size 2

Check again:

    ceph osd pool get rbd-demo size
    ceph osd pool get rbd-demo min_size

Expected:

    size: 3
    min_size: 2

---

### 11.3 Meaning of size and min_size

    size = 3

Means each object is normally stored with 3 replicas.

    min_size = 2

Means IO is allowed when at least 2 replicas are available.

Common production configuration:

    size = 3
    min_size = 2

Note:

    Higher size means stronger redundancy but lower usable capacity.
    Too low min_size increases data risk.
    Too high min_size reduces business availability during failures.

## 12. Using rados to Write and Read Objects

### 12.1 Creating a Test File

    echo "hello ceph pool and pg" > /tmp/ceph-pool-pg.txt

Viewing:

    cat /tmp/ceph-pool-pg.txt

---

### 12.2 Writing an Object

    rados -p rbd-demo put object-demo /tmp/ceph-pool-pg.txt

Listing objects:

    rados -p rbd-demo ls

Expected output:

    object-demo

---

### 12.3 Reading an Object

    rados -p rbd-demo get object-demo /tmp/ceph-pool-pg.out

Viewing:

    cat /tmp/ceph-pool-pg.out

Expected output:

    hello ceph pool and pg

---

### 12.4 Checking Pool Object Count

    ceph df

Observe if the OBJECTS count for rbd-demo increases.

---

### 12.5 Deleting the Test Object

    rados -p rbd-demo rm object-demo

Verification:

    rados -p rbd-demo ls

---

## 13. Creating RBD Image to Verify Pool

### 13.1 Creating RBD Image

    rbd create demo-image --size 10G -p rbd-demo

Viewing:

    rbd ls -p rbd-demo

Expected output:

    demo-image

---

### 13.2 Checking RBD Image Information

    rbd info rbd-demo/demo-image

Expected to see:

- size 10 GiB
- object size
- features
- op_features
- flags

---

### 13.3 Checking Cluster Status

    ceph -s
    ceph pg stat
    ceph df

Expected:

    PG eventually becomes active+clean.
    The object count and capacity of rbd-demo Pool change.

---

### 13.4 Deleting RBD Image

    rbd rm rbd-demo/demo-image

Verification:

    rbd ls -p rbd-demo

---

## 14. Viewing PG to OSD Mapping

### 14.1 Checking Pool ID

    ceph osd lspools

Sample output:

    1 .mgr
    2 rbd-demo

Assuming:

    The Pool ID for rbd-demo is 2

---

### 14.2 Checking PG Status

    ceph pg stat

Viewing detailed PG list:

    ceph pg dump pgs_brief

If output is extensive, filter by Pool ID:

    ceph pg dump pgs_brief | grep '^2\.'

---

### 14.3 Checking Specific PG Mapping

Assuming PG exists:

    2.0

Checking mapping:

    ceph pg map 2.0

Expected output similar to:

    pg 2.0 maps to up [0,1,2] acting [0,1,2]

Meaning:

| Field | Description |
|---|---|
| up | Set of target OSDs calculated by CRUSH |
| acting | Set of OSDs currently hosting this PG |

Normally:

    up and acting are typically consistent.

They may differ during anomalies or recovery processes.

---

### 14.4 Analyzing with OSD Tree

Viewing:

    ceph osd tree

Verification:

    Confirm if the OSDs for PG 2.0 are distributed across different hosts.

Example:

    pg 2.0 -> osd.0, osd.1, osd.2

If:

    osd.0 is on ceph-node01
    osd.1 is on ceph-node02
    osd.2 is on ceph-node03

This indicates the 3 replicas of this PG are distributed across different hosts.

---

### 14.5 Checking PG Details

    ceph pg 2.0 query

Output contains extensive information, focus on:

- state
- up
- acting
- primary
- info
- recovery_state
- blocked_by

This command is commonly used for in-depth troubleshooting of PG issues.

---

## 15. Adjusting Replica Count and Observing Status Changes

### 15.1 High-Risk Warning

Changing replica count affects data redundancy and data migration.

This experiment can be performed in a test environment.

In production environments, must evaluate:

- Current cluster health status
- Available capacity
- Business impact
- Number of failure domains
- Change window
- Rollback plan

---

### 15.2 Changing size from 3 to 2

Checking current status:

    ceph -s
    ceph osd pool get rbd-demo size
    ceph osd pool get rbd-demo min_size

Modifying:

    ceph osd pool set rbd-demo size 2
    ceph osd pool set rbd-demo min_size 1

Observing:

    ceph -s
    ceph pg stat
    ceph osd pool get rbd-demo size
    ceph osd pool get rbd-demo min_size

May see temporary remapped or peering, eventually recovering to active+clean.

---

### 15.3 Restoring size to 3

Restoring:

    ceph osd pool set rbd-demo size 3
    ceph osd pool set rbd-demo min_size 2

Observing:

    watch -n 3 'ceph -s'

Checking:

    ceph pg stat
    ceph osd pool get rbd-demo size
    ceph osd pool get rbd-demo min_size

Final goal:

    active+clean

---

### 15.4 Experimental Conclusion

Through this experiment, we can see:

    Replica count is not a pure configuration item.
    Changing replica count may cause Ceph to adjust data replicas.
    Lowering replica count is not advisable in production environments.
    After restoring to size=3, waiting for replicas to complete is required.

---

## 16. Viewing PG Autoscaler

### 16.1 Checking Automatic Scaling Status /think

# ceph osd pool autoscale-status

Focus on the following in the output:

- POOL
- SIZE
- TARGET SIZE
- RATE
- RAW CAPACITY
- RATIO
- TARGET RATIO
- PG_NUM
- NEW PG_NUM
- AUTOSCALE

---

### 16.2 View the autoscale mode of a specified Pool

    ceph osd pool get rbd-demo pg_autoscale_mode

Common modes:

| Mode | Meaning |
|---|---|
| on | Auto-adjust |
| warn | Only suggest |
| off | Disabled |

---

### 16.3 Set to warn

It is recommended to use warn for observation during experiments and initial production phases:

    ceph osd pool set rbd-demo pg_autoscale_mode warn

Check:

    ceph osd pool autoscale-status

---

### 16.4 Set target_size_ratio

If you expect a Pool to use 50% of the cluster capacity, you can set:

    ceph osd pool set rbd-demo target_size_ratio 0.5

Check:

    ceph osd pool autoscale-status

Note:

    target_size_ratio is not a quota.
    It is only a reference for the PG Autoscaler's capacity planning.

---

### 16.5 Is it recommended to enable on?

You can try on in experimental environments.

In production environments, be cautious:

    ceph osd pool set rbd-demo pg_autoscale_mode on

Risks:

    Auto-adjusting PG numbers may trigger data remap.
    Data remap may cause backfill.
    Backfill will consume network and OSD IO.

Recommendations:

    Use warn first in production.
    Decide whether to adjust based on suggestions and change windows.

---

## SeventeenI don't know.Experiment 9: Clean Test Pool

### 17.1 Delete RBD Image

If you created RBD Images earlier:

    rbd ls -p rbd-demo

Delete:

    rbd rm rbd-demo/demo-image

Confirm:

    rbd ls -p rbd-demo

---

### 17.2 Delete Objects

List objects:

    rados -p rbd-demo ls

Delete objects:

    rados -p rbd-demo rm object-demo

---

### 17.3 Enable Pool Deletion Switch

Check:

    ceph config get mon mon_allow_pool_delete

Temporarily enable in experimental environments:

    ceph config set mon mon_allow_pool_delete true

---

### 17.4 Delete Pool

    ceph osd pool rm rbd-demo rbd-demo --yes-i-really-really-mean-it

Check:

    ceph osd pool ls

---

### 17.5 Disable Pool Deletion Switch

    ceph config set mon mon_allow_pool_delete false

Confirm:

    ceph config get mon mon_allow_pool_delete

Production warning:

    Deleting a Pool is a high-risk operation.
    Deleting a Pool will delete all data in the Pool.
    In production environments, it must be confirmed with backups and change approval.
    After deletion, you must disable mon_allow_pool_delete.

---

## EighteenI don't know.Experiment 10: Common Pool/PG Alarm Troubleshooting

### 18.1 POOL_APP_NOT_ENABLED

Symptoms:

    ceph -s shows pool application not enabled

Cause:

    The Pool has not enabled the application type.

Check:

    ceph health detail

Resolution:

If the Pool is for RBD:

    ceph osd pool application enable <pool-name> rbd

If the Pool is for CephFS:

    ceph osd pool application enable <pool-name> cephfs

If the Pool is for RGW:

    ceph osd pool application enable <pool-name> rgw

Check:

    ceph osd pool application get <pool-name>

---

### 18.2 PG degraded

Symptoms:

    ceph -s shows degraded

Check:

    ceph health detail
    ceph pg stat
    ceph osd tree

Common causes:

- OSD down
- OSD out
- Insufficient replicas
- Insufficient OSD count
- CRUSH Rule restrictions
- Recovery is in progress

Resolution approach:

    First check if any OSD is down.
    Then check Pool size and min_size.
    Then check CRUSH failure domain.
    Then observe if recovery is ongoing.
    Do not directly repair or delete PG.

---

### 18.3 PG undersized

Meaning:

    The number of replicas in the current PG is less than the Pool's set size.

Troubleshoot:

    ceph health detail
    ceph pg stat
    ceph osd tree
    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> crush_rule

Common causes:

- Fewer OSDs than required replicas
- Insufficient failure domain count
- OSD down
- OSD out
- CRUSH Rule set too high, e.g., rack failure domain but insufficient rack count

---

### 18.4 PG stale

Meaning:

    The MON has not received status updates from the OSD responsible for the PG for a long time.

Troubleshoot:

    ceph health detail
    ceph osd tree
    ceph orch ps --daemon_type osd
    ceph pg <pg-id> query

Common causes:

- Node where the OSD resides is down
- Network issues
- OSD process is stuck
- Communication issues between MON and OSD

---

### 18.5 PG Inconsistent

Meaning:

    Inconsistency detected between replicas of objects within PG.

Check:

    ceph health detail

Further check:

    ceph pg <pg-id> query

Possible handling:

    ceph pg repair <pg-id>

High-risk warning:

    Understand the specific error before repair.
    Do not blindly repair in production environments.
    Need to judge based on scrub / deep scrub results.

---

### 18.6 PG Stuck

Check stuck PGs:

    ceph pg dump_stuck

Check inactive:

    ceph pg dump_stuck inactive

Check unclean:

    ceph pg dump_stuck unclean

Check stale:

    ceph pg dump_stuck stale

Handling approach:

    First check ceph health detail.
    Then check OSD status.
    Then check PG query.
    Then determine if it's an OSD, network, CRUSH, capacity, or replica count issue.

---

## NineteenI don't know.Pool and PG Common Commands Summary

### 19.1 Pool Check

    ceph osd pool ls
    ceph osd pool ls detail
    ceph osd lspools
    ceph df

---

### 19.2 Pool Parameters

    ceph osd pool get <pool-name> all
    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> min_size
    ceph osd pool get <pool-name> pg_num
    ceph osd pool get <pool-name> pgp_num
    ceph osd pool get <pool-name> crush_rule
    ceph osd pool get <pool-name> pg_autoscale_mode

---

### 19.3 Pool Creation and Deletion

    ceph osd pool create <pool-name> <pg-num>
    ceph osd pool application enable <pool-name> rbd
    ceph osd pool rm <pool-name> <pool-name> --yes-i-really-really-mean-it

---

### 19.4 Replica Count

    ceph osd pool set <pool-name> size 3
    ceph osd pool set <pool-name> min_size 2

---

### 19.5 PG Check

    ceph pg stat
    ceph pg dump pgs_brief
    ceph pg dump_stuck
    ceph pg map <pg-id>
    ceph pg <pg-id> query

---

### 19.6 PG Autoscaler

    ceph osd pool autoscale-status
    ceph osd pool set <pool-name> pg_autoscale_mode warn
    ceph osd pool set <pool-name> pg_autoscale_mode on
    ceph osd pool set <pool-name> pg_autoscale_mode off
    ceph osd pool set <pool-name> target_size_ratio 0.5

---

### 19.7 RADOS Test

    rados -p <pool-name> put <object-name> <local-file>
    rados -p <pool-name> ls
    rados -p <pool-name> get <object-name> <local-file>
    rados -p <pool-name> rm <object-name>

---

## TwentyI don't know.Production Environment Precautions

### 20.1 Do Not Arbitrarily Create Pools

Creating a Pool increases management complexity.

Before creating a Pool in production, ensure:

- Pool name
- Business ownership
- Application type
- Replica count
- min_size
- PG count
- CRUSH Rule
- Capacity planning
- Alert strategy
- Whether quota is needed
- Whether backup is needed

---

### 20.2 Do Not Arbitrarily Delete Pools

Deleting a Pool is a high-risk operation.

Production requirements:

    Confirm the business has been decommissioned before deletion.
    Confirm no PVC, RBD, CephFS, RGW is still in use before deletion.
    Complete data backup before deletion.
    Complete change approval before deletion.
    Disable mon_allow_pool_delete after deletion.

---

### 20.3 Do Not Only Look at Size

Misunderstanding:

    size=3 means it's definitely safe.

Correct understanding:

    size=3 means there are 3 replicas.
    Whether replicas are safe also depends on CRUSH failure domain.
    If all 3 replicas are on the same host, it's still dangerous if the host fails.

---

### 20.4 Do Not Blindly Adjust PG Count

Adjusting PG count may lead to:

- remap
- backfill
- recovery
- Increased OSD IO
- Increased network traffic
- Business performance degradation

In production, should:

    First check autoscaler recommendations.
    Then evaluate during business low-traffic periods.
    Then adjust step by step.
    Then continuously monitor ceph -s and ceph pg stat.

---

### 20.5 Capacity Water Level Must Be Addressed in Advance

Ceph cannot be used to 95% capacity and then handle it like ordinary disks.

Reasons:

    Ceph needs extra space for recovery and rebalancing.
    If the cluster is close to full, recovery/backfill may be blocked.
    Pool writes may fail.

Recommendations:

    Start paying attention at 70%.
    Prepare for expansion or cleanup at 80%.
    Above 85% enters high-risk zone.
    nearfull/full must be handled immediately.

---

## Twenty-oneI don't know.Advanced SRE Methodology

### 21.1 Pool Is the Boundary of Business and Policy

Advanced SREs look at Pools, not just names.

Need to check: /think

- Who uses this Pool  
- What type of data is stored  
- How many replicas  
- What is the min_size  
- Is the number of PGs reasonable  
- Does the CRUSH Rule align with the failure domain  
- Is there a capacity quota  
- Are there monitoring alerts  
- Are there backup and recovery strategies  

---

### 21.2 PG is the observation window for failures and recovery  

Many Ceph failures ultimately manifest in PG states.  

Examples:  
```
active+degraded  
undersized  
stale  
inconsistent  
remapped  
backfilling  
recovering  
```  

Understanding PG states helps determine:  
- Is data recovering?  
- Is there insufficient OSDs?  
- Is the replica count insufficient?  
- Is CRUSH unable to find a suitable location?  
- Is there a network or OSD anomaly?  
- Is recovery too slow or stuck?  

---

### 21.3 size / min_size is the balance between safety and availability  

size represents the desired replica count.  

min_size represents the minimum replica count allowed for I/O operations.  

If min_size is too low:  
- Business operations may continue writing during failures, but data risks increase.  

If min_size is too high:  
- Data is more conservative, but business operations may stop writing during failures.  

In production, do not blindly lower min_size to allow continued writes.  

---

## Twenty-two, Interview Answer Structure  

If asked in an interview:  
- "What are Pools and PGs in Ceph, and what are their purposes?"  

You can answer:  
- "A Pool is a logical storage pool in Ceph used to host specific business data, with configurations like replica count, min_size, PG count, CRUSH Rule, and application type. For example, RBD uses an rbd Pool, CephFS has metadata and data Pools, and RGW has multiple object storage-related Pools."  
- "PG is a Placement Group, acting as an intermediate mapping layer between objects and OSDs. Ceph does not directly map each object to an OSD but first maps objects to PGs, then uses CRUSH to map PGs to a group of OSDs. This reduces the complexity of scheduling a large number of objects directly and facilitates recovery and rebalancing."  
- "Pool's size indicates the replica count, e.g., size=3 means each object has 3 replicas; min_size indicates the minimum replica count allowed for I/O operations, commonly configured as size=3 and min_size=2."  
- "PG count should be planned based on OSD count, Pool count, replica count, and data scale. Too few PGs may cause uneven data distribution, while too many increases MON and OSD management overhead. Newer versions can reference pg_autoscaler recommendations."  
- "During operations, if HEALTH_WARN appears, typically use commands like ceph -s, ceph health detail, ceph pg stat, and ceph osd tree to check for degraded, undersized, stale, or inconsistent PG states, then analyze causes using OSDs and CRUSH rules."  

---

## Twenty-three, Summary of This Article  

This article primarily organizes core content about Ceph Pools and PGs:  

1. A Pool is a logical storage pool in Ceph, serving as the boundary for business data and policies.  
2. Pools can configure size, min_size, pg_num, CRUSH Rule, and application type.  
3. PG is an intermediate mapping layer between objects and OSDs.  
4. Ceph's data mapping relationship is Object → PG → OSD Set.  
5. size indicates the replica count, commonly configured as size=3 in production.  
6. min_size indicates the minimum replica count allowed for I/O operations, commonly configured as min_size=2.  
7. Too few PGs may cause uneven data distribution, while too many increases management overhead.  
8. PG Autoscaler can provide PG count recommendations, but auto-adjustment should not be blindly enabled in production.  
9. active+clean is the ideal state, while degraded, undersized, stale, and inconsistent PGs require attention.  
10. Pool deletion, PG adjustments, and replica count changes are high-risk operations.  
11. Advanced SREs should be able to infer data distribution, failure impact, and recovery progress from Pool and PG states.  

---

## Twenty-four, Reference Documents  

Ceph Pool operation documentation:  
    https://docs.ceph.com/en/latest/rados/operations/pools/  

Ceph Placement Groups documentation:  
    https://docs.ceph.com/en/latest/rados/operations/placement-groups/  

Ceph PG Autoscaler documentation:  
    https://docs.ceph.com/en/latest/rados/operations/placement-groups/#autoscaling-placement-groups  

Ceph Health Checks documentation:  
    https://docs.ceph.com/en/latest/rados/operations/health-checks/  

Ceph CRUSH Map documentation:  
    https://docs.ceph.com/en/latest/rados/operations/crush-map/  

Ceph RADOS official documentation:  
    https://docs.ceph.com/en/latest/rados/  

Ceph rados command documentation:  
    https://docs.ceph.com/en/latest/man/8/rados/