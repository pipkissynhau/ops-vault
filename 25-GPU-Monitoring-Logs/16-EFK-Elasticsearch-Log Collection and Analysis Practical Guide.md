# 16-ELK-EFK-Log Collection and Retrieval in Practice

## Document Description

This document systematically organizes the architecture, deployment, collection, indexing, retrieval, visualization, alerting, capacity planning, and production troubleshooting methods for the ELK/EFK log system in a Kubernetes environment.

This document focuses not on simply explaining "how to install Elasticsearch", but on understanding from an operations/SRE perspective:

- Why ELK/EFK is needed;
- What scenarios are suitable for ELK, EFK, and Loki respectively;
- The role of Elasticsearch/OpenSearch in a log system;
- The responsibilities of Filebeat, Fluent Bit, Fluentd, and Logstash;
- Where Kubernetes container logs come from;
- How to collect `/var/log/containers` logs;
- How to enrich logs with Namespace, Pod, Container, Node, and other metadata;
- How to plan Index/Data Stream/Index Template;
- How to design fields, Mapping, ECS, and time fields;
- How to query logs through Kibana/OpenSearch Dashboards;
- How to handle index bloat, disk explosion, high cardinality fields, and uncontrolled log volume;
- How to design log alerting;
- How to integrate with Prometheus, Grafana, AlertManager, and Loki for a troubleshooting loop;
- How to implement log retention, permission isolation, security desensitization, and capacity governance in production environments.

This document is suitable for study after completing:

- 11-Prometheus-Architecture and Core Metrics Analysis
- 12-Grafana-Dashboard Construction and Custom Monitoring
- 13-AlertManager-Alerting Strategy and Notification Implementation
- 14-K8S-Monitoring Practice-Node-Pod-Service Metrics Collection and Troubleshooting
- 15-Loki-Log Collection and Query Practice

---

## Tags

#ELK #EFK #Elasticsearch #OpenSearch #Kibana #OpenSearchDashboards #Filebeat #FluentBit #Fluentd #Logstash #Kubernetes #LogCollection #LogSearch #SRE #Observation

---

## Recommended Path

Recommended path:

    06-GPU and AI Infrastructure/05-Observability Foundation/16-ELK-EFK-Log Collection and Retrieval in Practice.md

---

## One, Why Learn ELK / EFK Again

In the previous Loki section, it was explained that Loki is very suitable for Kubernetes operations troubleshooting logs.

However, in production environments, ELK/EFK remains very common.

The reasons are:

- Many enterprises already have Elasticsearch/OpenSearch log platforms;
- Kibana/OpenSearch Dashboards have strong full-text search capabilities;
- Full-text search is often relied upon for audit, security, and business log retrieval;
- Elasticsearch/OpenSearch has mature capabilities for structured field queries and aggregation;
- Many log collection solutions natively support output to Elasticsearch/OpenSearch;
- ELK/EFK architectures are frequently encountered in interviews and production work.

Loki is more like:

    Kubernetes operations log aggregation system
    Friendly for troubleshooting with metric correlation
    Relatively cost-controlled
    Label-indexed primarily

ELK/EFK is more like:

    Full-text search log platform
    Strong capabilities for field search and aggregation analysis
    Suitable for audit, security, and business retrieval
    Higher requirements for resource and index governance

For operations engineers, it's not enough to only know Loki.

They also need to understand:

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
    Alerting and troubleshooting

---

## Two, Basic Concepts of ELK / EFK / ECK / OpenSearch

### 2.1 What is ELK

ELK typically refers to:

    Elasticsearch
    Logstash
    Kibana

Their roles:

    Elasticsearch:
        Log storage, indexing, search, and analysis engine.

    Logstash:
        Log reception, filtering, parsing, transformation, and output pipeline.

    Kibana:
        Querying, visualization, Dashboard, and management interface.

Typical pipeline:

    App Logs
      ↓
    Logstash
      ↓
    Elasticsearch
      ↓
    Kibana

### 2.2 What is EFK

EFK typically refers to:

    Elasticsearch
    Fluentd / Fluent Bit
    Kibana

Some also refer to OpenSearch + Fluent Bit + OpenSearch Dashboards as EFK-like architecture.

Their roles:

    Elasticsearch / OpenSearch:
        Log storage, indexing, search, and analysis.

    Fluentd / Fluent Bit:
        Log collection, filtering, parsing, and forwarding.

    Kibana / OpenSearch Dashboards:
        Log querying and visualization.

Typical pipeline:

    Kubernetes Pod stdout/stderr
      ↓
    /var/log/containers
      ↓
    Fluent Bit DaemonSet
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 2.3 What is ECK

ECK is Elastic Cloud on Kubernetes.

It is based on Kubernetes Operator pattern to manage Elastic Stack components in Kubernetes.

Suitable for:

- Deploying Elasticsearch in Kubernetes;
- Deploying Kibana in Kubernetes;
- Managing certificates;
- Managing cluster topology;
- Managing rolling upgrades;
- Managing storage and node roles.

If deploying Elastic Stack in K8S production environment, it's worth understanding ECK.

However, if an enterprise already has an external Elasticsearch cluster, K8S only needs to deploy log collection agents.

### 2.4 What is OpenSearch

OpenSearch is an open-source search and analysis suite.

Common components:

    OpenSearch:
        Search and analysis engine.

    OpenSearch Dashboards:
        Visualization and query interface.

    Data Prepper:
        Data processing pipeline.

Fluent Bit / Logstash / Beats:
    Common collection or forwarding components.

OpenSearch is used as an alternative to Elasticsearch in many enterprises.

### 2.5 Strategy of This Article

This article does not force-bind to any specific distribution.

Unified from the perspective of log system capabilities:

    Elasticsearch / OpenSearch:
        Log storage and retrieval backend.

    Kibana / OpenSearch Dashboards:
        Query and visualization interface.

    Filebeat / Fluent Bit / Fluentd / Logstash:
        Log collection, parsing, and forwarding components.

---

## ThreeI don't know.Differences Between ELK / EFK and Loki

### 3.1 Different Indexing Methods

Loki:

    Mainly indexes labels.
    Log body is not indexed by default.
    Suitable for first narrowing down the scope via label, then filtering log content.

Elasticsearch / OpenSearch:

    Can build inverted indexes for log fields.
    Supports powerful full-text search and field search.
    Stronger query capabilities, but higher indexing costs.

### 3.2 Different Query Methods

Loki:

    LogQL

Example:

    {namespace="prod", app="api"} |= "ERROR"

Elasticsearch / OpenSearch:

    KQL
    Lucene Query
    Query DSL

Example:

    kubernetes.namespace: "prod" and log.level: "error"

Or Query DSL:

    GET logs-*/_search
    {
      "query": {
        "bool": {
          "must": [
            { "match": { "kubernetes.namespace": "prod" } },
            { "match": { "log.level": "error" } }
          ]
        }
      }
    }

### 3.3 Different Cost Models

Loki:

    Fewer indexes, relatively low storage costs.
    Suitable for Kubernetes operations logs and troubleshooting.

Elasticsearch / OpenSearch:

    More field indexes, stronger query capabilities.
    But higher storage, memory, CPU, and shard management costs.

### 3.4 Different Use Cases

Loki is more suitable for:

- K8S application log aggregation;
- SRE troubleshooting;
- Integration with Prometheus / Grafana;
- Cost-sensitive scenarios;
- Locating logs by label dimensions.

ELK / EFK is more suitable for:

- Full-text log search;
- Audit logs;
- Security logs;
- Business log analysis;
- Multi-field complex queries;
- Log platforms requiring strong search capabilities.

### 3.5 Production Selection Recommendations

Small-scale K8S operation logs:

    Loki is sufficient.

Medium-to-large-scale business log search:

    Elasticsearch / OpenSearch is more suitable.

Audit and security logs:

    Elasticsearch / OpenSearch is more common.

Cost-sensitive but only for troubleshooting:

    Loki is lighter.

Complex queries and field aggregation:

    ELK / EFK is stronger.

---

## FourI don't know.Sources of Kubernetes Container Logs

### 4.1 Standard Log Output

Kubernetes recommends container logs output to:

    stdout
    stderr

The container runtime writes logs to the node's local storage.

Common paths in containerd environment:

    /var/log/containers/
    /var/log/pods/

Example:

    /var/log/containers/nginx-xxx_default_nginx-abc123.log

View:

    ls -l /var/log/containers/
    ls -l /var/log/pods/

### 4.2 Why Use DaemonSet for Log Collection Agents

Log collection agents in Kubernetes typically run as DaemonSets.

Reasons:

    Each node has container log files.
    Each node needs local collection.
    Agents need to mount the host's log directory.
    After Pod migration, agents on new nodes continue collection.

Typical DaemonSets:

    Filebeat DaemonSet
    Fluent Bit DaemonSet
    Fluentd DaemonSet

### 4.3 Basic Log Collection Pipeline

    Pod stdout/stderr
      ↓
    containerd writes to node log files
      ↓
    /var/log/containers/*.log
      ↓
    Filebeat / Fluent Bit / Fluentd
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 4.4 Recommended Application Log Output

Recommended:

    Output application logs to stdout/stderr.
    Use JSON log format.
    Logs include service, level, trace_id, request_id, etc.
    Do not output sensitive information.
    Control DEBUG logs.

Not recommended:

    Only write to internal container files.
    Only write to local disk.
    Print large amounts of unstructured text.
    Print passwords, tokens, Cookies, Authorization headers.
    Print complete large request bodies and response bodies.

---

## FiveI don't know.Comparison of Log Collection Components

### 5.1 Filebeat

Filebeat is a lightweight log collector in the Elastic Beats ecosystem.

Features:

- Lightweight;
- Commonly used for collecting file logs;
- Supports Kubernetes DaemonSet;
- Supports autodiscover;
- Outputs to Elasticsearch, Logstash, Kafka, etc.;
- Friendly integration with Elastic Stack.

Suitable for:

- Elastic Stack environments;
- Collecting container logs;
- Collecting system logs;
- Lighter log forwarding.

### 5.2 Fluent Bit

Fluent Bit is a lightweight log collector and forwarder.

Features:

- Low resource usage;
- Implemented in C;
- Suitable for containers and edge environments;
- Supports multiple inputs and outputs;
- Supports Kubernetes metadata;
- Commonly used for K8S log collection.

Suitable for:

- Kubernetes DaemonSet log collection;
- Large number of nodes;
- Resource-sensitive environment;
- Output to Elasticsearch / OpenSearch / Loki / Kafka.

### 5.3 Fluentd

Fluentd is heavier than Fluent Bit, with a rich plugin ecosystem.

Features:

- Ruby implementation;
- Numerous plugins;
- Strong processing capabilities;
- Higher resource consumption than Fluent Bit;
- Suitable as an aggregation layer.

Common combinations:

    Fluent Bit DaemonSet
      ↓
    Fluentd Aggregator
      ↓
    Elasticsearch / OpenSearch

### 5.4 Logstash

Logstash is the data processing pipeline in the Elastic Stack.

Features:

- Powerful filter capabilities;
- Supports grok, mutate, date, geoip, etc.;
- Numerous plugins;
- High resource consumption;
- Often used as a centralized processing layer.

Common combinations:

    Filebeat
      ↓
    Logstash
      ↓
    Elasticsearch

Suitable for:

- Complex parsing;
- Unified processing of logs from multiple sources;
- Need for rich filters;
- Need to output to multiple backends.

### 5.5 Selection Recommendations

Kubernetes lightweight collection:

    Fluent Bit

Elastic Stack native collection:

    Filebeat

Complex log processing:

    Logstash

Large-scale scenarios:

    Fluent Bit collection + Kafka buffering + Logstash/Fluentd processing + Elasticsearch/OpenSearch

Learning environment:

    Either Filebeat or Fluent Bit is sufficient.

---

## SixI don't know.Typical Architecture One: Filebeat + Elasticsearch + Kibana

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

- Using Elastic Stack;
- Collecting Kubernetes container logs;
- Needing quick access to Kibana;
- No need for complex intermediate processing;
- Logs have relatively standardized formats.

### 6.3 Advantages

- Simple deployment;
- Official Elastic ecosystem;
- Good support for Kubernetes autodiscover;
- Can directly output to Elasticsearch;
- Friendly to Kibana index pattern / data view.

### 6.4 Disadvantages

- Less capability for complex log processing than Logstash;
- Less flexibility when outputting to non-Elastic backends than Fluent Bit;
- Need to monitor Filebeat resources and backpressure in high log volume scenarios.

---

## SevenI don't know.Typical Architecture Two: Fluent Bit + Elasticsearch / OpenSearch + Dashboard

### 7.1 Architecture Diagram

    Kubernetes Pod
      ↓
    /var/log/containers
      ↓
    Fluent Bit DaemonSet
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 7.2 Applicable Scenarios

Suitable for:

- Kubernetes container log collection;
- OpenSearch environment;
- Resource-sensitive scenarios;
- Need for lightweight collection agent;
- Multiple output targets.

### 7.3 Advantages

- Low resource consumption;
- Good performance;
- Flexible configuration;
- Supports Kubernetes metadata;
- Supports multiple outputs;
- Commonly used in container environments.

### 7.4 Disadvantages

- Limited capability for complex parsing;
- More configuration details;
- Some complex processing needs to be handled by downstream or Fluentd / Logstash.

---

## EightI don't know.Typical Architecture Three: Filebeat + Logstash + Elasticsearch + Kibana

### 8.1 Architecture Diagram

    Kubernetes Pod
      ↓
    Filebeat DaemonSet
      ↓
    Logstash
      ↓
    Elasticsearch
      ↓
    Kibana

### 8.2 Applicable Scenarios

Suitable for:

- Multiple log sources;
- Need for complex grok parsing;
- Need for field cleaning;
- Need for multiple outputs;
- Need for unified processing layer;
- Enterprise already has Logstash pipeline.

### 8.3 Advantages

- Strong parsing capabilities;
- Flexible pipeline;
- Easy to perform field conversion;
- Suitable for aggregating logs from multiple sources.

### 8.4 Disadvantages

- High resource consumption for Logstash;
- Adds an extra layer in the pipeline;
- Increases troubleshooting complexity;
- Needs planning for Logstash high availability.

---

## NineI don't know.Typical Architecture Four: Agent + Kafka + Elasticsearch

### 9.1 Architecture Diagram

    Filebeat / Fluent Bit
      ↓
    Kafka
      ↓
    Logstash / Fluentd / Consumer
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

### 9.2 Why Introduce Kafka

Kafka is used for buffering and traffic shaping.

Suitable for:

- Large volume of logs;
- Fluctuating Elasticsearch write pressure;
- Need to prevent log loss when backend is temporarily unavailable;
- Multiple consumers consuming logs;
- Need to decouple the log pipeline.

### 9.3 Advantages

- Strong resistance to sudden traffic spikes;
- Decouples upstream and downstream;
- Can buffer when backend maintenance is ongoing;
- Available for multiple systems to consume.

### 9.4 Disadvantages

- Complex architecture;
- Operational cost for Kafka itself;
- Increased latency;
- More complex troubleshooting in the pipeline.

### 9.5 Production Recommendations

Small-to-medium environments:

    Agent writes directly to Elasticsearch / OpenSearch

Large-scale environments:

    Agent → Kafka → Logstash/Consumer → Elasticsearch/OpenSearch

---

## TenI don't know.Elasticsearch / OpenSearch Basic Concepts

### 10.1 Cluster

A Cluster is a search cluster.

Composed of multiple Nodes.

### 10.2 Node

A Node is an instance in the cluster.

Common roles: /think

master / cluster_manager
data
ingest
coordinating
ml
transform

The master node role in OpenSearch is typically referred to as cluster_manager.

### 10.3 Index

Index is a collection of documents.

Common in log systems:

    logs-prod-2026.04.30
    k8s-app-2026.04.30
    nginx-access-2026.04.30

### 10.4 Document

Document is a record written to the search engine.

A single log entry is typically a document.

Example:

    {
      "@timestamp": "2026-04-30T12:00:00+08:00",
      "message": "database connection failed",
      "log.level": "error",
      "kubernetes.namespace": "prod",
      "kubernetes.pod.name": "api-xxx"
    }

### 10.5 Field

Field is a field in a document.

Examples:

    @timestamp
    message
    log.level
    service.name
    kubernetes.namespace
    kubernetes.pod.name
    host.name

### 10.6 Mapping

Mapping defines field types.

Examples:

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

Poor mapping design can lead to:

- Query failures;
- Aggregation failures;
- Field type conflicts;
- Index write errors;
- Kibana unable to display correctly.

### 10.7 Shard

Shard is a shard of an index.

An index can have multiple primary shards.

Too many shards can lead to:

- Cluster state bloat;
- High memory consumption;
- High query overhead;
- Too many small indices;
- Heavy master pressure.

Too few shards can lead to:

- Single shard too large;
- Insufficient query/write concurrency;
- Slow recovery.

### 10.8 Replica

Replica is a replica shard.

Functions:

- Improve availability;
- Improve query concurrency;
- Still available after node failure.

More replicas:

    Higher availability
    Higher storage cost
    Higher write pressure

### 10.9 Data Stream

Data Stream is used for time-series data, such as logs, metrics, and events.

It can hide underlying backing indices and manage logs through rollover and lifecycle policies.

It's recommended to understand Data Stream first in production log systems.

### 10.10 ILM / ISM

Common in Elastic:

    ILM:Index Lifecycle Management

Common in OpenSearch:

    ISM:Index State Management

They are used to manage index lifecycles.

Example:

    hot
      ↓
    warm
      ↓
    cold
      ↓
    delete

Log systems must configure lifecycle management, otherwise disks will eventually fill up.

---

## ElevenI don't know.Log Field Specification

### 11.1 Why Field Specification Matters

If fields are disorganized, it can lead to:

- Difficult querying in Kibana;
- Inconsistent field names across services;
- Status sometimes being string, sometimes number;
- Time fields not being unified;
- Logs unable to be aggregated;
- Mapping conflicts;
- Dashboards unable to be reused;
- Difficult to write alert rules.

### 11.2 Recommended Fields

It's recommended to use field naming conventions close to ECS.

Common fields:

    @timestamp
    message
    log.level
    service.name
    service.version
    event.dataset
    host.name
    kubernetes.namespace
    kubernetes.pod.name
    kubernetes.container.name
    kubernetes.node.name
    trace.id
    transaction.id
    http.request.method
    url.path
    http.response.status_code
    event.duration
    error.message
    error.stack_trace

### 11.3 Kubernetes Metadata Fields

Recommended to retain:

    kubernetes.namespace
    kubernetes.pod.name
    kubernetes.container.name
    kubernetes.node.name
    kubernetes.labels.app
    kubernetes.labels.version
    kubernetes.pod.uid

Notes:

    pod.uid may have high cardinality.
    Use for aggregation cautiously.
    Generally retain as a field, avoid using as high-frequency aggregation dimension.

### 11.4 Log Level Fields

Recommended to unify:

    log.level

Standardized values:

    debug
    info
    warn
    error
    fatal

Avoid mixing:

    ERROR
    Error
    err
    ERR
    warning
    WARN

They can be uniformly converted to lowercase in the collection pipeline.

### 11.5 Time Fields

Must have:

    @timestamp

Notes:

    @timestamp should represent the time when the log occurred.
    Not the collection time.
    Time zone should be unified.
    Recommend using ISO 8601 format.

---

## TwelveI don't know.Mapping Design

### 12.1 text vs keyword Differences

text:

    For full-text search.
    Will be tokenized.
    Suitable for message, error.stack_trace.

keyword:

    For exact matches, aggregations, and sorting.
    Suitable for namespace, pod, service, level, status_code.

Example:

    message:
        text

log.level:
    keyword

kubernetes.namespace:
    keyword

### 12.2 Common Field Types

    date:
        Time field

    keyword:
        Exact match field

    text:
        Full-text search field

    long / integer:
        Numeric field

    boolean:
        Boolean field

    ip:
        IP Address field

    object:
        Object field

    nested:
        Nested object

### 12.3 Mapping Conflict Example

Service A output:

    "status": 200

Service B output:

    "status": "success"

Writing to the same index may cause field type conflicts.

Solution:

- Standardize field specifications;
- Use numeric for status_code;
- Use string for status_text;
- Separate indexes by log type;
- Use pipeline to normalize fields.

### 12.4 Dynamic Mapping Risks

Elasticsearch / OpenSearch can automatically infer field types.

But relying entirely on dynamic Mapping in production environments carries risks.

Risks:

- Uncontrollable field types;
- Field type determined on first write;
- Subsequent writes with different types fail;
- Field count explosion;
- Mapping explosion.

Recommendations:

    Use Index Template.
    Clearly define Mapping for core fields.
    Limit dynamic fields.
    Standardize business fields.

---

## ThirteenI don't know.Index / Data Stream Naming Convention

### 13.1 Naming Principles

Index names should reflect:

    Log type
    Environment
    Cluster
    Application
    Time

But avoid overBreakdown.

### 13.2 Common Naming

By log type:

    logs-k8s-app-prod
    logs-k8s-system-prod
    logs-nginx-access-prod
    logs-audit-prod

By Data Stream approach:

    logs-kubernetes.application-prod
    logs-kubernetes.system-prod
    logs-nginx.access-prod

### 13.3 Not Recommended Naming

Not recommended to create an index per Pod:

    logs-api-pod-xxx-2026.04.30

Issues:

- Index count explosion;
- Shard count explosion;
- Complex queries;
- Difficult lifecycle management.

### 13.4 Recommended Partitioning

Production recommendations:

    k8s-app logs:
        Partition by environment + cluster + log_type.

    ingress logs:
        Separate partition.

    audit logs:
        Separate partition.

    security logs:
        Separate partition.

    debug logs:
        Separate short-term retention.

---

## FourteenI don't know.Lifecycle Management ILM / ISM

### 14.1 Why Must Lifecycle Management Be Done

Logs are continuously growing data.

Without lifecycle management, you may encounter:

- Disk full;
- Cluster turns red;
- Write failures;
- Slow queries;
- Too many shards;
- Operations forced to manually delete indexes.

### 14.2 Common Lifecycle

Example:

    hot:
        Current writing and high-frequency queries.

    warm:
        Low-frequency queries, reduced resources.

    cold:
        Low-frequency queries, cost priority.

    delete:
        Expired deletion.

### 14.3 Retention Period Recommendations

Development environment:

    3d - 7d

Test environment:

    7d - 15d

Production application logs:

    15d - 30d

Audit logs:

    90d / 180d / 1y, based on compliance requirements

Debug logs:

    1d - 3d

### 14.4 Rollover Strategy

Trigger conditions can be based on:

    Time
    Index size
    Document count

Example design:

    Rollover when a single index reaches 30GB
    Or 1 day rollover

Production recommendations:

    Don't let a single shard too large.
    Don't have too many small indexes.
    Determine after testing log volume.

---

## FifteenI don't know.Filebeat Kubernetes Collection Practice

### 15.1 Filebeat DaemonSet Core Concept

Filebeat runs as a DaemonSet.

Mounts:

    /var/log/containers
    /var/log/pods

After collecting logs, output to:

    Elasticsearch
    Logstash
    Kafka

### 15.2 Filebeat Configuration Example

Example structure:

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

    In production, recommend using data stream or explicit index template.
    If Elasticsearch enables authentication, configure username/password or API key.
    HTTPS environment needs CA certificate configuration.

### 15.3 Filebeat DaemonSet Key Mounts

Example focus:

    volumeMounts:
      - name: varlog
        mountPath: /var/log
        readOnly: true

- name: varlibdockercontainers
        mountPath: /var/lib/docker/containers
        readOnly: true

containerd environment focus:

    /var/log/containers
    /var/log/pods

Docker environment may also require:

    /var/lib/docker/containers

### 15.4 Filebeat Environment Variables

Common:

    NODE_NAME

Example:

    env:
      - name: NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName

Used to supplement log information with node details.

### 15.5 Filebeat Troubleshooting

Check Pod:

    kubectl get pod -n logging -o wide | grep filebeat

Check logs:

    kubectl logs <filebeat-pod> -n logging

Enter Pod:

    kubectl exec -it <filebeat-pod> -n logging -- sh

Check mounts:

    ls -l /var/log/containers/

Common errors:

- Elasticsearch address is unreachable;
- Authentication failure;
- TLS certificate failure;
- Insufficient index permissions;
- Mapping conflict;
- Log path not mounted;
- Kubernetes metadata cannot be supplemented;
- Output endpoint throttling or write rejection.

---

## SixteenI don't know.Fluent Bit Kubernetes Collection Practical Implementation

### 16.1 Fluent Bit Architecture

Fluent Bit configuration typically includes:

    INPUT
    FILTER
    OUTPUT

Process:

    INPUT tail collects files
      ↓
    FILTER kubernetes adds metadata
      ↓
    FILTER parser parses logs
      ↓
    OUTPUT outputs to Elasticsearch / OpenSearch

### 16.2 INPUT Example

Collect container logs:

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

### 16.3 FILTER Example

Add Kubernetes metadata:

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

### 16.4 OUTPUT to Elasticsearch

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch.logging.svc
        Port            9200
        Index           logs-k8s-app
        Logstash_Format On
        Logstash_Prefix k8s-app
        Retry_Limit     False

If authentication is enabled:

    HTTP_User      elastic
    HTTP_Passwd    <password>

If HTTPS:

    tls            On
    tls.verify     On

### 16.5 OUTPUT to OpenSearch

OpenSearch typically also supports Elasticsearch output plugin, but production environments should verify compatibility based on version.

Example:

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            opensearch.logging.svc
        Port            9200
        Index           logs-k8s-app
        Logstash_Format On
        Retry_Limit     False
        HTTP_User       admin
        HTTP_Passwd     <password>
        tls             On
        tls.verify      Off

Production environments should not disable tls.verify long-term.

### 16.6 Fluent Bit Troubleshooting

Check:

    kubectl get pod -n logging -o wide | grep fluent-bit
    kubectl logs <fluent-bit-pod> -n logging

Enter:

    kubectl exec -it <fluent-bit-pod> -n logging -- sh
    ls -l /var/log/containers/

Common errors:

- Tail path is incorrect;
- Parser mismatch;
- Kubernetes metadata retrieval failed;
- Output endpoint rejected;
- Mapping conflict;
- Authentication failed;
- TLS failed;
- Buffer full;
- Log line too long and skipped.

---

## 17. Logstash Pipeline Practical Implementation

### 17.1 Logstash Basic Structure

Logstash pipeline contains:

    input
    filter
    output

Example:

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

    Field renaming, type conversion, case handling.

drop:

    Discards useless logs.

geoip:

    IP geolocation parsing.

### 17.3 grok Example

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

- Complex log formats;
- Need for unified cleaning;
- Multiple outputs required;
- Complex grok needed;
- Centralized field processing required.

Not suitable for:

- Very resource-constrained environments;
- Simple K8s log direct collection;
- Small clusters without complex processing.

---

## 18. Kibana / OpenSearch Dashboards Query Practical Implementation

### 18.1 Creating Data View / Index Pattern

In Kibana:

    Stack Management
      ↓
    Data Views
      ↓
    Create data view

Index pattern example:

    logs-k8s-app-*

Time field:

    @timestamp

In OpenSearch Dashboards, it's similar:

    Stack Management / Dashboards Management
      ↓
    Index Patterns
      ↓
    Create index pattern

### 18.2 Discover Query

In Kibana:

    Discover
      ↓
    Select Data View
      ↓
    Enter query conditions
      ↓
    Set time range

Common queries:

    kubernetes.namespace : "prod"

    kubernetes.pod.name : "api-*"

    log.level : "error"

    message : "connection refused"

### 18.3 KQL Query Examples

Query error logs in prod namespace:

    kubernetes.namespace : "prod" and log.level : "error"

Query a specific Pod:

    kubernetes.namespace : "prod" and kubernetes.pod.name : "api-xxx"

Query containing timeout:

    message : "timeout"

Query 5xx errors:

    http.response.status_code >= 500

Query a specific service:

    service.name : "order-api"

### 18.4 Lucene Query Examples

    kubernetes.namespace:prod AND log.level:error

    message:"connection refused"

    http.response.status_code:[500 TO 599]

### 18.5 Time Range

Log queries must first confirm the time range.

Common options:

    Last 15 minutes
    Last 1 hour
    Last 6 hours
    Last 24 hours

Avoid querying the full 30-day log volume at first.

---

## 19. Query DSL Basics

### 19.1 match Query

    GET logs-k8s-app-*/_search
    {
      "query": {
        "match": {
          "message": "connection refused"
        }
      }
    }

### 19.2 term Query

Used for keyword exact matching:

    GET logs-k8s-app-*/_search
    {
      "query": {
        "term": {
          "log.level": "error"
        }
      }
    }

### 19.3 bool Query /think

## Twenty, Log Alert Design

### 20.1 What Log Alerts Are Suitable For

Suitable:

- Error log count abnormally increases;
- Critical errors appear;
- Database connection fails;
- Too many authentication failures;
- Business critical error codes;
- CUDA OOM;
- Java Exception;
- Python Traceback;
- panic;
- Too many Nginx 5xx errors.

### 20.2 What Log Alerts Are Not Suitable For

Not suitable:

- Single ordinary ERROR directly critical;
- Fuzzy keyword global matching;
- One-size-fits-all for all namespaces;
- Full alerts for debug/warn;
- Alerts without owner.

### 20.3 Kibana Alerting Approach

Common rules:

    Query conditions:
        kubernetes.namespace: prod AND log.level: error

    Time window:
        Last 5 minutes

    Threshold:
        count > 20

    Actions:
        Webhook / Email / Slack / Other notifications

### 20.4 OpenSearch Alerting Approach

OpenSearch Dashboards can create Monitors via the Alerting plugin.

Common:

    Scheduled query
    Query log indices
    Judge count or aggregated value
    Trigger alert
    Send notifications

### 20.5 Production Alerting Recommendations

Log alerts should include:

    alertname
    severity
    cluster
    namespace
    service
    pod
    keyword
    count
    time window
    dashboard url
    runbook url

Do not only send:

    error logs too many

---

## Twenty-one, Log and Prometheus Joint Troubleshooting

### 21.1 Metrics Discovering Issues

Prometheus alert:

    ServiceHigh5xxErrorRate

Explanation:

    Service 5xx error rate increases.

### 21.2 Log Locating Causes

Kibana query:

    service.name : "order-api" and http.response.status_code >= 500

Or:

    kubernetes.namespace : "prod" and service.name : "order-api" and log.level : "error"

Find:

- timeout;
- connection refused;
- database error;
- redis error;
- permission denied;
- upstream unavailable.

### 21.3 Kubernetes Event Confirmation

    kubectl describe pod <pod-name> -n <namespace>
    kubectl get events -n <namespace> --sort-by=.lastTimestamp

### 21.4 Full Path

    Prometheus alert discovers anomalies
      ↓
    Grafana view trends
      ↓
    Kibana query error logs
      ↓
    kubectl describe view events
      ↓
    kubectl logs --previous supplement confirmation
      ↓
    Fix configuration / rollback / scale up / recover dependencies
      ↓
    Observe metrics and logs recovery
      ↓
    Post-mortem analysis

---

## Twenty-two, GPU/AI Log Search Scenarios

### 22.1 Common GPU/AI Log Keywords

Query:

    CUDA out of memory
    RuntimeError
    Traceback
    NCCL
    model load failed
    checkpoint
    no space left on device
    connection refused
    timeout
    failed to allocate memory
    CUBLAS
    CUDNN

### 22.2 Kibana Query CUDA OOM

KQL:

    kubernetes.namespace : "ai-prod" and message : "CUDA out of memory"

### 22.3 Query PyTorch Exceptions

    kubernetes.namespace : "ai-prod" and message : "RuntimeError"

### 22.4 Query Model Load Failures

kubernetes.namespace : "ai-prod" and message : "model load failed"

### 22.5 GPU Metric Correlation

Prometheus Query:

    DCGM_FI_DEV_FB_USED
    DCGM_FI_DEV_GPU_UTIL
    DCGM_FI_DEV_GPU_TEMP
    DCGM_FI_DEV_XID_ERRORS

Node Troubleshooting:

    nvidia-smi
    dmesg | grep -i xid
    journalctl -k | grep -i nvidia

Pod Troubleshooting:

    kubectl describe pod <pod-name> -n ai-prod
    kubectl logs <pod-name> -n ai-prod --previous

### 22.6 GPU Troubleshooting Judgment

If logs show CUDA OOM:

    Prioritize checking memory.
    Check batch size.
    Check concurrency.
    Check model size.
    Check worker count.
    Check for multiple model re-loading.

If logs show NCCL:

    Check multi-GPU communication.
    Check network.
    Check RDMA.
    Check GPU topology.
    Check distributed training configuration.

If logs show model load failed:

    Check model file path.
    Check object storage permissions.
    Check PVC mounting.
    Check file integrity.
    Check image and dependencies.

---

## Twenty-Three, Log Security and Desensitization

### 23.1 Information Not to Output

Prohibited outputs:

    Password
    token
    access key
    secret key
    private key
    Authorization header
    Cookie
    ID card
    Bank card
    Phone number in plain text
    Email in plain text
    User privacy data
    Database connection string password

### 23.2 Desensitization Locations

Priority:

    Application side does not output sensitive information
      ↓
    SDK / logging library filtering
      ↓
    Agent desensitization
      ↓
    Logstash / Fluentd pipeline desensitization
      ↓
    Index permission control

Do not rely on log platform to delete sensitive information after the fact.

### 23.3 Desensitization Examples

Phone number:

    13812345678
      ↓
    138****5678

token:

    Bearer abcdefghijklmn
      ↓
    Bearer ***

### 23.4 Permission Isolation

Production log platform must consider:

- Who can view production logs;
- Who can view sensitive Namespace;
- Who can export logs;
- Who can delete indexes;
- Who can modify index templates;
- Who can configure alerts;
- Who can view audit logs.

Log system permissions cannot be overly open.

---

## Twenty-Four, Log Volume Governance

### 24.1 Reasons for Log Volume Out of Control

Common causes:

- DEBUG is enabled;
- Each request prints full request body;
- Each response prints full response body;
- Loop error flooding;
- Too much health check logs;
- Access log too large;
- Exception stack repeated printing;
- Sidecar logs too much;
- Collected useless system logs;
- Multi-line logs not merged.

### 24.2 Governance Methods

Recommendations:

    1. Production default INFO
    2. DEBUG only enabled for short-term troubleshooting
    3. Filter health checks
    4. Limit large field output
    5. Sample or tiered retention for access log
    6. Limit flood of error logs
    7. dev/test set shorter retention period
    8. Statistics Top log sources
    9. Clean logs without owner
    10. Set write speed limit for log platform

### 24.3 Top Log Source Analysis

Can be analyzed by:

- namespace;
- service;
- pod;
- container;
- node;
- log.level;
- index;

Statistics log volume.

Goals:

    Find the most resource-consuming log source.
    Determine if it's reasonable.
    Optimize output and retention strategy.

---

## Twenty-Five, Elasticsearch / OpenSearch Capacity Planning

### 25.1 Factors Affecting Capacity

Main factors:

- Daily raw log volume;
- Replica count;
- Index compression rate;
- Retention days;
- Field count;
- Mapping;
- Shard count;
- Query concurrency;
- Aggregation query complexity;
- hot/warm/cold architecture;
- Whether to save source;
- Whether to enable full-text indexing.

### 25.2 Rough Estimation

Assumption:

    Daily raw logs: 100GB
    Replica count: 1
    Retention: 30 days
    Index expansion factor: 1.2

Rough storage:

    100GB * 30 * 2 * 1.2 = 7200GB

Which is approximately:

    7.2TB

Actual considerations also include:

- Disk watermark;
- Segment merging;
- OS reserved space;
- Snapshots;
- Temporary space;
- Query overhead.

### 25.3 Disk Watermark

Elasticsearch / OpenSearch typically has disk watermark protection.

When disk usage is too high, may appear:

- Shards no longer allocated;
- Index becomes read-only;
- Write failure;
- Cluster yellow/red.

Therefore, do not wait until disk approaches 100% to handle.

### 25.4 Production Recommendations

Recommendations:

    hot nodes use SSD.
    Log indexes use lifecycle policies.
    Control shard count.
    Control retention period.
    Control replica count.
    Regularly do snapshots.
    Monitor disk watermark.
    Monitor JVM heap.
    Monitor search/write latency.
    Monitor rejected requests.
    Monitor cluster health.

---

## Twenty-Six, Elasticsearch / OpenSearch Operation Metrics

### 26.1 Cluster Health

API:

    GET _cluster/health

Focus on:

    status
    number_of_nodes
    active_primary_shards
    active_shards
    relocating_shards
    initializing_shards
    unassigned_shards

Status:

    green:
        All primary and replica shards are normal.

    yellow:
        Primary shards are normal, but replica shards are incomplete.

    red:
        At least one primary shard is unavailable.

### 26.2 Node Status

API:

    GET _cat/nodes?v

Focus on: /think

heap.percent
ram.percent
cpu
load
node.role
master
name

### 26.3 Index Status

API:

    GET _cat/indices?v

Focus:

    health
    status
    index
    pri
    rep
    docs.count
    store.size
    pri.store.size

### 26.4 Shard Status

API:

    GET _cat/shards?v

Focus:

    index
    shard
    prirep
    state
    docs
    store
    node

### 26.5 Thread Pool Rejections

API:

    GET _cat/thread_pool?v

Focus:

    rejected

Increased rejected writes or queries indicate excessive pressure.

### 26.6 JVM Heap

Focus:

    JVM heap usage rate
    GC count
    GC time
    Old GC
    Circuit Breaker

Long-term high JVM heap usage can cause:

- Slow queries;
- Frequent GC;
- Node instability;
- OOM risk.

---

## 27. Common Troubleshooting Issues

### 27.1 Kibana Cannot Retrieve Logs

Troubleshooting order:

    1. Is the time range correct?
    2. Is the Data View / Index Pattern correct?
    3. Does the index exist?
    4. Is Filebeat / Fluent Bit functioning normally?
    5. Is Elasticsearch / OpenSearch receiving data?
    6. Is the query condition too strict?
    7. Is @timestamp correct?
    8. Is the timezone correct?

Commands:

    GET _cat/indices?v

    GET logs-k8s-app-*/_search
    {
      "size": 1,
      "sort": [
        { "@timestamp": "desc" }
      ]
    }

### 27.2 Agent is Normal but No Data is Written

Troubleshoot:

    kubectl logs <agent-pod> -n logging

Look for:

    connection refused
    authentication failed
    forbidden
    mapper_parsing_exception
    index_not_found_exception
    circuit_breaking_exception
    rejected execution

Common causes:

- Incorrect Elasticsearch address;
- Authentication failure;
- TLS failure;
- Insufficient index permissions;
- Mapping conflict;
- Disk watermark causing index read-only;
- Output endpoint rejecting writes.

### 27.3 Mapping Conflict

Error example:

    mapper_parsing_exception

Cause:

    The same field is written with different types.

Example:

    status: 200
    status: "success"

Resolution:

- Fix the application field;
- Convert fields using Logstash/Fluent Bit;
- Create a new index template;
- Switch to the new index;
- Clean up erroneous data.

### 27.4 Cluster Yellow

Cause:

- Replica allocation failure;
- Insufficient node count;
- Shard allocation rules limitation;
- High disk watermark;
- Node offline.

Troubleshoot:

    GET _cluster/health
    GET _cat/shards?v
    GET _cluster/allocation/explain

### 27.5 Cluster Red

Cause:

- Primary shard unavailable;
- Data node failure;
- Disk damage;
- Index corruption;
- Node loss;
- Shard recovery failure.

Resolution:

    1. Do not blindly delete data
    2. Check cluster health
    3. Check unassigned shards
    4. Check allocation explain
    5. Check node logs
    6. Recover failed nodes
    7. Restore from snapshot if necessary

### 27.6 Disk Full

Symptoms:

- Write failure;
- Index read-only;
- Cluster yellow/red;
- Agent reports write errors.

Troubleshoot:

    GET _cat/allocation?v
    GET _cat/indices?v
    df -h

Resolution:

- Expand disk space;
- Delete expired indices;
- Adjust ILM/ISM;
- Reduce retention period;
- Clean debug logs;
- Temporarily remove read-only status only after freeing space.

Common read-only status removal:

    PUT */_settings
    {
      "index.blocks.read_only_allow_delete": null
    }

Note:

    Remove read-only status only after resolving disk space.
    Otherwise, it will trigger again.

### 27.7 Slow Queries

Cause:

- Large time range;
- Too many wildcard indices;
- Too many shards;
- Unoptimized fields;
- Wildcard prefix queries;
- High cardinality field aggregations;
- Insufficient node resources;
- High JVM heap;
- Slow disk IO.

Optimization:

- Narrow time range;
- Use more precise index pattern;
- Optimize shards;
- Increase hot node resources;
- Use keyword fields for aggregations;
- Avoid large wildcard ranges;
- Create reasonable index templates.

---

## 28. Kubernetes Deployment Considerations

### 28.1 Elasticsearch / OpenSearch Deployment in Kubernetes

It can be deployed inside Kubernetes but with caution.

Considerations:

- StatefulSet;
- PVC;
- StorageClass;
- Node affinity;
- Anti-affinity;
- Resource requests/limits;
- JVM heap;
- PodDisruptionBudget;
- Rolling upgrades;
- Data backup;
- Cluster recovery;
- Disk performance;
- Cross-node data migration.

Development environments can be deployed inside Kubernetes.

Production environments with high log volume should evaluate:

    Dedicated VM / physical cluster
    Hosted Elasticsearch / OpenSearch
    Dedicated node pool inside Kubernetes

### 28.2 Log Collection Agent Deployment in Kubernetes

Filebeat / Fluent Bit are suitable for DaemonSet deployment in Kubernetes.

Must configure: /think

- ServiceAccount  
- ClusterRole  
- ClusterRoleBinding  
- HostPath  
- tolerations  
- nodeSelector  
- resources  
- priorityClass  
- Output Authentication  
- TLS Certificate  
- buffer  

### 28.3 Resource Limits  

The Agent for data collection also requires resource limits.  

Example:  

    resources:  
      requests:  
        cpu: 100m  
        memory: 128Mi  
      limits:  
        cpu: 500m  
        memory: 512Mi  

If the log volume is large, the limits should be appropriately increased.  

### 28.4 Node Pressure  

Log collection consumes:  

- CPU  
- Memory  
- Disk IO  
- Network Bandwidth  

When the log volume is too large, the Agent may affect node business operations.  

Monitoring the Agent itself is mandatory.  

---  

## Twenty-Nine, Production Security Recommendations  

### 29.1 Access Security  

Kibana / OpenSearch Dashboards should not be exposed to the public internet.  

Recommendations:  

- HTTPS  
- Unified Authentication  
- RBAC  
- IP Whitelist  
- VPN  
- Bastion Host  
- Audit Login  
- Read-Only Permissions for Regular Users  
- Minimize Management Permissions  

### 29.2 Data Permissions  

Different teams should only see their own logs.  

Common isolation methods:  

- Isolation by index  
- Isolation by namespace  
- Isolation by tenant  
- Isolation by role mapping  
- Isolation by dashboard permissions  

### 29.3 Sensitive Log Governance  

Must:  

- Prohibit output of passwords  
- Prohibit output of tokens  
- Prohibit output of private keys  
- Prohibit output of Authorization  
- Desensitize phone numbers, ID cards, and emails  
- Control log export permissions  
- Regularly scan for sensitive fields  

### 29.4 Audit  

The log platform itself should have audit capabilities:  

- Who logged in  
- Who queried sensitive logs  
- Who exported data  
- Who deleted indices  
- Who modified permissions  
- Who modified lifecycle policies  

---  

## Thirty, Dashboard Design  

### 30.1 Log Overview Dashboard  

Recommended panels:  

    Daily Log Ingestion Volume  
    Log Ingestion Rate Per Minute  
    ERROR Log Trend  
    WARN Log Trend  
    Top Namespace Log Volume  
    Top Service Log Volume  
    Top Pod Log Volume  
    Index Size Trend  
    Cluster Health Status  
    Disk Usage Rate  

### 30.2 Application Log Dashboard  

Recommended variables:  

    environment  
    cluster  
    namespace  
    service  
    pod  
    level  

Panels:  

    ERROR Trend  
    WARN Trend  
    5xx Logs  
    Timeout Logs  
    Recent Error Logs  
    Top Error Services  
    Top Error Pods  
    Trace ID Query Entry  

### 30.3 Elasticsearch / OpenSearch Cluster Dashboard  

Panels:  

    Cluster Health  
    Node Count  
    JVM Heap  
    CPU  
    Disk Usage  
    Index Count  
    Shard Count  
    Search Latency  
    Indexing Rate  
    Rejected Requests  
    GC Time  
    Query Latency  
    Write Failures  

### 30.4 GPU / AI Log Dashboard  

Panels:  

    CUDA OOM Log Trend  
    NCCL Error Logs  
    RuntimeError  
    Traceback  
    Model Load Failure  
    Inference Error  
    Training Task Failure  
    Linked GPU Metrics  

---  

## Thirty-One, Log System Alert Baseline  

### 31.1 Log Platform Self-Alerts  

Must alert:  

    Elasticsearch / OpenSearch cluster red  
    cluster yellow lasting too long  
    data node down  
    JVM heap high  
    disk water level high  
    write rejected growth  
    query rejected growth  
    Filebeat / Fluent BitMass Send Failed  
    Logstash pipeline blocked  
    index read-only  
    snapshot failure  
    ILM / ISM execution failure  

### 31.2 Application Log Alerts  

Recommended alerts:  

    Sudden increase in production ERROR logs  
    Sudden increase in service 5xx logs  
    panic / fatal  
    Sudden increase in Java Exceptions  
    Sudden increase in Python Tracebacks  
    Database connection failure  
    Redis connection failure  
    CUDA OOM  
    Sudden increase in authentication failures  

### 31.3 Alert Classification  

Critical:  

    Log Platform Unwritable  
    Cluster red  
    Sudden increase in production service error logs affecting business  
    Disk about to be full  
    Large amount of log loss  

Warning:  

    Cluster yellow  
    JVM heap high  
    Slow queries  
    Agent send failure  
    ERROR growth for a specific service  

Info:  

    High log volume for a specific namespace  
    Too many DEBUG logs  
    About to reach retention period  
    Top log source governance reminder  

---  

## Thirty-Two, Complete Troubleshooting Case: Pod is Normal but Business 500 Increased  

### 32.1 Phenomenon  

Prometheus alert:  

    ServiceHigh5xxErrorRate  

Kubernetes view:  

    Pod Running  
    Service Normal  
    Endpoints Normal  

### 32.2 Kibana Query  

KQL:  

    kubernetes.namespace : "prod"  
    and service.name : "order-api"  
    and http.response.status_code >= 500  

Check error logs.  

### 32.3 Further Filtering  

Query database anomalies:  

    kubernetes.namespace : "prod"  
    and service.name : "order-api"  
    and message : "database"  

Query timeout: /think

kubernetes.namespace : "prod"
and service.name : "order-api"
and message : "timeout"

Query the latest released version:

    kubernetes.namespace : "prod"
    and service.name : "order-api"
    and service.version : "v1.2.3"

### 32.4 kubectl Verification

    kubectl describe pod <pod-name> -n prod
    kubectl logs <pod-name> -n prod --previous
    kubectl get events -n prod --sort-by=.lastTimestamp

### 32.5 Possible Root Causes

- New version bug;
- Database connection pool full;
- Redis timeout;
- Downstream service unavailable;
- Configuration change error;
- Secret update error;
- NetworkPolicy blocking;
- DNS anomaly.

### 32.6 Handling

- Rollback version;
- Fix configuration;
- Restore downstream;
- Expand connection pool;
- Fix network policy;
- Observe 5xx and log recovery;
- Review and supplement alerts.

---

## Thirty-ThreeI don't know.Complete Troubleshooting Case: Log Platform Write Failure

### 33.1 Phenomenon

Filebeat / Fluent Bit logs show:

    429 Too Many Requests
    403 Forbidden
    400 mapper_parsing_exception
    connection refused
    no living connections

### 33.2 Troubleshoot the Collector

    kubectl logs <agent-pod> -n logging
    kubectl describe pod <agent-pod> -n logging

Confirm:

- Whether all nodes are abnormal;
- Whether single node is abnormal;
- Whether output endpoint is unreachable;
- Whether authentication failed;
- Whether mapping conflict;
- Whether throttling.

### 33.3 Troubleshoot Elasticsearch / OpenSearch

    GET _cluster/health
    GET _cat/nodes?v
    GET _cat/indices?v
    GET _cat/thread_pool?v
    GET _cat/allocation?v

### 33.4 Judgment

429:

    Write pressure is high or throttling.

403:

    Insufficient permissions.

400 mapping:

    Field type conflict.

connection refused:

    Service is unreachable.

no living connections:

    All output endpoints are unavailable or network anomaly.

### 33.5 Handling

- Expand data nodes;
- Fix permissions;
- Fix index template;
- Create new index;
- Fix mapping conflict;
- Reduce log volume;
- Temporarily throttle;
- Supplement monitoring after recovery.

---

## Thirty-FourI don't know.Operation Command Summary

### 34.1 Kubernetes Side

View collection components:

    kubectl get pods -n logging -o wide
    kubectl get ds -n logging
    kubectl get svc -n logging

View logs:

    kubectl logs <filebeat-pod> -n logging
    kubectl logs <fluent-bit-pod> -n logging
    kubectl logs <logstash-pod> -n logging

View node log paths:

    ls -l /var/log/containers/
    ls -l /var/log/pods/

View business logs:

    kubectl logs <pod-name> -n <namespace>
    kubectl logs <pod-name> -n <namespace> --previous

### 34.2 Elasticsearch / OpenSearch API

Cluster health:

    GET _cluster/health

Nodes:

    GET _cat/nodes?v

Indices:

    GET _cat/indices?v

Shards:

    GET _cat/shards?v

Allocation explanation:

    GET _cluster/allocation/explain

View index template:

    GET _index_template

View index settings:

    GET logs-k8s-app-*/_settings

View Mapping:

    GET logs-k8s-app-*/_mapping

Search:

    GET logs-k8s-app-*/_search
    {
      "query": {
        "match_all": {}
      },
      "size": 10
    }

### 34.3 Linux Node Side

View disk:

    df -h
    df -i

View directories:

    du -xh /var/log | sort -h | tail
    du -xh /var/lib/containerd | sort -h | tail

View network:

    ss -lntp
    curl http://<elasticsearch>:9200/_cluster/health

---

## Thirty-FiveI don't know.Production Implementation Recommendations

### 35.1 Recommended Architecture for Small to Medium Scale

    Fluent Bit DaemonSet
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

Suitable for:

- Not too large log volume;
- No need for complex intermediate processing;
- Want simple architecture.

### 35.2 Recommended Architecture for Large Scale

    Fluent Bit / Filebeat
      ↓
    Kafka
      ↓
    Logstash / Fluentd / Consumer
      ↓
    Elasticsearch / OpenSearch
      ↓
    Kibana / OpenSearch Dashboards

Suitable for:

- Large log volume;
- Need buffering;
- Need multiple consumers;
- Need complex parsing;
- Backend needs maintenance window.

### 35.3 Coexistence Suggestions with Loki

You can divide responsibilities like this:

    Loki:
        K8S operation and troubleshooting logs, Grafana integration, short and medium-term logs.

Elasticsearch / OpenSearch:
    Business search, audit, security, complex field search, long-term analysis.

### 35.4 Production Must Have

    [ ] Lifecycle management
    [ ] Index template
    [ ] Field specification
    [ ] Permission isolation
    [ ] Sensitive information governance
    [ ] Snapshot backup
    [ ] Cluster health monitoring
    [ ] Disk watermark alert
    [ ] Agent send failure alert
    [ ] Log volume Top statistics
    [ ] Runbook
    [ ] Capacity planning

---

## Thirty-sixI don't know.Acceptance Checklist

### 36.1 Collection Pipeline

    [ ] Filebeat / Fluent Bit DaemonSet Running
    [ ] Agent covers all nodes
    [ ] /var/log/containers mounted
    [ ] Kubernetes metadata added
    [ ] Elasticsearch / OpenSearch writable
    [ ] Kibana / Dashboard queryable
    [ ] @timestamp correct
    [ ] namespace / pod / container fields exist

### 36.2 Query Capabilities

    [ ] Queryable by namespace
    [ ] Queryable by pod
    [ ] Queryable by service
    [ ] Queryable by log.level
    [ ] Queryable by status_code
    [ ] Queryable by message keyword
    [ ] Queryable by trace.id
    [ ] Queryable recent error logs
    [ ] Queryable historical logs

### 36.3 Index Governance

    [ ] Index Template configured
    [ ] Mapping core fields correct
    [ ] ILM / ISM configured
    [ ] Rollover configured
    [ ] Retention period configured
    [ ] Shard count reasonable
    [ ] Replica count reasonable
    [ ] Disk watermark alert configured

### 36.4 Security Governance

    [ ] Kibana / Dashboard uses HTTPS
    [ ] Authentication enabled
    [ ] Minimal permissions
    [ ] Sensitive fields desensitized
    [ ] token / password not written to logs
    [ ] Log export permissions controlled
    [ ] Index deletion permissions controlled
    [ ] Audit logs queryable

### 36.5 Operations Governance

    [ ] Agent resource limits configured
    [ ] Agent send failure alerts
    [ ] Log platform cluster health alerts
    [ ] Log volume Top statistics
    [ ] Debug logs governance
    [ ] Runbook written
    [ ] Dashboard established
    [ ] Alerts integrated with notification channels

---

## Thirty-sevenI don't know.Common Misconceptions

### 37.1 Misconception One: ELK installed equals log platform completion

Error.

Log platform also needs:

- Field specification;
- Index template;
- Lifecycle;
- Permissions;
- Desensitization;
- Alerts;
- Capacity planning;
- Runbook.

### 37.2 Misconception Two: All logs should be permanently retained

Error.

Log retention should be tiered:

- debug short-term;
- application troubleshooting medium-term;
- audit long-term;
- security compliance-based.

### 37.3 Misconception Three: One index per service is best

Not necessarily.

Too many services can cause index and shard explosion.

Indexes should be designed based on log volume, query requirements, and permission models.

### 37.4 Misconception Four: More fields are better

Error.

Too many fields cause mapping inflation and storage cost increase.

Core field specification is sufficient.

### 37.5 Misconception Five: Elasticsearch disk full just delete a few indexes

Incomplete.

Also need:

- Check ILM;
- Check log source;
- Check shard;
- Check replica;
- Check disk watermark;
- Check index read-only status;
- Supplement alerts and capacity planning.

### 37.6 Misconception Six: Log system can replace metrics monitoring

Error.

Log system is suitable for root cause analysis.

Metrics system is suitable for trends, alerts, and capacity analysis.

Production troubleshooting requires both.

---

## Thirty-eightI don't know.Summary

ELK / EFK is a critical log search and analysis system in production environments.

Its core pipeline is:

    Application stdout/stderr
      ↓
    Container runtime writes node logs
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
    Query / Dashboard / Alert / Audit

Compared to Loki:

    Loki is lighter, better suited for K8S operations troubleshooting and Grafana integration.

    ELK / EFK has stronger query capabilities, better suited for full-text search, field analysis, audit, and business log search.

When implementing in production, the focus is not on "logs can be written", but:

    Field standardization
    Mapping stability
    Reasonable index planning
    Controllable lifecycle
    Stable query performance
    Clear permission boundaries
    No sensitive information leakage
    Log volume governance
    Alerts with closed-loop
    Predictable capacity

Recommended troubleshooting path:

    Prometheus metrics discover anomalies
      ↓
    Grafana view trends
      ↓
    Kibana / OpenSearch Dashboards query logs
      ↓
    kubectl describe view events
      ↓
    kubectl logs --previous supplement confirmation
      ↓
    Elasticsearch / OpenSearch API view platform health
      ↓
    Fix the fault
      ↓
    Review and optimize logs and alerts

For operations/SRE, mastering ELK / EFK is not about knowing how to use the Kibana search bar, but understanding the complete production pipeline from log generation, collection, parsing, indexing, querying, alerts, storage governance to security compliance.

---

## Reference Documents

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes System Logs:
  https://kubernetes.io/docs/concepts/cluster-administration/system-logs/

- Elasticsearch Reference:
  https://www.elastic.co/docs/reference/elasticsearch

- Elastic Stack Features:
  https://www.elastic.co/elastic-stack/features

- Run Filebeat on Kubernetes:
  https://www.elastic.co/docs/reference/beats/filebeat/running-on-kubernetes

- Filebeat Autodiscover:
  https://www.elastic.co/docs/reference/beats/filebeat/configuration-autodiscover

- Filebeat Hints Based Autodiscover:
  https://www.elastic.co/docs/reference/beats/filebeat/configuration-autodiscover-hints

- Elastic Common Schema:
  https://www.elastic.co/docs/reference/ecs

- OpenSearch Documentation:
  https://docs.opensearch.org/latest/

- OpenSearch Introduction:
  https://docs.opensearch.org/latest/getting-started/intro/

- Fluent Bit Documentation:
  https://docs.fluentbit.io/

- Fluentd Documentation:
  https://docs.fluentd.org/

- Logstash Documentation:
  https://www.elastic.co/docs/reference/logstash

- Kibana Documentation:
  https://www.elastic.co/docs/explore-analyze

- OpenSearch Dashboards:
  https://docs.opensearch.org/latest/dashboards/