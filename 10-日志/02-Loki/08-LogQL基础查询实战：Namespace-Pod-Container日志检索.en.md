# 08-LogQL Basic Query Practice: Namespace-Pod-Container Log Retrieval

## Document Description

This article is the eighth in the Loki-focused series, designed to systematically teach the basic query capabilities of LogQL, with a focus on retrieving Pod logs in Kubernetes scenarios based on dimensions such as Namespace, Pod, Container, App, Node, and Team.

Previously completed tasks include:

    01-Loki Basic Understanding and Experimental Environment Setup
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experimental Selection
    04-Loki Single-Node Mode Helm Deployment Practice
    05-Loki Object Storage Integration with MinIO Practice
    06-Grafana-Alloy for Collecting Kubernetes-Pod Logs Practice
    07-Loki Tag Design and High Cardinality Issues Experiment

In Article 06, Alloy was used to collect Kubernetes Pod logs.

Article 07 clearly explained Tags, Streams, Cardinality, and high cardinality issues.

This article officially delves into LogQL basic query practice.

It addresses the following key questions:

- What is LogQL?
- What is the relationship between LogQL and PromQL?
- What is the basic structure of LogQL queries?
- How to use a stream selector?
- How to use a label matcher?
- How to use a line filter?
- How to query logs by Namespace?
- How to query logs by Pod?
- How to query logs by Container?
- How to query logs by App?
- How to query logs by Node?
- How to search for keywords such as ERROR, timeout, Exception, panic?
- How to use regular expressions in log queries?
- How to exclude health check logs?
- How to query logs through the Loki HTTP API?
- How to query logs through Grafana Explore?
- How to control limit, time range, and direction?
- How to troubleshoot when no logs are found?
- What are some production considerations for basic queries?

This article only covers basic queries and does not delve into JSON parsing, logfmt parsing, unwrap, or aggregate statistics.

These topics will be covered in the next article:

    09-LogQL Advanced Query Practice: json-logfmt-regexp-unwrap

---

## Tags

#Loki #LogQL #Grafana #Kubernetes #Pod Logs #Log Queries #Namespace #Pod #Container #SRE #Observability #Log Troubleshooting

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/08-LogQL Basic Query Practice: Namespace-Pod-Container Log Retrieval.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Understand the basic structure of LogQL queries.
    2. Use a Label Selector to query log streams.
    3. Use namespace to query logs.
    4. Use pod to query logs.
    5. Use container to query logs.
    6. Use app to query logs.
    7. Use node to query logs.
    8. Use |= to query logs containing a specific string.
    9. Use != to exclude logs containing a specific string.
    10. Use |~ for regular expression matching of logs.
    11. Use !~ for regular expression exclusion of logs.
    12. Query logs through the Loki HTTP API.
    13. Query logs through Grafana Explore.
    14. Control the query time range, limit, and direction.
    15. Identify common reasons for not finding logs when querying.
    16. Write common LogQL queries for Kubernetes log troubleshooting.

---

## II. Experimental Environment

### 2.1 Kubernetes Cluster

Experimental nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespaces:

    logging
    app-demo
    monitoring
    minio

Components deployed:

    Loki
    Loki Gateway
    Grafana Alloy
    MinIO
    nginx-demo
    json-log-demo

### 2.2 Prerequisites

Confirm the following:

    [ ] The Loki Pod is running.
    [ ] The Loki Gateway Service exists.
    [ ] The Alloy DaemonSet is running.
    [ ] Alloy can collect Pod logs.
    [ ] There is a test application in the app-demo Namespace.
    [ ] App-demo logs are available in Loki.

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Forward ports for Loki Gateway:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verify:

    curl -s http### 6.1 Strings Included: |=

Syntax:

    |= "string"

Example:

    {namespace="app-demo"} |= "GET"

Meaning:

    Query log lines in app-demo that contain the word GET.

### 6.2 Strings Not Included: !=

Syntax:

    != "string"

Example:

    {namespace="app-demo"} != "/healthz"

Meaning:

    Query log lines in app-demo that do not contain the path /healthz.

### 6.3 Regular Expression Matching: |~

Syntax:

    |~ "regex"

Example:

    {namespace="app-demo"} |~ "(?i)error|exception|panic|timeout"

Meaning:

    Query log lines that contain any of the words error, exception, panic, or timeout.
    (?i) indicates case-insensitivity.

### 6.4 Regular Expression Exclusion: !~

Syntax:

    !~ "regex"

Example:

    {namespace="app-demo"} !~ "healthz|metrics|readyz"

Meaning:

    Exclude log lines that contain the words healthz, metrics, or readyz.

### 6.5 Multiple Line Filters Chained

Example:

    {namespace="app-demo"} |= "GET" != "/healthz"

Meaning:

    Include log lines that contain GET.
    At the same time, exclude those that contain /healthz.

Example:

    {namespace="app-demo"} |~ "(?i)error|exception" != "ignore"

Meaning:

    Include log lines that contain error or exception.
    Exclude those that contain ignore.curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"}' \
      --data-urlencode 'limit=20' | jq

### 10.4 Query logs in Pods that contain "404"

LogQL:

    {namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"} |= "404"' \
      --data-urlencode 'limit=20' | jq

### 10.5 Query recent errors in a specific Pod

LogQL:

    {namespace="app-demo", pod=~"nginx-demo-.*"} |~ "(?i)error|exception|panic|timeout|failed"

Note:

    (?i) indicates case-insensitivity.
    Suitable for quickly searching for error-related logs.

---

## XI. Querying Container Logs

### 11.1 View container label values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/container/values" | jq

### 11.2 Query nginx container logs

LogQL:

    {namespace="app-demo", container="nginx"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", container="nginx"}' \
      --data-urlencode 'limit=20' | jq

### 11.3 Querying multiple containers in a Pod

If a Pod contains multiple containers:

    app
    sidecar
    istio-proxy
    log-agent

You can query them separately:

    {namespace="app-prod", pod="api-xxx", container="app"}

    {namespace="app-prod", pod="api-xxx", container="istio-proxy"}

This is useful for troubleshooting:

    Main business container logs
    Sidecar logs
    Service Mesh proxy logs
    Init container logs

### 11.4 Querying non-sidecar logs

If you want to exclude istio-proxy:

    {namespace="app-prod", container!="istio-proxy"}

Note:

    If app-prod is large, it's recommended to add the app label as well:

    {namespace="app-prod", app="order-api", container!="istio-proxy"}

---

## XII. Querying App Logs

### 12.1 View app label values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

### 12.2 Query nginx-demo

LogQL:

    {namespace="app-demo", app="nginx-demo"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode 'limit=20' | jq

### 12.3 Query access logs within the app

LogQL:

    {namespace="app-demo", app="nginx-demo"} |= "GET"

### 12.4 Query 404 errors within the app

LogQL:

    {namespace="app-demo", app="nginx-demo"} |= "404"

### 12.5 Query non-healthy checks within the app

LogQL:

    {namespace="app-demo", app="nginx-demo"} != "/healthz" != "/metrics"

### 12.6 The significance of app queries

Compared to pod queries, app queries are more suitable for:

    Multiple replicas of a Deployment
    Multiple Pods coexisting during rolling releases
    Querying logs across the entire application
    Dashboard panels
    Alarm rules

For production troubleshooting, it's recommended to start with:

    namespace + app

Then drill down to:

    pod + container

---

## XIII. Querying Logs on Nodes

### 13.1 View node label values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/node/values" | jq

### 13.2 Query logs on a specific node

LogQL:

    {node="k8s-worker01"}

Note:

    The query range may be large.
   {namespace=~".+"} |~ "(?i)error|exception|panic"Alloy was not collected.
Relabeling did not take effect.
Writing to Loki failed.
The query time range is incorrect.

### Step 19.5: Confirm label values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq

If app-demo does not exist:

    The collection scope may not include app-demo.
    There have been no logs for app-demo recently.
    The Alloy configuration may have filtered out that Namespace.

### Step 19.6: Expand the query scope

First, check a wider range:

    {namespace="app-demo"}

If results are found, then narrow it down to:

    {namespace="app-demo", app="nginx-demo"}

If the app label does not exist, you can start with pod:

    {namespace="app-demo", pod=~"nginx-demo-.*"}

### Step 19.7: Check the time range

If using the API for queries, specify the last hour explicitly:

    END=$(date +%s%N)
    START=$(date -d "1 hour ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' | jq

Also confirm the time range in Grafana Explore.

---

## Section 20: Common Troubleshooting Query Templates

### 20.1: Query the latest logs for a certain Namespace

    {namespace="app-demo"}

### 20.2: Query the latest logs for a specific application

    {namespace="app-demo", app="nginx-demo"}

### 20.3: Query the latest logs for a particular Pod

    {namespace="app-demo", pod="nginx-demo-xxxx"}

### 20.4: Query logs for a group of Pods within a Deployment

    {namespace="app-demo", pod=~"nginx-demo-.*"}

### 20.5: Query logs for a specific container

    {namespace="app-demo", pod=~"nginx-demo-.*", container="nginx"}

### 20.6: Query business logs on a certain node

    {namespace="app-demo", node="k8s-worker01"}

### 20.7: Query error logs

    {namespace="app-demo"} |~ "(?i)error|exception|panic|failed"

### 20.8: Query for timeout issues

    {namespace="app-demo"} |~ "(?i)timeout|timed out|deadline exceeded"

### 20.9: Query database connection problems

    {namespace="app-demo"} |~ "(?i)database|mysql|postgres|connection refused|too many connections"

### 20.10: Query for 5xx errors

For Nginx access logs:

    {namespace="app-demo", app="nginx-demo"} |~ " 5[0-9][0-9] "

Note:

    This is a simple regular expression based on the default Nginx log format.
    For structured logs, it is recommended to use | json in the next section.

### 20.11: Query for 404 errors

    {namespace="app-demo", app="nginx-demo"} |= "404"

### 20.12: Exclude health check requests

    {namespace="app-demo", app="nginx-demo"} != "/healthz" != "/metrics"

### 20.13: Query for ERROR logs but exclude known noise

    {namespace="app-prod", app="api"} |~ "(?i)error|exception" != "expected error" != "ignore"

### 20.14: Query for CUDA out of memory errors

    {namespace="ai-prod"} |~ "(?i)cuda out of memory|CUDA error|out of memory"

### 20.15: Query for Java exceptions

    {namespace="app-prod"} |~ "(?i)Exception|Caused by|NullPointerException|OutOfMemoryError"

### 20.16: Query for Python traceback errors

    {namespace="app-prod"} |~ "(?i)Traceback|ModuleNotFoundError|KeyError|Value```bash
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

Acceptance:

    [ ] kubectl logs should display the Nginx access logs
    [ ] Loki should record new logs

### 23.3 Task Three: Query by Namespace

Execution:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Logs for app-demo should be displayed

### 23.4 Task Four: Query by Pod

Execution:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", pod=~"nginx-demo-.*"}' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Logs for the nginx-demo Pod should be displayed

### 23.5 Task Five: Query by Container

Execution:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", container="nginx"}' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Logs for the nginx container should be displayed

### 23.6 Task Six: Query for 404 Errors

Execution:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "404"' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Logs related to /notfound should be displayed

### 23.7 Task Seven: Exclude Health Check Logs

Execution:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} != "/healthz" != "/metrics"' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] The query results should not include /healthz or /metrics

### 23.8 Task Eight: Query for the Last 10 Minutes

Execution:

    END=$(date +%s%N)
    START=$(date -d "10 minutes ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq

Acceptance:

    [ ] Logs from the last 10 minutes should be displayed
    [ ] The start/end values should be understood as nanosecond timestamps

---

## Twenty-Four, Acceptance Checklist

After completing this document, you should confirm that you understand:

    [ ] The basic structure of LogQL
    [ ] Stream selectors
    [ ] Line filters
    [ ] How to use = for precise Label matching
    [ ] How to use != to exclude Labels
    [ ] How to use =~ for regular expression matching of Labels
    [ ] How to use !~ for regular expression exclusion of Labels
    [ ] How to use |= to search for strings containing specific text
    [ ] How to use != to exclude strings
    [ ] How to use |~ for regular expression matching of log content
    [ ] How to use !~ for regular expression exclusion of log content
    [ ] How to query logs by namespace
    [ ] How to query logs by app
    [ ] How to query logs by pod
    [ ] How to query logs by container
    [ ] How to query logs by node
    [OpenSearch

---

## Chapter 27: Summary

This article has covered practical examples of basic LogQL queries.

The core components of basic LogQL include:

    Stream Selector
      +
    Line Filter

Common Label queries include:

    {namespace="app-demo"}

    {namespace="app-demo", app="nginx-demo"}

    {namespace="app-demo", pod=~"nginx-demo-.*"}

    {namespace="app-demo", container="nginx"}

Common content filters are:

    |= "ERROR"
    != "/healthz"
    |~ "(?i)error|exception|panic"
    !~ "healthz|metrics"

Recommended query approaches are:

    First, use Labels to narrow down the scope.
    Then apply string filters.
    Use regular expressions when necessary.
    Control the time range and the number of results returned.
    Avoid performing global searches.

Common troubleshooting queries in Kubernetes include:

    At the Pod level:
        {namespace="app-demo", pod=~"nginx-demo-.*"}

    At the Container level:
        {namespace="app-demo", container="nginx"}

    At the App level:
        {namespace="app-demo", app="nginx-demo"}

    For error logs:
        {namespace="app-demo"} |~ "(?i)error|exception|panic|timeout"

    To exclude health check results:
        {namespace="app-demo"} != "/healthz" != "/metrics"

In production, it is crucial to:

    Not start by searching over a large range.
    Not abuse regular expressions.
    Not rely on Pod names for long-term queries.
    Not treat Loki as an unlimited full-text search system.
    Ensure that queries are aligned with label design, time range, and business context.

The next chapter will cover:

    09-Advanced LogQL Queries: json-logfmt-regexp-unwrap

Key topics to learn include:

    JSON log parsing
    logfmt log parsing
    Extracting fields using regular expressions
    Formatting output with line_format
    Generating labels with label_format
    Unwrapping numerical fields
    Performing duration_ms queries
    Searching for status codes greater than or equal to 500
    Generating metrics from logs

---

## References

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Querying Loki:
  https://grafana.com/docs/loki/latest/query/

- Log Queries:
  https://grafana.com/docs/loki/latest/query/log_queries/

- LogQL Reference:
  https://grafana.com/docs/loki/latest/query/query_reference/

- Query Examples:
  https://grafana.com/docs/loki/latest/query/query_examples/

- Loki HTTP API:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- Getting Started with LogCLI:
  https://grafana.com/docs/loki/latest/query/logcli/getting-started/

- Understanding Labels:
  https://grafana.com/docs/loki/latest/get-started/labels/

- Best Practices for Queries:
  https://grafana.com/docs/loki/latest/query/bp-query/

- Grafana Explore:
  https://grafana.com/docs/grafana/latest/explore/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Using kubectl logs in Kubernetes:
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/