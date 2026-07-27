# MinIO Data Protection: Erasure Coding, Node Failures, and Disk Failure Recovery

Recommended Path: 05-Storage/02-MinIO/08-MinIO Data Protection: Erasure Coding, Node Failures, and Disk Failure Recovery.md

Tags: #MinIO #ErasureCoding #DataProtection #NodeFailures #DiskFailures #FailureRecovery #Healing #ObjectStorage #S3 #AdvancedSRE #ProductionOps

---

## I. Document Overview

This article is the eighth in the MinIO series, focusing on MinIO's data protection mechanisms, node failures, disk failures, and recovery processes.

Previously covered topics include:

- Basics of MinIO Object Storage
- Single-machine single-disk deployment
- Single-node multiple-disk deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS access design
- Nginx HTTPS unified entry point
- mc client configuration and object operations
- User, Policy, AccessKey, and SecretKey permission management

This article delves into the core capabilities of MinIO Ops:

    How is data protected?
    What is Erasure Coding?
    Can reading and writing still be performed after a node failure?
    How to recover from a disk failure?
    What is healing?
    How to use mc admin heal?
    Can Erasure Coding replace backups?
    How to design fault recovery scenarios in a production environment?

This article emphasizes practicality and will conduct experiments using a 4-node MinIO distributed cluster.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of MinIO Erasure Coding.
2. Distinguish between Erasure Coding and multiple replicas.
3. Comprehend the impacts of node and disk failures on MinIO.
4. Grasp the basic concepts of read quorum and write quorum.
5. Use mc admin info to check node and disk status.
6. Utilize mc admin heal to monitor recovery progress.
7. Simulate single-node failures.
8. Verify object reading during a node failure.
9. Verify object writing during a node failure.
10. Restore failed nodes and observe cluster recovery.
11. Simulate single-disk directory failures.
12. Recover damaged disk directories and trigger or observe the healing process.
13. Identify which failures are tolerable and which lead to unavailability.
14. Recognize that Erasure Coding cannot replace backups.
15. Establish a production fault recovery process for MinIO.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues using the 4-node distributed MinIO cluster:

| IP | Host Name | Role | Data Directory |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.45 | minio-client | mc Client/Testing Client | - |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry Point | - |

---

### 3.2 Access Points

Backend direct access:

    http://10.0.0.41:9000

HTTPS unified entry point:

    https://s3.minio.local

Console access:

    https://console.minio.local

This article uses the default entry point:

    https://s3.minio.local

If a self-signed certificate is used in the experimental environment, use --insecure temporarily in mc commands.

Do not use --insecure for production environments.

---

### 3.3 Image Versions

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

## IV. Basic Understanding of Erasure Coding

### 4.1 What is Erasure Coding

Erasure Coding, also known as erasure coding, works on the principle of:

    Dividing```markdown
--insecure alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using a formal certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

---

### 7.3 Viewing Aliases

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

---

### 7.4 Viewing Cluster Information

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Requirements:

    All nodes must be online.
    All disks must be online.
    No drives should be marked as offline.
    The cluster must be able to read and write data properly.

---

## Section 8: Experiment 1: Health Check Before Fault Simulation

### 8.1 Checking MinIO Backend Health

Execute the following commands on minio-client or minio-entry:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "Checking if $ip is online"
      curl -I http://$ip:9000/minio/health/live
    done

Check the readiness status:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "Checking if $ip is ready"
      curl -I http://$ip:9000/minio/health/ready
    done

---

### 8.2 Checking Container Status

Execute the following commands on each of the 4 MinIO nodes:

    docker ps | grep minio
    docker logs --tail=50 minio

---

### 8.3 Checking Disk Capacity

Execute the following commands on each of the 4 MinIO nodes:

    df -hT
    du -sh /data/minio/disk1
    du -sh /data/minio/disk2
    lsblk

Key checks include:

    The data directory must exist.
    The data directory must be writable.
    The system disk should not be full.
    There should be no mismounted or lost data directories.

---

### 8.4 Checking Time Synchronization

Execute the following command on each of the 4 MinIO nodes:

    timedatectl

If the times are inconsistent, resolve the time synchronization issue first.

---

## Section 9: Experiment 2: Creating a Data Protection Test Bucket

### 9.1 Creating a Bucket

Create a test bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-admin/heal-demo

View the bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-admin

---

### 9.2 Preparing Test Objects

Create a small file:

    echo "hello minio healing demo" > /tmp/minio-heal-demo/upload/hello.txt

Create a large file:

    dd if=/dev/zero of=/tmp/minio-heal-demo/upload/file-100m.bin bs=1M count=100

Create multiple objects:

    for i in $(seq 1 20); do
      echo "object $i for minio healing demo" > /tmp/minio-heal-demo/upload/object-$i.txt
    done

View the files created:

sha256sum /tmp/minio-heal-demo/check/file-100m-before.bin

Save the output to the log file:

sha256sum /tmp/minio-heal-demo/check/hello-before.txt > /tmp/minio-heal-demo/check/checksum-before.txt
sha256sum /tmp/minio-heal-demo/check/file-100m-before.bin >> /tmp/minio-heal-demo/check/checksum-before.txt

View:

cat /tmp/minio-heal-demo/check/checksum-before.txt

Explanation:

After the failure is recovered, the object can be downloaded again and its checksum can be calculated. If the checksums match, it indicates that the content of the object has not changed.--insecure ls minio-admin/heal-demo

If the upload is successful:

The current cluster still meets the write quorum requirement. After the failed node is restored, relevant data may need to be synchronized or repaired.

If the upload fails:

Under the current faulty condition, the write quorum may not be met. It is necessary to check the admin information and error messages.      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/upload/write-disk2-failed.txt minio-admin/heal-demo/write-disk2-failed.txt

If the upload is successful:

    The current cluster still meets the write quorum requirement.

If the upload fails:

    The current failure state affects writing operations; it needs to be assessed in conjunction with the admin info.```markdown
Start a MinIO container.
Stop a test node.
Simulate the absence of a single disk directory.
Upload and download test objects.
Execute a heal dry-run.
Perform a heal on the test Bucket.
```

---

### 21.2 High-Risk Experiment Content

Use with caution in a production environment:

- Delete data directories.
- Format disks.
- Stop multiple nodes simultaneously.
- Remove multiple disks at once.
- Modify cluster endpoints.
- Change data directory paths.
- Perform a full recursive heal on extremely large Buckets.
- Execute healing operations during business peak hours.
- Conduct destructive tests without any backup in place.
```

---

### 21.3 Must-Confirm Items Before the Experiment

Before starting the experiment, ensure the following:

- It is a test environment.
- Complete backups are available.
- There is sufficient business window time for the experiment.
- You know the recovery procedures.
- A rollback plan is in place.
- Monitoring systems are properly set up for observation.
- Someone will review the experiment process.
- The start and end times of the experiment are clearly recorded.
```

---

## Chapter 22: Erasure Coding Cannot Replace Backups

### 22.1 What Erasure Coding Can Solve

It can address or mitigate:

- Single-disk failures.
- Single-node failures.
- Loss of some data blocks.
- Local hardware damage.
- Temporary node outages.
```

---

### 22.2 What Erasure Coding Cannot Solve

It cannot resolve:

- Accidental deletion of objects or Buckets.
- Incorrect overwriting of files by applications.
- Administrators using the `rm --recursive --force` command.
- Ransomware attacks.
- Complete damage to the MinIO cluster.
- Data center failures.
- Situations where multiple nodes exceed their tolerance limits simultaneously.
- Malicious data deletion by a compromised root user account.
```

---

### 22.3 Production Environments Must Include Additional Backups

Production systems should include additional backup measures such as:

- Using `mc mirror` to replicate data to another MinIO cluster.
- Mirroring data to cloud object storage services provided by vendors.
- Performing cross-cluster data replication.
- Regular offline backups.
- Implementing Bucket version control.
- Using lifecycle management strategies.
- Enabling delete protection mechanisms.
- Conducting regular recovery drills.

Detailed information on backup and migration methods will be provided in the subsequent document:

**10-MinIO Backup Migration: mc mirror, Cross-Cluster Synchronization, and Data Migration.md**

---

## Chapter 23: Monitoring and Alerting Recommendations

### 23.1 Node Alerts

Alerts should be triggered for the following situations:

- MinIO nodes go offline.
- The MinIO API becomes unavailable.
- Failed checks on the `MinIO ready` status.
- Issues with Nginx upstream connections.
- Abnormal CPU or memory usage on nodes.
- Network-related problems with nodes.
```

---

### 23.2 Disk Alerts

Alerts are necessary when:

- Disks go offline.
- Disk capacity exceeds 80% or 90%.
- The disk becomes read-only.
- I/O errors occur on disks.
- Inode abnormalities are detected.
- Mount points are lost.
- SMART monitoring indicators show issues.
```

---

### 23.3 Object Storage Alerts

Alerts should be issued for the following situations:

- Abnormal increase in Bucket capacity or the number of objects stored.
- A significant rise in API errors with code 5xx.
- Increased API response times.
- A large number of `DeleteObject` requests.
- A surge in `AccessDenied` incidents.
- Healing processes taking an unusually long time to complete.
- Failed backup mirroring attempts.
```

---

### 23.4 Security Alerts

Pay attention to the following alerts:

- Frequent access attempts by the root user account.
- Abnormal logins from the Console.
- Unrecognized IP addresses making requests.
- Large-scale data downloads or deletions.
- An unusually high number of `AccessKey` usage failures.
- Nginx errors with codes 4xx or 5xx.
```

---

## Chapter 24: Production Fault Handling Record Templates

### 24.1 Basic Fault Information

| Item          | Details                  |
|---------------|-------------------------|
| Fault Time    |                          |
| Discovery Method | Alert / User Report / Inspection |
| Impact Scope   |                          |
| Affected Buckets |                          |
| Business Implications |                          |
| Reading Affected |                          |
| Writing Affected |                          |
| Current Cluster Status |                          |
```

---

### 24.2 Fault Symptoms

Record the following details:

- Output from `admin info`.
- Results of health check tests.
- Nginx error logs.
- MinIO log files.
- System logs of affected nodes.
- Disk status information.
- User-reported issues.
```

---

### ```markdown
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure rb minio-admin/heal-demo

---

### 27.2 Clear the local test directory

    rm -rf /tmp/minio-heal-demo

---

### 27.3 Verify that the simulated fault directory has been restored

On minio-node04:

    ls -ld /data/minio/disk1 /data/minio/disk2
    docker ps | grep minio
    docker logs --tail=50 minio

On the client:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

---

## Section 28: Production Data Protection Checklist

### 28.1 Cluster Status Inspection

| Inspection Item | Requirement | Result |
|---|---|---|
| Node Status | All nodes must be online | |
| Disk Status | All disks must be online | |
| API Readiness | Health check must indicate normal status | |
| Nginx Backend | Upstream connections must be functioning normally | |
| Admin Information | No abnormal offline instances | |

---

### 28.2 Disaster Recovery Inspection

| Inspection Item | Requirement | Result |
|---|---|---|
| Node Failure Process | Documentation must exist | |
| Disk Failure Process | Documentation must exist | |
| Disk Replacement Process | Revisions must be documented | |
| Healing Process | Dry-run and execution strategies must be in place | |
| Recovery Verification | Object read and write tests must be conducted | |
| Disaster Review | Templates must be available for analysis | |

---

### 28.3 Backup Inspection

| Inspection Item | Requirement | Result |
|---|---|---|
| Erasure Coding | Must be enabled but not considered a backup solution | |
| MC Mirror | Cross-cluster or off-site backups must be implemented | |
| Recovery Drills | Must be performed regularly | |
| Deletion Protection | Access controls must be in place | |
| Key Protection | AccessKeys must not be compromised | |
| Version Control | Important Buckets should consider enabling versioning | |

---

## Section 29: Interview Answer Guidelines

If interviewed about how MinIO ensures data reliability and how it handles node or disk failures, you can respond as follows:

MinIO primarily uses Erasure Coding to protect object data. It divides objects into data blocks and parity blocks, which are distributed across multiple disks or nodes. In the event of a failure in some disks or nodes, as long as the remaining blocks meet the read quorum or write quorum requirements, data can still be accessed or written.

In production environments, I would never rely on a single node or disk for high availability. Instead, I would deploy multiple nodes and disks, such as 4 nodes with independent data disks. I would then use Nginx or a load balancer to provide a unified HTTPS interface.

If a node fails, I would first check the status of the node and disks using `mc admin info`. I would also verify the health status through APIs, Docker containers, network connections, disk mountings, and MinIO logs. Once the node is restored, I would recheck the status and ensure that objects can be uploaded and downloaded normally.

In the case of a disk failure, I would first determine whether the issue was due to physical damage, lost mounting, or file system abnormalities. If it was a physical disk failure, I would replace it with a new one, mount it back to the original location, and then use Minio's healing process to restore missing data. Before executing the healing process, I would typically perform a `mc admin heal --dry-run` to assess the impact, avoiding full restoration during peak business hours for large Buckets.

It is important to note that Erasure Coding is not a backup solution. While it can help handle partial disk or node failures, it cannot prevent accidental deletions, malicious actions, data corruption by programs, or entire cluster failures. Therefore, additional measures such as MC mirrors, cross-cluster synchronization, off-site backups, access control, and regular recovery drills are still necessary in production environments.

---

## Section 30: Summary of This Article

This article has covered the practical aspects of MinIO data protection and disaster recovery:

1. MinIO uses Erasure Coding to protect object data.
2. Erasure Coding divides objects into data blocks and parity blocks for distribution across multiple disks or nodes.
3. It can tolerate partial disk or node failures.
4. The read qu