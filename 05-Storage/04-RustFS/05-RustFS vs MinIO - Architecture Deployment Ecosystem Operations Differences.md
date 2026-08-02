# RustFS vs. MinIO: Architecture, Deployment, Ecosystem, and Operations Differences

Suggested path: 05-Storage/04-RustFS/05-RustFS vs. MinIO: Architecture, Deployment, Ecosystem, and Operations Differences.md

Tags: #RustFS #MinIO #ObjectStorage #S3 #StructureComparison #DeploymentComparison #OperationComparison #CompatibilityValidation #mc #AWSCLI #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the fifth article of the RustFS module, focusing on the architecture, deployment, ecosystem, operations, security, and production suitability boundary comparison between RustFS and MinIO.

Previously completed:

    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Single-Machine and Cluster Modes
    03-RustFS Single-Machine Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multi-Node, Multi-Disk, and Access Entry

MinIO module has been completed:

    MinIO Single-Machine Docker Deployment
    MinIO Distributed Docker / VM Cluster Deployment
    MinIO mc Management
    MinIO Bucket / Object Operations
    MinIO Reverse Proxy Unified Entry
    Internal HTTP and External HTTPS
    Permissions, Security, Backup Migration, and Stage Summary

This article focuses on solving:

    What type of storage are RustFS and MinIO?
    Why are both S3-compatible object storage?
    What architectural similarities exist between the two?
    What deployment similarities exist between the two?
    How to compare their images, versions, ports, and data directories?
    How to compare their client access methods?
    How to compare their permissions, security, and HTTPS entry points?
    How to judge their ecosystem maturity?
    How to make trade-offs when selecting for production?
    How to use the same mc command to verify both MinIO and RustFS simultaneously?
    How to conduct a basic S3 compatibility comparison experiment?
    Why you cannot simply replace production MinIO with RustFS just because it's new and claims better performance?

This article emphasizes:

    RustFS and MinIO both belong to S3-compatible object storage.
    MinIO is a more mature object storage practice baseline.
    RustFS is a new object storage solution worth learning and verifying.
    Production replacement must be based on compatibility, stability, fault recovery, performance stress testing, and operations capability verification, rather than based on publicity or single experiment results.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the shared positioning of RustFS and MinIO.
2. Understand that both RustFS and MinIO are object storage, not block storage.
3. Understand that both RustFS and MinIO provide services through S3 API.
4. Understand the consistency of Bucket, Object, AccessKey, SecretKey, and Endpoint between the two.
5. Understand the architectural similarities between RustFS and MinIO.
6. Understand the differences between RustFS and MinIO in terms of maturity, ecosystem, and production cases.
7. Understand the deployment similarities between RustFS and MinIO.
8. Connect both MinIO and RustFS using mc.
9. Create identical Buckets in both MinIO and RustFS.
10. Upload the same batch of objects to both MinIO and RustFS.
11. Download objects from both MinIO and RustFS and perform sha256sum verification.
12. Use AWS CLI to verify basic S3 compatibility.
13. Compare their logs, ports, data directories, and reverse proxy configurations.
14. Design a validation process for migrating from MinIO to RustFS.
15. Explain the criteria for determining whether RustFS is suitable for replacing MinIO.
16. Formulate an object storage selection methodology.

---

## III. Core Conclusions First

### 3.1 RustFS and MinIO Belong to Object Storage

Common points:

    Both are S3-compatible object storage.
    Both use Bucket / Object as the core model.
    Both provide API through HTTP / HTTPS.
    Both can use AccessKey / SecretKey for authentication.
    Both can be accessed using mc, AWS CLI, and S3 SDK.
    Both are suitable for non-structured data such as images, attachments, backup packages, log archives, artifact packages, AI datasets, and model files.
    Both are unsuitable as real-time data directories for databases.
    Both do not replace block storage solutions like Longhorn, Ceph RBD, or cloud disks.

In one sentence:

    If an application needs object upload/download interface, consider MinIO / RustFS.
    If a Pod needs a persistent data disk, consider Longhorn / Ceph RBD / cloud disk.
    If multiple nodes need shared directory access, consider NFS / CephFS / NAS.

---

### 3.2 MinIO is a Mature Baseline, RustFS is a New Validation Object

MinIO's value:

    Higher maturity.
    More community and user case support.
    More comprehensive documentation, tools, and operational experience.
    Suitable as a baseline for learning S3 object storage.
    Suitable as a private object storage production candidate.

RustFS's value:

    Implemented using Rust.
    Positioned as S3-compatible object storage.
    Suitable as a new scheme for comparison and learning after MinIO.
    Suitable for validating compatibility, performance, deployment, and operations boundaries.
    Suitable as an evaluation object for object storage replacement options.

Production judgment:

    RustFS can be learned.
    RustFS can be tested.
    RustFS can be used for non-core pilot projects.
    Whether RustFS can replace production MinIO must be validated through complete testing.

---

### 3.3 Cannot Judge Solely by "Uploading and Downloading"

Object storage for production use must at least validate:

    S3 API compatibility
    Multipart Upload
    Presigned URL
    AWS SDK compatibility
    mc compatibility
    AWS CLI compatibility
    Permission model
    Bucket Policy
    User and key management
    Large object upload
    High concurrency for small objects
    Node fault recovery
    Disk fault recovery
    Capacity expansion
    Log auditing
    Monitoring and alerts
    Backup and migration
    Version upgrades
    Rollback plan
    HTTPS reverse proxy
    Production cases
    Community activity
    License and enterprise compliance

Conclusion:

    Being able to upload and download only indicates basic S3 operations are available.
    It cannot directly indicate suitability for production replacement.

---

## IV. Basic Positioning Comparison

| Comparison Item | MinIO | RustFS |
|---|---|---|
| Type | S3-compatible object storage | S3-compatible object storage |
| Core Model | Bucket / Object | Bucket / Object |
| Access Protocol | HTTP / HTTPS S3 API | HTTP / HTTPS S3 API |
| Client | mc / AWS CLI / SDK | mc / AWS CLI / SDK |
| Typical Use Cases | Images, attachments, backups, archives, AI data | Images, attachments, backups, archives, AI data |
| Is Block Storage | No | No |
| Is File System | No | No |
| Is Kubernetes CSI | No | No |
| Suitable for Database Data Directory | Not suitable | Not suitable |
| Suitable for Object Upload/Download | Suitable | Suitable, requires verification |
| Learning Positioning | Object storage mainline baseline | MinIO's successor comparison module |

---

## V. Architectural Philosophy Comparison

### 5.1 Common Architectural Philosophies

MinIO and RustFS share many similarities in their object storage architecture.

Common features:

    Provide S3 API externally.
    Serve client requests via HTTP / HTTPS.
    Use AccessKey / SecretKey for authentication.
    Manage object space via Bucket.
    Manage objects via Object Key.
    Support single-node and multi-node deployment modes.
    Support data directory mounting.
    Support unified access via Nginx / LB.
    Support access via mc or AWS CLI etc. S3 tools.
    Performance is affected by network, disk, CPU, object size, and concurrency.

---

### 5.2 Typical Object Write Path

When an application uploads an object:

    App / mc / AWS CLI
        |
        v
    S3 Endpoint
        |
        v
    MinIO / RustFS API Server
        |
        v
    Object Metadata / Object Manager
        |
        v
    Storage Engine
        |
        v
    Data Directory / Disk

Object storage focuses on:

    Object upload
    Object download
    Object listing
    Object deletion
    Object metadata
    Permission authentication
    Multi-node data reliability

Not focused on:

    File system mount
    Block device attach
    PVC Bound
    Pod VolumeMount
    Random write block device for databases

---

### 5.3 Difference with Longhorn Architecture

| Comparison Item | MinIO / RustFS | Longhorn |
|---|---|---|
| Type | Object storage | Block storage |
| External Interface | S3 API | CSI / PV / PVC |
| Users | Applications, SDKs, mc | Pods |
| Data Unit | Object | Volume |
| Top-level Container | Bucket | PVC / PV |
| Access Method | HTTP / HTTPS | File system mounting |
| Typical Scenarios | Attachments, backups, archives | Database data disks, application data directories |

One-sentence summary:

    MinIO / RustFS are object storage systems.
    Longhorn is Kubernetes persistent volume storage.
    They are not alternatives but complementary systems.

---

## VI. Deployment Mode Comparison

### 6.1 Single-node Deployment

MinIO single-node:

    Single node
    Single data directory or multiple data directories
    Docker startup
    API port typically 9000
    Console port typically 9001
    Suitable for learning and testing

RustFS single-node:

    Single node
    Single data directory or multiple data directories
    Docker startup
    API port uses 9000
    Console port uses 9001
    Suitable for learning and testing

Comparison:

| Comparison Item | MinIO single-node | RustFS single-node |
|---|---|---|
| Complexity | Low | Low |
| Learning Value | High | High |
| Production High Availability | No | No |
| Suitable for Validation | S3 basic capabilities | S3 basic capabilities |
| Focus | Mature baseline | New solution verification |

---

### 6.2 Distributed Deployment

MinIO distributed:

    Multi-node
    Multi-disk
    Erasure Coding
    Unified entry point
    Suitable for production evaluation

RustFS distributed:

    Multi-node
    Multi-disk
    Erasure Coding / distributed object storage mechanism
    Unified entry point
    Suitable for pre-production verification

Comparison:

| Comparison Item | MinIO distributed | RustFS distributed |
|---|---|---|
| Node Planning | Multi-node multi-disk | Multi-node multi-disk |
| Data Directory | Multi-disk paths | Multi-disk paths |
| Unified Entry Point | Nginx / LB | Nginx / LB |
| External HTTPS | Recommended / Mandatory for production | Recommended / Mandatory for production |
| Internal HTTP | Available in trusted networks | Available in trusted networks |
| Fault Simulation | Mandatory | Mandatory |
| Production Maturity | More mature | Requires verification |

---

### 6.3 Experiment Environment Comparison

MinIO planning example:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 |
| 10.0.0.45 | minio-client | mc client |
| 10.0.0.46 | minio-entry | Nginx unified entry point |

RustFS planning example:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | mc Client |
| 10.0.0.56 | rustfs-entry | Nginx Unified Entry |

Notes:

    Two independent experimental environments are planned.
    Do not mix data directories.
    Do not mix port entries.
    Do not overlap images and configurations.
    Facilitate independent learning, comparison, and cleanup.

---

## SevenI don't know.Image and Version Comparison

### 7.1 MinIO Fixed Version

MinIO module uses a fixed version:

    Official server image:
    minio/minio:RELEASE.2025-04-22T22-12-26Z

    User's Alibaba Cloud image:
    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

    mc Official image:
    minio/mc:RELEASE.2025-04-16T18-13-26Z

    User's Alibaba Cloud image:
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Selection reasons:

    Fixed version facilitates experiment reproducibility.
    Avoids differences caused by latest version changes in commands, interfaces, or features.
    mc and MinIO server version are close in time, facilitating compatibility verification.
    Use user's Alibaba Cloud image repository for more stable pulls under domestic network conditions.

---

### 7.2 RustFS Fixed Version

RustFS module uses a fixed version:

    Official image:
    rustfs/rustfs:1.0.0-alpha.99

    User's Alibaba Cloud image:
    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

Selection reasons:

    Fixed version facilitates experiment reproducibility.
    Avoids differences caused by latest version changes in commands, parameters, or features.
    Current version is suitable for learning and verification.
    Alpha stage is not recommended for direct use as a production core version.
    Production must re-evaluate stability, security fixes, upgrade paths, and recovery capabilities.

---

### 7.3 Version Management Common Principles

MinIO and RustFS should both follow:

    Do not use latest.
    Fixed version.
    Record source image.
    Record target image.
    Record synchronization method.
    Record synchronization time.
    Record version selection reasons.
    Test before production upgrades.
    Backup before production upgrades.
    Prepare rollback before production upgrades.

Image synchronization template:

    docker pull <official image>:<fixed version>

    docker tag <official image>:<fixed version> \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/<image name>:<fixed version>

    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/<image name>:<fixed version>

---

## EightI don't know.Port and Entry Comparison

### 8.1 Common Ports

| Type | MinIO | RustFS |
|---|---|---|
| S3 API | 9000 | 9000 |
| Console | 9001 | 9001 |
| Nginx HTTP | 80 or custom | 80 or custom |
| Nginx HTTPS | 443 | 443 |
| SSH | 22 | 22 |

Notes:

    Ports are similar, facilitating migration understanding.
    Same port cannot be occupied by multiple services on the same machine.
    Experimental environments should be planned separately by nodes.
    Production entries should be uniformly exposed via Nginx / LB as HTTPS.

---

### 8.2 Internal HTTP and External HTTPS

MinIO and RustFS can both adopt:

    Internal backend nodes: HTTP
    External client entry: HTTPS

Design principles:

    Internal HTTP is limited to trusted networks.
    Backend API ports should not be exposed to the public.
    External access must use HTTPS.
    Certificates are uniformly managed by Nginx / LB.
    Clients only configure a unified endpoint.
    Management entry is separately restricted by source.

Example:

    Client -> https://s3.example.com -> Nginx/LB -> http://Backend:9000

---

### 8.3 Unified Entry Comparison

MinIO entry:

    s3.minio.local
    minio-entry
    10.0.0.46

RustFS entry:

    s3.rustfs.local
    rustfs-entry
    10.0.0.56

Benefits of unified entry:

    Client configuration is not aware of backend nodes.
    Backend nodes can be maintained.
    Unified HTTPS.
    Unified logging.
    Unified upload size.
    Unified timeout.
    Unified source control.
    Unified error troubleshooting entry.

---

## NineI don't know.Data Directory and Disk Comparison

### 9.1 MinIO Data Directory

Single machine example:

    /data/minio

Multi-node multi-disk example:

    /data/minio/disk1
    /data/minio/disk2
    /data/minio/disk3
    /data/minio/disk4

Or:

    /data/minio0
    /data/minio1
    /data/minio2
    /data/minio3

---

### 9.2 RustFS Data Directory

Single machine example:

    /data/rustfs

Multi-node multi-disk example:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

---

### 9.3 Common Production Principles

Common principles:

    Use independent data disks.
    Do not store production object data on the system disk.
    Data disks recommend XFS or ext4.
    Officially recommends mature file systems.
    Data directories should not be mixed with logs, cache, or temporary directories.
    NFS is not recommended as the underlying data directory.
    Do not manually modify object storage internal data.
    Do not directly rm -rf data directories.
    Delete objects via S3 API / mc / Console.

---

## TenI don't know.Client Tool Comparison

### 10.1 mc /think

mc is the MinIO Client.

It can access not only MinIO but also other S3-compatible object storage.

Common capabilities:

    alias set
    mb
    rb
    ls
    cp
    mirror
    rm
    stat
    anonymous
    admin

MinIO uses mc:

    mc alias set minio http://10.0.0.41:9000 <ACCESS_KEY> <SECRET_KEY>

RustFS uses mc:

    mc alias set rustfs http://10.0.0.51:9000 <ACCESS_KEY> <SECRET_KEY>

Unified entry point:

    mc alias set minio https://s3.minio.local <ACCESS_KEY> <SECRET_KEY>
    mc alias set rustfs https://s3.rustfs.local <ACCESS_KEY> <SECRET_KEY>

---

### 10.2 AWS CLI

AWS CLI can access S3-compatible object storage.

MinIO:

    aws --endpoint-url http://10.0.0.41:9000 s3 ls

RustFS:

    aws --endpoint-url http://10.0.0.51:9000 s3 ls

Unified entry point:

    aws --endpoint-url https://s3.minio.local s3 ls
    aws --endpoint-url https://s3.rustfs.local s3 ls

Notes:

    AWS_ACCESS_KEY_ID must be configured.
    AWS_SECRET_ACCESS_KEY must be configured.
    Region is typically us-east-1.
    Check time synchronization, Endpoint, Region, and path-style behavior when signature errors occur.

---

### 10.3 SDK

Common SDKs:

    Java AWS SDK
    Python boto3
    Go AWS SDK
    Node.js AWS SDK
    Rust S3 SDK
    MinIO SDK

When accessing RustFS or MinIO, you typically need:

    endpoint
    access_key
    secret_key
    bucket
    region
    path_style
    ssl / tls

Production validation must cover SDKs.

You cannot rely solely on mc for validation.

---

## Eleven. Hands-on Exercise One: Prepare Comparison Environment

### 11.1 Experiment Objective

This section uses the same mc client to access both MinIO and RustFS simultaneously to verify:

    alias configuration
    Bucket creation
    Object upload
    Object download
    sha256sum verification
    Basic compatibility
    Migration approach

---

### 11.2 Prerequisites

MinIO has been deployed:

    Endpoint: http://10.0.0.41:9000
    AccessKey: minioadmin
    SecretKey: MinIOAdmin@123456

RustFS has been deployed:

    Endpoint: http://10.0.0.51:9000
    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

If using a unified entry point:

    MinIO Endpoint: http://s3.minio.local:9000 or https://s3.minio.local
    RustFS Endpoint: http://s3.rustfs.local:9000 or https://s3.rustfs.local

Notes:

    Account credentials are based on actual deployment.
    The passwords in this article are only for experimental purposes.
    Do not use experimental passwords in production environments.

---

### 11.3 Prepare mc Configuration Directory

Execute on the client node:

    mkdir -p /data/object-storage-compare/mc-config

Test mc:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

### 11.4 Configure MinIO alias

Execute:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-lab http://10.0.0.41:9000 minioadmin 'MinIOAdmin@123456'

Check:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-lab

---

### 11.5 Configure RustFS alias

Execute:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-lab http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-lab

---

### 11.6 View alias

Execute:

docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

Expected output:

    minio-lab
    rustfs-lab

---

## Twelve. Practical Exercise Two: Create Same Bucket

### 12.1 Create Comparison Bucket

Bucket name:

    s3-compare-demo

Create in MinIO:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-lab/s3-compare-demo

Create in RustFS:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-lab/s3-compare-demo

---

### 12.2 View Bucket

MinIO:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-lab

RustFS:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-lab

---

## Thirteen. Practical Exercise Three: Upload Same Object

### 13.1 Prepare Test Files

Execute on client node:

    mkdir -p /tmp/s3-compare-test
    cd /tmp/s3-compare-test

Create small file:

    echo "hello object storage compare" > hello.txt

Create configuration file:

    cat > app.conf <<'EOF'
    app_name=s3-compare-demo
    env=lab
    object_storage=minio-and-rustfs
    EOF

Create directory:

    mkdir -p logs data

Create log file:

    echo "2026-04-28 INFO compare minio and rustfs" > logs/app.log

Create JSON file:

    cat > data/user.json <<'EOF'
    {"id":1,"name":"ops-user","storage":["minio","rustfs"]}
    EOF

Create large file:

    dd if=/dev/zero of=file-100m.bin bs=1M count=100

Calculate checksum:

    sha256sum hello.txt app.conf logs/app.log data/user.json file-100m.bin > sha256-before.txt
    cat sha256-before.txt

---

### 13.2 Upload to MinIO

Execute:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-compare-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/ minio-lab/s3-compare-demo/

View:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive minio-lab/s3-compare-demo

---

### 13.3 Upload to RustFS

Execute:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-compare-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/ rustfs-lab/s3-compare-demo/

View:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-lab/s3-compare-demo

---

### 13.4 Compare Object Lists

MinIO: /think

docker run --rm \
  -v /data/object-storage-compare/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls --recursive minio-lab/s3-compare-demo > /tmp/minio-object-list.txt

RustFS:

  docker run --rm \
    -v /data/object-storage-compare/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    ls --recursive rustfs-lab/s3-compare-demo > /tmp/rustfs-object-list.txt

View:

  cat /tmp/minio-object-list.txt
  cat /tmp/rustfs-object-list.txt

---

## FourteenI don't know.Practical Operation Four: Download Objects and Verify

### 14.1 Download from MinIO

Create directory:

  mkdir -p /tmp/s3-compare-download/minio

Download:

  docker run --rm \
    -v /data/object-storage-compare/mc-config:/root/.mc \
    -v /tmp/s3-compare-download/minio:/download \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    cp --recursive minio-lab/s3-compare-demo/ /download/

View:

  find /tmp/s3-compare-download/minio -type f -print

---

### 14.2 Download from RustFS

Create directory:

  mkdir -p /tmp/s3-compare-download/rustfs

Download:

  docker run --rm \
    -v /data/object-storage-compare/mc-config:/root/.mc \
    -v /tmp/s3-compare-download/rustfs:/download \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    cp --recursive rustfs-lab/s3-compare-demo/ /download/

View:

  find /tmp/s3-compare-download/rustfs -type f -print

---

### 14.3 Verify MinIO Download Files

Execute:

  sha256sum /tmp/s3-compare-download/minio/hello.txt
  sha256sum /tmp/s3-compare-download/minio/app.conf
  sha256sum /tmp/s3-compare-download/minio/logs/app.log
  sha256sum /tmp/s3-compare-download/minio/data/user.json
  sha256sum /tmp/s3-compare-download/minio/file-100m.bin

Compare:

  cat /tmp/s3-compare-test/sha256-before.txt

---

### 14.4 Verify RustFS Download Files

Execute:

  sha256sum /tmp/s3-compare-download/rustfs/hello.txt
  sha256sum /tmp/s3-compare-download/rustfs/app.conf
  sha256sum /tmp/s3-compare-download/rustfs/logs/app.log
  sha256sum /tmp/s3-compare-download/rustfs/data/user.json
  sha256sum /tmp/s3-compare-download/rustfs/file-100m.bin

Compare:

  cat /tmp/s3-compare-test/sha256-before.txt

---

### 14.5 Verification Conclusion

If the sha256sum of the downloaded files from both MinIO and RustFS match the pre-upload values, it indicates:

  Basic PutObject functionality is normal.
  Basic GetObject functionality is normal.
  Basic object content is intact.
  mc basic access compatibility is confirmed.
  Basic testing for small files and 100Mi files is passed.

However, this still cannot confirm:

  Full compatibility with Multipart Upload.
  Full compatibility with Presigned URL.
  Full SDK compatibility.
  High concurrency performance.
  Fault recovery capability.
  Production readiness for direct replacement.

---

## FifteenI don't know.Practical Operation Five: AWS CLI Basic Comparison

### 15.1 Install AWS CLI

Ubuntu example:

  apt update
  apt install -y awscli

View:

  aws --version

---

### 15.2 Configure MinIO Environment Variables

Execute:

  export AWS_ACCESS_KEY_ID="minioadmin"
  export AWS_SECRET_ACCESS_KEY="MinIOAdmin@123456"
  export AWS_DEFAULT_REGION="us-east-1"

View MinIO Bucket:

  aws --endpoint-url http://10.0.0.41:9000 s3 ls

View objects:

  aws --endpoint-url http://10.0.0.41:9000 s3 ls s3://s3-compare-demo/ --recursive

---

### 15.3 Configure RustFS Environment Variables

Execute:

export AWS_ACCESS_KEY_ID="rustfsadmin"
export AWS_SECRET_ACCESS_KEY="RustFSAdmin@123456"
export AWS_DEFAULT_REGION="us-east-1"

View RustFS Bucket:

    aws --endpoint-url http://10.0.0.51:9000 s3 ls

View Objects:

    aws --endpoint-url http://10.0.0.51:9000 s3 ls s3://s3-compare-demo/ --recursive

---

### 15.4 AWS CLI Upload Comparison

Create File:

    echo "aws cli compare test" > /tmp/aws-cli-compare.txt

Upload to MinIO:

    export AWS_ACCESS_KEY_ID="minioadmin"
    export AWS_SECRET_ACCESS_KEY="MinIOAdmin@123456"
    export AWS_DEFAULT_REGION="us-east-1"

    aws --endpoint-url http://10.0.0.41:9000 \
      s3 cp /tmp/aws-cli-compare.txt s3://s3-compare-demo/aws-cli-compare.txt

Upload to RustFS:

    export AWS_ACCESS_KEY_ID="rustfsadmin"
    export AWS_SECRET_ACCESS_KEY="RustFSAdmin@123456"
    export AWS_DEFAULT_REGION="us-east-1"

    aws --endpoint-url http://10.0.0.51:9000 \
      s3 cp /tmp/aws-cli-compare.txt s3://s3-compare-demo/aws-cli-compare.txt

---

### 15.5 Common AWS CLI Issues

If MinIO is available but RustFS is not, focus on checking:

    Current RustFS S3 compatibility version.
    Endpoint correctness.
    Region consistency.
    AccessKey/SecretKey accuracy.
    System time synchronization.
    Path-style requirement.
    SignatureDoesNotMatch error.
    AccessDenied error.
    501 Not Implemented error.

If RustFS is available but MinIO is not, focus on checking:

    MinIO service status.
    MinIO account credentials.
    Bucket existence.
    MinIO Console and API confusion.
    Nginx entry forwarding correctness.

---

## Sixteen, Permissions and Security Comparison

### 16.1 Common Security Principles

Both MinIO and RustFS should follow:

    Avoid weak passwords.
    Do not use default keys long-term.
    Root/Admin accounts only for initialization.
    Independent AccessKey for business use.
    Do not commit AccessKey/SecretKey to Git.
    Different buckets for different businesses.
    Principle of least privilege.
    Caution in granting deletion permissions.
    External access must use HTTPS.
    Management entry not exposed to public internet.
    Access logs need to be retained.
    Keys must be disableable and rotatable after leakage.

---

### 16.2 MinIO Security Ecosystem

MinIO's security capabilities typically include:

    User management
    Group management
    Policy management
    Bucket Policy
    AccessKey/SecretKey
    Console management
    mc admin
    TLS
    KMS/Encryption capabilities
    Audit logs
    Integration with external identity systems

MinIO's security governance experience is more mature in production.

---

### 16.3 RustFS Security Verification Focus

RustFS needs to focus on verifying:

    Whether user management meets requirements.
    Whether policies are compatible with business.
    Whether bucket permissions meet requirements.
    Whether read-only, read-write, and delete permissions can be split.
    Whether business key rotation is supported.
    Whether audit logs are supported.
    Whether HTTPS reverse proxy is stable.
    Whether console management entry is controllable.
    Whether SDK permission behavior is consistent with existing systems.

Conclusion:

    RustFS replacing MinIO, permission system is the key verification item.
    Cannot only verify root account upload/download.

---

## Seventeen, Logs and Operations Comparison

### 17.1 MinIO Logs

MinIO common log entries:

    docker logs minio
    mc admin trace
    mc admin info
    mc admin heal
    mc admin user
    mc admin policy
    Console
    Nginx access.log
    Nginx error.log

MinIO's operations toolchain is more mature.

---

### 17.2 RustFS Logs

RustFS current experimental common log entries:

    docker logs rustfs-single
    docker logs rustfs-cluster
    Console
    Nginx access.log
    Nginx error.log
    mc ls / cp / stat
    AWS CLI output

RustFS needs to focus on confirming before production:

    Whether complete admin tool capabilities exist.
    Whether metrics are exposed.
    Whether Prometheus support is available.
    Whether audit logs are supported.
    Whether fault diagnosis commands are supported.
    Whether cluster status viewing is supported.
    Whether user and policy management commands are supported.

---

### 17.3 Daily Patrol Comparison

MinIO Patrol:

    docker ps | grep minio
    docker logs minio --tail=100
    mc admin info minio
    mc ls minio
    df -hT /data/minio
    du -sh /data/minio
    curl -i http://minio-endpoint/minio/health/live

RustFS Patrol:

docker ps | grep rustfs
docker logs rustfs-cluster --tail=100
mc ls rustfs
df -hT /data/rustfs*
du -sh /data/rustfs*
curl -i http://rustfs-endpoint/health

Common Inspection:

    Is the service running?
    Are the ports listening?
    Is the Endpoint accessible?
    Can the Bucket be listed?
    Is upload/download functioning normally?
    Is disk capacity sufficient?
    Does Nginx have 4xx/5xx errors?
    Is the certificate expired?
    Is time synchronized?

---

## EighteenI don't know.Performance Comparison Method

### 18.1 Do Not Trust Single Benchmark Directly

Object storage performance is influenced by many factors:

    Disk type
    Network bandwidth
    CPU
    Memory
    Object size
    Concurrency level
    Erasure Coding configuration
    Whether TLS is used
    Whether Nginx is used
    Whether compression or encryption is enabled
    Client location
    File system
    Number of data disks
    Number of nodes

Therefore:

    Do not directly use official benchmark as production conclusion.
    Do not judge performance by single 100Mi file upload.
    Do not directly compare results across different hardware.
    Do not ignore performance costs from data reliability.

---

### 18.2 Basic Performance Testing Dimensions

Recommended testing:

| Test Item | Example |
|---|---|
| Small Object Upload | 4KiB, 16KiB, 64KiB |
| Medium Object Upload | 1MiB, 10MiB |
| Large Object Upload | 100MiB, 1GiB, 10GiB |
| Concurrent Upload | 10, 50, 100 concurrent |
| Concurrent Download | 10, 50, 100 concurrent |
| Multipart Upload | Large file chunk upload |
| HTTPS Overhead | HTTP vs HTTPS |
| Nginx Overhead | Direct connection vs unified entry |
| Node Failure | Read/write after stopping one node |
| Disk Pressure | Read/write under high capacity water level |

---

### 18.3 Simple Concurrent Upload Test Example

Prepare files:

    mkdir -p /tmp/s3-perf-test/small
    for i in $(seq 1 1000); do echo "object-$i" > /tmp/s3-perf-test/small/object-$i.txt; done

Upload to MinIO:

    time docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-perf-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/small/ minio-lab/s3-compare-demo/small/

Upload to RustFS:

    time docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-perf-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/small/ rustfs-lab/s3-compare-demo/small/

Notes:

    This is basic observation, not formal benchmarking.
    Formal benchmarking should use professional tools, fixed variables, repeated testing, and record system metrics.

---

## NineteenI don't know.Compatibility Comparison Checklist

### 19.1 Basic S3 Operations

Must verify:

    CreateBucket
    DeleteBucket
    PutObject
    GetObject
    DeleteObject
    ListObjects
    HeadObject
    CopyObject

---

### 19.2 Advanced S3 Capabilities

Recommended verification before production:

    Multipart Upload
    Presigned URL
    Bucket Policy
    Object Tagging
    Object Metadata
    Range Download
    Versioning
    Lifecycle
    Object Lock
    SSE Encryption
    CORS
    Event Notification
    SDK Compatibility
    Large Object Resumable Upload
    Batch Delete
    Cross Bucket Copy

Notes:

    Different object storages support advanced S3 capabilities to varying degrees.
    You must verify what your business uses.
    Record "not in current business scope" for unused capabilities.

---

### 19.3 SDK Compatibility Testing

Recommended at least verify:

    Python boto3
    Java AWS SDK
    Go AWS SDK
    Node.js AWS SDK
    SDK used by the application

Test content:

    Initialize client.
    Create Bucket.
    Upload object.
    Download object.
    List objects.
    Delete object.
    Generate presigned URL.
    Multipart Upload.
    Exception handling.

---

## TwentyI don't know.Migration Approach Comparison

### 20.1 Principles for Migrating from MinIO to RustFS

Not recommended to replace in-place directly.

Recommended process:

    1. Build RustFS test cluster.
    2. Use same version business SDK to verify compatibility.
    3. Create same Bucket.
    4. Use mc mirror for data synchronization.
    5. Compare object count and capacity.
    6. Sample download and verify sha256sum.
    7. Verify application read/write RustFS.
    8. Verify permission policies.
    9. Verify HTTPS Endpoint.
    10. Verify large and small objects.
    11. Perform stress testing.
    12. Perform node failure drill.
    13. Prepare rollback plan.
    14. Gradually switch non-core business.
    15. Observe stability before expanding scope.

---

### 20.2 mc mirror Migration Example

From MinIO to RustFS: /think

docker run --rm \
  -v /data/object-storage-compare/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  mirror --overwrite minio-lab/s3-compare-demo rustfs-lab/s3-compare-demo

Syncing back from RustFS to MinIO:

  docker run --rm \
    -v /data/object-storage-compare/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    mirror --overwrite rustfs-lab/s3-compare-demo minio-lab/s3-compare-demo

Notes:

  mirror is one of the migration approaches.
  Production migration needs to consider incremental sync, write freeze, data verification, permission migration, and rollback.
  It is not recommended to only synchronize object data while ignoring account, policy, bucket configuration, and application configuration.

---

### 20.3 Migration Verification

Verification directions:

  Bucket count
  Object count
  Object size
  Object metadata
  Permission policies
  Application read/write
  Download verification
  Random sampling
  Full verification
  Log errors
  Business access error rate

Base commands:

  mc ls --recursive minio-lab/s3-compare-demo | wc -l
  mc ls --recursive rustfs-lab/s3-compare-demo | wc -l

Docker version:

  docker run --rm \
    -v /data/object-storage-compare/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    ls --recursive minio-lab/s3-compare-demo | wc -l

  docker run --rm \
    -v /data/object-storage-compare/mc-config:/root/.mc \
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
    ls --recursive rustfs-lab/s3-compare-demo | wc -l

---

## Twenty-one, Production Selection Judgment Table

| Judgment item | Preference for MinIO | Preference for RustFS |
|---|---|---|
| Production maturity priority | Yes | Caution needed |
| Existing MinIO experience | Yes | Can be validated |
| Need for rapid deployment | Yes | Not recommended for core deployment |
| Pursuit of new solution validation | Can be compared | Yes |
| Need for Apache 2.0 direction assessment | Depends on version and scenario | Yes |
| Need for extensive production cases | Yes | Needs confirmation |
| Need for compatibility with existing S3 tools | Yes | Needs verification |
| Need for core business replacement | More stable | Needs complete PoC |
| Non-core pilot | Can be | Can be |
| Learning object storage principles | Yes | Yes |
| Learning new technology evaluation methods | Can be | More suitable |

---

## Twenty-two, Pre-checklist Before Replacing MinIO with RustFS

### 22.1 Technical Check

| Check item | Pass |
|---|---|
| Basic S3 API |  |
| Multipart Upload |  |
| Presigned URL |  |
| Bucket Policy |  |
| User / Key Management |  |
| SDK Compatibility |  |
| mc Compatibility |  |
| AWS CLI Compatibility |  |
| Large object upload |  |
| High concurrency for small objects |  |
| Range Download |  |
| HTTPS Reverse Proxy |  |
| Nginx Unified Entry |  |
| Node failure recovery |  |
| Disk failure recovery |  |
| Version upgrade |  |
| Data migration |  |
| Rollback plan |  |

---

### 22.2 Operations Check

| Check item | Pass |
|---|---|
| Readable logs |  |
| Access logs |  |
| Error logs |  |
| Audit capabilities |  |
| Monitoring metrics |  |
| Prometheus integration |  |
| Capacity statistics |  |
| Bucket statistics |  |
| Object count statistics |  |
| Alarm rules |  |
| Fault record template |  |
| Backup strategy |  |
| Recovery drill |  |

---

### 22.3 Security Check

| Check item | Pass |
|---|---|
| HTTPS |  |
| Certificate renewal |  |
| Management entry isolation |  |
| Strong password |  |
| Minimum permissions |  |
| Business-independent keys |  |
| Key rotation |  |
| Key leakage handling |  |
| Bucket deletion protection |  |
| Access source restriction |  |
| Nginx upload size limit |  |
| Nginx timeout settings |  |

---

## Twenty-three, Common Misunderstandings

### 23.1 Error: RustFS and MinIO both support S3, so they are completely the same

Error.

S3 compatibility does not mean 100% identical behavior.

Must verify:

  API support scope.
  Error codes.
  SDK behavior.
  Permission policies.
  Multipart Upload.
  Presigned URL.
  Metadata handling.
  Large objects.
  Small objects.
  Concurrency.
  Fault recovery.

---

### 23.2 Error: RustFS claims better performance, so it must replace MinIO

Error.

Performance is just one of the selection factors.

Also consider:

  Stability.
  Data reliability.
  Operations capability.
  Monitoring capability.
  Recovery capability.
  Ecosystem maturity.
  Fault diagnosis capability.
  Upgrade strategy.
  Production cases.
  Team familiarity.

---

### 23.3 Error: mc can access means the application can access

Error.

mc being able to access only indicates basic operations are available.

Applications may involve: /think

SDK Version.  
path-style.  
virtual-hosted-style.  
region.  
presigned URL.  
multipart upload.  
retry logic.  
timeout.  
metadata.  
CORS.  
signature method.  

---

### 23.4 Error: MinIO and RustFS cannot share the same data directory  

Error.  

Different object storage systems cannot arbitrarily share the underlying data directory.  

Migration should be performed via:  

    S3 API  
    mc mirror  
    official migration tools  
    backup recovery process  

Should NOT:  

    Directly copy internal data directory.  
    Allow two services to write the same directory simultaneously.  
    Manually modify internal metadata.  

---

### 23.5 Error: Object storage migration only requires copying files  

Error.  

Additional considerations are required:  

    Bucket  
    Object  
    Metadata  
    Policy  
    User  
    Group  
    AccessKey  
    SecretKey  
    Lifecycle  
    Versioning  
    Application Config  
    DNS  
    HTTPS  
    Rollback  

---

## Twenty-Four, Experiment Cleanup  

### 24.1 Delete Objects in Comparison Bucket  

Delete MinIO objects:  

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force minio-lab/s3-compare-demo  

Delete RustFS objects:  

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force rustfs-lab/s3-compare-demo  

---

### 24.2 Delete Bucket  

Delete MinIO Bucket:  

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb minio-lab/s3-compare-demo  

Delete RustFS Bucket:  

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb rustfs-lab/s3-compare-demo  

High-risk warning:  

    Production bucket deletion must go through approval.  
    Confirm business ownership before deletion.  
    Confirm backups before deletion.  
    Confirm recovery plan before deletion.  

---

### 24.3 Delete Temporary Files  

Execute:  

    rm -rf /tmp/s3-compare-test  
    rm -rf /tmp/s3-compare-download  
    rm -rf /tmp/s3-perf-test  
    rm -f /tmp/minio-object-list.txt  
    rm -f /tmp/rustfs-object-list.txt  
    rm -f /tmp/aws-cli-compare.txt  

---

### 24.4 Delete mc Configuration  

If not used anymore:  

    rm -rf /data/object-storage-compare/mc-config  

If further migration and comparison experiments are planned, it is recommended to retain.  

---

## Twenty-Five, Completion Criteria for This Article  

After completing this article, the following should be at least achieved:  

| Item | Standard |  
|---|---|  
| Theoretical Understanding | Clearly understand that MinIO and RustFS are both S3 object storage |  
| Type Boundary | Clearly understand that neither replaces Longhorn |  
| Version Awareness | Clearly understand that MinIO is a mature baseline, RustFS is a new validation object |  
| mc alias | Configure both minio-lab and rustfs-lab |  
| Bucket | Create s3-compare-demo on both sides |  
| Object Upload | Upload the same batch of objects to MinIO and RustFS |  
| Object Download | Download objects from both sides |  
| Data Verification | sha256sum consistency |  
| AWS CLI | Basic s3 ls / cp testing on both sides |  
| Operations Comparison | Ability to compare logs, ports, directories, and entry points |  
| Migration Understanding | Understand the mc mirror migration approach |  
| Selection Capability | Ability to explain when to use MinIO and when to evaluate RustFS |  
| Risk Awareness | Clearly understand that RustFS replacement of MinIO requires a complete PoC before |  

---

## Twenty-Six, Interview Answer Approach  

If asked in an interview:  

    What are the differences between RustFS and MinIO? How would you choose?  

You can answer:

# RustFS and MinIO: S3-Compatible Object Storage Comparison

Both RustFS and MinIO belong to the category of S3-compatible object storage systems. They share a core model based on Bucket and Object, and both provide services through HTTP/HTTPS S3 APIs. They can all be accessed using mc, AWS CLI, and various S3 SDKs. They are both suitable for storing unstructured data such as images, attachments, backup packages, log archives, artifact packages, AI datasets, and model files.

Neither is a block storage solution, and they do not replace Longhorn, Ceph RBD, or cloud disks. For example, database real-time data directories should use block storage or local disks, not directly place objects in object storage. Object storage is more suitable for uploading and downloading objects via API.

MinIO has the advantage of higher maturity, more production cases, and a more mature toolchain and operational experience, making it more suitable as a mature baseline for private S3 object storage. RustFS is a new S3-compatible object storage solution implemented in Rust, suitable as a learning and validation option after MinIO. Whether it can replace production MinIO depends on actual compatibility, stability, fault recovery, permission systems, monitoring logs, and performance stress test results.

If evaluating RustFS as a replacement for MinIO, I would not only check if it can upload and download. I would first set up a RustFS test cluster, use the same mc client to connect both MinIO and RustFS simultaneously, create identical Buckets, upload the same batch of objects, and perform sha256sum verification after downloading. Then continue validating AWS CLI, business SDKs, Multipart Upload, Presigned URL, Bucket Policy, large object uploads, high-concurrency small object operations, HTTPS reverse proxy, node failure recovery, disk failure recovery, version upgrades, and data migration.

For migration, I would prioritize data synchronization through S3 API or mc mirror, rather than directly copying the underlying data directory. Before migration, object counts, capacity, metadata, permissions, and application read/write verification should be conducted, along with a rollback plan.

Thus, my selection principle is: when production stability is prioritized, MinIO is preferred; for learning new solutions, evaluating Apache 2.0 S3-compatible object storage, or non-core pilots, RustFS can be considered. RustFS should not be directly replaced for production MinIO just because it's new or has strong performance claims. It must undergo a complete PoC before gradual gray-scale deployment.

---

## Twenty-Seven, Summary of This Article

This article completed a comparative study of RustFS and MinIO:

1. Both RustFS and MinIO belong to S3-compatible object storage.
2. Both are based on the Bucket/Object core model.
3. Both can be accessed via mc, AWS CLI, and SDKs.
4. Both are suitable for images, attachments, backup packages, log archives, and AI data.
5. Neither is block storage.
6. Neither replaces Longhorn.
7. MinIO is a more mature object storage practice baseline.
8. RustFS is a worthy validation option for new object storage solutions.
9. Both have deployment modes including single-node and multi-node multi-disk.
10. Both should fix image versions and avoid using latest.
11. Both production environments should use independent data disks.
12. Both should use HTTPS for external access.
13. Both should not expose backend node ports directly to the public.
14. mc can access both MinIO and RustFS simultaneously.
15. Basic upload/download does not equal full production compatibility.
16. Before replacing MinIO with RustFS, SDK, Multipart, Presigned URL, Policy, and other capabilities must be validated.
17. Migration should use S3 API, mc mirror, or official tools, not directly share underlying data directories.
18. Production selection should comprehensively consider maturity, performance, reliability, ecosystem, operations, and compliance.
19. RustFS is more suitable for learning, validation, non-core pilots, and gradual evaluation of production value.
20. The next article will study RustFS client access: S3 API, mc tools, and application integration.

---

## Twenty-Eight, Reference Documents

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS comparison with other storage products:

    https://docs.rustfs.com/concepts/comparison.html

RustFS S3 compatibility:

    https://docs.rustfs.com/features/s3-compatibility/

RustFS Docker installation documentation:

    https://docs.rustfs.com/installation/docker/

RustFS multi-node multi-disk installation documentation:

    https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html

RustFS Nginx reverse proxy configuration:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS TLS configuration:

    https://docs.rustfs.com/integration/tls-configuration/

RustFS GitHub:

    https://github.com/rustfs/rustfs

RustFS Docker Hub:

    https://hub.docker.com/r/rustfs/rustfs

MinIO official documentation:

    https://min.io/docs/minio/linux/index.html

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO Server documentation:

    https://min.io/docs/minio/linux/reference/minio-server/minio-server.html

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

AWS CLI S3 documentation:

    https://docs.aws.amazon.com/cli/latest/reference/s3/

Nginx official documentation:

    https://nginx.org/en/docs/

Docker official documentation:

    https://docs.docker.com/