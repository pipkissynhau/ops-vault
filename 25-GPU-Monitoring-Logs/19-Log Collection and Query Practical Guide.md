# 19-Pod Monitoring and Log Collection Specialized Practice

## Document Overview

This document is used for specialized organization of Kubernetes Pod dimension monitoring, log collection, event troubleshooting, Dashboard, alert rules, and production fault diagnosis methods.

The 14th article already discussed K8S monitoring methods from the Node / Pod / Service three objects.

The 15th article discusses Loki log collection and query.

The 16th article discusses ELK / EFK log collection and retrieval.

The 17th article discusses log alerts and automated response.

The 18th article discusses comprehensive cases combining monitoring, GPU, and logs.

This article, as the 19th, does not repeat the large-scale content of previous chapters, but focuses on a core question:

    When a Pod has an issue, how to simultaneously view metrics, logs, Events, Service backend, alerts, and Dashboard?

This article focuses on answering the following questions:

- What metrics should be monitored for a Pod?
- Where do Pod logs come from?
- What's the difference between kubectl logs and centralized log platforms?
- What's the relationship between /var/log/containers and /var/log/pods?
- What do Prometheus, Metrics Server, Loki, and EFK respectively handle?
- How to troubleshoot Pod Pending, CrashLoopBackOff, OOMKilled, and Running but business anomalies?
- How to query a Pod's logs via Loki / Grafana?
- How to query a Pod's logs via EFK / Kibana?
- How to design a Pod Dashboard?
- How to design Pod-level alerts?
- How to link Pod metrics, Pod logs, Pod Events, Service Endpoints into a troubleshooting chain?
- How to perform production-level Pod observability acceptance?

This document is suitable for study after completing the following content:

- 14-K8S-Monitoring Practice-Node-Pod-Service Metric Collection and Troubleshooting
- 15-Loki-Log Collection and Query Practice
- 16-ELK-EFK-Log Collection and Retrieval Practice
- 17-Log Alert and Automated Response Practice
- 18-Monitoring-GPU-Log Combined Case-K8S-Pod Anomaly Detection and Report Generation

---

## Tags

#Kubernetes #PodSurveillance. #PodLog #Prometheus #Grafana #Loki #EFK #Filebeat #FluentBit #GrafanaAlloy #MetricsServer #kube-state-metrics #cAdvisor #SRE #FaultCheck. #Observation

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Foundation/19-Pod Monitoring and Log Collection Specialized Practice.md

---

## One, Why a Specialized Practice for Pod Monitoring and Log Collection is Needed

In Kubernetes, Pod is the smallest scheduling unit for business operations.

Most production faults ultimately need to be confirmed at the Pod level:

    Did the Pod schedule successfully?
    Is the Pod Running?
    Is the Pod Ready?
    Is the Pod frequently restarting?
    Was the Pod OOMKilled?
    Is the Pod connected to Service?
    Does the Pod have Endpoints?
    What errors are reported in the Pod's application logs?
    Is the Pod's node abnormal?
    Are the Pod's CPU / Memory usage abnormal?
    Was the Pod killed by a probe?
    Did the Pod fail to start due to configuration, permissions, image, or dependencies?

Only looking at a single signal is insufficient.

For example:

    Pod Running:
        Only indicates the container process exists, not necessarily business normality.

    Pod Ready:
        Only indicates the readinessProbe passed, not necessarily all dependencies are normal.

    Service has Endpoints:
        Only indicates Service backend exists, not necessarily interface errors.

    kubectl logs has errors:
        Only indicates application errors, not necessarily affecting traffic and availability.

    CPU / Memory normal:
        Only indicates resources are not obviously abnormal, not necessarily business logic normal.

Therefore, Pod-level troubleshooting must combine the following information:

    Pod status
    Pod metrics
    Pod Events
    Pod logs
    Pod's node
    Pod's workload
    Pod's Service
    Pod's Endpoints entry
    Application business metrics
    Alerts and Dashboard

The goal of this article is to form a fixed troubleshooting model:

    Metrics show phenomena
    Events show Kubernetes behavior
    Logs show application causes
    Service / Endpoints show traffic backend
    Node shows underlying resources
    Dashboard shows trends
    Runbook shows handling steps

---

## Two, Complete Observability Model for Pod

The complete observability of a Pod can be divided into seven layers.

### 2.1 First Layer: Object Status

Focus on:

    Pod Phase
    Pod Ready
    Container State
    Container Last State
    Restart Count
    Reason
    Exit Code

Common commands:

    kubectl get pod <pod-name> -n <namespace> -o wide

    kubectl describe pod <pod-name> -n <namespace>

Common metrics:

    kube_pod_status_phase
    kube_pod_container_status_ready
    kube_pod_container_status_restarts_total
    kube_pod_container_status_last_terminated_reason

### 2.2 Second Layer: Resource Metrics

Focus on:

    CPU usage
    Memory usage
    CPU request / limit
    Memory request / limit
    Network receive
    Network transmit
    Container filesystem usage

Common commands:

    kubectl top pod <pod-name> -n <namespace>

PromQL:

    container_cpu_usage_seconds_total
    container_memory_working_set_bytes
    container_network_receive_bytes_total
    container_network_transmit_bytes_total

### 2.3 Third Layer: Kubernetes Events

Events record what Kubernetes does to a Pod.

Focus:

    Scheduling failure
    Failed to pull image
    Probe failure
    Mount failure
    PVC not bound
    OOMKilled
    Container startup failure
    Node resource insufficient

Commands:

    kubectl describe pod <pod-name> -n <namespace>

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

Events common content:

    FailedScheduling
    FailedMount
    FailedPull
    BackOff
    Unhealthy
    Killing
    Evicted

### 2.4 Fourth Layer: Pod Logs

Focus:

    Application startup logs
    Current container logs
    Previous container logs
    ERROR
    Exception
    Traceback
    panic
    timeout
    connection refused
    database connection failed
    CUDA out of memory

Commands:

    kubectl logs <pod-name> -n <namespace>

    kubectl logs <pod-name> -n <namespace> --previous

    kubectl logs <pod-name> -n <namespace> -c <container-name>

Centralized logging:

    Loki
    Elasticsearch
    OpenSearch

### 2.5 Fifth Layer: Service Backend

Focus:

    Whether Service selector matches Pod label
    Whether Pod enters Endpoints
    Whether Pod is Ready
    Whether targetPort is correct
    Whether Service is accessible
    Whether DNS resolves normally

Commands:

    kubectl describe svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

    kubectl get endpointslice -n <namespace> | grep <service-name>

### 2.6 Sixth Layer: Workload Controller

Pods generally do not exist independently, but are managed by controllers.

Common controllers:

    Deployment
    StatefulSet
    DaemonSet
    Job
    CronJob

Focus:

    Desired replica count
    Current replica count
    Ready replica count
    Available replica count
    Rolling update status
    Version change
    ReplicaSet history

Commands:

    kubectl get deploy -n <namespace>

    kubectl describe deploy <deployment-name> -n <namespace>

    kubectl rollout status deploy/<deployment-name> -n <namespace>

    kubectl rollout history deploy/<deployment-name> -n <namespace>

### 2.7 Seventh Layer: Business Metrics

Normal resource status at Pod level does not guarantee business normalcy.

Also check application-specific metrics:

    QPS
    5xx error rate
    P95 / P99 latency
    Request success rate
    Queue length
    Current concurrency
    Dependency call error rate
    Database connection pool
    Redis connection pool
    Model inference latency

Typical metrics:

    http_requests_total
    http_request_duration_seconds_bucket
    http_requests_errors_total
    app_queue_length
    app_inflight_requests

---

## Three: Relationship Between Pod Monitoring and Log Collection Components

### 3.1 What Does Prometheus Handle

Prometheus is responsible for collecting metrics.

It does not collect Pod logs.

Prometheus can collect:

    Metrics exposed by kube-state-metrics for Pod status
    Metrics exposed by kubelet / cAdvisor for container resources
    Business metrics exposed by application /metrics
    Node metrics exposed by node-exporter
    GPU metrics exposed by dcgm-exporter

Prometheus is suitable for answering:

    Is Pod Pending?
    Has Pod restarted?
    Is Pod CPU high?
    Is Pod memory high?
    Has Pod been OOMKilled?
    Is service error rate high?
    Has latency increased?

### 3.2 What Does Metrics Server Handle

Metrics Server is responsible for providing real-time resource metrics to Kubernetes Metrics API.

Common commands:

    kubectl top node
    kubectl top pod -A

Metrics Server main services:

    kubectl top
    HPA
    Partial capabilities of VPA

Notes:

    Metrics Server is not suitable for long-term metric storage.
    Metrics Server is not suitable for complex alerts.
    Metrics Server does not replace Prometheus.

### 3.3 What Does kube-state-metrics Handle

kube-state-metrics is responsible for converting Kubernetes object status into Prometheus metrics.

It is suitable for answering:

    Is Pod Pending?
    Is Pod Running?
    Is Pod Ready?
    How many times has Pod restarted?
    What was the last reason Pod was terminated?
    Does Deployment available replica count meet requirements?
    Is PVC Pending?

It does not collect:

    CPU usage
    Memory usage
    Log content

### 3.4 What Does kubelet / cAdvisor Handle

kubelet / cAdvisor is responsible for exposing container resource usage metrics.

It is suitable for answering:

    How much CPU does Pod use?
    How much memory does Pod use?
    How much network traffic does Pod use?
    How much file system does container use?

### 3.5 What Does Loki / ELK Handle?

Loki / ELK handles log collection, storage, and querying.

They are suitable for answering:

    Why did the application fail to start?
    What logs were output before the Pod entered CrashLoopBackOff?
    Was there a large request before OOM?
    Did database connection failed occur?
    Did timeout occur?
    Did Python Traceback occur?
    Did Java Exception occur?
    Did CUDA out of memory occur?

### 3.6 What Does Grafana Handle?

Grafana is a unified viewing entry point.

It can integrate with:

    Prometheus
    Loki
    Elasticsearch
    OpenSearch

Grafana is suitable for:

    Displaying Pod metric trends
    Displaying Pod logs
    Navigating from metrics to logs
    Navigating from alerts to Dashboard
    Filtering data by namespace / pod / app

---

## FourI don't know.Pod Metric Monitoring Checklist

### 4.1 Pod Status Metrics

#### 4.1.1 Pod Phase

Metrics:

    kube_pod_status_phase

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

### 4.2 Pod Ready Metrics

Metrics:

    kube_pod_container_status_ready

Query Unready Containers:

    kube_pod_container_status_ready == 0

Count Unready Containers by Namespace:

    sum by (namespace) (
      kube_pod_container_status_ready == 0
    )

### 4.3 Pod Restart Metrics

Metrics:

    kube_pod_container_status_restarts_total

Restart increases in the last 10 minutes:

    increase(kube_pod_container_status_restarts_total[10m])

Top 10 Restarting Pods:

    topk(10,
      increase(kube_pod_container_status_restarts_total[10m])
    )

Aggregate by namespace / pod:

    sum by (namespace, pod) (
      increase(kube_pod_container_status_restarts_total[10m])
    )

### 4.4 Pod OOMKilled Metrics

Metrics:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

Query:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

Notes:

    This metric indicates the last termination reason of the container.
    After seeing this metric, you still need to combine it with kubectl describe pod and previous logs.

### 4.5 Pod CPU Usage Metrics

Metrics:

    container_cpu_usage_seconds_total

Query Pod CPU:

    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
    )

Unit:

    CPU core

Example:

    0.5 is approximately 0.5 core.

Top 10 CPU Pods:

    topk(10,
      sum by (namespace, pod) (
        rate(container_cpu_usage_seconds_total{container!="", image!=""}[5m])
      )
    )

### 4.6 Pod Memory Usage Metrics

Recommended Metrics:

    container_memory_working_set_bytes

Query Pod Memory:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{container!="", image!=""}
    )

Top 10 Memory Pods:

    topk(10,
      sum by (namespace, pod) (
        container_memory_working_set_bytes{container!="", image!=""}
      )
    )

Notes:

    container_memory_working_set_bytes is more suitable for observing the actual working set.
    container_memory_usage_bytes may include more cache effects.

### 4.7 Pod Network Metrics

Incoming Traffic:

    sum by (namespace, pod) (
      rate(container_network_receive_bytes_total{pod!=""}[5m])
    )

Outgoing Traffic:

    sum by (namespace, pod) (
      rate(container_network_transmit_bytes_total{pod!=""}[5m])
    )

### 4.8 Pod request / limit Metrics

CPU request:

    sum by (namespace, pod) (
      kube_pod_container_resource_requests{resource="cpu"}
    )

Memory request: /think

CPU limit:

    sum by (namespace, pod) (
      kube_pod_container_resource_limits{resource="cpu"}
    )

Memory limit:

    sum by (namespace, pod) (
      kube_pod_container_resource_limits{resource="memory"}
    )

Note:

    Resource metric names may vary across different kube-state-metrics versions.
    When in use, first search for metrics starting with kube_pod_container_resource in Prometheus.

---

## FiveI don't know.Pod Log Sources

### 5.1 Where should applications output logs

Kubernetes recommends containerized applications to output logs to:

    stdout
    stderr

That is, standard output and standard error.

It is not recommended to write logs only to internal container files.

Recommended approach:

    Application logs → stdout / stderr
      ↓
    Container runtime takes over
      ↓
    Node log files
      ↓
    Log collection Agent
      ↓
    Loki / Elasticsearch / OpenSearch

### 5.2 Log paths on nodes

Common paths in containerd environment:

    /var/log/containers/
    /var/log/pods/

Example:

    /var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

Explanation:

    /var/log/containers is typically the entry point for container logs.
    /var/log/pods usually stores log directories at the Pod level.
    The specific path and symlink relationships are influenced by the container runtime.

View node logs:

    ls -l /var/log/containers/

    ls -l /var/log/pods/

### 5.3 kubectl logs

View current container logs:

    kubectl logs <pod-name> -n <namespace>

View logs of the previous container instance:

    kubectl logs <pod-name> -n <namespace> --previous

View logs of a specific container:

    kubectl logs <pod-name> -n <namespace> -c <container-name>

View last 100 lines:

    kubectl logs <pod-name> -n <namespace> --tail=100

Continuous viewing:

    kubectl logs <pod-name> -n <namespace> -f

With timestamps:

    kubectl logs <pod-name> -n <namespace> --timestamps

View last 10 minutes:

    kubectl logs <pod-name> -n <namespace> --since=10m

### 5.4 Limitations of kubectl logs

kubectl logs is suitable for temporary troubleshooting of individual Pods.

But it has clear limitations:

- Not suitable for querying across multiple Pods;
- Not suitable for querying across Namespaces;
- Not suitable for long-term historical queries;
- Logs may be lost after Pod deletion;
- Local logs may be inaccessible after node failure;
- Not suitable for keyword aggregation analysis;
- Not suitable for log alerts;
- Not suitable for Dashboard.

Therefore, a centralized logging system is needed in production environments.

---

## SixI don't know.Pod Log Collection Route One: Loki + Alloy

### 6.1 When is the Loki route suitable

Suitable for:

- Kubernetes operation and maintenance troubleshooting logs;
- Integration with Grafana metric Dashboard;
- Cost-sensitive;
- Query by namespace / pod / app / container;
- No need for complex full-text indexing;
- Want to unify metrics and logs in Grafana.

Typical flow:

    Pod stdout / stderr
      ↓
    /var/log/containers
      ↓
    Grafana Alloy
      ↓
    Loki
      ↓
    Grafana Explore / Dashboard

### 6.2 Alloy's core approach for collecting Pod logs

Alloy is responsible for:

    Discovering Kubernetes Pods
    Collecting Pod logs
    Adding Kubernetes labels
    Pushing to Loki

Key labels:

    namespace
    pod
    container
    node
    app
    cluster
    environment

### 6.3 Querying Pod logs in Loki

Query by Namespace:

    {namespace="app-prod"}

Query by Pod:

    {namespace="app-prod", pod="api-xxx"}

Query by container:

    {namespace="app-prod", pod="api-xxx", container="api"}

Query ERROR:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)error|exception|panic|traceback"

Query timeout:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)timeout|timed out|deadline exceeded"

Query database connection failure:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)database connection failed|too many connections|connection refused"

### 6.4 Notes on Loki label design

Recommended labels:

    cluster
    environment
    namespace
    pod
    container
    node
    app
    workload
    team

Not recommended as Loki labels: /think

request_id
trace_id
user_id
order_id
session_id
full_url
error_message
stacktrace

Reason:

    These fields are high-cardinality fields.
    Putting them into Loki labels will cause log stream count explosion.
    Query and storage costs will increase.

These fields can remain in the log body, and can be queried using LogQL content filtering or JSON parsing.

---

## SevenI don't know.Pod Log Collection Route Two: EFK / Filebeat / Fluent Bit

### 7.1 EFK Route Suitable Scenarios

Suitable for:

- Enterprise already has Elasticsearch / OpenSearch;
- Need full-text search;
- Need field-based search;
- Need audit logs;
- Need security log analysis;
- Need complex search and aggregation;
- Need Kibana / OpenSearch Dashboards query experience.

Typical chain:

    Pod stdout / stderr
      ↓
    /var/log/containers
      ↓
    Filebeat / Fluent Bit
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 7.2 Filebeat Collection Approach

Filebeat typically runs as a DaemonSet.

It needs to mount:

    /var/log/containers
    /var/log/pods

After collection, output to:

    Elasticsearch
    Logstash
    Kafka

Typical fields:

    kubernetes.namespace
    kubernetes.pod.name
    kubernetes.container.name
    kubernetes.node.name
    message
    log.level
    @timestamp

### 7.3 Fluent Bit Collection Approach

Fluent Bit typically also runs as a DaemonSet.

Configuration usually includes:

    INPUT:
        tail /var/log/containers/*.log

    FILTER:
        kubernetes metadata

    OUTPUT:
        Elasticsearch / OpenSearch / Loki / Kafka

Fluent Bit is suitable for:

    Low resource consumption
    K8S log collection
    Multiple outputs
    Large-scale nodes

### 7.4 Kibana Query Pod Logs

Query by Namespace:

    kubernetes.namespace : "app-prod"

Query by Pod:

    kubernetes.namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"

Query by container:

    kubernetes.namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"
    and kubernetes.container.name : "api"

Query ERROR:

    kubernetes.namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"
    and log.level : "error"

Query keywords:

    kubernetes.namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"
    and message : "connection refused"

---

## EightI don't know.Pod Monitoring and Log Standard Troubleshooting Process

Recommend all Pod issues start from the following process.

### 8.1 Step One: Confirm Pod Basic Status

    kubectl get pod <pod-name> -n <namespace> -o wide

Focus on:

    STATUS
    READY
    RESTARTS
    AGE
    NODE
    IP

Judgment:

    Pending:
        First check scheduling and resources.

    CrashLoopBackOff:
        First check previous logs and Events.

    Running but NotReady:
        First check readinessProbe and application health.

    Running and Ready but business anomaly:
        First check Service, Endpoints, business metrics and application logs.

### 8.2 Step Two: View Pod Details and Events

    kubectl describe pod <pod-name> -n <namespace>

Focus on:

    Events
    Conditions
    Containers
    State
    Last State
    Reason
    Exit Code
    Restart Count
    Liveness Probe
    Readiness Probe
    Mounts
    Environment
    Node

### 8.3 Step Three: View Current Logs

    kubectl logs <pod-name> -n <namespace> --tail=100

If multiple containers:

    kubectl logs <pod-name> -n <namespace> -c <container-name> --tail=100

### 8.4 Step Four: View Previous Log

If Pod has restarted:

    kubectl logs <pod-name> -n <namespace> --previous --tail=100

Previous logs are important.

Many CrashLoopBackOff root causes are only in the previous container logs.

### 8.5 Step Five: View Resource Metrics

    kubectl top pod <pod-name> -n <namespace>

Prometheus Query:

    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}[5m])
    )

sum by (namespace, pod) (
  container_memory_working_set_bytes{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}
)

### 8.6 Step 6: Check Centralized Logs

Loki:

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)error|exception|panic|traceback|timeout|failed"

EFK:

    kubernetes.namespace : "<namespace>"
    and kubernetes.pod.name : "<pod-name>"
    and message : ("error" or "exception" or "timeout" or "failed")

### 8.7 Step 7: Confirm Service Backend

If the issue is related to access:

    kubectl describe svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

    kubectl get endpointslice -n <namespace> | grep <service-name>

Confirm:

    Whether the Pod is in Endpoints
    Whether the Service selector matches Pod label
    Whether targetPort is correct
    Whether the Pod is Ready

### 8.8 Step 8: Check Associated Workload

Deployment:

    kubectl describe deploy <deployment-name> -n <namespace>

    kubectl rollout status deploy/<deployment-name> -n <namespace>

    kubectl rollout history deploy/<deployment-name> -n <namespace>

StatefulSet:

    kubectl describe sts <statefulset-name> -n <namespace>

DaemonSet:

    kubectl describe ds <daemonset-name> -n <namespace>

---

## Nine, Scenario One: Pod Pending

### 9.1 Phenomenon

    kubectl get pod -n app-prod

Shows:

    STATUS: Pending

### 9.2 Metric Discovery

PromQL:

    kube_pod_status_phase{phase="Pending"} == 1

Alert:

    PodPendingTooLong

### 9.3 Troubleshooting Commands

    kubectl describe pod <pod-name> -n <namespace>

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 9.4 Common Events and Judgment

#### 9.4.1 Insufficient CPU

Event:

    0/3 nodes are available: insufficient cpu

Judgment:

    Cluster schedulable CPU is insufficient.
    Pod request is set too high.
    Remaining available resources on nodes are insufficient.

Resolution:

    Reduce CPU request.
    Expand nodes.
    Clean up unused Pods.
    Adjust scheduling policies.

#### 9.4.2 Insufficient Memory

Event:

    0/3 nodes are available: insufficient memory

Resolution:

    Reduce memory request.
    Expand nodes.
    Check if there are Pods with excessively high memory requests.

#### 9.4.3 Insufficient GPU

Event:

    insufficient nvidia.com/gpu

Resolution:

    Check if GPU nodes are Ready.
    Check if NVIDIA Device Plugin is functioning normally.
    Check if GPU is occupied by other Pods.
    Check if Pod resource limits are correct.
    Check if nodes have taints requiring toleration.

#### 9.4.4 PVC Not Bound

Event:

    pod has unbound immediate PersistentVolumeClaims

Resolution:

    kubectl get pvc -n <namespace>

    kubectl describe pvc <pvc-name> -n <namespace>

Check:

    Whether StorageClass exists.
    Whether PV is available.
    Whether dynamic provisioner is functioning normally.
    Whether PVC capacity and accessModes match.

#### 9.4.5 Taint Mismatch

Event:

    node(s) had untolerated taint

Resolution:

    kubectl describe node <node-name> | grep -i taint

Check whether Pod needs to add tolerations.

#### 9.4.6 nodeSelector / affinity Mismatch

Event:

    node(s) didn't match Pod's node affinity/selector

Resolution:

    kubectl get nodes --show-labels

Check:

    Whether Pod nodeSelector is written incorrectly.
    Whether node labels exist.
    Whether affinity rules are too strict.

### 9.5 Summary of Pod Pending Troubleshooting

Pod Pending is typically not an issue with application code.

Prioritize checking:

    Events
    Scheduler events
    Node resources
    PVC
    taint/toleration
    nodeSelector / affinity
    ResourceQuota

---

## Ten, Scenario Two: Pod CrashLoopBackOff

### 10.1 Phenomenon

    STATUS: CrashLoopBackOff

Or:

    RESTARTS continuously increasing

### 10.2 Metric Discovery

PromQL:

increase(kube_pod_container_status_restarts_total[10m]) > 3

Alert:

    PodRestartTooOften

### 10.3 Troubleshooting Commands

    kubectl get pod <pod-name> -n <namespace> -o wide

    kubectl describe pod <pod-name> -n <namespace>

    kubectl logs <pod-name> -n <namespace> --tail=100

    kubectl logs <pod-name> -n <namespace> --previous --tail=100

### 10.4 Loki Query

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)error|exception|panic|traceback|failed|connection refused"

### 10.5 EFK Query

    kubernetes.namespace : "<namespace>"
    and kubernetes.pod.name : "<pod-name>"
    and message : ("error" or "exception" or "panic" or "failed" or "connection refused")

### 10.6 What to Focus On

In describe:

    State
    Last State
    Reason
    Exit Code
    Started
    Finished
    Restart Count
    Events

In logs:

    Startup failure
    Missing configuration
    Database connection failure
    Permission error
    File not found
    Port conflict
    Dependency service unreachable
    Code exception
    panic
    Traceback
    Exception

### 10.7 Common Causes

#### 10.7.1 Configuration Error

Logs:

    config file not found
    invalid config
    missing environment variable

Check:

    kubectl get cm -n <namespace>

    kubectl get secret -n <namespace>

    kubectl describe pod <pod-name> -n <namespace>

#### 10.7.2 Database Connection Failure

Logs:

    database connection failed
    connection refused
    access denied
    too many connections

Check:

    Does the database Service exist?
    Is the Secret correct?
    Are the database credentials correct?
    Is NetworkPolicy blocking?
    Is DNS resolving properly?

#### 10.7.3 Probe Configuration Too Aggressive

Events:

    Liveness probe failed

Judgment:

    Application startup is slow.
    initialDelaySeconds is too short.
    timeoutSeconds is too short.
    failureThreshold is too low.
    Probe path is unstable.

Handling:

    Adjust livenessProbe.
    Distinguish between startupProbe, readinessProbe, and livenessProbe.
    Do not use livenessProbe as a substitute for business health checks.

#### 10.7.4 Permission Issues

Logs:

    permission denied

Check:

    securityContext
    runAsUser
    fsGroup
    volume permissions
    file path permissions
    Secret/ConfigMap mount permissions

#### 10.7.5 Image Entry Command Error

Logs:

    exec format error
    command not found
    no such file or directory

Check:

    image
    command
    args
    entrypoint
    Architecture compatibility

### 10.8 CrashLoopBackOff Troubleshooting Summary

CrashLoopBackOff core principles:

    Check previous logs first.
    Then check Events.
    Then check Last State and Exit Code.
    Do not restart blindly.
    The Pod is already restarting; restarting randomly is usually meaningless.

---

## ElevenI don't know.Scenario Three: Pod OOMKilled

### 11.1 Phenomenon

    kubectl describe pod

See:

    Reason: OOMKilled

Or Prometheus alert:

    PodOOMKilled

### 11.2 Metric Discovery

PromQL:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

Check memory usage:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}
    )

Check memory limit:

    kube_pod_container_resource_limits{namespace="<namespace>", pod="<pod-name>", resource="memory"}

### 11.3 Troubleshooting Commands

    kubectl describe pod <pod-name> -n <namespace>

    kubectl logs <pod-name> -n <namespace> --previous --tail=100

    kubectl top pod <pod-name> -n <namespace>

### 11.4 Loki Query

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)oom|memory|out of memory|killed"

### 11.5 Common Causes

- memory limit set too low;
- application memory leak;
- JVM heap settings unreasonable;
- Python reading large files;
- batch processing task data volume too large;
- request body too large;
- cache grows infinitely;
- high concurrency;
- AI model loading consumes excessive memory.

### 11.6 OOMKilled vs CUDA OOM Difference

#### OOMKilled

Meaning:

    Container memory exceeds cgroup limit, killed by kernel.

Check:

    Pod describe
    Last State
    container_memory_working_set_bytes
    memory limit
    previous logs

#### CUDA out of memory

Meaning:

    GPU memory insufficient, usually reported as an error by AI framework in application logs.

Check:

    Application logs
    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_FB_FREE
    batch size
    model size
    worker count
    concurrency level

Note:

    OOMKilled is a container memory issue.
    CUDA OOM is a GPU memory issue.
    They are not the same problem.

### 11.7 Handling Recommendations

Temporary handling:

    Reduce concurrency.
    Reduce batch size.
    Increase memory limit cautiously.
    Roll back abnormal version.
    Clean abnormal cache.

Long-term handling:

    Fix memory leak.
    Optimize data loading.
    Optimize cache strategy.
    Establish memory baseline.
    Set reasonable request/limit.
    Add memory trend alert.

---

## TwelveI don't know.Scenario Four: Pod Running but Business Abnormal

### 12.1 Phenomena

    Pod Running
    Pod Ready
    Service has Endpoints

But user feedback:

    Access failure
    Interface 500
    Interface slow
    Occasional timeout
    Login failure
    Database operation failure

### 12.2 Key Understanding

Pod Running does not equal business normal.

Pod Ready does not equal business fully normal.

Service having Endpoints does not equal interface not reporting errors.

### 12.3 Troubleshooting Order

    1. Check Service selector
    2. Check Endpoints
    3. Check Pod Ready
    4. Check container port listening
    5. Cluster internal curl Service
    6. Check business metrics
    7. Check Pod logs
    8. Check dependent services
    9. Check Ingress/Gateway
    10. Check NetworkPolicy
    11. Check DNS

### 12.4 Commands

Service:

    kubectl describe svc <service-name> -n <namespace>

Endpoints:

    kubectl get endpoints <service-name> -n <namespace>

EndpointSlice:

    kubectl get endpointslice -n <namespace> | grep <service-name>

Pod:

    kubectl get pod -n <namespace> -o wide

Logs:

    kubectl logs <pod-name> -n <namespace> --tail=100

Port listening:

    kubectl exec -it <pod-name> -n <namespace> -- ss -lntp

Temporary testing:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside container:

    curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>

### 12.5 Business Metrics Query

Error rate:

    sum by (service) (
      rate(http_requests_total{status=~"5.."}[5m])
    )
    /
    sum by (service) (
      rate(http_requests_total[5m])
    )
    * 100

P95 latency:

    histogram_quantile(
      0.95,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket[5m])
      )
    )

### 12.6 Log Query

Loki:

    {namespace="<namespace>", app="<app>"} |~ "(?i)error|timeout|connection refused|database|redis|exception"

EFK:

    kubernetes.namespace : "<namespace>"
    and message : ("error" or "timeout" or "connection refused" or "database")

### 12.7 Common Causes

- targetPort error;
- application only listens on 127.0.0.1;
- readinessProbe too lenient;
- business health interface does not check real dependencies;
- database connection failure;
- Redis connection failure;
- downstream interface timeout;
- DNS resolution anomaly;
- NetworkPolicy blocking;
- Ingress forwarding error;
- configuration change error;
- new version bug.

---

## ThirteenI don't know.Scenario Five: Pod Log Collection Abnormal

### 13.1 Phenomena

    kubectl logs has logs
    But Loki/Kibana cannot find logs

Or:

    No logs on a certain node's Pod
    No logs in a certain namespace
    No logs for a certain application
    Logs exist but missing namespace/pod fields

### 13.2 Troubleshooting Order /think

1. Does the application output stdout / stderr  
2. Can kubectl logs see the logs  
3. Is the Log Agent running  
4. Is the Agent deployed as DaemonSet to cover all nodes  
5. Is the Agent running on the node where the Pod is located  
6. Is /var/log/containers mounted  
7. Is /var/log/pods mounted  
8. Does the Agent have read permissions  
9. Is Kubernetes metadata normal  
10. Is namespace / pod / container filtered  
11. Is Loki / Elasticsearch writable  
12. Is the query time range correct  
13. Are the query labels / fields written correctly  

### 13.3 Check Agent  

    kubectl get pods -n logging -o wide  

    kubectl get ds -n logging  

    kubectl logs <agent-pod> -n logging  

### 13.4 Enter Agent to Check Mounts  

    kubectl exec -it <agent-pod> -n logging -- sh  

Inside the container:  

    ls -l /var/log/containers/  

    ls -l /var/log/pods/  

### 13.5 Diagnosis Methods  

If kubectl logs also does not show anything:  
    The application may not output stdout / stderr.  
    The application may only write to internal container files.  
    The log level may not be output.  
    The application may not have run to the log logic.  

If kubectl logs shows something, but Loki / EFK does not:  
    Prioritize checking the Log Agent.  
    Check mounts.  
    Check collection configuration.  
    Check output endpoints.  
    Check filtering rules.  
    Check time range and query conditions.  

If only a specific node has no logs:  
    Prioritize checking the Agent Pod on that node.  

If only a specific namespace has no logs:  
    Prioritize checking if the collection rule filters the namespace.  

If logs exist but no pod field:  
    Prioritize checking Kubernetes metadata configuration and RBAC permissions.  

---

## Fourteen, Pod Dashboard Design  

### 14.1 Dashboard Variables  

Recommended variables:  
    cluster  
    environment  
    namespace  
    workload  
    pod  
    container  
    node  
    app  

### 14.2 Top Overview  

Recommended panels:  
    Total Pods  
    Running Pod Count  
    Pending Pod Count  
    Failed Pod Count  
    NotReady Pod Count  
    Restarted Pod Count  
    OOMKilled Pod Count  
    CPU Top Pod  
    Memory Top Pod  

### 14.3 Pod Detail Table  

Fields:  
    Namespace  
    Pod  
    Node  
    Phase  
    Ready  
    Restart Count  
    Last Terminated Reason  
    CPU Usage  
    Memory Usage  
    Age  

### 14.4 Pod Resource Trends  

Panels:  
    Pod CPU Usage  
    Pod Memory Working Set  
    Pod Network Receive  
    Pod Network Transmit  
    Container Restart Trend  

### 14.5 Pod Log Panel  

Loki Query:  
    {namespace="$namespace", pod="$pod"}  

Error Logs:  
    {namespace="$namespace", pod="$pod"} |~ "(?i)error|exception|panic|traceback|timeout"  

If using EFK:  
    Add a Kibana Discover jump link.  

### 14.6 Pod Events Panel  

Events can be displayed through an event collection component or a log platform.  

If no event collection is available, retain the following commands in the Runbook:  
    kubectl describe pod $pod -n $namespace  

    kubectl get events -n $namespace --sort-by=.lastTimestamp  

### 14.7 Dashboard Goals  

The goal of the Pod Dashboard is not to pile up charts, but to reduce troubleshooting paths.  

Ideal path:  
    From alert to Dashboard  
      ↓  
    View Pod status and resource trends  
      ↓  
    See recent restarts and OOMKilled  
      ↓  
    One-click jump to Pod logs  
      ↓  
    One-click view Service backend  
      ↓  
    One-click enter Runbook  

---

## Fifteen, Pod Alert Rule Design  

### 15.1 Pod Pending  

    - alert: PodPendingTooLong  
      expr: kube_pod_status_phase{phase="Pending"} == 1  
      for: 10m  
      labels:  
        severity: warning  
        team: sre  
      annotations:  
        summary: "Pod Pending Time Too Long"  
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} Pending exceeds 10 minutes."  
        runbook_url: "https://wiki.example.com/runbook/pod-pending""  

### 15.2 Pod Restarted Too Many Times /think

- alert: PodRestartTooOften
  expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
  for: 5m
  labels:
    severity: warning
    team: sre
  annotations:
    summary: "Pod has restarted too often"
    description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has restarted more than 3 times in the last 10 minutes."
    runbook_url: "https://wiki.example.com/runbook/pod-restart"

### 15.3 Pod OOMKilled

- alert: PodOOMKilled
  expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
  for: 1m
  labels:
    severity: warning
    team: sre
  annotations:
    summary: "Pod was OOMKilled"
    description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} was last terminated due to OOMKilled."
    runbook_url: "https://wiki.example.com/runbook/pod-oomkilled"

### 15.4 Pod NotReady

- alert: PodNotReadyTooLong
  expr: kube_pod_container_status_ready == 0
  for: 10m
  labels:
    severity: warning
    team: sre
  annotations:
    summary: "Pod container has been unready for too long"
    description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} has been unready for more than 10 minutes."
    runbook_url: "https://wiki.example.com/runbook/pod-not-ready"

### 15.5 ERROR logs too many

Loki Rule example:

  groups:
    - name: pod-log-alerts
      rules:
        - alert: PodErrorLogsTooMany
          expr: |
            sum by (namespace, pod) (
              count_over_time({namespace=~"app-prod|ai-prod"} |~ "(?i)error|exception|panic|traceback" [5m])
            ) > 20
          for: 5m
          labels:
            severity: warning
            team: app
            source: loki
          annotations:
            summary: "Pod has too many error logs"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has more than 20 error logs in the last 5 minutes."
            runbook_url: "https://wiki.example.com/runbook/pod-error-logs"

### 15.6 timeout logs too many

  groups:
    - name: pod-timeout-log-alerts
      rules:
        - alert: PodTimeoutLogsTooMany
          expr: |
            sum by (namespace, pod) (
              count_over_time({namespace=~"app-prod|ai-prod"} |~ "(?i)timeout|timed out|deadline exceeded" [5m])
            ) > 10
          for: 5m
          labels:
            severity: warning
            team: app
            source: loki
          annotations:
            summary: "Pod has too many timeout logs"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has exceeded the threshold for timeout-related logs in the last 5 minutes."
            runbook_url: "https://wiki.example.com/runbook/pod-timeout"

---

## SixteenI don't know.Pod Alert Grouping and Silencing

### 16.1 Why Grouping is Necessary

A single incident may trigger:

  PodRestartTooOften
  PodErrorLogsTooMany
  PodTimeoutLogsTooMany
  ServiceHigh5xxErrorRate

Without grouping, alerts will flood the interface.

### 16.2 AlertManager group_by Recommendations

Pod dimension:

  group_by:
    - cluster
    - namespace
    - pod

Application dimension:

  group_by:
    - cluster
    - namespace
    - app

Service dimension:

  group_by:
    - cluster
    - namespace
    - service

### 16.3 Silencing Recommendations

If core business alerts have already triggered:

  ServiceHigh5xxErrorRate

You can silence low-level log noise alerts under the same app:

  PodErrorLogsTooMany

But do not silence critical root cause alerts, such as:

PodOOMKilled
PodPendingTooLong
GPUXIDError
CUDALogOOMDetected

### 16.4 Alert Label Recommendations

Unified Labels:

    cluster
    environment
    namespace
    app
    pod
    container
    node
    team
    severity
    source

source can be:

    prometheus
    loki
    elasticsearch
    opensearch

---

## SeventeenI don't know.Complete Case One: CrashLoopBackOff

### 17.1 Alert

    PodRestartTooOften

### 17.2 Phenomenon

    Pod app-prod/api-xxx has restarted 5 times in the last 10 minutes.

### 17.3 Automatic Troubleshooting

Check Pod:

    kubectl get pod api-xxx -n app-prod -o wide

Check Details:

    kubectl describe pod api-xxx -n app-prod

Check Current Logs:

    kubectl logs api-xxx -n app-prod --tail=100

Check Previous Logs:

    kubectl logs api-xxx -n app-prod --previous --tail=100

Loki Query:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)error|exception|panic|traceback|connection refused"

### 17.4 Diagnostic Judgment

If previous logs show:

    database connection failed

And Events do not show OOMKilled, then preliminary judgment:

    Application failed to connect to database on startup, causing process exit.

### 17.5 Handling Recommendations

    1. Check database Service.
    2. Check Secret for account password.
    3. Check NetworkPolicy.
    4. Check database connection count.
    5. Check recent configuration changes.
    6. Rollback version if necessary.

---

## EighteenI don't know.Complete Case Two: OOMKilled

### 18.1 Alert

    PodOOMKilled

### 18.2 Phenomenon

    Pod app-prod/worker-xxx was OOMKilled.

### 18.3 Automatic Troubleshooting

Pod Details:

    kubectl describe pod worker-xxx -n app-prod

Previous Logs:

    kubectl logs worker-xxx -n app-prod --previous --tail=100

Memory Metrics:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="app-prod", pod="worker-xxx", container!="", image!=""}
    )

Memory Limit:

    kube_pod_container_resource_limits{namespace="app-prod", pod="worker-xxx", resource="memory"}

### 18.4 Diagnostic Judgment

If found:

    Memory usage consistently near limit
    Previous logs show processing large files
    OOMKilled occurred during peak task time

Preliminary judgment:

    Batch processing task memory usage exceeded limit.

### 18.5 Handling Recommendations

    1. Limit data volume per task.
    2. Process large files in batches.
    3. Check for memory leaks.
    4. Reasonably adjust memory request/limit.
    5. Add task queue throttling.
    6. Add memory trend Dashboard.

---

## NineteenI don't know.Complete Case Three: Pod Running but Service Access Anomalies

### 19.1 Alert

    ServiceHigh5xxErrorRate

Along with log alerts:

    PodErrorLogsTooMany

### 19.2 Phenomenon

    Pod Running
    Pod Ready
    Service has Endpoints
    User access to interface returns 500

### 19.3 Automatic Troubleshooting

Service:

    kubectl describe svc order-api -n app-prod

Endpoints:

    kubectl get endpoints order-api -n app-prod

Pod:

    kubectl get pod -n app-prod -o wide

Logs:

    kubectl logs order-api-xxx -n app-prod --tail=100

Loki:

    {namespace="app-prod", app="order-api"} |~ "(?i)error|database|timeout|connection refused"

Prometheus:

    sum by (service) (
      rate(http_requests_total{service="order-api", status=~"5.."}[5m])
    )

### 19.4 Diagnostic Judgment

If found:

    Endpoints are normal
    Pod has no restarts
    Logs showMass database connection failed
    5xx error rate increases

Preliminary judgment:

    Service routing is normal, application internal dependency on database is abnormal.

### 19.5 Handling Recommendations

    1. Check database status.
    2. Check database connection pool.
    3. Check if Secret has changed.
    4. Check application configuration.
    5. Check recent deployment.
    6. Rollback if necessary.

---

## TwentyI don't know.Complete Case Four: Pod Log Loss

### 20.1 Phenomenon

    kubectl logs can see logs
    But Grafana Loki cannot find logs

### 20.2 Troubleshooting

Confirm Pod's node:

    kubectl get pod api-xxx -n app-prod -o wide

Check logging namespace:

    kubectl get pods -n logging -o wide

Confirm if the Agent is running on the node:

    kubectl get pods -n logging -o wide | grep <node-name>

Check Agent logs:

    kubectl logs <agent-pod> -n logging

Enter the Agent:

    kubectl exec -it <agent-pod> -n logging -- sh

Check mounts:

    ls -l /var/log/containers/

### 20.3 Possible Causes

- The Agent is not running on this node;
- DaemonSet is restricted by nodeSelector;
- Tolerations do not match;
- /var/log/containers is not mounted;
- Agent RBAC permissions are insufficient;
- Collection rules filter the namespace;
- Loki address configuration is incorrect;
- Loki write failure;
- Query labels are incorrect;
- Query time range is wrong.

### 20.4 Recommended Actions

    1. Fix Agent DaemonSet scheduling.
    2. Fix hostPath mounting.
    3. Fix Loki address.
    4. Fix namespace filtering rules.
    5. Confirm log labels.
    6. Create PodLogsMissing Runbook.

---

## Twenty-one, Pod Observability Runbook Template

Each Pod-related alert should include a Runbook.

### 21.1 Runbook Basic Structure

    Alert Name:
    Alert Meaning:
    Impact Scope:
    Possible Causes:
    First Check:
    Troubleshooting Commands:
    PromQL:
    LogQL:
    KQL:
    Temporary Fix:
    Root Cause Resolution:
    Rollback Method:
    Escalation Contacts:

### 21.2 PodRestartTooOften Runbook Example

    Alert Name:
        PodRestartTooOften

    Meaning:
        Pod restarts frequently within a short period.

    Impact:
        May cause service instability, request failures, and Service backend jitter.

    First Priority:
        Review previous logs.

    Commands:
        kubectl describe pod <pod> -n <namespace>
        kubectl logs <pod> -n <namespace> --previous --tail=100
        kubectl get events -n <namespace> --sort-by=.lastTimestamp

    PromQL:
        increase(kube_pod_container_status_restarts_total{namespace="<namespace>", pod="<pod>"}[10m])

    Loki:
        {namespace="<namespace>", pod="<pod>"} |~ "(?i)error|exception|panic|traceback"

    Common Causes:
        Configuration errors, dependencies unreachable, OOMKilled, probe failures, startup command errors.

    Resolution:
        Determine root cause based on logs and Events; do not blindly restart.

---

## Twenty-two, Production Implementation Recommendations

### 22.1 Metrics and Logs Must Be Built Simultaneously

Only building Prometheus:

    Can detect resource and status anomalies.
    Lacks application error context.

Only building a log platform:

    Can see error content.
    Lacks trends, alerts, capacity, and status metrics.

Recommendation:

    Prometheus + Grafana + AlertManager
      +
    Loki or EFK
      +
    Runbook

### 22.2 Pods Must Have Standard Labels

Recommend all business Pods have:

    app
    component
    version
    team
    environment

Example:

    labels:
      app: order-api
      component: backend
      version: v1.2.3
      team: app
      environment: prod

Benefits:

    Metrics can be aggregated.
    Logs can be filtered.
    Alerts can be routed.
    Dashboards can be reused.
    Owner can be identified.

### 22.3 Application Logs Should Be JSON-formatted

Recommended format:

    {
      "timestamp": "2026-04-30T12:00:00+08:00",
      "level": "error",
      "service": "order-api",
      "trace_id": "abc123",
      "msg": "database connection failed",
      "duration_ms": 1200
    }

Benefits:

    Loki can parse JSON.
    EFK can query fields.
    Alert rules are more accurate.
    Troubleshooting information is more complete.

### 22.4 Do Not Output Sensitive Information

Prohibited outputs:

    Passwords
    Tokens
    Access keys
    Secret keys
    Authorization headers
    Cookies
    Private keys
    Plain-text phone numbers
    IDs
    Bank cards
    Database connection string passwords

### 22.5 Pod Logs Must Be Centralized

Production should not rely only on kubectl logs.

Reasons:

    Pods may be recreated.
    Nodes may fail.
    Local logs will rotate.
    Querying multiple replicas is inconvenient.
    Long-term tracing is not possible.
    Log alerts are not supported.

---

## Twenty-three, Acceptance Checklist

### 23.1 Pod Metrics Acceptance /think

[ ] kube_pod_status_phase is available for inspection
    [ ] kube_pod_container_status_ready is available for inspection
    [ ] kube_pod_container_status_restarts_total is available for inspection
    [ ] kube_pod_container_status_last_terminated_reason is available for inspection
    [ ] container_cpu_usage_seconds_total is available for inspection
    [ ] container_memory_working_set_bytes is available for inspection
    [ ] Pod CPU Dashboard has data
    [ ] Pod Memory Dashboard has data
    [ ] Pod Restart Dashboard has data
    [ ] Pod OOMKilled alert can be triggered

### 23.2 Pod Logs Verification

    [ ] Application output stdout / stderr
    [ ] kubectl logs can view logs
    [ ] kubectl logs --previous can view previous round logs
    [ ] /var/log/containers path exists
    [ ] Log Agent runs as DaemonSet
    [ ] Log Agent covers all nodes
    [ ] Log Agent mounts /var/log/containers
    [ ] Loki / EFK can query Pod logs
    [ ] Logs contain namespace field
    [ ] Logs contain pod field
    [ ] Logs contain container field
    [ ] Logs contain node field

### 23.3 Pod Interoperability Troubleshooting Verification

    [ ] Pod alerts can jump to Grafana Dashboard
    [ ] Dashboard can view Pod CPU / Memory
    [ ] Dashboard can view Pod Restart
    [ ] Dashboard can jump to Pod logs
    [ ] Runbook contains kubectl describe
    [ ] Runbook contains kubectl logs --previous
    [ ] Runbook contains Loki / EFK query statements
    [ ] Runbook contains Service / Endpoints checks
    [ ] Runbook contains troubleshooting suggestions

### 23.4 Alert Verification

    [ ] PodPendingTooLong is configured
    [ ] PodRestartTooOften is configured
    [ ] PodOOMKilled is configured
    [ ] PodNotReadyTooLong is configured
    [ ] PodErrorLogsTooMany is configured
    [ ] PodTimeoutLogsTooMany is configured
    [ ] Alerts contain namespace
    [ ] Alerts contain pod
    [ ] Alerts contain severity
    [ ] Alerts contain team
    [ ] Alerts contain runbook_url
    [ ] Alerts contain dashboard_url

---

## Twenty-four, Common Misconceptions

### 24.1 Mistake 1: Pod Running means the business is normal

Error.

Pod Running only indicates the container process exists.

Whether the business is normal also depends on:

    Ready
    Service Endpoints
    Application logs
    Business metrics
    Dependent services
    5xx error rate
    Latency

### 24.2 Mistake 2: kubectl logs is sufficient

Error.

kubectl logs is suitable for temporary troubleshooting of single Pods.

Production requires centralized logging systems.

### 24.3 Mistake 3: Prometheus collects logs

Error.

Prometheus collects metrics, not logs.

Logs require Loki / ELK / OpenSearch.

### 24.4 Mistake 4: OOMKilled and CUDA OOM are the same

Error.

OOMKilled is container memory exceeded.

CUDA OOM is GPU memory insufficient.

### 24.5 Mistake 5: Pod restart means directly restarting Deployment

Error.

The Pod is already restarting.

Must first check previous logs, Events, Last State, Exit Code.

### 24.6 Mistake 6: Service having Endpoints means there's no problem

Error.

Endpoints only indicates backend existence.

Backend applications may still have 500, timeout, dependency issues, or configuration errors.

---

## Twenty-five, Summary

Pod monitoring and log collection are one of the most core capabilities in Kubernetes observability.

Pod metrics answer:

    What is the Pod's status?
    Is it Pending?
    Is it Ready?
    Has it restarted?
    Was it OOMKilled?
    Are CPU / memory / network abnormal?

Pod Events answer:

    What did Kubernetes do to the Pod?
    Was scheduling failed?
    Was image pull failed?
    Was probe failed?
    Was mount failed?
    Was resource insufficient?

Pod logs answer:

    Why did the application fail?
    What happened during startup?
    Why can't dependencies connect?
    Why return 500?
    What happened before OOM?
    What error occurred before CrashLoopBackOff?

Service / Endpoints answer:

    Is the Pod actually receiving service traffic?
    Is the Service selector correct?
    Is targetPort correct?
    Is the backend Ready?

Complete troubleshooting path:

    Alert detects anomaly
      ↓
    Grafana view Pod metrics
      ↓
    kubectl describe view Events
      ↓
    kubectl logs --previous view application failure cause
      ↓
    Loki / EFK query cross-replica logs
      ↓
    Service / Endpoints verify backend
      ↓
    Prometheus view business metrics
      ↓
    Runbook guide handling
      ↓
    Post-mortem optimize alerts and logs

Production-grade goal is not just:

    kubectl logs

But to form:

    Pod metrics + Pod Events + Pod logs + Service backend + alerts + Dashboard + Runbook

This complete closed-loop system.

---

## Twenty-six, Reference Documents

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes Resource Metrics Pipeline:
  https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/

- Kubernetes Metrics Reference:
  https://kubernetes.io/docs/reference/instrumentation/metrics/

- Kubernetes Debug Pods:
  https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/

- Kubernetes kubectl logs:
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/

- kube-state-metrics:
  https://github.com/kubernetes/kube-state-metrics

- Metrics Server:
  https://github.com/kubernetes-sigs/metrics-server

- Prometheus Querying Basics:
  https://prometheus.io/docs/prometheus/latest/querying/basics/

- Prometheus Alerting Rules:
  https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Grafana Alloy Collect Kubernetes Logs:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- Grafana Alloy loki.source.kubernetes:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- Filebeat on Kubernetes:
  https://www.elastic.co/docs/reference/beats/filebeat/running-on-kubernetes

- Fluent Bit Documentation:
  https://docs.fluentbit.io/

- Elasticsearch Documentation:
  https://www.elastic.co/docs

- OpenSearch Documentation:
  https://docs.opensearch.org/