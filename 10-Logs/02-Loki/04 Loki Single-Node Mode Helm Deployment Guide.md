# 04-Loki Monolithic Mode Helm Deployment Practice

## Document Explanation

This article is the fourth in the Loki specialty learning series, used to deploy Loki in monolithic mode through Helm in a Kubernetes cluster and complete basic verification.

The focus of this article is not on production-grade high availability deployment, but to first run the minimum available chain of Loki:

    Helm Install Loki
      ↓
    Loki Pod Running
      ↓
    Loki Service Normal
      ↓
    /ready Returns ready
      ↓
    /metrics Accessible
      ↓
    Manually Write a Test Log
      ↓
    Query the Test Log via LogQL

This article temporarily does not integrate with Grafana Alloy to collect Pod logs.

Reasons:

    The 04th article first verifies the availability of the Loki server.
    The 05th article integrates with MinIO object storage.
    The 06th article then integrates Grafana Alloy to collect K8S Pod logs.

This article focuses on answering the following questions:

- How to use Helm to install Loki in monolithic mode;
- Whether to use the grafana-community or grafana Helm Chart repository currently;
- How to fix the Loki Chart version;
- How to export default values;
- How to write a monolithic mode values.yaml;
- How to use helm template to pre-check resources;
- How to install Loki;
- How to verify Loki Pod, Service, Endpoint;
- How to access Loki /ready, /metrics;
- How to manually write a test log;
- How to query logs using query_range;
- How to troubleshoot when Loki fails to start, is in Pending, CrashLoopBackOff, or the Service is unreachable;
- Why the monolithic mode is suitable for learning but cannot be directly used as a production final solution.

---

## Tags

#Loki #Grafana #Kubernetes #Helm #LogSystem #SingleMode #Monolithic #SingleBinary #LogQL #SRE #Observation #LogCollection

---

## Recommended Path

Recommended path:

    10-logs/02-Loki/04-Loki Monolithic Mode Helm Deployment Practice.md

---

## OneI don't know.Experiment Objectives

After completing this article, you should be able to:

    1. Create the logging Namespace.
    2. Add the grafana-community Helm repository.
    3. View the Loki Chart version.
    4. Export the default Loki values.
    5. Write a monolithic mode values.yaml.
    6. Use helm template to pre-check YAML.
    7. Use helm install to deploy Loki.
    8. View Loki Pod / Service / Endpoint.
    9. Use port-forward to access Loki.
    10. Verify /ready.
    11. Verify /metrics.
    12. Manually write a test log.
    13. Query the test log.
    14. Master the basic troubleshooting methods for Loki monolithic mode.
    15. Clarify why the 05th article needs to integrate MinIO object storage.

---

## TwoI don't know.Experiment Environment

### 2.1 Kubernetes Environment

Experiment node planning:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespace:

    logging

Container runtime:

    containerd

Tools:

    kubectl
    helm
    curl
    jq
    grep

### 2.2 Version Requirements

Recommendations:

    Kubernetes >= 1.25
    Helm 3.x
    Loki uses a fixed Helm Chart version

Check versions:

    kubectl version --client

    kubectl get nodes -o wide

    helm version

Notes:

    Do not use latest blindly.
    The Helm Chart version must be fixed.
    When writing to the knowledge base later, record the chart version and appVersion.

---

## ThreeI don't know.Current Helm Repository Selection

### 3.1 New Repository Recommendation

The current new version Loki Community Chart recommends using:

    grafana-community

Add command:

    helm repo add grafana-community https://grafana-community.github.io/helm-charts

    helm repo update

View:

    helm search repo grafana-community/loki --versions

### 3.2 Old Repository Notes

Old documents or old environments may still see:

    helm repo add grafana https://grafana.github.io/helm-charts

    helm install loki grafana/loki

Notes:

    The old repository is still common in historical environments.
    New learning and new deployment are recommended to prioritize grafana-community/loki.
    If the company has an old version of Loki, you need to first confirm the existing Chart version and do not directly upgrade across major versions.

### 3.3 This Article Uses

This article uses:

    grafana-community/loki

Example version variable:

    CHART_VERSION="<version obtained by helm search repo>"

In actual operation, it is recommended to write as:

    helm search repo grafana-community/loki --versions | head

Then select a specific version.

Example:

    helm install loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml

Notes:

    <CHART_VERSION> should not be copied verbatim.
    It must be replaced with the actual version found.
    It is recommended to record this version in notes or Git commits.

---

## FourI don't know.Monolithic Mode Deployment Approach

### 4.1 This Article Uses Monolithic Mode

This article uses:

    Monolithic / Monolithic Mode

Its characteristics:

A single Loki process assumes the responsibilities of multiple components.
Kubernetes resources are limited.
Suitable for learning and small-scale experiments.
Easier to understand the writing, querying, and troubleshooting workflow.

In the monolithic mode, the following components are still logically present:

    Distributor
    Ingester
    Querier
    Query Frontend
    Compactor
    Ruler

But they may not appear as independent Pods.

### 4.2 This article temporarily does not use MinIO

This article first uses the monolithic mode to run the Loki server.

Chapter 05 will specifically cover:

    Loki integration with MinIO object storage

This learning path is clearer:

    Chapter 04:
        First make the Loki server able to start, write, and query.

    Chapter 05:
        Then upgrade the storage from learning mode to object storage.

    Chapter 06:
        Then integrate Alloy to collect real Pod logs.

### 4.3 Production Notes

Learning environments can use simpler storage.

Production environments should prioritize:

    Object storage
    Multi-replica
    High availability
    Query limits
    Write throttling
    Self-monitoring
    Log retention period
    Alerts
    Backup and recovery
    Access control

Do not directly apply the values from this article to production.

---

## Five. Prepare Namespace

### 5.1 Create logging Namespace

    kubectl create namespace logging

If it already exists, it will prompt:

    AlreadyExists

You can ignore this.

### 5.2 Verify Namespace

    kubectl get ns logging

Expected output:

    NAME      STATUS   AGE
    logging   Active   ...

---

## Six. Add Helm Repository

### 6.1 Add grafana-community repository

    helm repo add grafana-community https://grafana-community.github.io/helm-charts

    helm repo update

### 6.2 View repository

    helm repo list

Expected to see:

    grafana-community    https://grafana-community.github.io/helm-charts

### 6.3 Query Loki Chart version

    helm search repo grafana-community/loki --versions | head -20

Record:

    Chart Version:
        <actual version>

    App Version:
        <actual version>

It is recommended to document in your experiment records:

    This experiment uses:
        grafana-community/loki chart version: x.x.x
        Loki appVersion: x.x.x

---

## Seven. Export Default values

### 7.1 Export default values

    helm show values grafana-community/loki \
      --version <CHART_VERSION> \
      > values-loki-default.yaml

### 7.2 Search for key fields

    grep -n "deploymentMode" values-loki-default.yaml

    grep -n "Monolithic" values-loki-default.yaml

    grep -n "SingleBinary" values-loki-default.yaml

    grep -n "monolithic" values-loki-default.yaml

    grep -n "singleBinary" values-loki-default.yaml

    grep -n "gateway:" values-loki-default.yaml

    grep -n "minio:" values-loki-default.yaml

    grep -n "schemaConfig" values-loki-default.yaml

    grep -n "commonConfig" values-loki-default.yaml

### 7.3 Why it is necessary to first view default values

The fields in different versions of Helm Chart may change.

For example:

    Old versions may use singleBinary.
    New versions may use monolithic / Monolithic.
    The values of deploymentMode may vary with Chart version.
    Fields like monitoring.selfMonitoring may change or be removed in new versions.

Therefore:

    Do not directly copy values from old articles.
    Do not directly copy values from others' clusters.
    Must use the current helm show values output as the basis.

---

## Eight. Write monolithic mode values file

### 8.1 Create values file

Create the file:

    values-loki-monolithic.yaml

### 8.2 Recommended writing: Current community Chart approach

The following values are for learning environments.

Note:

    If helm template reports a field does not exist, you need to refer back to values-loki-default.yaml to adjust the current Chart fields.
    This article focuses on the monolithic deployment method and does not aim to cover all Chart version differences.

Content:

    loki:
      auth_enabled: false

      commonConfig:
        replication_factor: 1

      schemaConfig:
        configs:
          - from: "2024-04-01"
            store: tsdb
            object_store: filesystem
            schema: v13
            index:
              prefix: loki_index_
              period: 24h

      storage_config:
        filesystem:
          directory: /var/loki/chunks

      ruler:
        enable_api: true

limits_config:
  allow_structured_metadata: true
  volume_enabled: true

deploymentMode: Monolithic

monolithic:
  replicas: 1
  persistence:
    enabled: true
    size: 10Gi

gateway:
  enabled: true

lokiCanary:
  enabled: false

minio:
  enabled: false

backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0

ingester:
  replicas: 0
querier:
  replicas: 0
queryFrontend:
  replicas: 0
queryScheduler:
  replicas: 0
distributor:
  replicas: 0
compactor:
  replicas: 0
indexGateway:
  replicas: 0
bloomPlanner:
  replicas: 0
bloomBuilder:
  replicas: 0
bloomGateway:
  replicas: 0

### 8.3 If the current Chart still uses singleBinary

If your default values do not contain:

  monolithic

But contain:

  singleBinary

You can use the compatible syntax:

  loki:
    auth_enabled: false

    commonConfig:
      replication_factor: 1

    schemaConfig:
      configs:
        - from: "2024-04-01"
          store: tsdb
          object_store: filesystem
          schema: v13
          index:
            prefix: loki_index_
            period: 24h

    storage_config:
      filesystem:
        directory: /var/loki/chunks

    ruler:
      enable_api: true

    limits_config:
      allow_structured_metadata: true
      volume_enabled: true

  deploymentMode: SingleBinary

  singleBinary:
    replicas: 1
    persistence:
      enabled: true
      size: 10Gi

  gateway:
    enabled: true

  lokiCanary:
    enabled: false

  minio:
    enabled: false

  backend:
    replicas: 0
  read:
    replicas: 0
  write:
    replicas: 0

  ingester:
    replicas: 0
  querier:
    replicas: 0
  queryFrontend:
    replicas: 0
  queryScheduler:
    replicas: 0
  distributor:
    replicas: 0
  compactor:
    replicas: 0
  indexGateway:
    replicas: 0
  bloomPlanner:
    replicas: 0
  bloomBuilder:
    replicas: 0
  bloomGateway:
    replicas: 0

### 8.4 values Field Explanation

  loki.auth_enabled: false

Indicates multi-tenant authentication is disabled.

It's acceptable to disable in learning environments.

Not recommended to disable blindly in production environments.

  commonConfig.replication_factor: 1

Single-replica Loki must be set to 1.

If set to a value greater than 1 with only one replica, writes may fail.

  schemaConfig

Defines the schema Loki uses.

Here, we use:

  tsdb
  schema v13

  object_store: filesystem

Indicates using filesystem storage first.

  storage_config.filesystem.directory

Specifies the Loki chunk storage directory.

  deploymentMode: Monolithic

Specifies the monolithic mode.

  monolithic.replicas: 1

Deploys one Loki monolithic replica.

  gateway.enabled: true

Enables Loki Gateway.

In learning scenarios, you can expose a unified entry point via gateway.

  minio.enabled: false

MinIO is temporarily disabled in Chapter 04.

MinIO will be separately integrated in Chapter 05.

  backend/read/write/ingester/querier etc. replicas: 0

Prevents Helm from creating components of other deployment modes simultaneously.

---

## Nine. Helm Rendering Check

Before installation, render YAML first.

### 9.1 Execute helm template

  helm template loki grafana-community/loki \
    --namespace logging \
    --version <CHART_VERSION> \
    -f values-loki-monolithic.yaml \
    > loki-monolithic-rendered.yaml

### 9.2 View resource types /think

grep "^kind:" loki-monolithic-rendered.yaml | sort | uniq -c

### 9.3 View Workloads

    grep -n "kind: StatefulSet" loki-monolithic-rendered.yaml

    grep -n "kind: Deployment" loki-monolithic-rendered.yaml

    grep -n "kind: DaemonSet" loki-monolithic-rendered.yaml

### 9.4 View Service

    grep -n "kind: Service" loki-monolithic-rendered.yaml

### 9.5 View Images

    grep -n "image:" loki-monolithic-rendered.yaml | sort -u

If pulling images in domestic environments is difficult, first record these image details, and consider synchronizing them to an internal image repository later.

### 9.6 Common Rendering Errors

#### Field Not Found

Phenomenon:

    template error
    nil pointer
    field not found

Resolution:

    helm show values grafana-community/loki --version <CHART_VERSION> > values-loki-default.yaml

Recheck the fields:

    grep -n "monolithic" values-loki-default.yaml

    grep -n "singleBinary" values-loki-default.yaml

#### Unsupported deploymentMode

Phenomenon:

    deploymentMode value invalid

Resolution:

    Check the deployment mode names supported by the current Chart.
    The current version may still use SingleBinary.
    The current version may have already switched to Monolithic.

#### Mismatched Storage Configuration

Phenomenon:

    object_store
    schemaConfig
    storage_config
    filesystem

Related error messages.

Resolution:

    First use helm template to check.
    Then view Loki Pod startup logs.
    Change to the official current version example configuration if necessary.

---

## Ten. Execute Helm Installation

### 10.1 Install Loki

    helm install loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml

### 10.2 View Helm Release

    helm list -n logging

Expected output:

    NAME    NAMESPACE   REVISION    STATUS      CHART       APP VERSION
    loki    logging     1           deployed    loki-...    ...

### 10.3 View Values

    helm get values loki -n logging

View full values:

    helm get values loki -n logging -a

Notes:

    helm get values can be used for subsequent troubleshooting and review.
    Must back up values before production changes.

---

## Eleven. View Kubernetes Resources

### 11.1 View Pods

    kubectl get pods -n logging -o wide

Expected to see:

    loki-0
    loki-gateway-xxx

Or similar names.

The exact name depends on the Chart version.

### 11.2 View StatefulSet / Deployment

    kubectl get statefulset -n logging

    kubectl get deployment -n logging

### 11.3 View Service

    kubectl get svc -n logging

Focus on:

    loki
    loki-gateway
    loki-headless
    loki-memberlist

Names may differ across versions.

### 11.4 View Endpoints

    kubectl get endpoints -n logging

    kubectl get endpointslice -n logging

Confirm:

    Service has backend endpoints.
    Endpoints point to Loki Pods.
    Gateway has backend endpoints.

### 11.5 View PVC

If persistence is enabled:

    kubectl get pvc -n logging

Expected:

    Loki-related PVCs are Bound.

If Pending:

    kubectl describe pvc <pvc-name> -n logging

Common causes:

    No default StorageClass.
    StorageClass does not exist.
    Dynamic provisioner failure.
    PVC accessModes mismatch.

---

## Twelve. Verify Loki Pod Status

### 12.1 View Pod Details

    kubectl describe pod <loki-pod-name> -n logging

Focus on:

    State
    Ready
    Restart Count
    Events
    Image
    Mounts
    Volumes
    Liveness Probe
    Readiness Probe

### 12.2 View Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=200

Continuous monitoring:

    kubectl logs <loki-pod-name> -n logging -f

Filter errors:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|panic|storage|ring|ready|memberlist"

### 12.3 Common Log Focus Points /think

Normal Direction:

    server listening
    module started
    ready
    running

Abnormal Direction:

    failed to create object client
    permission denied
    invalid schema config
    failed parsing config
    mkdir permission denied
    memberlist error
    ring not healthy
    timeout
    panic

---

## ThirteenI don't know.Accessing Loki /ready

### 13.1 Port Forwarding Method One: Forward Loki Service

First check the Service:

    kubectl get svc -n logging

If there is a loki Service:

    kubectl port-forward svc/loki 3100:3100 -n logging

Access:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

### 13.2 Port Forwarding Method Two: Forward Gateway

If using gateway, the Service port may be 80.

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Access:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

### 13.3 If /ready Does Not Return "ready"

Troubleshoot:

    kubectl get pods -n logging -o wide

    kubectl describe pod <loki-pod-name> -n logging

    kubectl logs <loki-pod-name> -n logging --tail=200

Possible Causes:

    Loki configuration error.
    Loki Pod is not Ready.
    Storage directory permission anomaly.
    PVC is not mounted.
    ring is not ready.
    Gateway forwarding anomaly.
    Port forwarding to the wrong Service.

---

## FourteenI don't know.Accessing Loki /metrics

### 14.1 Query Metrics

After maintaining port-forward, execute:

    curl -s http://127.0.0.1:3100/metrics | head

Filter Loki metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^loki_" | head -50

Filter Go metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^go_" | head

Filter process metrics:

    curl -s http://127.0.0.1:3100/metrics | grep "^process_" | head

### 14.2 Why Check /metrics

Loki itself must also be monitored.

In production, Prometheus should scrape Loki's /metrics.

Key Monitoring Directions:

    Whether Loki is Ready
    Number of write requests
    Number of failed writes
    Number of queries
    Number of failed queries
    Query latency
    Ingester active streams
    Chunk flush
    Compactor status
    Ruler status
    Gateway 5xx

---

## FifteenI don't know.Manually Write a Test Log

Alloy will be used in Chapter 06 to collect real Pod logs.

However, in Chapter 04, you can manually write a test log via Loki API to verify Loki's ability to write and query.

### 15.1 Prepare Timestamp

Loki write API requires nanoseconds timestamp.

Execute:

    TS=$(date +%s%N)

View:

    echo $TS

### 15.2 Write Test Log

Execute:

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"loki-manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki from manual curl test\"]
            ]
          }
        ]
      }"

If there is no output, it usually indicates success.

### 15.3 Query Written Results

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-test"}' \
      --data-urlencode 'limit=10' | jq

Expected to see:

    hello loki from manual curl test

### 15.4 Query Specific Content

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-test"} |= "hello loki"' \
      --data-urlencode 'limit=10' | jq

### 15.5 If Not Found

Troubleshoot:

    1. Is /ready ready?
    2. Was the push request successful?
    3. Is the time range correct?
    4. Is the query label correct?
    5. Is auth_enabled false?
    6. Is the correct path forwarded via gateway?
    7. Are there write errors in Loki logs?

Check Loki logs:

    kubectl logs <loki-pod-name> -n logging --tail=200 | grep -Ei "error|push|ingest|stream|label"

---

## SixteenI don't know.Using Loki API to Query Labels

### 16.1 Query Label Names

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Expected to see:

    app
    job
    namespace

### 16.2 Query Job Label Values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/job/values" | jq

manual-test

### 16.3 Query namespace label values

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

Expected:

    app-demo

### 16.4 Significance

If labels can be queried, it indicates:

    Loki already has log streams.
    Label indexing is available.
    query_range can query via label selectors.

Pod log labels like pod, container, node will appear only after integrating with Alloy in Chapter 06.

---

## Seventeen. Verify Gateway

### 17.1 View Gateway Pod

    kubectl get pods -n logging | grep gateway

### 17.2 View Gateway Service

    kubectl get svc -n logging | grep gateway

### 17.3 View Gateway Logs

    kubectl logs <loki-gateway-pod> -n logging --tail=100

### 17.4 Access Loki via Gateway

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

    curl -s http://127.0.0.1:3100/metrics | head

### 17.5 Common Gateway Issues

If Gateway returns 502 / 503:

    Backend Loki Service is unreachable.
    Loki Pod is not Ready.
    Service selector mismatch.
    Endpoint is empty.
    Gateway configuration path error.
    Loki internal port is incorrect.

Troubleshooting:

    kubectl get svc -n logging

    kubectl get endpoints -n logging

    kubectl describe svc loki-gateway -n logging

    kubectl logs <loki-gateway-pod> -n logging --tail=200

---

## Eighteen. Post-Installation Resource Checklist

### 18.1 Helm

    helm list -n logging

    helm get values loki -n logging

    helm status loki -n logging

### 18.2 Kubernetes

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl get endpoints -n logging

    kubectl get pvc -n logging

    kubectl get cm -n logging

    kubectl get secret -n logging

### 18.3 Loki API

    curl -s http://127.0.0.1:3100/ready

    curl -s http://127.0.0.1:3100/metrics | head

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

### 18.4 Write and Query

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"loki-manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki from manual curl test\"]
            ]
          }
        ]
      }"

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-test"}' \
      --data-urlencode 'limit=10' | jq

---

## Nineteen. Common Troubleshooting

### 19.1 Helm install failed

Phenomenon:

    INSTALLATION FAILED

Troubleshooting:

    helm repo list

    helm search repo grafana-community/loki --versions

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml

Common causes:

    Chart version is incorrect.
    Values fields mismatch.
    deploymentMode is incorrect.
    YAML indentation error.
    Kubernetes version does not meet requirements.
    CRD or API version incompatibility.

### 19.2 Pod Pending

Check:

    kubectl get pods -n logging -o wide

    kubectl describe pod <pod-name> -n logging

Common causes:

    CPU insufficient.
    Memory insufficient.
    PVC Pending.
    StorageClass does not exist.
    nodeSelector mismatch.
    taint/toleration mismatch.

If PVC is Pending:

    kubectl get pvc -n logging

    kubectl describe pvc <pvc-name> -n logging

Resolution:

    Check default StorageClass.
    Configure available StorageClass.
    Or temporarily disable persistence for learning verification.

### 19.3 Pod CrashLoopBackOff

Check:

kubectl describe pod <loki-pod-name> -n logging

kubectl logs <loki-pod-name> -n logging --previous --tail=200

Common Causes:

    Loki configuration error.
    storage_config configuration error.
    schemaConfig configuration error.
    File directory has no permissions.
    PVC mounting anomaly.
    deploymentMode conflicts with replica configuration.
    replication_factor does not match the replica count.

Key Points:

    Single replica must have commonConfig.replication_factor: 1.

### 19.4 /ready Not Ready

Check:

    kubectl get pods -n logging

    kubectl logs <loki-pod-name> -n logging --tail=200

    kubectl get endpoints -n logging

Common Causes:

    Loki is still starting.
    Ring is not ready.
    Storage initialization failed.
    Configuration error.
    Gateway forwarding anomaly.

### 19.5 /metrics Not Accessible

Check:

    Whether port forwarding is still running.
    Whether the service name is correct.
    Whether the service port is correct.
    Whether it's forwarding Loki Service or Gateway Service.
    Whether the pod is Ready.

Commands:

    kubectl get svc -n logging

    kubectl describe svc <service-name> -n logging

### 19.6 Manual Push Failed

Possible Symptoms:

    400 Bad Request
    404 Not Found
    429 Too Many Requests
    500 Internal Server Error

Judgment:

    400:
        JSON format error, timestamp error, invalid label.

    404:
        Path error, confirm whether it's /loki/api/v1/push.

    429:
        Write rate limiting.

    500:
        Loki internal error or storage error.

Troubleshoot:

    kubectl logs <loki-pod-name> -n logging --tail=200

### 19.7 query_range Cannot Find Data

Check:

    Whether the labels are written correctly.
    Whether the time is within the query range.
    Whether the logs were written successfully.
    Whether the wrong gateway was queried.
    Whether auth_enabled affects tenant.
    Whether there is a LogQL syntax error.

First check labels:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

Then check job:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/job/values" | jq

---

## Twenty, Upgrade and Rollback Basics

### 20.1 View Current Release

    helm list -n logging

    helm status loki -n logging

### 20.2 View Historical Versions

    helm history loki -n logging

### 20.3 Upgrade After Modifying Values

    helm upgrade loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml

### 20.4 Rollback

    helm rollback loki <REVISION> -n logging

### 20.5 Backup Values Before Upgrade

    helm get values loki -n logging -a > backup-loki-values.yaml

Before production upgrade, also back up:

    PVC
    Object storage data
    Current Chart version
    Current Loki appVersion
    Current running configuration
    Current Service / Ingress configuration

---

## Twenty-one, Uninstallation and Cleanup

### 21.1 Uninstall Loki

    helm uninstall loki -n logging

### 21.2 Check for Residual Resources

    kubectl get all -n logging

    kubectl get pvc -n logging

    kubectl get cm -n logging

    kubectl get secret -n logging

### 21.3 Whether to Delete PVC

In learning environments, you can delete:

    kubectl delete pvc <pvc-name> -n logging

Do not delete PVCs in production environments.

Reasons:

    PVC may contain log data.
    Data may not be recoverable after deletion.
    Need to confirm backup and retention policies first.

### 21.4 Delete Namespace

After confirming it's no longer needed:

    kubectl delete namespace logging

Note:

    Deleting a namespace will delete all resources within it.
    Prohibited in production environments.

---

## Twenty-two, Production Considerations

### 22.1 Monolithic Mode Is Not the Final Answer for Production

Monolithic mode is suitable for:

    Learning
    Small-scale experiments
    Quick validation
    Internal low-logging environment

Production requires evaluation:

    Logging volume
    Query concurrency
    Retention period
    High availability
    Object storage
    Rate limiting
    Query control
    Alerts
    Self-monitoring
    Backup and recovery

### 22.2 File System Storage Is Only Suitable for Learning or Low-risk Scenarios

This article uses the file system approach to get Loki running first.

But production recommends:

    S3
    MinIO
    OSS
    COS
    OBS
    GCS
    Azure Blob

MinIO will be integrated in:

    Chapter 05

### 22.3 Must Fix Version

In production, must fix:

    Helm Chart version
    Loki appVersion
    values.yaml
    Image version

Do not use:

    latest

### 22.4 Must Monitor Loki Itself

At least monitor: /think

Loki Pod Ready  
Loki Gateway 5xx  
Loki Write Failure  
Loki Query Failure  
Loki Query Latency  
Ingester active streams  
Loki PVC Usage  
Loki Object Storage Error  
Ruler Execution Failure  
Compactor Execution Failure  

### 22.5 Must Manage values.yaml  

Recommendations:  

    values-loki-monolithic.yaml should be included in Git.  
    All changes should go through Git diff.  
    Backup helm get values before every upgrade.  
    Verify /ready, /metrics, push, and query after every upgrade.  

---  

## Twenty-Three, Hands-on Tasks  

### 23.1 Task One: Prepare Helm Repository  

    helm repo add grafana-community https://grafana-community.github.io/helm-charts  

    helm repo update  

    helm search repo grafana-community/loki --versions | head -20  

Verification:  

    [ ] Can find grafana-community/loki  
    [ ] Fixed CHART_VERSION has been selected  

### 23.2 Task Two: Export Default Values  

    helm show values grafana-community/loki \
      --version <CHART_VERSION> \
      > values-loki-default.yaml  

Verification:  

    [ ] values-loki-default.yaml has been saved  
    [ ] deploymentMode field has been confirmed  
    [ ] monolithic or singleBinary field has been confirmed  
    [ ] gateway field has been confirmed  
    [ ] storage-related fields have been confirmed  

### 23.3 Task Three: Write values File  

File:  

    values-loki-monolithic.yaml  

Verification:  

    [ ] auth_enabled has been disabled according to the learning environment  
    [ ] replication_factor has been set to 1  
    [ ] deploymentMode has been set to monolithic mode  
    [ ] Other mode replicas have been set to 0  
    [ ] gateway has been enabled  
    [ ] minio has not been enabled  

### 23.4 Task Four: Render Check  

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml \
      > loki-monolithic-rendered.yaml  

Verification:  

    [ ] helm template has no errors  
    [ ] Can see Loki StatefulSet / Deployment  
    [ ] Can see Loki Service  
    [ ] Can see Gateway Service  
    [ ] Can see the used image  

### 23.5 Task Five: Install Loki  

    kubectl create namespace logging  

    helm install loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml  

Verification:  

    [ ] helm list -n logging shows deployed  
    [ ] Loki Pod Running  
    [ ] Loki Service is normal  
    [ ] Endpoint is not empty  

### 23.6 Task Six: Verify Loki API  

    kubectl port-forward svc/loki-gateway 3100:80 -n logging  

    curl -s http://127.0.0.1:3100/ready  

    curl -s http://127.0.0.1:3100/metrics | head  

Verification:  

    [ ] /ready returns ready  
    [ ] /metrics has output  

### 23.7 Task Seven: Manual Write and Query  

    TS=$(date +%s%N)  

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"loki-manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki from manual curl test\"]
            ]
          }
        ]
      }"  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-test"}' \
      --data-urlencode 'limit=10' | jq  

Verification:  

    [ ] Can query \"hello loki from manual curl test\"  

---  

## Twenty-Four, Verification Checklist  

After completing this document, you should confirm:

[ ] logging Namespace has been created  
[ ] grafana-community Helm repository has been added  
[ ] Loki Chart version has been fixed  
[ ] values-loki-default.yaml has been exported  
[ ] values-loki-monolithic.yaml has been written  
[ ] helm template check has passed  
[ ] helm install has been successful  
[ ] helm list shows deployed  
[ ] Loki Pod is Running  
[ ] Loki Pod is Ready  
[ ] Loki Service exists  
[ ] Gateway Service exists  
[ ] Endpoint is not empty  
[ ] PVC is Bound  
[ ] /ready returns ready  
[ ] /metrics is accessible  
[ ] manual push has been successful  
[ ] query_range can retrieve test logs  
[ ] can view Loki Pod logs  
[ ] understand that monolithic mode is only suitable for learning and small-scale verification  
[ ] have clearly identified that Chapter 05 will integrate MinIO object storage  

---

## 25. Common Misconceptions  

### 25.1 Misconception 1: Pod Running indicates Loki is fully operational  

Error.  

Need to verify:  

    /ready  
    /metrics  
    push  
    query_range  
    labels  
    Gateway  
    PVC  
    Loki logs  

### 25.2 Misconception 2: Loki logs can be viewed after deployment  

Error.  

Loki is a server-side component.  

To view Pod logs, you need a collector:  

    Grafana Alloy  
    Promtail  
    Fluent Bit  
    Filebeat  

This article only deploys Loki.  

Chapter 06 will integrate Alloy to collect Pod logs.  

### 25.3 Misconception 3: Prometheus collects Loki logs  

Error.  

Prometheus collects metrics.  

Loki stores logs.  

Prometheus can collect Loki's own /metrics, but not business logs.  

### 25.4 Misconception 4: Learning values can be directly used in production  

Error.  

Production requires additional components:  

    Object storage  
    High availability  
    limits  
    retention  
    monitoring  
    alerts  
    security  
    backups  
    access control  

### 25.5 Misconception 5: Chart fields remain unchanged  

Error.  

Loki Helm Chart fields change with versions.  

Must use the current version:  

    helm show values  

as reference.  

---

## 26. Summary  

This article completed the Helm deployment of Loki monolithic mode.  

Core chain:  

    grafana-community Helm Repo  
      ↓  
    values-loki-monolithic.yaml  
      ↓  
    helm template pre-check  
      ↓  
    helm install  
      ↓  
    Loki Pod / Service / Gateway  
      ↓  
    /ready  
      ↓  
    /metrics  
      ↓  
    /loki/api/v1/push  
      ↓  
    /loki/api/v1/query_range  

The focus of this article is not to achieve production-ready state, but to confirm:  

    Loki server can start  
    Loki API is accessible  
    Loki can receive logs  
    Loki can query logs  

Currently, no real Kubernetes Pod logs are being collected.  

Future roadmap:  

    Chapter 05:  
        Loki object storage integration with MinIO  

    Chapter 06:  
        Grafana Alloy collection of K8S Pod logs  

    Chapter 08:  
        LogQL basic query practice  

    Chapter 10:  
        Grafana integration with Loki and log dashboard  

Production-grade Loki is more than just helm install.  

What truly needs to be mastered:  

    Reliable storage  
    Stable log collection  
    Reasonable label design  
    Controllable query range  
    Accurate alerts  
    Complete self-monitoring  
    Clear troubleshooting path  

---

## Reference Documents  

- Grafana Loki Documentation:  
  https://grafana.com/docs/loki/latest/  

- Install Grafana Loki with Helm:  
  https://grafana.com/docs/loki/latest/setup/install/helm/  

- Install the monolithic Helm chart:  
  https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/  

- Configure storage:  
  https://grafana.com/docs/loki/latest/setup/install/helm/configure-storage/  

- Upgrade to Grafana Community Helm chart:  
  https://grafana.com/docs/loki/latest/setup/upgrade/upgrade-to-community/  

- Loki Helm Chart:  
  https://github.com/grafana/loki/tree/main/production/helm/loki  

- Grafana Community Helm Charts:  
  https://github.com/grafana-community/helm-charts  

- Helm Documentation:  
  https://helm.sh/docs/  

- Kubernetes Logging Architecture:  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/  

- Kubernetes kubectl port-forward:  
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/