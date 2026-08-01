# 15-Loki Comprehensive Practical: Building a K8S Log Platform Loop from Scratch

## Document Description

This is the fifteenth article in the Loki specialized learning series, used to connect the previous Loki, MinIO, Grafana Alloy, Grafana, LogQL, Ruler, AlertManager, retention, limits_config, production troubleshooting, and other content into a complete Kubernetes log platform loop practical implementation.

Previous articles have been completed:

    01-Loki Basic Understanding and Experiment Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experiment Selection
    04-Loki Single-Instance Helm Deployment Practical
    05-Loki Object Storage Access MinIO Practical
    06-Grafana-Alloy Collecting K8S-Pod Logs Practical
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practical: Namespace-Pod-Container Log Retrieval
    09-LogQL Advanced Query Practical: json-logfmt-regexp-unwrap
    10-Grafana Integration with Loki and Log Dashboard Practical
    11-Loki Log Alert Practical: Ruler and AlertManager Integration
    12-Loki Production Governance Practical: Log Volume-Retention Period-Limiting-Security
    13-Loki Performance and High Availability Practical: Simple-Scalable Mode Introduction
    14-Loki Common Fault Diagnosis Practical: Collection-Write-Query-Storage-Alert

This article no longer focuses on a single component, but starts from scratch to build a verifiable, queryable, alertable, governable, and troubleshootable Kubernetes log platform loop.

Complete chain is as follows:

    Kubernetes Pod stdout / stderr
      ↓
    Grafana Alloy DaemonSet
      ↓
    Loki Gateway
      ↓
    Loki
      ↓
    MinIO Object Storage
      ↓
    Grafana Explore / Dashboard
      ↓
    Loki Ruler
      ↓
    AlertManager
      ↓
    Runbook / On-Call Handling

This article focuses on answering the following questions:

- How to plan a Kubernetes log platform from scratch;
- How to prepare Namespace, MinIO, Bucket;
- How to deploy Loki;
- How to configure Loki to access MinIO;
- How to configure retention;
- How to configure limits_config;
- How to deploy Grafana Alloy;
- How to collect Kubernetes Pod logs;
- How to verify Alloy → Loki write chain;
- How to deploy test applications to generate logs;
- How to query Namespace / App / Pod / Container logs via LogQL;
- How to deploy Grafana and add Loki Data Source;
- How to create a basic Dashboard;
- How to create log alert rules;
- How to configure Ruler to connect AlertManager;
- How to simulate ERROR / 5xx / timeout alerts;
- How to perform log volume governance;
- How to perform high cardinality checks;
- How to perform sensitive information checks;
- How to organize production Runbook;
- How to complete final acceptance.

---

## Tags

#Loki #Grafana #GrafanaAlloy #MinIO #AlertManager #LogQL #Kubernetes #PodLog #LogPlatform #Observation #It'sALogCall. #LogGovernance #SRE #Runbook #ProductionOperations

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/15-Loki Comprehensive Practical: Building a K8S Log Platform Loop from Scratch.md

---

## One, Comprehensive Practical Goals

After completing this article, you should be able to independently build a basic Kubernetes log platform loop:

    1. Prepare logging, monitoring, minio, app-demo namespaces.
    2. Deploy MinIO as Loki object storage.
    3. Create the required Bucket for Loki.
    4. Deploy Loki using Helm.
    5. Configure Loki to use MinIO.
    6. Configure Loki retention.
    7. Configure Loki limits_config.
    8. Verify Loki /ready, /metrics, push, query_range.
    9. Deploy Grafana Alloy using Helm.
    10. Configure Alloy to collect Kubernetes Pod logs.
    11. Configure Alloy to add namespace, pod, container, node, app, team, environment, cluster labels to logs.
    12. Deploy test applications nginx-demo, json-log-demo, logfmt-log-demo.
    13. Query Pod logs via LogQL.
    14. Deploy Grafana.
    15. Add Loki Data Source.
    16. Create a basic log Dashboard.
    17. Deploy AlertManager.
    18. Configure Loki Ruler.
    19. Create log alert rules.
    20. Simulate ERROR, 5xx, timeout alerts.
    21. Check high log volume sources.
    22. Check high cardinality labels.
    23. Check sensitive information.
    24. Organize production troubleshooting Runbook.
    25. Complete final acceptance of the log platform loop.

---

## Two, Final Architecture

### 2.1 Overall Architecture Diagram /think

```
+-----------------------------+
| Kubernetes Applications     |
|                             |
| nginx-demo                  |
| json-log-demo               |
| logfmt-log-demo             |
| stdout / stderr             |
+--------------+--------------+
                   |
                   v
+-----------------------------+
| Grafana Alloy DaemonSet     |
|                             |
| discovery.kubernetes        |
| discovery.relabel           |
| loki.source.kubernetes      |
| loki.process                |
| loki.write                  |
+--------------+--------------+
                   |
                   v
+-----------------------------+
| Loki Gateway                |
| /loki/api/v1/push           |
| /loki/api/v1/query_range    |
+--------------+--------------+
                   |
                   v
+-----------------------------+
| Loki                        |
|                             |
| ingestion                   |
| query                       |
| ruler                       |
| compactor                   |
| retention                   |
+--------------+--------------+
                   |
                   v
+-----------------------------+
| MinIO / S3 Compatible Store |
|                             |
| loki-chunks                 |
| loki-ruler                  |
| loki-admin                  |
+--------------+--------------+
                   |
                   v
+-----------------------------+
| Grafana                     |
|                             |
| Explore                     |
| Dashboard                   |
| Variables                   |
+--------------+--------------+
                   |
                   v
+-----------------------------+
| AlertManager                |
|                             |
| Grouping                    |
| Routing                     |
| Webhook / Mail / IM         |
+-----------------------------+

### 2.2 Data Flow

Log ingestion flow:

    Pod stdout / stderr
      ↓
    Alloy
      ↓
    Loki Gateway
      ↓
    Loki
      ↓
    MinIO

Log query flow:

    Grafana Explore / Dashboard
      ↓
    Loki Gateway
      ↓
    Loki Query
      ↓
    MinIO / Ingester
      ↓
    Return log results

Log alert flow:

    Loki Ruler
      ↓
    Regular LogQL execution
      ↓
    Threshold met
      ↓
    AlertManager
      ↓
    Notification channels

Troubleshooting loop:

    Prometheus / Grafana detects anomalies
      ↓
    Loki queries logs
      ↓
    AlertManager alert notification
      ↓
    Runbook processing
      ↓
    Post-mortem optimization of log standards

---

## Three. Experimental Environment Planning

### 3.1 Kubernetes Nodes

Experimental nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Container runtime:

    containerd

System:

    Ubuntu Server 22.04.5 LTS

Kubernetes:

    kubeadm cluster

CNI:

    Calico

### 3.2 Namespace Planning

Namespaces:

    minio:
        Deploy MinIO object storage.

    logging:
        Deploy Loki, Loki Gateway, and Grafana Alloy.
```

monitoring:
    Deploy Grafana, AlertManager.

app-demo:
    Deploy test application to generate logs.

Create namespaces:

    kubectl create namespace minio

    kubectl create namespace logging

    kubectl create namespace monitoring

    kubectl create namespace app-demo

If already exists, ignore the error.

Check:

    kubectl get ns

### 3.3 Component Version Strategy

Production recommendation:

    Fix Helm Chart version
    Fix image version
    Manage values files via Git
    Do not use latest
    Template with helm before changes
    Validate after changes
    Rollback capability

Placeholders used in this article:

    <LOKI_CHART_VERSION>
    <GRAFANA_CHART_VERSION>
    <ALLOY_CHART_VERSION>
    <ALERTMANAGER_CHART_VERSION>

Query via helm search repo before actual execution.

---

## Four. Prepare Helm Repository

### 4.1 Add Grafana Helm Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 4.2 Add prometheus-community Repository

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

    helm repo update

### 4.3 Query Chart

Loki:

    helm search repo grafana/loki --versions | head -20

Grafana:

    helm search repo grafana/grafana --versions | head -20

Alloy:

    helm search repo grafana/alloy --versions | head -20

AlertManager:

    helm search repo prometheus-community/alertmanager --versions | head -20

Record versions:

    LOKI_CHART_VERSION=<actual version>
    GRAFANA_CHART_VERSION=<actual version>
    ALLOY_CHART_VERSION=<actual version>
    ALERTMANAGER_CHART_VERSION=<actual version>

### 4.4 Export default values

    helm show values grafana/loki \
      --version <LOKI_CHART_VERSION> \
      > values-loki-default.yaml

    helm show values grafana/grafana \
      --version <GRAFANA_CHART_VERSION> \
      > values-grafana-default.yaml

    helm show values grafana/alloy \
      --version <ALLOY_CHART_VERSION> \
      > values-alloy-default.yaml

    helm show values prometheus-community/alertmanager \
      --version <ALERTMANAGER_CHART_VERSION> \
      > values-alertmanager-default.yaml

Note:

    Chart fields may differ across versions.
    Must reference current version default values before formal values writing.
    This article's example is for learning and methodology, must verify with actual versions before production.

---

## Five. Deploy MinIO Object Storage

### 5.1 MinIO Purpose

MinIO is used as Loki's S3-compatible object storage.

Used for storing:

    Loki chunk data
    Loki index related data
    Loki ruler rules
    Loki deletion request data

Experimental Bucket:

    loki-chunks
    loki-ruler
    loki-admin

### 5.2 Create MinIO Single-node Experiment YAML

Create file:

    minio-lab.yaml

Content:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: minio-data
      namespace: minio
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 50Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
  labels:
    app: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: registry.cn-hangzhou.aliyuncs.com/pub-syq/minio:RELEASE.2025-04-22T22-12-26Z
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          env:
            - name: MINIO_ROOT_USER
              value: minioadmin
            - name: MINIO_ROOT_PASSWORD
              value: minioadmin123
          ports:
            - name: api
              containerPort: 9000
            - name: console
              containerPort: 9001
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-data

---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio
spec:
  selector:
    app: minio
  ports:
    - name: api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001

Application:

    kubectl apply -f minio-lab.yaml

### 5.3 View MinIO

    kubectl get pods -n minio -o wide

    kubectl get svc -n minio

    kubectl get pvc -n minio

Wait for Pod Running.

### 5.4 Create Bucket

Run mc:

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Execute inside container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

    mc mb local/loki-chunks

    mc mb local/loki-ruler

    mc mb local/loki-admin

    mc ls local

Exit:

    exit

### 5.5 Verify MinIO

Verification commands:

    kubectl get pods -n minio -o wide

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

    kubectl run curl-minio-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside container:

    curl -I http://minio.minio.svc.cluster.local:9000

Exit:

    exit

Verification:

    [ ] MinIO Pod Running
    [ ] MinIO Service exists
    [ ] Endpoint is not empty
    [ ] loki-chunks has been created
    [ ] loki-ruler has been created
    [ ] loki-admin has been created
    [ ] logging Namespace can access MinIO

---

## SixI don't know.Deploy Loki

### 6.1 Loki Deployment Mode Selection

This article adopts the following comprehensive practical approach:

    Monolithic / Single-Instance Mode

Reasons:

    Experimentation is more stable.
    Lower resource consumption.
    Easier for closed-loop verification.
    We have already learned Simple Scalable.
    This article focuses on complete closed-loop, not large-scale high-availability architecture.

Production notes:

    Small-scale can start with single-instance mode for evaluation.
    Medium to large-scale should evaluate Microservices / Distributed.
    Simple Scalable has deprecation risks and is not recommended as a long-term production target.

### 6.2 Write Loki values

Create file:

    values-loki-lab.yaml

Content:

    loki:
      auth_enabled: false

      commonConfig:
        replication_factor: 1

schemaConfig:
        configs:
          - from: "2024-04-01"
            store: tsdb
            object_store: s3
            schema: v13
            index:
              prefix: loki_index_
              period: 24h

      storage:
        type: s3
        bucketNames:
          chunks: loki-chunks
          ruler: loki-ruler
          admin: loki-admin
        s3:
          endpoint: minio.minio.svc.cluster.local:9000
          region: us-east-1
          accessKeyId: minioadmin
          secretAccessKey: minioadmin123
          s3ForcePathStyle: true
          insecure: true

      limits_config:
        allow_structured_metadata: true
        volume_enabled: true
        retention_period: 168h
        ingestion_rate_mb: 8
        ingestion_burst_size_mb: 16
        per_stream_rate_limit: 5MB
        per_stream_rate_limit_burst: 20MB
        max_entries_limit_per_query: 5000
        max_query_series: 5000
        max_query_length: 168h
        reject_old_samples: true
        reject_old_samples_max_age: 168h

      compactor:
        working_directory: /var/loki/compactor
        compaction_interval: 10m
        retention_enabled: true
        retention_delete_delay: 2h
        delete_request_store: s3

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

    deploymentMode: SingleBinary

    singleBinary:
      replicas: 1
      persistence:
        enabled: true
        size: 20Gi

    gateway:
      enabled: true

    minio:
      enabled: false

    test:
      enabled: false

    monitoring:
      selfMonitoring:
        enabled: false
      lokiCanary:
        enabled: false

Explanation:

    auth_enabled: false:
        Multi-tenant authentication is disabled in the experimental environment.

    replication_factor: 1:
        1 is used for single-node experimentation.

    schema: v13:
        TSDB schema is used.

    storage.type: s3:
        Loki uses S3-compatible object storage.

    endpoint:
        MinIO Service address is used.

    insecure: true:
        HTTP is used in the experimental environment.

    retention_period: 168h:
        Default retention period of 7 days.

    compactor.retention_enabled:
        Retention is enabled.

    ruler.enable_api:
        Allows managing rules via Ruler API.

    gateway.enabled:
        Loki Gateway is enabled, serving as unified write and query entry point.

Note:

    This values file is for experimental reference.
    Field names may differ across Chart versions.
    Helm template checks are required before production deployment.
    MinIO keys should not be stored in plain text in values for production.

### 6.3 Helm template Check

    helm template loki grafana/loki \
      --namespace logging \
      --version <LOKI_CHART_VERSION> \
      -f values-loki-lab.yaml \
      > loki-rendered.yaml

Check resources:

    grep "^kind:" loki-rendered.yaml | sort | uniq -c

Check MinIO configuration:

    grep -n "minio.minio.svc.cluster.local" loki-rendered.yaml

Check retention:

    grep -n "retention" loki-rendered.yaml

Check ruler:

    grep -n "ruler" loki-rendered.yaml

Check gateway:

    grep -n "gateway" loki-rendered.yaml | head -50

### 6.4 Install Loki

helm install loki grafana/loki \
  --namespace logging \
  --version <LOKI_CHART_VERSION> \
  -f values-loki-lab.yaml

### 6.5 View Loki

    helm list -n logging

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl get pvc -n logging

### 6.6 Verify Loki Gateway

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Run in another terminal:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

Check metrics:

    curl -s http://127.0.0.1:3100/metrics | head

### 6.7 Manual push/query Verification

Write:

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"loki-bootstrap-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki bootstrap test\"]
            ]
          }
        ]
      }"

Query:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="loki-bootstrap-test"}' \
      --data-urlencode 'limit=10' | jq

Acceptance:

    [ ] Loki Helm Release deployed
    [ ] Loki Pod Running
    [ ] Loki Gateway Service exists
    [ ] /ready returns ready
    [ ] Manual push succeeds
    [ ] query_range can find hello loki bootstrap test

---

## SevenI don't know.Deploy Grafana Alloy to Collect Pod Logs

### 7.1 Alloy Deployment Method

This document uses:

    Grafana Alloy
    DaemonSet
    loki.source.kubernetes
    loki.write

Link:

    Kubernetes Pod logs
      ↓
    Alloy
      ↓
    Loki Gateway

### 7.2 Write Alloy values

Create file:

    values-alloy-loki.yaml

Content:

    alloy:
      mounts:
        varlog: true
        dockercontainers: false

      configMap:
        content: |
          logging {
            level  = "info"
            format = "logfmt"
          }

          discovery.kubernetes "pod" {
            role = "pod"

            selectors {
              role  = "pod"
              field = "spec.nodeName=" + coalesce(sys.env("HOSTNAME"), constants.hostname)
            }
          }

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

```yaml
rule {
  source_labels = ["__meta_kubernetes_pod_label_app"]
  action        = "replace"
  target_label  = "app"
}

rule {
  source_labels = ["__meta_kubernetes_pod_label_team"]
  action        = "replace"
  target
```

kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|loki|push|forbidden|denied|timeout|429"

Acceptance:

    [ ] Alloy DaemonSet has been created
    [ ] Alloy Pod is scheduled on expected nodes
    [ ] Alloy RBAC is functioning normally
    [ ] Alloy has no persistent error in logs
    [ ] Alloy write URL points to Loki Gateway

---

## VIII. Deploy test application and generate logs

### 8.1 Deploy nginx-demo

Create:

    nginx-demo.yaml

Content:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-demo
      namespace: app-demo
      labels:
        app: nginx-demo
        app.kubernetes.io/name: nginx-demo
        team: sre
        environment: lab
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx-demo
      template:
        metadata:
          labels:
            app: nginx-demo
            app.kubernetes.io/name: nginx-demo
            team: sre
            environment: lab
        spec:
          containers:
            - name: nginx
              image: nginx:1.25
              ports:
                - containerPort: 80

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-demo
      namespace: app-demo
      labels:
        app: nginx-demo
    spec:
      selector:
        app: nginx-demo
      ports:
        - name: http
          port: 80
          targetPort: 80

Apply:

    kubectl apply -f nginx-demo.yaml

Check:

    kubectl get pod -n app-demo -o wide --show-labels

    kubectl get svc -n app-demo

    kubectl get endpoints nginx-demo -n app-demo

### 8.2 Generate Nginx logs

Run temporary curl:

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Execute inside container:

    curl http://nginx-demo.app-demo.svc.cluster.local

    curl http://nginx-demo.app-demo.svc.cluster.local/notfound

    curl http://nginx-demo.app-demo.svc.cluster.local/healthz

    curl http://nginx-demo.app-demo.svc.cluster.local/metrics

Exit:

    exit

Check Pod logs:

    kubectl get pod -n app-demo | grep nginx-demo

    kubectl logs <nginx-pod-name> -n app-demo --tail=50

### 8.3 Deploy JSON log Demo

Create:

    json-log-demo.yaml

Content: /think

apiVersion: batch/v1
kind: Job
metadata:
  name: json-log-demo
  namespace: app-demo
  labels:
    app: json-log-demo
    app.kubernetes.io/name: json-log-demo
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
              while [ $i -le 120 ]; do
                echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"info\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/orders\",\"method\":\"GET\",\"status\":200,\"duration_ms\":$((20 + RANDOM % 200)),\"trace_id\":\"trace-$i\",\"user_id\":\"user-$i\",\"msg\":\"request success\"}"
                echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"warn\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/orders\",\"method\":\"GET\",\"status\":404,\"duration_ms\":$((50 + RANDOM % 400)),\"trace_id\":\"trace-warn-$i\",\"user
              i=$((i+1))
              sleep 1
            done

Apply:

    kubectl apply -f json-log-demo.yaml

View:

    kubectl get pod -n app-demo | grep json-log-demo

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

### 8.4 Deploy logfmt Log Demo

Create:

    logfmt-log-demo.yaml

Content:

apiVersion: batch/v1
kind: Job
metadata:
  name: logfmt-log-demo
  namespace: app-demo
  labels:
    app: logfmt-log-demo
    app.kubernetes.io/name: logfmt-log-demo
    team: sre
    environment: lab
spec:
  template:
    metadata:
      labels:
        app: logfmt-log-demo
        app.kubernetes.io/name: logfmt-log-demo
        team: sre
        environment: lab
    spec:
      restartPolicy: Never
      containers:
        - name: logfmt-log-demo
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              i=1
              while [ $i -le 120 ]; do
                echo "level=info service=logfmt-log-demo method=GET path=/api/v1/users status=200 duration_ms=$((20 + RANDOM % 200)) trace_id=trace-$i msg=\"request success\""
                echo "level=warn service=logfmt-log-demo method=GET path=/api/v1/users status=404 duration_ms=$((50 + RANDOM % 400)) trace_id=trace-warn-$i msg=\"user not found\""
                echo "level=error service=logfmt-log-demo method=POST path=/api/v1/payment status=500 duration_ms=$((1000 + RANDOM % 3000)) trace_id=trace-error-$i msg=\"upstream timeout\""
                i=$((i+1))
                sleep 1
              done

Application:

    kubectl apply -f logfmt-log-demo.yaml

Check:

    kubectl get pod -n app-demo | grep logfmt-log-demo

    kubectl logs <logfmt-log-demo-pod> -n app-demo --tail=20

---

## Nine. Verifying Loki Has Received Pod Logs

### 9.1 Querying Labels

Ensure port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Query all labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Expected to include:

    app
    cluster
    container
    environment
    job
    namespace
    node
    pod
    team

### 9.2 Querying Namespace Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

Expected to include:

    app-demo
    logging

### 9.3 Querying App Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq

Expected to include:

    nginx-demo
    json-log-demo
    logfmt-log-demo

### 9.4 Querying All Logs for app-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

### 9.5 Querying nginx-demo Logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode 'limit=20' | jq

Querying 404:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "404"' \
      --data-urlencode 'limit=20' | jq

### 9.6 Querying JSON Error

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"' \
      --data-urlencode 'limit=20' | jq

### 9.7 Querying JSON 5xx

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | status >= 500' \
      --data-urlencode 'limit=20' | jq

### 9.8 Querying Slow Requests

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Loki labels contains namespace
    [ ] namespace values contains app-demo
    [ ] app values contains nginx-demo/json-log-demo/logfmt-log-demo
    [ ] Can query nginx access log
    [ ] Can query JSON error
    [ ] Can query JSON 5xx
    [ ] Can query slow requests

---

## Ten. Deploying Grafana

### 10.1 Writing Grafana values

Create:

    values-grafana.yaml

Content:

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

Notes:

    Learning environment uses simple passwords.
    Production must use strong passwords or unified authentication.
    Data Source is automatically added via provisioning.

### 10.2 Installing Grafana

    helm template grafana grafana/grafana \
      --namespace monitoring \
      --version <GRAFANA_CHART_VERSION> \
      -f values-grafana.yaml \
      > grafana-rendered.yaml

    helm install grafana grafana/grafana \
      --namespace monitoring \
      --version <GRAFANA_CHART_VERSION> \
      -f values-grafana.yaml

### 10.3 Viewing Grafana

    helm list -n monitoring

    kubectl get pods -n monitoring -o wide

    kubectl get svc -n monitoring

### 10.4 Accessing Grafana

Port forwarding:

    kubectl port-forward svc/grafana 3000:80 -n monitoring

Access:

    http://127.0.0.1:3000

Login:

    Username:
        admin

    Password:
        admin123

### 10.5 Verifying Loki Data Source

Navigate to:

    Connections
      ↓
    Data sources
      ↓
    Loki
      ↓
    Save & test

Expected:

    Successfully connected to Loki.

If failed, test from Grafana Pod:

    kubectl get pod -n monitoring | grep grafana

    kubectl exec -it <grafana-pod-name> -n monitoring -- sh

Inside container:

    wget -qO- http://loki-gateway.logging.svc.cluster.local/ready

If image lacks wget/curl, use temporary Pod:

    kubectl run curl-grafana-loki-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n monitoring \
      -- sh

Inside container:

    curl -s http://loki-gateway.logging.svc.cluster.local/ready

Exit:

    exit

---

## Eleven. Grafana Explore Query Verification

### 11.1 Basic Queries

Navigate to:

    Explore
      ↓
    Select Loki data source

Execute:

    {namespace="app-demo"}

    {namespace="app-demo", app="nginx-demo"}

    {namespace="app-demo", app="nginx-demo"} |= "404"

### 11.2 JSON Queries

Execute:

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | status >= 500

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000

### 11.3 Formatted Queries

Execute: /think

```json
{
  "namespace": "app-demo",
  "app": "json-log-demo"
}
```

| json |
| __error__="" |
| level="error" |
| line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 11.4 Metric Queries

ERROR Log Trend:

    sum by (app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

JSON 5xx:

    sum by (app, status) (
      count_over_time(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    )

P95 Latency:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

Acceptance:

    [ ] Explore can query app-demo logs
    [ ] Can query nginx-demo logs
    [ ] Can query JSON error
    [ ] Can query JSON 5xx
    [ ] Can query slow requests
    [ ] Can execute metric LogQL

---

## TwelveI don't know.Create Basic Log Dashboard

### 12.1 Dashboard Name

Recommended name:

    K8S Loki Log Platform Overview

Or:

    Kubernetes / Loki Logs Overview

### 12.2 Create Variables

Create the following variables:

    cluster
    namespace
    app
    pod
    container
    node
    team

### 12.3 cluster Variable

Type:

    Query

Data source:

    Loki

Query:

    label_values(cluster)

Name:

    cluster

### 12.4 namespace Variable

Query:

    label_values({cluster="$cluster"}, namespace)

If cluster has no value:

    label_values(namespace)

Name:

    namespace

### 12.5 app Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, app)

If no cluster:

    label_values({namespace="$namespace"}, app)

Name:

    app

### 12.6 pod Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace", app="$app"}, pod)

Name:

    pod

### 12.7 container Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace", pod="$pod"}, container)

Name:

    container

### 12.8 node Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, node)

Name:

    node

### 12.9 team Variable

Query:

    label_values({cluster="$cluster", namespace="$namespace"}, team)

Name:

    team

### 12.10 Panel One: Log Volume Trend

Panel type:

    Time series

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}[$__interval]
      )
    )

Title:

    Log Volume Trend by app

### 12.11 Panel Two: ERROR Log Trend

Query:

    sum by (app) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ "(?i)error|exception|panic|failed" [$__interval]
      )
    )

Title:

    ERROR Log Trend by app

### 12.12 Panel Three: JSON 5xx Trend

Query:

    sum by (app, status) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          | json
          | __error__=""
          | status >= 500 [$__interval]
      )
    )

Title:

    JSON 5xx Log Trend

### 12.13 Panel Four: P95 Request Latency

Query:

quantile_over_time(
  0.95,
  {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
    | json
    | unwrap duration_ms
    | __error__="" [$__interval]
)

Title:

  P95 Request Duration duration_ms

Unit:

  milliseconds

### 12.14 Panel Five: Top Error Pod

Query:

  topk(10,
    sum by (pod) (
      count_over_time(
        {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
          |~ "(?i)error|exception|panic|failed" [$__range]
      )
    )
  )

Title:

  Top 10 Error Pod

### 12.15 Panel Six: Recent Error Logs

Panel type:

  Logs

Query:

  {cluster=~"$cluster", namespace=~"$namespace", app=~"$app"}
    |~ "(?i)error|exception|panic|failed|timeout"

Title:

  Recent Error Logs

### 12.16 Panel Seven: Pod Log Details

Panel type:

  Logs

Query:

  {cluster=~"$cluster", namespace=~"$namespace", pod=~"$pod", container=~"$container"}

Title:

  Pod Log Details

### 12.17 Dashboard Description Panel

Text Panel Content:

  Usage Instructions:

  1. First select cluster, namespace, and app, then view logs.
  2. Do not recommend default selecting All in production environments.
  3. ERROR keyword queries should only be used as auxiliary judgment, structured logs should prioritize level="error".
  4. 5xx panel depends on the presence of status field in logs.
  5. P95 panel depends on the presence of duration_ms numeric field in logs.
  6. If query results are empty, first check time range, variable values, and Loki labels.
  7. If query is very slow, prioritize narrowing time range and namespace/app.
  8. If sensitive information is found, should prioritize fixing application log outputs.

Acceptance:

  [ ] Dashboard has been created
  [ ] namespace variable is available
  [ ] app variable is available
  [ ] pod variable is available
  [ ] Log volume trend has data
  [ ] ERROR trend has data
  [ ] Recent error logs have data
  [ ] Pod log details can be filtered by variables

---

## ThirteenI don't know.Deploying AlertManager

### 13.1 Writing AlertManager values

Create:

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

  This document uses webhook demo to verify alert notifications.
  If not deploying webhook demo temporarily, can also observe alerts via AlertManager UI.
  Production environment should integrate with email, enterprise WeChat, Feishu, DingTalk, or unified alert platform.

### 13.2 Deploying webhook demo

Create:

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

Apply: /think

kubectl apply -f alertmanager-webhook-demo.yaml

### 13.3 Installing AlertManager

    helm template alertmanager prometheus-community/alertmanager \
      --namespace monitoring \
      --version <ALERTMANAGER_CHART_VERSION> \
      -f values-alertmanager.yaml \
      > alertmanager-rendered.yaml

    helm install alertmanager prometheus-community/alertmanager \
      --namespace monitoring \
      --version <ALERTMANAGER_CHART_VERSION> \
      -f values-alertmanager.yaml

### 13.4 Viewing AlertManager

    helm list -n monitoring

    kubectl get pods -n monitoring -o wide | grep -i alertmanager

    kubectl get svc -n monitoring | grep -i alertmanager

### 13.5 Accessing AlertManager

Port forwarding:

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

Accessing:

    http://127.0.0.1:9093

Health check:

    curl -s http://127.0.0.1:9093/-/ready

Expected:

    OK

---

## FourteenI don't know.Configuring Loki Ruler Alert Rules

### 14.1 Confirming Ruler API

Ensure Loki Gateway port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Query:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq

If an empty rule list is returned, it indicates the Ruler API is accessible.

If a 404 or disabled response is returned, check:

    loki.ruler.enable_api
    ruler storage
    gateway routing
    Loki logs

### 14.2 Creating Rule Files

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
    summary: "timeout logs exceed threshold"
    description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} has exceeded 3 timeout-related logs in the last 5 minutes"
    runbook_url: "https://example.com/runbooks/timeout-logs"

### 14.3 Upload Rules

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

Check rules:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

### 14.4 Simulate ERROR Alert

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

Query confirmation:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="alert-error-demo"} |= "ERROR"' \
      --data-urlencode 'limit=20' | jq

### 14.5 Simulate timeout Alert

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

### 14.6 View AlertManager

Port forwarding:

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

Access:

    http://127.0.0.1:9093

Check webhook demo:

    kubectl logs deploy/alertmanager-webhook-demo -n monitoring --tail=200

Verification:

    [ ] Ruler API is accessible
    [ ] Rules uploaded successfully
    [ ] ERROR log alert triggered
    [ ] timeout log alert triggered
    [ ] AlertManager UI shows alerts
    [ ] webhook demo receives alert requests

---

## FifteenI don't know.Log Volume Governance Verification

### 15.1 View Top namespace

LogQL:

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 15.2 View Top app

LogQL:

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 15.3 View Top pod

LogQL:

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 15.4 View ERROR Top App

LogQL:

    topk(10,
      sum by (namespace, app) (
        count_over_time(
          {namespace=~".+"}
            |~ "(?i)error|exception|panic|failed" [5m]
        )
      )
    )

### 15.5 Governance Approach

If an app has abnormally high log volume, handle it in this order:

    1. Check if DEBUG logs are enabled.
    2. Check for excessive health check logs.
    3. Check for business loop anomalies.
    4. Check for duplicate collection.
    5. Check for high cardinality labels.
    6. Reduce log level on the application side.
    7. Filter out-value logs on Alloy side.
    8. Use limits_config on Loki side to restrict abnormal writes.
    9. Do not use global queries by default on Dashboard.

---

## SixteenI don't know.High Cardinality Check

### 16.1 View All Labels

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Focus on checking for:

    request_id
    trace_id
    user_id
    order_id
    session_id
    client_ip
    full_url
    error_message
    timestamp

### 16.2 View Label Values Count

Example:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/app/values" | jq '.data | length'

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/pod/values" | jq '.data | length'

If trace_id is found:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq '.data | length'

### 16.3 View Series Count

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' \
      | jq '.data | length'

### 16.4 High Cardinality Remediation Principles

Principles:

    Do not use request_id as Label.
    Do not use trace_id as Label.
    Do not use user_id as Label.
    Do not use order_id as Label.
    Do not use full_url as Label.
    Do not use error_message as Label.

Correct approach:

    {namespace="app-demo", app="json-log-demo"} | json | trace_id="trace-error-10"

---

## SeventeenI don't know.Sensitive Information Check

### 17.1 Query Sensitive Keywords

LogQL:

    {namespace="app-demo"} |~ "(?i)password|token|authorization|cookie|secret|access_key|private_key"

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"} |~ "(?i)password|token|authorization|cookie|secret|access_key|private_key"' \
      --data-urlencode 'limit=20' | jq

### 17.2 Sensitive Information Governance Principles

Prohibited from entering logs:

    Password
    Token
    Access key
    Secret key
    Authorization header
    Cookie
    Session
    Private key
    Database connection string password
    Plain phone number
    ID number
    Bank card number
    User privacy fields

Priority:

    Application side does not output
      ↓
    Gateway side filtering
      ↓
    Collection side desensitization
      ↓
    Loki permission control
      ↓
    Emergency handling for historical data

### 17.3 Emergency Handling

If sensitive information is found:

    1. Immediately confirm the scope of impact.
    2. Stop further output of sensitive logs.
    3. Modify application log desensitization.
    4. Limit Grafana/Loki query permissions.
    5. Rotate leaked keys/tokens.
    6. Evaluate whether to delete historical logs.
    7. Record the security incident.
    8. Review log standards and code review processes.

---

## EighteenI don't know.Retention Verification

### 18.1 View Loki Configuration

    helm get values loki -n logging -a | grep -n "retention" -A 80

    helm get values loki -n logging -a | grep -n "compactor" -A 80

### 18.2 View Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=800 | grep -Ei "compactor|retention|delete|marker|bucket|permission|error|warn"

### 18.3 View MinIO Objects

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

    mc ls local/loki-chunks

    mc find local/loki-chunks | head

    mc du local/loki-chunks

Exit:

    exit

### 18.4 Notes

Retention will not immediately delete data.

Need to satisfy:

    Compactor enabled
    retention_enabled=true
    retention_period configured correctly
    retention_delete_delay has passed
    Object storage has delete permissions
    Current data exceeds retention period

---

## NineteenI don't know.Limits Verification

### 19.1 View limits_config

    helm get values loki -n logging -a | grep -n "limits_config" -A 100

### 19.2 Check discarded metrics

    curl -s http://127.0.0.1:3100/metrics | grep -E "loki_discarded_samples_total|loki_discarded_bytes_total"

### 19.3 Check 429

Check Alloy:

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "429|rate|failed"

Check Loki:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "429|rate limit|stream limit|ingestion"

### 19.4 Handling Principles

If 429 occurs:

    1. Do not directly increase the limits.
    2. First identify the top namespace/app/pod.
    3. Check for high cardinality.
    4. Check for duplicate scraping.
    5. Check DEBUG or health check logs.
    6. Apply application-side governance.
    7. Then reasonably adjust limits_config.

---

## Twenty, Full-Chain Troubleshooting Runbook

### 20.1 Unable to query business logs

Trigger condition:

    kubectl logs has logs, but Loki cannot query.

Troubleshooting steps:

    1. Verify application logs with kubectl logs.
    2. Confirm Pod node with kubectl get pod -o wide.
    3. Confirm Alloy covers the node with kubectl get pods -n logging -o wide | grep alloy.
    4. Check Alloy errors with kubectl logs <alloy-pod> -n logging.
    5. Check RBAC with kubectl auth can-i get pods/log.
    6. Check Alloy loki.write URL.
    7. Check Loki with curl /ready.
    8. Check Loki label with curl /labels.
    9. Query LogQL from broad to narrow.
    10. Check if logs are written to another Loki.

### 20.2 Loki Write Failure

Trigger condition:

    Alloy failed to send batch.

Troubleshooting steps:

    1. Check Alloy logs.
    2. Determine HTTP status code: 400 / 404 / 429 / 500.
    3. 400: Check log format, timestamp, label.
    4. 404: Check push URL.
    5. 429: Check rate limiting, log volume, high cardinality.
    6. 500: Check Loki, MinIO, ring, PVC.
    7. Manually push to verify.
    8. Re-validate with business logs after fixing.

### 20.3 Grafana Query Empty

Troubleshooting steps:

    1. Confirm Data Source is Loki.
    2. Confirm time range.
    3. Query {namespace="app-demo"} in Explore.
    4. Check Dashboard variables.
    5. Check namespace/app/pod label values.
    6. Check if Multi-value variables use =~.
    7. Confirm correct Loki instance is selected.

### 20.4 Loki Alert Not Triggered

Troubleshooting steps:

    1. Manually execute rule expr.
    2. Confirm expr returns numeric value exceeding threshold.
    3. Confirm for time has been met.
    4. Query /loki/api/v1/rules.
    5. Check Ruler logs.
    6. Check AlertManager URL.
    7. Check AlertManager UI.
    8. Check silence / route.
    9. Check webhook demo logs.

---

## Twenty-one, Final Acceptance Checklist

### 21.1 MinIO Acceptance

    [ ] minio Namespace exists
    [ ] MinIO Pod Running
    [ ] MinIO Service exists
    [ ] MinIO Endpoint is not empty
    [ ] loki-chunks Bucket exists
    [ ] loki-ruler Bucket exists
    [ ] loki-admin Bucket exists
    [ ] logging Namespace can access MinIO

### 21.2 Loki Acceptance

    [ ] Loki Helm Release deployed
    [ ] Loki Pod Running
    [ ] Loki Gateway Running
    [ ] Loki PVC Bound
    [ ] /ready returns ready
    [ ] /metrics is accessible
    [ ] Manual push succeeds
    [ ] query_range query succeeds
    [ ] Loki successfully connects to MinIO
    [ ] Loki has no persistent storage error
    [ ] retention configuration exists
    [ ] limits_config configuration exists
    [ ] Ruler API is accessible

### 21.3 Alloy Acceptance

    [ ] Alloy Helm Release deployed
    [ ] Alloy DaemonSet Running
    [ ] Alloy Pod covers expected nodes
    [ ] Alloy RBAC is normal
    [ ] Alloy loki.write URL is correct
    [ ] Alloy logs have no persistent error
    [ ] namespace/pod/container/app appears in Loki labels
    [ ] app-demo logs are queryable

### 21.4 Test Application Acceptance

[ ] nginx-demo Running
    [ ] nginx-demo Service Yes. Endpoint
    [ ] kubectl logs Yeah. Nginx access log
    [ ] json-log-demo Can output JSON Log
    [ ] logfmt-log-demo Can output logfmt Log
    [ ] Loki Query nginx-demo
    [ ] Loki Query json-log-demo
    [ ] Loki Query logfmt-log-demo

### 21.5 Grafana Acceptance

    [ ] Grafana Pod Running
    [ ] Grafana Web Accessible
    [ ] Loki Data Source Save & test Success
    [ ] Explore Query app-demo
    [ ] Explore Query JSON error
    [ ] Dashboard Created
    [ ] namespace/app/pod/container Variables Available
    [ ] Log trend panel has data
    [ ] ERROR Data on trend panel
    [ ] Data from the recent error log panel

### 21.6 AlertManager / Ruler Acceptance

    [ ] AlertManager Pod Running
    [ ] AlertManager UI Accessible
    [ ] /-/ready Back OK
    [ ] webhook demo Running
    [ ] Loki Ruler Rules uploaded successfully
    [ ] ERROR The alarm triggers.
    [ ] timeout The alarm triggers.
    [ ] AlertManager UI I can see the alarm.
    [ ] webhook demo We can receive the alarm.

### 21.7 Production Governance Acceptance

    [ ] Statistics Top namespace Log Volume
    [ ] Statistics Top app Log Volume
    [ ] Statistics Top pod Log Volume
    [ ] Can check high base label
    [ ] Can check sensitive information
    [ ] Can View retention Configure
    [ ] Can View limits_config
    [ ] Can View Loki metrics
    [ ] Yes. Loki No log for query Runbook
    [ ] Yes. Loki Writing failed Runbook
    [ ] Yes. Loki The alarm is not triggered. Runbook

---

## Twenty-two, Common Misconceptions

### 22.1 Misconception One: Installing Loki Equals a Completed Logging Platform

Error.

A complete logging platform must at least include:

    Collection
    Writing
    Storage
    Querying
    Dashboard
    Alerting
    Retention
    Limits
    Permissions
    Runbook

### 22.2 Misconception Two: Loki Can't Find Logs Means Loki Is Broken

Not necessarily.

Possible reasons:

    Application has no stdout logs
    Alloy hasn't collected logs
    Alloy wrote wrong address
    Label written incorrectly
    Time range is incorrect
    Grafana data source is wrong
    Logs were written to another Loki

### 22.3 Misconception Three: More Logs Are Better

Error.

Useless logs will cause:

    Storage costs
    Slower queries
    Alert noise
    Troubleshooting difficulties
    Risk of sensitive information

### 22.4 Misconception Four: Using request_id / trace_id as Labels Makes Querying Easier

Error.

This causes high cardinality issues.

Correct approach:

    Place trace_id in log body.

Query:

    {namespace="app-demo", app="json-log-demo"} | json | trace_id="trace-error-10"

### 22.5 Misconception Five: Dashboard Variables Can Be Used for Permission Control

Error.

Dashboard variables are not permission boundaries.

Real permissions require:

    Grafana permissions
    Data Source permissions
    Loki multi-tenancy
    Reverse proxy authentication
    NetworkPolicy

### 22.6 Misconception Six: More Log Alerts Are Better

Error.

Log alerts can generate too much noise.

Recommendations:

    Use metric alerts as primary.
    Use log alerts as supplementary.
    Only alert for logs with strong fault semantics.
    Alerts must have owner and Runbook.

### 22.7 Misconception Seven: Retention Is Immediately Applied Once Configured

Error.

Retention depends on Compactor, with cycles and delays.

Need to confirm:

    retention_enabled
    retention_period
    delete_request_store
    Object storage deletion permissions
    retention_delete_delay

---

## Twenty-three, Production Deployment Recommendations

### 23.1 Minimum Closed-loop for Small-scale Production

Must at least include:

    Loki + Object storage
    Alloy DaemonSet
    Grafana Data Source
    Basic Dashboard
    Basic ERROR / timeout alerts
    retention
    limits_config
    MinIO/S3 capacity monitoring
    Loki self-monitoring
    Runbook

### 23.2 Enhanced for Medium-scale Production

Add:

    High-availability Loki deployment
    Multi-replica Gateway
    More complete limits
    More detailed retention_stream
    Grafana permissions
    Namespace / Team dimension Dashboard
    AlertManager routing
    Application log standards
    High cardinality inspection
    Sensitive information inspection

### 23.3 Direction for Large-scale Production

Consider:

    Microservices / Distributed Loki
    Independent object storage cluster
    Caching
    Multi-tenancy
    Tenant-level limits
    Tenant-level retention
    Query concurrency governance
    Dashboard as Code
    GitOps management
    Log cost reports
    Integration with CMDB / permission systems

### 23.4 Log Standardization Recommendations

Recommended unified fields for application logs:

    timestamp
    level
    service
    environment
    trace_id
    span_id
    method
    route
    status
    duration_ms
    msg
    error_type

Avoid field confusion:

### 23.5 Label Best Practices

Recommended Labels:

    cluster
    environment
    namespace
    app
    container
    pod
    node
    team

Prohibited Labels:

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

---

## 24. Cleaning Up Experimental Resources

If it's just an experimental environment, you can clean up as needed.

### 24.1 Deleting Test Applications

    kubectl delete -f nginx-demo.yaml

    kubectl delete -f json-log-demo.yaml

    kubectl delete -f logfmt-log-demo.yaml

Or:

    kubectl delete namespace app-demo

### 24.2 Deleting Alloy

    helm uninstall alloy -n logging

### 24.3 Deleting Loki

    helm uninstall loki -n logging

Note:

    Do not delete PVCs arbitrarily.
    If you confirm it's an experimental environment, you can delete PVCs.

Check PVCs:

    kubectl get pvc -n logging

Delete PVCs:

    kubectl delete pvc <pvc-name> -n logging

### 24.4 Deleting Grafana

    helm uninstall grafana -n monitoring

Check PVCs:

    kubectl get pvc -n monitoring

### 24.5 Deleting AlertManager

    helm uninstall alertmanager -n monitoring

Delete webhook demo:

    kubectl delete -f alertmanager-webhook-demo.yaml

### 24.6 Deleting MinIO

    kubectl delete -f minio-lab.yaml

Note:

    Deleting MinIO PVCs will remove object storage data.
    Do not perform this in production environments.

---

## 25. Summary

This article completed the end-to-end setup of a Kubernetes logging platform from scratch.

Complete pipeline:

    Application Pod outputs logs
      ↓
    Alloy collects Pod logs
      ↓
    Loki Gateway receives and writes logs
      ↓
    Loki stores and queries logs
      ↓
    MinIO saves log data
      ↓
    Grafana queries and displays logs
      ↓
    Loki Ruler executes log alerts
      ↓
    AlertManager receives and routes alerts
      ↓
    Runbook supports on-call handling

This closed-loop covers:

    MinIO object storage
    Loki Helm deployment
    Loki Gateway
    Loki retention
    Loki limits_config
    Grafana Alloy DaemonSet
    Kubernetes Pod log collection
    LogQL basic queries
    LogQL JSON queries
    Grafana Explore
    Grafana Dashboard
    AlertManager
    Loki Ruler
    Log alerts
    Log volume governance
    High cardinality governance
    Sensitive information checks
    Production Runbook

True mastery of Loki is not just executing helm install, but being able to answer:

    Where do logs come from?
    Who collects them?
    Where are they written?
    Where are they stored?
    How to query?
    How to alert?
    How long to retain?
    Who can view?
    Who is responsible?
    How to troubleshoot failures?
    How to control costs?
    How to prevent sensitive information?

After completing this article, you already have the end-to-end capability to build a basic Kubernetes logging platform.

Next article recommendation:

    16-Loki Production Expansion Design: Multi-tenancy - Permissions - Cost - Platformization

Key topics to study:

    Multi-tenancy design
    X-Scope-OrgID
    Grafana permissions
    Data Source permissions
    Namespace / Team isolation
    Log cost statistics
    Tenant-level retention
    Tenant-level limits
    Dashboard as Code
    GitOps management
    Integration with CMDB / Unified permission platform

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Install Grafana Loki with Helm:
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Loki deployment modes:
  https://grafana.com/docs/loki/latest/get-started/deployment-modes/

- Loki architecture:
  https://grafana.com/docs/loki/latest/get-started/architecture/

- Loki configuration:
  https://grafana.com/docs/loki/latest/configure/

- Loki storage:
  https://grafana.com/docs/loki/latest/configure/storage/

- Loki HTTP API:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- Query Loki:
  https://grafana.com/docs/loki/latest/query/

- Log queries:
  https://grafana.com/docs/loki/latest/query/log_queries/

- Metric queries:
  https://grafana.com/docs/loki/latest/query/metric_queries/

- LogQL Reference:
  https://grafana.com/docs/loki/latest/query/query_reference/

- Query best practices:
  https://grafana.com/docs/loki/latest/query/bp-query/

- Log retention:
  https://grafana.com/docs/loki/latest/operations/storage/retention/

- Request validation and rate limits:
  https://grafana.com/docs/loki/latest/operations/request-validation-rate-limits/

- Loki alerting and recording rules:
  https://grafana.com/docs/loki/latest/alert/

- Grafana Alloy Documentation:
  https://grafana.com/docs/alloy/latest/

- Collect Kubernetes logs and forward them to Loki:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- loki.source.kubernetes:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- loki.write:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/

- discovery.kubernetes:
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.kubernetes/

- discovery.relabel:
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/

- Grafana Loki Data Source:
  https://grafana.com/docs/grafana/latest/datasources/loki/

- Grafana Dashboards:
  https://grafana.com/docs/grafana/latest/dashboards/

- Grafana Variables:
  https://grafana.com/docs/grafana/latest/dashboards/variables/

- AlertManager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- AlertManager Concepts:
  https://prometheus.io/docs/alerting/latest/alertmanager/

- MinIO Documentation:
  https://min.io/docs/minio/kubernetes/upstream/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes kubectl logs:
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/

- Kubernetes Secrets:
  https://kubernetes.io/docs/concepts/configuration/secret/

- Kubernetes NetworkPolicy:
  https://kubernetes.io/docs/concepts/services-networking/network-policies/