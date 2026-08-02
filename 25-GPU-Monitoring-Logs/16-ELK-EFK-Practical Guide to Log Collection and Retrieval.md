# 16-ELK-EFK-Practical Guide to Log Collection and Retrieval

## Document Description

This document systematically outlines the architecture, deployment, collection, indexing, retrieval, visualization, alerting, capacity planning, and troubleshooting methods for the ELK/EFK log system in a Kubernetes environment.

The focus of this document is not simply on how to "install Elasticsearch," but rather on understanding it from an operations/SRE perspective:

- Why ELK/EFK are needed;
- What scenarios each component (ELK, EFK, Loki) is suitable for;
- The role of Elasticsearch/OpenSearch in log systems;
- The responsibilities of Filebeat, Fluent Bit, Fluentd, and Logstash;
- Where Kubernetes container logs come from;
- How to collect `/var/log/containers` logs;
- How to add metadata such as Namespace, Pod, Container, and Node to logs;
- How to plan Indexes/Data Streams/Index Templates;
- How to design fields, Mappings, ECS, and time fields;
- How to query logs using Kibana/OpenSearch Dashboards;
- How to handle issues like index expansion, disk outages, high cardinality fields, and uncontrolled log volumes;
- How to design log alerts;
- How to use Prometheus, Grafana, AlertManager, and Loki for a closed-loop troubleshooting process;
- How to manage log retention, permission isolation, security masking, and capacity management in production environments.

This document is recommended for those who have completed the following topics:

- 11-Prometheus-Architecture and Core Metric Analysis
- 12-Grafana-Dashboard Construction and Custom Monitoring
- 13-AlertManager-Alert Strategies and Notification Implementation
- 14-K8S-Monitoring Practice-Node-Pod-Service Metric Collection and Troubleshooting
- 15-Loki-Log Collection and Query Practice

---

## Tags

#ELK #EFK #Elasticsearch #OpenSearch #Kibana #OpenSearchDashboards #Filebeat #FluentBit #Fluentd #Logstash #Kubernetes #Log Collection #Log Retrieval #SRE #Observability

---

## Recommended Reading Path

Recommended path:

    06-GPU and AI Infrastructure/05-Fundamentals of Observability/16-ELK-EFK-Practical Guide to Log Collection and Retrieval.md

---

## I. Why Learn ELK/EFK?

In the previous section on Loki, it was explained that Loki is very suitable for Kubernetes operations log troubleshooting.

However, in production environments, ELK/EFK are still widely used.

The reasons are:

- Many organizations already have Elasticsearch/OpenSearch log platforms;
- Kibana/OpenSearch Dashboards offer powerful full-text search capabilities;
- Audit, security, and business log retrieval often rely on full-text searches;
- Elasticsearch/OpenSearch have mature capabilities for structured field queries and aggregations;
- Many log collection solutions naturally support output to Elasticsearch/OpenSearch;
- ELK/EFK architectures are frequently encountered in interviews and production work.

Loki is more like:

    A system for aggregating Kubernetes operation logs
    Friendly for linking metrics for troubleshooting
    Relatively cost-effective
    Mainly uses tag indexing

ELK/EFK, on the other hand, are more like:

    A full-text log retrieval platform
    Strong in field search and aggregation analysis
    Suitable for audit, security, and business log retrieval
    Requires higher levels of resource and index management

For operations engineers, it is essential to understand not only Loki but also:

    Log collection
      ↓
    Log parsing
      ↓
    Field standardization
      ↓
    Index writing
      ↓
    Query analysis
      ↓
    Lifecycle management
      ↓
    Permission isolation
      ↓
    Alerts and troubleshooting

---

## II. Basic Concepts of ELK/EFK/ECK/OpenSearch

### 2.1 What is ELK?

ELK typically refers to:

    Elasticsearch
    Logstash
    Kibana

Their roles are:

    Elasticsearch:
        Logs storage, indexing, searching, and analysis engine.

    Logstash:
        Logs reception, filtering, parsing, transformation, and output pipeline.

    Kibana:
        Querying, visualization, Dashboard, and management interface.

Typical workflow:

    App logs
      ↓
    Logstash
      ↓
    Elasticsearch
      ↓
    Kibana

### 2.2 What is EFK?

EFK typically refers to:

    Elasticsearch
    Fluentd/Fluent Bit
    Kibana

Some also include OpenSearch + Fluent Bit + OpenSearch Dashboards in the EFK category.

Their roles are:

    Elasticsearch/OpenSearch:
        Logs storage, indexing, searching, and analysis.

    Fluentd/Fluent Bit:
        Logs collection, filtering, parsing, and forwarding.

    Kibana/OpenSearch Dashboards:
        Logs querying### ELK / EFK are More Powerful.

---

## IV. Sources of Kubernetes Container Logs

### 4.1 Standard Log Output

Kubernetes recommends logging container logs to:

    stdout
    stderr

During container execution, logs are written locally on the node.

Common paths in a containerd environment:

    /var/log/containers/
    /var/log/pods/

Example:

    /var/log/containers/nginx-xxx_default_nginx-abc123.log

To view these logs:

    ls -l /var/log/containers/
    ls -l /var/log/pods/

### 4.2 Why Use DaemonSet for Log Collection Agents

In Kubernetes, log collection agents are typically run as DaemonSets.

Reasons:

    Each node has its own container log files.
    Logs need to be collected locally on each node.
    The agent needs to mount the host's log directory.
    When pods are dynamically migrated, the agent on the new node continues collecting logs.

Typical DaemonSets include:

    Filebeat DaemonSet
    Fluent Bit DaemonSet
    Fluentd DaemonSet

### 4.3 Basic Log Collection Process

    Pod stdout/stderr
      ↓
    containerd writes logs to local node files
      ↓
    /var/log/containers/*.log
      ↓
    Filebeat / Fluent Bit / Fluentd
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 4.4 Recommendations for Application Log Output

It is recommended to:

    Output application logs via stdout/stderr.
    Use JSON log format.
    Include fields such as service, level, trace_id, request_id in the logs.
    Avoid disclosing sensitive information.
    Control the production of DEBUG-level logs.

It is not recommended to:

    Only write logs to internal container files.
    Only store logs on local disks.
    Print large amounts of unstructured text.
    Include passwords, tokens, cookies, or Authorization headers in logs.
    Print entire request and response bodies.

---

## V. Comparison of Log Collection Components

### 5.1 Filebeat

Filebeat is a lightweight log collector within the Elastic Beats ecosystem.

Features:

- Lightweight;
- Suitable for collecting file-based logs;
- Supports Kubernetes DaemonSets;
- Supports autodiscovery;
- Can be used to output logs to Elasticsearch, Logstash, Kafka, etc.;
- Integrates well with the Elastic Stack.

Suitable for:

- Elastic Stack environments;
- Collecting container logs;
- Collecting system logs;
- For lightweight log forwarding.

### 5.2 Fluent Bit

Fluent Bit is a low-footprint log collection and forwarding tool.

Features:

- Requires minimal resources;
- Implemented in C language;
- Suitable for container and edge environments;
- Supports various inputs and outputs;
- Handles Kubernetes metadata;
- Commonly used for Kubernetes log collection.

Suitable for:

- Collecting Kubernetes DaemonSet logs;
- Large-scale environments with many nodes;
- Resource-constrained scenarios;
- Outputting logs to Elasticsearch / OpenSearch / Loki / Kafka.

### 5.3 Fluentd

Fluentd is more resource-intensive than Fluent Bit but offers a richer plugin ecosystem.

Features:

- Implemented in Ruby;
- Comes with numerous plugins;
- Has stronger processing capabilities;
- Requires more resources than Fluent Bit;
- Often used as an aggregation layer.

Common configuration:

    Fluent Bit DaemonSet
      ↓
    Fluentd Aggregator
      ↓
    Elasticsearch / OpenSearch

### 5.4 Logstash

Logstash is a data processing pipeline within the Elastic Stack.

Features:

- Powerful filtering capabilities;
- Supports grok, mutate, date, geoip, and other transformations;
- Offers many plugins;
- Requires relatively more resources;
- Often used as a centralized processing layer.

Common configuration:

    Filebeat
      ↓
    Logstash
      ↓
    Elasticsearch

Suitable for:

- Complex log parsing;
- Unified processing of logs from multiple sources;
- Scenarios requiring extensive filtering;
- Environments needing multiple output targets.

### 5.5 Selection Recommendations

For lightweight Kubernetes log collection:

    Fluent Bit

For native Elastic Stack log collection:

    Filebeat

For complex log processing:

    Logstash

For large-scale scenarios:

    Collect logs with Fluent Bit, buffer them in Kafka, then process them with Logstash/Fluentd and store the results in Elasticsearch/OpenSearch.

For learning purposes:

    Either Filebeat or Fluent Bit is suitable as a starting point.

---

## VI. Typical Architecture 1: Filebeat + Elasticsearch + Kibana

### 6.1 Architecture Diagram

    Kubernetes Pod
      ↓
    /var/log/containers
      ↓
    Filebeat DaemonSet
      ↓
    Elasticsearch
      ↓
    Kibana

### 6.2 Applicable Scenarios

Suitable for:

- Using the Elastic Stack;
- Collecting Kubernetes container logs;
- Needing quickIn OpenSearch, the primary node role is commonly referred to as cluster_manager.

### 10.3 Index

An index is a collection of documents.

Common examples in logging systems include:

    logs-prod-2026.04.30
    k8s-app-2026.04.30
    nginx-access-2026.04.30

### 10.4 Document

A document is a record written into a search engine.

A single log entry typically constitutes one document.

Example:

    {
      "@timestamp": "2026-04-30T12:00:00+08:00",
      "message": "database connection failed",
      "log.level": "error",
      "kubernetes.namespace": "prod",
      "kubernetes.pod.name": "api-xxx"
    }

### 10.5 Field

A field is a specific piece of information within a document.

Examples include:

    @timestamp
    message
    log.level
    service.name
    kubernetes.namespace
    kubernetes.pod.name
    host.name

### 10.6 Mapping

Mapping defines the type of data stored in each field.

For example:

    @timestamp:
        date

    message:
        text

    log.level:
        keyword

    duration_ms:
        long

    status:
        integer

Incorrect mapping design can lead to:

- Failed queries;
- Incorrect aggregation results;
- Field type conflicts;
- Issues with index writing;
- Inaccurate display in Kibana.

### 10.7 Shard

A shard is a portion of an index.

An index can have multiple primary shards.

Too many shards can result in:

- Increased cluster complexity;
- Higher memory consumption;
- Slower query performance;
- Excessive number of small indexes;
- Greater load on the master node.

Too few shards can cause:

- A single shard becoming too large to manage effectively;
- Insufficient concurrent querying and writing capabilities;
- Slow recovery times.

### 10.8 Replica

A replica is a copy of a shard.

Their roles include:

- Improving availability;
- Increasing concurrent query capacity;
- Ensuring functionality even in the event of node failures.

The more replicas there are, the higher the availability and storage costs will be, as well as the greater the writing load on the system.

### 10.9 Data Stream

Data streams are designed for time-series data, such as logs, metrics, and events.

They can hide underlying backing indices and manage log retention through rollover and lifecycle policies.

It is recommended to prioritize understanding data streams for production logging systems.

### 10.10 ILM / ISM

In Elasticsearch, Index Lifecycle Management (ILM) is commonly used.

In OpenSearch, Index State Management (ISM) is similar in purpose.

Both tools are used to manage the lifecycle of indexes.

Examples include:

    hot
      ↓
    warm
      ↓
    cold
      ↓
    delete

Log systems must implement lifecycle management to prevent disk space from being quickly occupied.Testing Environment:

    7d - 15d

Production Application Logs:

    15d - 30d

Audit Logs:

    90d / 180d / 1y, depending on compliance requirements

Debug Logs:

    1d - 3d

### 14.4 Rollover Policy

Trigger conditions can be based on:

    Time
    Index Size
    Number of Documents

Example Design:

    Rollover when a single index reaches 30GB
    Or rollover every 1 day

Production Recommendations:

    Do not allow a single shard to become too large.
    Avoid having too many small indexes.
    Determine these settings after testing the volume of logs.

---

## Fifteen, Practical Filebeat Kubernetes Collection

### 15.1 Core Concept of Filebeat DaemonSet

Filebeat operates as a DaemonSet.

Mount Points:

    /var/log/containers
    /var/log/pods

After collecting logs, they are sent to:

    Elasticsearch
    Logstash
    Kafka

### 15.2 Example Filebeat Configuration

Example Structure:

    filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
        processors:
          - add_kubernetes_metadata:
              host: ${NODE_NAME}
              matchers:
                - logs_path:
                    logs_path: "/var/log/containers/"

    output.elasticsearch:
      hosts: ["http://elasticsearch.logging.svc:9200"]
      index: "logs-k8s-app-%{+yyyy.MM.dd}"

    setup.template.name: "logs-k8s-app"
    setup.template.pattern: "logs-k8s-app-*"

Notes:

    In production, it is recommended to use data streams or specific index templates.
    If Elasticsearch requires authentication, configure a username/password or API key.
    In HTTPS environments, configure the CA certificate.

### 15.3 Key Mount Points for Filebeat DaemonSet

Example Highlights:

    volumeMounts:
      - name: varlog
        mountPath: /var/log
        readOnly: true

      - name: varlibdockercontainers
        mountPath: /var/lib/docker/containers
        readOnly: true

For containerd environments:

    /var/log/containers
    /var/log/pods

Docker environments may also require:

    /var/lib/docker/containers

### 15.4 Filebeat Environment Variables

Common ones include:

    NODE_NAME

Example:

    env:
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName

This is used to add node information to the logs.

### 15.5 Troubleshooting Filebeat

To check a Pod:

    kubectl get pod -n logging -o wide | grep filebeat

To view logs:

    kubectl logs <filebeat-pod> -n logging

To enter a Pod:

    kubectl exec -it <filebeat-pod> -n logging -- sh

To check mount points:

    ls -l /var/log/containers/

Common errors include:

- Unreachable Elasticsearch address;
- Authentication failure;
- TLS certificate issues;
- Insufficient index permissions;
- Mapping conflicts;
- Logs not mounted correctly;
- Unable to retrieve Kubernetes metadata;
- Output end experiencing throttling or write failures.

---

## Sixteen, Practical Fluent Bit Kubernetes Collection

### 16.1 Architecture of Fluent Bit

Fluent Bit configurations typically include:

    INPUT
    FILTER
    OUTPUT

Process Flow:

    INPUT tail collects files
      ↓
    FILTER kubernetes adds metadata
      ↓
    FILTER parser parses logs
      ↓
    OUTPUT sends to Elasticsearch / OpenSearch

### 16.2 INPUT Example

For collecting container logs:

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

### 16.3 FILTER Example

For adding Kubernetes metadata:

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_FILE     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging Parser  On
        K8S-Logging.Exclude On

### 16.4 OUTPUT to Elasticsearch

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch.logging.svc
        Port            9200
        Index           logs-k```markdown
input {
  beats {
    port => 5044
  }
}

filter {
  json {
    source => "message"
  }

  mutate {
    lowercase => [ "[log][level]" ]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch.logging.svc:9200"]
    index => "logs-k8s-app-%{+YYYY.MM.dd}"
  }
}

### 17.2 Common Filters

json:

    Parses JSON logs.

grok:

    Parses unstructured text logs.

date:

    Parses log time as @timestamp.

mutate:

    Renames fields, converts types, and handles case conversion.

drop:

    Discards useless logs.

geoip:

    Parses IP geographic information.

### 17.3 Grok Examples

Nginx logs can be parsed using grok.

Example:

    filter {
      grok {
        match => {
          "message" => "%{IPORHOST:client_ip} - %{DATA:user} \\[%{HTTPDATE:timestamp}\\] \"%{WORD:method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:status:int} %{NUMBER:bytes:int}"
        }
      }

      date {
        match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
      }
    }

### 17.4 Logstash Usage Recommendations

Suitable for:

- Logs with complex formats;
- Need for unified processing;
- Multiple output requirements;
- Complex grok patterns;
- Need to centrally manage fields.

Not suitable for:

- Extremely limited resources;
- Simple direct collection of K8S logs;
- Small clusters that do not require advanced processing.
---

## Chapter Eighteen: Practical Queries with Kibana / OpenSearch Dashboards

### 18.1 Creating Data Views / Index Patterns

In Kibana:

    Stack Management
      ↓
    Data Views
      ↓
    Create data view

Example of an index pattern:

    logs-k8s-app-*

Time field:

    @timestamp

Similar in OpenSearch Dashboards:

    Stack Management / Dashboards Management
      ↓
    Index Patterns
      ↓
    Create index pattern

### 18.2 Discover Queries

In Kibana:

    Discover
      ↓
    Select Data View
      ↓
    Enter query criteria
      ↓
    Set time range

Common queries:

    kubernetes.namespace : "prod"

    kubernetes.pod.name : "api-*"

    log.level : "error"

    message : "connection refused"

### 18.3 KQL Query Examples

Query for error logs in the prod namespace:

    kubernetes.namespace : "prod" and log.level : "error"

Query for a specific Pod:

    kubernetes.namespace : "prod" and kubernetes.pod.name : "api-xxx"

Query for messages containing "timeout":

    message : "timeout"

Query for 5xx errors:

    http.response.status_code >= 500

Query for a particular service:

    service.name : "order-api"

### 18.4 Lucene Query Examples

    kubernetes.namespace:prod AND log.level:error

    message:"connection refused"

    http.response.status_code:[500 TO 599]

### 18.5 Time Range

Log queries must first specify a time range.

Common options:

    Last 15 minutes
    Last 1 hour
    Last 6 hours
    Last 24 hours

Do not start by querying all logs from the past 30 days.
---

## Chapter Nineteen: Basics of Query DSL

### 19.1 Match Queries

    GET logs-k8s-app-*/_search
    {
      "query": {
        "match": {
          "message": "connection refused"
        }
      }
    }

### 19.2 Term Queries

Used for precise matching of keywords:

    GET logs-k8s-app-*/_search
    {
      "query": {
        "term": {
          "log.level": "error"
        }
      }
    }

### 19.3 Bool Queries

    GET logs-k8s-app-*/_search
    {
      "query": {
        "bool": {
          "must": [
            { "term": { "kubernetes.namespace": "prod" } },
            { "term": { "log.level": "error" } }
          ]
        }
      }
    }

### 19.4 Range Queries

Queries for recent periods are usually handled by Kibana's time selector.

API example:

    GET logs-k8s-app-*/_search
    {
      "query": {
        "range": {
          "@timestamp": {
            "gte": "now-15m",
            "lte": "now"
          }
        }
      }
    }

### 1```markdown
kubernetes.namespace : "prod" and service.name : "order-api" and log.level : "error"

Search for:

- timeout;
- connection refused;
- database error;
- redis error;
- permission denied;
- upstream unavailable.

### 21.3 Confirming Kubernetes Events

    Use `kubectl describe pod <pod-name> -n <namespace>` to check.
    Use `kubectl get events -n <namespace> --sort-by=.lastTimestamp` for more details.

### 21.4 Full Path

    Prometheus alarm detected an issue.
      ↓
    Grafana monitors trends.
      ↓
    Kibana analyzes error logs.
      ↓
    Use `kubectl describe` to view events.
      ↓
    Use `kubectl logs --previous` for additional confirmation.
      ↓
    Repair configuration / Roll back / Scale up / Restore dependencies.
      ↓
    Monitor metric and log recovery progress.
      ↓
    Conduct a post-event review.

---

## Chapter 22: GPU/AI Log Retrieval Scenarios

### 22.1 Common GPU/AI Log Keywords

Search for:

    CUDA out of memory;
    RuntimeError;
    Traceback;
    NCCL;
    model load failed;
    checkpoint;
    no space left on device;
    connection refused;
    timeout;
    failed to allocate memory;
    CUBLAS;
    CUDNN.

### 22.2 Searching for CUDA OOM in Kibana

KQL query:

    kubernetes.namespace : "ai-prod" and message : "CUDA out of memory"

### 22.3 Searching for PyTorch Exceptions

    kubernetes.namespace : "ai-prod" and message : "RuntimeError"

### 22.4 Searching for Model Load Failures

    kubernetes.namespace : "ai-prod" and message : "model load failed"

### 22.5 Monitoring GPU Metrics

Use Prometheus to check:

    DCGM_FI_DEV_FB_used;
    DCGM_FI_DEV_GPU_UTIL;
    DCGM_FI_DEV_GPU_TEMP;
    DCGM_FI_DEV_XID_errors;

For node troubleshooting, use:

    `nvidia-smi`;
    `dmesg | grep -i xid`;
    `journalctl -k | grep -i nvidia`.

For Pod troubleshooting, use:

    `kubectl describe pod <pod-name> -n ai-prod`;
    `kubectl logs <pod-name> -n ai-prod --previous`.

### 22.6 GPU Troubleshooting Guidelines

If CUDA OOM is reported in the logs:

    Check显存 usage first.
    Verify batch size and concurrency settings.
    Examine model size and number of workers.
    Check if multiple models are being loaded repeatedly.

If NCCL issues arise:

    Investigate multi-GPU communication and network settings.
    Check RDMA configuration and GPU topology.
    Evaluate distributed training setup.

If a model load failure is reported:

    Verify the path to the model file.
    Check object storage permission settings.
    Ensure PVC mounting is correct.
    Confirm the file is not damaged.
    Verify image and dependency versions.

---

## Chapter 23: Log Security and Masking

### 23.1 Information That Should Not Be Published

Prohibit the disclosure of:

    passwords;
    tokens;
    access keys;
    secret keys;
    private keys;
    Authorization headers;
    Cookies;
    identification documents;
    bank card details;
    plaintext phone numbers or email addresses;
    user privacy data;
    database connection strings.

### 23.2 Masking Locations and Priorities

Priority order:

    Prevent sensitive information from being displayed on the application side.
      ↓
    Filter data at the SDK/log library level.
      ↓
    Mask data before it is collected by agents.
      ↓
    Apply masking in Logstash/Fluentd pipelines.
      ↓
    Implement index access control measures.

Do not rely solely on log platforms to remove sensitive information afterward.

### 23.3 Masking Examples

Phone number:

    13812345678
      ↓
    138****5678

Token:

    Bearer abcdefghijklmn
      ↓
    Bearer ***

### 23.4 Permission Isolation

Production log platforms must ensure:

    Who can view production logs;
    Who has access to sensitive namespaces;
    Who is authorized to export logs;
    Who can delete index data;
    Who can modify index templates;
    Who can configure alerts;
    Who can access audit logs.

Log system permissions should not be overly permissive.

---

## Chapter 24: Log Volume Management

### 24.1 Common Causes of Uncontrolled Log Volumes

Common reasons include:

- Enabling DEBUG logging level;
- Printing the entire request body for each request;
- Printing the entire response body for each response- Unstable nodes;
- Risk of Out Of Memory (OOM).CUDA Out of Memory
A surge in authentication failures

### 31.3 Alarm Classification

Critical:

    Log platform is unreadable
    Cluster is in red status
    A massive increase in production service error logs affecting business operations
    Disk is nearly full
    Large amount of logs lost

Warning:

    Cluster is in yellow status
    High JVM heap usage
    Slow query performance
    Agent sending failures
    An increase in ERROR logs for a certain service

Info:

    Excessive log volume in a specific namespace
    Too many DEBUG level logs
    About to reach the retention period
    Reminder for managing top log sources

---

## Case Study 32: Pod is Running Normally but Business Errors with 500 Status Codes Have Increased

### 32.1 Symptoms

Prometheus alarm:

    ServiceHigh5xxErrorRate

Kubernetes inspection:

    Pod is running normally
    Service is functioning correctly
    Endpoints are operational

### 32.2 Kibana Query

KQL:

    kubernetes.namespace : "prod"
    and service.name : "order-api"
    and http.response.status_code >= 500

Check the error logs.

### 32.3 Further Filtering

Query database exceptions:

    kubernetes(namespace : "prod"
    and service.name : "order-api"
    and message : "database"

Query timeout issues:

    kubernetes.namespace : "prod"
    and service.name : "order-api"
    and message : "timeout"

Search for recently released versions:

    kubernetes.namespace : "prod"
    and service.name : "order-api"
    and service.version : "v1.2.3"

### 32.4 kubectl Verification

    kubectl describe pod <pod-name> -n prod
    kubectl logs <pod-name> -n prod --previous
    kubectl get events -n prod --sort-by=.lastTimestamp

### 32.5 Possible Root Causes

- Bug in the new version;
- Database connection pool is full;
- Redis timeout;
- Downstream services are unavailable;
- Configuration changes caused errors;
    Secret updates went wrong;
    NetworkPolicy restrictions;
    DNS issues.

### 32.6 Solutions

- Roll back to a previous stable version;
- Fix configuration errors;
- Restore downstream services;
- Increase the capacity of the connection pool;
    Adjust network policies;
    Monitor recovery of 5xx errors and log volume;
    Document and add additional alarms as needed.

---

## Case Study 33: Log Platform Writing Failures

### 33.1 Symptoms

Filebeat/Fluent Bit logs show:

    429 Too Many Requests
    403 Forbidden
    400 mapper_parsing_exception
    connection refused
    no living connections

### 33.2 Checking the Collection Endpoints

    kubectl logs <agent-pod> -n logging
    kubectl describe pod <agent-pod> -n logging

Verify:

- Whether all nodes are experiencing issues;
- If there are any single-node failures;
- Whether the output endpoints are unreachable;
    If authentication failures occur;
    If there are mapping conflicts;
    If rate limiting is in effect.

### 33.3 Checking Elasticsearch/OpenSearch

    GET _cluster/health
    GET _cat/nodes?v
    GET _cat/indices?v
    GET _cat/thread_pool?v
    GET _cat/allocation?v

### 33.4 Determining the Cause

429:

    High writing load or rate limiting.

403:

    Insufficient permissions.

400 mapping error:

    Field type conflicts.

connection refused:

    Service is unavailable.

no living connections:

    All output endpoints are down or there are network issues.

### 33.5 Solutions

- Add more data nodes;
    Fix permission issues;
    Adjust index templates;
    Create new indexes;
    Resolve mapping conflicts;
    Reduce log volume;
    Implement temporary rate limiting;
    Resume monitoring after repairs.

---

## Summary of Operations Commands

### 34.1 Kubernetes Side

Check collection components:

    kubectl get pods -n logging -o wide
    kubectl get ds -n logging
    kubectl get svc -n logging

View logs:

    kubectl logs <filebeat-pod> -n logging
    kubectl logs <fluent-bit-pod> -n logging
    kubectl logs <logstash-pod> -n logging

Check node log paths:

    ls -l /var/log/containers/
    ls -l /var/log/pods/

View business logs:

    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

### 34.2 Elasticsearch/OpenSearch API

Cluster health check:

    GET _cluster/health

Nodes information:

    GET _[ ] Index Template is configured.
[ ] Core fields for Mapping are correct.
[ ] ILM / ISM have been configured.
[ ] Rollover settings are in place.
[ ] Retention periods have been defined.
[ ] The number of shards is appropriate.
[ ] The number of replicas is reasonable.
[ ] Disk usage alerts have been set up.

### 36.4 Security Governance

[ ] Kibana / Dashboard use HTTPS.
[ ] Authentication is enabled.
[ ] Permissions are minimized.
[ ] Sensitive fields are masked.
[ ] Tokens / passwords are not logged.
[ ] Log export permissions are controlled.
[ ] Index deletion permissions are restricted.
[ ] Audit logs are available for review.

### 36.5 Operations Governance

[ ] Agent resource limits have been set.
[] Alerts are triggered for failed Agent transmissions.
[ } Alerts indicate the health of the log platform cluster.
[ ] Statistics are maintained for top log volumes.
[ ] Debug logs are managed properly.
[ ] Runbooks have been prepared.
[ ] Dashboards have been established.
[ ] Alerts are integrated with notification channels.

---

## Section Thirty-Seven: Common Misconceptions

### 37.1 Misconception 1:Installing ELK means the log platform is complete

Wrong.

A log platform also requires:

- Field standards;
- Index templates;
- Lifecycle management;
- Permission controls;
- Data masking;
- Alerting mechanisms;
- Capacity planning;
- Runbooks.

### 37.2 Misconception 2: All logs should be permanently retained

Wrong.

Log retention should be tiered:

- Debug logs for short-term storage;
- Application troubleshooting logs for medium-term retention;
- Audit logs for long-term archiving;
- Security-related logs according to compliance requirements.

### 37.3 Misconception 3: It's best to create one index per service

Not necessarily.

Too many indexes can lead to exponential growth in shards and storage costs.

Indexes should be designed based on log volume, query needs, and permission models.

### 37.4 Misconception 4: The more fields, the better

Wrong.

Excessive fields can cause Mapping inefficiencies and increased storage costs.

Standardizing core fields is sufficient.

### 37.5 Misconception 5: Simply deleting some indexes when Elasticsearch's disk space is full is enough

Incomplete.

Additional steps are needed:

- Check ILM settings;
- Analyze log source volumes;
- Assess shard and replica configurations;
- Monitor disk usage levels;
- Verify index read-only status;
- Implement additional alerts and capacity planning measures.

### 37.6 Misconception 6: A log system can replace metric monitoring

Wrong.

Log systems are suitable for issue identification, while metric systems are better for trend analysis, alerting, and capacity planning.

Combining both is essential for effective production troubleshooting.

---

## Section Thirty-Eight: Summary

ELK / EFK constitute a crucial log retrieval and analysis framework in production environments. Its core workflow includes:

    Application stdout/stderr
      ↓
    Container runtime writes to node logs
      ↓
    /var/log/containers
      ↓
    Filebeat / Fluent Bit / Fluentd
      ↓
    Logstash / Kafka
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards
      ↓
    Queries / Dashboards / Alerts / Auditing

Compared to Loki:

    Loki is more lightweight and better suited for Kubernetes operations and integration with Grafana.

    ELK / EFK offer stronger querying capabilities, making them ideal for full-text searches, field analysis, auditing, and business log retrieval.

In production settings, the focus should not be solely on logging but rather on:

- Standardizing fields;
- Ensuring stable Mapping;
- Reasonable index design;
- Controllable lifecycle management;
- Stable query performance;
    Clear permission boundaries;
    Protecting sensitive information;
    Managing log volumes effectively;
    Ensuring closed-loop alerts;
    Predicting capacity needs.

For troubleshooting, the recommended approach is:

    Prometheus identifies anomalies in metrics.
    Grafana monitors trends.
    Kibana / OpenSearch Dashboards provide log queries.
    kubectl describe reveals event details.
    kubectl logs --previous confirms issues.
    Elasticsearch / OpenSearch APIs assess platform health.
    Fix faults and optimize log management and alerts.

For Ops/SRE professionals, mastering ELK / EFK means understanding the entire production chain from log generation to storage and security compliance.