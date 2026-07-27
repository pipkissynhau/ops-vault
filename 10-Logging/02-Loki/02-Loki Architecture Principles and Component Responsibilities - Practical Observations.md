# 02-Loki Architecture Principles and Component Responsibilities: Practical Observations

## Documentation Notes

This article is the second part of a specialized study on Loki, aimed at providing a systematic understanding of its architectural principles, the responsibilities of its core components, the log writing process, the log querying process, and how to monitor Loki's operational status in Kubernetes using kubectl, Helm, the Loki HTTP API, and log output observations.

This article is not about simply memorizing component names such as Distributor, Ingester, and Querier. Instead, it addresses the following practical questions from an operational perspective:

- Why does Loki have a modular architecture?
- What are the differences between components in Loki's monolithic mode, Simple Scalable mode, and Microservices mode?
- After logs are written from Alloy to Loki, which internal components do they pass through?
- How does Loki handle queries from Grafana internally?
- What are the specific responsibilities of Distributor, Ingester, Querier, Query Frontend, Compactor, and Ruler?
- What roles do Gateway, Index Gateway, Cache, and Object Storage play in production environments?
- Why aren't many individual Pods visible in monolithic mode?
- How can kubectl be used to observe Loki Pods, Services, StatefulSets, and Deployments?
- How can the ready status, metrics, ring, config, and other information be viewed through the Loki API?
- How can the normal operation of Loki's logging, querying, compression, and alerting functions be determined based on logs and metrics?
- What phenomena should be closely monitored when deploying Loki in production?

This article is recommended to be read after completing the following content:

- 01-Loki Basic Understanding and Experimental Environment Setup

This article leads to the following topics:

- 03-Loki Deployment Mode Comparison and Experimental Selection
- 04-Loki Monolithic Mode: Helm Deployment Practice
- 05-Loki Object Storage Integration with MinIO Practice
- 06-Grafana-Alloy for Collecting K8S-Pod Logs Practice

---

## Tags

#Loki #Grafana #GrafanaAlloy #Kubernetes #Log System #Loki Architecture #Distributor #Ingester #Querier #Compactor #Ruler #QueryFrontend #SRE #Observability

---

## Recommended Reading Path

Recommended reading path:

    10-Logs/02-Loki/02-Loki Architecture Principles and Component Responsibilities: Practical Observations.md

---

## I. Experimental Objectives

After completing this article, you should be able to understand and verify the following:

    1. What specific tasks each of Loki's core components performs.
    2. How the logging writing process in Loki works.
    3. How the logging querying process in Loki functions.
    4. The differences in component presentation between monolithic mode and split modes.
    5. How to monitor Loki Pods, Services, logs, and metrics in Kubernetes.
    6. How to use APIs to determine whether Loki is ready for use.
    7. How to view Loki's own metrics using the /metrics endpoint.
    8. How to understand the relationships between Loki, Alloy, Grafana, and object storage.

The focus of this article is:

    First, understand the architecture.
    Then, observe the practical effects.
    Finally, proceed with formal deployment.

---

## II. Experimental Environment

This article assumes the use of the following Kubernetes experimental environment:

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

Note:

    This article focuses on architectural observation. If Loki has not been deployed yet, you can first read about the architecture and commands. The actual observation commands can be executed after deploying Loki in Chapter 04. If Loki is already present in the environment, you can directly proceed with the observation commands outlined in this article.

---

## III. Overall Loki Architecture

Loki is a log aggregation system.

Its core concept is:

    Log bodies are not indexed by default; only key tags are primarily indexed.
    During queries, the scope is first narrowed down using tags, and then the log content is filtered accordingly.

Typical Kubernetes log flow:

    Kubernetes Pod
      ↓
    stdout / stderr
      ↓
    containerd writes logs to node-specific files
      ↓
    /var/log/containers/*.log
      ↓
    Grafana Alloy / Promtail / Fluent## Section Six: Component One: Distributor

### 6.1 Function of the Distributor

The Distributor serves as the entry point for writing logs in Loki.

Log collection agents push logs to the Distributor.

Typical sources include:

    Grafana Alloy
    Promtail
    Fluent Bit
    Logstash
    Custom clients

The main responsibilities of the Distributor are:

    Receiving log write requests
    Verifying log format
    Checking tenant information
    Validating timestamps
    Checking labels
    Enforcing rate limits
    Distributing logs to the Ingester

### 6.2 Write API

Common paths for Loki's write API include:

    /loki/api/v1/push

Typical workflow:

    Alloy
      ↓
    POST /loki/api/v1/push
      ↓
    Distributor
      ↓
    Ingester

### 6.3 What to Pay Attention To with the Distributor

In production, focus on:

    Whether write requests are successful
    Occurrence of 400 errors
    Occurrence of 429 errors
    Occurrence of 500 errors
    Label limits being exceeded
    Log lines being too large
    Exceeding the ingestion rate limit
    Tenant authentication failures

Common issues include:

    400 Bad Request:
        Illegal log format, timestamp, or label.

    401 / 403:
        Authentication or tenant permission issues.

    429 Too Many Requests:
        Write rate exceeds the limit.

    500:
        Internal Loki errors or issues with downstream components.
---

## Section Seven: Component Two: Ingester

### 7.1 Function of the Ingester

The Ingester is responsible for receiving logs distributed by the Distributor and organizing them into chunks.

Its main tasks include:

    Receiving log streams
    Temporarily storing logs
    Organizing logs by label stream
    Constructing chunks
    Writing to WAL
    Flushing chunks to object storage
    Maintaining query capability for recent data

### 7.2 What is a Stream?

In Loki, logs with the same set of labels belong to the same stream.

For example:

    {namespace="app-demo", pod="nginx-xxx", container="nginx"}

This represents a log stream.

If the label changes, it creates a new stream.

For example:

    request_id="abc"
    request_id="def"

Using request_id as a label could result in multiple streams for each request, leading to high cardinality issues.

### 7.3 What is a Chunk?

A chunk is the basic unit for storing logs in Loki.

Simply put:

    Multiple logs
      ↓
    Grouped by stream
      ↓
    Cut into chunks based on time and size
      ↓
    Written to storage

The Ingester will flush chunks to backend storage according to configuration.

Backend storage options include:

    File system
    MinIO
    S3
    GCS
    Azure Blob
    Other compatible object stores

### 7.4 What to Pay Attention To with the Ingester

In production, monitor:

    Memory usage
    WAL status
    Whether chunk flushing fails
    Frequent restarts of the ingester
    Number of active streams
    Excessive number of streams
    Flush delays
    Success rate of object storage writes

Common issues include:

    Incorrect label design causing excessive stream growth.
    Increased memory usage by the ingester.
    Object storage unavailability leading to flush failures.
    Long WAL recovery times.
    Short-term unavailability due to pod restarts.
---

## Section Eight: Component Three: Querier

### 8.1 Function of the Querier

The Querier is responsible for executing LogQL queries.

When Grafana queries logs, the Querier ultimately retrieves the data.

Data sources for Querier queries generally include:

    Latest data from the ingester
    Historical chunks in object storage
    Index data

### 8.2 Query Workflow

Typical query process:

    {namespace="app-demo"} |= "ERROR"

Workflow:

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
    Return log results

### 8.3 What to Pay Attention To with the Querier

In production, monitor:

    Query speed
    Query timeouts
    Excessively large query scopes
    Scanning too many streams
    Cache hits
    Rate limiting
    High CPU/memory consumption during queries

Common issues include:

    Users performing broad queries like {namespace=~".+"} |= "error"
    Too wide query time ranges
    Overly general label filters
    Not narrowing the scope with namespace/app/pod first
    Volumes of logs being too largeGateway typically serves as a unified entry point ahead of Loki.

In Helm deployments, Nginx is a common implementation for Gateway.

Its responsibilities include:

    Receiving external requests
    Forwarding write requests to the writing components
    Forwarding query requests to the reading components
    Uniformly exposing the Loki API
    Simplifying client configuration

### 14.2 The Significance of Gateway

For collection agents, it is inconvenient to configure multiple component addresses separately.

The ideal scenario is:

    Alloy only needs to specify one Loki Gateway address.

For example:

    http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push

Gateway then forwards requests to the appropriate backend based on the path.

### 14.3 Key Points to Consider for Gateway

Pay attention to:

    Whether the gateway is accessible
    Whether Nginx configuration is correct
    Whether the Service is functioning normally
    Whether there are any 502 or 503 errors
    Whether TLS configuration is correct
    Whether authentication settings are appropriate
    Whether path forwarding is functioning correctly

---

## Chapter Fifteen: Component Ten: Cache

### 15.1 The Role of Cache

In production, cache can be used in Loki to improve query performance.

Common types of caches include:

    Query result cache
    Chunk cache
    Index cache

Common implementations include:

    Memcached
    Redis
    Other caching systems

### 15.2 What Problems Does Cache Solve?

It mainly helps to:

    Reduce the burden of repeated queries
    Lower the read load on object storage
    Improve query response times
    Alleviate the pressure from large-scale queries

### 15.3 Is Cache Necessary During Learning?

During the initial learning phase, cache configuration may not be necessary.

In production, whether to use cache depends on factors such as:

    Number of queries
    Volume of logs
    Performance of object storage
    Query latency
    Number of users

---

## Chapter Sixteen: Component Eleven: Object Storage

### 16.1 The Role of Object Storage

Object storage is used for storing Loki's long-term log data.

Common object storage solutions include:

    MinIO
    AWS S3
    Alibaba Cloud OSS
    Tencent Cloud COS
    Huawei Cloud OBS
    GCS
    Azure Blob

For these experiments, we will use:

    MinIO

### 16.2 Why Is Object Storage Needed?

Production log systems should not rely solely on local disks for long-term storage.

Reasons include:

    Pods may be recreated
    Nodes may fail
    Local disk capacity is limited
    Lower availability
    Limited scalability
    Inconvenient for multi-replica queries
    Not suitable for long-term archiving

Object storage is ideal for storing:

    Chunks
    Indexes
    Ruler rules
    Historical log data

### 16.3 Comparison Between Local Storage and Object Storage

Local storage:

    Suitable for learning purposes
    Simple to configure
    Low cost
    Not suitable for high availability in production

Object storage:

    Suitable for production use
    Highly scalable
    Large capacity
    Ideal for multi-replica and high-availability scenarios
    Slightly more complex to configure

---

## Chapter Seventeen: Loki Writing Pipeline

### 17.1 Writing Pipeline Diagram

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

### 17.2 Explanation of the Writing Process

Step one: Application outputs logs

    Applications write logs to stdout / stderr.

Step two: Container runtime writes files

    containerd stores logs in the local log directory on the node.

Step three: Alloy collects logs

    Alloy runs as a DaemonSet on each node.
    It reads from /var/log/containers/*.log.
    It adds tags such as namespace, pod, container, and node.
    It pushes logs to Loki.

Step four: Distributor receives requests

    The Distributor processes /loki/api/v1/push requests.
    It verifies tags, timestamps, tenants, and throttling settings.

Step five: Ingester writes logs

    The Ingester organizes logs by stream.
    It writes them to WAL.
    It constructs chunks.
    Later, it flushes the data to storage.

Step six: Object storage stores data

    Chunks and index data are ultimately stored in the file system or object storage.

### 17.3 Common Fault Points in the Writing Pipeline

    Applications do not output logs to stdout / stderr
    No logs are found in /var/log/containers/
    Alloy is not scheduled to the correct node
    Alloy### 20.2 Viewing Controllers

Use the following command to check:

```bash
kubectl get deploy,statefulset,daemonset -n logging
```

Focus on:

- Whether Loki is a Deployment or a StatefulSet.
- Whether the Gateway is a Deployment.
- Whether Alloy is a DaemonSet.
- Whether read/write/background services exist separately.

### 20.3 Viewing Services

Use the following command to check:

```bash
kubectl get svc -n logging
```

Focus on:

- Services named `loki`, `loki-gateway`, `loki-headless`, `loki-memberlist`, and `loki-canary`.
- Service names may vary depending on the deployment mode.

### 20.4 Viewing Endpoints

Use the following commands to check:

```bash
kubectl get endpoints -n logging
kubectl get endpointslice -n logging
```

These commands help determine:

- Whether the service has backend Pods.
- Whether the Gateway can forward requests to the backend.
- Whether Loki is ready for use.

### 20.5 Viewing ConfigMap/Secrets

Use the following commands to check Loki's configuration:

```bash
kubectl get cm -n logging
kubectl get secret -n logging
```

To view specific configuration details, use:

```bash
kubectl get cm <loki-configmap-name> -n logging -o yaml
```

Note:

- Do not disclose real Secret values in public notes.
- If the configuration contains access keys or secrets, mask them appropriately.

---

## Chapter 21: Practical Observations III: Checking Loki Pod Logs

### 21.1 Viewing Loki Logs

Use the following command to view logs from a specific Loki pod:

```bash
kubectl logs <loki-pod-name> -n logging --tail=200
```

If there are multiple containers, use:

```bash
kubectl logs <loki-pod-name> -n logging -c <container-name> --tail=200
```

To continuously monitor logs, use:

```bash
kubectl logs <loki-pod-name> -n logging -f
```

### 21.2 Filtering Keywords

Use the following command to filter log messages containing specific keywords:

```bash
kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|timeout|ring|ingester|distributor|querier|compactor|ruler"
```

### 21.3 Analyzing Log Contents

Pay attention to the following aspects:

- Whether Loki has started successfully.
- Whether the configuration was loaded correctly.
- Whether the ring is functioning normally.
- Whether storage initialization was successful.
- Whether the compactor and ruler processes have started.
- Any errors related to object storage connections, permission denied issues, 429 responses, query timeouts, or panics.

### 21.4 Common Error Logs

**Storage-related issues:**
- Incorrect object storage address.
- Bucket not found.
- Access key or secret key errors.
- TLS configuration errors.

**Ring-related issues:**
- Memberlist exceptions.
- Nodes failing to join the ring.
- DNS resolution problems.

**Query-related issues:**
- Query timeouts.
- Excessive query ranges.
- Slow object storage reads.

**Write-related issues:**
- Ingestion rate limits exceeded.
- Label restrictions violated.
- Too many streams.
- Log lines being too large.

---

## Chapter 22: Practical Observations IV: Checking Status via Loki API

### 22.1 Port Forwarding

First, check the Service:

```bash
kubectl get svc -n logging
```

Example of port forwarding:

```bash
kubectl port-forward svc/<loki-service-name> 3100:3100 -n logging
```

If it is exposed through the Gateway, use:

```bash
kubectl port-forward svc/<loki-gateway-service-name> 3100:80 -n logging
```

### 22.2 Checking the "Ready" Status

Use the following command to check if Loki is ready:

```bash
curl -s http://127.0.0.1:3100/ready
```

A successful response usually looks like:

```
ready
```

If it is not ready, check the following:

- Whether the Loki Pod is ready.
- Whether the configuration is correct.
- Whether the ring is functioning normally.
- Whether storage is operational.

### 22.3 Checking Metrics

Use the following command to view Loki metrics:

```bash
curl -s http://127.0.0.1:3100/metrics | head
```

To filter Loki-specific metrics, use:

```bash
curl -s http://127.0.0.1:3100/metrics | grep      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

If no results are obtained, check the following:

    Whether there are logs for app-demo.
    Whether Alloy has successfully collected data.
    Whether the specified query time range is correct.
    Whether the label exists.
    Whether Loki has successfully written the data.

---

## Chapter 25: The Relationship Between Loki Components and Kubernetes Resources

### 25.1 Single-Instance Mode

Possible resources:

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

Explanation:

    In single-instance mode, a single Pod may perform multiple component functions. It is not possible to directly identify components such as distributor, ingester, and querier by the Pod name.

### 25.2 Simple Scalable Mode

Possible resources:

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

Explanation:

    The read component is responsible for queries.
    The write component handles data writes.
    The backend manages background tasks.
    The gateway serves as a unified entry point.

### 25.3 Microservices Mode

Possible resources:

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

Explanation:

    Each component runs independently in this mode, making it suitable for large-scale production. However, troubleshooting requires identifying issues at the component level.

---

## Chapter 26: Monitoring Loki's Own Metrics

As a logging system, Loki itself also needs to be monitored.

### 26.1 Viewing Loki Metrics

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

### 26.2 Common Metric Categories

**Writing-related metrics:**

    Number of requests received by the distributor
    Number of write failures of the distributor
    Number of logs received by the ingester
    Active streams of the ingester
    Number of flush operations performed by the ingester
    Flush failures of the ingester

**Querying-related metrics:**

    Number of queries processed by the querier
    Query queue length of the query frontend
    Query latency
    Query timeout
    Cache hit/miss ratio

**Storage-related metrics:**

    Number of chunk writes
    Number of chunk reads
    Number of compactor executions
    Number of retention deletions
    Object storage errors

**Rule-related metrics:**

    Number of rule executions by the ruler
    Number of alert notifications sent by the ruler
    Rule execution failures

### 26.3 Production Environment Monitoring Requirements for Loki

At minimum, the following metrics should be monitored:

    Whether the Loki Pod is ready.
    Whether there are write failures in Loki.
    Whether query operations in Loki time out.
    An increase in Loki 429 errors.
    Number of ingester streams.
    Memory usage of the ingester.
    Normal operation of the compactor.
    Proper functioning of the ruler.
    Object storage access errors.
    Gateway 5xx responses.
    Disk/PVC utilization rate of Loki.
    Capacity of Loki's object storage.

---

## Chapter 27: The Relationship Between Loki and Alloy

### 27.1 The Role of Alloy

In the Loki architecture, Alloy acts as the collection agent, not a server component of Loki.

**Linkage:** 

    Pod logs
      ↓
    Alloy
      ↓
    Loki

Alloy is responsible for:

    Identifying Pods.
    Reading log files.
    Adding labels to logs.
    Forwarding logs.
    Managing label reassignment.
    Controlling the scope of data collection.

Loki is responsible for:

    Receiving logs.
    Storing logs.
    Querying logs.
    Generating log alerts.

### 27.2 Monitoring Alloy

After deploying Alloy, monitor the following:

    `kubectl get ds -n logging`

    `kubectl get pods -n logging -o wide`

    `kubectl logs <alloy-pod> -n logging`

Check whether:

    Each node of Alloy has a corresponding Pod.
    Alloy is mounted to /var/log/containers.
    Alloy can successfully connect to Loki.
    There are no errors related to log transmission failures.
    Alloy correctly adds namespace/pod/container labels.

---

## Chapter 28: The Relationship Between Loki and Grafana

### 28.1 Grafana as a Query Entry Point

GrafanaThe Loki Query API is unreachable.
The query range is too large.
There is a syntax error in LogQL.
The Querier resources are insufficient.
Reading from Object Storage failed.

### 31.3 Logs have expired but not been deleted

Priority checks:

    retention configuration
    compactor
    object storage permissions

Possible reasons:

    retention is not enabled
    the compactor is not running
    the compactor does not have deletion permissions
    the configuration is not taking effect
    there is an error in the multi-replica compactor configuration

### 31.4 Alarms are not triggered

Priority checks:

    whether the Ruler is enabled
    whether the rule files have been loaded
    whether LogQL is correct
    whether the AlertManager address is correct
    whether there are any errors in Ruler logs

Possible reasons:

    the Ruler is not enabled
    the rule path is incorrect
    there is a syntax error in the rules
    no data was found in the query
    the AlertManager is unreachable

---

## Section Thirty-Two: Practical Tasks

These practical tasks are divided into two categories.

### 32.1 When Loki has not been deployed

Tasks to complete:

    [ ] Add the Grafana Helm repository
    [ ] Use `helm search repo grafana/loki --versions`
    [ ] Use `helm show values grafana/loki > values-loki-default.yaml`
    [ ] Use `helm template loki grafana/loki -n logging > loki-rendered.yaml`
    [ ] Check the `deploymentMode` in `values`
    [ ] Check `StatefulSet`, `Deployment`, and `Service` in the rendered YAML
    [ ] Understand the differences between resources in monolithic and split modes

Command summary:

    `helm repo add grafana https://grafana.github.io/helm-charts`

    `helm repo update`

    `helm search repo grafana/loki --versions`

    `helm show values grafana/loki > values-loki-default.yaml`

    `helm template loki grafana/loki \
      -n logging \
      > loki-rendered.yaml`

    `grep -n "deploymentMode" values-loki-default.yaml`

    `grep -n "kind:" loki-rendered.yaml`

### 32.2 When Loki has been deployed

Tasks to complete:

    [ ] Check the Loki Pod
    [ ] Check the Loki Service
    [ ] Check the Loki StatefulSet/Deployment
    [ ] Check the Loki ConfigMap
    [ ] View Loki logs
    [ ] Port-forward Loki
    [ ] Access `/ready`
    [ ] Access `/metrics`
    [ ] Try accessing `/ring`
    [ ] Query labels
    [ ] Execute a `query_range`

Command summary:

    `kubectl get pods -n logging -o wide`

    `kubectl get svc -n logging`

    `kubectl get deploy,statefulset -n logging`

    `kubectl logs <loki-pod-name> -n logging --tail=200`

    `kubectl port-forward svc/<loki-service-name> 3100:3100 -n logging`

    `curl -s http://127.0.0.1:3100/ready`

    `curl -s http://127.0.0.1:3100/metrics | head`

    `curl -s http://127.0.0.1:3100/loki/api/v1/labels`

---

## Section Thirty-Three: Acceptance Checklist

After completing this document, you should be able to:

    [ ] Clearly explain the function of the Distributor
    [ ] Clearly explain the function of the Ingester
    [ ] Clearly explain the function of the Querier
    [ ] Clearly explain the function of the Query Frontend
    [ ] Clearly explain the function of the Compactor
    [ ] Clearly explain the function of the Ruler
    [ ] Clearly explain the function of the Gateway
    [ ] Clearly understand the role of Object Storage
    [ ] Draw the Loki writing pipeline
    [ ] Draw the Loki querying pipeline
    [ ] Understand that components in monolithic mode are not necessarily independent Pods
    [ ] Understand the meaning of read/write/backend in Simple Scalable mode
    [ ] Understand that Microservices mode is suitable for large-scale production
    [ ] Be able to view Loki resources using `kubectl`
    [ ] Be able to view startup and error information of Loki using logs
    [ } Be able to determine if Loki is available by checking `/ready`
    [ ] Be able to view Loki's own metrics using `/metrics`
    [ ] Understand that `/config` may contain sensitive information and should not be publicly exposed
    [ ] Know that Alloy is the collectionIndex compression, retention periods, and cleanup of expired data.

Ruler:
Log alert rules and Recording Rules.

Gateway:
Unified entry point and routing.

Object Storage:
Long-term storage of log data.

The key to understanding the Loki architecture is not simply memorizing component names, but being able to determine during a failure:

Is it a collection issue?
Is it a writing issue?
Is it a querying issue?
Is it a storage issue?
Is it a rule-related issue?
Is it a display-related issue?

In the next article, we will explore:

03 - Comparison of Loki Deployment Modes and Experimental Selection

Key comparisons will include:

Monolithic mode
Simple Scalable mode
Microservices mode

We will also explain why this series starts with the monolithic mode before gradually moving on to object storage and distributed deployment.

---

## References

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
  https://grafana.com/docs/loki/latest setup/install/helm/install-monolithic/

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