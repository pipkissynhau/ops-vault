# Longhorn Troubleshooting: Volume Degraded, Replica Rebuilding, and Node Abnormalities

Recommended Path: 05-Storage/03-LongHorn/08-Longhorn Troubleshooting: Volume Degraded, Replica Rebuilding, and Node Abnormalities.md

Tags: #Longhorn #FaultCheck. #VolumeDegraded #ReplicaRebuild #PVC #PV #CSI #FailedMount #iSCSI #Kubernetes #BlockStorage #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the eighth article in the Longhorn module, focusing on troubleshooting methods in Longhorn production operations.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence
- Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability
- Longhorn Backup and Recovery: Backup Target, Snapshot, and Volume Restore

This article enters the Longhorn practical troubleshooting phase, focusing on solving:

    How to investigate PVCs that remain Pending?
    How to investigate Pods that remain ContainerCreating?
    How to investigate FailedMount?
    How to investigate FailedAttachVolume?
    How to investigate RWO volume Multi-Attach?
    How to investigate Volume Degraded?
    How to investigate Replica continuously Rebuilding?
    How to investigate Replica scheduling failure?
    How to handle Volume Faulted?
    What impact does a node NotReady have on Longhorn?
    What impact does node DiskPressure have on Longhorn?
    Why does iscsid abnormality cause mount failure?
    How to read longhorn-manager logs?
    How to use Longhorn CRD for troubleshooting?
    When not to arbitrarily delete Replica/Volume?
    How to establish a Longhorn troubleshooting process in production?

This article emphasizes practical operation, organizing content with commands, phenomena, judgment paths, and handling actions.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Master the layered approach to Longhorn troubleshooting.
2. Investigate PVC Pending.
3. Investigate Pod ContainerCreating.
4. Investigate FailedMount.
5. Investigate FailedAttachVolume.
6. Investigate RWO volume Multi-Attach.
7. Investigate Volume Degraded.
8. Observe Replica Rebuild.
9. Investigate Replica scheduling failure.
10. Investigate the impact of node NotReady on Volume.
11. Investigate node DiskPressure and disk capacity issues.
12. Investigate iscsid/open-iscsi issues.
13. View Longhorn Manager, CSI, and Instance Manager logs.
14. Locate fault scope based on PVC, PV, Volume, and Replica associations.
15. Establish a production troubleshooting record template.
16. Understand which operations are high-risk and should not be executed arbitrarily in production.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node Planning:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 3.2 Longhorn Environment

Longhorn Namespace:

    longhorn-system

Longhorn StorageClass:

    longhorn

Longhorn Data Directory:

    /data/longhorn

Experiment Namespace:

    longhorn-troubleshoot-demo

---

### 3.3 Fault Simulation Notes in This Article

This article includes some fault simulations, such as:

    Incorrect StorageClass causing PVC Pending
    3 replicas failing to schedule when node count is insufficient
    Stopping kubelet or shutting down a Worker node to observe Volume Degraded
    Cross-node pod mounting to observe Multi-Attach risks

High-risk warning:

    Stopping kubelet, shutting down nodes, deleting PVCs, deleting Volumes, and deleting Replicas are all high-risk operations.
    These actions are only allowed in experimental environments.
    In production environments, you must first confirm business impact, perform backups, schedule during maintenance windows, and have rollback plans.
    The first principle of production troubleshooting is to preserve the scene and never delete resources arbitrarily.

---

## IV. Longhorn Troubleshooting General Method

### 4.1 Layered Troubleshooting

Longhorn troubleshooting cannot rely solely on UI or Kubernetes Pod status.

Recommended troubleshooting layers:

    1. Business Layer
       Is the Pod Running?
       Can the application read/write the mounted directory?

    2. Kubernetes Storage Layer
       Is the PVC Bound?
       Does the PV exist?
       Is the StorageClass correct?

    3. CSI Layer
       Is the csi-provisioner normal?
       Is the csi-attacher normal?
       Is the csi-plugin normal?

    4. Longhorn Control Plane
       Is the longhorn-manager normal?
       Is the Longhorn Volume status Healthy?
       Are Longhorn Nodes/Disk schedulable?

5. Longhorn Data Plane
   - Is the Engine healthy?
   - Are the Replicas running?
   - Is the Instance Manager healthy?

6. Node System Layer
   - Is kubelet healthy?
   - Is containerd healthy?
   - Is iscsid healthy?
   - Is the data disk healthy?
   - Is the network healthy?

7. Backup Recovery Layer
   - Is there a Snapshot?
   - Is there a Backup?
   - Is the Backup Target available?
   - Is a Restore needed?

---

### 4.2 Troubleshooting Entry Summary Table

| Fault Phenomenon | First Entry | Second Entry | Deep Direction |
|---|---|---|---|
| PVC Pending | describe pvc | StorageClass / CSI | longhorn-manager / node disk |
| Pod ContainerCreating | describe pod | events | FailedMount / iscsid / attach |
| FailedMount | pod events | kubelet logs | CSI plugin / iscsid |
| FailedAttachVolume | events | volume status | Engine / node / Multi-Attach |
| Multi-Attach | pod distribution | volume attach node | RWO volume used by multiple nodes |
| Volume Degraded | volumes.longhorn.io | replicas.longhorn.io | node, disk, scheduling, rebuild |
| Replica Rebuilding Stuck | replica status | manager logs | disk capacity, network, node status |
| Volume Faulted | volume status | replica status | Backup / Restore / preserve scene |
| Node NotReady | kubectl get nodes | kubelet logs | Pod eviction, Volume detach |
| DiskPressure | describe node | df / du | disk space, Replica scheduling |
| Backup Failed | Longhorn UI | manager logs | NFS / S3 / Secret / permissions |

---

### 4.3 Common Command Quick Reference

Kubernetes Layer:

    kubectl get nodes -o wide
    kubectl get pods -A -o wide
    kubectl get pvc -A
    kubectl get pv
    kubectl get sc
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Longhorn Layer:

    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get engines.longhorn.io -o wide
    kubectl -n longhorn-system get instancemanagers.longhorn.io
    kubectl -n longhorn-system get nodes.longhorn.io

Log Layer:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200
    kubectl -n longhorn-system logs <csi-pod-name> --tail=100
    kubectl -n longhorn-system logs <instance-manager-pod-name> --tail=100
    journalctl -u kubelet --since "30 minutes ago" | tail -100
    journalctl -u containerd --since "30 minutes ago" | tail -100

Node Layer:

    systemctl status kubelet
    systemctl status containerd
    systemctl status iscsid
    iscsiadm --version
    df -hT
    lsblk -f
    dmesg | tail -100
    journalctl -k --since "1 hour ago" | tail -100

---

## Five, Pre-Fault Baseline Checks

### 5.1 Check Longhorn Components

Execute:

    kubectl -n longhorn-system get pods -o wide

Check for anomalies:

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If anomalies exist, first check:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl -n longhorn-system logs <pod-name> --tail=100
    kubectl get events -n longhorn-system --sort-by=.lastTimestamp | tail -100

---

### 5.2 Check StorageClass

Execute:

    kubectl get sc
    kubectl describe sc longhorn

Focus on:

    Provisioner
    ReclaimPolicy
    VolumeBindingMode
    AllowVolumeExpansion
    Parameters
    numberOfReplicas

---

### 5.3 Check Longhorn Volume

Execute:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

If a Volume already exists, check for:

    Degraded
    Faulted
    Detached (abnormal)
    Unknown
    Rebuilding (long-running without completion)

---

### 5.4 Check Longhorn Replica

Execute:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Focus on:

    running
    stopped
    rebuilding
    error
    failed
    failed at
    node id
    disk id

---

### 5.5 Check Longhorn Node and Disk

Execute:

    kubectl -n longhorn-system get nodes.longhorn.io

Review details:

    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01
    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker02

Focus on:

    Allow Scheduling
    Conditions
    Disk Path
    Storage Maximum
    Storage Available
    Storage Scheduled
    Disk Status

---

### 5.6 Check Kubernetes Nodes

Execute:

    kubectl get nodes -o wide

Check for pressure:

    kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"

Requirements:

    Ready=True
    DiskPressure=False
    MemoryPressure=False
    PIDPressure=False

---

### 5.7 Check Node Dependencies

On each Worker node execute:

    systemctl is-active iscsid
    iscsiadm --version
    systemctl status kubelet --no-pager
    systemctl status containerd --no-pager
    df -hT /data/longhorn
    lsblk -f

---

## Six. Practical Exercise I: Create Troubleshooting Test Resources

### 6.1 Create Namespace

Execute:

    kubectl create namespace longhorn-troubleshoot-demo

Check:

    kubectl get ns longhorn-troubleshoot-demo

---

### 6.2 Create Working Directory

Execute:

    mkdir -p ~/longhorn-troubleshoot-demo
    cd ~/longhorn-troubleshoot-demo

---

### 6.3 Create PVC

Create file:

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

Apply:

    kubectl apply -f 01-pvc.yaml

Check:

    kubectl get pvc -n longhorn-troubleshoot-demo
    kubectl get pv

---

### 6.4 Create Pod

Create file:

    cat > 02-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: troubleshoot-pod
      namespace: longhorn-troubleshoot-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          imagePullPolicy: IfNotPresent
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
            claimName: troubleshoot-pvc
    EOF

Apply:

    kubectl apply -f 02-pod.yaml

Check:

    kubectl get pod -n longhorn-troubleshoot-demo -o wide

---

### 6.5 Write Test Data

Execute: /think

kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "echo 'longhorn troubleshooting demo' > /data/hello.txt"
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "date > /data/write-time.txt"
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "mkdir -p /data/app/logs /data/db"
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "echo 'id,name' > /data/db/users.csv"
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- sh -c "echo '1,alice' >> /data/db/users.csv"

View:

kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- find /data -maxdepth 4 -type f -print
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- cat /data/hello.txt
kubectl exec -n longhorn-troubleshoot-demo troubleshoot-pod -- cat /data/db/users.csv

---

### 6.6 Record Volume Name

View PVC:

kubectl describe pvc troubleshoot-pvc -n longhorn-troubleshoot-demo

Record:

Volume: pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

Set variable:

export LH_VOLUME=<replace with actual Longhorn Volume name>

Example:

export LH_VOLUME=pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

View Longhorn Volume:

kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}

---

## SevenI don't know.Practical Exercise Two: Establishing a Troubleshooting Baseline Report

### 7.1 Create Baseline Collection Script

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

Grant permissions:

chmod +x collect-longhorn-baseline.sh

Execute: /think

./collect-longhorn-baseline.sh

View:

    ls -lh longhorn-baseline-*.log
    less longhorn-baseline-*.log

---

### 7.2 Purpose of Baselines

Pre-failure baselines help determine:

    Whether the Volume was Healthy before the failure.
    On which nodes the Replica was distributed before the failure.
    On which node the Pod was located before the failure.
    What StorageClass parameters were in place before the failure.
    Whether there were already abnormal events before the failure.
    Whether Longhorn components were functioning normally before the failure.

Baselines are crucial in production troubleshooting.

Without a baseline, many issues can only be guessed at.

---

## Eight, Fault One: PVC Pending

### 8.1 Fault Symptoms

The PVC status remains:

    Pending

Check:

    kubectl get pvc -n longhorn-troubleshoot-demo

Common symptoms:

    NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
    wrong-sc-pvc      Pending                                      wrong-longhorn

---

### 8.2 Simulating an Error StorageClass Causing Pending

Create an erroneous PVC:

    cat > 03-pvc-wrong-sc.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: wrong-sc-pvc
      namespace: longhorn-troubleshoot-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: wrong-longhorn
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f 03-pvc-wrong-sc.yaml

Check:

    kubectl get pvc -n longhorn-troubleshoot-demo

Detailed check:

    kubectl describe pvc wrong-sc-pvc -n longhorn-troubleshoot-demo

---

### 8.3 Troubleshooting Path for PVC Pending

Troubleshooting commands:

    kubectl describe pvc wrong-sc-pvc -n longhorn-troubleshoot-demo
    kubectl get sc
    kubectl describe sc longhorn
    kubectl get events -n longhorn-troubleshoot-demo --sort-by=.lastTimestamp
    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Focus on:

    Whether the StorageClass exists.
    Whether the Provisioner is correct.
    Whether PVC Events indicate storageclass not found.
    Whether csi-provisioner is Running.
    Whether longhorn-manager is Running.
    Whether Longhorn Nodes allow scheduling.
    Whether disk capacity is sufficient.

---

### 8.4 Common Causes

Common causes for PVC Pending:

    StorageClass name is incorrect.
    Longhorn StorageClass does not exist.
    Longhorn CSI Provisioner is abnormal.
    longhorn-manager is abnormal.
    Node disk capacity is insufficient.
    Longhorn Nodes do not allow scheduling.
    Replica count exceeds available nodes.
    Default StorageClass settings do not meet expectations.

---

### 8.5 Resolution Methods

If the StorageClass was incorrectly specified:

    Modify the storageClassName in the PVC YAML.

Note:

    The storageClassName of a PVC is typically not directly modifiable after creation.
    In experimental environments, you can delete the erroneous PVC and recreate it.

Delete the erroneous PVC:

    kubectl delete -f 03-pvc-wrong-sc.yaml

If Longhorn CSI is abnormal:

    kubectl -n longhorn-system get pods | grep csi
    kubectl -n longhorn-system describe pod <csi-provisioner-pod>
    kubectl -n longhorn-system logs <csi-provisioner-pod> --tail=100

If disk capacity is insufficient:

    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
    df -hT /data/longhorn

---

## Nine, Fault Two: Pod ContainerCreating / FailedMount

### 9.1 Fault Symptoms

Pods remain stuck in:

    ContainerCreating

Check:

    kubectl get pod -n longhorn-troubleshoot-demo -o wide

Detailed check:

    kubectl describe pod troubleshoot-pod -n longhorn-troubleshoot-demo

Possible events:

    FailedMount
    MountVolume.MountDevice failed
    Unable to attach or mount volumes
    timeout expired waiting for volumes to attach or mount

---

### 9.2 Troubleshooting Path

Kubernetes Layer: /think

kubectl describe pod troubleshoot-pod -n longhorn-troubleshoot-demo  
kubectl get events -n longhorn-troubleshoot-demo --sort-by=.lastTimestamp  
kubectl get events -A --sort-by=.lastTimestamp | tail -100  

PVC / PV Layer:  

    kubectl get pvc -n longhorn-troubleshoot-demo  
    kubectl describe pvc troubleshoot-pvc -n longhorn-troubleshoot-demo  
    kubectl get pv  
    kubectl describe pv <pv-name>  

Longhorn Layer:  

    kubectl -n longhorn-system get volumes.longhorn.io -o wide  
    kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}  
    kubectl -n longhorn-system get engines.longhorn.io -o wide  
    kubectl -n longhorn-system get replicas.longhorn.io -o wide  

CSI Layer:  

    kubectl -n longhorn-system get pods | grep csi  
    kubectl -n longhorn-system logs <longhorn-csi-plugin-pod> --tail=100  

Node Layer:  

    systemctl status iscsid  
    iscsiadm --version  
    journalctl -u kubelet --since "30 minutes ago" | tail -100  

---

### 9.3 Common Causes  

Pod mount failure common causes:  

    iscsid is not running.  
    open-iscsi is not installed.  
    Longhorn CSI Plugin is abnormal.  
    Volume attach failed.  
    Engine is abnormal.  
    Instance Manager is abnormal.  
    RWO volume is occupied by another node.  
    Node is NotReady.  
    kubelet is abnormal.  
    SELinux / permission issues.  
    Node disk or network anomalies.  

---

### 9.4 iSCSI Check  

Execute on the node where the Pod resides:  

    systemctl status iscsid --no-pager  
    systemctl is-active iscsid  
    which iscsiadm  
    iscsiadm --version  

If iscsid is not running:  

    systemctl enable --now iscsid  

Ubuntu Installation:  

    apt update  
    apt install -y open-iscsi  
    systemctl enable --now iscsid  

Rocky Linux 9 Installation:  

    dnf install -y iscsi-initiator-utils  
    systemctl enable --now iscsid  

---

### 9.5 kubelet Log Check  

Execute on the node where the Pod resides:  

    journalctl -u kubelet --since "30 minutes ago" | tail -200  

Focus on searching for:  

    MountVolume  
    FailedMount  
    attach  
    iscsi  
    longhorn  
    timeout  
    permission denied  

---

### 9.6 Handling Principles  

Handling order:  

    1. First confirm if the PVC is Bound.  
    2. Then check the specific error in Pod events.  
    3. Then confirm if the Volume is Attached.  
    4. Then check the CSI Plugin.  
    5. Then check iscsid.  
    6. Then check kubelet.  
    7. Then check node disk and network.  

Do not directly:  

    Delete PVC  
    Delete PV  
    Delete Volume  
    Delete Replica  
    Reinstall Longhorn  

---

## Ten, Fault Three: Multi-Attach  

### 10.1 Fault Phenomenon  

Events show similar messages:  

    Multi-Attach error for volume  
    Volume is already exclusively attached to one node and can't be attached to another  

Commonly occurs when:  

    RWO PVC is used simultaneously by Pods on multiple nodes.  
    Deployment with multiple replicas shares the same RWO PVC.  
    Old Pod's node is abnormal, and Volume is not detached promptly.  
    Force deletion of Pod leaves mount state un-released.  

---

### 10.2 Troubleshooting Commands  

Check Pod distribution:  

    kubectl get pod -n longhorn-troubleshoot-demo -o wide  

Check PVC users:  

    kubectl describe pvc troubleshoot-pvc -n longhorn-troubleshoot-demo  

Check events:  

    kubectl get events -A --sort-by=.lastTimestamp | tail -100  

Check Volume attach node:  

    kubectl -n longhorn-system get volumes.longhorn.io -o wide  
    kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}  

---

### 10.3 Common Misuse Examples  

Error example:  

    A Deployment sets replicas: 2  
    Both Pods mount the same ReadWriteOnce PVC  

Risks:  

    If both Pods are scheduled to different nodes, Multi-Attach may occur.  
    Even if it runs temporarily, it's not a correct design for stateful applications.  
    Multi-replica stateful applications should use StatefulSet + volumeClaimTemplates.  

---

### 10.4 Handling Method

If it's a Deployment with multiple replicas sharing RWO PVC:

    kubectl scale deploy <deploy-name> -n <namespace> --replicas=1

If the old Pod hasn't been released:

    kubectl get pod -A -o wide | grep <pvc or application name>
    kubectl delete pod <old-pod> -n <namespace>

If node anomalies cause detach to get stuck:

    First confirm the node status.
    If the node can be recovered, prioritize restoring the node.
    Do not arbitrarily force detach.
    When forced handling is necessary, first confirm business write stop, Volume backup, and risks.

---

## 11. Fault Four: Volume Degraded

### 11.1 Fault Phenomenon

Longhorn UI or command shows:

    Volume Degraded

Check:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Example:

    NAME                                      STATE      ROBUSTNESS
    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx attached   degraded

---

### 11.2 Meaning of Degraded

Degraded indicates:

    The Volume may still be usable.
    But replica count is insufficient.
    Or some Replica is abnormal.
    Or it's in the process of rebuilding.
    Or cannot schedule to the expected replica count.

Degraded is not a normal state.

It cannot be ignored for long in production.

---

### 11.3 Degraded Troubleshooting Path

Check Volume:

    kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}

Check Replica:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Check Longhorn Node:

    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>

Check nodes:

    kubectl get nodes -o wide
    kubectl describe node <node-name>

Check disks:

    df -hT /data/longhorn
    lsblk -f
    dmesg | tail -100

Check logs:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300

---

### 11.4 Common Causes

Common causes for Volume Degraded:

    A node is NotReady.
    A data disk is unavailable.
    A Replica failed.
    Replicas are rebuilding.
    Insufficient node count.
    Replica count set too high.
    Insufficient disk space.
    Longhorn Node disallows scheduling.
    Instance Manager abnormality.
    Network jitter.
    Excessive node CPU/memory pressure.

---

### 11.5 Handling Principles

Handling order:

    1. First confirm which PVC and business the Volume belongs to.
    2. Confirm if the business is affected.
    3. Confirm if there is a Backup.
    4. Check Replica distribution.
    5. Determine if a node failure occurred.
    6. Determine if a disk failure occurred.
    7. Determine if it's in the process of rebuilding.
    8. If rebuilding, observe progress and resources.
    9. If rebuilding fails, check scheduling conditions and disk capacity.
    10. Do not arbitrarily delete Replica or Volume.

---

## 12. Hands-on Three: Simulate Replica Scheduling Insufficiency Leading to Degraded

### 12.1 Scenario Description

If there are currently only two valid Longhorn data nodes, but a 3-replica Volume is created, it may result in:

    Replica scheduling insufficiency
    Volume Degraded
    Scheduled condition anomalies

This experiment is suitable for verifying the issue of replica count mismatch with node count.

---

### 12.2 Create 3-Replica StorageClass

Create:

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

Apply:

    kubectl apply -f 04-sc-replica-3.yaml

---

### 12.3 Create 3-Replica PVC

Create:

    cat > 05-pvc-replica-3-debug.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: replica-3-debug-pvc
      namespace: longhorn-troubleshoot-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-replica-3-debug
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f 05-pvc-replica-3-debug.yaml

Check:

kubectl get pvc -n longhorn-troubleshoot-demo
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

### 12.4 Checking Scheduling Conditions

Find the corresponding Volume:

    kubectl describe pvc replica-3-debug-pvc -n longhorn-troubleshoot-demo

Record the Volume name:

    export REPLICA3_VOLUME=<replace with actual volume name>

Check:

    kubectl -n longhorn-system describe volumes.longhorn.io ${REPLICA3_VOLUME}

Focus on:

    Conditions
    Scheduled
    Robustness
    Number Of Replicas
    Kubernetes Status

If Scheduled shows an anomaly, it indicates the replica scheduling conditions are not met.

---

### 12.5 Handling Options

Optional handling:

    Add Longhorn data nodes.
    Allow more nodes to participate in scheduling.
    Increase available disk space.
    Reduce replica count.
    Adjust StorageClass or Volume replica count.
    Check if Longhorn Node allows scheduling.

Experimental cleanup:

    kubectl delete -f 05-pvc-replica-3-debug.yaml
    kubectl delete -f 04-sc-replica-3.yaml

---

## Thirteen, Fault Five: Replica Rebuilding Stuck Long-Term

### 13.1 Fault Phenomenon

Longhorn UI or commands show Replica always:

    rebuilding

Check:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Volume remains:

    degraded

---

### 13.2 Normal Behavior During Rebuild

Replica Rebuild itself is a normal mechanism.

When a node recovers or a replica is lost, Longhorn attempts to restore the replica.

Rebuild consumes:

    Source replica disk read
    Target replica disk write
    Node-to-node network bandwidth
    CPU
    Longhorn Engine / Replica resources

---

### 13.3 Troubleshooting Long-Term Rebuilding

Check Volume:

    kubectl -n longhorn-system describe volumes.longhorn.io ${LH_VOLUME}

Check Replica:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Check Longhorn Node:

    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>

Check disk:

    df -hT /data/longhorn
    du -sh /data/longhorn
    iostat -x 1 5

If iostat is not available:

    apt update
    apt install -y sysstat

Check network:

    ip a
    ping <other-node-ip>
    ss -lntp

Check logs:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=500

---

### 13.4 Common Causes

Common reasons for Replica Rebuilding not ending long-term:

    Insufficient disk space.
    Unstable node network.
    Slow disk I/O on target node.
    High load on source replica node.
    Frequent node reboots.
    Instance Manager anomaly.
    Replica repeatedly failing.
    Too many Backups or Snapshots causing complex chain.
    Multiple Volumes rebuilding simultaneously causing resource exhaustion.

---

### 13.5 Handling Principles

Handling options:

    1. Do not blindly delete Replica.
    2. First confirm if there are healthy replicas.
    3. First confirm if there are Backups.
    4. Check disk capacity.
    5. Check node network.
    6. Check node load.
    7. Check Longhorn Manager logs.
    8. If node or disk is unstable, isolate unstable node or disk first.
    9. Handle long-term abnormal replicas during maintenance window.

Production reminder:

    If only one healthy replica remains, the risk is very high.
    At this point, prioritize backup and data security, avoid operations that could expand the fault.

---

## Fourteen, Fault Six: Node NotReady Causing Volume Anomalies

### 14.1 Fault Phenomenon

Check nodes:

    kubectl get nodes -o wide

A node shows:

    NotReady

It may also show:

    Pod evictions
    Volume Degraded
    Replica Failed
    Pod rescheduling failure
    Volume detach / attach stuck

---

### 14.2 Checking Node Status

Execute:

    kubectl describe node <node-name>

Focus on:

    Conditions
    Ready
    DiskPressure
    MemoryPressure
    NetworkUnavailable
    Taints
    Events

Execute on the faulty node: /think

systemctl status kubelet --no-pager
systemctl status containerd --no-pager
systemctl status iscsid --no-pager
journalctl -u kubelet --since "30 minutes ago" | tail -100
journalctl -u containerd --since "30 minutes ago" | tail -100
df -hT
free -h
dmesg | tail -100

---

### 14.3 Troubleshooting Longhorn Node Status

Run:

    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>

Check:

    Conditions
    Disk Status
    Scheduling
    Storage Available
    Storage Scheduled

---

### 14.4 Handling Principles

If the node is recoverable:

    Prioritize node recovery.
    Recover kubelet.
    Recover containerd.
    Recover iscsid.
    Check data disks.
    Wait for Longhorn to recognize node recovery.
    Observe Replica Rebuild.

If the node is unrecoverable:

    Confirm which Replicas exist on the node.
    Confirm if the Volume still has healthy replicas.
    Confirm if Backup exists.
    Prepare a new node or new disk.
    Let Longhorn reschedule Replica.
    Do not directly delete Volume.

---

## Fifteen, Fault Seven: Node DiskPressure and Insufficient Disk Space

### 15.1 Fault Symptoms

Node status:

    DiskPressure=True

Check:

    kubectl describe node <node-name>

Longhorn may exhibit:

    New PVC creation failure.
    Replica scheduling failure.
    Volume Degraded.
    Rebuild failure.
    Pod evicted.
    Node unschedulable.

---

### 15.2 Troubleshooting Commands

Kubernetes:

    kubectl describe node <node-name>
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Node disk:

    df -hT
    lsblk -f
    du -sh /data/longhorn
    du -sh /var/lib/containerd
    du -sh /var/log

Longhorn:

    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

### 15.3 Common Causes

DiskPressure common causes:

    Longhorn data directory full.
    containerd image cache too large.
    Container logs too large.
    System disk too small.
    /data/longhorn not mounted separately, actual writes to system disk.
    Too much temporary data from Snapshot or Backup.
    Replica rebuild generates additional pressure.

---

### 15.4 Handling Principles

Handling order:

    1. First confirm whether system disk or data disk is full.
    2. If Longhorn data is written to system disk, immediately stop creating production PVCs.
    3. Clean up unused images and logs with caution.
    4. Do not manually delete internal data in /data/longhorn.
    5. Expand data disk or add nodes.
    6. Clean up unused Snapshot/Backup should be done via Longhorn UI/API.
    7. After recovery, observe if Volume returns from Degraded to Healthy.

High-risk warning:

    Prohibit direct rm -rf /data/longhorn.
    Prohibit manual deletion of replica directories.
    Prohibit manual modification of Longhorn data files.

---

## Sixteen, Fault Eight: Volume Faulted

### 16.1 Fault Symptoms

Longhorn Volume status:

    Faulted

Check:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Faulted typically indicates:

    Volume severe anomalies.
    Insufficient healthy replicas.
    Pod may be unable to read/write.
    Business may already be unavailable.

---

### 16.2 First Principle: Preserve the Scene

After seeing Faulted, do not immediately:

    Delete PVC
    Delete PV
    Delete Volume
    Delete Replica
    Delete longhorn-system
    Reinstall Longhorn
    Clean /data/longhorn

Should first:

    Preserve the scene.
    Export status.
    Check Backup.
    Confirm business impact.
    Upgrade handling level.

---

### 16.3 Status Collection

Run: /think

kubectl get pvc -A > pvc-all.txt
kubectl get pv > pv-all.txt
kubectl -n longhorn-system get volumes.longhorn.io -o yaml > longhorn-volumes.yaml
kubectl -n longhorn-system get replicas.longhorn.io -o yaml > longhorn-replicas.yaml
kubectl -n longhorn-system get engines.longhorn.io -o yaml > longhorn-engines.yaml
kubectl -n longhorn-system get nodes.longhorn.io -o yaml > longhorn-nodes.yaml
kubectl get events -A --sort-by=.lastTimestamp > events-all.txt
kubectl -n longhorn-system logs -l app=longhorn-manager --tail=1000 > longhorn-manager.log

---

### 16.4 Troubleshooting Path

Check Volume:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Check Replica:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Check Nodes:

    kubectl get nodes -o wide
    kubectl describe node <node-name>

Check Disks:

    df -hT
    lsblk -f
    dmesg | tail -100

Check Backup:

    Longhorn UI -> Backup

---

### 16.5 Handling Principles

If there is an available Backup:

    Prioritize restoring from Backup to a new Volume.
    Do not directly overwrite the original Volume.
    Create a new PVC and validate the Pod.
    Switch the business after verifying the data.

If there is no Backup:

    First preserve the current state.
    Do not escalate the failure.
    Check if there are any salvageable replicas.
    Handle with caution using Longhorn UI, logs, and official procedures.
    Production environments are advised to seek vendor or community support.

---

## SeventeenI don't know.Fault Nine: Backup Target Abnormality

### 17.1 Fault Symptoms

Manifestations:

    Longhorn UI Backup page shows errors.
    Backup creation fails.
    Restore fails.
    longhorn-manager logs show backupstore access failure.

---

### 17.2 NFS Backup Target Troubleshooting

NFS Server:

    systemctl status nfs-server
    exportfs -v
    df -hT /data/longhorn-backup

Kubernetes Node Testing:

    showmount -e 10.0.0.10
    mkdir -p /mnt/test-longhorn-backup
    mount -t nfs4 10.0.0.10:/data/longhorn-backup /mnt/test-longhorn-backup
    touch /mnt/test-longhorn-backup/test-$(hostname).txt
    umount /mnt/test-longhorn-backup

Longhorn Settings:

    kubectl -n longhorn-system get settings.longhorn.io | grep -i backup
    kubectl -n longhorn-system get settings.longhorn.io backup-target -o yaml

---

### 17.3 S3 / MinIO Backup Target Troubleshooting

Check Secret:

    kubectl -n longhorn-system get secret longhorn-s3-backup-secret -o yaml

Check Bucket:

    mc ls minio-admin/longhorn-backup

Check MinIO User Permissions:

    mc admin user info minio-admin longhorn-backup-user
    mc admin policy info minio-admin longhorn-backup-policy

Check Endpoint:

    curl -I http://10.0.0.41:9000/minio/health/live
    curl -I http://10.0.0.41:9000/minio/health/ready

Check Longhorn Logs:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=300

Common Causes:

    Bucket does not exist.
    AccessKey is incorrect.
    SecretKey is incorrect.
    Endpoint is incorrect.
    HTTP / HTTPS configuration mismatch.
    Self-signed certificate is not trusted.
    Policy lacks PutObject / GetObject / ListBucket permissions.
    MinIO is unreachable.
    Backup Target has insufficient capacity.

---

## EighteenI don't know.Longhorn Log Troubleshooting Methods

### 18.1 longhorn-manager Logs

View:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Continuous Viewing:

    kubectl -n longhorn-system logs -l app=longhorn-manager -f

Key Search Terms:

    failed
    error
    degraded
    rebuild
    replica
    scheduling
    backup
    restore
    attach
    detach
    disk
    node

Save Logs: /think

kubectl -n longhorn-system logs -l app=longhorn-manager --tail=1000 > longhorn-manager-$(date +%F-%H%M%S).log

---

### 18.2 CSI Components Logs

Check CSI Pod:

    kubectl -n longhorn-system get pods | grep csi

Check specific logs:

    kubectl -n longhorn-system logs <csi-pod-name> --tail=100

Common components to check:

    longhorn-csi-plugin
    csi-provisioner
    csi-attacher
    csi-resizer
    csi-snapshotter

If the Pod has multiple containers, first check container names:

    kubectl -n longhorn-system get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'

Check specific container:

    kubectl -n longhorn-system logs <pod-name> -c <container-name> --tail=100

---

### 18.3 Instance Manager Logs

Check:

    kubectl -n longhorn-system get pods | grep instance-manager

Logs:

    kubectl -n longhorn-system logs <instance-manager-pod> --tail=100

Instance Manager issues may affect:

    Engine startup.
    Replica startup.
    Volume attach.
    Replica rebuild.

---

### 18.4 kubelet Logs

Execute on the target node:

    journalctl -u kubelet --since "30 minutes ago" | tail -200

Focus on searching:

    MountVolume
    FailedMount
    AttachVolume
    DetachVolume
    CSI
    iscsi
    longhorn

---

### 18.5 containerd Logs

Execute on the target node:

    journalctl -u containerd --since "30 minutes ago" | tail -200

Monitor:

    Container startup failure.
    Image pull failure.
    Runtime exceptions.
    Sandbox creation failure.

---

## NineteenI don't know.Production Fault Handling Process

### 19.1 Fault Discovery

Discovery sources:

    Grafana alerts
    Alertmanager alerts
    Longhorn UI
    kubectl inspection
    Business feedback
    Pod anomalies
    PVC Pending
    Volume Degraded
    Backup failure

---

### 19.2 Fault Tiering

| Tier | Phenomenon | Recommended Actions |
|---|---|---|
| P0 | Volume Faulted, business unavailable | Immediate response, preserve scene, assess Restore |
| P1 | Multiple Volume Degraded, high business risk | Immediate handling, confirm Backup |
| P2 | Single Volume Degraded, but business available | Restore replica as soon as possible |
| P3 | Backup failure, but business normal | Investigate Backup Target |
| P4 | Single test PVC Pending | Routine troubleshooting |

---

### 19.3 Handling Steps

Standard steps:

    1. Confirm impact scope.
    2. Confirm business read/write availability.
    3. Confirm PVC / PV / Volume mapping.
    4. Confirm Volume status.
    5. Confirm Replica status.
    6. Confirm node status.
    7. Confirm disk status.
    8. Confirm iscsid / kubelet / containerd.
    9. Check Longhorn Manager logs.
    10. Check Kubernetes Events.
    11. Confirm Backup availability.
    12. Decide on node recovery, replica rebuild, or Restore.
    13. Validate business after handling.
    14. Output fault post-mortem.

---

### 19.4 Fault Record Template

| Item | Content |
|---|---|
| Fault Time |  |
| Discovery Method | Alert / Inspection / Business Feedback |
| Affected Business |  |
| Namespace |  |
| PVC |  |
| PV |  |
| Longhorn Volume |  |
| Volume Status | Healthy / Degraded / Faulted |
| Replica Status |  |
| Affected Nodes |  |
| Impact Read | Yes / No |
| Impact Write | Yes / No |
| Backup Available | Yes / No |
| Handling Actions |  |
| Recovery Time |  |
| Root Cause |  |
| Improvement Measures |  |

---

## TwentyI don't know.Monitoring Alert Recommendations

### 20.1 Mandatory Alert Items

Production mandatory alerts:

    Longhorn Volume Faulted.
    Longhorn Volume Degraded beyond specified time.
    Replica Rebuild long time not completed.
    Longhorn Node Down.
    Longhorn Disk unavailable.
    Longhorn Disk space insufficient.
    longhorn-manager Pod anomaly.
    CSI component anomaly.
    Instance Manager anomaly.
    Backup failure.
    Backup Target unavailable.
    Node DiskPressure.
    Node NotReady.

---

### 20.2 Capacity Alerts

Recommended thresholds:

| Usage | Action |
|---|---|
| 70% | Monitor trend |
| 80% | Prepare expansion |
| 85% | Define expansion window |
| 90% | High priority handling |
| 95% | Severe risk |

---

### 20.3 Inspection Commands

Daily inspection: /think

kubectl -n longhorn-system get volumes.longhorn.io -o wide  
kubectl -n longhorn-system get replicas.longhorn.io -o wide  
kubectl -n longhorn-system get nodes.longhorn.io  
kubectl get pvc -A  
kubectl get nodes  
kubectl get events -A --sort-by=.lastTimestamp | tail -100  

Node Inspection:  

    df -hT /data/longhorn  
    systemctl is-active iscsid  
    systemctl is-active kubelet  
    systemctl is-active containerd  

---

## 21. High-Risk Operation Warnings  

The following operations must be approached with caution in production environments:  

    Delete PVC  
    Delete PV  
    Delete Longhorn Volume  
    Delete Replica  
    Delete Engine  
    Delete Snapshot  
    Delete Backup  
    Delete longhorn-system namespace  
    Delete Longhorn CRD  
    Delete /data/longhorn  
    Format Longhorn data disk  
    Force detach Volume  
    Force delete abnormal Pod  
    Upgrade Longhorn during Volume Degraded  
    Perform node failure simulation without backup  

Before execution, confirm:  

    Which business will be affected.  
    Whether there is an available Backup.  
    Whether there is a Snapshot.  
    Whether there is a maintenance window.  
    Whether there is a rollback plan.  
    Whether on-site information has been exported.  
    Whether business confirmation has been obtained.  
    Whether dual-person verification has been completed.  

---

## 22. Experiment Cleanup  

### 22.1 Delete Error PVC  

Execute:  

    kubectl delete -f 03-pvc-wrong-sc.yaml --ignore-not-found=true  

---

### 22.2 Delete 3-Replica Scheduling Failure Experiment Resources  

Execute:  

    kubectl delete -f 05-pvc-replica-3-debug.yaml --ignore-not-found=true  
    kubectl delete -f 04-sc-replica-3.yaml --ignore-not-found=true  

---

### 22.3 Delete Main Test Pod  

Execute:  

    kubectl delete -f 02-pod.yaml --ignore-not-found=true  

---

### 22.4 Delete Main Test PVC  

High-Risk Warning:  

    Deleting PVC may delete Longhorn Volume.  
    This action only deletes experimental resources.  

Execute:  

    kubectl delete -f 01-pvc.yaml --ignore-not-found=true  

Check:  

    kubectl get pvc -n longhorn-troubleshoot-demo  
    kubectl get pv  
    kubectl -n longhorn-system get volumes.longhorn.io  

---

### 22.5 Delete Namespace  

After confirming no resources remain:  

    kubectl delete namespace longhorn-troubleshoot-demo  

---

### 22.6 Delete Local Files  

Execute:  

    rm -f 01-pvc.yaml  
    rm -f 02-pod.yaml  
    rm -f 03-pvc-wrong-sc.yaml  
    rm -f 04-sc-replica-3.yaml  
    rm -f 05-pvc-replica-3-debug.yaml  
    rm -f collect-longhorn-baseline.sh  
    rm -f longhorn-baseline-*.log  

---

## 23. Completion Criteria for This Article  

After completing this article, the following standards should be met:  

| Item | Standard |  
|---|---|  
| Baseline Collection | Ability to collect PVC, PV, Volume, Replica, Events, and logs |  
| PVC Pending | Ability to simulate and troubleshoot using an incorrect StorageClass |  
| Pod Mount Failure | Mastery of FailedMount troubleshooting path |  
| Multi-Attach | Understanding of RWO multi-node mount conflicts |  
| Volume Degraded | Ability to view Volume and Replica status |  
| Replica Rebuild | Ability to observe rebuild status and logs |  
| Node Abnormality | Ability to analyze from Node, kubelet, and Longhorn Node |  
| DiskPressure | Ability to locate system disk and Longhorn data disk issues |  
| Backup Target | Ability to troubleshoot NFS/S3 backup target anomalies |  
| Logs | Ability to view manager, CSI, instance-manager, and kubelet logs |  
| High-Risk Operations | Clear understanding that PVC/Volume/Replica cannot be deleted arbitrarily |  
| Production Process | Ability to form fault records and handling procedures |  

---

## 24. Interview Answer Framework  

If asked in an interview:  

    How would you troubleshoot Longhorn Volume Degraded or Pod mount failure?  

You could respond:

I will first perform a layered troubleshooting without directly deleting PVC or Volume. The first step is to confirm the status of business Pods, such as using `kubectl get pod -o wide` and `kubectl describe pod` to check for events like FailedMount, FailedAttachVolume, and Multi-Attach.

If it's a PVC issue, I will check `kubectl get pvc`, `describe pvc`, `get pv`, and `describe pv` to confirm whether the PVC is Bound, whether the StorageClass is longhorn, and whether the CSI Driver of the PV is correct. If the PVC is Pending, I will focus on checking the StorageClass, CSI provisioner, longhorn-manager, node disks, and replica scheduling conditions.

If the Pod is in ContainerCreating or FailedMount state, I will check the iscsid on the node where the Pod is located, because Longhorn volume mounting depends on iSCSI capabilities. On the node, I will check `systemctl status iscsid`, `iscsiadm --version`, and kubelet logs via `journalctl -u kubelet` to confirm whether the mounting or attach failed.

If the Volume is Degraded, I will check `volumes.longhorn.io`, `replicas.longhorn.io`, and `engines.longhorn.io` under the longhorn-system namespace to confirm the Volume's robustness, replica count and distribution, and whether there are failed or rebuilding replicas. Then I will continue to check the status of Longhorn Node and Disk to see if the node is Ready, whether there is DiskPressure, whether there is space in `/data/longhorn`, and whether the Longhorn Node allows scheduling.

If there is a node anomaly, I will check `kubectl get nodes`, `describe node`, kubelet, containerd, iscsid, disk, and kernel logs. After the node recovers, I will observe whether Longhorn automatically performs Replica Rebuild and whether the Volume returns from Degraded to Healthy.

If the Volume has already Faulted, I will first preserve the scene, export PVC, PV, Volume, Replica, Events, and longhorn-manager logs to confirm whether there is a Backup. In production, it is recommended to first restore from Backup to a new Volume, then create a new PVC and validate the Pod to check data, instead of directly performing random operations on the original Volume.

The overall principle is: first confirm the impact scope, then locate the relationship between PVC/PV/Volume/Replica/Node; first preserve the scene, then decide whether to recover the node, wait for rebuilding, or restore from backup; Replica is for high availability, not backup, so critical data must have Backup Target and recovery drills.

---

## 25. Summary of This Article

This article completes the learning of Longhorn troubleshooting:

1. Longhorn troubleshooting should be divided into business layer, Kubernetes layer, CSI layer, Longhorn control plane, data plane, node layer, and backup layer.
2. PVC Pending should focus on checking StorageClass, CSI, longhorn-manager, node disks, and replica scheduling.
3. Pod ContainerCreating should focus on checking FailedMount, FailedAttachVolume, iscsid, CSI Plugin, and kubelet logs.
4. Multi-Attach is commonly seen in RWO PVC being used simultaneously by Pods on multiple nodes.
5. Volume Degraded indicates insufficient replicas, anomalies, or ongoing rebuilding.
6. Degraded is not a normal state and should not be ignored for a long time.
7. Replica Rebuild will consume disk, network, and CPU resources.
8. If Replica Rebuild does not end for a long time, check disk, network, node status, and manager logs.
9. Node NotReady may lead to Replica anomalies, Volume Degraded, and failed Pod rescheduling.
10. DiskPressure will affect PVC creation, Replica scheduling, and replica rebuilding.
11. Longhorn data directory should not be manually deleted or modified directly.
12. iscsid is an important check item for Longhorn mounting troubleshooting.
13. longhorn-manager logs are an important entry point for troubleshooting control plane issues.
14. CSI component logs are an important entry point for troubleshooting PVC creation and mounting issues.
15. Instance Manager anomalies may affect Engine and Replica.
16. When Volume is Faulted, the first principle is to preserve the scene.
17. When there is a Backup, it is recommended to restore to a new Volume for verification first.
18. Without Backup, do not arbitrarily delete Volume or Replica.
19. Production must establish monitoring alerts and fault record templates.
20. The next article will learn about Longhorn performance and production recommendations: disk, network, scheduling, and resource limits.

---

## 26. Reference Documents

Longhorn Official Documentation:

    https://longhorn.io/docs/latest/

Longhorn Troubleshooting:

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