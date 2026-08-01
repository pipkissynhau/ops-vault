# 18 - Monitoring - GPU Logs Combined Case - K8S Pod Anomaly Detection and Report Generation

## Document Description

This document is used to organize a complete Kubernetes observability case: by combining Prometheus metrics, DCGM GPU metrics, Loki / ELK logs, AlertManager alerts, automated diagnostic scripts, and report generation mechanisms, it realizes automatic detection and assisted troubleshooting for K8S Pod anomalies, GPU anomalies, application error logs, service 5xx errors, CUDA OOM, Pod restarts, Pod Pending, and other issues.

This document is not about explaining Prometheus, Grafana, Loki, ELK, or AlertManager separately, but rather connects the previous chapters into a complete production troubleshooting loop.

Complete flowchart:

    Kubernetes Pod / GPU Pod
      ↓
    Metric Collection:
        kube-state-metrics
        kubelet / cAdvisor
        node-exporter
        dcgm-exporter
        app /metrics
      ↓
    Log Collection:
        Grafana Alloy / Promtail → Loki
        Filebeat / Fluent Bit → Elasticsearch / OpenSearch
      ↓
    Prometheus / Loki / ELK Query
      ↓
    PrometheusRule / Loki Ruler / Log Alert
      ↓
    AlertManager Grouping, Deduplication, Routing
      ↓
    Webhook Alert Gateway
      ↓
    Automated Diagnostic Service
      ↓
    Kubernetes API / Prometheus API / Loki API / Elasticsearch API
      ↓
    Automatically Generate Diagnostic Report
      ↓
    Enterprise WeChat / DingTalk / Feishu / Email / Ticket / Runbook
      ↓
    Human Confirmation / Semi-Automatic Fix / Review andDeposition

This document focuses on answering the following questions:

- How to connect metrics, logs, alerts, and automated diagnostics into a production troubleshooting chain;
- How to detect Pod Pending, CrashLoopBackOff, OOMKilled, excessive restarts;
- How to detect high GPU memory usage, high GPU temperature, GPU XID, CUDA OOM;
- How to query critical error logs through Loki / ELK;
- How to associate Prometheus alerts with log alerts;
- How to unify alerts through AlertManager;
- How to trigger automated diagnostics through Webhook;
- How to automatically generate a readable fault diagnosis report;
- How to design report content, report templates, diagnostic commands, and judgment logic;
- How to avoid automated misoperations;
- How toDeposition diagnostic results into Runbook.

---

## Tags

#Kubernetes #Prometheus #Grafana #AlertManager #Loki #ELK #DCGMExporter #GpuSurveillance #It'sALogCall. #Autodiagnosis #FaultReport. #SRE #Observation #It'sABattleOfLuck.

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Foundation/18-Monitoring-GPU-Logs Combined Case-K8S Pod Anomaly Detection and Report Generation.md

---

## One, Why a Comprehensive Case is Needed

The previous chapters separately explained:

    Prometheus:
        Metric collection, PromQL, alerting rules.

    Grafana:
        Dashboard, Panel, variables, visualization.

    AlertManager:
        Alert grouping, deduplication, suppression, silence, routing, notifications.

    Loki:
        Kubernetes log collection, LogQL query, log alerting.

    ELK / EFK:
        Log full-text search, indexing, Kibana query, log governance.

    GPU Monitoring:
        DCGM Exporter, GPU utilization, memory, temperature, power consumption, XID.

    Automated Response:
        Webhook, diagnostic Job, Runbook, ChatOps, automatic report.

But real production environment issues don't appear separately by component.

For example:

    User reports increased AI inference interface 500 errors

May simultaneously appear:

    Service 5xx error rate increases
    P95 latency increases
    Pod restart count increases
    Logs show CUDA out of memory
    GPU memory usage exceeds 95%
    A Pod is OOMKilled
    A GPU node temperature increases
    AlertManager receives multiple alerts

If only looking at individual systems, it's hard to quickly locate the issue.

Comprehensive troubleshooting should connect these signals:

    Metrics tell you "where the anomaly is"
    Logs tell you "why the anomaly occurred"
    Events tell you "what K8S did"
    kubectl describe tells you "resource status and events"
    GPU metrics tell you "whether hardware resources are abnormal"
    Automated diagnostic report tells you "current evidence and next steps"

Therefore, this article's goal is to integrate all previous knowledge into a production-grade practical workflow.

---

## Two, Case Objectives

This case ultimately achieves the following capabilities:

### 2.1 Metric Detection Capabilities

Can detect through Prometheus:

    Pod Pending
    Pod excessive restarts
    Pod OOMKilled
    Pod high CPU usage
    Pod high memory usage
    Service high 5xx error rate
    Service high P95 / P99 latency
    GPU high memory usage
    GPU high temperature
    GPU XID error
    DCGM Exporter Down
    Prometheus Target Down

### 2.2 Log Detection Capabilities

Can detect through Loki / ELK:

    Excessive ERROR logs
    Excessive timeout logs
    database connection failed
    connection refused
    Java Exception
    Python Traceback
    panic / fatal
    CUDA out of memory
    model load failed
    NCCL error
    no space left on device
    ImagePullBackOff related errors

### 2.3 Alert Correlation Capabilities

Can consolidate related alerts for the same fault:

ServiceHigh5xxErrorRate  
AppErrorLogsTooMany  
CUDALogOOMDetected  
GPUMemoryUsageHigh  
PodRestartTooOften  

These alerts, if belonging to the same:  

    cluster  
    namespace  
    app  
    pod  
    node  

Should be merged into a single notification group rather than being sent individually.  

### 2.4 Automatic Diagnostic Capabilities  

After an alert is triggered, automatically collect:  

    Pod status  
    Pod Events  
    Pod current logs  
    Pod previous logs  
    Deployment replica status  
    Service details  
    Endpoints / EndpointSlice  
    Node status  
    Prometheus metrics snapshot  
    Loki / ELK error logs  
    GPU metrics  
    GPU node information  
    Possible causes  
    Recommended remediation actions  

### 2.5 Report Generation Capabilities  

Automatically generate a structured report:  

    Incident summary  
    Impact scope  
    Current status  
    Metric evidence  
    Log evidence  
    Kubernetes events  
    GPU evidence  
    Preliminary judgment  
    Recommended actions  
    Runbook links  
    Dashboard links  
    Post-mortem items  

---

## ThreeI don't know.Case Architecture Overview  

### 3.1 Architecture Diagram  

    +-----------------------------+  
    | Kubernetes Cluster          |  
    |-----------------------------|  
    | Node / Pod / Service        |  
    | GPU Pod / AI Service        |  
    +--------------+--------------+  
                   |  
                   |  
      -------------------------------  
      |                             |  
      v                             v  
+-------------+              +----------------+  
| Metrics     |              | Logs           |  
|-------------|              |----------------|  
| node-exporter              | Alloy/Promtail |  
| kubelet/cAdvisor           | Filebeat       |  
| kube-state-metrics         | Fluent Bit     |  
| dcgm-exporter              | Logstash       |  
| app /metrics               +-------+--------+  
+------+------+                      |  
       |                             |  
       v                             v  
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
             | grouping/deduplication/routing |  
             +--------+-------+  
                      |  
                      v  
             +---------------------+  
             | Webhook Gateway     |  
             +--------+-------+  
                      |  
                      v  
             +---------------------+  
             | Automatic Diagnostic Service |  
             |---------------------|  
             | K8S API             |  
             | Prometheus API      |  
             | Loki API            |  
             | ELK API             |  
             +----------+----------+  
                        |  
                        v  
             +---------------------+  
             | Diagnostic Report   |  
             | Markdown / HTML     |  
             | IM / Work Order / Wiki |  
             +---------------------+  

---

## FourI don't know.Experimental Environment Planning  

### 4.1 Kubernetes Cluster  

Example environment:  

    k8s-master      10.0.0.20  
    k8s-worker01    10.0.0.21  
    k8s-worker02    10.0.0.22  
    k8s-gpu-node01  10.0.0.30  

Namespace: /think

monitoring
logging
ai-prod
app-prod
default

### 4.2 Monitoring Components

Recommended components:

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
    Runbook documentation library

---

## Five. Pre-Validation

### 5.1 Validate Prometheus

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring
    kubectl get prometheus -n monitoring
    kubectl get servicemonitor -A
    kubectl get prometheusrule -A

Access Prometheus:

    kubectl port-forward svc/<prometheus-service-name> 9090:9090 -n monitoring

Open:

    http://127.0.0.1:9090

Validate metrics:

    up
    kube_pod_status_phase
    kube_pod_container_status_restarts_total
    container_cpu_usage_seconds_total
    container_memory_working_set_bytes

### 5.2 Validate GPU Metrics

Prometheus query:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE
    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_POWER_USAGE
    DCGM_FI_DEV_XID_ERRORS

If no data, check:

    kubectl get pods -A | grep -i dcgm
    kubectl get svc -A | grep -i dcgm
    kubectl logs <dcgm-exporter-pod> -n <namespace>

### 5.3 Validate Loki

Grafana Explore query:

    {namespace=~".+"}

View logging components:

    kubectl get pods -n logging
    kubectl logs -n logging <alloy-pod>
    kubectl logs -n logging <loki-pod>

### 5.4 Validate ELK / OpenSearch

Kibana / OpenSearch Dashboards query:

    kubernetes.namespace : "app-prod"

API query:

    GET _cat/indices?v

    GET logs-k8s-app-*/_search
    {
      "size": 1,
      "sort": [
        { "@timestamp": "desc" }
      ]
    }

### 5.5 Validate AlertManager

    kubectl get pods -n monitoring | grep alertmanager
    kubectl get svc -n monitoring | grep alertmanager

Access:

    kubectl port-forward svc/<alertmanager-service-name> 9093:9093 -n monitoring

Open:

    http://127.0.0.1:9093

---

## Six. Anomaly Detection Objects

This case focuses on detecting 6 types of anomalies.

### 6.1 Pod Scheduling Anomalies

Includes:

    Pod Pending
    insufficient CPU
    insufficient memory
    insufficient nvidia.com/gpu
    unbound PVC
    nodeSelector mismatch
    taint/toleration mismatch
    ResourceQuota exceeded

### 6.2 Pod Runtime Anomalies

Includes:

    CrashLoopBackOff
    OOMKilled
    excessive restarts
    livenessProbe failed
    readinessProbe failed
    container startup failure
    configuration error
    missing Secret

### 6.3 Service Access Anomalies

Includes:

    high 5xx error rate
    high P95 / P99 latency
    Service has no Endpoints
    Endpoints backend not Ready
    targetPort error
    DNS resolution failure
    NetworkPolicy blocking

### 6.4 GPU Anomalies

Includes:

    high GPU memory usage
    high GPU temperature
    GPU XID errors
    abnormal GPU utilization
    DCGM Exporter Down
    CUDA OOM
    model loading failure
    NCCL error

### 6.5 Log Anomalies

Includes:

    excessive ERROR logs
    timeout
    connection refused
    database connection failed
    Exception
    Traceback
    panic
    no space left on device
    ImagePullBackOff

### 6.6 Monitoring Link Anomalies

Includes:

    Prometheus Target Down
    node-exporter Down
    kube-state-metrics Down
    dcgm-exporter Down
    Loki No Logs
    Filebeat / Fluent Bit Send Failure
    AlertManager Notification Failure

---

## SevenI don't know.Test Application Design

To verify the complete link, you can prepare a simulation application.

### 7.1 Ordinary Web Application

Namespace:

    app-prod

Application:

    app-demo

Function:

    /healthz
    /ready
    /api/success
    /api/error
    /api/slow
    /metrics

Simulate anomalies:

    Return 500
    Print ERROR log
    Print timeout log
    Occupy CPU
    Occupy memory
    Exit process to trigger restart

### 7.2 GPU / AI Simulation Application

Namespace:

    ai-prod

Application:

    ai-inference-demo

Function:

    Load model
    Simulate inference request
    Expose /metrics
    Output JSON log

Simulate anomalies:

    CUDA out of memory
    model load failed
    batch size too large
    inference timeout
    high GPU memory usage
    Pod restart

### 7.3 Log Format Recommendation

Recommend outputting JSON logs:

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

Field recommendations:

    timestamp
    level
    service
    trace_id
    msg
    status
    duration_ms
    error
    model
    batch_size

Note:

    Do not use trace_id as a Loki label.
    trace_id can be retained as a log field.

---

## EightI don't know.Prometheus Metric Detection Rules

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
                summary: "Pod Pending Time Too Long"
                description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} Pending exceeded 10 minutes."
                runbook_url: "https://wiki.example.com/runbook/pod-pending"

### 8.2 Pod Restart Too Often

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
                summary: "Pod Restart Count Too High"
                description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted more than 3 times in the last 10 minutes."
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
            summary: "Pod was OOMKilled"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} had its last termination reason as OOMKilled."
            runbook_url: "https://wiki.example.com/runbook/pod-oomkilled"

### 8.4 High 5xx Error Rate for Service

Prerequisites:

    Application exposes the http_requests_total metric.
    The metric has service and status labels.

Rules:

    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: service-slo-rules
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      groups:
        - name: service.slo.rules
          rules:
            - alert: ServiceHigh5xxErrorRate
              expr: |
                sum by (namespace, service) (
                  rate(http_requests_total{status=~"5.."}[5m])
                )
                /
                sum by (namespace, service) (
                  rate(http_requests_total[5m])
                )
                * 100 > 5
              for: 5m
              labels:
                severity: critical
                team: app
                source: prometheus
              annotations:
                summary: "High 5xx Error Rate for Service"
                description: "Service {{ $labels.namespace }}/{{ $labels.service }} has a 5xx error rate exceeding 5%."
                runbook_url: "https://wiki.example.com/runbook/service-5xx"

### 8.5 High P95 Latency for Service

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
                summary: "High P95 Latency for Service"
                description: "Service {{ $labels.namespace }}/{{ $labels.service }} has a P95 latency exceeding 1 second."
                runbook_url: "https://wiki.example.com/runbook/service-latency"

---

## NineI don't know.GPU Metric Detection Rules

### 9.1 High GPU Memory Usage

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
                summary: "High GPU Temperature"
                description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} temperature exceeds 80°C."
                runbook_url: "https://wiki.example.com/runbook/gpu-high-temperature"

### 9.3 GPU XID Error

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
                summary: "GPU XID Error"
                description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} has XID errors. Check dmesg, journalctl, and GPU health status."
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
            team: ai-platform
            source: prometheus
          annotations:
            summary: "DCGM Exporter is unavailable"
            description: "DCGM Exporter target {{ $labels.instance }} Down for more than 5 minutes."
            runbook_url: "https://wiki.example.com/runbook/dcgm-exporter-down"

---

## 10. Loki Log Detection Rules

### 10.1 Too Many ERROR Logs

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
              summary: "Too Many Application Error Logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has more than 30 error logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-error-logs"

### 10.2 Too Many timeout Logs

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
              summary: "Too Many Application timeout Logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has exceeded the threshold for timeout-related logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/app-timeout"

### 10.3 Database Connection Failure

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
              summary: "Excessive Database Connection Abnormal Logs"
              description: "{{ $labels.namespace }}/{{ $labels.app }} has experienced multiple database connection abnormalities in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/database-connection-error"

### 10.4 CUDA OOM

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
          summary: "CUDA OOM Log Detected"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has CUDA out of memory logs in the last 5 minutes."
          runbook_url: "https://wiki.example.com/runbook/cuda-oom"

### 10.5 Model Load Failure

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
          summary: "Model Load Failure"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has model load failure logs."
          runbook_url: "https://wiki.example.com/runbook/model-load-failed"

---

## ElevenI don't know.ELK / OpenSearch Log Detection Queries

If using ELK / OpenSearch, use KQL or Query DSL for detection.

### 11.1 Query ERROR Logs

KQL:

    kubernetes.namespace : "app-prod" and log.level : "error"

### 11.2 Query Timeout

    kubernetes.namespace : "app-prod" and message : "timeout"

### 11.3 Query CUDA OOM

    kubernetes.namespace : "ai-prod" and message : "CUDA out of memory"

### 11.4 Query Database Connection Failure

    kubernetes.namespace : "app-prod"
    and (
      message : "database connection failed"
      or message : "too many connections"
      or message : "connection refused"
    )

### 11.5 Query DSL Example

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

## TwelveI don't know.AlertManager Alert Correlation Design

### 12.1 Label Standardization

To merge Prometheus alerts with Loki alerts, standardize labels.

Recommended standardization:

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

PrometheusRule example:

    labels:
      severity: warning
      team: ai-platform
      source: prometheus
      environment: prod

Loki Rule example:

    labels:
      severity: warning
      team: ai-platform
      source: loki
      environment: prod

### 12.2 group_by Design

Recommended grouping dimensions:

group_by:
  - alertname
  - cluster
  - namespace
  - app

GPU scenarios can add:

  node
  gpu

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

### 12.3 Suppression Rule Design

If there are already core business metric alert:

  ServiceHigh5xxErrorRate

can suppress the same service's low-priority log noise alerts:

  AppErrorLogsTooMany

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

  Suppression is not deleting the issue.
  Log alerts can still enter reports as diagnostic context.
  Do not suppress critical log alerts excessively.

---

## ThirteenI don't know.Webhook Alert Gateway Design

### 13.1 Webhook Input

AlertManager sends JSON to Webhook gateway.

Gateway receives:

  receiver
  status
  alerts
  groupLabels
  commonLabels
  commonAnnotations
  externalURL

### 13.2 Gateway Processing Flow

  1. Receive AlertManager Webhook
  2. Validate Token / Signature
  3. Parse alerts
  4. Determine diagnosis type by alertname
  5. Generate diagnosis task by namespace / pod / node / app
  6. Call auto-diagnosis service
  7. Generate report
  8. Push notification
  9. Write audit record

### 13.3 alertname to Diagnosis Type Mapping

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

Webhook gateway must have:

  HTTPS
  Authentication
  IP Whitelist
  Rate Limiting
  Idempotency
  Audit
  Timeout Control
  Failure Retry
  Alert Storm Protection

---

## FourteenI don't know.Auto-Diagnosis Service Design

### 14.1 Diagnosis Service Input

Input fields:

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

### 14.2 Diagnosis Service Output

Output structure:

  diagnosis_id
  title
  summary
  impact
  current_status
  metric_evidence
  log_evidence
  event_evidence
  kubernetes_evidence
  gpu_evidence
  possible_causes
  recommended_actions
  commands
  dashboard_links
  runbook_links
  raw_attachments
  created_at

### 14.3 Data Sources for Query

Auto-diagnosis service needs to query:

  Kubernetes API
  Prometheus HTTP API
  Loki HTTP API
  Elasticsearch / OpenSearch API
  Grafana Dashboard Links
  Runbook Document Links

### 14.4 Permission Principles

Default read-only:

  get/list pods
  get pod logs
  get/list events
  get/list svc
  get/list endpoints
  get/list endpointslices
  get/list nodes
  get/list deployments
  get/list statefulsets
  get/list daemonsets

Default forbidden:

  delete
  patch
  update
  exec to business Pod
  Modify Secret
  Delete PVC
  Drain Node

---

## FifteenI don't know.Auto-Diagnosis RBAC

### 15.1 ServiceAccount /think

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

---

## Sixteen, Diagnosis Report Template

### 16.1 Report Structure

Recommend using Markdown format for the report.

Template:

    # Fault Diagnosis Report

    ## One, Alert Summary

    - Alert Name:
    - Alert Level:
    - Alert Source:
    - Cluster:
    - Environment:
    - Namespace:
    - App:
    - Pod:
    - Node:
    - Trigger Time:
    - Current Status:

    ## Two, Impact Scope

    - Affected Service:
    - Affected Pod:
    - Affected Node:
    - Affected GPU:
    - Is Production:
    - Is Core Business:

    ## Three, Metric Evidence

    - CPU:
    - Memory:
    - Restart:
    - Error Rate:
    - Latency:
    - GPU Memory:
    - GPU Util:
    - GPU Temperature:
    - XID:

    ## Four, Log Evidence

    - Recent ERROR:
    - Recent timeout:
    - Recent Exception:
    - Recent CUDA OOM:
    - Key Log Snippet:

    ## Five, Kubernetes Events

    - Pod Phase:
    - Container State:
    - Last State:
    - Events:
    - Deployment Replicas:
    - Service Endpoints:

    ## Six, Preliminary Judgment

    - Possible Cause 1:
    - Possible Cause 2:
    - Possible Cause 3:

    ## Seven, Recommended Actions

    - Temporary Containment:
    - Root Cause Investigation:
    - Long-term Optimization:

    ## Eight, Related Links

    - Grafana Dashboard:
    - Loki Explore:
    - Kibana Discover:
    - Runbook:
    - AlertManager:

    ## Nine, Original Commands

    - kubectl Command:
    - PromQL:
    - LogQL:
    - KQL:

---

## Seventeen, Pod Pending Diagnosis Logic

### 17.1 Trigger Alert

    PodPendingTooLong

### 17.2 Automatic Collection

Commands:

    kubectl get pod <pod> -n <namespace> -o wide

    kubectl describe pod <pod> -n <namespace>

    kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -50

Prometheus:

    kube_pod_status_phase{namespace="<namespace>", pod="<pod>", phase="Pending"}

### 17.3 Key Judgment

If Events contains:

    insufficient cpu

Judgment:

    Insufficient CPU resources.

If Events contains:

    insufficient memory

Judgment:

    Insufficient memory resources.

If Events contains:

    insufficient nvidia.com/gpu

Judgment: /think

GPU resources are insufficient or GPU is not registered.

If the following appears in Events:

    pod has unbound immediate PersistentVolumeClaims

Judgment:

    PVC is not bound.

If the following appears in Events:

    node(s) had untolerated taint

Judgment:

    taint/toleration does not match.

If the following appears in Events:

    didn't match Pod's node affinity/selector

Judgment:

    nodeSelector or affinity does not match.

### 17.4 Recommended Actions

Output:

    1. Check Pod Events.
    2. Check Node Allocatable.
    3. Check Namespace ResourceQuota.
    4. Check PVC status.
    5. Check GPU Device Plugin.
    6. Adjust resources, scheduling rules, or scale nodes based on root cause.

---

## Eighteen, Pod Restart Diagnosis Logic

### 18.1 Trigger Alert

    PodRestartTooOften

### 18.2 Automatic Collection

Commands:

    kubectl get pod <pod> -n <namespace> -o wide

    kubectl describe pod <pod> -n <namespace>

    kubectl logs <pod> -n <namespace> --tail=100

    kubectl logs <pod> -n <namespace> --previous --tail=100

Prometheus:

    increase(kube_pod_container_status_restarts_total{namespace="<namespace>", pod="<pod>"}[10m])

Loki:

    {namespace="<namespace>", pod="<pod>"} |~ "(?i)error|exception|panic|traceback"

### 18.3 Key Judgments

If Last State:

    OOMKilled

Judgment:

    Container memory limit exceeded.

If Events:

    Liveness probe failed

Judgment:

    Liveness probe failure or application response anomaly.

If logs:

    connection refused

Judgment:

    Dependency service connection failure.

If logs:

    permission denied

Judgment:

    Permission, mount, or security context issues.

If logs:

    config not found

Judgment:

    ConfigMap / Secret / configuration path issues.

### 18.4 Recommended Actions

Output:

    1. Check --previous logs.
    2. Check Last State Reason.
    3. Check Exit Code.
    4. Check probe configuration.
    5. Check ConfigMap / Secret.
    6. Determine if version rollback is needed.
    7. Do not recommend blind restarts.

---

## Nineteen, Pod OOMKilled Diagnosis Logic

### 19.1 Trigger Alert

    PodOOMKilled

### 19.2 Automatic Collection

Prometheus:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

Pod memory:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="<namespace>", pod="<pod>", container!="", image!=""}
    )

Pod limit:

    kube_pod_container_resource_limits{namespace="<namespace>", pod="<pod>", resource="memory"}

Commands:

    kubectl describe pod <pod> -n <namespace>
    kubectl logs <pod> -n <namespace> --previous --tail=100

### 19.3 Key Judgments

Possible causes:

    memory limit is too small
    application memory leak
    sudden large request
    cache too large
    JVM heap configuration is unreasonable
    Python data loading is too large
    batch processing task input is too large

### 19.4 Recommended Actions

Output:

    1. Check memory usage trend.
    2. Check if limit is too low.
    3. Check application logs for large requests or batch tasks.
    4. Check for recent version changes.
    5. Temporarily increasing limit should be cautious.
    6. Long-term, fix memory leak or optimize memory model.

---

## Twenty, Service 5xx Diagnosis Logic

### 20.1 Trigger Alert

    ServiceHigh5xxErrorRate

### 20.2 Automatic Collection

Service:

    kubectl describe svc <service> -n <namespace>

Endpoints:

    kubectl get endpoints <service> -n <namespace>

EndpointSlice:

    kubectl get endpointslice -n <namespace> | grep <service>

Pod:

    kubectl get pod -n <namespace> -o wide

Prometheus:

    5xx error rate
    QPS
    P95 / P99 latency

Loki:

    {namespace="<namespace>", app="<app>"} |~ "(?i)error|exception|timeout|connection refused"

### 20.3 Key Judgments

If Endpoints is empty:

    Service selector or Pod Ready is abnormal.

If Endpoints are normal but 5xx is high:

    Application internal error or downstream dependency anomaly.

If logs haveMass timeout:

    Downstream dependency is slow or network anomaly.

If logs haveMass database connection failed:

    Database connection anomaly.

If latency increases simultaneously with error rate increase:

It might be due to slow dependent services, insufficient resources, or exhausted thread pools.

### 20.4 Report Recommended Actions

Output:

    1. Check Service selector.
    2. Check Endpoints.
    3. Check backend Pod Ready.
    4. Check application error logs.
    5. Check recent deployments.
    6. Inspect downstream database, Redis, and external interfaces.
    7. Roll back version if necessary.

---

## Twenty-one, CUDA OOM Diagnosis Logic

### 21.1 Trigger Alert

    CUDALogOOMDetected
    GPUMemoryUsageHigh
    PodRestartTooOften

### 21.2 Automatic Collection

Pod:

    kubectl get pod <pod> -n <namespace> -o wide
    kubectl describe pod <pod> -n <namespace>
    kubectl logs <pod> -n <namespace> --previous --tail=100

GPU metrics:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE
    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_GPU_TEMP

Logs:

    {namespace="<namespace>", pod="<pod>"} |= "CUDA out of memory"

    {namespace="<namespace>", pod="<pod>"} |~ "(?i)batch|model|worker|memory"

Node-side manual commands:

    nvidia-smi
    nvidia-smi -q

### 21.3 Key Judgment

Possible causes:

    Batch size too large
    Model too large
    Too many workers
    High concurrency
    Multiple models loaded repeatedly
    Memory leak
    GPU sharing causing memory contention
    MIG instance memory insufficient

### 21.4 Report Recommended Actions

Output:

    1. Reduce batch size.
    2. Reduce concurrency.
    3. Decrease number of workers.
    4. Check for repeated model loading.
    5. Check for continuous memory growth.
    6. Use FP16 / BF16 / quantization.
    7. Migrate to larger memory GPU if necessary.
    8. Not recommended to automatically delete Pod.

---

## Twenty-two, GPU XID Diagnosis Logic

### 22.1 Trigger Alert

    GPUXIDError

### 22.2 Automatic Collection

Prometheus:

    DCGM_FI_DEV_XID_ERRORS
    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_POWER_USAGE
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_GPU_UTIL

Kubernetes:

    kubectl get pod -A -o wide | grep <gpu-node>
    kubectl describe node <gpu-node>

Node-side manual commands:

    dmesg | grep -i xid
    journalctl -k | grep -i xid
    journalctl -k | grep -i nvrm
    nvidia-smi
    nvidia-smi -q

### 22.3 Key Judgment

If XID occurs occasionally:

    Record and observe.

If occurs multiple times in short time:

    Isolate node and perform deep investigation.

If accompanied by high temperature:

    Prioritize checking cooling, data center, fans, and power consumption.

If accompanied by fallen off the bus:

    High risk, prioritize cordon node and contact hardware vendor.

If aligned with a specific Pod timepoint:

    Investigate application CUDA, model, and driver compatibility.

### 22.4 Report Recommended Actions

Output:

    1. Check dmesg and journalctl.
    2. Check GPU temperature and power consumption.
    3. Check GPU Pod running at the time.
    4. Determine if it occurs repeatedly.
    5. Recommend cordon node if it repeats.
    6. Perform hardware inspection and vendor support if necessary.

---

## Twenty-three, Automatic Diagnosis Script Design

### 23.1 Bash Version Approach

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
    POD="${POD:-}"
    NODE="${NODE:-}"
    SERVICE="${SERVICE:-}"
    REPORT="/tmp/diagnosis-report.md"

    echo "# Fault Diagnosis Report" > "${REPORT}"
    echo "" >> "${REPORT}"

    echo "## One, Alert Summary" >> "${REPORT}"
    echo "- Alert Name: ${ALERT_NAME}" >> "${REPORT}"
    echo "- Namespace: ${NAMESPACE}" >> "${REPORT}"
    echo "- Pod: ${POD}" >> "${REPORT}"
    echo "- Node: ${NODE}" >> "${REPORT}"
    echo "- Service: ${SERVICE}" >> "${REPORT}"
    echo "" >> "${REPORT}"

    if [ -n "${POD}" ]; then
      echo "## Two, Pod Status" >> "${REPORT}"
      kubectl get pod "${POD}" -n "${NAMESPACE}" -o wide >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"

      echo "## Three, Pod Describe" >> "${REPORT}"
      kubectl describe pod "${POD}" -n "${NAMESPACE}" >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"

echo "## Four. Pod Current Logs" >> "${REPORT}"
      kubectl logs "${POD}" -n "${NAMESPACE}" --tail=100 >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"

      echo "## Five. Pod Previous Logs" >> "${REPORT}"
      kubectl logs "${POD}" -n "${NAMESPACE}" --previous --tail=100 >> "${REPORT}" 2>&1 || true
      echo "" >> "${REPORT}"
    fi

    echo "## Six. Recent Events" >> "${REPORT}"
    kubectl get events -n "${NAMESPACE}" --sort-by=.lastTimestamp | tail -50 >> "${REPORT}" 2>&1 || true

    cat "${REPORT}"

### 23.2 Python Version Approach

Python is more suitable for production expansion.

Module Design:

    alert_parser.py
        Parses AlertManager Webhook.

    k8s_client.py
        Queries Pod, Service, Endpoints, Events.

    prometheus_client.py
        Queries PromQL.

    loki_client.py
        Queries LogQL.

    elasticsearch_client.py
        Queries KQL / Query DSL.

    report_renderer.py
        Generates Markdown / HTML report.

    notifier.py
        Pushes Enterprise WeChat / DingTalk / Feishu / Email.

    main.py
        Main entry point.

### 23.3 Report Generation Flow

    1. Parse Alert
    2. Determine Diagnosis Type
    3. Query Kubernetes Status
    4. Query Prometheus Metrics
    5. Query Loki / ELK Logs
    6. Generate Preliminary Judgment
    7. Generate Markdown Report
    8. Upload Report
    9. Push Notification

---

## Twenty-Four. Diagnosis Job Example

### 24.1 Job Purpose

Used to run one-time diagnostic tasks within the cluster.

Suitable for:

    Collecting kubectl output
    Pulling logs
    Generating reports
    Pushing results

### 24.2 Job YAML Example

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

### 24.3 Notes

Do not use:

    latest

Production should use fixed image version:

    registry.example.com/sre/diagnosis-toolkit:v1.0.0

Recommended tools inside the image:

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

## Twenty-five, Report Push Design

### 25.1 Push Content

Alert notifications should not send the full long report to the group chat.

Recommended group message content:

    Alert Summary
    Impact Scope
    Preliminary Conclusion
    Top 3 Evidence
    Recommended Actions
    Report Link
    Dashboard Link
    Runbook Link

Full report saved to:

    Object Storage
    Wiki
    Ticketing System
    Internal Diagnosis Platform

### 25.2 Message Example

    [FIRING][warning] CUDALogOOMDetected

    Cluster: prod-k8s
    Namespace: ai-prod
    Pod: ai-inference-demo-xxx
    Preliminary Judgment: Detected CUDA OOM, GPU memory usage approaching 98%.

    Key Evidence:
    1. Loki shows CUDA out of memory.
    2. DCGM_FI_DEV_FB_USED remains high.
    3. Pod restarted 2 times in the last 10 minutes.

    Recommended Actions:
    - Check batch size and concurrency.
    - Check if model is reloaded repeatedly.
    - Migrate to larger GPU if necessary.

    Diagnosis Report:
    https://diagnosis.example.com/report/xxx

    Runbook:
    https://wiki.example.com/runbook/cuda-oom

---

## Twenty-six, Grafana Dashboard Design

### 26.1 Comprehensive Overview Dashboard

Panels:

    Current Alert Count
    Critical Alert Count
    Warning Alert Count
    Pod Pending Count
    Pod Restart Top
    OOMKilled Pod Count
    Service 5xx Top
    P95 Delay Top
    GPU Memory Top
    GPU Temperature Top
    GPU XID Count
    ERROR Log Top App
    CUDA OOM Log Trend
    Automated Diagnosis Success Rate

### 26.2 Pod Anomaly Dashboard

Variables:

    namespace
    pod
    node
    app

Panels:

    Pod Phase
    Pod Ready
    Pod Restart
    Pod CPU
    Pod Memory
    OOMKilled
    Recent Events
    Recent Error Logs

### 26.3 GPU Anomaly Dashboard

Variables:

    node
    gpu
    namespace
    pod

Panels:

    GPU Utilization
    GPU Memory Usage
    GPU Temperature
    GPU Power Consumption
    GPU XID
    GPU Pod List
    CUDA OOM Logs
    Model Load Failure Logs

### 26.4 Automated Diagnosis Dashboard

Panels:

    Diagnosis Task Count
    Diagnosis Success Rate
    Diagnosis Failure Rate
    Average Diagnosis Duration
    Recent Diagnosis Reports
    Top Alert Types
    Top Anomalous Namespaces
    Top Anomalous Apps
    Automated Throttling Count

---

## Twenty-seven, Alert and Report Governance

### 27.1 Alerts Must Have Runbook

Every important alert must include:

    runbook_url

Example:

    annotations:
      runbook_url: "https://wiki.example.com/runbook/pod-pending"

### 27.2 Alerts Must Have Dashboard

Every important alert should include:

    dashboard_url

Example:

    annotations:
      dashboard_url: "https://grafana.example.com/d/pod-detail"

### 27.3 Reports Must Have Diagnosis ID

Each diagnosis generates a unique ID:

    diagnosis_id: diag-20260430-0001

Used for:

    Historical Query
    Ticketing Association
    Postmortem Reference
    Audit Tracking

### 27.4 Reports Must Be Traceable

Record:

    Original Alert Payload
    Query Time
    Query Conditions
    Executed Commands
    Execution Results
    Diagnosis Version
    Diagnosis Service Version

---

## Twenty-eight, Automated Response Security Boundaries

### 28.1 Default Read-Only

Automated diagnosis service defaults to read-only access:

    Pod
    Service
    Endpoints
    Events
    Deployment
    Node
    Logs
    Metrics

### 28.2 Actions Requiring Manual Confirmation

Include:

    rollout restart
    rollback
    Scaling Up
    cordon
    silence
    Delete Failed Job
    Restart Non-Critical Pod

### 28.3 Prohibited Automated Actions

Include:

    Delete PVC
    Delete Database Data
    Drain Production Node
    Delete Elasticsearch Index
    Modify Secret
    Modify NetworkPolicy
    Restart Database Master
    Switch Master-Slave
    Batch Delete Pod

### 28.4 Must Have Audit

Record:

    Who Triggered
    What Triggered
    Target Object
    What Was Done
    Result
    Whether Manually Confirmed
    Whether Failed
    Whether Rolled Back

---

## Twenty-nine, Complete Case Study One: Pod Restart + ERROR Logs

### 29.1 Phenomenon

Alerts:

    PodRestartTooOften
    AppErrorLogsTooMany

### 29.2 Automated Diagnosis Report Summary

The report shows: /think

# 30. Complete Case Two: GPU Memory High + CUDA OOM

### 30.1 Phenomenon

Alarms:
- GPUMemoryUsageHigh
- CUDALogOOMDetected
- PodRestartTooOften

### 30.2 Automatic Diagnostic Report Summary

The report shows:
- ai-prod/ai-inference-demo-xxx has encountered CUDA out of memory.
- GPU memory usage is at 98%.
- The Pod has restarted 2 times in the last 10 minutes.
- The logs show batch_size=64.
- GPU temperature is normal, no XID detected.

### 30.3 Preliminary Judgment

Possible causes:
- Batch size is too large.
- Inference concurrency is too high.
- GPU memory capacity is insufficient.
- Not a GPU hardware issue.

### 30.4 Recommended Actions

1. Reduce batch size.
2. Reduce concurrency.
3. Check if the model is being reloaded repeatedly.
4. Check worker count.
5. Evaluate the need for a larger GPU with more memory.
6. Do not recommend automatic restarts.

---

# 31. Complete Case Three: Service 5xx + Timeout

### 31.1 Phenomenon

Alarms:
- ServiceHigh5xxErrorRate
- AppTimeoutLogsTooMany

### 31.2 Automatic Diagnostic Report Summary

The report shows:
- app-prod/order-api has a 18% 5xx error rate.
- P95 latency has risen from 150ms to 2.5s.
- There are 120 timeout logs in Loki.
- Endpoints are normal.
- No obvious Pod restarts.
- There is a deployment record in the last 30 minutes.

### 31.3 Preliminary Judgment

Possible causes:
- The new version introduced downstream call timeouts.
- Downstream service is slow.
- Connection pool is insufficient.
- Network or DNS anomalies are unlikely but need confirmation.

### 31.4 Recommended Actions

1. Check the deployment changes.
2. Check downstream service monitoring.
3. Check the application connection pool.
4. Roll back the version if necessary.
5. Add downstream dependency latency metrics.

---

# 32. Complete Case Four: GPU XID

### 32.1 Phenomenon

Alarms:
- GPUXIDError

### 32.2 Automatic Diagnostic Report Summary

The report shows:
- GPU 0 on k8s-gpu-node01 has encountered XID.
- The job ai-prod/train-job-xxx was running at the time.
- GPU temperature is 72°C.
- GPU power consumption is normal.
- No fallen off the bus metrics yet.
- Manual login to the node is required to check dmesg.

### 32.3 Preliminary Judgment

Possible causes:
- CUDA application anomaly.
- Driver or hardware intermittent error.
- Needs to be determined by combining dmesg and XID number.

### 32.4 Recommended Actions

1. Check dmesg | grep -i xid.
2. Check journalctl -k | grep -i nvrm.
3. Check training task logs.
4. Cordon the node if it repeats.
5. Contact hardware vendor if fallen off the bus occurs.

---

# 33. Acceptance Checklist

### 33.1 Metric Acceptance

- [ ] kube-state-metrics is normal
- [ ] kubelet / cAdvisor is normal
- [ ] node-exporter is normal
- [ ] dcgm-exporter is normal
- [ ] app /metrics is normal
- [ ] Pod metrics are accessible
- [ ] Service metrics are accessible
- [ ] GPU metrics are accessible

### 33.2 Log Acceptance

- [ ] Loki can access Pod logs
- [ ] ELK / OpenSearch can access application logs
- [ ] Logs include namespace
- [ ] Logs include pod
- [ ] Logs include app
- [ ] Logs include level
- [ ] Can query ERROR
- [ ] Can query CUDA OOM
- [ ] Can query timeout
- [ ] Can query database connection failed

### 33.3 Alarm Acceptance

- [ ] PrometheusRule is loaded
- [ ] Loki Ruler is loaded
- [ ] ELK / OpenSearch alarms can be triggered
- [ ] AlertManager can receive alarms
- [ ] Alarms can be grouped
- [ ] Alarms can be routed to the correct team
- [ ] Alarms can be silenced
- [ ] Alarms have Runbook
- [ ] Alarms have Dashboard

### 33.4 Automatic Diagnosis Acceptance

- [ ] Webhook gateway can receive alarms
- [ ] Can parse AlertManager payload
- [ ] Can query Kubernetes API
- [ ] Can query Prometheus API
- [ ] Can query Loki API
- [ ] Can query ELK / OpenSearch
- [ ] Can generate Markdown report
- [ ] Can push report link
- [ ] Diagnosis service has read-only permissions
- [ ] Diagnosis failure has error logs

### 33.5 Security Acceptance

- [ ] Webhook has authentication
- [ ] Automation service has minimal permissions
- [ ] Prohibit high-risk actions from being executed automatically
- [ ] All actions have audit logs
- [ ] Has rate limiting strategy
- [ ] Has failure retry
- [ ] Secrets are not committed to Git in plain text
- [ ] Reports do not leak sensitive information

---

# 34. Common Misconceptions

### 34.1 Misconception One: Having Prometheus Means No Need for Logs

Error.

Prometheus can only tell you that metrics are abnormal; logs are needed to provide specific error context.

### 34.2 Misconception Two: You don't need metrics if you have Loki / ELK

Incorrect.

Logs are suitable for root cause analysis; metrics are suitable for trend analysis, alerts, capacity planning, and SLO evaluation.

### 34.3 Misconception Three: Automatic diagnosis equals automatic repair

Incorrect.

Automatic diagnosis is typically read-only by default; automatic repair must be carefully graded.

### 34.4 Misconception Four: The longer the report, the better

Incorrect.

Reports should highlight key evidence, initial judgment, and recommended actions.

Do not blindly pile all command outputs.

### 34.5 Misconception Five: Restarting the Pod directly fixes GPU OOM

Incorrect.

CUDA OOM is usually related to batch size, concurrency, model size, worker count, and GPU memory leaks.

Restarting can only temporarily release memory, but cannot resolve the root cause.

### 34.6 Misconception Six: XID always means the GPU is broken

Incorrect.

XID may originate from the application, driver, hardware, PCIe, temperature, power, and other directions.

It must be judged in combination with XID numbering, logs, temperature, power, repetition frequency, and hardware status.

---

## Thirty-Five, Production Implementation Recommendations

### 35.1 Phase One: Connect alerts and reports

Goals:

    Prometheus alerts
    Loki / ELK alerts
    AlertManager notifications
    Webhook reception
    Automatically generate basic reports

### 35.2 Phase Two: Improve diagnostic logic

Add:

    Pod Pending diagnosis
    Pod Restart diagnosis
    Service 5xx diagnosis
    CUDA OOM diagnosis
    GPU XID diagnosis
    Log anomaly diagnosis

### 35.3 Phase Three: Integrate ChatOps

Implement:

    View reports
    View logs
    View Dashboard
    Silent alerts
    Create tickets
    Execute low-risk actions after manual confirmation

### 35.4 Phase Four: Limited automatic repair

Only for low-risk scenarios:

    Non-production Pod restart
    Test environment failed Job cleanup
    Non-core service manual confirmation for scaling
    Automatically create resource governance tickets

Automatic repair is not recommended for production core systems.

---

## Thirty-Six, Summary

This article is a comprehensive case study on Prometheus, Grafana, AlertManager, Loki, ELK, GPU monitoring, and automated response.

A complete production observability loop should be:

    Metrics detect anomalies
      ↓
    Logs supplement causes
      ↓
    Alerts converge uniformly
      ↓
    Automatic diagnosis collects evidence
      ↓
    Generate structured reports
      ↓
    Push to the correct team
      ↓
    Process according to Runbook
      ↓
    Review and optimize rules

Division of labor among systems:

    Prometheus:
        Metrics collection, PromQL, metric alerts.

    DCGM Exporter:
        GPU utilization, memory, temperature, power, XID.

    Loki:
        Kubernetes log aggregation, LogQL, log alerts.

    ELK / OpenSearch:
        Full-text search, field queries, audit, and complex log analysis.

    AlertManager:
        Alert grouping, deduplication, suppression, silence, routing.

    Webhook gateway:
        Notification adaptation, alert storage, trigger diagnosis.

    Automatic diagnosis service:
        Query K8S, Prometheus, Loki, ELK, generate reports.

    Grafana:
        Dashboard, metric and log linkage, troubleshooting entry.

The true goal of a production-grade system is not "having charts, logs, and alerts", but:

    Anomalies can be detected
    Alerts can be converged
    Context can be automatically collected
    Reports can be understood
    Processing has Runbook
    Automation has boundaries
    Review can be accumulated
    Rules can be continuously optimized

For Kubernetes GPU / AI infrastructure operations, this capability can cover:

    Pod scheduling anomalies
    Application runtime anomalies
    Service access anomalies
    GPU resource anomalies
    CUDA OOM
    Model loading failure
    Log error burst
    Alert storm
    Monitoring chain anomalies

Finally, forming a production-oriented SRE observability and automation response capability.

---

## Reference Documents

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes Monitoring, Logging, and Debugging:
  https://kubernetes.io/docs/tasks/debug/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- Prometheus AlertManager:
  https://prometheus.io/docs/alerting/latest/alertmanager/

- Prometheus Alerting Practices:
  https://prometheus.io/docs/practices/alerting/

- Grafana Loki Alerting and Recording Rules:
  https://grafana.com/docs/loki/latest/alert/

- Grafana Loki LogQL:
  https://grafana.com/docs/loki/latest/query/

- NVIDIA DCGM Exporter:
  https://docs.nvidia.com/datacenter/dcgm/latest/gpu-telemetry/dcgm-exporter.html

- NVIDIA DCGM Exporter GitHub:
  https://github.com/NVIDIA/dcgm-exporter

- Grafana Documentation:
  https://grafana.com/docs/grafana/latest/

- Grafana Alerting:
  https://grafana.com/docs/grafana/latest/alerting/

- Kubernetes RBAC:
  https://kubernetes.io/docs/reference/access-authn-authz/rbac/

- Kubernetes Jobs:
  https://kubernetes.io/docs/concepts/workloads/controllers/job/

- Elasticsearch Alerting / Kibana Alerting:
  https://www.elastic.co/docs/explore-analyze/alerts-cases

- OpenSearch Alerting:
  https://docs.opensearch.org/latest/observing-your-data/alerting/