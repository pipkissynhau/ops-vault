# RustFS Cluster Deployment Practice: Multiple Nodes, Multiple Disks, and Access Points

Recommended Path: 05-Storage/04-RustFS/04-RustFS Cluster Deployment Practice: Multiple Nodes, Multiple Disks, and Access Points.md

Tags: #RustFS #Object Storage #S3 #Distributed Object Storage #Docker #Multiple Nodes #Multiple Disks #ErasureCoding #Nginx #Reverse Proxy #Unified Entrance #HTTPS #mc #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the fourth part of the RustFS module, focusing on the practical deployment of RustFS clusters with multiple nodes and disks.

Previously completed sections include:

    01-RustFS Basics: S3-compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-node and Cluster Modes
    03-RustFS Single-node Deployment Practice: Service Startup, Data Directory Setup, and Access Verification

This article delves into the practical deployment of RustFS clusters, covering:

    - Multi-node deployment planning
    - Multiple disk directory configuration
    - Hostname resolution settings
    - Time synchronization checks
    - Data directory preparation
    - XFS/ext4 file system selection
    - Using a fixed version of the RustFS Docker image across nodes
    - Starting a 4-node RustFS cluster
    - Checking the status of each node's services
    - Verifying the health of the S3 API
    - Authenticating access using the mc client
    - Creating Buckets
    - Uploading and downloading Objects
    - Observing data directory performance across nodes
    - Monitoring single-node failures and recovery processes
    - Configuring a unified Nginx entrance point
    - Providing access to RustFS through a single entry point
    - Troubleshooting common issues in cluster mode
    - Explanation of production boundaries

The deployment model used in this article is:

    MNMD: Multiple Node Multiple Disk

This document emphasizes that the Docker-based multi-node cluster experiments are intended to help understand how to start, access, and maintain RustFS clusters. However, they do not represent a complete production architecture. Before going live, it is necessary to conduct additional tests, such as performance benchmarking, S3 compatibility verification, failure recovery testing, permission management, HTTPS configuration, security auditing, and backup and migration validation.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Plan the experimental environment for a RustFS multi-node cluster.
2. Understand the MNMD multi-node multiple disk deployment model of RustFS.
3. Grasp why unified hostname resolution is essential in a cluster setup.
4. Comprehend why time synchronization is critical for multi-node object storage systems.
5. Prepare data directories on 4 nodes for RustFS.
6. Use a fixed version of the RustFS Docker image to start multiple nodes.
7. Configure RUSTFS_VOLUMES to point to multiple nodes and data directories.
8. Monitor the status and logs of each RustFS container.
9. Verify the health of the S3 API on each node.
10. Access the cluster using the mc client from any node.
11. Create Buckets.
12. Upload and download Objects.
13. Ensure the consistency of uploaded and downloaded files.
14. Configure a unified access point through Nginx.
15. Force clients to use the unified entrance instead of accessing individual nodes directly.
16. Understand the design differences between internal HTTP and external HTTPS.
17. Simulate the impact of a single-node failure and observe the resulting effects.
18. Restore a failed node and monitor the cluster's recovery process.
19. Troubleshoot common issues related to ports, DNS, permissions, time synchronization, directory mounting, keys, reverse proxies, etc.
20. Lay a foundation for future comparisons between RustFS and MinIO, as well as for addressing client access, security, and operational challenges.

---

## III. Key Official Deployment Considerations

When deploying a RustFS cluster with multiple nodes and disks, the following factors are crucial:

    - Multiple Nodes
    - Multiple Disks
    - Continuous Hostname Assignment
    - Network Connectivity Between Nodes
    - Time Synchronization
    - Unified Port Configuration
    - Capacity Planning
    - Disk Selection
    - File System Configuration
    - Erasure Coding
    - Access Points

In official documentation, the MNMD mode is often represented in the following format:

    RUSTFS_VOLUMES="http://node{1...4}:9000/data/rustfs{0...3}"

This format indicates that there are 4 nodes (node1 to node4), and each node has 4 data directories (rustfs0 to rustfs3). In total, this results in 16 data access paths. RustFS uses these paths to form a distributed object storage    │ /data/rustfs2 /data/rustfs3               │
    └───────────────────────────────────────────┘

    ┌───────────────────────────────────────────┐
    │ rustfs-node03                             │
    │ 10.0.0.53                                 │
    │ /data/rustfs0 /data/rustfs1               │
    │ /data/rustfs2 /data/rustfs3               │
    └───────────────────────────────────────────┘

    ┌───────────────────────────────────────────┐
    │ rustfs-node04                             │
    │ 10.0.0.54                                 │
    │ /data/rustfs0 /data/rustfs1               │
    │ /data/rustfs2 /data/rustfs3               │
    └───────────────────────────────────────────┘

---

### 5.2 Unified Entry Topology

    ┌───────────────────────────────┐
    │ rustfs-client                 │
    │ 10.0.0.55                     │
    │ mc / aws cli                  │
    └───────────────┬───────────────┘
                    │
                    │ HTTP / HTTPS
                    v
    ┌───────────────────────────────┐
    │ rustfs-entry                  │
    │ 10.0.0.56                     │
    │ Nginx Unified Entry                │
    └───────────────┬───────────────┘
                    │
                    │ HTTP Internal Network Forwarding
                    v
    ┌──────────────────────────────────────────────┐
    │ RustFS Backends                              │
    │ 10.0.0.51:9000                               │
    │ 10.0.0.52:9000                               │
    │ 10.0.0.53:9000                               │
    │ 10.0.0.54:9000                               │
    └──────────────────────────────────────────────┘

---

## VI. Pre-Deployment Checks

### 6.1 Checking Host Information on All RustFS Nodes

Execute the following commands on nodes 10.0.0.51 to 10.0.0.54:

    hostname
    hostname -I
    cat /etc/os-release
    uname -a
    timedatectl

Requirements:

    The hostname must be correct.
    The IP address must be correct.
    Time synchronization must be functioning properly.
    The operating system must match the expected configuration.

---

### 6.2 Configuring Hostnames

Execute the following commands on each node:

For node 10.0.0.51:

    hostnamectl set-hostname rustfs-node01

For node 10.0.0.52:

    hostnamectl set-hostname rustfs-node02

For node 10.0.0.53:

    hostnamectl set-hostname rustfs-node03

For node 10.0.0.54:

    hostnamectl set-hostname rustfs-node04

For node 10.0.0.55:

    hostnamectl set-hostname rustfs-client

For node 10.0.0.56:

    hostnamectl set-hostname rustfs-entry

---

### 6.3 Configuring Hosts Resolution

Execute the following command on all RustFS nodes, client nodes, and entry node:

    cat >> /etc/hosts <<'EOF'
    10.0.0.51 rustfs-node01
    10.0.0.52 rustfs-node02
    10.0.0.53 rustfs-node03
    10.0.0.54 rustfs-node04
    10.0.0.55 rustfs-client
    10.0.0.56 rustfs-entry
    10.0.0.56 s3.rustfs.local
    EOF

Check the configuration by executing:

    getent hosts rustfs-node01
    getent hosts rustfs-node02
    getent hosts rustfs-node03
    getent hosts rustfs-node04
    getent hosts s3.rustfs.local

---

### 6.4 Checking Network Connectivity Between Nodes

Execute the following command on each RustFS node:

    ping -c 3 rustfs-node01
    ping -c 3 rustfs-node02
    ping -c 3 rustfs-node03
   ```markdown
mkfs.xfs -f -i size=512 -n ftype=1 -L RUSTFS3 /dev/sde

Create mount directories:

mkdir -p /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Write to /etc/fstab:

cat >> /etc/fstab <<'EOF'
LABEL=RUSTFS0 /data/rustfs0 xfs defaults,noatime,nodiratime 0 0
LABEL=RUSTFS1 /data/rustfs1 xfs defaults,noatime,nodiratime 0 0
LABEL=RUSTFS2 /data/rustfs2 xfs defaults,noatime,nodiratime 0 0
LABEL=RUSTFS3 /data/rustfs3 xfs defaults,noatime,nodiratime 0 0
EOF

Mount:

mount -a

Check:

df -hT /data/rustfs0
df -hT /data/rustfs1
df -hT /data/rustfs2
df -hT /data/rustfs3
lsblk -f

---

### 7.3 Set directory permissions

RustFS Docker containers typically use UID 10001.

Execute on each RustFS node:

chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

View:

ls -ld /data/rustfs*

It is not recommended to use:

chmod -R 777 /data/rustfs*

---

## Section 8: Image Preparation

### 8.1 Set image variables

Execute on each RustFS node:

export RUSTFS_VERSION="1.0.0-alpha.99"
export RUSTFS_IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:${RUSTFS_VERSION}"

View:

echo ${RUSTFS IMAGE}

---

### 8.2 Pull the image

Execute:

docker pull ${RUSTFS_IMAGE}

View:

docker images | grep rustfs

If the image does not exist, you need to first pull it from the official repository on a network-connected machine and then sync it to the Alibaba Cloud repository:

docker pull rustfs/rustfs:1.0.0-alpha.99
docker tag rustfs/rustfs:1.0.0-alpha.99 registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99
docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

---

## Section 9: Planning RustFS Cluster Startup Parameters

### 9.1 Unified variables

All RustFS nodes use the same variables:

RUSTFS_ACCESS_KEY=rustfsadmin
RUSTFS_SECRET_KEY=RustFSAdmin@123456
RUSTFS_ADDRESS=:9000
RUSTFS-console_ADDRESS=:9001
RUSTFS_CONSOLE_ENABLE=true
RUSTFS_VOLUMES=http://rustfs-node{01...04}:9000/data/rustfs{0...3}

Explanation:

- rustfs-node{01...04} refers to nodes rustfs-node01 to rustfs-node04.
- rustfs{0...3} refers to data directories rustfs0 to rustfs3.
- In total, 4 nodes x 4 data directories = 16 data paths.
- All nodes must use the same RUSTFS_VOLUMES setting.
- All nodes must be able to resolve domain names from rustfs-node01 to rustfs-node04.
- All nodes must use port 9000.

---

### 9.2 Key explanation

For this experiment, the keys are:

AccessKey: rustfsadmin
SecretKey: RustFSAdmin@123456

Production notes:

- Do not use these experimental keys.
- Never commit the keys to Git.
- Do not share administrator keys among multiple services.
- In production environments, access keys should be managed at the service level with proper permission controls.
- It is essential to implement a rotation mechanism for management keys.

---

## Section 10: Starting RustFS Cluster Containers

### 10.1 Clean up old containers on all RustFS nodes before starting

Execute on nodes 10.0.0.51 to 10.0.0.54:

docker rm -f rustfs-cluster || true

---

### 10.2 Start containers on all RustFS nodes

Execute the same command on each node within the range 10.0.0.51 to 10.http://10.0.0.54:9001/rustfs/console

Login:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

Note:

    During the experimental phase, you can directly access the node Console.
    In production, it is not recommended to expose multiple Consoles.
    The production management entrance should have separate access controls and use HTTPS, authentication, and auditing.

---

## Section XII: mc Client Access Verification

### 12.1 Prepare the mc Configuration Directory

Execute this command on rustfs-client or any test node:

    mkdir -p /data/rustfs/mc-config

For Docker version of mc:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

### 12.2 Configure an Alias through a Single Backend Node

First, test it with rustfs-node01:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-node01 http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

Check the configuration:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 12.3 Create a Bucket

Create a Bucket:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-node01/cluster-demo

Check the creation:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-node01

Expected result:

    cluster-demo

---

### 12.4 Upload a Test File

Prepare the test directory:

    mkdir -p /tmp/rustfs-cluster-test
    cd /tmp/rustfs-cluster-test

Create files:

    echo "hello rustfs cluster" > hello.txt
    date > create-time.txt
    mkdir -p logs config

    cat > config/app.conf <<'EOF'
    app_name=rustfs-cluster-demo
    env=lab
    endpoint=http://10.0.0.56:9000
    EOF

    echo "2026-04-28 INFO rustfs cluster upload test" > logs/app.log

Create a large file:

    dd if=/dev/zero of=file-100m.bin bs=1M count=100

Calculate the hash value:

    sha256sum hello.txt create-time.txt config/app.conf logs/app.log file-100m.bin > sha256-before.txt
    cat sha256-before.txt

Upload the files:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-cluster-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/ rustfs-node01/cluster-demo/

Check the upload:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-node01/cluster-demo

---

### 12.5 Read Objects from Another Node

Configure an alias for rustfs-node02:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
     🔤 Do not make any manual modifications.
🔤 Do not delete anything manually.
🔤 Do not directly use the command `rm -rf /data/rustfs*`. Object deletion should be performed through the S3 API, mc, or the Console.```bash
tail -100 /var/log/nginx/error.log
```

Explanation:

Nginx may record connection failures with node04. However, as long as the other backends are available, the unified entry point can still continue to provide service. To determine whether read and write operations are fully functional, it is necessary to verify the current cluster status of RustFS, including the number of ECs, failed nodes, and object distribution.## Section Nineteen: Production Precautions

### 19.1 Do Not Use Experimental Directory Structures as Production Solutions

In experiments:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

These directories may all reside on the same system disk.

In production:

    Each directory should correspond to an independent disk or mount point.
    Production data should not be stored on the system disk.
    Multiple data directories should not be placed on the same disk; claiming multi-disk fault tolerance based on this is incorrect.

---

### 19.2 Avoid Using NFS for Underlying Data Directories

The use of NFS as the underlying data directory for RustFS is not recommended.

Reasons:

    High I/O operations may lead to lock issues, latency, write consistency problems, and performance degradation.
    The object storage software layer should directly manage local disks.
    In production, it is suggested to expose local physical disks using JBOD technology.

---

### 19.3 Do Not Directly Expose Backend Node Ports to the Public Network

In production, the following practices should be avoided:

    Public network access to ports like 10.0.0.51:9000 or 10.0.0.52:9000.
    Public network access to the Console port at 9001.

Recommended practices:

    Backend nodes should only be accessible via the internal network.
    External access should be routed through a unified Nginx/LB entry point.
    All external connections must use HTTPS.
    The Console management interface should have source restrictions.
    AccessKey and SecretKey should not be transmitted over the public network in plaintext via HTTP.

---

### 19.4 Do Not Use Experimental Keys

The keys used in this document, such as `rustfsadmin` and `RustFSAdmin@123456`, are for experimental purposes only.

In production:

    Strongly random keys must be generated.
    Root/ Admin keys should only be used for initialization purposes.
    Business operations should use separate AccessKeys.
    Permissions should be minimized.
    Keys should be rotated regularly.
    In case of leakage, they can be immediately disabled.

---

### 19.5 RustFS Still Requires Production-Proofing

Even if the cluster deployment is successful, additional verification is necessary for the following aspects:

    S3 API compatibility.
    Multipart Upload functionality.
    Presigned URL support.
    SDK compatibility.
    Ability to upload and download large objects.
    High-concurrency handling of small objects.
    Recovery from node failures.
    Recovery from disk failures.
    Data recovery capabilities.
    Version upgrade mechanisms.
    Monitoring metrics.
    Log auditing features.
    Permission management system.
    HTTPS reverse proxy settings.
    Backup and migration procedures.
    Comparison with solutions like MinIO/Ceph RGW.

---

## Section Twenty: Experimental Cleanup

### 20.1 Delete the Test Bucket

On the `rustfs-client` node, execute the following command:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force rustfs-entry/cluster-demo

To delete the bucket, use this command in a Docker container:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb rustfs-entry/cluster-demo

High-risk warning:

    Deleting a bucket in production is a highly risky operation.
    Before deletion, make sure to confirm the business relevance, backup measures, and obtain necessary approvals.

---

### 20.2 Stop the RustFS Containers

On each RustFS node, execute the following command:

    docker stop rustfs-cluster

---

### 20.3 Delete the RustFS Containers

On each RustFS node, execute the following command:

    docker rm -f rustfs-cluster

---

### 20.4 Delete Data Directories

High-risk warning:

    The following commands will delete all experimental data from the RustFS cluster.
    These operations are strictly prohibited in a production environment.

After confirming that only experimental data needs to be cleared, execute the following commands on each node:

    rm -rf /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

---

### 20.5 Delete Nginx Configuration Files

On the `rustfs-entry` node, execute the following command:

    rm -f /etc/nginx2. MNMD is more similar to the architecture of production object storage clusters.
3. This document plans to use 4 RustFS nodes.
4. This document plans to use 1 client node.
5. This document plans to use 1 Nginx unified entry node.
6. All RustFS nodes must be able to resolve hostnames for each other.
7. All RustFS nodes must maintain time synchronization.
8. All RustFS nodes must use the same listening ports.
9. In this document, each node is planned to have directories from /data/rustfs0 to /data/rustfs3.
10. The experimental directory structure cannot replace independent production disks.
11. For production environments, it is recommended that each data directory correspond to an independent disk.
12. It is not recommended to use NFS for RustFS's underlying data directories.
13. Docker container data directories must be mounted on the host machine.
14. The host machine's data directories must allow UID 10001 to write to them.
15. All nodes should use the same RUSTFS_VOLUMES setting.
16. The mc can access the same cluster through any node.
17. A unified entry point can be implemented using Nginx.
18. When setting up the Nginx entry point, attention should be paid to upload size, request buffering, and timeouts.
19. External production access must use HTTPS.
20. The ports of backend nodes should not be directly exposed to the public internet.
21. Stopping a single node can help observe the impact on cluster access.
23. Just because a cluster is successfully deployed does not mean it is ready for production use.
24. In the next article, we will compare RustFS with MinIO in terms of architecture, deployment, ecosystem, and operational differences.

---

## 24. References

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS Docker installation documentation:

    https://docs.rustfs.com/installation/docker/

RustFS multi-node, multi-disk installation documentation:

    https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html

RustFS on Docker Hub:

    https://hub.docker.com/r/rustfs/rustfs

RustFS GitHub repository:

    https://github.com/rustfs/rustfs

RustFS Nginx reverse proxy configuration documentation:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS TLS configuration documentation:

    https://docs.rustfs.com/integration/tls-configuration/

MinIO mc client documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

AWS S3 API documentation:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

AWS CLI S3 documentation:

    https://docs.aws.amazon.com/cli/latest/reference/s3/

Nginx official documentation:

    https://nginx.org/en/docs/

Docker official documentation:

    https://docs.docker.com/