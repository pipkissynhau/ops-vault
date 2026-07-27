# 04-Loki Standalone Mode Helm Deployment Practice

## Document Description

This article is the fourth in the Loki-focused learning series, aimed at deploying Loki in standalone mode using Helm within a Kubernetes cluster and performing basic verifications.

The focus of this article is not on production-grade high-availability deployment but rather on ensuring that the minimum viable setup of Loki is successfully operational:

    Helm installs Loki
      ↓
    Loki Pod is running
      ↓
    Loki Service is functioning properly
      ↓
    The /ready endpoint returns a "ready" status
      ↓
    The /metrics endpoint is accessible
      ↓
    Manually write a test log
      ↓
    Query the test log using LogQL

This article does not currently integrate Grafana Alloy for logging collection from Pods.

Reasons:

    In Article 04, we first need to verify that the Loki server is operational.
    In Article 05, we will set up MinIO object storage.
    In Article 06, we will integrate Grafana Alloy for collecting Kubernetes Pod logs.

This article addresses the following key questions:

- How to use Helm to deploy Loki in standalone mode;
- Whether to use the grafana-community or grafana Helm repository for current Loki charts;
- How to fix the version of the Loki chart;
- How to export default configuration values;
- How to create a values.yaml file for standalone mode;
- How to use helm templates for pre-checking resource configurations;
- How to install Loki;
- How to verify the Loki Pod, Service, and endpoints;
- How to access the /ready and /metrics endpoints of Loki;
- How to manually write a test log;
- How to query logs using query_range;
- How to troubleshoot issues such as Loki startup failures, Pending status, CrashLoopBackOff, or non-functioning Services;
- Why standalone mode is suitable for learning but not directly applicable in production.

---

## Tags

#Loki #Grafana #Kubernetes #Helm #Log System #Monolithic Mode #SingleBinary #LogQL #SRE #Observability #Log Collection

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/04-Loki Standalone Mode Helm Deployment Practice.md

---

## I. Experiment Objectives

After completing this article, you should be able to:

    1. Create a logging Namespace.
    2. Add the grafana-community Helm repository.
    3. Check the version of the Loki chart.
    4. Export default configuration values for Loki.
    5. Create a values.yaml file for standalone mode.
    6. Use helm templates to pre-check the YAML file.
    7. Deploy Loki using helm install.
    8. Verify the Loki Pod, Service, and endpoints.
    9. Access Loki through port-forwarding.
    10. Verify the /ready and /metrics endpoints.
    11. Manually write a test log.
    12. Query the test log.
    13. Master basic troubleshooting methods for Loki in standalone mode.
    14. Understand why MinIO object storage is needed in Article 05.

---

## II. Experiment Environment

### 2.1 Kubernetes Environment

Planned experiment nodes:

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
    Use a fixed version of the Loki Helm chart

Check versions:

    kubectl version --client

    kubectl get nodes -o wide

    helm version

Notes:

    Do not blindly use the latest version.
    The version of the Helm chart must be fixed.
    Record the chart version and app version in the knowledge base.

---

## III. Current Helm Repository Selection

### 3.1 Recommended New Repository

For the new version of the Loki Community Chart, it is recommended to use:

    grafana-community

Add command:

    helm repo add grafana-community https://grafana-community.github.io/helm-charts

    helm repo update

Check versions:

    helm search repo grafana-community/loki --versions

### 3.2 Explanation for Old Repositories

Old documents or environments may still refer to:

    helm repo add grafana https://grafana.github.io/helm-charts

    helm install loki grafana/loki

Note:

    Old repositories are still commonly used in historical contexts.
    For new learning and deployments, it is recommended to use grafana-community/lo> values-loki-default.yaml

### 7.2 Searching for Key Fields

    grep -n "deploymentMode" values-loki-default.yaml

    grep -n "Monolithic" values-loki-default.yaml

    grep -n "SingleBinary" values-loki-default.yaml

    grep -n "monolithic" values-loki-default.yaml

    grep -n "singleBinary" values-loki-default.yaml

    grep -n "gateway:" values-loki-default.yaml

    grep -n "minio:" values-loki-default.yaml

    grep -n "schemaConfig" values-loki-default.yaml

    grep -n "commonConfig" values-loki-default.yaml

### 7.3 Why You Must Check the Default Values First

Field definitions in different versions of Helm Charts may vary.

For example:

    Older versions might use singleBinary.
    Newer versions might use monolithic or Monolithic.
    The value of deploymentMode can change depending on the Chart version.
    Fields like monitoring.selfMonitoring may be changed or removed in newer versions.

Therefore:

    Do not directly copy values from old documents.
    Do not directly copy values from someone else's cluster.
    Always refer to the output of helm show values for your current setup.

---

## VIII. Creating Values Files for Monolithic Mode

### 8.1 Creating a Values File

Create the file:

    values-loki-monolithic.yaml

### 8.2 Recommended Approach: Current Community Chart Practices

The following values are for learning purposes only.

Note:

    If the helm template reports that a field does not exist, you need to check values-loki-default.yaml and adjust it according to the current Chart fields.
    This document focuses on monolithic deployment methods and does not aim to cover all version differences in Charts.

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

### 8.3 If Your Current Chart Still Uses singleBinary

If your default values file does not contain:

    monolithic

but does include:

    singleBinary

you can use the following compatible configuration:

    loki:
      authenabled: false

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
### 14.2 Why Check /metrics?

Monitoring metrics is essential for understanding the performance and health of your Kubernetes application, especially for services like Loki. By checking these metrics, you can identify potential issues, optimize resource usage, and ensure that your service is running smoothly. Some key reasons for monitoring metrics include:Loki itself must also be monitored. In subsequent production, Prometheus should capture Loki's /metrics data.

Key monitoring areas include:

- Whether Loki is ready
- Number of write requests
- Number of write failures
- Number of query requests
- Number of query failures
- Query latency
- Active streams in the ingester
- Chunk flushing
- Status of the compactor
- Status of the Ruler
- Gateway 5xx errors

---

## Section Fifteen: Manually Writing a Test Log

Real Pod logs will be collected using Alloy in Chapter Six. However, in Chapter Four, you can manually write a test log through the Loki API to verify that the Loki server can both write and retrieve data.

### 15.1 Prepare a Timestamp

The Loki write API requires a nanosecond timestamp.

To generate it:

    TS=$(date +%s%N)

To check it:

    echo $TS

### 15.2 Write a Test Log

Execute the following command:

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-test\",
              \"namespace\": \"app-demo\"",
              \"app\": \"loki-manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki from manual curl test\"]
            ]
          }
        ]
      }"

If there is no output, it usually means the request was successful.

### 15.3 Verify the Write Result

Execute the following command:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-test"}' \
      --data-urlencode 'limit=10' | jq

You should see:

    hello loki from manual curl test

### 15.4 Query Specific Content

Execute the following command:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-test"} |= "hello loki"' \
      --data-urlencode 'limit=10' | jq

### 15.5 If Nothing Is Found

Troubleshoot by checking:

- Whether /ready is set to ready.
- Whether the push request was successful.
- Whether the time range is correct.
- Whether the query label is accurate.
- Whether auth_enabled is set to false.
- Whether the request is being forwarded through the gateway to the correct path.
- Whether there are any write errors in Loki's logs.

To view Loki logs:

    kubectl logs <loki-pod-name> -n logging --tail=200 | grep -Ei "error|push|ingest|stream|label"

---

## Section Sixteen: Using the Loki API to Query Labels

### 16.1 Query Label Names

Execute the following command:

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

You may see:

    app
    job
    namespace

### 16.2 Query Job Label Values

Execute the following command:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/job/values" | jq

Expected output:

    manual-test

### 16.3 Query Namespace Label Values

Execute the following command:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/namespace/values" | jq

Expected output:

    app-demo

### 16.4 Meaning

If labels can be queried, it means that:

- There are log streams in Loki.
- Label indexing is available.
- Query_range can be used with label filters.

Tags for actual Pod logs, such as pod, container, and node, will only appear after Alloy is integrated in Chapter Six.

---

## Section Seventeen: Verifying the Gateway

### 17.1 View the Gateway Pod

Execute the following command:

    kubectl get pods -n logging | grep gateway

### 17.2 View the Gateway Service

Execute the following command:

    kubectl get svc -n logging | grep gateway

### 17.3 View Gateway Logs

Execute the following command:

    kubectl logs <loki-gateway-pod> -n logging --tail=100

### 17.4 Access Loki Through the Gateway

Use port forwarding```markdown
--namespace logging \
--version <CHART_VERSION> \
-f values-loki-monolithic.yaml

Common causes:

- The Chart version is incorrect.
- The values field does not match.
- The deploymentMode is set incorrectly.
- There are YAML indentation errors.
- The Kubernetes version does not meet the requirements.
- The CRD or API versions are incompatible.

### 19.2 Pod Pending

To check:

- `kubectl get pods -n logging -o wide`
- `kubectl describe pod <pod-name> -n logging`

Common causes:

- Insufficient CPU resources.
- Insufficient memory resources.
- PVC is pending.
- The StorageClass does not exist.
- The nodeSelector does not match.
- The taint/toleration settings do not match.

If the issue is with a Pending PVC:

- `kubectl get pvc -n logging`
- `kubectl describe pvc <pvc-name> -n logging`

Possible solutions:

- Check the default StorageClass.
- Configure an available StorageClass.
- Temporarily disable persistence for testing purposes.

### 19.3 Pod CrashLoopBackOff

To check:

- `kubectl describe pod <loki-pod-name> -n logging`
- `kubectl logs <loki-pod-name> -n logging --previous --tail=200`

Common causes:

- Incorrect Loki configuration.
- Incorrect storage_config settings.
- Incorrect schemaConfig settings.
- Lack of permissions on the file directory.
- Abnormal PVC mounting.
- Conflict between deploymentMode and replica configurations.
- Mismatch between replication_factor and the number of replicas.

Note:

- For a single replica, `commonConfig.replication_factor: 1` must be set.

### 19.4 /ready or not ready

To check:

- `kubectl get pods -n logging`
- `kubectl logs <loki-pod-name> -n logging --tail=200`
- `kubectl get endpoints -n logging`

Common causes:

- Loki is still starting up.
- The Ring has not become ready.
- Storage initialization failed.
- Configuration errors.
- Abnormal Gateway forwarding.

### 19.5 /metrics cannot be accessed

To check:

- Verify that port forwarding is running.
- Ensure the Service name is correct.
- Check if the Service port is correct.
- Determine whether it is forwarding to the Loki Service or the Gateway Service.
- Confirm that the Pod is in the Ready state.

Commands to check:

- `kubectl get svc -n logging`
- `kubectl describe svc <service-name> -n logging`

### 19.6 Manual push fails

Possible issues:

- 400 Bad Request
- 404 Not Found
- 429 Too Many Requests
- 500 Internal Server Error

Reasons for these errors:

- For 400:
  - JSON format error, incorrect timestamp, or invalid label.
- For 404:
  - Incorrect path; make sure it is `/loki/api/v1/push`.
- For 429:
  - Rate-limiting issues during writing.
- For 500:
 - Internal errors in Loki or storage-related problems.

To troubleshoot:

- `kubectl logs <loki-pod-name> -n logging --tail=200`

### 19.7 query_range returns no data

To check:

- Verify that the labels are correct.
- Ensure that the time range is within the valid period.
- Confirm that the logs have been successfully written.
- Check if the wrong gateway was used for querying.
- Verify if `auth_enabled` settings affect the tenant's access.
- Check for any syntax errors in the LogQL query.

First, check the labels:

```bash
curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq
```

Then, check the job-related logs:

```bash
curl -s "http://127.0.0.1:3100/loki/api/v1/label/job/values" | jq
```

---

## Section 20: Basics of Upgrade and Rollback

### 20.1 Check the current Release

- `helm list -n logging`
- `helm status loki -n logging`

### 20.2 View historical versions

- `helm history loki -n logging`

### 20.3 Upgrade after modifying values.yaml

- `helm upgrade loki grafana-community/loki \
  --namespace logging \
  --version <CHART_VERSION> \
  -f values-loki-monolithic.yaml`

### 20.4 Rollback

- `helm rollback loki <REVISION> -n logging`

### 20.5 Back up values.yaml before upgrading

- `helm get values loki -n logging -a > backup-lo[ ] The monolithic or singleBinary fields have been confirmed.
[ ] The gateway field has been confirmed.
[ ] The storage-related fields have been confirmed.

### Task 23.3: Create the values file

File:

    values-loki-monolithic.yaml

Acceptance:

[ ] auth_enabled has been disabled as per the learning environment requirements.
[ ] replication_factor has been set to 1.
[ ] deploymentMode has been set to monolithic mode.
[ ] Replicas for other modes have been set to 0.
[ ] The gateway has been enabled.
[ ] minio is not enabled yet.

### Task 23.4: Render and check

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml \
      > loki-monolithic-rendered.yaml

Acceptance:

[ ] The helm template executed without any errors.
[ ] The Loki StatefulSet and Deployment can be seen.
[ ] The Loki Service can be seen.
[ ] The Gateway Service can be seen.
[ ] The used image can be identified.

### Task 23.5: Install Loki

    kubectl create namespace logging

    helm install loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki-monolithic.yaml

Acceptance:

[ ] helm list -n logging shows that it has been deployed.
[ ] The Loki Pod is running.
[ ] The Loki Service is functioning normally.
[ ] The Endpoint is available.

### Task 23.6: Verify the Loki API

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

    curl -s http://127.0.0.1:3100/metrics | head

Acceptance:

[ ] "ready" is returned when accessing /ready.
[ ] Output is available when accessing /metrics.

### Task 23.7: Manually write and query data

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-test\",
              \"namespace\": \"app-demo\"",
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

Acceptance:

[ ] "hello loki from manual curl test" can be queried successfully.

---

## Chapter 24: Acceptance Checklist

After completing this document, the following should be confirmed:

[ ] The logging Namespace has been created.
[ ] The grafana-community Helm repository has been added.
[ ] The Loki Chart version has been determined.
[ ] values-loki-default.yaml has been exported.
[ ] values-loki-monolithic.yaml has been prepared.
[ ] The helm template check was successful.
[ ] The helm installation was completed successfully.
[ ] helm list shows that it has been deployed.
[ ] The Loki Pod is running.
[ ] The Loki Pod is ready.
[ ] The Loki Service exists.
[ ] The Gateway Service exists.
[ ] The Endpoint is available.
[ ] PVCs are bound.
[ ] "ready" is returned when accessing /ready.
[ ] /metrics is accessible.
[ ] Manual data push was successful.
[ ] Test logs can be queried using query_range.
[ ] The Loki Pod logs can be viewed.
[ ] It has been understood that the monolithic mode is only suitable for learning and small-scale verification.
[ ] It has been clarified that MinIO object storage will be integrated in Article 05.

---

## Chapter 25: Common Misconceptions

### 25.1 Misconception 1: If the Pod is running, Loki must be functioning normally.

Wrong.

The following also need to be verified:

/ready
/metrics
push
query_range
labels
Gateway
PVC
Loki logs

### 25.2 Misconception 2: Loki logs can be viewed immediately after deployment.

Wrong.

Loki is a server-side component.

To view Pod logs, additional components such as:

Grafana Alloy
Promtail
Flhttps://grafana.com/docs/loki/latest/setup/upgrade/upgrade-to-community/

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