# MinIO Client Tool: mc Configuration, Bucket Management, and Object Operations

Suggested path: 05-Storage/02-MinIO/06-MinIO Client Tool: mc Configuration, Bucket Management, and Object Operations.md

Tags: #MinIO #mc #ObjectStorage #S3 #Bucket #Object #ClientTool #UploadDownload #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the sixth article of the MinIO module, focusing on learning the MinIO official client tool mc's configuration, Bucket management, and object operations.

Previously completed:

- MinIO object storage basics
- Single-node single-disk deployment
- Single-node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS access entry design
- Nginx HTTPS unified entry configuration

This article enters the MinIO daily operation phase.

mc is one of the most commonly used command-line tools for MinIO operations, similar to kubectl in the object storage field.

It can be used for:

    Configuring MinIO connection aliases
    Viewing cluster information
    Creating Buckets
    Deleting Buckets
    Uploading objects
    Downloading objects
    Viewing object metadata
    Viewing Bucket capacity
    Recursively listing objects
    Deleting objects
    Synchronizing directories
    mirror backup migration
    Managing users, Policies, lifecycle, versions, and other advanced capabilities

This article will first cover the basic high-frequency capabilities:

    mc alias
    mc admin info
    mc mb
    mc ls
    mc cp
    mc stat
    mc du
    mc find
    mc rm
    mc rb
    mc mirror basic usage

Permissions, users, and Policies will be covered in the next article:

    07-MinIO Permission Management: Users, Policies, AccessKey and SecretKey.md

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand mc's role in MinIO operations.
2. Run mc via Docker without relying on local installation.
3. Configure MinIO backend HTTP alias.
4. Configure MinIO HTTPS unified entry alias.
5. Use mc to view MinIO cluster information.
6. Use mc to create Buckets.
7. Use mc to upload files to Buckets.
8. Use mc to download files from Buckets.
9. Use mc to view object metadata.
10. Use mc to view Bucket capacity and object count.
11. Use mc to recursively list objects.
12. Use mc to delete individual objects, Prefix objects, and Buckets.
13. Use mc mirror for basic directory synchronization.
14. Master common mc error troubleshooting methods.
15. Understand the high-risk boundaries of mc operations in production environments.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues from the previous MinIO distributed cluster:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc Client |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry |

---

### 3.2 Access Entries

Backend direct access entry:

    http://10.0.0.41:9000

HTTPS unified entry:

    https://s3.minio.local

Console entry:

    https://console.minio.local

Notes:

    mc connects to the S3 API entry.
    mc should connect to the 9000 API port or HTTPS API domain.
    Do not configure the 9001 Console address for mc.

---

### 3.3 Image Version

mc Client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Source images:

    minio/mc:RELEASE.2025-04-16T18-13-26Z
    minio/minio:RELEASE.2025-04-22T22-12-26Z

Notes:

    This module uses fixed versions uniformly to avoid inconsistencies caused by latest changes in commands, outputs, and experimental results.
    The mc version is close in time to the MinIO server version for easier experimental reproducibility.

---

## IV. mc Tool Positioning

### 4.1 What is mc

mc is the command-line tool for MinIO Client.

It can be understood as:

    Object storage operation tool
    S3 management tool
    MinIO operation and maintenance tool
    Bucket and Object operation tool

Similar relationships:

| Scenario | Tool |
|---|---|
| Kubernetes | kubectl |
| Docker | docker CLI |
| Linux Files | cp / ls / rm |
| S3 / MinIO | mc |
| AWS S3 | aws cli |

---

### 4.2 What mc Can Do

mc common capabilities:

| Command | Function |
|---|---|
| mc alias | Configure object storage connection |
| mc admin info | View MinIO cluster information |
| mc mb | Create Bucket |
| mc rb | Delete Bucket |
| mc ls | List Buckets or objects |
| mc cp | Upload or download objects |
| mc stat | View object metadata |
| mc du | View capacity usage |
| mc find | Search objects |
| mc rm | Delete objects |
| mc mirror | Mirror synchronization |
| mc cat | View object content |
| mc version | View version |

---

### 4.3 mc Connection to Objects

mc connects to the Console.

mc connects to the MinIO S3 API address.

Correct:

    http://10.0.0.41:9000
    https://s3.minio.local

Incorrect:

    http://10.0.0.41:9001
    https://console.minio.local

9001 is the Web Console, not the S3 API.

---

## V. Preparing mc Runtime Environment

### 5.1 Pull mc Image

Run on minio-client or any management node:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Check:

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq/mc

---

### 5.2 Create mc Configuration Directory

mc alias will be saved to the configuration directory.

Create:

    mkdir -p /data/minio/mc-config

Check:

    ls -ld /data/minio/mc-config

Notes:

    Subsequent steps will mount this directory to the mc container using -v /data/minio/mc-config:/root/.mc
    This ensures alias configurations won't be lost when the container exits

---

### 5.3 Create Local Experiment Directory

Create:

    mkdir -p /tmp/minio-mc-demo/upload
    mkdir -p /tmp/minio-mc-demo/download

Check:

    tree /tmp/minio-mc-demo

If tree is not installed:

    apt update
    apt install -y tree

---

### 5.4 Check mc Version

Run:

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

## SixI don't know.Configure alias

### 6.1 Configure Backend HTTP alias

If connecting directly to MinIO backend node:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-backend http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Notes:

    minio-backend is the alias
    http://10.0.0.41:9000 is the MinIO API address
    minioadmin is the AccessKey
    MinioAdmin@123456 is the SecretKey

Production Reminder:

    Root user is only suitable for management
    Business applications should not use root user
    Subsequent steps should create business users with minimal permissions Policy

---

### 6.2 Configure HTTPS Unified Entry alias

If Nginx HTTPS entry has been completed:

For formal trusted certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

For experimental self-signed certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-https https://s3.minio.local minioadmin 'MinioAdmin@123456'

Notes:

    --insecure is only suitable for self-signed certificate experiments
    Production should use trusted certificates and should not long-term skip certificate verification

---

### 6.3 Check alias List

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

If you see:

    minio-backend
    minio-https

Configuration is successful.

---

### 6.4 Remove alias

If configuration is incorrect, you can delete:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias remove minio-backend

Then reconfigure.

---

## SevenI don't know.Check MinIO Cluster Information

### 7.1 Check Backend Cluster Information

Run:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-backend

Focus on:

    MinIO version
    Network
    Drives
    Pool
    Used
    Online
    Offline
    Healing
    Errors

---

### 7.2 Check HTTPS Entry Cluster Information

If using self-signed certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-https

If using formal certificate:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  admin info minio-https

---

### 7.3 Diagnostic Value of admin info

admin info can be used to determine:

    Whether the cluster is accessible.
    Whether nodes are online.
    Whether disks are online.
    Whether the cluster capacity is normal.
    Whether there are offline drives.
    Whether there is healing.
    Whether the reverse proxy entry is normal.
    Whether the AccessKey / SecretKey is correct.

If admin info fails, prioritize checking:

    Whether the alias endpoint is correct.
    Whether the 9001 Console was mistakenly used.
    Whether the MinIO backend is healthy.
    Whether Nginx is functioning normally.
    Whether the certificate is trusted.
    Whether the AccessKey / SecretKey is correct.

---

## VIII. Bucket Management

### 8.1 View Bucket List

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https

If connecting to the backend HTTP:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-backend

---

### 8.2 Create Bucket

Create an experimental Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure mb minio-https/mc-demo

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https

---

### 8.3 Bucket Naming Recommendations

Production naming recommendations:

    Business name - Purpose - Environment

Examples:

    app-uploads-prod
    app-backup-prod
    nginx-logs-prod
    devops-artifacts-prod
    ai-dataset-test
    resume-attachments-prod

Not recommended:

    test
    data
    bucket1
    aaa
    file

Reasons:

    Not conducive to capacity governance.
    Not conducive to permission governance.
    Not conducive to business ownership identification.
    Not conducive to alerts and auditing.

---

### 8.4 Delete Empty Bucket

Before deleting a Bucket, confirm it is empty.

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https/mc-demo

Delete:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-https/mc-demo

High-risk warning:

    Deleting a Bucket is a high-risk operation.
    In production environments, confirm business ownership, backups, and approval before deletion.
    Non-empty Buckets require objects to be deleted first.

---

## IX. Object Upload Operations

### 9.1 Prepare Test Files

Create a test file:

    echo "hello minio mc tool" > /tmp/minio-mc-demo/upload/hello.txt

Create a log file:

    mkdir -p /tmp/minio-mc-demo/upload/logs/nginx/2026/04/28

    echo "nginx access log demo" > /tmp/minio-mc-demo/upload/logs/nginx/2026/04/28/access.log

Create a large file:

    dd if=/dev/zero of=/tmp/minio-mc-demo/upload/file-100m.bin bs=1M count=100

Check:

    tree /tmp/minio-mc-demo/upload
    ls -lh /tmp/minio-mc-demo/upload

---

### 9.2 Upload Single File

Upload hello.txt:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/upload/hello.txt minio-https/mc-demo/hello.txt

Check: /think

---
docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls minio-https/mc-demo

---

### 9.3 Upload to Specified Prefix

Upload log file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/upload/logs/nginx/2026/04/28/access.log minio-https/mc-demo/logs/nginx/2026/04/28/access.log

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https/mc-demo/logs/nginx/2026/04/28/

Explanation:

    logs/nginx/2026/04/28/access.log is the Object Key.
    Among them, logs/nginx/2026/04/28/ is the Prefix.
    It looks like a directory, but in object storage it's essentially a Key prefix.

---

### 9.4 Upload Large Files

Upload 100M file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /demo/upload/file-100m.bin minio-https/mc-demo/file-100m.bin

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure stat minio-https/mc-demo/file-100m.bin

If large file upload fails, focus on checking:

    Nginx client_max_body_size
    Nginx proxy_request_buffering
    Nginx proxy_read_timeout
    MinIO backend ready status
    Disk capacity
    Network stability

---

### 9.5 Recursive Directory Upload

Upload entire upload directory:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp --recursive /demo/upload/ minio-https/mc-demo/full-upload/

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-https/mc-demo/full-upload/

---

## Ten. Object Download Operations

### 10.1 Download Single Object

Download hello.txt:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-https/mc-demo/hello.txt /demo/download/hello-download.txt

View:

    cat /tmp/minio-mc-demo/download/hello-download.txt

---

### 10.2 Download Prefix Objects

Download log objects:

    mkdir -p /tmp/minio-mc-demo/download/logs

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp minio-https/mc-demo/logs/nginx/2026/04/28/access.log /demo/download/logs/access-download.log

View:

    cat /tmp/minio-mc-demo/download/logs/access-download.log

---

### 10.3 Recursive Directory Download

Download full-upload directory:

    mkdir -p /tmp/minio-mc-demo/download/full-upload

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-mc-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp --recursive minio-https/mc-demo/full-upload/ /demo/download/full-upload/

View:

    tree /tmp/minio-mc-demo/download/full-upload

---

## Eleven. Viewing Object Information

### 11.1 Viewing Object List

Viewing Bucket root path:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https/mc-demo

Viewing specified Prefix:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-https/mc-demo/logs/nginx/2026/04/28/

---

### 11.2 Recursive Object Search

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-https/mc-demo

Search by name:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-https/mc-demo --name "*.log"

---

### 11.3 Viewing Object Metadata

Viewing hello.txt:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure stat minio-https/mc-demo/hello.txt

Focus on:

    Name
    Date
    Size
    ETag
    Content-Type

Explanation:

    stat can be used to confirm if an object exists.
    It can be used to confirm object size.
    It can be used to troubleshoot if upload is complete.

---

### 11.4 Viewing Object Content

For text objects, you can directly view:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cat minio-https/mc-demo/hello.txt

Do not use cat for large objects.

---

## Twelve. Viewing Capacity and Object Scale

### 12.1 Viewing Bucket Capacity

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-https/mc-demo

---

### 12.2 Viewing Prefix Capacity

Viewing logs prefix:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-https/mc-demo/logs/

Viewing full-upload prefix:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-https/mc-demo/full-upload/

---

### 12.3 Operational Value

Capacity viewing is used for:

    Judging Bucket growth trend.
    Judging if business upload is abnormal.
    Judging if a Prefix is abnormally growing.
    Troubleshooting object storage capacity alerts.
    Capacity assessment before migration.
    Capacity assessment before backup.

Production recommendations:

    Each business Bucket should have ownership.
    Each Bucket should have capacity statistics.
    Large Buckets should have lifecycle policies.
    Important Buckets should have backup policies.
    Abnormal growth should trigger alerts.

---

## Thirteen. Object Deletion Operations

### 13.1 Deleting a Single Object

Delete hello.txt:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure rm minio-https/mc-demo/hello.txt

Confirmation:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure ls minio-https/mc-demo

---

### 13.2 Deleting Objects Under a Prefix

High-risk warning:

  --recursive will recursively delete objects.
  --force will skip confirmation.
  Production environments must confirm the Bucket and Prefix.
  Must confirm there is a backup before deletion.

Delete logs prefix:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure rm --recursive --force minio-https/mc-demo/logs/

Confirmation:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure find minio-https/mc-demo

---

### 13.3 Deleting All Objects in a Bucket

High-risk warning:

  This operation will delete all objects in the Bucket.
  Prohibited to execute without approval in production environments.

Delete all objects in the Bucket:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure rm --recursive --force minio-https/mc-demo

Note:

  This will clear objects in the mc-demo Bucket.
  The Bucket itself will not be deleted.

---

### 13.4 Deleting a Bucket

Delete the Bucket after confirming it is empty:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure rb minio-https/mc-demo

If the Bucket is not empty, first clear the objects, then delete the Bucket.

---

## FourteenI don't know.mirror Basic Synchronization

### 14.1 What is mirror

mc mirror is used for mirroring synchronization between directories and Buckets.

Common use cases:

  Synchronize local directory to Bucket
  Synchronize Bucket to local directory
  Synchronize one Bucket to another Bucket
  Synchronize one MinIO cluster to another MinIO cluster
  Data migration
  Backup archiving

This document only covers basic experiments.

Cross-cluster backup and migration will be detailed in:

  10-MinIO Backup Migration: mc mirror, Cross-Cluster Synchronization and Data Migration.md

---

### 14.2 Create mirror Experiment Bucket

Create Bucket:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure mb minio-https/mirror-demo

---

### 14.3 Prepare Local Directory

Create:

  mkdir -p /tmp/minio-mc-demo/mirror-source/app
  mkdir -p /tmp/minio-mc-demo/mirror-source/logs

Write files:

  echo "app config v1" > /tmp/minio-mc-demo/mirror-source/app/config.txt
  echo "log line 1" > /tmp/minio-mc-demo/mirror-source/logs/app.log

View:

  tree /tmp/minio-mc-demo/mirror-source

---

### 14.4 Synchronize Local Directory to Bucket

Execute:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    -v /tmp/minio-mc-demo:/demo \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure mirror /demo/mirror-source/ minio-https/mirror-demo/

View:

  docker run --rm \
    -v /data/minio/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    --insecure find minio-https/mirror-demo

---

### 14.5 Synchronize Bucket to Local Directory

Create local recovery directory:

  mkdir -p /tmp/minio-mc-demo/mirror-restore

Execute: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-mc-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure mirror minio-https/mirror-demo/ /demo/mirror-restore/

Check:

  tree /tmp/minio-mc-demo/mirror-restore

Verify:

  cat /tmp/minio-mc-demo/mirror-restore/app/config.txt
  cat /tmp/minio-mc-demo/mirror-restore/logs/app.log

---

### 14.6 Differences between mirror and cp

| Command | Suitable Scenario |
|---|---|
| mc cp | Upload or download single files, small number of files |
| mc cp --recursive | Recursive copy of directories |
| mc mirror | Directory or Bucket mirror synchronization |
| mc mirror --watch | Continuous synchronization, requires caution |
| mc mirror --remove | Remove objects from target that do not exist in source, high-risk |

Production warning:

  mc mirror --remove is a high-risk operation.
  It may delete data on the target end.
  Production must first perform a dry-run or validate in a test environment.
  Migration and synchronization tasks should have logs and verification.

---

## FifteenI don't know.Simplified Command Methods

### 15.1 Creating Temporary Alias Function

If running docker run repeatedly is too long, you can temporarily define a function in the current shell:

  export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
  export MC_CONFIG=/data/minio/mc-config
  export MC_WORKDIR=/tmp/minio-mc-demo

  mcx() {
    docker run --rm \
      -v ${MC_CONFIG}:/root/.mc \
      -v ${MC_WORKDIR}:/demo \
      ${MC_IMAGE} "$@"
  }

If using a self-signed certificate, you can call it like this:

  mcx --insecure ls minio-https

Create Bucket:

  mcx --insecure mb minio-https/simple-demo

Upload:

  mcx --insecure cp /demo/upload/hello.txt minio-https/simple-demo/hello.txt

Check:

  mcx --insecure ls minio-https/simple-demo

Note:

  This function is only valid in the current shell session.
  It needs to be redefined after reopening the terminal.
  Suitable for experimental phases to reduce repetitive input.

---

### 15.2 Production Recommendations

In production, it is recommended:

  Install a fixed version of mc on the management machine.
  Or encapsulate it into an internal operations script.
  Do not hardcode business SecretKey in commands.
  Use a dedicated operations account.
  High-risk delete commands must be confirmed twice.
  Operation logs should be retained.

---

## SixteenI don't know.Common Issues Troubleshooting

### 16.1 mc alias set Failed

Possible reasons:

  endpoint written incorrectly.
  Confusing 9001 Console with API.
  AccessKey error.
  SecretKey error.
  MinIO backend unreachable.
  Nginx 443 unreachable.
  Self-signed certificate not trusted.

Troubleshoot:

  curl -I http://10.0.0.41:9000/minio/health/live
  curl -k -I https://s3.minio.local/minio/health/live
  docker logs minio
  tail -f /var/log/nginx/error.log

---

### 16.2 mcHintcertificate signed by unknown authority

Cause:

  Using a self-signed certificate.
  Client does not trust the certificate.

Experimental handling:

  Use --insecure.

Production handling:

  Use an official certificate.
  Or import enterprise CA.
  Not recommended to use --insecure long-term in production.

---

### 16.3 mc Upload Failed

Troubleshooting order:

  1. Is the alias correct?
  2. Does the Bucket exist?
  3. Does the user have write permissions?
  4. Is Nginx limiting request body size?
  5. Is the MinIO cluster ready?
  6. Is the disk full?
  7. Are nodes offline?
  8. Are AccessKey/SecretKey correct?

Commands:

  mc admin info
  mc ls
  mc stat
  df -hT
  docker logs minio
  tail -f /var/log/nginx/error.log

---

### 16.4 mc Download Failed

Common reasons:

  Object does not exist.
  Object Key written incorrectly.
  User has no read permissions.
  Bucket written incorrectly.
  Prefix written incorrectly.
  Network interruption.
  Nginx proxy anomaly.
  MinIO backend node anomaly.

Troubleshoot:

  mc ls
  mc find
  mc stat
  mc admin info

---

### 16.5 Bucket Deletion Failed

Common reasons:

  Bucket is not empty.
  User has no permissions.
  Bucket name is incorrect.
  Historical versions exist after version control is enabled.
  Object lock or retention policy restrictions.

Basic handling:

  First ls / find to confirm objects.
  Confirm it's an experimental Bucket.
  Clear objects and then rb.

---

## SeventeenI don't know.High-Risk Operations List

The following operations must be cautious in production environments:

mc rm --recursive --force
mc rb
mc mirror --remove
Delete Production Bucket
Delete Production Prefix
Delete Business Archive Objects
Use root User for Batch Operations
Use --insecure to Connect to Production Environment
Write AccessKey / SecretKey in Plain Text in Scripts
Execute Deletion Operations on Error Alias

Before Executing High-Risk Operations, Must Confirm:

    Which Environment is the Current alias.
    Whether the Current Bucket is Production.
    Whether the Current Prefix is Correct.
    Whether There is a Backup.
    Whether There is Approval.
    Whether There is a Rollback Plan.
    Whether There is Business Confirmation.
    Whether There is Dual Person Review.

---

## EighteenI don't know.Production Environment Recommendations

### 18.1 alias Naming Convention

Recommendations:

| alias | Description |
|---|---|
| minio-dev | Development Environment |
| minio-test | Testing Environment |
| minio-prod | Production Environment |
| minio-backup | Backup Cluster |
| minio-dr | Disaster Recovery Cluster |

Not Recommended:

    local
    test
    aaa
    minio1

Reasons:

    Prone to Accidental Operations.
    Difficult to Determine Environment During High-Risk Deletion.

---

### 18.2 Confirm Environment Before Operation

Before Executing High-Risk Commands, First Check:

    mc alias list
    mc admin info minio-prod
    mc ls minio-prod

Confirm:

    Whether the Current is Production.
    Whether the Current is Target Bucket.
    Whether Current User Permissions are Reasonable.
    Whether Current Operation is Rollbackable.

---

### 18.3 Business Account Principles

In Production:

    root User is Only for Platform Management.
    Business Applications Use Independent AccessKey.
    Each Business Has One User or Group of Users.
    Permissions are Restricted to Specific Bucket or Prefix.
    Keys are Rotated Regularly.
    Keys are Not Submitted to Git.
    Keys are Not Written to Public Documents.

---

### 18.4 Daily Inspection Recommendations

Daily Checks Can Include:

    mc admin info
    mc ls
    mc du Key Bucket
    mc find Key Prefix
    Nginx 4xx / 5xx
    MinIO Node Status
    Disk Capacity
    Bucket Abnormal Growth

---

## NineteenI don't know.Experiment Cleanup

### 19.1 Delete mc-demo Objects

High-Risk Warning:

    Only Delete Experimental Bucket.
    Do Not Accidentally Delete Production Bucket.

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-https/mc-demo

Delete Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-https/mc-demo

---

### 19.2 Delete mirror-demo Objects

Execution:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force minio-https/mirror-demo

Delete Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rb minio-https/mirror-demo

---

### 19.3 Clean Up Local Test Directory

    rm -rf /tmp/minio-mc-demo

---

## TwentyI don't know.Interview Answer Strategy

If Interviewed with:

    What is the mc Tool of MinIO Usually Used For?

Can Answer:

    mc is the official command-line client of MinIO, which can be understood as a commonly used management tool for object storage operations. It can configure alias to connect MinIO or other S3-compatible object storage, and then perform Bucket management, object upload/download, capacity viewing, metadata viewing, object deletion, directory synchronization, and mirror migration operations.
    In daily operations, I would use mc alias set to configure connections for different environments, such as minio-prod, minio-test; use mc admin info to check cluster nodes, disks, and capacity status; use mc mb to create Buckets; use mc cp to upload or download objects; use mc stat to view object metadata; use mc du to view Bucket or Prefix capacity; use mc find to search for objects; use mc mirror for directory synchronization or cross-cluster migration.
    Note that mc connects to the S3 API address of MinIO, which is the 9000 port or HTTPS API entry, not the 9001 Console port. In production environments, root users should not be used for business operations; instead, independent users and minimal permission Policies should be created.
    At the same time, mc rm --recursive --force, mc rb, and mc mirror --remove are all high-risk operations. Before executing in production, must confirm alias, Bucket, Prefix, backup, approval, and rollback plan to avoid accidental deletion of object data.

---

## Twenty-OneI don't know.Summary of This Article

This article completed the practical operations of the MinIO mc client tool:

1. mc is the official command-line client for MinIO.
2. mc connects to the S3 API address, not the Console address.
3. 9000 is the API port, 9001 is the Console port.
4. mc alias is used to configure MinIO connections.
5. mc admin info can view cluster nodes, disks, and capacity status.
6. mc mb is used to create a Bucket.
7. mc rb is used to delete a Bucket.
8. mc ls is used to list Buckets or objects.
9. mc cp can upload and download objects.
10. mc cp --recursive can recursively copy directories.
11. mc stat can view object metadata.
12. mc cat can view text object content.
13. mc du can view Bucket or Prefix capacity.
14. mc find can recursively search for objects.
15. mc rm can delete objects.
16. mc mirror can perform basic directory synchronization and migration operations.
17. Self-signed certificate experiments can temporarily use --insecure, but it's not recommended for long-term use in production.
18. Root users are only suitable for management; business operations should use independent AccessKey and Policy.
19. Deleting Buckets, recursively deleting objects, and mirror --remove all belong to high-risk operations.
20. The next article will enter MinIO permission management: users, Policy, AccessKey, and SecretKey.

---

## Twenty-TwoI don't know.Reference Documents

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO mc alias Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-alias.html

MinIO mc cp Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-cp.html

MinIO mc mirror Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-mirror.html

MinIO mc admin Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc-admin.html

MinIO Object Storage Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Identity and Access Management:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html