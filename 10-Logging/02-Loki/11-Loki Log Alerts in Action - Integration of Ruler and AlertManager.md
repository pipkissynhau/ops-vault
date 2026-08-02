# 11-Loki Log Alerts in Action: Integration of Ruler and AlertManager

## Document Description

This article is the eleventh part of a dedicated study on Loki, focusing on systematically understanding Loki's log alert capabilities. It covers topics such as Loki Ruler, LogQL alert rules, integration with AlertManager, error log alerts, 5xx alerts, timeout alerts, CUDA OOM alerts, and how to implement log alerts in Kubernetes/SRE scenarios.

What has been completed previously:

    01-Loki Basics and Experimental Environment Setup
    02-Loki Architecture and Component Responsibilities
    03-Loki Deployment Modes and Experimental Selection
    04-Helm-Based Deployment of Loki in Standalone Mode
    05-Connecting Loki to MinIO for Object Storage
    06-Grafana-Alloy for Collecting K8S-Pod Logs
    07-Loki Tag Design and High基数 Issues
    08-Basic LogQL Queries: Searching for Namespace-Pod-Container Logs
    09-Advanced LogQL Queries: json-logfmt-regexp-unwrap
    10-Integrating Loki with Grafana for Log Dashboards

In Article 10, the log query capabilities of Loki were integrated into the Grafana Dashboard.

This article continues by upgrading some stable LogQL queries into log alert rules.

It addresses the following key questions:

- What is Loki Ruler?
- What is the relationship between Loki Ruler and Prometheus Rule?
- What are the differences between Loki log alerts and Prometheus metric alerts?
- What types of logs are suitable for alerts?
- What types of logs are not suitable for alerts?
- How to deploy or verify AlertManager?
- How to configure Loki Ruler to send alerts to AlertManager?
- How to write Loki alerting rules?
- How to load rules through the Loki Ruler API?
- How to manage rules using ConfigMap/Helm values?
- How to verify whether Ruler has loaded rules?
- How to confirm whether AlertManager has received alerts?
- How to simulate ERROR log alerts?
- How to simulate 5xx log alerts?
- How to simulate timeout log alerts?
- How to simulate CUDA OOM log alerts?
- How to design alert tags, grouping, suppression, routing, and Runbooks?
- What are the risks and governance requirements for Loki log alerts in production environments?

---

## Tags

#Loki #LogQL #Ruler #AlertManager #Log Alerts #Grafana #Kubernetes #SRE #Observability #Alert Governance #Runbook #Pod Logs

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/11-Loki Log Alerts in Action: Integration of Ruler and AlertManager.md

---

## I. Experimental Objectives

After completing this article, you should be able to:

    1. Understand the role of Loki Ruler.
    2. Distinguish between Loki log alerts and Prometheus metric alerts.
    3. Comprehend the role of AlertManager in the log alert workflow.
    4. Deploy or verify that AlertManager is functional.
    5. Configure Loki Ruler to connect with AlertManager.
    6. Write Loki alerting rules.
    7. Load Loki log alert rules.
    8. Check the status of Loki Ruler rules.
    9. Simulate ERROR logs triggering alerts.
    10. Simulate 5xx logs triggering alerts.
    11. Simulate timeout logs triggering alerts.
    12. Simulate CUDA OOM logs triggering alerts.
    13. View Loki alerts in AlertManager.
    14. Test receiving alerts through webhook.
    15. Master methods for troubleshooting issues where log alerts do not trigger.
    16. Understand how to manage false positives, alert storms, and duplicate notifications for log alerts.
    17. Develop production-grade guidelines for Loki log alerts.

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

Components already deployed:

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

Must be confirmed:

    [ ] Loki is running normally.
    [ ] Loki Gateway is accessible.
    [ ] Loki has been connected to MinIO or at least has the capability to store rules.
    [ ] The `loki ruler.enable_api` setting is enabled.
    [### Deadline exceeded
CUDA out of memory
OOM
License expired
Auth failed (severe exception)
Status >= 500
Payment failed
Data corruption
Backup failed
Replica sync failed

These logs typically:

- Have a clear meaning indicating an exception.
- Occur infrequently.
- Are strongly related to business failures.
- Require manual attention.
- Can be used to create processing runbooks.

### 5.2 Logs Not Suitable for Immediate Alerts

Not suitable for:

- Ordinary INFO messages
- Ordinary WARN messages
- Healthy check 404 responses
- User input errors
- Expected business failures
- Expected errors
- Errors that can be ignored
- Debug logs
- Each 4xx access log entry
- Short-term, occasional timeouts
- Known exceptions that can be disregarded

These logs may cause:

- Excessive alert noise
- Alert fatigue among operators
- Operators losing trust in alerts
- Genuine failures being overshadowed

### 5.3 Log Alerts Must Have Thresholds

Do not trigger an alert immediately just because one error message appears.

A more reasonable approach is to set thresholds, such as:

- More than 10 error logs in the last 5 minutes
- More than 20 5xx errors for a specific app in the last 5 minutes
- More than 5 timeouts in the last 10 minutes
- More than 0 CUDA OOM events in the last 5 minutes (only applicable to ai-prod)
- More than 0 panic events in the last 5 minutes

Different types of errors should have different threshold settings.

---

## VI. Alert Chain Architecture

### 6.1 Architecture Diagram

    Pod stdout/stderr
      ↓
    Alloy
      ↓
    Loki
      ↓
    Ruler periodically executes LogQL queries
      ↓
    Alert conditions are met
      ↓
    AlertManager
      ↓
    Receiver
      ↓
    Webhook/Email/WeCom/FlyBook/DingTalk

### 6.2 Experimental Chain in This Document

This document uses:

    app-demo test application
      ↓
    Alloy for log collection
      ↓
    Loki for log storage
      ↓
    Loki Ruler for rule computation
      ↓
    AlertManager for receiving alerts
    AlertManager Web UI for viewing alerts

Optional components include:

    webhook.site/self-built webhook receiver/alertmanager-webhook-demo

This document primarily verifies the functionality using the AlertManager Web UI.

---

## VII. Deploying or Confirming AlertManager

If AlertManager has already been deployed through the kube-prometheus-stack, you can skip this section.

### 7.1 Checking the monitoring Namespace

    kubectl get ns monitoring

If it doesn't exist:

    kubectl create namespace monitoring

### 7.2 Checking if AlertManager is Already Present

    kubectl get pods -n monitoring | grep -i alertmanager

    kubectl get svc -n monitoring | grep -i alertmanager

Record the Service name if it exists.

Common Service names might include:

    alertmanager
    kube-prometheus-stack-alertmanager
    prometheus-kube-prometheus-alertmanager

Refer to the actual environment for the specific name.

### 7.3 Deploying a Standalone AlertManager Using prometheus-community/alertmanager

Add the Helm repository:

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

    Update the Helm repository:

    helm repo update

Search for the chart:

    helm search repo prometheus-community/alertmanager --versions | head -20

Record the version:

    ALERTMANAGER_CHART_VERSION=<actual version>

Create the values file:

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

Explanation:

    This example uses a webhook receiver.
    If there is no webhook-demo available, you can use an empty receiver or check through the AlertManager Web UI.
    In a production environment, configure WeCom, FlyBook, DingTalk, email, or a unified alert platform.

Install the chart:

    helm install alertmanager prometheus-community/alertmanager \
      -n monitoring \
      --version <ALERTMANAGER_CHART_VERSION> \
      -f values-alertmanager.yaml

Check the installation:

    helm list -n monitoring

    kubectl get pods -        - name: http
          port: 8080
          targetPort: 8080

Application:

    kubectl apply -f alertmanager-webhook-demo.yaml

View:

    kubectl get pods -n monitoring | grep webhook

    kubectl get svc -n monitoring | grep webhook

### 8.3 Viewing Alarms Received by the Webhook

    kubectl logs deploy/alertmanager-webhook-demo -n monitoring -f

If AlertManager sends an alarm, you will see the HTTP request content in the logs.

---

## Section Nine: Configuring Loki Ruler to Connect with AlertManager

### 9.1 Viewing Current Loki Values

    helm get values loki -n logging -a > values-loki-current.yaml

Search for Ruler:

    grep -n "ruler" values-loki-current.yaml

You can also view the default values:

    helm show values grafana-community/loki \
      --version <CHART_VERSION> \
      > values-loki-default.yaml

Search:

    grep -n "ruler" values-loki-default.yaml

    grep -n "alertmanager" values-loki-default.yaml

### 9.2 Core Configuration for Loki Ruler

In the Loki configuration, you usually need to specify:

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
        The address of the AlertManager.

    enable_api:
        Allows management of rules through the Ruler API.

    storage:
        The backend for storing rules.

    rule_path:
        The local directory where Ruler rules are stored.

### 9.3 Example Helm values Configuration

Create or modify:

    values-loki-monolithic-minio-ruler.yaml

Add or confirm on top of the existing Loki values:

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

    The above snippet needs to be merged into the complete values from your Chapter 05.
    Do not simply copy this section and overwrite the entire values file.
    Ensure that field names match those in the current Loki Helm Chart version.
    Different chart versions may have variations, so verify with the helm template first.

### 9.4 Confirming the AlertManager Service Address

If the AlertManager Service name is alertmanager:

    http://alertmanager.monitoring.svc.cluster.local:9093

If it is kube-prometheus-stack:

    http://kube-prometheus-stack-alertmanager.monitoring.svc.cluster.local:9093

Or:

    http://alertmanager-operated_monitoring.svc.cluster.local:9093

To confirm:

    kubectl get svc -n monitoring | grep -i alertmanager

Test from the logging Namespace:

    kubectl run curl-alertmanager-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -s http://alertmanager.monitoring.svc.cluster.local:9093/-/ready

To exit:

    exit

### 9.5 Pre-checking with Helm Template

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio-ruler.yaml \
      > loki-ruler-rendered.yaml

Check:

    grep -n "alertmanager" loki-ruler-rendered.yaml

    grep -n "enable_api" loki-ruler-rendered.yaml

    grep -n "ruler" loki-ruler-rendered.yaml

### 9.6 Upgrading Loki

Back up the current values:

    helm get values loki -n logging -a > backup-values-loki-before-ruler.yaml

Perform the upgrade:

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic-minio-ruler```markdown
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
      summary: "Too many application error logs"
      description: "In namespace {{ $labels.namespace }} and app {{ $labels.app}}, the number of error logs in the last 5 minutes exceeds the threshold."
      runbook_url: "https://example.com/runbooks/loki-app-error-logs"

### 11.2 Field Description

    groups:
        Rule group.

    name:
        Rule group name.

    interval:
        Interval between rule executions.

    rules:
        List of rules.

    alert:
        Alert name.

    expr:
        LogQL expression.

    for:
        Duration after which the alert is triggered if the condition persists.

    labels:
    Alert labels used for grouping, routing, and setting severity levels.

    annotations:
    Alert description, including notification content and reference to a Runbook.

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
```

### 11.4 Annotations Design

Recommended annotations:

    summary
    description
    runbook_url
    dashboard_url

Example:

    annotations:
      summary: "Too many pod error logs"
      description: "In namespace {{ $labels.namespace }} and pod {{ $labels.pod}}, the number of error logs in the last 5 minutes exceeds the threshold."
      runbook_url: "https://example.com/runbooks/pod-error-logs"
      dashboard_url: "https://grafana.example.com/d/loki-pod-logs"

---

## Section XII: Creating the First ERROR Log Alert Rule

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
              summary: "Too many error logs in app-demo"
              description: "In namespace {{ $labels.namespace }} and app {{ $labels.app}}, the number of error logs in the last 5 minutes exceeds 5."
              runbook_url: "https://example.com/runbooks/app-demo-error-logs"

### 12.2 Rule Description

Expression:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    ) > 5

Meaning:

    Query for logs in the app-demo namespace that contain "error", "exception", "panic", or "failed" within the last 5 minutes.
    Group and count these logs by namespace and app.
    If the count exceeds 5, trigger an alert.
    The alert is triggered after a duration of 1 minute.

### 12.3 Threshold Description

Learning environment:

    > 5

Production environment:

    Set based on the volume of business logs.
    Different apps should have different thresholds.
    It is not recommended to use a unified threshold for all apps.

---
## Section XIII: Loading Rules via the Ruler API

### 13.1 API Path

The Loki Ruler API uses the concept of namespace.

Example path:

    /loki/api/v1/rules/{namespace}

Here, "namespace" refers to the naming space of the rule file, not a Kubernetes Namespace.

Recommended usage:

    app-demo

### 13.2 Uploading Rules

Execute:

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

If there is no output, it usually indicates success.

In case of an error, troubleshoot according to the prompts.

### 13.3 Viewing Rules

```bash
--data-urlencode 'query={namespace="app-demo", app="alert-error-demo"} |= "ERROR"' \
--data-urlencode 'limit=20' | jq

### 14.3 Waiting for the Rule to Execute

Wait:

    1 to 2 minutes

Check AlertManager:

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

Access:

    http://127.0.0.1:9093

Expected outcome:

    AppDemoErrorLogsTooMany should be in the firing or pending state.

### 14.4 Viewing Loki Ruler Logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "AppDemoErrorLogsTooMany|ruler|alert|error"

### 14.5 Checking Webhook Receipts

If the webhook demo is deployed:

    kubectl logs deploy/alertmanager-webhook-demo -n monitoring --tail=200

You should see the alert payload pushed by AlertManager.

---

## Section Fifteen: JSON 5xx Log Alerts

### 15.1 Creating Rules

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
        summary: "Too many JSON 5xx logs"
        description: "In namespace {{ $labels.namespace }}, app {{ $labels.app}}, more than 5 logs with status >= 500 have been recorded in the last 5 minutes."
        runbook_url: "https://example.com/runbooks/json-5xx-logs"

Example of a complete set of rules:

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
              description: "In namespace {{ $labels.namespace }}, app {{ $labels.app}}, more than 5 error logs have been recorded in the last 5 minutes."
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
              summary: "Too many JSON 5xx logs"
              description: "In namespace {{ $labels.namespace }}, app {{ $labels.app}}, more than 5 logs with status >= 500 have been recorded in the last 5 minutes."
              runbook_url: "https://example.com/runbooks/json-5xx-logs"

### 15.2 Uploading Rules

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

Check the rules uploaded:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

### 15.3 Triggering 5xx Logs

If json-log-demo has been deployed, you can delete and re-deploy it:

    kubectl delete job json-log-demo -n app-demo

    kubectl apply -f json-log-demo.yaml

Check the logs:

    kubectl get pod -n app-demo | grep json-log-demo

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

### 15.4 Confirming### 16.2 Simulating Timeout Logs

    for i in $(seq 1 6); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\"",
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

### 16.3 Query Confirmation

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="alert-timeout-demo"} |~ "(?i)timeout|deadline exceeded"' \
      --data-urlencode 'limit=20' | jq

---

## Section Seventeen: CUDA Out of Memory Logs Alarms

### 17.1 Applicable Scenarios

Suitable for GPU/AI workloads.

Typical logs:

    CUDA out of memory
    RuntimeError: CUDA error
    out of memory
    CUBLAS_STATUS_ALLOC_FAILED
    NCCL error

### 17.2 Rules

In production, it is recommended to limit the namespace, for example:

    ai-prod
    gpu-prod
    model-serving

For experiments, `app-demo` can be used.

Rules:

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
        description: "GPU/CUDA-related error logs occurred in namespace={{ $labels.namespace }}, app={{ $labels.app }}, pod={{ $labels.pod }}"
        runbook_url: "https://example.com/runbooks/cuda-oom"

### 17.3 Simulating CUDA OOM Logs

    for i in $(seq 1 3); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\"",
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

### 17.4 Query Confirmation

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="gpu-worker-demo"} |~ "(?i)cuda out of memory|CUDA error|out of memory"' \
      --data-urlencode 'limit=20' | jq

### 17.5 Production Notes

For CUDA OOM alarms, it is recommended to combine the following:

    DCGM Exporter GPU memory metrics
    Pod OOMKilled metrics
    Application logs
    GPU Node resource allocation
    Requests/limits
    Scheduling strategies
    Model batch size
    Inference concurrency
    Memory fragmentation

Do not rely solely on log alarms.

---

## Section Eighteen: Example of a Complete Rules File

Create:

    loki-rules-app-demo.yaml

Complete content:

    groups:
      - name: app-demo-log-alerts
        interval: 1m
        rules:
          - alert: AppDemoErrorLogsTooMany
            expr: |
              sum by (namespace, app) (
                count_over_time(
                  {namespace="app```markdown
summary: "Excessive JSON 5xx logs"
description: "In the namespace {{ $labels.namespace }} and app {{ $labels.app}}, there are more than 5 status codes >=500 logs in the last 5 minutes."
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
    summary: "Too many timeout logs"
    description: "In the namespace {{ $labels.namespace }} and app {{ $labels.app}}, there are more than 3 timeout-related logs in the last 5 minutes."
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
    description: "In the namespace {{ $labels.namespace }} and app {{ $labels.app}}, GPU/CUDA-related error logs have occurred in pod {{ $labels.pod }}."
    runbook_url: "https://example.com/runbooks/cuda-oom"

Upload:
```bash
curl -s -X POST \
  -H "Content-Type: application/yaml" \
  --data-binary @loki-rules-app-demo.yaml \
  http://127.0.0.1:3100/loki/api/v1/rules/app-demo
```

View:
```bash
curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq
```

---

## Section 19: Managing Rules Using ConfigMap

### 19.1 Why Manual API Uploads Are Not Recommended for Long-Term Use

API uploads are suitable for learning and temporary testing purposes.

For production environments, it is recommended to use:

- Git to manage rule files
- Helm values to manage rules
- ConfigMap to manage rules
- GitOps for rule synchronization
- Change approval processes
- Change rollback capabilities

### 19.2 ConfigMap Example

Create a file:
```yaml
loki-rules-configmap.yaml
```
Content:
```yaml
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
              summary: "Excessive error logs in app-demo"
              description: "In the namespace {{ $labels.namespace }} and app {{ $labels.app}}, there are more than 5 error logs in the last 5 minutes."
              runbook_url: "https://example.com/runbooks/app-demo-error-logs"
```
Apply the configuration:
```bash
kubectl apply -f loki-rules-configmap.yaml
```

### 19.3 Mounting the ConfigMap to Loki

You need to mount this ConfigMap in the Loki Helm values and set the correct rule path.

Since field names may vary depending on the Helm Chart version, first check:
```bash
helm show values grafana-community/loki --version <CHART_VERSION> > values-loki-default.yaml
```
Search for the following fields:
```bash
grep -n "extraVolumes" values-loki-default.yaml
grep -n "extraVolumeMounts" values-loki-default.yaml
grep -n "ruler" values-loki-default.yaml
```

Common configurations include:
```yaml
extraVolumes:
  - name:```markdown
group_interval:
    Interval between repeated notifications for new alarms within the same group.

repeat_interval:
    Interval between repeated notifications for unresolved alarms.

### 20.2 Routing by Severity

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

### 20.3 Routing by Team

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

### 20.4 Requirements for Loki Alert Labels

To ensure correct routing by AlertManager, Loki rule labels must include:

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

## Chapter Twenty-One: Alarm States: Inactive, Pending, Firing

### 21.1 Inactive

This indicates that:

The current rule conditions are not met.

### 21.2 Pending

This indicates that:

The current rule conditions are met, but the condition has not been sustained for the specified `for` time period.

Example:

for: 2m

If the conditions are just met for 30 seconds, the state will be pending.

### 21.3 Firing

This indicates that:

The conditions have been met and sustained for the specified `for` time period.

At this point, the Ruler will send the alarm to AlertManager.

### 21.4 Why Use `for`

The `for` time period helps avoid false alarms caused by temporary fluctuations in data.

Example:

If there is a single error that occurs instantaneously, it may not be necessary to trigger an alarm.
However, if the error persists for 1 minute and exceeds the threshold, it would be more meaningful to generate an alarm.

---

## Chapter Twenty-Two: Alarm Recovery

### 22.1 Conditions for Alarm Recovery

An alarm will recover when the specified `expr` condition is no longer met.

Example:

If the number of error logs in the last 5 minutes is <= 5, then the alarm will change from firing to resolved.

### 22.2 send_resolved

In the AlertManager receiver settings, if `send_resolved` is set to `true`, notifications will also be sent when an alarm recovers.

Production recommendations:

- Send recovery notifications for critical alarms.
- For low-priority alarms, decide whether to send notifications based on specific scenarios.

### 22.3 Delay in Log Alarm Recovery

Due to the use of a time window, such as [5 minutes], even if error logging stops, the recent 5-minute window may still contain old errors.

Therefore, there will be a natural delay in alarm recovery.

---
## Chapter Twenty-Three: Log Alert Troubleshooting Process

### 23.1 When an Alarm Does Not Trigger

The troubleshooting sequence is as follows:

1. Check whether the LogQL query itself returns any results.
2. Verify whether the expression returns a numerical value.
3. Confirm whether the threshold is set too high.
4. Ensure that the `for` time period has not yet elapsed.
5. Check whether the Ruler is enabled.
6. Verify whether the rule has been successfully loaded.
7. Confirm whether the Ruler can access Loki data.
8. Check whether the Ruler can connect to AlertManager.
9. Verify whether AlertManager has received the alarm.
10. Ensure that AlertManager's routing settings match the receiver.

### 23.2 Manually Execute LogQL First

For example:

```sql
sum by (namespace, app) (
  count_over_time(
    {namespace="app-demo"}
      |~ "(?i)error|exception|panic|failed" [5m]
  )
)
```

This query can be executed in Grafana Explore or through an API.

### 23.3 Verify Whether the Rule Has Been Loaded

Run the following command:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq
```

### 23.4 Check Loki Ruler Logs

Use the following command to view logs related to the Ruler:

```bash
kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "ruler|rule|alertmanager|failed|error"
```

### 23.5 Check AlertManager UI

      -n logging \
      -- sh

Inside the container:

    curl -s http://alertmanager.monitoring.svc.cluster.local:9093/-/ready

Exit:

    exit

### 26.3 Viewing Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "alertmanager|notify|ruler|error"

### 26.4 Viewing AlertManager Logs

    kubectl logs <alertmanager-pod-name> -n monitoring --tail=500 | grep -Ei "notify|webhook|error|warn"

---

## Chapter Twenty-seven: Common Fault Four: Too Many Alerts

### 27.1 Causes

    The threshold is too low.
    The search range is too broad.
    group_by is too detailed.
    Grouping by pod results in one alert per pod.
    The log keywords are too general.
    The application continuously reports errors.
    Healthy checks or expected errors are also matched.
    The repeat_interval is too short.

### 27.2 Solutions

    Increase the threshold.
    Extend the for period.
    Narrow down the namespace/app scope.
    Exclude expected errors.
    Use level="error" instead of the keyword error.
    Group by app, not pod.
    Adjust AlertManager's group_by setting.
    Increase the repeat_interval.
    Use silence for known maintenance periods.

### 27.3 Example: Excluding Expected Errors

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed"
          != "expected error"
          != "ignore" [5m]
      )
    ) > 10

---

## Chapter Twenty-eight: Common Fault Five: Alerts That Do Not Disappear

### 28.1 Possible Reasons

    There are still old logs within the time window.
    The application continues to produce errors.
    The search expression is too broad.
    AlertManager's repeat_interval causes duplicate notifications.
    The AlertManager UI has not updated yet.
    There is a delay in Ruler status synchronization.

### 28.2 Troubleshooting

Execute the current expression:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

If the result is still above the threshold, the alerts will not disappear.

### 28.3 Solutions

    Wait for the time window to pass naturally.
    Stop the source of error logs.
    Adjust the threshold and time window.
    Verify if the app is still experiencing errors.

---

## Chapter Twenty-nine: Log Alert Design Standards

### 29.1 Alert Name Standards

Recommendations:

    AppErrorLogsTooMany
    AppJson5xxLogsTooMany
    AppTimeoutLogsTooMany
    CudaOomLogsDetected
    DatabaseConnectionFailedLogsTooMany

Avoid using:

    ErrorAlert
    TestAlert
    LokiAlert1
    AppProblem

### 29.2 Severity Standards

Suggestions:

    critical:
        Requires immediate action, may affect business availability.

    warning:
        Requires attention but does not necessarily need to interrupt operations immediately.

    info:
       仅为提示或观察用途, not recommended for strong notification channels.

### 29.3 Labels Standards

Required:

    severity
    source
    team
    category

Suggestions:

    environment
    service
    runbook

Example:

    labels:
      severity: warning
      source: loki
      team: sre
      category: logs

### 29.4 Annotations Standards

Required:

    summary
    description

Suggestions:

    runbook_url
    dashboard_url

Example:

    annotations:
      summary: "Too many application error logs"
      description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has exceeded the threshold for error logs in the last 5 minutes"
      runbook_url: "https://example.com/runbooks/app-error-logs"
      dashboard_url: "https://grafana.example.com/d/k8s-loki-logs"

### 29.5 Threshold Standards

Do not use a global uniform threshold.

Set thresholds based on:

    app type
    amount of logs
    business importance
    historical baseline
    nature of the error

---

## Chapter Thirty: Log Alerts and Runbooks

### 30.1 Each Alert Must Provide Answers to the Following Questions

    What is the issue?
    What is the impact?
    Which dashboard should be checked?
    What kubectl commands need to be executed?
    Which team should be contacted?
    Under what circumstances should it be escalated?
    How can it be resolved?
    How can false positives be mitigated?

### 30.2    kubectl get svc -n monitoring | grep -i alertmanager

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

    curl -s http://127.0.0.1:9093/-/ready

Verification:

    [ ] AlertManager Pod is running
    [ ] The AlertManager Service exists
    [ ] The /-/ready response returns OK
    [ ] The AlertManager UI is accessible

### 32.2 Task Two: Configuring Loki Ruler

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

Verification:

    [ ] The Helm template execution was error-free
    [ ] The Helm upgrade was successful
    [ ] The Loki Pod is running
    [ ] There are no obvious errors in the Loki logs related to the Ruler
    [ ] The /loki/api/v1/rules endpoint is accessible

### 32.3 Task Three: Uploading Rules

Execution:

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

Verification:

    [ ] The rules were successfully uploaded
    [ ] The rules can be retrieved
    [ } The rule group name is correct
    [ ] The alarm names are correct

### 32.4 Task Four: Triggering ERROR Alarms

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
                \"namespace\": \"app-demo\"",
                \"app\": \"alert-error-demo\"
              },
              \"values\": [
                [\"${TS}\", \"ERROR: Database connection failed, test alert line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

Verification:

    [ ] Loki can detect the ERROR logs
    [ ] The AppDemoErrorLogsTooMany alarm appears in AlertManager
    [ ] The webhook demo receives the alarm notifications

### 32.5 Task Five: Triggering timeout Alarms

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
                \"namespace\": \"app-demo\"",
                \"app\": \"alert-timeout-demo\"
              },
              \"values\": [
                [\"${TS}\", \"ERROR: Upstream timeout occurred, deadline exceeded, test alert line ${i}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 1
    done

Verification:

    [ ] Alarms related to timeouts appear in AlertManager

### 32.6 Task Six: Triggering CUDA OOM Alarms

Execution:

    for i in $(seq 1 3); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\",
                \"namespace\": \"app-demo\"",
                \"app\": \"gpu-worker-demo\",
                \"pod\": \"gpu-worker-demo-001\"
              },
              \"values\": [
                [\"${TS}\", \"RuntimeError: CUDA out of memory. Attempted to allocate 2.00 GiB on GPU 0, test line ${i}\"]
              ]
            }
          ]
        }" >/devLog alerts are used to provide additional context regarding specific anomalies.

### 34.3 Misunderstanding Three: All applications should use the same threshold

Wrong.

Since the amount of log data varies across applications, thresholds should be set based on each application's baseline.

### 34.4 Misunderstanding Four: It's okay if alerts don't have a Runbook

Wrong.

Alerts without a Runbook are difficult to manage effectively.

Every production alert should come with clear instructions for handling it.

### 34.5 Misunderstanding Five: The more detailed the Pod grouping, the better

Not necessarily.

Grouping by Pod can result in a large number of alerts.

In production, it is usually better to group logs based on:

    namespace
    app
    severity

First, and then drill down to Pod level for temporary troubleshooting.

### 34.6 Misunderstanding Six: Using the Ruler API to upload rules is the best practice in production

Wrong.

Using the API is suitable for learning and testing purposes.

In production, it is recommended to use Git, Helm, ConfigMap, or GitOps to manage rules.

---

## Chapter Thirty-Five: Summary

This article covers the practical application of Loki log alerts.

Key steps include:

    Pod logs
      ↓
    Alloy
      ↓
    Loki
      ↓
    Ruler periodically executes LogQL queries
      ↓
    Alerting Rules determine thresholds
      ↓
    AlertManager receives and processes alerts
      ↓
    Alerts are then grouped, deduplicated, routed, and notifications are sent

Types of alerts implemented in this article include:

    Excessive ERROR logs
    Too many JSON 5xx errors
    Frequent timeout errors
    CUDA OOM detection

The key to successful Loki log alerts is not just about triggering them, but also ensuring that:

    The query scope is appropriate
    The thresholds are reasonable
    The timing for checking is suitable
    Labels facilitate proper routing
    Annotations provide relevant context
    The runbook_url is accessible
    AlertManager can handle the alerts effectively
    Alerts are actually addressed
    Noise in the alerts is managed properly

In production, it is recommended to:

    Prioritize Prometheus metric-based alerts
    Use Loki log alerts as a supplement
    Set up alerts for logs that indicate serious issues
    Avoid issuing alerts for routine INFO/WARN messages
    Do not trigger alerts blindly based on the global "error" keyword
    Manage alert rules using Git
    Ensure each alert has an owner and a corresponding Runbook

In the next article, we will explore:

    12-Loki Production Governance Practices: Log Volume, Retention Period, Throttling, Security

Key topics to learn include:

    Managing log volumes
    Setting retention periods
    Implementing limits
    Controlling query scopes
    Enforcing write throttling
    Managing label bases
    Ensuring tenant isolation
    Protecting sensitive information
    Establishing basic security measures for Loki in production environments

---

## References

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Alerting and Recording Rules:
  https://grafana.com/docs/loki/latest/alert/

- Manage Recording Rules:
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

- Query Best Practices:
  https://grafana.com/docs/loki/latest/query/bp-query/

- AlertManager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- AlertManager Concepts:
  https://prometheus.io/docs ALERTing/latest/alertmanager/

- prometheus-community AlertManager Helm Chart:
  https://github.com/prometheus-community/helm-charts/tree/main/charts/alertmanager

- Grafana Alerting:
  https://grafana.com/docs/grafana/latest/alerting/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/