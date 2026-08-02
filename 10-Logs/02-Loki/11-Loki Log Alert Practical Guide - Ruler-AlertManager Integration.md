# 11-Loki Log Alerting: Ruler and AlertManager Integration

## Document Overview

This is the eleventh article in the Loki specialized learning series, aimed at systematically mastering Loki's log alerting capabilities. The focus is on Loki Ruler, LogQL alerting rules, AlertManager integration, error log alerts, 5xx alerts, timeout alerts, CUDA OOM alerts, and the production implementation methods of log alerts in Kubernetes/SRE scenarios.

Previously completed:

    01-Loki Basic Concepts and Experimental Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experimental Selection
    04-Loki Single-Instance Helm Deployment Practical
    05-Loki Object Storage Access MinIO Practical
    06-Grafana-Alloy Collection K8S-Pod Logs Practical
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practical: Namespace-Pod-Container Log Retrieval
    09-LogQL Advanced Query Practical: json-logfmt-regexp-unwrap
    10-Grafana Integration with Loki and Log Dashboard Practical

The tenth article has already integrated Loki query capabilities into the Grafana Dashboard.

This article continues to upgrade some stable LogQL queries into log alerting rules.

This article answers the following questions:

- What is Loki Ruler;
- What is the relationship between Loki Ruler and Prometheus Rule;
- What are the differences between Loki log alerts and Prometheus metric alerts;
- What types of logs are suitable for alerts;
- What types of logs are not suitable for alerts;
- How to deploy or confirm AlertManager;
- How to configure Loki Ruler to send alerts to AlertManager;
- How to write Loki alerting rules;
- How to load Loki alerting rules via the Loki Ruler API;
- How to manage rules via ConfigMap / Helm values;
- How to verify if Ruler has loaded rules;
- How to verify if AlertManager has received alerts;
- How to simulate ERROR log alerts;
- How to simulate 5xx log alerts;
- How to simulate timeout log alerts;
- How to simulate CUDA OOM log alerts;
- How to design alert labels, grouping, suppression, routing, and Runbook;
- What are the risks and governance requirements for Loki log alerts in production environments.

---

## Tags

#Loki #LogQL #Ruler #AlertManager #It'sALogCall. #Grafana #Kubernetes #SRE #Observation #PoliceGovernance #Runbook #PodLog

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/11-Loki Log Alerting: Ruler and AlertManager Integration.md

---

## One, Experimental Objectives

After completing this article, you should be able to:

    1. Understand the role of Loki Ruler.
    2. Understand the differences between Loki log alerts and Prometheus metric alerts.
    3. Understand the role of AlertManager in the log alerting pipeline.
    4. Deploy or confirm AlertManager availability.
    5. Configure Loki Ruler to connect to AlertManager.
    6. Write Loki alerting rules.
    7. Load Loki log alerting rules.
    8. View Loki Ruler rule status.
    9. Simulate ERROR log triggering alerts.
    10. Simulate 5xx log triggering alerts.
    11. Simulate timeout log triggering alerts.
    12. Simulate CUDA OOM log triggering alerts.
    13. View Loki alerts in AlertManager.
    14. Test alert reception via webhook.
    15. Master troubleshooting methods for log alerts not triggering.
    16. Master governance methods for log alert false positives, storms, and duplicate notifications.
    17. Be able to design production-grade Loki log alerting specifications.

---

## Two, Experimental Environment

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

    [ ] Loki is running normally
    [ ] Loki Gateway is accessible
    [ ] Loki has been integrated with MinIO or has at least rule storage capability
    [ ] Loki ruler.enable_api has been enabled
    [ ] Alloy can collect Pod logs
    [ ] app-demo logs can enter Loki
    [ ] Grafana can query Loki logs
    [ ] AlertManager is accessible

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

Check log queries:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

---

## Three, What is Loki Ruler

### 3.1 Ruler's Role

Loki Ruler is the component in Loki responsible for rule calculation.

It mainly handles:

    1. Periodically executing LogQL queries.
    2. Judging whether the query results meet alerting conditions.
    3. Maintaining alerting states.
    4. Sending triggered alerts to AlertManager.
    5. Executing recording rules, converting log query results into metrics.

This article focuses on:

    Alerting Rule

That is, log alerts.

Recording Rules will be understood in production governance and performance optimization phases.

### 3.2 Relationship Between Ruler and Querier

When Ruler executes rules, it also needs to query log data from Loki.

It can be understood as:

    Ruler periodically executes LogQL
      ↓
    Obtain time series results
      ↓
    Determine if threshold is exceeded
      ↓
    Generate alerts
      ↓
    Send to AlertManager

Example:

    More than 10 app error logs in the last 5 minutes
      ↓
    Alert triggered

### 3.3 Relationship Between Ruler and AlertManager

Loki Ruler does not handle final notifications.

It is responsible for:

    Calculating rules
    Generating alerts
    Pushing alerts

AlertManager is responsible for:

    Receiving alerts
    Alert grouping
    Alert deduplication
    Alert suppression
    Alert silencing
    Alert routing
    Sending notifications

Chain:

    Loki Ruler
      ↓
    AlertManager
      ↓
    Webhook / Email / Feishu / Enterprise WeChat / DingTalk / PagerDuty

---

## Four. Differences Between Log Alerts and Metric Alerts

### 4.1 Prometheus Metric Alerts

Prometheus alerts are based on metrics.

Examples:

    CPU usage exceeds 90%
    Pod restart count increases
    HTTP 5xx ratio exceeds 5%
    P95 latency exceeds 1 second
    GPU memory exceeds 90%

Features:

    Stable
    Structured
    Suitable for thresholds
    Suitable for trends
    Suitable for SLO / SLA
    Suitable for long-term alerting mainline

### 4.2 Loki Log Alerts

Loki alerts are based on logs.

Examples:

    A large number of ERRORs in the last 5 minutes
    "database connection failed" appears in logs
    "CUDA out of memory" appears in logs
    "panic" appears in logs
    "timeout" appears in logs
    "status >= 500" in business logs exceeds threshold

Features:

    Closer to error context
    Suitable for anomaly keywords
    Suitable for supplementing metric alerts
    Suitable for discovering anomalies in applications without exposed metrics
    Prone to false positives
    Prone to being affected by log format
    Prone to generating alert noise

### 4.3 Recommended Relationship

In production, it is recommended:

    Metric alerts as primary
    Log alerts as supplementary

Do not use log alerts to completely replace metric alerts.

Recommended troubleshooting loop:

    Prometheus metric alert detects phenomenon
      ↓
    Grafana Dashboard views trend
      ↓
    Loki log query identifies root cause
      ↓
    Loki log alert supplements special errors
      ↓
    AlertManager unifies notifications and converges alerts

---

## Five. What Logs Are Suitable for Alerts

### 5.1 Logs Suitable for Alerts

Suitable:

    panic
    fatal
    crash
    exception
    database connection failed
    too many connections
    connection refused
    timeout
    deadline exceeded
    CUDA out of memory
    OOM
    license expired
    auth failed (serious anomalies)
    status >= 500
    payment failed
    data corruption
    backup failed
    replica sync failed

These logs typically have:

    Clear meaning of anomalies
    Low occurrence frequency
    Strong correlation with business failures
    Need human attention
    Can be written with a Runbook

### 5.2 Logs Not Suitable for Direct Alerts

Not suitable:

    Ordinary INFO
    Ordinary WARN
    Health check 404
    User input errors
    Expected business failures
    expected error
    ignore error
    debug logs
    Access logs with 4xx per line
    Short-term occasional timeouts
    Known ignorable anomalies

These logs can easily cause:

    Alert noise
    Alert fatigue
    On-call personnel no longer trust alerts
    True failures being buried

### 5.3 Log Alerts Must Have Thresholds

Do not alert immediately just because an error appears once.

More reasonable:

    More than 10 error logs in the last 5 minutes
    More than 20 5xx logs for a specific app in the last 5 minutes
    More than 5 timeout logs in the last 10 minutes
    More than 0 CUDA OOM logs in the last 5 minutes, but only for ai-prod
    More than 0 panic logs in the last 5 minutes

Different errors should have different thresholds.

---

## Six. Alerting Architecture

### 6.1 Architecture Diagram

    Pod stdout / stderr
      ↓
    Alloy
      ↓
    Loki
      ↓
    Ruler periodically executes LogQL
      ↓
    Alert condition met
      ↓
    AlertManager
      ↓
    Receiver
      ↓
    Webhook / Email / Enterprise WeChat / Feishu / DingTalk

### 6.2 Experiment Chain in This Article

This article uses:

    app-demo test application
      ↓
    Alloy collects logs
      ↓
    Loki stores logs
      ↓
    Loki Ruler calculates rules
      ↓
    AlertManager receives alerts
      ↓
    AlertManager Web UI views alerts

Optional:

    webhook.site / self-hosted webhook receiver / alertmanager-webhook-demo

This article primarily verifies through AlertManager UI.

---

## Seven. Deployment or Verification of AlertManager

If you have already deployed AlertManager via kube-prometheus-stack, you can skip the deployment section.

### 7.1 Check monitoring Namespace

    kubectl get ns monitoring

If it does not exist:

    kubectl create namespace monitoring

### 7.2 Check if AlertManager Already Exists

    kubectl get pods -n monitoring | grep -i alertmanager

    kubectl get svc -n monitoring | grep -i alertmanager

If the Service already exists, record the Service name.

Common Service names may be:

    alertmanager
    kube-prometheus-stack-alertmanager
    prometheus-kube-prometheus-alertmanager

Please refer to the actual environment for specifics.

### 7.3 Deploying Independent AlertManager with prometheus-community/alertmanager

Add Helm repository:

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

    helm repo update

Search for Chart:

    helm search repo prometheus-community/alertmanager --versions | head -20

Record version:

    ALERTMANAGER_CHART_VERSION=<actual version>

Create values:

    values-alertmanager.yaml

Content:

    config:
      global:
        resolve_timeout: 5m

      route:
        receiver: default-webhook
        group_by:
          - alertname
          - namespace
          - app
          - severity
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 2h

      receivers:
        - name: default-webhook
          webhook_configs:
            - url: http://alertmanager-webhook-demo.monitoring.svc.cluster.local:8080/alert
              send_resolved: true

    service:
      type: ClusterIP

    persistence:
      enabled: false

Notes:

    This example uses webhook receiver.
    If there is no webhook-demo temporarily, you can first change the receiver to an empty receiver or use AlertManager UI to view.
    Production environments need to configure enterprise WeChat, Feishu, DingTalk, email, or unified alerting platform.

Install:

    helm install alertmanager prometheus-community/alertmanager \
      -n monitoring \
      --version <ALERTMANAGER_CHART_VERSION> \
      -f values-alertmanager.yaml

Check:

    helm list -n monitoring

    kubectl get pods -n monitoring -o wide

    kubectl get svc -n monitoring

### 7.4 Accessing AlertManager UI

Port forward:

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

Access:

    http://127.0.0.1:9093

If the Service name is not alertmanager:

    kubectl get svc -n monitoring | grep -i alertmanager

Then replace the Service name.

### 7.5 AlertManager Health Check

    curl -s http://127.0.0.1:9093/-/ready

Expected:

    OK

---

## VIII. Optional: Deploying webhook Test Receiver

### 8.1 Why Need webhook Test Receiver

AlertManager UI can view alerts, but webhook receiver can verify:

    Whether AlertManager actually sends notifications
    What the alert payload looks like
    Whether send_resolved takes effect
    Whether route is correct

### 8.2 Creating Simple webhook Demo

Create file:

    alertmanager-webhook-demo.yaml

Content:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: alertmanager-webhook-demo
      namespace: monitoring
      labels:
        app: alertmanager-webhook-demo
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: alertmanager-webhook-demo
      template:
        metadata:
          labels:
            app: alertmanager-webhook-demo
        spec:
          containers:
            - name: webhook
              image: mendhak/http-https-echo:34
              ports:
                - containerPort: 8080

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: alertmanager-webhook-demo
      namespace: monitoring
    spec:
      selector:
        app: alertmanager-webhook-demo
      ports:
        - name: http
          port: 8080
          targetPort: 8080

Apply:

    kubectl apply -f alertmanager-webhook-demo.yaml

Check:

    kubectl get pods -n monitoring | grep webhook

    kubectl get svc -n monitoring | grep webhook

### 8.3 Viewing alerts received by webhook /think

kubectl logs deploy/alertmanager-webhook-demo -n monitoring -f

If AlertManager sends an alert, you will see the HTTP request content in the logs.

---

## 9. Configuring Loki Ruler to Connect with AlertManager

### 9.1 View Current Loki Values

    helm get values loki -n logging -a > values-loki-current.yaml

Search for Ruler:

    grep -n "ruler" values-loki-current.yaml

You can also view default values:

    helm show values grafana-community/loki \
      --version <CHART_VERSION> \
      > values-loki-default.yaml

Search:

    grep -n "ruler" values-loki-default.yaml

    grep -n "alertmanager" values-loki-default.yaml

### 9.2 Core Loki Ruler Configuration

Loki configuration typically requires:

    ruler:
      enable_api: true
      alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093
      storage:
        type: s3 or local
      rule_path: /tmp/loki/rules

If using MinIO:

    ruler:
      enable_api: true
      alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093
      storage:
        type: s3
        s3:
          bucketnames: loki-ruler

Explanation:

    alertmanager_url:
        AlertManager address.

    enable_api:
        Allows managing rules via Ruler API.

    storage:
        Rule storage backend.

    rule_path:
        Ruler's local rule working directory.

### 9.3 Helm values Example

Create or modify:

    values-loki-monolithic-minio-ruler.yaml

Add or confirm the following on top of your existing Loki values:

    loki:
      auth_enabled: false

      commonConfig:
        replication_factor: 1

      ruler:
        enable_api: true
        alertmanager_url: http://alertmanager.monitoring.svc.cluster.local:9093
        storage:
          type: s3
          s3:
            bucketnames: loki-ruler
        rule_path: /tmp/loki/rules
        ring:
          kvstore:
            store: inmemory

      limits_config:
        allow_structured_metadata: true
        volume_enabled: true

Explanation:

    The above snippet needs to be merged into your complete values from Chapter 05.
    Do not copy this small segment directly to overwrite the complete values.
    Field names are based on the current Loki Helm Chart version.
    Different Chart versions may vary, so first verify with helm template.

### 9.4 Confirm AlertManager Service Address

If the AlertManager Service name is alertmanager:

    http://alertmanager.monitoring.svc.cluster.local:9093

If it's kube-prometheus-stack:

    http://kube-prometheus-stack-alertmanager.monitoring.svc.cluster.local:9093

Or:

    http://alertmanager-operated.monitoring.svc.cluster.local:9093

Actual confirmation:

    kubectl get svc -n monitoring | grep -i alertmanager

Test from the logging Namespace:

    kubectl run curl-alertmanager-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -s http://alertmanager.monitoring.svc.cluster.local:9093/-/ready

Exit:

    exit

### 9.5 Helm Template Pre-check

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio-ruler.yaml \
      > loki-ruler-rendered.yaml

Check:

    grep -n "alertmanager" loki-ruler-rendered.yaml

    grep -n "enable_api" loki-ruler-rendered.yaml

    grep -n "ruler" loki-ruler-rendered.yaml

### 9.6 Upgrade Loki

Backup current values:

    helm get values loki -n logging -a > backup-values-loki-before-ruler.yaml

Execute upgrade:

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio-ruler.yaml

Check:

    helm history loki -n logging

    kubectl get pods -n logging -o wide

    kubectl logs <loki-pod-name> -n logging --tail=300 | grep -Ei "ruler|alertmanager|rule|error|warn"

## 10. Verifying the Ruler API

### 10.1 Port Forwarding Loki Gateway

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

### 10.2 Viewing Ruler rules

    curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq

If an empty list is returned, it indicates there are currently no rules.

If a 404 or disabled response is returned, check:

    ruler.enable_api is set to true
    Whether the Gateway forwards ruler API
    Whether Loki configuration is active
    Whether the current Chart requires additional ruler enablement
    Whether the Service points to the correct component

### 10.3 Viewing ruler logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "ruler|rule|alertmanager"

Pay attention to:

    ruler started
    loading rules
    syncing rules
    alertmanager url
    failed to load rules
    failed to send alerts
    permission denied
    bucket not found

---

## 11. Loki Alert Rule Format

### 11.1 Basic Structure

Loki alert rule format is similar to Prometheus rules.

Example:

    groups:
      - name: app-log-alerts
        interval: 1m
        rules:
          - alert: AppErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo"}
                    |~ "(?i)error|exception|panic|failed" [5m]
                )
              ) > 10
            for: 2m
            labels:
              severity: warning
              source: loki
              team: sre
            annotations:
              summary: "Application has excessive error logs"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has exceeded threshold for error logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/loki-app-error-logs"

### 11.2 Field Descriptions

    groups:
        Rule group.

    name:
        Rule group name.

    interval:
        Rule execution interval.

    rules:
        Rule list.

    alert:
        Alert name.

    expr:
        LogQL expression.

    for:
        Duration the condition must persist before triggering an alert.

    labels:
        Alert labels for grouping, routing, and severity level.

    annotations:
        Alert description for notification content and Runbook.

### 11.3 Labels Design

Recommended labels:

    severity
    source
    team
    category
    environment

Example:

    labels:
      severity: warning
      source: loki
      team: sre
      category: logs

### 11.4 Annotations Design

Recommended annotations:

    summary
    description
    runbook_url
    dashboard_url

Example:

    annotations:
      summary: "Pod has excessive error logs"
      description: "namespace={{ $labels.namespace }}, pod={{ $labels.pod }} has exceeded threshold for error logs in the last 5 minutes"
      runbook_url: "https://example.com/runbooks/pod-error-logs"
      dashboard_url: "https://grafana.example.com/d/loki-pod-logs"

---

## 12. Creating the First ERROR Log Alert Rule

### 12.1 Creating the Rule File

Create:

    loki-rules-app-demo.yaml

Content:

    groups:
      - name: app-demo-log-alerts
        interval: 1m
        rules:
          - alert: AppDemoErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo"}
                    |~ "(?i)error|exception|panic|failed" [5m]
                )
              ) > 5
            for: 1m
            labels:
              severity: warning
              source: loki
              team: sre
              category: logs
            annotations:
              summary: "app-demo has excessive error logs"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has exceeded threshold for error logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/app-demo-error-logs"

### 12.2 Rule Explanation

Expression:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    ) > 5

Meaning:

    Query logs containing error/exception/panic/failed in the app-demo namespace from the last 5 minutes.
    Group statistics by namespace and app.
    If the count exceeds 5, it enters pending.
    Fires after 1 minute of duration.

### 12.3 Threshold Explanation

Learning environment:

    > 5

Production environment:

    Set according to business log volume.
    Different apps should have different thresholds.
    It's not recommended to use a unified threshold for all applications.

---

## ThirteenI don't know.Loading Rules via Ruler API

### 13.1 API Path

Loki Ruler API uses the namespace concept.

Example path:

    /loki/api/v1/rules/{namespace}

Here, namespace is not a Kubernetes Namespace, but the namespace of the rule file.

Recommended usage:

    app-demo

### 13.2 Upload Rules

Execute:

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

If there is no output, it typically indicates success.

If an error is returned, troubleshoot based on the prompt.

### 13.3 View Rules

    curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq

View specified namespace:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

### 13.4 View Rule Status

Supported by some versions:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

You can also observe via Loki logs:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "rule|ruler|app-demo|alert"

---

## FourteenI don't know.Simulating ERROR Alert

### 14.1 Manually Write ERROR Logs

Execute:

    for i in $(seq 1 10); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"alert-error-demo\"
              },
              \"values\": [
                [\"${TS}\", \"ERROR database connection failed, test alert line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

### 14.2 Query to Confirm Logs Exist

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="alert-error-demo"} |= "ERROR"' \
      --data-urlencode 'limit=20' | jq

### 14.3 Wait for Rule Execution

Wait:

    1 to 2 minutes

Check AlertManager:

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

Access:

    http://127.0.0.1:9093

Expected:

    AppDemoErrorLogsTooMany enters firing or pending.

### 14.4 View Loki Ruler Logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "AppDemoErrorLogsTooMany|ruler|alert|error"

### 14.5 View Webhook Reception

If a webhook demo is deployed:

    kubectl logs deploy/alertmanager-webhook-demo -n monitoring --tail=200

Expected to see the alert payload pushed by AlertManager.

---

## FifteenI don't know.JSON 5xx Log Alert

### 15.1 Create Rules

Add the following to loki-rules-app-demo.yaml:

- alert: AppDemoJson5xxLogsTooMany
  expr: |
    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    ) > 5
  for: 1m
  labels:
    severity: warning
    source: loki
    team: sre
    category: logs
  annotations:
    summary: "Too Many JSON 5xx Logs"
    description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 5 status>=500 logs in the last 5 minutes"
    runbook_url: "https://example.com/runbooks/json-5xx-logs"

Complete Rule Group Example:

    groups:
      - name: app-demo-log-alerts
        interval: 1m
        rules:
          - alert: AppDemoErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo"}
                    |~ "(?i)error|exception|panic|failed" [5m]
                )
              ) > 5
            for: 1m
            labels:
              severity: warning
              source: loki
              team: sre
              category: logs
            annotations:
              summary: "Too Many Error Logs in app-demo"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 5 error logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/app-demo-error-logs"

          - alert: AppDemoJson5xxLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo", app="json-log-demo"}
                    | json
                    | __error__=""
                    | status >= 500 [5m]
                )
              ) > 5
            for: 1m
            labels:
              severity: warning
              source: loki
              team: sre
              category: logs
            annotations:
              summary: "Too Many JSON 5xx Logs"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 5 status>=500 logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/json-5xx-logs"

### 15.2 Upload Rules

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

View:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

### 15.3 Trigger 5xx Logs

If json-log-demo has already been deployed, you can delete and redeploy:

    kubectl delete job json-log-demo -n app-demo

    kubectl apply -f json-log-demo.yaml

View logs:

    kubectl get pod -n app-demo | grep json-log-demo

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

### 15.4 Query Verification

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | status >= 500' \
      --data-urlencode 'limit=20' | jq

### 15.5 View AlertManager

    http://127.0.0.1:9093

Expected:

    AppDemoJson5xxLogsTooMany alert is triggered.

---

## SixteenI don't know.Timeout Log Alerts

### 16.1 Rules /think

- alert: AppDemoTimeoutLogsTooMany
  expr: |
    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)timeout|timed out|deadline exceeded" [5m]
      )
    ) > 3
  for: 1m
  labels:
    severity: warning
    source: loki
    team: sre
    category: logs
  annotations:
    summary: "Too many timeout logs"
    description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has exceeded 3 timeout-related logs in the last 5 minutes"
    runbook_url: "https://example.com/runbooks/timeout-logs"

### 16.2 Simulating timeout logs

    for i in $(seq 1 6); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"alert-timeout-demo\"
              },
              \"values\": [
                [\"${TS}\", \"ERROR upstream timeout, deadline exceeded, test alert line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

### 16.3 Query confirmation

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="alert-timeout-demo"} |~ "(?i)timeout|deadline exceeded"' \
      --data-urlencode 'limit=20' | jq

---

## SeventeenI don't know.CUDA OOM Log Alert

### 17.1 Applicable scenarios

Suitable for GPU/AI workloads.

Typical logs:

    CUDA out of memory
    RuntimeError: CUDA error
    out of memory
    CUBLAS_STATUS_ALLOC_FAILED
    NCCL error

### 17.2 Rule

In production, it's recommended to limit namespace, for example:

    ai-prod
    gpu-prod
    model-serving

In experiments, you can use app-demo.

Rule:

    - alert: CudaOomLogsDetected
      expr: |
        sum by (namespace, app, pod) (
          count_over_time(
            {namespace="app-demo"}
              |~ "(?i)cuda out of memory|CUDA error|CUBLAS_STATUS_ALLOC_FAILED|NCCL error|out of memory" [5m]
          )
        ) > 0
      for: 1m
      labels:
        severity: critical
        source: loki
        team: sre
        category: gpu
      annotations:
        summary: "Detected CUDA OOM/GPU-related error logs"
        description: "namespace={{ $labels.namespace }}, app={{ $labels.app }}, pod={{ $labels.pod }} has GPU/CUDA-related error logs"
        runbook_url: "https://example.com/runbooks/cuda-oom"

### 17.3 Simulating CUDA OOM logs

    for i in $(seq 1 3); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"gpu-worker-demo\",
                \"pod\": \"gpu-worker-demo-001\"
              },
              \"values\": [
                [\"${TS}\", \"RuntimeError: CUDA out of memory. Tried to allocate 2.00 GiB on GPU 0, test line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

### 17.4 Query confirmation

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="gpu-worker-demo"} |~ "(?i)cuda out of memory|CUDA error|out of memory"' \
      --data-urlencode 'limit=20' | jq

### 17.5 Production Notes

CUDA OOM alerts should be combined with:

    DCGM Exporter GPU Memory Metrics
    Pod OOMKilled Metrics
    Application Logs
    GPU Node Resource Allocation
    requests/limits
    Scheduling Policies
    Model batch size
    Inference Concurrency
    Memory Fragmentation

Do not rely solely on log alerts.

---

## Eighteen, Complete Rule File Example

Create:

    loki-rules-app-demo.yaml

Full content:

    groups:
      - name: app-demo-log-alerts
        interval: 1m
        rules:
          - alert: AppDemoErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo"}
                    |~ "(?i)error|exception|panic|failed" [5m]
                )
              ) > 5
            for: 1m
            labels:
              severity: warning
              source: loki
              team: sre
              category: logs
            annotations:
              summary: "app-demo has excessive error logs"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 5 error logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/app-demo-error-logs"

          - alert: AppDemoJson5xxLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo", app="json-log-demo"}
                    | json
                    | __error__=""
                    | status >= 500 [5m]
                )
              ) > 5
            for: 1m
            labels:
              severity: warning
              source: loki
              team: sre
              category: logs
            annotations:
              summary: "Excessive JSON 5xx logs"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 5 status>=500 logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/json-5xx-logs"

          - alert: AppDemoTimeoutLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app-demo"}
                    |~ "(?i)timeout|timed out|deadline exceeded" [5m]
                )
              ) > 3
            for: 1m
            labels:
              severity: warning
              source: loki
              team: sre
              category: logs
            annotations:
              summary: "Excessive timeout logs"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 3 timeout-related logs in the last 5 minutes"
              runbook_url: "https://example.com/runbooks/timeout-logs"

- alert: CudaOomLogsDetected
            expr: |
              sum by (namespace, app, pod) (
                count_over_time(
                  {namespace="app-demo"}
                    |~ "(?i)cuda out of memory|CUDA error|CUBLAS_STATUS_ALLOC_FAILED|NCCL error|out of memory" [5m]
                )
              ) > 0
            for: 1m
            labels:
              severity: critical
              source: loki
              team: sre
              category: gpu
            annotations:
              summary: "CUDA OOM/GPU-related error logs detected"
              description: "namespace={{ $labels.namespace }}, app={{ $labels.app }}, pod={{ $labels.pod }} has GPU/CUDA-related error logs"
              runbook_url: "https://example.com/runbooks/cuda-oom"

Upload:

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

Check:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

---

## NineteenI don't know.Managing Rules via ConfigMap

### 19.1 Why Not Recommend Long-term Manual API Upload

API uploads are suitable for learning and temporary testing.

Production recommends:

    Git manage rule files
    Helm values manage rules
    ConfigMap manage rules
    GitOps synchronize rules
    Change approval
    Change rollback

### 19.2 ConfigMap Example

Create file:

    loki-rules-configmap.yaml

Content:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: loki-alert-rules
      namespace: logging
    data:
      app-demo.yaml: |
        groups:
          - name: app-demo-log-alerts
            interval: 1m
            rules:
              - alert: AppDemoErrorLogsTooMany
                expr: |
                  sum by (namespace, app) (
                    count_over_time(
                      {namespace="app-demo"}
                        |~ "(?i)error|exception|panic|failed" [5m]
                    )
                  ) > 5
                for: 1m
                labels:
                  severity: warning
                  source: loki
                  team: sre
                  category: logs
                annotations:
                  summary: "Too many error logs in app-demo"
                  description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has more than 5 error logs in the last 5 minutes"
                  runbook_url: "https://example.com/runbooks/app-demo-error-logs"

Apply:

    kubectl apply -f loki-rules-configmap.yaml

### 19.3 Mount to Loki

Need to additionally mount this ConfigMap in Loki Helm values and set the Ruler read path.

Due to different field names in different Loki Helm Chart versions, first check:

    helm show values grafana-community/loki --version <CHART_VERSION> > values-loki-default.yaml

Search:

    grep -n "extraVolumes" values-loki-default.yaml

    grep -n "extraVolumeMounts" values-loki-default.yaml

    grep -n "ruler" values-loki-default.yaml

Common approach:

    extraVolumes:
      - name: loki-alert-rules
        configMap:
          name: loki-alert-rules

    extraVolumeMounts:
      - name: loki-alert-rules
        mountPath: /etc/loki/rules/fake
        readOnly: true

    loki:
      ruler:
        rule_path: /tmp/loki/rules

Notes:

    The exact syntax depends on the Helm Chart version.
    Learning environments prioritize using Ruler API for intuitiveness.
    Production environments should solidify to Helm values based on current Chart version.

---

## TwentyI don't know.AlertManager Route Configuration

### 20.1 Basic route

AlertManager route example: /think

route:
  receiver: default-webhook
  group_by:
    - alertname
    - namespace
    - app
    - severity
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 2h

Explanation:

  group_by:
      Which labels to group alerts by.

  group_wait:
      Waiting time before the first notification.

  group_interval:
      Interval for re-notifying new alerts in the same group.

  repeat_interval:
      Interval for repeating notifications for unresolved alerts.

### 20.2 Route by severity

Example:

  route:
    receiver: default-webhook
    group_by:
      - alertname
      - namespace
      - app
      - severity
    routes:
      - matchers:
          - severity="critical"
        receiver: critical-webhook
      - matchers:
          - severity="warning"
        receiver: warning-webhook

### 20.3 Route by team

Example:

  route:
    receiver: sre-webhook
    routes:
      - matchers:
          - team="sre"
        receiver: sre-webhook
      - matchers:
          - team="platform"
        receiver: platform-webhook
      - matchers:
          - team="ai"
        receiver: ai-webhook

### 20.4 Loki Alert Label Requirements

For AlertManager to route correctly, Loki rule labels should include:

  team
  severity
  source
  category

Example:

  labels:
    severity: critical
    source: loki
    team: ai
    category: gpu

---

## 21. Alert States: inactive, pending, firing

### 21.1 inactive

Indicates:

  The rule condition is not met.

### 21.2 pending

Indicates:

  The rule condition is met, but hasn't sustained for the required duration.

Example:

  for: 2m

When the condition is just met for 30 seconds, the state is pending.

### 21.3 firing

Indicates:

  The condition is met and has sustained for longer than the for duration.

At this point, Ruler will send the alert to AlertManager.

### 21.4 Why Use for

The for clause prevents false positives from transient spikes.

Example:

  A single error appearing for an instant doesn't require an alert.
  An error exceeding the threshold for 1 minute is more meaningful.

---

## 22. Alert Recovery

### 22.1 Alert Recovery Conditions

When the expr no longer matches, the alert will recover.

Example:

  The number of error logs in the last 5 minutes <= 5

The alert will transition from firing to resolved.

### 22.2 send_resolved

In AlertManager receiver:

  send_resolved: true

Indicates notifications are sent when the alert recovers.

Production recommendation:

  Send recovery notifications for critical alerts.
  Low-priority alerts may be optional based on scenario.

### 22.3 Log Alert Recovery Has Delay

Because of the time window:

  [5m]

Even after stopping error log writes, the recent 5-minute window may still contain old errors.

Thus, recovery naturally has a delay.

---

## 23. Log Alert Troubleshooting Flow

### 23.1 Alert Not Triggered

Troubleshooting order:

  1. Does the LogQL query itself return results?
  2. Does the expression return numerical values?
  3. Is the threshold too high?
  4. Has the for duration not yet passed?
  5. Is Ruler enabled?
  6. Did the rule load successfully?
  7. Can Ruler query Loki?
  8. Can Ruler connect to AlertManager?
  9. Did AlertManager receive the alert?
  10. Does the AlertManager routing match the receiver?

### 23.2 Manually Execute LogQL

Example:

  sum by (namespace, app) (
    count_over_time(
      {namespace="app-demo"}
        |~ "(?i)error|exception|panic|failed" [5m]
    )
  )

Can be executed in Grafana Explore or via API.

### 23.3 Check if Rule is Loaded

  curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq

### 23.4 Check Loki Ruler Logs

  kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "ruler|rule|alertmanager|failed|error"

### 23.5 Check AlertManager UI

  http://127.0.0.1:9093

Check:

  Alerts
  Silences
  Status
  Receivers

### 23.6 Check AlertManager Logs

  kubectl logs <alertmanager-pod-name> -n monitoring --tail=300

Filter:

  kubectl logs <alertmanager-pod-name> -n monitoring --tail=500 | grep -Ei "error|warn|notify|webhook|receiver"

---

## 24. Common Issue: Ruler API Unavailable

### 24.1 Symptoms

Accessing: /think

curl -s http://127.0.0.1:3100/loki/api/v1/rules

Returns:

    404
    ruler API disabled
    route not found
    empty response

### 24.2 Possible Causes

    ruler.enable_api is not enabled.
    Loki configuration is not active.
    Gateway is not forwarding Ruler API.
    Ruler is not enabled in the current deployment mode.
    Helm values field is written incorrectly.
    Loki Pod has not been restarted to load new configuration.

### 24.3 Troubleshooting

    helm get values loki -n logging -a | grep -n "ruler" -A 40

    helm get manifest loki -n logging | grep -n "enable_api"

    kubectl logs <loki-pod-name> -n logging --tail=300 | grep -Ei "ruler|rule"

### 24.4 Fix

    1. Confirm that ruler.enable_api: true in values.
    2. Check configuration with helm template.
    3. Update Loki with helm upgrade.
    4. Wait for Loki Pod to be Ready.
    5. Revisit /loki/api/v1/rules.

---

## 25. Common Issue Two: Rule Upload Failure

### 25.1 Symptoms

Rule upload returns:

    400 Bad Request
    YAML parse error
    rule parse error
    invalid expression
    group name missing

### 25.2 Possible Causes

    YAML indentation error.
    expr LogQL syntax error.
    groups/rules field format error.
    alert name is invalid.
    interval format error.
    annotations template error.

### 25.3 Troubleshooting

First test expr in Grafana Explore.

Confirm the expression can return results.

Then check YAML:

    cat loki-rules-app-demo.yaml

    yamllint loki-rules-app-demo.yaml

If yamllint is not available, manually check indentation first.

### 25.4 Fix Principles

    First ensure LogQL can execute alone.
    Then put into rule file.
    Keep YAML indentation consistent.
    Use | multiline format for expr.
    Each rule has alert, expr, for, labels, annotations.

---

## 26. Common Issue Three: AlertManager Not Receiving Alerts

### 26.1 Possible Causes

    Ruler is not connected to AlertManager.
    alertmanager_url is incorrect.
    DNS resolution failure.
    NetworkPolicy blocking.
    AlertManager Service port error.
    Rules are not in firing state.
    Alert is silenced by AlertManager.
    Alert is inhibited.
    route does not match receiver.

### 26.2 Test AlertManager from logging Namespace

    kubectl run curl-alertmanager-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside container:

    curl -s http://alertmanager.monitoring.svc.cluster.local:9093/-/ready

Exit:

    exit

### 26.3 Check Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "alertmanager|notify|ruler|error"

### 26.4 Check AlertManager Logs

    kubectl logs <alertmanager-pod-name> -n monitoring --tail=500 | grep -Ei "notify|webhook|error|warn"

---

## 27. Common Issue Four: Too Many Alerts

### 27.1 Causes

    Threshold is too low.
    Query range is too wide.
    group_by is too detailed.
    Grouping by pod causes one alert per pod.
    Log keyword is too broad.
    Application continuously logs errors.
    Health checks or expected errors are also matched.
    repeat_interval is too short.

### 27.2 Governance Methods

    Increase threshold.
    Add for time duration.
    Narrow namespace/app scope.
    Exclude expected error.
    Use level="error" instead of keyword error.
    Group by app, not by pod.
    Adjust AlertManager group_by.
    Increase repeat_interval.
    Use silence for known maintenance windows.

### 27.3 Example: Exclude Expected Error

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed"
          != "expected error"
          != "ignore" [5m]
      )
    ) > 10

---

## 28. Common Issue Five: Alert Not Recovering

### 28.1 Possible Causes

    Old logs still exist in time window.
    Application still outputs errors.
    Query expression range is too wide.
    AlertManager repeat_interval causes repeated notifications.
    AlertManager UI has not refreshed.
    Ruler status synchronization delay.

### 28.2 Troubleshooting

Execute current expression:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

If still greater than threshold, alert will not recover.

### 28.3 Handling

Wait for the window to naturally slide past.
Stop the error log source.
Adjust thresholds and time window.
Confirm if the app still reports errors.

---

## Twenty-Nine, Log Alert Design Guidelines

### 29.1 Alert Name Guidelines

Recommended:

    AppErrorLogsTooMany
    AppJson5xxLogsTooMany
    AppTimeoutLogsTooMany
    CudaOomLogsDetected
    DatabaseConnectionFailedLogsTooMany

Do not use:

    ErrorAlert
    TestAlert
    LokiAlert1
    AppProblem

### 29.2 Severity Guidelines

Recommended:

    critical:
        Needs immediate attention, may affect business availability.

    warning:
        Needs attention, but not necessarily interrupt business immediately.

    info:
        Only for notification or observation, not recommended for strong notification channels.

### 29.3 Labels Guidelines

Mandatory:

    severity
    source
    team
    category

Recommended:

    environment
    service
    runbook

Example:

    labels:
      severity: warning
      source: loki
      team: sre
      category: logs

### 29.4 Annotations Guidelines

Mandatory:

    summary
    description

Recommended:

    runbook_url
    dashboard_url

Example:

    annotations:
      summary: "Too many application error logs"
      description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has exceeded threshold for errors in the last 5 minutes"
      runbook_url: "https://example.com/runbooks/app-error-logs"
      dashboard_url: "https://grafana.example.com/d/k8s-loki-logs"

### 29.5 Threshold Guidelines

Do not use a global unified threshold.

Should be set separately based on:

    app type
    log volume
    business importance
    historical baseline
    error semantics

---

## Thirty, Log Alerts and Runbook

### 30.1 Each Alert Must Answer

    What is the issue?
    What is the impact scope?
    Which Dashboard should be checked?
    Which kubectl commands should be executed?
    Which team should be contacted?
    Under what circumstances should escalation occur?
    How to recover?
    How to close false positives?

### 30.2 AppErrorLogsTooMany Runbook Example

Troubleshooting Steps:

    1. Open the Grafana Loki Dashboard.
    2. Select namespace and app.
    3. View recent error logs.
    4. Query LogQL:
        {namespace="xxx", app="xxx"} |~ "(?i)error|exception|panic|failed"
    5. View Pod:
        kubectl get pod -n xxx -o wide
    6. View Pod Events:
        kubectl describe pod <pod> -n xxx
    7. View previous logs:
        kubectl logs <pod> -n xxx --previous --tail=100
    8. Determine if it relates to deployment, configuration, dependent services, database, or network.
    9. Roll back the application or escalate to the business team if necessary.

### 30.3 CudaOomLogsDetected Runbook Example

Troubleshooting Steps:

    1. Check the namespace/app/pod in the alert.
    2. Query Loki:
        {namespace="xxx", pod="xxx"} |~ "(?i)cuda out of memory|CUDA error|out of memory"
    3. Check Pod resources:
        kubectl describe pod <pod> -n xxx
    4. Check GPU metrics:
        DCGM Exporter / Grafana GPU Dashboard
    5. Check batch size, concurrency, and model size.
    6. Check if the Pod shares GPU.
    7. Reduce batch size or scale up GPU resources if necessary.
    8. For training tasks, confirm checkpoint and retry strategies.

---

## Thirty-One, Production Considerations

### 31.1 Do Not Overuse Log Alerts

Log alerts can easily generate noise.

Principles:

    Only alert for strong fault semantic logs.
    Do not alert for ordinary INFO/WARN logs.
    Do not alert for expected business errors.
    Do not blindly alert for global "error" keywords.

### 31.2 Metric Alerts Take Priority

For example, HTTP 5xx:

    Prioritize Prometheus metrics for ratio alerts.
    Loki 5xx log alerts as supplementary for localization.

For example, latency:

    Prioritize request latency metrics.
    Loki duration_ms for supplementary use when no metrics are available.

### 31.3 Alert Queries Should Be Lightweight

Avoid:

    {namespace=~".+"} | json | level="error"

Recommended:

    {environment="prod", namespace="order", app="order-api"} | json | __error__="" | level="error"

### 31.4 Alerts Must Have Owners

Each alert must be routable to a team.

Dependent labels:

    team
    namespace
    app
    severity

Alerts without owners become SRE noise.

### 31.5 Alerts Should Be Regularly Reviewed

Regularly check:

    Which alerts are most frequent?
    Which alerts are unattended?
    Which alerts are false positives?
    Which alerts have excessively low thresholds?
    Which alerts lack Runbook?
    Which alerts have duplicate notifications?
    Which logs require application-side governance?

---

## Thirty-Two, Operational Tasks

### 32.1 Task One: Confirm AlertManager

Execute:

    kubectl get pods -n monitoring | grep -i alertmanager

    kubectl get svc -n monitoring | grep -i alertmanager

kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

curl -s http://127.0.0.1:9093/-/ready

Acceptance:

    [ ] AlertManager Pod Running
    [ ] AlertManager Service exists
    [ ] /-/ready returns OK
    [ ] AlertManager UI is accessible

### 32.2 Task Two: Configure Loki Ruler

Execution:

    helm get values loki -n logging -a > backup-values-loki-before-ruler.yaml

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio-ruler.yaml \
      > loki-ruler-rendered.yaml

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio-ruler.yaml

Acceptance:

    [ ] helm template has no errors
    [ ] helm upgrade succeeds
    [ ] Loki Pod Running
    [ ] Loki logs show no obvious errors for Ruler
    [ ] /loki/api/v1/rules is accessible

### 32.3 Task Three: Upload Rules

Execution:

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

View:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

Acceptance:

    [ ] Rules uploaded successfully
    [ ] Can query rules
    [ ] Rule group name is correct
    [ ] Alert name is correct

### 32.4 Task Four: Trigger ERROR Alert

Execution:

    for i in $(seq 1 10); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"alert-error-demo\"
              },
              \"values\": [
                [\"${TS}\", \"ERROR database connection failed, test alert line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

Acceptance:

    [ ] Loki can query ERROR logs
    [ ] AlertManager shows AppDemoErrorLogsTooMany
    [ ] webhook demo can see alert requests

### 32.5 Task Five: Trigger timeout Alert

Execution:

    for i in $(seq 1 6); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"alert-timeout-demo\"
              },
              \"values\": [
                [\"${TS}\", \"ERROR upstream timeout, deadline exceeded, test alert line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

Acceptance:

    [ ] AlertManager shows timeout-related alerts

### 32.6 Task Six: Trigger CUDA OOM Alert

Execution: /think

for i in $(seq 1 3); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\",
                \"app\": \"gpu-worker-demo\",
                \"pod\": \"gpu-worker-demo-001\"
              },
              \"values\": [
                [\"${TS}\", \"RuntimeError: CUDA out of memory. Tried to allocate 2.00 GiB on GPU 0, test line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

Acceptance:

    [ ] AlertManager shows CudaOomLogsDetected
    [ ] Alert severity is critical
    [ ] Alert category is gpu

---

## Thirty-ThreeI don't know.Acceptance Checklist

After completing this document, confirm:

    [ ] Understand the role of Loki Ruler
    [ ] Understand the role of AlertManager
    [ ] Understand the difference between log alerts and metric alerts
    [ ] AlertManager is deployed or confirmed to be available
    [ ] AlertManager UI is accessible
    [ ] Loki Ruler is enabled
    [ ] Loki Ruler API is accessible
    [ ] Loki Ruler can connect to AlertManager
    [ ] Loki alerting rule has been written
    [ ] Rule has been uploaded to Loki
    [ ] Can view rule content
    [ ] Can simulate ERROR logs
    [ ] ERROR log alerts can trigger
    [ ] Can simulate 5xx logs
    [ ] 5xx log alerts can trigger
    [ ] Can simulate timeout logs
    [ ] Timeout alerts can trigger
    [ ] Can simulate CUDA OOM logs
    [ ] CUDA OOM alerts can trigger
    [ ] AlertManager can display Loki alerts
    [ ] Webhook demo can receive alert requests
    [ ] Understand pending and firing
    [ ] Understand the role of for
    [ ] Understand the role of send_resolved
    [ ] Can troubleshoot alerts not triggering
    [ ] Can troubleshoot AlertManager not receiving alerts
    [ ] Can write log alert production standards

---

## Thirty-FourI don't know.Common Misconceptions

### 34.1 Misconception One: Any ERROR in logs must trigger an alert

Incorrect.

Judgment should be based on quantity, time window, business meaning, and impact level.

More reasonable:

    Alert only if error count exceeds threshold in the last 5 minutes.

### 34.2 Misconception Two: Log alerts can replace metric alerts

Incorrect.

Metric alerts remain the mainstay.

Log alerts are used to supplement specific abnormal semantics.

### 34.3 Misconception Three: All applications use the same threshold

Incorrect.

Different applications have different log volumes, so thresholds should be set based on application baselines.

### 34.4 Misconception Four: Alerts without Runbook are acceptable

Incorrect.

Alerts without Runbook are hard to close the loop.

Every production alert should have handling instructions.

### 34.5 Misconception Five: The more detailed the Pod grouping, the better

Not necessarily.

Grouping by Pod may generate excessive alerts.

In production, prioritize grouping by:

    namespace
    app
    severity

Aggregation. Drill down to Pod only during temporary troubleshooting.

### 34.6 Misconception Six: Uploading rules via Ruler API is production best practice

Incorrect.

API upload is suitable for learning and testing.

Production should use Git, Helm, ConfigMap, or GitOps to manage rules.

---

## Thirty-FiveI don't know.Summary

This document completes a Loki log alerting practical guide.

Core chain:

    Pod logs
      ↓
    Alloy
      ↓
    Loki
      ↓
    Ruler periodically executes LogQL
      ↓
    Alerting Rule judges threshold
      ↓
    AlertManager receives alerts
      ↓
    Grouping / deduplication / routing / notification

Alert types implemented in this document:

    Excessive ERROR logs
    Excessive JSON 5xx logs
    Excessive timeout logs
    CUDA OOM log detection

The key to Loki log alerts is not just "triggering", but:

    Reasonable query scope
    Reasonable threshold
    Reasonable for duration
    Suitable labels for routing
    Annotations with context
    runbook_url available
    AlertManager can converge
    Alerts can be handled
    Noise can be managed

Production recommendations:

    Mainly use Prometheus metric alerts.
    Use Loki log alerts as supplementary.
    Alert on logs with strong fault semantics.
    Do not alert on ordinary INFO/WARN logs.
    Do not blindly alert on global error keywords.
    Alerting rules must be Git-managed.
    Every alert must have owner and Runbook.

Next article will enter:

    12-Loki Production Governance: Log Volume - Retention - Rate Limiting - Security

Key learning points:

    Log volume governance
    retention period
    limits_config restrictions
    Query scope control
    Write rate limiting
    Label cardinality governance
    Tenant isolation
    Sensitive information protection
    Loki production security baseline

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Alerting and recording rules:
  https://grafana.com/docs/loki/latest/alert/

- Manage recording rules:
  https://grafana.com/docs/loki/latest/operations/recording-rules/

- Loki HTTP API - Ruler:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/#ruler

- Loki Configuration:
  https://grafana.com/docs/loki/latest/configure/

- Query Loki:
  https://grafana.com/docs/loki/latest/query/

- LogQL Reference:
  https://grafana.com/docs/loki/latest/query/query_reference/

- Metric queries:
  https://grafana.com/docs/loki/latest/query/metric_queries/

- Query best practices:
  https://grafana.com/docs/loki/latest/query/bp-query/

- AlertManager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- AlertManager Concepts:
  https://prometheus.io/docs/alerting/latest/alertmanager/

- prometheus-community AlertManager Helm Chart:
  https://github.com/prometheus-community/helm-charts/tree/main/charts/alertmanager

- Grafana Alerting:
  https://grafana.com/docs/grafana/latest/alerting/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/