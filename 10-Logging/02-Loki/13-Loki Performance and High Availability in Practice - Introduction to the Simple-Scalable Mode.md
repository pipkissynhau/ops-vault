# 13-Loki Performance and High Availability in Practice: Introduction to the Simple-Scalable Mode

## Document Description

This article is the thirteenth in the series dedicated to learning about Loki, focusing on the basic deployment of the Simple Scalable mode, component segmentation, read/write paths, high availability strategies, performance scaling methods, and troubleshooting approaches.

Previous steps have included:

    01-Loki Basics and Experimental Environment Setup
    02-Loki Architecture Principles and Component Responsibilities
    03-Comparison of Loki Deployment Modes and Experimental Selection
    04-Practical Helm Deployment of the Monolithic Mode in Loki
    05-Practical Integration of Object Storage with MinIO for Loki
    06-Practical Collection of Kubernetes Pod Logs Using Grafana-Alloy
    07-Loki Tag Design and Experiments on High Cardinality Issues
    08-Practical Basic LogQL Queries: Searching for Logs by Namespace, Pod, and Container
    09-Advanced LogQL Queries: json-logfmt-regexp-unwrap
    10-Practical Integration of Grafana with Loki for Log Dashboards
    11-Practical Log Alerts in Loki: Interaction between Ruler and AlertManager
    12-Production Governance with Loki: Log Volume, Retention Period, Throttling, and Security

In Article 04, the minimum deployment of Loki using the Monolithic mode was completed.
In Article 05, object storage integration with MinIO was implemented.
In Article 06, Alloy was used to collect Kubernetes Pod logs.
In Article 12, concepts such as retention settings, limits_config, log volume management, and security baselines were covered.

This article begins the exploration of Loki's extended deployment mode:

    Simple Scalable Deployment

Also commonly referred to as:

    SSD

The core concept of the Simple Scalable mode is to divide Loki into three distinct roles based on their functions:

    read
    write
    backend

Among them:

    write:
        Responsible for handling log writing paths.

    read:
        Responsible for handling log query paths.

    backend:
        Handles background tasks such as compaction, rule processing, and index management.

Important Note:

    The Simple Scalable Deployment has been officially marked as deprecated.
    The official migration documentation indicates that the deprecation will occur before the release of Loki 4.0.
    It is also recommended that production environments prioritize the Microservices/Distributed mode.
    Therefore, this article does not consider the Simple Scalable mode as a long-term production solution but rather serves as a means to understand the concepts of read/write separation, high availability, object storage dependency, scaling, and troubleshooting in Loki.

This article addresses the following key questions:

- What is the Simple Scalable mode?
- What are the responsibilities of read, write, and backend roles?
- What are the differences between the Simple Scalable mode and the Monolithic mode?
- What are the differences between the Simple Scalable mode and Microservices?
- Why does the Simple Scalable mode require object storage?
- Why is the number of write replicas usually more critical than that of read/backend replicas?
- Why is the Gateway so important in the Simple Scalable mode?
- How to deploy the Simple Scalable mode using Helm?
- How to observe read/write/backend resources through helm templates?
- How to verify whether the read/write/backend Pods are functioning correctly?
- How to confirm that Alloy is writing logs to the Loki Gateway?
- How to verify log queries through Grafana and the Gateway?
- How to monitor the write and query paths?
- How to perform experiments on scaling read/write replicas?
- How to observe the high availability performance in the event of a Pod failure?
- How to troubleshoot issues such as unavailable writes, failed read queries, or backend exceptions?
- What are the limitations and deprecation risks associated with the Simple Scalable mode in production environments?
- How to understand the transition from the Simple Scalable mode to Microservices/Distributed solutions?

---

## Tags

#Loki #Grafana #SimpleScalable #SSD #ReadWriteBackend #HighAvailability #PerformanceOptimization #Helm #Kubernetes #MinIO #ObjectStorage #SRE #LogSystem #Observability

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/13-Loki Performance and High Availability in Practice: Introduction to the Simple-Scalable Mode.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Understand the role of the Simple Scalable mode.
    2. Comprehend the responsibilities of read, write, and backend roles.
    3. Recognize the importance of the Loki Gateway in the Simple Scalable mode.
    4. Understand why the Simple Scalable mode relies on object storage.
    5. Write Simple ScalTypical Components:

    Distributor
    Ingester
    Querier
    Query-frontend
    Query-scheduler
    Compactor
    Ruler
    Index-Gateway
    Gateway

Suitable for:

    Large-scale log platforms
    Multi-tenant environments
    High write throughput
    High query concurrency
    Scenarios requiring independent component-level scaling

---

## IV. Overview of the Simple Scalable Architecture

### 4.1 Role Division

Simple Scalable is primarily divided into:

    Read
    Write
    Backend
    Gateway

### 4.2 Architecture Diagram

    Alloy / Promtail / Fluent Bit
      ↓
    Loki Gateway
      ↓
    Write
      ├── Distributor
      └── Ingester
      ↓
    Object Storage / MinIO

    Grafana / LogQL / curl
      ↓
    Loki Gateway
      ↓
    Read
      ├── Query Frontend
      └── Querier
      ↓
    Ingester + Object Storage / MinIO

    Backend
      ├── Compactor
      ├── Ruler
      └── Index Gateway and other backend services
      ↓
    Object Storage / MinIO

### 4.3 Role of the Gateway

The Gateway serves as a unified entry point.

Write Requests:

    /loki/api/v1/push
      ↓
    Gateway
      ↓
    Write

Query Requests:

    /loki/api/v1/query
    /loki/api/v1/query_range
    /loki/api/v1/labels
    /loki/api/v1/series
      ↓
    Gateway
      ↓
    Read

Rule Requests:

    /loki/api/v1/rules
      ↓
    Gateway
      ↓
    Backend/Ruler-related components

The significance of the Gateway:

    Unified entry point.
    Conceals the details of the backend read/write/backend.
    Allows Alloy and Grafana to operate without needing to understand the internal segmentation of Loki.
    Can be enhanced with Ingress, authentication, TLS, NetworkPolicy at the front end.

---

## V. Detailed Explanation of Read/Write/Backend Roles

### 5.1 Write Role

The write component is primarily responsible for the log writing process.

Logical Responsibilities:

    Accept log writing requests
    Validate tags and timestamps
    Implement write throttling
    Maintain active streams
    Receive logs and construct chunks
    Write to WAL
    Flush data to object storage
    Participate in the ring mechanism

In Simple Scalable, the write component typically includes:

    Distributor
    Ingester

Writing Process:

    Alloy
      ↓
    Gateway
      ↓
    Write
      ↓
    WAL/Ingester Memory
      ↓
    MinIO/S3

Key metrics for monitoring the write component:

    Number of write requests
    Number of write failures
    429 throttling events
    Active streams count
    Ingester memory usage
    Chunk flushing status
    WAL status
    Object storage write errors

### 5.2 Read Role

The read component is primarily responsible for handling query requests.

Logical Responsibilities:

    Accept query requests
    Parse LogQL queries
    Retrieve recent data from the ingester
    Access historical data from object storage
    Combine query results and return them to Grafana

In Simple Scalable, the read component typically includes:

    Querier
    Query frontend

Query Process:

    Grafana
      ↓
    Gateway
      ↓
    Read
      ↓
    Ingester + Object Storage
      ↓
    Return results

Key metrics for monitoring the read component:

    Number of query requests
    Number of query failures
    Query execution time
    Query timeouts
    Concurrent queries
    Query range specifications
    Object storage read latency
    Grafana dashboard load

### 5.3 Backend Role

The backend component is responsible for executing various background tasks.

Logical Responsibilities:

    Compaction
    Rule enforcement
    Index maintenance
    Background maintenance activities

Key metrics for monitoring the backend component:

    Whether the compactor is functioning correctly
    Whether retention policies are being applied
    Whether ruler rules have been successfully loaded
    Whether ruler rule execution encounters errors
    Whether the index gateway is operational
    Whether object storage access is stable
    Whether there are any backlogs in background tasks

### 5.4 Gateway Role

The Gateway serves as a central entry point and handles route forwarding.

Key considerations:

    5xx and 4xx response codes
    Correct routing of requests
    Availability of read/write/backend endpoints
    Proper configuration of Nginx
    Proper exposure of Services
    Whether Grafana’s data source is set up correctly
    Whether Alloy is writing data to the Gateway

---

## VI. Why Object Storage is Necessary for Simple Scalable

### 6.1 Filesystem can be used in a mon8. Understand production considerations.

### 8.2 Relationship with the previous Loki monolith

If a single Loki instance has already been deployed in the current logging Namespace, there are two approaches:

    Method One:
        Uninstall the original Loki and then install Simple Scalable.

    Method Two:
        Use a new Release name and Namespace, such as loki-ssd / logging-ssd.

Learning recommendations:

    It is recommended to use the new Namespace:
        logging-ssd

Reasons:

    This will not disrupt previous experiments with the single Loki instance.
    It makes it easier to compare the two deployment methods.
    In case of issues, the logging-ssd Namespace can be deleted directly.
    It reduces the impact on existing Grafana / Alloy setups.

In this document, we will use:

    Namespace:
        logging-ssd

    Release:
        loki-ssd

### 8.3 Connection strategies for Alloy and Grafana

After deploying Simple Scalable, you can choose one of the following methods:

    Method One:
        Manually perform push/query operations using curl for verification.

    Method Two:
    Update Alloy to use the new Gateway address.

    Method Three:
    Add a new Loki-SSD Data Source in Grafana.

Our recommendation is:

    First, verify the setup using curl.
    Then add the Loki-SSD Data Source in Grafana.
    Finally, modify Alloy as needed or temporarily deploy a test version of Alloy.      enabled: false

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

### 12.3 Configuration Description

    deploymentMode: SimpleScalable

Enable the Simple Scalable mode.

    write.replicas: 3

Deploy 3 replicas for the write path.

    read.replicas: 2

Deploy 2 replicas for the query path.

    backend.replicas: 2

Deploy 2 replicas for background tasks.

    gateway.enabled: true

Enable the unified entry point.

    gateway.replicas: 2

Two replicas for the Gateway to avoid single points of failure.

    commonConfig.replication_factor: 3

The write replication factor is set to 3.

Note:

    The replication_factor should match the number of write replicas, production availability goals, and resource scale.
    In learning environments with limited resources, it can be temporarily reduced to 1 or 2, but this will decrease high availability.

    minio.enabled: false

Do not use Chart's built-in MinIO; instead, use the independently deployed MinIO.

    storage.type: s3

Access MinIO through an S3-compatible method.

    s3ForcePathStyle: true

MinIO usually requires path-style access.

    insecure: true

Use HTTP in experimental environments.
For production environments, HTTPS is recommended.

---

## Section 13: Helm Template Rendering Check

### 13.1 Execute Rendering

    helm template loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml \
      > loki-ssd-rendered.yaml

### 13.2 View Resource Types

    grep "^kind:" loki-ssd-rendered.yaml | sort | uniq -c

### 13.3 View read/write/backend

    grep -n "loki-ssd-read" loki-ssd-rendered.yaml

    grep -n "loki-ssd-write" loki-ssd-rendered.yaml

    grep -n "loki-ssd-backend" loki-ssd-rendered.yaml

    grep -n "loki-ssd-gateway" loki-ssd-rendered.yaml

### 13.4 View Workloads

    grep -n "kind: Deployment" loki-ssd-rendered.yaml

    grep -n "kind: StatefulSet" loki-ssd-rendered.yaml

### 13.5 View Services

    grep -n "kind: Service" loki-ssd-rendered.yaml

### 13.6 View MinIO Configuration

    grep -n "minio.minio.svc.cluster.local" loki-ssd-rendered.yaml

    grep -n "loki-ssd-chunks" loki-ssd-rendered.yaml

### 13.7 View Images

    grep -n "image:" loki-ssd-rendered.yaml | sort -u

### 13.8 Common Rendering Errors

#### deploymentMode Not Supported

Solution:

    grep -n "deploymentMode" values-loki-default.yaml

Check if the current Chart supports this value.

#### monolithic Field Does Not Exist

Solution:

    The current Chart may use singleBinary.
    Delete the non-existent field or adjust it according to default values.

#### monitoring.selfMonitoring Field Does Not Exist

Solution:

    Adjust or delete this configuration based on the current Chart values.

#### storage Field Does Not Match

Solution:

    Refer to the official storage examples for the current version.
    Use helm show values to search for storage and s3 fields.

---

## Section 14: Install Simple Scalable Loki

### 14.1 Installation

    helm install loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

### 14.2 View Helm Release

    helm list -n logging-ssd

Expected output:

    loki-ssd    logging-ssd    deployed

### 14.3 View Pods

    kubectl get    kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=100

To view the gateway logs:

    kubectl logs -n logging-ssd -l app.kubernetes.iocomponent=gateway --tail=100

If the labels do not match, first check the actual labels:

    kubectl get pod -n logging-ssd --show-labels

---

## Section Sixteen: Manual Writing and Querying Tests

### 16.1 Writing Test Logs

Maintain port forwarding:

    kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

In another terminal, execute the following command:

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"ssd-manual-test\",
              \"namespace\": \"app-demo\", \
              \"app\": \"ssd-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki simple scalable mode\"]
            ]
          }
        ]
      }"

If there is no output, it usually indicates success.

### 16.2 Querying Test Logs

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-manual-test"}' \
      --data-urlencode 'limit=10' | jq

The expected result is:

    hello loki simple scalable mode

### 16.3 Checking Labels

    curl -s "http://127.0.0.1:3101/loki/api/v1/labels" | jq

To view the job labels, execute:

    curl -s "http://127.0.0.1:3101/loki/api/v1/label/job/values" | jq

The expected result should include:

    ssd-manual-test

---

## Section Seventeen: Verifying Object Storage Writing

### 17.1 Entering the minio Container

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

To check, execute:

    mc ls local/loki-ssd-chunks

For a recursive view:

    mc find local/loki-ssd-chunks | head

To exit, use:

    exit

### 17.2 Why Objects May Not Appear Immediately

When a log is first written, it may not immediately appear in MinIO.

Reasons:

    The log first goes to the write / ingester.
    The chunks are flushed only when they meet certain conditions.
    Recent data might be stored in the ingester's memory.
    There can be a delay in writing objects to storage.

Judgment criteria:

    The push operation was successful.
    The query operation was successful.
    There are no errors related to object storage in Loki.
    Wait for a while, and the object should appear.

### 17.3 Batch Writing Tests

    for i in $(seq 1 300); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"ssd-batch-test\",
                \"namespace\": \"app-demo\", \
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

To query the results:

    curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-batch-test"}' \
      --data-urlencode 'limit=20' | jq```markdown
helm upgrade alloy grafana/alloy \
  --namespace logging \
  --version <ALLOY_CHART_VERSION> \
  -f values-alloy-loki.yaml

View results:

  kubectl rollout status ds/alloy -n logging

  kubectl logs <alloy-pod-name> -n logging --tail=200 | grep -Ei "error|warn|loki|failed|429"

### 19.5 Pod logs generation

Run the following command in a container:

  kubectl run curl-test \
    --rm -it \
    --image=curlimages/curl:8.5.0 \
    -n app-demo \
    -- sh

Inside the container, perform the following operations:

  curl http://nginx-demo.app-demo.svc.cluster.local
  curl http://nginx-demo.app-demo.svc.cluster.local/notfound
  exit

### 19.6 Querying data in Loki-SSD

Perform queries using the following syntax:

  {namespace="app-demo"}

  {namespace="app-demo", app="nginx-demo"}

  {namespace="app-demo", app="nginx-demo"} |= "404"

---

## Section 20: Observing Writing Paths

### 20.1 Overview of writing paths

    Alloy / curl
      ↓
    Gateway
      ↓
    Write
      ↓
    WAL / Ingester
      ↓
    Object Storage

### 20.2 Viewing Gateway logs

Use the following command to view Gateway logs:

  kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=200

If the label does not match, use the following command to specify the Pod:

  kubectl logs <gateway-pod-name> -n logging-ssd --tail=200

Observe the following log entries:

    POST /loki/api/v1/push
    2xx
    4xx
    5xx

### 20.3 Viewing Write logs

Use the following command to view Write logs:

  kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=300

Filter the logs using the following command:

  kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=500 | grep -Ei "push|ingest|stream|chunk|flush|error|warn|s3|minio|rate|429"

### 20.4 Viewing Loki metrics

Use port forwarding to view metrics:

  kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

View the metrics using the following command:

  curl -s http://127.0.0.1:3101/metrics | grep "^loki_" | head -50

Pay attention to the following metrics:

    distributor requests
    ingester stream
    chunk flush
    request duration
    rate limit

The specific metric names may vary depending on the version. Refer to the actual /metrics file for accurate information.

---

## Section 21: Observing Query Paths

### 21.1 Overview of query paths

    Grafana / curl
      ↓
    Gateway
      ↓
    Read
      ↓
    Write ingester + object storage
      ↓
    Return query results

### 21.2 Executing queries

Use the following command to execute a query:

  curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
    --data-urlencode 'query={job="ssd-batch-test"}' \
    --data-urlencode 'limit=50' | jq

### 21.3 Viewing Read logs

Use the following command to view Read logs:

  kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=300

Filter the logs using the following command:

  kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=500 | grep -Ei "query|querier|frontend|error|warn|timeout|s3|minio"

### 21.4 Viewing Gateway query logs

Use the following command to view Gateway query logs:

  kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=200 | grep -Ei "query|labels|series"

### 21.5 Query load testing experiment

Perform 20 queries in a loop:

  for i in $(seq 1 20); do
    curl -G -s "http://127.0.0.1:31        -H "Content-Type: application/json" \
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

### 23.4 Understanding the Effects of Scaling Out

Scaling out for writing operations primarily improves:

    Write throughput
    Write availability
    Stream capacity
    Load distribution among ingesters

However, scaling out does not address the following issues:

    Abnormal log flushing by applications
    High基数 labels
    Slow writes to MinIO
    Unavailability of object storage
    Insufficient limits settings
    Single-point failures in Gateways

---

## Chapter 24: Backend Scaling Experiments

### 24.1 Checking the Backend

    kubectl get pods -n logging-ssd -o wide | grep backend

### 24.2 Modifying Backend Replicas

Modification:

    backend:
      replicas: 3

Execution:

    helm upgrade loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

Verification:

    kubectl get pods -n logging-ssd -o wide | grep backend

### 24.3 Verifying Ruler and Compactor

Check backend logs:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=500 | grep -Ei "ruler|compactor|retention|index|error|warn"

### 24.4 Understanding the Effects of Scaling Out

Scaling out the backend does not necessarily directly improve query or write performance.

It mainly affects the following backend components:

    Compactor
    Ruler
    Index Gateway
    Retention Mechanisms

In production, when scaling out the backend, it is essential to understand whether specific components support multi-replica collaboration and how the current Chart/Version configuration impacts performance.

---

## Chapter 25: Pod Failure Experiments: Deleting Read Pods

### 25.1 Checking Read Pods

    kubectl get pod -n logging-ssd -o wide | grep read

### 25.2 Deleting a Read Pod

    kubectl delete pod <read-pod-name> -n logging-ssd

### 25.3 Observing Recovery

    kubectl get pod -n logging-ssd -w

### 25.4 Simultaneous Log Queries

In another terminal, execute the following command:

    for i in $(seq 1 20); do
      curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
        --data-urlencode 'query={job="ssd-batch-test"}' \
        --data-urlencode 'limit=5' >/dev/null
      echo "Query $i completed"
      sleep 1
    done

### 25.5 Observing Conclusions

If there are more than 1 read replicas:

    Deleting a read Pod should not cause complete interruptions in queries.
    There may be temporary fluctuations in performance.
    The Service will automatically remove any failed endpoints.
    A new Pod will be created automatically.

If there is only 1 read replica:

    Queries may fail during the deletion process.

---

## Chapter 26: Pod Failure Experiments: Deleting Write Pods

### 26.1 Checking Write Pods

    kubectl get pod -n logging-ssd -o wide | grep write

### 26.2 Deleting a Write Pod

    kubectl delete pod <write-pod-name> -n logging-ssd

### 26.3 Observing Recovery

    kubectl get pod -n logging-ssd -w

### 26.4 Simultaneous Log Writes

In another terminal, execute the following command:

    for i in $(seq 1 30); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -dUse PDB
Set requests/limits
Avoid having all replicas concentrated on one node

### 28.3 topologySpreadConstraints Approach

Example approach:

    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: loki

Note:

    Check whether specific fields are supported by the Helm Chart values.
    Confirm this using `helm show values`.
    Always test the scheduling effect in production.

---

## Twenty-Nine, Resource Requests / Limits

### 29.1 Why Configure Resources

The Loki component is quite sensitive to CPU/memory usage.

Not configuring resources may lead to:

    The Pod being scheduled to a node with insufficient resources
    Sudden high memory consumption during queries
    OOMKilling during writes
    Resource contention among nodes
    Lack of basis for HPA/scheduling strategies

### 29.2 Key Points for read Resources

Read operations mainly consume:

    CPU
    Memory
    Object storage reads
    Query concurrency

Example configuration:

    read:
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 2
          memory: 4Gi

### 29.3 Key Points for write Resources

Write operations mainly consume:

    Memory
    Disk WAL
    Network
    Object storage writes
    CPU

Example configuration:

    write:
      resources:
        requests:
          cpu: 500m
          memory: 2Gi
        limits:
          cpu: 2
          memory: 6Gi

### 29.4 Key Points for backend Resources

Backend operations mainly consume:

    Compactor CPU/I/O
    Ruler resource queries
    Index gateway memory
    Object storage read/write operations

Example configuration:

    backend:
      resources:
        requests:
          cpu: 300m
          memory: 1Gi
        limits:
          cpu: 1
          memory: 3Gi

### 29.5 Notes

The above are just example configurations.

Production resource settings should be based on:

    Daily log volume
    Query concurrency
    Retention requirements
    Number of unique labels
    Number of streams
    Object storage performance
    Number of Dashboards
    Number of Ruler rules

Conduct stress tests and make necessary adjustments accordingly.

---

## Thirty, Key Monitoring Areas for Simple Scalable

### 30.1 Write Monitoring

Monitor:

    Number of write requests
    Number of write failures
    429 rate limiting errors
    Active streams
    Ingester memory usage
    WAL size
    Chunk flushing status
    Object storage write errors
    Pod restarts
    Ring status

### 30.2 Read Monitoring

Monitor:

    Number of read requests
    Number of read failures
    Query time
    Query timeouts
    Queue length at the query frontend
    CPU/memory usage of the querier
    Object storage read errors
    Dashboard load due to queries

### 30.3 Backend Monitoring

Monitor:

    Whether the compactor is functioning properly
    Whether retention tasks are being executed
    Whether the ruler is working correctly
    Errors in ruler rule evaluation
    Index gateway failures
    Object storage access issues

### 30.4 Gateway Monitoring

Monitor:

    Response codes (2xx, 4xx, 5xx)
    Request latency
    Upstream availability issues
    Empty endpoints
    Nginx reloads
    Pod restarts

### 30.5 Object Storage Monitoring

Monitor:

    MinIO availability
    Bucket capacity
    S3 API errors (4xx/5xx)
    Request latency
    Disk capacity
    Node status
    Certificate validity period

---

## Thirty-One, Common Fault 1: Write Unavailability

### 31.1 Symptoms

    Alloy fails to send batches
    Push operations return code 500
    Push operations return code 429
    Loki Gateway returns error codes 502/503
    The write Pod enters a CrashLoopBackOff state
    The write Endpoint becomes unavailable

### 31.2 Troubleshooting Steps

Check the Pod:

    `kubectl get pod -n logging-ssd -o wide | grep write`

Check Service/endpoints:

    `kubectl get svc -n logging-ssd | grep write`
    `kubectl get endpoints -n logging-ssd | grep write`

Review logs:

    `kubectl logs -n logging-ssd -l app.kubernetes.io/component=write --tail=500 | grep -Ei "error|warn|ring|ingester|s3|minio|rate|429|flush|wal"`

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=500 | grep -Ei "compactor|ruler|retention|rule|index|error|warn|s3|minio"

Check the Ruler API:

    curl -s http://127.0.0.1:3101/loki/api/v1/rules | jq

Check the retention configuration:

    helm get values loki-ssd -n logging-ssd -a | grep -n "retention" -A 50

### 33.3 Common Causes

    The backend Pod is unavailable.
    Ruler is not enabled.
    The rules bucket does not exist.
    There is an error in the compactor configuration.
    There is an error in the delete_request_store configuration.
    Insufficient permissions for object storage.
    Incorrect AlertManager URL.
    Insufficient backend resources.

### 33.4 Solutions

    1. Restore the backend Pod.
    2. Check the MinIO bucket.
    3. Verify the Ruler configuration.
    4. Check the AlertManager.
    5. Check the Compactor retention settings.
    6. Verify object storage permissions.
    7. Adjust backend resources if necessary.

---

## Section Thirty-Four: Common Fault Four: Gateway 502 / 503

### 34.1 Symptoms

    Failure to save and test in Grafana.
    Failure of the `curl /ready` command.
    The `push` operation returns a 502 error.
    The `query` operation returns a 503 error.
    The Gateway logs indicate an upstream issue.

### 34.2 Troubleshooting

Check the Gateway:

    kubectl get pod -n logging-ssd -o wide | grep gateway

View Gateway logs:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=gateway --tail=500

Check Service/Endpoints:

    kubectl get svc -n logging-ssd

    kubectl get endpoints -n logging-ssd

Verify if there are Endpoints for `read/write/backend`:

    kubectl get endpoints -n logging-ssd | grep -E "read|write|backend"

### 34.3 Common Causes

    The `read` Pod is unavailable.
    The `write` Pod is unavailable.
    The backend Pod is unavailable.
    The Service selector does not match.
    The Endpoint is missing.
    An error occurred in generating the Gateway configuration.
    The Chart values configuration is incorrect.
    NetworkPolicy restrictions are blocking access.

### 34.4 Solutions

    1. Restore the backend Pod.
    2. Check the Endpoints.
    3. Verify the Gateway configuration.
    4. Check the NetworkPolicy settings.
    5. Verify the Helm values configuration.
    6. Roll back changes if necessary.

---

## Section Thirty-Five: Risks of Abandoning Simple Scalable

### 35.1 Current Status

The Simple Scalable Deployment has been officially marked as deprecated.

Official migration guidance:

    Deprecation date TBD.
    The transition will occur before the release of Loki 4.0.
    Environments using SSDs should plan to migrate to Distributed/Microservices architectures.

### 35.2 Implications for Learning

It is still worth understanding the concept:

    The separation of read/write backend functions is an important design principle.
    Older environments may still utilize SSDs.
    This knowledge helps in understanding Loki's data path mechanisms and object storage best practices.
    It also aids in the transition to Microservices-based architectures.

### 35.3 Impact on New Production Environments

For new production setups, it is not recommended to rely on SSDs long-term.

Suggestions:

    **Small-scale environments:** Consider using a monolithic HA architecture with object storage, evaluating options based on version compatibility and official recommendations.

    **Large-scale environments:** Adopt Microservices or Distributed architectures.

    **For existing SSD-based environments:** Develop a migration plan.

### 35.4 Recommendations for Existing SSD Environments

Existing environments should:

    Fix the current Loki version.
    Keep an eye on Loki 4.0 development.
    Study the official migration documentation.
    Conduct migration trials.
    Assess whether the cluster can handle both SSDs and Distributed systems simultaneously.
    Ensure that object storage configurations require minimal adjustments or no migration at all.
    Prepare a rollback strategy in case of issues.
    Avoid rushing into migrations just before the version becomes officially deprecated.

---

## Section Thirty-Six: Understanding the Transition from Simple Scalable to Distributed

### 36.1 Changes from SSD to Distributed Architecture

**Simple Scalable:**  
- Read operations are handled by a single backend service.  
- Write operations also occur through this same backend.  

```markdown
mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

mc mb local/loki-ssd-chunks

mc mb local/loki-ssd-ruler

mc mb local/loki-ssd-admin

mc ls local

Acceptance:

[ ] loki-ssd-chunks created
[ ] loki-ssd-ruler created
[ ] loki-ssd-admin created

### 38.3 Task Three: Write values-loki-ssd.yaml

Acceptance:

[ ] deploymentMode set to SimpleScalable
[ ] read replicas configured
[ ] write replicas configured
[ ] backend replicas configured
[ ] gateway enabled
[ ] storage using MinIO / S3
[ ] replication_factor appropriate
[ ] minio.enabled set to false

### 38.4 Task Four: Helm Template Check

Execution:

helm template loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml \
      > loki-ssd-rendered.yaml

Acceptance:

[ ] No errors in the helm template
[ ] read resources visible
[ ] write resources visible
[ ] backend resources visible
[ ] gateway resources visible

### 38.5 Task Five: Install Loki SSD

Execution:

helm install loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

Acceptance:

[ ] helm list shows deployed resources
[ ] read Pod running
[ ] write Pod running
[ ] backend Pod running
[ ] gateway Pod running
[ ] Endpoint available

### 38.6 Task Six: Manual Writing and Querying

Execution:

kubectl port-forward svc/loki-ssd-gateway 3101:80 -n logging-ssd

Writing:

TS=$(date +%s%N)

curl -s -X POST "http://127.0.0.1:3101/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"ssd-manual-test\",
              \"namespace\": \"app-demo\"",
              \"app\": \"ssd-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki simple scalable mode\"]
            ]
          }
        ]
      }"

Querying:

curl -G -s "http://127.0.0.1:3101/loki/api/v1/query_range" \
      --data-urlencode 'query={job="ssd-manual-test"}' \
      --data-urlencode 'limit=10' | jq

Acceptance:

[ ] Push successful
[ ] Query successful
[ ] "hello loki simple scalable mode" displayed

### 38.7 Task Seven: Adding Loki-SSD to Grafana

Configuration:

Name:
        Loki-SSD

URL:
        http://loki-ssd-gateway.logging-ssd.svc.cluster.local

Acceptance:

[ ] Save & test successful
[ ] "ssd-manual-test" can be queried in Grafana

### 38.8 Task Eight: Read Replication Expansion

Modification:

read:
      replicas: 3

Execution:

helm upgrade loki-ssd grafana-community/loki \
      --namespace logging-ssd \
      --version <CHART_VERSION> \
      -f values-loki-ssd.yaml

Acceptance:

[ ] Number of read Pods increased
[ ] Queries still function normally

### 38.9 Task Nine: Write Fault Experiment

Execution:

kubectl delete pod <write-pod-name> -n logging-ssd

Acceptance:

[ ] The write Pod is recreated
[ ] Overall writing operations remain uninterrupted
[ ] Relationship between replicas and replication_factor understood

### 38.10 Task Ten: Documenting Discontinued Practices

Acceptance:

[ ] Noted that Simple Scalable is being phased out
[ ] Understood that Microservices / Distributed architectures are more recommended for new production
[ ] Recognized that this document serves to aid in architectural understanding and historical environment maintenance

---

## Thirty-Nine, Acceptance Checklist

Upon completing this document, you should confirm:

[ ] Understanding of the role of Simple Scalable mode
[ ] Awareness that SSDs are being phased out
[ ] Comprehension of the functions of read and write components
[ ] Knowledge of the importance of the backend and gateway components
[ ] Understanding of object storage dependencies
[ ] Ability toCapacity Governance

---

## Section 41: Production Considerations

### 41.1 Use SSDs with Caution in New Production Environments

Due to the potential for SSDs to become obsolete, new production environments should prioritize evaluating the following aspects:

    Monolithic HA
    Microservices / Distributed

The specific approach will depend on factors such as:

    Log volume
    Query concurrency
    Team capability
    Operational complexity
    Development roadmap
    Official recommendations

### 41.2 Develop a Migration Plan for Existing SSD Environments

It is essential to pay attention to the following aspects:

    Loki 4.0
    Official migration documentation
    Resource capacity
    Compatibility with object storage
    Migration of Helm values
    Rollback strategy
    Downtime window
    Testing in a mock environment

### 41.3 SSDs Still Require Comprehensive Governance

Even for SSDs, the following governance measures are necessary:

    Retention policies
    Configuration limits
    Label management
    Dashboard monitoring
    Alerting mechanisms
    Object storage monitoring
    Self-monitoring tools for Loki
    Security baseline settings
    Preparation of runbooks

### 41.4 Distinguish Between Query and Write Operations

When troubleshooting issues, it is important to distinguish between the following cases:

    Writing failures:
        Focus on the sequence: gateway → write → object storage.

    Reading failures:
        Focus on the sequence: gateway → read → object storage / ingester.

    Back-end failures:
        Focus on the components involved: backend → ruler / compactor / index gateway.

    Do not make general conclusions based solely on Loki failures.

### 41.5 Expansion of Components Should Be Based on Measured Metrics

Before expanding any component, consider the following metrics:

    Writing volume
    Reading volume
    Query latency
    Ingestion rate
    Active streams
    Error codes like 429 and 5xx
    Memory usage
    CPU utilization
    Object storage latency

Do not expand components based on intuition alone.

---

## Section 42: Summary

This article provides a practical guide to understanding the Loki Simple Scalable mode.

**Core Architecture:**

    Gateway
      ↓
    Read/Write/Backend
      ↓
    Object Storage

**Three Types of Roles:**

    **Writing:** Responsible for logging, ingestion, chunk processing, WAL creation, and object storage writes.

    **Reading:** Handles LogQL queries, frontend interactions, query processing, and object storage reads.

    **Backend:** Performs compaction, rule application, index management, and retention tasks.

**Advantages of Simple Scalable Compared to Monolithic:**

    Easier to understand the separation of read and write operations.
    Easier to scale read and write components independently.
    Closer to distributed deployment concepts.
    However, it also introduces increased complexity.

**Disadvantages of Simple Scalable Compared to Distributed:**

    Components are not finely divided enough.
    Tuning granularity is lower than in distributed systems.
    Officially marked as being phased out.
    Not recommended as a long-term production solution for new projects.

**Practical Exercises Completed in This Article:**

    Created a logging-ssd Namespace.
    Prepared a MinIO Bucket.
    Written SimpleScalable Helm values.
    Checked the Helm template.
    Installed Loki with SSD support.
    Observed read/write/backend/gateway operations.
    Performed manual push/query operations.
    Integrated Grafana with Loki-SSD.
    Expanded both read and write capabilities.
    Observed failures in the read/write/backend/gateway components.
    Comprehended the potential risks associated with using SSDs.

**Key Takeaways:**

    Simple Scalable is worth learning from, but it should not be blindly adopted as a future production solution.
    The focus of learning should be understanding how Loki manages read and write operations, its high-availability limitations, object storage dependencies, and expansion strategies.
    If dealing with existing SSD environments, it is necessary to have the skills to maintain and migrate them.
    When designing new production systems, Microservices or Distributed architectures should be carefully considered.

**Next Article:**

    14-Loki Common Fault Troubleshooting: Collection, Writing, Querying, Storage, and Alerting

    Key topics to learn:

    Issues with Alloy log collection.
    Loki writing failures.
    Gateway errors like 502/503.
    Empty query results.
    Query_range timeouts.
    MinIO object storage anomalies.
    Ineffective retention policies.
    Ruler alerts not firing.
    High基数 causing writing/readings issues.
    Troubleshooting 429 rate limits.
    Creating production-grade Loki runbooks.

---

## References

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Loki deployment modes:
  https://grafana.com/docs/loki/latest/get-started/deployment-modes/

- Loki