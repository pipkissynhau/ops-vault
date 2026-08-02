# RustFS Cluster Deployment Practice: Multi-node, Multi-disk, and Access Entry

Recommended path: 05-Storage/04-RustFS/04-RustFS Cluster Deployment Practice: Multi-node, Multi-disk, and Access Entry.md

Tags: #RustFS #ObjectStorage #S3 #DistributiveObjectStorage #Docker #Multinodes #Multidisk #ErasureCoding #Nginx #ReverseAgent #UnifiedEntrance #HTTPS #mc #AdvancedSre #ProductionTransport

---

## I. Document Notes

This is the fourth article of the RustFS module, focusing on completing the RustFS multi-node, multi-disk cluster deployment practice.

Previously completed:

    01-RustFS Basics: S3-compatible object storage and use cases
    02-RustFS Deployment Modes: Single-node mode and cluster mode understanding
    03-RustFS Single-node Deployment Practice: Service startup, data directory, and access verification

This article enters the RustFS cluster deployment practice, focusing on:

    Multi-node deployment planning
    Multi-disk directory planning
    Host name resolution configuration
    Time synchronization check
    Data directory preparation
    XFS/ext4 file system planning
    RustFS Docker image using fixed version
    4-node RustFS cluster startup
    Node service status check
    S3 API health check
    mc client access verification
    Bucket creation
    Object upload and download
    Multi-node data directory observation
    Single-node failure observation
    Single-node recovery observation
    Nginx unified entry configuration
    Accessing RustFS through unified entry
    Cluster mode common issue troubleshooting
    Production boundary notes

This article's deployment mode is:

    MNMD: Multiple Node Multiple Disk
    Multi-node multi-disk mode

This article emphasizes:

    Docker multi-node cluster experiment is used to understand RustFS cluster startup, access, and operation methods.
    Not equal to a complete production architecture.
    Before production, must continue to complete performance stress testing, S3 compatibility verification, fault recovery verification, permission governance, HTTPS, security audit, and backup migration verification.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Plan the RustFS multi-node cluster experiment environment.
2. Understand the RustFS MNMD multi-node multi-disk mode.
3. Understand why cluster mode requires unified host name resolution.
4. Understand why multi-node object storage must ensure time synchronization.
5. Prepare RustFS data directories on 4 nodes.
6. Start multiple nodes using a fixed version RustFS image.
7. Configure RUSTFS_VOLUMES to point to multiple nodes and multiple data directories.
8. View the status and logs of each RustFS container.
9. Verify the health of each node's S3 API.
10. Access the cluster via mc through any node.
11. Create a Bucket.
12. Upload and download Objects.
13. Verify the consistency of uploaded/downloaded files.
14. Configure a unified access entry via Nginx.
15. Allow clients to access only through the unified entry, not directly multiple backend nodes.
16. Understand the design boundary between internal HTTP and external HTTPS.
17. Simulate single-node stoppage and observe access impact.
18. Recover nodes and observe cluster status.
19. Troubleshoot common issues like ports, DNS, permissions, time synchronization, directory mounting, keys, and reverse proxy.
20. Lay the foundation for subsequent RustFS vs MinIO comparison, client access, security, and operation troubleshooting.

---

## III. Official Deployment Key Points Understanding

RustFS multi-node multi-disk mode needs to focus on:

    Multiple nodes
    Multiple disks
    Host name continuity
    Node network
    Time synchronization
    Unified port
    Capacity planning
    Disk planning
    File system planning
    Erasure Coding
    Access entry

Official documentation typically uses similar:

    RUSTFS_VOLUMES="http://node{1...4}:9000/data/rustfs{0...3}"

This format represents:

    node1 to node4, 4 nodes
    Each node has rustfs0 to rustfs3, 4 data directories
    Total 16 data paths
    RustFS combines these data paths into a distributed object storage cluster

This article's experiment adopts the same concept, but with current experimental network planning:

    rustfs-node01 to rustfs-node04
    Each node /data/rustfs0 to /data/rustfs3

Final RUSTFS_VOLUMES planning is:

    http://rustfs-node{01...04}:9000/data/rustfs{0...3}

---

## IV. Experiment Environment Planning

### 4.1 Node Planning

RustFS Cluster Nodes:

| IP | Hostname | Role | Data Directory |
|---|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 | /data/rustfs0-3 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 | /data/rustfs0-3 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 | /data/rustfs0-3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 | /data/rustfs0-3 |

Client Node:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.55 | rustfs-client | mc / AWS CLI Test Node |

Unified Entry Node:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.56 | rustfs-entry | Nginx / LB Unified Entry |

---

### 4.2 Operating System

Default experimental system:

    Ubuntu Server 22.04.5 LTS

Optional systems:

    Rocky Linux 9

This article primarily uses Ubuntu Server 22.04.5 LTS.

---

### 4.3 RustFS Version

This article continues using the fixed version from 03:

    rustfs/rustfs:1.0.0-alpha.99

User's Alibaba Cloud mirror repository address:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

Version selection notes:

    Fixed version facilitates experiment reproducibility.
    Avoids issues caused by latest version changes in commands, parameters, features, or interfaces.
    Current version is still in alpha stage, suitable for learning and verification.
    Not recommended for direct use in production core business.
    Production version selection must be combined with official Release Notes, S3 compatibility, performance stress testing, security patches, upgrade strategy, and recovery drills.

---

### 4.4 Port Planning

| Port | Purpose | Exposure Recommendation |
|---|---|---|
| 9000 | RustFS S3 API | Internal network access, unified entry reverse proxy |
| 9001 | RustFS Console | Access only from maintenance network segments |
| 80 | Nginx HTTP | Optional, for redirecting to HTTPS or experimental use |
| 443 | Nginx HTTPS | External client unified access entry |
| 22 | SSH | Access only from maintenance sources |

This section covers:

    RustFS backend nodes: 10.0.0.51-54:9000
    RustFS Console: 10.0.0.51-54:9001
    Nginx unified entry: http://10.0.0.56:9000

HTTPS full certificate configuration will be expanded in 07-RustFS Permissions and Security.

---

### 4.5 Data Directory Planning

Each RustFS node creates 4 data directories:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

Container mount paths remain consistent:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

Experimental environment notes:

    If the virtual machine lacks 4 independent data disks, 4 directories can be used as simulation.
    This can verify cluster startup, S3 API, Bucket, Object, and unified entry.
    However, this cannot realistically simulate 4 independent disk failures.
    Production environments must use independent data disks; it's not recommended to place multiple data directories on the same system disk.

Production recommendations:

    Each rustfs data directory corresponds to an independent disk or mount point.
    Each disk uses XFS or ext4.
    Officially, XFS is preferred.
    Each node's disk count and capacity should be as consistent as possible.
    Each node's disk performance should be as consistent as possible.
    Do not use NFS as theBottom data directory for RustFS.
    Do not store production object data directly on the system disk.

---

## FiveI don't know.Cluster Topology

### 5.1 Backend Cluster Topology

    ┌───────────────────────────────────────────┐
    │ rustfs-node01                             │
    │ 10.0.0.51                                 │
    │ /data/rustfs0 /data/rustfs1               │
    │ /data/rustfs2 /data/rustfs3               │
    └───────────────────────────────────────────┘

    ┌───────────────────────────────────────────┐
    │ rustfs-node02                             │
    │ 10.0.0.52                                 │
    │ /data/rustfs0 /data/rustfs1               │
    │ /data/rustfs2 /data/rustfs3               │
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

```
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
│ Nginx Unified Entry Point     │
└───────────────┬───────────────┘
                │
                │ HTTP Internal Forwarding
                v
┌──────────────────────────────────────────────┐
│ RustFS Backends                              │
│ 10.0.0.51:9000                               │
│ 10.0.0.52:9000                               │
│ 10.0.0.53:9000                               │
│ 10.0.0.54:9000                               │
└──────────────────────────────────────────────┘

---

## Six. Pre-deployment Checks

### 6.1 Check Host Information for All RustFS Nodes

Execute on 10.0.0.51-10.0.0.54 respectively:

    hostname
    hostname -I
    cat /etc/os-release
    uname -a
    timedatectl

Requirements:

    Hostname is correct.
    IP address is correct.
    Time synchronization is normal.
    Operating system matches expectations.

---

### 6.2 Configure Hostnames

Execute on each node respectively:

10.0.0.51:

    hostnamectl set-hostname rustfs-node01

10.0.0.52:

    hostnamectl set-hostname rustfs-node02

10.0.0.53:

    hostnamectl set-hostname rustfs-node03

10.0.0.54:

    hostnamectl set-hostname rustfs-node04

10.0.0.55:

    hostnamectl set-hostname rustfs-client

10.0.0.56:

    hostnamectl set-hostname rustfs-entry

---

### 6.3 Configure hosts Resolution

Execute on all RustFS nodes, client node, and entry node:

    cat >> /etc/hosts <<'EOF'
    10.0.0.51 rustfs-node01
    10.0.0.52 rustfs-node02
    10.0.0.53 rustfs-node03
    10.0.0.54 rustfs-node04
    10.0.0.55 rustfs-client
    10.0.0.56 rustfs-entry
    10.0.0.56 s3.rustfs.local
    EOF

Check:

    getent hosts rustfs-node01
    getent hosts rustfs-node02
    getent hosts rustfs-node03
    getent hosts rustfs-node04
    getent hosts s3.rustfs.local

---

### 6.4 Check Network Between Nodes

Execute on each RustFS node:

    ping -c 3 rustfs-node01
    ping -c 3 rustfs-node02
    ping -c 3 rustfs-node03
    ping -c 3 rustfs-node04

Check port connectivity after service startup:

    nc -vz rustfs-node01 9000
    nc -vz rustfs-node02 9000
    nc -vz rustfs-node03 9000
    nc -vz rustfs-node04 9000

If no nc is available:

    apt update
    apt install -y netcat-openbsd

---

### 6.5 Check Time Synchronization

Execute on all nodes:

    timedatectl

Focus on:

    System clock synchronized: yes

If not synchronized, Ubuntu can execute:

    apt update
    apt install -y systemd-timesyncd
    systemctl enable --now systemd-timesyncd
    timedatectl set-ntp true
    timedatectl

Notes:

    S3 signing depends on time.
    Multi-node object storage also requires consistent node time.
    Time desynchronization may cause authentication failures, service startup anomalies, and log troubleshooting difficulties.

---

### 6.6 Check Docker

Execute on 10.0.0.51-10.0.0.54:

    docker version
    docker info
    systemctl status docker --no-pager

If Docker is not installed, refer to the Docker installation method in section 03.

---

### 6.7 Check Port Occupancy

Execute on each RustFS node:

    ss -lntp | grep ':9000' || true
    ss -lntp | grep ':9001' || true

If a service is already using 9000 or 9001:

    Confirm the process first.
    Do not kill blindly.
    You can change the port, but all RustFS nodes must maintain the same listening port.
    This document defaults to using 9000 and 9001.
```

## VII. Data Disk and Directory Preparation

### 7.1 Experimental Directory Approach

If the current virtual machine does not have an independent data disk, first create 4 directories:

Execute on each RustFS node:

    mkdir -p /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Check:

    ls -ld /data/rustfs*
    df -hT /data/rustfs0
    df -hT /data/rustfs1
    df -hT /data/rustfs2
    df -hT /data/rustfs3

Notes:

    This approach is for experimentation.
    Cannot replace production independent disk planning.

---

### 7.2 Production Independent Disk Approach

If the node has 4 independent data disks, assume:

    /dev/sdb
    /dev/sdc
    /dev/sdd
    /dev/sde

High-risk warning:

    mkfs will clear the disk.
    Must confirm the device name before execution.
    Do not mistakenly format the system disk.
    Production must have dual-person verification.

Check disks:

    lsblk -f

Format as XFS example:

    mkfs.xfs -f -i size=512 -n ftype=1 -L RUSTFS0 /dev/sdb
    mkfs.xfs -f -i size=512 -n ftype=1 -L RUSTFS1 /dev/sdc
    mkfs.xfs -f -i size=512 -n ftype=1 -L RUSTFS2 /dev/sdd
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

### 7.3 Set Directory Permissions

RustFS Docker container typically uses UID 10001.

Execute on each RustFS node:

    chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Check:

    ls -ld /data/rustfs*

Not recommended:

    chmod -R 777 /data/rustfs*

---

## VIII. Image Preparation

### 8.1 Set Image Variables

Execute on each RustFS node:

    export RUSTFS_VERSION="1.0.0-alpha.99"
    export RUSTFS_IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:${RUSTFS_VERSION}"

Check:

    echo ${RUSTFS_IMAGE}

---

### 8.2 Pull Image

Execute:

    docker pull ${RUSTFS_IMAGE}

Check:

    docker images | grep rustfs

If the image does not exist, first pull from the official image on a machine with internet access and synchronize to the Aliyun registry:

    docker pull rustfs/rustfs:1.0.0-alpha.99
    docker tag rustfs/rustfs:1.0.0-alpha.99 registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99
    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

---

## IX. RustFS Cluster Startup Parameter Planning

### 9.1 Unified Variables

All RustFS nodes use the same variables:

    RUSTFS_ACCESS_KEY=rustfsadmin
    RUSTFS_SECRET_KEY=RustFSAdmin@123456
    RUSTFS_ADDRESS=:9000
    RUSTFS_CONSOLE_ADDRESS=:9001
    RUSTFS_CONSOLE_ENABLE=true
    RUSTFS_VOLUMES=http://rustfs-node{01...04}:9000/data/rustfs{0...3}

Explanation:

    rustfs-node{01...04} represents rustfs-node01 to rustfs-node04.
    rustfs{0...3} represents rustfs0 to rustfs3.
    Total 4 nodes x 4 data directories = 16 data paths.
    All nodes must use the same RUSTFS_VOLUMES.
    All nodes must resolve rustfs-node01 to rustfs-node04.
    All nodes must use the same port 9000.

---

### 9.2 Key Explanation

Experimental keys in this document:

    AccessKey: rustfsadmin
    SecretKey: RustFSAdmin@123456

Production reminder:

    Do not use experimental keys.
    Do not commit keys to Git.
    Do not share administrator keys across multiple services.
    Production requires service-level AccessKey and permission governance.
    Management keys require rotation mechanisms.

---

## X. Start RustFS Cluster Container

### 10.1 Clean Up Old Containers on All RustFS Nodes Before Execution

Execute on 10.0.0.51-10.0.0.54:

    docker rm -f rustfs-cluster || true

---

### 10.2 Start Container on All RustFS Nodes

Execute the same command on each node of 10.0.0.51-10.0.0.54:

docker run -d \
  --name rustfs-cluster \
  --restart=always \
  -p 9000:9000 \
  -p 9001:9001 \
  -v /data/rustfs0:/data/rustfs0 \
  -v /data/rustfs1:/data/rustfs1 \
  -v /data/rustfs2:/data/rustfs2 \
  -v /data/rustfs3:/data/rustfs3 \
  -e RUSTFS_ACCESS_KEY="rustfsadmin" \
  -e RUSTFS_SECRET_KEY="RustFSAdmin@123456" \
  -e RUSTFS_ADDRESS=":9000" \
  -e RUSTFS_CONSOLE_ADDRESS=":9001" \
  -e RUSTFS_CONSOLE_ENABLE="true" \
  -e RUSTFS_VOLUMES="http://rustfs-node{01...04}:9000/data/rustfs{0...3}" \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 \
  http://rustfs-node{01...04}:9000/data/rustfs{0...3}

**Notes:**

- Each node command remains consistent.
- Each node knows the complete cluster topology.
- Container paths must match the paths in RUSTFS_VOLUMES.
- If a node cannot resolve hosts, the cluster may fail to start.
- If a node's port is unreachable, the cluster may fail to start or become unhealthy.
- If a data directory has insufficient permissions, the container may report "permission denied".

---

### 10.3 Check Container Status

Execute on each node:

    docker ps | grep rustfs-cluster
    docker ps -a | grep rustfs-cluster

**Expected:**

    rustfs-cluster status is Up

---

### 10.4 Check Logs

Execute on each node:

    docker logs rustfs-cluster --tail=100

Continuous monitoring:

    docker logs -f rustfs-cluster

**Focus on:**

- Whether successfully listening on 9000.
- Whether Console is enabled.
- Whether able to connect to other nodes.
- Whether VolumeNotFound occurs.
- Whether permission denied occurs.
- Whether address already in use occurs.
- Whether hostname resolution failure occurs.
- Whether time synchronization issues occur.
- Whether cluster initialization anomalies occur.

---

### 10.5 Check Ports

Execute on each RustFS node:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

---

## Eleven. Health Check

### 11.1 Local Check on Each Node

Execute on each RustFS node:

    curl -i http://127.0.0.1:9000/health

**Expected:**

    HTTP/1.1 200 OK

---

### 11.2 Check Each Node from Client

Execute on rustfs-client node:

    curl -i http://10.0.0.51:9000/health
    curl -i http://10.0.0.52:9000/health
    curl -i http://10.0.0.53:9000/health
    curl -i http://10.0.0.54:9000/health

If failed, troubleshoot:

- Whether the container is running.
- Whether the port is listening.
- Whether firewall rules allow traffic.
- Whether client-to-backend network is reachable.
- Whether hosts are properly configured.
- Whether RustFS logs report errors.

---

### 11.3 Console Access

Access any node's Console:

    http://10.0.0.51:9001/rustfs/console
    http://10.0.0.52:9001/rustfs/console
    http://10.0.0.53:9001/rustfs/console
    http://10.0.0.54:9001/rustfs/console

**Login:**

- AccessKey: rustfsadmin
- SecretKey: RustFSAdmin@123456

**Notes:**

- During experiments, you can directly access node Console.
- In production, it's not recommended to expose multiple Consoles.
- In production, the management entry should be restricted to specific sources and use HTTPS, authentication, and auditing.

---

## Twelve. mc Client Access Verification

### 12.1 Prepare mc Configuration Directory

Execute on rustfs-client or any test node:

    mkdir -p /data/rustfs/mc-config

Use Docker version of mc:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --version

---

### 12.2 Configure alias via Single Backend Node

First test via rustfs-node01:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-node01 http://10.0.0.51:9000 rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias list

---

### 12.3 Create Bucket

Create Bucket:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      mb rustfs-node01/cluster-demo

Check: /think

docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-node01

Expected output:

    cluster-demo

---

### 12.4 Upload Test Files

Prepare test directory:

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

Create large file:

    dd if=/dev/zero of=file-100m.bin bs=1M count=100

Calculate checksums:

    sha256sum hello.txt create-time.txt config/app.conf logs/app.log file-100m.bin > sha256-before.txt
    cat sha256-before.txt

Upload:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-cluster-test:/test \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp --recursive /test/ rustfs-node01/cluster-demo/

Check:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-node01/cluster-demo

---

### 12.5 Read Objects from Another Node

Configure rustfs-node02 alias:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-node02 http://10.0.0.52:9000 rustfsadmin 'RustFSAdmin@123456'

View same Bucket via node02:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-node02/cluster-demo

If node02 can see data created and uploaded by node01, it indicates:

    The nodes belong to the same cluster.
    It's not multiple independent single-node object storages.
    Data and metadata are accessible across nodes at the cluster level.

---

### 12.6 Download Verification

Create download directory:

    mkdir -p /tmp/rustfs-cluster-download

Download large file:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-cluster-download:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs-node02/cluster-demo/file-100m.bin /download/file-100m.bin

Download small files:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp/rustfs-cluster-download:/download \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs-node02/cluster-demo/hello.txt /download/hello.txt

Verify:

    sha256sum /tmp/rustfs-cluster-test/file-100m.bin /tmp/rustfs-cluster-download/file-100m.bin
    sha256sum /tmp/rustfs-cluster-test/hello.txt /tmp/rustfs-cluster-download/hello.txt

Expected output:

    Hashes match.

---

## Thirteen. Observing Data Directories

### 13.1 Check Data Directory Capacity on Each Node

Execute on each RustFS node:

    du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    df -hT /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

---

### 13.2 View Directory Structure

Execute on each RustFS node:

find /data/rustfs0 -maxdepth 3 -type d | head -50
    find /data/rustfs1 -maxdepth 3 -type d | head -50
    find /data/rustfs2 -maxdepth 3 -type d | head -50
    find /data/rustfs3 -maxdepth 3 -type d | head -50

High-risk warning:

    You can observe the directory structure.
    Do not manually modify.
    Do not manually delete.
    Do not directly rm -rf /data/rustfs*.
    Object deletion should be done via S3 API, mc, or Console.

---

## 14. Configuring Nginx as Unified Entry Point

### 14.1 Unified Entry Point Explanation

Clients are not recommended to access multiple backend nodes directly.

Recommended unified entry point:

    rustfs-client -> rustfs-entry -> RustFS backend nodes

Benefits of a unified entry point:

    Simplified client configuration.
    Backend nodes can be replaced.
    Unified HTTPS support.
    Unified access logs.
    Unified upload size limits.
    Unified timeout parameters.
    Unified source restrictions.
    Hiding backend nodes.

---

### 14.2 Installing Nginx on rustfs-entry

Execute on 10.0.0.56:

    apt update
    apt install -y nginx

Start:

    systemctl enable --now nginx

Check:

    systemctl status nginx --no-pager
    nginx -v

---

### 14.3 Configuring Nginx HTTP Unified Entry Point

Create configuration file:

    cat > /etc/nginx/conf.d/rustfs-s3.conf <<'EOF'
    upstream rustfs_s3_backend {
        least_conn;
        server 10.0.0.51:9000 max_fails=3 fail_timeout=30s;
        server 10.0.0.52:9000 max_fails=3 fail_timeout=30s;
        server 10.0.0.53:9000 max_fails=3 fail_timeout=30s;
        server 10.0.0.54:9000 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 9000;
        server_name s3.rustfs.local;

        client_max_body_size 0;
        proxy_request_buffering off;
        proxy_buffering off;

        proxy_connect_timeout 60s;
        proxy_send_timeout 3600s;
        proxy_read_timeout 3600s;
        send_timeout 3600s;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_http_version 1.1;
            proxy_set_header Connection "";

            proxy_pass http://rustfs_s3_backend;
        }
    }
    EOF

Check configuration:

    nginx -t

Reload:

    systemctl reload nginx

Check port:

    ss -lntp | grep ':9000'

---

### 14.4 Health Check via Unified Entry Point

Execute on rustfs-client:

    curl -i http://10.0.0.56:9000/health
    curl -i http://s3.rustfs.local:9000/health

Expected:

    HTTP/1.1 200 OK

---

### 14.5 Configuring mc to Use Unified Entry Point

Execute on rustfs-client:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set rustfs-entry http://s3.rustfs.local:9000 rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-entry

Expected:

    You should see cluster-demo.

---

### 14.6 Uploading Objects via Unified Entry Point

Create file:

    echo "upload through nginx entry" > /tmp/rustfs-entry-test.txt

Upload:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /tmpdata/rustfs-entry-test.txt rustfs-entry/cluster-demo/rustfs-entry-test.txt

Check: /think

docker run --rm \
  -v /data/rustfs/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  ls rustfs-entry/cluster-demo

---

### 14.7 Nginx Log Inspection

Execute in rustfs-entry:

    tail -f /var/log/nginx/access.log

Run access in another terminal:

    curl -i http://s3.rustfs.local:9000/health

Check error logs:

    tail -f /var/log/nginx/error.log

---

## FifteenI don't know.HTTPS Entry Explanation

This document temporarily uses:

    http://s3.rustfs.local:9000

Later 07 will be upgraded to:

    https://s3.rustfs.local

Production requirements:

    External clients must use HTTPS.
    AccessKey / SecretKey should not be transmitted in plaintext HTTP over untrusted networks.
    Internal HTTP is only suitable for trusted networks.
    Backend 9000 port should not be exposed to the public.
    Console management entry should restrict sources.

HTTPS will cover:

    Certificate path
    Nginx 443
    HTTP to HTTPS redirect
    mc --insecure or trusted CA
    Certificate expiration monitoring
    Upload size and timeout configuration
    Management entry isolation

---

## SixteenI don't know.Single Node Failure Observation

### 16.1 Fault Simulation Explanation

This section is used to observe whether the unified entry and cluster access remain available after a backend node stops.

High-risk warning:

    Only execute in experimental environments.
    Do not arbitrarily stop nodes in production environments.
    Must confirm backups, maintenance window, impact scope, and recovery plan before production fault simulation.

---

### 16.2 Stop a RustFS Node

Execute in rustfs-node04:

    docker stop rustfs-cluster

Check:

    docker ps -a | grep rustfs-cluster

---

### 16.3 Access Unified Entry from Client

Execute in rustfs-client:

    curl -i http://s3.rustfs.local:9000/health

Check Bucket:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls rustfs-entry

Attempt to download object:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs-entry/cluster-demo/hello.txt /tmpdata/hello-after-node-stop.txt

Check:

    cat /tmp/hello-after-node-stop.txt

---

### 16.4 Observe Nginx Backend Status

Check Nginx error logs:

    tail -100 /var/log/nginx/error.log

Notes:

    Nginx may record node04 connection failure.
    However, as long as other backends are available, the unified entry may continue to serve.
    Whether it is fully readable/writable depends on the current cluster status, EC, number of failures, and object distribution in the RustFS version.

---

### 16.5 Recover Node

Execute in rustfs-node04:

    docker start rustfs-cluster

Check logs:

    docker logs rustfs-cluster --tail=100

Verify in client:

    curl -i http://10.0.0.54:9000/health
    curl -i http://s3.rustfs.local:9000/health

Upload object again:

    echo "after node04 recovery" > /tmp/after-node04-recovery.txt

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp /tmpdata/after-node04-recovery.txt rustfs-entry/cluster-demo/after-node04-recovery.txt

---

## SeventeenI don't know.Cluster Restart Verification

### 17.1 Restart Containers Sequentially

Execute sequentially on each RustFS node:

    docker restart rustfs-cluster
    docker logs rustfs-cluster --tail=50
    curl -i http://127.0.0.1:9000/health

Recommendations:

    Restart one node at a time.
    Restart the next node only after confirming recovery.
    Not recommended to restart all nodes simultaneously.

---

### 17.2 Verify Data Still Exists on Client

Execute in rustfs-client:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      ls --recursive rustfs-entry/cluster-demo

Download file and verify:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      cp rustfs-entry/cluster-demo/file-100m.bin /tmpdata/file-100m-after-restart.bin

sha256sum /tmp/rustfs-cluster-test/file-100m.bin /tmp/file-100m-after-restart.bin

---

## Eighteen. Common Issues in Cluster Deployment Troubleshooting

### 18.1 Container Exits Immediately After Start

Troubleshooting:

    docker ps -a | grep rustfs-cluster
    docker logs rustfs-cluster --tail=200

Common Causes:

    RUSTFS_VOLUMES is written incorrectly.
    Hostname resolution failure.
    Network connectivity between nodes is broken.
    A node's port is unreachable.
    Data directory does not exist.
    Data directory has insufficient permissions.
    Port 9000 is occupied.
    Node time is out of sync.
    Image versions are inconsistent.

Resolution:

    getent hosts rustfs-node01
    getent hosts rustfs-node02
    getent hosts rustfs-node03
    getent hosts rustfs-node04
    ping -c 3 rustfs-node01
    ss -lntp | grep ':9000'
    ls -ld /data/rustfs*
    chown -R 10001:10001 /data/rustfs*
    timedatectl

---

### 18.2 VolumeNotFound

If the log shows:

    VolumeNotFound

Key Checks:

    Whether the paths in RUSTFS_VOLUMES match the mount paths inside the container.
    Whether the host directory is mounted to the same path inside the container.
    Whether /data/rustfs0 to /data/rustfs3 all exist.
    Whether the number of directories per node is consistent.
    Whether the startup commands for each node are consistent.

Correct mounting should include:

    -v /data/rustfs0:/data/rustfs0
    -v /data/rustfs1:/data/rustfs1
    -v /data/rustfs2:/data/rustfs2
    -v /data/rustfs3:/data/rustfs3

RUSTFS_VOLUMES should include:

    http://rustfs-node{01...04}:9000/data/rustfs{0...3}

---

### 18.3 Nodes Cannot Access Each Other

Troubleshooting:

    ping rustfs-node01
    ping rustfs-node02
    ping rustfs-node03
    ping rustfs-node04

After service startup:

    nc -vz rustfs-node01 9000
    nc -vz rustfs-node02 9000
    nc -vz rustfs-node03 9000
    nc -vz rustfs-node04 9000

Check firewall:

Ubuntu:

    ufw status

Rocky:

    firewall-cmd --list-all

Temporary allow example:

    ufw allow 9000/tcp
    ufw allow 9001/tcp

Rocky:

    firewall-cmd --zone=public --add-port=9000/tcp --permanent
    firewall-cmd --zone=public --add-port=9001/tcp --permanent
    firewall-cmd --reload

---

### 18.4 mc Access Failure

Troubleshooting:

    curl -i http://s3.rustfs.local:9000/health
    curl -i http://10.0.0.51:9000/health
    docker logs rustfs-cluster --tail=100

Common Causes:

    AccessKey is incorrect.
    SecretKey is incorrect.
    Endpoint is written incorrectly.
    Nginx forwarding is abnormal.
    Time is out of sync.
    Bucket does not exist.
    S3 compatibility issues.

---

### 18.5 Nginx 502

Troubleshooting:

    nginx -t
    systemctl status nginx --no-pager
    tail -100 /var/log/nginx/error.log

Check backend:

    curl -i http://10.0.0.51:9000/health
    curl -i http://10.0.0.52:9000/health
    curl -i http://10.0.0.53:9000/health
    curl -i http://10.0.0.54:9000/health

Common Causes:

    Backend RustFS is not running.
    Backend port is unreachable.
    Nginx upstream is written incorrectly.
    Firewall interception.
    RustFS node health issues.

---

### 18.6 Upload Large File Failure

Troubleshooting:

    tail -100 /var/log/nginx/error.log
    docker logs rustfs-cluster --tail=200
    df -hT /data/rustfs0
    df -hT /data/rustfs1
    df -hT /data/rustfs2
    df -hT /data/rustfs3

Nginx needs to focus on:

    client_max_body_size 0
    proxy_request_buffering off
    proxy_read_timeout 3600s
    proxy_send_timeout 3600s

Common Causes:

    Nginx upload size limit.
    Nginx timeout.
    Backend node anomalies.
    Insufficient disk space.
    Client network interruption.
    Multipart Upload compatibility issues.

---

### 18.7 Data Directory Permission Issues

Symptoms:

    permission denied
    Write failure
    Container startup failure

Troubleshooting:

    ls -ld /data/rustfs*
    docker logs rustfs-cluster --tail=200

Resolution:

    chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    docker restart rustfs-cluster

---

## Nineteen. Production Considerations

### 19.1 Do Not Use Experimental Directory Mode as Production Disk Scheme

In experiments:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

May all be on the same system disk.

In production: /data/rustfs*

Each directory should correspond to an independent disk or independent mount point.  
Do not place production object data on the system disk.  
Do not place multiple data directories on the same disk and claim to have multi-disk fault tolerance capabilities.

---

### 19.2 Do Not Use NFS as the Underlying Data Directory

The underlying data directory for RustFS is not recommended to use NFS.

Reasons:

    Locking, latency, write consistency, and performance issues may occur under high I/O.  
    The object storage software layer should directly manage local disks.  
    Production recommends exposing local physical disks using the JBOD method.

---

### 19.3 Do Not Expose Backend Node Ports to the Public Internet

Production should avoid:

    Public access to 10.0.0.51:9000  
    Public access to 10.0.0.52:9000  
    Public access to Console 9001

Production recommendations:

    Backend nodes should only allow internal network access.  
    External access should only go through Nginx / LB unified entry points.  
    External access must use HTTPS.  
    Console management entry should restrict source origins.  
    AccessKey / SecretKey should not be transmitted over public HTTP in plaintext.

---

### 19.4 Do Not Use Experimental Keys

This document uses:

    rustfsadmin  
    RustFSAdmin@123456

These are only for experimental purposes.

Production must:

    Use strong random keys.  
    Root / Admin keys should only be used for initialization.  
    Business should use independent AccessKey.  
    Implement least privilege.  
    Regularly rotate keys.  
    Keys should be disableable after leakage.

---

### 19.5 RustFS Still Requires Production Validation

Even after successful cluster deployment, validation must be performed:

    S3 API compatibility.  
    Multipart Upload.  
    Presigned URL.  
    SDK compatibility.  
    Large object upload/download.  
    Small object high concurrency.  
    Node failure recovery.  
    Disk failure recovery.  
    Data recovery capability.  
    Version upgrades.  
    Monitoring metrics.  
    Log auditing.  
    Permission system.  
    HTTPS reverse proxy.  
    Backup migration.  
    Comparison with MinIO / Ceph RGW.

---

## TwentyI don't know.Experiment Cleanup

### 20.1 Delete Test Bucket

Execute on rustfs-client:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rm --recursive --force rustfs-entry/cluster-demo

Delete Bucket:

    docker run --rm \
      -v /data/rustfs/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      rb rustfs-entry/cluster-demo

High-risk warning:

    Deleting a Bucket in production is a high-risk operation.  
    Must confirm business ownership, backups, and approval before deletion.

---

### 20.2 Stop RustFS Containers

Execute on each RustFS node:

    docker stop rustfs-cluster

---

### 20.3 Remove RustFS Containers

Execute on each RustFS node:

    docker rm -f rustfs-cluster

---

### 20.4 Remove Data Directories

High-risk warning:

    The following commands will delete RustFS cluster experimental data.  
    Do not execute directly in production environments.

After confirming only experimental data will be cleaned, execute on each node:

    rm -rf /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

---

### 20.5 Remove Nginx Configuration

Execute on rustfs-entry:

    rm -f /etc/nginx/conf.d/rustfs-s3.conf  
    nginx -t  
    systemctl reload nginx

---

### 20.6 Remove Test Files

Execute on rustfs-client:

    rm -rf /tmp/rustfs-cluster-test  
    rm -rf /tmp/rustfs-cluster-download  
    rm -f /tmp/rustfs-entry-test.txt  
    rm -f /tmp/after-node04-recovery.txt  
    rm -f /tmp/hello-after-node-stop.txt  
    rm -f /tmp/file-100m-after-restart.bin

---

## Twenty-oneI don't know.Completion Criteria for This Document's Hands-on Practice

After completing this document, the following standards should be at least met:

| Item | Standard |
|---|---|
| Node Planning | 10.0.0.51-54 as RustFS nodes |
| Client | 10.0.0.55 as mc test node |
| Unified Entry | 10.0.0.56 as Nginx entry |
| Hostname Resolution | rustfs-node01-04 can resolve each other |
| Time Synchronization | All nodes have normal timedatectl |
| Data Directories | /data/rustfs0-3 created on each node |
| Directory Permissions | UID 10001 has write access |
| Image Version | Use registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99 |
| Container Status | 4 nodes' rustfs-cluster Running |
| API Health Check | Each node's /health returns 200 |
| mc Access | Accessible via node01 |
| Cross-node Access | node02 can see Buckets created by node01 |
| Bucket | cluster-demo created successfully |
| Object | Small files, directories, and large files uploaded successfully |
| Download Verification | sha256sum matches |
| Nginx Entry | s3.rustfs.local:9000 is accessible |
| Entry Upload | Objects can be uploaded via unified entry |
| Node Failure Observation | Can observe access impact after stopping node04 |
| Node Recovery | node04 can be accessed after recovery |
| Troubleshooting Capability | Can handle DNS, port, permission, Nginx, mc, and data directory issues |

---

## Twenty-twoI don't know.Interview Answer Structure

If asked in an interview:

    How would you deploy and validate a RustFS cluster mode?

You can answer:

# RustFS Multi-Node Multi-Disk Mode

RustFS multi-node multi-disk mode can be understood as MNMD, which refers to a distributed object storage cluster composed of multiple nodes, multiple data directories, or multiple disks. In the experiment, I will plan 4 RustFS nodes, such as 10.0.0.51 to 10.0.0.54, with each node preparing /data/rustfs0 to /data/rustfs3 as four data directories; I will also prepare a client node 10.0.0.55 for mc or AWS CLI testing, and an entry node 10.0.0.56 for Nginx unified entry.

Before deployment, I will first perform hostname resolution, time synchronization, port, Docker, disk, and data directory permission checks. The RustFS cluster relies on nodes accessing each other via hostname and port, so each node must be able to resolve rustfs-node01 to rustfs-node04, and port 9000 must be reachable. Time synchronization is also critical, as S3 signing and multi-node consistency both depend on time.

When starting Docker, all nodes use the same RUSTFS_VOLUMES, such as http://rustfs-node{01...04}:9000/data/rustfs{0...3}, and the mounted path within the container shall be consistent with that path. The commencement order for each node is consistent. Data directory requires chowned to the container runtime UID, such as 10001, otherwise permission denied errors may occur.

After startup, I will check docker ps, docker logs, ss -lntp, and access each node's /health interface. Then I will use mc to configure alias through node01, create a Bucket, and upload objects; then use the alias through node02 to view the same Bucket. If the same data can be seen, it indicates that this is not multiple independent single machines, but a single cluster.

For unified entry, I will configure Nginx upstream on 10.0.0.56, pointing to the 9000 port of the 4 RustFS nodes. The client only needs to configure http://s3.rustfs.local:9000 or subsequent https://s3.rustfs.local, does not directly sense the back end.Nginx, which requires closing request buffering, allowing upload size, and setting a longer timeout to avoid large object upload failures.

During validation, I will upload small files, directories, and files larger than 100Mi, perform sha256sum verification after downloading; I will also stop one node to observe whether the unified entry can still access objects, then restore the node to observe service status.

In production, I will not directly use this experimental directory mode as a production solution. In production, each data directory should correspond to an independent disk, backend node ports should not be directly exposed to the public internet, external access must use HTTPS, Console should restrict sources. As a new object storage, RustFS must also verify S3 compatibility, Multipart Upload, SDK compatibility, node failure recovery, disk failure recovery, monitoring logs, permission system, backup migration, and upgrade paths before going into production.

---

## 23. Summary of This Article

This article completes the practice of deploying a RustFS cluster:

1. RustFS multi-node multi-disk mode belongs to MNMD.
2. MNMD is closer to the production cluster form of object storage.
3. This article plans 4 RustFS nodes.
4. This article plans 1 client node.
5. This article plans 1 Nginx unified entry node.
6. All RustFS nodes must be able to resolve hostnames.
7. All RustFS nodes must maintain time synchronization.
8. All RustFS nodes must use the same listening port.
9. This article plans /data/rustfs0 to /data/rustfs3 for each node.
10. The experimental directory mode cannot replace production independent disks.
11. Production recommends each data directory corresponds to an independent data disk.
12. RustFSBottom data directories should not use NFS.
13. Docker container data directories must be mounted to the host.
14. Host data directories need to allow UID 10001 to write.
15. All nodes use the same RUSTFS_VOLUMES.
16. mc can access the same cluster through any node.
17. A unified entry can be achieved through Nginx.
18. The Nginx entry needs to pay attention to upload size, request buffering, and timeout.
19. External production access must use HTTPS.
20. Backend node ports should not be directly exposed to the public internet.
21. Stopping a single node can be used to observe the impact on cluster access.
22. Cluster deployment success does not equal production readiness.
23. RustFS must continue to verify compatibility, stress testing, failure recovery, permissions, security, and backup migration before production.
24. The next article will learn about the comparison between RustFS and MinIO: architecture, deployment, ecosystem, and operation differences.

---

## 24. Reference Documents

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS Docker installation documentation:

    https://docs.rustfs.com/installation/docker/

RustFS multi-node multi-disk installation documentation:

    https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html

RustFS Docker Hub:

    https://hub.docker.com/r/rustfs/rustfs

RustFS GitHub:

    https://github.com/rustfs/rustfs

RustFS Nginx reverse proxy configuration:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS TLS configuration:

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