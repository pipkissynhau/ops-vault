# 17-Log Alerting and Automated ResponseActual

## Document Overview

This document systematically organizes the design of log alerting, log alerting rules, AlertManager notifications, Webhook alert gateway, automated response, ChatOps, Runbook, fault diagnosis Job, semi-automated recovery, and production alert governance methods in a Kubernetes environment.

This document is not simply about "alerting when there's an ERROR in logs", but rather about establishing a production-ready log alerting and automated response system from the perspective of operations/SRE.

This document focuses on answering the following questions:

- Why can't we directly alert for all ERROR logs;
- What are the differences between log alerting and metric alerting;
- What roles do Loki, ELK/EFK, Prometheus, and AlertManager play in the alerting pipeline;
- How to design log alerting rules;
- How to use Loki Ruler for log alerting;
- How to use Elasticsearch/OpenSearch for log alerting;
- How to send log alerts to AlertManager;
- How to use AlertManager for grouping, suppression, silencing, and routing;
- How to design a Webhook alert gateway;
- How to trigger diagnostic scripts automatically with alerts;
- Which actions are suitable for automation and which require manual confirmation;
- How to design Kubernetes automated diagnosis Jobs;
- How to respond to alerts for Pod anomalies, CUDA OOM, Service 5xx errors, database connection failures, image pull failures, etc.;
- How to avoid alert storms and erroneous automation;
- How to form a closed-loop of "alert → diagnosis → handling → post-mortem".

This document is suitable for study after completing the following content:

- 11-Prometheus-Architecture and Core Metrics Analysis
- 12-Grafana-Dashboard Construction and Custom Monitoring
- 13-AlertManager-Alerting Strategies and Notification Implementation
- 14-K8S-MonitoringActual-Node-Pod-Service Metric Collection and Troubleshooting
- 15-Loki-Log Collection and QueryActual
- 16-ELK-EFK-Log Collection and SearchActual

---

## Tags

#Kubernetes #Loki #ELK #EFK #AlertManager #Webhook #It'sALogCall. #AutomatedResponse #Runbook #ChatOps #SRE #FaultCheck. #Observation #GpuTransport

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-observability Foundation/17-Log Alerting and Automated ResponseActual.md

---

## One, Why Log Alerting and Automated Response Are Needed

Prometheus metric alerts can tell you:

    CPU is high
    Memory is high
    Pod has restarted
    Pod is pending
    Service error rate has increased
    GPU temperature is high
    GPU memory usage is high

But metrics typically cannot directly tell you:

    Why the application failed to start
    Which configuration item is missing
    Which database connection failed
    Which interface timed out
    Where CUDA OOM occurred in a task
    Why model loading failed
    What is the Python Traceback content
    What is the specific Java Exception stack
    Which upstream caused Nginx to return 502
    What is the specific error for image pull failure

This information typically comes from logs.

The value of log alerting lies in:

    Detecting critical error patterns before metric anomalies are discovered;
    Quickly supplementing root cause context after metric alerts;
    Providing supplementary monitoring for issues that cannot be directly metricized;
    Real-time discovery of abnormal keywords, error codes, stack traces, and business failure reasons;
    Reducing manual repetitive troubleshooting through automated diagnostics.

The value of automated response lies in:

    Automatically collecting context after alerts are triggered;
    Automatically querying Pod, Service, Endpoints, and Events;
    Automatically fetching recent error logs;
    Automatically generating diagnostic reports;
    Automatically pushing to alert groups;
    Automatically fixing low-risk actions;
    Providing manual confirmation entry for high-risk actions.

The complete goal is not "automatically restarting everything", but rather:

    First, automatically diagnose
    Then, assist in judgment
    Finally, cautiously recover

---

## Two, Differences Between Log Alerting and Metric Alerting

### 2.1 What Metric Alerts Are Suitable For

Metric alerts are suitable for stable, quantifiable, and aggregatable states.

Examples:

    Node CPU > 90%
    Pod Pending > 10m
    Service 5xx error rate > 5%
    P95 latency > 1s
    GPU temperature > 80°C
    GPU memory usage > 90%
    Prometheus Target Down
    Deployment available replicas insufficient

Characteristics:

    Clear numerical values
    Clear trends
    Easy to aggregate
    Good alert stability
    Suitable for core SLO / SLI

### 2.2 What Log Alerts Are Suitable For

Log alerts are suitable for detecting text anomalies, key error patterns, and internal application anomalies.

Examples:

    CUDA out of memory
    database connection failed
    connection refused
    timeout
    panic
    Traceback
    NullPointerException
    ImagePullBackOff
    permission denied
    model load failed
    no space left on device
    too many connections

Characteristics:

    Rich context
    Can detect internal application errors
    Prone to noise
    Dependent on log quality
    Requires good keyword and structured field design

### 2.3 How They Should Be Combined

Correct approach:

    Metric alerts are responsible for detecting business impact and resource anomalies.
    Log alerts are responsible for supplementing key errors and root cause clues.
    Automated response is responsible for collecting context and assisting in handling.

Example:

    Prometheus alert:
        ServiceHigh5xxErrorRate

    Loki / ELK query:
        Find ERROR, timeout, connection refused, stack trace during the corresponding time period

    Automated response:
        Automatically collect Pod status, Service Endpoints, recent logs, Events, and generate a diagnostic report

---

## Three, What Log Alerts Should Not Do

### 3.1 Do Not Alert for All ERRORs Directly

Incorrect approach:

    Alert whenever an ERROR appears

Issues:

- ERRORs may not affect business;
- Some ERRORs are user input errors;
- Some ERRORs are retryable exceptions;
- Some ERRORs are low priority;
- A single release may generate a large number of ERRORs;
- Alert noise can drown out real failures.

More reasonable:

Recently 5 minutes ERROR count exceeds threshold  
and occurs in production Namespace  
and belongs to core services  
and is not a known ignorable error  

### 3.2 Do not use overly broad keywords  

Errors:  

    error  
    fail  
    exception  

Issues:  

    Too broad matching.  
    Too many false positives.  
    Different semantics across applications.  

Better:  

    "CUDA out of memory"  
    "database connection failed"  
    "connection refused"  
    "too many connections"  
    "panic:"  
    "Traceback"  
    "NullPointerException"  
    "no space left on device"  

### 3.3 Do not let log alerts replace business metrics  

Log alerts cannot replace:  

    QPS  
    error rate  
    P95 / P99 latency  
    availability probes  
    success rate  
    queue length  

Use metric alerts for business quality.  

Log alerts are for supplementing abnormal context.  

### 3.4 Do not prioritize high-priority alerts for non-production environments  

Log alerts for dev / test environments can be retained, but should not enter nighttime OnCall.  

Recommendations:  

    prod:  
        critical / warning  

    staging:  
        warning / info  

    dev:  
        info or daily report  

### 3.5 Do not let automation execute high-risk actions directly  

High-risk actions include:  

    Delete Pod  
    Restart Deployment  
    Drain node  
    Delete PVC  
    Delete index  
    Restart database  
    Clean disk  
    Modify firewall  
    Scale core services  
    Switch master-slave  
    Modify production configuration  

These actions must be manually confirmed and cannot be executed solely based on log alerts.  

---  

## Four. Overall Log Alert Architecture  

### 4.1 Loki Log Alerting Chain  

    Kubernetes Pod Logs  
      ↓  
    Grafana Alloy / Promtail  
      ↓  
    Loki  
      ↓  
    Loki Ruler  
      ↓  
    AlertManager  
      ↓  
    Webhook / Email / IM / OnCall  
      ↓  
    Automated diagnostic service  
      ↓  
    Diagnostic report / ChatOps / Runbook  

### 4.2 ELK / EFK Log Alerting Chain  

    Kubernetes Pod Logs  
      ↓  
    Filebeat / Fluent Bit / Logstash  
      ↓  
    Elasticsearch / OpenSearch  
      ↓  
    Kibana Alerting / OpenSearch Alerting  
      ↓  
    Webhook / AlertManager / Alert Gateway  
      ↓  
    Automated diagnostic service  
      ↓  
    Diagnostic report / Runbook  

### 4.3 Recommended Unified Chain  

In production, recommend:  

    Log platform generates alerts  
      ↓  
    Unified entry to AlertManager or Alert Gateway  
      ↓  
    Unified grouping, deduplication, suppression, routing  
      ↓  
    Unified trigger automated diagnostics  
      ↓  
    Unified notification  

Do not have each system send its own notifications.  

Otherwise, it will result in:  

- Duplicate alerts;  
- Inconsistent notification formats;  
- Difficulty in silence;  
- Difficulty in suppression;  
- Alert storm during failures;  
- Inability to unify post-mortem analysis.  

---  

## Five. Log Alert Severity Design  

### 5.1 Critical  

Suitable for:  

- Large 5xx logs in production core services;  
- Production service panic / fatal;  
- Database primary connection failure and persistent;  
- Large authentication failures possibly involving security risks;  
- GPU XID severe errors;  
- CUDA OOM causing production inference services unavailable;  
- Log platform unwriteable;  
- Large log loss;  
- Core service critical error codes persistently appear.  

Notifications:  

    OnCall  
    Enterprise WeChat / DingTalk high-priority groups  
    Phone / SMS  
    Incident group  

### 5.2 Warning  

Suitable for:  

- ERROR log count exceeds baseline;  
- Abnormal stack in a Pod;  
- Increased database connection failures;  
- Increased timeouts;  
- CUDA OOM occasional;  
- Abnormal log growth in a Namespace;  
- Increased errors in non-core services.  

Notifications:  

    Operations group  
    Business team group  
    Work order  
    Handle during working hours  

### 5.3 Info  

Suitable for:  

- Too many debug logs;  
- Low utilization GPU reminder;  
- Non-production error logs;  
- Top log volume reminder;  
- Approaching retention period;  
- Resource governance reminders.  

Notifications:  

    Daily report  
    Weekly report  
    Low-priority group  

---  

## Six. Log Alert Rule Design Principles  

### 6.1 Rules must have clear targets  

Alerts should clearly specify:  

    Which cluster  
    Which environment  
    Which Namespace  
    Which service  
    Which Pod  
    Which container  
    Which error  
    How long ago  
    How many times  

Do not just write:  

    Log anomaly  

### 6.2 Rules must have time window  

Log alerts must use time windows.  

Example:  

    Last 5 minutes  
    Last 10 minutes  
    Last 15 minutes  

Do not alert on single log entries blindly.  

### 6.3 Rules must have thresholds  

Examples:  

    ERROR > 20 in 5 minutes  
    timeout > 10 in 10 minutes  
    CUDA OOM > 0 in 5 minutes  
    panic > 0 in 5 minutes  
    Authentication failure > 100 in 5 minutes  

Thresholds vary by error type.  

### 6.4 Rules must exclude meaningless logs  

Common filters:  

    /healthz  
    /ready  
    /metrics  
    /favicon.ico  
    liveness probe  
    readiness probe  
    known benign error  
    debug  
    test namespace  

### 6.5 Rules must have owner  

Each alert must have:  

    team  
    service  
    owner  
    runbook_url  

Alerts without owner should not enter production real-time notification chain.  

---  

## Seven. Loki Log Alerting Basics  

### 7.1 Loki Ruler Function  

Loki Ruler can create:  

    Alerting Rules  
    Recording Rules /think

Log alerts are typically achieved through LogQL by counting the number of logs within a specific time window.

Examples:

    Number of ERROR logs in the last 5 minutes
    Number of CUDA OOM occurrences in the last 5 minutes
    Number of timeout occurrences in the last 10 minutes

Then sent to AlertManager.

### 7.2 Basic Structure of Loki Log Alerts

Example:

    groups:
      - name: app-log-alerts
        rules:
          - alert: AppErrorLogsTooMany
            expr: |
              sum by (namespace, pod) (
                count_over_time({namespace="prod"} |= "ERROR" [5m])
              ) > 20
            for: 5m
            labels:
              severity: warning
              team: app
            annotations:
              summary: "Excessive application ERROR logs"
              description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has exceeded the threshold for ERROR logs in the last 5 minutes."

### 7.3 count_over_time

Counts the number of log entries within a time range.

Example:

    count_over_time({namespace="prod"} |= "ERROR" [5m])

Meaning:

    Number of logs containing "ERROR" in the last 5 minutes.

### 7.4 rate

Counts the log rate.

Example:

    rate({namespace="prod"} |= "ERROR" [5m])

Meaning:

    Rate of ERROR logs per second in the last 5 minutes.

### 7.5 sum by

Aggregates by dimensions.

Example:

    sum by (namespace, pod) (
      count_over_time({namespace="prod"} |= "ERROR" [5m])
    )

Meaning:

    Counts the number of error logs by namespace and pod.

---

## VIII. Loki Log Alert Examples

### 8.1 Excessive Production ERROR Logs

    groups:
      - name: app-error-log-alerts
        rules:
          - alert: AppErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace="prod"} |~ "(?i)error|exception|panic" [5m])
              ) > 30
            for: 5m
            labels:
              severity: warning
              team: app
              source: loki
            annotations:
              summary: "Excessive application error logs"
              description: "Namespace {{ $labels.namespace }} app {{ $labels.app }} has exceeded 30 error logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-error-logs"

### 8.2 Excessive Service timeout Logs

    groups:
      - name: app-timeout-log-alerts
        rules:
          - alert: AppTimeoutLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace="prod"} |~ "(?i)timeout|timed out|deadline exceeded" [5m])
              ) > 10
            for: 5m
            labels:
              severity: warning
              team: app
              source: loki
            annotations:
              summary: "Excessive application timeout logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has exceeded the threshold for timeout-related logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-timeout"

### 8.3 Database Connection Failures

    groups:
      - name: db-error-log-alerts
        rules:
          - alert: DatabaseConnectionErrorLogs
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace="prod"} |~ "(?i)database connection failed|too many connections|connection refused" [5m])
              ) > 5
            for: 5m
            labels:
              severity: warning
              team: database
              source: loki
            annotations:
              summary: "Excessive database connection anomaly logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has experienced multiple database connection anomalies in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/database-connection-error"

### 8.4 Java Exception

    groups:
      - name: java-exception-log-alerts
        rules:
          - alert: JavaExceptionLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace="prod"} |~ "Exception|Caused by|NullPointerException" [5m])
              ) > 20
            for: 5m
            labels:
              severity: warning
              team: app
              source: loki
            annotations:
              summary: "Java Exception logs are excessive"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has excessive Java exception logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/java-exception"

### 8.5 Python Traceback

    groups:
      - name: python-traceback-log-alerts
        rules:
          - alert: PythonTracebackDetected
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace="prod"} |= "Traceback" [5m])
              ) > 0
            for: 1m
            labels:
              severity: warning
              team: app
              source: loki
            annotations:
              summary: "Python Traceback detected"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has detected Python Traceback in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/python-traceback"

### 8.6 CUDA OOM

    groups:
      - name: gpu-log-alerts
        rules:
          - alert: CUDALogOOMDetected
            expr: |
              sum by (namespace, pod) (
                count_over_time({namespace=~"ai-.*"} |= "CUDA out of memory" [5m])
              ) > 0
            for: 1m
            labels:
              severity: warning
              team: ai-platform
              source: loki
            annotations:
              summary: "CUDA OOM log detected"
              description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has CUDA out of memory in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/cuda-oom"

### 8.7 Model Load Failure

    groups:
      - name: ai-model-log-alerts
        rules:
          - alert: ModelLoadFailedLogs
            expr: |
              sum by (namespace, pod) (
                count_over_time({namespace=~"ai-.*"} |~ "(?i)model load failed|failed to load model|checkpoint not found" [5m])
              ) > 0
            for: 1m
            labels:
              severity: warning
              team: ai-platform
              source: loki
            annotations:
              summary: "Model load failure"
              description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has model load failure logs."
              runbook_url: "https://wiki.example.com/runbook/model-load-failed"

---

## NineI don't know.ELK / OpenSearch Log Alerting Logic

### 9.1 Kibana Alerts

Kibana can create rules based on Elasticsearch queries.

Common logic:

    Query logs-k8s-app-* indices
    Filter prod namespace
    Filter log.level:error
    Count logs in the last 5 minutes
    Trigger alert if count > 20

### 9.2 OpenSearch Alerting

OpenSearch Dashboards can create Monitors.

Basic structure:

    Monitor:
        Scheduled index query

    Trigger:
        Threshold judgment

    Action:
        Send notification

### 9.3 Elasticsearch Query DSL Alerting Example

Query the number of error logs in the prod namespace in the last 5 minutes: /think

GET logs-k8s-app-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "kubernetes.namespace": "prod" } },
        { "term": { "log.level": "error" } },
        {
          "range": {
            "@timestamp": {
              "gte": "now-5m",
              "lte": "now"
            }
          }
        }
      ]
    }
  }
}

### 9.4 KQL Alert Query Examples

    kubernetes.namespace : "prod" and log.level : "error"

or:

    kubernetes.namespace : "prod" and message : "CUDA out of memory"

or:

    kubernetes.namespace : "prod" and message : "connection refused"

### 9.5 Production Recommendations

ELK / OpenSearch log alerts are also recommended to be unified into:

    AlertManager
    or unified alert gateway

Do not let Kibana, OpenSearch, Prometheus, Grafana send alerts in different formats to different destinations.

---

## Ten, AlertManager Routing Design

### 10.1 Log Alert Label Recommendations

Log alerts are recommended to use unified labels:

    severity
    team
    source
    cluster
    environment
    namespace
    app
    service

source can be:

    loki
    elasticsearch
    opensearch

Example:

    labels:
      severity: warning
      team: app
      source: loki
      environment: prod

### 10.2 AlertManager Routing Example

    route:
      receiver: default-webhook
      group_by:
        - alertname
        - cluster
        - namespace
        - app
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        - receiver: sre-critical
          matchers:
            - severity="critical"
            - environment="prod"
          group_wait: 10s
          repeat_interval: 1h
          continue: true

        - receiver: ai-platform
          matchers:
            - team="ai-platform"

        - receiver: app-team
          matchers:
            - team="app"

        - receiver: log-info-digest
          matchers:
            - severity="info"
          repeat_interval: 24h

    receivers:
      - name: default-webhook
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/default
            send_resolved: true

      - name: sre-critical
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/sre-critical
            send_resolved: true

      - name: ai-platform
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/ai-platform
            send_resolved: true

      - name: app-team
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/app-team
            send_resolved: true

      - name: log-info-digest
        webhook_configs:
          - url: http://alert-webhook.monitoring.svc:8080/info
            send_resolved: false

### 10.3 Log Alert Suppression

If the service already has metric alerts:

    ServiceHigh5xxErrorRate

You can suppress low-level log alerts for the same service:

    AppErrorLogsTooMany

Example logic:

    ServiceHigh5xxErrorRate critical
        suppress AppErrorLogsTooMany warning in the same namespace/app

Configuration example:

    inhibit_rules:
      - source_matchers:
          - alertname="ServiceHigh5xxErrorRate"
          - severity="critical"
        target_matchers:
          - alertname="AppErrorLogsTooMany"
          - severity="warning"
        equal:
          - cluster
          - namespace
          - app

Note:

Suppress with caution.
Do not completely block valuable root cause log alerts.
A more common practice is to treat log alerts as contextual information within the same group, rather than as independent alerts.

---

## Eleven. Webhook Alert Gateway Design

### 11.1 Why an Alert Gateway is Needed

AlertManager can directly send Webhooks, but in production environments, a middle gateway is typically required.

Reasons:

- Adapt to enterprise WeChat, DingTalk, and Feishu;
- Standardize message format;
- Unified authentication;
- Unified rate limiting;
- Unified auditing;
- Unified alert storage;
- Unified automated response;
- Unified diagnostic report generation;
- Unified association with Dashboard and Runbook.

Recommended architecture:

    AlertManager
      ↓
    alert-webhook-gateway
      ↓
    Notification channels
      ↓
    Automated diagnostic service
      ↓
    ChatOps / Work order / Report

### 11.2 Responsibilities of the Alert Gateway

The alert gateway can perform:

    Receive AlertManager JSON
    Validate signature or Token
    Parse alerts
    Route based on labels
    Format messages
    Query additional context
    Trigger automated diagnostics
    Create work orders
    Push notifications
    Record audit logs
    Rate limiting and deduplication
    Failure retry

### 11.3 Key Information in Webhook Payload

AlertManager typically sends:

    receiver
    status
    alerts
    groupLabels
    commonLabels
    commonAnnotations
    externalURL

Each alert typically contains:

    labels
    annotations
    startsAt
    endsAt
    generatorURL
    fingerprint

Automated responses primarily rely on:

    alertname
    severity
    cluster
    namespace
    pod
    node
    app
    team
    source

### 11.4 Security Requirements

The Webhook gateway must have:

    Authentication
    HTTPS
    IP whitelist
    Rate limiting
    Audit logs
    Request signing
    Timeout control
    Retry control
    Idempotency handling

Do not expose an unauthenticated Webhook interface.

---

## Twelve. Automated Response Tiering

### 12.1 L0: Only Notification

Actions:

    Send alert messages only

Suitable for:

- info;
- low-priority warnings;
- non-production environments;
- governance alerts.

### 12.2 L1: Automated Diagnostics

Actions:

    Automatically collect context
    Generate diagnostic reports automatically
    Push notifications to groups
    Do not modify any production resources

Suitable for:

- Most production alerts;
- Pod anomalies;
- Service errors;
- GPU OOM;
- Database connection failures;
- Increased log errors.

This is the most recommended automated stage.

### 12.3 L2: Semi-Automated Repair

Actions:

    Provide repair suggestions
    Offer buttons or commands
    Require manual confirmation for execution

Suitable for:

- Restart non-core Pods;
- Scale non-core Deployments;
- Clean up temporary files;
- Trigger rollbacks;
- Cordon nodes;
- Restart tasks.

### 12.4 L3: Automated Repair

Actions:

    Automatically execute recovery actions under strict conditions

Suitable for very limited low-risk scenarios:

- Restart stateless non-core services;
- Rebuild failed test environment Pods;
- Automatically clean up expired test Jobs;
- Automatically scale temporary workers;
- Automatically disable non-production Debug logs.

### 12.5 L4: Prohibit Automated Execution

The following actions are prohibited from being executed without confirmation:

    Delete PVC
    Delete database data
    Restart database master
    Switch database master-slave
    Drain production nodes
    Delete production Pod batch replicas
    Modify production configuration
    Execute high-risk cleanup
    Modify security policies
    Delete Elasticsearch indices
    Delete object storage data

---

## Thirteen. What Should an Automated Diagnostic Report Include

### 13.1 Basic Information

    Alert name
    Alert level
    Alert source
    Cluster
    Environment
    Namespace
    Pod
    Container
    Node
    App
    Service
    Trigger time
    Current status

### 13.2 Kubernetes Status

    Pod Phase
    Pod Ready
    Pod Restart Count
    Last State
    Exit Code
    Reason
    Events
    Deployment replica count
    Service selector
    Endpoints
    Node Ready
    Node resources

### 13.3 Log Context

    Last 50 lines of error logs
    Number of critical errors in the last 5 minutes
    Whether OOMKilled occurred
    Whether timeout occurred
    Whether connection refused occurred
    Whether Traceback / Exception / panic occurred
    Whether CUDA OOM occurred

### 13.4 Metric Context

    CPU usage
    Memory usage
    Pod restart count
    Service error rate
    P95 / P99 latency
    GPU utilization
    GPU memory
    GPU temperature

### 13.5 Recommended Actions

    Possible causes
    Investigation commands
    Repair suggestions
    Runbook link
    Dashboard link
    Whether manual intervention is recommended
    Whether automated repair is allowed

---

## Fourteen. Automated Diagnostic Service Design

### 14.1 Service Responsibilities

The automated diagnostic service receives alerts and executes corresponding diagnostic workflows based on alertname.

Example:

    PodRestartTooOften:
        Collect Pod status, previous logs, Events, and resource usage.

    CUDALogOOMDetected:
        Collect Pod logs, GPU memory, nvidia-smi results, and Pod resource declarations.

    ServiceHigh5xxErrorRate:
        Collect Service, Endpoints, error logs, QPS, and latency.

    DatabaseConnectionErrorLogs:
        Collect application logs, Service DNS, and database Service reachability.

### 14.2 Recommended Architecture /think

AlertManager  
      ↓  
Webhook Gateway  
      ↓  
Diagnosis Controller / Worker  
      ↓  
Kubernetes API  
Prometheus API  
Loki API  
Elasticsearch API  
      ↓  
Diagnosis Report  
      ↓  
IM / Work Order / Dashboard  

### 14.3 Permission Control  

Diagnosis service needs access to:  

    Kubernetes API  
    Prometheus API  
    Loki API  
    Elasticsearch / OpenSearch API  

But permissions must be minimized.  

Kubernetes permission recommendations:  

    get/list/watch pods  
    get/list events  
    get/list services  
    get/list endpoints  
    get/list endpointslices  
    get/list deployments  
    get/list nodes  
    get pod logs  

Do not grant by default:  

    delete pods  
    update deployments  
    delete pvc  
    create clusterrole  
    patch nodes  

Auto-fix actions require separate authorization and should be split into separate service accounts.  

---  

## Fifteen, Kubernetes Automatic Diagnosis RBAC Example  

### 15.1 ServiceAccount  

    apiVersion: v1  
    kind: ServiceAccount  
    metadata:  
      name: alert-diagnosis  
      namespace: monitoring  

### 15.2 ClusterRole  

    apiVersion: rbac.authorization.k8s.io/v1  
    kind: ClusterRole  
    metadata:  
      name: alert-diagnosis-readonly  
    rules:  
      - apiGroups: [""]
        resources:  
          - pods  
          - pods/log  
          - services  
          - endpoints  
          - events  
          - nodes  
          - namespaces  
          - configmaps  
        verbs:  
          - get  
          - list  
          - watch  

      - apiGroups: ["discovery.k8s.io"]  
        resources:  
          - endpointslices  
        verbs:  
          - get  
          - list  
          - watch  

      - apiGroups: ["apps"]  
        resources:  
          - deployments  
          - statefulsets  
          - daemonsets  
          - replicasets  
        verbs:  
          - get  
          - list  
          - watch  

      - apiGroups: ["batch"]  
        resources:  
          - jobs  
          - cronjobs  
        verbs:  
          - get  
          - list  
          - watch  

### 15.3 ClusterRoleBinding  

    apiVersion: rbac.authorization.k8s.io/v1  
    kind: ClusterRoleBinding  
    metadata:  
      name: alert-diagnosis-readonly  
    subjects:  
      - kind: ServiceAccount  
        name: alert-diagnosis  
        namespace: monitoring  
    roleRef:  
      kind: ClusterRole  
      name: alert-diagnosis-readonly  
      apiGroup: rbac.authorization.k8s.io  

### 15.4 Security Notes  

This permission is read-only.  

Suitable for automatic diagnosis.  

If auto-fix is needed, create a separate ServiceAccount and strictly limit:  

    Only allow specific namespace  
    Only allow specific resources  
    Only allow specific actions  
    Must be audited  
    Must be rate-limited  
    Best requires manual confirmation  

---  

## Sixteen, Automatic Diagnosis Command Templates  

### 16.1 Pod Abnormality Diagnosis  

    kubectl get pod <pod> -n <namespace> -o wide  

    kubectl describe pod <pod> -n <namespace>  

    kubectl logs <pod> -n <namespace> --tail=100  

    kubectl logs <pod> -n <namespace> --previous --tail=100  

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -50  

### 16.2 Service Abnormality Diagnosis  

    kubectl get svc <service> -n <namespace> -o wide  

    kubectl describe svc <service> -n <namespace>  

    kubectl get endpoints <service> -n <namespace>  

    kubectl get endpointslice -n <namespace> | grep <service>  

    kubectl get pod -n <namespace> --show-labels  

### 16.3 Node Abnormality Diagnosis  

    kubectl get node <node> -o wide

kubectl describe node <node>

kubectl get pod -A -o wide | grep <node>

### 16.4 GPU Abnormality Diagnosis

    kubectl get pod -A -o wide | grep <gpu-node>

    kubectl describe node <gpu-node>

    kubectl logs <pod> -n <namespace> --previous --tail=100

Node Side:

    nvidia-smi

    nvidia-smi -q

    dmesg | grep -i xid

    journalctl -k | grep -i nvidia

### 16.5 Prometheus Queries

Pod CPU:

    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{namespace="<namespace>", pod="<pod>", container!="", image!=""}[5m])
    )

Pod Memory:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="<namespace>", pod="<pod>", container!="", image!=""}
    )

Pod Restart:

    increase(kube_pod_container_status_restarts_total{namespace="<namespace>", pod="<pod>"}[10m])

### 16.6 Loki Queries

Error Logs:

    {namespace="<namespace>", pod="<pod>"} |~ "(?i)error|exception|panic|traceback"

CUDA OOM:

    {namespace="<namespace>", pod="<pod>"} |= "CUDA out of memory"

timeout:

    {namespace="<namespace>", pod="<pod>"} |~ "(?i)timeout|timed out|deadline exceeded"

---

## SeventeenI don't know.Auto Diagnosis Job Design

### 17.1 Why Use Job

After an alert is triggered, a Kubernetes Job can be created to collect diagnostic information.

Advantages:

    Consistent with cluster environment
    Can use kubectl image
    Automatically exits after execution
    Can record logs
    Can create different Jobs based on alerts
    Permissions can be controlled by ServiceAccount

### 17.2 Diagnosis Job Example

The example is only for reference, and in production, parameters like namespace, pod, node, etc., should be dynamically generated by the alert gateway.

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: diagnose-pod-restart-sample
      namespace: monitoring
    spec:
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          serviceAccountName: alert-diagnosis
          restartPolicy: Never
          containers:
            - name: diagnose
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  set -eu

                  NAMESPACE="prod"
                  POD="api-xxx"

                  echo "===== Pod Basic Info ====="
                  kubectl get pod "${POD}" -n "${NAMESPACE}" -o wide || true

                  echo "===== Pod Describe ====="
                  kubectl describe pod "${POD}" -n "${NAMESPACE}" || true

                  echo "===== Pod Current Logs ====="
                  kubectl logs "${POD}" -n "${NAMESPACE}" --tail=100 || true

                  echo "===== Pod Previous Logs ====="
                  kubectl logs "${POD}" -n "${NAMESPACE}" --previous --tail=100 || true

                  echo "===== Recent Events ====="
                  kubectl get events -n "${NAMESPACE}" --sort-by=.lastTimestamp | tail -50 || true

### 17.3 Notes

Do not use latest.

In production, the image version should be fixed:

    bitnami/kubectl:<version>

Or build your own diagnosis image:

    registry.example.com/sre/diagnosis-toolkit:v1.0.0

The diagnosis image can include:

    kubectl
    curl
    jq
    yq
    dig
    nslookup
    iproute2
    bash
    python
    promtool
    loki query client

---

## EighteenI don't know.ChatOps Response Design

### 18.1 What is ChatOps

ChatOps integrates alerts, diagnosis, approval, and execution into chat tools.

For example: /think

# Alert Push to Enterprise WeChat Group  
## Display Diagnosis Report in Group  
## Provide Operation Buttons or Commands  
## Execute Repair After Manual Confirmation  
## Write Operation Results Back to Group  

### 18.2 Common Interactions  

Alert Message:  

    [FIRING] PodRestartTooOften  
    namespace: prod  
    pod: api-xxx  
    restart: 5 times / 10m  

Buttons:  

    View Logs  
    View Events  
    Execute Diagnosis  
    Create Ticket  
    Silent for 30 Minutes  
    Restart Pod  
    Open Runbook  

### 18.3 Operation Grading  

Can be Executed Directly:  

    View Logs  
    Execute Diagnosis  
    Create Ticket  
    Open Dashboard  
    Open Runbook  

Requires Confirmation:  

    Restart Pod  
    Scale Deployment  
    Rollback Version  
    Cordon Node  

Prohibited in Chat Tools:  

    Delete PVC  
    Delete Database  
    Delete Index  
    Batch Restart Production Services  
    Drain Production Node  
    Modify Production Security Policy  

### 18.4 Audit Requirements  

All ChatOps Operations Must Be Recorded:  

    Operator  
    Operation Time  
    Alert ID  
    Target Resource  
    Executed Action  
    Execution Result  
    Approver  
    Original Request  
    Response Result  

---

## Nineteen, Automation Response Scenario One: Pod Restarting Too Often  

### 19.1 Alert Source  

Prometheus:  

    PodRestartTooOften  

Or Loki:  

    AppErrorLogsTooMany  
    PythonTracebackDetected  
    JavaExceptionLogsTooMany  

### 19.2 Automatic Diagnosis Actions  

Collect:  

    kubectl get pod  
    kubectl describe pod  
    kubectl logs --tail=100  
    kubectl logs --previous --tail=100  
    Recent Events  
    Pod CPU / Memory  
    Restart Count  
    Last State Reason  
    Exit Code  

### 19.3 Judgment Logic  

If Last State Reason is:  

    OOMKilled:  
        Determine Memory Insufficiency  

If Logs Contain:  

    connection refused:  
        Determine Unreachable Dependencies  

If Logs Contain:  

    permission denied:  
        Determine Permission or Mount Issues  

If Logs Contain:  

    config not found:  
        Determine ConfigMap / Secret / Configuration Issues  

If Events Contain:  

    Liveness probe failed:  
        Determine Probe or Application Health Interface Issues  

### 19.4 Automatic Suggestions  

Report Output:  

    Possible Causes  
    Next Command  
    Suggest Rollback  
    Suggest Adjust Resources  
    Suggest Check Dependencies  
    Runbook Link  

### 19.5 Automatic Repair  

Default: No Automatic Restart  

Reasons:  

    Pod is Already Restarting  
    Rebooting Again is Unnecessary  
    Need to Identify Root Cause  

Can Provide Semi-Automatic Buttons:  

    Redeploy Deployment  
    Rollback to Previous Version  
    Temporary Scaling Up  
    Silent for 30 Minutes  

But Must Be Manually Confirmed  

---

## Twenty, Automation Response Scenario Two: Service 5xx Increase  

### 20.1 Alert Source  

Prometheus:  

    ServiceHigh5xxErrorRate  

Logs:  

    AppErrorLogsTooMany  
    AppTimeoutLogsTooMany  
    DatabaseConnectionErrorLogs  

### 20.2 Automatic Diagnosis Actions  

Collect:  

    Service Details  
    Endpoints  
    EndpointSlice  
    Backend Pod List  
    Recent Error Logs  
    QPS  
    5xx Error Rate  
    P95 / P99  
    Recent Release Version  
    Pod Restart Status  
    Events  

### 20.3 Diagnosis Commands  

    kubectl describe svc <service> -n <namespace>  

    kubectl get endpoints <service> -n <namespace>  

    kubectl get endpointslice -n <namespace> | grep <service>  

    kubectl get pod -n <namespace> -o wide  

    kubectl logs <pod> -n <namespace> --tail=100  

### 20.4 Loki Query  

    {namespace="<namespace>", app="<app>"} |~ "(?i)error|exception|timeout|connection refused"  

### 20.5 Judgment Logic  

If Endpoints Are Empty:  

    Service Selector or Pod Ready Issue  

If 5xx Correlates with New Version Release Time:  

    Prioritize Suspecting Release Changes  

If Logs Show Many Timeout:  

    Prioritize Checking Downstream Dependencies  

If Logs Show Many Database Connection Failed:  

    Prioritize Checking Database Connection Pool, Database Status, and Network  

If Pod Restarts Increase:  

    Combine with Pod Restart Diagnosis  

### 20.6 Automatic Suggestions  

Recommendations:  

    Check Recent Releases  
    Inspect Dependency Services  
    Check Database Status  
    Inspect Configuration Changes  
    Rollback if Necessary  

Do Not Recommend Automatic Rollback Unless Strict Release System and Approval Mechanism Are in Place  

---

## Twenty-One, Automation Response Scenario Three: CUDA OOM  

### 21.1 Alert Source  

Loki:  

    CUDALogOOMDetected  

Prometheus:  

    GPUMemoryUsageHigh  
    PodRestartTooOften  
    ServiceHigh5xxErrorRate  

### 21.2 Automatic Diagnosis Actions  

Collect:  

    Pod Logs  
    Pod Resource Claims  
    Pod's GPU Node  
    GPU Memory Metrics  
    GPU Utilization  
    GPU Temperature  
    Recent Pod Restarts  
    CUDA OOM Log Context  
    Business Request Volume  
    Batch Size Related Logs  

### 21.3 Prometheus Query

# GPU VRAM Usage:

    DCGM_FI_DEV_FB_USED

# GPU VRAM Free:

    DCGM_FI_DEV_FB_FREE

# GPU Utilization:

    DCGM_FI_DEV_GPU_UTIL

### 21.4 Loki Query

    {namespace="<namespace>", pod="<pod>"} |= "CUDA out of memory"

    {namespace="<namespace>", pod="<pod>"} |~ "(?i)batch|model|worker|memory"

### 21.5 Judgment Logic

Possible Causes:

    Batch size too large
    Model too large
    High concurrency
    Too many workers
    Multiple models loaded repeatedly
    VRAM leak
    GPU sharing causing VRAM contention
    MIG instance VRAM insufficient

### 21.6 Automatic Suggestions

Recommendations:

    Reduce batch size
    Reduce concurrency
    Reduce number of workers
    Check model loading strategy
    Check for duplicate model loading
    Use GPU with larger VRAM
    Use FP16 / BF16 / quantization
    Increase VRAM monitoring alerts

Do not automatically delete Pods.

Can be semi-automated:

    Restart non-production inference Pods
    Reduce non-production replica concurrency
    Create a work order for AI platform team

---

## 22. Automation Response Scenario Four: Database Connection Failure

### 22.1 Alert Source

Loki:

    DatabaseConnectionErrorLogs

Prometheus:

    ServiceHigh5xxErrorRate
    ServiceHighP95Latency

### 22.2 Automatic Diagnostic Actions

Collect:

    Error logs
    Connection failure keywords
    Affected services
    Affected Namespace
    Service Endpoints
    DNS resolution results
    Existence of database Service
    Recent changes to application configuration Secret / ConfigMap
    Business error rate and latency

### 22.3 Common Log Keywords

    connection refused
    too many connections
    database connection failed
    timeout
    access denied
    password authentication failed
    no route to host
    connection reset by peer

### 22.4 Judgment Logic

connection refused:

    Database port unreachable or service not listening.

too many connections:

    Database connection quota exhausted.

access denied:

    Username/password or permission issues.

timeout:

    Network, database slow, connection pool exhausted, or firewall issues.

no route to host:

    Network routing or NetworkPolicy issues.

### 22.5 Automatic Suggestions

Recommendations:

    Check database monitoring
    Check connection pool
    Check Secret changes
    Check NetworkPolicy
    Check DNS
    Check database service reachability

Do not automatically restart database.

---

## 23. Automation Response Scenario Five: Log Volume Abnormal Surge

### 23.1 Alert Source

Loki / ELK:

    Sudden increase in log volume for a namespace
    Sudden surge in ERROR logs for an app
    Abnormal log writing volume for a Pod

### 23.2 Automatic Diagnostic Actions

Collect:

    Top Namespace log volume
    Top Pod log volume
    Top App log volume
    Top error keywords
    Debug logging enabled
    Health check flooding
    Abnormal stack trace printing

### 23.3 Judgment Logic

Common Causes:

    DEBUG logging enabled
    Abnormal loop retry
    Excessive health check logs
    Access log surge
    Dependency issues causing flooding
    New version printing excessive logs
    Abnormal multi-line log splitting

### 23.4 Automatic Suggestions

Recommendations:

    Lower log level
    Filter healthcheck
    Fix abnormal loop
    Increase log sampling
    Adjust collection rules
    Shorten non-production retention period

Can be semi-automated:

    Temporarily lower collection level for non-production namespace
    Create a work order
    Notify owner

Do not automatically discard core logs in production unless there is a clear policy.

---

## 24. Automation Response Scenario Six: Image Pull Failure

### 24.1 Alert Source

Kubernetes Events or logs:

    ImagePullBackOff
    ErrImagePull
    pull access denied
    manifest unknown
    x509 certificate signed by unknown authority
    connection refused
    no basic auth credentials

### 24.2 Automatic Diagnostic Actions

Collect:

    Pod describe
    Events
    Image name
    imagePullSecrets
    node
    containerd status
    registry address
    Secret existence

### 24.3 Judgment Logic

manifest unknown:

    Image tag does not exist.

pull access denied:

    Insufficient permissions.

no basic auth credentials:

    Missing or incorrect imagePullSecret.

x509:

    Certificate issues.

connection refused:

    Repository unreachable.

### 24.4 Automatic Suggestions

Recommendations:

    Check image tag
    Check Harbor repository
    Check imagePullSecret
    Check containerd registry configuration
    Check node-to-repository network
    Check certificate or HTTP insecure configuration

Do not automatically modify containerd configuration.

---

## 25. Automation Response Action Whitelist

### 25.1 Low-Risk Actions

Can be automatically executed:

    Collect logs
    Collect Events
    Query Prometheus
    Query Loki
    Query Elasticsearch
    Generate diagnostic report
    Create work order
    Send notification
    Create read-only diagnostic Job
    Temporarily add info label
    Trigger non-production environment diagnostic task

### 25.2 Medium-Risk Actions

Require manual confirmation:

    Restart a single stateless Pod
    rollout restart non-core Deployment
    rollback non-core service
    scale replicas
    cordon node
    delete failed Job
    temporarily silence alert
    adjust non-production log level

### 25.3 High-Risk Actions

Prohibit automatic execution:

    Delete PVC
    Delete StatefulSet data
    Restart database master
    drain production node
    Delete Elasticsearch index
    Clean production log directory
    Modify production Secret
    Modify NetworkPolicy
    Modify firewall
    Switch master-slave
    Batch delete Pod

---

## Twenty-six, Automation Response Security Baseline

### 26.1 Minimize Permissions

Automation service defaults to read-only.

Write permissions must:

    Be granted separately
    Be approved separately
    Be limited to namespace
    Be limited to resource
    Be limited to action
    Have audit logs
    Have rollback plan

### 26.2 Idempotency

Automation actions must be repeatable and not produce uncontrollable side effects.

Examples:

    Diagnostic Job repeated creation should have unique ID.
    Repeated notifications should deduplicate.
    Repeated queries should not modify state.
    Repair actions should check current state before execution.

### 26.3 Rate Limiting

Must limit:

    Single-minute trigger count
    Single-alert trigger count
    Single-namespace trigger count
    Single-service trigger count
    Global concurrency

Avoid automation system being overwhelmed by alert storm during failure.

### 26.4 Auditing

Record:

    Alert ID
    Alert name
    Trigger time
    Executed action
    Executed parameters
    Executor or system
    Approver
    Execution result
    Output report
    Error information

### 26.5 Rollback

Any automatic repair action must clearly define:

    How to rollback
    What to do on failure
    What to do on timeout
    Who confirms
    Whether affects production

---

## Twenty-seven, Runbook Design

### 27.1 What Runbook Must Include

Every important alert should have a Runbook.

Content includes:

    Alert name
    Alert meaning
    Impact scope
    Common causes
    Troubleshooting steps
    Common commands
    Judgment criteria
    Temporary containment
    Root cause repair
    Rollback method
    Related Dashboard
    Related log query
    Escalation contacts

### 27.2 Example: CUDA OOM Runbook

    Alert name:
        CUDALogOOMDetected

    Meaning:
        CUDA out of memory appears in AI Pod logs.

    Impact:
        Inference request failure, training task interruption, Pod restart.

    Troubleshooting:
        1. Check Pod logs
        2. Check GPU memory
        3. Check batch size
        4. Check concurrency
        5. Check model loading method
        6. Check if multiple workers

    Temporary handling:
        Reduce concurrency
        Reduce batch size
        Restart non-core Pod
        Switch to GPU with larger memory

    Long-term repair:
        Model optimization
        FP16/BF16
        Memory leak repair
        Resource isolation
        Establish capacity baseline

### 27.3 Runbook and Alert Association

Alert annotations must include:

    runbook_url

Example:

    annotations:
      runbook_url: "https://wiki.example.com/runbook/cuda-oom"

---

## Twenty-eight, Alert Notification Template

### 28.1 Notification Content Suggestions

Alert notifications should include:

    Status
    Alert name
    Severity level
    Source
    Cluster
    Environment
    Namespace
    App
    Pod
    Node
    Current value
    Threshold
    Time window
    Dashboard
    Log query link
    Runbook
    Automatic diagnostic report
    Recommended action

### 28.2 Poor Example

    ERROR logs too many

Issues:

    Don't know where error.
    Don't know how many errors.
    Don't know if production.
    Don't know how to handle.

### 28.3 Good Example

    [FIRING][warning] AppErrorLogsTooMany

    Cluster: prod-k8s
    Environment: prod
    Namespace: prod
    App: order-api
    Source: Loki
    Time window: 5m
    Current value: 56
    Threshold: > 30

    Recent errors:
      database connection failed
      timeout waiting for connection
      too many connections

    Dashboard:
      https://grafana.example.com/d/app-log?var-app=order-api

    Runbook:
      https://wiki.example.com/runbook/app-error-logs

    Automatic diagnostic:
      https://diagnosis.example.com/report/alert-xxx

---

## Twenty-nine, Alert Storm Handling

### 29.1 Alert Storm Sources

Common causes:

    A core dependency failure
    Database unavailable
    DNS failure
    Network failure
    Log system anomaly
    New version has many errors
    Log alert rule too broad
    No group_by
    No inhibition
    repeat_interval too short

### 29.2 Emergency Strategy

Steps:

    1. Determine if platform-level failure
    2. Check if alerts are concentrated in a namespace/app/node
    3. Keep root cause alerts
    4. Temporarily silence derived alerts
    5. Do not globally silence critical
    6. Notify relevant owner
    7. Review rules after failure recovery

### 29.3 Long-term Governance

Measures:

    Adjust log alert threshold
    Add for time
    Add group_by
    Add inhibition
    Limit info alert notification frequency
    Downgrade in non-production environment
    Special governance for log volume surge
    Filter common harmless logs
    Weekly statistics on noise alerts Top 10

---

## Thirty, Log Alert and Automation Response Dashboard

### 30.1 Overview Dashboard

Panel: /think

Current Log Alert Count  
Critical Log Alert Count  
Warning Log Alert Count  
Top Alert Namespace  
Top Alert App  
Top Error Keywords  
Auto Diagnosis Success Rate  
Auto Diagnosis Failure Rate  
Webhook Notification Success Rate  
Alert Processing Average Duration  

### 30.2 Log Alert Trends  

Panels:  

    ERROR Log Trend  
    timeout Log Trend  
    Exception Trend  
    CUDA OOM Trend  
    Database Connection Failure Trend  
    Daily Alert Count  
    Weekly Noise Alert Top  

### 30.3 Automation Response Dashboard  

Panels:  

    Diagnosis Task Count  
    Diagnosis Task Success Rate  
    Diagnosis Task Average Duration  
    Diagnosis Job Failure Count  
    Auto Fix Execution Count  
    Manual Confirmation Count  
    Automation Failure Reason Top  
    Throttled Count  

---

## Thirty-one, Production Deployment Phases  

### 31.1 Phase One: Only Notification  

Goals:  

    Log alerts can trigger  
    AlertManager can receive  
    Notifications can reach the team  

Scope:  

    ERROR Count  
    CUDA OOM  
    timeout  
    database connection failed  

### 31.2 Phase Two: Auto Diagnosis  

Goals:  

    Automatically collect context after alert trigger  
    Generate diagnosis report  
    Push report link  

Actions:  

    Read-only queries  
    Do not modify resources  

This is the recommended phase for production.  

### 31.3 Phase Three: Semi-Automatic Response  

Goals:  

    Execute low-risk actions after manual confirmation  

Actions:  

    Restart non-core Pods  
    Create work orders  
    Temporary silence  
    Trigger rollback process  
    Scale non-core services  

### 31.4 Phase Four: Limited Auto Fix  

Goals:  

    Auto fix only for low-risk, deterministic, and reversible scenarios  

Examples:  

    Auto clean failed test environment Jobs  
    Auto restart non-production stateless services  
    Auto close expired diagnosis tasks  
    Auto generate resource governance work orders  

Production core systems should not directly auto fix.  

---

## Thirty-two, Acceptance Checklist  

### 32.1 Log Alert Acceptance  

    [ ] Loki / ELK can query logs  
    [ ] Logs contain namespace / pod / app fields  
    [ ] Can count ERROR quantity  
    [ ] Can count timeout quantity  
    [ ] Can count CUDA OOM  
    [ ] Log alerts can trigger  
    [ ] Alerts can enter AlertManager  
    [ ] Alerts can notify the correct team  
    [ ] Alerts have runbook_url  
    [ ] Alerts have dashboard_url  

### 32.2 Auto Diagnosis Acceptance  

    [ ] Webhook can receive AlertManager alerts  
    [ ] Can parse alert labels  
    [ ] Can query Kubernetes API  
    [ ] Can query Prometheus  
    [ ] Can query Loki / ELK  
    [ ] Can generate diagnosis report  
    [ ] Can push report link  
    [ ] Diagnosis service has read-only permissions  
    [ ] Diagnosis failures have error logs  
    [ ] Diagnosis tasks have timeout control  

### 32.3 Security Acceptance  

    [ ] Webhook has authentication  
    [ ] Automation service uses minimal permissions  
    [ ] No high-risk actions auto-executed  
    [ ] All actions have audit  
    [ ] Has throttling strategy  
    [ ] Has failure retry strategy  
    [ ] Has manual confirmation mechanism  
    [ ] Secrets not written in plain text to Git  

### 32.4 Governance Acceptance  

    [ ] Alerts have owner  
    [ ] Alerts have severity  
    [ ] Alerts have tiered routing  
    [ ] Has inhibition  
    [ ] Has silence standard  
    [ ] Has noise alert statistics  
    [ ] Has alert retrospective template  
    [ ] Has Runbook maintenance mechanism  

---

## Thirty-three, Common Misconceptions  

### 33.1 Misconception One: Automation Response is Auto Restart  

Error.  

Automation response first priority is:  

    Auto diagnosis  
    Auto collect context  
    Auto generate report  

Not restart immediately.  

### 33.2 Misconception Two: Log with ERROR Must Alert  

Error.  

Must combine:  

    Environment  
    Service importance  
    Error quantity  
    Time window  
    Error type  
    Business impact  

### 33.3 Misconception Three: Automation Permissions More is Better  

Error.  

More permissions, bigger accidents.  

Automation service defaults to read-only.  

Write operations must be strictly controlled.  

### 33.4 Misconception Four: Log Alerts Can Replace Metric Alerts  

Error.  

Core business availability should prioritize metric alerts.  

Log alerts are supplementary.  

### 33.5 Misconception Five: All Alerts Should Auto Fix  

Error.  

Only consider auto fix for low-risk, deterministic, reversible, and low-impact scenarios.  

### 33.6 Misconception Six: Diagnosis Report Longer is Better  

Error.  

Diagnosis report should be structured clearly, highlighting:  

    Key phenomenon  
    Key evidence  
    Possible causes  
    Next steps  

Do not blindly pile all command outputs.  

---

## Thirty-four, Complete Case Study: Service 5xx + Log Alert + Auto Diagnosis  

### 34.1 Phenomenon  

Prometheus triggers:  

    ServiceHigh5xxErrorRate  

Loki simultaneously triggers:  

    DatabaseConnectionErrorLogs  

### 34.2 AlertManager Grouping  

Group by the following labels:  

    alertname  
    cluster  
    namespace  
    app  

Merge related service alerts into one notification.  

### 34.3 Webhook Trigger Diagnosis  

Auto diagnosis service executes:  

    1. Query Service  
    2. Query Endpoints  
    3. Query backend Pod  
    4. Query recent error logs  
    5. Query Prometheus 5xx and latency  
    6. Query Pod Restart  
    7. Generate report  

### 34.4 Diagnosis Report Conclusion  

Example:  

    Conclusion:  
        order-api 5xx increase coincides with database connection failure.  

    Evidence:  
        5xx error rate 18% in last 5 minutes.  
        87 database connection failed logs in Loki.  
        Endpoints normal.  
        No obvious Pod restart.  
        P95 latency rises from 120ms to 2.4s.

## Recommendations

    Check database connection count.
    Check database Service reachability.
    Check recent configuration changes.
    Rollback order-api configuration if necessary.

### 34.5 Manual Handling

On-duty personnel should:

    Check database connection pool
    Fix configuration
    Or rollback version

### 34.6 Post-Incident Review

Supplement:

    Database connection pool monitoring
    Database connection failure log alerts
    order-api Runbook
    Pre-release configuration validation

---

## Thirty-Five, Complete Case: CUDA OOM Automatic Diagnosis

### 35.1 Alert

Loki:

    CUDALogOOMDetected

Prometheus:

    GPUMemoryUsageHigh

### 35.2 Automatic Diagnosis

Collect:

    Pod logs
    Pod describe
    GPU node
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_GPU_UTIL
    Pod restart
    Recent batch size logs
    Model loading logs

### 35.3 Report Example

    Conclusion:
        ai-prod/inference-xxx experienced CUDA OOM, with GPU memory usage approaching 98% at the time.

    Evidence:
        Loki detected CUDA out of memory.
        DCGM_FI_DEV_FB_USED remained at high levels.
        Pod restarted 2 times in the last 10 minutes.
        Logs show batch_size=64.

    Possible Causes:
        Batch size is too large.
        High concurrency.
        Model reloaded repeatedly.
        Memory leak.

    Recommendations:
        Reduce batch size.
        Reduce concurrency.
        Check worker count.
        Check model loading logic.
        Migrate to a GPU with larger memory if necessary.

### 35.4 Reason for Not Automatic Restart

Because:

    Restart cannot resolve the root cause of memory insufficiency.
    May cause business jitter.
    Need to first confirm batch/concurrency/model strategy.

---

## Thirty-Six, Post-Incident Review Template

Record after every important log alert and automated response:

    Alert Name:
    Alert Source:
    Alert Level:
    Trigger Time:
    Recovery Time:
    Affected Service:
    Affected Namespace:
    Affected Pod:
    Log Keywords:
    Alert Threshold:
    Actual Error Count:
    Is False Positive:
    Is Missed Alert:
    Is Duplicate Alert:
    Was Automated Diagnosis Successful:
    Was Diagnosis Report Useful:
    Was Automated Action Safe:
    Need to Adjust Threshold:
    Need to Adjust for Time:
    Need to Add Filtering Conditions:
    Need to Add Inhibition:
    Need to Update Runbook:
    Need to Add Metric Alert:
    Root Cause:
    Handling Actions:
    Subsequent Improvements:

---

## Thirty-Seven, Production Recommendations

### 37.1 Prioritize Automatic Diagnosis, Not Rushing to Automatic Fix

Most effective and lowest-risk implementation:

    Automatically collect context after alert
    Generate diagnosis report
    Push to on-duty personnel

This is more reliable than blind automatic restarts.

### 37.2 Log Alerts Should Be Few and Accurate

Prioritize alerts:

    panic
    fatal
    CUDA OOM
    database connection failed
    timeout surge
   Obviously. abnormal production ERROR count
    Critical business error codes

Avoid full ERROR flooding.

### 37.3 All Automation Must Be Auditable

Automation without audit cannot be deployed in production.

### 37.4 Alerts and Runbook Must Be Bound

Critical alerts without Runbook are incomplete.

### 37.5 Automated Responses Must Have Circuit Breakers

When alert storms occur, automated systems must be able to limit flow and break circuits.

Otherwise, it may cause secondary incidents.

---

## Thirty-Eight, Summary

The goal of log alerts and automated responses is not to let the system "fix itself randomly", but to provide evidence faster, locate directions faster, and assist recovery faster when a failure occurs.

Complete chain is:

    Application logs
      ↓
    Loki / ELK
      ↓
    Log alerting rules
      ↓
    AlertManager
      ↓
    Webhook alert gateway
      ↓
    Automated diagnosis service
      ↓
    Kubernetes / Prometheus / Loki / ELK queries
      ↓
    Diagnosis report
      ↓
    ChatOps / ticket / Runbook
      ↓
    Manual confirmation or low-risk automatic fix
      ↓
    Post-mortem and rule optimization

Log alerts are suitable for discovering:

    Critical error patterns
    Application exception stack traces
    Database connection anomalies
    CUDA OOM
    timeout
    panic
    fatal
    Business error codes

Metric alerts are suitable for discovering:

    Availability
    Error rate
    Latency
    Resource usage
    Pod status
    GPU metrics
    Node status

Combined, they form a complete production observability.

Automated responses should be advanced in stages:

    First stage:
        Notification

    Second stage:
        Automatic diagnosis

    Third stage:
        Semi-automated response

    Fourth stage:
        Limited automatic fix

Production environment recommends prioritizing:

    Log alerts + automatic diagnosis report + Runbook + manual confirmation

This approach has low risk and high benefits, suitable for Kubernetes, GPU, AI, business services, and middleware operations scenarios.

---

## Reference Documents

- Grafana Loki Alerting and Recording Rules:
  https://grafana.com/docs/loki/latest/alert/

- Grafana Loki LogQL:
  https://grafana.com/docs/loki/latest/query/

- Grafana Alerting:
  https://grafana.com/docs/grafana/latest/alerting/

- Prometheus Alertmanager:
  https://prometheus.io/docs/alerting/latest/alertmanager/

- Alertmanager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- Prometheus Alerting Practices:
  https://prometheus.io/docs/practices/alerting/

- Kubernetes Jobs:
  https://kubernetes.io/docs/concepts/workloads/controllers/job/

- Kubernetes RBAC:
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Elasticsearch Alerting / Kibana Alerting:
  https://www.elastic.co/docs/explore-analyze/alerts-cases

- OpenSearch Alerting:
  https://docs.opensearch.org/latest/observing-your-data/alerting/

- Grafana Alloy:
  https://grafana.com/docs/alloy/latest/