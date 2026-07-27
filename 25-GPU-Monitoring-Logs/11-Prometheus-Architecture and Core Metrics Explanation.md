# 11-Prometheus-Architecture and Core Metrics Explanation

## Document Description

This document systematically outlines Prometheus's architecture, working principles, core components, metric models, basic PromQL knowledge, monitoring collection methods in Kubernetes, commonly used metrics, alarm rules, storage planning, and considerations for production environments.

This document does not merely focus on "how to install Prometheus" but explains from an operations/SRE perspective why Prometheus has become the most widely used monitoring core component in Kubernetes, cloud-native systems, GPU clusters, and microservice architectures.

This document addresses the following key questions:

- What is Prometheus?
- Why does Prometheus use a Pull model?
- What are the roles of Prometheus Server, Exporter, AlertManager, and Grafana?
- What are time series in Prometheus?
- What are the differences between Counter, Gauge, Histogram, and Summary?
- Why are Labels important and why can they lead to high cardinality issues?
- Where do Node, Pod, Service, and GPU metrics in Kubernetes come from?
- What data do node-exporter, kube-state-metrics, cAdvisor, and DCGM Exporter collect respectively?
- How should one get started with PromQL?
- How to check common metrics for CPU, memory, disk, network, Pods, and GPUs?
- What are the differences between Recording Rules and Alert Rules?
- In what scenarios is Prometheus suitable for single-machine deployment, HA, or remote storage?
- How should scrape_interval, retention, labels, alarms, and capacity be planned in a production environment?

This document is recommended to be read before studying the following topics:

- 12-Grafana-Dashboard Building and Custom Monitoring
- 13-AlertManager-Alarm Strategies and Notification Implementation
- 14-K8S-Monitoring Practice-Node-Pod-Service Metric Collection and Troubleshooting
- 15-Loki-Log Collection and Query Practice
- 17-Log Alarms and Automated Response Practices
- 18-Monitoring-GPU-Log Integration Cases-K8S-Pod Abnormality Detection and Report Generation

---

## Tags

#Prometheus #Kubernetes #Monitoring #Metrics #PromQL #AlertManager #Grafana #NodeExporter #kube-state-metrics #DCGMExporter #SRE #Observability

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/05-Fundamentals of Observability/11-Prometheus-Architecture and Core Metrics Explanation.md

---

## I. Why Operations Engineers Must Understand Prometheus

In traditional operations, monitoring systems typically focus on:

- Whether a host is running;
- If CPU usage is too high;
- Whether memory is insufficient;
- If disk space is full;
- Whether service ports are accessible;
- Whether application processes exist.

However, in Kubernetes, microservices, GPU clusters, and cloud-native platforms, the monitoring targets become much more complex.

Now, it is necessary to monitor:

- Node status;
- Pod status;
- Container resources;
- Deployment replicas;
- Service availability;
- Ingress traffic;
- Application QPS;
- Application error rates;
- P95/P99 latency;
- GPU utilization;
- GPU memory usage;
- GPU temperature;
- Device Plugin status;
- Business-specific custom metrics;
- Resource usage across multiple clusters, tenants, and namespaces.

The value of Prometheus lies in:

    Using a unified metric model to collect, store, query, and trigger alerts for the status of hosts, containers, Kubernetes, GPUs, middleware, and application services.

For operations/SRE professionals, Prometheus is not just a monitoring tool but also forms the foundation of a production-grade fault detection system.

---

## II. Prometheus's Core Role

Prometheus is an open-source monitoring and alerting system.

Its core capabilities include:

- Metric collection;
- Time series storage;
- Multi-dimensional label modeling;
- PromQL queries;
- Alarm rule calculation;
- Service discovery;
- Exporter ecosystem;
- Integration with AlertManager;
- Integration with Grafana;
- Support for native Kubernetes monitoring scenarios.

Prometheus is most suitable for collecting and analyzing:

- Time series metrics;
- Resource usage rates;
- Service status;
- Request volumes;
- Error rates;
- Latency distributions;
- Queue lengths;
- Business-specific metrics;
- Cluster health indicators.

Prometheus is not designed to directly replace:

- Log systems;
- Tracing systems;
- CMDBs;
- Audit systems;
- Relational databases;
- Long-term, large-scale historical data warehouses.

Prometheus focuses on:

    What the value of a specific metric is at a given point in time and how it changes over time.

---

## III. Prometheus's Overall Architecture

The typical architecture of Prometheus is as follows:

    +-----------------------+
    |      Exporters        |
    |-----------------------|
    | node-exporter         |
    | kube-state-metrics    |
    | cAdvisor / kubelet    |
    | dcgm-exporter         |
    |Collects Kubernetes object status, such as Deployment, Pod, Node, PVC, ResourceQuota, etc.

cAdvisor:
 collects resource metrics for containers, including CPU, memory, network, and file systems.

dcgm-exporter:
 collects metrics for NVIDIA GPUs, such as utilization, video memory, temperature, power consumption, XID, ECC, etc.

blackbox-exporter:
 monitors external availability of protocols like HTTP, TCP, ICMP, DNS, etc.

mysql-exporter:
 collects status metrics for MySQL databases.

redis-exporter:
 collects status metrics for Redis servers.

### 4.4 AlertManager

AlertManager is responsible for processing alerts sent by Prometheus.

It does not handle data collection or determine whether metrics are abnormal.

Prometheus determines when to trigger alerts based on Alert Rules.

AlertManager performs the following tasks:

- Grouping alerts;
- Duplicating elimination of alerts;
- Suppressing alerts;
- Silencing alerts;
- Routing alerts;
- Sending notifications.

Common notification channels include:

- Email;
- Webhook;
- WeCom;
- DingTalk;
- Slack;
- PagerDuty;
- Custom OnCall systems.

### 4.5 Grafana

Grafana is used to visualize Prometheus metrics.

It does not collect data itself.

Grafana retrieves data from Prometheus data sources and displays it in various forms, such as:

- Line charts;
- Tables;
- Status panels;
- Gauges;
- Heatmaps;
- Statistics;
- Dashboards;
- Alert panels.

Common uses of Grafana include:

- Creating dashboards for Nodes, Pods, Services, GPUs, middleware, application services, and multi-cluster overview.

### 4.6 Pushgateway

Pushgateway is used to proactively push metrics for short-lived tasks.

Typical use cases include:

- CronJobs;
- Batch processing tasks;
- Jobs with very short lifecycles;
- Tasks where metrics need to be retained after completion.

However, it is important to note that:

- Pushgateway should not be overused.
- For long-running services, it is better to expose /metrics so that Prometheus can pull data automatically.

It is not recommended to use Pushgateway as a general-purpose metric entry point, as this may disrupt Prometheus' pull model and the semantics of instance lifecycles.

---

## V. Prometheus Pull Model

Prometheus uses the Pull model by default.

This means that:

- Prometheus actively retrieves metrics from targets.

The process is as follows:

    Prometheus Server
      ↓
    Identifies targets based on scrape_configs or Service Discovery
      ↓
    Regularly requests /metrics
      ↓
    Parses the received data
      ↓
    Stores it in its local TSDB

### 5.1 Advantages of the Pull Model

Advantages include:

- Prometheus can automatically check if targets are still active;
- Targets do not need to know Prometheus' address;
- It is more compatible with Kubernetes' dynamic service discovery mechanism;
- The scrape_interval can be uniformly managed;
- It makes it easier to detect when targets become unavailable;
- Configuration settings are centralized in Prometheus;
- It helps in quickly identifying issues when services experience temporary failures.

### 5.2 Disadvantages of the Pull Model

Disadvantages include:

- Prometheus must have access to the targets;
- It may be inconvenient when dealing with cross-network, NAT, or security boundary scenarios;
- It is not ideal for short-lived tasks;
- In large-scale clusters, a large number of targets can put additional strain on Prometheus.

### 5.3 Supplemental Role of Pushgateway

For short-lived tasks that end before Prometheus has the chance to collect data, Pushgateway can be used as a backup mechanism.

However, for long-running services, it is not recommended to rely on Pushgateway.

Long-running services should:

- Expose /metrics;
- Allow Prometheus to pull data automatically.

---

## VI. Prometheus Data Model

Prometheus stores time-series data.

A time-series consists of the following components:

- Metric name + Set of labels + Timestamp + Sample value

Example:

    node_cpu_seconds_total{cpu="0", mode="idle", instance="10.0.0.21:9100", job="node-exporter"} 123456.78

In this example:

- node_cpu_seconds_total is the metric name.
- cpu="0", mode="idle", instance="10.0.0.21:9100" are labels that provide additional context about the metric.
- 123456.78 is the current sample value.
- The timestamp records when Prometheus captured this data.

---

## VII. Metric Names and Labels

### 7.1 Metric Names

Metric names typically describe what is being measured.

Examples include:

    node_cpu_seconds_total
    node_memory_MemAvailable_bytes
    container_cpu_usage_seconds_total
    kube_pod_status_phase
    DCGM_FI_DEV_GPU_UTIL

It is important that metric namesCommonly used for calculating P95 / P99.

Example:

    histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

### 8.4 Summary

Summary is also used for statistical quantiles.

It calculates quantiles on the client side.

Compared to Histogram:

- The quantiles of Summary are calculated by the client;
- It is not easy to aggregate across instances;
- It is less flexible than Histogram for global P95/P99 calculations;
- It is suitable for certain local statistical scenarios on the client side.

In production, Histogram is more commonly used for overall service latency statistics.

---

## IX. Basics of PromQL

PromQL is the query language for Prometheus.

It is used for:

- Querying metrics;
- Aggregating metrics;
- Calculating rates;
- Calculating percentages;
- Calculating quantiles;
- Creating dashboards;
- Writing alert rules;
- Writing recording rules.

### 9.1 Directly Querying Metrics

    node_memory_MemAvailable_bytes

### 9.2 Filtering by Label

    kube_pod_status_phase{namespace="default"}

    container_cpu_usage_seconds_total{namespace="prod", pod=~"api-.*"}

### 9.3 Regular Expression Matching

    pod=~"nginx-.*"

Does not match:

    pod!~"test-.*"

### 9.4 rate()

Used to calculate the rate per second of a Counter.

Example:

    rate(node_cpu_seconds_total[5m])

Commonly used for:

- CPU usage;
- Request rates;
- Network traffic rates;
- Disk I/O rates.

### 9.5 increase()

Used to calculate how much a Counter has increased over a period of time.

Example:

    increase(kube_pod_container_status_restarts_total[10m])

Commonly used for:

- The number of restarts in the past 10 minutes;
- The number of error requests in the past 5 minutes;
- The number of task failures in the past 1 hour.

### 9.6 sum by()

Aggregates by dimension.

Example:

    sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})

### 9.7 avg by()

Calculates the average by dimension.

Example:

    avg by (instance) (DCGM_FI_DEV_GPU_UTIL)

### 9.8 max by()

Determines the maximum value by dimension.

Example:

    max by (node) (DCGM_FI_DEV_GPU_TEMP)

### 9.9 topk()

Queries the top N values.

Example:

    topk(5, container_memory_usage_bytes)

### 9.10 histogram_quantile()

Calculates quantiles.

Example:

    histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))

This means calculating the P95 latency for the last 5 minutes, grouped by service.

---

## X. Sources of Kubernetes Monitoring Metrics

Different metrics in Kubernetes come from various components.

### 10.1 Node Exporter

The node-exporter runs on each node.

It collects metrics for Linux hosts:

- CPU;
- Memory;
- Disk;
- File system;
- Network;
- System load;
- Kernel;
- Time;
- Hardware information.

Typical metrics:

    node_cpu_seconds_total
    node_memory_MemAvailable_bytes
    node_filesystem_avail_bytes
    node_network_receive_bytes_total
    node_network_transmit_bytes_total
    node_load1
    node_uname_info

### 10.2 kube-state-metrics

kube-state-metrics collects information about the status of Kubernetes objects.

It does not collect resource usage data but rather collects object declarations and their current status.

For example:

- The number of Deployment replicas;
- Pod status;
- Node status;
- PVC status;
- DaemonSet status;
- ResourceQuota;
- HPA;
- Jobs;
- CronJobs.

Typical metrics:

    kube_pod_status_phase
    kube_pod_container_status_restarts_total
    kube_deployment_status_replicas_available
    kube_deployment_spec_replicas
    kube_node_status_condition
    kube_resourcequota
    kube_persistentvolumeclaim_status_phase

### 10.3 kubelet / cAdvisor

kubelet includes built-in cAdvisor capabilities for collecting container resource usage data.

Typical metrics:

    container_cpu_usage_seconds_total
    container_memory_usage_bytes
    container_network_receive_bytes_total
    container_network_transmit_bytes_total
    container_fs_usage_bytes

These metrics are used to analyze:

- Pod CPU usage;
- Pod memory usage;
- Container network performance;
- Container file system activity;
- Resource pressure on containers.

### 10.4 DCGM Exporter

The DCGM Exporter collects metrics for NVIDIA GPUs.

Typical metrics:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_used    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install prometheus prometheus-community/prometheus \
      --namespace monitoring \
      --version <CHART_VERSION>

Note:

    <CHART_VERSION> should be determined by using the `helm search repo` command.
    In a production environment, it is recommended to use a fixed version instead of installing without a specific version.

### 12.2 kube-prometheus-stack

The kube-prometheus-stack is more suitable for production monitoring in Kubernetes environments.

It typically includes:

- Prometheus Operator;
- Prometheus;
- AlertManager;
- Grafana;
- node-exporter;
- kube-state-metrics;
- PrometheusRule;
- ServiceMonitor;
- PodMonitor;
- Default Dashboard;
- Default alert rules.

Suitable for:

- Production Kubernetes environments;
- Those who want to use ServiceMonitor;
- Those who wish to manage PrometheusRules uniformly;
- Those looking to quickly obtain a complete monitoring stack;
- Those seeking to standardize the use of Prometheus, AlertManager, and Grafana.

Example installation:

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --version <CHART_VERSION>

Production recommendations:

- Use a `values.yaml` file to manage configuration settings.
- Fix the chart version and image versions.
- Avoid using default configurations directly in a production environment.

---

## Chapter Thirteen: Basic Verification of Prometheus

### 13.1 Viewing Pods

    kubectl get pods -n monitoring -o wide

### 13.2 Viewing Services

    kubectl get svc -n monitoring

### 13.3 Port Forwarding to Access Prometheus

If using a regular Prometheus chart:

    kubectl port-forward svc/prometheus-server 9090:80 -n monitoring

Access address:

    http://127.0.0.1:9090

If using the kube-prometheus-stack, the Service name may be different.

To find it:

    kubectl get svc -n monitoring | grep prometheus

Then perform:

    kubectl port-forward svc/<prometheus-service-name> 9090:9090 -n monitoring

### 13.4 Viewing Targets

On the Prometheus web page:

    Status
      ↓
    Targets

Pay special attention to:

    node-exporter;
    kube-state-metrics;
    kubelet;
    apiserver;
    dcgm-exporter;
    application metrics.

The status should be:

    UP

If it is DOWN, further investigation is needed regarding network settings, Services, ServiceMonitor, ports, and authentication configurations.

---

## Chapter Fourteen: Service Discovery and Targets

Prometheus can automatically discover targets through service discovery mechanisms.

Common methods include:

- static_configs;
- file_sdConfigs;
- kubernetes.sdconfigs;
- ServiceMonitor;
- PodMonitor;
- Probe;
- Consul SD;
- EC2 SD;
- DNS SD.

### 14.1 static_configs

Suitable for fixed targets.

Example:

    scrape_configs:
      - job_name: 'node-exporter'
        staticConfigs:
          - targets:
              - '10.0.0.21:9100'
              - '10.0.0.22:9100'

Disadvantage:

    Configuration needs to be manually updated when nodes change.

### 14.2 kubernetes_sd_configs

Suitable for native Kubernetes discovery.

Example:

    scrapeConfigs:
      - job_name: 'kubernetes-pods'
        kubernetes_sdconfigs:
          - role: pod

This is often used in conjunction with `relabel_configs` to filter the Pods that need to be scraped.

### 14.3 ServiceMonitor

ServiceMonitor is commonly used in Prometheus Operator scenarios.

Description of ServiceMonitor:

- Which Namespace to select;
- Which Services to include;
- Which port to scrape data from;
- What path to use for scraping;
- What interval to use for data collection.

Example:

    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: dcgm-exporter
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
    spec:
      namespaceSelector:
        matchNames:
          - gpu-operator
      selector:
        matchLabels:
          app.kubernetes.io/name: dcgm-exporter
      endpoints:
        - port: metrics
          path: /metrics
          interval: 30s

Note:

- `selector.matchLabels` must match the Service's labels.
- `Endpoints.port` must correspond to the Service's port name.
- Whether to include the `metadata.labels.release` depends### 20. Recording Rules

Recording Rules are used to pre-calculate complex PromQL expressions and save them as new time series data.

**Benefits:**
- Reduce the load on dashboards during queries;
- Ensure consistent指标 calculation methods;
- Improve query performance;
- Facilitate the reuse of alerts;
- Avoid duplicating the effort of writing complex PromQL queries.

### 20.1 Example: Recording Node CPU Usage

```yaml
groups:
  - name: node.rules
    rules:
      - record: node:cpu_usage:percent
        expr: |
          100 - (
            avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          )
```

After setting this rule, you can directly query `node:cpu_usage:percent` to get the desired data.

### 20.2 Example: Recording GPU Memory Usage

```yaml
groups:
  - name: gpu.rules
    rules:
      - record: gpu:memory_usage:percent
        expr: |
          DCGM_FI_DEV_FB_used
          /
          (DCGM_FI_DEVFB_USED + DCGM_FI_DEV_FB_FREE)
          * 100
```

### 20.3 Production Recommendations

Common indicators suitable for Recording Rules include:
- Node CPU usage;
- Node memory usage;
- Pod CPU usage;
- Pod memory usage;
- GPU memory usage;
- Service QPS;
- Service error rate;
- Service P95/P99 latency.- alert: HighCPUUsage
    expr: node:cpu_usage:percent > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "The CPU usage of this node is too high."
      description: "The CPU usage of node {{ $labels.instance }} has been above 90% for 5 minutes."

### 21.2 The role of 'for'

    for: 5m

This indicates that the alert will be triggered only if the condition persists for 5 minutes. This prevents false alarms caused by temporary fluctuations.

### 21.3 The role of 'labels'

Labels are used to route and categorize alerts.

Common labels include:

    severity: warning
    severity: critical
    team: sre
    service: api
    cluster: prod

### 21.4 The role of 'annotations'

Annotations provide additional details about an alert.

Common annotations include:

    summary
    description
    dashboard_url
    runbook_url

### 21.5 Principles for designing alerts

It is recommended to:

- Issue alerts only when there is a significant impact;
- Avoid triggering alerts for temporary fluctuations;
- Ensure that alerts are actionable;
- Provide context in the alert messages;
- Link alerts to dashboards and runbooks;
- Differentiate between warning and critical alerts;
- Prevent multiple alerts from being triggered simultaneously;
- Use AlertManager to manage and group alerts.

---

## 22. Examples of common alert rules

### 22.1 High Node CPU usage

    - alert: NodeHighCPUUsage
      expr: node:cpu_usage:percent > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "The CPU usage of this node is too high."
        description: "The CPU usage of node {{ $labels.instance }} has been above 90% for 5 minutes."

### 22.2 High Node memory usage

    - alert: NodeHighMemoryUsage
      expr: |
        (
          1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
        ) * 100 > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "The memory usage of this node is too high."
        description: "The memory usage of node {{ $labels.instance }} has been above 90% for 5 minutes."

### 22.3 Insufficient disk space

    - alert: NodeDiskSpaceLow
      expr: |
        (
          1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}
        ) * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "The disk space of this node is insufficient."
        description: "The usage of the filesystem {{ $labels.mountpoint }} on node {{ $labels.instance }} has exceeded 85%."

### 22.4 Excessive Pod restarts

    - alert: PodRestartTooOften
      expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "The Pod has been restarted too many times."
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been restarted more than 3 times in the last 10 minutes."

### 22.5 Long-standing Pod Pending status

    - alert: PodPendingTooLong
      expr: kube_pod_status_phase{phase="Pending"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "The Pod has been in a Pending state for too long."
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been Pending for more than 10 minutes."

### 22.6 High GPU temperature

    - alert: GPUHighTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "The GPU temperature is too high."
        description: "The temperature of GPU {{ $labels gpu }} on host {{ $labels.Hostname }} has exceeded 80°C."

### 22.7 GPU XID errors

    - alert: GPUXIDError
      expr: DCGM_FI_DEV_XID_errors > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "A GPU XID error has occurred."
        description: "GPU {{ $labels.gpu }} on host {{```markdown
Use remote storage solutions such as Thanos, Cortex, Mimir, or VictoriaMetrics.
---

## Section 25: High Availability of Prometheus

### 25.1 Issues with Single-Instance Prometheus

Risks of using a single-instance Prometheus:

- Monitoring interruptions due to Prometheus Pod failures;
- Data loss in case of node failures;
- Loss of historical data due to disk failures;
- Excessive query load affecting data collection;
- Potential window periods during upgrades.

### 25.2 Basic Approaches to Prometheus HA

Common methods include:

    Using two Prometheus instances to collect the same set of targets;
    Sending alerts from both instances to the same AlertManager group;
    The AlertManager handles duplicate alerts.

Illustration:

    Prometheus A ----\
                      ---> AlertManager HA
    Prometheus B ----/

Note:

    Multiple Prometheus replicas do not automatically share local TSDB data.
    HA primarily ensures the availability of data collection and alerting functions.
    Long-term global queries often require remote storage solutions like Thanos.

### 25.3 AlertManager HA

AlertManager supports cluster mode.

Benefits:

- Elimination of duplicate alerts;
- High availability due to multiple replicas;
- Prevention of redundant notifications;
- Continued alert delivery even in case of node failures.

### 25.4 Remote Storage

Common remote/storage solutions for long-term data retention include:

- Thanos;
- Cortex;
- Grafana Mimir;
- VictoriaMetrics;
- Using Prometheus's `remote_write` feature to transfer data to other systems.

Suitable for:

- Multi-cluster monitoring scenarios;
- Long-term historical data analysis;
- Global data visualization;
- Handling large quantities of metrics;
- Cross-regional data aggregation;
- Reducing local storage pressure on Prometheus.

---
## Section 26: Monitoring Prometheus Itself

Prometheus also requires regular monitoring.

Key metrics to track:

### 26.1 Target Retrieval Status

    `up`

Targets with retrieval failures:

    `up == 0`

### 26.2 Scrape Duration

    `scrape_duration_seconds`

If `scrape_duration` is close to `scrape_timeout`, it indicates slow target data collection.

### 26.3 Number of Samples Collected

    `scrape_samples_scraped`

This metric shows the number of samples collected in each scrape operation.

### 26.4 TSDB Head Series

    `prometheus_tsdb_head_series`

It indicates the current number of active time series. A sudden increase may indicate high baseline values.

### 26.5 Rule Calculation Time

    `prometheus_rule_group_duration_seconds`

High rule calculation times may suggest optimization is needed.

### 26.6 Prometheus Memory and CPU Usage

Check these metrics via Pod or container-specific indicators:

    `container_cpu_usage_seconds_total`
    `container_memory_working_set_bytes`

---
## Section 27: Troubleshooting Common Prometheus Issues

### 27.1 Targets Are Down

Symptoms:

    In Prometheus -> Status -> Targets, some targets are shown as `DOWN`.

Troubleshooting steps:

    1. Check if the target Pod is running.
    2. Verify if the Service exists.
    3. Ensure the Service selector matches the Pod.
    4. Confirm if the EndpointSlice has a backend.
    5. Check if the port names are correct.
    6. Verify if `/metrics` is accessible.
    7. Investigate if Network Policies are blocking access.
    8. Confirm if Prometheus has the necessary permissions.
    9. Check if the ServiceMonitor selector is correct.

Commands to use:

    `kubectl get pod -A | grep <target>`
    `kubectl get svc -A | grep <target>`
    `kubectl get endpoints -n <namespace>`
    `kubectl get endpointslice -n <namespace>`
    `kubectl describe servicemonitor <name> -n <namespace>`
    `kubectl logs <prometheus-pod> -n monitoring`

### 27.2 No Data Returned in Queries

Possible reasons:

- Incorrect metric name;
- Targets were not successfully scraped;
- Wrong time range selected;
- Labels did not match;
- Metric values have changed;
- The exporter is not exposing the required metric;
- The ServiceMonitor configuration is ineffective;
- Prometheus's relabeling rules are filtering out data.

Troubleshooting steps:

    First, try querying the metric name directly.
    Then gradually add labels to the query.
    Avoid writing complex PromQL queries from the start.

### 27.3 Slow Query Performance

Possible causes:

- Excessively wide time range;
- High-value metrics with large datasets;
- Overly complex regular expression-based labels;
- Too many Dashboard panels;
- Excessive complexity in PromQL aggregation expressions;
- Lack of Recording Rules;
- Insufficient Prometheus memory resources.

Solutions:

    Narrow down the time range;
    Optimize label configuration;
    Implement Recording Rules;
    Reduce    15s

    GPU Metrics:
        15s or 30s

    Non-critical Exporters:
        60s

Not all metrics should be set to 5s.

### 29.5 Label Specification

It is recommended to retain stable dimensions:

    cluster
    environment
    namespace
    pod
    container
    node
    service
    app
    team

Avoid high cardinality dimensions:

    request_id
    user_id
    order_id
    trace_id
    full_url
    exception_message

### 29.6 Alarm Specification

Each alarm should include:

    alertname
    severity
    summary
    description
    dashboard_url
    runbook_url
    team
    service

Alarms must be actionable.

It is not advisable to generate a large number of unattended noise alarms.

---

## Thirty, Recommendations for Combining Prometheus with GPU Monitoring

In GPU clusters, Prometheus should collect the following metrics:

    node-exporter:
        Node CPU/memory/disk/network

    kube-state-metrics:
        Status of Nodes/Pods/Deployments/ResourceQuotas

    kubelet/cAdvisor:
        Container resource usage

    dcgm-exporter:
        GPU utilization/vram/temperature/power/XID

    Application/metrics:
        Inference QPS/error rate/latency/model status

Only by combining these metrics can we determine:

    Whether low GPU utilization is due to off-peak business activity or data loading bottlenecks;
    Whether high vram usage is caused by model persistence or vram leaks;
    Whether Pod Pending is due to insufficient GPUs or taints/labels/quotas;
    Whether slow inference is because the GPU is fully loaded or due to slow CPU preprocessing/postprocessing;
    Whether slow training is due to multi-GPU communication issues or storage read/write delays.

---

## Thirty-One, Experimental Verification Process

### 31.1 Verifying Prometheus

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring
    kubectl port-forward svc/<prometheus-service> 9090:9090 -n monitoring

    Access:

    http://127.0.0.1:9090

### 31.2 Verifying Targets

Prometheus Web:

    Status
      ↓
    Targets

Check:

    node-exporter is UP
    kube-state-metrics is UP
    kubelet is UP
    dcgm-exporter is UP
    Application metrics are UP

### 31.3 Verifying Node Metrics

Query:

    node_cpu_seconds_total
    node_memory_MemAvailable_bytes
    node_filesystem_avail_bytes

### 31.4 Verifying Pod Metrics

Query:

    container_cpu_usage_seconds_total
    container_memory_working_set_bytes
    kube_pod_status_phase

### 31.5 Verifying GPU Metrics

Query:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_used
    DCGM_FI_DEV_GPU_TEMP

### 31.6 Verifying Alarms

Check Rules:

    Status
      ↓
    Rules

View Alerts:

    Alerts

Access AlertManager:

    kubectl port-forward svc/<alertmanager-service> 9093:9093 -n monitoring

    Access:

    http://127.0.0.1:9093

---

## Thirty-Two, Prometheus Operation and Maintenance Checklist

### 32.1 Before Deployment

    [ ] Decided on the deployment method: Prometheus chart or kube-prometheus-stack
    [ ] Selected the Helm chart version
    [ ] Prepared values.yaml
    [ ] Planned the monitoring namespace
    [ ] Planned PVCs
    [ ] Decided on retention settings
    [ ] Defined scrape_interval
    [ ] Planned AlertManager notification channels
    [ ] Designed Grafana dashboards
    [ ] Determined whether GPU metrics are needed
    [ ] Decided whether remote storage is required

### 32.2 After Deployment

    [ ] Prometheus Pod is running
    [ ] AlertManager Pod is running
    [ ] Grafana Pod is running
    [ ] node-exporter is running
    [ ] kube-state-metrics is running
    [ ] Prometheus Targets are up and functioning
    [ ] Basic PromQL queries work correctly
    [ ] PrometheusRule loading is successful
    [ ] Alarms are being sent to AlertManager
    [ ] Notification channels are available
    [ ] Dashboards display data

### 32.3 Daily Checks

    [ ] Prometheus's CPU/memory usage is normal
    [ ] There is sufficient space in Prometheus PVCs
    [ ] prometheus_tsdb_head_series shows no abnormal growth
    [ ] No large number of Targets are down
    [ ] AlertManager is functioning normally
    [ ]https://prometheus.io/docs/prometheus/latest/querying/basics/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- AlertManager:
  https://prometheus.io/docs/alerting/latest(alertmanager/

- Node Exporter:
  https://github.com/prometheus/node_exporter

- kube-state-metrics:
  https://github.com/kubernetes/kube-state-metrics

- Prometheus Operator:
  https://prometheus-operator.dev/

- kube-prometheus-stack Helm Chart:
  https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

- NVIDIA DCGM Exporter:
  https://github.com/NVIDIA/dcgm-exporter

- Grafana Documentation:
  https://grafana.com/docs/