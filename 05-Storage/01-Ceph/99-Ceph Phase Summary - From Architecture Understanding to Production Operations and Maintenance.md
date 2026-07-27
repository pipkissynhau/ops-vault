# Ceph Phase Summary: From Architecture Understanding to Production Operations and Maintenance

Recommended Path: 05-Storage/01-Ceph/99-Ceph Phase Summary: From Architecture Understanding to Production Operations and Maintenance.md

Tags: #Ceph #Phase Summary #Distributed Storage #Advanced SRE #Production Operations and Maintenance #RBD #CephFS #RGW #CSI #Monitoring and Alerts #Troubleshooting #Backup and Recovery

---

## I. Document Description

This document serves as a summary of the Ceph module phase, aimed at concluding the entire learning journey regarding Ceph.

Previously, a comprehensive learning pathway from basic theories to practical applications has been covered, including:

- Ceph architecture and core components
- RADOS, Pool, PG, CRUSH data distribution models
- RBD block storage
- CephFS file storage
- RGW object storage
- cephadm multi-node deployment
- OSD management, scaling, removal, and replacement
- Kubernetes RBD CSI / CephFS CSI integration
- Routine inspections and health checks
- Troubleshooting
- Backup and recovery, as well as disaster recovery drills
- Performance optimization
- Security enhancement
- Monitoring and alerts
- Practical applications and production acceptance

The focus of this document is not to introduce new knowledge but to summarize:

    What has been learned during this phase?
    What capabilities have been acquired?
    Which contents are essential for production operations?
    Which areas require further in-depth study?
    Why learning MinIO, Longhorn, or RustFS after mastering Ceph makes it easier?

---

## II. Phase Positioning

Ceph is one of the most comprehensive, complex modules in the field of distributed storage systems and offers excellent opportunities to develop advanced SRE skills.

It encompasses three types of storage capabilities:

    Block storage: RBD
    File storage: CephFS
    Object storage: RGW

It also covers some of the most critical aspects of production operations and maintenance:

    High availability
    Data replication
    Fault domains
    Automatic recovery
    Capacity management
    Performance bottleneck identification
    Permission management
    Monitoring and alerts
    Backup and recovery
    Disaster recovery drills
    Kubernetes CSI integration

Therefore, the value of the Ceph phase lies not just in learning a storage software but in establishing a comprehensive understanding of distributed storage operations and maintenance.

---

## III. Main Learning Threads of the Ceph Phase

The entire Ceph module can be summarized into five main threads.

### 3.1 First Thread: Architecture Understanding

Key questions:

    What is Ceph?
    How does Ceph provide block, file, and object storage simultaneously?
    What are the roles of MON, MGR, OSD, MDS, and RGW?
    What is the position of RADOS within Ceph?

Gains from this phase:

    Understanding that Ceph's foundation is the unified RADOS distributed object storage system.
    Recognizing that RBD, CephFS, and RGW are different interfaces built on top of RADOS.
    Comprehending that OSDs are the core for storing actual data.
    Grasping that MON is responsible for maintaining cluster status and consistency.
    Understanding that MGR manages, orchestrates, monitors, and provides a dashboard.
    Realizing that MDS serves exclusively CephFS.
    Recognizing that RGW acts as an object storage interface but does not store data itself.

---

### 3.2 Second Thread: Data Distribution

Key questions:

    Where do data go after being written to Ceph?
    What is the relationship between Pool, PG, CRUSH, and OSDs?
    How does Ceph ensure that replicas are distributed across different fault domains?
    Why does the performance of a PG degrade when an OSD goes down?

Gains from this phase:

    Understanding that Pool represents a logical storage pool.
    Recognizing that PG acts as an intermediate layer that maps objects to OSDs.
    Comprehending how CRUSH determines where data is distributed across OSDs.
    Knowing how size controls the number of replicas and min_size sets the minimum required number of writable replicas.
    Understanding the importance of host/rack fault domain design for high availability.
    Recognizing the various states of PG, such as active+clean, degraded, undersized, inactive, and inconsistent.

---

### 3.3 Third Thread: Storage Capabilities

Key questions:

    What are the appropriate use cases for RBD, CephFS, and RGW?
    Why are databases better suited for RBD?
    Why are shared directories among multiple pods more suitable for CephFS?
    Why are images, attachments, and backups better suited for RGW?

Gains from this phase:

    Recognizing that RBD is similar to cloud block storage and is ideal for block storage and Kubernetes Read-Write Once operations.
    Understanding that CephFS is like a shared file system, suitable for multi-client sharing and Kubernetes Read-Write Many operations.
    Real### 6.1 Cluster Health

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

## VII. Ceph Troubleshooting Guide

| Issue | First Step | Second Step | Further Investigation |
|---|---|---|---|
| HEALTH_WARN | ceph health detail | ceph -s | Filter by alarm type |
| OSD Down | ceph osd tree | ceph orch ps --daemon_type osd | Check nodes, disks, network, logs |
| PG Degraded | ceph pg stat | ceph health detail | Verify OSDs, Pools, CRUSH, Recovery |
| PG Inactive | ceph pg dump_stuck inactive | ceph pg query | High-risk situation; preserve evidence |
| Capacity Alert | ceph df | ceph osd df | Check Pools, RBD, RGW, CephFS |
| RBD Mount Failure | rbd info/status | dmesg | Verify keyring, MON, features, permissions |
| CephFS Slowness | ceph fs status | ceph mds stat | Check MDS, small files, large directories, OSDs |
| RGW Access Failure | curl /aws s3 ls | radosgw-admin user info | Verify endpoint, Key, Nginx, certificates |
| PVC Pending | kubectl describe pvc | CSI Controller logs | Check SC, Secret, Pool, permissions |
| Pod Failed Mount | kubectl describe pod | CSI Node logs | Verify nodes, network, Secret, modules |
| Slow Operations | ceph health detail | ceph osd perf | Evaluate disks, network, Recovery processes |

---

## VIII. Summary of Ceph Production Risks

### 8.1 Data Risks

Common risks:

- Accidental deletion of Pools
- Accidental deletion of RBD Images
- Accidental deletion of PVCs
- Using --purge-data when deleting RGW users
- Untimely cleanup of expired snapshots
- Lack of independent backups
- Misunderstanding three replicas as sufficient backup
- Failing to perform recovery drills

Mitigation strategies:

    Approval for high-risk commands.
    Retention of critical data.
    Regular🔤 MON quorum exception.
🔤 MDS exception.
🔤 RGW S3 access issue.
🔤 PVC pending.
🔤 Pod FailedMount.
🔤 Slow operations.
🔤 Abnormal capacity growth.

---

### 9.5 Production Governance Capabilities

Ability to design:

- Daily inspection checklists.
- Monitoring and alert rules.
- Capacity alert thresholds.
- Backup and recovery procedures.
- Security enhancement baselines.
- Least privilege user models.
- Disaster recovery drill plans.
- Production acceptance forms.
- Fault analysis templates.

---

## Section X: Why Learning Ceph Makes It Easier to Learn Other Storage Solutions Later On

### 10.1 Learning MinIO Is Easier

Because you already understand:

- What object storage is.
- What Bucket, Object, AccessKey, and SecretKey are.
- How to use the S3 API.
- Why HTTPS is necessary for external access.
- Why Nginx/LB are used at the front end.
- Why it's important to monitor Bucket capacity, number of objects, and quotas.
- The importance of backup, lifecycle management, and permission control.

When learning MinIO, you only need to focus on comparing:

- MinIO's erasure coding mechanism.
- MinIO's distributed deployment methods.
- The MinIO Console interface.
- The MinIO mc command tool.
- Architectural differences between MinIO and RGW.
- How to choose and manage community versions of MinIO.

---

### 10.2 Learning Longhorn Is Easier

Because you already understand:

- Kubernetes concepts such as PV, PVC, and StorageClass.
- The role of CSI Controllers and Node Plugins.
- The difference between RWO and RWX.
- How backups, nodes, volumes, and recovery processes are related.
- How storage systems integrate with Kubernetes.
- How to identify issues with PVC Pending or FailedMount.
- Why it's crucial not to disrupt the underlying containerd runtime.

When learning Longhorn, you only need to focus on comparing:

- How Longhorn is integrated into Kubernetes as a native storage solution.
- How Longhorn manages volume replicas across nodes.
- The components of Longhorn such as Manager, Engine, and Instance Manager.
- The user interface of Longhorn.
- Longhorn's backup targets.
- Architectural differences between Longhorn and RBD CSI.

---

### 10.3 Learning RustFS Is Easier

Because you already understand:

- The basic model of object storage.
- The S3 API.
- Both single-machine and cluster deployment options.
- The role of reverse proxies and HTTPS.
- Internal communication mechanisms.
- Image management practices.
- Concepts related to Buckets, users, permissions, backups, and monitoring.

When learning RustFS, you only need to focus on comparing:

- RustFS's deployment models.
- How RustFS differs from and is compatible with MinIO.
- Planning for cluster nodes in a RustFS environment.
- The management interface of RustFS.
- The maturity level and applicable use cases of RustFS.

---

### 10.4 Learning Other Cloud Vendor Storage Solutions Is Easier

After studying Ceph, understanding other cloud vendor storage solutions becomes much clearer:

| Cloud Product | Corresponding Concepts |
|---|---|
| Cloud Block Storage | Similar to RBD |
| NAS/File Storage | Similar to CephFS |
| OSS/S3 | Similar to RGW/MinIO |
| Snapshots | Similar to RBD Snapshot/Cloud Disk Snapshots |
| Multiple Replicas | Similar to Ceph's Pool size configuration |
| Availability Zones | Similar to Fault Domains |
| Storage Classes | Similar to Pool/StorageClass concepts |
| Lifecycle Management | Similar to object storage governance strategies |
- Permission Policies | Similar to CephX/S3 Policy mechanisms |

---

## Section XI: Areas for Further Improvement

Although the Ceph phase has covered everything from basic SRE knowledge to production-level operations, there are still areas for further exploration:

### 11.1 Advanced Ceph Topics

Future areas of study could include:

- RBD Mirror technology.
- Multi-site RGW configurations.
- Multiple active MDS setups in CephFS.
- In-depth understanding of Erasure Coding and its pool management.
- The internal workings of BlueStore.
- Planning for RocksDB/WAL/DB devices.
- Investigating OSDs' internal recovery mechanisms.
- Delving into PG peering configuration.
- Advanced design of CRUSH Maps.
- Upgrading and rolling back Cephadm versions.
- Managing large-scale Ceph clusters.

---

### 11.2 Kubernetes Storage Related Topics

Future areas of study could include:

- Integrating Rook with Ceph for additional storage solutions.
- Using VolumeSnapshot and CSI Snapshot Controller features.
- Implementing Velero for PVC backup.
- Protecting StatefulSet data using appropriate storage strategies.
- Designing multi-StorageClass configurations.
- Migrating PVCs between different storage systems.
- Monitoringhttps://docs.ceph.com/en/latest/rados/operations/placement-groups/

Ceph CRUSH Map:

https://docs.ceph.com/en/latest/rados/operations/crush-map/

Ceph Health Checks:

https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Troubleshooting:

https://docs.ceph.com/en/latest/rados/troubleshooting/

Ceph Dashboard:

https://docs.ceph.com/en/latest/mgr/dashboard/

Ceph Prometheus Module:

https://docs.ceph.com/en/latest/mgr/prometheus/

Ceph CSI Project:

https://github.com/ceph/ceph-csi

Kubernetes StorageClass:

https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Persistent Volumes:

https://kubernetes.io/docs/concepts/storage/persistent-volumes/