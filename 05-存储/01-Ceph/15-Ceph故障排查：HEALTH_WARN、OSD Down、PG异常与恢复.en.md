# Ceph Troubleshooting: Common Issues with OSD, PG, MON, MDS, RGW, and CSI

Recommended Path: 05-Storage/01-Ceph/15-Ceph Troubleshooting: Common Issues with OSD, PG, MON, MDS, RGW, and CSI.md

Tags: #Ceph #Troubleshooting #OSD #PG #MON #MGR #MDS #RGW #RBD #CephFS #CSI #SlowOps #AdvancedSRE

---

## I. Document Overview

This article is the fifteenth in the Ceph Advanced SRE Storage module, focusing on common troubleshooting methods for Ceph.

Previous topics include:

- Ceph Cluster Deployment
- OSD Management
- Pools and PGs
- CRUSH Fault Domains
- RBD Block Storage
- CephFS File Storage
- RGW Object Storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph Routine Maintenance

This article enters the troubleshooting phase.

When troubleshooting Ceph, it's essential to establish a comprehensive diagnostic process:

    First, assess overall health.
      |
      v
    Then, examine detailed health status.
      |
      v
    Next, identify the faulty component.
      |
      v
    Determine the scope of the issue.
      |
      v
    Decide whether manual intervention is needed.
      |
      v
    Finally, verify the recovery results.

This article covers:

- General troubleshooting methods
- Interpretation of HEALTH_WARN and HEALTH_ERR messages
- Troubleshooting OSD down issues
- Troubleshooting OSD full/nearfull issues
- Troubleshooting PG degraded/undersized/stale/inactive/inconsistent issues
- MON quorum anomaly troubleshooting
- MGR anomaly troubleshooting
- MDS/CephFS anomaly troubleshooting
- RGW/S3 access anomaly troubleshooting
- RBD mounting and locking issue troubleshooting
- Kubernetes CSI PVC Pending/FailedMount troubleshooting
- Slow Ops and performance issues troubleshooting
- Network, disk, and time synchronization problems troubleshooting
- Boundaries for high-risk troubleshooting commands
- Troubleshooting review templates

---

## II. Troubleshooting Objectives

After completing this article, you should be able to:

1. Quickly assess the overall cluster status using `ceph -s`.
2. Identify the type of alarm based on `ceph health detail`.
3. Distinguish between HEALTH_WARN and HEALTH_ERR messages and know their handling priorities.
4. Troubleshoot issues such as OSD down, OSD out, OSD full, and OSD latency.
5. Resolve problems with PG degraded, undersized, stale, inactive, incomplete, and inconsistent states.
6. Address MON quorum anomalies and clock skew issues.
7. Investigate MGR active absence, Dashboard inaccessibility, and Prometheus data collection failures.
8. Troubleshoot CephFS MDS anomalies, mounting failures, and slow access issues.
9. Resolve RGW access failures, S3 authentication errors, Bucket abnormalities, and proxy layer issues.
10. Deal with RBD map failures, unmap failures, lock remnants, and snapshot deletion failures.
11. Address common Kubernetes RBD/CephFS CSI issues.
12. Recognize which commands are for read-only diagnosis and which are high-risk repair commands.
13. Establish a standard process for troubleshooting response, handling, verification, and review.

---

## III. Experimental and Production Environments

### 3.1 Ceph Cluster Nodes

This article uses the experimental Ceph module environment.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON/MGR/OSD/RGW/MDS |
| 10.0.0.32 | ceph-node02 | MON/MGR/OSD/RGW/MDS |
| 10.0.0.33 | ceph-node03 | MON/MGR/OSD/RGW |
| 10.0.0.34 | ceph-node04 | OSD/Expansion/Fault Testing (optional) |
| 10.0.0.35 | ceph-client | RBD/CephFS/RGW Client Testing (optional) |

Primary experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Kubernetes Cluster

If CSI troubleshooting is involved, a Kubernetes cluster will be required:

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Operating environment:

    kubeadm
    ceph orch ps --daemon_type mgr

---

### 6.3 OSD / PG Commands

    ceph osd tree
    ceph osd df
    ceph osd perf
    ceph osd metadata osd.X
    ceph pg stat
    ceph pg dump_stuck
    ceph pg map <pg-id>
    ceph pg <pg-id> query

---

### 6.4 Pool Commands

    ceph osd pool ls
    ceph osd pool ls detail
    ceph osd pool get <pool-name> all
    ceph osd pool autoscale-status
    ceph df

---

### 6.5 Node-Side Commands

    hostname
    ip addr
    ip route
    ping -c 3 <target-ip>
    ss -lntp
    lsblk
    df -hT
    free -h
    top
    vmstat 1
    iostat -x 1
    dmesg | tail -100
    journalctl -k --no-pager | tail -100
    timedatectl
    chronyc sources -v

---

### 6.6 cephadm Log Commands

To view the service:

    ceph orch ps

To view logs for a specific daemon:

    cephadm logs --name osd.0
    cephadm logs --name mon.ceph-node01
    cephadm logs --name mgr.ceph-node01

The actual daemon names should be based on the output from `ceph orch ps`.

---

## Section 7: Troubleshooting OSD Down Issues

### 7.1 Symptoms

Running `ceph -s` may display:

    health: HEALTHWARN
    1 osds down
    Degraded data redundancy

By checking:

    ceph osd stat

You might see:

    6 osds: 5 up, 6 in

This indicates:

- There are a total of 6 OSDs.
- Only 5 OSDs are online.
- However, all 6 OSDs are still listed as "in" status.

---

### 7.2 Identifying the Down OSD

Check:

    ceph osd tree

For example:

    osd.3 down in

This means:

- The osd.3 process is unavailable.
- But it is still included in the CRUSH data distribution.

This is a critical condition that requires attention.

---

### 7.3 Viewing Detailed Health Information

Use:

    ceph health detail

Focus on:

- Which OSD is down.
- Which PGs are degraded.
- Whether recovery is ongoing.
- If there are any slow requests.
- Any disk or network-related issues.

---

### 7.4 Checking OSD Metadata

Use:

    ceph osd metadata osd.3

Verify key details:

- hostname
- bluestore_bdev_dev_node
- devices
- ceph_version
- osd_objectstore

This helps determine:

- On which node osd.3 is located.
- Which disk it corresponds to.

---

### 7.5 Checking the cephadm Service

Use:

    ceph orch ps --daemon_type osd

Filter results:

    ceph orch ps --daemon_type osd | grep osd.3

Confirm:

- Whether it is running.
- If there are any errors or stoppages.
- Recent restart attempts.

---

### 7.6 Logging In to the Faulty Node for Troubleshooting

Assume osd.3 is on ceph-node02.

Log in:

    ssh root@ceph-node02

Check the hostname:

    hostname

View disks:

    lsblk

Check system and data disk capacities:

    df -hT

Review kernel logs:

    dmesg | grep -i error
    dmesg | grep -i fail
    journalctl -k | grep -i error

Monitor system resources:

    top
    free -h
    iostat -x 1

Check services:

    systemctl status ceph-*.target

---

### 7.7 Common Causes

| Cause | Description |
|---|---|
| Node Downtime | Multiple OSDs go down on the host. |
| Disk Failure | I/O errors are reported in dmesg. |
| Container Runtime Issues | Abnormalities with the cephadm daemon. |
| Network Problems | OSD heartbeat failures occur. |
| System Disk Fullness | Daemons cannot write logs or metadata. |
| OSD Data Corruption | The OSD fails to start. |
| Time-related Issues | Communication and authentication errors occur. |

---

### 7.8 Handling Strategies

For temporary service issues, try restarting the OSD:

    ceph orch daemon restart osd.3

Monitor:

    ceph osd tree
    ceph -s

If it's a nodeCheck the size.
Check the number of hosts/racks.

---

## Section X: Fault 4: PG Under-sized Troubleshooting

### 10.1 Symptoms

The `ceph -s` command displays:

PGs are under-sized.

---

### 10.2 Meaning

"PGs under-sized" indicates that:

The current number of PG replicas is less than the pool size.

For example:

Pool size = 3
But some PGs only have 2 replicas.

---

### 10.3 Common Causes

| Cause | Explanation |
|---|---|
| Insufficient number of OSDs | Size set to 3, but fewer available OSDs |
| Insufficient number of hosts | Not enough available hosts within the failure domain |
| Insufficient number of racks | Not enough racks within the failure domain |
| OSD down/out | Not enough available replica locations |
| Improper CRUSH Rule settings | Excessively high failure domain level |
| OSD mistakenly removed from CRUSH | Data cannot be properly placed |

---

### 10.4 Troubleshooting Commands

`ceph health detail`
`ceph pg stat`
`ceph osd tree`
`ceph osd crush tree`
`ceph osd pool get <pool-name> size`
`ceph osd pool get <pool-name> crush_rule`
`ceph osd crush rule dump <rule-name>`

---

### 10.5 Handling Approach

Handle according to the cause:

- OSD down: Restore the OSD.
- Insufficient number of OSDs: Add new OSDs.
- Insufficient failure domains: Increase hosts/racks or adjust CRUSH Rules.
- Excessively high size setting: Reduce the size in a test environment; proceed with caution in production.
- CRUSH errors: Fix the CRUSH topology.

Production recommendations:

- Prioritize restoring resources and failure domains.
- Do not blindly reduce the number of replicas to eliminate alerts.

---

## Section XI: Fault 5: PG Stale Troubleshooting

### 11.1 Symptoms

The `ceph health detail` command displays:

PG_STALE

---

### 11.2 Meaning

"PG Stale" indicates that:

The MON has not received the latest status of this PG for a long time.

This usually suggests that the OSD responsible for this PG may be unreachable.

---

### 11.3 Common Causes

- OSD down
- Node where the OSD is located crashed
- Network interruption
- OSD process stuck
- Abnormal communication between MON and OSD
- Excessive node load causing heartbeat issues

---

### 11.4 Troubleshooting Commands

`ceph health detail`
`ceph osd tree`
`ceph orch ps --daemon_type osd`
`ceph pg <pg-id> query`

On the node side:

`ping <node-ip>`
`ssh root@<node>`
`top`
`iostat -x 1`
`dmesg | tail -100`
`journalctl -k | tail -100`

---

### 11.5 Handling Approach

- First, restore the OSD or node connectivity.
- Then check if the PG has recovered.
- Do not directly delete the PG.
- Do not attempt to repair a stale PG directly.

---

## Section XII: Fault 6: PG Inactive/Incomplete Troubleshooting

### 12.1 Symptoms

The `ceph -s` command displays:

PGs inactive
PGs incomplete

---

### 12.2 Meaning

"PG Inactive":

The PG cannot provide normal I/O operations.

"PG Incomplete":

Ceph is unable to find the complete history or replicas required by this PG.

These issues are more serious than "degraded" status.

---

### 12.3 Common Causes

- Multiple OSDs fail simultaneously
- Replica OSDs are lost
- OSD data corruption
- Changes in CRUSH rules causing mapping errors
- Forcible deletion of OSDs resulting in insufficient replicas
- Accidental disk cleanup operations
- Abnormal cluster capacity or metadata issues

---

### 12.4 Troubleshooting Commands

`ceph -s`
`ceph health detail`
`ceph pg stat`
`ceph pg dump_stuck inactive`
`ceph pg dump_stuck unclean`
`ceph pg <pg-id> query`
`ceph osd tree`
`ceph osd crush tree`

---

### 12.5 Handling Principles

This is a high-risk fault; it is not recommended to attempt direct repairs in a production environment based on experience alone.

Handling principles:

1. Preserve the original state of the system.
2. Do not continue deleting OSDs.
3. Do not perform any disk cleanup operations.
4. Avoid blind repairs.
5. Identify the active/available OSDs for the relevant PG.
6. Try to restore the original OSDs if possible.
7. Seek higher-level support if necessary.
8. Confirm backup measures and potential business impacts before proceeding.

### ceph mgr services

---

## Section Sixteen: Fault Ten: Troubleshooting CephFS / MDS Exceptions

### 16.1 Symptoms

Possible manifestations include:

- Failure to mount CephFS
- Lagging mounted directories
- Slow file operations
- MDS_SLOW_REQUEST
- Missing active MDS
- Missing standby MDS
- Kubernetes CephFS PVC FailedMount

---

### 16.2 Checking CephFS Status

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ceph -s
    ceph health detail

---

### 16.3 Common Causes

| Cause | Description |
|---|---|
| MDS not deployed | CephFS unavailable |
| Active MDS malfunction | File system unusable or lagging |
| Standby missing | Insufficient high availability |
| Metadata pool issue | Affecting metadata operations |
| Large number of small files | High load on MDS |
| Large directories | Slow ls / find operations |
| Client issues | Session or lock problems |
| OSD slow | Slow file data read/write |

---

### 16.4 Client Troubleshooting

Execute the following commands on the client:

    mount | grep ceph
    df -hT
    dmesg | tail -100
    lsof +f -- <mount-path>
    fuser -vm <mount-path>

---

### 16.5 Solutions

    MDS not deployed: Deploy the MDS.
    Active MDS malfunction: Check MDS logs and restart the faulty daemon.
    Standby missing: Add an additional MDS standby.
    Metadata pool issue: Address problems with the Pool/PG/OSD first.
    Slow processing of large numbers of small files: Analyze directory structure and MDS load.
    Client lagging: Investigate session, mount logs, and business processes.

---

## Section Seventeen: Fault Eleven: Troubleshooting RGW / S3 Exceptions

### 17.1 Symptoms

Possible manifestations include:

- Failure to access the S3 Endpoint
- Inaccessible curl ports
- AWS CLI reports AccessDenied
- SignatureDoesNotMatch errors
- InvalidAccessKeyId issues
- Bucket creation failure
- Object upload failure
- Abnormal access after using a reverse proxy

---

### 17.2 Checking RGW Services

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw

Test ports:

    curl -I http://10.0.0.31:7480
    curl -I http://10.0.0.32:7480
    curl -I http://10.0.0.33:7480

---

### 17.3 Checking User Information

    radosgw-admin user info --uid=<uid>

Verify:

- access_key
- secret_key
- user_quota
- bucketquota
- suspended status

---

### 17.4 Checking Buckets

    radosgw-admin bucket list
    radosgw-admin bucket stats --bucket=<bucket-name>

---

### 17.5 AWS CLI Troubleshooting

Check configurations:

    cat ~/.aws/config
    cat ~/.aws/credentials

Perform tests:

    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 ls

Common issues:

| Error | Possible Cause |
|---|---|
| InvalidAccessKeyId | Incorrect AccessKey or non-existent user |
| SignatureDoesNotMatch | Incorrect SecretKey, host change, time discrepancy, proxy modification |
| AccessDenied | Insufficient permissions |
| NoSuchBucket | Bucket does not exist |
| RequestTimeTooSkewed | Time mismatch |
| Connection timed out | Issues with RGW or network connectivity |

---

### 17.6 Reverse Proxy Troubleshooting

If using Nginx / LB, check the following:

- HTTPS certificate
- Host header settings
- Whether proxy_pass modifies the path
- client_max_body_size setting
- proxy_request_buffering configuration
- Health status of the backend RGW
- Nginx access log/error log

Key configurations:

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_request_buffering off;
    proxy_buffering off;
    client_max_body_size 0;

---

## Section Eighteen: Fault Twelve: Troubleshooting RBD Exceptions

### 18.1 RBD map failure

Troubleshoot by:

    ceph -s
    rbd ls -p <pool-name>
    rbd info <pool-name>/<image-name>
    ls -l /etc/ceph/
    dmesg | tail -100

Common causes:

- Missing ceph.conf file
- The SubvolumeGroup does not exist.
- The MDS is not active.
- There is an error with the Secret.
- The Ceph user does not have sufficient permissions.
- The clusterID is incorrect.
- The MON is unreachable.

Troubleshooting steps:

    kubectl get storageclass
    kubectl get pods -n ceph-csi
    kubectl logs -n ceph-csi <cephfs-provisioner-pod>
    ceph fs ls
    ceph fs status
    ceph mds stat
    ceph fs subvolumegroup ls cephfs
    ceph auth get client.k8s-cephfs

---

### 20.2 Pod FailedMount

Check:

    kubectl describe pod -n <namespace> <pod-name>

Node plugin logs:

    kubectl logs -n ceph-csi <cephfs-node-pod> -c csi-cephfsplugin

On the node side:

    modprobe ceph
    lsmod | grep ceph
    dmesg | tail -100

Common causes:

- MDS exception
- Secret error
- Incompatible mounter
- The node cannot connect to the MON network
- Insufficient permissions
- Abnormality in the CephFS metadata pool

---

## Chapter 21: Troubleshooting Slow Operations and Performance Issues

### 21.1 Symptoms

The ceph health detail may display:

    SLOW_OPS
    Slow requests are blocked

Business performance impacts:

- Slow RBD read and write operations
- Slow CephFS ls / write operations
- Slow RGW upload and download operations
- Slow Kubernetes Pod I/O operations
- High database latency

---

### 21.2 Initial Troubleshooting

    ceph -s
    ceph health detail
    ceph osd perf
    ceph osd df
    ceph pg stat

Check for the following issues:

- Recovery
- Backfill
- Degraded status
- Near-full capacity
- High OSD latency
- Abnormal OSD usage rates
- Slow operations

---

### 21.3 Troubleshooting High OSD Latency

    ceph osd perf

If a particular OSD shows significantly high latency, locate the relevant node:

    ceph osd metadata osd.X

Log in to the node and check:

    iostat -x 1
    vmstat 1
    top
    dmesg | grep -i error
    journalctl -k | grep -i error

---

### 21.4 Common Causes

| Cause | Description |
|---|---|
| Slow or faulty disk | High OSD latency |
- Recovery / Backfill processes | Consuming resources in the background |
- Network packet loss or delay | Affecting OSD heartbeat and I/O operations |
- Excessive OSD usage | Performance degradation when approaching full capacity |
- High pressure on a single pool | Especially for hot workload scenarios |
- Large number of small I/O requests | Additional load on metadata and random I/O operations |
- High MDS load | Slowing down CephFS performance |
- Restrictions with RGW front-end proxies | Slowening upload and download speeds |
- Issues with Kubernetes nodes | CSI mounting or I/O-related problems |

---

### 21.5 Handling Principles

    First, determine whether the system is currently in recovery mode.
    Next, identify if only a single OSD is experiencing slow performance.
    Then, check the node's disk and network connectivity.
    Examine the business I/O patterns as well.
    Avoid making random adjustments to recovery parameters without proper understanding of the issue.
    Do not forcibly increase the recovery settings without assessing the potential impacts.

---

## Chapter 22: Time Synchronization Issues

### 22.1 Symptoms

Possible issues include:

    clock skew detected
    MON_clock_SKEW
    Authentication errors
    Abnormal S3 signature verification
    Disordered log timestamps

---

### 22.2 Troubleshooting Steps

Execute the following commands on all nodes:

    timedatectl
    chronyc sources -v

Check the chrony service status:

Ubuntu:

    systemctl status chrony

Rocky Linux:

    systemctl status chronyd

---

### 22.3 Resolution

Start time synchronization:

Ubuntu:

    systemctl enable --now chrony

Rocky Linux:

    systemctl enable --now chronyd

Set the time zone:

    timedatectl set-timezone Asia/Shanghai

---

### 22.4 Production Recommendations

    All Ceph nodes must use the same time source.
    The same should apply to Kubernetes nodes as well.
    Ensure that RGW and S3 clients have accurate time settings.
    Time synchronization issues can significantly increase the difficulty of authentication processes and troubleshooting efforts.

---

## Chapter 23: Network Issues Troubleshooting

### 23.1 Symptoms

Possible manifestations🔤 When a failure occurs, first protect the scene and do not rush to clean it up.
🔤 Check ceph -s and ceph health detail first.
🔤 Do not execute deletion commands without understanding the scope of impact.
🔤 Do not make unrelated changes when the cluster is unhealthy.
🔤 Do not handle multiple issues simultaneously to avoid expanding the impact.
🔤 When an OSD is down, locate the cause before directly purging it.
🔤 Do not blindly repair inconsistent PGs.
🔤 For nearfull/full capacity situations, prioritize capacity management rather than simply adjusting thresholds.
🔤 In case of MON quorum anomalies, restore network, time, and services first.
🔤 When MDS is abnormal, pay attention to the metadata pool and client sessions.
🔤 For RGW abnormalities, check the client, proxy layer, RGW, and Ceph backend simultaneously.
🔤 For CSI abnormalities, examine Kubernetes, CSI, nodes, and Ceph together.
🔤 All high-risk operations must record the commands and execution times.
🔤 After a failure is resolved, conduct a post-event review.
🔤 The focus of the review should not be on assigning blame but on enhancing monitoring, processes, and drills.

---

## Chapter 27: Advanced SRE Methodologies

### 27.1 Ceph Fault Diagnosis Should Be Layered

Recommended layering:

    Business Layer:
      Pod / VM / Application / S3 Client

    Access Layer:
      CSI / RBD / CephFS / RGW / Nginx / LB

    Ceph Service Layer:
      MON / MGR / MDS / RGW / OSD

    Data Distribution Layer:
      Pool / PG / CRUSH / OSD Set

    Node Resource Layer:
      CPU / Memory / Disk / Network / Time Synchronization

---

### 27.2 Ceph Failures Are Not Always Related to Storage Only

Ceph failures can stem from:

- Disks
- Networks
- Time
- Container runtime
- Operating system
- Kubernetes
- Permissions
- Proxy layer
- Application I/O modes
- Operational errors

Therefore, it is not sufficient to focus only on Ceph commands.

---

### 27.3 Prioritize Restoring Business Operations Before Addressing the Root Cause

Fault resolution typically involves two steps:

    Phase 1: Restore business availability.
    Phase 2: Analyze the root cause and prevent recurrence.

Do not let the business remain unavailable for an extended period while trying to find the root cause.

However, also do not stop conducting post-event reviews after the business has been restored.

---

### 27.4 Automatic Recovery Does Not Mean the Fault Is Over

Although Ceph can automatically restore replicas, it is still necessary to analyze:

- Why did the OSD go down?
- Why was there a network interruption?
- Why was the disk latency high?
- Why was there abnormal capacity growth?
- Why did the MDS slow down?
- Why did the RGW authentication fail?
- Why were there numerous CSI FailedMounts?

---

### 27.5 Fault Drills Are More Important Than Ad-hoc Handling

Advanced SRE professionals should not rely solely on real-world failures for learning.

Regular drills should include:

- Single OSD failure
- Single node failure
- Adding new OSDs for expansion
- Replacing OSDs
- RBD recovery
- CephFS/MDS failover
- RGW instance failures
- CSI PVC mounting issues
- Nearfull capacity management
- Backup and recovery procedures

---

## Chapter 28: Interview Answer Guidelines

If asked in an interview:

    How would you diagnose a HEALTHWARN issue in a Ceph cluster?

You could respond like this:

    I would first execute ceph -s to check the overall status, including health, MON, MGR, OSDs, placement groups, usage, and whether there are any recovery/backfill processes in progress. Then, I would use ceph health detail to examine specific alarm types.
    If an OSD is down, I would use ceph osd tree to identify the affected OSD, followed by ceph osd metadata osd.X to confirm its node and disk location. Next, I would check ceph orch ps --daemon_type osd to gather more details, and then log in to the node to inspect the disk, system logs, network status, and resource usage.
    If a placement group is degraded or undersized, I would review ceph pg stat, ceph osd tree, Pool size/min_size, CRUSH Rules, and fault domains to determine if the issue is due to an OSD failure, recovery process, insufficient number of OSDs, or inadequate fault domain replication.
    For capacity-related alarms, I would compare ceph df and ceph osd df to identify whether the overall capacity is insufficient or if there is uneven distribution among OSDs. Based on the