# RustFS Basics: S3-Compatible Object Storage and Use Cases

Recommended path: 05-Storage/04-RustFS/01-RustFS Basics: S3-Compatible Object Storage and Use Cases.md

Tags: #RustFS #ObjectStorage #S3 #Bucket #Object #MinioComparison #Docker #mc #AWSCLI #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This article is the first in the RustFS module, focusing on understanding RustFS's basic positioning, S3 object storage model, and applicable scenarios.

This article does not directly enter deployment; deployment practices will be covered in subsequent articles 03 and 04.

This article addresses the following key questions:

    What is RustFS
    What is the relationship between RustFS and S3 object storage
    What is a Bucket
    What is an Object
    What is AccessKey / SecretKey
    What is an S3 Endpoint
    What is the relationship between RustFS and MinIO
    What are the differences between RustFS and Longhorn, Ceph, NFS
    What scenarios is RustFS suitable for
    What scenarios is RustFS not suitable for
    Why RustFS is currently more suitable as a learning, verification, comparison, and pilot evaluation object
    What capabilities must be validated before production use of RustFS

This article emphasizes a core principle:

    RustFS can be used as a new type of S3 object storage solution for learning and verification.
    But it cannot be directly used as a mature production storage replacement for MinIO or Ceph RGW due to its S3 compatibility.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand RustFS's basic positioning.
2. Understand that RustFS is S3-compatible object storage, not block storage.
3. Understand the basic concepts of Bucket, Object, Prefix, and Endpoint.
4. Understand the role of AccessKey / SecretKey.
5. Understand the relationship between RustFS and MinIO.
6. Understand the differences between RustFS and Longhorn.
7. Understand the differences between RustFS and Ceph RGW.
8. Understand what types of data RustFS is suitable for storing.
9. Understand what types of data RustFS is not suitable for storing.
10. Clearly explain the differences between object storage, file systems, and block storage.
11. Determine whether a business is suitable for integration with RustFS.
12. List the capabilities that must be validated before production evaluation of RustFS.
13. Lay the foundation for subsequent single-node Docker deployment, cluster deployment, client access, and reverse proxy implementation.

---

## III. What is RustFS

RustFS is an S3-compatible object storage system written in Rust.

It can be simply understood as:

    RustFS is a private S3 object storage service.

It provides access interfaces similar to AWS S3, allowing applications or clients to access RustFS via S3 API.

Common access methods include:

    mc tool
    AWS CLI
    S3 SDK
    Application S3 client
    Backup programs
    Log archiving programs
    AI / big data system object storage backend

Core keywords of RustFS:

    Rust
    S3 Compatible
    Object Storage
    Bucket
    Object
    AccessKey
    SecretKey
    Endpoint
    Distributed Object Storage
    MinIO Alternative
    Apache 2.0

---

## IV. What RustFS Is Not

RustFS is not block storage.

It is not:

    Longhorn
    Ceph RBD
    Cloud disks
    Local disks
    Kubernetes PVC block volumes

RustFS is not file system storage.

It is not:

    NFS
    SMB
    CephFS
    NAS shared directories

RustFS is not a Kubernetes CSI storage plugin.

It is not:

    PV / PVC dynamic provisioner
    StorageClass backend
    Pod-mounted disk

RustFS is object storage.

Its usage method is:

    Applications upload objects via S3 API
    Applications download objects via S3 API
    Applications manage Buckets and Objects via S3 API

---

## V. Object Storage Fundamentals

### 5.1 What is Object Storage

Object storage is a storage method for storing unstructured data.

Object storage stores individual objects rather than traditional directories and block devices.

Objects typically include:

    Object data
    Object name
    Metadata
    Bucket it belongs to
    Access permissions
    Version information

Common objects include:

    Images
    Videos
    Documents
    Attachments
    Backup packages
    Log archives
    Model files
    Datasets
    Artifact packages
    Static resources
    Large file download packages

---

### 5.2 What is S3

S3 was originally AWS's object storage service.

Later, the S3 API became a de facto standard.

Many private object storage systems provide S3-compatible interfaces, such as:

    MinIO
    Ceph RGW
    RustFS
    SeaweedFS
    Object storage interfaces compatible with cloud vendors

The significance of S3 compatibility is:

    Applications can use a unified S3 SDK.
    Tools can access via a unified S3 protocol.
    Object storage migration requires minimal application changes.
    Tools like mc and AWS CLI can be used for management.

---

### 5.3 What is a Bucket

A Bucket is the top-level container in object storage.

It can be understood as:

    A Bucket is similar to an object storage space.

For example:

    images
    backups
    logs
    devops-artifacts
    longhorn-backup
    app-uploads
    model-files

Buckets are used to organize objects.

A RustFS service can have multiple Buckets.

Different business scenarios should use different Buckets.

---

### 5.4 What is an Object

An Object is the data entity in object storage.

For example:

    images/avatar-001.png
    backups/mysql/full-2026-04-28.sql.gz
    logs/nginx/2026/04/28/access.log.gz
    models/llm/model.bin
    artifacts/app-v1.2.0.tar.gz

Objects are identified by Keys.

For example:

    backups/mysql/full-2026-04-28.sql.gz

This entire string is the Object Key.

### 5.5 What is a Prefix

Object storage does not have a traditional directory structure.

For example, an object name:

    logs/nginx/2026/04/28/access.log.gz

Here:

    logs/
    logs/nginx/
    logs/nginx/2026/

are more like prefixes of the object key.

Object storage clients display these prefixes as directory-like structures.

But fundamentally:

    Bucket + Object Key is the way to locate objects.

---

### 5.6 What is an Endpoint

An Endpoint is the access entry point for object storage.

For example:

    http://10.0.0.51:9000
    http://10.0.0.56:9000
    https://s3.rustfs.local
    https://rustfs.example.com

When clients access RustFS, they need to configure:

    Endpoint
    AccessKey
    SecretKey
    Bucket
    Region

In private object storage, Region is typically set to:

    us-east-1

The specific configuration will depend on the actual setup of RustFS and client requirements.

---

### 5.7 What are AccessKey and SecretKey

AccessKey and SecretKey are authentication credentials for accessing object storage.

Similar to:

    Username
    Password

Clients accessing RustFS typically need:

    AccessKey
    SecretKey
    Endpoint

Security principles:

    Do not write AccessKey / SecretKey into public Git.
    Do not give Root account to business for long-term use.
    Different businesses should use different credentials.
    Credentials should be disabled and rotated after leakage.
    Permissions must be minimized.
    Granting deletion permissions should be done cautiously.

---

## Six, Differences Between Object Storage, Block Storage, and File Storage

### 6.1 Object Storage

Object storage represents:

    RustFS
    MinIO
    Ceph RGW
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS

Access methods:

    HTTP / HTTPS
    S3 API
    SDK
    mc
    AWS CLI

Suitable for:

    Images
    Attachments
    Backup packages
    Log archives
    Large files
    Model files
    Static resources
    Unstructured data

Not suitable for:

    Direct use as database data directory
    Direct mounting as local disk for Pod
    Low-latency random write block device scenarios

---

### 6.2 Block Storage

Block storage represents:

    Longhorn
    Ceph RBD
    Cloud disks
    Local disks
    SAN LUN

Access methods:

    Mount as file system
    Use as disk for operating system
    Kubernetes PV / PVC

Suitable for:

    Database data directory
    Redis persistence
    Prometheus data directory
    Jenkins home
    Application runtime data directory

Not suitable for:

    Massive object upload/download
    Image attachment archive
    Large amount of static resource objects service

---

### 6.3 File Storage

File storage represents:

    NFS
    SMB
    CephFS
    NAS

Access methods:

    Mounting
    Shared directory
    POSIX file path

Suitable for:

    Multi-node shared files
    Traditional application shared directory
    File server
    RWX scenarios

Not suitable for:

    Massive object API scenarios
    High-concurrency object upload/download
    S3 API ecosystem integration

---

### 6.4 Comparison of the Three

| Type | Representative | Access Method | Typical Use Cases |
|---|---|---|---|
| Object Storage | RustFS / MinIO / S3 | HTTP / HTTPS / S3 API | Images, attachments, backups, archives |
| Block Storage | Longhorn / Ceph RBD | Mount as disk or PVC | Database, application data disk |
| File Storage | NFS / CephFS | Shared directory mounting | Multi-node shared files |

In one sentence:

    Use RustFS / MinIO for uploading objects.
    Use Longhorn for a Pod's persistent disk.
    Use NFS / CephFS for multi-node shared directories.

---

## Seven, Scenarios Suitable for RustFS

### 7.1 Image and Attachment Storage

Suitable for:

    User avatar
    Product images
    Resume attachments
    PDF files
    Word documents
    Table files
    Uploaded files

Examples:

    app-uploads/avatar/user-1001.png
    app-uploads/resume/user-1001/resume.pdf
    app-uploads/document/2026/04/report.docx

Business integration method:

    Backend applications use S3 SDK to upload to RustFS.
    Database only stores object path, Bucket, and Object Key.
    Users access through backend-generated download links or pre-signed URLs.

---

### 7.2 Backup Package Storage

Suitable for:

    MySQL backup
    PostgreSQL backup
    GitLab backup
    Jenkins backup
    Configuration backup
    K8s YAML backup
    Longhorn Backup Target verification

Examples:

    backups/mysql/full-2026-04-28.sql.gz
    backups/postgresql/base-2026-04-28.tar.gz
    backups/gitlab/gitlab-backup-2026-04-28.tar

Notes:

    When RustFS itself is a backup target, consider its own data protection and recovery capabilities.
    Do not place unique backups and production data in the same fault domain.

---

### 7.3 Log Archiving

Suitable for:

    Nginx historical log archiving
    Application log compression archiving
    Audit log archiving
    Security log archiving

Examples:

    logs/nginx/2026/04/28/access.log.gz
    logs/app/devatlas/2026/04/28/app.log.gz
    logs/audit/2026/04/audit.log.gz

Object storage is suitable for storing archival logs, but not for directly replacing query systems like Elasticsearch / Loki.

Correct understanding:

    RustFS stores archival logs.
    Elasticsearch / Loki handles retrieval and querying.
    They are not the same role.

---

### 7.4 Artifacts and Release Packages

Suitable for:

# Application Build Artifacts

## Binary Packages
Offline Installation Packages
Image Export tar Packages
Helm Chart Archives
Release Packages

Example:

artifacts/devatlas/devatlas-api-v1.0.0.tar.gz
artifacts/scripts/deploy-v20260428.tar.gz
artifacts/images/nginx-1.25.tar

Note:

If it's a container image, prioritize using Harbor.
RustFS is more suitable for saving regular file artifacts and does not replace image repositories.

---

### 7.5 AI Data and Model Files

Suitable for:

Data Sets
Model Files
Embedding Files
Intermediate Training Results
Large File Archives

Example:

ai-datasets/logs/2026/nginx-sample.tar.gz
ai-models/model-v1/model.bin
ai-output/batch-001/result.jsonl

Note:

AI scenario objects may be very large.
Must verify large object upload/download, Multipart Upload, resumable transfer, concurrent access, and permission control.

---

### 7.6 Private S3 Compatible Environment Validation

RustFS is also suitable for:

Learning S3 API
Validating client compatibility
Comparing with MinIO
Validating application object storage access
Testing S3 SDK
Testing backup tools support for S3 Endpoint
Testing reverse proxy with HTTPS

---

## VIII. Scenarios RustFS is Not Suitable For

### 8.1 Not Suitable as Direct Database Data Directory

Incorrect Practices:

Putting MySQL datadir in RustFS
Putting PostgreSQL data in RustFS
Writing Redis AOF directly to RustFS

Reasons:

Databases require local block devices or file system semantics.
RustFS is object storage, not POSIX File System.
S3 API is unsuitable for database random write files.
Database consistency and object storage semantics are incompatible.

Database backups can be stored in RustFS.

Real-time database data directories cannot be directly placed in RustFS.

---

### 8.2 Not Suitable as Replacement for Longhorn

RustFS cannot replace:

PVC
PV
StorageClass
Kubernetes Block Storage
Pod Mount Volumes

If Pod needs:

/var/lib/mysql
/var/lib/postgresql
/data/prometheus
/var/jenkins_home

Better Options:

Longhorn
Ceph RBD
Cloud Disk
Local PV

---

### 8.3 Not Suitable as Replacement for Log Retrieval Systems

RustFS can store log archives.

But not suitable for direct replacement:

Elasticsearch
OpenSearch
Loki
ClickHouse
Log Retrieval Platforms

Differences:

RustFS is responsible for storing objects.
Log platforms are responsible for indexing, querying, analysis, and alerts.

Correct Combination:

Real-time log query: Loki / ELK
Historical compressed archive: RustFS / MinIO / OSS

---

### 8.4 Not Suitable for Direct Production Replacement of MinIO

Not Recommended:

Directly replacing production MinIO.
Directly hosting core business attachments.
Directly as the sole backup storage.
Directly as a large-scale data lake foundation.
Launching without pressure testing.
Launching without recovery drills.
Launching without S3 compatibility testing.
Launching without monitoring and alerts.

Reasons:

RustFS is still a novel solution.
Needs verification of version stability.
Needs verification of functional completeness.
Needs verification of client compatibility.
Needs verification of operations toolchains.
Needs verification of fault recovery.
Needs verification of upgrade paths.

---

## IX. Relationship Between RustFS and MinIO

### 9.1 Commonalities

RustFS and MinIO both belong to:

S3 Compatible Object Storage

Common Features:

Both provide S3 API.
Both support Bucket / Object model.
Both can be accessed via AccessKey / SecretKey.
Both can use mc client.
Both can use AWS CLI.
Both can be accessed via Nginx / LB as a unified entry.
Both are suitable for scenarios like images, attachments, backups, log archives, AI datasets, etc.
Both need attention to HTTPS, permissions, logs, capacity, backups, and fault recovery.

---

### 9.2 Differences

| Comparison Item | MinIO | RustFS |
|---|---|---|
| Implementation Language | Go | Rust |
| Maturity | More mature, more production cases | Novel solution, needs careful verification |
| Ecosystem | More mature documentation, cases, and toolchains | Ecosystem still needs verification |
| Learning Focus | Object storage mainline foundation | Novel object storage comparison and verification |
| Production Recommendation | Can be used as a mature private S3 solution for evaluation | Learn, test, pilot first, then consider production |
| Replacement Risk | Relatively controllable | Needs thorough verification |
| Migration Cost | Depends on the original system | Needs testing of compatibility and data migration |

---

### 9.3 Learning Order Recommendation

Recommended to learn first:

MinIO

Then learn:

RustFS

Reasons:

MinIO is more mature.
MinIO has more documentation and cases.
MinIO can help establish object storage foundation models first.
When learning RustFS, it can be compared with MinIO in deployment, client, permissions, reverse proxy, and operations methods.
Easier to judge the value and boundaries of RustFS.

---

## X. Relationship Between RustFS and Ceph RGW

### 10.1 What is Ceph RGW

Ceph RGW is the object storage gateway of Ceph.

It provides object storage interface based on Ceph RADOS.

Ceph RGW can provide:

S3 API
Swift API
Multi-tenant object storage
Integration with Ceph Pool / CRUSH / OSD

---

### 10.2 Comparison Between RustFS and Ceph RGW /think

| Comparison Item | RustFS | Ceph RGW |
|---|---|---|
| System Position | Independent S3 object storage | Ceph unified storage system object gateway |
| Underlying Complexity | Relatively lightweight | More complex |
| Learning Focus | S3, deployment, client, permissions, entry | RADOS, Pool, PG, OSD, RGW |
| Suitable Environment | Private object storage validation, small/medium scenarios | Existing Ceph cluster or large-scale unified storage |
| Operation Difficulty | Medium, requires validation | High |
| Feature Maturity | Needs version evaluation | Mature but complex |

One sentence:

    Already have a Ceph unified storage system, can evaluate Ceph RGW.
    Want lightweight S3 object storage, can evaluate MinIO or RustFS.

---

## Eleven. RustFS and Longhorn Relationship

### 11.1 What Longhorn Solves

Longhorn solves:

    Kubernetes Pod needs persistent block storage.

Example:

    MySQL data disk
    PostgreSQL data disk
    Redis data directory
    Prometheus data directory
    Jenkins home

Longhorn usage:

    PVC
    PV
    StorageClass
    CSI

---

### 11.2 What RustFS Solves

RustFS solves:

    Application needs for object upload/download.

Example:

    User avatar
    Attachments
    Backup packages
    Log archiving
    AI model files
    Large file downloads

RustFS usage:

    S3 Endpoint
    AccessKey
    SecretKey
    Bucket
    Object Key
    S3 SDK
    mc
    AWS CLI

---

### 11.3 Selection Mnemonic

    Need a single disk: Longhorn
    Need shared directory: NFS / CephFS
    Need object interface: RustFS / MinIO / S3
    Need unified large-scale storage foundation: Ceph
    Need cloud vendor-managed object storage: OSS / COS / S3

---

## Twelve. RustFS Basic Access Model

### 12.1 What Client Access Requires

A client accessing RustFS typically needs:

    Endpoint
    AccessKey
    SecretKey
    Bucket
    Region
    Path-style or Virtual-hosted-style configuration

Example:

    Endpoint: http://10.0.0.51:9000
    AccessKey: rustfsadmin
    SecretKey: rustfsadmin123
    Bucket: app-uploads
    Region: us-east-1

Note:

    This is just a conceptual example.
    Actual AccessKey, SecretKey, ports, and startup parameters will be determined by subsequent deployment practices.
    Do not use weak passwords or default passwords in production.

---

### 12.2 mc Access Model

mc is MinIO Client, also commonly used to access S3-compatible object storage.

Typical workflow:

    mc alias set rustfs http://10.0.0.51:9000 <ACCESS_KEY> <SECRET_KEY>
    mc mb rustfs/app-uploads
    mc cp test.txt rustfs/app-uploads/
    mc ls rustfs/app-uploads
    mc cp rustfs/app-uploads/test.txt ./test-download.txt

Note:

    Specific commands will be executed after successful deployment.
    Whether RustFS fully supports all advanced capabilities of mc needs actual verification.
    Basic Bucket and Object operations are the first validation focus.

---

### 12.3 AWS CLI Access Model

AWS CLI can also access S3-compatible services.

Typical format:

    aws --endpoint-url http://10.0.0.51:9000 s3 ls

Upload:

    aws --endpoint-url http://10.0.0.51:9000 s3 cp test.txt s3://app-uploads/test.txt

Download:

    aws --endpoint-url http://10.0.0.51:9000 s3 cp s3://app-uploads/test.txt ./test-download.txt

Note:

    AWS CLI requires configuration of access_key and secret_key.
    Need to verify RustFS compatibility with AWS CLI.
    If signature errors occur, check Region, Endpoint, Path-style, time synchronization, and keys.

---

## Thirteen. RustFS Basic Experiment Plan

This article does not directly deploy, but first plans for subsequent experiments.

### 13.1 Single-node Experiment

Node:

    10.0.0.51 rustfs-node01

Data Directory:

    /data/rustfs

Objective:

    Start RustFS container.
    Expose API port.
    Configure mc alias.
    Create Bucket.
    Upload object.
    Download object.
    Delete object.
    Verify data remains after container restart.

---

### 13.2 Multi-node Experiment

Nodes:

    10.0.0.51 rustfs-node01
    10.0.0.52 rustfs-node02
    10.0.0.53 rustfs-node03
    10.0.0.54 rustfs-node04

Unified Entry:

    10.0.0.56 rustfs-entry

Objective:

    Multi-node start RustFS.
    Each node prepares data directory.
    Unified entry access.
    Internal HTTP.
    External HTTPS.
    Verify unified entry with mc.
    Simulate node failure.
    Check logs and capacity.
    Evaluate recovery capability.

---

### 13.3 Client Experiment

Client Node:

    10.0.0.55 rustfs-client

Installed Tools:

    mc
    aws cli
    curl
    jq

Objective:

    Configure RustFS alias.
    Create Bucket.
    Upload small file.
    Upload large file.
    Download object.
    Verify object integrity.
    Test wrong key.
    Test non-existent Bucket.
    Test HTTPS entry.

---

## Fourteen. RustFS Production Evaluation Checklist

Must verify before RustFS production deployment:

### 14.1 S3 Compatibility

Need to verify: /think

# Bucket Creation  
# Bucket Deletion  
# Object Upload  
# Object Download  
# Object Deletion  
# ListObjects  
# Multipart Upload  
# Presigned URL  
# SDK Compatibility  
# AWS CLI Compatibility  
# mc Compatibility  
# Path-style Access  
# Virtual-hosted-style Access  
# Region Behavior  
# Signature V4  

---

### 14.2 Performance Capabilities  

Need to verify:  

    Small Object Upload Performance  
    Small Object Download Performance  
    Large Object Upload Performance  
    Large Object Download Performance  
    Concurrent Uploads  
    Concurrent Downloads  
    Multipart Upload  
    Latency  
    Throughput  
    CPU Usage  
    Memory Usage  
    Disk I/O  
    Network Bandwidth  

---

### 14.3 Reliability Capabilities  

Need to verify:  

    Whether data remains normal after container restart.  
    Whether service recovers after node restart.  
    Impact of single node failure.  
    Behavior when disk capacity is insufficient.  
    Behavior when data directory permissions are abnormal.  
    Behavior when reverse proxy fails.  
    Behavior after client upload interruption.  
    Node recovery capability in multi-node mode.  
    Version upgrade process.  

---

### 14.4 Security Capabilities  

Need to verify:  

    HTTPS Access.  
    AccessKey / SecretKey Management.  
    Bucket Permissions.  
    Read-only Permissions.  
    Read-write Permissions.  
    Delete Permissions.  
    Root Account Protection.  
    Credential Disable and Rotation after leakage.  
    Reverse Proxy Source Limitation.  
    Upload Size Limitation.  
    Access Logs.  
    Audit Capabilities.  

---

### 14.5 Operations Capabilities  

Need to verify:  

    Log Location.  
    Log Format.  
    Container Status.  
    Health Check Interface.  
    Metrics Interface.  
    Prometheus Integration.  
    Capacity Statistics.  
    Bucket Statistics.  
    Object Statistics.  
    Alerting Capabilities.  
    Backup and Migration.  
    mc mirror Compatibility.  
    Fault Recovery Process.  

---

## FifteenI don't know.RustFS Security Baseline  

RustFS subsequent experiments and production design must follow:  

    Do not use weak passwords.  
    Do not use default keys for long-term operation.  
    Root account is only for initialization.  
    Business uses independent AccessKey.  
    AccessKey / SecretKey not written to public Git.  
    External access must use HTTPS.  
    Internal HTTP only allows trusted networks.  
    Limit source for management entry.  
    Nginx configuration upload size limitation.  
    Nginx configuration reasonable timeout.  
    Bucket permissions minimized.  
    Delete permissions granted cautiously.  
    Certificate expiration needs monitoring.  
    Access logs need retention.  
    Key leakage must be rotatable.  

---

## SixteenI don't know.RustFS Operations Baseline  

Daily operations need to focus on:  

    Whether RustFS container is running.  
    Whether RustFS port is listening.  
    Whether RustFS API is accessible.  
    Whether Bucket is normal.  
    Whether object upload/download is normal.  
    Data directory capacity.  
    Node disk capacity.  
    Node disk I/O.  
    Node network.  
    Nginx access logs.  
    Nginx error logs.  
    Client 4xx / 5xx.  
    AccessDenied.  
    SignatureDoesNotMatch.  
    Upload failure.  
    Download failure.  
    Certificate status.  
    Backup and migration tasks.  

Common command directions:  

    docker ps  
    docker logs  
    docker inspect  
    ss -lntp  
    curl -I  
    mc alias list  
    mc ls  
    mc stat  
    mc cp  
    aws --endpoint-url ...  
    df -hT  
    du -sh  
    tail -f /var/log/nginx/access.log  
    tail -f /var/log/nginx/error.log  

---

## SeventeenI don't know.Common Error Understanding  

### 17.1 Error: RustFS is compatible with S3, so it can be directly used as a cloud disk  

Error.  

S3 is an object storage interface, not a block device.  

RustFS cannot directly replace cloud disks or PVCs.  

---

### 17.2 Error: RustFS can replace Longhorn  

Error.  

RustFS and Longhorn solve completely different problems.  

RustFS is an object storage.  

Longhorn is a Kubernetes block storage.  

---

### 17.3 Error: Object storage is a file system  

Error.  

Object storage is not a traditional POSIX file system.  

Object Key appears like a directory, but essentially it's an object name prefix.  

---

### 17.4 Error: S3 compatibility equals 100% compatibility with AWS S3  

Error.  

Different S3-compatible systems may have differences in advanced features.  

Must verify:  

    Multipart Upload  
    Presigned URL  
    SDK Behavior  
    Permission Policies  
    Region  
    Signature Method  
    Large Objects  
    Small Objects  
    Concurrency  
    Error Codes  

---

### 17.5 Error: Being able to upload and download means it's production-ready  

Error.  

Production also needs to verify:  

    Monitoring  
    Logs  
    Backup  
    Recovery  
    Permissions  
    HTTPS  
    Failure Drill  
    Capacity Alert  
    Version Upgrade  
    Data Migration  
    Performance Stress Testing  

---

## EighteenI don't know.Interview Answer Approach  

If asked in an interview:  

    What is RustFS? What scenarios is it suitable for?  

You can answer:

# RustFS is an S3-compatible object storage system written in Rust, positioned similarly to MinIO, primarily providing object storage capabilities such as Bucket, Object, S3 API, AccessKey, and SecretKey. It is suitable for storing unstructured data such as images, attachments, backup packages, log archives, artifact packages, AI datasets, and model files.

It differs from Longhorn. Longhorn is a Kubernetes block storage system, mainly providing persistent data disks to Pods through PV/PVC; RustFS is an object storage system, where applications upload and download objects via HTTP/HTTPS S3 API, and cannot be directly used as a database data directory or PVC block storage.

It is closer to MinIO, both being S3-compatible object storage systems. MinIO has greater maturity and more production cases, while RustFS is a new solution with advantages in Rust implementation, S3 compatibility, and Apache 2.0 license, but production use requires thorough validation of version stability, S3 API compatibility, Multipart Upload, SDK compatibility, permission system, logging, monitoring, node failure recovery, backup migration, and upgrade paths.

My learning and verification approach is: first deploy using Docker single-node to verify service startup, Bucket creation, object upload/download, and data persistence; then deploy multi-node Docker or VM cluster to verify nodes, disks, unified entry points, and fault recovery; finally provide external HTTPS entry via Nginx, and use HTTP for internal node communication within a trusted network. Pre-production testing requires load testing, recovery drills, permission governance, and monitoring alerts.

---

## 19. Summary of This Section

This article completes the foundational learning of RustFS:

1. RustFS is an S3-compatible object storage system written in Rust.
2. RustFS belongs to object storage, not block storage.
3. RustFS does not replace Longhorn.
4. RustFS does not replace database data disks.
5. RustFS's core model is Bucket and Object.
6. S3 Endpoint is the entry point for clients to access object storage.
7. AccessKey / SecretKey are credentials for object storage access.
8. Bucket is the top-level container for object storage.
9. Object is the data entity in object storage.
10. Prefix appears like a directory but is essentially an Object Key prefix.
11. RustFS is suitable for images, attachments, backup packages, log archives, artifact packages, and AI data.
12. RustFS is unsuitable for direct use as MySQL, PostgreSQL, Redis data directories.
13. RustFS is closest to MinIO, suitable for comparative learning.
14. RustFS and Ceph RGW both provide object interfaces, but differ in underlying complexity.
15. RustFS must validate S3 compatibility before production use.
16. RustFS must validate performance, reliability, security, and operations capabilities before production use.
17. "Being able to upload/download" cannot be directly equated to "production readiness".
18. Subsequent learning will focus on deployment patterns, then proceed to single-node Docker deployment practice.

---

## 20. References

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS GitHub:

    https://github.com/rustfs/rustfs

RustFS Docker Hub:

    https://hub.docker.com/r/rustfs/rustfs

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

AWS S3 User Guide:

    https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html

MinIO official documentation:

    https://min.io/docs/minio/linux/index.html

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

AWS CLI S3 documentation:

    https://docs.aws.amazon.com/cli/latest/reference/s3/

Nginx official documentation:

    https://nginx.org/en/docs/

Docker official documentation:

    https://docs.docker.com/