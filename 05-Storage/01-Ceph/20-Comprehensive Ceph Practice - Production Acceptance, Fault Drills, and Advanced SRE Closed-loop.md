# Comprehensive Ceph Practice: Production Acceptance, Fault Drills, and Advanced SRE Closed-loop

Recommended Path: 05-Storage/01-Ceph/20-Ceph Comprehensive Practice: Production Acceptance, Fault Drills, and Advanced SRE Closed-loop.md

Tags: #Ceph #Comprehensive Practice #Production Acceptance #Fault Drills #RBD #CephFS #RGW #CSI #Monitoring & Alerts #Backup and Recovery #Advanced SRE

---

## I. Document Overview

This article is the twentieth in the series on advanced Ceph SRE storage modules and also serves as a comprehensive practice guide for the Ceph module.

Previous topics covered include:

- Understanding Ceph infrastructure and components
- Deploying cephadm clusters
- Managing OSDs
- Pools and PGs
- CRUSH fault domains
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Daily operations and maintenance
- Fault troubleshooting
- Backup and recovery
- Performance optimization
- Security enhancements
- Monitoring and alerts

This article does not focus on any single component but instead integrates the previously learned skills to form a comprehensive advanced SRE experiment that closely resembles a production environment.

The objectives are to verify:

    Whether the Ceph cluster can be deployed successfully.
    Whether Ceph storage services are functional.
    Whether Ceph data can be restored reliably.
    Whether Ceph services can be monitored effectively.
    Whether fault scenarios can be simulated and resolved.
    Whether seamless integration with Kubernetes is possible.
    Whether Ceph meets the requirements for production deployment.

---

## II. Comprehensive Practice Objectives

After completing this article, you should be able to:

1. Design a Ceph + Kubernetes external storage experiment topology.
2. Conduct a production acceptance check for the Ceph cluster.
3. Set up three types of storage services: RBD, CephFS, and RGW.
4. Verify the read/write, snapshotting, and recovery capabilities of RBD block storage.
5. Verify the read/write, snapshotting, and recovery capabilities of CephFS shared file systems.
6. Confirm that RGW supports S3 Bucket operations, object upload/download, and HTTPS access.
7. Verify the dynamic volume provisioning provided by Kubernetes RBD CSI.
8. Verify the ReadWriteMany sharing feature of Kubernetes CephFS CSI.
9. Execute a single OSD fault drill.
10. Conduct a single-node maintenance drill.
11. Perform an MDS failover experiment.
12. Test the multi-instance access functionality of RGW.
13. Verify the monitoring and alerting capabilities using Prometheus, Grafana, and Alertmanager.
14. Ensure the backup and recovery process functions correctly.
15. Generate a final production acceptance report.
16. Establish an advanced SRE closed-loop for Ceph operations.

---

## III. Experiment Topology

### 3.1 Ceph Cluster Nodes

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / MDS / RGW |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / MDS / RGW |
| 10.0.0.33 | ceph-node03 | MON / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Drills (optional) |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client |
| 10.0.0.36 | ceph-entry | Nginx / Unified RGW HTTPS Access / Monitoring Node (optional) |

Operating System:

    Ubuntu Server 22.04.5 LTS

Alternative:

    Rocky Linux 9 can also be used as an alternative experimental system.

---

### 3.2 Kubernetes Cluster Nodes

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Running Environment:

    kubeadm
    containerd
    Calico
    Ubuntu Server 22.04.5 LTS

---

### 3.3 Network Relationships

All nodes are located within the same IP range:

    10.0.0.0/24

Key network connections for Ceph:

    Kubernetes Node -> Ceph MON: 3300 / 6789
    Kubernetes Node -> Ceph OSD:### 6.2 Viewing Health Details

    ceph health detail

If there are any alerts, they must be recorded:

| Alert | Cause | Affects Business? | Action Taken |
|---|---|---|---|
|  |  |  |  |

---

### 6.3 Checking OSD Status

    ceph osd tree
    ceph osd stat
    ceph osd df
    ceph osd perf

Acceptance criteria:

    All OSDs are up/in.
    There is no significant unevenness in OSD usage.
    No single-point exceptions in OSD latency.
    The distribution of OSDs meets the requirements of the host failure domain.

---

### 6.4 Checking PG Status

    ceph pg stat

Acceptance criteria:

    All PGs are active+clean.
    None are degraded, undersized, inactive, incomplete, inconsistent, or stale.

---

### 6.5 Checking Service Status

    ceph orch ps
    ceph orch ls

Acceptance criteria:

    The mon, mgr, osd, mds, and rgw services are all running.
    No daemons are in the error, stopped, or unknown state.

---

## Section Seven: Phase Two: RBD Block Storage Verification

### 7.1 Creating an RBD Pool

    ceph osd pool create prod-rbd 64
    ceph osd pool application enable prod-rbd rbd
    ceph osd pool set prod-rbd size 3
    ceph osd pool set prod-rbd min_size 2
    rbd pool init prod-rbd

Verify:

    ceph osd pool get prod-rbd all
    ceph osd pool application get prod-rbd

---

### 7.2 Creating a User with Minimized Permissions

    ceph auth get-or-create client.prod-rbd \
      mon 'profile rbd' \
      mgr 'profile rbd pool=prod-rbd' \
      osd 'profile rbd pool=prod-rbd' \
      -o /etc/ceph/ceph.client.prod-rbd.keyring

Verify:

    ceph auth get client.prod-rbd

---

### 7.3 Creating an RBD Image

    rbd --id prod-rbd create prod-image-01 --size 10G -p prod-rbd --image-feature layering

Verify:

    rbd --id prod-rbd info prod-rbd/prod-image-01

---

### 7.4 Mapping, Formatting, and Mounting on the Client

After preparing the keyring and ceph.conf on the ceph-client, execute the following commands:

    rbd --id prod-rbd map prod-rbd/prod-image-01

Expected output:

    /dev/rbd0

Format the volume:

    mkfs.xfs /dev/rbd0

Mount the volume:

    mkdir -p /mnt/prod-rbd
    mount /dev/rbd0 /mnt/prod-rbd

Write data:

    echo "ceph rbd production validation" > /mnt/prod-rbd/test.txt
    cat /mnt/prod-rbd/test.txt

Acceptance criteria:

    The RBD volume can be created.
    It can be mapped to the client.
    The file system can be formatted.
    Data can be written and read after mounting.
    The data is persistent.

---

### 7.5 Verifying RBD Snapshots and Recovery

Before creating a snapshot, write data:

    echo "before snapshot" > /mnt/prod-rbd/snap.txt
    sync

Unmount and unmap the volume:

    umount /mnt/prod-rbd
    rbd unmap /dev/rbd0

Create a snapshot:

    rbd --id prod-rbd snap create prod-rbd/prod-image-01@snap01

Re-mount the volume and modify it:

    rbd --id prod-rbd map prod-rbd/prod-image-01
    mount /dev/rbd0 /mnt/prod-rbd
    echo "after snapshot" > /mnt/prod-rbd/snap.txt
    sync
    umount /mnt/prod-rbd
    rbd unmap /dev/rbd0

Roll back the changes:

    rbd --id prod-rbd snap rollback prod-rbd/prod-image-01@snap01

Verify the results:

    rbd --id prod-rbd map prod-rbd/prod-image-01
    mount /dev/rbd0 /mnt/prod-rbd
    cat /mnt/prod-rbd/snap.txt

Expected result:

    The content of "before snapshot" file should be displayed.

High-risk note:

    Snap rollback will restore the data to its previous state.
    This must be done in a production environment only after ensuring that services have been stopped, backups have been made, and recovery plans```markdown
ceph orch apply rgw prod-rgw --placement="3 ceph-node01 ceph-node02 ceph-node03" --port=7480

Check:

    ceph orch ps --daemon_type rgw
    ceph orch ls --service_type rgw

Acceptance criteria:

    Three RGW instances are running.
    The port 7480 is accessible on each node.

---

### 9.2 Testing the RGW Port

    curl -I http://10.0.0.31:7480
    curl -I http://10.0.0.32:7480
    curl -I http://10.0.0.33:7480

A return of 200 or 403 indicates that the port is responding.

---

### 9.3 Creating an S3 User

    radosgw-admin user create \
      --uid="prod-s3-user" \
      --display-name="Production S3 User" \
      > /root/prod-s3-user.json

Check:

    cat /root/prod-s3-user.json | jq

Security note:

    This file contains the AccessKey and SecretKey.
    Make sure to set its permission to 600.
    Do not commit it to Git.

---

### 9.4 Configuring AWS CLI

On the ceph-client:

    aws configure --profile prod-rgw

Enter:

    AccessKey
    SecretKey
    region: us-east-1
    output: json

Set the path-style:

    aws configure set profile.prod-rgw.s3.addressing_style path

Set the Endpoint:

    export RGW_ENDPOINT="http://10.0.0.31:7480"

---

### 9.5 Reading and Writing Buckets and Objects

Create a Bucket:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_endpoint} \
      s3 mb s3://prod-bucket

Upload a file:

    echo "rgw production validation" > /tmp/rgw-test.txt

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-test.txt s3://prod-bucket/rgw-test.txt

Check the content:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_endpoint} \
      s3 ls s3://prod-bucket/

Download the file:

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp s3://prod-bucket/rgw-test.txt /tmp/rgw-test-download.txt

Verify the content:

    cat /tmp/rgw-test-download.txt

Acceptance criteria:

    The Bucket can be created.
    Objects can be uploaded and downloaded.
    The content is consistent.

---

### 9.6 Acceptance of HTTPS Unified Entrance

Recommended entrance:

    https://s3.example.com

Nginx backend ports:

    10.0.0.31:7480
    10.0.0.32:7480
    10.0.0.33:7480

Verify:

    curl -I https://s3.example.com

With AWS CLI:

    export RGW_ENDPOINT="https://s3.example.com"

    aws --profile prod-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 ls

Acceptance criteria:

    Only HTTPS is used for external access.
    The RGW backend HTTP is accessible only from the internal network.
    The 443 certificate is valid.
    S3 access is functioning properly.

---

## Section Ten: Phase Five: Kubernetes RBD CSI Acceptance

### 10.1 Preparing the Pool on the Ceph Side

    ceph osd pool create k8s-prod-rbd 64
    ceph osd pool application enable k8s-prod-rbd rbd
    ceph osd pool set k8s-prod-rbd size 3
    ceph osd pool set k8s-prod-rbd min_size 2
    rbd pool init k8s-prod-rbd

---

### 10.2 Creating a CSI User

    ceph auth get-or-create client.k8s-prod-rbd \
      mon 'profile rbd' \
      mgr 'profile rbd pool=k8s-prod-rbd' \
      osd 'profile rbd pool=k8s-prod-rbd' \
      -o /etc/ceph/ceph.client.k8s-prod-rbd.keyring

Retrieve the key:

    ceph auth get-key client.k8s-prod-rbd

---

### claimName: rbd-prod-pvc

Verification:

    kubectl get pvc -n storage-demo
    kubectl get pod -n storage-demo -o wide

Writing:

    kubectl exec -n storage-demo rbd-prod-pod -- sh -c "echo 'rbd csi validation' > /data/test.txt"

Reading:

    kubectl exec -n storage-demo rbd-prod-pod -- cat /data/test.txt

Acceptance Criteria:

    PVC Bound.
    PV Automatically Created.
    RBD Image Automatically Created in Ceph.
    Pod Can Be Mounted.
    Data Is Readable and Writable.

---

## Section Eleven: Phase Six: Kubernetes CephFS CSI Verification

### 11.1 CephFS SubvolumeGroup

    ceph fs subvolumegroup create prod-cephfs csi

Verification:

    ceph fs subvolumegroup ls prod-cephfs

---

### 11.2 Creating a CephFS CSI User

    ceph auth get-or-create client.k8s-prod-cephfs \
      mon 'allow r' \
      mgr 'allow rw' \
      mds 'allow rw fsname=prod-cephfs' \
      osd 'allow rw tag cephfs data=prod-cephfs' \
      -o /etc/ceph/ceph.client.k8s-prod-cephfs.keyring

Verification:

    ceph auth get client.k8s-prod-cephfs

---

### 11.3 Kubernetes-side Check for CSI

    kubectl get pods -n ceph-csi -o wide
    kubectl get csidriver

Acceptance Criteria:

    cephfs.csi.ceph.com Exists.
    CephFS CSI Controller Is Running.
    CephFS CSI Node Plugins Are Running on All Nodes.

---

### 11.4 Creating a CephFS StorageClass

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
      csi.storage.k8s.io-controller-expand-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
      csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
      csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
    reclaimPolicy: Retain
    allowVolumeExpansion: true
    volumeBindingMode: Immediate

---

### 11.5 Creating an RWX PVC and Two Pods

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

Two Pods Share the Same PVC.

Verification:

    kubectl get pvc -n cephfs-demo
    kubectl get pod -n cephfs-demo -o wide

Writing to Pod A:

    kubectl exec -n cephfs-demo cephfs-pod-a -- sh -c "echo 'from pod A' > /data/a.txt"

Reading from Pod B:

    kubectl exec -n cephfs-demo cephfs-pod-b -- cat /data/a.txt

Acceptance Criteria:

    PVC Is Bound.
    Multiple Pods Can Be Mounted Simultaneously.
    Writing to Pod A Allows Reading from Pod B.
    CephFS Subvolume Is Automatically Created.
    MDS Status Is Normal.

---

## Section Twelve: Phase Seven: OSD Fault Experiment

### 12.1 Experiment Objectives

Verification:

    When a single OSD Fails, the cluster remains available.
    The PG Enters a Degraded/Recovering State.
    After the OSD Is Restored, the PG Returns to an Active+Clean State.
    Monitoring Alarms Are Triggered and Resolved.

---

### 12.2 Pre-experiment Checks

    ceph -s
    ceph osd tree
    ceph pg stat
    ceph osd df

Requirements:

    The cluster is healthy.
    No recovery/backfill operations are in progress.
    There is sufficient capacity.
    It is not during a production or maintenance window### 13.5 Verification After Node Recovery

After the node is recovered:

    ceph -s
    ceph osd tree
    ceph orch ps
    ceph pg stat

Cancel noout:

    ceph osd unset noout

Acceptance criteria:

    The node has been recovered.
    The OSD of this node has been restored.
    Related daemons have been recovered.
    PG is in the active+clean state.
    Noout has been canceled.

---

## Section Fourteen: Phase Nine: MDS Failover Drill

### 14.1 Drill Objectives

Verify that:

    After the active MDS fails, the standby MDS can take over.
    CephFS mounting and read/write operations can be restored.
    Monitoring can detect abnormalities in the active MDS.

---

### 14.2 Pre-Drill Checks

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds

Requirements:

    At least 1 active MDS.
    At least 1 standby MDS.

---

### 14.3 Stop the Active MDS

First, identify the name of the active MDS using `ceph fs status`.

Stop it:

    ceph orch daemon stop <active-mds-daemon-name>

Check:

    ceph fs status
    ceph mds stat
    ceph -s

Expected outcome:

    The standby MDS should have taken over as active.

---

### 14.4 Verify CephFS Read/Write Operations

    echo "mds failover test" > /mnt/prod-cephfs/mds-test.txt
    cat /mnt/prod-cephfs/mds-test.txt

---

### 14.5 Restore the MDS

    ceph orch daemon start <active-mds-daemon-name>

Check:

    ceph fs status
    ceph orch ps --daemon_type mds

Acceptance criteria:

    CephFS should have at least 1 active and 1 standby MDS.
    Business read/write operations should be restored.
    Alarms should be triggered and resolved correctly.

---

## Section Fifteen: Phase Ten: RGW High Availability Drill

### 15.1 Drill Objectives

Verify that:

    When a single RGW instance fails, the HTTPS unified access point remains accessible.
    Nginx/LB can automatically route requests to healthy backend instances.
    S3 upload and download operations are not significantly affected.

---

### 15.2 Pre-Drill Checks

    ceph orch ps --daemon_type rgw

Test:

    aws --profile prod-rgw --endpoint-url https://s3.example.com s3 ls

---

### 15.3 Stop One RGW Instance

Stop it:

    ceph orch daemon stop <rgw-daemon-name>

Check:

    ceph orch ps --daemon_type rgw

---

### 15.4 Verify S3 Access

    aws --profile prod-rgw \
      --endpoint-url https://s3.example.com \
      s3 ls

Upload a file:

    echo "rgw failover test" > /tmp/rgw-failover.txt

    aws --profile prod-rgw \
      --endpoint-url https://s3.example.com \
      s3 cp /tmp/rgw-failover.txt s3://prod-bucket/rgw-failover.txt

Download the file:

    aws --profile prod-rgw \
      --endpoint-url https://s3.example.com \
      s3 cp s3://prod-bucket/rgw-failover.txt /tmp/rgw-failover-download.txt

Verify the download:

    cat /tmp/rgw-failover-download.txt

---

### 15.5 Restore the RGW Instance

    ceph orch daemon start <rgw-daemon-name>

Acceptance criteria:

    When a single RGW instance is down, the unified access point remains functional.
    S3 upload and download operations return to normal.
    The restored RGW instance rejoin(s) the backend cluster.
    Alarms are triggered and resolved correctly.

---

## Section Sixteen: Phase Eleven: Backup and Recovery Drill

### 16.1 RBD Backup and Recovery

Export the volume:

    rbd export prod-rbd/prod-image-01 /backup/ceph/rbd/prod-image-01.raw

Restore it to a new image:

    rbd import /backup/ceph/rbd/prod-image-01.raw prod-rbd/prod-image-01-restore

Verify:

    rbd info prod-rbd/prod-image-01-restore
    rbd map prod-rbd/prod-image-01-restore
    mount /dev/rbdX /mnt/rbd-restore
    ls -l /mnt/rbd-restore

---

### 16.2--- 

### Access Restricted to Only Operations Network Segments

---

### 18.5 Checking Kubernetes Secrets

    Run `kubectl get secret -n ceph-csi`.

Verifications:

    Ordinary users should not be able to read the `ceph-csi` Secret.
    The modification of `StorageClass` is controlled.
    There are procedures in place for deleting `PVCs`.

---

## Table of Production Acceptance Items

### 19.1 Basic Cluster Acceptance

| Inspection Item | Acceptance Standard | Result |
|-----------------|--------------------|---------|
| ceph -s            | HEALTH_OK or explainable alarms |         |
| MON                | Normal quorum of 3 MON nodes |         |
| MGR                | Active + standby configuration |         |
| OSD                 | All OSDs are up/in operation |         |
| PG                  | Active + clean status             |         |
| Capacity            | No nearfull/full conditions     |         |
| Services           | All daemons are running          |         |

---

### 19.2 Storage Capability Acceptance

| Type                | Acceptance Standard                        | Result       |
|-------------------|-----------------------------------------------|--------------|
| RBD                 | Can be created, mapped, mounted, read/written, backed up, and restored | Passed        |
| CephFS              | Can be mounted, shared among multiple clients, and snapshots can be restored | Passed        |
| RGW                  | Buckets and objects can be uploaded/downloaded via HTTPS | Passed        |
| RBD CSI             | PVCs can be bound, Pods can be mounted, and data is persistent      | Passed        |
| CephFS CSI             | RWX PVCs are available, and multiple Pods can share read/write access | Passed        |

---

### 19.3 Operational Capability Acceptance

| Capability            | Acceptance Standard                         | Result       |
|-------------------|-----------------------------------------------|--------------|
| Fault Troubleshooting | Procedures for troubleshooting OSDs/PGs/MDS/RGWs/CSI are in place | Passed        |
| Regular Inspections    | Inspection checklists and scripts are available         | Passed        |
| Monitoring and Alerts   | Prometheus/Grafana/Alertmanager tools are functional       | Passed        |
| Backup and Recovery     | RBD/CephFS/RGW can be successfully restored      | Passed        |
| Performance Baselines  | Test records for rados/fio/CephFS/RGW are available    | Passed        |
| Security Baselines     | Minimum permissions are enforced, HTTPS is used, ports are securely managed, and keys are properly governed | Passed        |
| Fault Drills         | Drills involving OSDsNodes/MDS/RGWs have been conducted  | Passed        |

---

## Chapter 20: Guidelines for Cleaning Up After Comprehensive Tests

### 20.1 Cleaning Principles

Before cleaning up, confirm:

- Whether it is a test resource.
- Whether it is still in use by business operations.
- Whether there are any backups.
- Whether any retention policies are preventing its deletion.
- Whether the data needs to be retained for post-test analysis.

---

### 20.2 Contents Not Recommended for Cleaning Up

Contents in production or long-term experimental environments should not be cleared:

- Basic Ceph clusters
- Production pools
- Production RBD images
- Production CephFS instances
- Production RGW buckets
- Production Kubernetes PVCs
- Monitoring data
- Backup files
- Security audit records

---

### 20.3 Test Resources That Can Be Cleared

Resources that are confirmed to be no longer needed can be cleared:

- Test Pods
- Test PVCs
- Test buckets
- Test objects
- Test RBD images
- Test snapshots
- Temporary test directories
- Test pools

High-risk warnings:

- Commands like `ceph osd pool rm`, `rbd rm`, `rbd snap purge`, `ceph fs rm`, `aws s3 rm --recursive`, and `kubectl delete pvc` can potentially result in data loss.

---

## Chapter 21: Common Issues in Comprehensive Tests

### 21.1 RBD Is Available, but the Kubernetes PVC Remains Pending

Priority troubleshooting steps:

- Run `kubectl describe pvc`.
- Check `kubectl get storageclass`.
- Verify `kubectl get secret -n ceph-csi`.
- Examine `kubectl logs -n ceph-csi <rbd-provisioner-pod>`.
- Retrieve `ceph auth get client.k8s-prod-rbd`.
- List all OSD pools using `ceph osd pool ls`.

Common causes:

- Incorrect `StorageClass` clusterID.
- Errors with the Secret configuration.
- Wrong Pool name.
- Issues with the CSI Controller.
- Insufficient permissions for the Ceph user.

---

### 21.2 The CephFS Client Can Be Mounted, but the CephFS CSI PVC Remains Pending

Priority troubleshooting steps:

8. Node maintenance drills can verify the usage and recovery processes of noout.
9. MDS failure drills can confirm the high availability of CephFS.
10. RGW failure drills can test the high availability of object storage access points.
11. Backup and recovery drills must cover RBD, CephFS, and RGW.
12. Monitoring and alert systems must be verified for their ability to trigger alerts, notify relevant parties, and facilitate recovery processes.
13. Security best practices must include minimum permission settings, HTTPS encryption, proper port management, and secure key governance.
14. The core capability of an advanced SRE professional is not merely knowing how to execute commands but rather being able to establish a comprehensive cycle that encompasses deployment, usage, monitoring, troubleshooting, recovery, security measures, and post-event analysis.
15. To date, the Ceph module has established a complete advanced SRE learning path that covers theory, deployment, operations, troubleshooting, performance optimization, security practices, monitoring, and practical applications.

---

## Section 25: Reference Documents

Ceph Official Documentation:

    https://docs.ceph.com/en/latest/

Cephadm Documentation:

    https://docs.ceph.com/en/latest/cephADM/

Ceph RADOS Operations Documentation:

    https://docs.ceph.com/en/latest/rados/operations/

Ceph RBD Documentation:

    https://docs.ceph.com/en/latest/rbd/

CephFS Documentation:

    https://docs.ceph.com/en/latest/cephfs/

Ceph RGW Documentation:

    https://docs.ceph.com/en/latest/radosgw/

Ceph Monitoring Documentation:

    https://docs.ceph.com/en/latest/rados/operations/monitoring/

Ceph Health Checks:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Dashboard:

    https://docs.ceph.com/en/latest/mgr/dashboard/

Ceph Prometheus Module Documentation:

    https://docs.ceph.com/en/latest/mgr/prometheus/

Ceph CSI Project Documentation:

    https://github.com/ceph/ceph-csi

Kubernetes StorageClass Documentation:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Persistent Volumes Documentation:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/