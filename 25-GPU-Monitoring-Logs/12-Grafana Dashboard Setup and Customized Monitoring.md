# 12-Grafana Dashboard Setup and Customized Monitoring

## Document Description

This document systematically outlines how to utilize Grafana in Kubernetes, Prometheus, GPU monitoring, and business observability, covering data source configuration, dashboard design, panel type selection, variable setup, PromQL writing, design of Node/Pod/GPU/business dashboards, dashboard import/export, permission management, alarm integration, performance optimization, and production environment guidelines.

This document doesn’t merely explain “how to create charts with a few clicks.” Instead, it provides an operational/SRE perspective on:

- The role of Grafana within the monitoring ecosystem;
- The relationship between Prometheus and Grafana;
- How to design dashboards in layers;
- How to choose appropriate panels;
- How variables enable dashboards to adapt to multiple clusters, namespaces, nodes, and businesses;
- Which metrics should be monitored for Node, Pod, Service, and GPU dashboards;
- How to integrate Grafana with AlertManager/Grafana Alerting;
- How to avoid issues such as slow dashboard performance, heavy queries, excessive variables, and chaotic panels;
- How to transform Grafana from a “chart-viewing tool” into a production monitoring solution that supports troubleshooting, retrospection, and governance.

This document is recommended for those who have completed the following courses:

- 11-Prometheus-Architecture and Core Metrics Analysis
- 08-GPU-Monitoring and Alarm Integration
- 09-GPU-Fault Diagnosis Cases and Practical Applications
- 10-GPU-Operational Experiment Environment Setup and Verification

---

## Tags

#Grafana #Prometheus #Kubernetes #Dashboard #Panel #PromQL #GPU Monitoring #DCGMExporter #AlertManager #SRE #Observability #Operational Practice

---

## Recommended Reading Order

Recommended sequence:

    06-GPU and AI Infrastructure/05-Fundamentals of Observability/12-Grafana Dashboard Setup and Customized Monitoring.md

---

## I. Why We Need Grafana

Prometheus is responsible for collecting, storing, and querying metrics.

However, Prometheus’ native web interface is more suitable for temporary queries and debugging rather than long-term production use.

The value of Grafana lies in its ability to:

    Unify the display of data from multiple sources such as Prometheus, Loki, Elasticsearch, MySQL, PostgreSQL, InfluxDB, CloudWatch, etc., through dashboards.

For operations/SRE professionals, Grafana is not just a “charting tool” but a critical component of production monitoring. It should provide insights into:

- The health of the current cluster;
- Nodes with high resource usage;
- Pods that are restarting abnormally;
- Services experiencing increased latency;
- Namespaces consuming the most resources;
- Idle or fully utilized GPUs;
- GPU errors such as XID issues;
- Abnormalities in QPS, error rates, or latency for specific services;
- Which dashboard to consult when an alarm is triggered.

Grafana’s ultimate goal is not to produce numerous charts but to:

    Help identify problems more quickly;
    Locate faults more swiftly;
    Make metrics easier to understand;
    Facilitate the analysis of capacity trends;
    Standardize troubleshooting procedures.

---

## II. Grafana’s Role in the Observability Ecosystem

A complete observability system typically includes:

    Metrics: Quantifiable data points
    Logs: Detailed records of system activities
    Traces: Detailed path information for transactions
    Events: Specific incidents that occur within a system
    Alerts: Notifications when certain conditions are met

Grafana primarily handles:

    Visual presentation;
    Multi-source data queries;
    Dashboard management;
    Variable filtering;
    Alert display;
    Data integration;
    Troubleshooting entry points.

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

If logs are integrated:

    App Logs / Node Logs
      ↓
    Promtail / Fluent Bit
      ↓
    Loki / Elasticsearch
      ↓
    Grafana Logs Panel

If trace data is integrated:

    OpenTelemetry / Jaeger / Tempo
      ↓
    Grafana Trace Panel

---

## III. The Relationship Between Grafana and Prometheus

Prometheus handles:

- Metric collection;
- Time series storage;
- Execution of PromQL queries;
- Calculation- Adjust the display structure.

Transformation is suitable for handling the presentation layer.

However, don't overwhelm Grafana with overly complex business calculations.

For complex calculations, consider the following options first:

    PromQL
    Recording Rules
    Aggregation at the data source level
    Metric design on the application side

---

## Section 5: Grafana Deployment Methods

### 5.1 Grafana Included in kube-prometheus-stack

In Kubernetes monitoring, the most common approach is to install it using the kube-prometheus-stack.

It typically includes:

- Prometheus Operator;
- Prometheus;
- AlertManager;
- Grafana;
- node-exporter;
- kube-state-metrics;
- Default dashboards;
- Default Prometheus Rules;
- ServiceMonitor support.

Example installation:

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --version <CHART_VERSION>

Production recommendations:

    Use the values.yaml file.
    Fix the chart version.
    Do not install without a specific version.
    Avoid using default passwords.
    Configure persistent storage.
    Set up Ingress or internal access routes.
    Configure RBAC and permissions.

### 5.2 Installing Grafana Separately Using Helm

If you already have Prometheus but want to install Grafana separately, use the Grafana Helm Chart.

Example:

    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update

    kubectl create namespace monitoring

    helm install grafana grafana/grafana \
      --namespace monitoring \
      --version <CHART_VERSION>

To check the installation:

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring

### 5.3 Using Docker for Local Testing

This method is suitable for local experimentation:

    docker run -d \
      --name grafana \
      -p 3000:3000 \
      grafana/grafana:<version>

It is not recommended for long-term use in production environments due to the lack of persistent storage.

### 5.4 Binary or Package Installation

This method is suitable for traditional host-based deployments:

    Ubuntu / Debian
    RHEL / Rocky Linux
    systemd management
    Nginx reverse proxy

Production recommendations:

    In Kubernetes environments, prefer using Helm.
    For traditional VMs, use package managers or binary installations.
    Maintain consistent configuration management across different environments.

---

## Section 6: Accessing Grafana

### 6.1 Checking the Service

    kubectl get svc -n monitoring | grep grafana

### 6.2 Port Forwarding for Access

    kubectl port-forward svc/<grafana-service-name> 3000:80 -n monitoring

Access address:

    http://127.0.0.1:3000

### 6.3 Obtaining the Default Password

If you used a Helm Chart, the default admin password is usually stored in a Secret.

Example:

    kubectl get secret -n monitoring

To view the specific Secret:

    kubectl get secret <grafana-secret-name> -n monitoring -o yaml

Common method to retrieve the password:

    kubectl get secret <grafana-secret-name> -n monitoring \
      -o jsonpath="{.data.admin-password}" | base64 -d

Note:

    In production environments, change the default password.
    Do not disclose the admin password in public documents.
    Avoid using weak passwords.
    It is recommended to integrate with a unified authentication system.

### 6.4 Using Ingress for Exposure

In production environments, you can expose Grafana through an Ingress.

Example configuration:

    grafana.example.com
      ↓
    Ingress
      ↓
    grafana Service
      ↓
    grafana Pod

Note:

    Enable HTTPS.
    Configure authentication.
    Restrict access sources.
    It is not recommended to expose the Grafana management interface directly over the public internet.

---

## Section 7: Configuring Prometheus Data Sources

### 7.1 UI Configuration Method

On the Grafana page:

    Connections
      ↓
    Data sources
      ↓
    Add data source
      ↓
    Prometheus

Enter the configuration URL:

    http://prometheus-operated.monitoring.svc:9090

or:

    http://prometheus-server_monitoring.svc

The specific address depends on the name of your Prometheus Service.

To check the Service:

    kubectl get svc -n monitoring | grep prometheus

Click "Save & test".

If successful, it will indicate that the data source is available.

### 7.2 Common Prometheus Service Addresses

In a kube-prometheus-stack setup:

    http://kube-prometheus-stack-prometheus.monitoring.svc:909### 1. Cluster Overview Dashboard### 12.1 Objectives

The Cluster Overview Dashboard addresses the following questions:

- Is the current Kubernetes cluster healthy?
- Are there any abnormal nodes?
- Are there a large number of abnormal Pods?
- Is there any resource pressure?
- Are there any alerts?
- Is the GPU cluster functioning normally?

### 12.2 Recommended Panels

**Stat Panel:**

- Total number of Nodes
- Number of Ready Nodes
- Number of NotReady Nodes
- Total number of Pods
- Number of Running Pods
- Number of Pending Pods
- Number of Failed Pods
- Current number of active alerts
- Total number of GPUs
- Current average GPU utilization

**Time Series Panel:**

- Trend in cluster CPU usage
- Trend in cluster memory usage
- Trend in the number of Pods
- Trend in pending Pods
- Trend in GPU utilization
- Trend in GPU video memory usage

**Table Panel:**

- List of NotReady Nodes
- List of Pending Pods
- Top 10 Pods that need to be restarted
- Top 10 Pods with high CPU usage
- Top 10 Pods with high memory usage
- List of GPUs with high temperatures

### 12.3 Common PromQL Queries

**Node Ready:**

```promql
kube_node_status_condition{condition="Ready", status="true"}
```

**NotReady Node:**

```promql
kube_node_status_condition{condition="Ready", status="true"} == 0
```

**Pod Pending:**

```promql
kube_pod_status_phase{phase="Pending"} == 1
```

**Pod Failed:**

```promql
kube_pod_status_phase{phase="Failed"} == 1
```

**Top 10 Restarting Pods:**

```promql
topk(10, increase(kube_pod_container_status_restarts_total[$__rate_interval]))
```

**GPU Average Utilization:**

```promql
avg(DCGM_FI_DEV_GPU_UTIL)
```

---

## Section 13: Node Dashboard Design

### 13.1 Objectives

The Node Dashboard provides insights into the following aspects of a specific node:

- Are there any abnormalities with the node's CPU, memory, disk, or network?
- Does the node face any resource bottlenecks?
- Are there any abnormal Pods on this node?
- Are there any issues with the temperature, power consumption, or XID of GPU nodes?

### 13.2 Recommended Variables

```yaml
$node
```

Variable queries:

```yaml
label_values(kube_node_info, node)
```

or:

```yaml
label_values(node_uname_info, instance)
```

### 13.3 Recommended Panels

**Time Series:**

- CPU usage
- Memory usage
- Disk usage
- Network receive rate
- Network send rate
- Load average
- Number of containers
- Number of Pods

**Table:**

- List of Pods on this node
- Top 10 Pods with high CPU usage
- Top 10 Pods with high memory usage
- Pod restart history for this node
- Abnormal events related to this node

**Additional GPU Node Panels:**

- GPU utilization
- GPU video memory usage
- GPU temperature
- GPU power consumption
- GPU XID errors

### 13.4 PromQL Queries for CPU Usage:

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle", instance="$node"}[$__rate_interval])
  ) * 100
)
```

### 13.5 PromQL Query for Memory Usage:

```promql
(
  1 - node_memory_MemAvailable_bytes{instance="$node"}
  / node_memory_MemTotal_bytes{instance="$node"}
) * 100
```

### 13.6 PromQL Query for Disk Usage:

```promql
(
  1 - node_filesystem_avail_bytes{instance="$node", fstype!="tmpfs"}
  / node_filesystem_size_bytes{instance="$node", fstype!="tmpfs"}
) * 100
```

### 13.7 PromQL Query for Network Receive Rate:

```promql
rate(node_network_receive_bytes_total{instance="$node", device!~"lo"}[$__rate_interval])
```

### 13.8 PromQL Query for Network Send Rate:

```promql
rate(node_network_transmit_bytes_total{instance="$node", device!~"lo"}[$__rate_interval])
```

---

## Section 14: Pod / Namespace Dashboard Design

### 14.1 Objectives

The Pod / Namespace Dashboard provides information on the following aspects:

- Which namespace is consuming the most resources?
- Which Pods have high CPU usage?
- Which Pods have high memory usage?
- Which Pods are restarting frequentlyIt is necessary to first query the original metrics in Prometheus to confirm the label structure.

### 15.3 GPU Overview Panel

Statistics:

    Total number of GPUs
    Average GPU utilization rate
    Average GPU memory usage rate
    Number of GPUs with high temperatures
    Number of XID errors
    Status of DCGM Exporter Target

### 15.4 GPU Utilization

    DCGM_FI_DEV_GPU_UTIL{Hostname=~"$node", gpu=~"$gpu"}

Legend:

    {{Hostname}} GPU {{gpu}}

### 15.5 GPU Memory Usage

    DCGM_FI_DEV_FB_USED{Hostname=~"$node", gpu=~"$gpu"}

Unit:

    Bytes or MiB

If the DCGM output is in MiB, select MiB as the unit.

### 15.6 GPU Memory Usage Rate

    DCGM_FI_DEV_FB_used{Hostname=~"$node", gpu=~"$gpu"}
    /
    (
      DCGM_FI_DEV_FB_USED{Hostname=~"$node", gpu=~"$gpu"}
      +
      DCGM_FI_DEV_FB_FREE{Hostname=~"$node", gpu=~"$gpu"}
    )
    * 100

### 15.7 GPU Temperature

    DCGM_FI_DEV_GPU_TEMP{Hostname=~"$node", gpu=~"$gpu"}

Thresholds:

    70: Yellow
    80: Orange
    90: Red

### 15.8 GPU Power Consumption

    DCGM_FI_DEV_POWER_usage{Hostname=~"$node", gpu=~"$gpu"}

Unit:

    Watts

### 15.9 XID Errors

    DCGM_FI_DEV_XID-errors{Hostname=~"$node", gpu=~"$gpu"}

It is recommended to display these using:

    Table
    Statistics
    Alert list

### 15.10 Identifying Idle GPUs

Approach:

    High memory usage
    But long-term low GPU utilization rate

PromQL example:

    avg_over_time(DCGM_FI_DEV_GPU_UTIL[1h]) < 5

In conjunction with memory usage:

    DCGM_FI_DEV_FB_USED > 1000

Note:

    It is normal for the inference service model to consume a certain amount of memory permanently.
    Low utilization rate alone cannot be used to determine waste.
    It should be judged in combination with business QPS and Pod status.

---

## Section Sixteen: Business Service Dashboard Design

### 16.1 Objectives

The business dashboard aims to answer the following questions:

    Is the service available?
    Is the QPS normal?
    Has the error rate increased?
    Have the P95/P99 delays increased?
    Have any Pods restarted?
    Are CPU/memory/GPU becoming bottlenecks?
    Is there any correlation with upstream/downstream exceptions?

### 16.2 Key Metrics

Four common types of key metrics:

    Latency: Delay
    Traffic: Volume of traffic
    Errors: Number of errors
    Saturation: Degree of saturation

Corresponding panels:

    QPS
    5xx Error Rate
    P95 / P99 Delays
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
      rate(httprequests_total{service=~"$service", status/~"5.."}[$__rate_interval])
    )
    /
    sum by (service) (
      rate(http_requests_total{service=~"$service"}[$__rate_interval])
    )
    * 100

### 16.5 P95 Delay

    histogram_quantile(
      0.95,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket{service=~"$service"}[$__rate_interval])
      )
    )

### 16.6 P99 Delay

    histogram_quantile(
      0.99,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket{service=~"$service"}[$__rate_interval])
      )
    )

### 16.7 Special Metrics for AI Inference Services

It is recommended to add the following metrics for AI inference services:

    model_inference_total
    model_inference_errors_total
    model_inference_duration_seconds_bucket
    model_load_status
    model_queue_length
    model_batch_size
    cuda_oom_total

Without custom business metrics, it is impossible to accurately determine whether the service is healthy based solely on GPU metrics.

---

## Section Seventeen: Dashboard Import and Export

### 17.1 Importing CommunityThe kube-prometheus-stack supports mounting Dashboards via values or ConfigMap.

Production Recommendations:

    Use ConfigMap or sidecar for automatic Dashboard discovery.
    Manage Dashboard JSON files using Git.
    Do not arbitrarily edit critical dashboards in the production UI.
    Make all changes through Pull Requests.

---

## Section 19: Grafana Permissions and Folder Organization

### 19.1 Folder Organization

It is recommended to create folders based on topics:

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

Common roles include:

    Viewer:
        Only able to view Dashboards.

    Editor:
        Can edit Dashboards.

    Admin:
        Can manage data sources, users, permissions, etc.

Production Recommendations:

    Assign the Viewer role to regular developers.
    Grant the Editor role to specific folders for application owners.
    Let the SRE/Platform team manage global Dashboards.
    Avoid giving everyone the Admin role.

### 19.3 Dashboard Permissions

Permissions can be set per folder.

For example:

    GPU Folder:
        SRE can edit
        AI team can view
        Other teams have no access or read-only permission

    Application Folder:
        Relevant business teams can edit
        SRE can manage it

### 19.4 Data Source Permissions

In a production environment, consider the following:

    Grafana data sources may connect to Prometheus, Loki, or databases.
    Different teams should not necessarily have access to all namespaces or logs.
    In a multi-tenant environment, use data source permissions, LBAC, reverse proxies, or multiple Grafana instances for isolation.

---

## Section 20: The Relationship Between Grafana Alerting and Prometheus AlertManager

### 20.1 Prometheus Alert Rules

Prometheus Alert Rules are suitable for:

- Infrastructure alerts;
- Kubernetes alerts;
- GPU alerts;
- Rules maintained uniformly by SRE;
- Rules managed by GitOps;
- Rules deeply integrated with AlertManager.

Chain of Events:

    Prometheus Rule
      ↓
    AlertManager
      ↓
    Notification Channel

### 20.2 Grafana Alerting

Grafana Alerting allows creating alerts based on multiple data sources.

It is suitable for:

- Unified alert management within Grafana;
- Expressions across data sources;
- Alerts based on metrics and log queries;
- Inline alerts within team dashboards;
- Quickly creating panel-related alerts.

Chain of Events:

    Grafana Query/Expression
      ↓
    Grafana Alerting
      ↓
    Contact Point
      ↓
    Notification Policy

### 20.3 How to Choose

For basic platform alerts, it is recommended to use:

    PrometheusRule + AlertManager

Examples include:

- NodeDown;
- PodCrashLoop;
- PodPending;
- GPUHighTemperature;
- GPUXIDError;
- DCGMExporterDown.

For business teams to customize visual alerts, Grafana Alerting can be used.

Examples include:

- Specific business error rates;
- Metrics exclusive to certain dashboards;
- Composite alerts across Prometheus/Loki.

### 20.4 Production Recommendations

Do not configure duplicate alerts for the same metric in both Prometheus and Grafana.

This can lead to:

- Duplicate notifications;
- Inconsistent alert definitions;
- Confusion among handlers;
- Difficulty in silencing alerts;
- Alert storms.

It is recommended to specify:

    Platform-based basic alerts should use PrometheusRule.
    Business-supporting auxiliary alerts can use Grafana Alerting.
    All alerts should be routed through the same OnCall or notification system.

---

## Section 21: Integrating Grafana with Logs

### 21.1 Identifying Issues with Metrics and Locating Causes in Logs

Typical process:

    Notice an increase in service 5xx errors in Grafana metrics
      ↓
    Click to navigate to Loki logs
      ↓
    Filter by namespace/pod/app
      ↓
    View the ERROR stack
      ↓
    Locate the cause of the exception

### 21.2 Using the Loki Data Source

After configuring the Loki data source, you can query LogQL in Grafana.

Example:

    {namespace="ai-prod", app="inference"} |= "ERROR"

To query for CUDA OOM:

    {namespace=~"ai-.*"} |= "CUDA out of memory"

To query for XID:

    {node="$node"} |= "Xid"

### 21.3 Dashboard Linking

You can configure Data links within Panels.

For example, a GPU XID Panel could link directly to logs:

    /explore?left={"datasource":"Loki","queries":[{"expr":"{node="$node\"} |= \"Xid\""}]}

In production, it is recommended to:

- Directly link GPU alert panels to the corresponding node logs.
- Direct```markdown
kubectl get secret -n monitoring | grep grafana

Check:

- admin password;
- OAuth configuration;
- LDAP configuration;
- Database connection;
- Whether the Grafana Pod has been restarted;
- Whether the Ingress is functioning properly.

---

## 24. Production Environment Dashboard Specifications

### 24.1 Naming Conventions

Recommendations:

    Kubernetes / Cluster Overview
    Kubernetes / Node Overview
    Kubernetes / Namespace Overview
    Kubernetes / Pod Overview
    GPU / Cluster Overview
    GPU / Node Detail
    GPU / Pod Detail
    App / Inference Service
    App / API Service
    Database / MySQL Overview

Avoid using:

    test
    new dashboard
    dashboard copy
    temporary dashboard
    unnamed

### 24.2 Panel Naming Conventions

Suggestions:

    Node CPU Utilization
    Node Memory Utilization
    Top 10 Pod CPUs
    Top 10 Pod Restart Counts
    GPU Usage
    GPU Video Memory Usage
    GPU Temperature
    HTTP 5xx Error Rate
    P95 Request Latency

### 24.3 Unit Conventions

Units must be specified:

    bytes
    percent
    seconds
    milliseconds
    requests/sec
    celsius
    watt
    cores

Failing to specify units will make the charts difficult to interpret.

### 24.4 Threshold Conventions

It is recommended to set thresholds based on indicators:

    CPU:
        70 for warning
        90 for critical

    Memory:
        80 for warning
        90 for critical

    Disk:
        85 for warning
        95 for critical

    GPU Temperature:
        80 for warning
        90 for critical

    Error Rate:
        1 for warning
        5 for critical

### 24.5 Documentation

It is suggested to add a Text Panel to key Dashboards to explain:

- The purpose of the dashboard;
- Data sources;
- Variable explanations;
- Meanings of key indicators;
- Common troubleshooting steps;
- Links to relevant Runbooks.

---

## 25. Production Template Ideas for GPU Dashboards

### 25.1 Top Overview Row

Stat panels:

    Total number of GPUs
    Average GPU utilization
    Average video memory usage
    Number of GPUs with high temperatures
    Number of XID errors
    Number of DCGM Exporter failures

### 25.2 Node Dimension Row

Panels:

    Average GPU utilization per node
    Maximum temperature per node
    Video memory usage per node
    Number of GPU Pods per node
    Current number of alerts per node

### 25.3 Single Card Dimension Row

Panels:

    Time series of GPU utilization
    Time series of video memory usage
    Time series of GPU temperature
    Time series of GPU power consumption
    Stat/table of XID errors

### 25.4 Pod Dimension Row

Panels:

    List of GPU Pods
    GPU utilization of pods
    Video memory usage of pods
    Node where the pod is located
    Pod namespace
    Number of pod restarts

### 25.5 Resource Management Row

Panels:

    GPUs with long-term low utilization
    GPUs with high video memory usage but low utilization
    GPU usage by namespace
    Trend in the number of idle GPUs
    Pending Pods for GPUs

---

## 26. Grafana and Capacity Management

Grafana is not only a troubleshooting tool but can also be used for capacity analysis.

### 26.1 Node Capacity

Analysis:

- Trends in CPU usage;
- Trends in memory usage;
- Trends in disk growth;
- Trends in network traffic;
- Changes in the number of nodes;
- Changes in the number of pods.

### 26.2 GPU Capacity

Analysis:

- GPU allocation rate;
- Actual GPU utilization;
- Peak video memory usage;
- Number of pending GPU requests;
- GPUs with low utilization;
- Usage by different namespaces;
- Usage by different services;
- Expansion requirements.

### 26.3 Service Capacity

Analysis:

- Trends in QPS;
- Trends in error rates;
- P95/P99 latency;
- Changes in the number of replicas;
- HPA scaling;
- Queue lengths;
- Resource costs.

### 26.4 Recommendations for Capacity Dashboards

Time range:

    Last 7 days
    Last 30 days
    Last 90 days

Refresh frequency:

    Every 5 minutes or 15 minutes

Panel types:

    Time series
    Table
    Bar gauge
    Stat

Avoid using high-frequency refreshes.

---

## 27. Grafana Operation and Maintenance Checklist

### 27.1 Before Deployment

    [ ] The deployment method has been confirmed.
    [ ] The version of Grafana has### 29.4 Creating a Node CPU Panel

PromQL:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval])
      ) * 100
    )

Panel Type:

    Time series

Unit:

    percent

### 29.5 Creating a GPU Utilization Panel

PromQL:

    DCGM_FI_DEV_GPU_UTIL

Panel Type:

    Time series

Unit:

    percent

### 29.6 Creating a Pod Restart Table

PromQL:

    topk(10, increase(kube_pod_container_status_restarts_total[$__rate_interval]))

Panel Type:

    Table

---

## Thirty, Production Implementation Recommendations

### 30.1 Dashboard Change Process

Recommendations:

    1. Make changes to the dashboard in a test environment.
    2. Export the JSON format.
    3. Submit it via Git.
    4. Conduct a code review.
    5. After merging, synchronize automatically.
    6. The production Grafana instance will automatically load the updated dashboard.

### 30.2 Dashboard Owners

Each key dashboard should have an assigned owner.

Records:

    Dashboard Name
    Owner
    Data Source
    Usage Scenarios
    Associated Alerts
    Related Runbooks
    Last Update Time

### 30.3 Regular Cleanup

Periodically remove:

- Dashboards with no data;
- Duplicate dashboards;
- Temporary dashboards;
- Dashboards that cause slow queries;
- Dashboards without an owner;
- Dashboards containing outdated metrics.

### 30.4 Multi-environment Isolation

Recommendations:

    Use separate variables or folders for development, testing, and production environments.
    The production environment dashboard should only display data from the production environment.
    Different clusters can be distinguished using the $cluster variable.
    In a multi-tenant environment, control access permissions by team.

---

## Thirty-one, Summary

Grafana is the core visualization tool in cloud-native observability systems.

It does not collect metrics itself nor store Prometheus metrics directly. Instead, it retrieves data from systems such as Prometheus, Loki, Tempo, Elasticsearch, and databases, and then organizes the results into dashboards and panels.

The main workflow of Grafana involves:

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

A production-grade Grafana dashboard should not simply consist of "a few charts." It should have:

    A clear theme;
    Well-defined variables;
    Correct units;
    Reasonable thresholds;
    Legible legends;
    An structure that facilitates troubleshooting;
    Links to logs for further investigation;
    Associated alerts;
    Links to Runbooks for action plans;
    Support for version control;
    Access controls tailored to user roles.

In Kubernetes and GPU operations, Grafana should cover at least the following areas:

    Cluster overview;
    Node resources;
    Pod/Namespace resources;
    Service/Ingress settings;
    GPU utilization;
    GPU memory usage;
    GPU temperature;
    GPU XID information;
    Application QPS;
    Application error rates;
    P95/P99 latency levels;
    An overview of alerts;
    Capacity trends.

When troubleshooting, follow this process:

    Grafana detects an issue;
    Prometheus is used to query relevant metrics;
    Kubernetes Events help confirm the status;
    Logs are checked through Loki/EFK;
    Resources are identified using `kubectl describe`;
    Detailed checks are performed on nodes via command lines;
    Runbooks are executed as needed.

The ultimate goal of Grafana is not just to display data but to enable operations personnel to quickly understand system conditions, locate problems, and ensure the stability of production services.

---

## References

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

- Prometheus and Grafana Integration:
  https://prometheus.io/docs/visualization/grafana/

- Prometheus Querying Basics:
  https://prometheus.io/docs/prometheus/latest/querying/basics/

- Prometheus Alerting Rules Configuration:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- kube-prometheus-stack Helm Chart