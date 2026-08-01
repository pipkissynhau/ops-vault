# CephFS File Storage: MDS, File System Creation and Mounting Practice

Suggested path: 05-Storage/01-Ceph/10-CephFS File Storage: MDS, File System Creation and Mounting Practice.md

Tags: #Ceph #CephFS #MDS #FileStorage #SharedStorage #ReadWriteMany #LinuxMount #Kubernetes #CSI #SRE #DistributedStorage #AdvancedSre

---

## I. Document Explanation

This is the tenth article of the Ceph advanced SRE storage module, focusing on the theory, hands-on practice, and troubleshooting methods for CephFS file storage.

Previously completed:

- Ceph basic architecture
- RADOS, MON, MGR, OSD, CRUSH
- Differences between three storage types: RBD, CephFS, RGW
- cephadm cluster initialization
- OSD management
- Pool and PG
- CRUSH failure domain
- RBD block storage practice

This article enters the second core usage pattern of Ceph:

    CephFS file storage

CephFS, full name:

    Ceph File System

Can be understood as:

    A distributed shared file system provided by Ceph.

CephFS is suitable for:

- Multi-client shared directory
- Multi-node shared file
- Kubernetes ReadWriteMany PVC
- Shared upload directory
- Shared configuration directory
- AI/data processing shared data directory
- Replacing some NFS scenarios

This article covers:

- What is CephFS
- What is MDS
- Differences between Metadata Pool and Data Pool
- How to create CephFS file system
- How to deploy MDS
- How to mount CephFS on Linux client
- How to use kernel client mounting
- How to use ceph-fuse mounting
- How to perform read/write verification
- How to create minimal privilege user
- How to verify multi-client sharing
- How to check CephFS status
- How to troubleshoot MDS anomalies and mounting failures
- Relationship between CephFS and Kubernetes CephFS CSI
- Production environment considerations

---

## II. Experiment Objectives

After completing this article, you should be able to:

1. Understand the file storage model of CephFS.
2. Understand the role of MDS in CephFS.
3. Create CephFS Metadata Pool.
4. Create CephFS Data Pool.
5. Create CephFS file system.
6. Deploy MDS using cephadm.
7. Check CephFS status.
8. Install ceph-common and ceph-fuse on Linux client.
9. Mount CephFS using kernel client.
10. Mount CephFS using ceph-fuse.
11. Perform file read/write verification.
12. Verify multi-client shared read/write.
13. Create CephFS minimal privilege user.
14. Mount CephFS using non-admin user.
15. Understand the basic concepts of CephFS snapshots, quotas, and subdirectory mounting.
16. Troubleshoot CephFS mounting failures, MDS anomalies, permission errors, and performance issues.
17. Understand the relationship between CephFS and Kubernetes ReadWriteMany PVC.

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
| 10.0.0.35 | ceph-client | CephFS client testing (optional) |

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 Client Node Notes

It is recommended to prepare an independent client node:

    ceph-client
    10.0.0.35

Purpose:

- Install ceph-common
- Install ceph-fuse
- Obtain ceph.conf and keyring
- Mount CephFS
- Perform multi-client shared read/write testing
- Validate permission control

If you don't have a ceph-client temporarily, you can perform basic experiments on ceph-node01.

In production environments, it is recommended to:

    Separate Ceph cluster nodes and business client nodes.

---

## IV. What is CephFS

CephFS is the distributed file system capability provided by Ceph.

It provides clients with access similar to traditional file systems:

    Directory
    File
    Permissions
    inode
    Mount point
    Read/write operations

Clients can mount CephFS to a local directory:

    /mnt/cephfs

Then use it like a regular file system:

    mkdir
    touch
    cp
    mv
    rm
    cat
    echo

However, the underlying data is not stored on local disks but written to the Ceph cluster.

---

## V. Understanding CephFS Architecture

### 5.1 CephFS Data Path

The simplified architecture of CephFS is as follows:

    Application / User process
      |
      v
    Linux filesystem interface
      |
      v
    CephFS Client
      |
      v
    MDS handles metadata
      |
      v
    RADOS Pools
      |
      v
    OSD cluster

Diagram: /think

```
┌──────────────────────────┐
│        Application / User        │
└────────────┬─────────────┘
             v
┌──────────────────────────┐
│       /mnt/cephfs         │
└────────────┬─────────────┘
             v
┌──────────────────────────┐
│      CephFS Client        │
└────────────┬─────────────┘
             │
   ┌─────────┴─────────┐
   v                   v
┌──────────┐       ┌──────────────┐
│   MDS    │       │  OSD / RADOS  │
│ Metadata Service │   File Data    │
└──────────┘       └──────────────┘

---

### 5.2 What is MDS

MDS, full name:

    Metadata Server

MDS is the metadata service of CephFS.

It mainly handles:

- Directory structure
- File names
- inode
- File permissions
- File attributes
- File locks
- Metadata cache
- Directory traversal
- Metadata coordination across multiple clients

Simple understanding:

    CephFS file data is stored in OSDs.
    CephFS filesystem metadata is managed by MDS.

If you only use RBD or RGW, you don't need MDS.

If you use CephFS, you must deploy MDS.

---

### 5.3 Metadata Pool and Data Pool

CephFS typically requires at least two Pools:

| Pool | Purpose |
|---|---|
| cephfs_metadata | Stores filesystem metadata |
| cephfs_data | Stores file content data |

Illustration:

    CephFS
      |
      ├── Metadata Pool: directories, filenames, inode, permissions
      |
      └── Data Pool: actual file content

Notes:

    Metadata Pool is sensitive to latency.
    Data Pool mainly carries file storage capacity.
    In production environments, metadata pool and data pool can use different storage strategies.

---

## SixI don't know.CephFS Suitable and Unsuitable Scenarios

### 6.1 Suitable Scenarios

CephFS is suitable for:

- Multi-client shared directories
- Multi-Pod shared read/write
- Kubernetes ReadWriteMany PVC
- Web upload directory sharing
- Multi-node shared dataset
- Configuration file sharing
- AI / big data task shared data directory
- Replacement for some NFS scenarios

Typical scenarios:

    Multiple Web Pods sharing /uploads.
    Multiple compute nodes reading the same batch of training data.
    Multiple business instances sharing file-based data.

---

### 6.2 Unsuitable Scenarios

CephFS may not be suitable for:

- Database primary data disk
- Extremely high IOPS block device scenarios
- Scenarios with extremely sensitive metadata performance without tuning capabilities
- Environments without MDS operation experience
- Simple small-scale shared directories where NFS is sufficient
- Applications accessing objects via S3 API

Notes:

    Database primary data disks are usually better suited for RBD.
    Object storage scenarios are better suited for RGW or MinIO.
    CephFS has higher complexity than regular NFS, requiring attention to MDS status and metadata performance.

---

## SevenI don't know.Pre-Operation Checks

### 7.1 Check Ceph Cluster Status

    ceph -s

Ideal status:

    health: HEALTH_OK

At least confirm:

    mon is normal
    mgr is normal
    osd up/in
    pgs active+clean

If there are OSD down, PG degraded, or nearfull, it's not recommended to proceed with CephFS creation.

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
    No nearfull/full
    OSD usage has no obvious anomalies

---

### 7.4 Check if CephFS Exists

    ceph fs ls

If there is no filesystem, the output may be empty.

Check MDS status:

    ceph mds stat

If there is no CephFS, it may show no active MDS.

---

## EightI don't know.Experiment Checklist

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Create CephFS Metadata/Data Pool | Medium |
| Experiment 2 | Create CephFS Filesystem | Medium |
| Experiment 3 | Deploy MDS Service | Medium |
| Experiment 4 | Check CephFS Status | Low |
| Experiment 5 | Configure Linux Client | Medium |
| Experiment 6 | Mount CephFS with kernel client | Medium |
| Experiment 7 | Mount CephFS with ceph-fuse | Medium |
| Experiment 8 | File Read/Write Verification | Low |
| Experiment 9 | Multi-client Shared Read/Write Verification | Medium |
| Experiment 10 | Create Minimal Privilege User to Mount CephFS | Medium |
| Experiment 11 | View and Understand CephFS Snapshots | Medium |
| Experiment 12 | Clean Up Test Resources | High |

High-risk warning:

    Deleting CephFS will delete filesystem metadata.
    Deleting CephFS Pool will delete filesystem data.
    Do not directly delete CephFS or Pool in production environments.
    Using admin key to mount is only suitable for experiments, not for production.

---

## NineI don't know.Experiment 1: Create CephFS Pool

### 9.1 Create metadata pool

Execute on Ceph management node:

    ceph osd pool create cephfs_metadata 32

Notes:

    cephfs_metadata is used to store CephFS metadata.
    A small experimental environment can use 32 PGs.

---

### 9.2 Create data pool
```

# 8. Create CephFS Data Pool

ceph osd pool create cephfs_data 64

## Description:

- cephfs_data is used to store actual file content.
- A small experimental environment can use 64 PGs.

---

### 9.3 Enable application type

ceph osd pool application enable cephfs_metadata cephfs
ceph osd pool application enable cephfs_data cephfs

Check:

ceph osd pool application get cephfs_metadata
ceph osd pool application get cephfs_data

Expected:

{
    "cephfs": {}
}

---

### 9.4 Set replica count

Set metadata pool:

ceph osd pool set cephfs_metadata size 3
ceph osd pool set cephfs_metadata min_size 2

Set data pool:

ceph osd pool set cephfs_data size 3
ceph osd pool set cephfs_data min_size 2

Check:

ceph osd pool get cephfs_metadata size
ceph osd pool get cephfs_metadata min_size
ceph osd pool get cephfs_data size
ceph osd pool get cephfs_data min_size

---

### 9.5 Check Pool

ceph osd pool ls
ceph df

Confirm existence:

cephfs_metadata
cephfs_data

---

## TenI don't know.Experiment Two: Create CephFS File System

### 10.1 Create File System

Execute:

ceph fs new cephfs cephfs_metadata cephfs_data

Parameter description:

- cephfs: File system name
- cephfs_metadata: Metadata Pool
- cephfs_data: Data Pool

Check:

ceph fs ls

Expected:

name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]

---

### 10.2 Check File System Details

ceph fs get cephfs

Focus on:

- name
- metadata pool
- data pools
- max_mds
- flags
- standby_count_wanted
- session_timeout

---

### 10.3 Check MDS Status

ceph mds stat

If MDS hasn't been deployed yet, active MDS might not be visible.

Deployment of MDS needs to be done through cephadm later.

---

## ElevenI don't know.Experiment Three: Deploy MDS Service

### 11.1 Deploy MDS using cephadm

Deploy 2 MDS instances:

ceph orch apply mds cephfs --placement="2 ceph-node01 ceph-node02 ceph-node03"

Description:

- mds: Service type
- cephfs: File system name
- placement specifies deployment count and candidate nodes

---

### 11.2 Check MDS Service

ceph orch ps --daemon_type mds

Expected similar to:

mds.cephfs.ceph-node01.xxxxx  ceph-node01  running
mds.cephfs.ceph-node02.xxxxx  ceph-node02  running

---

### 11.3 Check CephFS Status

ceph fs status

Expected similar to:

cephfs - 0 clients
=====
RANK  STATE    MDS
0     active   cephfs.ceph-node01.xxxxx
          standby  cephfs.ceph-node02.xxxxx

Check MDS:

ceph mds stat

Expected:

cephfs:1 {0=cephfs.ceph-node01=up:active} 1 up:standby

Description:

- Active MDS handles current CephFS metadata service.
- Standby MDS is used for failover.

---

## TwelveI don't know.Experiment Four: CephFS Status Check

### 12.1 Check File System List

ceph fs ls

---

### 12.2 Check File System Details

ceph fs get cephfs

---

### 12.3 Check MDS Status

ceph mds stat

---

### 12.4 Check CephFS Status

ceph fs status

---

### 12.5 Check MDS Service

ceph orch ps --daemon_type mds

---

### 12.6 Check Cluster Status

ceph -s

Confirm:

- health normal
- mds normal
- pgs active+clean

---

## ThirteenI don't know.Experiment Five: Configure Linux Client

### 13.1 Install Client Tools on ceph-client

Ubuntu:

apt update
apt install -y ceph-common ceph-fuse

Rocky Linux 9:

dnf install -y ceph-common ceph-fuse

Verify:

ceph --version
ceph-fuse --version

---

### 13.2 Copy ceph.conf and keyring

If using admin key for experiment, execute on ceph-node01:

ssh root@ceph-client "mkdir -p /etc/ceph"

scp /etc/ceph/ceph.conf root@ceph-client:/etc/ceph/
scp /etc/ceph/ceph.client.admin.keyring root@ceph-client:/etc/ceph/

Verify on ceph-client:

    ls -l /etc/ceph/

Expected output:

    ceph.conf
    ceph.client.admin.keyring

---

### 13.3 Verifying Client Connection

Execute on ceph-client:

    ceph -s

Expected output:

    Should see Ceph cluster status.

If failed, troubleshoot:

- Does ceph.conf exist?
- Does keyring exist?
- Can ceph-client resolve Ceph nodes?
- Can ceph-client access MON ports 3300 / 6789?
- Are keyring permissions correct?

---

## FourteenI don't know.Experiment Six: Mounting CephFS with Kernel Client

### 14.1 Create Mount Directory

Execute on ceph-client:

    mkdir -p /mnt/cephfs-kernel

---

### 14.2 Extract Admin Secret

View key:

    ceph auth get-key client.admin

Save to file:

    ceph auth get-key client.admin > /etc/ceph/admin.secret

Set permissions:

    chmod 600 /etc/ceph/admin.secret

---

### 14.3 Mount with Kernel Client

Method 1: Mount using MON addresses

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-kernel \
      -o name=admin,secretfile=/etc/ceph/admin.secret,fs=cephfs

Check:

    df -hT | grep ceph

Expected output:

    CephFS mounted to /mnt/cephfs-kernel

---

### 14.4 Verify Mount

    mount | grep ceph
    df -hT /mnt/cephfs-kernel

Write test:

    echo "hello cephfs kernel client" > /mnt/cephfs-kernel/kernel-test.txt

Read:

    cat /mnt/cephfs-kernel/kernel-test.txt

Expected output:

    hello cephfs kernel client

---

### 14.5 Unmount

    umount /mnt/cephfs-kernel

If prompted "busy":

    lsof +f -- /mnt/cephfs-kernel
    fuser -vm /mnt/cephfs-kernel

Handle processes occupying the mount point before unmounting.

---

## FifteenI don't know.Experiment Seven: Mounting CephFS with ceph-fuse

### 15.1 Create Mount Directory

    mkdir -p /mnt/cephfs-fuse

---

### 15.2 Mount with ceph-fuse

    ceph-fuse --name client.admin --client_fs cephfs /mnt/cephfs-fuse

Check:

    df -hT | grep ceph-fuse

Or:

    mount | grep ceph-fuse

---

### 15.3 Write Test

    echo "hello cephfs fuse client" > /mnt/cephfs-fuse/fuse-test.txt

Read:

    cat /mnt/cephfs-fuse/fuse-test.txt

Expected output:

    hello cephfs fuse client

---

### 15.4 Unmount ceph-fuse

    fusermount -u /mnt/cephfs-fuse

If fusermount is not available, use:

    umount /mnt/cephfs-fuse

---

### 15.5 Comparison between kernel client and ceph-fuse

| Comparison Item | kernel client | ceph-fuse |
|---|---|---|
| Implementation | Linux kernel module | User-space FUSE |
| Performance | Usually better | Usually slightly lower |
| Compatibility | Depends on kernel version | More flexible deployment |
| Troubleshooting | dmesg, kernel logs | ceph-fuse logs |
| Production Usability | Commonly used | Also usable, depends on scenario |

Simple recommendation:

    Prioritize kernel client for production.
    Use ceph-fuse for compatibility or debugging scenarios.

---

## SixteenI don't know.Experiment Eight: Verifying Multi-Client Shared Read/Write

### 16.1 Prepare Two Clients

Can use:

    ceph-client01
    ceph-client02

If only one client is available, temporarily simulate with ceph-node01 and ceph-client.

Both clients mount the same CephFS:

    /mnt/cephfs

---

### 16.2 Client A Writes

Execute on client A:

    mkdir -p /mnt/cephfs-kernel/shared-demo

    echo "write from client A" > /mnt/cephfs-kernel/shared-demo/a.txt

---

### 16.3 Client B Reads

Execute on client B:

    cat /mnt/cephfs-kernel/shared-demo/a.txt

Expected output:

    write from client A

---

### 16.4 Client B Writes

    echo "write from client B" > /mnt/cephfs-kernel/shared-demo/b.txt

---

### 16.5 Client A Views

    ls -l /mnt/cephfs-kernel/shared-demo/
    cat /mnt/cephfs-kernel/shared-demo/b.txt

Expected output:

    Can see b.txt
    Content is "write from client B"

---

### 16.6 Experiment Conclusion

This experiment demonstrates:

CephFS can be mounted as a shared filesystem by multiple clients.
Suitable for multi-node shared directory scenarios.
Corresponds to ReadWriteMany storage capability in Kubernetes.

---

## SeventeenI don't know.Experiment Nine: Creating a Minimum Privilege User to Mount CephFS

### 17.1 Why Not Use the Admin Key

Using the admin key is for convenience in experiments.

It is not recommended for production use by business clients:

    client.admin

Reasons:

- Excessive permissions
- High risk of leakage
- Difficult to audit
- Does not comply with the principle of least privilege

Production should create dedicated users.

---

### 17.2 Creating a CephFS User

Create user:

    ceph auth get-or-create client.cephfs-user \
      mon 'allow r' \
      mds 'allow rw fsname=cephfs' \
      osd 'allow rw tag cephfs data=cephfs' \
      -o /etc/ceph/ceph.client.cephfs-user.keyring

Check:

    ceph auth get client.cephfs-user

Explanation:

    mon allow r: Allows reading cluster basic information
    mds allow rw fsname=cephfs: Allows read/write cephfs
    osd allow rw tag cephfs data=cephfs: Allows accessing cephfs data

---

### 17.3 Copying the User Keyring to the Client

    scp /etc/ceph/ceph.client.cephfs-user.keyring root@ceph-client:/etc/ceph/

Check on ceph-client:

    ls -l /etc/ceph/ceph.client.cephfs-user.keyring

---

### 17.4 Extracting the User Secret

Execute on ceph-client:

    ceph auth get-key client.cephfs-user > /etc/ceph/cephfs-user.secret

Set permissions:

    chmod 600 /etc/ceph/cephfs-user.secret

---

### 17.5 Mounting with Minimum Privilege User

Create directory:

    mkdir -p /mnt/cephfs-user

Mount:

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-user \
      -o name=cephfs-user,secretfile=/etc/ceph/cephfs-user.secret,fs=cephfs

Verify:

    echo "hello cephfs user" > /mnt/cephfs-user/user-test.txt
    cat /mnt/cephfs-user/user-test.txt

---

### 17.6 Unmount

    umount /mnt/cephfs-user

---

## EighteenI don't know.Experiment Ten: Subdirectory Mounting Basics

### 18.1 Creating a Subdirectory

First mount CephFS with admin or cephfs-user.

Create directory:

    mkdir -p /mnt/cephfs-kernel/apps/app01
    echo "app01 data" > /mnt/cephfs-kernel/apps/app01/data.txt

---

### 18.2 Mounting a Subdirectory

Create mount point:

    mkdir -p /mnt/app01

Mount subdirectory:

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/apps/app01 /mnt/app01 \
      -o name=cephfs-user,secretfile=/etc/ceph/cephfs-user.secret,fs=cephfs

Check:

    ls -l /mnt/app01
    cat /mnt/app01/data.txt

Expected:

    app01 data

---

### 18.3 Use Cases for Subdirectory Mounting

Subdirectory mounting is suitable for:

- Different businesses using different paths
- Avoiding direct visibility of the entire CephFS by business
- Directory-level isolation
- Combined use with permission controls

In production, it should also be combined with:

- CephX permissions
- Linux file permissions
- Kubernetes subdirectory mounting strategies
- Auditing and directory standards

---

## NineteenI don't know.Experiment Eleven: CephFS Snapshot Basics

### 19.1 CephFS Snapshot Overview

CephFS supports directory snapshot capabilities.

Snapshots are typically implemented through special snapshot directories under the directory:

    .snap

Availability depends on CephFS configuration.

Check the filesystem:

    ceph fs get cephfs

---

### 19.2 Creating a Test Directory

    mkdir -p /mnt/cephfs-kernel/snap-demo

Write file:

    echo "before cephfs snapshot" > /mnt/cephfs-kernel/snap-demo/file.txt

---

### 19.3 Creating a Snapshot

Enter directory:

    cd /mnt/cephfs-kernel/snap-demo

Create snapshot:

    mkdir .snap/snap01

Check:

    ls .snap

Expected:

    snap01

---

### 19.4 Modifying the File

    echo "after cephfs snapshot" > /mnt/cephfs-kernel/snap-demo/file.txt

Check current file:

    cat /mnt/cephfs-kernel/snap-demo/file.txt

Expected:

    after cephfs snapshot

Check file in snapshot:

    cat /mnt/cephfs-kernel/snap-demo/.snap/snap01/file.txt

Expected:

    before cephfs snapshot

---

### 19.5 Deleting a Snapshot

    rmdir /mnt/cephfs-kernel/snap-demo/.snap/snap01

Note:

    Snapshots are not independent backups.
    Snapshots still depend on the current Ceph cluster.
    In production, they should be combined with backup and recovery drills.

---

## TwentyI don't know.Experiment Twelve: Cleaning Up Test Resources

### 20.1 Unmounting the Client Mount Point /think

umount /mnt/cephfs-kernel
umount /mnt/cephfs-user
umount /mnt/app01

Ceph-fuse Unmount:

    fusermount -u /mnt/cephfs-fuse

If prompted busy:

    lsof +f -- /mnt/cephfs-kernel
    fuser -vm /mnt/cephfs-kernel

---

### 20.2 Delete Test User

Execute on Ceph management node:

    ceph auth del client.cephfs-user

Check:

    ceph auth ls | grep cephfs-user

---

### 20.3 Delete MDS Service

High-risk warning:

    Do not delete MDS service if there are business data in CephFS.

Can be deleted in experimental environment:

    ceph orch rm mds.cephfs

Check:

    ceph orch ps --daemon_type mds

---

### 20.4 Delete CephFS File System

High-risk warning:

    Deleting CephFS will affect the file system.
    Deleting Pool will delete all data.
    Prohibited in production environment.

Delete CephFS in experimental environment:

    ceph fs fail cephfs
    ceph fs rm cephfs --yes-i-really-mean-it

Check:

    ceph fs ls

---

### 20.5 Delete Pool

Enable Pool deletion:

    ceph config set mon mon_allow_pool_delete true

Delete:

    ceph osd pool rm cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
    ceph osd pool rm cephfs_data cephfs_data --yes-i-really-really-mean-it

Disable Pool deletion:

    ceph config set mon mon_allow_pool_delete false

Check:

    ceph osd pool ls

---

## Twenty-oneI don't know.CephFS Common Commands Summary

### 21.1 File System

    ceph fs ls
    ceph fs get cephfs
    ceph fs status
    ceph fs dump
    ceph mds stat

---

### 21.2 MDS

    ceph orch apply mds cephfs --placement="2 ceph-node01 ceph-node02 ceph-node03"
    ceph orch ps --daemon_type mds
    ceph mds stat

---

### 21.3 Pool

    ceph osd pool create cephfs_metadata 32
    ceph osd pool create cephfs_data 64
    ceph osd pool application enable cephfs_metadata cephfs
    ceph osd pool application enable cephfs_data cephfs

---

### 21.4 Create CephFS

    ceph fs new cephfs cephfs_metadata cephfs_data

---

### 21.5 Kernel Mount

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-kernel \
      -o name=admin,secretfile=/etc/ceph/admin.secret,fs=cephfs

---

### 21.6 ceph-fuse Mount

    ceph-fuse --name client.admin --client_fs cephfs /mnt/cephfs-fuse

---

### 21.7 Unmount

    umount /mnt/cephfs-kernel
    fusermount -u /mnt/cephfs-fuse

---

### 21.8 Permission User

    ceph auth get-or-create client.cephfs-user \
      mon 'allow r' \
      mds 'allow rw fsname=cephfs' \
      osd 'allow rw tag cephfs data=cephfs'

---

## Twenty-twoI don't know.Common Faults and Troubleshooting

### 22.1 CephFS Mount Failure

Common causes:

- ceph.conf does not exist
- keyring does not exist
- secretfile error
- MON address unreachable
- MDS not deployed
- MDS not active
- fs parameter error
- insufficient user permissions
- client lacks ceph-common
- kernel client compatibility issue

Troubleshoot:

    ceph -s
    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ls -l /etc/ceph/
    dmesg | tail -50

---

### 22.2 MDS Not Active

Check:

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds

Common causes:

- MDS not deployed
- MDS container abnormal
- MDS cannot connect to MON
- MDS node abnormal
- File system creation abnormal

Resolution:

    ceph orch apply mds cephfs --placement="2 ceph-node01 ceph-node02 ceph-node03"
    ceph orch ps --daemon_type mds
    ceph -s

---

### 22.3 ceph-fuse Mount Failure

Troubleshoot:

    ceph-fuse --name client.admin --client_fs cephfs /mnt/cephfs-fuse -d

Note:

    -d enables foreground debugging output.

Check:

    ceph -s
    ceph fs status
    ls -l /etc/ceph/

---

### 22.4 Insufficient Permissions

Symptom: /think

permission denied  
Operation not permitted  
access denied  

Troubleshoot user permissions:  

    ceph auth get client.cephfs-user  

Confirm the presence of:  

    mon allow r  
    mds allow rw fsname=cephfs  
    osd allow rw tag cephfs data=cephfs  

---  

### 22.5 Unmount Failure  

Phenomenon:  

    target is busy  

Troubleshoot:  

    lsof +f -- /mnt/cephfs-kernel  
    fuser -vm /mnt/cephfs-kernel  
    pwd  

Common causes:  

- Current shell is in a mounted directory  
- A process is reading/writing files  
- Application has not exited  
- Terminal is occupying the directory  

Resolution:  

    Exit the mounted directory.  
    Stop processes occupying the directory.  
    Re-execute umount.  

---  

### 22.6 Slow CephFS Access  

Troubleshooting directions:  

- Is MDS functioning normally?  
- Is MDS under high load?  
- Are there too many small files?  
- Are there too many directory entries?  
- Is OSD slow?  
- Is the network slow?  
- Is there Recovery / Backfill in progress?  
- Are there too many clients?  

Commands:  

    ceph -s  
    ceph fs status  
    ceph mds stat  
    ceph osd perf  
    ceph osd df  
    ceph pg stat  

Node-side checks:  

    top  
    iostat -x 1  
    vmstat 1  

---  

### 22.7 Files Not Visible or Multi-Client Sync Anomalies  

Troubleshoot:  

    mount | grep ceph  
    ceph fs status  
    ceph -s  
    dmesg | tail -50  

Verify:  

- Are both clients mounted to the same fs?  
- Are the mount paths consistent?  
- Are they mounted to different subdirectories?  
- Are permissions allowing read access?  
- Does the application have caching logic?  

---  

## Twenty-Three, Relationship Between CephFS and Kubernetes  

In Kubernetes, CephFS is typically used via CephFS CSI.  

Typical workflow:  

    StorageClass  
      |  
      v  
    PVC  
      |  
      v  
    CephFS CSI  
      |  
      v  
    CephFS Subvolume  
      |  
      v  
    Pod mounts shared directory  

CephFS in Kubernetes typically corresponds to:  

    ReadWriteMany  

Suitable for:  

- Multiple Pods sharing an upload directory  
- Multi-replica web applications sharing files  
- Multiple tasks sharing a dataset  
- Shared configuration or result directories  

Not suitable for:  

    Database primary data disks.  
    Single Pod exclusive high-performance block storage scenarios.  

Database primary data disks are better suited for:  

    RBD CSI  

This will be detailed in:  

    13-CephandKubernetesMatch:CephFS CSIFile-sharing storage practices.md  

---  

## Twenty-Four, Production Environment Considerations  

### 24.1 Do Not Use admin key for business clients  

Using admin key for experimentation is convenient.  

In production, use:  

    client.cephfs-user  

and restrict permissions.  

Principles:  

    Minimal permissions  
    User segmentation by business  
    Permission restrictions by path or filesystem  
    Keyring integrated into security management  
    Do not place admin key on business servers  

### 24.2 MDS Must Be Highly Available  

In production environments, CephFS must focus on MDS high availability.  

Recommendations:  

    At least 1 active MDS  
    At least 1 standby MDS  

Check with:  

    ceph fs status  
    ceph mds stat  

If no standby exists, MDS failure will impact filesystem availability.  

### 24.3 Metadata Pool Is Critical  

Metadata Pool stores filesystem metadata.  

If the metadata pool fails, CephFS is significantly impacted.  

Production recommendations:  

- Use reliable storage policies for Metadata Pool  
- Set reasonable replica counts  
- Monitor capacity  
- Monitor PG status  
- Avoid metadata pool nearfull  
- Focus on MDS performance for small file scenarios  

### 24.4 CephFS Is Not Backup  

CephFS provides shared filesystem capabilities.  

But it is not a backup system.  

Production still requires:  

- Independent backups  
- Snapshot strategies  
-Alien. backups  
- Recovery drills  
- Protection against accidental deletions  
- Permission audits  

### 24.5 Caution with Large Numbers of Small Files  

Large numbers of small files increase MDS pressure.  

Monitor:  

- Directory entry count  
- File count  
- MDS CPU  
- MDS memory  
- Metadata cache  
- Client count  
- Directory structure design  

If the business involves massive small files, pre-stress testing is essential.  

### 24.6 Do Not Use CephFS as Primary Database Storage  

Database primary data disks are typically better suited for RBD.  

CephFS is suitable for shared files, not all stateful applications.  

Selection principles:  

    Use like cloud disks: RBD  
    Use like shared directories: CephFS  
    Use like object storage: RGW / MinIO  

---  

## Twenty-Five, Advanced SRE Methodology  

### 25.1 CephFS's Key Is Not "Mountable", But MDS Status  

Basic understanding:  

    Mount success is sufficient.  

Advanced SRE understanding:  

    CephFS stability heavily depends on MDS.  
    MDS anomalies affect metadata operations.  
    Large numbers of small files, large directories, and multiple clients amplify MDS pressure.  

### 25.2 CephFS Troubleshooting Must Consider Clients, MDS, and OSD  

Client-side checks:  

    mount  
    df -hT  
    dmesg  
    lsof  
    fuser  

MDS-side checks:  

    ceph fs status  
    ceph mds stat  
    ceph orch ps --daemon_type mds  

OSD-side checks:  

    ceph -s  
    ceph osd perf  
    ceph osd df  
    ceph pg stat  

### 25.3 File Storage Must Focus on Permission Boundaries  

CephFS is a shared filesystem.  

Sharing means:  

- Multiple clients accessing  
- Multiple businesses sharing paths  
- Complex permission boundaries  
- Large impact range for accidental deletions  
- Higher risk of key leaks  

Production must design:  

- Directory standards  
- User permissions  
- Mount scope  
- Backup strategies  
- Audit strategies  

---  

## Twenty-Six, Interview Answering Approach  

If asked in an interview:  

    What is CephFS? How does it differ from RBD?  

You can answer:

# CephFS is a distributed file system provided by Ceph, suitable for multiple clients or multiple Pods to share the same file directory. It differs from RBD, which is block storage and more akin to cloud disks, typically used by single clients or single Pods exclusively; CephFS is file storage, providing file system semantics such as directories, files, permissions, and inodes, supporting shared read/write operations.
# CephFS relies on MDS, the Metadata Server. MDS is responsible for managing file system metadata, such as directory structures, filenames, inodes, permissions, and file locks. The file data itself is still stored on Ceph's OSDs. Typically, CephFS will have metadata pool and data pool, storing metadata and file data respectively.
# Operationally, CephFS is not just about mounting; it also requires attention to MDS active/standby status, metadata pool status, data pool status, PG cleanliness, client mount status, permission control, and small file performance issues.
# In Kubernetes, CephFS is typically provided via CephFS CSI for ReadWriteMany PVCs, suitable for multiple Pods to share directories; whereas scenarios like database primary data disks are better suited for RBD CSI.

---

## 27. Summary of This Chapter

This article mainly organizes the core content of CephFS file storage:

1. CephFS is a distributed shared file system provided by Ceph.
2. CephFS is suitable for multi-client directory sharing and Kubernetes ReadWriteMany scenarios.
3. CephFS relies on MDS to manage metadata.
4. CephFS typically uses metadata pool and data pool.
5. Metadata pool stores directory, filename, inode, permission, and other metadata.
6. Data pool stores file content.
7. Use `ceph fs new` to create a CephFS file system.
8. Use `ceph orch apply mds` to deploy MDS.
9. Linux clients can mount CephFS via kernel client or ceph-fuse.
10. Production environments should not use admin key for business clients.
11. CephFS supports multi-client shared read/write, but attention should be paid to permissions, MDS performance, and small file issues.
12. CephFS snapshots are not independent backups; production still requires backup and recovery drills.
13. Advanced SREs troubleshooting CephFS issues need to simultaneously check clients, MDS, and OSD.
14. In Kubernetes, CephFS is typically used for ReadWriteMany PVCs, and will be expanded through CephFS CSI later.

---

## 28. Reference Documents

CephFS official documentation:

    https://docs.ceph.com/en/latest/cephfs/

CephFS management documentation:

    https://docs.ceph.com/en/latest/cephfs/administration/

CephFS creating file system:

    https://docs.ceph.com/en/latest/cephfs/createfs/

CephFS client mounting:

    https://docs.ceph.com/en/latest/cephfs/mount-using-kernel-driver/

Ceph-FUSE mounting:

    https://docs.ceph.com/en/latest/cephfs/mount-using-fuse/

CephFS snapshots:

    https://docs.ceph.com/en/latest/cephfs/snapshots/

CephX permission management:

    https://docs.ceph.com/en/latest/rados/operations/user-management/

Ceph CSI project:

    https://github.com/ceph/ceph-csi