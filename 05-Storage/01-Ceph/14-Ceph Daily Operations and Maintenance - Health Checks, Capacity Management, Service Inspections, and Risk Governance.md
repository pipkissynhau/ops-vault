# Ceph Daily Operations and Maintenance: Health Checks, Capacity Management, Service Inspections, and Risk Governance

Recommended path: 05-Storage/01-Ceph/14-Ceph Daily Operations and Maintenance: Health Checks, Capacity Management, Service Inspections, and Risk Governance.md

Tags: #Ceph #Daily Operations and Maintenance #Health Checks #Capacity Management #OSD #MON #MGR #MDS #RGW #RBD #CephFS #Inspections #SRE #Advanced SRE

---

## I. Document Overview

This article is the fourteenth in the Ceph Advanced SRE storage module series, focusing on daily operations and maintenance methods for Ceph clusters.

Previous topics have included:

- Ceph cluster deployment
- OSD management
- Pools and PGs
- CRUSH fault domains
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI

Starting from this article, we will delve into the phase of Ceph operations and maintenance governance.

After a Ceph cluster is deployed, the real challenge is not whether it can run at all, but rather:

    How to perform daily checks?
    How to determine if the cluster is healthy?
    How to identify capacity risks?
    How to detect abnormalities in OSDs?
    How to check for issues with PGs?
    How to ensure MON/MGR/MDS/RGW are functioning properly?
    Which alerts should be monitored?
    Which alerts require immediate attention?
    Which commands pose high risks?
    How to develop regular inspection habits?

The goal of this article is to establish a set of practical methods for daily Ceph inspections and operations.

---

## II. Experimental and Production Environments

### 2.1 Ceph Cluster Nodes

This article uses the experimental environment from the Ceph module series.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON/MGR/OSD/RGW/MDS |
| 10.0.0.32 | ceph-node02 | MON/MGR/OSD/RGW/MDS |
| 10.0.0.33 | ceph-node03 | MON/MGR/OSD/RGW |
| 10.0.0.34 | ceph-node04 | OSD (for expansion/fault testing, optional) |
| 10.0.0.35 | ceph-client | RBD/CephFS/RGW client testing, optional |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 2.2 Daily Operations and Maintenance Objects

Daily Ceph operations and maintenance focus on the following areas:

| Object | Key Points of Attention |
|---|---|---|
| Cluster | Overall health, version, capacity, alerts |
| MON | Quorum, quantity, clock, status |
| MGR | Active/standby status, Dashboard, Prometheus, orchestrator |
| OSD | Up/in status, capacity, latency, disk errors, recovery |
| PG | Active+clean status, degradation, undersized status, stale status, inconsistency |
| Pool | Number of replicas, number of PGs, capacity, application usage |
| CRUSH | Fault domains, OSD distribution, weights |
| RBD | Images, snapshots, locks, client mappings |
| CephFS | MDS, metadata pool, data pool, client sessions |
| RGW | Gateway instances, S3 users, Buckets, objects, quotas |
| CSI | Kubernetes PVCs, PVs, CSI Pods, mount events |
| Host | CPU, memory, disks, network, time synchronization, system logs |

---

## III. Daily Operations and Maintenance Objectives

The core objectives of Ceph daily operations and maintenance are:

1. To promptly identify any abnormalities in cluster health.
2. To quickly detect OSD downouts, disk failures, and node issues.
3. To timely recognize degraded, undersized, stale, or inconsistent PGs.
4. To early notice capacity-related risks.
5. To immediately identify unusual Pool usage patterns.
6. To swiftly detect issues with MDS/RGW/CSI interfaces.
7. To quickly address any client mounting or access problems.
8. To prevent data loss due to misoperations.
9. To establish a closed-loop process for inspections, alerts, changes, recoveries, and post-mortem analyses.
10. To elevate Ceph from merely being able to run to being effectively managed, governed, and restored.

---

## IV. Inspection Frequency Recommendations

### 4.1 Daily Inspections

Daily recommended checks include:

- ceph -s
- ceph health detail
- ceph osd tree
- ceph osd df
- ceph pg statImmediately stop unnecessary changes.
First, collect the current status.
Determine whether it affects business operations.
Prioritize restoring cluster health.
Do not blindly delete OSDs, Pools, or PGs.
If necessary, escalate the issue to handle as a fault event.

---

## VII. Health Detail Inspection

### 7.1 View Health Details

    ceph health detail

This command is the primary entry point for troubleshooting Ceph alerts.

If `ceph -s` displays HEALTH_WARN or HEALTH_ERR, you must continue with:

    ceph health detail

---

### 7.2 Common Alert Categories

| Alert Type | Direction |
|---|---|
| OSD_DOWN | OSD or node failure |
| PG_DEGRADED | Insufficient replicas or recovery in progress |
| PG_UNDERSIZED | Number of replicas below the required size |
| PG_STALE | PG status has not been updated for a long time |
| PG_INCONSISTENT | Replicas are inconsistent |
| MON_clock_SKEW | Node times are not synchronized |
| OSD_nearFULL | OSD is nearly full |
| OSD_FULL | OSD is full |
| POOL_APP_NOT_ENABLED | Pool application type is not enabled |
| SLOW_OPS | Slow operations exist in the cluster |
| MDS_SLOW_REQUEST | CephFS metadata requests are slow |
| RECENT_CRASH | A daemon recently crashed |

---

### 7.3 Handling Principles

Upon seeing an alert, analyze it in the following order:

    1. What is the type of alert?
    2. Does it affect data security?
    3. Does it affect business read and write operations?
    4. Is it automatically recovering?
    5. Is human intervention required?
    6. Are there any high-risk operational risks?
    7. Is it necessary to adjust the change window?
    8. Should the business team be notified?

---

## VIII. MON Daily Inspection

### 8.1 View MON Status

    ceph mon stat

Expected output:

    3 mons at {ceph-node01=...,ceph-node02=...,ceph-node03=...}, election epoch ..., quorum 0,1,2 ceph-node01,ceph-node02,ceph-node03

---

### 8.2 View Quorum Details

    ceph quorum_status --format json-pretty

Key areas to focus on:

- quorum_names
- monmap
- election_epoch
- outside_quorum
- extra_probe_peers
- sync_provider

Normal target:

    All expected MONs are included in the quorum.

---

### 8.3 View MON Services

    ceph orch ps --daemon_type mon

Expected output:

    mon.ceph-node01 running
    mon.ceph-node02 running
    mon.ceph-node03 running

---

### 8.4 Common MON Issues

| Issue | Possible Causes |
|---|---|
| MON not in quorum | Network issues, time drift, service failures |
| clock skew | Time synchronization problems |
| MON disk full | /var/lib/ceph or system disk is full |
| MON startup failed | Corrupted data directory, configuration errors, container issues |
| Quorum lost | Multiple MONs fail or network partitions occur |

---

### 8.5 MON Troubleshooting Commands

    ceph mon stat
    ceph quorum_status --format json-pretty
    ceph orch ps --daemon_type mon
    ceph health detail

For node-specific checks:

    hostname
    timedatectl
    chronyc sources -v
    df -hT
    journalctl -u ceph-*.target --no-pager | tail -100

---

## IX. MGR Daily Inspection

### 9.1 View MGR Status

    ceph mgr stat

Normal target:

    There is 1 active MGR.
    At least 1 standby MGR is available.

---

### 9.2 View MGR Services

    ceph orch ps --daemon_type mgr

---

### 9.3 View MGR Modules

    ceph mgr module ls

Common modules include:

- dashboard
- prometheus
- orchestrator
- iostat
- restful (depending on version and requirements)

---

### 9.4 View Dashboard/Prometheus Services

    ceph mgr services

Possible output:

    {
      "dashboard": "https://ceph-node01:8443/",
      "prometheus": "http://ceph-node01:9283/"
    }

---

### 9.5 Common MGR Issues

| Issue | Impact |
|---|---|
| Lack of active MGR | Issues with the dashboard, orchestrator, and some management functions |
| Inaccessible dashboard | Operational access is affected |
| Unreachable Prometheus endpoint | Monitoring data collection is impaired |
| Missing standby MGR | Insufficient    pg 1.0 maps to up [0,1,2] acting [0,1,2]

---

### 11.5 Viewing PG Details

    ceph pg <pg-id> query

This command produces a long output, which is suitable for in-depth troubleshooting.

Focus on:

- state
- up
- acting
- primary
- recovery_state
- blocked_by
- peer information

---

### 11.6 Principles for Handling PG Exceptions

Do not immediately attempt to repair a PG exception.

Follow this sequence:

    1. Check the ceph health details first.
    2. Check if any OSDs are down.
    3. Check the Pool size/min_size settings.
    4. Check for CRUSH failure zones.
    5. Determine if recovery/backfill is in progress.
    6. Decide whether human intervention is needed.
    7. Only consider repairing after all other steps have been attempted.

---

## Section XII: Daily Pool Inspection

### 12.1 Viewing the Pool List

    ceph osd pool ls

For more details:

    ceph osd pool ls detail

To view Pool IDs:

    ceph osd lspools

---

### 12.2 Checking Pool Capacity

    ceph df

Pay attention to:

- STORED
- OBJECTS
- USED
- %USED
- MAX AVAIL of each Pool

---

### 12.3 Viewing Pool Parameters

To view all parameters:

    ceph osd pool get <pool-name> all

Common parameters:

    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> min_size
    ceph osd pool get <pool-name> pg_num
    ceph osd pool get <pool-name> pgp_num
    ceph osd pool get <pool-name> crush_rule
    ceph osd pool get <pool-name> pg_autoscale_mode

---

### 12.4 Checking Pool Applications

    ceph osd pool application get <pool-name>

Common applications include:

- rbd
- cephfs
- rgw

If no application is enabled, you may see:

    POOL_APP_NOT_ENABLED

---

### 12.5 Viewing PG Autoscaler

    ceph osd pool autoscale-status

Focus on:

- PG_NUM
- NEW PG_NUM
- AUTOSCALE
- TARGET RATIO

Production advice:

    It is recommended to first use the "warn" observation level.
    Do not automatically adjust a large number of PGs without understanding the potential impacts.
    Adjusting PGs may trigger data remap and backfill processes.

---

## Section XIII: Capacity Level Management

### 13.1 Why Ceph Cannot Be Fully Utilized

Ceph requires reserved space for:

- Replica recovery
- Backfilling
- Rebalancing
- Data migration after OSD failures
- New data writes
- Metadata growth

If the capacity is too high:

    There may not be enough space to restore replicas after an OSD failure.
    PGs might remain degraded for an extended period.
    Data writes may be rejected.
    The cluster could reach nearfull, backfillfull, or full states.

---

### 13.2 Checking Capacity

Cluster capacity:

    ceph df

OSD capacity:

    ceph osd df

Pool capacity:

    ceph df

---

### 13.3 Capacity Level Recommendations

In a test environment, you can allow for more flexibility.

For production environments, the following guidelines apply:

| Utilization | Recommendation |
|---|---|
| Below 70% | Normal observation |
| 70%-80% | Begin planning capacity expansion |
| 80%-85% | Define clear expansion plans |
| Above 85% | High risk; immediate action required |
| Nearfull | Trigger alarm processing |
| Backfillfull | Affects recovery capabilities |
| Full | Serious risk; may result in write rejections |

Note:

    The specific thresholds should be adjusted based on business needs, hardware resources, cluster size, and corporate guidelines.

---

### 13.4 Checking Full-related Configurations

    ceph osd dump | grep full

You can also check the configuration directly:

    ceph config dump | grep full

Common relevant items include:

- mon_osd_nearfull_ratio
- mon_osd_backfillfull_ratio
- mon_osd_full_ratio

Production reminder:

    It is not recommended to simply increase the full threshold just to eliminate alarms.
    The proper approach is usually to expand capacity, clean up unnecessary data, migrate files, and optimize storage usage.

---

### 13.5 Approaches for Handling Capacity Risks

When capacity approaches high levels:

    1. Check the ceph df output.
    2. Identify the Pool that is growing the fastest    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ceph health detail

Client-side:

    mount | grep ceph
    df -hT
    dmesg | tail -100
    lsof +f -- <mount-path>

---

## Section Seventeen: Regular Inspection of RGW

If RGW is in use, it is necessary to monitor the status of the object storage interface and buckets.

### 17.1 Checking the RGW Service

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw

---

### 17.2 Testing RGW Ports

    curl -I http://10.0.0.31:7480
    curl -I http://10.0.0.32:7480
    curl -I http://10.0.0.33:7480

Note:

    Returning HTTP responses such as 200 or 403 usually indicates that the ports are accessible. If connections fail, it is necessary to check the service, ports, firewall settings, and network connectivity.

---

### 17.3 Viewing RGW Users

    radosgw-admin user list

To view a specific user:

    radosgw-admin user info --uid=<uid>

---

### 17.4 Checking Buckets

    radosgw-admin bucket list

To view bucket statistics:

    radosgw-admin bucket stats --bucket=<bucket-name>

To view a user's buckets:

    radosgw-admin bucket list --uid=<uid>

---

### 17.5 Checking RGW Pools

    ceph osd pool ls | grep rgw
    ceph df

Production Note:

    Do not delete RGW-related pools arbitrarily. Deletion of objects should be handled through the S3 API or radosgw-admin management processes.

---

### 17.6 Key Points for RGW Inspection

- Whether the RGW instance is running.
- Whether the frontend Nginx/LB is healthy.
- Whether the HTTPS certificate has expired.
- Whether there are any 4xx/5xx errors.
- Whether the number of buckets or objects is increasing abnormally.
- Whether user quotas are approaching their limits.
- Whether RGW pools are close to full capacity.
- Whether OSDs/PGs are healthy.

---

## Section Eighteen: Regular Inspection of RBD

### 18.1 Checking RBD Pools

    rbd ls -p <pool-name>

Example:

    rbd ls -p k8s-rbd

---

### 18.2 Viewing RBD Image Information

    rbd info <pool-name>/<image-name>

---

### 18.3 Viewing RBD Usage

    rbd du <pool-name>/<image-name>

---

### 18.4 Checking RBD Status

    rbd status <pool-name>/<image-name>

If the image is mounted by a client, you may see related watchers.

---

### 18.5 Viewing Snapshots

    rbd snap ls <pool-name>/<image-name>

---

### 18.6 Common Risks with RBD

| Risk | Description |
|---|---|
| Unused Images | Occupy capacity indefinitely |
| Expired Snapshots | Consume space and hinder cleanup |
| Clone Dependencies Not Flattened | Prevent deletion of parent snapshots |
| Lock Issues | Abnormal client exits may leave locks |
| Incorrect PVC Deletion Policies | May result in accidental data loss |

---

## Section Nineteen: Regular Inspection of Kubernetes CSI

If Ceph is integrated with Kubernetes, it is necessary to monitor CSI-related components.

### 19.1 Checking CSI Pods

    kubectl get pods -n ceph-csi -o wide

Pay attention to:

- RBD CSI Provisioner
- RBD CSI Node Plugin
- CephFS CSI Provisioner
- CephFS CSI Node Plugin

---

### 19.2 Checking StorageClasses

    kubectl get storageclass

For detailed information:

    kubectl describe storageclass <storage-class-name>

---

### 19.3 Checking PVCs/PVs

    kubectl get pvc -A
    kubectl get pv

To check for abnormal PVCs:

    kubectl describe pvc -n <namespace> <pvc-name>

---

### 19.4 Checking VolumeAttachments

    kubectl get volumeattachment

If a Pod fails to mount, you can check:

    kubectl describe volumeattachment <name>

---

### 19.5 Checking Pod Mount Events

    kubectl describe pod -n <namespace> <pod-name>

Pay special attention to events such as:

- FailedMount
- FailedAttachVolume
- MountVolume.MountDevice failed
- timed out waiting for condition
- permission### 22.2 Common Flags

| Flag | Meaning |
|---|---|
| noout | Do not automatically mark an OSD as out after it goes down |
| nobackfill | Prevent backfilling of data |
| norebalance | Prohibit rebalancing of data |
| norecover | Stop recovery processes |
| noscrub | Disable scrubbing operations |
| nodeep-scrub | Prevent deep scrubbing of nodes |
| pause | Temporarily suspend client I/O operations (high-risk) |

---

### 22.3 Setting noout

Before maintenance:

    `ceph osd set noout`

To check the setting:

    `ceph osd dump | grep flags`

After maintenance, this setting must be undone:

    `ceph osd unset noout`

---

### 22.4 High-Risk Warnings

Do not permanently set the following flags:

- noout
- nobackfill
- norebalance
- norecover
- noscrub
- nodeep-scrub

Permanently setting these flags can mask issues, leading to:

- Failed OSDs not being migrated
- Data not being recovered for an extended period
- Scrubbing operations not being performed
- Accumulation of cluster risks

Best practices in production environments:

- Record the settings before applying them.
- Trigger alerts after making changes.
- Unset these flags immediately after maintenance is complete.
- Regularly check these flags during daily inspections.

---

### 22.5 Checking for Abnormal Flags

Run the following command to check for unusual flags:

    `ceph osd dump | grep flags`

If you find the following flags:

- noout, nobackfill, norebalance

Verify whether they are still in use due to ongoing maintenance tasks.

---

## Chapter 23: Common Fault Classification

### 23.1 P3: Minor Alerts

Examples:

- The pool application is not enabled.
- Some recommended parameters have alerts.
- There are reminders regarding the dashboard certificate.
- A single non-critical module is indicating issues.

Actions to take:

- Record the issue.
- Schedule it for processing.
- Ensure it does not affect core business operations.

---

### 23.2 P2: Moderate Faults

Examples:

- A single OSD is down, but there are sufficient backups.
- The PG is degraded and is in the process of recovery.
- The MGR standby is missing.
- The MDS standby is missing.
- An RGW instance is abnormal, but load balancing is available.

Actions to take:

- Address the issue immediately.
- Continuously monitor the recovery progress.
- Notify relevant personnel.

---

### 23.3 P1: Serious Faults

Examples:

- Multiple OSDs are down.
- The PG is inactive.
- The PG is incomplete.
- An OSD is full.
- The MON quorum is abnormal.
- The active MDS for CephFS is missing.
- A large number of PVCs are failing to mount.
- All RGW instances are unavailable.

Actions to take:

- Address the issue immediately.
- Stop any non-essential changes.
- Notify business stakeholders if necessary.
- Proceed with the fault response process.

---

## Chapter 24: Sample Daily Inspection Scripts

### 24.1 Simple Inspection Script

Recommended location:

`05-Storage/01-Ceph/scripts/ceph-daily-check.sh`

Script content:

    #!/usr/bin/env bash

    set -euo pipefail

    echo "===== Ceph Daily Check ====="
    echo

    echo "===== 1. ceph -s ====="
    ceph -s
    echo

    echo "===== 2. Ceph health detail ====="
    ceph health detail || true
    echo

    echo "===== 3. ceph osd stat ====="
    ceph osd stat
    echo

    echo "===== 4. ceph osd tree ====="
    ceph osd tree
    echo

    echo "===== 5. ceph osd df ====="
    ceph osd df
    echo

    echo "===== 6. ceph pg stat ====="
    ceph pg stat
    echo

    echo "===== 7. ceph df ====="
    ceph df
    echo

    echo "===== 8. ceph orch ps ====="
    ceph orch ps
    echo

    echo "===== 9. ceph osd dump flags ====="
    ceph osd dump | grep flags || true
    echo

    echo "===== 10. ceph crash ls-new ====="
    ceph crash ls-new || true
    echo

    echo "===== Check Finished ====="

---

### 24.2 How to Use It

Add execution permissions:

`chmod +x ceph-daily-check.sh`

Run the script:

`./ceph-daily-check.sh### 27.3 Do Not Confuse Recovery with Repair

Recovery / Backfill is the process by which Ceph automatically restores data copies.

However, automatic recovery does not mean that the root cause has been resolved.

For example:

- If an OSD is restored after going down, it only means that the copy has been replenished.
- But the reasons behind the disk failure, node outage, or network instability still need to be analyzed further.

---

### 27.4 Capacity Governance Should Be Conducted in Advance

Ceph capacity management should not wait until 90% of the capacity is used before starting.

Advanced SREs should:

- Begin paying attention to trends when 70% of the capacity is used.
- Develop expansion plans when 80% of the capacity is utilized.
- Assign high priority to handling issues when 85% of the capacity is reached.
- Address situations where the capacity is nearfull immediately.
- Treat cases where the capacity is full as serious failures.

---

### 27.5 Changes Are More Important Than Commands

Ceph operations and maintenance are not about memorizing specific commands, but about understanding:

- Under what circumstances a command can be executed.
- Under what circumstances it cannot be executed.
- What to check before executing a command.
- What to verify after executing it.
- How to perform rollback in case of failure.
- How to assess the impact of any changes.

---

## Twenty-Eight. Interview Answer Framework

If you are asked in an interview:

- How do you perform routine inspections on a Ceph cluster?

You can answer like this:

- I would start by checking the overall health status using `ceph -s` to confirm whether the values for health, MON, MGR, OSD, PG, and usage are normal. If they are not at HEALTH_OK, I would further examine the `ceph health detail` to identify if the issue lies with OSDs, PGs, MONs, MDS, capacity, or slow requests.
- Next, I would look at the `ceph osd tree` and `ceph osd df` to check whether any OSDs are up/in, whether there are any down OSDs, whether the utilization of OSDs is balanced, and whether they are approaching nearfull or full capacity. I would also check the `ceph pg stat` to ensure that PGs are active+clean and not degraded, undersized, stale, or inconsistent.
- For pools, I would examine `ceph df`, `ceph osd pool ls detail`, and `pg autoscale-status` to verify the correct amount of capacity growth, number of copies, number of PGs, and whether the applications are functioning correctly.
- If CephFS is being used, I would check `ceph fs status` and `ceph mds stat` to ensure that the MDS active/standby states are normal. For RGW, I would monitor the rgw daemon, S3 access, Bucket growth, and quotas. If integrating with Kubernetes, I would check for ceph-csi Pods, PVC/PVs, VolumeAttachment, and Pod FailedMount events.
- On the node side, I would inspect the disks, network, time synchronization, system logs, and OSD delays. In a production environment, I would also pay attention to capacity trends, alerts, Crash records, whether maintenance flags have been set for an extended period without being canceled, and the presence of any unnecessary snapshots, orphaned RBD Images, or abnormal Buckets.
- I would never directly execute repair commands such as `osd purge`, `pg repair`, or `pool rm` without first understanding the cause of the issue. These are high-risk operations that require identifying the impact scope, preparing backups, and having a rollback plan in place.

---

## Twenty-Nine. Summary of This Article

This article mainly outlines the routine operations and inspection methods for Ceph:

1. The primary tools for daily Ceph operations and maintenance are `ceph -s` and `ceph health detail`.
2. For MONs, focus on quorum, time synchronization, and service status.
3. For MGRs, monitor active/standby states, the Dashboard, Prometheus metrics, and orchestrator activities.
4. For OSDs, check up/in status, capacity, delays, disk errors, and service status.
5. For PGs, ensure they are active+clean and not degraded, undersized, stale, or inconsistent.
6. For pools, monitor size, min_size, number of PGs, application usage, and capacity growth.
7. Capacity governance should be proactive, not delayed until full capacity is reached.
8. Recovery / Backfill is a normal recovery mechanism but can consume business resources.
9. Scrub / Deep Scrub are important for consistency checks and should not be neglected for long periods.
10. With CephFS, pay special attention to the MDS and metadata pool.
11. For RGW, monitor object access, Buckets, users