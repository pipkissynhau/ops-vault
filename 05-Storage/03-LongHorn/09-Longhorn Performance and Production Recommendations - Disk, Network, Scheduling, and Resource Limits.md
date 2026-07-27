# Longhorn Performance and Production Recommendations: Disk, Network, Scheduling, and Resource Limits

Recommended Path: 05-Storage/03-LongHorn/09-Longhorn Performance and Production Recommendations: Disk, Network, Scheduling, and Resource Limits.md

Tags: #Longhorn #Performance Optimization #Production Recommendations #Disk Performance #Network Performance #Replication Mechanism #StorageClass #Scheduling Strategy #Resource Limits #Kubernetes #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the ninth in the Longhorn series, focusing on understanding Longhorn's performance limitations, production recommendations, disk planning, network planning, scheduling strategies, and resource limits.

Previous topics covered include:

- Longhorn Basics: Kubernetes Cloud-Native Block Storage and CSI
- Longhorn Architecture: Manager, Engine, Replica, Instance Manager
- Longhorn Installation Planning: Node Disks, Dependencies, and StorageClass
- Longhorn Helm Installation Methodology: Charts, Images, values.yaml, and Version Management
- Longhorn Dynamic Volume Practices: PVCs, PVs, Pod Mounting, and Data Persistence
- Longhorn Replication Mechanism: Number of Replicas, Node Distribution, and Data High Availability
- Longhorn Backup and Recovery: Backup Targets, Snapshots, and Volume Restore
- Longhorn Troubleshooting: Volume Degradation, Replica Rebuilding, and Node Issues

This article addresses the following key questions:

- Where are the performance bottlenecks in Longhorn?
- Why do disk, network, and replication factors affect performance?
- Why is it not recommended to store data directories on system disks in production environments?
- Why isn't a higher number of replicas always better?
- In what scenarios is Longhorn suitable?
- In what scenarios should Longhorn be used with caution?
- Can database applications run on Longhorn?
- How can StorageClass be stratified based on performance?
- How can nodeSelector/diskSelector be used for scheduling?
- What is the role of dataLocality?
- How can Longhorn component resources be monitored?
- How to perform simple I/O tests?
- How to create a production-ready acceptance checklist?

This article emphasizes practicality, providing check commands, StorageClass examples, fio testing methods, resource monitoring commands, and a list of production recommendations.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the main performance bottlenecks of Longhorn.
2. Comprehend how disk performance affects Longhorn.
3. Recognize the impact of network connectivity between nodes on replica synchronization.
4. Analyze how the number of replicas affects capacity, write latency, and network traffic.
5. Understand why it is recommended to use dedicated data disks in production environments.
6. Be able to check the disk capacity and I/O status of Longhorn nodes.
7. Monitor node network connectivity and bandwidth.
8. View the status of Longhorn volumes, replicas, and engines.
9. Create volumes with different replication strategies using StorageClass.
10. Plan storage stratification using nodeSelector/diskSelector.
11. Understand the role and limitations of dataLocality.
12. Perform basic I/O tests on Longhorn PVCs using fio.
13. Evaluate test results to determine their suitability for production use.
14. Design production baselines for Longhorn capacity, backup, monitoring, and alerts.
15. Clearly identify the appropriate and inappropriate applications for Longhorn.

---

## III. Key Conclusions

### 3.1 Longhorn is Not a Universal High-Performance Storage Solution

Longhorn's strengths include:

- Cloud-nativity
- Easy installation and management
- Good integration with Kubernetes PVCs
- Support for replication, snapshots, backup, and recovery
- Suitable for small to medium-scale Kubernetes persistent storage needs

However, Longhorn is not equivalent to:

- High-end SAN storage solutions
- Enterprise-grade all-flash arrays
- Large-scale Ceph clusters
- Cloud vendors' high-performance cloud disks
- Database-specific high-performance storage solutions
- Object storage systems

In production environments, Longhorn should be used according to specific use cases, rather than for all data storage needs.

---

### 3.2 Longhorn Performance is Primarily Affected by Four Factors

The main factors include:

1. Disk performance
2. Network connectivity between nodes
3. Number of replicas
4. Workload characteristics

More detailed considerations include:

- Differences between SSDs and HDDs
- Variations in SATA, SAS, and NVMe interfaces
- Differences in network speeds (1Gbps vs. 10Gbps)
- Network latency and packet loss
- The number of replicas (1, 2, or 3)
- Whether Replica Rebuilding is underway
- The presence of numerous snapshots
- Whether the workload involves high-frequency random writes
- Simultaneous high I/O operations on multiple volumes

---

### 3.3 Priority of Longhorn---

### 6.2 Recommendations for Data Disk File Systems

Common choices:

    xfs
    ext4

Recommendations:

    Use mature and stable file systems.
    Do not experiment with unfamiliar file systems in production environments.
    Keep mounting parameters simple and controllable.
    Enable noatime to reduce unnecessary metadata writes.
    Write mounting configurations to /etc/fstab.

Example fstab entries:

    UUID=<actualUUID> /data xfs defaults,noatime 0 0
    or:
    UUID=<actualUUID> /data ext4 defaults,noatime 0 0

Verification commands:

    df -hT /data/longhorn
    lsblk -f
    mount | grep /data

---

### 6.3 Disk Performance Tiering

If a node has different types of disks, storage tiering can be implemented.

Example:

| Tier      | Disk Type          | Suitable for         |
|-----------|-----------------|-----------------------|
| Fast       | NVMe SSD           | Databases, Prometheus, low-latency tasks |
| Standard   | SATA SSD           | General middleware, Jenkins, Nacos     |
| Capacity    | HDD                | Infrequent access data, testing environments |

Longhorn can utilize:

    node tags
    disk tags
    StorageClass nodeSelector
    StorageClass diskSelector

to assign different storage tiers to various PVCs.

---

### 6.4 Capacity Planning Formula

Basic formula:

    Actual required capacity ≈ Requested PVC capacity × Number of replicas + Snapshot reserve + Rebuild reserve + Growth reserve

Example:

    Total requested capacity for business PVCs: 500GiB
    Number of replicas: 2
    Snapshot reserve: 20%
    Rebuild and growth reserves: 30%

Estimated total capacity:

    500GiB × 2 = 1000GiB
    Snapshot reserve: 200GiB
    Rebuild/growth reserve: 300GiB
    Total planned capacity: Approximately 1500GiB

Production tips:

    Do not plan disk capacity solely based on the requested PVC size.
    Do not forget to consider the number of replicas and snapshot requirements.
    Account for the additional load during Rebuild operations.
    Ensure that disk usage does not exceed 80% for extended periods.

---

## VII. Network Planning Recommendations

### 7.1 Why the Network Affects Longhorn Performance

Longhorn's multi-replica data writes require synchronization between nodes.

If replicas are distributed across different nodes:

    Data writes must go through the network between nodes.
    Network latency can affect write speeds.
    Bandwidth limitations can impact throughput.
    Network instability can lead to decreased performance or read-only volumes.

---

### 7.2 Network Inspection Commands

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

Detect interface errors:

    ip -s link

If ethtool is installed:

    ethtool <interface-name>

Example:

    ethtool ens33

---

### 7.3 Using iperf3 to Test Node-to-Node Bandwidth

Installation:

    apt update
    apt install -y iperf3

Run the server on k8s-worker01:

    iperf3 -s

Run the client on k8s-worker02:

    iperf3 -c 10.0.0.21

For reverse testing:

    iperf3 -c 10.0.0.21 -R

Note:

    iperf3 measures the network performance between nodes.
    It does not directly reflect Longhorn PVC performance.
    However, it can help identify potential network bottlenecks.

---

### 7.4 Production Network Recommendations

Suggestions:

    Ensure stable networks between storage nodes.
    Prefer 10Gbps or higher bandwidth for production use cases.
    Avoid excessive competition between Longhorn replica synchronization and regular traffic.
    Consider dedicated storage networks for critical environments.
    Avoid networks with high packet loss or latency.
    Be extremely cautious when deploying Longhorn replicas across different data centers.

Disadvantages:

    Do not use unstable public networks or cross-regional connections for Longhorn multi-replica setups.
    Do not run I/O-intensive Longhorn volumes on high-latency links.
    Do not mistake network issues for Longhorn component problems.

---

## VIII. Resource Planning Recommendations

### 8.1 Resources Required for Longhorn Components

Pay attention to the following components:

volumeBindingMode: Immediate
parameters:
      numberOfReplicas: "3"
      staleReplicaTimeout: "30"
      fsType: "ext4"
      dataLocality: "best-effort"
    EOF

Application:

    kubectl apply -f sc-longhorn-ha.yaml

Explanation:

    Retain is more suitable for important data, but requires manual cleanup.
    When setting 3 replicas, it is necessary to consider capacity and the number of nodes.
    The dataLocality: best-effort option attempts to place data near the workload, but does not guarantee localization in all scenarios.

---

### 9.4 Performance-oriented StorageClass

This type is suitable for scenarios where latency is a critical factor, but the application already has its own replication capabilities.

Examples:

    Distributed databases may already have multiple replicas.
    Applications that handle data replication themselves.
    Situations where additional Longhorn-based replication would increase write overhead.

Consider setting:

    numberOfReplicas: "1"
    dataLocality: "strict-local"

High-risk warning:

    strict-local should generally be used with only 1 replica.
    A single replica does not provide high availability at the Longhorn layer.
    It is only appropriate for applications that already have replication and recovery mechanisms in place.
    Not suitable for ordinary single-instance databases.

Example:

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

Application:

    kubectl apply -f sc-longhorn-local-performance.yaml

Production note:

    This type of StorageClass must clearly indicate its risks.
    It should not be used in general business operations without proper caution.
    The application layer must have its own replication, backup, and recovery capabilities.

---

### 9.5 SSD-tiered StorageClass

If nodes and disks are labeled, you can specify the use of high-performance disks.

Example approach:

    nodeSelector: "storage,fast"
    diskSelector: "ssd,fast"

Sample StorageClass configuration:

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

Application:

    kubectl apply -f sc-longhorn-ssd.yaml

Explanation:

    The nodeSelector/diskSelector use Longhorn's node and disk labeling concepts.
    You must first configure the corresponding tags in the Longhorn UI or through Longhorn Node/Disk settings.
    This is not the same mechanism as Kubernetes node labels.
    Ensure that the tag configurations for nodes and disks are correct before using this option.

---

## Section Ten: Planning Longhorn Nodes and Disk Labels

### 10.1 Why Use Labels

Labels are used to:

    Differentiate between SSDs and HDDs.
    Distinguish between high-performance nodes and regular nodes.
    Restrict certain PVCs from being scheduled only on specified nodes.
    Prevent high-performance workloads from being placed on low-performance disks.
    Enable storage tiering.

---

### 10.2 Example Label Planning

Node labels:

| Longhorn Node Tag | Meaning |
|---|---|
| storage | Suitable for storing data |
| fast | High-performance nodes |
| standard | Regular nodes |
| database | Nodes dedicated to database services |

Disk labels:

| Longhorn Disk Tag | Meaning |
|---|---|
| ssd | SSD disks |
| nvme | NVMe disks |
| hdd | HDD disks |
| fast | High-performance disks |
| capacity | Capacity-oriented disks |

---

### 10.3 Practical Check of Longhorn Nodes

Viewing Longhorn nodes:

    kubectl -n longhorn-system get nodes.longhorn.io

Viewing details:

    kubectl -n longhorn-system describe nodes.longhorn.io k8s-worker01

Viewing the YAML configuration:

    kubectl -n longhorn-system get nodes.longhorn.io k8s-worker01 -o yaml

Explanation:

    Label configurations can be managed through the Longhorn UI.
    In production, it is recommended to document each node      restartPolicy: Never
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

Application:

    kubectl apply -f 02-fio-pod.yaml

View:

    kubectl get pod -n longhorn-performance-demo -o wide

If image pull fails:

    Synchronize the fio image to your own Harbor or Alibaba Cloud repository.
    Modify the image field.
    Do not disrupt the global containerd configuration for a test image.

---

### 12.3 Check if fio is available

Execute:

    kubectl exec -n longhorn-performance-demo fio-test-pod -- fio --version

View mounts:

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

Pay attention to:

    WRITE bandwidth
    IOPS
    latency
    clat
    lat percentiles

---

### 12.5 Sequential Read Test

First, ensure there is a test file, then execute:

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

Pay attention to:

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

Pay attention to:

    Random write IOPS
    latency
    Whether there are significantly high delays

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

### 12.8 Mixed Read and Write Test

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

Explanation:

    70% read, 30% write.
    This simulates some database and application scenarios better.
    However, it still cannot fully represent real-world business operations.

---

## Chapter Thirteen: Practical Operation Three: Observing Node Resources During Testing

### 13.1 Observing the Node Where the Pod Is Located

Execute:

    kubectl get pod fio-test-pod -n longhorn-performance-demo -o wide

Record the node:

    NODE

---

### 13.2 Observing the Status of Longhorn Volumes

Execute:

    kubectl -n longhorn-system get volumes.longhorn.io -o wide
    kubectl -n longhorn-system get replicas.longhorn.io -o wide
    kubectl -n longhorn-system get engines.longhorn.io -o wide

---

### 13.3 Observing Node CPU and Memory Usage

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

Application:

kubectl apply -f 04-sc-perf-replica-2.yaml

---

### 14.3 Creating a StorageClass with 3 Replicas for Testing

Prerequisites:

At least 3 available Longhorn data nodes are required.
Otherwise, degradation or insufficient replica scheduling may occur.

Creation:

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

Application:

kubectl apply -f 05-sc-perf-replica-3.yaml

---

### 14.4 Testing and Comparison Methods

It is recommended to create the following separately:

fio-pvc-replica-1
fio-pvc-replica-2
fio-pvc-replica-3

For each PVC:

Ensure the same capacity, fio parameters, nodes, and testing time.
Avoid interference from other services.
Record the Volume status and Replica distribution.

Comparison indicators include:

Sequential write bandwidth
Random write IOPS
Average latency
P95 / P99 latency
Node network traffic
Disk wait times
Longhorn component CPU usage

Conclusions usually reveal:

The more replicas there are, the longer the write path becomes.
Higher network and disk pressures may occur.
A balance between reliability and performance is necessary.

---

## Chapter Fifteen: Recommendations for Using Database Applications with Longhorn

### 15.1 Can Databases Run on Longhorn?

Yes, but with caution.

Suitable for:

Test environments for MySQL and PostgreSQL
Small to medium-sized business databases
Internal systems with low IOPS requirements
Scenarios where complete backup and recovery drills are necessary

Cautions:

Core transaction databases
Databases with high-concurrency write operations
Databases requiring extremely low latency
Large-scale production databases
Databases that are sensitive to storage latency

---

### 15.2 Risks of Using Longhorn for Databases

Risks include:

Write amplification
Increased delay in replica synchronization
Network fluctuations affecting I/O performance
Performance impacts due to Replica Rebuild
Snapshot/Backup methods may not ensure data consistency for database applications
Possible increased business delays during node failures and recovery periods

---

### 15.3 Database Backup Recommendations

For MySQL:

Use mysqldump, xtrabackup, or binlog
Implement master-slave replication
Conduct regular recovery drills

For PostgreSQL:

Use pg_dump or pg_basebackup
Utilize WAL archives and PITR
Perform regular recovery drills

For Longhorn backups:

They can serve as supplementary protection at the volume level.
However, they should not replace the database's own backup mechanisms.
It is essential to verify that database backups are truly recoverable.

---

### 15.4 Special Considerations for Distributed Databases

If a distributed database already has its own replication mechanism, such as:

TiDB, CockroachDB, Cassandra, Elasticsearch, Kafka, or MongoDB ReplicaSet

It is necessary to evaluate whether adding additional Longhorn replicas would lead to duplicate replication.
Check if write amplification becomes excessive.
Consider whether using 1 replica with strict-local configuration is appropriate.
Assess if the distributed database has its own data recovery mechanisms.
Determine if Longhorn backups are needed as an extra layer of protection.

Conclusion:

For distributed databases, adding additional 3 Longhorn replicas may not always be necessary.
It is crucial to design storage strategies based on the specific architecture of the application itself.

---

## Chapter Sixteen: Recommendations for Using Prometheus with Longhorn

### 16.1 Characteristics of Prometheus

Prometheus's local TSDB has the following characteristics:

Continuous writing
Rapid growth of time-series data
Certain requirements for disk I/O performance
Data retention periods affect storage capacity
Crash recovery depends on WAL files

Longhorn can support Prometheus PVCs, but the following factors need attention:

Disk capacity
Write performance
Retention settings
Backup strategies
Whether remote writing is available
Whether long-term storage solutions like Thanos or VictoriaMetrics are utilized

---

### 16.2 Recommendations for Using Prometheus with Longhorn

It### Instance-Manager Pod Status
Backup success/failure
Availability of backup target
Node DiskPressure
Node NotReady
PVC Pending
Pod FailedMount

---

### 19.2 Performance Monitoring
It is recommended to monitor the following:
Volume IOPS
Volume throughput
Volume latency
Node disk wait time
Node disk utilization
Node network throughput
CPU usage of Longhorn components
Memory usage of Longhorn components
Rebuild duration
Backup duration

---

### 19.3 Alarm Classification
| Alarm | Level | Description |
|---|---|---|
| Volume Faulted | Critical | Business may become unavailable. |
| Multiple volumes degraded | Critical | High storage risk. |
| A single volume degraded for more than 10 minutes | Warning | Requires immediate recovery. |
| Replica rebuild exceeds the threshold | Warning | Possible resource shortage. |
| Disk utilization > 85% | Warning | Expansion is needed. |
| Disk utilization > 90% | Critical | Risk of writing and rebuilding issues. |
| Backup failure | Warning/Critical | Depends on the importance of the business. |
| Backup target unavailable | Critical | Unable to perform backup and recovery. |
| CSI component exception | Critical | Affects PVC creation and mounting.

---

## Section XX: Node Maintenance Recommendations

### 20.1 Pre-Maintenance Checks
Perform the following before maintenance:
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep <node-name>
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system get replicas.longhorn.io -o wide
kubectl -n longhorn-system get nodes.longhorn.io

Confirm the following:
No faulted volumes.
No volumes that have been degraded for a long time.
No volumes undergoing extensive rebuilds.
Critical volumes have backups.
The maintenance window is approved by the business.

---

### 20.2 Principles for Taking Nodes Offline
It is recommended to:
Maintain only one storage node at a time.
Do not restart multiple Longhorn data nodes simultaneously.
Confirm the replica distribution before maintenance.
Confirm that backups are in place before maintenance.
Observe the rebuild process after maintenance.
Move on to the next node only after the volumes have returned to a healthy state.

---

### 20.3 Precautions for Drain Operations
For Kubernetes drain operations:
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

High-risk reminders:
Drain operations will evict Pods.
For stateful Pods, confirm PDB, PVC, and RWO mount restrictions.
Longhorn volume detachments and attachments take time.
Do not randomly drain other nodes when volumes are degraded.

To restore the node:
kubectl uncordon <node-name>

Observe the following after restoration:
kubectl get nodes
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system get replicas.longhorn.io -o wide

---

## Section XXI: Summary of Production-Ready Scenarios

### 21.1 Recommended Usage Scenarios
Recommended for:
Medium and small-scale Kubernetes clusters
Private deployment environments
Edge clusters
Development and testing clusters
Medium and small-scale stateful applications
Jenkins
Medium and small-scale Prometheus instances
Nacos
Redis persistence solutions
General business configuration and data directories
Small-scale MySQL/PostgreSQL installations
Bare-metal clusters that require dynamic PVC provisioning

---

### 21.2 Scenarios Where Use Should Be Cautioned
Use with caution in:
Core, high-concurrency databases
Large-scale GitLab environments
High-write loads for Prometheus
Large-scale Elasticsearch setups
Kafka scenarios with high throughput requirements
Scenarios requiring high IOPS and low latency
Cross-data-center replication setups
Low-quality network environments
HDDs handling a large amount of random writes
Production environments without backup targets

---

### 21.3 Scenarios Where Use Is Not Recommended
Not recommended for:
Using Longhorn as an object storage solution in place of MinIO.
Storing massive amounts of images or attachments with Longhorn.
Storing large-scale product packages using Longhorn.
Using Longhorn to host critical production databases without backups.
Using system disks to store production Longhorn data.
Attempting to simulate high availability with a single-node Longhorn setup.
Running critical business processes without monitoring and alarm mechanisms in place.
Assuming data security is guaranteed without conducting recovery drills.

---

## Section XXII: Production Go-Live Acceptance Checklist

### 22.1 Architecture Acceptance
| Check Item | Requirement | Result |
|---|---|---|
| Data nodes | At least 2; 3 or more recommended for production | |
| Data disks | Independent data disks | |
| Data directories | /data/longhorn or a specified path | |
| System disk | Should not store production Longhorn data | |
| Node network | Stable; high bandwidth recommended for production | |
| iscsid | Must be running on all nodes | |
|### 23.4 Deleting a Namespace

Execute:

    kubectl delete namespace longhorn-performance-demo

---

### 23.5 Removing Local YAML Files

Execute:

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

## Section Twenty-Four: High-Risk Operation Warnings

The following operations must be carried out with caution in a production environment:

    Conducting fio stress tests on production PVCs
    Performing large-scale I/O tests during peak periods
    Modifying the default StorageClass
    Deleting a StorageClass
    Reducing the number of replicas
    Increasing the number of replicas without scaling out
    Modifying dataLocality
    Adjusting nodeSelector/diskSelector settings
    Deleting snapshots
    Clearing backups
    Drainning nodes when a volume is degraded
    Restarting multiple storage nodes simultaneously
    Manually deleting data from /data/longhorn
    Directly formatting Longhorn data disks

Before performing any of these operations, it is essential to confirm the following:

    Whether the operation will affect business continuity.
    Whether there are existing backups.
    Whether recovery plans have been established.
    Whether there are scheduled maintenance windows.
    Whether monitoring mechanisms are in place.
    Whether rollback strategies are available.
    Whether the operation has been reviewed by two independent individuals.

---

## Section Twenty-Five: Interview Answer Guidelines

If you are asked during an interview:

    How should performance and capacity planning be done for Longhorn in a production environment?

You can answer as follows:

    The performance of Longhorn is primarily influenced by factors such as disks, networking, the number of replicas, and the business's I/O patterns. When data is written to a Longhorn PVC, it first passes through the Longhorn Engine before being synchronized across multiple replicas. With 3 replicas, the write process involves multiple nodes and disks, so network performance between nodes, disk I/O operations, and the distribution of replicas all affect the final latency and throughput.

In a production environment, I would recommend using dedicated data disks for Longhorn and avoid storing /data/longhorn on system disks. If a system disk becomes full due to Longhorn data, it could impact kubelet and containerd operations, potentially leading to DiskPressure issues, Pod eviction, and volume failures. It is advisable to use SSDs or NVMe disks for data storage, depending on the business's IOPS requirements and capacity needs.

Network stability is crucial for Longhorn, especially since multiple replicas need to be synchronized across nodes. In production settings, high-bandwidth, low-latency networks should be utilized, and in critical environments, dedicated storage networks can be planned. Network fluctuations or packet loss may cause replica failures, volume degradations, or increased I/O delays.

The number of replicas is not necessarily the higher, the better. Having 1 replica results in lower performance overhead but no node-level high availability; 2 replicas are suitable for small to medium-scale environments or two-node setups; 3 replicas or more are more appropriate for environments with three or more data nodes, although this comes with increased capacity and write loads. For distributed databases that already have built-in replication capabilities, considering using fewer replicas or strict-local settings can help avoid unnecessary write amplification.

Regarding StorageClass settings, I would not recommend directly changing the default longhorn StorageClass. Instead, it is better to create different StorageClasses based on business requirements, such as standard, ha, ssd, or local-performance. This allows for more tailored management through parameters like numberOfReplicas, dataLocality, diskSelector, and nodeSelector.

Monitoring and backup are essential in a production Longhorn environment. It is necessary to monitor metrics such as Volume Health/Degraded/Faulted status, Replica Rebuilding activities, disk capacity, node health, backup failures, and CSI component status. Since Longhorn replicas do not provide true backup capabilities, it is crucial to configure Backup Targets and set up recurring backups, along with regular Restore drills.

For database applications, a cautious evaluation is required. Longhorn can be used for smaller-scale databases, but for core, high-concurrency databases, additional backup mechanisms such as primary-secondary replication, WAL/binlog, and specific latency and IOPS requirements must also be considered. Relying solely on Longhorn replicas may not be sufficient.

---

## Section Twenty-Six: Summary of This Article

This article has covered various aspects related to the performance and production recommendations for Longhorn:

1. Longhorn's performance is determined