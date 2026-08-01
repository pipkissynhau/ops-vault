# Ceph Backup, Recovery, and Disaster Recovery Drills: RBD, CephFS, RGW, and Cluster Metadata Protection

Recommended path: 05-Storage/01-Ceph/16-Ceph Backup, Recovery, and Disaster Recovery Drills: RBD, CephFS, RGW, and Cluster Metadata Protection.md

Tags: #Ceph #BackupRestore #It'sADisaster. #RBD #CephFS #RGW #Snapshot #Export #Restore #DisasterPreparedness #SRE #AdvancedSre

---

## I. Document Overview

This is the sixteenth article in the Ceph Advanced SRE Storage module, focusing on methods for Ceph backup, recovery, and disaster recovery drills.

Previously completed:

- Ceph Cluster Deployment
- OSD Management
- Pool and PG
- CRUSH Failure Domain
- RBD Block Storage Practice
- CephFS File Storage Practice
- RGW Object Storage Practice
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph Daily Operations
- Ceph Troubleshooting

This article enters the data protection phase.

Ceph provides replication, CRUSH, snapshots, recovery, and high availability capabilities, but it must be clearly understood that:

    Ceph replication is not backup.
    Ceph snapshots are not full backups.
    Ceph high availability is not disaster recovery.
    Ceph internal redundancy cannot replaceAlien. backup.

This article covers the following key areas:

- Ceph Data Protection Layers
- Cluster Metadata Backup
- CephX Users and Configuration Backup
- CRUSH Map Backup
- RBD Image Backup and Recovery
- RBD Snapshot and export-diff Incremental Backup
- CephFS File-Level Backup and Recovery
- CephFS Snapshot Recovery
- RGW Bucket Object Backup and Recovery
- Kubernetes PVC Data Protection Approach
- OSD Failure and Data Recovery Boundaries
- Disaster Recovery Drill Process
- Production Environment Backup Strategy
- High-Risk Recovery Operation Boundaries

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the differences between Ceph replication, snapshots, backup, and disaster recovery.
2. Clearly identify which critical metadata needs to be backed up in a Ceph cluster.
3. Backup ceph.conf, keyring, auth, CRUSH Map, Pool, Service Spec, and other information.
4. Use rbd export to backup RBD Image.
5. Use rbd import to recover RBD Image.
6. Use rbd snapshot for change-before-protection.
7. Use rbd export-diff/import-diff for RBD incremental backup and recovery.
8. Use CephFS .snap snapshots to recover mistakenly deleted files.
9. Use rsync/tar for CephFS file-level backup.
10. Use AWS CLI for RGW Bucket object-level backup and recovery.
11. Understand the recovery boundaries for Kubernetes PVC corresponding to RBD Image/CephFS Subvolume.
12. Understand the difference between OSD failure recovery and business data recovery.
13. Design a basic disaster recovery drill process.
14. Clearly identify which recovery commands are high-risk operations.
15. Establish a production environment backup and recovery closed-loop.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article continues using the Ceph module experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Failure Drill, Optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client Testing, Optional |
| 10.0.0.36 | backup-server | Backup Server, Optional |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Backup Directory Planning

It is recommended to prepare an independent backup directory:

    /backup/ceph

Subdirectory planning:

    /backup/ceph/config
    /backup/ceph/auth
    /backup/ceph/crush
    /backup/ceph/pool
    /backup/ceph/orch
    /backup/ceph/rbd
    /backup/ceph/cephfs
    /backup/ceph/rgw
    /backup/ceph/logs

Create directories:

    mkdir -p /backup/ceph/{config,auth,crush,pool,orch,rbd,cephfs,rgw,logs}

Check:

    tree /backup/ceph

If tree is not installed:

    apt install -y tree

---

## IV. Ceph Data Protection Layers

### 4.1 First Layer: Ceph Internal Replication

Example:

    Pool size = 3
    min_size = 2

Purpose:

    Mitigate single OSD, single node, partial failure domain failures.

Cannot resolve:

    Accidental file deletion
    Accidental RBD Image deletion
    Accidental Bucket deletion
    Ransomware encryption
    Cluster-level accidental operations
    Entire cluster damage
    Multiple replicas being incorrectly overwritten simultaneously

---

### 4.2 Second Layer: Snapshots

Example:

    RBD Snapshot
    CephFS Snapshot

Purpose:

    Quickly record the state at a specific moment.
    Suitable for pre-change protection.
    Suitable for short-term rollback.

Cannot resolve:

    Entire Ceph cluster damage
    All OSDs lost
    Pool deleted
    Severe underlying data damage
    Snapshots and source data in the same failure domain

---

### 4.3 Third Layer: Backup

Example:

    rbd export
    rbd export-diff
    rsync CephFS data
    aws s3 sync RGW Bucket
    Backup configuration, auth, CRUSH Map

Purpose:

    Replicate data to an independent location.
    Useful for accidental deletion recovery.
    Useful for migration.
    Useful for disaster recovery.

Requirements:

    Backup data should be placed in a location other than the same Ceph cluster.
    Important data should be stored in independent storage,Alien., or offline media.

---

### 4.4 Fourth Layer: Disaster Recovery

Example: /think

Cross-Cluster Replication  
Disaster Recovery  
RGW Multi-Site  
RBD Mirror  
Business-Level Primary-Secondary  
Scheduled Recovery Drills  

Purpose:  

    To address failures at the data center, cluster, and region levels.  

Advanced SREs must clearly understand:  

    High availability resolves localized failures.  
    Backup resolves accidental deletions and historical recovery.  
    Disaster recovery resolves site-level failures.  
    The three cannot replace each other.  

---

## Five. Pre-Operation Checks  

### 5.1 Check Cluster Status  

    ceph -s  
    ceph health detail  
    ceph osd tree  
    ceph pg stat  
    ceph df  

It is recommended to perform backups when the cluster is healthy.  

If any of the following exist:  

- OSD down  
- PG degraded  
- PG inactive  
- nearfull / full  
- MON quorum anomaly  

You must first determine whether it is suitable to continue with backup or recovery operations.  

---

### 5.2 Check Existing Pools  

    ceph osd pool ls  
    ceph df  

---

### 5.3 Check RBD Images  

Example:  

    rbd ls -p rbd-pool  
    rbd ls -p k8s-rbd  

---

### 5.4 Check CephFS  

    ceph fs ls  
    ceph fs status  
    ceph mds stat  

---

### 5.5 Check RGW Services  

    ceph orch ps --daemon_type rgw  
    radosgw-admin user list  
    radosgw-admin bucket list  

---

## Six. Experiment Task List  

| Experiment | Objective | Risk Level |  
|---|---|---|  
| Experiment 1 | Backup Ceph Cluster Configuration and Metadata | Low |  
| Experiment 2 | Backup CephX Users and Keyring | Medium |  
| Experiment 3 | Backup CRUSH Map | Low |  
| Experiment 4 | Backup cephadm Service Spec | Low |  
| Experiment 5 | RBD Full Backup and Recovery | Medium |  
| Experiment 6 | RBD Snapshots and Rollback | High |  
| Experiment 7 | RBD Incremental Backup and Recovery | Medium-High |  
| Experiment 8 | CephFS Snapshot Recovery of Deleted Files | Medium |  
| Experiment 9 | CephFS File-Level Backup and Recovery | Medium |  
| Experiment 10 | RGW Bucket Object Backup and Recovery | Medium |  
| Experiment 11 | Kubernetes PVC Data Protection Strategy | Medium-High |  
| Experiment 12 | Disaster Recovery Drill Process Design | Medium-High |  
| Experiment 13 | Clean Up Test Resources | High |  

High-Risk Warning:  

    rbd rm, rbd snap purge, ceph osd pool rm, ceph fs rm, radosgw-admin user rm --purge-data may all cause data loss.  
    Recovery operations must first be validated in a test environment.  
    Before production recovery, confirm that backups are valid, recovery targets are correct, and business operations have stopped or entered maintenance mode.  

---

## Seven. Experiment 1: Backup Ceph Cluster Configuration and Metadata  

### 7.1 Backup /etc/ceph  

Execute on the Ceph management node:  

    mkdir -p /backup/ceph/config/$(date +%F)  

    cp -a /etc/ceph /backup/ceph/config/$(date +%F)/  

Check:  

    ls -l /backup/ceph/config/$(date +%F)/ceph  

Key files:  

| File | Purpose |  
|---|---|  
| ceph.conf | Ceph client configuration |  
| ceph.client.admin.keyring | admin keyring |  
| ceph.pub | cephadm SSH public key |  
| cephadm-ssh-key | cephadm SSH private key |  

Security Reminder:  

    keyring and cephadm-ssh-key are sensitive files.  
    After backup, permissions must be restricted.  
    Do not commit to Git repositories.  
    Do not share in chat tools.  

---

### 7.2 Backup Cluster Base Information  

    BACKUP_DATE=$(date +%F-%H%M%S)  
    mkdir -p /backup/ceph/config/${BACKUP_DATE}  

    ceph -s > /backup/ceph/config/${BACKUP_DATE}/ceph-status.txt  
    ceph health detail > /backup/ceph/config/${BACKUP_DATE}/ceph-health-detail.txt  
    ceph fsid > /backup/ceph/config/${BACKUP_DATE}/ceph-fsid.txt  
    ceph versions > /backup/ceph/config/${BACKUP_DATE}/ceph-versions.txt  
    ceph osd tree > /backup/ceph/config/${BACKUP_DATE}/ceph-osd-tree.txt  
    ceph df > /backup/ceph/config/${BACKUP_DATE}/ceph-df.txt  
    ceph pg stat > /backup/ceph/config/${BACKUP_DATE}/ceph-pg-stat.txt  

Check:  

    ls -lh /backup/ceph/config/${BACKUP_DATE}  

---

### 7.3 Backup Configuration Database  

Export configuration:  

    ceph config dump > /backup/ceph/config/${BACKUP_DATE}/ceph-config-dump.txt  

Check:  

    head /backup/ceph/config/${BACKUP_DATE}/ceph-config-dump.txt  

Note:  

    ceph config dump can record cluster configurations set via ceph config set.  
    This is helpful for configuration recovery, cluster migration, and failure analysis.  

---

## Eight. Experiment 2: Backup CephX Users and Keyring  

### 8.1 Export All Authentication Information  

Execute:  

    BACKUP_DATE=$(date +%F-%H%M%S)  
    mkdir -p /backup/ceph/auth/${BACKUP_DATE}  

    ceph auth export > /backup/ceph/auth/${BACKUP_DATE}/ceph-auth-export.keyring  

Check: /backup/ceph/auth/${BACKUP_DATE}

ls -lh /backup/ceph/auth/${BACKUP_DATE}/ceph-auth-export.keyring

High-risk warning:

    This file contains Ceph user keys.
    Must be protected properly.
    Do not commit to Git.
    Do not place in regular shared directories.
    Should set strict permissions.

Setting permissions:

    chmod 600 /backup/ceph/auth/${BACKUP_DATE}/ceph-auth-export.keyring

---

### 8.2 Exporting User List

    ceph auth ls > /backup/ceph/auth/${BACKUP_DATE}/ceph-auth-list.txt

Viewing:

    head /backup/ceph/auth/${BACKUP_DATE}/ceph-auth-list.txt

---

### 8.3 Backing up Key Users Individually

Example: Backing up Kubernetes RBD user:

    ceph auth get client.k8s-rbd \
      -o /backup/ceph/auth/${BACKUP_DATE}/ceph.client.k8s-rbd.keyring

Backing up CephFS user:

    ceph auth get client.k8s-cephfs \
      -o /backup/ceph/auth/${BACKUP_DATE}/ceph.client.k8s-cephfs.keyring

RGW user information is not in ceph auth, RGW users need to be exported separately using radosgw-admin.

---

## NineI don't know.Experiment Three: Backing up CRUSH Map

### 9.1 Exporting CRUSH Map Binary File

    BACKUP_DATE=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/crush/${BACKUP_DATE}

    ceph osd getcrushmap -o /backup/ceph/crush/${BACKUP_DATE}/crushmap.bin

Viewing:

    ls -lh /backup/ceph/crush/${BACKUP_DATE}/crushmap.bin

---

### 9.2 Decoding CRUSH Map

Requires crushtool.

Execution:

    crushtool -d /backup/ceph/crush/${BACKUP_DATE}/crushmap.bin \
      -o /backup/ceph/crush/${BACKUP_DATE}/crushmap.txt

Viewing:

    less /backup/ceph/crush/${BACKUP_DATE}/crushmap.txt

---

### 9.3 Backing up CRUSH Tree

    ceph osd crush tree > /backup/ceph/crush/${BACKUP_DATE}/crush-tree.txt
    ceph osd crush rule dump > /backup/ceph/crush/${BACKUP_DATE}/crush-rule-dump.json

---

### 9.4 Significance of CRUSH Backup

CRUSH Map backup can be used for:

- Recording fault domain design
- Recording rack / host / osd hierarchy
- Recording rule
- Documentation before changes
- Comparison after accidental operations
- Fault analysis

Production recommendations:

    Must back up before any CRUSH modification.
    Must back up again after modification.
    Should record reasons and timestamps for changes before and after backup.

---

## TenI don't know.Experiment Four: Backing up cephadm Service Spec

### 10.1 Exporting Service List

    BACKUP_DATE=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/orch/${BACKUP_DATE}

    ceph orch ls > /backup/ceph/orch/${BACKUP_DATE}/ceph-orch-ls.txt
    ceph orch ps > /backup/ceph/orch/${BACKUP_DATE}/ceph-orch-ps.txt

---

### 10.2 Exporting Service Spec

cephadm-managed services can be recorded by exporting spec:

    ceph orch ls --export > /backup/ceph/orch/${BACKUP_DATE}/ceph-orch-service-spec.yaml

Viewing:

    less /backup/ceph/orch/${BACKUP_DATE}/ceph-orch-service-spec.yaml

Content may include:

- mon
- mgr
- osd
- mds
- rgw
- prometheus
- grafana
- alertmanager
- node-exporter

Notes:

    This file has reference value for reconstructing service orchestration status.
    Not equal to full data backup.
    Needs to be combined with /etc/ceph, auth, CRUSH, and Pool information for saving.

---

## ElevenI don't know.Experiment Five: Full RBD Backup and Recovery

### 11.1 Creating Test RBD Pool

If rbd-pool already exists, it can be reused.

Creation:

    ceph osd pool create rbd-backup-demo 32
    ceph osd pool application enable rbd-backup-demo rbd
    ceph osd pool set rbd-backup-demo size 3
    ceph osd pool set rbd-backup-demo min_size 2
    rbd pool init rbd-backup-demo

---

### 11.2 Creating Test Image

    rbd create test-image --size 1G -p rbd-backup-demo

Viewing:

    rbd ls -p rbd-backup-demo
    rbd info rbd-backup-demo/test-image

---

### 11.3 Writing Test Data

Map on client or Ceph node:

    rbd map rbd-backup-demo/test-image

Assuming mapped to:

    /dev/rbd0

Formatting:

    mkfs.xfs /dev/rbd0

Mounting:

    mkdir -p /mnt/rbd-backup-demo
    mount /dev/rbd0 /mnt/rbd-backup-demo

Writing: /mnt/rbd-backup-demo

```
echo "rbd backup test data" > /mnt/rbd-backup-demo/data.txt
sync

View:

    cat /mnt/rbd-backup-demo/data.txt

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

### 11.4 Full Export of RBD Image

Create backup directory:

    mkdir -p /backup/ceph/rbd/$(date +%F)

Execute export:

    rbd export rbd-backup-demo/test-image \
      /backup/ceph/rbd/$(date +%F)/test-image-full.raw

View:

    ls -lh /backup/ceph/rbd/$(date +%F)/test-image-full.raw

Notes:

    rbd export is a full export.
    Larger images take longer to export and consume more space.
    In production, combine with incremental backups, compression, dedicated backup systems, or cross-cluster replication.

---

### 11.5 Restore to a New RBD Image

Restore to a new Image:

    rbd import /backup/ceph/rbd/$(date +%F)/test-image-full.raw \
      rbd-backup-demo/test-image-restore

View:

    rbd ls -p rbd-backup-demo
    rbd info rbd-backup-demo/test-image-restore

---

### 11.6 Verify Restored Data

Map the restored Image:

    rbd map rbd-backup-demo/test-image-restore

Assume mapped to:

    /dev/rbd0

Mount:

    mkdir -p /mnt/rbd-restore-demo
    mount /dev/rbd0 /mnt/rbd-restore-demo

View:

    cat /mnt/rbd-restore-demo/data.txt

Expected output:

    rbd backup test data

Clean up mount:

    umount /mnt/rbd-restore-demo
    rbd unmap /dev/rbd0

---

## TwelveI don't know.Experiment Six: RBD Snapshots and Rollback

### 12.1 Write Data Before Creating Snapshot

Map original Image:

    rbd map rbd-backup-demo/test-image

Mount:

    mount /dev/rbd0 /mnt/rbd-backup-demo

Write data:

    echo "before snapshot" > /mnt/rbd-backup-demo/snap-test.txt
    sync

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

### 12.2 Create RBD Snapshot

Create snapshot:

    rbd snap create rbd-backup-demo/test-image@snap01

View:

    rbd snap ls rbd-backup-demo/test-image

---

### 12.3 Modify Data

Remap and mount:

    rbd map rbd-backup-demo/test-image
    mount /dev/rbd0 /mnt/rbd-backup-demo

Modify data:

    echo "after snapshot" > /mnt/rbd-backup-demo/snap-test.txt
    sync

View:

    cat /mnt/rbd-backup-demo/snap-test.txt

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

### 12.4 Rollback Snapshot

High-risk warning:

    rbd snap rollback will roll back the Image to the snapshot moment.
    Data after the snapshot may be lost.
    Must stop business operations, make backups, and confirm the recovery point before production execution.

Execute:

    rbd snap rollback rbd-backup-demo/test-image@snap01

---

### 12.5 Verify Rollback

Remap:

    rbd map rbd-backup-demo/test-image
    mount /dev/rbd0 /mnt/rbd-backup-demo

View:

    cat /mnt/rbd-backup-demo/snap-test.txt

Expected output:

    before snapshot

Cleanup:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

## ThirteenI don't know.Experiment Seven: RBD Incremental Backup and Restore

### 13.1 Basic Concept of Incremental Backup

RBD supports differential export based on snapshots:

    rbd export-diff
    rbd import-diff

Typical workflow:

    1. Create a base snapshot snap-base.
    2. Perform a full backup.
    3. Data changes.
    4. Create a new snapshot snap-001.
    5. Export the differential between snap-base and snap-001.
    6. Restore by first importing the full backup, then the incremental diff.

---

### 13.2 Create Base Snapshot

    rbd snap create rbd-backup-demo/test-image@snap-base

Full export:

    mkdir -p /backup/ceph/rbd/diff-demo

    rbd export rbd-backup-demo/test-image \
      /backup/ceph/rbd/diff-demo/test-image-base.raw

---

### 13.3 Modify Data and Create New Snapshot

Map and mount:

    rbd map rbd-backup-demo/test-image
    mount /dev/rbd0 /mnt/rbd-backup-demo

Write new data:

    echo "incremental data 001" > /mnt/rbd-backup-demo/incremental-001.txt
    sync

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0
```

Create a New Snapshot:

    rbd snap create rbd-backup-demo/test-image@snap-001

---

### 13.4 Export Incremental Diff

Execute:

    rbd export-diff \
      --from-snap snap-base \
      rbd-backup-demo/test-image@snap-001 \
      /backup/ceph/rbd/diff-demo/test-image-snap-base-to-snap-001.diff

View:

    ls -lh /backup/ceph/rbd/diff-demo/

---

### 13.5 Restore Full Base Image

Import base backup as new Image:

    rbd import /backup/ceph/rbd/diff-demo/test-image-base.raw \
      rbd-backup-demo/test-image-diff-restore

View:

    rbd info rbd-backup-demo/test-image-diff-restore

---

### 13.6 Import Incremental Diff

Execute:

    rbd import-diff \
      /backup/ceph/rbd/diff-demo/test-image-snap-base-to-snap-001.diff \
      rbd-backup-demo/test-image-diff-restore

---

### 13.7 Verify Incremental Restore

Map:

    rbd map rbd-backup-demo/test-image-diff-restore

Mount:

    mkdir -p /mnt/rbd-diff-restore
    mount /dev/rbd0 /mnt/rbd-diff-restore

View:

    ls -l /mnt/rbd-diff-restore
    cat /mnt/rbd-diff-restore/incremental-001.txt

Expected:

    incremental data 001

Cleanup:

    umount /mnt/rbd-diff-restore
    rbd unmap /dev/rbd0

---

### 13.8 Notes on Incremental Backup

1. Incremental backups depend on the snapshot chain.
2. The snapshot chain cannot be deleted arbitrarily.
3. Imports must be performed in order during restoration.
4. Full backups should be performed regularly in production environments to avoid excessively long incremental chains.
5. Backup files must be stored in an independent and reliable location.
6. Incremental restoration must be regularly tested.

---

## FourteenI don't know.Experiment 8: CephFS Snapshot Recovery for Accidental File Deletion

### 14.1 Prerequisites

CephFS has been created and mounted to:

    /mnt/cephfs-kernel

If not mounted, refer to Chapter 10 for mounting instructions.

---

### 14.2 Create Test Directory and Files

    mkdir -p /mnt/cephfs-kernel/backup-demo

    echo "important cephfs file" > /mnt/cephfs-kernel/backup-demo/important.txt

View:

    cat /mnt/cephfs-kernel/backup-demo/important.txt

---

### 14.3 Create CephFS Snapshot

Enter directory:

    cd /mnt/cephfs-kernel/backup-demo

Create snapshot:

    mkdir .snap/snap01

View:

    ls .snap

Expected:

    snap01

---

### 14.4 Simulate Accidental Deletion

Delete file:

    rm -f /mnt/cephfs-kernel/backup-demo/important.txt

Confirm:

    ls -l /mnt/cephfs-kernel/backup-demo/

---

### 14.5 Recover File from Snapshot

View snapshot content:

    cat /mnt/cephfs-kernel/backup-demo/.snap/snap01/important.txt

Restore:

    cp /mnt/cephfs-kernel/backup-demo/.snap/snap01/important.txt \
      /mnt/cephfs-kernel/backup-demo/important.txt

Verify:

    cat /mnt/cephfs-kernel/backup-demo/important.txt

Expected:

    important cephfs file

---

### 14.6 Delete Snapshot

    rmdir /mnt/cephfs-kernel/backup-demo/.snap/snap01

Note:

    CephFS snapshots are suitable for short-term accidental deletion recovery.
    They are not equivalent to independent backups.
    Snapshots may be affected during cluster-level failures.

---

## FifteenI don't know.Experiment 9: CephFS File-Level Backup and Restore

### 15.1 Use tar to Backup Directory

Create backup directory:

    mkdir -p /backup/ceph/cephfs/$(date +%F)

Backup:

    tar czf /backup/ceph/cephfs/$(date +%F)/backup-demo.tar.gz \
      -C /mnt/cephfs-kernel backup-demo

View:

    ls -lh /backup/ceph/cephfs/$(date +%F)/backup-demo.tar.gz

---

### 15.2 Simulate Accidental Deletion of Directory

    rm -rf /mnt/cephfs-kernel/backup-demo

Confirm:

    ls -l /mnt/cephfs-kernel/

---

### 15.3 Restore from tar

    tar xzf /backup/ceph/cephfs/$(date +%F)/backup-demo.tar.gz \
      -C /mnt/cephfs-kernel

View:

    ls -l /mnt/cephfs-kernel/backup-demo
    cat /mnt/cephfs-kernel/backup-demo/important.txt

---

### 15.4 Use rsync for Backup

Backup to independent directory:

    mkdir -p /backup/ceph/cephfs/rsync-backup/backup-demo

rsync -avh --delete \
      /mnt/cephfs-kernel/backup-demo/ \
      /backup/ceph/cephfs/rsync-backup/backup-demo/

Restoration:

    rsync -avh \
      /backup/ceph/cephfs/rsync-backup/backup-demo/ \
      /mnt/cephfs-kernel/backup-demo/

---

### 15.5 CephFS File-Level Backup Considerations

1. tar is suitable for small-scale directory backups.
2. rsync is suitable for incremental synchronization.
3. Backing up a large number of small files will be very slow.
4. File changes during backup may lead to inconsistencies.
5. Database directories should not be simply rsynced; application-level consistency is required.
6. Production environments should combine snapshots, backup windows, and business write-stop strategies.

---

## SixteenI don't know.Experiment Ten: RGW Bucket Object Backup and Restoration

### 16.1 Prerequisites

RGW has been deployed and configured with an AWS CLI profile:

    ceph-rgw

Endpoint example:

    export RGW_ENDPOINT="http://10.0.0.31:7480"

Or a unified HTTPS entry:

    export RGW_ENDPOINT="https://rgw.example.com"

---

### 16.2 Creating a Test Bucket and Objects

Create a Bucket:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 mb s3://backup-demo-bucket

Create test files:

    mkdir -p /tmp/rgw-backup-demo
    echo "rgw object backup test" > /tmp/rgw-backup-demo/object-01.txt
    echo "rgw object backup test 02" > /tmp/rgw-backup-demo/object-02.txt

Upload:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-backup-demo s3://backup-demo-bucket/ --recursive

Check:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://backup-demo-bucket/

---

### 16.3 Backing Up a Bucket to a Local Directory

Create a backup directory:

    mkdir -p /backup/ceph/rgw/backup-demo-bucket

Synchronize:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 sync s3://backup-demo-bucket /backup/ceph/rgw/backup-demo-bucket

Check:

    find /backup/ceph/rgw/backup-demo-bucket -type f -maxdepth 2

---

### 16.4 Simulating Accidental Object Deletion

Delete an object:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://backup-demo-bucket/object-01.txt

Check:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://backup-demo-bucket/

---

### 16.5 Restoring Objects from Local Backup

Restore:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /backup/ceph/rgw/backup-demo-bucket/object-01.txt \
      s3://backup-demo-bucket/object-01.txt

Verify:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://backup-demo-bucket/

Download verification:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp s3://backup-demo-bucket/object-01.txt /tmp/object-01-restore.txt

    cat /tmp/object-01-restore.txt

---

### 16.6 Restoring to a New Bucket

Create a new Bucket:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 mb s3://backup-demo-bucket-restore

Synchronize restoration:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 sync /backup/ceph/rgw/backup-demo-bucket \
      s3://backup-demo-bucket-restore

Check:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://backup-demo-bucket-restore/

---

### 16.7 RGW Backup Considerations

1. S3 sync is object-level backup.
2. Object metadata, ACL, versioning, and other capabilities should be verified based on actual tool support.
3. Large-scale Buckets need to consider synchronization time and listing performance.
4. Production environments may consider cross-cluster replication for object storage, RGW Multi-Site, or dedicated backup tools.
5. Do not directly delete RGWBottom Pools to clean up objects.

---

## SeventeenI don't know.Experiment Eleven: Kubernetes PVC Data Protection Approach

### 17.1 RBD CSI PVC

Kubernetes RBD PVC typically corresponds to:

    Ceph RBD Image

View PVC: /think

kubectl get pvc -A

Check PV:

    kubectl get pv

Find the corresponding VolumeHandle based on the CSI information of the PV.

In Ceph:

    rbd ls -p k8s-rbd
    rbd info k8s-rbd/<image-name>

Protection methods:

- VolumeSnapshot at Kubernetes layer, depends on CSI Snapshot component
- Snapshot of RBD Image at Ceph layer
- rbd export / export-diff for backup
- Application layer database backup

Note:

    PVCs for database cannot only rely on storage snapshots.
    Need to consider application consistency.

---

### 17.2 CephFS CSI PVC

Kubernetes CephFS PVC typically corresponds to:

    CephFS Subvolume

Check:

    ceph fs subvolume ls cephfs --group_name csi
    ceph fs subvolume info cephfs <subvolume-name> --group_name csi

Protection methods:

- CephFS Subvolume Snapshot
- CephFS file-level backup
- rsync / tar
- Application layer backup

---

### 17.3 Kubernetes Deletion Risks

If StorageClass uses:

    reclaimPolicy: Delete

Deleting PVC may result in deletion of underlying RBD Image or CephFS Subvolume.

Production recommendations:

    Critical business should use Retain or controlled deletion process.
    Must confirm data retention needs before deleting PVC.
    Establish PVC deletion approval process.
    Take snapshots or backups for important PVCs.

---

## EighteenI don't know.Experiment Twelve: Disaster Recovery Drill Process Design

### 18.1 Disaster Recovery Drills Are Not Direct Production Destruction

Disaster recovery drills should be phased:

    1. Document walkthrough
    2. Test environment drill
    3. Pre-production drill
    4. Production only performs low-risk switch verification
    5. Regular review and improvement

Do not directly simulate high-risk deletions in production environment.

---

### 18.2 RBD Recovery Drill

Drill objectives:

    Backup an RBD Image.
    Delete test Image.
    Restore from backup to new Image.
    Mount and verify data.

Verification items:

    rbd info
    Mount successful
    Files readable
    Application startup

---

### 18.3 CephFS Recovery Drill

Drill objectives:

    Restore deleted files using CephFS snapshot.
    Restore directories using tar / rsync.
    Verify permissions and file content.

Verification items:

    File restoration
    Correct permissions
    Multi-client visibility
    Application accessibility

---

### 18.4 RGW Recovery Drill

Drill objectives:

    Backup Bucket to local or another object storage.
    Delete test objects.
    Restore objects from backup.
    Verify object content using S3 API.

Verification items:

    Bucket listing available
    Object downloadable
    Content consistency
    Application access normal

---

### 18.5 Cluster Metadata Recovery Drill

Drill objectives:

    Verify configuration backup availability.
    Verify auth backup readability.
    Verify CRUSH Map backup readability.
    Verify Service Spec usability for reference reconstruction.

Note:

    Not recommended to perform CRUSH Map re-injection, MON store reconstruction, etc. in production or core test clusters during initial stages.
    These are advanced disaster recovery operations requiring separate drills and strict documentation.

---

## NineteenI don't know.Recovery Verification Checklist

### 19.1 RBD Recovery Verification

| Verification Item | Command |
|---|---|
| Image exists | rbd ls -p <pool> |
| Image information | rbd info <pool>/<image> |
| Mappable | rbd map <pool>/<image> |
| Mountable | mount /dev/rbdX <mount-point> |
| Files exist | ls / cat |
| Data consistency | checksum / application verification |

---

### 19.2 CephFS Recovery Verification

| Verification Item | Command |
|---|---|
| CephFS normal | ceph fs status |
| MDS normal | ceph mds stat |
| Directory exists | ls |
| Files readable | cat |
| Multi-client visibility | Multi-node read |
| Correct permissions | ls -l / getfacl |

---

### 19.3 RGW Recovery Verification

| Verification Item | Command |
|---|---|
| Bucket exists | aws s3 ls |
| Object exists | aws s3 ls s3://bucket |
| Downloadable | aws s3 cp |
| Content consistency | cat / md5sum |
| User permissions normal | radosgw-admin user info |

---

### 19.4 Cluster Status Verification

After recovery, must check:

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree
    ceph df

Objectives:

    PG active+clean
    No new abnormal alerts
    Normal capacity water level
    Service status normal

---

## TwentyI don't know.Experiment Thirteen: Clean Up Test Resources

### 20.1 Clean Up RBD Test Resources

Check:

    rbd ls -p rbd-backup-demo
    rbd snap ls rbd-backup-demo/test-image

Delete snapshots:

    rbd snap purge rbd-backup-demo/test-image

Delete Image:

    rbd rm rbd-backup-demo/test-image
    rbd rm rbd-backup-demo/test-image-restore
    rbd rm rbd-backup-demo/test-image-diff-restore

Delete Pool:

    ceph config set mon mon_allow_pool_delete true

    ceph osd pool rm rbd-backup-demo rbd-backup-demo --yes-i-really-really-mean-it

    ceph config set mon mon_allow_pool_delete false

High-risk warning: /think

Deleting a Pool will delete all data within it. Do not execute directly in a production environment.

---

### 20.2 Cleaning CephFS Test Directories

    rm -rf /mnt/cephfs-kernel/backup-demo

Note:

    Only delete the test directory.
    Do not delete business directories.

---

### 20.3 Cleaning RGW Test Resources

Delete objects and Bucket:

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://backup-demo-bucket --recursive

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rb s3://backup-demo-bucket

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://backup-demo-bucket-restore --recursive

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rb s3://backup-demo-bucket-restore

---

## Twenty-one, Common Issues and Troubleshooting

### 21.1 Slow rbd export

Possible Causes:

- Image is very large
- OSD disk is slow
- Network is slow
- Recovery/backfill is in progress
- Backup target disk is slow
- Client IO limitations

Troubleshooting:

    ceph -s
    ceph osd perf
    ceph osd df
    iostat -x 1
    iftop

---

### 21.2 RBD Cannot Mount After Recovery

Possible Causes:

- File system damage
- Device not properly mapped
- Wrong Image used
- Incomplete recovery files
- Inconsistent snapshot point

Troubleshooting:

    rbd info <pool>/<image>
    rbd map <pool>/<image>
    lsblk
    blkid /dev/rbdX
    dmesg | tail -100

---

### 21.3 Failed RBD Incremental Import

Possible Causes:

- Incremental chain order error
- Base Image mismatch
- Missing snapshot
- Diff file damage
- Target Image status is incorrect

Resolution:

    Confirm full import first.
    Then import-diff in order.
    Check each diff's corresponding snapshot chain.
    Do not skip recovery steps.

---

### 21.4 CephFS Snapshot Directory Unavailable

Possible Causes:

- Current directory does not support snapshots
- CephFS snapshot capability not enabled
- Insufficient permissions
- Path error
- Client mount method differences

Troubleshooting:

    ceph fs get cephfs
    mount | grep ceph
    ls -la <dir>
    dmesg | tail -100

---

### 21.5 RGW s3 sync Failure

Possible Causes:

- AccessKey / SecretKey error
- Endpoint error
- Bucket does not exist
- Insufficient user permissions
- HTTPS certificate issue
- Time synchronization problem
- Reverse proxy rewriting Host or path

Troubleshooting:

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls
    radosgw-admin user info --uid=<uid>
    timedatectl
    curl -I ${RGW_ENDPOINT}

---

### 21.6 Is It Safe to Store Backup Files in the Same Ceph Cluster

Not safe.

If backup files remain in the same Ceph cluster:

    Cluster-level failure may make source data and backups unavailable simultaneously.
    Accidental Pool deletion may result in simultaneous loss.
    Ransomware or accidental operations may affect both source and backups simultaneously.

Production Recommendations:

    Store backups in independent storage.
    PerformAlien. backup for critical data.
    Conduct regular recovery drills for key systems.

---

## Twenty-two, Production Environment Backup Strategy Recommendations

### 22.1 RBD Backup Strategy

Recommendations:

- Take snapshots after business downtime or application consistency
- Perform periodic full rbd export
- Perform periodic export-diff incremental backups
- Regular recovery verification
- Combine with database-specific backups for critical business
- Evaluate RBD Mirror or professional backup systems for large-scale scenarios

Database RBD:

    Cannot rely solely on storage snapshots.
    Must combine with database backups, binlog / WAL, and application consistency policies.

---

### 22.2 CephFS Backup Strategy

Recommendations:

- Enable snapshot policies for important directories
- Regularly rsync to independent backup storage
- Evaluate backup performance separately for large numbers of small files
- Design permission boundaries for business directories
- Regularly test accidental deletion recovery
- PerformAlien. backup for critical data

---

### 22.3 RGW Backup Strategy

Recommendations:

- Divide Buckets by business
- Synchronize important Buckets across sites
- Use s3 sync, mc mirror, or dedicated tools
- Plan lifecycle and version control strategies
- Regularly rotate AccessKeys
- Regularly sample object recovery
- Evaluate RGW Multi-Site in production environments

---

### 22.4 Cluster Metadata Backup Strategy

Recommend to regularly backup:

- /etc/ceph
- ceph auth export
- ceph config dump
- CRUSH Map
- ceph osd tree
- ceph osd pool ls detail
- ceph orch ls --export
- ceph fs dump
- ceph mon dump
- ceph osd dump
- RGW user and Bucket inventory
- Kubernetes StorageClass / PVC / PV / Secret inventory

---

## Twenty-three, Backup Frequency Recommendations

| Data Type | Recommended Frequency | Notes |
|---|---|---|
| Ceph Configuration and auth | After every change + weekly | Must backup after changes |
| CRUSH Map | Before and after every CRUSH change | High priority |
| RBD Critical Image | Daily incremental + weekly full | Depends on business |
| CephFS Shared Directory | Daily or more frequent | Depends on data changes |
| RGW Bucket | Daily or real-time sync | Depends on business importance |
| Kubernetes Storage Resource YAML | After every change | Facilitates reconstruction |
| Recovery Drill | Monthly or quarterly | Cannot backup without recovery |

## 24. High-Risk Recovery Command Boundaries

### 24.1 High-Risk Commands

The following commands may cause data loss:

    ceph osd pool rm
    ceph fs rm
    rbd rm
    rbd snap purge
    rbd snap rollback
    radosgw-admin user rm --purge-data
    radosgw-admin bucket rm --purge-objects
    rm -rf /mnt/cephfs/<business directory>
    wipefs -a
    sgdisk --zap-all

---

### 24.2 Pre-Execution Confirmation

Before execution, the following must be confirmed:

    1. Is the operation target correct?
    2. Is it a test environment?
    3. Is there a backup?
    4. Is business downtime required?
    5. Has it been approved?
    6. Is there a rollback plan?
    7. Is the command recorded?
    8. Has someone reviewed it?

---

## 25. Advanced SRE Methodology

### 25.1 Replication Is Not Backup

Ceph size=3 can prevent single OSD or single-node failure.

But it cannot prevent:

- Human error deletion
- Logical errors
- Application corruption
- Ransomware
- Cluster-level failure
- Operational misoperation

Backup is still required.

---

### 25.2 Snapshots Are Not Backup

Snapshots are suitable for quick rollback, but they typically reside in the same system as the source data.

If the underlying cluster fails, snapshots may also become unavailable.

In production:

    Snapshots are for short-term protection.
    Backups are for long-term retention.
    Disaster recovery is for site-level failure.

---

### 25.3 Backups Must Be Recoverable

Only doing backups without recovery drills is equivalent to having no real verification.

Backups must be regularly validated:

- Can backup files be read?
- Can RBD be imported?
- Can CephFS files be recovered?
- Can RGW objects be downloaded?
- Can applications be started?
- Is data consistent?

---

### 25.4 Recovery Goals Are More Important Than Recovery Commands

Before recovery, the following must be clearly defined:

    Where to recover to?
    Which time point?
    Which business?
    Will original data be overwritten?
    Is recovery to a new environment verified?
    Is business downtime required?
    Who confirms recovery success?

---

### 25.5 Data Protection by Business Tier

Different businesses require different strategies:

| Business Tier | Strategy |
|---|---|
| Temporary Test Data | Can be low-frequency backed up or not backed up |
| Ordinary Business Data | Regular backup |
| Critical Business Data | Snapshot + Backup +Alien. |
| Core Database | Application-level backup + Storage Snapshot +Alien. Disaster Recovery |
| Compliance Data | Long-term retention + Audit + Anti-tampering |

---

## 26. Interview Answer Framework

If asked in an interview:

    Does Ceph need backup if it has 3 replicas?

You can answer:

    Yes. Ceph's 3 replicas mainly address hardware failures and localized failures, such as single OSD or single-node failures, where data remains available through other replicas. However, replicas are not backups and cannot resolve issues like accidental deletion, application corruption, ransomware, pool deletion, or cluster-level failures.
    Ceph snapshots also cannot fully replace backups, as they typically still rely on the same Ceph cluster. If the underlying cluster suffers severe failure, snapshots may also become unavailable.
    In production environments, I would implement layered data protection: the first layer is Ceph replicas and CRUSH fault domains to ensure high availability; the second layer is RBD Snapshots or CephFS Snapshots for short-term quick rollback; the third layer is independent backups, such as rbd export/export-diff, CephFS rsync/tar, RGW Bucket synchronization; the fourth layer isAlien. disaster recovery, such as RBD Mirror, RGW Multi-Site, or business-level primary-secondary.
    Regular recovery drills are also required to verify if backups can truly be restored. Only backing up without recovery drills poses significant risks.

---

## 27. Summary of This Article

This article mainly organizes methods for Ceph backup, recovery, and disaster recovery drills:

1. Ceph replicas are not backups.
2. Ceph snapshots are not complete backups.
3. High availability, backup, and disaster recovery are three different layers.
4. Cluster metadata needs to be backed up: /etc/ceph, auth, config, CRUSH Map, Service Spec.
5. RBD can use rbd export for full backups.
6. RBD can use rbd export-diff / import-diff for incremental backup recovery.
7. RBD Snapshots are suitable for protection before changes, but rollback is a high-risk operation.
8. CephFS can use .snap snapshots to recover accidentally deleted files.
9. CephFS can use tar / rsync for file-level backups.
10. RGW Bucket can use aws s3 sync for object-level backup recovery.
11. Kubernetes PVC must confirm reclaimPolicy and underlying data retention policy before deletion.
12. Backup files should be stored in independent storage and not just in the same Ceph cluster.
13. Production backups must undergo regular recovery drills.
14. Recovery goals, time points, impact scope, and rollback plans must be clearly defined before recovery.
15. Advanced SRE must establish a closed-loop for data protection, recovery verification, and disaster recovery drills.

---

## 28. Reference Documents

Ceph RBD Official Documentation:

    https://docs.ceph.com/en/latest/rbd/

RBD Snapshot Documentation:

    https://docs.ceph.com/en/latest/rbd/rbd-snapshot/

RBD Command Documentation:

    https://docs.ceph.com/en/latest/man/8/rbd/

CephFS Snapshot Documentation:

    https://docs.ceph.com/en/latest/cephfs/snapshots/

CephFS Management Documentation:

    https://docs.ceph.com/en/latest/cephfs/administration/

Ceph RGW Management Documentation:

    https://docs.ceph.com/en/latest/radosgw/admin/

Ceph RGW Multi-Site:

    https://docs.ceph.com/en/latest/radosgw/multisite/

Ceph CRUSH Map Documentation:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

Cephadm Operations Documentation:

    https://docs.ceph.com/en/latest/cephadm/operations/

Ceph Health Check:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

AWS CLI S3 Command Reference:

    https://docs.aws.amazon.com/cli/latest/reference/s3/