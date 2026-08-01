# 10-Grafana Integration with Loki and Log Dashboard Practical Guide

## Document Description

This article is the tenth in the Loki specialization series, used to systematically learn Grafana integration with Loki, Explore log queries, create Loki log Dashboard, configure variables, create log trend panels, error log panels, 5xx panels, slow request panels, and consolidate Loki log query capabilities into a production troubleshooting Dashboard.

Previously completed:

    01-Loki Basic Understanding and Experiment Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experiment Selection
    04-Loki Single-Instance Helm Deployment Practical Guide
    05-Loki Object Storage Integration with MinIO Practical Guide
    06-Grafana-Alloy Collection of K8S-Pod Logs Practical Guide
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practical Guide: Namespace-Pod-Container Log Retrieval
    09-LogQL Advanced Query Practical Guide: json-logfmt-regexp-unwrap

By the 08th article, you have mastered basic LogQL:

    {namespace="app-demo"}
    {namespace="app-demo", app="nginx-demo"}
    {namespace="app-demo"} |= "404"
    {namespace="app-demo"} |~ "(?i)error|exception|panic"

By the 09th article, you have mastered advanced LogQL:

    | json
    | logfmt
    | regexp
    | line_format
    | label_format
    | unwrap
    count_over_time
    rate
    avg_over_time
    quantile_over_time

This article begins to place these query capabilities into Grafana to form a visualization troubleshooting entry point.

This article focuses on answering the following questions:

- How to deploy or confirm Grafana availability;
- How to add a Loki Data Source in Grafana;
- How to use the UI to add a Loki data source;
- How to use the provisioning method to declare a Loki data source;
- How to query Loki logs in Explore;
- How to design Namespace / App / Pod / Container / Node variables;
- How to create a Pod log details panel;
- How to create an ERROR log trend panel;
- How to create a 5xx log trend panel;
- How to create a slow request P95 panel;
- How to create a Top Error Pod panel;
- How to create a recent error log panel;
- How to use a Dashboard to support Kubernetes Pod troubleshooting;
- How to link Prometheus metrics troubleshooting to Loki logs;
- What are theAttention for Dashboard query performance;
- How to govern permissions, variables, query ranges, and alert jump in production environments with Grafana + Loki.

---

## Tags

#Grafana #Loki #LogQL #Dashboard #Explore #Kubernetes #PodLog #LogVisualization #SRE #Observation #LogDetachment #GrafanaVariable

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/10-Grafana Integration with Loki and Log Dashboard Practical Guide.md

---

## One. Experiment Objectives

After completing this article, you should be able to:

    1. Confirm that Grafana has been deployed and is accessible.
    2. Add a Loki Data Source in Grafana.
    3. Use Grafana Explore to query Loki logs.
    4. Create a Loki log Dashboard.
    5. Create a cluster variable.
    6. Create a namespace variable.
    7. Create an app variable.
    8. Create a pod variable.
    9. Create a container variable.
    10. Create a node variable.
    11. Create a Pod log details panel.
    12. Create an ERROR log trend panel.
    13. Create a 5xx log trend panel.
    14. Create a slow request P95 panel.
    15. Create a Top Error Pod panel.
    16. Create a recent error log panel.
    17. Understand the difference between Grafana Explore and Dashboard.
    18. Understand the difference between log panels and metric panels.
    19. Understand how to link Prometheus metrics to Loki logs.
    20. Master the production design principles of Grafana + Loki Dashboard.

---

## Two. Experiment Environment

### 2.1 Kubernetes Cluster

Experiment nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespaces:

    logging
    monitoring
    app-demo
    minio

Deployed components:

    Loki
    Loki Gateway
    Grafana Alloy
    MinIO
    Grafana
    nginx-demo
    json-log-demo
    logfmt-log-demo

### 2.2 Prerequisites

Need to confirm:

    [ ] Loki Pod Running
    [ ] Loki Gateway Service is normal
    [ ] Alloy DaemonSet is normal
    [ ] app-demo logs have entered Loki
    [ ] Grafana has been deployed or is ready to deploy
    [ ] Can query app-demo logs via LogQL
    [ ] Can query logs via query_range API

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Port forward Loki:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

Check logs: /think

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

---

## Three. Grafana's Position in the Loki Ecosystem

### 3.1 What Grafana is Responsible For

Grafana is responsible for:

    Data source management
    Explore temporary queries
    Dashboard visualization
    Variable filtering
    Panel display
    Alert rule configuration
    Data linkage
    Permission control

In the Loki context, Grafana is primarily used for:

    Querying Pod logs
    Displaying log trends
    Displaying error logs
    Displaying 5xx trends
    Displaying slow requests
    Filtering logs by namespace/app/pod/container
    Jumping from metric panels to logs
    Serving as a troubleshooting entry point

### 3.2 What Grafana is Not Responsible For

Grafana is not responsible for:

    Log collection
    Log storage
    Log compression
    Log cleanup
    Controlling Loki writes
    Replacing Alloy
    Replacing Loki
    Replacing object storage

The log pipeline should be understood as:

    Pod stdout/stderr
      ↓
    Alloy
      ↓
    Loki
      ↓
    Grafana

Grafana is a query and display entry point, not a log storage backend.

---

## Four. Differences Between Explore and Dashboard

### 4.1 Explore

Explore is suitable for:

    Temporary troubleshooting
    Debugging LogQL
    Quickly viewing a specific Pod's logs
    Validating query statements
    Troubleshooting recently occurred issues
    Temporarily deep analysis after entering from an alert

Examples:

    {namespace="app-demo", app="nginx-demo"} |= "404"

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000

### 4.2 Dashboard

Dashboard is suitable for:

    Solidifying common queries
    Forming standardized troubleshooting entry points
    Unified use by the team
    Combining variable filtering
    Displaying trends
    Displaying TopN
    Supporting on-call and post-mortem analysis
    Linking with metric panels

Dashboard should avoid becoming:

    A collection of ad-hoc queries
    No variables
    No range control
    No title
    No explanation
    Wide queries
    Many panels but unable to locate issues

### 4.3 Recommended Usage

Recommended workflow:

    1. First debug LogQL in Explore.
    2. Confirm the query accuracy.
    3. Save it to a Dashboard panel.
    4. Add variables.
    5. Add panel explanations.
    6. Add links and Runbook.
    7. Form a production troubleshooting entry point.

---

## Five. Confirming or Deploying Grafana

If you already have Grafana, you can skip the deployment section.

### 5.1 Check the monitoring Namespace

    kubectl get ns monitoring

If it doesn't exist:

    kubectl create namespace monitoring

### 5.2 Add Helm Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 5.3 Query Grafana Chart

    helm search repo grafana/grafana --versions | head -20

Record the version:

    GRAFANA_CHART_VERSION=<actual version queried>

### 5.4 Create Grafana values

Create a file:

    values-grafana.yaml

Content example:

    adminUser: admin
    adminPassword: admin123

    service:
      type: ClusterIP

    persistence:
      enabled: true
      size: 5Gi

    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

Notes:

    Learning environments can use simple passwords.
    Production environments must use strong passwords or integrate with centralized authentication.
    Do not commit production passwords in plain text to Git.

### 5.5 Install Grafana

    helm install grafana grafana/grafana \
      -n monitoring \
      --version <GRAFANA_CHART_VERSION> \
      -f values-grafana.yaml

### 5.6 Check Grafana Status

    helm list -n monitoring

    kubectl get pods -n monitoring -o wide

    kubectl get svc -n monitoring

    kubectl get pvc -n monitoring

### 5.7 Access Grafana

Port forwarding:

    kubectl port-forward svc/grafana 3000:80 -n monitoring

Access:

    http://127.0.0.1:3000

Login:

    Username:
        admin

    Password:
        admin123

If using Helm-generated password, run:

    kubectl get secret --namespace monitoring grafana \
      -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

---

## Six. Adding Loki Data Source to Grafana: UI Method

### 6.1 Enter the Data Source Page

Path: /think

# Left Sidebar Menu
  ↓
Connections
  ↓
Data sources
  ↓
Add data source
  ↓
Select Loki

The UI menu may vary slightly across different Grafana versions.

Core objective:

  Add Loki Data Source

### 6.2 Configure Loki URL

If both Grafana and Loki are within a Kubernetes cluster, it's recommended to use the cluster's Service address.

Loki Gateway address:

  http://loki-gateway.logging.svc.cluster.local

If there's no Gateway, use Loki Service:

  http://loki.logging.svc.cluster.local:3100

This article recommends:

  http://loki-gateway.logging.svc.cluster.local

### 6.3 Save and Test

Click:

  Save & test

Expected outcome:

  Data source connected and labels found

Or similar success message.

If failed, check:

  Whether Loki Gateway Service exists.
  Whether Loki Gateway has an Endpoint.
  Whether Loki /ready is ready.
  Whether Grafana Pod can access Loki Service.
  Whether URL is written correctly.
  Whether Namespace is written correctly.
  Whether port is written correctly.

### 6.4 Test Loki from Grafana Pod

Enter Grafana Pod:

  kubectl get pod -n monitoring | grep grafana

  kubectl exec -it <grafana-pod-name> -n monitoring -- sh

Execute inside container:

  wget -qO- http://loki-gateway.logging.svc.cluster.local/ready

Or:

  curl -s http://loki-gateway.logging.svc.cluster.local/ready

If curl/wget isn't available in the image, use a temporary Pod for testing.

---

## VII. Grafana Add Loki Data Source: Provisioning Method

### 7.1 Why Need Provisioning

UI addition is suitable for learning.

Production recommends managing data sources via provisioning.

Benefits:

  Versioned
  Git-managed
  Repeatable deployment
  Avoid manual configuration drift
  Consistent across environments

### 7.2 Add datasources to values-grafana

You can add in Grafana Helm values:

  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Loki
          type: loki
          access: proxy
          url: http://loki-gateway.logging.svc.cluster.local
          isDefault: false
          editable: true
          jsonData:
            maxLines: 1000

Explanation:

  name:
      The name displayed in Grafana for the data source.

  type:
      loki.

  access:
      proxy indicates Grafana backend accesses Loki.

  url:
      Cluster's Loki Gateway address.

  maxLines:
      Configuration related to maximum displayed lines in Explore queries.

### 7.3 Upgrade Grafana

After saving values-grafana.yaml, execute:

  helm upgrade grafana grafana/grafana \
    -n monitoring \
    --version <GRAFANA_CHART_VERSION> \
    -f values-grafana.yaml

Check:

  kubectl rollout status deploy/grafana -n monitoring

Re-enter Grafana:

  Connections
    ↓
  Data sources
    ↓
  Loki

Confirm the data source exists.

### 7.4 Production Recommendations

Production recommends:

  Data Source provisioning
  Dashboard provisioning
  GitOps management
  Environment-specific values
  Avoid long-term manual maintenance of core configurations in UI

---

## VIII. Grafana Explore Query Loki

### 8.1 Enter Explore

Path:

  Left sidebar menu
    ↓
  Explore
    ↓
  Select Loki data source

### 8.2 Basic Queries

Query app-demo:

  {namespace="app-demo"}

Query nginx-demo:

  {namespace="app-demo", app="nginx-demo"}

Query a group of Pods:

  {namespace="app-demo", pod=~"nginx-demo-.*"}

Query 404:

  {namespace="app-demo", app="nginx-demo"} |= "404"

Query error logs:

  {namespace="app-demo"} |~ "(?i)error|exception|panic|timeout|failed"

### 8.3 JSON Log Queries

Query json-log-demo error:

  {namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"

Query 5xx:

  {namespace="app-demo", app="json-log-demo"} | json | __error__="" | status >= 500

Query slow requests:

  {namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000

Format display: /think

```json
{
  "namespace": "app-demo",
  "app": "json-log-demo"
}
```

| json
| __error__=""
| level="error"
| line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 8.4 Time Range

Select from the top-right dropdown:

    Last 5 minutes
    Last 15 minutes
    Last 1 hour
    Last 6 hours

Recommendations:

    Daily troubleshooting should prioritize Last 15 minutes.
    Fault analysis should select based on the fault window.
    Avoid querying Last 7 days initially.

### 8.5 View Log Details

Expand a log entry in Explore to see:

    Timestamp
    Labels
    Fields
    Parsed fields
    Structured metadata

Troubleshooting focus:

    Is namespace correct?
    Is app correct?
    Is pod correct?
    Is container correct?
    Is node correct?
    Is team correct?
    Are parsed fields present?
    Is __error__ appearing?

---

## Nine. Create Loki Dashboard

### 9.1 New Dashboard

Path:

    Dashboards
      ↓
    New
      ↓
    New dashboard

Recommended name:

    Kubernetes / Loki Pod Logs Overview

or:

    K8S Loki Log Troubleshooting Overview

### 9.2 Dashboard Purpose

This dashboard is not for fancy visualization, but to support troubleshooting.

Core questions:

    Which namespace has the most logs?
    Which app has the most errors?
    Which pod has the most errors?
    What are the recent error logs?
    Are there 5xx errors?
    Are there timeout errors?
    Are there slow requests?
    Can we quickly drill down to a specific pod/container?
    Can we combine with Prometheus metrics for troubleshooting?

### 9.3 Suggested Panel Partitioning

Recommended partitioning:

    First row:
        Query variables and explanations

    Second row:
        Total logs, error logs, 5xx logs, timeout logs

    Third row:
        Log volume trend, ERROR trend, 5xx trend, slow request trend

    Fourth row:
        Top Error App, Top Error Pod, Top 5xx Path

    Fifth row:
        Recent error logs, Pod log details

    Sixth row:
        Runbook links, common LogQL explanations

---

## Ten. Design Dashboard Variables

### 10.1 Variable Design Principles

Variables should help users narrow down scope step by step:

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
      ↓
    node

Don't let users query globally at first:

    {namespace=~".+"}

### 10.2 cluster Variable

Type:

    Query

Data source:

    Loki

Query:

    label_values(cluster)

Name:

    cluster

Multi-value:

    Optional

Include All option:

    Optional

Note:

    If the learning environment has only one cluster, fix it to k8s-lab initially.

### 10.3 namespace Variable

Type:

    Query

Data source:

    Loki

Query:

    label_values({cluster="$cluster"}, namespace)

Name:

    namespace

Multi-value:

    Optional

Include All option:

    Optional

If no cluster label:

    label_values(namespace)

### 10.4 app Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, app)

If no cluster:

    label_values({namespace="$namespace"}, app)

Name:

    app

### 10.5 pod Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace", app="$app"}, pod)

If app is empty or some logs lack app:

    label_values({namespace="$namespace"}, pod)

Name:

    pod

### 10.6 container Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace", pod="$pod"}, container)

Name:

    container

### 10.7 node Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, node)

Name:

    node

### 10.8 team Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, team)

Name:

    team

### 10.9 Variable Usage Notes

If Multi-value or All is enabled:

    Use regex matching in LogQL:

        namespace=~"$namespace"
        app=~"$app"
        pod=~"$pod"

If Multi-value is not enabled:

    Use exact matching:

        namespace="$namespace"
        app="$app"

Production Dashboard recommendation: /think

Important variables with Multi-value should be used cautiously.
All queries may cover a large range.
If enabling All, restrict time range and panel count.

---

## Eleven, Panel One: Log Volume Trend

### 11.1 Panel Objective

Display changes in log volume within the selected range.

Purpose:

    Observe sudden log spikes.
    Determine if an app is spamming logs.
    Detect abnormal log peaks.
    Serve as an entry point for log cost governance.

### 11.2 Panel Type

Panel type:

    Time series

Data source:

    Loki

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}[$__interval]
      )
    )

If no cluster variable:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}[$__interval]
      )
    )

### 11.3 Title

    Log Volume Trend by app

### 11.4 Unit

Unit:

    logs

Or:

    none

### 11.5 Notes

Logs without an app label may not be displayed.

Should first check:

    label_values(app)

Production recommendation: All business Pods should have the app label set.

---

## Twelve, Panel Two: ERROR Log Trend

### 12.1 Panel Objective

Display trends of error-type logs like ERROR / exception / panic / failed.

### 12.2 Panel Type

Panel type:

    Time series

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ "(?i)error|exception|panic|failed" [$__interval]
      )
    )

No cluster version:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          |~ "(?i)error|exception|panic|failed" [$__interval]
      )
    )

### 12.3 Title

    ERROR Log Trend by app

### 12.4 Production Recommendations

The ERROR keyword does not necessarily mean a real error.

Should combine with:

    Log standards
    level field
    error_type field
    Application semantics

If the app outputs JSON, it's recommended to use:

    level="error"

---

## Thirteen, Panel Three: JSON ERROR Log Trend

### 13.1 Applicable Scenarios

Suitable for structured JSON logs.

Example: json-log-demo.

### 13.2 Panel Type

Time series

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          | json
          | __error__=""
          | level="error" [$__interval]
      )
    )

No cluster:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          | json
          | __error__=""
          | level="error" [$__interval]
      )
    )

### 13.3 Title

    JSON ERROR Log Trend

### 13.4 Notes

If the selected app is not a JSON log, parsing errors will occur.

Already using:

    __error__=""

To filter parsing failures.

In production, it's recommended to create separate panels for JSON and non-JSON apps.

---

## Fourteen, Panel Four: 5xx Log Trend

### 14.1 Panel Objective

Display trends of HTTP 5xx errors.

Suitable for:

    API services
    Nginx access logs
    Ingress logs
    JSON business access logs

### 14.2 JSON Method

Panel type:

    Time series

Query:

    sum by (app, status) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          | json
          | __error__=""
          | status >= 500 [$__interval]
      )
    )

No cluster:

    sum by (app, status) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          | json
          | __error__=""
          | status >= 500 [$__interval]
      )
    )

### 14.3 Nginx Text Log Method

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ " 5[0-9][0-9] " [$__interval]
      )
    )

No cluster:

```
sum by (app) (
  count_over_time(
    {namespace=~"$namespace", app=~"$app"}
      |~ " 5[0-9][0-9] " [$__interval]
  )
)

### 14.4 Title

  5xx Log Trends

### 14.5 Note

Text-based approach depends on log format.

If Nginx log format changes, the regex may fail.

In production, it's recommended to use applications outputting structured JSON fields:

    status: 500

---

## FifteenI don't know.Panel Five: Timeout Log Trends

### 15.1 Panel Objective

Display timeout, deadline exceeded, timed out, and other timeout logs.

### 15.2 Query

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ "(?i)timeout|timed out|deadline exceeded" [$__interval]
      )
    )

Without cluster:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          |~ "(?i)timeout|timed out|deadline exceeded" [$__interval]
      )
    )

### 15.3 Title

    Timeout Log Trends

### 15.4 Suitable Scenarios

    Database access timeout
    Redis timeout
    HTTP downstream timeout
    gRPC deadline exceeded
    AI inference timeout
    Object storage access timeout

---

## SixteenI don't know.Panel Six: Slow Request P95

### 16.1 Panel Objective

Calculate P95 from the duration_ms field in JSON logs.

Suitable for:

    API request duration
    Inference request duration
    Batch processing task duration
    Database operation duration

### 16.2 Query

    quantile_over_time(
      0.95,
      {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

Without cluster:

    quantile_over_time(
      0.95,
      {namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

### 16.3 Title

    P95 Request Duration duration_ms

### 16.4 Unit

Unit:

    milliseconds

### 16.5 Note

Requires the log to have a numeric field:

    duration_ms

If the field is a string or missing, unwrap may fail.

Must add:

    __error__=""

---

## SeventeenI don't know.Panel Seven: Average Request Duration

### 17.1 Query

    avg_over_time(
      {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

Without cluster:

    avg_over_time(
      {namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

### 17.2 Title

    Average Request Duration duration_ms

### 17.3 Unit

    milliseconds

### 17.4 Usage Recommendation

Average values are easily influenced by extreme values and may mask long-tail issues.

In production, it's recommended to also check:

    Average value
    P95
    P99
    Maximum value

---

## EighteenI don't know.Panel Eight: Top Error Pod

### 18.1 Panel Objective

Display the Pod with the most error logs in recent time.

### 18.2 Panel Type

Panel type:

    Bar chart
    Table
    Time series

Recommended:

    Table or Bar chart

### 18.3 Query

    topk(10,
      sum by (pod) (
        count_over_time(
          {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
            |~ "(?i)error|exception|panic|failed" [$__range]
        )
      )
    )

Without cluster:

    topk(10,
      sum by (pod) (
        count_over_time(
          {namespace=~"$namespace", app=~"$app"}
            |~ "(?i)error|exception|panic|failed" [$__range]
        )
      )
    )

### 18.4 Title

    Top 10 Error Pod

### 18.5 Purpose

Quickly identify:

    Which Pod has the most errors
    Whether only a single replica is abnormal
    Whether new Pods are abnormal after rolling release
    Whether Pods on a certain node areFocus. reporting errors

---

## NineteenI don't know.Panel Nine: Top Error App

### 19.1 Query
```

topk(10,
  sum by (app) (
    count_over_time(
      {cluster=~"$cluster", namespace=~"$namespace"}
        |~ "(?i)error|exception|panic|failed" [$__range]
    )
  )
)

Without cluster:

  topk(10,
    sum by (app) (
      count_over_time(
        {namespace=~"$namespace"}
          |~ "(?i)error|exception|panic|failed" [$__range]
      )
    )
  )

### 19.2 Title

  Top 10 Error App

### 19.3 Purpose

Suitable for platform perspective quick discovery:

  Which app has the most errors
  Which team needs to handle it
  Which namespace has the most anomalies

---

## Twenty, Panel Ten: Recent Error Logs

### 20.1 Panel Goal

Display detailed information about recent error logs.

### 20.2 Panel Type

Panel type:

  Logs

### 20.3 Query

  {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
    |~ "(?i)error|exception|panic|failed|timeout"

Without cluster:

  {namespace=~"$namespace", app=~"$app"}
    |~ "(?i)error|exception|panic|failed|timeout"

### 20.4 Title

  Recent Error Logs

### 20.5 Option Suggestions

Recommended configuration:

  Show time:
      true

  Enable log details:
      true

  Deduplication:
      none or exact

  Prettify JSON:
      true

  Limit:
      100 or 200

### 20.6 Note

This panel is a logs detail panel.

Do not set the time range too large.

Recommended Dashboard default time:

  Last 15 minutes
  Last 1 hour

---

## Twenty-one, Panel Eleven: Pod Log Details

### 21.1 Panel Goal

After selecting a specific Pod and Container via variables, display the logs for that Pod.

### 21.2 Panel type

  Logs

### 21.3 Query

If variables support multiple selections:

  {cluster=~"$cluster", namespace=~"$namespace", pod=~"$pod", container=~"$container"}

Without cluster:

  {namespace=~"$namespace", pod=~"$pod", container=~"$container"}

If variables do not support multiple selections:

  {namespace="$namespace", pod="$pod", container="$container"}

### 21.4 Title

  Pod Log Details

### 21.5 Purpose

Suitable for:

  Pod CrashLoopBackOff troubleshooting
  Single Pod log tracking
  Multi-container Pod investigation
  Sidecar issue troubleshooting

### 21.6 Recommended Pairing Notes

Write in the panel description:

  Used to view the original logs for a specified Pod / Container.
  If the Pod just restarted, it's still recommended to use kubectl logs --previous to view the previous container logs.
  Whether Loki retains previous logs depends on the collection and writing timing.

---

## Twenty-two, Panel Twelve: JSON-formatted Error Logs

### 22.1 Goal

Format JSON logs into more readable troubleshooting content.

### 22.2 Panel type

  Logs

### 22.3 Query

  {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
    | json
    | __error__=""
    | level="error"
    | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

Without cluster:

  {namespace=~"$namespace", app=~"$app"}
    | json
    | __error__=""
    | level="error"
    | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 22.4 Title

  JSON-formatted Error Logs View

### 22.5 Note

Only applicable for JSON logs.

If the app is not JSON logs, this panel may be empty.

---

## Twenty-three, Panel Thirteen: Log Count by Status

### 23.1 Goal

Count logs by status code.

### 23.2 Panel type

  Bar chart
  Table
  Time series

### 23.3 Query

  sum by (status) (
    count_over_time(
      {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
        | json
        | __error__="" [$__range]
    )
  )

Without cluster:

```
sum by (status) (
  count_over_time(
    {namespace=~"$namespace", app=~"$app"}
      | json
      | __error__="" [$__range]
  )
)

### 23.4 Title

  Status Code Distribution

### 23.5 Note

  Requires the log to contain a "status" field.

---

## Twenty-Four, Panel Fourteen: Statistics of 5xx by Path

### 24.1 Objective

  Identify which path has the most 5xx errors.

### 24.2 Query

    topk(10,
      sum by (path) (
        count_over_time(
          {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
            | json
            | __error__=""
            | status >= 500 [$__range]
        )
      )
    )

  No cluster:

    topk(10,
      sum by (path) (
        count_over_time(
          {namespace=~"$namespace", app=~"$app"}
            | json
            | __error__=""
            | status >= 500 [$__range]
        )
      )
    )

### 24.3 Title

  Top 5xx Path

### 24.4 Note

  Whether path is suitable as a grouping field depends on the specification.

  If path contains dynamic IDs:

    /api/order/100001
    /api/order/100002

  This will cause excessive grouping.

  In production, it's recommended to use:

    route="/api/order/:id"

  Or:

    path_template="/api/order/{id}"

---

## Twenty-Five, Panel Fifteen: Loki Query Error Check

### 25.1 Objective

  Display the count of JSON parsing failure logs to help identify issues with log formatting.

### 25.2 Query

    sum by (app, __error__) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          | json
          | __error__!="" [$__range]
      )
    )

  No cluster:

    sum by (app, __error__) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          | json
          | __error__!="" [$__range]
      )
    )

### 25.3 Title

  JSON Parsing Error Statistics

### 25.4 Usage

  Used to identify:

    Some logs are not JSON
    Application log format is inconsistent
    Field types are not uniform
    Dashboard query is unstable
    Alert rules may misfire

---

## Twenty-Six, Dashboard Layout Recommendations

### 26.1 First Row: Overview

  Panels:

    Log Volume Trend by App
    ERROR Log Trend
    5xx Log Trend
    Timeout Log Trend

### 26.2 Second Row: Request Quality

  Panels:

    P95 Request Latency
    Average Request Latency
    Status Code Distribution
    Top 5xx Path

### 26.3 Third Row: Problem Localization

  Panels:

    Top 10 Error App
    Top 10 Error Pod
    JSON Parsing Error Statistics

### 26.4 Fourth Row: Log Details

  Panels:

    Recent Error Logs
    Pod Log Details
    JSON Error Log Formatted View

### 26.5 Fifth Row: Notes

  Panels:

    Common LogQL
    Runbook Links
    Label Specification Notes
    Querying Notes

---

## Twenty-Seven, Dashboard Common Notes

  It is recommended to add a Text panel with the following content:

    Usage Instructions:

    1. Prioritize selecting cluster, namespace, and app before viewing logs.
    2. It is not recommended to query full namespace in production environments.
    3. ERROR keyword queries should only be used as auxiliary judgment, prefer using level="error" for structured logs.
    4. 5xx panels depend on the presence of a "status" field in logs.
    5. P95 latency panels depend on the presence of a "duration_ms" numeric field in logs.
    6. Pod log details are suitable for temporary troubleshooting, long-term observation should use app dimension.
    7. If the query returns empty results, first check the time range, variable values, and Loki labels.
    8. If sensitive information appears in logs, prioritize fixing the application's log output.

---

## Twenty-Eight, Linking Prometheus Metrics with Loki Logs

### 28.1 Why Integration is Needed

  Production troubleshooting typically starts with metrics rather than logs:

    Pod restarts
    High CPU
    High memory
    5xx increase
    P95 latency increase
    High GPU memory
    OOMKilled

  Then jump to logs to identify the root cause.

  Recommended workflow:

    Prometheus Alert
      ↓
    Grafana Metrics Panel
      ↓
    Loki Log Panel
      ↓
    kubectl describe / Events
      ↓
    Runbook Handling

### 28.2 Using Variables in Prometheus Panels

  If Prometheus and Loki use consistent labels:

    namespace
    app
    pod
    container

  They can share variables.

  Prometheus Query Example:

    sum by (pod) (
      rate(container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod", container!="", image!=""}[5m])
    )

  Corresponding Loki Query:

    {namespace="$namespace", pod=~"$pod"} /think
```

### 28.3 Data Link Concept

In the Prometheus Pod panel, you can configure Data link to jump to the Loki Dashboard.

Passed variables:

    namespace
    pod
    container
    from
    to

Example concept:

    Click a pod in the Pod CPU panel
      ↓
    Jump to the Loki Logs Dashboard
      ↓
    Automatically carry namespace, pod, container
      ↓
    View the Pod logs

### 28.4 Production Requirements

To achieve good integration, you need:

    Unified metric and log labels
    Consistent namespace
    Consistent pod
    Consistent container
    Consistent app
    Unified Grafana variable naming
    Stable Dashboard UID

---

## Twenty-Nine, Derived Fields and Trace Integration

### 29.1 What are Derived Fields

The Grafana Loki data source can configure derived fields to extract fields from logs and generate jump links.

Common scenarios:

    Jump to Tempo Trace from trace_id in logs.
    Jump to internal query system from request_id in logs.
    Jump to business troubleshooting page from order_id in logs.

### 29.2 trace_id Example

If logs contain:

    trace_id=abc123

Or JSON field:

    "trace_id":"abc123"

You can configure Derived Field:

    Name:
        trace_id

    Regex:
        trace_id["=:\s]+([a-zA-Z0-9-]+)

    URL / Internal link:
        Point to Tempo or other system

### 29.3 Note

trace_id should not be used as a Loki Label.

Correct approach:

    Place trace_id in log body or structured field.
    Grafana extracts it via derived field.
    Click to jump to Trace system.

### 29.4 Series Explanation

This article only discusses Loki Dashboard.

Tempo trace linkage is not expanded in this article.

But establish awareness:

    Logs + Metrics + Traces integration is a key direction for production observability.

---

## Thirty, Dashboard Export and Import

### 30.1 Export Dashboard

In the Grafana Dashboard page:

    Settings
      ↓
    JSON model
      ↓
    Copy JSON

Or:

    Share
      ↓
    Export

Recommended save name:

    k8s-loki-logs-overview.json

### 30.2 Import Dashboard

Path:

    Dashboards
      ↓
    New
      ↓
    Import
      ↓
    Upload JSON file

### 30.3 Production Recommendations

Dashboard should be Git managed.

Recommended directory:

    observability/grafana/dashboards/loki/k8s-loki-logs-overview.json

Or knowledge base path:

    10-Logs/02-Loki/dashboard/k8s-loki-logs-overview.json

### 30.4 Dashboard Change Guidelines

Recommendations:

    Export backup before modification.
    Record change notes after modification.
    Avoid long-term manual edits on production Grafana.
    Sync via Dashboard provisioning or GitOps.

---

## Thirty-One, Dashboard Provisioning Concept

### 31.1 Why Need Dashboard Provisioning

UI creation is suitable for learning.

Production recommends:

    Dashboard as Code

Benefits:

    Version management
    Rollback capability
    Consistency across environments
    Avoid manual drift
    Facilitate audit

### 31.2 Grafana Helm values Example Concept

You can configure dashboardProviders and dashboardsConfigMaps in Grafana Helm values.

Example concept:

    dashboardProviders:
      dashboardproviders.yaml:
        apiVersion: 1
        providers:
          - name: loki-dashboards
            orgId: 1
            folder: Loki
            type: file
            disableDeletion: false
            editable: true
            options:
              path: /var/lib/grafana/dashboards/loki

    dashboardsConfigMaps:
      loki-dashboards: grafana-loki-dashboards

Then place Dashboard JSON in ConfigMap.

Note:

    Specific fields follow current Grafana Helm Chart values.
    In production, confirm fields via helm show values first.

### 31.3 Create ConfigMap Example

Assume Dashboard JSON file is:

    k8s-loki-logs-overview.json

Create ConfigMap:

    kubectl create configmap grafana-loki-dashboards \
      -n monitoring \
      --from-file=k8s-loki-logs-overview.json

Then upgrade Grafana:

    helm upgrade grafana grafana/grafana \
      -n monitoring \
      --version <GRAFANA_CHART_VERSION> \
      -f values-grafana.yaml

### 31.4 Note

Do not hardcode environment variables in Dashboard JSON.

Recommend using variables: /think

cluster  
namespace  
app  
pod  
container  

Avoid duplicating a Dashboard for each environment.

---

## 32. Grafana + Loki Common Issues Troubleshooting

### 32.1 Data Source Connection Failed

Symptoms:

    Save & test failed  
    Unable to connect with Loki  
    Bad Gateway  
    404  
    502  
    503  

Troubleshooting:

    1. Does the Loki Gateway Service exist.  
    2. Is the Loki Gateway Endpoint empty.  
    3. Can the Grafana Pod access the Loki Gateway.  
    4. Is the URL correct.  
    5. Did you use the wrong port.  
    6. Is the Loki /ready endpoint normal.  
    7. Is the NetworkPolicy blocking.  
    8. Is DNS resolution normal.  

Commands:

    kubectl get svc -n logging  

    kubectl get endpoints -n logging  

    kubectl get pod -n monitoring | grep grafana  

    kubectl exec -it <grafana-pod-name> -n monitoring -- sh  

Inside the container:

    wget -qO- http://loki-gateway.logging.svc.cluster.local/ready  

### 32.2 Explore Query Returns Empty Results

Troubleshooting:

    1. Is the time range correct.  
    2. Do Loki labels exist.  
    3. Are the namespace/app/pod variables populated.  
    4. Is the query statement correct.  
    5. Has Alloy collected logs.  
    6. Did Loki write successfully.  
    7. Did you select the wrong data source.  

First, directly query in Explore:

    {namespace="app-demo"}  

Then narrow down step by step:

    {namespace="app-demo", app="nginx-demo"}  

### 32.3 Dashboard Variables Are Empty

Troubleshooting:

    1. Is the variable data source selected as Loki.  
    2. Did you write the query incorrectly.  
    3. Is the label_values query correct.  
    4. Is there no data in the current time range.  
    5. Does Loki have this label.  
    6. Is the variable dependency order correct.  

Check labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq  

Check app:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq  

### 32.4 Panel Shows Parse Error

Common causes:

    LogQL syntax error.  
    Syntax error after variable expansion.  
    Multi-value variables used = instead of =~.  
    Missing quotes.  
    Regex escaping error.  
    json / unwrap position error.  

Example:

    Error:  
        {namespace="$namespace"}  

    May fail or return no results when namespace is multi-selected.  

    Recommended:  
        {namespace=~"$namespace"}  

### 32.5 JSON Panel Has No Data

Possible causes:

    Application logs are not JSON.  
    Parsing failed.  
    __error__ is filtered out.  
    status field does not exist.  
    duration_ms field is not a number.  
    app variable selected incorrectly.  

First check raw logs:

    {namespace="$namespace", app="$app"}  

Then check parsing errors:

    {namespace="$namespace", app="$app"} | json | __error__!=""

### 32.6 Dashboard Query Is Slow

Causes:

    Query range is too large.  
    Variables selected All.  
    Too many panels.  
    Regex is too broad.  
    Used large json/regexp parsing.  
    $__range is too large.  
    Loki resources are insufficient.  
    Object storage read is slow.  

Optimization:

    Reduce default time range.  
    Avoid default All.  
    Reduce number of panels.  
    Use label to narrow range first.  
    Prefer structured logs.  
    Consider recording rule for large queries.  
    Plan caching and rate limiting for hot panels.  

---

## 33. Production Dashboard Design Principles

### 33.1 Do Not Default to Global Queries

Not recommended for Dashboard defaults:

    namespace=All  
    app=All  
    Time range Last 7 days  

Recommended:

    namespace is required  
    app is required  
    Time range Last 15 minutes or Last 1 hour  

### 33.2 Dashboard Should Have Clear Purpose

Separate different Dashboards:

    K8S Loki Log Overview  
    Application Log Troubleshooting  
    Pod Log Details  
    Ingress Access Logs  
    Error Log Analysis  
    Slow Request Analysis  
    Loki Self-Monitoring  
    Alloy Collection Monitoring  

Do not pile all content into one Dashboard.

### 33.3 Panels Should Have Descriptions

Each key panel should have a description:

    Query logic  
    Dependent fields  
    Applicable scenarios  
    How to troubleshoot empty results  

Example P95 panel description:

    This panel depends on the duration_ms field in JSON logs.  
    If the application does not output this field, the panel has no data.  
    If duration_ms is not a number, unwrap will fail.  

### 33.4 Label Specifications Must Be Unified

Dashboard reusability depends on label specifications.

Must unify:

    cluster  
    environment  
    namespace  
    app  
    pod  
    container  
    node  
    team  

If label specifications are inconsistent across applications, Dashboard reusability is difficult.

### 33.5 Log Fields Must Be Unified

Recommended unified structured log fields:

    level  
    service  
    trace_id  
    method  
    path  
    route  
    status  
    duration_ms  
    msg  
    error_type  

Do not have some services call:

    duration  

Others call:

    cost  

Others call:

    elapsed_time  

This would cause Dashboard and alerting to be inconsistent.

### 33.6 Queries Should Consider Loki Costs

Dashboard is the entry point for continuous queries.

If each panel queries a large range of logs, it will consume Loki over a long period.

Must pay attention to:

    Default time range
    Number of panels
    Query complexity
    Default values of variables
    Whether to use regexp
    Whether to use JSON parsing for full logs
    Whether to use topk for wide-range queries

---

## 34. Relationship Between Dashboard and Alerts

### 34.1 Dashboard is Not an Alert

Dashboard is used for observation and troubleshooting.

Alerts are used for active notifications.

You cannot only create Dashboard without creating alerts.

### 34.2 Dashboard Queries Can Be Evolved into Alerts

For example, an ERROR log trend query:

    sum by (app) (
      count_over_time(
        {namespace="app-prod", app="order-api"}
          | json
          | __error__=""
          | level="error" [5m]
      )
    )

Can evolve into a Loki Ruler alert:

    Alert when the number of error logs in the last 5 minutes exceeds the threshold.

### 34.3 Next Content

The next content will enter:

    11-Loki Log Alerting in Practice: Ruler and AlertManager Integration

Focus:

    Convert Dashboard queries into alert rules.
    ERROR log alert.
    5xx log alert.
    timeout log alert.
    CUDA OOM log alert.
    AlertManager grouping and routing.

---

## 35. Practical Tasks

### 35.1 Task One: Confirm Grafana is Accessible

Execute:

    kubectl get pods -n monitoring -o wide

    kubectl get svc -n monitoring

    kubectl port-forward svc/grafana 3000:80 -n monitoring

Access:

    http://127.0.0.1:3000

Verification:

    [ ] Grafana Pod is Running
    [ ] Grafana Service exists
    [ ] Can open Grafana Web
    [ ] Can log in to Grafana

### 35.2 Task Two: Add Loki Data Source

Configuration:

    Name:
        Loki

    URL:
        http://loki-gateway.logging.svc.cluster.local

    Access:
        Server / Proxy

Click:

    Save & test

Verification:

    [ ] Loki Data Source added successfully
    [ ] Save & test succeeded
    [ ] Grafana can connect to Loki

### 35.3 Task Three: Explore Log Queries

Execute LogQL:

    {namespace="app-demo"}

    {namespace="app-demo", app="nginx-demo"}

    {namespace="app-demo", app="nginx-demo"} |= "404"

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"

Verification:

    [ ] Explore can find app-demo logs
    [ ] Can find nginx-demo logs
    [ ] Can find 404 logs
    [ ] Can find JSON error logs

### 35.4 Task Four: Create Dashboard Variables

Create variables:

    cluster
    namespace
    app
    pod
    container
    node
    team

Verification:

    [ ] namespace variable can display app-demo
    [ ] app variable can display nginx-demo/json-log-demo
    [ ] pod variable can display corresponding Pods
    [ ] container variable can display nginx/json-log-demo containers

### 35.5 Task Five: Create Log Trend Panel

Create panel:

    Log volume trend by app

Query:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}[$__interval]
      )
    )

Verification:

    [ ] Can see log volume trend
    [ ] Panel changes after switching namespace/app

### 35.6 Task Six: Create ERROR Trend Panel

Query:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          |~ "(?i)error|exception|panic|failed" [$__interval]
      )
    )

Verification:

    [ ] Can see error log trend

### 35.7 Task Seven: Create 5xx Panel

JSON Query:

    sum by (app, status) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          | json
          | __error__=""
          | status >= 500 [$__interval]
      )
    )

Verification:

    [ ] json-log-demo has 5xx statistics

### 35.8 Task Eight: Create P95 Slow Request Panel

Query:

    quantile_over_time(
      0.95,
      {namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

Verification:

    [ ] Can see P95 duration_ms
    [ ] Unit set to milliseconds

### 35.9 Task Nine: Create Recent Error Log Panel

Query:

```markdown
{namespace=~"$namespace", app=~"$app"}
  |~ "(?i)error|exception|panic|failed|timeout"

Acceptance:

  [ ] Can view detailed recent error logs

### 35.10 Task Ten: Export Dashboard

Execution:

  Dashboard Settings
    ↓
  JSON model
    ↓
  Copy / Export

Save as:

  k8s-loki-logs-overview.json

Acceptance:

  [ ] Dashboard JSON has been exported
  [ ] Saved to knowledge base or Git repository

---

## Thirty-six, Acceptance Checklist

After completing this document, the following should be confirmed:

  [ ] Grafana has been deployed
  [ ] Grafana Web is accessible
  [ ] Loki Data Source has been added
  [ ] Data Source Save & test succeeded
  [ ] Explore can query app-demo logs
  [ ] Explore can query nginx-demo logs
  [ ] Explore can query json-log-demo logs
  [ ] Dashboard has been created
  [ ] cluster variable has been created
  [ ] namespace variable has been created
  [ ] app variable has been created
  [ ] pod variable has been created
  [ ] container variable has been created
  [ ] node variable has been created
  [ ] Log volume trend panel has been created
  [ ] ERROR trend panel has been created
  [ ] 5xx trend panel has been created
  [ ] timeout trend panel has been created
  [ ] P95 slow request panel has been created
  [ ] Top Error Pod panel has been created
  [ ] Recent error log panel has been created
  [ ] Pod log details panel has been created
  [ ] Dashboard supports variable filtering
  [ ] Dashboard default time range is reasonable
  [ ] Dashboard JSON has been exported
  [ ] Understand the difference between Dashboard and Explore
  [ ] Understand the difference between Dashboard and alerts
  [ ] Understand the importance of query scope and variable governance in production

---

## Thirty-seven, Common Misconceptions

### 37.1 Misconception One: Grafana automatically collects logs after adding Loki

Incorrect.

Grafana is only an entry point for querying and displaying logs.

Log collection depends on:

  Alloy
  Promtail
  Fluent Bit
  Filebeat

Log storage depends on:

  Loki

### 37.2 Misconception Two: Dashboard default All is most convenient

Not necessarily.

All can easily lead to broad range queries.

Production recommendations:

  namespace is mandatory
  app is mandatory
  Default time range is Last 15 minutes or Last 1 hour

### 37.3 Misconception Three: More log dashboards are better

Incorrect.

More panels increase query pressure.

Panels should be designed based on troubleshooting workflows, not by filling all possible queries.

### 37.4 Misconception Four: ERROR keyword equals real errors

Not necessarily.

Some logs may contain:

  no error
  expected error
  ignore error

Structured logs are recommended:

  level="error"

### 37.5 Misconception Five: P95 panel can be adapted to all applications

Incorrect.

P95 panel depends on log field:

  duration_ms

If the application lacks this field, the panel has no data.

### 37.6 Misconception Six: Pod name is suitable as a long-term Dashboard main dimension

Pod names change frequently.

For long-term observation, use:

  namespace + app

For temporary troubleshooting, drill down to:

  pod + container

### 37.7 Misconception Seven: Slow Dashboard queries are Grafana issues

Not necessarily.

Possible causes include:

  Loki query is slow
  LogQL scope is too large
  All variable expansion is excessive
  Heavy regex
  Global JSON parsing
  Object storage is slow
  Loki resources are insufficient
  Too many panels

---

## Thirty-eight, Production Notes

### 38.1 Data sources should be managed via provisioning

Production recommendations:

  Data Source provisioning
  Dashboard provisioning
  Git manage JSON
  Environment differences managed via values

Avoid long-term reliance on manual configuration.

### 38.2 Dashboards should have access controls

Different teams may only view their own namespace/app.

Consider:

  Grafana Folder permissions
  Team permissions
  Data Source permissions
  Multi-tenant Loki
  Label Based Access Control
  Organization isolation

### 38.3 Query scopes should be controlled

Production recommendations:

  Default Last 15 minutes or Last 1 hour
  Limit large range queries
  Avoid All variables
  Avoid complex regex
  Avoid global JSON parsing
  High-frequency panels use lighter queries

### 38.4 Dashboards should be combined with Runbooks

Each core Dashboard should include:

  Fault description
  Common queries
  Troubleshooting steps
  kubectl commands
  Responsible person
  Alert links
  Change log

### 38.5 Dashboards should form a closed-loop with alerts

Dashboards are only observation tools.

Production must combine with:

  Loki Ruler
  AlertManager
  Prometheus alerts
  Runbook
  On-call process

---

## Thirty-nine, Summary

This document completes the practical implementation of Grafana integration with Loki and log Dashboards.

Core workflow:

  Loki Data Source
    ↓
  Grafana Explore
    ↓
  LogQL queries
    ↓
  Dashboard variables
    ↓
  Log trend panels
    ↓
  Error log panels
    ↓
  5xx panels
    ↓
  Slow request panels
    ↓
  Pod log details
    ↓
  Troubleshooting closed-loop

The correct way to use Grafana + Loki is: /think
```

Explore is used for temporary query debugging.  
Dashboard is used to consolidate stable troubleshooting entry points.  
Variables are used to narrow down query scope.  
Panels are used to display trends and details.  
Prometheus detects anomalies, Loki identifies root causes.  
Runbook guides incident handling.  

Production-grade Dashboard is not simply displaying logs, but must support:  

    Quickly locate namespace  
    Quickly locate app  
    Quickly locate pod  
    Quickly locate error logs  
    Quickly locate 5xx errors  
    Quickly locate slow requests  
    Quickly jump from metrics to logs  
    Quickly enter handling workflow  

Key principles:  

    Labeling standards are unified.  
    Log fields are standardized.  
    Query scope is controlled.  
    Variable design is reasonable.  
    Panel count is restrained.  
    Default time range is reasonable.  
    Avoid overuse of All.  
    Avoid global wide-range scanning.  
    Dashboard integrates with alerts and Runbook.  

Next topic:  

    11-Loki Log Alerting in Practice: Ruler and AlertManager Integration  

Key learning points:  

    Loki Ruler  
    Log alerting rules  
    ERROR log alerting  
    5xx log alerting  
    timeout log alerting  
    CUDA OOM log alerting  
    AlertManager receives Loki alerts  
    Alert grouping, suppression, routing, and Runbook links  

---

## Reference Documentation  

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/  

- Configure the Loki data source:  
  https://grafana.com/docs/grafana/latest/datasources/loki/configure-loki-data-source/  

- Loki data source:  
  https://grafana.com/docs/grafana/latest/datasources/loki/  

- Loki query editor:  
  https://grafana.com/docs/grafana/latest/datasources/loki/query-editor/  

- Loki template variables:  
  https://grafana.com/docs/grafana/latest/datasources/loki/template-variables/  

- Logs in Explore:  
  https://grafana.com/docs/grafana/latest/visualizations/explore/logs-integration/  

- Visualize log data:  
  https://grafana.com/docs/loki/latest/visualize/grafana/  

- Query Loki:  
  https://grafana.com/docs/loki/latest/query/  

- Log queries:  
  https://grafana.com/docs/loki/latest/query/log_queries/  

- LogQL Reference:  
  https://grafana.com/docs/loki/latest/query/query_reference/  

- Query examples:  
  https://grafana.com/docs/loki/latest/query/query_examples/  

- Grafana Dashboards:  
  https://grafana.com/docs/grafana/latest/dashboards/  

- Add variables:  
  https://grafana.com/docs/grafana/latest/dashboards/variables/  

- Dashboard provisioning:  
  https://grafana.com/docs/grafana/latest/administration/provisioning/  

- Grafana Helm Chart:  
  https://github.com/grafana/helm-charts/tree/main/charts/grafana  

- Kubernetes Logging Architecture:  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/