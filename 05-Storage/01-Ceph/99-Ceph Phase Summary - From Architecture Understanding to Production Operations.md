# Ceph Phase Summary: From Architecture Understanding to Production Operations

Recommended path: 05-Storage/01-Ceph/99-Ceph Phase Summary: From Architecture Understanding to Production Operations.md

Tags: #Ceph #SummaryOfPhases #DistributedStorage #AdvancedSre #ProductionTransport #RBD #CephFS #RGW #CSI #SecurityAlert. #FaultCheck. #BackupRestore

---

## I. Document Overview

This document serves as a phase summary for the Ceph module, consolidating the entire learning journey.

Previously, we have completed a comprehensive learning path from foundational theory to integrated practical implementation, including:

- Ceph architecture and core components
- RADOS, Pool, PG, CRUSH data distribution model
- RBD block storage
- CephFS file storage
- RGW object storage
- cephadm multi-node deployment
- OSD management, expansion, removal, replacement
- Kubernetes RBD CSI / CephFS CSI integration
- Routine inspection and health checks
- Fault diagnosis
- Backup recovery and disaster recovery drills
- Performance optimization
- Security reinforcement
- Monitoring and alerts
- Integrated practical implementation and production validation

The focus of this document is not to expand new knowledge, but to summarize:

    What was learned in this phase
    What capabilities have been acquired
    Which aspects belong to production core capabilities
    Which areas require further in-depth study
    Why learning MinIO, Longhorn, and RustFS becomes easier after completing Ceph

---

## II. Phase Positioning

Ceph is one of the most comprehensive, complex, and capable of cultivating advanced SRE skills modules in distributed storage systems.

It covers three types of storage capabilities:

    Block storage: RBD
    File storage: CephFS
    Object storage: RGW

It also covers the most critical operational capabilities in production environments:

    High availability
    Data replication
    Fault domain
    Automatic recovery
    Capacity governance
    Performance bottleneck identification
    Permission governance
    Monitoring and alerts
    Backup and recovery
    Disaster recovery drills
    Kubernetes CSI integration

Therefore, the value of the Ceph phase is not just learning a storage software, but establishing a complete understanding of distributed storage operations.

---

## III. Ceph Phase Learning Mainlines

The entire Ceph module can be summarized into five mainlines.

### 3.1 First Mainline: Architecture Understanding

Core questions:

    What is Ceph?
    Why can Ceph provide block, file, and object storage simultaneously?
    What responsibilities do MON, MGR, OSD, MDS, and RGW have?
    Where does RADOS stand in Ceph?

Phase gains:

    Understanding that Ceph's underlying layer is a unified RADOS distributed object storage.
    Understanding that RBD, CephFS, and RGW are different interfaces built on top of RADOS.
    Understanding that OSD is the core of real data storage.
    Understanding that MON is responsible for cluster status and consistency.
    Understanding that MGR handles management, orchestration, monitoring, and Dashboard.
    Understanding that MDS serves exclusively for CephFS.
    Understanding that RGW is the object storage entry point, not the underlying storage itself.

---

### 3.2 Second Mainline: Data Distribution

Core questions:

    Where does data written to Ceph ultimately reside?
    What is the relationship between Pool, PG, CRUSH, and OSD?
    How does Ceph ensure replicas are not placed in the same fault domain?
    Why does PG degrade when an OSD goes down?

Phase gains:

    Understanding that Pool is a logical storage pool.
    Understanding that PG is an intermediate mapping layer between objects and OSDs.
    Understanding that CRUSH determines how data is distributed across different OSDs.
    Understanding that size controls replica count.
    Understanding that min_size controls the minimum number of replicas for write operations.
    Understanding the importance of host/rack fault domain design for high availability.
    Understanding the meanings of PG states like active+clean, degraded, undersized, inactive, and inconsistent.

---

### 3.3 Third Mainline: Storage Capabilities

Core questions:

    When is RBD, CephFS, and RGW suitable?
    Why are databases better suited for RBD?
    Why is CephFS better for multi-Pod shared directories?
    Why are images, attachments, and backup archives better suited for RGW?

Phase gains:

    RBD is similar to cloud disks, suitable for block storage and Kubernetes RWO.
    CephFS is similar to a shared file system, suitable for multi-client sharing and Kubernetes RWX.
    RGW is similar to S3/OSS, suitable for object storage scenarios.
    Different storage interfaces have distinct failure points, performance bottlenecks, and operational approaches.
    RBD, CephFS, and RGW cannot be treated as a single storage type.

---

### 3.4 Fourth Mainline: Production Operations

Core questions:

    How to monitor a Ceph cluster daily after deployment?
    How to troubleshoot OSD down?
    How to identify PG degraded status?
    What to do when storage capacity is nearly full?
    Where to check for CephFS slowness?
    Where to investigate RGW access failures?
    How to troubleshoot Kubernetes PVC Pending?

Phase gains:

    Mastering ceph -s and ceph health detail as troubleshooting entry points.
    Mastering common diagnostic commands for OSD, PG, Pool, CRUSH, and capacity.
    Mastering independent troubleshooting paths for RBD, CephFS, RGW, and CSI.
    Understanding that HEALTH_OK does not equate to no risks.
    Understanding that HEALTH_WARN cannot be ignored.
    Understanding that HEALTH_ERR requires immediate response.
    Understanding that high-risk commands must have approval, backups, rollback, and verification.

---

### 3.5 Fifth Mainline: Production Governance

Core questions:

    How to achieve production readiness for Ceph?
    How to monitor?
    How to set alerts?
    How to back up?
    How to recover?
    How to strengthen security?
    How to conduct failure drills?

Phase gains:

    Understanding that Ceph replicas are not backups.
    Understanding that snapshots are not complete backups.
    Mastering backup and recovery approaches like RBD export/import, CephFS rsync/tar, and RGW s3 sync.
    Mastering the Prometheus, Grafana, and Alertmanager monitoring and alerting system.
    Mastering CephX minimal privilege user design.
    Mastering RGW HTTPS unified entry point design.
    Mastering security governance for Dashboard, Secret, keyring, and AccessKey.
    Understanding that production readiness is not just successful deployment, but a complete cycle of deployment, usage, monitoring, failure, recovery, security, and processes.

---

## IV. Ceph Core Knowledge Consolidation

### 4.1 Ceph Overall Architecture

Ceph can be understood in three layers:

    First layer: Underlying storage layer
      RADOS, OSD, Pool, PG, CRUSH

    Second layer: Control and management layer
      MON, MGR, CephX, cephadm, Dashboard, Prometheus

### Third Layer: Access Interface Layer  
  RBD, CephFS, RGW, CSI  

Simplified Architecture:  

    Client / Kubernetes / S3 SDK  
        |  
        | RBD / CephFS / RGW / CSI  
        v  
    RADOS  
        |  
        | Pool -> PG -> CRUSH  
        v  
    OSD  
        |  
        v  
    Disk / Node / Failure Domain  

---

### 4.2 Responsibilities of Core Components  

| Component | Role | Stores Business Data |  
|---|---|---|  
| MON | Maintains cluster Map, quorum, and state consistency | No |  
| MGR | Management, Dashboard, Prometheus, orchestrator | No |  
| OSD | Stores real object data, replicas, recovery | Yes |  
| MDS | CephFS metadata service | Does not directly store business file content |  
| RGW | S3 / Swift object storage entry point | Does not directly write to disk, data lands in RADOS |  
| RADOS | Ceph's underlying distributed object storage | Yes |  
| CRUSH | Data distribution algorithm | No |  
| Pool | Logical storage pool | Yes |  
| PG | Data placement group | Yes |  

---

### 4.3 Three Types of Storage Interfaces  

| Type | Essence | Typical Scenarios | Kubernetes Access Mode |  
|---|---|---|---|  
| RBD | Block Storage | Database, virtual machine disk, single instance data disk | ReadWriteOnce |  
| CephFS | File Storage | Multi-node shared directory, shared upload directory | ReadWriteMany |  
| RGW | Object Storage | Images, attachments, backups, archives, S3 API | Not via PVC, uses S3 API |  

Selection Mnemonic:  

    Use like a cloud disk: RBD  
    Use like a shared directory: CephFS  
    Use like OSS/S3: RGW  

---

## Five, From "Deployment" to "Operations" Capability Evolution  

### 5.1 Before Deployment  

When first learning Ceph, the focus is typically:  

    How to install?  
    How to pull images?  
    How to bootstrap?  
    How to add nodes?  
    How to add OSDs?  
    How to see HEALTH_OK?  

This is the beginner stage.  

---

### 5.2 After Deployment  

Once the cluster is running, the real issues become:  

    What to do if an OSD is down?  
    What to do if PG is not clean?  
    What to do if an OSD is nearly full?  
    What to do if RBD is unreachable?  
    What to do if CephFS access is slow?  
    What to do if RGW S3 reports signature errors?  
    What to do if Kubernetes PVC is always Pending?  
    How to troubleshoot if users say the business is slow?  
    Will deleting PVC delete underlying data?  
    Is backup still needed if there are three replicas?  
    Can the Dashboard be exposed to the public internet?  
    What to do if an AccessKey is leaked?  
    What to do if alerts are not notified to people?  

This is the operations stage.  

---

### 5.3 Production Stage  

In the production stage, the focus shifts further to:  

    Is there monitoring?  
    Is there alerting?  
    Is there backup?  
    Has recovery drills been done?  
    Has failure drills been done?  
    Is there least privilege?  
    Is there capacity planning?  
    Is there a change management process?  
    Is there a rollback plan?  
    Can risk levels be judged?  
    Can each alert's impact be explained?  

This is the advanced SRE stage.  

---

## Six, Core Commands for Ceph Production Operations  

### 6.1 Cluster Health  

    ceph -s  
    ceph health detail  
    ceph versions  
    ceph crash ls-new  

---

### 6.2 MON / MGR  

    ceph mon stat  
    ceph quorum_status --format json-pretty  
    ceph mgr stat  
    ceph mgr services  
    ceph mgr module ls  

---

### 6.3 OSD  

    ceph osd stat  
    ceph osd tree  
    ceph osd df  
    ceph osd perf  
    ceph osd metadata osd.X  
    ceph orch ps --daemon_type osd  

---

### 6.4 PG  

    ceph pg stat  
    ceph pg dump_stuck  
    ceph pg map <pg-id>  
    ceph pg <pg-id> query  

---

### 6.5 Pool  

    ceph osd pool ls  
    ceph osd pool ls detail  
    ceph osd pool get <pool-name> all  
    ceph osd pool application get <pool-name>  
    ceph osd pool autoscale-status  
    ceph df  

---

### 6.6 RBD  

    rbd ls -p <pool-name>  
    rbd info <pool-name>/<image-name>  
    rbd status <pool-name>/<image-name>  
    rbd snap ls <pool-name>/<image-name>  
    rbd map <pool-name>/<image-name>  
    rbd unmap /dev/rbdX  
    rbd export <pool>/<image> <file>  
    rbd import <file> <pool>/<image>  

---

### 6.7 CephFS  

    ceph fs ls  
    ceph fs status  
    ceph mds stat  
    ceph orch ps --daemon_type mds  
    ceph fs subvolumegroup ls <fs-name>  
    ceph fs subvolume ls <fs-name> --group_name <group-name>  

---

### 6.8 RGW

ceph orch ps --daemon_type rgw
radosgw-admin user list
radosgw-admin user info --uid=<uid>
radosgw-admin bucket list
radosgw-admin bucket stats --bucket=<bucket-name>
aws --profile <profile> --endpoint-url <endpoint> s3 ls

---

### 6.9 Kubernetes CSI

    kubectl get pods -n ceph-csi -o wide
    kubectl get storageclass
    kubectl get pvc -A
    kubectl get pv
    kubectl get volumeattachment
    kubectl describe pvc -n <namespace> <pvc-name>
    kubectl describe pod -n <namespace> <pod-name>
    kubectl logs -n ceph-csi <pod-name> -c <container-name>

---

## SevenI don't know.Ceph Troubleshooting Matrix

| Fault Phenomenon | First Entry Point | Second Entry Point | Deep Dive Direction |
|---|---|---|---|
| HEALTH_WARN | ceph health detail | ceph -s | Split by alert type |
| OSD down | ceph osd tree | ceph orch ps --daemon_type osd | Node, disk, network, log |
| PG degraded | ceph pg stat | ceph health detail | OSD, Pool, CRUSH, Recovery |
| PG inactive | ceph pg dump_stuck inactive | ceph pg query | High risk, preserve scene |
| Capacity Alert | ceph df | ceph osd df | Pool, RBD, RGW, CephFS |
| RBD Mount Failure | rbd info/status | dmesg | keyring, MON, feature, permission |
| CephFS Slow | ceph fs status | ceph mds stat | MDS, small files, large directory, OSD |
| RGW Access Failure | curl / aws s3 ls | radosgw-admin user info | Endpoint, Key, Nginx, certificate |
| PVC Pending | kubectl describe pvc | CSI Controller logs | SC, Secret, Pool, permission |
| Pod FailedMount | kubectl describe pod | CSI Node logs | Node, network, Secret, module |
| slow ops | ceph health detail | ceph osd perf | Disk, network, Recovery |

---

## EightI don't know.Ceph Production Risk Summary

### 8.1 Data Risk

Common Risks:

- Accidentally delete Pool
- Accidentally delete RBD Image
- Accidentally delete PVC
- Delete RGW user with --purge-data
- Snapshot expired without cleanup
- No independent backup
- Mistakenly think three replicas equal backup
- Only backup without recovery drill

Governance Methods:

    High-risk command approval.
    Critical data Retain.
    Regular backup.
    Regular recovery drill.
    Confirm ownership before deletion.
    Backup files in independent storage.

---

### 8.2 Capacity Risk

Common Risks:

- OSD nearfull
- OSD full
- Pool growth anomaly
- RGW Bucket object count surge
- RBD snapshot long-term not cleaned
- CephFS large number of small files
- Only check cluster total capacity, not OSD and Pool

Governance Methods:

    70% focus on trend.
    80% plan expansion.
    85% high priority handling.
    nearfull handle immediately.
    full is severe failure.
    Continuously monitor Pool growth trend.

---

### 8.3 Security Risk

Common Risks:

- Business uses client.admin
- Admin key leakage
- RGW HTTP exposed to public internet
- Dashboard exposed to public internet
- Kubernetes regular user can read ceph-csi Secret
- AccessKey long-term not rotated
- cephadm SSH private key too broad
- auth export placed in regular directory

Governance Methods:

    Minimum privilege.
    Network isolation.
    HTTPS entry.
    Secret RBAC control.
    keyring permission convergence.
    Regular permission review.
    Key rotation process.

---

### 8.4 Operation Risk

Common Risks:

- Not check health detail
- Repair immediately upon seeing PG inconsistent
- Purge directly after OSD down
- Continue expansion/upgrade when cluster unhealthy
- Forget to cancel noout setting
- Long-term silence alerts
- No change record
- No rollback plan

Governance Methods:

    Read-only commands first.
    High-risk command review.
    Maintenance flag daily check.
    Alert silence must have expiration.
    Change must be recorded.
    Fault must be post-mortem.

---

## NineI don't know.Ceph Advanced SRE Capability Achievement

After this phase of learning, should possess the following capabilities.

### 9.1 Architecture Explanation Capability

Can explain:

    What is Ceph.
    What is RADOS.
    What do MON, MGR, OSD, MDS, RGW respectively handle.
    Difference between RBD, CephFS, RGW.
    Relationship between Pool, PG, CRUSH, OSD.
    How Ceph achieves replica and failure domain high availability.

---

### 9.2 Deployment Implementation Capability

Can complete:

    Ubuntu / Rocky Linux basic environment preparation.
    Domestic software source configuration.
    cephadm installation.
    Multi-node bootstrap.
    MON / MGR / OSD deployment.
    MDS / RGW deployment.
    Dashboard enablement.
    Prometheus module enablement.
    Client ceph-common configuration.

---

### 9.3 Storage Delivery Capability

Can deliver:

# Ceph Storage Systems

RBD Block Storage.  
CephFS Shared File System.  
RGW S3 Object Storage.  
Kubernetes RBD CSI.  
Kubernetes CephFS CSI.  
RGW HTTPS Unified Entry Point.

---

### 9.4 Operational Troubleshooting Capabilities

Can troubleshoot:

    OSD down.  
    OSD full / nearfull.  
    PG degraded.  
    PG undersized.  
    PG inactive.  
    PG inconsistent.  
    MON quorum anomaly.  
    MDS anomaly.  
    RGW S3 access anomaly.  
    PVC Pending.  
    Pod FailedMount.  
    slow ops.  
    Capacity growth anomaly.

---

### 9.5 Production Governance Capabilities

Can design:

    Daily inspection checklist.  
    Monitoring alert rules.  
    Capacity alert thresholds.  
    Backup recovery process.  
    Security hardening baseline.  
    Minimum privilege users.  
    Fault drill plan.  
    Production acceptance checklist.  
    Fault post-mortem template.

---

## Ten, Why It's Easier to Learn Other Systems After Ceph

### 10.1 Why Learning MinIO Is Easier

Because you already understand:

    What object storage is.  
    What Bucket / Object / AccessKey / SecretKey are.  
    How to use S3 API.  
    Why HTTPS is needed externally.  
    Why Nginx / LB is placed in front.  
    Why Bucket capacity, object count, and quotas matter.  
    Why backup, lifecycle, and permission governance are important.

When learning MinIO, focus on comparing:

    MinIO's erasure coding.  
    MinIO's distributed deployment.  
    MinIO Console.  
    MinIO mc tool.  
    Differences between MinIO and RGW architecture.  
    MinIO community edition version selection and image management.

---

### 10.2 Why Learning Longhorn Is Easier

Because you already understand:

    Kubernetes PV / PVC / StorageClass.  
    CSI Controller / Node Plugin.  
    Differences between RWO and RWX.  
    Relationship between replicas, nodes, volumes, and recovery.  
    How storage systems integrate with Kubernetes.  
    How to troubleshoot PVC Pending / FailedMount.  
    Why you can't disrupt containerd's underlying runtime.

When learning Longhorn, focus on comparing:

    Longhorn is Kubernetes-native storage.  
    How Longhorn volume replicas are distributed.  
    Longhorn Manager / Engine / Instance Manager.  
    Longhorn UI.  
    Longhorn Backup Target.  
    Differences between Longhorn and RBD CSI.

---

### 10.3 Why Learning RustFS Is Easier

Because you already understand:

    Object storage basic model.  
    S3 API.  
    Single-machine vs. cluster mode.  
    Reverse proxy and HTTPS.  
    Internal communication strategy.  
    Image management.  
    Bucket, user, permissions, backup, and monitoring.

When learning RustFS, focus on comparing:

    RustFS deployment mode.  
    Compatibility and differences between RustFS and MinIO.  
    RustFS cluster node planning.  
    RustFS management entry point.  
    RustFS production maturity and applicability boundaries.

---

### 10.4 Why Learning Other Cloud Vendor Storage Is Easier

After mastering Ceph, cloud vendor storage becomes clearer:

| Cloud Product | Corresponding Understanding |
|---|---|
| Cloud Disk | Similar to RBD |
| NAS / File Storage | Similar to CephFS |
| OSS / S3 | Similar to RGW / MinIO |
| Snapshots | Similar to RBD Snapshot / Cloud Disk Snapshot |
| Multi-replica | Similar to Ceph Pool size |
| Availability Zone | Similar to fault domain |
| Storage Class | Similar to Pool / StorageClass |
| Lifecycle | Similar to object storage governance |
| Permission Policy | Similar to CephX / S3 Policy |

---

## Eleven, Subsequent Strengthening Directions

The Ceph phase has completed the transition from advanced SRE entry to productionization closure, but further deepening is still possible.

### 11.1 Ceph Advanced Directions

Subsequent deepening can include:

- RBD Mirror  
- RGW Multi-Site  
- CephFS Multi-active MDS  
- Erasure Coding Pool  
- BlueStore Internal Mechanism  
- RocksDB / WAL / DB Device Planning  
- OSD Internal Recovery Mechanism  
- PG Peering Deepening  
- CRUSH Map Advanced Design  
- Cephadm Upgrade and Rollback  
- Large-Scale Cluster Operations

---

### 11.2 Kubernetes Storage Directions

Subsequent deepening can include:

- Rook Ceph  
- VolumeSnapshot  
- CSI Snapshot Controller  
- Velero Backup PVC  
- StatefulSet Data Protection  
- Multi StorageClass Design  
- PVC Migration  
- CSI Monitoring Metrics  
- Storage Admission Control  
- Multi-Tenant Storage Quota

---

### 11.3 Platformization Directions

Subsequent deepening can include:

- Self-service storage resource application  
- RBD / PVC / Bucket Lifecycle Management  
- Storage Capacity Quota System  
- Storage Monitoring Platform  
- Automated Inspection Platform  
- Fault Automation Diagnosis  
- Backup Recovery Platformization  
- CMDB Integration with Ceph Resources  
- Unified Management Platform for Multiple Storage Systems

---

## Twelve, Stage Summary Evaluation

From the learning path, this phase has progressed from "knowing what Ceph is" to "being able to understand Ceph production operations from an advanced SRE perspective."

The capability evolution can be summarized as:

    From component names -> architecture relationships  
    From installation deployment -> production acceptance  
    From single-command operations -> troubleshooting paths  
    From storage usage -> data protection  
    From health status -> monitoring alerts  
    From being able to run -> long-term operations  
    From single-software learning -> distributed storage methodology

After completing the Ceph phase, you already have the foundation to continue learning MinIO, Longhorn, RustFS, and cloud-native storage systems.

---

## Thirteen, Interview Expression Summary

If asked in an interview:

    How familiar are you with Ceph?

You can respond:

I have systematically studied and organized the complete Ceph operations framework, not just deployment. I understand the data distribution relationships of Ceph's underlying RADOS, Pool, PG, CRUSH, and OSD, and can distinguish the applicable scenarios for the three storage interfaces: RBD, CephFS, and RGW.

In deployment, I can use cephadm to plan multi-node Ceph clusters, configure MON, MGR, OSD, MDS, RGW, and combine with Ubuntu or Rocky Linux for base environment initialization and software source configuration.

In usage, I can create RBD block storage, CephFS file systems, RGW object storage, and integrate with Kubernetes RBD CSI and CephFS CSI to implement RWO and RWX type PVCs.

In operations, I will start troubleshooting OSD, PG, capacity, performance, and service anomalies from commands like ceph -s, ceph health detail, ceph osd tree, ceph pg stat, ceph osd df, and ceph osd perf.

In production governance, I focus on monitoring alerts, backup recovery, security reinforcement, minimal permissions, RGW HTTPS, Dashboard security, Kubernetes Secret protection, fault drills, and production validation.

I understand that Ceph's three-replica is not backup, and snapshots are not full backups. Production requires independent backup and recovery drills. Overall, I don't stop at installation level, but understand Ceph's deployment, usage, monitoring, fault handling, recovery, and security closure from an advanced SRE perspective.

---

## Fourteen, Summary of This Section

This article completes the Ceph phase summary:

1. The Ceph phase establishes a complete distributed storage knowledge system.
2. Ceph covers block, file, and object storage simultaneously.
3. MON, MGR, OSD, MDS, and RGW are core Ceph operation components.
4. Pool, PG, and CRUSH are core elements of Ceph's data distribution and high availability.
5. RBD is suitable for cloud disks and Kubernetes RWO.
6. CephFS is suitable for shared directories and Kubernetes RWX.
7. RGW is suitable for S3 object storage scenarios.
8. Ceph daily operations must focus on health, OSD, PG, Pool, capacity, MDS, RGW, and CSI.
9. Ceph fault troubleshooting must start with ceph -s and ceph health detail.
10. Ceph production environments must have monitoring alerts, backup recovery, security reinforcement, and fault drills.
11. Ceph replicas are not backups, and snapshots are not full backups.
12. Ceph performance optimization should first locate bottlenecks, not directly adjust parameters.
13. Ceph security governance must adhere to minimal permissions, network isolation, HTTPS, and key protection.
14. The value of Ceph comprehensive practice lies in verifying the complete production closure.
15. After studying Ceph, learning MinIO, Longhorn, and RustFS will be significantly easier.
16. At this point, the Ceph phase has completed the full closure from architecture understanding to production operations.

---

## Fifteen, Reference Documents

Ceph official documentation:

    https://docs.ceph.com/en/latest/

Cephadm documentation:

    https://docs.ceph.com/en/latest/cephadm/

Ceph RADOS:

    https://docs.ceph.com/en/latest/rados/

Ceph RBD:

    https://docs.ceph.com/en/latest/rbd/

CephFS:

    https://docs.ceph.com/en/latest/cephfs/

Ceph RGW:

    https://docs.ceph.com/en/latest/radosgw/

Ceph user management:

    https://docs.ceph.com/en/latest/rados/operations/user-management/

Ceph Pool management:

    https://docs.ceph.com/en/latest/rados/operations/pools/

Ceph Placement Groups:

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph CRUSH Map:

    https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph health checks:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph troubleshooting:

    https://docs.ceph.com/en/latest/rados/troubleshooting/

Ceph Dashboard:

    https://docs.ceph.com/en/latest/mgr/dashboard/

Ceph Prometheus module:

    https://docs.ceph.com/en/latest/mgr/prometheus/

Ceph CSI project:

    https://github.com/ceph/ceph-csi

Kubernetes StorageClass:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/