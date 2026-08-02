# MinIO Distributed Cluster Deployment: 4-Node Multi-Disk Experiment Practice

Suggested Path: 05-Storage/02-MinIO/03-MinIO Distributed Cluster Deployment: 4-Node Multi-Disk Experiment Practice.md

Tags: #MinIO #DistributedClusters #ObjectStorage #S3 #Docker #ErasureCoding #Multinodes #mc #AdvancedSre #ProductionTransport

---

## I. Document Description

This document is the third article in the MinIO module, focusing on completing a hands-on 4-node multi-disk MinIO distributed cluster experiment.

Previously completed:

- MinIO basic concepts
- Object storage, Bucket, Object, Prefix
- S3 API basics
- AccessKey / SecretKey
- mc client basic operations
- Single-node single-disk deployment
- Single-node multi-disk deployment
- MinIO deployment mode comparison

This document officially enters the distributed cluster deployment phase.

The goal of this document is not just to write a single startup command, but to fully run through:

    4-node planning
    Data directory planning
    Hostname and network checks
    Docker image preparation
    4-node MinIO container startup
    mc client connection
    Bucket creation
    Object upload/download
    Cluster status check
    Node failure preliminary verification
    Common failure troubleshooting
    Experiment cleanup

This experiment uses:

    Internal node communication: HTTP
    Client experiment access: HTTP 9000
    Subsequent production entry: Nginx / LB HTTPS unified entry

Notes:

    This article focuses on the distributed cluster itself.
    External HTTPS unified entry will be discussed separately in sections 04 and 05.
    It is not recommended to expose the 9000 HTTP port directly to the public in production environments.

---

## II. Learning Objectives

After completing this document, you should be able to:

1. Plan a 4-node MinIO distributed cluster.
2. Understand MinIO's multi-node multi-disk deployment method.
3. Understand why each node's startup parameters must be consistent.
4. Use Docker to start a MinIO distributed cluster on 4 nodes.
5. Deploy using a fixed version MinIO image.
6. Connect to a distributed MinIO using the mc client.
7. Create a Bucket and upload/download objects.
8. View MinIO cluster status.
9. Preliminary verification of service status changes after stopping a single node.
10. Master common troubleshooting methods for distributed MinIO startup failures.
11. Understand that distributed clusters still require monitoring, backup, HTTPS, and access control.
12. Lay the foundation for subsequent Nginx unified entry, access management, data protection, and monitoring operations.

---

## III. Experiment Environment Planning

### 3.1 Experiment Network Segment

This experiment's network segment:

    10.0.0.0/24

Need to avoid existing addresses:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

---

### 3.2 MinIO Cluster Nodes

This document uses 4 MinIO nodes:

| IP | Hostname | Role | Data Directory |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | /data/minio/disk1, /data/minio/disk2 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | /data/minio/disk1, /data/minio/disk2 |

Auxiliary nodes:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.45 | minio-client | mc client, optional |
| 10.0.0.46 | minio-entry | Nginx / HTTPS unified entry, for subsequent use |

---

### 3.3 Operating System

Recommended:

    Ubuntu Server 22.04.5 LTS

Optional:

    Rocky Linux 9

This document's commands default to Ubuntu Server 22.04.5 LTS.

---

### 3.4 Container Runtime

This document uses:

    Docker

Does not use Kubernetes.

Reasons:

    This module focuses on learning MinIO's own distributed object storage capabilities.
    Docker is more suitable for quickly setting up multi-node experiments.
    MinIO typically provides services through S3 API, not necessarily via Kubernetes PVC.
    Does not disrupt the Kubernetes containerdBottom runtime, aligning with advanced SRE experiment methods.

---

## IV. Image Version Notes

### 4.1 MinIO Server Image

This document uses a fixed version image already synchronized to the Alibaba Cloud registry:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

Source image:

    minio/minio:RELEASE.2025-04-22T22-12-26Z

---

### 4.2 mc Client Image

This document uses a fixed version image already synchronized to the Alibaba Cloud registry:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source image:

    minio/mc:RELEASE.2025-04-16T18-13-26Z

---

### 4.3 Reasons for Using Fixed Versions

This document does not use latest, reasons:

    Avoid inconsistencies in Web Console, command parameters, log formats, and experimental results due to latest changes.
    Fixed versions facilitate experiment reproducibility.
    Server and mc client versions are close in time, reducing capability differences between client and server.
    Subsequent Nginx proxy, access control, monitoring, mc mirror, etc., experiments all proceed based on the same version.

---

### 4.4 Image Source Record

This module's image source process:

    1. Pull from official fixed version:
       minio/minio:RELEASE.2025-04-22T22-12-26Z
       minio/mc:RELEASE.2025-04-16T18-13-26Z

2. Retag to the user's Alibaba Cloud image repository:
   registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
   registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

   3. Push to Alibaba Cloud repository for stable object retrieval under domestic network environments.

---

## Five. Distributed Cluster Architecture

### 5.1 Experiment Deployment Architecture

Deployment structure in this article:

    4 nodes
    2 data directories per node
    Total of 8 data directories
    1 MinIO container per node
    All nodes use identical startup parameters
    All nodes use same root user and password
    Clients access S3 API via any node's 9000 port

Architecture diagram:

    ┌──────────────────────────────┐
    │        minio-client           │
    │          10.0.0.45            │
    └──────────────┬───────────────┘
                   │
                   │ S3 API / mc
                   v
    ┌──────────────────────────────────────────────┐
    │          MinIO Distributed Cluster            │
    │                                              │
    │ 10.0.0.41 minio-node01  /data1 /data2        │
    │ 10.0.0.42 minio-node02  /data1 /data2        │
    │ 10.0.0.43 minio-node03  /data1 /data2        │
    │ 10.0.0.44 minio-node04  /data1 /data2        │
    │                                              │
    │ API:     9000                                │
    │ Console: 9001                                │
    └──────────────────────────────────────────────┘

---

### 5.2 Subsequent Production Entry Architecture

Nodes 04 and 05 will evolve to:

    Client
      |
      | HTTPS 443
      v
    Nginx / LB
      |
      | HTTP 9000
      v
    MinIO 4-node distributed cluster

Principles:

    Internal trusted networks use HTTP.
    External client access must use HTTPS.
    Port 9000 should not be exposed publicly.
    Port 9001 Console should not be exposed publicly.
    Console access should be restricted to maintenance network segments.

---

## Six. Key Principles

### 6.1 Startup Parameters Must Be Consistent Across All Nodes

In a distributed MinIO cluster, each node must use the same server endpoint list.

This article uniformly uses:

    http://10.0.0.41:9000/data1
    http://10.0.0.41:9000/data2
    http://10.0.0.42:9000/data1
    http://10.0.0.42:9000/data2
    http://10.0.0.43:9000/data1
    http://10.0.0.43:9000/data2
    http://10.0.0.44:9000/data1
    http://10.0.0.44:9000/data2

Notes:

    Startup commands on all 4 nodes must use the same endpoint group.
    Cannot have node01 using one set and node02 using another.
    /data1 and /data2 in endpoint paths are container paths.
    Host directories are mapped to container /data1 and /data2 via volumes.

---

### 6.2 All Nodes Must Use Same Root User and Password

All nodes use:

    MINIO_ROOT_USER=minioadmin
    MINIO_ROOT_PASSWORD=MinioAdmin@123456

Production reminder:

    Actual production must use stronger passwords.
    Root user is only for management.
    Business applications should not use root user.
    Subsequent steps should create independent business users and Policies.

---

### 6.3 Data Directories Cannot Be Manually Modified

Prohibited:

    Manual deletion of objects in /data/minio/disk1
    Manual movement of MinIO data directories
    Manual modification of MinIO metadata
    Manual copying of partial data to other nodes to fake recovery

Correct approach:

    Operate objects via S3 API.
    Manage Buckets and objects via mc.
    Use mc admin heal for repair observation.
    Handle disk failure recovery via MinIO data protection procedures.

---

## Seven. Pre-Operation Checks

### 7.1 Check Hostnames on All Nodes

Execute on each of the 4 nodes.

node01:

    hostnamectl set-hostname minio-node01

node02:

    hostnamectl set-hostname minio-node02

node03:

    hostnamectl set-hostname minio-node03

node04:

    hostnamectl set-hostname minio-node04

Check:

    hostname

---

### 7.2 Configure hosts

Execute on all 4 nodes:

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

Note: /etc/hosts

The startup command uses IP endpoint, not strongly relying on the hosts inside the container.
But the host's hosts configuration helps with manual troubleshooting and subsequent Nginx configuration.
If using hostname as endpoint later, ensure the hostname can be resolved inside the container.

---

### 7.3 Check Time Synchronization

All nodes execute:

    timedatectl

Recommended timezone:

    timedatectl set-timezone Asia/Shanghai

If using chrony:

    systemctl status chrony

If not installed:

    apt update
    apt install -y chrony
    systemctl enable --now chrony

Notes:

    Object storage authentication, log troubleshooting, and incident review all depend on accurate time.
    Inconsistent time across multiple nodes increases troubleshooting difficulty.

---

### 7.4 Check Docker

All nodes execute:

    docker version
    docker info

Ensure Docker service is running:

    systemctl enable --now docker
    systemctl status docker

---

### 7.5 Pull Images

All MinIO nodes execute:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client node or any management node execute:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Check images:

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq

---

### 7.6 Domestic Network Optimization Notes

This document assumes users have already synchronized MinIO and mc images to Alibaba Cloud's registry.

If pulling other images later, handle according to the currently configured Docker image acceleration source, for example:

    mdockler image acceleration source
    Alibaba Cloud personal image repository
    Enterprise internal network Harbor
    Other trusted Registry Mirror

Principles:

    Prioritize fixed version images.
    Prioritize controllable image repositories.
    Do not use latest.
    Do not disrupt Kubernetes containerd configuration for pulling images.

---

## VIII. Data Directory Preparation

### 8.1 Create Directories on All Nodes

On 4 MinIO nodes execute:

    mkdir -p /data/minio/disk1
    mkdir -p /data/minio/disk2

Check:

    ls -ld /data/minio/disk1 /data/minio/disk2

---

### 8.2 Confirm Disk Location of Directories

Execute:

    df -hT /data/minio/disk1
    df -hT /data/minio/disk2
    lsblk

In experimental environments:

    disk1 and disk2 can temporarily be directories.
    But if both are on the same system disk, they can only be used for learning.
    Cannot be used as real production high availability.

In production environments:

    /data/minio/disk1 should be an independent data disk mount point.
    /data/minio/disk2 should be an independent data disk mount point.
    Do not recommend writing object data to the system disk.
    Do not recommend multiple "disk directories" actually falling on the same physical disk.

---

### 8.3 Optional: Real Disk Mount Example

If the node has /dev/sdb and /dev/sdc, you can refer to the following process.

High-risk warning:

    The following commands will format the disk.
    Only execute in confirmed experimental environments with no data on the disk.
    In production environments, must confirm disk numbering to avoid accidental formatting.

Check disks:

    lsblk

Format example:

    mkfs.xfs /dev/sdb
    mkfs.xfs /dev/sdc

Create mount points:

    mkdir -p /data/minio/disk1
    mkdir -p /data/minio/disk2

Temporary mount:

    mount /dev/sdb /data/minio/disk1
    mount /dev/sdc /data/minio/disk2

Check UUID:

    blkid /dev/sdb
    blkid /dev/sdc

When writing to /etc/fstab, recommend using UUID instead of directly relying on /dev/sdb names.

Verify:

    df -hT
    lsblk

---

## IX. Firewall and Port Check

### 9.1 Check Port Usage

All nodes execute:

    ss -lntp | grep -E '9000|9001'

If an old MinIO container is using the port, stop it first:

    docker ps -a | grep minio
    docker stop <container-name>
    docker rm <container-name>

---

### 9.2 Allow Ports

In experimental environments, if ufw is enabled, execute on 4 nodes:

    ufw allow from 10.0.0.0/24 to any port 9000 proto tcp
    ufw allow from 10.0.0.0/24 to any port 9001 proto tcp
    ufw status numbered

Notes:

    9000 is the S3 API and node communication port.
    9001 is the Console port.
    In production, 9001 should not be open to general networks.

Skip if no firewall is enabled.

---

### 9.3 Check Node Connectivity

Before startup, 9000 may not be listening yet. You can first check basic network:

    ping -c 2 10.0.0.41
    ping -c 2 10.0.0.42
    ping -c 2 10.0.0.43
    ping -c 2 10.0.0.44

Check 9000 after startup:

    nc -vz 10.0.0.41 9000
    nc -vz 10.0.0.42 9000
    nc -vz 10.0.0.43 9000
    nc -vz 10.0.0.44 9000

If nc is not installed:

    apt install -y netcat-openbsd

---

## X. Start MinIO Distributed Cluster

### 10.1 Startup Command Notes

All 4 nodes must execute the startup command.

Each node's difference: /think

Containers are running on different hosts.
Mounted are the local /data/minio/disk1 and /data/minio/disk2.

Same Points for Each Node:

- Container name can be the same as well, because they are on different hosts.
- Port can be the same as well, because they are on different hosts.
- endpoint list must be consistent.
- root user and password must be consistent.
- Image version must be consistent.

---

### 10.2 Execute on minio-node01

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

### 10.3 Execute on minio-node02

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

### 10.4 Execute on minio-node03

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

### 10.5 Execute on minio-node04

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

### 10.6 Startup Order Notes

When starting a distributed MinIO, some nodes may temporarily wait for others to start first.

This is a normal phenomenon.

Recommendations:

- Start all 4 nodes as soon as possible.
- Do not judge failure after starting only one node.
- After startup, check docker logs uniformly.
- If the cluster cannot form for a long time, then troubleshoot network, ports, and parameter consistency.

---

## Eleven. Check Cluster Status

### 11.1 Check Container Status

Execute on each of the 4 nodes:

    docker ps | grep minio

Expected:

- minio container is in Up status

If not running:

    docker ps -a | grep minio
    docker logs minio

---

### 11.2 Check Logs

Execute on each node:

    docker logs --tail=100 minio

If real-time viewing is needed:

    docker logs -f minio

Normal logs should show:

    API:
    Console:
    Status:
    Documentation: /think

If the logs repeatedly show waiting, connection refused, unable to connect, check if other nodes are running, ports are accessible, and endpoints are consistent.

---

### 11.3 Check API Health

Execute from any node or minio-client:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.42:9000/minio/health/live
    curl -I http://10.0.0.43:9000/minio/health/live
    curl -I http://10.0.0.44:9000/minio/health/live

Check ready:

    curl -I http://10.0.0.41:9000/minio/health/ready
    curl -I http://10.0.0.42:9000/minio/health/ready
    curl -I http://10.0.0.43:9000/minio/health/ready
    curl -I http://10.0.0.44:9000/minio/health/ready

Notes:

    live is used to check if the process is alive.
    ready is used to check if the service is ready to provide services externally.
    If ready is abnormal, it indicates the cluster may not be fully available.

---

### 11.4 Access Console

Access any node via browser:

    http://10.0.0.41:9001
    http://10.0.0.42:9001
    http://10.0.0.43:9001
    http://10.0.0.44:9001

Login:

    Username: minioadmin
    Password: MinioAdmin@123456

Notes:

    Each node has a Console port.
    Exposing all Console ports directly is not recommended in production.
    Subsequent access should be through controlled entry points.

---

## Twelve. Configure mc Client

### 12.1 Create mc Configuration Directory

Execute on minio-client or any management node:

    mkdir -p /data/minio/mc-config

---

### 12.2 Set alias

Connect to node01's API port:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-cluster http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

View alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 12.3 View Cluster Information

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Focus on:

    MinIO version
    Number of nodes
    Number of disks
    Online status
    Capacity information
    Presence of offline drives
    Presence of healing or warning states

---

### 12.4 Test Different Node Entrypoints

You can also set multiple aliases:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-node02 http://10.0.0.42:9000 minioadmin 'MinioAdmin@123456'

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-node02

Notes:

    In a distributed cluster, clients can access any normal node.
    It is not recommended to let clients manually select nodes in production.
    Subsequent access should be through Nginx / LB for unified entry.

---

## Thirteen. Create Bucket and Verify Object Read/Write

### 13.1 Create Bucket

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb minio-cluster/app-uploads

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-cluster

---

### 13.2 Upload Small Files

Create test file:

    mkdir -p /tmp/minio-cluster-demo

    echo "hello distributed minio cluster" > /tmp/minio-cluster-demo/hello.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/hello.txt minio-cluster/app-uploads/hello.txt

View:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls minio-cluster/app-uploads

---

### 13.3 Downloading Small Files

Delete local old files:

    rm -f /tmp/minio-cluster-demo/hello-download.txt

Download:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-cluster/app-uploads/hello.txt /demo/hello-download.txt

Verify:

    cat /tmp/minio-cluster-demo/hello-download.txt

Expected output:

    hello distributed minio cluster

---

### 13.4 Uploading Large Files

Create 100M file:

    dd if=/dev/zero of=/tmp/minio-cluster-demo/file-100m.bin bs=1M count=100

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/file-100m.bin minio-cluster/app-uploads/file-100m.bin

Check object:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      stat minio-cluster/app-uploads/file-100m.bin

---

### 13.5 Viewing Bucket Capacity

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      du minio-cluster/app-uploads

View recursive objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      find minio-cluster/app-uploads

---

## FourteenI don't know.Observing Data Directories

### 14.1 Viewing Each Node's Disk Directory

Execute on each MinIO node:

    du -sh /data/minio/disk1
    du -sh /data/minio/disk2

View file distribution:

    find /data/minio -maxdepth 4 -type f | head -50

Notes:

    MinIO maintains object data and metadata across multiple data directories.
    The directory structure seen is not intended for direct human operation.
    Do not manually modify.
    Do not manually delete.
    Do not manually copy parts of directories.

---

### 14.2 Checking Disk Capacity

Execute on all MinIO nodes:

    df -hT
    lsblk

Focus on:

    Whether data directories are located on expected disks.
    Whether data directories are written to the system disk.
    Whether each node's capacity is consistent.
    Whether any node shows significantly low disk space.

---

## FifteenI don't know.Initial Node Failure Verification

### 15.1 Experiment Description

This document only performs light node failure verification:

    Stop one MinIO container.
    Observe cluster status changes.
    Try reading existing objects.
    Try writing new objects.
    Restore the container.
    Observe cluster recovery.

Formal disk failure, healing, node recovery, and data protection boundaries will be detailed in:

    08-MinIO Data Protection: Erasure Coding, Node Failure and Disk Failure Recovery.md

---

### 15.2 Pre-Experiment Checks

First confirm cluster health:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Confirm object readability:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-cluster/app-uploads/hello.txt /demo/before-failure.txt

Check:

    cat /tmp/minio-cluster-demo/before-failure.txt

---

### 15.3 Stopping minio-node04 Container

Execute on minio-node04:

    docker stop minio

Check:

    docker ps -a | grep minio

---

### 15.4 Viewing Cluster Status

Execute on minio-client or management node:

docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Expected output:

    Some nodes or disks may show as offline.
    The cluster status may show warning.
    Availability depends on current erasure set, number of online disks, and read/write quorum.

---

### 15.5 Verifying Object Read

Attempt to download an existing object:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp minio-cluster/app-uploads/hello.txt /demo/read-during-node04-down.txt

Check the result:

    cat /tmp/minio-cluster-demo/read-during-node04-down.txt

If the read succeeds, it indicates that the remaining nodes and disks still meet the read requirements.

If the read fails, check:

    admin info
    docker logs
    Number of online nodes
    Number of online disks
    Error messages

---

### 15.6 Verifying Object Write

Create a new file:

    echo "write during node04 down" > /tmp/minio-cluster-demo/write-during-node04-down.txt

Upload the file:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-cluster-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/write-during-node04-down.txt minio-cluster/app-uploads/write-during-node04-down.txt

Notes:

    If the upload succeeds, it indicates the cluster still meets the write quorum.
    If the upload fails, it indicates the write quorum is not met or the cluster is unavailable in this failure state.
    Specific behavior may vary depending on version and erasure set configuration; actual testing is recommended.

---

### 15.7 Recovering minio-node04

Run on minio-node04:

    docker start minio

Check status:

    docker ps | grep minio
    docker logs --tail=100 minio

---

### 15.8 Observing Recovery

Run on the management node:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

Check objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-cluster/app-uploads

Optional: Check healing status:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin heal --recursive --dry-run minio-cluster

Notes:

    If new objects were written during node failure, healing may be required after recovery.
    Formal data protection and repair processes will be covered separately later.

---

## SixteenI don't know.Common Troubleshooting

### 16.1 Container Fails to Start

Check:

    docker ps -a | grep minio
    docker logs minio

Common causes:

| Cause | Phenomenon |
|---|---|
| Port 9000 is occupied | bind: address already in use |
| Port 9001 is occupied | Console port binding failed |
| Data directory does not exist | Path mounting anomaly |
| Inconsistent startup parameters | Cluster cannot form |
| Some nodes not started | Other nodes are waiting |
| Image does not exist | Unable to find image |
| Password inconsistency | Authentication or cluster anomaly |
| Residual old data directory | Format or cluster ID inconsistency |

---

### 16.2 Cluster Always Fails to Ready

Troubleshooting steps:

    1. Check if all 4 node containers are running.
    2. Verify if startup parameters for all 4 nodes are exactly the same.
    3. Confirm if each node's port 9000 is listening.
    4. Check if nodes can communicate on port 9000.
    5. Verify if data directories exist.
    6. Check if data directories are empty or belong to the same cluster.
    7. Confirm if root user and password are consistent.
    8. Check if time is synchronized.
    9. Check if firewall is blocking.
    10. Check Docker logs for clear error messages.

Commands:

    docker ps
    docker logs minio
    ss -lntp | grep 9000
    nc -vz 10.0.0.41 9000
    nc -vz 10.0.0.42 9000
    nc -vz 10.0.0.43 9000
    nc -vz 10.0.0.44 9000
    timedatectl
    df -hT

---

### 16.3 mc alias Connection Failure

Common error: /think

Used 9001 Console port.  
AccessKey error.  
SecretKey error.  
9000 port unreachable.  
MinIO container not running.  
alias cached old address.

Reconfiguration:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-cluster http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Testing:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

---

### 16.4 A node is offline

Troubleshooting:

    ping -c 2 10.0.0.44
    ssh root@10.0.0.44
    docker ps -a | grep minio
    docker logs minio
    ss -lntp | grep 9000
    df -hT
    lsblk
    dmesg | tail -100

Common causes:

    Node crash.
    Docker service stopped.
    MinIO container exited.
    Data disk unavailable.
    Network unreachable.
    Port blocked by firewall.
    System disk full.
    Time anomaly.

---

### 16.5 Bucket creation failed

Troubleshooting:

    Check if mc alias is normal.
    Check if current user has permissions.
    Check if cluster is ready.
    Check if Bucket already exists.
    Check if Bucket name is valid.
    Check if cluster meets write quorum.

Commands:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info minio-cluster

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls minio-cluster

---

### 16.6 Slow upload/download

Basic troubleshooting:

    admin info
    docker stats
    df -hT
    iostat -x 1
    iftop
    ping
    iperf3

Possible causes:

    Slow disk.
    Slow network.
    High node load.
    Insufficient Docker host resources.
    Low client concurrency.
    If using Nginx later, it could also be proxy configuration issues.

---

## Seventeen, Production Environment Notes

### 17.1 This cluster is still an experimental cluster

Although this document describes a 4-node distributed cluster, it still belongs to an experimental environment.

Production environment needs additional:

    Formal data disks
    Nginx / LB unified HTTPS entry
    Console access control
    Regular business users and Policy
    Prometheus monitoring
    Alert notifications
    mc mirror backup
    Failure recovery drills
    Certificate management
    Log collection
    Change management process

---

### 17.2 Do not expose 9000 to the public directly

This document directly accesses:

    http://10.0.0.41:9000

Production is not recommended:

    http://InternetIP:9000

Production recommendation:

    https://s3.example.com

Forward to backend MinIO nodes via Nginx or LB:

    http://10.0.0.41:9000
    http://10.0.0.42:9000
    http://10.0.0.43:9000
    http://10.0.0.44:9000

---

### 17.3 Do not let business use root user

Current experiment uses:

    minioadmin
    MinioAdmin@123456

Only for learning.

Production requirements:

    Root user only for management.
    Business applications use independent AccessKey.
    Each business has independent Policy.
    Permissions control by Bucket or Prefix.
    Keys can be disabled and rotated after leakage.

---

### 17.4 Erasure Coding is not equal to backup

Distributed MinIO's Erasure Coding is used for:

    Disk failure tolerance
    Node failure tolerance
    Data block reconstruction

But cannot prevent:

    Accidental Bucket deletion
    Accidental object deletion
    Administrator misoperation
    Ransomware
    Cluster-wide damage
    Data center-level failure

Production still needs:

    mc mirror
    Cross-cluster synchronization
   Alien. backup
    Regular recovery drills

---

## Eighteen, Experiment Cleanup

### 18.1 Stop 4 node containers

Execute on each MinIO node:

    docker stop minio
    docker rm minio

---

### 18.2 Delete test data directory

High-risk warning:

    The following commands will delete all MinIO object data.
    Only allowed to execute in test environment.
    Prohibited to execute directly in production.

Execute on each MinIO node:

    rm -rf /data/minio/disk1
    rm -rf /data/minio/disk2

---

### 18.3 Delete mc configuration

If confirmed not to use:

    rm -rf /data/minio/mc-config

---

### 18.4 Production environment cleanup principles

Do not directly rm -rf data directories in production.

Production cleanup should follow:

    Confirm business ownership first.
    Confirm backup first.
    Confirm Bucket / Prefix first.
    Delete objects via mc or S3 API.
    Bucket deletion requires approval.
    High-risk operations need dual verification.
    Keep operation records.

---

## Nineteen, Productionization Acceptance Checklist

### 19.1 Deployment acceptance /think

| Check Item | Acceptance Criteria | Result |
|---|---|---|
| Node Count | At least 4 nodes, deployed according to design |  |
| Data Directory | Data directory exists on each node |  |
| Disk Mount | Data directory not on system disk or explicitly stated |  |
| Docker Container | 4-node MinIO containers all running |  |
| Port | 9000 / 9001 listening normally |  |
| Version | All nodes have consistent MinIO version |  |
| endpoint | All nodes have consistent startup parameters |  |

---

### 19.2 Functional Acceptance

| Check Item | Acceptance Criteria | Result |
|---|---|---|
| mc alias | Can connect to cluster |  |
| admin info | Can view cluster information |  |
| Bucket Creation | Can create Bucket |  |
| Object Upload | Can upload small and large files |  |
| Object Download | Downloaded content matches original |  |
| Multi-node Entry | API accessible via any normal node |  |
| Console | Can log in to management interface |  |

---

### 19.3 Fault Acceptance

| Check Item | Acceptance Criteria | Result |
|---|---|---|
| Stop Single Node | Cluster status observable |  |
| Read Object | Readable when quorum satisfied |  |
| Write Object | Writable when quorum satisfied |  |
| Recover Node | Cluster status recovers after node recovery |  |
| heal Check | Can observe via mc admin heal |  |

---

### 19.4 Production Supplement Items

| Check Item | Completed |
|---|---|
| Nginx / LB HTTPS Unified Entry |  |
| Console Access Restriction |  |
| Regular Users and Policy |  |
| Prometheus Monitoring |  |
| Grafana Dashboard |  |
| Capacity Alert |  |
| Node Failure Alert |  |
| mc mirror Backup |  |
| Recovery Drill |  |
| Log Collection |  |
| Certificate Expiry Monitoring |  |

---

## Twenty, Interview Answer Outline

If asked in an interview:

    How would you deploy a MinIO distributed cluster? How would you plan it?

You could answer:

    I would prioritize deploying MinIO with multiple nodes and disks rather than single-node single-disk. Single-node single-disk is suitable for learning and testing, while single-node multi-disk helps understand Erasure Coding but still cannot resolve node failures. Production environments recommend at least multi-node multi-disk deployment, such as 4 nodes with multiple independent data disks per node.
    During deployment, each node runs a MinIO service, and all nodes must have identical server endpoint lists in startup parameters, such as writing all data directories for 4 nodes. All nodes must also have identical root users, passwords, and image versions. Each node listens on 9000 API port and 9001 Console port.
    Internal nodes communicate via HTTP on a trusted internal network to reduce TLS management complexity and encryption/decryption overhead; however, external client access must expose HTTPS through Nginx or LB, not directly expose 9000 HTTP to public internet.
    After deployment, I would connect to the cluster using mc alias, check node and disk status with mc admin info, create Buckets, and verify S3 API by uploading/downloading objects. Then I would conduct simple node stop drills to observe cluster status and object read/write availability.
    Production environments also require additional items like regular business users with minimal permissions Policy, Prometheus monitoring, capacity alerts, node alerts, mc mirror backups, recovery drills, and Console access restrictions. Erasure Coding provides fault tolerance for disks and nodes but cannot replace backups.

---

## Twenty-one, Summary of This Article

This article completes a 4-node multi-disk MinIO distributed cluster deployment practice:

1. This article uses 10.0.0.41-10.0.0.44 as 4 MinIO nodes.
2. Each node uses 2 data directories.
3. All nodes use the same MinIO fixed version image.
4. All nodes must have identical server endpoint lists.
5. All nodes must have identical root users and passwords.
6. 9000 is the S3 API and node access port.
7. 9001 is the Web Console port.
8. mc alias should connect to the 9000 API port.
9. Buckets can be created, objects uploaded and downloaded via mc.
10. Cluster status can be viewed via mc admin info.
11. Live and ready status can be checked via curl health endpoint.
12. After stopping a single node, observe cluster status, read/write performance, and recovery.
13. Production environments should not directly expose HTTP 9000.
14. Production should provide HTTPS unified entry through Nginx / LB.
15. Erasure Coding is not equal to backup; further learning on backup migration and fault recovery is needed.

---

## Twenty-two, Reference Documents

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Docker Deployment Documentation:

    https://min.io/docs/minio/container/index.html

MinIO Multi-node Multi-disk Deployment Documentation:

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