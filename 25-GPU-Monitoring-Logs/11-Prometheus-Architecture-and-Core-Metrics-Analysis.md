# 11-Prometheus-Architecture and Core Metrics Analysis

## Document Overview

This document systematically organizes Prometheus's architecture, working principles, core components, metric model, PromQL basics, Kubernetes monitoring collection methods, common metrics, alerting rules, storage planning, and production environment considerations.

This document is not solely about "how to install Prometheus", but rather about understanding from an operations/SRE perspective why Prometheus has become the most commonly used monitoring core component in Kubernetes, cloud-native, GPU clusters, and microservices systems.

This document focuses on answering the following questions:

- What is Prometheus?
- Why does Prometheus adopt a Pull model?
- What responsibilities do Prometheus Server, Exporter, AlertManager, and Grafana each have?
- What is a time series in Prometheus?
- What are the differences between Counter, Gauge, Histogram, and Summary?
- Why are Labels important, and why do they easily cause high cardinality issues?
- Where do Node, Pod, Service, and GPU metrics in Kubernetes come from?
- What do node-exporter, kube-state-metrics, cAdvisor, and DCGM Exporter each collect?
- How should one get started with PromQL?
- How to query common metrics for CPU, memory, disk, network, Pods, and GPU?
- What are the differences between Recording Rule and Alert Rule?
- What scenarios are suitable for single-node deployment, HA, and remote storage in Prometheus?
- How to plan scrape_interval, retention, labels, alerts, and capacity in production environments.

This document is suitable to read before studying the following content:

- 12-Grafana-Dashboard Setup and Custom Monitoring
- 13-AlertManager-Alerting Strategies and Notification Implementation
- 14-K8S-Monitoring Practice-Node-Pod-Service Metric Collection and Troubleshooting
- 15-Loki-Log Collection and Query Practice
- 17-Log Alerting and Automated Response Practice
- 18-Monitoring-GPU-Log Integration Case-K8S-Pod Anomaly Detection and Report Generation

---

## Tags

#Prometheus #Kubernetes #Monitor #Indicators #PromQL #AlertManager #Grafana #NodeExporter #kube-state-metrics #DCGMExporter #SRE #Observation

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Basics/11-Prometheus-Architecture and Core Metrics Analysis.md

---

## One: Why Operations Engineers Must Understand Prometheus

In traditional operations, monitoring systems typically focus on:

- Whether hosts are alive;
- Whether CPU is too high;
- Whether memory is insufficient;
- Whether disk is full;
- Whether service ports are accessible;
- Whether application processes exist.

However, in Kubernetes, microservices, GPU clusters, and cloud-native platforms, monitoring targets have become more complex.

Now, we need to pay attention to:

- Node status;
- Pod status;
- Container resources;
- Deployment replicas;
- Service availability;
- Ingress traffic;
- Application QPS;
- Application error rate;
- P95 / P99 latency;
- GPU utilization;
- GPU memory;
- GPU temperature;
- Device Plugin status;
- Business custom metrics;
- Multi-cluster, multi-tenant, namespace dimension resource usage.

Prometheus's value lies in:

    Using a unified metric model to collect, store, query, and alert on the status of hosts, containers, Kubernetes, GPUs, middleware, and application services.

For operations/SREs, Prometheus is not just a monitoring tool, but the foundation of a production fault judgment system.

---

## Two: Prometheus's Core Positioning

Prometheus is an open-source monitoring and alerting system.

Its core capabilities include:

- Metric collection;
- Time series storage;
- Multi-dimensional label model;
- PromQL query;
- Alerting rule calculation;
- Service discovery;
- Exporter ecosystem;
- Integration with AlertManager;
- Integration with Grafana;
- Support for native Kubernetes monitoring scenarios.

Prometheus is most suitable for collecting and analyzing:

- Time series metrics;
- Resource utilization;
- Service status;
- Request volume;
- Error rate;
- Latency distribution;
- Queue length;
- Business metrics;
- Cluster health status.

Prometheus is not suitable for directly replacing:

- Log systems;
- Tracing systems;
- CMDB;
- Audit systems;
- Relational databases;
- Long-term large-scale historical data warehouses.

Prometheus focuses on:

    What is the value of a metric at a specific time point, and how it changes over time.

+-----------------------+
    |      Exporters        |
    |-----------------------|
    | node-exporter         |
    | kube-state-metrics    |
    | cAdvisor / kubelet    |
    | dcgm-exporter         |
    | blackbox-exporter     |
    | mysql-exporter        |
    | redis-exporter        |
    +-----------+-----------+
                |
                | /metrics
                v
    +-----------------------+
    |   Prometheus Server   |
    |-----------------------|
    | Service Discovery     |
    | Scrape Engine         |
    | TSDB Storage          |
    | PromQL Engine         |
    | Rule Evaluation       |
    +-----------+-----------+
                |
                | Alert
                v
    +-----------------------+
    |     AlertManager      |
    |-----------------------|
    | Grouping              |
    | Inhibition            |
    | Deduplication         |
    | Notification          |
    +-----------+-----------+
                |
                v
    +-----------------------+
    | Email / Webhook / IM  |
    | OnCall / DingTalk     |
    | WeCom / Slack         |
    +-----------------------+

                |
                v
    +-----------------------+
    |       Grafana         |
    |-----------------------|
    | Dashboard             |
    | Variables             |
    | Panels                |
    | Visualization         |
    +-----------------------+

---

## IV. Prometheus Core Components

### 4.1 Prometheus Server

Prometheus Server is the core component.

Main responsibilities:

- Periodically scrapes targets' `/metrics`;
- Stores time series data;
- Executes PromQL queries;
- Calculates Recording Rules;
- Calculates Alert Rules;
- Sends alerts to AlertManager;
- Provides Web UI and API.

Prometheus Server typically listens on:

    9090

Common access methods:

    kubectl port-forward svc/prometheus-server 9090:80 -n monitoring

or:

    http://<prometheus-ip>:9090

### 4.2 Scrape Target

Scrape Target is the target scraped by Prometheus.

Typical targets:

- node-exporter;
- kube-state-metrics;
- kubelet;
- cAdvisor;
- dcgm-exporter;
- blackbox-exporter;
- mysql-exporter;
- redis-exporter;
- self-developed application `/metrics` interface.

Each target typically exposes an HTTP interface:

    /metrics

Prometheus periodically accesses:

    http://<target>:<port>/metrics

### 4.3 Exporter

Exporter is responsible for converting the status of a system, service, or hardware into Prometheus metric format.

Common Exporters:

    node-exporter:
        Collects Linux host CPU, memory, disk, network, file system, etc. metrics.

    kube-state-metrics:
        Collects Kubernetes object status, such as Deployment, Pod, Node, PVC, ResourceQuota, etc.

    cAdvisor:
        Collects container CPU, memory, network, file system, etc. resource metrics.

    dcgm-exporter:
        Collects NVIDIA GPU utilization, memory, temperature, power consumption, XID, ECC, etc. metrics.

    blackbox-exporter:
        Probes HTTP, TCP, ICMP, DNS, etc. external availability.

    mysql-exporter:
        Collects MySQL status metrics.

    redis-exporter:
        Collects Redis status metrics.

### 4.4 AlertManager

AlertManager is responsible for handling alerts sent by Prometheus.

It does not collect metrics or determine if metrics are abnormal.

Prometheus is responsible for determining if alerts are triggered based on Alert Rules.

AlertManager is responsible for:

- Alert Grouping;
- Alert Deduplication;
- Alert Inhibition;
- Alert Silencing;
- Alert Routing;
- Notification Sending.

Common notification channels:

- Email;
- Webhook;
- Enterprise WeChat;
- DingTalk;
- Slack;
- PagerDuty;
- Self-developed OnCall system.

### 4.5 Grafana

Grafana is used for visualizing Prometheus metrics.

Grafana does not collect metrics.

Grafana queries data through the Prometheus data source and then displays it as:

- Line Chart;
- Table;
- Status Panel;
- Gauge;
- Heatmap;
- Stat;
- Dashboard;
- Alert Panel.

Common uses of Grafana:

- Node Dashboard;
- Pod Dashboard;
- Service Dashboard;
- GPU Dashboard;
- Middleware Dashboard;
- Application Business Dashboard;
- Multi-cluster Overview Dashboard.

### 4.6 Pushgateway

Pushgateway is used for short-lived tasks to actively push metrics.

Typical scenarios:

- CronJob;
- Batch processing tasks;
- Jobs with very short lifecycles;
- Metrics should be retained for some time after task completion.

But note:

    Pushgateway should not be abused.
    Long-running services should prefer exposing /metrics, allowing Prometheus to actively scrape.

It is not recommended to use Pushgateway as a general-purpose metrics entry point, as this would break Prometheus's Pull model and instance lifecycle semantics.

---

## FiveI don't know.Prometheus Pull Model

Prometheus defaults to using the Pull model.

That is:

    Prometheus actively pulls metrics from targets.

Process:

    Prometheus Server
      ↓
    Find Targets based on scrape_configs or Service Discovery
      ↓
    Periodically request /metrics
      ↓
    Parse metrics
      ↓
    Write to local TSDB

### 5.1 Advantages of the Pull Model

Advantages:

- Prometheus can actively determine if a target is alive;
- Targets do not need to know Prometheus's address;
- More suitable for Kubernetes dynamic service discovery;
- scrape_interval can be uniformly controlled;
- Easier to detect target down;
- Configuration is centralized on the Prometheus side;
- Easy to locate issues during brief service anomalies.

### 5.2 Disadvantages of the Pull Model

Disadvantages:

- Prometheus must be able to access targets;
- May be inconvenient across networks, NAT, or security boundaries;
- Not friendly for short-lived Jobs;
- A large number of targets in massive clusters can increase pressure.

### 5.3 Supplementary Role of Pushgateway

If short-lived Jobs end before being scraped by Prometheus, metrics may be lost.

In such cases, Pushgateway can be used.

However, Pushgateway is not recommended for long-running services.

Long-running services should:

    Expose /metrics
    Be scraped by Prometheus

---

## SixI don't know.Prometheus Data Model

Prometheus stores time series data.

A time series consists of the following components:

    Metric name + Label set + Timestamp + Sample value

Example:

    node_cpu_seconds_total{cpu="0", mode="idle", instance="10.0.0.21:9100", job="node-exporter"} 123456.78

Where:

    node_cpu_seconds_total:
        Metric name.

    cpu="0", mode="idle", instance="10.0.0.21:9100":
        Labels.

    123456.78:
        Current sample value.

    Timestamp:
        Recorded when Prometheus scraped.

---

## SevenI don't know.Metric Names and Labels

### 7.1 Metric Names

Metric names typically describe "the object being measured".

Examples:

    node_cpu_seconds_total
    node_memory_MemAvailable_bytes
    container_cpu_usage_seconds_total
    kube_pod_status_phase
    DCGM_FI_DEV_GPU_UTIL

Metric names should be as stable as possible.

### 7.2 Labels

Labels are used to describe metric dimensions.

Examples:

    instance
    job
    namespace
    pod
    container
    node
    device
    mode
    cpu
    gpu
    service
    endpoint

Labels support multi-dimensional queries.

Examples:

    Query for a specific namespace:
        kube_pod_status_phase{namespace="default"}

    Query for a specific Pod:
        container_cpu_usage_seconds_total{pod="nginx-xxx"}

    Query for a specific GPU:
        DCGM_FI_DEV_GPU_UTIL{gpu="0"}

### 7.3 Value of Labels

Labels allow Prometheus to flexibly query:

- Which node;
- Which Pod;
- Which container;
- Which Namespace;
- Which GPU;
- Which service;
- Which status;
- Which application;
- Which environment.

### 7.4 High Cardinality Issues with Labels

Labels can also bring serious problems.

High cardinality refers to an excessive number of label combinations, leading to an explosion in time series count.

Avoid using the following as labels:

- User ID;
- Order ID;
- Request ID;
- Trace ID;
- Random IP port;
- File path;
- SQL statement;
- Error detail text;
- Dynamic URL;
- A large number of unique business IDs.

Incorrect example:

    http_request_total{user_id="123456789", request_id="abc-xxxx"}

This approach causes rapid growth in time series count.

Consequences:

- Prometheus memory increases;
- Queries become slower;
- Disk usage surges;
- TSDB pressure increases;
- OOM risk rises;
- Remote write pressure increases;
- Dashboard lag.

Production recommendations:

    Use labels for stable dimensions.
    Do not put infinitely growing dynamic values into labels.
    High cardinality fields should enter logs or traces, not Prometheus labels.

---

## EightI don't know.Prometheus Metric Types

Prometheus has four common metric types:

- Counter;
- Gauge;
- Histogram;
- Summary.

### 8.1 Counter

Counter is a counter that only increases.

Suitable for representing cumulative counts.

Typical metrics: /think

http_requests_total  
node_cpu_seconds_total  
container_cpu_usage_seconds_total  
kube_pod_container_status_restarts_total  

Features:  

  - Only increases.  
  - May reset to zero after process restart.  
  - Typically use rate() or increase() when querying.  

Example:  

  rate(http_requests_total[5m])  

Represents the request rate per second over the last 5 minutes.  

Example:  

  increase(kube_pod_container_status_restarts_total[10m])  

Represents the increase in container restarts over the last 10 minutes.  

### 8.2 Gauge  

Gauge is an instantaneous value that can increase or decrease.  

Suitable for representing current status.  

Typical metrics:  

  - node_memory_MemAvailable_bytes  
  - node_filesystem_avail_bytes  
  - kube_deployment_status_replicas_available  
  - DCGM_FI_DEV_GPU_UTIL  
  - DCGM_FI_DEV_GPU_TEMP  
  - DCGM_FI_DEV_FB_USED  

Features:  

  - Can increase or decrease.  
  - Represents current value.  
  - Typically does not need rate().  

Example:  

  node_memory_MemAvailable_bytes  

### 8.3 Histogram  

Histogram is used for statistical distribution.  

Commonly used for request latency and response size.  

Typical metrics:  

  - http_request_duration_seconds_bucket  
  - http_request_duration_seconds_sum  
  - http_request_duration_seconds_count  

Histogram statistics by bucket.  

Commonly used for calculating P95 / P99.  

Example:  

  histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))  

### 8.4 Summary  

Summary is also used for statistical percentiles.  

It calculates percentiles on the client side.  

Compared to Histogram:  

- Summary percentiles are calculated on the client side;  
- Not easy to aggregate across instances;  
- Less flexible for global P95/P99;  
- Suitable for some client-side statistical scenarios.  

Histogram is more commonly used for service-wide latency statistics in production.  

---

## NineI don't know.PromQL Basics  

PromQL is the query language of Prometheus.  

It is used for:  

- Querying metrics;  
- Aggregating metrics;  
- Calculating rates;  
- Calculating ratios;  
- Calculating percentiles;  
- Building Dashboards;  
- Writing alert rules;  
- Writing Recording Rules.  

### 9.1 Direct Metric Query  

  node_memory_MemAvailable_bytes  

### 9.2 Filtering by Label  

  kube_pod_status_phase{namespace="default"}  

  container_cpu_usage_seconds_total{namespace="prod", pod=~"api-.*"}  

### 9.3 Regular Expression Matching  

  pod=~"nginx-.*"  

Not matching:  

  pod!~"test-.*"  

### 9.4 rate()  

Used to calculate the per-second rate of a Counter.  

Example:  

  rate(node_cpu_seconds_total[5m])  

Commonly used for:  

- CPU utilization;  
- Request rate;  
- Network traffic rate;  
- Disk I/O rate.  

### 9.5 increase()  

Used to calculate how much a Counter has increased over a period of time.  

Example:  

  increase(kube_pod_container_status_restarts_total[10m])  

Commonly used for:  

- Number of restarts in the last 10 minutes;  
- Number of failed requests in the last 5 minutes;  
- Number of failed tasks in the last hour.  

### 9.6 sum by()  

Aggregate by dimension.  

Example:  

  sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})  

### 9.7 avg by()  

Take the average by dimension.  

Example:  

  avg by (instance) (DCGM_FI_DEV_GPU_UTIL)  

### 9.8 max by()  

Take the maximum by dimension.  

Example:  

  max by (node) (DCGM_FI_DEV_GPU_TEMP)  

### 9.9 topk()  

Query the top N.  

Example:  

  topk(5, container_memory_usage_bytes)  

### 9.10 histogram_quantile()  

Calculate percentiles.  

Example:  

  histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))  

Represents the P95 latency over the last 5 minutes calculated by service.  

---

## TenI don't know.Kubernetes Monitoring Metric Sources  

In Kubernetes, different metrics come from different components.  

### 10.1 Node Exporter  

node-exporter runs on each node.  

Collects Linux host metrics:  

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

  - node_cpu_seconds_total  
  - node_memory_MemAvailable_bytes  
  - node_filesystem_avail_bytes  
  - node_network_receive_bytes_total  
  - node_network_transmit_bytes_total  
  - node_load1  
  - node_uname_info  

### 10.2 kube-state-metrics  

kube-state-metrics collects Kubernetes object status.  

It does not collect resource usage, but instead collects object declarations and status.  

Examples:  

- Deployment replica count;  
- Pod status;  
- Node status;  
- PVC status;  
- DaemonSet status;  
- ResourceQuota;  
- HPA;  
- Job;  
- CronJob.  

Typical metrics:

kube_pod_status_phase  
kube_pod_container_status_restarts_total  
kube_deployment_status_replicas_available  
kube_deployment_spec_replicas  
kube_node_status_condition  
kube_resourcequota  
kube_persistentvolumeclaim_status_phase  

### 10.3 kubelet / cAdvisor  

kubelet includes built-in cAdvisor capabilities for collecting container resource usage.  

Typical metrics:  

    container_cpu_usage_seconds_total  
    container_memory_usage_bytes  
    container_network_receive_bytes_total  
    container_network_transmit_bytes_total  
    container_fs_usage_bytes  

Used for analysis:  

- Pod CPU usage rate;  
- Pod memory usage;  
- Container network;  
- Container filesystem;  
- Container resource pressure.  

### 10.4 DCGM Exporter  

DCGM Exporter collects NVIDIA GPU metrics.  

Typical metrics:  

    DCGM_FI_DEV_GPU_UTIL  
    DCGM_FI_DEV_FB_USED  
    DCGM_FI_DEV_FB_FREE  
    DCGM_FI_DEV_GPU_TEMP  
    DCGM_FI_DEV_POWER_USAGE  
    DCGM_FI_DEV_XID_ERRORS  

Used for analysis:  

- GPU utilization;  
- GPU memory;  
- GPU temperature;  
- GPU power consumption;  
- XID errors;  
- ECC errors;  
- GPU health status.  

### 10.5 Application itself /metrics  

Applications can expose business metrics directly.  

Examples:  

    http_requests_total  
    http_request_duration_seconds_bucket  
    app_queue_length  
    model_inference_total  
    model_inference_latency_seconds_bucket  
    model_error_total  

Used for analysis:  

- QPS;  
- Error rate;  
- Latency;  
- Queue length;  
- Inference count;  
- Model loading status;  
- Business success rate.  

---

## ElevenI don't know.Kubernetes Monitoring Architecture Diagram  

Recommended architecture:  

    +---------------------+  
    |    Prometheus       |  
    | monitoring namespace|  
    +----------+----------+  
               |  
     -------------------------  
     |           |           |  
     v           v           v  
+----------+ +----------+ +----------------+  
| kubelet  | | node-    | | kube-state-    |  
| cAdvisor | | exporter | | metrics        |  
+----------+ +----------+ +----------------+  
     |  
     v  
+---------------------+  
| Pod / Container     |  
+---------------------+  

GPU node:  

    +-----------------------------+  
    | GPU Node                    |  
    |                             |  
    | node-exporter               |  
    | kubelet / cAdvisor          |  
    | dcgm-exporter               |  
    | nvidia-device-plugin        |  
    | GPU Pods                    |  
    +-----------------------------+  

Visualization and alerting:  

    Prometheus  
      ↓  
    Grafana  
      ↓  
    Dashboard  

    Prometheus  
      ↓  
    AlertManager  
      ↓  
    Email / Webhook / IM / OnCall  

---

## TwelveI don't know.Prometheus Deployment Methods in Kubernetes  

### 12.1 prometheus-community/prometheus  

This is a basic Helm Chart.  

Suitable for:  

- Learning;  
- Small-scale experiments;  
- Quick deployment of Prometheus Server;  
- Manual understanding of components.  

Example:  

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
    helm repo update  

    kubectl create namespace monitoring  

    helm install prometheus prometheus-community/prometheus \  
      --namespace monitoring \  
      --version <CHART_VERSION>  

Notes:  

    <CHART_VERSION> should be checked via helm search repo.  
    Production environments should fix the version and avoid installing without version.  

### 12.2 kube-prometheus-stack  

kube-prometheus-stack is more suitable for Kubernetes production monitoring.  

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
- Default alerting rules.  

Suitable for:

- Kubernetes production environment;
- Want to use ServiceMonitor;
- Want to centrally manage PrometheusRule;
- Want to quickly get a complete monitoring stack;
- Want to standardize Prometheus, AlertManager, and Grafana.

Installation example:

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --version <CHART_VERSION>

Production recommendations:

    Use values.yaml to manage configurations.
    Fix the chart version.
    Fix the image version.
    Do not directly deploy production with default configurations.

---

## ThirteenI don't know.Prometheus Basic Validation

### 13.1 Check Pods

    kubectl get pods -n monitoring -o wide

### 13.2 Check Services

    kubectl get svc -n monitoring

### 13.3 Port Forwarding to Access Prometheus

If using a regular prometheus chart:

    kubectl port-forward svc/prometheus-server 9090:80 -n monitoring

Access:

    http://127.0.0.1:9090

If using kube-prometheus-stack, the Service name may differ.

Check:

    kubectl get svc -n monitoring | grep prometheus

Then execute:

    kubectl port-forward svc/<prometheus-service-name> 9090:9090 -n monitoring

### 13.4 Check Targets

Prometheus Web interface:

    Status
      ↓
    Targets

Focus on:

    node-exporter
    kube-state-metrics
    kubelet
    apiserver
    dcgm-exporter
    application metrics

Status should be:

    UP

If it is DOWN, further investigation is needed for network, Service, ServiceMonitor, port, and authentication configurations.

---

## FourteenI don't know.Service Discovery and Targets

Prometheus can automatically discover targets through service discovery.

Common methods:

- static_configs;
- file_sd_configs;
- kubernetes_sd_configs;
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
        static_configs:
          - targets:
              - '10.0.0.21:9100'
              - '10.0.0.22:9100'

Drawbacks:

    Requires manual configuration changes when nodes change.

### 14.2 kubernetes_sd_configs

Suitable for native Kubernetes discovery.

Example:

    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod

Usually combined with relabel_configs to filter pods to scrape.

### 14.3 ServiceMonitor

ServiceMonitor is commonly used in Prometheus Operator scenarios.

ServiceMonitor description:

    Select which Namespace
    Select which Service
    Scrape which port
    Scrape which path
    Scrape interval

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

Notes:

    selector.matchLabels must match the Service's labels.
    endpoints.port must match the Service port name.
    Whether metadata.labels needs the release label depends on the Prometheus Operator's selector configuration.

### 14.4 PodMonitor

PodMonitor directly selects Pods instead of Services.

Suitable for:

- Some Pods without a Service;
- Wanting to scrape directly by Pod label;
- Special application metrics.

ServiceMonitor is more common in production because Services are more stable.

---

## FifteenI don't know.Common Node Metrics

### 15.1 CPU Usage

Original metric:

    node_cpu_seconds_total

Common PromQL for CPU usage:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

Note:

node_cpu_seconds_total is a Counter.  
Use rate() to calculate the growth rate per unit time.  
mode="idle" indicates idle CPU.  
100 - idle% gives the usage rate.

### 15.2 Memory Availability

Available memory:  
    node_memory_MemAvailable_bytes

Total memory:  
    node_memory_MemTotal_bytes

Memory usage rate:  
    (
      1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
    ) * 100

### 15.3 Filesystem Usage Rate

Available space:  
    node_filesystem_avail_bytes

Total space:  
    node_filesystem_size_bytes

Usage rate:  
    (
      1 - node_filesystem_avail_bytes / node_filesystem_size_bytes
    ) * 100

Recommended filters:  
    fstype!="tmpfs"  
    mountpoint!~"/run.*"

Example:  
    (
      1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}
    ) * 100

### 15.4 Network Receive Rate

    rate(node_network_receive_bytes_total[5m])

By node and network interface:  
    sum by (instance, device) (
      rate(node_network_receive_bytes_total[5m])
    )

### 15.5 Network Transmit Rate

    rate(node_network_transmit_bytes_total[5m])

### 15.6 System Load

    node_load1  
    node_load5  
    node_load15

Load should be evaluated in conjunction with CPU core count.  
Do not assume anomalies solely based on high load.

---

## Sixteen, Common Pod / Container Metrics

### 16.1 Pod CPU Usage Rate

Raw metric:  
    container_cpu_usage_seconds_total

Aggregated by Pod:  
    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
    )

Unit:  
    CPU cores

If the result is 0.5, it represents approximately 0.5 cores.

### 16.2 Pod Memory Usage

    sum by (namespace, pod) (
      container_memory_usage_bytes{container!="", image!=""}
    )

A metric closer to working set:  
    container_memory_working_set_bytes

### 16.3 Pod Restart Count

From kube-state-metrics:  
    kube_pod_container_status_restarts_total

Recent 10-minute restart increase:  
    increase(kube_pod_container_status_restarts_total[10m])

### 16.4 Pod Status

    kube_pod_status_phase

Query Pending Pods:  
    kube_pod_status_phase{phase="Pending"} == 1

Query Running Pods:  
    kube_pod_status_phase{phase="Running"} == 1

### 16.5 OOMKilled

Common kube-state-metrics metrics may include:  
    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

Query:  
    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

---

## Seventeen, Common Kubernetes Object Metrics

### 17.1 Deployment Desired Replicas

    kube_deployment_spec_replicas

### 17.2 Deployment Available Replicas

    kube_deployment_status_replicas_available

### 17.3 Deployment Replica Shortage

    kube_deployment_spec_replicas  
    -  
    kube_deployment_status_replicas_available

### 17.4 Node Ready Status

    kube_node_status_condition{condition="Ready", status="true"}

If it is 1, it indicates Ready.

### 17.5 PVC Status

    kube_persistentvolumeclaim_status_phase

Query Pending PVCs:  
    kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1

### 17.6 ResourceQuota

    kube_resourcequota

Used for analyzing Namespace quota usage.

---

## Eighteen, Common GPU Metrics

GPU metrics typically come from DCGM Exporter.

### 18.1 GPU Utilization

    DCGM_FI_DEV_GPU_UTIL

Average by node:  
    avg by (Hostname) (DCGM_FI_DEV_GPU_UTIL)

Average by GPU:  
    avg by (Hostname, gpu) (DCGM_FI_DEV_GPU_UTIL)

### 18.2 GPU Memory Usage

    DCGM_FI_DEV_FB_USED

### 18.3 GPU Memory Free

    DCGM_FI_DEV_FB_FREE

### 18.4 GPU Memory Usage Rate

    DCGM_FI_DEV_FB_USED  
    /  
    (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)  
    * 100

### 18.5 GPU Temperature

    DCGM_FI_DEV_GPU_TEMP

### 18.6 GPU Power Usage

    DCGM_FI_DEV_POWER_USAGE

### 18.7 GPU XID Errors

    DCGM_FI_DEV_XID_ERRORS

Note:

    XID errors need to be analyzed in combination with node dmesg / journalctl.
    Do not directly judge hardware damage based on Prometheus metrics alone.

Node Troubleshooting:

    dmesg | grep -i xid
    journalctl -k | grep -i xid

---

## NineteenI don't know.Common Service / Application Metrics

Prometheus itself does not directly know business QPS.

Applications need to expose metrics.

Typical HTTP service metrics:

    http_requests_total
    http_request_duration_seconds_bucket
    http_request_duration_seconds_count
    http_request_duration_seconds_sum

### 19.1 QPS

    sum by (service) (
      rate(http_requests_total[5m])
    )

### 19.2 Error Rate

If there is a status label:

    sum by (service) (
      rate(http_requests_total{status=~"5.."}[5m])
    )
    /
    sum by (service) (
      rate(http_requests_total[5m])
    )
    * 100

### 19.3 P95 Latency

    histogram_quantile(
      0.95,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket[5m])
      )
    )

### 19.4 P99 Latency

    histogram_quantile(
      0.99,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket[5m])
      )
    )

---

## TwentyI don't know.Recording Rules

Recording Rules are used to pre-calculate complex PromQL and save it as new time series.

Purpose:

- Reduce Dashboard query pressure;
- Standardize metric calculation;
- Improve query speed;
- Facilitate alert reuse;
- Avoid redundant writing of complex PromQL.

### 20.1 Example: Record Node CPU Usage

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

You can then directly query:

    node:cpu_usage:percent

### 20.2 Example: Record GPU Memory Usage

    groups:
      - name: gpu.rules
        rules:
          - record: gpu:memory_usage:percent
            expr: |
              DCGM_FI_DEV_FB_USED
              /
              (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)
              * 100

### 20.3 Production Recommendations

Common metrics for Recording Rules:

- Node CPU Usage;
- Node Memory Usage;
- Pod CPU Usage;
- Pod Memory Usage;
- GPU Memory Usage;
- Service QPS;
- Service Error Rate;
- Service P95/P99 Latency.

---

## Twenty-oneI don't know.Alert Rules

Alert Rules are used to trigger alerts based on PromQL.

Prometheus is responsible for determining if the alert condition is met, then sending the alert to AlertManager.

### 21.1 Alert Rule Structure

Basic structure:

    groups:
      - name: example.rules
        rules:
          - alert: HighCPUUsage
            expr: node:cpu_usage:percent > 90
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Node CPU Usage is Too High"
              description: "Node {{ $labels.instance }} CPU usage is above 90% for 5 minutes."

### 21.2 Purpose of 'for'

    for: 5m

Indicates that the condition must persist for 5 minutes before triggering an alert.

Do not trigger alerts due to transient fluctuations.

### 21.3 Purpose of Labels

Labels are used for alert routing and prioritization.

Common:

    severity: warning
    severity: critical
    team: sre
    service: api
    cluster: prod

### 21.4 Purpose of Annotations

Annotations are used to display alert details.

Common:

    summary
    description
    dashboard_url
    runbook_url

### 21.5 Alert Design Principles

Recommendations:

- Alert only if there is a clear impact;
- Do not alert for all transient fluctuations;
- Alerts must be actionable;
- Alerts should have context;
- Alerts should point to Dashboard and Runbook;
- Alerts should be categorized as warning / critical;
- Avoid duplicate alert storms;
- Combine with AlertManager suppression and grouping.

## 22. Common Alert Rule Examples

### 22.1 Node CPU High

    - alert: NodeHighCPUUsage
      expr: node:cpu_usage:percent > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node CPU Usage is Too High"
        description: "Node {{ $labels.instance }} CPU usage is above 90% for 5 minutes."

### 22.2 Node Memory High

    - alert: NodeHighMemoryUsage
      expr: |
        (
          1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
        ) * 100 > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node Memory Usage is Too High"
        description: "Node {{ $labels.instance }} memory usage is above 90%."

### 22.3 Low Disk Space

    - alert: NodeDiskSpaceLow
      expr: |
        (
          1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}
        ) * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Node Disk Space is Low"
        description: "Node {{ $labels.instance }} filesystem {{ $labels.mountpoint }} usage is above 85%."

### 22.4 Pod Restarts Too Often

    - alert: PodRestartTooOften
      expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod Restarts Too Often"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted more than 3 times in 10 minutes."

### 22.5 Pod Pending

    - alert: PodPendingTooLong
      expr: kube_pod_status_phase{phase="Pending"} == 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod Pending Too Long"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been Pending for more than 10 minutes."

### 22.6 GPU High Temperature

    - alert: GPUHighTemperature
      expr: DCGM_FI_DEV_GPU_TEMP > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "GPU Temperature is Too High"
        description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} temperature is above 80°C."

### 22.7 GPU XID Error

    - alert: GPUXIDError
      expr: DCGM_FI_DEV_XID_ERRORS > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "GPU XID Error Occurred"
        description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} reported XID error. Check dmesg and GPU health."

---

## 23. Prometheus Storage Mechanism

Prometheus uses local TSDB to store time series data.

### 23.1 Samples

Each scrape generates samples.

Samples include:

    Timestamp
    Metric name
    Label set
    Value

### 23.2 Block

Prometheus stores time series data in blocks.

The local directory is typically:

    /prometheus

Or the PVC mount path configured in Helm Chart.

### 23.3 Retention

Prometheus controls retention time or size via retention.

Common parameters:

    --storage.tsdb.retention.time=15d

Or:

    --storage.tsdb.retention.size=100GB

### 23.4 WAL

WAL is a write-ahead log to reduce data loss after crashes.

Prometheus recovers recent data through WAL after restart.

### 23.5 Production Storage Recommendations

Recommendations:

- Use persistent storage;
- Do not use emptyDir for production monitoring data;
- Plan disk space based on scrape_interval and metric count;
- Avoid disk bloat caused by high cardinality;
- Use remote storage for long-term historical data;
- Monitor Prometheus' own disk usage rate.

---

## 24. Prometheus Capacity Estimation Approach

Prometheus storage pressure is mainly influenced by the following factors:

- Target count;
- Number of metrics exposed by each target;
- Label cardinality;
- scrape_interval;
- retention;
- Whether to enable high cardinality application metrics;
- Whether to collect excessive container metrics;
- Whether to collect GPU/MIG multi-dimensional metrics;
- Whether to enable remote_write.

Rough understanding:

    Higher scrape frequency means more samples.
    More label combinations mean more time series.
    Longer retention means larger disk usage.
    More query dimensions mean higher memory pressure.

### 24.1 scrape_interval Impact

Example:

    scrape_interval: 15s

Fetches 4 times per minute.

If changed to:

    scrape_interval: 30s

Fetches 2 times per minute.

Sample volume is roughly halved.

### 24.2 Production Recommendations

Default recommendations:

    Basic metrics:
        scrape_interval: 30s

    Critical service metrics:
        scrape_interval: 15s

    Non-critical metrics:
        scrape_interval: 60s

    GPU metrics:
        15s or 30s, depending on scenario

    retention:
        7d / 15d / 30d, determined by storage capacity planning

Long-term historical data:

    Use remote storage solutions like Thanos, Cortex, Mimir, or VictoriaMetrics.

---

## Twenty-fiveI don't know.Prometheus High Availability

### 25.1 Issues with Single-Instance Prometheus

Single-instance Prometheus risks:

- Prometheus Pod failure causing monitoring interruption;
- Node failure leading to data loss;
- Disk failure causing historical data loss;
- Query pressure affecting collection;
- Upgrade window period.

### 25.2 Prometheus HA Basic Approach

Common approaches:

    Two Prometheus instances collect the same batch of targets.
    Both instances send alerts to the same AlertManager group.
    AlertManager handles deduplication.

Diagram:

    Prometheus A ----\
                      ---> AlertManager HA
    Prometheus B ----/

Note:

    Prometheus multi-replica does not automatically share local TSDB.
    HA mainly ensures collection and alert availability.
    Long-term global queries typically require remote storage or Thanos solutions.

### 25.3 AlertManager HA

AlertManager supports cluster mode.

Functions:

- Alert deduplication;
- Multi-replica high availability;
- Avoid duplicate notifications;
- Notifications still possible during node failure.

### 25.4 Remote Storage

Common remote storage/long-term storage solutions:

- Thanos;
- Cortex;
- Grafana Mimir;
- VictoriaMetrics;
- Prometheus remote_write to other systems.

Suitable for:

- Multi-cluster monitoring;
- Long-term historical queries;
- Global view;
- Massive metrics;
- Cross-regional aggregation;
- Reducing local Prometheus storage pressure.

---

## Twenty-sixI don't know.Prometheus Self-Monitoring

Prometheus also needs to be monitored.

Key metrics:

### 26.1 Target Scrape Status

    up

Query failed targets:

    up == 0

### 26.2 Scrape Duration

    scrape_duration_seconds

If scrape_duration approaches scrape_timeout, it indicates slow target scraping.

### 26.3 Sample Count

    scrape_samples_scraped

Used to view sample volume per scrape.

### 26.4 TSDB Head Series

    prometheus_tsdb_head_series

Represents current active time series count.

If it continuously surges, there may be a high cardinality issue.

### 26.5 Rule Calculation Duration

    prometheus_rule_group_duration_seconds

If rule calculation duration is too high, optimize the rules.

### 26.6 Prometheus Memory and CPU

Check via Pod or container metrics:

    container_cpu_usage_seconds_total
    container_memory_working_set_bytes

---

## Twenty-sevenI don't know.Prometheus Common Issue Troubleshooting

### 27.1 Targets Down

Symptoms:

    Prometheus -> Status -> Targets
    Some targets = DOWN

Troubleshoot:

    1. Is the target Pod running?
    2. Does the Service exist?
    3. Does the Service selector match the Pod?
    4. Are there backend endpoints in EndpointSlice?
    5. Is the port name correct?
    6. Is /metrics accessible?
    7. Is NetworkPolicy blocking?
    8. Does Prometheus have permissions?
    9. Is the ServiceMonitor selector correct?

Commands:

    kubectl get pod -A | grep <target>
    kubectl get svc -A | grep <target>
    kubectl get endpoints -n <namespace>
    kubectl get endpointslice -n <namespace>
    kubectl describe servicemonitor <name> -n <namespace>
    kubectl logs <prometheus-pod> -n monitoring

### 27.2 No Data in Queries

Possible causes:

- Metric name written incorrectly;
- Target not scraped;
- Time range mismatch;
- Label mismatch;
- Metric changed;
- Exporter not exposing the metric;
- ServiceMonitor not effective;
- Prometheus relabel filtered it out.

Troubleshoot:

    First query the metric name directly.
    Then gradually add labels.
    Don't write complex PromQL at first.

### 27.3 Slow Queries

Possible causes:

- Too large time range;
- High cardinality metrics;
- Heavy regular expression labels;
- Too many dashboard panels;
- Too complex PromQL aggregation;
- No Recording Rule;
- Prometheus memory insufficient.

Handling: /think

- Narrow the time range;
- Optimize Label;
- Use Recording Rule;
- Reduce high cardinality metrics;
- Split Dashboard;
- Add resources;
- Use remote storage or query layer.

### 27.4 Prometheus High Memory

Possible causes:

- Too many time series;
- High cardinality Label;
- scrape_interval too short;
- Too many targets;
- High query pressure;
- Too many Recording Rules;
- Dashboard refresh frequency too high.

Troubleshooting:

    prometheus_tsdb_head_series
    scrape_samples_scraped
    topk(20, count by (__name__)({__name__=~".+"}))

Resolution:

- Remove high cardinality metrics;
- Adjust scrape_interval;
- Optimize relabel;
- Limit application metrics;
- Add Prometheus resources;
- Shard or cluster deployment;
- Use remote storage.

### 27.5 Alerts Not Triggering

Troubleshooting:

- Does PromQL have results;
- Is the for duration not met;
- Is PrometheusRule loaded;
- Do labels match;
- Is Alert page showing Pending;
- Did AlertManager receive it;
- Does routing match;
- Is it silenced;
- Is it inhibited.

Commands:

    kubectl get prometheusrule -n monitoring
    kubectl describe prometheusrule <name> -n monitoring
    kubectl logs <prometheus-pod> -n monitoring
    kubectl logs <alertmanager-pod> -n monitoring

---

## Twenty-Eight, Prometheus and Log System Boundaries

Prometheus is responsible for metrics.

Loki / EFK is responsible for logs.

They cannot replace each other.

### 28.1 Prometheus is Suitable for Answering

- What is CPU usage;
- Is Pod Pending;
- Has error rate increased;
- Is GPU memory over 90%;
- What is service P95 latency;
- How many restarts in the last 10 minutes;
- Is a node Ready;
- Is Target UP.

### 28.2 Log System is Suitable for Answering

- What is the specific error stack;
- Which request failed;
- Which SQL failed;
- Why did the application exit;
- Where did CUDA OOM occur;
- Why did model loading fail;
- What error message did the service return.

### 28.3 Correct Usage

Metrics are used to discover issues:

    GPU memory suddenly increased
    Pod restart count increased
    Error rate increased
    P99 latency increased

Logs are used to locate causes:

    CUDA out of memory
    connection refused
    timeout
    model load failed
    permission denied
    image pull failed

Production troubleshooting should combine:

    Prometheus metrics
      +
    Loki / EFK logs
      +
    Kubernetes Events
      +
    kubectl describe
      +
    Node system logs

---

## Twenty-Nine, Prometheus Production Environment Design Recommendations

### 29.1 Namespace Recommendations

Recommend deploying uniformly in:

    monitoring

Common components:

    Prometheus
    AlertManager
    Grafana
    node-exporter
    kube-state-metrics
    blackbox-exporter
    dcgm-exporter
    prometheus-operator

### 29.2 Resource Recommendations

Prometheus needs explicit resource settings:

    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "8Gi"

Actual values should be adjusted according to cluster scale.

### 29.3 Storage Recommendations

Production must use PVC.

Do not use emptyDir to save Prometheus data.

Recommendations:

    retention: 15d
    PVC: Start at 100Gi, adjust according to metric volume
    StorageClass use stable storage
    Regularly monitor disk usage

### 29.4 scrape_interval Recommendations

Recommendations:

    Default:
        30s

    Critical services:
        15s

    GPU metrics:
        15s or 30s

    Non-critical exporters:
        60s

Do not set all metrics to 5s.

### 29.5 Label Standards

Recommend retaining stable dimensions:

    cluster
    environment
    namespace
    pod
    container
    node
    service
    app
    team

Avoid high cardinality:

    request_id
    user_id
    order_id
    trace_id
    full_url
    exception_message

### 29.6 Alert Standards

Each alert needs:

    alertname
    severity
    summary
    description
    dashboard_url
    runbook_url
    team
    service

Alerts must be actionable.

Do not generate large volumes of unhandled noise alerts.

---

## Thirty, Prometheus and GPU Monitoring Integration Recommendations

In GPU clusters, Prometheus needs to collect:

    node-exporter:
        Node CPU / Memory / Disk / Network

    kube-state-metrics:
        Node / Pod / Deployment / ResourceQuota status

    kubelet / cAdvisor:
        Container resource usage

    dcgm-exporter:
        GPU utilization / Memory / Temperature / Power / XID

    Application /metrics:
        Inference QPS / Error rate / Latency / Model status

Only by combining these metrics can you determine:

    Low GPU utilization is due to low business peak or data loading bottleneck;
    High memory usage is due to model residency or memory leak;
    Pod Pending is due to GPU shortage, or taint/label/quota;
    Slow inference is due to GPU full load, or slow CPU preprocessing/postprocessing;
    Slow training is due to multi-card communication issues, or slow storage read.

---

## 31. Experimental Verification Process

### 31.1 Verify Prometheus

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring
    kubectl port-forward svc/<prometheus-service> 9090:9090 -n monitoring

Access:

    http://127.0.0.1:9090

### 31.2 Verify Targets

Prometheus Web:

    Status
      ↓
    Targets

Check:

    node-exporter UP
    kube-state-metrics UP
    kubelet UP
    dcgm-exporter UP
    application metrics UP

### 31.3 Verify Node Metrics

Query:

    node_cpu_seconds_total
    node_memory_MemAvailable_bytes
    node_filesystem_avail_bytes

### 31.4 Verify Pod Metrics

Query:

    container_cpu_usage_seconds_total
    container_memory_working_set_bytes
    kube_pod_status_phase

### 31.5 Verify GPU Metrics

Query:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_GPU_TEMP

### 31.6 Verify Alerts

View Rules:

    Status
      ↓
    Rules

View Alerts:

    Alerts

View AlertManager:

    kubectl port-forward svc/<alertmanager-service> 9093:9093 -n monitoring

Access:

    http://127.0.0.1:9093

---

## 32. Prometheus Operations Checklist

### 32.1 Before Deployment

    [ ] Deployment method has been confirmed: prometheus chart or kube-prometheus-stack
    [ ] Helm chart version has been confirmed
    [ ] values.yaml has been prepared
    [ ] monitoring namespace has been planned
    [ ] PVC has been planned
    [ ] retention has been planned
    [ ] scrape_interval has been planned
    [ ] AlertManager notification channels have been planned
    [ ] Grafana Dashboard has been planned
    [ ] GPU metrics requirement has been confirmed
    [ ] Remote storage requirement has been confirmed

### 32.2 After Deployment

    [ ] Prometheus Pod Running
    [ ] AlertManager Pod Running
    [ ] Grafana Pod Running
    [ ] node-exporter Running
    [ ] kube-state-metrics Running
    [ ] Prometheus Target UP
    [ ] Basic PromQL queries are normal
    [ ] PrometheusRule loading is normal
    [ ] Alerts can reach AlertManager
    [ ] Notification channels are available
    [ ] Dashboard has data

### 32.3 Daily Inspection

    [ ] Prometheus itself CPU/memory is normal
    [ ] Prometheus PVC space is sufficient
    [ ] prometheus_tsdb_head_series has no abnormal growth
    [ ] No large number of Targets Down
    [ ] AlertManager is normal
    [ ] No abnormal alert storm
    [ ] Dashboard query speed is normal
    [ ] Recording Rules have no failures
    [ ] Critical alerts are reachable
    [ ] GPU/Node/Pod metrics are complete

---

## 33. Common Misconceptions

### 33.1 Misconception 1: Prometheus can save all historical data

Prometheus local TSDB is not suitable for infinite long-term storage.

Long-term data should use remote storage solutions.

### 33.2 Misconception 2: All metrics should be collected every 5 seconds

The more frequent the collection, the higher the pressure.

Not all metrics require high-frequency collection.

### 33.3 Misconception 3: More Labels are Better

Too many labels will cause high cardinality.

High cardinality is a common root cause of Prometheus performance issues.

### 33.4 Misconception 4: Having a Dashboard Equals Having Monitoring

Dashboard is only for display.

Complete monitoring requires:

- Alerts;
- Notifications;
- Runbook;
- On-call;
- Post-mortem;
- Capacity planning.

### 33.5 Misconception 5: Prometheus can replace logs

Prometheus handles metrics.

Log systems handle details.

Production troubleshooting must combine metrics, logs, and events.

### 33.6 Misconception 6: High GPU utilization should always trigger an alert

High GPU utilization in training tasks may be normal.

Whether to alert depends on business latency, error rate, temperature, power consumption, XID, and queue length.

---

## 34. Summary

Prometheus is the core component of cloud-native monitoring systems.

It periodically pulls metrics from Exporters and applications exposing `/metrics`, stores data as time series, and provides powerful query and alerting capabilities through PromQL.

Prometheus's core workflow is:

    Exporter / Application /metrics
      ↓
    Prometheus Scrape
      ↓
    TSDB
      ↓
    PromQL
      ↓
    Recording Rule
      ↓
    Alert Rule
      ↓
    AlertManager
      ↓
    Grafana / Notification channels

Common metrics sources in Kubernetes monitoring:

    node-exporter:
        Host CPU, memory, disk, network.

kube-state-metrics:
    Kubernetes object status.

kubelet / cAdvisor:
    Container resource usage.

dcgm-exporter:
    NVIDIA GPU metrics.

Application /metrics:
    QPS, error rate, latency, business status.

When using Prometheus in production environments, the following must be prioritized:

- Label design;
- High cardinality control;
- scrape_interval;
- retention;
- PVC storage;
- Recording Rule;
- Alert Rule;
- AlertManager routing;
- Dashboard performance;
- Prometheus self-monitoring;
- HA and remote storage;
- Metric and log correlation for troubleshooting.

Prometheus's goal is not to "collect as much as possible", but to establish a stable, queryable, alertable, and manageable metric system to support fault detection, performance analysis, capacity planning, and cost governance in production environments.

---

## Reference Documents

- Prometheus official documentation:
  https://prometheus.io/docs/

- Prometheus Configuration:
  https://prometheus.io/docs/prometheus/latest/configuration/configuration/

- Prometheus Querying:
  https://prometheus.io/docs/prometheus/latest/querying/basics/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- AlertManager:
  https://prometheus.io/docs/alerting/latest/alertmanager/

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

- Grafana documentation:
  https://grafana.com/docs/