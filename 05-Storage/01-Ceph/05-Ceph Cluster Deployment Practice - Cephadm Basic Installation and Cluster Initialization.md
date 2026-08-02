# Ceph Cluster Deployment Practice: cephadm Basic Installation and Cluster Initialization

Suggested Path: 05-Storage/01-Ceph/05-Ceph Cluster Deployment Practice: cephadm Basic Installation and Cluster Initialization.md

Tags: #Ceph #cephadm #DistributedStorage #ClusterDeployment #Ubuntu #RockyLinux #OSD #MON #MGR #SRE #DomesticSources #AdvancedSre

---

## I. Document Overview

This article is the fifth in the Ceph Advanced SRE Storage Module series, focusing on completing a reproducible Ceph multi-node cluster initialization experiment.

This article is not merely a record of installation commands, but organized according to advanced SRE operational standards:

- Clearly define experiment objectives
- Clearly define node planning
- Clearly define disk planning
- Clearly define operating system
- Clearly define domestic software sources
- Clearly define prerequisite checks
- Clearly define deployment steps
- Clearly define expected outputs
- Clearly define verification commands
- Clearly define common fault diagnosis
- Clearly define cleanup and rollback boundaries
- Clearly define production environment considerations

Main experiment system:

    Ubuntu Server 22.04.5 LTS

Production reference systems:

    Rocky Linux 9

Ceph deployment method:

    cephadm

Ceph experiment main version:

    Ceph Reef

Domestic source strategy:

    Ubuntu base source: Alibaba Cloud Ubuntu source
    Rocky Linux base source: Alibaba Cloud Rocky Linux source
    Ceph software source: Alibaba Cloud Ceph source

---

## II. Experiment Objectives

After completing this article, the following objectives should be achieved:

1. Initialize a basic Ceph cluster on 3 VM/bare-metal nodes.
2. Complete the first node initialization using cephadm bootstrap.
3. Add ceph-node02 and ceph-node03 to the Ceph cluster.
4. Deploy 3 MONs to form a basic quorum.
5. Deploy multiple MGRs to form active/standby configuration.
6. Add independent data disks as OSDs for each node.
7. Use ceph -s to verify cluster health status.
8. Use ceph orch commands to view hosts, services, devices, and OSDs.
9. Access Ceph Dashboard.
10. Create test pools and perform basic validation through rados object writes.
11. Master the troubleshooting paths for bootstrap, node addition, OSD addition, and Dashboard access failures.
12. Clearly identify which commands are high-risk and cannot be executed arbitrarily in production environments.

---

## III. Experiment Topology

### 3.1 Node Planning

Experiment network segment:

    10.0.0.0/24

Existing environment:

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab / Jenkins / Harbor / Image Transfer |
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Ceph experiment nodes:

| IP | Hostname | Role | Notes |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD | First node, execute bootstrap |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD | Second storage node |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD | Third storage node |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation | Optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Testing | Optional |

Minimum experiment nodes for this article:

    ceph-node01
    ceph-node02
    ceph-node03

---

### 3.2 Disk Planning

Each Ceph node should have at least:

| Disk | Purpose |
|---|---|
| /dev/sda | System disk |
| /dev/sdb | Ceph OSD data disk |
| /dev/sdc | Ceph OSD data disk (optional) |

This article defaults to at least one OSD data disk per node:

    /dev/sdb

If each node has two OSD data disks, both can be used:

    /dev/sdb
    /dev/sdc

Important reminder:

    OSD data disks will be taken over by Ceph.
    Confirm disks have no business data before adding OSDs.
    Do not add system disk /dev/sda to Ceph OSD.
    wipefs and sgdisk will clear disk signatures and partition information, which are high-risk operations.

---

### 3.3 Network Topology

The experiment environment uses a single network:

    10.0.0.0/24

Topology diagram:

    ┌───────────────────────────────────────────────┐
    │                10.0.0.0/24                     │
    └───────────────────────────────────────────────┘

┌──────────────────────┐
        │ ceph-node01          │
        │ 10.0.0.31            │
        │ MON / MGR / OSD      │
        │ /dev/sdb             │
        └──────────┬───────────┘
                   │
        ┌──────────┼───────────┐
        │          │           │
        v          v           v
    ┌──────────────────────┐ ┌──────────────────────┐
    │ ceph-node02          │ │ ceph-node03          │
    │ 10.0.0.32            │ │ 10.0.0.33            │
    │ MON / MGR / OSD      │ │ MON / MGR / OSD      │
    │ /dev/sdb             │ │ /dev/sdb             │
    └──────────────────────┘ └──────────────────────┘

Production Environment Recommendations:

    Public Network: Client access, MON, MGR, RGW
    Cluster Network: OSD replica replication, Recovery, Backfill

This article simplifies the experiment by using a single network.

---

## Four. Experiment Steps Overview

Complete workflow:

    1. Configure hostname on all nodes
    2. Configure /etc/hosts on all nodes
    3. Configure time synchronization on all nodes
    4. Configure system base repositories on all nodes
    5. Install base dependencies on all nodes
    6. Configure Ceph domestic repositories on all nodes
    7. Install cephadm and ceph-common on the first node
    8. Execute cephadm bootstrap on the first node
    9. Verify ceph -s
    10. Distribute cephadm SSH public key to other nodes
    11. Add ceph-node02, ceph-node03
    12. Deploy 3 MONs
    13. Deploy multiple MGRs
    14. Check available disks
    15. Clean OSD data disks
    16. Manually add OSD
    17. Verify OSD up/in
    18. Verify PG active+clean
    19. Access Dashboard
    20. Create test Pool and write objects for verification
    21. Clean test resources

---

## Five. Pre-Deployment Checklist

Before formal deployment, it is recommended to verify each item.

### 5.1 Node Check

Execute on all nodes:

    hostname
    ip addr
    cat /etc/os-release
    uname -a
    timedatectl
    lsblk
    df -hT
    ss -lntp

Key verification:

| Check Item | Requirement |
|---|---|
| Hostname | Matches the plan |
| IP Address | Matches the plan |
| System Version | Ubuntu 22.04.5 or Rocky Linux 9 |
| Time Synchronization | Normal |
| System Disk | Not used for OSD |
| Data Disk | Exists and has no business data |
| Port Usage | 3300, 6789, 6800-7300, 8443, 9283 not occupied |

---

### 5.2 Network Check

Execute on ceph-node01:

    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

Execute after configuring hosts:

    ping -c 3 ceph-node02
    ping -c 3 ceph-node03

Expected:

    Can resolve hostnames normally
    Can ping through normally

If ping fails, do not proceed with Ceph deployment until troubleshooting:

- IP configuration
- Gateway configuration
- Firewall
- Virtual network
- Hostname resolution
- Routing issues

---

### 5.3 Disk Check

Execute on all nodes:

    lsblk

Expected example:

    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    sda      8:0    0   50G  0 disk
    ├─sda1   8:1    0    1G  0 part /boot
    └─sda2   8:2    0   49G  0 part /
    sdb      8:16   0  100G  0 disk

Confirm /dev/sdb is not mounted:

    df -hT

Check disk signature:

    wipefs -n /dev/sdb

Check for LVM:

    pvs
    vgs
    lvs

Requirements:

    /dev/sdb has no business data
    /dev/sdb is not mounted
    /dev/sdb is not a system disk
    /dev/sdb can be taken over by Ceph

---

## Six. All Nodes: Configure Hostname

Execute on ceph-node01:

    hostnamectl set-hostname ceph-node01

Execute on ceph-node02:

    hostnamectl set-hostname ceph-node02

Execute on ceph-node03:

    hostnamectl set-hostname ceph-node03

Optional ceph-node04:

    hostnamectl set-hostname ceph-node04

Verification:

    hostname

Expected:

    ceph-node01
    ceph-node02
    ceph-node03

---

## Seven. All Nodes: Configure hosts

Execute on all Ceph nodes:

    cat >> /etc/hosts <<'EOF'
    10.0.0.31 ceph-node01
    10.0.0.32 ceph-node02
    10.0.0.33 ceph-node03
    10.0.0.34 ceph-node04
    10.0.0.35 ceph-client
    EOF

Verification:

    getent hosts ceph-node01
    getent hosts ceph-node02
    getent hosts ceph-node03

Expected output similar to:

10.0.0.31      ceph-node01
    10.0.0.32      ceph-node02
    10.0.0.33      ceph-node03

Continue verification:

    ping -c 2 ceph-node01
    ping -c 2 ceph-node02
    ping -c 2 ceph-node03

---

## Eight. All Nodes: Time Synchronization

### 8.1 Ubuntu 22.04

All Ubuntu nodes execute:

    apt update
    apt install -y chrony

    systemctl enable --now chrony

Check status:

    systemctl status chrony

Check synchronization sources:

    chronyc sources -v

Check timezone:

    timedatectl

Set timezone:

    timedatectl set-timezone Asia/Shanghai

Expected:

    chrony service active
    timedatectl shows Asia/Shanghai
    System clock synchronized: yes

---

### 8.2 Rocky Linux 9

All Rocky 9 nodes execute:

    dnf install -y chrony

    systemctl enable --now chronyd

Check status:

    systemctl status chronyd

Check synchronization sources:

    chronyc sources -v

Set timezone:

    timedatectl set-timezone Asia/Shanghai

Check:

    timedatectl

---

### 8.3 Why Time Synchronization is Important

Ceph is sensitive to time synchronization.

Time anomalies may cause:

- MON quorum anomalies
- Authentication failures
- Confused log timestamps
- Dashboard certificate validation anomalies
- Difficulty in troubleshooting

Time synchronization must be confirmed before deployment.

---

## Nine. Ubuntu 22.04: Configure Alibaba Cloud Ubuntu Repository

This section is executed on Ubuntu Server 22.04.5 LTS nodes.

Backup original sources:

    cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F-%H%M%S)

Write Alibaba Cloud Ubuntu 22.04 sources:

    cat > /etc/apt/sources.list <<'EOF'
    deb https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
    deb https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
    deb https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
    deb https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
    EOF

Update cache:

    apt clean
    apt update

Install base dependencies:

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
      lvm2 \
      gdisk \
      jq \
      openssh-client

Verify:

    curl -I https://mirrors.aliyun.com/ubuntu/

Expected:

    HTTP/2 200
    Or returns normal HTTP response headers

---

## Ten. Ubuntu 22.04: Configure Alibaba Cloud Ceph Repository

### 10.1 Configure Ceph release key

Create keyrings directory:

    mkdir -p /etc/apt/keyrings

Import Ceph release key:

    wget -q -O- https://mirrors.aliyun.com/ceph/keys/release.asc \
      | gpg --dearmor \
      > /etc/apt/keyrings/ceph-release.gpg

Check:

    ls -l /etc/apt/keyrings/ceph-release.gpg

Expected:

    File exists, size not zero.

---

### 10.2 Configure Ceph Reef apt repository

Ubuntu 22.04 codename:

    jammy

Ceph Reef apt repository:

    debian-reef

Write repository:

    cat > /etc/apt/sources.list.d/ceph.list <<'EOF'
    deb [signed-by=/etc/apt/keyrings/ceph-release.gpg] https://mirrors.aliyun.com/ceph/debian-reef/ jammy main
    EOF

Update cache:

    apt update

Check cephadm:

    apt-cache policy cephadm

Check ceph-common:

    apt-cache policy ceph-common

Expected:

    Candidate shows Ceph Reef related versions.

Install:

    apt install -y cephadm ceph-common

Verify:

    cephadm version
    ceph --version

Expected output example:

    cephadm version 18.x.x
    ceph version 18.x.x reef ...

Note:

    Actual minor version depends on the current source version.
    Production environments should lock versions and establish upgrade strategies.

---

## Eleven. Rocky Linux 9: Configure Alibaba Cloud Rocky Repository

This section is for Rocky Linux 9 nodes.

Backup repo:

    mkdir -p /etc/yum.repos.d/backup-$(date +%F-%H%M%S)
    cp /etc/yum.repos.d/Rocky-*.repo /etc/yum.repos.d/backup-$(date +%F-%H%M%S)/

Replace with Alibaba Cloud Rocky repository: /think

sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak \
    /etc/yum.repos.d/Rocky-*.repo

Generate cache:

    dnf makecache

Install base dependencies:

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
      gdisk \
      jq \
      python3 \
      podman \
      openssh-clients

Verification:

    podman --version
    python3 --version

---

## Twelve、Rocky Linux 9: Configure Alibaba Cloud Ceph Source

### 12.1 Write Ceph Reef repo

Create Ceph repo:

    cat > /etc/yum.repos.d/ceph.repo <<'EOF'
    [ceph]
    name=Ceph Reef packages for $basearch
    baseurl=https://mirrors.aliyun.com/ceph/rpm-reef/el9/$basearch
    enabled=1
    gpgcheck=1
    type=rpm-md
    gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

    [ceph-noarch]
    name=Ceph Reef noarch packages
    baseurl=https://mirrors.aliyun.com/ceph/rpm-reef/el9/noarch
    enabled=1
    gpgcheck=1
    type=rpm-md
    gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
    EOF

Generate cache:

    dnf makecache

Check:

    dnf list cephadm
    dnf list ceph-common

Install:

    dnf install -y cephadm ceph-common

Verification:

    cephadm version
    ceph --version

---

## Thirteen、Rocky Linux 9: firewalld and SELinux

### 13.1 firewalld

Check status:

    systemctl status firewalld

In experimental environments, temporarily disable:

    systemctl disable --now firewalld

If not disabled, need to allow ports:

    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=3300/tcp
    firewall-cmd --permanent --add-port=6789/tcp
    firewall-cmd --permanent --add-port=6800-7300/tcp
    firewall-cmd --permanent --add-port=8443/tcp
    firewall-cmd --permanent --add-port=9283/tcp
    firewall-cmd --reload

Check:

    firewall-cmd --list-ports

---

### 13.2 SELinux

Check:

    getenforce

In experimental environments, temporarily set to permissive:

    setenforce 0

Permanently configure to permissive:

    sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

Check:

    grep '^SELINUX=' /etc/selinux/config

Production note:

    Not recommended to simply disable SELinux in production environments.
    If enterprise security baseline requires enabling SELinux, need to configure according to Ceph official recommendations and enterprise policies.

---

## Fourteen、Container Images and Domestic Network Optimization

### 14.1 cephadm Container Images

cephadm will run Ceph components as containers.

Common official images:

    quay.io/ceph/ceph

In domestic networks, pulling quay.io directly may fail or be slow.

---

### 14.2 Image Handling Recommendations

Recommended strategy:

    1. Use default method if can directly access quay.io.
    2. If access is slow or fails, first pull official images in a networked environment.
    3. Retag to Harbor or Alibaba Cloud image repository.
    4. Specify internal image address using --image when bootstraping.
    5. Do not arbitrarily modify existing Kubernetes node containerd configurations.

Note:

    This Ceph experimental node is an independent VM / bare metal node.
    Do not disrupt the underlying runtime of existing Kubernetes nodes 10.0.0.20/21/22.

---

### 14.3 Pre-pull Image Example

If using Podman:

    podman pull quay.io/ceph/ceph:v18

If using Docker:

    docker pull quay.io/ceph/ceph:v18

Check:

    podman images | grep ceph

Or:

    docker images | grep ceph

If already synchronized to internal repository, assume internal image as:

    registry.cn-hangzhou.aliyuncs.com/<namespace>/ceph:v18

Subsequent bootstrap can specify:

    --image registry.cn-hangzhou.aliyuncs.com/<namespace>/ceph:v18

---

## FifteenI don't know.Pre-check Before First Node Bootstrap

The following operations are executed on ceph-node01.

### 15.1 Confirm Basic Environment

    hostname
    ip addr
    cat /etc/os-release
    timedatectl
    cephadm version
    ceph --version
    lsblk
    df -hT
    ss -lntp

Expected: /think

hostname is ceph-node01  
IP includes 10.0.0.31  
cephadm is executable  
ceph command is executable  
/dev/sdb exists  
Time synchronization is normal  

---

### 15.2 Confirm ports are not in use  

Check:  

    ss -lntp | egrep '3300|6789|8443|9283'  

If ports are occupied, confirm the service using them first.  

---

### 15.3 Confirm container runtime  

Check Podman:  

    podman --version  

Or Docker:  

    docker --version  

cephadm requires a container runtime.  

---

## Sixteen, Execute cephadm bootstrap  

### 16.1 Basic bootstrap command  

Execute on ceph-node01:  

    cephadm bootstrap \
      --mon-ip 10.0.0.31 \
      --initial-dashboard-user admin \
      --initial-dashboard-password 'Ceph_Admin_ChangeMe_123' \
      --dashboard-password-noupdate  

Explanation:  

| Parameter | Description |  
|---|---|  
| --mon-ip | IP used by the first MON |  
| --initial-dashboard-user | Initial Dashboard user |  
| --initial-dashboard-password | Initial Dashboard password |  
| --dashboard-password-noupdate | Do not enforce password change on first login |  

Example password:  

    Ceph_Admin_ChangeMe_123  

Production requirements:  

    Production environments must use more complex passwords.  
    Passwords should be stored in a controlled password management system.  
    Should not be written to public repositories.  

---

### 16.2 Specify Ceph image for bootstrap  

If specifying an image:  

    cephadm bootstrap \
      --mon-ip 10.0.0.31 \
      --image quay.io/ceph/ceph:v18 \
      --initial-dashboard-user admin \
      --initial-dashboard-password 'Ceph_Admin_ChangeMe_123' \
      --dashboard-password-noupdate  

If using an internal registry:  

    cephadm bootstrap \
      --mon-ip 10.0.0.31 \
      --image registry.cn-hangzhou.aliyuncs.com/<namespace>/ceph:v18 \
      --initial-dashboard-user admin \
      --initial-dashboard-password 'Ceph_Admin_ChangeMe_123' \
      --dashboard-password-noupdate  

---

### 16.3 Expected bootstrap output  

After successful bootstrap, you'll typically see output like:  

    Ceph Dashboard is now available at:  

             URL: https://ceph-node01:8443/  
            User: admin  
        Password: Ceph_Admin_ChangeMe_123  

    You can access the Ceph CLI with:  

        sudo /usr/sbin/cephadm shell --fsid <fsid> -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring  

The following files will also be generated:  

    /etc/ceph/ceph.conf  
    /etc/ceph/ceph.client.admin.keyring  
    /etc/ceph/ceph.pub  
    /etc/ceph/cephadm-ssh-key  

Check:  

    ls -l /etc/ceph/  

Expected at least:  

    ceph.conf  
    ceph.client.admin.keyring  
    ceph.pub  

---

## Seventeen, Basic verification after bootstrap  

### 17.1 Check cluster status  

    ceph -s  

If the host cannot execute directly, enter shell:  

    cephadm shell  

Then execute:  

    ceph -s  

Initially, you might see:  

    health: HEALTH_WARN  

Common reasons:  

    OSD count is insufficient  
    Only one MON  
    No other nodes added  
    No OSDs added  

This is normal during initial phase and does not indicate deployment failure.  

---

### 17.2 Expected initial status  

After bootstrap, before adding other nodes and OSDs, you might see:  

    cluster:  
      health: HEALTH_WARN  

    services:  
      mon: 1 daemons, quorum ceph-node01  
      mgr: ceph-node01(active)  
      osd: 0 osds: 0 up, 0 in  

This is a reasonable state.  

After adding nodes and OSDs, it will normalize.  

---

### 17.3 Check MGR service  

    ceph mgr services  

Possible output:  

    {
        "dashboard": "https://ceph-node01:8443/",
        "prometheus": "http://ceph-node01:9283/"
    }

---

## Eighteen, Configure cephadm SSH management capability  

### 18.1 Check cephadm public key  

Execute on ceph-node01:  

    cat /etc/ceph/ceph.pub  

---

### 18.2 Distribute public key to other nodes  

    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node02  
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node03  

Optional:  

    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node04  

---

### 18.3 Verify cephadm SSH connection

ssh -i /etc/ceph/cephadm-ssh-key root@ceph-node02 hostname
    ssh -i /etc/ceph/cephadm-ssh-key root@ceph-node03 hostname

Expected output:

    ceph-node02
    ceph-node03

If failed, check:

- Whether root is allowed SSH
- Whether sshd is running
- Whether /etc/hosts is correct
- Whether the key is successfully distributed
- Whether firewall allows port 22
- Whether the target node has python3 installed

---

## 19. Adding Ceph Hosts

### 19.1 Adding ceph-node02

    ceph orch host add ceph-node02 10.0.0.32

Expected output:

    Added host 'ceph-node02' with addr '10.0.0.32'

---

### 19.2 Adding ceph-node03

    ceph orch host add ceph-node03 10.0.0.33

Expected output:

    Added host 'ceph-node03' with addr '10.0.0.33'

---

### 19.3 Viewing Host List

    ceph orch host ls

Expected similar output:

    HOST          ADDR        LABELS  STATUS
    ceph-node01   10.0.0.31
    ceph-node02   10.0.0.32
    ceph-node03   10.0.0.33

---

### 19.4 Adding Labels

Add labels to the node:

    ceph orch host label add ceph-node01 mon
    ceph orch host label add ceph-node02 mon
    ceph orch host label add ceph-node03 mon

    ceph orch host label add ceph-node01 mgr
    ceph orch host label add ceph-node02 mgr
    ceph orch host label add ceph-node03 mgr

    ceph orch host label add ceph-node01 osd
    ceph orch host label add ceph-node02 osd
    ceph orch host label add ceph-node03 osd

View:

    ceph orch host ls

---

## 20. Deploying MON

### 20.1 Deploying 3 MONs

    ceph orch apply mon --placement="ceph-node01 ceph-node02 ceph-node03"

View MON service:

    ceph orch ps --daemon_type mon

View MON status:

    ceph mon stat

View quorum:

    ceph quorum_status --format json-pretty

---

### 20.2 Expected Results

    mon: 3 daemons, quorum ceph-node01,ceph-node02,ceph-node03

If only 1 MON is present, wait and check again:

    watch -n 3 'ceph -s'

If deployment to other nodes still fails, check:

    ceph orch ps
    ceph orch host ls
    ceph health detail

---

## 21. Deploying MGR

### 21.1 Deploying Multiple MGRs

    ceph orch apply mgr --placement="ceph-node01 ceph-node02 ceph-node03"

View:

    ceph orch ps --daemon_type mgr
    ceph mgr stat

---

### 21.2 Expected Results

    active_name: ceph-node01
    num_standby: 2

Or similar:

    mgr: ceph-node01(active), standbys: ceph-node02, ceph-node03

Note:

    MGR has active and standby roles.
    Dashboard, Prometheus, ceph orch, etc., depend on MGR.

---

## 22. Checking Available Disks

### 22.1 Viewing All Devices

    ceph orch device ls

View detailed information:

    ceph orch device ls --wide

Expected example:

    HOST         PATH      TYPE  DEVICE ID  SIZE   AVAILABLE  REFRESHED  REJECT REASONS
    ceph-node01 /dev/sdb  hdd              100G   Yes
    ceph-node02 /dev/sdb  hdd              100G   Yes
    ceph-node03 /dev/sdb  hdd              100G   Yes

If AVAILABLE is No, check REJECT REASONS.

---

### 22.2 Common Unavailable Reasons

| Reject Reason | Description |
|---|---|
| Has a FileSystem | Disk has a file system |
| LVM detected | Disk is used by LVM |
| mounted | Disk is mounted |
| locked | Device is occupied |
| Insufficient space | Insufficient capacity |
| not empty | Disk is not empty |

---

## 23. Cleaning OSD Data Disks

### 23.1 High-Risk Warning

The following commands will clean disk signatures and partition information.

Must confirm:

    Target disk is not a system disk.
    Target disk is not a business disk.
    Target disk has no data to preserve.

View disks:

    lsblk

Confirm /dev/sdb is not a system disk.

---

### 23.2 Clean /dev/sdb on Each Node

On ceph-node01:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

On ceph-node02:

wipefs -a /dev/sdb
sgdisk --zap-all /dev/sdb

On ceph-node03 execute:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

If /dev/sdc exists, it can also be wiped:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

---

### 23.3 Rechecking

On ceph-node01 execute:

    ceph orch device ls --wide

Expected output:

    /dev/sdb AVAILABLE is Yes

---

## Twenty-Four, Manual OSD Addition

### 24.1 Why Manual Addition

It is recommended to manually add OSDs in experiments:

    ceph orch daemon add osd ceph-node01:/dev/sdb

Not recommended to start with:

    ceph orch apply osd --all-available-devices

Reasons:

    Manual addition is safer.
    It clearly associates each OSD with a specific disk.
    Avoids accidentally adding unsuitable disks to Ceph.

---

### 24.2 Adding ceph-node01 OSD

    ceph orch daemon add osd ceph-node01:/dev/sdb

Expected output:

    Created osd(s) ...

---

### 24.3 Adding ceph-node02 OSD

    ceph orch daemon add osd ceph-node02:/dev/sdb

---

### 24.4 Adding ceph-node03 OSD

    ceph orch daemon add osd ceph-node03:/dev/sdb

---

### 24.5 If each node has /dev/sdc

Continue adding:

    ceph orch daemon add osd ceph-node01:/dev/sdc
    ceph orch daemon add osd ceph-node02:/dev/sdc
    ceph orch daemon add osd ceph-node03:/dev/sdc

---

## Twenty-Five, Verifying OSD Status

### 25.1 Checking OSD Services

    ceph orch ps --daemon_type osd

Expected output:

    osd.0
    osd.1
    osd.2

If each node has two disks, you might see:

    osd.0
    osd.1
    osd.2
    osd.3
    osd.4
    osd.5

---

### 25.2 Checking OSD Tree

    ceph osd tree

Expected output similar to:

    ID  CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
    -1         3.00000  root default
    -3         1.00000      host ceph-node01
     0    hdd  1.00000          osd.0             up   1.00000  1.00000
    -5         1.00000      host ceph-node02
     1    hdd  1.00000          osd.1             up   1.00000  1.00000
    -7         1.00000      host ceph-node03
     2    hdd  1.00000          osd.2             up   1.00000  1.00000

Key checks:

    OSD is up
    OSD is in
    OSDs are distributed across different hosts
    No OSDs are down

---

### 25.3 Checking Cluster Status

    ceph -s

Expected output approaching:

    cluster:
      health: HEALTH_OK

    services:
      mon: 3 daemons, quorum ceph-node01,ceph-node02,ceph-node03
      mgr: ceph-node01(active), standbys: ceph-node02, ceph-node03
      osd: 3 osds: 3 up, 3 in

    data:
      pgs: active+clean

If the cluster isn't HEALTH_OK immediately after adding OSDs, you can wait:

    watch -n 3 'ceph -s'

---

## Twenty-Six, Dashboard Access Verification

### 26.1 Checking Dashboard Address

    ceph mgr services

Expected output similar to:

    {
        "dashboard": "https://ceph-node01:8443/",
        "prometheus": "http://ceph-node01:9283/"
    }

Access:

    https://10.0.0.31:8443/

User:

    admin

Password:

    Ceph_Admin_ChangeMe_123

---

### 26.2 If Dashboard Is Unreachable

Check MGR:

    ceph mgr stat

Check modules:

    ceph mgr module ls

Enable Dashboard:

    ceph mgr module enable dashboard

Check services:

    ceph mgr services

Check ports:

    ss -lntp | grep 8443

Check firewall:

Ubuntu:

    ufw status

Rocky:

    firewall-cmd --list-ports

---

### 26.3 Changing Dashboard Password

    ceph dashboard ac-user-set-password admin

Enter a new password as prompted.

Production recommendations:

    Do not use the experimental password.
    Do not expose Dashboard to the public internet.
    Recommend accessing via VPN, bastion host, or internal network.
    HTTPS, access control, and auditing are required if exposed externally.

---

## Twenty-Seven, Basic Pool and Object Write Verification

### 27.1 Creating a Test Pool

    ceph osd pool create test-pool 32

Enable application type:

    ceph osd pool application enable test-pool rbd

Set replica count:

```
ceph osd pool set test-pool size 3
ceph osd pool set test-pool min_size 2

View:

    ceph osd pool ls
    ceph osd pool get test-pool size
    ceph osd pool get test-pool min_size
    ceph pg stat

---

### 27.2 Writing Test Objects

Create test file:

    echo "hello ceph" > /tmp/hello-ceph.txt

Write object:

    rados -p test-pool put hello-ceph /tmp/hello-ceph.txt

List objects:

    rados -p test-pool ls

Expected:

    hello-ceph

Read object:

    rados -p test-pool get hello-ceph /tmp/hello-ceph.out

View content:

    cat /tmp/hello-ceph.out

Expected:

    hello ceph

---

### 27.3 Deleting Test Objects

    rados -p test-pool rm hello-ceph

Confirm:

    rados -p test-pool ls

---

### 27.4 Deleting Test Pool

By default, Ceph may prohibit deleting a Pool.

Check:

    ceph config get mon mon_allow_pool_delete

Temporarily enable in experimental environment:

    ceph config set mon mon_allow_pool_delete true

Delete:

    ceph osd pool rm test-pool test-pool --yes-i-really-really-mean-it

Disable:

    ceph config set mon mon_allow_pool_delete false

Production warning:

    Deleting a Pool will delete all data within it.
    Do not casually enable mon_allow_pool_delete in production environments.
    Confirm that the business no longer uses it and complete backup and change approval before deletion.

---

## Twenty-Eight, Final Verification Checklist After Deployment

### 28.1 Cluster Status

    ceph -s

Expected:

    HEALTH_OK

If it's HEALTH_WARN, the reason should be explainable.

---

### 28.2 MON

    ceph mon stat
    ceph quorum_status --format json-pretty

Expected:

    3 MONs
    quorum contains ceph-node01, ceph-node02, ceph-node03

---

### 28.3 MGR

    ceph mgr stat
    ceph orch ps --daemon_type mgr

Expected:

    1 active
    At least 1 standby

---

### 28.4 OSD

    ceph osd tree
    ceph osd stat
    ceph osd df

Expected:

    OSD up/in
    OSD distributed across different nodes
    Capacity normal

---

### 28.5 PG

    ceph pg stat

Expected:

    active+clean

---

### 28.6 Service Orchestration

    ceph orch host ls
    ceph orch ls
    ceph orch ps

Expected:

    Hosts online
    Services normal
    No abnormal daemons

---

### 28.7 Dashboard

    ceph mgr services

Expected:

    Dashboard address available
    Browser accessible
    Admin can log in

---

## Twenty-Nine, Common Faults and Troubleshooting

### 29.1 Bootstrap Image Pull Failure

Symptoms:

    Failed to pull container image
    Error response from daemon
    connection timeout
    no route to host

Troubleshoot:

    podman pull quay.io/ceph/ceph:v18
    docker pull quay.io/ceph/ceph:v18
    ping quay.io
    curl -I https://quay.io

Resolution:

    1. Check DNS.
    2. Check network proxy.
    3. Check firewall.
    4. Use Harbor or Alibaba Cloud to transit Ceph images.
    5. Use --image to specify internal image during bootstrap.

---

### 29.2 Ceph Command Not Available

Symptoms:

    ceph: command not found

Cause:

    ceph-common not installed.

Resolution:

Ubuntu:

    apt install -y ceph-common

Rocky:

    dnf install -y ceph-common

Or use:

    cephadm shell

---

### 29.3 Node Addition Failure

Symptoms:

    ceph orch host add ceph-node02 10.0.0.32 failed

Troubleshoot:

    ping ceph-node02
    ssh -i /etc/ceph/cephadm-ssh-key root@ceph-node02 hostname
    ceph orch host ls
    ceph health detail

Common causes:

- Hosts configuration error
- SSH key not distributed
- Root login restricted
- Target node missing python3
- Firewall blocking port 22
- Time synchronization issues

---

### 29.4 MON Not Expanded to 3

Troubleshoot:

    ceph orch ps --daemon_type mon
    ceph mon stat
    ceph quorum_status --format json-pretty
    ceph orch host ls

Reapply:

    ceph orch apply mon --placement="ceph-node01 ceph-node02 ceph-node03"

Monitor:

    watch -n 3 'ceph -s'

---

### 29.5 OSD Disk Unavailable

Symptom: /think
```

# 29.1 Available is No in ceph orch device ls --wide

Troubleshooting:

    ceph orch device ls --wide
    lsblk
    df -hT
    wipefs -n /dev/sdb
    pvs
    vgs
    lvs

Resolution:

After confirming no business data exists:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

Then recheck:

    ceph orch device ls --wide

---

### 29.6 OSD Addition Failed

Troubleshooting:

    ceph orch daemon add osd ceph-node01:/dev/sdb
    ceph orch ps --daemon_type osd
    ceph health detail
    ceph orch device ls --wide

Check on the target node:

    lsblk
    journalctl -u ceph-*.target
    dmesg | grep -i error

Common causes:

- Dirty disk
- Incorrect device name
- Node not joined to cluster
- Container runtime issues
- cephadm cannot SSH to node

---

### 29.7 PG Long Time Not Clean

Troubleshooting:

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree
    ceph osd df

Common causes:

- Insufficient OSD count
- OSD down
- Replica count exceeds available failure domain
- OSD in recovery
- Slow disk or network

---

### 29.8 Dashboard Access Failure

Troubleshooting:

    ceph mgr services
    ceph mgr stat
    ceph mgr module ls
    ss -lntp | grep 8443

Check firewall.

Rocky:

    firewall-cmd --list-ports

Ubuntu:

    ufw status

---

## 30. Cleanup and Rollback Notes

### 30.1 When to Clean Up

The following scenarios may require cleaning up the experimental environment:

- Bootstrap failed and ready for reinstallation
- Node join error
- Disk mistakenly added but no business data yet
- Experimental environment needs reinitialization
- Before virtual machine snapshot rollback, need to understand residual data

---

### 30.2 Recommended Cleanup Method

For experimental environments, the safest cleanup method is:

    Delete virtual machine
    Recreate virtual machine
    Remount empty data disk

This is the cleanest and least residual method.

---

### 30.3 cephadm Cleanup Commands

If it's just an experimental environment and data doesn't need to be preserved, use:

    cephadm rm-cluster --force --zap-osds --fsid <fsid>

Check fsid:

    ceph fsid

High-risk warning:

    This command will delete the Ceph cluster.
    --zap-osds will clean up OSDs.
    Prohibited in production environments.
    Must confirm all data can be discarded before execution.

---

### 30.4 Manual Disk Cleanup

If cleaning up OSD data disks:

    wipefs -a /dev/sdb
    sgdisk --zap-all /dev/sdb

If there is /dev/sdc:

    wipefs -a /dev/sdc
    sgdisk --zap-all /dev/sdc

High-risk warning:

    Only allowed to execute on confirmed OSD disks with no business data in experimental environments.
    Do not execute on system disks.

---

## 31. Production Environment Notes

1. Capacity, network, failure domain, and recovery strategy planning must be done before production deployment.
2. Do not mix Ceph OSD with business nodes or Kubernetes nodes in production.
3. MON is recommended to have 3 or 5, not 1 or 2.
4. MGR must have at least 2, ensuring active/standby.
5. OSD must use dedicated data disks, not mixed with system disks.
6. Do not use --all-available-devices to blindly take over all disks.
7. Production environments must clearly define Ceph version, not mixing different main versions across nodes.
8. Plan Ceph container images and software sources in advance for domestic network environments.
9. Do not expose Dashboard directly to the public internet.
10. Do not arbitrarily delete Pools in production.
11. Do not arbitrarily execute rm-cluster, zap-osds, wipefs, sgdisk in production.
12. All OSD operations must first check ceph -s and ceph health detail.
13. All high-risk changes must have backups, windows, approvals, and rollback plans.
14. Ceph deployment must be integrated with monitoring and alerts after completion.
15. Fault and recovery drills must be completed before business deployment.

---

## 32. Interview Answer Approach

If asked in an interview:

    How do you deploy a Ceph cluster using cephadm?

You can answer:

    I would first plan nodes, disks, network, and failure domains. At least prepare 3 nodes, deploying MON, MGR, and OSD on separate nodes, using dedicated data disks for OSDs, not mixed with system disks.
    At the system level, I would configure hostname, hosts, time synchronization, SSH keyless login, basic dependencies, and domestic software sources. For Ubuntu environments, use Alibaba Cloud Ubuntu source and Ceph apt source; for Rocky Linux 9, use Alibaba Cloud Rocky source and Ceph dnf source, while paying attention to firewalld, SELinux, and Podman.
    During deployment, execute cephadm bootstrap on the first node, specifying --mon-ip as the first node's IP. After bootstrap success, it will generate /etc/ceph/ceph.conf, admin keyring, cephadm SSH public key, and Dashboard address.
    Then add other nodes via ceph orch host add, deploy 3 MONs via ceph orch apply mon, deploy multiple MGRs via ceph orch apply mgr, and manually add OSDs via ceph orch daemon add osd node:/dev/sdX.
    After deployment, validate cluster status using ceph -s, ceph orch ps, ceph orch host ls, ceph osd tree, ceph pg stat, and ceph health detail, confirming MON quorum is normal, MGR has active/standby, OSD up/in, and PG active+clean.
    In production, also focus on Ceph image sources, software sources, Dashboard exposure, disk misoperation, firewall, SELinux, capacity water level, CRUSH failure domains, and recovery drills.

---

## 33. Summary of This Section

This document completes the Ceph cluster basic deployment experiment:

1. Clarified the 3-node Ceph experiment topology on 10.0.0.31-33.
2. Clarified Ubuntu 22.04.5 LTS as the main experimental system.
3. Supplemented Rocky Linux 9 installation source configuration methods.
4. Used Alibaba Cloud Ubuntu, Rocky, and Ceph image sources to resolve domestic network environment issues.
5. Completed cephadm and ceph-common installation.
6. Executed cephadm bootstrap on ceph-node01.
7. Verified /etc/ceph/ceph.conf, admin keyring, and Dashboard.
8. Distributed cephadm SSH public key.
9. Added ceph-node02 and ceph-node03.
10. Deployed 3 MONs.
11. Deployed multiple MGRs.
12. Manually added OSD data disks.
13. Verified using ceph -s, ceph osd tree, and ceph pg stat.
14. Verified basic object read/write through testing Pool and rados put/get.
15. Summarized troubleshooting paths for bootstrap, node addition, OSD addition, and Dashboard access failures.
16. Clarified experimental environment cleanup and production environment high-risk operation boundaries.

---

## Thirty-Four, Reference Documents

Cephadm official installation documentation:

    https://docs.ceph.com/en/latest/cephadm/install/

Ceph Reef cephadm installation documentation:

    https://docs.ceph.com/en/reef/cephadm/install/

Ceph software package acquisition documentation:

    https://docs.ceph.com/en/reef/install/get-packages/

Ceph OSD service documentation:

    https://docs.ceph.com/en/reef/cephadm/services/osd/

Ceph Orchestrator documentation:

    https://docs.ceph.com/en/reef/mgr/orchestrator/

Ceph health check documentation:

    https://docs.ceph.com/en/reef/rados/operations/health-checks/

Ceph Dashboard documentation:

    https://docs.ceph.com/en/reef/mgr/dashboard/

Alibaba Cloud Ubuntu image:

    https://developer.aliyun.com/mirror/ubuntu

Alibaba Cloud Rocky Linux image:

    https://developer.aliyun.com/mirror/rockylinux

Alibaba Cloud Ceph image:

    https://developer.aliyun.com/mirror/ceph