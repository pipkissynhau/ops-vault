# MinIO Deployment Modes: Single Machine, Single Node Multi-Disk, and Multi-Node Cluster

Recommended path: 05-Storage/02-MinIO/02-MinIO Deployment Modes: Single Machine, Single Node Multi-Disk, and Multi-Node Cluster.md

Tags: #MinIO #DeploymentPattern #UnitDeployment #SingleNodeMultidisk #MultipointCluster #Docker #ErasureCoding #ObjectStorage #S3 #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the second article of the MinIO module, focusing on learning several common deployment modes of MinIO.

The previous article has completed:

- Object storage basic concepts
- Bucket / Object / Prefix
- S3 API
- AccessKey / SecretKey
- mc client basic operations
- Single-machine Docker minimal experiment
- Erasure Coding basic understanding

This article continues to advance toward deployment architecture, focusing on answering:

    What are the deployment modes of MinIO?
    What scenarios is a single-machine single-disk suitable for?
    What are the differences between single-node multi-disk and single-machine single-disk?
    Why is multi-node multi-disk closer to production?
    How should MinIO nodes, disks, and ports be planned?
    Why is it not recommended to deploy only a single machine in production?
    Why can internal node communication use HTTP, but external access must use HTTPS?
    Which deployment modes are just experimental, and which are suitable for production?

This article still emphasizes practicality, providing directly executable Docker commands.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the three deployment modes of MinIO: single-machine single-disk, single-node multi-disk, and multi-node multi-disk.
2. Start a single-machine single-disk MinIO using Docker.
3. Start a single-node multi-disk MinIO using Docker.
4. Understand that single-node multi-disk can verify Erasure Coding but cannot provide node-level high availability.
5. Understand that multi-node multi-disk is a more production-ready deployment mode.
6. Plan a 4-node MinIO distributed cluster's nodes, ports, and data directories.
7. Understand the difference between MinIO API port 9000 and Console port 9001.
8. Understand the architectural boundary between internal HTTP communication and external HTTPS access.
9. Understand the fault risks in different deployment modes.
10. Prepare for the next article "4-Node Multi-Disk Distributed Cluster Deployment Practice".

---

## III. Experimental Environment

### 3.1 Experimental Network Segment

This module's experiments use:

    10.0.0.0/24

Need to avoid existing addresses:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |

---

### 3.2 MinIO Node Planning

This article involves three experimental modes:

| Mode | Using Nodes | Purpose |
|---|---|---|
| Single-machine single-disk | 10.0.0.41 | Minimal startup, understanding API and Console |
| Single-node multi-disk | 10.0.0.41 | Understanding Erasure Coding and multiple data directories |
| Multi-node multi-disk | 10.0.0.41-10.0.0.44 | Understanding distributed cluster architecture |

Node planning:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.41 | minio-node01 | Single-machine / Distributed node 1 |
| 10.0.0.42 | minio-node02 | Distributed node 2 |
| 10.0.0.43 | minio-node03 | Distributed node 3 |
| 10.0.0.44 | minio-node04 | Distributed node 4 |
| 10.0.0.45 | minio-client | mc client, optional |
| 10.0.0.46 | minio-entry | Nginx / HTTPS unified entry, optional |

---

### 3.3 Operating System

Default:

    Ubuntu Server 22.04.5 LTS

Optional:

    Rocky Linux 9

This article defaults to Ubuntu Server 22.04.5 LTS.

---

### 3.4 Image Version

This article continues to use a fixed version image.

MinIO server:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

Source images:

    minio/minio:RELEASE.2025-04-22T22-12-26Z
    minio/mc:RELEASE.2025-04-16T18-13-26Z

Reason for choosing a fixed version:

    Avoid inconsistencies caused by the latest version changing, leading to inconsistent experimental commands, Web Console, log formats, and functional behavior.
    Facilitate the reproducibility of subsequent experiments, including single-machine, multi-disk, multi-node, Nginx, permissions, monitoring, backup, and migration.
    The mc client version is close in time to the MinIO server version, reducing the capability differences between client and server.

---

## IV. Overview of MinIO Deployment Modes

MinIO common deployment modes can be divided into three categories:

    Single-machine single-disk
    Single-node multi-disk
    Multi-node multi-disk

### 4.1 Single-machine single-disk

Structure:

    minio-node01
      |
      v
    /data/minio/data

Features:

    Simplest.
    Most suitable for beginners.
    Only needs one data directory.
    Can quickly understand S3 API, Console, and mc operations.
    No high availability.
    No true distributed fault tolerance.

Suitable for:

    Learning
    Development
    Temporary testing
    Feature verification

Not suitable for:

    Production
    Important data
    Long-term use by multiple people
    High availability object storage

---

### 4.2 Single-node multi-disk

Structure:

    minio-node01
      |
      |-- /data/minio/disk1
      |-- /data/minio/disk2
      |-- /data/minio/disk3
      |-- /data/minio/disk4

Features:

    Still only one node.
    Can simulate multiple disks.
    Can understand Erasure Coding.
    Can withstand partial disk failures, provided there are real multiple physical disks.
    Cannot withstand node failures.
    If multiple directories are actually on the same disk, cannot truly withstand disk failures.

Suitable for:

    Learning Erasure Coding
    Single-node multi-disk experiments
    Small-scale internal testing

Not suitable for: /think

High Availability Production  
Business requiring node-level fault tolerance  

---

### 4.3 Multi-node Multi-disk  

Structure:  

    minio-node01: /data1 /data2  
    minio-node02: /data1 /data2  
    minio-node03: /data1 /data2  
    minio-node04: /data1 /data2  

Features:  

    Multi-node.  
    Multi-disk.  
    Closer to production.  
    Can tolerate a certain number of disk or node failures.  
    Requires unified startup parameters.  
    Nodes need to communicate with each other.  
    RequiresPrefix Nginx / LB to provide a unified entry point.  
    Requires monitoring, alerts, backups, and disaster recovery drills.  

Suitable for:  

    Production object storage  
    Private S3  
    Attachments storage  
    Backup archives  
    Log archives  
    Artifact storage  

---

## Five, Comparison of Three Deployment Modes  

| Deployment Mode | Number of Nodes | Number of Disks | High Availability | Suitable for Production | Main Use |  
|---|---:|---:|---|---|---|  
| Single-node Single-disk | 1 | 1 | No | No | Entry-level, feature verification |  
| Single-node Multi-disk | 1 | Multiple | Limited disk-level fault tolerance | Not recommended generally | Understanding EC, multi-disk experiments |  
| Multi-node Multi-disk | Multiple | Multiple | Yes, depends on planning | Yes | Production object storage |  

Core conclusions:  

    Single-node Single-disk: Learn to use.  
    Single-node Multi-disk: Understand Erasure Coding.  
    Multi-node Multi-disk: Close to production.  

---

## Six, Port Planning  

### 6.1 MinIO Default Ports  

| Port | Purpose |  
|---|---|  
| 9000 | S3 API port |  
| 9001 | Web Console port |  
| 443 | External HTTPS unified entry point |  
| 80 | Optional, used for redirecting to HTTPS |  

---

### 6.2 Difference Between API Port and Console Port  

9000:  

    For applications, mc, aws cli, S3 SDK usage.  
    The API entry point for object upload, download, listing, and deletion.  

9001:  

    For administrators to access the Web Console via browser.  
    Used for managing Buckets, users, policies, and monitoring.  

Common errors:  

    Configuring 9001 as the S3 API port for mc or applications.  

Correct way:  

    mc alias set local http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'  

---

### 6.3 Production Entry Recommendations  

Production recommendations:  

    Internal MinIO nodes use HTTP 9000.  
    External clients use HTTPS 443.  
    Console 9001 should not be exposed to the public internet.  
    API and Console should be separated by domain or access source restrictions.  

Example:  

    https://s3.example.com       -> MinIO API 9000  
    https://minio-console.example.com -> MinIO Console 9001, restricted to maintenance network segments  

---

## Seven, Experiment One: Single-node Single-disk Deployment  

### 7.1 Experiment Objective  

Verify through single-node single-disk deployment:  

    MinIO container can start normally.  
    9000 API is accessible.  
    9001 Console is accessible.  
    mc can connect.  
    Bucket can be created.  
    Object can be uploaded and downloaded.  

---

### 7.2 Create Data Directory  

Execute on minio-node01:  

    mkdir -p /data/minio-single/data  

Check:  

    ls -ld /data/minio-single/data  

---

### 7.3 Start Single-node Single-disk MinIO  

Execute:  

    docker run -d \  
      --name minio-single \  
      --restart unless-stopped \  
      -p 9000:9000 \  
      -p 9001:9001 \  
      -e MINIO_ROOT_USER=minioadmin \  
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \  
      -v /data/minio-single/data:/data \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \  
      server /data --console-address ":9001"  

---

### 7.4 Check Container Status  

    docker ps | grep minio-single  

Check logs:  

    docker logs -f minio-single  

See API and Console information to confirm successful startup.  

---

### 7.5 Check Ports  

    ss -lntp | grep -E '9000|9001'  

Expected:  

    9000 is listening  
    9001 is listening  

---

### 7.6 Access Console  

Browser access:  

    http://10.0.0.41:9001  

Login:  

    Username: minioadmin  
    Password: MinioAdmin@123456  

---

### 7.7 Configure mc alias  

Create mc configuration directory:  

    mkdir -p /data/minio/mc-config  

Set alias:  

    docker run --rm \  
      -v /data/minio/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      alias set single http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'  

Check alias:  

    docker run --rm \  
      -v /data/minio/mc-config:/root/.mc \  
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \  
      alias list  

---

### 7.8 View Service Information

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  admin info single

---

### 7.9 Creating Bucket

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb single/single-demo

Viewing:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls single

---

### 7.10 Uploading and Downloading Objects

Create test file:

    mkdir -p /tmp/minio-single-demo

    echo "hello minio single mode" > /tmp/minio-single-demo/hello.txt

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-single-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/hello.txt single/single-demo/hello.txt

Viewing:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls single/single-demo

Download:

    rm -f /tmp/minio-single-demo/hello-download.txt

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-single-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp single/single-demo/hello.txt /demo/hello-download.txt

Verification:

    cat /tmp/minio-single-demo/hello-download.txt

---

### 7.11 Single-Node Single-Disk Risks

Single-node single-disk risks:

    Node failure, service unavailable.
    Disk damage, data may be lost.
    No node-level high availability.
    No real distributed fault tolerance.
    Not suitable for production critical data.

Single-node single-disk suitable for:

    Getting started
    Local development
    Feature verification
    mc operation practice
    S3 SDK debugging

---

## VIII. Experiment Two: Single-Node Multi-Disk Deployment

### 8.1 Experiment Objectives

Through single-node multi-disk deployment, understand:

    MinIO can use multiple data directories.
    Multiple directories can simulate multiple disks.
    MinIO organizes object data across multiple directories.
    You can initially understand Erasure Coding.
    Single-node multi-disk still cannot resolve node failure issues.

---

### 8.2 Stopping Single-Node Single-Disk Container

If 9000 / 9001 is occupied by minio-single, stop first:

    docker stop minio-single
    docker rm minio-single

Confirm port release:

    ss -lntp | grep -E '9000|9001'

---

### 8.3 Creating Multi-Disk Directories

Execute on minio-node01:

    mkdir -p /data/minio-multi-disk/disk1
    mkdir -p /data/minio-multi-disk/disk2
    mkdir -p /data/minio-multi-disk/disk3
    mkdir -p /data/minio-multi-disk/disk4

Viewing:

    ls -l /data/minio-multi-disk

Notes:

    Directories are used to simulate disks in the experiment.
    If these 4 directories are on the same physical disk, they can only be used for learning.
    Production should use real independent disks or independent mount points.

---

### 8.4 Starting Single-Node Multi-Disk MinIO

Execute:

    docker run -d \
      --name minio-multi-disk \
      --restart unless-stopped \
      -p 9000:9000 \
      -p 9001:9001 \
      -e MINIO_ROOT_USER=minioadmin \
      -e MINIO_ROOT_PASSWORD='MinioAdmin@123456' \
      -v /data/minio-multi-disk/disk1:/disk1 \
      -v /data/minio-multi-disk/disk2:/disk2 \
      -v /data/minio-multi-disk/disk3:/disk3 \
      -v /data/minio-multi-disk/disk4:/disk4 \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z \
      server /disk1 /disk2 /disk3 /disk4 --console-address ":9001"

---

### 8.5 Checking Container Status

    docker ps | grep minio-multi-disk

Viewing logs:

    docker logs -f minio-multi-disk

### 8.6 Configuring mc alias

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set multidisk http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

View service information:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin info multidisk

---

### 8.7 Creating Bucket and Uploading Objects

Create Bucket:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb multidisk/multidisk-demo

Create test file:

    mkdir -p /tmp/minio-multidisk-demo

    dd if=/dev/zero of=/tmp/minio-multidisk-demo/file-50m.bin bs=1M count=50

Upload:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-multidisk-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /demo/file-50m.bin multidisk/multidisk-demo/file-50m.bin

View:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls multidisk/multidisk-demo

---

### 8.8 Viewing Data Directory Distribution

Check on host machine:

    find /data/minio-multi-disk -maxdepth 4 -type f | head -100

Check directory size:

    du -sh /data/minio-multi-disk/disk*

Notes:

    You can observe how MinIO maintains data and metadata across multiple directories.
    Do not manually modify these directories.
    Do not delete files directly in data directories.
    Object operations must be completed through S3 API or mc.

---

### 8.9 Verifying Download

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-multidisk-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp multidisk/multidisk-demo/file-50m.bin /demo/file-50m-download.bin

Check:

    ls -lh /tmp/minio-multidisk-demo/file-50m-download.bin

---

### 8.10 Risks of Single-Node Multi-Disk

Single-node multi-disk can resolve some disk fault tolerance issues, but cannot resolve node failures.

Risks:

    Node downtime makes the entire service unavailable.
    OS damage makes the service unavailable.
    Docker anomalies make the service unavailable.
    Network disconnection makes the service unavailable.
    If multiple directories are on the same physical disk, disk failure may still cause overall data damage.
    Single-node mode is unsuitable for production high availability.

Production Understanding:

    Single-node multi-disk is a transitional mode for learning Erasure Coding.
    Production environments should use multi-node multi-disk.

---

## NineI don't know.Experiment Three: Multi-Node Multi-Disk Deployment Planning

### 9.1 Section Notes

This article only provides planning for multi-node multi-disk mode and does not fully expand all deployment commands.

Complete 4-node multi-disk distributed deployment will be detailed in the next article:

    03-MinIO Distributed Cluster Deployment: 4-Node Multi-Disk Experiment Practice.md

---

### 9.2 4-Node Planning

| IP | Hostname | Data Directory |
|---|---|---|
| 10.0.0.41 | minio-node01 | /data1, /data2 |
| 10.0.0.42 | minio-node02 | /data1, /data2 |
| 10.0.0.43 | minio-node03 | /data1, /data2 |
| 10.0.0.44 | minio-node04 | /data1, /data2 |

Notes:

    4 nodes.
    Each node has 2 data directories.
    Total of 8 data directories.
    Can be expanded to more disks per node later.

---

### 9.3 Distributed Startup Address Format

When starting MinIO in distributed mode, each node's server parameter must be consistent.

Example format:

    server http://minio-node{01...04}/data{1...2} --console-address ":9001"

Or expanded as:

    server \
      http://minio-node01/data1 http://minio-node01/data2 \
      http://minio-node02/data1 http://minio-node02/data2 \
      http://minio-node03/data1 http://minio-node03/data2 \
      http://minio-node04/data1 http://minio-node04/data2 \
      --console-address ":9001"

Notes:

    The cluster address list in each node's startup command must be consistent.
    Hostnames between nodes must be resolvable.
    Port 9000 must be reachable between nodes.
    Data directories must be pre-created and mounted.
    Do not use different address sets for different nodes.

---

### 9.4 Hosts Resolution Example

Each MinIO node is recommended to configure: /think

```bash
cat >> /etc/hosts <<'EOF'
10.0.0.41 minio-node01
10.0.0.42 minio-node02
10.0.0.43 minio-node03
10.0.0.44 minio-node04
EOF
```

**Verification:**

```bash
ping -c 2 minio-node01
ping -c 2 minio-node02
ping -c 2 minio-node03
ping -c 2 minio-node04
```

---

### 9.5 Network Connectivity Requirements

Nodes must communicate with each other:

```
9000 API / Cluster Communication Port
```

**Testing:**

```bash
nc -vz minio-node01 9000
nc -vz minio-node02 9000
nc -vz minio-node03 9000
nc -vz minio-node04 9000
```

If `nc` is not available:

```bash
apt install -y netcat-openbsd
```

---

### 9.6 Multi-Node Architecture Diagram

```
┌──────────────┐
│ minio-client │
└──────┬───────┘
       │
       │ S3 API
       v
┌────────────────────────────────────┐
│          Nginx / LB (Optional)       │
└──────────────┬─────────────────────┘
               │
               v
┌────────────────────────────────────┐
│          MinIO Distributed Cluster  │
│                                    │
│ minio-node01: /data1 /data2        │
│ minio-node02: /data1 /data2        │
│ minio-node03: /data1 /data2        │
│ minio-node04: /data1 /data2        │
└────────────────────────────────────┘
```

---

### 9.7 Why Multi-Node Multi-Disk is Better for Production

Multi-node multi-disk provides:

- Node-level fault tolerance
- Disk-level fault tolerance
- Better capacity scalability
- Better throughput
- Closer to object storage production architecture
- More suitable for前置 Nginx / LB
- Easier for monitoring and fault drills

But also introduces:

- Increased node planning complexity
- Increased network dependency
- Higher disk planning requirements
- More important fault recovery processes
- Monitoring and alerts must be complete
- Backup and migration must be planned

---

## Ten. Internal HTTP vs External HTTPS Strategy

### 10.1 Unified Strategy for This Module

MinIO module will uniformly adopt:

- Internal node communication uses HTTP.
- External client access uses HTTPS.
- Nginx / LB as a unified entry point.
- Backend proxy for MinIO 9000 API.
- Console 9001 is only exposed to operations or through independent entry points.

---

### 10.2 Why Internal Can Use HTTP

Internal HTTP is suitable under the following conditions:

- Nodes are in a trusted internal network.
- Network is not exposed to the public internet.
- Firewall or security group controls are in place.
- MinIO node communication does not cross untrusted networks.
- Operations can control access sources.

**Advantages:**

- Simple configuration.
- Reduces complexity of TLS certificate management.
- Reduces internal communication encryption/decryption overhead.
- Facilitates experimentation and troubleshooting.

---

### 10.3 Why External Must Use HTTPS

External access must use HTTPS, reasons:

- S3 requests involve AccessKey signing.
- Uploaded/downloaded data is business object data.
- External networks cannot be fully trusted.
- HTTP plaintext poses risks of interception and tampering.
- Production security and compliance typically require HTTPS.

**Recommended External Entry:**

```
https://s3.example.com
```

**Backend:**

```
http://minio-node01:9000
http://minio-node02:9000
http://minio-node03:9000
http://minio-node04:9000
```

---

### 10.4 Should Console Be Open to Public

Not recommended to directly open Console to the public internet.

**Recommendations:**

- Only allow access from operations network segments.
- Access via VPN / Bastion Host.
- Use independent domain names.
- Use HTTPS.
- Use strong passwords.
- Do not hand over root user to business users.

**Example:**

```
https://minio-console.example.com
```

**Restrictions:**

- Only allow access from operations network segments.

---

## Eleven. Deployment Mode Selection Recommendations

### 11.1 Learning Phase

**Recommended:**

- Single-node single-disk

**Goals:**

- Understand API, Console, mc, Bucket, Object.

---

### 11.2 Understanding Erasure Coding Phase

**Recommended:**

- Single-node multi-disk

**Goals:**

- Understand multiple data directories.
- Understand how MinIO organizes data.
- Understand the basic role of Erasure Coding.

**Note:**

- If multiple directories are on the same physical disk, only suitable for learning.

---

### 11.3 Near Production Phase

**Recommended:**

- 4-node multi-disk distributed cluster

**Goals:**

- Understand real distributed deployment.
- Validate node failures.
- Validate disk failures.
- Validate Nginx unified entry point.
- Validate mc client management.
- Validate monitoring and alerts.
- Validate backup and migration.

---

### 11.4 Production Phase

**Production Recommendations:**

- At least multi-node multi-disk.
- Use fixed versions.
- Use formal domain names and HTTPS.
- Clear boundaries for API and Console access.
- Business users use minimal privilege Policy.
- Prometheus monitoring.
- Capacity and node alerts.
- Regular backups.
- Regular recovery drills.
- Complete change and fault handling processes.

---

## Twelve. Pre-Deployment Checklist

### 12.1 Operating System Check

```
hostname
ip addr
ip route
timedatectl
df -hT
lsblk
free -h
```

**Requirements:**

- Clear hostname.
- Fixed IP.
- Time synchronization is normal.
- Data disks are mounted properly.
- System disk has sufficient capacity.

---

### 12.2 Docker Check

docker version
docker info
docker images | grep minio

If Docker is not installed, install Docker first.

---

### 12.3 Image Check

    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq/minio
    docker images | grep registry.cn-hangzhou.aliyuncs.com/pub-syq/mc

If not found:

    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
    docker pull registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

### 12.4 Port Check

    ss -lntp | grep -E '9000|9001'

If ports are occupied, handle them first.

---

### 12.5 Data Directory Check

Single machine single disk:

    ls -ld /data/minio-single/data

Single node multi-disk:

    ls -ld /data/minio-multi-disk/disk*

Multi-node multi-disk:

    ls -ld /data1 /data2

Production reminder:

    The data directory must be clearly defined.
    Do not mistakenly mount to the system disk.
    Do not place multiple "disk directories" actually on the same small system disk.
    Do not directly delete the data directory.

---

## Thirteen, Basic Troubleshooting Methods

### 13.1 Container Start Failure

Check:

    docker ps -a | grep minio
    docker logs <container-name>

Common causes:

- Ports are occupied
- Password does not meet requirements
- Data directory permission issues
- Start parameter errors
- Image does not exist
- Data directory contains incompatible data

---

### 13.2 Console Access Failure

Troubleshoot:

    docker ps | grep minio
    ss -lntp | grep 9001
    curl -I http://10.0.0.41:9001

Check:

    Firewall
    IP correctness
    Port mapping correctness
    Container restart status

---

### 13.3 mc Connection Failure

Troubleshoot:

    curl -I http://10.0.0.41:9000

Reconfigure alias:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set single http://10.0.0.41:9000 minioadmin 'MinioAdmin@123456'

Common causes:

- Mistakenly writing 9001 as API address
- AccessKey error
- SecretKey error
- Container not started
- 9000 port not accessible
- Firewall interception

---

### 13.4 Multi-node Mode Start Failure

Common causes:

- Hostname resolution between nodes fails
- Inconsistent start parameters
- Data directory missing on a node
- 9000 port not accessible on a node
- Time synchronization issues on a node
- Container not started on a node
- Firewall blocking
- Data directory previously used by an old cluster

Troubleshoot:

    ping minio-node01
    ping minio-node02
    nc -vz minio-node01 9000
    docker logs <container-name>
    docker ps -a
    timedatectl

---

## Fourteen, Production Environment Considerations

### 14.1 Do Not Use Single-machine Experiment as Production

Single-machine experiments are suitable for:

    Learning
    Feature verification
    Development integration testing

Not suitable for:

    Production attachment storage
    Production backup archiving
    Multi-business shared object storage
    Long-term storage of important data

---

### 14.2 Do Not Mount Data Directory to System Disk

Production must confirm:

    Data directory mounted to an independent data disk.
    Data disk capacity is sufficient.
    Data disk file system is stable.
    System disk will not be filled with object data.

Check:

    df -hT
    lsblk

---

### 14.3 Do Not Expose HTTP 9000 to Public Internet

Production external access should use:

    HTTPS 443

Do not expose directly:

    http://InternetIP:9000

Reasons:

    HTTP plaintext.
    Credentials and data are at risk.
    Not conducive to unified certificate and audit.
    Not conducive to future expansion of multi-node.

---

### 14.4 Do Not Let Business Use root User

Root user is only for management.

Business should use:

    Independent user
    Independent AccessKey
    Minimum permission Policy
    Can be individually rotated
    Can be individually disabled

---

### 14.5 Do Not Manually Modify Data Directory

Prohibited:

    Manually delete object files in data directory.
    Manually move files in data directory.
    Manually modify MinIO metadata.
    Copy part of data directory to other nodes without understanding consequences.

Object operations should be performed through:

    mc
    S3 API
    MinIO Console
    Official tools

---

## Fifteen, Clean Up Test Environment

### 15.1 Clean Up Single-machine Single-disk

Stop container:

    docker stop minio-single
    docker rm minio-single

Delete test data directory:

    rm -rf /data/minio-single

---

### 15.2 Clean Up Single-node Multi-disk

Stop container:

    docker stop minio-multi-disk
    docker rm minio-multi-disk

Delete test data directory:

    rm -rf /data/minio-multi-disk

---

### 15.3 Clean Up mc Configuration

If confirmed no longer needed:

    rm -rf /data/minio/mc-config

High-risk warning:

    Deleting data directory will delete object data.
    Only allowed to execute in test environment.
    Production environment is prohibited to directly rm -rf data directory.

---

## Sixteen, Interview Answer Approach

If asked in an interview:

    What are the deployment modes of MinIO? How to deploy in production?

You can answer:

# Common MinIO Deployment Patterns

MinIO common deployment patterns can be divided into single-node single-disk, single-node multi-disk, and multi-node multi-disk.  
Single-node single-disk is the simplest, suitable for learning, development, and functional verification, but lacks high availability. Node or disk failures can lead to service unavailability or data loss.  
Single-node multi-disk allows MinIO to use multiple data directories or multiple disks, helping understand Erasure Coding and providing some tolerance for disk failures. However, it still cannot resolve node failures, so it is generally not recommended as a production high availability solution.  
For production, a multi-node multi-disk distributed deployment is recommended, such as 4 nodes with multiple data disks per node. This provides disk and node-level fault tolerance through Erasure Coding, with a unified HTTPS entry provided by Nginx or LB.  
In production, I would run MinIO nodes on a trusted internal network, using HTTP for internal communication to reduce TLS management complexity and overhead. However, external client access must use HTTPS to protect S3 credentials and object data. The Console is not recommended to be exposed to the public. Business should not use root users, but instead use independent AccessKey and minimal permissions Policy.  
Beyond deployment itself, monitoring nodes, disks, bucket capacity, object count, and API error rate is required. Backup and migration can be done via mc mirror or cross-cluster solutions. Erasure Coding cannot replace backups.

---

## 17. Summary of This Article

This article completes the study of MinIO deployment patterns:

1. Common MinIO deployment patterns include single-node single-disk, single-node multi-disk, and multi-node multi-disk.  
2. Single-node single-disk is suitable for learning and functional verification, but not for production.  
3. Single-node multi-disk helps understand Erasure Coding but cannot resolve node failures.  
4. Multi-node multi-disk is closer to production deployment.  
5. 9000 is the S3 API port, and 9001 is the Web Console port.  
6. mc should connect to the 9000 API port, not the 9001 Console port.  
7. External access in production must use HTTPS.  
8. Internal node communication can use HTTP in a trusted network.  
9. The Console is not recommended to be exposed to the public.  
10. Business should not use root users.  
11. Data directories should not be deleted or manually modified arbitrarily.  
12. In multi-node mode, each node's startup parameters must be consistent.  
13. Multi-node mode depends on hostname resolution, network connectivity, time synchronization, and data directory planning.  
14. MinIO's Erasure Coding is not equivalent to backups.  
15. The next article will enter the practice of deploying a 4-node multi-disk distributed MinIO cluster.

---

## 18. Reference Documents

MinIO official documentation:  

    https://min.io/docs/minio/linux/index.html  

MinIO Docker deployment documentation:  

    https://min.io/docs/minio/container/index.html  

MinIO distributed deployment documentation:  

    https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html  

MinIO Erasure Coding documentation:  

    https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html  

MinIO mc client documentation:  

    https://min.io/docs/minio/linux/reference/minio-mc.html  

MinIO user and permissions documentation:  

    https://min.io/docs/minio/linux/administration/identity-access-management.html  

AWS S3 API documentation:  

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html