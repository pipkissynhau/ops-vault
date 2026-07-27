# Ceph RBD Block Storage: Image, Snapshot, Clone, and Common Operations

Recommended Path: 05-Storage/01-Ceph/09-Ceph RBD Block Storage: Image, Snapshot, Clone, and Common Operations.md

Tags: #Ceph #RBD #Block Storage #Image #Snapshot #Clone #Linux Mounting #Kubernetes #CSI #SRE #Distributed Storage #Advanced SRE

---

## I. Document Overview

This article is the ninth in the Ceph Advanced SRE storage series, focusing on the theory, practical operations, and troubleshooting methods of Ceph RBD block storage.

Previously covered topics include:

- Ceph Architecture
- RADOS, MON, MGR, OSD, CRUSH
- Differences Between RBD, CephFS, and RGW Storage Types
- cephadm Cluster Initialization
- OSD Management
- Pools and PGs
- CRUSH Fault Domains and Data Placement

This article delves into the first core usage of Ceph:

    RBD Block Storage

RBD stands for:

    RADOS Block Device

It can be understood as:

    Ceph's distributed cloud disk capability.

RBD is commonly used for:

- Virtual machine cloud disks
- OpenStack Cinder backends
- KubeVirt virtual machine disks
- Kubernetes RBD CSI PVCs
- Database data disks
- Stateful application data disks
- Snapshot and cloning scenarios

This article covers the following key areas:

- What RBD is
- What an RBD Image is
- How to create an RBD Pool
- How to create an RBD Image
- How to map an RBD on a Linux client
- How to format and mount an RBD
- How to expand an RBD Image
- How to create a Snapshot
- How to roll back from a Snapshot
- How to create a Clone based on a Snapshot
- How to flatten a Clone
- How to check the status and locks of an RBD
- How to safely uninstall and delete an RBD
- Common troubleshooting
- Best practices for production environments

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand the block storage model of RBD.
2. Create a dedicated RBD Pool.
3. Set the Pool's size, min_size, and application settings.
4. Create an RBD Image.
5. View information about the RBD Image.
6. Map the RBD Image on a Linux client.
7. Format the RBD device as XFS or ext4.
8. Mount the RBD to a local directory.
9. Verify basic read and write operations.
10. Expand the RBD Image.
11. Expand the file system online.
12. Create an RBD Snapshot.
13. Roll back data from a Snapshot.
14. Create a Clone based on a Snapshot.
15. Understand the relationship between protected snapshots and clones.
16. Flatten a clone to remove dependencies.
17. Safely uninstall, unmap, and delete an RBD Image.
18. Troubleshoot issues such as failed RBD mapping, incompatible features, occupied devices, and failed snapshot deletions.
19. Understand the relationship between RBD and Kubernetes RBD CSI.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article uses the same experimental environment as the Ceph module series.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Testing (optional) |
| 10.0.0.35 | ceph-client | RBD Client Testing (optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 Client Node Description

It is recommended to prepare a dedicated client node for this experiment:

    ceph-client
    10.0.0.35

Purpose:

- Install ceph-common
- Retrieve ceph.conf and keyring
- Map RBD Images
- Format RBD block devices
- Conduct mount tests
- Perform snapshot, rollback, and expansion experiments

If you don't have a ceph-client node yet, you can also perform basic experiments on ceph-node01.

However, in a production environment, it is better to separate the Ceph cluster nodes from the business client nodes.

---

## IV. What is RBDNo nearfull/full status observed.
There are no significant abnormalities in the OSD usage rates.

---

### 7.4 Checking if ceph-common is Installed on the Client

Execute the following commands on the ceph-client:

    ceph --version
    rbd --version

If these commands are not available, you need to install ceph-common.

For Ubuntu:

    apt update
    apt install -y ceph-common xfsprogs

For Rocky Linux 9:

    dnf install -y ceph-common xfsprogs

---

## VIII. Experimental Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Create an RBD Pool | Medium |
| Experiment 2 | Create an RBD Image | Low |
| Experiment 3 | Map an RBD Image on a Linux Client | Medium |
| Experiment 4 | Format and Mount an RBD Image | Medium to High |
| Experiment 5 | Write and Read Test Data | Low |
| Experiment 6 | Expand an RBD Image and File System | Medium |
| Experiment 7 | Create and View Snapshots | Low |
| Experiment 8 | Roll Back a Snapshot | High |
| Experiment 9 | Create a Clone Based on a Snapshot | Medium |
| Experiment 10 | Flatten a Clone | Medium |
| Experiment 11 | Safely Uninstall, Unmap, and Delete Resources | Medium to High |

High-Risk Reminder:

    mkfs will format the block device.
    rollback will revert data to its previous state.
    rm image will delete the RBD Image.
    snap purge will remove all snapshots.
    In a production environment, ensure that services are shut down, data is backed up, and a proper change window is scheduled.

---

## IX. Experiment 1: Create an RBD Pool

### 9.1 Create the Pool

Execute the following command on the Ceph management node:

    ceph osd pool create rbd-pool 64

Explanation:

    rbd-pool is the RBD Pool used in this experiment.
    64 represents the number of PGs; a smaller value of 32 or 64 can be used for small-scale experiments.

To check the creation, execute:

    ceph osd pool ls

---

### 9.2 Enable the RBD Application Type

    ceph osd pool application enable rbd-pool rbd

To verify the setting, execute:

    ceph osd pool application get rbd-pool

Expected output:

    {
        "rbd": {}
    }

---

### 9.3 Set the Number of Replicas

Set the number of replicas:

    ceph osd pool set rbd-pool size 3

Set the minimum number of replicas:

    ceph osd pool set rbd-pool min_size 2

To check the settings, execute:

    ceph osd pool get rbd-pool size
    ceph osd pool get rbd-pool min_size

Expected output:

    size: 3
    min_size: 2

---

### 9.4 Initialize the RBD Pool

Execute the following command:

    rbd pool init rbd-pool

Explanation:

    rbd pool init initializes the relevant metadata for the RBD Pool.
    In newer versions, some of these operations may be automated, but explicit execution ensures clarity.

---

### 9.5 Check the Pool Status

Execute the following commands:

    ceph df
    ceph osd pool get rbd-pool all
    ceph -s

Expected output:

    The cluster is healthy.
    The rbd-pool exists.
    The application type for the pool is set to rbd.

---

## X. Experiment 2: Create an RBD Image

### 10.1 Create a 10GB RBD Image

    rbd create demo-image --size 10G -p rbd-pool

To check the creation, execute:

    rbd ls -p rbd-pool

Expected output:

    demo-image

---

### 10.2 View Image Information

    rbd info rbd-pool/demo-image

Expected output similar to:

    rbd image 'demo-image':
            size 10 GiB in 2560 objects
            order 22
            snapshot_count: 0
            id: xxx
            block_name_prefix: rbd_data.xxx
            format: 2
            features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
            op_features:
            flags:
            create_timestamp: ...

Key fields:

| Field | Description |
|---|---|
| size | Image size |
| objects | Number of underlying objects |
| order | Parameter related to object size |
| format | RBD format, usually 2 |
| features | Enabled RBD features |
| snapshot_count |    rbd create demo-image-basic \
      --size 10G \
      -p rbd-pool \
      --image-feature layering

Then map:

    rbd map rbd-pool/demo-image-basic

Explanation:

    This is suitable for testing compatibility. In a production environment, it should be planned in conjunction with the kernel version, ceph-common version, and CSI version.

---

### 12.4 Viewing Block Devices

    lsblk

Expected output:

    rbd0  10G disk

You can also view it using:

    fdisk -l /dev/rbd0

---

## Section Thirteen: Experiment Four: Formatting and Mounting RBD

### 13.1 High-Risk Warning

The following command will format the block device:

    mkfs.xfs /dev/rbd0

Before executing, make sure that:

    /dev/rbd0 is a newly created test RBD.
    /dev/rbd0 does not contain any business data.
    It is not a system disk or any other business disk.

---

### 13.2 Formatting as XFS

    mkfs.xfs /dev/rbd0

If you want to use ext4:

    mkfs.ext4 /dev/rbd0

This example uses XFS.

---

### 13.3 Creating a Mount Directory

    mkdir -p /mnt/rbd-demo

Mount it:

    mount /dev/rbd0 /mnt/rbd-demo

Check it:

    df -hT | grep rbd

Expected output similar to:

    /dev/rbd0 xfs 10G ... /mnt/rbd-demo

---

### 13.4 Writing Test Data

    echo "hello ceph rbd" > /mnt/rbd-demo/hello.txt

Check it:

    cat /mnt/rbd-demo/hello.txt

Expected output:

    hello ceph rbd

To create a larger test file:

    dd if=/dev/zero of=/mnt/rbd-demo/test-100m.bin bs=1M count=100 oflag=direct

Check it:

    ls -lh /mnt/rbd-demo/

---

### 13.5 Checking Ceph Status

    ceph -s
    ceph df
    rbd du rbd-pool/demo-image

Explanation:

    rbd du can show the actual usage of the Image. If features like fast-diff are enabled, the statistics will be faster.

---

## Section Fourteen: Experiment Five: Unmounting and Unmapping RBD

### 14.1 Unmounting the File System

Execute this in the ceph-client:

    umount /mnt/rbd-demo

If it prompts "busy", check for occupied processes:

    lsof +f -- /mnt/rbd-demo

Or:

    fuser -vm /mnt/rbd-demo

Handle any occupying processes before unmounting.

---

### 14.2 Unmapping RBD

Check the mappings:

    rbd showmapped

Unmap it:

    rbd unmap /dev/rbd0

Check again:

    rbd showmapped

Expected output:

    demo-image should no longer be displayed.

---

### 14.3 Common Reasons for Unmapping Failures

Common reasons include:

- The file system is still mounted.
- There are processes occupying the directory.
- The current working directory is within the mount point.
- There is active I/O on the device.
- Issues with multipath or locking mechanisms.

Troubleshooting steps include:

    mount | grep rbd
    lsof +f -- /mnt/rbd-demo
    fuser -vm /mnt/rbd-demo
    rbd showmapped

---

## Section Fifteen: Experiment Six: Expanding RBD Images

### 15.1 Experiment Objective

Expand the RBD Image from 10G to 20G.

Prerequisites:

- The RBD has already been mapped to /dev/rbd0.
- The file system is either XFS or ext4.
- It is currently mounted at /mnt/rbd-demo.

If it has been unmapped, re-map and remount it first:

    rbd map rbd-pool/demo-image
    mount /dev/rbd0 /mnt/rbd-demo

---

### 15.2 Checking the Current Size

Check the RBD Image:

    rbd info rbd-pool/demo-image

Check the block device:

    lsblk

Check the file system:

    df -hT /mnt/rbd-demo

---

### 15.3 Expanding the RBD Image

    rbd resize rbd-pool/demo-image --size 20G

Check again:

    rbd info rbd-pool/demo-image

Expected output:

    size should now be 20 GiB.

---

### 15.4 Refreshing the Client Block DeviceExpected:

    demo-image
    demo-clone

To view clone information:

    rbd info rbd-pool/demo-clone

You can see the parent information, indicating that it depends on a snapshot of the parent Image.

---

### 18.3 Map Clone

    rbd map rbd-pool/demo-clone

To check:

    rbd showmapped

Assuming the clone is mapped to:

    /dev/rbd1

Create a mount directory:

    mkdir -p /mnt/rbd-clone

If there is already a file system in the clone, you can mount it directly:

    mount /dev/rbd1 /mnt/rbd-clone

To verify:

    ls -l /mnt/rbd-clone

---

### 18.4 The Value of Clone

Clones are suitable for:

- Quickly creating test environments
- Creating new disks based on template disks
- Quickly replicating virtual machine disks
- Creating database test copies
- CI testing data environments

Advantages:

    They can be created quickly.
    Initially, not all data is copied.
    Space is saved through Copy-on-Write technology.

---

## Chapter Nineteen: Experiment Ten: Flatten Clone

### 19.1 Why Flatten is Needed

By default, a clone depends on a snapshot of the parent Image.

Relationship:

    demo-clone
      |
      v
    demo-image@snap01

If you want the clone to exist independently, you need to flatten it.

After flattening:

    demo-clone will no longer depend on demo-image@snap01

---

### 19.2 Performing Flatten

First, unmount the clone:

    umount /mnt/rbd-clone

Unmap it:

    rbd unmap /dev/rbd1

Then perform flattening:

    rbd flatten rbd-pool/demo-clone

To check:

    rbd info rbd-pool/demo-clone

Expected:

    The parent information should disappear.

---

### 19.3 Canceling Protection and Deleting the Snapshot

After confirming that no clones depend on it, you can cancel the protection:

    rbd snap unprotect rbd-pool/demo-image@snap01

Delete the snapshot:

    rbd snap rm rbd-pool/demo-image@snap01

To verify:

    rbd snap ls rbd-pool/demo-image

---

## Chapter Twenty: Experiment Eleven: Cleaning Up RBD Resources

### 20.1 Unmounting Mount Points

    umount /mnt/rbd-demo
    umount /mnt/rbd-clone

If you receive a message indicating that the points do not exist or are not mounted, ignore it.

To check:

    mount | grep rbd

---

### 20.2 Unmapping RBD

To check:

    rbd showmapped

Unmap each one individually:

    rbd unmap /dev/rbd0
    rbd unmap /dev/rbd1

If the device numbers are different, adjust according to the output of rbd showmapped.

---

### 20.3 Deleting Clones

To check:

    rbd ls -p rbd-pool

Delete the clone:

    rbd rm rbd-pool/demo-clone

---

### 20.4 Deleting Snapshots

To view snapshots:

    rbd snap ls rbd-pool/demo-image

If a snapshot is still protected:

    rbd snap unprotect rbd-pool/demo-image@snap01

Delete the snapshot:

    rbd snap rm rbd-pool/demo-image@snap01

If you want to delete all snapshots:

    rbd snap purge rbd-pool/demo-image

High-risk warning:

    Snap purge will delete all snapshots of that Image.
    Do not perform this in a production environment.

---

### 20.5 Deleting the Original Image

    rbd rm rbd-pool/demo-image

To confirm:

    rbd ls -p rbd-pool

---

### 20.6 Deleting the Pool

Before deleting the pool, confirm:

    ceph osd pool ls
    rbd ls -p rbd-pool
    rados -p rbd-pool ls

Enable pool deletion:

    ceph config set mon mon_allow_pool_delete true

Delete the pool:

    ceph osd pool rm rbd-pool rbd-pool --yes-i-really-really-mean-it

Disenable pool deletion:

    ceph config set mon mon_allow_pool_delete false

Production reminder:

    Deleting a pool will remove all Images and objects within it.
    In a production environment, make sure to back up the data, obtain business approval, and go through change management procedures before proceeding.

---

## Chapter Twenty-One: Summary of Common RBD Commands

### 21.1 Pools

    ceph osd pool create rbd-pool 64
    ceph---  

### Re-execute rbd unmap.  

---

### 22.4 Failed to Delete Snapshot  

**Phenomenon:**  
The snapshot cannot be removed because it is protected.  

**Reason:**  
The snapshot is protected usually due to a clone dependency.  

**View:**  
`rbd children rbd-pool/demo-image@snap01`  

**Solution:**  
Delete or flatten the dependent clone, remove the protection, and then delete the snapshot.  

**Commands:**  
```bash
rbd flatten rbd-pool/demo-clone
rbd snap unprotect rbd-pool/demo-image@snap01
rbd snap rm rbd-pool/demo-image@snap01
```  

---  

### 22.5 Failed to Delete Image  

**Common Reasons:**  
- The image is still mapped.  
- The image has snapshots.  
- The image has clone dependencies.  
- The image is locked by a client.  

**Troubleshooting:**  
```bash
rbd status rbd-pool/demo-image
rbd snap ls rbd-pool/demo-image
rbd children rbd-pool/demo-image@snap01
rbd showmapped
```  

**Solution:**  
- Stop all services first.  
- Unmap the RBD volume.  
- Delete any snapshots or clones.  
- Finally, delete the image itself.  

---  

### 22.6 Poor RBD Performance  

**Possible Issues:**  
- OSD latency.  
- High OSD usage.  
- Abnormal PG status.  
- Network delays.  
- Client IO mode settings.  
- File system type.  
- RBD features not optimized.  
- Insufficient pool replicas.  
- Background recovery processes.  

**Diagnostic Commands:**  
```bash
ceph -s
ceph osd perf
ceph osd df
ceph pg stat
rbd perf image iostat
```
If `iostat` is available, use:  
`iostat -x 1`  

---  

## Chapter 23: The Relationship between RBD and Kubernetes  

In Kubernetes, RBD is typically used through Ceph CSI.  

**Typical Configuration:**  
```bash
StorageClass |
| -> PVC |
| -> Ceph RBD CSI |
| -> RBD Image |
| -> Pod Mount
```  

**RBD in Kubernetes:**  
RBD usually maps to the `ReadWriteOnce` storage class.  

**Suitable Use Cases:**  
- Single Pod with exclusive data access.  
- Databases.  
- Stateful services.  
- Application data directories.  

**Inappropriate Use:**  
- Multiple Pods sharing the same directory for read/write operations.  

For multi-Pod shared directories, consider using **CephFS CSI** instead.  

For more details, refer to:  
`12-Ceph and Kubernetes Integration: RBD CSI Dynamic Volume Provisioning Practice.md`  

---  

## Chapter 24: Precautions in a Production Environment  

### 24.1 Never Use the Admin Key for Business Clients  

Experimental use of the admin key is convenient, but in production, use a user with minimal permissions:  
`client.rbd-user`, and limit their access to specific pools:  
`osd 'profile rbd pool=rbd-pool'`.  

### 24.2 mkfs Is a High-Risk Operation  

The following commands format the data:  
```bash
mkfs.xfs /dev/rbd0
mkfs.ext4 /dev/rbd0
```  
Before executing them in production, ensure:  
- The device name is correct.  
- It’s not a data disk already in use by other services.  
- Backups have been taken.  
- Services have been stopped.  
- Any changes have been approved.  

### 24.3 Snapshots Are Not Equivalent to Backups  

RBD snapshots are convenient, but they rely on the same Ceph cluster. If the cluster fails, snapshots may also be lost. Therefore, in production:  
- Always maintain independent backups.  
- Implement cross-cluster data replication.  
- Use offline backup strategies.  
- Regularly conduct recovery drills.  

### 24.4 Manage Clone Dependencies Clearly  

By default, clones depend on the snapshot of their parent image. If you don’t flatten the clone, deleting the parent snapshot will fail. In production, clarify:  
- Which clones depend on which snapshots.  
- Whether flattening is necessary.  
- Whether deleting parent snapshots is allowed.  
- Whether clones are only for temporary testing purposes.  

### 24.5 RBD Is Not Suitable for Multi-Client Shared File Systems  

Avoid having multiple clients mount the same RBD volume using ordinary file systems like ext4/xfs for simultaneous read/write operations. This can cause:  
- File system corruption.  
- Data inconsistency.  
- Application failures.  

For multi-client shared fileshttps://docs.ceph.com/en/latest/rados/operations/placement-groups/  

Ceph CRUSH Map Documentation:  
https://docs.ceph.com/en/latest/rados/operations/crush-map/  

Ceph CSI Project:  
https://github.com/ceph/ceph-csi