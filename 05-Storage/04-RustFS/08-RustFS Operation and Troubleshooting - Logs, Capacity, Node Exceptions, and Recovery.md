# RustFS Operation and Troubleshooting: Logs, Capacity, Node Exceptions, and Recovery

Recommended path: 05-Storage/04-RustFS/08-RustFS Operation and Troubleshooting: Logs, Capacity, Node Exceptions, and Recovery.md

Tags: #RustFS #Object Storage #S3 #Operation and Maintenance Inspection #Fault Troubleshooting #Log Analysis #Capacity Management #Node Exception #Data Recovery #Nginx #Docker #Advanced SRE #Production Operation and Maintenance

---

## I. Document Description

This article is the eighth in the RustFS module, focusing on the daily operation and maintenance of RustFS, log viewing, capacity management, node exceptions, client access exceptions, Nginx reverse proxy exceptions, and recovery methods.

What has been covered before:

    01-RustFS Basics: S3-compatible Object Storage and Use Cases
    02-RustFS Deployment Modes: Understanding Single-machine and Cluster Modes
    03-RustFS Single-machine Deployment Practice: Service Startup, Data Directory, and Access Verification
    04-RustFS Cluster Deployment Practice: Multiple Nodes, Multiple Disks, and Access Entries
    05-RustFS vs. MinIO: Architectural, Deployment, Ecosystem, and Operation and Maintenance Differences
    06-RustFS Client Access: S3 API, mc Tool, and Application Integration
    07-RustFS Permissions and Security: Access Keys, HTTPS, and Reverse Proxy

This article focuses on:

    What to check during daily RustFS inspections
    How to view RustFS container logs
    How to check the health status of the RustFS API
    How to troubleshoot RustFS Console access exceptions
    How to inspect RustFS Buckets and Objects
    How to check the capacity of RustFS data directories
    How to handle insufficient disk space on RustFS nodes
    How to determine if a single RustFS node is abnormal
    How to verify a recovered RustFS node
    How to troubleshoot abnormal data directory permissions in RustFS
    How to resolve AccessDenied issues for RustFS clients
    How to address SignatureDoesNotMatch errors in RustFS
    How to troubleshoot Nginx 502, 413, and 504 errors in RustFS
    How to diagnose failures in large object uploads in RustFS
    How to record production faults in RustFS
    How to establish a production inspection checklist for RustFS

This article emphasizes:

    Object storage operation and maintenance should not rely solely on whether the container is running.
    It is necessary to check the API, Buckets, Objects, capacity, logs, reverse proxy, permissions, certificates, client access, and node health simultaneously.
    As a new type of object storage solution, RustFS must undergo thorough fault drills, monitoring alerts, and recovery verifications before being put into production use.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Establish a routine inspection approach for RustFS.
2. View the status of RustFS Docker containers.
3. Analyze RustFS container logs.
4. Check the health status of the RustFS API.
5. Verify the health of the Nginx entry points.
6. Use the mc tool to inspect Buckets and Objects.
7. Check the capacity of data directories.
8. Evaluate the status of node disks and file systems.
9. Troubleshoot container startup failures.
10. Identify port conflicts.
11. Resolve data directory permission issues.
12. Address AccessDenied errors.
13. Diagnose SignatureDoesNotMatch problems.
14. Resolve EndpointConnectionError issues.
15. Fix Nginx 502, 413, and 504 errors.
16. Investigate failures in large object uploads.
17. Simulate single-node exceptions and observe their effects.
18. Recover abnormal nodes and verify service functionality.
19. Understand the key steps involved in node replacement.
20. Develop production inspection, alert, and fault recording templates.

---

## III. Key Conclusions

### 3.1 RustFS operation and maintenance should be conducted layer by layer from the entry points to the data directories

Recommended troubleshooting sequence:

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

Do not only rely on:

    docker ps

Also check:

    curl /health
    mc ls
    docker logs
    Nginx access.log
    Nginx## Section 5: Daily Inspection Overview

### 5.1 Daily Inspection Items

Items to check daily include:

- Whether the RustFS container is running.
- Whether the RustFS API is functioning properly.
- Whether the RustFS Console is accessible.
- Whether Nginx is running.
- Whether the HTTPS certificate is valid.
- Whether buckets can be listed.
- Whether file uploads and downloads are working correctly.
- Whether the data directory capacity is normal.
- Whether there are any issues with node disks.
- Whether node time is synchronized.
- Whether Nginx is experiencing many 4xx/5xx errors.
- Whether there are any abnormalities in RustFS logs.
- Whether clients are reporting AccessDenied or SignatureDoesNotMatch errors.

---

### 5.2 Weekly Inspection Items

Items to check weekly include:

- The number of buckets.
- The growth trend of the capacity of important buckets.
- The number of large objects.
- Whether backup buckets are continuously increasing in size.
- Whether log archives need to be cleaned up.
- Whether there are any abnormal AccessKeys.
- Whether there are any keys that have not been rotated for a long time.
- The size of Nginx logs.
- The size of RustFS logs.
- The SMART status of node disks.
- Node load and network bandwidth usage.
- Whether there are any requests that have failed repeatedly.

---

### 5.3 Monthly Inspection Items

Items to check monthly include:

- Conduct fault drills.
- Perform node restart drills.
- Test the recovery process in case of a single-node failure.
- Check the certificate expiration dates.
- Review the rotation of AccessKeys.
- Recheck the principle of least privilege settings.
- Review the bucket lifecycle management policies.
- Evaluate capacity expansion plans.
- Assess version upgrade requirements.
- Recheck the recovery procedures.
- Test the effectiveness of monitoring and alarm systems.

---

## Section 6: Basic Inspection Commands

### 6.1 Checking Node Status

Execute these commands on all RustFS nodes:

    hostname
    hostname -I
    uptime
    date
    timedatectl
    df -hT
    free -h
    top -b -n 1 | head -20

Pay special attention to:

- Whether the node is online.
- Whether time is synchronized.
- Whether disk space is nearly full.
- Whether there are any memory issues.
- Whether the load is abnormal.

---

### 6.2 Checking Docker Status

Execute these commands:

    systemctl status docker --no-pager
    docker version
    docker ps
    docker ps -a

To check RustFS containers:

    docker ps | grep rustfs
    docker ps -a | grep rustfs

In single-node mode:

    docker ps | grep rustfs-single

In cluster mode:

    docker ps | grep rustfs-cluster

---

### 6.3 Checking RustFS Logs

In single-node mode:

    docker logs rustfs-single --tail=100

In cluster mode:

    docker logs rustfs-cluster --tail=100

For continuous monitoring:

    docker logs -f rustfs-cluster

To view recent errors:

    docker logs rustfs-cluster --since "30m" | grep -Ei "error|failed|denied|panic|timeout|refused|permission"

Note:

- Docker logs are the most direct way to troubleshoot service issues.
- If a container keeps restarting, check the Docker logs first. Do not rely solely on docker ps.

---

### 6.4 Checking Port Listening

Execute these commands on RustFS nodes:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

On the entry node:

    ss -lntp | grep ':80'
    ss -lntp | grep ':443'

Note:

- Port 9000 is used for the S3 API.
- Port 9001 is used for the Console.
- Ports 80 and 443 are used for Nginx. If these ports are not listening, clients will not be able to access the services.

---

### 6.5 Checking API Health Status

For backend nodes:

    curl -i http://10.0.0.51:9000/health
    curl -i http://10.0.0.52:9000/health
    curl -i http://10.0.0.53:9000/health
    curl -i http://10.0.0.54:9000/health

For the unified entry point:

    curl -k -i https://s3.rustfs.local/health

Expected response:

- HTTP 200

If the backend is normal but the unified entry point fails:

- First, check Nginx, certificates, DNS, upstream servers, and```markdown
-v /data/rustfs-ops/mc-config:/root/.mc \
registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
--insecure stat rustfs-ops/prod-app-uploads/hello.txt

---

### 8.4 Upload and Download Inspection Objects

Create the inspection file:

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

Verify:

    diff /tmp/rustfs-daily-check.txt /tmp/rustfs-daily-check-download.txt

If the diff command returns no results, it means that the uploaded and downloaded content are identical.

---
## Section 9: Capacity Inspection

### 9.1 Check Node Disk Capacity

Perform this on each RustFS node:

    df -hT

Focus on checking:

    /data/rustfs0
    /data/rustfs1
    /data/rustfs2
    /data/rustfs3

Alternatively, you can execute:

    df -hT /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3
---

### 9.2 Check Data Directory Usage

Run the following command:

    du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

For more detailed directory checks, use:

    du -h --max-depth=1 /data/rustfs0 | sort -h | tail -20
    du -h --max-depth=1 /data/rustfs1 | sort -h | tail -20
    du -h --max-depth=1 /data/rustfs2 | sort -h | tail -20
    du -h --max-depth=1 /data/rustfs3 | sort -h | tail -20

Notes:

    You can view the capacity of each directory.
    Do not manually modify RustFS's internal data files.
    Do not directly delete internal object data directories.
    Object deletion should be done through the S3 API, mc, or Console.

---
### 9.3 Check System Disk Capacity

Run these commands:

    df -hT /
    du -h --max-depth=1 /var | sort -h | tail -20
    du -h --max-depth=1 /var/lib/docker | sort -h | tail -20

Reasons:

    Docker logs may fill up the system disk.
    Nginx logs may fill up the system disk.
    A full system disk can affect Docker, Nginx, and other system services.

---
### 9.4 Check Docker Log Size

Run the following command:

    docker inspect rustfs-cluster --format='{{.LogPath}}'

Assume the output is:

    /var/lib/docker/containers/<container-id>/<container-id>-json.log

To check the size, use:

    ls -lh /var/lib/docker/containers/*/*-json.log | sort -k5 -h | tail -20

Production recommendations:

    Configure Docker's log options.
    Limit the size of individual container logs.
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

Before restarting Docker, evaluate the potential impact:

    systemctl restart docker

High-risk warning:

    Restarting Docker will affect all containers on the current node.
    This should only be done during scheduled maintenance periods.

---
### 9| connection refused | Other nodes are not started or ports are blocked | Check the network and firewall settings. |
| version mismatch | Node versions are inconsistent | Use a unified image version. |
| secret mismatch | Key pairs are different | Set the AccessKey/SecretKey to the same value. |

---

### 11.4 Example of Permission Fixing

Command to execute:

    ls -ld /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Command to fix the issue:

    chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3

Command to restart the service:

    docker restart rustfs-cluster

Command to check the logs:

    docker logs rustfs-cluster --tail=100

---

## Section Twelve: Practical Example Two: Port Conflict Troubleshooting

### 12.1 Checking Port Availability

Command to execute:

    ss -lntp | grep ':9000'
    ss -lntp | grep ':9001'

Command to check running processes:

    ps -ef | grep <PID>

---

### 12.2 Handling Strategies

If port 9000 is already in use:

    Verify if there are any RustFS containers running.
    Check if they are MinIO or other object storage services.
    Confirm if they are from previous testing sessions.
    Do not terminate unfamiliar processes.

If it's an old container:

    Use the command `docker ps -a` to identify it.
    Then remove it using `docker rm -f <container_name>`.

If you need to change the port:

    Ensure that all RustFS nodes use the same port number.
    Update the configuration in Nginx upstream services.
    Also, adjust the settings for the mc/SDK endpoints accordingly.

---

## Section Thirteen: Practical Example Three: AccessDenied Issues Troubleshooting

### 13.1 Symptoms

The client receives error messages such as:

    AccessDenied
    403 Forbidden
    Access Denied

---

### 13.2 Troubleshooting Steps

Step 1: Verify the alias configuration:

    Run the following command:
    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure alias list

Step 2: Check if the administrator account can access resources:

    Run the following command:
    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-ops

Step 3: Verify if the business account can only access the intended bucket:

    Run the following command:
    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-app/prod-app-uploads

Step 4: Try accessing an unrelated bucket:

    Run the following command:
    docker run --rm \
      -v /data/rustfs-security/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure ls rustfs-app/prod-backups

---

### 13.3 Possible Causes and Solutions

If only the administrator account can access resources:

    The issue is likely with the business account's AccessKey permissions.
    Check the associated policies and bucket settings.
    Verify if the correct actions are allowed for that account.

If no account can access resources:

    There might be issues with the service, entry points, Nginx configuration, certificates, or the entire cluster.

If only specific operations fail:

    For example, if `PutObject` fails, it means the account lacks the necessary write permissions.
    Similarly, if `GetObject` fails, it indicates that read permissions are insufficient.
    For `DeleteObject` failures, check for delete permissions.
    `ListBucket` failures suggest that listing rights are not granted.

---

## Section Fourteen: Practical Example Four: SignatureDoesNotMatch Issues Troubleshooting

### 14.1 Symptoms

The client receives anclient_max_body_size 10g;

---

### 16.4 Overloading Configuration

Execute:

    nginx -t
    systemctl reload nginx

Test uploading again.

---

## Section Seventeen: Practical Exercise Seven: Troubleshooting Nginx 504 Errors

### 17.1 Symptoms

When attempting to upload or download large files, the following error occurs:

    504 Gateway Timeout

---

### 17.2 Common Causes

Common reasons include:

    The proxy_read_timeout is set too short.
    The proxy_send_timeout is set too short.
    The RustFS backend responds slowly.
    High I/O operations on the backend disk.
    Network fluctuations at the node level.
    Slow client network connections.
    The time taken to upload large objects exceeds Nginx's timeout limit.

---

### 17.3 Checking Configuration

Execute:

    grep -R "proxy_read_timeout" /etc/nginx/conf.d/
    grep -R "proxy_send_timeout" /etc/nginx/conf.d/
    grep -R "proxy_connect_timeout" /etc/nginx/conf.d/

Recommendations:

    Set proxy_connect_timeout to 60 seconds.
    Set proxy_send_timeout to 3600 seconds.
    Set proxy_read_timeout to 3600 seconds.
    Set send_timeout to 3600 seconds.

Reload the configuration:

    nginx -t
    systemctl reload nginx

---

### 17.4 Checking Backend Resources Simultaneously

On the RustFS node, execute the following commands:

    iostat -x 1 10
    sar -n DEV 1 10
    docker logs rustfs-cluster --tail=100
    df -hT

If the disk or network is already at full capacity, simply increasing Nginx's timeout settings will not resolve the underlying issue.

---

## Section Eighteen: Troubleshooting Large Object Upload Failures

### 18.1 Preparing a Large Test File

On the rustfs-client, execute the following commands:

    dd if=/dev/zero of=/tmp/rustfs-large-test.bin bs=1M count=1024
    sha256sum /tmp/rustfs-large-test.bin > /tmp/rustfs-large-test.sha256

---

### 18.2 Conducting the Upload Test

Execute:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      -v /tmp:/tmpdata \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure cp /tmpdata/rustfs-large-test.bin rustfs-ops/prod-app-uploads/large/rustfs-large-test.bin

---

### 18.3 Troubleshooting Failures

On the client side:

    Check if there are any network interruptions.
    Verify if mc reports errors such as 413, 504, SignatureDoesNotMatch, or AccessDenied.
    
On the Nginx side:

    Check the logs in /var/log/nginx/rustfs-s3-error.log and /var/log/nginx/rustfs-s3-access.log.
    
On the RustFS side:

    View the logs generated by docker logs rustfs-cluster --tail=200.

On the node side:

    Execute the commands iostat -x 1 10 and sar -n DEV 1 10 to check disk and I/O performance.

Common causes include:

    Nginx's upload size limit.
    Nginx timeout errors.
    Insufficient backend disk space.
    High I/O pressure on the backend.
    Unstable network connections.
    Compatibility issues with the client SDK for multipart uploads.
    Certificate or signature-related problems.

---

## Section Nineteen: Single-Node Failure Experiment

### 19.1 Experiment Description

Objective:

    Simulate a failure of one RustFS backend node.
    Verify if the unified entry point is still accessible.
    Check if Buckets and Objects can be accessed normally.
    Verify service functionality after restoring the node.

High-Risk Warning:

    This experiment should only be conducted in a test environment.
    Any production-level fault recovery exercises must be approved in advance.
    Ensure that backup, recovery plans, and maintenance windows are established before proceeding.

---

### 19.2 Shutting Down a Node

On the rustfs-node04 node, execute:

    docker stop rustfs-cluster

Verify the status of the nodes using:

    docker ps -a | grep rustfs-cluster

---

### 19.3 Client Access Verification

On the rustfs-client, execute:

    curl -k -i https://s3.rustfs.local/health

To check Buckets, execute:

    docker run --14. Record the recovery time and scope of impact.

---

### 20.4 Prohibited Operations When Replacing Nodes

Prohibited:

    Directly adding nodes using images of different versions.
    Using different RUSTFS_VOLUMES.
    Using different hostnames without updating DNS resolutions.
    Using different AccessKey/SecretKey pairs.
    Directly modifying the internal data directories.
    Arbitrarily copying residual data from old nodes to new ones.
    Manually repairing internal files without understanding the data structure.
    Replacing multiple nodes simultaneously.

---

## Section 21: Disk Exception Handling

### 21.1 Insufficient Disk Space

Symptoms:

    Upload failures
    500/503 errors
    Container logs indicating insufficient space
    `df -hT` showing 100% disk usage
    Nginx may report 502/504 errors

Inspection Steps:

    Check `df -hT /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3`
    Check `du -sh /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3`

Action Steps:

    Identify which bucket is experiencing abnormal growth.
    Remove unnecessary objects and old backups.
    Expand the disk capacity if needed.
    Add more nodes if required.
    Adjust the lifecycle management settings.
    Do not manually delete internal RustFS data files.

---

### 21.2 Disk I/O Abnormalities

Inspection Steps:

    Run `iostat -x 1 10`
    Check `dmesg | tail -100`
    View `journalctl -k --since "1 hour ago" | tail -100`

Key Indicators to Watch:

    `await`, `%util`, disk errors, I/O errors, filesystem errors

Action Steps:

    Determine if there is a hard drive failure.
    Check if the system is in recovery mode or undergoing healing processes.
    Assess if there are peak periods for large object uploads.
    Verify if any other processes are consuming excessive disk resources.
    Prepare to replace faulty nodes or disks if necessary.

---

### 21.3 Abnormal Permissions on Data Directories

Symptoms:

    `Permission denied` errors
    Container startup failures
    Object writing failures

Inspection Steps:

    Check `ls -ld /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3`
    View `docker logs rustfs-cluster --tail=200`

Action Steps:

    Change directory permissions to `chown -R 10001:10001 /data/rustfs0 /data/rustfs1 /data/rustfs2 /data/rustfs3`
    Restart the `rustfs-cluster` container.

---

## Section 22: Monitoring and Alerting Recommendations

### 22.1 Essential Metrics to Monitor

Must monitor:

    RustFS API health status
    RustFS container status
    Ports 9000/9001 of RustFS
    Nginx ports 80/443
    Nginx errors 502/504/413
    Data disk capacity
    System disk capacity
    Node CPU/memory usage
    Node disk I/O performance
    Node network traffic
    Certificate expiration date
    Increase in `AccessDenied` errors
    Increase in `SignatureDoesNotMatch` errors
    Frequent `DeleteObject` operations
    Sudden increases in bucket capacity

---

### 22.2 Using Prometheus for Monitoring

RustFS officially supports Prometheus-compatible metrics and health check endpoints.

Production recommendations:

    Use Prometheus to collect RustFS metrics.
    Use Prometheus blackbox probes for health checks.
    Use Node Exporters to monitor host resources.
    Use Nginx Exporters to monitor entry points.
    Use Grafana to visualize capacity, request volume, error rates, and latency.
    Use Alertmanager to send alerts.

Basic monitoring architecture:

    Prometheus
      |
      +--> RustFS Metrics
      +--> RustFS health checks
      +--> Node Exporters
      +--> Nginx Exporters
      +--> Blackbox Exporters

---

### 22.3 Alarm Classification

| Alarm          | Level   | Description                                      |
|-----------------|-------|---------------------------------------------------------|
| RustFS health check failure | Critical | Object storage may be unavailable.                |
| Multiple backend nodes down       | Critical | High-risk for the entire cluster.                  |
| Single backend node down        | Warning | Requires immediate recovery.                        |
| Data disk usage > 90%      | Critical | May affect writing operations.                         |
| Data disk usage > 80%      | Warning | Capacity expansion is needed.                            docker logs rustfs-single --tail=30 || true
    else
      echo "No rustfs container found"
    fi
    echo

    echo "----- Health -----"
    curl -sS -i http://127.0.0.1:9000/health || true
    echo
    EOF

Authorization:

    chmod +x /usr/local/bin/rustfs-node-check.sh

Execution:

    /usr/local/bin/rustfs-node-check.sh

---

### 24.3 Entrance Inspection Script

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

Authorization:

    chmod +x /usr/local/bin/rustfs-entry-check.sh

Execution:

    /usr/local/bin/rustfs-entry-check.sh

---

## Twenty-Five, Fault Record Template

| Item | Content |
|---|---|
| Fault Time |  |
| Discovery Method | Alarm / User Feedback / Inspection |
| Business Impact |  |
| Bucket Affected |  |
| Endpoint |  |
| Affects Upload | Yes / No |
| Affects Download | Yes / No |
| Affects Console | Yes / No |
| Error Code | 403 / 413 / 502 / 504 / SignatureDoesNotMatch |
| Affected Nodes |  |
| Nginx Logs |  |
| RustFS Logs |  |
| Disk Capacity |  |
| Node Resources |  |
| Recent Changes |  |
| Preliminary Assessment |  |
| Action Taken |  |
| Recovery Time |  |
| Root Cause |  |
| Follow-up Improvements |  |

---

## Twenty-Six, High-Risk Operation Warnings

The following operations must be performed with caution in a production environment:

    rm -rf /data/rustfs*
    docker rm -f rustfs-cluster
    Restart all RustFS nodes simultaneously
    Modify RUSTFS_VOLUMES
    Change AccessKey/SecretKey
    Delete a Bucket
    Batch delete objects
    Clear backup Buckets
    Format data disks
    Modify /etc/fstab
    Adjust Nginx upstream settings
    Update certificates
    Change DNS/hosts configuration
    Switch Endpoints
    Disable certificate verification
    Manually modify RustFS internal data directories

Before executing any of these operations, it is essential to confirm:

    Whether it is a production environment.
    If there is a maintenance window available.
    If backups have been made.
    If a recovery plan exists.
    If business stakeholders have given approval.
    If a rollback strategy is in place.
    If current logs will be retained.
    If the operations have been reviewed.

---

## Twenty-Seven, Experiment Cleanup

### 27.1 Delete Inspection Test Objects

Execute:

    docker run --rm \
      -v /data/rustfs-ops/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure rm --recursive --force rustfs-ops/prod-app-uploads/ops

---

### 2The storage layer mainly involves checking df -hT, du -sh, iostat, dmesg, and the capacity of the system disk. Object storage is particularly vulnerable to situations where the data disk or system disk becomes full; a full data directory can affect uploads, and when Docker or Nginx logs fill up the system disk, it can lead to service disruptions.

For permission issues, the main focus should be on AccessDenied and SignatureDoesNotMatch. AccessDenied usually indicates whether the business AccessKey has the appropriate permissions for the Bucket and the intended actions; SignatureDoesNotMatch typically involves checking whether the SecretKey, time synchronization, Region, Endpoint, Host header, Path-style, and HTTP/HTTPS settings are consistent.

In the event of a node failure, I would first determine the scope of the impact before directly deleting any data directories. For a single-node issue, the node can be removed or restored to see if Nginx can still forward requests to other backend servers. After restoring the node, it is necessary to check the RustFS logs to confirm whether it has rejoinned the cluster and whether data restoration processes have been triggered. When replacing a node, it is important to ensure that the hostname, version, startup parameters, data directory, keys, and time synchronization settings remain consistent.

In production environments, it is also essential to integrate tools such as Prometheus, Node Exporter, Nginx Exporter, and black-box monitoring systems. Alerts should be set up for indicators like /health checks, disk capacity, request error rates, certificate expiration, node status, and large-scale object deletions.