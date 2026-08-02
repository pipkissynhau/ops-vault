# MinIO Data Protection: Erasure Coding, Node Failures, and Disk Failure Recovery

Recommended path: 05-Storage/02-MinIO/08-MinIO Data Protection: Erasure Coding, Node Failures, and Disk Failure Recovery.md

Tags: #MinIO #ErasureCoding #DataProtection #NodeFailure #DiskFailure #FailureRecovery. #Healing #ObjectStorage #S3 #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the eighth article in the MinIO module, focusing on learning MinIO's data protection, node failures, disk failures, and recovery drills.

Previously completed:

- MinIO object storage basics
- Single machine single disk deployment
- Single node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS access entry design
- Nginx HTTPS unified entry
- mc client configuration and object operations
- User, Policy, AccessKey, and SecretKey permission management

This article begins to explore the core capabilities of MinIO operations:

    How is data protected?
    What is Erasure Coding?
    Can read/write continue after node failure?
    How to recover after disk failure?
    What is healing?
    How to use mc admin heal?
    Can Erasure Coding replace backups?
    How to design fault drills in production environments?

This article emphasizes practical operations, conducting drills around a 4-node MinIO distributed cluster.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of MinIO Erasure Coding.
2. Understand the difference between Erasure Coding and triple replication.
3. Understand the impact of MinIO node and disk failures.
4. Understand the basic meaning of read quorum and write quorum.
5. Use mc admin info to check node and disk status.
6. Use mc admin heal to check and fix status.
7. Simulate single-node failure.
8. Validate object reads during node failure.
9. Validate object writes during node failure.
10. Recover failed nodes and observe cluster recovery.
11. Simulate single-disk directory failure.
12. Recover disk directories and trigger or observe healing.
13. Understand which failures can be tolerated and which cause unavailability.
14. Understand that Erasure Coding cannot replace backups.
15. Establish MinIO production fault drill and recovery procedures.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues with the 4-node distributed MinIO cluster:

| IP | Hostname | Role | Data Directory |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.45 | minio-client | mc Client / Test Client | - |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry | - |

---

### 3.2 Access Entry

Backend direct access entry:

    http://10.0.0.41:9000

HTTPS unified access entry:

    https://s3.minio.local

Console entry:

    https://console.minio.local

This article defaults to using:

    https://s3.minio.local

If the experimental environment uses a self-signed certificate, temporarily use:

    --insecure

in mc commands. Production environments should not use --insecure long-term.

---

### 3.3 Image Version

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

## IV. Understanding Erasure Coding Basics

### 4.1 What is Erasure Coding

Erasure Coding is commonly referred to as erasure coding.

Its core idea is:

    Split object data into multiple data blocks.
    Generate multiple parity blocks.
    Distribute data blocks and parity blocks across multiple disks or nodes.
    When some disks or nodes are lost, the original object can be recovered through the remaining blocks.

Simple understanding:

    The original object is not fully replicated multiple times.
    Instead, it is split and encoded.
    As long as the remaining available blocks meet the requirements, the object can be recovered.

---

### 4.2 Difference Between Erasure Coding and Replication

Replication mode:

    The original data is fully replicated multiple times.

For example, triple replication:

    Original data 1GB
    Actual storage about 3GB

Advantages:

    Simple to understand.
    Direct recovery.
    Clear read path.

Disadvantages:

    Low space utilization.

---

Erasure Coding:

    Original data is split into data blocks and parity blocks.

For example:

    4 data blocks + 4 parity blocks

Advantages:

    Space utilization is usually higher than multi-replication.
    Can tolerate partial disk or node failures.
    Suitable for large-capacity object storage scenarios.

Disadvantages:

    Encoding and recovery logic is more complex.
    More sensitive to node, disk, and network planning.
    Fault recovery consumes disk and network resources.
    Not suitable for being misunderstood as backup.

---

### 4.3 Data Protection in MinIO

MinIO uses Erasure Coding to protect object data.

It can handle:

    Single disk failure.
    Multiple disk failures.
    Single node failure.
    Partial node unavailability.
    Partial object data block loss.

But it cannot resolve:

    User accidental object deletion.
    Administrator accidental Bucket deletion.
    Program writing incorrect data.
    Ransomware deletion or encryption of objects.
    Entire cluster damage.
    Data center-level failure.
    Multiple nodes exceeding fault tolerance boundaries simultaneously.

Conclusion:

    Erasure Coding is a high availability and data protection mechanism.
    Erasure Coding is not backup.
    Erasure Coding cannot replaceAlien. backup or cross-cluster synchronization.

---

## V. read quorum and write quorum

### 5.1 What is Quorum

Quorum can be understood as:

    The minimum number of available data blocks required for read/write operations.

In distributed object storage, not all disks must be online to perform read/write operations.

As long as the minimum requirement for read/write operations is met, the service can continue to be provided.

---

### 5.2 Read Quorum

Read quorum indicates:

    The minimum number of available blocks required to read an object.

If the number of failed disks or nodes does not exceed the tolerable range, existing objects are typically still readable.

If too many data blocks are lost, reading an object may fail.

---

### 5.3 Write Quorum

Write quorum indicates:

    The minimum number of available blocks required to write an object.

Writing is typically more sensitive than reading.

If there are insufficient online disks or nodes, the following may occur:

    Old objects can still be read.
    New objects cannot be written.
    Uploads may fail.
    Bucket operations may fail.

---

### 5.4 Operations Understanding

After a failure occurs, do not only ask:

    How many nodes are left?

Also check:

    How many disks are left?
    How many available blocks are left in the erasure set where the current object resides?
    Does it meet the read quorum?
    Does it meet the write quorum?
    Are multiple failures concentrated in the same erasure set?
    Are there nodes or disks that have been offline for a long time?

Therefore, after a failure occurs, the actual commands must be used to confirm the status, rather than judging based on intuition.

---

## Six. Experimental Tasks in This Article

This article will complete the following hands-on experiments:

| Experiment | Content |
|---|---|
| Experiment One | Pre-health Check Before Fault Simulation |
| Experiment Two | Create a Data Protection Test Bucket |
| Experiment Three | Upload Small and Large Files |
| Experiment Four | View Cluster Node and Disk Status |
| Experiment Five | Simulate Single Node Failure |
| Experiment Six | Verify Read During Node Failure |
| Experiment Seven | Verify Write During Node Failure |
| Experiment Eight | Recover the Faulty Node |
| Experiment Nine | Use mc admin heal to Check and Repair |
| Experiment Ten | Simulate Single Disk Directory Failure |
| Experiment Eleven | Recover the Disk Directory and Observe Healing |
| Experiment Twelve | Summarize the Production Fault Simulation Process |

---

## Seven. Prepare the mc Management Environment

### 7.1 Create Directories

Execute on minio-client or management node:

    mkdir -p /data/minio/mc-config
    mkdir -p /tmp/minio-heal-demo/upload
    mkdir -p /tmp/minio-heal-demo/download
    mkdir -p /tmp/minio-heal-demo/check

---

### 7.2 Configure Alias

If using HTTPS unified entry with self-signed certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using official certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

---

### 7.3 View Alias

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

---

### 7.4 View Cluster Information

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Requirements:

    All nodes are online.
    All disks are online.
    No obvious offline drives.
    The cluster can read and write normally.

---

## Eight. Experiment One: Pre-fault Simulation Health Check

### 8.1 Check MinIO Backend Health

Execute on minio-client or minio-entry:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check live $ip"
      curl -I http://$ip:9000/minio/health/live
    done

Check ready:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check ready $ip"
      curl -I http://$ip:9000/minio/health/ready
    done

---

### 8.2 Check Container Status

Execute on each of the 4 MinIO nodes:

    docker ps | grep minio
    docker logs --tail=50 minio

---

### 8.3 Check Disk Capacity

Execute on each of the 4 MinIO nodes:

    df -hT
    du -sh /data/minio/disk1
    du -sh /data/minio/disk2
    lsblk

Key confirmations:

    Data directory exists.
    Data directory is writable.
    System disk is not full.
    Data directory is not mistakenly mounted and data is not lost.

---

### 8.4 Check Time Synchronization

Execute on each of the 4 MinIO nodes:

    timedatectl

If time is not synchronized, address the time synchronization issue first.

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mb minio-admin/heal-demo

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls minio-admin

---

### 9.2 Prepare Test Objects

Create small file:

echo "hello minio healing demo" > /tmp/minio-heal-demo/upload/hello.txt

Create large file:

dd if=/dev/zero of=/tmp/minio-heal-demo/upload/file-100m.bin bs=1M count=100

Create multiple objects:

for i in $(seq 1 20); do
  echo "object $i for minio healing demo" > /tmp/minio-heal-demo/upload/object-$i.txt
done

View:

ls -lh /tmp/minio-heal-demo/upload
du -sh /tmp/minio-heal-demo/upload

---

### 9.3 Upload Test Objects

Upload small file:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-heal-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp /demo/upload/hello.txt minio-admin/heal-demo/hello.txt

Upload large file:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-heal-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp /demo/upload/file-100m.bin minio-admin/heal-demo/file-100m.bin

Recursively upload multiple objects:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-heal-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp --recursive /demo/upload/ minio-admin/heal-demo/batch/

---

### 9.4 View Objects

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/heal-demo

View capacity:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure du minio-admin/heal-demo

---

## TenI don't know.Experiment Three: Pre-Failure Object Integrity Record

### 10.1 Download Objects for Baseline

Download hello.txt:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-heal-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp minio-admin/heal-demo/hello.txt /demo/check/hello-before.txt

Download large file:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-heal-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp minio-admin/heal-demo/file-100m.bin /demo/check/file-100m-before.bin

---

### 10.2 Calculate Checksums

sha256sum /tmp/minio-heal-demo/check/hello-before.txt
sha256sum /tmp/minio-heal-demo/check/file-100m-before.bin

Save output to record file: /tmp/minio-heal-demo/check/record.txt

sha256sum /tmp/minio-heal-demo/check/hello-before.txt > /tmp/minio-heal-demo/check/checksum-before.txt
sha256sum /tmp/minio-heal-demo/check/file-100m-before.bin >> /tmp/minio-heal-demo/check/checksum-before.txt

View:

    cat /tmp/minio-heal-demo/check/checksum-before.txt

Explanation:

    After fault recovery, you can re-download the object and calculate checksums.
    Matching checksums indicate the object content has not changed.

---

## ElevenI don't know.Experiment Four: Check Cluster Status and Healing Status

### 11.1 Check admin info

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Focus on:

    Number of nodes.
    Number of disks.
    Online status.
    Offline status.
    Healing status.
    Capacity usage.
    Presence of warnings.

---

### 11.2 heal dry-run Check

Execute dry-run without actual repair:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin heal --recursive --dry-run minio-admin/heal-demo

Explanation:

    --dry-run is used to view objects that may need repair.
    Does not perform actual repairs.
    Suitable for pre- and post-fault comparison.
    In production, it's recommended to prioritize dry-run to observe impacts.

---

### 11.3 Check Service Logs

On each MinIO node, you can view:

    docker logs --tail=100 minio

For continuous observation:

    docker logs -f minio

---

## TwelveI don't know.Experiment Five: Simulate Single Node Failure

### 12.1 Exercise Objective

Verify:

    Whether the cluster can still access after stopping one MinIO node.
    Whether existing objects can still be read.
    Whether new objects can still be written.
    Whether admin info can detect node anomalies.
    Whether the cluster recovers normally after restoring the node.

---

### 12.2 Pre-Exercise Confirmation

Run on minio-client:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Requirements:

    Cluster is normal.
    All nodes are online.
    No ongoing obvious healing.
    No disk space shortages.

---

### 12.3 Stop minio-node04

Run on minio-node04:

    docker stop minio

View:

    docker ps -a | grep minio

---

### 12.4 Check Cluster Status

Run on minio-client:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Expected:

    You can see nodes or disks offline.
    The cluster may show degraded / warning states.
    Actual output may vary by version.

---

### 12.5 Check Health Interface

Run on minio-entry or minio-client:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.42:9000/minio/health/live
    curl -I http://10.0.0.43:9000/minio/health/live
    curl -I http://10.0.0.44:9000/minio/health/live

node04 is expected to fail or have no response.

Check ready:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

---

## ThirteenI don't know.Experiment Six: Verify Reads During Node Failure

### 13.1 Download Existing Small Files

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-heal-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-admin/heal-demo/hello.txt /demo/download/hello-node04-down.txt

View:

    cat /tmp/minio-heal-demo/download/hello-node04-down.txt

Expected:

    If the current number of failures does not exceed the read quorum, the download succeeds.

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-heal-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp minio-admin/heal-demo/file-100m.bin /demo/download/file-100m-node04-down.bin

Check size:

  ls -lh /tmp/minio-heal-demo/download/file-100m-node04-down.bin

Calculate checksum:

  sha256sum /tmp/minio-heal-demo/download/file-100m-node04-down.bin

Compare with the checksum before the failure.

---

### 13.3 Interpreting Read Results

If read succeeds:

  The remaining nodes and disks still meet the read quorum.
  Erasure Coding can recover object data under partial node failures.

If read fails:

  It might exceed the read tolerance of the erasure set containing the object.
  Or there might be other cluster failures.
  Check admin info, node status, disk status, Nginx logs, and MinIO logs.

---

## FourteenI don't know.Experiment Seven: Verifying Write During Node Failure

### 14.1 Create New Object

Run in minio-client:

  echo "write object while node04 down" > /tmp/minio-heal-demo/upload/write-node04-down.txt

Upload:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-heal-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp /demo/upload/write-node04-down.txt minio-admin/heal-demo/write-node04-down.txt

---

### 14.2 Check Upload Results

Check object:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure ls minio-admin/heal-demo

If upload succeeds:

  The cluster still meets the write quorum.
  After the failed node recovers, related data may need synchronization or healing.

If upload fails:

  The current failure state may not meet the write quorum.
  Check admin info and error messages.

---

### 14.3 Operations Understanding

Read and write results during node failure should be judged separately:

  Readable existing objects do not guarantee new objects are writable.
  Writable new objects indicate the write condition is still met.
  After failure recovery, check object status and healing status.
  Long-term node failure increases data risk.

---

## FifteenI don't know.Experiment Eight: Recovering Failed Node

### 15.1 Start minio-node04

Run on minio-node04:

  docker start minio

Check:

  docker ps | grep minio
  docker logs --tail=100 minio

---

### 15.2 Check Cluster Status

Run in minio-client:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure admin info minio-admin

Observe:

  Whether node04 is back online.
  Whether offline drives disappear.
  Whether healing status exists.
  Whether any warnings remain.

---

### 15.3 Re-verify Objects

Download the object written during failure:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-heal-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp minio-admin/heal-demo/write-node04-down.txt /demo/download/write-node04-down-download.txt

Check:

  cat /tmp/minio-heal-demo/download/write-node04-down-download.txt

---

## SixteenI don't know.Experiment Nine: Using mc admin heal to Check and Repair

### 16.1 Dry-run Check

Run:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure admin heal --recursive --dry-run minio-admin/heal-demo

Notes:

  Dry-run does not perform real repairs.
  Suitable for checking objects that may need healing.
  In production, recommend dry-run first, then decide whether to execute repairs.

---

### 16.2 Execute Healing Check or Repair

In the experimental environment, you can execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure admin heal --recursive minio-admin/heal-demo

**Note:**

  heal scans and attempts to repair objects.
  Performing this on large Buckets may consume significant IO and network resources.
  Full recursive heal should not be executed arbitrarily during business peaks in production.
  Production should evaluate scope, window, and impact.

---

### 16.3 Viewing heal Help

Parameter differences may exist between mc versions. Check help:

  docker run --rm \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    admin heal --help

---

### 16.4 Production Healing Considerations

Before executing healing in production, confirm:

  Whether the current failure has been resolved.
  Whether disks have been replaced or re-mounted.
  Whether nodes are stable.
  Whether the Bucket is very large.
  Whether it will affect business IO.
  Whether to execute by Prefix.
  Whether a maintenance window is needed.
  Whether business has been notified.
  Whether monitoring is in place for disk and network pressure.

---

## SeventeenI don't know.Experiment Ten: Simulating Single Disk Directory Failure

### 17.1 Exercise Notes

In this environment, each node uses two directories:

  /data/minio/disk1
  /data/minio/disk2

If these directories are merely ordinary directories on the same disk, this experiment can only simulate directory unavailability, not fully replicate real disk failure.

In real production, disk1 and disk2 should be different disk mount points.

This experiment selects to simulate disk2 failure on minio-node04.

**High-risk warning:**

  Do not arbitrarily move or delete MinIO data directories in production.
  This exercise is only suitable for test environments.
  Production disk failures should focus on replacing disks, re-mounting, and observing healing.
  Do not manually modify object data content.

---

### 17.2 Pre-exercise Checks

Execute on minio-client:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure admin info minio-admin

Execute on minio-node04:

  df -hT
  ls -ld /data/minio/disk1 /data/minio/disk2
  du -sh /data/minio/disk1 /data/minio/disk2

---

### 17.3 Stop minio-node04 Container

Execute on minio-node04:

  docker stop minio

---

### 17.4 Move disk2 Directory to Simulate Disk Loss

Execute on minio-node04:

  mv /data/minio/disk2 /data/minio/disk2.failed

Recreate empty directory to simulate new disk or empty mount point:

  mkdir -p /data/minio/disk2

View:

  ls -ld /data/minio/disk2 /data/minio/disk2.failed

**Note:**

  This simulates original disk2 data loss or disk unavailability.
  In real production, it's typically disk damage followed by replacing with new disk and mounting to same path.

---

### 17.5 Start minio-node04

Execute on minio-node04:

  docker start minio

View logs:

  docker logs --tail=100 minio

---

### 17.6 Check Cluster Status

Execute on minio-client:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure admin info minio-admin

**Expected:**

  You may see a drive status anomaly, needing healing, or capacity change.
  Specific manifestations depend on MinIO version and detection results.

---

### 17.7 Verify Object Read

Download hello.txt:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-heal-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp minio-admin/heal-demo/hello.txt /demo/download/hello-disk2-failed.txt

View:

  cat /tmp/minio-heal-demo/download/hello-disk2-failed.txt

---

### 17.8 Verify Object Write

Create file:

  echo "write after disk2 failed" > /tmp/minio-heal-demo/upload/write-disk2-failed.txt

Upload:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-heal-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure cp /demo/upload/write-disk2-failed.txt minio-admin/heal-demo/write-disk2-failed.txt

If upload succeeds:

  The cluster still meets write quorum.

If upload fails:

    The current fault state affects writing, and needs to be judged in combination with admin info.

---

## EighteenI don't know.Experiment Eleven: Restore Disk Directory and Observe Healing

### 18.1 Recovery Method Explanation

There are two understandings of recovery after disk failure:

Method One: Original disk recovery.

    Original directory data is still present.
    Remount to the original path.
    Start service.
    Check status.

Method Two: Original disk damage, replace with new disk.

    New disk mounted to the same path.
    Original data is empty.
    MinIO attempts to rebuild missing data through healing.

This experiment simulates Method Two:

    The disk2 directory is empty.
    Let MinIO attempt to repair through remaining data.

---

### 18.2 Execute heal dry-run

Execute in minio-client:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin heal --recursive --dry-run minio-admin/heal-demo

---

### 18.3 Execute healing

Execute in experimental environment:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin heal --recursive minio-admin/heal-demo

Observe:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Check directory size changes on minio-node04:

    du -sh /data/minio/disk2
    find /data/minio/disk2 -maxdepth 4 -type f | head -50

---

### 18.4 Download Objects and Verify After Recovery

Download large file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-heal-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-admin/heal-demo/file-100m.bin /demo/check/file-100m-after-heal.bin

Calculate checksum:

    sha256sum /tmp/minio-heal-demo/check/file-100m-after-heal.bin

Compare with pre-fault checksum:

    cat /tmp/minio-heal-demo/check/checksum-before.txt

If checksums match, the object content has returned to normal.

---

### 18.5 Restore Original Experiment Directory

If it's just an experiment and you don't want to retain the simulated fault state, you can restore the original directory.

High-risk warning:

    The following operations are only suitable for experimental environments.
    Do not arbitrarily overwrite data directories in production.

Stop minio-node04:

    docker stop minio

Delete current empty disk directory:

    rm -rf /data/minio/disk2

Restore original directory:

    mv /data/minio/disk2.failed /data/minio/disk2

Start:

    docker start minio

Check:

    docker logs --tail=100 minio

Check on client:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

---

## NineteenI don't know.Real Disk Fault Handling Process

### 19.1 Discovery of Disk Fault

Common discovery methods:

    mc admin info shows drive offline.
    Prometheus alert.
    Host dmesg reports I/O error.
    df doesn't show mount point.
    lsblk discovers disk loss.
    SMART alarm.
    MinIO logs show drive error.

Troubleshooting commands:

    df -hT
    lsblk
    dmesg | tail -100
    journalctl -k | tail -100
    smartctl -a /dev/sdX
    docker logs minio

---

### 19.2 Fault Confirmation

Confirm content:

    Is it a disk fault or mount point loss?
    Is it a file system issue or hardware problem?
    Is it a single-disk fault or multi-disk fault?
    Does it affect read quorum?
    Does it affect write quorum?
    Are there other nodes also abnormal?
    Can current business still read/write?

---

### 19.3 Replace Disk

High-risk warning:

    Must confirm device name before replacing disk.
    Do not format the wrong disk.
    Production recommends dual-person verification.

Example process:

    lsblk
    umount /data/minio/disk2
    Replace physical disk
    mkfs.xfs /dev/sdX
    mount /dev/sdX /data/minio/disk2
    blkid /dev/sdX
    Update /etc/fstab
    mount -a
    df -hT

---

### 19.4 Restore MinIO Service

If MinIO container is still running, confirm it can recognize the new disk.

Restart current node MinIO if necessary:

    docker restart minio

Check logs:

    docker logs -f minio

Check cluster: /think

# mc admin info

---

### 19.5 Performing or Observing Healing

First perform a dry-run:

    mc admin heal --recursive --dry-run minio-admin/heal-demo

Then execute by range:

    mc admin heal --recursive minio-admin/heal-demo

Production recommendations:

    Do not perform full recursive healing on the entire cluster initially.
    You can process by Bucket or Prefix in batches.
    Avoid during business peak hours.
    Monitor disk IO, network, and API latency.

---

## Twenty, Node Failure Handling Process

### 20.1 Detecting Node Failure

Common manifestations:

    admin info shows node offline.
    Nginx upstream backend failure.
    MinIO health ready failure.
    Host ping unreachable.
    SSH unreachable.
    Docker container exited.
    Node downtime or reboot.

Troubleshooting:

    ping 10.0.0.44
    ssh root@10.0.0.44
    docker ps -a | grep minio
    docker logs minio
    systemctl status docker
    df -hT
    dmesg | tail -100

---

### 20.2 Node Recovery

If only the container stopped:

    docker start minio

If Docker is abnormal:

    systemctl restart docker
    docker start minio

If node rebooted:

    Wait for node to start.
    Check data directory.
    Check Docker.
    Check MinIO container.
    Check admin info.

---

### 20.3 Long-term Unrecoverable Node

If hardware damage occurs, you need:

    Prepare new node.
    Use original IP or ensure cluster endpoint is reachable.
    Mount new data disk.
    Use same startup parameters.
    Use same MINIO_ROOT_USER / MINIO_ROOT_PASSWORD.
    Use same MINIO_SERVER_URL.
    Use same MINIO_BROWSER_REDIRECT_URL.
    Start MinIO.
    Observe healing.

Note:

    Distributed MinIO's server endpoint is part of the cluster topology.
    If replacing nodes in production, try to maintain original IP, hostname, and data path.
    Complex changes should follow official procedures and be validated in test environments.

---

## Twenty-one, Fault Drill Risk Boundaries

### 21.1 Safe Drill Contents

Test environment can drill:

    Stop a MinIO container.
    Start a MinIO container.
    Stop a test node.
    Simulate single disk directory loss.
    Upload/download test objects.
    Execute heal dry-run.
    Perform heal on test Bucket.

---

### 21.2 High-risk Drill Contents

Production environment should be cautious:

    Delete data directory.
    Format disk.
    Stop multiple nodes simultaneously.
    Remove multiple disks simultaneously.
    Modify cluster endpoint.
    Modify data directory path.
    Perform full recursive heal on large Bucket.
    Execute healing during business peak.
    Perform destructive tests without backup.

---

### 21.3 Must Confirm Before Drills

Must confirm before drills:

    Whether it's a test environment.
    Whether there is a complete backup.
    Whether there is a business window.
    Whether recovery steps are known.
    Whether rollback plan exists.
    Whether monitoring observation is in place.
    Whether someone will review.
    Whether start and end times are recorded.

---

## Twenty-two, Erasure Coding Cannot Replace Backup

### 22.1 What Erasure Coding Can Solve

Can resolve or mitigate:

    Single disk failure.
    Single node failure.
    Partial data block loss.
    Hardware partial damage.
    Node short-term offline.

---

### 22.2 What Erasure Coding Cannot Solve

Cannot resolve:

    Accidental object deletion.
    Accidental Bucket deletion.
    Application overwriting files.
    Administrator rm --recursive --force.
    Ransomware attack.
    MinIO cluster-wide damage.
    Data center failure.
    Multiple nodes exceeding tolerance simultaneously.
    Data deletion after root user leakage.

---

### 22.3 Production Must Supplement Backup

Production should supplement:

    mc mirror to another MinIO cluster.
    mc mirror to cloud vendor object storage.
    Cross-cluster replication.
    Regular offline backup.
    Bucket versioning.
    Lifecycle policies.
    Delete protection policies.
    Regular recovery drills.

Backup migration will be detailed in:

    10-MinIO Backup Migration: mc mirror, Cross-cluster Synchronization and Data Migration.md

---

## Twenty-three, Monitoring Alert Recommendations

### 23.1 Node Alerts

Need alerts:

    MinIO node offline.
    MinIO API unavailable.
    MinIO ready check failure.
    Nginx upstream failure.
    Node CPU/memory anomalies.
    Node network anomalies.

---

### 23.2 Disk Alerts

Need alerts:

    Disk offline.
    Disk capacity over 80%.
    Disk capacity over 90%.
    Disk read-only.
    Disk I/O error.
    Inode anomalies.
    Mount point lost.
    SMART anomalies.

---

### 23.3 Object Storage Alerts

Need alerts:

    Bucket capacity abnormal growth.
    Object count abnormal growth.
    Increase in API 5xx errors.
    API latency increase.
    Large number of DeleteObject requests.
    Access Denied surge.
    Healing long time not completed.
    Backup mirror failure.

---

### 23.4 Security Alerts

Need attention:

    Root user frequent access.
    Console abnormal login.
    Unknown IP access.
    Large downloads.
    Large deletions.
    AccessKey failure count anomalies.
    Nginx 4xx/5xx anomalies.

---

## Twenty-four, Production Incident Report Template

### 24.1 Incident Basic Information /think

| Item | Content |
|---|---|
| Fault Time |  |
| Discovery Method | Alert / User Feedback / Patrol |
| Impact Scope |  |
| Affected Bucket |  |
| Affected Business |  |
| Does It Affect Read |  |
| Does It Affect Write |  |
| Current Cluster Status |  |

---

### 24.2 Fault Phenomenon

Record:

    admin info output.
    Health check results.
    Nginx error log.
    MinIO log.
    Node system log.
    Disk status.
    User feedback phenomenon.

---

### 24.3 Handling Process

Record:

| Time | Operation | Operator | Result |
|---|---|---|---|
|  |  |  |  |

---

### 24.4 Recovery Verification

Verification:

    All MinIO nodes are online.
    All disks are online.
    admin info is normal.
    Business Bucket can be listed.
    Objects can be uploaded.
    Objects can be downloaded.
    Large files can be uploaded.
    Nginx entry is normal.
    Console is normal.
    Alerts are recovered.

---

### 24.5 Post-Mortem

Post-Mortem Issues:

    What is the root cause of the fault?
    Was there an alert in advance?
    Did it affect read?
    Did it affect write?
    Was there data loss?
    Is healing needed?
    How long did healing take?
    Is expansion needed?
    Is monitoring optimization needed?
    Is backup supplementation needed?
    Is change process optimization needed?

---

## Twenty-Five, Common Troubleshooting

### 25.1 admin info shows nodes offline

Troubleshoot:

    ping node IP
    ssh node
    docker ps -a
    docker logs minio
    systemctl status docker
    ss -lntp | grep 9000
    df -hT
    dmesg | tail -100

Common Causes:

    Node failure.
    Docker stopped.
    MinIO container exited.
    9000 port is occupied.
    Data directory is unavailable.
    Firewall block.
    Network failure.

---

### 25.2 admin info shows drive offline

Troubleshoot:

    df -hT
    lsblk
    mount | grep minio
    dmesg | tail -100
    journalctl -k | tail -100
    smartctl -a /dev/sdX
    docker logs minio

Common Causes:

    Disk failure.
    Mount point lost.
    File system error.
    Permission anomaly.
    Directory deleted.
    Disk read-only.
    Container mount path error.

---

### 25.3 Object Read Failure

Troubleshoot:

    mc stat
    mc cp
    mc admin info
    mc admin heal --dry-run
    Nginx error.log
    MinIO log

Possible Causes:

    Object does not exist.
    Insufficient permissions.
    Bucket written incorrectly.
    Prefix written incorrectly.
    Read quorum not met.
    Multiple disks or nodes abnormal.
    Reverse proxy anomaly.

---

### 25.4 Object Write Failure

Troubleshoot:

    mc admin info
    df -hT
    Nginx error.log
    docker logs minio
    mc ls
    User Policy

Possible Causes:

    Write quorum not met.
    Insufficient disk space.
    User lacks PutObject permission.
    Nginx upload size limit.
    Backend node offline.
    AccessKey / SecretKey error.
    Bucket does not exist.

---

### 25.5 Healing is very slow

Possible Causes:

    Too many objects.
    Too many small objects.
    Slow disk I/O.
    Insufficient network bandwidth.
    High node load.
    Large amount of business traffic.
    Recursive heal on entire large Bucket.

Handling Suggestions:

    Avoid business peak hours.
    Perform by Bucket / Prefix.
    Monitor disk and network metrics.
    Do not trigger multiple large-scale healings.
    Evaluate impact with monitoring.

---

## Twenty-Six, Advanced SRE Methodology

### 26.1 Fault Drill is Not Data Destruction

Correct fault drill should be:

    Targeted.
    Scoped.
    Pre-checks.
    Step-by-step operations.
    Observational metrics.
    Recovery steps.
    Verification results.
    Post-mortem records.

Incorrect ways:

    Arbitrarily delete directories.
    Arbitrarily stop nodes.
    No records.
    No recovery plan.
    No business confirmation.
    No backup.
    No monitoring.

---

### 26.2 Observe First, Fix Later

Do not perform destructive operations immediately after a fault occurs.

Prioritize:

    mc admin info
    health check
    docker logs
    df -hT
    lsblk
    Nginx logs
    mc admin heal --dry-run

Decide later:

    Whether to restart containers.
    Whether to recover nodes.
    Whether to replace disks.
    Whether to execute healing.
    Whether to switch entry points.
    Whether to pause writes.
    Whether to notify business.

---

### 26.3 High Availability Does Not Equal Zero Risk

MinIO distributed cluster improves availability, but still requires:

    Monitoring.
    Alerts.
    Backup.
    Permission control.
    Entry protection.
    Fault drills.
    Capacity planning.
    Change process.
    Regular recovery verification.

---

## Twenty-Seven, Experiment Cleanup

### 27.1 Delete Test Bucket

High-risk warning:

    Only delete experimental Buckets.
    Do not accidentally delete production Buckets.

Delete objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-admin/heal-demo

Delete Bucket: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure rb minio-admin/heal-demo

---

### 27.2 Clean up local test directory

    rm -rf /tmp/minio-heal-demo

---

### 27.3 Confirm simulated fault directory has been restored

On minio-node04 confirm:

    ls -ld /data/minio/disk1 /data/minio/disk2
    docker ps | grep minio
    docker logs --tail=50 minio

On client confirm:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

---

## Twenty-Eight, Production Data Protection Checklist

### 28.1 Cluster status check

| Check Item | Requirement | Result |
|---|---|---|
| Node status | All nodes online |  |
| Disk status | All disks online |  |
| API ready | health ready normal |  |
| Nginx backend | upstream normal |  |
| admin info | No abnormal offline |  |

---

### 28.2 Fault recovery check

| Check Item | Requirement | Result |
|---|---|---|
| Node failure process | Documented |  |
| Disk failure process | Documented |  |
| Disk replacement process | Reviewed |  |
| Healing process | Dry-run and execution strategy |  |
| Recovery verification | Object read/write verification |  |
| Fault post-mortem | Template available |  |

---

### 28.3 Backup check

| Check Item | Requirement | Result |
|---|---|---|
| Erasure Coding | Enabled but not used as backup |  |
| mc mirror | Cross-cluster orAlien. backup |  |
| Recovery drill | Regular execution |  |
| Delete protection | Access control |  |
| Key protection | AccessKey not leaked |  |
| Version control | Important Bucket can be enabled |  |

---

## Twenty-Nine, Interview Answer Outline

If asked in an interview:

    How does MinIO ensure data reliability? How to handle node or disk failures?

You can answer:

    MinIO primarily protects object data through Erasure Coding. It splits objects into data blocks and parity blocks, distributing them across multiple disks or nodes. When part of the disks or nodes fail, data can still be read or written as long as the remaining blocks meet the read quorum or write quorum.
    In production, I would not use single-node single-disk as a high availability solution. Instead, I would adopt multi-node multi-disk deployment, such as 4-node with multiple independent data disks, and expose HTTPS unified entry through Nginx or LB.
    If a node fails, I would first use mc admin info to check node and disk status, then check health interface, Docker containers, host network, disk mounts, and MinIO logs. After node recovery, I would observe admin info and healing status, and verify if objects can be normally uploaded/downloaded.
    For disk failure, I would first confirm whether it's disk damage, mount loss, or filesystem anomaly. When real disk damage occurs, new disks need to be replaced, mounted back to original data path, and MinIO would heal missing data through healing. Before executing healing, I would typically use mc admin heal --dry-run to check impact scope to avoid full repair on large buckets during business peak.
    It's important to emphasize that Erasure Coding is not backup. It can handle partial disk/node failures but cannot prevent accidental deletion, malicious deletion, data corruption by programs, or entire cluster failures. Therefore, production still requires mc mirror, cross-cluster sync,Alien. backup, access control, and regular recovery drills.

---

## Thirty, Summary of This Chapter

This article completed MinIO data protection and fault recovery practice:

1. MinIO uses Erasure Coding to protect object data.
2. Erasure Coding splits objects into data blocks and parity blocks.
3. Erasure Coding can tolerate partial disk or node failures.
4. Read quorum determines if objects can be read.
5. Write quorum determines if objects can be written.
6. After node failure, first observe admin info and health status.
7. Read and write operations during node failure need separate verification.
8. After node recovery, observe if healing exists.
9. mc admin heal --dry-run is suitable for initial scope observation.
10. mc admin heal --recursive can be used for test bucket repair, but cautious in production.
11. Single-disk directory failure experiment only simulates disk loss, not equivalent to real physical disk damage.
12. Real disk failure requires disk replacement, mount point recovery, and observe healing.
13. Do not manually modify MinIO data directory.
14. Do not arbitrarily delete data directory for testing in production.
15. Erasure Coding cannot replace backup.
16. Production still requires mc mirror, cross-cluster backup, access governance, and recovery drills.
17. Fault drills must have objectives, steps, verification, and post-mortem.
18. Subsequent learning will continue on MinIO monitoring and operations: Prometheus metrics, logs, and capacity management.

---

## Thirty-One, Reference Documents

MinIO official documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Erasure Coding documentation:

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

MinIO multi-node multi-disk deployment documentation:

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

MinIO mc admin heal documentation:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-heal.html

MinIO mc admin info documentation:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-info.html

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO monitoring documentation:

    https://min.io/docs/minio/linux/operations/monitoring.html

MinIO identity and access management:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html