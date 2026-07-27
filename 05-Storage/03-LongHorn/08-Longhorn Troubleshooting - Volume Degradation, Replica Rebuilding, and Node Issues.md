# Longhorn Troubleshooting: Volume Degradation, Replica Rebuilding, and Node Issues

Recommended Path: 05-Storage/03-LongHorn/08-Longhorn Troubleshooting: Volume Degradation, Replica Rebuilding, and Node Issues.md

Tags: #Longhorn #Troubleshooting #VolumeDegradation #ReplicaRebuild #PVC #PV #CSI #FailedMount #iSCSI #Kubernetes #Block Storage #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the eighth in the Longhorn series, focusing on troubleshooting methods for Longhorn in production operations.

Previous topics covered:

- Longhorn Basics: Kubernetes-based Cloud Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methods: Charts, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practices: PVC, PV, Pod Mounting, and Data Persistence
- Longhorn Replication Mechanisms: Number of Replicas, Node Distribution, and Data High Availability
- Longhorn Backup and Recovery: Backup Targets, Snapshots, and Volume Restoration

This article delves into practical troubleshooting for Longhorn, addressing issues such as:

- How to diagnose PVC pending status?
- How to troubleshoot Pod container creation delays?
- How to resolve FailedMount errors?
- How to handle FailedAttachVolume problems?
- How to investigate Multi-Attachment scenarios?
- How to detect Volume Degradation?
- How to address Replica rebuilding failures?
- How to troubleshoot failed Replica scheduling?
- How to deal with volume faults?
- What impact does a NotReady node have on Longhorn?
- How does DiskPressure affect Longhorn performance?
- Why do iscsid errors cause mounting failures?
- How to analyze longhorn-manager logs?
- How can Longhorn CRDs be used for troubleshooting?
- Under what circumstances should Replica/Volumes not be deleted arbitrarily?
- How to establish a production-longhorn troubleshooting process?

This article emphasizes practical steps, organizing content around commands, observed phenomena, diagnostic approaches, and solutions.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the layered approach to Longhorn troubleshooting.
2. Diagnose PVC pending issues effectively.
3. Troubleshoot Pod container creation delays.
4. Resolve FailedMount errors.
5. Handle FailedAttachVolume problems.
6. Investigate Multi-Attachment scenarios involving RWO volumes.
7. Detect Volume Degradation.
8. Monitor Replica rebuilding processes.
9. Address failed Replica scheduling issues.
10. Understand the impact of NotReady nodes on volumes.
11. Analyze DiskPressure and disk capacity-related issues.
12. Identify and resolve iscsid/open-iscsi-related problems.
13. How to view logs from Longhorn Manager, CSI, and Instance Manager.
14. How to locate fault sources based on PVC, PV, Volume, and Replica relationships.
15. Create a template for recording production troubleshooting incidents.
16. Recognize which operations are high-risk and should not be performed arbitrarily in production.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

- Kubernetes: Kubeadm cluster
- Operating system: Ubuntu Server 22.04.5 LTS
- Container runtime: containerd
- CNI: Calico
- Node IP range: 10.0.0.0/24

Node configuration:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 3.2 Longhorn Environment

Longhorn namespace:

- longhorn-system

Longhorn StorageClass:

- longhorn

Longhorn data directory:

- /data/longhorn

Experimental namespace:

- longhorn-troubleshoot-demo

---

### 3.3 Experimental Scenario Description

This article includes simulated fault scenarios, such as:

- A faulty StorageClass causing PVC pending status
- Failed Replica scheduling due to insufficient nodes
- Observing Volume Degradation by stopping kubelet or disabling Worker nodes
- Investigating Multi-Attachment risks during cross-node Pod mounting

High-risk warnings:

- Stopping kubelet, disabling nodes, deleting PVCs, volumes, or Replicas are high-risk operations.
- These actions should only be performed in the experimental environment.
- In a production environment, carefully assess business impact, perform backups, ensure maintenance window availability, and have a rollback plan in place.
- The primary principle of production troubleshooting is to preserve the current state and avoid arbitrary resource deletions.

---

##```bash
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system get replicas.longhorn.io -o wide
kubectl -n longhorn-system get engines.longhorn.io -o wide
kubectl -n longhorn-system get instancemanagers.longhorn.io
kubectl -n longhorn-system get nodes.longhorn.io

Log Monitoring:

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200
kubectl -n longhorn-system logs <csi-pod-name> --tail=100
kubectl -n longhorn-system logs <instance-manager-pod-name> --tail=100
journalctl -u kubelet --since "30 minutes ago" | tail -100
journalctl -u containerd --since "30 minutes ago" | tail -100

Node Monitoring:

systemctl status kubelet
systemctl status containerd
systemctl status iscsid
iscsiadm --version
df -hT
lsblk -f
dmesg | tail -100
journalctl -k --since "1 hour ago" | tail -100
---

## Section 5: Baseline Checks Before Faults

### 5.1 Checking Longhorn Components

Execute:

kubectl -n longhorn-system get pods -o wide

Check for any anomalies:

kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If abnormalities are found, further investigate by:

kubectl -n longhorn-system describe pod <pod-name>
kubectl -n longhorn-system logs <pod-name> --tail=100
kubectl get events -n longhorn-system --sort-by=.lastTimestamp | tail -100
---

### 5.2 Checking StorageClass

Execute:

kubectl get sc
kubectl describe sc longhorn

Pay attention to the following fields:

Provisioner
ReclaimPolicy
VolumeBindingMode
AllowVolumeExpansion
Parameters
numberOfReplicas
---

### 5.3 Checking Longhorn Volumes

Execute:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

If volumes exist, check for any issues such as:

Degraded
Faulted
Detached
Unknown
Rebuilding (if it takes too long)
---

### 5.4 Checking Longhorn Replicas

Execute:

kubectl -n longhorn-system get replicas.longhorn.io -o wide

Focus on the following fields:

running
stopped
rebuilding
error
failed
failed at
node id
disk id
---

### 5.5 Checking Longhorn Nodes and Disks

Execute:

kubectl -n longhorn-system get nodes.longhorn.io

View detailed information for specific nodes:

kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01
kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker02

Check the following fields:

Allow Scheduling
Conditions
Disk Path
Storage Maximum
Storage Available
Storage Scheduled
Disk Status
---

### 5.6 Checking Kubernetes Nodes

Execute:

kubectl get nodes -o wide

Check for any signs of high load:

kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"

Ensure that:

Ready=True
DiskPressure=False
MemoryPressure=False
PIDPressure=False
---

### 5.7 Checking Node Dependencies

On each Worker node, execute the following commands:

systemctl is-active iscsid
iscsiadm --version
systemctl status kubelet --no-pager
systemctl status containerd --no-pager
df -hT /data/longhorn
lsblk -f
---

## Section 6: Practical Exercise 1: Creating Troubleshooting Experiment Resources

### 6.1 Creating a Namespace

Execute:

kubectl create namespace longhorn-troubleshoot-demo

Verify creation:

kubectl get ns longhorn-troubleshoot-demo
---

### 6.2 Creating a Working Directory

Execute:

mkdir -p ~/longhorn-troubleshoot-demo
cd ~/longhorn-troubleshoot-demo
---

### 6.3 Creating a PVC

Create the YAML file:

cat > 01-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: troubleshoot-pvc
  namespace: longhorn-troubleshoot-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
EOF

Apply the configuration:

kubectl apply -f 01-pvc.yaml

Verify creation:

kubectl get pvc -n longhorn-troubleshoot-demo
kubectl get pv
---

### 6.4 Creating a Pod

Create the YAML file:

cat > 02-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: troubleshoot-p```markdown
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "mkdir -p /data/app/logs /data/db"
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "echo 'id,name' > /data/db/users.csv"
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "echo '1,alice' >> /data/db/users.csv"

View:

kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- find /data -maxdepth 4 -type f -print
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- cat /data/hello.txt
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- cat /data/db/users.csv

---

### 6.6 Record the Volume Name

View the PVC:

kubectl describe pvc troubleshoot-pvc -n longhorn-troubleshoot-demo

Record:

Volume: pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

Set a variable:

export LH_VOLUME=<replace with the actual Longhorn Volume name>

For example:

export LH_volume=pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

View the Longhorn Volume:

kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}

---

## VII. Practical Exercise 2: Create a Troubleshooting Baseline Report

### 7.1 Create a Baseline Collection Script

Create:

cat > collect-longhorn-baseline.sh <<'EOF'
#!/bin/bash

set -euo pipefail

NS="longhorn-troubleshoot-demo"
LH_NS="longhorn-system"
REPORT="longhorn-baseline-$(date +%F-%H%M%S).log"

{
  echo "===== Longhorn Baseline Report ====="
  date
  echo

  echo "===== Kubernetes Nodes ====="
  kubectl get nodes -o wide || true
  echo

  echo "===== StorageClass ====="
  kubectl get sc || true
  echo

  echo "===== PVC ====="
  kubectl get pvc -n ${NS} -o wide || true
  echo

  echo "===== PV ====="
  kubectl get pv || true
  echo

  echo "===== Pods ====="
  kubectl get pod -n ${NS} -o wide || true
  echo

  echo "===== Longhorn Pods ====="
  kubectl -n ${LH_NS} get pods -o wide || true
  echo

  echo "===== Longhorn Volumes ====="
  kubectl -n ${LH_NS} get volumes.longhorn.io -o wide || true
  echo

  echo "===== Longhorn Replicas ====="
  kubectl -n ${LH_NS} get replicas.longhorn.io -o wide || true
  echo

  echo "===== Longhorn Engines ====="
  kubectl -n ${LH_NS} get engines.longhorn.io -o wide || true
  echo

  echo "===== Longhorn Nodes ====="
  kubectl -n ${LH_NS} get nodes.longhorn.io || true
  echo

  echo "===== Recent Events ====="
  kubectl get events -A --sort-by=.lastTimestamp | tail -100 || true
  echo

  echo "===== Longhorn Manager Logs ====="
  kubectl -n ${LH_NS} logs -l app=longhorn-manager --tail=200 || true
  echo

  echo "===== Report Finished ====="
} > "${REPORT}" 2>&1

echo "Report saved to ${REPORT}"
EOF

Authorize:

chmod +x collect-longhorn-baseline.sh

Execute:

./collect-longhorn-baseline.sh

View:

ls -lh longhorn-baseline-*.log
less longhorn-baseline-*.log

---

### 7.2 The Role of Baselines

A baseline before a fault can help determine:

- Whether the Volume was healthy before the fault.
- On which nodes the Replica instances were distributed before the fault.
- On which node the Pod was located before the fault.
- What the StorageClass parameters were before the fault.
- If there were any previous abnormal events.
- Whether the Longhorn components were functioning normally before the fault.

In production troubleshooting, baselines are extremely important.

Without them, many issues would have to be resolved through guesswork.

---

## VIII. Fault 1: PVC in a "Pending" State

### 8.1 Fault Phenomenon

The status of the PVC remains:

Pending

View:

kubectl get pvc -n longhorn-troubleshoot-demo

Common scenario:

NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
wrong-sc-pvc      Pending                                      wrong-longhorn

---

### 8.2 Simulating a "Pending" State Due to an Incorrect StorageClass

Create a faultyIs the csi-provisioner running?
Is the longhorn-manager running?
Are Longhorn Nodes allowed to be scheduled?
Is there sufficient disk capacity?

---

### 8.4 Common Causes

Common causes for PVC Pending:

    The StorageClass name is incorrect.
    The Longhorn StorageClass does not exist.
    There is an exception with the Longhorn CSI Provisioner.
    There is an exception with the longhorn-manager.
    The node's disk capacity is insufficient.
    Longhorn Nodes are not allowed to be scheduled.
    The number of replicas exceeds the available number of nodes.
    The default StorageClass settings do not meet expectations.

---

### 8.5 Solutions

If the issue is with the incorrect StorageClass name:

    Modify the storageClassName in the PVC YAML file.

Note:

    The storageClassName of a PVC usually cannot be directly modified after it has been created.
    In a test environment, you can delete the incorrect PVC and create it again.

To delete an incorrect PVC:

    kubectl delete -f 03-pvc-wrong-sc.yaml

If there is an exception with the Longhorn CSI Provisioner:

    kubectl -n longhorn-system get pods | grep csi
    kubectl -n longhorn-system describe pod <csi-provisioner-pod>
    kubectl -n longhorn-system logs <csi-provisioner-pod> --tail=100

If the issue is with insufficient disk capacity:

    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
    df -hT /data/longhorn

---

## Section 9: Fault 2: Pod ContainerCreating / FailedMount

### 9.1 Fault Symptoms

A Pod remains stuck in the state of:

    ContainerCreating

To check this:

    kubectl get pod -n longhorn-troubleshoot-demo -o wide

For more detailed information:

    kubectl describe pod troubleshoot-pod -n longhorn-troubleshoot-demo

Possible issues:

    FailedMount
    MountVolume.MountDevice failed
    Unable to attach or mount volumes
    Timeout occurred while waiting for volumes to be attached or mounted

---

### 9.2 Troubleshooting Steps

At the Kubernetes level:

    kubectl describe pod troubleshoot-pod -n longhorn-troubleshoot-demo
    kubectl get events -n longhorn-troubleshoot-demo --sort-by=.lastTimestamp
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

At the PVC/PV level:

    kubectl get pvc -n longhorn-troubleshoot-demo
    kubectl describe pvc troubleshoot-pvc -n longhorn-troubleshoot-demo
    kubectl get pv
    kubectl describe pv <pv-name>

At the Longhorn level:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}
    kubectl -n longhorn-system get engines.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

At the CSI level:

    kubectl -n longhorn-system get pods | grep csi
    kubectl -n longhorn-system logs <longhorn-csi-plugin-pod> --tail=100

At the node level:

    systemctl status iscsid
    iscsiadm --version
    journalctl -u kubelet --since "30 minutes ago" | tail -100

---

### 9.3 Common Causes of Mount Failures

Common reasons for Pod mounting failures include:

    iscsid is not running.
    open-iscsi is not installed.
    There is an exception with the Longhorn CSI Plugin.
    Volume attachment failed.
    The Engine is experiencing an issue.
    The Instance Manager is malfunctioning.
    A RWO volume is being used by another node.
    The node is in a NotReady state.
    There is an exception with kubelet.
    SELinux or permission issues.
    Abnormalities with the node's disk or network.

---

### 9.4 iSCSI Checking

Execute the following commands on the node where the Pod resides:

    systemctl status iscsid --no-pager
    systemctl is-active iscsid
    which iscsiadm
    iscsiadm --version

If iscsid is not running:

    systemctl enable --now iscsid

For Ubuntu installations:

    apt update
    apt install -y open-iscsi
    systemctl enable --now iscsid

For Rocky Linux 9 installations:

    dnf install -y iscsi-initiator-utils
    systemctl enable --now iscsid

---

### 9.5 Checking kubelet Logs

Execute the following command on the node where the Pod resides:

    journalctl -u kubelet --since "30 minutes ago" | tail    kubectl get pod -A -o wide | grep <pvc or application name>
    kubectl delete pod <old-pod> -n <namespace>

If the node encounters an exception that causes detachment to become stuck:

    First, check the status of the node.
    If the node can be recovered, prioritize restoring it.
    Do not force detachment arbitrarily.
    When forced removal is necessary, ensure that business writes are stopped, Volume backups have been made, and potential risks have been assessed.

---

## Section Eleven: Fault Four: Volume Degradation

### 11.1 Symptoms of the Fault

The Longhorn UI or commands will display:

    Volume Degraded

To check this:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Example:

    NAME                                      STATE      ROBUSTNESS
    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx attached   degraded

---

### 11.2 What "Degraded" Means

"Degraded" indicates that:

    The Volume may still be usable.
    However, the number of replicas is insufficient.
    Or some replicas are malfunctioning.
    Or the Volume is in the process of being rebuilt.
    Or it cannot be scheduled with the desired number of replicas.

A Degraded state is not normal and should not be ignored in production environments for an extended period.

---

### 11.3 Steps to Troubleshoot a Degraded Volume

To check the Volume:

    kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}

To check the replicas:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

To check the Longhorn Node:

    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>

To check the node itself:

    kubectl get nodes -o wide
    kubectl describe node <node-name>

To check the disk:

    df -hT /data/longhorn
    lsblk -f
    dmesg | tail -100

To check the logs:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300

---

### 11.4 Common Causes of Volume Degradation

Common reasons for Volume Degradation include:

    A node being in an NotReady state.
    An unavailable data disk.
    A failed replica.
    The replicas are in the process of being rebuilt.
    Insufficient number of nodes.
    Setting too many replicas.
    Insufficient disk space.
    The Longhorn Node not allowing scheduling.
    Issues with the Instance Manager.
    Network instability.
    Excessive CPU or memory pressure on the node.

---

### 11.5 Handling Principles

The order of handling is as follows:

    1. First, determine which PVC and business the Volume belongs to.
    2. Check if the business is affected by the issue.
    3. Verify if there are any backups available.
    4. Examine the distribution of replicas.
    5. Determine whether the problem lies with a node or a disk.
    6. Check if the Volume is in the process of being rebuilt.
    7. If it is rebuilding, monitor its progress and resource usage.
    8. If rebuilding fails, examine the scheduling conditions and disk capacity.
    9. Do not arbitrarily delete replicas or the Volume itself.

---

## Section Twelve: Practical Exercise Three: Simulating Volume Degradation Due to Insufficient Replica Scheduling

### 12.1 Description of the Scenario

If there are only two valid Longhorn data nodes but three replicas are created for a Volume, the following issues may occur:

    Insufficient replica scheduling.
    Volume Degradation.
    Abnormal scheduled conditions.

This experiment is suitable for verifying situations where the number of replicas does not match the number of available nodes.

---

### 12.2 Creating a StorageClass with 3 Replicas

Create the configuration file:

    cat > 04-sc-replica-3.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-replica-3-debug
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    parameters:
      numberOfReplicas: "3"
      staleReplicaTimeout: "30"
      fsType: "ext4"
    EOF

Apply the configuration file:

    kubectl apply -f 04-sc-replica-3.yaml

---

### 12.3 Creating a PVC with 3 Replicas

Create the configuration file:

    cat >```markdown
df -hT /data/longhorn
du -sh /data/longhorn
iostat -x 1 5

If iostat is not available:

apt update
apt install -y sysstat

Check the network status:

ip a
ping <other-node-ip>
ss -lntp

View logs:

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=500
```

---

### 13.4 Common Causes

Common reasons for the prolonged replication rebuild process include:

- Insufficient disk space.
- Unstable node network.
- Slow disk I/O on the target node.
- High load on the source replica node.
- Frequent node restarts.
- Instance Manager failures.
- Repeated replication failures.
- Excessive backups or snapshots causing complex link structures.
- Simultaneous reconstruction of multiple volumes leading to resource exhaustion.
```

---

### 13.5 Handling Principles

Action steps:

1. Do not blindly delete replicas.
2. Verify if there are any healthy replicas available.
3. Check if there are any backups in place.
4. Check the disk capacity.
5. Inspect the node network status.
6. Evaluate the node load.
7. Review the Longhorn Manager logs.
8. If a node or disk is unstable, isolate it first.
9. Address persistent issues during maintenance windows.

Production reminder:

If only one healthy replica remains, the risk is significantly increased. Priority should be given to ensuring backup and data integrity; avoid any actions that may exacerbate the issue.
```

## Chapter Fourteen: Fault 6: Volume Issues Due to Unready Nodes

### 14.1 Fault Symptoms

When checking a node:

```bash
kubectl get nodes -o wide
```

If a certain node shows "NotReady," additional issues such as:

- Pod eviction
- Volume degradation
- Replica failures
- Failed pod rescheduling
- Stuck volume attach/detach operations
may also occur.
```

---

### 14.2 Investigating Node Status

Execute the following command:

```bash
kubectl describe node <node-name>
```

Pay special attention to the following fields:

- Conditions
- Ready status
- DiskPressure
- MemoryPressure
- NetworkUnavailable status
- Taints
- Events

On the faulty node, execute the following commands:

```bash
systemctl status kubelet --no-pager
systemctl status containerd --no-pager
systemctl status iscsid --no-pager
journalctl -u kubelet --since "30 minutes ago" | tail -100
journalctl -u containerd --since "30 minutes ago" | tail -100
df -hT
free -h
dmesg | tail -100
```

---

### 14.3 Investigating Longhorn Node Status

Execute the following commands:

```bash
kubectl -n longhorn-system get nodes.longhorn.io
kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
```

Check the following fields:

- Conditions
- Disk status
- Scheduling status
- Available storage
- Scheduled storage
```

---

### 14.4 Handling Principles

If the node can be restored:

1. Restore the node first.
2. Reinstall kubelet.
3. Reinstall containerd.
4. Reinstall iscsid.
5. Check the data disk.
6. Wait for Longhorn to recognize the node as recovered.
7. Monitor the replication rebuild process.

If the node cannot be restored:

1. Identify which replicas are located on this node.
2. Verify if there are any healthy copies of the volume.
3. Check if any backups exist.
4. Prepare a new node or disk.
5. Have Longhorn reschedule the replicas.
6. Do not directly delete the volume.
```

## Chapter Fifteen: Fault 7: High DiskPressure and Insufficient Disk Space

### 15.1 Fault Symptoms

When checking a node status:

```bash
kubectl describe node <node-name>
```

If "DiskPressure" is set to "True," Longhorn may experience the following issues:

- Failure to create new PVCs.
- Replication scheduling failures.
- Volume degradation.
- Rebuild failures.
- Pod eviction.
- Node becoming unschedulable.
```

---

### 15.2 Investigation Commands

For Kubernetes:

```bash
kubectl describe node <node-name>
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For node disks:

```bash
df -hT
lsblk -f
du -sh /data/longhorn
du -sh /var/lib/containerd
du -sh /var/log
```

For Longhorn:

```bash
kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
kubectl -n```markdown
kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

View nodes:

kubectl get nodes -o wide
kubectl describe node <node-name>

View disks:

df -hT
lsblk -f
dmesg | tail -100

View backups:

Longhorn UI -> Backup

---

### 16.5 Handling Principles

If there is an available backup:

优先考虑 restoring from the backup to a new volume.
Do not directly overwrite the original volume.
Create a new PVC and verify the Pod.
Verify the data before switching services.

If there is no backup:

Preserve the current state first.
Avoid expanding the issue.
Check if there are any salvable replicas.
Handle the situation carefully, using Longhorn UI, logs, and official procedures.
For production environments, it is recommended to seek support from the vendor or the community.

---

## Section 17: Issue 9: Abnormalities with Backup Targets

### 17.1 Symptoms of the Issue

Manifestations include:

Errors on the Longhorn UI backup page.
Failed backup creation.
Failed restoration attempts.
Logs from longhorn-manager indicate access failures to the backupstore.

---

### 17.2 Troubleshooting for NFS Backup Targets

NFS Server:

Check status: systemctl status nfs-server
Export files: exportfs -v
View disk usage: df -hT /data/longhorn-backup

Kubernetes Node Testing:

Mount the NFS volume: showmount -e 10.0.0.10
Create a test directory: mkdir -p /mnt/test-longhorn-backup
Mount the volume: mount -t nfs4 10.0.0.10:/data/longhorn-backup /mnt/test-longhorn-backup
Create a file for testing: touch /mnt/test-longhorn-backup/test-$(hostname).txt
Unmount the volume: umount /mnt/test-longhorn-backup

Longhorn Configuration:

Check settings: kubectl -n longhorn-system get settings.longhorn.io | grep -i backup
Retrieve backup-target details in YAML format: kubectl -n longhorn-system get settings.longhorn.io backup-target -o yaml

---

### 17.3 Troubleshooting for S3 / MinIO Backup Targets

Check the Secret file:

kubectl -n longhorn-system get secret longhorn-s3-backup-secret -o yaml

Inspect the Bucket:

mc ls minio-admin/longhorn-backup

Verify MinIO user permissions:

mc admin user info minio-admin longhorn-backup-user
mc admin policy info minio-admin longhorn-backup-policy

Check the Endpoint configuration:

curl -I http://10.0.0.41:9000/minio/health/live
curl -I http://10.0.0.41:9000/minio/health/ready

Review Longhorn logs:

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300

Common issues include:

Non-existent Bucket.
Incorrect AccessKey or SecretKey.
Invalid Endpoint configuration.
Mismatch between HTTP and HTTPS settings.
Untrusted self-signed certificates.
Lack of necessary PutObject, GetObject, or ListBucket policies in the Policy file.
MinIO being unreachable.
Insufficient capacity for the backup target.

---
## Section 18: Methods for Troubleshooting Longhorn Logs

### 18.1 longhorn-manager Logs

View logs:

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Monitor continuously:

kubectl -n longhorn-system logs -l app=longhorn-manager -f

Search for relevant terms:

failed, error, degraded, rebuild, replica, scheduling, backup, restore, attach, detach, disk, node.

Save logs for analysis:

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=1000 > longhorn-manager-$(date +%F-%H%M%S).log

---

### 18.2 CSI Component Logs

Check CSI Pods:

kubectl -n longhorn-system get pods | grep csi

View specific logs:

kubectl -n longhorn-system logs <csi-pod-name> --tail=100

Common components to check include:

longhorn-csi-plugin, csi-provisioner, csi-attacher, csi-resizer, csi-snapshotter.

If a Pod contains multiple containers, first identify the container name:

kubectl -n longhorn-system get pod <pod-name> -o jsonpath '{.spec.containers[*].name}'

View logs for a specific container:

kubectl -n longhorn-system logs <pod-name> -c <container-name> --tail=100

---

### 18.3 Instance Manager Logs

Check logs:

kubectl -n longhorn-system get pods| PVC |  |
| PV |  |
| Longhorn Volume |  |
| Volume Status | Healthy / Degraded / Faulty |
| Replica Status |  |
| Affected Nodes |  |
| Affects Reading | Yes / No |
| Affects Writing | Yes / No |
| Backup Available | Yes / No |
| Action Taken |  |
| Recovery Time |  |
| Root Cause |  |
| Follow-up Improvements |  |

---

## Section 20: Monitoring and Alarm Recommendations

### 20.1 Mandatory Alarm Items

Production must be alerted for the following:

    Longhorn Volume is faulty.
    Longhorn Volume has been degraded for an extended period.
    Replica rebuild has not completed after a long time.
    A Longhorn node is down.
    A Longhorn disk is unavailable.
    The Longhorn disk is running out of space.
    Abnormalities with the longhorn-manager Pod.
    Issues with CSI components.
    Abnormalities with the Instance Manager.
    Backup failed.
    The backup target is unavailable.
    High DiskPressure on a node.
    A node is in an NotReady state.

---

### 20.2 Capacity Alerts

Recommended thresholds:

| Utilization | Action |
|---|---|
| 70% | Monitor trends closely. |
| 80% | Prepare for scale-out. |
| 85% | Determine the optimal time for scale-out. |
| 90% | Handle with high priority. |
| 95% | Consider it a serious risk.

---

### 20.3 Inspection Commands

Daily inspections:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl get pvc -A
    kubectl get nodes
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Node inspections:

    df -hT /data/longhorn
    systemctl is-active iscsid
    systemctl is-active kubelet
    systemctl is-active containerd

---

## Section 21: High-Risk Operation Warnings

The following operations must be performed with caution in a production environment:

    Delete PVCs.
    Delete PVs.
    Delete Longhorn Volumes.
    Delete Replicas.
    Delete Engines.
    Delete Snapshots.
    Delete Backups.
    Delete the longhorn-system namespace.
    Delete Longhorn CRDs.
    Delete /data/longhorn.
    Format Longhorn data disks.
    Forcedly detach volumes.
    Forcefully delete abnormal pods.
    Upgrade Longhorn when a volume is degraded.
    Conduct node failure drills without a backup.

Before executing any of these operations, ensure to:

    Determine which business functions will be affected.
    Verify if there are available backups.
    Check if any snapshots exist.
    Confirm whether there is a maintenance window available.
    Have a rollback plan in place.
    Ensure that all relevant information has been exported.
    Obtain approval from the relevant departments.
    Double-check all steps.

---

## Section 22: Experimental Cleanup

### 22.1 Delete Incorrect PVCs

Execution:

    kubectl delete -f 03-pvc-wrong-sc.yaml --ignore-not-found=true

---

### 22.2 Delete Resources Used in Failed Replica Scheduling Experiments

Execution:

    kubectl delete -f 05-pvc-replica-3-debug.yaml --ignore-not-found=true
    kubectl delete -f 04-sc-replica-3.yaml --ignore-not-found=true

---

### 22.3 Delete the Main Test Pod

Execution:

    kubectl delete -f 02-pod.yaml --ignore-not-found=true

---

### 22.4 Delete the Main Test PVC

High-risk warning:

    Deleting a PVC may result in the deletion of the associated Longhorn Volume.
    Only experimental resources will be deleted here.

Execution:

    kubectl delete -f 01-pvc.yaml --ignore-not-found=true

Verification:

    kubectl get pvc -n longhorn-troubleshoot-demo
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 22.5 Delete the Namespace

After confirming that no resources remain:

    kubectl delete namespace longhorn-troubleshoot-demo

---

### 22.6 Delete Local Files

Execution:

    rm -f 01-pvc.yaml
    rm -f 02-pod.yaml
    rm -f 03-pvc-wrong-sc.yaml
    rm -f 04-sc-replica-3.yaml
    rm -f 05-pvc-replica-3-debug.yaml
    rm -f collect-longhorn-baseline.sh
    rm -f longIf the Volume has already failed, I will first preserve the current state of the system and export logs related to PVCs, PVs, Volumes, Replicas, Events, as well as those from the longhorn-manager. This is done to confirm whether there is a backup available. In production environments, it is preferable to restore data from a backup to a new Volume first, before creating new PVCs and verifying the Pod data. This approach avoids making arbitrary changes directly on the original Volume.

The general principle here is to: first determine the scope of the issue, then identify the relationships between PVCs, PVs, Volumes, Replicas, and Nodes; next, preserve the current state before deciding whether to restore nodes, wait for reconstruction, or perform a backup recovery. It's important to note that Replicas are designed for high availability, not just as backups, so critical data must always have a backup target and a recovery plan in place.

---

## Summary of Chapter 25

This chapter has covered the troubleshooting process for Longhorn:

1. Troubleshooting should be approached at multiple levels, including the business layer, Kubernetes layer, CSI layer, Longhorn control plane, data plane, node layer, and backup layer.
2. When dealing with PVC Pending issues, focus on checking StorageClass settings, CSI components, longhorn-manager logs, node disks, and replica scheduling.
3. For Pod ContainerCreating errors, examine FailedMount, FailedAttachVolume, iscsid, CSI Plugin logs, and kubelet logs for relevant clues.
4. Multi-Attach scenarios are commonly seen when RWO PVCs are used simultaneously by multiple Pods on different nodes.
5. A Volume being marked as Degraded indicates that there are insufficient or abnormal replicas, or that the volume is in the process of being rebuilt.
6. A Degraded state is not normal and should not be ignored for extended periods.
7. Replica Rebuilding can consume significant resources such as disk space, network bandwidth, and CPU power.
8. If a Replica Rebuild does not complete within a reasonable time, check for issues with disks, networks, node status, and manager logs.
9. When a Node is marked as NotReady, it may cause Replicas to malfunction, the Volume to degrade, or Pod re-scheduling attempts to fail.
10. DiskPressure can affect PVC creation, replica scheduling, and replica rebuilding processes.
11. The Longhorn data directory should not be manually deleted or modified.
12. iscsid is an important check item when troubleshooting disk mounting issues with Longhorn.
13. longhorn-manager logs are crucial for identifying control plane-related problems.
14. CSI component logs provide valuable information when troubleshooting PVC creation and mounting issues.
15. Abnormalities in the Instance Manager can affect both the Engine and Replica components.
16. When a Volume fails, the primary action should be to preserve the current state of the system.
17. If there is a backup available, restore data to a new Volume first for verification purposes.
18. In the absence of a backup, never delete volumes or replicas without proper caution.
19. Production environments must implement monitoring systems and failover procedures.
20. The next chapter will discuss Longhorn performance optimization and best practices regarding disk usage, network configuration, scheduling, and resource management.

---

## References

Longhorn Official Documentation:

    https://longhorn.io/docs/latest/

Longhorn Troubleshooting Guide:

    https://longhorn.io/docs/latest/troubleshoot/troubleshooting/

Longhorn Volume Conditions:

    https://longhorn.io/docs/latest/nodes-and-volumes/volumes/volume-conditions/

Longhorn Replica Rebuilding:

    https://longhorn.io/docs/latest/advanced-resources/rebuilding/

Longhorn Offline Replica Rebuilding:

    https://longhorn.io/docs/latest/advanced-resources/rebuilding/offline-replica-rebuilding/

Longhorn Nodes and Volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn StorageClass Parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn Backup and Restore:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Best Practices:

    https://longhorn.io/docs/latest/best-practices/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Events:

    https://kubernetes.io/docs/reference/kubectl/generated/kubectl_events/

Kubernetes Safely Drain a Node:

    https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/