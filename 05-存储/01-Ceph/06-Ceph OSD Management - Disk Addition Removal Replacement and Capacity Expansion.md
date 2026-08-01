# Ceph OSD Management: Disk Addition, Removal, Replacement, and Capacity Expansion

Suggested Path: 05-Storage/01-Ceph/06-Ceph OSD Management: Disk Addition, Removal, Replacement, and Capacity Expansion.md

Tags: #Ceph #OSD #DiskManagement #CapacityExtension #FailureRecovery. #Backfill #Rebalance #SRE #DistributedStorage #AdvancedSre

---

## I. Document Overview

This is the sixth article in the Ceph Advanced SRE Storage module series, focusing on practical management methods for OSDs.

OSD is the core component in Ceph responsible for storing data. Previous articles have covered:

- Ceph basic concepts
- Ceph core architecture
- Differences between RBD / CephFS / RGW
- Ceph cluster deployment planning
- cephadm cluster initialization

From this article, we enter the core of Ceph operations.

This article covers:

- What is OSD
- Relationship between OSD and disks
- How to check OSD status
- How to check available disks
- How to add OSD
- How to observe data Rebalance
- How to safely remove OSD
- How to replace faulty disks
- How to expand OSD capacity
- How to troubleshoot OSD down
- How to handle disk unavailability
- How to identify high-risk operations
- How to perform experiment environment cleanup and rollback

This article emphasizes:

    OSD management is not simply executing add / rm commands.
    The core of OSD management is understanding data replication, data migration, recovery processes, and business impact.

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand the relationship between OSD and disks.
2. Use `ceph osd tree` to view OSD distribution.
3. Use `ceph orch device ls` to check available disks.
4. Manually add a specified disk as an OSD in Ceph.
5. Observe the Rebalance process after adding a new OSD.
6. Safely remove a specified OSD from the cluster.
7. Simulate the OSD disk replacement process.
8. Complete capacity expansion with new nodes or disks.
9. Identify OSD up/down, in/out statuses.
10. Troubleshoot issues like OSD down, OSD addition failure, disk unavailability, PG degraded, etc.
11. Clearly identify which OSD operations are high-risk in production.

---

## III. Experiment Environment

### 3.1 Node Planning

This article continues using the Ceph module experiment environment.

| IP | Hostname | Role | Disks |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD | /dev/sdb, /dev/sdc |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD | /dev/sdb, /dev/sdc |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD | /dev/sdb, /dev/sdc |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation | /dev/sdb, /dev/sdc, optional |

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 OSD Disk Planning

Basic experiment:

    Each node uses /dev/sdb as an OSD data disk

Recommended experiment:

    Each node uses /dev/sdb and /dev/sdc as OSD data disks

Example:

| Node | OSD Data Disks |
|---|---|
| ceph-node01 | /dev/sdb, /dev/sdc |
| ceph-node02 | /dev/sdb, /dev/sdc |
| ceph-node03 | /dev/sdb, /dev/sdc |

Important reminder:

    All commands involving /dev/sdb, /dev/sdc must be confirmed based on actual environment.
    If your virtual machine disk name is /dev/vdb, /dev/vdc, replace it with the actual device name.
    Do not perform wipefs, sgdisk, ceph orch daemon add osd operations on the system disk /dev/sda.

---

## IV. OSD Basic Concepts

### 4.1 What is OSD

OSD, full name:

    Object Storage Daemon

OSD is the daemon process in Ceph responsible for storing data.

Typically:

    A single data disk corresponds to one OSD

Example:

    /dev/sdb -> osd.0
    /dev/sdc -> osd.1

OSD is responsible for:

- Storing RADOS Objects
- Handling client read/write requests
- Participating in replica replication
- Reporting heartbeats
- Reporting capacity
- Participating in Recovery
- Participating in Backfill
- Participating in Rebalance
- Executing Scrub / Deep Scrub
- Reporting status to MON

You can understand it this way:

    MON is responsible for maintaining cluster status.
    MGR is responsible for management and monitoring.
    CRUSH is responsible for calculating data placement.
    OSD is responsible for actually storing data.

---

### 4.2 Relationship Between OSD and Disks

Recommended relationship:

    One independent disk = One OSD

Example:

| Node | Disk | OSD |
|---|---|---|
| ceph-node01 | /dev/sdb | osd.0 |
| ceph-node01 | /dev/sdc | osd.1 |
| ceph-node02 | /dev/sdb | osd.2 |
| ceph-node02 | /dev/sdc | osd.3 |
| ceph-node03 | /dev/sdb | osd.4 |
| ceph-node03 | /dev/sdc | osd.5 |

Not recommended:

    Mixing system disk with OSD data disk
    Placing multiple OSDs arbitrarily on the same system disk directory
    Directly adding existing business data disks to Ceph
    Executing wipefs / sgdisk without confirmation
    Batch removing OSDs without understanding data migration impact

---

## V. OSD Status Explanation

### 5.1 up / down

Indicates whether the OSD process is running.

| Status | Meaning |
|---|---|
| up | OSD process is online |
| down | OSD process is offline or unreachable |

---

### 5.2 in / out

Indicates whether the OSD participates in data distribution.

| Status | Meaning |
|---|---|
| in | OSD participates in CRUSH data distribution |
| out | OSD does not participate in new data distribution |

---

### 5.3 Common Combinations /think

| Status Combination | Meaning |
|---|---|
| up + in | Normal status |
| down + in | OSD failure, but still participating in data distribution, cluster may be degraded |
| up + out | OSD is online but not participating in data distribution, common during maintenance |
| down + out | OSD is offline and not participating in data distribution |

Normal daily status:

    up + in

Needs close attention:

    down + in

Because this typically indicates:

    OSD failure has affected replica integrity, may trigger PG degraded or recovery.

---

## Six. Pre-Operation Checklist

Before performing any OSD operation, conduct checks.

### 6.1 Check Cluster Overall Status

    ceph -s

Focus on:

- health
- osd
- pgs
- recovery io
- degraded objects
- misplaced objects

Under normal circumstances, it should be:

    HEALTH_OK

If it's already HEALTH_WARN, understand the alert cause first before proceeding with operations.

---

### 6.2 Check Health Details

    ceph health detail

If output is empty or no obvious anomalies, it indicates no detailed health alerts currently.

If there are alerts, record and analyze them first.

---

### 6.3 Check OSD Tree

    ceph osd tree

Focus on:

- Whether OSD is up
- Whether OSD is in
- Which hosts OSDs are distributed on
- Whether OSDs are concentrated on a single node
- Whether any OSDs are already down
- Whether OSD weight is abnormal

---

### 6.4 Check PG Status

    ceph pg stat

Ideal status:

    active+clean

If there are:

    degraded
    undersized
    remapped
    backfilling
    recovering

It indicates the cluster may be recovering or not fully healthy, and high-risk operations should be avoided.

---

### 6.5 Check Capacity

    ceph osd df
    ceph df

Focus on:

- Whether any OSD has excessively high usage
- Whether the cluster is near full
- Whether remaining space is sufficient for replica recovery
- Whether OSD usage is severely unbalanced

---

### 6.6 Check Devices

    ceph orch device ls --wide

Focus on:

- Whether AVAILABLE is Yes
- Whether REJECT REASONS is empty
- Whether disk paths match expectations
- Whether system disks were mistakenly identified as available devices

---

## Seven. Experiment Task List

| Experiment | Target | Risk Level |
|---|---|---|
| Experiment One | Check OSD status and disk mapping | Low |
| Experiment Two | Add /dev/sdb as OSD | Medium |
| Experiment Three | Add /dev/sdc to expand OSD | Medium |
| Experiment Four | Safely remove an OSD | High |
| Experiment Five | Simulate disk failure replacement | High |
| Experiment Six | Add ceph-node04 to expand cluster | Medium-High |
| Experiment Seven | Troubleshoot OSD down | Medium-High |

Notes:

    Operations involving wipefs, sgdisk, osd rm, osd purge are all high-risk.
    Production environments must follow change procedures.
    In experimental environments, it's recommended to create VM snapshots first.

---

## Eight. Experiment One: Check OSD Status and Disk Mapping

### 8.1 Check OSD Summary Status

    ceph osd stat

Expected example:

    3 osds: 3 up, 3 in

If each node has two OSD disks:

    6 osds: 6 up, 6 in

---

### 8.2 Check OSD Tree

    ceph osd tree

Expected example:

    ID  CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
    -1         3.00000  root default
    -3         1.00000      host ceph-node01
     0    hdd  1.00000          osd.0             up   1.00000  1.00000
    -5         1.00000      host ceph-node02
     1    hdd  1.00000          osd.1             up   1.00000  1.00000
    -7         1.00000      host ceph-node03
     2    hdd  1.00000          osd.2             up   1.00000  1.00000

Key confirmations:

    OSDs are distributed across different hosts.
    OSD status is up.
    OSDs are not concentrated on a single node.

---

### 8.3 Check OSD Services

    ceph orch ps --daemon_type osd

Expected example:

    NAME     HOST          STATUS         REFRESHED
    osd.0    ceph-node01   running
    osd.1    ceph-node02   running
    osd.2    ceph-node03   running

---

### 8.4 Check OSD Metadata

Check osd.0:

    ceph osd metadata osd.0

Key fields:

- hostname
- ceph_version
- osd_objectstore
- bluestore_bdev_dev_node
- devices

Used to confirm:

    Which node osd.0 is on, which block device it uses, and what version it runs.

---

### 8.5 Check OSD Capacity

    ceph osd df

Key fields:

| Field | Description |
|---|---|
| SIZE | Total OSD capacity |
| RAW USE | Raw used capacity |
| DATA | Data capacity |
| AVAIL | Available capacity |
| %USE | Usage rate |
| VAR | Deviation from average usage rate |
| PGS | Number of PGs |

---

### 8.6 Check OSD Latency

    ceph osd perf

Expected example:

osd  commit_latency(ms)  apply_latency(ms)
      0                  1                  1
      1                  1                  1
      2                  2                  2

If any OSD shows significantly higher latency than others, investigate:

- Disk performance
- Node IO
- Network latency
- Recovery / Backfill
- Host load
- Disk errors

---

## IX. Experiment 2: Adding /dev/sdb as an OSD

### 9.1 Pre-operation Verification

Check disks on all nodes:

    lsblk

Confirm:

    /dev/sdb exists
    /dev/sdb is not mounted
    /dev/sdb is not a system disk
    /dev/sdb has no business data

Check mounts:

    df -hT

Check signatures:

    wipefs -n /dev/sdb

Check LVM:

    pvs
    vgs
    lvs

---

### 9.2 Check Ceph-recognized Devices

Execute on ceph-node01 or any Ceph management node:

    ceph orch device ls --wide

Expected output:

    ceph-node01 /dev/sdb AVAILABLE Yes
    ceph-node02 /dev/sdb AVAILABLE Yes
    ceph-node03 /dev/sdb AVAILABLE Yes

If AVAILABLE is No, check REJECT REASONS.

---

### 9.3 Clean Data Disks

**High-risk warning:**

    The following commands will erase disk signatures and partition information.
    Only execute on confirmed data disks with no business data.
    Do not execute on /dev/sda.

Execute on ceph-node01:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

Execute on ceph-node02:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

Execute on ceph-node03:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

Check again after cleaning:

    ceph orch device ls --wide

---

### 9.4 Manually Add OSD

Execute on Ceph management node:

    ceph orch daemon add osd ceph-node01:/dev/sdb
    ceph orch daemon add osd ceph-node02:/dev/sdb
    ceph orch daemon add osd ceph-node03:/dev/sdb

Note:

    Manually specifying nodes and disks is safest.
    Not recommended for beginners to use --all-available-devices directly.

---

### 9.5 Verify OSD Addition Results

Check OSD services:

    ceph orch ps --daemon_type osd

Check OSD tree:

    ceph osd tree

Check OSD status:

    ceph osd stat

Check cluster status:

    ceph -s

Expected:

    3 osds: 3 up, 3 in

PG status should eventually be:

    active+clean

If just added, may temporarily show:

    creating
    peering
    active+clean

Wait using:

    watch -n 3 'ceph -s'

---

## X. Experiment 3: Adding /dev/sdc to Expand OSD

### 10.1 Experiment Objective

If each node has a second data disk:

    /dev/sdc

It can be added to Ceph to observe capacity expansion and data Rebalance.

---

### 10.2 Pre-operation Checks

Execute on all nodes:

    lsblk
    df -hT
    wipefs -n /dev/sdc

Execute on Ceph management node:

    ceph orch device ls --wide

Confirm:

    /dev/sdc AVAILABLE is Yes

---

### 10.3 Clean /dev/sdc

Execute on ceph-node01:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

Execute on ceph-node02:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

Execute on ceph-node03:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

---

### 10.4 Add OSD

    ceph orch daemon add osd ceph-node01:/dev/sdc
    ceph orch daemon add osd ceph-node02:/dev/sdc
    ceph orch daemon add osd ceph-node03:/dev/sdc

---

### 10.5 Observe Post-expansion Status

Check:

    ceph osd tree
    ceph osd stat
    ceph osd df
    ceph -s

Expected:

    6 osds: 6 up, 6 in

Monitor data balancing:

    watch -n 5 'ceph osd df'

Monitor cluster recovery:

    watch -n 5 'ceph -s'

May see:

    remapped
    backfilling
    recovering

Final goal:

    active+clean

---

## XI. Understanding Rebalance After Adding New OSD

### 11.1 Why Adding OSD Triggers Rebalance

After adding an OSD, the Ceph cluster capacity changes.

CRUSH will recalculate optimal PG locations.

Some data will migrate to the new OSD.

This is:

    Rebalance

---

### 11.2 Phenomena During Rebalance

Check:

    ceph -s

May see:

    pgs remapped
    pgs backfilling
    pgs recovering
    recovery io

Check capacity:

    ceph osd df

New OSD usage rate will rise gradually.

### 11.3 Resource Consumption of Rebalance

Rebalance consumes:

- Network bandwidth
- OSD disk IO
- CPU
- Memory
- Background recovery threads

Production recommendations:

    Do not add a large number of OSDs in batches during business peak hours.
    Continuously monitor business IO and Ceph recovery status after expansion.
    Balance recovery speed and business impact.

---

## Twelve, Experiment Four: Safely Removing an OSD

### 12.1 Experiment Objective

Simulate the safe removal of an OSD from the cluster.

Example target:

    osd.3

Note:

    Please select the target OSD based on the actual results of `ceph osd tree`.
    Do not directly copy `osd.3`.
    Do not execute arbitrarily in production environments.

---

### 12.2 Pre-Operation Checks

Execute:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph osd df
    ceph pg stat

Requirements:

    The cluster should be in HEALTH_OK state.
    PGs should be active+clean.
    Remaining OSD capacity should be sufficient to support replica recovery.
    Confirm that it is not removing multiple OSDs consecutively.

---

### 12.3 Confirm OSD Location

View:

    ceph osd tree

View metadata:

    ceph osd metadata osd.3

Confirm:

- The node where osd.3 is located
- The disk corresponding to osd.3
- Whether osd.3 is currently up/in

---

### 12.4 Mark OSD as out

    ceph osd out osd.3

View:

    ceph osd tree
    ceph -s

At this point, you may see:

    degraded
    recovering
    backfilling

Monitor continuously:

    watch -n 5 'ceph -s'

Wait for PG recovery:

    active+clean

---

### 12.5 Remove OSD

In a cephadm environment, prioritize using:

    ceph orch osd rm 3

View removal progress:

    ceph orch osd rm status

View OSD tree:

    ceph osd tree

View services:

    ceph orch ps --daemon_type osd

---

### 12.6 Post-Removal Verification

Execute:

    ceph -s
    ceph osd tree
    ceph osd stat
    ceph pg stat
    ceph health detail

Goals:

    Cluster recovers to stable state.
    PGs are active+clean.
    No unexplained HEALTH_ERR.
    The removed OSD no longer participates in data distribution.

---

## Thirteen, Experiment Five: Fault Disk Replacement Process

### 13.1 Scenario Description

Assume:

    osd.3 on ceph-node02 corresponds to /dev/sdc
    /dev/sdc disk failure
    Need to replace the disk with a new one

Goals:

    Safely remove the faulty OSD
    Replace the disk
    Add a new OSD
    Wait for the cluster to recover to active+clean

---

### 13.2 Confirm Faulty OSD

View cluster status:

    ceph -s

View health details:

    ceph health detail

View OSD tree:

    ceph osd tree

View OSD metadata:

    ceph osd metadata osd.3

Check disks on the faulty node:

    lsblk
    dmesg | grep -i error
    journalctl -k | grep -i error

View OSD services:

    ceph orch ps --daemon_type osd

---

### 13.3 Mark Faulty OSD as out

If osd.3 is still in:

    ceph osd out osd.3

Monitor recovery:

    watch -n 5 'ceph -s'

Wait for PGs to recover to:

    active+clean

If the failure is severe, it may not clean quickly, requiring risk assessment based on replica count and remaining OSDs.

---

### 13.4 Remove Faulty OSD

    ceph orch osd rm 3

View progress:

    ceph orch osd rm status

View:

    ceph osd tree
    ceph orch ps --daemon_type osd

---

### 13.5 Replace Disk

In a virtual machine environment:

    Shut down the virtual machine or hot-remove the faulty virtual disk
    Add a new virtual disk
    Start the virtual machine or rescan the disk

In physical servers:

    Replace the faulty disk according to hardware maintenance procedures
    Confirm the system recognizes the new disk

View the new disk:

    lsblk
    fdisk -l

---

### 13.6 Clean New Disk

Assume the new disk is still:

    /dev/sdc

Clean:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

Check:

    wipefs -n /dev/sdc
    lsblk

---

### 13.7 Add New OSD

    ceph orch daemon add osd ceph-node02:/dev/sdc

View:

    ceph orch ps --daemon_type osd
    ceph osd tree
    ceph -s

Monitor recovery:

    watch -n 5 'ceph -s'

Final goals:

    HEALTH_OK
    active+clean
    New OSD is up/in

---

## Fourteen, Experiment Six: Adding ceph-node04 to Expand Cluster

### 14.1 Experiment Objective

Add ceph-node04 to the cluster as a new OSD node, and observe the expansion and Rebalance process.

Prerequisites:

    ceph-node04 has installed the system
    ceph-node04 has configured the hostname
    ceph-node04 has configured /etc/hosts
    ceph-node04 has completed time synchronization
    ceph-node04 has an independent data disk /dev/sdb
    cephadm SSH public key has been distributed

---

### 14.2 Add ceph-node04 Host

    ceph orch host add ceph-node04 10.0.0.34

View:

    ceph orch host ls

Add label:

    ceph orch host label add ceph-node04 osd

---

### 14.3 View Devices /think

```
ceph orch device ls --wide

Confirmation:

    ceph-node04 /dev/sdb AVAILABLE Yes

---

### 14.4 Cleaning Disks

Execute on ceph-node04:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

If /dev/sdc exists:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

---

### 14.5 Adding OSD

    ceph orch daemon add osd ceph-node04:/dev/sdb

If /dev/sdc exists:

    ceph orch daemon add osd ceph-node04:/dev/sdc

---

### 14.6 Observing Expansion

Check:

    ceph osd tree
    ceph osd df
    ceph -s
    ceph pg stat

Monitor continuously:

    watch -n 5 'ceph -s'

Expected:

    After a new OSD joins, Ceph may briefly show backfilling / remapped states.
    Eventually, it will recover to active+clean.

---

## FifteenI don't know.Experiment Seven: Simulating OSD Down Troubleshooting

### 15.1 Experiment Description

This experiment is only recommended for test environments.

You can simulate stopping an OSD service and observe Ceph's response.

Example:

    osd.3

---

### 15.2 Finding OSD Services

Check the node where the OSD resides:

    ceph osd metadata osd.3

Check cephadm services:

    ceph orch ps --daemon_type osd

---

### 15.3 Stopping OSD Services

In a cephadm environment, it's not recommended to arbitrarily use systemctl on container services.

You can manage via ceph orch commands for observation or restart.

Check service names:

    ceph orch ps --daemon_type osd

If you want to restart:

    ceph orch daemon restart osd.3

For testing environments simulating down, temporarily stop related containers or services on the corresponding node, but commands vary by deployment method. Always confirm the service name first.

A safer experimental approach:

    Shut down a single test OSD node on ceph-node04
    Or temporarily disconnect the test disk in a virtualization platform

---

### 15.4 Observing Cluster Status

    ceph -s
    ceph health detail
    ceph osd tree
    ceph pg stat

You might see:

    osd down
    degraded
    undersized
    recovering

---

### 15.5 Recovering OSD

If the service is abnormal:

    ceph orch daemon restart osd.3

If the node recovers:

    Start the node
    Confirm network
    Confirm the disk
    Observe if the OSD automatically comes up

Check:

    ceph osd tree
    ceph -s

---

## SixteenI don't know.BackfillI don't know.Recovery and Rebalance

### 16.1 Recovery

Recovery typically indicates:

    Recovering missing replicas after a failure.

Scenarios:

- OSD down
- OSD out
- Disk failure
- Node failure

---

### 16.2 Backfill

Backfill typically indicates:

    An OSD needs to fill data after newly joining or rejoining.

Scenarios:

- Adding a new OSD
- Replacing an OSD
- An OSD rejoins after long offline
- CRUSH rule changes

---

### 16.3 Rebalance

Rebalance indicates:

    The cluster rebalances data based on new OSD distribution.

Scenarios:

- Adding an OSD
- Removing an OSD
- Adding a node
- Modifying OSD weights
- Modifying CRUSH rules

---

### 16.4 Common Impact

These processes consume:

- Network bandwidth
- Disk I/O
- CPU
- OSD background resources

In production environments, balance is needed:

    Faster recovery may impact business I/O more.
    Slower recovery means data remains in degraded state longer.

---

## SeventeenI don't know.OSD Daily Maintenance Commands

### 17.1 Status Commands

    ceph -s
    ceph health detail
    ceph osd stat
    ceph osd tree
    ceph pg stat

---

### 17.2 Capacity Commands

    ceph df
    ceph osd df

---

### 17.3 Performance Commands

    ceph osd perf

---

### 17.4 Metadata Commands

    ceph osd metadata osd.0

---

### 17.5 Orchestration Commands

    ceph orch host ls
    ceph orch device ls --wide
    ceph orch ps --daemon_type osd

---

### 17.6 Addition Commands

    ceph orch daemon add osd ceph-node01:/dev/sdb

---

### 17.7 Removal Commands

    ceph osd out osd.3
    ceph orch osd rm 3
    ceph orch osd rm status

---

## EighteenI don't know.Common Failures and Troubleshooting

### 18.1 OSD Down

Symptoms:

    ceph -s shows osd down

Troubleshooting:

    ceph osd tree
    ceph health detail
    ceph orch ps --daemon_type osd
    ceph osd metadata osd.X

Check on the corresponding node:

    hostname
    lsblk
    df -hT
    dmesg | grep -i error
    journalctl -k | grep -i error
    systemctl status ceph-*.target

Possible causes:

- OSD process anomaly
- Disk failure
- Node downtime
- Network interruption
- Container runtime anomaly
- BlueStore error

---

### 18.2 Failed OSD Addition

Symptoms:

    ceph orch daemon add osd ceph-node01:/dev/sdb fails

Troubleshooting: /think
```

ceph orch device ls --wide
lsblk
wipefs -n /dev/sdb
pvs
vgs
lvs
ceph health detail

Common Causes:

- Disk already has a filesystem
- Disk is mounted
- Disk has LVM metadata
- Device name is incorrect
- Node not added to ceph orch
- Disk is too small
- Container runtime anomaly

---

### 18.3 OSD Usage Imbalance

Check:

    ceph osd df

Possible Causes:

- OSD capacity differences
- OSD weights are unreasonable
- reweight has been modified
- New OSD just joined, Rebalance not completed
- PG count is unreasonable
- Some Pool data is concentrated

Continue Troubleshooting:

    ceph osd tree
    ceph osd df
    ceph pg stat
    ceph osd pool autoscale-status

---

### 18.4 High OSD Latency

Check:

    ceph osd perf

Troubleshoot on corresponding node:

    iostat -x 1
    top
    vmstat 1
    dmesg | grep -i error
    journalctl -k | grep -i error

Possible Causes:

- Slow disk
- Disk failure
- Node IO is saturated
- OSD is recovering
- Network latency
- Host resources are insufficient

---

### 18.5 PG degraded

Check:

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree

Common Causes:

- OSD down
- OSD out
- Insufficient replicas
- Insufficient OSD count
- CRUSH rule restrictions
- Recovery is in progress

Handling Approach:

    First confirm OSD status.
    Then confirm replica count and failure domain.
    Then confirm if there are recovery tasks.
    Do not directly delete OSD or execute repair.

---

## NineteenI don't know.Cleanup and Rollback Notes

### 19.1 Revoke After Adding OSD

If you just added a test OSD and there is no important business data, you can safely remove it following the procedure:

    ceph osd out osd.X
    watch -n 5 'ceph -s'
    ceph orch osd rm X
    ceph orch osd rm status

Wait for completion before cleaning the disk.

---

### 19.2 Clean Disk

After confirming the disk no longer belongs to an active OSD, execute on the corresponding node:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

High-Risk Warning:

    Only for experimental environments or disks confirmed to have no business data.
    Do not execute on system disks.
    Production environments must go through change approval.

---

### 19.3 Cleanest Rollback Method for Experimental Environments

For virtual machine experimental environments, the cleanest method is:

    Rollback virtual machine snapshot
    Or delete virtual machine and rebuild
    Or re-mount a new empty data disk

This is more reliable than manual cleanup of residual data.

---

## TwentyI don't know.Production Environment Notes

1. Do not batch out multiple OSDs.
2. Do not batch delete multiple OSDs.
3. Do not perform high-risk OSD operations when the cluster is unhealthy.
4. Do not execute wipefs or sgdisk on system disks.
5. Do not blindly use --all-available-devices to take over all disks.
6. Must confirm remaining capacity is sufficient before removing OSDs.
7. Must confirm replica count and failure domain safety before removing OSDs.
8. OSD replacement must first confirm the corresponding node and disk for the faulty OSD.
9. New OSDs trigger Rebalance, need to monitor business IO.
10. Recovery / Backfill may affect business performance.
11. Production environments should perform expansion, removal, and replacement operations during business low-peak hours.
12. OSD operations should be included in monitoring, alerts, change, and rollback processes.
13. Before handling any OSD failures, first execute ceph -s and ceph health detail.
14. Do not blindly adjust recovery/backfill parameters for quick recovery.
15. OSD monitoring must cover up/down, in/out, capacity, latency, PG status, and disk errors.

---

## Twenty-OneI don't know.Advanced SRE Methodology

### 21.1 OSD Management Core is Not Commands, but Data Migration

Basic Understanding:

    Add OSD
    Remove OSD
    Replace OSD

Advanced SRE Understanding:

    Adding an OSD triggers data redistribution.
    Removing an OSD triggers replica migration.
    Replacing an OSD triggers recovery.
    Recovery consumes network and disk resources.
    Recovery speed and business performance need to be balanced.

---

### 21.2 Always Check Cluster Health Before OSD Operations

Standard Pre-Commands:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph osd df
    ceph pg stat

If the cluster is already degraded, do not proceed with unrelated high-risk changes.

---

### 21.3 OSD Failures Should Not Be Viewed in Isolation

OSD down could be:

- Disk failure
- Node failure
- Network failure
- Container runtime failure
- BlueStore failure
- Permission issues
- Cephadm orchestration issues

Troubleshooting should converge layer by layer from cluster status, OSD status, node status, disk status, and log status.

---

## Twenty-TwoI don't know.Interview Answer Approach

If asked in an interview:

    How to handle a Ceph OSD disk failure?

You can answer:

    I would first use ceph -s, ceph health detail, ceph osd tree, and ceph osd metadata to confirm which OSD failed, its node, and corresponding disk.
    If confirmed as disk failure, I would not directly delete data or remove the disk. Instead, I would first assess cluster status, Pool replica count, remaining capacity, and PG status.
    The standard process is to mark the faulty OSD as out, allowing Ceph to recover the data replicas from other OSDs. Monitor ceph -s and wait for PGs to return to active+clean.
    Then use ceph orch osd rm to remove the OSD, and check ceph orch osd rm status to confirm removal completion.
    Afterward, replace the disk, confirm the new disk has no business data, clean the disk signature with wipefs and sgdisk, then add the new OSD using ceph orch daemon add osd node:/dev/sdX.
    After adding, continue monitoring ceph -s, ceph osd tree, ceph osd df, and PG status until the cluster returns to normal.
    In production environments, do not batch remove OSDs, and avoid making high-risk changes when the cluster is already unhealthy.

---

## Twenty-ThreeI don't know.Summary of This Section

# Ceph OSD Management Methods

1. OSD is the core component in Ceph that actually stores data.
2. It is recommended to use a dedicated disk for each OSD.
3. OSD status requires attention to both up/down and in/out.
4. Before adding an OSD, ensure the disk is not a system disk or a business disk.
5. In experiments, it is recommended to manually specify ceph-node:/dev/sdX to add an OSD.
6. Adding an OSD may trigger a Rebalance.
7. When removing an OSD, first mark it as out, wait for data migration to complete, then remove it.
8. Replacing a faulty disk should follow the standard procedure and cannot be done by directly and roughly clearing the disk.
9. Recovery, Backfill, and Rebalance all consume network and disk resources.
10. In production environments, OSD operations must be combined with capacity, failure domain, replica count, and monitoring evaluation.
11. For advanced SREs managing OSDs, the focus is not on memorizing commands but understanding the data migration and replica recovery process.

---

## 24. Reference Documents

Ceph OSD Management:

    https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/

Cephadm OSD Service:

    https://docs.ceph.com/en/latest/cephadm/services/osd/

Ceph Orchestrator:

    https://docs.ceph.com/en/latest/mgr/orchestrator/

Ceph Health Checks:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Placement Groups:

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph CRUSH Map:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph Troubleshooting OSDs:

    https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-osd/