# 01-Loki Basic Understanding and Experiment Environment Planning

## Document Notes

This article is the first in the Loki specialty learning series, used to establish a basic understanding of the Loki log system, and to plan the Kubernetes environment, Namespace, components, access methods, storage methods, and verification paths required for the subsequent 15 Loki hands-on experiments.

This article does not directly enter Helm deployment, but instead first solves the following questions:

- What is Loki;
- Why does Kubernetes need a centralized logging system;
- What are the differences between Loki and ELK / EFK;
- What roles do Prometheus, Loki, Grafana, and Alloy play respectively;
- Where do Pod logs come from;
- Where are Pod logs typically stored in a containerd environment;
- What are the differences between kubectl logs and a centralized logging platform;
- Why is it recommended to prioritize using Grafana Alloy instead of Promtail in a new environment;
- What components are needed for subsequent Loki experiments;
- How to plan directories, Namespaces, access entry points, storage, and verification methods.

This article is the foundational piece for the subsequent Loki hands-on series.

---

## Tags

#Loki #Grafana #GrafanaAlloy #Kubernetes #PodLog #LogCollection #LogQL #SRE #Observation #LogSystem

---

## Recommended Path

Recommended path:

    10-logs/02-Loki/01-Loki Basic Understanding and Experiment Environment Planning.md

---

## One, What is Loki

Loki is a log aggregation system in the Grafana ecosystem.

It is primarily used for:

- Collecting logs;
- Storing logs;
- Querying logs;
- Displaying logs in Grafana;
- Generating alerts based on logs;
- Forming troubleshootingAssociation with Prometheus metrics.

In a Kubernetes environment, the most common usage of Loki is:

    Kubernetes Pod
      ↓
    stdout / stderr
      ↓
    Node log files
      ↓
    Grafana Alloy / Promtail / Fluent Bit
      ↓
    Loki
      ↓
    Grafana Explore / Dashboard
      ↓
    LogQL query / log alerts / troubleshooting analysis

Simple understanding:

    Prometheus is responsible for metrics.
    Loki is responsible for logs.
    Grafana is responsible for display.
    AlertManager is responsible for notifications.
    Alloy is responsible for log collection and forwarding.

Loki is very suitable for Kubernetes operations, SRE, cloud-native platforms, and log troubleshooting for GPU / AI workloads.

---

## Two, Why Kubernetes Needs a Centralized Logging System

In traditional servers, log troubleshooting is typically:

    ssh login to the server
    cd /var/log
    tail -f app.log
    grep ERROR app.log

However, this approach quickly becomes ineffective in Kubernetes.

The reasons are:

- Pods are dynamically created and destroyed;
- Pods are scheduled to different Nodes;
- Old container logs may be lost or rotated after Pod restarts;
- An application typically has multiple replicas;
- A service may span multiple Namespaces;
- Local logs may be inaccessible after node failure;
- When multiple teams share a cluster, it's impossible to let everyone log in to nodes to view logs;
- kubectl logs is only suitable for temporarily viewing a single Pod, not for long-term retrieval;
- Production environments require unified log queries by Namespace, Pod, Container, App, Node, and time range;
- Logs need to be used for alerts, post-mortem analysis, auditing, and problem tracking.

Therefore, Kubernetes production environments typically require:

    Application output stdout / stderr
      ↓
    Container runtime writes to node logs
      ↓
    Log collection Agent collects logs
      ↓
    Write to centralized logging system
      ↓
    Grafana / Kibana query
      ↓
    Log alerts / troubleshooting / post-mortem analysis

---

## Three, Differences Between Loki and ELK / EFK

### 3.1 What is ELK / EFK

ELK typically refers to:

    Elasticsearch
    Logstash
    Kibana

EFK typically refers to:

    Elasticsearch
    Fluentd / Fluent Bit
    Kibana

It can also use:

    OpenSearch
    Fluent Bit
    OpenSearch Dashboards

ELK / EFK is more like a full-text search-based logging system.

Features:

- Supports full-text search;
- Strong field query capabilities;
- Strong aggregation analysis capabilities;
- Mature query experience with Kibana;
- Suitable for auditing, security, and business log retrieval;
- Relatively high resource consumption;
- Complex index, shard, and lifecycle management.

### 3.2 Features of Loki

Core features of Loki:

    Mainly indexes log labels, does not default to full-text indexing of log content.

In other words, Loki emphasizes:

    First use labels to narrow the scope
    Then search for keywords in the log content

For example:

    {namespace="prod", app="api"} |= "ERROR"

Meaning:

    First filter logs from the prod namespace for the api application
    Then search for "ERROR" in these log contents

Advantages of Loki:

- Well-integrated with Grafana;
- Closely aligned with Prometheus label model;
- Suitable for Kubernetes Pod logs;
- Relatively more cost-effective;
- Very suitable for operations troubleshooting;
- Very suitable forAssociation with Prometheus metrics;
- Lower deployment and usage threshold compared to a full ELK cluster.

Limitations of Loki:

- Not suitable as a complete replacement for Elasticsearch as a complex full-text search platform;
- Not suitable for building complex indexes on all log fields;
- Not suitable for using high-cardinality fields as labels;
- Poor label design can severely impact performance.

### 3.3 Selection Recommendations

If the goal is:

    Centralized viewing of Kubernetes application logs
    Pod anomaly troubleshooting
    Service 5xx troubleshooting
    Grafana metric and logAssociation
    Relatively controllable costs
    Log alerts
    SRE daily operations

Prioritize Loki.

If the goal is:

    Security auditing
    Complex full-text search
    Large-scale business field search
    Multi-field aggregation analysis
    Log compliance long-term retention
    Security operations analysis

Consider ELK / OpenSearch.

Production environments can also coexist:

    Loki:
        Focused on cloud-native operations troubleshooting.

    Elasticsearch / OpenSearch:
        Focused on full-text search, auditing, security, and complex log analysis.

## Four. The Relationship Between Prometheus, Loki, Grafana, and Alloy

### 4.1 Prometheus

Prometheus is responsible for metrics.

It collects:

- Node CPU;
- Node memory;
- Pod CPU;
- Pod memory;
- Pod restart count;
- Pod status;
- Service QPS;
- Service error rate;
- P95 / P99 latency;
- GPU memory;
- GPU utilization;
- Exporter metrics;
- Application /metrics metrics.

Prometheus does not collect logs.

### 4.2 Loki

Loki is responsible for logs.

It receives logs sent by log collection agents, performs storage and querying.

Loki is suitable for querying:

- Logs of a specific Pod;
- Logs of a specific Namespace;
- Error logs of a specific App;
- Timeout logs;
- Database connection failed;
- Java Exception;
- Python Traceback;
- CUDA out of memory;
- Panic / fatal logs.

### 4.3 Grafana

Grafana is the unified display entry point.

It can integrate with:

    Prometheus
    Loki
    Elasticsearch
    OpenSearch
    MySQL
    PostgreSQL

In Loki scenarios, Grafana is mainly used for:

- Explore log queries;
- Dashboard to display log trends;
- Jump from Pod metrics to Pod logs;
- Display ERROR log trends;
- Display timeout log trends;
- View Loki alerts;
- Combine Prometheus metrics and Loki logs in the same troubleshooting view.

### 4.4 Grafana Alloy

Grafana Alloy is the new generation log collection agent in the Grafana ecosystem.

It can collect:

- logs;
- metrics;
- traces;
- profiles;
- Kubernetes Pod logs;
- Kubernetes Events;
- Node logs.

In Loki new environments, it is recommended to prioritize using:

    Grafana Alloy → Loki

Instead of continuing to create:

    Promtail → Loki

### 4.5 Promtail

Promtail is the log collection agent commonly used in Loki's early days.

Historically, it was often seen as:

    Promtail DaemonSet
      ↓
    Loki
      ↓
    Grafana

However, Promtail has reached EOL and is not recommended for new environments as a long-term solution.

Learning still requires knowing Promtail, as many historical clusters and old documents still use it.

---

## Five. Where Do Pod Logs Come From

### 5.1 Standard Output and Standard Error

Kubernetes recommends container applications to output logs to:

    stdout
    stderr

That is, standard output and standard error.

The container runtime will take over these outputs and write them to log files on the node.

Applications are not recommended to only write logs to internal container files.

Reasons:

- Files may be lost after container rebuilds;
- Log collection agents typically collect stdout / stderr corresponding container log files by default;
- kubectl logs defaults to reading container standard output logs;
- Logs from multiple replicas are difficult to unify;
- Production troubleshooting is inconvenient.

Recommendation:

    Application logs → stdout / stderr

Not recommended:

    Application logs only written to /app/logs/app.log

If the business indeed needs to write files, it is necessary to configure log collection agents to collect this path separately.

### 5.2 Log Paths in containerd Environment

Common paths in containerd environments:

    /var/log/containers/
    /var/log/pods/

Common file formats:

    /var/log/containers/<pod>_<namespace>_<container>-<container-id>.log

Check node log paths:

    ls -l /var/log/containers/

    ls -l /var/log/pods/

These paths are the main targets for log agents like Alloy, Fluent Bit, Filebeat, etc.

### 5.3 kubectl logs

View current Pod logs:

    kubectl logs <pod-name> -n <namespace>

View logs of the previous container instance:

    kubectl logs <pod-name> -n <namespace> --previous

View logs of a specific container:

    kubectl logs <pod-name> -n <namespace> -c <container-name>

View the last 100 lines:

    kubectl logs <pod-name> -n <namespace> --tail=100

Continuous viewing:

    kubectl logs <pod-name> -n <namespace> -f

View the last 10 minutes:

    kubectl logs <pod-name> -n <namespace> --since=10m

### 5.4 Limitations of kubectl logs

kubectl logs is very suitable for temporary troubleshooting.

But it is not suitable as a production log system.

Limitations:

- Not suitable for long-term log storage;
- Not suitable for cross-Pod retrieval;
- Not suitable for cross-Namespace retrieval;
- Not suitable for keyword aggregation;
- Not suitable for Dashboard;
- Not suitable for log alerts;
- Logs may be unavailable after Pod deletion;
- Logs may be unavailable after node failure;
- Troubleshooting efficiency is low for multi-replica services.

Therefore, a centralized log system like Loki is needed.

---

## Six. Basic Architecture Understanding of Loki

Loki can run in different deployment modes.

Common modes:

    monolithic
    simple scalable
    microservices

### 6.1 Monolithic Single Instance Mode

Features:

    All Loki components run in a single process or a group of single-instance replicas.

Suitable for:

- Learning;
- Experimentation;
- Small-scale environments;
- Not a large volume of logs;
- Rapid feature verification.

Advantages:

- Simple deployment;
- Simple configuration;
- Suitable for understanding the overall chain;
- Suitable for early-stage experiments in this series.

Disadvantages:

- Limited scalability;
- Limited high availability;
- Not suitable for large-scale production.

### 6.2 Simple Scalable Mode

Features:

    Loki is divided into roles such as read, write, and backend.

Suitable for: /think

- Small to medium-scale production;
- Hope to have some scalability;
- Don't want to adopt a complex microservices model at first;
- Cluster with gradually increasing log volume.

Advantages:

- Read and write can be scaled separately;
- Architecture is closer to production than monolithic;
- Moderate complexity.

Disadvantages:

- More complex configuration than monolithic;
- Requires object storage;
- Needs to understand component roles.

### 6.3 Microservices Mode

Features:

    Loki components are fully split and deployed separately.

May include:

    distributor
    ingester
    querier
    query-frontend
    query-scheduler
    compactor
    ruler
    index-gateway
    gateway

Suitable for:

- Large-scale log platform;
- Multi-tenancy;
- High write throughput;
- High query concurrency;
- Has a dedicated platform team for maintenance.

Advantages:

- Strong scalability;
- Components can be independently tuned;
- Suitable for large-scale production.

Disadvantages:

- Complex architecture;
- Many parameters;
- Troubleshooting is difficult;
- Not suitable as the first step.

### 6.4 This Series' Experimental Selection

This series adopts a phased approach:

    Phase 1:
        Monolithic mode, first run Loki.

    Phase 2:
        Integrate Alloy to collect K8S Pod logs.

    Phase 3:
        Integrate MinIO object storage.

    Phase 4:
        Learn LogQL, Grafana Dashboard, and log alerts.

    Phase 5:
        Learn Simple Scalable mode and production governance.

This series will not directly enter Microservices mode at the beginning.

Reasons:

    Ensure the pipeline is working during the learning phase.
    First understand where logs come from, how to collect, how to query, and how to alert.
    Then gradually understand high availability and scalability modes.

---

## SevenI don't know.This Series' Experimental Environment Planning

### 7.1 Kubernetes Cluster Planning

Experimental environment:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Optional GPU node:

    k8s-gpu-node01  10.0.0.30

System recommendation:

    Ubuntu Server 22.04.5 LTS

Container runtime:

    containerd

Kubernetes deployment method:

    kubeadm

### 7.2 Namespace Planning

Recommended to create the following Namespaces:

    logging
    monitoring
    app-demo
    minio

Notes:

    logging:
        Deploy Loki, Alloy, and other log components.

    monitoring:
        Deploy Prometheus, Grafana, AlertManager, and other monitoring components.

    app-demo:
        Deploy test applications to generate logs.

    minio:
        Deploy MinIO for use in Loki's object storage experiment.

Creation commands:

    kubectl create namespace logging
    kubectl create namespace monitoring
    kubectl create namespace app-demo
    kubectl create namespace minio

Verification:

    kubectl get ns

### 7.3 Component Planning

This series will gradually use the following components:

    Loki
    Grafana
    Grafana Alloy
    MinIO
    Prometheus
    AlertManager
    Example application app-demo
    Optional Elasticsearch / OpenSearch comparison
    Optional DCGM Exporter for GPU scenarios

Minimum components in Phase 1:

    Loki
    Grafana
    Grafana Alloy
    app-demo

Enhanced components later:

    MinIO
    Prometheus
    AlertManager

### 7.4 Access Method Planning

Learning environment can prioritize using port-forward.

Grafana:

    kubectl port-forward svc/<grafana-service> 3000:80 -n monitoring

Loki:

    kubectl port-forward svc/<loki-service> 3100:3100 -n logging

MinIO:

    kubectl port-forward svc/<minio-service> 9000:9000 -n minio
    kubectl port-forward svc/<minio-console-service> 9001:9001 -n minio

Production environment recommendations:

    Ingress
    HTTPS
    Authentication
    RBAC
    Network access control

Don't rush to use Ingress exposure in the learning phase.

---

## EightI don't know.This Series' Directory Planning

Loki-specific directory:

    10-Logs/02-Loki/

Planned notes:

    01-Loki Basics and Experimental Environment Planning.md
    02-Loki Architecture Principles and Component Responsibilities.md
    03-Loki Deployment Mode Comparison and Experimental Selection.md
    04-Loki Monolithic Mode Helm Deployment.md
    05-Loki Object Storage Integration with MinIO.md
    06-Grafana-Alloy Kubernetes Pod Log Collection.md
    07-Loki Label Design and High Cardinality Issues.md
    08-LogQL Basics: Namespace-Pod-Container Log Search.md
    09-LogQL Advanced: json-logfmt-regexp-unwrap.md
    10-Grafana Integration with Loki and Log Dashboard.md
    11-Loki Log Alerting: Ruler and AlertManager Integration.md
    12-Loki Production Governance: Log Volume - Retention - Throttling - Security.md
    13-Loki Performance and High Availability: Simple-Scalable Mode.md
    14-Loki Common Troubleshooting: Collection Failure - Slow Query - Write Failure.md
    15-Loki Comprehensive Troubleshooting: Pod Anomalies - Service - 5xx - CUDA-OOM Log Loop.md

---

## NineI don't know.Pre-experiment Checks

### 9.1 Check kubectl

Check version:

    kubectl version --client

View Cluster:

    kubectl cluster-info

View Nodes:

    kubectl get nodes -o wide

Expected:

    All nodes Ready.

### 9.2 Check Helm

View Helm version:

    helm version

Add Grafana Helm repository:

    helm repo add grafana https://grafana.github.io/helm-charts

Update repository:

    helm repo update

View Loki Chart:

    helm search repo grafana/loki --versions

View Alloy Chart:

    helm search repo grafana/alloy --versions

View Grafana Chart:

    helm search repo grafana/grafana --versions

### 9.3 Check Storage Classes

View StorageClass:

    kubectl get storageclass

If there is no default StorageClass, subsequent Loki PVC or MinIO PVC will need to be handled separately.

In a learning environment, you can first use:

    hostPath
    local-path-provisioner
    NFS StorageClass
    Longhorn
    Manual PV/PVC

In production environments, it is recommended to use reliable block storage or object storage.

### 9.4 Check Node Log Path

Execute on any Worker node:

    ls -l /var/log/containers/

    ls -l /var/log/pods/

If the path exists, it indicates that the container log path basically meets the requirements for subsequent log collection.

If the path does not exist, you need to confirm:

    Whether Kubernetes is running Pods normally
    Whether containerd configuration is normal
    Whether there are existing business Pods on the node
    Whether the log path has been cleaned or there are system differences

### 9.5 Create Test Application Namespace

Create app-demo:

    kubectl create namespace app-demo

If it already exists, it will prompt AlreadyExists, which can be ignored.

Verification:

    kubectl get ns app-demo

---

## Ten. Prepare a Test Application That Generates Logs

Subsequent Loki learning requires a stable log source.

You can first deploy a simple Nginx as a basic log source.

### 10.1 Deploy Nginx

    kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo

Expose Service:

    kubectl expose deployment nginx-demo \
      --port=80 \
      --target-port=80 \
      --type=ClusterIP \
      -n app-demo

View:

    kubectl get pod -n app-demo -o wide

    kubectl get svc -n app-demo

### 10.2 Access Nginx to Generate Logs

Create a temporary curl Pod:

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute in the curl-test container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

Exit:

    exit

### 10.3 View kubectl logs

View Pod:

    kubectl get pod -n app-demo

View logs:

    kubectl logs <nginx-pod-name> -n app-demo --tail=50

If you can see the access logs, it indicates:

    Application logs are output normally.
    kubectl logs is readable.
    The subsequent log Agent can collect this log source.

---

## Eleven. Preparation Ideas for a Test Application That Outputs Error Logs

Nginx access logs are suitable for basic verification, but not suitable for LogQL advanced usage.

Subsequently, you need a demo application that can output the following content:

    INFO logs
    WARN logs
    ERROR logs
    JSON logs
    status field
    duration_ms field
    timeout field
    exception field

Example log format:

    {"timestamp":"2026-04-30T12:00:00+08:00","level":"info","service":"app-demo","msg":"request success","status":200,"duration_ms":32}

    {"timestamp":"2026-04-30T12:00:03+08:00","level":"error","service":"app-demo","msg":"database connection failed","status":500,"duration_ms":1200}

    {"timestamp":"2026-04-30T12:00:05+08:00","level":"error","service":"app-demo","msg":"timeout waiting for upstream","status":504,"duration_ms":3000}

In the 09th article, we will use structured logs for practice:

    | json
    level="error"
    status>=500
    unwrap duration_ms
    avg_over_time
    quantile_over_time

---

## Twelve. Recommended Learning Order for This Series

It is recommended not to skip lessons.

Recommended order:

    01:
        First understand what Loki is and prepare the experimental environment.

    02:
        Observe Loki architecture and component responsibilities.

    03:
        Compare deployment modes and clarify why we start with a single-node mode.

    04:
        Helm deploy Loki single-node mode.

    05:
        Integrate with MinIO object storage.

    06:
        Use Alloy to collect K8S Pod logs.

    07:
        Learn label design and high cardinality issues.

    08:
        Learn LogQL basic queries.

9:
        Learn advanced LogQL parsing and statistics.

    10:
        Integrate Grafana with Loki and create a Dashboard.

    11:
        Integrate Loki log alerts with AlertManager.

    12:
        Learn production governance, retention periods, throttling, and security.

    13:
        Learn the Simple Scalable mode.

    14:
        Learn common troubleshooting scenarios.

    15:
        Perform a comprehensive troubleshooting case.

---

## Thirteen. Key Understandings in Loki Learning

### 13.1 Loki is not a complete replacement for Elasticsearch

Loki is more suitable for:

    Kubernetes operations logging
    Pod log troubleshooting
    Integration with Grafana
    Integration with Prometheus metrics
    Low-cost log aggregation

Elasticsearch is more suitable for:

    Full-text search
    Audit logs
    Security logs
    Complex field search
    Business log analysis

### 13.2 Loki's core is label design

Loki queries are typically:

    Select labels first
    Then search content

Good query:

    {namespace="app-demo", app="nginx-demo"} |= "404"

Bad query:

    {namespace=~".+"} |= "error"

Issues:

    Too broad scope.
    Slow query.
    Easily impacts Loki.
    May be restricted in production environments.

### 13.3 Labels are not the more, the better

Recommended labels:

    cluster
    environment
    namespace
    pod
    container
    node
    app
    team

Not recommended labels:

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message
    stacktrace

Reasons:

    These fields have high cardinality.
    Causes log stream explosion.
    Increases index pressure.
    Reduces query and write performance.

### 13.4 Prometheus and Loki should be used together

Prometheus detects phenomena:

    Pod restarts
    5xx increase
    P95 latency increase
    GPU memory high
    Pod OOMKilled

Loki finds the cause:

    database connection failed
    timeout
    Traceback
    Exception
    CUDA out of memory
    panic

Troubleshooting flow:

    Prometheus alert
      ↓
    Grafana view metric trends
      ↓
    Loki check log context
      ↓
    kubectl describe view Events
      ↓
    Service / Endpoints verify traffic
      ↓
    Runbook handling

### 13.5 New environments should prioritize Alloy

New environment recommendations:

    Grafana Alloy
      ↓
    Loki

Legacy environments may still be:

    Promtail
      ↓
    Loki

You need to know Promtail for learning, but operational focus should be on Alloy.

---

## Fourteen. Experimental Risks and Production Considerations

### 14.1 Do not directly copy learning configurations to production

Learning environments can:

    Single-node mode
    port-forward
    Simple PVC
    Default resources
    Simple authentication
    Short-term retention

Production environments should consider:

    High availability
    Object storage
    Retention period
    Resource requests/limits
    Query limits
    Write throttling
    Tenant isolation
    HTTPS
    Authentication
    Alerts
    Backup
    Audit

### 14.2 Logs may contain sensitive information

Production logs must avoid outputting:

    Passwords
    tokens
    access keys
    secret keys
    Authorization headers
    Cookies
    Private keys
    Plaintext phone numbers
    IDs
    Bank cards
    Database connection string passwords

Principles:

    Application-side avoidance of sensitive information is the top priority.
    Data desensitization at the collection side is supplementary.
    Do not rely on log platform post-cleanup of sensitive data.

### 14.3 Log volume needs governance

Logs are not the more, the better.

Excessive log volume will lead to:

- Increased storage costs;
- Higher Loki write pressure;
- Slower queries;
- Increased alert noise;
- Resource consumption by collection agents;
- Increased network bandwidth.

Production recommendations:

    Control DEBUG logs.
    Filter health check logs.
    Limit large field outputs.
    Set reasonable retention.
    Govern high log volume applications.
    Regularly statistics Top log sources.

---

## Fifteen. This Experiment Tasks

This section does not deploy Loki, only completing environment planning and basic verification.

### 15.1 Create Namespace

    kubectl create namespace logging
    kubectl create namespace monitoring
    kubectl create namespace app-demo
    kubectl create namespace minio

Verification:

    kubectl get ns

### 15.2 Add Grafana Helm repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

Verification:

    helm search repo grafana/loki --versions

    helm search repo grafana/alloy --versions

    helm search repo grafana/grafana --versions

### 15.3 Deploy Nginx Test Application

    kubectl create deployment nginx-demo \
      --image=nginx:1.25 \
      --replicas=2 \
      -n app-demo

kubectl expose deployment nginx-demo \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP \
  -n app-demo

View:

  kubectl get pod -n app-demo -o wide

  kubectl get svc -n app-demo

### 15.4 Generate Logs

  kubectl run curl-test \
    --rm -it \
    --image=curlimages/curl:8.5.0 \
    -n app-demo \
    -- sh

Execute inside container:

  curl http://nginx-demo.app-demo.svc.cluster.local

  curl http://nginx-demo.app-demo.svc.cluster.local/notfound

Exit:

  exit

### 15.5 View Pod Logs

View Pod:

  kubectl get pod -n app-demo

View logs:

  kubectl logs <nginx-pod-name> -n app-demo --tail=50

Expected:

  You can see Nginx access log.

### 15.6 View Node Log Paths

Execute on node where Pod resides:

  ls -l /var/log/containers/ | grep nginx-demo

  ls -l /var/log/pods/ | grep app-demo

Expected:

  You can find the container log files or Pod log directory corresponding to nginx-demo.

---

## Sixteen, Acceptance Checklist

After completing this section, confirm:

  [ ] Logging Namespace has been created
  [ ] Monitoring Namespace has been created
  [ ] app-demo Namespace has been created
  [ ] minio Namespace has been created
  [ ] kubectl can access the cluster normally
  [ ] Helm has been installed
  [ ] Grafana Helm repository has been added
  [ ] Can search for grafana/loki Chart
  [ ] Can search for grafana/alloy Chart
  [ ] nginx-demo has been deployed
  [ ] nginx-demo Service has been created
  [ ] curl-test can access nginx-demo
  [ ] kubectl logs can view nginx-demo logs
  [ ] /var/log/containers exists on node
  [ ] /var/log/pods exists on node
  [ ] Can find test Pod-related logs in node log directory
  [ ] Loki series directory planning has been clarified

---

## Seventeen, Common Issues

### 17.1 kubectl logs has no logs

Possible causes:

- Application has no stdout / stderr output;
- Pod has not received requests;
- Wrong Namespace is being viewed;
- Wrong Pod is being viewed;
- Multi-container Pod has not specified container;
- Logs have been rotated;
- Container has restarted, need to add --previous.

Troubleshoot:

  kubectl get pod -n app-demo

  kubectl logs <pod-name> -n app-demo --tail=100

  kubectl logs <pod-name> -n app-demo --previous --tail=100

### 17.2 curl-test fails to access nginx-demo

Check Service:

  kubectl get svc -n app-demo

  kubectl describe svc nginx-demo -n app-demo

Check Endpoints:

  kubectl get endpoints nginx-demo -n app-demo

Check Pod:

  kubectl get pod -n app-demo -o wide

Possible causes:

- Service has not been created;
- Pod is not Ready;
- DNS anomaly;
- NetworkPolicy blocks access;
- Image failed to pull;
- Wrong Namespace.

### 17.3 Cannot find /var/log/containers on node

Possible causes:

- Current node does not run business Pod;
- Need to check Pod's Node;
- Cluster runtime or system path differences;
- Insufficient permissions;
- Node is not Kubernetes Worker;
- Log directory has been cleaned.

First check Pod's node:

  kubectl get pod -n app-demo -o wide

Then login to corresponding node and check:

  ls -l /var/log/containers/

### 17.4 Helm cannot find grafana/loki

Check Helm repositories:

  helm repo list

Re-add:

  helm repo add grafana https://grafana.github.io/helm-charts

Update:

  helm repo update

Search again:

  helm search repo grafana/loki --versions

---

## Eighteen, Production Environment Extensions

### 18.1 Differences Between Learning and Production Environments

Learning environment focus:

  Run through the entire chain
  Understand log sources
  Understand Alloy collection
  Understand LogQL
  Understand Grafana queries
  Understand Loki alerts

Production environment focus:

  High availability
  Object storage
  Multi-replica
  Rate limiting
  Query optimization
  Tenant isolation
  Log desensitization
  Permission control
  Cost governance
  Alert convergence
  Runbook

### 18.2 Loki in Production is More Than Just Installation

Production Loki requires attention to:

  Write volume
  Query volume
  Label cardinality
  Retention period
  Object storage
  Caching
  Query limits
  Alert rules
  Self-monitoring
  Collection Agent resources
  Log security

If you only do:

  helm install loki

You're not truly mastering Loki.

What you truly need to do:

  Know where logs come from
  Know how logs are collected
  Know how to query logs
  Know how to design labels
  Know how to troubleshoot slow queries
  Know how to locate failed writes
  Know how to set up alerts
  Know how to govern in production

---

## Nineteen, Summary

Loki is a log aggregation system in the Grafana ecosystem, particularly suitable for Kubernetes Pod log collection, operations and troubleshooting, Grafana integration, and SRE scenarios.

This article requires establishing several core concepts:

    Prometheus collects metrics.
    Loki stores and queries logs.
    Grafana displays metrics and logs.
    Alloy collects and forwards logs.
    AlertManager handles alert notifications.

Kubernetes Pod logs typically originate from:

    Application stdout / stderr
      ↓
    containerd writes to node log files
      ↓
    /var/log/containers
      ↓
    Alloy / Fluent Bit / Filebeat
      ↓
    Loki / Elasticsearch / OpenSearch

This series of Loki learning path adopts:

    First run a single-node mode
    Then integrate Alloy
    Then integrate MinIO
    Then learn LogQL
    Then create Grafana Dashboard
    Then implement log alerts
    Then handle production governance and troubleshooting

The goal of learning Loki is not just to query logs, but to form complete capabilities:

    Pod log collection
    Label design
    LogQL queries
    Grafana integration
    Log alerts
    Production governance
    Troubleshooting

The next article will enter:

    02-Loki Architecture Principles and Component Responsibilities Practical Observation

Focus on observing Loki's component responsibilities, service entry points, Pod structure, and log writing/querying pipeline.

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Install Grafana Loki with Helm:
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Loki Helm Chart:
  https://github.com/grafana/loki/tree/main/production/helm/loki

- Grafana Alloy Documentation:
  https://grafana.com/docs/alloy/latest/

- Collect Kubernetes logs and forward them to Loki:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- Grafana Alloy loki.source.kubernetes:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- Promtail Agent:
  https://grafana.com/docs/loki/latest/send-data/promtail/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes kubectl logs:
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/

- Helm Documentation:
  https://helm.sh/docs/

- Grafana Documentation:
  https://grafana.com/docs/grafana/latest/