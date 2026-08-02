# RustFS Basics: S3-Compatible Object Storage and Use Cases

Recommended Path: 05-Storage/04-RustFS/01-RustFS Basics: S3-Compatible Object Storage and Use Cases.md

Tags: #RustFS #ObjectStorage #S3 #Bucket #Object #MinIO Comparison #Docker #mc #AWSCLI #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the first in the RustFS series, focusing on understanding RustFS’s fundamental role, its compatibility with the S3 object storage model, and its applicable scenarios.

This document does not delve into deployment practices; those will be covered in subsequent sections 03 and 04.

The main objectives of this article are to:

- Explain what RustFS is.
- Clarify the relationship between RustFS and S3 object storage.
- Define concepts such as Bucket, Object, AccessKey, SecretKey, and Endpoint.
- Discuss the differences between RustFS and MinIO.
- Compare RustFS with other solutions like Longhorn, Ceph, and NFS.
- Identify suitable and unsuitable use cases for RustFS.
- Explain why RustFS is currently more suitable for learning, testing, comparison, and pilot evaluations.
- Outline the capabilities that must be verified before using RustFS in production.

A key principle emphasized in this article is:

- RustFS can serve as a learning and testing tool for S3-compatible object storage solutions. However, it should not be directly used to replace mature production storage systems like MinIO or Ceph RGW just because it is compatible with S3.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand RustFS’s basic purpose.
2. Recognize that RustFS is an S3-compatible object storage system, not a block storage solution.
3. Comprehend the fundamental concepts of Bucket, Object, Prefix, and Endpoint.
4. Know the function of AccessKey and SecretKey.
5. Distinguish between RustFS and MinIO.
6. Understand the differences between RustFS and Longhorn/Ceph/RGW.
7. Identify data types suitable for storage using RustFS.
8. Recognize data types that are not well-suited for RustFS.
9. Clearly explain the distinctions between object storage, file systems, and block storage.
10. Determine whether a business scenario is appropriate for adopting RustFS.
11. List the capabilities that must be verified before deploying RustFS in production.
12. Lay the foundation for subsequent single-machine Docker deployment, cluster setup, client access, and reverse proxy configurations.

---

## III. What is RustFS

RustFS is an object storage system written in Rust that is compatible with S3.

In simple terms:

- RustFS is a private object storage service that provides an interface similar to AWS S3, allowing applications or clients to interact with it through the S3 API.

Common methods of accessing RustFS include:

- Using the mc tool
- Leveraging AWS CLI
- Employing the S3 SDK
- Through application-specific S3 clients
- For backup and archiving purposes
- As a backend for AI/big data systems

Key terms related to RustFS include:

- Rust
- S3 Compatible
- Object Storage
- Bucket
- Object
- AccessKey
- SecretKey
- Endpoint
- Distributed Object Storage
- MinIO Alternative
- Apache 2.0

---

## IV. What RustFS Is Not

RustFS is not a block storage system. It is not:

- Longhorn
- Ceph RBD
- Cloud Block Storages
- Local Disks
- Kubernetes PVC Block Volumes

Nor is it a file system storage solution. It is not:

- NFS
- SMB
- CephFS
- NAS Shared Directories

Additionally, RustFS is not a Kubernetes CSI storage plugin. It is not used for:

- PV/PVC dynamic provisioning
- StorageClass backend services
- Pod mounting as disks

RustFS is specifically designed as an object storage system, where applications can:

- Upload objects via the S3 API
- Download objects via the S3 API
- Manage Buckets and Objects through the S3 API

---

## V. Basic Concepts of Object Storage

### 5.1 What is Object Storage

Object storage is a method of storing unstructured data. It does not use traditional directories or block devices but stores individual objects instead.

An object typically includes:

- The actual data
- An object name
- Metadata
- The Bucket it belongs to
- Access permissions
- Version information

Common types of objects include:

- Images
- Videos
- Documents
- Attachments
- Backup files
- Log archives
- Model files
- Data sets
- Product packages
- Static resources
- Large file download packages

---

### 5.2 What is S3

Originally developed by AWS, the S3### 6.4 Comparison of the Three

| Type          | Representative      | Access Method       | Typical Uses                          |
|---------------|-------------------|--------------------|---------------------------------------------|
| Object Storage | RustFS / MinIO / S3     | HTTP / HTTPS / S3 API    | Images, attachments, backups, archiving        |
| Block Storage  | Longhorn / Ceph RBD      | Mounted as disk or PVC   | Databases, application data disks            |
| File Storage   | NFS / CephFS         | Shared directory mounting | Multi-node shared files                      |

In summary:

    For uploading objects, use RustFS / MinIO.
    For persistent block storage in Pods, choose Longhorn.
    For multi-node shared directories, opt for NFS / CephFS.

---

## VII. Scenarios Suitable for RustFS

### 7.1 Image and Attachment Storage

Suitable for:

    User avatars
    Product images
    Resume attachments
    PDF files
    Word documents
    Spreadsheet files
    File uploads

Examples:

    app-uploads/avatar/user-1001.png
    app-uploads/resume/user-1001/resume.pdf
    app-uploads/document/2026/04/report.docx

Business Implementation:

    The application backend uses the S3 SDK to upload files to RustFS.
    The database only stores object paths, Bucket names, and Object Keys.
    Users can access files via download links or pre-signed URLs generated by the backend.

---

### 7.2 Backup Package Storage

Suitable for:

    MySQL backups
    PostgreSQL backups
    GitLab backups
    Jenkins backups
    Configuration backups
    K8s YAML backups
    Longhorn Backup Target verification

Examples:

    backups/mysql/full-2026-04-28.sql.gz
    backups/postgresql/base-2026-04-28.tar.gz
    backups/gitlab/gitlab-backup-2026-04-28.tar

Note:

    When RustFS is used as a backup target itself, its own data protection and recovery capabilities must be considered.
    Do not place the sole backup and production original data in the same failure domain.

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

Object storage is suitable for storing archived logs but cannot directly replace query systems like Elasticsearch / Loki.

Correct Understanding:

    RustFS is used to store archived logs.
    Elasticsearch / Loki are responsible for retrieval and querying.
    They serve different purposes.

---

### 7.4 Product and Release Packages

Suitable for:

    Application build artifacts
    Binary packages
    Offline installation packages
    Image export tar packages
    Helm Chart archiving
    Release packages

Examples:

    artifacts/devatlas/devatlas-api-v1.0.0.tar.gz
    artifacts/scripts/deploy-v20260428.tar.gz
    artifacts/images/nginx-1.25.tar

Note:

    For container images, Harbor should be preferred.
    RustFS is more suitable for storing ordinary file artifacts and cannot replace image repositories.

---

### 7.5 AI Data and Model Files

Suitable for:

    Datasets
    Model files
    Embedding files
    Intermediate training results
    Large file archiving

Examples:

    ai-datasets/logs/2026/nginx-sample.tar.gz
    ai-models/model-v1/model.bin
    ai-output/batch-001/result.jsonl

Note:

    AI-related objects can be quite large.
    It is essential to verify the capabilities of RustFS for uploading, downloading, multipart uploads, resuming downloads, concurrent access, and permission control of such large files.

---

### 7.6 Private S3 Compatibility Environment Verification

RustFS is also suitable for:

    Learning the S3 API
    Verifying client compatibility
    Comparing it with MinIO
    Testing application object storage integration
    Testing the S3 SDK
    Checking if backup tools support the S3 Endpoint
    Testing reverse proxies and HTTPS

---

## VIII. Scenarios Unsuitable for RustFS

### 8.1 Not Suitable as a Direct Database Data Directory

Incorrect Practices:

    Storing the MySQL datadir in RustFS
    Storing the PostgreSQL data in RustFS
    Writing Redis AOF files directly to RustFS

Reasons:

    Databases require local block devices or file system semantics.
    RustFS is an object storage system, not a POSIX file## Chapter Eleven: The Relationship Between RustFS and Longhorn

### 11.1 What Does Longhorn Solve?

Longhorn addresses the issue of persistent block storage requirements for Kubernetes Pods. For example:

- MySQL data disks
- PostgreSQL data disks
- Redis data directories
- Prometheus data directories
- Jenkins home directories

The ways to use Longhorn include:

- PVC
- PV
- StorageClass
- CSI

---

### 11.2 What Does RustFS Solve?

RustFS addresses the need for object upload and download capabilities in applications. For example:

- User avatars
- Attachments
- Backup packages
- Log archives
- AI model files
- Large file downloads

The ways to use RustFS include:

- S3 Endpoint
- AccessKey
- SecretKey
- Bucket
- Object Key
- S3 SDK
- mc
- AWS CLI

---

### 11.3 Selection Guidelines

- If you need a single disk: Longhorn.
- If you need shared directories: NFS / CephFS.
- If you need object interfaces: RustFS / MinIO / S3.
- If you need a unified large-scale storage foundation: Ceph.
- If you need cloud vendors to manage object storage: OSS / COS / S3.

---

## Chapter Twelve: RustFS Basic Access Models

### 12.1 What Does a Client Need to Access RustFS?

A client typically needs the following to access RustFS:

- Endpoint
- AccessKey
- SecretKey
- Bucket
- Region
- Path-style or Virtual-hosted-style configuration

Example:

- Endpoint: http://10.0.0.51:9000
- AccessKey: rustfsadmin
- SecretKey: rustfsadmin123
- Bucket: app-uploads
- Region: us-east-1

Note:

- These are just conceptual examples.
- Actual AccessKey, SecretKey, ports, and startup parameters should be determined based on subsequent deployment practices.
- Never use weak or default passwords in production.

---

### 12.2 The mc Access Model

mc is a MinIO Client and is also commonly used to access S3-compatible object storage.

Typical steps:

- `mc alias set rustfs http://10.0.0.51:9000 <ACCESS_KEY> <SECRET_KEY>`
- `mc mb rustfs/app-uploads`
- `mc cp test.txt rustfs/app-uploads/`
- `mc ls rustfs/app-uploads`
- `mc cp rustfs/app-uploads/test.txt ./test-download.txt`

Note:

- Execute these specific commands after successful deployment.
- Whether RustFS fully supports all advanced mc features needs to be verified in practice.
- Basic Bucket and Object operations should be the focus of initial verification.

---

### 12.3 The AWS CLI Access Model

AWS CLI can also be used to access S3-compatible services.

Typical usage:

- `aws --endpoint-url http://10.0.0.51:9000 s3 ls`
- To upload:
  `aws --endpoint-url http://10.0.0.51:9000 s3 cp test.txt s3://app-uploads/test.txt`
- To download:
  `aws --endpoint-url http://10.0.0.51:9000 s3 cp s3://app-uploads/test.txt ./test-download.txt`

Note:

- AWS CLI requires configuration of access_key and secret_key.
- The compatibility of RustFS with AWS CLI needs to be verified.
- If signature errors occur, check the Region, Endpoint, Path-style, time synchronization, and keys.

---

## Chapter Thirteen: Basic Experimental Planning for RustFS

This document does not involve direct deployment but outlines subsequent experiments.

### 13.1 Single-Machine Experiment

Node:

- 10.0.0.51 rustfs-node01

Data directory:

/data/rustfs

Objectives:

- Start the RustFS container.
- Expose API ports.
- Configure an alias using mc.
- Create a Bucket.
- Upload objects.
- Download objects.
- Delete objects.
- Verify that data remains after restarting the container.

---

### 13.2 Multi-Node Experiment

Nodes:

- 10.0.0.51 rustfs-node01
- 10.0.0.52 rustfs-node02
- 10.0.0.53 rustfs-node03
- 10.0.0.54 rustfs-node04

Unified entry point:

- 10.0.0.56 rustfs-entry

Objectives:

- Start RustFS on multiple nodes.
- Prepare data directories on each node.
- Access through a unified entry point.
- Implement internal HTTP and external HTTPS.
### 17.1 Error: RustFS is compatible with S3, so it can be used directly as a cloud disk.

Error.

S3 is an object storage interface, not a block device.

RustFS cannot directly replace cloud disks or PVCs.

---

### 17.2 Error: RustFS can replace Longhorn.

Error.

RustFS and Longhorn solve completely different problems.

RustFS is for object storage.

Longhorn is for Kubernetes block storage.

---

### 17.3 Error: Object storage is the same as a file system.

Error.

Object storage is not a traditional POSIX file system.

Object Keys may look like directories, but they are essentially prefixes for object names.

---

### 17.4 Error: S3 compatibility means 100% compatibility with AWS S3.

Error.

Different S3-compatible systems may vary in advanced features.

It is essential to verify:

    Multipart Upload
    Presigned URL
    SDK behavior
    Permission policies
    Region
    Signature methods
    Large objects
    Small objects
    Concurrency
    Error codes

---

### 17.5 Error: If it can upload and download, then it can be used in production.

Error.

Production use also requires verifying:

    Monitoring
    Logging
    Backup
    Recovery
    Permissions
    HTTPS
    Fault tolerance testing
    Capacity alerts
    Version upgrades
    Data migration
    Performance testing

---

## Section Eighteen: Interview Answer Guidelines

If asked in an interview:

    What is RustFS? In what scenarios is it suitable?

You can answer as follows:

    RustFS is an object storage system written in Rust that is compatible with S3. It is similar to MinIO in its functionality, providing features such as Buckets, Objects, S3 APIs, Access Keys, and Secret Keys. It is ideal for storing unstructured data such as images, attachments, backup files, log archives, build packages, AI datasets, and model files.
    RustFS is different from Longhorn. Longhorn is a Kubernetes block storage solution that provides persistent data disks for Pods through PV/PVCs, while RustFS is an object storage system where applications use HTTP/HTTPS S3 APIs to upload and download objects. It cannot be used directly as a database directory or a PVC block storage.
    RustFS is somewhat similar to MinIO, as both are S3-compatible object storage solutions. However, MinIO has more maturity and practical use cases, while RustFS is a newer approach with advantages such as being implemented in Rust, being compatible with S3, and using the Apache 2.0 license. Before deploying it in production, it is crucial to thoroughly verify its stability, compatibility with S3 APIs, Multipart Upload functionality, SDK compatibility, permission management, logging, monitoring, fault recovery, backup and migration capabilities, and upgrade paths.
    My approach to learning and verifying RustFS would be to first deploy it on a single Docker machine to test service startup, Bucket creation, object upload/download, and data persistence. Then, I would deploy it in a multi-node Docker or VM cluster to verify node functionality, disk performance, unified access points, and fault recovery mechanisms. Finally, I would set up an external HTTPS interface using Nginx, with internal node communication using HTTP over a trusted network. Before production use, I would also conduct performance testing, failure recovery drills, implement permission management, and set up monitoring and alerts.

---

## Section Nineteen: Summary of This Article

This article provides an overview of RustFS:

1. RustFS is an S3-compatible object storage system written in Rust.
2. It belongs to the category of object storage, not block storage.
3. RustFS does not replace Longhorn or serve as a database data disk.
4. The core components of RustFS are Buckets and Objects.
5. The S3 Endpoint is used by clients to access the object storage system.
6. Access Keys and Secret Keys are required for accessing object storage.
7. A Bucket acts as the top-level container in object storage.
8. An Object represents the actual data stored in object storage.
9. Prefixes may resemble directories but are essentially Object Key prefixes.
10. RustFS is suitable for storing non-structured data such as images, attachments, and backup files.
11. It is not suitable for use directly as a data directory for databases like MySQL, PostgreSQL, or Redis.
12. RustFS is most similar to MinIO and can be used for comparative study.
13. Both RustFS and Ceph RGW provide object storage interfaces, but their underlying complexities differ.
14. Before deploying RustFS in production, it is essential to verify its compatibility with S3.
15. Production readiness also requires testing performance, reliability, security, and operational capabilities.
16. The ability to upload and download data does not equate to it being ready for production use.
17. Next steps will involve