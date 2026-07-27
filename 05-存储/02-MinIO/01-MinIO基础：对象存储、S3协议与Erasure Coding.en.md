# MinIO Basics: Object Storage, S3 Protocol, and Erasure Coding

Recommended Path: 05-Storage/02-MinIO/01-MinIO Basics: Object Storage, S3 Protocol, and Erasure Coding.md

Tags: #MinIO #ObjectStorage #S3 #Bucket #Object #ErasureCoding #Docker #mc #AdvancedSRE #ProductionOps

---

## I. Document Introduction

This article is the first in the MinIO series, focusing on understanding the basic concepts of MinIO and conducting essential hands-on experiments.

Rather than being purely theoretical, this article demonstrates the following concepts through a single-machine Docker MinIO experiment:

- What object storage is
- What a Bucket is
- What an Object is
- What the S3 protocol is
- What AccessKey and SecretKey are
- How to connect to MinIO using the mc client
- How to create a Bucket
- How to upload and download objects
- How to view object information
- How to understand the concept of “directories” in object storage
- What problem Erasure Coding solves
- How MinIO differs from traditional file systems, block storage, and Ceph RGW

The purpose of this article is:

- First, to establish a basic understanding of MinIO's object storage functionality.
- Second, to introduce the concepts of S3, Bucket, Object, and Erasure Coding.
- Later on, we will explore topics such as single-machine setups, multi-disk configurations, multi-node systems, Nginx HTTPS, permissions, monitoring, and backup and migration strategies.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand that MinIO is an object storage system compatible with the S3 API.
2. Distinguish between object storage, block storage, and file storage.
3. Comprehend the relationship between Bucket, Object, and Prefix.
4. Know the role of AccessKey and SecretKey.
5. Launch a single-machine MinIO instance using Docker.
6. Connect to MinIO using the mc client.
7. Create a Bucket.
8. Upload, download, view, and delete objects.
9. Recognize that “directories” in object storage are essentially prefixes of Object Keys.
10. Understand the basic function of Erasure Coding.
11. Identify the differences between single-machine MinIO experiments and production-scale distributed clusters.
12. Lay a foundation for further studies on MinIO deployment, permissions, reverse proxies, monitoring, and data protection.

---

## III. Experimental Environment

### 3.1 Experimental Nodes

This article uses a single-machine Docker experiment.

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | Single-machine MinIO server |
| 10.0.0.45 | minio-client | mc client (optional) |

If you only have one machine available, you can run both the MinIO server and the mc client on the same machine.

---

### 3.2 Operating System

Default system:

    Ubuntu Server 22.04.5 LTS

Alternative systems:

    Rocky Linux 9

The commands in this article are primarily based on Ubuntu.

---

### 3.3 Container Execution Method

This article uses:

    Docker

Not Kubernetes.

Reasons:

- At this stage, the focus is on understanding object storage and S3 operations.
- Docker is more suitable for quick setup, testing, and cleanup.
- Kubernetes and CSI are not essential for the basic understanding of MinIO.
- MinIO is usually accessed through the S3 API by applications and does not require the use of PVCs.

---

### 3.4 Image Versions

This article uses fixed version images synchronized by users to the Alibaba Cloud image repository:

MinIO server:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source images:

    minio/minio:RELEASE.2025-04-22T22-12-26Z
    minio/mc:RELEASE.2025-04-16T18-13-26Z

Reason for using fixed versions:

- To avoid inconsistencies in the Web Console, command parameters, log formats, and experimental results due to changes in the latest version.
- To ensure reproducibility of experiments involving single-machine Docker, distributed Docker, reverse proxies, permissions, Buckets, Policies, and mc mirrors.
- Keeping the mc client and MinIO server at similar versions reduces potential differences in their functionality.

Kubernetes RWO PVC
Dedicated data disk for a single instance

---

### 5.2 File Storage

File storage is similar to shared directories.

Typical examples:

    CephFS
    NFS
    NAS

Usage:

    Multiple machines can mount the same shared directory.
    Files are read and written through specific paths.

Suitable for:

    Shared directories among multiple clients
    Shared upload directories
    Kubernetes RWX PVC
    Sharing configuration files
    Sharing datasets

---

### 5.3 Object Storage

Object storage is accessed through APIs.

Typical examples:

    MinIO
    Ceph RGW
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS

Usage:

    Applications upload and download objects through HTTP/HTTPS APIs.
    There is no need to mount local directories.
    It does not provide traditional POSIX file system semantics.

Suitable for:

    Images
    Attachments
    Videos
    Backup and archiving
    Log archiving
    Product repositories
    Static resources
    AI datasets
    Large amounts of unstructured data

---

### 5.4 Mnemonic for Choosing Among the Three Types of Storage

    Use it like a cloud disk: Block storage
    Use it like a shared directory: File storage
    Use it like OSS/S3: Object storage

Corresponding to the current module:

    Ceph RBD: Block storage
    CephFS: File storage
    MinIO: Object storage
    Ceph RGW: Object storage

---

## VI. Experiment 1: Pulling MinIO and mc Images

### 6.1 Pulling the MinIO Server Image

Execute on minio-node01:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Check:

    docker images | grep minio

---

### 6.2 Pulling the mc Client Image

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Check:

    docker images | grep mc

---

### 6.3 Checking the Images

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq

Expected results:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio
    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc

---

## VII. Experiment 2: Starting a Standalone MinIO Instance

### 7.1 Creating a Data Directory

Execute on minio-node01:

    mkdir -p /data/minio/data

Check:

    ls -ld /data/minio/data

---

### 7.2 Starting the MinIO Container

Execute:

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

Parameter explanations:

| Parameter | Description |
|---|---|
| --name minio-basic | Container name |
| --restart unless-stopped | The container will restart automatically if it exits abnormally |
| -p 9000:9000 | Maps the S3 API port |
| -p 9001:9001 | Maps the Web Console port |
| MINIO_ROOT_USER | MinIO root user |
| MINIO_ROOT_PASSWORD | MinIO root password |
| -v /data/minio/data:/data | Mounts the data directory |
| server /data | Uses /data as the object data directory |
| --console-address ":9001" | The Console listens on port 9001 |

---

### 7.3 Checking the Container Status

    docker ps | grep minio-basic

Check the logs:

    docker logs -f minio-basic

If you see similar information in the logs, it means the installation was successful:

    API:
    Console:
    Documentation:

---

### 7.4 Accessing the Web Console

Access via browser:

    http://10.0.0.41:9001

Login credentials:

    Username: minioadmin
    Password: MinioAdmin@1234echo "hello minio object storage" > /tmp/minio-basic-demo/hello.txt

View:

cat /tmp/minio-basic-demo/hello.txt

---

### 10.2 Upload an Object to a Bucket

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-basic-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp /demo/hello.txt local/app-uploads/hello.txt

---

### 10.3 View an Object

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls local/app-uploads

Expected to see:

hello.txt

---

### 10.4 View Detailed Object Information

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

## Chapter Eleven, Experiment Six: Downloading Objects

### 11.1 Delete the Local Original File

rm -f /tmp/minio-basic-demo/hello.txt

Confirm:

ls -l /tmp/minio-basic-demo/

---

### 11.2 Download an Object from MinIO

Execute:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-basic-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp local/app-uploads/hello.txt /demo/hello-download.txt

View:

cat /tmp/minio-basic-demo/hello-download.txt

Expected:

hello minio object storage

Note:

Both uploading and downloading are completed through the S3 API.
In the future, similar operations in the application will also be done using the S3 SDK.

---

## Chapter Twelve, Experiment Seven: Understanding "Directories" in Object Storage

### 12.1 Upload an Object with a Path

Create a test file:

mkdir -p /tmp/minio-basic-demo/logs

echo "nginx access log demo" > /tmp/minio-basic-demo/logs/access.log

Upload it to an object Key with a prefix:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-basic-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  cp /demo/logs/access.log local/app-uploads/logs/nginx/2026/04/28/access.log

---

### 12.2 View the Bucket

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

### 12.3 Key Concepts

The object Key is:

logs/nginx/2026/04/28/access.log

This is not a traditional multi-level directory in a file system.

It is essentially an object name.

The MinIO Console and mc will display it in a directory-like structure using slashes.

The essence of directories in object storage:

Prefix

This is very important.

When designing object Keys in production, plan ahead considering:

- Business type
- Date
- User ID
- File type
- Environment
- Lifecycle

Examples:

uploads/user/10001/avatar.png
logs/nginx/2026/0      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb local/app-uploads

Production Reminder:

    Deleting a Bucket is a high-risk operation.
    Make sure the objects have been backed up before deletion.
    Verify whether the service still relies on this Bucket before deleting it.
    It is recommended to obtain approval and conduct a review before proceeding with the deletion.

---

## Section 15: Basic Understanding of Erasure Coding

### 15.1 What is Erasure Coding?

Erasure Coding can be understood as a form of error-correction coding.

It divides data into multiple data blocks and parity blocks, which are distributed across multiple disks or nodes.

When some disks or nodes fail, as long as there are enough remaining data blocks and parity blocks, the original data can be restored.

Simply put:

    Data is not simply copied three times.
    Instead, it is divided into smaller chunks.
    Parity data is generated simultaneously.
    Fault tolerance is achieved through these data blocks and parity blocks.

---

### 15.2 Differences between Replicas and Erasure Coding

Replica Mode:

    One copy of the data is replicated multiple times.

For example, with 3 replicas:

    Original data: 1GB
    Total space used: Approximately 3GB

Advantages:

    Simple to implement.
    Easy to read data.
    Facilitates data recovery.

Disadvantages:

    Low space utilization efficiency.

---

Erasure Coding:

    Data is divided into data blocks and parity blocks.

For example:

    4 data blocks + 2 parity blocks

Advantages:

    Higher space utilization efficiency.
    Still provides fault tolerance.

Disadvantages:

    More complex in terms of encoding and decoding.
    More sensitive to disk, network, and node configurations.
    Data block reconstruction is required in case of failures.

---

### 15.3 The Role of Erasure Coding in MinIO

MinIO uses Erasure Coding to protect data.

It helps to handle:

    Single disk failures.
    Multiple disk failures.
    Single node failures.
    Partially unavailable nodes.

However, it is important to note that:

    Erasure Coding is not a backup mechanism.
    It cannot prevent accidental deletions of data.
    It cannot replace cross-cluster backups.
    It cannot replace mc mirror or off-site disaster recovery solutions.

---

### 15.4 Why Single-Center Single-Disk Deployments Are Not Suitable for Production

Single-center single-disk setups are suitable for:

    Learning purposes.
    Development activities.
    Temporary testing.
    Functional verification.

But they are not suitable for production use:

    A single node failure will cause the service to stop.
    A single disk failure may result in data loss.
    The high availability features of Erasure Coding cannot be fully utilized in such setups.
    It is impossible to verify the recovery process in case of node or disk failures.

For production, it is recommended to use a multi-node, multi-disk distributed architecture.

For example:

    4 nodes.
    Each node has multiple data disks.
    Use Nginx/Load Balancers in front.
    Provide external HTTPS services.
    Use internal HTTP for backend operations.
    Monitor using Prometheus.
    Implement mc mirror for backup.

---

## Section 16: Optional Experiment: Understanding Erasure Coding with Single Node and Multiple Directories

### 16.1 Experiment Description

This experiment is only intended to help understand MinIO's multi-disk architecture.

Note:

    Using a single node with multiple directories does not guarantee high production-level availability.
    If multiple directories are stored on the same physical disk, it still cannot protect against disk failures.
    For production use, a multi-node, multi-disk setup is essential.

---

### 16.2 Stop the Single-Node Container

If the `minio-basic` container is currently running, stop it first:

    docker stop minio-basic
    docker rm minio-basic

---

### 16.3 Create Multiple Data Directories

    mkdir -p /data/minio-ec/disk1
    mkdir -p /data/minio-ec/disk2
    mkdir -p /data/minio-ec/disk3
    mkdir -p /data/minio-ec/disk4

    View the directories:

    tree /data/minio-ec

---

### 16.4 Start MinIO with Multiple Directories on a Single Node

Run the following command:

    docker run -d \
      --name minio-ec-basic \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@12345### 17.2 Viewing Logs

    docker logs minio-basic

Or:

    docker logs minio-ec-basic

Common Issues:

- Password does not meet requirements
- Data directory permission issues
- Port is already in use
- Incorrect startup parameters
- Container keeps restarting

---

### 17.3 Checking Ports

    ss -lntp | grep -E '9000|9001'

Expected Output:

    Listening on 9000
    Listening on 9001

---

### 17.4 Checking API Accessibility

    curl -I http://10.0.0.41:9000

If responses like 403, 400, or 405 are received, it indicates that the service port is responding.

If a connection timeout occurs:

    Check if the container is running.
    Verify port mapping.
    Check the firewall settings.
    Ensure the IP address is correct.

---

### 17.5 mc Connection Failure

Reconfigure the alias:

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

- Incorrect Endpoint specified
- Wrong AccessKey or SecretKey provided
- MinIO container not running
- Port 9000 is blocked
- Firewall interference
- Using Console port 9001 as API port

Note:

    The `mc alias` should point to the 9000 API port, not the 9001 Console port.

---

## Section Eighteen: Considerations for Production Environments

### 18.1 Single-Machine Experiments Are Not Suitable for Production

The single-machine Docker experiments described in this document are intended solely for learning basic concepts.

Production scenarios should avoid:

- Using a single machine with a single disk
- Operating on a single node with a single disk
- Directly exposing the 9000 HTTP port to the public internet
- Using the root user for business operations
- Hardcoding AccessKeys in code
- Lacking monitoring mechanisms
- Failing to implement backups
- Not using reverse proxies
- Not enabling HTTPS

---

### 18.2 Recommended Practices for Production

Production environments should adopt:

- Multiple nodes with multiple disks
- Erasure Coding for data durability
- Nginx or Load Balancers as unified entry points
- External HTTPS for secure access
- Internal HTTP for trusted network communication
- Separation of API and Console access
- Assigning minimal permissions to business users
- Using Prometheus for monitoring
- Setting up alerts for bucket and disk capacity
- Implementing mc mirror backups
- Conducting regular recovery drills

---

### 18.3 Internal HTTP and External HTTPS

In this module, the following practices will be uniformly adopted:

- Internal MinIO nodes communicate via HTTP.
- External clients use HTTPS for access.
- Nginx acts as a proxy for the 9000 API.
- The 9001 Console port is only accessible within the operations network segment or through a dedicated domain name.

Reasons:

- Using HTTP within trusted networks reduces complexity and overhead.
- External access must be encrypted via HTTPS to protect credentials and data.
- A unified entry point facilitates certificate management, auditing, rate limiting, and security controls.

---

## Section Nineteen: Common Misunderstandings

### 19.1 Treating Object Storage as a File System

Misconception:

    MinIO can be mounted as a shared directory like NFS.

Correct Understanding:

    MinIO is an object storage system. Applications should access objects through the S3 API. It is not a traditional POSIX file system and is not suitable for replacing local database data directories.

---

### 19.2 Using Port 9001 as the S3 API Port

Error:

    mc alias set local http://10.0.0.41:9001 ...

Correct:

    mc alias set local http://10.0.0.41:9000 ...

Explanation:

    Port 9000 is the S3 API port.
    Port 9001 is reserved for the Web Console.

---

https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO Erasure Coding Documentation:

https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

MinIO Distributed Deployment Documentation:

https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

MinIO User and Permissions Documentation:

https://min.io/docs/minio/linux/administration/identity-access-management.html

AWS S3 API Documentation:

https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html