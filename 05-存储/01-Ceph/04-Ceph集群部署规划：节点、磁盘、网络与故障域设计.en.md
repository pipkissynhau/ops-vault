# Ceph Cluster Deployment Planning: Node, Disk, Network, and Fault Domain Design

Recommended Path: 05-Storage/01-Ceph/04-Ceph Cluster Deployment Planning: Node, Disk, Network, and Fault Domain Design.md

Tags: #Ceph #Distributed Storage #Deployment Planning #OSD #MON #MGR #CRUSH #Fault Domain #SRE #Ubuntu #RockyLinux

---

## I. Document Overview

This article is the fourth in the advanced SRE storage module for Ceph, focusing on the planning methods before deploying a Ceph cluster.

Deploying Ceph is not simply about executing an installation command. For production or near-production experimental environments, what truly matters is:

- How to plan the nodes
- How to distribute the roles
- How to use the disks
- How to divide the network
- How to configure the hostnames
- How to design the fault domains
- How to prepare domestic software repositories
- How to ensure compatibility between Ubuntu and Rocky Linux environments
- Whether it will be convenient for subsequent OSD scaling, fault recovery, and integration with Kubernetes

This article does not directly perform the cephadm bootstrap process but instead focuses on completing the pre-deployment planning. The actual installation and deployment steps will be covered in the next article:

    05-Ceph Cluster Deployment Practice: cephadm Basic Installation and Cluster Initialization.md

---

## II. Why Pre-Planning is Essential for Ceph Deployment

Ceph is a distributed storage system, and many issues arise not during installation but during the planning phase.

Common problems include:

- Too few nodes, resulting in insufficient high availability
- An unreasonable number of MON nodes deployed
- All OSDs concentrated on a few nodes
- Mixing system disks with data disks
- Not reserving nodes for scaling
- Lacking dedicated data disks
- Not having consistent hostnames
- Time synchronization issues causing MON failures
- Insufficient network bandwidth leading to slow recovery
- Unclear firewall port settings
- Unreasonable CRUSH fault domain design
- Inadequate capacity planning, causing the cluster to quickly reach nearfull status
- Failure during installation due to issues with domestic software repositories
- Mixing Ceph nodes with Kubernetes nodes, affecting the existing K8s environment

The essence of Ceph is not just about being able to install it but about ensuring that:

    The data can be securely distributed
    The cluster can recover after node failures
    The recovery process does not disrupt business operations
    Data can be reasonably rebalanced after scaling
    The storage cluster has clear operational boundaries

---

## III. General Principles for the Experimental Environment

### 3.1 Principle of Module Independence

Ceph modules should be tested independently, without relying on MinIO, Longhorn, or RustFS.

Independent planning for Ceph experimental nodes:

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

These nodes will be used for Longhorn and Kubernetes CSI integration experiments later on and will not be directly integrated into the Ceph storage cluster.

---

### 3.2 Experimental Implementation Method

Ceph experiments should be conducted using:

    VMs / Bare Metal with Multiple Nodes

Do not use a single-machine Docker environment to simulate a Ceph cluster.

Reasons:

    The core capability of Ceph is distributed storage.
    It is essential to understand its workings through multiple nodes, disks, fault domains, and replica recovery mechanisms.
    A single-machine simulation cannot effectively demonstrate OSD failures, node failures, CRUSH distribution, Backfill, or Rebalance processes.

---

### 3.3 Main Experimental System

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Explanation:

    Ubuntu 22.04.5 LTS is used for the main experiments and is consistent with the existing operational environment.
    Rocky Linux 9 is used as a supplement to demonstrate RHEL-based installation methods and can serve as a reference for production environments.

---

## IV. IP and Hostname Planning

### 4.1 Network Subnet

Experimental network:

    10.0.0.0/24

Existing nodes:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab / Jenkins / Harbor / Image Relay |
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.| ceph-node02 | /dev/sda | /dev/sdb, /dev/sdc |
| ceph-node03 | /dev/sda | /dev/sdb, /dev/sdc |
| ceph-node04 | /dev/sda | /dev/sdb, /dev/sdc, optional |

This results in:

    3 nodes x 2 OSDs = 6 OSDs

Or after expansion:

    4 nodes x 2 OSDs = 8 OSDs

---

### 6.3 Why Use Independent Disks for OSDs

Reasons for using independent disks for OSDs:

- Reduce the impact of system disk failures
- Facilitate disk replacement
- Make it easier to monitor OSD failures
- Simplify capacity monitoring
- Enable better performance isolation
- Facilitate future scale-out
- More in line with production environments

In a production environment, the planning of OSD disks directly affects:

- Available capacity
- Writing performance
- Recovery speed
- Difficulty in handling failures
- Data security

---

### 6.4 Disk Inspection Commands

To view disks:

    lsblk

For detailed disk information:

    fdisk -l

To view file systems:

    df -hT

To view block devices:

    blkid

To check disk models:

    lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL

To determine if a disk has already been partitioned:

    parted -l

---

### 6.5 Requirements for OSD Data Disks

Disks intended for use as OSDs should meet the following criteria:

- Not mounted
- Not formatted with any business file systems
- Do not contain important data
- Not used as system disks
- Not occupied by LVM or other services
- Each disk should correspond to one OSD

Before deployment, you can clean the disks:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

If there is a second data disk:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

Note:

    The above commands will erase the disk's signature and partition information.
    Always confirm that the disks are not in use for any critical operations before executing these commands in a production environment.

---

## Section 7: Capacity Planning

### 7.1 Raw Capacity vs. Usable Capacity

In Ceph, it is important to distinguish between:

    Raw Capacity: The total physical storage capacity of the device.
    Usable Capacity: The actual amount of space available for data storage after accounting for various factors.

For example:

    With 3 nodes, each node having 2 100GB OSD disks:
    Raw Capacity = 3 x 2 x 100GB = 600GB

If the pool has a replication factor of 3:

    Theoretical Usable Capacity ≈ 600GB / 3 = 200GB

It is important to note that:

    A higher replication factor increases data redundancy but reduces usable capacity.

---

### 7.2 Planning the Number of Replicas

The default number of replicas in experiments is:

    size = 3

This means that each object is stored in 3 copies.

Recommendations:

| Environment | Number of Replicas |
|---|---|
| Single-node testing | Not recommended for Ceph |
| Three-node experiment | 3 |
| General production use | 3 |
| Critical applications | 3 or consider more advanced solutions |
| Large-scale cold data storage | Consider using erasure coding |

Note:

    Setting size = 2 can save space but reduces fault tolerance.
    Setting size = 1 provides no redundancy and is only suitable for temporary testing, not for production use.

---

### 7.3 Planning min_size

min_size specifies the minimum number of available replicas required.

For example:

    If size = 3 and min_size = 2:

    This means that even when only 2 replicas are available, read and write operations will continue.
    If the number of replicas falls below 2, writes may be paused to maintain data consistency.

In production, min_size should not be lowered arbitrarily.

---

### 7.4 Capacity Level Planning

Common capacity levels in Ceph include:

- nearfull
- backfillfull
- full

It is important to avoid letting the cluster approach the "full" level during operation and maintenance.

Recommendations:

    When the capacity reaches 70%, start monitoring capacity growth.
    When it reaches 80%, plan for expansion or data cleanup.
    When it exceeds 85%, take immediate action as it poses a high risk.
    When it is nearfull/full, potential issues with writing and recovery may occur.

The specific thresholds depend on the cluster configuration and will be discussed in later operational maintenance sections.

---

## Section 8: Network Planning

### 8.1 Types```markdown
lsof \
telnet \
chrony \
ca-certificates \
gnupg \
software-properties-common \
lvm2

---

### 11.2 Rocky Linux 9

Additional Systems:

    Rocky Linux 9

Applicable to:

- Reference for RHEL-based production environments
- dnf/yum source configuration
- firewalld management
- SELinux considerations
- Using Podman as a container runtime for cephadm

Basic Tools:

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

Note:

    The official Ceph cephadm requires a container runtime, and Podman is commonly used on Rocky 9.
    Docker or Podman can also be used in Ubuntu environments; specific details will be provided in the subsequent deployment sections.

---

## Section XII: Domestic Software Source Planning

### 12.1 Ubuntu Alibaba Cloud Source

For Ubuntu 22.04, use the Alibaba Cloud Ubuntu source:

    https://mirrors.aliyun.com/ubuntu/

The configuration for the jammy source will be detailed in the subsequent deployment section.

Common Components:

    jammy
    jammy-updates
    jammy-security
    jammy-backports

---

### 12.2 Rocky Linux Alibaba Cloud Source

Rocky Linux uses the Alibaba Cloud Rocky Linux source:

    https://mirrors.aliyun.com/rockylinux/

After changing the repository for Rocky Linux 9, you need to execute:

    dnf makecache

Specific commands will be provided in the subsequent deployment section.

---

### 12.3 Ceph Alibaba Cloud Source

Ceph uses the Alibaba Cloud Ceph source:

    https://mirrors.aliyun.com/ceph/

Configuration Principle:

    Replace "download.ceph.com" in the official documentation with "mirrors.aliyun.com/ceph".

The following will be covered in the subsequent deployment section:

- Ubuntu apt Ceph source configuration
- Rocky Linux 9 dnf/yum Ceph source configuration
- cephadm installation
- ceph-common installation
- Methods for verifying domestic sources

---

## Section XIII: Time Synchronization Planning

### 13.1 Why Time Synchronization is Important

Ceph is quite sensitive to time synchronization, especially in scenarios involving MON and authentication.

If there is a significant difference in the time across nodes, it may lead to:

- Abnormalities in the MON quorum
- Authentication issues
- Disordered log timestamps
- Difficulty in troubleshooting
- Issues with certificate verification
- Incorrect assessment of cluster status

---

### 13.2 Ubuntu Time Synchronization

Install chrony:

    apt install -y chrony

Start it:

    systemctl enable --now chrony

Check its status:

    systemctl status chrony

View the synchronization sources:

    chronyc sources -v

Check the current time:

    timedatectl

---

### 13.3 Rocky Linux 9 Time Synchronization

Install chrony:

    dnf install -y chrony

Start it:

    systemctl enable --now chronyd

Check its status:

    systemctl status chronyd

View the synchronization sources:

    chronyc sources -v

Check the current time:

    timedatectl

---

### 13.4 Time Zone Setting

Set the time zone uniformly:

    timedatectl set-timezone Asia/Shanghai

Check the time zone setting:

    timedatectl

---

## Section XIV: SSH and Management Channels Planning

### 14.1 cephadm's Dependency on SSH

When managing Ceph nodes with cephadm, it is necessary to connect to other nodes via SSH.

Therefore, the following aspects need to be planned:

- Management node
- SSH passwordless login
- Use of root or a designated management user
- Hostname resolution for nodes
- Ensuring SSH port connectivity

Typically, the initial node is set as:

    ceph-node01

as the bootstrap node.

---

### 14.2 SSH Passwordless Login Planning

Generate keys on the ceph-node01:

    ssh-keygen -t rsa -b 4096

Copy the keys to other nodes:

    ssh-copy-id root@ceph-node01
    ssh-copy-id root@ceph-node02
    ssh-copy-id root@ceph-node03
    ssh-copy-id root@ceph-node04

Verify the connections:

    ssh root@ceph-node02 hostname
    ssh root@ceph-node03 hostname

Note:

    In a testing environment, you can use root for login.
    In a production environment, it is recommended to follow corporate security guidelines and use a controlled management user with minimal permissions.

---

## Section XV: Firewall and SELinux Planning

### 15.1 Ubuntu    │ /dev/sdb /dev/sdc    │ │ /dev/sdb /dev/sdc    │
    └──────────────────────┘ └──────────────────────┘

        Optional Expansion:

    ┌──────────────────────┐
    │ ceph-node04          │
    │ 10.0.0.34            │
    │ OSD / Expansion / Fault Simulation │
    │ /dev/sdb /dev/sdc    │
    └──────────────────────┘

---

## Section Eighteen: Ceph and Kubernetes Interface Boundary

The Ceph cluster operates independently:

    10.0.0.31-34

The Kubernetes cluster operates independently:

    10.0.0.20-22

Subsequent connection is achieved through CSI:

    Kubernetes PVC
        |
        v
    Ceph CSI
        |
        v
    Ceph RBD / CephFS
        |
        v
    Ceph OSD cluster

Note:

    The use of Ceph in Kubernetes does not mean that Ceph is deployed on Kubernetes nodes.
    In this experiment, the Ceph cluster and the K8s cluster remain separate.
    This setup closely reflects the mode of external storage clusters in production environments.

---

## Section Nineteen: Pre-Deployment Checklist

### 19.1 Node Inspection

| Inspection Item | Command |
|---|---|
| Hostname | hostname |
| IP Address | ip addr |
| Host Resolution | ping ceph-node02 |
| System Version | cat /etc/os-release |
| Kernel Version | uname -a |
| Time Synchronization | timedatectl |
| Disk List | lsblk |
    │ Port Usage | ss -lntp |
    │ Firewall Status | ufw status or firewall-cmd --state |

---

### 19.2 Disk Inspection

Confirm the data disk:

    lsblk

Ensure no disks are mounted:

    df -hT

Verify there are no old signatures:

    wipefs -n /dev/sdb

Check for any business data:

    fdisk -l /dev/sdb

---

### 19.3 Network Inspection

Node connectivity:

    ping -c 3 ceph-node01
    ping -c 3 ceph-node02
    ping -c 3 ceph-node03

SSH connectivity:

    ssh root@ceph-node02 hostname
    ssh root@ceph-node03 hostname

Port status:

    ss -lntp

---

### 19.4 Time Inspection

Check the current time:

    timedatectl

Verify chrony configuration:

    chronyc sources -v

Requirements:

    All nodes must have synchronized time.
    The time zones must be consistent.
    The time difference should not be significant.

---

## Section Twenty: Common Planning Errors

### 20.1 Deploying Ceph on a Single Machine

Error:

    Deploying Ceph on a single machine and assuming one understands distributed storage.

Issue:

    High availability of the MON cannot be demonstrated.
    Host failure domains cannot be effectively implemented.
    Node recovery from failures cannot be tested.
    The distribution of CRUSH replicas is not understood.

Correct approach:

    At least 3 nodes are required.

---

### 20.2 Using the System Disk as an OSD

Error:

    Using the system disk directly as an OSD data disk.

Issue:

    The system and storage components interfere with each other.
    This setup poses risks during fault simulations.
    Replacement of disks becomes complicated later on.
    It does not conform to production practices.

Correct approach:

    Keep the system disk separate from the OSD data disk.

---

### 20.3 All Replicas Located on the Same Node

Error:

    Focusing only on the number of replicas (e.g., 3) without considering failure domains.

Issue:

    A host failure can result in multiple replicas becoming unavailable simultaneously.

Correct approach:

    Implement host-level failure domains to ensure replicas are distributed across different nodes.

---

### 20.4 Ignoring Network Recovery Traffic

Error:

    Assuming that sufficient disk space is enough for Ceph to function properly.

Issue:

    OSD recovery and rebalancing processes generate significant network traffic.
    Insufficient network bandwidth can slow down the recovery process or even disrupt business operations.

Correct approach:

    In a test environment, a single network segment can be used. However, in a production environment, it is essential to separate the Public Network from the Cluster Network.

---

### 20.5 Not Planning for Domestic Software Sources

Error:

    Using default foreign software sources during deployment.

Issue:

    Slow download speeds.
    Possible installation failures.
    Unstable version availability.
    Inability to reproduce experimental results.

Correct approach:

    Use Alibaba Cloud’s Ubuntu, Rocky Linux, or Ceph mirror sources. The deployment instructions should clearly provide guidance on setting up these source