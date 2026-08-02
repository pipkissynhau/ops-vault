# Ceph OSD Management: Adding, Removing, Replacing Disks, and Capacity Expansion

Recommended Path: 05-Storage/01-Ceph/06-Ceph OSD Management: Adding, Removing, Replacing Disks, and Capacity Expansion.md

Tags: #Ceph #OSD #Disk Management #Capacity Expansion #Fault Recovery #Backfill #Rebalance #SRE #Distributed Storage #Advanced SRE

---

## I. Document Overview

This article is the sixth in the Ceph Advanced SRE storage module series, focusing on practical management methods for OSDs.

OSDs are the core components in Ceph that actually store data. Previously, we have covered:

- Basic concepts of Ceph
- Core architecture of Ceph
- Differences between RBD, CephFS, and RGW
- Cluster deployment planning for Ceph
- Initializing a cephadm cluster

Starting from this article, we delve into the core areas of Ceph operations and maintenance.

This document covers:

- What OSDs are
- The relationship between OSDs and disks
- How to check the status of OSDs
- How to view available disks
- How to add an OSD
- Observing the data rebalancing process
- Safely removing an OSD
- Replacing a failed disk
- Expanding the capacity of OSDs
- Troubleshooting OSD down events
- Dealing with unavailable disks
- Identifying high-risk operations
- Cleaning up and rolling back experimental environments

It is important to note that:

    Managing OSDs is not simply about executing `add`/`rm` commands.
    The essence of OSD management lies in understanding data replication, data migration, recovery processes, and their impact on the business.

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the correspondence between OSDs and disks.
2. Use `ceph osd tree` to view OSD distribution.
3. Use `ceph orch device ls` to check available disks.
4. Manually add a specified disk as an OSD in Ceph.
5. Observe the rebalancing process after adding a new OSD.
6. Safely remove a specified OSD from the cluster.
7. Simulate the disk replacement process for OSDs.
8. Expand the capacity by adding new nodes or disks.
9. Identify the up/down, in/out status of OSDs.
10. Troubleshoot issues such as OSD down, failed OSD addition, unavailable disks, and PG degradation.
11. Recognize which OSD operations are considered high-risk in a production environment.

---

## III. Experimental Environment

### 3.1 Node Planning

This article uses the same experimental environment as the Ceph module series.

| IP | Host Name | Role | Disks |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD | /dev/sdb, /dev/sdc |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD | /dev/sdb, /dev/sdc |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD | /dev/sdb, /dev/sdc |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Experiment | /dev/sdb, /dev/sdc (optional) |

Primary experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 OSD Disk Planning

Basic experiment:

    Each node uses `/dev/sdb` as the OSD data disk.

Recommended experiment:

    Each node uses both `/dev/sdb` and `/dev/sdc` as OSD data disks.

Example:

| Node | OSD Data Disks |
|---|---|
| ceph-node01 | /dev/sdb, /dev/sdc |
| ceph-node02 | /dev/sdb, /dev/sdc |
| ceph-node03 | /dev/sdb, /dev/sdc |

Important reminder:

    All commands involving `/dev/sdb` and `/dev/sdc` must be confirmed based on your actual environment.
    If your virtual machine uses different disk names, replace them accordingly.
    Do not perform operations such as `wipefs`, `sgdisk`, or `ceph orch daemon add osd` on the system disk `/dev/sda`.

---

## IV. Basic Concepts of OSDs

### 4.1 What are OSDs

OSD stands for:

    Object Storage Daemon

OSDs are the background processes in Ceph that actually store data.

Generally, each independent data disk corresponds to one OSD.

For example:

    /dev/sdb -> osd.0
    /dev/sdc -> osd.    ID  CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
    -1         3.00000  root default
    -3         1.00000      host ceph-node01
     0    hdd  1.00000          osd.0             up   1.00000  1.00000
    -5         1.00000      host ceph-node02
     1    hdd  1.00000          osd.1             up   1.00000  1.00000
    -7         1.00000      host ceph-node03
     2    hdd  1.00000          osd.2             up   1.00000  1.00000

Key points to confirm:

- OSDs are distributed across different hosts.
- All OSDs are in the "up" status.
- No OSDs are concentrated on a single node.

---

### 8.3 Checking OSD Services

Use the command:

    ceph orch ps --daemon_type osd

Expected output example:

    NAME     HOST          STATUS         REFRESHED
    osd.0    ceph-node01   running
    osd.1    ceph-node02   running
    osd.2    ceph-node03   running

---

### 8.4 Checking OSD Metadata

To check osd.0, use:

    ceph osd metadata osd.0

Key fields to observe:

- hostname
- ceph_version
- osd_objectstore
- bluestore_bdev_dev_node
- devices

These fields help confirm:

- On which node osd.0 is located.
- Which block device it uses.
- And which version of Ceph it is running on.

---

### 8.5 Checking OSD Capacity

Use the command:

    ceph osd df

Key fields:

| Field         | Description                |
|---------------|----------------------------------------|
| SIZE          | Total capacity of the OSD         |
| RAW USE        | Used raw capacity                   |
| DATA          | Data capacity                         |
| AVAIL         | Available capacity                      |
| %USE          | Utilization rate                        |
| VAR           | Deviation from the average utilization rate |
| PGS            | Number of PGs                           |

---

### 8.6 Checking OSD Latency

Use the command:

    ceph osd perf

Expected example output:

    osd  commit_latency(ms)  apply_latency(ms)
      0                  1                  1
      1                  1                  1
      2                  2                  2

If an OSD shows significantly higher latency than others, investigate possible issues such as:

- Disk performance
- Node I/O operations
- Network delays
- Recovery or backfilling processes
- Hostload
- Disk errors

---

## Section Nine: Experiment Two: Adding /dev/sdb as an OSD

### 9.1 Pre-operation Verification

Check the disks on all nodes using:

    lsblk

Confirm that:

- /dev/sdb exists.
- It is not currently mounted or used for system operations.
- It contains no important data.

Also, check the current disk usage and mount status:

    df -hT
    df -hT | grep /dev/sdb

Verify the disk signature to ensure it can be used in Ceph:

    wipefs -n /dev/sdb

Check for any LVM configurations related to /dev/sdb:

    pvs
    vgs
    lvs

---

### 9.2 Checking Devices Recognized by Ceph

Execute the following command on ceph-node01 or any other Ceph management node:

    ceph orch device ls --wide

Expected output:

    ceph-node01 /dev/sdb AVAILABLE Yes
    ceph-node02 /dev/sdb AVAILABLE Yes
    ceph-node03 /dev/sdb AVAILABLE Yes

If "AVAILABLE" is listed as "No", check the reasons for rejection.

---

### 9.3 Clearing the Data Disk

High-risk warning:

- The following commands will erase the disk signature and partition information.
- Only use these commands on disks that do not contain any important data.
- Never apply them to /dev/sda.

On ceph-node01, execute:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

On ceph-node02 and ceph-node03, perform the same operations.

After clearing, recheck the devices using:

    ceph orch device ls --wide

---

###    ceph osd df
    ceph pg stat

Requirements:

The current cluster should preferably be in the HEALTH_OK state.
The PG should be active+clean.
There should be sufficient remaining OSD capacity to support replica recovery.
It must be confirmed that multiple OSDs are not being removed consecutively.

---

### 12.3 Confirming the Location of the OSD

View:

ceph osd tree

View metadata:

ceph osd metadata osd.3

Confirm:

- The node where osd.3 is located
- The disk corresponding to osd.3
- Whether osd.3 is currently up/in

---

### 12.4 Marking the OSD as Out

ceph osd out osd.3

View:

ceph osd tree
ceph -s

At this point, you may see:

degraded
recovering
backfilling

Continue to monitor:

watch -n 5 'ceph -s'

Wait for the PG to return to active+clean status:

active+clean

---

### 12.5 Removing the OSD

In a cephadm environment, prefer using:

ceph orch osd rm 3

Check the removal progress:

ceph orch osd rm status

View the OSD tree:

ceph osd tree

View services:

ceph orch ps --daemon_type osd

---

### 12.6 Verification After Removal is Complete

Execute:

ceph -s
ceph osd tree
ceph osd stat
ceph pg stat
ceph health detail

Goals:

- The cluster should return to stability.
- The PG should be active+clean.
- There should be no unexplained HEALTH_ERR messages.
- The removed OSD should no longer participate in data distribution.

---

## Chapter Thirteen: Experiment Five: Process for Replacing a Faulty Disk

### 13.1 Scenario Description

Assume:

- osd.3 on ceph-node02 corresponds to /dev/sdc
- The /dev/sdc disk is faulty and needs to be replaced with a new one

Goals:

- Safely remove the faulty OSD
- Replace the disk
- Add the new OSD
- Wait for the cluster to return to active+clean status

---

### 13.2 Confirming the Faulty OSD

Check the cluster status:

ceph -s

View health details:

ceph health detail

View the OSD tree:

ceph osd tree

View OSD metadata:

ceph osd metadata osd.3

On the faulty node, check the disk:

lsblk
dmesg | grep -i error
journalctl -k | grep -i error

Check the OSD service:

ceph orch ps --daemon_type osd

---

### 13.3 Marking the Faulty OSD as Out

If osd.3 is still in status, mark it as out:

ceph osd out osd.3

Monitor the recovery process:

watch -n 5 'ceph -s'

Wait for the PG to return to active+clean status.

If the fault is severe and cleaning may take a long time, assess the risk based on the number of replicas and remaining OSDs.

---

### 13.4 Removing the Faulty OSD

ceph orch osd rm 3

Check the progress:

ceph orch osd rm status

View:

ceph osd tree
ceph orch ps --daemon_type osd

---

### 13.5 Replacing the Disk

In a virtual machine environment:

- Shut down the virtual machine or hot remove the faulty virtual disk.
- Add the new virtual disk.
- Restart the virtual machine or rescan the disks.

On a physical server:

- Follow the hardware maintenance procedures to replace the faulty disk.
- Ensure the system recognizes the new disk.

Check the new disk:

lsblk
fdisk -l

---

### 13.6 Cleaning the New Disk

Assume the new disk is still /dev/sdc:

Clean it:

wipefs -a /dev/sdc
sgdisk --zap-all /dev/sdc

Verify:

wipefs -n /dev/sdc
lsblk

---

### 13.7 Adding the New OSD

ceph orch daemon add osd ceph-node02:/dev/sdb

View:

ceph orch ps --daemon_type osd
ceph osd tree
ceph -s

Monitor the recovery process:

watch -n 5 'ceph -s'

Final goals:

- The cluster should return to HEALTH_OK status.
- The PG should be active+clean.
- The new OSD should be up/in status.

---

## Chapter Fourteen: Experiment Six: Adding ceph-node04 to Expand the Cluster

### 14.1 Experimental Objectives

Add ceph-node04 as a new OSD node to the cluster and observe the expansion and rebalancing process.

Prerequisites:

- Ceph-node04 has already had its system installed.
- The hostname of ceph-node04 has been configured.
### 17.2 Capacity-related Commands

    ceph df
    ceph osd df

---

### 17.3 Performance-related Commands

    ceph osd perf

---

### 17.4 Metadata-related Commands

    ceph osd metadata osd.0

---

### 17.5 Orchestration-related Commands

    ceph orch host ls
    ceph orch device ls --wide
    ceph orch ps --daemon_type osd

---

### 17.6 Adding OSDs

    ceph orch daemon add osd ceph-node01:/dev/sdb

---

### 17.7 Removing OSDs

    ceph osd out osd.3
    ceph orch osd rm 3
    ceph orch osd rm status

---

## Chapter Eighteen: Common Faults and Troubleshooting

### 18.1 OSD Down

**Symptoms:**

    `ceph -s` shows the OSD is down.

**Troubleshooting Steps:**

    Check:
    `ceph osd tree`
    `ceph health detail`
    `ceph orch ps --daemon_type osd`
    `ceph osd metadata osd.X`

    Verify on the corresponding node:
    `hostname`
    `lsblk`
    `df -hT`
    `dmesg | grep -i error`
    `journalctl -k | grep -i error`
    `systemctl status ceph-*.target`

**Possible Causes:**

- Abnormal OSD process
- Disk failure
- Node crash
- Network interruption
- Container runtime issues
- BlueStore errors

---

### 18.2 Failed OSD Addition

**Symptoms:**

    Attempt to add an OSD with `ceph orch daemon add osd ceph-node01:/dev/sdb` fails.

**Troubleshooting Steps:**

    Check:
    `ceph orch device ls --wide`
    `lsblk`
    `wipefs -n /dev/sdb`
    `pvs`
    `vgs`
    `lvs`
    `ceph health detail`

**Common Causes:**

- The disk already has a file system.
- The disk is already mounted.
- The disk contains LVM metadata.
- Incorrect device name.
- Node not yet added to the Ceph orchestration.
- Disk too small.
- Container runtime issues.

---

### 18.3 Unbalanced OSD Utilization

**Check:**

    `ceph osd df`

**Possible Causes:**

- Large differences in OSD capacities.
- Improper OSD weights.
- Modified reweight settings.
- Newly added OSDs with incomplete rebalancing.
- Inappropriate number of PGs.
- Data concentration in certain pools.

**Further Troubleshooting Steps:**

    Check:
    `ceph osd tree`
    `ceph osd df`
    `ceph pg stat`
    `ceph osd pool autoscale-status`

---

### 18.4 High OSD Latency

**Check:**

    `ceph osd perf`

**Troubleshoot on the corresponding node:**

    `iostat -x 1`
    `top`
    `vmstat 1`
    `dmesg | grep -i error`
    `journalctl -k | grep -i error`

**Possible Causes:**

- Slow disk.
- Disk failure.
- High node I/O load.
- OSD recovery in progress.
- Network latency.
- Insufficient host resources.

---

### 18.5 PG Degradation

**Check:**

    `ceph -s`
    `ceph health detail`
    `ceph pg stat`
    `ceph osd tree`

**Common Causes:**

- OSD down.
- OSD removed.
- Insufficient replicas.
- Inadequate number of OSDs.
- CRUSH rule constraints.
- Recovery process in progress.

**Action Plan:**

- First, verify the status of the affected OSD.
- Check the number of replicas and fault domains.
- Confirm if there are any recovery tasks pending.
- Avoid directly deleting or repairing the OSD.

---

## Chapter Nineteen: Cleaning Up and Rolling Back

### 19.1 Wanting to Remove an Added OSD

If you just added a test OSD and it doesn’t contain any critical business data, you can follow these steps:

    `ceph osd out osd.X`
    `watch -n 5 'ceph -s'
    ceph orch osd rm X`
    `ceph orch osd rm status`

**Clean up the disk afterward.**

---

### 19.2 Cleaning Up a Disk

After confirming that the disk is no longer used by any active OSD, perform the following on the corresponding node:

    `wipefs -a /dev/sdb`
    `sgdisk --zap-all /dev/sdb`

**High-risk Warning:**

- This procedure is only applicable to experimental environments orhttps://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Placement Groups:

https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph CRUSH Map:

https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph Troubleshooting OSDs:

https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-osd/