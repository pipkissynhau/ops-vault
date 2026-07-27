# 13-AlertManager-Alarm Policies and Notification Implementation

## Document Description

This document systematically explains Prometheus AlertManager's alarm processing mechanisms, core concepts, configuration structures, routing strategies, grouping, deduplication, suppression, silencing, notification channels, template design, Kubernetes implementation methods, GPU-specific alarm scenarios, production environment alarm management, and common troubleshooting techniques.

This document does not merely focus on "how to send alerts via email or DingTalk"; instead, it provides an operational/SRE perspective on:

- The relationship between Prometheus Alert Rules and AlertManager;
- The entire process from alert triggering to notification delivery;
- Why grouping, deduplication, suppression, and silencing are necessary;
- How to design warning/critical/info alarm severity levels;
- How to route different alerts to various teams;
- How to prevent alert storms;
- How to configure email, webhook, WeCom, DingTalk, and other notification channels;
- How to create alert templates;
- How to include context information such as Dashboards, Runbooks, Namespaces, Pods, Nodes, and GPUs in alerts;
- How to manage AlertManager using the kube-prometheus-stack in Kubernetes;
- How to integrate Node, Pod, Service, GPU, and log-related alerts into production processes;
- How to address the issue of "many alerts without anyone handling them."

This document is recommended for those who have completed the following courses:

- 11-Prometheus-Architecture and Core Metrics Analysis
- 12-Grafana-Dashboard Construction and Custom Monitoring
- 08-GPU-Monitoring and Alarm Integration
- 09-GPU-Fault Troubleshooting Cases and Practices
- 10-GPU-Operational Experiment Environment Setup and Verification

---

## Tags

#Prometheus #AlertManager #Alarms #AlarmManagement #AlertRule #PrometheusRule #Kubernetes #Grafana #Webhook #WeCom #DingTalk #GPUMonitoring #SRE #Observability

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/05-Fundamentals of Observability/13-AlertManager-Alarm Policies and Notification Implementation.md

---

## I. Why Do We Need AlertManager?

Prometheus can use Alerting Rules to determine whether metrics are abnormal.

For example:

- Node CPU usage exceeds 90%
- A Pod remains in the Pending state for over 10 minutes
- GPU temperature surpasses 80°C
- A GPU encounters an XID error
- The service has a 5xx error rate exceeding 5%
- P99 latency exceeds 2 seconds

However, Prometheus is not suitable for handling complex notification logic directly.

If all alerts were sent out directly by Prometheus, numerous issues would arise:

- Repeated notifications for the same alert
- Numerous chain reactions triggered by a single failure
- Alert storms caused by simultaneous anomalies across multiple nodes
- Inability to distribute alerts among different teams
- Lack of temporary silencing options during maintenance windows
- False positives due to upstream failures
- Difficulty distinguishing between warning and critical alerts
- lack of contextual information in alert messages
- Inconsistent management of notification channels
- Inability to perform deduplication, grouping, suppression, and routing uniformly

AlertManager is designed to address these challenges. It acts as a intermediary between Prometheus and notification channels.

The complete process is as follows:

    Exporter / Application /metrics
      ↓
    Prometheus collects metrics
      ↓
    Prometheus Alert Rules determine anomalies
      ↓
    Alerts are generated
      ↓
    Alerts are sent to AlertManager
      ↓
    AlertManager groups, deduplicates, suppresses, silences, and routes alerts
      ↓
    Alerts are sent via Email, Webhook, WeCom, DingTalk, Slack, OnCall, etc.
      ↓
    On-duty personnel process the alerts
      ↓
    Runbooks, Dashboards, logs, and kubectl are used for troubleshooting
      ↓
    Failures are resolved and lessons are learned

Therefore, AlertManager is not just a "message delivery tool"; it serves as a central hub for managing production-level alarms.

---

## II. The Relationship Between Prometheus Alert Rules and AlertManager

### 2.1 What Do Prometheus Alert Rules Do?

Prometheus Alert Rules are responsible for determining whether:

    A specific PromQL condition is met.

For example:

- Whether the CPU usage exceeds a threshold
- Whether a Pod remains in the Pending state
- Whether the GPU temperature is too high
- Whether the service error rate has increased
- Whether a target is down

Example:

    - alert: NodeHighCPUUsage
      expr: node:cpu_usage:percent > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node CPU usage is too high"
        description: "Node {{ $labels.instance }} has a CPU### 4.1 Inactive

The condition is not met.

For example:

    The CPU usage is below the threshold.

### 4.2 Pending

The Prometheus rule expression has been met, but it has not yet reached the duration specified by `for`.

For example:

    CPU > 90%
    for: 5 minutes

From the first minute to before the fifth minute, the alarm remains pending.

### 4.3 Firing

Once the condition is continuously met and exceeds the `for` time, the alarm enters the firing state.

At this point, Prometheus will send the alarm to the AlertManager.

### 4.4 Resolved

The condition is no longer met, and the alarm returns to its normal state.

The AlertManager can send a resolution notification or not, depending on the configuration and the capabilities of the receiver.

### 4.5 Why `for` is Important

Do not write:

    expr: cpu_usage > 90

This kind of instantaneous trigger can easily generate noise.

Instead, write:

    expr: cpu_usage > 90
    for: 5 minutes

This way, short-term fluctuations can be filtered out.

The `for` time should vary depending on the type of alarm:

    NodeDown:
        2 minutes - 5 minutes

    High CPU usage:
        5 minutes - 10 minutes

    Insufficient disk space:
        10 minutes - 30 minutes

    GPU XID issue:
        1 minute

    High GPU temperature:
        2 minutes - 5 minutes

    Pod Pending:
        5 minutes - 10 minutes

---

## V. AlertManager Configuration File Structure

AlertManager configurations are usually in YAML format.

Core structure:

    global:
      resolve_timeout: 5 minutes

    route:
      group_by: [...]
      group_wait: 30 seconds
      group_interval: 5 minutes
      repeat_interval: 4 hours
      receiver: default
      routes:
        - matchers:
            - severity="critical"
          receiver: oncall

    receivers:
      - name: default
        emailConfigs: []

      - name: oncall
        webhook_configs: []

    inhibit_rules:
      - source_matchers: []
        target_matchers: []
        equal: []

    templates:
      - /etc/alertmanager/templates/*.tmpl

Meaning of each part:

    global:
        Global settings, such as SMTP and timeout values.

    route:
        The alarm routing tree, which determines where the alarms will be sent.

    receivers:
        Define notification methods such as email, webhook, Slack, or WeCom.

    inhibit_rules:
        Rules to suppress certain alarms.

    templates:
        Notification template files.

---

## VI. Detailed Explanation of Route Routing Rules

### 6.1 Root Route

Example:

    route:
      receiver: default
      group_by: ['alertname', 'cluster']
      group_wait: 30 seconds
      group_interval: 5 minutes
      repeat_interval: 4 hours

Meaning:

    receiver:
        The default receiver.

    group_by:
        Labels to use for grouping alarms.

    group_wait:
        Wait time before sending the first notification for a new alarm group.

    group_interval:
    Time interval between notifications when new alarms join the same group.

    repeat_interval:
    How often to remind again if the same alarm continues to fire.

### 6.2 group_by

Example:

    group_by: ['alertname', 'cluster', 'namespace']

This means that alarms with the same `alertname`, `cluster`, and `namespace` will be combined into a single notification group.

Common grouping strategies:

    Infrastructure alarms:

        ['alertname', 'cluster', 'node']

    Kubernetes namespace alarms:

        ['alertname', 'cluster', 'namespace']

    Business service alarms:

        ['alertname', 'cluster', 'namespace', 'service']

    GPU alarms:

        ['alertname', 'cluster', 'node', 'gpu']

Do not use too many fields for `group_by`.

Adding `pod`, `container`, `instance`, and `endpoint` can lead to poor grouping results.

### 6.3 group_wait

This indicates how long to wait before sending the first notification after a new alarm group appears.

Example:

    group_wait: 30 seconds

Meaning:

    Wait for related alarms to arrive together before combining them and sending the notification.

If set too short:

    Each alarm might be sent individually.

If set too long:

    Critical alarms may be delayed in notification.

Suggestions:

    critical:
        10 seconds - 30 seconds

    warning:
        30 seconds - 2 minutes

### 6.4 group_interval

This indicates how long to wait before sending another notification if new alarms join the same alarm group after it has already been notified once.

Example:

    group_interval: 5 minutes

This prevents frequentDo not write passwords in plain text within Git. Use Secret management for them. Email is suitable for warnings, daily reports, and low-priority alerts. It is not recommended to rely solely on email for critical alerts.

### 9.2 Example of webhook receiver

    receivers:
      - name: sre-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/alert
            send_resolved: true

Advantages of webhooks:

- Can be integrated with WeCom;
- Can be integrated with DingTalk;
- Can be integrated with Lark;
- Can be integrated with self-developed platforms;
- Allow for secondary formatting;
- Enable alert storage in databases;
- Support automated responses.

### 9.3 WeCom / DingTalk / Lark

AlertManager may not natively support all domestic instant messaging services.

Common approaches:

    AlertManager
      ↓
    webhook
      ↓
    alertmanager-webhook-adapter
      ↓
    WeCom / DingTalk / Lark bots

Or:

    AlertManager
      ↓
    Self-developed alert gateway
      ↓
    Multiple notification channels

Production recommendations:

    Do not let AlertManager handle complex business logic directly.
    It is recommended to use a webhook gateway for unified adaptation of domestic IM notifications.

### 9.4 Example of multiple receivers

    receivers:
      - name: default
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/default

      - name: sre-critical
        webhookConfigs:
          - url: http://alert-webhook.monitoring.svc:8080/sre-critical
            send_resolved: true

      - name: ai-team
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/ai-team
            send_resolved: true

      - name: email-warning
        emailConfigs:
          - to: sre-warning@example.com
            from: prometheus@example.com
            smarthost: smtp.example.com:587
            auth_username: prometheus@example.com
            auth_password: <password>
            send_resolved: true

---

## Section Ten: Inhibition Rules

### 10.1 Why Inhibition is Needed

When an upstream failure occurs, it can trigger a chain reaction of alerts downstream.

For example, if a node goes down:

    NodeDown
    KubeletDown
    NodeExporterDown
    PodDown
    PodNotReady
    TargetDown
    DCGMExporterDown

The root cause is actually NodeDown. The other alerts are merely consequences. In such cases, inhibition rules can be used to suppress these downstream alerts.

### 10.2 Example of Inhibition Rules: Suppressing warnings when a critical alert occurs

    inhibit_rules:
      - source_matchers:
          - severity="critical"
        target_matchers:
          - severity="warning"
        equal:
          - alertname
          - cluster
          - namespace

This means that when there is a critical alert with the same alertname, cluster, and namespace, all corresponding warning alerts will be suppressed.

### 10.3 Example of Inhibition Rules: Suppressing other alerts on the same node when NodeDown occurs

    inhibit_rules:
      - source_matchers:
          - alertname="NodeDown"
        target_matchers:
          - severity=~"warning|critical"
        equal:
          - cluster
          - node

This rule suppresses any warning or critical alerts that occur on the same node as soon as a NodeDown alert is triggered.

### 10.4 Example of Inhibition Rules: Suppressing component alerts when the Kubernetes API Server fails

    inhibit_rules:
      - source_matchers:
          - alertname="KubeAPIDown"
        target_matchers:
          - alertname=~"Kube.*"
        equal:
          - cluster

When the Kubernetes API Server is down, this rule suppresses any derived alerts from other Kubernetes components.

### 10.5 Precautions for Using Inhibition Rules

Do not overuse inhibition rules. Incorrect application can lead to the suppression of truly important alerts. It is recommended to:

    Only apply inhibition rules when there is a clear upstream-downstream relationship.
    Provide comments for each inhibition rule to explain its purpose.
    Verify the rules in a test environment before making any production changes.
    Regularly review and update inhibition rules as needed.

---

## Section Eleven: Silence Mechanisms

### 11.1 Scenarios Where Silence Is Appropriate

Silence mechanisms are suitable for:

- Scheduled maintenance;
- Version upgrades;
- Node restarts;
- Driver updates;
- GPU node maintenance;
- Load testing;
- Drills;
- Temporary suppression of known issues;
- Migration of services offline.

### 11.2 Situations Where Silence Should    alertname
    severity
    cluster
    namespace
    pod
    node
    service
    team
    environment

 추가된 GPU 알림 항목:

    gpu
    gpu_uuid
    gpu_model

 추가된 비즈니스 알림 항목:

    app
    service
    route
    method
    status

### 13.2 severity

값을 통일하여 사용해야 합니다.

권장 사항:

    severity="critical"
    severity="warning"
    severity="info"

혼용하지 마세요:

    high
    error
    warn
    major
    P1

회사 내에서 이미 표준화된 규칙이 없는 경우에만 사용하십시오.

### 13.3 team

루팅에 사용됩니다.

예시:

    team="sre"
    team="ai"
    team="database"
    team="middleware"
    team="network"

### 13.4 environment

환경을 구분하는 데 사용됩니다.

예시:

    environment="prod"
    environment="test"
    environment="dev"

생산 환경과 테스트 환경의 알림은 반드시 별도로 라우팅되어야 합니다.

### 13.5 cluster

다중 클러스터 환경에서는 반드시 cluster 레이블을 사용해야 합니다.

예시:

    cluster="shanghai-prod"
    cluster="shenzhen-prod"
    cluster="dev-k8s"

cluster 레이블이 없으면 다중 클러스터 알림을 찾기가 어려워집니다.

---

## 십사, 알림 주석 설계

주석은 알림 내용을 보여주는 데 사용됩니다.

다음 항목들을 포함하는 것이 좋습니다:

    summary
    description
    dashboard_url
    runbook_url

### 14.1 summary

간단한 설명입니다.

예시:

    summary: "GPU 온도가 너무 높습니다."

### 14.2 description

상세한 설명입니다.

예시:

    description: "노드 {{ $labels.node }}에 위치한 GPU {{ $labels gpu }}의 온도가 {{ $value }}°C를 5분 이상 초과하고 있습니다."

### 14.3 dashboard_url

Grafana로 이동합니다.

예시:

    dashboard_url: "https://grafana.example.com/d/gpu-overview?var-node={{ $labels.node }}"

### 14.4 runbook_url

처리 문서로 이동합니다.

예시:

    runbook_url: "https://wiki.example.com/runbook/gpu-high-temperature"

### 14.5 권장 형식

알림은 다음과 같은 내용을 포함해야 합니다:

    description: "CPU 사용량이 높습니다."

다음과 같은 정보도 추가하는 것이 좋습니다:

    어떤 클러스터인지
    어떤 노드인지
    어떤 Namespace인지
    어떤 Pod인지
    현재 값
    임계값
    지속 시간
    처리 문서
    대시보드 링크

---

## 십오, PrometheusRule 예제

다음은 Kubernetes에서 Prometheus Operator / kube-prometheus-stack을 사용할 때의 PrometheusRule 예제입니다.

### 15.1 노드 CPU 사용량이 높은 알림

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
                summary: "노드 CPU 사용량이 너무 높습니다."
                description: "노expr: DCGM_FI_DEV_GPU_TEMP > 80
for: 5 minutes
labels:
    severity: warning
    team: ai-platform
annotations:
    summary: "GPU temperature is too high"
    description: "The GPU {{ $labels gpu }} on {{ $labels.Hostname }} has a temperature above 80°C for more than 5 minutes."
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
              expr: DCGM_FI_DEV_XID-errors > 0
              for: 1 minute
              labels:
                severity: critical
                team: ai-platform
annotations:
                summary: "A GPU XID error has occurred"
                description: "The GPU {{ $labels.gpu }} on {{ $labels.Hostname }} has reported an XID error value of {{ $value }}. Check dmesg and GPU health immediately."
                dashboard_url: "https://grafana.example.com/d/gpu-overview"
                runbook_url: "https://wiki.example.com/runbook/gpu-xid-error"

---

## Example Basic Configuration for AlertManager

The following is a basic configuration suitable for learning environments.

    global:
      resolve_timeout: 5 minutes

    route:
      receiver: default-webhook
      group_by:
        - alertname
        - cluster
        - namespace
      group_wait: 30 seconds
      group_interval: 5 minutes
      repeat_interval: 4 hours
      routes:
        - receiver: critical-webhook
          matchers:
            - severity="critical"
          group_wait: 10 seconds
          repeat_interval: 1 hour

        - receiver: warning-webhook
          matchers:
            - severity="warning"
          repeat_interval: 4 hours

        - receiver: info-webhook
          matchers:
            - severity="info"
          repeat_interval: 24 hours

    receivers:
      - name: default-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/default
            send_resolved: true

      - name: critical-webhook
        webhookConfigs:
          - url: http://alert-webhook.monitoring.svc:8080/critical
            send_resolved: true

      - name: warning-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/warning
            send_resolved: true

      - name: info-webhook
        webhook configs:
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

## Example of Production-Level Routing Design

### 17.1 Design Goals

The goals are to:

    Send critical alerts to the SRE OnCall team
    Deliver GPU alerts to the AI platform team
    Route database alerts to the DBA
    Prevent dev/test environment alerts from disturbing nighttime OnCall processes
    Ensure that info-level alerts go through low-priority channels
    Assign default alerts to the standard SRE group

### 17.2 Example Configuration

    route:
      receiver: sre-default
      group_by:
        - alertname
        - cluster
        - namespace
      group_wait: 30 seconds
      group_interval: 5 minutes
      repeat_interval: 4 hours

      routes:
        - receiver: sre-oncall-critical
          matchers:
            - severity="critical"
            - environment="prod"
          group_wait: 10 seconds
          repeat_interval: 1 hour
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
          repeat_interval: 24 hours

    receivers:
      - name: sre-default
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:```markdown
- url: http://alert-webhook.monitoring.svc:8080/ai-platform
    send_resolved: true

- name: dba-team
  webhook_configs:
    - url: http://alert-webhook.monitoring.svc:8080/dba
    send_resolved: true

- name: dev-warning
  webhookconfigs:
    - url: http://alert-webhook.monitoring.svc:8080/dev-warning
    send_resolved: true

- name: info-digest
  webhook_configs:
    - url: http://alert-webhook.monitoring.svc:8080/info
    send_resolved: false

### 17.3 Precautions

Use `continue: true` with caution.

For example, critical alerts are first sent to the SRE OnCall team, and if `team=ai-platform`, they are also sent to the AI team.

If you do not want duplicate notifications, do not use `continue`.

---

## Section Eighteen: Notification Template Design

### 18.1 Why Templates Are Needed

Default notification formats are often unsuitable for production use.

Production alerts need to include sufficient context.

Templates can ensure a consistent format.

### 18.2 Recommended Template Content

Alert messages should include:

    Alert Status: FIRING / RESOLVED
    Alert Name
    Severity Level
    Cluster
    Environment
    Namespace
    Pod
    Node
    Service
    Current Value
    Start Time
    Dashboard Link
    Runbook Link
    Description
    Handling Recommendations

### 18.3 Simplified Template Example

    {{ define "alert.title" }}
    [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }} - {{ .CommonLabels.severity }}
    {{ end }}

    {{ define "alert.message" }}
    {{ range .Alerts }}
    Alert Name: {{ .Labels(alertname }}
    Severity Level: {{ .Labels.severity }}
    Cluster: {{ .Labels.cluster }}
    Namespace: {{ .Labels.namespace }}
    Node: {{ .Labels.node }}
    Pod: {{ .Labels.pod }}
    Description: {{ .Annotations.description }}
    Dashboard: {{ .Annotations/dashboard_url }}
    Runbook: {{ . Annotations.runbook_url }}
    Start Time: {{ .StartsAt }}
    {{ end }}
    {{ end }}
```

### 18.4 Template File Configuration

For AlertManager configuration:

    templates:
      - /etc/alertmanager/templates/*.tmpl

In receivers, reference the templates as follows:

    receivers:
      - name: webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/alert
            send_resolved: true

Note:

    The actual message format for webhooks is often converted again by the webhook adapter.
    Native receivers like email and Slack support templates more directly.

---

## Section Nineteen: Webhook Alert Gateway Design

### 19.1 Why a Webhook Gateway Is Needed

In Chinese-speaking production environments, common notification channels include:

- WeCom;
- DingTalk;
- Lark;
- SMS;
    Telephone;
    Custom Work Orders;
    Custom OnCall Systems.

AlertManager is not suitable for directly handling all these channel differences.

It is more advisable to use a structure like this:

    AlertManager
      ↓
    Webhook Gateway
      ↓
    WeCom / DingTalk / Lark / Email / Work Orders / Telephone

### 19.2 Functions of the Webhook Gateway

The webhook gateway can perform the following tasks:

- Receive JSON alerts from AlertManager;
- Format the messages;
- Select the appropriate channel based on severity;
    Determine the target group based on team assignment;
    Add @mentions;
    Store alerts in a database;
    Create work orders;
    Trigger automated processes;
    Implement rate limiting;
    Remove duplicate notifications;
    Retry failed notifications;
    Maintain comprehensive statistics.

### 19.3 AlertManager Webhook Payload

AlertManager sends JSON data to the webhook, which typically includes:

    receiver
    status
    alerts
    groupLabels
    commonLabels
    commonAnnotations
    externalURL

The webhook service must parse these fields correctly.

### 19.4 Production Best Practices

A webhook gateway should have the following features:

    High availability;
    Logging capabilities;
    Failure retry mechanisms;
    Rate limiting;
    Authentication;
    Audit tracking;
    Alert storage functionality;
    Channel failover options;
    Template management tools.

Do not rely on a temporary script to handle production alert notifications.

---

## Section Twenty: Implementation Strategies for WeCom, DingTalk, and Lark

### 20.1 WeCom

Common approaches include:

    AlertManager
      ↓
    Webhook Adapter
      ↓
    WeCom Bot Webhook

Notes:

- WeCom bots have specific message format requirements;
- The```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version <CHART_VERSION> \
  -f values.yaml
```

### 21.4 Using Secrets to Manage Configurations

In certain deployment scenarios, the AlertManager configuration is stored in Secrets.

To view it:

```bash
kubectl get secret -n monitoring | grep alertmanager
```

When updating the configuration, be careful not to manually modify the production Secret and forget to synchronize it to Git. It is recommended to use Helm values or GitOps for management.

---

## Chapter 22: AlertmanagerConfig CRD

### 22.1 What is AlertmanagerConfig

The Prometheus Operator supports AlertmanagerConfig, which allows different namespaces or teams to manage their own AlertManager routing segments.

This is suitable for:

- Multi-team environments;
- Multi-tenant setups;
- Different namespaces managing notifications independently;
- A unified main route at the platform level;
- Teams customizing sub-routes.

### 22.2 Example

```yaml
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
```

Note:

- Actual fields may change with versions of the Prometheus Operator.
- Always refer to the current cluster's CRD schema before using it.

### 22.3 Usage Recommendations

AlertmanagerConfig is suitable for production scenarios involving multiple teams.

However, consider the following points:

- The platform team should control the overall routing;
- Each team should only manage its own receivers;
- Secret permissions must be carefully managed;
- Teams are not allowed to configure global suppression settings;
- Prevent misconfigurations that could lead to missed alerts.

---

## Chapter 23: Case Studies on Alert Routing Design

### 23.1 Node Alerts

Tags:

    team="sre"
    severity="critical|warning"
    component="node"

Routing:

    critical -> SRE OnCall
    warning -> SRE Operations Team

### 23.2 Kubernetes Pod Alerts

Tags:

    team="sre"
    namespace
    workload
    severity

Routing:

    prod namespace -> SRE + Business Teams
    dev/test -> General Notifications

### 23.3 GPU Alerts

Tags:

    team="ai-platform"
    component="gpu"
    node
    gpu
    severity

Routing:

    GPUXIDError -> AI Platform + SRE
    GPUHighTemperature critical -> AI Platform + SRE OnCall
    GPULowUtilization info -> GPU Resource Management Daily Report

### 23.4 Database Alerts

Tags:

    team="database"
    component="mysql|postgresql|redis"

Routing:

    Primary database unavailable -> DBA OnCall + SRE
    High number of slow queries -> DBA
    High connection count -> DBA + Business Owner

### 23.5 Business Service Alerts

Tags:

    team="app-team-a"
    service="order-api"
    severity="critical"

Routing:

    critical -> Business Team OnCall + SRE
    warning -> Business Team Group
```

---

## Chapter 24: Examples of Alert Notification Content

### 24.1 Poor Example

    CPU high

Issues:

- Unknown cluster;
- Unknown node;
- Unknown current value;
- Unknown threshold;
- Unknown duration;
- No clear actions to take.

### 24.2 Good Example

    [FIRING][warning] NodeHighCPUUsage

    Cluster: prod-k8s
    Node: k8s-worker01
    Current value: 94.3%
    Threshold: > 90%
    Duration: 5 minutes
    Impact: Pods on this node may experience scheduling or performance issues.
    Dashboard: https://grafana.example.com/d/node-detail?var-node=k8s-worker01
    Runbook: https://wiki.example.com/runbook/node-high-cpu
    Recommended actions:
      1. Check the top CPU-consuming pods on the node.
      2. Determine if it's during peak business hours.
      3. Expand or relocate pods if necessary.
      4. Monitor system processes if abnormalities persist.

### 24.3 Example of a GPU Alert

    [FIRING][critical] GPUXIDError

    Cluster: prod-k8s
    Node: k8s-gpu-node01
    GPU: 0
    X### 26.4 DiskSpaceLow

Trigger condition:

    Disk usage > 85%

Recommendation level:

    warning

If it exceeds 95%:

    critical

Action:

    Check logs, images, containers, and data directories.[ ] Each critical alert has a dashboard_url.
[ ] Alerts have a reasonable for time setting.
[ ] Alerts are based on actual symptoms, not meaningless fluctuations.
[ ] There are no alerts that remain untreated for an extended period.
[ ] There are no alerts that continuously trigger but go unnoticed.
[ ] There are no duplicate alerts.

### 32.2 AlertManager Configuration Check

[ ] The route configuration is clear.
[ ] Group_by is appropriate.
[ ] Group_wait is appropriate.
[ ] Group_interval is appropriate.
[ ] Repeat_interval is appropriate.
[ ] The receiver is available.
[ ] Inhibition rules are reasonable.
[ ] Silence settings have an expiration time.
[ ] Notification channels are accessible.
[ ] Critical alerts reach the on-duty personnel.
[ ] Warning alerts do not disturb night shift personnel.

### 32.3 Notification Chain Check

[ ] Email notifications are available.
[ ] Webhook notifications are available.
[ } Enterprise WeChat / DingTalk / Lark notifications are available.
[ ] OnCall notifications are available.
[ ] Logs are generated in case of notification failures.
[ ] The robot token uses a Secret for authentication.
[ ] Notification templates contain sufficient context.
[ ] Recovery notification settings are correct.

### 32.4 Security Check

[ ] The AlertManager UI is not exposed to the public internet.
[ ] HTTPS is used.
[ ] Access requires authentication.
[ ] Webhook notifications have authentication.
[ ] Secrets are not submitted in plaintext to Git.
[ ] Silence operations are audited.
[ ] Changes to production alerts require approval or review.

---

## Section 33: Alarm Review Template

After each critical alert or an alarm storm, the following should be recorded:

- Alert Name:
- Alert Level:
- Trigger Time:
- Recovery Time:
- Business Impact:
- Scope of Impact:
- Trigger Conditions:
- Actual Root Cause:
- Accuracy of the Alert:
- Timeliness of Response:
- Whether it was a Duplicate Alert:
- Whether it was a False Positive:
- Whether it was a Missed Alarm:
- Whether Thresholds Need to be Adjusted:
- Whether the for Time Setting Needs to Be Adjusted:
- Whether New Inhibition Rules Are Needed:
- Whether the Route Configuration Needs to Be Adjusted:
- Whether the Receiver Needs to Be Changed:
- Whether the Runbook Needs to Be Updated:
- Whether a New Dashboard Is Required:
- On-Duty Personnel:
- Handling Process:
- Final Conclusion:
- Next Action Items:

---

## Section 34: Common Misconceptions

### 34.1 Misconception 1: More Alerts Mean Greater Security

Wrong.

Too many alerts can lead to:

- Alert fatigue;
- Indifference among on-duty personnel;
- True critical alerts being overlooked;
- Slower failure responses;
- The team losing trust in the monitoring system.

### 34.2 Misconception 2: All Alerts Should Be Sent to the Same Group

Wrong.

Different alerts should be sent to different owners.

Database alerts should go to DBAs.

GPU alerts should go to AI platforms or GPU operations teams.

Business error rates should go to business teams.

Platform failures should go to SRE teams.

### 34.3 Misconception 3: All Alerts Should Be Classified as Critical

Wrong.

“Critical” should indicate issues that require immediate attention.

If all alerts are classified as critical, then the term loses its meaning.

### 34.4 Misconception 4: Silence Can Conceal Problems for a Long Time

Wrong.

Silence is only suitable for maintenance periods and short-term known issues.

Long-term problems should be resolved, taken offline, or their rules adjusted.

### 34.5 Misconception 5: Receiving an Alert Means There Is Definitely a Failure

Not necessarily.

An alert is just an indication of an anomaly.

It needs to be verified in conjunction with:

- Dashboards;
- Logs;
- kubectl describe commands;
- Events;
- Node commands;
- Business impact;

to determine if it truly represents a failure.

### 34.6 Misconception 6: No Alerts Mean the System Is Working Fine

Wrong.

It could mean:

- Metrics are not being collected;
- Rules have not been loaded;
- The AlertManager configuration is incorrect;
- Notification channels are down;
- Alarm thresholds are unreasonable;
- Business metrics are missing;
- Alerts have been silenced or inhibited.

Therefore, it is essential to monitor the monitoring system itself.

---

## Section 35: Experimental Verification Process

### 35.1 Verifying Prometheus Rules

After creating a PrometheusRule:

    kubectl get prometheusrule -n monitoring
    kubectl describe prometheusrule <rule-name> -n monitoring

In the Prometheus UI:

    Status
      ↓
    Rules

Confirm that the rule has been loaded.

### 35.2 Verifying Alert Status

In the Prometheus UI:

    Alerts

Check for:

    inactiveAlert Design Principles:

- Prioritize alarms that can affect business operations.
- Avoid disturbing people with issues that cannot be resolved immediately.
- Be cautious when issuing alerts for issues that can resolve automatically.
- For issues requiring manual intervention, ensure there is a corresponding Runbook in place.
- Critical alarms should be rare but precise.
- Warning alarms should allow for follow-up action.
- Info alarms are better suited for reporting and governance purposes.
- All alarms must have an assigned owner.
- Regular reviews of alarm settings are necessary.

In Kubernetes and GPU operations, the AlertManager needs to be integrated with Prometheus, Grafana, Loki/EFK, Runbooks, and OnCall processes to create a fully functional production-grade fault response system.

---

## Reference Documents

- Prometheus Alertmanager:
  https://prometheus.io/docs/alerting/latest-alertmanager/

- Prometheus Alerting Overview:
  https://prometheus.io/docs/alerting/latest/overview/

- Alertmanager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- Prometheus Alerting Practices:
  https://prometheus.io/docs/practices ALERTING/

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