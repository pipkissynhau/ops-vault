# MinIO Backup and Migration: mc mirror, Cross-Cluster Synchronization, and Data Transfer

Recommended Path: 05-Storage/02-MinIO/10-MinIO Backup and Migration: mc mirror, Cross-Cluster Synchronization, and Data Transfer.md

Tags: #MinIO #Backup and Migration #mc #mirror #Object Storage #S3 #Cross-Cluster Synchronization #Data Transfer #Disaster Recovery #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the tenth in the MinIO series, focusing on backup and migration of MinIO, mc mirror synchronization, cross-cluster migration, and data verification.

What has been covered previously includes:

- Basics of MinIO Object Storage
- Single-machine single-disk deployment
- Single-node multiple-disk deployment
- 4-node multi-disk distributed cluster deployment
- Design of internal HTTP and external HTTPS access points
- Nginx HTTPS unified entry point
- mc client configuration, Bucket management, and object operations
- User, Policy, AccessKey, and SecretKey permission management
- Erasure Coding, node failure, and disk failure recovery
- Prometheus metrics, logging, and capacity management

This article delves into the critical aspects of protecting production data with MinIO.

Key issues addressed include:

    Is backup still necessary even with Erasure Coding?
    What can mc mirror do?
    What is the difference between mc mirror and regular mc cp?
    How to synchronize a local directory to MinIO?
    How to back up a MinIO Bucket to local storage?
    How to transfer one MinIO cluster to another?
    How to perform capacity assessment before migration?
    How to verify data after migration?
    Why is `mc mirror --remove` dangerous?
    Can `mc mirror --watch` be used for production-level replication?
    How to choose between mc mirror, Bucket Replication, and Site Replication?
    How to design an object storage backup and migration process in a production environment?

This article emphasizes practicality, providing replicable commands for all core processes.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand that Erasure Coding does not replace backup.
2. Comprehend the role and limitations of mc mirror.
3. Use mc mirror to synchronize local directories with MinIO Buckets.
4. Use mc mirror to back up MinIO Buckets to local directories.
5. Use mc mirror to transfer one MinIO Bucket to another.
6. Migrate one MinIO cluster to another.
7. Handle target objects that already exist using `--overwrite`.
8. Recognize the risks associated with `--remove`.
9. Understand the appropriate use cases and risks of `--watch`.
10. Use `--dry-run` to preview synchronization operations.
11. Generate synchronization summaries using `--summary`.
12. Filter objects with `--exclude`/`--include`.
13. Control bandwidth with `--limit-upload`/`--limit-download`.
14. Verify Bucket capacity and object counts before and after migration.
15. Write basic backup scripts.
16. Design a production-grade MinIO backup and migration process.
17. Distinguish between the uses of mc mirror, Bucket Replication, and Site Replication.

---

## III. Experimental Environment

### 3.1 Source MinIO Cluster

The source MinIO cluster continues from the previous 4-node distributed setup:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO source cluster node 1 |
| 10.0.0.42 | minio-node02 | MinIO source cluster node 2 |
| 10.0.0.43 | minio-node03 | MinIO source cluster node 3 |
| 10.0.0.44 | minio-node04 | MinIO source cluster node 4 |
| 10.0.0.46 | minio-entry | Source cluster Nginx HTTPS unified entry |

Source cluster API:

    https://s3.minio.local

Source cluster Console:

    https://console.minio.local

---

### 3.2 Backup MinIO Experimental Node

To demonstrate cross-cluster synchronization, an additional standalone MinIO instance is started on the minio-client node as a “backup cluster”.

| IP | Host Name | Purpose | Ports |
|---|---|---|---|
| 10.0.0.45 | minio-client | mc client / backup MinIO experimental node | 19000 / 19001 |

Backup cluster API:

    http://10.0.0.45:19000

Backup cluster Console:

    http://10.0.0.45:19001

Note:

    This backup MinIO| Experiment 1 | Prepare the mc management environment |
| Experiment 2 | Start the backup MinIO experimental instance |
| Experiment 3 | Configure aliases for the source and backup clusters |
| Experiment 4 | Create a source Bucket and a target Bucket |
| Experiment 5 | Synchronize local directories to the source MinIO |
| Experiment 6 | Back up the source MinIO Bucket to local storage |
| Experiment 7 | Synchronize the source Bucket to the backup Bucket |
| Experiment 8 | Use --dry-run to perform a simulated synchronization |
| Experiment 9 | Use --overwrite to handle object updates |
| Experiment 10 | Demonstrate high-risk deletion synchronization using --remove |
| Experiment 11 | Filter synchronization using include/exclude options |
| Experiment 12 | Control the impact of synchronization by setting bandwidth limits |
| Experiment 13 | Write basic backup scripts |
| Experiment 14 | Verify data after migration |
| Experiment 15 | Design a production backup migration process |

---

## VII. Experiment 1: Prepare the mc management environment

### 7.1 Create directories

Execute on the minio-client node:

    mkdir -p /data/minio/mc-config
    mkdir -p /tmp/minio-mirror-demo/source
    mkdir -p /tmp/minio-mirror-demo/restore
    mkdir -p /tmp/minio-mirror-demo/reports
    mkdir -p /tmp/minio-mirror-demo/scripts

---

### 7.2 Pull images

Pull the mc image:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Pull the MinIO server image:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Check the images installed:

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq

---

### 7.3 Configure command variables

Define them in the current shell:

    export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    export MC_CONFIG=/data/minio/mc-config
    export MC WORKDIR=/tmp/minio-mirror-demo

    The `mcx()` function is effective only within the current shell session. It needs to be redefined after reopening a terminal. In production scripts, it is recommended to either include the full command or encapsulate it into a dedicated script.

---

### 7.4 Check the mc version

Run the following command:

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

## VIII. Experiment 2: Start the backup MinIO experimental instance

### 8.1 Create a backup data directory

Execute on the minio-client node:

    mkdir -p /data/minio-backup/data

Verify the directory creation:

    ls -ld /data/minio-backup/data

---

### 8.2 Start the backup MinIO service

Run the following command:

    docker run -d \
      --name minio-backup \
      --restart unless-stopped \
      -p 19000:9000 \
      -p 19001:9001 \
      -e MINIO_ROOT_USER=backupadmin \
      -e MINIO_ROOT_PASSWORD='BackupAdmin@123456' \
      -v /data/minio-backup/data:/data \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-22T22-12-26Z \
      server /data --console-address ":9001"

---

### 8.3 Check the status of the backup MinIO service

View the running containers:

    docker ps | grep minio-backup

Check the logs:

    docker logs --tail=100 minio-backup

Verify the API status:

    curl -I http://10.0.0.45:19000/minio/health/live
    curl -I http://10.0.0.45:19000/minio/health/ready

Access the console:

    http://10.0.0.45:19001

Login with:

    Username: backupadmin
    Password: BackupAdmin@123456

---

###      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-src

---

### 10.2 Creating the Target Bucket

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-bak/mirror-bak-demo

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-bak

---

## Chapter Eleven, Experiment Five: Synchronizing Local Directories to Source MinIO

### 11.1 Preparing the Local Source Directory

Create directories:

    mkdir -p /tmp/minio-mirror-demo/source/app
    mkdir -p /tmp/minio-mirror-demo/source/logs/nginx/2026/04/28
    mkdir -p /tmp/minio-mirror-demo/source/backup/mysql
    mkdir -p /tmp/minio-mirror-demo/source/tmp

Create files:

    echo "app config v1" > /tmp/minio-mirror-demo/source/app/config.txt
    echo "nginx access log line 1" > /tmp/minio-mirror-demo/source/logs/nginx/2026/04/28/access.log
    echo "mysql backup demo" > /tmp/minio-mirror-demo/source/backup/mysql/mysql-full.sql
    echo "temporary file should be excluded later" > /tmp/minio-mirror-demo/source/tmp/temp.txt

Create a large file:

    dd if=/dev/zero of=/tmp/minio-mirror-demo/source/backup/mysql/mysql-full-100m.sql bs=1M count=100

Check:

    tree /tmp/minio-mirror-demo/source
    du -sh /tmp/minio-mirror-demo/source

---

### 11.2 Preparing the Local Directory Synchronization

Use --dry-run for a trial run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --dry-run /demo/source/ minio-src/mirror-src-demo/

Note:

    --dry-run does not actually perform the synchronization.
    It is used to confirm that the source and target are set up correctly.
    It is recommended to perform a dry-run before executing the actual synchronization, especially when using options like --remove or --overwrite.

---

### 11.3 Performing the Local Directory Synchronization

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --summary /demo/source/ minio-src/mirror-source-demo/

---

### 11.4 Checking the Source Bucket Objects

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-src/mirror-source-demo

Check the capacity:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-src/mirror-source-demo

---

## Chapter Twelve, Experiment Six: Backing Up Source MinIO Bucket to Local Directory

### 12.1 Creating the Local Restoration Directory

    mkdir -p /tmp/minio-mirror-demo/restore/from-src

---

### 12.2 Preparing the Bucket Synchronization to Local

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.```markdown
--insecure mirror --summary minio-src/mirror-src-demo/ minio-bak/mirror-bak-demo/

---

### 13.3 Viewing the Target Bucket

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo

To view the capacity, run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      du minio-bak/mirror-bak-demo

---

### 13.4 Downloading the Target Objects for Verification

Create a directory:

    mkdir -p /tmp/minio-mirror-demo/restore/from-backup

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-bak/mirror-bak-demo/app/config.txt /demo/restore/from-backup/config.txt

View the content:

    cat /tmp/minio-mirror-demo/restore/from-backup/config.txt

Expected output:

    app config v1

---

## Chapter Fourteen: Experiment Eight: Using --overwrite for Object Updates

### 14.1 Modifying the Source File

Modify the local source file:

    echo "app config v2" > /tmp/minio-mirror-demo/source/app/config.txt

Synchronize the local changes to the source Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --overwrite --summary /demo/source/ minio-src/mirror-source-demo/

View the content of the source file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cat minio-src/mirror-source-demo/app/config.txt

---

### 14.2 Preparing for Overwriting and Synchronizing to the Backup Cluster

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --overwrite --dry-run minio-src/mirror-source-demo/ minio-bak/mirror-bak-demo/

---

### 14.3 Performing the Overwriting and Synchronization

Run the following command:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mirror --overwrite --summary minio-src/mirror-source-demo/ minio-bak/mirror-bak-demo/

View the content of the backup Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cat minio-bak/mirror-bak-demo/app/config.txt

Expected output:

    app config v2

---

### 14.4 Explanation of --overwrite

--overwrite means that if the content of an object on the source side changes, it is allowed to overwrite the corresponding object on the target side.

Risks:

    The historical content on the target side will be overwritten.
    If a wrong file is synchronized from the source side, the target side will also contain the wrong file.
    Without version control or independent backups, it will beExpected:

The file `target-only.txt` has been deleted.

---

### 15.5 Principles for Using --remove in Production

Before using `--remove` in production, it is necessary to confirm the following:

- Whether the source and target are correct.
- Whether it is a production bucket.
- Whether there is a risk of accidental deletion.
- Whether version control is in place.
- Whether independent backups exist.
- Whether a dry-run has been conducted.
- Whether approval has been obtained.
- Whether double-checking has been performed.
- Whether a rollback plan is available.

Recommendations:

- Do not use `--remove` for routine backups.
- It can be used before migration when the target state is clear.
- Do not use `--remove` for archived backups.
- If the target is meant to retain history, do not use `--remove`.

---

## Chapter Sixteen: Experiment Ten: Using Include/Exclude Rules for Filtered Synchronization

### 16.1 Filtering the tmp Directory

The current source directory contains:

`tmp/temp.txt`

If you do not want to synchronize the `tmp` directory, you can use `--exclude`.

Preview:

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --exclude "tmp/*" --dry-run /demo/source/ minio-src/mirror-source-demo-filtered/
```

Execution:

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --exclude "tmp/*" --summary /demo/source/ minio-src/mirror-source-demo-filtered/
```

View:

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-src/mirror-source-demo-filtered
```

---

### 16.2 Synchronizing Only Log Files

To synchronize only `.log` files:

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mirror-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --include "*.log" --summary /demo/source/ minio-src/mirror-logs-only/
```

View:

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-src/mirror-logs-only
```

---

### 16.3 Recommendations for Using Filter Parameters

Production recommendations:

- Always perform a dry-run with include/exclude rules first.
- Verify wildcard rules in a test environment.
- Avoid using complex filter rules for the first time in production.
- Record the filter conditions before critical migrations.
- After migration, sample the results to ensure they meet expectations.

---

## Chapter Seventeen: Experiment Eleven: Using Bandwidth Limits to Control Synchronization Impact

### 17.1 Why Limit Speeds

Large-scale synchronization can affect:

- Disk I/O of the source cluster.
- Disk I/O of the target cluster.
- Network bandwidth of the source cluster.
- Network bandwidth of the target cluster.
- Nginx inbound traffic.
- Normal business upload and download operations.

In production, it is not recommended to use unlimited speeds during peak periods for large-scale bucket migrations.

---

### 17.2 Limiting Upload Bandwidth

Example:

```bash
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror --limit-upload 20MiB --summary minio-src/mirror### Batch Copy Task
Specialized Backup System

---

## Chapter Nineteen, Experiment Thirteen: Data Verification Before and After Migration

### 19.1 Verification Objectives

After migration, it is essential to confirm the following:

    The source Bucket exists.
    The target Bucket exists.
    The capacity of the source Bucket is similar to that of the target Bucket.
    Source objects can be downloaded normally.
    Target objects can be downloaded normally.
    The content of key objects remains consistent.
    Business applications can use the new Endpoint.
    Permissions after migration are as expected.

---

### 19.2 Checking Source and Target Capacities

On the source side:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-src/mirror-src-demo

On the target side:

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
      --insecure find minio-src/mirror-source-demo > /tmp/minio-mirror-demo/reports/src-objects.txt

Target object list:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-bak/mirror-bak-demo > /tmp/minio-mirror-demo/reports/bak-objects.txt

To check:

    wc -l /tmp/minio-mirror-demo/reports/src-objects.txt
    wc -l /tmp/minio-mirror-demo/reports/bak-objects.txt

Note:

    Due to different path prefixes, a direct `diff` comparison may not yield consistent results. It is recommended to use `sed` to remove the alias/bucket prefixes before comparing.

---

### 19.4 Sampling Download Verification

Download the source object:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-src/mirror-source-demo/app/config.txt /demo/reports/src-config.txt

Download the target object:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-bak/mirror-bak-demo/app/config.txt /demo/reports/bak-config.txt

Calculate the verification sums:

    sha256sum /tmp/minio-mirror-demo/reports/src-config.txt
    sha256sum /tmp/minio-mirror-demo/reports/bak-config.txt

Compare the contents:

    diff /tmp/minio-mirror-demo/reports/src-config.txt /tmp/minio-mirror-demo/reports/bak-config.txt

---

### 19.5 Large File Verification

Download the source large file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mirror-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-src/mirror-source-demo/backup/mysql/mysql-full-100m.sql /demo/reports/src-100m.sql

Download the target large file:

    docker runcp /tmp/minio-mirror-demo/scripts/minio-bucket-backup.sh /usr/local/bin/minio-bucket-backup.sh

---

### 20.3 Execute the Script

Run:

    /usr/local/bin/minio-bucket-backup.sh

Check logs:

    ls -lh /var/log/minio-backup/
    tail -100 /var/log/minio-backup/mirror-src-demo-*.log

---

### 20.4 Schedule Cron Jobs for Regular Backups

Edit:

    crontab -e

Example: Execute once every 2 a.m.

    0 2 * * * /usr/local/bin/minio-bucket-backup.sh >/dev/null 2>&1

Production Recommendations:

    Backup tasks must have failure alerts.
    Don't just set up cron jobs without checking results.
    Logs should be integrated into Loki/ELK systems.
    Backup success rates should be monitored.
    Regular recovery drills should be conducted.

---

## Chapter Twenty-One: Production Migration Process Design

### 21.1 Pre-Migration Assessment

Before migrating, it is essential to confirm:

    Source cluster version.
    Target cluster version.
    Number of source buckets.
    Target bucket planning.
    Total capacity.
    Number of objects.
    Number of large objects.
    Number of small objects.
    Network bandwidth.
    Business read and write patterns.
    Whether write operations can be paused.
    Whether incremental synchronization is required.
    Whether version history needs to be retained.
    Whether migration policies need to be adjusted.
    Whether users and AccessKeys need to be migrated.
    Whether a rollback plan is in place.

---

### 21.2 Pre-Migration Commands

View source buckets:

    mc ls minio-src

Check capacity:

    mc du minio-src/<bucket>

View objects:

    mc find minio-src/<bucket>

Check permissions:

    mc admin user list minio-src
    mc admin policy list minio-src

Note:

    User configurations, policies, bucket settings, and object data are different aspects of the migration process.
    `mc mirror` is primarily used for synchronizing object data.
    IAM roles, policies, bucket versions, and replication rules need to be planned separately.

---

### 21.3 Migration Steps

Recommended procedure:

    1. Create a target cluster.
    2. Create target buckets.
    3. Set up target users and policies.
    4. First, perform a `mc mirror --dry-run`.
    5. Execute the initial full synchronization.
    6. Perform multiple incremental synchronizations.
    7. Pause business write operations or enter maintenance mode.
    8. Carry out the final incremental synchronization.
    9. Verify the number of objects and their capacity.
    10. Download key objects for verification.
    11. Switch the application endpoint.
    12. Test business read and write functions.
    13. Retain the source cluster for a certain period.
    14. Conduct a post-migration review.

---

### 21.4 Post-Migration Verification

After migration, confirm:

    The application can upload objects successfully.
    The application can download objects successfully.
    Bucket permissions are correct.
    Large file uploads are functioning properly.
    Nginx is working correctly at the entry point.
    HTTPS certificates are valid.
    `mc admin info` shows no issues.
    Prometheus monitoring is accurate.
    Alert rules are functioning properly.
    Backup tasks are running smoothly.
    The retention period for the source cluster is clear.

---

### 21.5 Rollback Plan

A rollback plan is essential for any migration:

    Retain the source cluster and buckets.
    Keep the original application endpoint configurations.
    Preserve the original AccessKeys.
    Maintain a backup of the configuration before switching.
    Define DNS TTL settings clearly.
    Identify the person responsible for rolling back.
    Clearly define the verification items for rollback.

Principle:

    No migration should proceed without a rollback plan.
    Migration is not considered complete without data verification.

---

## Chapter Twenty-Two: Comparison of `mc mirror`, Bucket Replication, and Site Replication

### 22.1 `mc mirror`

Suitable for:

    Manual migrations.
    Regular backups.
    Temporary synchronizations.
    Synchronization between local directories and buckets.
    Medium to small-scale migrations.
    One-time migrations.
    Post-migration data reconciliation.

Features:

    Client-driven synchronization.
    Depends on the machine running `mc`.
    Requires scripts, logs, and monitoring.
    Does not preserve full object version history and metadata.
    Flexible in usage.

---

### 22.2 Bucket Replication

Suitable for:

    Automated synchronization of specific buckets.
    From source buckets to target buckets.
    Continuous replication.
    Cross-cluster object synchronization.
    More formal object-level replication.

Features:

    Server-side replication.
    Better suited for long-term replication tasks than `The source cluster and the backup cluster should be isolated from each other. The backup cluster must not share any single points of failure with the source cluster. Backup account permissions should be minimized, and ordinary business operations are not allowed to write to the backup bucket. Backup data should have a specified retention period, and there should be logs for backup tasks. Alerts should be triggered in case of backup failures, and regular recovery verifications should be conducted.

---

### 25.2 Backup Frequency Design

Based on the importance of the data, the following frequency recommendations are provided:

| Data Type | Recommended Frequency |
|-------------|-------------------------|
| User-uploaded Attachments | Hourly or Daily |
| Critical Business Files | Hourly or More Frequent |
| Log Archival | Daily |
| Backup Archival | Daily or According to Backup Strategy |
| CI/CD Artifacts | Daily or Per Version Retention |
| Temporary Files | May Not Need Backups or Backups Should Be Infrequent |

---

### 25.3 Retention Period Design

The following retention periods are suggested:

- 7 days for short-term recovery.
- 30 days for accidental deletion recovery.
- 90 days for audit purposes.
- 180 days or longer for compliance requirements.

The specific retention period should depend on factors such as business needs, compliance obligations, storage costs, recovery objectives, and data growth rates.

---

### 25.4 Recovery Exercises

If the backup is not verified during recovery, it means the entire process has not been successfully completed. Recovery exercises should ensure that objects can be downloaded from the backup bucket, restored to a new bucket, key prefixes can be recovered, large files can be restored, and the directory structure required by the business applications can be accurately reproduced. The recovery process should be documented, and someone must be responsible for executing the recovery operations.

---

## Chapter 26: Experimental Cleanup

### 26.1 Deleting the Source Experimental Bucket

High-risk warning:

- Only delete the experimental bucket; never accidentally remove the production bucket.
- To delete objects on the source side:
  ```
  docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-src/mirror-src-demo
  ```
- To delete the source bucket:
  ```
  docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-src/mirror-source-demo
  ```
- To delete filtered experimental buckets:
  ```
  docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-src/mirror-source-demo-filtered
  ```
  ```
  docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-src/mirror-source-demo-filtered
  ```
- To delete log experimental buckets:
  ```
  docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-src/mirror-logs-only
  ```
  ```
  docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-src/mirror-logs-only
  ```

---

### 26.2 Deleting the Backup Experimental Bucket

To delete objects on the backup side:
```
docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-4. mc mirror supports the synchronization of Buckets to local directories.
5. mc mirror allows for the synchronization between MinIO clusters.
6. Before performing a sync, it is recommended to use the --dry-run option for a trial run.
7. The --summary option can generate a summary of the synchronization process.
8. Using the --overwrite option will overwrite objects with the same name on the target side.
9. The --remove option will delete objects that do not exist on the target side; this is considered a high-risk setting.
10. The --watch option enables continuous synchronization, but it does not constitute a complete disaster recovery system.
11. The include and exclude options can be used to filter objects during synchronization.
12. The limit-upload and limit-download options allow control over the synchronization bandwidth.
13. The max-workers option helps to manage the number of concurrent tasks.
14. Before performing any migration, it is essential to evaluate factors such as capacity, number of objects, permissions, network connectivity, and backup recovery plans.
15. After migration, it is necessary to verify the capacity, number of objects, and the content of critical objects.
16. Production backup tasks must include logging, alerts, and recovery drills.
17. mc mirror is suitable for manual migrations, scheduled backups, and temporary sync operations.
18. For long-term, production-grade cross-cluster replication, consider using Bucket Replication or Site Replication.
19. Next, we will summarize the MinIO phase: from S3 object storage to a production-ready cluster.

---

## Thirty, References

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