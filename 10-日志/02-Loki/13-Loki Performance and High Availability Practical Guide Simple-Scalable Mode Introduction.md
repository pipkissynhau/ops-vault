# 13-Loki Performance and High Availability: Simple-Scalable Mode Introduction

## Document Notes

This is the thirteenth article in the Loki specialized learning series, used to learn the basic deployment, component splitting, read/write paths, high availability concepts, performance scaling methods, and experimental troubleshooting methods of the Loki Simple Scalable mode.

Previously completed:

    01-Loki Basic Understanding and Experimental Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experimental Selection
    04-Loki Monolithic Mode Helm Deployment Practical
    05-Loki Object Storage Integration with MinIO Practical
    06-Grafana-Alloy Collection Kubernetes Pod Logs Practical
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practical: Namespace-Pod-Container Log Retrieval
    09-LogQL Advanced Query Practical: json-logfmt-regexp-unwrap
    10-Grafana Integration with Loki and Log Dashboard Practical
    11-Loki Log Alert Practical: Ruler and AlertManager Coordination
    12-Loki Production Governance Practical: Log Volume - Retention Period - Limiting - Security

The 04th article completed Loki's minimal deployment using Monolithic / single mode.
The 05th article integrated MinIO object storage.
The 06th article completed Alloy collection of Kubernetes Pod logs.
The 12th article learned about retention, limits_config, log volume governance, and security baseline.

This article begins to learn Loki's extended deployment mode:

    Simple Scalable Deployment

Also commonly abbreviated as:

    SSD

The core of the Simple Scalable mode is to split Loki into three roles based on responsibilities:

    read
    write
    backend

Where:

    write:
        Responsible for log writing path.

    read:
        Responsible for log query path.

    backend:
        Responsible for background tasks, such as compactor, ruler, index gateway, etc.

Important Notes:

    The Simple Scalable Deployment has been marked as deprecated by the official.
    The official migration document states the deprecation time TBD, but it will occur before Loki 4.0 release.
    The official also recommends using Microservices / Distributed mode for production environments.
    Therefore, this article is not treating Simple Scalable as a long-term production solution, but rather to understand Loki's read/write splitting, high availability, object storage dependency, scaling, and troubleshooting concepts.

This article focuses on answering the following questions:

- What is the Simple Scalable mode?
- What responsibilities do read / write / backend have?
- What are the differences between Simple Scalable and Monolithic?
- What are the differences between Simple Scalable and Microservices?
- Why does Simple Scalable require object storage?
- Why are write replicas usually more critical than read / backend?
- Why is Gateway important in Simple Scalable mode?
- How to deploy Simple Scalable mode using Helm?
- How to observe read/write/backend resources through helm template?
- How to verify if read/write/backend Pods are normal?
- How to verify Alloy writing to Loki Gateway?
- How to verify Grafana querying Loki Gateway?
- How to observe the writing and query paths?
- How to perform read/write replica scaling experiments?
- How to observe high availability after Pod failure?
- How to troubleshoot write unavailability, read query failure, backend anomalies?
- What are the limitations and deprecation risks of Simple Scalable in production?
- How to understand migrating from Simple Scalable to Microservices / Distributed?

---

## Tags

#Loki #Grafana #SimpleScalable #SSD #ReadWriteBackend #HighAvailable #PerformanceModified #Helm #Kubernetes #MinIO #ObjectStorage #SRE #LogSystem #Observation

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/13-Loki Performance and High Availability: Simple-Scalable Mode Introduction.md

---

## One, Experimental Objectives

After completing this article, you should be able to:

    1. Understand the positioning of the Simple Scalable mode.
    2. Understand the three roles of read / write / backend.
    3. Understand the role of Loki Gateway in Simple Scalable.
    4. Understand why Simple Scalable depends on object storage.
    5. Be able to write Simple Scalable deployment configuration through Helm values.
    6. Be able to observe generated Kubernetes resources through helm template.
    7. Be able to deploy Loki Simple Scalable mode.
    8. Be able to view read / write / backend Pods.
    9. Be able to verify Gateway, Service, Endpoint.
    10. Be able to verify Alloy writing to Gateway.
    11. Be able to query logs through Grafana / curl.
    12. Be able to simulate write replica scaling.
    13. Be able to simulate read replica scaling.
    14. Be able to simulate the impact of a Loki Pod failure.
    15. Understand the high availability boundaries of Simple Scalable mode.
    16. Understand the deprecation risks of Simple Scalable mode.
    17. Explain why Microservices / Distributed is more recommended for production environments.
    18. Be able to write the evolution plan from monolithic to split mode for Loki.

---

## Two, Experimental Environment

### 2.1 Kubernetes Cluster

Experimental nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Container runtime:

    containerd

Namespaces:

    logging
    monitoring
    minio
    app-demo

Deployed or planned components: /think

Loki Simple Scalable  
Loki Gateway  
Grafana Alloy  
Grafana  
MinIO  
AlertManager  
nginx-demo  
json-log-demo  

### 2.2 Prerequisites  

Recommended completion:  

    [ ] Kubernetes cluster is operational  
    [ ] Helm is available  
    [ ] MinIO is available  
    [ ] Grafana is available  
    [ ] Alloy configuration is understood  
    [ ] Loki single-node experiment is completed  
    [ ] Object storage configuration is understood  
    [ ] Basic LogQL queries are mastered  
    [ ] Loki limits / retention basics are understood  

Check cluster:  

    kubectl get nodes -o wide  

Check namespaces:  

    kubectl get ns  

Check Helm:  

    helm version  

Check MinIO:  

    kubectl get pods -n minio -o wide  

    kubectl get svc -n minio  

Check Grafana:  

    kubectl get pods -n monitoring -o wide  

---  

## Three, Simple Scalable Deployment Model Overview  

### 3.1 Review of Three Deployment Models  

Common Loki deployment models:  

    Monolithic / Single Binary  
    Simple Scalable  
    Microservices / Distributed  

### 3.2 Monolithic  

Features:  

    All major components run in a single Loki process.  
    Simple deployment.  
    Suitable for learning and small-scale environments.  
    Low resource usage, short troubleshooting chain.  

Typical resources:  

    loki-0  
    loki-gateway  

Suitable for:  

    Learning  
    Local experiments  
    Small-scale clusters  
    Low log volume environments  

Not suitable for:  

    Large-scale log platforms  
    Multi-team high-concurrency queries  
    High write throughput  
    Component-level independent scaling  

### 3.3 Simple Scalable  

Features:  

    Split into three roles: read / write / backend.  
    Easier to scale than the monolithic model.  
    Simpler than the microservices model.  
    Suitable for understanding Loki read/write splitting and object storage dependencies.  

Typical resources:  

    loki-read  
    loki-write  
    loki-backend  
    loki-gateway  

Suitable for:  

    Learning Loki's scalable architecture  
    Understanding read/write separation  
    Maintaining historical environments  
    Intermediate architecture understanding  
    Transitioning from monolithic to distributed  

Not recommended as a long-term new production final solution:  

    Because the official has marked Simple Scalable Deployment as deprecated.  
    New production environments should prioritize evaluating Microservices / Distributed.  

### 3.4 Microservices / Distributed  

Features:  

    Loki components are fully split.  
    distributor, ingester, querier, query-frontend, compactor, ruler, index-gateway, etc., can be independently deployed and scaled.  
    Suitable for large-scale production.  

Typical components:  

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
    Multi-tenancy  
    High write throughput  
    High query concurrency  
    Scenarios requiring component-level independent scaling  

---  

## Four, Simple Scalable Architecture Overview  

### 4.1 Role Division  

Simple Scalable is mainly divided into:  

    read  
    write  
    backend  
    gateway  

### 4.2 Architecture Diagram  

    Alloy / Promtail / Fluent Bit  
      ↓  
    Loki Gateway  
      ↓  
    write  
      ├── Distributor  
      └── Ingester  
      ↓  
    Object Storage / MinIO  

    Grafana / LogQL / curl  
      ↓  
    Loki Gateway  
      ↓  
    read  
      ├── Query Frontend  
      └── Querier  
      ↓  
    Ingester + Object Storage / MinIO  

    backend  
      ├── Compactor  
      ├── Ruler  
      └── Index Gateway and other backend capabilities  
      ↓  
    Object Storage / MinIO  

### 4.3 Gateway's Role  

Gateway is a unified entry point.  

Write requests:  

    /loki/api/v1/push  
      ↓  
    Gateway  
      ↓  
    write  

Query requests:  

    /loki/api/v1/query  
    /loki/api/v1/query_range  
    /loki/api/v1/labels  
    /loki/api/v1/series  
      ↓  
    Gateway  
      ↓  
    read  

Rule requests:  

    /loki/api/v1/rules  
      ↓  
    Gateway  
      ↓  
    backend / ruler-related components  

Significance of Gateway:  

    Unified entry point.  
    Hides details of backend read/write/backend.  
    Allows Alloy and Grafana to not perceive Loki's internal splitting.  
    Can overlay Ingress, authentication, TLS, NetworkPolicy in front.  

---  

## Five, Detailed Explanation of read / write / backend Roles  

### 5.1 write Role  

write is mainly responsible for the write path.  

Logical responsibilities:  

    Receives log write requests  
    Validates labels and timestamps  
    Executes write throttling  
    Maintains active streams  
    Receives logs and constructs chunks  
    Writes to WAL  
    Flushes data to object storage  
    Participates in ring  

In Simple Scalable, write typically includes:  

    distributor  
    ingester

# Write Chain:

    Alloy
      ↓
    Gateway
      ↓
    write
      ↓
    WAL / Ingester Memory
      ↓
    MinIO / S3

Write Focus Metrics:

    Write Request Count
    Write Failure Count
    429 Rate Limiting
    active streams
    ingester memory
    chunk flush
    WAL Status
    Object Storage Write Errors

### 5.2 read Role

read primarily handles query paths.

Logical Responsibilities:

    Receive query requests
    Parse LogQL
    Query recent Ingester data
    Query historical data from object storage
    Merge query results
    Return to Grafana

In Simple Scalable, read typically includes:

    querier
    query frontend

Query Chain:

    Grafana
      ↓
    Gateway
      ↓
    read
      ↓
    Ingester + Object Storage
      ↓
    Return Results

read Focus Metrics:

    Query Request Count
    Query Failure Count
    Query Latency
    Query Timeout
    Query Concurrency
    Query Scope
    Object Storage Read Latency
    Grafana Dashboard Query Pressure

### 5.3 backend Role

backend primarily handles background tasks.

Logical Responsibilities:

    Compactor
    Ruler
    Retention
    Index Gateway
    Rule Execution
    Index Maintenance
    Background Maintenance Tasks

backend Focus Metrics:

    compactor health
    retention execution
    ruler rule loading
    ruler execution failure
    index gateway health
    object storage access
    background task backlog

### 5.4 Gateway Role

Gateway handles unified entry and path forwarding.

Focus Points:

    5xx
    4xx
    Correct forwarding
    read/write/backend Endpoint existence
    Nginx configuration
    Service exposure
    Grafana DataSource pointing to Gateway
    Alloy writing to Gateway

---

## Six. Why Simple Scalable Requires Object Storage

### 6.1 Monolithic Mode Can Use filesystem

In monolithic learning environments, you can use:

    filesystem
    PVC
    local directory

This approach is suitable for learning.

However, in split mode, if multiple write/read/backend replicas need to share data, they cannot rely on local directories in single Pod.

### 6.2 Object Storage is a Shared Backend

In Simple Scalable, multiple components need to access log data together.

Object storage provides:

    Shared access
    Capacity expansion
    Historical log retrieval
    chunk persistence
    index data storage
    multi-replica access capability

Common object storage:

    MinIO
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS
    Huawei Cloud OBS
    GCS
    Azure Blob

This article's experiment uses:

    MinIO

### 6.3 This Article Continues to Use MinIO

Bucket Planning:

    loki-chunks
    loki-ruler
    loki-admin

MinIO Service:

    minio.minio.svc.cluster.local:9000

Access Method:

    HTTP

Production Recommendations:

    HTTPS
    Dedicated user
    Minimal permissions
    Multi-node MinIO
    Erasure Coding
    Capacity monitoring
    API latency monitoring

---

## Seven. High Availability Strategy for Simple Scalable

### 7.1 High Availability Isn't Just Increasing Replica Count

High availability includes:

    Multiple replicas
    Cross-node distribution
    Reliable object storage
    Gateway availability
    Service Endpoint normality
    PodDisruptionBudget
    Resource requests/limits
    Anti-affinity
    Monitoring alerts
    Rollback capability

Simply increasing replicas doesn't guarantee true high availability.

### 7.2 write High Availability

write handles log writing.

In production, write needs to focus on:

    Replica count
    Data replication
    WAL
    Ingester ring
    Object storage writing
    Node distribution
    Pod restart recovery
    Write rate limiting

Example:

    write:
      replicas: 3

Common reasons:

    The write chain directly affects log loss more than the query chain.
    Multiple replicas can improve write path availability.
    replication_factor should match replica count.

### 7.3 read High Availability

read handles queries.

Impact of read unavailability:

    Grafana can't find logs.
    Dashboard is empty.
    Alert queries may be affected.
    Incident investigation capability decreases.

Example:

    read:
      replicas: 2

read scaling mainly improves:

    Query concurrency
    Query throughput
    Dashboard response
    Multi-user query capability

### 7.4 backend High Availability

backend handles background tasks.

backend anomalies may affect:

    retention
    compactor
    ruler
    index gateway
    rule execution
    background maintenance

Example:

    backend:
      replicas: 2

Whether multiple backend replicas are needed depends on the current version, component behavior, and production requirements.

### 7.5 Gateway High Availability

Gateway is the unified entry point.

If Gateway is a single point:

    Alloy writes may fail.
    Grafana queries may fail.

Production recommendations:

    gateway replicas >= 2
    With Service
    With Ingress / LoadBalancer
    With PodDisruptionBudget
    With anti-affinity

---

## Eight. Experiment Deployment Strategy

### 8.1 This Article's Experiment Goal

This article doesn't pursue a complete large-scale production architecture.

Experiment goal: /think

1. Deploy Simple Scalable.
2. Monitor read/write/backend resources.
3. Validate write path.
4. Validate query path.
5. Validate object storage.
6. Perform simple scaling experiment.
7. Perform simple failure experiment.
8. Understand production considerations.

### 8.2 Relationship with Previous Single-Instance Loki

If the current logging Namespace has already deployed a single-instance Loki, there are two options:

    Option 1:
        Uninstall the original Loki, then install Simple Scalable.

    Option 2:
        Use a new Release name and Namespace, such as loki-ssd / logging-ssd.

Learning recommendation:

    Recommended to use new Namespace:
        logging-ssd

Reasons:

    Not disrupt previous single-instance experiments.
    Facilitate comparison between two deployment modes.
    Can directly delete logging-ssd if issues occur.
    Reduce impact on existing Grafana / Alloy.

This article adopts:

    Namespace:
        logging-ssd

    Release:
        loki-ssd

### 8.3 Alloy and Grafana Connection Strategy

After deploying Simple Scalable, you can choose:

    Option 1:
        Temporarily use curl to manually push/query for verification.

    Option 2:
        Modify Alloy's write address to the new Gateway.

    Option 3:
        Add a new Loki-SSD Data Source in Grafana.

This article recommends:

    First verify with curl.
    Then add a new Loki-SSD data source in Grafana.
    Finally modify Alloy or temporarily deploy a test Alloy as needed.

---

## Nine. Prepare Namespace

### 9.1 Create logging-ssd Namespace

    kubectl create namespace logging-ssd

Check:

    kubectl get ns logging-ssd

### 9.2 Record Experiment Information

Recommended to record:

    Namespace:
        logging-ssd

    Helm Release:
        loki-ssd

    Chart:
        grafana-community/loki

    Deployment Mode:
        SimpleScalable

    Storage:
        MinIO

    Gateway:
        enabled

---

## Ten. Prepare Helm Repository and Chart Version

### 10.1 Add grafana-community Repository

    helm repo add grafana-community https://grafana-community.github.io/helm-charts

    helm repo update

### 10.2 Check Loki Chart Version

    helm search repo grafana-community/loki --versions | head -20

Record:

    CHART_VERSION=<actual version>

### 10.3 Export Default Values

    helm show values grafana-community/loki \
      --version <CHART_VERSION> \
      > values-loki-default.yaml

Search key fields:

    grep -n "deploymentMode" values-loki-default.yaml

    grep -n "SimpleScalable" values-loki-default.yaml

    grep -n "read:" values-loki-default.yaml

    grep -n "write:" values-loki-default.yaml

    grep -n "backend:" values-loki-default.yaml

    grep -n "gateway:" values-loki-default.yaml

    grep -n "minio:" values-loki-default.yaml

    grep -n "storage:" values-loki-default.yaml

### 10.4 Version Note

Loki Helm Chart fields change with versions.

Must use the current version:

    helm show values

As the reference.

Do not directly copy old article configurations to production.

---

## Eleven. MinIO Bucket Preparation

If the previous 05th article has already created the following Buckets, they can be reused directly:

    loki-chunks
    loki-ruler
    loki-admin

### 11.1 Enter mc Container

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

Check:

    mc ls local

### 11.2 Create Dedicated Bucket

To avoid mixing with single-instance experiments, this article recommends creating SSD-specific Buckets:

    mc mb local/loki-ssd-chunks

    mc mb local/loki-ssd-ruler

    mc mb local/loki-ssd-admin

Check:

    mc ls local

Exit:

    exit

### 11.3 Production Notes

In production environments, it's not recommended for all Loki instances to share the same set of Buckets unless explicitly planned.

Recommendations:

    Distinguish Buckets by environment
    Distinguish Buckets by cluster
    Plan Buckets by tenant or platform
    Avoid using default generic bucket names
    Avoid using MinIO root user
    Grant minimal permissions

---

## Twelve. Write Simple Scalable values

### 12.1 Create values File

Create:

    values-loki-ssd.yaml

### 12.2 Example Configuration

Note: /think

The following configuration is for experimental learning.
Do not directly apply it to production.
Fields must be based on the current Chart version helm show values.
If helm template reports an error, adjust the fields according to the current version.

Content:

    loki:
      auth_enabled: false

      commonConfig:
        replication_factor: 3

      schemaConfig:
        configs:
          - from: "2024-04-01"
            store: tsdb
            object_store: s3
            schema: v13
            index:
              prefix: loki_index_
              period: 24h

      ingester:
        chunk_encoding: snappy

      querier:
        max_concurrent: 4

      storage:
        type: s3
        bucketNames:
          chunks: loki-ssd-chunks
          ruler: loki-ssd-ruler
          admin: loki-ssd-admin
        s3:
          endpoint: minio.minio.svc.cluster.local:9000
          region: us-east-1
          accessKeyId: minioadmin
          secretAccessKey: minioadmin123
          s3ForcePathStyle: true
          insecure: true

      limits_config:
        allow_structured_metadata: true
        volume_enabled: true
        ingestion_rate_mb: 8
        ingestion_burst_size_mb: 16
        max_entries_limit_per_query: 5000
        reject_old_samples: true
        reject_old_samples_max_age: 168h

      ruler:
        enable_api: true
        alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093

    deploymentMode: SimpleScalable

    backend:
      replicas: 2

    read:
      replicas: 2

    write:
      replicas: 3
      persistence:
        enabled: true
        size: 10Gi

    gateway:
      enabled: true
      replicas: 2

    minio:
      enabled: false

    lokiCanary:
      enabled: false

    monitoring:
      selfMonitoring:
        enabled: false
      lokiCanary:
        enabled: false

    singleBinary:
      replicas: 0

    monolithic:
      replicas: 0

    ingester:
      replicas: 0
    querier:
      replicas: 0
    queryFrontend:
      replicas: 0
    queryScheduler:
      replicas: 0
    distributor:
      replicas: 0
    compactor:
      replicas: 0
    indexGateway:
      replicas: 0
    bloomPlanner:
      replicas: 0
    bloomBuilder:
      replicas: 0
    bloomGateway:
      replicas: 0

### 12.3 Configuration Notes

    deploymentMode: SimpleScalable

Enable Simple Scalable mode.

    write.replicas: 3

Deploy 3 replicas for the write path.

    read.replicas: 2

Deploy 2 replicas for the query path.

    backend.replicas: 2

Deploy 2 replicas for background tasks.

    gateway.enabled: true

Enable the unified entry point.

    gateway.replicas: 2

Gateway with 2 replicas to avoid single point of failure.

    commonConfig.replication_factor: 3

Write replication factor set to 3.

Note:

    replication_factor should match write replica count, production availability goals, and resource scale.
    In resource-constrained learning environments, temporarily reduce to 1 or 2, but understand this reduces high availability capabilities.

    minio.enabled: false

Do not use the built-in MinIO in the Chart, instead use the previously deployed standalone MinIO.

    storage.type: s3

Access MinIO via S3-compatible interface.

    s3ForcePathStyle: true

MinIO typically requires path-style access.

    insecure: true

Use HTTP in experimental environments.
Recommend HTTPS in production environments.

---

## Thirteen. Helm Template Rendering Check

### 13.1 Execute Rendering

    helm template loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml \
      > loki-ssd-rendered.yaml

### 13.2 View Resource Types /think

```
grep "^kind:" loki-ssd-rendered.yaml | sort | uniq -c

### 13.3 View read/write/backend

    grep -n "loki-ssd-read" loki-ssd-rendered.yaml

    grep -n "loki-ssd-write" loki-ssd-rendered.yaml

    grep -n "loki-ssd-backend" loki-ssd-rendered.yaml

    grep -n "loki-ssd-gateway" loki-ssd-rendered.yaml

### 13.4 View Workloads

    grep -n "kind: Deployment" loki-ssd-rendered.yaml

    grep -n "kind: StatefulSet" loki-ssd-rendered.yaml

### 13.5 View Service

    grep -n "kind: Service" loki-ssd-rendered.yaml

### 13.6 View MinIO Configuration

    grep -n "minio.minio.svc.cluster.local" loki-ssd-rendered.yaml

    grep -n "loki-ssd-chunks" loki-ssd-rendered.yaml

### 13.7 View Images

    grep -n "image:" loki-ssd-rendered.yaml | sort -u

### 13.8 Common Rendering Errors

#### deploymentMode is not supported

Resolution:

    grep -n "deploymentMode" values-loki-default.yaml

Confirm the supported values for the current Chart.

#### monolithic field does not exist

Resolution:

    The current Chart may use singleBinary.
    Remove the non-existent field or adjust according to default values.

#### monitoring.selfMonitoring field does not exist

Resolution:

    Adjust or remove the configuration based on the current Chart values.

#### storage field mismatch

Resolution:

    Check the official current version storage example.
    Use helm show values to search for storage and s3 fields.

---

## FourteenI don't know.Install Simple Scalable Loki

### 14.1 Installation

    helm install loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

### 14.2 View Helm Release

    helm list -n logging-ssd

Expected:

    loki-ssd    logging-ssd    deployed

### 14.3 View Pods

    kubectl get pods -n logging-ssd -o wide

Expected to see similar:

    loki-ssd-backend-0
    loki-ssd-backend-1
    loki-ssd-read-xxxxxxxx
    loki-ssd-read-yyyyyyyy
    loki-ssd-write-0
    loki-ssd-write-1
    loki-ssd-write-2
    loki-ssd-gateway-xxxx
    loki-ssd-gateway-yyyy

The exact names depend on the actual Chart version.

### 14.4 View Controllers

    kubectl get deploy,statefulset -n logging-ssd

Pay attention to:

    Whether read is a Deployment or StatefulSet
    Whether write is a StatefulSet or Deployment
    Whether backend is a StatefulSet or Deployment
    Whether gateway is a Deployment

Differences may exist across versions.

### 14.5 View Service

    kubectl get svc -n logging-ssd

Focus on:

    loki-ssd-gateway
    loki-ssd-read
    loki-ssd-write
    loki-ssd-backend
    loki-ssd-memberlist

### 14.6 View Endpoint

    kubectl get endpoints -n logging-ssd

Or:

    kubectl get endpointslice -n logging-ssd

Confirm:

    Gateway Endpoint is not empty
    Read Endpoint is not empty
    Write Endpoint is not empty
    Backend Endpoint is not empty

---

## FifteenI don't know.Basic Health Checks

### 15.1 Forward via Gateway Port

    kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

Note:

    Use local port 3101 to avoid conflict with the previous single Loki's port 3100.

### 15.2 Check ready

    curl -s http://127.0.0.1:3101/ready

Expected:

    ready

### 15.3 Check metrics

    curl -s http://127.0.0.1:3101/metrics | head

### 15.4 Check labels

    curl -s http://127.0.0.1:3101/loki/api/v1/labels | jq

There may be no business logs immediately after installation.

### 15.5 View Pod Logs

Check write:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=100

Check read:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=100

Check backend:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=100

Check gateway:
```

kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=100

If labels do not match, first check the actual labels:

    kubectl get pod -n logging-ssd --show-labels

---

## SixteenI don't know.Manual Write and Query Testing

### 16.1 Write Test Logs

Maintain port forwarding:

    kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

Execute in another terminal:

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"ssd-manual-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"ssd-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki simple scalable mode\"]
            ]
          }
        ]
      }"

If there is no output, it typically indicates success.

### 16.2 Query Test Logs

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-manual-test"}' \
      --data-urlencode 'limit=10' | jq

Expected to see:

    hello loki simple scalable mode

### 16.3 View Labels

    curl -s "http://127.0.0.1:3101/loki/api/v1/labels" | jq

View job:

    curl -s "http://127.0.0.1:3101/loki/api/v1/label/job/values" | jq

Expected to include:

    ssd-manual-test

---

## SeventeenI don't know.Verify Object Storage Write

### 17.1 Enter mc Container

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

Check:

    mc ls local/loki-ssd-chunks

Recursive check:

    mc find local/loki-ssd-chunks | head

Exit:

    exit

### 17.2 Why Objects Are Not Visible Immediately

When a log is written, objects may not appear immediately in MinIO.

Reasons:

    Logs first enter write / ingester.
    Chunks are flushed only after meeting conditions.
    Querying recent data may come from ingester memory.
    Object writing has latency.

Judgment criteria:

    Push succeeds
    Query succeeds
    Loki logs have no object storage errors
    Wait for some time until objects appear

### 17.3 Batch Write Test

    for i in $(seq 1 300); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"ssd-batch-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"ssd-test\"
              },
              \"values\": [
                [\"${TS}\", \"simple scalable batch log line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

Query:

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-batch-test"}' \
      --data-urlencode 'limit=20' | jq

---

## EighteenI don't know.Grafana Integration with Simple Scalable Loki

### 18.1 Add Data Source

Add a Loki data source in Grafana:

    Name:
        Loki-SSD

    URL:
        http://loki-ssd-gateway.logging-ssd.svc.cluster.local

    Access:
        Server / Proxy

Click:

    Save & test

### 18.2 Explore Query

Select:

    Loki-SSD

Query:

    {job="ssd-manual-test"}

    {job="ssd-batch-test"}

    {namespace="app-demo"}

### 18.3 If Connection Fails

Test from Grafana Pod:

    kubectl exec -it <grafana-pod-name> -n monitoring -- sh

Inside the container:

    wget -qO- http://loki-ssd-gateway.logging-ssd.svc.cluster.local/ready

If wget/curl is not available, use a temporary Pod:

    kubectl run curl-ssd-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n monitoring \
      -- sh

Inside the container:

    curl -s http://loki-ssd-gateway.logging-ssd.svc.cluster.local/ready

## 19. Alloy Writing to Simple Scalable Loki

### 19.1 Modify Write Address

If you want Alloy to write to a new SSD Loki, you need to modify the loki.write URL in the Alloy configuration.

Original address may be:

    http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push

Change to:

    http://loki-ssd-gateway.logging-ssd.svc.cluster.local/loki/api/v1/push

### 19.2 Modify values-alloy-loki.yaml

Find:

    loki.write "loki" {
      endpoint {
        url = "http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push"
      }
    }

Change to:

    loki.write "loki" {
      endpoint {
        url = "http://loki-ssd-gateway.logging-ssd.svc.cluster.local/loki/api/v1/push"
      }
    }

### 19.3 Helm Template Check

    helm template alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml \
      > alloy-ssd-rendered.yaml

Check:

    grep -n "loki-ssd-gateway" alloy-ssd-rendered.yaml

### 19.4 Upgrade Alloy

    helm upgrade alloy grafana/alloy \
      --namespace logging \
      --version <ALLOY_CHART_VERSION> \
      -f values-alloy-loki.yaml

Check:

    kubectl rollout status ds/alloy -n logging

    kubectl logs <alloy-pod-name> -n logging --tail=200 | grep -Ei "error|warn|loki|failed|429"

### 19.5 Generate Pod Logs

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Inside the container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    exit

### 19.6 Query

In Loki-SSD, query:

    {namespace="app-demo"}

    {namespace="app-demo", app="nginx-demo"}

    {namespace="app-demo", app="nginx-demo"} |= "404"

---

## 20. Observing Write Path

### 20.1 Write Path Review

    Alloy / curl
      ↓
    Gateway
      ↓
    write
      ↓
    WAL / Ingester
      ↓
    Object Storage

### 20.2 Check Gateway Logs

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=200

If label mismatch:

    kubectl get pod -n logging-ssd --show-labels

Specify pod:

    kubectl logs <gateway-pod-name> -n logging-ssd --tail=200

Observe:

    POST /loki/api/v1/push
    2xx
    4xx
    5xx

### 20.3 Check Write Logs

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=300

Filter:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=500 | grep -Ei "push|ingest|stream|chunk|flush|error|warn|s3|minio|rate|429"

### 20.4 Check Loki Metrics

Port forward:

    kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

Check:

    curl -s http://127.0.0.1:3101/metrics | grep "^loki_" | head -50

Focus on:

    distributor requests
    ingester stream
    chunk flush
    request duration
    rate limit

Specific metric names vary by version, refer to actual /metrics.

---

## 21. Observing Query Path

### 21.1 Query Path Review

    Grafana / curl
      ↓
    Gateway
      ↓
    read
      ↓
    write ingester + object storage
      ↓
    Return query results

### 21.2 Execute Query

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-batch-test"}' \
      --data-urlencode 'limit=50' | jq

### 21.3 Check Read Logs

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=300

Filter:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=500 | grep -Ei "query|querier|frontend|error|warn|timeout|s3|minio"

### 21.4 Check Gateway Query Logs

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=200 | grep -Ei "query|labels|series"

### 21.5 Query Stress Test Experiment

Execute 20 queries:

    for i in $(seq 1 20); do
      curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
        --data-urlencode 'query={job="ssd-batch-test"}' \
        --data-urlencode 'limit=20' >/dev/null &
    done

    wait

Observe read Pod resources:

    kubectl top pod -n logging-ssd

Skip this command if metrics-server is not available.

---

## Twenty-two, read Expansion Experiment

### 22.1 Check Current read Replicas

    kubectl get deploy,statefulset -n logging-ssd | grep read

If read is a Deployment:

    kubectl get deploy -n logging-ssd | grep read

If read is a StatefulSet:

    kubectl get statefulset -n logging-ssd | grep read

### 22.2 Use Helm to Modify read Replicas

Modify values-loki-ssd.yaml:

    read:
      replicas: 3

Execute:

    helm upgrade loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

Check:

    kubectl get pods -n logging-ssd -o wide | grep read

### 22.3 Validate Queries

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-batch-test"}' \
      --data-urlencode 'limit=20' | jq

### 22.4 Understand Expansion Effects

read expansion mainly improves:

    Query concurrency capability
    Multi-user query capability
    Dashboard query response capability

But read expansion cannot solve:

    Log write failures
    Object storage write slowness
    High cardinality label
    Unreasonable wide-range queries
    Gateway single point
    MinIO single point

---

## Twenty-three, write Expansion Experiment

### 23.1 Check Current write Replicas

    kubectl get pods -n logging-ssd -o wide | grep write

### 23.2 Modify write Replicas

Modify values-loki-ssd.yaml:

    write:
      replicas: 4

If replication_factor is still 3, it can generally remain unchanged.

Note:

    After increasing write replicas, pay attention to resources, ring, WAL, PVC.
    Expanding write in production requires more caution.

Upgrade:

    helm upgrade loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

Check:

    kubectl get pods -n logging-ssd -o wide | grep write

### 23.3 Validate Writes

    for i in $(seq 1 50); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"ssd-write-scale-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"ssd-write-test\"
              },
              \"values\": [
                [\"${TS}\", \"write scale test line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

Query:

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-write-scale-test"}' \
      --data-urlencode 'limit=20' | jq

### 23.4 Understand Expansion Effects

write expansion mainly improves:

    Write throughput
    Write availability
    Stream carrying capacity
    Ingester pressure distribution

But write expansion cannot solve:

    Application abnormal log flushing
    High cardinality label
    MinIO write slowness
    Object storage unavailable
    Low limits configuration
    Gateway single point

---

## Twenty-four, backend Expansion Experiment

### 24.1 Check backend

    kubectl get pods -n logging-ssd -o wide | grep backend

### 24.2 Modify backend Replicas

Modify:

    backend:
      replicas: 3

Execute:

    helm upgrade loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

Check:

    kubectl get pods -n logging-ssd -o wide | grep backend

### 24.3 Validate Ruler / Compactor

Check backend logs: /think

kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=500 | grep -Ei "ruler|compactor|retention|index|error|warn"

### 24.4 Understanding Expansion Effects

Backend expansion does not necessarily directly improve query or write performance.

It is primarily related to background tasks:

    compactor
    ruler
    index gateway
    retention

In production, backend expansion requires understanding whether specific components support multi-replica coordination, and the current Chart/version's operational mode.

---

## 25. Pod Failure Experiment: Deleting a read Pod

### 25.1 View read Pod

    kubectl get pod -n logging-ssd -o wide | grep read

### 25.2 Delete a read Pod

    kubectl delete pod <read-pod-name> -n logging-ssd

### 25.3 Observe Recovery

    kubectl get pod -n logging-ssd -w

### 25.4 Simultaneously Query Logs

In another terminal execute:

    for i in $(seq 1 20); do
      curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
        --data-urlencode 'query={job="ssd-batch-test"}' \
        --data-urlencode 'limit=5' >/dev/null
      echo "query $i done"
      sleep 1
    done

### 25.5 Observation Conclusion

If read replica count is greater than 1:

    Query should remain largely uninterrupted after deleting a read Pod.
    Brief fluctuations may occur.
    Service will remove abnormal Endpoints.
    New Pod will be recreated.

If read has only 1 replica:

    Queries may fail during deletion.

---

## 26. Pod Failure Experiment: Deleting a write Pod

### 26.1 View write Pod

    kubectl get pod -n logging-ssd -o wide | grep write

### 26.2 Delete a write Pod

    kubectl delete pod <write-pod-name> -n logging-ssd

### 26.3 Observe Recovery

    kubectl get pod -n logging-ssd -w

### 26.4 Simultaneously Write Logs

In another terminal:

    for i in $(seq 1 30); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"ssd-write-failover-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"ssd-ha-test\"
              },
              \"values\": [
                [\"${TS}\", \"write failover test line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      echo "push $i done"
      sleep 1
    done

Query:

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-write-failover-test"}' \
      --data-urlencode 'limit=50' | jq

### 26.5 Observation Conclusion

If write replicas are sufficient and replication_factor configuration is reasonable:

    Deleting a single write Pod should not make overall writing completely unavailable.
    Brief fluctuations may occur.
    Ring will stabilize after new Pod recovery.

If write replicas are insufficient:

    Writes may fail.
    replication_factor may not be met.
    Loki logs may show ring / quorum / unhealthy related errors.

---

## 27. Pod Failure Experiment: Deleting a gateway Pod

### 27.1 View gateway Pod

    kubectl get pod -n logging-ssd -o wide | grep gateway

### 27.2 Delete a gateway Pod

    kubectl delete pod <gateway-pod-name> -n logging-ssd

### 27.3 Simultaneously Check Readiness

    for i in $(seq 1 20); do
      curl -s http://127.0.0.1:3101/ready
      echo
      sleep 1
    done

### 27.4 Observation Conclusion

If gateway replicas >= 2:

    After deleting a gateway Pod, Service should still have other Endpoints.
    Queries and writes should not be completely interrupted.

If gateway has only 1 replica:

    Gateway becomes unavailable during deletion.
    Alloy writes and Grafana queries will be affected.

---

## 28. Node Distribution and Anti-Affinity

### 28.1 View Pod Distribution

    kubectl get pod -n logging-ssd -o wide

Observe:

    Which nodes read are distributed on
    Which nodes write are distributed on
    Which nodes backend are distributed on
    Which nodes gateway are distributed on

### 28.2 Why Need Dispersion

If 3 write Pods are on the same node:

    Node failure will affect the entire write path.

If 2 gateways are on the same node:

    Node failure will affect the unified entry point.

Production recommendation: /think

# Using podAntiAffinity  
# Using topologySpreadConstraints  
# Using PDB  
# Setting requests/limits  
# Avoiding all replicas concentrated on a single node  

### 28.3 topologySpreadConstraints Approach  

Example approach:  

    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: loki

Notes:  
    - Specific fields depend on whether Helm Chart values support them.  
    - Confirm via helm show values.  
    - Must test scheduling effects in production.  

---

## Twenty-Nine, Resource requests / limits  

### 29.1 Why configure resources  

Loki components are sensitive to CPU/memory.  

Not configuring resources may lead to:  
    - Pods scheduled to nodes with insufficient resources  
    - Memory spikes during queries  
    - Being OOMKilled during writes  
    - Resource contention on nodes  
    - HPA/scheduling strategies lacking basis  

### 29.2 Read resource focus  

Read primarily consumes:  
    - CPU  
    - Memory  
    - Object storage reads  
    - Query concurrency  

Example direction:  
    read:
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 2
          memory: 4Gi  

### 29.3 Write resource focus  

Write primarily consumes:  
    - Memory  
    - Disk WAL  
    - Network  
    - Object storage writes  
    - CPU  

Example direction:  
    write:
      resources:
        requests:
          cpu: 500m
          memory: 2Gi
        limits:
          cpu: 2
          memory: 6Gi  

### 29.4 Backend resource focus  

Backend primarily consumes:  
    - Compactor CPU/IO  
    - Ruler query resources  
    - Index gateway memory  
    - Object storage read/write  

Example direction:  
    backend:
      resources:
        requests:
          cpu: 300m
          memory: 1Gi
        limits:
          cpu: 1
          memory: 3Gi  

### 29.5 Notes  

The above are just directional examples.  

Production resources should be based on:  
    - Daily log volume  
    - Query concurrency  
    - Retention  
    - Label cardinality  
    - Stream count  
    - Object storage performance  
    - Dashboard count  
    - Ruler rule count  

Conduct stress tests and adjustments.  

---

## Thirty, Simple Scalable Monitoring Focus  

### 30.1 Write Monitoring  

Focus on:  
    - Write request volume  
    - Write failure count  
    - 429 rate limiting  
    - Active streams  
    - Ingester memory  
    - WAL size  
    - Chunk flush  
    - Object storage write errors  
    - Pod restarts  
    - Ring status  

### 30.2 Read Monitoring  

Focus on:  
    - Query request volume  
    - Query failure count  
    - Query latency  
    - Query timeouts  
    - Query frontend queue  
    - Querier CPU/memory  
    - Object storage read errors  
    - Dashboard query pressure  

### 30.3 Backend Monitoring  

Focus on:  
    - Compactor health  
    - Retention execution  
    - Ruler health  
    - Ruler rule evaluation failures  
    - Index gateway errors  
    - Object storage access errors  

### 30.4 Gateway Monitoring  

Focus on:  
    - 2xx/4xx/5xx status codes  
    - Request latency  
    - Upstream unavailable  
    - Empty endpoint  
    - Nginx reload  
    - Pod restarts  

### 30.5 Object Storage Monitoring  

Focus on:  
    - MinIO availability  
    - Bucket capacity  
    - S3 API 4xx/5xx  
    - Request latency  
    - Disk capacity  
    - Node status  
    - Certificate validity  

---

## Thirty-One, Common Fault: Write Unavailable  

### 31.1 Symptoms  

    Alloy failed to send batch  
    Push returns 500  
    Push returns 429  
    Loki Gateway returns 502/503  
    Write Pod CrashLoopBackOff  
    Write Endpoint is empty  

### 31.2 Troubleshooting  

Check Pod:  
    kubectl get pod -n logging-ssd -o wide | grep write  

Check Service/Endpoint:  
    kubectl get svc -n logging-ssd | grep write  
    kubectl get endpoints -n logging-ssd | grep write  

Check logs:  
    kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=500 | grep -Ei "error|warn|ring|ingester|s3|minio|rate|429|flush|wal"  

Check Gateway:  
    kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=300 | grep -Ei "push|502|503|500"  

### 31.3 Common Causes

write Not enough copies  
replication_factor Too high.  
MinIO Unattainable.  
Bucket does not exist  
S3 Document error  
WAL PVC Unusual  
Inadequate resources OOM  
High base figure stream Too much.  
Writing limit stream  
Gateway Forward error  

### 31.4 Processing  

    1. Restore write PodI don't know.  
    2. Inspection MinIOI don't know.  
    3. Inspection bucket And authority.  
    4. Inspection replication_factorI don't know.  
    5. Inspection limits_configI don't know.  
    6. Check the source of the bulge.  
    7. Increase if necessary writeI don't know.  
    8. Process high base figure label Or repeat collections.  

---  

## XXII. Common failures II:read Query failed  

### 32.1 phenomena  

    Grafana Query failed  
    Explore Wrong.  
    query_range Back 500  
    Gateway 502 / 503  
    read Pod CrashLoopBackOff  
    Query timeout  

### 32.2 Check.  

View read Pod:  

    kubectl get pod -n logging-ssd -o wide | grep read  

View read Log:  

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=500 | grep -Ei "error|warn|query|timeout|s3|minio|frontend|querier"  

Check object storage:  

    kubectl get pod -n minio  

    kubectl logs <minio-pod-name> -n minio --tail=200  

Test query:  

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \  
      --data-urlencode 'query={job="ssd-batch-test"}' \  
      --data-urlencode 'limit=10' | jq  

### 32.3 Common causes  

    read Inadequate resources  
    It's too extensive.  
    LogQL Use wide range regexp/json  
    Object Storage Reading Slow  
    MinIO Not Available  
    High base figure leads to high search pressure  
    Gateway Transfer abnormal.  
    Grafana Dashboard Too many panels.  

### 32.4 Processing  

    1. Reduces the query range.  
    2. Optimization LogQLI don't know.  
    3. Adjustment Dashboard Default time.  
    4. Expansion readI don't know.  
    5. Optimizes the storage of objects.  
    6. Adds a query limit.  
    7. Consideration of hot spot queries recording ruleI don't know.  
    8. Large-scale scenarios Distributed Mode and cache.  

---  

## XXXIII. Common failures III:backend Unusual  

### 33.1 phenomena  

    retention Not effective  
    Ruler Not carried out.  
    Compactor Wrong.  
    rules API Unusual  
    index gateway Wrong.  

### 33.2 Check.  

View backend:  

    kubectl get pod -n logging-ssd -o wide | grep backend  

View log:  

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=500 | grep -Ei "compactor|ruler|retention|rule|index|error|warn|s3|minio"  

Inspection Ruler API:  

    curl -s http://127.0.0.1:3101/loki/api/v1/rules | jq  

Inspection retention Configure:  

    helm get values loki-ssd -n logging-ssd -a | grep -n "retention" -A 50  

### 33.3 Common causes  

    backend Pod Not Available  
    Ruler Not enabled  
    rules bucket does not exist  
    compactor Configure Error  
    delete_request_store Configure Error  
    Object storage permissions are insufficient  
    AlertManager URL Error  
    backend Inadequate resources  

### 33.4 Processing  

    1. Restore backend PodI don't know.  
    2. Inspection MinIO bucketI don't know.  
    3. Inspection Ruler Configure.  
    4. Inspection AlertManagerI don't know.  
    5. Inspection Compactor retention Configure.  
    6. Checks the object storage privileges.  
    7. Adjustments if necessary backend Resources.  

---  

## XXXIV. Common malfunctions IV:Gateway 502 / 503  

### 34.1 phenomena  

    Grafana Save & test Failed  
    curl /ready Failed  
    push Back 502  
    query Back 503  
    Gateway Log upstream unavailable  

### 34.2 Check.  

View Gateway:  

    kubectl get pod -n logging-ssd -o wide | grep gateway  

View Gateway Log:  

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=500  

View Service / Endpoint:  

    kubectl get svc -n logging-ssd  

    kubectl get endpoints -n logging-ssd  

Confirm. read/write/backend Is there any? Endpoint:  

    kubectl get endpoints -n logging-ssd | grep -E "read|write|backend"  

### 34.3 Common causes  

    read Pod Not Available  
    write Pod Not Available  
    backend Pod Not Available  
    Service selector Do not match  
    Endpoint Empty  
    Gateway Configure Generation Error  
    Chart values Configuration does not match  
    NetworkPolicy Block  

### 34.4 Processing

1. Recover backend Pod.
2. Check Endpoint.
3. Check Gateway configuration.
4. Check NetworkPolicy.
5. Check Helm values.
6. Rollback if necessary.

---

## 35. Simple Scalable Deprecation Risk

### 35.1 Current Status

Simple Scalable Deployment is currently marked as deprecated by the official team.

Official migration documentation states:

    Deprecation time TBD.
    Will happen before Loki 4.0 release.
    Environments using SSD should plan migration to Distributed/Microservices.

### 35.2 Impact on Learning

It's still worth understanding for learning purposes.

Reasons:

    The idea of splitting read/write/backend is important.
    Old environments may still use SSD.
    Helps understand Loki read/write paths.
    Helps understand object storage and high availability.
    Helps transition to Microservices.

### 35.3 Impact on New Production Environments

New production environments should not directly target SSD as a long-term solution.

Recommendations:

    Small scale:
        Monolithic HA + object storage, evaluate based on version and official recommendations.

    Large scale:
        Microservices/Distributed.

    Existing SSD:
        Develop a migration plan.

### 35.4 Recommendations for Existing SSD Environments

Existing SSD environments should:

    Fix current Loki version.
    Follow Loki 4.0 roadmap.
    Read official migration documentation.
    Perform migration drills.
    Evaluate if resources can run SSD and Distributed simultaneously.
    Confirm object storage doesn't need migration or only minor adjustments.
    Design rollback plan.
    Avoid forced migration near version deprecation.

---

## 36. Understanding Migration from Simple Scalable to Distributed

### 36.1 Changes from SSD to Distributed

Simple Scalable:

    read
    write
    backend

Distributed:

    distributor
    ingester
    querier
    query-frontend
    query-scheduler
    compactor
    ruler
    index-gateway
    gateway
    cache

### 36.2 Conceptual Changes

SSD groups by major categories.

Distributed splits by components.

SSD:

    Simplified deployment.
    Three-tier read/write/backend.

Distributed:

    Fine-grained scaling.
    Component-level tuning.
    More suitable for large-scale production.

### 36.3 Core Migration Focus

Focus on:

    Helm values differences
    deploymentMode changes
    Pod resource increases
    Service changes
    Gateway routing changes
    Shared object storage
    Ruler rule migration
    Compactor configuration migration
    limits_config consistency
    retention consistency
    Grafana DataSource changes
    Alloy write address changes
    Rollback plan

### 36.4 Pre-Migration Preparation

Prepare:

    1. Backup Helm values.
    2. Backup current rules.
    3. Record current Loki version.
    4. Record current Chart version.
    5. Record current Bucket.
    6. Evaluate cluster resources.
    7. Practice in test environment.
    8. Validate write and query.
    9. Validate Grafana.
    10. Validate Ruler.
    11. Validate retention.
    12. Design rollback plan.

---

## 37. Production Notes for Simple Scalable

### 37.1 Must Use Object Storage

Production should not use local filesystem.

Recommended:

    S3 / OSS / COS / OBS / MinIO

### 37.2 Must Configure Limits

At least configure:

    ingestion_rate_mb
    ingestion_burst_size_mb
    per_stream_rate_limit
    max_streams_per_user
    max_entries_limit_per_query
    max_query_series
    max_query_length
    reject_old_samples

### 37.3 Must Configure Retention

Need:

    compactor
    retention_enabled
    retention_period
    retention_stream
    delete_request_store

### 37.4 Must Monitor Self

Monitor:

    read
    write
    backend
    gateway
    object storage
    Alloy
    Grafana
    AlertManager

### 37.5 Must Control Queries

Avoid:

    Default All
    Last 7 days
    Global json
    Global regexp
    Too many dashboard panels
    Unrestricted Explore

### 37.6 Must Govern Labels

Prohibit:

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message

As Loki Label.

### 37.7 Must Plan Deprecation Migration

Since SSD has deprecation risk, production must:

    Record version
    Follow Loki 4.0 roadmap
    Reserve migration window
    Learn Distributed
    Design migration plan

---

## 38. Hands-on Tasks

### 38.1 Task 1: Create Namespace

Execute:

    kubectl create namespace logging-ssd

Acceptance:

    [ ] logging-ssd Namespace exists

### 38.2 Task 2: Create MinIO Bucket

Execute: /think

kubectl run minio-mc \
  --rm -it \
  --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  -n minio \
  -- sh

Container:

  mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

  mc mb local/loki-ssd-chunks

  mc mb local/loki-ssd-ruler

  mc mb local/loki-ssd-admin

  mc ls local

Acceptance:

  [ ] loki-ssd-chunks has been created
  [ ] loki-ssd-ruler has been created
  [ ] loki-ssd-admin has been created

### 38.3 Task Three: Writing values-loki-ssd.yaml

Acceptance:

  [ ] deploymentMode is SimpleScalable
  [ ] read replicas have been configured
  [ ] write replicas have been configured
  [ ] backend replicas have been configured
  [ ] gateway is enabled
  [ ] storage uses MinIO / S3
  [ ] replication_factor is reasonable
  [ ] minio.enabled=false

### 38.4 Task Four: Helm Template Check

Execute:

  helm template loki-ssd grafana-community/loki \
    --namespace logging-ssd \
    --version <CHART_VERSION> \
    -f values-loki-ssd.yaml \
    > loki-ssd-rendered.yaml

Acceptance:

  [ ] helm template has no errors
  [ ] read resources can be seen
  [ ] write resources can be seen
  [ ] backend resources can be seen
  [ ] gateway resources can be seen

### 38.5 Task Five: Install Loki SSD

Execute:

  helm install loki-ssd grafana-community/loki \
    --namespace logging-ssd \
    --version <CHART_VERSION> \
    -f values-loki-ssd.yaml

Acceptance:

  [ ] helm list shows deployed
  [ ] read Pod Running
  [ ] write Pod Running
  [ ] backend Pod Running
  [ ] gateway Pod Running
  [ ] Endpoint is not empty

### 38.6 Task Six: Manual Write and Query

Execute:

  kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

Write:

  TS=$(date +%s%N)

  curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
    -H "Content-Type: application/json" \
    -d "{
      \"streams\": [
        {
          \"stream\": {
            \"job\": \"ssd-manual-test\",
            \"namespace\": \"app-demo\",
            \"app\": \"ssd-test\"
          },
          \"values\": [
            [\"${TS}\", \"hello loki simple scalable mode\"]
          ]
        }
      ]
    }"

Query:

  curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
    --data-urlencode 'query={job="ssd-manual-test"}' \
    --data-urlencode 'limit=10' | jq

Acceptance:

  [ ] push succeeds
  [ ] query succeeds
  [ ] can see hello loki simple scalable mode

### 38.7 Task Seven: Add Loki-SSD in Grafana

Configuration:

  Name:
      Loki-SSD

  URL:
      http://loki-ssd-gateway.logging-ssd.svc.cluster.local

Acceptance:

  [ ] Save & test succeeds
  [ ] Explore can query ssd-manual-test

### 38.8 Task Eight: Read Scaling

Modify:

  read:
    replicas: 3

Execute:

  helm upgrade loki-ssd grafana-community/loki \
    --namespace logging-ssd \
    --version <CHART_VERSION> \
    -f values-loki-ssd.yaml

Acceptance:

  [ ] read Pod increases
  [ ] query still works normally

### 38.9 Task Nine: Write Fault Experiment

Execute:

  kubectl delete pod <write-pod-name> -n logging-ssd

Acceptance:

  [ ] write Pod is rebuilt
  [ ] overall write is minimally interrupted
  [ ] understand the relationship between replica count and replication_factor

### 38.10 Task Ten: Record Deprecation Risk

Acceptance:

  [ ] has recorded Simple Scalable is being deprecated
  [ ] understands new production prefers Microservices / Distributed
  [ ] understands this article is for architectural awareness and historical environment maintenance

---

## Thirty-NineI don't know.Acceptance Checklist

After completing this article, confirm:

- [ ] Understand the positioning of Simple Scalable mode
- [ ] Understand that SSD is being deprecated
- [ ] Understand the read role
- [ ] Understand the write role
- [ ] Understand the backend role
- [ ] Understand the gateway's role
- [ ] Understand object storage dependencies
- [ ] Can create logging-ssd Namespace
- [ ] Can create MinIO Bucket
- [ ] Can write values-loki-ssd.yaml
- [ ] Can helm template check resources
- [ ] Can install Simple Scalable Loki
- [ ] Can view read/write/backend/gateway Pods
- [ ] Can view Service and Endpoint
- [ ] Can access Gateway /ready
- [ ] Can manually push logs
- [ ] Can query_range query logs
- [ ] Can add Loki-SSD to Grafana
- [ ] Can understand how Alloy writes to SSD Gateway
- [ ] Can observe the write path
- [ ] Can observe the query path
- [ ] Can scale out read
- [ ] Can scale out write
- [ ] Can simulate read Pod failure
- [ ] Can simulate write Pod failure
- [ ] Can simulate gateway Pod failure
- [ ] Can understand SSD's production limitations
- [ ] Can explain why large-scale production recommends Distributed
- [ ] Can write migration considerations from SSD to Distributed

---

## 40. Common Misconceptions

### 40.1 Misconception One: Simple Scalable is the current long-term production best practice

Incorrect.

Simple Scalable has been officially marked as deprecated.

Learning it is to understand architecture and maintain legacy environments, but it should not be used as the long-term production target for new deployments.

### 40.2 Misconception Two: Multiple replicas of read/write/backend ensures high availability

Incorrect.

Also required:

    Object storage high availability
    Gateway high availability
    Pod dispersed scheduling
    PDB
    limits
    monitoring
    retention
    rollback strategy

### 40.3 Misconception Three: Scaling out write can solve all write issues

Incorrect.

If the issue comes from:

    Application log flushing
    High cardinality label
    Duplicate scraping
    Slow MinIO
    Too low limits

Simple scaling out write cannot resolve the root cause.

### 40.4 Misconception Four: Scaling out read can solve all query slowness

Incorrect.

If the issue comes from:

    Too large query range
    Global regexp
    Global json
    Too many dashboard panels
    Slow object storage
    Poor label design

Simple scaling out read has limited effect.

### 40.5 Misconception Five: Gateway is unimportant

Incorrect.

In Simple Scalable, Gateway is the unified entry point.

Alloy and Grafana should typically access Gateway.

Single Gateway point affects both writing and querying.

### 40.6 Misconception Six: Single Pod MinIO can be used as production object storage

Incorrect.

Single Pod MinIO is only suitable for experimentation.

Production requires:

    Multi-node
    Erasure Coding
    Monitoring
    Backup
    TLS
    Minimum permissions
    Capacity governance

---

## 41. Production Considerations

### 41.1 Caution using SSD in new production environments

Due to SSD deprecation risk, new production environments should prioritize evaluating:

    Monolithic HA
    Microservices / Distributed

Depending on:

    Log volume
    Query concurrency
    Team capability
    Operation complexity
    Version roadmap
    Official recommendations

### 41.2 Existing SSD environments must plan migration

Must focus on:

    Loki 4.0
    Official migration documentation
    Resource capacity
    Object storage compatibility
    Helm values migration
    Rollback strategy
    Downtime window
    Test environment rehearsal

### 41.3 SSD still requires full governance

Even for SSD, must have:

    Retention
    limits_config
    Label governance
    Dashboard governance
    Alerting governance
    Object storage monitoring
    Loki self monitoring
    Security baseline
    Runbook

### 41.4 Queries and writes should be viewed separately

When troubleshooting, distinguish:

    Write failure:
        Focus on gateway → write → object storage.

    Query failure:
        Focus on gateway → read → object storage / ingester.

    Background failure:
        Focus on backend → ruler / compactor / index gateway.

Don't broadly judge Loki failures.

### 41.5 Component scaling must have metrics basis

Before scaling, check:

    Write volume
    Query volume
    Query latency
    Ingestion rate
    Active streams
    429
    5xx
    Memory
    CPU
    Object storage latency

Don't scale based on intuition.

---

## 42. Summary

This document completes the hands-on tutorial for Loki Simple Scalable mode.

Core architecture:

    Gateway
      ↓
    read / write / backend
      ↓
    Object Storage

Three roles:

    write:
        Responsible for log writing, ingester, chunk, WAL, object storage writing.

    read:
        Responsible for LogQL queries, query frontend, querier, object storage reading.

    backend:
        Responsible for compactor, ruler, index gateway, retention, etc. background tasks.

Compared to Monolithic:

    More intuitive read/write separation.
    Easier to scale read/write separately.
    Closer to distributed deployment thinking.
    But complexity increases significantly.

Compared to Distributed:

Component splitting is not fine-grained enough.  
Tuning granularity is worse than Distributed.  
The official documentation has marked it as deprecated.  
Not recommended as a long-term production final goal.

This experiment completed:

- Create logging-ssd Namespace  
- Prepare MinIO Bucket  
- Write SimpleScalable values  
- Helm template check  
- Install Loki SSD  
- View read/write/backend/gateway  
- Manual push/query  
- Grafana integration with Loki-SSD  
- Read scaling  
- Write scaling  
- Fault observation for read/write/gateway  
- Understanding of SSD deprecation risks  

Core conclusions:

- Simple Scalable is worth learning, but cannot be blindly used as a future production final solution.  
- The focus of learning is to understand Loki's read/write splitting, high availability boundaries, object storage dependencies, and scaling directions.  
- If facing an existing SSD environment in the future, maintenance and migration capabilities are needed.  
- If designing a new production environment in the future, carefully evaluate Microservices / Distributed.  

Next article:  
14-Loki Common Troubleshooting: Collection-Write-Query-Storage-Alerting  

Key learning points:  
- Alloy fails to collect logs  
- Loki write failure  
- Gateway 502/503  
- Empty query results  
- query_range timeout  
- MinIO object storage anomaly  
- retention not taking effect  
- Ruler alert not triggering  
- High cardinality causing write/query anomalies  
- 429 rate limit troubleshooting  
- Loki production Runbook  

---

## Reference Documents

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/

- Loki deployment modes:  
  https://grafana.com/docs/loki/latest/get-started/deployment-modes/

- Loki architecture:  
  https://grafana.com/docs/loki/latest/get-started/architecture/

- Install Grafana Loki with Helm:  
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Install the simple scalable Helm chart:  
  https://grafana.com/docs/loki/latest/setup/install/helm/install-scalable/

- Migrate from SSD to distributed:  
  https://grafana.com/docs/loki/latest/setup/migrate/ssd-to-distributed/

- Configure storage:  
  https://grafana.com/docs/loki/latest/setup/install/helm/configure-storage/

- Loki configuration:  
  https://grafana.com/docs/loki/latest/configure/

- Request validation and rate limits:  
  https://grafana.com/docs/loki/latest/operations/request-validation-rate-limits/

- Query best practices:  
  https://grafana.com/docs/loki/latest/query/bp-query/

- Grafana Alloy Documentation:  
  https://grafana.com/docs/alloy/latest/

- MinIO Documentation:  
  https://min.io/docs/minio/kubernetes/upstream/

- Helm Documentation:  
  https://helm.sh/docs/

- Kubernetes StatefulSet:  
  https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

- Kubernetes Deployment:  
  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

- Kubernetes Pod Disruption Budget:  
  https://kubernetes.io/docs/tasks/run-application/configure-pdb/

- Kubernetes Topology Spread Constraints:  
  https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/