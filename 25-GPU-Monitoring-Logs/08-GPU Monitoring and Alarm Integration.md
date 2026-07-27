# 08-GPU Monitoring and Alarm Integration

## Document Description

This document aims to outline the monitoring, metric collection, Prometheus integration, Grafana visualization, AlertManager alerts, GPU troubleshooting, and production environment observability baselines for Kubernetes GPU clusters.

It addresses the following key questions:

- Why can't GPU performance be monitored manually solely through `nvidia-smi`?
- What metrics should be collected for GPU monitoring?
- What are DCGM and DCGM Exporter, and how do they function?
- How is DCGM Exporter integrated into Kubernetes?
- How to use the DCGM Exporter within a GPU Operator?
- How does Prometheus capture GPU metrics?
- How to choose between ServiceMonitor and scrape_configs?
- Which panels should be included in a Grafana GPU dashboard?
- How to set up alerts for GPU temperature, memory usage, utilization, power consumption, XID, and ECC errors?
- Is low GPU utilization always a sign of waste?
- Is high memory usage necessarily abnormal?
- How to diagnose GPU Pod issues by combining metrics, logs, and events?
- How to establish baseline monitoring and alerting for GPU resources in a production environment?

This document is recommended for reading after completing the following chapters:

- 03-NVIDIA Driver Installation and Verification
- 04-CUDA Installation and Testing
- 05-K8S-GPU Resource Concepts and Scheduling Principles
- 06-NVIDIA Device Plugin and Operator Installation
- 07-GPU Pod Deployment and Scheduling Practices

---

## Tags

#Kubernetes #GPU #NVIDIA #DCGM #DCGMExporter #Prometheus #Grafana #AlertManager #SRE #Observability #Ops Troubleshooting

---

## Recommended Reading Path

Recommended path:

    06-GPU and AI Infrastructure/03-GPU Monitoring and Troubleshooting/08-GPU-Monitoring-and-Alarm-Integration.md

---

## I. Why Specialized GPU Monitoring is Needed

Traditional Kubernetes monitoring primarily focuses on:

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

However, GPU nodes require additional monitoring for specific metrics.

GPUs are high-cost, high-power-consuming, high-density, and high-value computing resources. Without dedicated monitoring, operators can only determine:

    Whether a Pod is running or not;
    Whether a Node is ready;
    If CPU or memory usage is high.

But they cannot know:

    Whether the GPU is actually being used;
    If the GPU is idle but still consuming resources;
    If the video memory is nearly full;
    If the GPU is overheating;
    If the GPU is running at reduced performance;
    If power consumption is restricted;
    If there are XID or ECC errors;
    If there are GPU failures;
    If the Device Plugin is malfunctioning;
    Whether a GPU Pod is pending;
    If a team is continuously using a GPU;
    Whether the current GPU resource utilization warrants scaling.

The core goal of GPU monitoring is not just to generate visual reports but to establish a comprehensive system that includes:

    Metric collection
      ↓
    Dashboard display
      ↓
    Alert triggering
      ↓
    Log correlation
      ↓
    Event analysis
      ↓
    Fault diagnosis
      ↓
    Capacity planning
      ↓
    Cost management

---

## II. Overall GPU Observability Architecture

In a production environment, GPU observability can be divided into five layers:

    1. Node Hardware Layer
    2. NVIDIA Driver Layer
    3. Kubernetes Scheduling Layer
    4. Container and Application Layer
    5. Monitoring, Logging, and Alerting Layer

The overall flow is as follows:

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

At the Kubernetes level:

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

At the logging level:

    GPU Pod Logs
      ↓
    Fluent Bit / Promtail
      ↓
    Loki / Elasticsearch
      ↓
    Grafana / Kibana
      ↓
    Combined Metric and Log Troubleshooting

---

## III. DCGM and DCGM Exporter

### 3.1 What is DCGM

DCGM, short for Data Center GPU Manager, is a set of tools provided by NVIDIA for managing and monitoring GPUs in data centers.

DCGM offers the following metrics:

- GPU utilizationThese metrics are related to the GPU model, DCGM configuration, and the exporter metric set; not all environments collect them by default.

### 4.6 Kubernetes Scheduling Category

GPU metrics also need to be considered in conjunction with Kubernetes resource status:

- Node Ready;
- Node Allocatable GPU;
- GPU Pod Pending;
- GPU Pod Restart;
- Device Plugin Pod Status;
- DCGM Exporter Pod Status;
- GPU Operator Pod Status;
- Namespace GPU Quota;
- Node Where the GPU Pod Is Located;
- Team and Application to Which the GPU Pod Belongs.

These metrics usually come from:

    kube-state-metrics
    kubelet / cAdvisor
    Prometheus Kubernetes SD
    DCGM Exporter
    GPU Operator Components

---

## V. Common DCGM Metric Explanations

The following metric names are based on common outputs from the DCGM Exporter; whether these metrics are actually available depends on the GPU model, DCGM Exporter version, metrics configuration, and GPU Operator configuration.

### 5.1 GPU Utilization

Metric:

    DCGM_FI_DEV_GPU_UTIL

Meaning:

    The percentage of GPU compute resources that are currently in use.

Example PromQL:

    DCGM_FI_DEV_GPU_UTIL

Viewing by Pod:

    DCGM_FI_DEV_GPU_UTIL{pod!="", namespace!=""}

Production Explanation:

- High utilization is not necessarily abnormal; for example, training tasks with over 90% utilization for an extended period may be normal.
- Low utilization is also not necessarily abnormal, especially during off-peak推理 times.
- If utilization remains at 0% continuously while the Pod is consuming memory, it may indicate unnecessary occupation of resources.
- Large fluctuations in utilization could suggest bottlenecks in data loading, CPU performance, network issues, or storage limitations.

### 5.2 Used Video Memory

Metric:

    DCGM_FI_DEV_FB_USED

Meaning:

    The amount of video memory currently in use by the GPU, typically measured in MiB.

Example:

    DCGM_FI_DEV_FB_used

### 5.3 Free Video Memory

Metric:

    DCGM_FI_DEV_FB_FREE

Meaning:

    The remaining amount of video memory available on the GPU, also measured in MiB.

### 5.4 Video Memory Utilization Rate

This can be calculated using PromQL:

    DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_used + DCGM_FI_DEV_FB_FREE) * 100

Example for viewing by Pod:

    DCGM_FI_DEV_FB_USED{pod!="", namespace!=""}
    /
    (DCGM_FI_DEVFBUSED{pod!“”, namespace!“”) + DCGM_FI_DEV_FB_FREE{pod!“”, namespace!“”})
    * 100

Note:

- If the label dimensions are inconsistent, use `on()` or `ignoring()` to adjust PromQL matching.
- The actual query should be adjusted according to the label structure in your cluster.

### 5.5 GPU Temperature

Metric:

    DCGM_FI_DEV_GPU_TEMP

Meaning:

    The current temperature of the GPU, typically measured in degrees Celsius.

Example:

    DCGM_FI_DEV_GPU(Temp)

### 5.6 Video Memory Temperature

Metric:

    DCGM_FI_DEV_MEMORY_TEMP

Meaning:

    The temperature of the video memory.

Not all GPUs provide this metric.

### 5.7 Power Consumption

Metric:

    DCGM_FI_DEV_POWER_USAGE

Meaning:

    The current power consumption of the GPU.

Example:

    DCGM_FI_DEV_POWER_usage

### 5.8 XID Errors

Metric:

    DCGM_FI_DEV_XID_errors

Meaning:

    The most recent value of XID errors.

XID errors should be analyzed in conjunction with node kernel logs.

Node log troubleshooting:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

### 5.9 ECC Errors

Common metrics:

    DCGM_FI_DEV_ECC_SBE_VOL_TOTAL
    DCGM_FI_DEV_ECC_DBE_volTOTAL

Meaning:

    The cumulative number of single-bit and double-bit ECC errors.

Production recommendations:

- Do not simply ignore ECC errors.
- A continuous increase in these errors may indicate potential hardware issues with the video memory.

---

## VI. Deployment Options

There are two common ways to deploy GPU monitoring:

    1. Use the DCGM Exporter within the GPU Operator
    2. Install the DCGM Exporter separately

### 6.1 Integrating with the GPU Operator

If your cluster already uses the GPU Operator, it is generally recommended to manage the DCGM Exporter through the Operator.

Advantages:

- Unified component management;
- Consistent with DeviceUse a fixed chart version.
Use a fixed image tag.
Do not use the latest version.
Synchronize the image in advance for internal network environments.

### 8.4 Viewing Pods

    kubectl get pods -n gpu-monitoring -o wide

### 8.5 Viewing DaemonSets

    kubectl get ds -n gpu-monitoring

### 8.6 Viewing Logs

    kubectl logs <dcgm-exporter-pod> -n gpu-monitoring

### 8.7 Verifying Metrics

    kubectl port-forward -n gpu-monitoring <dcgm-exporter-pod> 9400:9400
    curl http://127.0.0.1:9400/metrics

---

## IX. Prometheus Integration Methods

There are two common ways to integrate Prometheus with the DCGM Exporter:

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
    Whether to include the labels.release depends on the Prometheus Operator's configuration.

Viewing Service labels:

    kubectl get svc -n gpu-operator --show-labels
    kubectl describe svc <dcgm-exporter-service> -n gpu-operator

### 9.2 Using scrape_configs

Suitable for regular Prometheus installations.

Static configuration example:

    scrapeConfigs:
      - job_name: 'dcgm-exporter'
        static_configs:
          - targets:
              - 'gpu-node-01:9400'
              - 'gpu-node-02:9400'

Kubernetes SD example:

    scrape_configs:
      - job_name: 'dcgm-exporter'
        kubernetes_sdconfigs:
          - role: pod
        relabelConfigs:
          - source_labels: [__meta_kubernetes_namespace]
            action: keep
            regex: gpu-operator
          - source_labels: [__meta_kubernetes_pod_name]
            action: keep
            regex: .*dcgm.*exporter.*
          - source_labels: [__meta_kubernetes_pod_ip]
            target_label: __address__
            replacement: $1:9400

The actual configuration should be adjusted based on the DCGM Exporter's Namespace, Pod labels, Service settings, and network configurations.

### 9.3 Verifying Prometheus Targets

Enter the Prometheus Web interface:

    Status
      ↓
    Targets

Search for:

    dcgm
    gpu
    nvidia

Check the status:

    UP

If it shows DOWN, check the following:

- ServiceMonitor selector;
- Service port settings;
- Namespace selector;
- Prometheus RBAC permissions;
- NetworkPolicy settings;
- Pod IP connectivity;
- Whether the Exporter is exposing `/metrics`;
- Prometheus logs.

---

## X. Common PromQL Queries

### 10.1 Viewing Overall GPU Utilization

    DCGM_FI_DEV_GPU_UTIL

### 10.2 Viewing GPU Utilization by Node

    avg by (Hostname, gpu) (DCGM_FI_DEV_GPU_UTIL)

Actual label names may include:

    instance
    node
    Hostname
    gpu
    UUID
    device
    pod
    namespace

Refer to the actual metrics output for accurate labels.

View original label:

    DCGM_FI_DEV_gpu_UTIL

### 10.3 Viewing GPU Video Memory Usage

    DCGM_FI_DEV_FB_used

### 10.4 Calculating Video Memory Utilization Rate

    DCGM_FI_DEV_FB_USED
    /
    (DCGM_FI_DEV_FB.used + DCGM_FI_DEV_FB_FREE)
    * 100

### 10.5 Viewing GPU Temperature

    DCGM_FI_DEV_GPU_TEMP

### 10.6 Viewing GPU Power Consumption

    DCGM_FI_DEV_POWER_usage

### 10.7 Checking for XID Errors

    DCGM_FIDEV_XID_errors

### 10.8 Viewing Average GPU Utilization Over the Last 5 Minutes

    avg_over_time(DCGM_FI_DEV_GPU_UTIL[5m])

### 10.9 Identifying GPUs with LongYou can use the NVIDIA DCGM Exporter Dashboard available in the Grafana Dashboard marketplace.

When importing, pay attention to the following:

- The name of the Prometheus data source;
- Whether the metric names match;
- Whether the labels match;
- Whether the GPU Operator is being used;
- Whether MIG is enabled;
- Whether the Namespace is different;
- Whether the version of the DCGM Exporter is different.

### 12.2 Custom Dashboard

It is recommended to customize variables such as:

    cluster
    node
    namespace
    pod
    gpu
    model
    workload

Common variable queries include:

    label_values(DCGM_FI_DEV_GPU_UTIL, Hostname)

or:

    label_values(DCGM_FI_DEV_GPU_UTIL, instance)

The actual labels should be based on the metrics in Prometheus.

### 12.3 Common Issues with Dashboards

If a panel shows no data:

- Check whether the Prometheus Target is running;
- Verify if the PromQL metric name exists;
- Confirm that the label names match;
- Check the time range;
- Verify the ServiceMonitor;
- Ensure that Prometheus is capturing data from the DCGM Exporter;
- Check if the `/metrics` endpoint of the DCGM Exporter is functioning properly.

---

## Chapter Thirteen: Principles for Designing GPU Alarm Strategies

GPU alarms should not be set too simply, such as triggering an alarm whenever the GPU utilization exceeds 90%, because high utilization during training tasks may be normal.

GPU alarms should be categorized:

    1. Hardware health alarms
    2. Temperature and power consumption alarms
    3. Scheduling anomaly alarms
    4. Resource waste alarms
    5. Auxiliary alarms for business anomalies
    6. Alerts related to monitoring components themselves

### 13.1 Alarm Levels

It is suggested to define the following levels:

    Warning:
        Requires attention but does not necessarily require immediate action.

    Critical:
        May affect business operations or hardware safety and requires prompt handling.

    Info:
        Used for resource management, daily reports, and capacity analysis.

### 13.2 Alarms Should Have a Duration

Alarms should not be triggered by temporary fluctuations.

Examples:

    - for: 5 minutes
    - for: 10 minutes
    - for: 30 minutes

Different metrics may require different duration settings.

### 13.3 Alarms Should Include Context Information

The alarm message should include the following details:

- Cluster;
- Node;
- GPU ID;
- GPU UUID;
- Namespace;
- Pod;
- Current value;
- Threshold;
- Duration;
- Action recommendations;
- Dashboard link;
- Runbook link.

---

## Chapter Fourteen: Examples of PrometheusRules

The following examples need to be adjusted according to the actual labels used.

### 14.1 GPU High Temperature Alarm

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
                summary: "GPU temperature is too high"
                description: "The temperature of GPU {{ $labels gpu }} on host {{ $labels.hostname }} is {{ $value }}°C and has remained above this level for more than 5 minutes."

### 14.2 Critical GPU High Temperature Alarm

    - alert: GPUCriticalTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 90
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "GPU temperature is extremely high"
        description: "The temperature of GPU {{ $labelsgpu }} on host {{ $labels.hostname }} is {{ $value }}°C. Immediately check the cooling system, fans, chassis airflow, and workload."

### 14.3 High GPU Memory Usage Alarm

    - alert: GPUMemoryUsageHigh
      expr: |
        (
          DCGM_FI_DEV_FB_USED
          /
          (DCGM_FI_DEV_FB_used + DCGM_FI_DEV_FB_FREE)
          * 100
        ) > 90
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "GPU memory usage is too high"
        description: "The memory usage of GPU {{ $labels.gpu }} on host {{ $labels.hostname }} is above 90% and has persisted for more than 10 minutes."

### 14.4 GPU XID Error Alarm

    - alert: GPUXIDError
      expr:## Section Sixteen: Linkage between GPU Metrics and Kubernetes Metrics

Analyzing GPU metrics alone often provides insufficient information. It is necessary to integrate them with Kubernetes metrics for a more comprehensive understanding.

### 16.1 Pending GPU Pods

Information comes from `kube-state-metrics`:

    `kube_pod_status_phase{phase="Pending"}`

Combined with Pod resource requests:

    `kube_pod_container_resource_limits{resource="nvidia_com_gpu"}`

The actual metric names may vary slightly depending on the version of `kube-state-metrics`.

### 16.2 Device Plugin Status

Monitor the following:

- The desired and available status of the `nvidia-device-plugin DaemonSet`;
- Restarts of Device Plugin Pods;
- Readiness of Device Plugin Pods;
- Device Plugin logs.

### 16.3 DCGM Exporter Status

Monitor:

    `up{job=~".*dcgm.*"}`

As well as Pod status:

    `kube_pod_status_ready`
    `kube_pod_container_status_restarts_total`

### 16.4 GPU Node Status

Monitor:

- Node readiness;
- Node CPU usage;
- Node memory usage;
- Node disk status;
- Node network status;
- `kubelet` status;
- `containerd` status;
- GPU temperature;
- GPU XID;
- GPU utilization.

### 16.5 Business Service Status

Monitor:

- QPS;
- Error rate;
- P95/P99 latency;
- Model loading failures;
- Inference timeouts;
- CUDA out of memory errors;
- Business log errors.

---

## Section Seventeen: Linkage between GPU Metrics and Logs

GPU metrics indicate “what changes have occurred”, while logs explain “why these changes happened”.

### 17.1 Log Troubleshooting After Metric Abnormalities

If the GPU utilization drops suddenly:

    Check business Pod logs;
    Review data loading logs;
    Examine network request logs;
    Look at model inference logs;
    Check for CUDA errors.

If GPU memory usage increases abruptly:

    Check model loading logs;
    Verify the batch size;
    Check the number of concurrent tasks;
    Determine if multiple models are being loaded;
    Investigate any abnormal processes.

If there are XID errors:

    Check node kernel logs;
    Evaluate the running time of business Pods;
    Review driver logs;
    Monitor temperature and power consumption;
    Check for PCIe-related issues.

### 17.2 Loki Query Examples

To check for GPU Pod errors:

    `{namespace="ai-prod", pod=~"gpu-.*"} |= "ERROR"

To detect CUDA out of memory errors:

    `{namespace=~"ai-.*"} |= "CUDA out of memory"

To search for XID-related logs:

    {node="<gpu-node-name>"} |= "Xid"

To identify model loading failures:

    {namespace="ai-prod"} |= "model load failed"

### 17.3 Elasticsearch/Kibana Query Examples

To query for CUDA out of memory errors:

    `message:"CUDA out of memory"`

To search for GPU XID-related logs:

    `message:"Xid" AND node:"gpu-node-01"`

To query a specific business Pod:

    `kubernetes.namespace:"ai-prod" AND kubernetes.pod.name:"gpu-inference-*"

---

## Section Eighteen: Analysis of Common GPU Monitoring Scenarios

### 18.1 Low GPU Utilization but Occupied Memory

Possible causes:

- The inference service is at a low peak, and models remain in memory;
- Training tasks are stuck;
- The application loads models but does not initiate requests;
- Business processes are running idly;
- Data loading fails;
- Programs are deadlocked;
- Tasks have become abnormal, but the processes have not terminated.

Troubleshooting steps:

    Use `nvidia-smi`;
    Check `kubectl logs <pod-name>` and `kubectl describe pod <pod-name>`;
    Verify business QPS;
    Review application logs;
    Confirm the integrity of data sources.

Actions to take:

- If it is a normal low-peak period, no action is required;
- If resources are being unnecessarily occupied, terminate the Pod;
- If tasks are stuck, restart or repair them;
- For inference services, consider scaling down based on business traffic.

### 18.2 High GPU Utilization but Low Business Throughput

Possible causes:

- The model is inefficient;
- The batch size is inappropriate;
- There are bottlenecks in CPU preprocessing/postprocessing;
- Network issues exist;
- Storage read speeds are slow;
- Single requests require excessive computation;
- The inference framework configuration is incorrect;
- Multiple processes are competing for GPU resources.

Troubleshooting steps:

    Monitor GPU and CPU utilization;
    Review Pod logs;
    Check application QPS;
    Evaluate P95/P99 latency;
    Measure data loading times;
    Assess network throughput;
    Analyze disk I/O performance.

Actions### 20.2 Capacity Assessment

Signals indicating the need for expansion:

- GPU Pods frequently remain in a Pending state;
- The production inference service is unable to increase replicas;
- Training tasks are severely backlogged;
- GPU utilization remains high for an extended period;
- Video memory usage stays high;
- High-priority tasks experience long waiting times.

Signals requiring intervention:

- High GPU allocation rate but low utilization;
- Experimental tasks consume resources for an extended time;
- Video memory is used without any business traffic;
- Large GPUs are used for small-scale tasks;
- Training tasks without checkpoints take too long to complete;
- Significant differences in resource usage among different teams.

### 20.3 Recommendations for GPU Cost Management

It is recommended to establish:

- Daily and weekly reports on GPU usage;
- Namespace-based cost allocation;
- Alarms for idle GPUs;
- Processes for clearing low-utilization tasks;
- Procedures for requesting GPU usage;
- Limits on the duration of training tasks;
- Automatic cleanup of experimental environments;
- Automatic scaling-down strategies for inference services;
- Guidelines for matching GPU models with specific tasks.

---

## Chapter 21: Baseline Monitoring for Production Environment GPUs

### 21.1 Node Statuses That Must Be Monitored

    [ ] Node Ready
    [ ] kubelet status
    [ ] containerd status
    [ ] NVIDIA Driver status
    [ ] Device Plugin status
    [ ] DCGM Exporter status
    [ ] GPU Operator status
    [ ] CPU / Memory / Disk / Network resources of GPU nodes

### 21.2 GPU Metrics That Must Be Monitored

    [ ] GPU utilization
    [ ] Video memory usage
    [ ] GPU temperature
    [ ] Video memory temperature
    [ ] GPU power consumption
    [ ] XID errors
    [ ] ECC errors
    [ ] PCIe / NVLink metrics
    [ ] MIG metrics
    [ ] GPU health status

### 21.3 Kubernetes Resources That Must Be Monitored

    [ ] GPU Pod Pending status
    [ ] GPU Pod restart attempts
    [ ] OOMKilled events for GPU Pods
    [ ] Namespace-based GPU usage
    [ ] GPU ResourceQuota settings
    [ ] Status of Device Plugin DaemonSet
    [ ] Status of DCGM Exporter DaemonSet
    [ ] Validation results from the GPU Operator
    [ ] Taints and Labels associated with GPU nodes

### 21.4 Business Metrics That Must Be Monitored

    [ ] Inference QPS
    [ ] Inference error rate
    [ ] P95 / P99 latency levels
    [ ] Model loading failures
    [ ] CUDA Out of Memory (OOM) incidents
    [ ] Training throughput
    [ ] Training failure rate
    [ ] Execution duration of Jobs
    [ ] Task queue length

---

## Chapter 22: Recommended Thresholds for GPU Alerts

The following thresholds are based on learning and initial design considerations. In production environments, they must be adjusted according to the specific GPU model, data center conditions, business characteristics, and historical data.

### 22.1 Temperature

    Warning:
        GPU temperature exceeds 80°C for 5 consecutive minutes

    Critical:
        GPU temperature exceeds 90°C for 2 consecutive minutes

Note:

    The safe temperature range varies depending on the GPU model.
    Refer to NVIDIA's recommendations as well as those provided by the server manufacturer.

### 22.2 Video Memory Usage

    Warning:
        Video memory usage exceeds 90% for 10 consecutive minutes

    Critical:
        Video memory usage exceeds 98% for 5 consecutive minutes, accompanied by CUDA OOM logs

### 22.3 GPU Utilization

    Info:
        GPU utilization is below 5% for 2 hours

    Warning:
        If GPU utilization remains close to 100% for an extended period while business latency increases, it may indicate an issue.

Note:

    High GPU utilization alone does not necessarily require an alert.
    It should be evaluated in conjunction with business latency, error rates, and queue lengths.

### 22.4 XID Errors

    Critical:
    An alarm should be triggered whenever any XID error occurs

Action Steps:

    First, record and analyze the issue; immediate restart may not be necessary.
    If repeated or severe XID errors occur, isolate the affected node.

### 22.5 DCGM Exporter

    Critical:
    If the DCGM Exporter Target is down for 5 consecutive minutes, an alarm should be triggered

### 22.6 Device Plugin

    Critical:
    If the NVIDIA Device Plugin DaemonSet becomes unavailable for 5 consecutive minutes, an alarm should be triggered

### 22.7 GPU Pod Pending Status

    Warning:
    If a GPU Pod remains in a Pending- Node;
- Device Plugin;
- DCGM Exporter.

### 25.5 Step Five: Configure Alarm Rules

At least configure the following:

- GPU High Temperature;
- GPU XID;
- GPU Memory Usage Too High;
- DCGM Exporter Down;
- Device Plugin Down;
- GPU Pod Pending;
- GPU Long-Term Low Utilization;
- GPU Node NotReady.

### 25.6 Step Six: Configure Notification Channels

For example:

- Email;
- WeCom;
- DingTalk;
- Slack;
- Webhook;
- OnCall.

### 25.7 Step Seven: Prepare Runbooks

Each alarm should have a corresponding handling document:

    GPUHighTemperature -> How to Handle It
    GPUXIDError -> How to Handle It
    GPUMemoryUsageHigh -> How to Handle It
    GPUPodPending -> How to Handle It
    DCGMExporterDown -> How to Handle It

### 25.8 Step Eight: Conduct Drills

At least perform drills for the following scenarios:

- DCGM Exporter Down;
- GPU Pod Pending;
- High Memory Usage;
- Business CUDA Out of Memory;
- GPU Node Cordon;
- GPU Pod Migration;
- Alarm Notification Processes.

---

## Chapter 26: GPU Monitoring Checklist

### 26.1 Before Deployment

    [ ] GPU Nodes Are Ready
    [ ] nvidia-smi Is Functional
    [ ] Device Plugin Is Working Properly
    [ ] The Node Can Be Reached via nvidia.com/gpu
    [ ] GPU Pod Tests Have Passed
    [ ] Prometheus Has Been Deployed
    [ ] Grafana Has Been Deployed
    [ ] AlertManager Has Been Deployed
    [ ] It Has Been Decided Whether to Use the GPU Operator or a Standalone DCGM Exporter
    [ ] The Image Repository Is Accessible
    [ ] The Helm Chart Version Is Fixed

### 26.2 During Deployment

    [ ] DCGM Exporter Pod Is Running
    [ ] DCGM Exporter DaemonSet Is Ready
    [ ] The Service Is Functional
    [ ] /metrics Is Accessible
    [ ] Prometheus Targets Are Up and Running
    [ ] Metrics Can Be Retrieved
    [ ] Data Can Be Displayed in Grafana

### 26.3 After Deployment

    [ ] GPU Utilization Dashboard Is Working Properly
    [ ] Memory Usage Dashboard Is Working Properly
    [ ] Temperature Dashboard Is Working Properly
    [ ] Power Consumption Dashboard Is Working Properly
    [ ] XID Metrics Are Available
    [ ] Namespace and Pod Dimensions Can Be Associated
    [ ] Alarm Rules Have Been Loaded
    [ ] AlertManager Receives Alarms
    [ ] Notification Channels Are Functional
    [ ] Runbooks Have Been Prepared
    [ ] On-Duty Personnel Are Aware of the Handling Procedures

---

## Chapter 27: Summary

GPU monitoring is not simply about deploying a DCGM Exporter.

A comprehensive GPU observability system should cover:

    GPU Hardware Status
    NVIDIA Driver Status
    Device Plugin Status
    GPU Operator Status
    DCGM Exporter Status
    GPU Utilization
    GPU Memory Usage
    GPU Temperature
    GPU Power Consumption
    XID Errors
    ECC Errors
    GPU Pod Pending Situations
    GPU Pod Restart Events
    Namespace-Level GPU Usage Data
    Business QPS, Latency, and Error Rates
    Application CUDA Error Logs

When troubleshooting, it is essential to conduct a comprehensive analysis:

    Low GPU Utilization:
        Check business traffic, data loading processes, CPU usage, and log records.

    High Memory Usage:
        Examine model loading times, batch sizes, concurrency levels, and whether out-of-memory errors have occurred.

    High GPU Temperature:
        Verify power consumption, fan performance, data center conditions, and node load.

    XID Errors:
        Analyze dmesg, journalctl logs, temperature readings, PCIe status, power supply issues, and business-related timing events.

    Pod Pending Situations:
        Check nvidia.com/gpu reports, quota settings, taint labels, resource labels, CPU/Memory usage statistics.

    Cases Where a Pod Is Running but Without a GPU:
        Verify the Container Toolkit, runtime environment, Device Plugin configuration, and image integrity.

In a production environment, the goal of GPU monitoring is:

    To detect faults promptly
    To ensure business continuity
    To minimize waste
    To support capacity planning decisions
    To improve GPU utilization efficiency
    To provide data-driven insights for scaling and maintenance activities

Avoid creating a GPU monitoring system that consists solely of a few static dashboards. A truly valuable system should include:

    Metrics
      ↓
    Dashboards
      ↓
    Alarms
      ↓
    Logs
      ↓
    Runbooks
      ↓
    Handling