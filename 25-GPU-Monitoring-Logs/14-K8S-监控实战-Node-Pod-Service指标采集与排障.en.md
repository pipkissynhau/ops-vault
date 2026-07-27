# 14-K8S-Monitoring Practice - Node, Pod, Service Metric Collection and Troubleshooting

## Document Description

This document systematically organizes the methods for collecting metrics from three core Kubernetes objects: Nodes, Pods, and Services. It covers how to query these metrics using Prometheus, display them in Grafana, design alerts, integrate Pod log collection, and troubleshoot issues.

This document is not merely about how to install Prometheus or list PromQL commands. Instead, it focuses on establishing the following operational and SRE-related links from an operations perspective:

    Kubernetes Objects
      ↓
    Metric Sources
      ↓
    Prometheus Collection
      ↓
    PromQL Queries
      ↓
    Grafana Dashboards
      ↓
    AlertManager Alerts
      ↓
    kubectl /Events /Logs /Service /Endpoints for Troubleshooting
      ↓
    Fault Closure Loop

This document addresses the following key questions:

- What metrics should be monitored for Nodes, Pods, and Services in Kubernetes?
- What data do node-exporter, kubelet/cAdvisor, and kube-state-metrics collect respectively?
- What is the difference between Metrics Server and Prometheus?
- Which monitoring components, logging tools, alert systems, and GPU-related tools need to be installed?
- How can we verify whether Prometheus is successfully collecting Kubernetes metrics?
- How can we check Node CPU, memory, disk, and network metrics?
- How can we monitor Pod CPU, memory, restarts, Pending status, and OOMKilled events?
- Why can't we rely solely on Service objects to determine if a Service is functioning properly?
- What is the relationship between Services, Endpoints, EndpointSlice, Ingresses, and application-specific metrics?
- How should we troubleshoot Prometheus Target Down issues?
- How can we identify problems when Grafana shows no data?
- How can we combine metrics to diagnose issues when both Pods and Services are functioning normally but the business is experiencing errors?
- Where do Pod logs come from, and how can we collect them using Loki/EFK?
- How can we link Pod metrics, Pod Events, Pod logs, and Service backend status to effectively troubleshoot problems?
- How should we design alerts for Nodes, Pods, and Services?
- How can we translate monitoring results into practical troubleshooting actions in production?

This document is recommended for readers who have already completed the following topics:

- 11-Prometheus-Architecture and Core Metric Analysis
- 12-Grafana-Dashboard Construction and Custom Monitoring
- 13-AlertManager-Alert Strategies and Notification Implementation
- 15-Loki-Log Collection and Query Practice
- 16-ELK-EFK-Log Collection and Search Practice
- 17-Log Alerts and Automated Response Practices
- 18-Monitoring, GPU, and Logs: Case Studies on K8S-Pod Exception Detection and Report Generation

---

## Tags

#Kubernetes #Prometheus #Grafana #AlertManager #NodeExporter #kube-state-metrics #cAdvisor #MetricsServer #ServiceMonitor #PodMonitor #Service #Endpoints #EndpointSlice #PodLogs #Loki #EFK #SRE #Monitoring&Troubleshooting

---

## Recommended Reading Path

Recommended reading path:

    06-GPU and AI Infrastructure/05-Observability Basics/14-K8S-Monitoring Practice - Node, Pod, Service Metric Collection and Troubleshooting.md

---

## I. Core Objects of Kubernetes Monitoring

Kubernetes monitoring should not focus solely on whether a node's CPU usage is high.

In a production environment, at least three types of objects need to be monitored:

    Node
    Pod
    Service

Each of these represents different levels of issues that require attention.

### 1.1 What to Monitor for Nodes

Nodes are the foundation where workloads run.

Key areas for node monitoring include:

- Whether the node is in a Ready state;
- Whether kubelet is functioning correctly;
- Whether containerd is running smoothly;
- CPU usage;
- Memory usage;
- Disk usage;
- Inode usage;
- Network traffic;
- Number of network errors;
- System load;
- Number of Pods hosted on the node;
- Number of containers running;
- Whether the node is experiencing resource constraints;
- Whether the node has been cordoned off;
- Whether the node has any taints applied;
- Frequent occurrences of the node being in a NotReady state.

Issues with nodes can lead to:

- Failed Pod scheduling;
- Pod eviction;
- Reduction in Service backend resources;
- Unstable business traffic;
- Errors in GPU/database/intermediate service Pods;
- Unavailability of services across the entire node.

### 1.2 What to Monitor for Pods

Pods are the smallest units within Kubernetes that are scheduled for execution.

Key areas for pod monitoring include:

- Whether the pod is running or in a Pending state;
- Whether the pod has failed or is in a    node_network_transmit_bytes_total
    node_load1
    node_load5
    node_load15
    node_uname_info

These metrics can help answer questions such as:

- Whether the node's CPU usage is high;
- Whether the node is running out of memory;
- Whether the node's disk space is full;
- Whether the node has exhausted its inode space;
- Whether there are any abnormalities in the node's network traffic;
- Whether the node's load is too high;
- Whether the node's file system is read-only or lacks sufficient space.

### 3.2 kubelet / cAdvisor

kubelet is responsible for managing Pods on nodes.

cAdvisor metrics provide insight into container resource usage.

Common metrics include:

    container_cpu_usage_seconds_total
    container_memory_usage_bytes
    container_memory_working_set_bytes
    container_network_receive_bytes_total
    container_network_transmit_bytes_total
    container_fs_usage_bytes

These metrics can help answer questions such as:

- How much CPU is being used by a particular Pod;
- How much memory is being used by a particular Pod;
- Whether a container is experiencing resource issues;
- Which Pods are consuming the most resources;
- Whether there are any abnormalities in the Pod's network traffic;
- How the container's file system is being used.

It's important to note that kubelet is a component of Kubernetes nodes, and cAdvisor metrics are typically exposed by kubelet. In most cases, there is no need to install the cAdvisor DaemonSet separately.

### 3.3 kube-state-metrics

kube-state-metrics monitors the Kubernetes API Server and converts object status into metrics.

It does not collect resource usage data but rather focuses on the "status" of objects.

Common metrics include:

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

These metrics can help answer questions such as:

- Whether a Node is Ready;
- Whether a Pod is Pending, Running, or Restarting;
- Whether a Pod has been OOMKilled;
- Whether the number of Deployment replicas meets requirements;
- Whether a PVC is Pending;
- Whether a Service exists;
- Whether the ResourceQuota is approaching its limit.

### 3.4 Metrics Server

Metrics Server is part of the Kubernetes resource metrics pipeline.

It primarily serves the following purposes:

    kubectl top
    HPA (Horizontal Pod Autoscaler)
    VPA (Vertical Pod Autoscaler)

Common commands include:

    kubectl top node
    kubectl top pod -A

Metrics Server monitors the current CPU and memory usage of Nodes and Pods. However, it is not suitable for long-term monitoring systems.

### 3.5 Application /metrics

Applications expose their own business metrics.

For example, these metrics could indicate:

    http_requests_total
    http_request_duration_seconds_bucket
    http_request_errors_total
    app_queue_length
    app_request_inflight
    model_inference_total
    model_inference_latency_seconds_bucket

These metrics can help answer questions such as:

- Whether the service's QPS is normal;
- Whether the error rate has increased;
- Whether there are any issues with P95 or P99 latency;
- Whether there is a backlog in the queue;
- Whether the service is actually available for use;
- Whether there are requests for GPU inference services;
- Whether model loading operations are successful.

The overall availability of a Service at the application layer ultimately depends on these application metrics and detection indicators.

---

## IV. Experimental Environment Assumptions

This document assumes that a Kubernetes cluster already exists:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22
    k8s-gpu-node01  10.0.0.30

The monitoring namespace is set to:

    monitoring

The logging namespace is set to:

    logging

Core components include:

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

If the kube-prometheus-stack is used, many basic components such as Prometheus, AlertManager, Grafana, node-exporter, kube-state-metrics, Prometheus Operator, and PrometheusRule will be deployed along with Helm Charts.

---

## V. Deployment Methods

There are two common ways toInstall Prometheus, AlertManager, Grafana, node-exporter, kube-state-metrics, Prometheus Operator, default rules, and some dashboards all at once.

Recommended method:

`helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack`

It is not a single plugin but a complete set of Kubernetes monitoring components. It is recommended to use it first in both production and learning environments.

### 6.1.4 node-exporter

Function:

Collects metrics at the Linux node level.

Typical metrics:

CPU, memory, disk, inode, network, load, file system, host information.

Common deployment method:

DaemonSet: One instance of node-exporter runs on each node.

### 6.1.5 kube-state-metrics

Function:

Collects status metrics of Kubernetes objects.

Typical objects:

Node, Pod, Deployment, StatefulSet, DaemonSet, Service, PVC, ResourceQuota, Namespace, Job, CronJob.

Typical metrics:

kube_pod_status_phase, kube_pod_container_status_restarts_total, kube_pod_container_status_last_terminated_reason, kube_node_status_condition, kube_deployment_status_replicas_available.

Note:

kube-state-metrics does not collect CPU/memory usage rates. It collects status information of Kubernetes objects.

### 6.1.6 kubelet / cAdvisor

Function:

Collects resource usage metrics of containers.

Typical metrics:

container_cpu_usage_seconds_total, container_memory_working_set_bytes, container_network_receive_bytes_total, container_network_transmit_bytes_total, container_fs_usage_bytes.

Explanation:

kubelet is a Kubernetes node component. cAdvisor-related metrics are usually exposed by kubelet. Generally, there is no need to install a separate cAdvisor DaemonSet.

### 6.1.7 Metrics Server

Function:

Provides real-time resource metrics for commands like `kubectl top`, HPA, and VPA.

Common commands:

`kubectl top node`, `kubectl top pod -A`

Note:

Metrics Server is not a replacement for Prometheus. It is not suitable for long-term monitoring, dashboards, or alerts. However, it is still recommended to install it in production clusters.

### 6.1.8 blackbox-exporter

Function:

Performs HTTP/TCP/ICMP/DNS probing.

Suitable for monitoring:

Ingress addresses, NodePort addresses, internal HTTP APIs, TCP ports, DNS resolution, TLS certificate validity periods.

It is used to supplement the availability monitoring of Services and business entrances.

---

### 6.2 Alerting Components

#### 6.2.1 AlertManager

Function:

Groups alerts, removes duplicates, suppresses alerts, silences alerts, routes alerts, and sends notifications.

AlertManager receives alerts generated by Prometheus and then sends them to:

Email, Webhook, WeCom, DingTalk, Lark, OnCall systems, ticketing systems.

AlertManager is not responsible for calculating metrics; it handles already triggered alerts.

#### 6.2.2 PrometheusRule

Function:

Manages Prometheus alert rules and Recording Rules through Kubernetes CRDs.

Example alerts:

NodeNotReady, PodPendingTooLong, PodRestartTooOften, PodOOMKilled, ServiceHigh5xxErrorRate, PrometheusTargetDown.

If using kube-prometheus-stack, it is recommended to use PrometheusRule to manage alerts instead of manually modifying Prometheus configuration files.

---

### 6.3 Grafana Visualization Components

#### 6.3.1 Grafana

Function:

 Displays metric dashboards, provides a log query interface, shows alerts, and enables unified visualization of multiple data sources.

Common data sources:

Prometheus, Loki, Elasticsearch, OpenSearch, MySQL, PostgreSQL.

In a Kubernetes monitoring system, Grafana is typically used for:

Observing trends, performing aggregations, conducting multi-dimensional filtering, identifying top anomalies, navigating from dashboards to logs, and jumping from alerts to troubleshooting views.

#### 6.3.2 Grafana Data Sources

Prometheus data source:

Used to query Prometheus metrics.

Loki data source:

Used to query Pod logs.

Elasticsearch/OpenSearch data sources:

Used to query EFK logs.

Note:

Prometheus and Loki are common built-in data sources for Grafana. Generally, there is no need to install additional third-party Grafana plugins. The focus should be on configuring the Data Source addresses and permissions.

---

### 6.4 Pod Log Collection Components

Pod log collection is not the responsibility of Prometheus. Prometheus collects metrics but not logs. Pod log collection requires the deployment of a separate log agent.

#### 6.4.1 Grafana Alloy

Function:

Collects Kubernetes Pod logs, metrics, and traces, and forwards them to backend systems like Loki, Prometheus, or Tempo.

Recommended for new environments:

Grafana Alloy + Loki.

Explanation:

Alloy is the new generation of collection agents recommended by the Grafana ecosystem. It can be used as a replacement for Promtail.

#### 6.4.2 Promtail

Function:

Collects Pod logs and sends them to Loki.

Typical**Log Alerts**
**Field Analysis**

If an enterprise already has ELK/OpenSearch, a Kubernetes cluster typically only needs to deploy a log agent to forward logs to the existing logging platform.

---

### 6.6 GPU Monitoring Components

If there are GPU nodes in the cluster, additional GPU-related components need to be installed.

### 6.6.1 NVIDIA Driver

Function:

    The basic driver for node recognition and use of NVIDIA GPUs.

Installation Location:

    On the host machine where the GPU nodes reside

### 6.6.2 NVIDIA Container Toolkit

Function:

    Allows containers to utilize NVIDIA GPUs.

Without it, containers would not be able to access GPU devices properly.

### 6.6.3 NVIDIA Device Plugin

Function:

    Registers GPU resources with Kubernetes.

After installation, the following will appear on nodes:

    nvidia.com/gpu

Pods can request GPU resources in the following way:

    resources:
      limits:
        nvidia.com/gpu: 1

### 6.6.4 DCGM Exporter

Function:

    Exposes GPU metrics to Prometheus.

Typical Metrics:

    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_dev_FB_USED
    DCGM_FI_dev=DB_FREE
    DCGM_FI_DEV_gpu_TEMP
    DCGM_FI_DEV_POWER_usage
    DCGM_FI_DEV_XID_errors

Used for Monitoring:

    GPU utilization
    GPU video memory
    GPU temperature
    GPU power consumption
    GPU XID errors

### 6.6.5 NVIDIA GPU Operator

Function:

    Automates the management of GPU node-related components.

May include:

    NVIDIA Driver
    NVIDIA Container Toolkit
    NVIDIA Device Plugin
    DCGM Exporter
    GPU Feature Discovery

Note:

    If the NVIDIA GPU Operator is used, many GPU components can be installed and managed uniformly.
    If not using the NVIDIA GPU Operator, the Device Plugin and DCGM Exporter can still be installed manually.

---

### 6.7 Automation and Notification Components

### 6.7.1 Webhook Alert Gateway

Function:

    Receives AlertManager Webhooks
    Formats messages in a unified manner
    Integrates with enterprise communication tools like WeChat Work, DingTalk, or Lark
    Triggers automatic diagnostics
    Logs alert audits

Common Workflow:

    AlertManager
      ↓
    Webhook Gateway
      ↓
    WeChat Work / DingTalk / Lark / Tickets / OnCall

### 6.7.2 Runbook Documentation Library

Function:

    Provides guidance on how to handle alerts once they are triggered.

It is recommended that each significant alert include a:

    runbook_url

Examples:

    PodPendingTooLong
    PodRestartTooOften
    PodOOMKilled
    ServiceHigh5xxErrorRate
    NodeNotReady
    GPUXIDError

### 6.7.3 Automatic Diagnosis Service

Function:

    Receives alerts
    Queries Kubernetes APIs
    Retrieves data from Prometheus
    Accesses Loki/ELK systems
    Generates diagnostic reports
    Sends them to on-duty personnel

Automatic diagnosis should ideally be set to read-only mode and should not directly modify production resources.

---

### 6.8 Minimum Installation Configuration

### 6.8.1 For Basic Kubernetes Metric Monitoring Only

At minimum, the following are required:

    kube-prometheus-stack
    Prometheus
    AlertManager
    Grafana
    node-exporter
    kube-state-metrics
    kubelet/cAdvisor

Optional:

    Metrics Server
    blackbox-exporter

### 6.8.2 For Pod Log Collection Using Loki

Required components:

    Loki
    Grafana Alloy or Promtail
    Grafana Loki Data Source

Recommended setup:

    Grafana Alloy + Loki + Grafana

### 6.8.3 For Pod Log Collection Using EFK

Required components:

    Elasticsearch or OpenSearch
    Kibana or OpenSearch Dashboards
    Filebeat or Fluent Bit

Optional:

    Logstash
    Kafka

### 6.8.4 For GPU Monitoring

Required components:

    NVIDIA Driver
    NVIDIA Container Toolkit
    NVIDIA Device Plugin or NVIDIA GPU Operator
    DCGM Exporter
    Prometheus
    Grafana

### 6.8.5 For Comprehensive Production-Ready Observability

Recommended combination:

    kube-prometheus-stack
    Metrics Server
    blackbox-exporter
    Loki
    Grafana Alloy
    Or Elasticsearch/OpenSearch + Fluent Bit
    AlertManager
    Grafana
    DCGM Exporter
    Webhook Alert Gateway
    Runbook Documentation
    Automatic Diagnosis Service

### 6.9 Key Concepts to Understand

It is important to distinguish between the following components:

    Prometheus:
        Collects metrics but does not collect logs.

    Loki/ELK:
        Collect and query logs butThe status should be:

    UP

If it shows DOWN, proceed to the troubleshooting section for resolution.

### 8.3 Basic Metric Verification

On the Prometheus query page, enter the following metrics in sequence:

    up

    node_cpu_seconds_total

    node_memory_MemAvailable_bytes

    container_cpu_usage_seconds_total

    kube_pod_status_phase

    kube_node_status_condition

If data is available for all of them, it indicates that the basic collection process is functioning normally.

---

## IX. Practical Guide to Node Metric Collection

### 9.1 Node Ready Status

Metric Source:

    kube-state-metrics

PromQL:

    kube_node_status_condition{condition="Ready", status="true"}

Value of 1:

    Node is Ready

Value of 0:

    Node is NotReady

To check for NotReady nodes:

    kube_node_status_condition{condition="Ready", status="true"} == 0

### 9.2 Whether a Node Is Cordoned

Metric:

    kube_node_spec_unschedulable

Query:

    kube_node_spec_unschedulable == 1

This indicates that the node has been set to unschedulable.

Corresponding command:

    kubectl get nodes

If a node displays:

    SchedulingDisabled

it means it is cordoned.

### 9.3 Node CPU Usage Rate

Metric Source:

    node-exporter

Original Metric:

    node_cpu_seconds_total

PromQL:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

Explanation:

    `node_cpu_seconds_total` is a Counter.
    The `rate()` function must be used.
    `mode="idle"` refers to idle CPU usage.
    Subtracting the idle percentage from 100 gives the CPU usage rate.

To display by node:

    100 - (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

### 9.4 Node Memory Usage Rate

Metric Source:

    node-exporter

PromQL:

    (
      1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
    ) * 100

Explanation:

    `MemAvailable` is a more accurate indicator of available memory in Linux systems than `MemFree`.
    Linux uses caching, so relying solely on `free` may be misleading.

### 9.5 Node Disk Usage Rate

PromQL:

    (
      1 - node_filesystem_avail_bytes{fstype!="tmpfs"}
      / node_filesystem_size_bytes{fstype!"tmpfs"}
    ) * 100

It is recommended to filter out the following types of disks:

    tmpfs
    overlay
    proc
    sysfs

A more precise example:

    (
      1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
      / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
    ) * 100

### 9.6 Node Inode Usage Rate

PromQL:

    (
      1 - node_filesystem_files_free{fstype!~"tmpfs|overlay|proc|sysfs"}
      / node_filesystem_files{fstype!~"tmpfs|overlay|proc|sysfs"}
    ) * 100

High inode usage can lead to various issues, including:

- Inability to create files in Pods;
- Log writing failures;
- Image pull failures;
- Application errors;
- kubelet anomalies.

It is essential to monitor both disk capacity and inode usage.

### 9.7 Node Network Receive Rate

PromQL:

    rate(node_network_receive_bytes_total{device!~"lo"}[5m])

Aggregated by node:

    sum by (instance) (
      rate(node_network.receive_bytes_total{device!~"lo"}[5m])
    )

### 9.8 Node Network Send Rate

PromQL:

    sum by (instance) (
      rate(node_network_transmit_bytes_total{device!~"lo"}[5m])
    )

### 9.9 Node Network Error Packets

Receive errors:

    rate(node_network_receive_errs_total{device!~"lo"}[5m])

Send errors:

    rate(node_network Transmit_errs_total{device!~"lo"}[5m])

If the number of network error packets continues to increase, investigate potential issues with:

- Network cards;
- Drivers;
- Switches;
- MTU settings;
- Packet loss;
- Duplex mode;
- Physical connections;
- Virtual networks.

### 9.10 Node Load

PromQL:

    node_load1
    node_load5
    node_load1- Increase node resources.

### 10.4 Issue: Full Node Disk

PromQL:

    (
      1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
      / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
    ) * 100 > 85

Node troubleshooting:

    df -h
    df -i
    du -xh /var/lib/containerd | sort -h | tail
    du -xh /var/log | sort -h | tail
    crictl images
    crictl ps -a

Common causes:

- Large container logs;
- Uncleaned images;
- Large containerd data directories;
- Excessive usage of emptyDir;
- Applications writing to local files;
- Growth in Prometheus/log system local disks;
- Large node system logs.

Solutions:

- Clean up unused images;
- Configure log rotation;
- Limit the use of emptyDir;
- Adjust application log output settings;
- Move local data to other storage;
- Expand disk capacity;
- Set up disk monitoring and alerts.### 12.4 Phenomenon: High Pod CPU Usage

PromQL:

```plaintext
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
)
```

Viewing the Top:

```plaintext
topk(10,
  sum by (namespace, pod) (
    rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
  )
)
```

kubectl:

```plaintext
kubectl top pod -A --sort-by=cpu
```

Problem-solving approaches:

- Check if it is during business peak hours;
- Verify if there are any infinite loops;
- Check for GC exceptions;
- Confirm if there is excessive log output;
- Assess whether the CPU request is too low;
- Determine if scaling up replicas is necessary;
- Consider implementing rate limiting if required.

---

## Chapter Thirteen: Service Metric Collection and Understanding

A Service itself does not equate to business availability.

A Service merely serves as a stable entry point and load balancing mechanism.

Service-related monitoring should be combined with the following components:

    Service
    EndpointSlice / Endpoints
    Pod Ready status
    kube-proxy
    CoreDNS
    Ingress / Gateway
    Application/metrics
    Blackbox testing

### 13.1 Service Object Metrics

kube-state-metrics may expose the following metrics:

    kube_service_info
    kube_service_created
    kube_service_spec_type
    kube_service_spec_cluster_ip

To query a Service:

```plaintext
kubectl service info <service-name>
```

To view by Namespace:

```plaintext
kubectl service info --namespace=<namespace> <service-name>
```

### 13.2 Endpoint Metrics

Possible metrics include:

    kube_endpoint_info

Or EndpointSlice-related metrics, depending on the version of kube-state-metrics.

To view Endpoints:

```plaintext
kubectl get endpoints -n <namespace>
kubectl get endpointslice -n <namespace>
```

If a Service does not have Endpoints, common reasons include:

- Incorrect selector;
- Mismatch between Pod labels and Service selector;
- Pod is not in the "Ready" state;
- Pod does not exist;
- Service and Pod are in different Namespaces;
- Mismatch in targetPort may cause no Endpoints to be displayed, but it will lead to connection failures.

### 13.3 Checking if Service Backend Pods Are Ready

To verify:

```plaintext
kubectl get pod -n <namespace> --show-labels
kubectl describe svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>
kubectl get endpointslice -n <namespace> | grep <service-name>
```

### 13.4 Sources of Service Business Metrics

Real business metrics at the Service level usually come directly from the application itself.

Examples include:

    http_requests_total
    http_request_duration_seconds_bucket
    httprequests_errors_total

To calculate QPS:

```plaintext
sum by (service) (
  rate(http_requests_total[5m])
)
```

To calculate error rate:

```plaintext
sum by (service) (
  rate(http_requests_total{status=~"5.."}[5m])
)
/
sum by (service) (
  rate(httprequests_total[5m])
)
* 100
```

To calculate P95 latency:

```plaintext
histogram_quantile(
  0.95,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

If the application does not expose these business metrics, it will be difficult to assess the actual quality of the Service using Prometheus.

### 13.5 Blackbox Testing

The blackbox-exporter can be used to externally verify:

- Whether HTTP responses are 200;
- Whether TCP ports are accessible;
- Whether DNS resolutions work;
- Whether ICMP connections are successful;
- Whether TLS certificates are valid.

This is useful for supplementing Service availability monitoring.

Example scenarios include:

    Testing Ingress addresses
    Checking Service NodePort connectivity
    Verifying internal HTTP APIs
    Testing database port accessibility
    Checking DNS resolution

---

## Chapter Fourteen: Practical Service Troubleshooting

### 14.1 Phenomenon: The Service Exists, but No Backend is Found

To verify:

```plaintext
kubectl get svc -n <namespace>
kubectl describe svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>
kubectl get endpointslice -n <namespace> | grep <service-name>
```

Common reasons include:

- Incorrect Service selector;
- Mismatch between Pod labels and Service selector;
- Pod is not in the "Ready" state;
- Pod is located in a different Namespace;
- Pod has been deleted;
- The Deployment has no replicas;
- readinessProbe failures.

Solution steps:

    1. Verify the Service selector.
    12. NetworkPolicy
13. kube-proxy / CNI

Common Commands:

    kubectl describe svc <service-name> -n <namespace>
    kubectl get endpoints <service-name> -n <namespace>
    kubectl get pod -n <namespace> -o wide
    kubectl logs <pod-name> -n <namespace>
    kubectl exec -it <pod-name> -n <namespace> -- ss -lntp

Temporary Test:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

---

## Chapter Fifteen: Troubleshooting Prometheus Targets

### 15.1 Issue: Target DOWN

Prometheus Page:

    Status
      ↓
    Targets

If you see a target listed as:

    DOWN

Follow these steps to troubleshoot:

    1. Check if the target Pod is Running.
    2. Verify if the target Service exists.
    3. Confirm whether the Service selector matches the Pod.
    4. Ensure that there are endpoints with backend services.
    5. Check if the port names match.
    6. Verify if the /metrics endpoint is accessible.
    7. Confirm if the ServiceMonitor matches the settings.
    8. Check if NetworkPolicy rules are blocking access.
    9. Verify if Prometheus has the necessary RBAC permissions.
    10. Ensure that TLS/Bearer Token configuration is correct.

### 15.2 Checking ServiceMonitors

    kubectl get servicemonitor -A
    kubectl describe servicemonitor <name> -n <namespace>

Check the following:

- namespaceSelector;
- selector.matchLabels;
- endpoints.port;
- endpoints.path;
- interval;
- Ensure that labels are selected by Prometheus.

### 15.3 Viewing Service Labels

    kubectl get svc <service-name> -n <namespace> --show-labels
    kubectl describe svc <service-name> -n <namespace>

The ServiceMonitor's selector must match the Service's labels.

### 15.4 Checking Port Names

ServiceMonitors typically use port names, not port numbers.

Example Service configuration:

    ports:
      - name: metrics
        port: 9100
        targetPort: 9100

ServiceMonitor configuration:

    endpoints:
      - port: metrics

If the port name is incorrect, Prometheus may not be able to retrieve data.

### 15.5 Testing /metrics

Create a temporary Pod:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Access the /metrics endpoint:

    curl http://<service-name>.<namespace>.svc.cluster.local:<port>/metrics

If access fails, it indicates that the issue is not with Prometheus but rather with the target service or network connectivity.

---

## Chapter Sixteen: Troubleshooting Grafana Data Absence

### 16.1 Troubleshooting Steps

When Grafana shows no data, do not modify the dashboard immediately. Follow these steps:

    1. Verify if the time range is correct.
    2. Check if the Prometheus data source is functioning properly.
    3. Ensure that metrics are available in Prometheus.
    4. Confirm whether PromQL queries are correct.
    5. Verify if label values match.
    6. Check if any variables are set to null.
    7. Ensure that target services are Running.
    8. Verify if any metrics have been relabelled or dropped.
    9. Check if the dashboard is using outdated metric names.

### 16.2 Checking Directly in Prometheus

For example, if there is no data for GPU usage in Grafana:

First, check in Prometheus:

    DCGM_FI_DEV_GPU_UTIL

If even Prometheus shows no data, it indicates that the issue is not with Grafana.

If Prometheus provides data, review Grafana's PromQL queries and variable settings.

### 16.3 Checking Variables

Dashboard variables might be set to null.

Example:

    $namespace
    $pod
    $node

Verify if variable queries are correct:

    label_values(kube_pod_info, namespace)

In Grafana, go to:

    Dashboard Settings
      ↓
    Variables
      ↓
    Preview of values

Check if there are any valid values.

### 16.4 Checking Label Names

Node labels for different metrics may vary:

    instance
    node
    Hostname

Do not assume that all metrics use the same label name. Verify the actual label names using original metric names such as:

    node_cpu_seconds_total

    kube_node_info

    DCGM_FI_DEV_GPU_UTIL

---

## Chapter Seventeen: Designing Node Dashboards

### 17.1 Top Overview Panel

Recommended panels include:

    Total Number of Nodes
    Number of Ready Nodes
    Number of NotReady Nodes
           team: sre
      annotations:
        summary: "Node NotReady"
        description: "Node {{ $labels.node }} has been NotReady for more than 5 minutes."
        runbook_url: "https://wiki.example.com/runbook/node-not-ready"

### 20.2 High Node CPU Usage

    - alert: NodeHighCPUUsage
      expr: |
        100 - (
          avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          ) * 100
        ) > 90
      for: 10 minutes
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "High Node CPU Usage"
        description: "Node {{ $labels.instance }} has a CPU usage of over 90% for more than 10 minutes."

### 20.3 High Node Memory Usage

    - alert: NodeHighMemoryUsage
      expr: |
        (
          1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
        ) * 100 > 90
      for: 10 minutes
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "High Node Memory Usage"
        description: "Node {{ $labels.instance }} has a memory usage of over 90% for more than 10 minutes."

### 20.4 Insufficient Node Disk Space

    - alert: NodeDiskSpaceLow
      expr: |
        (
          1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
          / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|proc|sysfs"}
        ) * 100 > 85
      for: 15 minutes
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Insufficient Node Disk Space"
        description: "The usage of the filesystem {{ $labels.mountpoint }} on node {{ $labels.instance }} is over 85%."

### 20.5 Long-Pending Pods

    - alert: PodPendingTooLong
      expr: kube_pod_status_phase{phase="Pending"} == 1
      for: 10 minutes
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Pods Pending for Too Long"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a Pending state for more than 10 minutes."

### 20.6 Excessive Pod Reboots

    - alert: PodRestartTooOften
      expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
      for: 5 minutes
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Pods Restarting Too Frequently"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has restarted more than 3 times in 10 minutes."

### 20.7 Pod OOMKilled

    - alert: PodOOMKilled
      expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
      for: 1 minute
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Pod Killed Due to OOM"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} was terminated due to Out of Memory."

### 20.8 Down Prometheus Target

    - alert: PrometheusTargetDown
      expr: up == 0
      for: 5 minutes
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "Prometheus Target Down"
        description: "Target {{ $labels.job }} / {{ $labels.instance }} has been down for more than 5 minutes."

---

## Section Twenty-One: Service Availability Alerts

### 21.1 High 5xx Error Rate in Applications

Prerequisite:

    The application must expose `http_requests_total` and have a status tag.

Rule:

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
      for: 5 minutes
      labels:
        severity: critical
        team: app
      annotations:
        summary: "High 5xx Error Rate in Service"
        description: "Service {{ $labels.service }} has a 5xx error### 22.1 Observations

User feedback:

    Service access failed

Checking with kubectl:

    kubectl get svc -n app
    kubectl get pod -n app

Findings:

    The Service exists.
    The Pod is running.

### 22.2 Troubleshooting Steps

Step 1: Check Service details

    kubectl describe svc <service-name> -n app

Check the following:

    selector
    port
    targetPort
    type
    endpoints

Step 2: Check Endpoints

    kubectl get endpoints <service-name> -n app
    kubectl get endpointslice -n app | grep <service-name>

If Endpoints are empty:

    Verify the selector and Pod labels.

Step 3: Check Pod labels

    kubectl get pod -n app --show-labels

Confirm whether the Pod labels match the Service selector.

Step 4: Check if the Pod is Ready

    kubectl get pod -n app
    kubectl describe pod <pod-name> -n app

If the Pod is running but not ready:

    Verify the readinessProbe.

Step 5: Enter the Pod to check listening ports

    kubectl exec -it <pod-name> -n app -- sh
    ss -lntp

Confirm whether the application is listening on the targetPort.

Step 6: Access the Service from within the cluster

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside the container:

    curl -v http://<service-name>.app.svc.cluster.local:<port>

Step 7: Check application logs

    kubectl logs <pod-name> -n app

Step 8: Check business metrics

Using PromQL:

    rate(http_requests_total[5m])

    rate(http_requests_total{status=~"5.."}[5m])

    histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

### 22.3 Possible Conclusions

- The Service selector may be incorrect.
- The Pod labels may not match.
- The readinessProbe may have failed.
- The targetPort may be set incorrectly.
- The application may not be listening on 0.0.0.0.
- The container port may not match the Service port.
- There might be a NetworkPolicy blocking access.
- Ingress forwarding could be incorrect.
- There may be internal dependencies within the application that are failing.
- Database connections could be failed.
- DNS resolution issues might exist.

---

## Twenty-three: Complete Troubleshooting Case: Pod is Running but Prometheus Shows No Metrics

### 23.1 Observations

The application Pod is running.

Access to the service is normal.

However, Prometheus does not display any metrics for the application.

### 23.2 Troubleshooting Steps

Step 1: Check if the application exposes /metrics

    kubectl exec -it <pod-name> -n <namespace> -- sh
    curl http://127.0.0.1:<metrics-port>/metrics

Step 2: Check if the Service exposes the metrics port

    kubectl describe svc <service-name> -n <namespace>

Step 3: Verify if the Service port has a name

Example:

    ports:
      - name: metrics
        port: 9100
        targetPort: 9100

Step 4: Check if the ServiceMonitor matches the Service

    kubectl describe servicemonitor <servicemonitor-name> -n <namespace>

Verify the following:

    namespaceSelector
    selector.matchLabels
    endpoints.port
    endpoints.path

Step 5: Confirm whether Prometheus has selected the ServiceMonitor

If the kube-prometheus-stack has a label selector for the ServiceMonitor, check if the ServiceMonitor labels match.

Example:

    release: kube-prometheus-stack

Step 6: Check Prometheus Targets

On the Prometheus dashboard:

    Status -> Targets

Search for the application name.

### 23.3 Common Causes

- The application does not expose /metrics.
- The metrics path is different from /metrics.
- The Service does not have a metrics port exposed.
- The Service port does not have a name.
- The ServiceMonitor selector does not match.
- The ServiceMonitor namespaceSelector does not match.
- The ServiceMonitor lacks the release label.
- There might be a NetworkPolicy blocking access.
- Prometheus may not have sufficient RBAC permissions.
- Relabel configuration could be filtering out the target.

---

## Twenty-four: Complete Troubleshooting Case: Pod Pending Alarm

### 24.1 Alarm

The AlertManager received:

    PodPendingTooLong

### 24.2 Prometheus Query

    kube_pod_status_phase{phase="Pending"} == 1

Locate the issue:

    namespace
    pod

### 24.3 k      expr: |
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
          container_memory-working_set_bytes{container!="", image!=""}
        )

### 26.5 Service Error Rate

    - record: service:http_5xx_error_rate:percent
      expr: |
        sum by (service) (
          rate(http_requests_total{status=~"5.."}[5m])
        )
        /
        sum by (service) (
          rate(httprequests_total[5m])
        )
        * 100

---

## Twenty-Seven, Production Monitoring Metrics Baselines

### 27.1 Nodes That Must Be Monitored

    [ ] Node Readiness
    [ ] Node CPU Usage
    [ ] Node Memory Usage
    [ ] Node Disk Usage
    [ ] Node Inode Usage
    [ ] Node Network Traffic
    [ ] Node Network Error Packets
    [ ] Node Load
    [ ] kubelet Status
    [ ] containerd Status
    [ ] Whether the Node Is Cordoned
    [ ] Node Resource Pressure

### 27.2 Pods That Must Be Monitored

    [ ] Pod Phase
    [ ] Pod Readiness
    [ ] Pod Pending
    [ ] Pod Failed
    [ ] Pod Restart
    [ ] Pod OOMKilled
    [ ] Pod CPU Usage
    [ ] Pod Memory Usage
    [ ] Pod Network Traffic
    [ ] Number of Available Deployment Replicas
    [ ] Number of Available StatefulSet Replicas
    [ ] Number of Ready DaemonSets

### 27.3 Services That Must Be Monitored

    [ ] Whether the Service Exists
    [ ] Whether the Endpoint Is Empty
    [ ] Whether the Backend Pod Is Ready
    [ ] Application QPS
    [ ] Application 5xx Error Rate
    [ ] P95 / P99 Latency
    [ ] Blackbox Detection Success Rate
    [ ] Ingress Status
    [ ] DNS Resolution Availability

### 27.4 Monitoring System Itself That Must Be Monitored

    [ ] Prometheus Target UP
    [ ] Prometheus TSDB Head Series
    [ ] Prometheus Storage Space
    [ ] Failure in Calculating Prometheus Rules
    [ ] AlertManager Status
    [ ] Grafana Status
    [ ] node-exporter Status
    [ ] kube-state-metrics Status
    [ ] Whether ServiceMonitor Is Effective

---

## Twenty-Eight, Production Alarm Baselines

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

## Twenty-Nine, Common Misconceptions

### 29.1 Misconception 1: If kubectl top Shows Data, There Is No Need for Prometheus

Wrong.

kubectl top is from the Metrics Server and is suitable for viewing current CPU/Memory usage.

Prometheus is used for:

- Long-term trends;
- Dashboards;
- Alerts;
- Complex PromQL queries;
- Business metrics;
- GPU metrics;
- Capacity planning.

### 29.2 Misconception 2: If a Pod Is Running, ItMeans the Business Is Working Fine

Wrong.

A Pod being running only indicates that the container process exists.

Whether the business is working properly also depends on:

- readinessProbe results;
- Application logs;
- Service Endpoints;
- QPS;
- Error rates;
- Latency;
- Dependent services;
- Database connections;
- Blackbox testing results.

### 29.3 Misconception 3: If a Service Has a ClusterIP, ItMeans the Service Is Available

Wrong.

Just```markdown
kubectl get endpointslice -n <namespace>

### 30.6 Testing DNS and Service

    kubectl run dns-test --rm -it --image=busybox:1.36 -- sh

Inside the container:

    nslookup kubernetes.default.svc.cluster.local
    nslookup <service-name>.<namespace>.svc.cluster.local

HTTP testing:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside the container:

    curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>

### 30.7 Troubleshooting on the Node Side

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

## Section Thirty-One: Production Implementation Recommendations

### 31.1 Metric Collection Recommendations

It is recommended to use uniformly:

    kube-prometheus-stack
    node-exporter
    kube-state-metrics
    kubelet/cAdvisor
    Metrics Server
    blackbox-exporter
    dcgm-exporter
    Application /metrics

### 31.2 Log Collection Recommendations

Loki route:

    Grafana Alloy
      ↓
    Loki
      ↓
    Grafana

EFK route:

    Fluent Bit / Filebeat
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

Recommendations for selection:

    For operational troubleshooting, Grafana integration, and cost sensitivity:
        Loki is more suitable.

    For full-text search, auditing, security, and complex field searches:
        Elasticsearch / OpenSearch is more appropriate.

### 31.3 Dashboard Recommendations

At least create the following dashboards:

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

### 31.4 Alarm Recommendations

Alarms must meet the following requirements:

    Have an owner
    Have a severity level
    Include a summary
    Contain a description
    Have a runbook_url
    Have a dashboard_url
    Be scheduled at appropriate times
    Specify clear action items

### 31.5 Runbook Recommendations

At least create the following runbooks:

    NodeNotReady
    NodeHighCPUUsage
    NodeHighMemoryUsage
    NodeDiskSpaceLow
    PodPendingTooLong
    PodCrashLoopBackOff
    PodOOMKilled
    ServiceNoEndpoint
    TargetDown
    CoreDNSUnavailable
    GPUXIDError
    PodLogsMissing
    AppErrorLogsTooMany

### 31.6 Multi-Team Standards

It is recommended that:

- Each Namespace have an owner;
- Each Service have an owner;
- Each alarm have a team label;
- Each dashboard have a responsible person;
- Each runbook have a maintainer;
- Alarm changes be tracked via Git;
- Dashboard JSON files be stored in Git;
- Prometheus Rules be managed in Git;
- Log collection settings also be version-controlled via Git.

---

## Section Thirty-Two: Acceptance Checklist

### 32.1 Metric Collection Chain

    [ ] node-exporter is running
    [ ] kube-state-metrics is running
    [ ] kubelet is running
    [ ] cAdvisor metrics are available
    [ ] Application /metrics data is accessible
    [ ] dcgm-exporter is running
    [ ] Prometheus own metrics are available

### 32.2 Node Metrics

    [ ] Node Ready status is available
    [ ] CPU usage is available
    [ ] Memory usage is available
    [ ] Disk usage is available
    [ ] Inode usage is available
    [ ] Network traffic is available
    [ ] Load value is available

### 32.3 Pod Metrics

    [ ] Pod Phase is available
    [ ] Pod Ready status is available
    [ ] Pod CPU usage is available
    [ ] Pod memory usage is available
    [ ] Pod restart history is available
    [ ] OOMKilled events are available
    [ ] Pending Pods are visible

### 32.4 Service Metrics

    [ ] Service objects are available
    [ ] Endpoints are available
    [ ] EndpointSlice information is available
    [] Application QPS is available
    [] Application error rate is available
    [] P95/P99 latency values are available
    [ ] Blackbox detection results are available

### 32.5 Log Collection Chain

    [ ] The log agent is running as a DaemonSet
    [View the previous container instance logs:

    kubectl logs <pod-name> -n <namespace> --previous

View the specified container logs:

    kubectl logs <pod-name> -n <namespace> -c <container-name>

View the last 100 lines:

    kubectl logs <pod-name> -n <namespace> --tail=100

Continuously monitor:

    kubectl logs <pod-name> -n <namespace> -f

---

### 33.3 Pod Log Collection Pipeline

Centralized logging systems typically use DaemonSets to collect logs on each node.

Typical pipeline:

    Pod stdout / stderr
      ↓
    containerd writes to node log files
      ↓
    /var/log/containers/*.log
      ↓
    Log collection agent
      ↓
    Loki / Elasticsearch / OpenSearch
      ↓
    Grafana / Kibana for querying

Common agents:

    Grafana Alloy
    Promtail
    Fluent Bit
    Filebeat
    Fluentd

Loki route:

    Pod Logs
      ↓
    Alloy / Promtail
      ↓
    Loki
      ↓
    Grafana Explore

EFK route:

    Pod Logs
      ↓
    Filebeat / Fluent Bit
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

---

### 33.4 Tags/Fields to Retain for Pod Log Collection

Whether using Loki or EFK, it is recommended to retain the following Kubernetes metadata:

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
    Which Container
    Which Node
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

    kubernetes(namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"
    and log.level : "error"

Do not use the following high-frequency fields as Loki labels:

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message
    stacktrace

These fields can be saved as part of the log content or in Elasticsearch fields, but they are not suitable for indexing with Loki.

---

### 33.5 Pod Monitoring and Log Integration: CrashLoopBackOff

Phenomenon:

    The Pod restarts repeatedly.
    The status may show "CrashLoopBackOff".

Indicators that may indicate this issue:

    increase(kube_pod_container_status_restarts_total[10m]) > 3

Troubleshooting with kubectl:

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

Key considerations:

- What is the Last State Reason?
- What is the Exit Code?
- Are there any startup failure messages in the previous logs?
- Are there any probe failures recorded in Events?
- Are ConfigMap/Secrets missing?
- Are there database connection issues?
- Are there port conflicts?
- Are there permission errors?
- Are there any issues with the image startup command?

Typical conclusions:

    The Pod's restart is not the root cause; it is a result of another problem.
    The actual cause can often be found in the previous logs, Events, Exit Code, and application startup logs.

---

### 33.6 Pod Monitoring and Log Integration: OOMKilled

Phenomenon:

    The Pod is killed due to Out Of Memory (OOM).
    The container restarts.
    The service becomes temporarily unavailable.

Indicators that may indicate this issue:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

Check memory trends:

    sum by (namespace, pod) (
      container_memory_working_set### 33.8 Pod Log Collection Exception Troubleshooting

Phenomenon:

`kubectl logs` shows logs, but Loki/Kibana cannot find them.

Troubleshooting Steps:

1. Check if the log collection Agent is running.
2. Verify if the Agent is deployed as a DaemonSet on that node.
3. Confirm whether `/var/log/containers` is correctly mounted.
4. Ensure the Agent has permission to read the log files.
5. Check if the Agent has successfully added Kubernetes metadata.
6. Verify if the specific namespace or container is being filtered out.
7. Confirm whether Loki/Elasticsearch can write logs.
8. Check if the query time range is correct.
9. Verify if the label/field names used in the query are accurate.

Viewing the Agent:

`kubectl get pods -n logging -o wide`
`kubectl get ds -n logging`

Viewing Agent Logs:

`kubectl logs <agent-pod> -n logging`

Checking Mounts within the Agent Pod:

`kubectl exec -it <agent-pod> -n logging -- sh`
`ls -l /var/log/containers/`

If no logs are available for any Pod on a node, prioritize checking if the Agent Pod on that node is running.

If there are no logs for a particular namespace, first check if the collection rules exclude that namespace.

If only one application lacks logs, verify whether it actually outputs stdout/stderr.

Command:

`kubectl logs <pod-name> -n <namespace>`

If `kubectl logs` returns nothing, it indicates that the application may not be producing standard logs at all.

---

### 33.9 Pod Monitoring and Log-Linked Dashboard Recommendations

In addition to CPU, memory, and restart counts, it is recommended to include log access in Pod dashboards.

Recommended Panels:

- Pod Phase
- Pod Ready
- Pod Restart Count
- Pod CPU Usage
- Pod Memory Working Set
- Pod Network Receive/Transmit
- Last Terminated Reason
- OOMKilled Status
- Recent Events
- Recent Error Logs
- Recent Warning Logs
- Log Link to Loki Explore
- Log Link to Kibana Discover

Grafana Variables:

- `$namespace`
- `$pod`
- `$container`
- `$node`
- `$app`

Loki Query Panel:

- `{namespace="$namespace", pod="$pod"}`

Error Logs Panel:

- `{namespace="$namespace", pod="$pod"} |~ "(?i)error|exception|panic|traceback|timeout"`

If using EFK, include Kibana linklets in the dashboard:

- `namespace=<namespace>`
- `pod=<pod>`
- `time range=last 30 minutes`

Goal:

- Allow one-click navigation from Pod metric charts to corresponding log records.
- Provide one-click access from alerts to related dashboards and logs for quick troubleshooting.

---

### 33.10 Pod Monitoring and Log-Linked Alerts Recommendations

Metric Alerts:

- `PodPendingTooLong`
- `PodRestartTooOften`
- `PodOOMKilled`
- `PodNotReadyTooLong`
- `PodHighCPUUsage`
- `PodHighMemoryUsage`

Log Alerts:

- `AppErrorLogsTooMany`
- `AppTimeoutLogsTooMany`
- `PythonTracebackDetected`
- `JavaExceptionLogsTooMany`
- `DatabaseConnectionErrorLogs`
- `CUDALogOOMDetected`
- `ModelLoadFailedLogs`

Recommended Approach:

- Metric alerts detect abnormal conditions.
- Log alerts provide detailed error causes.
- The AlertManager handles grouping, deduplication, and suppression of alerts.
- Runbooks guide the response to these alerts.
- Dashboards display trends over time.
- Loki/ELK provide context around log events.

Example: Associating Pod Restart Alerts with Log Alerts

- `PodRestartTooOften`
  +
  `AppErrorLogsTooMany`
  +
  `PythonTracebackDetected`

If they share common criteria such as `cluster`, `namespace`, `app`, and `pod`, try grouping them in the AlertManager to avoid duplicate notifications.

---

### 33.11 Pod Monitoring and Log-Linked Acceptance Checklist

- [ ] Pod Phase metrics are available for viewing.
- [ ] Pod Ready metrics are available for viewing.
- [ ] Pod CPU metrics are available for viewing.
- [ ] Pod Memory metrics are available for viewing.
- [ ] Pod Restart metrics are available for viewing.
- [ ] Pod OOMKilled metrics are available for viewing.
- [ ] `kubectl logs` allows access to business logs.
- [ ] `kubectl logs --previous` shows previous container logs.
- [ ] The `/var/log/containers` path exists.
- [ ] The log collection Agent is running as a DaemonSet.
- [ ] The Agent is deployed on all nodes.
- [ ] Loki/ELK can query logs by namespace and pod.
- [ ] Loki/ELK can query logsAre the resources sufficient?
Are kubelet/containerd functioning normally?
Is the node still suitable for hosting Pods?

Pod Monitoring Answers:

Is the workload running properly?
Is it in a Pending state?
Has it been restarted?
Has it been killed due to Out of Memory (OOM)?
Are there any abnormalities in resource usage?

Service Monitoring Answers:

Is service discovery correct?
Do the Endpoints exist?
Are the backend Pods ready?
Have business requests been successful?
Are the error rates and latency within normal ranges?

Pod Logs Additional Answers:

Why did the application fail to start?
Why were dependency connections unsuccessful?
Why did the probes fail?
What happened before the OOM or CrashLoop occurred?
Why are there still issues with the service while it is in a Running state?

It is essential to distinguish the sources of various metrics:

node-exporter:
Host resource metrics.

kubelet/cAdvisor:
Container resource metrics.

kube-state-metrics:
Kubernetes object status metrics.

application /metrics:
Business quality metrics.

blackbox-exporter:
External availability checks.

dcgm-exporter:
GPU metrics.

Loki / ELK:
Pod logs, error context, and root causes of anomalies.

When troubleshooting, do not focus solely on a single metric. The correct approach is as follows:

Prometheus metric abnormalities detected
      ↓
    Trend analysis using Grafana
      ↓
    Notification collection via AlertManager
      ↓
    Check events using kubectl describe
      ↓
    View application logs with kubectl logs / --previous
      ↓
    Query cross-replica logs using Loki / ELK
      ↓
    Verify service/Endpoints/DNS links
      ↓
    Confirm underlying status via node commands
      ↓
    Execute corresponding Runbooks
      ↓
    Conduct post-troubleshooting and optimize alerts.

Production-grade Kubernetes monitoring should achieve the following:

Complete set of metrics
Accessible logs
Clear dashboards
Accurate alerts
Reliable notifications
Well-defined troubleshooting procedures
Runbooks for issue resolution
Ability to learn from past incidents
Capacity planning capabilities

---

## Section 35: Reference Documents

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