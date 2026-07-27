# 12-Loki Production Governance in Action: Log Volume, Retention Period, Throttling, and Security

## Document Description

This article is the twelfth installment in the dedicated study series on Loki, focusing on its governance capabilities in a production environment. It covers topics such as log volume management, retention period settings, write throttling, query limitations, label base control, multi-tenant isolation, secure access, sensitive information protection, object storage integration, and self-monitoring of Loki.

Previous articles have covered:

    01-Loki Basics and Experimental Environment Setup
    02-Loki Architecture and Component Responsibilities in Practice
    03-Loki Deployment Modes Comparison and Experiment Selection
    04-Loki Single-Node Mode Deployment Using Helm
    05-Loki Integration with MinIO Object Storage
    06-Grafana-Alloy for Collecting Kubernetes Pod Logs
    07-Loki Label Design and High-Base Number Issues
    08-LogQL Basic Queries: Retrieving Namespace, Pod, and Container Logs
    09-Advanced LogQL Queries: json, logfmt, regexp, and unwrap
    10-Grafana Integration with Loki for Log Dashboards
    11-Loki Log Alerts: Integrating Ruler and AlertManager

Article 04 covered the deployment of Loki in a single-node mode. Article 05 discussed integrating Loki with MinIO object storage. Article 06 explained how Alloy collects Kubernetes Pod logs. Article 07 explored label management and high-base number issues. Articles 08 and 09 focused on LogQL queries. Article 10 demonstrated the creation of Grafana dashboards. Article 11 discussed how to use Loki Ruler for log alerts.

This article now enters the stage of production governance.

It addresses the following key questions:

- Why can't production-level Loki focus solely on "whether logs can be retrieved"?
- Why is it necessary to manage log volume?
- How to determine which namespace/app/pod generates the most logs?
- How to handle high-frequency logs such as DEBUG, health check, metrics, and access logs?
- How does Loki's retention mechanism work?
- What role does the Compactor play in the log retention process?
- How to configure global retention settings?
- How to set different retention periods for different namespaces/applications?
- Why can't the object storage lifecycle be shorter than Loki's retention period?
- How to configure write throttling?
- How to limit query operations?
- How to prevent extensive queries from impacting Loki's performance?
- How to control high-base number labels?
- How to design a multi-tenant system?
- What is the difference between setting `auth_enabled=false` and `authenabled=true` for Loki?
- How to implement security controls using gateways, reverse proxies, Grafana permissions, and NetworkPolicy?
- How to prevent sensitive information from being included in logs?
- How to establish a production security baseline for Loki?
- How to monitor and set up alerts for Loki itself?

This article is not merely a list of configuration parameters but provides practical guidance based on real-world production scenarios.

---

## Tags

#Loki #Grafana #Log Governance #Retention #Compactor #LimitsConfig #RateLimit #Multi-tenant #Security Baseline #Sensitive Information Protection #SRE #Kubernetes #MinIO #Object Storage #Observability

---

## Recommended Reading Path

Recommended reading path:

    10-Logs/02-Loki/12-Loki Production Governance in Action: Log Volume, Retention Period, Throttling, and Security.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Understand the core objectives of Loki production governance.
    2. Calculate log volumes at the namespace/app/pod levels.
    3. Identify sources of high log volumes.
    4. Manage unnecessary high-frequency logs.
    5. Comprehend how Loki's retention mechanism functions.
    6. Understand the role of the Compactor in retention processes.
    7. Configure global log retention periods.
    8. Set different retention periods for various streams.
    9. Recognize the relationship between object storage lifecycle and Loki retention.
    10. Configure write throttling for Loki.
    11. Limit query operations.
    12. Understand the reasons behind 429 write throttling errors.
    13. Diagnose issues related to slow or excessive queries.
    14. Manage high-base number labels.
    15. Comprehend Loki's multi-tenant architecture.
    16. Plan production access methods with `auth_enabled=true`.
    17. Design permission controls for Loki Gateway/Ingress/Grafana.
    18. Identify potential risks of sensitive information in logs.
    19. Establish a production security baseline### 5.1 Statistically Analyzing Log Volumes by Namespace

LogQL:

    sum by (namespace) (
      count_over_time({namespace=~".+"}[5m])
    )

Purpose:

    To identify which namespace generates the most logs.

Production Notes:

    This query covers a wide range of data.
    It is recommended to use it during off-peak hours or within a specific cluster/environment.

A more preferable option:

    sum by (namespace) (
      count_over_time({cluster="k8s-lab"}[5m])
    )

### 5.2 Statistically Analyzing Log Volumes by App

LogQL:

    sum by (namespace, app) (
      count_over_time({namespace=~".+"}[5m])
    )

If a specific namespace is specified:

    sum by (app) (
      count_over_time({namespace="app-demo"}[5m])
    )

### 5.3 Statistically Analyzing Log Volumes by Pod

LogQL:

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace="app-demo"}[5m])
      )
    )

Purpose:

    To identify the Pod that generates the most logs.

### 5.4 Statistically Analyzing Log Volumes by Container

LogQL:

    topk(10,
      sum by (namespace, pod, container) (
        count_over_time({namespace="app-demo"}[5m])
      )
    )

Purpose:

    To determine whether the primary container, sidecar, or proxy container in a multi-container Pod generates the most logs.

### 5.5 Statistically Analyzing Error Logs

LogQL:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 5.6 Statistically Analyzing 5xx Status Codes Logs

Structured JSON:

    sum by (namespace, app, status) (
      count_over_time(
        {namespace="app-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    )

Nginx Text Logs:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo", app="nginx-demo"}
          |~ " 5[0-9][0-9] " [5m]
      )
    )

### 5.7 Statistically Analyzing Timeout Logs

LogQL:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)timeout|timed out|deadline exceeded" [5m]
      )
    )

---

## VI. Log Volume Governance: Application Side

### 6.1 The Application Side Is the Primary Point of Governance

The most effective governance measures should be implemented at the application level.

Reason:

    If applications do not generate unnecessary logs, it will significantly reduce the workload on subsequent systems.
    If applications already produce a large volume of logs, the collection and processing systems can only handle them passively.

### 6.2 Production Log Level Specifications

Recommendations:

    dev:
        DEBUG levels can be used appropriately.

    test:
        Mainly use INFO levels; DEBUG can be enabled temporarily when necessary.

    staging:
        Similar to production, default to INFO or WARN levels.

    prod:
        Default to INFO/WARN/ERROR levels.
        The use of DEBUG should be restricted and have a set expiration time.

### 6.3 Contents That Should Not Be Reported

It is not recommended to report:

    Logs for each successful health check
    Logs for each metrics request
    Large amounts of progress logs in loops
    Complete request bodies
    Complete response bodies
    Large JSON objects
    Large array contents
    Plain-text SQL parameters
    Duplicate exception stacks
    User privacy information
    Tokens/passwords/secrets

### 6.4 Recommended Structured Field Names

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

    {"timestamp":"2026-05-01T12:00:00+08:00","level":"error","service":"order-api","trace_id":"abc123","method":"POST","route":"/api/orders/{id}","status":500,"duration_ms":1300,"error_type":"database_error","msg":"Database connection failed"}

### 6.5 Using route Instead of full path

It is not recommended to use:

    path="/api/orders/100001"
    path="/api/orders/100002"
    path="/api/orders## VIII. Basics of Log Retention Period

### 8.1 What is Retention?

Retention refers to the period for which logs are kept.

For example:

    Development logs are retained for 7 days.
    Test logs are retained for 15 days.
    Production logs are retained for 30 days.
    Security logs are retained for 90 days.

After the retention period expires, the logs should be cleaned up.

### 8.2 Loki Retention Depends on Compactor

In the current Loki architecture, retention is usually handled by the Compactor.

The Compactor is responsible for:

    Compressing indexes.
    Managing retention settings.
    Marking expired data.
    Deleting expired chunks.
    Cleaning up expired objects in object storage.

### 8.3 Default Does Not Mean Automatic Cleanup

Do not assume that Loki will automatically delete logs by default.

In production environments, the following settings must be explicitly configured:

    compactor.retention_enabled
    limits_config(retention_period)
    delete_request_store
    retention_delete_delay

If retention is not enabled, logs may be retained indefinitely until the storage is manually cleaned up or the object storage lifecycle takes effect.

### 8.4 Retention and Object Storage Lifecycle

If MinIO/S3 has a lifecycle configuration, please note:

    The object storage lifecycle should not be shorter than the Loki retention period.

Incorrect configuration:

    Loki retention = 30 days
    MinIO lifecycle = 7 days

Consequences:

    The Loki index may still consider the logs to exist.
    However, the chunks might have already been deleted by object storage.
    Querying historical logs may result in errors or missing data.

Recommended configuration:

    MinIO lifecycle > Loki retention

For example:

    Loki retention = 30 days
    MinIO lifecycle = 35 days or longer

---

## IX. Configuring Global Retention Settings

### 9.1 Back up Current Values

Perform the following command:

    helm get values loki -n logging -a > backup-values-loki-before-retention.yaml

### 9.2 View Current Loki Values

Run:

    helm get values loki -n logging -a > values-loki-current.yaml

Search for the following fields:

    grep -n "compactor" values-loki-current.yaml
    grep -n "retention" values-loki-current.yaml
    grep -n "limits_config" values-loki-current.yaml

### 9.3 Sample Configuration

Create or modify the file:

    values-loki-prod-governance.yaml

Add or confirm the following sections based on the existing Loki settings:

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
        Logs will be retained for 7 days globally.

    retentionenabled: true
        The Compactor retention feature is enabled.

    retention_delete_delay: 2h
        A delay of 2 hours is set before deleting logs to avoid synchronization issues between indexes and chunk deletion.

    delete_request_store: s3
        Object storage is used for storing delete requests.
        If a filesystem is used, adjustments may be required based on the actual configuration.

Note:

    This is just a sample configuration.
    Field names may vary depending on the version of the Loki Helm Chart.
    Always verify the settings using `helm show values` and `helm template` before making any changes.

### 9.4 Checking Helm Template Configuration

Execute the following command:

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-prod-governance.yaml \
      > loki-governance-rendered.yaml

Check for the following fields in the generated file:

    grep -n "retention" loki-governance-rendered.yaml
    grep -n "compactor" loki-governance-rendered.yaml
    grep -n "retention_period" loki-governance-rendered.yaml

### 9.5 Upgrading Loki

Run the following command to update Loki to the specified version:

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-prod-governance.yaml

Verify the installation by running:

    helm history loki -n logging
    kubectl get pods -n logging -o wide

### 9.6 Viewing Loki Logs

Use the following command to view logs from a specific Loki pod:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "comp      -n minio \
      -- sh

Setting an alias:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

Viewing buckets:

    mc ls local

Viewing Loki chunks:

    mc ls local/loki-chunks

Recursively viewing:

    mc find local/loki-chunks | head

### 11.2 Checking Bucket Size

You can use:

    mc du local/loki-chunks

If the mc version supports recursive statistics:

    mc du --recursive local/loki-chunks

Note:

    Commands may vary slightly depending on the mc version.
    If a command is not available, you can check the capacity through the MinIO Console.

### 11.3 Attention to MinIO Lifecycle Management

In production, if you set up a MinIO lifecycle:

    Ensure that the lifecycle duration is longer than the Loki retention period.

For example:

    Loki retention:
        30 days

    MinIO lifecycle:
        35 or 45 days

This prevents MinIO from deleting chunks that are still needed by Loki prematurely.

### 11.4 MinIO Permission Management

In production, it is not recommended to use the root user for managing Loki.

Suggestions:

    Create a dedicated user for Loki.
    Generate a dedicated access key for Loki.
    Authorize only specific operations such as loki-chunks, loki-ruler, and loki-admin.
    Do not grant MinIO administrative privileges.
    Regularly rotate keys.
    Ensure that secrets are not stored in plain text in Git.

### 11.5 MinIO Availability Management

In production, MinIO should not be configured with a single copy.

Recommendations:

    Use multiple nodes.
    Implement Erasure Coding.
    Monitor disk and capacity usage.
    Track API latency.
    Monitor 4xx and 5xx responses.
    Check certificate validity periods.
    Conduct regular backup and recovery tests.

---

## Chapter Twelve: Write Throttling Management

### 12.1 Why Limit Writing

If an application suddenly generates a large amount of logs, it can affect the entire Loki system.

Write throttling is used to protect:

    The Loki Ingester
    The Distributor
    Object storage
    The network
    Memory resources
    Query performance
    Other users

### 12.2 Common Write Limitations

Common limitations include:

    Per-tenant write rate
    Per-tenant burst write volume
    Single stream write rate
    Single stream burst write volume
    Maximum number of streams
    Maximum size of a single log
    Limit on the number of labels
    Length restrictions for label names and values
    Rejection of old logs
    Rejection of logs scheduled for future times

### 12.3 Example limits_config

Example configuration:

    loki:
      limits_config:
        ingestion_rate_mb: 8
        ingestion_burst_size_mb: 16

        per_stream_rate_limit: 5MB
        per_stream_rate_limit_burst: 20MB

        max_streams_per_user: 10000
        max_globalStreams_per_user: 50000

        max_label_names_per_series: 20
        max_label_name_length: 1024
        max_label_value_length: 2048

        max_line_size: 256KB
        max_line_size_truncate: true

        reject_old_samples: true
        reject_old_samples_max_age: 168h

Explanation:

    ingestion_rate_mb:
        Average write rate per tenant.

    ingestion_burst_size_mb:
        Maximum burst size for writing.

    per_stream_rate_limit:
        Limit on the write rate for a single stream.

    per_stream_rate_limit_burst:
        Limit on the burst write volume for a single stream.

    max_streams_per_user:
        Maximum number of streams per tenant.

    max_globalstreams_per_user:
        Maximum number of streams globally.

    max_label_names_per_series:
        Maximum number of labels per log series.

    max_line_size:
        Maximum size of a single log line.

    reject_old_samples:
        Rejection of outdated logs.

Note:

    Parameter names and units are based on the current Loki version.
    Always verify changes in the Helm template and test environment before applying them.

### 12.4 Troubleshooting 429 Write Throttling Issues

You may encounter errors such as:

    429 Too Many Requests
    ingestion rate limit exceeded
    per stream rate limit exceeded
    maximum active stream limit exceeded

To troubleshoot, check the Loki logs:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "429|rate limit|stream limit|ingestion"

For Alloy logs:

    k## Section XIV: Managing High Cardinality Data

### 14.1 Review of High Cardinality Fields

It is not recommended to use the following fields as labels:

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
    container_id (in their complete form)

### 14.2 How to Identify High Cardinality Labels

To view all labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

If you see fields such as:

    request_id
    trace_id
    user_id
    order_id
    session_id,

you should be highly cautious.

To check the number of values for a specific label:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq '.data | length'

### 14.3 Checking the Number of Series

By namespace:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match="{namespace="app-demo"}' \
      | jq '.data | length'

By app:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match="{namespace="app-demo", app="json-log-demo"}' \
      | jq '.data | length'

### 14.4 Methods for Handling High Cardinality Data

Options include:

    1. Adjusting the Alloy relabel configuration.
    2. Removing rules that use these fields as labels.
    3. Keeping the fields in the log text.
    4. Using commands like | json or | logfmt to query dynamic fields.
    5. Letting old data expire naturally.
    6. If necessary, shortening the retention period for error streams.
    7. Strengthening application logging standards.

### 14.5 Correct Way to Query trace_id

Do not use:

    {trace_id="abc123"}

It is recommended to use:

    {namespace="app-prod", app="order-api"} | json | trace_id="abc123"

---

## Section XV: Multi-Tenant Management

### 15.1 What is Loki Multi-Tenancy

Loki supports multi-tenancy.

Multi-tenancy is used for:

    Team isolation
    Environment separation
    Customer differentiation
    Cost tracking
    Throttling control
    Defining permission boundaries
    Preventing interference between different teams

In Loki, multi-tenant requests are typically identified through the HTTP Header:

    X-Scope-OrgID

### 15.2 Using auth_enabled=false in Learning Environments

In learning environments, it is common to use:

    authenabled: false

This setup has the following advantages:

    Simplicity
    No need for X-Scope-OrgID
    Suitable for experimentation

However, it has limitations:

    Not suitable for multi-team production environments
    Lack of tenant isolation
    Weak permission controls
    Inability to enforce strict governance by tenant

### 15.3 Using authenabled=true in Production Environments

For production use, it is recommended to evaluate whether to use:

    authenabled: true

In this case, requests must include:

    X-Scope-OrgID: <tenant-id>

Examples:

    X-Scope-OrgID: sre
    X-Scope-OrgID: payment
    X-Scope-OrgID: ai
    X-Scope-OrgID: platform

### 15.4 Who Should Set the X-Scope-OrgID

It is not recommended that regular users set this manually.

It is better to have it set by:

    Authentication gateways
    Reverse proxies
    Loki Gateways
    Grafana data source configurations
    Multi-tenant proxies

This ensures consistent and unified injection of the value.

### 15.5 Examples of Multi-Tenant Design

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

It is generally recommended that:

    Small and medium-sized teams can be organized by team.
    Large platforms can be divided by organization, business line, or customer.
    Avoid excessive segmentation to prevent increasing governance complexity.

### 15.6 Limits in a Multi-Tenant Environment

Each tenant can have different settings### 18.3 Example Approach for NetworkPolicy

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

    The specific podSelector should be adjusted according to the actual labels of the Loki Gateway Pod.
    Not all CNI solutions enable NetworkPolicy by default.
    Calico supports NetworkPolicy.
    It is recommended to test this in a staging environment first to avoid disrupting traffic.

---

## Chapter Nineteen: Protection of Sensitive Information

### 19.1 Common Types of Sensitive Information

The following should not appear in logs:

    Passwords
    Tokens
    Access keys
    Secret keys
    Authorization headers
    Cookies
    Sessions
    Private keys
    Database connection string passwords
    Plain-text phone numbers
    Identity card numbers
    Bank card numbers
    Email addresses
    Physical addresses
    User privacy fields
    Internal credentials
    OAuth tokens
    JWT payloads in plaintext

### 19.2 Common Reasons for the Presence of Sensitive Information

Reasons:

    Printing the entire request header
    Printing the entire request body
    Printing the entire response body
    Configuring abnormal log printing to include objects
    Printing environment variables in DEBUG mode
    Failing to mask SQL connection strings
    Third-party SDKs printing tokens
    Gateways printing query strings in access logs
    Applications including Authorization headers in logs

### 19.3 Priority Levels for Protection

Priority levels:

    First:
        Applications should not output sensitive information.

    Second:
        Gateways should filter out query strings and headers.

    Third:
        The Alloy data collection tool should mask sensitive information.

    Fourth:
        Loki should restrict query access rights.

    Fifth:
    Emergency procedures and data deletion processes should be implemented upon detection of sensitive information.

### 19.4 Examples of Data Masking on the Application Side

Original:

    user login failed, password=123456

Masked:

    user login failed, password=******

Original:

    Authorization: Bearer eyJhbGciOi...

Masked:

    Authorization: Bearer ******

Original:

    mysql://root:password123@mysql:3306/app

Masked:

    mysql://root:******@mysql:3306/app

### 19.5 Precautions for Data Masking on the Collection Side

Risks of data masking on the collection side:

    Incorrect regular expressions may result in incomplete masking.
    Overly broad regular expressions could accidentally modify logs.
    Complex regular expressions may increase the CPU load on agents.
    Historical sensitive logs that have already been stored in Loki will not be automatically removed.

Therefore:

    Data masking on the collection side should be used as a supplementary measure.

---

## Chapter Twenty: Security of Loki Secrets and Configurations

### 20.1 Do Not Submit Values in Plain Text

Do not include the following in Git repositories:

    accessKeyId
    secretAccessKey
    MinIO root password
    S3 secret key
    AlertManager webhook secret key
    Basic Auth passwords
    Grafana adminPassword
    TLS private keys

### 20.2 Using Kubernetes Secrets

It is recommended to create a Secret using the following command:

    kubectl create secret generic loki-minio-secret \
      -n logging \
      --from-literal=MINIO_ACCESS_KEY=<access-key> \
      --from-literal=MINIO_SECRET_KEY=<secret-key>

Then, reference this Secret through existingSecrets in Helm Charts or environment variables.

Specific field values should be determined based on the current Chart configuration:

    helm show values grafana-community/loki --version <CHART_VERSION> > values-loki-default.yaml

To search for relevant fields:

    grep -n "existingSecret" values-loki-default.yaml
    grep -n "secret" values-loki-default.yaml
    grep -n "accessKey" values-loki-default.yaml

### 20.3 Best Practices for Secret Management

In production environments, consider using the following methods:

    External Secrets
    SealedSecrets
    Vault
    Cloud providers' Secret Managers
    KMS
    Regularly rotate secrets.
    Apply the principle of least privilege.
    Only store references to secrets in Git, not the actual secrets themselves.

---

## Chapter Twenty-One: Internal Monitoring of Loki

### 2Monitor bucket/disk usage through MinIO Exporter or built-in metrics.

### 22.3 Log Platform Alarm Principles

Log platform alarms should have high priority.

Because Loki failures can affect:

    Fault troubleshooting
    Log alerts
    Incident recovery
    Audit queries
    Self-troubleshooting by business teams

---

## Chapter 23: Production Change Process

### 23.1 Loki Values Must Be Managed via Git

It is recommended to save:

    values-loki-prod.yaml
    values-alloy-prod.yaml
    values-grafana-prod.yaml
    values-alertmanager-prod.yaml

Prohibited:

    Direct kubectl edit in production
    Direct manual modification of ConfigMap in production (not logged)
    No backup of values during Helm upgrades
    Changes to limits without review

### 23.2 Pre-Change Check

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

### 23.3 Post-Change Verification

Verify:

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

Verify after rollback:

    kubectl get pods -n logging

    curl -s http://127.0.0.1:3100/ready

---

## Chapter 24: Common Fault 1: Sudden Surge in Log Volume

### 24.1 Symptoms

    Increased writing to Loki
    Faster growth of MinIO capacity
    Higher Alloy CPU usage
    Slower queries
    Lagging Grafana dashboards
    Increase in Loki 429 errors

### 24.2 Troubleshooting

Check by namespace:

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Check by app:

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Check by pod:

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 24.3 Solutions

    1. Identify the app/pod generating excessive logs.
    2. Check for any abnormal loops.
    3. Verify if DEBUG logging is enabled.
    4. Check if there are too many health check logs.
    5. Temporarily reduce the log level.
    6. Consider dropping unnecessary data collection if needed.
    7. If the platform is affected, temporarily adjust throttling or isolate the tenant.
    8. Optimize application logging standards afterward.

---

## Chapter 25: Common Fault 2: Slow Loki Queries

### 25.1 Possible Causes

    Too wide query time range
    Unspecified namespace/app
    Use of extensive regular expressions
    Global JSON parsing
    High number of labels
    Slow object storage reading
    Insufficient Querier resources
    Too many dashboards
    Excessive expansion of the "All" variable

### 25.2 Troubleshooting

Check the slow query LogQL.

Optimize if it looks similar to:

    {namespace=~".+"} | json | level="error"

Change it to:

    {namespace="app-prod", app="order-api"} | json | __error__="" | level="error"

### 25.3 Solutions

    1. Narrow down the3. Temporarily desensitize collection rules.
4. Restrict query permissions for Grafana / Loki.
5. Evaluate whether to delete historical logs.
6. Rotate leaked keys or tokens.
7. Record security incidents.
8. Review log standards and code review processes.

### 28.3 Notes

Deleting historical Loki data is not simply about deleting files.

It requires consideration of:

    The Loki deletion API
    Compactor
    Retention settings
    Object storage
    Compliance requirements

This must be done carefully in production environments.

---

## Chapter Twenty-Nine: Production Security Baselines

### 29.1 Loki Server

Requirements:

    [ ] Do not expose the Loki API publicly.
    [ ] The /config directory should not be accessible to general users.
    [ ] Only allow Prometheus to access the /metrics directory.
    [ ] Use HTTPS.
    [ ] Use a Gateway / Ingress for unified access control.
    [ ] Enable authentication in production or use reverse proxy for authorization.
    [ ] In multi-team scenarios, implement multi-tenancy or equivalent isolation measures.
    [ ] Configure NetworkPolicy.
    [ ] Configure limits_config.
    [ ] Set retention policies.
    [ ] Manage Loki values using Git.

### 29.2 Alloy Collection Agent

Requirements:

    [ ] Only collect necessary namespaces.
    [ ] Avoid duplicate collections.
    [ ] Do not collect worthless, high-frequency logs.
    [ ] Do not use high cardinality fields as labels.
    [] Implement RBAC with minimal permissions.
    [ ] Monitor resource requests and limits.
    [ ] Keep configuration changes tracked in Git.

### 29.3 Grafana

Requirements:

    [ ] Integrate with unified authentication systems.
    [ ] Control permissions by team/folder.
    [] Restrict access to data sources.
    [] Limit Explore functionality.
    [ ] Set reasonable default time ranges for dashboards.
    [ ] Do not enable global queries by default.
    [ ] Manage Dashboard JSON configurations using Git.
    [ ] Implement permission controls for accessing sensitive logs.

### 29.4 MinIO / S3

Requirements:

    [ ] Use dedicated users for Loki operations.
    [] Grant minimal access permissions to buckets.
    [ ] Avoid using the root user.
    [ ] Use HTTPS.
    [ ] Ensure high availability across multiple nodes.
    [ ] Monitor storage capacity.
    [ ] Set object storage lifecycle policies longer than Loki retention periods.
    [ ] Rotate keys regularly.
    [ ] Conduct bucket permission audits.

---

## Chapter Thirty: Practical Tasks

### 30.1 Task One: Identify the top namespace with the highest log volume

Execution:

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Acceptance:

    [ ] The top namespace with the highest log volume should be identified.

### 30.2 Task Two: Identify the top app with the highest log volume

Execution:

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Acceptance:

    [ ] The top app with the highest log volume should be identified.

### 30.3 Task Three: Configure retention settings

Modify the values file:

    loki:
      limits_config:
        retention_period: 168h

      compactor:
        working_directory: /var/loki/compactor
        compaction_interval: 10m
        retention_enabled: true
        retention_delete_delay: 2h
        delete_request_store: s3

Execution:

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

    [ ] The Helm template should execute without errors.
    [ ] The Helm upgrade should complete successfully.
    [ ] The Loki Pod should be running.
    [ ] There should be no compactor/retention-related errors in Loki logs.

### 30.4 Task Four: Configure limits_config settings

Add example configurations:

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
        reject_old_samples_max_age: 168hThe possible reasons include a too broad query range for Loki, overly complex regular expressions, slow object storage, or poorly designed labels.

### Misconception 32.6: Grafana Variables Can Achieve Permission Isolation

Wrong.

Variables are merely aids for querying and do not serve as security boundaries.

Permission isolation must rely on mechanisms such as Grafana permissions, data source permissions, Loki multi-tenancy, and reverse proxy authentication.

### Misconception 32.7: Sensitive Information Can Be Included in Logs and Deleted Later

Wrong.

Once sensitive information enters the logging system, it can spread to object storage, backups, dashboards, alarm notifications, and exported files.

It is better to prohibit such information from entering at the source.