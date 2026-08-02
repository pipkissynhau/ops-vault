# 12-Loki Production Governance: Log Volume - Retention Period - Throttling - Security

## Document Overview

This is the twelfth article in the Loki specialized learning series, aimed at systematically understanding Loki's governance capabilities in production environments. The focus is on log volume governance, log retention periods, write throttling, query limits, label cardinality control, multi-tenant isolation, secure access, sensitive information protection, object storage governance, and Loki's own monitoring.

Previously completed:

    01-Loki Basic Understanding and Experimental Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experimental Selection
    04-Loki Single-Instance Helm Deployment Practical
    05-Loki Object Storage Integration with MinIO Practical
    06-Grafana-Alloy Collection of Kubernetes Pod Logs Practical
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practical: Namespace-Pod-Container Log Retrieval
    09-LogQL Advanced Query Practical: json-logfmt-regexp-unwrap
    10-Grafana Integration with Loki and Log Dashboard Practical
    11-Loki Log Alert Practical: Ruler and AlertManager Integration

The 04th article completed Loki single-instance deployment.
The 05th article completed Loki integration with MinIO object storage.
The 06th article completed Alloy collection of Kubernetes Pod logs.
The 07th article learned about Labels and high cardinality.
The 08th and 09th articles learned about LogQL queries.
The 10th article completed the Grafana Dashboard.
The 11th article completed Loki Ruler log alerts.

This article begins the production governance phase.

This article answers the following key questions:

- Why production Loki cannot only focus on "can it query logs";
- Why log volume needs governance;
- How to statistics which namespace / app / pod has the largest log volume;
- How to govern high-frequency logs such as DEBUG, health check, metrics, access log;
- How Loki retention works;
- What Compactor does in the log retention period;
- How to configure global retention;
- How to configure differentiated retention periods for different namespace / app;
- Why object storage lifecycle cannot be arbitrarily shorter than Loki retention;
- How to configure write throttling;
- How to configure query limits;
- How to avoid large-scale queries from overwhelming Loki;
- How to control high cardinality labels;
- How to plan multi-tenancy;
- What is the difference between Loki auth_enabled=false and auth_enabled=true;
- How to implement security control through gateway, reverse proxy, Grafana permissions, and NetworkPolicy;
- How to prevent sensitive information from entering logs;
- How to establish a Loki production security baseline;
- How to monitor and alert Loki itself.

This article is not just a parameter configuration list, but focuses on real production governance scenarios.

---

## Tags

#Loki #Grafana #LogGovernance #Retention #Compactor #LimitsConfig #RateLimit #Multi-tenant #SecurityBaseline #SensitiveInformationProtection #SRE #Kubernetes #MinIO #ObjectStorage #Observation

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/12-Loki Production Governance: Log Volume - Retention Period - Throttling - Security.md

---

## One. Experimental Objectives

After completing this article, you should be able to:

    1. Understand the core objectives of Loki production governance.
    2. Statistics log volume by namespace / app / pod dimensions.
    3. Identify high log volume sources.
    4. Govern low-value high-frequency logs.
    5. Understand how Loki retention works.
    6. Understand the role of Compactor in retention.
    7. Configure global log retention period.
    8. Configure differentiated retention periods for different streams.
    9. Understand the relationship between object storage lifecycle and Loki retention.
    10. Configure Loki write throttling.
    11. Configure Loki query limits.
    12. Understand the reason for 429 write throttling.
    13. Troubleshoot query slowness and large query issues.
    14. Govern high cardinality labels.
    15. Understand Loki's multi-tenant model.
    16. Plan production access methods with auth_enabled=true.
    17. Design Loki Gateway / Ingress / Grafana permission control.
    18. Identify risks of sensitive information in logs.
    19. Establish a Loki production security baseline.
    20. Establish monitoring and alerting for Loki itself.

---

## Two. Experimental Environment

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
    AlertManager
    nginx-demo
    json-log-demo
    logfmt-log-demo

### 2.2 Prerequisites

Need to confirm:

    [ ] Loki Pod Running
    [ ] Loki Gateway Accessible
    [ ] Loki has been integrated with MinIO
    [ ] Alloy can collect Pod logs
    [ ] Grafana can query Loki
    [ ] Loki Ruler has completed basic validation
    [ ] AlertManager can receive Loki alerts

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

---

## Three. Why Loki Needs Production Governance

### 3.1 Installing Loki Alone Does Not Equal Mastering Loki

Learning phase focus:

    Can Loki start
    Can Alloy collect
    Can Grafana query
    Can LogQL find logs
    Can AlertManager receive alerts

Production stage must also focus on:

    Is the log volume under control?
    Is storage continuously growing?
    Is the retention period taking effect?
    Are queries slowing down Loki?
    Is writing being rate-limited?
    Is the tag cardinality too high?
    Is sensitive information entering logs?
    Are multiple teams properly isolated?
    Are permissions reasonable?
    Is Loki itself being monitored?
    Is there a Runbook for failures?

Common issues with production Loki are not about "installation failure", but:

    Increasing log volume
    Slower queries
    Expensive storage
    Noisy alerts
    Messy tags
    Scattered permissions
    Difficulty cleaning up logs with sensitive information

### 3.2 Loki Production Governance Goals

Production governance goals:

    Accurate collection:
        Only collect valuable logs.

    Reliable storage:
        Object storage is reliable, data can be retained.

    Reasonable retention:
        Different environments and business have different retention policies.

    Fast querying:
        Reasonable query scope, tags, and Dashboard design.

    Controlled limits:
        Writing, querying, and stream counts have boundaries.

    Clear separation:
        Clear isolation for multi-tenancy, multi-teams, and multi-environments.

    Visibility:
        Loki has monitoring and alerts.

    Security:
        Sensitive information, secure access, and permission controls are in place.

---

## Four, Log Volume Governance Overview

### 4.1 Why Log Volume Might Lose Control

Common causes:

    Application defaults to DEBUG level
    Full access log output
    Health check logs every second
    /metrics frequently accessed and recorded
    Business loop abnormal error logging
    Exception stack repeated printing
    Sidecar / Service Mesh proxy logs are excessive
    Ingress logs not filtered
    Batch processing tasks output large details
    AI inference tasks output large fields
    Request/response bodies fully printed
    SQL parameters and error stacks repeated printing
    Broad log collection scope
    Alloy not filtering unnecessary namespaces
    Multiple collectors redundantly collect the same logs

### 4.2 Consequences of Log Volume Loss Control

Consequences:

    Loki write pressure increases
    Ingester memory increases
    Chunk count increases
    Object storage capacity grows
    Queries slow down
    Dashboard lags
    Ruler execution slows
    Alert noise increases
    Network bandwidth increases
    Node collector Agent pressure increases
    MinIO / S3 request volume increases
    Costs are uncontrollable

### 4.3 Log Volume Governance Priorities

Governance priorities:

    First priority:
        Reduce useless logs at the application level.

    Second priority:
        Filter obviously useless logs at the collection level.

    Third priority:
        Limit writing and querying at the Loki level.

    Fourth priority:
        Limit query scope at the Dashboard and alerting level.

Don't offload all pressure to Loki.

Correct approach:

    Application reduces useless logs
      ↓
    Alloy reduces useless logs
      ↓
    Loki limits abnormal writes
      ↓
    Grafana controls query scope
      ↓
    AlertManager converges alert noise

---

## Five, Statistics Log Volume: By namespace / app / pod

### 5.1 Statistics namespace log volume

LogQL:

    sum by (namespace) (
      count_over_time({namespace=~".+"}[5m])
    )

Purpose:

    Find which namespace has the most logs.

Production note:

    This query scope is large.
    Suggest using it during off-peak hours or with cluster/environment restrictions.

More recommended:

    sum by (namespace) (
      count_over_time({cluster="k8s-lab"}[5m])
    )

### 5.2 Statistics app log volume

LogQL:

    sum by (namespace, app) (
      count_over_time({namespace=~".+"}[5m])
    )

If specifying namespace:

    sum by (app) (
      count_over_time({namespace="app-demo"}[5m])
    )

### 5.3 Statistics pod log volume

LogQL:

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace="app-demo"}[5m])
      )
    )

Purpose:

    Find the pod with the most log writing.

### 5.4 Statistics container log volume

LogQL:

    topk(10,
      sum by (namespace, pod, container) (
        count_over_time({namespace="app-demo"}[5m])
      )
    )

Purpose:

    Identify whether the main container, sidecar, or proxy container is writing logs in a multi-container pod.

### 5.5 Statistics error log volume

LogQL:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 5.6 Statistics 5xx log volume

Structured JSON:

    sum by (namespace, app, status) (
      count_over_time(
        {namespace="app-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    )

Nginx text logs:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo", app="nginx-demo"}
          |~ " 5[0-9][0-9] " [5m]
      )
    )

### 5.7 Statistics timeout log volume

LogQL:

sum by (namespace, app) (
  count_over_time(
    {namespace="app-demo"}
      |~ "(?i)timeout|timed out|deadline exceeded" [5m]
  )
)

---

## Six. Log Volume Governance: Application Side

### 6.1 Application Side is the First Governance Point

The most effective governance must occur at the application side.

Reasons:

    Applications should not output logs without value, making all subsequent systems effortless.
    If applications already output massive logs, the collection side and Loki side can only passively handle them.

### 6.2 Production Log Level Guidelines

Recommendations:

    dev:
        DEBUG can be used appropriately.

    test:
        INFO is primary, with DEBUG enabled temporarily when necessary.

    staging:
        Should be close to prod, defaulting to INFO or WARN.

    prod:
        Default to INFO / WARN / ERROR.
        Long-term DEBUG is prohibited.
        DEBUG must have a switch and expiration time.

### 6.3 Unrecommended Output Content

Avoid outputting:

    Log for every health check success
    Log for every metrics access
    Large volume of loop progress logs
    Full request body
    Full response body
    Large object JSON
    Large array content
    Plaintext SQL parameters
    Repeated exception stack traces
    User privacy information
    token / password / secret

### 6.4 Recommended Structured Fields

Recommended JSON fields:

    timestamp
    level
    service
    environment
    trace_id
    span_id
    method
    path
    route
    status
    duration_ms
    msg
    error_type

Example:

    {"timestamp":"2026-05-01T12:00:00+08:00","level":"error","service":"order-api","trace_id":"abc123","method":"POST","route":"/api/orders/{id}","status":500,"duration_ms":1300,"error_type":"database_error","msg":"database connection failed"}

### 6.5 route is Better than Full Path

Not recommended:

    path="/api/orders/100001"
    path="/api/orders/100002"
    path="/api/orders/100003"

Recommended:

    route="/api/orders/{id}"

Reasons:

    Full path may result in excessive field values.
    Dashboard grouping by path can explode.
    route is more suitable for statistics and aggregation.

---

## Seven. Log Volume Governance: Alloy Collection Side

### 7.1 Collection Side Governance Goals

Alloy side can do:

    Limit collection namespace
    Exclude unnecessary namespace
    Exclude health check logs
    Exclude metrics logs
    Exclude low-value logs
    Perform desensitization on sensitive fields
    Supplement logs with stable labels
    Avoid duplicate collection

### 7.2 Collect Only Specified Namespace

Add a keep rule in discovery.relabel:

    rule {
      source_labels = ["__meta_kubernetes_namespace"]
      action        = "keep"
      regex         = "app-demo|app-prod|ai-prod"
    }

Notes:

    Only collect logs from app-demo, app-prod, and ai-prod.

Production Recommendations:

    Do not default collect all namespaces.
    System component logs, platform component logs, and business logs can be managed with separate strategies.

### 7.3 Exclude kube-system

Example:

    rule {
      source_labels = ["__meta_kubernetes_namespace"]
      action        = "drop"
      regex         = "kube-system"
    }

Notes:

    kube-system logs are valuable for cluster troubleshooting.
    Whether to exclude depends on platform requirements.
    kube-system can have shorter retention set separately.

### 7.4 Exclude Health Check Logs

If using loki.process, consider dropping health check logs.

Example idea:

    loki.process "pod_logs" {
      stage.drop {
        expression = ".*(/healthz|/readyz|/metrics).*"
      }

      forward_to = [loki.write.loki.receiver]
    }

Notes:

    The exact stage syntax depends on the current Alloy documentation.
    It is recommended to test in a test environment first in production.
    Do not accidentally delete valuable logs.

### 7.5 Desensitization Processing Idea

Replace obvious sensitive fields.

Example idea:

    Authorization: Bearer xxxxxx
      ↓
    Authorization: Bearer ******

    password=123456
      ↓
    password=******

Notes:

    Desensitization should be done at the application side.
    Desensitization at the collection side is just a supplement.
    Do not rely on Loki for post-cleanup of sensitive information.

### 7.6 Avoid Duplicate Collection

In DaemonSet mode, confirm:

    Each Alloy only collects Pod on its node.
    Do not have each Alloy watch the entire cluster and duplicate tail.
    Do not have Promtail, Alloy, and Fluent Bit send the same batch of logs to the same Loki.

Check:

    kubectl get pod -n logging -o wide | grep alloy

    kubectl logs <alloy-pod> -n logging --tail=200

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

If log volume suddenly doubles, check for duplicate collection.

## VIII. Log Retention Period Foundation

### 8.1 What is retention

Retention is the log retention period.

Example:

    dev logs are retained for 7 days
    test logs are retained for 15 days
    prod logs are retained for 30 days
    security logs are retained for 90 days

Logs should be cleaned up after exceeding the retention period.

### 8.2 Loki retention depends on Compactor

In the current Loki architecture, retention is typically executed by Compactor.

Compactor is responsible for:

    Index compression
    Processing retention
    Marking expired data
    Deleting expired chunks
    Cleaning up expired objects in object storage

### 8.3 Default does not equal automatic cleanup

Do not assume Loki will automatically delete logs by default.

In production, you must explicitly configure:

    compactor.retention_enabled
    limits_config.retention_period
    delete_request_store
    retention_delete_delay

If retention is not enabled, logs may remain indefinitely until storage is manually cleaned or object storage lifecycle deletes them.

### 8.4 Retention and object storage lifecycle

If MinIO / S3 is configured with lifecycle, please note:

    Object storage lifecycle should not be shorter than Loki retention.

Error:

    Loki retention = 30d
    MinIO lifecycle = 7d

Consequences:

    Loki index may still consider logs exist.
    But chunks have been deleted by object storage.
    Querying historical logs may result in errors or missing data.

Recommendation:

    MinIO lifecycle > Loki retention

Example:

    Loki retention = 30d
    MinIO lifecycle = 35d or longer

---

## IX. Configuring Global Retention

### 9.1 Backup current values

Execute:

    helm get values loki -n logging -a > backup-values-loki-before-retention.yaml

### 9.2 View current Loki values

    helm get values loki -n logging -a > values-loki-current.yaml

Search:

    grep -n "compactor" values-loki-current.yaml

    grep -n "retention" values-loki-current.yaml

    grep -n "limits_config" values-loki-current.yaml

### 9.3 Example configuration

Create or modify:

    values-loki-prod-governance.yaml

Add or confirm the following snippet based on original Loki values:

    loki:
      limits_config:
        retention_period: 168h

      compactor:
        working_directory: /var/loki/compactor
        compaction_interval: 10m
        retention_enabled: true
        retention_delete_delay: 2h
        delete_request_store: s3

Explanation:

    retention_period: 168h
        Retain logs globally for 7 days.

    retention_enabled: true
        Enable Compactor retention.

    retention_delete_delay: 2h
        Deletion delay to avoid risks of index and chunk deletion being out of sync.

    delete_request_store: s3
        Use object storage for deletion requests.
        If using filesystem, adjust configuration according to actual setup.

Note:

    This is an example configuration.
    Different Loki Helm Chart versions may have different fields.
    Must verify with helm show values and helm template before implementation.

### 9.4 Helm template check

Execute:

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-prod-governance.yaml \
      > loki-governance-rendered.yaml

Check:

    grep -n "retention" loki-governance-rendered.yaml

    grep -n "compactor" loki-governance-rendered.yaml

    grep -n "retention_period" loki-governance-rendered.yaml

### 9.5 Upgrade Loki

Execute:

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-prod-governance.yaml

Check:

    helm history loki -n logging

    kubectl get pods -n logging -o wide

### 9.6 View Loki logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "compactor|retention|delete|error|warn"

Monitor:

    Whether compactor is started
    Whether retention is enabled
    Whether there are object storage permission errors
    Whether there are bucket errors
    Whether there are delete permission errors

---

## X. Configuring Differential Retention

### 10.1 Why differential retention is needed

Different environments have different retention periods:

    dev:
        3 days or 7 days

    test:
        7 days or 15 days

prod:
        30 days

security:
        90 days or longer

debug temporary logs:
        1 day

Not all logs can be retained for the same duration.

### 10.2 Retention Based on Stream Selector

Example:

    loki:
      limits_config:
        retention_period: 720h
        retention_stream:
          - selector: '{namespace="app-demo"}'
            priority: 1
            period: 72h
          - selector: '{environment="dev"}'
            priority: 2
            period: 168h
          - selector: '{environment="prod"}'
            priority: 3
            period: 720h

Explanation:

    Default retention_period: 720h
        Default retention of 30 days.

    namespace="app-demo":
        Retained for 3 days.

    environment="dev":
        Retained for 7 days.

    environment="prod":
        Retained for 30 days.

Priority:

    When multiple retention_stream rules match, the rule with higher priority takes effect.
    The specific comparison rules follow the current Loki documentation.

### 10.3 Design Recommendations

Recommendations:

    lab/dev:
        3d - 7d

    test/staging:
        7d - 15d

    prod:
        30d

    audit/security:
        90d or per compliance requirements

    debug temporary namespace:
        1d - 3d

### 10.4 Verifying Retention Configuration

Check the final configuration:

    helm get values loki -n logging -a | grep -n "retention" -A 50

If /config is available:

    curl -s http://127.0.0.1:3100/config | grep -Ei "retention|compactor" -A 20

Note:

    /config may expose sensitive information.
    Do not expose it in production environments.

---

## ElevenI don't know.MinIO Object Storage Governance

### 11.1 Viewing Loki Bucket

Enter mc container:

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Set alias:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

View bucket:

    mc ls local

View Loki chunks:

    mc ls local/loki-chunks

Recursive view:

    mc find local/loki-chunks | head

### 11.2 Viewing Bucket Size

You can use:

    mc du local/loki-chunks

If mc version supports recursive statistics:

    mc du --recursive local/loki-chunks

Note:

    Commands may vary slightly across mc versions.
    If the command is unavailable, you can check capacity via MinIO Console.

### 11.3 MinIO Lifecycle Notice

In production, if setting MinIO lifecycle:

    Ensure the lifecycle duration is greater than Loki retention.

Example:

    Loki retention:
        30d

    MinIO lifecycle:
        35d or 45d

Do not let MinIO delete Loki chunks before they are needed.

### 11.4 MinIO Permission Governance

Do not recommend using root user for Loki in production.

Recommendations:

    Create a dedicated loki user
    Create a dedicated loki access key
    Only authorize loki-chunks / loki-ruler / loki-admin
    Do not grant MinIO administrator permissions
    Regularly rotate keys
    Secrets should not be stored in plain text in Git

### 11.5 MinIO Availability Governance

MinIO should not be a single replica in production.

Recommendations:

    Multi-node
    Erasure Coding
    Disk monitoring
    Capacity monitoring
    API delay monitoring
    4xx / 5xx monitoring
    Certificate validity monitoring
    Backup and recovery drills

---

## TwelveI don't know.Write Throttling Governance

### 12.1 Why Throttling is Needed

If an application suddenly writes a large amount of logs, it may affect the entire Loki.

Write throttling is used to protect:

    Loki Ingester
    Distributor
    Object storage
    Network
    Memory
    Query performance
    Other tenants

### 12.2 Common Write Limitations

Common limitation directions:

    Per-tenant write rate
    Per-tenant burst write
    Per stream write rate
    Per stream burst write
    Maximum stream count
    Maximum single log size
    Label count limit
    Label name length limit
    Label value length limit
    Rejection of old logs
    Rejection of logs with future timestamps

### 12.3 Example limits_config

Example:

    loki:
      limits_config:
        ingestion_rate_mb: 8
        ingestion_burst_size_mb: 16

        per_stream_rate_limit: 5MB
        per_stream_rate_limit_burst: 20MB

        max_streams_per_user: 10000
        max_global_streams_per_user: 50000

max_label_names_per_series: 20
        max_label_name_length: 1024
        max_label_value_length: 2048

        max_line_size: 256KB
        max_line_size_truncate: true

        reject_old_samples: true
        reject_old_samples_max_age: 168h

**Explanation:**

    ingestion_rate_mb:
        Average write rate per tenant.

    ingestion_burst_size_mb:
        Write burst capacity.

    per_stream_rate_limit:
        Write rate limit per stream.

    per_stream_rate_limit_burst:
        Burst limit per stream.

    max_streams_per_user:
        Maximum number of streams per tenant.

    max_global_streams_per_user:
        Global maximum number of streams.

    max_label_names_per_series:
        Maximum number of labels per log stream.

    max_line_size:
        Maximum size of a single log line.

    reject_old_samples:
        Reject outdated logs.

**Note:**

    Parameter names and units are based on the current Loki version.
    Helm template and test environment validation are required before modification.

### 12.4 429 Write Rate Limit Troubleshooting

You may see the following in Alloy or Loki logs:

    429 Too Many Requests
    ingestion rate limit exceeded
    per stream rate limit exceeded
    maximum active stream limit exceeded

**Troubleshoot Loki logs:**

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "429|rate limit|stream limit|ingestion"

**Troubleshoot Alloy logs:**

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "429|failed to send|rate limit"

### 12.5 When Encountering Rate Limits, Don't Just Increase Limits

**Wrong approach:**

    Increase limits indefinitely when seeing 429.

**Correct approach:**

    1. Identify which namespace/app/pod has the highest log volume.
    2. Determine if there's abnormal log flushing.
    3. Check for high cardinality labels.
    4. Check for duplicate ingestion.
    5. Check for excessive health check logs.
    6. First, govern the log sources.
    7. Then, adjust limits reasonably.

---

## Thirteen. Query Limit Governance

### 13.1 Why Limit Queries

Uncontrolled Loki queries may cause:

    High Querier CPU usage
    High Querier memory usage
    Heavy object storage read pressure
    Grafana Dashboard lag
    Ruler execution slowdown
    Other users' queries affected

**Typical dangerous queries:**

    {namespace=~".+"} |~ "(?i)error"

    {pod=~".+"} | json | level="error"

    {namespace=~".+"} | regexp "complex regular expression"

    Last 30 days query all logs

### 13.2 Common Query Limit Items

Common directions:

    Maximum query time range
    Maximum query parallelism
    Maximum returned log lines
    Maximum query series count
    Query timeout
    Query shard interval
    Maximum query lookback
    Per-tenant query rate
    Per-tenant query concurrency

### 13.3 Example limits_config

**Example:**

    loki:
      limits_config:
        max_query_length: 720h
        max_query_lookback: 720h
        max_entries_limit_per_query: 5000
        max_query_series: 5000
        max_query_parallelism: 16
        split_queries_by_interval: 30m
        query_timeout: 2m

**Explanation:**

    max_query_length:
        Maximum query time span per request.

    max_query_lookback:
        Maximum lookback time.

    max_entries_limit_per_query:
        Maximum number of returned log lines per query.

    max_query_series:
        Maximum number of series per query.

    max_query_parallelism:
        Query parallelism limit.

    split_queries_by_interval:
        Split large queries by time intervals.

    query_timeout:
        Query timeout.

**Note:**

    Field support and specific names are based on the current Loki version.
    Check official configuration and helm show values before modification.

### 13.4 Grafana Dashboard Side Limits

Dashboard should avoid:

    Default Last 7 days
    namespace Default All
    app Default All
    Multiple heavy query panels refreshing simultaneously
    Large range json / regexp queries
    Panel auto-refresh too frequently

**Recommendations:**

    Default time range:
        Last 15 minutes or Last 1 hour

    Auto-refresh:
        30s / 1m / 5m, set according to scenario

    Variables:
        namespace and app should be mandatory as much as possible

    Panels:
        Design by troubleshooting chain, avoid overcrowding

---

## Fourteen. High Cardinality Governance

### 14.1 Review of High Cardinality Fields

Not recommended as labels:

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
pod_uid
container_id full value

### 14.2 How to detect high cardinality labels

Check all labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

If you see:

    request_id
    trace_id
    user_id
    order_id
    session_id

You need to be highly cautious.

Check the number of values for a specific label:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq '.data | length'

### 14.3 How to check series count

By namespace:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' \
      | jq '.data | length'

By app:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo", app="json-log-demo"}' \
      | jq '.data | length'

### 14.4 Methods to handle high cardinality

Actions:

    1. Modify Alloy relabel configuration.
    2. Remove rules that treat high cardinality fields as labels.
    3. Keep the field in the log body.
    4. Use | json or | logfmt to query dynamic fields.
    5. Wait for old data to naturally expire.
    6. Reduce retention for error streams if necessary.
    7. Strengthen application log standards.

### 14.5 Correct way to query trace_id

Do not:

    {trace_id="abc123"}

Recommended:

    {namespace="app-prod", app="order-api"} | json | trace_id="abc123"

---

## FifteenI don't know.Multi-tenant Governance

### 15.1 What is Loki multi-tenant

Loki supports multi-tenant.

Multi-tenant is used for:

    Team isolation
    Environment isolation
    Customer isolation
    Cost statistics
    Rate limiting isolation
    Permission boundaries
    Avoiding mutual interference between teams

Loki multi-tenant requests are typically identified by HTTP Header:

    X-Scope-OrgID

### 15.2 auth_enabled=false

Common in learning environments:

    auth_enabled: false

Features:

    Simple
    No need for X-Scope-OrgID
    Suitable for experiments

Issues:

    Not suitable for multi-team production
    No tenant isolation
    Weak permission boundaries
    Can't perform strict governance by tenant

### 15.3 auth_enabled=true

Recommended for production evaluation:

    auth_enabled: true

Requests must include:

    X-Scope-OrgID: <tenant-id>

Examples:

    X-Scope-OrgID: sre
    X-Scope-OrgID: payment
    X-Scope-OrgID: ai
    X-Scope-OrgID: platform

### 15.4 Who should set X-Scope-OrgID

Not recommended to let regular users manually set it.

Recommended by:

    Authentication gateway
    Reverse proxy
    Loki Gateway
    Grafana data source configuration
    Multi-tenant proxy

Uniformly inject.

### 15.5 Multi-tenant design example

By team:

    tenant=sre
    tenant=payment
    tenant=platform
    tenant=ai

By environment:

    tenant=prod
    tenant=dev

By business line:

    tenant=business-a
    tenant=business-b

Recommendations:

    Small teams can divide by team.
    Large platforms can divide by organization/business line/customer.
    Don't over-split, increasing governance complexity.

### 15.6 Limits under multi-tenant

Each tenant can set different limits.

Examples:

    sre:
        Larger query range
        Longer retention period

    dev:
        Lower write limits
        Shorter retention period

    ai:
        Larger log lines
        Need separate control for GPU task logs

Specific implementation depends on Loki runtime_config/overrides mechanisms, and production environments need to design according to version documentation.

---

## SixteenI don't know.Security Governance: Access Entry

### 16.1 Do not expose Loki directly

Not recommended to expose Loki Service to public in production.

Not recommended:

    Expose Loki via NodePort directly
    Expose Loki via LoadBalancer without authentication
    Expose Loki via Ingress without authentication
    All /config /metrics /ready endpoints publicly accessible

### 16.2 Recommended access chain

Recommended:

    User
      ↓
    Grafana
      ↓
    Loki Gateway
      ↓
    Loki

Collection end:

    Alloy
      ↓
    Loki Gateway
      ↓
    Loki

Administrator:

    VPN / Bastion host
      ↓
    Grafana / Internal Loki entry point

### 16.3 Gateway / Ingress security

Production Gateway / Ingress should consider:

    HTTPS
    Authentication
    IP whitelist
    Internal network access
    NetworkPolicy
    Request size limits
    Timeout settings
    Access logs
    WAF or API Gateway
    Not expose sensitive management interfaces

### 16.4 /config interface risks

/config may return Loki runtime configuration.

Risks:

    Expose object storage address
    Expose bucket name
    Expose internal service name
    Expose partial sensitive configuration
    Help attackers understand system structure

In production:

    Do not open /config to regular users.
    Do not open /config to public.
    Even for temporary access during troubleshooting, be cautious about data desensitization.

### 16.5 /metrics interface risks

/metrics is used for Prometheus scraping.

Production:

    Only allow Prometheus access.
    Do not expose to the public internet.
    Can be scraped internally via ServiceMonitor / PodMonitor.

---

## Seventeen. Security Governance: Grafana Permissions

### 17.1 Grafana is the primary user entry point

Ordinary users should not directly access Loki API.

Recommended to query logs through Grafana.

Grafana can provide:

    User authentication
    Team permissions
    Folder permissions
    Dashboard permissions
    Data Source permissions
    Auditing
    SSO integration

### 17.2 Grafana Permission Design

Recommended:

    SRE:
        Can view all logs.

    Business Team A:
        Can only view Dashboard for team=a or namespace=a.

    Business Team B:
        Can only view Dashboard for team=b or namespace=b.

    Security Team:
        Can view security audit-related logs.

### 17.3 Data Source Permissions

If Grafana version supports Data Source permissions, should restrict:

    Who can query Loki
    Who can edit Loki Data Source
    Who can create Dashboard
    Who can view sensitive logs

### 17.4 Dashboard Variables are not permission boundaries

Do not mistakenly assume:

    Dashboard variables only show a specific namespace
    Users cannot query other namespaces

If users have Explore permissions and can directly write LogQL, they may still query other logs.

True permissions require:

    Loki multi-tenancy
    Grafana permissions
    Data source permissions
    Reverse proxy authentication
    Label/tenant isolation

---

## Eighteen. Security Governance: NetworkPolicy

### 18.1 Why NetworkPolicy is needed

If the cluster enables NetworkPolicy, recommend restricting Loki access relationships.

Examples:

    Only allow Alloy to write to Loki.
    Only allow Grafana to query Loki.
    Only allow Prometheus to scrape Loki /metrics.
    Only allow Loki to access MinIO.
    Prohibit ordinary business Pods from directly accessing Loki.

### 18.2 Example Targets

Allow:

    logging/alloy → logging/loki-gateway
    monitoring/grafana → logging/loki-gateway
    monitoring/prometheus → logging/loki metrics
    logging/loki → minio/minio

Deny:

    app-demo any Pod → loki API
    Unauthorized namespace → loki-gateway

### 18.3 NetworkPolicy Example Approach

Example:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: loki-gateway-ingress-policy
      namespace: logging
    spec:
      podSelector:
        matchLabels:
          app.kubernetes.io/component: gateway
      policyTypes:
        - Ingress
      ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: logging
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: monitoring
          ports:
            - protocol: TCP
              port: 80

Note:

    Specific podSelector should be adjusted according to actual Loki Gateway Pod labels.
    Not all CNI supports NetworkPolicy by default.
    Calico supports NetworkPolicy.
    Should first validate in test environment to avoid traffic disruption.

---

## Nineteen. Protection of Sensitive Information

### 19.1 Common Sensitive Information

Logs must not contain:

    Passwords
    Tokens
    Access keys
    Secret keys
    Authorization header
    Cookie
    Session
    Private keys
    Database connection string passwords
    Plaintext phone numbers
    ID numbers
    Bank card numbers
    Emails
    Addresses
    User privacy fields
    Internal credentials
    OAuth tokens
    JWT plaintext

### 19.2 Common Causes of Sensitive Information

Causes:

    Printing full request headers
    Printing full request bodies
    Printing full response bodies
    Printing full configuration objects in error logs
    DEBUG printing environment variables
    SQL connection strings without desensitization
    Third-party SDK printing tokens
    Gateway access logs printing query strings
    Applications logging Authorization headers

### 19.3 Protection Priority

Priority:

    First:
        Applications should not output sensitive information.

    Second:
        Gateway side should filter query strings / headers.

    Third:
        Collection side Alloy should desensitize.

    Fourth:
        Loki side should restrict query permissions.

    Fifth:
        Execute emergency handling and data deletion process after discovery.

### 19.4 Application-side Desensitization Example

Original:

    user login failed, password=123456

Desensitized:

    user login failed, password=******

Original:

    Authorization: Bearer eyJhbGciOi...

Desensitized: /think

Authorization: Bearer ******

Original:

    mysql://root:password123@mysql:3306/app

Desensitized:

    mysql://root:******@mysql:3306/app

### 19.5 Desensitization on the Collection Side

Desensitization risks on the collection side:

    Incorrect regular expressions will miss desensitization.
    Too broad regular expressions will mismodify logs.
    Complex regular expressions will increase Agent CPU.
    Historical sensitive logs already in Loki will not automatically disappear.

Therefore:

    Desensitization on the collection side can only be used as a supplement.

---

## Twenty, Loki Secret and Configuration Security

### 20.1 Do Not Submit Values in Plain Text

Do not submit the following to Git:

    accessKeyId
    secretAccessKey
    MinIO root password
    S3 secret
    AlertManager webhook secret
    Basic Auth password
    Grafana adminPassword
    TLS private key

### 20.2 Use Kubernetes Secret

Recommendation:

    kubectl create secret generic loki-minio-secret \
      -n logging \
      --from-literal=MINIO_ACCESS_KEY=<access-key> \
      --from-literal=MINIO_SECRET_KEY=<secret-key>

Then reference via Helm Chart's existingSecret or environment variables.

Specific fields are determined by current Chart values:

    helm show values grafana-community/loki --version <CHART_VERSION> > values-loki-default.yaml

Search:

    grep -n "existingSecret" values-loki-default.yaml

    grep -n "secret" values-loki-default.yaml

    grep -n "accessKey" values-loki-default.yaml

### 20.3 Secret Management Recommendations

Production recommendations:

    External Secrets
    SealedSecrets
    Vault
    Cloud vendor Secret Manager
    KMS
    Regular rotation
    Minimal permissions
    Save only references in Git, not plaintext

---

## Twenty-one, Loki Self Monitoring

### 21.1 Why Loki Needs to Be Monitored Itself

Loki is a log platform.

If Loki itself fails, business log queries and log alerts will be affected.

Must monitor:

    Whether Loki is Ready
    Whether writes are successful
    Whether queries are successful
    Whether Ruler executes
    Whether Compactor works
    Whether object storage is available
    Whether Gateway has 5xx errors
    Whether Alloy sends failed
    Whether MinIO is healthy

### 21.2 Key Monitoring Metrics

Recommended monitoring:

    Loki Pod Ready
    Loki Pod Restart
    Loki Gateway 5xx
    Loki Distributor write failure
    Loki Ingester active streams
    Loki Ingester memory
    Loki Querier query latency
    Loki Query timeout
    Loki Ruler rule execution failure
    Loki Compactor runtime failure
    Loki object storage request error
    Loki PVC usage
    MinIO bucket capacity
    Alloy send failure
    Alloy Pod coverage node count

### 21.3 View /metrics

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

View:

    curl -s http://127.0.0.1:3100/metrics | head

Filter:

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

### 21.4 Grafana Dashboard

Recommend to establish:

    Loki Overview
    Loki Write Path
    Loki Query Path
    Loki Ruler
    Loki Compactor
    Loki Gateway
    Alloy Agent
    MinIO Object Storage

---

## Twenty-two, Production Alert Recommendations

### 22.1 Loki Service Alerts

Recommended alerts:

    LokiPodDown
    LokiPodRestartTooMany
    LokiGateway5xxTooMany
    LokiWriteFailed
    LokiQueryFailed
    LokiQueryLatencyHigh
    LokiIngesterHighMemory
    LokiTooManyStreams
    LokiRulerFailed
    LokiCompactorFailed
    LokiObjectStorageError
    AlloySendFailed
    MinIOUnavailable
    MinIOBucketCapacityHigh

### 22.2 Alert Example Directions

Loki Pod unavailable:

    kube_pod_status_ready{namespace="logging", condition="true"} == 0

Gateway 5xx:

    nginx_ingress_controller_requests{service="loki-gateway", status=~"5.."} > 0

Alloy send failure:

    Monitor via the send failure metric in Alloy /metrics

MinIO capacity:

    Monitor bucket/disk usage via MinIO Exporter or built-in metrics

### 22.3 Log Platform Alert Principles

Log platform alerts should have high priority.

Because Loki failure affects:

    Troubleshooting
    Log alerts
    Incident review
    Audit queries
    Business team self-troubleshooting

---

## Twenty-three, Production Change Process

### 23.1 Loki values Must Be Managed by Git

Recommend to save: /think

values-loki-prod.yaml  
values-alloy-prod.yaml  
values-grafana-prod.yaml  
values-alertmanager-prod.yaml  

Prohibited:  

    Direct kubectl edit in production  
    Manual ConfigMap modification without record in production  
    Helm upgrade without backup values  
    Modification of limits without review  

### 23.2 Pre-Change Checks  

Execute:  

    helm get values loki -n logging -a > backup-values-loki-$(date +%F-%H%M).yaml  

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-prod.yaml \
      > loki-rendered-check.yaml  

Check:  

    grep -n "retention" loki-rendered-check.yaml  

    grep -n "limits_config" loki-rendered-check.yaml  

    grep -n "compactor" loki-rendered-check.yaml  

### 23.3 Post-Change Validation  

Validate:  

    helm history loki -n logging  

    kubectl get pods -n logging -o wide  

    curl -s http://127.0.0.1:3100/ready  

    curl -s http://127.0.0.1:3100/metrics | head  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq  

    kubectl logs <loki-pod-name> -n logging --tail=300 | grep -Ei "error|warn|retention|compactor|limit"  

### 23.4 Rollback  

View history:  

    helm history loki -n logging  

Rollback:  

    helm rollback loki <REVISION> -n logging  

Post-rollback validation:  

    kubectl get pods -n logging  

    curl -s http://127.0.0.1:3100/ready  

---  

## Twenty-Four, Common Issue One: Sudden Log Volume Surge  

### 24.1 Symptoms  

    Loki write volume increases  
    MinIO capacity growth accelerates  
    Alloy CPU increases  
    Query performance slows  
    Grafana Dashboard lags  
    Loki 429 errors increase  

### 24.2 Troubleshooting  

Check namespace:  

    topk(10,  
      sum by (namespace) (  
        count_over_time({namespace=~".+"}[5m])  
      )  
    )  

Check app:  

    topk(10,  
      sum by (namespace, app) (  
        count_over_time({namespace=~".+"}[5m])  
      )  
    )  

Check pod:  

    topk(10,  
      sum by (namespace, pod) (  
        count_over_time({namespace=~".+"}[5m])  
      )  
    )  

### 24.3 Resolution  

    1. Identify the logging app/pod.  
    2. Check for abnormal loops.  
    3. Verify if DEBUG is enabled.  
    4. Check for excessive health check logs.  
    5. Temporarily reduce log level.  
    6. Collect side drop if necessary.  
    7. Temporarily adjust rate limiting or isolate tenants if affecting the platform.  
    8. Optimize application log standards post-incident.  

---  

## Twenty-Five, Common Issue Two: Slow Loki Queries  

### 25.1 Possible Causes  

    Query time range too large  
    Namespace/app not restricted  
    Use of broad regular expressions  
    Global JSON parsing  
    High label cardinality  
    Object storage read latency  
    Querier resource insufficiency  
    Too many Dashboard panels  
    All variable expansion too large  

### 25.2 Troubleshooting  

Review the slow query's LogQL.  

Check if similar to:  

    {namespace=~".+"} | json | level="error"  

Optimize to:  

    {namespace="app-prod", app="order-api"} | json | __error__="" | level="error"  

### 25.3 Resolution  

    1. Narrow the time range.  
    2. Add namespace/app labels.  
    3. Avoid global JSON/regexp.  
    4. Optimize Dashboard variables.  
    5. Reduce panel count.  
    6. Increase query limits.  
    7. Optimize object storage.  
    8. Consider read expansion and caching for large-scale scenarios.  

---  

## Twenty-Six, Common Issue Three: Logs Not Deleted by Retention  

### 26.1 Possible Causes  

    Compactor not enabled  
    Retention_enabled not enabled  
    Retention_period not configured  
    Retention_stream not matched  
    Delete_request_store configuration error  
    Object storage deletion permission insufficient  
    MinIO lifecycle conflicts with Loki retention  
    Not yet reached deletion delay time  
    Current data still within retention period  

### 26.2 Troubleshooting  

Check values:  

    helm get values loki -n logging -a | grep -n "retention" -A 50  

Check logs:  

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "compactor|retention|delete|error|permission|bucket"  

Check MinIO:  

    mc find local/loki-chunks | head  

### 26.3 Resolution /think

1. Confirm Compactor is enabled.
2. Confirm retention_enabled=true.
3. Confirm retention_period is correct.
4. Confirm object storage has delete permissions.
5. Confirm delete_request_store is correct.
6. Wait for Compactor cycle to execute.
7. Do not use object storage lifecycle to delete prematurely.

---

## 27. Common Fault Four: Write 429

### 27.1 Phenomenon

Alloy logs:

    failed to send batch
    429 Too Many Requests

Loki logs:

    ingestion rate limit exceeded
    per stream rate limit exceeded
    maximum active stream limit exceeded

### 27.2 Troubleshooting

Check Alloy:

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "429|rate|failed"

Check Loki:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "429|rate limit|stream"

Find app with highest log volume:

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 27.3 Resolution

    1. First address abnormal log sources.
    2. Check for high cardinality labels.
    3. Check for duplicate collection.
    4. Check if per_stream is too large for single stream.
    5. Reasonably adjust limits.
    6. Split high traffic streams when necessary, but avoid introducing high cardinality labels.
    7. Consider expanding Loki write components for large scale scenarios.

---

## 28. Common Fault Five: Sensitive Information Entering Loki

### 28.1 Phenomenon

Logs contain:

    password
    token
    Authorization
    Cookie
    secret
    access_key
    phone number
    ID card
    private key

### 28.2 Emergency Handling

Steps:

    1. Immediately confirm the scope of impact.
    2. Pause or modify application log output.
    3. Temporarily desensitize collection rules.
    4. Restrict Grafana / Loki query permissions.
    5. Evaluate whether to delete historical logs.
    6. Rotate leaked keys or tokens.
    7. Record security incident.
    8. Review log standards and code review processes.

### 28.3 Notes

Deleting Loki historical data is not simply deleting files.

Must combine:

    Loki Delete API
    Compactor
    retention
    object storage
    compliance requirements

Must execute carefully in production.

---

## 29. Production Security Baseline

### 29.1 Loki Server

Requirements:

    [ ] Do not expose Loki API to public internet
    [ ] /config not open to regular users
    [ ] /metrics only allow Prometheus access
    [ ] Use HTTPS
    [ ] Use Gateway / Ingress as unified entry
    [ ] Enable authentication or reverse proxy authorization in production
    [ ] Enable multi-tenancy or equivalent isolation for multi-team scenarios
    [ ] Configure NetworkPolicy
    [ ] Configure limits_config
    [ ] Configure retention
    [ ] Loki values should be managed in Git

### 29.2 Alloy Collector

Requirements:

    [ ] Only collect necessary namespaces
    [ ] Avoid duplicate collection
    [ ] Do not collect low-value high-frequency logs
    [ ] Do not use high cardinality fields as labels
    [ ] RBAC with minimal permissions
    [ ] Resource requests/limits
    [ ] Monitor collector itself
    [ ] Configuration changes should go through Git

### 29.3 Grafana

Requirements:

    [ ] Integrate with unified authentication
    [ ] Control permissions by Team / Folder
    [ ] Restrict data source edit permissions
    [ ] Restrict Explore permissions
    [ ] Dashboard default time range should be reasonable
    [ ] Do not default to global queries
    [ ] Dashboard JSON should be managed in Git
    [ ] Access to sensitive logs should have permission control

### 29.4 MinIO / S3

Requirements:

    [ ] Loki should use dedicated user
    [ ] Minimum permissions for bucket access
    [ ] Do not use root user
    [ ] HTTPS
    [ ] Multi-node high availability
    [ ] Capacity monitoring
    [ ] Lifecycle policy longer than Loki retention
    [ ] Regular key rotation
    [ ] Bucket permission auditing

---

## 30. Hands-on Tasks

### 30.1 Task 1: Statistic Top namespace for log volume

Execute:

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Acceptance:

    [ ] Can see namespace with highest log volume

### 30.2 Task 2: Statistic Top app for log volume

Execute:

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Acceptance:

    [ ] Can see app with highest log volume

### 30.3 Task 3: Configure retention

Modify values:

    loki:
      limits_config:
        retention_period: 168h

      compactor:
        working_directory: /var/loki/compactor
        compaction_interval: 10m
        retention_enabled: true
        retention_delete_delay: 2h
        delete_request_store: s3

Execute: /think

helm template loki grafana-community/loki \
  --namespace logging \
  --version <CHART_VERSION> \
  -f values-loki-prod-governance.yaml \
  > loki-governance-rendered.yaml

helm upgrade loki grafana-community/loki \
  --namespace logging \
  --version <CHART_VERSION> \
  -f values-loki-prod-governance.yaml

Acceptance:

  [ ] helm template has no errors
  [ ] helm upgrade is successful
  [ ] Loki Pod Running
  [ ] No compactor/retention errors in Loki logs

### 30.4 Task Four: Configure limits_config

Add example:

  loki:
    limits_config:
      ingestion_rate_mb: 8
      ingestion_burst_size_mb: 16
      per_stream_rate_limit: 5MB
      per_stream_rate_limit_burst: 20MB
      max_entries_limit_per_query: 5000
      max_query_series: 5000
      max_query_length: 720h
      reject_old_samples: true
      reject_old_samples_max_age: 168h

Acceptance:

  [ ] Configuration can be rendered
  [ ] Loki can start normally
  [ ] /ready returns ready
  [ ] Querying and writing are normal

### 30.5 Task Five: Check High Cardinality Labels

Execute:

  curl -s "http://127.0.0.1:3100/loki/api/v1/labels"" | jq

Check if the following appear:

  request_id
  trace_id
  user_id
  order_id
  session_id

Acceptance:

  [ ] No obvious high cardinality labels found
  [ ] If found, can trace the source and develop a modification plan

### 30.6 Task Six: Check for Sensitive Information

Query:

  {namespace="app-demo"} |~ "(?i)password|token|authorization|cookie|secret|access_key"

Acceptance:

  [ ] No sensitive information found
  [ ] If found, can develop remediation plans for application-side and collection-side

### 30.7 Task Seven: Check Loki's Own Metrics

Execute:

  curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

Acceptance:

  [ ] Can access Loki metrics
  [ ] Can include Loki itself in Prometheus monitoring plan

---

## Thirty-One, Acceptance Checklist

After completing this document, confirm:

  [ ] Understand Loki production governance goals
  [ ] Can statistics namespace log volume
  [ ] Can statistics app log volume
  [ ] Can statistics pod log volume
  [ ] Can identify high log volume sources
  [ ] Understand application-side log governance has the highest priority
  [ ] Understand Alloy collection-side filtering function
  [ ] Understand Loki retention depends on Compactor
  [ ] Can configure global retention
  [ ] Can configure differentiated retention
  [ ] Understand object storage lifecycle cannot be shorter than Loki retention
  [ ] Can configure write rate limiting
  [ ] Can configure query limits
  [ ] Can troubleshoot 429 write rate limiting
  [ ] Can troubleshoot slow queries
  [ ] Can check high cardinality labels
  [ ] Understand multi-tenancy X-Scope-OrgID
  [ ] Understand auth_enabled=false is only suitable for learning environments
  [ ] Understand auth_enabled=true's production significance
  [ ] Can design Loki access entry security policy
  [ ] Can design Grafana permission governance policy
  [ ] Can understand NetworkPolicy's protective role for Loki
  [ ] Can identify sensitive information log risks
  [ ] Can develop Loki Secret management standards
  [ ] Can list Loki's own monitoring metrics
  [ ] Can develop Loki production security baseline

---

## Thirty-Two, Common Misconceptions

### 32.1 Misconception One: More Logs Are Better

Error.

More logs mean higher costs, slower queries, and more noise.

Correct:

  Only valuable logs should be retained.
  High-frequency, valueless logs should be governed.

### 32.2 Misconception Two: Retention is Immediately Deleted Once Configured

Error.

Retention depends on Compactor execution, with cycles and delays.

Object deletion is not real-time.

### 32.3 Misconception Three: Object Storage Lifecycle Can Replace Loki Retention

Not recommended.

Early deletion of chunks in object storage may cause inconsistency between Loki index and chunks.

### 32.4 Misconception Four: Seeing 429 Means Increasing Limits

Error.

First check if the log source is abnormal, high cardinality, or duplicate collection.

### 32.5 Misconception Five: Slow Dashboard Queries Are Grafana's Problem

Not necessarily.

It could be due to large query ranges, heavy regex, slow object storage, or poor label design.

### 32.6 Misconception Six: Grafana Variables Can Achieve Permission Isolation

Error.

Variables are just query helpers, not security boundaries.

Permission isolation requires Grafana permissions, data source permissions, Loki multi-tenancy, and reverse proxy authentication mechanisms.

### 32.7 Misconception Seven: Sensitive Information Can Enter Logs and Be Deleted Later

Error.

Once sensitive information enters the log system, it spreads to object storage, backups, Dashboards, alerts, and exported files.

Prevention should start from the source.

---

## Thirty-Three, Production Deployment Recommendations

### 33.1 Phase One: Basic Governance

Complete:

  Loki connected to object storage
  Configure retention
  Configure limits_config
  Alloy only collects necessary namespaces
  Grafana Dashboard default time range control
  Loki's own metrics connected to Prometheus

### 33.2 Phase Two: Standardized Governance

Complete:

# Application Log Format Standards
## Label Standards
## Sensitive Information Standards
## Dashboard Standards
## Alert Rule Standards
## Runbook Standards
## values Git Management

### 33.3 Third Stage: Multi-team Governance

Completed:

    Multi-tenant Planning
    Grafana Permission Isolation
    Team / Folder Permissions
    Data Source Permissions
    Tenant-level Limits
    Tenant-level Retention
    Tenant-level Cost Statistics

### 33.4 Fourth Stage: Platform Governance

Completed:

    Loki High Availability
    Read/Write Scalability
    Caching
    Query Acceleration
    Log Cost Analysis
    Automated Reporting
    Log Anomaly Detection
    Integration with CMDB / Permission Systems

---

## 34. Summary

This document completes the core content of Loki production governance.

Loki production governance is not a single-point configuration, but an entire system:

    Log Volume Governance:
        Control low-value logs, identify high-volume log sources.

    Retention:
        Control retention period through Compactor.

    Object Storage Governance:
        MinIO/S3 permissions, capacity, lifecycle, security.

    Write Limiting:
        Prevent single application or tenant from overwhelming Loki.

    Query Limiting:
        Prevent wide-range queries from slowing down the system.

    Label Governance:
        Control high cardinality, ensure query efficiency.

    Multi-tenant Governance:
        Use X-Scope-OrgID for tenant isolation.

    Security Governance:
        Control access entry points, Grafana permissions, NetworkPolicy, Secret.

    Sensitive Information Governance:
        Prohibit sensitive information from entering logs at the application level.

    Self-monitoring:
        Loki, Alloy, MinIO, Grafana, AlertManager must all be included in monitoring.

Core principles of production Loki:

    Logs are not better with more volume.
    Labels are not better with more quantity.
    Queries are not better with wider scope.
    Retention is not better with longer duration.
    Alerts are not better with more quantity.
    Permissions cannot rely solely on Dashboard variables.
    Sensitive information cannot depend on post-hoc deletion.
    Loki itself must also be monitored.

Next article will enter:

    13-Loki Performance and High Availability Practical Guide: Simple-Scalable Mode Introduction

Key learning points:

    read/write/backend roles
    Differences between single-node mode and split mode
    Query path expansion
    Write path expansion
    backend/compactor/ruler roles
    Object storage dependencies
    High availability and performance tuning basics
    Historical value and version evolution notes of Simple Scalable

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Log retention:
  https://grafana.com/docs/loki/latest/operations/storage/retention/

- Storage:
  https://grafana.com/docs/loki/latest/configure/storage/

- Loki configuration:
  https://grafana.com/docs/loki/latest/configure/

- Request validation and rate limits:
  https://grafana.com/docs/loki/latest/operations/request-validation-rate-limits/

- Manage tenant isolation:
  https://grafana.com/docs/loki/latest/operations/multi-tenancy/

- Manage authentication:
  https://grafana.com/docs/loki/latest/operations/authentication/

- Query Loki:
  https://grafana.com/docs/loki/latest/query/

- Query best practices:
  https://grafana.com/docs/loki/latest/query/bp-query/

- Loki HTTP API:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- Grafana Alloy Documentation:
  https://grafana.com/docs/alloy/latest/

- Collect Kubernetes logs and forward them to Loki:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- MinIO Documentation:
  https://min.io/docs/minio/kubernetes/upstream/

- Kubernetes NetworkPolicy:
  https://kubernetes.io/docs/concepts/services-networking/network-policies/

- Kubernetes Secrets:
  https://kubernetes.io/docs/concepts/configuration/secret/