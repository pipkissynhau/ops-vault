# Ceph RBD Block Storage: Image, Snapshot, Clone, and Common Operations

Recommended path: 05-Storage/01-Ceph/09-Ceph RBD Block Storage: Image, Snapshot, Clone and Common Operations.md

Tags: #Ceph #RBD #BlockStorage #Image #Snapshot #Clone #LinuxMount #Kubernetes #CSI #SRE #DistributedStorage #AdvancedSre

---

## I. Document Overview

This is the ninth article in the Ceph advanced SRE storage module series, focusing on the theory, hands-on operations, and troubleshooting methods for Ceph RBD block storage.

Previously completed:

- Ceph foundation architecture
- RADOS, MON, MGR, OSD, CRUSH
- Differences between three storage types: RBD, CephFS, RGW
- cephadm cluster initialization
- OSD management
- Pool and PG
- CRUSH fault domain and data placement

This article begins the first core usage pattern of Ceph:

    RBD block storage

RBD, full name:

    RADOS Block Device

Can be understood as:

    Ceph's distributed cloud disk capability.

RBD is commonly used for:

- Virtual machine cloud disks
- OpenStack Cinder backend
- KubeVirt virtual machine disks
- Kubernetes RBD CSI PVC
- Database data disks
- Stateful application data disks
- Snapshot and clone scenarios

This article covers:

- What is RBD
- What is RBD Image
- How to create RBD Pool
- How to create RBD Image
- How to map RBD on Linux client
- How to format and mount RBD
- How to expand RBD Image
- How to create Snapshot
- How to roll back Snapshot
- How to create Clone based on Snapshot
- How to flatten Clone
- How to check RBD status and locks
- How to safely unmount and delete RBD
- Common troubleshooting
- Production environment considerations

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand the block storage model of RBD.
2. Create a dedicated RBD Pool.
3. Set Pool's size, min_size, and application.
4. Create RBD Image.
5. View RBD Image information.
6. Map RBD Image on Linux client.
7. Format RBD block device as XFS or ext4.
8. Mount RBD to local directory.
9. Complete basic read/write verification.
10. Expand RBD Image.
11. Online expand file system.
12. Create RBD Snapshot.
13. Roll back data from Snapshot.
14. Create Clone based on Snapshot.
15. Understand relationship between protected snapshot and clone.
16. Flatten Clone, remove dependency.
17. Safely unmount, unmap, and delete RBD Image.
18. Troubleshoot issues like RBD map failure, feature incompatibility, device occupation, snapshot deletion failure.
19. Understand relationship between RBD and Kubernetes RBD CSI.

---

## III. Experiment Environment

### 3.1 Ceph Cluster Nodes

This article continues using the Ceph module experiment environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (optional) |
| 10.0.0.35 | ceph-client | RBD Client Test (optional) |

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 Client Node Description

This article recommends preparing an independent client node:

    ceph-client
    10.0.0.35

Purpose:

- Install ceph-common
- Obtain ceph.conf and keyring
- Map RBD Image
- Format RBD block device
- Mount test
- Perform snapshot, rollback, and expansion experiments

If you don't have ceph-client temporarily, you can do basic experiments on ceph-node01.

But in production environments, it's recommended to:

    Separate Ceph cluster nodes and business client nodes.

---

## IV. What is RBD

RBD is Ceph's block storage capability.

It abstracts Ceph cluster storage resources into block devices.

To clients, RBD looks like a cloud disk:

    /dev/rbd0

Clients can perform:

- Create file system
- Mount directory
- Store database data
- Store application data
- Take snapshots
- Create clones
- Expand capacity

---

## V. Understanding RBD Architecture

### 5.1 RBD Data Path

The simplified data path of RBD is as follows:

    Application
      |
      v
    File system ext4 / xfs
      |
      v
    Linux block device /dev/rbdX
      |
      v
    RBD Image
      |
      v
    RADOS Objects
      |
      v
    Pool / PG / CRUSH
      |
      v
    OSD Cluster

Diagram: /think

```
┌──────────────────────┐
│        Application           │
└──────────┬───────────┘
           v
┌──────────────────────┐
│   File System ext4/xfs   │
└──────────┬───────────┘
           v
┌──────────────────────┐
│      /dev/rbd0        │
└──────────┬───────────┘
           v
┌──────────────────────┐
│      RBD Image        │
└──────────┬───────────┘
           v
┌──────────────────────┐
│    RADOS Objects      │
└──────────┬───────────┘
           v
┌──────────────────────┐
│       OSD Cluster        │
└──────────────────────┘

---

### 5.2 What is RBD Image

RBD Image can be understood as:

    A virtual disk in Ceph.

Example:

    rbd-pool/demo-image

Where:

    rbd-pool is the Pool name
    demo-image is the RBD Image name

Create a 10G RBD Image:

    rbd create demo-image --size 10G -p rbd-pool

This Image is not directly mapped to a single file on a node, but is split into multiple RADOS Objects and distributed across multiple OSDs via PG and CRUSH.

---

### 5.3 Differences Between RBD and Regular Disks

| Comparison Item | Regular Local Disk | Ceph RBD |
|---|---|---|
| Storage Location | Local disk on a single machine | Ceph multi-node OSD Cluster |
| High Availability | Depends on RAID or backup | Depends on Ceph replication and CRUSH |
| Scalability | Single machine expansion | Backend cluster can scale horizontally |
| Snapshot | Depends on file system or storage system | RBD natively supports Snapshot |
| Clone | Usually inconvenient | RBD supports Clone |
| Suitable Scenarios | Single machine applications | Cloud disks, virtual machines, K8s PVC |

---

## Six, Suitable and Unsuitable Scenarios for RBD

### 6.1 Suitable Scenarios

RBD is suitable for:

- Database data disks
- Virtual machine system disks
- Virtual machine data disks
- Kubernetes ReadWriteOnce PVC
- Single Pod exclusive persistent volumes
- Redis / MySQL / PostgreSQL / MongoDB data directories
- Harbor / GitLab / Jenkins and other stateful component data disks
- Block device scenarios requiring snapshots and clones

---

### 6.2 Unsuitable Scenarios

RBD is not suitable for:

- Multiple hosts simultaneously reading/writing the same volume via a regular file system
- Multiple Pods sharing the same read/write directory
- Scenarios requiring POSIX shared file system semantics
- Scenarios requiring S3 API access to objects

Reasons:

    RBD is a block device.
    Regular ext4 / xfs file systems typically do not support multiple clients simultaneously reading/writing the same block device.
    For multi-client shared file directories, consider CephFS.
    For object storage scenarios, consider RGW or MinIO.

---

## Seven, Pre-Experiment Checks

### 7.1 Check Ceph Cluster Status

    ceph -s

Ideal status:

    health: HEALTH_OK

At least confirm:

    mon is normal
    mgr is normal
    osd up/in
    pgs active+clean

If there are OSD down, PG degraded, nearfull, etc. issues, it is not recommended to continue with RBD write experiments.

---

### 7.2 Check OSD Status

    ceph osd tree

Confirm:

    All OSDs are up/in
    OSDs are distributed across different hosts
    No obvious abnormal OSDs

---

### 7.3 Check Capacity

    ceph df
    ceph osd df

Confirm:

    Cluster capacity is sufficient
    No nearfull / full
    OSD usage rates have no obvious anomalies

---

### 7.4 Check if ceph-common is installed on the client

On the ceph-client, execute:

    ceph --version
    rbd --version

If these commands are not available, install ceph-common.

Ubuntu:

    apt update
    apt install -y ceph-common xfsprogs

Rocky Linux 9:

    dnf install -y ceph-common xfsprogs

---

## Eight, Experiment Task List

| Experiment | Target | Risk Level |
|---|---|---|
| Experiment 1 | Create RBD Pool | Medium |
| Experiment 2 | Create RBD Image | Low |
| Experiment 3 | Linux client map RBD | Medium |
| Experiment 4 | Format and mount RBD | Medium-High |
| Experiment 5 | Write and read test data | Low |
| Experiment 6 | Expand RBD Image and file system | Medium |
| Experiment 7 | Create and view Snapshot | Low |
| Experiment 8 | Rollback Snapshot | High |
| Experiment 9 | Create Clone based on Snapshot | Medium |
| Experiment 10 | Flatten Clone | Medium |
| Experiment 11 | Secure unmount, unmap, and delete resources | Medium-High |

High-risk warning:

    mkfs will format the block device.
    rollback will roll back data state.
    rm image will delete RBD Image.
    snap purge will delete all snapshots.
    In production environments, confirm business downtime, data backup, and change window.
```

ceph osd pool ls

---

### 9.2 Enable RBD Application Type

    ceph osd pool application enable rbd-pool rbd

Check:

    ceph osd pool application get rbd-pool

Expected:

    {
        "rbd": {}
    }

---

### 9.3 Set Replication Count

Set replication count:

    ceph osd pool set rbd-pool size 3

Set minimum replication count:

    ceph osd pool set rbd-pool min_size 2

Check:

    ceph osd pool get rbd-pool size
    ceph osd pool get rbd-pool min_size

Expected:

    size: 3
    min_size: 2

---

### 9.4 Initialize RBD Pool

Execute:

    rbd pool init rbd-pool

Note:

    rbd pool init initializes RBD-related metadata.
    Some operations may be automatically handled in newer versions, but explicit execution is clearer.

---

### 9.5 Check Pool Status

    ceph df
    ceph osd pool get rbd-pool all
    ceph -s

Expected:

    Cluster health.
    rbd-pool exists.
    Pool application is rbd.

---

## TenI don't know.Experiment Two: Create RBD Image

### 10.1 Create 10G RBD Image

    rbd create demo-image --size 10G -p rbd-pool

Check:

    rbd ls -p rbd-pool

Expected:

    demo-image

---

### 10.2 Check Image Information

    rbd info rbd-pool/demo-image

Expected similar to:

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
| snapshot_count | Number of snapshots |

---

### 10.3 RBD Feature Explanation

Common features:

| Feature | Description |
|---|---|
| layering | Supports snapshot cloning |
| exclusive-lock | Exclusive lock |
| object-map | Object mapping for accelerated operations |
| fast-diff | Fast differential calculation |
| deep-flatten | Deep flatten support |

Note:

    Some older kernel RBD clients may not support certain features.
    If mapping fails, incompatible features may need to be disabled.
    Kubernetes RBD CSI typically handles related capabilities, but feature compatibility still needs to be understood.

---

## ElevenI don't know.Client Preparation: Configure ceph.conf and keyring

### 11.1 Copy Configuration File to ceph-client

If using an independent client ceph-client, copy from the Ceph management node:

    /etc/ceph/ceph.conf
    /etc/ceph/ceph.client.admin.keyring

Execute on ceph-node01:

    scp /etc/ceph/ceph.conf root@ceph-client:/etc/ceph/
    scp /etc/ceph/ceph.client.admin.keyring root@ceph-client:/etc/ceph/

If the ceph-client directory doesn't exist:

    ssh root@ceph-client "mkdir -p /etc/ceph"

Then re-copy the files.

---

### 11.2 Verify Ceph Connection on Client

Execute on ceph-client:

    ceph -s

Expected:

    Can see Ceph cluster status.

If failed, check:

- Whether /etc/ceph/ceph.conf exists
- Whether /etc/ceph/ceph.client.admin.keyring exists
- Whether ceph-client can resolve ceph-node01/02/03
- Whether ceph-client can access MON port 3300 / 6789
- Whether keyring permissions are correct

---

### 11.3 More Secure User Method Explanation

This experiment uses admin keyring for convenience in learning.

Production environments should not directly use admin key for business clients.

In production, create a user with minimal permissions, for example:

    client.rbd-user

And limit caps.

Example:

    ceph auth get-or-create client.rbd-user \
      mon 'profile rbd' \
      osd 'profile rbd pool=rbd-pool' \
      mgr 'profile rbd pool=rbd-pool'

Export:

    ceph auth get client.rbd-user -o /etc/ceph/ceph.client.rbd-user.keyring

Client usage:

    --id rbd-user

Further minimal permission strategies will be organized in the production practice section.

demo-image exists.

---

### 12.2 map RBD Image

Execute on ceph-client:

    rbd map rbd-pool/demo-image

Expected output similar to:

    /dev/rbd0

Check:

    rbd showmapped

Expected similar to:

    id  pool      namespace  image       snap  device
    0   rbd-pool             demo-image  -     /dev/rbd0

---

### 12.3 If map Fails: Feature Incompatibility

Possible error similar to:

    RBD image feature set mismatch

Or:

    sysfs write failed

Cause:

    Current Linux kernel RBD module does not support certain RBD features.

One way to handle this is to specify more conservative features when creating the image.

Example:

    rbd create demo-image-basic \
      --size 10G \
      -p rbd-pool \
      --image-feature layering

Then map:

    rbd map rbd-pool/demo-image-basic

Note:

    This is suitable for experimental compatibility handling.
    In production environments, it should be combined with kernel version, ceph-common version, and CSI version for unified planning.

---

### 12.4 View Block Device

    lsblk

Expected to see:

    rbd0  10G disk

Alternatively, check:

    fdisk -l /dev/rbd0

---

## ThirteenI don't know.Experiment Four: Format and Mount RBD

### 13.1 High-risk Warning

The following command will format the block device:

    mkfs.xfs /dev/rbd0

Before execution, must confirm:

    /dev/rbd0 is the newly created test RBD.
    /dev/rbd0 has no business data.
    It is not a system disk.
    It is not other business disks.

---

### 13.2 Format as XFS

    mkfs.xfs /dev/rbd0

If using ext4 instead:

    mkfs.ext4 /dev/rbd0

This article example uses XFS.

---

### 13.3 Create Mount Directory

    mkdir -p /mnt/rbd-demo

Mount:

    mount /dev/rbd0 /mnt/rbd-demo

Check:

    df -hT | grep rbd

Expected similar to:

    /dev/rbd0 xfs 10G ... /mnt/rbd-demo

---

### 13.4 Write Test Data

    echo "hello ceph rbd" > /mnt/rbd-demo/hello.txt

Check:

    cat /mnt/rbd-demo/hello.txt

Expected:

    hello ceph rbd

Create a larger test file:

    dd if=/dev/zero of=/mnt/rbd-demo/test-100m.bin bs=1M count=100 oflag=direct

Check:

    ls -lh /mnt/rbd-demo/

---

### 13.5 Check Ceph Status

    ceph -s
    ceph df
    rbd du rbd-pool/demo-image

Note:

    rbd du can view the actual usage of the image.
    If features like fast-diff are enabled, the statistics will be faster.

---

## FourteenI don't know.Experiment Five: Unmount and unmap RBD

### 14.1 Unmount File System

On ceph-client execute:

    umount /mnt/rbd-demo

If prompted busy, check the usage:

    lsof +f -- /mnt/rbd-demo

Or:

    fuser -vm /mnt/rbd-demo

Handle the occupying processes before unmounting.

---

### 14.2 unmap RBD

Check mapping:

    rbd showmapped

Unmap:

    rbd unmap /dev/rbd0

Check again:

    rbd showmapped

Expected:

    No longer shows demo-image.

---

### 14.3 Common unmap Failure Reasons

Common reasons:

- File system is still mounted
- There are processes occupying the directory
- Current working directory is within the mount point
- There is active IO on the device
- Multipath or lock anomalies

Troubleshoot:

    mount | grep rbd
    lsof +f -- /mnt/rbd-demo
    fuser -vm /mnt/rbd-demo
    rbd showmapped

---

## FifteenI don't know.Experiment Six: Resize RBD Image

### 15.1 Experiment Objective

Resize RBD Image from 10G to 20G.

Prerequisites:

    RBD has already been mapped to /dev/rbd0.
    File system is XFS or ext4.
    Currently mounted at /mnt/rbd-demo.

If already unmap, need to re-map and remount:

    rbd map rbd-pool/demo-image
    mount /dev/rbd0 /mnt/rbd-demo

---

### 15.2 Check Current Size

Check RBD Image:

    rbd info rbd-pool/demo-image

Check block device:

    lsblk

Check file system:

    df -hT /mnt/rbd-demo

---

### 15.3 Resize RBD Image

    rbd resize rbd-pool/demo-image --size 20G

Check:

    rbd info rbd-pool/demo-image

Expected:

    size 20 GiB

---

### 15.4 Refresh Client Block Device

Check:

    lsblk

If /dev/rbd0 still shows old size, can try:

    blockdev --getsize64 /dev/rbd0

Most of the time, rbd devices will recognize the new size.

Alternatively, unmap and re-map:

    umount /mnt/rbd-demo
    rbd unmap /dev/rbd0
    rbd map rbd-pool/demo-image
    mount /dev/rbd0 /mnt/rbd-demo

---

### 15.5 Resize XFS File System

If file system is XFS:

    xfs_growfs /mnt/rbd-demo

Check: /think

df -hT /mnt/rbd-demo

Expected:

    The file system capacity becomes approximately 20G.

---

### 15.6 Expanding ext4 File System

If the file system is ext4:

    resize2fs /dev/rbd0

Check:

    df -hT /mnt/rbd-demo

---

### 15.7 Production Considerations

RBD expansion typically supports online expansion, but in production environments, the following should still be noted:

- Whether the application supports online expansion
- File system type
- Whether there is a backup
- Whether there is monitoring
- Whether it is during business peak hours
- In Kubernetes, it also involves PVC, StorageClass, CSI, and file system expansion support

---

## SixteenI don't know.Experiment Seven: Creating RBD Snapshot

### 16.1 Writing Test File

Ensure RBD is mounted:

    df -hT | grep rbd

Write file:

    echo "before snapshot" > /mnt/rbd-demo/snapshot-test.txt

Check:

    cat /mnt/rbd-demo/snapshot-test.txt

---

### 16.2 Synchronizing Data Before Creating Snapshot

Execute:

    sync

Note:

    It is recommended to ensure the file system data has been flushed to disk before creating a snapshot.
    For databases and other business systems, additional application-level consistency handling is required in production.

---

### 16.3 Creating Snapshot

RBD Snapshot format:

    pool/image@snapshot

Create:

    rbd snap create rbd-pool/demo-image@snap01

Check:

    rbd snap ls rbd-pool/demo-image

Expected:

    snap01

---

### 16.4 Significance of Snapshot

Snapshot is a record of the Image state at a specific moment.

Suitable for:

- Protection before changes
- Protection before upgrades
- Protection before testing
- Quick rollback
- Creating Clone

Note:

    Snapshot is not equal to a full backup.
    If data in the underlying Ceph cluster is lost, the snapshot may also be affected.
    A separate backup strategy is still required in production.

---

## SeventeenI don't know.Experiment Eight: Rolling Back Snapshot

### 17.1 High-Risk Warning

Snapshot rollback will revert the Image to the state at the snapshot moment.

Risks:

    Data written after the snapshot may be lost.
    Business should be stopped before rollback.
    The file system should be unmounted before rollback.
    Confirm whether a backup of the current state is needed before rollback.

---

### 17.2 Modifying Test File

Write new content:

    echo "after snapshot" > /mnt/rbd-demo/snapshot-test.txt

Check:

    cat /mnt/rbd-demo/snapshot-test.txt

Expected:

    after snapshot

---

### 17.3 Unmount and unmap

It is recommended to unmount before rollback:

    umount /mnt/rbd-demo

Check mapping:

    rbd showmapped

Unmap:

    rbd unmap /dev/rbd0

---

### 17.4 Executing Rollback

    rbd snap rollback rbd-pool/demo-image@snap01

Note:

    Rollback time depends on the amount of changed data.
    The larger the data volume, the longer the rollback time.

---

### 17.5 Remapping and Verification

Remap:

    rbd map rbd-pool/demo-image

Mount:

    mount /dev/rbd0 /mnt/rbd-demo

Check file:

    cat /mnt/rbd-demo/snapshot-test.txt

Expected:

    before snapshot

Note:

    The file content has been rolled back to the state at snap01.

---

## EighteenI don't know.Experiment Nine: Creating Clone Based on Snapshot

### 18.1 Prerequisites for Clone

RBD Clone is created based on Snapshot.

However, the Snapshot needs to be protected first:

    rbd snap protect rbd-pool/demo-image@snap01

Check:

    rbd snap ls rbd-pool/demo-image

---

### 18.2 Creating Clone

Create clone:

    rbd clone rbd-pool/demo-image@snap01 rbd-pool/demo-clone

Check:

    rbd ls -p rbd-pool

Expected:

    demo-image
    demo-clone

Check clone information:

    rbd info rbd-pool/demo-clone

You can see parent information, indicating it depends on the parent Image's snapshot.

---

### 18.3 Mapping Clone

    rbd map rbd-pool/demo-clone

Check:

    rbd showmapped

Assume clone is mapped to:

    /dev/rbd1

Create mount directory:

    mkdir -p /mnt/rbd-clone

If the clone already has a file system, you can directly mount:

    mount /dev/rbd1 /mnt/rbd-clone

Check:

    ls -l /mnt/rbd-clone

---

### 18.4 Value of Clone

Clone is suitable for:

- Quickly creating test environments
- Creating new disks based on template disks
- Quickly replicating virtual machine disks
- Database test replicas
- CI test data environments

Advantages:

    Fast creation.
    Does not initially copy all data.
    Saves space through Copy-on-Write.

---

## NineteenI don't know.Experiment Ten: Flatten Clone

### 19.1 Why Flatten is Needed

Clone defaults to relying on the parent Image's snapshot.

Relationship:

    demo-clone
      |
      v
    demo-image@snap01

If you want the clone to exist independently, you need to flatten.

After flattening:

    demo-clone no longer depends on demo-image@snap01

---

### 19.2 Executing Flatten

Unmount clone first:

    umount /mnt/rbd-clone

Unmap:

    rbd unmap /dev/rbd1

Execute flatten:

    rbd flatten rbd-pool/demo-clone

Check:

    rbd info rbd-pool/demo-clone

Expected:

    Parent information disappears.

### 19.3 Unprotect and Delete Snapshot

After confirming there are no clone dependencies, you can unprotect:

    rbd snap unprotect rbd-pool/demo-image@snap01

Delete the snapshot:

    rbd snap rm rbd-pool/demo-image@snap01

Check:

    rbd snap ls rbd-pool/demo-image

---

## 20. Experiment Eleven: Clean Up RBD Resources

### 20.1 Unmount Mount Points

    umount /mnt/rbd-demo
    umount /mnt/rbd-clone

If prompted that the mount point does not exist or is not mounted, you can ignore the message.

Check:

    mount | grep rbd

---

### 20.2 Unmap RBD

Check:

    rbd showmapped

Unmap each device individually:

    rbd unmap /dev/rbd0
    rbd unmap /dev/rbd1

If the device numbers differ, adjust according to the output of `rbd showmapped`.

---

### 20.3 Delete Clone

Check:

    rbd ls -p rbd-pool

Delete clone:

    rbd rm rbd-pool/demo-clone

---

### 20.4 Delete Snapshot

Check snapshots:

    rbd snap ls rbd-pool/demo-image

If the snapshot is still protected:

    rbd snap unprotect rbd-pool/demo-image@snap01

Delete the snapshot:

    rbd snap rm rbd-pool/demo-image@snap01

If you want to delete all snapshots:

    rbd snap purge rbd-pool/demo-image

High-risk warning:

    snap purge will delete all snapshots of this Image.
    Do not execute in production environments arbitrarily.

---

### 20.5 Delete Original Image

    rbd rm rbd-pool/demo-image

Confirm:

    rbd ls -p rbd-pool

---

### 20.6 Delete Pool

Confirm before deletion:

    ceph osd pool ls
    rbd ls -p rbd-pool
    rados -p rbd-pool ls

Enable pool deletion:

    ceph config set mon mon_allow_pool_delete true

Delete:

    ceph osd pool rm rbd-pool rbd-pool --yes-i-really-really-mean-it

Disable pool deletion:

    ceph config set mon mon_allow_pool_delete false

Production warning:

    Deleting a Pool will delete all Images and objects within it.
    Must be confirmed through backup, business confirmation, and change approval in production environments.

---

## 21. Common RBD Commands Summary

### 21.1 Pool

    ceph osd pool create rbd-pool 64
    ceph osd pool application enable rbd-pool rbd
    ceph osd pool set rbd-pool size 3
    ceph osd pool set rbd-pool min_size 2
    rbd pool init rbd-pool

---

### 21.2 Image

    rbd create demo-image --size 10G -p rbd-pool
    rbd ls -p rbd-pool
    rbd info rbd-pool/demo-image
    rbd du rbd-pool/demo-image
    rbd resize rbd-pool/demo-image --size 20G
    rbd rm rbd-pool/demo-image

---

### 21.3 Map / Unmap

    rbd map rbd-pool/demo-image
    rbd showmapped
    rbd unmap /dev/rbd0

---

### 21.4 Snapshot

    rbd snap create rbd-pool/demo-image@snap01
    rbd snap ls rbd-pool/demo-image
    rbd snap rollback rbd-pool/demo-image@snap01
    rbd snap rm rbd-pool/demo-image@snap01
    rbd snap purge rbd-pool/demo-image

---

### 21.5 Clone

    rbd snap protect rbd-pool/demo-image@snap01
    rbd clone rbd-pool/demo-image@snap01 rbd-pool/demo-clone
    rbd info rbd-pool/demo-clone
    rbd flatten rbd-pool/demo-clone
    rbd snap unprotect rbd-pool/demo-image@snap01
    rbd rm rbd-pool/demo-clone

---

### 21.6 Status and Locks

    rbd status rbd-pool/demo-image
    rbd lock list rbd-pool/demo-image
    rbd lock remove rbd-pool/demo-image <lock-id> <locker>

Notes:

    lock remove is a high-risk operation.
    Must confirm that clients have abnormally exited before executing in production environments, otherwise it may cause data risks.

---

## 22. Common Issues and Troubleshooting

### 22.1 rbd map Failed

Common causes:

- ceph.conf does not exist
- keyring does not exist
- MON connection failed
- Client cannot resolve Ceph nodes
- RBD feature is incompatible with kernel
- Insufficient permissions
- Image does not exist

Troubleshoot:

    ceph -s
    rbd ls -p rbd-pool
    rbd info rbd-pool/demo-image
    ls -l /etc/ceph/
    dmesg | tail -50

If the issue is feature incompatibility, you can create a conservative feature Image: /think

rbd create demo-image-basic \
  --size 10G \
  -p rbd-pool \
  --image-feature layering

---

### 22.2 Mount Failed

Troubleshooting:

    rbd showmapped
    lsblk
    blkid /dev/rbd0
    dmesg | tail -50

Common Causes:

- RBD is not mapped
- Device name error
- No file system
- File system corruption
- Mount point does not exist

Resolution:

    mkdir -p /mnt/rbd-demo
    mkfs.xfs /dev/rbd0
    mount /dev/rbd0 /mnt/rbd-demo

Note:

    mkfs will erase existing file system data on the device.
    Do not execute in production environments arbitrarily.

---

### 22.3 Unmap Failed

Symptoms:

    rbd: sysfs write failed
    device is busy

Troubleshooting:

    mount | grep rbd
    lsof +f -- /mnt/rbd-demo
    fuser -vm /mnt/rbd-demo
    rbd showmapped

Resolution:

    Exit the mounted directory.
    Stop the occupying process.
    Unmount the mount point.
    Execute rbd unmap again.

---

### 22.4 Snapshot Deletion Failed

Symptoms:

    snapshot is protected from removal

Cause:

    The snapshot is protected, usually due to clone dependencies.

Check:

    rbd children rbd-pool/demo-image@snap01

Resolution:

    Delete or flatten the dependent clone.
    Remove protection.
    Delete the snapshot again.

Commands:

    rbd flatten rbd-pool/demo-clone
    rbd snap unprotect rbd-pool/demo-image@snap01
    rbd snap rm rbd-pool/demo-image@snap01

---

### 22.5 Image Deletion Failed

Common Causes:

- Image is still mapped
- Image has snapshots
- Image has clone dependencies
- Image is locked by clients

Troubleshooting:

    rbd status rbd-pool/demo-image
    rbd snap ls rbd-pool/demo-image
    rbd children rbd-pool/demo-image@snap01
    rbd showmapped

Resolution:

    Stop business operations first.
    Unmount the file system.
    Unmap RBD.
    Delete snapshots or clones.
    Delete the Image finally.

---

### 22.6 Poor RBD Performance

Troubleshooting Directions:

- OSD latency
- OSD usage
- PG status
- Network latency
- Client IO mode
- File system type
- RBD feature
- Pool replica count
- Background Recovery / Backfill

Commands:

    ceph -s
    ceph osd perf
    ceph osd df
    ceph pg stat
    rbd perf image iostat

If iostat is available:

    iostat -x 1

---

## Twenty-Three, Relationship Between RBD and Kubernetes

In Kubernetes, RBD is typically used via Ceph CSI.

Typical Chain:

    StorageClass
      |
      v
    PVC
      |
      v
    Ceph RBD CSI
      |
      v
    RBD Image
      |
      v
    Pod Mount

RBD in Kubernetes typically corresponds to:

    ReadWriteOnce

Suitable for:

- Single Pod exclusive data disk
- Databases
- Stateful services
- Application data directories

Not suitable for:

    Multiple Pods sharing read/write the same directory simultaneously.

For multiple Pods sharing directories, consider:

    CephFS CSI

This will be detailed in:

    12-CephandKubernetesMatch:RBD CSIDynamic volume supply practice.md

---

## Twenty-Four, Production Environment Notes

### 24.1 Do Not Use admin key for business clients

Using admin key for experiments is for convenience.

In production, use a user with minimal permissions:

    client.rbd-user

And restrict to a specific Pool:

    osd 'profile rbd pool=rbd-pool'

---

### 24.2 mkfs is a high-risk operation

The following commands will format data:

    mkfs.xfs /dev/rbd0
    mkfs.ext4 /dev/rbd0

Before executing in production, must confirm:

- Correct device name
- Not a business data disk
- Completed backup
- Business has been stopped
- Change has been approved

---

### 24.3 Snapshot is Not Equal to Backup

RBD Snapshot is convenient, but it still depends on the same Ceph cluster.

If the underlying cluster is severely damaged, the snapshot may also be affected.

In production, still need:

- Independent backup
- Cross-cluster replication
- Offline backup
- Regular recovery drills

---

### 24.4 Clone dependencies must be managed clearly

Clone defaults to depend on the parent Image's Snapshot.

Without flattening, deleting the parent snapshot will fail.

In production, need to clarify:

- Which clones depend on which snapshots
- Whether to flatten
- Whether to allow deletion of parent snapshot
- Whether the clone is just a temporary test disk

---

### 24.5 RBD is unsuitable for multiple clients sharing a regular file system

Do not allow multiple clients to mount the same RBD with regular ext4/xfs and write simultaneously.

This may lead to:

- File system corruption
- Data inconsistency
- Application anomalies

For multiple clients sharing files, use:

    CephFS

---

### 24.6 RBD expansion must combine with file system

RBD Image expansion only increases the block device size.

Also need to expand the file system:

- XFS uses xfs_growfs
- ext4 uses resize2fs

In Kubernetes, also check:

- Whether StorageClass allows expansion
- Whether PVC has modified capacity
- Whether CSI supports file system expansion
- Whether Pod needs to restart

---

## Twenty-Five, Advanced SRE Methodology

### 25.1 RBD is a block device, not a shared file system

Advanced SRE must understand: /think

RBD is like a cloud hard disk.
CephFS is like a shared file system.
RGW is like an object storage.

Wrong selection will greatly increase the complexity of subsequent operations and maintenance.

---

### 25.2 RBD Troubleshooting Should Be Viewed from Both Client and Cluster Sides

RBD issues cannot be viewed solely from the client side.

You need to check both:

Client side:

    rbd showmapped
    lsblk
    mount
    dmesg
    lsof
    fuser

Ceph cluster side:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph osd perf
    ceph pg stat
    rbd status

---

### 25.3 Snapshots and Clones Are Change Protection Tools, Not Universal Rollback Solutions

RBD Snapshot is suitable for protecting before changes.

But in production, you still need:

- Application consistency
- Database flush / lock / backup
- Independent backup
- Recovery drills
- Snapshot lifecycle management

---

## Twenty-six, Interview Answering Strategy

If asked in an interview:

    What is Ceph RBD? How is it generally used?

You can answer:

    Ceph RBD is the block storage capability provided by Ceph, full name is RADOS Block Device, which can be understood as a distributed cloud hard disk. It abstracts storage resources in the Ceph cluster into RBD Image, and clients can map the Image into a block device similar to /dev/rbd0, then create ext4 or XFS file systems and mount them for use.
    The underlying data of RBD is divided into RADOS Objects, which are distributed to multiple OSDs through Pool, PG, and CRUSH, and data reliability is ensured by the replica count and failure domain of the Pool.
    Common use cases include virtual machine cloud disks, database data disks, Kubernetes ReadWriteOnce PVCs, and data disks for stateful applications.
    Daily operations include creating RBD Pool, enabling rbd application, creating Image, mapping to clients, formatting and mounting, expansion, creating Snapshot, Clone, and Flatten.
    In operations, note that RBD is a block device, and ordinary ext4/xfs is unsuitable for multiple clients to read/write simultaneously; Snapshot is not equal to backup; production clients should not directly use admin key but create users with minimal permissions; deleting Image, rolling back Snapshot, and formatting devices are all high-risk operations.
    In Kubernetes, RBD is usually provided through Ceph CSI for dynamic volume provisioning, suitable for persistent storage scenarios where a single Pod exclusively uses it.

---

## Twenty-seven, Summary of This Chapter

This article mainly organizes the core content of Ceph RBD block storage:

1. RBD is the block storage capability provided by Ceph, similar to a distributed cloud hard disk.
2. RBD Image is a virtual disk in Ceph.
3. The underlying data of RBD is divided into RADOS Objects and distributed to OSDs through Pool, PG, and CRUSH.
4. RBD is suitable for virtual machine cloud disks, database data disks, Kubernetes ReadWriteOnce PVCs.
5. RBD is unsuitable for ordinary multi-client shared write scenarios.
6. In experiments, rbd-pool and demo-image were created.
7. The Image was mapped to /dev/rbd0 via rbd map.
8. The file system was mounted via mkfs.xfs and mount.
9. Expansion was completed via rbd resize and xfs_growfs.
10. Snapshots were created via rbd snap create.
11. Snapshot rollback was verified via rbd snap rollback.
12. Clones were created via rbd clone.
13. The dependency on the parent snapshot was removed via rbd flatten.
14. In production, minimal permission users should be used instead of directly using admin key.
15. RBD Snapshot is not an independent backup; production still requires backup and recovery drills.
16. Advanced SREs troubleshooting RBD issues need to check both client status and Ceph cluster status.

---

## Twenty-eight, Reference Documents

Ceph RBD official documentation:

    https://docs.ceph.com/en/latest/rbd/

RBD command documentation:

    https://docs.ceph.com/en/latest/man/8/rbd/

RBD snapshot documentation:

    https://docs.ceph.com/en/latest/rbd/rbd-snapshot/

RBD Exclusive Lock:

    https://docs.ceph.com/en/latest/rbd/rbd-exclusive-locks/

Ceph RADOS Pool documentation:

    https://docs.ceph.com/en/latest/rados/operations/pools/

Ceph Placement Groups documentation:

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph CRUSH Map documentation:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph CSI project:

    https://github.com/ceph/ceph-csi