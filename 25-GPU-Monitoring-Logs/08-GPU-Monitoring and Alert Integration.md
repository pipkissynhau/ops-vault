# 08-GPU Monitoring and Alert Integration

## Document Overview

This document is used to organize GPU monitoring, metric collection, Prometheus integration, Grafana visualization, AlertManager alerts, GPU troubleshooting, and observability baselines for Kubernetes GPU clusters.

This document focuses on answering the following questions:

- Why can't GPU be monitored solely by `nvidia-smi` manual inspection;
- What metrics should be collected for GPU monitoring;
- What are DCGM and DCGM Exporter;
- How to integrate DCGM Exporter with Kubernetes;
- How to use DCGM Exporter in GPU Operator;
- How Prometheus collects GPU metrics;
- How to choose ServiceMonitor and scrape_configs;
- Which panels should be viewed in the Grafana GPU Dashboard;
- How to alert on GPU temperature, memory, utilization, power consumption, XID, and ECC;
- Is low GPU utilization necessarily a waste;
- Is high memory usage necessarily abnormal;
- How to troubleshoot GPU Pod anomalies by combining metrics, logs, and events;
- How to establish monitoring and alert baselines in production environments.

This document is suitable for study after completing the following chapters:

- 03-NVIDIA Driver Installation and Verification
- 04-CUDA Installation and Testing
- 05-K8S-GPU Resource Concepts and Scheduling Principles
- 06-NVIDIA Device Plugin and Operator Installation
- 07-GPU Pod Deployment and Scheduling Practical Guide

---

## Tags

#Kubernetes #GPU #NVIDIA #DCGM #DCGMExporter #Prometheus #Grafana #AlertManager #SRE #Observation #TransportBarriers

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/03-GPU Monitoring and Troubleshooting/08-GPU-Monitoring and Alert Integration.md

---

## One, Why GPU Needs Specialized Monitoring

Traditional Kubernetes monitoring typically focuses on:

- Node CPU;
- Node Memory;
- Node Disk;
- Node Network;
- Pod CPU;
- Pod Memory;
- Pod Restart;
- Pod Pending;
- Service Availability;
- Application QPS / Latency / Error Rate.

But GPU nodes also need to pay special attention to GPU-specific metrics.

Because GPU is a high-cost, high-power consumption, high-density, high-value computing resource.

Without GPU monitoring, operations can only know:

    Whether Pod is Running
    Whether Node is Ready
    Whether CPU is high
    Whether memory is high

But don't know:

    Whether GPU is actually being used
    Whether GPU is idle but occupied
    Whether memory is nearly full
    Whether GPU is overheating
    Whether GPU is throttled
    Whether GPU is power-constrained
    Whether there are XID errors
    Whether there are ECC errors
    Whether there is GPU card drop
    Whether Device Plugin is abnormal
    Whether GPU Pod is Pending
    Whether a team is long-term occupying GPU
    Whether GPU resource utilization is worth expanding

The core goal of GPU monitoring is not "drawing a few charts", but forming:

    Metric Collection
      ↓
    Dashboard Display
      ↓
    Alert Trigger
      ↓
    Log Correlation
      ↓
    Event Troubleshooting
      ↓
    Fault Localization
      ↓
    Capacity Planning
      ↓
    Cost Governance

---

## Two, GPU Observability Architecture

Production environment GPU observability can be divided into five layers:

    1. Node Hardware Layer
    2. NVIDIA Driver Layer
    3. Kubernetes Scheduling Layer
    4. Container and Application Layer
    5. Monitoring, Logging, and Alerting Layer

Overall flow:

    GPU Hardware
      ↓
    NVIDIA Driver / DCGM
      ↓
    DCGM Exporter
      ↓
    Prometheus
      ↓
    Grafana Dashboard
      ↓
    AlertManager
      ↓
    Webhook / Email / IM / OnCall

Kubernetes perspective:

    GPU Pod
      ↓
    Device Plugin / GPU Operator
      ↓
    kubelet / Node Status
      ↓
    kube-state-metrics
      ↓
    Prometheus
      ↓
    Grafana / AlertManager

Logging perspective:

    GPU Pod Logs
      ↓
    Fluent Bit / Promtail
      ↓
    Loki / Elasticsearch
      ↓
    Grafana / Kibana
      ↓
    Joint Troubleshooting with Metrics and Logs

---

## Three, DCGM and DCGM Exporter

### 3.1 What is DCGM

DCGM, full name Data Center GPU Manager.

It is a collection of management and monitoring capabilities for data center GPUs from NVIDIA.

DCGM can provide:

- GPU utilization;
- GPU memory usage;
- GPU temperature;
- GPU power consumption;
- GPU health status;
- ECC errors;
- XID errors;
- PCIe-related metrics;
- NVLink-related metrics;
- MIG metrics;
- Profiling metrics.

From an operations perspective, DCGM is the foundational capability for GPU monitoring.

### 3.2 What is DCGM Exporter

DCGM Exporter is an exporter that exposes DCGM metrics in Prometheus-compatible format.

It typically exposes an HTTP interface:

    /metrics

Default port is commonly:

    9400

Prometheus collects metrics via scrape:

    http://<dcgm-exporter>:9400/metrics

### 3.3 DCGM Exporter's Position in Kubernetes

DCGM Exporter is usually deployed as a DaemonSet on GPU nodes.

Each GPU node runs a DCGM Exporter Pod.

It is responsible for collecting GPU metrics on the node and exposing them to Prometheus.

Simplified topology: /think

+----------------------+
    | GPU Node             |
    |                      |
    | NVIDIA Driver        |
    | DCGM                 |
    | DCGM Exporter :9400  |
    | GPU Pods             |
    +----------+-----------+
               |
               | /metrics
               v
    +----------------------+
    | Prometheus           |
    +----------+-----------+
               |
               v
    +----------------------+
    | Grafana              |
    +----------------------+

---

## IV. GPU Monitoring Metric Categories

GPU monitoring cannot focus solely on GPU utilization.

It is recommended to classify metrics by the following dimensions.

### 4.1 Resource Utilization

Used to determine whether GPU is in use, idle, or experiencing resource waste.

Common metrics:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_MEM_COPY_UTIL
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE

Attention points:

- Whether GPU utilization is consistently too low over time;
- Whether GPU is occupied by Pod but not performing computation;
- Whether VRAM is consistently occupied;
- Whether VRAM is approaching full capacity;
- Whether training or inference tasks are genuinely utilizing GPU.

### 4.2 Temperature and Cooling

Used to detect cooling issues and frequency reduction risks.

Common metrics:

    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_MEMORY_TEMP

Attention points:

- Whether GPU temperature is persistently too high;
- Whether VRAM temperature is excessively high;
- Whether data center temperature is abnormal;
- Whether fan strategy is appropriate;
- Whether nodes may experience frequency reduction;
- Whether cooling or airflow issues exist.

### 4.3 Power and Performance Limitation

Used to determine whether GPU is constrained by power limits.

Common metrics:

    DCGM_FI_DEV_POWER_USAGE
    DCGM_FI_DEV_POWER_VIOLATION
    DCGM_FI_DEV_THERMAL_VIOLATION

Attention points:

- Whether GPU is approaching power limit;
- Whether performance degradation is caused by power constraints;
- Whether frequency reduction is caused by thermal constraints;
- Whether power capacity is insufficient;
- Whether nodes are suitable for long-term full-load training.

### 4.4 Error and Health

Used to detect hardware, driver, VRAM, and runtime anomalies.

Common metrics:

    DCGM_FI_DEV_XID_ERRORS
    DCGM_FI_DEV_ECC_DBE_VOL_TOTAL
    DCGM_FI_DEV_ECC_SBE_VOL_TOTAL
    DCGM_FI_DEV_RETIRED_SBE
    DCGM_FI_DEV_RETIRED_DBE

Attention points:

- Whether XID errors occur;
- Whether ECC errors exist;
- Whether VRAM pages are retired;
- Whether GPU hardware failure may exist;
- Whether node isolation or hardware replacement is needed.

### 4.5 Topology and Communication

Used for multi-GPU training and high-performance computing scenarios.

Potential attention points:

- PCIe throughput;
- NVLink bandwidth;
- GPU and NIC topology;
- Multi-GPU communication status;
- NCCL communication anomalies.

These metrics are related to GPU model, DCGM configuration, and exporter metric sets. Not all environments will collect them by default.

### 4.6 Kubernetes Scheduling

GPU metrics also need to be combined with Kubernetes resource status:

- Node Ready;
- Node Allocatable GPU;
- GPU Pod Pending;
- GPU Pod Restart;
- Device Plugin Pod status;
- DCGM Exporter Pod status;
- GPU Operator Pod status;
- Namespace GPU Quota;
- GPU Pod node;
- GPU Pod team and application.

These metrics typically come from:

    kube-state-metrics
    kubelet / cAdvisor
    Prometheus Kubernetes SD
    DCGM Exporter
    GPU Operator components

---

## V. Common DCGM Metric Explanations

The following metric names are examples from DCGM Exporter, but actual existence depends on GPU model, DCGM Exporter version, metrics configuration, and GPU Operator configuration.

### 5.1 GPU Utilization

Metric:

    DCGM_FI_DEV_GPU_UTIL

Meaning:

    GPU compute utilization, typically as a percentage.

PromQL example:

    DCGM_FI_DEV_GPU_UTIL

View by Pod:

    DCGM_FI_DEV_GPU_UTIL{pod!="", namespace!=""}

Production interpretation:

- High utilization is not necessarily abnormal; long-term 90%+ for training tasks may be normal;
- Low utilization is not necessarily abnormal; low-traffic inference may be normal;
- Long-term 0% with Pod occupying VRAM requires attention to potential idle occupation;
- Fluctuating utilization may indicate data loading, CPU, network, or storage bottlenecks.

### 5.2 VRAM Used

Metric:

    DCGM_FI_DEV_FB_USED

Meaning:

    GPU Frame Buffer used VRAM, typically in MiB.

Example:

    DCGM_FI_DEV_FB_USED

### 5.3 VRAM Free

Metric:

    DCGM_FI_DEV_FB_FREE

Meaning:

    GPU remaining VRAM, typically in MiB.

### 5.4 VRAM Utilization

Can be calculated via PromQL:

    DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) * 100

Pod-based calculation example:

    DCGM_FI_DEV_FB_USED{pod!="", namespace!=""}
    /
    (DCGM_FI_DEV_FB_USED{pod!="", namespace!=""} + DCGM_FI_DEV_FB_FREE{pod!="", namespace!=""})
    * 100

Note: /think

If label dimensions are inconsistent, use on() or ignoring() to adjust PromQL matching.  
Actual queries should be adjusted according to the label structure in the cluster.

### 5.5 GPU Temperature

Metrics:

    DCGM_FI_DEV_GPU_TEMP

Meaning:

    Current GPU temperature, typically in Celsius.

Example:

    DCGM_FI_DEV_GPU_TEMP

### 5.6 Memory Temperature

Metrics:

    DCGM_FI_DEV_MEMORY_TEMP

Meaning:

    Memory temperature.  
Not all GPUs expose this metric.

### 5.7 Power Consumption

Metrics:

    DCGM_FI_DEV_POWER_USAGE

Meaning:

    Current GPU power consumption.

Example:

    DCGM_FI_DEV_POWER_USAGE

### 5.8 XID Errors

Metrics:

    DCGM_FI_DEV_XID_ERRORS

Meaning:

    Latest XID error value.  
XID errors typically require analysis in conjunction with node kernel logs.

Node log troubleshooting:

    dmesg | grep -i xid  
    journalctl -k | grep -i xid

### 5.9 ECC Errors

Common metrics:

    DCGM_FI_DEV_ECC_SBE_VOL_TOTAL  
    DCGM_FI_DEV_ECC_DBE_VOL_TOTAL

Meaning:

    Cumulative ECC single-bit / double-bit error values.

Production recommendations:

    Do not ignore ECC errors lightly.  
    Repeated growth may indicate hardware health risks for memory.

---

## Six. Deployment Method Selection

GPU monitoring deployment commonly uses two methods:

    1. Using DCGM Exporter in GPU Operator  
    2. Installing DCGM Exporter separately

### 6.1 GPU Operator Integration

If the cluster already uses GPU Operator, it's typically preferred to manage DCGM Exporter via Operator.

Advantages:

- Unified components;  
- Consistent management with Device Plugin, Driver, and Toolkit;  
- More suitable for production;  
- Can unify values.yaml;  
- Compatible with GPU Feature Discovery;  
- Can integrate with MIG metrics.

Check:

    kubectl get pods -n gpu-operator | grep -i dcgm  
    kubectl get svc -n gpu-operator | grep -i dcgm

### 6.2 Separate DCGM Exporter Deployment

Suitable for:

- No GPU Operator usage;  
- Only Device Plugin installed;  
- Small-scale experiments;  
- Want to manage monitoring components separately;  
- Already has a custom monitoring stack.

Can be deployed via Helm Chart.

### 6.3 Production Recommendations

If already using GPU Operator:

    Prioritize managing DCGM Exporter via Operator.

If only using Device Plugin:

    Can install DCGM Exporter separately.

Regardless of the method, ensure:

- Fixed image version;  
- Clear port exposure;  
- Prometheus can scrape metrics;  
- Grafana Dashboard is available;  
- AlertManager alert configuration is complete;  
- Metrics labels can associate with namespace/pod/node;  
- Alerts have clear handling procedures.

---

## Seven. Using GPU Operator to Deploy DCGM Exporter

### 7.1 Check GPU Operator Components

    kubectl get pods -n gpu-operator -o wide  
    kubectl get ds -n gpu-operator  
    kubectl get svc -n gpu-operator

Search for:

    dcgm  
    dcgm-exporter

### 7.2 Check DCGM Exporter Pod

    kubectl get pods -n gpu-operator | grep -i dcgm

Check logs:

    kubectl logs <dcgm-exporter-pod> -n gpu-operator

### 7.3 Check Service

    kubectl get svc -n gpu-operator | grep -i dcgm

Check service details:

    kubectl describe svc <dcgm-exporter-service> -n gpu-operator

### 7.4 Verify Metrics

Port forwarding:

    kubectl port-forward -n gpu-operator <dcgm-exporter-pod> 9400:9400

Access:

    curl http://127.0.0.1:9400/metrics

If you see similar metrics like:

    DCGM_FI_DEV_GPU_UTIL  
    DCGM_FI_DEV_FB_USED  
    DCGM_FI_DEV_GPU_TEMP

It indicates metrics are exposed.

### 7.5 values.yaml Management Recommendations

In production environments, avoid stacking `--set` in command lines.

Recommendations:

    helm get values gpu-operator -n gpu-operator > values-gpu-operator.yaml

Or:

    helm show values nvidia/gpu-operator --version <CHART_VERSION> > values-gpu-operator.yaml

Manage via values.yaml:

- Whether dcgmExporter is enabled;  
- Image repository;  
- Image tag;  
- ServiceMonitor;  
- Runtime;  
- Driver;  
- Toolkit;  
- MIG;  
- nodeSelector;  
- tolerations.

---

## Eight. Installing DCGM Exporter Separately

### 8.1 Add Helm Repository

    helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts  
    helm repo update

Check versions:

    helm search repo gpu-helm-charts/dcgm-exporter --versions

### 8.2 Create Namespace

    kubectl create namespace gpu-monitoring

### 8.3 Install DCGM Exporter

Recommended to specify version:

helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace gpu-monitoring \
  --version <CHART_VERSION>

Production environment recommendations:

  Use fixed chart version.
  Use fixed image tag.
  Do not use latest.
  Synchronize images in internal network environment in advance.

### 8.4 View Pod

  kubectl get pods -n gpu-monitoring -o wide

### 8.5 View DaemonSet

  kubectl get ds -n gpu-monitoring

### 8.6 View logs

  kubectl logs <dcgm-exporter-pod> -n gpu-monitoring

### 8.7 Verify metrics

  kubectl port-forward -n gpu-monitoring <dcgm-exporter-pod> 9400:9400
  curl http://127.0.0.1:9400/metrics

---

## NineI don't know.Prometheus Integration Methods

Prometheus commonly uses two methods to scrape DCGM Exporter:

    1. Prometheus Operator + ServiceMonitor
    2. Regular Prometheus scrape_configs

### 9.1 Using ServiceMonitor

Suitable for kube-prometheus-stack or Prometheus Operator environments.

ServiceMonitor example:

    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: dcgm-exporter
      namespace: monitoring
      labels:
        release: prometheus
    spec:
      namespaceSelector:
        matchNames:
          - gpu-operator
      selector:
        matchLabels:
          app.kubernetes.io/name: dcgm-exporter
      endpoints:
        - port: metrics
          interval: 30s
          path: /metrics

Notes:

    The selector.matchLabels must match the labels of the DCGM Exporter Service.
    The port name must match the port name in the Service.
    Whether the labels.release is needed depends on the Prometheus Operator's selector configuration.

Check Service labels:

    kubectl get svc -n gpu-operator --show-labels
    kubectl describe svc <dcgm-exporter-service> -n gpu-operator

### 9.2 Using scrape_configs

Suitable for regular Prometheus.

Static configuration example:

    scrape_configs:
      - job_name: 'dcgm-exporter'
        static_configs:
          - targets:
              - 'gpu-node-01:9400'
              - 'gpu-node-02:9400'

Kubernetes SD example:

    scrape_configs:
      - job_name: 'dcgm-exporter'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            action: keep
            regex: gpu-operator
          - source_labels: [__meta_kubernetes_pod_name]
            action: keep
            regex: .*dcgm.*exporter.*
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: __address__
            replacement: $1:9400

Actual configuration should be adjusted according to the Namespace, Pod label, Service, and network method of DCGM Exporter.

### 9.3 Verify Prometheus Target

Enter Prometheus Web:

    Status
      ↓
    Targets

Search for:

    dcgm
    gpu
    nvidia

Confirm status:

    UP

If it is DOWN, check:

- ServiceMonitor selector;
- Service port name;
- Namespace selector;
- Prometheus RBAC;
- NetworkPolicy;
- Pod IP reachability;
- Whether the Exporter exposes `/metrics`;
- Prometheus logs.

---

## TenI don't know.Common PromQL Queries

### 10.1 View all GPU utilization

    DCGM_FI_DEV_GPU_UTIL

### 10.2 View GPU utilization by node

    avg by (Hostname, gpu) (DCGM_FI_DEV_GPU_UTIL)

Actual labels may be:

    instance
    node
    Hostname
    gpu
    UUID
    device
    pod
    namespace

Use the current metrics output as reference.

View original labels:

    DCGM_FI_DEV_GPU_UTIL

### 10.3 View GPU memory usage

    DCGM_FI_DEV_FB_USED

### 10.4 Calculate memory usage rate /think

DCGM_FI_DEV_FB_USED
/
(DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)
* 100

### 10.5 Viewing GPU Temperature

    DCGM_FI_DEV_GPU_TEMP

### 10.6 Viewing GPU Power Usage

    DCGM_FI_DEV_POWER_USAGE

### 10.7 Viewing XID Errors

    DCGM_FI_DEV_XID_ERRORS

### 10.8 Viewing 5-Minute Average GPU Utilization

    avg_over_time(DCGM_FI_DEV_GPU_UTIL[5m])

### 10.9 Viewing Long-Term Low-Utilization GPUs

    avg_over_time(DCGM_FI_DEV_GPU_UTIL[30m]) < 5

Note:

    Low utilization doesn't necessarily indicate an issue.
    Low utilization during inference service off-peak hours may be normal.
    Needs to be judged in combination with memory, Pod status, and business traffic.

### 10.10 Viewing High Memory Usage

    (
      DCGM_FI_DEV_FB_USED
      /
      (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)
      * 100
    ) > 90

### 10.11 Viewing High-Temperature GPUs

    DCGM_FI_DEV_GPU_TEMP > 80

---

## ElevenI don't know.Grafana Dashboard Design

A single GPU utilization chart is not recommended for the GPU Dashboard.

Recommend a layered design.

### 11.1 Cluster Overview Panel

Display:

- Number of GPU nodes;
- Total number of GPUs;
- Number of allocated GPUs;
- Number of idle GPUs;
- Average GPU utilization;
- Average GPU memory usage;
- Number of high-temperature GPUs;
- Number of XID errors;
- Number of GPU Pods in Pending state;
- Device Plugin status;
- DCGM Exporter status.

### 11.2 Node Dimension Panel

Display:

- GPU utilization per GPU node;
- Memory usage per GPU node;
- Temperature per GPU node;
- Power usage per GPU node;
- Node CPU / Memory;
- Number of GPU Pods;
- Node Ready status;
- Device Plugin Pod status;
- DCGM Exporter Pod status.

### 11.3 GPU Card Dimension Panel

Display:

- GPU number;
- GPU UUID;
- GPU model;
- GPU utilization;
- Memory usage;
- Temperature;
- Power usage;
- XID;
- ECC;
- Current Pod in use;
- Current Namespace in use.

### 11.4 Pod / Namespace Dimension Panel

Display:

- Namespace GPU usage;
- Pod GPU utilization;
- Pod memory usage;
- Pod restarts;
- Pod pending status;
- Pod node;
- Pod application;
- Pod runtime.

### 11.5 Cost and Utilization Panel

Display:

- GPU allocation per team;
- Average GPU utilization per team;
- Long-term low-utilization GPU Pods;
- Long-term memory-occupied but low-utilization Pods;
- Number of idle GPUs;
- GPU usage trend;
- Whether expansion or recycling is needed.

---

## TwelveI don't know.Grafana Dashboard Import

### 12.1 Using Official or Community Dashboard

You can use the NVIDIA DCGM Exporter Dashboard from the Grafana Dashboard Marketplace.

When importing, note:

- Prometheus data source name;
- Whether metric names match;
- Whether labels match;
- Whether GPU Operator is used;
- Whether MIG is enabled;
- Whether Namespaces differ;
- Whether DCGM Exporter versions differ.

### 12.2 Custom Dashboard

Recommend custom variables:

    cluster
    node
    namespace
    pod
    gpu
    model
    workload

Common variable queries:

    label_values(DCGM_FI_DEV_GPU_UTIL, Hostname)

or:

    label_values(DCGM_FI_DEV_GPU_UTIL, instance)

Actual labels are based on metrics in Prometheus.

### 12.3 Dashboard Common Issues

If panels show no data:

- Check if Prometheus Target is UP;
- Check if PromQL metric names exist;
- Check if label names match;
- Check time range;
- Check ServiceMonitor;
- Check if Prometheus has scraped DCGM Exporter;
- Check if DCGM Exporter `/metrics` is normal.

---

## ThirteenI don't know.GPU Alert Strategy Design Principles

GPU alerts should not simply be set as:

    GPU utilization > 90% triggers an alert

Because high GPU utilization for training tasks may be normal.

GPU alerts should be categorized:

    1. Hardware health alerts
    2. Temperature and power alerts
    3. Scheduling anomaly alerts
    4. Resource waste alerts
    5. Business anomaly auxiliary alerts
    6. Monitoring component self-alerts

### 13.1 Alert Levels

Recommend dividing into:

    Warning:
        Needs attention but not necessarily immediate action.

    Critical:
        May affect business or hardware safety, needs timely handling.

    Info:
        Used for resource governance, daily reports, and capacity analysis.

### 13.2 Alerts Should Have Duration

Do not trigger alerts due to transient fluctuations.

Example:

    for: 5m
    for: 10m
    for: 30m

Different metrics should have different durations.

### 13.3 Alerts Should Include Context

Alert content should include:

- Cluster;
- Node;
- GPU number;
- GPU UUID;
- Namespace;
- Pod;
- Current value;
- Threshold;
- Duration;
- Handling suggestions;
- Dashboard link;
- Runbook link.

---

## FourteenI don't know.PrometheusRule Example

The following example needs to be adjusted according to actual labels.

### 14.1 High-Temperature GPU Alert /think

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alert-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: gpu.rules
      rules:
        - alert: GPUHighTemperature
          expr: DCGM_FI_DEV_GPU_TEMP > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "GPU Temperature Too High"
            description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} temperature is {{ $value }}°C for more than 5 minutes."

### 14.2 GPU Critical High Temperature Alert

    - alert: GPUCriticalTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 90
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "GPU Severe High Temperature"
        description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} temperature is {{ $value }}°C. Check cooling, fan, chassis airflow and workload immediately."

### 14.3 GPU Memory Usage Too High

    - alert: GPUMemoryUsageHigh
      expr: |
        (
          DCGM_FI_DEV_FB_USED
          /
          (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)
          * 100
        ) > 90
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "GPU Memory Usage Too High"
        description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} memory usage is above 90% for more than 10 minutes."

### 14.4 GPU XID Error Alert

    - alert: GPUXIDError
      expr: DCGM_FI_DEV_XID_ERRORS > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "GPU XID Error Occurred"
        description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} reported XID error value {{ $value }}. Check dmesg and GPU health immediately."

### 14.5 GPU Long-Term Low Utilization Alert

    - alert: GPULowUtilization
      expr: avg_over_time(DCGM_FI_DEV_GPU_UTIL[1h]) < 5
      for: 2h
      labels:
        severity: info
      annotations:
        summary: "GPU Long-Term Low Utilization"
        description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} has average utilization below 5% for 2 hours. Check whether the GPU is idle or reserved by inactive workload."

Note:

    The low utilization alert is not recommended to be sent as Critical.
    It is more suitable for resource governance and cost optimization.
    Low utilization during inference business off-peak hours may be normal, and needs to be judged in combination with business traffic.

### 14.6 DCGM Exporter Down

    - alert: DCGMExporterDown
      expr: up{job=~".*dcgm.*"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "DCGM Exporter Unavailable"
        description: "Prometheus target {{ $labels.instance }} for DCGM Exporter has been down for more than 5 minutes."

---

## Fifteen, AlertManager Notification Design

After GPU alerts are integrated with AlertManager, it is necessary to configure grouping, suppression, and notification channels.

### 15.1 Alert Grouping

Recommend grouping by the following dimensions:

    cluster
    alertname
    node
    severity

Example:

    group_by: ['cluster', 'alertname', 'node']

### 15.2 Alert Suppression

Example:

    If a GPU node is NotReady, suppress some GPU metric alerts on that node.
    If DCGM Exporter Down, suppress GPU utilization missing alerts on that node.

### 15.3 Notification Channels

Common channels:

- Email;
- Webhook;
- Enterprise WeChat;
- DingTalk;
- Slack;
- PagerDuty;
- Self-developed OnCall platform.

### 15.4 Alert Content Suggestions

Alert content should at least include:

Alert Name  
Severity Level  
Cluster Name  
Node Name  
GPU Number  
GPU UUID  
Namespace  
Pod  
Current Value  
Threshold  
Duration  
Dashboard Link  
Runbook Link  

---

## Sixteen. GPU Metrics and Kubernetes Metrics Integration  

Viewing GPU metrics in isolation is often insufficient.  

Integration with Kubernetes metrics is required.  

### 16.1 GPU Pod Pending  

From kube-state-metrics:  

    kube_pod_status_phase{phase="Pending"}  

Combined with Pod resource requests:  

    kube_pod_container_resource_limits{resource="nvidia_com_gpu"}  

The actual metric name may vary slightly depending on the kube-state-metrics version.  

### 16.2 Device Plugin Status  

Monitor:  

- nvidia-device-plugin DaemonSet desired / available;  
- Device Plugin Pod Restart;  
- Device Plugin Pod Ready;  
- Device Plugin Logs.  

### 16.3 DCGM Exporter Status  

Monitor:  

    up{job=~".*dcgm.*"}  

And Pod status:  

    kube_pod_status_ready  
    kube_pod_container_status_restarts_total  

### 16.4 GPU Node Status  

Monitor:  

- Node Ready;  
- Node CPU;  
- Node Memory;  
- Node Disk;  
- Node Network;  
- kubelet;  
- containerd;  
- GPU Temperature;  
- GPU XID;  
- GPU Utilization.  

### 16.5 Business Service Status  

Monitor:  

- QPS;  
- Error Rate;  
- P95 / P99 Latency;  
- Model Load Failure;  
- Inference Timeout;  
- CUDA OOM;  
- Business Log Errors.  

---

## Seventeen. GPU Metrics and Log Integration  

GPU metrics tell you "what has changed".  

Logs tell you "why it happened".  

### 17.1 Log Investigation After Metric Anomalies  

GPU utilization suddenly drops:  

    Check business Pod logs  
    Check data loading logs  
    Check network request logs  
    Check model inference logs  
    Check CUDA errors  

GPU memory suddenly increases:  

    Check model loading logs  
    Check batch size  
    Check concurrency  
    Check if multiple models are loaded  
    Check for abnormal processes  

GPU XID error:  

    Check node kernel logs  
    Check business Pod runtime  
    Check driver logs  
    Check temperature and power  
    Check for PCIe errors  

### 17.2 Loki Query Examples  

Check GPU Pod errors:  

    {namespace="ai-prod", pod=~"gpu-.*"} |= "ERROR"  

Check CUDA OOM:  

    {namespace=~"ai-.*"} |= "CUDA out of memory"  

Check XID-related logs:  

    {node="<gpu-node-name>"} |= "Xid"  

Check model load failure:  

    {namespace="ai-prod"} |= "model load failed"  

### 17.3 Elasticsearch / Kibana Query Examples  

Query CUDA OOM:  

    message:"CUDA out of memory"  

Query GPU XID:  

    message:"Xid" AND node:"gpu-node-01"  

Query a specific business Pod:  

    kubernetes.namespace:"ai-prod" AND kubernetes.pod.name:"gpu-inference-*"  

---

## Eighteen. Common GPU Monitoring Scenario Analysis  

### 18.1 GPU Utilization Long-Term at 0, but Memory is Occupied  

Possible causes:  

- Low inference service traffic, model resides in memory;  
- Training task is stuck;  
- Application loads model but no requests;  
- Business process is idle;  
- Data loading failure;  
- Program deadlock;  
- Task already failed but process not exited.  

Investigation:  

    nvidia-smi  
    kubectl logs <pod-name>  
    kubectl describe pod <pod-name>  
    Check business QPS  
    Check application logs  
    Check data source status  

Resolution:  

- If it's normal low traffic, no action needed;  
- If it's idle resource occupation, reclaim Pod;  
- If task is stuck, restart or fix the task;  
- If it's inference service, combine with business traffic to decide if scaling down is needed.  

### 18.2 High GPU Utilization, but Low Business Throughput  

Possible causes:  

- Low model efficiency;  
- Unreasonable batch size;  
- CPU preprocessing/postprocessing bottleneck;  
- Network bottleneck;  
- Slow storage read;  
- Single request too heavy;  
- Inference framework configuration issues;  
- GPU being competed by multiple processes.  

Investigation:  

    GPU utilization  
    CPU usage  
    Pod logs  
    Application QPS  
    P95/P99 latency  
    Data loading duration  
    Network throughput  
    Disk I/O  

Resolution:  

- Adjust batch size;  
- Optimize data loading;  
- Increase CPU request;  
- Use TensorRT;  
- Model quantization;  
- Split services;  
- Adjust concurrency.  

### 18.3 Continuous Memory Growth  

Possible causes:  

- Application memory leak;  
- Request cache not released;  
- Dynamically loaded model not unloaded;  
- Multiple workers reloading models;  
- Framework caching policy;  
- Batch size increase;  
- Abnormal input data.  

Investigation:  

    nvidia-smi  
    DCGM_FI_DEV_FB_USED  
    Application logs  
    Business request volume  
    Pod Restart records  
    Model loading logs  

Resolution:  

- Fix the application;  
- Limit concurrency;  
- Reduce batch size;  
- Fix model loading strategy;  
- Regular rolling restarts;  
- Use larger memory GPU;  
- Use MIG for isolation.  

### 18.4 Continuous High GPU Temperature  

Possible causes:  

- High data center temperature;  
- Abnormal airflow;  
- Inappropriate fan strategy;  
- GPU long-term full load;  
- Node dust;  
- Small GPU spacing;  
- Passive cooling card installed in inappropriate chassis;  
- Insufficient server cooling design.  

Investigation:  

    nvidia-smi  
    DCGM_FI_DEV_GPU_TEMP  
    ipmitool sensor  
    ipmitool sel list  
    Data center temperature  
    Fan speed  
    Node location /think

Processing:

- Adjust fan strategy;
- Check data center cooling;
- Reduce load;
- Migrate Pod;
- Check server airflow;
- Check hardware;
- Offline node if necessary.

### 18.5 XID Error Occurrence

Possible causes:

- Driver anomaly;
- CUDA application anomaly;
- GPU hardware issue;
- Memory error;
- PCIe issue;
- Power supply issue;
- Temperature issue;
- GPU card drop.

Troubleshooting:

    dmesg | grep -i xid
    journalctl -k | grep -i xid
    nvidia-smi -q
    DCGM_FI_DEV_XID_ERRORS
    ipmitool sel list
    kubectl get pod -A -o wide | grep <gpu-node-name>

Processing:

- Record XID code;
- Associate with business Pod;
- Check temperature and power consumption;
- Restart abnormal Pod;
- Cord node if necessary;
- Reboot node in severe cases;
- Replace GPU or contact vendor if recurring.

---

## NineteenI don't know.GPU Node Maintenance and Alert Handling Process

### 19.1 Received GPU High Temperature Alert

Handling process:

    1. Check Grafana temperature trend
    2. Check GPU utilization and power consumption
    3. Check Pods running on the node
    4. Check data center temperature and BMC sensor
    5. Determine if business is normally fully loaded
    6. If temperature continues to rise, migrate or stop low-priority tasks
    7. Cord node if necessary
    8. Check fans, airflow, and hardware

Commands:

    kubectl get pod -A -o wide | grep <gpu-node-name>
    nvidia-smi
    ipmitool sensor
    kubectl cordon <gpu-node-name>

### 19.2 Received XID Alert

Handling process:

    1. Check XID code
    2. Check node kernel logs
    3. Check Pods at the time of occurrence
    4. Check temperature, power consumption, PCIe errors
    5. Determine if it's a one-time occasional issue
    6. If recurring, isolate the node
    7. Reboot node or report hardware issue if necessary

Commands:

    dmesg | grep -i xid
    journalctl -k | grep -i xid
    kubectl get pod -A -o wide | grep <gpu-node-name>
    kubectl cordon <gpu-node-name>

### 19.3 Received GPU Long Low Utilization Alert

Handling process:

    1. Check if Pod is still running
    2. Check memory usage
    3. Check business QPS
    4. Check if experimental tasks are not cleaned up
    5. Contact business for confirmation
    6. Recycle idle tasks or reduce replicas
    7. Update capacity report

Commands:

    kubectl get pod -A -o wide | grep <gpu-node-name>
    kubectl logs <pod-name> -n <namespace>
    nvidia-smi

Note:

    Do not directly delete Pod just because of low utilization.
    Low peak and model residency in memory may be normal states.

---

## TwentyI don't know.GPU Monitoring and Capacity Planning

GPU monitoring is not only used for fault alerts but also for capacity planning.

### 20.1 Data to be Statistically Collected

Recommended to collect daily, weekly, and monthly:

- Total GPU count;
- Allocated GPU count;
- Idle GPU count;
- Average GPU utilization;
- Peak GPU utilization;
- Average memory usage;
- Peak memory usage;
- Usage per Namespace;
- Usage per business;
- Utilization per node;
- Long low utilization Pods;
- GPU Pending count;
- GPU OOM count;
- XID error count;
- High temperature count.

### 20.2 Capacity Judgment

Signals for expansion:

- GPU Pod frequently Pending;
- Production inference services cannot scale replicas;
- Training tasks are severely queued;
- GPU utilization is long high;
- Memory is long high;
- High priority tasks wait too long.

Signals for governance:

- High GPU allocation rate but low utilization;
- Experimental tasks long occupied;
- Memory occupied but no business traffic;
- Large cards run small tasks;
- Training tasks without checkpoint occupy too long;
- Resource usage differences between teams are too large.

### 20.3 GPU Cost Governance Suggestions

Recommended to establish:

- Daily GPU usage report;
- Weekly GPU usage report;
- Namespace cost allocation;
- Idle GPU alert;
- Low utilization task cleanup process;
- GPU usage application process;
- Training task duration limit;
- Experimental environment auto cleanup;
- Inference service auto scaling strategy;
- GPU model and task matching specification.

---

## Twenty-oneI don't know.Production Environment GPU Monitoring Baseline

### 21.1 Must Monitor Node Status

    [ ] Node Ready
    [ ] kubelet status
    [ ] containerd status
    [ ] NVIDIA Driver status
    [ ] Device Plugin status
    [ ] DCGM Exporter status
    [ ] GPU Operator status
    [ ] GPU node CPU / Memory / Disk / Network

### 21.2 Must Monitor GPU Metrics

    [ ] GPU utilization
    [ ] GPU memory usage
    [ ] GPU temperature
    [ ] Memory temperature
    [ ] GPU power consumption
    [ ] XID error
    [ ] ECC error
    [ ] PCIe / NVLink metrics
    [ ] MIG metrics
    [ ] GPU health status

### 21.3 Must Monitor Kubernetes Resources

    [ ] GPU Pod Pending
    [ ] GPU Pod Restart
    [ ] GPU Pod OOMKilled
    [ ] Namespace GPU usage
    [ ] GPU ResourceQuota
    [ ] Device Plugin DaemonSet Ready
    [ ] DCGM Exporter DaemonSet Ready
    [ ] GPU Operator Validator
    [ ] GPU node Taint / Label

### 21.4 Must Monitor Business Metrics

    [ ] Inference QPS
    [ ] Inference error rate
    [ ] P95 / P99 latency
    [ ] Model loading failure
    [ ] CUDA OOM
    [ ] Training throughput
    [ ] Training failure rate
    [ ] Job execution duration
    [ ] Task queueing duration

---

## 22. GPU Alert Threshold Recommendations

The following thresholds are for learning and initial design reference. In production, they must be adjusted based on GPU model, data center environment, business characteristics, and historical data.

### 22.1 Temperature

    Warning:
        GPU Temperature > 80°C for 5 minutes

    Critical:
        GPU Temperature > 90°C for 2 minutes

Note:

    Different GPU models have different safe temperature ranges.
    Please follow NVIDIA and server vendor recommendations.

### 22.2 GPU Memory

    Warning:
        GPU Memory Usage Rate > 90% for 10 minutes

    Critical:
        GPU Memory Usage Rate > 98% for 5 minutes, accompanied by CUDA OOM logs

### 22.3 GPU Utilization

    Info:
        GPU Utilization < 5% for 2 hours

    Warning:
        GPU Utilization consistently near 100% with rising business latency

Note:

    High GPU utilization alone does not necessarily require an alert.
    It should be combined with business latency, error rate, and queue length for judgment.

### 22.4 XID

    Critical:
        Alert triggered after any XID error occurs

Handling:

    First record and analyze, not necessarily restart immediately.
    Repeated XID or severe XID requires isolating the node.

### 22.5 DCGM Exporter

    Critical:
        DCGM Exporter Target Down for 5 minutes

### 22.6 Device Plugin

    Critical:
        NVIDIA Device Plugin DaemonSet unavailable for 5 minutes

### 22.7 GPU Pod Pending

    Warning:
        GPU Pod Pending exceeds 10 minutes

    Critical:
        Production Namespace GPU Pod Pending exceeds 5 minutes

---

## 23. Common GPU Monitoring Misconceptions

### 23.1 Misconception 1: Only Monitor GPU Utilization

GPU utilization is just one metric.

Also consider:

- Memory;
- Temperature;
- Power consumption;
- Pod;
- Business QPS;
- Business latency;
- Logs;
- Data loading;
- CPU;
- Network;
- Storage.

### 23.2 Misconception 2: High GPU Utilization is Always an Issue

Long-term GPU utilization above 90% for training tasks may be normal.

True anomalies include:

- High utilization with low throughput;
- High utilization with excessively high temperature;
- High utilization with rising business latency;
- High utilization with rising error rate;
- High utilization with XID occurrence.

### 23.3 Misconception 3: High Memory is Always a Fault

Constant memory residency for inference service models is normal.

Abnormalities to check:

- Whether memory continuously grows;
- Whether OOM occurs;
- Whether it affects new tasks;
- Whether memory is occupied without business traffic;
- Whether Pod is already abnormal but memory isn't released.

### 23.4 Misconception 4: Prometheus Having Data Means Monitoring is Complete

Incomplete.

Also need:

- Dashboard;
- AlertRule;
- AlertManager;
- Notification channels;
- Runbook;
- On-call process;
- Fault drills;
- Capacity reports;
- Cost governance.

### 23.5 Misconception 5: GPU Monitoring Doesn't Need Business Metrics

Incorrect.

GPU is a computing resource supporting business operations.

Without business metrics, it's impossible to determine:

- Whether high GPU utilization brings high throughput;
- Whether low GPU utilization is due to business low traffic;
- Whether high memory is due to model residency;
- Whether expansion is needed;
- Whether scaling down is needed.

---

## 24. GPU Monitoring Troubleshooting Command Summary

### 24.1 Check DCGM Exporter

    kubectl get pods -A | grep -i dcgm
    kubectl get svc -A | grep -i dcgm
    kubectl logs <dcgm-exporter-pod> -n <namespace>

### 24.2 Verify Metrics

    kubectl port-forward -n <namespace> <dcgm-exporter-pod> 9400:9400
    curl http://127.0.0.1:9400/metrics

### 24.3 Check Prometheus Targets

    Open Prometheus Web
    Status -> Targets
    Search for dcgm

### 24.4 Check GPU Nodes

    kubectl get nodes -o wide
    kubectl describe node <gpu-node-name>
    kubectl get node <gpu-node-name> --show-labels

### 24.5 Check GPU Pods

    kubectl get pod -A -o wide | grep <gpu-node-name>
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>

### 24.6 Local GPU Check on Node

    nvidia-smi
    nvidia-smi -L
    nvidia-smi -q
    nvidia-smi topo -m
    dmesg | grep -i xid
    journalctl -k | grep -i nvidia
    journalctl -k | grep -i xid
    ipmitool sensor
    ipmitool sel list

### 24.7 Check Device Plugin

    kubectl get pods -A | grep -i nvidia
    kubectl get ds -A | grep -i nvidia
    kubectl logs <device-plugin-pod> -n <namespace>

---

## 25. Production Deployment Steps

### 25.1 Step 1: Confirm GPU Node Availability

    lspci | grep -i nvidia
    nvidia-smi
    kubectl describe node <gpu-node-name>

Confirm:

    [ ] Driver is normal
    [ ] Device Plugin is normal
    [ ] Node has nvidia.com/gpu
    [ ] GPU Pod can run

### 25.2 Step 2: Deploy DCGM Exporter

Method 1:

    Use GPU Operator to manage DCGM Exporter.

Method 2: /think

# Installing DCGM Exporter via Helm Alone

## 25.3 Step 3: Integrating Prometheus

Choose:

- ServiceMonitor
- scrape_configs

Confirm:

- Prometheus Target = UP

## 25.4 Step 4: Import or Create Grafana Dashboard

Must include at least:

- GPU Utilization;
- Memory;
- Temperature;
- Power;
- XID;
- Namespace;
- Pod;
- Node;
- Device Plugin;
- DCGM Exporter.

## 25.5 Step 5: Configure Alerting Rules

Must configure at least:

- GPU High Temperature;
- GPU XID;
- GPU High Memory Usage;
- DCGM Exporter Down;
- Device Plugin Down;
- GPU Pod Pending;
- GPU Low Utilization for Long Period;
- GPU Node NotReady.

## 25.6 Step 6: Configure Notification Channels

Examples:

- Email;
- Enterprise WeChat;
- DingTalk;
- Slack;
- Webhook;
- OnCall.

## 25.7 Step 7: Write Runbook

Each alert should have a handling document:

- GPUHighTemperature -> How to Handle
- GPUXIDError -> How to Handle
- GPUMemoryUsageHigh -> How to Handle
- GPUPodPending -> How to Handle
- DCGMExporterDown -> How to Handle

## 25.8 Step 8: Drills

At least drill:

- DCGM Exporter Down;
- GPU Pod Pending;
- High Memory;
- Business CUDA OOM;
- GPU Node Cordon;
- GPU Pod Migration;
- Alert Notification Chain.

---

## Twenty-Six, GPU Monitoring Checklist

### 26.1 Before Deployment

    [ ] GPU Node Ready
    [ ] nvidia-smi Normal
    [ ] Device Plugin Normal
    [ ] Node has nvidia.com/gpu
    [ ] GPU Pod Test Passed
    [ ] Prometheus Deployed
    [ ] Grafana Deployed
    [ ] AlertManager Deployed
    [ ] Determined to Use GPU Operator or Independent DCGM Exporter
    [ ] Image Repository Accessible
    [ ] Helm Chart Version Fixed

### 26.2 During Deployment

    [ ] DCGM Exporter Pod Running
    [ ] DCGM Exporter DaemonSet Ready
    [ ] Service Normal
    [ ] /metrics Accessible
    [ ] Prometheus Target UP
    [ ] Metrics Queryable
    [ ] Grafana Can Display Data

### 26.3 After Deployment

    [ ] GPU Utilization Dashboard Normal
    [ ] Memory Dashboard Normal
    [ ] Temperature Dashboard Normal
    [ ] Power Dashboard Normal
    [ ] XID Metrics Normal
    [ ] Namespace / Pod Dimension Association Available
    [ ] Alerting Rules Loaded
    [ ] AlertManager Can Receive Alerts
    [ ] Notification Channels Available
    [ ] Runbook Written
    [ ] On-Call Staff Know Handling Process

---

## Twenty-Seven, Summary

GPU monitoring is not simply deploying a DCGM Exporter.

Complete GPU observability should cover:

    GPU Hardware Status
    NVIDIA Driver Status
    Device Plugin Status
    GPU Operator Status
    DCGM Exporter Status
    GPU Utilization
    GPU Memory
    GPU Temperature
    GPU Power
    XID Errors
    ECC Errors
    GPU Pod Pending
    GPU Pod Restart
    Namespace GPU Usage
    Business QPS / Latency / Error Rate
    Application CUDA Error Logs

When troubleshooting, maintain cross-analysis:

    GPU Low Utilization:
        Check Business Traffic, Data Loading, CPU, and Logs.

    High Memory:
        Check Model Loading, Batch Size, Concurrency, and OOM.

    GPU High Temperature:
        Check Power, Fans, Data Center, and Node Load.

    XID Errors:
        Check dmesg, journalctl, Temperature, PCIe, Power Supply, and Business Timeline.

    Pod Pending:
        Check nvidia.com/gpu, Quota, Taint, Label, CPU/Memory.

    Pod Running but No GPU:
        Check Container Toolkit, Runtime, Device Plugin, and Image.

In production environments, GPU monitoring aims to:

    Detect Failures
    Ensure Business Operations
    Reduce Waste
    Support Capacity Planning
    Improve GPU Utilization Efficiency
    Provide Data for Expansion and Governance

Do not make GPU monitoring into "just a few charts".

A truly valuable GPU monitoring system must form:

    Metrics
      ↓
    Dashboard
      ↓
    Alerts
      ↓
    Logs
      ↓
    Runbook
      ↓
    Handling Process
      ↓
    Capacity Governance

Only then can GPU clusters evolve from "being able to run tasks" to "being observable, governable, scalable, and troubleshootable" production-grade AI infrastructure.

---

## Reference Documents

- NVIDIA DCGM Exporter Documentation:
  https://docs.nvidia.com/datacenter/dcgm/latest/gpu-telemetry/dcgm-exporter.html

- NVIDIA DCGM Field Identifiers:
  https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html

- NVIDIA DCGM Exporter GitHub:
  https://github.com/NVIDIA/dcgm-exporter

- NVIDIA DCGM Exporter Helm Chart:
  https://nvidia.github.io/dcgm-exporter/

- NVIDIA GPU Operator:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/

- NVIDIA GPU Telemetry for Kubernetes:
  https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/kube-prometheus.html

- Prometheus Documentation:
  https://prometheus.io/docs/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- AlertManager Documentation:
  https://prometheus.io/docs/alerting/latest/alertmanager/

- Grafana Documentation:
  https://grafana.com/docs/

- Kubernetes Monitoring with kube-state-metrics:
  https://github.com/kubernetes/kube-state-metrics