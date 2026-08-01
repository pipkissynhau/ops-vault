# Ceph Performance Optimization: Benchmarking, Bottleneck Identification, Parameter Tuning, and Production Baselines

Recommended path: 05-Storage/01-Ceph/17-Ceph Performance Optimization: Benchmarking, Bottleneck Identification, Parameter Tuning, and Production Baselines.md

Tags: #Ceph #PerformanceOptimization #Pressure #BottleneckPositioning #OSD #RBD #CephFS #RGW #fio #rados-bench #SRE #AdvancedSre

---

## I. Document Overview

This is the seventeenth article in the advanced SRE storage module for Ceph, focusing on methods for Ceph performance optimization, benchmarking, bottleneck identification, and production baselines.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH failure domain
- RBD block storage practice
- CephFS file storage practice
- RGW object storage practice
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations
- Ceph troubleshooting
- Ceph backup recovery and disaster recovery drills

This document enters the performance governance phase.

Ceph performance optimization cannot be simply understood as "adjusting a few parameters".

Advanced SREs need to first answer:

    Where is the current slowdown?
    Is it the client slow?
    Is it the network slow?
    Is it the OSD slow?
    Is it the disk slow?
    Is it uneven Pool/PG distribution?
    Is it Recovery/Backfill resource contention?
    Is it CephFS MDS slow?
    Is it RGW gateway slow?
    Is it Kubernetes CSI mount slow?
    Is it the business IO model unsuitable for the current storage type?

This document covers:

- Ceph performance optimization fundamentals
- Preparations for performance benchmarking
- rados bench basic benchmarking
- RBD fio benchmarking
- CephFS file benchmarking
- RGW S3 upload/download benchmarking
- OSD latency analysis
- Disk bottleneck analysis
- Network bottleneck analysis
- Impact of Pool/PG/CRUSH on performance
- Impact of Recovery/Backfill on business performance
- RBD performance optimization directions
- CephFS performance optimization directions
- RGW performance optimization directions
- Kubernetes CSI performance troubleshooting
- Production environment optimization baselines
- High-risk tuning parameter boundaries

---

## II. Learning Objectives

After completing this document, you should be able to:

1. Understand the fundamental methodology of Ceph performance optimization.
2. Distinguish between capacity-related issues, latency-related issues, throughput-related issues, and metadata-related issues.
3. Use ceph -s, ceph osd perf, and ceph osd df to preliminarily assess performance status.
4. Use rados bench for basic RADOS layer benchmarking.
5. Use fio for sequential read/write and random read/write testing on RBD block devices.
6. Use basic commands to perform file write/read testing on CephFS.
7. Use AWS CLI to perform object upload/download testing on RGW.
8. Use tools like iostat, vmstat, top, iftop, etc., to identify node bottlenecks.
9. Determine the impact of OSD latency, disk latency, network bandwidth, and Recovery tasks on performance.
10. Understand the impact of Pool replica count, PG count, and CRUSH failure domains on performance and stability.
11. Understand the performance differences between RBD, CephFS, and RGW workloads.
12. Master the checklist for production environment tuning.
13. Clearly identify which parameters can be cautiously adjusted and which cannot be arbitrarily adjusted.
14. Establish an advanced SRE methodology of "benchmarking baselines, monitoring observation, incremental tuning, and change rollback."

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This document continues using the Ceph module experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (Optional) |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Benchmarking Client |
| 10.0.0.36 | backup-server / rgw-lb | Backup or RGW Entry Point (Optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Kubernetes Cluster

If testing CSI performance, a Kubernetes cluster may be involved:

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.20 | k8s-master | Kubernetes Master |
| 10.0.0.21 | k8s-worker01 | Kubernetes Worker |
| 10.0.0.22 | k8s-worker02 | Kubernetes Worker |

Runtime environment:

    kubeadm
    containerd
    Calico
    Ubuntu Server 22.04.5 LTS

---

### 3.3 Performance Testing Principles

All benchmarking should prioritize test environments.

Production environment benchmarking must meet:

    Has a change window
    Has business confirmation
    Has monitoring observation
    Has stop plan
    Has rollback plan
    Clear benchmarking scope
    Clear benchmarking duration
    Clear data cleanup method

Prohibited actions in production environments:

    Large-scale rados bench write
    Large-scale fio randwrite
    Large-scale object uploads
    Long-term full-load benchmarking
    Testing without isolated business Pools

---

## IV. Fundamental Principles of Ceph Performance Optimization

### 4.1 Identify Bottlenecks First, Then Optimize

Incorrect approach:

    Seeing slow performance directly adjusting parameters.
    Seeing slow ops directly increasing recovery.
    Seeing high latency directly restarting OSD.
    Seeing uneven capacity directly reweighting.
    Seeing slow CephFS directly restarting MDS.

Correct approach:

    First determine where the slowdown is.
    Then determine why it's slow.
    Then assess whether it can be optimized.
    Finally make parameter or architecture adjustments.

---

### 4.2 Common Categories of Performance Issues /think

</think>

> [!note] Common categories of performance issues:

| Type | Manifestation | Common Causes |
|---|---|---|
| High Latency | Single IO slow | Slow disk, slow network, slow OSD, slow MDS |
| Low Throughput | Slow large file read/write | Network bandwidth, disk bandwidth, insufficient client concurrency |
| Low IOPS | Poor small block random read/write | Weak HDD random IO, few OSDs, insufficient queue |
| Metadata Slow | ls, find, create slow | CephFS MDS pressure, large number of small files |
| Gateway Slow | S3 upload/download slow | Few RGW instances, LB, network, Bucket index |
| Mount Slow | PVC/RBD/CephFS mount slow | CSI, node, permissions, network, MDS |
| Recovery Slow | Long recovery/backfill time | Few OSDs, slow disk, rate-limiting parameters |
| Business Jitter | Delay fluctuates high/low | Recovery competing for resources, disk jitter, network jitter |

---

### 4.3 Check Four Fundamental Aspects First

Ceph performance issues should first check:

    1. Ceph cluster status
    2. OSD latency and capacity
    3. Node disk and network
    4. Business IO model

Corresponding commands:

    ceph -s
    ceph health detail
    ceph osd perf
    ceph osd df
    ceph pg stat
    iostat -x 1
    vmstat 1
    iftop
    top

---

### 4.4 Optimization Focus Varies by Storage Type

| Type | Focus |
|---|---|
| RBD | Block device latency, IOPS, filesystem, client kernel, RBD feature |
| CephFS | MDS, metadata, small files, large directories, multi-client concurrency |
| RGW | RGW instance count, HTTP entry, Bucket index, object size, concurrency |
| CSI | Kubernetes objects, CSI components, Node plugin, kubelet, network |

No single method can optimize all scenarios.

---

## Five. Pre-Operation Checks

### 5.1 Check Ceph Cluster Health

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree
    ceph osd df

Ideal state before benchmarking:

    HEALTH_OK
    PG active+clean
    All OSDs up/in
    No recovery/backfill
    No nearfull/full
    No slow ops

---

### 5.2 Check Service Status

    ceph orch ps
    ceph orch ls

If using CephFS:

    ceph fs status
    ceph mds stat

If using RGW:

    ceph orch ps --daemon_type rgw

---

### 5.3 Check Node Resources

Run on Ceph nodes:

    top
    free -h
    df -hT
    lsblk
    iostat -x 1
    vmstat 1

If no iostat:

    apt install -y sysstat

---

### 5.4 Check Network Status

    ip addr
    ip route
    ss -lntp

Simple test between nodes:

    ping -c 3 10.0.0.31
    ping -c 3 10.0.0.32
    ping -c 3 10.0.0.33

If installed iperf3, network testing is possible.

Ubuntu:

    apt install -y iperf3

Rocky Linux 9:

    dnf install -y iperf3

---

## Six. Experiment Task List

| Experiment | Target | Risk Level |
|---|---|---|
| Experiment 1 | Record performance test baseline | Low |
| Experiment 2 | rados bench test RADOS layer performance | Medium |
| Experiment 3 | fio test RBD block storage performance | Medium-High |
| Experiment 4 | CephFS file read/write test | Medium |
| Experiment 5 | RGW S3 upload/download test | Medium |
| Experiment 6 | OSD latency and disk bottleneck analysis | Medium |
| Experiment 7 | Network bandwidth test | Medium |
| Experiment 8 | Recovery/Backfill impact observation | Medium-High |
| Experiment 9 | RBD performance optimization verification | Medium |
| Experiment 10 | CephFS performance optimization verification | Medium |
| Experiment 11 | RGW performance optimization verification | Medium |
| Experiment 12 | Kubernetes CSI performance troubleshooting | Medium |
| Experiment 13 | Clean up benchmark data | High |

High-risk warning:

    Benchmarking consumes cluster resources.
    Write benchmarking writes real data.
    fio randwrite may cause significant pressure on disk and business.
    Confirm cleanup is for test data before rados bench cleanup.
    Do not run long write benchmarking in production environment.

---

## Seven. Experiment 1: Record Performance Test Baseline

### 7.1 Create Baseline Directory

    BASELINE_TIME=$(date +%F-%H%M%S)
    mkdir -p /backup/ceph/perf/${BASELINE_TIME}

---

### 7.2 Record Ceph Status

    ceph -s > /backup/ceph/perf/${BASELINE_TIME}/ceph-status.txt
    ceph health detail > /backup/ceph/perf/${BASELINE_TIME}/ceph-health-detail.txt
    ceph osd tree > /backup/ceph/perf/${BASELINE_TIME}/ceph-osd-tree.txt
    ceph osd df > /backup/ceph/perf/${BASELINE_TIME}/ceph-osd-df.txt
    ceph osd perf > /backup/ceph/perf/${BASELINE_TIME}/ceph-osd-perf.txt
    ceph pg stat > /backup/ceph/perf/${BASELINE_TIME}/ceph-pg-stat.txt
    ceph df > /backup/ceph/perf/${BASELINE_TIME}/ceph-df.txt
    ceph orch ps > /backup/ceph/perf/${BASELINE_TIME}/ceph-orch-ps.txt

---

### 7.3 Record Node Resources /think

Run the following commands on each Ceph node:

    hostname
    uptime
    free -h
    df -hT
    lsblk
    iostat -x 1 3
    vmstat 1 3

You can save the output to a file.

---

### 7.4 Why Record Baselines

Baselines are used for comparison:

    Whether the system is healthy before stress testing
    Whether anomalies occur during stress testing
    Whether the system recovers after stress testing
    Whether optimizations genuinely improve performance
    Whether post-failure analysis is possible

Without baselines, it's difficult to determine if optimizations are effective.

---

## VIII. Experiment 2: Testing RADOS Layer Performance with rados bench

### 8.1 Creating a Test Pool

Run the following on the Ceph management node:

    ceph osd pool create bench-pool 64
    ceph osd pool application enable bench-pool rados
    ceph osd pool set bench-pool size 3
    ceph osd pool set bench-pool min_size 2

Check:

    ceph osd pool ls
    ceph osd pool get bench-pool size
    ceph osd pool get bench-pool min_size

---

### 8.2 Write Stress Test

High-risk warning:

    rados bench will write real test objects.
    Do not run during production peak hours.
    Control time and concurrency carefully.

Run 60-second write test:

    rados bench -p bench-pool 60 write --no-cleanup

Notes:

    -p specifies the pool.
    60 indicates the test duration in seconds.
    write indicates write testing.
    --no-cleanup means objects are not automatically cleaned up after the test, facilitating subsequent read tests.

Observe the output:

- Bandwidth
- Average IOPS
- Latency
- Total time
- Total writes

---

### 8.3 Sequential Read Test

Run:

    rados bench -p bench-pool 60 seq

---

### 8.4 Random Read Test

Run:

    rados bench -p bench-pool 60 rand

---

### 8.5 Clean Up Test Objects

Clean up:

    rados bench -p bench-pool cleanup

Check objects:

    rados -p bench-pool ls | head

---

### 8.6 Monitoring Ceph During Testing

Open a new window to monitor:

    watch -n 3 'ceph -s'

Check OSD latency:

    watch -n 3 'ceph osd perf'

Check capacity:

    watch -n 5 'ceph osd df'

---

### 8.7 Understanding rados bench Results

rados bench tests RADOS layer performance and can help determine:

    Whether Ceph's underlying object storage capability is normal.
    Whether there are obvious bottlenecks in the OSD layer.
    Whether the network and disk can handle the load.
    Whether there are obvious distribution issues with the pool and PGs.

However, it cannot fully represent:

    RBD filesystem performance
    CephFS metadata performance
    RGW S3 performance
    Kubernetes PVC performance

---

## IX. Experiment 3: Testing RBD Block Storage Performance with fio

### 9.1 Creating a RBD Test Pool

If you already have a rbd-pool, you can reuse it.

Create a dedicated test pool:

    ceph osd pool create rbd-perf-pool 64
    ceph osd pool application enable rbd-perf-pool rbd
    ceph osd pool set rbd-perf-pool size 3
    ceph osd pool set rbd-perf-pool min_size 2
    rbd pool init rbd-perf-pool

---

### 9.2 Creating a RBD Image

Create a 20G test image:

    rbd create rbd-perf-image --size 20G -p rbd-perf-pool --image-feature layering

Check:

    rbd info rbd-perf-pool/rbd-perf-image

---

### 9.3 Map RBD

Run on the ceph-client:

    rbd map rbd-perf-pool/rbd-perf-image

Assuming output:

    /dev/rbd0

Check:

    rbd showmapped
    lsblk

---

### 9.4 Installing fio

Ubuntu:

    apt update
    apt install -y fio

Rocky Linux 9:

    dnf install -y fio

Verify:

    fio --version

---

### 9.5 Sequential Write Test

High-risk warning:

    fio directly writes to /dev/rbd0 and will overwrite the block device data.
    Only allowed for test images.
    Prohibited to directly execute on production RBDs.

Sequential write:

    fio --name=rbd-seq-write \
      --filename=/dev/rbd0 \
      --direct=1 \
      --ioengine=libaio \
      --rw=write \
      --bs=1M \
      --iodepth=16 \
      --numjobs=1 \
      --runtime=60 \
      --time_based \
      --group_reporting

Monitor:

- IOPS
- BW
- lat
- clat
- util

---

### 9.6 Sequential Read Test

    fio --name=rbd-seq-read \
      --filename=/dev/rbd0 \
      --direct=1 \
      --ioengine=libaio \
      --rw=read \
      --bs=1M \
      --iodepth=16 \
      --numjobs=1 \
      --runtime=60 \
      --time_based \
      --group_reporting

---

### 9.7 Random Write Test

fio --name=rbd-rand-write \
      --filename=/dev/rbd0 \
      --direct=1 \
      --ioengine=libaio \
      --rw=randwrite \
      --bs=4k \
      --iodepth=32 \
      --numjobs=1 \
      --runtime=60 \
      --time_based \
      --group_reporting

Notes:

    randwrite places heavy stress on HDD.
    Test duration and concurrency should not be too high.
    Exercise caution in production environments.

---

### 9.8 Random Read Test

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

### 9.9 Mixed Read/Write Test

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

Notes:

    rwmixread=70 indicates 70% read, 30% write.

---

### 9.10 Observing During Test

Ceph Side:

    ceph -s
    ceph osd perf
    ceph osd df
    ceph pg stat

Node Side:

    iostat -x 1
    vmstat 1
    top
    iftop

---

### 9.11 Clean Up RBD Test Resources

unmap:

    rbd unmap /dev/rbd0

Delete Image:

    rbd rm rbd-perf-pool/rbd-perf-image

Delete Pool:

    ceph config set mon mon_allow_pool_delete true
    ceph osd pool rm rbd-perf-pool rbd-perf-pool --yes-i-really-really-mean-it
    ceph config set mon mon_allow_pool_delete false

---

## TenI don't know.Experiment Four: CephFS File Read/Write Test

### 10.1 Prerequisites

CephFS has been created and mounted to the client:

    /mnt/cephfs-kernel

Check:

    df -hT | grep ceph

---

### 10.2 Large File Write Test

Create test directory:

    mkdir -p /mnt/cephfs-kernel/perf-demo

Write 1G file:

    dd if=/dev/zero of=/mnt/cephfs-kernel/perf-demo/test-1g.bin bs=1M count=1024 oflag=direct

Record speed.

---

### 10.3 Large File Read Test

Clearing cache is only suitable for test environments.

Exercise caution:

    sync
    echo 3 > /proc/sys/vm/drop_caches

Read:

    dd if=/mnt/cephfs-kernel/perf-demo/test-1g.bin of=/dev/null bs=1M iflag=direct

---

### 10.4 Small File Creation Test

Create 1000 small files:

    mkdir -p /mnt/cephfs-kernel/perf-demo/small-files

    for i in $(seq 1 1000); do
      echo "file-$i" > /mnt/cephfs-kernel/perf-demo/small-files/file-$i.txt
    done

Record time:

    time sh -c 'for i in $(seq 1 1000); do echo "file-$i" > /mnt/cephfs-kernel/perf-demo/small-files/file-$i.txt; done'

---

### 10.5 Directory Traversal Test

    time ls -l /mnt/cephfs-kernel/perf-demo/small-files > /dev/null

    time find /mnt/cephfs-kernel/perf-demo/small-files -type f > /dev/null

---

### 10.6 Observing During CephFS Test

CephFS Status:

    ceph fs status
    ceph mds stat

Ceph Status:

    ceph -s
    ceph osd perf

MDS Processes:

    ceph orch ps --daemon_type mds

Node Resources:

    top
    iostat -x 1
    vmstat 1

---

### 10.7 CephFS Performance Focus Points

CephFS performance is not only about OSD.

Also check:

- Whether MDS is under high load
- Whether metadata pool is healthy
- Number of small files
- Number of directory entries
- Number of clients
- Whether there are slow requests
- Whether there are many metadata operations

---

### 10.8 Clean Up Test Data

    rm -rf /mnt/cephfs-kernel/perf-demo

Warning:

    Confirm the path is correct.
    Do not accidentally delete business directories.

---

## ElevenI don't know.Experiment Five: RGW S3 Upload/Download Test

### 11.1 Prerequisites

RGW has been deployed, and AWS CLI has been configured.

Endpoint Example:

    export RGW_ENDPOINT="http://10.0.0.31:7480"

Or:

    export RGW_ENDPOINT="https://rgw.example.com"

---

### 11.2 Create Test Bucket

aws --profile ceph-rgw \
  --endpoint-url ${RGW_ENDPOINT} \
  s3 mb s3://rgw-perf-demo

---

### 11.3 Creating Test Files

Create a 100M file:

    dd if=/dev/zero of=/tmp/rgw-100m.bin bs=1M count=100

Create a 1G file:

    dd if=/dev/zero of=/tmp/rgw-1g.bin bs=1M count=1024

---

### 11.4 Uploading Tests

Upload 100M:

    time aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-100m.bin s3://rgw-perf-demo/rgw-100m.bin

Upload 1G:

    time aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-1g.bin s3://rgw-perf-demo/rgw-1g.bin

---

### 11.5 Downloading Tests

    time aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp s3://rgw-perf-demo/rgw-100m.bin /tmp/rgw-100m-download.bin

---

### 11.6 Multi-Object Upload Test

Create small files:

    mkdir -p /tmp/rgw-small-files

    for i in $(seq 1 1000); do
      echo "object-$i" > /tmp/rgw-small-files/object-$i.txt
    done

Upload:

    time aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 cp /tmp/rgw-small-files s3://rgw-perf-demo/small-files/ --recursive

---

### 11.7 Observing During RGW Tests

RGW service:

    ceph orch ps --daemon_type rgw

Ceph status:

    ceph -s
    ceph osd perf
    ceph df

Bucket statistics:

    radosgw-admin bucket stats --bucket=rgw-perf-demo

If using Nginx:

    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log

---

### 11.8 Cleaning Up RGW Test Data

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rm s3://rgw-perf-demo --recursive

    aws --profile ceph-rgw \
      --endpoint-url ${RGW_ENDPOINT} \
      s3 rb s3://rgw-perf-demo

Clean up local files:

    rm -f /tmp/rgw-100m.bin /tmp/rgw-1g.bin /tmp/rgw-100m-download.bin
    rm -rf /tmp/rgw-small-files

---

## TwelveI don't know.Experiment Six: OSD Latency and Disk Bottleneck Analysis

### 12.1 Checking OSD Latency

    ceph osd perf

Example:

    osd  commit_latency(ms)  apply_latency(ms)
      0                  1                  1
      1                  2                  2
      2                 80                 80

If an OSD shows significantly higher values than others, focus on troubleshooting the node hosting that OSD.

---

### 12.2 Locating OSD Node and Disk

    ceph osd metadata osd.2

Key fields:

- hostname
- bluestore_bdev_dev_node
- devices

---

### 12.3 Checking Disk on Node

Log in to the corresponding node:

    ssh root@<hostname>

Execute:

    lsblk
    iostat -x 1
    dmesg | grep -i error
    journalctl -k | grep -i error

Focus on iostat:

| Field | Description |
|---|---|
| r/s | Reads per second |
| w/s | Writes per second |
| rkB/s | Read throughput |
| wkB/s | Write throughput |
| await | Average wait time |
| %util | Disk utilization |

---

### 12.4 Identifying Disk Bottlenecks

Common manifestations:

    await remains very high
    %util remains close to 100%
    dmesg shows I/O errors
    Single OSD latency is significantly high
    Business IO fluctuation is severe

Handling directions:

- Confirm if recovery/backfill is ongoing
- Check disk health
- Check host resources
- Check for other processes competing for IO
- Replace disk if necessary
- For long-term optimization, consider SSD/NVMe or increasing OSD count

---

## ThirteenI don't know.Experiment Seven: Network Bandwidth Testing

### 13.1 Using iperf3 for Testing

Start server on ceph-node01:

    iperf3 -s

Run client on ceph-node02:

    iperf3 -c 10.0.0.31

Reverse test:

    iperf3 -c 10.0.0.31 -R

Multi-threaded test:

    iperf3 -c 10.0.0.31 -P 4

---

### 13.2 Focus Points for Network Testing

Focus on:

- Whether bandwidth meets expectations
- Whether there is noticeable jitter
- Whether there are retransmissions
- Whether bidirectional bandwidth is consistent
- Whether multi-threading significantly improves performance

---

### 13.3 Ceph Network Bottleneck Indications

May manifest as:

- OSD Heartbeat Issues
- PG Stale
- Slow Ops
- Recovery is Slow
- RBD / CephFS High Latency
- RGW Upload/Download Slow
- OSD-to-OSD Copy Slow

---

### 13.4 Production Network Recommendations

Production environment recommendations:

- Separate Ceph Public Network and Cluster Network, depending on scale and budget
- Avoid using OSD replication network for heavy business traffic
- Use stable low-latency network
- Monitor switches, uplinks, MTU, packet loss, and retransmission
- Avoid severely oversubscribed virtualization networks
- Monitor network card error packets and packet loss

---

## FourteenI don't know.Experiment Eight: Recovery / Backfill Impact Observation

### 14.1 Observing Recovery Status

When the cluster performs OSD expansion, replacement, or failure recovery, check:

    ceph -s

May see:

    recovering
    backfilling
    remapped
    misplaced

---

### 14.2 Observing OSD Latency During Recovery

    watch -n 3 'ceph osd perf'

---

### 14.3 Observing Node IO

Execute on OSD node:

    iostat -x 1
    vmstat 1
    top

---

### 14.4 Checking Recovery-Related Parameters

    ceph config get osd osd_max_backfills
    ceph config get osd osd_recovery_max_active
    ceph config get osd osd_recovery_op_priority

Notes:

    Parameters may differ across versions.
    Refer to the official documentation of the current Ceph version.

---

### 14.5 Example of Adjusting Recovery Parameters

High-risk warning:

    Do not adjust arbitrarily in production environments.
    Record original values before adjustment.
    Observe business latency and recovery speed after adjustment.

Record original values:

    ceph config get osd osd_max_backfills
    ceph config get osd osd_recovery_max_active

Reduce backfill concurrency:

    ceph config set osd osd_max_backfills 1

Restore default or revert to original values:

    ceph config rm osd osd_max_backfills

Or set to original value:

    ceph config set osd osd_max_backfills <old-value>

---

### 14.6 Balancing Recovery Speed and Business Performance

Principles:

    Faster recovery may more significantly impact business IO.
    Slower recovery leads to longer data degradation time.
    Balance based on business importance, failure risk, and load conditions.

---

## FifteenI don't know.Experiment Nine: RBD Performance Optimization Directions

### 15.1 RBD Performance Influencing Factors

RBD performance is affected by the following factors:

- Number of OSDs
- OSD disk type
- Network bandwidth and latency
- Pool replica count
- PG distribution
- RBD image feature
- Client kernel version
- File system type
- fio parameters
- Business IO model
- Recovery / Backfill
- Whether HDD / SSD / NVMe is used

---

### 15.2 Basic RBD Checks

    rbd info <pool>/<image>
    rbd status <pool>/<image>
    ceph osd perf
    ceph osd df
    ceph pg stat

Client:

    lsblk
    mount
    iostat -x 1
    dmesg | tail -100

---

### 15.3 File System Selection

Common choices:

| File System | Features |
|---|---|
| XFS | Commonly used for large files, databases, and block devices |
| ext4 | Good general-purpose performance, simple and stable |

In production, combine with business testing rather than relying solely on experience.

---

### 15.4 RBD Feature Compatibility

In experiments, use conservative features:

    --image-feature layering

Advanced features like:

- exclusive-lock
- object-map
- fast-diff
- deep-flatten

May provide specific capabilities but also require client compatibility consideration.

Principles:

    Do not enable features without compatibility confirmation.
    In Kubernetes CSI scenarios, follow CSI official recommendations.
    Test before upgrading kernel, ceph-common, and ceph-csi.

---

### 15.5 RBD Optimization Recommendations

- Use sufficient number of OSDs
- Avoid single OSD or single-node bottlenecks
- Use SSD / NVMe for critical business
- Avoid cluster nearing full capacity
- Avoid recovery/backfill during business peak hours
- Use appropriate file systems
- Use suitable IO queue depth
- Optimize application layer for databases
- Monitor Pool capacity and latency for RBD
- Clearly define StorageClass and reclaimPolicy for Kubernetes PVC

---

## SixteenI don't know.Experiment Ten: CephFS Performance Optimization Directions

### 16.1 CephFS Performance Influencing Factors

CephFS is additionally affected by MDS.

Main factors:

- MDS CPU / Memory
- MDS active/standby status
- Metadata pool performance
- Data pool performance
- Number of small files
- Number of files per directory
- Number of clients
- File locks and metadata operations
- OSD latency
- Network latency

---

### 16.2 CephFS Check Commands

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ceph -s
    ceph osd perf

Client:

    mount | grep ceph
    df -hT
    dmesg | tail -100

---

### 16.3 Common CephFS Performance Issues

| Issue | Possible Causes |
|---|---|
| Slow ls | Large directory, MDS pressure |
| Slow file creation | Too many small files, slow metadata pool |
| Slow read/write | Slow OSD, slow network |
| Slow with multiple clients | MDS load, lock contention |
| Slow mount | MDS, permissions, network, client compatibility |

---

### 16.4 CephFS Optimization Recommendations

- Production must have at least one standby MDS
- Small file business requires pre-stress testing
- Avoid placing massive files in a single directory
- Use reliable and high-performance OSDs for metadata pool
- Monitor MDS CPU/memory for critical business
- Stress test for multi-client concurrent scenarios
- Directory structure should be reasonably layered
- Do not use CephFS as the primary database storage
- Databases are better suited for RBD

---

## SeventeenI don't know.Experiment Eleven: RGW Performance Optimization Directions

### 17.1 RGW Performance Impact Factors

RGW performance is affected by the following factors:

- Number of RGW instances
- Frontend Nginx / LB
- Location of HTTPS termination
- Bucket index
- Object size
- Number of objects
- Concurrent request count
- RGW Pool status
- OSD latency
- Network bandwidth
- Client SDK configuration
- Whether through public internet or cross-region

---

### 17.2 RGW Check Commands

    ceph orch ps --daemon_type rgw
    ceph -s
    ceph osd perf
    ceph df
    radosgw-admin bucket stats --bucket=<bucket-name>
    radosgw-admin user info --uid=<uid>

Testing:

    curl -I http://10.0.0.31:7480

AWS CLI:

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls

---

### 17.3 RGW Common Performance Issues

| Issue | Possible Causes |
|---|---|
| Slow upload | Few RGW instances, slow network, slow OSD |
| Slow download | Insufficient bandwidth, proxy limitations, slow OSD |
| Slow listing Bucket | Large number of objects, Bucket index pressure |
| Slow small objects | High request count, metadata pressure |
| Slow after proxy | Nginx buffering, body size, TLS overhead |
| Slow authentication or failure | Time synchronization, signature, Host header |

---

### 17.4 RGW Optimization Recommendations

- Deploy at least multiple RGW instances
- Use Nginx / HAProxy / LB in front
- Use HTTPS externally, choose HTTP/HTTPS internally based on security requirements
- Evaluate large and small objects separately
- Control object count and lifecycle in Bucket
- Use quotas to manage users and Buckets
- Monitor 4xx/5xx/errors/delay/throughput
- Do not let single RGW become a bottleneck
- Configure reverse proxy buffering and body size appropriately
- Production object storage requires capacity and lifecycle management

---

## EighteenI don't know.Experiment Twelve: Kubernetes CSI Performance Troubleshooting

### 18.1 RBD CSI Performance Troubleshooting

Kubernetes side:

    kubectl get pvc -A
    kubectl get pv
    kubectl get pods -n ceph-csi -o wide
    kubectl describe pod -n <namespace> <pod-name>
    kubectl get volumeattachment

Ceph side:

    rbd ls -p k8s-rbd
    rbd info k8s-rbd/<image-name>
    rbd status k8s-rbd/<image-name>
    ceph osd perf
    ceph -s

Node side:

    modprobe rbd
    lsmod | grep rbd
    iostat -x 1
    dmesg | tail -100

---

### 18.2 CephFS CSI Performance Troubleshooting

Kubernetes side:

    kubectl get pvc -A
    kubectl get pods -n ceph-csi -o wide
    kubectl describe pod -n <namespace> <pod-name>

CephFS side:

    ceph fs status
    ceph mds stat
    ceph fs subvolume ls cephfs --group_name csi
    ceph osd perf

Node side:

    modprobe ceph
    lsmod | grep ceph
    dmesg | tail -100

---

### 18.3 Common Causes of CSI Performance Issues

| Issue | Possible Causes |
|---|---|
| Slow Pod startup | Slow PVC mounting, slow image pulling, slow CSI Node |
| Slow PVC creation | Slow CSI Controller, Ceph permissions or Pool issues |
| Slow RBD IO | Slow OSD, slow network, RBD features, node issues |
| Slow CephFS IO | Slow MDS, many small files, slow OSD |
| FailedMount | Secret, network, permissions, driver, node module |

---

## NineteenI don't know.Performance Data Recording Template

### 19.1 Basic Information

| Item | Content |
|---|---|
| Test Date | YYYY-MM-DD |
| Tester |  |
| Ceph Version |  |
| Node Count |  |
| OSD Count |  |
| Disk Type | HDD / SSD / NVMe |
| Network Type | 1G / 10G / 25G |
| Pool |  |
| Replication Count |  |
| PG Count |  |
| Test Type | RADOS / RBD / CephFS / RGW |

---

### 19.2 Pre-Test Status

| Check Item | Result |
|---|---|
| ceph -s |  |
| ceph health detail |  |
| ceph osd perf |  |
| ceph osd df |  |
| ceph pg stat |  |
| Recovery/backfill | Yes / No |
| Node Load |  |
| Network Status |  |

---

### 19.3 Test Results

| Test Item | Parameter | Result |
|---|---|---|
| rados write | 60s |  |
| rados seq | 60s |  |
| rados rand | 60s |  |
| fio seq write | bs=1M iodepth=16 |  |
| fio seq read | bs=1M iodepth=16 |  |
| fio randwrite | bs=4k iodepth=32 |  |
| fio randread | bs=4k iodepth=32 |  |
| CephFS write | dd 1G |  |
| RGW upload | 1G |  |

---

### 19.4 Post-Test Status

| Check Item | Result |
|---|---|
| ceph -s |  |
| ceph health detail |  |
| ceph osd perf |  |
| ceph osd df |  |
| ceph pg stat |  |
| Are there slow ops |  |
| Are there abnormal alerts |  |

---

## 20. Production Environment Performance Optimization Notes

1. Do not perform write benchmarking during production peak hours.
2. Do not execute fio write tests directly on business RBD.
3. Do not make large-scale recovery parameter adjustments without understanding the impact.
4. Do not rely solely on single benchmark results; analyze trends and multiple rounds of data.
5. Do not equate rados bench results with business performance.
6. Do not directly apply RBD performance conclusions to CephFS.
7. Do not use CephFS as the primary storage for databases.
8. Do not let a single RGW instance carry the production object storage entry point.
9. Do not delay performance optimization until capacity approaches full.
10. Do not ignore OSD usage imbalance.
11. Do not ignore disk await and %util.
12. Do not ignore network bandwidth, packet loss, and jitter.
13. Do not long-term disable scrub / deep scrub to gain short-term performance.
14. Do not let recovery parameters kill business IO for fast recovery.
15. All parameter adjustments must record original values and rollback methods.

---

## 21. Advanced SRE Performance Optimization Methodology

### 21.1 Performance Optimization is Not Tuning, It's Bottleneck Localization

Advanced SRE sequence:

    1. Clarify the manifestation of business slowness.
    2. Confirm whether the slowness is RBD, CephFS, RGW, or CSI.
    3. Check Ceph cluster health.
    4. Check OSD latency and capacity.
    5. Check node disk and network.
    6. Check business IO model.
    7. Perform small-scale benchmarking verification.
    8. Adjust item by item.
    9. Compare data before and after optimization.
    10. Establish baseline and documentation.

---

### 21.2 Distinguish Throughput and IOPS

For large file sequential read/write focus:

    Throughput
    MB/s
    Network bandwidth
    Disk sequential performance

For small block random read/write focus:

    IOPS
    Latency
    Disk random performance
    Queue depth
    OSD count

Database scenarios typically focus more on:

    Small block random IO
    Latency stability
    fsync
    WAL / redo log
    Application consistency

Object storage large files focus more on:

    Throughput
    Concurrency
    RGW instances
    Frontend entry point
    Network bandwidth

---

### 21.3 Performance Optimization Should Be Layered

Recommended layers:

    Business Layer:
      Application IO model, concurrency, object size, file count

    Access Layer:
      RBD / CephFS / RGW / CSI / Nginx / LB

    Ceph Service Layer:
      MON / MGR / MDS / RGW / OSD

    Data Distribution Layer:
      Pool / PG / CRUSH / Replication count / Failure domain

    Node Resource Layer:
      CPU / Memory / Disk / Network / Time synchronization

    Hardware Architecture Layer:
      HDD / SSD / NVMe / 10G network / Dedicated storage network

---

### 21.4 Must Have Rollback Plan Before Optimization

Any parameter adjustment must record:

    Original parameter value
    Modification time
    Modification reason
    Expected effect
    Verification method
    Rollback command

Example:

    ceph config get osd osd_max_backfills

Adjustment:

    ceph config set osd osd_max_backfills 1

Rollback:

    ceph config rm osd osd_max_backfills

Or:

    ceph config set osd osd_max_backfills <old-value>

---

## 22. Interview Answer Structure

If asked in an interview:

    How would you troubleshoot slow Ceph performance?

You can answer:

    I would first clarify which type of business is slow: RBD block storage, CephFS file storage, RGW object storage, or Kubernetes CSI-mounted PVC. Different entry points have different bottlenecks, and you can't directly apply the same method.
    First, I would check ceph -s and ceph health detail to confirm if there are OSD down, PG degraded, nearfull, slow ops, recovery/backfill issues. Then check ceph osd perf and ceph osd df to confirm if any OSD has significantly high latency, uneven capacity, or is nearly full.
    Second, I would locate to the node, using iostat -x 1, vmstat, top, dmesg, journalctl to check disk await, util, system load, and disk errors. If OSD latency is high, I would typically confirm if it's due to disk failure, IO saturation, network issues, or recovery resource contention.
    Third, I would analyze the business type. For RBD, I would check rbd info, rbd status, client kernel module, filesystem, and fio testing. For CephFS, I would focus on ceph fs status, MDS status, small file count, and directory structure. For RGW, I would check RGW instances, Nginx/LB, Bucket index, object size, and concurrency. For Kubernetes, I would check PVC/PV, CSI Pod, VolumeAttachment, and kubelet events.
    I would not directly adjust parameters first. Before tuning, I would record the baseline, clarify benchmarking methods, and validate in a small scope. For example, use rados bench to check RADOS layer, fio to check RBD layer, and S3 tools to test RGW layer. When adjusting recovery/backfill parameters, I must record original values, observe business impact, and prepare rollback.

---

## 23. Summary of This Article

This article mainly organizes Ceph performance optimization and benchmarking methods:

1. The premise of Ceph performance optimization is to first identify the bottleneck.  
2. Performance issues can be categorized into latency, throughput, IOPS, metadata, gateway, and mount issues.  
3. `rados bench` can test the performance of the RADOS layer.  
4. `fio` can test the performance of RBD block devices.  
5. CephFS performance requires special attention to MDS and metadata.  
6. RGW performance requires attention to RGW instances, ingress layer, Bucket index, and object size.  
7. Kubernetes CSI performance requires simultaneous monitoring of K8s objects, CSI Pods, nodes, and Ceph backend.  
8. `ceph osd perf` is the key entry point for observing OSD latency.  
9. `iostat -x 1` is an important command for identifying disk bottlenecks.  
10. `iperf3` can be used to validate network bandwidth between nodes.  
11. Recovery / Backfill will affect business IO, requiring a balance between recovery speed and business performance.  
12. Optimization directions for RBD, CephFS, and RGW differ, and conclusions cannot be mixed.  
13. Production environment stress testing must include windowing, monitoring, stoppage plan, and rollback plan.  
14. Original values must be recorded before tuning parameters.  
15. Advanced SREs should form a closed-loop optimization process including baseline, stress testing, observation, adjustment, and post-mortem analysis.  

---

## Twenty-Four, Reference Documents  

Ceph Performance and Benchmarking:  

    https://docs.ceph.com/en/latest/rados/operations/benchmark/  

Ceph RADOS Operations:  

    https://docs.ceph.com/en/latest/rados/  

Ceph OSD Configuration Reference:  

    https://docs.ceph.com/en/latest/rados/configuration/osd-config-ref/  

Ceph Pool Management:  

    https://docs.ceph.com/en/latest/rados/operations/pools/  

Ceph Placement Groups:  

    https://docs.ceph.com/en/latest/rados/operations/placement-groups/  

Ceph RBD Documentation:  

    https://docs.ceph.com/en/latest/rbd/  

CephFS Management Documentation:  

    https://docs.ceph.com/en/latest/cephfs/administration/  

Ceph RGW Management Documentation:  

    https://docs.ceph.com/en/latest/radosgw/admin/  

Ceph Health Check:  

    https://docs.ceph.com/en/latest/rados/operations/health-checks/  

fio Official Documentation:  

    https://fio.readthedocs.io/en/latest/fio_doc.html  

iperf3 Official Documentation:  

    https://iperf.fr/