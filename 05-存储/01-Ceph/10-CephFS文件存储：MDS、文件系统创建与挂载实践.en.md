# CephFS File Storage: MDS, File System Creation, and Mounting Practices

Recommended Path: 05-Storage/01-Ceph/10-CephFS File Storage: MDS, File System Creation, and Mounting Practices.md

Tags: #Ceph #CephFS #MDS #File Storage #Shared Storage #ReadWriteMany #Linux Mounting #Kubernetes #CSI #SRE #Distributed Storage #Advanced SRE

---

## I. Document Overview

This article is the tenth in the Ceph Advanced SRE storage series, focusing on the theory, practical operations, and troubleshooting methods of CephFS file storage.

Previous topics covered include:

- Ceph Architecture
- RADOS, MON, MGR, OSD, CRUSH
- Differences between RBD, CephFS, and RGW Storage Types
- cephadm Cluster Initialization
- OSD Management
- Pools and PGs
- CRUSH Fault Domains
- RBD Block Storage Practices

This article delves into the second core usage of Ceph:

    CephFS File Storage

CephFS, short for:

    Ceph File System,

can be understood as:

    a distributed shared file system provided by Ceph.

CephFS is suitable for:

- Multi-client shared directories
- Multi-node shared files
- Kubernetes ReadWriteMany PVCs
- Shared upload directories
- Shared configuration directories
- AI/data processing shared data directories
- Replacing certain NFS use cases

This article covers the following key areas:

- What CephFS is
- What MDS is
- Differences between Metadata Pool and Data Pool
- How to create a CephFS file system
- How to deploy MDS
- How to mount CephFS on Linux clients
- How to use the kernel client for mounting
- How to use ceph-fuse for mounting
- How to perform read and write verification
- How to create a user with minimal permissions
- How to verify multi-client sharing
- How to check the status of CephFS
- How to troubleshoot MDS exceptions and mount failures
- The relationship between CephFS and Kubernetes CephFS CSI
- Precautions for production environments

---

## II. Experimental Objectives

After completing this article, you should be able to:

1. Understand the file storage model of CephFS.
2. Comprehend the role of MDS in CephFS.
3. Create a CephFS Metadata Pool.
4. Create a CephFS Data Pool.
5. Establish a CephFS file system.
6. Deploy MDS using cephadm.
7. Check the status of CephFS.
8. Install ceph-common and ceph-fuse on a Linux client.
9. Mount CephFS using the kernel client.
10. Mount CephFS using ceph-fuse.
11. Perform file read and write verification.
12. Verify multi-client shared reading and writing.
13. Create a user with minimal permissions for CephFS.
14. Mount CephFS using a non-admin user.
15. Understand the basic concepts of CephFS snapshots, quotas, and subdirectory mounting.
16. Troubleshoot issues such as mount failures, MDS exceptions, permission errors, and performance bottlenecks in CephFS.
17. Comprehend the relationship between CephFS and Kubernetes ReadWriteMany PVCs.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article uses the same experimental environment as previous Ceph modules.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Testing (optional) |
| 10.0.0.35 | ceph-client | CephFS Client Testing (optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

---

### 3.2 Client Node Description

It is recommended to prepare a dedicated client node:

    ceph-client
    10.0.0.35

Purpose:

- Install ceph-common and ceph-fuse.
- Retrieve the ceph.conf and keyring files.
- Mount CephFS.
- Conduct multi-client shared reading and writing tests.
- Verify permission controls.

If you don't have a dedicated ceph-client node, basic experiments can still be performed on ceph-node01.

In production environments,```markdown
ceph osd df

Confirmation:

    The cluster capacity is sufficient.
    There are no nearfull/full status conditions.
    The OSD utilization does not show any significant abnormalities.

---

### 7.4 Checking if CephFS Already Exists

    ceph fs ls

If there is no file system, the output may be empty.

Checking the MDS status:

    ceph mds stat

If there is no CephFS, it may indicate that there are no active MDS instances.

---

## VIII. Experimental Task List

| Experiment | Objective | Risk Level |
|-------------|--------------|---------------|
| Experiment 1 | Create CephFS Metadata/Data Pool | Medium |
| Experiment 2 | Create a CephFS File System | Medium |
| Experiment 3 | Deploy the MDS Service | Medium |
| Experiment 4 | Check the Status of CephFS | Low |
| Experiment 5 | Configure a Linux Client | Medium |
| Experiment 6 | Mount CephFS Using the Kernel Client | Medium |
| Experiment 7 | Mount CephFS Using ceph-fuse | Medium |
| Experiment 8 | Verify File Read/Write Operations | Low |
| Experiment 9 | Verify Multi-Client Shared Read/Write | Medium |
| Experiment 10 | Create a User with Minimum Permissions to Mount CephFS | Medium |
| Experiment 11 | Understand and View CephFS Snapshots | Medium |
| Experiment 12 | Clean Up Test Resources | High |

High-Risk Warning:

    Deleting CephFS will erase the file system metadata.
    Deleting a CephFS Pool will remove the file system data.
    Direct deletion of CephFS or Pool is not allowed in a production environment.
    Using the admin key for mounting is suitable only for experiments and not for production use.

---

## IX. Experiment 1: Creating a CephFS Pool

### 9.1 Creating the Metadata Pool

Execute on the Ceph management node:

    ceph osd pool create cephfs_metadata 32

Explanation:

    `cephfs_metadata` is used to store CephFS metadata.
    For small experimental environments, using 32 PGs is sufficient.

---

### 9.2 Creating the Data Pool

    ceph osd pool create cephfs_data 64

Explanation:

    `cephfs_data` is used to store the actual file content.
    In a small experimental environment, 64 PGs can be used.

---

### 9.3 Enabling the Application Type

    ceph osd pool application enable cephfs_metadata cephfs
    ceph osd pool application enable cephfs_data cephfs

Check:

    ceph osd pool application get cephfs_metadata
    ceph osd pool application get cephfs_data

Expected Output:

    {
        "cephfs": {}
    }

---

### 9.4 Setting the Number of Replicas

Setting the metadata pool:

    ceph osd pool set cephfsMetadata size 3
    ceph osd pool set cephfs_metadata min_size 2

Setting the data pool:

    ceph osd pool set cephfs_data size 3
    ceph osd pool set cephfs_data min_size 2

Check:

    ceph osd pool get cephfs_metadata size
    ceph osd pool get cephfs_metadata min_size
    ceph osd pool get cephfs_data size
    ceph osd pool get cephfs_data min_size

---

### 9.5 Checking the Pool

    ceph osd pool ls
    ceph df

Confirmation:

    `cephfs_metadata` and `cephfs_data` pools exist.

---

## X. Experiment 2: Creating a CephFS File System

### 10.1 Creating the File System

Execute:

    ceph fs new cephfs cephfs_metadata cephfs_data

Parameter Explanation:

    `cephfs`: Name of the file system.
    `cephfs_metadata`: Metadata pool.
    `cephfs_data`: Data pool.

Check:

    ceph fs ls

Expected Output:

    name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]

---

### 10.2 Checking File System Details

    ceph fs get cephfs

Key Points to Check:

- Name
- Metadata pool
- Data pools
- Max MDS
- Flags
- Standby count wanted
- Session timeout

---

### 10.3 Checking the MDS Status

    ceph mds stat

If the MDS has not been deployed yet, you may not see any active MDS instances.

The MDS needs to be deployed later using `cephadm`.

---

## XI. Experiment 3: Deploying the MDS Service

### 11.1 Using c### 14.3 Mounting Using the Kernel Client

Method One: Mounting using the MON address

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-kernel \
      -o name=admin,secretfile=/etc/ceph/admin.secret,fs=cephfs

To check:

    df -hT | grep ceph

Expected result:

    CephFS is mounted at /mnt/cephfs-kernel

---

### 14.4 Verifying the Mount

    mount | grep ceph
    df -hT /mnt/cephfs-kernel

Writing a test file:

    echo "hello cephfs kernel client" > /mnt/cephfs-kernel/kernel-test.txt

Reading the file:

    cat /mnt/cephfs-kernel/kernel-test.txt

Expected result:

    hello cephfs kernel client

---

### 14.5 Unmounting

    umount /mnt/cephfs-kernel

If it prompts "busy":

    lsof +f -- /mnt/cephfs-kernel
    fuser -vm /mnt/cephfs-kernel

Handle any occupied processes before unmounting.

---

## Chapter Fifteen: Experiment Seven: Mounting CephFS Using ceph-fuse

### 15.1 Creating a Mount Directory

    mkdir -p /mnt/cephfs-fuse

---

### 15.2 Mounting Using ceph-fuse

    ceph-fuse --name client.admin --client_fs cephfs /mnt/cephfs-fuse

To check:

    df -hT | grep ceph-fuse

Or:

    mount | grep ceph-fuse

---

### 15.3 Writing a Test File

    echo "hello cephfs fuse client" > /mnt/cephfs-fuse/fuse-test.txt

Reading the file:

    cat /mnt/cephfs-fuse/fuse-test.txt

Expected result:

    hello cephfs fuse client

---

### 15.4 Unmounting ceph-fuse

    fusermount -u /mnt/cephfs-fuse

If `fusermount` is not available, you can use:

    umount /mnt/cephfs-fuse

---

### 15.5 Comparison between kernel client and ceph-fuse

| Comparison Item | kernel client | ceph-fuse |
|---|---|---|
| Implementation Method | Linux kernel module | User-space FUSE |
| Performance | Usually better | Usually slightly lower |
| Compatibility | Depends on kernel version | More flexible in deployment |
| Troubleshooting Methods | dmesg, kernel logs | ceph-fuse logs |
| Common Use in Production | Frequently used | Also usable, depending on the scenario |

General recommendations:

    For production use, prioritize the kernel client.
    In scenarios requiring compatibility or debugging, consider using ceph-fuse.

---

## Chapter Sixteen: Experiment Eight: Verification of Multi-Client Shared Read/Write

### 16.1 Preparing Two Clients

You can use:

    ceph-client01
    ceph-client02

If you don't have two separate clients, you can temporarily simulate them using `ceph-node01` and `ceph-client`.

Both clients should mount the same CephFS directory:

    /mnt/cephfs

---

### 16.2 Client A Writes Data

On Client A, perform the following operations:

    mkdir -p /mnt/cephfs-kernel/shared-demo

    echo "Write from client A" > /mnt/cephfs-kernel/shared-demo/a.txt

---

### 16.3 Client B Reads Data

On Client B, perform the following operation:

    cat /mnt/cephfs-kernel/shared-demo/a.txt

Expected result:

    The message "Write from client A" should be displayed.

---

### 16.4 Client B Writes Data Again

    echo "Write from client B" > /mnt/cephfs-kernel/shared-demo/b.txt

---

### 16.5 Client A Checks the Results

On Client A, perform the following operations:

    ls -l /mnt/cephfs-kernel/shared-demo/
    cat /mnt/cephfs-kernel/shared-demo/b.txt

Expected result:

    The file `b.txt` should be displayed, and its content should read "Write from client B".

---

### 16.6 Experimental Conclusion

This experiment demonstrates that:

    CephFS can be mounted by multiple clients as a shared file system.
    It is suitable for scenarios where multiple nodes need to share directories.
    In Kubernetes, it corresponds to the ReadWriteMany storage capability.

---

## Chapter Seventeen: Experiment Nine: Creating a User with Minimized Permissions to Mount CephFS

### 1---

### 19.2 Create a Test Directory

    mkdir -p /mnt/cephfs-kernel/snap-demo

Write to the file:

    echo "before cephfs snapshot" > /mnt/cephfs-kernel/snap-demo/file.txt

---

### 19.3 Create a Snapshot

Enter the directory:

    cd /mnt/cephfs-kernel/snap-demo

Create a snapshot:

    mkdir .snap/snap01

View the snapshots:

    ls .snap

Expected output:

    snap01

---

### 19.4 Modify the File

    echo "after cephfs snapshot" > /mnt/cephfs-kernel/snap-demo/file.txt

View the current file:

    cat /mnt/cephfs-kernel/snap-demo/file.txt

Expected output:

    after cephfs snapshot

View the file in the snapshot:

    cat /mnt/cephfs-kernel/snap-demo/.snap/snap01/file.txt

Expected output:

    before cephfs snapshot

---

### 19.5 Delete the Snapshot

    rmdir /mnt/cephfs-kernel/snap-demo/.snap/snap01

Note:

    Snapshots are not independent backups.
    Snapshots still depend on the current Ceph cluster.
    In a production environment, backup and recovery procedures should be combined.

---

## Section Twenty: Experiment Twelve: Clean Up Test Resources

### 20.1 Unmount Client Mount Points

    umount /mnt/cephfs-kernel
    umount /mnt/cephfs-user
    umount /mnt/app01

Unmount ceph-fuse:

    fusermount -u /mnt/cephfs-fuse

If it prompts "busy":

    lsof +f -- /mnt/cephfs-kernel
    fuser -vm /mnt/cephfs-kernel

---

### 20.2 Delete the Test User

Execute on the Ceph management node:

    ceph auth del client.cephfs-user

Verify:

    ceph auth ls | grep cephfs-user

---

### 20.3 Delete the MDS Service

High-risk warning:

    If there are business data in CephFS, do not delete the MDS service.

In a test environment, it can be deleted:

    ceph orch rm mds.cephfs

Verify:

    ceph orch ps --daemon_type mds

---

### 20.4 Delete the CephFS File System

High-risk warning:

    Deleting CephFS will affect the file system.
    Deleting a Pool will erase all data.
    This operation is strictly prohibited in a production environment.

In a test environment, to delete CephFS:

    ceph fs fail cephfs
    ceph fs rm cephfs --yes-i-really-mean-it

Verify:

    ceph fs ls

---

### 20.5 Delete the Pool

To enable Pool deletion:

    ceph config set mon mon_allow_pool_delete true

Delete the Pool:

    ceph osd pool rm cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
    ceph osd pool rm cephfs_data cephfs_data --yes-i-really-really-mean-it

To disable Pool deletion:

    ceph config set mon mon_allow_pool_delete false

Verify:

    ceph osd pool ls

---

## Section Twenty-one: Summary of Common CephFS Commands

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

### 21.4 Create a CephFS

    ceph fs new cephfs cephfsMetadata cephfsData

---

### 21.5 Kernel Mount

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/cephfs-kernel \
      -o name=admin,secretfile=/etc/ceph/admin.secret,fs=cephfs

---

### 21.6 ceph-fuse Mount

    ceph-fuse --name client.admin --🔤 ceph fs status  
🔤 ceph -s  
🔤 dmesg | tail -50  

Confirm:  
- Are both clients mounting the same file system?  
- Are the mount paths identical?  
- Are they mounted into different subdirectories?  
- Do the permissions allow reading?  
- Does the application have any caching mechanisms in place?