# Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting and Data Persistence

Suggested Path: 05-Storage/03-LongHorn/05-Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting and Data Persistence.md

Tags: #Longhorn #Kubernetes #PVC #PV #StorageClass #CSI #PodMount #DataSustainability #StatefulSet #BlockStorage #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the fifth article of the Longhorn module, focusing on verifying Longhorn's dynamic volume capabilities through real Kubernetes resources.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management

This article officially enters Longhorn core operations:

    Create PVC
    Observe PV Auto-Creation
    Create Pod Mounting PVC
    Write Data to Mounted Directory
    Delete Pod
    Rebuild Pod
    Verify Data Still Exists
    Check Longhorn Volume, Engine, Replica Status
    Understand Differences Between Deployment and StatefulSet Using Longhorn
    Master Basic Troubleshooting for PVC Pending, Pod Mount Failure, Volume Degraded

This article's focus is not "writing a YAML", but understanding the complete chain:

    PVC -> StorageClass -> CSI -> Longhorn Volume -> PV -> Pod Mounting -> Data Persistence

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Create PVC using Longhorn StorageClass.
2. Understand how PVC triggers dynamic PV creation.
3. Understand the relationship between PVC, PV, and Longhorn Volume.
4. Create Pod mounting Longhorn PVC.
5. Write data to the mounted directory in Pod.
6. Verify PVC and Volume still exist after Pod deletion.
7. Verify data still exists after Pod rebuild.
8. View Longhorn Volume, Engine, Replica objects.
9. Understand RWO volume mounting limitations.
10. Understand differences between Deployment and StatefulSet using PVC.
11. Troubleshoot PVC Pending.
12. Troubleshoot Pod long ContainerCreating.
13. Troubleshoot FailedMount, FailedAttachVolume.
14. Understand PVC deletion risks from a production perspective.
15. Lay the foundation for subsequent replication mechanisms, backup recovery, and fault troubleshooting.

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

Longhorn Data Directory Planning:

    /data/longhorn

Check Commands:

    kubectl -n longhorn-system get pods -o wide
    kubectl get sc
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 3.3 Experiment Namespace

This article uses the experiment namespace:

    longhorn-volume-demo

All test PVCs, Pods, Deployments, StatefulSets are placed in this namespace.

---

## IV. Dynamic Volume Complete Chain

### 4.1 Longhorn Dynamic Volume Chain

Complete process:

    User creates PVC
        |
        v
    PVC specifies storageClassName: longhorn
        |
        v
    Kubernetes finds Longhorn StorageClass
        |
        v
    CSI Provisioner calls Longhorn
        |
        v
    Longhorn creates Volume
        |
        v
    Kubernetes automatically creates PV
        |
        v
    PVC and PV bind
        |
        v
    Pod mounts PVC
        |
        v
    kubelet calls CSI to complete attach/mount
        |
        v
    Application writes data to mounted directory
        |
        v
    Data enters Longhorn Engine
        |
        v
    Engine writes to multiple Replicas
        |
        v
    Replicas stored in node-local data directory

---

### 4.2 What This Article Verifies

This article verifies:

    1. Whether PVC can be automatically Bound.
    2. Whether PV is automatically generated.
    3. Whether Longhorn Volume is automatically generated.
    4. Whether Pod can mount PVC.
    5. Whether data written by Pod is persistent.
    6. Whether data still exists after Pod deletion.
    7. Whether old data can be read after Pod rebuild.
    8. Whether Longhorn internal Volume, Replica are normal.

---

## V. Pre-Practice Checks

### 5.1 Check Longhorn Components

Execute:

    kubectl -n longhorn-system get pods -o wide

Requirement:

longhorn-manager Running  
longhorn-csi-plugin Running  
csi-provisioner Running  
csi-attacher Running  
csi-resizer Running  
instance-manager Running  
longhorn-ui Running  

Check for anomalies:  

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"  

If anomalies exist, troubleshoot first:  

    kubectl -n longhorn-system describe pod <pod-name>  
    kubectl -n longhorn-system logs <pod-name> --tail=100  
    kubectl get events -n longhorn-system --sort-by=.lastTimestamp | tail -100  

---

### 5.2 Check StorageClass  

Execute:  

    kubectl get sc  

Confirm existence:  

    longhorn  

View details:  

    kubectl describe sc longhorn  

Focus on:  

    Provisioner  
    ReclaimPolicy  
    VolumeBindingMode  
    AllowVolumeExpansion  
    Parameters  
    numberOfReplicas  

Example focus:  

    Provisioner: driver.longhorn.io  

Explanation:  

    If no longhorn StorageClass exists, PVCs cannot dynamically create PVs via Longhorn.  
    Return to the installation guide to verify if Helm installation was completed.  

---

### 5.3 Check Node Status  

Execute:  

    kubectl get nodes -o wide  

Check node pressure:  

    kubectl describe nodes | grep -E "Name:|DiskPressure|MemoryPressure|PIDPressure|Ready"  

Requirements:  

    Ready=True  
    DiskPressure=False  
    MemoryPressure=False  
    PIDPressure=False  

If nodes have DiskPressure, do not proceed with creating Longhorn PVCs.  

---

### 5.4 Check iscsid  

Execute on each Worker node:  

    systemctl is-active iscsid  
    iscsiadm --version  

If iscsid is not running:  

    systemctl enable --now iscsid  

Explanation:  

    PVC Bound does not guarantee Pod mounting success.  
    Pod mounting Longhorn Volume requires normal iSCSI capabilities on the node.  

---

### 5.5 Check Longhorn Nodes and Disks  

Execute:  

    kubectl -n longhorn-system get nodes.longhorn.io  

View details:  

    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01  
    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker02  

Focus on:  

    Allow Scheduling  
    Conditions  
    Disk Path  
    Storage Available  
    Storage Scheduled  
    Storage Maximum  

Node-side checks:  

    df -hT /data/longhorn  
    lsblk -f  

---

## SixI don't know.Practical Step 1: Create an Experimental Namespace  

### 6.1 Create Namespace  

Execute:  

    kubectl create namespace longhorn-volume-demo  

View:  

    kubectl get ns longhorn-volume-demo  

---

### 6.2 Create Experimental Directory  

Create YAML directory on the management node:  

    mkdir -p ~/longhorn-volume-demo  
    cd ~/longhorn-volume-demo  

---

## SevenI don't know.Practical Step 2: Create PVC and Observe PV Auto-creation  

### 7.1 Create PVC YAML  

Create file:  

    cat > 01-pvc-rwo.yaml <<'EOF'  
    apiVersion: v1  
    kind: PersistentVolumeClaim  
    metadata:  
      name: data-pvc  
      namespace: longhorn-volume-demo  
    spec:  
      accessModes:  
        - ReadWriteOnce  
      storageClassName: longhorn  
      resources:  
        requests:  
          storage: 1Gi  
    EOF  

Explanation:  

    accessModes: ReadWriteOnce indicates the volume is typically mounted read-write by one node only.  
    storageClassName: longhorn indicates using Longhorn to dynamically create Volume.  
    storage: 1Gi indicates requesting 1Gi capacity.  

---

### 7.2 Apply PVC  

Execute:  

    kubectl apply -f 01-pvc-rwo.yaml  

View PVC:  

    kubectl get pvc -n longhorn-volume-demo  

Expected:  

    NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS  
    data-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            longhorn  

---

### 7.3 View PVC Details  

Execute:  

    kubectl describe pvc data-pvc -n longhorn-volume-demo  

Focus on:

Status  
Volume  
StorageClass  
Capacity  
Access Modes  
Events  

If the status is **Bound**, it indicates the PVC has been successfully bound to a PV.  

If the status is **Pending**, check the errors in the **Events**.  

---

### 7.4 View PV  

Execute:  

    kubectl get pv  

Find the corresponding PV:  

    kubectl get pv | grep data-pvc  

Check details:  

    kubectl describe pv <pv-name>  

Focus on:  

    StorageClass  
    Claim  
    Reclaim Policy  
    Status  
    CSI Driver  
    VolumeHandle  
    VolumeAttributes  

Notes:  

    PV is a persistent volume object at the Kubernetes layer.  
    Longhorn Volume is the actual data volume managed internally by Longhorn.  
    They are associated through the CSI field of the PV.  

---

### 7.5 View Longhorn Volume  

Execute:  

    kubectl -n longhorn-system get volumes.longhorn.io  

Alternatively:  

    kubectl -n longhorn-system get volumes.longhorn.io -o wide  

Check details:  

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>  

Focus on:  

    State  
    Robustness  
    Size  
    Number Of Replicas  
    Current Node  
    Kubernetes Status  
    Conditions  

Expected:  

    state may be **detached** or **attached**.  
    If no Pod is using the PVC yet, the Volume is typically **detached**.  
    When a Pod mounts, the Volume will **attach** to the node where the Pod resides.  

---

## VIII. Hands-on Practice 3: Create Pod with PVC Mounted  

### 8.1 Create Pod YAML  

Create file:  

    cat > 02-pod-with-pvc.yaml <<'EOF'  
    apiVersion: v1  
    kind: Pod  
    metadata:  
      name: pvc-test-pod  
      namespace: longhorn-volume-demo  
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
            claimName: data-pvc  
    EOF  

Notes:  

    The Pod mounts the **data-pvc** to the container's **/data**.  
    Subsequent writes to **/data** will be persisted.  
    If the **busybox:1.36** image fails to pull, you can sync it to your Harbor or Alibaba Cloud registry and replace the **image** field.  
    Do not resolve image issues by breaking the containerd global configuration.  

---

### 8.2 Apply Pod  

Execute:  

    kubectl apply -f 02-pod-with-pvc.yaml  

Check status:  

    kubectl get pod -n longhorn-volume-demo -o wide  

Expected:  

    pvc-test-pod Running  

If the Pod remains in **ContainerCreating** for a long time:  

    kubectl describe pod pvc-test-pod -n longhorn-volume-demo  
    kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp  
    kubectl get events -A --sort-by=.lastTimestamp | tail -100  

---

### 8.3 Check Pod Mount Information  

Execute:  

    kubectl describe pod pvc-test-pod -n longhorn-volume-demo  

Focus on:  

    Volumes  
    Mounts  
    Events  

You should see:  

    **data-pvc** mounted to **/data**  

---

### 8.4 Check Pod's Node  

Execute:  

    kubectl get pod pvc-test-pod -n longhorn-volume-demo -o wide  

Record:  

    NODE  

Example:  

    pvc-test-pod Running 10.244.x.x k8s-worker01  

Notes:  

    The Longhorn Volume will **attach** to the node where the Pod resides.  
    RWO volumes are typically used in read-write mode by only one node at a time.  

---

## IX. Hands-on Practice 4: Write Data and Verify Persistence  

### 9.1 Check Mounted Directory in Pod  

Execute:  

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- df -hT /data  

Check directory contents:  

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- ls -lah /data  

---

### 9.2 Write Test File  

Execute:  

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "echo 'hello longhorn pvc' > /data/hello.txt"  

Record write time:  

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "date > /data/create-time.txt"  

Write node information: /think

kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "hostname > /data/pod-hostname.txt"

View:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- ls -lah /data
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/hello.txt
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/create-time.txt
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/pod-hostname.txt

---

### 9.3 Creating Multi-Level Directories and Files

Execute:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "mkdir -p /data/app/logs /data/app/config"

Write configuration file:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "echo 'app_name=longhorn-demo' > /data/app/config/app.conf"

Write log:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- sh -c "echo 'log line 1' > /data/app/logs/app.log"

View:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- find /data -maxdepth 4 -type f -print

---

### 9.4 Viewing Longhorn Volume Status

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

View details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Focus on:

    Current Node
    State
    Robustness
    Number Of Replicas
    Kubernetes Status

Expected:

    Volume is attached to the node where the Pod resides.
    Robustness should be healthy or similar healthy status.

---

### 9.5 Viewing Replica Distribution

Execute:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

View details:

    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Focus on:

    Node ID
    Disk ID
    Data Path
    State
    Healthy At

Note:

    After data is written to /data, Longhorn will write the Replica through the Engine.
    The actual data of the Replica is located in the Longhorn node data directory; manual modification is not recommended.

---

## TenI don't know.Practice Five: Verifying Data Persistence After Pod Deletion

### 10.1 Deleting Pod

Execute:

    kubectl delete pod pvc-test-pod -n longhorn-volume-demo

View:

    kubectl get pod -n longhorn-volume-demo
    kubectl get pvc -n longhorn-volume-demo
    kubectl get pv

Expected:

    Pod is deleted.
    PVC still exists.
    PV still exists.
    Longhorn Volume still exists.

---

### 10.2 Viewing Volume Status

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Note:

    After Pod deletion, the Volume may change from attached to detached.
    Data remains in the Longhorn Volume.
    Deleting a Pod does not equal deleting a PVC.
    Deleting a Pod does not equal deleting a Longhorn Volume.

---

### 10.3 Rebuilding Pod

Reapply Pod YAML:

    kubectl apply -f 02-pod-with-pvc.yaml

View:

    kubectl get pod -n longhorn-volume-demo -o wide

Wait for Running.

---

### 10.4 Verifying Old Data Still Exists

Execute:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/hello.txt
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/create-time.txt
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/pod-hostname.txt
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/app/config/app.conf
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/app/logs/app.log

Expected:

    All files still exist.

Conclusion:

    After Pod deletion and recreation, data is not lost.
    Data is stored in the Longhorn Volume corresponding to the PVC.
    Pod is merely a runtime instance consuming the Volume; it is not the data itself.

---

## ElevenI don't know.Practice Six: Deleting Pod and Rescheduling to Other Nodes

### 11.1 Why Verify Cross-Node

One of Longhorn's values is:

    Even if a Pod is rescheduled to another node, it can remount the original Longhorn Volume.

Note:

# RWO Volume Rules

A RWO volume can be mounted in read-write mode by only one node at a time.
The original Pod must first release the mount.
A new Pod can then attach and mount on another node.
The node-side iscsid must be functioning normally.

---

### 11.2 Record Current Pod Node

Execute:

    kubectl get pod pvc-test-pod -n longhorn-volume-demo -o wide

Assume the current node is:

    k8s-worker01

---

### 11.3 Schedule to Specified Node via nodeSelector

Delete the current Pod:

    kubectl delete pod pvc-test-pod -n longhorn-volume-demo

Create the Pod YAML for the specified node.

If scheduling to k8s-worker02:

    cat > 03-pod-with-pvc-on-worker02.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: pvc-test-pod
      namespace: longhorn-volume-demo
    spec:
      nodeSelector:
        kubernetes.io/hostname: k8s-worker02
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
          claimName: data-pvc
    EOF

Apply:

    kubectl apply -f 03-pod-with-pvc-on-worker02.yaml

Check:

    kubectl get pod pvc-test-pod -n longhorn-volume-demo -o wide

---

### 11.4 Verify Data Persistence Across Nodes

Execute:

    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/hello.txt
    kubectl exec -n longhorn-volume-demo pvc-test-pod -- cat /data/app/config/app.conf

Expected:

    Data remains readable.

Check Longhorn Volume current node:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Note:

    The Volume should attach to the new Pod's node.
    If attachment fails, focus on checking iscsid, CSI, Longhorn Engine, node status, and events.

---

## 12. Practical Exercise Seven: Using Deployment to Mount the Same RWO PVC

### 12.1 Risks of Using PVC with Deployment

Deployment is typically suitable for stateless applications.

If a Deployment uses a RWO PVC, note:

    replicas can only safely be set to 1.
    Multi-replica Deployments sharing the same RWO PVC may cause scheduling and mounting conflicts.
    A RWO volume cannot be mounted in read-write mode by multiple nodes simultaneously.
    For stateful applications like databases and middleware, StatefulSet is recommended.

---

### 12.2 Delete Current Test Pod

Execute:

    kubectl delete pod pvc-test-pod -n longhorn-volume-demo --ignore-not-found=true

---

### 12.3 Create Single-Replica Deployment

Create file:

    cat > 04-deployment-single-replica.yaml <<'EOF'
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: pvc-demo-deploy
      namespace: longhorn-volume-demo
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: pvc-demo-deploy
      template:
        metadata:
          labels:
            app: pvc-demo-deploy
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
                claimName: data-pvc
    EOF

Apply:

    kubectl apply -f 04-deployment-single-replica.yaml

Check:

kubectl get deploy -n longhorn-volume-demo
kubectl get pod -n longhorn-volume-demo -o wide

---

### 12.4 Verify Deployment Pod Reads Old Data

Get Pod name:

    POD_NAME=$(kubectl get pod -n longhorn-volume-demo -l app=pvc-demo-deploy -o jsonpath='{.items[0].metadata.name}')

View:

    echo $POD_NAME

Read data:

    kubectl exec -n longhorn-volume-demo $POD_NAME -- cat /data/hello.txt

Write new data:

    kubectl exec -n longhorn-volume-demo $POD_NAME -- sh -c "echo 'write from deployment' > /data/deployment.txt"

View:

    kubectl exec -n longhorn-volume-demo $POD_NAME -- cat /data/deployment.txt

---

### 12.5 Attempt to Scale Deployment to 2 Replicas

Warning:

    This experiment is used to observe the conflict between RWO PVC and multi-replica Deployment.
    Only execute in the experimental namespace.

Execute:

    kubectl scale deploy pvc-demo-deploy -n longhorn-volume-demo --replicas=2

View:

    kubectl get pod -n longhorn-volume-demo -o wide
    kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp | tail -50

Possible phenomena:

    If two Pods are scheduled to the same node, they may still mount.
    If scheduled to different nodes, Multi-Attach, FailedAttachVolume, or mounting failure may occur.
    Actual behavior depends on Kubernetes scheduling, volume access mode, and node location.

Conclusion:

    It is not recommended to share the same RWO PVC among multiple Deployment replicas.
    Stateful multi-replica applications should use StatefulSet + volumeClaimTemplates, allowing each Pod to have an independent PVC.

Restore to 1 replica:

    kubectl scale deploy pvc-demo-deploy -n longhorn-volume-demo --replicas=1

---

## ThirteenI don't know.Hands-on Eight: StatefulSet and volumeClaimTemplates Basics

### 13.1 Why StatefulSet is More Suitable for Stateful Applications

StatefulSet is suitable for:

    Stable network identity
    Stable storage
    Ordered startup
    Ordered deletion
    Each replica has independent PVC

Typical applications:

    MySQL
    PostgreSQL
    Redis
    ZooKeeper
    Kafka
    Elasticsearch
    Prometheus
    Stateful middleware

Issues with Deployment + single PVC:

    Multiple replicas sharing the same RWO PVC may cause conflicts.
    Pod names are unstable.
    Storage identity is unstable.
    Not suitable for stateful replica scaling.

---

### 13.2 Clean Up Deployment

Execute:

    kubectl delete -f 04-deployment-single-replica.yaml

Note:

    Deleting Deployment does not delete data-pvc.
    data-pvc still exists.

---

### 13.3 Create StatefulSet Example

Create file:

    cat > 05-statefulset-volumeclaimtemplates.yaml <<'EOF'
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: web
      namespace: longhorn-volume-demo
    spec:
      serviceName: web
      replicas: 2
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          containers:
            - name: app
              image: busybox:1.36
              imagePullPolicy: IfNotPresent
              command:
                - sh
                - -c
                - "while true; do echo $(hostname) $(date) >> /data/app.log; sleep 30; done"
              volumeMounts:
                - name: data
                  mountPath: /data
      volumeClaimTemplates:
        - metadata:
            name: data
          spec:
            accessModes:
              - ReadWriteOnce
            storageClassName: longhorn
            resources:
              requests:
                storage: 1Gi
    EOF

Explanation:

    Each StatefulSet Pod automatically creates its own PVC.
    web-0 uses data-web-0.
    web-1 uses data-web-1.
    This is more reasonable than multiple Pods sharing a single RWO PVC.

---

### 13.4 Apply StatefulSet

Execute:

kubectl apply -f 05-statefulset-volumeclaimtemplates.yaml

Check:

    kubectl get sts -n longhorn-volume-demo
    kubectl get pod -n longhorn-volume-demo -o wide
    kubectl get pvc -n longhorn-volume-demo
    kubectl get pv

Expected PVC:

    data-web-0
    data-web-1

---

### 13.5 Verify Each Pod Has Independent Data

Check web-0:

    kubectl exec -n longhorn-volume-demo web-0 -- sh -c "hostname && tail -5 /data/app.log"

Check web-1:

    kubectl exec -n longhorn-volume-demo web-1 -- sh -c "hostname && tail -5 /data/app.log"

Explanation:

    web-0 and web-1 each write to their own PVC.
    They will not share the same RWO volume.
    This is the correct direction for most stateful multi-replica applications.

---

### 13.6 Check Longhorn Volume Count

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io

You should see:

    The Volume corresponding to data-pvc
    The Volume corresponding to data-web-0
    The Volume corresponding to data-web-1

Check Replica:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

## Fourteen, Relationship Between PVC, PV, and Volume

### 14.1 Check PVC

Execute:

    kubectl get pvc -n longhorn-volume-demo -o wide

---

### 14.2 Check PV

Execute:

    kubectl get pv

Find PV based on PVC:

    kubectl describe pvc data-pvc -n longhorn-volume-demo

Check among:

    Volume:

Then execute:

    kubectl describe pv <pv-name>

---

### 14.3 Check Longhorn Volume

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io

Generally, the name of a Longhorn Volume corresponds to the volume handle of PV/PVC.

Check details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

---

### 14.4 Relationship Summary

A typical relationship:

    Namespace: longhorn-volume-demo
    PVC: data-pvc
      |
      v
    PV: pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      |
      v
    Longhorn Volume: pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      |
      v
    Engine + Replicas
      |
      v
    Data under /data/longhorn or /var/lib/longhorn

---

## Fifteen, Common Troubleshooting

### 15.1 PVC Stays in Pending State

Check:

    kubectl get pvc -n longhorn-volume-demo
    kubectl describe pvc data-pvc -n longhorn-volume-demo
    kubectl get sc
    kubectl describe sc longhorn
    kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp

Troubleshoot Longhorn:

    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200
    kubectl -n longhorn-system get nodes.longhorn.io

Common causes:

    The longhorn StorageClass does not exist.
    CSI Provisioner is abnormal.
    longhorn-manager is abnormal.
    Nodes lack schedulable disks.
    Replica count exceeds available node count.
    Insufficient disk space.
    Longhorn installation is incomplete.

---

### 15.2 Pod Stays in ContainerCreating State for a Long Time

Check:

    kubectl describe pod pvc-test-pod -n longhorn-volume-demo
    kubectl get events -n longhorn-volume-demo --sort-by=.lastTimestamp
    kubectl get events -A --sort-by=.lastTimestamp | tail -100

Focus on:

    FailedMount
    FailedAttachVolume
    Multi-Attach
    timeout
    iscsi
    mount

Node checks:

    systemctl status iscsid
    iscsiadm --version
    journalctl -u kubelet --since "30 minutes ago" | tail -100

Longhorn checks:

    kubectl -n longhorn-system get volumes.longhorn.io
    kubectl -n longhorn-system get engines.longhorn.io
    kubectl -n longhorn-system get replicas.longhorn.io
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

---

### 15.3 ImagePullBackOff

Check:

    kubectl describe pod pvc-test-pod -n longhorn-volume-demo

If busybox image pull fails:

    1. Pull busybox:1.36 on a machine with internet access.
    2. Tag it to your Harbor or Alibaba Cloud registry.
    3. Push it to the private registry.
    4. Modify the image field in the Pod YAML.
    5. Reapply the configuration.

Example:

    docker pull busybox:1.36
    docker tag busybox:1.36 registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:1.36
    docker push registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:1.36

Then modify:

    image: registry.cn-hangzhou.aliyuncs.com/pub-syq/busybox:1.36

Notes:

    Image issues are unrelated to Longhorn itself.
    Do not misdiagnose business Pod image pull failures as storage failures.

---

### 15.4 Multi-Attach Error

Symptoms:

    Multi-Attach error for volume

Common causes:

    RWO PVC being used by multiple Pods on different nodes.
    Old Pods not fully releasing the volume.
    Deployment with multiple replicas sharing the same RWO PVC.
    Node anomalies causing delayed volume detach.

Troubleshooting:

    kubectl get pod -n longhorn-volume-demo -o wide
    kubectl describe pod <pod-name> -n longhorn-volume-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Resolution:

    Confirm if multiple Pods are using the same RWO PVC.
    Reduce Deployment replicas to 1.
    Wait for old mounts to release.
    Check the Volume attach status in Longhorn UI.
    Do not forcibly delete the Volume.

---

### 15.5 Volume Degraded

Check:

    kubectl -n longhorn-system get volumes.longhorn.io
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Possible causes:

    A replica failed.
    A node is unavailable.
    Insufficient disk space.
    Insufficient replica count.
    Replicas are in the process of rebuilding.
    Insufficient nodes to satisfy replica distribution.

Resolution:

    Check node status.
    Check disk capacity.
    Check Longhorn Node.
    Wait for replica rebuilding.
    If degraded for a long time, enter the troubleshooting process.

---

## SixteenI don't know.Basic Principles for Using PVC in Production

### 16.1 Explicitly Specify StorageClass

Recommendation:

    storageClassName: longhorn

Reasons:

    Avoid misusing the default StorageClass.
    More clear when multiple storage systems coexist.
    Facilitates troubleshooting and migration.

---

### 16.2 StatefulSet Preferred for Stateful Multi-Replica

Recommendation:

    Single-replica general applications can use Deployment + PVC.
    Stateful multi-replica applications should prioritize StatefulSet + volumeClaimTemplates.
    Do not share a RWO PVC across multiple Pods.

---

### 16.3 PVC Should Not Be Deleted Arbitrarily

Deleting PVC may lead to:

    PV being deleted.
    Longhorn Volume being deleted.
    Data being deleted.

It depends on:

    StorageClass ReclaimPolicy
    PV Recycling Policy
    Longhorn Configuration

Production requirements:

    Confirm business ownership before deleting PVC.
    Confirm backups before deleting PVC.
    Confirm recovery plans before deleting PVC.
    Deleting PVC should go through an approval process.

---

### 16.4 Replica Is Not Backup

Longhorn multi-replica can handle:

    Node failure
    Disk failure
    Replica anomalies

Cannot handle:

    User rm -rf /data
    Application data corruption
    Accidental PVC deletion
    Accidental Volume deletion
    Cluster-wide failure

Therefore, production must:

    Configure Backup Target.
    Perform Volume Backup.
    Conduct recovery drills.

---

### 16.5 Do Not Use PVC as Object Storage

Longhorn PVC is suitable for:

    Database data directories
    Middleware data directories
    Application state directories

Not suitable for:

    Massive image objects
    Attachment object storage
    Large-scale backup package download services
    Log archive object storage

These are better suited for:

    MinIO
    RustFS
    Cloud vendor OSS / S3

---

## SeventeenI don't know.High-Risk Operation Warnings

The following operations must be handled with caution in production environments:

    Delete PVC
    Delete PV
    Delete Longhorn Volume
    Delete StatefulSet without confirming PVC
    Delete longhorn-system
    Delete /data/longhorn
    Modify StorageClass ReclaimPolicy
    Multiple replicas Deployment sharing a RWO PVC
    Test deletion recovery without backup
    Manually edit Longhorn data directory

Confirm before execution:

    Which business the PVC belongs to.
    Which Volume the PV corresponds to.
    Whether the Volume has backups.
    Whether there are snapshots.
    Whether there is a Backup Target.
    Whether recovery is possible after deletion.
    Whether business confirmation has been obtained.
    Whether it is within maintenance window.
    Whether there is a rollback plan.

---

## EighteenI don't know.Experiment Cleanup

### 18.1 Delete StatefulSet

Execute: /think

kubectl delete -f 05-statefulset-volumeclaimtemplates.yaml

Note:

- Deleting a StatefulSet will not automatically delete PVCs created by volumeClaimTemplates.
- Need to check PVCs separately.

View:

- kubectl get pvc -n longhorn-volume-demo

---

### 18.2 Deleting PVCs Created by StatefulSet

High-risk warning:

- The following command will delete experimental PVCs.
- Production environments should not batch delete this way.

Execute:

- kubectl delete pvc data-web-0 -n longhorn-volume-demo
- kubectl delete pvc data-web-1 -n longhorn-volume-demo

---

### 18,3 Deleting Deployment

If still exists:

- kubectl delete -f 04-deployment-single-replica.yaml --ignore-not-found=true

---

### 18.4 Deleting Pod

If still exists:

- kubectl delete pod pvc-test-pod -n longhorn-volume-demo --ignore-not-found=true

---

### 18.5 Deleting Main Test PVC

High-risk warning:

- Deleting PVC may delete Longhorn Volume.
- This section only cleans up experimental resources.

Execute:

- kubectl delete -f 01-pvc-rwo.yaml

View:

- kubectl get pvc -n longhorn-volume-demo
- kubectl get pv
- kubectl -n longhorn-system get volumes.longhorn.io

---

### 18.6 Deleting Namespace

After confirming no resources remain:

- kubectl delete namespace longhorn-volume-demo

---

### 18.7 Deleting Local YAML

Execute:

- rm -f 01-pvc-rwo.yaml
- rm -f 02-pod-with-pvc.yaml
- rm -f 03-pod-with-pvc-on-worker02.yaml
- rm -f 04-deployment-single-replica.yaml
- rm -f 05-statefulset-volumeclaimtemplates.yaml

---

## 19. Completion Standards for This Article's Hands-on Practice

After completing this article, the following standards should be met at least:

| Project | Standard |
|---|---|
| Namespace | longhorn-volume-demo has been created |
| PVC | data-pvc successfully Bound |
| PV | Automatically created Longhorn PV |
| Pod | pvc-test-pod successfully Running |
| Mount Directory | /data is readable and writable |
| Data Writing | hello.txt and similar files written successfully |
| Pod Deletion and Re-creation | Data still exists |
| Cross-node Re-creation | Data remains readable |
| Volume | Longhorn Volume can be viewed |
| Replica | Longhorn Replica can be viewed |
| Deployment Test | Understand single RWO PVC and multi-replica risks |
| StatefulSet Test | Understand volumeClaimTemplates for independent PVC |
| Cleanup | Experimental resources can be safely deleted |

---

## 20. Interview Answering Approach

If asked in an interview:

- How is PVC used in Longhorn? How to verify data persistence?

You can answer:

- After Longhorn installation, it provides a StorageClass typically named longhorn. When creating PVCs, specify storageClassName: longhorn. Kubernetes will dynamically create PVs via Longhorn CSI, and Longhorn backend will create corresponding Volume, Engine, and Replica.
- For verification, I'll first create a PVC with 1Gi and ReadWriteOnce. Then use kubectl get pvc and kubectl get pv to confirm PVC is Bound. Next, check volumes.longhorn.io and replicas.longhorn.io under longhorn-system to confirm Longhorn internal Volume and Replica creation.
- Then create a Pod mounting the PVC to /data, write files like echo hello > /data/hello.txt. Delete the Pod, recreate it using the same PVC, and check if /data/hello.txt still exists. If data remains, it indicates data is stored in Longhorn Volume, not Pod lifecycle.
- I'll also verify if the Pod can remount the same RWO Volume after scheduling to another node, confirming Longhorn's attach and mount functionality. Note that RWO volumes cannot be read/write mounted by multiple nodes, so Deployments using same RWO PVC usually have replica count of 1. StatefulSet is better for multi-replica stateful apps via volumeClaimTemplates for independent PVCs per Pod.
- In production, note that deleting Pods doesn't delete data, but deleting PVCs may trigger PV and Longhorn Volume deletion depending on ReclaimPolicy. Before deleting PVCs in production, confirm business ownership, backup, and recovery plans. Longhorn Replica isn't backup, critical data still needs Backup Target and recovery drills.

---

## 21. Summary of This Article

This article completed Longhorn dynamic volume practice:

1. PVC is a user's request for persistent storage.
2. StorageClass specifies the dynamic volume provisioning method.
3. Longhorn dynamically creates PVs through CSI.
4. After PVC is Bound, it corresponds to a Longhorn Volume.
5. Pods can mount PVCs to container directories.
6. Data written to the mounted directory enters the Longhorn Volume.
7. Deleting a Pod does not delete PVC.
8. Deleting a Pod does not delete the Longhorn Volume.
9. After Pod reconstruction, old data can still be read.
10. RWO volumes are unsuitable for multiple nodes to read/write mount simultaneously.
11. When using a single RWO PVC with Deployment, replicas should be cautiously set to 1.
12. Multi-replica stateful applications are better suited for StatefulSet + volumeClaimTemplates.
13. Each StatefulSet replica can have an independent PVC.
14. PVC, PV, Longhorn Volume, and Replica should be able to correspond.
15. PVC Pending should focus on checking StorageClass, CSI, Longhorn Manager, and node disks.
16. Pod ContainerCreating should focus on checking FailedMount, iscsid, CSI, and Volume Attach.
17. Volume Degraded should focus on checking Replica, nodes, and disks.
18. Deleting PVC is a high-risk operation; production must confirm backups and business ownership.
19. Longhorn Replica is not a backup; subsequent learning must focus on Backup Target.
20. The next article will learn about Longhorn's replica mechanism: Replica count, node distribution, and data high availability.

---

## Twenty-twoI don't know.Reference Documents

Longhorn Official Documentation:

    https://longhorn.io/docs/latest/

Longhorn Nodes and Volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn StorageClass Parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn Volumes and Nodes:

    https://longhorn.io/docs/latest/nodes-and-volumes/volumes/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Longhorn Backup and Recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes StatefulSet:

    https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

Kubernetes Deployment:

    https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Kubernetes CSI Documentation:

    https://kubernetes-csi.github.io/docs/