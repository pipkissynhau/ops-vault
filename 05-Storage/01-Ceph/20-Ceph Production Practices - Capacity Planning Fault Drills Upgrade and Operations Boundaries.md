# Ceph Comprehensive Practical: Production Acceptance, Fault Drills, and Advanced SRE Loop

Recommended Path: 05-Storage/01-Ceph/20-Ceph Comprehensive Practical: Production Acceptance, Fault Drills, and Advanced SRE Loop.md

Tags: #Ceph #IntegratedFieldOperations #ProductionLaboratoryCollection #FaultExercise #RBD #CephFS #RGW #CSI #SecurityAlert. #BackupRestore #AdvancedSre

---

## I. Document Description

This is the twentieth article of the Ceph advanced SRE storage module, and also the comprehensive practical article for the Ceph module.

Previously completed:

- Understanding of Ceph architecture and components
- cephadm cluster deployment
- OSD management
- Pool and PG
- CRUSH fault domain
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Daily operations and maintenance
- Fault diagnosis
- Backup and recovery
- Performance optimization
- Security reinforcement
- Monitoring and alerts

This article does not focus on a single component anymore, but combines the previous capabilities to form a comprehensive experiment close to production scenarios for advanced SRE.

The goal is to verify:

    Whether the Ceph cluster can be deployed
    Whether Ceph storage can be used
    Whether Ceph data can be recovered
    Whether Ceph services can be monitored
    Whether Ceph faults can be simulated
    Whether Ceph can integrate with Kubernetes
    Whether Ceph meets production delivery conditions

---

## II. Comprehensive Practical Objectives

After completing this article, you should be able to:

1. Design an experimental topology for Ceph + Kubernetes external storage.
2. Complete production acceptance checks for the Ceph cluster.
3. Create three types of storage services: RBD, CephFS, and RGW.
4. Verify RBD block storage read/write, snapshots, and recovery.
5. Verify CephFS shared file system read/write, snapshots, and recovery.
6. Verify RGW S3 Bucket, object upload/download, and HTTPS entry.
7. Verify Kubernetes RBD CSI dynamic volume provisioning.
8. Verify Kubernetes CephFS CSI ReadWriteMany shared storage.
9. Execute single OSD fault drill.
10. Execute single node maintenance drill.
11. Execute MDS fault switch drill.
12. Execute RGW multi-instance entry fault drill.
13. Verify Prometheus / Grafana / Alertmanager monitoring and alerts.
14. Verify backup and recovery process.
15. Output production acceptance checklist.
16. Form an advanced SRE Ceph operations loop.

---

## III. Experimental Topology

### 3.1 Ceph Cluster Nodes

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / MDS / RGW |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / MDS / RGW |
| 10.0.0.33 | ceph-node03 | MON / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Drill, Optional |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client |
| 10.0.0.36 | ceph-entry | Nginx / RGW HTTPS Unified Entry / Monitoring Node, Optional |

Operating System:

    Ubuntu Server 22.04.5 LTS

Supplement:

    Rocky Linux 9 can be used as a supplementary experimental system.

---

### 3.2 Kubernetes Cluster Nodes

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Runtime Environment:

    kubeadm
    containerd
    Calico
    Ubuntu Server 22.04.5 LTS

---

### 3.3 Network Relationships

All nodes are located in:

    10.0.0.0/24

Ceph critical access relationships:

    Kubernetes nodes -> Ceph MON: 3300 / 6789
    Kubernetes nodes -> Ceph OSD: 6800-7300
    ceph-client -> Ceph MON / OSD / RGW
    ceph-entry -> RGW backend 7480
    External clients -> ceph-entry HTTPS 443

---

## IV. Comprehensive Practical Architecture Diagram /think

```
┌──────────────────────────────────────────────┐
│              Kubernetes Cluster                │
│  10.0.0.20 / 10.0.0.21 / 10.0.0.22            │
│                                              │
│  PVC -> RBD CSI    -> Ceph RBD                │
│  PVC -> CephFS CSI -> CephFS                  │
└──────────────────────┬───────────────────────┘
                       │
                       │ MON / OSD Network
                       v
┌──────────────────────────────────────────────┐
│                 Ceph Cluster                   │
│                                              │
│  MON: ceph-node01/02/03                       │
│  MGR: ceph-node01/02                          │
│  OSD: ceph-node01/02/03/04                    │
│  MDS: ceph-node01/02                          │
│  RGW: ceph-node01/02/03                       │
│                                              │
│  RBD    -> Block Storage                      │
│  CephFS -> File Sharing                        │
│  RGW    -> S3 Object Storage                   │
└──────────────────────┬───────────────────────┘
                       │
                       │ HTTP 7480
                       v
┌──────────────────────────────────────────────┐
│         Unified Entry / Monitoring / Backup Node              │
│              10.0.0.36 ceph-entry              │
│                                              │
│  Nginx HTTPS 443 -> RGW Backend                │
│  Prometheus / Grafana / Alertmanager           │
│  Backup Directory                             │
└──────────────────────────────────────────────┘
```

---

## Five. Comprehensive Practical Task List

| Stage | Task | Objective |
|---|---|---|
| Stage One | Basic Health Acceptance | Confirm Ceph Cluster Availability |
| Stage Two | Storage Capability Acceptance | RBD / CephFS / RGW Availability |
| Stage Three | Kubernetes Integration Acceptance | RBD CSI / CephFS CSI Availability |
| Stage Four | Fault Simulation | OSD / MDS / RGW / Node Faults Recoverable |
| Stage Five | Backup Recovery Verification | RBD / CephFS / RGW Recoverable |
| Stage Six | Monitoring Alert Verification | Alerts Trigger, Notify, and Recover |
| Stage Seven | Security Check | Permissions, Keys, Entry, Ports Meet Baseline |
| Stage Eight | Production Acceptance | Output Final Acceptance Results |

---

## Six. Stage One: Ceph Cluster Basic Health Acceptance

### 6.1 Check Cluster Status

    ceph -s

Acceptance Criteria:

    health is HEALTH_OK, or HEALTH_WARN with clear reasons.
    MON quorum is normal.
    MGR is active.
    All OSDs are up/in.
    PG is active+clean.
    No nearfull / full.
    No slow ops.
    No unknown crashes.

---

### 6.2 Check Health Details

    ceph health detail

If there are alerts, they must be recorded:

| Alert | Reason | Impact on Business | Action Required |
|---|---|---|---|
|  |  |  |  |

---

### 6.3 Check OSD Status

    ceph osd tree
    ceph osd stat
    ceph osd df
    ceph osd perf

Acceptance Criteria:

    All OSDs are up/in.
    OSD usage has no severe imbalance.
    OSD latency has no single-point anomalies.
    OSD distribution meets host failure domain requirements.

---

### 6.4 Check PG Status

    ceph pg stat

Acceptance Criteria:

    All PGs are active+clean.
    No degraded, undersized, inactive, incomplete, inconsistent, or stale PGs.

---

### 6.5 Check Service Status

    ceph orch ps
    ceph orch ls

Acceptance Criteria:

    mon, mgr, osd, mds, rgw services are running.
    No daemon in error, stopped, or unknown state.

---

## Seven. Stage Two: RBD Block Storage Acceptance

### 7.1 Create RBD Pool

# 7.1 Create RBD Pool

    ceph osd pool create prod-rbd 64
    ceph osd pool application enable prod-rbd rbd
    ceph osd pool set prod-rbd size 3
    ceph osd pool set prod-rbd min_size 2
    rbd pool init prod-rbd

Check:

    ceph osd pool get prod-rbd all
    ceph osd pool application get prod-rbd

---

### 7.2 Create Minimal Privilege User

    ceph auth get-or-create client.prod-rbd \
      mon 'profile rbd' \
      mgr 'profile rbd pool=prod-rbd' \
      osd 'profile rbd pool=prod-rbd' \
      -o /etc/ceph/ceph.client.prod-rbd.keyring

Check:

    ceph auth get client.prod-rbd

---

### 7.3 Create RBD Image

    rbd --id prod-rbd create prod-image-01 --size 10G -p prod-rbd --image-feature layering

Check:

    rbd --id prod-rbd info prod-rbd/prod-image-01

---

### 7.4 Client Map, Format, Mount

After preparing keyring and ceph.conf on ceph-client, execute:

    rbd --id prod-rbd map prod-rbd/prod-image-01

Assume output:

    /dev/rbd0

Format:

    mkfs.xfs /dev/rbd0

Mount:

    mkdir -p /mnt/prod-rbd
    mount /dev/rbd0 /mnt/prod-rbd

Write:

    echo "ceph rbd production validation" > /mnt/prod-rbd/test.txt
    cat /mnt/prod-rbd/test.txt

Acceptance Criteria:

    RBD can be created.
    RBD can be mapped.
    File system can be formatted.
    Read/write after mounting.
    Data persistence.

---

### 7.5 RBD Snapshot & Recovery Acceptance

Write before snapshot:

    echo "before snapshot" > /mnt/prod-rbd/snap.txt
    sync

Unmount and unmap:

    umount /mnt/prod-rbd
    rbd unmap /dev/rbd0

Create snapshot:

    rbd --id prod-rbd snap create prod-rbd/prod-image-01@snap01

Remount and modify:

    rbd --id prod-rbd map prod-rbd/prod-image-01
    mount /dev/rbd0 /mnt/prod-rbd
    echo "after snapshot" > /mnt/prod-rbd/snap.txt
    sync
    umount /mnt/prod-rbd
    rbd unmap /dev/rbd0

Rollback:

    rbd --id prod-rbd snap rollback prod-rbd/prod-image-01@snap01

Verify:

    rbd --id prod-rbd map prod-rbd/prod-image-01
    mount /dev/rbd0 /mnt/prod-rbd
    cat /mnt/prod-rbd/snap.txt

Expected:

    before snapshot

High-risk warning:

    snap rollback will roll back data.
    Must stop business operations, confirm backups and recovery points before execution in production environments.

---

## VIII. Phase Three: CephFS File Storage Acceptance

### 8.1 Create CephFS Pool

If cephfs already exists, skip creation and only perform status checks.

Create metadata pool:

    ceph osd pool create prod-cephfs-metadata 32
    ceph osd pool application enable prod-cephfs-metadata cephfs
    ceph osd pool set prod-cephfs-metadata size 3
    ceph osd pool set prod-cephfs-metadata min_size 2

Create data pool:

    ceph osd pool create prod-cephfs-data 64
    ceph osd pool application enable prod-cephfs-data cephfs
    ceph osd pool set prod-cephfs-data size 3
    ceph osd pool set prod-cephfs-data min_size 2

Create file system:

    ceph fs new prod-cephfs prod-cephfs-metadata prod-cephfs-data

---

### 8.2 Deploy MDS

    ceph orch apply mds prod-cephfs --placement="2 ceph-node01 ceph-node02 ceph-node03"

Check:

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds

Acceptance Criteria:

    prod-cephfs exists.
    At least 1 active MDS.
    At least 1 standby MDS.

---

### 8.3 Create CephFS Minimal Privilege User

    ceph auth get-or-create client.prod-cephfs \
      mon 'allow r' \
      mds 'allow rw fsname=prod-cephfs' \
      osd 'allow rw tag cephfs data=prod-cephfs' \
      -o /etc/ceph/ceph.client.prod-cephfs.keyring

Check: /think

ceph auth get client.prod-cephfs

---

### 8.4 Client Mount CephFS

Extract secret:

    ceph auth get-key client.prod-cephfs > /etc/ceph/prod-cephfs.secret
    chmod 600 /etc/ceph/prod-cephfs.secret

Mount:

    mkdir -p /mnt/prod-cephfs

    mount -t ceph 10.0.0.31,10.0.0.32,10.0.0.33:/ /mnt/prod-cephfs \
      -o name=prod-cephfs,secretfile=/etc/ceph/prod-cephfs.secret,fs=prod-cephfs

Write:

    echo "cephfs production validation" > /mnt/prod-cephfs/test.txt
    cat /mnt/prod-cephfs/test.txt

---

### 8.5 Multi-Client Shared Validation

Client A writes:

    echo "write from client A" > /mnt/prod-cephfs/shared.txt

Client B reads:

    cat /mnt/prod-cephfs/shared.txt

Acceptance criteria:

    Multiple clients can mount simultaneously.
    A writes, B can read.
    CephFS status is normal.
    MDS has no abnormal alerts.

---

## Nine. Stage Four: RGW Object Storage Acceptance

### 9.1 Deploy RGW Multi-Instance

    ceph orch apply rgw prod-rgw --placement="3 ceph-node01 ceph-node02 ceph-node03" --port=7480

Check:

    ceph orch ps --daemon_type rgw
    ceph orch ls --service_type rgw

Acceptance criteria:

    Three RGW instances are running.
    Each node has port 7480 accessible.

---

### 9.2 Test RGW Port

    curl -I http://10.0.0.31:7480
    curl -I http://10.0.0.32:7480
    curl -I http://10.0.0.33:7480

A return of 200 or 403 indicates the port is responsive.

---

### 9.3 Create S3 User

    radosgw-admin user create \
      --uid="prod-s3-user" \
      --display-name="Production S3 User" \
      > /root/prod-s3-user.json

Check:

    cat /root/prod-s3-user.json | jq

Security reminder:

    This file contains AccessKey and SecretKey.
    Must chmod 600.
    Do not commit to Git.

---

### 9.4 Configure AWS CLI

On ceph-client:

    aws configure --profile prod-rgw

Input:

    AccessKey
    SecretKey
    region: us-east-1
    output: json

Set path-style:

    aws configure set profile.prod-rgw.s3.addressing_style path

Set Endpoint:

    export RGW_ENDPOINT="http://10.0.0.31:7480"

---

### 9.5 Bucket and Object Read/Write

Create Bucket:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 mb s3://prod-bucket

Upload:

    echo "rgw production validation" > /tmp/rgw-test.txt

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-test.txt s3://prod-bucket/rgw-test.txt

Check:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://prod-bucket/

Download:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp s3://prod-bucket/rgw-test.txt /tmp/rgw-test-download.txt

Verify:

    cat /tmp/rgw-test-download.txt

Acceptance criteria:

    Bucket can be created.
    Objects can be uploaded.
    Objects can be downloaded.
    Contents match.

---

### 9.6 HTTPS Unified Entry Acceptance

Recommended entry:

    https://s3.example.com

Nginx backend:

    10.0.0.31:7480
    10.0.0.32:7480
    10.0.0.33:7480

Verify:

    curl -I https://s3.example.com

AWS CLI:

    export RGW_ENDPOINT="https://s3.example.com"

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

Acceptance criteria:

    Only HTTPS is used externally.
    RGW backend HTTP is only accessible internally.
    443 certificate is valid.
    S3 access is normal.

---

## Ten. Stage Five: Kubernetes RBD CSI Acceptance

### 10.1 Ceph Side Preparation Pool

    ceph osd pool create k8s-prod-rbd 64
    ceph osd pool application enable k8s-prod-rbd rbd
    ceph osd pool set k8s-prod-rbd size 3
    ceph osd pool set k8s-prod-rbd min_size 2
    rbd pool init k8s-prod-rbd

---

### 10.2 Create CSI User /think

ceph auth get-or-create client.k8s-prod-rbd \
  mon 'profile rbd' \
  mgr 'profile rbd pool=k8s-prod-rbd' \
  osd 'profile rbd pool=k8s-prod-rbd' \
  -o /etc/ceph/ceph.client.k8s-prod-rbd.keyring

Retrieve key:

  ceph auth get-key client.k8s-prod-rbd

---

### 10.3 Kubernetes Side CSI Check

  kubectl get pods -n ceph-csi -o wide
  kubectl get csidriver

Acceptance criteria:

  rbd.csi.ceph.com exists.
  RBD CSI controller Running.
  RBD CSI node plugin Running on all nodes.

---

### 10.4 Create StorageClass

Example:

  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: ceph-rbd-prod
  provisioner: rbd.csi.ceph.com
  parameters:
    clusterID: <ceph-fsid>
    pool: k8s-prod-rbd
    imageFeatures: layering
    csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
    csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
    csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
    csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
    csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
    csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
  reclaimPolicy: Retain
  allowVolumeExpansion: true
  volumeBindingMode: Immediate

Production recommendations:

  Critical business applications should use Retain.
  Avoid accidental PVC deletion causingBottom RBD Image deletion.

---

### 10.5 Create PVC and Pod

PVC:

  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: rbd-prod-pvc
    namespace: storage-demo
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: ceph-rbd-prod
    resources:
      requests:
        storage: 5Gi

Pod:

  apiVersion: v1
  kind: Pod
  metadata:
    name: rbd-prod-pod
    namespace: storage-demo
  spec:
    containers:
      - name: app
        image: registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:latest
        command:
          - sh
          - -c
          - "while true; do sleep 3600; done"
        volumeMounts:
          - name: data
            mountPath: /data
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: rbd-prod-pvc

Verification:

  kubectl get pvc -n storage-demo
  kubectl get pod -n storage-demo -o wide

Write:

  kubectl exec -n storage-demo rbd-prod-pod -- sh -c "echo 'rbd csi validation' > /data/test.txt"

Read:

  kubectl exec -n storage-demo rbd-prod-pod -- cat /data/test.txt

Acceptance criteria:

  PVC Bound.
  PV automatically created.
  RBD Image automatically created in Ceph.
  Pod can mount.
  Data can be read and written.

---

## ElevenI don't know.Phase Six: Kubernetes CephFS CSI Acceptance

### 11.1 CephFS SubvolumeGroup

  ceph fs subvolumegroup create prod-cephfs csi

Check:

  ceph fs subvolumegroup ls prod-cephfs

---

### 11.2 Create CephFS CSI User

  ceph auth get-or-create client.k8s-prod-cephfs \
    mon 'allow r' \
    mgr 'allow rw' \
    mds 'allow rw fsname=prod-cephfs' \
    osd 'allow rw tag cephfs data=prod-cephfs' \
    -o /etc/ceph/ceph.client.k8s-prod-cephfs.keyring

Check:

  ceph auth get client.k8s-prod-cephfs

---

### 11.3 Kubernetes Side CSI Check

kubectl get pods -n ceph-csi -o wide
kubectl get csidriver

Acceptance Criteria:

    cephfs.csi.ceph.com exists.
    CephFS CSI controller Running.
    CephFS CSI node plugin Running on all nodes.

---

### 11.4 Creating CephFS StorageClass

Example:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: cephfs-prod-rwx
    provisioner: cephfs.csi.ceph.com
    parameters:
      clusterID: <ceph-fsid>
      fsName: prod-cephfs
      pool: prod-cephfs-data
      subvolumeGroup: csi
      mounter: kernel
      csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
      csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
      csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
    reclaimPolicy: Retain
    allowVolumeExpansion: true
    volumeBindingMode: Immediate

---

### 11.5 Creating RWX PVC and Two Pods

PVC:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: cephfs-prod-pvc
      namespace: cephfs-demo
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: cephfs-prod-rwx
      resources:
        requests:
          storage: 5Gi

Two Pods share the same PVC.

Verification:

    kubectl get pvc -n cephfs-demo
    kubectl get pod -n cephfs-demo -o wide

Pod A writes:

    kubectl exec -n cephfs-demo cephfs-pod-a -- sh -c "echo 'from pod A' > /data/a.txt"

Pod B reads:

    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/a.txt

Acceptance Criteria:

    PVC Bound.
    Multiple Pods can mount simultaneously.
    Pod A writes, Pod B can read.
    CephFS Subvolume automatically created.
    MDS status normal.

---

## TwelveI don't know.Stage Seven: OSD Failure Drill

### 12.1 Drill Objective

Verification:

    Cluster remains available when a single OSD fails.
    PG enters degraded / recovering state.
    PG returns to active+clean state after OSD recovery.
    Monitoring alerts trigger and recover.

---

### 12.2 Pre-Drill Checks

    ceph -s
    ceph osd tree
    ceph pg stat
    ceph osd df

Requirements:

    Cluster healthy.
    No ongoing recovery/backfill.
    Sufficient capacity.
    Not in production or maintenance window.

---

### 12.3 Set noout for short-term maintenance

For short-term drills, set:

    ceph osd set noout

Check:

    ceph osd dump | grep flags

Notes:

    noout prevents OSD from being automatically out after short-term down.
    Must unset after drill completes.

---

### 12.4 Stop a Test OSD

Check OSD:

    ceph osd tree

Stop:

    ceph orch daemon stop osd.0

Observe:

    ceph -s
    ceph osd tree
    ceph pg stat
    ceph health detail

Expected:

    osd.0 down.
    Cluster shows HEALTH_WARN.
    Possible degraded state.
    Business read/write should not be interrupted, provided replica and min_size requirements are met.

---

### 12.5 Verify Business Read/Write

RBD:

    echo "osd down test" > /mnt/prod-rbd/osd-test.txt
    cat /mnt/prod-rbd/osd-test.txt

CephFS:

    echo "osd down cephfs test" > /mnt/prod-cephfs/osd-test.txt
    cat /mnt/prod-cephfs/osd-test.txt

RGW:

    aws --profile prod-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls

Kubernetes PVC:

    kubectl exec -n storage-demo rbd-prod-pod -- cat /data/test.txt

---

### 12.6 Recover OSD

Start:

    ceph orch daemon start osd.0

Observe:

    ceph -s
    ceph osd tree
    ceph pg stat

Unset noout:

    ceph osd unset noout

Acceptance Criteria:

OSD recovery up/in.  
PG eventually recovers active+clean.  
Monitoring alerts automatically recover.  
Business data not lost.  

---

## Thirteen, Phase Eight: Node Maintenance Drill

### 13.1 Drill Objectives

Verify single-node maintenance process:

    Set noout  
    Stop or restart node  
    Observe cluster degradation  
    Recover node  
    Cancel noout  
    Verify cluster recovery  

---

### 13.2 Pre-Maintenance Checks

    ceph -s  
    ceph osd tree  
    ceph orch ps  
    ceph pg stat  

Confirm:

    Not multiple nodes are abnormal simultaneously.  
    Not during recovery/backfill peak.  
    Sufficient capacity.  
    Business party has confirmed maintenance window.  

---

### 13.3 Set noout

    ceph osd set noout  

---

### 13.4 Restart Node

Example restart ceph-node03:

    ssh root@ceph-node03  
    reboot  

Observe:

    ceph -s  
    ceph osd tree  
    ceph orch ps  

---

### 13.5 Post-Node Recovery Verification

After node recovery:

    ceph -s  
    ceph osd tree  
    ceph orch ps  
    ceph pg stat  

Cancel noout:

    ceph osd unset noout  

Acceptance Criteria:

    Node recovery.  
    The node OSD recovers.  
    Related daemon recovers.  
    PG active+clean.  
    noout has been canceled.  

---

## Fourteen, Phase Nine: MDS Failure Switch Drill

### 14.1 Drill Objectives

Verify:

    After active MDS failure, standby MDS can take over.  
    CephFS mount and read/write can recover.  
    Monitoring can detect active MDS anomalies.  

---

### 14.2 Pre-Drill Checks

    ceph fs status  
    ceph mds stat  
    ceph orch ps --daemon_type mds  

Requirements:

    At least 1 active MDS.  
    At least 1 standby MDS.  

---

### 14.3 Stop Active MDS

First find the active MDS name from ceph fs status.

Stop:

    ceph orch daemon stop <active-mds-daemon-name>  

Observe:

    ceph fs status  
    ceph mds stat  
    ceph -s  

Expected:

    Standby MDS takes over as active.  

---

### 14.4 Verify CephFS Read/Write

    echo "mds failover test" > /mnt/prod-cephfs/mds-test.txt  
    cat /mnt/prod-cephfs/mds-test.txt  

---

### 14.5 Recover MDS

    ceph orch daemon start <active-mds-daemon-name>  

Check:

    ceph fs status  
    ceph orch ps --daemon_type mds  

Acceptance Criteria:

    CephFS eventually has at least 1 active + 1 standby.  
    Business read/write can recover.  
    Alarms can trigger and recover.  

---

## Fifteen, Phase Ten: RGW High Availability Entry Drill

### 15.1 Drill Objectives

Verify:

    When a single RGW instance fails, HTTPS unified entry remains accessible.  
    Nginx / LB can automatically bypass abnormal backend.  
    S3 upload/download is not significantly affected.  

---

### 15.2 Pre-Drill Checks

    ceph orch ps --daemon_type rgw  

Test:

    aws --profile prod-rgw --endpoint-url https://s3.example.com s3 ls  

---

### 15.3 Stop One RGW Instance

Stop:

    ceph orch daemon stop <rgw-daemon-name>  

Check:

    ceph orch ps --daemon_type rgw  

---

### 15.4 Verify S3 Access

    aws --profile prod-rgw \
      --endpoint-url https://s3.example.com \
      s3 ls  

Upload:

    echo "rgw failover test" > /tmp/rgw-failover.txt  

    aws --profile prod-rgw \
      --endpoint-url https://s3.example.com \
      s3 cp /tmp/rgw-failover.txt s3://prod-bucket/rgw-failover.txt  

Download:

    aws --profile prod-rgw \
      --endpoint-url https://s3.example.com \
      s3 cp s3://prod-bucket/rgw-failover.txt /tmp/rgw-failover-download.txt  

Verify:

    cat /tmp/rgw-failover-download.txt  

---

### 15.5 Recover RGW

    ceph orch daemon start <rgw-daemon-name>  

Acceptance Criteria:

    When single RGW is down, unified entry remains available.  
    S3 upload/download is normal.  
    RGW rejoins backend after recovery.  
    Alarms can trigger and recover.  

---

## Sixteen, Phase Eleven: Backup Recovery Drill

### 16.1 RBD Backup Recovery

Export:

    rbd export prod-rbd/prod-image-01 /backup/ceph/rbd/prod-image-01.raw  

Recover to new Image:

    rbd import /backup/ceph/rbd/prod-image-01.raw prod-rbd/prod-image-01-restore  

Verify:

    rbd info prod-rbd/prod-image-01-restore  
    rbd map prod-rbd/prod-image-01-restore  
    mount /dev/rbdX /mnt/rbd-restore  
    ls -l /mnt/rbd-restore  

---

### 16.2 CephFS File Recovery

Backup: /think

# 16. CephFS Object Restore

Backup directory:

    tar czf /backup/ceph/cephfs/prod-cephfs-backup.tar.gz \
      -C /mnt/prod-cephfs .

Restore to new directory:

    mkdir -p /mnt/prod-cephfs-restore-test

    tar xzf /backup/ceph/cephfs/prod-cephfs-backup.tar.gz \
      -C /mnt/prod-cephfs-restore-test

Authentication:

    ls -l /mnt/prod-cephfs-restore-test

---

### 16.3 RGW Object Restore

Backup Bucket:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 sync s3://prod-bucket /backup/ceph/rgw/prod-bucket

Restore to New Bucket:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 mb s3://prod-bucket-restore

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 sync /backup/ceph/rgw/prod-bucket s3://prod-bucket-restore

Authentication:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls s3://prod-bucket-restore

---

## XVII. Phase XII: Monitoring and verification Take it.

### 17.1 Prometheus Indicator acceptance and acceptance

View MGR Prometheus:

    ceph mgr services

Test:

    curl -s http://10.0.0.31:9283/metrics | grep '^ceph_' | head

Prometheus Targets:

    job="ceph" Status UP

---

### 17.2 Grafana Big acceptance.

At least:

- Ceph Overview Disk
- OSD Big Disk
- Pool Big Disk
- CephFS / MDS Big Disk
- RGW Big Disk
- Kubernetes CSI Big Disk
- Node Resource Disk

---

### 17.3 Alertmanager Call the police and check it out.

At least verify:

| Police! | Whether or not to trigger | Notification | Recovery |
|---|---|---|---|
| HEALTH_WARN |  |  |  |
| OSD down |  |  |  |
| PG degraded |  |  |  |
| Capacity above threshold |  |  |  |
| MDS down |  |  |  |
| RGW down |  |  |  |
| PVC Pending |  |  |  |

---

## Phase XIII: Security baseline acceptance and inspection

### 18.1 CephX User Check

    ceph auth ls

Inspection:

    Business usage client.adminI don't know.
    RBD Is there a dedicated user?
    CephFS Is there a dedicated user?
    CSI Is there a dedicated user?
    Existence allow * Business user.

---

### 18.2 keyring Permission Check

    find /etc/ceph -type f -name "*.keyring" -exec ls -l {} \;

Acceptance criteria:

    keyring It should not be readable by ordinary users.
    admin key It should not be distributed to operational nodes.
    key Should not be submitted GitI don't know.

---

### 18.3 RGW HTTPS Inspection

    curl -I https://s3.example.com

Acceptance criteria:

    External use HTTPSI don't know.
    Backend 7480 Don't expose the public web.
    The certificate is valid.
    Nginx / LB Have access logs.

---

### 18.4 Dashboard Inspection

    ceph mgr services

Acceptance criteria:

    Dashboard Use HTTPSI don't know.
    Don't expose the public web.
    Use a strong password.
    Access is limited to the web.

---

### 18.5 Kubernetes Secret Inspection

    kubectl get secret -n ceph-csi

Inspection:

    Unreadable for ordinary users ceph-csi SecretI don't know.
    StorageClass Modify controlled.
    PVC Deletes a process.

---

## Summary of production laboratory collections

### 19.1 Cluster basic acceptance

| Checkpoint | Acceptance and inspection standards | Result |
|---|---|---|
| ceph -s | HEALTH_OK Or the police can explain. |  |
| MON | 3 individual MON quorum Normal |  |
| MGR | active + standby |  |
| OSD | All up/in |  |
| PG | active+clean |  |
| Capacity | None nearfull / full |  |
| Services | daemon All running |  |

---

### 19.2 Storage capacity acceptance

| Type | Acceptance and inspection standards | Result |
|---|---|---|
| RBD | Created,map, mounted, read and write, snapshot, restored |  |
| CephFS | Mountable, multi-client shared, snapshot restored |  |
| RGW | Bucket, object upload download,HTTPS Entry |  |
| RBD CSI | PVC BoundI don't know.Pod Mount, Data Endurance |  |
| CephFS CSI | RWX PVCA lot. Pod Shared reading and writing |  |

---

### 19.3 Transport capacity acceptance

| Capacity | Acceptance and inspection standards | Result |
|---|---|---|
| Fault check. | Yes. OSD / PG / MDS / RGW / CSI Query process |  |
| Daily inspections | Checklist and script. |  |
| Security alert. | Prometheus / Grafana / Alertmanager Available |  |
| Backup Restore | RBD / CephFS / RGW Restore Authentication |  |
| Performance baseline | Yes. rados / fio / CephFS / RGW Test Record |  |
| Security baseline | Minimum permission,HTTPSPort control, key governance |  |
| Fault Exercise | OSD / Nodes / MDS / RGW Exercise complete. |  |

---

## Integrated operational clean-up statement

### 20.1 Clearing principle

Pre-Cleanup Confirmation:

    Is it a test resource.
    Is it still in use by business operations.
    Is there a backup.
    Is there a Retain policy preventing resource retention.
    IsExercise data needed for post-mortem analysis.

---

### 20.2 Contents Not Recommended for Cleanup

Production or long-term experimental environments are not recommended for cleanup:

- Base Ceph Cluster
- Production Pool
- Production RBD Image
- Production CephFS
- Production RGW Bucket
- Production Kubernetes PVC
- Monitoring Data
- Backup Files
- Security Audit Records

---

### 20.3 Test Resources That Can Be Cleaned

Clean after confirming they are no longer needed:

- Test Pod
- Test PVC
- Test Bucket
- Test Object
- Test RBD Image
- Test Snapshot
- Test Temporary Directory
- Test Pool

High-risk warning:

    ceph osd pool rm
    rbd rm
    rbd snap purge
    ceph fs rm
    aws s3 rm --recursive
    kubectl delete pvc

All may cause data loss.

---

## Twenty-one, Comprehensive Practical Common Issues

### 21.1 RBD is Available, but Kubernetes PVC is Pending

Priority troubleshooting steps:

    kubectl describe pvc
    kubectl get storageclass
    kubectl get secret -n ceph-csi
    kubectl logs -n ceph-csi <rbd-provisioner-pod>
    ceph auth get client.k8s-prod-rbd
    ceph osd pool ls

Common causes:

- StorageClass clusterID error
- Secret error
- Pool name error
- CSI Controller anomaly
- Ceph user permissions insufficient

---

### 21.2 CephFS Client Can Mount, but CephFS CSI PVC is Pending

Priority troubleshooting steps:

    ceph fs status
    ceph fs subvolumegroup ls prod-cephfs
    kubectl describe pvc
    kubectl logs -n ceph-csi <cephfs-provisioner-pod>
    ceph auth get client.k8s-prod-cephfs

Common causes:

- fsName error
- SubvolumeGroup does not exist
- MDS not active
- Secret error
- mgr permissions insufficient

---

### 21.3 RGW Single Node is Accessible, but HTTPS Unified Entry is Abnormal

Priority troubleshooting steps:

    curl -I http://10.0.0.31:7480
    curl -I https://s3.example.com
    nginx -t
    tail -f /var/log/nginx/error.log

Common causes:

- Backend RGW unreachable
- Nginx upstream configuration error
- Certificate error
- Host header causing S3 signature failure
- proxy_request_buffering configuration unreasonable

---

### 21.4 PG Remains Unrecovered After OSD Failure Drill

Priority troubleshooting steps:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph pg stat
    ceph osd df

Common causes:

- OSD has not truly recovered
- noout / norecover flags not canceled
- Capacity insufficient
- Another OSD also abnormal
- Network or disk issues
- CRUSH failure domain insufficient

---

## Twenty-two, Advanced SRE Methodology

### 22.1 Comprehensive Practicality is Not Just Command Accumulation, but Verification Loop

Advanced SRE is not just executing:

    ceph -s
    ceph osd tree
    rbd create
    kubectl apply

But verifying:

    Deployment reliability.
    Data readability and writability.
    Fault recoverability.
    Monitoring capability to detect issues.
    Alert notification capability.
    Backup recoverability.
    Permission controllability.
    Entry security.
    Documentation completeness.

---

### 22.2 Ceph Productionization Core is Four Capabilities

Whether Ceph has production capabilities depends on four key aspects:

    Availability:
      OSD, MON, MDS, RGW have high availability design.

    Recoverability:
      RBD, CephFS, RGW have backup and recovery drills.

    Observability:
      Prometheus, Grafana, Alertmanager can detect issues.

    Governance:
      Permissions, capacity, changes, inspections, and fault drills have processes.

---

### 22.3 Do Not Equate Experimental Results Directly with Production Capabilities

Passing experiments only indicates:

    Technical pipeline can run.

Production also requires:

- Stable hardware
- Reliable network
- Capacity planning
- Monitoring and alerts
- Backup and recovery
- Security permissions
- Change processes
- On-call response
- Fault post-mortem
- Long-term maintenance capability

---

### 22.4 Advanced SRE Should Be Able to Design Drills

Drills are not to create accidents, but to validate response plans.

Good drills should have:

    Drill objectives
    Drill scope
    Risk assessment
    Rollback plan
    Execution steps
    Observation metrics
    Recovery criteria
    Post-mortem summary

---

## Twenty-three, Interview Answering Approach

If asked in an interview:

    How do you determine if a Ceph cluster is production-ready?

You can answer:

I will not only check whether Ceph can be deployed successfully, but will verify from the perspectives of cluster health, storage capacity, fault recovery, monitoring alerts, backup recovery, security permissions, and operations processes.
First, check ceph -s, ceph health detail, ceph osd tree, ceph pg stat to confirm MON quorum, MGR, OSD up/in, PG active+clean, and capacity water level are normal. Then validate the three capabilities of RBD, CephFS, and RGW: RBD can create Image, map, mount, snapshots, and recovery; CephFS can support multi-client shared read/write, with active and standby MDS; RGW can upload/download via S3 API and access through HTTPS unified entry.
If integrated with Kubernetes, also verify RBD CSI and CephFS CSI to confirm PVC can be Bound, Pod can mount, RBD suitable for RWO, CephFS suitable for RWX.
Then conduct fault drills, such as single OSD down, node maintenance, MDS fault switch, RGW single instance failure, observe whether business is available, whether PG can recover, and whether alerts are triggered.
Finally check production governance capabilities, including Prometheus/Grafana/Alertmanager monitoring alerts, RBD/CephFS/RGW backup recovery, client.admin not issuing business, RGW HTTPS, Dashboard access restrictions, key and Secret management, daily inspection and fault review processes.
Only when all these closed-loop capabilities are in place will I consider the Ceph cluster to have basic production readiness.

---

## Twenty-Four, Summary of This Section

This article completes the comprehensive practical closed-loop for the Ceph module:

1. Ceph production acceptance cannot focus solely on whether the cluster can start.
2. Basic health requires checking MON, MGR, OSD, PG, Pool, capacity, and service status.
3. RBD needs to verify creation, map, mount, read/write, snapshots, and recovery.
4. CephFS needs to verify MDS high availability, multi-client sharing, and snapshot recovery.
5. RGW needs to verify S3 users, Bucket, object read/write, and HTTPS unified entry.
6. Kubernetes needs to verify RBD CSI and CephFS CSI.
7. OSD fault drills can verify replication and recovery capabilities.
8. Node maintenance drills can verify noout usage and recovery process.
9. MDS fault drills can verify CephFS high availability.
10. RGW fault drills can verify object storage entry high availability.
11. Backup recovery drills must cover RBD, CephFS, and RGW.
12. Monitoring alerts must verify trigger, notification, and recovery.
13. Security baseline must cover minimal permissions, HTTPS, port, and key governance.
14. Advanced SRE's core capability is not just command knowledge, but establishing complete closed-loop for deployment, usage, monitoring, fault, recovery, security, and review.
15. At this point, the Ceph module has formed a complete advanced SRE learning path covering theory, deployment, operations, fault, performance, security, monitoring, and comprehensive practicals.

---

## Twenty-Five, Reference Documents

Ceph official documentation:

    https://docs.ceph.com/en/latest/

Cephadm documentation:

    https://docs.ceph.com/en/latest/cephadm/

Ceph RADOS operations:

    https://docs.ceph.com/en/latest/rados/operations/

Ceph RBD documentation:

    https://docs.ceph.com/en/latest/rbd/

CephFS documentation:

    https://docs.ceph.com/en/latest/cephfs/

Ceph RGW documentation:

    https://docs.ceph.com/en/latest/radosgw/

Ceph monitoring documentation:

    https://docs.ceph.com/en/latest/rados/operations/monitoring/

Ceph health check:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

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