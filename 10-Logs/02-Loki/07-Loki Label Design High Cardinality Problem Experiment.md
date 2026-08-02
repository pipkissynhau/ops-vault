# 07-Loki Label Design and High Cardinality Problem Experiment

## Document Notes

This is the seventh article in the Loki specialized learning series, used to systematically learn Loki's Label design principles, Stream concept, high cardinality issues, and how to observe the impact of poor label design on Loki's writing, querying, and storage through experiments.

Previously completed:

    01-Loki Basic Understanding and Experiment Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experiment Selection
    04-Loki Single-node Helm Deployment Practical
    05-Loki Object Storage Access MinIO Practical
    06-Grafana-Alloy Collection K8S-Pod Logs Practical

The 06th article has already used Grafana Alloy to collect Kubernetes Pod logs, and can query in Loki:

    {namespace="app-demo"}
    {namespace="app-demo", pod=~"nginx-demo-.*"}
    {namespace="app-demo", container="nginx"}

This article begins to enter one of Loki's core capabilities:

    Label Design

Loki's query efficiency, writing performance, storage cost, and alert maintainability largely depend on whether the label design is reasonable.

This article focuses on answering the following questions:

- What is a Loki Label?
- What is a Loki Stream?
- What is the difference between Label and log body?
- What is Cardinality?
- Why does high cardinality undermine Loki?
- Which fields are suitable as Labels?
- Which fields are not suitable as Labels?
- Are namespace / pod / container / node / app / team suitable as Labels?
- Why are request_id / trace_id / user_id / order_id unsuitable as Labels?
- How to view current Labels in Loki?
- How to view the number of values for a specific Label?
- How to observe the problems caused by high cardinality fields through experiments?
- How to move high cardinality fields to log body or structured fields instead of Labels?
- How to establish Loki Label specifications in production environments.

---

## Labels

#Loki #LogQL #Label #Stream #Cardinality #HighBaseFigure #GrafanaAlloy #Kubernetes #PodLog #SRE #LogGovernance #Observation

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/07-Loki Label Design and High Cardinality Problem Experiment.md

---

## One, Experiment Objectives

After completing this article, you should be able to:

    1. Understand the role of Loki Labels.
    2. Understand the meaning of Loki Streams.
    3. Understand what Cardinality is.
    4. Understand why high cardinality affects Loki.
    5. Know which fields are suitable as Labels.
    6. Know which fields are not suitable as Labels.
    7. Be able to view current Labels through Loki API.
    8. Be able to query logs under different Label combinations through LogQL.
    9. Be able to simulate the risks of using request_id / user_id as Labels through experiments.
    10. Be able to adjust Alloy relabel configuration to control Labels written to Loki.
    11. Be able to design a recommended Label set for Kubernetes Pod logs.
    12. Be able to establish Loki Label specifications for production environments.

---

## Two, Experiment Environment

### 2.1 Kubernetes Cluster

Experiment nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespaces:

    logging
    app-demo
    monitoring
    minio

Deployed components:

    Loki
    Loki Gateway
    Grafana Alloy
    MinIO
    nginx-demo test application

### 2.2 Prerequisites

Need to have completed:

    [ ] Loki /ready returns ready
    [ ] Alloy DaemonSet Running
    [ ] Alloy can collect app-demo logs
    [ ] Loki can query app-demo logs
    [ ] app-demo contains nginx-demo test application

Verify Loki:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

Verify logs:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

---

## Three, What is a Loki Label

### 3.1 Basic Concept of Labels

A Label is a key-value pair used to identify log streams in Loki.

Example:

    {namespace="app-demo", pod="nginx-demo-6f7c9b5c8d-abcde", container="nginx"}

Where:

    namespace="app-demo"
    pod="nginx-demo-6f7c9b5c8d-abcde"
    container="nginx"

Are all Labels.

In Loki queries, you must first use a Label Selector to narrow down the log scope.

For example:

    {namespace="app-demo"}

    {namespace="app-demo", container="nginx"}

    {namespace="app-demo", pod=~"nginx-demo-.*"}

### 3.2 Purpose of Labels

Main purposes of Labels:

    1. Narrow down query scope.
    2. Help Loki find the corresponding log stream.
    3. Support Dashboard variable filtering.
    4. Support alert grouping.
    5. Support team and environment isolation.
    6. Support querying by namespace / app / pod / container.
    7. Support log and metric correlation.

For example, common queries during production troubleshooting:

    {cluster="prod-k8s", namespace="order", app="order-api"} |= "ERROR"

Meaning: /think

First narrow down to the prod-k8s cluster's order namespace order-api application.
Then search for ERROR in the log body.

### 3.3 Label is not the log body

Need to distinguish between:

    Label:
        Used for indexing, filtering, and locating log streams.

    Log body:
        The actual log content output by the application.

Example log:

    {"level":"error","trace_id":"abc123","msg":"database connection failed","duration_ms":1200}

Reasonable design:

    Label:
        namespace="app-demo"
        app="api"
        container="api"
        environment="lab"

    Log body:
        level="error"
        trace_id="abc123"
        msg="database connection failed"
        duration_ms=1200

Unreasonable design:

    Label:
        trace_id="abc123"
        msg="database connection failed"
        duration_ms="1200"

Reason:

    trace_id may vary for each request.
    msg changes frequently.
    duration_ms may also change frequently.
    These fields will cause high cardinality issues.

---

## FourI don't know.What is a Loki Stream

### 4.1 Definition of a Stream

In Loki, logs with exactly the same Label set belong to the same Stream.

Example:

    {namespace="app-demo", pod="nginx-demo-1", container="nginx"}

This is a Stream.

If Label values change, a new Stream will be created.

Example:

    {namespace="app-demo", pod="nginx-demo-1", container="nginx"}
    {namespace="app-demo", pod="nginx-demo-2", container="nginx"}

These are two different Streams.

### 4.2 Why Stream Count Matters

The more Streams, the more indexes and chunks Loki needs to maintain.

If Stream count is reasonable:

    Stable writing
    Clear querying
    Chunks are easier to compress
    Lower object storage pressure
    Query scope is controllable

If Stream count is too high:

    Writing pressure increases
    Ingester memory increases
    Indexes grow
    Chunks become fragmented
    Queries slow down
    Object storage object count increases
    Loki cost rises
    May trigger limits or errors

### 4.3 A Wrong Example

Assume putting request_id into Label:

    {namespace="app-demo", app="api", request_id="req-000001"}
    {namespace="app-demo", app="api", request_id="req-000002"}
    {namespace="app-demo", app="api", request_id="req-000003"}

Each request has a unique request_id.

Result:

    Each request may create a new Stream.

If there are 1 million requests per day:

    May create nearly 1 million Streams.

This is a classic cardinality disaster.

---

## FiveI don't know.What is Cardinality

### 5.1 Meaning of Cardinality

Cardinality can be understood as:

    The number of unique label combinations

Or:

    The number of different Label Sets forming Streams

Example:

    namespace has 5 values
    app has 20 values
    container has 3 values

Theoretical combinations may be:

    5 × 20 × 3 = 300

If adding request_id:

    request_id has 1,000,000 values

Combinations may expand to:

    5 × 20 × 3 × 1,000,000 = 300,000,000

This is the high cardinality problem.

### 5.2 Low Cardinality Fields

Low cardinality fields usually have limited and stable values.

Examples:

    cluster
    environment
    namespace
    app
    container
    team
    region
    workload

These fields are suitable for Labels.

### 5.3 High Cardinality Fields

High cardinality fields usually have many values and change rapidly.

Examples:

    request_id
    trace_id
    span_id
    user_id
    order_id
    session_id
    client_ip
    full_url
    error_message
    timestamp
    container_id full value
    pod uid

These fields are typically unsuitable for Loki Labels.

### 5.4 Low Cardinality Does Not Mean Permanent Safety

Even seemingly reasonable fields should be evaluated by actual value counts.

Example:

    pod

In Kubernetes, pod names change continuously with rolling updates.

If the cluster is large and has many Pods, pod may also cause high cardinality.

But pod is very useful for troubleshooting, so it's usually retained.

In production, need to balance:

    Query value
    Value count
    Lifecycle
    Change frequency
    Cost impact

---

## SixI don't know.Recommended Label Design

### 6.1 Recommended Labels for Kubernetes Pod Logs

Recommended:

    cluster
    environment
    namespace
    app
    container
    pod
    node
    team
    workload

Explanation:

    cluster:
        Must be retained in multi-cluster scenarios.

    environment:
        Differentiate between lab / dev / test / staging / prod.

    namespace:
        Core field for Kubernetes troubleshooting.

    app:
        Core field for application-level querying.

    container:
        Required for troubleshooting multi-container Pods.

    pod:
        Required for troubleshooting single Pods.

node:
    Troubleshooting at the node level is required.

team:
    Alert routing and ownership are required.

workload:
    Useful for querying at the Deployment / StatefulSet / DaemonSet level.

### 6.2 Recommended Labels for Learning Environment

A learning environment can use:

    cluster="k8s-lab"
    environment="lab"
    namespace
    app
    pod
    container
    node
    team

This is sufficient to complete:

    Log search by Namespace
    Log search by App
    Log search by Pod
    Log search by Container
    Log search by Node
    Alert routing by Team

### 6.3 Recommended Labels for Production Environment

For production environments, it is recommended to use:

    cluster
    environment
    namespace
    app
    component
    container
    team
    workload
    node

Whether to retain pod labels should be evaluated based on scale.

General recommendation:

    Small to medium scale:
        Retain pod labels for convenient troubleshooting.

    Very large scale:
        Carefully evaluate the cost of using pod as a label.
        Consider combining with structured metadata or retaining pod information in log content.

### 6.4 Discouraged Labels

Not recommended:

    request_id
    trace_id
    span_id
    user_id
    order_id
    session_id
    client_ip
    full_url
    error_message
    stacktrace
    timestamp
    duration_ms
    status_code (consider only if values are limited, but typically better suited for log fields)
    container_id (full value)
    pod_uid

Principles:

    Do not use dynamic fields as labels.
    Do not use fields that differ for each log entry as labels.
    Do not use request-level fields as labels.
    Do not use user-level fields as labels.
    Do not use long text fields as labels.

---

## SevenI don't know.Viewing Current Labels in Loki

### 7.1 Port Forward Loki Gateway

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

### 7.2 View All Label Names

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Expected to see:

    cluster
    namespace
    pod
    container
    node
    app
    team
    environment
    job

### 7.3 View namespace Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

### 7.4 View app Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

### 7.5 View pod Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq

### 7.6 View container Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/container/values" | jq

### 7.7 View node Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/node/values" | jq

---

## EightI don't know.Using LogQL to Observe Label Query Differences

### 8.1 Query by namespace

    {namespace="app-demo"}

Suitable for:

    Viewing all logs in a specific namespace.

Risks:

    If namespace logs are large in volume, the query scope remains broad.

### 8.2 Query by app

    {namespace="app-demo", app="nginx-demo"}

Suitable for:

    Viewing logs for a specific application.

This is a recommended query method.

### 8.3 Query by pod

    {namespace="app-demo", pod=~"nginx-demo-.*"}

Suitable for:

    Locating a specific Pod or a group of Pods.

### 8.4 Query by container

    {namespace="app-demo", container="nginx"}

Suitable for:

    Locating logs for a specific container in a multi-container Pod.

### 8.5 Query by node

    {node="k8s-worker01"}

Suitable for:

    Investigating log anomalies on a specific node.

Note:

    Querying by node may have a broad scope.
    It is recommended to combine with namespace or app.

Example:

    {node="k8s-worker01", namespace="app-demo"}

### 8.6 Query by team

    {team="sre"}

Suitable for:

    Quickly locating ownership in a multi-team environment.

Production alert routing can also use team.

### 8.7 Discouraged Broad-Range Queries

Not recommended:

    {namespace=~".+"} |= "error"

    {pod=~".+"} |= "timeout"

    {container=~".+"} |= "Exception"

Issues:

    Large scanning scope.
    Queries may be slow.
    Can put pressure on Loki.

Preferred:

    {namespace="app-prod", app="api"} |~ "(?i)error|exception|timeout"

---

## NineI don't know.Experiment One: Observing Current Stream Count

### 9.1 Use series API to View Log Streams

Loki API can query the number of series that meet conditions.

View series for app-demo:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' | jq

### 9.2 Count series

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' \
      | jq '.data | length'

### 9.3 View nginx-demo series /think

curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
  --data-urlencode 'match[]={namespace="app-demo", app="nginx-demo"}' \
  | jq

### 9.4 Understanding the Results

If nginx-demo has two replicas, you may see two or more Streams.

Because:

  - Pod names are different
  - Container names are the same
  - Namespace is the same
  - App is the same

If each Pod has different pod Labels, each Pod typically forms at least one independent Stream.

---

## TenI don't know.Experiment Two: Generate More Logs and Observe Label Queries

### 10.1 Access Nginx to Generate Logs

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute inside the container:

    for i in $(seq 1 20); do
      curl -s http://nginx-demo.app-demo.svc.cluster.local/path-${i} >/dev/null
    done

    exit

### 10.2 Query Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "path-"' \
      --data-urlencode 'limit=20' | jq

### 10.3 Query 404

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "404"' \
      --data-urlencode 'limit=20' | jq

### 10.4 Observation Conclusion

This type of query is reasonable:

  - Labels are used to limit the scope.
  - Log body is used to search for dynamic content.

For example:

  - path-1
  - path-2
  - path-3

These paths should not be used as Labels.

They should be searched as log body content.

---

## ElevenI don't know.Experiment Three: Simulate Correct Structured Logs

Nginx access log is not suitable for simulating business JSON logs.

Here we create a simple Job to output structured logs.

### 11.1 Create JSON Log Job

Create file:

    json-log-demo.yaml

Content:

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: json-log-demo
      namespace: app-demo
      labels:
        app: json-log-demo
        team: sre
        environment: lab
    spec:
      template:
        metadata:
          labels:
            app: json-log-demo
            app.kubernetes.io/name: json-log-demo
            team: sre
            environment: lab
        spec:
          restartPolicy: Never
          containers:
            - name: json-log-demo
              image: busybox:1.36
              command:
                - sh
                - -c
                - |
                  i=1
                  while [ $i -le 50 ]; do
                    echo "{\"level\":\"info\",\"service\":\"json-log-demo\",\"msg\":\"request success\",\"status\":200,\"duration_ms\":$((RANDOM % 200)),\"trace_id\":\"trace-$i\",\"user_id\":\"user-$i\"}"
                    echo "{\"level\":\"error\",\"service\":\"json-log-demo\",\"msg\":\"database connection failed\",\"status\":500,\"duration_ms\":$((1000 + RANDOM % 2000)),\"trace_id\":\"trace-error-$i\",\"user_id\":\"user-error-$i\"}"
                    i=$((i+1))
                    sleep 1
                  done

Apply:

    kubectl apply -f json-log-demo.yaml

Check:

    kubectl get job -n app-demo

    kubectl get pod -n app-demo | grep json-log-demo

### 11.2 View kubectl logs

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

Expect to see JSON logs.

### 11.3 Loki Query

Query all:

    {namespace="app-demo", app="json-log-demo"}

Query error:

    {namespace="app-demo", app="json-log-demo"} |= "\"level\":\"error\""

Use JSON parsing:

    {namespace="app-demo", app="json-log-demo"} | json | level="error"

Query status 500:

    {namespace="app-demo", app="json-log-demo"} | json | status=500

Query slow requests:

# 11.4 Key Understandings

trace_id, user_id appear in the log body.

Reasonable query:

    {namespace="app-demo", app="json-log-demo"} | json | trace_id="trace-error-10"

Unreasonable approach:

    Turn trace_id into a Loki Label.

Because trace_id may differ for each request.

---

## Twelve, Experiment Four: Simulating Error Label Design

This experiment is not recommended for long-term configuration retention, only for understanding high cardinality risks.

### 12.1 Wrong Approach

Assume extracting trace_id from JSON logs as a Label.

Pseudo logic:

    Log body:
        trace_id="trace-1"
        trace_id="trace-2"
        trace_id="trace-3"

Wrong Label:

    trace_id="trace-1"
    trace_id="trace-2"
    trace_id="trace-3"

Result:

    Each trace_id may form a new Stream.

### 12.2 Why Not Recommend Actually Changing Production Alloy

Do not extract trace_id as a Label in production for experiments.

Learning environments can simulate through manual push.

### 12.3 Manual Push Simulation of High Cardinality Label

Execute:

    for i in $(seq 1 100); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"bad-label-demo\",
                \"namespace\": \"app-demo\",
                \"app\": \"bad-label-demo\",
                \"trace_id\": \"trace-${i}\"
              },
              \"values\": [
                [\"${TS}\", \"bad label demo log line trace-${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

### 12.4 Check if trace_id Label Appears

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Check trace_id values:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq

Count the number:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq '.data | length'

### 12.5 Check the Number of Series for bad-label-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={job="bad-label-demo"}' \
      | jq '.data | length'

Expected:

    Close to 100

Reason:

    Each trace_id generates a different label set.
    Each different label set is a different stream.

### 12.6 Compare with Correct Approach

Correct approach: trace_id in log body.

Manual push:

    for i in $(seq 1 100); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"good-label-demo\",
                \"namespace\": \"app-demo\",
                \"app\": \"good-label-demo\"
              },
              \"values\": [
                [\"${TS}\", \"{\\\"level\\\":\\\"info\\\",\\\"trace_id\\\":\\\"trace-${i}\\\",\\\"msg\\\":\\\"good label demo log line\\\"}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

Check series count:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={job="good-label-demo"}' \
      | jq '.data | length'

Expected:

    Usually much less than bad-label-demo.

Query a specific trace_id:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="good-label-demo"} | json | trace_id="trace-50"' \
      --data-urlencode 'limit=10' | jq

### 12.7 Experiment Conclusion

Wrong approach:

    trace_id as Label
      ↓
    100 trace_id values
      ↓
    Nearly 100 Streams

Correct approach:

    trace_id in log body
      ↓
    Labels remain stable
      ↓
    Use | json to filter trace_id during queries

Conclusion:

    Do not use dynamic fields as Labels.
    Dynamic fields should be in log body or structured fields.

## ThirteenI don't know.Experiment 5: Observing the Impact of High Cardinality on Querying

### 13.1 Querying bad-label-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="bad-label-demo"}' \
      --data-urlencode 'limit=10' | jq

### 13.2 Querying a Specific trace_id

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="bad-label-demo", trace_id="trace-50"}' \
      --data-urlencode 'limit=10' | jq

It appears querying by trace_id is fast.

But the cost is:

    Loki creates a Stream for each trace_id.
    Write and indexing costs are shifted to the system side.
    Serious issues occur at scale.

### 13.3 Querying good-label-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="good-label-demo"} | json | trace_id="trace-50"' \
      --data-urlencode 'limit=10' | jq

This approach better aligns with Loki's design:

    Use stable Labels to narrow the scope.
    Use log parsing to filter dynamic fields.

### 13.4 Comparison Conclusion

bad-label-demo:

    Querying by trace_id is convenient.
    But Stream count is high.
    Write and indexing costs are high.
    Not suitable for production.

good-label-demo:

    Stream count remains stable.
    Queries require parsing log content.
    More suitable for production.

---

## FourteenI don't know.Alloy Label Configuration Review

In the 06th article about Alloy configuration, we retained these Labels:

    namespace
    pod
    container
    node
    app
    team
    environment
    job
    cluster

Configuration snippet:

    discovery.relabel "pod_logs" {
      targets = discovery.kubernetes.pod.targets

      rule {
        source_labels = ["__meta_kubernetes_namespace"]
        action        = "replace"
        target_label  = "namespace"
      }

      rule {
        source_labels = ["__meta_kubernetes_pod_name"]
        action        = "replace"
        target_label  = "pod"
      }

      rule {
        source_labels = ["__meta_kubernetes_pod_container_name"]
        action        = "replace"
        target_label  = "container"
      }

      rule {
        source_labels = ["__meta_kubernetes_pod_node_name"]
        action        = "replace"
        target_label  = "node"
      }

      rule {
        source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
        action        = "replace"
        target_label  = "app"
      }

      rule {
        source_labels = ["__meta_kubernetes_pod_label_team"]
        action        = "replace"
        target_label  = "team"
      }

      rule {
        source_labels = ["__meta_kubernetes_pod_label_environment"]
        action        = "replace"
        target_label  = "environment"
      }
    }

These Labels are suitable for learning environments.

Production environments should also consider scale evaluation.

---

## FifteenI don't know.How to Remove Unreasonable Labels

### 15.1 Do Not Extract High-Cardinality Fields in relabel

Do not extract the following fields from log content as Labels:

    trace_id
    request_id
    user_id
    order_id

### 15.2 If There Are Already Incorrect Labels

If incorrect Labels have already been created in Alloy configuration, remove the corresponding relabel rules.

Example of an incorrect configuration:

    rule {
      source_labels = ["__meta_kubernetes_pod_annotation_trace_id"]
      action        = "replace"
      target_label  = "trace_id"
    }

This should be removed or modified to not be a Label.

### 15.3 Will Old Data Disappear Immediately After Removing Labels

No.

Old logs already written to Loki still contain old Labels until they are cleaned up after the retention period.

Therefore, in production:

    Label standards must be designed early.
    Incorrect Labels will pollute data for some time.
    After correction, only new data will avoid generating incorrect Labels.

---

## SixteenI don't know.Recommended Label Standards for Production

### 16.1 Basic Standards

Recommended standard Labels:

    cluster
    environment
    namespace
    app
    component
    container
    team
    workload

Optional Labels:

    pod
    node
    version
    region
    zone

Caution Labels:

    pod
    node
    version

Prohibited Labels: /think

request_id
trace_id
span_id
user_id
order_id
session_id
client_ip
full_url
error_message
stacktrace
timestamp

### 16.2 namespace Specification

namespace is a required field that must be retained.

Used for:

    Multi-team isolation
    Query scope control
    Dashboard variables
    Alert routing
    Cost statistics

Example query:

    {namespace="app-prod"}

### 16.3 app Specification

app is strongly recommended to be retained.

Requirements:

    All business Pods must have app or app.kubernetes.io/name label.

Recommendations:

    app.kubernetes.io/name

Compatibility:

    app

Example:

    labels:
      app.kubernetes.io/name: order-api
      app: order-api

### 16.4 team Specification

team is used for alert routing and responsibility assignment.

Example:

    labels:
      team: sre

Query:

    {team="sre"}

Alert routing:

    team=sre → SRE notification channel
    team=payment → Payment team notification channel

### 16.5 environment Specification

environment is used to distinguish environments.

Recommended values:

    lab
    dev
    test
    staging
    prod

Avoid confusing values:

    production
    Prod
    PROD
    pro
    prd

Should be standardized:

    prod

### 16.6 version Specification

version can be used for release troubleshooting.

Example:

    version="v1.2.3"

But use with caution:

    If releases are very frequent, version values will also increase.
    In large-scale environments, evaluate whether to use this as a Label.

---

## SeventeenI don't know.Label and Dashboard Variables

### 17.1 Grafana Common Variables

Recommended variables:

    cluster
    environment
    namespace
    app
    pod
    container
    node
    team

### 17.2 namespace Variable

Query:

    label_values(namespace)

### 17.3 app Variable

Query:

    label_values({namespace="$namespace"}, app)

### 17.4 pod Variable

Query:

    label_values({namespace="$namespace", app="$app"}, pod)

### 17.5 container Variable

Query:

    label_values({namespace="$namespace", pod="$pod"}, container)

### 17.6 Dashboard Design Principles

Do not let users query:

    {namespace=~".+"}

Instead guide users:

    Select cluster first
    Then select namespace
    Then select app
    Then select pod

This reduces broad-range queries.

---

## EighteenI don't know.Label and Alert Routing

### 18.1 Alerts must have team

Log alerts are recommended to include:

    team
    namespace
    app
    pod
    severity
    source

Example:

    PodErrorLogsTooMany

Should include:

    team="sre"
    namespace="app-demo"
    app="nginx-demo"
    source="loki"

### 18.2 AlertManager Routing

Routing can be based on team:

    team=sre       → SRE group
    team=platform  → Platform group
    team=payment   → Payment group

### 18.3 If there is no team

Log alerts without team will lead to:

    Not knowing who to notify
    Alerts unclaimed
    SRE forced to take responsibility
    Difficulty in fault closure

Therefore, it is recommended that business Pods must have:

    team

---

## NineteenI don't know.Label and Log Retention Strategy

Loki retention can be set differently for each stream selector.

Example idea:

    {namespace="app-prod"} retain 30 days
    {namespace="app-dev"} retain 7 days
    {team="security"} retain 90 days

If Label design is poor, it's hard to implement retention strategy.

Therefore in production:

    namespace
    environment
    team

Typically have governance value.

---

## TwentyI don't know.Label and Cost Governance

### 20.1 Where does log cost come from

Log system costs mainly come from:

    Write volume
    Storage volume
    Query volume
    Label cardinality
    Stream count
    Object count
    Compression efficiency
    Retention time

### 20.2 Label design affects cost

Reasonable Label:

    Controllable stream count
    Clear query scope
    Easier chunk compression
    More efficient queries

Wrong Label:

    Stream count explodes
    Chunk becomes fragmented
    Index grows
    Object storage objects increase
    Query and write costs rise

### 20.3 Cost governance recommendations

Recommend to regularly statistics:

    Log volume per namespace
    Log volume per app
    Log volume per team
    Number of label values
    Stream count per app
    Query duration
    Top high log volume applications
    Top high error log applications

---

## Twenty-oneI don't know.Common Mistake Label Design Cases

### 21.1 Using request_id as Label

Error:

    {app="api", request_id="req-123"}

Reason:

    Each request has a unique request_id.
    Stream count explodes.

Correct:

    request_id should be placed in JSON log body.

Query:

    {app="api"} | json | request_id="req-123"

### 21.2 user_id as Label

Error:

    {app="api", user_id="100001"}

Reason:

    Large user base.
    High value variability.
    Query value is not worth the indexing cost.

Correct:

    Place user_id in the log body.

### 21.3 full_url as Label

Error:

    {app="api", full_url="/api/order/123456?token=xxx"}

Issues:

    High cardinality.
    May contain sensitive information.
    High query and storage cost.

Correct:

    path_group="/api/order"

Or process it in application metrics and log body.

### 21.4 error_message as Label

Error:

    {error_message="database connection failed timeout after 3000ms"}

Issues:

    Error messages vary widely.
    Long text is unsuitable for Label.
    Easily generates many distinct values.

Correct:

    error_type="database_error"

Keep full error_message in log body.

### 21.5 status_code as Label

status_code has limited values, e.g.:

    200
    400
    401
    403
    404
    500
    502
    503

Theoretically low cardinality.

But caution is advised.

In access log scenarios, it can also be queried via JSON field or log content:

    {app="api"} | json | status=500

If frequently used in Dashboard and alerts, consider extracting as Label.

Principle:

    Low cardinality + High query frequency + Clear governance value

Only makes it suitable as Label.

---

## Twenty-two, Troubleshooting High Cardinality Issues

### 22.1 Observe Label Names

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

If you see:

    trace_id
    request_id
    user_id
    order_id
    session_id

Be cautious.

### 22.2 Observe Value Count for a Label

Example:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq '.data | length'

If the count is very large, it may indicate a high cardinality issue.

### 22.3 Observe Series Count

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' \
      | jq '.data | length'

By app:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo", app="json-log-demo"}' \
      | jq '.data | length'

### 22.4 Potential Errors in Loki Logs

Check Loki logs:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "cardinality|stream|limit|label|too many|429"

May see:

    too many streams
    maximum active stream limit exceeded
    label value too long
    cardinality limit exceeded
    ingestion rate limit exceeded

### 22.5 Handling Directions

Actions:

    1. Identify abnormal Labels.
    2. Modify Alloy relabel or application log collection rules.
    3. Stop extracting high cardinality fields as Labels.
    4. Keep high cardinality fields in log body or structured metadata.
    5. Wait for old data to expire beyond retention period for natural cleanup.
    6. Perform separate governance for high log volume applications.
    7. Adjust Loki limits if necessary, but do not use increased limits to mask poor design.

---

## Twenty-three, Cleaning Experimental Data

### 23.1 Delete JSON Log Job

    kubectl delete job json-log-demo -n app-demo

### 23.2 Handling bad-label-demo Data

Manually pushing bad-label-demo logs to Loki will not disappear due to deletion of Kubernetes resources.

It will be cleaned up after Loki retention period expires.

In learning environments, it can be retained for future observation.

If cleanup is required, it needs to be handled via Loki deletion API and retention capabilities. Caution is advised in production.

### 23.3 Querying Experimental Data

bad-label-demo:

    {job="bad-label-demo"}

good-label-demo:

    {job="good-label-demo"}

json-log-demo:

    {namespace="app-demo", app="json-log-demo"}

---

## Twenty-four, Production Considerations

### 24.1 Label Specifications Must Be Prioritized

Loki label design should be determined before going live.

Do not wait until log volume increases to govern.

Before going live, clarify:

    Mandatory Labels
    Optional Labels
    Prohibited Labels
    Naming Conventions
    Value Specifications
    Team Responsibility
    Review Process

### 24.2 Label Changes Require Change Process

Modifying Labels may affect:

    Grafana Dashboard
    Loki Alert Rules
    AlertManager Routing
    Query Statements
    Runbook
    Historical Data Queries

Therefore, Label changes should not be done arbitrarily in production.

### 24.3 Maintain Stable Labels

Labels should be stable, low cardinality, and governable.

They should not change frequently.

Examples:

    environment should have fixed values.
    team should have fixed values.
    app should be fixed by service name.
    namespace should be named according to platform standards.

### 24.4 Standardize Application Labels

Kubernetes Pods should have unified labels:

    app.kubernetes.io/name
    app.kubernetes.io/component
    app.kubernetes.io/version
    team
    environment

Alloy can only stably extract.

### 24.5 Governance of High-Risk Log Sources

High-risk log sources:

    Gateway access log
    Nginx ingress log
    High-traffic API service
    Database slow log
    Debug-level DEBUG logs
    AI inference request log
    Batch task log

These logs require special attention:

    Log volume
    Field design
    Whether contains sensitive information
    Whether generates high cardinality
    Whether needs sampling or filtering

---

## Twenty-Five, Practical Tasks

### 25.1 Task One: Check Current Label

Execute:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Verification:

    [ ] Can see namespace
    [ ] Can see pod
    [ ] Can see container
    [ ] Can see app
    [ ] Can see cluster

### 25.2 Task Two: Check Label Values

Execute:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq

Verification:

    [ ] namespace can see app-demo
    [ ] app can see nginx-demo
    [ ] pod can see nginx-demo related Pod

### 25.3 Task Three: Check Stream Count

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' \
      | jq '.data | length'

Verification:

    [ ] Can count the series count of app-demo

### 25.4 Task Four: Deploy JSON Log Job

Execute:

    kubectl apply -f json-log-demo.yaml

    kubectl get pod -n app-demo | grep json-log-demo

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

Verification:

    [ ] JSON logs output normally
    [ ] Loki can query json-log-demo logs
    [ ] Can query level=error via | json

### 25.5 Task Five: Simulate bad-label-demo

Execute:

    for i in $(seq 1 100); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"bad-label-demo\",
                \"namespace\": \"app-demo\",
                \"app\": \"bad-label-demo\",
                \"trace_id\": \"trace-${i}\"
              },
              \"values\": [
                [\"${TS}\", \"bad label demo log line trace-${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

Verification:

    [ ] trace_id label appears
    [ ] trace_id value count approaches 100
    [ ] bad-label-demo series count increases significantly

### 25.6 Task Six: Simulate good-label-demo

Execute:

    for i in $(seq 1 100); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"good-label-demo\",
                \"namespace\": \"app-demo\",
                \"app\": \"good-label-demo\"
              },
              \"values\": [
                [\"${TS}\", \"{\\\"level\\\":\\\"info\\\",\\\"trace_id\\\":\\\"trace-${i}\\\",\\\"msg\\\":\\\"good label demo log line\\\"}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

Query:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="good-label-demo"} | json | trace_id="trace-50"' \
      --data-urlencode 'limit=10' | jq

Verification:

    [ ] trace_id is not a Label
    [ ] Can query trace_id via JSON parsing
    [ ] good-label-demo series count is significantly less than bad-label-demo

---

## Twenty-Six, Acceptance Checklist

After completing this document, confirm:

[ ] Understand what a Label is  
[ ] Understand what a Stream is  
[ ] Understand what Cardinality is  
[ ] Understand why high cardinality is dangerous  
[ ] Be able to name recommended Labels  
[ ] Be able to name prohibited Labels  
[ ] Be able to view Label names via API  
[ ] Be able to view Label values via API  
[ ] Be able to view Stream count via series API  
[ ] Be able to query logs by namespace  
[ ] Be able to query logs by app  
[ ] Be able to query logs by pod  
[ ] Be able to deploy JSON log Demo  
[ ] Be able to use | json to query structured logs  
[ ] Be able to simulate high cardinality issues with trace_id as Label  
[ ] Be able to compare bad-label-demo and good-label-demo  
[ ] Be able to explain why request_id / trace_id / user_id are unsuitable as Labels  
[ ] Be able to write production Loki Label standards  
[ ] Be able to know that incorrect Labels won't immediately disappear from old data  

---

## 27. Common Misconceptions  

### 27.1 Misconception 1: More Labels Are Better  

Incorrect.  

More Labels mean more combinations, potentially more Streams.  

Loki is not suitable for making all fields Labels.  

### 27.2 Misconception 2: Fields convenient for querying should be Labels  

Incorrect.  

Making request_id a Label makes querying convenient, but the cost is high.  

Correct approach:  

    Use stable Labels to narrow the scope.  
    Use log body or JSON fields to filter dynamic fields.  

### 27.3 Misconception 3: Low-cardinality fields are always safe  

Not absolutely.  

For example, pod can also be numerous in a very large cluster.  

Need to evaluate based on scale and query value.  

### 27.4 Misconception 4: Old Labels disappear immediately after deleting relabel rules  

Incorrect.  

Old data still exists.  

Need to wait for retention expiration or execute deletion policies.  

### 27.5 Misconception 5: error_message can be a Label  

Incorrect.  

error_message is usually volatile and long, unsuitable as a Label.  

Can use:  

    error_type="database_error"  

Full error message should be placed in log body.  

---

## 28. Summary  

Loki label design is one of the core capabilities of using Loki.  

Loki is not Elasticsearch.  

Loki's basic approach is:  

    Use low-cardinality Labels to locate log streams  
    Use log body to search for specific content  
    Use JSON / logfmt to parse dynamic fields  
    Do not make every request-level field a Label  

Reasonable Labels:  

    cluster  
    environment  
    namespace  
    app  
    container  
    team  
    workload  
    pod  
    node  

Unreasonable Labels:  

    request_id  
    trace_id  
    span_id  
    user_id  
    order_id  
    session_id  
    client_ip  
    full_url  
    error_message  
    stacktrace  
    timestamp  

This article explains through two experiments:  

    bad-label-demo:  
        Using trace_id as Label generates a large number of Streams.  

    good-label-demo:  
        Placing trace_id in log body, querying via | json, results in more controlled Stream counts.  

Production-grade Loki label standards must be designed upfront.  

Core principles:  

    Low cardinality  
    Stable  
    Frequently used for querying  
    Has governance value  
    Does not contain sensitive information  
    Does not use request-level dynamic fields  
    Avoid overusing Labels  

Next article:  

    08-LogQLBasic queries:Namespace-Pod-ContainerLog Search  

Key learning points:  

    Basic LogQL queries  
    Label selectors  
    String filtering  
    Regular expression filtering  
    Exclusion filtering  
    Time range  
    limit  
    Query logs by Namespace / Pod / Container / App  

---

## Reference Documents  

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/  

- Understand labels:  
  https://grafana.com/docs/loki/latest/get-started/labels/  

- Label best practices:  
  https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/  

- Cardinality:  
  https://grafana.com/docs/loki/latest/get-started/labels/cardinality/  

- Structured Metadata:  
  https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/  

- Modify default OpenTelemetry labels:  
  https://grafana.com/docs/loki/latest/get-started/labels/modify-default-labels/  

- Query Loki:  
  https://grafana.com/docs/loki/latest/query/  

- Query best practices:  
  https://grafana.com/docs/loki/latest/query/bp-query/  

- Loki HTTP API:  
  https://grafana.com/docs/loki/latest/reference/loki-http-api/  

- Grafana Alloy Documentation:  
  https://grafana.com/docs/alloy/latest/  

- discovery.relabel:  
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/  

- loki.process:  
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.process/  

- Kubernetes Recommended Labels:  
  https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/