# 15-Loki Comprehensive Practice: Building a Complete Kubernetes Log Platform from Scratch

## Document Description

This article is the fifteenth in the Loki series of tutorials, aimed at integrating previous topics such as Loki, MinIO, Grafana Alloy, Grafana, LogQL, Ruler, AlertManager, retention settings, limits_config, and production troubleshooting into a comprehensive Kubernetes log platform practice.

What has been covered previously includes:

    01-Loki Basics and Experimental Environment Setup
    02-Loki Architecture Principles and Component Responsibilities
    03-Comparison of Loki Deployment Modes and Experimental Selection
    04-Helm-Based Deployment of Single-Loki Mode
    05-Integrating Loki with MinIO for Object Storage
    06-Gathering Kubernetes-Pod Logs Using Grafana-Alloy
    07-Loki Tag Design and High Cardinality Issues
    08-Basic LogQL Queries: Retrieving Namespace-Pod-Container Logs
    09-Advanced LogQL Queries: json-logfmt-regexp-unwrap
    10-Integrating Grafana with Loki for Log Dashboards
    11-Loki Log Alerts: Integrating Ruler and AlertManager
    12-Loki Production Management: Log Volume, Retention Period, Throttling, Security
    13-Loki Performance and High Availability: Introduction to Simple-Scalable Mode
    14-Common Loki Troubleshooting: Collection, Writing, Querying, Storage, Alerts

This article does not focus on any single component but instead builds a complete, verifiable, queryable, alert-capable, manageable, and troubleshootable Kubernetes log platform from scratch.

The complete workflow is as follows:

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
    Runbook / Incident Handling

This article addresses the following key questions:

- How to plan a Kubernetes log platform from scratch;
- How to prepare namespaces, MinIO, and buckets;
- How to deploy Loki;
- How to configure Loki to use MinIO for object storage;
- How to set up retention policies;
- How to configure limits_config;
- How to deploy Grafana Alloy;
- How to collect Kubernetes Pod logs;
- How to verify the write process from Alloy to Loki;
- How to deploy test applications to generate logs;
- How to query Namespace/App/Pod/Container logs using LogQL;
- How to deploy Grafana and add the Loki data source;
- How to create basic dashboards;
- How to create log alert rules;
- How to configure Ruler to connect with AlertManager;
- How to simulate ERROR/5xx/timeout alerts;
- How to manage high log volumes;
- How to identify high-cardinality labels;
- How to detect sensitive information;
- How to organize production incident handling runbooks;
- How to complete the final acceptance process.

---

## Tags

#Loki #Grafana #GrafanaAlloy #MinIO #AlertManager #LogQL #Kubernetes #PodLogs #LogPlatform #Observability #Log Alerts #LogManagement #SRE #Runbook #ProductionPractice

---

## Recommended Reading Path

Recommended reading path:

    10-Logs/02-Loki/15-LokiComprehensivePractice:BuildingaCompleteKubernetesLogPlatformfromScratch.md

---

## I. Comprehensive Practice Objectives

Upon completing this article, you should be able to independently build a basic Kubernetes log platform:

    1. Prepare the logging, monitoring, minio, and app-demo namespaces.
    2. Deploy MinIO as Loki's object storage.
    3. Create the necessary buckets for Loki.
    4. Use Helm to deploy Loki.
    5. Configure Loki to use MinIO.
    6. Set up Loki retention policies.
    7. Configure Loki limits_config.
    8. Verify Loki's /ready, /metrics, push, and query_range services.
    9. Use Helm to deploy Grafana Alloy.
    10. Configure Alloy to collect Kubernetes Pod logs.
    11. Add namespace, pod, container, node, app, team, environment, and cluster labels to the logs.
    12. Deploy test applications nginx-demo, json-log-demo, and logfmt-log-demo.
    13. Query Pod logs using LogQL.
    14. Deploy Grafana.
    15. Add the Loki data source to Grafana.
    16. Create basic log dashboards.
    17. Deploy AlertManager.
    18. Configure Loki Ruler.
    19. Create log alert rules| Dashboard                   |
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

Log Writing Flow:

    Pod stdout / stderr
      ↓
    Alloy
      ↓
    Loki Gateway
      ↓
    Loki
      ↓
    MinIO

Log Querying Flow:

    Grafana Explore / Dashboard
      ↓
    Loki Gateway
      ↓
    Loki Query
      ↓
    MinIO / Ingester
      ↓
    Return Log Results

Log Alerting Flow:

    Loki Ruler
      ↓
    Regularly Execute LogQL
      ↓
    Trigger Thresholds
      ↓
    AlertManager
      ↓
    Notify Channels

Troubleshooting Loop:

    Prometheus / Grafana Detect Issues
      ↓
    Query Logs with Loki
      ↓
    AlertManager Issues Notifications
      ↓
    Runbook for Resolution
      ↓
    Review and Optimize Log Management Practices

---

## III. Experimental Environment Planning

### 3.1 Kubernetes Nodes

Experimental Nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Container Runtime:

    containerd

System:

    Ubuntu Server 22.04.5 LTS

Kubernetes:

    kubeadm Cluster

CNI:

    Calico

### 3.2 Namespace Planning

Namespaces:

    minio:
        Deploy MinIO object storage.

    logging:
        Deploy Loki, Loki Gateway, Grafana Alloy.

    monitoring:
        Deploy Grafana, AlertManager.

    app-demo:
        Deploy test applications for log generation.

Create Namespaces:

    kubectl create namespace minio

    kubectl create namespace logging

    kubectl create namespace monitoring

    kubectl create namespace app-demo

If existing, ignore any error messages.

Check:

    kubectl get ns

### 3.3 Component Versioning Strategy

Production Recommendations:

    Use fixed versions for Helm Charts, images, and values files.
    Manage version changes via Git for the values file.
    Avoid using "latest" versions.
    Always update the helm template before making changes.
    Verify new versions thoroughly before deployment.
    Ensure rollback capabilities are available.

 placeholders used in this document:

    <LOKI_CHART_VERSION>
    <GRAFANA_CHART_VERSION>
    <ALLOY_CHART_VERSION>
    <ALERTMANAGER_CHART_VERSION>

Please replace these with actual version numbers before implementation.

---

## IV. Setting Up the Helm Repository

### 4.1 Adding the Grafana Helm Repository

    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

### 4.2 Adding the prometheus-community Repository

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

    helm repo update

### 4.3 Checking Available Charts

Loki:

    helm search repo grafana/loki --versions | head -20

Grafana:

    helm search repo grafana/grafana --versions | head -20

Alloy:

    helm search repo grafana/alloy --versions | head -20

AlertManager:

    helm search repo prometheus-community/alertmanager --versions | head -20

Record the version numbers:

    LOKI_CHART_VERSION=<actual version>
    GRAFANA_CHART_VERSION=<actual version>
    ALLOY_CHART_VERSION=<actual version>
    ALERTMANAGER_CHART_VERSION=<actual version>

### 4.4 Exporting Default Values

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

    Field names may vary across different chart versions.
    Always refer to the default values of the current version before modifying them.
    The examples in this document are for learning purposes only; production settings must be verified according to the actual version numbers.

---

## V. Deploying MinIO Object Storage

### 5.1 Purpose of MinIO

MinIO serves as an S3-compatible object storage system for Loki.

It is used to store:

   ```yaml
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
```

Application:

```bash
kubectl apply -f minio-lab.yaml
```

### 5.3 Checking MinIO

Run the following commands:

```bash
kubectl get pods -n minio -o wide
kubectl get svc -n minio
kubectl get pvc -n minio
```

Wait for the Pod to start running.

### 5.4 Creating a Bucket

Run `mc` inside the container:

```bash
kubectl run minio-mc \
  --rm -it \
  --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
  -n minio \
  -- sh
```

Inside the container, execute the following commands:

```bash
mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123
mc mb local/loki-chunks
mc mb local/loki-ruler
mc mb local/loki-admin
mc ls local
exit
```

### 5.5 Verifying MinIO

Run the following commands to verify:

```bash
kubectl get pods -n minio -o wide
kubectl get svc -n minio
kubectl get endpoints minio -n minio
kubectl run curl-minio-test \
  --rm -it \
  --image=curlimages/curl:8.5.0 \
  -n logging \
  -- sh
```

Inside the container, execute:

```bash
curl -I http://minio.minio.svc.cluster.local:9000
exit
```

Verification results:

[ ] MinIO Pod is running.
[ ] The MinIO Service exists.
[ ] Endpoints are available.
[ ] loki-chunks, loki-ruler, and loki-admin have been created.
[ ] The logging Namespace can access MinIO.
---

## VI. Deploying Loki

### 6.1 Choosing the Loki Deployment Mode

In this practical example, we will use the **Monolithic** mode.

Reasons:

- The experiment is more stable.
- It consumes fewer resources.
- It facilitates closed-loop verification.
- We have already learned about **Simple Scalable** earlier. The focus of this example is on a complete closed-loop, not on large-scale, high-availability architectures.

Production guidance:

- For small-scale applications, you can start with the Monolithic mode.
- For medium to large-scale applications, consider using **Microservices** or **Distributed** approaches.
- **Simple Scalable** is considered outdated and is not recommended as a long-term production goal.

### 6.2 Writing Loki Values

Create a file called `values-loki-lab.yaml` with the following content:

```yaml
loki:
  authenabled: false

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
    volumeenabled: true
    retention_period: 168h
    ingestion_rate_mb: 8
    ingestion_burst_size_mb: 16
    per_stream_rate_limit: 5MB
    per_stream_rate_limit_burst: 20MB
    max_entries_limit_per_query: 5000
    maxcompactor.retention_enabled:
    Retention is enabled.

ruler.enable_api:
    Management of rules via the Ruler API is allowed.

gateway.enabled:
    The Loki Gateway is enabled, providing a unified entry point for writing and querying data.

Note:

    These values are for experimental reference only.
    Field names may vary depending on the Chart version.
    It is essential to verify the helm template before production use.
    In production environments, MinIO credentials should not be stored explicitly in these values.

### 6.3 Helm Template Verification

Execute the following command to generate the loki grafana/loki helm template:

    helm template loki grafana/loki \
      --namespace logging \
      --version <LOKI_CHART_VERSION> \
      -f values-loki-lab.yaml \
      > loki-rendered.yaml

Check the generated resources:

    Use `grep "^kind:" loki-rendered.yaml | sort | uniq -c` to verify the list of resources.

Check the MinIO configuration:

    Use `grep -n "minio.minio.svc.cluster.local" loki-rendered.yaml` to confirm the MinIO settings.

Verify retention settings:

    Use `grep -n "retention" loki-rendered.yaml` to check the retention configuration.

Verify ruler settings:

    Use `grep -n "ruler" loki-rendered.yaml` to ensure the ruler is configured correctly.

Check the gateway settings:

    Use `grep -n "gateway" loki-rendered.yaml | head -50` to confirm the gateway details.

### 6.4 Installing Loki

Install Loki using the helm command:

    helm install loki grafana/loki \
      --namespace logging \
      --version <LOKI_CHART_VERSION> \
      -f values-loki-lab.yaml

### 6.5 Viewing Loki Resources

Use the following commands to check Loki resources:

    `helm list -n logging`
    `kubectl get pods -n logging -o wide`
    `kubectl get svc -n logging`
    `kubectl get pvc -n logging`

### 6.6 Verifying the Loki Gateway

Set up port forwarding for the Loki Gateway:

    `kubectl port-forward svc/loki-gateway 3100:80 -n logging`

Verify the gateway by accessing `http://127.0.0.1:3100/ready` in another terminal. The response should be "ready".

View metrics using:

    `curl -s http://127.0.0.1:3100/metrics | head`

### 6.7 Manual Push/Query Verification

For writing data:

    `TS=$(date +%s%N)`
    `curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"loki-bootstrap-test\",
              \"namespace\": \"app-demo\"",
              \"app\": \"manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki bootstrap test\"]
            ]
          }
        ]
      }"

For querying data:

    `curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="loki-bootstrap-test"}' \
      --data-urlencode 'limit=10' | jq`

Confirmation:

    [ ] Loki Helm Release has been deployed successfully.
    [ ] The Loki Pod is running.
    [ ] The Loki Gateway Service exists.
    [ ] The "ready" response is returned when accessing /ready.
    [ ] Manual data push was successful.
    [ ] Data can be queried using query_range, with the expected result "hello loki bootstrap test" displayed.

---

## VII. Deploying Grafana Alloy for Pod Log Collection

### 7.1 Deployment Method Using Alloy

In this section, we will use:

    Grafana Alloy
    DaemonSet
    loki.source.kubernetes
    loki.write

The data flow is as follows:

    Kubernetes Pod logs
      ↓
    Alloy
      ↓
    Loki Gateway

### 7.2 Creating Alloy Values File

Create a file named `values-alloy-loki.yaml` with the following content:

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
              field = "spec.nodeName=" + coalesce(sys.env("HOST```yaml
action="replace"
target_label="environment"

rule {
source_labels=["__meta_kubernetes_namespace", "__meta_kubernetes_pod_container_name"]
action="replace"
target_label="job"
separator "="
replacement="$1"
}

loki.source.kubernetes "pod_logs" {
targets=discovery.relabel.pod_logs.output
forward_to=[loki.process.pod_logsreceiver]
}

loki.process "pod_logs" {
stage.static_labels {
values={cluster="k8s-lab"}
}

forward_to=[loki.write.loki.receiver]
}

loki.write "loki" {
endpoint {
url="http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push"
}
}

controller:
type: daemonset

serviceAccount:
create: true

rbac:
create: true
```

### 7.3 Helm template inspection

```bash
helm template alloy grafana/alloy \
  --namespace logging \
  --version <ALLOY_CHART_VERSION> \
  -f values-alloy-loki.yaml \
  > alloy-rendered.yaml
```

Inspections:

```bash
grep "^kind:" alloy-rendered.yaml | sort | uniq -c
grep -n "kind: DaemonSet" alloy-rendered.yaml
grep -n "loki.write" alloy-rendered.yaml
grep -n "loki-gateway.logging.svc.cluster.local" alloy-rendered.yaml
```

### 7.4 Install Alloy

```bash
helm install alloy grafana/alloy \
  --namespace logging \
  --version <ALLOY_CHART_VERSION> \
  -f values-alloy-loki.yaml
```

### 7.5 Check Alloy

```bash
helm list -n logging
kubectl get ds -n logging
kubectl get pods -n logging -o wide | grep alloy
```

### 7.6 Verify Alloy RBAC

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:logging:alloy
kubectl auth can-i watch pods \
  --as=system:serviceaccount:logging:alloy
kubectl auth can-i get pods/log \
  --as=system:serviceaccount:logging:alloy
```

Expected outcome:

```bash
yes
```

### 7.7 View Alloy Logs

```bash
kubectl logs <alloy-pod-name> -n logging --tail=200
kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|loki|push|forbidden|denied|timeout|429"
```

Acceptance criteria:

```bash
[ ] The Alloy DaemonSet has been successfully created.
[ ] The Alloy Pod is deployed on the intended nodes.
[ ] Alloy RBAC functions correctly.
[ ] There are no persistent error messages in Alloy logs.
[ ] The designated URL for writing Alloy logs points to the Loki Gateway.
```

---

## Section 8: Deploying Test Applications and Generating Logs

### 8.1 Deploying nginx-demo

Create the YAML file:

```yaml
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
app.kubernetes.io.name: nginx-demo
team: sre
environment: lab
spec:
containers:
- name: nginx
  image: nginx:1.25
  ports:
  - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f nginx-demo.yaml
```

Check the results:

```bash
kubectl get pod -n app-demo -o wide --show-labels
kubectl get svc -n app-demo
kubectl get endpoints nginx-demo -n app-demo
```

### 8.2 Generating Nginx Logs

Run a temporary curl container:

```bash
kubectl run curl-test \
  --rm -it \
  --image=curlimages/curl:8.5.0 \
  -n app-demo \
  -- sh
```

Inside the container, execute the following commands to test Nginx services:

```bash
curl http://nginx-demo.app-demo.svc.cluster.local
curl http://nginx-demo.app-demo.svc.cluster.local/notfound
curl http://nginx-demo.app-demo.svc.cluster.local/healthz
curl http://nginx-demo.app-demo.svc.cluster.local/metrics
```

Exit the container:
```bash
exit
```exit

To view the Pod logs:

```
kubectl get pod -n app-demo | grep nginx-demo

kubectl logs <nginx-pod-name> -n app-demo --tail=50
```

### 8.3 Deploying JSON Log Demo

Create the YAML file:

```yaml
json-log-demo.yaml
```

Content:

```yaml
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
            echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"info\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/orders\",\"method\":\"GET\",\"status\":200,\"duration_ms\":$((20 + RANDOM % 200)),\\\"trace_id\":\"trace-$i\",\"user_id\":\"user-$i\",\"msg\":\"request success\"}"
            echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"warn\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/orders\",\"method\":\"GET\",\"status\":404,\"duration_ms\":$((50 + RANDOM % 400)),\\\"trace_id\":\"trace-warn-$i\",\"user_id\":\"user-warn-$i\",\"msg\":\"resource not found\"}"
            echo "{\"timestamp\":\"$(date -Iseconds)\",\\\"level\":\"error\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/payment\",\"method\":\"POST\",\"status\":500,\"duration_ms\":$((1000 + RANDOM % 3000)),\\\"trace_id\":\"trace-error-$i\",\"user_id\":\"user-error-$i\",\"msg\":\"database connection failed\"}"
            i=$((i+1))
            sleep 1
          done
```

Apply the configuration:

```
kubectl apply -f json-log-demo.yaml
```

To view the logs:

```
kubectl get pod -n app-demo | grep json-log-demo

kubectl logs <json-log-demo-pod> -n app-demo --tail=20
```

### 8.4 Deploying logfmt Log Demo

Create the YAML file:

```yaml
logfmt-log-demo.yaml
```

Content:

```yaml
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
            echo "level=warn service=logfmt-log-demo method=GET path=/api/v1/users status=404 duration_ms=$((50 + RANDOM % 400)) trace_id=trace-warn-$i msg\":\"user not found\""
            echo "level=error service=logfmt-log-demo method=POST path=/api/v1/payment status=500 duration_ms=$((1000 + RANDOM % 3000)) trace_id=trace-error-$i msg=\"upstream timeout\""
            i=$((i+1))
            sleep 1
          done
```

Apply the configuration:

```
kubectl apply -f logfmt-log-demo.yaml
```

To view the logs:

```
kubectl get pod -n app-demo | grep logfmt-log-demo

kubectl logs <logfmt-log-demo-pod> -n app-demo --tail=20
```

---

## IX. Verifying That Loki Has Received Pod Logs

### 9.1 Querying Labels

Ensure port forwarding is set up:

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq
```

### 9.5 Querying nginx-demo logs

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"}' \
      --data-urlencode 'limit=20' | jq
```

Querying 404 errors:

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "404"' \
      --data-urlencode 'limit=20' | jq
```

### 9.6 Querying JSON errors

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"' \
      --data-urlencode 'limit=20' | jq
```

### 9.7 Querying JSON 5xx errors

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | status >= 500' \
      --data-urlencode 'limit=20' | jq
```

### 9.8 Querying slow requests

```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000' \
      --data-urlencode 'limit=20' | jq
```

Verification:

```bash
[ ] The Loki labels contain the namespace.
[ ] The namespace values include "app-demo".
[ ] The app values consist of "nginx-demo/json-log-demo/logfmt-log-demo".
[ ] It is possible to query nginx access logs.
[ ] JSON errors can be queried.
[ ] JSON 5xx errors can be queried.
[ ] Slow requests can be queried.
```

---

## Section X: Deploying Grafana

### 10.1 Creating Grafana configuration values

Create a file named `values-grafana.yaml` with the following content:

```yaml
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
```

Note:

- Use a simple password for the learning environment.
- In production, use a strong password or implement unified authentication.
- The Loki Data Source will be automatically added through provisioning.

### 10.2 Installing Grafana

Use the following Helm commands to create and install Grafana:

```bash
helm template grafana grafana/grafana \
  --namespace monitoring \
  --version <GRAFANA_CHART_VERSION> \
  -f values-grafana.yaml \
  > grafana-rendered.yaml

helm install grafana grafana/grafana \
  --namespace monitoring \
  --version <GRAFANA_CHART_VERSION> \
  -f values-grafana.yaml
```

### 10.3 Viewing Grafana

Use the following Helm commands to list deployed resources and view the Grafana Pod:

```bash
helm list -n monitoring
kubectl get pods -n monitoring -o wide
kubectl get svc -n monitoring
```

### 10.4 Accessing Grafana

Set up port forwarding using Kubernetes:

```bash
kubectl port-forward svc/grafana 3```json
{
  "5xx": {
    "sum by (app, status)": {
      "count_over_time": {
        {"namespace": "app-demo", "app": "json-log-demo": {
          | "json",
          | "__error__\": "",
          | "status >= 500": { "5m": true }
        }
      }
    }
  },
  "p95.duration_ms": {
    "quantile_over_time": {
      "0.95": {
        {"namespace": "app-demo", "app": "json-log-demo": {
          | "json",
          | "unwrap duration_ms": {
            | "__error__\": ""
          }
        }
      }
    }
  },
  "acceptance": [
    ["Can query app-demo logs"], 
    ["Can query nginx-demo logs"], 
    ["Can query JSON errors"], 
    ["Can query JSON 5xx errors"], 
    ["Can query slow requests"], 
    ["Can execute metric-based LogQL"]
  ]
}
```send_resolved: true

service:
  type: ClusterIP

persistence:
  enabled: false

Note:

This document uses a webhook demo to verify alarm notifications. If you do not deploy the webhook demo for now, you can still observe alarms through the AlertManager UI. In a production environment, it is recommended to integrate with email, WeCom, Lark, DingTalk, or a unified alarm platform.

### 13.2 Deploying the Webhook Demo

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

Apply:

kubectl apply -f alertmanager-webhook-demo.yaml

### 13.3 Installing AlertManager

Use the Helm template:

helm template alertmanager prometheus-community/alertmanager \
  --namespace monitoring \
  --version <ALERTMANAGER_CHART_VERSION> \
  -f values-alertmanager.yaml \
  > alertmanager-rendered.yaml

Then install it:

helm install alertmanager prometheus-community ALERTMANAGER_CHART_VERSION \
  --namespace monitoring \
  -f values-alertmanager.yaml

### 13.4 Checking AlertManager

List services in the monitoring namespace:

helm list -n monitoring

Check pods and services related to AlertManager:

kubectl get pods -n monitoring -o wide | grep -i alertmanager
kubectl get svc -n monitoring | grep -i alertmanager

### 13.5 Accessing AlertManager

Use port forwarding:

kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

Access the interface:

http://127.0.0.1:9093

Check its health status:

curl -s http://127.0.0.1:9093/-/ready

Expected response:

OK

---

## Section Fourteen: Configuring Loki Ruler Alarm Rules

### 14.1 Verifying the Ruler API

Ensure that the Loki Gateway port is forwarded correctly:

kubectl port-forward svc/loki-gateway 3100:80 -n logging

Query the Ruler API:

curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq

If an empty rule list is returned, it means the Ruler API is accessible.

If a 404 error or "disabled" status is received, check the following settings:

- loki.ruler.enable_api
- ruler storage configuration
- gateway routing
- Loki log storage

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
          summary: "Too many error logs in app-demo"
          description: "The number of error logs in namespace {{ $labels.namespace }} and app {{ $labels.app }} exceeded 5 in the last 5 minutes."
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
          summary: "Too many JSON```markdown
annotations:
              summary: "Too many timeout logs"
              description: "In the namespace {{ $labels.namespace }} and app {{ $labels.app}}, more than 3 timeout-related logs have been recorded in the last 5 minutes."
              runbook_url: "https://example.com/runbooks/timeout-logs"

### 14.3 Uploading Rules

    curl -s -X POST \
      -H "Content-Type: application/yaml" \
      --data-binary @loki-rules-app-demo.yaml \
      http://127.0.0.1:3100/loki/api/v1/rules/app-demo

    To view the rules:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq

### 14.4 Simulating ERROR Alerts

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

    To verify the alerts:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="alert-error-demo"} |= "ERROR"' \
      --data-urlencode 'limit=20' | jq
```

### 14.5 Simulating Timeout Alerts

    for i in $(seq 1 6); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"loki-alert-test\"",
                \"namespace\": \"app-demo\",
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

### 14.6 Viewing the AlertManager

    Port forwarding:

    kubectl port-forward svc/alertmanager 9093:9093 -n monitoring

    Access:

    http://127.0.0.1:9093

    To view the webhook demo:

    kubectl logs deploy-alertmanager-webhook-demo -n monitoring --tail=200

    Verification:

    [ ] The Ruler API is accessible.
    [ ] Rules were uploaded successfully.
    [ ] ERROR log alerts were triggered.
    [ ] Timeout log alerts were triggered.
    [ ] Alerts can be seen in the AlertManager UI.
    [ ] The webhook demo receives alert requests.
```

---

## Chapter Fifteen: Verification of Log Volume Management

### 15.1 Viewing Top Namespaces

    LogQL:

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 15.2 Viewing Top Apps

    LogQL:

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 15.3 Viewing Top Pods

    LogQL:

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 15.4 Viewing Top ERROR Apps

    LogQL:

    topk(10,
      sum by (namespace, app) (
        count_over_time(
          {namespace=~".+"}
            |~ "(?i)error|exception|panic|failed" [5m]
        )
      )
    )

### 15.5 Management Approach

    If the log volume for a particular app is excessively high, follow these steps:

    1. Check if DEBUG logs are enabled.
    2. Verify if there are too many health check logs.
```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"} |~ "(?i)password|token|authorization|cookie|secret|access_key|private_key"' \
      --data-urlencode 'limit=20' | jq

### 17.2 Principles for Managing Sensitive Information

Prohibited from being included in logs:

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
    User privacy fields

Priorities:

    Should not be output on the application side
      ↓
    Filtered at the gateway level
      ↓
    Desensitized during collection
      ↓
    Controlled by Loki permissions
      ↓
    Emergency handling for historical data
```

### 17.3 Emergency Response Measures

In case sensitive information is detected:

    1. Immediately determine the scope of impact.
    2. Stop the output of sensitive logs immediately.
    3. Adjust the log desensitization settings in the application.
    4. Restrict query permissions for Grafana/Loki.
    5. Replace any leaked keys/token.
    6. Evaluate whether it is necessary to delete historical logs.
    7. Document the security incident.
    8. Review log management guidelines and code review processes.
```---

## Chapter Twenty-Two: Common Misconceptions

### 22.1 Misconception 1: Installing Loki Means the Log Platform is Complete

Wrong.

A complete log platform includes at least:

    Collection
    Writing
    Storage
    Querying
    Dashboards
    Alerts
    Retention
    Limits
    Permissions
    Runbooks

### 22.2 Misconception 2: If Loki Cannot Find Logs, It Must Be Broken

Not necessarily.

Possible reasons include:

    The application does not produce stdout logs.
    Alloy has not collected the logs.
    Alloy wrote to the wrong address.
    Labels were incorrectly set.
    The time range is incorrect.
    There is an issue with the Grafana data source.
    Logs were written to another Loki instance.

### 22.3 Misconception 3: The More Logs, the Better

Wrong.

Useless logs can lead to:

    Increased storage costs
    Slower query times
    More noise in alerts
    Greater difficulty in troubleshooting
    Risks associated with sensitive information

### 22.4 Misconception 4: Using request_id/trace_id as Labels Makes Queries Easier

Wrong.

This can cause high cardinality issues.

The correct approach is to include the trace_id in the log body.

Example query:

    {namespace="app-demo", app="json-log-demo"} | json | trace_id="trace-error-10"

### 22.5 Misconception 6: Dashboard Variables Can Be Used for Permission Control

Wrong.

Dashboard variables are not the appropriate means for permission control.

Realistic permission controls require:

    Grafana permissions
    Data source permissions
    Loki multi-tenancy features
    Reverse proxy authentication
    NetworkPolicy settings

### 22.6 Misconception 7: The More Log Alerts, the Better

Wrong.

Excessive log alerts can lead to too much noise.

It is recommended to:

    Focus on metric-based alerts.
    Use log alerts as a secondary tool.
    Trigger alerts only for critical failure-related logs.
    Ensure that each alert has an owner and a corresponding Runbook.

### 22.7 Misconception 8: Retention Settings Are Applied Immediately

Wrong.

Retention depends on the Compactor and involves timing and delays.

Important settings to check include:

    retention_enabled
    retention_period
    delete_request_store
    Object Storage deletion permissions
    retention_delete_delay

---

## Chapter Twenty-Three: Production Implementation Suggestions

### 23.1 Minimum Closed Loop for Small-Scale Production

Essential components include:

    Loki + Object Storage
    Alloy DaemonSet
    Grafana Data Source
    Basic dashboards
    Basic ERROR/timing alerts
    Retention settings
    Limits configuration
    MinIO/S3 capacity monitoring
    Loki self-monitoring
    Runbooks

### 23.2 Enhanced Configuration for Medium-Scale Production

Add the following:

    High-availability deployment of Loki
    Multiple replica Gateways
    More comprehensive limits settings
    More detailed retention streams
    Grafana permissions
    Dashboard with namespace/team dimensions
    AlertManager routing rules
    Application-specific log formatting guidelines
    Regular checks for high cardinality labels and sensitive information

### 23.3 Large-Scale Production Considerations

Consider implementing:

    Microservices/Distributed Loki architecture
    Standalone Object Storage clusters
    Caching mechanisms
    Multi-tenancy support
    Tenant-level limits and retention settings
    Concurrency management for log queries
    Dashboard configuration as code
    GitOps for version control
    Log cost reporting
    Integration with CMDB/permission systems

### 23.4 Suggestions for Log Formatting Standards

Application logs should include the following standard fields:

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

Avoid mixing up similar fields, such as:

    duration/cost/elapsed_time
    level/severity/log_level
    status/code/http_status

### 23.5 Recommendations for Label Formatting

Recommended labels include:

    cluster
    environment
    namespace
    app
    container
    pod
    node
    team

Labels that should be avoided include:

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

## Chapter Twenty-Four: Clearing Up Experimental Resources

If it's just an experimental environment, resources can be cleared as needed.

### 24.1 Deleting Test Applications

    kubectl delete -f nginx-demo.yaml

    kubectl delete -f json-log-demo.yaml

    kubectl delete -f logfmt-log-demo.yaml

Or:

    kubectl deleteGrafana Query and Visualization
      ↓
    Loki Ruler for Log Alert Execution
      ↓
    AlertManager for Alert Receipt and Routing
      ↓
    Runbook for Duty Processing Support

This closed-loop process covers the following components:

    MinIO Object Storage
    Loki Helm Deployment
    Loki Gateway
    Loki Retention Settings
    Loki Limits Configuration
    Grafana Alloy DaemonSet
    Kubernetes Pod Log Collection
    Basic LogQL Queries
    LogQL JSON Queries
    Grafana Explore Functionality
    Grafana Dashboards
    AlertManager
    Loki Ruler
    Log Alerts
    Log Volume Management
    High-Volume Log Handling
    Sensitive Information Detection
    Production-Level Runbooks

To truly master Loki, it’s not enough to simply perform a helm install; one must also be able to answer the following questions:

    Where do the logs come from?
    Who is responsible for collecting them?
    Where are they written?
    Where are they stored?
    How can I query them?
    How can I set up alerts for them?
    For how long should they be retained?
    Who has access to them?
    Who is in charge of managing them?
    How do I troubleshoot any issues that arise?
    How can I control costs related to log management?
    How can I safeguard sensitive information?

After completing this guide, you will have acquired a comprehensive set of skills for building an end-to-end Kubernetes logging platform.

For the next step, it is recommended to explore:

    16-Loki Production-Grade Scaling Design: Multi-Tenancy, Permissions, Cost Management, and Platformization

Key areas to focus on include:

    Multi-Tenancy Architecture
    X-Scope-OrgID Implementation
    Grafana Permission Settings
    Data Source Permission Management
    Namespace/Team Segregation
    Log Cost Tracking
    Tenant-Level Retention Policies
    Tenant-Level Limits
    Dashboard as Code Approach
    GitOps for Management
    Integration with CMDBs and Unified Permission Platforms

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Setting Up Grafana Loki with Helm:
  https://grafana.com/docs/loki/latest/setup/install/helm/

- Loki Deployment Options:
  https://grafana.com/docs/loki/latest/get-started/deployment-modes/

- Loki Architecture Overview:
  https://grafana.com/docs/loki/latest/get-started/architecture/

- Loki Configuration Guidelines:
  https://grafana.com/docs/loki/latest/configure/

- Loki Storage Management:
  https://grafana.com/docs/loki/latest/configure/storage/

- Loki HTTP API Reference:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- How to Query Loki Data:
  https://grafana.com/docs/loki/latest/query/

- Log Queries and Usage:
  https://grafana.com/docs/loki/latest/query/log_queries/

- Metric Queries:
  https://grafana.com/docs/loki/latest/query/metricqueries/

- LogQL Reference Guide:
  https://grafana.com/docs/loki/latest/query/query_reference/

- Best Practices for Querying Data:
  https://grafana.com/docs/loki/latest/query/bp-query/

- Log Retention Strategies:
  https://grafana.com/docs/loki/latest/operations/storage/retention/

- Request Validation and Rate Limiting:
  https://grafana.com/docs/loki/latest/operations/request-validation-rate-limits/

- Loki Alerting and Recording Rules:
  https://grafana.com/docs/loki/latest/alert/

- Grafana Alloy Documentation:
  https://grafana.com/docs/alloy/latest/

- Collecting Kubernetes Logs and Forwarding Them to Loki:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- loki.source.kubernetes Component Reference:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- loki.write Component Reference:
  https://grafana.com/docs/alloy/latest/referencecomponents/loki/loki.write/

- discovery.kubernetes Component Reference:
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.kubernetes/

- discovery.relabel Component Reference:
  https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/

- Grafana Loki Data Source Configuration:
  https://grafana.com/docs/grafana/latest/datasources/loki/

- Grafana Dashboards:
  https://grafana.com/docs/grafana/latest/dashboards/

- Grafana Variables:
  https://grafana.com/docs/grafana/latest/dashboards/variables/

- AlertManager Configuration Guide:
  https://prometheus.io/docs/alerting/latest/configuration/

- AlertManager Concepts:
  https://prometheus.io/docs警报系统/latest/alertmanager/

- MinIO Documentation:
 