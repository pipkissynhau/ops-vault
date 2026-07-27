# Comparison between RustFS and MinIO: Architecture, Deployment, Ecosystem, and Operations Differences

Recommended Path: 05-Storage/04-RustFS/05-Comparison between RustFS and MinIO: Architecture, Deployment, Ecosystem, and Operations Differences.md

Tags: #RustFS #MinIO #Object Storage #S3 #Architecture Comparison #Deployment Comparison #Operations Comparison #Compatibility Verification #mc #AWSCLI #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the fifth in the RustFS series, focusing on comparing RustFS and MinIO in terms of architecture, deployment, ecosystem, operations, security, and suitability for production use.

Previous articles include:

    01-RustFS Basics: S3-Compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-Node and Cluster Modes
    03-RustFS Single-Node Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multiple Nodes, Multiple Disks, and Access Points

The MinIO series includes:

    MinIO Single-Node Docker Deployment
    MinIO Distributed Docker/VM Cluster Deployment
    MinIO mc Management
    MinIO Bucket/Object Operations
    MinIO Reverse Proxy Unified Entrance
    Internal HTTP and External HTTPS
    Permissions, Security, Backup Migration, and Summary

This article addresses the following key points:

    What types of storage systems are RustFS and MinIO?
    Why are both considered S3-compatible object stores?
    What similarities exist in their architectural approaches?
    How do their deployment methods compare?
    How can we examine their images, versions, ports, and data directories?
    How do their client access methods differ?
    How do their permission models, security measures, and HTTPS configurations compare?
    How can we assess the maturity of their ecosystems?
    How should one choose between them for production use?
    How can we use the same mc commands to test both MinIO and RustFS?
    How can we conduct basic S3 compatibility tests?
    Why shouldn't we simply replace MinIO with RustFS just because it is newer or claims higher performance?

This article emphasizes:

    Both RustFS and MinIO are S3-compatible object stores.
    MinIO represents a more mature baseline for object storage practices.
    RustFS is a new object storage solution worth studying and testing.
    Any decision to replace MinIO with RustFS in production should be based on compatibility, stability, fault recovery, performance testing, and operational capabilities, rather than marketing claims or single experimental results.

---

## II. Learning Objectives

After reading this article, you should be able to:

1. Understand the common positioning of RustFS and MinIO.
2. Recognize that both are object stores, not block storage solutions.
3. Comprehend how both use the S3 API for external services.
4. Identify the consistency of concepts such as Bucket, Object, AccessKey, SecretKey, and Endpoint in both systems.
5. Understand the similarities in their architectural philosophies.
6. Analyze the differences in their maturity levels, ecosystems, and real-world applications.
7. Recognize the similarities in their deployment methods.
8. Be able to connect to both MinIO and RustFS using mc commands.
9. Create identical Buckets on both systems.
10. Upload the same set of objects to both systems.
11. Download objects from both systems and verify their integrity using sha256sum.
12. Use AWS CLI to test basic S3 compatibility.
13. Compare their logs, ports, data directories, and reverse proxy configurations.
14. Design a verification process for migrating data from MinIO to RustFS.
15. Identify the criteria for determining whether RustFS is suitable as a replacement for MinIO.
16. Develop a methodology for selecting object storage solutions.

---

## III. Key Conclusions

### 3.1 RustFS and MinIO Are Both S3-Compatible Object Stores

Similarities:

    Both are S3-compatible object stores.
    Both use the Bucket/Object model as their core structure.
    Both provide APIs via HTTP/HTTPS.
    Both support authentication using AccessKey/SecretKey.
    Both can be accessed using tools like mc, AWS CLI, and S3 SDKs.
    Both are suitable for storing non-structured data such as images, attachments, backups, logs, and AI datasets.
    Neither is designed to serve as a real-time database directory.
    Neither replaces block storage solutions like Longhorn, Ceph RBD, or cloud disks.

In summary:

    If an application requires object upload and download capabilities, RustFS or MinIO could be considered.
    If a Pod needs a persistent data disk, options like Longhorn, Ceph RBD, or cloud disks are more appropriate.
    For multi-node shared directories, NFS, CephFS, or NAS would be better choices.

---

### 3### 10.1 mc

mc is the official client for MinIO. It can be used to access not only MinIO but also other S3-compatible object storage services.

Common features:

- `alias set`: Allows setting aliases for access addresses.
- `mb`: Copies data in megabytes.
- `rb`: Copies data in bytes.
- `ls`: Lists objects.
- `cp`: Copies objects.
- `mirror`: Creates a mirror of an object or directory.
- `rm`: Removes objects.
- `stat`: Displays object statistics.
- `anonymous`: Allows anonymous access.
- `admin`: Provides administrative functions.

Example usage for MinIO:

```bash
mc alias set minio http://10.0.0.41:9000 <ACCESS_KEY> <SECRET_KEY>
```

Example usage for RustFS:

```bash
mc alias set rustfs http://10.0.0.51:9000 <ACCESS_KEY> <SECRET_KEY>
```

Unified access:

```bash
mc alias set minio https://s3.minio.local <ACCESS_KEY> <SECRET_KEY>
```    sha256sum hello.txt data/user.json file-100m.bin >> summary.txt

---

### 13.2 上传对象到 MinIO

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-lab/s3-compare-demo hello.txt app.conf

---

### 13.3 上传对象到 RustFS

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-lab/s3-compare-demo hello.txt app.conf

---

### 13.4 查看上传后的对象

MinIO:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-lab/s3-compare-demo

RustFS:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-lab/s3-compare-demo

---

## 十四、实操四：下载相同对象

### 14.1 下载对象 from MinIO

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-lab/s3-compare-demo hello.txt app.conf

---

### 14.2 下载对象 from RustFS

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-lab/s3-compare-demo hello.txt app.conf

---

### 14.3 查看下载后的对象

MinIO:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-lab/s3-compare-demo

RustFS:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-lab/s3-compare-demo

---

## 十五、实操五：验证对象内容

### 15.1 验证 MinIO 对象内容

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-lab/s3-compare-demo hello.txt | grep "hello"

### 15.2 验证 RustFS 对象内容

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-lab/s3-compare-demo hello.txt | grep "hello"

---

## 十六、实操六：验证校验值

### 16.1 验证 MinIO 校验值

执行：

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/```bash
sha256sum hello.txt app.conf logs/app.log data/user.json file-100m.bin > sha256-before.txt
cat sha256-before.txt

---

### 13.2 Upload to MinIO

Run the following command:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-compare-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/ minio-lab/s3-compare-demo/

Check the results:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive minio-lab/s3-compare-demo

---

### 13.3 Upload to RustFS

Run the following command:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-compare-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/ rustfs-lab/s3-compare-demo/

Check the results:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-lab/s3-compare-demo

---

### 13.4 Compare Object Lists

MinIO:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive minio-lab/s3-compare-demo > /tmp/minio-object-list.txt

RustFS:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-lab/s3-compare-demo > /tmp/rustfs-object-list.txt

Check the results:

    cat /tmp/minio-object-list.txt
    cat /tmp/rustfs-object-list.txt

---

## Chapter Fourteen: Practical Exercise Four: Downloading Objects and Verifying Them

### 14.1 Download from MinIO

Create a directory:

    mkdir -p /tmp/s3-compare-download/minio

Download the files:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-compare-download/minio:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive minio-lab/s3-compare-demo/ /download/

Check the downloaded files:

    find /tmp/s3-compare-download/minio -type f -print

---

### 14.2 Download from RustFS

Create a directory:

    mkdir -p /tmp/s3-compare-download/rustfs

Download the files:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      -v /tmp/s3-compare-download/rustfs:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive rustfs-lab/s3-compare-demo/ /download/

Check the downloaded files:

    find /tmp/s3-compare-download/rustfs -type f -print

---

### 14.3 Verify MinIO Downloaded Files

Run the following commands:

    sha256sum /tmp/s3-compare-download/minio/hello.txt
    sha256sum /tmp/s3-compare-download/minio/app.conf
   ```markdown
aws --endpoint-url http://10.0.0.51:9000 s3 ls s3://s3-compare-demo/ --recursive

---

### 15.4 AWS CLI Upload Comparison

Create a file:

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

    The current S3 compatibility of the RustFS version.
    Whether the endpoint is correct.
    If the regions are consistent.
    Whether the AccessKey/SecretKey are correct.
    If the system time is synchronized.
    Whether path-style is required.
    If there are any SignatureDoesNotMatch errors.
    If AccessDenied occurs.
    If 501 Not Implemented is received.

If RustFS is available but MinIO is not, focus on checking:

    The status of the MinIO service.
    The MinIO account credentials.
    Whether the bucket exists.
    Whether there is confusion between the MinIO Console and API.
    If Nginx forwarding is correct.

---

## Section Sixteen: Permission and Security Comparison

### 16.1 Common Security Principles

Both MinIO and RustFS should adhere to the following principles:

    Avoid using weak passwords.
    Do not use default keys for long-term operation.
    Use Root/Admin accounts only for initialization purposes.
    Assign separate AccessKeys for different business scenarios.
    Do not commit AccessKey/SecretKey pairs to Git.
    Use different buckets for different services.
    Implement least privilege access.
    Be cautious when granting deletion permissions.
    Ensure all external accesses use HTTPS.
    Keep management interfaces private from the public network.
    Retain access logs.
    Enable key deactivation and rotation in case of leaks.

---

### 16.2 MinIO's Security Ecosystem

MinIO typically includes the following security features:

    User management.
    Group management.
    Policy management.
    Bucket policies.
    AccessKey/SecretKey management.
    Console management.
    mc admin tools.
    TLS support.
    KMS and encryption capabilities.
    Audit logging.
    Integration with external identity systems.

In production environments, MinIO has more mature security governance practices.

---

### 16.3 Key Security Verification Points for RustFS

For RustFS, it is essential to verify the following:

    Whether user management meets requirements.
    If policies are compatible with business needs.
    Whether bucket permissions are appropriate.
    Support for separate read, write, and deletion permissions.
    Capability for key rotation.
    Availability of audit logging.
    Stability of HTTPS reverse proxies.
    Control over the Console management interface.
    Consistency of permission behaviors with existing SDKs.

Conclusion:

    When considering whether RustFS can replace MinIO, the permission system is a critical factor to evaluate.

---

## Section Seventeen: Log and Operations Comparison

### 17.1 MinIO Logs

Common log collection methods for MinIO include:

    docker logs minio
    mc admin trace
    mc admin info
    mc admin heal
    mc admin user
    mc admin policy
    Console monitoring.
    Nginx access.log and error.log.

MinIO has a more established operations toolchain.

---

### 17.2 RustFS Logs

Currently, common log collection methods for RustFS include:

    docker logs rustfs-single
    docker logs rustfs-cluster
    Console monitoring.
    Nginx access.log and error.log.
    mc ls / cp / stat commands.
    AWS CLI output.

Before deploying RustFS in production, it is crucial to confirm the following:

    Availability of comprehensive admin tools.
    Provision of metrics exposure.
    Support for Prometheus integration.
    Existence of audit logging capabilities.
    Support for fault diagnosis commands.
    Capability to monitor cluster status.
    Availability of user and policy management commands.

---

### 17.3 Routine Inspection Comparison

For MinIO, routine inspections include:

    docker ps | grep minio
    docker logs minioThis is a basic observation, not an official stress test.
For official stress testing, professional tools should be used, with fixed variables, repeated tests, and system metrics recorded.

---

## Chapter 19: Compatibility Comparison Checklist

### 19.1 Basic S3 Operations

The following must be verified:

    CreateBucket
    DeleteBucket
    PutObject
    GetObject
    DeleteObject
    ListObjects
    HeadObject
    CopyObject

---

### 19.2 Advanced S3 Features

It is recommended to verify these before production:

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
    Large Object Resume Upload
    Batch Deletion
    Cross-Bucket Copy

Note:

    Different object storage systems support different levels of advanced S3 features.
    Only verify what your business requires.
    Even if a feature is not used, record it as "not within the current business scope".

---

### 19.3 SDK Compatibility Testing

It is recommended to verify at least the following:

    Python boto3
    Java AWS SDK
    Go AWS SDK
    Node.js AWS SDK
    The SDK actually used by the application

Testing items include:

    Initializing the client.
    Creating a Bucket.
    Uploading an object.
    Downloading an object.
    Listing objects.
    Deleting an object.
    Generating a presigned URL.
    Multipart Upload.
    Error handling.

---

## Chapter 20: Migration Strategy Comparison

### 20.1 Principles for Migrating from MinIO to RustFS

It is not recommended to replace it directly in place.

Recommended process:

    1. Set up a RustFS test cluster.
    2. Verify compatibility using the same version of business SDKs.
    3. Create the same Buckets.
    4. Use mc mirror for data synchronization.
    5. Compare the number and size of objects.
    6. Download samples to verify sha256sums.
    7. Verify that applications can read and write to RustFS.
    8. Verify permission policies.
    9. Verify HTTPS endpoints.
    10. Verify large and small objects.
    11. Conduct stress tests.
    12. Perform node failure drills.
    13. Prepare rollback plans.
    14. Gradually switch non-core services.
    15. Expand the scope after observing stability.

---

### 20.2 Example of Using mc mirror for Migration

Migrating from MinIO to RustFS:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mirror --overwrite minio-lab/s3-compare-demo rustfs-lab/s3-compare-demo

Migrating back from RustFS to MinIO:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mirror --overwrite rustfs-lab/s3-compare-demo minio-lab/s3-compare-demo

Note:

    Mirror is one of the migration methods.
    Production migrations need to consider incremental synchronization, write freezing, data verification, permission migration, and rollback.
    It is not recommended to only synchronize object data while ignoring accounts, policies, Bucket configurations, and application settings.

---

### 20.3 Migration Verification

Verification aspects include:

    Number of Buckets
    Number of objects
    Object size
    Object metadata
    Permission policies
    Application read/write performance
    Download verification
    Random sampling
    Full verification
    Log errors
    Business access error rate

Basic commands:

    mc ls --recursive minio-lab/s3-compare-demo | wc -l
    mc ls --recursive rustfs-lab/s3-compare-demo | wc -l

For Docker versions:

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive minio-lab/s3-compare-demo | wc -l

    docker run --rm \
      -v /data/object-storage-compare/mc-config:/root/.mc \
Backup and Recovery Process

Do not:

- Directly copy the internal data directory.
- Allow two services to write to the same directory simultaneously.
- Manually modify the internal metadata.

---

### 23.5 Error: Object storage migration only requires copying files

Incorrect.

Other considerations include:

- Bucket
- Object
- Metadata
- Policy
- User
- Group
- AccessKey
- SecretKey
- Lifecycle
- Versioning
- Application Config
- DNS
- HTTPS
- Rollback

---

## 24. Experiment Cleanup

### 24.1 Delete Comparison Buckets and Objects

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

### 24.2 Delete Buckets

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

High-Risk Reminder:

- Deleting a production Bucket must undergo approval.
- Confirm the business ownership before deletion.
- Ensure backups are in place before deletion.
- Verify if there is a recovery plan before deletion.

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

If it will not be used later:

    rm -rf /data/object-storage-compare/mc-config

If migration and comparison experiments are still planned, it is recommended to keep it.

---

## 25. Completion Standards for This Article

After completing this article, you should have achieved at least the following:

| Item | Standard |
|---|---|
| Theoretical Understanding | Understand that both MinIO and RustFS are S3-compatible object storages. |
| Type Differences | Recognize that neither replaces Longhorn. |
| Version Awareness | Know that MinIO is a mature baseline, while RustFS is a new validation option. |
| mc Configuration | Set up both minio-lab and rustfs-lab. |
| Bucket Creation | Create s3-compare-demo on both platforms. |
| Object Upload | Upload the same batch of objects to MinIO and RustFS. |
| Object Download | Objects can be downloaded from both platforms. |
| Data Verification | Ensure sha256sums match. |
| AWS CLI Testing | Basic S3 commands like ls and cp should work on both. |
| Operational Comparison | Compare logs, ports, directories, and interfaces. |
| Migration Understanding | Comprehend the mc mirror migration approach. |
| Selection Ability | Be able to explain when to use MinIO and when to evaluate RustFS. |
| Risk Awareness | Recognize that a thorough PoC is necessary before replacing MinIO with RustFS.

---

## 26. Interview Answer Guidelines

If asked in an interview:

- What are the differences between RustFS and MinIO? How would you choose one?

You can answer as follows:

- Both RustFS and MinIO are S3-compatible object storages. Their core models include Buckets and Objects, and they both provide services through HTTP/HTTPS S3 APIs. They can also be accessed using tools like mc, AWS CLI, and various S3 SDKs. Both are suitable for storing non-structured data such as images, attachments, backup fileshttps://docs.rustfs.com/integration/tls-configuration/

RustFS GitHub page:

https://github.com/rustfs/rustfs

RustFS Docker Hub page:

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