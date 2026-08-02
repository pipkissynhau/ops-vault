# Ceph Daily Operations: Health Checks, Capacity Management, Service Patrols, and Risk Governance

Suggested path: 05-Storage/01-Ceph/14-Ceph Daily Operations: Health Checks, Capacity Management, Service Patrols, and Risk Governance.md

Tags: #Ceph #DailyServices #HealthScreening #CapacityManagement #OSD #MON #MGR #MDS #RGW #RBD #CephFS #Inspection #SRE #AdvancedSre

---

## I. Document Description

This is the fourteenth article of the Ceph advanced SRE storage module, focusing on daily operation methods for Ceph clusters.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH failure domain
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI

From this article, we enter the Ceph operations governance phase.

After Ceph cluster deployment, the real challenge is not "can it run," but:

    How to check daily?
    How to determine if the cluster is healthy?
    How to detect capacity risks?
    How to check if OSD is abnormal?
    How to check if PG is abnormal?
    How to check if MON / MGR / MDS / RGW is normal?
    Which alarms can be observed?
    Which alarms must be handled immediately?
    Which commands are high-risk?
    How to form a production patrol habit?

The goal of this article is to establish an executable Ceph daily patrol and operations methodology.

---

## II. Experimental and Production Environments

### 2.1 Ceph Cluster Nodes

This article continues using the Ceph module experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation, Optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client Testing, Optional |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary systems:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 2.2 Daily Operation Objects

Ceph daily operations mainly focus on:

| Object | Focus Points |
|---|---|
| Cluster | Overall health, version, capacity, alarms |
| MON | Quorum, count, clock, status |
| MGR | Active/standby, Dashboard, Prometheus, orchestrator |
| OSD | Up/in, capacity, delay, disk errors, recovery |
| PG | Active+clean, degraded, undersized, stale, inconsistent |
| Pool | Replication count, PG count, capacity, application |
| CRUSH | Failure domain, OSD distribution, weight |
| RBD | Image, snapshot, lock, client mapping |
| CephFS | MDS, metadata pool, data pool, client sessions |
| RGW | Gateway instance, S3 users, Bucket, object, quota |
| CSI | Kubernetes PVC, PV, CSI Pod, mount events |
| Host | CPU, memory, disk, network, time synchronization, system logs |

---

## III. Daily Operation Goals

The core goals of Ceph daily operations:

1. Timely detect cluster health anomalies.
2. Timely detect OSD down, disk failure, and node anomalies.
3. Timely detect PG degraded, undersized, stale, inconsistent.
4. Timely detect capacity water level risks.
5. Timely detect pool usage anomalies.
6. Timely detect MDS / RGW / CSI entry anomalies.
7. Timely detect client mount or access anomalies.
8. Avoid data loss due to misoperations.
9. Establish patrol, alarm, change, recovery, and post-mortem closed-loop.
10. Elevate Ceph from "can run" to "can be operated, governed, and recovered."

---

## IV. Patrol Frequency Recommendations

### 4.1 Daily Patrol

Daily recommended checks:

- ceph -s
- ceph health detail
- ceph osd tree
- ceph osd df
- ceph pg stat
- ceph orch ps
- ceph df
- ceph fs status, if using CephFS
- RGW service status, if using RGW
- Kubernetes PVC / CSI status, if integrated with K8s

---

### 4.2 Weekly Patrol

Weekly recommended checks:

- OSD usage rate trend
- Pool capacity growth trend
- Whether PG count is reasonable
- OSD delay trend
- Slow requests slow ops
- Dashboard / Prometheus alarms
- MDS performance and client connection count
- RGW Bucket / Object growth
- CSI error events
- System disk and log directory capacity

---

### 4.3 Monthly Patrol

Monthly recommended checks:

- Ceph version and patch strategy
- Backup and recovery drill records
- Failure drill records
- Expired snapshots
- Orphaned RBD Images
- Unused Pools
- Unused users and keyring
- RGW expired users and Buckets
- Capacity expansion plan
- Alarm rule validity
- Operation documentation and change records

---

## V. Patrol Task Checklist

| Inspection Item | Command | Frequency | Importance |
|---|---|---|---|
| Cluster Overall Status | ceph -s | Daily | High |
| Health Details | ceph health detail | Daily | High |
| OSD Status | ceph osd tree | Daily | High |
| OSD Capacity | ceph osd df | Daily | High |
| PG Status | ceph pg stat | Daily | High |
| Pool Capacity | ceph df | Daily | High |
| Service Status | ceph orch ps | Daily | High |
| MON Quorum | ceph quorum_status | Weekly | Medium-High |
| MGR Status | ceph mgr stat | Daily | Medium-High |
| MDS Status | ceph fs status | Daily, if using CephFS | High |
| RGW Status | ceph orch ps --daemon_type rgw | Daily, if using RGW | High |
| RBD Usage | rbd ls / rbd info | As needed | Medium |
| CSI Status | kubectl get pods -n ceph-csi | Daily, if integrated with K8s | High |
| Node System Resources | top / iostat / df | Weekly | Medium-High |
| Time Synchronization | timedatectl / chronyc | Weekly | Medium-High |
| Log Errors | journalctl / ceph logs | Weekly | Medium-High |

---

## SixI don't know.Cluster Overall Health Check

### 6.1 View Cluster Status

Core command:

    ceph -s

Key focus areas:

- health
- services
- data
- usage
- io
- progress

Target for normal status:

    health: HEALTH_OK
    mon quorum normal
    mgr active normal
    osd up/in normal
    pgs active+clean
    no large recovery/backfill
    no nearfull/full

---

### 6.2 ceph -s Normal Example

Example:

    cluster:
      id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      health: HEALTH_OK

    services:
      mon: 3 daemons, quorum ceph-node01,ceph-node02,ceph-node03
      mgr: ceph-node01(active), standbys: ceph-node02
      osd: 6 osds: 6 up, 6 in

    data:
      pools:   5 pools, 160 pgs
      objects: 1.2k objects
      usage:   120 GiB used, 480 GiB / 600 GiB avail
      pgs:     160 active+clean

---

### 6.3 HEALTH_OK

Indicates no obvious health issues detected in Ceph.

But note:

    HEALTH_OK does not mean there are no risks.
    Still need to monitor capacity trends, OSD latency, business errors, and monitoring metrics.

---

### 6.4 HEALTH_WARN

Indicates alerts exist, but business operations may not be interrupted.

Common HEALTH_WARN:

- OSD down
- PG degraded
- Pool application not enabled
- clock skew
- nearfull
- too few PGs
- too many PGs
- slow ops
- MDS slow requests
- insecure global_id reclaim, depending on version and configuration

Handling principles:

    Must check ceph health detail.
    Do not ignore alerts based solely on HEALTH_WARN.
    Do not execute repair commands without understanding the alert.

---

### 6.5 HEALTH_ERR

Indicates serious anomalies that may affect data security or business availability.

Common HEALTH_ERR:

- full
- PG inactive
- PG incomplete
-Mass OSD failures
- MON quorum lost
- Data irrecoverable risk

Handling principles:

    Immediately stop non-essential changes.
    Collect status first.
    Determine if business is affected.
    Prioritize restoring cluster health.
    Do not blindly delete OSD, Pool, PG.
    Escalate to incident handling if necessary.

---

## SevenI don't know.Health Detail Check

### 7.1 View Health Details

    ceph health detail

This command is the first entry point for troubleshooting Ceph alerts.

If ceph -s shows HEALTH_WARN or HEALTH_ERR, must continue with:

    ceph health detail

---

### 7.2 Common Alert Categories

| Alert Type | Direction |
|---|---|
| OSD_DOWN | OSD or node failure |
| PG_DEGRADED | Insufficient replicas or in recovery |
| PG_UNDERSIZED | Replica count below size |
| PG_STALE | PG status not updated for a long time |
| PG_INCONSISTENT | Inconsistent replicas |
| MON_CLOCK_SKEW | Node time unsynchronized |
| OSD_NEARFULL | OSD near full |
| OSD_FULL | OSD full |
| POOL_APP_NOT_ENABLED | Pool not enabled for application |
| SLOW_OPS | Cluster has slow operations |
| MDS_SLOW_REQUEST | Slow CephFS metadata requests |
| RECENT_CRASH | Daemon recently crashed |

---

### 7.3 Handling Principles

Analyze alerts in this order after seeing them:

    1. What is the alert type?
    2. Does it affect data security?
    3. Does it affect business read/write?
    4. Is it automatically recovering?
    5. Does it require manual intervention?
    6. Are there high-risk operation risks?
    7. Does it require a change window?
    8. Does it require notifying business parties?

---

## EightI don't know.MON Daily Inspection

### 8.1 View MON Status

    ceph mon stat

Expected: /think

3 mons at {ceph-node01=...,ceph-node02=...,ceph-node03=...}, election epoch ..., quorum 0,1,2 ceph-node01,ceph-node02,ceph-node03

---

### 8.2 View Quorum Details

    ceph quorum_status --format json-pretty

Focus on:

- quorum_names
- monmap
- election_epoch
- outside_quorum
- extra_probe_peers
- sync_provider

Normal target:

    All expected MONs are in quorum.

---

### 8.3 View MON Services

    ceph orch ps --daemon_type mon

Expected:

    mon.ceph-node01 running
    mon.ceph-node02 running
    mon.ceph-node03 running

---

### 8.4 Common MON Issues

| Issue | Possible Cause |
|---|---|
| MON not in quorum | Network issues, time drift, service anomalies |
| clock skew | Time synchronization anomalies |
| MON disk full | /var/lib/ceph or system disk full |
| MON startup failure | Data directory corruption, configuration errors, container anomalies |
| quorum loss | Multiple MON failures or network partitioning |

---

### 8.5 MON Troubleshooting Commands

    ceph mon stat
    ceph quorum_status --format json-pretty
    ceph orch ps --daemon_type mon
    ceph health detail

Node-side:

    hostname
    timedatectl
    chronyc sources -v
    df -hT
    journalctl -u ceph-*.target --no-pager | tail -100

---

## NineI don't know.MGR Routine Inspection

### 9.1 View MGR Status

    ceph mgr stat

Normal target:

    1 active MGR.
    At least 1 standby MGR.

---

### 9.2 View MGR Services

    ceph orch ps --daemon_type mgr

---

### 9.3 View MGR Modules

    ceph mgr module ls

Common modules:

- dashboard
- prometheus
- orchestrator
- iostat
- restful, depending on version and requirements

---

### 9.4 View Dashboard / Prometheus Services

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
| Missing active MGR | Dashboard, orchestrator, partial management capabilities anomalies |
| Dashboard inaccessible | Operations entry point affected |
| Prometheus endpoint inaccessible | Monitoring collection anomalies |
| Missing standby | MGR failover capability insufficient |

---

## TenI don't know.OSD Routine Inspection

### 10.1 View OSD Overall Status

    ceph osd stat

Expected:

    6 osds: 6 up, 6 in

Focus:

    Whether the up count equals the total OSD count.
    Whether the in count equals the total OSD count.

---

### 10.2 View OSD Tree

    ceph osd tree

Focus on:

- Whether OSD is up
- Whether OSD is in
- Which hosts OSDs are distributed on
- Whether there are down + in
- Whether there are up + out
- Whether there are reweight anomalies
- Whether OSD class is correct

Normal target:

    All OSDs up/in.
    OSD distribution conforms to fault domain planning.
    No abnormal down OSDs.

---

### 10.3 View OSD Capacity

    ceph osd df

Key fields:

| Field | Description |
|---|---|
| SIZE | OSD total capacity |
| RAW USE | Raw used capacity |
| DATA | Data usage |
| AVAIL | Available capacity |
| %USE | Usage rate |
| VAR | Relative average usage deviation |
| PGS | PG count |

---

### 10.4 OSD Capacity Focus Points

Key checks:

    Whether any OSD has significantly higher usage than others.
    Whether any OSD is near full.
    Whether any OSD is full.
    Whether any OSD has significantly abnormal PGS count.
    Whether there's long-term imbalance after expansion.

---

### 10.5 View OSD Latency

    ceph osd perf

Focus on:

- commit_latency
- apply_latency

If an OSD's latency is significantly higher than others, investigate:

- Disk failure
- Poor disk performance
- Node IO saturation
- Network anomalies
- OSD in recovery/backfill
- Insufficient system resources

---

### 10.6 View OSD Metadata

    ceph osd metadata osd.0

Can be used to confirm:

- OSD host
- OSD device
- Ceph version
- BlueStore information
- Operating system
- Kernel version

---

### 10.7 View OSD Services

    ceph orch ps --daemon_type osd

---

### 10.8 OSD Abnormality Priority Troubleshooting Path

When OSD anomalies occur, recommend checking in this order:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph orch ps --daemon_type osd
    ceph osd metadata osd.X
    Login to corresponding node
    lsblk
    df -hT
    dmesg | grep -i error
    journalctl -k | grep -i error
    iostat -x 1

---

## ElevenI don't know.PG Routine Inspection

### 11.1 View PG Summary

    ceph pg stat /think

Normal Target:

    active+clean

---

### 11.2 Common PG States

| State | Meaning |
|---|---|
| active+clean | Normal |
| creating | Creating |
| peering | Negotiating state |
| degraded | Insufficient replicas |
| undersized | Number of replicas below size |
| recovering | Recovering |
| backfilling | Backfilling |
| remapped | Remapped |
| stale | State has not been updated for a long time |
| inconsistent | Replicas are inconsistent |
| inactive | Not available |
| incomplete | Missing necessary replicas or history, severe |

---

### 11.3 Checking Abnormal PGs

Check stuck PGs:

    ceph pg dump_stuck

Check inactive:

    ceph pg dump_stuck inactive

Check unclean:

    ceph pg dump_stuck unclean

Check stale:

    ceph pg dump_stuck stale

---

### 11.4 Checking PG Mapping

    ceph pg map <pg-id>

Example:

    ceph pg map 1.0

Output:

    pg 1.0 maps to up [0,1,2] acting [0,1,2]

---

### 11.5 Checking PG Details

    ceph pg <pg-id> query

This command outputs a long result, suitable for in-depth troubleshooting.

Focus on:

- state
- up
- acting
- primary
- recovery_state
- blocked_by
- peer information

---

### 11.6 Principles for Handling PG Abnormalities

Do not directly repair PGs when you see abnormalities.

Handling order:

    1. First check ceph health detail.
    2. Then check if any OSDs are down.
    3. Then check Pool size/min_size.
    4. Then check CRUSH failure domain.
    5. Then check if recovery/backfill is ongoing.
    6. Then determine if manual intervention is needed.
    7. Finally consider repair.

---

## 12. Pool Routine Inspection

### 12.1 Checking Pool List

    ceph osd pool ls

Check details:

    ceph osd pool ls detail

Check Pool ID:

    ceph osd lspools

---

### 12.2 Checking Pool Capacity

    ceph df

Focus on:

- Each Pool's STORED
- OBJECTS
- USED
- %USED
- MAX AVAIL

---

### 12.3 Checking Pool Parameters

Check all parameters:

    ceph osd pool get <pool-name> all

Common parameters:

    ceph osd pool get <pool-name> size
    ceph osd pool get <pool-name> min_size
    ceph osd pool get <pool-name> pg_num
    ceph osd pool get <pool-name> pgp_num
    ceph osd pool get <pool-name> crush_rule
    ceph osd pool get <pool-name> pg_autoscale_mode

---

### 12.4 Checking Pool Application

    ceph osd pool application get <pool-name>

Common applications:

- rbd
- cephfs
- rgw

If no application is enabled, you may see:

    POOL_APP_NOT_ENABLED

---

### 12.5 Checking PG Autoscaler

    ceph osd pool autoscale-status

Focus on:

- PG_NUM
- NEW PG_NUM
- AUTOSCALE
- TARGET RATIO

Production recommendations:

    First use warn to observe suggestions.
    Do not automatically adjust large numbers of PGs without understanding the impact.
    PG adjustments may trigger data remap and backfill.

---

## 13. Capacity Watermark Management

### 13.1 Why Ceph Cannot Be Fully Utilized

Ceph needs to reserve space for:

- Replica recovery
- Backfill
- Rebalance
- Data migration after OSD failure
- New write data
- Metadata growth

If capacity is too high:

    OSD failure may leave no space for replica recovery.
    PGs may remain degraded for a long time.
    Writes may be rejected.
    The cluster may enter nearfull / backfillfull / full.

---

### 13.2 Checking Capacity

Cluster capacity:

    ceph df

OSD capacity:

    ceph osd df

Pool capacity:

    ceph df

---

### 13.3 Capacity Watermark Recommendations

Experimental environments can be more lenient.

Production recommendations:

| Usage Rate | Recommendation |
|---|---|
| Below 70% | Normal observation |
| 70%-80% | Start planning expansion |
| 80%-85% | Clear expansion plan |
| Above 85% | High risk, needs prompt handling |
| nearfull | Enter alert handling |
| backfillfull | Affects recovery capability |
| full | Severe risk, may reject writes |

Note:

    Specific thresholds should be adjusted based on business needs, hardware, cluster scale, and enterprise standards.

---

### 13.4 Checking Full-Related Configurations

    ceph osd dump | grep full

You can also check the configuration:

    ceph config dump | grep full

Common related items:

- mon_osd_nearfull_ratio
- mon_osd_backfillfull_ratio
- mon_osd_full_ratio

Production reminder:

    Do not simply raise full thresholds to eliminate alerts.
    The correct approach is usually expansion, cleanup, migration, and data optimization.

---

### 13.5 Capacity Risk Handling Approach

When capacity approaches high watermark:

1. Check `ceph df`.
2. Identify the Pool with the fastest growth.
3. Check `ceph osd df`.
4. Determine if a single OSD is abnormal.
5. Determine if business data is growing.
6. Clean up unused snapshots, unused objects, and unused test data.
7. Plan for adding new OSDs or new nodes.
8. Do not directly delete unknown Pools.
9. Do not directly deleteBottom objects.

---

## Fourteen, Recovery / Backfill Inspection

### 14.1 Check if recovery is in progress

    ceph -s

If recovery is in progress, you may see:

    recovering
    backfilling
    remapped
    misplaced
    degraded

---

### 14.2 Continuous monitoring

    ceph -w

Or:

    watch -n 5 'ceph -s'

---

### 14.3 Check OSD load

    ceph osd perf
    ceph osd df

Node side:

    iostat -x 1
    vmstat 1
    top

---

### 14.4 Recovery and business impact

Recovery / Backfill consumes:

- Network
- Disk IO
- CPU
- OSD background threads

In production, balance:

    Faster recovery may result in greater impact on business IO.
    Slower recovery means data remains in degraded state longer.

---

### 14.5 Common recovery parameters

Check:

    ceph config get osd osd_max_backfills
    ceph config get osd osd_recovery_max_active
    ceph config get osd osd_recovery_op_priority

Example settings:

    ceph config set osd osd_max_backfills 2
    ceph config set osd osd_recovery_max_active 3

High-risk warning:

    Not recommended to adjust arbitrarily in the initial learning phase.
    Must record original values before production adjustments.
    After adjustment, observe business IO, recovery speed, and OSD latency.
    Parameter semantics may vary slightly across Ceph versions; refer to official documentation for the corresponding version.

---

## Fifteen, Scrub and Deep Scrub Inspection

### 15.1 What is Scrub

Scrub is a background mechanism in Ceph for checking object consistency.

It can be understood as:

    Regularly checking consistency between replicas.

Deep Scrub performs more in-depth data validation.

---

### 15.2 Check Scrub-related alerts

    ceph health detail

Common alerts:

- PG_NOT_SCRUBBED
- PG_NOT_DEEP_SCRUBBED
- PG_DAMAGED
- PG_INCONSISTENT

---

### 15.3 Check PG inconsistent

    ceph health detail

If inconsistent appears, further check:

    ceph pg <pg-id> query

If necessary:

    ceph pg repair <pg-id>

High-risk warning:

    Repair must be done after understanding the issue.
    Do not repair blindly.
    In production environments, it is recommended to preserve the scene, analyze logs, and confirm the impact scope first.

---

### 15.4 Production considerations for Scrub

Scrub / Deep Scrub consumes resources.

In production environments, should:

- Avoid business peak hours
- Monitor OSD latency
- Monitor slow ops
- Monitor inconsistency alerts
- Do not disable scrub long-term
- Do not ignore deep scrub expiration alerts

---

## Sixteen, CephFS Daily Inspection

If using CephFS, additional checks on MDS and file system status are needed.

### 16.1 Check CephFS status

    ceph fs status

Focus on:

- Active MDS
- Standby MDS
- Client count
- Request status
- Metadata pool
- Data pool

---

### 16.2 Check MDS status

    ceph mds stat

Target goals:

    Active MDS exists.
    Standby MDS exists (recommended for production).

---

### 16.3 Check MDS services

    ceph orch ps --daemon_type mds

---

### 16.4 Check CephFS file system

    ceph fs ls
    ceph fs get cephfs

---

### 16.5 Common risks for CephFS

| Risk | Impact |
|---|---|
| Missing active MDS | CephFS unavailable |
| Missing standby MDS | Insufficient failover capability |
| Metadata pool abnormal | Metadata risk for file system |
| Large number of small files | Increased MDS pressure |
| Large directory | Slow directory traversal and metadata operations |
| Client abnormal | May affect locks and sessions |

---

### 16.6 CephFS troubleshooting entry points

    ceph -s
    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ceph health detail

Client side:

    mount | grep ceph
    df -hT
    dmesg | tail -100
    lsof +f -- <mount-path>

---

## Seventeen, RGW Daily Inspection

If using RGW, pay attention to object storage entry points and Bucket status.

### 17.1 Check RGW services

    ceph orch ls --service_type rgw
    ceph orch ps --daemon_type rgw

---

### 17.2 Test RGW ports

    curl -I http://10.0.0.31:7480
    curl -I http://10.0.0.32:7480
    curl -I http://10.0.0.33:7480

Note:

    Returning HTTP responses like 200, 403 usually indicates the port is reachable.
    Connection failure requires troubleshooting services, ports, firewalls, and network.

---

### 17.3 Check RGW users

    radosgw-admin user list

Check specific user:

    radosgw-admin user info --uid=<uid>

---

### 17.4 Check Buckets

    radosgw-admin bucket list

Check Bucket statistics: /think

radosgw-admin bucket stats --bucket=<bucket-name>

View User Bucket:

    radosgw-admin bucket list --uid=<uid>

---

### 17.5 View RGW Pool

    ceph osd pool ls | grep rgw
    ceph df

Production Note:

    Do not delete RGW-related pools arbitrarily.
    Delete objects should be handled through S3 API or radosgw-admin management process.

---

### 17.6 RGW Inspection Focus

- Is RGW instance Running
- Is frontend Nginx / LB healthy
- Is HTTPS certificate expired
- Are 4xx / 5xx errors abnormal
- Is bucket count abnormally increasing
- Is object count abnormally increasing
- Is user quota approaching the limit
- Is RGW Pool approaching full
- Are OSD / PG healthy

---

## EighteenI don't know.RBD Daily Inspection

### 18.1 View RBD Pool

    rbd ls -p <pool-name>

Example:

    rbd ls -p k8s-rbd

---

### 18.2 View RBD Image Information

    rbd info <pool-name>/<image-name>

---

### 18.3 View RBD Usage

    rbd du <pool-name>/<image-name>

---

### 18.4 View RBD Status

    rbd status <pool-name>/<image-name>

If mounted by client, you may see watcher.

---

### 18.5 View Snapshots

    rbd snap ls <pool-name>/<image-name>

---

### 18.6 Common RBD Risks

| Risk | Description |
|---|---|
| Unused Image | Long-term capacity occupation |
| Expired Snapshot | Occupies capacity and affects cleanup |
| Clone dependency not flatten | Parent snapshot cannot be deleted |
| Lock anomaly | Client abnormal exit may leave residual |
| PVC deletion policy error | May delete or leave data erroneously |

---

## NineteenI don't know.Kubernetes CSI Daily Inspection

If Ceph is integrated with Kubernetes, CSI inspection is required.

### 19.1 View CSI Pod

    kubectl get pods -n ceph-csi -o wide

Focus on:

- RBD CSI provisioner
- RBD CSI node plugin
- CephFS CSI provisioner
- CephFS CSI node plugin

---

### 19.2 View StorageClass

    kubectl get storageclass

View details:

    kubectl describe storageclass <storageclass-name>

---

### 19.3 View PVC / PV

    kubectl get pvc -A
    kubectl get pv

View abnormal PVC:

    kubectl describe pvc -n <namespace> <pvc-name>

---

### 19.4 View VolumeAttachment

    kubectl get volumeattachment

If Pod mounting fails, you can view:

    kubectl describe volumeattachment <name>

---

### 19.5 View Pod Mount Events

    kubectl describe pod -n <namespace> <pod-name>

Focus on Events:

- FailedMount
- FailedAttachVolume
- MountVolume.MountDevice failed
- timed out waiting for condition
- permission denied
- failed to get secret

---

### 19.6 CSI Logs

View RBD CSI:

    kubectl logs -n ceph-csi <rbd-provisioner-pod> -c csi-rbdplugin
    kubectl logs -n ceph-csi <rbd-node-pod> -c csi-rbdplugin

View CephFS CSI:

    kubectl logs -n ceph-csi <cephfs-provisioner-pod> -c csi-cephfsplugin
    kubectl logs -n ceph-csi <cephfs-node-pod> -c csi-cephfsplugin

---

## TwentyI don't know.Node System Inspection

### 20.1 CPU and Load

    top
    uptime
    mpstat 1

If mpstat is not available:

    apt install -y sysstat

---

### 20.2 Memory

    free -h
    vmstat 1

---

### 20.3 Disk

    lsblk
    df -hT
    iostat -x 1

Focus on:

- Is system disk approaching full
- Is OSD data disk abnormal
- Is await too high
- Is util long-term close to 100%
- Are there disk error logs

---

### 20.4 Network

    ip addr
    ip route
    ss -lntp
    iftop

If iftop is not available:

    apt install -y iftop

---

### 20.5 Time Synchronization

Ubuntu:

    systemctl status chrony
    chronyc sources -v
    timedatectl

Rocky Linux:

    systemctl status chronyd
    chronyc sources -v
    timedatectl

Ceph is sensitive to time, time drift may affect MON, authentication and log analysis.

---

### 20.6 System Logs

    journalctl -xe --no-pager | tail -100
    journalctl -k --no-pager | tail -100
    dmesg | tail -100

Disk errors:

    dmesg | grep -i error
    journalctl -k | grep -i error

---

## Twenty-oneI don't know.Ceph Log Inspection

### 21.1 Checking Services in a Cephadm Environment

    ceph orch ps

---

### 21.2 Viewing Daemon Logs

In a Cephadm environment, you can view logs via cephadm or systemd/journal.

Common methods:

    cephadm logs --name osd.0

Viewing MON:

    cephadm logs --name mon.ceph-node01

Viewing MGR:

    cephadm logs --name mgr.ceph-node01

Viewing MDS:

    cephadm logs --name mds.cephfs.ceph-node01.xxxxx

Viewing RGW:

    cephadm logs --name rgw.rgw-demo.ceph-node01.xxxxx

Note:

    The actual daemon name is determined by the output of ceph orch ps.

---

### 21.3 Viewing Recent Crashes

    ceph crash ls

Viewing new crashes:

    ceph crash ls-new

Viewing details:

    ceph crash info <crash-id>

Archiving:

    ceph crash archive <crash-id>

Archiving all:

    ceph crash archive-all

Note:

    Do not ignore crashes directly.
    Check the cause before archiving.
    Frequent crashes require investigation of daemon, image version, node resources, disks, and configuration.

---

## Twenty-Two, Maintenance Mode and Common Flags

### 22.1 Why Maintenance Flags Are Needed

When maintaining OSDs or nodes, Ceph may automatically trigger data migration.

Examples:

    Temporarily restarting a node.
    Temporarily stopping an OSD.
    Short-term network maintenance.

Without control, this may lead to unnecessary recovery/backfill.

Ceph provides some flags to control automatic behavior.

---

### 22.2 Common Flags

| Flag | Meaning |
|---|---|
| noout | Do not automatically mark OSDs as out after they are down |
| nobackfill | Disable backfill |
| norebalance | Disable rebalance |
| norecover | Disable recovery |
| noscrub | Disable scrub |
| nodeep-scrub | Disable deep scrub |
| pause | Pause client IO, extremely high risk |

---

### 22.3 Setting noout

Before maintenance:

    ceph osd set noout

Checking:

    ceph osd dump | grep flags

Must be removed after maintenance:

    ceph osd unset noout

---

### 22.4 High-Risk Warning

Do not set long-term:

    noout
    nobackfill
    norebalance
    norecover
    noscrub
    nodeep-scrub

Long-term use may mask issues, leading to:

- Faulty OSDs not being migrated
- Data not recovering long-term
- Scrub not executing long-term
- Accumulated cluster risks

Production principles:

    Record before setting.
    Alert after setting.
    Remove immediately after maintenance.
    Daily inspections must check flags.

---

### 22.5 Checking for Abnormal Flags

    ceph osd dump | grep flags

If you see:

    noout,nobackfill,norebalance

Confirm whether you're still in a maintenance window.

---

## Twenty-Three, Common Fault Levels

### 23.1 P3: Minor Alert

Examples:

- Pool application not enabled
- Some recommended parameter alerts
- Dashboard certificate reminder
- Single non-critical module prompt

Handling:

    Record.
    Schedule for later.
    Does not affect core business.

---

### 23.2 P2: Moderate Fault

Examples:

- Single OSD down, but replicas are sufficient
- PG degraded and recovering
- MGR standby missing
- MDS standby missing
- RGW single instance abnormal but load-balanced available

Handling:

    Address on the same day.
    Continuously monitor recovery.
    Notify relevant personnel.

---

### 23.3 P1: Severe Fault

Examples:

- Multiple OSDs down
- PG inactive
- PG incomplete
- OSD full
- MON quorum abnormal
- CephFS active MDS missing
- Large number of PVC FailedMount
- All RGW instances unavailable

Handling:

    Address immediately.
    Stop non-essential changes.
    Notify business if necessary.
    Enter fault response process.

---

## Twenty-Four, Daily Inspection Script Example

### 24.1 Simple Daily Inspection Script

Recommended path:

    05-Storage/01-Ceph/scripts/ceph-daily-check.sh

Script content:

    #!/usr/bin/env bash

    set -euo pipefail

    echo "===== Ceph Daily Check ====="
    echo

    echo "===== 1. ceph -s ====="
    ceph -s
    echo

    echo "===== 2. ceph health detail ====="
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

# 10. ceph crash ls-new

```bash
echo "===== 10. ceph crash ls-new ====="
ceph crash ls-new || true
echo

echo "===== Check Finished ====="
```

---

### 24.2 Usage

Add execute permissions:

```bash
chmod +x ceph-daily-check.sh
```

Execute:

```bash
./ceph-daily-check.sh
```

Save logs:

```bash
./ceph-daily-check.sh > ceph-daily-check-$(date +%F).log 2>&1
```

---

### 24.3 Script Usage Notes

This script only performs read-only checks and does not perform automatic repairs.

It is not recommended to run inspection scripts automatically in production:

- osd out
- osd rm
- pg repair
- pool rm
- unset/set flag
- Modify recovery parameters
- Delete snapshots
- Delete RBD Image

Automatic repairs must be handled with extreme caution.

---

## 25. Daily Inspection Template for Production Environments

### 25.1 Basic Information

| Item | Content |
|---|---|
| Inspection Date | YYYY-MM-DD |
| Inspector | Operations Staff |
| Cluster Name | ceph-prod |
| Ceph Version |  |
| Node Count |  |
| OSD Count |  |
| Pool Count |  |
| K8s Integration | Yes / No |
| CephFS Usage | Yes / No |
| RGW Usage | Yes / No |

---

### 25.2 Cluster Health

| Check Item | Result | Abnormal? |
|---|---|---|
| ceph -s |  |  |
| health detail |  |  |
| MON quorum |  |  |
| MGR active/standby |  |  |
| OSD up/in |  |  |
| PG Status |  |  |
| Recovery/Backfill |  |  |
| Crash |  |  |

---

### 25.3 Capacity

| Check Item | Result | Abnormal? |
|---|---|---|
| Cluster Total Capacity |  |  |
| Cluster Used Capacity |  |  |
| Cluster Usage Rate |  |  |
| Max OSD Usage Rate |  |  |
| Nearfull? |  |  |
| Fastest Growing Pool |  |  |

---

### 25.4 Services

| Service | Status | Abnormal? |
|---|---|---|
| MON |  |  |
| MGR |  |  |
| OSD |  |  |
| MDS |  |  |
| RGW |  |  |
| CSI |  |  |

---

### 25.5 Risks and Recommendations

| Risk | Impact | Recommendation |
|---|---|---|
|  |  |  |

---

## 26. Production Environment Notes

1. Must check ceph -s and ceph health detail daily.
2. Do not ignore HEALTH_WARN.
3. Do not execute repair commands without understanding the alert.
4. Do not perform expansion, deletion, or migration operations when the cluster is unhealthy.
5. Do not set noout, nobackfill, or norebalance long-term.
6. Do not delete pools arbitrarily.
7. Do not delete RBD Images arbitrarily.
8. Do not directly delete RGW Pools.
9. Do not execute wipefs or sgdisk on system disks.
10. After OSD down, confirm the cause before purging.
11. Do not repair PG inconsistent issues blindly.
12. Prioritize capacity management for nearfull/full, not just adjust thresholds.
13. CephFS must focus on MDS, not just OSD.
14. RGW must monitor entry layer, credentials, and Bucket growth.
15. Kubernetes CSI must monitor PVC, PV, VolumeAttachment, and CSI Pods.
16. All high-risk operations must have change windows, backups, rollback plans, and monitoring.
17. Inspections results must be documented, not just glanced at.
18. Capacity trends are more important than daily capacity.
19. Fault post-mortems are more important than temporary fixes.
20. Regular recovery drills must be conducted in production.

---

## 27. Advanced SRE Methodology

### 27.1 Ceph Operations Are Not Just HEALTH_OK

Advanced SREs cannot only check:

```bash
ceph -s shows HEALTH_OK
```

They must also check:

- Capacity trends
- OSD usage deviation
- OSD latency
- Pool growth
- PG count
- Recovery frequency
- Crash records
- MDS pressure
- RGW error rate
- CSI mount failure events

---

### 27.2 Ceph Troubleshooting Should Be Layered

Recommended layered approach:

```bash
Business Layer:
  Pod / VM / Application / S3 Client

Access Layer:
  CSI / RBD / CephFS / RGW / Nginx / LB

Ceph Service Layer:
  MON / MGR / MDS / RGW / OSD

Data Distribution Layer:
  Pool / PG / CRUSH / OSD Set

Node Resource Layer:
  CPU / Memory / Disk / Network / Time Sync
```

---

### 27.3 Do Not Confuse Recovery with Repair

Recovery/Backfill is Ceph's automatic process to restore data replicas.

But automatic recovery does not mean the root cause is resolved.

For example:

```bash
After OSD down recovery completes, it only indicates replicas are restored.
But the reason for disk failure, node outage, or network jitter still needs analysis.
```

---

### 27.4 Capacity Governance Should Be Proactive

Ceph capacity management should not wait until 90% full.

Advanced SREs should:

```bash
Start monitoring trends at 70%.
Plan expansion at 80%.
Enter high-priority handling at 85%.
Handle nearfull immediately.
Full capacity is a critical failure.
```

---

### 27.5 Changes Are More Important Than Commands

Ceph operations are not about remembering commands, but understanding:

```bash
When it's safe to execute.
When it's not safe to execute.
What to check before execution.
What to verify after execution.
How to rollback on failure.
How to assess impact scope.
```

---

## 28. Interview Answer Strategy

If asked in an interview:

```bash
How to inspect a Ceph cluster daily?
```

You can answer:

I will first check the overall health status using `ceph -s`, confirming whether `health`, `mon`, `mgr`, `osd`, `pg`, and `usage` are normal. If it's not `HEALTH_OK`, I'll proceed to check `ceph health detail` to identify the specific issue: whether it's related to `OSD`, `PG`, `MON`, `MDS`, capacity, or slow requests.

Then I'll check `ceph osd tree` and `ceph osd df` to confirm whether `OSD` is `up/in`, whether there are `down OSD`, whether `OSD` usage is balanced, and whether it's approaching `nearfull` or `full`. I'll also check `ceph pg stat` to confirm whether `PG` is `active+clean`, and whether there are `degraded`, `undersized`, `stale`, or `inconsistent` states.

For `Pool`, I'll check `ceph df`, `ceph osd pool ls detail`, and `pg autoscale-status`, focusing on `Pool` capacity growth, replica count, `PG` count, and application correctness.

If using CephFS, I'll check `ceph fs status` and `ceph mds stat` to confirm `MDS` `active/standby` status. If using RGW, I'll check `rgw daemon`, S3 access, `Bucket` growth, and quotas. If integrated with Kubernetes, I'll check `ceph-csi` Pod, PVC/PV, `VolumeAttachment`, and `Pod FailedMount` events.

On the node side, I'll check disks, network, time synchronization, system logs, and `OSD` latency. In production environments, I'll also monitor capacity trends, alerts, `Crash` records, whether maintenance flags are long-term set, and whether there are unused snapshots, orphaned RBD Images, or abnormal `Bucket`s.

I will not execute repair commands without understanding the alert first, such as `osd purge`, `pg repair`, or `pool rm`—these are high-risk operations that require confirming the impact scope, backup, and rollback plans before proceeding.

---

## 29. Summary of This Section

This article mainly organizes methods for daily Ceph operations and inspections:

1. The core entry for daily Ceph operations is `ceph -s` and `ceph health detail`.
2. MON should focus on `quorum`, time synchronization, and service status.
3. MGR should focus on `active/standby`, Dashboard, Prometheus, and orchestrator.
4. OSD should focus on `up/in`, capacity, latency, disk errors, and service status.
5. PG should focus on `active+clean`, `degraded`, `undersized`, `stale`, and `inconsistent`.
6. Pool should focus on `size`, `min_size`, `PG` count, application, and capacity growth.
7. Capacity governance should be proactive and not wait until `full` to handle.
8. `Recovery` / `Backfill` is a normal recovery mechanism but will consume business resources.
9. `Scrub` / `Deep Scrub` is a consistency check mechanism that should not be ignored long-term.
10. CephFS requires special attention to `MDS` and `metadata pool`.
11. RGW requires attention to object entry points, `Bucket`, users, quotas, and `RGW Pool`.
12. Kubernetes CSI requires attention to PVC, PV, `VolumeAttachment`, CSI Pod, and `kubelet` events.
13. Maintenance flags should not be set long-term; they must be canceled after maintenance.
14. Inspection scripts should focus on read-only checks and should not automatically execute high-risk repairs.
15. Advanced SRE operations on Ceph should simultaneously monitor health, capacity, performance, recovery, changes, and risk governance.

---

## 30. Reference Documents

Ceph Health Check:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Cluster Monitoring:

    https://docs.ceph.com/en/latest/rados/operations/monitoring/

Ceph OSD Management:

    https://docs.ceph.com/en/latest/rados/operations/add-or-rm-osds/

Ceph Placement Groups:

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph Pool Management:

    https://docs.ceph.com/en/latest/rados/operations/pools/

Ceph CRUSH Map:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

CephFS Management:

    https://docs.ceph.com/en/latest/cephfs/administration/

Ceph RGW Management:

    https://docs.ceph.com/en/latest/radosgw/admin/

Cephadm Operations:

    https://docs.ceph.com/en/latest/cephadm/operations/

Ceph Dashboard:

    https://docs.ceph.com/en/latest/mgr/dashboard/