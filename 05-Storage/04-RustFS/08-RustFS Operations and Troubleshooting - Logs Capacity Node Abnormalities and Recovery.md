# RustFS Operations and Troubleshooting: Logs, Capacity, Node Abnormalities and Recovery

Recommended path: 05-Storage/04-RustFS/08-RustFS Operations and Troubleshooting: Logs, Capacity, Node Abnormalities and Recovery.md

Tags: #RustFS #ObjectStorage #S3 #TransportInspection #FaultCheck. #LogAnalysis #CapacityManagement #NodeAbnormal #DataRestore #Nginx #Docker #AdvancedSre #ProductionTransport

---

## I. Document Explanation

This is the eighth article of the RustFS module, focusing on learning RustFS daily operations, log viewing, capacity management, node abnormalities, client access abnormalities, Nginx reverse proxy abnormalities, and recovery methods.

Previously completed:

    01-RustFS Basics: S3-compatible object storage and use cases
    02-RustFS Deployment Modes: Single-node and cluster modes
    03-RustFS Single-node Deployment Practice: Service startup, data directory, and access verification
    04-RustFS Cluster Deployment Practice: Multi-node, multi-disk, and access entry
    05-RustFS vs MinIO: Architecture, deployment, ecosystem, and operation differences
    06-RustFS Client Access: S3 API, mc tool, and application integration
    07-RustFS Permissions and Security: Access keys, HTTPS, and reverse proxy

This article focuses on solving:

    What to check during RustFS routine inspections
    How to view RustFS container logs
    How to check RustFS API health status
    How to troubleshoot RustFS Console access abnormalities
    How to inspect RustFS Bucket and Object
    How to check RustFS data directory capacity
    How to handle RustFS node disk space shortages
    How to determine RustFS single-node abnormalities
    How to verify service after RustFS node recovery
    How to troubleshoot RustFS data directory permission abnormalities
    How to troubleshoot RustFS AccessDenied
    How to troubleshoot RustFS SignatureDoesNotMatch
    How to troubleshoot RustFS Nginx 502 / 413 / 504
    How to troubleshoot RustFS large object upload failures
    How to record production failures
    How to establish a production inspection checklist

This article emphasizes:

    Object storage operations cannot only check if the container is Running.
    Need to simultaneously check API, Bucket, Object, capacity, logs, reverse proxy, permissions, certificates, client access, and node health.
    RustFS, as a new object storage solution, must strengthen fault drills, monitoring alerts, and recovery verification before production use.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Establish a routine inspection approach for RustFS.
2. View RustFS Docker container status.
3. View RustFS container logs.
4. Check RustFS API health status.
5. Check Nginx entry health status.
6. Use mc to inspect Bucket and Object.
7. Check data directory capacity.
8. Check node disk and file system status.
9. Troubleshoot container startup failures.
10. Troubleshoot port conflicts.
11. Troubleshoot data directory permission issues.
12. Troubleshoot AccessDenied.
13. Troubleshoot SignatureDoesNotMatch.
14. Troubleshoot EndpointConnectionError.
15. Troubleshoot Nginx 502, 413, 504.
16. Troubleshoot large object upload failures.
17. Simulate single-node abnormalities and observe impacts.
18. Recover abnormal nodes and verify service.
19. Understand key steps for node replacement.
20. Formulate production inspection, alert, and failure record templates.

---

## III. Core Conclusions First

### 3.1 RustFS operations should be layered troubleshooting from entry to data directory

Recommended troubleshooting chain:

    Client / SDK / mc / AWS CLI
        |
        v
    DNS / hosts / Endpoint
        |
        v
    HTTPS / TLS / Certificate
        |
        v
    Nginx / LB
        |
        v
    RustFS API / Console
        |
        v
    RustFS Container / Process
        |
        v
    Data Directory / Disk
        |
        v
    Node Network / CPU / Memory / Time

Do not only check:

    docker ps

Also check:

    curl /health
    mc ls
    docker logs
    Nginx access.log
    Nginx error.log
    df -hT
    du -sh
    ss -lntp
    timedatectl
    Certificate status
    Bucket permissions
    Client error messages

---

### 3.2 Object storage failures should distinguish between "service unavailable" and "permission unavailable"

Common two types of issues:

    Service unavailable:
        Container exited
        Port unreachable
        Nginx 502
        DNS resolution failure
        Node downtime
        Disk full
        Backend health check failure

    Permission unavailable:
        AccessDenied
        SignatureDoesNotMatch
        SecretKey error
        Bucket Policy error
        Business AccessKey insufficient permissions
        Presigned URL expired
        Host rewritten by reverse proxy

Troubleshoot by first determining:

    Is the service unreachable?
    Or is it reachable but no permission?
    Are all Buckets abnormal?
    Or is a specific business Bucket abnormal?
    Are all clients abnormal?
    Or is a specific SDK abnormal?

---

### 3.3 Capacity issues are high-risk points for object storage production failures

Object storage capacity risks include:

    Data directory full
    System disk full
    Nginx temporary directory full
    Logs full
    Single-node capacity imbalance
    A disk full
    Bucket object count out of control
    Backup packages not cleaned for a long time
    Log archive without lifecycle policy
    Large object upload causing sudden capacity increase

Production must set capacity thresholds:

| Usage Rate | Recommended Actions |
|---|---|
| 70% | Start monitoring growth trends |
| 80% | Plan expansion and cleanup |
| 85% | Enter high-priority handling |
| 90% | Severe risk, expand or clean up as soon as possible |
| 95% | Critical risk, may affect writes |

### 3.4 Node Recovery Is Not Just a Simple Container Restart

When a distributed object storage node fails, recovery should focus on:

    Whether the node IP / hostname is consistent
    Whether hosts / DNS is correct
    Whether RustFS version is consistent
    Whether startup parameters are consistent
    Whether AccessKey / SecretKey is consistent
    Whether data directory is correctly mounted
    Whether data directory permissions are correct
    Whether the disk is healthy
    Whether node time is synchronized
    Whether container logs are normal
    Whether the cluster has started repair / healing
    Whether client read/write is normal

Do not directly:

    Start a container with different parameters after replacing the node
    Change the original hostname without updating cluster resolution
    Use a different version image
    Use a different data directory
    Manually modify internal files in RustFS data directory
    rm -rf data directory without understanding

---

## Four. Experimental Environment

### 4.1 RustFS Cluster Nodes

| IP | Hostname | Purpose |
|---|---|---|
| 10.0.0.51 | rustfs-node01 | RustFS Node 1 |
| 10.0.0.52 | rustfs-node02 | RustFS Node 2 |
| 10.0.0.53 | rustfs-node03 | RustFS Node 3 |
| 10.0.0.54 | rustfs-node04 | RustFS Node 4 |
| 10.0.0.55 | rustfs-client | mc / AWS CLI client |
| 10.0.0.56 | rustfs-entry | Nginx HTTPS unified entry |

---

### 4.2 Access Entry

RustFS backend API:

    http://10.0.0.51:9000
    http://10.0.0.52:9000
    http://10.0.0.53:9000
    http://10.0.0.54:9000

RustFS backend Console:

    http://10.0.0.51:9001
    http://10.0.0.52:9001
    http://10.0.0.53:9001
    http://10.0.0.54:9001

Unified S3 API entry:

    https://s3.rustfs.local

Unified Console entry:

    https://console.rustfs.local/rustfs/console

---

### 4.3 Data Directory

Each RustFS node:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

Experimental reminder:

    If these directories are all on the same system disk, they can only be used for functional experiments.
    In production environment, each directory should correspond to an independent data disk or mount point.
    Do not store production object data on the system disk.
    Do not use NFS as the underlying data directory for RustFS.

---

### 4.4 Fixed Image Version

Current module uses:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/rustfs:1.0.0-alpha.99

Notes:

    Fixed version facilitates experiment reproducibility.
    Do not use latest.
    Current version is suitable for learning and verification.
    Production version must be re-evaluated with stability, security fixes, compatibility, and recovery drills.

---

## Five. Daily Inspection Overview

### 5.1 Daily Inspection Items

Daily inspection recommendations include:

    Whether RustFS container is running
    Whether RustFS API is healthy
    Whether RustFS Console is accessible
    Whether Nginx is running
    Whether HTTPS certificate is normal
    Whether Bucket can be listed
    Whether test object upload/download is normal
    Whether data directory capacity is normal
    Whether node disk is abnormal
    Whether node time is synchronized
    Whether Nginx has a large number of 4xx / 5xx
    Whether RustFS logs have anomalies
    Whether client reports AccessDenied / SignatureDoesNotMatch

---

### 5.2 Weekly Inspection Items

Weekly inspection recommendations include:

    Bucket count
    Capacity growth trend of important Buckets
    Large object count
    Whether backup Buckets are continuously growing
    Whether log archive needs cleaning
    Whether there are abnormal AccessKeys
    Whether there are long-term unrotated keys
    Nginx log size
    RustFS log size
    Node disk SMART status
    Node load and network bandwidth
    Whether there are long-term failed requests

---

### 5.3 Monthly Inspection Items

Monthly inspection recommendations include:

    Fault drill
    Node restart drill
    Single-node anomaly recovery drill
    Certificate expiration time
    AccessKey rotation status
    Minimum privilege review
    Bucket lifecycle policy review
    Capacity expansion plan
    Version upgrade evaluation
    Recovery process review
    Monitoring alert effectiveness test

---

## Six. Basic Inspection Commands

### 6.1 Check Node Status

Execute on all RustFS nodes:

    hostname
    hostname -I
    uptime
    date
    timedatectl
    df -hT
    free -h
    top -b -n 1 | head -20

Focus on:

    Whether the node is online.
    Whether time is synchronized.
    Whether disk is near full.
    Whether memory is abnormal.
    Whether load is abnormal.

---

### 6.2 Check Docker Status

Execute:

    systemctl status docker --no-pager
    docker version
    docker ps
    docker ps -a

Check RustFS containers:

    docker ps | grep rustfs
    docker ps -a | grep rustfs

Single-node mode:

    docker ps | grep rustfs-single

Cluster mode:

    docker ps | grep rustfs-cluster

---

### 6.3 Check RustFS Logs

Single-node mode:

    docker logs rustfs-single --tail=100

Cluster mode:

    docker logs rustfs-cluster --tail=100

Continuous observation:

    docker logs -f rustfs-cluster

Check recent errors:

    docker logs rustfs-cluster --since "30m" | grep -Ei "error|failed|denied|panic|timeout|refused|permission"

Notes:

    Docker logs are the most direct service troubleshooting entry.
    If the container restarts repeatedly, check docker logs first.
    Do not only check docker ps.

---

### 6.4 Check Port Listening

On RustFS nodes execute: /think

ss -lntp | grep ':9000'
ss -lntp | grep ':9001'

Run at the entry node:

    ss -lntp | grep ':80'
    ss -lntp | grep ':443'

Determine:

    9000 is the S3 API port.
    9001 is the Console port.
    80 / 443 is the Nginx entry port.
    If the port is not listening, the client will definitely be unable to access.

---

### 6.5 Check API Health Status

Backend node:

    curl -i http://10.0.0.51:9000/health
    curl -i http://10.0.0.52:9000/health
    curl -i http://10.0.0.53:9000/health
    curl -i http://10.0.0.54:9000/health

Unified entry point:

    curl -k -i https://s3.rustfs.local/health

Expected:

    HTTP 200

If the backend is normal but the unified entry point fails:

    Prioritize checking Nginx, certificates, DNS, upstream, and firewall.

If the backend also fails:

    Prioritize checking RustFS container, ports, logs, and node network.

---

## SevenI don't know.Nginx Entry Point Inspection

### 7.1 Check Nginx Service

Run on rustfs-entry:

    systemctl status nginx --no-pager
    nginx -t
    nginx -v

Check ports:

    ss -lntp | grep nginx

---

### 7.2 Check S3 API Entry Point Logs

Access log:

    tail -100 /var/log/nginx/rustfs-s3-access.log

Error log:

    tail -100 /var/log/nginx/rustfs-s3-error.log

Real-time observation:

    tail -f /var/log/nginx/rustfs-s3-access.log
    tail -f /var/log/nginx/rustfs-s3-error.log

Focus on:

    403
    404
    413
    499
    500
    502
    504

---

### 7.3 Check Console Entry Point Logs

Access log:

    tail -100 /var/log/nginx/rustfs-console-access.log

Error log:

    tail -100 /var/log/nginx/rustfs-console-error.log

Focus on:

    Login failure
    Certificate error
    WebSocket connection anomaly
    502
    504
    Source IP anomaly

---

### 7.4 Check Certificates

Check certificate validity:

    openssl x509 -in /etc/nginx/certs/rustfs/rustfs.local.crt -noout -dates

Check certificate SAN:

    openssl x509 -in /etc/nginx/certs/rustfs/rustfs.local.crt -noout -text | grep -E "DNS:|IP Address"

Check from client:

    openssl s_client -connect s3.rustfs.local:443 -servername s3.rustfs.local </dev/null 2>/dev/null | openssl x509 -noout -dates

Production requirements:

    Must alert before certificate expiration.
    Private key permissions must be strictly restricted.
    Certificate domain must match the Endpoint.

---

## EightI don't know.Bucket and Object Inspection

### 8.1 Configure mc Inspection Alias

Run on rustfs-client:

    mkdir -p /data/rustfs-ops/mc-config

Configure HTTPS alias:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias set rustfs-ops https://s3.rustfs.local rustfsadmin 'RustFSAdmin@123456'

Check:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

Notes:

    --insecure is only for self-signed certificate experiments.
    Production should use trusted certificates and not rely long-term on --insecure.

---

### 8.2 View Bucket List

Run:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-ops

---

### 8.3 View Bucket Objects

View prod-app-uploads:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls --recursive rustfs-ops/prod-app-uploads | head -50

View object details:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure stat rustfs-ops/prod-app-uploads/hello.txt

---

### 8.4 Upload/Download Inspection Objects

Create inspection file: /think

```markdown
echo "rustfs daily check $(date)" > /tmp/rustfs-daily-check.txt

Upload:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /tmpdata/rustfs-daily-check.txt rustfs-ops/prod-app-uploads/ops/rustfs-daily-check.txt

Download:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp rustfs-ops/prod-app-uploads/ops/rustfs-daily-check.txt /tmpdata/rustfs-daily-check-download.txt

Verification:

    diff /tmp/rustfs-daily-check.txt /tmp/rustfs-daily-check-download.txt

If diff produces no output, it indicates the uploaded and downloaded content are consistent.

---

## IX. Capacity Inspection

### 9.1 Check Node Disk Capacity

Execute on each RustFS node:

    df -hT

Focus on:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

Alternatively execute:

    df -hT /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

---

### 9.2 Check Data Directory Usage

Execute:

    du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Check finer directories:

    du -h --max-depth=1 /data/rustfs0 | sort -h | tail -20
    du -h --max-depth=1 /data/rustfs1 | sort -h | tail -20
    du -h --max-depth=1 /data/rustfs2 | sort -h | tail -20
    du -h --max-depth=1 /data/rustfs3 | sort -h | tail -20

Notes:

    Can check directory capacity.
    Do not manually modify RustFS internal data files.
    Do not directly delete internal object data directories.
    Object deletion should be performed via S3 API, mc, or Console.

---

### 9.3 Check System Disk Capacity

Execute:

    df -hT /
    du -h --max-depth=1 /var | sort -h | tail -20
    du -h --max-depth=1 /var/lib/docker | sort -h | tail -20

Reasons:

    Docker logs may fill up the system disk.
    Nginx logs may fill up the system disk.
    A full system disk will affect Docker, Nginx, and system services.

---

### 9.4 Check Docker Log Size

Execute:

    docker inspect rustfs-cluster --format='{{.LogPath}}'

Assume output is:

    /var/lib/docker/containers/<container-id>/<container-id>-json.log

Check size:

    ls -lh /var/lib/docker/containers/*/*-json.log | sort -k5 -h | tail -20

Production recommendations:

    Configure Docker log-opts.
    Limit individual container log size.
    Collect logs to a centralized system.
    Prevent container logs from growing indefinitely.

Example:

    cat > /etc/docker/daemon.json <<'EOF'
    {
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m",
        "max-file": "5"
      }
    }
    EOF

Restart Docker only after evaluating impacts:

    systemctl restart docker

High-risk warning:

    Restarting Docker will affect all containers on the current node.
    Production changes must be done during maintenance windows.

---

### 9.5 Capacity Alert Recommendations

Recommended alerts:

| Metric | Threshold |
|---|---|
| Data disk usage | > 80% Warning |
| Data disk usage | > 90% Critical |
| System disk usage | > 80% Warning |
| System disk usage | > 90% Critical |
| Abnormal growth of Nginx log directory | Warning |
| Abnormal growth of Docker logs | Warning |
| Abnormal growth of Bucket objects | Warning |
| Abnormal daily capacity increase | Warning |

---

## X. Node Resource Inspection

### 10.1 CPU and Memory

Execute:

    top
    htop
    free -h
    uptime

If htop is not available:

    apt update
    apt install -y htop

Monitor:

    Whether CPU is under long-term high load.
    Whether memory is continuously increasing.
    Whether swap is used.
    Whether RustFS containers are abnormally consuming resources.

---

### 10.2 Disk I/O

Install tools:

    apt update
    apt install -y sysstat

Check:

    iostat -x 1 10

Focus on:

    r/s
    w/s
    rkB/s
    wkB/s
    await
    %util

Judgment: /think
```

%util is close to 100%, disk is nearly full.
await is significantly elevated, indicating high disk I/O latency.
I/O will increase during large object uploads, node recovery, and healing.

---

### 10.3 Network Traffic

Run:

    sar -n DEV 1 10

Or install iftop:

    apt install -y iftop
    iftop

Monitor:

    Traffic between nodes.
    Nginx ingress traffic.
    Upload/download peaks.
    Presence of abnormal external sources.
    Whether network is saturated during node recovery.

---

### 10.4 Time Synchronization

Run:

    timedatectl

Requirements:

    System clock synchronized: yes

If time is out of sync:

    apt update
    apt install -y systemd-timesyncd
    systemctl enable --now systemd-timesyncd
    timedatectl set-ntp true
    timedatectl

Reasons:

    S3 signatures depend on time.
    Time drift may cause SignatureDoesNotMatch.
    Fault log analysis relies on accurate time.

---

## Eleven. Practical Operation I: Container Startup Failure Troubleshooting

### 11.1 Check Container Status

Run:

    docker ps -a | grep rustfs

If you see:

    Exited
    Restarting
    Created

It indicates the container is not running normally.

---

### 11.2 Check Logs

Run:

    docker logs rustfs-cluster --tail=200

Focus on searching:

    error
    failed
    permission denied
    address already in use
    no such file
    not found
    refused
    panic
    timeout

Command:

    docker logs rustfs-cluster --tail=300 | grep -Ei "error|failed|permission|denied|panic|timeout|refused|not found"

---

### 11.3 Common Causes and Solutions

| Phenomenon | Possible Cause | Solution |
|---|---|---|
| permission denied | Data directory permission error | chown -R 10001:10001 |
| address already in use | Port is occupied | ss -lntp to check process |
| no such file | Data directory does not exist | Create directory and mount |
| hostname resolve failed | hosts/DNS error | getent hosts to check |
| connection refused | Other nodes not started or port unreachable | Check network and firewall |
| version mismatch | Node versions are inconsistent | Unify image version |
| secret mismatch | Secret keys are inconsistent | Unify AccessKey/SecretKey |

---

### 11.4 Permission Repair Example

Run:

    ls -ld /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Fix:

    chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Restart:

    docker restart rustfs-cluster

Check:

    docker logs rustfs-cluster --tail=100

---

## Twelve. Practical Operation II: Port Conflict Troubleshooting

### 12.1 Check Port Usage

Run:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

Check process:

    ps -ef | grep <PID>

---

### 12.2 Handling Approach

If 9000 is occupied:

    Confirm if there is an existing RustFS container.
    Confirm if it's MinIO or other object storage.
    Confirm if it's a historical test process.
    Do not kill unknown processes directly.

If it's an old container:

    docker ps -a
    docker rm -f <old container name>

If you need to change the port:

    All RustFS node ports and RUSTFS_VOLUMES must remain consistent.
    Nginx upstream must be updated synchronously.
    mc/SDK Endpoint must be updated synchronously.

---

## Thirteen. Practical Operation III: AccessDenied Troubleshooting

### 13.1 Phenomenon

Client error:

    AccessDenied
    403 Forbidden
    Access Denied

---

### 13.2 Troubleshooting Path

First, confirm alias:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

Second, confirm if admin account can access:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-ops

Third, confirm if business account can only access target Bucket:

    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-app/prod-app-uploads

Fourth, try accessing unrelated Bucket:

docker run --rm \
  -v /data/rustfs-security/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls rustfs-app/prod-backups

---

### 13.3 Judgment

If the administrator can access but business accounts cannot:

    Most likely an issue with business AccessKey permissions.
    Check Policy.
    Check Bucket.
    Check if the corresponding actions are granted.

If no account can access:

    It could be a service, entry, Nginx, certificate, or cluster anomaly.

If only a specific operation fails:

    PutObject failure: lack of upload permissions.
    GetObject failure: lack of read permissions.
    DeleteObject failure: lack of delete permissions.
    ListBucket failure: lack of listing permissions.

---

## FourteenI don't know.Operation Four: SignatureDoesNotMatch Troubleshooting

### 14.1 Phenomenon

Client error:

    SignatureDoesNotMatch
    The request signature we calculated does not match the signature you provided

---

### 14.2 Common Causes

Common causes:

    SecretKey error.
    AccessKey and SecretKey mismatch.
    Client time not synchronized.
    Server time not synchronized.
    Region configuration inconsistency.
    Endpoint configuration inconsistency.
    HTTP / HTTPS inconsistency.
    Nginx Host rewrite.
    Presigned URL generated with internal network address but accessed via external network.
    SDK uses Virtual-hosted-style, but entry only supports Path-style.
    Request path rewritten after proxy.

---

### 14.3 Troubleshooting Commands

Check time:

    timedatectl

Check Endpoint:

    curl -k -i https://s3.rustfs.local/health

Check Nginx Host transmission configuration:

    grep -R "proxy_set_header Host" /etc/nginx/conf.d/

Should include:

    proxy_set_header Host $http_host;

Check alias:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

Check Nginx logs:

    tail -100 /var/log/nginx/rustfs-s3-access.log
    tail -100 /var/log/nginx/rustfs-s3-error.log

Check RustFS logs:

    docker logs rustfs-cluster --tail=200

---

### 14.4 Handling Recommendations

Handling direction:

    Reconfirm AccessKey / SecretKey.
    Synchronize client and server time.
    Test with Region us-east-1.
    Use unified external HTTPS domain for Endpoint.
    Nginx preserves Host.
    SDK enables path-style.
    Generate Presigned URL with user's actual access domain.
    Do not use internal network Endpoint to generate public access URL.

---

## FifteenI don't know.Operation Five: Nginx 502 Troubleshooting

### 15.1 Phenomenon

Access:

    https://s3.rustfs.local

Returns:

    502 Bad Gateway

---

### 15.2 Troubleshooting Entry

Execute in rustfs-entry:

    nginx -t
    systemctl status nginx --no-pager
    tail -100 /var/log/nginx/rustfs-s3-error.log

Check upstream backend:

    curl -i http://10.0.0.51:9000/health
    curl -i http://10.0.0.52:9000/health
    curl -i http://10.0.0.53:9000/health
    curl -i http://10.0.0.54:9000/health

---

### 15.3 Common Causes

Common causes:

    Backend RustFS container stopped.
    Backend port 9000 unreachable.
    Upstream address error.
    Firewall block.
    RustFS node restart.
    RustFS health anomaly.
    DNS / hosts error.

---

### 15.4 Handling

If a backend is abnormal:

    Log in to the corresponding RustFS node.
    Check docker ps.
    Check docker logs.
    Check ss -lntp.
    Check df -hT.
    Restore backend service.

If multiple backends are abnormal:

    Determine if it's a cluster-level failure.
    Suspend changes.
    Preserve logs.
    Do not blindly delete data directories.
    Follow the failure handling process.

---

## SixteenI don't know.Operation Six: Nginx 413 Troubleshooting

### 16.1 Phenomenon

Error when uploading large files:

    413 Request Entity Too Large

---

### 16.2 Cause

Nginx limits request body size.

---

### 16.3 Check Configuration

Execute:

    grep -R "client_max_body_size" /etc/nginx/conf.d/

If no configuration or too small, adjust it.

Object storage scenario recommendation:

    client_max_body_size 0;

Or based on business limits:

    client_max_body_size 10g;

---

### 16.4 Reload Configuration

Execute:

    nginx -t
    systemctl reload nginx

Test upload again.

---

## SeventeenI don't know.Operation Seven: Nginx 504 Troubleshooting

### 17.1 Phenomenon

Error when uploading or downloading large files:

    504 Gateway Timeout

---

### 17.2 Common Causes

Common causes:

proxy_read_timeout is too short.
proxy_send_timeout is too short.
RustFS backend responds slowly.
Backend disk I/O is high.
Node network jitter.
Client network is slow.
Large object upload exceeds Nginx timeout.

---

### 17.3 Check Configuration

Execute:

    grep -R "proxy_read_timeout" /etc/nginx/conf.d/
    grep -R "proxy_send_timeout" /etc/nginx/conf.d/
    grep -R "proxy_connect_timeout" /etc/nginx/conf.d/

Recommendations:

    proxy_connect_timeout 60s;
    proxy_send_timeout 3600s;
    proxy_read_timeout 3600s;
    send_timeout 3600s;

Reload:

    nginx -t
    systemctl reload nginx

---

### 17.4 Check Backend Resources Simultaneously

On RustFS node execute:

    iostat -x 1 10
    sar -n DEV 1 10
    docker logs rustfs-cluster --tail=100
    df -hT

If disk or network is already full, simply increasing Nginx timeout cannot resolve the root cause.

---

## EighteenI don't know.Practical Step 8: Troubleshoot Large Object Upload Failures

### 18.1 Prepare Large File for Testing

On rustfs-client execute:

    dd if=/dev/zero of=/tmp/rustfs-large-test.bin bs=1M count=1024
    sha256sum /tmp/rustfs-large-test.bin > /tmp/rustfs-large-test.sha256

---

### 18.2 Upload Test

Execute:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /tmpdata/rustfs-large-test.bin rustfs-ops/prod-app-uploads/large/rustfs-large-test.bin

---

### 18.3 Troubleshoot Failure

Client side:

    Is network interrupted?
    Does mc report 413?
    Does mc report 504?
    Does mc report SignatureDoesNotMatch?
    Does mc report AccessDenied?

Nginx side:

    tail -100 /var/log/nginx/rustfs-s3-error.log
    tail -100 /var/log/nginx/rustfs-s3-access.log

RustFS side:

    docker logs rustfs-cluster --tail=200

Node side:

    df -hT
    iostat -x 1 10
    sar -n DEV 1 10

Common causes:

    Nginx upload size limit.
    Nginx timeout.
    Backend disk space insufficient.
    Backend I/O pressure too high.
    Network instability.
    Client SDK multipart upload compatibility issues.
    Certificate or signature issues.

---

## NineteenI don't know.Single Node Abnormality Drill

### 19.1 Drill Description

Objective:

    Simulate a RustFS backend node abnormality.
    Observe if the unified entry point remains available.
    Observe if Bucket and Object are accessible.
    Validate service after node recovery.

High-risk warning:

    Only execute in experimental environment.
    Production fault drills must be approved in advance.
    Must confirm backups, recovery plans, and maintenance window before drill.

---

### 19.2 Stop One Node

On rustfs-node04 execute:

    docker stop rustfs-cluster

Check:

    docker ps -a | grep rustfs-cluster

---

### 19.3 Client Access Verification

On rustfs-client execute:

    curl -k -i https://s3.rustfs.local/health

Check Bucket:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-ops

Download test object:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp rustfs-ops/prod-app-uploads/ops/rustfs-daily-check.txt /tmpdata/rustfs-daily-check-after-node-stop.txt

---

### 19.4 Check Nginx Logs

On rustfs-entry execute:

    tail -100 /var/log/nginx/rustfs-s3-error.log

May see:

    connect() failed
    upstream timed out
    no live upstreams

If only one backend node is abnormal, Nginx should continue trying other backends.

If multiple backends are abnormal, the unified entry point may become unavailable.

---

### 19.5 Recover Node

On rustfs-node04 execute:

    docker start rustfs-cluster

Check logs:

    docker logs rustfs-cluster --tail=200

Check health:

    curl -i http://10.0.0.54:9000/health

Client re-check:

    curl -k -i https://s3.rustfs.local/health

---

### 19.6 Verify After Recovery

Upload object: /think

```bash
echo "node04 recovery check $(date)" > /tmp/node04-recovery-check.txt

docker run --rm \
  -v /data/rustfs-ops/mc-config:/root/.mc \
  -v /tmp:/tmpdata \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure cp /tmpdata/node04-recovery-check.txt rustfs-ops/prod-app-uploads/ops/node04-recovery-check.txt

View:

docker run --rm \
  -v /data/rustfs-ops/mc-config:/root/.mc \
  registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  --insecure ls rustfs-ops/prod-app-uploads/ops/

---

## Twenty, Node Replacement Strategy

### 20.1 Applicable Scenarios

Node replacement applies to:

    Node hardware failure
    System disk damage
    Node irrecoverable failure
    Network card failure
    Host long-term offline
    Need to replace old node with new server

---

### 20.2 Node Replacement Principles

Replace nodes should maintain:

    Consistent hostname
    Consistent IP or DNS resolution
    Consistent OS version
    Docker version close to original
    RustFS image version consistent
    Startup parameters consistent
    RUSTFS_VOLUMES consistent
    AccessKey / SecretKey consistent
    Data directory path consistent
    Disk quantity consistent
    Disk capacity and performance as consistent as possible
    Time synchronization normal

---

### 20.3 Replacement Process

Recommended process:

    1. Confirm the faulty node is irrecoverable.
    2. Preserve the faulty site logs.
    3. Prepare new node hardware and system.
    4. Set the same hostname.
    5. Update DNS / hosts to point old hostname to new node.
    6. Install Docker.
    7. Pull the same RustFS image.
    8. Prepare the same data directory.
    9. Set directory permissions to UID 10001.
    10. Join the cluster with the same startup parameters.
    11. Check RustFS logs.
    12. Observe healing / recovery process.
    13. Validate client read/write.
    14. Record recovery time and impact scope.

---

### 20.4 Prohibited Operations During Node Replacement

Prohibited:

    Use different image versions to join directly.
    Use different RUSTFS_VOLUMES.
    Use different hostnames without updating resolution.
    Use different AccessKey / SecretKey.
    Directly modify internal data directory.
    Arbitrarily copy old node residual data to new node.
    Manually fix internal files without understanding data layout.
    Replace multiple nodes simultaneously.

---

## Twenty-one, Disk Abnormality Handling

### 21.1 Disk Space Insufficient

Phenomenon:

    Upload failure
    500 / 503
    Container logs report space insufficient
    df -hT shows 100%
    Nginx may report 502 / 504

Check:

    df -hT /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Handling direction:

    Confirm which Bucket has abnormal growth.
    Clean up unused objects.
    Clean up historical backups.
    Expand disk.
    Add nodes.
    Add lifecycle policies.
    Do not manually delete RustFS internal data files.

---

### 21.2 Disk I/O Abnormality

Check:

    iostat -x 1 10
    dmesg | tail -100
    journalctl -k --since "1 hour ago" | tail -100

Focus on:

    await
    %util
    Disk errors
    I/O error
    Filesystem error

Handling:

    Determine if hard disk failure.
    Determine if recovery/healing is ongoing.
    Determine if large object upload peak.
    Determine if logs or other processes are occupying disk.
    Prepare node or disk replacement if necessary.

---

### 21.3 Data Directory Permission Abnormality

Phenomenon:

    Permission denied
    Container startup failure
    Object write failure

Check:

    ls -ld /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    docker logs rustfs-cluster --tail=200

Fix:

    chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
    docker restart rustfs-cluster

---

## Twenty-two, Monitoring and Alerting Recommendations

### 22.1 Mandatory Monitoring

Must monitor:

    RustFS API /health
    RustFS container status
    RustFS ports 9000 / 9001
    Nginx 80 / 443
    Nginx 502 / 504
    Nginx 413
    Data disk capacity
    System disk capacity
    Node CPU / Memory
    Node disk I/O
    Node network traffic
    Certificate expiration time
    AccessDenied exceptions increasing
    SignatureDoesNotMatch exceptions increasing
    Large number of DeleteObject operations
    Bucket capacity sudden increase

---

### 22.2 Prometheus Direction

RustFS official documentation states its support for Prometheus-compatible metrics and health check endpoints.

Production planning:

    Prometheus scrape RustFS metrics
    Prometheus blackbox probe /health
    Node Exporter monitor host resources
    Nginx Exporter monitor entry status
    Grafana display capacity, request volume, error rate, latency
    Alertmanager send alerts

Basic monitoring structure: /think

Prometheus
      |
      +--> RustFS Metrics
      +--> RustFS /health
      +--> Node Exporter
      +--> Nginx Exporter
      +--> Blackbox Exporter

---

### 22.3 Alert Levels

| Alert | Level | Description |
|---|---|---|
| RustFS /health Unreachable | Critical | Object storage may be unavailable |
| Multiple backend nodes unavailable | Critical | Cluster high risk |
| Single backend node unavailable | Warning | Needs to be recovered soon |
| Data disk > 90% | Critical | May affect writing |
| Data disk > 80% | Warning | Needs expansion plan |
| Nginx 502 increase | Critical | Backend anomaly |
| Nginx 504 increase | Warning / Critical | Timeout or performance anomaly |
| Certificate expires within 15 days | Warning | Needs renewal |
| AccessDenied surge | Warning | Permission or attack risk |
| SignatureDoesNotMatch surge | Warning | Signature, time, or attack risk |
| Large number of object deletions | Critical | Possible accidental deletion or attack |

---

## Twenty-ThreeI don't know.Production Incident Handling Process

### 23.1 First Stage: Confirm Impact

Confirm:

    Whether upload is affected.
    Whether download is affected.
    Whether all Buckets are affected.
    Whether a single business is affected.
    Whether Console is affected.
    Whether API is affected.
    Whether only HTTPS is abnormal.
    Whether only a specific client is abnormal.
    Whether Nginx entry is abnormal.
    Whether backend RustFS is abnormal.

---

### 23.2 Second Stage: Preserve Scene

Preserve:

    Nginx access.log
    Nginx error.log
    RustFS docker logs
    docker ps -a
    df -hT
    ss -lntp
    timedatectl
    Client error information
    Change records
    Recent release records
    Recent permission change records

Commands:

    mkdir -p /tmp/rustfs-incident-$(date +%F-%H%M%S)

---

### 23.3 Third Stage: Layered Localization

In order:

    1. DNS / Endpoint
    2. TLS / Certificate
    3. Nginx
    4. RustFS API
    5. RustFS Container
    6. Data directory / Disk
    7. AccessKey / Policy
    8. SDK / Client
    9. Network
    10. Recent changes

---

### 23.4 Fourth Stage: Restore Service

Restore priority:

    Prioritize restoring read.
    Then restore write.
    Then restore Console.
    Then handle non-core Buckets.
    Then perform root cause analysis.
    Then perform long-term fixes.

Notes:

    Do not delete data before root cause is identified.
    Do not directly rebuild the cluster.
    Do not restart all nodes simultaneously.
    Do not manually modify internal data directories.

---

## Twenty-FourI don't know.Production Patrol Script Example

### 24.1 Patrol Script Description

The following script is for basic patrol:

    Check node information
    Check Docker
    Check RustFS container
    Check ports
    Check disk
    Check time synchronization
    Check API health

Save as:

    /usr/local/bin/rustfs-node-check.sh

---

### 24.2 Node Patrol Script

Create:

    cat > /usr/local/bin/rustfs-node-check.sh <<'EOF'
    #!/usr/bin/env bash
    set -euo pipefail

    echo "========== RustFS Node Check =========="
    echo "Time: $(date)"
    echo

    echo "----- Host -----"
    hostname
    hostname -I || true
    uptime
    echo

    echo "----- Time Sync -----"
    timedatectl || true
    echo

    echo "----- Docker -----"
    systemctl is-active docker || true
    docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep -E 'rustfs|NAMES' || true
    echo

    echo "----- Ports -----"
    ss -lntp | grep -E ':9000|:9001' || true
    echo

    echo "----- Disk -----"
    df -hT / /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3 2>/dev/null || df -hT
    echo

    echo "----- RustFS Logs Last 30 Lines -----"
    if docker ps -a --format '{{.Names}}' | grep -q '^rustfs-cluster$'; then
      docker logs rustfs-cluster --tail=30 || true
    elif docker ps -a --format '{{.Names}}' | grep -q '^rustfs-single$'; then
      docker logs rustfs-single --tail=30 || true
    else
      echo "No rustfs container found"
    fi
    echo

    echo "----- Health -----"
    curl -sS -i http://127.0.0.1:9000/health || true
    echo
    EOF

Authorize:

    chmod +x /usr/local/bin/rustfs-node-check.sh

Execute:

    /usr/local/bin/rustfs-node-check.sh

---

### 24.3 Entry Patrol Script /think

Save as:

    /usr/local/bin/rustfs-entry-check.sh

Create:

    cat > /usr/local/bin/rustfs-entry-check.sh <<'EOF'
    #!/usr/bin/env bash
    set -euo pipefail

    echo "========== RustFS Entry Check =========="
    echo "Time: $(date)"
    echo

    echo "----- Nginx Status -----"
    systemctl is-active nginx || true
    nginx -t || true
    echo

    echo "----- Ports -----"
    ss -lntp | grep -E ':80|:443' || true
    echo

    echo "----- Cert Dates -----"
    if [ -f /etc/nginx/certs/rustfs/rustfs.local.crt ]; then
      openssl x509 -in /etc/nginx/certs/rustfs/rustfs.local.crt -noout -dates || true
    else
      echo "Cert file not found"
    fi
    echo

    echo "----- Backend Health -----"
    for ip in 10.0.0.51 10.0.0.52 10.0.0.53 10.0.0.54; do
      echo "Checking $ip"
      curl -sS -i --connect-timeout 3 http://$ip:9000/health || true
      echo
    done

    echo "----- HTTPS Entry Health -----"
    curl -k -i https://s3.rustfs.local/health || true
    echo

    echo "----- Recent Nginx Errors -----"
    tail -50 /var/log/nginx/rustfs-s3-error.log 2>/dev/null || true
    EOF

Grant execution permissions:

    chmod +x /usr/local/bin/rustfs-entry-check.sh

Run:

    /usr/local/bin/rustfs-entry-check.sh

---

## 25. Fault Record Template

| Item | Content |
|---|---|
| Fault Time |  |
| Discovery Method | Alert / User Feedback / Patrol |
| Affected Business |  |
| Affected Bucket |  |
| Endpoint |  |
| Affects Upload | Yes / No |
| Affects Download | Yes / No |
| Affects Console | Yes / No |
| Error Code | 403 / 413 / 502 / 504 / SignatureDoesNotMatch |
| Affected Node |  |
| Nginx Log |  |
| RustFS Log |  |
| Disk Capacity |  |
| Node Resources |  |
| Recent Changes |  |
| Preliminary Judgment |  |
| Handling Actions |  |
| Recovery Time |  |
| Root Cause |  |
| Subsequent Improvements |  |

---

## 26. High-Risk Operation Reminder

The following operations must be handled with caution in production environments:

    rm -rf /data/rustfs*
    docker rm -f rustfs-cluster
    Restart all RustFS nodes simultaneously
    Modify RUSTFS_VOLUMES
    Modify AccessKey / SecretKey
    Delete Bucket
    Batch delete objects
    Clear backup Bucket
    Format data disk
    Modify /etc/fstab
    Modify Nginx upstream
    Modify certificate
    Modify DNS / hosts
    Switch Endpoint
    Disable certificate verification
    Directly manually modify RustFS internal data directory

Before execution, ensure:

    Whether it is a production environment.
    Whether there is a maintenance window.
    Whether there is a backup.
    Whether there is a recovery plan.
    Whether there is business confirmation.
    Whether there is a rollback plan.
    Whether the on-site logs are preserved.
    Whether it has been reviewed.

---

## 27. Experiment Cleanup

### 27.1 Delete Inspection Test Objects

Run:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force rustfs-ops/prod-app-uploads/ops

---

### 27.2 Delete Local Test Files

Run:

    rm -f /tmp/rustfs-daily-check.txt
    rm -f /tmp/rustfs-daily-check-download.txt
    rm -f /tmp/rustfs-daily-check-after-node-stop.txt
    rm -f /tmp/node04-recovery-check.txt
    rm -f /tmp/rustfs-large-test.bin
    rm -f /tmp/rustfs-large-test.sha256

---

### 27.3 Delete Inspection Scripts

If no longer needed:

    rm -f /usr/local/bin/rustfs-node-check.sh
    rm -f /usr/local/bin/rustfs-entry-check.sh

Production recommendation:

    Inspection scripts can be retained.
    But should be included in version management and change review.
    Do not write sensitive keys into inspection scripts.

---

### 27.4 Delete mc Configuration

If continuing experiments or comparisons, recommend retaining:

    /data/rustfs-ops/mc-config

If confirmed not needed:

    rm -rf /data/rustfs-ops/mc-config

---

## 28. Completion Criteria for This Article's Hands-On Practice

After completing this article, the following should be at least achieved:

| Item | Standard |
|---|---|
| Container Inspection | Can check RustFS container status |
| Log Viewing | Can view docker logs |
| Health Check | Can access /health |
| Nginx Inspection | Can view Nginx status and logs |
| Certificate Check | Can view certificate validity |
| Bucket Inspection | Can use mc ls to view Bucket |
| Object Verification | Can upload/download inspection objects |
| Capacity Check | Can check df / du |
| Disk I/O | Can use iostat to determine disk pressure |
| Network Check | Can use sar / iftop to observe traffic |
| Permission Troubleshooting | Can troubleshoot AccessDenied |
| Signature Troubleshooting | Can troubleshoot SignatureDoesNotMatch |
| Entry Troubleshooting | Can troubleshoot 502 / 413 / 504 |
| Node Abnormality | Can simulate and recover single node abnormality |
| Node Replacement | Can clearly explain key points of node replacement |
| Monitoring Planning | Can design basic monitoring and alerts |
| Fault Record | Can fill out production fault template |

---

## Twenty-Nine, Interview Answering Approach

If asked in an interview:

    How do you routinely inspect RustFS object storage? How do you troubleshoot when a fault occurs?

You can answer:

    I divide RustFS operations and troubleshooting into five layers: entry layer, service layer, storage layer, permission layer, and client layer. Routine inspections won't only check docker ps, but will also inspect RustFS /health, Nginx status, whether Bucket can be listed, test object upload/download, data directory capacity, node disk I/O, Nginx 4xx/5xx, certificate validity, and RustFS logs.
    Entry layer mainly checks DNS, HTTPS, Nginx, certificates, and upstream. If 502 occurs, I'll first check Nginx error.log, then curl each backend RustFS node's /health to confirm if it's an entry issue or backend node issue. If 413 occurs, it's usually client_max_body_size limit; if 504 occurs, focus on proxy_read_timeout, proxy_send_timeout, backend disk I/O, and network status.
    Service layer mainly checks RustFS container status, docker logs, 9000/9001 ports, node time synchronization, and startup parameters. If container fails to start, I'll check if data directory exists, permissions are UID 10001, port conflicts, RUSTFS_VOLUMES consistency, hostname resolution, and image version consistency.
    Storage layer mainly checks df -hT, du -sh, iostat, dmesg, and system disk capacity. Object storage is most vulnerable to data disk or system disk fullness. Data directory fullness affects uploads, Docker or Nginx logs filling system disk also causes service anomalies.
    Permission layer mainly troubleshoots AccessDenied and SignatureDoesNotMatch. AccessDenied usually checks business AccessKey permissions for Bucket and actions; SignatureDoesNotMatch usually checks SecretKey, time synchronization, Region, Endpoint, Host header, Path-style, and HTTP/HTTPS consistency.
    When node abnormality occurs, I'll first confirm the impact scope and won't directly delete data directory. For single node abnormality, I can first isolate or recover the node, check if Nginx can still forward to other backends. After node recovery, check RustFS logs to confirm if it rejoins the cluster and triggers data repair. When replacing nodes, keep hostname, version, startup parameters, data directory, keys, and time synchronization consistent.
    In production, also integrate Prometheus, Node Exporter, Nginx Exporter, and black-box probing, with alerts for /health, disk capacity, request error rate, certificate expiration, node status, and large object deletion.

---

## Thirty, Summary of This Article

This article completes RustFS operations and troubleshooting learning:

1. RustFS operations cannot only check if containers are Running.
2. Routine inspections must cover API, Nginx, Bucket, Object, capacity, logs, and certificates.
3. /health is an important entry for RustFS API health check.
4. docker logs is the first entry for container fault troubleshooting.
5. Nginx access.log and error.log are core for entry layer troubleshooting.
6. AccessDenied mainly troubleshoots permissions, Bucket, and AccessKey.
7. SignatureDoesNotMatch mainly troubleshoots keys, time, Endpoint, Host, and path style.
8. 502 mainly troubleshoots Nginx upstream and RustFS backend.
9. 413 mainly troubleshoots upload size limits.
10. 504 mainly troubleshoots timeouts, backend performance, and network.
11. Large object uploads need attention to Nginx buffering, timeouts, disk capacity, and network stability.
12. Data directory permission anomalies commonly occur when UID 10001 has no write permissions.
13. Node recovery needs attention to version, hostname, resolution, startup parameters, and data directory.
14. Node replacement is not simply starting a new container.
15. Do not manually modify or delete RustFS internal data directory.
16. Capacity alerts are production baseline for object storage.
17. Certificate expiration alerts are production baseline for HTTPS entry.
18. RustFS can plan Prometheus metrics, health checks, and log auditing.
19. Production faults must retain on-site logs and change records.
20. Next article will enter RustFS phase summary: positioning and practice boundaries of new object storage.

---

## Thirty-One, Reference Documents

RustFS official website:

    https://rustfs.com/

RustFS official documentation:

    https://docs.rustfs.com/

RustFS Logging and Auditing:

    https://docs.rustfs.com/features/logging

RustFS Node Failure Troubleshooting:

    https://docs.rustfs.com/troubleshooting/node.html

RustFS Docker Installation Documentation:

    https://docs.rustfs.com/installation/docker/

RustFS Multi-node Multi-disk Installation Documentation:

    https://docs.rustfs.com/installation/linux/multiple-node-multiple-disk.html

RustFS Security Checklist:

    https://docs.rustfs.com/installation/checklists/security-checklists.html

RustFS Nginx Reverse Proxy Configuration:

    https://docs.rustfs.com/integration/nginx-reverse-proxy-configuration/

RustFS TLS Configuration:

    https://docs.rustfs.com/integration/tls-configured.html

RustFS mc Client:

    https://docs.rustfs.com/developer/mc.html

MinIO mc Client Documentation:

    https://min.io/docs/minio/linux/reference/minio-mc.html

AWS S3 API:

    https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html

AWS CLI S3:

    https://docs.aws.amazon.com/cli/latest/reference/s3/

Nginx official documentation:

    https://nginx.org/en/docs/

Docker official documentation:

    https://docs.docker.com/

Prometheus official documentation:

    https://prometheus.io/docs/introduction/overview/

Grafana official documentation:

    https://grafana.com/docs/