# Longhorn Performance and Production Recommendations: Disks, Networks, Scheduling, and Resource Limits

Recommended path: 05-Storage/03-LongHorn/09-Longhorn Performance and Production Recommendations: Disks, Networks, Scheduling, and Resource Limits.md

Tags: #Longhorn #PerformanceOptimization #ProductionRecommendations #DiskPerformance #NetworkPerformance #CopyMechanism #StorageClass #SchedulePolicy #ResourceConstraints #Kubernetes #AdvancedSre #ProductionTransport

---

## I. Document Overview

This is the ninth article of the Longhorn module, focusing on learning Longhorn's performance boundaries, production recommendations, disk planning, network planning, scheduling strategies, and resource limits.

Previously completed:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependent Components, and StorageClass
- Longhorn Helm Installation Methodology: Chart, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practice: PVC, PV, Pod Mounting, and Data Persistence
- Longhorn Replica Mechanism: Replica Count, Node Distribution, and Data High Availability
- Longhorn Backup and Recovery: Backup Target, Snapshot, and Volume Restore
- Longhorn Troubleshooting: Volume Degraded, Replica Rebuilding, and Node Anomalies

This article focuses on solving:

    Where are Longhorn's performance bottlenecks?
    Why do disks, networks, and replica counts affect performance?
    Why is it not recommended to place data directories on system disks in production?
    Why isn't a higher replica count always better?
    What scenarios are suitable for Longhorn?
    What scenarios should use Longhorn cautiously?
    Can database applications run on Longhorn?
    How to layer StorageClass by performance?
    How to use nodeSelector / diskSelector for scheduling?
    What is the role of dataLocality?
    How to inspect Longhorn component resources?
    How to perform simple I/O testing?
    How to establish a production-ready acceptance checklist?

This article emphasizes practical operations, providing inspection commands, StorageClass examples, fio testing methods, resource inspection commands, and production recommendation checklists.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand Longhorn's main performance bottlenecks.
2. Understand how disk performance affects Longhorn.
3. Understand how node-to-node networks affect replica synchronization.
4. Understand how replica count affects capacity, write latency, and network traffic.
5. Understand why it's recommended to use dedicated data disks in production environments.
6. Inspect Longhorn node disk capacity and I/O status.
7. Inspect node network connectivity and bandwidth.
8. View Longhorn Volume, Replica, and Engine status.
9. Create volumes with different replica strategies using StorageClass.
10. Plan storage tiering using nodeSelector / diskSelector.
11. Understand the role and usage boundaries of dataLocality.
12. Use fio for basic I/O testing on Longhorn PVC.
13. Judge whether test results are suitable for production reference.
14. Design Longhorn production capacity, backup, monitoring, and alerting baselines.
15. Clearly explain what Longhorn is suitable for and what it's not.

---

## III. Core Conclusions First

### 3.1 Longhorn is not a universal high-performance storage solution

Longhorn's advantages:

    Cloud-native
    Easy installation
    Easy management
    Well-integrated with Kubernetes PVC
    Supports replicas, snapshots, backups, and recovery
    Suitable for small-to-medium Kubernetes persistent storage

But Longhorn is not equivalent to:

    High-end SAN storage
    Enterprise-grade all-flash arrays
    Large-scale Ceph clusters
    Cloud provider high-performance cloud disks
    Database-optimized high-performance storage
    Object storage

In production, use it according to scenarios, not for all data.

---

### 3.2 Longhorn performance is mainly influenced by four factors

Core factors:

    1. Disk performance
    2. Node-to-node network
    3. Replica count
    4. Workload characteristics

Further refinement:

    SSD / HDD differences
    SATA / SAS / NVMe differences
    1Gbps / 10Gbps network differences
    Network latency and packet loss
    Replica count 1 / 2 / 3
    Whether replica rebuilding is ongoing
    Presence of many snapshots
    Running database with high random writes
    Multiple volumes with simultaneous high I/O

---

### 3.3 Longhorn Production Recommendations Priority

Priority from highest to lowest:

    First: Dedicated data disks
    Second: Stable node network
    Third: Reasonable replica count
    Fourth: Backup Target
    Fifth: Monitoring and alerts
    Sixth: Recovery drills
    Seventh: Performance stress testing
    Eighth: Tiered StorageClass usage by business

Do not reverse:

    Deploy production workloads first, then add backups
    Run databases first, then check disks
    Increase replicas first, then check capacity
    Create many PVCs first, then consider monitoring
    Expose UI first, then think about security controls

---

## IV. Experimental Environment

### 4.1 Kubernetes Cluster

Default experimental environment:

    Kubernetes: kubeadm cluster
    Operating System: Ubuntu Server 22.04.5 LTS
    Container Runtime: containerd
    CNI: Calico
    Node network segment: 10.0.0.0/24

Node planning:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master01 | Control Plane |
| 10.0.0.21 | k8s-worker01 | Worker |
| 10.0.0.22 | k8s-worker02 | Worker |

---

### 4.2 Longhorn Environment

Longhorn namespace:

    longhorn-system

Longhorn default StorageClass:

    longhorn

Longhorn data directory:

    /data/longhorn

Experimental namespace:

    longhorn-performance-demo

---

### 4.3 Article Experiment Notes

This article will create:

    longhorn-performance-demo namespace
    Performance testing PVC
    fio testing Pod
    Different StorageClass examples
    Basic read/write testing tasks

Reminder: /think

fio testing generates I/O pressure.
Do not execute directly during production peak hours.
Do not perform arbitrary pressure testing on production PVCs.
Test results are only for understanding trends and do not equate to formal performance benchmark reports.
Formal performance testing requires an independent environment, fixed variables, repeated testing, and monitoring support.

---

## Five. Understanding Longhorn Performance Path

### 5.1 Pod Write Path to Longhorn Volume

Simplified path:

    Pod
     |
     v
    PVC mount directory
     |
     v
    kubelet / CSI
     |
     v
    Longhorn Engine
     |
     +--> Replica 1
     +--> Replica 2
     +--> Replica 3
     |
     v
    Node local data disk

If the Volume is 3 replicas:

    A write requires synchronization to multiple Replicas.
    Replicas may be distributed across different nodes.
    Writes pass through inter-node network.
    Eventually land on multiple nodes' data disks.

---

### 5.2 Impact of Replica Count on Performance

More replicas:

    Higher disk usage.
    More pronounced write amplification.
    More network synchronization.
    Write latency may increase.
    Replica Rebuild time may be longer.

Fewer replicas:

    Less disk usage.
    Lower write pressure.
    But reduced fault tolerance.

Common understanding:

| Replica Count | Capacity Overhead | Availability | Write Pressure | Suitable Scenarios |
|---|---|---|---|---|
| 1 | Low | Low | Low | Learning, temporary, rebuildable data |
| 2 | Medium | Medium | Medium | Two-node experiments, small-to-medium business |
| 3 | High | High | High | Three-node+ production common configuration |

---

### 5.3 Impact of Replica Rebuild on Performance

When a node fails, replicas are lost, or replicas expire, Longhorn rebuilds the Replica.

Rebuild consumes:

    Source replica disk reads
    Target replica disk writes
    Inter-node network
    Longhorn Engine resources
    CPU
    Memory

Production impact:

    Business I/O may slow down.
    Network bandwidth is occupied.
    Disk await rises.
    Volume long-term Degraded risk increases.
    Multiple Volume Rebuilds simultaneously risk higher.

Production recommendations:

    Avoid restarting multiple storage nodes simultaneously.
    Avoid large-scale node maintenance during peak hours.
    Monitor Rebuild status.
    Reserve capacity and I/O headroom.

---

## Six. Disk Planning Recommendations

### 6.1 Not Recommended to Use System Disk for Production Data

Risks of using system disk for Longhorn data:

    System disk full causes kubelet anomalies.
    containerd anomalies.
    Node DiskPressure.
    Pods evicted.
    Longhorn Replica anomalies.
    Volume Degraded.
    Replica unable to rebuild.
    System and data mutual impact during failures.

Production recommendations:

    /data/longhorn use independent data disks.
    Separate data disk and system disk.
    Plan data disk capacity in advance.
    Data disk I/O performance meets business requirements.
    Mount data disk write to /etc/fstab.
    Do not use soft links instead of formal mounting.

---

### 6.2 File System Recommendations for Data Disks

Common choices:

    xfs
    ext4

Recommendations:

    Use mature and stable file systems.
    Do not arbitrarily try unfamiliar file systems in production.
    Keep mount parameters simple and controllable.
    Use noatime to reduce unnecessary metadata writes.
    Write mount configuration to /etc/fstab.

Example fstab:

    UUID=<actual UUID> /data xfs defaults,noatime 0 0

or:

    UUID=<actual UUID> /data ext4 defaults,noatime 0 0

Check:

    df -hT /data/longhorn
    lsblk -f
    mount | grep /data

---

### 6.3 Disk Performance Tiering

If nodes have different types of disks, storage tiering can be implemented.

Example:

| Tier | Disk Type | Suitable Business |
|---|---|---|
| fast | NVMe SSD | Database, Prometheus, low-latency business |
| standard | SATA SSD | Ordinary middleware, Jenkins, Nacos |
| capacity | HDD | Low-frequency data, test environment, low-performance business |

Longhorn can combine with:

    node tag
    disk tag
    StorageClass nodeSelector
    StorageClass diskSelector

To enable different PVCs to use different storage tiers.

---

### 6.4 Capacity Planning Formula

Basic formula:

    Actual raw capacity demand ≈ PVC requested capacity × replica count + Snapshot reservation + Rebuild reservation + growth reservation

Example:

    Business PVC total requested 500Gi
    Replica count 2
    Snapshot reservation 20%
    Rebuild/growth reservation 30%

Estimate:

    500Gi × 2 = 1000Gi
    Snapshot reservation 200Gi
    Rebuild/growth reservation 300Gi
    Total planning at least about 1500Gi

Production reminder:

    Do not plan disks only based on PVC requested capacity.
    Do not forget replica count.
    Do not forget Snapshot.
    Do not forget additional pressure during Rebuild.
    Do not let disks exceed 80% long-term.

---

## Seven. Network Planning Recommendations

### 7.1 Why Network Affects Longhorn Performance

Longhorn's multi-replica writes require inter-node synchronization.

If Replicas are distributed across different nodes:

    Writes pass through node networks.
    Network latency affects write latency.
    Network bandwidth affects throughput.
    Network jitter affects stability.
    Network interruption may cause Volume Degraded or read-only.

---

### 7.2 Network Check Commands

Check network interfaces:

    ip addr
    ip route

Test connectivity:

    ping -c 5 10.0.0.21
    ping -c 5 10.0.0.22

Test MTU:

    ip link show
    ping -M do -s 1472 10.0.0.22

Check connections:

    ss -lntp
    ss -antp | grep longhorn

Check network interface errors: /think

ip -s link

If ethtool is installed:

    ethtool <network interface name>

Example:

    ethtool ens33

---

### 7.3 iperf3 Test Bandwidth Between Nodes

Installation:

    apt update
    apt install -y iperf3

Start the server on k8s-worker01:

    iperf3 -s

Run the client on k8s-worker02:

    iperf3 -c 10.0.0.21

Reverse test:

    iperf3 -c 10.0.0.21 -R

Notes:

    iperf3 measures the network capability between nodes.
    It is not equal to Longhorn PVC performance.
    But it can help identify obvious network bottlenecks.

---

### 7.4 Production Network Recommendations

Recommendations:

    Ensure stable network between storage nodes.
    Use 10Gbps or higher bandwidth for production.
    Avoid severe contention between Longhorn replica synchronization and business traffic.
    Plan independent storage network for critical environments.
    Avoid networks with high packet loss and high latency.
    Deploy Longhorn replicas across data centers with extreme caution.

Not Recommended:

    Use Longhorn multi-replicas on unstable public networks or cross-regional networks.
    Run high I/O Longhorn Volumes on high latency links.
    Mistake node network quality issues for Longhorn itself problems.

---

## VIII. Resource Planning Recommendations

### 8.1 Longhorn Component Resources

Monitor:

    longhorn-manager
    longhorn-csi-plugin
    csi-provisioner
    csi-attacher
    csi-resizer
    instance-manager
    engine
    replica
    share-manager

Check:

    kubectl -n longhorn-system get pods -o wide
    kubectl -n longhorn-system top pods

If there is no top command data, install metrics-server.

---

### 8.2 Check Node Resources

Execute:

    kubectl top nodes

Check node details:

    kubectl describe node <node-name>

Monitor:

    CPU usage
    Memory usage
    Allocatable
    Requests
    Limits
    DiskPressure
    MemoryPressure
    PIDPressure

---

### 8.3 instance-manager Resources

Instance Manager manages Engine and Replica instances.

Check:

    kubectl -n longhorn-system get pods | grep instance-manager

Check details:

    kubectl -n longhorn-system describe pod <instance-manager-pod>

Check logs:

    kubectl -n longhorn-system logs <instance-manager-pod> --tail=100

Production recommendations:

    Do not let node CPU run at full capacity long-term.
    Do not let Longhorn components and business workloads compete for resources.
    Set reasonable resource requests for critical components.
    Monitor instance-manager CPU, memory, and restart counts.

---

### 8.4 Business Pod Resource Limits

Business Pods without resource limits may affect node stability.

Recommendations:

    Set requests.
    Set reasonable limits.
    Database applications especially need clear CPU/Memory definitions.
    Avoid business Pods competing with Longhorn component resources.
    Avoid node memory pressure causing system jitter.

Example:

    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "2"
        memory: "2Gi"

---

## IX. StorageClass Production Tiered Design

### 9.1 Do Not Directly Modify Default longhorn StorageClass

Production recommendations:

    Keep the default longhorn StorageClass.
    Create new StorageClasses based on business needs.
    Clearly define which StorageClass each business uses.
    Do not arbitrarily modify existing StorageClass parameters during operation.
    Parameter changes only affect new PVCs and do not automatically modify existing Volumes.

---

### 9.2 General Purpose StorageClass

Suitable for ordinary business:

    2 replicas
    ext4
    Default data locality disabled or best-effort
    ReclaimPolicy decided based on business needs

Example:

    cat > sc-longhorn-standard.yaml <<'EOF'
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: longhorn-standard
    provisioner: driver.longhorn.io
    allowVolumeExpansion: true
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    parameters:
      numberOfReplicas: "2"
      staleReplicaTimeout: "30"
      fsType: "ext4"
      dataLocality: "disabled"
    EOF

Apply:

    kubectl apply -f sc-longhorn-standard.yaml

---

### 9.3 High Availability StorageClass

Suitable for relatively important business:

    3 replicas
    Requires at least 3 available data nodes
    Suitable for scenarios with ample capacity and network

Example: /think

```bash
cat > sc-longhorn-ha.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ha
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "best-effort"
EOF
```

**Apply:**

```bash
kubectl apply -f sc-longhorn-ha.yaml
```

**Notes:**

- Retain is more suitable for important data, but requires manual recycling.
- 3 replicas require evaluation of capacity and node count.
- dataLocality: best-effort attempts to localize data near workloads, but does not guarantee localization in all scenarios.

---

### 9.4 Performance-Oriented StorageClass

Suitable for scenarios sensitive to latency where the application itself has replication capabilities.

**Examples:**

- Distributed databases already have multiple replicas.
- Applications have their own data replication.
- Do not want Longhorn to perform additional multi-replica writes.

**Considerations:**

- numberOfReplicas: "1"
- dataLocality: "strict-local"

**High-Risk Warning:**

- strict-local should typically be paired with 1 replica.
- 1 replica lacks node high availability at the Longhorn layer.
- Only applicable for scenarios where the application itself has replication and recovery capabilities.
- Not suitable for ordinary single-instance databases.

**Example:**

```bash
cat > sc-longhorn-local-performance.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-local-performance
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "strict-local"
EOF
```

**Apply:**

```bash
kubectl apply -f sc-longhorn-local-performance.yaml
```

**Production Warning:**

- These StorageClass types must be explicitly marked with risk.
- Not allowed for ordinary business use.
- Must require applications to have their own replication, backup, and recovery capabilities at the application layer.

---

### 9.5 SSD Tiered StorageClass

If nodes and disks are labeled, you can specify the use of high-performance disks.

**Example Concept:**

- nodeSelector: "storage,fast"
- diskSelector: "ssd,fast"

**StorageClass Example:**

```bash
cat > sc-longhorn-ssd.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ssd
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  diskSelector: "ssd,fast"
  nodeSelector: "storage,fast"
  dataLocality: "best-effort"
EOF
```

**Apply:**

```bash
kubectl apply -f sc-longhorn-ssd.yaml
```

**Notes:**

- nodeSelector / diskSelector use Longhorn node and disk label concepts.
- Need to set corresponding tags first in Longhorn UI or Longhorn Node/Disk configuration.
- Not the same mechanism as Kubernetes node labels.
- Must confirm Longhorn node and disk label configurations are correct before use.

---

## 10. Longhorn Node and Disk Label Planning

### 10.1 Why Labels Are Needed

Labels are used for:

- Distinguishing SSD / HDD.
- Distinguishing high-performance nodes / standard nodes.
- Restricting certain PVCs to schedule only on specific nodes.
- Avoiding high-performance workloads on low-performance disks.
- Implementing storage tiering.

---

### 10.2 Label Planning Example

**Node Labels:**

| Longhorn Node Tag | Meaning |
|---|---|
| storage | Can host storage |
| fast | High-performance node |
| standard | Standard node |
| database | Database workload node |

**Disk Labels:**

| Longhorn Disk Tag | Meaning |
|---|---|
| ssd | SSD disk |
| nvme | NVMe disk |
| hdd | HDD disk |
| fast | High-performance disk |
| capacity | Capacity-oriented disk |

---

### 10.3 Practical Check Longhorn Nodes

View Longhorn nodes:

```bash
kubectl -n longhorn-system get nodes.longhorn.io
```

View detailed information:

```bash
kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01
```

View YAML:

```bash
kubectl -n longhorn-system get nodes.longhorn.io k8s-worker01 -o yaml
```

**Note:**

Tags can be configured via Longhorn UI.
Production recommendation: document each node, disk, label, and purpose in the documentation.
Do not arbitrarily modify storage labels, as it may affect the scheduling of new Volumes.

---

## 11. Creating a Performance Test PVC

### 11.1 Creating a Namespace

Execute:

    kubectl create namespace longhorn-performance-demo

Check:

    kubectl get ns longhorn-performance-demo

---

### 11.2 Creating a Working Directory

Execute:

    mkdir -p ~/longhorn-performance-demo
    cd ~/longhorn-performance-demo

---

### 11.3 Creating a Test PVC

Create the file:

    cat > 01-pvc-standard.yaml <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: fio-pvc-standard
      namespace: longhorn-performance-demo
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-standard
      resources:
        requests:
          storage: 5Gi
    EOF

If there is no longhorn-standard, temporarily change to:

    storageClassName: longhorn

Apply:

    kubectl apply -f 01-pvc-standard.yaml

Check:

    kubectl get pvc -n longhorn-performance-demo
    kubectl get pv

---

### 11.4 Viewing Longhorn Volumes

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide

Find the corresponding Volume:

    kubectl describe pvc fio-pvc-standard -n longhorn-performance-demo

Record the Volume name:

    export PERF_VOLUME=<replace with actual Volume name>

Check:

    kubectl -n longhorn-system describe volumes.longhorn.io ${PERF_VOLUME}

---

## 12. Using fio to Test PVC Basic Performance

### 12.1 fio Test Notes

fio can test:

    Sequential read
    Sequential write
    Random read
    Random write
    Mixed read/write
    IOPS
    Bandwidth
    Latency

Notes:

    fio testing will generate I/O pressure.
    It can be executed in experimental environments.
    Production environments must select maintenance windows.
    Do not directly stress-test production business PVCs.
    Test results are affected by nodes, disks, network, replica count, caching, and load.

---

### 12.2 Creating a fio Test Pod

Create the file:

    cat > 02-fio-pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: fio-test-pod
      namespace: longhorn-performance-demo
    spec:
      restartPolicy: Never
      containers:
        - name: fio
          image: xridge/fio:latest
          imagePullPolicy: IfNotPresent
          command:
            - sh
            - -c
            - "sleep 3600"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: fio-pvc-standard
    EOF

Apply:

    kubectl apply -f 02-fio-pod.yaml

Check:

    kubectl get pod -n longhorn-performance-demo -o wide

If the image pull fails:

    Sync the fio image to your Harbor or Aliyun repository.
    Modify the image field.
    Do not break containerd global configuration for a single test image.

---

### 12.3 Checking fio Availability

Execute:

    kubectl exec -n longhorn-performance-demo fio-test-pod -- fio --version

Check the mount:

    kubectl exec -n longhorn-performance-demo fio-test-pod -- df -hT /data

---

### 12.4 Sequential Write Test

Execute:

    kubectl exec -n longhorn-performance-demo fio-test-pod -- fio \
      --name=seqwrite \
      --directory=/data \
      --rw=write \
      --bs=1M \
      --size=1G \
      --numjobs=1 \
      --iodepth=16 \
      --direct=1 \
      --runtime=60 \
      --time_based \
      --group_reporting

Monitor:

    WRITE bandwidth
    IOPS
    latency
    clat
    lat percentiles

---

### 12.5 Sequential Read Test

First ensure there is a test file, then execute:

kubectl exec -n longhorn-performance-demo fio-test-pod -- fio \
  --name=seqread \
  --directory=/data \
  --rw=read \
  --bs=1M \
  --size=1G \
  --numjobs=1 \
  --iodepth=16 \
  --direct=1 \
  --runtime=60 \
  --time_based \
  --group_reporting

Focus on:

  READ bandwidth
  latency
  Whether it is significantly lower than expected

---

### 12.6 Random Write Test

Execute:

  kubectl exec -n longhorn-performance-demo fio-test-pod -- fio \
    --name=randwrite \
    --directory=/data \
    --rw=randwrite \
    --bs=4k \
    --size=1G \
    --numjobs=1 \
    --iodepth=16 \
    --direct=1 \
    --runtime=60 \
    --time_based \
    --group_reporting

Focus on:

  random write IOPS
  latency
  Whether there is significant high latency

---

### 12.7 Random Read Test

Execute:

  kubectl exec -n longhorn-performance-demo fio-test-pod -- fio \
    --name=randread \
    --directory=/data \
    --rw=randread \
    --bs=4k \
    --size=1G \
    --numjobs=1 \
    --iodepth=16 \
    --direct=1 \
    --runtime=60 \
    --time_based \
    --group_reporting

---

### 12.8 Mixed Read/Write Test

Execute:

  kubectl exec -n longhorn-performance-demo fio-test-pod -- fio \
    --name=randrw \
    --directory=/data \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --size=1G \
    --numjobs=1 \
    --iodepth=16 \
    --direct=1 \
    --runtime=60 \
    --time_based \
    --group_reporting

Notes:

  70% read, 30% write.
  Closer to some database and application scenarios.
  But still cannot fully represent real business

---

## Thirteen、Hands-on Three: Observing Node Resources During Testing

### 13.1 Observe Pod's Node

Execute:

  kubectl get pod fio-test-pod -n longhorn-performance-demo -o wide

Record node:

  NODE

---

### 13.2 Observe Longhorn Volume Status

Execute:

  kubectl -n longhorn-system get volumes.longhorn.io -o wide
  kubectl -n longhorn-system get replicas.longhorn.io -o wide
  kubectl -n longhorn-system get engines.longhorn.io -o wide

---

### 13.3 Observe Node CPU and Memory

Execute:

  kubectl top nodes
  kubectl -n longhorn-system top pods
  kubectl top pod -n longhorn-performance-demo

If metrics-server is not available, install it first before proceeding.

---

### 13.4 Observe Disk I/O

Execute on the relevant Worker node:

  iostat -x 1 10

If iostat is not available:

  apt update
  apt install -y sysstat

Focus on:

  r/s
  w/s
  rkB/s
  wkB/s
  await
  %util

Judgment:

  %util consistently approaching 100% indicates the disk is nearly saturated.
  await significantly increasing indicates high I/O latency.
  If write is slow, first check the disk, then the network, then Longhorn.

---

### 13.5 Observe Network

Execute on the node:

  sar -n DEV 1 10

If sar is not available:

  apt install -y sysstat

Alternatively:

  ip -s link
  iftop

If iftop is not available:

  apt install -y iftop

Focus on:

  Whether the network between nodes is saturated.
  Whether there are packet losses.
  Whether there are error packets.
  Whether the network significantly increases during Rebuild.

---

## Fourteen、Hands-on Four: Compare Performance with Different Replica Counts

### 14.1 Create 1 Replica Test StorageClass

Create: /think

```bash
cat > 03-sc-perf-replica-1.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-perf-replica-1
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "strict-local"
EOF

Apply:

kubectl apply -f 03-sc-perf-replica-1.yaml

---

### 14.2 Creating a 2-Replica Test StorageClass

Create:

cat > 04-sc-perf-replica-2.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-perf-replica-2
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "best-effort"
EOF

Apply:

kubectl apply -f 04-sc-perf-replica-2.yaml

---

### 14.3 Creating a 3-Replica Test StorageClass

Prerequisites:

At least 3 available Longhorn data nodes.
Otherwise, Degraded or insufficient replica scheduling may occur.

Create:

cat > 05-sc-perf-replica-3.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-perf-replica-3
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "best-effort"
EOF

Apply:

kubectl apply -f 05-sc-perf-replica-3.yaml

---

### 14.4 Test Comparison Method

Recommended to create separately:

fio-pvc-replica-1
fio-pvc-replica-2
fio-pvc-replica-3

Each PVC:

Same capacity
Same fio parameters
Same node
Same test duration
Avoid interference from other workloads
Record Volume status and Replica distribution

Comparison metrics:

Sequential write bandwidth
Random write IOPS
Average latency
P95 / P99 latency
Node network traffic
Disk await
Longhorn component CPU

Conclusions typically show:

More replicas mean longer write paths.
Higher network and disk pressure.
A balance is needed between reliability and performance.

---

## Fifteen, Database Application Usage Recommendations

### 15.1 Can Databases Run on Longhorn?

Yes, but with caution.

Suitable for:

Test environment MySQL
Test environment PostgreSQL
Medium-scale business databases
Internal systems with low IOPS requirements
Scenarios with complete backup and recovery drills

Caution for:

Core transaction databases
High-concurrency write databases
Low-latency critical databases
Large-scale production databases
Databases sensitive to storage latency

---

### 15.2 Risks of Using Longhorn for Databases

Risks:

Write amplification.
Replica synchronization increases latency.
Node network jitter affects I/O.
Replica Rebuild impacts performance.
Snapshot / Backup may not guarantee database application consistency.
Business latency may rise during node failure recovery.

---

### 15.3 Database Backup Recommendations

MySQL:

mysqldump
xtrabackup
binlog
Master-slave replication
Regular recovery drills

PostgreSQL:

pg_dump
pg_basebackup
WAL archiving
PITR
Regular recovery drills

Longhorn Backup:

Can serve as supplemental volume-level protection.
Should not replace database-native backups.
Database backups must be validated for recoverability.

---

### 15.4 Special Notes for Distributed Databases

If the database already has its own replication mechanism, such as:

TiDB
CockroachDB
Cassandra
Elasticsearch
Kafka
MongoDB ReplicaSet

Need to evaluate:

Application-layer replication + Longhorn multi-replica causing duplicate replication.
Whether write amplification is excessively high.
Whether 1 replica + strict-local is suitable.
Whether it has its own data recovery mechanism.
Whether Longhorn Backup should be used as a supplement.

Conclusion:

Distributed databases may not be suitable for additional 3-replica Longhorn.
Storage strategies should be designed according to the application's architecture.

---

## Sixteen, Prometheus-like Application Usage Recommendations

### 16.1 Prometheus Characteristics

Prometheus local TSDB characteristics:

Continuous writing.  
Time series data grows rapidly.  
Has certain requirements for disk I/O.  
Data retention period affects capacity.  
Crash recovery depends on WAL.

Longhorn can host Prometheus PVC, but note:

    Disk capacity.  
    Write performance.  
    Retention settings.  
    Backup strategy.  
    Whether remote writing is enabled.  
    Whether long-term storage solutions like Thanos / VictoriaMetrics are used.

---

### 16.2 Prometheus Recommendations

Recommendations:

    Use an independent StorageClass for Prometheus PVC.  
    Determine replica count based on business importance.  
    Control Prometheus retention.  
    Monitor Longhorn Volume capacity.  
    Monitor disk I/O.  
    Remote write important monitoring data to external long-term storage.  
    Do not let Prometheus fill Longhorn data disks.

---

## Seventeen, Jenkins / GitLab Class Application Recommendations

### 17.1 Jenkins

Jenkins home can use Longhorn PVC.

Recommendations:

    Use 2 or 3 replicas.  
    Regularly back up Jenkins home.  
    Do not store artifacts long-term in PVC.  
    Large artifacts should be stored in MinIO / Harbor / object storage.  
    Backup Jenkins plugins and task configurations.  
    Perform regular recovery drills.

---

### 17.2 GitLab

GitLab has a large footprint, and using Longhorn requires caution.

Need to evaluate:

    PostgreSQL  
    Redis  
    Gitaly repository data  
    CI artifacts  
    registry  
    backup  
    I/O mode

Recommendations:

    Small-scale experiments can use Longhorn.  
    Production GitLab is recommended to prioritize official architecture and independent backups.  
    Repository and artifact data should be planned separately.  
    Do not rely solely on Longhorn replicas to protect GitLab.

---

## Eighteen, Capacity Management Recommendations

### 18.1 Daily Capacity Check

Commands:

    kubectl -n longhorn-system get nodes.longhorn.io  
    kubectl -n longhorn-system get volumes.longhorn.io -o wide  
    kubectl get pvc -A  
    df -hT /data/longhorn  
    du -sh /data/longhorn

Focus on:

    Total capacity of Longhorn nodes.  
    Capacity already scheduled.  
    Actual used capacity.  
    PVC requested capacity.  
    Snapshot usage.  
    Backup growth.  
    Whether orphaned data exists.

---

### 18.2 Capacity Threshold Recommendations

| Usage Rate | Recommended Actions |  
|---|---|  
| 70% | Start paying attention to trends |  
| 80% | Plan for expansion |  
| 85% | Clearly define expansion window |  
| 90% | High priority handling |  
| 95% | Severe risk, may affect writing and rebuilding |

---

### 18.3 Snapshot Space Management

Risks:

    Long-term failure to clean up snapshots.  
    System-generated snapshot accumulation.  
    Too many recurring snapshots retained.  
    Snapshot usage growth due to heavy writes.  
    Space not reclaimed promptly after file deletion.

Recommendations:

    Configure recurring job retention count.  
    Regularly check snapshots.  
    Clean up unused snapshots.  
    Periodically trim the file system.  
    Reserve reasonable recovery points for critical business.  
    Do not retain snapshots without limits.

---

### 18.4 File System Trim

Applicable scenarios:

    Recover space after deleting large amounts of data.  
    Recover underlying space after snapshot cleanup.  
    Longhorn data usage inconsistent with business expectations.

Try inside Pod:

    fstrim -v /data

Prerequisites:

    Image has fstrim.  
    File system and volume support.  
    Testing required before production execution.

Can also use maintenance Pod with tool image.

---

## Nineteen, Monitoring and Alerting Recommendations

### 19.1 Must Monitor

Must monitor:

    Volume Healthy / Degraded / Faulted  
    Replica status  
    Replica Rebuild status  
    Longhorn Node status  
    Longhorn Disk available capacity  
    longhorn-manager Pod status  
    CSI Pod status  
    instance-manager Pod status  
    Backup success / failure  
    Backup Target availability  
    Node DiskPressure  
    Node NotReady  
    PVC Pending  
    Pod FailedMount

---

### 19.2 Performance Monitoring

Recommended monitoring:

    Volume IOPS  
    Volume Throughput  
    Volume Latency  
    Node disk await  
    Node disk util  
    Node network throughput  
    Longhorn component CPU  
    Longhorn component memory  
    Rebuild duration  
    Backup duration

---

### 19.3 Alert Level

| Alert | Level | Description |  
|---|---|---|  
| Volume Faulted | Critical | Business may be unavailable |  
| Multiple Volume Degraded | Critical | High storage risk |  
| Single Volume Degraded over 10 minutes | Warning | Needs recovery asap |  
| Replica Rebuild over threshold | Warning | Possible resource shortage |  
| Disk usage > 85% | Warning | Needs expansion |  
| Disk usage > 90% | Critical | Writing and rebuilding risk |  
| Backup failure | Warning / Critical | Depends on business importance |  
| Backup Target unavailable | Critical | Unable to backup and recover |  
| CSI component anomaly | Critical | Affects PVC creation and mounting |

---

## Twenty, Node Maintenance Recommendations

### 20.1 Pre-Maintenance Checks

Before maintenance, execute:

kubectl get nodes -o wide
kubectl get pods -A -o wide | grep <node-name>
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system get replicas.longhorn.io -o wide
kubectl -n longhorn-system get nodes.longhorn.io

Confirm:

    No Faulted Volume.
    No long-term Degraded Volume.
    No heavy Rebuild in progress.
    Critical Volumes have Backup.
    Business allows maintenance window.

---

### 20.2 Node Decommissioning Principles

Recommendations:

    Maintain one storage node at a time.
    Do not restart multiple Longhorn data nodes simultaneously.
    Confirm replica distribution before maintenance.
    Confirm Backup before maintenance.
    Observe Rebuild after maintenance.
    Maintain next node only after Volume recovery to Healthy.

---

### 20.3 Drain Considerations

Kubernetes drain:

    kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

High-risk warnings:

    Drain will evict Pods.
    Stateful Pods require confirmation of PDB, PVC, and RWO mount limitations.
    Longhorn Volume detach/attach requires time.
    Do not drain other nodes arbitrarily when Volume is Degraded.

Recovery:

    kubectl uncordon <node-name>

Observation:

    kubectl get nodes
    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

## Twenty-one, Production Use Case Summary

### 21.1 Recommended Use Cases

Recommended:

    Medium-to-small Kubernetes clusters
    Private deployment environments
    Edge clusters
    Development/test clusters
    Medium-to-small stateful applications
    Jenkins
    Prometheus medium-to-small instances
    Nacos
    Redis persistence
    Ordinary business configuration and data directories
    Small-scale MySQL / PostgreSQL
    Bare-metal clusters requiring PVC dynamic provisioning

---

### 21.2 Cautionary Use Cases

Caution:

    Core high-concurrency databases
    Large-scale GitLab
    High-write Prometheus
    Large-scale Elasticsearch
    Kafka high-throughput scenarios
    High IOPS low-latency scenarios
    Cross-datacenter multi-replica
    Low-quality network environments
    HDD hosting high random write workloads
    Production environments without Backup Target

---

### 21.3 Not Recommended Use Cases

Not recommended:

    Using Longhorn as an alternative to MinIO for object storage.
    Using Longhorn to store massive image attachments.
    Using Longhorn to store large-scale artifact packages.
    Using Longhorn to host core production databases without backup.
    Using system disks to host production Longhorn data.
    Single-node Longhorn pretending to be highly available.
    Running critical business without monitoring alerts.
    Believing data is secure without recovery drills.

---

## Twenty-two, Production Deployment Acceptance Checklist

### 22.1 Architecture Acceptance

| Check Item | Requirements | Result |
|---|---|---|
| Data Nodes | At least 2, production recommends 3 or more |  |
| Data Disks | Independent data disks |  |
| Data Directories | /data/longhorn or explicit path |  |
| System Disk | Not hosting production Longhorn data |  |
| Node Network | Stable, production recommends high bandwidth |  |
| iscsid | All nodes running |  |
| NFS Client | RWX scenario already installed |  |
| Longhorn UI | Not exposed to public internet |  |

---

### 22.2 StorageClass Acceptance

| Check Item | Requirements | Result |
|---|---|---|
| Default longhorn | Retained or explicitly managed |  |
| Business StorageClass | Tiered by business |  |
| Replica Count | Matches node count |  |
| ReclaimPolicy | Set according to business risk |  |
| dataLocality | Explicitly enabled or disabled |  |
| nodeSelector/diskSelector | If used, labels are configured |  |
| Change Records | YAML saved |  |

---

### 22.3 Data Protection Acceptance

| Check Item | Requirements | Result |
|---|---|---|
| Backup Target | Configured |  |
| Recurring Backup | Configured |  |
| Snapshot Retention | Planned |  |
| Restore Drill | Completed |  |
| Database Backup | Using database-native backup |  |
| Deletion Approval | PVC/Volume deletion has process |  |

---

### 22.4 Monitoring Acceptance

| Check Item | Requirements | Result |
|---|---|---|
| Volume Status | Monitoring for Healthy/Degraded/Faulted |  |
| Replica Status | Monitoring |  |
| Rebuild | Monitoring |  |
| Disk Capacity | Monitoring |  |
| Node Status | Monitoring |  |
| CSI Components | Monitoring |  |
| Backup Failure | Alert |  |
| Disk Pressure | Alert |  |

---

## Twenty-three, Experiment Cleanup

### 23.1 Delete fio Test Pod

Execute:

    kubectl delete -f 02-fio-pod.yaml --ignore-not-found=true

---

### 23.2 Delete Test PVC

Execute:

    kubectl delete -f 01-pvc-standard.yaml --ignore-not-found=true

Check:

    kubectl get pvc -n longhorn-performance-demo
    kubectl get pv
    kubectl -n longhorn-system get volumes.longhorn.io

---

### 23.3 Delete Test StorageClass

If it's just a temporary experiment, you can delete:

    kubectl delete -f 03-sc-perf-replica-1.yaml --ignore-not-found=true
    kubectl delete -f 04-sc-perf-replica-2.yaml --ignore-not-found=true
    kubectl delete -f 05-sc-perf-replica-3.yaml --ignore-not-found=true

If you created the following files, you can also delete them:

    kubectl delete -f sc-longhorn-standard.yaml --ignore-not-found=true
    kubectl delete -f sc-longhorn-ha.yaml --ignore-not-found=true
    kubectl delete -f sc-longhorn-local-performance.yaml --ignore-not-found=true
    kubectl delete -f sc-longhorn-ssd.yaml --ignore-not-found=true

High-risk warning:

    Do not arbitrarily delete StorageClass still used by PVCs in production environments.
    Deleting StorageClass will not delete existing PV/PVCs, but will affect subsequent PVC creation.

---

### 23.4 Delete Namespace

Run:

    kubectl delete namespace longhorn-performance-demo

---

### 23.5 Delete Local YAML

Run:

    rm -f 01-pvc-standard.yaml
    rm -f 02-fio-pod.yaml
    rm -f 03-sc-perf-replica-1.yaml
    rm -f 04-sc-perf-replica-2.yaml
    rm -f 05-sc-perf-replica-3.yaml
    rm -f sc-longhorn-standard.yaml
    rm -f sc-longhorn-ha.yaml
    rm -f sc-longhorn-local-performance.yaml
    rm -f sc-longhorn-ssd.yaml

---

## Twenty-Four, High-risk Operation Warnings

The following operations must be handled with caution in production environments:

    Perform fio stress testing on production PVCs
    Conduct large-scale I/O testing during peak hours
    Modify the default StorageClass
    Delete StorageClass
    Reduce replica count
    Increase replica count without expanding capacity
    Modify dataLocality
    Modify nodeSelector / diskSelector
    Delete Snapshot
    Clean up Backup
    Drain nodes when Volume is Degraded
    Restart multiple storage nodes simultaneously
    Manually delete data under /data/longhorn
    Directly format Longhorn data disks

Confirm before execution:

    Whether it will affect business operations.
    Whether there is a Backup.
    Whether there is a recovery drill.
    Whether there is a maintenance window.
    Whether there is monitoring in place.
    Whether there is a rollback plan.
    Whether it has been reviewed by two people.

---

## Twenty-Five, Interview Answer Approach

If asked in an interview:

    How to do performance and capacity planning for Longhorn in production environments?

You can answer:

    Longhorn's performance is primarily influenced by disks, network, replica count, and business I/O models. After a Pod writes to a Longhorn PVC, data passes through the Longhorn Engine and synchronizes to multiple Replicas. With 3 replicas, the write path involves multiple nodes and disks, so node-to-node network, disk I/O, and Replica distribution all affect final latency and throughput.
    In production environments, I would first require using dedicated data disks and not recommend placing /data/longhorn on the system disk. If the system disk is filled by Longhorn data, it will affect kubelet, containerd, and even trigger DiskPressure, causing Pod evictions and Volume anomalies. Data disks should use SSDs or NVMe, with capacity and IOPS planning based on business needs.
    Network-wise, since Longhorn replicas synchronize across nodes, the node-to-node network must be stable. Production environments should use high-bandwidth, low-latency networks, and critical environments can plan dedicated storage networks. Network jitter or packet loss may cause Replica anomalies, Volume Degraded, or increased business I/O latency.
    Replica count isn't always better. 1 replica has low performance overhead but no node-level high availability; 2 replicas suit small-to-medium scale or two-node environments; 3 replicas are better for three or more data nodes but have higher capacity and write pressure. For distributed databases with built-in replication capabilities, consider using lower replicas or strict-local to avoid write amplification from redundant replication.
    For StorageClass, I don't recommend directly modifying the default longhorn StorageClass. Instead, create different StorageClasses for business needs, such as standard, ha, ssd, and local-performance. Use parameters like numberOfReplicas, dataLocality, diskSelector, and nodeSelector for tiered management.
    Production environments must also have monitoring and backups. Monitor Volume Healthy/Degraded/Faulted, Replica Rebuild, disk capacity, node status, Backup failures, and CSI component status. Longhorn Replicas are not backups; critical data must have Backup Targets and recurring Backups, with regular Restore drills.
    For database applications, I would carefully evaluate. Longhorn can run small-to-medium scale databases, but core high-concurrency databases require combining database-specific backups, master-slave replication, WAL/binlog, latency requirements, IOPS, and recovery targets for comprehensive judgment, not relying solely on Longhorn replicas.

---

## Twenty-Six, Summary

This article completes the study on Longhorn performance and production recommendations:

1. Longhorn performance is primarily influenced by disk, network, replica count, and business I/O model.  
2. It is not recommended to use the system disk to host Longhorn data in production environments.  
3. /data/longhorn should be prioritized to mount independent data disks.  
4. SSD/NVMe is more suitable for high I/O workloads.  
5. Node-to-node network affects replica synchronization performance.  
6. Production environments should use stable, high-bandwidth, low-latency node networks.  
7. Higher replica count increases capacity usage and write pressure.  
8. Replica Rebuild consumes disk, network, CPU, and memory resources.  
9. It is not recommended to restart multiple Longhorn data nodes simultaneously.  
10. Direct modification of the default longhorn StorageClass is not advised.  
11. Standard, ha, ssd, local-performance, and other StorageClasses should be created based on business needs.  
12. nodeSelector/diskSelector can be used for Longhorn storage tiering.  
13. dataLocality can improve performance in certain scenarios but has usage limitations.  
14. strict-local is typically suitable for applications with their own replication capabilities.  
15. fio can be used for basic I/O testing but cannot represent full production performance.  
16. Databases can use Longhorn, but core high-concurrency databases require careful evaluation.  
17. Applications like Prometheus, Jenkins can use Longhorn, but capacity and backup should be controlled.  
18. Longhorn must be paired with monitoring and alerting.  
19. Longhorn Replica is not a backup; production environments must configure Backup Target.  
20. The next section will summarize Longhorn: From K8s Storage Plugin to Production Operations.  

---

## 27. Reference Documents  

Longhorn Official Documentation:  

    https://longhorn.io/docs/latest/  

Longhorn Best Practices:  

    https://longhorn.io/docs/latest/best-practices/  

Longhorn StorageClass Parameters:  

    https://longhorn.io/docs/latest/references/storage-class-parameters/  

Longhorn Nodes and Volumes:  

    https://longhorn.io/docs/latest/nodes-and-volumes/  

Longhorn Scheduling:  

    https://longhorn.io/docs/latest/nodes-and-volumes/nodes/scheduling/  

Longhorn Storage Tags:  

    https://longhorn.io/docs/latest/nodes-and-volumes/nodes/storage-tags/  

Longhorn Data Locality:  

    https://longhorn.io/docs/latest/high-availability/data-locality/  

Longhorn Auto Balance Replicas:  

    https://longhorn.io/docs/latest/high-availability/auto-balance-replicas/  

Longhorn Replica Rebuilding:  

    https://longhorn.io/docs/latest/advanced-resources/rebuilding/  

Longhorn Monitoring:  

    https://longhorn.io/docs/latest/monitoring/  

Longhorn Backup and Restore:  

    https://longhorn.io/docs/latest/snapshots-and-backups/  

Kubernetes Persistent Volumes:  

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/  

Kubernetes Storage Classes:  

    https://kubernetes.io/docs/concepts/storage/storage-classes/  

Kubernetes Node Maintenance:  

    https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/  

fio Official Repository:  

    https://github.com/axboe/fio