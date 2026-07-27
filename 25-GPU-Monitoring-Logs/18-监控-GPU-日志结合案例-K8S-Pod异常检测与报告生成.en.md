# 18-Monitoring: GPU and Logs Integration Case - K8S Pod Exception Detection and Report Generation

## Document Description

This document presents a comprehensive Kubernetes observability case study that integrates Prometheus metrics, DCGM GPU metrics, Loki/ELK logging solutions, AlertManager alerts, automated diagnostic scripts, and report generation mechanisms to achieve automatic detection and assistance in troubleshooting issues such as K8S Pod anomalies, GPU errors, application error logs, service 5xx responses, CUDA out of memory (OOM), Pod restarts, and Pod Pending status.

This document does not focus on Prometheus, Grafana, Loki, ELK, or AlertManager individually but rather combines these components into a complete production-grade troubleshooting workflow.

The complete process flow is as follows:

    Kubernetes Pod / GPU Pod
      ↓
    Metric Collection:
        kube-state-metrics
        kubelet/cAdvisor
        node-exporter
        dcgm-exporter
        app/metrics
      ↓
    Log Collection:
        Grafana Alloy/Promtail → Loki
        Filebeat/Fluent Bit → Elasticsearch/OpenSearch
      ↓
    Prometheus/Loki/ELK Querying
      ↓
    PrometheusRule/Loki Ruler/Log Alerts
      ↓
    AlertManager Grouping, Deduplication, and Routing
      ↓
    Webhook Alert Gateway
      ↓
    Automated Diagnostic Services
      ↓
    Kubernetes API/Prometheus API/Loki API/Elasticsearch API
      ↓
    Automatically Generate Diagnostic Reports
      ↓
    Enterprise WeChat/DingTalk/FlyBook/Email/Work Tickets/Runbooks
      ↓
    Manual Confirmation, Semi-automatic Fixing, and Retrospective Learning

This document addresses the following key questions:

- How to integrate metrics, logs, alerts, and automated diagnostics into a production-grade troubleshooting workflow?
- How to detect issues such as Pod Pending, CrashLoopBackOff, OOMKilled, and excessive restarts?
- How to monitor GPU memory usage, temperature, XID, and CUDA OOM?
- How to retrieve critical error logs using Loki/ELK?
- How to correlate Prometheus alerts with log alerts?
- How to unify alerts through AlertManager?
- How to trigger automated diagnostics via Webhook?
- How to automatically generate readable fault diagnosis reports?
- How to design report content, templates, diagnostic commands, and judgment logic?
- How to prevent automated system failures?
- How to document diagnostic findings in Runbooks?

---

## Tags

#Kubernetes #Prometheus #Grafana #AlertManager #Loki #ELK #DCGMExporter #GPU Monitoring #Log Alerts #Automated Diagnosis #Fault Reports #SRE #Observability #Ops Practices

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/05-Fundamentals of Observability/18-Monitoring: GPU and Logs Integration Case - K8S Pod Exception Detection and Report Generation.md

---

## I. Why a Comprehensive Case Study is Needed

Previous chapters covered the following topics:

    Prometheus:
        Metric collection, PromQL, alert rules.

    Grafana:
        Dashboards, panels, variables, visualization tools.

    AlertManager:
        Alert grouping, deduplication, suppression, silence, routing, notifications.

    Loki:
        Kubernetes log collection, LogQL queries, log alerts.

    ELK/EFK:
        Full-text log search, indexing, Kibana queries, log management.

    GPU Monitoring:
        DCGM Exporter, GPU usage, memory, temperature, power consumption, XID.

    Automated Response:
    Webhooks, diagnostic jobs, Runbooks, ChatOps, automatic reports.

However, real-world production issues rarely occur in isolation within these components.

For example:

    If users report an increase in 500 errors from the AI inference API, it may be accompanied by:

    - A higher service 5xx error rate
    - Increased P95 latency
    - Frequent Pod restarts
    - CUDA out of memory errors in logs
    - Over 95% GPU memory usage
   - A certain Pod being killed due to OOM
    - Elevated temperature on a specific GPU node
    - Multiple alerts triggered by AlertManager

Analyzing these issues individually makes it difficult to quickly identify the root cause. Comprehensive troubleshooting requires integrating these signals:

    Metrics help identify "where the issue lies"
    Logs provide insights into "why the issue occurred"
    Kubernetes Events show "what actions were taken"
    `kubectl describe` reveals "resource status and events"
    GPU metrics indicate "if hardware resources are abnormal"
    Automated diagnostic reports offer "current evidence and next steps"

Therefore, the goal of this document is to integrate all these concepts into a practical production workflow.

---

## II. Case Objectives

This case study aims to achieve the following capabilities:

### 2.1 Metric Detection Capability

Ability to detect through Prometheus:

    - Pod Pending status       v                             v
+--------------+             +----------------+
| Prometheus   |             | Loki / ELK     |
| PromQL       |             | LogQL / KQL    |
| Rules        |             | Log Alerts     |
+------+-------+             +--------+-------+
       |                              |
       |                              |
       +--------------+---------------+
                      |
                      v
             +----------------+
             | AlertManager   |
             | Grouping/ Deduplication/ Routing |
             +--------+-------+
                      |
                      v
             +----------------+
             | Webhook Gateway   |
             +--------+-------+
                      |
                      v
             +---------------------+
             | Automatic Diagnosis Service        |
             | ---------------------|
             | K8S API             |
             | Prometheus API      |
             | Loki API            |
             | ELK API             |
             +----------+----------+
                        |
                        v
             +---------------------+
             | Diagnostic Reports            |
             | Markdown / HTML     |
             | IM / Tickets / Wiki    |
             +---------------------+

---

## IV. Experimental Environment Planning

### 4.1 Kubernetes Cluster

Example Environment:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22
    k8s-gpu-node01  10.0.0.30

Namespaces:

    monitoring
    logging
    ai-prod
    app-prod
    default

### 4.2 Monitoring Components

Recommended Components:

    kube-prometheus-stack
    Prometheus
    AlertManager
    Grafana
    node-exporter
    kube-state-metrics
    dcgm-exporter

### 4.3 Logging Components

Choose one or both:

    Loki + Grafana Alloy

or:

    Elasticsearch / OpenSearch + Filebeat / Fluent Bit + Kibana

### 4.4 GPU Components

Recommended:

    NVIDIA Driver
    NVIDIA Container Toolkit
    NVIDIA Device Plugin
    or NVIDIA GPU Operator
    DCGM Exporter

### 4.5 Automation Components

Recommended:

    alert-webhook-gateway
    alert-diagnosis-service
    diagnosis-report-storage
    ChatOps webhook
    Runbook Documentation Library

---

## V. Preliminary Verification

### 5.1 Verifying Prometheus

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring
    kubectl get prometheus -n monitoring
    kubectl get servicemonitor -A
    kubectl get prometheusrule -A

Access Prometheus:

    kubectl port-forward svc/<prometheus-service-name> 9090:9090 -n monitoring

Open:

    http://127.0.0.1:9090

Verify Metrics:

    up
    kube_pod_status_phase
    kube_pod_container_status_restarts_total
    container_cpu_usage_seconds_total
    container_memory_working_set_bytes

### 5.2 Verifying GPU Metrics

Prometheus Query:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE
    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_POWER_usage
    DCGM_FI_DEV_XID_errors

If no data is available, check:

    kubectl get pods -A | grep -i dcgm
    kubectl get svc -A | grep -i dcgm
    kubectl logs <dcgm-exporter-pod> -n <namespace>

### 5.3 Verifying Loki

Grafana Explore Query:

    {namespace=~".+"}

View Logging Components:

    kubectl get pods -n logging
    kubectl logs -n logging <alloy-pod>
    kubectl logs -n logging <loki-pod>

### 5.4 Verifying ELK / OpenSearch

Kibana / OpenSearch Dashboards Query:

    kubernetes.namespace : "app-prod"

API Query:

    GET _cat/indices?v

    GET logs-k8s-app-*/_search
    {
      "size": 1,
      "sort": [
        { "@timestamp": "desc" }
      ]
    }

### 5.5 Verifying AlertManager

    kubectl get pods -n monitoring | grep alertmanager
    kubectl get svc -n monitoring | grep alertmanager

Access:

    kubectl port-forward svc/<alertmanager-service-name> 9093:9093 -n monitoring

Open:

    http://127.0.0.1:9093

---

## VI. Abnormality Detection Targets

This case focuses on detecting six types of abnormalities.

### 6.### Exceptions
Traceback
Panic
No space left on device
ImagePullBackOff

### 6.6 Monitoring Linkage Errors

This includes:

- Prometheus Target Down
- node-exporter Down
- kube-state-metrics Down
- dcgm-exporter Down
- No logs available from Loki
- Failure in sending data via Filebeat/Fluent Bit
- Failure in AlertManager notifications

---

## VII. Test Application Design

To verify the entire operational chain, a simulated application can be prepared.

### 7.1 Regular Web Applications

Namespace:

    app-prod

Application:

    app-demo

Functions:

    /healthz
    /ready
    /api/success
    /api/error
    /api/slow
    /metrics

Simulated Errors:

- Return a 500 error code
- Log ERROR messages
- Log timeout errors
- Consume excessive CPU resources
- Use up all memory
- Terminate the process and trigger a restart

### 7.2 GPU/AI Simulation Applications

Namespace:

    ai-prod

Application:

    ai-inference-demo

Functions:

- Load models
- Simulate inference requests
- Expose /metrics data
- Generate JSON logs

Simulated Errors:

- Out of memory error in CUDA
- Model loading failure
- Too large batch size
- Inference timeout
- High GPU memory usage
- Pod restart

### 7.3 Log Format Recommendations

It is recommended to use JSON format for log output:

    {
      "timestamp": "2026-04-30T12:00:00+08:00",
      "level": "error",
      "service": "ai-inference-demo",
      "trace_id": "abc123",
      "msg": "CUDA out of memory",
      "model": "demo-model",
      "batch_size": 64,
      "duration_ms": 1200
    }

Recommended fields:

- timestamp
- level
- service
- trace_id
- msg
- status
- duration_ms
- error
- model
- batch_size

Note:

- The `trace_id` should not be used as a label in Loki.
- The `trace_id` can be retained as a log field.

---

## VIII. Prometheus Metric Monitoring Rules

### 8.1 Pod Pending

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: pod-detect-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: pod.detect.rules
          rules:
            - alert: PodPendingTooLong
              expr: kube_pod_status_phase{phase="Pending"} == 1
              for: 10m
              labels:
                severity: warning
                team: sre
                source: prometheus
              annotations:
                summary: "The Pod has been in a Pending state for too long."
                description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been pending for more than 10 minutes."
                runbook_url: "https://wiki.example.com/runbook/pod-pending"

### 8.2 Excessive Pod Restarts

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: pod-restart-rules
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
                source: prometheus
              annotations:
                summary: "The Pod has restarted too frequently."
                description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has restarted more than 3 times in the last 10 minutes."
                runbook_url: "https://wiki.example.com/runbook/pod-restart"

### 8.3 Pod OOMKilled

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: pod-oom-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: pod.oom.rules
          rules:
            - alert: PodOOMKilled
              expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
              for: 1m
              labels:
                severity: warning
                team: sre
                source: prometheus
              annotations:
                summary: "The Pod was terminated due to OOM."
                descriptionlabels:
    severity: critical
    team: app
    source: prometheus

annotations:
    summary: "The 5xx error rate of the service is too high"
    description: "The 5xx error rate for the service {{ $labels.namespace }}/{{ $labels.service }} exceeds 5%."

runbook_url: "https://wiki.example.com/runbook/service-5xx"

### 8.5 High P95 Service Latency

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
    name: service-latency-rules
    namespace: monitoring
    labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: service.latency.rules
          rules:
            - alert: ServiceHighP95Latency
              expr: |
                histogram_quantile(
                  0.95,
                  sum by (le, namespace, service) (
                    rate(http_request_duration_seconds_bucket[5m])
                  )
                ) > 1
              for: 5m
              labels:
                severity: warning
                team: app
                source: prometheus
              annotations:
                summary: "The P95 latency of the service is too high"
                description: "The P95 latency for the service {{ $labels.namespace }}/{{ $labels.service }} exceeds 1 second."

runbook_url: "https://wiki.example.com/runbook/service-latency"

---

## IX. GPU Metric Monitoring Rules

### 9.1 High GPU Memory Usage

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
    name: gpu-memory-rules
    namespace: monitoring
    labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: gpu.memory.rules
          rules:
            - alert: GPUMemoryUsageHigh
              expr: |
                (
                  DCGM_FI_DEV_FB_USED
                  /
                  (DCGM_FI_dev_FB_used + DCGM_FIDEV_FB_FREE)
                  * 100
                ) > 90
              for: 10m
              labels:
                severity: warning
                team: ai-platform
                source: prometheus
              annotations:
                summary: "The GPU memory usage is too high"
                description: "The memory usage of GPU {{ $labels.gpu }} on {{ $labels.Hostname }} exceeds 90%."

runbook_url: "https://wiki.example.com/runbook/gpu-memory-high"

### 9.2 High GPU Temperature

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
    name: gpu-temperature-rules
    namespace: monitoring
    labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: gpu.temperature.rules
          rules:
            - alert: GPUHighTemperature
              expr: DCGM_FI_DEV_GPU_TEMP > 80
              for: 5m
              labels:
                severity: warning
                team: ai-platform
                source: prometheus
              annotations:
                summary: "The GPU temperature is too high"
                description: "The temperature of GPU {{ $labels.gpu }} on {{ $labels.Hostname }} exceeds 80°C."

runbook_url: "https://wiki.example.com/runbook/gpu-high-temperature"

### 9.3 GPU XID Errors

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
    name: gpu-xid-rules
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
                source: prometheus
              annotations:
                summary: "A GPU XID error has occurred"
                description: "A GPU XID error has been detected on GPU {{ $labels.gpu }} on {{ $labels.Hostname }}. It is recommended to check the dmesg, journalctl, and GPU health status."

runbook_url: "https://wiki.example.com/runbook/gpu-xid-error"

### 9.4 DCGM Exporter Down

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
    name: dcgm-exporter-rules
    namespace: monitoring
    labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: dcgm.exporter.rules
          rules:
            - alert: DCGMExporterDown
              expr: up{job=~".*dcgm.*"} == 0
              for: 5m
              labels:
                severity: warning
                team: ai-platform                source: prometheus
              annotations:
                summary: "DCGM Exporter is unavailable"
                description: "The DCGM Exporter target {{ $labels.instance }} has been down for more than 5 minutes."
                runbook_url: "https://wiki.example.com/runbook/dcgm-exporter-down"

---

## Section X: Loki Log Monitoring Rules

### 10.1 Excessive ERROR Logs

    groups:
      - name: app-log-detect-rules
        rules:
          - alert: AppErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace=~"app-prod|ai-prod"} |~ "(?i)error|exception|panic" [5m])
              ) > 30
            for: 5m
            labels:
              severity: warning
              team: app
              source: loki
            annotations:
              summary: "Too many application error logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has more than 30 error logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-error-logs"

### 10.2 Excessive Timeout Logs

    groups:
      - name: app-timeout-log-rules
        rules:
          - alert: AppTimeoutLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace=~"app-prod|ai-prod"} |~ "(?i)timeout|timed out|deadline exceeded" [5m])
              ) > 10
            for: 5m
            labels:
              severity: warning
              team: app
              source: loki
            annotations:
              summary: "Excessive application timeout logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has more than the threshold number of timeout logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-timeout"

### 10.3 Database Connection Failures

    groups:
      - name: app-db-log-rules
        rules:
          - alert: DatabaseConnectionErrorLogs
            expr: |
              sum by (namespace, app) (
                count_over_time({namespace="app-prod"} |~ "(?i)database connection failed|too many connections|connection refused|access denied" [5m])
              ) > 5
            for: 5m
            labels:
              severity: warning
              team: database
              source: loki
            annotations:
              summary: "Too many database connection error logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has experienced multiple database connection errors in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/database-connection-error"

### 10.4 CUDA Out of Memory

    groups:
      - name: gpu-log-rules
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
              description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has experienced CUDA out of memory in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/cuda-oom"

### 10.5 Model Loading Failures

    groups:
      - name: ai-model-log-rules
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
              summary: "Model loading failures"
              description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has logs indicating model loading failures."
              runbook_url: "https://wiki.example.com/runbook/model-load-failed"

---

## Section XI: ELK / OpenSearch Log Monitoring Queries

If using ELK / OpenSearch, you can use KQL or Query DSL for monitoring.

### 11.1 Query ERROR Logs

KQL:

    kubernetes.namespace : "app-prod" and log.level : "error"

### 11.2 Query Timeout Logs

    kubernetes.namespace : "app-prod" and message :### 11.5 Example of Query DSL

    GET logs-k8s-app-*/_search
    {
      "size": 0,
      "query": {
        "bool": {
          "filter": [
            { "term": { "kubernetes.namespace": "app-prod" } },
            {
              "range": {
                "@timestamp": {
                  "gte": "now-5m",
                  "lte": "now"
                }
              }
            }
          ],
          "must": [
            {
              "query_string": {
                "query": "message:(\"connection refused\" OR \"database connection failed\" OR \"too many connections\")"
              }
            }
          ]
        }
      }
    }

---

## Section Twelve: AlertManager Alarm Association Design

### 12.1 Unified Tags

To merge Prometheus and Loki alerts, it is necessary to unify the tags used.

It is recommended to use the following unified tags:

    cluster
    environment
    namespace
    app
    service
    pod
    node
    team
    severity
    source

Example of a PrometheusRule:

    labels:
      severity: warning
      team: ai-platform
      source: prometheus
      environment: prod

Example of a Loki Rule:

    labels:
      severity: warning
      team: ai-platform
      source: loki
      environment: prod

### 12.2 group_by Design

It is recommended to group alerts by the following dimensions:

    group_by:
      - alertname
      - cluster
      - namespace
      - app

For GPU scenarios, additional grouping by node and gpu can be added.

Example:

    route:
      receiver: default-webhook
      group_by:
        - cluster
        - namespace
        - app
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

### 12.3 Inhibition Rule Design

If there is already an alarm for a core business metric, such as ServiceHigh5xxErrorRate, it is possible to suppress low-priority noise alerts for the same service, like AppErrorLogsTooMany.

Example:

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

    Inhibition does not mean deletion.
    Log alerts can still be used as diagnostic context in reports.
    It is not recommended to excessively suppress critical log alerts.

---

## Section Thirteen: Webhook Alarm Gateway Design

### 13.1 Webhook Input

AlertManager sends JSON data to the webhook gateway.

The gateway receives:

    receiver
    status
    alerts
    groupLabels
    commonLabels
    commonAnnotations
    externalURL

### 13.2 Gateway Processing Flow

    1. Receive the AlertManager Webhook request.
    2. Verify the Token and perform signature verification.
    3. Parse the received alerts.
    4. Determine the diagnostic type based on the alertname.
    5. Generate diagnostic tasks based on namespace, pod, node, or app.
    6. Call the automatic diagnostic service.
    7. Generate a report.
    8. Send notifications.
    9. Record audit details.

### 13.3 Mapping of alertnames to Diagnostic Types

    PodPendingTooLong:
        pod_scheduling_diagnosis

    PodRestartTooOften:
        pod_restart_diagnosis

    PodOOMKilled:
        pod_oom_diagnosis

    ServiceHigh5xxErrorRate:
        service_error_diagnosis

    AppErrorLogsTooMany:
        app_log_diagnosis

    CUDALogOOMDetected:
        gpu_cuda_oom_diagnosis

    GPUMemoryUsageHigh:
        gpu_memory_diagnosis

    GPUXIDError:
        gpu_xid_diagnosis

    DCGMExporterDown:
        gpu_monitoring_diagnosis

### 13.4 Security Requirements

The webhook gateway must have the following features:

    HTTPS
    Authentication
    IP Whitelist
    Rate Limiting
    Idempotence
    Auditing
    Timeout Control
    Retry Mechanism
    Protection against Flood of Alerts

---

## Section Fourteen: Automatic Diagnostic Service Design

### 14.1 Input for the Diagnostic Service

Input fields include:

    alertname
    severity
    cluster
    environment
    namespace
    app
    service
    pod
    container
    node
    gpu
    startsAt
    generatorURL
    dashboard_url
    runbook_url

### 14.2 Output of the Diagnostic Service

The output structure includes:

    diagnosis_id
    title
    summary
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
      apiGroup: rbacauthorization.k8s.io

---

## Sixteen, Diagnostic Report Templates

### 16.1 Report Structure

It is recommended to use Markdown format for reports.

Template:

    # Fault Diagnosis Report

    ## I. Alarm Summary

    - Alarm Name:
    - Alarm Level:
    - Alarm Source:
    - Cluster:
    - Environment:
    - Namespace:
    - App:
    - Pod:
    - Node:
    - Trigger Time:
    - Current Status:

    ## II. Impact Scope

    - Affected Services:
    - Affected Pods:
    - Affected Nodes:
    - Affected GPUs:
    - Whether Production:
    - Whether Core Business:

    ## III. Metric Evidence

    - CPU:
    - Memory:
    - Restart:
    - Error Rate:
    - Latency:
    - GPU Memory:
    - GPU Utilization:
    - GPU Temperature:
    - XID:

    ## IV. Log Evidence

    - Latest ERROR:
    - Latest Timeout:
    - Latest Exception:
    - Latest CUDA OOM:
    - Key Log Fragments:

    ## V. Kubernetes Events

    - Pod Phase:
    - Container State:
    - Last State:
    - Events:
    - Deployment Replicas:
    - Service Endpoints:

    ## VI. Preliminary Judgment

    - Possible Cause 1:
    - Possible Cause 2:
    - Possible Cause 3:

    ## VII. Recommended Actions

    - Temporary Fix:
    - Root Cause Investigation:
    - Long-Term Optimization:

    ## VIII. Related Links

    - Grafana Dashboard:
    - Loki Explore:
    - Kibana Discover:
    - Runbook:
    - AlertManager:

    ## IX. Original Commands

    - kubectl Commands:
    - PromQL:
    - LogQL:
    - KQL:

---

## Seventeen, Pod Pending Diagnosis Logic

### 17.1 Triggering Alarms

    PodPendingTooLong

### 17.2 Automatic Data Collection

Commands:

    kubectl get pod <pod> -n <namespace> -o wide

    kubectl describe pod <pod> -n <namespace>

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -50

Prometheus:

    kube_pod_status_phase{namespace="<namespace>", pod="<pod>", phase="Pending"}

### 17.3 Key Decisions

If the following appears in Events:

    insufficient cpu

Conclusion:

    Insufficient CPU resources.

If the following appears in Events:

    insufficient memory

Conclusion:

    Insufficient memory resources.

If the following appears in Events:

    insufficient nvidia.com/gpu

Conclusion:

    Insufficient GPU resources or GPU not registered.

If the following appears in Events:

    pod has unbound immediate PersistentVolumeClaims

Conclusion:

    PVCs are not bound.

If the following appears in Events:

    node(s) had untolerated taint

Conclusion:

    Taint/toleration mismatch.

If the following appears in Events:

    didn't match Pod's node affinity/selector

Conclusion:

    NodeSelector or affinity does not match.

### 17.4 Recommended Action Steps

Output:

    1. Check Pod Events.
    2. Verify Node Allocatable Resources.
    3. Check Namespace Resource Quotas.
    4. Inspect PVC Status.
    5. Verify GPU Device Plugins.
    6. Adjust resources, scheduling rules, or expand nodes based on the root cause.

---

## Eighteen, Pod Restart Diagnosis Logic

### 18.1 Triggering Alarms

    PodRestartTooOften

### 18.2 Automatic Data Collection

Commands:

    kubectl get pod <pod> -n <namespace> -o wide

    kubectl describe pod <pod>### 23.1 Bash Version Approach

Suitable for simple environments.

Script input:

    ALERT_NAME
    NAMESPACE
    POD
    NODE
    APP
    SERVICE

Output:

    diagnosis-report.md

Example:

    #!/bin/bash
    set -euo pipefail

    ALERT_NAME="${ALERT_NAME:-unknown}"
    NAMESPACE="${NAMESPACE:-default}"
    POD "${POD:-}"
    NODE "${NODE:-}"
    SERVICE "${SERVICE:-}"
    REPORT="/tmp/diagnosis-report.md"

    echo "# Failure Diagnosis Report" > "${REPORT}"
    echo "" >> "${REPORT}"

    echo "## I. Alert Summary" >> "${REPORT}"
    echo "- Alert Name: ${ALERT_NAME}" >> "${REPORT}"
    echo "- Namespace: ${NAMESPACE}" >> "${REPORT}"
    echo "- Pod: ${POD}" >> "${REPORT}"
    echo "- Node: ${NODE}" >> "${REPORT}"
    echo "- Service: ${SERVICE}" >> "${REPORT}"
    echo "" >> "${REPORT}"

    if [ -n "${POD}" ]; then
      echo "## II. Pod Status" >> "${REPORT}"
      kubectl get pod "${POD}" -n "${NAMESPACE}" -o wide >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT"}

      echo "## III. Pod Description" >> "${REPORT}"
      kubectl describe pod "${POD}" -n "${NAMESPACE}" >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"

      echo "## IV. Current Pod Logs" >> "${REPORT}"
      kubectl logs "${POD}" -n "${NAMESPACE}" --tail=100 >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"

      echo "## V. Previous Pod Logs" >> "${REPORT}"
      kubectl logs "${POD}" -n "${NAMESPACE}" --previous --tail=100 >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"
    fi

    echo "## VI. Recent Events" >> "${REPORT}"```markdown
kubectl get events -n "${NAMESPACE}" --sort-by=.lastTimestamp | tail -50 >> "${REPORT}" 2>&1 || true

cat "${REPORT}"

### 23.2 Python Version Approach

Python is more suitable for production scalability.

Module Design:

    alert_parser.py
        Parses AlertManager Webhooks.

    k8s_client.py
        Queries Pod, Service,Endpoints, Events.

    prometheus_client.py
        Queries PromQL.

    loki_client.py
        Queries LogQL.

    elasticsearch_client.py
        Queries KQL / Query DSL.

    reportrenderer.py
        Generates Markdown / HTML reports.

    notifier.py
        Sends notifications via WeCom, DingTalk, Lark, email.

    main.py
        Main entry point.

### 23.3 Report Generation Process

    1. Parse alerts
    2. Determine diagnostic type
    3. Query Kubernetes status
    4. Query Prometheus metrics
    5. Query Loki / ELK logs
    6. Generate preliminary findings
    7. Create Markdown report
    8. Upload the report
    9. Send notifications

---

## Chapter 24: Diagnostic Job Examples

### 24.1 Job Purpose

Used to execute one-time diagnostic tasks within a cluster.

Suitable for:

    Collecting kubectl output
    Pulling logs
    Generating reports
    Sending results

### 24.2 Sample Job YAML

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: diagnose-pod-sample
      namespace: monitoring
    spec:
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          serviceAccountName: alert-diagnosis
          restartPolicy: Never
          containers:
            - name: diagnose
              image: registry.example.com/sre/diagnosis-toolkit:v1.0.0
              env:
                - name: ALERT_NAME
                  value: "PodRestartTooOften"
                - name: NAMESPACE
                  value: "app-prod"
                - name: POD
                  value: "app-demo-xxx"
              command:
                - /bin/bash
                - -c
                - |
                  set -euo pipefail

                  REPORT="/tmp/report.md"

                  echo "# Fault Diagnosis Report" > ${REPORT}
                  echo "" >> ${REPORT}
                  echo "## Pod Status" >> ${REPORT}
                  kubectl get pod ${POD} -n ${NAMESPACE} -o wide >> ${REPORT} 2>&1 || true

                  echo "" >> ${REPORT}
                  echo "## Pod Describe" >> ${REPORT}
                  kubectl describe pod ${POD} -n ${NAMESPACE} >> ${REPORT} 2>&1 || true

                  echo "" >> ${REPORT}
                  echo "## Current Logs" >> ${REPORT}
                  kubectl logs ${POD} -n ${NAMESPACE} --tail=100 >> ${REPORT} 2>&1 || true

                  echo "" >> ${REPORT}
                  echo "## Previous Logs" >> ${REPORT}
                  kubectl logs ${POD} -n ${NAMESPACE} --previous --tail=100 >> ${REPORT} 2>&1 || true

                  echo "" >> ${REPORT}
                  echo "## Events" >> ${REPORT}
                  kubectl get events -n ${NAMESPACE} --sort-by=.lastTimestamp | tail -50 >> ${REPORT} 2>&1 || true

                  cat ${REPORT}

### 24.3 Precautions

Do not use:

    latest

For production, use a fixed image version:

    registry.example.com/sre/diagnosis-toolkit:v1.0.0

The image should include:

    kubectl
    curl
    jq
    yq
    bash
    python3
    dig
    nslookup
    iproute2

---

## Chapter 25: Report Distribution Design

### 25.1 Content of Notifications

Alert notifications should not include the entire long report.

It is recommended that group messages contain:

    Alert summary
    Impact scope
    Preliminary conclusion
    Top 3 key findings
    Recommended actions
    Report link
    Dashboard link
    Runbook link

The complete report should be stored in:

    Object storage
    Wiki
    Ticket system
    Internal diagnostic platform

### 25.2 Message Example

    [FIRING][warning] CUDALogOOMDetected

    Cluster: prod-k8s
    Namespace: ai-prod
    Pod: ai-inference-demo-xxx
    Preliminary conclusion: CUDA Out of Memory detected; GPU memory usage is nearly 98%.

    Key evidence:
    1. Loki shows CUDA out of memory.
    2. DCGM_FI_DEV_FB_used remains high.
    3.## Thirty-Five, Production Implementation Recommendations### 35.1 Phase One: Integrating Alerts and Reports

Objectives:

    Prometheus alerts
    Loki / ELK alerts
    AlertManager notifications
    Webhook reception
    Automatic generation of basic reports

### 35.2 Phase Two: Improving Diagnostic Logic

Additions:

    Pod Pending diagnosis
    Pod Restart diagnosis
    Service 5xx diagnosis
    CUDA OOM diagnosis
    GPU XID diagnosis
    Log anomaly diagnosis

### 35.3 Phase Three: Integrating with ChatOps

Implementations:

    View reports
    View logs
    View dashboards
    Silent alerts
    Create tickets
    Execute low-risk actions after manual confirmation

### 35.4 Phase Four: Limited Automatic Repair

Only for low-risk scenarios:

    Restart non-production Pods
    Clean up failed jobs in the test environment
    Manually confirm scale-out for non-core services
    Automatically create resource governance tickets

Automatic repair is not recommended for production core systems.

---

## Section Thirty-Six: Summary

This document presents a comprehensive case study on integrating Prometheus, Grafana, AlertManager, Loki, ELK, and GPU monitoring with automated response mechanisms.

A complete production-grade observability loop should include:

    Detection of metric anomalies
      ↓
    Additional log analysis to determine causes
      ↓
    Unified alert management
      ↓
    Automatic diagnostic evidence collection
      ↓
    Generation of structured reports
      ↓
    Delivery to the appropriate teams
      ↓
    Handling according to established runbooks
      ↓
    Review and optimization of rules

Role allocation among different systems:

    Prometheus:
        Metric collection, PromQL, metric alerts.

    DCGM Exporter:
        GPU utilization, memory, temperature, power consumption, XID.

    Loki:
        Kubernetes log aggregation, LogQL, log alerts.

    ELK / OpenSearch:
        Full-text search, field queries, auditing, and complex log analysis.

    AlertManager:
        Alert grouping, deduplication, suppression, silencing, routing.

    Webhook Gateway:
        Notification integration, alert storage, diagnostic triggers.

    Automatic Diagnostic Services:
        Querying K8S, Prometheus, Loki, ELK, report generation.

    Grafana:
        Dashboards, metric and log integration, troubleshooting tools.

The true goal of production-grade systems is not just to have charts, logs, and alerts, but to:

    Identify anomalies effectively
    Manage alerts efficiently
    Automatically collect relevant context
    Make reports understandable
    Have established processes for handling issues
    Ensure automation has appropriate boundaries
    Learn from past incidents for continuous improvement
    Continuously optimize rules and processes

For Kubernetes GPU / AI infrastructure operations, this suite of capabilities can address:

    Abnormal Pod scheduling
    Application runtime issues
    Service access problems
    GPU resource discrepancies
    CUDA OOM errors
    Model loading failures
    Sudden log errors
    Alert storms
    Monitoring linkages failures

Ultimately, this approach creates a production-ready SRE observability and automated response framework.