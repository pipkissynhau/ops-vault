# 14-K8S-Monitoring-Node-Pod-Service Metric Collection and Troubleshooting

## Document Overview

This document systematically organizes the metric collection, Prometheus queries, Grafana visualization, alert design, Pod log collection integration, and troubleshooting methods for the three core objects: Node, Pod, and Service in Kubernetes environments.

This document is not about how to install Prometheus or merely listing PromQL. Instead, it provides an SRE/operations perspective to connect the following chain:

    Kubernetes Objects
      ↓
    Metric Sources
      ↓
    Prometheus Collection
      ↓
    PromQL Queries
      ↓
    Grafana Dashboard
      ↓
    AlertManager Alerts
      ↓
    kubectl / Events / Logs / Service / Endpoints Troubleshooting
      ↓
    Fault Closure

This document answers the following questions:

- What should be monitored for Node, Pod, and Service in Kubernetes?
- What do node-exporter, kubelet/cAdvisor, and kube-state-metrics collect?
- What's the difference between Metrics Server and Prometheus?
- What monitoring, logging, alerting, and GPU components need to be installed?
- How to verify if Prometheus successfully collects Kubernetes metrics?
- How to query Node CPU, memory, disk, and network metrics?
- How to query Pod CPU, memory, restarts, Pending, and OOMKilled metrics?
- Why can't we only check Service objects to determine if Service is normal?
- What's the relationship between Service, Endpoints, EndpointSlice, Ingress, and application metrics?
- How to troubleshoot Prometheus Target Down?
- How to locate issues when Grafana has no data?
- How to troubleshoot business anomalies when Pod and Service are normal?
- Where do Pod logs come from, and how to collect them via Loki/EFK?
- How to integrate Pod metrics, Pod Events, Pod logs, and Service backend status for troubleshooting?
- How to design alerts for Node/Pod/Service?
- How to convert monitoring results into production troubleshooting actions.

This document is suitable for study after completing the following content:

- 11-Prometheus-Architecture and Core Metric Analysis
- 12-Grafana-Dashboard Setup and Custom Monitoring
- 13-AlertManager-Alerting Strategy and Notification Implementation
- 15-Loki-Log Collection and Querying
- 16-ELK-EFK-Log Collection and Retrieval
- 17-Log Alerting and Automation Response
- 18-Monitoring-GPU-Log Integration Case-K8S-Pod Anomaly Detection and Report Generation

---

## Tags

#Kubernetes #Prometheus #Grafana #AlertManager #NodeExporter #kube-state-metrics #cAdvisor #MetricsServer #ServiceMonitor #PodMonitor #Service #Endpoints #EndpointSlice #PodLog #Loki #EFK #SRE #SurveillanceBarrier

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Foundation/14-K8S-Monitoring-Node-Pod-Service Metric Collection and Troubleshooting.md

---

## I. Core Objects of Kubernetes Monitoring

Kubernetes monitoring cannot only focus on "whether node CPU is high".

In production environments, at least three layers of objects need to be covered:

    Node
    Pod
    Service

Each represents different levels of issues.

### 1.1 What to Monitor for Node

Node is the foundation where workloads run.

Node monitoring focus:

- Is the node Ready?
- Is kubelet normal?
- Is containerd normal?
- CPU usage rate
- Memory usage rate
- Disk usage rate
- inode usage rate
- Network traffic
- Network error packets
- System load
- Pod count
- Container count
- Is there resource pressure on the node?
- Is the node cordoned?
- Does the node have taint?
- Is the node frequently NotReady?

Node anomalies can lead to:

- Pod scheduling failure
- Pod eviction
- Service backend reduction
- Business traffic jitter
- GPU/Database/Middleware Pod anomalies
- Unavailability of business on the entire node

### 1.2 What to Monitor for Pod

Pod is the smallest scheduling unit for business in Kubernetes.

Pod monitoring focus:

- Is the Pod Running?
- Is the Pod Pending?
- Is the Pod Failed?
- Is the Pod CrashLoopBackOff?
- Is the Pod frequently restarting?
- Container CPU usage
- Container memory usage
- OOMKilled
- Image pull failure
- Probe failure
- Pod's node
- Pod's namespace
- Pod's Deployment/StatefulSet/DaemonSet
- Is the Pod Ready?
- Is the Pod connected to Service?
- Is Pod log accessible?
- Is the last container log of the Pod accessible?

Common impacts of Pod anomalies:

- Insufficient replicas
- Insufficient Service backend
- Request failure
- Deployment failure
- HPA anomalies
- Increased business latency
- Service unavailability

### 1.3 What to Monitor for Service

Service is the entry point for service discovery and load balancing inside Kubernetes.

However, Service itself is not a business process.

Service monitoring focus:

- Does the Service exist?
- Is the selector correct?
- Are Endpoints/EndpointSlice available?
- Are the backend Pods Ready?
- Is the Service ClusterIP accessible?
- Is the Service port correct?
- Is the targetPort correct?
- Is kube-proxy normal?
- Is DNS resolution normal?
- Is Ingress/Gateway correctly forwarding?
- Is the application's QPS, error rate, and latency normal?

Notes:

    Service being normal does not mean business is normal.
    Pod Running does not mean business is normal.
    Having backend in Endpoints does not guarantee the application interface is normal.

Service troubleshooting must combine:

    Service
    EndpointSlice
    Pod Ready
    kube-proxy
    CoreDNS
    Ingress
    Application logs
    Business metrics

---

## II. Overview of Kubernetes Metric Sources

Kubernetes monitoring metrics come from multiple components.

Different components are responsible for metrics at different levels.

    +------------------------+
    | node-exporter          |
    | Host CPU/Memory/Disk/Network |
    +------------------------+

    +------------------------+
    | kubelet / cAdvisor     |
    | Container CPU/Memory/Network/Disk |
    +------------------------+

    +------------------------+
    | kube-state-metrics     |
    | K8S Object Status            |
    +------------------------+

    +------------------------+
    | kube-apiserver         |
    | API Server Metrics         |
    +------------------------+

    +------------------------+
    | kube-scheduler         |
    | Scheduler Metrics              |
    +------------------------+

    +------------------------+
    | kube-controller-manager|
    | Controller Metrics              |
    +------------------------+

    +------------------------+
    | Application /metrics          |
    | QPS/Error Rate/Delay/Business Metrics |
    +------------------------+

    +------------------------+
    | blackbox-exporter      |
    | HTTP/TCP/ICMP/DNS Probe |
    +------------------------+

    +------------------------+
    | dcgm-exporter          |
    | NVIDIA GPU Metrics         |
    +------------------------+

---

## III. Common Monitoring Component Responsibilities

### 3.1 node-exporter

node-exporter runs on each node, typically deployed as a DaemonSet.

It is responsible for collecting Linux host-level metrics.

Typical metrics:

    node_cpu_seconds_total
    node_memory_MemAvailable_bytes
    node_memory_MemTotal_bytes
    node_filesystem_avail_bytes
    node_filesystem_size_bytes
    node_network_receive_bytes_total
    node_network_transmit_bytes_total
    node_load1
    node_load5
    node_load15
    node_uname_info

Suitable answers:

- Is node CPU high?
- Is node memory insufficient?
- Is node disk full?
- Is node inode exhausted?
- Is node network traffic abnormal?
- Is node load too high?
- Is node filesystem read-only or space insufficient?

### 3.2 kubelet / cAdvisor

kubelet is responsible for managing Pods on nodes.

cAdvisor metrics are used to observe container resource usage.

Typical metrics:

    container_cpu_usage_seconds_total
    container_memory_usage_bytes
    container_memory_working_set_bytes
    container_network_receive_bytes_total
    container_network_transmit_bytes_total
    container_fs_usage_bytes

Suitable answers:

- How much CPU does a Pod use?
- How much memory does a Pod use?
- Is a container resource abnormal?
- Which Pods consume the most resources?
- Is Pod network traffic abnormal?
- Container filesystem usage status.

Notes:

    kubelet is a Kubernetes node component.
    cAdvisor-related metrics are typically exposed by kubelet.
    It is generally not necessary to install cAdvisor DaemonSet separately.

### 3.3 kube-state-metrics

kube-state-metrics listens to Kubernetes API Server, converting object status into metrics.

It does not collect resource utilization, but rather object "status".

Typical metrics:

    kube_node_status_condition
    kube_node_spec_unschedulable
    kube_pod_status_phase
    kube_pod_container_status_restarts_total
    kube_pod_container_status_last_terminated_reason
    kube_pod_container_status_ready
    kube_deployment_spec_replicas
    kube_deployment_status_replicas_available
    kube_service_info
    kube_endpoint_info
    kube_resourcequota
    kube_persistentvolumeclaim_status_phase

Suitable answers:

- Is Node Ready?
- Is Pod Pending?
- Is Pod Running?
- Has Pod restarted?
- Is Pod OOMKilled?
- Are Deployment replicas satisfied?
- Is PVC Pending?
- Is Service present?
- Is ResourceQuota approaching the limit?

### 3.4 Metrics Server

Metrics Server is part of the Kubernetes resource metrics pipeline.

It primarily serves:

    kubectl top
    HPA
    Some capabilities of VPA

Common commands: /think

kubectl top node  
kubectl top pod -A  

Metrics Server Focuses On:  

- Current CPU / Memory Usage of Nodes  
- Current CPU / Memory Usage of Pods  

However, Metrics Server is not suitable as a long-term monitoring system.  

Differences:  

    Metrics Server:  
        Used for Kubernetes real-time resource metrics API, primarily serving HPA and kubectl top.  

    Prometheus:  
        Used for long-term metric collection, querying, alerting, Dashboard, and capacity analysis.  

Do not treat Metrics Server as a replacement for Prometheus.  

### 3.5 Application /metrics  

Applications expose business metrics themselves.  

Examples:  

    http_requests_total  
    http_request_duration_seconds_bucket  
    http_request_errors_total  
    app_queue_length  
    app_request_inflight  
    model_inference_total  
    model_inference_latency_seconds_bucket  

Suitable for answering:  

- Whether service QPS is normal  
- Whether error rate has increased  
- Whether P95 / P99 latency has increased  
- Whether queues are accumulating  
- Whether the business is truly available  
- Whether there are requests for GPU inference services  
- Whether model loading was successful  

Service-layer business availability ultimately depends on application metrics and probe metrics to determine.  

---  

## FourI don't know.Experimental Environment Assumptions  

This article assumes the existence of a Kubernetes cluster:  

    k8s-master      10.0.0.20  
    k8s-worker01    10.0.0.21  
    k8s-worker02    10.0.0.22  
    k8s-gpu-node01  10.0.0.30  

Monitoring Namespace:  

    monitoring  

Logging Namespace:  

    logging  

Core Components:  

    Prometheus  
    AlertManager  
    Grafana  
    node-exporter  
    kube-state-metrics  
    kubelet/cAdvisor  
    Metrics Server  
    blackbox-exporter  
    Loki or Elasticsearch / OpenSearch  
    Grafana Alloy / Fluent Bit / Filebeat  
    dcgm-exporter  

If using kube-prometheus-stack, Prometheus, AlertManager, Grafana, node-exporter, kube-state-metrics, Prometheus Operator, and PrometheusRule will be deployed together with the Helm Chart.  

---  

## FiveI don't know.Deployment Method Selection  

Kubernetes monitoring commonly has two deployment methods.  

### 5.1 prometheus-community/prometheus  

Suitable For:  

- Learning  
- Small-scale experiments  
- Quick understanding of Prometheus  
- Manual configuration of scrape_configs  

Features:  

- Simple  
- Fewer components  
- Suitable for beginners  
- Less complete support for ServiceMonitor / PrometheusRule compared to Operator system  

### 5.2 kube-prometheus-stack  

Suitable For:  

- Production environments  
- Kubernetes-native monitoring  
- Prometheus Operator  
- ServiceMonitor  
- PodMonitor  
- PrometheusRule  
- AlertManager  
- Grafana  
- Default rules and Dashboard  

Production recommendation priority:  

    kube-prometheus-stack  

Reasons:  

- Complete components  
- Kubernetes-native  
- More suitable for multi-team management  
- More standardized alerting rules  
- Easier integration with ServiceMonitor  
- Easier expansion for GPU / middleware / application monitoring  

---  

## SixI don't know.Which Monitoring and Logging Components Need to Be Installed  

Kubernetes Node / Pod / Service monitoring and log collection cannot be completed by installing just one Prometheus. Instead, multiple types of components are required to work together.  

They can be categorized by function as follows:  

    Metrics Monitoring Components  
    Alerting Components  
    Grafana Visualization Components  
    Pod Log Collection Components  
    Log Storage and Query Components  
    GPU Monitoring Components  
    Automation and Notification Components  

### 6.1 Metrics Monitoring Components  

#### 6.1.1 Prometheus  

Role:  

    Metric collection  
    Metric storage  
    PromQL query  
    Alerting rule calculation  

Prometheus is the core of the entire metrics monitoring system.  

It is responsible for scraping metrics from the following targets:  

    node-exporter  
    kube-state-metrics  
    kubelet / cAdvisor  
    kube-apiserver  
    Application /metrics  
    blackbox-exporter  
    dcgm-exporter  

Prometheus addresses:  

    What metrics are abnormal?  
    How long has the anomaly persisted?  
    What is the trend of change?  
    Whether the alerting conditions are met?  

### 6.1.2 Prometheus Operator  

Role:  

    Managing Prometheus via Kubernetes CRD  
    Support for ServiceMonitor  
    Support for PodMonitor  
    Support for PrometheusRule  
    Support for AlertManager management  

If using kube-prometheus-stack, Prometheus Operator is typically automatically installed.  

Production environments recommend using Prometheus Operator instead of manually writing scrape_configs.  

Common CRDs:  

    Prometheus  
    Alertmanager  
    ServiceMonitor  
    PodMonitor  
    PrometheusRule  
    Probe  
    ThanosRuler  

### 6.1.3 kube-prometheus-stack  

Role:  

    One-time installation of Prometheus, AlertManager, Grafana, node-exporter, kube-state-metrics, Prometheus Operator, default rules, and some Dashboards.  

Recommended method: /think

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack

It is not a single plugin, but a complete set of Kubernetes monitoring components.

It is recommended to prioritize using it in both production and learning environments.

### 6.1.4 node-exporter

Purpose:

    Collects Linux node-level metrics.

Typical metrics:

    CPU
    Memory
    Disk
    inode
    Network
    Load
    File system
    Host information

Common deployment method:

    DaemonSet

Runs one node-exporter on each node.

### 6.1.5 kube-state-metrics

Purpose:

    Collects Kubernetes object status metrics.

Typical objects:

    Node
    Pod
    Deployment
    StatefulSet
    DaemonSet
    Service
    PVC
    ResourceQuota
    Namespace
    Job
    CronJob

Typical metrics:

    kube_pod_status_phase
    kube_pod_container_status_restarts_total
    kube_pod_container_status_last_terminated_reason
    kube_node_status_condition
    kube_deployment_status_replicas_available

Note:

    kube-state-metrics does not collect CPU/memory usage rates.
    It collects Kubernetes object status metrics.

### 6.1.6 kubelet / cAdvisor

Purpose:

    Collects container resource usage metrics.

Typical metrics:

    container_cpu_usage_seconds_total
    container_memory_working_set_bytes
    container_network_receive_bytes_total
    container_network_transmit_bytes_total
    container_fs_usage_bytes

Notes:

    kubelet is a Kubernetes node component.
    cAdvisor-related metrics are typically exposed by kubelet.
    It is generally not necessary to install cAdvisor DaemonSet separately.

### 6.1.7 Metrics Server

Purpose:

    Provides real-time resource metrics for kubectl top, HPA, VPA.

Common commands:

    kubectl top node
    kubectl top pod -A

Note:

    Metrics Server is not a replacement for Prometheus.
    It is unsuitable for long-term monitoring, Dashboard, and alerts.
    However, it is still recommended to install it in production clusters.

### 6.1.8 blackbox-exporter

Purpose:

    Performs HTTP / TCP / ICMP / DNS probing.

Suitable for monitoring:

    Ingress addresses
    NodePort addresses
    Internal HTTP APIs
    TCP ports
    DNS resolution
    TLS certificate validity period

It is used to supplement Service and business entry availability monitoring.

---

### 6.2 Alerting Components

#### 6.2.1 AlertManager

Purpose:

    Alert grouping
    Alert deduplication
    Alert suppression
    Alert silencing
    Alert routing
    Notification sending

AlertManager receives alerts generated by Prometheus, then sends them to:

    Email
    Webhook
    Enterprise WeChat
    DingTalk
    Feishu
    OnCall system
    Ticketing system

AlertManager is not responsible for calculating metrics; it handles already triggered alerts.

### 6.2.2 PrometheusRule

Purpose:

    Manages Prometheus alerting rules and Recording Rules via Kubernetes CRD.

Example alerts:

    NodeNotReady
    PodPendingTooLong
    PodRestartTooOften
    PodOOMKilled
    ServiceHigh5xxErrorRate
    PrometheusTargetDown

If using kube-prometheus-stack, it is recommended to manage alerts via PrometheusRule rather than manually editing Prometheus configuration files.

---

### 6.3 Grafana Visualization Components

#### 6.3.1 Grafana

Purpose:

    Metric Dashboard
    Log query entry
    Alert display
    Unified visualization of multiple data sources

Common data sources:

    Prometheus
    Loki
    Elasticsearch
    OpenSearch
    MySQL
    PostgreSQL

In Kubernetes monitoring systems, Grafana typically handles:

    Viewing trends
    Viewing aggregations
    Viewing multi-dimensional filtering
    Viewing top anomalies
    Jumping from Dashboard to logs
    Jumping from alerts to troubleshooting views

### 6.3.2 Grafana Data Sources

Prometheus data source:

    Used for querying Prometheus metrics.

Loki data source:

    Used for querying Pod logs.

Elasticsearch / OpenSearch data source:

    Used for querying EFK logs.

Note:

    Prometheus and Loki are common built-in data sources for Grafana.
    It is generally not necessary to install third-party Grafana plugins.
    The key is to configure Data Source addresses and permissions.

---

### 6.4 Pod Log Collection Components

Pod log collection is not handled by Prometheus.

Prometheus collects metrics, not logs.

Pod log collection requires deploying a separate log agent.

### 6.4.1 Grafana Alloy

Purpose:

    Collects Kubernetes Pod logs
    Collects metrics
    Collects traces
    Forwards to backend systems like Loki / Prometheus / Tempo

New environments are recommended to prioritize using:

    Grafana Alloy + Loki

Notes:

    Alloy is the recommended new-generation collection agent in the Grafana ecosystem.
    It can serve as an alternative to Promtail.

### 6.4.2 Promtail

Purpose:

    Collects Pod logs and sends them to Loki.

Typical Chain:

    /var/log/containers
      ↓
    Promtail
      ↓
    Loki
      ↓
    Grafana

Notes:

    Promtail is a commonly used log collector for Loki in earlier versions.
    New environments are recommended to use Alloy instead.
    Old environments may still have a large number of Promtail instances.

### 6.4.3 Fluent Bit

Purpose:

    Lightweight log collection and forwarding.

Suitable For:

    Kubernetes container log collection
    Output to Elasticsearch
    Output to OpenSearch
    Output to Loki
    Output to Kafka

Deployment Method:

    DaemonSet

Common Log Paths:

    /var/log/containers/*.log

Fluent Bit is very suitable as a log collector for Kubernetes clusters.

### 6.4.4 Filebeat

Purpose:

    Lightweight log collector in the Elastic ecosystem.

Suitable For:

    Filebeat DaemonSet
      ↓
    Elasticsearch / Logstash / Kafka
      ↓
    Kibana

If an enterprise uses ELK, Filebeat is very common.

### 6.4.5 Fluentd / Logstash

Purpose:

    Log aggregation, filtering, parsing, and transformation.

Suitable For:

    Complex log processing
    grok parsing
    Field cleaning
    Multiple outputs
    Large log pipelines

Common Chain:

    Fluent Bit / Filebeat
      ↓
    Fluentd / Logstash
      ↓
    Elasticsearch / OpenSearch

---

### 6.5 Log Storage and Query Components

### 6.5.1 Loki

Purpose:

    Stores and queries Pod logs.

Features:

    Primary indexed labels
    Does not default to full-text indexing of log content
    Well-integrated with Grafana
    Suitable for Kubernetes operations and troubleshooting logs

Typical Chain:

    Pod stdout / stderr
      ↓
    Alloy / Promtail
      ↓
    Loki
      ↓
    Grafana Explore

### 6.5.2 Elasticsearch / OpenSearch

Purpose:

    Full-text search for logs
    Field queries
    Aggregation analysis
    Audit logs
    Security logs
    Business log search

Typical Chain:

    Pod stdout / stderr
      ↓
    Filebeat / Fluent Bit
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 6.5.3 Kibana / OpenSearch Dashboards

Purpose:

    Log query
    Log search
    Log Dashboard
    Log alerts
    Field analysis

If an enterprise already has ELK / OpenSearch, Kubernetes clusters typically only need to deploy log agents and forward logs to the existing log platform.

---

### 6.6 GPU Monitoring Components

If the cluster has GPU nodes, additional GPU-related components need to be installed.

### 6.6.1 NVIDIA Driver

Purpose:

    Base driver for node recognition and use of NVIDIA GPUs.

Installation Location:

    GPU node host

### 6.6.2 NVIDIA Container Toolkit

Purpose:

    Allows containers to use NVIDIA GPUs.

Without it, containers cannot access GPU devices normally.

### 6.6.3 NVIDIA Device Plugin

Purpose:

    Registers GPU resources to Kubernetes.

After installation, the node will have:

    nvidia.com/gpu

Pods can request GPUs like this:

    resources:
      limits:
        nvidia.com/gpu: 1

### 6.6.4 DCGM Exporter

Purpose:

    Exposes GPU metrics to Prometheus.

Common Metrics:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE
    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_POWER_USAGE
    DCGM_FI_DEV_XID_ERRORS

Used for monitoring:

    GPU utilization
    GPU memory
    GPU temperature
    GPU power consumption
    GPU XID errors

### 6.6.5 NVIDIA GPU Operator

Purpose:

    Automates management of GPU node-related components.

May include:

    NVIDIA Driver
    NVIDIA Container Toolkit
    NVIDIA Device Plugin
    DCGM Exporter
    GPU Feature Discovery

Notes:

    If using GPU Operator, many GPU components can be installed and managed uniformly.
    If not using GPU Operator, Device Plugin and DCGM Exporter can be manually installed.

---

### 6.7 Automation and Notification Components

### 6.7.1 Webhook Alert Gateway

Purpose:

    Receives AlertManager Webhook
    Formats messages uniformly
    Integrates with WeCom / DingTalk / Feishu
    Triggers automated diagnostics
    Logs alert audit

Common Chain:

    AlertManager
      ↓
    Webhook Gateway
      ↓
    WeCom / DingTalk / Feishu / Ticket / OnCall

### 6.7.2 Runbook Documentation Library

Purpose:

    Guides handling steps after alert triggers.

Each important alert should have:

    runbook_url

Examples:

    PodPendingTooLong
    PodRestartTooOften
    PodOOMKilled
    ServiceHigh5xxErrorRate
    NodeNotReady
    GPUXIDError

### 6.7.3 Automated Diagnostic Service

Purpose:

Receive Alerts
Query Kubernetes API
Query Prometheus
Query Loki / ELK
Generate Diagnostic Report
Push to On-Call Personnel

Auto-diagnosis should default to read-only and should not directly modify production resources.

---

### 6.8 Minimal Installation Components

### 6.8.1 Monitoring Kubernetes Basic Metrics Only

At least requires:

    kube-prometheus-stack
    Prometheus
    AlertManager
    Grafana
    node-exporter
    kube-state-metrics
    kubelet / cAdvisor

Optional:

    Metrics Server
    blackbox-exporter

### 6.8.2 Collecting Pod Logs with Loki

Requires:

    Loki
    Grafana Alloy or Promtail
    Grafana Loki Data Source

Recommended:

    Grafana Alloy + Loki + Grafana

### 6.8.3 Collecting Pod Logs with EFK

Requires:

    Elasticsearch or OpenSearch
    Kibana or OpenSearch Dashboards
    Filebeat or Fluent Bit

Optional:

    Logstash
    Kafka

### 6.8.4 Monitoring GPU

Requires:

    NVIDIA Driver
    NVIDIA Container Toolkit
    NVIDIA Device Plugin or GPU Operator
    DCGM Exporter
    Prometheus
    Grafana

### 6.8.5 Full Production Observability

Recommended combination:

    kube-prometheus-stack
    Metrics Server
    blackbox-exporter
    Loki
    Grafana Alloy
    or Elasticsearch / OpenSearch + Fluent Bit
    AlertManager
    Grafana
    DCGM Exporter
    Webhook Alert Gateway
    Runbook Documentation
    Auto-Diagnosis Service

### 6.9 Key Understandings

Must clearly distinguish:

    Prometheus:
        Collects metrics, does not collect logs.

    Loki / ELK:
        Collects and queries logs, does not replace Prometheus metric monitoring.

    Metrics Server:
        Supports kubectl top / HPA, does not replace Prometheus.

    kube-state-metrics:
        Collects K8S object status, does not collect CPU / memory usage.

    kubelet / cAdvisor:
        Provides container resource usage metrics.

    node-exporter:
        Collects host resource metrics.

    AlertManager:
        Handles and sends alerts, does not collect metrics.

    Grafana:
        Displays metrics and logs, does not collect.

---

## Seven. Installing kube-prometheus-stack

### 7.1 Add Helm Repository

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update

Check versions:

    helm search repo prometheus-community/kube-prometheus-stack --versions

### 7.2 Create Namespace

    kubectl create namespace monitoring

### 7.3 Export Default Values

    helm show values prometheus-community/kube-prometheus-stack \
      --version <CHART_VERSION> > values-kube-prometheus-stack.yaml

Notes:

    <CHART_VERSION> should be obtained via helm search repo.
    Production environments must fix the version.

### 7.4 Install

    helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --version <CHART_VERSION> \
      -f values-kube-prometheus-stack.yaml

### 7.5 View Components

    kubectl get pods -n monitoring -o wide
    kubectl get svc -n monitoring
    kubectl get ds -n monitoring
    kubectl get prometheus -n monitoring
    kubectl get alertmanager -n monitoring
    kubectl get servicemonitor -n monitoring
    kubectl get prometheusrule -n monitoring

---

## Eight. Verifying Prometheus Collection Status

### 8.1 Access Prometheus

Check Service:

    kubectl get svc -n monitoring | grep prometheus

Port forward:

    kubectl port-forward svc/<prometheus-service-name> 9090:9090 -n monitoring

Access:

    http://127.0.0.1:9090

### 8.2 Check Targets

Prometheus Page:

    Status
      ↓
    Targets

Focus on checking:

    kubelet
    node-exporter
    kube-state-metrics
    apiserver
    scheduler
    controller-manager
    prometheus
    alertmanager

Status should be:

    UP

If DOWN appears, refer to the troubleshooting section.

### 8.3 Basic Metric Verification

Enter the following queries sequentially in the Prometheus query page:

    up

node_cpu_seconds_total

node_memory_MemAvailable_bytes

container_cpu_usage_seconds_total

kube_pod_status_phase

kube_node_status_condition

If all have data, it indicates that the basic collection pipeline is normal.

---

## IX. Node Metrics Collection Practice

### 9.1 Node Ready Status

Metrics source:

    kube-state-metrics

PromQL:

    kube_node_status_condition{condition="Ready", status="true"}

Value of 1:

    Node Ready

Value of 0:

    Node NotReady

Check NotReady nodes:

    kube_node_status_condition{condition="Ready", status="true"} == 0

### 9.2 Whether Node is Cordoned

Metrics:

    kube_node_spec_unschedulable

Query:

    kube_node_spec_unschedulable == 1

Indicates the node is set to unschedulable.

Corresponding command:

    kubectl get nodes

If the node shows:

    SchedulingDisabled

It indicates the node is cordoned.

### 9.3 Node CPU Usage Rate

Metrics source:

    node-exporter

Original metric:

    node_cpu_seconds_total

PromQL:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

Note:

    node_cpu_seconds_total is a Counter.
    Use rate() function.
    mode="idle" represents idle CPU.
    100 - idle% gives CPU usage rate.

By node:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

### 9.4 Node Memory Usage Rate

Metrics source:

    node-exporter

PromQL:

    (
      1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
    ) * 100

Note:

    MemAvailable is more suitable for judging available memory than MemFree.
    Linux uses caching, so only free memory should not be considered.

### 9.5 Node Disk Usage Rate

PromQL:

    (
      1 - node_filesystem_avail_bytes{fstype!="tmpfs"}
      / node_filesystem_size_bytes{fstype!="tmpfs"}
    ) * 100

Recommended filtering:

    tmpfs
    overlay
    proc
    sysfs

More rigorous example:

    (
      1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
      / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
    ) * 100

### 9.6 Node inode Usage Rate

PromQL:

    (
      1 - node_filesystem_files_free{fstype!~"tmpfs|overlay|proc|sysfs"}
      / node_filesystem_files{fstype!~"tmpfs|overlay|proc|sysfs"}
    ) * 100

Full inodes will cause:

- Pod unable to create files;
- Log writing failure;
- Image pull failure;
- Application anomalies;
- kubelet anomalies.

Do not monitor disk capacity alone; also monitor inodes.

### 9.7 Node Network Receive Rate

PromQL:

    rate(node_network_receive_bytes_total{device!~"lo"}[5m])

By node aggregation:

    sum by (instance) (
      rate(node_network_receive_bytes_total{device!~"lo"}[5m])
    )

### 9.8 Node Network Transmit Rate

PromQL:

    sum by (instance) (
      rate(node_network_transmit_bytes_total{device!~"lo"}[5m])
    )

### 9.9 Node Network Error Packets

Receive errors:

    rate(node_network_receive_errs_total{device!~"lo"}[5m])

Transmit errors:

    rate(node_network_transmit_errs_total{device!~"lo"}[5m])

Growing network error packets require investigation:

- Network card;
- Driver;
- Switch;
- MTU;
- Packet loss;
- Duplex mode;
- Physical link;
- Virtual network.

### 9.10 Node Load

PromQL:

    node_load1
    node_load5
    node_load15

Note:

    Load should be judged in combination with CPU core count.
    Load 8 on an 8-core machine and Load 8 on a 2-core machine have different meanings.

Combined with CPU core count:

    count by (instance) (node_cpu_seconds_total{mode="idle"})

---

## X. Node Troubleshooting Practice

### 10.1 Phenomenon: Node NotReady

PromQL:

    kube_node_status_condition{condition="Ready", status="true"} == 0

kubectl:

    kubectl get nodes
    kubectl describe node <node-name>

Node side: /think

systemctl status kubelet  
systemctl status containerd  
journalctl -u kubelet -f  
journalctl -u containerd -f  

Common Causes:  

- kubelet anomalies;  
- containerd anomalies;  
- node network anomalies;  
- node disk full;  
- node memory pressure;  
- CNI anomalies;  
- certificate issues;  
- API Server connection failure;  
- node reboot;  
- cloud instance or physical machine failure.  

Troubleshooting:  

    1. Check Node Conditions  
    2. Check kubelet logs  
    3. Check containerd status  
    4. Check disk and memory  
    5. Check network connectivity  
    6. Cordon node if necessary  
    7. Migrate workloads  
    8. Uncordon node after repair  

### 10.2 Phenomenon: Node CPU high for a long time  

PromQL:  

    100 - (  
      avg by (instance) (  
        rate(node_cpu_seconds_total{mode="idle"}[5m])  
      ) * 100  
    ) > 90  

Troubleshooting:  

    kubectl top node  
    kubectl top pod -A --sort-by=cpu  
    top  
    pidstat 1  
    ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head  

Judgment:  

- Business Pod usage;  
- System process usage;  
- kubelet/containerd anomalies;  
- Log collection process usage;  
- Non-K8S processes running on the node;  
- CPU request too low causing contention.  

Handling:  

- Scale out replicas;  
- Adjust resource requests/limits;  
- Migrate Pods;  
- Limit abnormal tasks;  
- Investigate system processes;  
- Isolate the node if necessary.  

### 10.3 Phenomenon: Node memory high  

PromQL:  

    (  
      1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes  
    ) * 100 > 90  

Troubleshooting:  

    kubectl top node  
    kubectl top pod -A --sort-by=memory  
    free -h  
    ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head  

Check for evictions:  

    kubectl describe node <node-name> | grep -i memory -A10  
    kubectl get events -A --sort-by=.lastTimestamp | grep -i evict  

Common Causes:  

- High Pod memory usage;  
- Memory leaks;  
- System process anomalies;  
- Page Cache misjudgment;  
- High Pod density on the node;  
- Unreasonable requests settings;  
- BestEffort Pods being evicted.  

Handling:  

- Optimize Pod memory;  
- Set reasonable requests/limits;  
- Investigate memory leaks;  
- Reduce node load;  
- Adjust eviction thresholds cautiously;  
- Add node resources.  

### 10.4 Phenomenon: Node disk full  

PromQL:  

    (  
      1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}  
      / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}  
    ) * 100 > 85  

Node Troubleshooting:  

    df -h  
    df -i  
    du -xh /var/lib/containerd | sort -h | tail  
    du -xh /var/log | sort -h | tail  
    crictl images  
    crictl ps -a  

Common Causes:  

- Large container logs;  
- Unremoved images;  
- Large containerd data directory;  
- High emptyDir usage;  
- Application writing local files;  
- Prometheus/log system local disk growth;  
- Large system logs on the node.  

Handling:  

- Clean up unused images;  
- Configure log rotation;  
- Limit emptyDir;  
- Adjust application log output;  
- Migrate local data;  
- Expand disk;  
- Set disk alerts and inspections.  

---

## ElevenI don't know.Pod Metrics Collection Practice  

### 11.1 Current Pod Status  

Metrics Source:  

    kube-state-metrics  

Query Pending:  

    kube_pod_status_phase{phase="Pending"} == 1  

Query Running:  

    kube_pod_status_phase{phase="Running"} == 1  

Query Failed:  

    kube_pod_status_phase{phase="Failed"} == 1  

Count Pending Pods by Namespace:  

    sum by (namespace) (  
      kube_pod_status_phase{phase="Pending"} == 1  
    )  

### 11.2 Pod Ready Status  

Metrics:  

    kube_pod_container_status_ready  

Query Unready Containers:  

    kube_pod_container_status_ready == 0  

### 11.3 Pod CPU Usage  

Metrics Source:  

    kubelet / cAdvisor  

PromQL:  

    sum by (namespace, pod) (  
      rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])  
    )  

Unit:  

    CPU cores  

Example:  

    0.5 represents approximately 0.5 cores.  

### 11.4 Pod Memory Usage  

PromQL:  

    sum by (namespace, pod) (  
      container_memory_working_set_bytes{container!="", image!=""}  
    )  

Recommended Use:  

    container_memory_working_set_bytes  

It is more suitable for observing actual working set compared to container_memory_usage_bytes.  

### 11.5 Pod Restart Count  

Metrics Source: /think

# kube-state-metrics

PromQL:

    kube_pod_container_status_restarts_total

Recent 10-minute restarts increase:

    increase(kube_pod_container_status_restarts_total[10m])

Top 10 restarted Pods:

    topk(10,
      increase(kube_pod_container_status_restarts_total[10m])
    )

### 11.6 OOMKilled

Metrics:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

Query:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

Note:

    This metric indicates the reason for the last termination.
    Troubleshooting still requires combining with kubectl describe pod and logs.

### 11.7 Pod Network Receive

PromQL:

    sum by (namespace, pod) (
      rate(container_network_receive_bytes_total{pod!=""}[5m])
    )

### 11.8 Pod Network Transmit

PromQL:

    sum by (namespace, pod) (
      rate(container_network_transmit_bytes_total{pod!=""}[5m])
    )

### 11.9 Pod CPU Request

Metrics source:

    kube-state-metrics

PromQL:

    sum by (namespace, pod) (
      kube_pod_container_resource_requests{resource="cpu"}
    )

Resource metric names may vary across different kube-state-metrics versions; use the actual metrics in the current environment.

### 11.10 Pod Memory Request

PromQL:

    sum by (namespace, pod) (
      kube_pod_container_resource_requests{resource="memory"}
    )

---

## TwelveI don't know.Pod Troubleshooting in Practice

### 12.1 Phenomenon: Pod Pending

PromQL:

    kube_pod_status_phase{phase="Pending"} == 1

kubectl:

    kubectl get pod -n <namespace> -o wide
    kubectl describe pod <pod-name> -n <namespace>

Focus on Events.

Common causes:

- CPU insufficient;
- Memory insufficient;
- GPU insufficient;
- PVC Pending;
- nodeSelector mismatch;
- nodeAffinity mismatch;
- taint/toleration mismatch;
- image pull failure;
- ResourceQuota exceeded;
- LimitRange restriction;
- scheduler anomaly.

Typical events:

    insufficient cpu
    insufficient memory
    insufficient nvidia.com/gpu
    node(s) had untolerated taint
    node(s) didn't match Pod's node affinity/selector
    pod has unbound immediate PersistentVolumeClaims
    exceeded quota

Handling methods:

    1. kubectl describe pod to check Events
    2. Determine if it's resource, storage, scheduling constraints, or quota
    3. Check Node resources
    4. Check PVC
    5. Check Namespace ResourceQuota
    6. Correct YAML or expand resources

### 12.2 Phenomenon: Pod CrashLoopBackOff

Prometheus query for restarts:

    increase(kube_pod_container_status_restarts_total[10m]) > 3

kubectl:

    kubectl get pod <pod-name> -n <namespace>
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

Common causes:

- Application startup failure;
- Configuration file error;
- Environment variable error;
- Port conflict;
- Dependency service unreachable;
- Database connection failure;
- Permission error;
- livenessProbe too aggressive;
- Image entry command error;
- Repeated restarts after OOMKilled.

Handling:

- First check `--previous` logs;
- Check Events;
- Check ConfigMap / Secret;
- Check probes;
- Check dependency services;
- Check resource limits;
- Roll back image or fix configuration.

### 12.3 Phenomenon: Pod OOMKilled

PromQL:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

kubectl:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

Check resources:

    kubectl top pod <pod-name> -n <namespace>

Common causes:

- Memory limit set too low;
- Application memory leak;
- Sudden large request;
- JVM heap configuration unreasonable;
- Python process memory growth;
- Cache infinite growth;
- Large file loading;
- Batch size too big.

Handling:

- Analyze memory trends;
- Adjust limit;
- Optimize program;
- Limit concurrency;
- Adjust JVM / application parameters;
- Combine with logs to locate memory spikes before requests or tasks.

### 12.4 Phenomenon: Pod High CPU

PromQL:

```markdown
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
)

View Top:

  topk(10,
    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
    )
  )

kubectl:

  kubectl top pod -A --sort-by=cpu

Troubleshooting Approach:

- Determine if it's a business peak;
- Determine if there's a dead loop;
- Determine if there's GC anomaly;
- Determine if there's log flooding;
- Determine if CPU request is too low;
- Determine if replica scaling is needed;
- Determine if rate limiting is needed.

---

## ThirteenI don't know.Service Metric Collection and Understanding

Service itself is not equal to business availability.

Service is merely a stable entry point and load balancing rule.

Service-related monitoring should be combined with the following objects:

    Service
    EndpointSlice / Endpoints
    Pod Ready
    kube-proxy
    CoreDNS
    Ingress / Gateway
    Application /metrics
    Blackbox Probe

### 13.1 Service Object Metrics

kube-state-metrics may expose:

    kube_service_info
    kube_service_created
    kube_service_spec_type
    kube_service_spec_cluster_ip

Query Service:

    kube_service_info

View by Namespace:

    kube_service_info{namespace="default"}

### 13.2 Endpoint Metrics

Possible metrics:

    kube_endpoint_info

Or EndpointSlice-related metrics, depending on the kube-state-metrics version.

View Endpoints:

    kubectl get endpoints -n <namespace>
    kubectl get endpointslice -n <namespace>

If Service has no Endpoints, common reasons:

- Selector is written incorrectly;
- Pod label does not match;
- Pod is not Ready;
- Pod does not exist;
- Service and Pod are not in the same Namespace;
- targetPort mismatch does not necessarily cause empty Endpoint, but may cause connection failure.

### 13.3 Service Backend Pod Readiness

View:

    kubectl get pod -n <namespace> --show-labels
    kubectl describe svc <service-name> -n <namespace>
    kubectl get endpoints <service-name> -n <namespace>
    kubectl get endpointslice -n <namespace> | grep <service-name>

### 13.4 Service Business Metric Sources

The true business metrics at the Service layer usually come from the application itself.

For example:

    http_requests_total
    http_request_duration_seconds_bucket
    http_requests_errors_total

QPS:

    sum by (service) (
      rate(http_requests_total[5m])
    )

Error Rate:

    sum by (service) (
      rate(http_requests_total{status=~"5.."}[5m])
    )
    /
    sum by (service) (
      rate(http_requests_total[5m])
    )
    * 100

P95 Latency:

    histogram_quantile(
      0.95,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket[5m])
      )
    )

If the application does not expose business metrics, it's difficult to judge the service's real quality from Prometheus.

### 13.5 Blackbox Probe

blackbox-exporter can be used to probe from the outside:

- HTTP returns 200;
- TCP port is reachable;
- DNS resolution is successful;
- ICMP is reachable;
- TLS certificate is normal.

Suitable for supplementing Service availability monitoring.

Example Scenarios:

    Probe Ingress address
    Probe Service NodePort
    Probe internal HTTP API
    Probe database port
    Probe DNS resolution

---

## FourteenI don't know.Service Troubleshooting in Practice

### 14.1 Phenomenon: Service exists but has no backend

View:

    kubectl get svc -n <namespace>
    kubectl describe svc <service-name> -n <namespace>
    kubectl get endpoints <service-name> -n <namespace>
    kubectl get endpointslice -n <namespace> | grep <service-name>

Common reasons:

- Service selector is wrong;
- Pod label does not match;
- Pod is not Ready;
- Pod is in another Namespace;
- Pod is deleted;
- Deployment replica count is 0;
- readinessProbe failed.

Handling:

    1. View Service selector
    2. View Pod label
    3. View Pod Ready status
    4. View readinessProbe
    5. Correct label or selector

Example: /think
```

kubectl describe svc my-service -n test
kubectl get pod -n test --show-labels
kubectl get endpoints my-service -n test

### 14.2 Phenomenon: Service Has Endpoints, but Access Fails

Troubleshooting:

    kubectl get endpoints <service-name> -n <namespace>
    kubectl get pod -n <namespace> -o wide
    kubectl describe svc <service-name> -n <namespace>

Testing:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside the container:

    curl http://<service-name>.<namespace>.svc.cluster.local:<port>

Common Causes:

- targetPort is incorrect;
- Container is not listening on the corresponding port;
- Application only listens on 127.0.0.1;
- NetworkPolicy blocks traffic;
- Pod is Ready but internal application has issues;
- kube-proxy is abnormal;
- CoreDNS is abnormal;
- Application protocol mismatch;
- HTTP/HTTPS configuration errors.

### 14.3 Phenomenon: Service DNS Resolution Failure

Testing Pod:

    kubectl run dns-test --rm -it --image=busybox:1.36 -- sh

Inside the container:

    nslookup kubernetes.default.svc.cluster.local
    nslookup <service-name>.<namespace>.svc.cluster.local

Troubleshooting CoreDNS:

    kubectl get pods -n kube-system | grep coredns
    kubectl logs -n kube-system <coredns-pod>
    kubectl get svc -n kube-system kube-dns

Common Causes:

- CoreDNS Pod is abnormal;
- kube-dns Service is abnormal;
- Pod DNSPolicy is abnormal;
- NodeLocal DNSCache is abnormal;
- NetworkPolicy blocks DNS;
- /etc/resolv.conf is abnormal;
- Service name or Namespace is incorrect.

### 14.4 Phenomenon: Service is Normal, Pod is Normal, but Business is Abnormal

This is a very common issue in production environments.

Troubleshooting Order:

    1. Is Service selector correct?
    2. Are Endpoints correct?
    3. Is Pod Ready?
    4. Is container port listening?
    5. Has application health check passed?
    6. Curl local port inside container
    7. Curl Service within cluster
    8. Cross Namespace access test
    9. Ingress / Gateway forwarding
    10. Application logs
    11. Business metrics QPS / error rate / latency
    12. NetworkPolicy
    13. kube-proxy / CNI

Common Commands:

    kubectl describe svc <service-name> -n <namespace>
    kubectl get endpoints <service-name> -n <namespace>
    kubectl get pod -n <namespace> -o wide
    kubectl logs <pod-name> -n <namespace>
    kubectl exec -it <pod-name> -n <namespace> -- ss -lntp

Temporary Testing:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

---

## FifteenI don't know.Prometheus Target Troubleshooting

### 15.1 Phenomenon: Target is DOWN

Prometheus Page:

    Status
      ↓
    Targets

See a target:

    DOWN

Troubleshooting Order:

    1. Is target Pod Running?
    2. Does target Service exist?
    3. Does Service selector match Pod?
    4. Are there backend endpoints?
    5. Does port name match?
    6. Is /metrics accessible?
    7. Does ServiceMonitor match?
    8. Is NetworkPolicy blocking?
    9. Does Prometheus RBAC have permissions?
    10. Are TLS / Bearer Token configurations correct?

### 15.2 Check ServiceMonitor

    kubectl get servicemonitor -A
    kubectl describe servicemonitor <name> -n <namespace>

Check:

- namespaceSelector;
- selector.matchLabels;
- endpoints.port;
- endpoints.path;
- interval;
- Whether labels are selected by Prometheus.

### 15.3 Check Service Labels

    kubectl get svc <service-name> -n <namespace> --show-labels
    kubectl describe svc <service-name> -n <namespace>

ServiceMonitor's selector must match Service's labels.

### 15.4 Check Port Name

ServiceMonitor typically uses port name, not port number.

Service Example:

    ports:
      - name: metrics
        port: 9100
        targetPort: 9100

ServiceMonitor:

    endpoints:
      - port: metrics

If port name is wrong, Prometheus may not scrape.

### 15.5 Test /metrics

Temporary Pod:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Access:

    curl http://<service-name>.<namespace>.svc.cluster.local:<port>/metrics

If access fails, it indicates that the issue is not with Prometheus, but rather with target exposure or network problems.

---

## SixteenI don't know.Grafana No Data Troubleshooting

### 16.1 Troubleshooting Order

When Grafana has no data, do not directly modify the Dashboard.

Troubleshoot in this order:

    1. Is the time range correct?
    2. Is the Prometheus data source functioning normally?
    3. Can metrics be queried in Prometheus?
    4. Is the PromQL correct?
    5. Do the Labels match?
    6. Are the variables empty?
    7. Is the Target UP?
    8. Are the metrics relabeled/dropped?
    9. Has the Dashboard used an outdated metric name?

### 16.2 Direct Query in Prometheus

For example, when Grafana has no data for GPU utilization:

First query in Prometheus:

    DCGM_FI_DEV_GPU_UTIL

If Prometheus also has no data, it indicates that the issue is not with Grafana.

If Prometheus has data, check the Grafana PromQL and variables.

### 16.3 Check Variables

Dashboard variables may be empty.

For example:

    $namespace
    $pod
    $node

The variable query may be written incorrectly:

    label_values(kube_pod_info, namespace)

In Grafana:

    Dashboard Settings
      ↓
    Variables
      ↓
    Preview of values

Check if there are values.

### 16.4 Check Label Names

The node labels for different metrics may vary:

    instance
    node
    Hostname

Do not assume all metrics have the label "node."

First query the original metric:

    node_cpu_seconds_total

    kube_node_info

    DCGM_FI_DEV_GPU_UTIL

Check the actual labels.

---

## SeventeenI don't know.Node Dashboard Design

### 17.1 Top Overview

Recommended panels:

    Total Nodes
    Ready Node Count
    NotReady Node Count
    Cordon Node Count
    Average CPU Usage
    Average Memory Usage
    High Disk Usage Node Count
    Network Error Node Count

### 17.2 Node Resource Trends

Recommended panels:

    Node CPU Usage
    Node Memory Usage
    Node Disk Usage
    Node inode Usage
    Node Network Receive Rate
    Node Network Send Rate
    Node Load

### 17.3 Node Anomaly List

Table:

    NotReady Node
    CPU Top 10 Node
    Memory Top 10 Node
    Disk Top 10 Node
    inode Top 10 Node
    Network Error Top Node
    Cordon Node List

---

## EighteenI don't know.Pod Dashboard Design

### 18.1 Top Overview

Recommended panels:

    Total Pods
    Running Pod Count
    Pending Pod Count
    Failed Pod Count
    Restart Pod Count
    OOMKilled Pod Count
    CPU Top Pod
    Memory Top Pod

### 18.2 Namespace Dimension

Recommended panels:

    Namespace CPU Usage
    Namespace Memory Usage
    Namespace Pod Count
    Namespace Pending Pod
    Namespace Restart Count
    Namespace ResourceQuota Usage

### 18.3 Pod Details

Recommended Table:

    Namespace
    Pod
    Node
    Phase
    Ready
    Restart Count
    CPU Usage
    Memory Usage
    Last Terminated Reason

### 18.4 Pod Log Entry

A Pod Dashboard should not only display metrics.

Recommended additions:

    Recent Error Logs
    Recent Warning Logs
    Loki Explore Jump
    Kibana Discover Jump
    Pod Events Panel
    Pod's Service / Endpoint Information

This allows direct navigation from Pod metric anomalies to log context.

---

## NineteenI don't know.Service Dashboard Design

### 19.1 Service Overview

Recommended panels:

    Total Services
    ClusterIP Service Count
    NodePort Service Count
    LoadBalancer Service Count
    Services Without Endpoints
    Ingress Count
    Blackbox Probe Success Rate

### 19.2 Backend Status of Service

Recommended Table:

    Namespace
    Service
    Type
    ClusterIP
    Port
    Endpoint Count
    Ready Pod Count

### 19.3 Business Quality Metrics

If the application exposes metrics, recommended panels:

    QPS
    5xx Error Rate
    P95 Latency
    P99 Latency
    Request Success Rate
    Current Concurrency
    Queue Length

### 19.4 External Probe Metrics

blackbox-exporter panel:

    probe_success
    probe_duration_seconds
    probe_http_status_code
    probe_ssl_earliest_cert_expiry

---

## TwentyI don't know.Alert Rule Design

### 20.1 Node NotReady /think

- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready", status="true"} == 0
  for: 5m
  labels:
    severity: critical
    team: sre
  annotations:
    summary: "Node NotReady"
    description: "Node {{ $labels.node }} has been NotReady for more than 5 minutes."
    runbook_url: "https://wiki.example.com/runbook/node-not-ready"

### 20.2 Node CPU High

    - alert: NodeHighCPUUsage
      expr: |
        100 - (
          avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          ) * 100
        ) > 90
      for: 10m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Node CPU Usage is Too High"
        description: "Node {{ $labels.instance }} CPU usage is above 90% for more than 10 minutes."

### 20.3 Node Memory High

    - alert: NodeHighMemoryUsage
      expr: |
        (
          1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
        ) * 100 > 90
      for: 10m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Node Memory Usage is Too High"
        description: "Node {{ $labels.instance }} memory usage is above 90% for more than 10 minutes."

### 20.4 Node Disk Space Low

    - alert: NodeDiskSpaceLow
      expr: |
        (
          1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
          / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
        ) * 100 > 85
      for: 15m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Node Disk Space is Low"
        description: "Node {{ $labels.instance }} filesystem {{ $labels.mountpoint }} usage is above 85%."

### 20.5 Pod Pending

    - alert: PodPendingTooLong
      expr: kube_pod_status_phase{phase="Pending"} == 1
      for: 10m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Pod Pending Time is Too Long"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been Pending for more than 10 minutes."

### 20.6 Pod Restarts Too Often

    - alert: PodRestartTooOften
      expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
      for: 5m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Pod Restart Count is Too High"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarted more than 3 times in 10 minutes."

### 20.7 Pod OOMKilled

    - alert: PodOOMKilled
      expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
      for: 1m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Pod was OOMKilled"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} was OOMKilled."

### 20.8 Prometheus Target Down

- alert: PrometheusTargetDown
  expr: up == 0
  for: 5m
  labels:
    severity: warning
    team: sre
  annotations:
    summary: "Prometheus Target Down"
    description: "Target {{ $labels.job }} / {{ $labels.instance }} has been down for more than 5 minutes."

---

## 21. Service Availability Alert Design

### 21.1 High 5xx Error Rate

Prerequisites:

    The application must expose http_requests_total with status label.

Rules:

    - alert: ServiceHigh5xxErrorRate
      expr: |
        sum by (service) (
          rate(http_requests_total{status=~"5.."}[5m])
        )
        /
        sum by (service) (
          rate(http_requests_total[5m])
        )
        * 100 > 5
      for: 5m
      labels:
        severity: critical
        team: app
      annotations:
        summary: "Service 5xx Error Rate Too High"
        description: "Service {{ $labels.service }} 5xx error rate is above 5% for more than 5 minutes."

### 21.2 High P95 Latency

    - alert: ServiceHighP95Latency
      expr: |
        histogram_quantile(
          0.95,
          sum by (le, service) (
            rate(http_request_duration_seconds_bucket[5m])
          )
        ) > 1
      for: 5m
      labels:
        severity: warning
        team: app
      annotations:
        summary: "Service P95 Latency Too High"
        description: "Service {{ $labels.service }} P95 latency is above 1s for more than 5 minutes."

### 21.3 Blackbox Probe Failure

Prerequisites:

    Using blackbox-exporter.

Rules:

    - alert: ServiceProbeFailed
      expr: probe_success == 0
      for: 3m
      labels:
        severity: critical
        team: sre
      annotations:
        summary: "Service Probe Failed"
        description: "Probe target {{ $labels.instance }} has failed for more than 3 minutes."

---

## 22. Complete Troubleshooting Case: Service is Normal but Business Access Fails

### 22.1 Symptoms

User feedback:

    Service access failure

kubectl check:

    kubectl get svc -n app
    kubectl get pod -n app

Discovery:

    Service exists
    Pod Running

### 22.2 Troubleshooting Steps

Step 1: Check Service details

    kubectl describe svc <service-name> -n app

Check:

    selector
    port
    targetPort
    type
    endpoints

Step 2: Check Endpoints

    kubectl get endpoints <service-name> -n app
    kubectl get endpointslice -n app | grep <service-name>

If Endpoints is empty:

    Check selector and Pod label.

Step 3: Check Pod label

    kubectl get pod -n app --show-labels

Confirm Pod label matches Service selector.

Step 4: Check Pod Ready

    kubectl get pod -n app
    kubectl describe pod <pod-name> -n app

If Pod Running but not ready:

    Check readinessProbe.

Step 5: Enter Pod to check listening ports

    kubectl exec -it <pod-name> -n app -- sh
    ss -lntp

Confirm application is listening on targetPort.

Step 6: Access Service within cluster

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside container:

    curl -v http://<service-name>.app.svc.cluster.local:<port>

Step 7: Check application logs

    kubectl logs <pod-name> -n app

Step 8: Check business metrics

PromQL:

    rate(http_requests_total[5m])

    rate(http_requests_total{status=~"5.."}[5m])

    histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

### 22.3 Judgment Results

Possible conclusions: /think

- Service selector is written incorrectly;
- Pod label does not match;
- readinessProbe failed;
- targetPort is written incorrectly;
- Application is not listening on 0.0.0.0;
- Container port is inconsistent with Service port;
- NetworkPolicy is blocking;
- Ingress forwarding error;
- Application internal dependency anomaly;
- Database connection failure;
- DNS resolution anomaly.

---

## 23. Complete Troubleshooting Case: Pod is Normal but Prometheus has No Metrics

### 23.1 Phenomenon

Application Pod is Running.

Business access is normal.

However, Prometheus has no application metrics.

### 23.2 Troubleshooting Steps

Step 1: Check if the application exposes /metrics

    kubectl exec -it <pod-name> -n <namespace> -- sh
    curl http://127.0.0.1:<metrics-port>/metrics

Step 2: Check if the Service exposes metrics port

    kubectl describe svc <service-name> -n <namespace>

Step 3: Check if the Service port has a name

Example:

    ports:
      - name: metrics
        port: 9100
        targetPort: 9100

Step 4: Check if ServiceMonitor matches Service

    kubectl describe servicemonitor <servicemonitor-name> -n <namespace>

Check:

    namespaceSelector
    selector.matchLabels
    endpoints.port
    endpoints.path

Step 5: Check if Prometheus selects ServiceMonitor

If kube-prometheus-stack has a label selector for ServiceMonitor, confirm if ServiceMonitor labels match.

Example:

    release: kube-prometheus-stack

Step 6: Prometheus Targets

Prometheus page:

    Status -> Targets

Search for the application name.

### 23.3 Common Causes

- Application does not expose /metrics;
- metrics path is not /metrics;
- Service does not expose metrics port;
- Service port has no name;
- ServiceMonitor selector does not match;
- ServiceMonitor namespaceSelector does not match;
- ServiceMonitor lacks release label;
- NetworkPolicy is blocking;
- Prometheus RBAC permissions are insufficient;
- relabel configuration filters out target.

---

## 24. Complete Troubleshooting Case: Pod Pending Alert

### 24.1 Alert

AlertManager receives:

    PodPendingTooLong

### 24.2 Prometheus Query

    kube_pod_status_phase{phase="Pending"} == 1

Locate:

    namespace
    pod

### 24.3 kubectl Troubleshooting

    kubectl describe pod <pod-name> -n <namespace>

Check Events.

### 24.4 Common Root Cause Judgment

If the event is:

    insufficient cpu

Resolution:

    Reduce CPU request or scale up nodes.

If the event is:

    insufficient memory

Resolution:

    Reduce memory request or scale up nodes.

If the event is:

    pod has unbound immediate PersistentVolumeClaims

Resolution:

    Check PVC / StorageClass / PV.

If the event is:

    node(s) had untolerated taint

Resolution:

    Add toleration or adjust node taint.

If the event is:

    didn't match Pod's node affinity/selector

Resolution:

    Correct nodeSelector / affinity or node label.

If the event is:

    insufficient nvidia.com/gpu

Resolution:

    Check GPU node, Device Plugin, GPU usage, ResourceQuota.

---

## 25. Complete Troubleshooting Case: Prometheus Target Down

### 25.1 Alert

    PrometheusTargetDown

### 25.2 Query

    up == 0

Check labels:

    job
    instance
    namespace
    service
    pod

### 25.3 Determine Type

If node-exporter is Down:

    Check node-exporter DaemonSet and node status.

If kube-state-metrics is Down:

    Check kube-state-metrics Deployment / Service.

If kubelet is Down:

    Check kubelet authentication, network, and node status.

If dcgm-exporter is Down:

    Check GPU node and DCGM Exporter Pod.

If application metrics are Down:

    Check application Service, Pod, and /metrics.

### 25.4 Troubleshooting Commands

    kubectl get pod -A -o wide | grep <target>
    kubectl get svc -A | grep <target>
    kubectl get endpoints -A | grep <target>
    kubectl describe servicemonitor <name> -n <namespace>
    kubectl logs <target-pod> -n <namespace>

---

## 26. Recording Rule Recommendations

To reduce Grafana query pressure, commonly used PromQL can beDown.ed as Recording Rules.

### 26.1 Node CPU Usage /think

- record: node:cpu_usage:percent
  expr: |
    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

### 26.2 Node CPU Usage Rate

- record: node:memory_usage:percent
  expr: |
    (
      1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
    ) * 100

### 26.3 Pod CPU Usage

- record: pod:cpu_usage:cores
  expr: |
    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
    )

### 26.4 Pod Memory Usage

- record: pod:memory_working_set:bytes
  expr: |
    sum by (namespace, pod) (
      container_memory_working_set_bytes{container!="", image!=""}
    )

### 26.5 Service Error Rate

- record: service:http_5xx_error_rate:percent
  expr: |
    sum by (service) (
      rate(http_requests_total{status=~"5.."}[5m])
    )
    /
    sum by (service) (
      rate(http_requests_total[5m])
    )
    * 100

---

## Twenty-SevenI don't know.Production Monitoring Metric Baselines

### 27.1 Node Must Monitor

    [ ] Node Ready
    [ ] Node CPU Usage Rate
    [ ] Node Memory Usage Rate
    [ ] Node Disk Usage Rate
    [ ] Node inode Usage Rate
    [ ] Node Network Traffic
    [ ] Node Network Error Packets
    [ ] Node Load
    [ ] kubelet Status
    [ ] containerd Status
    [ ] Node Cordon Status
    [ ] Node Resource Pressure

### 27.2 Pod Must Monitor

    [ ] Pod Phase
    [ ] Pod Ready
    [ ] Pod Pending
    [ ] Pod Failed
    [ ] Pod Restart
    [ ] Pod OOMKilled
    [ ] Pod CPU Usage
    [ ] Pod Memory Usage
    [ ] Pod Network Traffic
    [ ] Deployment Available Replicas
    [ ] StatefulSet Available Replicas
    [ ] DaemonSet Ready Count

### 27.3 Service Must Monitor

    [ ] Service Existence
    [ ] Endpoint Empty
    [ ] Backend Pod Ready
    [ ] Application QPS
    [ ] Application 5xx Error Rate
    [ ] P95 / P99 Latency
    [ ] Blackbox Probe Success Rate
    [ ] Ingress Status
    [ ] DNS Resolution Availability

### 27.4 Monitoring System Itself Must Monitor

    [ ] Prometheus Target UP
    [ ] Prometheus TSDB Head Series
    [ ] Prometheus Storage Space
    [ ] Prometheus Rule Calculation Failure
    [ ] AlertManager Status
    [ ] Grafana Status
    [ ] node-exporter Status
    [ ] kube-state-metrics Status
    [ ] ServiceMonitor Effectiveness

---

## Twenty-EightI don't know.Production Alert Baseline

### 28.1 Critical Level Recommendations

    NodeNotReady
    KubeAPIDown
    EtcdDown
    CoreDNSUnavailable
    ProductionServiceProbeFailed
    ProductionServiceHigh5xxErrorRate
    ProductionServiceHighLatency
    GPUXIDError
    GPUCriticalTemperature
    DiskAlmostFull

### 28.2 Warning Level Recommendations

    NodeHighCPUUsage
    NodeHighMemoryUsage
    NodeDiskSpaceLow
    PodPendingTooLong
    PodRestartTooOften
    PodOOMKilled
    DeploymentReplicasMismatch
    TargetDown
    GPUMemoryUsageHigh
    GPUHighTemperature

### 28.3 Info Level Recommendations

    GPULowUtilization
    NamespaceQuotaNearLimit
    CertificateExpiringSoon
    LongRunningJob
    DevNamespaceResourceHigh

---

## Twenty-NineI don't know.Common Misconceptions

### 29.1 Misconception One: kubectl top has data, so Prometheus is not needed

Error.

kubectl top comes from Metrics Server, suitable for viewing current CPU / Memory.

Prometheus is used for:

- Long-term trends;
- Dashboard;
- Alerts;
- Complex PromQL;
- Business metrics;
- GPU metrics;
- Capacity planning.

### 29.2 Misconception Two: Pod Running means the business is normal

Error.

Pod Running only indicates the container process exists.

Whether the business is normal also depends on:

- readinessProbe;
- Application Logs;
- Service Endpoints;
- QPS;
- Error Rate;
- Latency;
- Dependent Services;
- Database Connections;
- Blackbox Probe.

### 29.3 Misconception Three: A Service Having a ClusterIP Means the Service is Available

Error.

A Service having a ClusterIP only indicates the Service object exists.

You also need to check:

- selector;
- Endpoints;
- Pod Ready;
- targetPort;
- Application Listening;
- Network Policy;
- DNS;
- kube-proxy;
- Business Interface.

### 29.4 Misconception Four: Prometheus Target Down Must Mean the Exporter is Down

Not necessarily.

It could be:

- ServiceMonitor selector error;
- Service port name error;
- NetworkPolicy blocking;
- Prometheus lacks permissions;
- Endpoints are empty;
- TLS configuration error;
- Path is not /metrics;
- Exporter is indeed abnormal.

### 29.5 Misconception Five: Judging Business Quality Based Only on Resource Metrics

Error.

CPU, memory, GPU are only resource metrics.

Business quality must be assessed by:

- QPS;
- Error Rate;
- Latency;
- Success Rate;
- Queue Length;
- Log Errors;
- User Request Results.

### 29.6 Misconception Six: Installing Prometheus Means You Have Logs

Error.

Prometheus collects metrics, not Pod logs.

Pod logs require:

    Loki + Alloy / Promtail

or:

    Elasticsearch / OpenSearch + Filebeat / Fluent Bit

---

## Thirty, K8S Monitoring Troubleshooting Command Summary

### 30.1 Check Monitoring Components

    kubectl get pods -n monitoring -o wide
    kubectl get svc -n monitoring
    kubectl get servicemonitor -A
    kubectl get prometheusrule -A
    kubectl get alertmanager -n monitoring
    kubectl get prometheus -n monitoring

### 30.2 Check Prometheus Targets

    kubectl port-forward svc/<prometheus-service-name> 9090:9090 -n monitoring

Access:

    http://127.0.0.1:9090/targets

### 30.3 Check Nodes

    kubectl get nodes -o wide
    kubectl describe node <node-name>
    kubectl top node

### 30.4 Check Pods

    kubectl get pod -A -o wide
    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous
    kubectl top pod -A

### 30.5 Check Services

    kubectl get svc -A
    kubectl describe svc <service-name> -n <namespace>
    kubectl get endpoints <service-name> -n <namespace>
    kubectl get endpointslice -n <namespace>

### 30.6 Test DNS and Service

    kubectl run dns-test --rm -it --image=busybox:1.36 -- sh

Inside the container:

    nslookup kubernetes.default.svc.cluster.local
    nslookup <service-name>.<namespace>.svc.cluster.local

HTTP Test:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside the container:

    curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>

### 30.7 Node-side Troubleshooting

    systemctl status kubelet
    systemctl status containerd
    journalctl -u kubelet -f
    journalctl -u containerd -f
    df -h
    df -i
    free -h
    top
    ss -lntp
    ip route
    ip addr

---

## Thirty-one, Production Deployment Recommendations

### 31.1 Metric Collection Recommendations

Recommend using unified:

    kube-prometheus-stack
    node-exporter
    kube-state-metrics
    kubelet/cAdvisor
    Metrics Server
    blackbox-exporter
    dcgm-exporter
    Application /metrics

### 31.2 Log Collection Recommendations

Loki Route:

    Grafana Alloy
      ↓
    Loki
      ↓
    Grafana

EFK Route:

    Fluent Bit / Filebeat
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

Recommendation Selection:

    Operations Troubleshooting, Grafana Integration, Cost-sensitive:
        Loki is more suitable.

    Full-text Search, Audit, Security, Complex Field Search:
        Elasticsearch / OpenSearch is more suitable.

### 31.3 Dashboard Recommendations

At least establish:

Kubernetes Cluster Overview  
Node Overview  
Pod / Namespace Overview  
Service Overview  
Ingress Overview  
GPU Overview  
Prometheus Self Monitoring  
Alert Overview  
Pod Logs Overview  
Loki / ELK Overview  

### 31.4 Alert Suggestions  

Alerts must meet the following criteria:  

    - Have an owner  
    - Have a severity level  
    - Have a summary  
    - Have a description  
    - Have a runbook_url  
    - Have a dashboard_url  
    - Have a reasonable for time  
    - Have clear handling actions  

### 31.5 Runbook Suggestions  

At least write the following:  

    - NodeNotReady  
    - NodeHighCPUUsage  
    - NodeHighMemoryUsage  
    - NodeDiskSpaceLow  
    - PodPendingTooLong  
    - PodCrashLoopBackOff  
    - PodOOMKilled  
    - ServiceNoEndpoint  
    - TargetDown  
    - CoreDNSUnavailable  
    - GPUXIDError  
    - PodLogsMissing  
    - AppErrorLogsTooMany  

### 31.6 Multi-Team Guidelines  

Recommendations:  

- Each Namespace has an owner;  
- Each Service has an owner;  
- Each alert has a team label;  
- Each Dashboard has a responsible person;  
- Each Runbook has a maintainer;  
- Alert changes go through Git;  
- Dashboard JSON is included in Git;  
- PrometheusRule is included in Git;  
- Log collection rules are included in Git.  

---  

## Thirty-Two, Acceptance Checklist  

### 32.1 Metric Collection Pipeline  

    [ ] node-exporter Target UP  
    [ ] kube-state-metrics Target UP  
    [ ] kubelet Target UP  
    [ ] cAdvisor metrics are accessible  
    [ ] Application /metrics are accessible  
    [ ] dcgm-exporter Target UP  
    [ ] Prometheus self metrics are accessible  

### 32.2 Node Metrics  

    [ ] Node Ready is accessible  
    [ ] CPU usage rate is accessible  
    [ ] Memory usage rate is accessible  
    [ ] Disk usage rate is accessible  
    [ ] inode usage rate is accessible  
    [ ] Network traffic is accessible  
    [ ] Load is accessible  

### 32.3 Pod Metrics  

    [ ] Pod Phase is accessible  
    [ ] Pod Ready is accessible  
    [ ] Pod CPU is accessible  
    [ ] Pod Memory is accessible  
    [ ] Pod Restart is accessible  
    [ ] OOMKilled is accessible  
    [ ] Pending Pod is accessible  

### 32.4 Service Metrics  

    [ ] Service objects are accessible  
    [ ] Endpoints are accessible  
    [ ] EndpointSlice is accessible  
    [ ] Application QPS is accessible  
    [ ] Application error rate is accessible  
    [ ] Application P95/P99 latency is accessible  
    [ ] Blackbox probe is accessible  

### 32.5 Log Collection Pipeline  

    [ ] Log Agent runs as DaemonSet  
    [ ] Agent covers all nodes  
    [ ] /var/log/containers is mounted  
    [ ] /var/log/pods is mounted  
    [ ] Loki / ELK can query Pod logs  
    [ ] Logs include namespace / pod / container / node fields  
    [ ] Can query logs by namespace  
    [ ] Can query logs by pod  
    [ ] Can query ERROR / timeout / exception / panic  

### 32.6 Display and Alert  

    [ ] Grafana Node Dashboard has data  
    [ ] Grafana Pod Dashboard has data  
    [ ] Grafana Service Dashboard has data  
    [ ] Grafana can jump to Pod logs  
    [ ] PrometheusRule is loaded  
    [ ] AlertManager can receive alerts  
    [ ] Notification channels are available  
    [ ] Runbook links are valid  

---  

## Thirty-Three, Pod Monitoring and Pod Log Collection Integration in Practice  

### 33.1 Why Combine Pod Monitoring and Pod Logs  

Pod monitoring addresses the following:  

    - Pod's current status  
    - Whether Pod is Ready  
    - Whether Pod is Pending  
    - Whether Pod has restarted  
    - Whether Pod was OOMKilled  
    - Whether Pod CPU / Memory / Network is abnormal  

Pod logs address the following:  

    - Why the application failed to start  
    - Why dependency connections failed  
    - Why business interface returns 500  
    - Why probes failed  
    - Whether there were large requests or abnormal tasks before OOM  
    - What errors were printed by the application before CrashLoopBackOff  
    - What happened internally in the application when Service is normal but business is abnormal  

Both must be combined.  

Only looking at Pod metrics will know "Pod is abnormal," but not necessarily the cause.  

Only looking at Pod logs will know "application reported errors," but not necessarily whether it affects scheduling, resources, restarts, Service backend, and overall availability.  

Production troubleshooting recommended path:  

    Pod metrics detect anomalies  
      ↓  
    kubectl describe to check status and Events  
      ↓  
    kubectl logs / --previous to check application logs  
      ↓  
    Loki / ELK query similar Pod logs  
      ↓  
    Prometheus query resource trends  
      ↓  
    Service / Endpoints verify backend traffic  
      ↓  
    Determine root cause and handle  

---  

### 33.2 Where Do Pod Logs Come From  

Kubernetes recommends container applications output logs to:  

    stdout  
    stderr  

The container runtime is responsible for writing stdout / stderr to local files on the node.  

Common path in containerd environment: /think

/var/log/containers/
/var/log/pods/

Example:

/var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

Viewing log files on nodes:

ls -l /var/log/containers/
ls -l /var/log/pods/

kubectl logs essentially reads container logs through the Kubernetes API.

Common commands:

kubectl logs <pod-name> -n <namespace>

Viewing logs from the previous container instance:

kubectl logs <pod-name> -n <namespace> --previous

Viewing logs for a specific container:

kubectl logs <pod-name> -n <namespace> -c <container-name>

Viewing the last 100 lines:

kubectl logs <pod-name> -n <namespace> --tail=100

Continuous viewing:

kubectl logs <pod-name> -n <namespace> -f

---

### 33.3 Pod Log Collection Pipeline

Centralized log systems typically use DaemonSet to collect logs from each node.

Typical pipeline:

Pod stdout / stderr
↓
containerd writes to node log files
↓
/var/log/containers/*.log
↓
Log collection Agent
↓
Loki / Elasticsearch / OpenSearch
↓
Grafana / Kibana Query

Common Agents:

Grafana Alloy
Promtail
Fluent Bit
Filebeat
Fluentd

Loki pipeline:

Pod Logs
↓
Alloy / Promtail
↓
Loki
↓
Grafana Explore

EFK pipeline:

Pod Logs
↓
Filebeat / Fluent Bit
↓
Elasticsearch / OpenSearch
↓
Kibana / OpenSearch Dashboards

---

### 33.4 Pod Log Collection Must Preserve Labels / Fields

Regardless of using Loki or EFK, it's recommended to preserve the following Kubernetes metadata:

cluster
environment
namespace
pod
container
node
app
workload
team

These fields help quickly locate:

Which cluster
Which environment
Which Namespace
Which Pod
Which container
Which node
Which application
Which team is responsible

Loki query example:

{namespace="app-prod", pod="api-xxx"}

Query by app:

{namespace="app-prod", app="api"}

Query error logs:

{namespace="app-prod", pod="api-xxx"} |~ "(?i)error|exception|panic|traceback"

EFK / Kibana query example:

kubernetes.namespace : "app-prod" and kubernetes.pod.name : "api-xxx"

Query error logs:

kubernetes.namespace : "app-prod"
and kubernetes.pod.name : "api-xxx"
and log.level : "error"

Do not use the following high-cardinality fields as Loki labels:

request_id
trace_id
user_id
order_id
session_id
full_url
error_message
stacktrace

These fields can be saved as log body or Elasticsearch fields, but are unsuitable for Loki's indexing labels.

---

### 33.5 Pod Monitoring and Log Correlation: CrashLoopBackOff

Phenomenon:

Pod repeatedly restarts
Status may show CrashLoopBackOff

Metrics side discovery:

increase(kube_pod_container_status_restarts_total[10m]) > 3

kubectl troubleshooting:

kubectl get pod <pod-name> -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

Loki query:

{namespace="<namespace>", pod="<pod-name>"} |~ "(?i)error|exception|panic|traceback|failed"

EFK query:

kubernetes.namespace : "<namespace>"
and kubernetes.pod.name : "<pod-name>"
and message : ("error" or "exception" or "panic" or "failed")

Key judgment:

- What is the Last State Reason?
- What is the Exit Code?
- Are there any startup failure information in previous logs?
- Are there probe failures in Events?
- Are ConfigMap/Secret missing?
- Is database connection failed?
- Is there port conflict?
- Is there permission error?
- Is there image entry command error?

Typical conclusions:

Pod restarts are not the cause, but the result.
The real cause is usually in previous logs, Events, Exit Code, and application startup logs.

---

### 33.6 Pod Monitoring and Log Correlation: OOMKilled

Phenomenon:

Pod is OOMKilled
Container restarts
Service temporarily unavailable

# 33.6 OOM Question examination:Pod Systemized OOM Killed

## Indicator side found:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

## View memory trends:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}
    )

## View Memory limit:

    kube_pod_container_resource_limits{namespace="<namespace>", pod="<pod-name>", resource="memory"}

## kubectl Check:

    kubectl describe pod <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

## Loki Query:

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)oom|memory|killed|out of memory"

## AI / GPU scene query:

    {namespace="<namespace>", pod="<pod-name>"} |= "CUDA out of memory"

## Focus:

- Memory limit Whether it is too low;
- Whether there is a memory leak;
- Whether there are large requests, large documents and large volumes of processing;
- Whether or not JVM heap Unreasonable setting;
- Whether or not Python / AI (a) The task loads large models or data;
- Whether or not batch size Oversized;
- Whether the container is being systemized OOM Or use the inside? CUDA OOMI don't know.

## A distinction needs to be made between:

    OOMKilled:
        Linux / Containment level memory excess.

    CUDA out of memory:
        GPU Insufficient visibility, common in applied logs.

The two issues are in different directions.

## OOMKilled Priority:

    container_memory_working_set_bytes
    memory limit
    Pod describe
    previous logs

## CUDA OOM Priority:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE
    GPU Pod Log
    batch size
    Model Size
    Quantity

---

### 33.7 Pod Monitor and log connection:Pod Running But it's an anomaly.

## Perceptions:

    Pod Running
    Pod Ready
    Service Yes. Endpoints
    But user access to business unusual

Such issues are most easily miscalculated.

Pod Running It can only be explained that the process exists and does not represent an operational correctness.

## Question order:

    1. View Service and Endpoints
    2. View Pod Ready
    3. View container port listening
    4. Within Clusters curl Service
    5. View operational indicators QPS / 5xx / Delay
    6. View Pod Log
    7. View Services Dependence
    8. View Ingress / Gateway
    9. View NetworkPolicy
    10. View DNS

## Command:

    kubectl describe svc <service-name> -n <namespace>
    kubectl get endpoints <service-name> -n <namespace>
    kubectl get pod -n <namespace> -o wide
    kubectl logs <pod-name> -n <namespace> --tail=100

## Access to the container for listening:

    kubectl exec -it <pod-name> -n <namespace> -- ss -lntp

## Temporary Pod Test:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

## In packagings:

    curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>

## Loki Query:

    {namespace="<namespace>", app="<app>"} |~ "(?i)error|timeout|connection refused|database|redis|exception"

## Prometheus Query service error rate:

    sum by (service) (
      rate(http_requests_total{status=~"5.."}[5m])
    )
    /
    sum by (service) (
      rate(http_requests_total[5m])
    )
    * 100

## Typical cause:

- Application of in-house dependency anomalies;
- Database connection failed;
- Redis Connection failed;
- Downstream interface timeout;
- Configuration change error;
- targetPort Errors;
- readinessProbe Too loose;
- Operational health interfaces do not cover real dependence;
- Ingress Other Organiser

---

### 33.8 Pod Log collection abnormally.

## Perceptions:

    kubectl logs Log
    But... Loki / Kibana No log.

## Question order:

    1. Log Collection Agent Whether or not Running
    2. Agent Is it DaemonSet Overwrite this node
    3. /var/log/containers Whether to mount correctly
    4. Agent Whether log files are readable
    5. Agent Successful Add Kubernetes metadata
    6. Whether it filters the namespace
    7. Whether it filters the container
    8. Loki / Elasticsearch Whether to write
    9. Ask if the time frame is correct
    10. Question label / Error Field

## View Agent:

    kubectl get pods -n logging -o wide
    kubectl get ds -n logging

## View Agent Log:

    kubectl logs <agent-pod> -n logging

## Enter Agent Pod Check mount:

    kubectl exec -it <agent-pod> -n logging -- sh
    ls -l /var/log/containers/

If there are no logs for any Pod on a node:

    First check if the Agent Pod on that node is Running.

If there are no logs for any Namespace:

    First check if the collection rules have filtered out the Namespace.

If only a specific application has no logs:

    First check if the application is actually outputting stdout / stderr.

Command:

    kubectl logs <pod-name> -n <namespace>

If kubectl logs also shows no content, it indicates the application may not be outputting standard logs.

---

### 33.9 Dashboard Recommendations for Pod Monitoring and Log Integration

Pod Dashboard, in addition to CPU, memory, and restart counts, is recommended to include log entries.

Recommended panels:

    Pod Phase
    Pod Ready
    Pod Restart Count
    Pod CPU Usage
    Pod Memory Working Set
    Pod Network Receive / Transmit
    Last Terminated Reason
    OOMKilled Status
    Recent Events
    Recent Error Logs
    Recent Warning Logs
    Log Link to Loki Explore
    Log Link to Kibana Discover

Grafana variables:

    $namespace
    $pod
    $container
    $node
    $app

Loki query panel:

    {namespace="$namespace", pod="$pod"}

Error log panel:

    {namespace="$namespace", pod="$pod"} |~ "(?i)error|exception|panic|traceback|timeout"

If using EFK, you can add a Kibana jump link in the Dashboard:

    namespace=<namespace>
    pod=<pod>
    time range=last 30 minutes

Goals:

    Jump from Pod metrics to corresponding Pod logs with one click.
    Jump from alerts to corresponding Dashboard and log queries.
    Reduce the time needed to manually input query conditions.

---

### 33.10 Alert Recommendations for Pod Monitoring and Log Integration

Metric alerts:

    PodPendingTooLong
    PodRestartTooOften
    PodOOMKilled
    PodNotReadyTooLong
    PodHighCPUUsage
    PodHighMemoryUsage

Log alerts:

    AppErrorLogsTooMany
    AppTimeoutLogsTooMany
    PythonTracebackDetected
    JavaExceptionLogsTooMany
    DatabaseConnectionErrorLogs
    CUDALogOOMDetected
    ModelLoadFailedLogs

Recommended approach:

    Metric alerts are responsible for detecting abnormal phenomena.
    Log alerts are responsible for supplementing error causes.
    AlertManager is responsible for grouping, deduplication, and suppression.
    Runbook is responsible for guiding handling.
    Dashboard is responsible for displaying trends.
    Loki / ELK is responsible for viewing context.

Example: Associating Pod restart alerts with log alerts

    PodRestartTooOften
      +
    AppErrorLogsTooMany
      +
    PythonTracebackDetected

If they share the same:

    cluster
    namespace
    app
    pod

They should be grouped together in AlertManager as much as possible to avoid duplicate notifications.

---

### 33.11 Verification Checklist for Pod Monitoring and Log Integration

    [ ] Pod Phase metric is available
    [ ] Pod Ready metric is available
    [ ] Pod CPU metric is available
    [ ] Pod Memory metric is available
    [ ] Pod Restart metric is available
    [ ] Pod OOMKilled metric is available
    [ ] kubectl logs can view business logs
    [ ] kubectl logs --previous can view previous container logs
    [ ] /var/log/containers path exists
    [ ] Log collection Agent runs as DaemonSet
    [ ] Agent covers all nodes
    [ ] Loki / ELK can query logs by namespace
    [ ] Loki / ELK can query logs by pod
    [ ] Logs include namespace / pod / container / node fields
    [ ] Grafana Pod Dashboard can jump to logs
    [ ] Pod restart alert can trigger
    [ ] Pod OOMKilled alert can trigger
    [ ] ERROR log alert can trigger
    [ ] Alerts include runbook_url
    [ ] Alerts include dashboard_url

---

### 33.12 Summary

Pod troubleshooting cannot rely on a single signal.

The minimal closed-loop should be:

    Pod metrics
      +
    Pod Events
      +
    Pod logs
      +
    Service Endpoints
      +
    Application business metrics

Corresponding relationships:

    kube-state-metrics:
        Tells you the status of the Pod.

    kubelet / cAdvisor:
        Tells you how much CPU / Memory / Network the Pod uses.

    kubectl describe:
        Tells you Kubernetes scheduling, probes, image pulling, mounting, etc. events.

    kubectl logs / --previous:
        Tells you why the application is abnormal now or in the previous container.

    Loki / ELK:
        Tells you log context across Pods, replicas, and time ranges.

    Prometheus:
        Tells you abnormal trends and alert conditions.

    Grafana:
        Places metrics and log entries in a single troubleshooting view.

Recommended production troubleshooting approach:

    Metrics show phenomena
    Events show K8S behavior
    Logs show causes
    Service shows backend traffic
    Runbook shows handling steps

---

## Thirty-Four, Summary /think

# Kubernetes Monitoring in Practice

The core of Kubernetes monitoring isn't simply installing Prometheus and seeing a few charts, but building an observability and troubleshooting system around Node, Pod, and Service.

## Node Monitoring Answers:

- Is the node healthy?
- Are resources sufficient?
- Is kubelet/containerd functioning normally?
- Is the node still suitable for hosting Pods?

## Pod Monitoring Answers:

- Is the workload running normally?
- Is it Pending?
- Has it restarted?
- Was it OOMKilled?
- Is there abnormal resource usage?

## Service Monitoring Answers:

- Is service discovery correct?
- Are Endpoints present?
- Are backend Pods Ready?
- Are business requests successful?
- Are error rates and latency normal?

## Pod Log Supplemental Answers:

- Why did the application fail to start?
- Why did dependencies fail to connect?
- Why did probes fail?
- What happened to the application before OOM or CrashLoop?
- Why is the business still abnormal in Running state?

Metric sources must be clearly distinguished:

| Source | Description |
|--------|-------------|
| node-exporter | Host resource metrics |
| kubelet/cAdvisor | Container resource metrics |
| kube-state-metrics | Kubernetes object status metrics |
| Application /metrics | Business quality metrics |
| blackbox-exporter | External availability probing |
| dcgm-exporter | GPU metrics |
| Loki / ELK | Pod logs, error context, root cause analysis |

Troubleshooting should not rely on single metrics.

Correct path:

Prometheus metrics detect anomalies  
↓  
Grafana view trends  
↓  
AlertManager consolidate notifications  
↓  
kubectl describe view events  
↓  
kubectl logs / --previous view application logs  
↓  
Loki / ELK query cross-replica logs  
↓  
Service / Endpoints / DNS validate the chain  
↓  
Node commands confirmBottom status  
↓  
Runbook execution  
↓  
Post-mortem and optimize alerts

Production-grade Kubernetes monitoring should ultimately achieve:

- Complete metrics
- Accessible logs
- Clear dashboards
- Accurate alerts
- Reliable notifications
- Troubleshooting path
- Handling with Runbook
- Post-mortem knowledgeDeposition
- Capacity planning

---

## 35. Reference Documents

- Kubernetes Observability:  
  https://kubernetes.io/docs/concepts/cluster-administration/observability/

- Kubernetes Metrics Reference:  
  https://kubernetes.io/docs/reference/instrumentation/metrics/

- Kubernetes Resource Metrics Pipeline:  
  https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/

- Kubernetes Logging Architecture:  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- kube-state-metrics:  
  https://github.com/kubernetes/kube-state-metrics

- Prometheus Configuration:  
  https://prometheus.io/docs/prometheus/latest/configuration/configuration/

- Prometheus Querying Basics:  
  https://prometheus.io/docs/prometheus/latest/querying/basics/

- Prometheus Query Functions:  
  https://prometheus.io/docs/prometheus/latest/querying/functions/

- Prometheus Operator:  
  https://prometheus-operator.dev/

- kube-prometheus-stack Helm Chart:  
  https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

- Grafana Documentation:  
  https://grafana.com/docs/

- AlertManager Documentation:  
  https://prometheus.io/docs/alerting/latest/alertmanager/

- Blackbox Exporter:  
  https://github.com/prometheus/blackbox_exporter

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/

- Grafana Alloy Documentation:  
  https://grafana.com/docs/alloy/latest/

- Fluent Bit Documentation:  
  https://docs.fluentbit.io/

- Filebeat on Kubernetes:  
  https://www.elastic.co/docs/reference/beats/filebeat/running-on-kubernetes

- Elasticsearch Documentation:  
  https://www.elastic.co/docs

- OpenSearch Documentation:  
  https://docs.opensearch.org/

- NVIDIA DCGM Exporter:  
  https://github.com/NVIDIA/dcgm-exporter

- NVIDIA GPU Operator:  
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html