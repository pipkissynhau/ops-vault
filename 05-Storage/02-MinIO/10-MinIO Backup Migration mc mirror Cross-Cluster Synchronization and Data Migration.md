# MinIO Backup and Migration: mc mirror, Cross-Cluster Synchronization, and Data Migration

Recommended Path: 05-Storage/02-MinIO/10-MinIO Backup and Migration: mc mirror, Cross-Cluster Synchronization and Data Migration.md

Tags: #MinIO #BackupMigration #mc #mirror #ObjectStorage #S3 #CrossClusterSync #DataMigration #DisasterPreparedness #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the tenth article of the MinIO module, focusing on learning MinIO's backup migration, mc mirror synchronization, cross-cluster migration, and data verification.

Previously completed:

- MinIO Object Storage Basics
- Single Machine Single Disk Deployment
- Single Node Multi-Disk Deployment
- 4-Node Multi-Disk Distributed Cluster Deployment
- Internal HTTP and External HTTPS Access Entry Design
- Nginx HTTPS Unified Entry
- mc Client Configuration, Bucket Management, and Object Operations
- Users, Policies, AccessKey, and SecretKey Permission Management
- Erasure Coding, Node Failure, and Disk Failure Recovery
- Prometheus Metrics, Logs, and Capacity Management

This article enters the critical phase of MinIO production data protection.

This article focuses on solving:

    Does MinIO still need backup after implementing Erasure Coding?
    What can mc mirror do?
    What's the difference between mc mirror and regular mc cp?
    How to synchronize a local directory to MinIO?
    How to back up a MinIO Bucket to local?
    How to synchronize one MinIO cluster to another MinIO cluster?
    How to perform capacity assessment before migration?
    How to verify data after migration?
    Why is mc mirror --remove dangerous?
    Can mc mirror --watch be used as production-grade replication?
    How to choose between mc mirror, Bucket Replication, and Site Replication?
    How to design object storage backup and migration processes in production environments?

This article emphasizes practical operations, providing replicable commands for all core processes.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand that Erasure Coding is not equivalent to backup.
2. Understand the role and limitations of mc mirror.
3. Use mc mirror to synchronize a local directory to a MinIO Bucket.
4. Use mc mirror to back up a MinIO Bucket to a local directory.
5. Use mc mirror to synchronize one MinIO Bucket to another Bucket.
6. Use mc mirror to migrate one MinIO cluster to another MinIO cluster.
7. Use --overwrite to handle object updates in the target endpoint.
8. Understand the risks of --remove.
9. Understand the use cases and risks of --watch.
10. Use --dry-run to simulate synchronization operations.
11. Use --summary to output synchronization summaries.
12. Use --exclude / --include to filter objects.
13. Use --limit-upload / --limit-download to control bandwidth.
14. Verify Bucket capacity and objects before and after migration.
15. Write basic backup scripts.
16. Design MinIO backup and migration processes for production environments.
17. Distinguish the usage boundaries of mc mirror, Bucket Replication, and Site Replication.

---

## III. Experimental Environment

### 3.1 Source MinIO Cluster

The source MinIO cluster continues from the previous 4-node distributed cluster:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Source Cluster Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Source Cluster Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Source Cluster Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Source Cluster Node 4 |
| 10.0.0.46 | minio-entry | Source Cluster Nginx HTTPS Unified Entry |

Source Cluster API Endpoint:

    https://s3.minio.local

Source Cluster Console Endpoint:

    https://console.minio.local

---

### 3.2 Backup MinIO Experimental Node

To demonstrate cross-cluster synchronization, this article starts an additional single-node MinIO on the minio-client node as a "backup cluster".

| IP | Hostname | Purpose | Port |
|---|---|---|---|
| 10.0.0.45 | minio-client | mc Client / Backup MinIO Experimental Node | 19000 / 19001 |

Backup Cluster API:

    http://10.0.0.45:19000

Backup Cluster Console:

    http://10.0.0.45:19001

Notes:

    This backup MinIO is only for experimental purposes.
    Production environment backup clusters should use independent nodes, independent disks, and independent fault domains.
    It is not recommended to place backup and source clusters on the same machine, same disk, or same data center.

---

### 3.3 Image Version

MinIO Server Image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc Client Image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Image Sources:

    minio/minio:RELEASE.2025-04-22T22-12-26Z
    minio/mc:RELEASE.2025-04-16T18-13-26Z

Notes:

    This module uses a fixed version uniformly.
    Does not use latest.
    Ensures the backup migration experiment is reproducible.

---

## IV. Why Backup is Still Needed After Erasure Coding

### 4.1 Problems Solved by Erasure Coding

MinIO Erasure Coding can address:

    Single disk failure
    Partial disk failure
    Single node failure
    Partial node unavailability
    Local hardware damage
    Node short-term offline

It belongs to:

    High availability mechanism
    Data block recovery mechanism
    Disk/node-level fault tolerance mechanism

---

### 4.2 Problems Not Solved by Erasure Coding

Erasure Coding cannot solve:

# 4.2 Potential Data Loss Scenarios

Data loss may occur due to:

- Accidental deletion of objects
- Administrator mistakenly deleting Bucket
- Application overwriting files incorrectly
- Program looping writing incorrect data
- Ransomware encrypting or deleting objects
- Root user leakage leading to malicious deletion
- Entire MinIO cluster damage
- Data center-level failure
- Multi-node failure exceeding tolerance boundary
- Nginx configuration error causing service unavailability
- Policy error causing service unavailability

---

### 4.3 Backup Solutions

Backup is used to address:

- Recovery after accidental deletion
- Recovery after incorrect overwriting
- Recovery after source cluster damage
- Cross-cluster recovery
- Cross-data center recovery
- Migration rollback
- Audit retention
- Archive preservation
- Disaster recovery

Conclusion:

- Erasure Coding is availability.
- Backup is recoverability.
- The two cannot replace each other.

---

## FiveI don't know.mc mirror Basic Understanding

### 5.1 What is mc mirror

mc mirror is the mirror synchronization command in MinIO client.

Its function is similar to:

- rsync

It can be used for:

- Local directory -> MinIO Bucket
- MinIO Bucket -> Local directory
- MinIO Bucket -> MinIO Bucket
- MinIO Cluster A -> MinIO Cluster B
- MinIO -> Other S3-compatible storage
- Other S3-compatible storage -> MinIO

---

### 5.2 Difference between mc mirror and mc cp

| Command | Suitable Scenario |
|---|---|
| mc cp | Upload/download single file or small number of objects |
| mc cp --recursive | Recursive copy directory or Prefix |
| mc mirror | Continuous or batch mirror synchronization between source and target |
| mc mirror --watch | Monitor source changes and continuous synchronization |
| mc mirror --remove | Delete objects in target that no longer exist in source, high risk |
| mc mirror --overwrite | Overwrite target objects when source objects are updated |

---

### 5.3 Common mc mirror Parameters

| Parameter | Function | Risk |
|---|---|---|
| --dry-run | Preview synchronization, no actual execution | Low |
| --overwrite | Allow overwrite target objects when existing | Medium |
| --remove | Delete target objects that no longer exist in source | High |
| --watch | Continuous synchronization monitoring | Medium |
| --summary | Output synchronization summary | Low |
| --exclude | Exclude matching objects | Medium |
| --include | Only include matching objects | Medium |
| --limit-upload | Limit upload bandwidth | Low |
| --limit-download | Limit download bandwidth | Low |
| --preserve | Try to preserve some attributes | Low |
| --skip-errors | Skip errors and continue synchronization | Medium |
| --max-workers | Control concurrent worker count | Medium |

Production reminder:

- --remove is a high-risk parameter.
- --watch is not equivalent to a complete disaster recovery system.
- --overwrite will overwrite target objects with the same name.
- Perform --dry-run before large-scale synchronization.
- Must confirm source and target are not reversed before synchronization.

---

## SixI don't know.Experiment Tasks in This Article

This article will complete the following experiments:

| Experiment | Content |
|---|---|
| Experiment 1 | Prepare mc management environment |
| Experiment 2 | Start backup MinIO experimental instance |
| Experiment 3 | Configure source cluster and backup cluster alias |
| Experiment 4 | Create source Bucket and target Bucket |
| Experiment 5 | Synchronize local directory to source MinIO |
| Experiment 6 | Backup source MinIO Bucket to local |
| Experiment 7 | Synchronize source Bucket to backup Bucket |
| Experiment 8 | Use --dry-run to preview synchronization |
| Experiment 9 | Use --overwrite to handle object updates |
| Experiment 10 | Demonstrate high-risk deletion synchronization with --remove |
| Experiment 11 | Use include/exclude to filter synchronization |
| Experiment 12 | Use bandwidth limits to control synchronization impact |
| Experiment 13 | Write basic backup script |
| Experiment 14 | Data verification after migration |
| Experiment 15 | Design production backup migration process |

---

## SevenI don't know.Experiment 1: Prepare mc Management Environment

### 7.1 Create Directories

Execute on minio-client node:

    mkdir -p /data/minio/mc-config
    mkdir -p /tmp/minio-mirror-demo/source
    mkdir -p /tmp/minio-mirror-demo/restore
    mkdir -p /tmp/minio-mirror-demo/reports
    mkdir -p /tmp/minio-mirror-demo/scripts

---

### 7.2 Pull Image

Pull mc:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Pull MinIO server image:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Check:

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq

---

### 7.3 Configure Command Variables

Define in current shell:

    export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    export MC_CONFIG=/data/minio/mc-config
    export MC_WORKDIR=/tmp/minio-mirror-demo

    mcx() {
      docker run --rm \
        -v ${MC_CONFIG}:/root/.mc \
        -v ${MC_WORKDIR}:/demo \
        ${MC_IMAGE} "$@"
    }

Note:

- This function is only valid in the current shell session.
- Needs to be redefined after reopening terminal.
- Production scripts recommend writing full commands or encapsulating fixed scripts.

---

### 7.4 Check mc Version

docker run --rm \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --version

---

## 8. Experiment Two: Launch Backup MinIO Instance

### 8.1 Create Backup Data Directory

Execute on minio-client node:

    mkdir -p /data/minio-backup/data

Check:

    ls -ld /data/minio-backup/data

---

### 8.2 Start Backup MinIO

Execute:

    docker run -d \
      --name minio-backup \
      --restart unless-stopped \
      -p 19000:9000 \
      -p 19001:9001 \
      -e MINIO_ROOT_USER=backupadmin \
      -e MINIO_ROOT_PASSWORD='BackupAdmin@123456' \
      -v /data/minio-backup/data:/data \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server /data --console-address ":9001"

---

### 8.3 Check Backup MinIO Status

    docker ps | grep minio-backup

Check logs:

    docker logs --tail=100 minio-backup

Check API:

    curl -I http://10.0.0.45:19000/minio/health/live
    curl -I http://10.0.0.45:19000/minio/health/ready

Console access:

    http://10.0.0.45:19001

Login:

    Username: backupadmin
    Password: BackupAdmin@123456

---

### 8.4 Production Notes

Current backup MinIO is a single-node experimental instance.

Production backup cluster should meet:

    Independent node
    Independent disk
    Independent data center or fault domain
    Independent account permissions
    Independent monitoring alerts
    Independent backup strategy
    Not share storage with source cluster
    Not use same root keys as source cluster

---

## 9. Experiment Three: Configure Source Cluster and Backup Cluster alias

### 9.1 Configure Source Cluster alias

If source cluster uses self-signed certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-src https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using official trusted certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-src https://s3.minio.local minioadmin 'MinioAdmin@123456'

---

### 9.2 Configure Backup Cluster alias

Backup cluster uses HTTP experimental endpoint:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-bak http://10.0.0.45:19000 backupadmin 'BackupAdmin@123456'

---

### 9.3 Check alias

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

---

### 9.4 Check Status of Both Clusters

Check source cluster:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-src

Check backup cluster:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-bak

---

## 10. Experiment Four: Create Source Bucket and Target Bucket

### 10.1 Create Source Bucket

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-src/mirror-src-demo

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-src

---

### 10.2 Create Target Bucket

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mb minio-bak/mirror-bak-demo

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls minio-bak

---

## Eleven, Experiment Five: Synchronizing Local Directory to Source MinIO

### 11.1 Prepare Local Source Directory

Create directory:

mkdir -p /tmp/minio-mirror-demo/source/app
mkdir -p /tmp/minio-mirror-demo/source/logs/nginx/2026/04/28
mkdir -p /tmp/minio-mirror-demo/source/backup/mysql
mkdir -p /tmp/minio-mirror-demo/source/tmp

Create file:

echo "app config v1" > /tmp/minio-mirror-demo/source/app/config.txt
echo "nginx access log line 1" > /tmp/minio-mirror-demo/source/logs/nginx/2026/04/28/access.log
echo "mysql backup demo" > /tmp/minio-mirror-demo/source/backup/mysql/mysql-full.sql
echo "temporary file should be excluded later" > /tmp/minio-mirror-demo/source/tmp/temp.txt

Create large file:

dd if=/dev/zero of=/tmp/minio-mirror-demo/source/backup/mysql/mysql-full-100m.sql bs=1M count=100

View:

tree /tmp/minio-mirror-demo/source
du -sh /tmp/minio-mirror-demo/source

---

### 11.2 Dry-run Local Directory Synchronization

Use --dry-run for simulation:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --dry-run /demo/source/ minio-src/mirror-src-demo/

Note:

--dry-run does not perform actual synchronization.
Used to confirm source and target correctness.
It is recommended to perform a dry-run before production mirror execution.
Especially before using --remove, --overwrite, must first simulate.

---

### 11.3 Execute Local Directory Synchronization

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --summary /demo/source/ minio-src/mirror-src-demo/

---

### 11.4 View Source Bucket Objects

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-src/mirror-src-demo

View capacity:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure du minio-src/mirror-src-demo

---

## Twelve, Experiment Six: Backup Source MinIO Bucket to Local Directory

### 12.1 Create Local Restore Directory

mkdir -p /tmp/minio-mirror-demo/restore/from-src

---

### 12.2 Dry-run Bucket to Local Synchronization

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --dry-run minio-src/mirror-src-demo/ /demo/restore/from-src/

---

### 12.3 Execute Bucket to Local Synchronization

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --summary minio-src/mirror-src-demo/ /demo/restore/from-src/

---

### 12.4 Viewing Locally Recovered Data

    tree /tmp/minio-mirror-demo/restore/from-src
    du -sh /tmp/minio-mirror-demo/restore/from-src

Viewing file contents:

    cat /tmp/minio-mirror-demo/restore/from-src/app/config.txt
    cat /tmp/minio-mirror-demo/restore/from-src/logs/nginx/2026/04/28/access.log

---

### 12.5 Use Cases for Local Backup

Local backup is suitable for:

    Small-scale Bucket
    Temporary migration
    Manual export
    Offline backup
    Data sampling recovery
    Backup before migration

Not suitable for:

    Extremely large production Bucket
    Long-term unique backup
    High-concurrency continuous synchronization
    Unified disaster recovery for multiple businesses

Recommended in production:

    MinIO Cluster to Independent MinIO Cluster
    MinIO to Cloud Provider Object Storage
    Bucket Replication
    Site Replication
    Combined with backup task monitoring and recovery drills

---

## ThirteenI don't know.Experiment Seven: Source Bucket Synchronization to Backup Bucket

### 13.1 Pre-visualization of Cross-Cluster Synchronization

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --dry-run minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

Explanation:

    Source: minio-src/mirror-src-demo/
    Target: minio-bak/mirror-bak-demo/
    Must confirm source and target are not reversed.

---

### 13.2 Execute Cross-Cluster Synchronization

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

---

### 13.3 View Target Bucket

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo

View capacity:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      du minio-bak/mirror-bak-demo

---

### 13.4 Download Target Objects for Verification

    mkdir -p /tmp/minio-mirror-demo/restore/from-backup

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-bak/mirror-bak-demo/app/config.txt /demo/restore/from-backup/config.txt

View:

    cat /tmp/minio-mirror-demo/restore/from-backup/config.txt

Expected:

    app config v1

---

## FourteenI don't know.Experiment Eight: Handling Object Updates with --overwrite

### 14.1 Modify Source Files

Modify local source files:

    echo "app config v2" > /tmp/minio-mirror-demo/source/app/config.txt

Sync local changes to source Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --overwrite --summary /demo/source/ minio-src/mirror-src-demo/

View source content:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cat minio-src/mirror-src-demo/app/config.txt

### 14.2 Preview Overwrite Sync to Backup Cluster

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --overwrite --dry-run minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

---

### 14.3 Execute Overwrite Sync

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --overwrite --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

Check backup content:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cat minio-bak/mirror-bak-demo/app/config.txt

Expected:

    app config v2

---

### 14.4 --overwrite Explanation

--overwrite indicates:

    Allows overwriting target-side objects with the same name if the source-side object content changes.

Risks:

    Target-side historical content will be overwritten.
    If erroneous files are synchronized from the source-side, the target-side will also become erroneous files.
    Restoring old versions will be difficult without version control or independent backups.

Production recommendations:

    Evaluate enabling version control for critical Buckets.
    Retain critical data snapshots or archives before synchronization.
    Do not treat --overwrite as a risk-free parameter.
    Perform --dry-run before use.
    Exercise caution during business change periods.

---

## FifteenI don't know.Experiment Nine: Demonstrate High-Risk Delete Sync Using --remove

### 15.1 --remove Function

--remove indicates:

    If an object exists on the target-side but no longer exists on the source-side, delete the object from the target-side.

This makes the target-side more strictly aligned with the source-side.

High risk:

    If objects are mistakenly deleted on the source-side, the target-side will also be deleted.
    If the source-side path is written incorrectly, it may lead to deletion of many objects on the target-side.
    If source and target are reversed, it may cause severe data loss.
    Recovery will be difficult without version control or backups.

---

### 15.2 Create Extra Objects on Target-Side

Create an object that exists only on the target-side:

    echo "target only object" > /tmp/minio-mirror-demo/target-only.txt

Upload to the target-side:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/target-only.txt minio-bak/mirror-bak-demo/target-only.txt

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo

---

### 15.3 Observe --remove with dry-run First

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --remove --dry-run minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

Observe whether target-only.txt will be deleted in the output.

---

### 15.4 Execute --remove

High-risk warning:

    The following command will delete objects on the target-side that no longer exist on the source-side.
    Only allowed to execute in experimental Buckets.
    Must be approved and reviewed in production environments.

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --remove --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

Check target-side:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo

Expected:

    target-only.txt is deleted.

---

### 15.5 Principles for Using --remove in Production

Before using --remove in production, must confirm:

    Whether the source-side is correct.
    Whether the target-side is correct.
    Whether it's a production Bucket.
    Whether there is a risk of accidental deletion.
    Whether version control exists.
    Whether independent backups exist.
    Whether --dry-run has been performed.
    Whether approval exists.
    Whether dual-person review exists.
    Whether rollback plans exist.

Recommendation: /think

Daily backups should not use --remove lightly.
Use --remove only when the target state is clearly defined before migration.
Do not use --remove for archival backups.
If the target end is used to retain history, avoid using --remove.

---

## Sixteen, Experiment Ten: Using include / exclude to Filter Synchronization

### 16.1 Filtering tmp Directory

Currently, the source directory contains:

    tmp/temp.txt

If you do not want to synchronize the tmp directory, use --exclude.

Dry-run preview:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --exclude "tmp/*" --dry-run /demo/source/ minio-src/mirror-src-demo-filtered/

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --exclude "tmp/*" --summary /demo/source/ minio-src/mirror-src-demo-filtered/

View results:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-src/mirror-src-demo-filtered

---

### 16.2 Synchronizing Only Log Files

Synchronize only .log files:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --include "*.log" --summary /demo/source/ minio-src/mirror-logs-only/

View results:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-src/mirror-logs-only

---

### 16.3 Recommendations for Filter Parameters

Production recommendations:

    Run include / exclude rules with dry-run first.
    Validate wildcard rules in a test environment.
    Avoid using complex filtering rules for the first time in production.
    Record filtering conditions before critical migration.
    Sample verify objects after migration to ensure they meet expectations.

---

## Seventeen, Experiment Eleven: Using Bandwidth Limiting to Control Synchronization Impact

### 17.1 Why Limit Speed

Massive synchronization may affect:

    Source cluster disk I/O
    Target cluster disk I/O
    Source cluster network bandwidth
    Target cluster network bandwidth
    Nginx entrance pressure
    Business normal upload/download

It is not recommended to perform large bucket migration without speed limiting during peak hours in production.

---

### 17.2 Limiting Upload Bandwidth

Example:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --limit-upload 20MiB --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

---

### 17.3 Limiting Download Bandwidth

Example:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --limit-download 20MiB --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

---

### 17.4 Controlling Concurrent Workers

Example:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --max-workers 4 --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

Notes:

    Too many workers may increase pressure on source and target ends.
    Too few workers may slow migration speed.
    Adjust based on network, disk, and business peak in production.

---

## Eighteen, Experiment Twelve: Using --watch for Continuous Synchronization

### 18.1 --watch Function

--watch continuously monitors changes in the source and synchronizes them to the target.

Suitable for:

    Experiment observation
    Temporary migration window
    Short-term incremental synchronization
    Catching up changes before cutover

Not suitable to directly replace: /think

Complete Disaster Recovery System  
Server-side Bucket Replication  
Site Replication  
Production-grade Long-term Replication Strategy  

---

### 18.2 Starting watch Synchronization  

Execute:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --watch --overwrite minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/  

Explanation:  

    This command will run continuously.  
    Need to open another terminal for testing.  
    Use Ctrl+C to stop.  

---

### 18.3 Writing New Objects in Another Terminal  

Create file:  

    echo "watch sync object" > /tmp/minio-mirror-demo/watch-object.txt  

Upload to source Bucket:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/watch-object.txt minio-src/mirror-src-demo/watch-object.txt  

Check target Bucket:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo  

---

### 18.4 --watch Risks  

Production risks:  

    The process will not automatically compensate for all historical issues after interruption, unless a full scan synchronization is performed again.  
    Requires process supervision and log monitoring.  
    If used with --remove, deletions from the source will propagate to the target.  
    Source-side errors will overwrite the target.  
    Not suitable for long-term background operation without monitoring.  
    Not equivalent to server-side replication mechanisms.  

For long-term replication in production, evaluate:  

    Bucket Replication  
    Site Replication  
    Batch replication tasks  
    Dedicated backup systems  

---

## Nineteen, Experiment Thirteen: Data Verification Before and After Migration  

### 19.1 Verification Objectives  

After migration, at least confirm:  

    Source Bucket exists.  
    Target Bucket exists.  
    Source Bucket capacity is close to target Bucket.  
    Source objects can be downloaded normally.  
    Target objects can be downloaded normally.  
    Key object contents are consistent.  
    Business applications can use the new Endpoint.  
    Permissions after migration meet expectations.  

---

### 19.2 Checking Source and Target Capacities  

Source:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-src/mirror-src-demo  

Target:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      du minio-bak/mirror-bak-demo  

---

### 19.3 Exporting Object Lists  

Source object list:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-src/mirror-src-demo > /tmp/minio-mirror-demo/reports/src-objects.txt  

Target object list:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo > /tmp/minio-mirror-demo/reports/bak-objects.txt  

Check:  

    wc -l /tmp/minio-mirror-demo/reports/src-objects.txt  
    wc -l /tmp/minio-mirror-demo/reports/bak-objects.txt  

Explanation:  

    Path prefixes differ, so direct diff may not be fully consistent.  
    Can use sed to remove different alias/bucket prefixes for comparison.  

---

### 19.4 Sample Download Verification  

Download source object:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-src/mirror-src-demo/app/config.txt /demo/reports/src-config.txt  

Download target object:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp minio-bak/mirror-bak-demo/app/config.txt /demo/reports/bak-config.txt

Calculate checksums:

  sha256sum /tmp/minio-mirror-demo/reports/src-config.txt
  sha256sum /tmp/minio-mirror-demo/reports/bak-config.txt

Compare contents:

  diff /tmp/minio-mirror-demo/reports/src-config.txt /tmp/minio-mirror-demo/reports/bak-config.txt

---

### 19.5 Large File Verification

Download source large file:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-mirror-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp minio-src/mirror-src-demo/backup/mysql/mysql-full-100m.sql /demo/reports/src-100m.sql

Download target large file:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-mirror-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    cp minio-bak/mirror-bak-demo/backup/mysql/mysql-full-100m.sql /demo/reports/bak-100m.sql

Calculate checksums:

  sha256sum /tmp/minio-mirror-demo/reports/src-100m.sql
  sha256sum /tmp/minio-mirror-demo/reports/bak-100m.sql

---

## Twenty, Experiment 14: Writing a Basic Backup Script

### 20.1 Script Objectives

Script implementation:

  Synchronize from source Bucket to backup Bucket.
  Use --overwrite.
  Default not use --remove.
  Output log file.
  Record start and end times.
  Preserve exit code.
  Suitable for crontab scheduled execution.

---

### 20.2 Create Backup Script

Create:

  cat > /tmp/minio-mirror-demo/scripts/minio-bucket-backup.sh <<'EOF'
  #!/bin/bash

  set -euo pipefail

  DATE=$(date +%F-%H%M%S)
  LOG_DIR="/var/log/minio-backup"
  LOG_FILE="${LOG_DIR}/mirror-src-demo-${DATE}.log"

  MC_IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z"
  MC_CONFIG="/data/minio/mc-config"

  SRC="minio-src/mirror-src-demo/"
  DST="minio-bak/mirror-bak-demo/"

  mkdir -p "${LOG_DIR}"

  echo "===== MinIO Bucket Backup Start: ${DATE} =====" | tee -a "${LOG_FILE}"
  echo "Source: ${SRC}" | tee -a "${LOG_FILE}"
  echo "Target: ${DST}" | tee -a "${LOG_FILE}"

  docker run --rm \
    -v ${MC_CONFIG}:/root/.mc \
    ${MC_IMAGE} \
    --insecure mirror \
    --overwrite \
    --summary \
    "${SRC}" "${DST}" 2>&1 | tee -a "${LOG_FILE}"

  EXIT_CODE=${PIPESTATUS[0]}

  END_DATE=$(date +%F-%H%M%S)
  echo "===== MinIO Bucket Backup End: ${END_DATE}, ExitCode=${EXIT_CODE} =====" | tee -a "${LOG_FILE}"

  exit ${EXIT_CODE}
  EOF

Grant permissions:

  chmod +x /tmp/minio-mirror-demo/scripts/minio-bucket-backup.sh

Copy to system path:

  cp /tmp/minio-mirror-demo/scripts/minio-bucket-backup.sh /usr/local/bin/minio-bucket-backup.sh

---

### 20.3 Execute Script

Execute:

  /usr/local/bin/minio-bucket-backup.sh

View logs:

  ls -lh /var/log/minio-backup/
  tail -100 /var/log/minio-backup/mirror-src-demo-*.log

---

### 20.4 crontab Scheduled Backup

Edit:

  crontab -e

Example: Run once daily at 2 AM.

  0 2 * * * /usr/local/bin/minio-bucket-backup.sh >/dev/null 2>&1

Production recommendation: /think

Backup tasks must have failure alerts.
Do not just write crontab without checking the results.
Logs should be integrated with Loki / ELK.
Backup success rate should be included in monitoring.
Perform regular recovery drills.

---

## 21. Production Migration Process Design

### 21.1 Pre-Migration Assessment

Before migration, must confirm:

    Source cluster version.
    Target cluster version.
    Source Bucket count.
    Target Bucket planning.
    Total capacity.
    Object count.
    Large object count.
    Small object count.
    Network bandwidth.
    Business read/write mode.
    Whether write is allowed to stop.
    Whether incremental sync is needed.
    Whether version history needs to be preserved.
    Whether Policy migration is needed.
    Whether user and AccessKey migration is needed.
    Whether rollback plan is available.

---

### 21.2 Pre-Migration Commands

Check source Bucket:

    mc ls minio-src

Check capacity:

    mc du minio-src/<bucket>

Check objects:

    mc find minio-src/<bucket>

Check permissions:

    mc admin user list minio-src
    mc admin policy list minio-src

Notes:

    Users, Policy, Bucket configurations, and object data are different levels of content.
    mc mirror mainly synchronizes object data.
    IAM, Policy, Bucket versions, replication rules, etc., need separate planning.

---

### 21.3 Migration Steps

Recommended process:

    1. Create target cluster.
    2. Create target Bucket.
    3. Create target users and Policy.
    4. First execute mc mirror --dry-run.
    5. Execute initial full sync.
    6. Multiple incremental syncs.
    7. Business stop write or enter maintenance window.
    8. Execute final incremental sync.
    9. Verify object count and capacity.
    10. Sample download critical objects.
    11. Switch application Endpoint.
    12. Validate business read/write.
    13. Retain source cluster for a period.
    14. Complete migration post-mortem.

---

### 21.4 Post-Migration Verification

After migration, confirm:

    Application can upload objects.
    Application can download objects.
    Bucket permissions are normal.
    Large file upload is normal.
    Nginx entry is normal.
    HTTPS certificate is normal.
    mc admin info is normal.
    Prometheus monitoring is normal.
    Alert rules are normal.
    Backup tasks are normal.
    Source cluster retention period is clear.

---

### 21.5 Rollback Plan

Migration must have rollback plan:

    Retain source cluster.
    Retain source Bucket.
    Retain original application Endpoint configuration.
    Retain original AccessKey.
    Retain pre-switch configuration backup.
    Clearly define DNS TTL.
    Clearly define rollback operator.
    Clearly define rollback verification items.

Principles:

    No rollback plan, no production migration.
    No data verification, not considered migration complete.

---

## 22. Comparison of mc mirror, Bucket Replication, Site Replication

### 22.1 mc mirror

Suitable for:

    Manual migration.
    Scheduled backup.
    Temporary sync.
    Local directory and Bucket sync.
    Small to medium-scale migration.
    One-time migration.
    Migration before/after difference filling.

Features:

    Client-driven.
    Depends on machine running mc.
    Requires scripts, logs, and monitoring.
    Does not fully sync object version history and metadata.
    Flexible usage.

---

### 22.2 Bucket Replication

Suitable for:

    Automatic sync of specified Bucket.
    Source Bucket to target Bucket.
    Continuous replication.
    Cross-cluster object sync.
    More formal object-level replication.

Features:

    Server-side replication.
    More suitable for long-term replication than mc mirror.
    Requires replication rule configuration.
    Must meet version, permission, and environment requirements.
    Suitable for continuous sync of important Buckets.

---

### 22.3 Site Replication

Suitable for:

    Multi-site full replication.
    Multi-cluster data and configuration sync.
    More complete disaster recovery/multi-active design.
    Multi-site object storage platform.

Features:

    Higher complexity.
    Higher requirements for version, identity, and configuration consistency.
    Suitable for more mature production environments.
    Not suitable as the first choice for beginners.

---

### 22.4 Selection Recommendations

| Scenario | Recommended Method |
|---|---|
| Local directory upload to Bucket | mc mirror |
| Temporary Bucket migration | mc mirror |
| Scheduled Bucket backup | mc mirror + script + alert |
| Long-term replication of specified Bucket | Bucket Replication |
| Multi-site full sync | Site Replication |
| Small-scale manual export | mc cp / mc mirror |
| Production disaster recovery | Bucket Replication / Site Replication + backup drill |
| Preserve object version history | Evaluate Bucket Replication / version control |

---

## 23. Common Issue Troubleshooting

### 23.1 mirror Execution Failure

Troubleshoot:

    Check alias correctness.
    Check source existence.
    Check target existence.
    Check if user has read source permission.
    Check if user has write target permission.
    Check if source cluster is ready.
    Check if target cluster is ready.
    Check network stability.
    Check certificate trust.
    Check if wrong port is used.

Commands:

    mc alias list
    mc ls minio-src
    mc ls minio-bak
    mc admin info minio-src
    mc admin info minio-bak

---

### 23.2 Access Denied

Common causes:

    Source user lacks GetObject.
    Source user lacks ListBucket.
    Target user lacks PutObject.
    Target user lacks DeleteObject.
    Used --remove but lacks DeleteObject.
    Policy restricts Bucket or Prefix.
    Alias used wrong user.

Resolution:

    Check user.
    Check Policy.
    Check Bucket.
    Check Prefix.
    Use admin user for temporary verification.

### 23.3 Slow Synchronization Speed

Possible Causes:

    Source-side disk is slow.
    Target-side disk is slow.
    Network bandwidth is insufficient.
    Too many small objects.
    Nginx proxy bottleneck.
    Too few mc workers.
    Low rate-limiting parameters.
    Source cluster is healing.
    Target cluster has high capacity pressure.

Troubleshooting:

    iostat -x 1
    iftop
    docker stats
    mc admin info
    Nginx access.log
    Prometheus metrics

---

### 23.4 Target-side Objects Exceed Source-side

Possible Causes:

    --remove was not used.
    Target-side originally had other objects.
    Historical synchronization residue.
    Source-side deletions not synchronized to target.
    Target-side being written by other tasks.

Resolution:

    Confirm whether target-side objects need to be retained.
    Do not directly use --remove.
    Perform --dry-run first.
    Confirm backup and approval before proceeding.

---

### 23.5 Target-side Objects Accidentally Deleted

Possible Causes:

    --remove was used.
    Source wrote to wrong location.
    Target wrote to wrong location.
    Source-side data is incomplete.
    Synchronization task misconfigured.
    Multiple tasks operating on same Bucket simultaneously.

Resolution:

    Immediately stop synchronization tasks.
    Disable related AccessKey.
    Check logs.
    Restore from independent backup.
    Check for version control.
    Review scripts and approval processes.

---

##24,High-risk Operations Warning

The following operations must be handled with caution in production environments:

    mc mirror --remove
    mc mirror --overwrite
    mc mirror --watch --remove
    Synchronizing across production environments
    Executing backup tasks as root user
    Using same AccessKey group to access source and target
    Synchronizing without dry-run
    Cron backups without logs
    Backup tasks without failure alerts
    Backup strategies without recovery drills
    Resynchronizing after deleting target Bucket
    Synchronizing without confirming Bucket ownership

Before production execution, must confirm:

    Who is the source.
    Who is the target.
    Whether current environment is production.
    Whether target objects will be overwritten.
    Whether target objects will be deleted.
    Whether there is backup.
    Whether there is rollback.
    Whether there is approval.
    Whether there is logging.
    Whether there is alerting.
    Whether test has been done.

---

##25,Production Backup Strategy Recommendations

### 25.1 Backup Target Design

Recommend at least:

    Source cluster and backup cluster isolated.
    Backup cluster not shared with source cluster.
    Backup account permissions minimized.
    Backup Bucket not allowed for regular business writes.
    Backup data has retention period.
    Backup tasks have logs.
    Backup failures have alerts.
    Regular recovery verification.

---

### 25.2 Backup Frequency Design

Divide by data importance:

| Data Type | Recommended Frequency |
|---|---|
| User-uploaded attachments | Hourly or daily |
| Business-critical files | Hourly or higher |
| Log archives | Daily |
| Backup archives | Daily or per backup policy |
| CI/CD artifacts | Daily or per version retention |
| Temporary files | Can be unbacked up or low-frequency backup |

---

### 25.3 Retention Period Design

Recommend considering:

    7-day short-term recovery.
    30-day accidental deletion recovery.
    90-day audit retention.
    180-day or longer compliance retention.

Specifically depends on:

    Business requirements.
    Compliance requirements.
    Storage costs.
    Recovery objectives.
    Data growth rate.

---

### 25.4 Recovery Drills

Backup without verifying recovery is equivalent to incomplete closure.

Recovery drills should verify:

    Ability to download objects from backup Bucket.
    Ability to recover to new Bucket.
    Ability to recover critical Prefix.
    Ability to recover large files.
    Ability to recover business application directory structure.
    Recovery time is acceptable.
    Recovery process has documentation.
    Recovery operations are performed by someone.

---

##26,Experion Cleanup

### 26.1 Delete Source-side Experiment Bucket

High-risk warning:

    Only clean up experiment Bucket.
    Do not accidentally delete production Bucket.

Delete source-side objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-src/mirror-src-demo

Delete source-side Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-src/mirror-src-demo

Delete filtered experiment Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-src/mirror-src-demo-filtered

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-src/mirror-src-demo-filtered

Delete log experiment Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-src/mirror-logs-only

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure rb minio-src/mirror-logs-only

---

### 26.2 Delete Backup Bucket Experiment

Delete objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force minio-bak/mirror-bak-demo

Delete Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb minio-bak/mirror-bak-demo

---

### 26.3 Stop Backup MinIO Experiment Instance

If the MinIO experiment instance no longer needs to be backed up:

    docker stop minio-backup
    docker rm minio-backup

Delete experiment data:

    rm -rf /data/minio-backup/data

High-risk warning:

    Deleting /data/minio-backup/data will remove all objects from the backup endpoint.
    Only allowed to execute in experimental environments.
    Production environments must not directly rm -rf the backup directory.

---

### 26.4 Clean Up Local Experiment Directory

    rm -rf /tmp/minio-mirror-demo

---

## Twenty-Seven, Production Backup Migration Checklist

### 27.1 Pre-Migration Checks

| Check Item | Requirement | Result |
|---|---|---|
| Source Cluster Health | admin info is normal |  |
| Target Cluster Health | admin info is normal |  |
| Source Bucket Capacity | Has been counted |  |
| Source Object Count | Has been evaluated |  |
| Target Bucket | Has been created |  |
| Permissions | Source read, Target write |  |
| Network Bandwidth | Has been evaluated |  |
| Maintenance Window | Has been confirmed |  |
| Rollback Plan | Has been prepared |  |
| dry-run | Has been executed |  |

---

### 27.2 During Migration Checks

| Check Item | Requirement | Result |
|---|---|---|
| Synchronization Logs | Normal output |  |
| Error Objects | None or already recorded |  |
| Source Cluster Pressure | Acceptable |  |
| Target Cluster Pressure | Acceptable |  |
| Network Bandwidth | Acceptable |  |
| Business Impact | Acceptable |  |
| Alarms | No critical anomalies |  |

---

### 27.3 Post-Migration Checks

| Check Item | Requirement | Result |
|---|---|---|
| Source Capacity | Has been recorded |  |
| Target Capacity | Has been recorded |  |
| Object Count | Basically consistent |  |
| Sample Validation | Passed |  |
| Large File Validation | Passed |  |
| Permission Verification | Passed |  |
| Application Switch | Passed |  |
| Monitoring Alarms | Normal |  |
| Source Cluster Retention | Has been clearly defined |  |

---

## Twenty-Eight, Interview Answer Approach

If asked in an interview:

    How does MinIO do backup migration? How to use mc mirror?

You can answer:

    MinIO has Erasure Coding which can tolerate partial disk or node failures, but it cannot replace backup. Accidental deletion, incorrect overwriting, AccessKey leaks, ransomware, and full cluster failures require independent backup or cross-cluster recovery capabilities.
    A common approach is to use mc mirror for object synchronization. mc mirror is similar to rsync, and can synchronize local directories to a Bucket, synchronize one MinIO Bucket to another Bucket, or synchronize between MinIO clusters.
    Before migration, I will first use mc du and mc find to evaluate source Bucket capacity and object scale, then create the target Bucket, configure source-side read-only permissions and target-side write permissions. Before formal synchronization, I will execute mc mirror --dry-run to confirm source and target are not reversed. Full synchronization can use mc mirror --summary, and incremental synchronization can be combined with --overwrite. If you want the target to be exactly the same as the source, you can use --remove, but this parameter is very dangerous and must first dry-run, approve, and review, because accidental deletion on the source will synchronize deletion on the target.
    For large-scale synchronization, I will combine --limit-upload, --limit-download, or --max-workers to control business impact, and integrate synchronization logs into the log platform with failure alerts. After migration, I will validate source and target capacity, object count, and sample download critical objects to calculate checksums.
    For long-term production replication, mc mirror can be used for scheduled backups or temporary migration, but more formal cross-cluster replication should evaluate Bucket Replication or Site Replication. Regardless of the method, regular recovery drills must be conducted, otherwise backups cannot be verified.

---

## Twenty-Nine, Summary of This Article

This article completes the MinIO backup migration hands-on practice:

1. Erasure Coding is not equal to backup.
2. mc mirror is similar to rsync and can be used for object storage synchronization.
3. mc mirror supports synchronization from local directory to Bucket.
4. mc mirror supports synchronization from Bucket to local directory.
5. mc mirror supports synchronization between MinIO clusters.
6. Use --dry-run to preview before synchronization.
7. --summary outputs a synchronization summary.
8. --overwrite will overwrite objects with the same name on the target end.
9. --remove will delete objects on the target end that no longer exist on the source end, which is a high-risk parameter.
10. --watch can be used for continuous synchronization, but it is not equal to a complete disaster recovery system.
11. include / exclude can be used to filter objects.
12. limit-upload / limit-download can be used to control synchronization bandwidth.
13. max-workers can be used to control concurrency.
14. Capacity, object count, permissions, network, and rollback plan must be evaluated before migration.
15. Capacity, object count, and critical object content must be validated after migration.
16. Production backup tasks must have logs, alerts, and recovery drills.
17. mc mirror is suitable for manual migration, scheduled backups, and temporary synchronization.
18. Long-term production-grade cross-cluster replication should evaluate Bucket Replication or Site Replication.
19. Subsequent content will enter the MinIO phase summary: from S3 object storage to production-ready cluster.

---

## 30. Reference Documents

MinIO mc mirror documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-mirror.html

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO mc alias documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-alias.html

MinIO mc du documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-du.html

MinIO mc find documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-find.html

MinIO Bucket Replication documentation:

    https://min.io/docs/minio/linux/administration/bucket-replication.html

MinIO Site Replication documentation:

    https://min.io/docs/minio/linux/administration/site-replication.html

MinIO Batch Replication documentation:

    https://min.io/docs/minio/linux/administration/batch-framework.html

MinIO Erasure Coding documentation:

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html