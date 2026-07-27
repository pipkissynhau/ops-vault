# 10-Grafana Integration with Loki and Log Dashboard Practical Guide

## Document Description

This article is the tenth in the Loki series, focusing on systematically learning how to integrate Grafana with Loki, use Explore to query logs, create Loki log dashboards, configure variables, build log trend panels, error logs panels, 5xx panels, slow request panels, and utilize Loki's logging capabilities as a production troubleshooting tool.

What has been completed previously:

    01-Loki Basics and Experimental Environment Setup
    02-Loki Architecture and Component Responsibilities in Practice
    03-Comparison of Loki Deployment Modes and Experimental Selection
    04-Helm Deployment of Loki in Standalone Mode
    05-Integration of Loki Object Storage with MinIO
    06-Grafana-Alloy for Collecting K8S-Pod Logs
    07-Loki Tag Design and High Cardinality Issues
    08-Basic LogQL Queries: Namespace-Pod-Container Log Retrieval
    09-Advanced LogQL Queries: json-logfmt-regexp-unwrap

In Chapter 08, you have mastered basic LogQL:

    {namespace="app-demo"}
    {namespace="app-demo", app="nginx-demo"}
    {namespace="app-demo"} |= "404"
    {namespace="app-demo"} |~ "(?i)error|exception|panic"

In Chapter 09, you have learned advanced LogQL:

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

This article will apply these querying capabilities in Grafana to create visual troubleshooting tools.

This article addresses the following key questions:

- How to deploy or verify that Grafana is available;
- How to add a Loki Data Source in Grafana;
- How to add a Loki data source using the UI;
- How to declare a Loki data source using provisioning;
- How to query Loki logs in Explore;
- How to design Namespace/App/Pod/Container/Node variables;
- How to create Pod log detail panels;
- How to create ERROR log trend panels;
- How to create 5xx log trend panels;
- How to create slow request P95 panels;
- How to create Top Error Pod panels;
- How to create recent error logs panels;
- How to use dashboards for Kubernetes Pod troubleshooting;
- How to link Prometheus metrics with Loki logs for troubleshooting;
- What considerations should be taken for dashboard query performance;
- How to manage permissions, variables, query scope, and alert redirects in a production environment using Grafana + Loki.

---

## Tags

#Grafana #Loki #LogQL #Dashboard #Explore #Kubernetes #Podlogs #Logvisualization #SRE #Observability #LogTroubleshooting #GrafanaVariables

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/10-GrafanaIntegrationwithLokiandLogDashboardPracticalGuide.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Verify that Grafana has been deployed and is accessible.
    2. Add a Loki Data Source in Grafana.
    3. Use Grafana Explore to query Loki logs.
    4. Create Loki log dashboards.
    5. Create cluster variables.
    6. Create namespace variables.
    7. Create app variables.
    8. Create pod variables.
    9. Create container variables.
    10. Create node variables.
    11. Create Pod log detail panels.
    12. Create ERROR log trend panels.
    13. Create 5xx log trend panels.
    14. Create slow request P95 panels.
    15. Create Top Error Pod panels.
    16. Create recent error logs panels.
    17. Understand the difference between Grafana Explore and Dashboard.
    18. Understand the difference between log panels and metric panels.
    19. Understand how to link Prometheus metrics with Loki logs.
    20. Master the production design principles for Grafana + Loki dashboards.

---

## II. Experimental Environment

### 2.1 Kubernetes Cluster

Experimental nodes:

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
    logfmt-log### 5.2 Adding a Helm Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 5.3 Querying Grafana Charts

    helm search repo grafana/grafana --versions | head -20

Record the version:

    GRAFANA_CHART_VERSION=<actual version found>

### 5.4 Creating Grafana Values

Create a file:

    values-grafana.yaml

Example content:

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

Note:

    Simple passwords can be used in a learning environment.
    Strong passwords must be used in a production environment, or unified authentication should be implemented.
    Do not submit production passwords in plaintext to Git.

### 5.5 Installing Grafana

    helm install grafana grafana/grafana \
      -n monitoring \
      --version <GRAFANA_CHART_VERSION> \
      -f values-grafana.yaml

### 5.6 Checking Grafana Status

    helm list -n monitoring

    kubectl get pods -n monitoring -o wide

    kubectl get svc -n monitoring

    kubectl get pvc -n monitoring

### 5.7 Accessing Grafana

Port forwarding:

    kubectl port-forward svc/grafana 3000:80 -n monitoring

Access:

    http://127.0.0.1:3000

Login:

    Username:
        admin

    Password:
        admin123

If Helm automatically generates a password, you can execute:

    kubectl get secret --namespace monitoring grafana \
      -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

---

## VI. Adding a Loki Data Source in Grafana: UI Method

### 6.1 Going to the Data Source Page

Path:

    Grafana left sidebar menu
      ↓
    Connections
      ↓
    Data sources
      ↓
    Add data source
      ↓
    Select Loki

The UI menu may vary slightly depending on the Grafana version.

Main objective:

    To add a Loki Data Source

### 6.2 Configuring the Loki URL

If both Grafana and Loki are within the same Kubernetes cluster, it is recommended to use the internal Service address.

Loki Gateway address:

    http://loki-gateway.logging.svc.cluster.local

If there is no Gateway, you can use the Loki Service:

    http://loki.logging.svc.cluster.local:3100

It is recommended in this document to use:

    http://loki-gateway.logging.svc.cluster.local

### 6.3 Saving and Testing

Click:

    Save & test

Expected result:

    "Data source connected and labels found"

or a similar successful message.

If it fails, check:

    Whether the Loki Gateway Service exists.
    Whether the Loki Gateway has an endpoint.
    Whether Loki is ready.
    Whether the Grafana Pod can access the Loki Service.
    If the URL or namespace is incorrect.
    If the port number is incorrect.

### 6.4 Testing Loki from Within the Grafana Pod

Enter the Grafana Pod:

    kubectl get pod -n monitoring | grep grafana

    kubectl exec -it <grafana-pod-name> -n monitoring -- sh

Inside the container, execute:

    wget -qO- http://loki-gateway.logging.svc.cluster.local/ready

or:

    curl -s http://loki-gateway.logging.svc.cluster.local/ready

If curl/wget is not available in the image, you can use a temporary Pod for testing.

---

## VII. Adding a Loki Data Source in Grafana: Provisioning Method

### 7.1 Why Provisioning is Needed

Adding data sources via the UI is suitable for learning purposes.

For production environments, it is recommended to manage data sources through provisioning.

Benefits:

    Version control is possible.
    Can be managed using Git.
    Allows for repeatable deployments.
    Prevents manual configuration drift.
    Ensures consistency across multiple environments.

### 7.2 Adding datasources to values-grafana.yaml

You can add the following to the Grafana Helm values file:

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
              maxLines:{namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000

Format:

{namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | level="error"
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 8.4 Time Range

Select from the top right corner of Explore:

    Last 5 minutes
    Last 15 minutes
    Last 1 hour
    Last 6 hours

Suggestions:

    For daily troubleshooting, start with Last 15 minutes.
    Use the appropriate time range for fault analysis.
    Avoid checking Last 7 days at the beginning.

### 8.5 View Log Details

Expand a log in Explore to see:

    Timestamp
    Labels
    Fields
    Parsed fields
    Structured metadata

Key points to check:

    Is the namespace correct?
    Is the app correct?
    Is the pod correct?
    Is the container correct?
    Is the node correct?
    Is the team correct?
    Are there any parsed fields?
    Is there an __error__?

---

## IX. Creating a Loki Dashboard

### 9.1 New Dashboard

Path:

    Dashboards
      ↓
    New
      ↓
    New dashboard

Suggested names:

    Kubernetes / Loki Pod Logs Overview

or:

    K8S Loki Log Troubleshooting Summary

### 9.2 Dashboard Purpose

This dashboard is not for decorative purposes but to assist in troubleshooting.

Key questions:

    Which namespace has the most logs?
    Which app has the most errors?
    Which pod has the most errors?
    What are the recent error logs?
    Are there any 5xx errors?
    Are there any timeout errors?
    Are there any slow request errors?
    Can I quickly drill down to a specific pod/container?
    Can it be used in conjunction with Prometheus metrics for troubleshooting?

### 9.3 Recommended Panel Layout

Suggested sections:

    First row:
        Query variables and explanations

    Second row:
        Total logs, error logs, 5xx logs, timeout logs

    Third row:
        Trends in log volume, ERROR trends, 5xx trends, slow request trends

    Fourth row:
        Top error app, top error pod, top 5xx paths

    Fifth row:
    Recent error logs, pod log details

    Sixth row:
    Runbook links, common LogQL explanations

---

## X. Designing Dashboard Variables

### 10.1 Variable Design Principles

Variables should help users narrow down their search:

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

Avoid starting with a global search:

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

    If there is only one cluster in the learning environment, use k8s-lab as the default.

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

If there is no cluster tag:

    label_values(namespace)

### 10.4 app Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, app)

If there is no cluster:

    label_values({namespace="$namespace"}, app)

Name:

    app

### 10.5 pod Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace", app="$app"}, pod)

If the app field is empty or some logs lack it:

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

### 10.9 Usage Notes for Variables

If Multi-value or All is enabled:

    Use regular expressions in LogQL:

        namespace=~"$namespace"
        app=~"$app"
        pod=~"$pod"

If Multi-value is not enabled:

    Use exact matches:

        namespace="$namespace"
        app="$app"

For production### 13.2 Panel Types

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

    JSON ERROR Log Trends

### 13.4 Note

If the selected app does not generate JSON logs, parsing errors will occur.

Already used:

    __error__=""

This filter is for parsing failures.

In production, it is recommended to create separate panels for JSON and non-JSON applications.

---

## Section Fourteen: Panel Four: 5xx Log Trends

### 14.1 Panel Objective

To display trends in HTTP 5xx errors.

Suitable for:

    API services
    Nginx access logs
    Ingress logs
    JSON-based service access logs

### 14.2 JSON Format

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
          | __error__ ""
          | status >= 500 [$__interval]
      )
    )

### 14.3 Nginx Text Log Format

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ " 5[0-9][0-9] " [$__interval]
      )
    )

No cluster:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          |~ " 5[0-9][0-9] " [$__interval]
      )
    )

### 14.4 Title

    5xx Log Trends

### 14.5 Note

The text log format depends on the specific logging structure.

If the Nginx log format changes, the regular expression used for parsing may no longer work effectively.

In production, it is recommended to use structured JSON fields in logs:

    status: 500

---

## Section Fifteen: Panel Five: Timeout Log Trends

### 15.1 Panel Objective

To display logs related to timeouts, such as "timeout", "deadline exceeded", and "timed out".

### 15.2 Query

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ "(?i)timeout|timed out|deadline exceeded" [$__interval]
      )
    )

No cluster:

    sum by (app) (
      count_over_time(
        {namespace=~"$namespace", app=~"$app"}
          |~ "(?i)timeout|timed out|deadline exceeded" [$__interval]
      )
    )

### 15.3 Title

    Timeout Log Trends

### 15.4 Suitable Scenarios

    Database access timeouts
    Redis timeouts
    HTTP downstream timeouts
    gRPC deadline exceeded errors
    AI inference timeout failures
    Object storage access timeouts

---

## Section Sixteen: Panel Six: Slow Requests P95

### 16.1 Panel Objective

To calculate the P95 value based on the `duration_ms` field in JSON logs.

Suitable for:

    API request durations
    Inference request times
    Batch processing task execution times
    Database operation durations

### 16.2 Query

    quantile_over_time(
      0.95,
      {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

No cluster:

    quantile_over_time(
      0.95,
      {namespace=~"$namespace", app=~"$app"}
        | json
        | unwrap duration_ms
        | __error__="" [$__interval]
    )

### 16.3 Title

    P95 Request Duration: duration_ms

### 16.4 Unit

    Unit:

    milliseconds

### 16.5 Note

The log must contain a numeric field named `duration_ms`.

If this field istopk(10,
      sum by (app) (
        count_over_time(
          {namespace=~"$namespace"}
            |~ "(?i)error|exception|panic|failed" [$__range]
        )
      )
    )

### 19.2 Title

Top 10 Error-Prone Apps

### 19.3 Use Case

Ideal for quickly identifying from a platform perspective:

- Which app generates the most errors
- Which team needs to address these issues
- Which namespace experiences the most anomalies

---

## Section 20: Panel 10: Recent Error Logs

### 20.1 Purpose of the Panel

Displays details of recent error logs.

### 20.2 Panel Type

Panel type:

    Logs

### 20.3 Query Parameters

    {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
      |~ "(?i)error|exception|panic|failed|timeout"

If no cluster is specified:

    {namespace=~"$namespace", app=~"$app"}
      |~ "(?i)error|exception|panic|failed|timeout"

### 20.4 Title

Recent Error Logs

### 20.5 Recommended Settings

It is recommended to configure the following options:

- Show time:
        true

- Enable log details:
        true

- Deduplication:
        none or exact

- Prettify JSON:
        true

- Limit:
        100 or 200

### 20.6 Notes

This panel is dedicated to viewing detailed logs.

Do not set a too wide time range for the display.

It is suggested that the Dashboard default to:

    Last 15 minutes
    Last 1 hour

---

## Section 21: Panel 11: Pod Log Details

### 21.1 Purpose of the Panel

After selecting a specific Pod and Container based on variables, this panel displays the logs for that Pod.

### 21.2 Panel Type

Panel type:

    Logs

### 21.3 Query Parameters

If multiple selections are allowed for the variables:

    {cluster=~"$cluster", namespace=~"$namespace", pod=~"$pod", container=~"$container"}

If multiple selections are not allowed:

    {namespace="$namespace", pod="$pod", container="$container"}

### 21.4 Title

Pod Log Details

### 21.5 Use Case

Suitable for:

- Troubleshooting Pod CrashLoopBackOff issues
- Tracking logs of a single Pod
- Investigating multi-container Pods
- Diagnosing Sidecar-related problems

### 21.6 Additional Notes

The panel description should indicate that it is used to view the original logs of a specified Pod/Container.

If a Pod has just been restarted, it is still recommended to use `kubectl logs --previous` to check the previous round of container logs as well.

Whether the previous logs are retained in Loki depends on the timing of data collection and writing.

---

## Section 22: Panel 12: JSON-Formatted Error Logs

### 22.1 Purpose

Formats JSON logs into a more readable format for troubleshooting purposes.

### 22.2 Panel Type

Panel type:

    Logs

### 22.3 Query Parameters

    {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
      | json
      | __error__=""
      | level="error"
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

If no cluster is specified:

    {namespace=~"$namespace", app=~"$app"}
      | json
      | __error__=""
      | level="error"
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 22.4 Title

JSON-Formatted Error Logs

### 22.5 Notes

This panel is only applicable to JSON logs.

If the app does not generate JSON logs, this panel will likely display no results.

---

## Section 23: Panel 13: Log Quantity by Status Code

### 23.1 Purpose

Counts the number of logs based on their status codes.

### 23.2 Panel Type

Panel type:

    Bar chart
    Table
    Time series

### 23.3 Query Parameters

    sum by (status) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          | json
          | __error__="" [$__range]
      )
    )

If no cluster is specified:

    sum by (status) (
      count_over_time### 26.4 Fourth Row: Log Details

Panel:

    Recent Error Logs
    Pod Log Details
    JSON Error Log Formatting View

### 26.5 Fifth Row: Description

Panel:

    Common LogQL Queries
    Runbook Links
    Tag Specification Notes
    Query Guidelines

---

## Twenty-Seven, Common Dashboard Explanation Text

It is recommended to add a Text panel with the following content:

    Usage Instructions:

    1. Preferably filter by cluster, namespace, and app before viewing logs.
    2. It is not advised to select all namespaces for queries in a production environment.
    3. Searching using the ERROR keyword should be used only as an auxiliary tool; structured logs with level="error" are preferred.
    4. The 5xx panel relies on the presence of a status field in the logs.
    5. The P95 latency panel depends on a duration_ms numerical field in the logs.
    6. Pod log details are suitable for temporary troubleshooting; for long-term monitoring, it is better to use the app dimension.
    7. If no results are found, check the time range, variable values, and Loki labels first.
    8. If sensitive information appears in the logs, prioritize fixing the application's logging configuration.

---

## Twenty-Eight, Linking Prometheus Metrics with Loki Logs

### 28.1 Why Link Them

In production environment troubleshooting, metrics are often used to identify issues before checking logs:

    Pod restart
    High CPU usage
    High memory usage
    Increase in 5xx errors
    Rise in P95 latency
    High GPU memory usage
    OOMKilled

Only after identifying the issue through metrics should you proceed to examine the logs for further details.

Recommended workflow:

    Prometheus alarm
      ↓
    Grafana metric panel
      ↓
    Loki log panel
      ↓
    kubectl describe /Events
      ↓
    Runbook intervention

### 28.2 Using Variables in Prometheus Panels

If Prometheus and Loki use the same tags:

    namespace
    app
    pod
    container

variables can be shared between them.

Example Prometheus query:

    sum by (pod) (
      rate(container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod", container!="", image!=""}[5m])
    )

Corresponding Loki query:

    {namespace="$namespace", pod=~"$pod"}

### 28.3 Data Link Concept

In the Prometheus Pod panel, you can configure a Data link to redirect to the Loki Dashboard.

Variables passed include:

    namespace
    pod
    container
    from
    to

Example workflow:

    Click on a specific pod in the Pod CPU panel
      ↓
    Redirect to the Loki log dashboard
      ↓
    The namespace, pod, and container are automatically populated
      ↓
    View the logs for that pod

### 28.4 Prerequisites for Production Use

For effective linking, ensure:

    Consistent metric and log tags
    Matching namespaces, pods, containers, and apps
    Unified variable naming in Grafana
    Stable Dashboard UIDs

---

## Twenty-Nine, Connecting Derived Fields with Traces

### 29.1 What are Derived Fields

Grafana's Loki data source allows you to configure derived fields, which extract specific information from logs and generate links.

Common use cases:

    Linking a trace_id in logs to the Tempo Trace system
    Redirecting a request_id in logs to an internal query system
    Orienting users to a business troubleshooting page based on an order_id

### 29.2 Example for trace_id

If a log contains:

    trace_id=abc123

or a JSON field:

    "trace_id":"abc123"

you can configure a Derived Field:

    Name:
        trace_id

    Regex:
        trace_id["=:\s]+([a-zA-Z0-9-]+)

    URL / Internal link:
        Points to Tempo or another system

### 29.3 Important Notes

Do not use the trace_id as a Loki Label.

The correct approach is:

    Include the trace_id in the log text or structured fields.
    Grafana extracts it through derived fields.
    Users can then click on the link to navigate to the Trace system.

### 29.4 Overview of This Series

This document focuses solely on the Loki Dashboard. The integration with Tempo Trace is not covered here.

However, it is important to understand that:

    The combination of logs, metrics, and traces is a crucial aspect of production-level observability.

---

## Thirty, Exporting and Importing Dashboards

### 30.1 Exporting a Dashboard

On the Grafana Dashboard page:

    Settings
      ↓
    JSON model
      ↓
    Copy JSON

Or:

    Share
      ↓
    Export

It is### 4. Is the URL correct?
### 5. Have the wrong port been specified?
### 6. Is Loki/ready functioning normally?
### 7. Is NetworkPolicy blocking anything?
### 8. Is DNS resolving properly?

Commands:

```bash
kubectl get svc -n logging
kubectl get endpoints -n logging
kubectl get pod -n monitoring | grep grafana
kubectl exec -it <grafana-pod-name> -n monitoring -- sh
```

Inside the container:

```bash
wget -qO- http://loki-gateway.logging.svc.cluster.local/ready
```

### 32.2 The Explore query returns no results

Troubleshooting:

1. Is the time range correct?
2. Do the Loki labels exist?
3. Are the namespace/app/pod variables set?
4. Is the query statement correct?
5. Has Alloy collected any logs?
6. Was the data successfully written to Loki?
7. Maybe the wrong data source was selected.

First, check directly in Explore:

```bash
{namespace="app-demo"}
```

Then narrow down the scope gradually:

```bash
{namespace="app-demo", app="nginx-demo"}
```

### 32.3 The Dashboard variables show no data

Troubleshooting:

1. Is the data source set to Loki?
2. Is the Query statement correct?
3. Is the label_values query correct?
4. Is there no data within the current time range?
5. Does Loki have that label?
6. Is the order of variable dependencies correct?

Check labels:

```bash
curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq
```

Check app:

```bash
curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq
```

### 32.4 The panel displays a "parse error"

Common causes:

- Syntax errors in LogQL.
- Syntax errors after variable expansion.
- For multi-value variables, use =~ instead of =.
- Missing quotes.
- Regular expression escape errors.
- Incorrect position for json/unwrap.

Example:

Incorrect:
```bash
{namespace="$namespace"}
```

This may fail or return no results if multiple namespaces are selected.

Recommended:
```bash
{namespace=~"$namespace"}
```

### 32.5 The JSON panel shows no data

Possible reasons:

- The application logs are not in JSON format.
- Parsing failed.
- __error__ was filtered out.
- The status field is missing.
- The duration_ms field is not a number.
- The wrong app variable was selected.

First, check the original logs:

```bash
{namespace="$namespace", app="$app"}
```

Then check for parsing errors:

```bash
{namespace="$namespace", app="$app"} | json | __error__!=""
```

### 32.6 Dashboard queries are very slow

Reasons:

- The query range is too large.
- All variables were selected by default.
- There are too many panels.
- The regular expression is too broad.
- Large-scale json/regexp parsing was used.
– __range is too extensive.
- Insufficient Loki resources.
- Slow reading from object storage.

Optimizations:

- Narrow the default time range.
- Avoid selecting All by default.
- Reduce the number of panels.
- First use labels to narrow down the scope.
- Prefer structured logs whenever possible.
- Consider using recording rules for large queries.
- Plan caching and throttling for frequently accessed panels.

---

## Section 33: Design Principles for Production Dashboards

### 33.1 Do not default to global queries

It is not recommended to set Dashboard defaults to:

```bash
namespace=All
app=All
time_range=Last 7 days
```

Recommendations:

- `namespace` must be specified.
- `app` must be specified.
- Time range should be `Last 15 minutes` or `Last 1 hour`.

### 33.2 Dashboards should have clear purposes

Separate different Dashboards for:

- K8S Loki log overview
- Application log troubleshooting
- Pod log details
- Ingress access logs
- Error log analysis
- Slow request analysis
- Loki self-monitoring
- Alloy collection monitoring

Do not combine all functions into one Dashboard.

### 33.3 Panels should include descriptions

It is recommended to provide descriptions for each key panel, including:

- Query logic
- Dependent fields
- Applicable scenarios
- Steps to troubleshoot if the query returns no results

For example, a description for the P95 panel could be:

```markdown
This panel relies on the `duration_ms` field in JSON logs.
If the application does not include this field, the panel will show no data.
If `container
node
team

Acceptance:

[ ] The namespace variable can display app-demo.
[ ] The app variable can display nginx-demo/json-log-demo.
[ ] The pod variable can display the corresponding Pod.
[ ] The container variable can display containers such as nginx/json-log-demo.

### 35.5 Task Five: Create a Log Trend Panel

Create the panel:

Log volume trend by app

Query:

sum by (app) (
  count_over_time(
    {namespace=~"$namespace", app=~"$app"}[$__interval]
  )
)

Acceptance:

[ ] The log volume trend can be seen.
[ ] The panel changes when switching namespaces/app.

### 35.6 Task Six: Create an ERROR Trend Panel

Query:

sum by (app) (
  count_over_time(
    {namespace=~"$namespace", app=~"$app"}
      |~ "(?i)error|exception|panic|failed" [$__interval]
  )
)

Acceptance:

[ ] The error log trend can be seen.

### 35.7 Task Seven: Create a 5xx Panel

JSON Query:

sum by (app, status) (
  count_over_time(
    {namespace=~"$namespace", app=~"$app"}
      | json
      | __error__=""
      | status >= 500 [$__interval]
  )
)

Acceptance:

[ ] There is 5xx statistics for json-log-demo.

### 35.8 Task Eight: Create a P95 Slow Request Panel

Query:

quantile_over_time(
  0.95,
  {namespace=~"$namespace", app=~"$app"}
    | json
    | unwrap duration_ms
    | __error__="" [$__interval]
)

Acceptance:

[ ] The P95 duration_ms can be seen.
[ ] The unit is set to milliseconds.

### 35.9 Task Nine: Create a Recent Error Logs Panel

Query:

{namespace=~"$namespace", app=~"$app"}
      |~ "(?i)error|exception|panic|failed|timeout"

Acceptance:

[ ] The details of recent error logs can be seen.

### 35.10 Task Ten: Export the Dashboard

Execution:

Dashboard Settings
      ↓
JSON model
      ↓
Copy / Export

Save as:

    k8s-loki-logs-overview.json

Acceptance:

[ ] The Dashboard JSON has been exported.
[ ] It has been saved to the knowledge base or Git repository.

---

## Thirty-Six, Acceptance Checklist

After completing this document, the following should be confirmed:

[ ] Grafana has been deployed.
[ ] The Grafana Web interface is accessible.
[ ] The Loki Data Source has been added.
[ ] Saving and testing the Data Source were successful.
[ ] It is possible to query app-demo logs in Explore.
[ ] It is possible to query nginx-demo logs in Explore.
[ ] It is possible to query json-log-demo logs in Explore.
[ ] The Dashboard has been created.
[ ] The cluster variable has been created.
[ ] The namespace variable has been created.
[ ] The app variable has been created.
[ ] The pod variable has been created.
[ ] The container variable has been created.
[ ] The node variable has been created.
[ ] The log volume trend panel has been created.
[ ] The ERROR trend panel has been created.
[ ] The 5xx trend panel has been created.
[ ] The timeout trend panel has been created.
[ ] The P95 slow request panel has been created.
[ ] The Top Error Pod panel has been created.
[ ] The recent error logs panel has been created.
[ ] The Pod log details panel has been created.
[ ] The Dashboard supports variable filtering.
[ ] The default time range of the Dashboard is appropriate.
[ ] The Dashboard JSON has been exported.
[ ] The differences between the Dashboard and Explore are understood.
[ ] The differences between the Dashboard and alerts are understood.
[ ] The importance of query scope and variable management in production environments is understood.

---

## Thirty-Seven, Common Misconceptions

### 37.1 Misconception One: Adding Loki to Grafana will automatically collect logs

Wrong.

Grafana is only an interface for querying and displaying data.

Log collection depends on:

    Alloy
    Promtail
    Fluent Bit
    Filebeat

Log storage depends on:

    Loki

### 37.2 Misconception Two: The default "All" setting on the Dashboard is the most convenient

Not necessarily.

Using "All" can lead to extensive queries.

Recommended for production:

    Namespace must be specified.
    App must be specified.
    The default time range should be Last 15 minutes or Last 1 hour.

### 37.3 Misconception Three: The more log Dashboards, the better

Wrong.

The more panels there are, the greater the query load will be.

The dashboard is integrated with alarms and runbooks.

Next topic:

11-Loki Log Alarms in Action: Integrating Ruler with AlertManager

Key topics to learn:

Loki Ruler
Log alarm rules
ERROR log alerts
5xx log alerts
Timeout log alerts
CUDA OOM log alerts
AlertManager receiving Loki alerts
Alarm grouping, suppression, routing, and linking with runbooks

---

## References

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Configuring the Loki data source:
  https://grafana.com/docs/grafana/latest/datasources/loki/configure-loki-data-source/

- Loki data source:
  https://grafana.com/docs/grafana/latest/datasources/loki/

- Loki query editor:
  https://grafana.com/docs/grafana/latest/datasources/loki/query-editor/

- Loki template variables:
  https://grafana.com/docs/grafana/latest/datasources/loki/template-variables/

- Integrating logs in Explore:
  https://grafana.com/docs/grafana/latest/visualizations/explore/logs-integration/

- Visualizing log data:
  https://grafana.com/docs/loki/latest/visualize/grafana/

- Querying Loki:
  https://grafana.com/docs/loki/latest/query/

- Log queries:
  https://grafana.com/docs/loki/latest/query/log_queries/

- LogQL Reference:
  https://grafana.com/docs/loki/latest/query/query_reference/

- Query examples:
  https://grafana.com/docs/loki/latest/query/query_examples/

- Grafana Dashboards:
  https://grafana.com/docs/grafana/latest/dashboards/

- Adding variables:
  https://grafana.com/docs/grafana/latest/dashboards/variables/

- Dashboard provisioning:
  https://grafana.com/docs/grafana/latest/administration/provisioning/

- Grafana Helm Chart:
  https://github.com/grafana/helm-charts/tree/main/charts/grafana

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/