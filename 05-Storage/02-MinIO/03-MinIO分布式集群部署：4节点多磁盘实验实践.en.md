# Deploying a MinIO Distributed Cluster: A Practical Experiment with 4 Nodes and Multiple Disks

Recommended path: 05-Storage/02-MinIO/03-Deploying a MinIO Distributed Cluster: A Practical Experiment with 4 Nodes and Multiple Disks.md

Tags: #MinIO #DistributedCluster #ObjectStorage #S3 #Docker #ErasureCoding #Multi-node-Multi-disk #mc #AdvancedSRE #ProductionOps

---

## I. Document Overview

This article is the third part of the MinIO series, focusing on a practical experiment with a 4-node MinIO distributed cluster using multiple disks.

Previously covered topics include:

- Basic concepts of MinIO
- Object storage, Buckets, Objects, and Prefixes
- Basics of the S3 API
- AccessKey / SecretKey
- Basic operations of the mc client
- Single-machine single-disk deployment
- Single-node multi-disk deployment
- Comparison of MinIO deployment modes

This article now moves on to the distributed cluster deployment phase.

The goal of this article is not just to provide a startup command but to ensure that the following steps are successfully completed:

    - Planning for 4 nodes
    - Organizing data directories
    - Checking hostnames and networking settings
    - Preparing Docker images
    - Starting MinIO containers on 4 nodes
    - Connecting the mc client
    - Creating Buckets
   - Uploading and downloading objects
    - Checking the cluster status
    - Preliminarily verifying node failures
    - Troubleshooting common issues
    - Cleaning up after the experiment

This experiment uses:

    - Internal node communication: HTTP
    - Client access: HTTP 9000
    - Production entry point: Nginx / LB with HTTPS

Note:

    This article focuses on the distributed cluster itself.
    The external HTTPS entry point will be discussed in parts 04 and 05 later on.
    It is not recommended to expose the HTTP port 9000 directly to the public network in a production environment.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Plan a 4-node MinIO distributed cluster.
2. Understand how MinIO is deployed with multiple nodes and disks.
3. Comprehend why the startup parameters must be consistent across all nodes.
4. Use Docker to deploy a MinIO distributed cluster on 4 nodes.
5. Deploy using fixed versions of MinIO images.
6. Connect to the distributed MinIO cluster using the mc client.
7. Create Buckets and perform object uploads and downloads.
8. Monitor the status of the MinIO cluster.
9. Preliminarily verify how the service behaves after a single node stops.
10. Master common methods for troubleshooting startup failures in a distributed MinIO cluster.
11. Recognize that distributed clusters still require monitoring, backup, HTTPS, and access control management.
12. Lay the foundation for subsequent experiments involving Nginx as an entry point, access control, data protection, and monitoring operations.

---

## III. Experimental Environment Planning

### 3.1 Experimental IP Range

The experimental IP range for this article is:

    10.0.0.0/24

Addresses that need to be avoided include:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

---

### 3.2 MinIO Cluster Nodes

This experiment uses 4 MinIO nodes:

| IP | Hostname | Role | Data Directories |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | /data/minio/disk1, /data/minio/disk2 |

Auxiliary node:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.45 | minio-client | mc client (optional) |
| 10.0.0.46 | minio-entry | Nginx / HTTPSThe Console should restrict access from the operations and maintenance network segment.

---

## VI. Key Principles

### 6.1 The startup parameters for each node must be consistent

In a distributed MinIO cluster, every node must use the same list of server endpoints.

This document uniformly uses:

    http://10.0.0.41:9000/data1
    http://10.0.0.41:9000/data2
    http://10.0.0.42:9000/data1
    http://10.0.0.42:9000/data2
    http://10.0.0.43:9000/data1
    http://10.0.0.43:9000/data2
    http://10.0.0.44:9000/data1
    http://10.0.0.44:9000/data2

Note:

    The startup commands on all 4 nodes must use the same set of endpoints.
    It is not allowed to have one set for node01 and another for node02.
    The /data1 and /data2 in the endpoint paths refer to internal container directories.
    The host directory is mapped to the container's /data1 and /data2 through volume mounting.

---

### 6.2 The root user and password on each node must be consistent

All nodes use:

    MINIO_ROOT_USER=minioadmin
    MINIO_ROOT_PASSWORD=MinioAdmin@123456

Production note:

    In actual production, much stronger passwords should be used.
    The root user is only for administrative purposes.
    Business applications should not use the root user.
    Independent business users and policies should be created later on.

---

### 6.3 The data directories must not be manually modified

The following actions are prohibited:

    Manually deleting object files under /data/minio/disk1
    Manually moving the MinIO data directory
    Manually modifying MinIO metadata
    Manually copying part of the data to other nodes in an attempt to restore it

The correct approach is:

    Use the S3 API to manipulate objects.
    Use mc to manage Buckets and objects.
    Use mc admin heal for repair and observation purposes.
    Handle disk failures according to the MinIO data protection process.

---

## VII. Pre-operation Checks

### 7.1 Check the hostname on all nodes

Perform this step on each of the 4 nodes:

node01:

    hostnamectl set-hostname minio-node01

node02:

    hostnamectl set-hostname minio-node02

node03:

    hostnamectl set-hostname minio-node03

node04:

    hostnamectl set-hostname minio-node04

Verify:

    hostname

---

### 7.2 Configure hosts files

Perform this step on all 4 nodes:

    cat >> /etc/hosts <<'EOF'
    10.0.0.41 minio-node01
    10.0.0.42 minio-node02
    10.0.0.43 minio-node03
    10.0.0.44 minio-node04
    10.0.0.45 minio-client
    10.0.0.46 minio-entry
    EOF

Verify:

    ping -c 2 minio-node01
    ping -c 2 minio-node02
    ping -c 2 minio-node03
    ping -c 2 minio-node04

Note:

    The startup commands in this document use IP endpoints and do not rely heavily on the hosts files inside containers.
    However, configuring the hosts files on the host machine can help with troubleshooting and subsequent Nginx configuration.
    If hostname-based endpoints are used later on, ensure that they can be resolved within the container as well.

---

### 7.3 Check time synchronization

Perform this step on all nodes:

    timedatectl

Recommended time zone:

    timedatectl set-timezone Asia/Shanghai

If using chrony:

    systemctl status chrony

If it is not installed:

    apt update
    apt install -y chrony
    systemctl enable --now chrony

Note:

    Object storage authentication, log analysis, and fault recovery all depend on accurate time.
    Inconsistent times across multiple nodes can increase troubleshooting difficulties.

---

### 7.4 Check Docker

Perform this step on all nodes:

    docker version
    docker info

Ensure that the Docker service is running:

    systemctl enable --now docker
    systemctl status docker

---

### 7.5 Pull images

Perform this step on## X. Starting the MinIO Distributed Cluster

### 10.1 Explanation of the Start Command

The start command must be executed on all 4 nodes.

Differences between each node:

    The containers run on different hosts.
    They mount the local /data/minio/disk1 and /data/minio/disk2 directories.

Similarities among each node:

    It is acceptable if the container names are the same, as they run on different hosts.
    It is also fine if the port numbers are the same, since they are on different hosts.
    The endpoint list must be identical.
    The root user and password must be the same.
    The image version must be consistent.

---

### 10.2 Execution on minio-node01

    docker run -d \
      --name minio \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -v /data/minio/disk1:/data1 \
      -v /data/minio/disk2:/data2 \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server \
      http://10.0.0.41:9000/data1 http://10.0.0.41:9000/data2 \
      http://10.0.0.42:9000/data1 http://10.0.0.42:9000/data2 \
      http://10.0.0.43:9000/data1 http://10.0.0.43:9000/data2 \
      http://10.0.0.44:9000/data1 http://10.0.0.44:9000/data2 \
      --console-address ":9001"

---

### 10.3 Execution on minio-node02

    docker run -d \
      --name minio \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -v /data/minio/disk1:/data1 \
      -v /data/minio/disk2:/data2 \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server \
      http://10.0.0.41:9000/data1 http://10.0.0.41:9000/data2 \
      http://10.0.0.42:9000/data1 http://10.0.0.42:9000/data2 \
      http://10.0.0.43:9000/data1 http://10.0.0.43:9000/data2 \
      http://10.0.0.44:9000/data1 http://10.0.0.44:9000/data2 \
      --console-address ":9001"

---

### 10.4 Execution on minio-node03

    docker run -d \
      --name minio \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -v /data/minio/disk1:/data1 \
      -v /data/minio/disk2:/data2 \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server \
      http://10.0.0.41:9000/data1 http://10.0.0.41:9000/data2 \
      http://10.0.0.42:9000/data1 http://10.0http://10.0.0.42:9001
http://10.0.0.43:9001
http://10.0.0.44:9001

Login:

Username: minioadmin
Password: MinioAdmin@123456

Note:

Each node has a Console port.
It is not recommended to expose all Consoles directly in production use.
In later stages, access to the Console should be controlled through designated entry points.

---

## Section Twelve: Configuring the mc Client

### 12.1 Creating the mc Configuration Directory

Execute this on the minio-client node or any management node:

mkdir -p /data/minio/mc-config

---

### 12.2 Setting Aliases

To connect to the API port of node01:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-cluster http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

To check the aliases:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 12.3 Viewing Cluster Information

Execute the following command:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Key details to note:

MinIO version
Number of nodes
Number of disks
Online status
Capacity information
Whether there are any offline drives
Whether there are any healing or warning issues

---

### 12.4 Testing Different Node Access Points

You can also set multiple aliases separately:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-node02 http://10.0.0.42:9000 minioadmin 'MinioAdmin@123456'

To check the information for node02:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-node02

Note:

In a distributed cluster, clients can access any healthy node.
It is not recommended to allow clients to manually select nodes in production use.
In later stages, a unified access point should be provided through Nginx or Load Balancers.

---

## Section Thirteen: Creating a Bucket and Verifying Object Read/Write Operations

### 13.1 Creating a Bucket

Execute the following command:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-cluster/app-uploads

To check the created bucket:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-cluster

---

### 13.2 Uploading a Small File

Create a test file:

mkdir -p /tmp/minio-cluster-demo

echo "hello distributed minio cluster" > /tmp/minio-cluster-demo/hello.txt

Upload the file:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demoKey Points to Note:

    Check whether the data directory is located on the expected disk.
    Verify if the data directory has been written to the system disk.
    Ensure that the capacity of each node is consistent.
    Identify any nodes with significantly insufficient disk space.

---

## Section XV: Preliminary Node Fault Verification

### 15.1 Experimental Instructions

This section only performs lightweight node fault verification:

    Stop one MinIO container.
    Observe changes in the cluster status.
    Attempt to read existing objects.
    Try to write new objects.
    Restart the container.
    Monitor the cluster's recovery process.

Detailed discussions on formal disk failures, healing processes, node recovery, and data protection mechanisms will be provided in the following document:

    08-MinIO Data Protection: Erasure Coding, Node Faults, and Disk Failure Recovery.md

---

### 15.2 Pre-Fault Verification Checks

First, confirm the cluster's health status:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Verify that objects can be read:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-cluster/app-uploads/hello.txt /demo/before-failure.txt

Check the content:

    cat /tmp/minio-cluster-demo/before-failure.txt

---

### 15.3 Stop the minio-node04 Container

Execute the following command on minio-node04:

    docker stop minio

Verify the status:

    docker ps -a | grep minio

---

### 15.4 Check the Cluster Status

Run the following command on either the minio-client or a management node:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Expected results:

    Some nodes or disks may be listed as offline.
    The cluster status might display warnings.
    The actual availability will depend on the current erasure set configuration, the number of online disks, and the read/write quorum requirements.

---

### 15.5 Verify Object Reading

Attempt to download an existing object:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-cluster/app-uploads/hello.txt /demo/read-during-node04-down.txt

Check the result:

    cat /tmp/minio-cluster-demo/read-during-node04-down.txt

If the reading is successful, it indicates that the remaining nodes and disks still meet the requirements for reading operations.

If the reading fails, check the following details:

    admin info
    docker logs
    Number of online nodes
    Number of online disks
    Error messages displayed

---

### 15.6 Verify Object Writing

Create a new file:

    echo "write during node04 down" > /tmp/minio-cluster-demo/write-during-node04-down.txt

Upload the file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/write-during-node04-down.txt minio-cluster/app-uploads/write-during-node04-down.txt

Note:

    If the upload is successful, it means that the current cluster still meets the requirements for write operations.
    If the upload fails, it indicates that the write quorum is not met under the current fault conditions or that the cluster is unavailable.
    The specific behavior may vary depending on the version and erasure set configuration; actual test results should be used as a reference.

---

### 15.7 Restart the minio-node04 Container

Restart minio-node04:

    docker start minio

Verify the status### Admin Info for Minio-Cluster

---

### 16.4 A Certain Node Goes Offline

Troubleshooting Steps:

    ping -c 2 10.0.0.44
    ssh root@10.0.0.44
    docker ps -a | grep minio
    docker logs minio
    ss -lntp | grep 9000
    df -hT
    lsblk
    dmesg | tail -100

Common Causes:

    Node crash.
    Docker service stopped.
    MinIO container exited.
    Data disk unavailable.
    Network issue.
    Port blocked by firewall.
    System disk full.
    Abnormal time.

---

### 16.5 Failed to Create a Bucket

Troubleshooting Steps:

    Check if the mc alias is functioning properly.
    Verify if the current user has the necessary permissions.
    Confirm whether the cluster is ready.
    Check if the bucket already exists.
    Ensure the bucket name is valid.
    Verify if the cluster meets the write quorum requirement.

Commands to Use:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-cluster

---

### 16.6 Slow Upload and Download Speeds

Basic Troubleshooting Steps:

    Run admin info.
    Check docker stats.
    View df -hT results.
    Use iostat -x 1 to analyze I/O performance.
    Monitor traffic with iftop.
    Perform ping and iperf3 tests.

Possible Causes:

    Slow disk performance.
    Poor network connection.
    High node load.
    Insufficient resources on the Docker host.
    Inadequate concurrent connections from a single client.
    If using Nginx as a proxy, check proxy configuration issues.

---

## Chapter Seventeen: Precautions for Production Environments

### 17.1 This Cluster Is Still an Experimental One

Although this is a distributed cluster with 4 nodes, it remains in an experimental phase.

Additional requirements for production environments include:

    Official data disks.
    Unified HTTPS access via Nginx/LB.
    Console access control measures.
    Separate policies for regular business users and specific use cases.
    Prometheus monitoring system.
    Alert notification mechanisms.
    mc mirror backup mechanism.
    Fault recovery drills.
    Certificate management procedures.
    Log collection systems.
    Established change management processes.

---

### 17.2 Do Not Expose Port 9000 Directly to the Public Network

For experimental purposes, direct access is allowed via:

    http://10.0.0.41:9000

However, in production environments, it is not recommended to do so. Instead, use:

    https://s3.example.com

This URL should be forwarded by Nginx or an LB to the backend MinIO nodes at:

    http://10.0.0.41:9000
    http://10.0.0.42:9000
    http://10.0.0.43:9000
    http://10.0.0.44:9000

---

### 17.3 Do Not Allow Business Applications to Use the root User

In this experimental setup, users like `minioadmin` and `MinioAdmin@123456` are used, but this is only for learning purposes.

For production environments:

    The root user should be restricted to administrative tasks only.
    Business applications should use separate AccessKeys.
    Each business scenario should have its own specific policy for access control.
    Permission controls should be based on Buckets or Prefixes.
    In case of key leakage, individual accounts can be quickly disabled and rotated.

---

### 17.4 Erasure Coding Is Not the Same as Backups

Erasure Coding in distributed MinIO serves the following purposes:

    Tolerance to disk failures.
    Tolerance to node failures.
    Data block reconstruction.

However, it cannot prevent:

    Accidental deletion of Buckets or objects.
    Malicious actions by administrators.
    Ransomware attacks.
    Complete cluster damage.
    Large-scale data center failures.

Therefore, in production environments, additional measures such as:

    mc mirror backup.
    Cross-cluster data synchronization.
    Offsite data backups.
11. You can check the "live" and "ready" status by using the curl health endpoint.
12. After stopping a single node, it is necessary to observe the cluster's status, read/write performance, and recovery process.
13. In a production environment, it is not recommended to expose HTTP port 9000 directly.
14. In production scenarios, HTTPS should be provided through Nginx or load balancing services as a unified entry point.
15. Erasure Coding is not the same as backup; additional knowledge in backup migration and disaster recovery is still required.

---

## Section Twenty-Two: Reference Documents

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO Multi-Node, Multi-Disk Deployment Documentation:

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html

MinIO Erasure Coding Documentation:

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

MinIO Management Command Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc-admin.html

MinIO Monitoring Documentation:

    https://min.io/docs/minio/linux/operations/monitoring.html

AWS S3 API Documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html