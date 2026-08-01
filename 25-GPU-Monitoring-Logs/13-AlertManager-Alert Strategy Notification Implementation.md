# 13-AlertManager-Alert Strategy and Notification Implementation

## Document Overview

This document systematically organizes the alert processing mechanism, core concepts, configuration structure, routing strategies, grouping, deduplication, suppression, silencing, notification channels, template design, Kubernetes implementation, GPU alert scenarios, production environment alert governance, and common troubleshooting for Prometheus AlertManager.

This document is not simply about "how to send alerts to email or DingTalk", but rather about understanding from an operations/SRE perspective:

- The relationship between Prometheus Alert Rule and AlertManager;
- The complete alerting pipeline from trigger to notification;
- Why grouping, deduplication, suppression, and silencing are needed;
- How to design warning/critical/info alert grading;
- How to route different alerts to different teams;
- How to avoid alert storms;
- How to configure email, webhook, enterprise WeChat, DingTalk, etc. notification channels;
- How to design alert templates;
- How to attach Dashboard, Runbook, Namespace, Pod, Node, GPU, etc. context to alerts;
- How to manage AlertManager via kube-prometheus-stack in Kubernetes;
- How to implement Node, Pod, Service, GPU, and log-based alerts into production workflows;
- How to address the issue of "many alerts but no one to handle them".

This document is suitable for study after completing the following content:

- 11-Prometheus-Architecture and Core Metrics Analysis
- 12-Grafana-Dashboard Setup and Custom Monitoring
- 08-GPU-Monitoring and Alert Integration
- 09-GPU-Fault Diagnosis Cases and Practical Exercises
- 10-GPU-Operations Experiment Environment Setup and Verification

---

## Tags

#Prometheus #AlertManager #Police! #PoliceGovernance #AlertRule #PrometheusRule #Kubernetes #Grafana #Webhook #CorporateWisdom #Nail. #GpuSurveillance #SRE #Observation

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Foundation/13-AlertManager-Alert Strategy and Notification Implementation.md

---

## One, Why AlertManager is Needed

Prometheus can determine if metrics are abnormal through Alerting Rules.

For example:

    Node CPU usage exceeds 90%
    Pod Pending exceeds 10 minutes
    GPU temperature exceeds 80°C
    GPU XID error occurs
    Service 5xx error rate exceeds 5%
    P99 latency exceeds 2 seconds

However, Prometheus is not suitable for handling complex notification logic directly.

If all alerts are sent directly by Prometheus, many issues will arise:

- Duplicate alerts sent;
- A single fault triggers a chain of alerts;
- Multiple nodes failing causes an alert storm;
- Alerts from different teams cannot be separated;
- Maintenance windows cannot temporarily silence alerts;
- Upstream failures cause downstream false positives;
- Warning and critical alerts cannot be distinguished;
- Alerts lack context;
- Notification channels cannot be uniformly managed;
- Deduplication, grouping, suppression, and routing cannot be uniformly handled.

AlertManager's role is to solve these issues.

It sits between Prometheus and notification channels.

Complete pipeline:

    Exporter / Application /metrics
      ↓
    Prometheus scrapes metrics
      ↓
    Prometheus Alert Rule determines anomalies
      ↓
    Generates Alert
      ↓
    Sends to AlertManager
      ↓
    AlertManager groups/deduplicates/suppresses/silences/routes
      ↓
    Sends to Email/Webhook/Enterprise WeChat/DingTalk/Slack/OnCall
      ↓
    On-call personnel handle alerts
      ↓
    Runbook/Dashboard/Logs/kubectl troubleshooting
      ↓
    Fault recovery and post-mortem analysis

Thus, AlertManager is not a "message sending tool", but a production alert governance center.

---

## Two, Relationship Between Prometheus Alert Rule and AlertManager

### 2.1 What Prometheus Alert Rule Handles

Prometheus Alert Rule determines whether a PromQL condition is met.

For example:

    Whether CPU exceeds threshold
    Whether Pod is Pending
    Whether GPU is overheating
    Whether service error rate increases
    Whether Target is Down

Example:

    - alert: NodeHighCPUUsage
      expr: node:cpu_usage:percent > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node CPU usage is too high"
        description: "Node {{ $labels.instance }} CPU usage is above 90% for 5 minutes."

Prometheus handles:

- Regularly calculating rules;
- Determining if an alert is active;
- Checking if the for duration is met;
- Generating alerts;
- Sending alerts to AlertManager.

### 2.2 What AlertManager Handles

AlertManager does not calculate PromQL.

It handles alerts that have already been generated.

AlertManager handles:

- Grouping;
- Deduplication;
- Suppression;
- Silencing;
- Routing;
- Notification;
- Template rendering;
- High availability deduplication;
- Notification failure retries.

AlertManager should not directly replace Prometheus Alert Rule.

Correct relationship:

    Prometheus:
        Detects issues, generates alerts.

    AlertManager:
        Manages alerts, sends notifications.

### 2.3 Relationship with Grafana Alerting

Grafana can also create alerts.

However, in Kubernetes infrastructure and SRE platform alerting, it's typically recommended:

    Infrastructure alerts:
        PrometheusRule + AlertManager

    Business temporary or Dashboard-level alerts:
        Can use Grafana Alerting

It's not recommended to configure the same alert in both Prometheus and Grafana.

Otherwise, issues like:

- Duplicate notifications;
- Inconsistent alert definitions;
- Difficulty in silencing;
- Unclear alert ownership;
- On-call personnel not knowing which alert to handle.

may occur.

---

## III. AlertManager Core Capabilities

### 3.1 Grouping: Grouping

Grouping is used to merge similar alerts into a single notification.

For example, after a node fails, it may trigger:

- NodeDown
- KubeletDown
- NodeExporterDown
- PodUnavailable
- PodNotReady
- ServiceErrorRateHigh
- GPUMetricMissing

Without grouping, on-call personnel may receive dozens or even hundreds of messages.

After grouping, similar alerts can be merged into a single notification.

For example, group by:

    alertname
    cluster
    node

### 3.2 Deduplication: Deduplication

Prometheus continuously sends firing alerts to AlertManager.

AlertManager needs to identify the same alert to avoid sending duplicate notifications repeatedly.

Deduplication relies on alert labels.

If two alerts have identical labels, they are typically considered the same alert instance.

Thus, alert label design is very important.

### 3.3 Routing: Routing

Routing determines where alerts are sent.

For example:

    severity=critical sent to OnCall
    severity=warning sent to operations group
    team=ai sent to AI team
    team=database sent to DBA
    namespace=prod sent to production on-call
    namespace=dev only sent to general group
    alertname=GPUXIDError sent to GPU operations team

Routing is based on label matching.

### 3.4 Inhibition: Inhibition

Inhibition is used to automatically suppress lower-level alerts when a higher-level alert exists.

A typical example:

    After NodeDown triggers
    Inhibit PodDown, TargetDown, GPUExporterDown, etc. alerts on the node

Reason:

    The node is already down, so Pod and Exporter anomalies on the node are results, not issues that need repeated notifications.

### 3.5 Silence: Silence

Silence is used to temporarily suppress certain alerts during maintenance windows.

For example:

    Planned maintenance on k8s-gpu-node01
    Upgrading NVIDIA Driver
    Restarting containerd
    The node will briefly be NotReady
    We don't want to generate a lot of alerts during this time

Create a silence:

    node="k8s-gpu-node01"
    duration=2h
    comment="GPU driver maintenance"

### 3.6 Notification Template: Notification Template

Templates are used to control the content of alert messages.

Good alert messages should include:

- Alert name
- Severity level
- Cluster
- Namespace
- Pod
- Node
- GPU
- Current value
- Duration
- Dashboard link
- Runbook link
- Handling suggestions
- Alert start time

Avoid sending just:

    CPU high

This type of alert has no value for handling.

---

## IV. Alert Lifecycle

Alerts in AlertManager typically have the following states:

    inactive
    pending
    firing
    resolved

### 4.1 inactive

Conditions are not met.

For example:

    CPU usage is below the threshold.

### 4.2 pending

Prometheus rule expressions have been satisfied, but have not reached `for` specified duration.

For example:

    CPU > 90%
    for: 5m

From the 1st minute to the 5th minute before, the alert is in pending.

### 4.3 firing

After the condition is continuously met and exceeds `for` time, the alert enters firing.

At this point, Prometheus will send the alert to AlertManager.

### 4.4 resolved

Conditions are no longer met, and the alert recovers.

AlertManager can send recovery notifications or not, depending on configuration and receiver capabilities.

### 4.5 Why "for" is Important

Do not write:

    expr: cpu_usage > 90

This instantaneous trigger can generate noise.

Should write:

    expr: cpu_usage > 90
    for: 5m

This filters short-term fluctuations.

The "for" time should vary for different alerts:

    NodeDown:
        2m - 5m

    CPU high:
        5m - 10m

    Disk space insufficient:
        10m - 30m

    GPU XID:
        1m

    GPU high temperature:
        2m - 5m

    Pod Pending:
        5m - 10m

---

## V. AlertManager Configuration File Structure

AlertManager configuration is usually YAML.

Core structure:

    global:
      resolve_timeout: 5m

    route:
      group_by: [...]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: default
      routes:
        - matchers:
            - severity="critical"
          receiver: oncall

    receivers:
      - name: default
        email_configs: []

      - name: oncall
        webhook_configs: []

    inhibit_rules:
      - source_matchers: []
        target_matchers: []
        equal: []

    templates:
      - /etc/alertmanager/templates/*.tmpl

Meanings of each part:

    global:
        Global configuration, such as SMTP, timeout settings, etc.

    route:
        Alert routing tree, determines where alerts are sent.

    receivers:
        Receivers, define notification methods like email, webhook, Slack, Enterprise WeChat, etc.

    inhibit_rules:
        Inhibition rules.

    templates:
        Notification template files.

---

## VI. Route Routing Rules Explained

### 6.1 Root Route

Example: /think

route:
  receiver: default
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

Meaning:

  receiver:
      Default receiver.

  group_by:
      Group by which labels.

  group_wait:
      How long to wait before sending the first notification for a new alert group.

  group_interval:
      How long to wait before sending another notification for the same alert group after new alerts are added.

  repeat_interval:
      How often to remind about the same alert when it is continuously firing.

### 6.2 group_by

Example:

  group_by: ['alertname', 'cluster', 'namespace']

Indicates that alerts with the same alertname, cluster, and namespace will be merged into one notification group.

Common grouping strategies:

Infrastructure alerts:

  ['alertname', 'cluster', 'node']

Kubernetes namespace alerts:

  ['alertname', 'cluster', 'namespace']

Business service alerts:

  ['alertname', 'cluster', 'namespace', 'service']

GPU alerts:

  ['alertname', 'cluster', 'node', 'gpu']

Do not group_by too many fields.

Adding pod, container, instance, and endpoint all into group_by will degrade grouping effectiveness.

### 6.3 group_wait

Indicates how long to wait before sending the first notification for a new alert group.

Example:

  group_wait: 30s

Meaning:

  Wait for related alerts to arrive together, then send them as a group.

If set too short:

  May send each alert individually.

If set too long:

  Critical alerts may experience notification delays.

Recommendation:

  critical:
      10s - 30s

  warning:
      30s - 2m

### 6.4 group_interval

Indicates how long to wait before sending another notification for the same alert group after it has already received a notification.

Example:

  group_interval: 5m

Avoid frequent notifications for the same alert group.

### 6.5 repeat_interval

Indicates how often to remind about the same alert when it is continuously firing.

Example:

  repeat_interval: 4h

If set too short:

  Continuous failures may cause screen flooding.

If set too long:

  Failures may be forgotten.

Recommendation:

  critical:
      30m - 2h

  warning:
      4h - 12h

  info:
      24h or no notification, only report

---

## SevenI don't know.matchers Matching Rules

AlertManager's new configuration recommends using matchers.

Example:

  routes:
    - receiver: sre-critical
      matchers:
        - severity="critical"

    - receiver: ai-team
      matchers:
        - team="ai"

    - receiver: gpu-team
      matchers:
        - alertname=~"GPU.*"

### 7.1 Exact Match

  severity="critical"

Indicates label `severity` equals `critical`.

### 7.2 Not Equal

  severity!="info"

### 7.3 Regular Expression Match

  alertname=~"GPU.*"

Matches:

  GPUHighTemperature
  GPUXIDError
  GPUMemoryUsageHigh

### 7.4 Regular Expression Not Match

  namespace!~"dev|test"

### 7.5 Multi-Condition Match

  matchers:
    - severity="critical"
    - namespace="prod"
    - team="sre"

Indicates all conditions must be met.

---

## EightI don't know.continue's Function

By default, AlertManager's routing tree will not continue matching subsequent peer routes after an alert matches a sub-route.

If you want to continue matching subsequent routes, set:

  continue: true

Example:

  routes:
    - receiver: sre-critical
      matchers:
        - severity="critical"
      continue: true

    - receiver: database-team
      matchers:
        - team="database"

Meaning:

  Critical alerts are first sent to SRE.
  If also team=database, they are also sent to DBA team.

Usage Notes:

  Misusing continue may cause duplicate notifications.
  Use with caution in production environments.

---

## NineI don't know.receivers Receivers

receiver defines specific notification channels.

Common receivers:

- email;
- webhook;
- Slack;
- PagerDuty;
- OpsGenie;
- Enterprise WeChat;
- DingTalk;
- Feishu;
- Self-developed OnCall;
- Self-developed alert platform.

AlertManager natively supports some channels and also supports webhook integration with any custom notification system.

### 9.1 email receiver Example /think

# 9. Email Receiver Example

receivers:
  - name: sre-email
    email_configs:
      - to: sre@example.com
        from: prometheus@example.com
        smarthost: smtp.example.com:587
        auth_username: prometheus@example.com
        auth_password: <password>
        require_tls: true
        send_resolved: true

Production Recommendations:

  - Do not store passwords in plain text in Git.
  - Use Secret management.
  - Email is suitable for warning, daily reports, and low-priority alerts.
  - Critical alerts are not recommended to rely solely on email.

### 9.2 Webhook Receiver Example

receivers:
  - name: sre-webhook
    webhook_configs:
      - url: http://alert-webhook.monitoring.svc:8080/alert
        send_resolved: true

Webhook Advantages:

- Can integrate with Enterprise WeChat
- Can integrate with DingTalk
- Can integrate with Feishu
- Can integrate with self-developed platforms
- Can perform secondary formatting
- Can archive alerts
- Can trigger automated responses

### 9.3 Enterprise WeChat / DingTalk / Feishu

AlertManager natively may not directly support all domestic IM platforms.

Common approach:

  AlertManager
    ↓
  webhook
    ↓
  alertmanager-webhook-adapter
    ↓
  Enterprise WeChat / DingTalk / Feishu robot

Or:

  AlertManager
    ↓
  Self-developed alert gateway
    ↓
  Multi-notification channels

Production Recommendations:

  - Do not let AlertManager directly handle complex business logic.
  - Domestic IM notifications are recommended to be uniformly adapted through webhook gateways.

### 9.4 Multi Receiver Example

receivers:
  - name: default
    webhook_configs:
      - url: http://alert-webhook.monitoring.svc:8080/default

  - name: sre-critical
    webhook_configs:
      - url: http://alert-webhook.monitoring.svc:8080/sre-critical
        send_resolved: true

  - name: ai-team
    webhook_configs:
      - url: http://alert-webhook.monitoring.svc:8080/ai-team
        send_resolved: true

  - name: email-warning
    email_configs:
      - to: sre-warning@example.com
        from: prometheus@example.com
        smarthost: smtp.example.com:587
        auth_username: prometheus@example.com
        auth_password: <password>
        send_resolved: true

---

## Ten, Inhibition Rules

### 10.1 Why Inhibition is Needed

When upstream failures occur, downstream will generate a large number of cascading alerts.

For example node failure:

  NodeDown
  KubeletDown
  NodeExporterDown
  PodDown
  PodNotReady
  TargetDown
  DCGMExporterDown

The real root cause is NodeDown.

Other alerts are just results.

At this time, inhibition can be used to suppress downstream alerts.

### 10.2 Inhibition Rule Example: Critical Inhibits Warning

inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal:
      - alertname
      - cluster
      - namespace

Meaning:

  When there exists a critical alert with the same alertname, cluster, and namespace,
  suppress the corresponding warning alert.

### 10.3 Inhibition Rule Example: NodeDown Inhibits Other Alerts on the Node

inhibit_rules:
  - source_matchers:
      - alertname="NodeDown"
    target_matchers:
      - severity=~"warning|critical"
    equal:
      - cluster
      - node

Meaning:

  When NodeDown occurs on the same cluster and node,
  suppress other warning/critical alerts on the node.

### 10.4 Inhibition Rule Example: KubeAPIDown Inhibits Component Alerts

inhibit_rules:
  - source_matchers:
      - alertname="KubeAPIDown"
    target_matchers:
      - alertname=~"Kube.*"
    equal:
      - cluster

Meaning:

  When Kubernetes API Server fails,
  suppress other Kubernetes component-derived alerts.

### 10.5 Inhibition Usage Notes

Do not over-inhibit.

Incorrect inhibition may cause important alerts to be suppressed.

Recommendations:

  - Only inhibit clearly related upstream/downstream relationships.
  - Each inhibition rule should have comments explaining its purpose.
  - Validate changes in test environment before production deployment.
  - Inhibition rules should be regularly reviewed.

---

## Eleven, Silence

### 11.1 Silence Applicable Scenarios

Silence is applicable for:

- Planned Maintenance;
- Version Upgrade;
- Node Restart;
- Driver Upgrade;
- GPU Node Maintenance;
- Pressure Testing;
- Drill;
- Temporary Shielding of Known Issues;
- Business Line Decommissioning and Migration.

### 11.2 When Silence is Not Appropriate

Do not use Silence to long-term mask real issues.

Wrong Practice:

    A Pod has been in CrashLoopBackOff for a long time
    Silence it for 30 days directly

Correct Practice:

    Fix the issue
    Or disable the corresponding alert rule
    Or adjust the threshold
    Or explicitly mark the service as decommissioned

### 11.3 Creating Silence

AlertManager Web UI:

    Silences
      ↓
    New Silence

Common Matchers:

    alertname="NodeHighCPUUsage"
    node="k8s-worker01"

Or:

    namespace="dev"
    severity="warning"

Must Fill:

    Duration
    Creator
    Comment

### 11.4 GPU Node Maintenance Silence Example

Maintaining GPU Node:

    node="k8s-gpu-node01"

Duration:

    2h

Notes:

    Upgrade NVIDIA Driver and restart containerd

Matcher Suggestions:

    cluster="prod"
    node="k8s-gpu-node01"

Do not silence directly:

    severity="critical"

Because this would suppress a too broad scope.

### 11.5 Silence Management Standards

Production Requirements:

    Each silence must have a reason.
    Each silence must have an expiration time.
    No permanent silence is allowed.
    No silence without a responsible person.
    Regularly clean up expired or long-term silences.
    Manually confirm alert recovery after maintenance.

---

## TwelveI don't know.Alert Severity Design

Alert severity classification is recommended to have at least three levels:

    critical
    warning
    info

### 12.1 critical

Indicates that immediate action is required.

Suitable for:

- Production service unavailable;
- All replicas unavailable;
- NodeDown;
- API Server unavailable;
- etcd unavailable;
- Severe error in production GPU XID;
- GPU temperature exceeding dangerous threshold;
- Large increase in 5xx errors in production business;
- P99 latency severely exceeding limits;
- Database master instance unavailable;
- Disk nearing full causing service risk.

Notification Methods:

    OnCall
    Phone call
    High-priority IM group
    PagerDuty / In-house on-call system

### 12.2 warning

Indicates that attention is needed, but not necessarily to interrupt current work.

Suitable for:

- Continuous high CPU usage;
- Continuous high memory usage;
- Disk usage exceeding 85%;
- Increased Pod restarts;
- High GPU memory usage;
- Elevated GPU temperature;
- Single replica unavailable but service still has redundancy;
- A Namespace quota approaching its limit.

Notification Methods:

    Operations group
    Email
    Handle during working hours
    Work order or pending task

### 12.3 info

Indicates informational or governance prompts.

Suitable for:

- Long-term low GPU utilization;
- Experimental Namespace resource usage;
- Certificate will expire in 30 days;
- Capacity trend reminder;
- Low-priority task failure;
- Abnormalities in non-production environments.

Notification Methods:

    Daily report
    Weekly report
    Low-priority group
    Not included in night-time OnCall

### 12.4 Severity Principles

Recommendations:

    Only enter critical if it affects production service availability.
    Enter warning for issues requiring human intervention but not urgent.
    Enter info for resource governance, capacity reminders, and non-real-time issues.

Do not set all alerts to critical.

Otherwise, critical will lose its meaning.

---

## ThirteenI don't know.Alert Label Design

### 13.1 Common Labels

Recommend that each alert includes at least:

    alertname
    severity
    cluster
    namespace
    pod
    node
    service
    team
    environment

GPU alerts add:

    gpu
    gpu_uuid
    gpu_model

Business alerts add:

    app
    service
    route
    method
    status

### 13.2 severity

Must use unified values.

Recommend:

    severity="critical"
    severity="warning"
    severity="info"

Do not mix:

    high
    error
    warn
    major
    P1

Unless the enterprise has its own unified standards.

### 13.3 team

Used for routing.

Examples:

    team="sre"
    team="ai"
    team="database"
    team="middleware"
    team="network"

### 13.4 environment

Used to distinguish environments.

Examples:

    environment="prod"
    environment="test"
    environment="dev"

Production environment alerts and test environment alerts must be routed separately.

### 13.5 cluster

In multi-cluster environments, the cluster label must be present.

Examples:

    cluster="shanghai-prod"
    cluster="shenzhen-prod"
    cluster="dev-k8s"

Without a cluster label, multi-cluster alerts will be hard to locate.

---

## FourteenI don't know.Alert Annotations Design

Annotations are used to display alert content.

Recommend including:

    summary
    description
    dashboard_url
    runbook_url

### 14.1 summary

Brief explanation.

Example:

    summary: "GPU Temperature Too High"

### 14.2 description

Detailed explanation.

Example:

    description: "GPU {{ $labels.gpu }} on node {{ $labels.node }} temperature is {{ $value }}°C for more than 5 minutes."

### 14.3 dashboard_url

Jump to Grafana.

Example:

    dashboard_url: "https://grafana.example.com/d/gpu-overview?var-node={{ $labels.node }}"

### 14.4 runbook_url

Jump to handling documentation.

runbook_url: "https://wiki.example.com/runbook/gpu-high-temperature"

### 14.5 Recommended Format

Alerts should NOT only contain:

    description: "CPU high"

They should include:

    Which cluster
    Which node
    Which Namespace
    Which Pod
    Current value
    Threshold
    Duration
    Handling documentation
    Dashboard link

---

## FifteenI don't know.PrometheusRule Example

The following is an example of PrometheusRule when using Prometheus Operator / kube-prometheus-stack in Kubernetes.

### 15.1 Node CPU High Alert

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: node-alert-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: node.rules
          rules:
            - alert: NodeHighCPUUsage
              expr: |
                100 - (
                  avg by (instance) (
                    rate(node_cpu_seconds_total{mode="idle"}[5m])
                  ) * 100
                ) > 90
              for: 5m
              labels:
                severity: warning
                team: sre
              annotations:
                summary: "Node CPU Usage is Too High"
                description: "Node {{ $labels.instance }} CPU usage is above 90% for more than 5 minutes."
                runbook_url: "https://wiki.example.com/runbook/node-high-cpu"

### 15.2 Pod Pending Alert

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: pod-alert-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: pod.rules
          rules:
            - alert: PodPendingTooLong
              expr: kube_pod_status_phase{phase="Pending"} == 1
              for: 10m
              labels:
                severity: warning
                team: sre
              annotations:
                summary: "Pod Pending Time is Too Long"
                description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been Pending for more than 10 minutes."
                runbook_url: "https://wiki.example.com/runbook/pod-pending"

### 15.3 Pod Restart Too Often Alert

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: pod-restart-alert-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: pod.restart.rules
          rules:
            - alert: PodRestartTooOften
              expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
              for: 5m
              labels:
                severity: warning
                team: sre
              annotations:
                summary: "Pod Restart Count is Too High"
                description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted more than 3 times in 10 minutes."
                runbook_url: "https://wiki.example.com/runbook/pod-restart"

### 15.4 GPU High Temperature Alert /think

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alert-rules
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: gpu.rules
      rules:
        - alert: GPUHighTemperature
          expr: DCGM_FI_DEV_GPU_TEMP > 80
          for: 5m
          labels:
            severity: warning
            team: ai-platform
          annotations:
            summary: "GPU Temperature is Too High"
            description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} temperature is above 80°C for more than 5 minutes."
            dashboard_url: "https://grafana.example.com/d/gpu-overview"
            runbook_url: "https://wiki.example.com/runbook/gpu-high-temperature"

### 15.5 GPU XID Alert

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: gpu-xid-alert-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: gpu.xid.rules
          rules:
            - alert: GPUXIDError
              expr: DCGM_FI_DEV_XID_ERRORS > 0
              for: 1m
              labels:
                severity: critical
                team: ai-platform
              annotations:
                summary: "GPU XID Error Occurred"
                description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} reported XID error value {{ $value }}. Check dmesg and GPU health immediately."
                dashboard_url: "https://grafana.example.com/d/gpu-overview"
                runbook_url: "https://wiki.example.com/runbook/gpu-xid-error"

---

## SixteenI don't know.AlertManager Basic Configuration Example

The following is a basic configuration suitable for a learning environment.

    global:
      resolve_timeout: 5m

    route:
      receiver: default-webhook
      group_by:
        - alertname
        - cluster
        - namespace
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        - receiver: critical-webhook
          matchers:
            - severity="critical"
          group_wait: 10s
          repeat_interval: 1h

        - receiver: warning-webhook
          matchers:
            - severity="warning"
          repeat_interval: 4h

        - receiver: info-webhook
          matchers:
            - severity="info"
          repeat_interval: 24h

    receivers:
      - name: default-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/default
            send_resolved: true

      - name: critical-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/critical
            send_resolved: true

      - name: warning-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/warning
            send_resolved: true

      - name: info-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/info
            send_resolved: false

    inhibit_rules:
      - source_matchers:
          - severity="critical"
        target_matchers:
          - severity="warning"
        equal:
          - alertname
          - cluster
          - namespace

---

## SeventeenI don't know.Production-Level Routing Design Example

### 17.1 Design Goals

Hope to achieve:

critical alerts are sent to SRE OnCall
GPU alerts are sent to AI Platform Team
Database alerts are sent to DBA
dev/test environments do not enter nighttime OnCall
info alerts only enter low priority channel
Default alerts enter SRE default group

### 17.2 Example Configuration

    route:
      receiver: sre-default
      group_by:
        - alertname
        - cluster
        - namespace
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

      routes:
        - receiver: sre-oncall-critical
          matchers:
            - severity="critical"
            - environment="prod"
          group_wait: 10s
          repeat_interval: 1h
          continue: true

        - receiver: ai-platform
          matchers:
            - team="ai-platform"

        - receiver: dba-team
          matchers:
            - team="database"

        - receiver: dev-warning
          matchers:
            - environment=~"dev|test"
            - severity=~"warning|info"

        - receiver: info-digest
          matchers:
            - severity="info"
          repeat_interval: 24h

    receivers:
      - name: sre-default
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/sre-default
            send_resolved: true

      - name: sre-oncall-critical
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/sre-critical
            send_resolved: true

      - name: ai-platform
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/ai-platform
            send_resolved: true

      - name: dba-team
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/dba
            send_resolved: true

      - name: dev-warning
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/dev-warning
            send_resolved: true

      - name: info-digest
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/info
            send_resolved: false

### 17.3 Notes

Use `continue: true` with caution.

For example, critical alerts are first sent to SRE OnCall, and if team=ai-platform, they will also be sent to the AI team.

If you don't want repeated notifications, do not use continue.

---

## EighteenI don't know.Notification Template Design

### 18.1 Why Templates Are Needed

Default notification formats are usually unsuitable for production.

Production alerts need to include sufficient context.

Templates can standardize the format.

### 18.2 Recommended Template Content

Alert messages should include:

    Alert status: FIRING / RESOLVED
    Alert name
    Severity level
    Cluster
    Environment
    Namespace
    Pod
    Node
    Service
    Current value
    Start time
    Dashboard link
    Runbook link
    Description
    Handling suggestions

### 18.3 Simplified Template Example

    {{ define "alert.title" }}
    [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }} - {{ .CommonLabels.severity }}
    {{ end }}

    {{ define "alert.message" }}
    {{ range .Alerts }}
    Alert name: {{ .Labels.alertname }}
    Severity level: {{ .Labels.severity }}
    Cluster: {{ .Labels.cluster }}
    Namespace: {{ .Labels.namespace }}
    Node: {{ .Labels.node }}
    Pod: {{ .Labels.pod }}
    Description: {{ .Annotations.description }}
    Dashboard: {{ .Annotations.dashboard_url }}
    Runbook: {{ .Annotations.runbook_url }}
    Start time: {{ .StartsAt }}
    {{ end }}
    {{ end }}

### 18.4 Template File Configuration

AlertManager configuration:

    templates:
      - /etc/alertmanager/templates/*.tmpl

Reference template in receiver:

    receivers:
      - name: webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/alert
            send_resolved: true

Notes:

    The actual message format of webhook is usually converted again by the webhook adapter.
    Native receivers like email, Slack, etc., have more direct support for templates.

## 19. Webhook Alert Gateway Design

### 19.1 Why a Webhook Gateway is Needed

Common notification channels in domestic production environments include:

- WeCom (Enterprise WeChat);
- DingTalk;
- Feishu;
- SMS;
- Phone call;
- Self-developed ticketing system;
- Self-developed OnCall.

AlertManager is not suitable for directly implementing all channel differences.

Recommended architecture:

    AlertManager
      ↓
    Webhook Gateway
      ↓
    WeCom / DingTalk / Feishu / Email / Ticketing / Phone call

### 19.2 Webhook Gateway Functions

Webhook Gateway can handle:

- Receiving AlertManager JSON;
- Formatting messages;
- Selecting channels based on severity;
- Selecting groups based on team;
- Adding @mentions;
- Writing to alert database;
- Creating tickets;
- Calling automation workflows;
- Rate limiting;
- Secondary deduplication;
- Retry on notification failure;
- Unified auditing.

### 19.3 AlertManager Webhook Payload

AlertManager sends JSON to webhook.

Typically includes:

    receiver
    status
    alerts
    groupLabels
    commonLabels
    commonAnnotations
    externalURL

Webhook service needs to parse these fields.

### 19.4 Production Recommendations

Webhook Gateway should have:

    High availability
    Logging
    Retry on failure
    Rate limiting
    Authentication
    Auditing
    Alert database storage
    Channel circuit breaking
    Template management

Do not let a temporary script handle production alert notification chains.

---

## 20. Implementation Approach for WeCom / DingTalk / Feishu

### 20.1 WeCom

Common approach:

    AlertManager
      ↓
    Webhook Adapter
      ↓
    WeCom Robot Webhook

Notes:

- WeCom robots have message format requirements;
- Robot webhook address must be kept confidential;
- May have frequency limits;
- Production recommends forwarding through internal alert gateway;
- Do not write robot URL into public Git.

### 20.2 DingTalk

DingTalk robots typically require:

- Webhook URL;
- Keywords;
- Signature;
- Message format;
- Frequency limits.

Recommended to process signature and format through webhook adapter.

### 20.3 Feishu

Feishu robots also typically connect via webhook.

Notes:

- Signature;
- Message card format;
- Rich text format;
- Frequency limits.

### 20.4 General Principles

    AlertManager only handles sending structured alerts to webhook.
    Webhook gateway handles adapting domestic IM.
    Sensitive tokens use Secret.
    Notification failures must have logs.
    Critical alerts cannot rely solely on group robots.

---

## 21. Deploying AlertManager in Kubernetes

### 21.1 kube-prometheus-stack

If using kube-prometheus-stack, AlertManager is typically already included.

Check:

    kubectl get pods -n monitoring | grep alertmanager
    kubectl get svc -n monitoring | grep alertmanager

Access:

    kubectl port-forward svc/<alertmanager-service-name> 9093:9093 -n monitoring

Open:

    http://127.0.0.1:9093

### 21.2 Viewing AlertManager Configuration

If using Prometheus Operator, configuration may come from:

- Helm values;
- Alertmanager CR;
- Secret;
- AlertmanagerConfig CRD.

Check resources:

    kubectl get alertmanager -n monitoring
    kubectl get secret -n monitoring | grep alertmanager
    kubectl get alertmanagerconfig -A

### 21.3 Configuring via Helm values

kube-prometheus-stack typically configures AlertManager via values.yaml.

Example structure:

    alertmanager:
      enabled: true
      config:
        global:
          resolve_timeout: 5m
        route:
          receiver: default
          group_by:
            - alertname
            - cluster
            - namespace
          group_wait: 30s
          group_interval: 5m
          repeat_interval: 4h
        receivers:
          - name: default
            webhook_configs:
              - url: http://alert-webhook.monitoring.svc:8080/alert
                send_resolved: true

Apply:

    helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --version <CHART_VERSION> \
      -f values.yaml

### 21.4 Managing Configuration via Secret

In some deployments, AlertManager configuration is stored in Secret.

Check:

    kubectl get secret -n monitoring | grep alertmanager

When updating configuration, note:

    Do not manually edit production Secret and forget to sync to Git.
    Recommended to manage via Helm values or GitOps.

---

## 22. AlertmanagerConfig CRD

### 22.1 What is AlertmanagerConfig

# Prometheus Operator supports AlertmanagerConfig, allowing different Namespaces or teams to manage their own AlertManager routing snippets.

## Suitable for:
- Multi-team environments
- Multi-tenant setups
- Namespace-level self-management of notifications
- Platform-wide main routing
- Team-specific custom sub-routing

### 22.2 Example

    apiVersion: monitoring.coreos.com/v1alpha1
    kind: AlertmanagerConfig
    metadata:
      name: ai-team-alert-config
      namespace: ai-prod
      labels:
        alertmanagerConfig: enabled
    spec:
      route:
        receiver: ai-team-webhook
        matchers:
          - name: team
            value: ai-platform
            matchType: "="
      receivers:
        - name: ai-team-webhook
          webhookConfigs:
            - url:
                name: ai-alert-webhook-secret
                key: url
              sendResolved: true

**Note:**
    The actual fields may vary with Prometheus Operator versions.
    Always verify with the current cluster CRD schema before use.

### 22.3 Usage Recommendations

AlertmanagerConfig is suitable for production multi-team scenarios.

However, note:
- Platform teams must control the overall routing
- Each team can only manage their own receivers
- Secret permissions must be controlled
- Teams are not allowed to configure global suppression
- Prevent misconfigurations that may cause alert loss

---

## Twenty-ThreeI don't know.Alert Routing Design Cases

### 23.1 Node Alert

**Labels:**
    team="sre"
    severity="critical|warning"
    component="node"

**Routing:**
    critical -> SRE OnCall
    warning -> SRE Operations Group

### 23.2 Kubernetes Pod Alert

**Labels:**
    team="sre"
    namespace
    workload
    severity

**Routing:**
    prod namespace -> SRE + Business Team
    dev/test -> Standard Notification

### 23.3 GPU Alert

**Labels:**
    team="ai-platform"
    component="gpu"
    node
    gpu
    severity

**Routing:**
    GPUXIDError -> AI Platform + SRE
    GPUHighTemperature critical -> AI Platform + SRE OnCall
    GPULowUtilization info -> GPU Resource Governance Daily Report

### 23.4 Database Alert

**Labels:**
    team="database"
    component="mysql|postgresql|redis"

**Routing:**
    Database primary instance unavailable -> DBA OnCall + SRE
    Slow query increase -> DBA
    High connection count -> DBA + Business Owner

### 23.5 Business Service Alert

**Labels:**
    team="app-team-a"
    service="order-api"
    severity="critical"

**Routing:**
    critical -> Business Team OnCall + SRE
    warning -> Business Team Group

---

## Twenty-FourI don't know.Alert Notification Content Examples

### 24.1 Unqualified Example

    CPU high

**Issues:**
- Don't know which cluster
- Don't know which node
- Don't know current value
- Don't know threshold
- Don't know duration
- Don't know how to handle

### 24.2 Qualified Example

    [FIRING][warning] NodeHighCPUUsage

    Cluster: prod-k8s
    Node: k8s-worker01
    Current value: 94.3%
    Threshold: > 90%
    Duration: 5m
    Impact: Pods on this node may experience scheduling or performance issues
    Dashboard: https://grafana.example.com/d/node-detail?var-node=k8s-worker01
    Runbook: https://wiki.example.com/runbook/node-high-cpu
    Recommended Actions:
      1. Check CPU Top Pods on the node
      2. Determine if it's business traffic peak
      3. Expand or migrate Pods if needed
      4. Check system processes if the issue persists

### 24.3 GPU Alert Example

    [FIRING][critical] GPUXIDError

    Cluster: prod-k8s
    Node: k8s-gpu-node01
    GPU: 0
    XID: 79
    Current business Pod: ai-prod/inference-xxx
    Impact: GPU may have driver, hardware, PCIe, or application anomalies
    Dashboard: https://grafana.example.com/d/gpu-detail?var-node=k8s-gpu-node01
    Runbook: https://wiki.example.com/runbook/gpu-xid-error
    Recommended Actions:
      1. Login to the node and check dmesg | grep -i xid
      2. Check nvidia-smi and nvidia-smi -q
      3. Check Pod logs at the time of occurrence
      4. Determine if the node needs to be cordoned
      5. Contact hardware vendor if it recurs

---

## Twenty-FiveI don't know.Alert Governance Principles

### 25.1 Alerts Must Be Actionable

Each alert must answer:
    Who handles it?
    When to handle it?
    How to assess impact?
    How to recover?
    Is there a Runbook?

If an alert has no handler, it's noise.

### 25.2 Prioritize Symptom-Based Alerts

Recommended based on symptoms:
    Service unavailable
    Error rate increase
    Latency increase
    Pod long pending
    GPU XID
    Node NotReady
    Disk nearly full

Not recommended based solely on causes:
    Internal metric fluctuation
    Queue transient increase
    CPU transient spike
    Log warning appears once

### 25.3 Control Alert Volume

Too many alerts cause alert fatigue.

Recommend regular statistics: /think

Daily Alert Count  
Weekly Alert Count  
Top 10 Noisy Alerts  
Unassigned Alerts  
Duplicate Alerts  
Long-running firing Alerts  
Long-running silence Alerts  

### 25.4 Alerts Must Have Owner  

Each alert should be assigned to:  

    SRE  
    Platform Team  
    Business Team  
    DBA  
    Network Team  
    Security Team  
    AI Platform Team  

Alerts without owner should not enter the production notification chain.  

### 25.5 Alerts Must Have Runbook  

Critical alerts must have Runbook.  

Warning alerts are recommended to have Runbook.  

Info alerts can have governance notes.  

---

## Twenty-six, Common Alert Scenario Design  

### 26.1 NodeDown  

Trigger condition:  

    kube_node_status_condition{condition="Ready", status="true"} == 0  

Recommended level:  

    critical  

for:  

    2m - 5m  

Handling:  

    Check node status, kubelet, containerd, network, hardware.  

### 26.2 PodPendingTooLong  

Trigger condition:  

    kube_pod_status_phase{phase="Pending"} == 1  

Recommended level:  

    warning or critical  

for:  

    5m - 10m  

Handling:  

    kubectl describe pod  
    Check Events  
    Check resource shortage, PVC, image, scheduling constraints.  

### 26.3 PodCrashLoopBackOff  

Can be determined via kube-state-metrics or restart count.  

Recommended level:  

    warning  

Production core services:  

    critical  

Handling:  

    kubectl logs --previous  
    kubectl describe pod  
    Check application startup failure reasons.  

### 26.4 DiskSpaceLow  

Trigger condition:  

    Disk usage > 85%  

Recommended level:  

    warning  

Over 95%:  

    critical  

Handling:  

    Check logs, image, containers, data directories.  

### 26.5 GPUHighTemperature  

Trigger condition:  

    DCGM_FI_DEV_GPU_TEMP > 80  

Recommended level:  

    warning  

Over 90:  

    critical  

Handling:  

    Check nvidia-smi, room temperature, fans, Pod load.  

### 26.6 GPUXIDError  

Trigger condition:  

    DCGM_FI_DEV_XID_ERRORS > 0  

Recommended level:  

    critical  

Handling:  

    Check dmesg, journalctl, temperature, power, PCIe, business Pod.  

### 26.7 GPULowUtilization  

Trigger condition:  

    avg_over_time(DCGM_FI_DEV_GPU_UTIL[1h]) < 5  

Recommended level:  

    info  

Handling:  

    Resource governance, should not wake up on-call staff at night.  

---

## Twenty-seven, Alert Storm Handling  

### 27.1 What is Alert Storm  

A large number of alerts flooding in at the same time.  

Common causes:  

- Kubernetes API Server anomaly;  
- Prometheus Target大量 Down;  
- A NodeDown triggering大量 Pod alerts;  
- Network failure;  
- DNS failure;  
- Storage failure;  
- Large number of business dependencies on the same component;  
- Alert Rule design不合理;  
- group_by too detailed;  
- repeat_interval too short;  
- No inhibition.  

### 27.2 Emergency Handling  

Steps:  

    1. Determine if it's a platform-level failure  
    2. Check if alerts are concentrated in the same cluster / node / namespace  
    3. Temporarily create precise silence  
    4. Keep key root cause alerts  
    5. Do not globally silence all critical alerts  
    6. Review alert rules after recovery  

### 27.3 Long-term Optimization  

Optimization directions:  

- Add inhibition;  
- Adjust group_by;  
- Extend repeat_interval;  
- Reduce meaningless alerts;  
- Alert by symptoms;  
- Assign owner to each alert;  
- Add Runbook;  
- Downgrade notification for non-production environments;  
- Do daily report instead of real-time notification for info alerts.  

---

## Twenty-eight, AlertManager High Availability  

### 28.1 Why Need HA  

If there is only one AlertManager:  

- Pod anomaly will cause notification interruption;  
- Node failure will prevent alerts from being sent;  
- There will be notification downtime during upgrades;  
- Risk of losing alert state and silence increases.  

### 28.2 HA Basic Approach  

AlertManager supports cluster mode.  

Multiple AlertManager instances communicate with each other to achieve:  

- Notification deduplication;  
- Silence synchronization;  
- State synchronization;  
- High availability sending.  

Typical pattern:  

    Prometheus A ----\  
                      ---> AlertManager Cluster  
    Prometheus B ----/  

    AlertManager 1  
    AlertManager 2  
    AlertManager 3  

### 28.3 Replica Count in Kubernetes  

In kube-prometheus-stack, AlertManager replica count can usually be configured.  

Example approach:  

    alertmanager:  
      alertmanagerSpec:  
        replicas: 3  

Actual field is determined by chart values.  

### 28.4 HA Notes  

- At least 3 replicas for stability;  
- AlertManager instances need network connectivity;  
- Persistence and configuration synchronization must be correct;  
- Prometheus should configure multiple AlertManager endpoints;  
- Notification channels must tolerate retries;  
- Templates and Secrets must be consistent.  

---

## Twenty-nine, AlertManager Security Design  

### 29.1 Access Control  

AlertManager Web UI can create silence.  

In production, access must be restricted.  

Recommendations:  

- Only allow internal network access;  
- Use Ingress + HTTPS;  
- Use authentication;  
- Use RBAC or reverse proxy authorization;  
- Do not expose to public internet;  
- Record operation audit logs.  

### 29.2 Secret Management  

Sensitive information includes: §§code_0§§

- SMTP password;
- Webhook URL;
- Enterprise WeChat robot URL;
- DingTalk robot token;
- Feishu robot token;
- PagerDuty key;
- Self-developed platform token.

Do not write to public Git.

Recommended usage:

    Kubernetes Secret
    SealedSecret
    External Secrets
    Vault
    Cloud vendor Secret Manager

### 29.3 Webhook Security

Webhook service should support:

- Authentication;
- IP whitelist;
- Signature verification;
- Rate limiting;
- HTTPS;
- Log auditing.

Do not allow anyone to send forged alerts to the alert gateway.

---

## Thirty, AlertManager Common Troubleshooting

### 30.1 Prometheus Alerts Not Entering AlertManager

Troubleshoot:

    1. Is there a firing alert on the Prometheus Alert page?
    2. Is the Alert Rule loaded?
    3. Is the alerting endpoint configured on Prometheus?
    4. Is the AlertManager Service accessible?
    5. Are there sending failures in Prometheus logs?
    6. Is NetworkPolicy blocking?

Commands:

    kubectl logs <prometheus-pod> -n monitoring
    kubectl get svc -n monitoring | grep alertmanager
    kubectl get endpoints -n monitoring | grep alertmanager

### 30.2 AlertManager Receives Alerts But No Notifications

Troubleshoot:

    1. Does the route match?
    2. Is the receiver configured?
    3. Is it silenced?
    4. Is it inhibited?
    5. Has group_wait not yet elapsed?
    6. Has repeat_interval not yet elapsed?
    7. Is the receiver sending failed?
    8. Are there errors in AlertManager logs?

Commands:

    kubectl logs <alertmanager-pod> -n monitoring

Check UI:

    Alerts
    Silences
    Status

### 30.3 Duplicate Notifications

Possible causes:

- Multiple AlertManager instances not properly forming a cluster;
- Inconsistent external labels across Prometheus replicas;
- Too fine group_by;
- Improper use of continue;
- Same alert configured in both Prometheus and Grafana;
- Webhook gateway forwarding duplicates.

### 30.4 No Recovery Notification for Alerts

Check:

    send_resolved is true
    Does the receiver support resolved?
    Does the webhook handle resolved?
    Is the alert actually resolved?
    Did AlertManager restart cause state change?

Example:

    webhook_configs:
      - url: http://alert-webhook.monitoring.svc:8080/alert
        send_resolved: true

### 30.5 Silence Not Taking Effect

Check:

- Is the matcher correct?
- Does the label exist?
- Is the regular expression written incorrectly?
- Is the time range valid?
- Is the timezone understood correctly?
- Do the alert labels match the silence?

---

## Thirty-one, AlertManager Operations Commands

### 31.1 View Pod

    kubectl get pods -n monitoring | grep alertmanager

### 31.2 View Service

    kubectl get svc -n monitoring | grep alertmanager

### 31.3 View Logs

    kubectl logs <alertmanager-pod> -n monitoring

### 31.4 Access UI

    kubectl port-forward svc/<alertmanager-service-name> 9093:9093 -n monitoring

Access:

    http://127.0.0.1:9093

### 31.5 View Secret

    kubectl get secret -n monitoring | grep alertmanager

### 31.6 View PrometheusRule

    kubectl get prometheusrule -n monitoring
    kubectl describe prometheusrule <rule-name> -n monitoring

### 31.7 View AlertmanagerConfig

    kubectl get alertmanagerconfig -A
    kubectl describe alertmanagerconfig <name> -n <namespace>

---

## Thirty-two, Production Alert Governance Checklist

### 32.1 Alert Rule Check

    [ ] Each alert has a clear owner
    [ ] Each alert has severity
    [ ] Each alert has summary
    [ ] Each critical alert has runbook_url
    [ ] Each important alert has dashboard_url
    [ ] Alerts have reasonable for duration
    [ ] Alerts are based on symptoms rather than meaningless fluctuations
    [ ] No long-term unhandled alerts
    [ ] No long-term firing but unattended alerts
    [ ] No duplicate alerts

### 32.2 AlertManager Configuration Check

    [ ] Route configuration is clear
    [ ] group_by is reasonable
    [ ] group_wait is reasonable
    [ ] group_interval is reasonable
    [ ] repeat_interval is reasonable
    [ ] Receiver is available
    [ ] Inhibition rules are reasonable
    [ ] Silence has expiration time
    [ ] Notification channels are reachable
    [ ] Critical alerts can reach on-call personnel
    [ ] Warning alerts will not disturb night shift personnel

### 32.3 Notification Chain Check

[ ] Email available
    [ ] Webhook available
    [ ] Enterprise WeChat / DingTalk / Feishu available
    [ ] OnCall available
    [ ] Send failure has logs
    [ ] Robot token uses Secret
    [ ] Notification template contains sufficient context
    [ ] Recovery notification configuration correct

### 32.4 Security Checks

    [ ] AlertManager UI not exposed to public internet
    [ ] Use HTTPS
    [ ] Access requires authentication
    [ ] Webhook has authorization
    [ ] Secret not submitted in plain text to Git
    [ ] Silence operations have audit
    [ ] Production alert changes go through approval or Review

---

## Thirty-three, Alert Review Template

After each important alert or alert storm, record:

    Alert name:
    Alert level:
    Trigger time:
    Recovery time:
    Affected business:
    Affected scope:
    Trigger conditions:
    Actual root cause:
    Accurate:
    Timely:
    Repeated:
    False positive:
    Missed:
    Need adjust threshold:
    Need adjust for time:
    Need add inhibition:
    Need adjust route:
    Need adjust receiver:
    Need update Runbook:
    Need add Dashboard:
    On-duty personnel:
    Handling process:
    Final conclusion:
    Subsequent actions:

---

## Thirty-four, Common Misconceptions

### 34.1 Misconception One: More Alarms Mean More Safety

Error.

Too many alarms will lead to:

- Alarm fatigue;
- On-duty personnel numbness;
- Important alarms being drowned out;
- Slower fault response;
- Team no longer trusting monitoring system.

### 34.2 Misconception Two: All Alarms Sent to One Group

Error.

Different alarms should be sent to different owners.

Database alarms sent to DBA.

GPU alarms sent to AI platform or GPU operations.

Business error rate sent to business team.

Platform failure sent to SRE.

### 34.3 Misconception Three: All Alarms Are Critical

Error.

Critical should represent issues that must be addressed immediately.

If all are critical, there is no critical.

### 34.4 Misconception Four: Silence Can Long-Term Cover Issues

Error.

Silence is only suitable for maintenance windows and short-term known issues.

Long-term issues should be fixed, decommissioned, or adjust rules.

### 34.5 Misconception Five: Receiving an Alarm Means a Fault

Not necessarily.

Alarms are just abnormal signals.

Need to combine:

- Dashboard;
- Logs;
- kubectl describe;
- Events;
- Node commands;
- Business impact;

To determine if it's a real fault.

### 34.6 Misconception Six: No Alarms Means System is Normal

Error.

It could be:

- Metrics not collected;
- Rule not loaded;
- AlertManager misconfiguration;
- Notification channel failure;
- Alarm threshold unreasonable;
- Business metric missing;
- Alarm silenced or inhibited.

Therefore, must monitor monitoring system itself.

---

## Thirty-five, Experiment Verification Process

### 35.1 Verify Prometheus Rule

After creating PrometheusRule:

    kubectl get prometheusrule -n monitoring
    kubectl describe prometheusrule <rule-name> -n monitoring

Prometheus UI:

    Status
      ↓
    Rules

Confirm rules are loaded.

### 35.2 Verify Alert Status

Prometheus UI:

    Alerts

Check:

    inactive
    pending
    firing

### 35.3 Verify AlertManager Reception

AlertManager UI:

    Alerts

Check if alarms received.

Access:

    kubectl port-forward svc/<alertmanager-service-name> 9093:9093 -n monitoring

    http://127.0.0.1:9093

### 35.4 Verify Notification Channels

Temporarily create a test alarm.

Example:

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: test-alert-rule
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: test.rules
          rules:
            - alert: TestAlert
              expr: vector(1)
              for: 1m
              labels:
                severity: warning
                team: sre
              annotations:
                summary: "Test Alert"
                description: "This is a test alert."

Apply:

    kubectl apply -f test-alert-rule.yaml

Wait for alarm to enter firing.

Confirm notification received.

After test, delete:

    kubectl delete prometheusrule test-alert-rule -n monitoring

### 35.5 Verify Silence

In AlertManager UI create silence:

    alertname="TestAlert"

Confirm notification stops.

After deleting silence, confirm alarm recovery notification logic is normal.

---

## Thirty-six, Production Deployment Recommendations

### 36.1 Alert Configuration in Git

Recommended directory: /think

monitoring/
  prometheus-rules/
    node-rules.yaml
    pod-rules.yaml
    gpu-rules.yaml
    service-rules.yaml
  alertmanager/
    alertmanager.yaml
    templates/
      default.tmpl
  runbooks/
    node-high-cpu.md
    pod-pending.md
    gpu-xid-error.md

### 36.2 Alert Change Review Process

Change content:

- Adding new alerts;
- Deleting alerts;
- Adjusting thresholds;
- Adjusting for duration;
- Adjusting routing;
- Adjusting inhibition;
- Adjusting receiver;
- Adjusting templates.

All changes must go through Review.

### 36,3 Regular Alert Inspection

Weekly or monthly checks:

- Total number of alerts;
- Top noisy alerts;
- Long-firing alerts;
- Long-silenced alerts;
- Alerts without owner;
- Alerts without runbook;
- Duplicate notifications;
- Nighttime false positives;
- Critical accuracy rate;
- Average response time.

### 36.4 Alert Drills

Regular drills:

- NodeDown;
- PodPending;
- PodCrashLoop;
- GPUXIDError;
- GPUHighTemperature;
- DCGMExporterDown;
- Prometheus Target Down;
- Webhook notification failure;
- AlertManager restart;
- Silence creation and recovery.

---

## Thirty-Seven, Summary

AlertManager is the core processing component of the Prometheus alerting system.

Prometheus is responsible for:

    Collecting metrics
    Calculating rules
    Generating alerts

AlertManager is responsible for:

    Grouping
    Deduplication
    Inhibition
    Silence
    Routing
    Notification
    Templates
    High availability

The production alerting pipeline should be:

    PrometheusRule
      ↓
    Prometheus Alert Evaluation
      ↓
    AlertManager
      ↓
    Grouping / Deduplication / Inhibition / Silence
      ↓
    Routing
      ↓
    Receiver
      ↓
    Email / Webhook / IM / OnCall
      ↓
    Dashboard / Runbook
      ↓
    Handling / Recovery / Post-mortem

A good alerting system is not "the loudest", but:

    Accurate alerts
    Timely notifications
    Clear ownership
    Complete content
    Operable
    Silentable
    Inhibitable
    Reviewable
    Governable

Alert design principles:

    Prioritize alerts for business impact.
    Do not disturb people for unhandled alerts.
    Be cautious with alerts for self-healing issues.
    Must have runbooks for manual handling.
    Critical alerts must be few and accurate.
    Warning alerts must be trackable.
    Info alerts are better suited for reports and governance.
    All alerts must have an owner.
    All alerts must be regularly reviewed.

In Kubernetes and GPU operations scenarios, AlertManager needs to integrate with Prometheus, Grafana, Loki/EFK, Runbook, and OnCall processes to form a production-grade fault response loop.

---

## Reference Documents

- Prometheus Alertmanager:
  https://prometheus.io/docs/alerting/latest/alertmanager/

- Prometheus Alerting Overview:
  https://prometheus.io/docs/alerting/latest/overview/

- Alertmanager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- Prometheus Alerting Practices:
  https://prometheus.io/docs/practices/alerting/

- Prometheus Operator Alerting:
  https://prometheus-operator.dev/docs/developer/alerting/

- kube-prometheus-stack Helm Chart:
  https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

- Alertmanager GitHub:
  https://github.com/prometheus/alertmanager

- Grafana Alerting:
  https://grafana.com/docs/grafana/latest/alerting/

- NVIDIA DCGM Exporter:
  https://github.com/NVIDIA/dcgm-exporter