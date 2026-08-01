# 02-Loki Architecture Principles and Component Responsibilities Practical Observation

## Document Overview

This is the second article in the Loki specialty learning series, used to systematically understand Loki's architecture principles, core component responsibilities, log ingestion pipeline, log query pipeline, and how to observe Loki's runtime status in Kubernetes through kubectl, Helm, Loki HTTP API, and log output.

This article is not simply memorizing component names like Distributor, Ingester, Querier, but answering the following questions from an operational perspective:

- Why is Loki a modular architecture;
- What are the differences between components in single-node mode, Simple Scalable mode, and Microservices mode;
- What components does log go through after being written from Alloy to Loki;
- How does Loki internally process log queries in Grafana;
- What responsibilities do Distributor, Ingester, Querier, Query Frontend, Compactor, and Ruler respectively have;
- What roles do Gateway, Index Gateway, Cache, and Object Storage play in production;
- Why are there not many independent Pods visible in single-node mode;
- How to use kubectl to observe Loki Pods, Services, StatefulSets, and Deployments;
- How to use Loki API to view ready, metrics, ring, and config information;
- How to judge whether Loki ingestion, query, compression, and alerting are normal through logs and metrics;
- What phenomena should be focused on when deploying Loki in the future.

This article is suitable to learn after completing the following content:

- 01-Loki Basic Concepts and Experiment Environment Planning

This article is followed by:

- 03-Loki Deployment Mode Comparison and Experiment Selection
- 04-Loki Single-Node Helm Deployment Practical Guide
- 05-Loki Object Storage Integration with MinIO Practical Guide
- 06-Grafana-Alloy Collection K8S-Pod Logs Practical Guide

---

## Tags

#Loki #Grafana #GrafanaAlloy #Kubernetes #LogSystem #LokiArchitecture #Distributor #Ingester #Querier #Compactor #Ruler #QueryFrontend #SRE #Observation

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/02-Loki Architecture Principles and Component Responsibilities Practical Observation.md

---

## One, Experiment Objectives

After completing this article, you should be able to understand and verify:

    1. What each core component of Loki does.
    2. How the Loki ingestion pipeline works.
    3. How the Loki query pipeline works.
    4. What differences there are in component presentation between single-node mode and split mode.
    5. How to observe Loki Pods, Services, logs, and metrics in Kubernetes.
    6. How to use API to determine if Loki is ready.
    7. How to use /metrics to observe Loki's own metrics.
    8. How to understand the relationship between Loki and Alloy, Grafana, and object storage.

The focus of this article is:

    First understand the architecture.
    Then observe phenomena.
    Deploy formally later.

---

## Two, Experiment Environment

This article assumes the following Kubernetes experiment environment:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespace planning:

    logging
    monitoring
    app-demo
    minio

Components involved in this article:

    Loki
    Grafana
    Grafana Alloy
    Prometheus
    AlertManager
    MinIO

Tools used in this article:

    kubectl
    helm
    curl
    grep
    jq

Notes:

    This article focuses on architecture observation.
    If Loki has not been deployed yet, you can first read about the architecture and commands.
    The actual observation commands can be executed after deploying Loki in Chapter 04.
    If Loki already exists in the environment, you can directly execute the observation commands in this article.

---

## Three, Loki Overall Architecture

Loki is a log aggregation system.

Its core idea is:

    Log content is not indexed by default.
    Mainly indexed by labels.
    When querying, first narrow down the scope through labels, then filter log content.

Typical Kubernetes log pipeline:

    Kubernetes Pod
      ↓
    stdout / stderr
      ↓
    containerd writes to node log files
      ↓
    /var/log/containers/*.log
      ↓
    Grafana Alloy / Promtail / Fluent Bit
      ↓
    Loki Distributor
      ↓
    Loki Ingester
      ↓
    Chunk / Index / Object Storage
      ↓
    Loki Querier / Query Frontend
      ↓
    Grafana Explore / Dashboard

Complete observability pipeline:

    Prometheus:
        Responsible for metrics.

    Loki:
        Responsible for logs.

    Grafana:
        Responsible for display and query entry.

    Alloy:
        Responsible for log collection and forwarding.

    AlertManager:
        Responsible for alert notifications.

    MinIO / S3:
        Responsible for object storage.

---

## Four, Loki's Three Operation Modes

Loki's code can be run in different ways.

Common deployment modes:

    Monolithic / Single Binary
    Simple Scalable
    Microservices

### 4.1 Monolithic / Single Binary

In single-node mode, Loki's multiple capabilities run in a single Loki process.

Common manifestations:

    Kubernetes may have only one Loki Pod or a few Loki replicas.
    Distributor, Ingester, Querier, Compactor, etc., capabilities run in the same process.
    You cannot see independent Deployments for each component.

Suitable for:

    Learning environment
    Small-scale experiments
    Function verification
    Environments with smaller log volumes

Advantages:

    Simple deployment.
    Simple configuration.
    Fewer components.
    Suitable for understanding Loki's basic pipeline.

Disadvantages:

Limited scalability.  
Weak high availability capabilities in production.  
Read and write cannot be fully independently scaled.  
Not suitable for large-scale log platforms.

### 4.2 Simple Scalable

The Simple Scalable mode splits Loki into logical roles:

    read
    write
    backend

Common roles:

    write:
        Responsible for log writing paths, including distributor, ingester, etc.

    read:
        Responsible for log query paths, including querier, query frontend, etc.

    backend:
        Responsible for compactor, ruler, index gateway, etc. backend capabilities.

Common manifestations:

    kubectl get pods -n logging

May see:

    loki-read-xxx
    loki-write-xxx
    loki-backend-xxx
    loki-gateway-xxx

Suitable for:

    Medium to small-scale production
    Requires read-write separation
    Needs better scalability than single-instance mode
    Environments that don't want to directly enter complex microservices mode

Notes:

    The official documentation has indicated that Simple Scalable Deployment will be deprecated in the future, specific time needs to be continuously followed up on the official documentation.
    Still need to understand it during learning, because many existing environments and materials will use this mode.
    The 13th article will specifically study the Simple Scalable mode.

### 4.3 Microservices

In the microservices mode, Loki components run independently.

May see:

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

    Large-scale log platforms
    High write throughput
    High query concurrency
    Multi-tenancy
    Special platform teams maintenance

Advantages:

    Each component can be independently scaled.
    Each component can be independently tuned.
    Best architectural elasticity.

Disadvantages:

    Many components.
    Complex configuration.
    Complex troubleshooting.
    Not recommended for direct use in the initial stage.

---

## FiveI don't know.Loki Core Components Overview

Loki common core components are as follows:

    Distributor
    Ingester
    Querier
    Query Frontend
    Query Scheduler
    Compactor
    Ruler
    Index Gateway
    Gateway
    Cache
    Object Storage

Simplified understanding:

    Write entry point:
        Distributor

    Write buffering and chunking:
        Ingester

    Query execution:
        Querier

    Query splitting, caching, queuing:
        Query Frontend / Query Scheduler

    Background compression and retention:
        Compactor

    Log alerting rules:
        Ruler

    Index access optimization:
        Index Gateway

    External unified entry point:
        Gateway

    Long-term storage:
        Object Storage

---

## SixI don't know.Component One: Distributor

### 6.1 Distributor Role

Distributor is the write entry point of Loki.

Log collection agents push logs to Distributor.

Common sources:

    Grafana Alloy
    Promtail
    Fluent Bit
    Logstash
    Custom client

Distributor mainly responsible for:

    Receiving log write requests
    Validating log format
    Validating tenant information
    Validating timestamp
    Validating label
    Executing rate limiting
    Distributing logs to Ingester

### 6.2 Write API

Common Loki write API path:

    /loki/api/v1/push

Typical flow:

    Alloy
      ↓
    POST /loki/api/v1/push
      ↓
    Distributor
      ↓
    Ingester

### 6.3 What to Pay Attention to for Distributor

In production, pay attention to:

    Whether write requests are successful
    Whether 400 occurs
    Whether 429 occurs
    Whether 500 occurs
    Whether label limits are exceeded
    Whether log lines are too large
    Whether ingestion rate limit is exceeded
    Whether tenant authentication fails

Common errors:

    400 Bad Request:
        Invalid log format, timestamp, or label.

    401 / 403:
        Authorization or tenant permission issues.

    429 Too Many Requests:
        Write rate exceeds the limit.

    500:
        Internal Loki error or downstream component error.

---

## SevenI don't know.Component Two: Ingester

### 7.1 Ingester Role

Ingester is responsible for receiving logs distributed by Distributor and organizing them into chunks.

It mainly responsible for:

    Receiving log streams
    Temporarily storing logs
    Organizing logs by label stream
    Building chunks
    Writing to WAL
    Flushing chunks to object storage
    Maintaining query capabilities for recent data

### 7.2 What is a Stream

In Loki, logs with the same group of labels belong to the same stream.

For example:

    {namespace="app-demo", pod="nginx-xxx", container="nginx"}

This is a log stream.

If labels change, a new stream will be generated.

For example:

    request_id="abc"
    request_id="def"

If request_id is made into a label, each request may become a new stream.

This is the high cardinality problem.

### 7.3 What is a Chunk

Chunk is the basic storage unit for logs in Loki.

Simple understanding:

    Multiple log lines
      ↓
    Grouped by stream
      ↓
    Split into chunks by time and size
      ↓
    Written to storage

Ingester will flush chunks to backend storage according to configuration.

Backend storage can be: /think

# File System  
MinIO  
S3  
GCS  
Azure Blob  
Other Compatible Object Storage  

### 7.4 What Ingester Should Monitor  

Production Monitoring:  

    Memory Usage  
    WAL Status  
    Chunk Flush Failure  
    Ingester Frequent Restarts  
    Active Stream Count  
    Stream Overload  
    Flush Latency  
    Object Storage Write Success  

Common Issues:  

    Label Design Errors Causing Stream Explosion.  
    Ingester Memory Increase.  
    Object Storage Unavailability Leading to Flush Failure.  
    WAL Recovery Taking Too Long.  
    Pod Restarts Causing Temporary Unavailability.  

---  

## Eight: Component Three: Querier  

### 8.1 Querier Function  

Querier is responsible for executing LogQL queries.  

When Grafana queries logs, the data is ultimately retrieved by Querier.  

Querier data sources generally include:  

    Latest Data from Ingester  
    Historical Chunks from Object Storage  
    Index Data  

### 8.2 Query Chain  

Typical Query:  

    {namespace="app-demo"} |= "ERROR"  

Chain:  

    Grafana Explore  
      ↓  
    Loki Query API  
      ↓  
    Query Frontend  
      ↓  
    Querier  
      ↓  
    Ingester / Object Storage / Index  
      ↓  
    Return Log Results  

### 8.3 What Querier Should Monitor  

Production Monitoring:  

    Slow Query  
    Query Timeout  
    Large Query Range  
    Query Scanning Too Many Streams  
    Query Hit Cache  
    Query Throttled  
    Query Consuming Excessive CPU / Memory  

Common Issues:  

    User Query {namespace=~".+"} |= "error"  
    Large Time Range  
    Too Broad Label Filtering  
    No Use of namespace / app / pod to Narrow Scope  
    Excessive Log Volume  
    Slow Backend Object Storage  
    Cache Not Configured or Missed  

---  

## Nine: Component Four: Query Frontend  

### 9.1 Query Frontend Function  

Query Frontend is the query entry optimization layer.  

It mainly handles:  

    Query Splitting  
    Query Queuing  
    Query Result Caching  
    Query Retry  
    Limit Concurrency  
    Improve Query Stability  

### 9.2 Why Need Query Frontend  

If a user queries 7 days of logs:  

    {namespace="prod"} |= "error"  

This query may be very heavy.  

Query Frontend can split large queries into smaller ones.  

Example:  

    7-day Query  
      ↓  
    Split into Multiple Small Queries by Time  
      ↓  
    Multiple Queriers Execute in Parallel  
      ↓  
    Merge Results and Return  

This improves query efficiency and stability.  

### 9.3 What Query Frontend Should Monitor  

Monitor:  

    Query Queue Accumulation  
    Query Cache Hit  
    Frequent Query Timeout  
    Query Throttled  
    User Performing Large Range Fuzzy Queries  

---  

## Ten: Component Five: Query Scheduler  

### 10.1 Query Scheduler Function  

Query Scheduler is used to manage query scheduling between Query Frontend and Querier.  

It helps:  

    Manage Query Queue  
    Control Query Fairness  
    Avoid Large Queries Overloading Querier  
    Improve Multi-tenant Query Stability  

### 10.2 When to Pay Attention  

Small-scale learning environments generally don't need excessive attention to Query Scheduler.  

Pay attention in the following scenarios:  

    Multi-tenant  
    High Query Concurrency  
    Many Large Range Queries  
    Uneven Querier Pressure  
    Query Queue Accumulation  
    Need for Fair Scheduling  

---  

## Eleven: Component Six: Compactor  

### 11.1 Compactor Function  

Compactor is a background maintenance component.  

It mainly handles:  

    Index Compression  
    Index Merging  
    Execute Retention Deletion  
    Clean Up Expired Data  
    Maintain Storage Structure  

### 11.2 Why Compactor is Important  

The biggest issue in log systems is:  

    Data Continuously Grows.  

Without retention policies and cleanup mechanisms, logs will keep occupying storage.  

Compactor is related to retention policies.  

Production monitoring must focus on:  

    How Long Logs Are Retained  
    Whether Expired Data Is Deleted  
    Whether Compactor Is Running Normally  
    Whether Deletion Is Delayed  
    Whether Old Data in Object Storage Continuously Grows  

### 11.3 Common Compactor Issues  

    Compactor Not Running:  
        Expired Logs May Not Be Cleaned.  

    Retention Configuration Not Effective:  
        Log Storage Continuously Grows.  

    Insufficient Object Storage Permissions:  
        Unable to Delete Expired Objects.  

    Improper Compactor Multi-Replica Configuration:  
        May Cause Task Conflicts or Abnormalities.  

---  

## Twelve: Component Seven: Ruler  

### 12.1 Ruler Function  

Ruler is responsible for executing Loki log rules.  

Rule Types:  

    Alerting Rule  
    Recording Rule  

Log Alert Examples:  

    ERROR Logs Exceeding 20 in the Last 5 Minutes.  
    CUDA out of memory Appears in the Last 5 Minutes.  
    Timeout Logs Exceeding Threshold in the Last 10 Minutes.  
    Database Connection Failed for a Service Surges.  

### 12.2 Ruler and AlertManager  

After Ruler executes LogQL rules, it can send alerts to AlertManager.  

Chain:  

    Loki Ruler  
      ↓  
    AlertManager  
      ↓  
    Enterprise WeChat / DingTalk / Feishu / Email / Webhook  

### 12.3 What Ruler Should Monitor  

Monitor:  

    Whether Rules Load Successfully  
    Whether Rules Execution Fails  
    Rule Execution Duration  
    Whether AlertManager Address Is Correct  
    Whether Alerts Are Sent Successfully  
    Whether Rule Files Are Mounted Correctly  
    Whether Multi-tenant Rules Are Isolated  

---  

## Thirteen: Component Eight: Index Gateway  

### 13.1 Index Gateway Function  

Index Gateway is mainly used in microservices or split deployment modes to help Querier access indexes.  

It can reduce the pressure on components accessing index storage directly.

### 13.2 When to Pay Attention

In learning environments and single-node mode, it's typically not necessary to focus on these aspects.

In production, if using:

    Simple Scalable
    Microservices
    Large-scale object storage
    High query concurrency

You need to understand the role of Index Gateway.

---

## Fourteen, Component Nine: Gateway

### 14.1 Gateway Role

Gateway is typically the unified entry point for Loki.

In Helm deployments, the common implementation of Gateway is Nginx.

It is responsible for:

    Receiving external requests
    Forwarding write requests to write components
    Forwarding query requests to read components
    Exposing Loki API uniformly
    Simplifying client configuration

### 14.2 Significance of Gateway

For log collection agents, it's undesirable to configure multiple component addresses separately.

Ideal scenario:

    Alloy only writes one Loki Gateway address.

Example:

    http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push

Gateway then forwards requests to the corresponding backend based on the path.

### 14.3 What to Pay Attention to Gateway

Pay attention to:

    Whether the gateway is accessible
    Whether Nginx configuration is correct
    Whether Service is normal
    Whether there are 502 / 503 errors
    Whether TLS configuration is correct
    Whether authentication configuration is correct
    Whether path forwarding is correct

---

## Fifteen, Component Ten: Cache

### 15.1 Cache Role

Loki can use caching to improve query performance in production.

Common caches:

    Query result cache
    Chunk cache
    Index cache

Common implementations:

    Memcached
    Redis
    Other caching systems

### 15.2 What Problems Does Cache Solve

Mainly solves:

    Reducing duplicate query pressure
    Lowering object storage read pressure
    Improving query response speed
    Alleviating pressure from large-scale queries

### 15.3 Is Cache Needed in Learning Phase

Single-node learning phase can skip cache configuration initially.

In production, whether to enable cache depends on:

    Query volume
    Log volume
    Object storage performance
    Query latency
    User count

---

## Sixteen, Component Eleven: Object Storage

### 16.1 Object Storage Role

Object storage is used to store long-term log data for Loki.

Common object storages:

    MinIO
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS
    Huawei Cloud OBS
    GCS
    Azure Blob

This series of experiments uses:

    MinIO

### 16.2 Why Object Storage is Needed

Production log systems are not recommended to rely solely on local disks long-term.

Reasons:

    Pods will be restarted
    Nodes may fail
    Local disk capacity is limited
    Weak high availability
    Limited scalability
    Not conducive to multi-replica queries
    Not conducive to long-term storage

Object storage is suitable for storing:

    Chunks
    Index
    Ruler rules
    Historical log data

### 16.3 Comparison Between Local Storage and Object Storage

Local storage:

    Suitable for learning.
    Simple configuration.
    Low cost.
    Not suitable for production high availability.

Object storage:

    Suitable for production.
    Good scalability.
    Large capacity.
    Suitable for multi-replica and high availability.
    Slightly complex configuration.

---

## Seventeen, Loki Write Pipeline

### 17.1 Write Pipeline Diagram

    Pod stdout / stderr
      ↓
    /var/log/containers/*.log
      ↓
    Grafana Alloy
      ↓
    Loki Gateway
      ↓
    Distributor
      ↓
    Ingester
      ↓
    WAL
      ↓
    Chunk
      ↓
    Object Storage / Filesystem

### 17.2 Explanation of Write Process

Step 1: Application outputs logs

    Applications output logs to stdout / stderr.

Step 2: Container runtime writes to files

    containerd writes logs to the node's local log path.

Step 3: Alloy collects logs

    Alloy runs as a DaemonSet on each node.
    It reads /var/log/containers/*.log.
    It adds namespace, pod, container, node, etc. labels.
    It pushes logs to Loki.

Step 4: Distributor receives requests

    Distributor receives /loki/api/v1/push requests.
    Validates labels, timestamps, tenants, and rate limits.

Step 5: Ingester writes logs

    Ingester organizes logs by stream.
    Writes to WAL.
    Builds chunk.
    Subsequently flushes to storage.

Step 6: Object storage saves data

    Chunk and index data eventually enter the filesystem or object storage.

### 17.3 Common Fault Points in Write Pipeline

    Application does not output stdout / stderr
    /var/log/containers has no logs
    Alloy is not scheduled to the corresponding node
    Alloy is not mounted with the log directory
    Alloy configuration is incorrect
    Loki Gateway is unreachable
    Distributor rejects writes
    Ingester is unhealthy
    Object storage is not writable
    Label limit exceeded
    Log line is too large
    Write rate limit exceeded

---

## Eighteen, Loki Query Pipeline

### 18.1 Query Pipeline Diagram

    Grafana Explore
      ↓
    Loki Query API
      ↓
    Gateway
      ↓
    Query Frontend
      ↓
    Querier
      ↓
    Ingester's recent data
      +
    Object Storage's historical data
      ↓
    Return results
      ↓
    Grafana display

### 18.2 Explanation of Query Process

Step 1: User inputs LogQL in Grafana

Example:

    {namespace="app-demo"} |= "ERROR"

Step 2: Grafana calls Loki Query API

Common APIs:

    /loki/api/v1/query
    /loki/api/v1/query_range

Step 3: Query Frontend processes the query

May perform:

    Query splitting
    Query caching
    Query queuing
    Query retries

Step 4: Querier executes the query

Querier finds relevant data based on labels and time range.

It will query:

    The latest data not yet flushed by the Ingester
    Historical chunks in object storage
    Index data

Step 5: Return log results

Grafana displays log lines, labels, and timestamps.

### 18.3 Common Fault Points in the Query Pipeline

    Grafana Loki datasource configuration error
    Loki Gateway unreachable
    Query Frontend queue backlog
    Querier resource insufficiency
    Query time range too large
    Too broad a label selector
    Slow object storage read
    Cache miss
    Query rejected by limits
    LogQL syntax error

---

## Nineteen, Practical Observation One: Viewing Loki Chart via Helm

Even without deploying Loki, you can first use Helm to observe the Loki Chart.

### 19.1 Add Helm Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 19.2 View Loki Chart Version

    helm search repo grafana/loki --versions

Note:

    It is recommended to fix the Chart version for both experiments and production.
    Avoid using uncontrolled latest versions directly.

### 19.3 Export Default Values

    helm show values grafana/loki > values-loki-default.yaml

View key fields:

    grep -n "deploymentMode" values-loki-default.yaml

    grep -n "singleBinary" values-loki-default.yaml

    grep -n "read:" values-loki-default.yaml

    grep -n "write:" values-loki-default.yaml

    grep -n "backend:" values-loki-default.yaml

    grep -n "gateway" values-loki-default.yaml

    grep -n "compactor" values-loki-default.yaml

    grep -n "ruler" values-loki-default.yaml

### 19.4 Generate Rendered YAML

Render without actual deployment:

    helm template loki grafana/loki \
      -n logging \
      > loki-rendered.yaml

View resources:

    grep -n "kind:" loki-rendered.yaml

View StatefulSet / Deployment:

    grep -n "StatefulSet" loki-rendered.yaml

    grep -n "Deployment" loki-rendered.yaml

View Service:

    grep -n "Service" loki-rendered.yaml

Note:

    helm template can help understand what resources the Chart will create.
    Actual deployment will be done in Chapter 04.

---

## Twenty, Practical Observation Two: Viewing Loki Kubernetes Resources

If Loki has already been deployed, you can execute the commands in this section.

### 20.1 View Pods

    kubectl get pods -n logging -o wide

In single-binary mode, you might see:

    loki-0

Or similar:

    loki-single-binary-0

In simple scalable mode, you might see:

    loki-read-xxx
    loki-write-xxx
    loki-backend-xxx
    loki-gateway-xxx

In microservices mode, you might see:

    distributor
    ingester
    querier
    compactor
    ruler
    query-frontend

The actual names depend on the Helm Chart and values configuration.

### 20.2 View Controllers

    kubectl get deploy,statefulset,daemonset -n logging

Focus on:

    Whether Loki is a Deployment or StatefulSet.
    Whether Gateway is a Deployment.
    Whether Alloy is a DaemonSet.
    Whether read/write/backend exist separately.

### 20.3 View Service

    kubectl get svc -n logging

Focus on:

    loki
    loki-gateway
    loki-headless
    loki-memberlist
    loki-canary

Service names vary by mode.

### 20.4 View Endpoint

    kubectl get endpoints -n logging

    kubectl get endpointslice -n logging

Used to determine:

    Whether a Service has backend Pods.
    Whether Gateway can forward to backend.
    Whether Loki is Ready.

### 20.5 View ConfigMap / Secret

    kubectl get cm -n logging

    kubectl get secret -n logging

View Loki configuration:

    kubectl get cm <loki-configmap-name> -n logging -o yaml

Note:

    Do not paste real Secrets in public notes.
    If the configuration contains access key / secret key, it needs to be desensitized.

---

## Twenty-one, Practical Observation Three: Viewing Loki Pod Logs

### 21.1 View Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=200

If there are multiple containers:

    kubectl logs <loki-pod-name> -n logging -c <container-name> --tail=200

For continuous viewing:

    kubectl logs <loki-pod-name> -n logging -f

### 21.2 Filter Keywords

kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|timeout|ring|ingester|distributor|querier|compactor|ruler"

### 21.3 Focus on Log Content

Focus on:

    Did Loki start successfully?
    Did the configuration load successfully?
    Is ring normal?
    Did storage initialize successfully?
    Did compactor start?
    Did ruler start?
    Did object storage connection fail?
    Did permission denied occur?
    Did 429 occur?
    Did query timeout occur?
    Did panic occur?

### 21.4 Common Abnormal Log Directions

Storage related:

    Object storage address error
    Bucket does not exist
    Access key error
    Secret key error
    TLS configuration error

Ring related:

    Memberlist anomaly
    Node cannot join ring
    DNS resolution anomaly

Query related:

    Query timeout
    Query range too large
    Object storage read slow

Write related:

    Ingestion rate limit
    Label limit exceeded
    Too many streams
    Log line too large

---

## Twenty-two, Practical Observation Four: Checking Status via Loki API

### 22.1 Port Forwarding

First check the Service:

    kubectl get svc -n logging

Port forwarding example:

    kubectl port-forward svc/<loki-service-name> 3100:3100 -n logging

If exposed via Gateway:

    kubectl port-forward svc/<loki-gateway-service-name> 3100:80 -n logging

### 22.2 Check Ready Status

    curl -s http://127.0.0.1:3100/ready

Normal return is typically similar to:

    ready

If not ready, need to check:

    Is Loki Pod Ready?
    Is the configuration correct?
    Is ring normal?
    Is storage normal?

### 22.3 Check Metrics

    curl -s http://127.0.0.1:3100/metrics | head

Filter Loki metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head

Filter Go runtime metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^go_" | head

Filter process metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^process_" | head

Note:

    /metrics is the entry point for Prometheus to scrape Loki's own monitoring metrics.
    In production environments, Loki itself should also be monitored by Prometheus.

### 22.4 Check Ring

Some deployment modes can access:

    curl -s http://127.0.0.1:3100/ring

Or open in browser:

    http://127.0.0.1:3100/ring

Ring is used to observe:

    Ingester members
    Distributor members
    Compactor members
    Ruler members
    Whether components are ACTIVE
    Whether components are abnormal

Note:

    Different versions, deployment modes, and exposure paths may vary.
    If /ring is unavailable, first confirm Loki's current mode and configuration.

### 22.5 Check Config

Some environments can access:

    curl -s http://127.0.0.1:3100/config

Note:

    /config may expose running configuration.
    If the configuration contains sensitive information, do not expose it externally.
    Do not copy production /config verbatim to public documentation.
    Production environments should restrict access.

### 22.6 Check Service List

Some versions may support:

    curl -s http://127.0.0.1:3100/services

If returns service status, it can be used to observe whether internal modules are Running.

If unavailable, it doesn't necessarily mean Loki is abnormal; need to confirm with current version.

---

## Twenty-three, Practical Observation Five: Querying Loki Labels

After deployment and integration with Alloy, you can query labels via API.

### 23.1 Query Label Names

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels"

Expected may include:

    namespace
    pod
    container
    node
    app

### 23.2 Query Values of a Specific Label

Query namespace:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values"

Query pod:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values"

### 23.3 Observing Significance

If namespace / pod / container does not exist, it indicates:

    Alloy may not have correctly added Kubernetes metadata.
    The collection configuration may be incomplete.
    The tenant or time range queried may be incorrect.
    Loki may not have written logs yet.
    The label name may differ from expectations.

---

## Twenty-four, Practical Observation Six: Executing Basic Query API

### 24.1 query_range Query

Example:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10'

### 24.2 Query ERROR

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"} |= "ERROR"' \
      --data-urlencode 'limit=10'

### 24.3 Observing Return Results

Can be combined with jq:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

If no results, check:

    Does app-demo have logs?
    Did Alloy collect successfully?
    Is the query time range correct?
    Does the label exist?
    Did Loki write successfully?

---

## Twenty-five, Loki Component and Kubernetes Resource Correspondence

### 25.1 Single-node Mode

Possible resources: /think

StatefulSet:
    loki

Service:
    loki
    loki-headless
    loki-memberlist

ConfigMap:
    loki

Secret:
    loki storage credentials

Note:

    In single-node mode, a single Pod may take on multiple component responsibilities.
    You cannot directly see independent components like distributor, ingester, querier, etc. through Pod names.

### 25.2 Simple Scalable Mode

Possible Resources:

    StatefulSet:
        loki-write
        loki-backend

    Deployment:
        loki-read
        loki-gateway

    Service:
        loki-gateway
        loki-read
        loki-write
        loki-backend

Note:

    read handles queries.
    write handles writes.
    backend handles background tasks.
    gateway handles unified entry point.

### 25.3 Microservices Mode

Possible Resources:

    Deployment / StatefulSet:
        distributor
        ingester
        querier
        query-frontend
        query-scheduler
        compactor
        ruler
        index-gateway
        gateway

Note:

    Each component runs independently.
    Suitable for large-scale production.
    Troubleshooting requires locating issues by component.

---

## Twenty-six, Observing Loki's Own Monitoring Metrics

As a logging system, Loki itself must also be monitored.

### 26.1 Viewing Loki Metrics

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

### 26.2 Common Metric Categories

Write-related:

    distributor request count
    distributor write failure count
    ingester log received count
    ingester active streams
    ingester flush count
    ingester flush failure

Query-related:

    querier query count
    query frontend query queue
    query latency
    query timeout
    cache hit / miss

Storage-related:

    chunk write
    chunk read
    compactor execution
    retention deletion
    object storage error

Rule-related:

    ruler rule execution
    ruler alert sending
    ruler execution failure

### 26.3 Production Must Monitor Loki Itself

At least monitor:

    Loki Pod readiness
    Loki write failure
    Loki query timeout
    Loki 429 increase
    Ingester stream count
    Ingester memory usage
    Compactor status
    Ruler status
    Object storage access error
    Gateway 5xx
    Loki disk / PVC usage
    Loki object storage capacity

---

## Twenty-seven, Relationship Between Loki and Alloy

### 27.1 Alloy's Position

Alloy belongs to the collection side in Loki architecture, not part of Loki service components.

Pipeline:

    Pod logs
      ↓
    Alloy
      ↓
    Loki

Alloy handles:

    Pod discovery
    Log file reading
    Tag addition
    Log forwarding
    Relabel processing
    Collection scope control

Loki handles:

    Log reception
    Log storage
    Log querying
    Log alerts

### 27.2 Observing Alloy

After deploying Alloy, observe:

    kubectl get ds -n logging

    kubectl get pods -n logging -o wide

    kubectl logs <alloy-pod> -n logging

Check:

    Whether Alloy has a Pod on each node
    Whether Alloy mounts /var/log/containers
    Whether Alloy can connect to Loki
    Whether Alloy outputs send failure errors
    Whether Alloy correctly supplements namespace / pod / container tags

---

## Twenty-eight, Relationship Between Loki and Grafana

### 28.1 Grafana is the Query Entry Point

Grafana queries logs through the Loki Data Source.

When users input:

    {namespace="app-demo"} |= "error"

in Grafana Explore, Grafana calls the Loki API to retrieve logs and display them.

### 28.2 Grafana is Not a Log Storage

Grafana does not handle log storage.

Grafana is only a display and query entry point.

The actual log storage is:

    Loki

Or other data sources:

    Elasticsearch
    OpenSearch

### 28.3 Grafana Dashboard Integration

You can combine Prometheus metrics and Loki logs in a single Dashboard.

Example:

    Pod CPU usage
    Pod memory usage
    Pod restart count
    Pod ERROR logs
    Pod timeout logs
    Pod latest logs

This allows direct entry into log context from metrics during troubleshooting.

---

## Twenty-nine, Relationship Between Loki and MinIO

### 29.1 MinIO's Role

MinIO is an S3-compatible object storage.

In this series, it's used to simulate production object storage.

Loki can write log chunks and index-related data to MinIO.

Pipeline:

    Loki Ingester
      ↓
    Chunk / Index
      ↓
    MinIO Bucket

### 29.2 Why Not Use Local Disks as the Final Solution

Local disks are suitable for learning.

But production has issues: /think

Pod Rebuild Risks
Node Failure Risk
Limited Capacity
Difficult to Scale
Multi-Replica Sharing Difficulty
Backup Difficulty

Object storage is more suitable for production.

### 29.3 Subsequent Experiments

The 05th chapter will implement:

    Deploy MinIO
    Create loki bucket
    Create access key
    Configure Loki to use MinIO
    Verify Loki data writing to MinIO

---

## 30. Architecture Observation Common Misconceptions

### 30.1 Misconception 1: Assuming there is no Distributor because you can't see the Distributor Pod

Error.

In monolithic mode, Distributor may run in the same Loki process.

There may not be an independent Pod.

### 30.2 Misconception 2: Gateway is the Loki itself

Error.

Gateway is usually just an entry proxy.

The backend Loki components handle writing and querying.

### 30.3 Misconception 3: Alloy is a Loki server component

Error.

Alloy is a log collection agent.

It is responsible for collecting logs and sending them to Loki.

### 30.4 Misconception 4: Grafana saves logs

Error.

Grafana does not save logs.

Grafana is only an entry for querying and displaying.

### 30.5 Misconception 5: Loki definitely needs all microservice components

Error.

Monolithic mode can be used in learning environments.

Split mode can be used in small to medium environments.

Only large-scale production needs the complete microservice mode.

### 30.6 Misconception 6: Slow log queries definitely means Loki is broken

Not necessarily.

Possible reasons:

    Query range too large
    Label filtering too broad
    Too much log volume
    Object storage slow
    Insufficient caching
    Unreasonable user query methods
    High cardinality label design error

---

## 31. Troubleshooting by Component

### 31.1 Write Failure

Prioritize checking:

    Alloy
    Gateway
    Distributor
    Ingester
    Object Storage

Commands:

    kubectl logs <alloy-pod> -n logging

    kubectl logs <loki-pod> -n logging

    curl -s http://127.0.0.1:3100/ready

    curl -s http://127.0.0.1:3100/metrics | grep -i error

Possible causes:

    Alloy configuration error
    Loki address error
    Distributor rejecting writes
    Ingester unhealthy
    Object storage not writable
    Label limit exceeded
    Write throttling

### 31.2 Query Failure

Prioritize checking:

    Grafana Data Source
    Gateway
    Query Frontend
    Querier
    Ingester
    Object Storage

Possible causes:

    Grafana data source address error
    Loki Query API unreachable
    Query range too large
    LogQL syntax error
    Querier resource insufficient
    Object storage read failure

### 31.3 Logs Expired but Not Deleted

Prioritize checking:

    retention configuration
    compactor
    object storage permissions

Possible causes:

    retention not enabled
    compactor not running
    compactor lacks delete permissions
    configuration not effective
    multi-replica compactor configuration error

### 31.4 Alerts Not Triggered

Prioritize checking:

    ruler enabled status
    rule file loading
    LogQL correctness
    AlertManager address
    Ruler logs for errors

Possible causes:

    Ruler not enabled
    Rule file path error
    Rule syntax error
    Query returns no data
    AlertManager unreachable

---

## 32. Hands-on Tasks

This chapter's hands-on tasks are divided into two categories.

### 32.1 When Loki is Not Deployed

Complete:

    [ ] Add Grafana Helm repository
    [ ] helm search repo grafana/loki --versions
    [ ] helm show values grafana/loki > values-loki-default.yaml
    [ ] helm template loki grafana/loki -n logging > loki-rendered.yaml
    [ ] Check deploymentMode in values
    [ ] Check StatefulSet / Deployment / Service in rendered YAML
    [ ] Understand resource differences between monolithic and split modes

Commands summary:

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

    helm search repo grafana/loki --versions

    helm show values grafana/loki > values-loki-default.yaml

    helm template loki grafana/loki \
      -n logging \
      > loki-rendered.yaml

    grep -n "deploymentMode" values-loki-default.yaml

    grep -n "kind:" loki-rendered.yaml

### 32.2 When Loki is Already Deployed

Complete:

    [ ] View Loki Pod
    [ ] View Loki Service
    [ ] View Loki StatefulSet / Deployment
    [ ] View Loki ConfigMap
    [ ] View Loki logs
    [ ] Port-forward Loki
    [ ] Access /ready
    [ ] Access /metrics
    [ ] Try accessing /ring
    [ ] Query labels
    [ ] Execute a query_range

Commands summary:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl get deploy,statefulset -n logging

    kubectl logs <loki-pod-name> -n logging --tail=200

kubectl port-forward svc/<loki-service-name> 3100:3100 -n logging

curl -s http://127.0.0.1:3100/ready

curl -s http://127.0.0.1:3100/metrics | head

curl -s http://127.0.0.1:3100/loki/api/v1/labels

---

## 33. Acceptance Checklist

After completing this document, you should confirm:

    [ ] Can explain the role of Distributor
    [ ] Can explain the role of Ingester
    [ ] Can explain the role of Querier
    [ ] Can explain the role of Query Frontend
    [ ] Can explain the role of Compactor
    [ ] Can explain the role of Ruler
    [ ] Can explain the role of Gateway
    [ ] Can explain the role of Object Storage
    [ ] Can draw the Loki write pipeline
    [ ] Can draw the Loki query pipeline
    [ ] Know that components may not be independent Pods in single-node mode
    [ ] Know the meaning of read/write/backend in Simple Scalable mode
    [ ] Know that Microservices mode is suitable for large-scale production
    [ ] Can use kubectl to view Loki resources
    [ ] Can use logs to view Loki startup and error information
    [ ] Can use /ready to determine if Loki is available
    [ ] Can use /metrics to view Loki's own metrics
    [ ] Know that /config may contain sensitive information and should not be exposed publicly
    [ ] Know that Alloy is the collector, not a Loki server component
    [ ] Know that Grafana is the visualization entry point and does not store logs

---

## 34. Production Environment Understanding

### 34.1 The most important thing in production is layered tracing

Loki troubleshooting should not be viewed in a mixed manner.

Tracing should be divided by pipeline layers:

    Collection Layer:
        Alloy / Promtail / Fluent Bit

    Entry Layer:
        Gateway / Distributor

    Write Layer:
        Ingester / WAL / Chunk

    Storage Layer:
        Object Storage / Index

    Query Layer:
        Query Frontend / Querier / Cache

    Background Layer:
        Compactor / Ruler

    Visualization Layer:
        Grafana

    Notification Layer:
        AlertManager

### 34.2 Production deployment should not only check Pod Running status

Loki Pod Running status does not guarantee the log system is fully functional.

Also check:

    /ready
    /metrics
    Write success rate
    Query success rate
    Gateway 5xx errors
    Ingester active streams
    Compactor operation status
    Ruler execution status
    Object storage read/write capability
    Grafana query capability
    Alloy send success status

### 34.3 Production requires monitoring Loki itself

Loki itself should be monitored by Prometheus.

At least establish:

    Loki Overview Dashboard
    Loki Write Dashboard
    Loki Query Dashboard
    Loki Storage Dashboard
    Loki Ruler Dashboard
    Loki Compactor Dashboard
    Loki Gateway Dashboard
    Alloy Agent Dashboard

### 34.4 Production requires Runbook

Recommended preparation:

    LokiWriteFailed
    LokiQuerySlow
    LokiGateway5xx
    LokiIngesterHighMemory
    LokiCompactorFailed
    LokiRulerFailed
    LokiObjectStorageError
    AlloySendFailed
    PodLogsMissing

---

## 35. Summary

Loki is a modular log system.

It can run in three modes:

    Monolithic / Single Binary:
        Suitable for learning and small-scale environments.

    Simple Scalable:
        Split by read/write/backend logic, suitable for medium-scale production and transitional scenarios.

    Microservices:
        Fully split by components, suitable for large-scale platformized log systems.

Loki's write pipeline:

    Pod stdout / stderr
      ↓
    /var/log/containers
      ↓
    Alloy
      ↓
    Gateway
      ↓
    Distributor
      ↓
    Ingester
      ↓
    Chunk / WAL
      ↓
    Object Storage

Loki's query pipeline:

    Grafana
      ↓
    Gateway
      ↓
    Query Frontend
      ↓
    Querier
      ↓
    Ingester + Object Storage
      ↓
    Return logs

Core component responsibilities:

    Distributor:
        Write entry point, validation, rate limiting, distribution.

    Ingester:
        Receive logs, build chunks, write to WAL, flush to storage.

    Querier:
        Execute LogQL queries.

    Query Frontend:
        Query splitting, caching, queuing, retries.

    Compactor:
        Index compression, retention period, expired data cleanup.

    Ruler:
        Log alert rules and Recording Rules.

    Gateway:
        Unified entry point and routing.

    Object Storage:
        Long-term storage of log data.

The key to learning Loki architecture is not memorizing component names, but being able to determine during failures:

    Is it a collection issue?
    Is it a write issue?
    Is it a query issue?
    Is it a storage issue?
    Is it a rule issue?
    Is it a visualization issue?

Next article will enter:

    03-Loki Deployment Mode Comparison and Experiment Selection

Key comparison: /think

Monolithic Mode  
Simple Scalable Mode  
Microservices Mode  

And clearly explain why this series starts with monolithic mode, then gradually expands to object storage and split deployment.

---

## Reference Documentation

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/

- Loki Architecture:  
  https://grafana.com/docs/loki/latest/get-started/architecture/

- Loki Components:  
  https://grafana.com/docs/loki/latest/get-started/components/

- Loki Deployment Modes:  
  https://grafana.com/docs/loki/latest/get-started/deployment-modes/

- Install Grafana Loki with Helm:  
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Install the monolithic Helm chart:  
  https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/

- Install the simple scalable Helm chart:  
  https://grafana.com/docs/loki/latest/setup/install/helm/install-scalable/

- Loki Configuration:  
  https://grafana.com/docs/loki/latest/configure/

- Loki Hash Rings:  
  https://grafana.com/docs/loki/latest/get-started/hash-rings/

- Grafana Alloy Documentation:  
  https://grafana.com/docs/alloy/latest/

- Collect Kubernetes logs and forward them to Loki:  
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- Kubernetes Logging Architecture:  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Helm Documentation:  
  https://helm.sh/docs/