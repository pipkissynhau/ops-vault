# MinIO Monitoring and Operations: Prometheus Metrics, Logging, and Capacity Management

Recommended Path: 05-Storage/02-MinIO/09-MinIO Monitoring and Operations: Prometheus Metrics, Logging, and Capacity Management.md

Tags: #MinIO #Monitoring and Operations #Prometheus #Grafana #Alertmanager #Object Storage #S3 #Capacity Management #Log Analysis #Advanced SRE #Production Operations

---

## I. Document Overview

This article is the ninth in the MinIO series, focusing on monitoring, logging, capacity management, and daily operations for MinIO.

Previous topics include:

- Basics of MinIO Object Storage
- Single-machine single-disk deployment
- Single-node multi-disk deployment
- 4-node multi-disk distributed cluster deployment
- Internal HTTP and external HTTPS entry design
- Nginx HTTPS unified entry point
- mc client configuration and object operations
- User, Policy, AccessKey, and SecretKey permission management
- Erasure Coding, node failure, and disk failure recovery

This article delves into the core aspects of production operations:

- How to determine if a MinIO cluster is healthy?
- Which metrics should be checked daily?
- How to monitor MinIO nodes and disk status?
- How to check Bucket capacity?
- How to count the number of objects?
- How to collect MinIO metrics using Prometheus?
- Which dashboards should be monitored in Grafana?
- What issues should be alarmed by Alertmanager?
- How can Nginx logs assist in troubleshooting?
- How can Docker logs help with debugging?
- How to plan capacity levels?
- How to detect abnormal growth in object storage usage?
- How to establish an advanced SRE framework for MinIO operations?

This article emphasizes practicality, providing commands for all the checks mentioned.

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Use `mc admin info` to check MinIO cluster status.
2. Use the `health` interface to check the live/ready status of MinIO nodes.
3. Use `mc du` to view Bucket or Prefix capacity.
4. Use `mc find` to inspect object distribution.
5. View MinIO service logs using Docker logs.
6. Troubleshoot entry issues with Nginx access.log/error.log.
7. Understand how Prometheus collects MinIO metrics.
8. Generate or configure Prometheus to collect MinIO metrics.
9. Plan MinIO Grafana monitoring dashboards.
10. Set up alerts for nodes, disks, capacity, APIs, certificates, and backups.
11. Create a daily MinIO inspection checklist.
12. Identify abnormal increases in Bucket capacity.
13. Detect potential high-risk operations in object storage.
14. Establish a baseline for MinIO production operations.

---

## III. Experimental Environment

### 3.1 MinIO Cluster Nodes

This article continues with the previously configured 4-node distributed MinIO cluster:

| IP | Host Name | Role | Port |
|---|---|---|---|
| 10.0.0.41 | minio-node01 | MinIO Node 1 | 9000 / 9001 |
| 10.0.0.42 | minio-node02 | MinIO Node 2 | 9000 / 9001 |
| 10.0.0.43 | minio-node03 | MinIO Node 3 | 9000 / 9001 |
| 10.0.0.44 | minio-node04 | MinIO Node 4 | 9000 / 9001 |
| 10.0.0.45 | minio-client | mc Client/Operations Management Node | - |
| 10.0.0.46 | minio-entry | Nginx HTTPS Unified Entry Point | 80 / 443 |

---

### 3.2 Access Entrypoints

Backend direct access:

    http://10.0.0.41:9000

HTTPS unified entry point:

    https://s3.minio.local

Console entry point:

    https://console.minio.local

This article uses the default entry point:

    https://s3.minio.local

If a self-signed certificate is used in the experimental environment, temporarily use `--insecure` in the mc command:

    --insecure

Do not use `--insecure` in a production environment.

---

### 3.3 Image Versions

MinIO server image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z

mc client image:

    registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2        -v ${MC_WORKDIR}:/demo \
        ${MC_IMAGE} "$@"
    }

Example of self-signed certificate environment:

    mcx --insecure admin info minio-admin

Note:

    This function is only valid within the current shell session.
    Formal documentation still includes the complete docker run command for easy replication and reuse.

---

## Section 6: Basic Health Checks

### 6.1 Viewing MinIO Cluster Information

Execute:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin info minio-admin

Key points to check:

    Whether the cluster is accessible.
    Whether nodes are online.
    Whether disks are online.
    If there are any offline drives.
    If there is any healing activity.
    Disk usage.
    Any warnings.
    Whether versions are consistent.

---

### 6.2 Checking the live Interface

Execute in minio-client or minio-entry:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check live $ip"
      curl -I http://$ip:9000/minio/health/live
    done

Explanation:

    The "live" check determines whether the MinIO process is running.
    A normal "live" status does not necessarily mean the entire cluster is writable.
    It is also necessary to check the "ready" and "admin info" status.

---

### 6.3 Checking the ready Interface

Execute:

    for ip in 10.0.0.41 10.0.0.42 10.0.0.43 10.0.0.44; do
      echo "check ready $ip"
      curl -I http://$ip:9000/minio/health/ready
    done

Explanation:

    The "ready" status is more suitable for determining whether MinIO is ready to provide services.
    In production health checks, it is recommended to focus on the "ready" status first.
    If the "ready" status is abnormal, it is necessary to combine it with "admin info" and log analysis for troubleshooting.

---

### 6.4 Checking Health via HTTPS Entry

If a Nginx HTTPS unified entry point is configured:

    curl -k -I https://s3.minio.local/minio/health/live
    curl -k -I https://s3.minio.local/minio/health/ready

If it uses a formal, trusted certificate, remove the "-k" option:

    curl -I https://s3.minio.local/minio/health/live
    curl -I https://s3.minio.local/minio/health/ready

---

### 6.5 Checking via Console Entry

Access via browser:

    https://console.minio.local

Or execute a command to check:

    curl -k -I https://console.minio.local

Explanation:

    A normal Console status does not necessarily mean the S3 API is functioning properly.
    Similarly, a normal S3 API status does not guarantee that the Console is working correctly.
    The two use different ports, entry points, and purposes.

---

## Section 7: Node and Container Operations and Maintenance Checks

### 7.1 Checking Docker Container Status

Execute on each of the 4 MinIO nodes:

    docker ps | grep minio

If no containers are running:

    docker ps -a | grep minio

View status details:

    docker inspect minio --format '{{.State.Status}}'
    docker inspect minio --format('{{.State.StartedAt}}'
    docker inspect minio --format('{{.RestartCount}}'

---

### 7.2 Viewing MinIO Container Logs

Execute on each node:

    docker logs --tail=100 minio

View in real-time:

    docker logs -f minio

Filter by time:

    docker logs --since 30m minio
    docker logs --since 2h minio

Common logs to monitor include:

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

### 7.3 Checking Container Resource Usage

Execute on each node:

    docker stats minio

Pay attention to:

    CPU usage.
    Memory usage.
    Network traffic.
    Block I/O operations.
    Any abnormal sustained increases.

---

### 7.4 Checking the Docker Service

    systemctl status docker
   ```markdown
docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-admin/policy-demo/app01/

---

### 9.4 Searching for Bucket Objects

Recursively list objects:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-admin/app-uploads

Search by file name:

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

### 9.5 Recording Bucket Capacity Monitoring

You can save the capacity output to a monitoring file:

    mkdir -p /tmp/minio-monitor-demo/reports

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-monitor-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure du minio-admin/app-uploads > /tmp/minio-monitor-demo/reports/app-uploads-du.txt

View the results:

    cat /tmp/minio-monitor-demo/reports/app-uploads-du.txt

---

### 9.6 Principles of Capacity Management

In production, you should pay attention to:

    The capacity of each Bucket.
    The number of objects in each Bucket.
    The growth rate of each business.
    The presence of exceptionally large objects.
    The existence of a large number of small objects.
    The presence of unassigned Buckets.
    Whether lifecycle cleanup is needed.
    Whether quota restrictions are required.
    Whether cold data migration is necessary.
    Whether cross-cluster backup is needed.

Object storage capacity cannot be assessed solely by `df -h`.

You also need to consider:

    Total cluster capacity.
    Capacity per node.
    Capacity per disk.
    Bucket capacity.
    Prefix capacity.
    Business growth trends.
---

## Section X: Number of Objects and Risks of Small Objects

### 10.1 Why Small Objects Require Attention

A large number of small objects may lead to:

    Increased metadata management pressure.
    Slower object listing.
    Slower backup and migration processes.
    Slower mirror synchronization.
    Delayed deletion operations.
    Increased difficulty in operational troubleshooting.
    Higher monitoring and statistical burdens.

---

### 10.2 Viewing the Object List

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-admin/app-uploads

---

### 10.3 Roughly Counting the Number of Objects

If the number of objects is not large, you can count them using this method:

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure find minio-admin/app-uploads | wc -l

Note:

    Frequent full-scale searches are not recommended for large production Buckets.
    Performing a full listing on extremely large Buckets may cause performance issues.
    It is advisable to combine this with Prometheus metrics, lifecycle policies, and business statistics.

---

### 10.4 Recommendations for Managing Small Objects

It is suggested to:

    Organize log-type objects by date.
    Set up lifecycle rules for expired objects.
    Establish cleaning rules for temporary objects.
    Conduct early assessments in scenarios involving a large number of/minio/v2/metrics/node  
/minio/v2/metrics/bucket  
/minio/v2/metrics/resource  

The supported paths may vary slightly between different versions.  

It is recommended to first check the current version's supported Prometheus configuration methods using mc:  

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin prometheus --help  

You can also try:  

    docker run --rm \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      admin prometheus generate --help  

---

## Section 13: Generating Prometheus Collection Configurations  

### 13.1 Using mc to Generate Prometheus Configurations  

Run the following command:  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin prometheus generate minio-admin  

Note:  
This command will output a sample Prometheus scrape_config. The output usually includes a bearer_token or authentication-related settings. The format may vary depending on the version; refer to the actual command output for details.  

---

### 13.2 Saving the Generated Results  

Save the results in a file:  

    mkdir -p /tmp/minio-monitor-demo/prometheus  

    docker run --rm \
      -v /data/minio/mc-config:/root/.mc \
      -v /tmp/minio-monitor-demo:/demo \
      registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      --insecure admin prometheus generate minio-admin > /tmp/minio-monitor-demo/prometheus/minio-prometheus.yml  

View the file:  

    cat /tmp/minio-monitor-demo/prometheus/minio-prometheus.yml  

Security Note:  
The generated file may contain a token. Do not submit it to Git or share it in public notes. In production environments, manage tokens carefully.  

---

### 13.3 Example Prometheus Scrape Configuration  

The example configuration is shown below; the actual settings should match those generated by mc:  

    scrape_configs:  
      - job_name: minio-cluster  
        metrics_path: /minio/v2/metrics/cluster  
        scheme: https  
        staticConfigs:  
          - targets:  
              - s3.minio.local  
        bearer_token: <Token generated by mc admin prometheus generate>  
        tls_config:  
          insecure_skip_verify: true  

Note:  
insecure.skip_verify: true is only suitable for experimental purposes with self-signed certificates. In production, use trusted certificates and set this option to false or remove it. The bearer_token is sensitive information; make sure to handle it securely. If collecting data through Nginx, ensure that Nginx allows Prometheus access to the specified path.  

---

### 13.4 Example of Collecting Backend HTTP Metrics  

If Prometheus and the MinIO backend are within the same trusted network, you can directly collect metrics from the backend nodes:  

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
        bearer_token: <Token>  

Recommendations:  
- If you want to simulate the client access scenario, collect data through the HTTPS unified entry point.  
- If you need to monitor each backend node individually, collect data from them separately.  
- In production, it is usually necessary to monitor both the entry point and backend nodes simultaneously.  

---

### 13.5 Testing the Metrics API  

If you have a token, you can test it using curl:  

    curl -k \
      -H "Authorization: Bearer <Token>" \
      https://s3.minio.local/minio/v2/metrics/cluster | head  

For backend HTTP testing:  

    curl \
      -H "Authorization: Bearer <Token>" \
      http://10.0.0.41:9000/minio/v2/metrics/clusterThe metric names may vary across different MinIO versions. Production alert rules must be based on the actual metrics collected. Do not blindly copy alert rules that are not compatible with your version.

---

## Section 15: Grafana Monitoring Dashboard Planning

### 15.1 Recommended Dashboard Types

For MinIO production, it is recommended to have at least the following dashboards:

| Dashboard | Focus Areas |
|---------------|---------------------------|
| MinIO Cluster Overview | Cluster status, capacity, request volume, error rate |
| MinIO Node Dashboard | Node availability, CPU, memory, network, processes |
| MinIO Disk Dashboard | Disk availability, capacity, utilization, I/O |
| MinIO Bucket Dashboard | Bucket capacity, number of objects, growth trend |
| MinIO S3 API Dashboard | Request volume, error rate, latency, method distribution |
| Nginx Inbound Dashboard | 4xx and 5xx errors, request volume, response time |
| Certificate Dashboard | HTTPS certificate validity period |
| Backup Sync Dashboard | Mirror task status, duration, failure count |

---

### 15.2 What the Cluster Overview Dashboard Should Include

It is recommended to include:

- Current cluster capacity.
- Used capacity.
- Available capacity.
- Utilization rate.
- Number of online nodes.
- Number of online disks.
- Number of offline disks.
- S3 request QPS.
- 4xx error rate.
- 5xx error rate.
- API latency.
- Healing status.
- Recent alerts.

---

### 15.3 What the Bucket Dashboard Should Include

It is recommended to include:

- List of Buckets.
- Capacity of each Bucket.
- Number of objects in each Bucket.
- Growth trend of each Bucket.
- Top N largest Buckets.
- Top N fastest-growing Buckets.
- Presence of orphaned Buckets.
- Presence of long-unvisited Buckets.

---

### 15.4 What the API Dashboard Should Include

It is recommended to include:

- Number of GET requests.
- Number of PUT requests.
- Number of DELETE requests.
- Number of LIST requests.
- 4xx and 5xx errors.
- AccessDenied errors.
- SignatureDoesNotMatch errors.
- Request latency P95/P99 values.
- Upload and download throughput.
- Distribution of request sources.

---

### 15.5 Sources for Grafana Panels

Optional methods include:

- Using the officially recommended MinIO dashboards.
- Importing dashboards from the Grafana community.
- Customizing them based on PromQL.
- Combining MinIO metrics with Nginx, Node Exporter, and Docker metrics.

It is advised not to assume that monitoring is complete just by importing one dashboard. Additional indicators such as capacity, error rates, backup, and security alerts should be added according to business requirements.

---

## Section 16: Alertmanager Alarm Design

### 16.1 Alarm Classification

Alarms are recommended to be divided into three categories:

| Category | Meaning | Example |
|-----------|----------|-------------------------|
| Critical | Affects business operations or may lead to data risks | Cluster unavailability, write failures, multiple disks offline |
| Warning | There is a risk but it does not completely disrupt operations | Single node offline, capacity exceeding 80% |
| Info | Trend alerts or inspection reminders | Excessive Bucket growth, insufficient certificate remaining time |

Principles:

- Not all alarms should be classified as critical.
- Alarms should not go unattended for extended periods.
- Critical alarms should not be silenced permanently.
- There must be an action plan for each alarm.

---

### 16.2 Mandatory Alarm Items

Mandatory alarms include:

- Unavailability of the MinIO API.
- Failure to meet health readiness criteria.
- Node offline status.
- Disk offline status.
- Disk capacity exceeding thresholds.
- Cluster capacity exceeding thresholds.
- Increase in S3 5xx error rates.
- Increase in Nginx 502/504 errors.
- Certificate nearing expiration.
- Backup tasks failing.
- Healing processes taking too long to complete.

---

### 16.3 Recommended Alarm Items

Recommended alarms include:

- Abnormal growth in Bucket capacity.
- Abnormal increase in the number of objects.
- Sudden surge in DELETE requests.
- Sharp rise in AccessDenied errors.
- Frequent occurrences of SignatureDoesNotMatch errors.
- Unusual logins to the Console.
- Frequent use by the root user.
- Increase in Nginx 4xx errors.
- Increased failures in large file uploads.

---

### 16.4 Recommended Capacity Thresholds

Recommended capacity thresholds are as follows:

| Utilization Rate | Action Recommendations |
|-------------------|------------------------------------------|
| 70% | Monitor trends closely. |
| 80% | Plan for capacity expansion. |
| 85% | Determine the exact time for expansion. |
| 90% | Treat it as a high---

## Chapter Nineteen: Examples of Inspection Scripts

### 19.1 Script Objectives

Script objectives:

    Automatically display MinIO admin information.
    Check HTTPS health status.
    Verify the health of backend nodes.
    Check local disk capacity.
    Examine recent Nginx errors.
    Save the results to an inspection report file.

---

### 19.2 Example Inspection Script

Create a script:

    cat > /usr/local/bin/minio-daily-check.sh <<'EOF'
    #!/bin/bash

    set -euo pipefail

    DATE=$(date +%F-%H%M%S)
    REPORT_DIR="/var/log/minio-check"
    REPORT_FILE="${REPORT_DIR}/minio-check-${DATE}.log"

    MC_IMAGE="registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z"
    MC_CONFIG="/data/minio/mc-config"

    mkdir -p "${REPORT_DIR"}

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

    echo "Report saved to ${REPORT_FILE}"
    EOF

Authorize the script:

    chmod +x /usr/local/bin/minio-daily-check.sh

Execute the script:

    /usr/local/bin/minio-daily-check.sh

View the report:

    ls -lh /var/log/minio-check/
    tail -100 /var/log/minio-check/minio-check-*.log

---

### 19.3 Example of Scheduled Execution

Add a crontab entry:

    crontab -e

Example:

    0 9 * * * /usr/local/bin/minio-daily-check.sh >/dev/null 2>&1

Explanation:

    The script runs at 09:00 every day.
    In production, it is recommended to integrate inspection results into a logging platform or notification system.
    If any issues are detected, alerts should be triggered instead of just saving the logs locally.

---

## Chapter Twenty: Capacity Management Methodology

### 20.1 Capacity Is Not Just About Total Size

MinIO capacity management involves multiple aspects:

    Total cluster capacity
    Node capacity
    Disk capacity
    Bucket capacity
    Prefix capacity
    Number of objects
    Growth trend
    Backup capacity

---

### 20.2 Sources of Capacity Increase

Common causes of capacity growth include:

    Users uploading attachments.
    Business log archiving.
    CI/CD build packages.
    Data backups.
    AI datasets.
    Unremoved temporary files.
    Long-term retention of versioned files.
    Mirror synchronization duplication.
    Abnormal loop uploads by programs.

---

### 20.3 Measures for Capacity Governance

Control methods include:

    Standardized Bucket naming conventions.
    Registration of Bucket ownership.
    Planning of Prefix structures.
    Lifecycle management and cleanup.
    Deletion of expired objects.
    Quota restrictions.
    Archiving of cold data.
    Cross-cluster backups.
    Alerts for abnormal growth.
    Regular capacity reviews.

---

### 20.4 Recommendations for Capacity Expansion

Before expanding, consider:

    Current number of nodes.
    Current number of disks.
    Current capacity utilization rate.
    Current growth rate.
    Current number of objects.
    Current distribution of Buckets.
    Current```markdown
-H "Authorization: Bearer <token>" \
https://s3.minio.local/minio/v2/metrics/cluster | head
```

---

### 22.2 Grafana shows no data

Possible issues:

- Check if the Prometheus target is running.
- Verify if the PromQL metric name exists.
- Ensure the dashboard matches the MinIO version.
- Confirm the time range and data source are correct.
- Verify if Prometheus has permission to collect data.

---

### 22.3 Abnormal increase in Bucket capacity

Possible causes:

- Use `mc du` to check Bucket usage.
- Run `mc find` to analyze object distribution.
- Check Nginx access.log for recent upload activities.
- Monitor the time when the business was deployed.
- Look for any repeated or cyclic uploads.
- Check if there are duplicate backups being written.
- Verify if the lifecycle policy is missing.

Action steps:

- First, determine which business is responsible for the increased capacity.
- Confirm whether it is an abnormal increase.
- Do not delete the Bucket immediately.
- Make sure to back up the data before deletion.
- If necessary, restrict the AccessKey or suspend the affected service.

---

### 22.4 Upload fails but MinIO is working normally

Possible reasons:

- Check Nginx's `client_max_body_size` setting.
- Verify if there is a timeout issue with the Nginx proxy.
- Ensure the user has the necessary `PutObject` permissions.
- Confirm that the Bucket exists.
- Check if the disk is full.
- Verify if the write quorum requirement is not met.
- Check if the client's endpoint configuration is incorrect.
- Verify if there are certificate issues.
- Check if the proxy Host header causes signature errors.

Action steps:

- Use `mc admin info` to gather basic information.
- Run `mc ls` to list files.
- Test small and large file transfers using `mc cp`.
- Check Nginx's `error.log` for any errors.
- Review MinIO's docker logs.
- Use `df -hT` to check disk usage.

---

### 22.5 Slow download speed

Possible causes:

- Disk I/O bottlenecks.
- Insufficient network bandwidth.
- Nginx proxy performance issues.
- Limited client bandwidth.
- Large number of healing operations.
- Offline backend nodes.
- Excessive small objects in the Bucket.
- Insufficient concurrent connections from a single client.

Action steps:

- Use `iostat -x 1` to check disk I/O statistics.
- Monitor network traffic with `iftop`.
- Check Docker stats for related information.
- Use `mc admin info` to gather system details.
- Analyze Nginx access.log for request times.
- Monitor Prometheus API latency indicators.

---

## Chapter 23: Reminders for High-Risk Operations

The following operations must be performed with caution in a production environment:

- Deleting a Bucket.
- Recursively deleting Prefixes.
- Using `mc rm --recursive --force`.
- Modifying Policies.
- Disabling or deleting users.
- Changing Nginx configuration.
- Updating certificates.
- Adjusting MinIO startup parameters.
- Deleting data directories.
- Directly cleaning the disk.
- Performing full-scale healing on large Buckets.
- Executing large-scale mirroring during peak hours.

Before proceeding, ensure the following:

- There are backups in place.
- Approval has been obtained.
- Business stakeholders have confirmed the operation.
- A rollback plan is available.
- The potential impact is understood.
- The operation is within a maintenance window.
- Someone has reviewed the plan.
- Operation records will be kept.

---

## Chapter 24: Production Ops Baselines

### 24.1 Monitoring Baselines

Essential monitoring indicators include:

- MinIO API availability.
- MinIO readiness status.
- Node health checks.
- Disk status and capacity.
- Bucket capacity monitoring.
- API error rate tracking.
- Nginx 5xx errors.
- Certificate expiration alerts.
- Backup task monitoring.

---

### 24.2 Alert Baselines

Alerts should be set for the following conditions:

- MinIO unavailability.
- Offline nodes.
- Offline disks.
- Capacity exceeding 80% or 90%.
- S3 5xx errors.
- Nginx 502/504 responses.
- Certificate expiration approaching.
- Backup failures.
- Healing operations taking too long to complete.

---

### 24.3 Log Baselines

Important logs to retain include:

- MinIO container logs.
- Nginx access.log and error.log.
- System logs.
- Docker logs.
- Backup task logs.
- Records of high-risk operations.

---

### 24.4 Capacity Baselines

Key capacity metrics include:

- Current total capacity.
- Available capacity.
- Current usage rate.
- Largest Buckets.
- Buckets with the fastest growth rates.
- Estimatedhttps://min.io/docs/minio/linux/reference/minio-mc-admin/mc-admin-info.html

MinIO mc delete:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-delete.html

MinIO mc find:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-find.html

MinIO mc mirror:

    https://min.io/docs/minio/linux/reference/minio-mc/mc-mirror.html

Nginx Official Documentation:

    https://nginx.org/en/docs/

Prometheus Official Documentation:

    https://prometheus.io/docs/introduction/overview/

Grafana Official Documentation:

    https://grafana.com/docs/

Alertmanager Official Documentation:

    https://prometheus.io/docs/alerting/latest警报管理器/