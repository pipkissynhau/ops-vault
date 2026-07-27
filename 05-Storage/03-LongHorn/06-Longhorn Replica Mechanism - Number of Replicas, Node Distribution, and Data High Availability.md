# Longhorn Replica Mechanism: Number of Replicas, Node Distribution, and Data High Availability

Recommended Path: 05-Storage/03-LongHorn/06-Longhorn Replica Mechanism: Number of Replicas, Node Distribution, and Data High Availability.md

Tags: #Longhorn #Replica #Replica Mechanism #Node Distribution #VolumeDegraded #ReplicaRebuild #Kubernetes #PVC #Block Storage #High Availability #Advanced SRE #Production Operations

---

## I. Document Introduction

This article is the sixth in the Longhorn series, focusing on Longhorn's replica mechanism, node distribution, Volume Degraded, Replica Rebuild, and data high availability.

Previously covered topics include:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependent Components, and StorageClass
- Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practices: PVC, PV, Pod Mounting, and Data Persistence

This article delves into the core aspects of Longhorn data high availability:

    What does the number of replicas mean?
    How are replicas distributed across different nodes?
    What are the differences between 2-replica and 3-replica configurations?
    Why isn't a higher number of replicas always better?
    What happens to a Volume when a node fails?
    How to understand Volume states: Healthy, Degraded, Faulted?
    How to monitor Replica Rebuild processes?
    How are replicas replenished after a node recovers?
    Why can't the replica mechanism replace backup solutions?
    How to plan the number of replicas, nodes, disk capacity, and fault recovery drills in production environments?

This article emphasizes practicality and will use real PVCs, Pods, StorageClasses, and node simulations to demonstrate Longhorn's replica behavior.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the role of Longhorn replicas.
2. Comprehend the relationship between the number of replicas and the number of nodes.
3. Identify suitable use cases for 1-replica, 2-replica, and 3-replica configurations.
4. Analyze how the number of replicas affects disk capacity and write performance.
5. Create a StorageClass with a specified number of replicas.
6. Generate a PVC and verify the number of Longhorn Volume replicas.
7. Determine which nodes and disks the replicas are distributed across.
8. Simulate cross-node Pod re-mounting.
9. Simulate Worker node failures.
10. Observe how a Volume changes from Healthy to Degraded.
11. Restore a node and observe the Replica Rebuild process.
12. Determine whether a Degraded Volume can still be used for reading and writing.
13. Troubleshoot replica scheduling failures.
14. Understand that the replica mechanism cannot prevent accidental data deletions or logical errors.
15. Establish production-level Longhorn replica planning and fault recovery procedures.

---

## III. Experimental Environment

### 3.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node IP Range: 10.0.0.0/24

Node configuration:

| IP | Host Name | Role | Description |
|---|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane | Optional for experiments |
| 10.0.0.21 | k8s-worker01 | Worker | Longhorn data node |
| 10.0.0.22 | k8s-worker02 | Worker | Longhorn data node |

---

### 3.2 Longhorn Environment

Longhorn namespace:

    longhorn-system

Longhorn default StorageClass:

    longhorn

Two additional experimental StorageClasses will be created in this article:

    longhorn-replica-1
    longhorn-replica-2

If the experiment allows the Master node to participate in data scheduling, another StorageClass can be created:

    longhorn-replica-3

---

### 3.3 Experimental Namespace

This article uses the following namespace:

    longhorn-replica-demo

Create directories:

    mkdir -p ~/longhorn-replica-demo
    cd ~/longhorn-replica-demo

---

## IV. Basic Understanding of Longhorn Replicas

### 4.1 What are Replicas?

Replicas are data copies of a Longhorn Volume.

When a PVC creates a Volume through Longhorn, Longhorn will generate multiple replicas based on the specified number of replicas.

For example:

    PVC### 6.2 Write Amplification

As the number of replicas increases, so does the number of write paths.

For example, with 3 replicas:

    Pod writes data
      |
      v
    Longhorn Engine
      |
      +--> Replica 1
      +--> Replica 2
      +--> Replica 3

Writing a piece of data requires synchronization to multiple replicas.

Impact:

    Increased network traffic.
    Increased disk I/O operations.
    Potential increase in write latency.
    Increased load on the nodes hosting the replicas.

Production understanding:

    The number of replicas represents a trade-off between reliability, capacity, and performance.
    More replicas are not always better.
    Do not blindly increase the number of replicas just to make it seem “safer.”

---

## VII. Pre-Operation Checks

### 7.1 Checking Longhorn Components

Execute:

    kubectl -n longhorn-system get pods -o wide

Check for any abnormalities:

    kubectl -n longhorn-system get pods | grep -Ev "Running|Completed"

If any issues are found:

    kubectl -n longhorn-system describe pod <pod-name>
    kubectl -n longhorn-system logs <pod-name> --tail=100

---

### 7.2 Checking StorageClass

Execute:

    kubectl get sc

View the default Longhorn StorageClass:

    kubectl describe sc longhorn

Pay attention to:

    provisioner
    reclaimPolicy
    volumeBindingMode
    allowVolumeExpansion
    numberOfReplicas

---

### 7.3 Checking Longhorn Nodes

Execute:

    kubectl -n longhorn-system get nodes.longhorn.io

View node details:

    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01
    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker02

Key considerations:

    Allow Scheduling
    Ready
    Schedulable
    Disk Path
    Storage Available
    Storage Scheduled
    Conditions

---

### 7.4 Checking Node Disks

On each data node, execute the following commands:

    df -hT /data/longhorn
    lsblk -f
    du -sh /data/longhorn

If /data/longhorn is on the system disk:

    This setup is only suitable for experimentation.
    It is not recommended for use in production critical applications.

---

### 7.5 Checking iSCSI

On each Worker node, execute:

    systemctl is-active iscsid
    iscsiadm --version

If it is not running:

    systemctl enable --now iscsid

---

## VIII. Practical Operation 1: Creating an Experimental Namespace

### 8.1 Creating a Namespace

Execute:

    kubectl create namespace longhorn-replica-demo

Verify:

    kubectl get ns longhorn-replica-demo

---

### 8.2 Creating a Working Directory

Execute:

    mkdir -p ~/longhorn-replica-demo
    cd ~/longhorn-replica-demo

---

## IX. Practical Operation 2: Creating a StorageClass with 1 Replica

### 9.1 Creating a StorageClass

Create the file:

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

Apply the configuration:

    kubectl apply -f 01-sc-longhorn-replica-1.yaml

Verify:

    kubectl get sc
    kubectl describe sc longhorn-replica-1

---

### 9.2 Creating a PVC with 1 Replica

Create the file:

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

Apply the configuration:

    kubectl apply -f 02-pvc-replica-1.yaml

Verify:

    kubectl get pvc -n longhorn-replica-demo
    kubectl get pv

---

### 9.3 Viewing the Longhorn Volume

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io

Find the corresponding volume and view its details:

    kubectl```bash
kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
```

Key points to confirm:

- Number of Replicas: 2
- Robustness
- Conditions
- Kubernetes Status

---

### 10.4 Viewing Replica Distribution

Execute:

```bash
kubectl -n longhorn-system get replicas.longhorn.io -o wide
```

For detailed information:

```bash
kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>
```

Focus on:

- Node ID
- Disk ID
- Data Path
- State
- Healthy At
- Failed At

Goal:

- Both replicas should be distributed across different nodes as much as possible.

---

## Section Eleven: Practical Exercise Four: Creating a Pod with 2 Replica PVCs Mounted

### 11.1 Creating a Pod

Create the file:

```bash
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
```

Apply the configuration:

```bash
kubectl apply -f 05-pod-replica-2.yaml
```

Check the result:

```bash
kubectl get pod -n longhorn-replica-demo -o wide
```

---

### 11.2 Writing Test Data

Execute:

```bash
kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "echo 'hello longhorn replica 2' > /data/hello.txt"
```

Record the time of writing:

```bash
kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "date > /data/write-time.txt"
```

View the content:

```bash
kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/hello.txt
kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-time.txt
```

---

### 11.3 Checking Volume Attached Nodes

Execute:

```bash
kubectl get pod pod-replica-2 -n longhorn-replica-demo -o wide
```

Record the node where the Pod is located.

View the Volume:

```bash
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
```

Focus on:

- Current Node
- State
- Robustness

---

## Section Twelve: Practical Exercise Five: Observing Replica Directories and Disk Usage

### 12.1 Viewing Longhorn Replica Objects

Execute:

```bash
kubectl -n longhorn-system get replicas.longhorn.io -o wide
```

For detailed information:

```bash
kubectl -n longhorn-system describe replicas.longhorn.io <replica-name>
```

Record:

- Node ID
- Disk ID
- Data Path

---

### 12.2 Viewing Data Directories on the Node

On the corresponding Worker node, execute:

```bash
df -hT /data/longhorn
du -sh /data/longhorn
find /data/longhorn -maxdepth 4 -type d | head -50
```

High-risk reminder:

- Do not manually modify any data under `/data/longhorn`.
- Do not delete the Replica directory.
- Do not use tools like vim, rm, or mv to manipulate Longhorn data files.
- Only observe on the node; do not make any modifications.

---

### 12.3 Checking Capacity Changes

On each data node, execute:

```bash
du -sh /data/longhorn
```

Then write a 200Mi file inside the Pod:

```bash
kubectl exec -n longhorn-replica-demo pod-replica-2 -- sh -c "dd if=/dev/zero of=/data/file-200m.bin bs=1M count=200"
```

Check again:

```bash
kubectl exec -n longhorn-replica-demo pod-replica-2 -- ls -lh /data/file-200m.bin
```

On the node, check the capacity change:

```bash
du -sh /data/longhorn
```

Note:

- Due to replicas## Section Fourteen: Practical Exercise Seven: Simulating a Worker Node Failure

### 14.1 Exercise Objectives

This exercise involves simulating a failure of a Worker node to observe the following:

- Whether the Pod is affected.
- Whether the Volume becomes Degraded.
- Whether the number of Replicas decreases.
- Whether data can still be read from the Volume.
- Whether a Replica Rebuild occurs after the node recovers.
- Whether the Volume returns to a Healthy state.

---

### 14.2 Pre-exercise Confirmation

On the management node, execute the following commands:

    kubectl get nodes -o wide
    kubectl get pod -n longhorn-replica-demo -o wide
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Confirm that:

- The Pod is running.
- The PVC is bound to the Pod.
- The Volume is in a Healthy state.
- The number of Replicas is normal.
- There are no Replicas undergoing reconstruction.
- The namespace is not for production services.

---

### 14.3 Selecting a Node for Failure

It is recommended to choose a Worker node that is carrying Replicas.

View the distribution of Replicas:

    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Assume you select:

    k8s-worker01

If the Pod is also running on k8s-worker01, then a failure of this node will affect both the Pod and the Replicas on it.

If the Pod runs on k8s-worker02 while k8s-worker01 only has Replicas, it will be easier to observe that the Volume becomes Degraded, but the production Pod may still continue to function.

---

### 14.4 Non-destructive Exercise: Cordoning the Node

First, cordon the node to prevent new Pods from being scheduled:

    kubectl cordon k8s-worker01

Check the status:

    kubectl get nodes

Note that:

- Cording a node does not cause it to go offline.
- It only prevents new Pods from being scheduled on that node.
- This method cannot simulate a real node failure, but it is useful for preparation before making changes.

To restore the node, execute:

    kubectl uncordon k8s-worker01

---

### 14.5 Method One for Fault Simulation: Stopping kubelet

High-risk warning:

- This method should only be used in an experimental environment.
- Do not stop kubelet on production nodes without proper authorization.
- Ensure that there are no production services running on this node before proceeding.

On k8s-worker01, execute:

    systemctl stop kubelet

On the management node, observe the following:

    kubectl get nodes -o wide
    kubectl get pod -A -o wide | grep k8s-worker01
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

Note that stopping kubelet does not mean powering down the entire node. Containers running in containerd may continue to operate, but Kubernetes will start detecting the node as abnormal. Changes in the Longhorn status may take some time to become apparent.

---

### 14.6 Method Two for Fault Simulation: Shutting Down the Node

A more realistic way to simulate a failure is to shut down the entire node.

High-risk warning:

- This method should only be used in an experimental environment.
- Do not shut down production nodes without proper planning.
- Ensure that no critical services are running on this node before shutting it down.

On k8s-worker01, execute:

    shutdown -h now

Alternatively, you can shut down the virtual machine through your virtualization platform.

On the management node, observe the following:

    kubectl get nodes -o wide
    kubectl get pod -A -o wide | grep k8s-worker01
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

### 14.7 Observing Volume Status

Execute the following command to check the status of the Volume:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

To view detailed information about a specific Volume, execute:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

You may see entries such as:

- robustness: degraded
- Scheduled condition: abnormal
- Insufficient number of Replicas
- A certain Replica has failed or stopped

These indications suggest that the Volume is in a Degraded state, which means it may still be usable, but there are issues with the number of Replicas or some Replicas are malfunctioning.

disk unavailable
node down

---

### 15.5 Verifying Volume Recovery to Healthy Status

Perform the following command:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Expected result:

    Robustness: healthy

If the volume remains degraded for an extended period:

    Check the number of replicas.
    Verify if the node allows scheduling.
    Check the disk space usage.
    Identify any failed replicas.
    Check for error messages in the Longhorn UI.
    Review the longhorn-manager logs.

---

### 15.6 Verifying Data Integrity

Perform the following commands:

    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/hello.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-time.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- cat /data/write-during-degraded.txt
    kubectl exec -n longhorn-replica-demo pod-replica-2 -- ls -lh /data/file-200m.bin

---

## Chapter Sixteen: Practical Example Nine: Demonstrating Insufficient Replica Scheduling

### 16.1 Reasons for Insufficient Replica Scheduling

Common causes of insufficient replica scheduling include:

    Insufficient number of available data nodes.
    Insufficient disk space on a node.
    A node being blocked from scheduling.
    High DiskPressure on a node.
    Replica anti-affinity policies preventing placement.
    Setting a higher number of replicas than the available nodes allow.
    The data directory not being configured or unavailable.

---

### 16.2 Creating a StorageClass with 3 Replicas

If there are only two Worker data nodes, attempting to create 3 replicas may result in insufficient scheduling.

Create the following file:

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

Apply the configuration:

    kubectl apply -f 07-sc-longhorn-replica-3.yaml

---

### 16.3 Creating a PVC with 3 Replicas

Create the following file:

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

Apply the configuration:

    kubectl apply -f 08-pvc-replica-3.yaml

Verify the settings:

    kubectl get pvc -n longhorn-replica-demo
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

### 16.4 Checking Scheduling Conditions

View the Volume details:

    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>

Pay special attention to:

    Conditions
    Scheduled
    Robustness
    Number Of Replicas

If insufficient replica scheduling is detected, you may see messages such as:

    Scheduled: False
    not enough nodes
    insufficient storage
    replica scheduling failed

Note:

    This does not indicate that Longhorn is malfunctioning. Instead, it means that the number of replicas planned does not match the available node/disk resources. In production environments, it is essential to carefully plan the number of nodes and replicas in advance.

---

## Chapter Seventeen: Understanding Volume Status

### 17.1 Healthy

A Healthy volume indicates that:

    The volume is functioning normally.
    The required number of replicas is available.
    All replicas are in healthy status.
    Current read and write operations are successful.

This is the desired state for a volume.

---

### 17.2 Degraded

A Degraded volume means that:

    The volume may still be usable, but there is an insufficient number of replicas or some replicas are malfunctioning.
    The volume may be in the process of being rebuilt.
    Current fault tolerance has decreased.

Common causes include:

    Node failures.
    Disk failures.
    Replica failures.
    In-progress replica reconstruction.
    Insufficient replica scheduling.
    Insufficient number of nodes.
    Insufficient---

### 19.3 Nodes Do Not Get Rebuilt After Recovery

Troubleshooting:

    kubectl get nodes -o wide
    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
    kubectl -n longhorn-system logs -l app=longhorn-manager --tail=200

Possible Reasons:

    The node is not ready.
    iscsid is not running.
    The Longhorn Node has not been restored.
    The disk is not schedulable.
    The disk space is insufficient.
    Rebuilding is waiting in the interval.
    Old replicas are being reused or cleaned up.

---

### 19.4 Pod Mounting Fails

Troubleshooting:

    kubectl describe pod pod-replica-2 -n longhorn-replica-demo
    kubectl get events -A --sort-by=.lastTimestamp | tail -100
    systemctl status iscsid
    iscsiadm --version
    kubectl -n longhorn-system get engines.longhorn.io
    kubectl -n longhorn-system get volumes.longhorn.io

Common Reasons:

    iscsid is not running.
    Volume attach failed.
    Engine exception.
    RWO multi-node mounting conflict.
    The node has not fully recovered from an exception.

---

### 19.5 Volume Is Long-Term Degraded

Action Steps:

    1. Determine which PVC the Volume belongs to.
    2. Identify which service is affected.
    3. Check the Replica distribution.
    4. Verify if there are any failed Replicas.
    5. Examine the status of the Longhorn Node and Disk.
    6. Check the data disk capacity.
    7. Review the longhorn-manager logs.
    8. Confirm whether rebuilding is in progress.
    9. Ensure that Backup is functioning correctly.
    10. Perform necessary maintenance during designated window hours.

---

## Section Twenty: Planning Methods for Production Replicas

### 20.1 Node Quantity Planning

Recommendations:

| Number of Data Nodes | Recommended Number of Replicas | Notes |
|---|---|---|
| 1 | 1 | Only for learning purposes; does not provide high availability for nodes |
| 2 | 2 | Can achieve basic high availability, but carries a higher risk |
| 3 | 3 | A common starting configuration for production |
| 4+ | 3 | Also a common configuration; can be adjusted based on business needs |

---

### 20.2 Disk Planning

Recommendations:

    Use dedicated data disks.
    Do not place /data/longhorn on system disks.
    Ensure that the capacity of data disks is consistent.
    Try to maintain similar performance levels among data disks.
    Avoid using slow and fast disks together for the same type of services.
    Set up alerts for capacity thresholds.
    Reserve space for replica rebuilding.

---

### 20.3 Replica and Backup Planning

Recommendations:

    For regular test volumes: 1 or 2 replicas.
    For regular production volumes: 2 or 3 replicas.
    For critical production volumes: 3 replicas + a backup target.
    For core databases: Follow the database's own backup mechanism.
    Do not consider 3 replicas as a form of backup.

---

### 20.4 Replica and Performance Planning

It is necessary to evaluate:

    Write latency.
    Network performance between nodes.
    Disk IOPS.
    The impact of replica rebuilding.
    Business peak usage times.
    The database's own replication capabilities.
    Whether strict-local or more specialized strategies are required.

If the application already has robust replication capabilities, such as those provided by distributed databases, it may be necessary to assess whether Longhorn's multi-replica setup is needed to avoid excessive overhead due to duplicate replication.

---

## Section Twenty-One: High-Risk Operations Reminder

The following operations must be carried out with extreme caution in a production environment:

    Stopping kubelet
    Shutting down Worker nodes
    Batch restarting multiple Longhorn data nodes
    Deleting Replicas
    Deleting Volumes
    Deleting PVCs
    Deleting /data/longhorn
    Increasing the number of replicas without evaluating capacity
    Decreasing the number of replicas without assessing risks
    Continuing to perform high-risk changes when a Volume is degraded
    Conducting fault drills without having a backup in place

Before proceeding, it is essential to confirm:

    Whether this is an experimental environment.
    Whether there will be any impact on business operations.
    Whether a backup exists.
    Whether a recovery plan has been prepared.
    Whether there are designated maintenance window hours.
    Whether there is double-checking by two individuals.
    And whether the current ownership of Volume and PVCs is clear.

---

##If a node fails, the Replica located on that node will become unavailable, and the Volume may change from Healthy to Degraded. A Degraded state means that the Volume is still potentially usable, but it has insufficient replicas, resulting in reduced fault tolerance. In this case, if there are still healthy Replicas, the service may still be able to perform read and write operations, but the risk increases, and a Degraded state cannot be sustained for an extended period.

Once the node recovers, Longhorn will attempt to restore or rebuild the missing Replica so that the Volume returns to the desired number of replicas. This process is called Replica Rebuild. Repbuilding consumes disk I/O, network resources, and CPU power; therefore, in a production environment, it is essential to monitor the rebuilding status to prevent multiple nodes from restarting simultaneously or multiple Volumes from undergoing large-scale rebuilds at the same time.

The number of replicas should be planned in conjunction with the total number of nodes. For a cluster with only two Worker nodes, setting 3 replicas would not necessarily ensure high availability. The more replicas there are, the greater the disk capacity occupied and the higher the pressure on write synchronization.

Additionally, Longhorn Replicas serve as a high-availability mechanism, but they are not backups. They can handle failures of individual nodes or replicas, but they cannot prevent accidental deletion of PVCs, data corruption due to application errors, user actions such as deleting files, or overall cluster damage. Therefore, for critical production data, it is necessary to configure Backup Targets and regularly conduct recovery drills.

---

## Summary of Chapter 25

This chapter has covered the Longhorn Replica mechanism:

1. Replicas are copies of data within a Longhorn Volume.
2. They enhance availability in the event of node or disk failures.
3. However, they cannot replace traditional backups.
4. 1 replica is suitable for learning purposes and non-critical data.
5. 2 replicas are appropriate for basic experiments with two data nodes.
6. 3 or more replicas are better for clusters with three or more data nodes.
7. An increased number of replicas leads to greater disk usage and higher write synchronization demands.
8. More replicas do not always mean better performance; the right balance is key.
9. The number of replicas can be specified using the StorageClass parameter.
10. The status of a Longhorn Volume can be checked through CRDs.
11. The distribution of Replicas can be viewed at replicas.longhorn.io.
12. When a Pod is remounted across nodes, the Volume will be reattached automatically.
13. After a node failure, the Volume may enter a Degraded state.
14. A Degraded state indicates reduced availability and should not be ignored for long.
15. Upon node recovery, Longhorn attempts to rebuild missing Replicas.
16. The rebuilding process consumes significant resources.
17. Failed replica scheduling is often related to factors such as the number of nodes, disk capacity, and scheduling strategies.
18. Planning production replicas requires considering node count, capacity, performance, and backup requirements.
19. In the next chapter, we will learn about Longhorn backup and recovery methods: Backup Targets, Snapshots, and Volume Restore.

---

## References

Longhorn Official Documentation:

    https://longhorn.io/docs/latest/

Longhorn Nodes and Volumes:

    https://longhorn.io/docs/latest/nodes-and-volumes/

Longhorn Volume Conditions:

    https://longhorn.io/docs/latest/nodes-and-volumes/volumes/volume-conditions/

Longhorn Replica Rebuilding:

    https://longhorn.io/docs/latest/advanced-resources/rebuilding/

Longhorn StorageClass Parameters:

    https://longhorn.io/docs/latest/references/storage-class-parameters/

Longhorn Best Practices:

    https://longhorn.io/docs/latest/best-practices/

Longhorn Backup and Recovery:

    https://longhorn.io/docs/latest/snapshots-and-backups/

Longhorn Troubleshooting:

    https://longhorn.io/kb/troubleshooting/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Kubernetes Storage Classes:

    https://kubernetes.io/docs/concepts/storage/storage-classes/

Kubernetes Node Maintenance:

    https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/