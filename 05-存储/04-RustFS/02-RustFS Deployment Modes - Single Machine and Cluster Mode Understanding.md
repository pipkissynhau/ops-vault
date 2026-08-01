# RustFS Deployment Modes: Single Machine Mode and Cluster Mode Understanding

Suggested path: 05-Storage/04-RustFS/02-RustFS Deployment Modes: Single Machine Mode and Cluster Mode Understanding.md

Tags: #RustFS #ObjectStorage #S3 #DeploymentPattern #UnitDeployment #ClusterDeployment #Docker #Multinodes #Multidisk #ReverseAgent #HTTPS #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This document is the second article of the RustFS module, focusing on understanding RustFS deployment modes, node planning, disk planning, access entry design, and production boundaries.

The previous article has been completed:

    RustFS Basics: S3-Compatible Object Storage and Use Cases

This document does not directly enter complete deployment commands. Complete single-machine deployment will be covered in 03, and multi-node cluster deployment in 04.

This document focuses on solving the following questions:

    What are the deployment modes of RustFS?
    When is single-machine mode suitable?
    When is single-node multi-disk mode suitable?
    When is multi-node multi-disk cluster mode suitable?
    Why do object storage clusters typically require multiple nodes and multiple disks?
    How should RustFS nodes, disks, ports, domains, and reverse proxies be planned?
    How should internal HTTP and external HTTPS be designed?
    What are the differences between Docker deployment and VM/bare-metal deployment?
    Why can't single-machine experiments be directly used in production?
    Why must RustFS be validated for version, compatibility, fault recovery, and data reliability before production?

This document emphasizes a principle:

    The RustFS module should be advanced in the order of "single-machine verification first, then multi-node cluster, then unified entry, then security and operations."
    Single-machine deployment is for learning and functional verification.
    Multi-node multi-disk mode is closer to the production object storage cluster form.
    Compatibility, pressure testing, backup, recovery, and fault drills must be completed before production use.

---

## II. Learning Objectives

After completing this document, you should be able to:

1. Understand the role of RustFS single-machine mode.
2. Understand the role of RustFS single-node multi-disk mode.
3. Understand the role of RustFS multi-node multi-disk mode.
4. Understand deployment forms like SNSD, SNMD, and MNMD.
5. Understand why distributed object storage cannot be judged solely by whether containers are running.
6. Plan a RustFS single-machine Docker experimental environment.
7. Plan a RustFS multi-node Docker/VM cluster environment.
8. Plan RustFS data directories and disk mount paths.
9. Plan RustFS API ports, management ports, and unified entry points.
10. Understand the difference between internal HTTP and external HTTPS.
11. Understand the role of Nginx/LB reverse proxies in front of RustFS.
12. Judge whether a scenario is suitable for single-machine, test cluster, or production cluster.
13. Understand the similarities between RustFS and MinIO deployment methods.
14. Understand that new RustFS solutions need cautious validation before production.
15. Lay the foundation for subsequent 03 single-machine deployment and 04 multi-node cluster deployment.

---

## III. Overview of RustFS Deployment Modes

RustFS can be understood as three deployment modes based on the number of nodes and disks:

    Single Node Single Disk Mode
    Single Node Multi-Disk Mode
    Multi-Node Multi-Disk Mode

These can also be corresponded to:

    SNSD: Single Node Single Disk
    SNMD: Single Node Multi-Disk
    MNMD: Multi-Node Multi-Disk

Chinese understanding:

| Mode | Chinese Name | Description |
|---|---|---|
| SNSD | Single Node Single Disk | One node, one data directory or one data disk |
| SNMD | Single Node Multi-Disk | One node, multiple data directories or multiple data disks |
| MNMD | Multi-Node Multi-Disk | Multiple servers, each with at least one or more data disks |

Learning order recommendation:

    1. First learn SNSD, understand service startup, ports, data directories, and S3 access.
    2. Then learn SNMD, understand single-node multi-disk and capacity planning.
    3. Then learn MNMD, understand distributed object storage, node failures, and unified entry.

---

## IV. Single Node Single Disk Mode: SNSD

### 4.1 Mode Definition

SNSD represents:

    Single Node Single Disk

That is:

    One RustFS node
    One data directory
    One Docker container or one process
    One S3 API entry

Experimental example:

    rustfs-node01
    10.0.0.51
    /data/rustfs
    RustFS API Port
    RustFS Console / Management Port, such as currently supported

---

### 4.2 SNSD Topology

    ┌─────────────────────────────────────┐
    │ rustfs-node01                       │
    │ 10.0.0.51                           │
    │                                     │
    │ Docker                              │
    │   └── rustfs                        │
    │       ├── S3 API Port               │
    │       └── Data: /data/rustfs        │
    └─────────────────────────────────────┘
                    ^
                    |
                    | S3 API
                    |
    ┌─────────────────────────────────────┐
    │ rustfs-client                       │
    │ 10.0.0.55                           │
    │ mc / aws cli / curl                 │
    └─────────────────────────────────────┘

---

### 4.3 SNSD Suitable Scenarios

Suitable for:

    Learning RustFS basics
    Quickly verifying service startup
    Verifying basic S3 API compatibility
    Verifying mc / AWS CLI access
    Verifying bucket creation
    Verifying object upload/download
    Verifying data directory persistence
    Verifying Nginx reverse proxy
    Verifying HTTPS entry
    Development/test environment
    Non-critical temporary object storage

---

### 4.4 SNSD Unsuitable Scenarios

Unsuitable: /think

High Availability for Production  
Node Fault Tolerance  
Disk Fault Tolerance  
Critical Business Object Storage  
Unique Backup Target  
Large-Scale Object Storage  
Multi-Tenant Production Object Storage  
Business with High Availability Requirements  

Reasons:  

    Single-node failure causes service unavailability.  
    Single-disk failure poses extremely high data risk.  
    No node-level high availability capability.  
    Unable to verify real distributed recovery capability.  

---

### 4.5 SNSD Learning Focus  

The SNSD phase focuses not on high availability, but on understanding the basic workflow:  

    How RustFS service starts  
    Where the data directory is located  
    How API ports are exposed  
    How AccessKey / SecretKey are configured  
    How mc accesses  
    How Bucket is created  
    How Object is uploaded/downloaded  
    Whether data is preserved after container restart  
    How logs are viewed  
    How to troubleshoot port conflicts  

---

## FiveI don't know.Single-Node Multi-Disk Mode: SNMD  

### 5.1 Mode Definition  

SNMD stands for:  

    Single Node Multiple Disk  

That is:  

    One RustFS node  
    Multiple data directories or multiple data disks  
    One service process  
    Multiple local storage paths  

Example:  

    rustfs-node01  
    10.0.0.51  

    /data/rustfs/disk1  
    /data/rustfs/disk2  
    /data/rustfs/disk3  
    /data/rustfs/disk4  

---

### 5.2 SNMD Topology  

    ┌─────────────────────────────────────┐  
    │ rustfs-node01                       │  
    │ 10.0.0.51                           │  
    │                                     │  
    │ Docker / RustFS                     │  
    │   ├── /data/rustfs/disk1            │  
    │   ├── /data/rustfs/disk2            │  
    │   ├── /data/rustfs/disk3            │  
    │   └── /data/rustfs/disk4            │  
    └─────────────────────────────────────┘  
                    ^  
                    |  
                    | S3 API  
                    |  
    ┌─────────────────────────────────────┐  
    │ rustfs-client                       │  
    │ 10.0.0.55                           │  
    └─────────────────────────────────────┘  

---

### 5.3 SNMD Suitable Scenarios  

Suitable for:  

    Single-node multi-disk learning  
    Understanding multi-data path planning  
    Validating single-node multi-disk capacity expansion  
    Validating disk directory permissions  
    Validating multi-disk startup parameters  
    Validating object write distribution  
    Small-scale test environment  

---

### 5.4 SNMD Unsuitable Scenarios  

Not suitable for:  

    Production high availability  
    Node fault tolerance  
    Cross-node recovery  
    Multi-node disaster recovery  
    Core business object storage  

Reasons:  

    Although there are multiple disks, there is still only one node.  
    After server failure, the entire object storage service becomes unavailable.  
    Node-level failures cannot be continued by other nodes.  

---

### 5.5 SNMD Learning Focus  

SNMD phase focus:  

    Multi-data directory planning  
    Multi-disk mounting  
    Each disk has an independent file system  
    Data directory permissions  
    Disk capacity consistency  
    Disk performance consistency  
    Container startup parameters  
    Single-node capacity expansion boundary  

---

## SixI don't know.Multi-Node Multi-Disk Mode: MNMD  

### 6.1 Mode Definition  

MNMD stands for:  

    Multiple Node Multiple Disk  

That is:  

    Multiple RustFS nodes  
    Each node has at least one or more data directories  
    Multiple nodes form a distributed object storage cluster  
    Clients access through a unified entry point  

Example plan:  

| IP | Hostname | Data Directory |  
|---|---|---|  
| 10.0.0.51 | rustfs-node01 | /data/rustfs |  
| 10.0.0.52 | rustfs-node02 | /data/rustfs |  
| 10.0.0.53 | rustfs-node03 | /data/rustfs |  
| 10.0.0.54 | rustfs-node04 | /data/rustfs |  

Unified entry point:  

| IP | Hostname | Purpose |  
|---|---|---|  
| 10.0.0.56 | rustfs-entry | Nginx / LB HTTPS entry |  

Clients:  

| IP | Hostname | Purpose |  
|---|---|---|  
| 10.0.0.55 | rustfs-client | mc / AWS CLI / application testing |  

---

### 6.2 MNMD Topology  

    ┌─────────────────────────────────────┐  
    │ rustfs-node01                       │  
    │ 10.0.0.51                           │  
    │ /data/rustfs                        │  
    └───────────────────┬─────────────────┘  
                        │  
    ┌───────────────────┼─────────────────┐  
    │                   │                 │  
    v                   v                 v  
    ┌─────────────────────────────────────┐  
    │ rustfs-node02                       │  
    │ 10.0.0.52                           │  
    │ /data/rustfs                        │  
    └─────────────────────────────────────┘

```
┌─────────────────────────────────────┐
│ rustfs-node03                       │
│ 10.0.0.53                           │
│ /data/rustfs                        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ rustfs-node04                       │
│ 10.0.0.54                           │
│ /data/rustfs                        │
└─────────────────────────────────────┘

                    ^
                    |
                    | Internal HTTP / Cluster Communication
                    |
┌─────────────────────────────────────┐
│ rustfs-entry                        │
│ 10.0.0.56                           │
│ Nginx / LB                          │
│ HTTPS: 443                          │
└─────────────────────────────────────┘
                    ^
                    |
                    | External HTTPS S3 API
                    |
┌─────────────────────────────────────┐
│ rustfs-client                       │
│ 10.0.0.55                           │
│ mc / aws cli / app                  │
└─────────────────────────────────────┘

---

### 6.3 MNMD Suitable Scenarios

Suitable:

    Distributed Object Storage Learning
    Node High Availability Verification
    Multi-node Fault Simulation
    Multi-disk Capacity Planning
    Unified Entry Design
    Reverse Proxy Testing
    HTTPS Entry Testing
    S3 Client Compatibility Testing
    Non-core Business Pilot
    Pre-production Verification Environment

---

### 6.4 MNMD Production Value

MNMD is closer to the production form of object storage.

It can verify:

    Multi-node Startup
    Node-to-node Communication
    Multi-disk Data Distribution
    Node Failure Impact
    Node Recovery Process
    Unified Access Entry
    Client Access Stability
    Object Upload/Download Consistency
    Large Object Upload
    Concurrent Access
    Backup Migration
    Monitoring Alerts

---

### 6.5 MNMD Still Requires Caution Before Production

Even with MNMD, it doesn't mean it's ready for production.

Still needs verification:

    Current RustFS Version Stability
    S3 API Compatibility
    Multipart Upload
    Presigned URL
    SDK Compatibility
    Node Failure Recovery
    Disk Failure Recovery
    Data Recovery Capability
    Permission System
    HTTPS Reverse Proxy
    Logs and Monitoring
    Version Upgrade
    Backup Migration
    Performance Stress Testing
    Community Activity and Documentation Completeness

---

## SevenI don't know.Three Deployment Mode Comparison

| Comparison Item | SNSD | SNMD | MNMD |
|---|---|---|---|
| Number of Nodes | 1 | 1 | Multiple |
| Number of Disks | 1 | Multiple | Multi-node Multi-disk |
| Learning Difficulty | Low | Medium | High |
| Deployment Complexity | Low | Medium | High |
| Suitable for Basic Learning | Suitable | Suitable | Suitable for Advanced |
| Node High Availability | Not Available | Not Available | Has Verification Value |
| Suitable for Production | Not Recommended | Caution | Needs Full Verification Assessment |
| Fault Simulation Value | Low | Medium | High |
| Unified Entry Requirement | Optional | Optional | Recommended |
| HTTPS Entry | Recommended | Recommended | Mandatory |
| Typical Use Cases | Entry-level Verification | Single-node Multi-disk Verification | Cluster and Production Boundary Verification |

---

## EightI don't know.RustFS Experiment Environment Planning

### 8.1 Network Planning

Continue using the 10.0.0.0/24 experimental network segment.

Occupied or reserved:

| IP | Purpose |
|---|---|
| 10.0.0.10 | ops-server, GitLab / Jenkins / Harbor / Operations Services |
| 10.0.0.20 | Kubernetes Master |
| 10.0.0.21 | Kubernetes Worker |
| 10.0.0.22 | Kubernetes Worker |
| 10.0.0.41 - 10.0.0.46 | MinIO Experiment Nodes and Entry Planning |

RustFS Dedicated Planning:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Single-node / Cluster Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Cluster Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Cluster Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Cluster Node 4 |
| 10.0.0.55 | rustfs-client | S3 Client Test Node |
| 10.0.0.56 | rustfs-entry | Nginx / LB HTTPS Unified Entry |

---

### 8.2 Hostname Planning

Recommended hostnames:

    rustfs-node01
    rustfs-node02
    rustfs-node03
    rustfs-node04
    rustfs-client
    rustfs-entry

Check:

    hostname

Set example:

    hostnamectl set-hostname rustfs-node01

hosts example: /think

```bash
cat >> /etc/hosts <<'EOF'
10.0.0.51 rustfs-node01
10.0.0.52 rustfs-node02
10.0.0.53 rustfs-node03
10.0.0.54 rustfs-node04
10.0.0.55 rustfs-client
10.0.0.56 rustfs-entry
EOF
```

**Notes:**

- The experimental environment can use `/etc/hosts`.
- Production environments are recommended to use internal DNS.
- Node communication should not rely on unstable resolution.

---

### 8.3 Operating System Planning

**Default System:**

- Ubuntu Server 22.04.5 LTS

**Optional Additions:**

- Rocky Linux 9

**Ubuntu is suitable for:**
- Rapid experimentation
- Docker deployment
- Nginx reverse proxy
- Consistency with previous K8s / MinIO / Longhorn experiments

**Rocky Linux 9 is suitable for:**
- Simulating RHEL-based production environments
- Practicing firewalld
- Practicing SELinux considerations
- Enterprise production operations supplements

---

## IX. Docker Deployment vs VM Deployment Differences

### 9.1 Docker Deployment

**Docker Deployment Characteristics:**

- Fast startup
- Easy cleanup
- Convenient for fixed image versions
- Suitable for single-node experiments
- Suitable for multi-node Docker / VM clusters
- Does not pollute the host's binary environment

**Suitable for:**
- Learning experiments
- Rapid validation
- Single-node deployment
- Multi-node Docker / VM clusters
- Client compatibility testing

**Notes:**
- Container data directories must be mounted to the host.
- Do not store data only inside containers.
- Need to set restart policies.
- Need to plan port mappings.
- Need to check log and data directory permissions.
- Need to fix image versions.

---

### 9.2 VM / Bare Metal Binary Deployment

**VM / Bare Metal Deployment Characteristics:**

- Closer to traditional production service deployment
- Processes managed by systemd
- Convenient for integration with system services, disks, and logs
- Suitable for serious production evaluation

**Suitable for:**
- Pre-production verification
- System-level operations learning
- systemd management
- Standardized log path
- Integration with enterprise host baselines

**Notes:**
- Binary version management
- systemd unit files
- Environment variable management
- Data directory permissions
- Upgrade rollback
- Log rotation
- Service auto-start

---

### 9.3 Module Selection

This module defaults to:
- Single-node Docker
- Multi-node Docker / VM cluster

**Reasons:**
- Fast hands-on practice.
- Easy to reproduce.
- Easy to clean up.
- Consistent with MinIO module experiment methods.
- Suitable for learning object storage deployment patterns.
- No dependency on Kubernetes.
- No modification to containerd.
- No impact on current K8s cluster.

**Production Notes:**
- Docker experiment success does not equate to production readiness.
- Production still requires evaluation of binary deployment, container deployment, system services, logs, monitoring, security, and upgrade strategies.

---

## X. Data Directory and Disk Planning

### 10.1 Single-Node Mode Data Directory

**Single-Node Mode:**
- `/data/rustfs`

**Creation:**
- `mkdir -p /data/rustfs`

**Verification:**
- `df -hT /data/rustfs`
- `lsblk -f`
- `ls -ld /data/rustfs`

---

### 10.2 Single-Node Multi-Disk Directory

**Single-Node Multi-Disk Mode:**
- `/data/rustfs/disk1`
- `/data/rustfs/disk2`
- `/data/rustfs/disk3`
- `/data/rustfs/disk4`

**Creation:**
- `mkdir -p /data/rustfs/disk{1..4}`

**Verification:**
- `df -hT /data/rustfs/disk1`
- `df -hT /data/rustfs/disk2`
- `df -hT /data/rustfs/disk3`
- `df -hT /data/rustfs/disk4`

---

### 10.3 Multi-Node Data Directory

**Per Node:**
- `/data/rustfs`

**Or Multi-Disk:**
- `/data/rustfs/disk1`
- `/data/rustfs/disk2`

**Recommendations:**
- All node paths should remain consistent.
- All node disk capacities should be as consistent as possible.
- All node disk performance should be as consistent as possible.
- Do not mix slow and fast disks for the same cluster data.
- Do not place data directories directly on the system disk for production.

---

### 10.4 Independent Data Disk Recommendations

**Production Recommendations:**
- Use independent data disks.
- Mount data disks to `/data`.
- Place RustFS data directories in `/data/rustfs`.
- Use xfs or ext4 file systems.
- Mount parameters can use `noatime`.
- Write to `/etc/fstab` to ensure automatic mounting after reboot.

**Verification:**
- `df -hT /data`
- `lsblk -f`
- `mount | grep /data`

**/etc/fstab Example:**
- `UUID=<actual UUID> /data xfs defaults,noatime 0 0`

or:

- `UUID=<actual UUID> /data ext4 defaults,noatime 0 0`

---

### 10.5 High-Risk Warnings

The following operations must be handled with caution in production environments:

- `mkfs.xfs /dev/sdb`
- `mkfs.ext4 /dev/sdb`
- `rm -rf /data/rustfs`
- Modify `/etc/fstab`
- Unmount data disks
- Move RustFS data directories
- Manually delete internal object storage data files

**Before execution, confirm:**
- Whether the device name is correct.
- Whether there is data.
- Whether there is a backup.
- Whether there is a maintenance window.
- Whether it affects business operations.
- Whether there is a recovery plan.

---

## XI. Port Planning

### 11.1 Port Types

RustFS deployment typically requires attention to:

- S3 API port
- Console / management port (e.g., provided in the current version)
- Nginx HTTPS entry port
- Nginx HTTP redirect port
- SSH maintenance port

Specific ports will follow the official documentation of the current RustFS version and actual startup parameters.

This module expresses this as: `/think`

| Port | Purpose | Exposure Recommendation |
|---|---|---|
| RustFS API Port | S3 API Service | Intranet access, external via Nginx |
| RustFS Console Port | Management interface, e.g. version support | Only access from operations network segment |
| 443 | Nginx HTTPS Unified Entry | Open to clients |
| 80 | HTTP Redirect to HTTPS | Optional |
| 22 | SSH Operations | Only access from operations source |

---

### 11.2 Port Conflict Check

Before deployment check:

    ss -lntp

Check specified ports:

    ss -lntp | grep 9000
    ss -lntp | grep 9001
    ss -lntp | grep 443

If port is occupied, need to confirm:

    Which process is using it.
    Whether it's an existing service.
    Whether RustFS port can be changed.
    Whether Nginx unified entry should be used to avoid port exposure.

---

### 11.3 Firewall Planning

Ubuntu ufw example:

    ufw status

Rocky firewalld example:

    firewall-cmd --list-all

Production recommendations:

    RustFS backend API port should only allow intranet or Nginx access.
    Management port should only allow operations network segment access.
    External clients should only access Nginx 443.
    SSH should only allow bastion host or operations IP.
    Do not expose RustFS API and management ports directly to public internet.

---

## TwelveI don't know.Access Entry Design

### 12.1 Why Unified Entry is Needed

Multi-node object storage does not recommend clients to remember multiple backend nodes.

Unified entry can solve:

    Simplified client configuration
    Scalable backend nodes
    Fault-tolerant backend nodes
    Unified HTTPS certificate
    Unified access logs
    Unified upload size limits
    Unified timeout parameters
    Unified source IP control

Unified entry can use:

    Nginx
    HAProxy
    Keepalived + Nginx
    Cloud load balancer
    Hardware load balancer

---

### 12.2 Experimental Entry Planning

Unified entry node:

    10.0.0.56 rustfs-entry

Domain:

    s3.rustfs.local

Hosts:

    10.0.0.56 s3.rustfs.local

Access method:

    https://s3.rustfs.local

Backend nodes:

    http://10.0.0.51:<RustFS API Port>
    http://10.0.0.52:<RustFS API Port>
    http://10.0.0.53:<RustFS API Port>
    http://10.0.0.54:<RustFS API Port>

---

### 12.3 Internal HTTP vs External HTTPS

Recommended design:

    Internal node communication: HTTP
    External client access: HTTPS

Reasons:

    Internal nodes are in trusted network.
    Internal HTTP can reduce TLS overhead.
    External access must use HTTPS to protect AccessKey, SecretKey and object transfer.
    Certificates are uniformly managed in Nginx / LB layer.
    Clients only perceive unified HTTPS endpoint.

Notes:

    Internal HTTP must be based on trusted network, access control and firewall restrictions.
    If across untrusted networks or across data centers, internal TLS should be evaluated.
    Do not expose internal HTTP backend ports directly to public internet.

---

### 12.4 External HTTPS Entry Diagram

    Client / mc / AWS CLI
             |
             | HTTPS
             v
    ┌─────────────────────────────┐
    │ Nginx / LB                  │
    │ s3.rustfs.local:443         │
    └──────────────┬──────────────┘
                   |
                   | Internal HTTP
                   v
    ┌─────────────────────────────┐
    │ rustfs-node01               │
    │ 10.0.0.51                   │
    └─────────────────────────────┘
    ┌─────────────────────────────┐
    │ rustfs-node02               │
    │ 10.0.0.52                   │
    └─────────────────────────────┘
    ┌─────────────────────────────┐
    │ rustfs-node03               │
    │ 10.0.0.53                   │
    └─────────────────────────────┘
    ┌─────────────────────────────┐
    │ rustfs-node04               │
    │ 10.0.0.54                   │
    └─────────────────────────────┘

---

## ThirteenI don't know.Domain and Certificate Planning

### 13.1 Experimental Domain

Experimental can use:

    s3.rustfs.local
    console.rustfs.local

Hosts:

    10.0.0.56 s3.rustfs.local
    10.0.0.56 console.rustfs.local

---

### 13.2 Production Domain

Production recommendations:

    s3.example.com
    rustfs-console.example.com

Or by business differentiation:

    s3.internal.example.com
    backup-s3.example.com

Production principles:

    API domain and management domain should be separated.
    Management domain should not be open to public internet.
    API domain must use HTTPS.
    Certificates must be renewable.
    Certificate expiration should have monitoring.
    S3 client Endpoint should be fixed and not changed arbitrarily.

---

### 13.3 Certificate Planning

Certificate sources:

    Public CA certificate
    Enterprise internal CA certificate
    Experimental self-signed certificate

Production recommendations:

Public or cross-organization access should use publicly trusted certificates.  
Internal access can use enterprise internal CA.  
Self-signed certificates are only suitable for experiments.  
Clients need to trust the certificate chain.  
Certificate expiration will cause mc, AWS CLI, and SDK access failures.

---

## FourteenI don't know.Image Version and Domestic Network Planning

### 14.1 Fixed Version Principle

Not recommended to use:

    latest

Reasons:

    latest will change.  
    Version changes may cause changes in startup parameters.  
    Functional behavior may change.  
    Interface may change.  
    Troubleshooting and experiment reproduction will be difficult.

Recommendations:

    Use fixed version images.  
    Record official image sources.  
    Record pull time.  
    Record tag.  
    Synchronize to your own Alibaba Cloud mirror repository or Harbor.

---

### 14.2 Domestic Acceleration Principle

Docker image pulls can use:

    mdocker acceleration address

But it is more recommended to:

    Official image -> Local pull -> Tag -> Alibaba Cloud mirror repository / Harbor -> Subsequent unified use of private repository images

Reasons:

    Domestic network is more stable.  
    Experiments can be reproduced.  
    Not dependent on real-time access to the internet.  
    Facilitate unified versioning.  
    Facilitate internal network pulls in production environments.

---

### 14.3 Image Synchronization Template

After confirming the fixed version of RustFS, execute:

    docker pull rustfs/rustfs:<fixed version>

    docker tag rustfs/rustfs:<fixed version> \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:<fixed version>

    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:<fixed version>

Notes:

    The specific image name and version will be determined by the actual verification in section 03 later.  
    Do not write fixed versions based on memory in advance.  
    Production environments must combine official Release Notes, compatibility, and security patch evaluations for versions.

---

## FifteenI don't know.Single-Node Deployment Planning

### 15.1 Single-Node

Node:

    10.0.0.51 rustfs-node01

Data directory:

    /data/rustfs

Access entry:

    http://10.0.0.51:<RustFS API Port>

Subsequently can be exposed via Nginx:

    https://s3.rustfs.local

---

### 15.2 Single-Node Deployment Check Items

Pre-deployment checks:

    hostname  
    cat /etc/os-release  
    docker version  
    docker info  
    df -hT  
    lsblk -f  
    ss -lntp  
    timedatectl

Data directory:

    mkdir -p /data/rustfs  
    df -hT /data/rustfs  
    ls -ld /data/rustfs

---

### 15.3 Single-Node Verification Goals

After single-node deployment, must verify:

    Container is running.  
    API port is listening.  
    No obvious errors in logs.  
    mc alias configuration is successful.  
    Bucket creation is successful.  
    Object upload is successful.  
    Object download is successful.  
    Objects remain after container restart.  
    Access failure with incorrect keys.  
    Errors for non-existent Buckets are understandable.

---

## SixteenI don't know.Cluster Deployment Planning

### 16.1 Cluster Nodes

Recommended 4 nodes:

    10.0.0.51 rustfs-node01  
    10.0.0.52 rustfs-node02  
    10.0.0.53 rustfs-node03  
    10.0.0.54 rustfs-node04

Each node:

    /data/rustfs

Entry node:

    10.0.0.56 rustfs-entry

Client:

    10.0.0.55 rustfs-client

---

### 16.2 Why Recommend Starting with 4 Nodes

Distributed object storage clusters typically require multiple nodes to demonstrate:

    Data distribution  
    Node high availability  
    Disk fault tolerance  
    Node fault tolerance  
    Cluster recovery  
    Unified entry  
    Load balancing  
    Real production boundaries

Official multi-node multi-disk documentation also emphasizes that securing the start of a distributed object storage cluster requires at least 4 servers, each with at least 1 disk.

For learning purposes, 4 nodes make it easier to verify:

    Impact of single-node failure  
    Single-node recovery  
    Multi-node access  
    Backend removal  
    Unified entry forwarding  
    Whether data is still accessible

---

### 16.3 Cluster Deployment Check Items

Check each node:

    hostname  
    ip addr  
    ping rustfs-node01  
    ping rustfs-node02  
    ping rustfs-node03  
    ping rustfs-node04  
    docker version  
    df -hT /data/rustfs  
    ss -lntp  
    timedatectl

Network check:

    ping -c 5 10.0.0.51  
    ping -c 5 10.0.0.52  
    ping -c 5 10.0.0.53  
    ping -c 5 10.0.0.54

Port check:

    ss -lntp

Firewall check:

    ufw status

Or Rocky:

    firewall-cmd --list-all

---

### 16.4 Cluster Verification Goals

After cluster deployment, must verify:

    All RustFS nodes are running normally.  
    All node ports are listening normally.  
    Unified entry access is normal.  
    mc successfully configures alias via unified entry.  
    Bucket creation is successful.  
    Small object upload/download is successful.  
    Large object upload/download is successful.  
    Multiple objects batch upload is successful.  
    Service recovery after node restart.  
    Impact of single-node anomalies on object access is understandable.  
    Logs are viewable.  
    Data directory capacity is observable.  
    Nginx access.log and error.log are recorded normally.

---

## SeventeenI don't know.Client Planning

### 17.1 Client Node

Client node:

    10.0.0.55 rustfs-client

Installation tools:

    mc  
    aws cli  
    curl  
    jq

Purpose:

    Configure RustFS alias.  
    Create Bucket.  
    Upload objects.  
    Download objects.  
    Generate test files.  
    Verify sha256sum.  
    Test HTTPS certificate.  
    Test incorrect keys.  
    Test non-existent objects.  
    Test large object upload.

---

### 17.2 mc Tool Verification

Subsequent verification commands: /think

mc alias set rustfs http://10.0.0.51:<RustFS API Port> <ACCESS_KEY> <SECRET_KEY>

Unified HTTPS Entry Point:

    mc alias set rustfs https://s3.rustfs.local <ACCESS_KEY> <SECRET_KEY>

Basic Operations:

    mc mb rustfs/test-bucket
    echo "hello rustfs" > hello.txt
    mc cp hello.txt rustfs/test-bucket/
    mc ls rustfs/test-bucket
    mc cp rustfs/test-bucket/hello.txt ./hello-download.txt
    sha256sum hello.txt hello-download.txt

---

### 17.3 AWS CLI Verification

Subsequent verification command format:

    aws --endpoint-url https://s3.rustfs.local s3 ls

Create Bucket:

    aws --endpoint-url https://s3.rustfs.local s3 mb s3://test-bucket

Upload:

    aws --endpoint-url https://s3.rustfs.local s3 cp hello.txt s3://test-bucket/hello.txt

Download:

    aws --endpoint-url https://s3.rustfs.local s3 cp s3://test-bucket/hello.txt ./hello-download.txt

---

## EighteenI don't know.Deployment Mode Selection Recommendations

### 18.1 Learning Phase

Recommended:

    SNSD Single-machine Docker

Reasons:

    Fastest to understand RustFS.
    Fastest to verify S3 API.
    Fastest to verify mc.
    Smaller troubleshooting scope.
    Low cost.

---

### 18.2 Experimental Advancement Phase

Recommended:

    MNMD Multi-node Docker / VM

Reasons:

    Verify cluster mode.
    Verify multi-node.
    Verify unified entry point.
    Verify node failure.
    Verify reverse proxy.
    Verify HTTPS.
    Closer to production environment.

---

### 18.3 Production Pilot Phase

Recommended:

    Multi-node multi-disk
    Independent data disk
    Unified HTTPS entry point
    Independent management entry point
    Monitoring alerts
    Backup migration plan
    Failure drill
    Fixed version
    Gray-scale business

Not recommended:

    Direct production with single-machine mode.
    Direct production with latest image.
    Production without HTTPS.
    Production without monitoring.
    Production without recovery drill.
    Production without S3 compatibility testing.

---

## NineteenI don't know.Pre-deployment Checklist

### 19.1 General Checks

All nodes:

    hostname
    cat /etc/os-release
    ip addr
    ip route
    timedatectl
    df -hT
    lsblk -f
    ss -lntp
    docker version
    docker info

---

### 19.2 Time Synchronization

Check:

    timedatectl

Requirements:

    System clock synchronized: yes

Reasons:

    S3 signature and time-related.
    Inconsistent node time may cause authentication failure.
    Log troubleshooting depends on accurate time.

---

### 19.3 DNS and hosts

Check:

    getent hosts rustfs-node01
    getent hosts rustfs-node02
    getent hosts rustfs-node03
    getent hosts rustfs-node04
    getent hosts s3.rustfs.local

---

### 19.4 Disks and Directories

Check:

    df -hT /data
    df -hT /data/rustfs
    ls -ld /data/rustfs
    du -sh /data/rustfs

---

### 19.5 Ports

Check:

    ss -lntp
    ss -lntp | grep 443
    ss -lntp | grep 9000
    ss -lntp | grep 9001

Notes:

    Specific RustFS ports will be determined by the actual version later.
    Must confirm port conflicts are resolved before deployment.

---

## TwentyI don't know.Common Misunderstandings

### 20.1 Error: Single-machine mode can be used for production just because it runs

Error.

Single-machine mode only proves the service can start and basic S3 API is available.

Cannot prove:

    High availability
    Node failure recovery
    Disk failure recovery
    Data reliability
    Concurrency capability
    Production stability

---

### 20.2 Error: Docker container running means data security

Error.

Must confirm:

    Whether data directory is mounted to host.
    Whether data directory is on independent data disk.
    Whether data is preserved after container restart.
    How data is recovered after node failure.
    Whether there is a backup migration plan.

---

### 20.3 Error: Multiple nodes means cluster

Error.

Multi-node cluster must confirm:

    Whether nodes are actually part of the same cluster.
    Whether data is distributed.
    Whether clients access through unified entry point.
    Whether node failure behavior meets expectations.
    Whether recovery process is controllable.

Multiple identical single-machine containers do not equal a distributed object storage cluster.

---

### 20.4 Error: Internal HTTP is always insecure

Not entirely correct.

Internal HTTP can be used for communication between trusted network nodes, provided:

    Internal network isolation.
    Firewall restrictions.
    Backend ports not exposed to public internet.
    External access uses HTTPS.
    Management entry point strictly restricted.
    Keys not leaked.

If across untrusted networks, internal TLS must be evaluated.

---

### 20.5 Error: External clients can also use HTTP

Not recommended.

External client access must use HTTPS.

Reasons:

    Protect AccessKey / SecretKey.
    Protect object transmission data.
    Avoid man-in-the-middle attacks.
    Meet basic security baseline.
    Facilitate production compliance.

---

## Twenty-oneI don't know.Production Risk Checklist

Must pay attention to before RustFS production:

Is the current version stable?  
Is the official documentation complete?  
Does the S3 API meet business requirements?  
Is the SDK compatible?  
Is Multipart Upload stable?  
Is large object upload stable?  
Is small object high concurrency stable?  
Is data available after node failure?  
How to recover after disk failure?  
Is version upgrade reliable?  
Is the data migration tool mature?  
Are monitoring metrics sufficient?  
Are logs suitable for auditing?  
Does the permission system meet requirements?  
Is HTTPS and reverse proxy compatible?  
Is backup and recovery verifiable?  

---

## Twenty-two, Experiment Completion Standards  

After completing this article, you should be able to output at least the following planning:  

| Project | Standard |  
|---|---|  
| Deployment Mode | Clearly define the differences between SNSD, SNMD, and MNMD |  
| Single-node Node | Clearly define 10.0.0.51 |  
| Cluster Node | Clearly define 10.0.0.51-10.0.0.54 |  
| Client Node | Clearly define 10.0.0.55 |  
| Unified Entry | Clearly define 10.0.0.56 |  
| Data Directory | Clearly define /data/rustfs |  
| Port Planning | Clearly define API, Console, Nginx 443 |  
| Access Method | Internal HTTP, External HTTPS |  
| Domain Name | s3.rustfs.local |  
| Image Strategy | Fixed version, synchronize to private repository |  
| Deployment Order | Start with single-node, then cluster, then entry, then security operations |  
| Production Boundary | Clearly define that you cannot directly jump to production from single-node experiment |  

---

## Twenty-three, Interview AnswerThinking.  

If asked in an interview:  

    What are the deployment modes of RustFS? How would you plan it?  

You can answer:  

    RustFS can be divided into three modes based on node and disk count: single-node single-disk (SNSD), single-node multi-disk (SNMD), and multi-node multi-disk (MNMD). This can be understood as SNSD, SNMD, and MNMD.  
    Single-node single-disk is suitable for learning and quick validation, such as starting RustFS with Docker on a single machine, mounting the /data/rustfs data directory, and then using mc or AWS CLI to validate Bucket creation, object upload/download, and data retention after container restart. This mode lacks node high availability and cannot be used directly for production.  
    Single-node multi-disk is suitable for validating multiple data directories or multiple disks on a single node, but it still has only one node. If the server fails, the entire object storage becomes unavailable, so it's also unsuitable for high availability production.  
    Multi-node multi-disk is closer to a production object storage cluster. Generally, multiple servers are planned, such as 4 RustFS nodes, each with at least one independent data disk, and Nginx or load balancer as a unified entry in front. Clients only access the unified HTTPS endpoint, and backend nodes communicate via HTTP in a trusted internal network.  
    In my experiment planning, I will use 10.0.0.51 to 10.0.0.54 as RustFS nodes, 10.0.0.55 as the client node, and 10.0.0.56 as the Nginx unified entry. The data directory is uniformly /data/rustfs, and external access uses https://s3.rustfs.localI don't know..  
    Before production, I won't just check if the container can run, but verify S3 API compatibility, Multipart Upload, SDK compatibility, large object upload, node failure recovery, disk capacity, HTTPS, permissions, logs, monitoring, backup migration, and version upgrades. Especially since RustFS is a new object storage solution, it's suitable to first do learning and pilot validation, and not recommended to directly replace mature production MinIO without pressure testing and recovery drills.  

---

## Twenty-four, Summary of This Article  

This article completed the learning of RustFS deployment modes:  

1. RustFS can be divided into SNSD, SNMD, and MNMD based on nodes and disk counts.  
2. SNSD is single-node single-disk, suitable for basic learning.  
3. SNMD is single-node multi-disk, suitable for single-node multi-disk validation.  
4. MNMD is multi-node multi-disk, closer to a production cluster.  
5. Single-node mode cannot directly represent production capacity.  
6. Multi-node cluster is not simply replicating multiple single-node containers.  
7. RustFS experiment planning uses 10.0.0.51-10.0.0.56.  
8. 10.0.0.51 can be used as a single-node.  
9. 10.0.0.51-10.0.0.54 can be used as cluster nodes.  
10. 10.0.0.55 is the client node.  
11. 10.0.0.56 is the Nginx / LB unified entry.  
12. Data directory is uniformly planned as /data/rustfs.  
13. Production recommends using independent data disks.  
14. Internal node communication can use HTTP in a trusted network.  
15. External client access must use HTTPS.  
16. Nginx / LB can unify certificates, logs, source control, and backend entry.  
17. Docker deployment is suitable for learning and quick validation.  
18. VM / bare-metal deployment is closer to traditional production evaluation.  
19. Images must be fixed versions; using latest is not recommended.  
20. For domestic network environments, it's recommended to synchronize images to Alibaba Cloud repository or Harbor.  
21. Before RustFS production, compatibility, pressure testing, failure drills, and recovery verification must be completed.  
22. The next article will enter RustFS single-node deployment practice: service startup, data directory, and access validation.  

---

## Twenty-five, Reference Documents  

RustFS official website:  

    https://rustfs.com/  

RustFS official documentation:  

    https://docs.rustfs.com/  

RustFS Docker installation documentation:  

    https://docs.rustfs.com/installation/docker/  

RustFS Linux installation documentation:  

    https://docs.rustfs.com/installation/linux/  

RustFS multi-node multi-disk installation documentation:  

    https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html  

RustFS GitHub:  

    https://github.com/rustfs/rustfs  

RustFS Docker Hub:  

    https://hub.docker.com/r/rustfs/rustfs  

AWS S3 API documentation:  

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html  

MinIO official documentation:  

    https://min.io/docs/minio/linux/index.html  

MinIO mc client documentation:  

    https://min.io/docs/minio/linux/reference/minio-mc.html  

Nginx official documentation:  

    https://nginx.org/en/docs/  

Docker official documentation:  

    https://docs.docker.com/