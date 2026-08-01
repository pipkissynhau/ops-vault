# 12-Grafana-Dashboard Setup and Custom Monitoring

## Document Overview

This document systematically organizes Grafana usage methods in Kubernetes, Prometheus, GPU monitoring, and business observability, including data source configuration, Dashboard design, Panel type selection, variable design, PromQL panel writing, Node/Pod/GPU/business Dashboard design, Dashboard import/export, permission management, alert linkage, performance optimization, and production environment standards.

This document is not simply about "clicking a few times to create charts", but rather from an operations/SRE perspective to understand:

- Grafana's positioning in the monitoring architecture;
- The relationship between Prometheus and Grafana;
- How to hierarchically design Dashboards;
- How to select Panels;
- How variables make Dashboards adaptable to multi-cluster, multi-namespace, multi-node, and multi-business scenarios;
- Which metrics to focus on for Node, Pod, Service, and GPU Dashboards;
- How Grafana integrates with AlertManager / Grafana Alerting;
- How to avoid slow Dashboards, query duplication, excessive variables, and chaotic Panels;
- How to transform Grafana from a "charting tool" into a production monitoring entry point for troubleshooting, retrospective analysis, and governance.

This document is suitable for study after completing the following content:

- 11-Prometheus-Architecture and Core Metrics Analysis
- 08-GPU-Monitoring and Alert Integration
- 09-GPU-Fault Diagnosis Cases and Practical Exercises
- 10-GPU-Operations Experiment Environment Setup and Verification

---

## Tags

#Grafana #Prometheus #Kubernetes #Dashboard #Panel #PromQL #GpuSurveillance #DCGMExporter #AlertManager #SRE #Observation #It'sABattleOfLuck.

---

## Recommended Learning Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Foundation/12-Grafana-Dashboard Setup and Custom Monitoring.md

---

## One: Why Use Grafana

Prometheus handles metric collection, storage, and querying.

However, Prometheus's native Web UI is better suited for temporary queries and debugging, not for long-term production Dashboards.

Grafana's value lies in:

    Unified display of data from multiple data sources like Prometheus, Loki, Elasticsearch, MySQL, PostgreSQL, InfluxDB, and CloudWatch through Dashboards.

For operations/SREs, Grafana is not a "charting tool" but the entry point for production monitoring.

It should be able to answer:

- Is the cluster currently healthy?;
- Which nodes have high resource pressure?;
- Which Pods have abnormal restarts?;
- Which services have increased latency?;
- Which Namespaces consume the most resources?;
- Which GPUs are idle?;
- Which GPUs have exhausted VRAM?;
- Which GPUs have XID errors?;
- Whether a business's QPS, error rate, or latency is abnormal?;
- Which Dashboard to navigate to when an alert occurs to locate the issue.

Grafana's core goal is not "having many charts", but:

    To discover issues faster
    To locate faults faster
    To make metrics easier to understand
    To judge capacity trends more easily
    To standardize troubleshooting paths

---

## Two: Grafana's Position in the Observability Architecture

A complete observability architecture typically includes:

    Metrics: Metrics
    Logs: Logs
    Traces: Trace tracking
    Events: Events
    Alerts: Alerts

Grafana mainly handles:

    Visualization
    Multi-data source querying
    Dashboard management
    Variable filtering
    Alert display
    Data linkage
    Troubleshooting entry point

Typical architecture:

    +---------------------+
    | Kubernetes / Node   |
    | App / GPU / DB      |
    +----------+----------+
               |
               v
    +---------------------+
    | Exporters           |
    | node-exporter       |
    | kube-state-metrics  |
    | dcgm-exporter       |
    | app /metrics        |
    +----------+----------+
               |
               v
    +---------------------+
    | Prometheus          |
    | Metric collection / TSDB      |
    | PromQL / Rules      |
    +----------+----------+
               |
               v
    +---------------------+
    | Grafana             |
    | Dashboard / Panel   |
    | Variables / Alert   |
    +---------------------+

If integrating logs:

    App Logs / Node Logs
      ↓
    Promtail / Fluent Bit
      ↓
    Loki / Elasticsearch
      ↓
    Grafana Logs Panel

If integrating trace tracking:

    OpenTelemetry / Jaeger / Tempo
      ↓
    Grafana Trace Panel

---

## Three: Relationship Between Grafana and Prometheus

Prometheus handles:

- Metric collection;
- Time series storage;
- PromQL execution;
- Recording Rules calculation;
- Alert Rules calculation;
- Sending alerts to AlertManager.

Grafana handles:

- Configuring Prometheus data sources;
- Writing PromQL queries;
- Displaying query results as charts;
- Dynamically switching nodes, namespaces, Pods, and GPUs via variables;
- Organizing Dashboards;
- Displaying alert states;
- Optionally creating Grafana-managed alerts.

Relationship between the two:

    Prometheus is the metric backend.
    Grafana is the visualization and analysis frontend.

Do not misunderstand as:

    Grafana collects metrics itself.

Grafana does not collect metrics from node-exporter, kube-state-metrics, or DCGM Exporter.

These metrics must first enter Prometheus, then Grafana queries Prometheus.

## Four. Grafana Basic Concepts

### 4.1 Data Source

A Data Source is the origin of data queried by Grafana.

Common data sources:

    Prometheus
    Loki
    Elasticsearch
    InfluxDB
    MySQL
    PostgreSQL
    Tempo
    Graphite
    CloudWatch
    OpenSearch

In Kubernetes monitoring scenarios, the most commonly used are:

    Prometheus: Metrics
    Loki: Logs
    Tempo: Trace
    Elasticsearch: Logs or search
    MySQL/PostgreSQL: Business or report data

This article focuses on using the Prometheus data source.

### 4.2 Dashboard

A Dashboard is a collection of Panels.

A Dashboard typically corresponds to a theme, for example:

    Kubernetes Cluster Overview
    Node Resource Dashboard
    Pod Resource Dashboard
    GPU Resource Dashboard
    Business Service Dashboard
    Database Dashboard
    Middleware Dashboard
    Alert Overview Dashboard

A Dashboard should be designed around "troubleshooting issues", not just randomly stacking charts.

### 4.3 Panel

A Panel is a chart or display component in a Dashboard.

Each Panel typically includes:

- Query;
- Visualization type;
- Title;
- Legend;
- Unit;
- Threshold;
- Color;
- Transform;
- Tooltip;
- Drilldown link.

Common Panel types:

    Time series
    Stat
    Gauge
    Bar gauge
    Table
    Heatmap
    Pie chart
    State timeline
    Status history
    Logs
    Alert list
    Text

### 4.4 Query

A Query is the statement used by a Panel to retrieve data.

In the Prometheus data source, a Query is PromQL.

Example:

    node_cpu_seconds_total

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

### 4.5 Variable

A Variable is a variable in a Dashboard.

Used to enable dynamic selection for a Dashboard:

- Cluster;
- Namespace;
- Node;
- Pod;
- Container;
- Service;
- GPU;
- Application;
- Environment;
- Time range.

Example:

    $namespace
    $node
    $pod
    $gpu

Variables allow a Dashboard to adapt to multiple objects, rather than duplicating a Dashboard for each node or application.

### 4.6 Transformation

A Transformation is a secondary processing of query results.

Common uses:

- Rename fields;
- Merge multiple query results;
- Join two datasets;
- Filter fields;
- Calculate new fields;
- Sort by field;
- Convert time series into tables;
- Adjust display structure.

Transformations are suitable for presentation layer processing.

But do not put too complex business calculations into Grafana.

Complex calculations should prioritize:

    PromQL
    Recording Rule
    Aggregation on the data source side
    Application-side metric design

---

## Five. Grafana Deployment Methods

### 5.1 kube-prometheus-stack Includes Grafana

In Kubernetes monitoring, the most common method is to install via kube-prometheus-stack.

It typically includes:

- Prometheus Operator;
- Prometheus;
- AlertManager;
- Grafana;
- node-exporter;
- kube-state-metrics;
- Default Dashboard;
- Default PrometheusRule;
- ServiceMonitor support.

Installation example:

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --version <CHART_VERSION>

Production recommendations:

    Use values.yaml.
    Fix chart version.
    Do not install without version.
    Do not directly rely on default password.
    Configure persistent storage.
    Configure Ingress or internal access entry.
    Configure RBAC and permissions.

### 5.2 Helm Install Grafana Separately

If you already have Prometheus and want to install Grafana separately, you can use the Grafana Helm Chart.

Example:

    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install grafana grafana/grafana \
      --namespace monitoring \
      --version <CHART_VERSION>

Check:

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring

### 5.3 Docker Method

Suitable for local experiments:

    docker run -d \
      --name grafana \
      -p 3000:3000 \
      grafana/grafana:<version>

Not recommended for long-term use in production without persistent Docker.

### 5.4 Binary or Package Installation

Suitable for traditional host deployment:

    Ubuntu / Debian
    RHEL / Rocky Linux
    systemd management
    Nginx reverse proxy

Production recommendations: /think

Kubernetes scenarios should prioritize Helm.
Traditional VM environments can use package managers or binary deployment.
Maintain consistent configuration management across different environments.

---

## Six. Accessing Grafana

### 6.1 View Service

    kubectl get svc -n monitoring | grep grafana

### 6.2 Port Forwarding Access

    kubectl port-forward svc/<grafana-service-name> 3000:80 -n monitoring

Access:

    http://127.0.0.1:3000

### 6.3 Retrieve Default Password

If using Helm Chart, the default admin password is typically stored in a Secret.

Example:

    kubectl get secret -n monitoring

View specific Secret:

    kubectl get secret <grafana-secret-name> -n monitoring -o yaml

Common method:

    kubectl get secret <grafana-secret-name> -n monitoring \
      -o jsonpath="{.data.admin-password}" | base64 -d

Note:

    Change the default password in production environments.
    Do not include the admin password in public documentation.
    Avoid using weak passwords.
    Recommend integrating with a unified identity authentication system.

### 6.4 Expose via Ingress

In production environments, expose internal access via Ingress.

Example structure:

    grafana.example.com
      ↓
    Ingress
      ↓
    grafana Service
      ↓
    grafana Pod

Note:

    HTTPS must be enabled.
    Configure authentication.
    Restrict access sources.
    Do not expose Grafana management interface publicly.

---

## Seven. Configuring Prometheus Data Source

### 7.1 UI Configuration Method

Grafana page:

    Connections
      ↓
    Data sources
      ↓
    Add data source
      ↓
    Prometheus

Configure URL:

    http://prometheus-operated.monitoring.svc:9090

Or:

    http://prometheus-server.monitoring.svc

The specific address depends on the Prometheus Service name.

Check Service:

    kubectl get svc -n monitoring | grep prometheus

Click:

    Save & test

If successful, it will prompt that the data source is available.

### 7.2 Common Prometheus Service Addresses

In kube-prometheus-stack:

    http://kube-prometheus-stack-prometheus.monitoring.svc:9090

Or:

    http://prometheus-operated.monitoring.svc:9090

In a standard prometheus chart:

    http://prometheus-server.monitoring.svc

Actual addresses depend on cluster output.

### 7.3 Pre-configure Data Source via values.yaml

In production, recommend using provisioning to manage data sources instead of manual UI configuration.

Example:

    datasources:
      datasources.yaml:
        apiVersion: 1
        datasources:
          - name: Prometheus
            type: prometheus
            access: proxy
            url: http://prometheus-operated.monitoring.svc:9090
            isDefault: true

Advantages:

- Version control;
- Repeatable deployment;
- Suitable for GitOps;
- Avoid manual configuration drift;
- Consistent across environments.

### 7.4 Troubleshooting Failed Data Source Test

If "Save & test" fails, check:

    kubectl get svc -n monitoring | grep prometheus
    kubectl get endpoints -n monitoring | grep prometheus
    kubectl get pod -n monitoring | grep prometheus

Enter Grafana Pod to test:

    kubectl exec -it <grafana-pod> -n monitoring -- sh

Test access:

    wget -qO- http://<prometheus-service>:9090/-/ready

Or:

    curl http://<prometheus-service>:9090/-/ready

Common causes:

- Incorrect Prometheus Service name;
- Incorrect Namespace;
- Incorrect port;
- NetworkPolicy blocking;
- Prometheus Pod not Ready;
- Grafana Pod DNS resolution failure;
- Data source URL uses external address but is unreachable from Pod.

---

## Eight. Dashboard Design Principles

### 8.1 Avoid Overloading with Graphs

A dashboard is not better with more graphs.

A good dashboard should answer specific questions.

Examples:

    Is this cluster currently healthy?
    Which node has the highest resource pressure?
    Which Namespace uses the most resources?
    Which Pods are abnormally restarting?
    Which GPUs are idle?
    Which GPUs have excessive temperatures?
    Which service has increased latency?
    Where should we start troubleshooting this alert?

### 8.2 Design by Troubleshooting Path

Recommended order:

    Overview
      ↓
    Cluster-level
      ↓
    Node-level
      ↓
    Namespace-level
      ↓
    Pod-level
      ↓
    Container-level
      ↓
    Business-level
      ↓
    GPU / Middleware-specific-level

### 8.3 Dashboard Layering

Recommend at least dividing into:

    1. Cluster Overview Dashboard
    2. Node Resource Dashboard
    3. Pod / Namespace Dashboard
    4. Service / Ingress Dashboard
    5. GPU Dashboard
    6. Application Business Dashboard
    7. Middleware Dashboard
    8. Alert Overview Dashboard
    9. Capacity & Cost Dashboard

### 8.4 Control Theme per Dashboard

Do not place all content in a single dashboard.

Incorrect approach:

    A single dashboard contains CPU, memory, Pods, GPUs, MySQL, Redis, business QPS, logs, alerts, disks, and network.

Problem:

- Slow loading;
- Heavy queries;
- Messy logic;
- Unable to find key points during troubleshooting;
- Too many variables;
- Difficult to maintain panels.

Correct approach:

    One Dashboard solves one problem.
    Navigate to other Dashboards via links.

### 8.5 Panel titles must be clear

Not recommended:

    CPU
    Memory
    GPU
    Error

Recommended:

    Node CPU Usage
    Node Memory Usage
    Pod CPU Usage Top 10
    GPU VRAM Usage
    Service 5xx Error Rate
    GPU XID Error

Titles should immediately inform on-call staff what the chart represents.

---

## Nine. Panel Type Selection

### 9.1 Time series

Suitable for metrics changing over time.

Common scenarios:

- CPU usage;
- Memory usage;
- GPU utilization;
- VRAM usage;
- QPS;
- Error rate;
- P95 latency;
- Network traffic;
- Disk I/O.

### 9.2 Stat

Suitable for single current values.

Common scenarios:

- Current number of nodes;
- Current number of Pods;
- Current total GPU count;
- Current number of alerts;
- Current QPS;
- Current error rate;
- Current maximum CPU usage.

### 9.3 Gauge

Suitable for percentage or capacity ratios.

Common scenarios:

- CPU usage;
- Memory usage;
- Disk usage;
- GPU VRAM usage;
- GPU temperature.

### 9.4 Bar gauge

Suitable for horizontal comparison of multiple objects.

Common scenarios:

- CPU usage per node;
- Memory usage per node;
- GPU utilization per GPU;
- Resource usage per Namespace;
- CPU usage Top for Pods.

### 9.5 Table

Suitable for detailed data.

Common scenarios:

- Pod restarts Top;
- Namespace resource usage;
- GPU allocation details;
- Node status list;
- PVC usage;
- Alert list;
- Service error rate ranking.

### 9.6 Heatmap

Suitable for distribution visualization.

Common scenarios:

- HTTP request latency distribution;
- Histogram metrics;
- Request size distribution;
- Batch processing duration distribution.

### 9.7 State timeline

Suitable for state changes.

Common scenarios:

- Pod status changes;
- Node Ready status;
- Service availability;
- Alert status;
- GPU node maintenance window.

### 9.8 Logs Panel

Suitable for log display.

Requires Loki or Elasticsearch data source.

Common scenarios:

- Error logs of a specific Pod;
- Logs of a specific service;
- CUDA OOM logs;
- XID-related system logs;
- Business exception stack traces.

### 9.9 Alert list

Suitable for current alert status.

Common scenarios:

- Current firing alerts;
- Current pending alerts;
- Key business alerts;
- GPU alert overview.

---

## Ten. Dashboard Variable Design

Variables are key to Grafana Dashboards.

Dashboards without variables quickly become fixed-object dashboards, hard to reuse.

### 10.1 Common variables

Recommend at least planning for Kubernetes monitoring Dashboards:

    $cluster
    $namespace
    $node
    $pod
    $container
    $deployment
    $service
    $gpu
    $app
    $job

For GPU Dashboards:

    $node
    $gpu
    $namespace
    $pod
    $gpu_model

For business Dashboards:

    $service
    $namespace
    $app
    $status
    $method

### 10.2 Namespace variables

Prometheus query example:

    label_values(kube_pod_info, namespace)

Used for filtering:

    kube_pod_status_phase{namespace="$namespace"}

### 10.3 Node variables

Query:

    label_values(kube_node_info, node)

Or based on node-exporter:

    label_values(node_uname_info, instance)

Used for filtering:

    node_memory_MemAvailable_bytes{instance="$node"}

Note:

    Node labels in different metrics may be called instance, node, or Hostname.
    Need to unify processing based on actual metrics.

### 10.4 Pod variables

Query:

    label_values(kube_pod_info{namespace="$namespace"}, pod)

Used for filtering:

    container_memory_working_set_bytes{namespace="$namespace", pod="$pod"}

### 10.5 GPU variables

Query:

    label_values(DCGM_FI_DEV_GPU_UTIL{Hostname="$node"}, gpu)

Or:

    label_values(DCGM_FI_DEV_GPU_UTIL, gpu)

Actual labels depend on DCGM Exporter output.

### 10.6 Multi-value

Multi-value allows selecting multiple objects at once.

Example: selecting multiple Namespaces.

Use regex in PromQL:

    namespace=~"$namespace"

Avoid writing:

    namespace="$namespace"

When variables support multi-selection, typically use:

    =~

### 10.7 Include All

Include All selects all options.

If enabled, PromQL generally writes:

    namespace=~"$namespace"

Variable All value can be set to:

    .*

### 10.8 Variable refresh strategy

Common refresh methods:

    On dashboard load
    On time range change

Recommendations:

- Namespace and Node variables can refresh on Dashboard load;
- Pod variables can refresh based on namespace changes;
- Avoid frequent refresh for all variables;
- Too many variable queries will slow down the Dashboard.

---

## Eleven. PromQL Writing Standards in Grafana

### 11.1 Use $__rate_interval

When writing rate / increase in Grafana, it is recommended to use:

    $__rate_interval

Example:

    rate(container_cpu_usage_seconds_total[$__rate_interval])

Compared to fixed intervals:

    rate(container_cpu_usage_seconds_total[5m])

`$__rate_interval` can automatically adjust according to Dashboard time range and sampling interval, making it more suitable for charts with variable time ranges.

### 11.2 Legend Naming

Do not directly display all labels in the Legend.

Not recommended:

    {{instance}} {{job}} {{pod}} {{container}} {{namespace}} {{endpoint}}

Recommended:

    {{namespace}}/{{pod}}
    {{node}}
    GPU {{gpu}}
    {{service}} {{status}}

### 11.3 Unit Settings

Common units:

    CPU Usage:
        percent

    Memory:
        bytes

    QPS:
        requests/sec

    Latency:
        seconds or milliseconds

    GPU Temperature:
        Celsius

    GPU Power Consumption:
        Watt

    GPU Memory:
        bytes or MiB/GiB

### 11.4 Threshold Settings

It is recommended to set threshold colors for critical metrics.

For example CPU:

    70% warning
    90% critical

Memory:

    80% warning
    90% critical

GPU Temperature:

    80°C warning
    90°C critical

Disk:

    85% warning
    95% critical

### 11.5 Do Not Stack Complex PromQL in Grafana

Complex PromQL can lead to:

- Slow Dashboard loading;
- High Prometheus query pressure;
- Difficult-to-maintain panels;
- Inconsistent alerting and dashboard definitions.

Complex calculations should beDown. to:

    Recording Rule

Example:

    node:cpu_usage:percent
    pod:cpu_usage:cores
    gpu:memory_usage:percent
    service:http_error_rate:percent

Querying Recording Rules in Grafana is clearer.

---

## TwelveI don't know.Cluster Overview Dashboard Design

### 12.1 Objectives

The Cluster Overview Dashboard should answer:

    Is the current Kubernetes cluster healthy?
    Are there any node anomalies?
    Are there many Pod anomalies?
    Is there resource pressure?
    Are there any alerts?
    Is the GPU cluster normal?

### 12.2 Recommended Panels

Stat Panel:

    Total Node Count
    Ready Node Count
    NotReady Node Count
    Total Pod Count
    Running Pod Count
    Pending Pod Count
    Failed Pod Count
    Current firing alert count
    Total GPU Count
    Average GPU Utilization

Time series Panel:

    Cluster CPU Usage Trend
    Cluster Memory Usage Trend
    Pod Count Trend
    Pending Pod Trend
    GPU Utilization Trend
    GPU Memory Usage Trend

Table Panel:

    NotReady Node List
    Pending Pod List
    Restart Top 10 Pods
    High CPU Pod Top 10
    High Memory Pod Top 10
    GPU High Temperature List

### 12.3 Common PromQL

Node Ready:

    kube_node_status_condition{condition="Ready", status="true"}

NotReady Node:

    kube_node_status_condition{condition="Ready", status="true"} == 0

Pod Pending:

    kube_pod_status_phase{phase="Pending"} == 1

Pod Failed:

    kube_pod_status_phase{phase="Failed"} == 1

Pod Restart Top:

    topk(10, increase(kube_pod_container_status_restarts_total[$__rate_interval]))

GPU Average Utilization:

    avg(DCGM_FI_DEV_GPU_UTIL)

---

## ThirteenI don't know.Node Dashboard Design

### 13.1 Objectives

The Node Dashboard should answer:

    Are CPU, memory, disk, and network on a node abnormal?
    Is there resource bottleneck on the node?
    Are there abnormal Pods on the node?
    Are there temperature, power, or XID anomalies on GPU nodes?

### 13.2 Recommended Variables

    $node

Variable query:

    label_values(kube_node_info, node)

Or:

    label_values(node_uname_info, instance)

### 13.3 Recommended Panels

Time series:

    CPU Usage
    Memory Usage
    Disk Usage
    Network Receive Rate
    Network Send Rate
    Load Average
    Container Count
    Pod Count

Table:

    Pod List on This Node
    High CPU Pod on This Node
    High Memory Pod on This Node
    Restarted Pod on This Node
    Abnormal Events on This Node

Additional panels for GPU nodes:

    GPU Utilization
    GPU Memory Usage
    GPU Temperature
    GPU Power Consumption
    GPU XID Error

### 13.4 CPU Usage PromQL

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle", instance="$node"}[$__rate_interval])
      ) * 100
    )

### 13.5 Memory Usage PromQL

(1 - node_memory_MemAvailable_bytes{instance="$node"}
/ node_memory_MemTotal_bytes{instance="$node"})
* 100

### 13.6 Disk Usage PromQL

(1 - node_filesystem_avail_bytes{instance="$node", fstype!="tmpfs"}
/ node_filesystem_size_bytes{instance="$node", fstype!="tmpfs"})
* 100

### 13.7 Network Receive Rate

rate(node_network_receive_bytes_total{instance="$node", device!~"lo"}[$__rate_interval])

### 13.8 Network Transmit Rate

rate(node_network_transmit_bytes_total{instance="$node", device!~"lo"}[$__rate_interval])

---

## FourteenI don't know.Pod / Namespace Dashboard Design

### 14.1 Objectives

Pod / Namespace Dashboard should answer:

    Which Namespace consumes the most resources?
    Which Pods have high CPU usage?
    Which Pods have high memory usage?
    Which Pods have frequent restarts?
    Which Pods are in Pending state?
    Which Pods were OOMKilled?

### 14.2 Recommended Variables

    $namespace
    $pod
    $container

Namespace Query:

    label_values(kube_pod_info, namespace)

Pod Query:

    label_values(kube_pod_info{namespace="$namespace"}, pod)

Container Query:

    label_values(kube_pod_container_info{namespace="$namespace", pod="$pod"}, container)

### 14.3 Namespace CPU Usage

sum by (namespace) (
  rate(container_cpu_usage_seconds_total{container!="", image!="", namespace=~"$namespace"}[$__rate_interval])
)

### 14.4 Namespace Memory Usage

sum by (namespace) (
  container_memory_working_set_bytes{container!="", image!="", namespace=~"$namespace"}
)

### 14.5 Top Pod CPU Usage

topk(10,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="", image!="", namespace=~"$namespace"}[$__rate_interval])
  )
)

### 14.6 Top Pod Memory Usage

topk(10,
  sum by (namespace, pod) (
    container_memory_working_set_bytes{container!="", image!="", namespace=~"$namespace"}
  )
)

### 14.7 Top Pod Restarts

topk(10,
  increase(kube_pod_container_status_restarts_total{namespace=~"$namespace"}[$__rate_interval])
)

### 14.8 OOMKilled Query

kube_pod_container_status_last_terminated_reason{namespace=~"$namespace", reason="OOMKilled"} == 1

---

## FifteenI don't know.GPU Dashboard Design

### 15.1 Objectives

GPU Dashboard should answer:

    Is the overall GPU health status good?
    Which GPUs are in use?
    Which GPUs are idle?
    Which GPUs have high memory usage?
    Which GPUs have high temperatures?
    Which GPUs have XID errors?
    Which Namespaces / Pods are using GPUs?
    Are there cases where GPUs are occupied but utilization is 0?

### 15.2 Recommended Variables

    $node
    $gpu
    $namespace
    $pod

Node Variable:

    label_values(DCGM_FI_DEV_GPU_UTIL, Hostname)

GPU Variable:

    label_values(DCGM_FI_DEV_GPU_UTIL{Hostname="$node"}, gpu)

Namespace Variable:

    label_values(DCGM_FI_DEV_GPU_UTIL, namespace)

Pod Variable:

    label_values(DCGM_FI_DEV_GPU_UTIL{namespace="$namespace"}, pod)

Note:

    DCGM Exporter's labels may vary depending on version and deployment method.
    Some labels are called Hostname, others instance, some have pod/namespace, others don't.
    Must first query the raw metrics in Prometheus to confirm label structure.

### 15.3 GPU Overview Panel

Stat:

    Total GPU count
    Average GPU utilization
    Average GPU memory usage
    Number of high-temperature GPUs
    XID error count
    DCGM Exporter Target status

### 15.4 GPU Utilization

DCGM_FI_DEV_GPU_UTIL{Hostname=~"$node", gpu=~"$gpu"}

Legend:

    {{Hostname}} GPU {{gpu}}

### 15.5 GPU Memory Usage /think

DCGM_FI_DEV_FB_USED{Hostname=~"$node", gpu=~"$gpu"}

Unit:

    bytes or MiB

If the DCGM output is in MiB, select the MiB unit.

### 15.6 GPU Memory Usage

    DCGM_FI_DEV_FB_USED{Hostname=~"$node", gpu=~"$gpu"}
    /
    (
      DCGM_FI_DEV_FB_USED{Hostname=~"$node", gpu=~"$gpu"}
      +
      DCGM_FI_DEV_FB_FREE{Hostname=~"$node", gpu=~"$gpu"}
    )
    * 100

### 15.7 GPU Temperature

    DCGM_FI_DEV_GPU_TEMP{Hostname=~"$node", gpu=~"$gpu"}

Threshold:

    70: Yellow
    80: Orange
    90: Red

### 15.8 GPU Power Consumption

    DCGM_FI_DEV_POWER_USAGE{Hostname=~"$node", gpu=~"$gpu"}

Unit:

    Watt

### 15.9 XID Errors

    DCGM_FI_DEV_XID_ERRORS{Hostname=~"$node", gpu=~"$gpu"}

Recommended Display:

    Table
    Stat
    Alert list

### 15.10 GPU Idle Detection

Approach:

    High memory usage
    But long-term low GPU utilization

PromQL Example:

    avg_over_time(DCGM_FI_DEV_GPU_UTIL[1h]) < 5

Combined with memory:

    DCGM_FI_DEV_FB_USED > 1000

Note:

    Resident memory for inference service models may be normal.
    Low utilization alone cannot directly indicate waste.
    Should combine with business QPS and Pod status for judgment.

---

## SixteenI don't know.Business Service Dashboard Design

### 16.1 Objectives

Business Dashboard should answer:

    Is the service available?
    Is QPS normal?
    Has error rate increased?
    Has P95/P99 latency increased?
    Has Pod restarted?
    Is CPU/memory/GPU a bottleneck?
    Is it related to upstream/downstream anomalies?

### 16.2 Golden Metrics

Common four types of golden metrics:

    Latency: Delay
    Traffic: Traffic
    Errors: Error
    Saturation: Saturation

Corresponding panels:

    QPS
    5xx Error Rate
    P95 / P99 Latency
    CPU Usage
    Memory Usage
    Pod Restarts
    GPU Utilization
    Queue Length

### 16.3 QPS

    sum by (service) (
      rate(http_requests_total{service=~"$service"}[$__rate_interval])
    )

### 16.4 Error Rate

    sum by (service) (
      rate(http_requests_total{service=~"$service", status=~"5.."}[$__rate_interval])
    )
    /
    sum by (service) (
      rate(http_requests_total{service=~"$service"}[$__rate_interval])
    )
    * 100

### 16.5 P95 Latency

    histogram_quantile(
      0.95,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket{service=~"$service"}[$__rate_interval])
      )
    )

### 16.6 P99 Latency

    histogram_quantile(
      0.99,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket{service=~"$service"}[$__rate_interval])
      )
    )

### 16.7 Inference Service Specific Metrics

AI inference services recommend adding:

    model_inference_total
    model_inference_errors_total
    model_inference_duration_seconds_bucket
    model_load_status
    model_queue_length
    model_batch_size
    cuda_oom_total

If there are no custom business metrics, relying solely on GPU metrics cannot accurately determine service health.

---

## SeventeenI don't know.Dashboard Import and Export

### 17.1 Import Community Dashboard

Grafana supports importing Dashboards from grafana.com.

Common methods:

    Dashboards
      ↓
    New
      ↓
    Import
      ↓
    Enter Dashboard ID or upload JSON

Common Dashboard Types:

- Node Exporter Full
- Kubernetes Cluster
- kube-state-metrics
- NVIDIA DCGM Exporter
- Nginx
- MySQL
- Redis

### 17.2 Must Check After Import

Do not directly deploy the Dashboard after import.

Must check:

- Data source name matches
- PromQL metric names exist
- Label names match
- Variables can query values
- Panels have data
- Units are correct
- Thresholds match your environment
- Dashboard is not too heavy
- No expired metrics
- Compatibility with current version

### 17.3 Export Dashboard

Path:

    Dashboard Settings
      ↓
    JSON Model
      ↓
    Copy / Save

Or:

    Share
      ↓
    Export
      ↓
    Save to file

### 17.4 Dashboard Version Management

Production recommendation:

    Dashboard JSON should be managed in Git.

Directory example: /think

monitoring/
  grafana/
    dashboards/
      kubernetes-cluster-overview.json
      node-overview.json
      pod-overview.json
      gpu-overview.json
      app-inference-service.json
    provisioning/
      dashboards.yaml
      datasources.yaml

Advantages:

- Rollback capability;
- Auditable;
- Reusable;
- Synchronizable across environments;
- Avoids loss from manual UI modifications.

---

## EighteenI don't know.Dashboard Provisioning

### 18.1 Why Provisioning is Needed

Issues with manual dashboard creation:

- Not version-controlled;
- Difficult to replicate across environments;
- Prone to accidental modifications;
- Challenging for disaster recovery;
- Inconsistencies across multi-cluster environments.

Provisioning allows Grafana to automatically load on startup:

- Data Source;
- Dashboard;
- Alerting;
- Notification;
- Plugin configuration.

### 18.2 Data Source Provisioning Example

    apiVersion: 1

    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus-operated.monitoring.svc:9090
        isDefault: true

### 18.3 Dashboard Provisioning Example

    apiVersion: 1

    providers:
      - name: kubernetes-dashboards
        orgId: 1
        folder: Kubernetes
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/kubernetes

### 18.4 Mounting Dashboard in Helm values

kube-prometheus-stack supports mounting Dashboard via values or ConfigMap.

Production recommendations:

    Use ConfigMap or sidecar for automatic Dashboard discovery.
    Dashboard JSON should be managed via Git.
    Avoid arbitrary edits to critical dashboards in production UI.
    Changes must go through Pull Request.

---

## NineteenI don't know.Grafana Permissions and Folder Planning

### 19.1 Folder Planning

Recommend creating folders by theme:

    Kubernetes
    Nodes
    Applications
    GPU
    Databases
    Middleware
    Logs
    Alerts
    Capacity
    SRE

### 19.2 Permission Roles

Common roles:

    Viewer:
        Can only view Dashboard.

    Editor:
        Can edit Dashboard.

    Admin:
        Can manage data sources, users, and permissions.

Production recommendations:

    Regular developers get Viewer access.
    Application owners get Editor access to specific folders.
    SRE / platform teams manage global dashboards.
    Do not grant Admin access to everyone.

### 19.3 Dashboard Permissions

Permissions can be set per folder.

Example:

    GPU Folder:
        SRE can edit
        AI team can view
        Other teams invisible or read-only

    Application Folder:
        Corresponding business teams can edit
        SRE can manage

### 19.4 Data Source Permissions

In production environments, note:

    Grafana data sources may connect to Prometheus, Loki, or databases.
    Different teams may not need to see all namespaces or logs.
    In multi-tenant environments, data source permissions, LBAC, reverse proxy, or multiple Grafana instances are needed for isolation.

---

## TwentyI don't know.Grafana Alerting and Prometheus AlertManager Relationship

### 20.1 Prometheus Alert Rules

Prometheus Alert Rules are suitable for:

- Infrastructure alerts;
- Kubernetes alerts;
- GPU alerts;
- SRE centrally maintained rules;
- GitOps managed rules;
- Rules deeply integrated with AlertManager.

Link:

    Prometheus Rule
      ↓
    AlertManager
      ↓
    Notification channels

### 20.2 Grafana Alerting

Grafana Alerting can create alerts based on multiple data sources.

Suitable for:

- Unified alert management within Grafana;
- Cross-data source expressions;
- Metric + log query alerts;
- Team-level Dashboard inline alerts;
- Quick creation of panel-related alerts.

Link:

    Grafana Query / Expression
      ↓
    Grafana Alerting
      ↓
    Contact Point
      ↓
    Notification Policy

### 20.3 How to Choose

Recommendation for foundational platform alerts:

    PrometheusRule + AlertManager

Examples:

- NodeDown;
- PodCrashLoop;
- PodPending;
- GPUHighTemperature;
- GPUXIDError;
- DCGMExporterDown.

Custom visualization alerts for business teams:

    Use Grafana Alerting

Examples:

- A business error rate;
- A Dashboard-specific metric;
- Composite alerts across Prometheus / Loki.

### 20.4 Production Recommendations

Do not configure duplicate alerts for the same metric in both Prometheus and Grafana.

Otherwise, it may cause:

- Duplicate notifications;
- Inconsistent alert definitions;
- Confusion about responsible parties;
- Difficulty in silencing alerts;
- Alert storms.

Recommendation: /think

Platform base alerts should be categorized under PrometheusRule.
Business auxiliary alerts can be categorized under Grafana Alerting.
All alert routing should enter OnCall or notification system.

---

## 21. Grafana and Log Integration

### 21.1 Metric discovers issues, log locates root cause

Typical workflow:

    Grafana metric sees service 5xx rising
      ↓
    Click to jump to Loki logs
      ↓
    Filter namespace / pod / app
      ↓
    View ERROR stack
      ↓
    Locate abnormal cause

### 21.2 Loki Data Source

After configuring Loki data source, you can query LogQL in Grafana.

Example:

    {namespace="ai-prod", app="inference"} |= "ERROR"

Query CUDA OOM:

    {namespace=~"ai-.*"} |= "CUDA out of memory"

Query XID:

    {node="$node"} |= "Xid"

### 21.3 Dashboard Jump

You can configure Data links in Panel.

Example GPU XID Panel jumps to logs:

    /explore?left={"datasource":"Loki","queries":[{"expr":"{node=\"$node\"} |= \"Xid\""}]}

Production recommendation:

    GPU alert panel directly links to corresponding node logs.
    Pod anomaly panel directly links to corresponding Pod logs.
    Service error rate panel directly links to business logs.

---

## 22. Grafana Performance Optimization

### 22.1 Common Causes of Slow Dashboard

- Too many panels;
- Too complex PromQL;
- Too large time range;
- Too heavy variable queries;
- Too frequent auto-refresh;
- Use ofMass regular expressions;
- Querying high cardinality metrics;
- Each Panel queries large range data;
- No Recording Rule;
- Table panel returns too many series;
- Prometheus itself under pressure.

### 22.2 Optimization Recommendations

Recommendations:

    1. Control number of panels per Dashboard
    2. Control default time range
    3. Control refresh interval
    4. Use Recording Rule
    5. Avoid large range topk + regular expressions
    6. Use cascading for variables
    7. Use Dashboard Links to split dashboards
    8. Don't show too detailed data on overview page
    9. Limit Table panel to Top N
    10. Avoid high cardinality Label queries

### 22.3 Default Time Range

Recommendations:

    Cluster overview:
        Last 1h or Last 6h

    Troubleshooting:
        Last 30m / Last 1h

    Capacity trend:
        Last 7d / Last 30d

Don't set all Dashboards to default Last 30d.

### 22.4 Auto-refresh

Recommendations:

    Production large screen:
        30s / 1m

    Ordinary Dashboard:
        Manual or 1m

    Capacity dashboard:
        5m / 15m

Don't set 5s refresh for entire Dashboard unless certain of query pressure.

### 22.5 Use Recording Rule

Complex PromQL example:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval])
      ) * 100
    )

Can beDown. to:

    node:cpu_usage:percent

Query directly in Grafana:

    node:cpu_usage:percent

---

## 23. Common Grafana Troubleshooting

### 23.1 Dashboard No Data

Troubleshooting steps:

    1. Confirm time range is correct
    2. Confirm data source is normal
    3. Directly query metric in Prometheus
    4. Check if variables are empty
    5. Check Label match
    6. Check PromQL for errors
    7. Check if Panel query is hidden
    8. Check data source permissions

Common troubleshooting:

    Click Inspect in Grafana Panel
    View Query
    View Data
    View Response

### 23.2 Variables No Values

Common causes:

- PromQL written incorrectly;
- Wrong data source;
- Previous variable is empty;
- Label name mismatch;
- No data in time range;
- Prometheus not collecting this metric.

Example:

    label_values(kube_pod_info, namespace)

If no values, directly execute this query in Prometheus to confirm.

### 23.3 Panel Query Error

Common causes:

- PromQL syntax error;
- Metric does not exist;
- Label match error;
- Many-to-many match error;
- Division with inconsistent labels;
- Data source timeout;
- Prometheus query timeout.

Handling:

    First debug PromQL in Prometheus Web.
    Confirm results before putting into Grafana.

### 23.4 Dashboard Load Slow

Check:

- Query time range;
- Number of panels;
- Each Panel query duration;
- Prometheus CPU / memory;
- TSDB head series;
- Use of high cardinality metrics;
- Use of complex regular expressions.

### 23.5 Login Failure

Troubleshoot:

    kubectl get pod -n monitoring | grep grafana
    kubectl logs <grafana-pod> -n monitoring
    kubectl get secret -n monitoring | grep grafana

Check:

- admin password;
- OAuth configuration;
- LDAP configuration;
- Database connection;
- Grafana Pod restart;
- Ingress status.

---

## 24. Production Dashboard Norms

### 24.1 Naming Convention

Recommend: /think

# Kubernetes / Cluster Overview
# Kubernetes / Node Overview
# Kubernetes / Namespace Overview
# Kubernetes / Pod Overview
# GPU / Cluster Overview
# GPU / Node Detail
# GPU / Pod Detail
# App / Inference Service
# App / API Service
# Database / MySQL Overview

---

### 24.2 Panel Naming Guidelines

Recommended:

    Node CPU Usage Rate
    Node Memory Usage Rate
    Pod CPU Top 10
    Pod Restart Count Top 10
    GPU Utilization
    GPU Memory Usage
    GPU Temperature
    HTTP 5xx Error Rate
    P95 Request Latency

---

### 24.3 Unit Guidelines

Units must be set:

    bytes
    percent
    seconds
    milliseconds
    requests/sec
    celsius
    watt
    cores

Failure to set units will make charts difficult to interpret.

---

### 24.4 Threshold Guidelines

Recommend setting thresholds based on metrics:

    CPU:
        70 warning
        90 critical

    Memory:
        80 warning
        90 critical

    Disk:
        85 warning
        95 critical

    GPU Temperature:
        80 warning
        90 critical

    Error Rate:
        1 warning
        5 critical

---

### 24.5 Documentation Guidelines

Key Dashboards should add Text Panel for explanation:

- Dashboard purpose;
- Data sources;
- Variable explanations;
- Key metric meanings;
- Common troubleshooting paths;
- Related Runbook links.

---

## Twenty-Five, GPU Dashboard Production Template Ideas

### 25.1 Top Overview Row

Stat Panel:

    Total GPUs
    Average GPU Utilization
    Average GPU Memory Usage
    High-Temperature GPUs
    XID Error Count
    DCGM Exporter Down Count

### 25.2 Node Dimension Row

Panel:

    Average GPU Utilization per GPU Node
    Maximum Temperature per GPU Node
    Memory Usage per GPU Node
    GPU Pod Count per GPU Node
    Current Alert Count per GPU Node

### 25.3 Per-GPU Dimension Row

Panel:

    GPU Utilization Time Series
    GPU Memory Usage Time Series
    GPU Temperature Time Series
    GPU Power Consumption Time Series
    XID Error Stat/Table

### 25.4 Pod Dimension Row

Panel:

    GPU Pod List
    Pod GPU Utilization
    Pod Memory Usage
    Pod Host Node
    Pod Namespace
    Pod Restart Count

### 25.5 Resource Governance Row

Panel:

    Long-Low Utilization GPUs
    High Memory Usage but Low Utilization GPUs
    Namespace GPU Usage
    Idle GPU Count Trend
    GPU Pending Pods

---

## Twenty-Six, Grafana and Capacity Governance

Grafana is not only a troubleshooting tool, but can also be used for capacity analysis.

### 26.1 Node Capacity

Analysis:

- CPU usage trend;
- Memory usage trend;
- Disk growth trend;
- Network traffic trend;
- Node count changes;
- Pod count changes.

### 26.2 GPU Capacity

Analysis:

- GPU allocation rate;
- Actual GPU utilization;
- Memory peak;
- GPU Pending count;
- Low utilization GPUs;
- Usage by Namespace;
- Usage by business;
- Expansion needs.

### 26.3 Business Capacity

Analysis:

- QPS trend;
- Error rate trend;
- P95/P99 latency;
- Replica count changes;
- HPA scaling;
- Queue length;
- Resource cost.

### 26.4 Capacity Dashboard Recommendations

Time range:

    Last 7d
    Last 30d
    Last 90d

Refresh frequency:

    5m or 15m

Panel types:

    Time series
    Table
    Bar gauge
    Stat

Avoid high-frequency refreshes.

---

## Twenty-Seven, Grafana Operations Checklist

### 27.1 Before Deployment

    [ ] Deployment method confirmed
    [ ] Grafana version confirmed
    [ ] Data source type confirmed
    [ ] Prometheus address confirmed
    [ ] Namespace planned
    [ ] Persistent storage planned
    [ ] Admin password management planned
    [ ] Access entry planned
    [ ] HTTPS planned
    [ ] Permission model planned
    [ ] Dashboard folder planned
    [ ] Dashboard version management planned

### 27.2 After Deployment

    [ ] Grafana Pod Running
    [ ] Grafana Service normal
    [ ] Grafana can login
    [ ] Default password changed
    [ ] Prometheus data source Save & Test successful
    [ ] Basic Dashboard has data
    [ ] Variables can load normally
    [ ] Panel queries have no errors
    [ ] Dashboard load speed normal
    [ ] Grafana logs have no obvious errors

### 27.3 Production Check

- [ ] Use HTTPS  
- [ ] Access unified authentication or strong passwords  
- [ ] Prohibit anonymous management access  
- [ ] Critical Dashboard integrated into Git  
- [ ] Data sources managed via provisioning  
- [ ] Dashboard managed via provisioning  
- [ ] Permissions managed by Folder  
- [ ] Alert rules have responsible persons  
- [ ] Dashboard has documentation  
- [ ] Critical panels have Runbook links  
- [ ] Regularly clean up unused Dashboard  
- [ ] Regularly check slow queries  
- [ ] Regularly check variable performance  

---

## Twenty-EightI don't know.Common Misconceptions

### 28.1 Misconception 1: Having a Grafana dashboard means monitoring is complete  

Error.  

Complete monitoring requires:  

    Metric collection  
    Data storage  
    Dashboard  
    Alerting  
    Notifications  
    Runbook  
    On-call  
    Post-mortem  

Grafana is merely the entry point for display and analysis.  

### 28.2 Misconception 2: More dashboards mean better monitoring  

Error.  

Too many dashboards will lead to:  

- Slow loading  
- Hard to understand  
- Disordered troubleshooting paths  
- Heavy query pressure  
- Difficult maintenance  

### 28.3 Misconception 3: Importing community dashboards directly makes them production-ready  

Error.  

After importing, you must check:  

- Whether metrics exist  
- Whether labels match  
- Whether units are correct  
- Whether thresholds are suitable  
- Whether queries are too heavy  
- Whether it suits the current business  

### 28.4 Misconception 4: Everyone should have Editor permissions  

Error.  

This will lead to:  

- Dashboards being accidentally modified  
- Data sources being accidentally deleted  
- Uncontrolled permissions  
- Unstable production dashboards  

### 28.5 Misconception 5: Grafana can replace Prometheus  

Error.  

Grafana does not collect or store Prometheus metrics.  

Without Prometheus or other data sources, Grafana has no data to display.  

### 28.6 Misconception 6: Grafana can replace log systems  

Error.  

Grafana can display Loki / Elasticsearch logs, but log collection and storage are still handled by Loki, EFK, etc.  

---

## Twenty-NineI don't know.Experiment Verification Process

### 29.1 Verify Grafana access  

    kubectl get pods -n monitoring | grep grafana  
    kubectl get svc -n monitoring | grep grafana  
    kubectl port-forward svc/<grafana-service-name> 3000:80 -n monitoring  

Access:  

    http://127.0.0.1:3000  

### 29.2 Verify Prometheus data source  

Grafana UI:  

    Connections  
      ↓  
    Data sources  
      ↓  
    Prometheus  
      ↓  
    Save & test  

### 29.3 Create the first Panel  

Dashboard:  

    New dashboard  
      ↓  
    Add visualization  
      ↓  
    Select Prometheus  
      ↓  
    Enter PromQL  

Example:  

    up  

Save Panel:  

    Title: Prometheus Target status  

### 29.4 Create Node CPU Panel  

PromQL:  

    100 - (  
      avg by (instance) (  
        rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval])  
      ) * 100  
    )  

Panel type:  

    Time series  

Unit:  

    percent  

### 29.5 Create GPU utilization Panel  

PromQL:  

    DCGM_FI_DEV_GPU_UTIL  

Panel type:  

    Time series  

Unit:  

    percent  

### 29.6 Create Pod restart Table  

PromQL:  

    topk(10, increase(kube_pod_container_status_restarts_total[$__rate_interval]))  

Panel type:  

    Table  

---

## ThirtyI don't know.Production Deployment Recommendations

### 30.1 Dashboard change process  

Recommendations:  

    1. Modify Dashboard in test environment  
    2. Export JSON  
    3. Commit to Git  
    4. Code Review  
    5. Sync automatically after merge  
    6. Production Grafana auto-load  

### 30.2 Dashboard owner  

Each critical Dashboard should have an owner.  

Record:  

    Dashboard name  
    Owner  
    Data source  
    Usage scenario  
    Associated alerts  
    Associated Runbook  
    Last updated time  

### 30.3 Regular cleanup  

Regularly clean up:  

- Dashboard with no data  
- Duplicate Dashboard  
- Temporary Dashboard  
- Slow query Dashboard  
- Dashboard without owner  
- Dashboard with outdated metrics  

### 30.4 Multi-environment isolation  

Recommendations:  

    dev / test / prod separated by variables or folders.  
    Production Dashboard only displays production data.  
    Different clusters distinguished by $cluster variable.  
    Multi-tenant environments controlled by team for permissions.  

---

## Thirty-OneI don't know.Summary  

Grafana is the core visualization entry in cloud-native observability systems.  

It does not collect metrics or directly store Prometheus metrics, but queries Prometheus, Loki, Tempo, Elasticsearch, databases, etc., then organizes results into Dashboard and Panel.  

Grafana's core chain is:  

    Data Source  
      ↓  
    Query  
      ↓  
    Transformation  
      ↓  
    Panel  
      ↓  
    Dashboard  
      ↓  
    Variable  
      ↓  
    Drilldown / Alert / Runbook  

A production-grade Grafana dashboard should not just be "a few charts", but should have:

Clear Theme  
Clear Variables  
Correct Units  
Reasonable Thresholds  
Readable Legend  
Troubleshootable Structure  
Navigable Logs  
Alert-Associated  
Runbook-Linked  
Version-Managed  
Permission-Controlled  

In Kubernetes and GPU operations scenarios, Grafana should at least cover:  

  Cluster Overview  
  Node Resources  
  Pod / Namespace Resources  
  Service / Ingress  
  GPU Utilization  
  GPU Memory  
  GPU Temperature  
  GPU XID  
  Application QPS  
  Application Error Rate  
  P95 / P99 Latency  
  Alert Overview  
  Capacity Trends  

When troubleshooting, follow this process:  

  Grafana Detects Anomalies  
    ↓  
  Prometheus Query Metrics  
    ↓  
  Kubernetes Events Confirm Status  
    ↓  
  Loki / EFK View Logs  
    ↓  
  kubectl describe Locate Resources  
    ↓  
  Node Commands Validate Underlying Layer  
    ↓  
  Runbook Execute Resolution  

Grafana's ultimate goal is not to display data, but to enable operations personnel to understand system status faster, locate issues quicker, and more stably support production operations.  

---

## Reference Documents  

- Grafana Documentation:  
  https://grafana.com/docs/grafana/latest/  

- Grafana Dashboards:  
  https://grafana.com/docs/grafana/latest/visualizations/dashboards/  

- Grafana Dashboard Variables:  
  https://grafana.com/docs/grafana/latest/visualizations/dashboards/variables/  

- Grafana Variable Syntax:  
  https://grafana.com/docs/grafana/latest/visualizations/dashboards/variables/variable-syntax/  

- Grafana Transformations:  
  https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/query-transform-data/transform-data/  

- Grafana Alerting:  
  https://grafana.com/docs/grafana/latest/alerting/  

- Prometheus Grafana Support:  
  https://prometheus.io/docs/visualization/grafana/  

- Prometheus Querying:  
  https://prometheus.io/docs/prometheus/latest/querying/basics/  

- Prometheus Alerting Rules:  
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/  

- kube-prometheus-stack Helm Chart:  
  https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack  

- Grafana Helm Chart:  
  https://github.com/grafana/helm-charts  

- NVIDIA DCGM Exporter:  
  https://github.com/NVIDIA/dcgm-exporter  

- Kubernetes Monitoring with kube-state-metrics:  
  https://github.com/kubernetes/kube-state-metrics