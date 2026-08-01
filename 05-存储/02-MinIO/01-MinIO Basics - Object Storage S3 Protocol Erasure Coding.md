# MinIO Basics: Object Storage, S3 Protocol, and Erasure Coding

Recommended path: 05-Storage/02-MinIO/01-MinIO Basics: Object Storage, S3 Protocol, and Erasure Coding.md

Tags: #MinIO #ObjectStorage #S3 #Bucket #Object #ErasureCoding #Docker #mc #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the first article of the MinIO module, focusing on learning the basic concepts and minimal operational workflow of MinIO.

This document is not purely theoretical but instead uses a single-machine Docker MinIO experiment to fully demonstrate the following concepts:

    What is object storage
    What is a Bucket
    What is an Object
    What is the S3 protocol
    What are AccessKey / SecretKey
    How does the mc client connect to MinIO
    How to create a Bucket
    How to upload objects
    How to download objects
    How to view object information
    How to understand "directories" in object storage
    What problems does Erasure Coding solve
    What are the differences between MinIO and traditional file systems, block storage, and Ceph RGW

Document positioning:

    First, run through the basic object storage workflow of MinIO.
    Then establish concepts of S3 / Bucket / Object / Erasure Coding.
    Subsequently enter single-machine, multi-disk, multi-node, Nginx HTTPS, permissions, monitoring, and backup migration.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand that MinIO is an object storage system compatible with the S3 API.
2. Understand the differences between object storage and block storage, file storage.
3. Understand the relationship between Bucket, Object, and Prefix.
4. Understand the role of AccessKey and SecretKey.
5. Use Docker to start a single-machine MinIO.
6. Use the mc client to connect to MinIO.
7. Create a Bucket.
8. Upload, download, view, and delete objects.
9. Understand that "directories" in object storage are essentially object Key prefixes.
10. Understand the basic function of Erasure Coding.
11. Understand the differences between a single-machine experiment and a production distributed cluster.
12. Establish a foundation for subsequent MinIO deployment, permissions, reverse proxy, monitoring, and data protection.

---

## III. Experimental Environment

### 3.1 Experimental Nodes

This article uses a single-machine Docker experiment.

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | Single-machine MinIO server |
| 10.0.0.45 | minio-client | mc client, optional |

If only one machine is available, you can also run the MinIO server and mc client on the same machine.

---

### 3.2 Operating System

Default system:

    Ubuntu Server 22.04.5 LTS

Optional system:

    Rocky Linux 9

The commands in this article are primarily based on Ubuntu.

---

### 3.3 Container Runtime

This article uses:

    Docker

Not Kubernetes.

Reasons:

    The current stage focuses on understanding object storage and S3 operations.
    Docker is more suitable for quick deployment, verification, and cleanup.
    Kubernetes and CSI are not the focus of the MinIO basics stage.
    MinIO is typically accessed via the S3 API by applications, not necessarily through PVC.

---

### 3.4 Image Version

This article uses the fixed version images already synchronized to the Alibaba Cloud image repository:

MinIO server:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source images:

    minio/minio:RELEASE.2025-04-22T22-12-26Z
    minio/mc:RELEASE.2025-04-16T18-13-26Z

Reason for fixed versions:

    To avoid inconsistencies caused by the latest version changing Web Console, command parameters, log formats, and experimental results.
    To ensure reproducibility of subsequent single-machine Docker, distributed Docker, reverse proxy, permissions, Bucket, Policy, mc mirror, etc. experiments.
    The mc client version is close in time to the MinIO server version, reducing client-server capability differences.

---

## IV. Basic Concepts of Object Storage

### 4.1 What is Object Storage

Object storage is a storage method that saves data in units of "objects".

An object typically consists of three parts:

    Object Data: The data itself
    Metadata: Object metadata
    Object Key: Object name or object path

Object storage is not a traditional directory tree file system.

It is more like:

    Bucket
      |
      |-- object-key-1
      |-- object-key-2
      |-- logs/2026/04/01/app.log
      |-- images/avatar/user-001.png

Where:

    logs/2026/04/01/app.log

Looks like a directory, but in object storage it is essentially an object Key.

s3://app-uploads/avatar/user-001.png

Where:

    app-uploads is the Bucket
    avatar/user-001.png is the Object Key

---

### 4.4 What is a Prefix

Prefix is the prefix of an object Key.

For example object:

    logs/nginx/2026/04/28/access.log

It can be considered to have the following prefixes:

    logs/
    logs/nginx/
    logs/nginx/2026/
    logs/nginx/2026/04/
    logs/nginx/2026/04/28/

The "directory" structure in object storage is typically simulated through Prefix.

Important understanding:

    Object storage is not a traditional file system.
    Directories may not exist physically.
    The directory hierarchy you see usually comes from the object Key's prefix.

---

### 4.5 What is S3 Protocol

S3 is a very common set of API specifications in the object storage domain.

MinIO is compatible with S3 API, so many tools and applications that support S3 can connect to MinIO.

Common S3 operations:

| Operation | Purpose |
|---|---|
| CreateBucket | Create Bucket |
| ListBuckets | View Buckets |
| PutObject | Upload object |
| GetObject | Download object |
| DeleteObject | Delete object |
| ListObjects | List objects |
| HeadObject | View object metadata |

Common access forms:

    http://minio.example.com/bucket/object
    https://s3.example.com/bucket/object

You can also access via SDK or tools, for example:

    mc
    aws cli
    s3cmd
    boto3
    Java S3 SDK
    Go S3 SDK

---

### 4.6 AccessKey and SecretKey

Object storage typically authenticates through AccessKey and SecretKey.

You can simply understand them as:

| Name | Similar Concept |
|---|---|
| AccessKey | Username |
| SecretKey | Password |

MinIO's default root user is set by these environment variables:

    MINIO_ROOT_USER
    MINIO_ROOT_PASSWORD

For experimentation:

    MINIO_ROOT_USER=minioadmin
    MINIO_ROOT_PASSWORD=MinioAdmin@123456

Production reminder:

    The root user is only for management.
    Business applications should not directly use the root user.
    Each business should create independent users and policies.
    AccessKey / SecretKey should not be submitted to Git.
    AccessKey / SecretKey should not be written in public documentation.
    They must be rotated after leakage.

---

## Five. Differences Between Object Storage and Block Storage, File Storage

### 5.1 Block Storage

Block storage is similar to cloud disks.

Typical representatives:

    Ceph RBD
    Cloud disk
    SAN LUN

Usage:

    First obtain a block device.
    Then format it into a file system.
    Then mount it to a host.
    Applications use it like local disks.

Suitable for:

    Databases
    Virtual machine disks
    Kubernetes RWO PVC
    Single instance dedicated data disks

---

### 5.2 File Storage

File storage is similar to shared directories.

Typical representatives:

    CephFS
    NFS
    NAS

Usage:

    Multiple machines mount the same shared directory.
    Read and write files through paths.

Suitable for:

    Multi-client shared directories
    Shared upload directories
    Kubernetes RWX PVC
    Shared configuration files
    Shared datasets

---

### 5.3 Object Storage

Object storage is accessed through API.

Typical representatives:

    MinIO
    Ceph RGW
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS

Usage:

    Applications upload and download objects via HTTP / HTTPS API.
    No need to mount as local directories.
    Does not provide traditional POSIX file system semantics.

Suitable for:

    Images
    Attachments
    Videos
    Backups
    Log archives
    Artifact repositories
    Static resources
    AI datasets
    Large amounts of unstructured data

---

### 5.4 Mnemonic for Choosing Between the Three Storage Types

    Use like cloud disks: Block storage
    Use like shared directories: File storage
    Use like OSS / S3: Object storage

Corresponding to this module:

    Ceph RBD: Block storage
    CephFS: File storage
    MinIO: Object storage
    Ceph RGW: Object storage

---

## Six. Experiment One: Pull MinIO and mc Images

### 6.1 Pull MinIO Server Image

Execute on minio-node01:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Check:

    docker images | grep minio

---

### 6.2 Pull mc Client Image

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Check:

    docker images | grep mc

---

### 6.3 Check Images

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq

Expected to see:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc

---

## Seven. Experiment Two: Start Single-node MinIO

### 7.1 Create Data Directory

Execute on minio-node01:

    mkdir -p /data/minio/data

Check:

    ls -ld /data/minio/data

---

### 7.2 Start MinIO Container

Execute: /think

docker run -d \
  --name minio-basic \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
  -v /data/minio/data:/data \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
  server /data --console-address ":9001"

Parameter Explanation:

| Parameter | Description |
|---|---|
| --name minio-basic | Container name |
| --restart unless-stopped | Automatically restart after container failure |
| -p 9000:9000 | Map S3 API port |
| -p 9001:9001 | Map Web Console port |
| MINIO_ROOT_USER | MinIO root user |
| MINIO_ROOT_PASSWORD | MinIO root password |
| -v /data/minio/data:/data | Data directory mounting |
| server /data | Use /data as object data directory |
| --console-address ":9001" | Console listens on 9001 |

---

### 7.3 Check Container Status

    docker ps | grep minio-basic

View logs:

    docker logs -f minio-basic

If logs show similar information, it indicates successful startup:

    API:
    Console:
    Documentation:

---

### 7.4 Access Web Console

Browser access:

    http://10.0.0.41:9001

Login credentials:

    Username: minioadmin
    Password: MinioAdmin@123456

Notes:

    9001 is the Web Console.
    9000 is the S3 API.
    It's not recommended to expose HTTP 9000 and 9001 to public networks in production.
    External access should use Nginx / LB for HTTPS unified entry.

---

## VIII. Experiment 3: Using mc to Connect to MinIO

### 8.1 Execute Commands Using mc Container

To avoid installing mc locally, you can directly use the mc image.

Check mc version:

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

### 8.2 Create mc Configuration Directory

To make mc alias configurations persistent, create a local directory:

    mkdir -p /data/minio/mc-config

---

### 8.3 Set Alias

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set local http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Notes:

    local is the alias for this MinIO service.
    http://10.0.0.41:9000 is the S3 API address.
    minioadmin is the AccessKey.
    MinioAdmin@123456 is the SecretKey.

View alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 8.4 Check MinIO Service Information

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info local

If you can see MinIO server, capacity, version, etc., it indicates mc has successfully connected.

---

## IX. Experiment 4: Create Bucket

### 9.1 Create Bucket

Create a test bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb local/app-uploads

Expected output:

    Bucket created successfully

---

### 9.2 View Bucket

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls local

Expected to see:

    app-uploads

---

### 9.3 Bucket Naming Recommendations

Production bucket naming recommendations:

    Business name - Purpose - Environment

Examples:

    resume-uploads-prod
    app-backup-prod
    nginx-logs-prod
    devops-artifacts-prod
    ai-dataset-test

Not recommended:

    test
    data
    file
    bucket1
    aaa

Reasons:

    Names are unclear, making it difficult to manage permissions, capacity, and ownership later.

### 10.1 Create Local Test File

Execute on minio-node01:

    mkdir -p /tmp/minio-basic-demo

    echo "hello minio object storage" > /tmp/minio-basic-demo/hello.txt

Check:

    cat /tmp/minio-basic-demo/hello.txt

---

### 10.2 Upload Object to Bucket

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-basic-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/hello.txt local/app-uploads/hello.txt

---

### 10.3 View Object

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls local/app-uploads

Expected to see:

    hello.txt

---

### 10.4 View Object Details

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      stat local/app-uploads/hello.txt

Focus on:

- Name
- Date
- Size
- ETag
- Content-Type

---

## ElevenI don't know.Experiment Six: Download Object

### 11.1 Delete Local Original File

    rm -f /tmp/minio-basic-demo/hello.txt

Confirm:

    ls -l /tmp/minio-basic-demo/

---

### 11.2 Download Object from MinIO

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-basic-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp local/app-uploads/hello.txt /demo/hello-download.txt

Check:

    cat /tmp/minio-basic-demo/hello-download.txt

Expected:

    hello minio object storage

Explanation:

    Upload and download are both completed through S3 API.
    Applications will also perform similar operations through S3 SDK in the future.

---

## TwelveI don't know.Experiment Seven: Understanding "Directories" in Object Storage

### 12.1 Upload Object with Path

Create test file:

    mkdir -p /tmp/minio-basic-demo/logs

    echo "nginx access log demo" > /tmp/minio-basic-demo/logs/access.log

Upload to object with prefix key:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-basic-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/logs/access.log local/app-uploads/logs/nginx/2026/04/28/access.log

---

### 12.2 View Bucket

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls local/app-uploads

May see:

    hello.txt
    logs/

Continue to view:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls local/app-uploads/logs/nginx/2026/04/28/

Expected to see:

    access.log

---

### 12.3 Key Understanding

Object Key is:

    logs/nginx/2026/04/28/access.log

This is not a real multi-level directory in traditional file system.

It is essentially an object name.

MinIO Console and mc will display it as a directory-like structure based on slashes.

Essence of directory in object storage:

    Prefix (Prefix)

This is very important.

In production, plan object Key design in advance:

    Business type
    Date
    User ID
    File type
    Environment
    Lifecycle

Example:

    uploads/user/10001/avatar.png
    logs/nginx/2026/04/28/access.log
    backup/mysql/2026/04/28/mysql-full.sql.gz
    artifacts/devatlas/prod/v1.0.0/app.tar.gz

---

## ThirteenI don't know.Experiment Eight: View Bucket Capacity

### 13.1 View Bucket Usage

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  du local/app-uploads

---

### 13.2 Viewing Recursive Objects

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  find local/app-uploads

---

### 13.3 Operational Significance

In daily operations, attention should be paid to:

- Bucket capacity
- Object count
- Large objects
- Small object count
- Business growth trends
- Presence of abnormal uploads
- Need for lifecycle cleanup
- Need for quota limits

Object storage is not complete after creating a Bucket.

Continuous capacity governance is required in production environments.

---

## FourteenI don't know.Experiment Nine: Deleting Objects & Bucket

### 14.1 Deleting a Single Object

Delete hello.txt:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  rm local/app-uploads/hello.txt

Check:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls local/app-uploads

---

### 14.2 Deleting Objects with Prefix

Delete objects under logs prefix:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  rm --recursive --force local/app-uploads/logs/

High-risk warning:

- --recursive will recursively delete objects.
- --force will skip confirmation.
- Must confirm Bucket and Prefix before execution in production.
- Do not use rm --recursive --force arbitrarily on production Buckets.

---

### 14.3 Deleting Bucket

Confirm Bucket is empty:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls local/app-uploads

Delete Bucket:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  rb local/app-uploads

Production warning:

- Deleting Bucket is a high-risk operation.
- Must confirm objects have been backed up before deletion.
- Must confirm business still uses the Bucket before deletion.
- Deletion should go through approval and review processes.

---

## FifteenI don't know.Erasure Coding Basic Understanding

### 15.1 What is Erasure Coding

Erasure Coding can be understood as a forward error correction code.

It splits data into multiple data blocks and parity blocks, distributing them across multiple disks or nodes.

When part of the disks or nodes fail, as long as the remaining data blocks and parity blocks are sufficient, the original data can be recovered.

Simple understanding:

- Data is not simply replicated three times.
- Data is split into blocks.
- Parity data is generated simultaneously.
- Data blocks + parity blocks achieve fault tolerance.

---

### 15.2 Difference Between Replication and Erasure Coding

Replication mode:

- Data is fully replicated multiple times.

Example with 3 replicas:

- Original data 1GB
- Actual storage about 3GB

Advantages:

- Simple
- Intuitive for reading
- Easy to recover

Disadvantages:

- Low space utilization

---

Erasure Coding:

- Data is split into data blocks and parity blocks.

Example:

- 4 data blocks + 2 parity blocks

Advantages:

- Higher space utilization
- Still has fault tolerance capability

Disadvantages:

- Higher complexity for encoding and decoding
- More sensitive to disk, network, and node planning
- Data recovery requires rebuilding data blocks

---

### 15.3 Role of Erasure Coding in MinIO

MinIO uses Erasure Coding to implement data protection.

It is used for:

- Single disk failure
- Multiple disk failures
- Single node failure
- Partial node unavailability

But note:

- Erasure Coding is not backup.
- Erasure Coding cannot prevent accidental deletion.
- Erasure Coding cannot replace cross-cluster backup.
- Erasure Coding cannot replace mc mirror orAlien. disaster recovery.

---

### 15.4 Why Not Deploy Single-node Single-disk

Single-node single-disk mode is suitable for:

- Learning
- Development
- Temporary testing
- Function verification

Not suitable for production.

Reasons:

- Single node failure causes service interruption.
- Single disk failure may lead to data loss.
- Does not fully leverage Erasure Coding's high availability.
- Cannot verify node and disk failure recovery.

Production recommends:

- Multi-node multi-disk distributed deployment.

Example:

- 4 nodes
- Multiple data disks per node
- Frontend using Nginx / LB
- HTTPS externally
- Internal HTTP backend
- Prometheus monitoring
- mc mirror backup

---

## SixteenI don't know.Optional Experiment: Single-node Multi-directory Understanding Erasure Coding

### 16.1 Experiment Description /think

# 16.2 Stop Single-Node Container

If minio-basic is running, stop it first:

    docker stop minio-basic
    docker rm minio-basic

---

# 16.3 Create Multiple Data Directories

    mkdir -p /data/minio-ec/disk1
    mkdir -p /data/minio-ec/disk2
    mkdir -p /data/minio-ec/disk3
    mkdir -p /data/minio-ec/disk4

Check:

    tree /data/minio-ec

---

# 16.4 Start Single-Node Multi-Directory MinIO

Run:

    docker run -d \
      --name minio-ec-basic \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -v /data/minio-ec/disk1:/disk1 \
      -v /data/minio-ec/disk2:/disk2 \
      -v /data/minio-ec/disk3:/disk3 \
      -v /data/minio-ec/disk4:/disk4 \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server /disk1 /disk2 /disk3 /disk4 --console-address ":9001"

Check:

    docker ps | grep minio-ec-basic
    docker logs -f minio-ec-basic

---

# 16.5 Reconfigure mc alias

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set local http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Check service status:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info local

---

# 16.6 Create Bucket and Upload Objects

Create Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb local/ec-demo

Create file:

    mkdir -p /tmp/minio-ec-demo

    dd if=/dev/zero of=/tmp/minio-ec-demo/test-10m.bin bs=1M count=10

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-ec-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/test-10m.bin local/ec-demo/test-10m.bin

Check:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls local/ec-demo

---

# 16.7 Observe Data Directories

Check each directory:

    find /data/minio-ec -maxdepth 3 -type f | head -50

You can observe that MinIO maintains its object data and metadata across different directories.

Note:

    Do not manually modify files in these directories.
    Do not directly delete objects from data directories.
    Object operations should be performed via S3 API or mc.

---

# 16.8 Optional Simulation: Recover After Container Stop

Stop:

    docker stop minio-ec-basic

Start:

    docker start minio-ec-basic

Verify objects still exist:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls local/ec-demo

Download verification:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-ec-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp local/ec-demo/test-10m.bin /demo/test-10m-download.bin

Check:

    ls -lh /tmp/minio-ec-demo/test-10m-download.bin

## 17. Basic Troubleshooting Methods for MinIO

### 17.1 Is the Container Running

    docker ps | grep minio

If it's not running:

    docker ps -a | grep minio

---

### 17.2 Check Logs

    docker logs minio-basic

Or:

    docker logs minio-ec-basic

Common Issues:

- Password does not meet requirements
- Data directory permission issues
- Port is occupied
- Startup parameter errors
- Container repeatedly restarts

---

### 17.3 Check Ports

    ss -lntp | grep -E '9000|9001'

Expected:

    9000 Listening
    9001 Listening

---

### 17.4 Check API Accessibility

    curl -I http://10.0.0.41:9000

HTTP responses like 403, 400, 405 indicate the service port is responsive.

If connection timeout:

    Check if the container is running.
    Check port mapping.
    Check firewall settings.
    Check if the IP is correct.

---

### 17.5 mc Connection Failure

Reconfigure alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set local http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Test:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info local

Common Causes:

- Endpoint is incorrect
- AccessKey is incorrect
- SecretKey is incorrect
- MinIO container is not running
- 9000 port is unreachable
- Firewall interception
- Using Console port 9001 as API port

Note:

    mc alias must connect to 9000 API port.
    Not to connect to 9001 Console port.

---

## 18. Production Environment Considerations

### 18.1 Single Machine Experiment ≠ Production

The single-machine Docker experiment in this document is only for learning basic concepts.

Not recommended for production:

    Single machine with single disk
    Single node with single disk
    Exposing 9000 HTTP directly to public internet
    Using root user for business
    Hardcoding AccessKey in code
    No monitoring
    No backup
    No reverse proxy
    No HTTPS

---

### 18.2 Recommended Production Directions

Production is more recommended:

    Multi-node multi-disk
    Erasure Coding
    Nginx / LB unified entry point
    External HTTPS
    Internal trusted network HTTP
    Separating API and Console
    Ordinary business user + minimal permission Policy
    Prometheus monitoring
    Bucket capacity alerts
    Disk capacity alerts
    mc mirror backup
    Regular recovery drills

---

### 18.3 Internal HTTP vs External HTTPS

This module will uniformly adopt:

    Internal MinIO node-to-node HTTP communication.
    External client HTTPS access.
    Nginx as unified entry point proxy for 9000 API.
    Console 9001 only open to operations network segment or via separate domain proxy.

Reasons:

    Internal trusted network uses HTTP to reduce complexity and overhead.
    External access must use HTTPS to protect credentials and object data.
    Unified entry point facilitates certificate management, auditing, rate limiting, and security controls.

---

## 19. Common Misconceptions

### 19.1 Treating Object Storage as a File System

Incorrect Understanding:

    MinIO can be mounted as a shared directory like NFS.

Correct Understanding:

    MinIO is object storage.
    Applications read/write objects via S3 API.
    It is not a traditional POSIX file system.
    Not suitable for direct replacement of database local data directories.

---

### 19.2 Using 9001 as S3 API Port

Error:

    mc alias set local http://10.0.0.41:9001 ...

Correct:

    mc alias set local http://10.0.0.41:9000 ...

Note:

    9000 is S3 API.
    9001 is Web Console.

---

### 19.3 Using latest Image

Error:

    minio/minio:latest

Correct:

    Use fixed version image.

This module defaults to:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

### 19.4 Believing Erasure Coding Equals Backup

Incorrect Understanding:

    MinIO has Erasure Coding, so no need for backup.

Correct Understanding:

    Erasure Coding is for tolerating disk or node failures.
    It cannot prevent accidental deletion.
    It cannot prevent administrator errors.
    It cannot prevent cluster-wide failures.
    Important data still requires mc mirror or cross-cluster backup.

---

### 19.5 Using root User for Business

Error:

    Directly providing MINIO_ROOT_USER and MINIO_ROOT_PASSWORD to business applications.

Correct:

    Root user is only for management.
    Business applications should use independent users and minimal permission Policy.
    Each business has independent AccessKey.
    Keys can be individually rotated and discarded.

---

## 20. Cleaning Up the Experimental Environment

### 20.1 Stop and Remove Single-Machine Containers

If using minio-basic:

    docker stop minio-basic
    docker rm minio-basic

If using minio-ec-basic:

    docker stop minio-ec-basic
    docker rm minio-ec-basic

---

### 20.2 Whether to Delete Data Directory

Warning:

    Deleting the data directory will delete all object data.
    Confirm this is a test environment before deletion.

Delete single-machine data: /think

rm -rf /data/minio/data

Delete EC experimental data:

    rm -rf /data/minio-ec

Delete mc configuration:

    rm -rf /data/minio/mc-config

Do not execute the above deletion commands directly in production environments.

---

## Twenty-one, Interview Answer Approach

If asked in an interview:

    What is MinIO? How does it differ from traditional file storage?

You can respond:

    MinIO is an object storage system compatible with the S3 API, primarily used for storing images, attachments, backups, log archives, and artifact packages - non-structured data. It is not a traditional file system, nor is it block storage.
    Block storage is similar to cloud disks, typically requiring formatting and mounting, suitable for databases or virtual machine disks. File storage is similar to NFS or CephFS, suitable for multiple clients mounting shared directories. Object storage accesses data via HTTP/HTTPS API, organizing data in Buckets and Objects. Applications typically upload/download objects via S3 SDK, mc, or aws cli.
    MinIO's core concepts include Bucket, Object, AccessKey, SecretKey, Policy, and Erasure Coding. A Bucket is an object container, while an Object is the actual data.So-called Directory essentially represents an Object Key prefix.
    In production environments, MinIO typically deploys with multiple nodes and disks, using Erasure Coding for disk and node-level fault tolerance. External access should use HTTPS as a unified entry point, while internal node communication can use HTTP within a trusted network. Business applications should not use root users, but instead create independent AccessKey and minimal-permission Policies. Monitoring, capacity alerts, and cross-cluster backups are also required, as Erasure Coding cannot replace backups.

---

## Twenty-two, Summary of This Section

This article completes the basic concepts and minimal practical operations of MinIO:

1. MinIO is an object storage system compatible with the S3 API.
2. Object storage organizes data through Buckets and Objects.
3. The "directory" in object storage essentially represents an Object Key prefix.
4. The S3 API is the core access method for object storage.
5. AccessKey and SecretKey are used for object storage authentication.
6. 9000 is the MinIO S3 API port.
7. 9001 is the MinIO Web Console port.
8. Single-node Docker is suitable for learning and verification, but not for production.
9. mc is a commonly used MinIO client tool.
10. mc alias should connect to the 9000 API port.
11. mc can create Buckets, upload objects, download objects, view capacity, and delete objects.
12. Erasure Coding is used for data protection and fault tolerance.
13. Erasure Coding is not equivalent to backups.
14. Production MinIO should use multi-node multi-disk deployments, unified HTTPS entry points, minimal-permission users, monitoring alerts, and backups/migration.
15. Subsequent sections will continue to learn MinIO's deployment modes, distributed clusters, Nginx HTTPS, permissions, monitoring, and backup migration.

---

## Twenty-three, Reference Documents

MinIO official documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker deployment documentation:

    https://min.io/docs/minio/container/index.html

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO Erasure Coding documentation:

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

MinIO distributed deployment documentation:

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

MinIO user and permissions documentation:

    https://min.io/docs/minio/linux/administration/identity-access-management.html

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html