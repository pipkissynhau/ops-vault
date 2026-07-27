# Ceph Backup, Recovery, and Disaster Recovery Exercises: Protection of RBD, CephFS, RGW, and Cluster Metadata

Recommended Path: 05-Storage/01-Ceph/16-Ceph Backup, Recovery, and Disaster Recovery Exercises: Protection of RBD, CephFS, RGW, and Cluster Metadata.md

Tags: #Ceph #Backup and Recovery #Disaster Recovery Exercises #RBD #CephFS #RGW #Snapshot #Export #Restore #Disaster Preparedness #SRE #Advanced SRE

---

## I. Document Description

This article is the sixteenth installment of the Ceph Advanced SRE storage module, focusing on methods for Ceph backup, recovery, and disaster recovery exercises.

Previously completed tasks include:

- Ceph cluster deployment
- OSD management
- Pools and PGs
- CRUSH fault domains
- RBD block storage practices
- CephFS file storage practices
- RGW object storage practices
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations and maintenance
- Ceph troubleshooting

This article now covers the data protection phase.

Ceph provides features such as replicas, CRUSH, snapshots, recovery, and high availability, but it is important to understand that:

    Ceph replicas are not backups.
    Ceph snapshots are not complete backups.
    Ceph high availability does not equate to disaster recovery.
    Internal cluster redundancy cannot replace off-site backups.

This article focuses on the following areas:

- Ceph data protection layers
- Cluster metadata backup
- CephX user and configuration backup
- CRUSH Map backup
- RBD Image backup and recovery
- RBD Snapshot and export-diff incremental backup
- CephFS file-level backup and recovery
- CephFS Snapshot restoration
- RGW Bucket object backup and recovery
- Kubernetes PVC data protection strategies
- OSD failure and data recovery boundaries
- Disaster recovery exercise procedures
- Backup strategies for production environments
- Boundaries for high-risk recovery operations

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the differences between Ceph replicas, snapshots, backups, and disaster recovery.
2. Identify which key metadata in a Ceph cluster needs to be backed up.
3. Back up information such as ceph.conf, keyring, auth, CRUSH Map, Pools, Service Spec, etc.
4. Use rbd export to back up RBD Images.
5. Use rbd import to restore RBD Images.
6. Use rbd snapshot for protection before making changes.
7. Use rbd export-diff / import-diff for incremental backup and recovery of RBD.
8. Use CephFS .snap snapshots to restore accidentally deleted files.
9. Use rsync / tar for file-level backups of CephFS.
10. Use AWS CLI for object-level backup and recovery of RGW Buckets.
11. Understand the restoration boundaries between Kubernetes PVCs and RBD Images/CephFS Subvolumes.
12. Distinguish between OSD failure recovery and business data recovery.
13. Design basic disaster recovery exercise procedures.
14. Identify which recovery commands are considered high-risk operations.
15. Establish a closed-loop for backup and recovery in production environments.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article continues to use the same experimental environment as the Ceph module.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Disaster recovery experiments (optional) |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW client testing (optional) |
| 10.0.0.36 | backup-server | Backup server (optional) |

Primary experimental system:

    Ubuntu Server 22.04.5 LTS

Additional system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Backup Directory Planning

It is recommended to prepare a dedicated backup directory:

    /backup/ceph

Subdirectory planning:

    /backup/ceph/config
    /backup/ceph/auth
    /backup/ceph/crush
    /backup/ceph/pool
    /backup/ceph/orch
    /backup/ceph/rbd
    /backup/ceph/cephfs
    /backup| ceph.conf | Ceph Client Configuration |
| ceph.client.admin.keyring | Admin Keyring |
| ceph.pub | cephadm SSH Public Key |
| cephADM-ssh-key | cephadm SSH Private Key |

Security Reminder:

    The keyring and cephADM-ssh-key are sensitive files.
    After backing them up, you must restrict their permissions.
    Do not commit them to a Git repository.
    Do not send them via chat tools.

---

### 7.2 Backing Up Basic Cluster Information

    BACKUP_DATE=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/config/${BACKUP DATE}

    ceph -s > /backup/ceph/config/${BACKUP DATE}/ceph-status.txt
    ceph health detail > /backup/ceph/config/${BACKUP DATE}/ceph-health-detail.txt
    ceph fsid > /backup/ceph/config/${BACKUP DATE}/ceph-fsid.txt
    ceph versions > /backup/ceph/config/${BACKUP DATE}/ceph-versions.txt
    ceph osd tree > /backup/ceph/config/${BACKUP DATE}/ceph-osd-tree.txt
    ceph df > /backup/ceph/config/${BACKUP DATE}/ceph-df.txt
    ceph pg stat > /backup/ceph/config/${BACKUP DATE}/ceph-pg-stat.txt

View:

    ls -lh /backup/ceph/config/${BACKUP DATE}

---

### 7.3 Backing Up the Configuration Database

Export the configuration:

    ceph config dump > /backup/ceph/config/${BACKUP DATE}/ceph-config-dump.txt

View:

    head /backup/ceph/config/${BACKUP DATE}/ceph-config-dump.txt

Explanation:

    The `ceph config dump` command records the cluster configuration set using `ceph config set`. This is very useful for restoring configurations, migrating clusters, and conducting fault analysis.

---

## Section Eight: Experiment Two: Backing Up CephX Users and Keyrings

### 8.1 Exporting All Authentication Information

Execute:

    BACKUP_DATE=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/auth/${BACKUP DATE}

    ceph auth export > /backup/ceph/auth/${BACKUP DATE}/ceph-auth-export.keyring

View:

    ls -lh /backup/ceph/auth/${BACKUP DATE}/ceph-auth-export.keyring

High-Risk Reminder:

    This file contains Ceph user secrets.
    It must be kept secure.
    Do not commit it to Git.
    Do not store it in a shared directory.
    Strict permissions should be set for this file.

Set permissions:

    chmod 600 /backup/ceph/auth/${BACKUP DATE}/ceph-auth-export.keyring

---

### 8.2 Exporting the User List

    ceph auth ls > /backup/ceph/auth/${BACKUP DATE}/ceph-auth-list.txt

View:

    head /backup/ceph/auth/${BACKUP DATE}/ceph-auth-list.txt

---

### 8.3 Backing Up Specific Key Users

For example, to back up a Kubernetes RBD user:

    ceph auth get client.k8s-rbd \
      -o /backup/ceph/auth/${BACKUP DATE}/ceph.client.k8s-rbd.keyring

To back up a CephFS user:

    ceph auth get client.k8s-cephfs \
      -o /backup/ceph/auth/${BACKUP DATE}/ceph.client.k8s-cephfs.keyring

RGW user information is not included in the `ceph auth` command. RGW users need to be exported separately using `radosgw-admin`.

---

## Section Nine: Experiment Three: Backing Up the CRUSH Map

### 9.1 Exporting the CRUSH Map Binary File

    BACKUP_DATE=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/crush/${BACKUP DATE}

    ceph osd getcrushmap -o /backup/ceph/crush/${BACKUP DATE}/crushmap.bin

View:

    ls -lh /backup/ceph/crush/${BACKUP DATE}/crushmap.bin

---

### 9.2 Decompiling the CRUSH Map

You will need `crushtool` for this step.

Execute:

    crushtool -d /backup/ceph/crush/${BACKUP DATE}/crushmap.bin \
      -o /backup/ceph/crush/${BACKUP DATE}/crushmap.txt

View:

    less /backup/ceph/crush/${BACKUP DATE}/crushmap.txt

---

### 9.3 Backing Up the CRUSH Tree

    ceph osd crush tree > /backup/ceph/crush/${BACKUP DATE}/crush-tree.txt
    ceph osd crush rule dump > /backup/ceph/crush/${BACKUP DATE}/crush-rule-dump.json    echo "rbd backup test data" > /mnt/rbd-backup-demo/data.txt
    sync

View:

    cat /mnt/rbd-backup-demo/data.txt

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

### 11.4 Full Export of RBD Image

Create a backup directory:

    mkdir -p /backup/ceph/rbd/$(date +%F)

Perform the export:

    rbd export rbd-backup-demo/test-image \
      /backup/ceph/rbd/$(date +%F)/test-image-full.raw

View:

    ls -lh /backup/ceph/rbd/$(date +%F)/test-image-full.raw

Explanation:

    rbd export is used for a full export.
    The larger the Image, the longer the export time and the more space it will occupy.
    In production, it should be combined with incremental backups, compression, dedicated backup systems, or cross-cluster replication.

---

### 11.5 Restore to a New RBD Image

Restore to the new Image:

    rbd import /backup/ceph/rbd/$(date +%F)/test-image-full.raw \
      rbd-backup-demo/test-image-restore

View:

    rbd ls -p rbd-backup-demo
    rbd info rbd-backup-demo/test-image-restore

---

### 11.6 Verify the Restored Data

Map the restored Image:

    rbd map rbd-backup-demo/test-image-restore

Assume it is mapped to:

    /dev/rbd0

Mount:

    mkdir -p /mnt/rbd-restore-demo
    mount /dev/rbd0 /mnt/rbd-restore-demo

View:

    cat /mnt/rbd-restore-demo/data.txt

Expected result:

    rbd backup test data

Unmount the mounted directory:

    umount /mnt/rbd-restore-demo
    rbd unmap /dev/rbd0

---

## Chapter Twelve: Experiment Six: RBD Snapshots and Rollbacks

### 12.1 Write Data Before Creating a Snapshot

Map the original Image:

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

### 12.2 Create an RBD Snapshot

Create a snapshot:

    rbd snap create rbd-backup-demo/test-image@snap01

View:

    rbd snap ls rbd-backup-demo/test-image

---

### 12.3 Modify Data

Re-map and mount the Image:

    rbd map rbd-backup-demo/test-image
    mount /dev/rbd0 /mnt/rbd-backup-demo

Modify the data:

    echo "after snapshot" > /mnt/rbd-backup-demo/snap-test.txt
    sync

View:

    cat /mnt/rbd-backup-demo/snap-test.txt

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

### 12.4 Roll Back the Snapshot

High-risk warning:

    rbd snap rollback will restore the Image to the state at the time of the snapshot.
    Data created after the snapshot may be lost.
    Before performing this in production, you must stop services, back up data, and confirm the recovery point.

Execute the rollback:

    rbd snap rollback rbd-backup-demo/test-image@snap01

---

### 12.5 Verify the Rollback

Re-map the Image:

    rbd map rbd-backup-demo/test-image
    mount /dev/rbd0 /mnt/rbd-backup-demo

View:

    cat /mnt/rbd-backup-demo/snap-test.txt

Expected result:

    before snapshot

Unmount:

    umount /mnt/rbd-backup-demo
    rbd unmap /dev/rbd0

---

## Chapter Thirteen: Experiment Seven: RBD Incremental Backups and Restores

### 13.1 Basic Concept of Incremental Backups

RBD supports differential exports based on snapshots:

    rbd export-diff
    rbd import-diff

Typical process:

    1. Create a base snapshot, snap-base.
    2. Perform a full backup.
    3. Data changes occur.
    4. Create a new snapshot, snap-001.
    5. Export the difference from snap-base to snap-001.
    6. During restoration, first import the full backup and then the incremental diff.

---

### 13.2 Create a Base Snapshot

    rbdecho "important cephfs file" > /mnt/cephfs-kernel/backup-demo/important.txt

View:

cat /mnt/cephfs-kernel/backup-demo/important.txt

---

### 14.3 Creating CephFS Snapshots

Enter the directory:

cd /mnt/cephfs-kernel/backup-demo

Create a snapshot:

mkdir .snap/snap01

View:

ls .snap

Expected output:

snap01

---

### 14.4 Simulating Accidental Deletion

Delete the file:

rm -f /mnt/cephfs-kernel/backup-demo/important.txt

Verify:

ls -l /mnt/cephfs-kernel/backup-demo/

---

### 14.5 Restoring Files from a Snapshot

View the snapshot content:

cat /mnt/cephfs-kernel/backup-demo/.snap/snap01/important.txt

Restore:

cp /mnt/cephfs-kernel/backup-demo/.snap/snap01/important.txt \
      /mnt/cephfs-kernel/backup-demo/important.txt

Verify:

cat /mnt/cephfs-kernel/backup-demo/important.txt

Expected output:

important cephfs file

---

### 14.6 Deleting the Snapshot

rmdir /mnt/cephfs-kernel/backup-demo/.snap/snap01

Note:

CephFS snapshots are suitable for short-term recovery from accidental deletions.
They are not equivalent to independent backups.
In the event of a cluster-level failure, snapshots may also be affected.

---

## Chapter Fifteen: Experiment Nine: CephFS File-Level Backup and Recovery

### 15.1 Using tar to Back up Directories

Create a backup directory:

mkdir -p /backup/ceph/cephfs/$(date +%F)

Back up:

tar czf /backup/ceph/cephfs/$(date +%F)/backup-demo.tar.gz \
      -C /mnt/cephfs-kernel backup-demo

View:

ls -lh /backup/ceph/cephfs/$(date +%F)/backup-demo.tar.gz

---

### 15.2 Simulating Accidental Deletion of a Directory

Delete the directory:

rm -rf /mnt/cephfs-kernel/backup-demo

Verify:

ls -l /mnt/cephfs-kernel/

---

### 15.3 Restoring from tar

Unzip the backup:

tar xzf /backup/ceph/cephfs/$(date +%F)/backup-demo.tar.gz \
      -C /mnt/cephfs-kernel

View:

ls -l /mnt/cephfs-kernel/backup-demo
cat /mnt/cephfs-kernel/backup-demo/important.txt

---

### 15.4 Using rsync for Backup

Back up to an independent directory:

mkdir -p /backup/ceph/cephfs/rsync-backup/backup-demo

Use rsync for backup:

rsync -avh --delete \
      /mnt/cephfs-kernel/backup-demo/ \
      /backup/ceph/cephfs/rsync-backup/backup-demo/

Restore:

rsync -avh \
      /backup/ceph/cephfs/rsync-backup/backup-demo/ \
      /mnt/cephfs-kernel/backup-demo/

---

### 15.5 Notes on CephFS File-Level Backup

1. tar is suitable for backing up small directories.
2. rsync is useful for incremental synchronization.
3. Backing up a large number of small files can be slow.
4. Changes to files during the backup process may cause inconsistencies.
5. It is not recommended to use simple rsync for database directories; application-level consistency is required.
6. In production environments, it is advisable to combine snapshots with backup windows and business write-stop strategies.

---

## Chapter Sixteen: Experiment Ten: RGW Bucket Object Backup and Recovery

### 16.1 Prerequisites

RGW has been deployed, and the AWS CLI profile ceph-rgw has been configured:

export RGW_ENDPOINT="http://10.0.0.31:7480"

Or use a unified HTTPS endpoint:

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

Upload the files:

aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-backup-demo s34. For production, consider cross-cluster replication of object storage, RGW Multi-Site, or dedicated backup tools.
5. Do not directly delete the underlying Pool of RGW to clean up objects.### 21.5 Failure in RGW s3 Sync

Possible causes:

- Incorrect AccessKey / SecretKey
- Endpoint error
- Bucket does not exist
- Insufficient user permissions
- HTTPS certificate issues
- Time synchronization problems
- Reverse proxy altering the Host or path

Troubleshooting steps:

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls
    radosgw-admin user info --uid=<uid>
    timedatectl
    curl -I ${RGW_endpoint}

---

### 21.6 Is It Secure to Store Backup Files in the Same Ceph Cluster?

It is not secure.

If backup files are still stored within the same Ceph cluster, then:

- In the event of a cluster-level failure, both the source data and backups may become unavailable.
- If a Pool is accidentally deleted, both the original data and backups could be lost.
- Ransomware attacks or operational errors could affect both the source data and backups.

Production recommendations:

- Store backups in separate storage systems.
- Perform off-site backups for critical data.
- Regularly conduct recovery drills for key systems.

---

## Section 22: Backup Strategy Recommendations for Production Environments

### 22.1 RBD Backup Strategy

Recommendations:

- Take snapshots after service interruptions or when application consistency is ensured.
- Periodically perform full rbd export.
- Regularly use export-diff for incremental backups.
- Conduct regular recovery verifications.
- For critical services, combine backup with the database's own backup mechanisms.
- In large-scale scenarios, consider using RBD Mirror or specialized backup systems.

For database-related RBD:

- Do not rely solely on storage snapshots.
- It is essential to combine database backups with binlog/WAL and application consistency strategies.

---

### 22.2 CephFS Backup Strategy

Recommendations:

- Enable snapshotting for important directories.
- Regularly back up these directories to independent storage using rsync.
- Assess backup performance separately for large numbers of small files.
- Define appropriate permission boundaries for business directories.
- Regularly test recovery from accidental deletions.
- Perform off-site backups for critical data.

---

### 22.3 RGW Backup Strategy

Recommendations:

- Organize Buckets based on business requirements.
- Perform cross-end synchronization for important Buckets.
- Use tools such as s3 sync, mc mirror, or specialized backup solutions.
- Develop lifecycle and version control strategies for Buckets.
- Regularly rotate AccessKeys.
- Periodically perform sample object restorations.
- Evaluate the use of RGW Multi-Site in production environments.

---

### 22.4 Backup Strategy for Cluster Metadata

It is recommended to back up regularly:

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
- Listings of RGW users and Buckets
- Lists of Kubernetes StorageClass/PVC/PV/Secret objects

---

## Section 23: Backup Frequency Recommendations

| Data Type | Recommended Frequency | Explanation |
|---|---|---|
| Ceph configuration and auth | Immediately after changes + Weekly | Must be backed up after any modifications. |
| CRUSH Map | Before and after each CRUSH change | High priority. |
| RBD critical Images | Daily incremental backups + Weekly full backup | Depends on the specific business requirements. |
| CephFS shared directories | Daily or more frequently | Depending on data changes. |
| RGW Buckets | Daily or real-time synchronization | Depending on their importance to the business. |
- Kubernetes storage resource YAML files | Immediately after any changes | Facilitates reconstruction in case of need. |
- Recovery drills | Monthly or quarterly | Regular testing is essential to ensure backup effectiveness. |

---

## Section 24: Boundaries for High-Risk Recovery Commands

### 24.1 High-Risk Commands

The following commands may result in data loss:

    ceph osd pool rm
    ceph fs rm
    rbd rm
    rbd snap purge
    rbd snap rollback
    radosgw-admin user rm --purge-data
    radosgw-admin bucket rm --purge-objects
    rm -rf /mnt/cephfs/<business_directory>
    wipefs -a
    sgdisk --zap-all

---

### 24.2 Pre-Execution Confirmation Steps

Before executing any of these commands, it is essential to confirm:

    1. That the target object is correct.
    2. That the operation is being performed in a test environment.
    3. That backups have been made.
    4. Whether service interruptions are required.
    5. That approval has been obtained.
    6.https://docs.ceph.com/en/latest/cephadm/operations/

Ceph Health Checks:

https://docs.ceph.com/en/latest/rados/operations/health-checks/

AWS CLI S3 Command Reference:

https://docs.aws.amazon.com/cli/latest/reference/s3/