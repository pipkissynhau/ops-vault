# 17-Practical Log Alerts and Automated Responses

## Documentation Overview

This document systematically outlines the design of log alerts, log alert rules, AlertManager notifications, Webhook alert gateways, automated responses, ChatOps, Runbooks, fault diagnosis jobs, semi-automatic recovery processes, and production alert governance methods in a Kubernetes environment.

This document does not simply discuss issuing alerts whenever an "ERROR" is detected in logs; instead, it aims to establish a practical log alert and automated response system from the perspective of operations and SRE.

This document addresses the following key questions:

- Why can't all ERROR logs be directly used to trigger alerts?
- What are the differences between log alerts and metric alerts?
- What roles do Loki, ELK/EFK, Prometheus, and AlertManager play in the alert mechanism?
- How should log alert rules be designed?
- How can Loki Ruler be utilized for log alerts?
- How can Elasticsearch/OpenSearch be used for log alerts?
- How can log alerts be sent to AlertManager?
- How can grouping, suppression, silencing, and routing be implemented through AlertManager?
- How should a Webhook alert gateway be designed?
- How can alerts automatically trigger diagnostic scripts?
- Which actions are suitable for automated execution, and which require manual confirmation?
- How should Kubernetes automatic diagnosis jobs be designed?
- How should alerts be responded to for issues such as Pod failures, CUDA OOM, Service 5xx errors, database connection failures, and image pull failures?
- How can alert storms and incorrect automated actions be avoided?
- How can a closed-loop of "alert → diagnosis → handling → review" be established?

This document is recommended for reading after completing the following topics:

- 11-Prometheus - Architecture and Core Metric Analysis
- 12-Grafana - Dashboard Construction and Custom Monitoring
- 13-AlertManager - Alert Strategies and Notification Implementation
- 14-K8S - Monitoring Practices - Node, Pod, Service Metric Collection and Troubleshooting
- 15-Loki - Log Collection and Querying Practices
- 16-ELK-EFK - Log Collection and Retrieval Practices

---

## Tags

#Kubernetes #Loki #ELK #EFK #AlertManager #Webhook #Log Alerts #Automated Responses #Runbook #ChatOps #SRE #Fault Diagnosis #Observability #GPU Operations

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/05-Foundation of Observability/17-Practical Log Alerts and Automated Responses.md

---

## I. Why Are Log Alerts and Automated Responses Needed?

Prometheus metric alerts can inform you of issues such as:

    High CPU usage
    High memory consumption
    Pod restarts
    Pending Pods
    Increasing service error rates
    High GPU temperatures
    High GPU memory usage

However, metrics usually cannot directly tell you why certain issues occurred, such as:

    Why an application failed to start
    Which configuration setting is missing
    Which database connection failed
    Which interface experienced a timeout
    In which task the CUDA OOM occurred
    Why a model loading failed
    What the content of a Python Traceback is
    The specific stack trace of a Java Exception
    Which upstream server returned a 502 error for Nginx
    The exact error causing an image pull failure

This type of information typically comes from logs.

The value of log alerts lies in:

    Identifying critical error patterns before metrics detect issues;
    Providing additional context quickly after metric alerts are triggered;
    Supplementing monitoring for issues that cannot be directly measured by metrics;
    Real-time detection of abnormal keywords, error codes, stack traces, and business failure reasons;
    Reducing the need for manual, repetitive troubleshooting through automated diagnosis.

The value of automated responses lies in:

    Automatically collecting relevant context after an alert is triggered;
    Automatically querying Pod, Service, Endpoints, and Events;
    Automatically retrieving recent error logs;
    Automatically generating diagnostic reports;
    Automatically sending notifications to the alert team;
    Automating low-risk actions;
    Providing a manual confirmation option for high-risk actions.

The ultimate goal is not to "automatically restart everything" but to:

    First, perform automated diagnosis;
    Then, assist in making judgments;
    Finally, restore systems with caution.

---

## II. Differences Between Log Alerts and Metric Alerts

### 2.1 When Metric Alerts Are Suitable

Metric alerts are suitable for monitoring stable, quantifiable, and aggregable conditions.

Examples:

    Node CPU > 90%
    Pod Pending > 10 minutes
    Service 5xx error rate > 5%
    P95 latency > 1 second
    GPU temperature > 80°C
    GPU memory usage > 90%
    Prometheus target down
    Insufficient available Deployment replicas

Characteristics:

    Clear numerical values
    Clear trends
    Easy to aggregate## V. Log Alert Classification Design

### 5.1 Critical

Applicable to:

- A large number of 5xx logs occurring in core production services;
- Panic/fatal errors in production services;
- Persistent failures in the main database connection;
- A significant number of authentication failures that may pose security risks;
- Serious GPU XID errors;
- CUDA OOM causing the production inference service to become unavailable;
- The log platform becoming unreadable;
- A large amount of logs being lost;
- Continuous occurrence of critical error codes in core services.

Notifications:

    OnCall
    High-priority WeCom/DingTalk groups
    Telephone/Text message
    Incident response group

### 5.2 Warning

Applicable to:

- The number of ERROR logs exceeding the baseline;
- Abnormal stack traces in a specific Pod;
- An increase in the number of database connection failures;
    An increase in timeouts;
    Occasional CUDA OOM incidents;
    Abnormally high log volumes in a certain Namespace;
    An increase in errors in non-core services.

Notifications:

    Operations team group
    Business team group
    Ticket system
    Handling during working hours

### 5.3 Info

Applicable to:

- Excessive Debug logs;
    Alerts for underutilized GPUs;
    Error logs from non-production environments;
    Notifications when log volumes reach the top threshold;
    Reminders related to resource management.

Notifications:

    Daily reports
    Weekly reports
    Low-priority groups

---

## VI. Principles for Designing Log Alert Rules

### 6.1 Rules Must Identify a Clear Object

Alerts should clearly specify:

- Which cluster
- Which environment
- Which Namespace
- Which service
- Which Pod
- Which container
- What type of error
- Within what time frame
- How many times it has occurred

Do not simply state:

    Log exception

### 6.2 Rules Must Include a Time Window

Log alerts must use a time window.

For example:

    In the last 5 minutes
    In the last 10 minutes
    In the last 15 minutes

Avoid triggering alerts for individual logs without a time frame.

### 6.3 Rules Must Have Thresholds

Examples:

    More than 20 ERROR events within 5 minutes
    More than 10 timeouts within 10 minutes
    No CUDA OOM incidents within 5 minutes
    No panic events within 5 minutes
    More than 100 authentication failures within 5 minutes

Thresholds vary for different types of errors.

### 6.4 Rules Must Exclude Meaningless Logs

Common logs to filter out include:

    /healthz
    /ready
    /metrics
    /favicon.ico
    liveness probe
    readiness probe
    Known benign errors
    Debug logs
    Test Namespace logs

### 6.5 Rules Must Have an Owner

Each alert should have:

    A responsible team
    The associated service
    An owner
    A runbook_url

Alerts without an owner should not be included in the production real-time notification system.

---

## VII. Basics of Loki Log Alerts

### 7.1 Role of Loki Ruler

Loki Ruler allows you to create:

    Alerting Rules
    Recording Rules

Based on LogQL, log alerts typically involve counting the number of logs within a specific time window.

For example:

    The number of ERROR logs in the last 5 minutes
    The number of CUDA OOM incidents in the last 10 minutes
    The number of timeouts in the last 15 minutes

These counts are then sent to the AlertManager.

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
              summary: "Too many application ERROR logs"
              description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has more than 20 ERROR logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-error-logs"

### 7.3 count_over_time

This function counts the number of logs within a specified time range.

Example:

    count_over_time({namespace="prod"} |= "ERROR" [5m])

Meaning:

    The number of ERROR logs in the last 5 minutes.

### 7.4 rate

This function calculates the rate at which logs occur.

Example:

    rate({namespace="prod"} |= "ERROR" [5m])

Meaning:

    The number of ERROR logs per second in the last               runbook_url: "https://wiki.example.com/runbook/database-connection-error"

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
              summary: "Too many Java exception logs"
              description: "Too many Java exception logs have been detected in {{ $labels.namespace }}/{{ $labels.app }} in the last 5 minutes."
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
              description: "A Python Traceback was found in {{ $labels.namespace }}/{{ $labels.app }} within the last 5 minutes."
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
              summary: "CUDA OOM logs detected"
              description: "CUDA out of memory errors were reported in pod {{ $labels.namespace }}/{{ $labels.pod }} within the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/cuda-oom"

### 8.7 Model Loading Failure

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
              summary: "Model loading failed"
              description: "Failures in model loading were recorded for pod {{ $labels.namespace }}/{{ $labels.pod }}."
              runbook_url: "https://wiki.example.com/runbook/model-load-failed"

---

## IX. ELK / OpenSearch Log Alerting Strategies

### 9.1 Kibana Alerts

Kibana allows creating rules based on Elasticsearch queries.

Common approach:

    Query the logs-k8s-app-* index
    Filter by the prod namespace
    Filter for log.level:error
    Count the occurrences in the last 5 minutes
    Trigger an alert if the count exceeds 20

### 9.2 OpenSearch Alerting

OpenSearch Dashboards can be used to create Monitors.

Basic structure:

    Monitor:
        Regularly queries the index

    Trigger:
        Checks if a threshold is reached

    Action:
        Sends a notification

### 9.3 Example of Elasticsearch Query DSL for Alerts

To count error logs in the prod namespace for the last 5 minutes:

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

### 9.4 Example of KQL for Alerts

    kubernetes.namespace : "prod" and log.level : "error"

Or:

    kubernetes.namespace : "prod" and message : "CUDA out of memory"

Or:

    kubernetes.namespace : "prod" and message : "connection refused"

### 9.5 Production### 10.2 AlertManager Routing Examples

```yaml
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
receptors:
  - name: default-webhook
    webhook_configs:
      - url: http://alert-webhook.monitoring.svc:8080/default
      - send_resolved: true

  - name: sre-critical
    webhookconfigs:
      - url: http://alert-webhook.monitoring.svc:8080/sre-critical
      - send_resolved: true

  - name: ai-platform
    webhook_configs:
      - url: http://alert-webhook_monitoring.svc:8080/ai-platform
      - send_resolved: true

  - name: app-team
    webhookconfigs:
      - url: http://alert-webhook.monitoring.svc:8080/app-team
      - send_resolved: true

  - name: log-info-digest
    webhook_configs:
      - url: http://alert-webhook_monitoring.svc:8080/info
      - send_resolved: false
```

### 10.3 Log Alarm Suppression

If a service already has an indicator alarm, such as `ServiceHigh5xxErrorRate`, lower-level log alarms for the same service can be suppressed, for example, `AppErrorLogsTooMany`.

**Example Approach:**  
If `ServiceHigh5xxErrorRate` is at the critical level, suppress all `AppErrorLogsTooMany` warnings within the same namespace/app.

**Configuration Example:**  
```yaml
inhibit_rules:
  - source_matchers:
    - alertname="ServiceHigh5xxErrorRate"
    - severity="critical"
  target_matchers:
    - alertname="AppError LogsTooMany"
    - severity="warning"
  equal:
    - cluster
    - namespace
    - app
```

**Note:**  
Suppression should be used cautiously. Do not completely block valuable root cause log alarms. It is more common to group log alarms within the same context rather than suppress them entirely.

---

## Chapter Eleven: Webhook Alarm Gateway Design

### 11.1 Why an Alarm Gateway is Needed

Although AlertManager can directly send Webhooks, a middle gateway is typically required in production environments for several reasons:

- To integrate with enterprise communication tools like WeChat Work, DingTalk, or Lark.
- To standardize the message format and authentication process.
- To implement rate limiting and traffic management.
- To provide unified logging and audit trails.
- To ensure consistent alarm storage and processing.
- To trigger automated responses in a coordinated manner.
- To generate diagnostic reports systematically.
- To link alarms with dashboards and runbooks effectively.

**Recommended Architecture:**  
```plaintext
AlertManager
↓
alert-webhook-gateway
↓
Notification Channels
↓
Automated Diagnostic Services
↓
ChatOps / Tickets / Reports
```

### 11.2 Responsibilities of the Alarm Gateway

The alarm gateway can perform the following tasks:

- Receive JSON alerts from AlertManager.
- Verify the signature or token for authenticity.
- Parse the alerts and route them based on specified labels.
- Format the messages appropriately for delivery.
- Retrieve additional context data as needed.
- Trigger automated diagnostic processes.
- Create service tickets or notifications.
- Log audit events related to alert processing.
- Implement rate limiting and deduplication mechanisms.
- Handle failed requests with retries.

### 11.3 Key Information in Webhook Payloads

AlertManager typically includes the following information in its payloads:

- `receiver`: The recipient of the alert.
- `status`: The current status of the alert.
- `alerts`: A list of alerts related to this event.
- `groupLabels`: Labels used for grouping alerts.
- `commonLabels`: Common labels applied across alerts.
- `commonAnnotations`: Additional annotations associated with the alerts.
- `externalURL`: An external URL that may be relevant to the alert.

Each alert contains details such as:

- `labels`: Specific identifiers used for routing and processing.
- `annotations`: Supplementary information about the alert.
- `startsAt`: The start time of the event.
-GPU Utilization
GPU Memory
GPU Temperature

### 13.5 Recommended Actions

Possible Causes
Troubleshooting Commands
Repair Suggestions
Runbook Links
Dashboard Links
Whether Manual Intervention Is Recommended
Whether Automatic Repair Is Allowed

---

## Chapter Fourteen: Design of Automatic Diagnosis Service

### 14.1 Service Responsibilities

Upon receiving an alarm, the automatic diagnosis service executes the corresponding diagnostic process based on the alertname.

Examples:

    PodRestartTooOften:
        Collects Pod status, previous round of logs, Events, and resource usage.

    CUDALogOOMDetected:
        Collects Pod logs, GPU memory, nvidia-smi results, and Pod resource declarations.

    ServiceHigh5xxErrorRate:
        Collects Service, Endpoints, error logs, QPS, and latency data.

    DatabaseConnectionErrorLogs:
        Collects application logs, Service DNS, and database service availability information.

### 14.2 Recommended Architecture

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
    IM / Ticketing / Dashboard

### 14.3 Permission Control

The diagnostic service requires access to:

    Kubernetes API
    Prometheus API
    Loki API
    Elasticsearch / OpenSearch API

However, permissions must be minimized.

Kubernetes Permission Recommendations:

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

Automatic repair actions require separate authorization and it is best to use dedicated service accounts.

---

## Chapter Fifteen: Kubernetes Automatic Diagnosis RBAC Examples

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

    apiVersion: rbacauthorization.k8s.io/v1
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
      apiGroup: rbacauthorization.k8s.io

### 15.4 Security Notes

These permissions are for read-only purposes only.

Suitable for automatic diagnosis.

If automatic repair is required, a separate ServiceAccount should be created with strict restrictions:

    Only allow access to specified namespaces.
    Only allow access to specific resources.
    Only allow specific actions.
    Auditing must be implemented.
    Rate limiting is necessary.
    Manual confirmation should always be required.

---

## Chapter Sixteen: Automatic Diagnosis Command Templates

### 16.1 Pod Exception Diagnosis

    kubectl get pod <pod> -n <namespace> -o wide

    kubectl describe pod <pod> -n <namespace>

    kubectl logs <pod> -n <namespace> --tail=100

    kubectl logs <pod> -n <namespace> --previous --tail=100

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -50

### 16.2 Service Exception Diagnosis

    kubectl get svc <service> -n <namespace> -o wide

    kubectl describe svc <service> -n <namespace>

    kubectl get endpoints <service> -n <namespace>

    kubectl get endpointslice -n <namespace> | grep <service>

CUDA OOM:

    {namespace="<namespace>", pod="<pod>"} |= "CUDA out of memory"

Timeout:

    {namespace="<namespace>", pod("<pod>"} |~ "(?i)timeout|timed out|deadline exceeded"

---

## Section Seventeen: Automated Diagnostic Job Design

### 17.1 Why Use Jobs

After an alarm is triggered, a Kubernetes Job can be created to collect diagnostic information.

Advantages:

    Consistent with the cluster environment
    Can use kubectl commands
    Terminates automatically after completion
    Allows for log recording
    Different Jobs can be created based on alarms
    Permissions can be controlled by ServiceAccounts

### 17.2 Example of a Diagnostic Job

This example is for illustrative purposes only; in production, parameters such as namespace, pod, and node should be dynamically generated by the alarm gateway.

```yaml
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

              echo "===== Basic Pod Info ====="
              kubectl get pod "${POD}" -n "${NAMESPACE}" -o wide || true

              echo "===== Pod Description ====="
              kubectl describe pod "${POD}" -n "${NAMESPACE}" || true

              echo "===== Current Pod Logs ====="
              kubectl logs "${POD}" -n "${NAMESPACE}" --tail=100 || true

              echo "===== Previous Pod Logs ====="
              kubectl logs "${POD}" -n "${NAMESPACE}" --previous --tail=100 || true

              echo "===== Recent Events ====="
              kubectl get events -n "${NAMESPACE}" --sort-by=.lastTimestamp | tail -50 || true
```

### 17.3 Notes

Do not use the `latest` image version.

In production, use a fixed image version:

    bitnami/kubectl:<version>

Or create your own diagnostic image:

    registry.example.com/sre/diagnosis-toolkit:v1.0.0

The diagnostic image should include tools such as:

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

## Section Eighteen: ChatOps Response Design

### 18.1 What is ChatOps

ChatOps integrates alarm notifications, diagnostics, approvals, and execution tasks into chat tools.

For example:

    Alarms are sent to enterprise WeChat groups
    Diagnostic reports are displayed in the group
    Operation buttons or commands are provided
    Manual confirmation is required before executing fixes
    Operation results are reported back to the group

### 18.2 Common Interactions

Alarm messages:

    [FIRING] PodRestartTooOften
    namespace: prod
    pod: api-xxx
    restart: 5 times / 10m

Buttons:

    View Logs
    View Events
    Perform Diagnosis
    Create Ticket
    Silence for 30 Minutes
    Restart Pod
    Open Runbook

### 18.3 Operation Levels

Actions that can be performed directly:

    View Logs
    Perform Diagnosis
    Create Ticket
    Open Dashboard
    Open Runbook

Actions that require confirmation:

    Restart Pod
    Expand Deployment
    Roll Back Version
    Isolate Nodes

Actions that should not be enabled in chat tools:

    Delete PVCs
    Delete Databases
    Remove Indexes
    Batch Restart Production Services
    Drain Production Nodes
    Modify Production Security Policies

### 18.4 Audit Requirements

All ChatOps operations must be recorded, including:

    Operator
    Operation Time
    Alarm ID
    Target Resource
    Action Taken
    Result
    Approver
    Original Request
    Response

---

## Section Nineteen: Automated Response Scenario 1: Excessive Pod Restarts

### 19.1 Source of Alarms

Prometheus:

    PodRestartTooOften

Loki:

    AppErrorLogsTooMany
    PythonTracebackDetected
    JavaExceptionLogsTooMany

### 19.2 Automatic Diagnostic Steps

Collect the following information:

    kubectl get pod
    kubectl describe pod
    kubectl logs --tail=100
    kubectl logs --previous --tail=100
    Recent Events
    Pod CPU / Memory Usage
    Number of Restarts
    Last#### Priority Check: Database Connection Pool, Database Status, and Network

If the issue increases after the Pod restarts:

    Combine with Pod restart diagnostics.

### 20.6 Automatic Recommendations

Suggestions:

    Check recent releases
    Verify dependent services
    Monitor database status
    Review configuration changes
    Roll back if necessary

Automatic rollback is not recommended unless there is a strict release system and approval process in place.

---

## Chapter 21: Automation Response Scenario 3: CUDA Out of Memory

### 21.1 Sources of Alerts

Loki:

    CUDALogOOMDetected

Prometheus:

    GPUMemoryUsageHigh
    PodRestartTooOften
    ServiceHigh5xxErrorRate

### 21.2 Automatic Diagnostic Actions

Collect:

    Pod logs
    Pod resource declarations
    GPU node where the Pod resides
    GPU memory metrics
    GPU utilization
    GPU temperature
    Recent Pod restarts
    CUDA OOM log context
    Business request volume
    batch size-related logs

### 21.3 Prometheus Queries

GPU Memory Usage:

    DCGM_FI_DEV_FB_USED

GPU Memory Free:

    DCGM_FI_DEV_FB_FREE

GPU Utilization:

    DCGM_FI_DEV_GPU_UTIL

### 21.4 Loki Queries

    {namespace="<namespace>", pod="<pod>"} |= "CUDA out of memory"

    {namespace="<namespace>", pod="<pod>"} |~ "(?i)batch|model|worker|memory"

### 21.5 Reasoning Logic

Possible causes:

    Excessive batch size
    Large model size
    High concurrency
    Too many workers
    Repeated loading of multiple models
    Memory leak
    GPU memory competition due to sharing
    Insufficient memory in MIG instances

### 21.6 Automatic Recommendations

Suggestions:

    Reduce the batch size
    Lower concurrency levels
    Decrease the number of workers
    Review model loading strategies
    Check for repeated model loading
    Use a GPU with more memory
    Adopt FP16 / BF16 / quantization techniques
    Increase memory monitoring alerts

Automatic Pod deletion is not recommended.

Semi-automatic actions include:

    Restart non-production inference Pods
    Reduce concurrency in non-production replicas
    Generate a ticket for the AI platform team

---

## Chapter 22: Automation Response Scenario 4: Database Connection Failure

### 22.1 Sources of Alerts

Loki:

    DatabaseConnectionErrorLogs

Prometheus:

    ServiceHigh5xxErrorRate
    ServiceHighP95Latency

### 22.2 Automatic Diagnostic Actions

Collect:

    Error logs
    Keywords related to connection failures
    Affected services
    Affected namespaces
    Service endpoints
    DNS resolution results
    Availability of the database service
    Whether application configuration secrets/configmaps have recently changed
    Business error rates and latency levels

### 22.3 Common Log Keywords

    connection refused
    too many connections
    database connection failed
    timeout
    access denied
    password authentication failed
    no route to host
    connection reset by peer

### 22.4 Reasoning Logic

connection refused:

    The database port is unreachable, or the service is not listening.

too many connections:

    The number of database connections has been exhausted.

access denied:

    Issues with username/password or permissions.

timeout:

    Network issues, slow database response, connection pool exhaustion, or firewall problems.

no route to host:

    Network routing or NetworkPolicy-related issues.

### 22.5 Automatic Recommendations

Suggestions:

    Monitor database performance
    Check the connection pool settings
    Verify if there have been any changes to secrets
    Review NetworkPolicy configurations
    Check DNS settings
    Ensure the database service is accessible

Automatic database restarts are not recommended.

---

## Chapter 23: Automation Response Scenario 5: Sudden Surge in Log Volume

### 23.1 Sources of Alerts

Loki / ELK:

    A sudden increase in log volume for a particular namespace
    An explosion of ERROR logs for a specific application
    Abnormal log write volume for a certain Pod

### 23.2 Automatic Diagnostic Actions

Collect:

    Top namespaces with highest log volumes
    Top Pods generating the most logs
    Most prevalent error keywords
    Whether debug mode is enabled
    Whether there are excessive health check logs
    Whether there are abnormal loopback stack prints

### 23.3 Reasoning Logic

Common causes:

    Debug mode is enabled
    Abnormal loopback retries
    Excessive health check logs
    A sudden surge in access requests
    Dependency issues leading to log flooding
    New versions generating a large volume of logs
    Issues with multi-line log splitting

### 23.4 Automatic Recommendations

Suggestions:

    Lower the log level
    Filter out### Alarm Name:
        CUDALogOOMDetected

**Meaning:**
        CUDA out of memory has been detected in the AI Pod logs.

**Impact:**
        Reason for推理 request failures, interruption of training tasks, and Pod restarts.

**Troubleshooting Steps:**
1. Check Pod logs.
2. Verify GPU memory usage.
3. Examine the batch size.
4. Review concurrent tasks.
5. Inspect how the model is loaded.
6. Determine if multiple workers are being used.

**Temporary Solutions:**
- Reduce concurrent tasks.
- Lower the batch size.
- Restart non-core Pods.
- Switch to a GPU with more memory.

**Long-term Fixes:**
- Optimize the model.
- Consider using FP16/BF16.
- Fix memory leaks.
- Implement resource isolation.
- Establish capacity benchmarks.

### 27.3 Association between Runbooks and Alarms

Alarm annotations must include:

    `runbook_url`

**Example:**

```json
annotations:
  runbook_url: "https://wiki.example.com/runbook/cuda-oom"
```

---

## Chapter 28: Alarm Notification Templates

### 28.1 Recommended Notification Content

Notification messages should include:

    Status
    Alarm Name
    Severity Level
    Source
    Cluster
    Environment
    Namespace
    App
    Pod
    Node
    Current Value
    Threshold
    Time Window
    Dashboard Link
    Log Query Link
    Runbook
    Automatic Diagnosis Report
    Recommended Actions

### 28.2 Example of Inadequate Notification

    **Error logs too many**

**Problems:**
- It's unclear where the errors occurred.
- The number of errors is unknown.
- It's uncertain whether these are production-related issues.
- There's no guidance on how to handle them.

### 28.3 Example of Proper Notification

    **[FIRING][warning] AppErrorLogsTooMany**

    - Cluster: prod-k8s
    - Environment: prod
    - Namespace: prod
    - App: order-api
    - Source: Loki
    - Time Window: 5 minutes
    - Current Value: 56
    - Threshold: > 30

    **Recent Errors:**
    - Database connection failed.
    - Timeout waiting for connection.
    - Too many connections.

    **Dashboard Link:**
    https://grafana.example.com/d/app-log?var-app=order-api

    **Runbook Link:**
    https://wiki.example.com/runbook/app-error-logs

    **Automatic Diagnosis Link:**
    https://diagnosis.example.com/report/alert-xxx

---

## Chapter 29: Handling Alarm Storms

### 29.1 Common Causes of Alarm Storms

    - Failure of a critical dependency.
    - Unavailability of the database.
    - DNS issues.
    - Network failures.
    - Abnormalities in the logging system.
    - A large number of errors due to new versions.
   - Excessively broad log alert rules.
    - Lack of `group_by`.
    - Absence of inhibition mechanisms.
    - Too short `repeat_interval`.

### 29.2 Emergency Response Steps

    1. Determine if it's a platform-level issue.
    2. Check whether alarms are concentrated in specific namespaces/app nodes.
    3. Retain the root cause alarm.
    4. Temporarily silence derived alarms.
    5. Avoid globally silencing critical alarms.
    6. Notify relevant owners immediately.
    7. Review and adjust rules after the issue is resolved.

### 29.3 Long-term Solutions

    - Adjust log alert thresholds.
    - Increase the `for` time period.
   - Add more `group_by` fields.
   - Enhance inhibition mechanisms.
    - Limit the frequency of info-level alarm notifications.
   - Downgrade non-production environments.
   - Implement separate handling for sudden increases in log volume.
    - Filter out common, harmless logs.
    - Regularly rank the top 10 most frequent noise alarms.

---

## Chapter 30: Log Alerts and Automated Response Dashboards

### 30.1 Overview Dashboard

Panels include:

- Current number of log alerts.
- Number of Critical alerts.
- Number of Warning alerts.
- Top-ranked namespaces for alerts.
- Top-ranked apps for alerts.
- Most common error keywords.
- Success rate of automatic diagnostics.
- Failure rate of automatic diagnostics.
- Success rate of webhook notifications.
- Average time taken to handle alerts.

### 30.2 Log Alert Trends

Panels show:

- Trends in ERROR logs.
- Trends in timeout-related logs.
- Trends in Exception logs.
- Trends in CUDA OOM errors.
- Trends in database connection failure incidents.
- Daily number of alerts.
- Top-ranked noise alerts on a weekly basis.

### 30.### 3. Query Backend Pods
### 4. Retrieve Recent Error Logs
### 5. Check Prometheus 5xx Errors and Latencies
### 6. Investigate Pod Restart Events
### 7. Generate Reports

### 34.4 Diagnostic Report Conclusion

Example:

Conclusion:
The increase in order-api 5xx errors coincided with database connection failures.

Evidence:
- The 5xx error rate was 18% in the last 5 minutes.
- There were 87 entries in Loki indicating database connection failures.
- Endpoints were functioning normally.
- No significant Pod restarts were observed.
- The P95 latency increased from 120ms to 2.4s.

Recommendations:
- Check the number of database connections.
- Verify the availability of the database Service.
- Examine recent configuration changes.
- Roll back order-api configurations if necessary.

### 34.5 Manual Intervention

On-duty personnel should take the following actions based on the report:

- Check the database connection pool.
- Fix any configuration issues.
- Or revert to a previous version if needed.

### 34.6 Post-Mortem Analysis

Additional steps include:

- Monitor the database connection pool.
- Set up alerts for database connection failure logs.
- Create an order-api Runbook.
- Perform configuration validation before deployment.

---

## Chapter Thirty-Five: Complete Case Study: Automatic Diagnosis of CUDA Out Of Memory Issues

### 35.1 Alerts

Loki:

CUDALogOOMDetected

Prometheus:

GPUMemoryUsageHigh

### 35.2 Automatic Diagnosis

Collect the following data:

- Pod logs
- Pod description details
- GPU node information
- DCGM_FI_DEV_FB_USED values
- DCGM_FI_DEV_GPU_UTIL values
- Pod restart records
- Recent batch size logs
- Model loading logs

### 35.3 Report Example

Conclusion:
The ai-prod/inference-xxx Pod experienced a CUDA out of memory issue, with the GPU memory utilization approaching 98% at the time of failure.

Evidence:
- Loki detected a CUDA memory shortage.
- DCGM_FI_DEV_FB_USED remained consistently high.
- The Pod restarted twice in the last 10 minutes.
- Logs indicated a batch size of 64.

Possible causes:
- The batch size was too large.
- There was excessive concurrency.
- Models were being loaded repeatedly.
- There might be memory leaks.

Recommendations:
- Reduce the batch size.
- Lower the level of concurrency.
- Check the number of workers involved.
- Re-examine the model loading logic.
- If necessary, switch to a GPU with more memory.

### 35.4 Reason Why Automatic Restart Was Not Implemented

Automatic restarts were avoided because:

- Restarting would not address the root cause of the memory shortage.
- It could potentially cause service disruptions.
- It was necessary to first verify the batch size, concurrency settings, and model execution strategies.

---

## Chapter Thirty-Six: Post-Mortem Analysis Template

After each significant log alert and automated response, record the following information:

- Alert Name:
- Source of the Alert:
- Alert Severity:
- Time of Trigger:
- Time of Resolution:
- Services Affected:
- Names of Namespaces Impacted:
- Pods That Were Affectened:
- Key Log Terms:
- Alert Thresholds:
- Actual Number of Errors Reported:
- Whether It Was a False Positive:
- Whether There Were Any Missed Alerts:
- If the Alert Reoccurred:
- Whether Automatic Diagnosis Was Successful:
- Whether the Diagnostic Report Was Helpful:
- Whether the Automated Action Was Safe:
- Whether It Was Necessary to Adjust the Thresholds:
- Whether Changes Were Needed for Specific Time Periods:
- Whether Additional Filtering Conditions Were Required:
- Whether Inhibition Mechanisms Were Needed:
- Whether Updates to the Runbook Were Needed:
- Whether More Metric Alerts Should Be Added:
- Root Cause of the Issue:
- Actions Taken in Response:
- Steps Taken for Improvement:

---

## Chapter Thirty-Seven: Production Recommendations

### 37.1 Prioritize Automatic Diagnosis Over Immediate Automatic Fixing

The most effective and low-risk approach is to:

- Automatically collect relevant data after an alert is triggered.
- Generate a diagnostic report.
- Send it to the on-duty personnel.

This approach is more reliable than blindly initiating automatic restarts.

### 37.2 Keep Log Alerts Few but Accurate

Give priority to alerts for the following critical events:

- Panic errors
- Fatal errors
- CUDA out of memory issues
- Database connection failures
- Sudden increases in timeout rates
- Abnormally high numbers of production-level errors
- Critical business error codes

Avoid sending out alerts for all minor errors.

### 37.3 All Automated Processes Must Be Auditable

Automated systems that are not auditable should not be deployed in a production environment.

### 37.4