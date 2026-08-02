# Ceph Monitoring Alerts: Dashboard, Prometheus, Grafana, and Alerting System Construction

Suggested path: 05-Storage/01-Ceph/19-Ceph Monitoring Alerts: Dashboard, Prometheus, Grafana, and Alerting System Construction.md

Tags: #Ceph #SecurityAlert. #Prometheus #Grafana #Dashboard #Alertmanager #OSD #PG #CapacityAlert. #PerformanceMonitoring #SRE #AdvancedSre

---

## I. Document Explanation

This is the nineteenth article of the Ceph advanced SRE storage module, focusing on organizing Ceph monitoring, metrics, alerts, and observability system construction.

Previously completed:

- Ceph cluster deployment
- OSD management
- Pool and PG
- CRUSH fault domain
- RBD block storage
- CephFS file storage
- RGW object storage
- Kubernetes RBD CSI
- Kubernetes CephFS CSI
- Ceph daily operations
- Ceph troubleshooting
- Ceph backup and recovery
- Ceph performance optimization
- Ceph security reinforcement

This article enters the production observability phase.

After Ceph cluster goes live, it cannot rely solely on manual execution:

    ceph -s
    ceph health detail
    ceph osd tree
    ceph osd df

Production environments must establish:

    Dashboard visualization
    Prometheus metric collection
    Grafana dashboard display
    Alertmanager alert notification
    Daily inspection and alert closure loop
    Fault review and threshold optimization

This article will cover:

- Ceph monitoring objectives
- Role of Ceph Dashboard
- MGR Prometheus module
- Prometheus scraping Ceph metrics
- Grafana Ceph monitoring dashboard
- Key metrics for OSD / PG / Pool / MON / MGR / MDS / RGW
- Kubernetes CSI-related monitoring
- Common alert rule design
- Capacity alert thresholds
- OSD down alerts
- PG degraded / inactive alerts
- MDS / RGW / CSI alerts
- Alertmanager notification chain
- Alert classification
- Alert noise reduction
- Production monitoring baseline

---

## II. Learning Objectives

After completing this article, you should be able to:

1. Understand why a monitoring alert system is essential in production environments for Ceph.
2. Understand the responsibilities and boundaries of Ceph Dashboard, Prometheus, Grafana, and Alertmanager.
3. Enable the Ceph MGR Prometheus module.
4. View the Ceph Prometheus metrics endpoint.
5. Configure Prometheus to scrape Ceph metrics.
6. Use Grafana to display Ceph cluster status.
7. Identify core Ceph monitoring metrics.
8. Design alerts for OSD down, PG degraded, capacity water level, MON quorum, MDS anomalies, RGW anomalies.
9. Understand the difference between capacity alerts and full alerts.
10. Understand the impact of slow ops, OSD latency, recovery/backfill on business operations.
11. Establish an alert classification strategy.
12. Integrate Alertmanager with email, Webhook, Enterprise WeChat, Feishu, or other notification channels.
13. Form a closed-loop of monitoring, alerts, response, review, and threshold optimization.
14. Clarify which alerts must be addressed immediately and which can be scheduled for governance in production environments.

---

## III. Experimental Environment

### 3.1 Ceph Cluster Nodes

This article continues using the Ceph module experimental environment.

| IP | Hostname | Role |
|---|---|---|
| 10.0.0.31 | ceph-node01 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.32 | ceph-node02 | MON / MGR / OSD / RGW / MDS |
| 10.0.0.33 | ceph-node03 | MON / MGR / OSD / RGW |
| 10.0.0.34 | ceph-node04 | OSD / Expansion / Fault Simulation (optional) |
| 10.0.0.35 | ceph-client | RBD / CephFS / RGW Client Testing (optional) |
| 10.0.0.36 | monitor-node | Prometheus / Grafana / Alertmanager (optional) |

Main experimental system:

    Ubuntu Server 22.04.5 LTS

Supplementary system:

    Rocky Linux 9

Deployment method:

    cephadm

---

### 3.2 Monitoring Component Planning

This article assumes monitoring components can be deployed on:

    10.0.0.36 monitor-node

Alternatively, they can be deployed by cephadm as Prometheus / Grafana / Alertmanager.

Common schemes:

| Scheme | Description |
|---|---|
| Cephadm built-in monitoring stack | Ceph includes Prometheus, Grafana, and Alertmanager |
| External Prometheus | Company's existing unified monitoring platform |
| Kubernetes Prometheus | Use kube-prometheus-stack for unified collection |
| Hybrid approach | Ceph metrics integrated with existing Prometheus / Grafana |

This article focuses on methodology and configuration ideas, without forcing a specific deployment method.

---

## IV. Ceph Monitoring Architecture

### 4.1 Typical Monitoring Chain

Typical Ceph monitoring chain:

    Ceph Daemons
      |
      v
    Ceph MGR Prometheus Module
      |
      v
    Prometheus
      |
      v
    Grafana
      |
      v
    Alertmanager
      |
      v
    Email / Webhook / Enterprise WeChat / Feishu / DingTalk

Diagram: /think

```
┌────────────────────────────┐
│         Ceph Cluster         │
│ MON / MGR / OSD / MDS / RGW │
└──────────────┬─────────────┘
               │
               v
┌────────────────────────────┐
│   MGR prometheus module     │
│        :9283/metrics        │
└──────────────┬─────────────┘
               │
               v
┌────────────────────────────┐
│        Prometheus           │
│     Metric Collection & Alert Calculation       │
└──────────────┬─────────────┘
               │
       ┌──────────┴──────────┐
       v                     v
┌──────────────┐     ┌──────────────┐
│   Grafana    │     │ Alertmanager │
│   Dashboard  │     │   Alert Notification    │
└──────────────┘     └──────────────┘

---

### 4.2 Component Responsibilities

| Component | Responsibility |
|---|---|
| Ceph MGR | Aggregate Ceph cluster status and metrics |
| prometheus module | Expose Ceph metrics endpoint |
| Prometheus | Pull metrics, store time-series data, calculate alerts |
| Grafana | Display cluster status and trends |
| Alertmanager | Alert grouping, deduplication, suppression, notification |
| Dashboard | Ceph built-in management and visualization entry |
| Node Exporter | Collect node CPU, memory, disk, network |
| kube-state-metrics | Collect Kubernetes PVC / Pod object status, CSI scenario required |

---

## FiveI don't know.Ceph Monitoring Targets

Production environment monitoring is not just about "whether online", but should cover:

1. Whether the cluster is healthy.
2. Whether all OSDs are up/in.
3. Whether PGs are active+clean.
4. Whether there are degraded, undersized, inactive, inconsistent states.
5. Whether capacity is approaching nearfull/full.
6. Whether pool capacity growth is abnormal.
7. Whether OSD usage is uneven.
8. Whether OSD latency has increased.
9. Whether there are slow ops.
10. Whether MON quorum is normal.
11. Whether MGR active/standby is normal.
12. Whether MDS active/standby is normal.
13. Whether RGW instances are normal.
14. Whether RBD / CephFS / RGW corresponding business is accessible.
15. Whether Kubernetes PVC is Pending or FailedMount.
16. Whether node CPU, memory, disk, network is abnormal.
17. Whether backup tasks are successful.
18. Whether alerts can notify people in time.

---

## SixI don't know.Pre-Operation Checks

### 6.1 Check Ceph Status

    ceph -s
    ceph health detail
    ceph orch ps
    ceph mgr stat

Requirements:

    MGR is normal.
    At least one active MGR.
    Cluster overall status is interpretable.

---

### 6.2 Check MGR Module

    ceph mgr module ls

Focus on:

    prometheus
    dashboard

---

### 6.3 Check MGR Service Address

    ceph mgr services

If Prometheus is already enabled, may see:

    prometheus: http://ceph-node01:9283/

---

## SevenI don't know.Experiment Task List

| Experiment | Target | Risk Level |
|---|---|---|
| Experiment One | Enable Ceph Dashboard | Medium |
| Experiment Two | Enable MGR Prometheus Module | Low |
| Experiment Three | Validate Ceph metrics endpoint | Low |
| Experiment Four | Configure Prometheus to scrape Ceph metrics | Medium |
| Experiment Five | Import Grafana Ceph Dashboard | Medium |
| Experiment Six | Configure Basic Alert Rules | Medium |
| Experiment Seven | Configure Alertmanager Notification Chain | Medium |
| Experiment Eight | Validate OSD down alert | Medium-High |
| Experiment Nine | Validate Capacity Alert | Medium |
| Experiment Ten | Validate PG Abnormal Alert | Medium |
| Experiment Eleven | Monitor CephFS / MDS | Medium |
| Experiment Twelve | Monitor RGW | Medium |
| Experiment Thirteen | Monitor Kubernetes CSI | Medium |
| Experiment Fourteen | Alert Tiering and Noise Reduction | Medium |
| Experiment Fifteen | Production Monitoring Inspection Template | Low |

High-Risk Reminder:

    Do not stop OSD directly in production when validating OSD down alert.
    Do not verify capacity alert by actually filling up the cluster.
    Alert rules should be tested in test environment before deployment.
    Alertmanager notification testing should not cause massive duplicate alert floods.

---

## EightI don't know.Experiment One: Enable Ceph Dashboard

### 8.1 Check Dashboard Module

    ceph mgr module ls | grep dashboard

If not enabled:

    ceph mgr module enable dashboard

---

### 8.2 Check Dashboard Service

    ceph mgr services

Possible output:

    dashboard: https://ceph-node01:8443/

---

### 8.3 Create Dashboard Administrator User

Create password file:

    echo 'StrongPasswordHere' > /root/dashboard-pass.txt
    chmod 600 /root/dashboard-pass.txt

Create user:

    ceph dashboard ac-user-create admin -i /root/dashboard-pass.txt administrator

Delete password file: /root/dashboard-pass.txt
```

shred -u /root/dashboard-pass.txt

---

### 8.4 Dashboard Access

Browser access:

    https://10.0.0.31:8443

or:

    https://ceph-node01:8443

Production recommendations:

    Dashboard should not be exposed to the public internet.
    Only allow access from the operations network segment.
    Use HTTPS.
    Use strong passwords.
    Minimize administrator accounts.

---

## Nine. Experiment Two: Enabling MGR Prometheus Module

### 9.1 Enable prometheus module

    ceph mgr module enable prometheus

Check:

    ceph mgr services

Expected to see:

    prometheus: http://<mgr-node>:9283/

---

### 9.2 View prometheus module configuration

    ceph config get mgr mgr/prometheus/server_addr
    ceph config get mgr mgr/prometheus/server_port

Default port is typically:

    9283

---

### 9.3 Set listening address

If needing to listen on all addresses:

    ceph config set mgr mgr/prometheus/server_addr 0.0.0.0

Set port:

    ceph config set mgr mgr/prometheus/server_port 9283

Restart MGR, as needed:

    ceph orch ps --daemon_type mgr
    ceph orch daemon restart mgr.<mgr-daemon-name>

Actual daemon name is determined by the output of ceph orch ps.

---

## Ten. Experiment Three: Verifying Ceph metrics endpoint

### 10.1 Get metrics address

Execute:

    ceph mgr services

Assume output:

    prometheus: http://10.0.0.31:9283/

---

### 10.2 curl Test

Run on monitor-node or Prometheus node:

    curl -s http://10.0.0.31:9283/metrics | head

If there is a large amount of metric output, it indicates normal operation.

Filter Ceph metrics:

    curl -s http://10.0.0.31:9283/metrics | grep '^ceph_' | head

---

### 10.3 Common issues

If access fails, check:

    ceph mgr services
    ceph mgr module ls
    ceph orch ps --daemon_type mgr
    ss -lntp | grep 9283

Network-side check:

    nc -vz 10.0.0.31 9283

---

## Eleven. Experiment Four: Configuring Prometheus to scrape Ceph metrics

### 11.1 Prometheus scrape configuration example

Add to Prometheus configuration:

    scrape_configs:
      - job_name: 'ceph'
        honor_labels: true
        static_configs:
          - targets:
              - '10.0.0.31:9283'

Notes:

    If active MGR switches to other nodes, the metrics endpoint may change.
    cephadm's built-in monitoring stack automatically handles more details.
    For external Prometheus scenarios, it is recommended to combine with DNS, LB, or multi-MGR strategies.

---

### 11.2 Reload Prometheus

For systemd deployment:

    systemctl reload prometheus

If reload is not supported:

    systemctl restart prometheus

For container deployment, restart according to actual deployment method.

---

### 11.3 View Prometheus Targets

Access Prometheus:

    http://monitor-node:9090/targets

Confirm:

    job="ceph"
    Status is UP

---

### 11.4 Prometheus Query Test

Query in Prometheus Web UI:

    ceph_health_status

Or:

    ceph_osd_up

If data is returned, it indicates successful collection.

---

## Twelve. Experiment Five: Grafana Ceph Dashboard

### 12.1 Grafana Data Source

Add Prometheus data source in Grafana:

    URL: http://monitor-node:9090

Test connection:

    Save & Test

---

### 12.2 Ceph Dashboard Sources

Common approaches:

1. Use Ceph Dashboard's built-in Grafana integration.
2. Use cephadm-deployed Grafana.
3. Import Ceph Dashboard from Grafana official or community.
4. Company-customized Ceph monitoring dashboard.

Dashboard should at least display:

- Cluster health status
- OSD up/in
- PG status
- Cluster capacity
- Pool capacity
- OSD usage
- OSD latency
- Recovery / Backfill
- MON quorum
- MDS status
- RGW request status
- Node CPU / Memory / Disk / Network

---

### 12.3 Grafana Dashboard Design Principles

Do not create a single overview dashboard.

Recommend splitting:

| Dashboard | Content |
|---|---|
| Ceph Overview | Cluster health, capacity, OSD, PG |
| OSD Details | OSD status, latency, capacity, PG distribution |
| Pool Details | Pool capacity, object count, PG, growth trend |
| CephFS | MDS, clients, metadata/data pool |
| RGW | Request count, error rate, Bucket, objects, traffic |
| Kubernetes CSI | PVC, PV, FailedMount, CSI Pod |
| Node Resources | CPU, memory, disk, network |

---

## Thirteen. Understanding Ceph Core Metrics

### 13.1 Cluster Health Metrics

Common metrics:

    ceph_health_status

Common value meanings:

| Value | Meaning |
|---|---|
| 0 | HEALTH_OK |
| 1 | HEALTH_WARN |
| 2 | HEALTH_ERR |

Alert recommendations:

    HEALTH_WARN alerts for 5 minutes.
    HEALTH_ERR alerts for 1 minute as critical alerts.

---

### 13.2 OSD Status Metrics

Common metrics: /think

ceph_osd_up
ceph_osd_in

Meaning:

| Metric | Description |
|---|---|
| ceph_osd_up | Whether the OSD process is online |
| ceph_osd_in | Whether the OSD is participating in data distribution |

Common anomalies:

    up = 0, in = 1

Indicates:

    OSD is down but still in.
    May cause degraded status.

---

### 13.3 PG Status Metrics

Common focus:

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

PG anomaly alerts have high priority.

---

### 13.4 Capacity Metrics

Focus:

- Cluster total capacity
- Cluster used capacity
- Cluster utilization
- OSD utilization
- Pool usage
- Pool growth trend
- nearfull / full status

Capacity alerts must be addressed proactively.

Do not wait until full to notify.

---

### 13.5 OSD Latency Metrics

Focus:

- commit latency
- apply latency
- OSD op latency
- slow ops

If an OSD's latency consistently exceeds other OSDs, investigate:

- Disk
- Network
- Node load
- recovery/backfill
- OSD failure

---

### 13.6 MON / MGR Metrics

Focus:

- MON quorum
- MON count
- MGR active
- MGR standby
- MGR module status
- Dashboard / Prometheus service status

---

### 13.7 MDS / CephFS Metrics

Focus:

- Active MDS count
- Standby MDS count
- MDS request latency
- MDS slow request
- CephFS client count
- metadata pool status
- data pool status

---

### 13.8 RGW Metrics

Focus:

- Whether RGW daemon is running
- Request volume
- 4xx error rate
- 5xx error rate
- Upload/download traffic
- Request latency
- Bucket count
- Object count
- User quota
- Bucket quota

RGW production monitoring typically requires combining:

    Ceph metrics
    RGW logs
    Nginx / LB metrics
    S3 client error rate

---

## FourteenI don't know.Experiment Six: Basic Alert Rule Examples

### 14.1 Prometheus Alert Rule File

Example path:

    /etc/prometheus/rules/ceph-alerts.yml

Rule example:

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
              summary: "Ceph cluster in HEALTH_WARN"
              description: "Ceph cluster HEALTH_WARN has persisted for 5 minutes, please run ceph health detail for details."

          - alert: CephHealthError
            expr: ceph_health_status == 2
            for: 1m
            labels:
              severity: critical
              service: ceph
            annotations:
              summary: "Ceph cluster in HEALTH_ERR"
              description: "Ceph cluster HEALTH_ERR, please immediately check ceph -s and ceph health detail."

---

### 14.2 OSD Down Alert

Example:

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
              summary: "Ceph OSD down"
              description: "OSD {{ $labels.ceph_daemon }} has been down for over 2 minutes, please check ceph osd tree and corresponding node."

          - alert: CephOSDOut
            expr: ceph_osd_in == 0
            for: 5m
            labels:
              severity: warning
              service: ceph
            annotations:
              summary: "Ceph OSD out"
              description: "OSD {{ $labels.ceph_daemon }} is currently in out state, please confirm if it's planned maintenance."

---

### 14.3 Capacity Alert

Example: /think

groups:
  - name: ceph-capacity-alerts
    rules:
      - alert: CephOSDUsageHigh
        expr: ceph_osd_stat_bytes_used / ceph_osd_stat_bytes * 100 > 80
        for: 10m
        labels:
          severity: warning
          service: ceph
        annotations:
          summary: "Ceph OSD usage exceeds 80%"
          description: "OSD {{ $labels.ceph_daemon }} usage exceeds 80%, please monitor capacity growth and expansion plan."

      - alert: CephOSDUsageCritical
        expr: ceph_osd_stat_bytes_used / ceph_osd_stat_bytes * 100 > 85
        for: 5m
        labels:
          severity: critical
          service: ceph
        annotations:
          summary: "Ceph OSD usage exceeds 85%"
          description: "OSD {{ $labels.ceph_daemon }} usage exceeds 85%, nearfull/full risk exists, please handle immediately."

Note:

  - Metric label names may differ across Ceph versions.
  - Actual rules need to be validated in Prometheus by querying metrics and labels first.
  - Do not copy directly to production without verification.

---

### 14.4 PG Abnormal Alert

Example idea:

  - Set alerts for PG states like degraded / undersized / inactive / inconsistent.

Pseudo example:

  - name: ceph-pg-alerts
    rules:
      - alert: CephPGDegraded
        expr: ceph_pg_degraded > 0
        for: 5m
        labels:
          severity: warning
          service: ceph
        annotations:
          summary: "Ceph has degraded PG"
          description: "Degraded PG exists, please check ceph -s, ceph health detail, ceph pg stat."

      - alert: CephPGInactive
        expr: ceph_pg_inactive > 0
        for: 1m
        labels:
          severity: critical
          service: ceph
        annotations:
          summary: "Ceph has inactive PG"
          description: "Inactive PG exists, may affect business IO, please handle immediately."

      - alert: CephPGInconsistent
        expr: ceph_pg_inconsistent > 0
        for: 1m
        labels:
          severity: critical
          service: ceph
        annotations:
          summary: "Ceph has inconsistent PG"
          description: "Inconsistent PG exists, please carefully investigate, do not blindly repair."

Note:

  - Metric names must match actual Prometheus query results.
  - If current version lacks direct ceph_pg_degraded metric, use PG state metric combinations.

---

## FifteenI don't know.Experiment Seven: Alertmanager Notification Chain

### 15.1 Alertmanager Role

Alertmanager is responsible for:

- Alert grouping
- Alert deduplication
- Alert suppression
- Alert silence
- Notification sending
- Routing alerts by severity level

Do not let Prometheus directly send all alerts without thinking.

---

### 15.2 Alert Level Classification

Recommended levels:

| Level | Example | Notification Method |
|---|---|---|
| critical | HEALTH_ERR, OSD full, PG inactive, MON quorum anomaly | Phone, IM, SMS, on-call group |
| warning | HEALTH_WARN, OSD nearfull, PG degraded, MDS standby missing | IM, email |
| info | Configuration suggestions, low-risk warnings | Email, ticket |

---

### 15.3 Alertmanager Routing Example

Example:

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
    webhook_configs:
      - url: 'http://127.0.0.1:5001/critical'

  - name: 'warning-webhook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/warning'

Note:

  Webhook addresses should be replaced with the company's actual notification service.
  Supported services include Enterprise WeChat, Feishu, DingTalk, SMS platforms, or self-developed alerting platforms.

---

### 15.4 Recommended Alert Content

Alerts should at least include:

- Alert name
- Alert level
- Cluster name
- Instance or object
- Current value
- Duration
- Impact description
- Initial handling suggestions
- Grafana link
- Prometheus link

---

## SixteenI don't know.Experiment 8: Verifying OSD down Alert

### 16.1 High-risk Reminder

Do not arbitrarily stop OSD in production environments.

Recommend testing only in test environments.

Before verification, confirm:

    ceph -s
    ceph osd tree
    ceph pg stat

---

### 16.2 Stop a Test OSD in Test Environment

Check OSD:

    ceph osd tree

Stop test OSD:

    ceph orch daemon stop osd.0

Observe:

    ceph -s
    ceph osd tree

Prometheus / Alertmanager should trigger OSD down alert.

---

### 16.3 Recover OSD

Start:

    ceph orch daemon start osd.0

Observe:

    ceph -s
    ceph osd tree
    ceph pg stat

Goals:

    OSD up/in
    PG active+clean
    Alert automatically recovered

---

### 16.4 Verification Points

| Verification Item | Result |
|---|---|
| Did Prometheus trigger an alert? | Yes / No |
| Did Alertmanager send a notification? | Yes / No |
| Is the alert content readable? | Yes / No |
| Does the alert recover after OSD recovery? | Yes / No |
| Did an alert storm occur? | Yes / No |

---

## SeventeenI don't know.Experiment 9: Capacity Alert Design

### 17.1 Capacity Alerts Should Not Be Verified by Filling Up

Do not fill up Ceph clusters to verify capacity alerts.

Correct approach:

- Test in test environment with lowered alert thresholds.
- Temporarily lower thresholds using expressions.
- Manually verify using Prometheus expression.
- Use test metrics or small-scale test pools.

---

### 17.2 Recommended Capacity Alert Tiering

| Usage Rate | Level | Handling |
|---|---|---|
| 70% | info | Monitor growth trend |
| 80% | warning | Plan expansion |
| 85% | critical | Expand or clean up as soon as possible |
| nearfull | critical | Immediate action required |
| full | disaster | Severe failure, may reject writes |

---

### 17.3 Capacity Alert Handling Suggestions

Do not directly delete unknown data after alerts.

Handling steps:

    1. ceph df to check Pool usage.
    2. ceph osd df to check OSD usage.
    3. Determine if overall capacity is insufficient or local imbalance.
    4. Identify the Pool with the fastest growth.
    5. Check RBD snapshots, RGW Buckets, CephFS directories.
    6. Clean up clearly unused data.
    7. Plan expansion.
    8. Do not simply raise full threshold to hide issues.

---

## EighteenI don't know.Experiment 10: PG Abnormality Alert Design

### 18.1 PG Alert Priority

| PG Status | Alert Level |
|---|---|
| degraded | warning |
| undersized | warning / critical, depending on quantity and duration |
| inactive | critical |
| incomplete | critical |
| inconsistent | critical |
| stale | critical |
| peering not recovered for a long time | warning / critical |

---

### 18.2 Alert Handling Suggestions

After receiving PG alerts:

    ceph -s
    ceph health detail
    ceph pg stat
    ceph osd tree

If you know the PG ID:

    ceph pg <pg-id> query
    ceph pg map <pg-id>

Do not immediately execute:

    ceph pg repair

Repair is only suitable for specific scenarios and must be analyzed first.

---

## NineteenI don't know.Experiment 11: CephFS / MDS Monitoring

### 19.1 CephFS Monitoring Goals

If using CephFS, must monitor:

- Existence of active MDS
- Existence of standby MDS
- MDS slow requests
- Number of CephFS clients
- Metadata pool capacity
- Data pool capacity
- MDS pressure from large number of small files
- CephFS PVC mount failures

---

### 19.2 Inspection Commands

    ceph fs status
    ceph mds stat
    ceph orch ps --daemon_type mds
    ceph health detail

---

### 19.3 MDS Alert Suggestions

| Alert | Level |
|---|---|
| Missing active MDS | critical |
| Missing standby MDS | warning |
| MDS slow request | warning / critical |
| Metadata pool nearfull | critical |
| MDS daemon down | critical |

---

## TwentyI don't know.Experiment 12: RGW Monitoring

### 20.1 RGW Monitoring Goals

If using RGW, must monitor:

- Is the RGW daemon running  
- Number of RGW instances  
- Are RGW ports accessible  
- Is Nginx / LB healthy  
- Is HTTPS certificate expired  
- 4xx error rate  
- 5xx error rate  
- Request latency  
- Upload/download throughput  
- Bucket count  
- Object count  
- User quota  
- Bucket quota  
- RGW Pool capacity  

---

### 20.2 Inspection Commands

    ceph orch ps --daemon_type rgw  
    ceph orch ls --service_type rgw  
    radosgw-admin user list  
    radosgw-admin bucket list  
    radosgw-admin bucket stats --bucket=<bucket-name>  
    ceph df  
    ceph osd pool ls | grep rgw  

Port testing:  

    curl -I http://10.0.0.31:7480  

S3 testing:  

    aws --profile ceph-rgw --endpoint-url ${RGW_ENDPOINT} s3 ls  

---

### 20.3 RGW Alert Recommendations

| Alert | Level |  
|---|---|  
| All RGW instances unavailable | critical |  
| Single RGW instance down | warning |  
| High RGW 5xx error rate | critical |  
| Abnormally high RGW 4xx error rate | warning |  
| Abnormal growth of Bucket objects | warning |  
| RGW Pool nearfull | critical |  
| HTTPS certificate will expire | warning |  

---

## Twenty-one, Experiment Thirteen: Kubernetes CSI Monitoring

### 21.1 RBD CSI Monitoring Targets

Monitoring objects:  

- Pod status in ceph-csi Namespace  
- csi-rbdplugin-provisioner  
- csi-rbdplugin DaemonSet  
- PVC Pending  
- PV Released / Failed  
- VolumeAttachment anomalies  
- Pod FailedMount  
- kubelet mount errors  
- k8s-rbd Pool capacity  
- RBD Image count  

---

### 21.2 CephFS CSI Monitoring Targets

Monitoring objects:  

- csi-cephfsplugin-provisioner  
- csi-cephfsplugin DaemonSet  
- PVC Pending  
- Pod FailedMount  
- CephFS Subvolume count  
- MDS status  
- metadata/data pool capacity  

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

### 21.4 CSI Alert Recommendations

| Alert | Level |  
|---|---|  
| ceph-csi controller unavailable | critical |  
| ceph-csi node plugin unavailable | critical |  
| PVC Pending exceeds 5 minutes | warning |  
| Pod FailedMount | warning / critical |  
| VolumeAttachment anomalies | warning |  
| CSI logs withMass errors | warning |  
| k8s-rbd Pool nearfull | critical |  

---

## Twenty-two, Experiment Fourteen: Alert Level and Noise Reduction

### 22.1 Why Need Alert Noise Reduction

Unfiltered alerts can lead to:  

- Too many alerts to be ignored  
- Critical issues being buried  
- On-call fatigue  
- Alert groups becoming noise groups  
- Slower incident response  

---

### 22.2 Alert Grouping

Recommend grouping by the following dimensions:  

    cluster  
    service  
    alertname  
    severity  

Example:  

    Merge multiple PG degraded alerts in the same cluster into one alert group.  
    Merge multiple OSD anomalies on the same node into a single notification.  
    A large number of PVC FailedMount alerts at the same time can be grouped as storage failure events.  

---

### 22.3 Alert Suppression

Example:  

    If CephHealthError has already triggered, suppress some low-level CephHealthWarn alerts.  
    If MON quorum is abnormal, some derived alerts may be suppressed.  
    If an entire node is down, suppress multiple daemon down alerts on that node.  

---

### 22.4 Alert Silence

Set silence during maintenance periods.  

Applicable scenarios:  

- Planned OSD node restart  
- Planned Ceph upgrade  
- Planned network maintenance  
- Planned disk replacement  
- Planned expansion or migration  

Notes:  

    Silence must have an expiration time.  
    Cannot permanently silence.  
    Confirm alert recovery after maintenance.  

---

### 22.5 Alert Repeat Interval

Recommendations:  

| Alert Level | repeat_interval |  
|---|---|  
| critical | 30 minutes |  
| warning | 4 hours |  
| info | 12 hours or longer |  

Adjust according to the actual on-call schedule of the enterprise.  

---

## Twenty-three, Production Monitoring Dashboard Baseline

### 23.1 Ceph Overview Dashboard

Must display:  

- Ceph health  
- MON quorum  
- MGR active  
- OSD up/in  
- PG active+clean ratio  
- Count of degraded / inactive / inconsistent PGs  
- Cluster capacity  
- OSD maximum usage rate  
- Pool usage ranking  
- Recovery / Backfill status  
- Slow ops  

---

### 23.2 OSD Dashboard

Must display: /think

- OSD up/in status
- OSD usage
- OSD commit/apply latency
- OSD op latency
- OSD PG count
- OSD node location
- OSD disk IO
- OSD network traffic

---

### 23.3 Pool Overview

Must display:

- Pool capacity
- Pool object count
- Pool usage
- Pool PG count
- Pool replica count
- Pool growth trend
- Pool application type

---

### 23.4 CephFS Overview

Must display:

- MDS active/standby
- MDS request latency
- MDS slow request
- Client count
- metadata pool capacity
- data pool capacity
- CephFS PVC status, if integrated with K8s

---

### 23.5 RGW Overview

Must display:

- RGW instance status
- Request volume
- 4xx / 5xx error rate
- Request latency
- Upload/download traffic
- Bucket count
- Object count
- Quota usage
- Certificate expiration time, entry-level monitoring

---

### 23.6 Kubernetes CSI Overview

Must display:

- ceph-csi Pod status
- PVC Pending count
- PV status
- VolumeAttachment anomalies
- Pod FailedMount events
- k8s-rbd Pool capacity
- CephFS Subvolume count

---

##24,Production Alert Baseline

### 24.1 Mandatory Alerts

The following must be alerted:

- Ceph HEALTH_ERR
- Ceph HEALTH_WARN persistent
- OSD down
- OSD full / nearfull
- PG inactive
- PG incomplete
- PG inconsistent
- MON quorum anomaly
- Missing active MDS
- All RGW unavailable
- CSI controller unavailable
- Large number of PVC Pending
- Large number of Pod FailedMount
- Capacity exceeding 80% / 85%
- Slow ops persistent
- Dashboard / RGW HTTPS certificate expiration

---

### 24.2 Recommended Alerts

The following are recommended alerts:

- MGR standby missing
- MDS standby missing
- Single RGW instance down
- OSD usage imbalance
- Pool growth anomaly
- RBD snapshot count anomaly
- RGW Bucket object count anomaly growth
- ceph-csi log errors increasing
- Node disk latency high
- Node time sync anomaly

---

##25,Monitoring Inspection Template

### 25.1 Daily Monitoring Check

| Check item | Result |
|---|---|
| Ceph health |  |
| OSD up/in |  |
| PG active+clean |  |
| Highest capacity OSD |  |
| Highest capacity Pool |  |
| Presence of slow ops |  |
| Presence of recovery/backfill |  |
| MDS status |  |
| RGW status |  |
| CSI status |  |
| Presence of unprocessed alerts |  |

---

### 25.2 Weekly Monitoring Check

| Check item | Result |
|---|---|
| Capacity growth trend |  |
| Pool growth ranking |  |
| OSD latency trend |  |
| Alert count trend |  |
| Repeated alert ranking |  |
| Alert misreporting |  |
| Need to adjust threshold |  |
| Presence of long-term silence |  |
| Presence of unprocessed warning |  |

---

### 25.3 Monthly Monitoring Governance

| Check item | Result |
|---|---|
| Alert rule review |  |
| Dashboard review |  |
| Grafana overview review |  |
| Alertmanager routing review |  |
| On-call notification chain test |  |
| Certificate expiration check |  |
| Capacity expansion plan |  |
| Monitoring blind spots supplement |  |

---

##26,Common Issues and Troubleshooting

### 26.1 Prometheus Can't Fetch Ceph Metrics

Troubleshoot:

    ceph mgr services
    ceph mgr module ls
    curl http://10.0.0.31:9283/metrics
    ss -lntp | grep 9283

Check Prometheus:

    Targets page whether UP
    prometheus.yml configuration correctness
    Network reachability
    Firewall rule allowance

---

### 26.2 Grafana Has No Data

Troubleshoot:

    Whether Prometheus has data
    Whether Grafana data source is correct
    Whether Dashboard query matches current metrics
    Whether metric label matches template variable
    Whether time range is correct

---

### 26.3 Alerts Not Triggered

Troubleshoot:

    Whether Prometheus rule is loaded
    Whether expression has results
    Whether for time is not met
    Whether labels match Alertmanager routing
    Whether Alertmanager is reachable

Check Prometheus rules page.

---

### 26.4 Alerts Triggered But No Notifications

Troubleshoot:

    Alertmanager route
    receiver configuration
    webhook address
    Network connectivity
    Notification platform token
    silence existence
    inhibition suppression

---

### 26.5 Too Many Alerts

Handling:

    Reasonably set for time.
    Reasonably group by group_by.
    Set repeat notification interval.
    Use inhibition to suppress derived alerts.
    Distinguish between warning and critical.
    Use silence during maintenance windows.
    Adjust thresholds or delete meaningless alerts.

---

##27,Production Environment Notes /think

1. Do not rely solely on manual ceph -s.
2. Do not only look at the Dashboard without configuring alerts.
3. Do not monitor Ceph without monitoring node resources.
4. Do not monitor only the cluster total capacity without monitoring OSD and Pool.
5. Do not ignore PG inactive / incomplete / inconsistent.
6. Do not wait until full before alerting.
7. Do not let Alertmanager alerts go unacknowledged.
8. Do not let alert groups remain full of false positives long-term.
9. Do not permanently silence alerts.
10. Do not deploy alert rules without verification.
11. Do not directly apply test environment alerts to production.
12. Do not ignore Kubernetes CSI's PVC / FailedMount alerts.
13. Do not ignore RGW entry layer HTTPS certificate alerts.
14. Do not ignore MDS standby missing.
15. Alerts must be acknowledged, documented, and reviewed.

---

## 28. Advanced SRE Methodology

### 28.1 Monitoring is not a dashboard, it is a response system

Grafana dashboards can only show "visibility".

A true production monitoring system should also include:

    Metric collection
    Alert rules
    Alert routing
    On-call response
    Incident handling
    Post-mortem analysis
    Threshold optimization

---

### 28.2 Alerts must guide action

Good alerts should tell on-call personnel:

    What is wrong?
    Which services are affected?
    What is the severity?
    Which commands should be checked first?
    Is immediate action required?
    Is there a known handling document?

Poor alerts are just:

    There is a problem.

---

### 28.3 Capacity trends are more important than current capacity

Ceph capacity alerts should not only look at current percentages.

Also check:

    7-day growth
    30-day growth
    Which Pool is growing fastest
    Which business growth is abnormal
    Estimated days to reach 80% / 85%

Advanced SREs should plan for expansion in advance, not wait until nearfull to fix.

---

### 28.4 Alert severity determines response efficiency

All alerts being critical equals no critical alerts.

Recommend:

    Affects business availability or data security: critical
    Risky but can be observed short-term: warning
    Suggest optimization or trend reminder: info

---

### 28.5 Monitoring must cover the entire chain

Ceph-related issues may occur in:

    Business applications
    Kubernetes PVC
    CSI
    RBD / CephFS / RGW
    Ceph services
    OSD / PG / Pool
    Node disks
    Network
    Time synchronization
    Entry Nginx / LB

Only monitoring Ceph itself is incomplete.

---

## 29. Interview Answer Structure

If asked:

    How to do monitoring and alerting in Ceph production environment?

You can answer:

    I will divide Ceph monitoring into metric collection, visualization, alert notification, and response closure layers. On Ceph side, I will enable MGR's prometheus module, exposing metrics via 9283 metrics port, Prometheus handles metric collection, Grafana provides cluster overview, OSD, Pool, CephFS, RGW, CSI dashboards, Alertmanager handles alert grouping, deduplication, suppression, and notification.
    Core monitoring metrics include Ceph health, MON quorum, MGR active, OSD up/in, PG active+clean, degraded, inactive, inconsistent, cluster capacity, OSD usage, Pool capacity growth, OSD latency, slow ops, recovery/backfill, etc.
    If using CephFS, also monitor MDS active/standby, MDS slow request, metadata pool and data pool. If using RGW, monitor RGW instances, S3 access, 4xx/5xx, request latency, Bucket/Object growth, quotas, and entry HTTPS certificate. If integrated with Kubernetes, also monitor ceph-csi Pod, PVC Pending, PV status, VolumeAttachment, and Pod FailedMount.
    Alerts will be graded as critical, warning, info. For example, HEALTH_ERR, OSD full, PG inactive/incomplete, MON quorum anomalies, active MDS missing, all RGW unavailable are critical; OSD nearfull, PG degraded, MDS standby missing, single RGW down are warning.
    I will not only do dashboards, but also focus on whether alerts are truly reaching people, whether handling documents exist, whether duplicate alerts are noise-reduced, whether maintenance window silence exists, and post-incident review and threshold optimization.

---

## 30. Summary

This article mainly organizes the construction of Ceph monitoring and alerting system:

1. Ceph production environment cannot rely solely on manual checks; monitoring and alerts must be established.
2. Ceph MGR prometheus module can expose Ceph metrics.
3. Prometheus handles metric collection and alert calculation.
4. Grafana handles Ceph cluster, OSD, Pool, CephFS, RGW, CSI dashboards.
5. Alertmanager handles alert grouping, deduplication, suppression, and notification.
6. Core metrics include health, OSD up/in, PG status, capacity, latency, slow ops.
7. Capacity alerts should be proactive, not wait until full to handle.
8. PG inactive, incomplete, inconsistent are high-priority alerts.
9. CephFS requires focus on MDS active/standby and metadata pool.
10. RGW requires monitoring instance status, error rate, latency, Bucket, Object, and certificate.
11. Kubernetes CSI requires monitoring PVC, PV, VolumeAttachment, CSI Pod, and FailedMount.
12. Alerts must be graded; not all alerts should be critical.
13. Alerts need noise reduction, grouping, suppression, and maintenance window silence.
14. Monitoring system should form a closed loop of discovery, notification, handling, review, and optimization.
15. Advanced SREs focus not only on "whether there is a dashboard" but whether alerts can truly guide action.

---

## 31. Reference Documents

Ceph monitoring cluster:

    https://docs.ceph.com/en/latest/rados/operations/monitoring/

Ceph health check:

    https://docs.ceph.com/en/latest/rados/operations/health-checks/

Ceph Dashboard:

    https://docs.ceph.com/en/latest/mgr/dashboard/

Ceph Prometheus module:

    https://docs.ceph.com/en/latest/mgr/prometheus/

Cephadm monitoring service:

    https://docs.ceph.com/en/latest/cephadm/services/monitoring/

Prometheus official documentation:

    https://prometheus.io/docs/introduction/overview/

Prometheus alert rules:

    https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

Alertmanager Official Documentation:

    https://prometheus.io/docs/alerting/latest/alertmanager/

Grafana Official Documentation:

    https://grafana.com/docs/

Kubernetes Persistent Volumes:

    https://kubernetes.io/docs/concepts/storage/persistent-volumes/