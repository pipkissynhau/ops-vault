# 08-LogQL Basic Query Practice: Namespace-Pod-Container Log Retrieval

## Document Overview

This is the eighth article in the Loki specialized learning series, used to systematically learn LogQL basic query capabilities, focusing on retrieving Pod logs under Kubernetes scenarios by dimensions such as Namespace, Pod, Container, App, Node, Team, etc.

Previously completed:

    01-Loki Basic Understanding and Experiment Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experiment Selection
    04-Loki Single-Instance Helm Deployment Practice
    05-Loki Object Storage Access MinIO Practice
    06-Grafana-Alloy Collection K8S-Pod Logs Practice
    07-Loki Label Design and High Cardinality Problem Experiment

The 06th article has completed Alloy collection of Kubernetes Pod logs.

The 07th article has clearly explained Label, Stream, Cardinality, and high cardinality issues.

This article officially enters LogQL basic query practice.

This article focuses on answering the following questions:

- What is LogQL;
- What is the relationship between LogQL and PromQL;
- What is the basic query structure of LogQL;
- How to use stream selector;
- How to use label matcher;
- How to use line filter;
- How to query logs by Namespace;
- How to query logs by Pod;
- How to query logs by Container;
- How to query logs by App;
- How to query logs by Node;
- How to query logs containing keywords such as ERROR, timeout, Exception, panic;
- How to use regular expressions to query logs;
- How to exclude health check logs;
- How to query logs through Loki HTTP API;
- How to query logs through Grafana Explore;
- How to control limit, time range, and direction;
- How to troubleshoot when logs are not found;
- What are the production notes in basic queries.

This article only covers basic queries, not in-depth JSON parsing, logfmt parsing, unwrap, and aggregation statistics.

These contents will be placed in the next article:

    09-LogQL Advanced Query Practice: json-logfmt-regexp-unwrap

---

## Tags

#Loki #LogQL #Grafana #Kubernetes #PodLog #LogQueries #Namespace #Pod #Container #SRE #Observation #LogDetachment

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/08-LogQL Basic Query Practice: Namespace-Pod-Container Log Retrieval.md

---

## One, Experiment Objectives

After completing this article, you should be able to:

    1. Understand the basic query structure of LogQL.
    2. Be able to query log streams using Label Selector.
    3. Be able to query logs using namespace.
    4. Be able to query logs using pod.
    5. Be able to query logs using container.
    6. Be able to query logs using app.
    7. Be able to query logs using node.
    8. Be able to query logs containing a specific string using |=.
    9. Be able to exclude logs containing a specific string using !=.
    10. Be able to use |~ for regular expression matching logs.
    11. Be able to use !~ to exclude logs via regular expressions.
    12. Be able to query logs through Loki HTTP API.
    13. Be able to query logs through Grafana Explore.
    14. Be able to control query time range, limit, and direction.
    15. Be able to determine common reasons for not finding logs.
    16. Be able to write common Kubernetes log troubleshooting LogQL.

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
    nginx-demo
    json-log-demo

### 2.2 Prerequisites

Need to confirm:

    [ ] Loki Pod Running
    [ ] Loki Gateway Service exists
    [ ] Alloy DaemonSet Running
    [ ] Alloy can collect Pod logs
    [ ] app-demo Namespace has test applications
    [ ] Loki can find app-demo logs

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Port forward Loki Gateway:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

Check labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

---

## Three, What is LogQL

LogQL is Loki's query language.

Its design style is similar to PromQL, but the core objectives are different.

PromQL mainly queries:

    Metrics

LogQL mainly queries:

    Logs

LogQL can do two things:

    1. Query log content.
    2. Extract and statistically analyze metrics from logs.

This article only covers the first type:

    Query log content

Example:

    {namespace="app-demo"}

    {namespace="app-demo"} |= "ERROR"

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"

The next article will cover:

    | json
    | logfmt
    unwrap
    count_over_time
    rate
    avg_over_time
    quantile_over_time

---

## Four, LogQL Basic Structure

### 4.1 Most Basic Structure

A basic LogQL typically consists of three parts:

    stream selector
      +
    line filter
      +
    label / parsed filter

This article will first focus on the first two parts: /think

# Stream Selector
# Line Filter

Example:

    {namespace="app-demo", app="nginx-demo"} |= "404"

Explanation:

    {namespace="app-demo", app="nginx-demo"}:
        Stream Selector, used to select log streams.

    |= "404":
        Line Filter, used to filter log lines containing "404" in the log body.

### 4.2 Query Order

Recommended approach:

    First use Label to narrow the log scope.
    Then use string or regex to filter the log body.

Example:

    Recommended:
        {namespace="app-prod", app="order-api"} |= "ERROR"

    Not recommended:
        {namespace=~".+"} |= "ERROR"

Reason:

    The latter scans a large range.
    The query is slow.
    It may affect Loki in production environments.

### 4.3 LogQL Query Minimum Unit

Loki queries logs must have a stream selector.

Valid:

    {namespace="app-demo"}

Invalid or not recommended:

    |= "ERROR"

Because Loki needs to know from which log streams to query.

---

## FiveI don't know.Stream Selector Basics

### 5.1 Exact Match

Syntax:

    {label="value"}

Example:

    {namespace="app-demo"}

    {app="nginx-demo"}

    {container="nginx"}

    {namespace="app-demo", app="nginx-demo"}

### 5.2 Not Equal Match

Syntax:

    {label!="value"}

Example:

    {namespace!="kube-system"}

Meaning:

    Query logs streams where namespace is not equal to kube-system.

Note:

    It is not recommended to use broad exclusion queries directly in production.
    It is better to first specify clear namespace or app.

### 5.3 Regular Expression Match

Syntax:

    {label=~"regex"}

Example:

    {pod=~"nginx-demo-.*"}

    {namespace=~"app-.*"}

    {app=~"nginx|api|worker"}

### 5.4 Regular Expression Exclusion

Syntax:

    {label!~"regex"}

Example:

    {namespace!~"kube-system|monitoring|logging"}

Note:

    Regular expression exclusion may expand the query scope.
    It should be used cautiously in production.

### 5.5 Multiple Conditions Combination

Example:

    {namespace="app-demo", app="nginx-demo", container="nginx"}

Meaning:

    namespace must equal app-demo.
    app must equal nginx-demo.
    container must equal nginx.

Multiple Label conditions are AND relationships.

---

## SixI don't know.Line Filter Basics

Line Filter is used to filter log bodies.

### 6.1 Contains String: |=

Syntax:

    |= "string"

Example:

    {namespace="app-demo"} |= "GET"

Meaning:

    Query log lines containing "GET" in app-demo.

### 6.2 Does Not Contain String: !=

Syntax:

    != "string"

Example:

    {namespace="app-demo"} != "/healthz"

Meaning:

    Query log lines not containing "/healthz" in app-demo.

### 6.3 Regular Expression Match: |~

Syntax:

    |~ "regex"

Example:

    {namespace="app-demo"} |~ "(?i)error|exception|panic|timeout"

Meaning:

    Query log lines containing error, exception, panic, or timeout.
    (?i) indicates case-insensitive matching.

### 6.4 Regular Expression Exclusion: !~

Syntax:

    !~ "regex"

Example:

    {namespace="app-demo"} !~ "healthz|metrics|readyz"

Meaning:

    Exclude log lines containing healthz, metrics, or readyz.

### 6.5 Multiple Line Filters in Series

Example:

    {namespace="app-demo"} |= "GET" != "/healthz"

Meaning:

    Contains GET.
    Does not contain /healthz.

Example:

    {namespace="app-demo"} |~ "(?i)error|exception" != "ignore"

Meaning:

    Contains error or exception.
    Does not contain ignore.

---

## SevenI don't know.Basic Query Preparation: Generating Test Logs

### 7.1 Check nginx-demo

    kubectl get pod -n app-demo -o wide

    kubectl get svc -n app-demo

    kubectl get endpoints nginx-demo -n app-demo

### 7.2 Generate Access Logs

Start a temporary curl Pod:

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute inside the container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    curl http://nginx-demo.app-demo.svc.cluster.local/api/v1/orders

    curl http://nginx-demo.app-demo.svc.cluster.local/healthz

    curl http://nginx-demo.app-demo.svc.cluster.local/metrics

Exit:

    exit

### 7.3 Use kubectl logs to Verify

Check Pod:

    kubectl get pod -n app-demo | grep nginx-demo

Check logs:

    kubectl logs <nginx-pod-name> -n app-demo --tail=50

Expected to see:

    GET /
    GET /notfound
    GET /api/v1/orders
    GET /healthz
    GET /metrics

Note: /think

kubectl logs can be seen, which indicates that the application actually has log output.

---

## Eight, Querying Logs via Loki HTTP API

### 8.1 Port Forward Loki Gateway

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

### 8.2 query_range Base Format

Base command:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

Explanation:

    -G:
        Let curl use GET and append parameters.

    --data-urlencode:
        URL-encode LogQL queries to avoid special character errors.

    query:
        LogQL query statement.

    limit:
        Maximum number of log entries returned.

### 8.3 Query Result Structure

The returned result generally includes:

    status
    data
      resultType
      result
        stream
        values

Where:

    stream:
        Label of the log stream.

    values:
        Log timestamp and content.

Example structure:

    "stream": {
      "namespace": "app-demo",
      "pod": "nginx-demo-xxx",
      "container": "nginx"
    }

    "values": [
      ["timestamp", "log line"]
    ]

### 8.4 Only View Log Content

If you only want to quickly view log content, you can extract it using jq:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' \
      | jq -r '.data.result[].values[][1]'

### 8.5 Only View stream Labels

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=5' \
      | jq '.data.result[].stream'

---

## Nine, Querying Namespace Logs

### 9.1 Query All Logs for app-demo

LogQL:

    {namespace="app-demo"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

### 9.2 Query logging Namespace Logs

LogQL:

    {namespace="logging"}

Purpose:

    View Loki / Alloy related logs.

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="logging"}' \
      --data-urlencode 'limit=20' | jq

### 9.3 Query Multiple Namespaces

LogQL:

    {namespace=~"app-demo|logging"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace=~"app-demo|logging"}' \
      --data-urlencode 'limit=20' | jq

### 9.4 Exclude System Namespaces

LogQL:

    {namespace!~"kube-system|monitoring|logging"}

Note:

    This query may cover a large scope.
    Use with caution in production.

Recommended alternative:

    {namespace="app-prod"}

---

## Ten, Querying Pod Logs

### 10.1 View Pod Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq

### 10.2 Query Specific Pod

LogQL:

    {namespace="app-demo", pod="nginx-demo-xxxx"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod="nginx-demo-xxxx"}' \
      --data-urlencode 'limit=20' | jq

### 10.3 Use Regex to Query a Group of Pods

Deployment-created Pod names include random suffixes.

More commonly used:

    {namespace="app-demo", pod=~"nginx-demo-.*"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"}' \
      --data-urlencode 'limit=20' | jq

### 10.4 Query Logs Containing 404 in Pod

LogQL:

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"' \
      --data-urlencode 'limit=20' | jq

### 10.5 Query Recent Errors for a Specific Pod

LogQL:

    {namespace="app-demo", pod=~"nginx-demo-.*"} |~ "(?i)error|exception|panic|timeout|failed"

Note:

(?i) indicates case-insensitive matching.
Suitable for quick error log checks.

---

## 11. Querying Container Logs

### 11.1 View Container Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/container/values" | jq

### 11.2 Query nginx Container Logs

LogQL:

    {namespace="app-demo", container="nginx"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", container="nginx"}' \
      --data-urlencode 'limit=20' | jq

### 11.3 Multi-container Pod Query Strategy

If a Pod has multiple containers:

    app
    sidecar
    istio-proxy
    log-agent

Query separately:

    {namespace="app-prod", pod="api-xxx", container="app"}

    {namespace="app-prod", pod="api-xxx", container="istio-proxy"}

Suitable for troubleshooting:

    Main business container logs
    Sidecar logs
    Service Mesh proxy logs
    Init container logs

### 11.4 Query Non-sidecar Logs

If you want to exclude istio-proxy:

    {namespace="app-prod", container!="istio-proxy"}

Note:

    If app-prod is large, recommend adding the app label:

    {namespace="app-prod", app="order-api", container!="istio-proxy"}

---

## 12. Querying App Logs

### 12.1 View App Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

### 12.2 Query nginx-demo

LogQL:

    {namespace="app-demo", app="nginx-demo"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode 'limit=20' | jq

### 12.3 Query App Access Logs

LogQL:

    {namespace="app-demo", app="nginx-demo"} |= "GET"

### 12.4 Query App 404 Logs

LogQL:

    {namespace="app-demo", app="nginx-demo"} |= "404"

### 12.5 Query App Non-healthcheck Logs

LogQL:

    {namespace="app-demo", app="nginx-demo"} != "/healthz" != "/metrics"

### 12.6 Significance of App Queries

Compared to pod queries, app queries are better suited for:

    A Deployment with multiple replicas
    Multiple Pods coexisting during rolling updates
    Querying entire application logs
    Dashboard panels
    Alerting rules

Production troubleshooting recommends prioritizing:

    namespace + app

Then drilling down to:

    pod + container

---

## 13. Querying Node Logs

### 13.1 View Node Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/node/values" | jq

### 13.2 Query Logs on a Specific Node

LogQL:

    {node="k8s-worker01"}

Note:

    The query scope may be large.
    Recommended to combine with namespace.

Preferred:

    {node="k8s-worker01", namespace="app-demo"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={node="k8s-worker01", namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

### 13.3 Scenarios for Node Queries

Suitable for:

    Multiple Pods on a node experiencing issues.
    Suspected node network or disk anomalies.
    Alloy collection anomalies on a node.
    Missing Pod logs on a node.
    A node experiencing eviction or container restarts.

Query strategy:

    Start by discovering Node anomalies from Prometheus.
    Then use Loki to check business logs on the Node.
    Finally use kubectl describe node / journalctl to troubleshoot the underlying issues.

---

## 14. Querying Team/Environment Logs

### 14.1 Query Team Logs

LogQL:

    {team="sre"}

Preferred:

    {environment="lab", team="sre"}

Or:

    {namespace="app-demo", team="sre"}

### 14.2 Query Environment Logs

LogQL:

    {environment="lab"}

Common in production:

    {environment="prod", namespace="order"}

### 14.3 Roles of Team and Environment

Team is suitable for:

    Alert routing
    Responsibility assignment
    Multi-team log governance
    Cost statistics

Environment is suitable for:

    Distinguishing development, testing, pre-production, and production
    Controlling query scope
    Distinguishing retention periods
    Distinguishing alerting strategies

---

## 15. Querying Common Error Keywords

### 15.1 Query ERROR

LogQL:

    {namespace="app-demo"} |= "ERROR"

If log case is inconsistent, recommend regex:

    {namespace="app-demo"} |~ "(?i)error"

### 15.2 Query Exception

    {namespace="app-demo"} |~ "(?i)exception"

### 15.3 Query panic

    {namespace="app-demo"} |~ "(?i)panic"

### 15.4 Query timeout

### 15.5 Query connection refused

    {namespace="app-demo"} |~ "(?i)connection refused|connect refused"

### 15.6 Query database errors

    {namespace="app-demo"} |~ "(?i)database|mysql|postgres|connection failed|too many connections"

### 15.7 Query Redis errors

    {namespace="app-demo"} |~ "(?i)redis|connection refused|timeout|NOAUTH"

### 15.8 Query OOM / memory

    {namespace="app-demo"} |~ "(?i)oom|out of memory|memory"

### 15.9 Query CUDA OOM

    {namespace="app-demo"} |~ "(?i)cuda out of memory|CUDA error|out of memory"

### 15.10 Recommended combinations

Do not query globally directly in production:

    {namespace=~".+"} |~ "(?i)error|exception|panic"

Recommended instead:

    {environment="prod", namespace="order", app="order-api"} |~ "(?i)error|exception|panic"

---

## SixteenI don't know.Exclude irrelevant logs

### 16.1 Exclude health checks

LogQL:

    {namespace="app-demo"} != "/healthz"

### 16.2 Exclude metrics

    {namespace="app-demo"} != "/metrics"

### 16.3 Exclude multiple paths simultaneously

    {namespace="app-demo"} != "/healthz" != "/metrics" != "/readyz"

### 16.4 Use regular expressions to exclude

    {namespace="app-demo"} !~ "healthz|metrics|readyz"

### 16.5 Query errors but exclude known noise

Example:

    {namespace="app-prod", app="api"} |~ "(?i)error|exception" != "expected error" != "ignore"

### 16.6 Production recommendations

Health checks, probes, and metrics requests typically generate large volumes of logs.

Recommendations:

    Reduce health check log level at the application side.
    Filter health check access logs at Ingress / Nginx side.
    Be cautious about filtering at Alloy side.
    Exclude irrelevant logs at Loki query side.

Priority:

    Application-side governance > Collection-side filtering > Query-side exclusion

---

## SeventeenI don't know.Use start, end, limit, direction

### 17.1 limit

limit controls the number of returned log entries.

Example:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=5' | jq

Notes:

    limit is not a scan upper limit.
    It simply limits the number of returned results.
    Large range queries may still consume resources.

### 17.2 direction

direction controls the return direction.

Common values:

    BACKWARD
    FORWARD

Query recent logs typically use:

    direction=BACKWARD

Example:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' \
      --data-urlencode 'direction=BACKWARD' | jq

### 17.3 start and end

start / end can specify the query time range.

Loki API supports Unix nanoseconds timestamps.

Example: Query the last 10 minutes.

    END=$(date +%s%N)
    START=$(date -d "10 minutes ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq

### 17.4 date command compatibility note

Linux GNU date supports:

    date -d "10 minutes ago" +%s%N

macOS default date does not support this syntax.

This document's experimental environment follows Linux operations.

### 17.5 Query the last 1 hour

    END=$(date +%s%N)
    START=$(date -d "1 hour ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=50' \
      --data-urlencode 'direction=BACKWARD' | jq

### 17.6 Production query recommendations

Do not default to querying long time ranges in production.

Recommendations:

    Temporary troubleshooting:
        Last 15 minutes to 1 hour.

    Incident review:
        Query by fault time window.

    Trend statistics:
        Use Dashboard and aggregated queries, do not manually scan large log volumes.

---

## EighteenI don't know.Query via Grafana Explore

### 18.1 Open Grafana

If Grafana is already deployed:

    kubectl port-forward svc/<grafana-service-name> 3000:80 -n monitoring

Access:

    http://127.0.0.1:3000

### 18.2 Select Loki Data Source

Navigate to:

    Explore
      ↓
    Select Loki Data Source

### 18.3 Basic Queries

Query app-demo:

    {namespace="app-demo"}

Query nginx-demo:

    {namespace="app-demo", app="nginx-demo"}

Query Pod:

    {namespace="app-demo", pod=~"nginx-demo-.*"}

Query 404:

    {namespace="app-demo", app="nginx-demo"} |= "404"

Query errors:

    {namespace="app-demo"} |~ "(?i)error|exception|panic|timeout"

### 18.4 Use Time Range

In Grafana's top-right corner, select:

    Last 5 minutes
    Last 15 minutes
    Last 1 hour
    Last 6 hours

Troubleshooting advice:

    Start with Last 15 minutes.
    If nothing is found, expand the time range based on the fault occurrence time.
    Do not start with 7 days.

### 18.5 View Log Details

Expand log lines in Grafana to see:

    Timestamp
    Log content
    Labels
    Detected fields

Troubleshooting focus:

    Are the labels correct?
    Is the namespace correct?
    Is the pod correct?
    Is the container correct?
    Is the app correct?
    Is the node correct?

---

## Nineteen, Basic Query Troubleshooting Process

### 19.1 Overall Process for No Log Results

If querying:

    {namespace="app-demo"}

yields no results, troubleshoot in this order:

    1. Can kubectl logs see the logs?
    2. Is the Alloy Pod Running?
    3. Does Alloy cover the Pod's Node?
    4. Does Alloy have permission to read Pod logs?
    5. Are Alloy logs failing to send?
    6. Is Loki /ready ready?
    7. Is Loki receiving labels?
    8. Is the time range correct?
    9. Are label names written incorrectly?
    10. Are label values written incorrectly?
    11. Were the labels dropped by Alloy relabel?
    12. Did you query the wrong Loki instance?

### 19.2 Step 1: Confirm Application Has Logs

    kubectl logs <pod-name> -n app-demo --tail=50

If there are no logs here:

    The application may not be outputting stdout / stderr.
    Requests may not be reaching the Pod.
    The wrong Pod may be being checked.
    The wrong container may be being checked.
    The container may have restarted; use --previous.

### 19.3 Step 2: Confirm Alloy is Working

    kubectl get pods -n logging -o wide | grep alloy

    kubectl logs <alloy-pod-name> -n logging --tail=200

Filter:

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|loki|forbidden|denied|timeout"

### 19.4 Step 3: Confirm Loki Labels

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

If there are no namespace / pod / container:

    Alloy hasn't collected them.
    relabel hasn't taken effect.
    Logging to Loki failed.
    The time range is incorrect.

### 19.5 Step 4: Confirm Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq

If app-demo doesn't exist:

    The collection scope may not include app-demo.
    app-demo may have no logs recently.
    Alloy's configuration may have filtered out this Namespace.

### 19.6 Step 5: Expand Query Scope

First query a broader scope:

    {namespace="app-demo"}

If there are results, narrow down to:

    {namespace="app-demo", app="nginx-demo"}

If the app label doesn't exist, start with pod:

    {namespace="app-demo", pod=~"nginx-demo-.*"}

### 19.7 Step 6: Check Time Range

If querying via API, explicitly specify the last 1 hour:

    END=$(date +%s%N)
    START=$(date -d "1 hour ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' | jq

Also confirm the time range in Grafana Explore.

---

## Twenty, Common Troubleshooting Query Templates

### 20.1 Query Recent Logs for a Namespace

    {namespace="app-demo"}

### 20.2 Query Recent Logs for an Application

    {namespace="app-demo", app="nginx-demo"}

### 20.3 Query Recent Logs for a Pod

    {namespace="app-demo", pod="nginx-demo-xxxx"}

### 20.4 Query Logs for a Group of Pods in a Deployment

    {namespace="app-demo", pod=~"nginx-demo-.*"}

### 20.5 Query Logs for a Container

### 20.6 Query Business Logs on a Specific Node

    {namespace="app-demo", node="k8s-worker01"}

### 20.7 Query Error Logs

    {namespace="app-demo"} |~ "(?i)error|exception|panic|failed"

### 20.8 Query Timeout

    {namespace="app-demo"} |~ "(?i)timeout|timed out|deadline exceeded"

### 20.9 Query Database Connection Issues

    {namespace="app-demo"} |~ "(?i)database|mysql|postgres|connection refused|too many connections"

### 20.10 Query 5xx

Nginx access log scenario:

    {namespace="app-demo", app="nginx-demo"} |~ " 5[0-9][0-9] "

Note:

    This is a simple regex based on the default Nginx log format.
    Structured logs are recommended to use | json in the next section.

### 20.11 Query 404

    {namespace="app-demo", app="nginx-demo"} |= "404"

### 20.12 Exclude Health Checks

    {namespace="app-demo", app="nginx-demo"} != "/healthz" != "/metrics"

### 20.13 Query ERROR but Exclude Known Noise

    {namespace="app-prod", app="api"} |~ "(?i)error|exception" != "expected error" != "ignore"

### 20.14 Query CUDA OOM

    {namespace="ai-prod"} |~ "(?i)cuda out of memory|CUDA error|out of memory"

### 20.15 Query Java Exceptions

    {namespace="app-prod"} |~ "(?i)Exception|Caused by|NullPointerException|OutOfMemoryError"

### 20.16 Query Python Traceback

    {namespace="app-prod"} |~ "(?i)Traceback|ModuleNotFoundError|KeyError|ValueError|RuntimeError"

---

## Twenty-one, Performance Baseline Query Principles

### 21.1 Use Label to Narrow Scope First

Recommended:

    {namespace="app-prod", app="order-api"} |= "ERROR"

Not Recommended:

    {namespace=~".+"} |= "ERROR"

### 21.2 Avoid Broad Regular Expressions

Not Recommended:

    {pod=~".+"} |~ "(?i)error"

Recommended:

    {namespace="app-prod", app="order-api"} |~ "(?i)error"

### 21.3 Control Time Range

Not Recommended:

    Directly query ERROR for Last 7 days

Recommended:

    First check Last 15 minutes
    Then expand the scope based on the fault time

### 21.4 Control Limit

Set reasonable limit when using API queries:

    limit=20
    limit=50
    limit=100

Do not return massive logs without thinking.

### 21.5 Avoid Using Log System as Full-Text Search Platform

Loki is suitable for:

    First use label to narrow the scope
    Then query log content

If you need complex full-text search, audit, security search, or field search with many fields, consider Elasticsearch / OpenSearch.

---

## Twenty-two, Basic Query and Production Troubleshooting Integration

### 22.1 Pod CrashLoopBackOff

First check Pod:

    kubectl get pod -n app-prod

Then query logs:

    {namespace="app-prod", pod="api-xxx"} |~ "(?i)error|exception|panic|failed"

Also check previous logs:

    kubectl logs api-xxx -n app-prod --previous --tail=100

### 22.2 Service 5xx Increase

Detected by Prometheus:

    ServiceHigh5xxErrorRate

Loki query:

    {namespace="app-prod", app="order-api"} |~ " 5[0-9][0-9] "

Or:

    {namespace="app-prod", app="order-api"} |~ "(?i)error|exception|database|timeout"

### 22.3 Pod Running but Business Abnormal

Query:

    {namespace="app-prod", app="api"} |~ "(?i)error|timeout|connection refused|database"

Also check:

    kubectl get endpoints <service-name> -n app-prod

    kubectl describe svc <service-name> -n app-prod

### 22.4 Business Abnormal on a Specific Node

After detecting Node anomaly via Prometheus:

    {node="k8s-worker01", namespace="app-prod"} |~ "(?i)error|timeout|exception"

Combined with:

    kubectl get pod -A -o wide | grep k8s-worker01

    kubectl describe node k8s-worker01

---

## Twenty-three, Hands-on Tasks

### 23.1 Task One: Confirm Labels

Execute:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Acceptance: /think

[ ] Can see namespace  
[ ] Can see pod  
[ ] Can see container  
[ ] Can see app  
[ ] Can see node  

### 23.2 Task Two: Generate Nginx Logs  

Execute:  

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh  

Inside the container:  

    curl http://nginx-demo.app-demo.svc.cluster.local  

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound  

    curl http://nginx-demo.app-demo.svc.cluster.local/healthz  

    curl http://nginx-demo.app-demo.svc.cluster.local/metrics  

    exit  

Verification:  
[ ] kubectl logs can see Nginx access log  
[ ] Loki can find new logs  

### 23.3 Task Three: Query by Namespace  

Execute:  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq  

Verification:  
[ ] Can see app-demo logs  

### 23.4 Task Four: Query by Pod  

Execute:  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"}' \
      --data-urlencode 'limit=20' | jq  

Verification:  
[ ] Can see nginx-demo Pod logs  

### 23.5 Task Five: Query by Container  

Execute:  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", container="nginx"}' \
      --data-urlencode 'limit=20' | jq  

Verification:  
[ ] Can see nginx container logs  

### 23.6 Task Six: Query 404  

Execute:  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "404"' \
      --data-urlencode 'limit=20' | jq  

Verification:  
[ ] Can query /notfound related logs  

### 23.7 Task Seven: Exclude Health Checks  

Execute:  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} != "/healthz" != "/metrics"' \
      --data-urlencode 'limit=20' | jq  

Verification:  
[ ] Query results do not include /healthz  
[ ] Query results do not include /metrics  

### 23.8 Task Eight: Query Last 10 Minutes  

Execute:  

    END=$(date +%s%N)  
    START=$(date -d "10 minutes ago" +%s%N)  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq  

Verification:  
[ ] Can query logs from last 10 minutes  
[ ] Can understand start/end are nanoseconds timestamps  

---

## Twenty-Four, Verification Checklist  

After completing this document, should confirm:  

    [ ] Understand LogQL basic structure  
    [ ] Understand stream selector  
    [ ] Understand line filter  
    [ ] Can use = for exact Label matching  
    [ ] Can use != to exclude Label  
    [ ] Can use =~ for regex Label matching  
    [ ] Can use !~ to exclude Label via regex  
    [ ] Can use |= to query containing strings  
    [ ] Can use != to exclude strings  
    [ ] Can use |~ for regex matching log content  
    [ ] Can use !~ to exclude log content via regex  
    [ ] Can query logs by namespace  
    [ ] Can query logs by app  
    [ ] Can query logs by pod  
    [ ] Can query logs by container  
    [ ] Can query logs by node  
    [ ] Can query ERROR / timeout / exception  
    [ ] Can exclude healthz / metrics  
    [ ] Can use query_range API to query logs  
    [ ] Can control limit  
    [ ] Can control start / end  
    [ ] Can control direction  
    [ ] Can execute basic queries in Grafana Explore  
    [ ] Can troubleshoot common issues for missing logs  
    [ ] Understand to first narrow scope with Labels in production  

---

## Twenty-Five, Common Misconceptions  

### 25.1 Misconception One: LogQL can search full text without Labels  

Error.  

Loki queries should first use stream selector.  

Correct:  

    {namespace="app-demo"} |= "ERROR"  

Incorrect understanding:  

    Search directly for ERROR  

### 25.2 Misconception Two: Wider queries are more convenient  

Error.  

Wider queries scan larger ranges and are slower.  

Not recommended:  

    {namespace=~".+"} |= "ERROR"  

Recommended:  

    {environment="prod", namespace="order", app="order-api"} |= "ERROR"  

### 25.3 Misconception Three: limit can restrict scan cost  

Partially correct.  

limit restricts returned results, not Loki's scan cost.  

True cost reduction method: /think

# Narrowing Label Scope  
# Narrowing Time Range  
# Avoid Broad Regular Expressions  
# Design Labels Reasonably  

### 25.4 Misconception Four: Not Finding Logs Definitely Means the Application Has No Logs  

Not necessarily.  

It could also be:  

    Alloy hasn't collected it.  
    Loki hasn't written it.  
    Labels are written incorrectly.  
    Time range is incorrect.  
    Querying the wrong Loki.  
    Logs are filtered.  
    App label doesn't exist.  

### 25.5 Misconception Five: Regular Expressions Are the Most Universal Query Method  

Regular expressions are convenient but have higher costs.  

Prioritize exact strings when possible:  

    |= "ERROR"  

Use regular expressions when multiple keywords are needed:  

    |~ "(?i)error|exception|panic"  

### 25.6 Misconception Six: Pod Names Are Suitable for Long-Term Fixed Queries  

Pod names change.  

Pod names change after Deployment rolling updates.  

For long-term Dashboards and alerts, recommend:  

    namespace + app  

Drill down to:  

    pod + container  

---

## Twenty-Six, Production Notes  

### 26.1 Production Queries Should Start with Small Ranges  

Recommended order:  

    cluster  
      ↓  
    environment  
      ↓  
    namespace  
      ↓  
    app  
      ↓  
    pod  
      ↓  
    container  

Don't start with global queries.  

### 26.2 Dashboards Should Guide Users to Select Variables  

Grafana Dashboards should design variables:  

    cluster  
    environment  
    namespace  
    app  
    pod  
    container  

Avoid users directly querying:  

    {namespace=~".+"}  

### 26.3 Log Queries Should Be Combined with Metric Alarms  

Production troubleshooting should be used as:  

    Prometheus alarm detects anomalies  
      ↓  
    Grafana metrics locate app/pod  
      ↓  
    Loki queries logs by app/pod  
      ↓  
    kubectl describe view Events  
      ↓  
    Service/Endpoints verify traffic  

### 26.4 Health Check Logs Should Be Governed  

If health check logs are large, prioritize reducing output at the application or gateway side.  

Excluding logs on the query side is only a temporary solution.  

### 26.5 Don't Let Loki Become an Infinite Full-Text Search System  

Loki is suitable for:  

    Label filtering + log filtering  

Not suitable for:  

    Unbounded full-text search  
    Large-scale audit search  
    High-frequency search on all fields  

For complex full-text search, consider:  

    Elasticsearch  
    OpenSearch  

---

## Twenty-Seven, Summary  

This article completes LogQL basic query practice.  

The core of basic LogQL is:  

    Stream Selector  
      +  
    Line Filter  

Common Label Queries:  

    {namespace="app-demo"}  

    {namespace="app-demo", app="nginx-demo"}  

    {namespace="app-demo", pod=~"nginx-demo-.*"}  

    {namespace="app-demo", container="nginx"}  

Common Content Filtering:  

    |= "ERROR"  
    != "/healthz"  
    |~ "(?i)error|exception|panic"  
    !~ "healthz|metrics"  

Recommended Query Approach:  

    First narrow the scope with Labels.  
    Then use string filtering.  
    Use regular expressions when necessary.  
    Control time range.  
    Control returned quantity.  
    Don't directly perform global searches.  

Common Kubernetes Troubleshooting Queries:  

    Pod Dimension:  
        {namespace="app-demo", pod=~"nginx-demo-.*"}  

    Container Dimension:  
        {namespace="app-demo", container="nginx"}  

    App Dimension:  
        {namespace="app-demo", app="nginx-demo"}  

    Error Logs:  
        {namespace="app-demo"} |~ "(?i)error|exception|panic|timeout"  

    Exclude Health Checks:  
        {namespace="app-demo"} != "/healthz" != "/metrics"  

In production, the most important is:  

    Don't start with large ranges.  
    Don't overuse regular expressions.  
    Don't rely on Pod names for long-term queries.  
    Don't treat Loki as an infinite full-text search system.  
    Queries must combine with label design, time range, and business context.  

Next article will enter:  

    09-LogQL Advanced Query Practice: json-logfmt-regexp-unwrap  

Focus on learning:  

    JSON log parsing  
    logfmt log parsing  
    regexp extract fields  
    line_format format output  
    label_format generate labels  
    unwrap numeric fields  
    duration_ms query  
    status >= 500 query  
    generate metrics from logs  

---

## Reference Documents  

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/  

- Query Loki:  
  https://grafana.com/docs/loki/latest/query/  

- Log queries:  
  https://grafana.com/docs/loki/latest/query/log_queries/  

- LogQL Reference:  
  https://grafana.com/docs/loki/latest/query/query_reference/  

- Query examples:  
  https://grafana.com/docs/loki/latest/query/query_examples/  

- Loki HTTP API:  
  https://grafana.com/docs/loki/latest/reference/loki-http-api/  

- LogCLI getting started:  
  https://grafana.com/docs/loki/latest/query/logcli/getting-started/  

- Understand labels:  
  https://grafana.com/docs/loki/latest/get-started/labels/  

- Query best practices:  
  https://grafana.com/docs/loki/latest/query/bp-query/  

- Grafana Explore:  
  https://grafana.com/docs/grafana/latest/explore/  

- Kubernetes Logging Architecture:  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/  

- Kubernetes kubectl logs:  
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/