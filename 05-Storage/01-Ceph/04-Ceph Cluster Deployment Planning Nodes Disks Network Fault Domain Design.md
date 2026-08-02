# Ceph Cluster Deployment Planning: Nodes, Disks, Networks, and Failure Domain Design

Recommended path: 05-Storage/01-Ceph/04-Ceph Cluster Deployment Planning: Nodes, Disks, Networks, and Failure Domain Design.md

Tags: #Ceph #DistributedStorage #DeploymentPlanning #OSD #MON #MGR #CRUSH #FaultField #SRE #Ubuntu #RockyLinux

---

## I. Document Overview

This is the fourth article of the Ceph Advanced SRE Storage Module, focusing on methods for planning before Ceph cluster deployment.

Ceph deployment is not simply executing an installation command. For production or near-production experimental environments, what truly matters is:

- How to plan nodes
- How to distribute roles
- How to use disks
- How to divide networks
- How to configure hostnames
- How to design failure domains
- How to prepare domestic software sources
- How to ensure compatibility between Ubuntu and Rocky Linux environments
- Whether subsequent OSD expansion, fault recovery, and Kubernetes integration will be convenient

This article does not directly execute cephadm bootstrap, but rather completes the pre-deployment planning first. The actual installation and deployment will be covered in the next article:

    05-Ceph Cluster Deployment Practice: cephadm Basic Installation and Cluster Initialization.md

---

## II. Why Ceph Deployment Must Be Planned First

Ceph is a distributed storage system, and many issues do not appear after installation but are already hidden during the planning phase.

Common issues include:

- Too few nodes to demonstrate high availability
- Unreasonable MON deployment count
- All OSDs concentrated on a few nodes
- Mixing system disks and data disks
- No reserved expansion nodes
- No independent data disks
- No unified hostnames
- Time desynchronization causing MON anomalies
- Insufficient network bandwidth leading to slow recovery
- Unclear firewall port configurations
- Unreasonable CRUSH failure domain design
- Insufficient capacity planning, causing the cluster to reach nearfull quickly
- No consideration of domestic software sources, leading to installation failures
- Mixing Ceph and Kubernetes nodes, affecting existing K8s environments

The core of Ceph is not "being able to install it", but:

    Whether data can be securely distributed
    Whether the cluster can recover after node failures
    Whether the recovery process will overwhelm business operations
    Whether data can be reasonably rebalanced after expansion
    Whether the storage cluster has clear operational boundaries

---

## III. General Principles for Experimental Environment

### 3.1 Module Independence Principle

The Ceph module must be independently tested and not depend on MinIO, Longhorn, or RustFS.

Ceph experimental nodes must be independently planned:

    ceph-node01
    ceph-node02
    ceph-node03
    ceph-node04 (optional)
    ceph-client (optional)

Do not use existing Kubernetes nodes as Ceph OSD nodes.

Existing Kubernetes environment:

    10.0.0.20  k8s-master
    10.0.0.21  k8s-worker01
    10.0.0.22  k8s-worker02

These nodes will be used for Longhorn and Kubernetes CSI integration experiments later and will not be directly mixed into the Ceph storage cluster.

---

### 3.2 Experimental Implementation Method

Ceph experiments adopt:

    VM / Bare Metal Multi-Node

Do not use single-machine Docker to simulate a Ceph cluster.

Reasons:

    Ceph's core capability is distributed storage.
    It must be understood through multi-node, multi-disk, failure domains, replica recovery, etc.
    Single-machine simulation cannot adequately demonstrate OSD failures, node failures, CRUSH distribution, Backfill, and Rebalance.

---

### 3.3 Main Experimental System

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary systems:

    Rocky Linux 9

Notes:

    Ubuntu 22.04.5 LTS is used for main experiments, maintaining consistency with existing operations and maintenance experimental environments.
    Rocky Linux 9 is used for supplementary RHEL system installation methods, suitable for production reference.

---

## IV. IP and Hostname Planning

### 4.1 Network Subnet

Experimental network:

    10.0.0.0/24

Existing nodes:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab / Jenkins / Harbor / Image Transfer |
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Ceph uses an independent address range:

| IP | Hostname | Role | Notes |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD | Initial deployment node |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD | Storage node |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD | Storage node |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Failure Simulation | Optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Testing | Optional |

---

### 4.2 Minimum Experimental Topology

Minimum viable experimental topology:

    ceph-node01  10.0.0.31
    ceph-node02  10.0.0.32
    ceph-node03  10.0.0.33

Each node:

    1 system disk
    1 independent OSD data disk

Suitable for learning:

- cephadm initialization
- 3 MON high availability
- Basic OSD management
- Pools and PGs
- Basic RBD / CephFS / RGW experiments
- Single OSD failure recovery
- Observing single-node failure

---

### 4.3 Recommended Experimental Topology

Recommended experimental topology:

    ceph-node01  10.0.0.31
    ceph-node02  10.0.0.32
    ceph-node03  10.0.0.33
    ceph-node04  10.0.0.34

Each storage node:

    1 system disk
    2 independent OSD data disks

Suitable for learning:

- Multi-OSD data distribution
- Node expansion
- OSD replacement
- Backfill
- Rebalance
- CRUSH host failure domain
- Capacity expansion
- Failure simulation

---

### 4.4 /etc/hosts Planning

Each Ceph node is recommended to write a unified hosts file: /etc/hosts

10.0.0.31 ceph-node01  
10.0.0.32 ceph-node02  
10.0.0.33 ceph-node03  
10.0.0.34 ceph-node04  
10.0.0.35 ceph-client  

Configuration command example:

    cat >> /etc/hosts <<'EOF'  
    10.0.0.31 ceph-node01  
    10.0.0.32 ceph-node02  
    10.0.0.33 ceph-node03  
    10.0.0.34 ceph-node04  
    10.0.0.35 ceph-client  
    EOF  

Verification:

    ping -c 2 ceph-node01  
    ping -c 2 ceph-node02  
    ping -c 2 ceph-node03  

Production recommendation:

    It is recommended to use a stable DNS in production environments.  
    In experimental environments, /etc/hosts can be used for quick reproducibility.  

---

## Five. Node Role Planning  

### 5.1 Ceph Core Roles  

Common Ceph components:

| Component | Function |  
|---|---|  
| MON | Maintains cluster Map and Quorum |  
| MGR | Management, monitoring, Dashboard, Prometheus, cephadm orchestration |  
| OSD | Stores actual data |  
| MDS | CephFS metadata service |  
| RGW | S3 / Swift object storage gateway |  

---

### 5.2 Three-Node Basic Role Planning  

Recommended basic configuration:

| Node | MON | MGR | OSD | MDS | RGW |  
|---|---|---|---|---|---|  
| ceph-node01 | Yes | Yes | Yes | Optional | Optional |  
| ceph-node02 | Yes | Yes | Yes | Optional | Optional |  
| ceph-node03 | Yes | Yes | Yes | Optional | Optional |  

Notes:

    3 MONs can form a majority.  
    Multiple MGRs can form active / standby.  
    OSDs are distributed across 3 nodes, facilitating host-level fault domain.  
    MDS and RGW can be deployed separately for functional experiments later.  

---

### 5.3 Four-Node Expansion Planning  

Expanded node:

| Node | Role |  
|---|---|  
| ceph-node04 | OSD / Expansion / Fault Simulation |  

Purpose:

- Add new OSD  
- Simulate node expansion  
- Simulate disk replacement  
- Observe Rebalance  
- Validate CRUSH data redistribution  

---

### 5.4 ceph-client Planning  

Optional client node:

| Node | Function |  
|---|---|  
| ceph-client | RBD, CephFS, RGW client testing |  

Can be used for:

- Mount RBD  
- Mount CephFS  
- Test RGW with s3cmd / mc  
- Simulate business client  
- Test network connectivity  
- Perform simple read/write verification  

---

## Six. Disk Planning  

### 6.1 Basic Disk Planning  

Each Ceph storage node is recommended to have at least:

| Disk | Purpose |  
|---|---|  
| /dev/sda | OS disk |  
| /dev/sdb | Ceph OSD data disk |  
| /dev/sdc | Ceph OSD data disk (optional) |  

Not recommended:

    Use the OS disk directly as an OSD data disk.  
    Simulate production OSDs under the OS disk directory.  
    Arbitrarily mix multiple OSDs on the same disk.  

---

### 6.2 Recommended Experimental Disk Planning  

Three nodes, two OSD data disks per node:

| Node | OS Disk | OSD Data Disks |  
|---|---|---|  
| ceph-node01 | /dev/sda | /dev/sdb, /dev/sdc |  
| ceph-node02 | /dev/sda | /dev/sdb, /dev/sdc |  
| ceph-node03 | /dev/sda | /dev/sdb, /dev/sdc |  
| ceph-node04 | /dev/sda | /dev/sdb, /dev/sdc (optional) |  

This configuration provides:

    3 nodes x 2 OSDs = 6 OSDs  

Or expanded to:

    4 nodes x 2 OSDs = 8 OSDs  

---

### 6.3 Why OSDs Should Use Independent Disks  

Reasons for using independent disks for OSDs:

- Reduce impact from OS disk failures  
- Facilitate disk replacement  
- Enable observation of OSD failures  
- Simplify capacity statistics  
- Enable performance isolation  
- Facilitate future expansion  
- Closer to production environments  

In production environments, OSD disk planning directly affects:

- Available capacity  
- Write performance  
- Recovery speed  
- Fault handling difficulty  
- Data security  

---

### 6.4 Disk Check Commands  

Check disks:

    lsblk  

Check disk details:

    fdisk -l  

Check file systems:

    df -hT  

Check block devices:

    blkid  

Check disk model:

    lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL  

Check if disks have partitions:

    parted -l  

---

### 6.5 OSD Data Disk Requirements  

Disks prepared for OSDs should meet:

- Unmounted  
- Unformatted business file system  
- No important data  
- Not used as OS disk  
- Not occupied by LVM or other services  
- Each disk corresponds to one OSD  

Before deployment, disks can be cleaned:

    wipefs -a /dev/sdb  
    sgdisk --zap-all /dev/sdb  

If there is a second data disk:

    wipefs -a /dev/sdc  
    sgdisk --zap-all /dev/sdc  

Note:

    These commands will clear disk signatures and partition information.  
    Production environments must confirm the disk is not a business disk before execution.  

---

## Seven. Capacity Planning  

### 7.1 Raw Capacity vs Usable Capacity  

In Ceph, it is necessary to distinguish:

    Raw Capacity: Original capacity  
    Usable Capacity: Available capacity  

Example:

    3 nodes  
    2 x 100GB OSD disks per node  
    Raw capacity = 3 x 2 x 100GB = 600GB  

If the Pool replica count is 3:

    Theoretical usable capacity ≈ 600GB / 3 = 200GB  

Note:

    Higher replica counts provide stronger data redundancy but lower usable capacity.  

---

### 7.2 Replica Count Planning  

Default replica count for experiments:

    size = 3  

Meaning:

    Each object retains 3 replicas.  

Recommendation: /think

| Environment | Replica Count |
|---|---|
| Single-node Testing | Not recommended for Ceph |
| Three-node Experiment | 3 |
| Production General Business | 3 |
| Critical Business | 3 or combined with more advanced solutions |
| Large-scale Cold Data | Can evaluate erasure coding |

Note:

    size = 2 can save capacity, but reduces fault tolerance.
    size = 1 lacks replica protection, only suitable for temporary testing, not recommended for production use.

---

### 7.3 min_size Planning

min_size represents the minimum number of available replicas.

Example:

    size = 3
    min_size = 2

Meaning:

    Normally store 3 replicas.
    Allow IO when only 2 replicas remain.
    If below 2, may stop writing to protect data consistency.

Do not arbitrarily reduce min_size in production.

---

### 7.4 Capacity Water Level Planning

Common Ceph capacity water levels:

- nearfull
- backfillfull
- full

Avoid clusters approaching full in operations.

Recommendation:

    Ceph clusters cannot be used to 99% like ordinary disks.
    Distributed storage needs to reserve space for recovery, backfill, and rebalance.

Planning suggestions:

| Water Level | Recommendation |
|---|---|
| 70% | Start paying attention to capacity growth |
| 80% | Prepare for expansion or cleanup |
| 85% or above | High risk, needs handling |
| nearfull/full | May affect writing and recovery |

Actual thresholds depend on cluster configuration, which will be expanded in the subsequent operations section.

---

## VIII. Network Planning

### 8.1 Ceph Network Types

Common Ceph networks are divided into:

| Network | Description |
|---|---|
| Public Network | Client access, MON communication, basic cluster communication |
| Cluster Network | OSD replica replication, recovery, backfill |

Experimental environments can initially use a single network:

    10.0.0.0/24

Production environments recommend splitting:

    Public Network: Client access
    Cluster Network: OSD replication and recovery

---

### 8.2 Single Network Experimental Planning

Experimental environment:

    public_network = 10.0.0.0/24

Advantages:

- Simple
- Easy to deploy
- Suitable for learning
- Reduces network configuration complexity

Disadvantages:

- Client access and OSD recovery share the same network
- Recovery/backfill may affect business IO
- Not suitable for direct use in high-load production environments

---

### 8.3 Dual Network Production Planning

Production recommendation:

    Public Network:
      Client access, MON, MGR, RGW

    Cluster Network:
      OSD replica replication, recovery, backfill

Illustration:

    Client / Kubernetes
          |
          | Public Network
          v
    Ceph MON / MGR / RGW / OSD
          |
          | Cluster Network
          v
    OSD <---- replication / recovery ----> OSD

Dual network value:

- Isolate client IO and recovery traffic
- Reduce impact on business access during recovery
- Improve stability of large-scale clusters
- Facilitate independent optimization of storage backend network

---

### 8.4 Network Bandwidth Recommendations

Experimental environment:

    1Gbps is suitable for learning

Production environment:

    At least 10Gbps is more reasonable

High-performance or large-scale environments:

    25Gbps / 40Gbps / 100Gbps

Ceph is very sensitive to network, especially:

- Replica writing
- OSD Recovery
- Backfill
- Rebalance
- Large object read/write
- RBD high IO scenarios

---

## IX. Port Planning

Common Ceph ports:

| Component | Port | Description |
|---|---|---|
| SSH | 22 | cephadm management node |
| MON v2 | 3300 | Ceph Messenger v2 |
| MON v1 | 6789 | Ceph Messenger v1 |
| OSD | 6800-7300 | OSD service port range |
| MGR Dashboard | 8443 | Dashboard HTTPS, depends on configuration |
| MGR Prometheus | 9283 | Prometheus metrics |
| RGW | 7480 | RGW default port, depends on configuration |
| RGW HTTPS | 443 | Exposed via reverse proxy or LB |

Experimental environments can initially allow relevant ports.

Production environments should minimize open ports:

- Internal OSD ports only allow access between Ceph nodes
- MON ports only allow access from clients and Ceph nodes
- Dashboard should not be directly exposed to the public internet
- RGW external access should go through HTTPS unified entry
- SSH should only allow access from the management network segment

---

## X. Fault Domain Design

### 10.1 What is a Fault Domain

A fault domain represents:

    A boundary of a group of resources that may be affected by the same failure.

Common fault domains:

| Fault Domain | Example |
|---|---|
| osd | Single disk |
| host | Single server |
| rack | Single rack |
| room | Single room |
| datacenter | Single data center |

Ceph uses CRUSH rules to control replica distribution across different fault domains.

---

### 10.2 Experimental Environment Fault Domain

Experimental environments use:

    host

That is:

    Spread replicas of the same object to different hosts as much as possible.

Example:

    Object A:
      Replica 1 -> ceph-node01
      Replica 2 -> ceph-node02
      Replica 3 -> ceph-node03

This can withstand single-node failures.

---

### 10.3 Incorrect Fault Domain Example

Incorrect example:

    Object A:
      Replica 1 -> ceph-node01 / osd.0
      Replica 2 -> ceph-node01 / osd.1
      Replica 3 -> ceph-node01 / osd.2

Although replica count is 3, all are on the same host.

If ceph-node01 fails, all 3 replicas become unavailable.

Correct understanding:

    Replica count is just the number of data redundancies.
    Fault domain determines whether replicas can truly resist corresponding level failures.

---

### 10.4 Production Fault Domain Recommendations

In production, select fault domains based on environment:

| Environment | Recommended Failure Domain |
|---|---|
| Small-scale 3-node | host |
| Multi-rack | rack |
| Multi-datacenter | room / datacenter |
| Cross-region | Requires more complex architecture, not recommended to solve with simple CRUSH rules |

Common Recommendations:

    At least achieve host-level failure domain.
    Multi-rack environments should plan rack-level failure domain.
    Do not only focus on size=3, confirm replicas are distributed across different failure domains.

---

## Eleven. Operating System Planning

### 11.1 Ubuntu Server 22.04.5 LTS

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Applicable to:

- Mainline deployment experiment
- cephadm initialization
- OSD management
- RBD / CephFS / RGW experiment
- Kubernetes CSI integration preparation

Basic tools:

    apt update

    apt install -y \
      vim \
      curl \
      wget \
      net-tools \
      iproute2 \
      lsof \
      telnet \
      chrony \
      ca-certificates \
      gnupg \
      software-properties-common \
      lvm2

---

### 11.2 Rocky Linux 9

Supplementary system:

    Rocky Linux 9

Applicable to:

- RHEL-based production environment reference
- dnf/yum repository configuration
- firewalld management
- SELinux notes
- Podman as cephadm container runtime

Basic tools:

    dnf install -y \
      vim \
      curl \
      wget \
      net-tools \
      iproute \
      lsof \
      telnet \
      chrony \
      ca-certificates \
      lvm2 \
      podman \
      python3

Notes:

    Ceph official cephadm requires container runtime support, Rocky 9 typically uses Podman.
    Ubuntu environment can also use Docker or Podman, specifics will be determined in the deployment guide.

---

## Twelve. Domestic Software Source Planning

### 12.1 Ubuntu Aliyun Source

Ubuntu 22.04 uses Aliyun Ubuntu source:

    https://mirrors.aliyun.com/ubuntu/

The deployment guide will write jammy source configuration.

Common components:

    jammy
    jammy-updates
    jammy-security
    jammy-backports

---

### 12.2 Rocky Linux Aliyun Source

Rocky Linux uses Aliyun Rocky Linux source:

    https://mirrors.aliyun.com/rockylinux/

After replacing repo on Rocky Linux 9, execute:

    dnf makecache

The deployment guide will expand specific commands.

---

### 12.3 Ceph Aliyun Source

Ceph uses Aliyun Ceph source:

    https://mirrors.aliyun.com/ceph/

Configuration principles:

    Replace official documentation's download.ceph.com with mirrors.aliyun.com/ceph.

The deployment guide will cover:

- Ubuntu apt Ceph source configuration
- Rocky Linux 9 dnf/yum Ceph source configuration
- cephadm installation
- ceph-common installation
- Domestic source verification methods

---

## Thirteen. Time Synchronization Planning

### 13.1 Why Time Synchronization is Important

Ceph is sensitive to time synchronization, especially in MON and authentication scenarios.

If node time differences are large, may cause:

- MON quorum anomalies
- Authentication anomalies
- Log time confusion
- Troubleshooting difficulties
- Certificate validation anomalies
- Cluster state judgment anomalies

---

### 13.2 Ubuntu Time Synchronization

Install chrony:

    apt install -y chrony

Start:

    systemctl enable --now chrony

Check status:

    systemctl status chrony

Check synchronization sources:

    chronyc sources -v

Check time:

    timedatectl

---

### 13.3 Rocky Linux 9 Time Synchronization

Install chrony:

    dnf install -y chrony

Start:

    systemctl enable --now chronyd

Check status:

    systemctl status chronyd

Check synchronization sources:

    chronyc sources -v

Check time:

    timedatectl

---

### 13.4 Time Zone Settings

Set unified time zone:

    timedatectl set-timezone Asia/Shanghai

Check:

    timedatectl

---

## Fourteen. SSH and Management Channel Planning

### 14.1 cephadm's Dependency on SSH

cephadm management node requires SSH connection to other nodes.

Therefore, plan:

- Management node
- SSH key-based authentication
- Root or specified management user
- Node hostname resolution
- SSH port connectivity

Typical initial node:

    ceph-node01

As bootstrap node.

---

### 14.2 SSH Key-based Authentication Planning

Generate key on ceph-node01:

    ssh-keygen -t rsa -b 4096

Distribute to other nodes:

    ssh-copy-id root@ceph-node01
    ssh-copy-id root@ceph-node02
    ssh-copy-id root@ceph-node03
    ssh-copy-id root@ceph-node04

Verify:

    ssh root@ceph-node02 hostname
    ssh root@ceph-node03 hostname

Notes:

    Experimental environment can use root.
    Production environment should follow enterprise security policies, using controlled management users and minimal privilege strategies.

---

## Fifteen. Firewall and SELinux Planning

### 15.1 Ubuntu Firewall

Ubuntu may not have ufw enabled by default.

Check:

    ufw status

Experimental environment can temporarily disable:

    ufw disable

Production environment recommends allowing ports, not recommended to directly disable.

### 15.2 Rocky Linux 9 firewalld

Rocky Linux 9 commonly uses firewalld by default.

Check:

    systemctl status firewalld

For experimental environments, temporarily disable:

    systemctl disable --now firewalld

For production environments, open the ports required by Ceph, for example:

    firewall-cmd --permanent --add-port=3300/tcp
    firewall-cmd --permanent --add-port=6789/tcp
    firewall-cmd --permanent --add-port=6800-7300/tcp
    firewall-cmd --permanent --add-port=8443/tcp
    firewall-cmd --permanent --add-port=9283/tcp
    firewall-cmd --reload

---

### 15.3 Rocky Linux 9 SELinux

Check SELinux:

    getenforce

For experimental environments, set to permissive:

    setenforce 0

Permanent configuration:

    sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

Production recommendation:

    It is not recommended to disable SELinux long-term in a simple manner.
    If the enterprise has security baseline requirements, the strategy should be configured according to Ceph's official recommendations and enterprise specifications.

---

## SixteenI don't know.Container Runtime Planning

### 16.1 cephadm and Container Runtime

cephadm manages Ceph components using containers.

Common container runtimes:

- Podman
- Docker

Rocky Linux 9 typically uses:

    Podman

Ubuntu environments can use Docker or Podman based on deployment methods.

Installation check:

    podman --version
    docker --version

---

### 16.2 Not Disrupting Kubernetes Underlying Runtime

Ceph experimental nodes are independent VMs / bare metal nodes, and Ceph OSD is not directly installed on existing Kubernetes nodes.

Therefore:

    No need to modify the containerd configuration of Kubernetes nodes.
    No disruption to existing K8s runtime.
    No impact on Longhorn's subsequent experiments.

This is an important boundary in advanced SRE experiment planning.

---

## SeventeenI don't know.Ceph Cluster Topology Diagram

Recommended three-node topology:

    ┌──────────────────────────────────────────────┐
    │                Ceph Experimental Cluster                  │
    │              Network: 10.0.0.0/24             │
    └──────────────────────────────────────────────┘

        ┌──────────────────────┐
        │ ceph-node01          │
        │ 10.0.0.31            │
        │ MON / MGR / OSD      │
        │ /dev/sdb /dev/sdc    │
        └──────────┬───────────┘
                   │
                   │
        ┌──────────┼───────────┐
        │          │           │
        v          v           v
    ┌──────────────────────┐ ┌──────────────────────┐
    │ ceph-node02          │ │ ceph-node03          │
    │ 10.0.0.32            │ │ 10.0.0.33            │
    │ MON / MGR / OSD      │ │ MON / MGR / OSD      │
    │ /dev/sdb /dev/sdc    │ │ /dev/sdb /dev/sdc    │
    └──────────────────────┘ └──────────────────────┘

        Optional expansion:

    ┌──────────────────────┐
    │ ceph-node04          │
    │ 10.0.0.34            │
    │ OSD / Expansion / Fault Simulation │
    │ /dev/sdb /dev/sdc    │
    └──────────────────────┘

---

## EighteenI don't know.Ceph and Kubernetes Integration Boundary

Ceph cluster runs independently:

    10.0.0.31-34

Kubernetes cluster runs independently:

    10.0.0.20-22

Future integration via CSI:

    Kubernetes PVC
        |
        v
    Ceph CSI
        |
        v
    Ceph RBD / CephFS
        |
        v
    Ceph OSD Cluster

Note:

    Kubernetes using Ceph does not mean Ceph is deployed on Kubernetes nodes.
    In this experiment, the Ceph cluster and K8s cluster remain independent.
    This approach better aligns with the external storage cluster model in production.

---

## NineteenI don't know.Pre-Deployment Checklist

### 19.1 Node Check

| Check Item | Command |
|---|---|
| Hostname | hostname |
| IP Address | ip addr |
| Hosts Resolution | ping ceph-node02 |
| System Version | cat /etc/os-release |
| Kernel Version | uname -a |
| Time Synchronization | timedatectl |
| Disk List | lsblk |
| Port Usage | ss -lntp |
| Firewall Status | ufw status or firewall-cmd --state |

fdisk -l /dev/sdb

---

### 19.3 Network Check

Node Communication:

    ping -c 3 ceph-node01
    ping -c 3 ceph-node02
    ping -c 3 ceph-node03

SSH Communication:

    ssh root@ceph-node02 hostname
    ssh root@ceph-node03 hostname

Port Reservation:

    ss -lntp

---

### 19.4 Time Check

Check Time:

    timedatectl

Check chrony:

    chronyc sources -v

Requirements:

    All nodes must have normal time synchronization.
    Time zones must be unified.
    Time differences cannot be too large.

---

## Twenty, Common Planning Errors

### 20.1 Deploying Ceph on a Single Machine

Error:

    Deploying Ceph on a single machine and assuming you understand distributed storage.

Issues:

    Cannot demonstrate MON high availability.
    Cannot demonstrate host fault domain.
    Cannot demonstrate node failure recovery.
    Cannot understand CRUSH replica distribution.

Correct Way:

    At least 3 nodes.

---

### 20.2 Using System Disk as OSD Data Disk

Error:

    Directly using the system disk as an OSD data disk.

Issues:

    System and storage interfere with each other.
    Fault simulation is dangerous.
    Disk replacement is unclear.
    Does not conform to production practices.

Correct Way:

    Separate system disk and OSD data disk.

---

### 20.3 All Replicas on the Same Node

Error:

    Only checking size=3 without considering CRUSH fault domain.

Issues:

    Host failure may cause multiple replicas to become unavailable simultaneously.

Correct Way:

    Use host-level fault domain.
    Ensure replicas are spread across different nodes.

---

### 20.4 Ignoring Network Recovery Traffic

Error:

    Assuming Ceph only needs sufficient disk space.

Issues:

    OSD failure recovery and Rebalance generate large network traffic.
    Insufficient network capacity leads to slow recovery and may affect business IO.

Correct Way:

    Single network is acceptable in experimental environments.
    Production environments should separate Public / Cluster networks.

---

### 20.5 Not Planning Domestic Sources

Error:

    Using default foreign sources during deployment.

Issues:

    Slow downloads.
    Installation failures.
    Unstable version retrieval.
    Irreproducible experimental process.

Correct Way:

    Use Alibaba Cloud Ubuntu / Rocky / Ceph mirror sources.
    Explicitly document source configuration methods in deployment instructions.

---

## Twenty-one, Production Environment Considerations

1. Ceph should start with at least three nodes for meaningful deployment.
2. MON is recommended to have 3 or 5 nodes; 1 or 2 nodes are not advised.
3. OSD should use independent data disks.
4. System disk and OSD data disk should not be mixed.
5. Replica count is generally 3; size=2 is cautiously used in production.
6. Fault domain should at least be at the host level.
7. In multi-rack environments, rack-level fault domain should be considered.
8. Production environments should plan Public Network and Cluster Network.
9. Capacity should not approach full long-term; recovery space must be reserved.
10. All nodes must have time synchronization.
11. Ceph cluster nodes and Kubernetes nodes should have clear boundaries.
12. Dashboard and RGW external access should be controlled through secure entry points.
13. Firewall and SELinux cannot be ignored; production environments should follow security baselines.
14. Domestic environments must plan software sources and mirror sources in advance.
15. Confirm container runtime availability before cephadm deployment.

---

## Twenty-two, Interview Answer Strategy

If asked in an interview:

    How to plan Ceph cluster deployment before deployment?

You can answer:

    Before deploying Ceph, planning nodes, disks, networks, and fault domains is essential.
    Nodes: At least 3 nodes are recommended to deploy 3 MONs for Quorum, and OSDs can be distributed across different hosts to demonstrate host-level fault domains.
    Disks: System disk and OSD data disk must be separated; generally, one independent disk corresponds to one OSD. System disk should not be used directly as OSD.
    Networks: Single network is acceptable in experimental environments; production environments should separate Public Network and Cluster Network to isolate client access and OSD replica recovery traffic.
    Fault domains: Do not only focus on replica count but also ensure replicas are distributed across different hosts, racks, or data centers. Three replicas on the same machine cannot withstand host failures.
    Operating system: Use Ubuntu 22.04 or Rocky Linux 9. In domestic environments, pre-configure domestic sources for Ubuntu, Rocky, and Ceph, such as Alibaba Cloud mirrors, to avoid installation failures due to foreign source access.
    Additionally, plan time synchronization, SSH keyless login, firewall, SELinux, container runtime, and Dashboard/RGW port access policies. Ceph is a data system; planning is more important than installation commands.

---

## Twenty-three, Summary of This Article

This article mainly summarizes key planning points for Ceph deployment:

1. Ceph modules use VM / bare metal multi-node independent experiments.
2. Ceph does not directly use existing Kubernetes nodes.
3. Experimental subnet uses 10.0.0.0/24.
4. Ceph nodes are planned as 10.0.0.31-34.
5. Main experimental system uses Ubuntu Server 22.04.5 LTS.
6. Supplement Rocky Linux 9 installation methods.
7. OSD recommends using independent data disks, not mixed with system disks.
8. Experimental environments can use single network; production environments should separate Public / Cluster networks.
9. Fault domains should at least be at the host level.
10. Domestic environments must plan Ubuntu, Rocky, and Ceph software sources.
11. Time synchronization, SSH, firewall, SELinux, and container runtime are mandatory pre-deployment checks.
12. Ceph deployment is not just executing cephadm; it requires prior architecture, capacity, network, and fault domain planning.

---

## Twenty-four, Reference Documents

Ceph Official Documentation Home:

    https://docs.ceph.com/

Cephadm Installation Documentation:

    https://docs.ceph.com/en/latest/cephadm/install/

Ceph Hardware Recommendations:

    https://docs.ceph.com/en/reef/start/hardware-recommendations/

Ceph Network Configuration:

    https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/

Ceph CRUSH Map:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph OSD Management:

    https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/

Alibaba Cloud Ubuntu Mirror:

    https://developer.aliyun.com/mirror/ubuntu

Alibaba Cloud Rocky Linux Mirror:

    https://developer.aliyun.com/mirror/rockylinux

Alibaba Cloud Ceph Mirror:

    https://developer.aliyun.com/mirror/ceph