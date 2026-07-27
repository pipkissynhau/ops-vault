# 19-Pod Monitoring and Log Collection Special Practice

## Document Description

This document specifically organizes the monitoring, log collection, event troubleshooting, Dashboard, alarm rules, and production failure localization methods at the Kubernetes Pod level.

In Article 14, the overall K8S monitoring approach was discussed from the perspectives of Node, Pod, and Service objects.

Article 15 covered Loki for log collection and querying.

Article 16 focused on ELK/EFK for log collection and retrieval.

Article 17 addressed log alerts and automated responses.

Article 18 presented a comprehensive case study combining monitoring, GPU, and logs.

As Article 19, this document does not repeat the broad content of previous chapters but instead focuses on one core issue:

    When a Pod encounters problems, how can one simultaneously view metrics, logs, Events, the Service backend, alerts, and the Dashboard?

This document addresses the following key questions:

- What metrics should be monitored for Pods?
- Where do Pod logs come from?
- What is the difference between `kubectl logs` and centralized log platforms?
- What is the relationship between `/var/log/containers` and `/var/log/pods`?
- What are the responsibilities of Prometheus, Metrics Server, Loki, and EFK?
- How to troubleshoot issues with Pods in the Pending, CrashLoopBackOff, OOMKilled, or Running states but with abnormal business performance?
- How to query a Pod's logs using Loki/Grafana?
- How to query a Pod's logs using EFK/Kibana?
- How to design a Pod Dashboard?
- How to design Pod-level alerts?
- How to link Pod metrics, logs, Events, and Service endpoints into a troubleshooting chain?
- How to conduct production-grade Pod observability audits?

This document is suitable for study after completing the following content:

- 14-K8S-Monitoring Practice-Node-Pod-Service Metric Collection and Troubleshooting
- 15-Loki-Log Collection and Query Practice
- 16-ELK-EFK-Log Collection and Retrieval Practice
- 17-Log Alerts and Automated Response Practice
- 18-Monitoring-GPU-Log Combination Case-Study-K8S-Pod Exception Detection and Reporting Generation

---

## Tags

#Kubernetes #PodMonitoring #PodLogs #Prometheus #Grafana #Loki #EFK #Filebeat #FluentBit #GrafanaAlloy #MetricsServer #kube-state-metrics #cAdvisor #SRE #FaultTroubleshooting #Observability

---

## Recommended Reading Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Basics/19-Pod Monitoring and Log Collection Special Practice.md

---

## I. Why a Separate Pod Monitoring and Log Collection Special Practice is Needed

In Kubernetes, a Pod is the smallest unit for scheduling business operations.

Most production-level issues ultimately need to be confirmed at the Pod level:

    Was the Pod successfully scheduled?
    Is the Pod running?
    Is the Pod ready?
    Does the Pod restart frequently?
    Has the Pod been killed due to out-of-memory errors?
- Is the Pod connected to a Service?
- Does the Pod have endpoints?
- What errors are reported in the Pod's application logs?
- Is there anything abnormal with the node where the Pod is located?
- Are the CPU/memory resources used by the Pod abnormal?
- Has the Pod been terminated by monitoring probes?
- Could the Pod fail to start due to configuration, permissions, images, or dependencies?

Looking at only one signal is insufficient.

For example:

    The Pod is running:
        This only indicates that the container process exists but does not necessarily mean the business is functioning normally.

    The Pod is ready:
        This only means that the readinessProbe has passed; it does not guarantee that all dependencies are working correctly.

    The Service has endpoints:
        This only indicates that the Service backend exists but does not mean that the API is error-free.

    `kubectl logs` shows errors:
        This only indicates that an application error occurred but may not reveal whether it affects traffic or availability.

    Normal CPU/memory usage:
        This only means that there are no obvious resource issues but does not guarantee that the business logic is functioning correctly.

Therefore, Pod-level troubleshooting requires combining the following types of information:

    Pod status
    Pod metrics
    Pod Events
    Pod logs
    The node where the Pod resides
    The workload to which the Pod belongs
    The Service to which the Pod belongs
    Whether the Pod has endpoints
    Application business metrics
    Alerts and Dashboards

The goal of this document is to establish a fixed troubleshooting framework:

    Use metrics to observe phenomena.
    Use Events to understand Kubernetes behavior.
    Use logs to determine the cause of application issues.
    Use Service/endpoints to check traffic and backend performance.
    Use the node to analyze underlying resources.
    Use the Dashboard to```bash
kubectl rollout status deploy/<deployment-name> -n <namespace>

kubectl rollout history deploy/<deployment-name> -n <namespace>
```

### 2.7 Layer Seven: Business Metrics

The normal operation of resources at the Pod level does not necessarily mean that the business is functioning properly.

It is also necessary to check the application's own metrics:

    QPS
    5xx error rate
    P95 / P99 latency
    Request success rate
    Queue length
    Current concurrency
    Dependency call error rate
    Database connection pool
    Redis connection pool
    Model inference time

Typical metrics include:

    http_requests_total
    http_request_duration_seconds_bucket
    httprequests_errors_total
    app_queue_length
    app_inflight_requests
```

---

## III. Relationship Between Pod Monitoring and Log Collection Components

### 3.1 What Does Prometheus Do?

Prometheus is responsible for collecting metrics.

It does not collect Pod logs.

Prometheus can collect:

    Pod status metrics exposed by kube-state-metrics
    Container resource metrics exposed by kubelet / cAdvisor
    Business metrics exposed by the application’s /metrics endpoint
    Node metrics exposed by node-exporter
    GPU metrics exposed by dcgm-exporter

Prometheus is useful for answering questions such as:

    Is the Pod in a Pending state?
    Has the Pod been restarted?
    Is the Pod using too much CPU?
    Is the Pod using too much memory?
    Was the Pod OOMKilled?
    Is the service experiencing a high error rate?
    Have there been increases in latency?

### 3.2 What Does Metrics Server Do?

Metrics Server provides real-time resource metrics to the Kubernetes Metrics API.

Common commands include:

    kubectl top node
    kubectl top pod -A

Metrics Server primarily serves the following purposes:

    kubectl top
    HPA (Horizontal Pod Autoscaler) and VPA (Vertical Pod Autoscaler) functionality

Note:

    Metrics Server is not suitable for long-term metric storage.
    It is not designed for complex alerting systems.
    It does not replace Prometheus.

### 3.3 What Does kube-state-metrics Do?

kube-state-metrics converts the status of Kubernetes objects into Prometheus metrics.

It is useful for answering questions such as:

    Is the Pod in a Pending state?
    Is the Pod Running?
    Is the Pod Ready?
    How many times has the Pod been restarted?
    What was the reason for the last termination of the Pod?
    Does the Deployment have enough available replicas?
    Is the PVC in a Pending state?

It does not collect:

    CPU usage
    Memory usage
    Log content

### 3.4 What Do kubelet / cAdvisor Do?

kubelet / cadvisor are responsible for exposing container resource usage metrics.

They are useful for answering questions such as:

    How much CPU is being used by the Pod?
    How much memory is being used by the Pod?
    What is the network traffic of the Pod?
    How much space is being used by the container’s file system?

### 3.5 What Do Loki / ELK Do?

Loki / ELK are responsible for log collection, storage, and querying.

They are useful for investigating issues such as:

    Why did the application fail to start?
    What logs were produced before a Pod entered a CrashLoopBackOff state?
    Were there any large requests before an OOM occurred?
    Were there any database connection failures?
    Were there any timeouts?
    Were there any Python Tracebacks?
    Were there any Java Exceptions?
    Were there any CUDA out of memory errors?

### 3.6 What Does Grafana Do?

Grafana serves as a unified viewing interface.

It can integrate with:

    Prometheus
    Loki
    Elasticsearch
    OpenSearch

Grafana is useful for:

    Displaying Pod metric trends
    Viewing Pod logs
    Linking metrics to logs
    Linking alerts to dashboards
    Filtering data by namespace, pod, or application
```

---

## IV. Pod Metric Monitoring Checklist

### 4.1 Pod Status Metrics

#### 4.1.1 Pod Phase

Metric:

    kube_pod_status_phase

To check if a Pod is in a Pending state:

    `kube_pod_status_phase{phase="Pending"} == 1`

To check if a Pod is Running:

    `kube_pod_status_phase{phase="Running"} == 1`

To check if a Pod has failed:

    `kube_pod_status_phase{phase="Failed"} == 1`

To count Pending Pods by namespace:

    `sum by (namespace) (
      kube_pod_status_phase{phase="Pending"} == 1
    )`

### 4.2 Pod Ready Metrics

Metric:

    kube_pod_container_status_ready

To check if a container is not ready:

    `kube_pod_container_status_ready == 0`

To count containers that are not ready by```markdown
kube_pod_container_resource_requests{resource="cpu"}
)

Memory request:

    sum by (namespace, pod) (
      kube_pod_container_resourcerequests{resource="memory"}
    )

CPU limit:

    sum by (namespace, pod) (
      kube_pod_container_resource_limits{resource="cpu"}
    )

Memory limit:

    sum by (namespace, pod) (
      kube_pod_container_resource_limits{resource="memory"}
    )

Note:

    Resource metric names may vary across different versions of kube-state-metrics.
    When using it in practice, first search for metrics starting with "kube_pod_container_resource" in Prometheus.

---

## Section 5: Pod Log Sources

### 5.1 Where Applications Should Output Logs

Kubernetes recommends that container applications output logs to:

    stdout
    stderr

That is, standard output and standard error.

It is not recommended that containerized applications only write logs to internal container files.

Recommended approach:

    Application logs → stdout / stderr
      ↓
    Taken over by the container runtime
      ↓
    Node log files
      ↓
    Log collection agent
      ↓
    Loki / Elasticsearch / OpenSearch

### 5.2 Log Paths on Nodes

Common paths in a containerd environment:

    /var/log/containers/
    /var/log/pods/

Example:

    /var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

Explanation:

    /var/log/containers is usually the entry point for container logs.
    /var/log/pods typically stores log directories at the Pod level.
    The specific paths and symbolic link relationships can be affected by the container runtime.

To view node logs:

    ls -l /var/log/containers/

    ls -l /var/log/pods/

### 5.3 kubectl logs

To view current container logs:

    kubectl logs <pod-name> -n <namespace>

To view previous container instance logs:

    kubectl logs <pod-name> -n <namespace> --previous

To view specific container logs:

    kubectl logs <pod-name> -n <namespace> -c <container-name>

To view the last 100 lines:

    kubectl logs <pod-name> -n <namespace> --tail=100

For continuous monitoring:

    kubectl logs <pod-name> -n <namespace> -f

With timestamps:

    kubectl logs <pod-name> -n <namespace> --timestamps

To view logs from the last 10 minutes:

    kubectl logs <pod-name> -n <namespace> --since=10m

### 5.4 Limitations of kubectl logs

kubectl logs is suitable for temporarily troubleshooting individual Pods.

However, it has significant limitations:

- It is not suitable for querying across multiple Pods;
- It is not suitable for cross-Namespace queries;
- It is not ideal for historical long-term searches;
- Logs may be lost after the Pod is deleted;
- Local logs may become inaccessible in case of node failures;
- It is not suitable for keyword-based aggregation and analysis;
- It is not appropriate for log alerts or dashboards.

Therefore, a centralized logging system is necessary in production environments.

---

## Section 6: Pod Log Collection Route 1: Loki + Alloy

### 6.1 Scenarios Suitable for the Loki Route

Suitable for:

- Kubernetes operations and troubleshooting logs;
- Integration with Grafana metrics dashboards;
- Cost-sensitive scenarios;
- Queries by namespace/pod/app/container;
- No need for complex full-text indexing;
- Want to consolidate metrics and logs in Grafana.

Typical workflow:

    Pod stdout / stderr
      ↓
    /var/log/containers
      ↓
    Grafana Alloy
      ↓
    Loki
      ↓
    Grafana Explore/Dashboards

### 6.2 Core Concept of Alloy for Collecting Pod Logs

Alloy is responsible for:

- Discovering Kubernetes Pods;
- Collecting Pod logs;
- Adding Kubernetes metadata labels;
- Pushing the data to Loki.

Key labels used:

    namespace
    pod
    container
    node
    app
    cluster
    environment

### 6.3 Querying Pod Logs with Loki

Query by Namespace:

    {namespace="app-prod"}

Query by Pod:

    {namespace="app-prod", pod="api-xxx"}

Query by Container:

    {namespace="app-prod", pod="api-xxx", container="api"}

To search for errors:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)error|exception|panic|traceback"

To search for timeout errors:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)timeout|timed out|deadline exceeded"

To search for database connection failures:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i    and kubernetes.pod.name : "api-xxx"
    and kubernetes.container.name : "api"

Query ERROR:

    kubernetes.namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"
    and log.level : "error"

Query Keywords:

    kubernetes.namespace : "app-prod"
    and kubernetes.pod.name : "api-xxx"
    and message : "connection refused"

---

## VIII. Standard Troubleshooting Process for Pod Monitoring and Logging

It is recommended that all Pod issues start with the following process.

### 8.1 Step 1: Confirm the Basic Status of the Pod

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
        Check scheduling and resources first.

    CrashLoopBackOff:
        Check previous logs and Events first.

    Running but NotReady:
        Check readinessProbe and application health first.

    Running and Ready but with business exceptions:
        Check Service,Endpoints, business metrics, and application logs first.

### 8.2 Step 2: View Pod Details and Events

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

### 8.3 Step 3: View Current Logs

    kubectl logs <pod-name> -n <namespace> --tail=100

If there are multiple containers:

    kubectl logs <pod-name> -n <namespace> -c <container-name> --tail=100

### 8.4 Step 4: View the Previous Round of Logs

If the Pod has restarted:

    kubectl logs <pod-name> -n <namespace> --previous --tail=100

Previous logs are very important.

The root cause of many CrashLoopBackOff issues can be found in the previous round of container logs.

### 8.5 Step 5: Check Resource Metrics

    kubectl top pod <pod-name> -n <namespace>

Prometheus queries:

    sum by (namespace, pod) (
      rate(container_cpu_usage_seconds_total{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}[5m])
    )

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}
    )

### 8.6 Step 6: View Centralized Logs

Loki:

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)error|exception|panic|traceback|timeout|failed"

EFK:

    kubernetes.namespace : "<namespace>"
    and kubernetes.pod.name : "<pod-name>"
    and message : ("error" or "exception" or "timeout" or "failed")

### 8.7 Step 7: Confirm the Service Backend

If the issue is related to access:

    kubectl describe svc <service-name> -n <namespace>

    kubectl get endpoints <service-name> -n <namespace>

    kubectl get endpointslice -n <namespace> | grep <service-name>

Confirm:

    Whether the Pod is included in Endpoints
    Whether the Service selector matches the Pod label
    Whether the targetPort is correct
    Whether the Pod is Ready

### 8.8 Step 8: Check the Associated Workload

Deployment:

    kubectl describe deploy <deployment-name> -n <namespace>

    kubectl rollout status deploy/<deployment-name> -n <namespace>

    kubectl rollout history deploy/<deployment-name> -n <namespace>

StatefulSet:

    kubectl describe sts <statefulset-name> -n <namespace>

DaemonSet:

    kubectl describe ds <daemonset-name> -n <namespace>

---

## IX. Scenario 1: Pod Pending

### 9.1 Phenomenon

    kubectl get pod -n app-prod

Display:

    STATUS: Pending

### 9.2 Metric Discovery

PromQL:

    kube_pod_status_phase{phase="Pending"} == 1

Alarm:

    PodPendingTooLong

### 9.3 Troubleshooting Commands

    kubectl describe pod <pod-name> -n <namespace>

    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 9.4 Common Events and Judgments

#### 9.4.1 Insufficient CPU

Event:

    0/3 nodes are available: insufficient cpu

Judgment:

    The cluster has insufficient schedulable CPU.
    The Pod's request is set too high.
### 10.1 Observations

    STATUS: CrashLoopBackOff

or:

    RESTARTS continue to increase

### 10.2 Metric Detection

PromQL:

    increase(kube_pod_container_status_restarts_total[10m]) > 3

Alarm:

    PodRestartTooOften

### 10.3 Troubleshooting Commands

    kubectl get pod <pod-name> -n <namespace> -o wide

    kubectl describe pod <pod-name> -n <namespace>

    kubectl logs <pod-name> -n <namespace> --tail=100

    kubectl logs <pod-name> -n <namespace> --previous --tail=100

### 10.4 Loki Queries

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)error|exception|panic|traceback|failed|connection refused"

### 10.5 EFK Queries

    kubernetes.namespace : "<namespace>"
    and kubernetes.pod.name : "<pod-name>"
    and message : ("error" or "exception" or "panic" or "failed" or "connection refused")

### 10.6 Key Points to Check

In the describe output:

    State
    Last State
    Reason
    Exit Code
    Started
    Finished
    Restart Count
    Events

In the logs:

    Startup failures
    Missing configurations
    Database connection errors
    Permission issues
    File not found
    Port conflicts
    Dependency services unavailable
    Code exceptions
    Panics
    Tracebacks
    Exceptions

### 10.7 Common Causes

#### 10.7.1 Configuration Errors

Logs:

    config file not found
    Invalid configuration
    Missing environment variables

Checks:

    kubectl get cm -n <namespace>

    kubectl get secret -n <namespace>

    kubectl describe pod <pod-name> -n <namespace>

### 10.7.2 Database Connection Failures

Logs:

    Database connection failed
    Connection refused
    Access denied
    Too many connections

Checks:

    Verify the existence of the database Service.
    Check if the Secret is correct.
    Ensure the database credentials are accurate.
    Confirm that NetworkPolicy is not blocking access.
    Verify DNS resolution is functioning properly.

### 10.7.3 Excessive Probe Configurations

Events:

    Liveness probe failed

Judgment:

    Application startup is slow.
    InitialDelaySeconds or timeoutSeconds are set too short.
    FailureThreshold is too low.
    Probe paths are unstable.

Actions:

    Adjust the livenessProbe settings.
    Distinguish between startupProbe, readinessProbe, and livenessProbe.
    Avoid using livenessProbe as a sole indicator of service health.

### 10.7.4 Permission Issues

Logs:

    Permission denied

Checks:

    SecurityContext
    RunAsUser
    FsGroup
    Volume permissions
    File path permissions
    Mount permissions for Secret/ConfigMap resources

### 10.7.5 Incorrect Image Entrypoint Commands

Logs:

    Exec format error
    Command not found
    No such file or directory

Checks:

    Verify the image, command, arguments, and entrypoint.
    Ensure they match the application's requirements and architecture.

### 10.8 CrashLoopBackOff Troubleshooting Summary

Key principles for dealing with CrashLoopBackOff:

    First, check previous logs.
    Then, examine Events.
    Next, review Last State and Exit Code.
    Avoid restarting immediately.
    If the Pod is already rebooting, reckless restarts are usually ineffective.

---

## Chapter Eleven: Scenario Three: Pod OOMKilled

### 11.1 Observations

    kubectl describe pod

Result:

    Reason: OOMKilled

or Prometheus alarm:

    PodOOMKilled

### 11.2 Metric Detection

PromQL:

    kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

Check memory usage:

    sum by (namespace, pod) (
      container_memory_working_set_bytes{namespace="<namespace>", pod="<pod-name>", container!="", image!=""}
    )

Check memory limits:

    kube_pod_container_resource_limits{namespace="<namespace>", pod="<pod-name>", resource="memory"}

### 11.3 Troubleshooting Commands

    kubectl describe pod <pod-name> -n <namespace>

    kubectl logs <pod-name> -n <namespace> --previous --tail=100

    kubectl top pod <pod-name> -n <namespace>

### 11.4 Loki Queries

    {namespace="<namespace>", pod="<pod-name>"} |~ "(?i)oom|memory|out of memory|killed"

### 11.5 Common Causes

- Memory limits set too low;
- Application```markdown
kubectl get endpointslice -n <namespace> | grep <service-name>

Pods:

    kubectl get pod -n <namespace> -o wide

Logs:

    kubectl logs <pod-name> -n <namespace> --tail=100

Port Listening:

    kubectl exec -it <pod-name> -n <namespace> -- ss -lntp

Temporary Test:

    kubectl run curl-test --rm -it --image=curlimages/curl -- sh

Inside the Container:

    curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>

### 12.5 Business Metric Queries

Error Rate:

    sum by (service) (
      rate(http_requests_total{status=~"5.."}[5m])
    )
    /
    sum by (service) (
      rate(httprequests_total[5m])
    )
    * 100

P95 Delay:

    histogram_quantile(
      0.95,
      sum by (le, service) (
        rate(http_request_duration_seconds_bucket[5m])
      )
    )

### 12.6 Log Queries

Loki:

    {namespace="<namespace>", app="<app>"} |~ "(?i)error|timeout|connection refused|database|redis|exception"

EFK:

    kubernetes.namespace : "<namespace>"
    and message : ("error" or "timeout" or "connection refused" or "database")

### 12.7 Common Causes

- targetPort error;
- Application listens only on 127.0.0.1;
- readinessProbe is too lenient;
- Business health checks do not verify true dependencies;
- Database connection failure;
- Redis connection failure;
- Downstream interface timeouts;
- DNS resolution issues;
- NetworkPolicy restrictions;
- Ingress forwarding errors;
- Configuration changes causing errors;
- New version bugs.

---

## Section Thirteen: Scenario Five: Abnormal Pod Log Collection

### 13.1 Symptoms

    kubectl logs shows logs exist
    but Loki/Kibana cannot find them

Or:

    No logs for Pods on a certain node
    No logs for a certain namespace
    No logs for a certain application
    Logs are present but lack namespace/pod fields

### 13.2 Troubleshooting Steps

    1. Check if the application outputs stdout/stderr.
    2. Verify if kubectl logs can display the logs.
    3. Confirm if the log agent is running.
    4. Ensure the agent covers all nodes via DaemonSet.
    5. Verify if the agent is running on the node where the Pod is located.
    6. Check if /var/log/containers and /var/log/pods are mounted.
    7. Confirm if the agent has read permissions.
    8. Verify if Kubernetes metadata is intact.
    9. Check if namespace/pod/container filters are applied.
    10. Ensure Loki/Elasticsearch is writable.
    11. Verify if the query time range and conditions are correct.
    12. Check if label/query fields are entered correctly.

### 13.3 Checking the Agent

    kubectl get pods -n logging -o wide

    kubectl get ds -n logging

    kubectl logs <agent-pod> -n logging

### 13.4 Entering the Agent to Verify Mounts

    kubectl exec -it <agent-pod> -n logging -- sh

Inside the container:

    ls -l /var/log/containers/

    ls -l /var/log/pods/

### 13.5 Determination Methods

If kubectl logs also shows no logs:

    The application may not be outputting stdout/stderr.
    The application might only write to internal container files.
    The log level settings might prevent logging.
    The application might not actually be running and generating logs.

If kubectl logs show logs, but Loki/EFK do not:

    First, check the log agent.
    Verify mount points.
    Check collection configurations.
    Examine the output destination.
    Review filtering rules.
    Confirm the time range and query conditions.

If only certain nodes lack logs:

    Focus on checking the Agent Pods on those nodes.

If only a certain namespace lacks logs:

    Verify if collection rules filter out that namespace.

If logs are present but lack pod fields:

    Check Kubernetes metadata configurations and RBAC permissions.
---

## Section Fourteen: Pod Dashboard Design

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

### 14.2 Top Overview Panel

Suggested items:

    Total Pods
    Running Pods
    Pending Pods
    Failed Pods
    NotReady Pods
    Restarted Pods
    OOMKilled Pods```markdown
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
        description: "The last reason the Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} terminated was OOMKilled."
        runbook_url: "https://wiki.example.com/runbook/pod-oomkilled"

### 15.4 Pod NotReady

    - alert: PodNotReadyTooLong
      expr: kube_pod_container_status_ready == 0
      for: 10m
      labels:
        severity: warning
        team: sre
      annotations:
        summary: "The Pod container has not been Ready for too long."
        description: "The Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} has not been Ready for more than 10 minutes."
        runbook_url: "https://wiki.example.com/runbook/pod-not-ready"

### 15.5 ERROR Logs Too Many

Loki Rule Example:

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
              summary: "Too many error logs in the Pod."
              description: "The Pod {{ $labels.namespace }}/{{ $labels.pod }} has more than 20 error logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/pod-error-logs"

### 15.6 Timeout Logs Too Many

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
              summary: "Too many timeout logs in the Pod."
              description: "The Pod {{ $labels.namespace }}/{{ $labels.pod }} has more than the threshold of timeout logs in the last 5 minutes."
              runbook_url: "https://wiki.example.com/runbook/pod-timeout"
---

## Section Sixteen: Pod Alert Grouping and Noise Reduction

### 16.1 Why Group Alerts

A single failure can trigger multiple alerts simultaneously:

    PodRestartTooOften
    PodErrorLogsTooMany
    PodTimeoutLogsTooMany
    ServiceHigh5xxErrorRate

Without grouping, these alerts could overwhelm the system.

### 16.2 AlertManager group_by Recommendations

For Pod dimension:

    group_by:
      - cluster
      - namespace
      - pod

For Application dimension:

    group_by:
      - cluster
      - namespace
      - app

For Service dimension:

    group_by:
      - cluster
      - namespace
      - service

### 16.3 Suppression Recommendations

If a critical alert has already been triggered:

    ServiceHigh5xxErrorRate

you may suppress lower-level noise alerts within the same application:

    PodErrorLogsTooMany

However, it is not recommended to suppress key root cause alerts, such as:

    PodOOMKilled
    PodPendingTooLong
    GPUXIDError
    CUDALogOOMDetected

### 16.4 Alert Label Recommendations

Use consistent labels:

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

Possible sources include:

    prometheus
    loki
    elasticsearch
    opensearch
---

## Section Seventeen: Complete Case Study One: CrashLoopBackOff

### 17.1 Alert

    PodRestartTooOften

### 17.2 Observation

    The Pod app-prod/api-xxx has restarted 5 times in the last 10 minutes.

### 17.3 Automated Troubleshooting

Check the Pod details:

    kubectl### 18.4 Diagnostic Assessment

If the following are observed:

    Continuous memory usage approaching the limit
    Previous logs indicate processing large files
    OOMKilled occurrences during peak task periods

Initial conclusion:

    The batch processing tasks are consuming more memory than allowed.

### 18.5 Recommended Actions

    1. Limit the amount of data per task.
    2. Process large files in batches.
    3. Check for memory leaks.
    4. Adjust the memory request and limit values appropriately.
    5. Increase the throttling of the task queue.
    6. Add a memory trend dashboard.

---

## Chapter Nineteen: Complete Case Three: Pod Running but Abnormal Service Access

### 19.1 Alarms

    ServiceHigh5xxErrorRate

    Additionally, log alarms indicate:

    PodErrorLogsTooMany

### 19.2 Observations

    The Pod is running and ready.
    The Service has endpoints available.
    However, user requests return a 500 error.

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

### 19.4 Diagnostic Assessment

If the following are observed:

    The endpoints are functioning normally.
    The Pod has not restarted.
    There are numerous database connection failure logs.
    The 5xx error rate has increased.

Initial conclusion:

    The Service routing is correct; the issue lies within the application's internal database dependency.

### 19.5 Recommended Actions

    1. Check the status of the database.
    2. Verify the database connection pool settings.
    3. Confirm whether there have been any changes to the Secret configuration.
    4. Review the application settings.
    5. Check recent updates or releases.
    6. Roll back changes if necessary.

---

## Chapter Twenty: Complete Case Four: Pod Logs Are Missing

### 20.1 Observations

    Logs can be viewed using `kubectl logs`, but they are not available in Grafana Loki.

### 20.2 Troubleshooting Steps

    Verify the node where the Pod is located:

    kubectl get pod api-xxx -n app-prod -o wide

    Check the logging namespace:

    kubectl get pods -n logging -o wide

    Confirm if an Agent is running on that node:

    kubectl get pods -n logging -o wide | grep <node-name>

    View the Agent logs:

    kubectl logs <agent-pod> -n logging

    Enter the Agent shell:

    kubectl exec -it <agent-pod> -n logging -- sh

    Check the mount points:

    ls -l /var/log/containers/

### 20.3 Possible Causes

- The Agent is not running on that node.
- The DaemonSet is restricted by the nodeSelector.
- Tolerations do not match.
- The `/var/log/containers` directory is not mounted correctly.
- The Agent lacks sufficient RBAC permissions.
- Collection rules may filter out certain namespaces.
- Incorrect configuration of Loki's address.
- Issues with Loki writing logs.
- Incorrect query tag settings.
- Incorrect time range for the query.

### 20.4 Recommended Actions

    1. Fix any scheduling issues with the Agent DaemonSet.
    2. Correct the hostPath mounting configuration.
    3. Adjust the Loki address settings properly.
    4. Revise the namespace filtering rules.
    5. Verify the log tags used for querying.
    6. Create a Runbook to handle PodLogsMissing scenarios.

---

## Chapter Twenty-One: Pod Observability Runbook Templates

Each Pod-related alarm recommendation should be accompanied by a Runbook.

### 21.1 Basic Structure of a Runbook

    Alarm Name:
    Alarm Description:
    Impact Scope:
    Possible Causes:
    First Steps to Take:
    Troubleshooting Commands:
    PromQL Queries:
    LogQL Queries:
    KQL Queries:
    Temporary Solutions:
    Root Cause Fixes:
    Rollback Procedures:
    Contact Information for Upgrades:

### 21.2 Example Runbook for PodRestartTooOften

    Alarm Name:
        PodRestartTooOften

    Description:
        The Pod is restarting frequently within a short period.

    Impact:
        This may lead---  
## Twenty-Four, Common Misconceptions  

### 24.1 Misconception 1: A Running Pod Means the Service is Working Normally  

Wrong.  

A running Pod only indicates that the container process is active. To determine if the service is functioning correctly, you need to check:  

- Whether the Pod is Ready;  
- If there are any issues with Service Endpoints;  
- Review application logs;  
- Examine business metrics;  
- Verify dependent services;  
- Check for 5xx error rates or latency.  

### 24.2 Misconception 2: kubectl logs Is Sufficient for All Needs  

Wrong.  

kubectl logs is useful for temporary troubleshooting of individual Pods, but for production environments, a centralized logging system is essential.  

### 24.3 Misconception 3: Prometheus Collects Logs Automatically  

Wrong.  

Prometheus collects metrics, not logs. For log collection, tools like Loki, ELK, or OpenSearch are required.  

### 24.4 Misconception 4: OOMKilled and CUDA OOM Are the Same Thing  

Wrong.  

OOMKilled refers to a container running out of memory, while CUDA OOM indicates insufficient GPU video memory.  

### 24.5 Misconception 6: Restarting a Pod Immediately Means Restarting the Entire Deployment  

Wrong.  

When a Pod restarts, you should first check previous logs, Events, Last State, and Exit Code before proceeding with any further actions.  

### 24.6 Misconception 7: A Service with Endpoints Is Always Healthy  

Wrong.  

Endpoints only confirm that a backend service exists; however, potential issues such as 500 errors, timeouts, dependency failures, or configuration errors may still exist.  

---  
## Twenty-Five, Summary  

Pod monitoring and log collection are fundamental components of Kubernetes observability. Pod metrics help answer questions like:  

- What is the current state of the Pod?  
- Is it Pending or Ready?  
- Has it restarted or experienced an OOMKilled?  
- Are there any issues with CPU, memory, or networking?  

Pod Events provide insight into what actions Kubernetes has taken regarding the Pod, such as scheduling failures or mirror creation errors. Pod logs reveal the reasons for application failures, startup issues, connection problems with dependencies, and so on. Service/Endpoints ensure that the Pod is properly connected to backend services.  

A comprehensive troubleshooting process involves:  

1. Identifying anomalies through alerts;  
2. Checking Pod metrics using Grafana;  
3. Investigating Events with kubectl describe;  
4. Analyzing application logs with kubectl logs --previous;  
5. Querying cross-replica logs using Loki/EFK;  
6. Verifying backend service connectivity via Service/Endpoints;  
7. Examining business metrics with Prometheus;  
8. Following a predefined Runbook to address issues;  
9. Finally, reviewing and optimizing alert settings and log collection strategies.  

The goal in production environments is not just to know how to use kubectl logs, but to establish a comprehensive monitoring framework that includes Pod metrics, Events, logs, backend service checks, alerts, dashboards, and runbooks—a complete closed-loop system.  

---  
## Twenty-Six, References  

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

