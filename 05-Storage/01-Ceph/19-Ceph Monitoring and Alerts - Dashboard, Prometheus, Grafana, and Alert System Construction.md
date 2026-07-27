# Ceph Monitoring and Alerts: Dashboard, Prometheus, Grafana, and Alert System Construction

Recommended Path: 05-Storage/01-Ceph/19-Ceph Monitoring and Alerts: Dashboard, Prometheus, Grafana, and Alert System Construction.md

Tags: #Ceph #Monitoring and Alerts #Prometheus #Grafana #Dashboard #Alertmanager #OSD #PG #Capacity Alerts #Performance Monitoring #SRE #Advanced SRE

---

## I. Document Overview

This article is the nineteenth in the Ceph Advanced SRE storage module series, focusing on organizing Ceph monitoring, metrics, alerts, and observability system construction.

Previous topics covered include:

- Ceph cluster deployment
- OSD management
- Pools and PGs
- CRUSH fault domains
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations
- Ceph troubleshooting
- Ceph backup and recovery
- Ceph performance optimization
- Ceph security enhancement

This article now enters the production observability phase.

After a Ceph cluster is deployed, it cannot rely solely on manual commands such as:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph osd df

In a production environment, the following must be established:

- Dashboard visualization
- Prometheus metric collection
- Grafana dashboard display
- Alertmanager notification system
- Regular inspections and closed-loop alert management
- Fault analysis and threshold optimization

This article focuses on the following areas:

- Ceph monitoring objectives
- The role of the Ceph Dashboard
- The MGR Prometheus module
- How Prometheus collects Ceph metrics
- The Grafana Ceph monitoring dashboard
- Key metrics for OSD, PG, Pool, MON, MGR, MDS, and RGW
- Kubernetes CSI-related monitoring
- Design of common alert rules
- Capacity alert thresholds
- OSD down alerts
- PG degraded/inactive alerts
- MDS/RGW/CSI alerts
- Alertmanager notification mechanisms
- Alert prioritization
- Alarm noise reduction
- Production monitoring benchmarks

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why it is essential to establish a monitoring and alert system in a Ceph production environment.
2. Comprehend the responsibilities of the Ceph Dashboard, Prometheus, Grafana, and Alertmanager.
3. Enable the Ceph MGR Prometheus module.
4. View Ceph Prometheus metric endpoints.
5. Configure Prometheus to collect Ceph metrics.
6. Use Grafana to display the Ceph cluster status.
7. Identify core Ceph monitoring indicators.
8. Design alerts for OSD down, PG degraded, capacity levels, MON quorum, MDS exceptions, and RGW anomalies.
9. Distinguish between capacity alerts and full alerts.
10. Understand the impact of slow operations, OSD latency, and recovery/backfill processes on business operations.
11. Establish an alert prioritization strategy.
12. Integrate Alertmanager with email, Webhook, WeCom, Lark, or other notification channels.
13. Form a closed-loop for monitoring, alerts, response, analysis, and threshold optimization.
14. Identify which alerts in a production environment require immediate attention and which can be scheduled for handling.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article uses the same experimental Ceph cluster nodes as before.

| IP | Host Name | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion/Fault Testing (optional) |
| 10.0.0.35 | ceph-client | RBD/CephFS/RGW Client Testing (optional) |
| 10.0.0.36 | monitor-node | Prometheus/Grafana/Alertmanager (optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Monitoring Component Planning

By default, the monitoring components in this article can be deployed on:

    10.0.0.36 monitor-node

Alternatively, Prometheus/Grafana/Alertmanager can be deployed using cephADM.

Common deployment options include:

| Option | Description |
|---|---|
### 6.2 Viewing the MGR Module

    ceph mgr module ls

Focus on:

    prometheus
    dashboard

---

### 6.3 Viewing the MGR Service Address

    ceph mgr services

If prometheus is enabled, you may see:

    prometheus: http://ceph-node01:9283/

---

## VII. Experimental Task List

| Experiment | Objective | Risk Level |
|---|---|---|
| Experiment 1 | Enable the Ceph Dashboard | Medium |
| Experiment 2 | Enable the MGR Prometheus Module | Low |
| Experiment 3 | Verify the Ceph Metrics Endpoints | Low |
| Experiment 4 | Configure Prometheus to Capture Ceph Metrics | Medium |
| Experiment 5 | Import Grafana for Ceph Monitoring | Medium |
| Experiment 6 | Set Up Basic Alarm Rules | Medium |
| Experiment 7 | Configure Alertmanager Notification Links | Medium |
| Experiment 8 | Verify OSD Down Alarms | Medium-High |
| Experiment 9 | Verify Capacity Alarms | Medium |
| Experiment 10 | Verify PG Exception Alarms | Medium |
| Experiment 11 | Monitor CephFS / MDS | Medium |
| Experiment 12 | Monitor RGW | Medium |
| Experiment 13 | Monitor Kubernetes CSI | Medium |
| Experiment 14 | Alarm Prioritization and Noise Reduction | Medium |
| Experiment 15 | Develop Production Monitoring Templates | Low |

High-Risk Reminders:

    When verifying OSD down alarms, do not directly stop the OSD in production.
    Do not verify capacity alarms by actually filling up the cluster.
    Always test alarm rules in a testing environment before deploying them.
    Be cautious when testing Alertmanager notifications to avoid generating excessive duplicate alerts.

---

## VIII. Experiment 1: Enable the Ceph Dashboard

### 8.1 Viewing the Dashboard Module

    ceph mgr module ls | grep dashboard

If it is not enabled:

    ceph mgr module enable dashboard

---

### 8.2 Viewing the Dashboard Service

    ceph mgr services

Expected output:

    dashboard: https://ceph-node01:8443/

---

### 8.3 Creating a Dashboard Administrator User

Create a password file:

    echo 'StrongPasswordHere' > /root/dashboard-pass.txt
    chmod 600 /root/dashboard-pass.txt

Create the user:

    ceph dashboard ac-user-create admin -i /root/dashboard-pass.txt administrator

Delete the password file:

    shred -u /root/dashboard-pass.txt

---

### 8.4 Accessing the Dashboard

Access via browser:

    https://10.0.0.31:8443

or:

    https://ceph-node01:8443

Production Recommendations:

    Do not expose the dashboard to the public internet.
    Only allow access from the operations network segment.
    Use HTTPS for secure communication.
    Employ a strong password.
    Minimize the number of administrator accounts.

---

## IX. Experiment 2: Enable the MGR Prometheus Module

### 9.1 Enabling the prometheus Module

    ceph mgr module enable prometheus

Check:

    ceph mgr services

Expected output:

    prometheus: http://<mgr-node>:9283/

---

### 9.2 Viewing the prometheus Module Configuration

    ceph config get mgr mgr/prometheus/server_addr
    ceph config get mgr mgr/prometheus/server_port

The default port is usually:

    9283

---

### 9.3 Setting the Listening Address

If you need to listen on all addresses:

    ceph config set mgr mgr/prometheus/server_addr 0.0.0.0

Set the port:

    ceph config set mgr mgr/prometheus/server_port 9283

Restart MGR, as needed:

    ceph orch ps --daemon_type mgr
    ceph orch daemon restart mgr.<mgr-daemon-name>

The actual name will be displayed by `ceph orch ps`.

---

## X. Experiment 3: Verifying the Ceph Metrics Endpoints

### 10.1 Obtaining the Metrics Address

Execute:

    ceph mgr services

Assumed output:

    prometheus: http://10.0.0.31:9283/

---

### 10.2 Testing with curl

Run on the monitor-node or Prometheus node:

    curl -s http://10.0.0.31:9283/metrics | head

If a large number of metrics are displayed, it indicates normal operation.

Filter for Ceph metrics:

    curl -s http://10.0.0.31:9283/metrics | grep '^ceph_' | head

---

### 10.3 Common Issues

If access fails, check:

    ceph mgr services
    ceph mgr module ls
    c| ceph_osd_in | Whether the OSD is involved in data distribution |

Common anomalies:

    up = 0, in = 1

Indicates:

    The OSD is down but still listed as "in".
    This may lead to a degraded status.

---

### 13.3 PG Status Metrics

Key areas of concern:

- active
- clean
- degraded
- undersized
- inactive
- inconsistent
- stale
- peering
- recovering
- backfilling

PG-related anomaly alerts carry high priority.

---

### 13.4 Capacity Metrics

Focus on:

- Total cluster capacity
- Used cluster capacity
- Cluster utilization rate
- OSD utilization rate
- Pool usage
- Pool growth trend
- Nearfull/full status

Capacity alerts must be issued in advance.

Do not wait until the system is at full capacity before notifying.

---

### 13.5 OSD Latency Metrics

Monitor:

- Commit latency
- Apply latency
- OSD operation latency
- Slow operations

If an OSD shows significantly higher latency than others, investigate potential issues such as:

- Disk performance
- Network connectivity
- Node load
- Recovery/backfilling processes
- OSD failures

---

### 13.6 MON / MGR Metrics

Watch for:

- MON quorum
- Number of MONs
- Active MGR status
- Standby MGR status
- Status of MGR modules
- Dashboard/Prometheus service status

---

### 13.7 MDS / CephFS Metrics

Track:

- Number of active MDSes
- Number of standby MDSes
- MDS request latency
- Slow MDS requests
- Number of CephFS clients
- Metadata pool status
- Data pool status

---

### 13.8 RGW Metrics

Check:

- Whether the RGW daemon is running
- Request volume
- 4xx and 5xx error rates
- Upload/download traffic
- Request latency
- Number of Buckets
- Number of Objects
- User quotas
- Bucket quotas

RGW monitoring should be combined with:

    Ceph metrics
    RGW logs
    Nginx/LB metrics
    S3 client error rates

---

## Chapter 14: Example of Basic Alarm Rules

### 14.1 Prometheus Alert Rule Files

Example path:

    /etc/prometheus/rules/ceph-alerts.yml

Rule examples:

    groups:
      - name: ceph-basic-alerts
        rules:
          - alert: CephHealthWarn
            expr: ceph_health_status == 1
            for: 5m
            labels:
              severity: warning
              service: ceph
            annotations:
              summary: "The Ceph cluster is in HEALTH_WARN status."
              description: "The Ceph cluster has been in HEALTHWARN for 5 minutes. Please check details using 'ceph health detail'."

          - alert: CephHealthError
            expr: ceph_health_status == 2
            for: 1m
            labels:
              severity: critical
              service: ceph
            annotations:
              summary: "The Ceph cluster is in HEALTH_ERR status."
              description: "The Ceph cluster has been in HEALTH_ERR for 1 minute. Please immediately check 'ceph -s' and 'ceph health detail'."

---

### 14.2 OSD Down Alerts

Examples:

    groups:
      - name: ceph-osd-alerts
        rules:
          - alert: CephOSDDown
            expr: ceph_osd_up == 0
            for: 2m
            labels:
              severity: critical
              service: ceph
            annotations:
              summary: "An OSD in {{ $labels.ceph_daemon }} is down for more than 2 minutes."
              description: "OSD {{ $labels.ceph_daemon }} has been down for over 2 minutes. Please check the 'ceph osd tree' and related nodes."

          - alert: CephOSDOut
            expr: ceph_osd_in == 0
            for: 5m
            labels:
              severity: warning
              service: ceph
            annotations:
              summary: "The OSD in {{ $labels.ceph_daemon }} is currently in the 'out' state."
              description: "OSD {{ $labels.ceph_daemon }} is currently out. Verify if it's due to scheduled maintenance."

---

### 14.3 Capacity Alerts

Examples:

    groups:
      - name: ceph-capacity-alerts
        rules:
          - alert: CephOSDUsageHigh
            expr: ceph_osd_stat_bytes_used / ceph_osd_stat_bytes * 100 > 80
            for: 10m
            labels:
              severity: warning
              service: ceph
            expr: ceph_pg_inconsistent > 0
            for: 1m
            labels:
              severity: critical
              service: ceph
            annotations:
              summary: "Inconsistent PGs have been detected in Ceph."
              description: "Inconsistent PGs exist. Please investigate carefully and avoid performing automatic repairs without understanding the issue."

Note:

    The metric name should be based on the actual Prometheus query results.
    If there is no direct `ceph.pg_degraded` metric in the current version, you need to combine relevant PG status metrics to represent it.

---

## Section 15: Experiment 7: Alertmanager Notification Mechanism

### 15.1 Role of Alertmanager

Alertmanager is responsible for:

- Grouping alerts
- Duplicating alert notifications
- Suppressing alerts
- Silencing alerts temporarily
- Sending notification messages
- Routing alerts based on their severity levels

It's essential not to let Prometheus send all alerts indiscriminately.

---

### 15.2 Alert Classification

Recommended classification levels:

| Level | Examples | Notification Method |
|---|---|---|
| critical | HEALTH_ERR, OSD full, PG inactive, MON quorum exception | Telephone, IM, SMS, duty shift notification group |
| warning | HEALTH_WARN, OSD near-full, PG degraded, MDS standby missing | IM, email |
| info | Configuration recommendations, low-risk alerts | Email, ticket system |

---

### 15.3 Example of Alertmanager Routing

Example configuration:

    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'default'

      routes:
        - matchers:
            - severity="critical"
          receiver: 'critical-webhook'
          repeat_interval: 30m

        - matchers:
            - severity="warning"
          receiver: 'warning-webhook'
          repeat_interval: 4h

    receivers:
      - name: 'default'
        webhook_configs:
          - url: 'http://127.0.0.1:5001/alert'

      - name: 'critical-webhook'
        webhookconfigs:
          - url: 'http://127.0.0.1:5001/critical'

      - name: 'warning-webhook'
        webhook_configs:
          - url: 'http://127.0.0.1:5001/warning'

Note:

    Replace the Webhook addresses with your actual notification services, such as WeCom, Lark, DingTalk, SMS platforms, or custom alert systems.

---

### 15.4 Recommendations for Alert Content

Alert messages should include at least:

- Alert name
- Alert level
- Cluster name
- Affected instance or object
- Current value
- Duration of the issue
- Impact description
- Initial handling suggestions
- Grafana link
- Prometheus link

---

## Section 16: Experiment 8: Verifying OSD Down Alerts

### 16.1 High-Risk Warning

Never stop an OSD in a production environment casually.

It is recommended to perform this verification only in a test environment.

Before verification, check:

    ceph -s
    ceph osd tree
    ceph pg stat

---

### 16.2 Stopping One OSD in the Test Environment

Check the status of the OSD:

    ceph osd tree

Stop the test OSD:

    ceph orch daemon stop osd.0

Observe the changes:

    ceph -s
    ceph osd tree

Prometheus and Alertmanager should trigger an OSD down alert.

---

### 16.3 Restoring the OSD

Start it again:

    ceph orch daemon start osd.0

Check the status after restoration:

    ceph -s
    ceph osd tree
    ceph pg stat

Expected results:

    The OSD should be listed as up/in.
    The PG should be in the active+clean state.
    The alert should have automatically resolved.

---

### 16.4 Verification Points

| Verification Item | Result |
|---|---|
| Did Prometheus trigger an alert? | Yes / No |
| Did Alertmanager send a notification? | Yes / No |
| Was the alert content understandable? | Yes / No |
| Did the alert disappear after the OSD was restored? | Yes / No |
| Were there any unexpected alerts triggered? | Yes / No |

---

## Section 17: Experiment 9: Capacity Alarm Design

### 17.1 Avoid Verifying Capacity Alerts by Filling Up the Cluster

Do not fill up the entire Ceph cluster just to verify capacity alarms.

Correct approaches include:

- Lowering the alarm threshold in a test environment for verification.
- Temporarily adjusting the threshold using Prometheus    radosgw-admin bucket stats --bucket=<bucket-name>
    ceph df
    ceph osd pool ls | grep rgw

Port Testing:

    curl -I http://10.0.0.31:7480

S3 Testing:

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls

---

### 20.3 RGW Alarm Recommendations

| Alarm | Level |
|---|---|
| All RGW instances unavailable | critical |
| Single RGW instance down | warning |
| High RGW 5xx error rate | critical |
| Abnormally high RGW 4xx error rate | warning |
| Abnormal increase in bucket object count | warning |
| RGW pool nearfull | critical |
| HTTPS certificate about to expire | warning |

---

## Chapter Twenty-One: Experiment Thirteen: Kubernetes CSI Monitoring

### 21.1 RBD CSI Monitoring Objectives

Monitoring Targets:

- Pod status within the ceph-csi Namespace
- csi-rbdplugin-provisioner
- csi-rbdplugin DaemonSet
- PVC Pending
- PV Released / Failed
- VolumeAttachment anomalies
- Pod FailedMount
- kubelet mounting errors
- k8s-rbd Pool capacity
- Number of RBD Images

---

### 21.2 CephFS CSI Monitoring Objectives

Monitoring Targets:

- csi-cephfsplugin-provisioner
- csi-cephfsplugin DaemonSet
- PVC Pending
- Pod FailedMount
- Number of CephFS Subvolumes
- MDS status
- Metadata/data pool capacity

---

### 21.3 Kubernetes Inspection Commands

    kubectl get pods -n ceph-csi -o wide
    kubectl get pvc -A
    kubectl get pv
    kubectl get volumeattachment
    kubectl get events -A --sort-by=.lastTimestamp

Troubleshooting:

    kubectl describe pvc -n <namespace> <pvc-name>
    kubectl describe pod -n <namespace> <pod-name>
    kubectl logs -n ceph-csi <csi-pod-name> -c csi-rbdplugin
    kubectl logs -n ceph-csi <csi-pod-name> -c csi-cephfsplugin

---

### 21.4 CSI Alarm Recommendations

| Alarm | Level |
|---|---|
| ceph-csi controller unavailable | critical |
| ceph-csi node plugin unavailable | critical |
- PVC Pending for more than 5 minutes | warning |
- Pod FailedMount | warning / critical |
- VolumeAttachment anomalies | warning |
- Abundant CSI log errors | warning |
- k8s-rbd Pool nearfull | critical |

---

## Chapter Twenty-Two: Experiment Fourteen: Alarm Prioritization and Noise Reduction

### 22.1 Why Alarm Noise Reduction is Needed

Without noise reduction, alarms can lead to:

- Too many alarms going unnoticed
- Serious issues being overlooked
- Staff fatigue
- Alarm clutter becoming overwhelming
- Slower failure response times

---

### 22.2 Alarm Grouping

It is recommended to group alarms based on the following dimensions:

    cluster
    service
    alertname
    severity

For example:

- Multiple PG degradations within the same cluster can be combined into one alarm group.
- Multiple OSD anomalies on the same node can be consolidated for notification.
- A large number of PVC FailedMounts occurring at the same time can be categorized as a storage failure event.

---

### 22.3 Alarm Suppression

Examples:

- If a CephHealthError has already been triggered, some lower-level CephHealthWarn alarms can be suppressed.
- In the event of a MON quorum issue, related derivative alarms may need to be suppressed.
- If an entire node fails, repeated daemon down alarms for that node can be suppressed.

---

### 22.4 Alarm Silence

During maintenance periods, silence settings can be applied.

Applicable Scenarios:

- Planned OSD node restarts
- Scheduled Ceph upgrades
- Planned network maintenance
- Planned disk replacements
- Planned scale-out or migrations

Notes:

- Silence settings must include an expiration time.
- Silencing should not be permanent.
- After maintenance, it is essential to ensure that alarms resume as normal.

---

### 22.5 Alarm Repeat Interval Recommendations

| Alarm Level | Repeat Interval |
|---|---|
| critical | 30 minutes |
| warning | 4 hours |
| info | 12 hours or longer |

Adjustments should be made according to the actual duty schedule of the organization.

---

## Chapter Twenty-Three: Baseline Production Monitoring Dashboards

### 23.1 Ceph Overview Dashboard

Must include:

- Ceph health status
- MON quorum status
- Active MGR status
- OSD up/in status
### 26.1 Prometheus Fails to Retrieve Ceph Metrics

Troubleshooting:

    Check `ceph mgr services`
    List available `ceph mgr module ls`
    Access metrics via `curl http://10.0.0.31:9283/metrics`
    Verify network connectivity with `ss -lntp | grep 9283`

Check Prometheus:

    Ensure the "targets" page shows it is up.
    Confirm the `prometheus.yml` configuration is correct.
    Verify network accessibility and firewall settings.

---

### 26.2 Grafana Displays No Data

Troubleshooting:

    Verify if Prometheus is generating data.
    Check if the Grafana data source is set correctly.
    Ensure the dashboard query matches the required metrics.
    Confirm that metric labels match template variables.
    Verify the time range specified is appropriate.

---

### 26.3 Alerts Are Not Triggered

Troubleshooting:

    Ensure Prometheus rules are loaded and valid.
    Check if the expressions produce results.
    Verify the "for" time range is satisfied.
    Confirm that label values match Alertmanager routing settings.
    Ensure Alertmanager is accessible.

Check the Prometheus rules page for any issues.

---

### 26.4 Alerts Are Triggered But No Notification Is Sent

Troubleshooting:

    Verify Alertmanager routing and receiver configuration.
    Check the webhook address and network connectivity.
    Confirm the notification platform token is valid.
    Check if there are silence settings in place.
    Verify if inhibition rules are preventing notifications.

---

### 26.5 Excessive Alerts

Solution:

    Set appropriate "for" time ranges for alerts.
    Use `group_by` to organize similar alerts.
    Adjust notification intervals for duplicate alerts.
    Apply inhibition rules to suppress derived alerts.
    Distinguish between warning and critical alerts.
    Use silence periods during maintenance windows.
    Adjust threshold settings or remove irrelevant alerts.

---

## Chapter 27: Best Practices for Production Environments

1. Do not rely solely on manual checks using `ceph -s`.
2. Always configure alerts alongside dashboard monitoring.
3. Monitor not only Ceph but also node resources.
4. Track both total cluster capacity and individual OSD/Pool usage.
5. Pay attention to issues with PG states such as inactive/incomplete/consistent.
6. Avoid waiting until storage is full before issuing alerts.
7. Ensure that Alertmanager notifications are delivered to the appropriate personnel.
8. Prevent the alert system from being overwhelmed by false positives.
9. Do not permanently silence alerts.
10. Never deploy alert rules without thorough testing.
11. Do not directly apply test environment settings to production.
12. Monitor Kubernetes CSI-related PVC/FailedMount issues.
13. Pay attention to RGW HTTPS certificate-related alerts.
14. Be aware of missing MDS standby replicas.
15. All alerts must be responded to, recorded, and used for improvement.

---

## Chapter 28: Advanced SRE Methodologies

### 28.1 Monitoring Is Not Just About Dashboards

A Grafana dashboard is merely a tool for visibility.

An effective production monitoring system should include:

    Metric collection
    Alert rules
    Alert routing
    Duty officer response
    Fault resolution
    Post-event analysis and optimization
    Threshold adjustment

---

### 28.2 Alerts Must Be Actionable

Good alerts should provide duty officers with the following information:

    What issue has occurred?
    Which services are affected?
    How severe is the impact?
    What commands should be checked first?
    Is immediate action required?
    Are there existing troubleshooting guides?

Poorly designed alerts merely indicate a problem without providing actionable guidance.

---

### 28.3 Capacity Trends Are More Important Than Current Levels

When monitoring Ceph capacity, focus not only on current percentages but also on:

    7-day and 30-day growth rates
    Which Pool is growing the fastest
    Which business areas are showing unusual trends
    Estimated time until 80% or 85% capacity levels are reached

Advanced SRE personnel should plan capacity expansions in advance, rather than waiting for near-full conditions before taking action.

---

### 28.4 Alert Classification Determines Response Efficiency

If all alerts are classified as critical, then no alerts will be considered critical at all.

Recommendation:

    Critical: Affects business availability or data security
    Warning: Poses a risk but can be monitored for a short period
    Info: Suggests potential optimization areas or trend observations

---

### 28.5 Monitoring Should Cover the Entire Chain of Events

Ceph-related issues can occur in various components:

    Business applications
    Kubernetes PVCs
    CSI interactions
    RBD/CephFS/RGW services
    OSD/PG/Pool structures
    Node disks and storage
    Network connectivity
    Time synchronization mechanisms
    Nginx/LB load balancers

Monitoring only Ceph itself is insufficient.

---

https://kubernetes.io/docs/concepts/storage/persistent-volumes/