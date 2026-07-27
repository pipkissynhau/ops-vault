# Ceph Performance Optimization: Load Testing, Bottleneck Identification, Parameter Tuning, and Production Baselines

Recommended Path: 05-Storage/01-Ceph/17-Ceph Performance Optimization: Load Testing, Bottleneck Identification, Parameter Tuning, and Production Baselines.md

Tags: #Ceph #Performance Optimization #Load Testing #Bottleneck Identification #OSD #RBD #CephFS #RGW #fio #rados-bench #SRE #Advanced SRE

---

## I. Document Overview

This article is the seventeenth in the Advanced SRE Storage module series for Ceph, focusing on methods for optimizing Ceph performance, conducting load testing, identifying bottlenecks, and establishing production baselines.

Previous topics covered:

- Ceph Cluster Deployment
- OSD Management
- Pools and PGs
- CRUSH Fault Domains
- RBD Block Storage Practices
- CephFS File Storage Practices
- RGW Object Storage Practices
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph Daily Operations and Maintenance
- Ceph Troubleshooting
- Ceph Backup, Recovery, and Disaster Recovery Exercises

This article now delves into the stage of performance governance.

Ceph performance optimization cannot be simply understood as "adjusting a few parameters."

Advanced SRE professionals must first answer:

    Where is the current bottleneck?
    Is it at the client end?
    Is it due to network issues?
    Are the OSDs slowing down?
    Are the disks lagging?
    Is there uneven distribution among Pools and PGs?
    Is resource competition during Recovery/Backfill processes causing delays?
    Is the CephFS MDS struggling?
    Are the RGW gateways slow?
    Is the Kubernetes CSI mounting process too slow?
    Or is the business's IO model not suitable for the current storage type?

This article focuses on:

- Basic principles of Ceph performance optimization
- Preparations before performance testing
- Basic Rados bench load testing
- RBD fio load testing
- CephFS file load testing
- RGW S3 upload and download load testing
- OSD latency analysis
- Disk bottleneck analysis
- Network bottleneck analysis
- The impact of Pools, PGs, and CRUSH on performance
- The impact of Recovery/Backfill on business performance
- Optimization directions for RBD performance
- Optimization directions for CephFS performance
- Optimization directions for RGW performance
- Performance troubleshooting for Kubernetes CSI
- Production environment optimization baselines
- Boundaries for adjusting critical parameters

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand the basic methodology of Ceph performance optimization.
2. Distinguish between capacity-related issues, latency-related issues, throughput-related issues, and metadata-related issues.
3. Use commands like `ceph -s`, `ceph osd perf`, and `ceph osd df` to preliminarily assess performance status.
4. Conduct basic RADOS layer load testing using rados bench.
5. Use fio to perform sequential and random read/write tests on RBD block devices.
6. Use basic commands to test file write and read operations in CephFS.
7. Use AWS CLI to test object upload and download operations with RGW.
8. Utilize tools such as iostat, vmstat, top, and iftop to identify node bottlenecks.
9. Determine the impact of OSD latency, disk latency, network bandwidth, and Recovery tasks on performance.
10. Understand how the number of Pool replicas, the quantity of PGs, and CRUSH fault domains affect performance and stability.
11. Recognize the performance differences between RBD, CephFS, and RGW services.
12. Master the checklist for pre-optimization inspections in a production environment.
13. Identify which parameters can be adjusted cautiously and which ones should not be changed arbitrarily.
14. Establish an advanced SRE methodology that includes "load testing baselines, monitoring observations, targeted parameter tuning, and rollback plans."

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article continues to use the same experimental environment as the Ceph module series.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Testing (optional) |
| 10.0.0.35 | ceph-client ||---|---|---|
| Experiment 1 | Record baseline before performance testing | Low |
| Experiment 2 | Rados bench to test RADOS layer performance | Medium |
| Experiment 3 | Fio to test RBD block storage performance | Medium to High |
| Experiment 4 | CephFS file read and write testing | Medium |
| Experiment 5 | RGW S3 upload and download testing | Medium |
| Experiment 6 | Analysis of OSD latency and disk bottlenecks | Medium |
| Experiment 7 | Network bandwidth testing | Medium |
| Experiment 8 | Observation of the impact of Recovery/Backfill | Medium to High |
| Experiment 9 | Verification of RBD performance optimization directions | Medium |
| Experiment 10 | Verification of CephFS performance optimization directions | Medium |
| Experiment 11 | Verification of RGW performance optimization directions | Medium |
| Experiment 12 | Performance troubleshooting for Kubernetes CSI | Medium |
| Experiment 13 | Cleaning up stress test data | High |

High-risk warnings:

    Stress testing will consume cluster resources.
    Write-based stress testing will actually write data to the system.
    Fio's random write mode may put significant strain on disks and services.
    Before cleaning up Rados bench results, ensure that only test data is removed.
    Do not perform prolonged write-based stress tests in a production environment.

---

## Experiment 7: Record baseline before performance testing

### 7.1 Create the baseline directory

    BASELINE_TIME=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/perf/${BASELINE_TIME}

---

### 7.2 Record Ceph status

    ceph -s > /backup/ceph/perf/${BASELINE_TIME}/ceph-status.txt
    ceph health detail > /backup/ceph/perf/${BASELINE_TIME}/ceph-health-detail.txt
    ceph osd tree > /backup/ceph/perf/${BASELINE_TIME}/ceph-osd-tree.txt
    ceph osd df > /backup/ceph/perf/${BASELINE_TIME}/ceph-osd-df.txt
    ceph osd perf > /backup/ceph/perf/${BASELINE_TIME}/ceph-osd-perf.txt
    ceph pg stat > /backup/ceph/perf/${BASELINE_TIME}/ceph-pg-stat.txt
    ceph df > /backup/ceph/perf/${BASELINE_TIME}/ceph-df.txt
    ceph orch ps > /backup/ceph/perf/${BASELINE_TIME}/ceph-orch-ps.txt

---

### 7.3 Record node resources

Execute the following commands on each Ceph node:

    hostname
    uptime
    free -h
    df -hT
    lsblk
    iostat -x 1 3
    vmstat 1 3

Save the output in respective files.

---

### 7.4 Why record a baseline?

A baseline is used to compare:

    Whether the system is healthy before stress testing.
    If any abnormalities occur during testing.
    Whether the system returns to normal after testing.
    Whether optimizations actually improve performance.
    How to diagnose issues after failures.

Without a baseline, it would be difficult to determine whether optimizations are effective.

---

## Experiment 8: Rados bench to test RADOS layer performance

### 8.1 Create a test pool

Execute the following commands on the Ceph management node:

    ceph osd pool create bench-pool 64
    ceph osd pool application enable bench-pool rados
    ceph osd pool set bench-pool size 3
    ceph osd pool set bench-pool min_size 2

Check the created pool:

    ceph osd pool ls
    ceph osd pool get bench-pool size
    ceph osd pool get bench-pool min_size

---

### 8.2 Write-based stress testing

High-risk warnings:

    Rados bench will actually write data to the test objects.
    Do not perform this during peak production times.
    Be careful to control the duration and number of concurrent tasks.

Perform a 60-second write test:

    rados bench -p bench-pool 60 write --no-cleanup

Explanation:

    -p specifies the pool to use for the test.
    60 indicates that the test will last for 60 seconds.
    write instructs Rados bench to perform a write operation.
    --no-cleanup prevents automatic cleanup of test objects after the test, allowing subsequent read tests.

Observe the output:

- Bandwidth
- Average IOPS
- Latency
- Total time
- Total writes

---

### 8.3 Sequential read testing

Execute the following command:

    rados bench -p bench-pool 60 seq

---

### 8.4 Random read testing

Execute the following command:

    rados bench -p bench-pool 60Random writes place significant stress on HDDs. It is important not to set the test duration or concurrent tasks too high. In a production environment, such tests should be conducted with extreme caution.

---

### 9.8 Random Read Testing

Use the following command to perform random read tests:

    fio --name=rbd-rand-read \
      --filename=/dev/rbd0 \
      --direct=1 \
      --ioengine=libaio \
      --rw=randread \
      --bs=4k \
      --iodepth=32 \
      --numjobs=1 \
      --runtime=60 \
      --time_based \
      --group_reporting

---

### 9.9 Mixed Read and Write Testing

Use the following command to perform mixed read and write tests:

    fio --name=rbd-rand-rw \
      --filename=/dev/rbd0 \
      --direct=1 \
      --ioengine=libaio \
      --rw=randrw \
      --rwmixread=70 \
      --bs=4k \
      --iodepth=32 \
      --numjobs=1 \
      --runtime=60 \
      --time_based \
      --group_reporting

Explanation:

The `rwmixread=70` parameter indicates that 70% of the operations should be reads and 30% writes.

---

### 9.10 Observations During Testing

On the Ceph side, you can use the following commands to monitor performance:

    ceph -s
    ceph osd perf
    ceph osd df
    ceph pg stat

On each node, you can also use:

    iostat -x 1
    vmstat 1
    top
    iftop

---

### 9.11 Cleaning Up RBD Test Resources

To clean up the test resources, perform the following steps:

Unmap the RBD volume:

    rbd unmap /dev/rbd0

Delete the test image:

    rbd rm rbd-perf-pool/rbd-perf-image

Delete the pool:

    ceph config set mon mon_allow_pool_delete true
    ceph osd pool rm rbd-perf-pool rbd-perf-pool --yes-i-really-really-mean-it
    ceph config set mon mon_allow_pool_delete false

---

## Section Ten: Experiment Four: CephFS File Read and Write Testing

### 10.1 Prerequisites

Make sure that CephFS has been created and mounted on the client system, for example, at `/mnt/cephfs-kernel`. You can verify its installation by running:

    df -hT | grep ceph

---

### 10.2 Large File Writing Test

Create a test directory:

    mkdir -p /mnt/cephfs-kernel/perf-demo

Write a 1GB file:

    dd if=/dev/zero of=/mnt/cephfs-kernel/perf-demo/test-1g.bin bs=1M count=1024 oflag=direct

Record the writing speed.

---

### 10.3 Large File Reading Test

Clearing cache is only recommended for test environments. Proceed with caution:

    sync
    echo 3 > /proc/sys/vm/drop_caches

Read the file:

    dd if=/mnt/cephfs-kernel/perf-demo/test-1g.bin of=/dev/null bs=1M iflag=direct

---

### 10.4 Small File Creation Test

Create 1,000 small files:

    mkdir -p /mnt/cephfs-kernel/perf-demo/small-files

    For each file, create a text file with the name `file-$i.txt` where `$i` ranges from 1 to 1,000.

Calculate the total time taken for this process:

    time sh -c 'for i in $(seq 1 1000); do echo "file-$i" > /mnt/cephfs-kernel/perf-demo/small-files/file-$i.txt; done'

---

### 10.5 Directory Traversal Test

Measure the time taken to traverse the directory:

    time ls -l /mnt/cephfs-kernel/perf-demo/small-files > /dev/null

    Also, measure the time taken to list all files in the directory:

    time find /mnt/cephfs-kernel/perf-demo/small-files -type f > /dev/null

---

### 10.6 Observations During CephFS Testing

Check the status of CephFS and its MDS:

    ceph fs status
    ceph mds stat

Monitor the overall Ceph status:

    ceph -s
    ceph osd perf

Check the performance of the MDS process:

    ceph orch ps --daemon_type mds

Monitor node resources:

    top
    iost## Chapter Twelve: Experiment Six: Analysis of OSD Latency and Disk Bottlenecks

### 12.1 Checking OSD Latency

    `ceph osd perf`

Example:

```
osd commit_latency(ms) apply_latency(ms)
      0                  1                  1
      1                  2                  2
      2                 80                 80
```

If the latency of a particular OSD is significantly higher than that of others, it is necessary to focus on checking the node where this OSD is located.

---

### 12.2 Identifying the Node and Disk Where the OSD Is Located

    `ceph osd metadata osd.2`

Key fields:

- `hostname`
- `bluestore_bdev_dev_node`
- `devices`

---

### 12.3 Checking the Disk on the Node

Log in to the corresponding node:

```
ssh root@<hostname>
```

Execute the following commands:

```
lsblk
iostat -x 1
dmesg | grep -i error
journalctl -k | grep -i error
```

Pay special attention to the `iostat` output:

| Field          | Description                          |
|-----------------|-------------------------------------------|
| r/s              | Read requests per second                |
| w/s              | Write requests per second                |
| rkB/s           | Read throughput in kilobytes per second  |
| wkB/s           | Write throughput in kilobytes per second  |
| await            | Average waiting time                   |
| %util            | Device utilization percentage               |

---

### 12.4Determining Disk Bottlenecks

Common signs include:

- Persistently high `await` values
- `%util` values consistently approaching 100%
- I/O errors reported in `dmesg`
- Significantly higher latency of a single OSD
- Severe fluctuations in business I/O performance

Possible solutions include:

- Verify whether the system is currently in recovery or backfill mode.
- Check the health of the disk.
- Assess the resources of the host machine.
- Ensure that no other processes are competing for I/O resources.
- Replace the disk if necessary.
- Consider long-term optimization measures such as using SSDs/NVMe drives or increasing the number of OSDs.

---

## Chapter Thirteen: Experiment Seven: Network Bandwidth Testing

### 13.1 Using iperf3 for Testing

On `ceph-node01`, start the server:

```
iperf3 -s
```

On `ceph-node02`, execute the client command:

```
iperf3 -c 10.0.0.31
```

For reverse testing:

```
iperf3 -c 10.0.0.31 -R
```

For multi-threaded testing:

```
iperf3 -c 10.0.0.31 -P 4
```

---

### 13.2 Key Points to Consider in Network Testing

Check the following aspects:

- Whether the bandwidth meets expectations.
-是否存在 significant fluctuations in performance.
- Whether there are any retransmissions occurring.
- Whether the bidirectional bandwidth is consistent.
- Whether multi-threaded testing yields improved results.

---

### 13.3 Possible Indicators of Ceph Network Bottlenecks

Possible issues include:

- Abnormal OSD heartbeat responses.
- PG stale conditions.
- Slow operation speeds.
- Prolonged recovery processes.
- High latency in RBD/CephFS operations.
- Slow upload and download speeds for RGW.
- Sluggish data replication between OSDs.

---

### 13.4 Recommendations for Production Network Settings

In a production environment, it is advisable to:

- Separate the Ceph Public Network from the Cluster Network, depending on the scale and budget of the deployment.
- Avoid overloading the OSD replication network with business access traffic.
- Use a stable and low-latency network infrastructure.
- Pay close attention to switch settings, uplink connections, MTU values, packet loss, and retransmissions.
- Avoid using virtualized networks with excessive overselling.
- Monitor for any error packets or packet loss on network interfaces.

---

## Chapter Fourteen: Experiment Eight: Observing the Impact of Recovery/Backfill Processes

### 14.1 Monitoring Recovery Progress

When the cluster undergoes OSD expansion, replacement, or fault recovery operations, check the following command:

```
ceph -s
```

Possible output messages include:

- `recovering`
- `backfilling`
- `remapped`
- `misplaced`

---

### 14.2 Monitoring OSD Latency During Recovery

Use the following command to monitor OSD latency in real-time:

```
watch -n 3 'ceph osd perf'
```

---

### 14.3 Checking Node I/O Performance

On the OSD node, execute the following commands```markdown
ceph df
radosgw-admin bucket stats --bucket=<bucket-name>
radosgw-admin user info --uid=<uid>

Testing:

curl -I http://10.0.0.31:7480

AWS CLI:

aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls

---

### 17.3 Common Performance Issues with RGW

| Issue | Possible Causes |
|---|---|
| Slow Upload | Few RGW instances, slow network, slow OSDs |
| Slow Download | Insufficient bandwidth, proxy restrictions, slow OSDs |
| Slow Bucket Listing | Large number of objects, high pressure on the bucket index |
| Slow Processing of Small Objects | High number of requests, metadata overhead |
| Slow Performance after Proxying | Nginx buffering, large body size, TLS overhead |
| Slow Authentication or Failure | Time synchronization issues, signature verification, Host header problems |

---

### 17.4 Optimization Suggestions for RGW

- Deploy at least multiple RGW instances.
- Use Nginx / HAProxy / Load Balancers in front.
- Use HTTPS for external access and HTTP/HTTPS depending on security requirements internally.
- Evaluate large and small objects separately.
- Control the number of objects in a bucket and their lifecycle.
- Implement quota management for users and buckets.
- Monitor 4xx/5xx errors, latency, and throughput.
- Avoid making a single RGW instance the bottleneck.
- Configure reverse proxy buffering and body size appropriately.
- Properly manage capacity and lifecycle for object storage.

---

## Chapter Eighteen: Experiment Twelve: Performance Troubleshooting for Kubernetes CSI

### 18.1 RBD CSI Performance Troubleshooting

On the Kubernetes side:

```bash
kubectl get pvc -A
kubectl get pv
kubectl get pods -n ceph-csi -o wide
kubectl describe pod -n <namespace> <pod-name>
kubectl get volumeattachment
```

On the Ceph side:

```bash
rbd ls -p k8s-rbd
rbd info k8s-rbd/<image-name>
rbd status k8s-rbd/<image-name>
ceph osd perf
ceph -s
```

On the node side:

```bash
modprobe rbd
lsmod | grep rbd
iostat -x 1
dmesg | tail -100
```

---

### 18.2 CephFS CSI Performance Troubleshooting

On the Kubernetes side:

```bash
kubectl get pvc -A
kubectl get pods -n ceph-csi -o wide
kubectl describe pod -n <namespace> <pod-name>
```

On the CephFS side:

```bash
ceph fs status
ceph mds stat
ceph fs subvolume ls cephfs --group_name csi
ceph osd perf
```

On the node side:

```bash
modprobe ceph
lsmod | grep ceph
dmesg | tail -100
```

---

### 18.3 Common Causes of CSI Performance Issues

| Issue | Possible Reasons |
|---|---|
| Slow Pod Startup | Slow PVC mounting, slow image retrieval, slow CSI Node performance |
| Slow PVC Creation | Slow CSI Controller response, Ceph permission or pool issues |
| Slow RBD I/O | Slow OSDs, poor network performance, RBD feature configuration issues, node problems |
| Slow CephFS I/O | Slow MDS performance, large number of small files, slow OSDs |
| Failed Mount | Issues with Secrets configuration, network connectivity, permissions, driver problems, or faulty node modules |
```

---

## Chapter Nineteen: Performance Data Recording Templates

### 19.1 Basic Information

| Item | Details |
|---|---|
| Test Date | YYYY-MM-DD |
| Tester |  |
| Ceph Version |  |
| Number of Nodes |  |
| Number of OSDs |  |
| Disk Types | HDD / SSD / NVMe |
| Network Speed | 1G / 10G / 25G |
| Pool Configuration |  |
| Number of Replicas |  |
| Number of PGs |  |
| Test Type | RADOS / RBD / CephFS / RGW |
```

---

### 19.2 Pre-Test Status

| Check Item | Result |
|---|---|
| ceph -s |  |
| ceph health detail |  |
| ceph osd perf |  |
| ceph osd df |  |
| ceph pg stat |  |
| Recovery/Backfill Status | Enabled/Disabled |
| Node Load |  |
| Network Status |  |
```

---

### 19.3 Test Results

| Test Item | Parameters | Results |
|---|---|---|
| Rados Write Speed | 60 seconds |  |
### Layer 1: Initial Inspection
First, I will check the `ceph -s` and `ceph health detail` outputs to identify any issues such as down OSDs, degraded PGs, near-full capacity, slow operations, or recovery/backfill processes. Then, I will examine the `ceph osd perf` and `ceph osd df` reports to see if there are any individual OSDs showing significantly high latency, uneven capacity, or nearing full capacity.

### Layer 2: Targeted Node Analysis
Next, I will use commands like `iostat -x 1`, `vmstat`, `top`, `dmesg`, and `journalctl` to analyze disk wait times, utilization, system load, and any disk errors. If an OSD has high latency, I will further investigate whether the issue is due to a disk failure, full IO operations, network anomalies, or resource competition during recovery processes.

### Layer 3: Business-Specific Optimization
Depending on the specific use case, I will apply different optimization strategies:
- **For RBD**: I will check `rbd info`, `rbd status`, client kernel modules, file systems, and perform FIO tests.
- **For CephFS**: I will focus on `ceph fs status`, MDS status, the number of small files, and directory structure.
- **For RGW**: I will examine RGW instances, Nginx/LB settings, Bucket indexes, object sizes, and concurrency.
- **For Kubernetes**: I will look at PVC/PVs, CSI Pods, VolumeAttachments, and kubelet events.

I will not adjust parameters directly from the start. Instead, I will first establish a baseline, determine the appropriate load testing methods, and conduct small-scale tests before making any changes. For example, I might use `rados bench` to test the RADOS layer, `fio` for RBD block devices, or S3 tools for RGW.

When adjusting parameters like recovery/backfill, it is essential to record the original values, observe the impact on business operations, and have a plan in place for rolling back changes if necessary.