# Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability

Recommended path: 05-Storage/03-LongHorn/06-Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability.md

Tags: #Longhorn #Replica #CopyMechanism #NodeDistribution #VolumeDegraded #ReplicaRebuild #Kubernetes #PVC #BlockStorage #HighAvailable #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the sixth article of the Longhorn module, focusing on learning Longhorn's replica mechanism, node distribution, Volume Degraded, Replica Rebuild, and data high availability.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence

This article enters the core of Longhorn's data high availability:

    What does replica count mean
    How replicas are distributed across nodes
    What's the difference between 2 replicas and 3 replicas
    Why replica count isn't always better
    What happens to a Volume when a node fails
    How to understand Volume Healthy / Degraded / Faulted
    How to observe Replica Rebuild
    How replicas are replenished after node recovery
    Why the replica mechanism can't replace backup
    How to plan replica count, node count, disk capacity, and fault drills in production

This document emphasizes practical operations, using real PVC, Pod, StorageClass, and node experiments to observe Longhorn replica changes.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of Longhorn Replica.
2. Understand the relationship between replica count and node count.
3. Understand the use cases for 1 replica, 2 replicas, and 3 replicas.
4. Understand the impact of replica count on disk capacity and write performance.
5. Create a StorageClass with a specified replica count.
6. Create a PVC and verify the replica count of a Longhorn Volume.
7. View which nodes and disks replicas are distributed on.
8. Simulate Pod cross-node remounting.
9. Simulate Worker node failure.
10. Observe a Volume transitioning from Healthy to Degraded.
11. Recover a node and observe Replica Rebuild.
12. Judge whether a Degraded Volume can continue read/write operations.
13. Troubleshoot replica scheduling failures.
14. Understand why the replica mechanism can't prevent accidental deletion and data logical errors.
15. Establish a Longhorn production replica planning and fault drill process.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node Network Segment: 10.0.0.0/24

Node planning:

| IP | Hostname | Role | Notes |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Optional for experiment |
| 10.0.0.21 | k8s-worker01 | Worker | Longhorn data node |
| 10.0.0.22 | k8s-worker02 | Worker | Longhorn data node |

---

### 3.2 Longhorn Environment

Longhorn namespace:

    longhorn-system

Longhorn default StorageClass:

    longhorn

This article will create two experimental StorageClasses:

    longhorn-replica-1
    longhorn-replica-2

If the experimental environment allows the Master to participate in Longhorn data scheduling, you can also create:

    longhorn-replica-3

---

### 3.3 Experimental Namespace

This article uses the namespace:

    longhorn-replica-demo

Create directory:

    mkdir -p ~/longhorn-replica-demo
    cd ~/longhorn-replica-demo

---

## IV. Basic Understanding of Longhorn Replica

### 4.1 What is a Replica

A Replica is a data copy of a Longhorn Volume.

After a PVC is created by Longhorn, Longhorn will create multiple Replicas for the Volume based on the replica count configuration.

For example:

    PVC: data-pvc
      |
      v
    Longhorn Volume: pvc-xxxx
      |
      +--> Replica 1 on k8s-worker01
      +--> Replica 2 on k8s-worker02

If the replica count is 3:

    Longhorn Volume: pvc-xxxx
      |
      +--> Replica 1 on k8s-worker01
      +--> Replica 2 on k8s-worker02
      +--> Replica 3 on k8s-worker03

---

### 4.2 What Problems Does Replica Solve

Replica mainly solves:

    Single replica damage
    Single node failure
    Single data disk failure
    Node temporary offline
    Replica rebuild
    Improved Volume availability

Simple understanding:

    One replica is damaged, there are other replicas.
    One node is offline, there are replicas on other nodes.
    After node recovery, Longhorn attempts to replenish replicas.

---

### 4.3 What Problems Can't Replica Solve

Replica cannot solve:

    User rm -rf deletion of data inside Pod
    Application writing incorrect data
    Database logical damage
    Accidental deletion of PVC
    Accidental deletion of PV
    Accidental deletion of Longhorn Volume
    Entire Kubernetes cluster damage
    All replicas simultaneously damaged
    Ransomware simultaneously destroying multiple replicas
    Cross-cluster recovery when Backup Target doesn't exist

Conclusion:

Replica is a high availability capability.
Replica is not a backup.
Snapshot cannot be fully equivalent toOffshore Backup.
Production critical data must configure Backup Target and perform recovery drills.

---

## FiveI don't know.Replica Count Planning

### 5.1 1 Replica

1 Replica meaning:

    A Volume has only one Replica.

Advantages:

    Less disk usage.
    Lower write amplification.
    Lower performance overhead.
    Suitable for learning, temporary environments, and data that can be rebuilt.

Disadvantages:

    High risk for the Volume if the Replica's node fails.
    Single disk damage may lead to data unavailability.
    Lacks node-level high availability.

Suitable:

    Learning experiments
    Temporary testing
    Data that can be rebuilt
    Non-critical business

Not suitable:

    Production critical data
    Core databases
    Important middleware
    Environments without backup

---

### 5.2 2 Replicas

2 Replicas meaning:

    A Volume has two Replicas.

Advantages:

    Can handle single Replica or single node anomalies.
    Lower capacity overhead than 3 Replicas.
    Suitable for experimental clusters with only 2 Worker nodes.

Disadvantages:

    Lower availability than 3 Replicas.
    After one Replica fails, only one healthy Replica remains, increasing risk window.
    If the second Replica also fails during reconstruction, data may become unavailable.

Suitable:

    Experimental environments with 2 data nodes
    Medium-scale non-core business
    Scenarios sensitive to capacity
    Business with additional backup protection

---

### 5.3 3 Replicas

3 Replicas meaning:

    A Volume has three Replicas.

Advantages:

    More common production Replica configuration.
    Still has two Replicas after single node failure.
    Stronger fault tolerance.
    More suitable for 3 or more data nodes.

Disadvantages:

    Capacity overhead is approximately 3x.
    Higher synchronous write pressure.
    More network traffic.
    Longer Replica reconstruction time.
    Scheduling failures are likely when node count is insufficient.

Suitable:

    3 or more data nodes
    Important business PVC
    Medium-scale production environments
    Clusters with stable network and independent data disks

---

### 5.4 Current Experimental Recommendations

Current cluster:

    k8s-master01
    k8s-worker01
    k8s-worker02

If only using two Workers as Longhorn data nodes:

    Recommended Replica count for experiment: 2

If allowing Master to also participate as Longhorn data node:

    Can verify Replica count: 3

Production recommendations:

    Control Plane is generally not recommended to host storage Replicas.
    Production 3 Replicas should have at least 3 Worker data nodes.
    Replica count cannot be planned independently from node count.

---

## SixI don't know.Replica Count and Capacity Relationship

### 6.1 Capacity Amplification

Longhorn Replicas will amplify underlying disk usage.

| PVC Requested Capacity | Replica Count | Theoretical Raw Capacity Usage |
|---|---|---|
| 1Gi | 1 | Approximately 1Gi |
| 1Gi | 2 | Approximately 2Gi |
| 1Gi | 3 | Approximately 3Gi |
| 10Gi | 2 | Approximately 20Gi |
| 10Gi | 3 | Approximately 30Gi |
| 100Gi | 3 | Approximately 300Gi |

Notes:

    Actual usage is also affected by file system, snapshots, reconstruction, reserved space, sparse files, etc.
    Capacity planning cannot only consider PVC requested capacity.
    Must calculate comprehensively based on replica count, snapshot quantity, backup strategy, and growth rate.

---

### 6.2 Write Amplification

As replica count increases, write paths also increase.

For example 3 Replicas:

    Pod writes data
      |
      v
    Longhorn Engine
      |
      +--> Replica 1
      +--> Replica 2
      +--> Replica 3

Writing one piece of data requires synchronizing to multiple Replicas.

Impacts:

    Increased network traffic.
    Increased disk I/O.
    Potential increase in write latency.
    Increased pressure on Replica nodes.

Production understanding:

    Replica count is a trade-off between reliability, capacity, and performance.
    More is not always better.
    Should not blindly increase replica count for "looking safer".

---

## SevenI don't know.Pre-Operational Checks

### 7.1 Check Longhorn Components

Execute:

    kubectl -n longhorn-system get pods -o wide

Check for anomalies:

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If anomalies exist:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl -n longhorn-system logs <pod-name> --tail=100

---

### 7.2 Check StorageClass

Execute:

    kubectl get sc

Check default longhorn:

    kubectl describe sc longhorn

Focus on:

    provisioner
    reclaimPolicy
    volumeBindingMode
    allowVolumeExpansion
    numberOfReplicas

---

### 7.3 Check Longhorn Nodes

Execute:

    kubectl -n longhorn-system get nodes.longhorn.io

Check node details:

    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01
    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker02

Focus on:

    Allow Scheduling
    Ready
    Schedulable
    Disk Path
    Storage Available
    Storage Scheduled
    Conditions

---

### 7.4 Check Node Disks

On each data node execute:

    df -hT /data/longhorn
    lsblk -f
    du -sh /data/longhorn

If /data/longhorn is on the system disk:

    Only suitable for experiments.
    Not recommended for production critical business.

---

### 7.5 Check iscsid

On each Worker node execute:

    systemctl is-active iscsid
    iscsiadm --version

If not running: /think

systemctl enable --now iscsid

---

## Eight, Hands-on Exercise One: Creating an Experimental Namespace

### 8.1 Creating a Namespace

Run:

    kubectl create namespace longhorn-replica-demo

Check:

    kubectl get ns longhorn-replica-demo

---

### 8.2 Creating a Working Directory

Run:

    mkdir -p ~/longhorn-replica-demo
    cd ~/longhorn-replica-demo

---

## Nine, Hands-on Exercise Two: Creating a 1-Replica StorageClass

### 9.1 Creating a StorageClass

Create file:

    cat > 01-sc-longhorn-replica-1.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-replica-1
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    parameters:
      numberOfReplicas: "1"
      staleReplicaTimeout: "30"
      fsType: "ext4"
    EOF

Apply:

    kubectl apply -f 01-sc-longhorn-replica-1.yaml

Check:

    kubectl get sc
    kubectl describe sc longhorn-replica-1

---

### 9.2 Creating a 1-Replica PVC

Create file:

    cat > 02-pvc-replica-1.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-replica-1
      namespace: longhorn-replica-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-replica-1
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f 02-pvc-replica-1.yaml

Check:

    kubectl get pvc -n longhorn-replica-demo
    kubectl get pv

---

### 9.3 Viewing Longhorn Volume

Run:

    kubectl -n longhorn-system get volumes.longhorn.io

After finding the corresponding volume, view details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Key confirmation:

    Number Of Replicas: 1

---

### 9.4 Viewing Replica Count

Run:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

You can also filter by Volume name:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide | grep pvc-replica-1

If grep returns nothing, filter by volume-name:

    kubectl -n longhorn-system get replicas.longhorn.io -o yaml | grep -A5 -B5 "<volume-name>"

---

## Ten, Hands-on Exercise Three: Creating a 2-Replica StorageClass

### 10.1 Creating a StorageClass

Create file:

    cat > 03-sc-longhorn-replica-2.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-replica-2
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    parameters:
      numberOfReplicas: "2"
      staleReplicaTimeout: "30"
      fsType: "ext4"
    EOF

Apply:

    kubectl apply -f 03-sc-longhorn-replica-2.yaml

Check:

    kubectl get sc
    kubectl describe sc longhorn-replica-2

---

### 10.2 Creating a 2-Replica PVC

Create file:

    cat > 04-pvc-replica-2.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-replica-2
      namespace: longhorn-replica-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-replica-2
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f 04-pvc-replica-2.yaml

Check:

    kubectl get pvc -n longhorn-replica-demo
    kubectl get pv

---

### 10.3 Viewing 2-Replica Volume

Run:

kubectl -n longhorn-system get volumes.longhorn.io -o wide

View Details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Key Confirmations:

    Number Of Replicas: 2
    Robustness
    Conditions
    Kubernetes Status

---

### 10.4 View Replica Distribution

Execute:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

View Details:

    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Focus On:

    Node ID
    Disk ID
    Data Path
    State
    Healthy At
    Failed At

Objective:

    Distribute the two Replicas across different nodes as much as possible.

---

## Eleven. Practical Operation Four: Create Pod Mounting 2-Replica PVC

### 11.1 Create Pod

Create file:

    cat > 05-pod-replica-2.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-replica-2
      namespace: longhorn-replica-demo
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
            claimName: pvc-replica-2
    EOF

Apply:

    kubectl apply -f 05-pod-replica-2.yaml

View:

    kubectl get pod -n longhorn-replica-demo -o wide

---

### 11.2 Write Test Data

Execute:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "echo 'hello longhorn replica 2' > /data/hello.txt"

Write Time:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "date > /data/write-time.txt"

View:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/hello.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-time.txt

---

### 11.3 View Volume Attach Node

Execute:

    kubectl get pod pod-replica-2 -n longhorn-replica-demo -o wide

Record the node where the Pod is located.

View Volume:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Focus On:

    Current Node
    State
    Robustness

---

## Twelve. Practical Operation Five: Observe Replica Directory and Disk Usage

### 12.1 View Longhorn Replica Object

Execute:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

View Details:

    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

Record:

    Node ID
    Disk ID
    Data Path

---

### 12.2 View Data Directory on Worker Node

Execute on the corresponding Worker node:

    df -hT /data/longhorn
    du -sh /data/longhorn
    find /data/longhorn -maxdepth 4 -type d | head -50

High-Risk Warning:

    Do not manually modify any data under /data/longhorn.
    Do not delete Replica directories.
    Do not use vim, rm, or mv to operate Longhorn data files.
    Only observe, do not modify, on the node side.

---

### 12.3 View Capacity Changes

Execute on each data node:

    du -sh /data/longhorn

Then write a 200Mi file in the Pod:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "dd if=/dev/zero of=/data/file-200m.bin bs=1M count=200"

Check again:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- ls -lh /data/file-200m.bin

Check on the node side:

    du -sh /data/longhorn

Note:

    Due to replication, sparse files, snapshots, and file system mechanisms, the du results may not strictly equal 200Mi multiplied by the number of replicas.
    However, the capacity amplification caused by replication must be considered in planning.

---

## Thirteen. Practical Operation Six: Simulate Pod Cross-Node Re-mount

### 13.1 Record Current Node

Execute: /think

kubectl get pod pod-replica-2 -n longhorn-replica-demo -o wide

Assume current location:

    k8s-worker01

---

### 13.2 Deleting Pod

Execute:

    kubectl delete pod pod-replica-2 -n longhorn-replica-demo

---

### 13.3 Specifying Scheduling to Another Node

If you want to schedule to k8s-worker02, create file:

    cat > 06-pod-replica-2-worker02.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-replica-2
      namespace: longhorn-replica-demo
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
            claimName: pvc-replica-2
    EOF

Apply:

    kubectl apply -f 06-pod-replica-2-worker02.yaml

Check:

    kubectl get pod pod-replica-2 -n longhorn-replica-demo -o wide

---

### 13.4 Verifying Data Still Readable

Execute:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/hello.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-time.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- ls -lh /data/file-200m.bin

Check Volume:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Explanation:

    Volume will detach from original node and attach to new node.
    RWO volumes cannot be mounted for read/write by two nodes simultaneously.
    If Multi-Attach occurs, it indicates the old node may not have released or there is an abnormal mount state.

---

## Fourteen、Hands-on Practice Seven: Simulating Worker Node Failure

### 14.1 Exercise Objective

This article selects simulating a Worker node failure to observe:

    Whether Pod is affected
    Whether Volume becomes Degraded
    Whether Replica count decreases
    Whether data can still be read
    Whether Replica Rebuild occurs after node recovery
    Whether Volume recovers to Healthy

---

### 14.2 Pre-Exercise Verification

Execute on management node:

    kubectl get nodes -o wide
    kubectl get pod -n longhorn-replica-demo -o wide
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Verify:

    Pod is Running.
    PVC is Bound.
    Volume is Healthy.
    Replica is normal.
    No ongoing replica rebuild.
    Not a production namespace.

---

### 14.3 Selecting Fault Node

Recommend selecting a Worker node that hosts Replica.

Check Replica distribution:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Assume select:

    k8s-worker01

If Pod is currently also running on k8s-worker01, node failure will simultaneously affect Pod and Replica on that node.

If Pod runs on k8s-worker02 while k8s-worker01 only has Replica, it's easier to observe Volume Degraded but business Pod may continue running.

---

### 14.4 Non-Destructive Exercise: Cordon Node

First cordon node to prevent new Pod scheduling:

    kubectl cordon k8s-worker01

Check:

    kubectl get nodes

Explanation:

    Cordon does not take node offline.
    Cordon only prevents new Pod scheduling.
    It cannot simulate real node failure.
    But suitable for preparation before changes.

Recover:

    kubectl uncordon k8s-worker01

---

### 14.5 Fault Simulation Method One: Stopping kubelet

High-risk warning:

    Only allowed to execute in experimental environment.
    Do not stop kubelet on production nodes arbitrarily.
    Confirm no production business before execution.

Execute on k8s-worker01:

    systemctl stop kubelet

Observe on management node:

    kubectl get nodes -o wide
    kubectl get pod -A -o wide | grep k8s-worker01
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Explanation:

Stopping kubelet is not equal to power-off.
Running containers in containerd may still be active.
But Kubernetes will gradually detect node anomalies.
Longhorn status changes may take some time.

---

### 14.6 Fault Simulation Method Two: Node Shutdown

A more realistic fault simulation method is to shut down the node.

High-risk warning:

    Only allowed in experimental environments.
    Do not power off nodes in production clusters arbitrarily.
    Confirm the node has no critical business before shutdown.

Execute on k8s-worker01:

    shutdown -h now

Or shut down the virtual machine through the virtualization platform.

Monitor from the management node:

    kubectl get nodes -o wide
    kubectl get pod -A -o wide | grep k8s-worker01
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

### 14.7 Observing Volume Status

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Check details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

May see:

    robustness: degraded
    Scheduled condition abnormal
    Insufficient replica count
    Some replica failed or stopped

Explanation:

    Degraded indicates the Volume may still be usable, but with insufficient replicas or replica anomalies.
    Do not ignore Degraded for a long time.
    Degraded is a risk state, not a normal state.

---

### 14.8 Verifying Data Read

If the Pod is still running, execute:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/hello.txt

If the Pod is also affected by node failure, you need to recreate it or wait for it to be scheduled to another node.

Check the Pod:

    kubectl get pod -n longhorn-replica-demo -o wide
    kubectl describe pod pod-replica-2 -n longhorn-replica-demo

If you need to rebuild the Pod to a healthy node:

    kubectl delete pod pod-replica-2 -n longhorn-replica-demo --force --grace-period=0

Then reapply the Pod YAML specified to a healthy node.

---

### 14.9 Verifying Data Write

Execute in a healthy Pod:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "echo 'write during degraded' > /data/write-during-degraded.txt"

Read:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-during-degraded.txt

Explanation:

    If there are still healthy replicas and the Volume meets runtime conditions, it may still be readable and writable.
    But the risk increases at this point.
    Volumes should not be kept in Degraded state for long in production.

---

## FifteenI don't know.Hands-on Eight: Recovering the Node and Observing Replica Rebuild

### 15.1 Recovering kubelet

If the previous action was stopping kubelet, recover:

    systemctl start kubelet
    systemctl status kubelet

Check from the management node:

    kubectl get nodes -o wide

---

### 15.2 Recovering the Powered-off Node

If the previous action was powering off the node, start the virtual machine or physical machine.

After node recovery, execute:

    systemctl status kubelet
    systemctl status containerd
    systemctl status iscsid
    df -hT /data/longhorn

Check from the management node:

    kubectl get nodes -o wide
    kubectl -n longhorn-system get pods -o wide | grep k8s-worker01

---

### 15.3 Observing Replica Rebuild

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Continuously monitor:

    watch -n 2 'kubectl -n longhorn-system get volumes.longhorn.io -o wide; echo; kubectl -n longhorn-system get replicas.longhorn.io -o wide'

Check Volume details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Check Replica details:

    kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>

May see:

    rebuilding
    running
    healthy
    failed at
    healthy at

---

### 15.4 Viewing Longhorn Manager Logs

Execute:

    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

If you want to continuously monitor:

    kubectl -n longhorn-system logs -l app=longhorn-manager -f

Monitor:

    replica rebuild
    failed replica
    scheduling
    volume degraded
    volume healthy
    disk unavailable
    node down

### 15.5 Verifying Volume Recovery to Healthy

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Expected:

    Robustness: healthy

If degraded for a long time:

    Check replica count.
    Check if nodes allow scheduling.
    Check disk space.
    Check for failed replicas.
    Check error messages in Longhorn UI.
    Check longhorn-manager logs.

---

### 15.6 Verifying Data Integrity

Execute:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/hello.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-time.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-during-degraded.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- ls -lh /data/file-200m.bin

---

## SixteenI don't know.Hands-on Practice 9: Replica Scheduling Insufficiency Demonstration

### 16.1 Why Replica Scheduling Insufficiency Occurs

Common reasons for replica scheduling insufficiency:

    Insufficient number of available data nodes.
    Insufficient disk space on nodes.
    Nodes disallow scheduling.
    Nodes have DiskPressure.
    Anti-affinity policies prevent placement.
    Replica count set higher than available nodes.
    Data directory not configured or unavailable.

---

### 16.2 Creating a 3-Replica StorageClass

If there are currently only two Worker data nodes, creating a 3-replica may result in scheduling insufficiency.

Create file:

    cat > 07-sc-longhorn-replica-3.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-replica-3
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

    kubectl apply -f 07-sc-longhorn-replica-3.yaml

---

### 16.3 Creating a 3-Replica PVC

Create file:

    cat > 08-pvc-replica-3.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-replica-3
      namespace: longhorn-replica-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-replica-3
      resources:
        requests:
          storage: 1Gi
    EOF

Apply:

    kubectl apply -f 08-pvc-replica-3.yaml

Check:

    kubectl get pvc -n longhorn-replica-demo
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

### 16.4 Checking Scheduling Conditions

Check Volume details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Focus on:

    Conditions
    Scheduled
    Robustness
    Number Of Replicas

If replica scheduling insufficiency occurs, you may see:

    Scheduled: False
    not enough nodes
    insufficient storage
    replica scheduling failed

Note:

    This is not Longhorn being "broken".
    It's a mismatch between replica count planning and node/disk resources.
    In production, node count and replica count must be planned in advance.

---

## SeventeenI don't know.Understanding Volume Status

### 17.1 Healthy

Healthy indicates:

    Volume is normal.
    Required replica count is met.
    Replica status is healthy.
    Current read/write is normal.

This is the target state.

---

### 17.2 Degraded

Degraded indicates:

    Volume may still be usable.
    But replica count is insufficient or there are replica anomalies.
    It may be in the process of rebuilding.
    Current fault tolerance has decreased.

Common causes:

    Node failure.
    Disk failure.
    Replica failure.
    Replica rebuilding.
    Replica scheduling insufficiency.
    Insufficient node count.
    Insufficient disk space.

Handling principles:

    Do not ignore for a long time.
    Restore nodes or disks as soon as possible.
    Observe if rebuilding is in progress.
    Check if replicas are replenished.
    Check if backups exist.

---

### 17.3 Faulted

Faulted indicates:

    Volume has serious anomalies.
    May not have enough healthy replicas.
    Pod read/write may fail.
    Business may be affected.

Handling principles:

    Immediately stop high-risk operations.
    Preserve the scene.
    Check Volume, Replica, Engine, Instance Manager.
    Check if there are available backups.
    Do not delete Replica or Volume arbitrarily.
    Evaluate whether restore is needed.

---

## EighteenI don't know.Understanding Replica Rebuilding Mechanism

### 18.1 What is Replica Rebuild

Replica Rebuild means:

    When a Replica is lost, expired, or abnormal, Longhorn rebuilds a new Replica from a healthy Replica.

Rebuild objectives:

    Restore the Volume to the desired number of replicas.
    Recover the Volume from Degraded to Healthy.

---

### 18.2 Impact During Rebuild

Rebuild consumes:

    Disk read from source node.
    Disk write to target node.
    Network bandwidth between nodes.
    Longhorn Engine / Replica resources.
    CPU and memory.

Production impact:

    Business I/O may slow down.
    Network pressure increases.
    Disk I/O increases.
    Multiple Volumes rebuilding simultaneously may have significant impact.

---

### 18.3 Production Recommendations

Recommendations for production:

    Avoid restarting multiple nodes simultaneously.
    Avoid bulk evicting storage nodes during peak hours.
    Monitor Replica Rebuild status.
    Control concurrent rebuild parameters.
    Ensure data disks have sufficient space.
    Ensure node network stability.
    Handle Volumes that are long Degraded.

---

## Nineteen, Common Troubleshooting

### 19.1 PVC Created but Volume Degraded

Troubleshoot:

    kubectl get pvc -n longhorn-replica-demo
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Common causes:

    Replica count exceeds available data nodes.
    Node disk space insufficient.
    Node disallows scheduling.
    Data path abnormal.
    Longhorn Node status abnormal.

---

### 19.2 Replica Scheduling Failed

Troubleshoot:

    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
    df -hT /data/longhorn
    kubectl describe node <node-name>

Focus on:

    Allow Scheduling
    Storage Available
    Storage Scheduled
    Disk Pressure
    Conditions
    Disk Path

---

### 19.3 Node Recovery but No Rebuild

Troubleshoot:

    kubectl get nodes -o wide
    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Possible causes:

    Node not Ready.
    iscsid not running.
    Longhorn Node not recovered.
    Disk not schedulable.
    Disk space insufficient.
    Rebuild waiting interval.
    Old replica reuse or cleanup in progress.

---

### 19.4 Pod Mount Failed

Troubleshoot:

    kubectl describe pod pod-replica-2 -n longhorn-replica-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    systemctl status iscsid
    iscsiadm --version
    kubectl -n longhorn-system get engines.longhorn.io
    kubectl -n longhorn-system get volumes.longhorn.io

Common causes:

    iscsid not running.
    Volume attach failed.
    Engine abnormal.
    RWO multi-node mount conflict.
    Node abnormal not fully recovered.

---

### 19.5 Volume Long Degraded

Handling steps:

    1. Confirm which PVC the Volume belongs to.
    2. Confirm which business is affected.
    3. Check Replica distribution.
    4. Check for failed Replica.
    5. Check Longhorn Node and Disk status.
    6. Check data disk capacity.
    7. Check longhorn-manager logs.
    8. Confirm if rebuild is ongoing.
    9. Confirm if Backup is normal.
    10. Handle during maintenance window if necessary.

---

## Twenty, Production Replica Planning Method

### 20.1 Node Count Planning

Recommendations:

| Data Node Count | Recommended Replica Count | Notes |
|---|---|---|
| 1 | 1 | Only for learning, no node high availability |
| 2 | 2 | Can achieve basic high availability, but with larger risk window |
| 3 | 3 | Common production starting configuration |
| 4+ | 3 | Common configuration, can adjust based on business |

---

### 20.2 Disk Planning

Recommendations:

    Use dedicated data disks.
    Do not place /data/longhorn on system disk.
    Data disks should have similar capacity.
    Data disks should have similar performance.
    Do not mix slow and fast disks for same business.
    Set capacity water level alerts.
    Reserve space for replica rebuild.

---

### 20.3 Replica and Backup Planning

Recommendations:

Ordinary Test Volume: 1 replica or 2 replicas.  
Ordinary Production Volume: 2 replicas or 3 replicas.  
Critical Production Volume: 3 replicas + Backup Target.  
Core Database: Combine with database's own backup mechanism.  
Do not treat 3 replicas as backup.

---

### 20.4 Replication and Performance Planning

Need to evaluate:

    Write latency.  
    Node-to-node network.  
    Disk IOPS.  
    Replica rebuild impact.  
    Business peak hours.  
    Database's own replication capability.  
    Whether to need strict-local or more specific strategy.

If the application already has strong replication capability, such as distributed database, need to combine with business architecture to evaluate whether Longhorn multi-replica is needed, avoid high overhead from duplicate replication.

---

## Twenty-one, High-Risk Operations Reminder

The following operations in production environment must be cautious:

    Stop kubelet  
    Close Worker node  
    Batch restart multiple Longhorn data nodes  
    Delete Replica  
    Delete Volume  
    Delete PVC  
    Delete /data/longhorn  
    Increase replica count without capacity evaluation  
    Decrease replica count without risk evaluation  
    Continue high-risk changes when Volume is Degraded  
    Do fault drill without Backup

Must confirm before execution:

    Whether it's experimental environment.  
    Whether there's business impact.  
    Whether there's Backup.  
    Whether there's recovery plan.  
    Whether there's maintenance window.  
    Whether there's dual-person review.  
    Whether know current Volume and PVC ownership.

---

## Twenty-two, Experiment Cleanup

### 22.1 Restore Node

If executed cordon:

    kubectl uncordon k8s-worker01

If kubelet was stopped:

    systemctl start kubelet  
    systemctl status kubelet

Confirm:

    kubectl get nodes -o wide

---

### 22.2 Delete Pod

Execute:

    kubectl delete -f 05-pod-replica-2.yaml --ignore-not-found=true  
    kubectl delete -f 06-pod-replica-2-worker02.yaml --ignore-not-found=true

---

### 22.3 Delete PVC

High-risk reminder:

    The following only deletes experimental PVC.  
    Production environment cannot directly delete.

Execute:

    kubectl delete -f 02-pvc-replica-1.yaml --ignore-not-found=true  
    kubectl delete -f 04-pvc-replica-2.yaml --ignore-not-found=true  
    kubectl delete -f 08-pvc-replica-3.yaml --ignore-not-found=true

Check:

    kubectl get pvc -n longhorn-replica-demo  
    kubectl get pv  
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 22.4 Delete StorageClass

Execute:

    kubectl delete -f 01-sc-longhorn-replica-1.yaml --ignore-not-found=true  
    kubectl delete -f 03-sc-longhorn-replica-2.yaml --ignore-not-found=true  
    kubectl delete -f 07-sc-longhorn-replica-3.yaml --ignore-not-found=true

---

### 22.5 Delete Namespace

After confirming no resources:

    kubectl delete namespace longhorn-replica-demo

---

### 22.6 Delete Local YAML

Execute:

    rm -f 01-sc-longhorn-replica-1.yaml  
    rm -f 02-pvc-replica-1.yaml  
    rm -f 03-sc-longhorn-replica-2.yaml  
    rm -f 04-pvc-replica-2.yaml  
    rm -f 05-pod-replica-2.yaml  
    rm -f 06-pod-replica-2-worker02.yaml  
    rm -f 07-sc-longhorn-replica-3.yaml  
    rm -f 08-pvc-replica-3.yaml

---

## Twenty-three, Completion Standard of This Article's Hands-on Practice

After completing this article, should at least reach:

| Item | Standard |  
|---|---|  
| StorageClass | Created longhorn-replica-1 and longhorn-replica-2 |  
| PVC | Created 1 replica and 2 replica PVC |  
| Volume | Can view Longhorn Volume |  
| Replica | Can view Replica count and node distribution |  
| Pod Mount | pod-replica-2 can mount PVC |  
| Data Write | Can write files to /data |  
| Cross-node Mount | Can delete Pod and remount on another node |  
| Node Failure Drill | Can observe Volume status after node anomaly |  
| Degraded | Understand and observe Volume Degraded |  
| Rebuild | Observe Replica Rebuild after node recovery |  
| Scheduling Insufficiency | Understand risk of 3 replicas when nodes are insufficient |  
| Cleanup | Can safely clean up experimental resources |

---

## Twenty-four, Interview Answering Strategy

If interviewed and asked:

    How does Longhorn's replication mechanism work? What happens when a node fails?

Can answer:  
> [!note] Longhorn uses distributed replication to maintain data consistency across multiple nodes. When a node fails, the system automatically redistributes data to healthy nodes, ensuring data availability and redundancy. The exact behavior depends on the replication configuration and the number of replicas.

# Longhorn Replica Mechanism

Each Volume in Longhorn can be configured with the number of Replicas, such as 2 or 3 Replicas. After users create a PVC, Longhorn will create the corresponding Volume and generate multiple Replicas based on the StorageClass or default settings. Replicas will be distributed across different nodes and disks as much as possible. When a Pod writes data, the write request will pass through the Longhorn Engine and be synchronized to multiple Replicas.

If a node fails, the Replicas located on that node will become unavailable, and the Volume may transition from Healthy to Degraded. Degraded indicates that the Volume may still be accessible, but the number of Replicas is insufficient, reducing fault tolerance. At this point, if there are still healthy Replicas, the business may still be able to read and write, but the risk increases, and the Volume cannot remain in Degraded state long-term.

After the node recovers, Longhorn will attempt to recover or rebuild the missing Replica to return the Volume to the expected number of Replicas. This process is called Replica Rebuild. Rebuild consumes disk I/O, network, and CPU resources, so in production environments, it's essential to monitor the rebuild status to avoid multiple nodes restarting simultaneously or multiple Volumes undergoing large-scale rebuilds at the same time.

The number of Replicas should be planned based on the number of nodes. A cluster with only two Worker nodes should not blindly set 3 Replicas and assume full high availability. The higher the number of Replicas, the greater the disk usage, and the higher the write synchronization pressure.

Additionally, Longhorn Replicas are a high availability mechanism, not a backup. They can handle single-node or single-Replica failures but cannot prevent accidental PVC deletion, application data corruption, user rm file deletion, or complete cluster damage. Therefore, critical production data must be configured with a Backup Target and regular recovery drills should be conducted.

---

## 25. Summary of This Chapter

This chapter completes the study of Longhorn's Replica mechanism:

1. Replica is a data copy of the Longhorn Volume.
2. Replica is used to enhance availability in case of node or disk failures.
3. Replica cannot replace backups.
4. 1 Replica is suitable for learning and non-critical data.
5. 2 Replicas are suitable for basic experiments with two data nodes.
6. 3 Replicas are more suitable for three or more data nodes.
7. The number of Replicas will amplify the underlying disk usage.
8. The number of Replicas will increase write synchronization pressure.
9. The number of Replicas is not necessarily better the higher it is.
10. PVC can specify the number of Replicas through StorageClass parameters.
11. Longhorn Volume status can be viewed through CRD.
12. Replica distribution can be viewed through replicas.longhorn.io.
13. When a Pod is remounted across nodes, the Volume will be re-attached.
14. After a node failure, the Volume may become Degraded.
15. Degraded indicates reduced availability and cannot be ignored long-term.
16. After node recovery, Longhorn will attempt Replica Rebuild.
17. Rebuild consumes disk, network, and CPU resources.
18. Replica scheduling failures are usually related to node count, disk capacity, and scheduling policies.
19. Production Replica planning must combine node count, capacity, performance, and backup strategies.
20. The next chapter will learn about Longhorn backup recovery: Backup Target, Snapshot, and Volume Restore.

---

## 26. Reference Documents

Longhorn official documentation:

    https://longhorn.io/docs/latest/

Longhorn nodes and volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn Volume Conditions:

    https://longhorn.io/docs/latest/nodes-and-volumes/volumes/volume-conditions/

Longhorn Replica Rebuilding:

    https://longhorn.io/docs/latest/advanced-resources/rebuilding/

Longhorn StorageClass parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn best practices:

    https://longhorn.io/docs/latest/best-practices/

Longhorn backup and recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes node maintenance:

    https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/