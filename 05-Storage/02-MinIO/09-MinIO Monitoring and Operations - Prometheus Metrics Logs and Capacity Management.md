# MinIO Monitoring and Operations: Prometheus Metrics, Logs, and Capacity Management

Recommended path: 05-Storage/02-MinIO/09-MinIO Monitoring and Operations: Prometheus Metrics, Logs, and Capacity Management.md

Tags: #MinIO #MonitorTraffic #Prometheus #Grafana #Alertmanager #ObjectStorage #S3 #CapacityManagement #LogAnalysis #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the ninth article of the MinIO module, focusing on learning MinIO monitoring, logs, capacity management, and daily operations.

Previously completed:

- MinIO object storage basics
- Single machine single disk deployment
- Single node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS entry design
- Nginx HTTPS unified entry
- mc client configuration and object operations
- Users, Policies, AccessKey, and SecretKey permission management
- Erasure Coding, node failure, and disk failure recovery

This article begins the core of production operations:

    How to determine if a MinIO cluster is healthy?
    What metrics to check during daily inspections?
    How to view MinIO node and disk status?
    How to view Bucket capacity?
    How to view object count?
    How to collect MinIO metrics via Prometheus?
    What dashboards should Grafana display?
    What alerts should Alertmanager trigger?
    How do Nginx logs assist in troubleshooting?
    How do Docker logs assist in troubleshooting?
    How to plan capacity water levels?
    How to detect abnormal growth in object storage?
    How to establish an advanced SRE perspective MinIO operations closed-loop?

This article emphasizes practical operations, with commands provided for all checks.

---

## II. Learning Objectives

After completing this article, you should be able:

1. Use `mc admin info` to view MinIO cluster status.
2. Use the health interface to check MinIO live/ready status.
3. Use `mc du` to view Bucket or Prefix capacity.
4. Use `mc find` to view object distribution.
5. Use `Docker logs` to view MinIO service logs.
6. Use Nginx access.log/error.log to troubleshoot entry issues.
7. Understand MinIO Prometheus metric collection methods.
8. Be able to generate or configure Prometheus to scrape MinIO metrics.
9. Be able to plan MinIO Grafana monitoring dashboards.
10. Be able to design alerts for nodes, disks, capacity, API, certificates, and backups.
11. Be able to create a MinIO daily inspection checklist.
12. Be able to detect abnormal capacity growth in Buckets.
13. Be able to judge if there are high-risk operations in object storage.
14. Be able to establish a MinIO production operations baseline.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues with the previous 4-node distributed MinIO cluster:

| IP | Hostname | Role | Port |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | 9000 / 9001 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | 9000 / 9001 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | 9000 / 9001 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | 9000 / 9001 |
| 10.0.0.45 | minio-client | mc Client / Operations Management Node | - |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry | 80 / 443 |

---

### 3.2 Access Entry

Backend direct access entry:

    http://10.0.0.41:9000

HTTPS unified entry:

    https://s3.minio.local

Console entry:

    https://console.minio.local

This article defaults to using:

    https://s3.minio.local

If the experimental environment uses self-signed certificates, temporarily use:

    --insecure

in mc commands. Do not use --insecure long-term in production environments.

---

### 3.3 Image Version

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z

---

## IV. MinIO Monitoring and Operations Focus Areas

MinIO production operations mainly focus on eight types of objects:

    1. Cluster status
    2. Node status
    3. Disk status
    4. Bucket capacity
    5. Object count
    6. S3 API requests
    7. Entry proxy status
    8. Security and permission risks

---

### 4.1 Cluster Status

Need to monitor:

    Whether MinIO is accessible.
    Whether the cluster is ready.
    Whether there are offline nodes.
    Whether there are offline drives.
    Whether there is healing.
    Whether there are version inconsistencies.
    Whether there are abnormal disk capacities.
    Whether there are network anomalies between nodes.

---

### 4.2 Node Status

Need to monitor:

    Whether nodes are online.
    Whether Docker containers are running.
    Whether CPU is too high.
    Whether memory is abnormal.
    Whether there are packet losses in the network.
    Whether the system disk is full.
    Whether data disks are mounted.
    Whether time is synchronized.

---

### 4.3 Disk Status

Need to monitor:

    Whether disks are online.
    Whether disks are read-only.
    Whether disk capacity is too high.
    Whether there are I/O errors on disks.
    Whether mount points are lost.
    Whether data directories are mistakenly placed on the system disk.
    Whether multiple disk directories are actually on the same physical disk.

---

### 4.4 Bucket and Object Status

Need to monitor:

    Bucket count.
    Bucket capacity.
    Object count.
    Prefix capacity.
    Abnormal growth in individual Buckets.
    Growth of a large number of small objects.
    Failed uploads of large objects.
    Whether there are abnormal deletions.
    Whether there are long-term unassigned Buckets.

---

### 4.5 S3 API Status

Need to monitor:

Request Volume.
4xx Errors.
5xx Errors.
AccessDenied.
SignatureDoesNotMatch.
Large File Upload Failure.
API Latency.
Upload/Download Throughput.
Client Source IP.
Abnormal High-Frequency Access.

---

### 4.6 Entrance Proxy Status

Need to monitor:

    Whether Nginx is running.
    Whether HTTPS certificate is valid.
    Whether upstream is available.
    Whether 502 / 504 errors have increased.
    Whether 413 has occurred.
    Whether client_max_body_size is reasonable.
    Whether proxy_read_timeout is reasonable.
    Whether API and Console are mixed usage.

---

## Five. Preparing mc Operations Environment

### 5.1 Creating Configuration Directory

Execute on minio-client or operations management node:

    mkdir -p /data/minio/mc-config
    mkdir -p /tmp/minio-monitor-demo

---

### 5.2 Configuring Administrator alias

If using self-signed certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

If using official trusted certificate:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      alias set minio-admin https://s3.minio.local minioadmin 'MinioAdmin@123456'

---

### 5.3 Viewing alias

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

---

### 5.4 Creating Temporary Command Function

To reduce command length, temporarily define:

    export MC_IMAGE=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z
    export MC_CONFIG=/data/minio/mc-config
    export MC_WORKDIR=/tmp/minio-monitor-demo

    mcx() {
      docker run --rm \
        -v ${MC_CONFIG}:/root/.mc \
        -v ${MC_WORKDIR}:/demo \
        ${MC_IMAGE} "$@"
    }

Self-signed certificate environment example:

    mcx --insecure admin info minio-admin

Note:

    This function is only valid in the current shell session.
    Official notes retain complete docker run commands for easy copying and reuse.

---

## Six. Basic Health Checks

### 6.1 Viewing MinIO Cluster Information

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Focus on:

    Whether the cluster is accessible.
    Whether nodes are online.
    Whether disks are online.
    Whether offline drives exist.
    Whether healing is occurring.
    Capacity usage.
    Whether warnings exist.
    Whether versions are consistent.

---

### 6.2 Checking live Interface

Execute on minio-client or minio-entry:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check live $ip"
      curl -I http://$ip:9000/minio/health/live
    done

Note:

    live indicates whether MinIO process is alive.
    live being normal does not necessarily mean the entire cluster is writable.
    Also need to check ready and admin info.

---

### 6.3 Checking ready Interface

Execute:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check ready $ip"
      curl -I http://$ip:9000/minio/health/ready
    done

Note:

    ready is more suitable for determining whether MinIO is ready to provide service.
    Production health checks recommend prioritizing ready.
    If ready is abnormal, need to combine with admin info and logs for troubleshooting.

---

### 6.4 Health Check via HTTPS Entry

If configured with Nginx HTTPS unified entry:

    curl -k -I https://s3.minio.local/minio/health/live
    curl -k -I https://s3.minio.local/minio/health/ready

If using official trusted certificate, remove -k:

    curl -I https://s3.minio.local/minio/health/live
    curl -I https://s3.minio.local/minio/health/ready

---

### 6.5 Console Entry Check

Browser access:

    https://console.minio.local

Or command check:

    curl -k -I https://console.minio.local

Note:

    Console being normal does not necessarily mean S3 API is normal.
    S3 API being normal does not necessarily mean Console is normal.
    Both have different ports, entries, and usage objects.

---

## Seven. Node and Container Operations Checks

### 7.1 Checking Docker Container Status

Execute on each of the 4 MinIO nodes:

    docker ps | grep minio

If not running: /think

docker ps -a | grep minio

Check status:

    docker inspect minio --format '{{.State.Status}}'
    docker inspect minio --format '{{.State.StartedAt}}'
    docker inspect minio --format '{{.RestartCount}}'

---

### 7.2 View MinIO Container Logs

Execute on each node:

    docker logs --tail=100 minio

Real-time view:

    docker logs -f minio

Filter by time:

    docker logs --since 30m minio
    docker logs --since 2h minio

Common logs to monitor:

    drive offline
    healing
    access denied
    disk full
    network error
    unable to connect
    quorum
    read failed
    write failed
    certificate
    permission denied

---

### 7.3 Check Container Resource Usage

Execute on each node:

    docker stats minio

Monitor:

    CPU usage.
    Memory usage.
    Network I/O.
    Block I/O.
    Whether it shows abnormal continuous increase.

---

### 7.4 Check Docker Service

    systemctl status docker
    journalctl -u docker --since "1 hour ago"

In production, if Docker service fails, MinIO container may exit or fail to restart.

---

## VIII. System and Disk Checks

### 8.1 Check System Resources

Execute on each MinIO node:

    uptime
    free -h
    top
    vmstat 1 5

Focus on:

    Whether load average remains excessively high.
    Whether memory is insufficient.
    Whether swap is heavily used.
    Whether CPU is occupied by other processes.
    Whether system-level lag exists.

---

### 8.2 Check Disk Mounts

    df -hT
    lsblk
    mount | grep minio

Focus on:

    Whether /data/minio/disk1 is mounted.
    Whether /data/minio/disk2 is mounted.
    Whether data directory is mistakenly written to system disk.
    Whether file system type matches the plan.
    Whether disk capacity approaches threshold.

---

### 8.3 Check Disk Directory Capacity

Execute on each node:

    du -sh /data/minio/disk1
    du -sh /data/minio/disk2

Batch execution:

    for d in /data/minio/disk1 /data/minio/disk2; do
      echo "check $d"
      du -sh $d
    done

---

### 8.4 Check Disk I/O

If sysstat is installed:

    iostat -x 1 5

If not installed:

    apt update
    apt install -y sysstat

Monitor:

    Whether %util remains close to 100% long-term.
    Whether await significantly increases.
    Whether r/s, w/s are abnormal.
    Whether disk has obvious hotspots.

---

### 8.5 Check Kernel Logs

    dmesg | tail -100
    journalctl -k --since "1 hour ago"

Monitor:

    I/O error
    read-only file system
    ext4 error
    xfs error
    reset
    timeout
    disk
    nvme
    scsi

If I/O error or read-only file system occurs, disk issues should be prioritized.

---

## IX. Bucket Capacity Management

### 9.1 View All Buckets

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls minio-admin

---

### 9.2 View Specific Bucket Capacity

Example:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-admin/app-uploads

If the bucket is created in previous experiments:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-admin/heal-demo

---

### 9.3 View Prefix Capacity

Example:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-admin/app-uploads/logs/

View business prefix:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-admin/policy-demo/app01/

---

### 9.4 Find Bucket Objects

Recursively list objects: /think

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/app-uploads

Search by filename:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/app-uploads --name "*.log"

Search by object suffix:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/app-uploads --name "*.zip"

---

### 9.5 Recording Bucket Capacity Inspection

You can save the capacity output to an inspection file:

mkdir -p /tmp/minio-monitor-demo/reports

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  -v /tmp/minio-monitor-demo:/demo \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure du minio-admin/app-uploads > /tmp/minio-monitor-demo/reports/app-uploads-du.txt

View the file:

cat /tmp/minio-monitor-demo/reports/app-uploads-du.txt

---

### 9.6 Capacity Governance Principles

In production environments, pay attention to:

- Capacity of each Bucket.
- Number of objects in each Bucket.
- Growth rate of each business.
- Presence of unusually large objects.
- Presence of numerous small objects.
- Presence of unassigned Buckets.
- Whether lifecycle cleanup is needed.
- Whether quota limits are needed.
- Whether cold data migration is needed.
- Whether cross-cluster backup is needed.

Object storage capacity isn't just about df -h.

You should also check:

- Total cluster capacity.
- Single node capacity.
- Single disk capacity.
- Bucket capacity.
- Prefix capacity.
- Business growth trends.

---

## Ten, Object Count and Small Object Risks

### 10.1 Why Small Objects Need Attention

A large number of small objects may cause:

- Metadata management pressure.
- Slower object listing.
- Slower backup migration.
- Slower mirror synchronization.
- Slower deletion operations.
- Difficulties in operations and troubleshooting.
- Increased monitoring statistics pressure.

---

### 10.2 Viewing Object Lists

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/app-uploads

---

### 10.3 Rough Object Count Statistics

If the object count isn't too large, you can use this method:

docker run --rm \
  -v /data/minio/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure find minio-admin/app-uploads | wc -l

Note:

- Don't frequently perform full find on large production Buckets.
- Performing a full listing on extremely large Buckets may cause pressure.
- Combine with Prometheus metrics, lifecycle policies, and business statistics.

---

### 10.4 Suggestions for Small Object Governance

Recommendations:

- Layer log objects by date.
- Set lifecycle policies for expired objects.
- Set cleanup rules for temporary objects.
- Evaluate in advance for massive small file scenarios.
- For backup archive scenarios, you can merge and package files before upload.
- Don't use object storage as a high-frequency small file database.

---

## Eleven, Nginx Entry Log Operations and Maintenance

### 11.1 Viewing Access Logs

Execute in minio-entry:

tail -f /var/log/nginx/access.log

Filter by domain:

grep "s3.minio.local" /var/log/nginx/access.log | tail -50

View Console access:

grep "console.minio.local" /var/log/nginx/access.log | tail -50

---

### 11.2 Viewing Error Logs

tail -f /var/log/nginx/error.log

Common errors:

upstream timed out
connect() failed
no live upstreams
client intended to send too large body
SSL_do_handshake failed
permission denied
connection refused

---

### 11.3 Troubleshooting 502

If you encounter a 502 when accessing S3 API:

curl -k -I https://s3.minio.local/minio/health/ready

Check Nginx error logs:

tail -100 /var/log/nginx/error.log

Check the backend:

for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
  curl -I http://$ip:9000/minio/health/ready
done

Common causes:

MinIO container is stopped.  
Backend 9000 is unreachable.  
Firewall interception.  
Upstream configuration error.  
All backend nodes are unavailable.  

---

### 11.4 Troubleshooting 413  

If uploading large files results in:  

    413 Request Entity Too Large  

Check Nginx configuration:  

    grep -R "client_max_body_size" /etc/nginx/conf.d/  

Recommendations:  

    client_max_body_size 0;  

Or set according to business needs:  

    client_max_body_size 10G;  

After modification, check and reload:  

    nginx -t  
    systemctl reload nginx  

---

### 11.5 Troubleshooting 504 or upload interruption  

Check Nginx configuration:  

    grep -R "proxy_read_timeout" /etc/nginx/conf.d/  
    grep -R "proxy_send_timeout" /etc/nginx/conf.d/  
    grep -R "proxy_request_buffering" /etc/nginx/conf.d/  

Recommendations to focus on:  

    proxy_connect_timeout 300;  
    proxy_send_timeout 300;  
    proxy_read_timeout 300;  
    proxy_buffering off;  
    proxy_request_buffering off;  

---

## TwelveI don't know.MinIO Prometheus Monitoring Concepts  

### 12.1 Why Prometheus is Needed  

mc and logs are suitable for manual troubleshooting, but production requires continuous monitoring.  

Prometheus can be used for:  

    Periodically collect metrics.  
    Save historical trends.  
    Combine with Grafana for dashboard visualization.  
    Combine with Alertmanager to trigger alerts.  
    Analyze capacity growth.  
    Analyze API error rates.  
    Analyze node and disk status.  

---

### 12.2 Common MinIO Metric Types in Prometheus  

MinIO Prometheus metrics can generally focus on:  

    Cluster capacity metrics  
    Node status metrics  
    Disk status metrics  
    S3 API request metrics  
    Error request metrics  
    Request latency metrics  
    Bucket-related metrics  
    Healing-related metrics  
    Network and process-related metrics  

Metric names may vary across versions.  

In production, use the actual /metrics output from the current MinIO instance as the reference.  

---

### 12.3 MinIO Metric Collection Endpoints  

Common MinIO metric paths include:  

    /minio/v2/metrics/cluster  
    /minio/v2/metrics/node  
    /minio/v2/metrics/bucket  
    /minio/v2/metrics/resource  

Supported paths may vary slightly across versions.  

It is recommended to first use mc to check the Prometheus configuration methods supported by the current version:  

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin prometheus --help  

You can also check:  

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin prometheus generate --help  

---

## ThirteenI don't know.Generating Prometheus Scrape Configuration  

### 13.1 Using mc to Generate Prometheus Configuration  

Execute:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin prometheus generate minio-admin  

Notes:  

    This command outputs an example of Prometheus scrape_config.  
    The output typically includes bearer_token or authentication-related configurations.  
    The output format may vary across versions.  
    Use the actual command output as the reference.  

---

### 13.2 Saving the Generated Results  

Save to a file:  

    mkdir -p /tmp/minio-monitor-demo/prometheus  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-monitor-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin prometheus generate minio-admin > /tmp/minio-monitor-demo/prometheus/minio-prometheus.yml  

View the file:  

    cat /tmp/minio-monitor-demo/prometheus/minio-prometheus.yml  

Security reminder:  

    The generated file may contain a token.  
    Do not commit to Git.  
    Do not place in public notes.  
    In production, manage the token securely.  

---

### 13.3 Prometheus Scrape Configuration Example  

The example configuration structure is as follows, with actual results based on mc generation:  

    scrape_configs:  
      - job_name: minio-cluster  
        metrics_path: /minio/v2/metrics/cluster  
        scheme: https  
        static_configs:  
          - targets:  
              - s3.minio.local  
        bearer_token: <token generated by mc admin prometheus generate>  
        tls_config:  
          insecure_skip_verify: true  

Notes: /minio/v2/metrics/cluster

insecure_skip_verify: true is only suitable for self-signed certificate experiments.
Production should use trusted certificates and set to false or remove this configuration.
bearer_token belongs to sensitive information.
If collecting via Nginx ingress, need to confirm Nginx allows Prometheus to access this path.

---

### 13.4 Example of Collecting Backend HTTP Metrics

If Prometheus and MinIO backend are in the same trusted internal network, you can directly collect backend nodes:

    scrape_configs:
      - job_name: minio-node01
        metrics_path: /minio/v2/metrics/node
        scheme: http
        static_configs:
          - targets:
              - 10.0.0.41:9000
              - 10.0.0.42:9000
              - 10.0.0.43:9000
              - 10.0.0.44:9000
        bearer_token: <token>

Recommendations:

    If you want to simulate client ingress status, collect HTTPS unified ingress.
    If you want to observe each backend node, collect each backend node.
    In production, it's usually necessary to monitor both ingress and backend node status.

---

### 13.5 Testing metrics Interface

If you already have a token, you can use curl to test:

    curl -k \
      -H "Authorization: Bearer <token>" \
      https://s3.minio.local/minio/v2/metrics/cluster | head

Backend HTTP test:

    curl \
      -H "Authorization: Bearer <token>" \
      http://10.0.0.41:9000/minio/v2/metrics/cluster | head

Notes:

    The token needs to be obtained from mc admin prometheus generate output.
    Do not write real tokens into public documents.
    If returns 403 or 401, it indicates authentication configuration is incorrect.
    If returns 502, it indicates Nginx or backend anomaly.

---

## FourteenI don't know.Prometheus Deployment Example

### 14.1 Quick Start Prometheus with Docker

If it's just for experimentation, create directories in minio-client or monitoring node:

    mkdir -p /data/prometheus
    mkdir -p /data/prometheus/conf
    mkdir -p /data/prometheus/data

Create configuration file:

    cat > /data/prometheus/conf/prometheus.yml <<'EOF'
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets:
              - localhost:9090

      - job_name: minio-cluster
        metrics_path: /minio/v2/metrics/cluster
        scheme: https
        static_configs:
          - targets:
              - s3.minio.local
        bearer_token: <replace with token generated by mc>
        tls_config:
          insecure_skip_verify: true
    EOF

Start Prometheus:

    docker run -d \
      --name prometheus-minio-demo \
      --restart unless-stopped \
      -p 9090:9090 \
      -v /data/prometheus/conf/prometheus.yml:/etc/prometheus/prometheus.yml \
      -v /data/prometheus/data:/prometheus \
      prom/prometheus:latest

Notes:

    Here prom/prometheus:latest is only for temporary experimentation.
    Production environment should use fixed version image.
    If facing network issues pulling from domestic network, synchronize to internal Harbor or Alibaba Cloud mirror repository.
    If you already have Prometheus platform, directly integrate into existing platform.

---

### 14.2 View Prometheus Targets

Browser access:

    http://<prometheus-ip>:9090

Check:

    Status -> Targets

Confirm:

    minio-cluster status is UP.

If DOWN:

    Check DNS.
    Check token.
    Check certificate.
    Check Nginx.
    Check metrics_path.
    Check MinIO health.

---

### 14.3 Prometheus Query Examples

Search in Prometheus UI:

    minio

View current collected metrics.

Common query directions:

    Metrics containing cluster
    Metrics containing drive
    Metrics containing bucket
    Metrics containing s3
    Metrics containing healing
    Metrics containing capacity

Notes:

    Metric names may vary across MinIO versions.
    Production alert rules must be based on actual collected metrics.
    Do not blindly copy alert rules that don't match the version.

---

## FifteenI don't know.Grafana Monitoring Dashboard Planning

### 15.1 Recommended Dashboard Types

MinIO production recommends at least the following dashboards:

| Dashboard | Focus Points |
|---|---|
| MinIO Cluster Overview | Cluster Status, Capacity, Request Volume, Error Rate |
| MinIO Node Dashboard | Node Online, CPU, Memory, Network, Processes |
| MinIO Disk Dashboard | Disk Online, Capacity, Usage Rate, I/O |
| MinIO Bucket Dashboard | Bucket Capacity, Object Count, Growth Trend |
| MinIO S3 API Dashboard | Request Volume, Error Rate, Latency, Method Distribution |
| Nginx Ingress Dashboard | 4xx, 5xx, Request Volume, Response Time |
| Certificate Dashboard | HTTPS Certificate Expiry |
| Backup Sync Dashboard | Mirror Task Status, Duration, Failure Count |

---

### 15.2 Cluster Overview Should Include

Recommended to include:

    Current Cluster Capacity.
    Used Capacity.
    Available Capacity.
    Usage Rate.
    Online Node Count.
    Online Disk Count.
    Offline Disk Count.
    S3 Request QPS.
    4xx Error Rate.
    5xx Error Rate.
    API Latency.
    Healing Status.
    Recent Alarms.

---

### 15.3 Bucket Dashboard Should Include

Recommended to include:

    Bucket List.
    Each Bucket Capacity.
    Each Bucket Object Count.
    Each Bucket Growth Trend.
    Top N Large Buckets.
    Top N Fastest Growing Buckets.
    Presence of Unassigned Buckets.
    Presence of Long Unused Buckets.

---

### 15.4 API Dashboard Should Include

Recommended to include:

    GET Request Count.
    PUT Request Count.
    DELETE Request Count.
    LIST Request Count.
    4xx Errors.
    5xx Errors.
    AccessDenied.
    SignatureDoesNotMatch.
    Request Latency P95 / P99.
    Upload/Download Throughput.
    Request Source Distribution.

---

### 15.5 Grafana Dashboard Sources

Optional approaches:

    Use MinIO Official Recommended Dashboard.
    Import Dashboard from Grafana Community.
    Customize Based on PromQL.
    Combine MinIO Metrics with Nginx, Node Exporter, Docker Metrics.

Recommendations:

    Do not assume monitoring is complete by importing a single Dashboard.
    Need to supplement capacity, error rate, backup, and security alarms based on business scenarios.

---

## SixteenI don't know.Alertmanager Alarm Design

### 16.1 Alarm Tiering

Recommended to divide into three categories:

| Tier | Meaning | Examples |
|---|---|---|
| critical | Affects business or may lead to data risks | Cluster Unavailable, Write Failure, Multiple Disks Offline |
| warning | Exist risks but not fully interrupted | Single Node Offline, Capacity Over 80% |
| info | Trend Reminder or Inspection Prompt | Bucket Growth Too Fast, Certificate Expiry Time Insufficient |

Principles:

    Do not mark all alarms as critical.
    Do not let alarms remain unprocessed for long.
    Do not silence critical alarms for long.
    Alarms must have handling manuals.

---

### 16.2 Mandatory Alarm Items

Must alarm:

    MinIO API Unavailable.
    health ready Failed.
    Node Offline.
    Disk Offline.
    Disk Capacity Over Threshold.
    Cluster Capacity Over Threshold.
    S3 5xx Error Rate Increase.
    Nginx 502 / 504 Increase.
    Certificate Expiry Soon.
    Backup Task Failed.
    Healing Long Uncompleted.

---

### 16.3 Recommended Alarm Items

Recommended to alarm:

    Bucket Capacity Abnormal Growth.
    Object Count Abnormal Growth.
    DELETE Request Abnormal Increase.
    AccessDenied Surge.
    SignatureDoesNotMatch Surge.
    Console Abnormal Login.
    root User Frequent Usage.
    Nginx 4xx Surge.
    Large File Upload Failure Increase.

---

### 16.4 Capacity Threshold Recommendations

Capacity threshold recommendations:

| Usage Rate | Handling Suggestions |
|---|---|
| 70% | Monitor Trends |
| 80% | Plan Expansion |
| 85% | Confirm Expansion Time |
| 90% | High Priority Handling |
| 95% | Severe Risk, May Affect Write |

Object storage capacity may grow rapidly, do not wait until 90% to start planning.

---

### 16.5 Alarm Rule Example Explanation

Due to potential changes in metric names across MinIO versions, confirm actual metric names in production.

Process:

    1. Open Prometheus.
    2. Search for minio.
    3. Confirm capacity, disk, request, and error-related metric names.
    4. Write alarm rules based on actual metrics.
    5. Validate alarm triggering in test environment.
    6. Deploy to production after verification.

Do not directly copy alarm rules that do not match the version.

---

## SeventeenI don't know.Log Collection Design

### 17.1 Logs to Collect

MinIO operations recommend collecting:

    MinIO Container Logs
    Docker Service Logs
    Nginx access.log
    Nginx error.log
    System Logs
    Kernel Logs
    Backup Task Logs
    Scheduled Inspection Logs

---

### 17.2 MinIO Container Logs

View:

    docker logs minio

Real-time:

    docker logs -f minio

Collection methods:

    Docker json-file Logs.
    Filebeat Collect Docker Logs.
    Promtail Collect Container Logs.
    journald Collect.
    Unified Output to ELK or Loki.

---

### 17.3 Nginx Logs

Paths:

    /var/log/nginx/access.log
    /var/log/nginx/error.log

Recommended fields to collect:

    client_ip
    host
    request_method
    request_uri
    status
    request_time
    upstream_addr
    upstream_status
    upstream_response_time
    user_agent
    bytes_sent

Usage:

    Analyze API Request Volume.
    Analyze 4xx / 5xx.
    Troubleshoot Upload Failures.
    Identify Abnormal IPs.
    Identify Console Access.
    Audit High-Risk Access Behavior.

---

### 17.4 System Logs

View: /think

journalctl -xe  
journalctl -k  
dmesg  

Attention:  

    Disk errors.  
    Network card errors.  
    OOM.  
    File system read-only.  
    Docker anomalies.  
    System reboot.  
    Time synchronization anomalies.  

---

### 17.5 Backup Task Logs  

If using mc mirror later, retain:  

    Task start time.  
    Task end time.  
    Source.  
    Target.  
    Synchronized object count.  
    Synchronized capacity.  
    Failed objects.  
    Error messages.  
    Exit code.  

The 10-MinIO backup migration will be detailed later.  

---

## Eighteen. Daily Inspection Checklist  

### 18.1 Daily Inspection  

Daily checks:  

    MinIO admin info.  
    health live / ready.  
    Nginx 502 / 504.  
    Nginx 413.  
    Disk capacity.  
    Bucket capacity Top N.  
    Whether there are offline drives.  
    Whether there are offline nodes.  
    Whether there are abnormal AccessDenied.  
    Whether there are backup failures.  
    Whether certificate validity is approaching expiration.  

Command examples:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin  

    df -hT  

    tail -100 /var/log/nginx/error.log  

---

### 18.2 Weekly Inspection  

Weekly checks:  

    Bucket capacity growth trend.  
    Object count growth trend.  
    Large Bucket ownership.  
    Unused Buckets.  
    Whether users and Policies are abnormal.  
    Whether there is frequent root user operations.  
    Whether mirror backups are normal.  
    Whether Grafana dashboard has long-term anomalies.  
    Whether alarm rules are effective.  

---

### 18.3 Monthly Inspection  

Monthly checks:  

    Certificate validity period.  
    AccessKey rotation plan.  
    Cleanup of unused users.  
    Cleanup of unused Policies.  
    Storage capacity expansion plan.  
    Failure drill records.  
    Recovery drill records.  
    Backup recovery availability.  
    Whether documents and contacts are updated.  

---

### 18.4 Pre-Major Change Inspection  

Pre-change checks:  

    Whether the current cluster is healthy.  
    Whether there are offline drives.  
    Whether there is healing.  
    Whether capacity is insufficient.  
    Whether backups are successful.  
    Whether there is a rollback plan.  
    Whether business has been notified.  
    Whether there is a maintenance window.  

Changes involving the following must be handled with caution:  

    Modifying Nginx configuration.  
    Modifying certificates.  
    Modifying MinIO startup parameters.  
    Modifying data directories.  
    Upgrading MinIO version.  
    Deleting Buckets.  
    Modifying Policies.  
    Replacing disks.  
    Replacing nodes.  

---

## Nineteen. Writing Inspection Script Examples  

### 19.1 Script Objectives  

Script objectives:  

    Automatically output MinIO admin info.  
    Check HTTPS health.  
    Check backend node health.  
    Check local disk capacity.  
    Check recent Nginx errors.  
    Save to inspection report file.  

---

### 19.2 Inspection Script Example  

Create script:  

    cat > /usr/local/bin/minio-daily-check.sh <<'EOF'  
    #!/bin/bash  

    set -euo pipefail  

    DATE=$(date +%F-%H%M%S)  
    REPORT_DIR="/var/log/minio-check"  
    REPORT_FILE="${REPORT_DIR}/minio-check-${DATE}.log"  

    MC_IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z"  
    MC_CONFIG="/data/minio/mc-config"  

    mkdir -p "${REPORT_DIR}"  

    {  
      echo "===== MinIO Daily Check ${DATE} ====="  
      echo  

      echo "===== 1. MinIO Admin Info ====="  
      docker run --rm \
        -v ${MC_CONFIG}:/root/.mc \
        ${MC_IMAGE} \
        --insecure admin info minio-admin || true  
      echo  

      echo "===== 2. HTTPS Health Check ====="  
      curl -k -I https://s3.minio.local/minio/health/live || true  
      curl -k -I https://s3.minio.local/minio/health/ready || true  
      echo  

      echo "===== 3. Backend Health Check ====="  
      for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do  
        echo "check ${ip}"  
        curl -I http://${ip}:9000/minio/health/live || true  
        curl -I http://${ip}:9000/minio/health/ready || true  
      done  
      echo  

      echo "===== 4. Disk Usage ====="  
      df -hT || true  
      echo  

      echo "===== 5. Nginx Recent Errors ====="  
      if [ -f /var/log/nginx/error.log ]; then  
        tail -100 /var/log/nginx/error.log || true  
      else  
        echo "nginx error.log not found"  
      fi  
      echo  

      echo "===== 6. Report Finished ====="  
    } > "${REPORT_FILE}" 2>&1

Report saved to ${REPORT_FILE}
EOF

Authorization:

    chmod +x /usr/local/bin/minio-daily-check.sh

Execution:

    /usr/local/bin/minio-daily-check.sh

View report:

    ls -lh /var/log/minio-check/
    tail -100 /var/log/minio-check/minio-check-*.log

---

### 19.3 Example of scheduled execution

Add crontab:

    crontab -e

Example:

    0 9 * * * /usr/local/bin/minio-daily-check.sh >/dev/null 2>&1

Explanation:

    Execute daily at 09:00.
    In production, it's recommended to integrate inspection results into a log platform or notification system.
    If anomalies are detected during inspection, alerts should be triggered rather than only saving to local files.

---

## Twenty, Capacity Management Methodology

### 20.1 Capacity is not just about total capacity

MinIO capacity management requires multi-layered monitoring:

    Cluster total capacity
    Node capacity
    Disk capacity
    Bucket capacity
    Prefix capacity
    Object count
    Growth trend
    Backup capacity

---

### 20.2 Sources of capacity growth

Common growth sources:

    User uploaded attachments.
    Business log archiving.
    CI/CD artifacts.
    Data backups.
    AI datasets.
    Temporary files not cleaned.
    Long-term retention of version files.
    Mirror synchronization duplication.
    Programmatic abnormal cyclic uploads.

---

### 20.3 Capacity governance methods

Governance methods:

    Bucket naming conventions.
    Bucket ownership registration.
    Prefix planning.
    Lifecycle cleanup.
    Expiration object deletion.
    Quota limits.
    Cold data archiving.
    Cross-cluster backup.
    Anomaly growth alerts.
    Regular capacity reviews.

---

### 20.4 Capacity expansion recommendations

Before expansion, confirm:

    Current node count.
    Current disk count.
    Current capacity utilization.
    Current growth rate.
    Current object count.
    Current Bucket distribution.
    Current network bandwidth.
    Current ingress layer capability.
    Current backup capability.

In production, MinIO expansion requires careful planning and cannot simply involve adding directories.

Principles:

    First check official expansion methods.
    First validate in test environments.
    Ensure node, disk, and network planning consistency.
    Backup critical data before changes.
    Monitor during changes.
    Verify read/write and healing status after changes.

---

## Twenty-one, Security Operations Monitoring

### 21.1 Security events to monitor

Focus on:

    Frequent root user usage.
    Abnormal console logins.
    Sudden increase in AccessDenied.
    Sudden increase in SignatureDoesNotMatch.
    Large-scale downloads from a specific IP.
    Large-scale object deletions by a specific user.
    Large uploads outside business hours.
    New unknown Buckets.
    New unknown users.
    Policy modifications.
    AccessKey leaks.

---

### 21.2 Security investigation entry points

Check users:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user list minio-admin

Check Policy:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin policy list minio-admin

View user details:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin user info minio-admin <user-name>

---

### 21.3 Nginx Logs for Security Analysis

View logs from a specific source IP:

    grep "<client-ip>" /var/log/nginx/access.log | tail -100

View 403 errors:

    awk '$9 == 403 {print}' /var/log/nginx/access.log | tail -100

View 5xx errors:

    awk '$9 ~ /^5/ {print}' /var/log/nginx/access.log | tail -100

View DELETE requests:

    grep "DELETE" /var/log/nginx/access.log | tail -100

Note:

    Nginx log formats vary, so field positions may differ.
    In production, it's recommended to customize log formats for easier analysis with ELK / Loki.

---

## Twenty-two, Common Issue Troubleshooting

### 22.1 Prometheus Target DOWN

Troubleshoot:

    1. Check if Prometheus can access MinIO ingress.
    2. Verify DNS resolution.
    3. Confirm metrics_path is correct.
    4. Check token validity.
    5. Verify certificate trust.
    6. Check if Nginx proxies this path.
    7. Confirm MinIO is ready.

Commands:

    curl -k -I https://s3.minio.local/minio/health/ready

    curl -k \
      -H "Authorization: Bearer <token>" \
      https://s3.minio.local/minio/v2/metrics/cluster | head

---

### 22.2 Grafana No Data

Troubleshoot:

    Check if Prometheus Target is UP.
    Verify if PromQL metric names exist.
    Confirm dashboard matches MinIO version.
    Check time range.
    Verify data source.
    Check if Prometheus has permission to scrape.

---

### 22.3 Abnormal Bucket Capacity Growth

Troubleshoot: /think

mc du Bucket.
mc find Bucket.
View Nginx access.log.
Check recent upload source IP.
Check business publish time.
Check for circular uploads.
Check for duplicate write-in backups.
Check if lifecycle policy is missing.

Handling:

First confirm business ownership.
Then confirm if it's abnormal.
Do not delete directly.
Confirm backup and approval before deletion.
Limit AccessKey or pause abnormal business when necessary.

---

### 22.4 Upload failure but MinIO is normal

Possible causes:

    Nginx client_max_body_size.
    Nginx proxy timeout.
    User lacks PutObject permissions.
    Bucket does not exist.
    Disk is full.
    Write quorum not satisfied.
    Client endpoint configured incorrectly.
    Certificate error.
    Proxy Host header causes signature error.

Troubleshooting:

    mc admin info.
    mc ls.
    mc cp small file test.
    mc cp large file test.
    Nginx error.log.
    MinIO docker logs.
    df -hT.

---

### 22.5 Slow download

Possible causes:

    Disk I/O bottleneck.
    Insufficient network bandwidth.
    Nginx proxy bottleneck.
    Client bandwidth insufficient.
    Large amount of healing.
    Backend node offline.
    Too many small objects in Bucket.
    Insufficient client concurrency.

Troubleshooting:

    iostat -x 1.
    iftop.
    docker stats.
    mc admin info.
    Nginx access.log request time.
    Prometheus API latency metrics.

---

## Twenty-three, High-risk operation reminders

The following operations must be handled with caution in production environment:

    Delete Bucket.
    Recursive delete Prefix.
    mc rm --recursive --force.
    Modify Policy.
    Disable user.
    Delete user.
    Modify Nginx configuration.
    Modify certificate.
    Modify MinIO startup parameters.
    Delete data directory.
    Directly clean disk.
    Full heal large Bucket.
    Execute large-scale mirror during peak hours.

Before execution, must confirm:

    Whether there is backup.
    Whether there is approval.
    Whether there is business confirmation.
    Whether there is rollback plan.
    Whether the impact scope is known.
    Whether it is within maintenance window.
    Whether someone has reviewed.
    Whether operation records are retained.

---

## Twenty-four, Production operations baseline

### 24.1 Monitoring baseline

Must have:

    MinIO API availability monitoring.
    MinIO ready monitoring.
    Node status monitoring.
    Disk status monitoring.
    Disk capacity monitoring.
    Bucket capacity monitoring.
    API error rate monitoring.
    Nginx 5xx monitoring.
    Certificate expiration monitoring.
    Backup task monitoring.

---

### 24.2 Alarm baseline

Must alarm:

    MinIO unavailable.
    Node offline.
    Disk offline.
    Capacity exceeds 80%.
    Capacity exceeds 90%.
    S3 5xx anomaly.
    Nginx 502 / 504.
    Certificate will expire.
    Backup failure.
    Healing not completed for a long time.

---

### 24.3 Log baseline

Must retain:

    MinIO container logs.
    Nginx access.log.
    Nginx error.log.
    System logs.
    Docker logs.
    Backup task logs.
    High-risk operation records.

---

### 24.4 Capacity baseline

Must be clear:

    Current total capacity.
    Current available capacity.
    Current usage rate.
    Largest Bucket.
    Fastest growing Bucket.
    Estimated days until full.
    Expansion trigger threshold.
    Backup occupied capacity.
    Lifecycle cleanup policy.

---

### 24.5 Security baseline

Must do:

    Root user does not deploy business.
    Business uses independent users.
    Policy with minimal permissions.
    AccessKey not submitted to Git.
    Console restricts access sources.
    External access uses HTTPS.
    Backend 9000 / 9001 not exposed to public.
    High-risk operations have approval and review.

---

## Twenty-five, Interview answer approach

If asked in an interview:

    How would you do monitoring and operations for MinIO in production environment?

Can answer:

    I would monitor and operate from cluster, node, disk, Bucket, S3 API, entry proxy, security, and backup aspects.
    First use mc admin info and /minio/health/live, /minio/health/ready to judge cluster availability, focus on node online status, disk online status, offline drive, and healing. On node side, also check Docker container, CPU, memory, network, disk I/O and data disk mount status.
    For capacity, I won't only check df -h, but also check Bucket and Prefix capacity, for example use mc du to check business Bucket growth, use mc find to analyze object distribution. In production, need to focus on Top N large Buckets, object count, growth trend, large number of small objects and abnormal uploads.
    On monitoring platform, I would connect MinIO's Prometheus metrics to Prometheus, then use Grafana to display cluster capacity, node status, disk status, S3 request volume, 4xx/5xx, request latency and Bucket growth trend. For alarms, at least cover API unavailable, ready failure, node offline, disk offline, capacity exceeding threshold, Nginx 502/504, certificate expiration, backup failure and healing not completed for a long time.
    For logs, I would collect MinIO container logs, Nginx access.log/error.log, system logs and backup task logs, for troubleshooting upload failure, signature error, AccessDenied, 5xx, abnormal deletion and abnormal download.
    Additionally, MinIO's Erasure Coding cannot replace backup, so also monitor mc mirror or cross-cluster backup tasks, and regularly perform recovery drills. Overall goal is not just to run MinIO, but to form an observable, alarmable, traceable, and recoverable production operations closed-loop.

---

## Twenty-six, Summary of this article

This article completes the study of MinIO monitoring and operations system:

1. MinIO Operations and Maintenance should focus on clusters, nodes, disks, Buckets, Objects, S3 API, ingress proxies, and security risks.
2. `mc admin info` is the core command for manual MinIO inspections.
3. `/minio/health/live` can be used to check process liveness.
4. `/minio/health/ready` is more suitable for determining if the service is ready.
5. Docker logs can be used to view MinIO container logs.
6. Nginx access.log / error.log can be used to troubleshoot ingress layer issues.
7. `df`, `lsblk`, `iostat`, `dmesg` can be used to troubleshoot disk and system issues.
8. `mc du` can view Bucket or Prefix capacity.
9. `mc find` can view object distribution, but large Buckets should not be fully scanned frequently.
10. Prometheus can collect MinIO metrics.
11. `mc admin prometheus generate` can generate Prometheus scrape configuration.
12. Grafana should focus on cluster overview, nodes, disks, Buckets, S3 API, Nginx, and backup tasks.
13. Alertmanager should cover API unavailability, node offline, disk offline, capacity thresholds, 5xx errors, certificate expiration, and backup failures.
14. Capacity management should not only consider total capacity, but also Buckets, Prefixes, object counts, and growth trends.
15. Nginx 502, 413, 504 are common MinIO ingress failures.
16. MinIO production operations must establish daily, weekly, and monthly inspection checklists.
17. High-risk operations must have approval, backups, review, and rollback plans.
18. We will continue to learn MinIO backup migration: `mc mirror`, cross-cluster synchronization, and data migration.

---

## 27. Reference Documents

MinIO Official Documentation:

    https://min.io/docs/minio/linux/index.html

MinIO Monitoring Documentation:

    https://min.io/docs/minio/linux/operations/monitoring.html

MinIO Prometheus Monitoring Documentation:

    https://min.io/docs/minio/linux/operations/monitoring/collect-minio-metrics-using-prometheus.html

MinIO `mc admin prometheus`:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-prometheus.html

MinIO `mc admin info`:

    https://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-info.html

MinIO `mc du`:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-du.html

MinIO `mc find`:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-find.html

MinIO `mc mirror`:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-mirror.html

Nginx Official Documentation:

    https://nginx.org/en/docs/

Prometheus Official Documentation:

    https://prometheus.io/docs/introduction/overview/

Grafana Official Documentation:

    https://grafana.com/docs/

Alertmanager Official Documentation:

    https://prometheus.io/docs/alerting/latest/alertmanager/