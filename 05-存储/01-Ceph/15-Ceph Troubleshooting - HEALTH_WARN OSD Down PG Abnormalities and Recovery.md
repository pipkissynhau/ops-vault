# Ceph Troubleshooting: Common Issues with OSD, PG, MON, MDS, RGW, and CSI

Suggested path: 05-Storage/01-Ceph/15-Ceph Troubleshooting: OSD, PG, MON, MDS, RGW, and CSI Common Issues.md

Tags: #Ceph #FaultCheck. #OSD #PG #MON #MGR #MDS #RGW #RBD #CephFS #CSI #SlowOps #AdvancedSre

---

## I. Document Explanation

This is the fifteenth article of the Ceph advanced SRE storage module, focusing on common troubleshooting methods for Ceph.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH fault domain
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations and maintenance inspection

This article enters the troubleshooting phase.

Ceph troubleshooting should not only memorize commands but establish a complete judgment chain:

    First check overall health
      |
      v
    Then check health details
      |
      v
    Then locate the faulty object
      |
      v
    Then assess the impact scope
      |
      v
    Then decide whether manual intervention is needed
      |
      v
    Finally verify the recovery result

This article covers:

- Common troubleshooting methods
- HEALTH_WARN / HEALTH_ERR judgment
- OSD down troubleshooting
- OSD full / nearfull troubleshooting
- PG degraded / undersized / stale / inactive / inconsistent troubleshooting
- MON quorum anomaly troubleshooting
- MGR anomaly troubleshooting
- MDS / CephFS anomaly troubleshooting
- RGW / S3 access anomaly troubleshooting
- RBD mounting and lock anomaly troubleshooting
- Kubernetes CSI PVC Pending / FailedMount troubleshooting
- Slow Ops and performance slowdown troubleshooting
- Network, disk, and time synchronization issues troubleshooting
- High-risk command boundaries for fault handling
- Fault review template

---

## II. Troubleshooting Objectives

After completing this article, you should be able to:

1. Determine the overall cluster status quickly using `ceph -s`.
2. Locate the alert type using `ceph health detail`.
3. Distinguish the priority between HEALTH_WARN and HEALTH_ERR.
4. Troubleshoot OSD down, OSD out, OSD full, and OSD high latency issues.
5. Troubleshoot PG degraded, undersized, stale, inactive, incomplete, and inconsistent issues.
6. Troubleshoot MON quorum anomalies and clock skew.
7. Troubleshoot MGR active missing, Dashboard unavailability, and Prometheus collection failure.
8. Troubleshoot CephFS MDS anomalies, mounting failure, and access slowness.
9. Troubleshoot RGW access failure, S3 authentication failure, Bucket anomalies, and proxy layer issues.
10. Troubleshoot RBD map failure, unmap failure, lock residue, and snapshot deletion failure.
11. Troubleshoot common issues with Kubernetes RBD / CephFS CSI.
12. Understand which commands are read-only troubleshooting commands and which are high-risk repair commands.
13. Establish a standard process for fault response, handling, verification, and review.

---

## III. Experimental and Production Environments

### 3.1 Ceph Cluster Nodes

This article continues using the Ceph module experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (Optional) |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client Testing (Optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Kubernetes Cluster

If troubleshooting CSI, the Kubernetes cluster is involved:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Runtime environment:

    kubeadm
    containerd
    Calico
    Ubuntu Server 22.04.5 LTS

---

## IV. Ceph Troubleshooting Basic Principles

### 4.1 First Check the Whole, Then the Local

Troubleshooting entry point:

    `ceph -s`
    `ceph health detail`

Do not log in to nodes to operate disks or execute repair commands directly.

Standard order:

    1. `ceph -s` to check overall status
    2. `ceph health detail` to check specific alerts
    3. `ceph osd tree` to check OSD status
    4. `ceph pg stat` to check PG status
    5. `ceph osd df` / `ceph df` to check capacity
    6. `ceph orch ps` to check services
    7. Log in to the corresponding node to check system resources and logs

---

### 4.2 First Determine Whether It Affects Business

After seeing alerts, first determine:

| Issue | May Affect Business |
|---|---|
| Single OSD down, but replicas are sufficient | May temporarily not affect business, but with risk |
| PG degraded | Business may be usable, but data redundancy decreases |
| PG inactive | May affect business read/write |
| OSD full | May affect writing |
| MON quorum anomaly | May affect cluster control plane |
| MDS active missing | CephFS may be unavailable |
| RGW all anomalies | S3 service unavailable |
| CSI FailedMount | Corresponding Pod storage unavailable |

---

### 4.3 Do Not Rush to Execute High-Risk Commands

The following commands are high-risk operations: /think

ceph osd purge
ceph osd rm
ceph osd crush remove
ceph osd pool rm
ceph pg repair
ceph fs rm
rbd rm
rbd snap purge
radosgw-admin user rm --purge-data
wipefs -a
sgdisk --zap-all
cephadm rm-cluster --force --zap-osds

Before executing in production environment:

    Is there a backup?
    Is there a change approval?
    Is there a rollback plan?
    Is the impact scope clearly defined?
    Is it within maintenance window?
    Has the business been notified?

---

### 4.4 What can automatically recover does not necessarily require handling

For example:

    OSD temporarily down then recover
    PG degraded then automatically recovery
    Recovered to HEALTH_OK after backfill completes

This does not mean the fault has ended.

Still need to investigate:

    Why did OSD go down?
    Is it a disk issue?
    Is it a network interruption?
    Is it a node reboot?
    Is it a container runtime anomaly?
    Is it a full system disk?
    Is it a time sync anomaly?
    Is it resource exhaustion?

Automatic recovery solves the result, but not necessarily the root cause.

---

## Five, Fault Grading Recommendations

### 5.1 P3: Minor Issues

Examples:

- POOL_APP_NOT_ENABLED
- Dashboard certificate reminder
- Non-critical module prompts
- pg autoscaler suggestions
- Configuration reminders without business impact

Handling:

    Record.
    Schedule for later.
    No need to interrupt business immediately.

---

### 5.2 P2: Moderate Fault

Examples:

- Single OSD down
- Few PG degraded
- MGR standby missing
- MDS standby missing
- Single RGW instance abnormal but service still available
- Single PVC Pending or FailedMount

Handling:

    Handle within the day.
    Observe recovery process.
    Record the cause.
    Notify business if necessary.

---

### 5.3 P1: Severe Fault

Examples:

- Multiple OSD down
- PG inactive
- PG incomplete
- OSD full
- MON quorum lost
- CephFS active MDS missing
- RGW completely unavailable
- Large number of PVC FailedMount
- Large number of slow ops with noticeable business impact

Handling:

    Immediate response.
    Stop non-essential changes.
    Preserve the scene.
    Notify relevant personnel.
    Prioritize business recovery.
    Must conduct a post-mortem after completion.

---

## Six, General Troubleshooting Command List

### 6.1 Cluster-side Read-only Commands

    ceph -s
    ceph health detail
    ceph osd stat
    ceph osd tree
    ceph osd df
    ceph pg stat
    ceph df
    ceph orch ps
    ceph orch ls
    ceph crash ls-new

---

### 6.2 MON / MGR Commands

    ceph mon stat
    ceph quorum_status --format json-pretty
    ceph mgr stat
    ceph mgr services
    ceph orch ps --daemon_type mon
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

### 6.5 Node-side Commands

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

Check services:

    ceph orch ps

Check logs for specific daemon:

    cephadm logs --name osd.0
    cephadm logs --name mon.ceph-node01
    cephadm logs --name mgr.ceph-node01

Actual daemon names are determined by ceph orch ps output.

---

## Seven, Fault One: OSD Down Troubleshooting

### 7.1 Phenomenon

ceph -s may show:

    health: HEALTH_WARN
    1 osds down
    Degraded data redundancy

Check:

    ceph osd stat

May see:

    6 osds: 5 up, 6 in

Explanation:

    Total of 6 OSDs.
    Only 5 OSDs online.
    But all 6 OSDs are still in in state.

---

### 7.2 Determine Down OSD

Check:

    ceph osd tree

Example:

    osd.3 down in

This indicates:

    osd.3 process unavailable
    Still in CRUSH data distribution

This is a state requiring close attention.

---

### 7.3 Check Health Details

    ceph health detail

Focus on:

- Which OSD is down
- Which PGs are degraded
- Whether recovery is ongoing
- Whether there are slow requests
- Whether there are disk or network-related warnings

---

### 7.4 Check OSD Metadata /think

ceph osd metadata osd.3

Key Confirmations:

- hostname
- bluestore_bdev_dev_node
- devices
- ceph_version
- osd_objectstore

For Localization:

    osd.3 on which node
    corresponding disk

---

### 7.5 View cephadm Service

    ceph orch ps --daemon_type osd

Filter:

    ceph orch ps --daemon_type osd | grep osd.3

Confirm:

- Whether running
- Whether error
- Whether stopped
- Whether recent restart

---

### 7.6 Login to Fault Node for Troubleshooting

Assume osd.3 is on ceph-node02.

Login:

    ssh root@ceph-node02

Check hostname:

    hostname

Check disks:

    lsblk

Check system disk and data disk capacity:

    df -hT

Check kernel logs:

    dmesg | grep -i error
    dmesg | grep -i fail
    journalctl -k | grep -i error

Check system resources:

    top
    free -h
    iostat -x 1

Check services:

    systemctl status ceph-*.target

---

### 7.7 Common Causes

| Cause | Manifestation |
|---|---|
| Node failure | Multiple OSDs down on the same host |
| Disk failure | I/O error in dmesg |
| Container runtime anomaly | cephadm daemon not functioning |
| Network anomaly | OSD heartbeat failure |
| System disk full | Daemon cannot write logs or metadata |
| OSD data corruption | OSD cannot start |
| Time anomaly | Communication and authentication issues |

---

### 7.8 Handling Approach

If it's a temporary service anomaly, try restarting the OSD:

    ceph orch daemon restart osd.3

Monitor:

    ceph osd tree
    ceph -s

If it's a node failure:

    First restore node network, power, and system.
    Wait for OSD to auto-recover.
    Then observe PG recovery.

If disk damage is confirmed:

    Follow OSD replacement procedure.
    Do not directly wipefs.
    Do not directly purge.
    First confirm replicas, capacity, and impact scope.

---

### 7.9 Verification of Recovery

    ceph -s
    ceph osd tree
    ceph pg stat
    ceph health detail

Goals:

    OSD up/in
    PG active+clean
    HEALTH_OK or explainable alerts

---

## VIII. Fault Two: OSD full / nearfull Troubleshooting

### 8.1 Symptoms

ceph -s may show:

    OSD_NEARFULL
    OSD_BACKFILLFULL
    OSD_FULL

Or:

    full osd(s)
    nearfull osd(s)

---

### 8.2 Check Capacity

    ceph df
    ceph osd df

Focus on:

- Which OSD has the highest usage
- Whether only a single OSD is abnormal
- Whether overall capacity is high
- Which Pool is growing fastest
- Whether there is an abnormal Pool usage

---

### 8.3 Check Pool Capacity

    ceph df

Find the Pool with high usage.

Check Pool information:

    ceph osd pool get <pool-name> all

---

### 8.4 Common Causes

| Cause | Description |
|---|---|
| Business data normal growth | Need expansion |
| Test data not cleaned | Clean test objects, Images, Buckets |
| Too many RBD snapshots | Clean expired snapshots |
| RGW object growth | Need Bucket governance |
| OSD distribution imbalance | Check CRUSH / weight / PG |
| New OSD not yet balanced | Wait for rebalance |
| Pool PG count unreasonable | Evaluate with autoscaler |

---

### 8.5 Handling Principles

Priority:

    1. Confirm if overall capacity is insufficient.
    2. Confirm if single OSD is abnormal.
    3. Clean clearly unused data.
    4. Clean expired snapshots.
    5. Expand OSD or add nodes.
    6. Evaluate CRUSH / PG / weight.
    7. Do not directly delete unknown Pools.
    8. Do not simply raise full threshold to mask issues.

---

### 8.6 High-Risk Warnings

The following operations must be handled with caution:

    ceph osd pool rm
    rbd rm
    rbd snap purge
    radosgw-admin bucket rm
    radosgw-admin user rm --purge-data

Confirm before deletion:

    Data ownership
    Whether still used by business
    Whether there is a backup
    Whether approved

---

## IX. Fault Three: PG degraded Troubleshooting

### 9.1 Symptoms

ceph -s may show:

    degraded data redundancy
    xxx objects degraded
    xxx pgs degraded

PG status may be:

    active+degraded

---

### 9.2 Basic Meaning

PG degraded indicates:

    PG replica count is insufficient.
    But PG is still active, usually still provides IO.

Common scenarios:

- OSD down
- OSD out
- OSD in recovery
- After adding OSD and rebalancing
- CRUSH failure domain insufficient
- Pool size set exceeds available failure domain count

---

### 9.3 Troubleshooting Commands

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree
    ceph osd df

If you know the PG ID:

    ceph pg <pg-id> query
    ceph pg map <pg-id>

---

### 9.4 Determine if Recovery is Ongoing

Check:

    ceph -s

If you see:

    recovery io
    recovering
    backfilling

It indicates Ceph is automatically recovering.

Monitor continuously: /think

watch -n 5 'ceph -s'

---

### 9.5 Handling Principles

If an OSD recovers after a brief down:

    Observe whether recovery progresses.
    Until active+clean.

If an OSD is down for a long time:

    Investigate the cause of the OSD.
    Confirm whether the disk needs to be replaced.

If the fault domain is insufficient:

    Check the CRUSH Rule.
    Check the size.
    Check the number of hosts / racks.

---

## Ten. Fault Four: PG Undersized Troubleshooting

### 10.1 Symptoms

ceph -s shows:

    pgs undersized

---

### 10.2 Meaning

PG undersized indicates:

    The current number of replicas in the PG is less than the Pool's size.

Example:

    Pool size = 3
    But some PGs only have 2 replicas

---

### 10.3 Common Causes

| Cause | Explanation |
|---|---|
| Insufficient OSDs | size=3, but available OSDs are insufficient |
| Insufficient hosts | insufficient available hosts in the host fault domain |
| Insufficient racks | insufficient racks in the rack fault domain |
| OSD down/out | insufficient replicas in available locations |
| Unreasonable CRUSH Rule settings | fault domain level is too high |
| OSD mistakenly removed from CRUSH | data cannot be placed normally |

---

### 10.4 Diagnostic Commands

    ceph health detail
    ceph pg stat
    ceph osd tree
    ceph osd crush tree
    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> crush_rule
    ceph osd crush rule dump <rule-name>

---

### 10.5 Handling Approach

Address based on the cause:

    OSD down: Recover the OSD.
    Insufficient OSDs: Add new OSDs.
    Insufficient fault domain: Add hosts / racks, or adjust the CRUSH Rule.
    Size set too high: In experimental environments, reduce size cautiously in production.
    CRUSH error: Fix the CRUSH topology.

Production recommendations:

    Prioritize restoring resources and fault domains.
    Do not blindly reduce replica counts to eliminate alerts.

---

## Eleven. Fault Five: PG Stale Troubleshooting

### 11.1 Symptoms

ceph health detail shows:

    PG_STALE

---

### 11.2 Meaning

PG stale indicates:

    The MON has not received the latest status of the PG for a long time.

This typically suggests that the OSD responsible for the PG may be unreachable.

---

### 11.3 Common Causes

- OSD down
- Node where OSD resides is down
- Network interruption
- OSD process is frozen
- Communication anomaly between MON and OSD
- High node load causing heartbeat anomalies

---

### 11.4 Diagnostic Commands

    ceph health detail
    ceph osd tree
    ceph orch ps --daemon_type osd
    ceph pg <pg-id> query

Node-side:

    ping <node-ip>
    ssh root@<node>
    top
    iostat -x 1
    dmesg | tail -100
    journalctl -k | tail -100

---

### 11.5 Handling Approach

    First restore OSD or node connectivity.
    Then observe if the PG recovers.
    Do not directly delete the PG.
    Do not directly repair stale PG.

---

## Twelve. Fault Six: PG Inactive / Incomplete Troubleshooting

### 12.1 Symptoms

ceph -s shows:

    pgs inactive
    pgs incomplete

---

### 12.2 Meaning

PG inactive:

    The PG cannot provide normal IO.

PG incomplete:

    Ceph cannot find the complete history or replicas required for the PG.

This type of issue is more severe than degraded.

---

### 12.3 Common Causes

- Multiple OSDs fail simultaneously
- Replicas' OSDs are lost
- OSD data corruption
- CRUSH changes causing mapping anomalies
- Forced OSD deletion leading to insufficient replicas
- Accidental disk cleanup
- Cluster capacity or metadata anomalies

---

### 12.4 Diagnostic Commands

    ceph -s
    ceph health detail
    ceph pg stat
    ceph pg dump_stuck inactive
    ceph pg dump_stuck unclean
    ceph pg <pg-id> query
    ceph osd tree
    ceph osd crush tree

---

### 12.5 Handling Principles

This is a high-risk fault; do not handle it based on experience in production.

Handling principles:

    1. Preserve the scene.
    2. Do not continue deleting OSDs.
    3. Do not clean disks.
    4. Do not blindly repair.
    5. Identify the acting / up OSDs related to the PG.
    6. Restore the original OSDs as much as possible.
    7. Seek higher-level support if necessary.
    8. Confirm backups and business impact before handling.

---

## Thirteen. Fault Seven: PG Inconsistent Troubleshooting

### 13.1 Symptoms

ceph health detail shows:

    PG_DAMAGED
    pgs inconsistent

---

### 13.2 Meaning

PG inconsistent indicates:

    Inconsistencies are found among multiple replicas within the PG.

This is typically discovered by scrub / deep scrub.

---

### 13.3 Diagnostic Commands

    ceph health detail
    ceph pg <pg-id> query
    ceph pg map <pg-id>

Check OSDs:

    ceph osd tree
    ceph osd perf

---

### 13.4 Should You Repair Directly

Do not immediately:

    ceph pg repair <pg-id>

Repair is a possible handling method, but not the first step.

Correct order:

1. Record PG ID.
2. Check ceph health detail.
3. Check pg query.
4. Determine if there are OSD disk errors.
5. Check related OSD logs.
6. Confirm replicas and business impact.
7. Execute repair if necessary.

---

### 13.5 repair Command

Execute after confirmation:

    ceph pg repair <pg-id>

Observe after execution:

    ceph -s
    ceph health detail
    ceph pg stat

High-risk warning:

    It is recommended to confirm backups and impact scope before execution in production environments.
    If inconsistent frequently occurs, focus on checking disks, memory, controllers, and OSD stability.

---

## FourteenI don't know.Fault Eight: MON Quorum Abnormality Troubleshooting

### 14.1 Phenomenon

May manifest as:

    mon quorum missing
    ceph -s hangs
    MON_DOWN
    clock skew detected
    Unable to execute some management commands normally

---

### 14.2 Check MON Status

    ceph mon stat
    ceph quorum_status --format json-pretty
    ceph orch ps --daemon_type mon

---

### 14.3 Common Causes

| Cause | Explanation |
|---|---|
| MON node failure | Quorum count insufficient |
| Network partition | MONs cannot communicate |
| Time drift | Clock skew |
| System disk full | MON cannot write |
| MON container abnormal | Daemon not running |
| Configuration error | Address, port, or monmap issues |

---

### 14.4 Node-side Troubleshooting

Login to MON node:

    ssh root@ceph-node01

Check time:

    timedatectl
    chronyc sources -v

Check disk:

    df -hT

Check ports:

    ss -lntp | grep -E '3300|6789'

Check logs:

    cephadm logs --name mon.ceph-node01

Or:

    journalctl -u ceph-*.target --no-pager | tail -100

---

### 14.5 Handling Principles

    If it's a time issue, fix time synchronization first.
    If it's a network issue, restore MON communication first.
    If the disk is full, clean the system disk first.
    If daemon is abnormal, try restarting.
    Do not delete MONs arbitrarily.
    Do not forcibly rebuild without understanding monmap.

---

## FifteenI don't know.Fault Nine: MGR Abnormality Troubleshooting

### 15.1 Phenomenon

May manifest as:

- Dashboard inaccessible
- ceph orch command anomalies
- Prometheus collection anomalies
- ceph mgr stat has no active
- MGR module anomalies

---

### 15.2 Check MGR

    ceph mgr stat
    ceph orch ps --daemon_type mgr
    ceph mgr services
    ceph mgr module ls

---

### 15.3 Common Causes

- MGR daemon anomalies
- Active MGR node failure
- MGR module anomalies
- Dashboard module not enabled
- Prometheus module not enabled
- MGR port blocked by firewall

---

### 15.4 Handling Commands

Restart MGR:

    ceph orch daemon restart mgr.<mgr-daemon-name>

Actual name via:

    ceph orch ps --daemon_type mgr

Check.

Enable Dashboard:

    ceph mgr module enable dashboard

Enable Prometheus:

    ceph mgr module enable prometheus

Check services:

    ceph mgr services

---

## SixteenI don't know.Fault Ten: CephFS / MDS Abnormality Troubleshooting

### 16.1 Phenomenon

May manifest as:

- CephFS mount failure
- Mounted directory hangs
- File operations are slow
- MDS_SLOW_REQUEST
- Active MDS missing
- Standby MDS missing
- Kubernetes CephFS PVC FailedMount

---

### 16.2 Check CephFS Status

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ceph -s
    ceph health detail

---

### 16.3 Common Causes

| Cause | Explanation |
|---|---|
| MDS not deployed | CephFS unavailable |
| Active MDS anomaly | Filesystem unavailable or hangs |
| Standby missing | Insufficient high availability |
| Metadata pool anomaly | Metadata operations affected |
| Large number of small files | MDS under pressure |
| Large directory | ls / find slow |
| Client anomaly | Session or lock issues |
| OSD slow | File data read/write slow |

---

### 16.4 Client Troubleshooting

Execute on client:

    mount | grep ceph
    df -hT
    dmesg | tail -100
    lsof +f -- <mount-path>
    fuser -vm <mount-path>

---

### 16.5 Handling Approach

    MDS not deployed: Deploy MDS.
    Active MDS anomaly: Check MDS logs, restart abnormal daemon.
    Standby missing: Add MDS standby.
    Metadata pool anomaly: Handle Pool/PG/OSD first.
    Slow with large number of small files: Analyze directory structure and MDS load.
    Client stuck: Combine session, mount logs, and business process troubleshooting.

---

## SeventeenI don't know.Fault Eleven: RGW / S3 Abnormality Troubleshooting

### 17.1 Phenomenon

May manifest as:

- S3 Endpoint access failure
- curl port unreachable
- AWS CLI reports AccessDenied
- SignatureDoesNotMatch
- InvalidAccessKeyId
- Bucket creation failure
- Object upload failure
- Abnormal access after reverse proxy

---

### 17.2 View RGW Service

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw

Test ports:

    curl -I http://10.0.0.31:7480
    curl -I http://10.0.0.32:7480
    curl -I http://10.0.0.33:7480

---

### 17.3 View Users

    radosgw-admin user info --uid=<uid>

Check:

- access_key
- secret_key
- user_quota
- bucket_quota
- suspended

---

### 17.4 View Bucket

    radosgw-admin bucket list
    radosgw-admin bucket stats --bucket=<bucket-name>

---

### 17.5 AWS CLI Troubleshooting

Check configuration:

    cat ~/.aws/config
    cat ~/.aws/credentials

Test:

    aws --profile ceph-rgw --endpoint-url http://10.0.0.31:7480 s3 ls

Common issues:

| Error | Possible Cause |
|---|---|
| InvalidAccessKeyId | AccessKey error or user does not exist |
| SignatureDoesNotMatch | SecretKey error, Host change, time desynchronization, proxy rewrite |
| AccessDenied | Insufficient permissions |
| NoSuchBucket | Bucket does not exist |
| RequestTimeTooSkewed | Time desynchronization |
| Connection timeout | RGW or network anomaly |

---

### 17.6 Reverse Proxy Troubleshooting

If passing through Nginx / LB, need to check:

- HTTPS certificate
- Host header
- Whether proxy_pass rewrite path
- client_max_body_size
- proxy_request_buffering
- Backend RGW health
- Nginx access log / error log

Key configuration:

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_request_buffering off;
    proxy_buffering off;
    client_max_body_size 0;

---

## Eighteen, Fault Twelve: RBD Abnormality Troubleshooting

### 18.1 RBD map failure

Troubleshoot:

    ceph -s
    rbd ls -p <pool-name>
    rbd info <pool-name>/<image-name>
    ls -l /etc/ceph/
    dmesg | tail -100

Common causes:

- ceph.conf does not exist
- keyring does not exist
- MON not reachable
- Insufficient permissions
- Image does not exist
- RBD feature incompatible with kernel

---

### 18.2 RBD feature incompatibility

Symptoms:

    rbd map failure
    feature mismatch in dmesg

Resolution:

Create an image with conservative features:

    rbd create demo-image-basic \
      --size 10G \
      -p rbd-pool \
      --image-feature layering

Production recommendations:

    Standardize kernel version.
    Standardize ceph-common version.
    Standardize CSI version.
    Do not enable incompatible features arbitrarily.

---

### 18.3 RBD unmap failure

Symptoms:

    device is busy

Troubleshoot:

    rbd showmapped
    mount | grep rbd
    lsof +f -- <mount-path>
    fuser -vm <mount-path>

Resolution:

    Stop business processes.
    Exit mount directory.
    Unmount filesystem.
    Then rbd unmap.

---

### 18.4 RBD Image deletion failure

Troubleshoot:

    rbd status <pool>/<image>
    rbd snap ls <pool>/<image>
    rbd children <pool>/<image>@<snap>
    rbd showmapped

Common causes:

- Image is being used by client
- Snapshot exists
- Clone dependency exists
- Lock exists
- Insufficient permissions

---

### 18.5 RBD lock residue

Check:

    rbd lock list <pool>/<image>

High-risk resolution:

    rbd lock remove <pool>/<image> <lock-id> <locker>

Reminder:

    Must confirm client has abnormally exited before deleting the lock.
    Force-deleting a lock while client is writing may cause data risks.

---

## Nineteen, Fault Thirteen: Kubernetes RBD CSI Abnormality Troubleshooting

### 19.1 PVC Pending

Check:

    kubectl describe pvc -n <namespace> <pvc-name>

Common causes:

- StorageClass does not exist
- Provisioner name error
- CSI Controller abnormal
- Secret error
- clusterID error
- Pool does not exist
- Ceph user permissions insufficient
- MON not reachable

Troubleshoot:

    kubectl get storageclass
    kubectl get pods -n ceph-csi
    kubectl logs -n ceph-csi <rbd-provisioner-pod>
    ceph -s
    ceph auth get client.k8s-rbd
    ceph osd pool ls

---

### 19.2 Pod FailedMount

Check:

    kubectl describe pod -n <namespace> <pod-name>

Troubleshoot:

    kubectl get volumeattachment
    kubectl describe volumeattachment <name>
    kubectl logs -n ceph-csi <rbd-node-pod> -c csi-rbdplugin

Node side: /think

modprobe rbd
lsmod | grep rbd
dmesg | tail -100

Common Causes:

- Node plugin anomalies
- Secret error
- Node to Ceph network unreachable
- rbd module issue
- RBD feature incompatibility
- VolumeAttachment residue

---

## Twenty, Fault Fourteen: Kubernetes CephFS CSI Abnormality Troubleshooting

### 20.1 PVC Pending

Check:

    kubectl describe pvc -n <namespace> <pvc-name>

Common Causes:

- StorageClass error
- CephFS does not exist
- fsName error
- SubvolumeGroup does not exist
- MDS not active
- Secret error
- Ceph user permission insufficient
- clusterID error
- MON unreachable

Troubleshoot:

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

Node side:

    modprobe ceph
    lsmod | grep ceph
    dmesg | tail -100

Common Causes:

- MDS anomaly
- Secret error
- Mounter incompatibility
- Node to MON network unreachable
- Permission insufficient
- CephFS metadata pool anomaly

---

## Twenty-one, Fault Fifteen: Slow Ops and Performance Slow Troubleshooting

### 21.1 Phenomenon

ceph health detail may show:

    SLOW_OPS
    slow requests are blocked

Business performance:

- RBD read/write slow
- CephFS ls/write slow
- RGW upload/download slow
- Kubernetes Pod IO slow
- Database high latency

---

### 21.2 Preliminary Troubleshooting

    ceph -s
    ceph health detail
    ceph osd perf
    ceph osd df
    ceph pg stat

Check for:

- recovery
- backfill
- degraded
- nearfull
- OSD high latency
- OSD usage anomaly
- slow ops

---

### 21.3 OSD Latency Troubleshooting

    ceph osd perf

If a particular OSD shows significantly high latency, locate the node:

    ceph osd metadata osd.X

Login to the node:

    iostat -x 1
    vmstat 1
    top
    dmesg | grep -i error
    journalctl -k | grep -i error

---

### 21.4 Common Causes

| Cause | Explanation |
|---|---|
| Slow or faulty disk | OSD high latency |
| Recovery / Backfill | Background recovery consumes resources |
| Network packet loss or latency | OSD heartbeat and IO affected |
| OSD usage too high | Performance degradation when near full |
| Single Pool pressure too high | Hotspot business |
| Large number of small IO | High metadata and random IO pressure |
| MDS high pressure | CephFS slow |
| RGW frontend proxy limitation | Upload/download slow |
| Kubernetes node issues | CSI mounting or IO slow |

---

### 21.5 Handling Principles

    First determine if recovery is ongoing.
    Then locate if a single OSD is slow.
    Then check node disk and network.
    Then examine business IO mode.
    Do not arbitrarily adjust recovery parameters.
    Do not forcibly increase recovery without understanding the impact.

---

## Twenty-two, Fault Sixteen: Time Synchronization Anomaly

### 22.1 Phenomenon

May show:

    clock skew detected
    MON_CLOCK_SKEW
    Authentication anomaly
    S3 signature anomaly
    Log time disorder

---

### 22.2 Troubleshooting

Execute on all nodes:

    timedatectl
    chronyc sources -v

Check chrony:

Ubuntu:

    systemctl status chrony

Rocky Linux:

    systemctl status chronyd

---

### 22.3 Handling

Start time synchronization:

Ubuntu:

    systemctl enable --now chrony

Rocky Linux:

    systemctl enable --now chronyd

Set timezone:

    timedatectl set-timezone Asia/Shanghai

---

### 22.4 Production Recommendations

    Ceph nodes must use the same time source.
    Kubernetes nodes should also use the same time source.
    RGW / S3 client time must be accurate.
    Time anomalies amplify authentication and troubleshooting difficulty.

---

## Twenty-three, Fault Seventeen: Network Anomaly Troubleshooting

### 23.1 Phenomenon

May manifest as:

- OSD down
- MON quorum anomaly
- PG stale
- slow ops
- RBD / CephFS mount failure
- RGW access timeout
- CSI FailedMount

---

### 23.2 Basic Troubleshooting

Between nodes:

    ping -c 3 <target-ip>

Ports:

    nc -vz 10.0.0.31 3300
    nc -vz 10.0.0.31 6789
    nc -vz 10.0.0.31 6800

Routing:

    ip route

Network interface:

    ip addr
    ethtool <interface>

Connection:

    ss -antp

---

### 23.3 Ceph Common Ports /think

| Service | Port |
|---|---|
| MON v2 | 3300 |
| MON v1 | 6789 |
| OSD | 6800-7300 |
| Dashboard | 8443 |
| Prometheus exporter | 9283 |
| RGW | 7480, depends on configuration |

---

### 23.4 Handling Principles

    First, confirm node reachability.
    Then, verify port connectivity.
    Then, check firewall and security groups.
    Then, verify routing and network interface.
    Then, confirm packet loss, jitter, and MTU issues.

---

## Twenty-Four, High-Risk Command Boundaries

### 24.1 Read-Only Troubleshooting Commands

The following commands are typically safe:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph osd df
    ceph pg stat
    ceph df
    ceph orch ps
    ceph fs status
    ceph mds stat
    rbd ls
    rbd info
    radosgw-admin user info
    kubectl get
    kubectl describe
    kubectl logs

---

### 24.2 Medium-Risk Commands

The following commands may affect services or recovery processes:

    ceph orch daemon restart
    ceph osd out
    ceph osd in
    ceph osd set noout
    ceph osd unset noout
    ceph config set
    rbd snap rollback
    kubectl delete pod

---

### 24.3 High-Risk Commands

The following commands may cause data loss or widespread impact:

    ceph osd purge
    ceph osd rm
    ceph osd crush remove
    ceph osd pool rm
    ceph fs rm
    ceph pg repair
    rbd rm
    rbd snap purge
    radosgw-admin user rm --purge-data
    wipefs -a
    sgdisk --zap-all
    cephadm rm-cluster --force --zap-osds

Production environments must execute these commands after approval.

---

## Twenty-Five, Fault Handling Record Template

### 25.1 Basic Information

| Item | Content |
|---|---|
| Fault Time | YYYY-MM-DD HH:MM |
| Fault Level | P1 / P2 / P3 |
| Discovery Method | Patrol / Alert / User Feedback |
| Affected Business |  |
| Impact Scope |  |
| Handler |  |
| Current Status | Recovered / In Progress / Under Observation |

---

### 25.2 Fault Phenomenon

Record:

    Summary of ceph -s output
    Summary of ceph health detail output
    Business error messages
    Monitoring alerts
    User feedback

---

### 25.3 Troubleshooting Process

Record by timeline:

| Time | Operation | Result |
|---|---|---|
|  | ceph -s |  |
|  | ceph health detail |  |
|  | ceph osd tree |  |
|  | Login to node check |  |
|  | Execute troubleshooting action |  |

---

### 25.4 Root Cause Analysis

Fill in:

    Direct Cause:
    Root Cause:
    Triggering Factor:
    Is it detectable in advance:
    Is there missing monitoring:
    Is there missing process:

---

### 25.5 Recovery Verification

Verification items:

    ceph -s
    ceph health detail
    ceph pg stat
    Business read/write test
    Client mount test
    S3 upload/download test
    Kubernetes Pod mount test

---

### 25.6 Improvement Measures

Example:

    Add disk error monitoring.
    Optimize capacity alert thresholds.
    Add MDS standby.
    Add RGW multi-instance.
    Adjust patrol script.
    Supplement fault drills.
    Improve change process.

---

## Twenty-Six, Production Environment Fault Handling Notes

1. When a fault occurs, first protect the scene and do not rush to clean up.
2. First check ceph -s and ceph health detail.
3. Do not execute deletion commands without understanding the impact scope.
4. Do not make unrelated changes when the cluster is unhealthy.
5. Do not handle multiple directions simultaneously to avoid expanding the impact.
6. For OSD down, first locate the cause and do not directly purge.
7. For PG inconsistent, do not blindly repair.
8. For nearfull / full, prioritize capacity governance and do not simply adjust thresholds.
9. For MON quorum anomalies, prioritize recovering network, time synchronization, and services.
10. For MDS anomalies, focus on metadata pool and client sessions.
11. For RGW anomalies, check clients, proxy layer, RGW, and Ceph backend simultaneously.
12. For CSI anomalies, check Kubernetes, CSI, nodes, and Ceph simultaneously.
13. All high-risk operations must record commands and execution time.
14. After fault recovery, must conduct a post-mortem.
15. Post-mortem focus is not on accountability, but on supplementing monitoring, processes, and drills.

---

## Twenty-Seven, Advanced SRE Methodology

### 27.1 Ceph Fault Troubleshooting Should Be Layered

Recommended layers:

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

### 27.2 Ceph Faults Are Not Only Storage Issues

Ceph faults may originate from:

- Disk
- Network
- Time
- Container runtime
- Operating System
- Kubernetes
- Permissions
- Proxy layer
- Application IO mode
- Operational misoperation

Therefore, do not only focus on Ceph commands.

---

### 27.3 Prioritize Business Recovery, Then Address Root Causes

During fault handling, it's typically divided into two stages:

    First stage: Restore business availability.
    Second stage: Analyze root causes and prevent recurrence.

Do not let business be unavailable for a long time to chase root causes.

But also do not skip post-mortem after business recovery.

---

### 27.4 Automatic Recovery Does Not Equal Fault Resolution

Ceph can automatically recover replicas, but still need to analyze:

Why is OSD down?  
Why is there network interruption?  
Why is disk latency high?  
Why is capacity growth abnormal?  
Why is MDS slow?  
Why is RGW authentication failing?  
Why is CSI experiencingMass FailedMount?

---

### 27.5 Fault Drills Are More Important Than On-the-Spot Performance

Advanced SREs should not only learn from real failures.

Regular drills should include:

- Single OSD failure  
- Single node failure  
- Adding new OSDs for expansion  
- OSD replacement  
- RBD recovery  
- CephFS MDS failover  
- RGW instance failure  
- CSI PVC mounting anomalies  
- nearfull capacity governance  
- Backup recovery process  

---

## Twenty-Eight, Interview Response Strategy

If asked in an interview:

    What do you do when a Ceph cluster shows HEALTH_WARN?

You can respond:

    I would first run ceph -s to check the overall status, confirming health, mon, mgr, osd, pg, usage, and whether recovery/backfill is happening. Then I would execute ceph health detail to identify the specific alert type.  
    If it's OSD down, I would use ceph osd tree to locate the specific OSD, then check ceph osd metadata osd.X to confirm the node and disk, followed by reviewing ceph orch ps --daemon_type osd. Finally, I would log into the node to check disk, system logs, network, and resource status.  
    If it's PG degraded or undersized, I would examine ceph pg stat, ceph osd tree, Pool size/min_size, CRUSH Rule, and fault domain to determine if it's due to OSD failure, ongoing recovery, insufficient OSD count, or fault domain not meeting replica count.  
    If it's a capacity alert, I would check ceph df and ceph osd df to confirm if it's overall capacity shortage or single OSD imbalance, then assess Pool usage to determine if it's business growth, excessive snapshots, or uncleaned test data.  
    For CephFS, RGW, or CSI-related issues, I would separately check MDS, RGW daemon, Kubernetes PVC/PV/CSI Pod/VolumeAttachment objects.  
    Throughout the process, I would not directly execute high-risk commands like osd purge, pool rm, or pg repair without first confirming the impact scope, data safety, backup, and rollback plan.

---

## Twenty-Nine, Summary of This Section

This article primarily organizes common Ceph troubleshooting methods:

1. The entry point for Ceph troubleshooting is ceph -s and ceph health detail.  
2. For OSD down, locate the OSD, node, disk, service, and logs.  
3. For OSD nearfull/full, first analyze the source of capacity, do not directly raise thresholds.  
4. PG degraded typically indicates insufficient replicas but still operational.  
5. PG undersized is usually related to replica count, OSD count, and fault domain.  
6. PG stale is typically due to OSD unreachable or stale status.  
7. PG inactive/incomplete is a serious issue and should not be handled blindly.  
8. PG inconsistent requires cautious handling, do not blindly repair.  
9. MON quorum anomalies should focus on network, time, disk, and service.  
10. MGR anomalies affect Dashboard, Prometheus, and orchestrator.  
11. CephFS issues should focus on MDS and metadata pool.  
12. RGW issues require checking clients, gateway, proxy layer, and Ceph backend.  
13. RBD issues should examine Image, map, locks, snapshots, and client status.  
14. CSI issues should be investigated from Kubernetes, CSI, node, and Ceph layers.  
15. Slow Ops should focus on OSD latency, disk, network, and recovery tasks.  
16. Time synchronization and network anomalies amplify Ceph failures.  
17. High-risk commands must have approval, backup, rollback, and verification.  
18. Advanced SREs should establish a closed-loop process for fault response, handling, verification, and post-mortem analysis.

---

## Thirty, Reference Documents

Ceph Health Check:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Troubleshooting Overview:

    https://docs.ceph.com/en/latest/rados/troubleshooting/

Ceph OSD Troubleshooting:

    https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-osd/

Ceph MON Troubleshooting:

    https://docs.ceph.com/en/latest/rados/troubleshooting/troubleshooting-mon/

Ceph PG Documentation:

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph Pool Management:

    https://docs.ceph.com/en/latest/rados/operations/pools/

CephFS Management:

    https://docs.ceph.com/en/latest/cephfs/administration/

Ceph RGW Management:

    https://docs.ceph.com/en/latest/radosgw/admin/

Ceph RBD Documentation:

    https://docs.ceph.com/en/latest/rbd/

Ceph CSI Project:

    https://github.com/ceph/ceph-csi

Cephadm Operations:

    https://docs.ceph.com/en/latest/cephadm/operations/