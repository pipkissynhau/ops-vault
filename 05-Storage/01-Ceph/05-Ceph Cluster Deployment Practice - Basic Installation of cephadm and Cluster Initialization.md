# Ceph Cluster Deployment Practice: Basic Installation of cephadm and Cluster Initialization

Recommended Path: 05-Storage/01-Ceph/05-Ceph Cluster Deployment Practice: Basic Installation of cephadm and Cluster Initialization.md

Tags: #Ceph #cephadm #Distributed Storage #Cluster Deployment #Ubuntu #RockyLinux #OSD #MON #MGR #SRE #Domestic Repository #Advanced SRE

---

## I. Document Description

This article is the fifth in the Ceph Advanced SRE storage module, focusing on completing a reproducible experiment for initializing a multi-node Ceph cluster.

This document does not merely record installation commands but is organized according to the practical standards of advanced SRE:

- Clearly define the experimental objectives
- Specify node planning
- Define disk planning
- Identify the operating system
- Determine the domestic software repository
- Outline pre-deployment checks
- Detail deployment steps
- Explain expected outcomes
- Provide verification commands
- Offer guidance for troubleshooting common issues
- Define boundaries for cleanup and rollback
- Highlight precautions for production environments

The primary experimental system in this article is:

    Ubuntu Server 22.04.5 LTS

Additional production reference systems include:

    Rocky Linux 9

Ceph deployment method used:

    cephadm

Main Ceph experiment version:

    Ceph Reef

Domestic repository strategy:

    Ubuntu base repository: Alibaba Cloud Ubuntu Repository
    Rocky Linux base repository: Alibaba Cloud Rocky Linux Repository
    Ceph software repository: Alibaba Cloud Ceph Repository

---

## II. Experimental Objectives

Upon completing this article, you should achieve the following objectives:

1. Initialize a basic Ceph cluster on 3 VMs/bare metal nodes.
2. Use cephadm bootstrap to initialize the first node.
3. Add ceph-node02 and ceph-node03 to the Ceph cluster.
4. Deploy 3 MONs to form a basic quorum.
5. Deploy multiple MGRs to establish an active/standby configuration.
6. Assign independent data disks to each node as OSDs.
7. Use ceph -s to verify the cluster's health status.
8. Utilize ceph orch commands to view hosts, services, devices, and OSDs.
9. Access the Ceph Dashboard.
10. Create a test Pool and write objects using rados for basic verification.
11. Understand the troubleshooting process for bootstrap failures, node additions, OSD additions, and Dashboard access issues.
12. Identify high-risk commands that should not be executed in production environments.

---

## III. Experimental Topology

### 3.1 Node Planning

Experimental IP range:

    10.0.0.0/24

Existing environment:

| IP | Host Name | Purpose |
|---|---|---|
| 10.0.0.10 | ops-server | GitLab / Jenkins / Harbor / Image relay |
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Ceph experimental nodes:

| IP | Host Name | Role | Notes |
|---|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD | First node, performs bootstrap |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD | Second storage node |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD | Third storage node |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault testing | Optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW testing | Optional |

The minimum experimental setup includes:

    ceph-node01
    ceph-node02
    ceph-node03

---

### 3.2 Disk Planning

It is recommended that each Ceph node have at least:

| Disk | Purpose |
|---|---|
| /dev/sda | System disk |
| /dev/sdb | Ceph OSD data disk |
| /dev/sdc | Ceph OSD data disk (optional) |

By default, this document assumes that each node has at least one OSD data disk:

    /dev/sdb

If each node has two OSD data disks, both can be used:

    /dev/sdb
    /dev/sdc

Important notes:

- OSD data disks will be managed by Ceph.
- Ensure no business data is on the disks before adding them as OSDs.
- Do not use the system disk /dev/sda for Ceph OSD```markdown
sdb      8:16   0  100G  0 disk

Confirm that /dev/sdb is not mounted:

    df -hT

View the disk signature:

    wipefs -n /dev/sdb

Check if there is LVM:

    pvs
    vgs
    lvs

Requirements:

    /dev/sdb should not contain any business data.
    /dev/sdb must not be mounted.
    /dev/sdb should not be the system disk.
    /dev/sdb should be available for Ceph to take over.

---

## Step 6: Configure hostnames for all nodes

Execute the following commands on ceph-node01:

    hostnamectl set-hostname ceph-node01

Repeat this on ceph-node02 and ceph-node03:

    hostnamectl set-hostname ceph-node02
    hostnamectl set-hostname ceph-node03

Optionally, perform the same on ceph-node04:

    hostnamectl set-hostname ceph-node04

Verify the hostnames:

    hostname

Expected output:

    ceph-node01
    ceph-node02
    ceph-node03
    ceph-node04

---

## Step 7: Configure the hosts file for all nodes

All Ceph nodes should execute the following command:

    cat >> /etc/hosts <<'EOF'
    10.0.0.31 ceph-node01
    10.0.0.32 ceph-node02
    10.0.0.33 ceph-node03
    10.0.0.34 ceph-node04
    10.0.0.35 ceph-client
    EOF

Verify the hosts file configuration:

    getent hosts ceph-node01
    getent hosts ceph-node02
    getent hosts ceph-node03

Expected output:

    10.0.0.31      ceph-node01
    10.0.0.32      ceph-node02
    10.0.0.33      ceph-node03

Test the connection using ping:

    ping -c 2 ceph-node01
    ping -c 2 ceph-node02
    ping -c 2 ceph-node03
```

---

## Step 8: Synchronize time for all nodes

### 8.1 Ubuntu 22.04

All Ubuntu nodes should perform the following steps:

    apt update
    apt install -y chrony

    systemctl enable --now chrony

Verify the status of chrony:

    systemctl status chrony

Check the synchronization sources:

    chronyc sources -v

Set the time zone to Asia/Shanghai:

    timedatectl set-timezone Asia/Shanghai

Expected output:

    Thechrony service is active.
    The current time zone is set to Asia/Shanghai.
    The system clock is synchronized.

---

### 8.2 Rocky Linux 9

All Rocky 9 nodes should perform the following steps:

    dnf install -y chrony

    systemctl enable --now chronyd

Verify the status of chronyd:

    systemctl status chronyd

Check the synchronization sources:

    chronyc sources -v

Set the time zone to Asia/Shanghai:

    timedatectl set-timezone Asia/Shanghai

Verify the time zone setting:

    timedatectl

---

### 8.3 Why Time Synchronization is Important for Ceph

Ceph is highly sensitive to time synchronization issues.

Time discrepancies can lead to various problems, such as:

- Issues with MON quorum formation.
- Authentication failures.
- Confusions in log timestamps.
- Errors in dashboard certificate verification.
- Increased difficulty in troubleshooting.

It is essential to ensure proper time synchronization before deploying Ceph.

---

## Step 9: Configure the Aliyun Ubuntu repository for Ubuntu 22.04

This step should be performed on a Ubuntu Server 22.04.5 LTS node.

Back up the original sources list:

    cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F-%H%M%S)

Create and write the Aliyun Ubuntu repository contents:

    cat > /etc/apt/sources.list <<'EOF'
    deb https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
    deb https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
    deb https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
    deb https://mirrors.aliyun.comubuntu/ jammy-backports main restricted universe multiverse
    EOF

Update the package cache:

    apt clean
    apt update

Install basic dependenciespython3 \
podman \
openssh-clients

Verification:

    podman --version
    python3 --version

---

## Section Twelve: Rocky Linux 9: Configuring Alibaba Cloud Ceph Repository

### 12.1 Adding the Ceph Reef Repo

Create a Ceph repo:

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

Generate the cache:

    dnf makecache

Check installation:

    dnf list cephadm
    dnf list ceph-common

Install:

    dnf install -y cephadm ceph-common

Verification:

    cephadm version
    ceph --version

---

## Section Thirteen: Rocky Linux 9: Firewalld and SELinux

### 13.1 Firewalld

Check status:

    systemctl status firewalld

In a test environment, it can be temporarily disabled:

    systemctl disable --now firewalld

If not disabled, ports need to be opened:

    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=3300/tcp
    firewall-cmd --permanent --add-port=6789/tcp
    firewall-cmd --permanent --add-port=6800-7300/tcp
    firewall-cmd --permanent --add-port=8443/tcp
    firewall-cmd --permanent --add-port=9283/tcp
    firewall-cmd --reload

Check ports:

    firewall-cmd --list-ports

---

### 13.2 SELinux

Check status:

    getenforce

In a test environment, it can be temporarily set to permissive:

    setenforce 0

To permanently set it to permissive:

    sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

Verify configuration:

    grep '^SELINUX=' /etc/selinux/config

Note for production:

    It is not recommended to simply disable SELinux in a production environment.
    If enterprise security policies require SELinux, it should be configured according to Ceph's official recommendations and corporate guidelines.

---

## Section Fourteen: Container Images and Domestic Network Optimization

### 14.1 cephadm Container Images

cephadm operates Ceph components within containers.

Common official images:

    quay.io/ceph/ceph

In domestic networks, directly pulling from quay.io may fail or be slow.

---

### 14.2 Image Handling Recommendations

Recommended approach:

    1. If direct access to quay.io is possible, use the default method.
    2. If access is slow or fails, pull the official image in a connected network environment.
    3. Re-tag the image to a local repository such as Harbor or Alibaba Cloud.
    4. Use the --image option during bootstrap to specify the internal image location.
    5. Avoid arbitrarily modifying the containerd configuration of existing Kubernetes nodes for Ceph purposes.

Note:

    The Ceph test nodes are independent VMs or bare metal instances.
    Do not affect the underlying runtime of the existing Kubernetes nodes 10.0.0.20/21/22.

---

### 14.3 Example of Pre-pulling Images

If using Podman:

    podman pull quay.io/ceph/ceph:v18

If using Docker:

    docker pull quay.io/ceph/ceph:v18

To check installed images:

    podman images | grep ceph

Or:

    docker images | grep ceph

If the images have been stored locally, use the following format:

    registry.cn-hangzhou.aliyuncs.com/<namespace>/ceph:v18

When bootstrapping later, specify:

    --image registry.cn-hangzhou.aliyuncs.com/<namespace>/ceph:v18

---

## Section Fifteen: Pre-checks Before Bootstrapping the First Node

The following steps are performed on ceph-node01.

### 15.1 Confirming Basic Environment

    hostname
    ip addr
    cat /etc/os-release
    timedatectl
    cephadmsudo /usr/sbin/cephadm shell --fsid <fsid> -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

This will also create the following files:

    /etc/ceph/ceph.conf
    /etc/ceph/ceph.client.admin.keyring
    /etc/ceph/ceph.pub
    /etc/ceph/cephadm-ssh-key

To verify, check:

    ls -l /etc/ceph/

You should see at least the following files:

    ceph.conf
    ceph.client.admin.keyring
    ceph.pub

---

## Section Seventeen: Basic Verification After Bootstrap

### 17.1 Checking Cluster Status

    ceph -s

If you cannot execute this command directly on the host, enter the shell first:

    cephadm shell

Then run:

    ceph -s

During the initial bootstrap phase, you may see:

    health: HEALTHWARN

Common reasons for this include:

- Not enough OSDs
- Only one MON node
- No additional nodes or OSDs added yet

This is a normal initial state and does not indicate a deployment failure.

---

### 17.2 Expected Initial State

After bootstrap, before adding any additional nodes or OSDs, you should see something like this:

    cluster:
      health: HEALTH_WARN

    services:
      mon: 1 daemon, quorum ceph-node01
      mgr: ceph-node01(active)
      osd: 0 osds: 0 up, 0 in

This is a reasonable initial state. The status will change once additional nodes and OSDs are added.

---

### 17.3 Checking the MGR Service

    ceph mgr services

The output might look like this:

    {
        "dashboard": "https://ceph-node01:8443/",
        "prometheus": "http://ceph-node01:9283/"
    }

---

## Section Eighteen: Configuring Cephadm SSH Management Access

### 18.1 Viewing the cephadm Public Key

Execute this command on ceph-node01:

    cat /etc/ceph/ceph.pub

---

### 18.2 Distributing the Public Key to Other Nodes

    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node02
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node03

Optional:

    ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node04

---

### 18.3 Verifying Cephadm SSH Connection

    ssh -i /etc/ceph/cephADM-ssh-key root@ceph-node02 hostname
    ssh -i /etc/ceph/cephADM-ssh-key root@ceph-node03 hostname

Expected output:

    ceph-node02
    ceph-node03

If the connection fails, check the following:

- Whether root has SSH access permissions.
- Whether sshd is running.
- Whether the /etc/hosts file contains the correct entries.
- Whether the keys were successfully distributed.
- Whether the firewall allows port 22.
- Whether python3 is installed on the target nodes.

---

## Section Nineteen: Adding Ceph Hosts

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

### 19.3 Viewing the Host List

    ceph orch host ls

Expected output:

    HOST          ADDR        LABELS  STATUS
    ceph-node01   10.0.0.31
    ceph-node02   10.0.0.32
    ceph-node03   10.0.0.33

---

### 19.4 Adding Labels to Nodes

    ceph orch host label add ceph-node01 mon
    ceph orch host label add ceph-node02 mon
    ceph orch host label add ceph-node03 mon

    ceph orch host label add ceph-node01 mgr
    ceph orch host label add ceph-node02 mgr
    ceph orch host label add ceph-node03 mgr

    ceph orch host label add ceph-node01 osd
    ceph orchOn ceph-node01, execute:

    ceph orch device ls --wide

Expected output:

    /dev/sdb should be listed as AVAILABLE

---

## Section 24: Manually Adding OSDs

### 24.1 Why Add OSDs Manually

In this experiment, it is recommended to add OSDs manually:

    ceph orch daemon add osd ceph-node01:/dev/sdb

It is not recommended to use the following command at the beginning:

    ceph orch apply osd --all-available-devices

Reasons:

    Manual addition is safer.
    It allows you to clearly identify which disk each OSD is associated with.
    This prevents accidentally adding disks that should not be part of Ceph.

---

### 24.2 Adding the ceph-node01 OSD

    ceph orch daemon add osd ceph-node01:/dev/sdb

Expected output:

    Created osd(s) ...

---

### 24.3 Adding the ceph-node02 OSD

    ceph orch daemon add osd ceph-node02:/dev/sdb

---

### 24.4 Adding the ceph-node03 OSD

    ceph orch daemon add osd ceph-node03:/dev/sdb

---

### 24.5 If Each Node Has /dev/sdc

Continue adding:

    ceph orch daemon add osd ceph-node01:/dev/sdc
    ceph orch daemon add osd ceph-node02:/dev/sdc
    ceph orch daemon add osd ceph-node03:/dev/sdc

---

## Section 25: Verifying OSD Status

### 25.1 Checking OSD Services

    ceph orch ps --daemon_type osd

Expected output:

    osd.0
    osd.1
    osd.2

If each node has two disks, you may see:

    osd.0
    osd.1
    osd.2
    osd.3
    osd.4
    osd.5

---

### 25.2 Checking the OSD Tree

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

Key points to confirm:

    OSDs should be listed as up and in.
    OSDs should be distributed across different nodes.
    No OSDs should be marked as down.

---

### 25.3 Checking the Cluster Status

    ceph -s

Expected output gradually approaching:

    cluster:
      health: HEALTH_OK

    services:
      mon: 3 daemons, quorum ceph-node01,ceph-node02,ceph-node03
      mgr: ceph-node01(active), standbys: ceph-node02, ceph-node03
      osd: 3 osds: 3 up, 3 in

    data:
      pgs: active+clean

If the status is not HEALTH_OK immediately after adding OSDs, wait and monitor:

    watch -n 3 'ceph -s'

---

## Section 26: Verifying Dashboard Access

### 26.1 Checking the Dashboard Address

    ceph mgr services

Expected output similar to:

    {
        "dashboard": "https://ceph-node01:8443/",
        "prometheus": "http://ceph-node01:9283/"
    }

Access the dashboard at:

    https://10.0.0.31:8443/

Username:

    admin

Password:

    Ceph_Admin_ChangeMe_123

---

### 26.2 If the Dashboard Cannot Be Accessed

Check the MGR status:

    ceph mgr stat

Check the available modules:

    ceph mgr module ls

Enable the dashboard if it is not available:

### Ceph MGR Services

Requirements:

- There should be a dashboard address.
- It must be accessible via a browser.
- Administrators should be able to log in.

---

## Chapter 29: Common Issues and Troubleshooting

### 29.1 Failed Bootstrap Image Pull

Symptoms:

- Failed to pull the container image.
- Error response from the daemon.
- Connection timeout.
- No route to the host.

Troubleshooting:

- `podman pull quay.io/ceph/ceph:v18`
- `docker pull quay.io/ceph/ceph:v18`
- `ping quay.io`
- `curl -I https://quay.io`

Solution:

1. Check DNS settings.
2. Verify the network proxy configuration.
3. Review firewall rules.
4. Use Harbor or Alibaba Cloud to relay Ceph images.
5. Specify an internal image using `--image` during bootstrap.

---

### 29.2 Unavailable Ceph Commands

Symptom:

- `ceph: command not found`

Cause:

- `ceph-common` is not installed.

Solution:

For Ubuntu:

```bash
apt install -y ceph-common
```

For Rocky:

```bash
dnf install -y ceph-common
```

Alternatively, use:

```bash
cephadm shell
```

---

### 29.3 Failed to Add Node

Symptom:

- `ceph orch host add ceph-node02 10.0.0.32` fails.

Troubleshooting:

- `ping ceph-node02`
- `ssh -i /etc/ceph/cephadm-ssh-key root@ceph-node02 hostname`
- `ceph orch host ls`
- `ceph health detail`

Common causes:

- Incorrect host configuration.
- SSH keys not distributed.
- Root login restricted.
- Python3 missing on the target node.
- Firewall blocking port 22.
- Time synchronization issues.

---

### 29.4 MON Not Expanded to 3 Nodes

Troubleshooting:

- `ceph orch ps --daemon_type mon`
- `ceph mon stat`
- `ceph quorum_status --format json-pretty`
- `ceph orch host ls`

Reapply configuration:

```bash
ceph orch apply mon --placement="ceph-node01 ceph-node02 ceph-node03"
```

Monitor progress:

```bash
watch -n 3 'ceph -s'
```

---

### 29.5 OSD Disk Unavailable

Symptom:

- `ceph orch device ls --wide` shows "AVAILABLE" as "No".

Troubleshooting:

- `ceph orch device ls --wide`
- `lsblk`
- `df -hT`
- `wipefs -n /dev/sdb`
- `pvs`
- `vgs`
- `lvs`

Solution:

After confirming no data is on the disk:

- `wipefs -a /dev/sdb`
- `sgdisk --zap-all /dev/sdb`

Recheck status:

```bash
ceph orch device ls --wide`
```

---

### 29.6 Failed to Add OSD

Troubleshooting:

- `ceph orch daemon add osd ceph-node01:/dev/sdb`
- `ceph orch ps --daemon_type osd`
- `ceph health detail`
- `ceph orch device ls --wide`

On the target node:

- `lsblk`
- `journalctl -u ceph-*.target`
- `dmesg | grep -i error`

Common causes:

- Disk is not clean.
- Incorrect device name.
- Node not added to the cluster.
- Container runtime issues.
- Unable to SSH into the node with `cephadm`.

---

### 29.7 PG Not Cleaning Up

Troubleshooting:

- `ceph -s`
- `ceph health detail`
- `ceph pg stat`
- `ceph osd tree`
- `ceph osd df`

Common causes:

- Insufficient number of OSDs.
- Some OSDs are down.
- Number of replicas exceeds available fault domains.
- OSDs are in recovery mode.
- Slow disk or network performance.

---

### 29.8 Failed to Access Dashboard

Troubleshooting:

- `ceph mgr services`
- `ceph mgr stat`
- `ceph mgr module ls`
- `ss -lntp | grep 8443`

Check firewall settings:

For Rocky:

```bash
firewall-cmd --list-ports
```

For Ubuntu:

```bash
ufw status
```

---

## Chapter 30: Cleaning Up and Rollback Instructions

### 30.1 When Cleaning Up is Needed

The following scenarios may require cleaning up the experimental environment:

- Bootstrap failed and needs to be reinstalled.
- Nodes were added incorrectly.
- Disks were mistakenly added but no data was stored yet.
- The experimental environment needs to be reset.
- Before rolling back a6. Execute cephadm bootstrap on ceph-node01.
7. Verify /etc/ceph/ceph.conf, the admin keyring, and the Dashboard.
8. Distribute the cephadm SSH public key.
9. Add ceph-node02 and ceph-node03.
10. Deploy 3 MONs.
11. Deploy multiple MGRs.
12. Manually add OSD data disks.
13. Complete verification using ceph -s, ceph osd tree, and ceph pg stat.
14. Verify basic object read and write operations by testing Pool and rados put/get functions.
15. Identify troubleshooting steps for issues with bootstrap, node addition, OSD addition, and Dashboard access.
16. Clearly define the boundaries between cleaning up the experimental environment and performing high-risk operations in the production environment.

---

## Section Thirty-Four: Reference Documents

Official Cephadm Installation Documentation:

    https://docs.ceph.com/en/latest/cephadm/install/

Ceph Reef Cephadm Installation Documentation:

    https://docs.ceph.com/en/reef/cephadm/install/

Ceph Software Package Acquisition Documentation:

    https://docs.ceph.com/en/reef/install/get-packages/

Ceph OSD Service Documentation:

    https://docs.ceph.com/en/reef/cephadm/services/osd/

Ceph Orchestrator Documentation:

    https://docs.ceph.com/en/reef/mgr/orchestrator/

Ceph Health Check Documentation:

    https://docs.ceph.com/en/reef/rados/operations/health-checks/

Ceph Dashboard Documentation:

    https://docs.ceph.com/en/reef/mgr/dashboard/

Aliyun Ubuntu Images:

    https://developer.aliyun.com/mirror/ubuntu

Aliyun Rocky Linux Images:

    https://developer.aliyun.com/mirror/rockylinux

Aliyun Ceph Images:

    https://developer.aliyun.com/mirror/ceph