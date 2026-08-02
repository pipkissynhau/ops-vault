# 14-Loki Common Troubleshooting: Collection-Write-Query-Storage-Alert

## Document Notes

This is the fourteenth article in the Loki specialized learning series, used to systematically organize common troubleshooting methods for Loki in Kubernetes production and experimental environments.

Previously completed:

    01-Loki Basic Understanding and Experimental Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experimental Selection
    04-Loki Single-Instance Helm Deployment Practical
    05-Loki Object Storage Access MinIO Practical
    06-Grafana-Alloy Collection K8S-Pod Logs Practical
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practical: Namespace-Pod-Container Log Retrieval
    09-LogQL Advanced Query Practical: json-logfmt-regexp-unwrap
    10-Grafana Integration with Loki and Log Dashboard Practical
    11-Loki Log Alert Practical: Ruler and AlertManager Coordination
    12-Loki Production Governance Practical: Log Volume-Retention Period-Limiting-Security
    13-Loki Performance and High Availability Practical: Simple-Scalable Mode Introduction

This article focuses on troubleshooting and does not expand on theoretical content.

Troubleshooting scope covers:

    Alloy Cannot Collect Pod Logs
    Alloy Has Logs but Loki Cannot Find Them
    Loki Write Failure
    Loki Gateway 502 / 503
    query_range Query Returns Empty
    query_range Query Timeout
    Grafana Explore Query Returns Empty
    Grafana Dashboard Variable Empty
    MinIO / S3 Object Storage Abnormality
    retention Not Effective
    Ruler Alert Not Triggered
    AlertManager Does Not Receive Loki Alerts
    High Cardinality Causing Write/Query Abnormality
    429 Rate Limiting
    Log Timestamp Abnormality
    JSON / logfmt / regexp Parsing Abnormality
    Pod Running But No Business Logs
    Loki Pod CrashLoopBackOff
    Loki Self-Monitoring and Emergency Response

The goal of this article is to form a set of implementable Loki troubleshooting Runbook.

---

## Tags

#Loki #Grafana #GrafanaAlloy #LogQL #Kubernetes #LogDetachment #Runbook #MinIO #AlertManager #Ruler #SRE #Observation #FaultManagement #ProductionBarriers

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/14-Loki Common Troubleshooting Practical: Collection-Write-Query-Storage-Alert.md

---

## One, Troubleshooting Goals

After completing this article, you should be able to:

    1. Establish a comprehensive Loki troubleshooting approach.
    2. Distinguish between collection issues, write issues, query issues, storage issues, and alert issues.
    3. Master the troubleshooting path for Alloy not collecting logs.
    4. Master the troubleshooting path for Loki push write failure.
    5. Master the troubleshooting path for Gateway 502 / 503.
    6. Master the troubleshooting path for query_range returning empty.
    7. Master the troubleshooting path for Grafana Explore / Dashboard query anomalies.
    8. Master the troubleshooting methods for MinIO / S3 storage anomalies.
    9. Master the troubleshooting methods for retention not taking effect.
    10. Master the troubleshooting methods for Ruler alerts not triggering.
    11. Master the troubleshooting methods for AlertManager not receiving Loki alerts.
    12. Master the troubleshooting methods for 429 rate limiting and high cardinality issues.
    13. Master the troubleshooting methods for Loki Pod CrashLoopBackOff.
    14. Be able to compile a Loki production fault handling Runbook.
    15. Be able to form a on-call troubleshooting checklist.

---

## Two, Experimental and Production Environment Assumptions

### 2.1 Kubernetes Cluster

Experimental nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespaces:

    logging
    logging-ssd
    monitoring
    minio
    app-demo

Components:

    Loki
    Loki Gateway
    Grafana Alloy
    Grafana
    MinIO
    AlertManager
    nginx-demo
    json-log-demo
    logfmt-log-demo

### 2.2 Key Access Addresses

Single-instance Loki Gateway:

    loki-gateway.logging.svc.cluster.local

Simple Scalable Loki Gateway:

    loki-ssd-gateway.logging-ssd.svc.cluster.local

MinIO:

    minio.minio.svc.cluster.local:9000

Grafana:

    grafana.monitoring.svc.cluster.local

AlertManager:

    alertmanager.monitoring.svc.cluster.local:9093

### 2.3 Common Local Ports

Single-instance Loki:

    3100

Simple Scalable Loki:

    3101

Grafana:

    3000

AlertManager:

    9093

MinIO Console:

    9001

---

## Three, Loki Troubleshooting General Principles

### 3.1 Prioritize Layered Troubleshooting, Avoid Random Checks

The Loki chain can be divided into five layers:

    Application Layer:
        Whether the Pod is actually outputting logs.

    Collection Layer:
        Whether Alloy has collected logs.

    Write Layer:
        Whether Alloy successfully sent logs to Loki.

    Storage Layer:
        Whether Loki successfully wrote to memory, WAL, and object storage.

    Query Layer:
        Whether Grafana / curl / LogQL can retrieve logs.

    Alert Layer:
        Whether Ruler executed rules, and whether AlertManager received alerts.

Troubleshooting must first determine which layer the issue is in.

Do not immediately suspect Loki itself.

### 3.2 Recommended Troubleshooting Order

Recommended order: /think

1. Can `kubectl logs` view application logs?  
2. Is the Alloy Pod running?  
3. Does Alloy cover the node where the application Pod resides?  
4. Are there errors in the Alloy logs?  
5. Is the Loki Gateway /ready ready?  
6. Do Loki labels have namespace / pod / app?  
7. Can Loki query_range retrieve logs?  
8. Can Grafana Explore retrieve logs?  
9. Is MinIO functioning normally?  
10. Are there error / warn logs in Loki's own logs?  
11. Further investigation is needed for limits / retention / ruler.  

### 3.3 Do Not Skip Application Log Verification  

The first step is always:  

    kubectl logs  

If `kubectl logs` shows no logs, it's normal for Loki to not find them.  

Common reasons:  

    The application does not output stdout / stderr.  
    Requests are not reaching this Pod.  
    The wrong Namespace is being checked.  
    The wrong Pod is being checked.  
    The wrong Container is being checked.  
    The container has restarted; use `--previous`.  
    Logs are written to a file inside the container instead of stdout / stderr.  

### 3.4 Do Not Use Dashboard to Determine Root Causes  

An empty Grafana Dashboard does not mean Loki has no data.  

Possible reasons:  

    The time range is incorrect.  
    Variables are empty.  
    The app label does not exist.  
    LogQL is written incorrectly.  
    Dashboard variables use `=` instead of `=~`.  
    The wrong data source is selected.  
    There are no logs in the current time window.  

When troubleshooting, first execute the simplest query using `curl` or Grafana Explore:  

    {namespace="app-demo"}  

---  

## Four. Common Troubleshooting Command Quick Reference  

### 4.1 View Loki-Related Resources  

    kubectl get pods -n logging -o wide  

    kubectl get svc -n logging  

    kubectl get endpoints -n logging  

    kubectl get endpointslice -n logging  

    kubectl get pvc -n logging  

    helm list -n logging  

    helm status loki -n logging  

    helm get values loki -n logging -a  

### 4.2 View Alloy  

    kubectl get ds -n logging  

    kubectl get pods -n logging -o wide | grep alloy  

    kubectl logs <alloy-pod-name> -n logging --tail=300  

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|loki|push|forbidden|denied|timeout|429"  

### 4.3 View Loki  

    kubectl logs <loki-pod-name> -n logging --tail=300  

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|warn|failed|panic|storage|s3|minio|ring|ruler|compactor|query|push|429"  

### 4.4 View Gateway  

    kubectl logs <loki-gateway-pod> -n logging --tail=300  

    kubectl logs <loki-gateway-pod> -n logging --tail=500 | grep -Ei "502|503|500|push|query|ready|upstream"  

### 4.5 Basic Loki API Checks  

Port forwarding:  

    kubectl port-forward svc/loki-gateway 3100:80 -n logging  

Check ready:  

    curl -s http://127.0.0.1:3100/ready  

Check metrics:  

    curl -s http://127.0.0.1:3100/metrics | head  

Check labels:  

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq  

Check namespace values:  

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq  

Check app values:  

    curl -s http://127.0.0.1:3100/loki/api/v1/label/app/values | jq  

### 4.6 Basic query_range Query  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq  

Query the last 1 hour:  

    END=$(date +%s%N)  
    START=$(date -d "1 hour ago" +%s%N)  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq  

### 4.7 Manual push Test  

    TS=$(date +%s%N) /think

curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-troubleshooting-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki troubleshooting test\"]
            ]
          }
        ]
      }"

Query:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-troubleshooting-test"}' \
      --data-urlencode 'limit=10' | jq

---

## Five. Fault One: kubectl logs has logs, but Loki cannot find them

### 5.1 Phenomenon

Application Pod can see logs:

    kubectl logs <pod-name> -n app-demo --tail=50

But Loki query has no results:

    {namespace="app-demo"}

### 5.2 Troubleshooting Path

Complete path:

    Application has logs
      ↓
    Alloy is running
      ↓
    Alloy is on the Pod's node
      ↓
    Alloy has permission to read pods/log
      ↓
    Alloy successfully sends to Loki
      ↓
    Loki received label
      ↓
    Query time range and labels are correct

### 5.3 Confirm Pod's Node

    kubectl get pod -n app-demo -o wide

Record:

    Pod:
        <pod-name>

    Node:
        <node-name>

### 5.4 Confirm Alloy is covering the node

    kubectl get pod -n logging -o wide | grep alloy

Confirm if there is an Alloy Pod running on the same node.

If not:

    The node's logs may not be collected.

Common causes:

    Alloy DaemonSet not scheduled to the node.
    Node has taint.
    Alloy has no toleration.
    nodeSelector restrictions.
    Resource insufficiency.
    Image pull failure.
    PodSecurity restrictions.

### 5.5 Check Alloy DaemonSet

    kubectl get ds -n logging

    kubectl describe ds alloy -n logging

If the DaemonSet name is not "alloy":

    kubectl get ds -n logging

Then replace the name.

Focus on:

    Desired Number of Nodes Scheduled
    Current Number of Nodes Scheduled
    Number of Nodes Scheduled with Up-to-date Pods
    Number of Available Pods
    Events

### 5.6 Check Alloy RBAC

Confirm ServiceAccount:

    kubectl get sa -n logging | grep alloy

Check permissions:

    kubectl auth can-i list pods \
      --as=system:serviceaccount:logging:alloy

    kubectl auth can-i watch pods \
      --as=system:serviceaccount:logging:alloy

    kubectl auth can-i get pods/log \
      --as=system:serviceaccount:logging:alloy

If returns "no", RBAC is insufficient.

Resolution:

    Check Helm values for rbac.create being true.
    Check ClusterRole / ClusterRoleBinding existence.
    Confirm ServiceAccount name is correct.

### 5.7 Check Alloy Logs

    kubectl logs <alloy-pod-name> -n logging --tail=300

Filter:

    kubectl logs <alloy-pod-name> -n logging --tail=500 | grep -Ei "error|warn|forbidden|denied|pods/log|kubernetes|loki|push|failed"

Common errors:

    forbidden
    cannot list resource pods
    cannot get resource pods/log
    failed to send batch
    connection refused
    no such host
    context deadline exceeded
    429 Too Many Requests
    500 Internal Server Error

### 5.8 Check Alloy Write Address

Check Alloy ConfigMap:

    kubectl get cm -n logging | grep alloy

    kubectl get cm <alloy-configmap-name> -n logging -o yaml | grep -n "loki.write" -A 20

Confirm write address:

    http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push

Or SSD:

    http://loki-ssd-gateway.logging-ssd.svc.cluster.local/loki/api/v1/push

Common issues: /think

Namespace is written incorrectly.  
Service name is written incorrectly.  
Missing /loki/api/v1/push.  
Wrote 3100 but Gateway Service is 80.  
Grafana queries one Loki, Alloy writes to another Loki.

### 5.9 Test Loki Gateway from Alloy's Namespace

    kubectl run curl-loki-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -s http://loki-gateway.logging.svc.cluster.local/ready

    curl -s http://loki-gateway.logging.svc.cluster.local/loki/api/v1/labels

Exit:

    exit

If not working:

    Troubleshoot Loki Gateway, Service, DNS, NetworkPolicy.

### 5.10 Check Loki labels

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

If missing:

    namespace
    pod
    container
    app

Indicates Alloy may not have written successfully, or written labels not matching expectations.

Continue checking:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

If no app-demo:

    Alloy did not collect app-demo.
    app-demo was relabeled and dropped.
    Query time range is incorrect.
    Alloy wrote to another Loki.

---

## Six, Fault Two: Loki can manually push, but Alloy cannot collect logs

### 6.1 Phenomenon

Manual push succeeds:

    {job="manual-troubleshooting-test"}

Can be queried.

But real Pod logs cannot be found:

    {namespace="app-demo"}

### 6.2 Conclusion Direction

This indicates:

    Loki server is likely normal.
    Gateway is likely normal.
    Write API is likely normal.

The problem is more likely in:

    Alloy collection configuration
    Alloy RBAC
    Alloy scheduling overlap
    Alloy relabel/drop rules
    Application log output method

### 6.3 Check if Alloy filtered namespace

Check Alloy configuration:

    kubectl get cm <alloy-configmap-name> -n logging -o yaml

Search for:

    keep
    drop
    namespace
    app-demo

If exists:

    action = "keep"
    regex = "xxx"

Confirm if regex contains app-demo.

If exists:

    action = "drop"
    regex = "app-demo"

Indicates app-demo is excluded.

### 6.4 Check if only collecting current node

If Alloy uses DaemonSet, usually need to limit to current node.

Check if configuration has:

    field = "spec.nodeName=" + coalesce(sys.env("HOSTNAME"), constants.hostname)

If configuration is wrong, may cause:

    No Pod collected.
    Each Alloy duplicates full Pod collection.
    Only collected wrong node.

### 6.5 Check log collection method

Alloy collects Pod logs in two ways:

    loki.source.kubernetes:
        Tails Pod logs via Kubernetes API.

    loki.source.file:
        Reads /var/log/containers or /var/log/pods.

If using loki.source.kubernetes:

    Focus on RBAC, Kubernetes API, pods/log permissions.

If using loki.source.file:

    Focus on hostPath mounting, path, file permissions, __path__ relabel.

### 6.6 Check container log path

If using file collection, common paths on nodes:

    /var/log/containers/*.log
    /var/log/pods/*/*/*.log

Check on node:

    ls -lh /var/log/containers | head

    ls -lh /var/log/pods | head

If Alloy container does not mount these paths, file collection won't work.

---

## Seven, Fault Three: Loki push returns 400 Bad Request

### 7.1 Phenomenon

Manual push or Alloy writing causes:

    400 Bad Request

### 7.2 Common Causes

    JSON format error
    values structure error
    timestamp is not string
    timestamp is not nanoseconds timestamp
    timestamp is too old
    timestamp is too new
    label name is invalid
    label value is too long
    single line log is too big
    request body format does not match Loki push API

### 7.3 Check push request body format

Correct structure:

    {
      "streams": [
        {
          "stream": {
            "job": "manual-test"
          },
          "values": [
            ["1710000000000000000", "hello loki"]
          ]
        }
      ]
    }

Note:

    timestamp must be string.
    values is a 2D array.
    Each values entry's first element is timestamp.
    Second element is log content.
    stream is label object.

### 7.4 Verify timestamp

Generate nanoseconds timestamp:

    date +%s%N

Check:

    TS=$(date +%s%N)

    echo $TS

If using seconds timestamp:

    date +%s

May cause timestamp mismatch.

### 7.5 Check Loki logs

kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "bad request|invalid|timestamp|label|line|sample|discard"

### 7.6 Handling

    1. Fix the JSON.
    2. Generate a timestamp using date +%s%N.
    3. Ensure the timestamp is enclosed in double quotes.
    4. Confirm the label names are valid.
    5. Reduce the size of single-line logs.
    6. Check reject_old_samples_max_age.
    7. Check max_line_size.

---

## VIII. Fault Four: Loki push Returns 404 Not Found

### 8.1 Symptoms

    404 Not Found

### 8.2 Common Causes

Incorrect write path.

Incorrect paths:

    /api/v1/push
    /push
    /loki/push

Correct path:

    /loki/api/v1/push

### 8.3 Check Alloy loki.write

Error:

    url = "http://loki-gateway.logging.svc.cluster.local"

Correct:

    url = "http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push"

### 8.4 Check Gateway

In some cases, if you bypass the Gateway to access Loki Service, the path and port must be confirmed.

Recommended priority:

    Alloy → Loki Gateway → Loki

### 8.5 Handling

    1. Modify the Alloy write URL.
    2. Check with helm template.
    3. Upgrade Alloy with helm.
    4. Check if Alloy logs still show 404.
    5. Manually push with curl to verify.

---

## IX. Fault Five: Loki push Returns 429 Too Many Requests

### 9.1 Symptoms

Alloy logs:

    failed to send batch
    429 Too Many Requests

Loki logs:

    ingestion rate limit exceeded
    per stream rate limit exceeded
    maximum active stream limit exceeded

### 9.2 Meaning of 429

429 indicates that Loki rejected the write request.

Common causes:

    Write rate exceeds the limit.
    Writing too fast per stream.
    Too many active streams.
    High cardinality labels causing stream explosion.
    Application abnormally logs.
    Duplicate collection.
    limits_config set too low.

### 9.3 Do Not Directly Increase Limits

Incorrect handling:

    Seeing 429, increase all limits.

Correct handling:

    First identify who is logging.

### 9.4 Find the Namespace with the Most Logs

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 9.5 Find the App with the Most Logs

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 9.6 Find the Pod with the Most Logs

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 9.7 Check for High Cardinality Labels

Check labels:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Watch out for:

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message

Check the number of values for a label:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/trace_id/values | jq '.data | length'

### 9.8 Check for Duplicate Collection

Symptoms:

    The same log appears multiple times in Loki.
    Deployed Alloy + Promtail + Fluent Bit collecting simultaneously.
    DaemonSet collects all pods in the cluster.
    Alloy field selector configured incorrectly.

Check:

    kubectl get pods -A | grep -Ei "alloy|promtail|fluent|filebeat"

### 9.9 Check Loki Discarded Metrics

If /metrics is available:

    curl -s http://127.0.0.1:3100/metrics | grep -E "loki_discarded_samples_total|loki_discarded_bytes_total"

These metrics help determine the reason for discard.

### 9.10 Handling Approach

Handling order:

    1. Identify the application logging excessively.
    2. Lower the application's log level.
    3. Filter healthz / metrics / debug logs.
    4. Fix high cardinality labels.
    5. Eliminate duplicate collection.
    6. Reasonably adjust limits_config.
    7. Expand write / ingester if needed.
    8. Continuously monitor if 429 disappears.

---

## X. Fault Six: Loki push Returns 500 Internal Server Error

### 10.1 Symptoms

    500 Internal Server Error

### 10.2 Common Causes

    Loki internal component failure
    write / ingester failure
    ring is unhealthy
    MinIO / S3 unavailable
    bucket does not exist
    insufficient storage permissions
    WAL / PVC failure
    configuration error
    replication_factor does not match replica count
    backend service Endpoint is empty

### 10.3 Troubleshoot Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|panic|failed|s3|minio|bucket|ring|ingester|flush|wal|permission"

### 10.4 Troubleshoot Gateway Logs

    kubectl logs <loki-gateway-pod> -n logging --tail=300 | grep -Ei "500|502|503|push|upstream"

### 10.5 Troubleshoot MinIO

    kubectl get pod -n minio -o wide

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

    kubectl logs <minio-pod-name> -n minio --tail=200

### 10.6 Test MinIO from Loki Namespace

    kubectl run curl-minio-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -I http://minio.minio.svc.cluster.local:9000

Exit:

    exit

### 10.7 Handling

    1. Restore Loki Pod.
    2. Restore MinIO.
    3. Check if bucket exists.
    4. Check access key / secret key.
    5. Check s3ForcePathStyle.
    6. Check insecure / HTTP / HTTPS configuration.
    7. Check replication_factor.
    8. Check PVC / WAL.
    9. Helm rollback if necessary.

---

## ElevenI don't know.Fault Seven: Gateway 502 / 503

### 11.1 Symptoms

Grafana or curl access:

    502 Bad Gateway
    503 Service Unavailable

May appear at:

    /ready
    /loki/api/v1/push
    /loki/api/v1/query_range
    /loki/api/v1/labels

### 11.2 Common Causes

    Gateway backend Loki Service is unavailable.
    Loki Pod is not Ready.
    Service Endpoint is empty.
    Gateway upstream configuration is incorrect.
    Some component of read/write/backend is unavailable.
    NetworkPolicy blocks traffic.
    Gateway Pod itself has issues.
    Service name mismatch after deployment mode change.

### 11.3 Check Gateway Pod

    kubectl get pod -n logging -o wide | grep gateway

    kubectl logs <loki-gateway-pod> -n logging --tail=300

### 11.4 Check Service and Endpoint

    kubectl get svc -n logging

    kubectl get endpoints -n logging

    kubectl get endpointslice -n logging

Focus on confirming:

    loki-gateway Endpoint is not empty.
    loki Service Endpoint is not empty.
    If it's SSD, read/write/backend Endpoint is not empty.

### 11.5 Check Backend Pod Ready

    kubectl get pods -n logging -o wide

If it's SSD:

    kubectl get pods -n logging-ssd -o wide

Focus on:

    read
    write
    backend
    gateway

### 11.6 Handling

    1. Restore backend Pod Ready.
    2. Check Service selector.
    3. Check Endpoint.
    4. Check Gateway configuration.
    5. Check NetworkPolicy.
    6. If upgraded recently, consider helm rollback.

---

## TwelveI don't know.Fault Eight: query_range Query Returns Empty

### 12.1 Symptoms

Query returns:

    result: []

Or no logs in Grafana Explore.

### 12.2 Common Causes

    Query time range is incorrect
    Label name is wrong
    Label value is wrong
    App label does not exist
    Pod name has changed
    Alloy did not collect logs
    Logs are written to another Loki
    Logs are dropped by relabel
    Logs have passed retention period
    Data has not been flushed yet but recent data is also not queryable
    Tenant header is incorrect

### 12.3 Check Labels First

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

### 12.4 Check Label Values

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

    curl -s http://127.0.0.1:3100/loki/api/v1/label/app/values | jq

    curl -s http://127.0.0.1:3100/loki/api/v1/label/pod/values | jq

### 12.5 Query from Broad to Narrow

First check:

    {namespace="app-demo"}

Then check:

    {namespace="app-demo", app="nginx-demo"}

Then check:

    {namespace="app-demo", pod=~"nginx-demo-.*"}

Do not start with complex queries.

### 12.6 Specify Query Time Range

    END=$(date +%s%N)
    START=$(date -d "1 hour ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq

### 12.7 Check if Querying the Wrong Loki

If the environment has both:

    Loki
    Loki-SSD

Confirm: /think

Which Loki does Alloy write to?  
Which Loki does the Grafana Data Source query?  
Which Loki does the curl port forwarding use?

### 12.8 Check if the app label exists

If querying:

    {namespace="app-demo", app="nginx-demo"}

returns empty, but:

    {namespace="app-demo"}

has results, it may be that the app label does not exist or the label name is different.

Check the stream:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=5' | jq '.data.result[].stream'

---

## Thirteen, Fault Nine: query_range query timeout or very slow

### 13.1 Phenomenon

Grafana Explore reports:

    timeout  
    query timeout  
    context deadline exceeded  
    maximum query length exceeded  
    too many outstanding requests  
    query exceeded limits

### 13.2 Common Causes

    Query time range is too large  
    Query lacks namespace/app restrictions  
    Using global regular expressions  
    Using global JSON / regexp parsing  
    High cardinality labels  
    Querying too many series  
    Too many dashboard panels  
    Grafana All variable expansion is too large  
    Object storage read is slow  
    Read / querier resources are insufficient  
    limits_config restrictions are triggered

### 13.3 Example of Bad Queries

Not recommended:

    {namespace=~".+"} |~ "(?i)error"

Not recommended:

    {pod=~".+"} | json | level="error"

Not recommended:

    {namespace=~".+"} | regexp `Complicated rules`

### 13.4 Recommended Query Methods

Recommended:

    {namespace="app-prod", app="order-api"} |~ "(?i)error|exception|panic"

Recommended JSON:

    {namespace="app-prod", app="order-api"} | json | __error__="" | level="error"

### 13.5 Check Loki read / query logs

Single instance:

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "query|timeout|limit|series|context|error"

SSD:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=read --tail=500 | grep -Ei "query|timeout|limit|series|context|error"

### 13.6 Handling

    1. Narrow the time range.  
    2. Add namespace/app conditions.  
    3. Avoid All variables.  
    4. Avoid global JSON / regexp.  
    5. Optimize dashboard panel count.  
    6. Adjust max_query_length / query_timeout limits.  
    7. Scale up read / querier.  
    8. Check object storage performance.  
    9. Govern high cardinality labels.

---

## Fourteen, Fault Ten: Grafana Data Source Save & Test Failure

### 14.1 Phenomenon

Failure when adding a Loki Data Source in Grafana.

Possible error messages:

    Unable to connect with Loki  
    Bad Gateway  
    Not found  
    Data source connected, but no labels found  
    timeout

### 14.2 Check URL

When Grafana accesses Loki within a Kubernetes cluster, use the Service DNS.

Single instance:

    http://loki-gateway.logging.svc.cluster.local

SSD:

    http://loki-ssd-gateway.logging-ssd.svc.cluster.local

Do not use:

    http://127.0.0.1:3100

Unless Grafana is also running in the same local environment.

### 14.3 Test from Grafana Pod

    kubectl get pod -n monitoring | grep grafana

    kubectl exec -it <grafana-pod-name> -n monitoring -- sh

Inside the container:

    wget -qO- http://loki-gateway.logging.svc.cluster.local/ready

If wget is not available:

    Use a temporary Pod with curl.

### 14.4 Test with a Temporary Pod

    kubectl run curl-grafana-loki-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n monitoring \
      -- sh

Inside the container:

    curl -s http://loki-gateway.logging.svc.cluster.local/ready

Exit:

    exit

### 14.5 Handling

    1. Fix the Data Source URL.  
    2. Confirm the Loki Gateway Service exists.  
    3. Confirm the Endpoint is not empty.  
    4. Confirm NetworkPolicy is not blocking.  
    5. Confirm Loki /ready is normal.  
    6. If auth_enabled is enabled, configure tenant / header / authentication.

---

## Fifteen, Fault Eleven: Grafana Dashboard Variables Are Empty

### 15.1 Phenomenon

Variables:

    namespace  
    app  
    pod  
    container

Display as empty.

### 15.2 Common Causes

    Wrong Loki data source selected  
    No data in the time range  
    label_values query is incorrect  
    Dependent variables are empty  
    cluster variable has no matching value  
    app label does not exist  
    Current user lacks data source permissions  
    Loki does not have the label

### 15.3 Check Loki label

curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

If no app:

    The Dashboard app variable is naturally empty.

### 15.4 Variable Query Check

namespace:

    label_values(namespace)

app:

    label_values({namespace="$namespace"}, app)

pod:

    label_values({namespace="$namespace", app="$app"}, pod)

If multi-select is enabled:

    label_values({namespace=~"$namespace"}, app)

### 15.5 Common Variable Errors

Error:

    {namespace="$namespace"}

May not match when namespace is multi-select or All.

Recommended:

    {namespace=~"$namespace"}

### 15.6 Handling

    1. Confirm the data source.
    2. Expand the Dashboard time range to Last 1 hour.
    3. First use label_values(namespace).
    4. Then add namespace/app dependencies step by step.
    5. Check if Multi-value requires =~.
    6. Verify if the app label actually exists.

---

## SixteenI don't know.Fault Twelve: MinIO / S3 Object Storage Abnormality

### 16.1 Phenomenon

Loki logs show:

    failed to create object client
    NoSuchBucket
    AccessDenied
    SignatureDoesNotMatch
    connection refused
    timeout
    failed to flush chunks
    failed to put object
    failed to get object
    slow query from object storage

### 16.2 Check MinIO Pod

    kubectl get pods -n minio -o wide

    kubectl logs <minio-pod-name> -n minio --tail=300

### 16.3 Check MinIO Service / Endpoint

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

### 16.4 Test MinIO from Loki Namespace

    kubectl run curl-minio-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -I http://minio.minio.svc.cluster.local:9000

Exit:

    exit

### 16.5 Use mc to Check Bucket

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

    mc ls local

    mc ls local/loki-chunks

    mc ls local/loki-ruler

    mc ls local/loki-admin

Exit:

    exit

### 16.6 Check Loki Storage Configuration

View values:

    helm get values loki -n logging -a | grep -n "storage" -A 80

Check:

    Whether endpoint is correct
    Whether bucketNames is correct
    Whether accessKeyId is correct
    Whether secretAccessKey is correct
    Whether s3ForcePathStyle is true
    Whether insecure matches HTTP / HTTPS
    Whether region is configured
    Whether bucket exists

### 16.7 Common Configuration Errors

Error 1: Endpoint includes a protocol but the current field does not require it.

May be written as:

    endpoint: http://minio.minio.svc.cluster.local:9000

Some configurations require:

    endpoint: minio.minio.svc.cluster.local:9000

Specifically follow the current Chart and Loki configuration requirements.

Error 2: HTTP MinIO but insecure is not enabled.

Error 3: MinIO requires path-style but s3ForcePathStyle is not enabled.

Error 4: Bucket name is written incorrectly.

Error 5: Using root user for testing works, but production environment switches to a dedicated user and lacks permissions.

### 16.8 Handling

    1. Restore MinIO Pod.
    2. Restore Service / Endpoint.
    3. Confirm bucket exists.
    4. Confirm keys are correct.
    5. Confirm path-style.
    6. Confirm HTTP / HTTPS.
    7. Check if Loki logs have recovered.
    8. Re-validate by re-pushing/querying.

---

## SeventeenI don't know.Fault Thirteen: Retention Not Taking Effect

### 17.1 Phenomenon

Retention period is configured, but objects in MinIO are never deleted.

### 17.2 Common Causes

    compactor not enabled
    retention_enabled not enabled
    retention_period not configured
    retention_stream not matched
    delete_request_store configuration error
    Object storage lacks delete permissions
    retention_delete_delay not reached
    compactor cycle not executed
    Current data still within retention period
    MinIO lifecycle conflicts with Loki retention
    Querying unexpired data

### 17.3 View Values

    helm get values loki -n logging -a | grep -n "retention" -A 80

    helm get values loki -n logging -a | grep -n "compactor" -A 80

### 17.4 View Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=800 | grep -Ei "compactor|retention|delete|marker|bucket|permission|error|warn"

### 17.5 View MinIO Objects

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

    mc find local/loki-chunks | head

    mc du local/loki-chunks

Exit:

    exit

### 17.6 Check Configuration Direction

Example configuration:

    loki:
      limits_config:
        retention_period: 168h

      compactor:
        working_directory: /var/loki/compactor
        compaction_interval: 10m
        retention_enabled: true
        retention_delete_delay: 2h
        delete_request_store: s3

Note:

    Specific fields are based on the current Loki and Helm Chart version.

### 17.7 Resolution

    1. Enable compactor.
    2. Enable retention_enabled.
    3. Set retention_period.
    4. Confirm delete_request_store.
    5. Confirm object storage deletion permissions.
    6. Wait for compactor cycle execution.
    7. Do not let MinIO lifecycle delete objects before Loki retention.

---

## Eighteen, Fault Fourteen: Ruler API Not Available

### 18.1 Symptoms

Access:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules

Returns:

    404
    ruler API disabled
    route not found
    empty response

### 18.2 Common Causes

    ruler.enable_api not enabled
    Ruler not enabled
    Gateway not forwarding Ruler API
    Using the wrong Service
    Ruler in backend mode but backend is abnormal
    Helm values fields not taking effect

### 18.3 Check Values

    helm get values loki -n logging -a | grep -n "ruler" -A 80

Confirm:

    enable_api: true

### 18.4 Check Loki Logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "ruler|rule|alertmanager|error|warn"

SSD:

    kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=500 | grep -Ei "ruler|rule|alertmanager|error|warn"

### 18.5 Resolution

    1. Enable ruler.enable_api.
    2. Configure Ruler storage.
    3. Configure alertmanager_url.
    4. Check with helm template.
    5. Perform helm upgrade.
    6. Re-access rules API after Loki is Ready.

---

## Nineteen, Fault Fifteen: Loki Alert Rule Upload Failure

### 19.1 Symptoms

Upload rules:

    curl -X POST --data-binary @rules.yaml ...

Returns:

    YAML parse error
    rule parse error
    invalid expression
    bad request
    namespace error

### 19.2 Common Causes

    YAML indentation error
    missing groups field
    missing rules field
    alert name error
    expr LogQL error
    expr returns log streams instead of numeric values
    for format error
    annotation template error
    incorrect Content-Type

### 19.3 First Validate LogQL

Do not write rules directly.

First validate the expression in Grafana Explore:

    sum by (namespace, app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

Confirm it returns numeric values before placing it in a rule.

### 19.4 Check Rule Format

Basic structure: /think

```yaml
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
        annotations:
          summary: "app-demo Too Many Error Logs"
          description: "namespace={{ $labels.namespace }}, app={{ $labels.app }} Too Many Error Logs"
```

### 19.5 Handling

1. Fix YAML indentation.
2. Validate expr in Explore first.
3. Confirm expr returns numerical values.
4. Use Content-Type: application/yaml.
5. Re-upload.
6. Check /loki/api/v1/rules.

---

## Twenty, Fault Sixteen: Ruler Alert Not Triggering

### 20.1 Phenomenon

Rules have been uploaded, but AlertManager has no alerts.

### 20.2 Common Causes

- LogQL itself has no results
- expr returns value not exceeding threshold
- for duration not met
- Ruler not loading rules
- Ruler cannot query Loki
- Ruler cannot connect to AlertManager
- AlertManager received but silenced
- AlertManager route not matching receiver
- Not enough logs in time window
- Query label written incorrectly

### 20.3 Manually Execute expr

Extract the expr from the rule and execute it.

Example:

```
sum by (namespace, app) (
  count_over_time(
    {namespace="app-demo"}
      |~ "(?i)error|exception|panic|failed" [5m]
  )
)
```

If returns 0 or no results, alert will not trigger.

### 20.4 Check Rule Loading

```
curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq
```

Specify namespace:

```
curl -s http://127.0.0.1:3100/loki/api/v1/rules/app-demo | jq
```

### 20.5 Check Ruler Logs

```
kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "ruler|rule|eval|alert|error|warn"
```

SSD:

```
kubectl logs -n logging-ssd -l app.kubernetes.io/component=backend --tail=500 | grep -Ei "ruler|rule|eval|alert|error|warn"
```

### 20.6 Check AlertManager Connection

Test from Loki Namespace:

```
kubectl run curl-alertmanager-test \
  --rm -it \
  --image=curlimages/curl:8.5.0 \
  -n logging \
  -- sh
```

Inside container:

```
curl -s http://alertmanager.monitoring.svc.cluster.local:9093/-/ready
```

Exit:

```
exit
```

### 20.7 Check AlertManager UI

Port forward:

```
kubectl port-forward svc/alertmanager 9093:9093 -n monitoring
```

Access:

```
http://127.0.0.1:9093
```

Check:

- Alerts
- Silences
- Status
- Receivers

### 20.8 Handling

1. Confirm expr has numerical values first.
2. Lower experimental threshold.
3. Shorten for duration.
4. Confirm Ruler has loaded rules.
5. Confirm alertmanager_url is correct.
6. Check AlertManager route/silence.
7. Check Ruler and AlertManager logs.

---

## Twenty-one, Fault Seventeen: AlertManager Not Receiving Loki Alerts

### 21.1 Phenomenon

Ruler rules look firing, but AlertManager UI has none.

### 21.2 Common Causes

- alertmanager_url is incorrect
- AlertManager Service name is incorrect
- AlertManager port is incorrect
- NetworkPolicy blocking
- Ruler sending failed
- AlertManager is not the current instance
- Alert is silenced
- AlertManager route configuration abnormal

### 21.3 Check Loki Ruler Configuration

```
helm get values loki -n logging -a | grep -n "alertmanager" -A 20
```

Confirm URL:

```
http://alertmanager.monitoring.svc.cluster.local:9093
```

### 21.4 Check Ruler Logs

```
kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "alertmanager|notify|send|alert|error|warn"
```

### 21.5 Check AlertManager Logs

```
kubectl logs <alertmanager-pod-name> -n monitoring --tail=500 | grep -Ei "notify|webhook|dispatch|error|warn|received"
```

### 21.6 Handling

1. Fix alertmanager_url.
2. Confirm AlertManager /-/ready.
3. Check NetworkPolicy.
4. Check AlertManager route.
5. Check silence.
6. Use webhook demo to verify if notifications are sent.

---

## Twenty-two, Fault Eighteen: JSON / logfmt / regexp Query Abnormality

### 22.1 Phenomenon

Query:

    | json
    | logfmt
    | regexp
    | unwrap

Returns no results, or the metrics panel throws an error.

### 22.2 Common Causes

    Logs are not JSON
    Logs are not logfmt
    regexp and log format mismatch
    Field name is incorrect
    Field type is not numeric
    unwrap field does not exist
    __error__ is unhandled
    Mixed log formats in app

### 22.3 View Parsing Errors

JSON:

    {namespace="app-demo", app="json-log-demo"} | json | __error__!=""

logfmt:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | __error__!=""

regexp:

    {namespace="app-demo", app="nginx-demo"} | regexp `Your rules.` | __error__!=""

### 22.4 Proper Handling of __error__

Query JSON error:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | level="error"

unwrap:

    avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 22.5 Field Name Troubleshooting

First format output fields:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | line_format "level={{.level}} status={{.status}} duration={{.duration_ms}} msg={{.msg}}"

If the field is empty:

    The field name may be incorrect.
    JSON structure may be nested.
    Logs are not in the expected format.

### 22.6 Resolution

    1. First check raw logs.
    2. Then add parser.
    3. Check __error__.
    4. Confirm field names.
    5. Then add field filtering.
    6. Finally perform unwrap and aggregation.
    7. Do not start with complex queries.

---

## Twenty-three, Fault Nineteen: Log Time Abnormality

### 23.1 Phenomenon

    Logs were just written, but not found in the last 15 minutes.
    Queries only return logs from long ago or future times.
    Loki logs indicate entry too far behind.
    Loki logs indicate timestamp too new.
    Loki rejects old logs.

### 23.2 Common Causes

    Application log time is incorrect.
    Node time is out of sync.
    Container time is abnormal.
    Manual push timestamp is incorrect.
    date +%s and date +%s%N are mixed.
    reject_old_samples is enabled.
    reject_old_samples_max_age is too short.
    Collector reads historical old logs.

### 23.3 Check Node Time

    date

    timedatectl

Check all nodes:

    ssh root@10.0.0.20 date

    ssh root@10.0.0.21 date

    ssh root@10.0.0.22 date

### 23.4 Check Pod Internal Time

    kubectl exec -it <pod-name> -n app-demo -- date

### 23.5 Query Specific Large Range

    END=$(date +%s%N)
    START=$(date -d "24 hours ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' | jq

### 23.6 Resolution

    1. Fix node NTP.
    2. Fix application log time.
    3. Manually push using date +%s%N.
    4. Check reject_old_samples_max_age.
    5. Avoid letting the collector import large amounts of old logs.
    6. Plan separately for historical import tasks.

---

## Twenty-four, Fault Twenty: Loki Pod CrashLoopBackOff

### 24.1 Phenomenon

    Loki Pod CrashLoopBackOff
    Loki Pod restart count increases
    /ready is not ready
    Loki Pod fails to start after helm install/upgrade

### 24.2 Check Pod

    kubectl get pods -n logging -o wide

    kubectl describe pod <loki-pod-name> -n logging

### 24.3 Check Previous Log

    kubectl logs <loki-pod-name> -n logging --previous --tail=300

Filter:

    kubectl logs <loki-pod-name> -n logging --previous --tail=500 | grep -Ei "error|panic|failed|config|yaml|storage|s3|minio|permission|ring|schema"

### 24.4 Common Causes /think

# values Configuration Error
# YAML Rendering Error
# schemaConfig Error
# storage Configuration Error
# Bucket Does Not Exist
# S3 Key Error
# PVC Pending
# PVC Permission Issue
# Out Of Memory OOMKilled
# replication_factor Unreasonable
# deploymentMode Configuration Conflict
# Chart Field Version Mismatch

### 24.5 Check Events

    kubectl describe pod <loki-pod-name> -n logging

Focus on:

    Events
    OOMKilled
    FailedMount
    FailedScheduling
    ImagePullBackOff
    CrashLoopBackOff

### 24.6 Check PVC

    kubectl get pvc -n logging

    kubectl describe pvc <pvc-name> -n logging

### 24.7 Check Helm Rendering

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <CHART_VERSION> \
      -f values-loki.yaml \
      > loki-rendered-debug.yaml

Check:

    grep -n "storage" loki-rendered-debug.yaml

    grep -n "schema" loki-rendered-debug.yaml

    grep -n "deploymentMode" loki-rendered-debug.yaml

### 24.8 Handling

    1. First check --previous logs.
    2. Fix values.
    3. Upgrade after helm template passes.
    4. Check PVC and StorageClass.
    5. Check MinIO / bucket / secret.
    6. If caused by upgrade, first helm rollback.
    7. Do not blindly delete PVC to avoid data loss.

---

## 25. Fault Twenty-One: Pod Running But Business Logs Not Entering Loki

### 25.1 Phenomenon

    Pod Running
    Service Normal
    curl Business Normal
    Loki Cannot Find Corresponding Business Logs

### 25.2 Common Causes

    Application Does Not Output stdout/stderr
    Application Writes Logs to File
    Log Path Not Collected by Alloy
    Log Level Too High, No Access Logs
    Request Not Hit Expected Pod
    Multi-Replica Access to Other Pods
    app Label Does Not Exist
    Container Name Mistake
    Alloy Filtered the Namespace/App

### 25.3 Check Application stdout

    kubectl logs <pod-name> -n app-demo --tail=50

If no logs but container has files:

    kubectl exec -it <pod-name> -n app-demo -- sh

Inside container:

    find / -name "*.log" 2>/dev/null | head

### 25.4 Handling Direction

    1. Prioritize changing application log output to stdout/stderr.
    2. If must use file logs, configure Alloy file collection.
    3. Add standard labels to Pod.
    4. Confirm request actually hits the Pod.
    5. Check Alloy collection rules.

---

## 26. Fault Twenty-Two: Service Normal, Pod Normal, But Loki Logs Show Business Abnormality

### 26.1 Scenario

This is a common issue in production:

    Kubernetes Resource Status Appears Normal.
    Pod Running.
    Service Endpoint Normal.
    But business access fails.
    Loki logs show error / timeout / 5xx.

### 26.2 Troubleshooting Approach

First check Service / Endpoint:

    kubectl get svc -n app-demo

    kubectl get endpoints -n app-demo

Then check logs:

    {namespace="app-demo", app="nginx-demo"} |~ "(?i)error|timeout|exception|failed|500|502|503"

Then check Pod Events:

    kubectl describe pod <pod-name> -n app-demo

Then check configuration:

    kubectl get cm -n app-demo

    kubectl get secret -n app-demo

### 26.3 Common Business Abnormalities

Logs may show:

    database connection refused
    redis timeout
    dns lookup failed
    upstream timeout
    config file not found
    permission denied
    authentication failed
    500 Internal Server Error
    connection reset by peer

### 26.4 Handling

    1. Confirm the type of error in logs.
    2. Determine if it's a dependency service, configuration, permission, network, or code issue.
    3. Combine with kubectl describe and Events.
    4. Combine with Prometheus metrics.
    5. Rollback application or configuration if necessary.

---

## 27. Fault Twenty-Three: High Cardinality Causes Loki Abnormality

### 27.1 Phenomenon

    Loki Writing Slows Down
    Active Streams Surge
    429 Increases
    Query Slows Down
    Memory Increases
    Object Storage Object Count Increases
    Loki Logs Show Too Many Streams
    Maximum Active Stream Limit Exceeded

### 27.2 Check Labels

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Warning: /think

trace_id  
request_id  
user_id  
order_id  
session_id  
client_ip  
full_url  
error_message  

### 27.3 Check Label Values Count  

    curl -s http://127.0.0.1:3100/loki/api/v1/label/trace_id/values | jq '.data | length'  

### 27.4 Check Series Count  

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match[]={namespace="app-demo"}' \
      | jq '.data | length'  

### 27.5 Handling  

    1. Immediately stop generating high cardinality labels.  
    2. Modify Alloy relabel configuration.  
    3. No longer extract trace_id / request_id as labels.  
    4. Keep dynamic fields in log body.  
    5. Use | json to query trace_id.  
    6. Wait for old data retention to expire.  
    7. Temporarily reduce retention for relevant data if necessary.  
    8. Increase max_streams limit to prevent spread.  

---

## Twenty-Eighth, Fault Twenty-Four: Alloy Resource Usage Too High  

### 28.1 Symptoms  

    Alloy CPU high  
    Alloy memory high  
    Alloy restarts  
    Alloy send failure  
    Log collection delay on nodes  
   Obviously. delay in Loki logs  

### 28.2 Common Causes  

    Large collection scope  
    Sudden surge in log volume  
    Heavy regular expression processing  
    Complex data masking rules  
    Complex multiline log processing  
    Slow Loki output  
    429 retry backlog  
    Unreasonable batch configuration  
    Too low resource limits  

### 28.3 Check Alloy Pod  

    kubectl get pod -n logging -o wide | grep alloy  

    kubectl describe pod <alloy-pod-name> -n logging  

    kubectl logs <alloy-pod-name> -n logging --tail=300  

If metrics-server is available:  

    kubectl top pod -n logging | grep alloy  

### 28.4 Handling  

    1. Limit collection namespace.  
    2. Filter health check and debug logs.  
    3. Reduce complex regular expressions.  
    4. Increase Alloy resources.  
    5. Check if Loki writing is throttled.  
    6. Check network and Gateway.  
    7. Avoid duplicate collection.  

---

## Twenty-Ninth, Fault Twenty-Five: MinIO Capacity Growth Too Fast  

### 29.1 Symptoms  

    MinIO bucket capacity grows rapidly  
    Disk space insufficient  
    Loki query slows down  
    Retention seems ineffective  
    Extremely large number of objects  

### 29.2 Investigate Log Source  

Top namespace:  

    topk(10,  
      sum by (namespace) (  
        count_over_time({namespace=~".+"}[5m])  
      )  
    )  

Top app:  

    topk(10,  
      sum by (namespace, app) (  
        count_over_time({namespace=~".+"}[5m])  
      )  
    )  

Top pod:  

    topk(10,  
      sum by (namespace, pod) (  
        count_over_time({namespace=~".+"}[5m])  
      )  
    )  

### 29.3 Check MinIO Capacity  

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh  

Inside the container:  

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123  

    mc du local/loki-chunks  

Exit:  

    exit  

### 29.4 Handling  

    1. Identify high log volume app.  
    2. Govern application logs.  
    3. Check for duplicate collection.  
    4. Check for high cardinality.  
    5. Check if retention is enabled.  
    6. Check if compactor is working.  
    7. Check if MinIO lifecycle is reasonable.  
    8. Evaluate expanding MinIO or migrating object storage.  

---

## Thirtieth, Fault Twenty-Six: Loki Abnormal After Upgrade  

### 30.1 Symptoms  

    Loki fails to start after helm upgrade  
    Query anomalies  
    Write anomalies  
    Retention not effective  
    Ruler rule anomalies  
    Incompatible values fields  

### 30.2 Must Do Before Upgrade  

Backup values:  

    helm get values loki -n logging -a > backup-values-loki-before-upgrade.yaml  

Check history:  

    helm history loki -n logging  

Export default values:  

    helm show values grafana-community/loki \
      --version <NEW_CHART_VERSION> \
      > values-loki-new-default.yaml  

Render:  

    helm template loki grafana-community/loki \
      --namespace logging \
      --version <NEW_CHART_VERSION> \
      -f values-loki-prod.yaml \
      > loki-upgrade-rendered.yaml  

### 30.3 Post-Upgrade Check  

    helm history loki -n logging  

    kubectl get pods -n logging -o wide  

    curl -s http://127.0.0.1:3100/ready  

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

kubectl logs <loki-pod-name> -n logging --tail=300 | grep -Ei "error|warn|deprecated|schema|storage|ruler|compactor"

### 30.4 Quick Rollback

    helm rollback loki <REVISION> -n logging

Post-rollback verification:

    kubectl get pods -n logging

    curl -s http://127.0.0.1:3100/ready

### 30.5 Upgrade Notes

Key focus areas:

    Chart field changes
    deploymentMode changes
    storage configuration changes
    schema compatibility
    compactor configuration changes
    ruler configuration changes
    Simple Scalable deprecation risk
    Grafana DataSource adjustment needed
    Alloy write address changes

---

## Thirty-one, Loki Fault Emergency Handling Priority

### 31.1 P0: Log Platform Completely Unavailable

Manifestations:

    Grafana queries all fail
    Loki /ready unavailable
    Alloy writes fail in bulk
    Loki PodMass CrashLoop
    Object storage unavailable

Priority handling:

    1. Restore Loki Pod Ready.
    2. Restore Gateway.
    3. Restore object storage.
    4. Rollback with helm if necessary.
    5. Temporarily pause high log sources.
    6. Notify business of log query impact.

### 31.2 P1: Write Failures but Historical Queries Available

Manifestations:

    Historical logs can be queried.
    New logs not entering Loki.
    Alloy failed to send batch.

Priority handling:

    1. Check Alloy errors.
    2. Check Gateway push.
    3. Check write / ingester.
    4. Check 429.
    5. Check MinIO write.
    6. Check limits.

### 31.3 P2: Slow Queries or Partial Query Failures

Manifestations:

    Simple queries normal.
    Large Dashboard slow.
    query_range timeout.

Priority handling:

    1. Limit Dashboard time range.
    2. Prohibit large range All queries.
    3. Optimize LogQL.
    4. Expand read capacity.
    5. Check object storage.
    6. Check high cardinality.

### 31.4 P3: Alert Anomalies

Manifestations:

    Logs can be queried.
    Ruler alerts not triggered.
    AlertManager not receiving.

Priority handling:

    1. Validate expr.
    2. Check rules API.
    3. Check Ruler logs.
    4. Check AlertManager.
    5. Check route / silence.

---

## Thirty-two, Production Troubleshooting Checklist

### 32.1 Collection Layer Checklist

    [ ] Application kubectl logs has output
    [ ] Alloy Pod Running
    [ ] Alloy DaemonSet covers target nodes
    [ ] Alloy RBAC has pods/log permissions
    [ ] Alloy configuration loaded successfully
    [ ] Alloy not filtering target namespace
    [ ] Alloy write URL correct
    [ ] Alloy logs no failed to send batch
    [ ] No duplicate collection

### 32.2 Write Layer Checklist

    [ ] Loki Gateway /ready normal
    [ ] /loki/api/v1/push path correct
    [ ] Manual push successful
    [ ] Loki logs no 400/429/500
    [ ] No rate limit exceeded
    [ ] No active stream limit exceeded
    [ ] No timestamp error
    [ ] No label error
    [ ] MinIO accessible
    [ ] Bucket exists

### 32.3 Query Layer Checklist

    [ ] labels API normal
    [ ] namespace values exist
    [ ] app values exist
    [ ] query_range simple query normal
    [ ] Query time range correct
    [ ] Label names and values correct
    [ ] Grafana Data Source correct
    [ ] Dashboard variables not empty
    [ ] No large range All queries
    [ ] No global complex regexp/json

### 32.4 Storage Layer Checklist

    [ ] MinIO Pod Running
    [ ] MinIO Service Endpoint normal
    [ ] Loki can access MinIO
    [ ] Bucket exists
    [ ] AccessKey / SecretKey correct
    [ ] s3ForcePathStyle correct
    [ ] HTTP / HTTPS configuration correct
    [ ] MinIO capacity sufficient
    [ ] Compactor normal
    [ ] Retention configuration correct

### 32.5 Alert Layer Checklist

    [ ] Ruler enabled
    [ ] Ruler API accessible
    [ ] Rule loaded
    [ ] expr manual execution has result
    [ ] Threshold reasonable
    [ ] for time met
    [ ] AlertManager URL correct
    [ ] AlertManager /-/ready normal
    [ ] AlertManager route correct
    [ ] Not silenced
    [ ] webhook / notification channels normal

---

## Thirty-three, Common Misconceptions

### 33.1 Misconception One: Grafana Can't Find Logs Means Loki Is Broken

Not necessarily.

Possible reasons:

    Time range incorrect
    Variables empty
    LogQL written incorrectly
    Data source selected incorrectly
    app label does not exist
    Alloy not collecting

### 33.2 Misconception Two: kubectl get pod Running Means Logs Can Definitely Be Collected

Incorrect.

Pod Running only indicates container is running, not: /think

Application has stdout logs  
Alloy has captured them  
Alloy successfully wrote to Loki  
Loki can query them  

### 33.3 Misconception Three: Seeing 429 Means You Should Increase Limits  

Error.  

Should first check:  

    Who is scraping logs  
    Whether high cardinality exists  
    Whether logs are being collected redundantly  
    Whether DEBUG is enabled  
    Whether health check logs are excessive  

### 33.4 Misconception Four: MinIO Bucket Being Empty Means Loki Didn't Write  

Not necessarily.  

Recent logs may still be in the ingester.  

Chunk flush to object storage has delay.  

Need to check simultaneously:  

    Whether push was successful  
    Whether query was successful  
    Whether Loki has storage errors in logs  
    Whether objects appear after waiting  

### 33.5 Misconception Five: Dashboard Variables Can Be Used for Access Control  

Error.  

Variables are only for query assistance, not access boundaries.  

Real access control needs:  

    Grafana permissions  
    Data source permissions  
    Loki multi-tenancy  
    Reverse proxy authentication  
    NetworkPolicy  

### 33.6 Misconception Six: Log Alert Not Triggering Means AlertManager Problem  

Not necessarily.  

More often:  

    LogQL has no results  
    Threshold not reached  
    for duration not satisfied  
    Ruler hasn't loaded rules  
    Ruler failed to connect to AlertManager  

### 33.7 Misconception Seven: Deleting PVC Can Quickly Recover Loki  

Dangerous.  

PVC may contain WAL, cache, index working directories, or log data.  

Do not delete PVCs in production environments.  

---

## Thirty-Four, Production Runbook Template  

### 34.1 Loki Can't Find Business Logs Runbook  

Trigger condition:  

    Business Pod has logs, but Grafana / Loki can't find them.  

Handling steps:  

    1. kubectl logs to verify application logs.  
    2. kubectl get pod -o wide to confirm Pod's node.  
    3. Confirm Alloy is running on the same node.  
    4. Check Alloy logs.  
    5. Check Alloy RBAC.  
    6. Check Alloy loki.write URL.  
    7. Check Loki /ready.  
    8. Check Loki labels.  
    9. Execute LogQL from broad to narrow.  
    10. Check if logs were written to another Loki.  
    11. Fix and re-generate business logs for verification.  

### 34.2 Loki Gets 429 on Write Runbook  

Trigger condition:  

    Alloy failed to send batch with 429.  

Handling steps:  

    1. Check Alloy logs to confirm 429.  
    2. Check Loki logs to confirm rate limit reason.  
    3. Statistic Top namespace/app/pod log volume.  
    4. Check high cardinality labels.  
    5. Check for redundant collection.  
    6. Temporarily lower log level of scraping application.  
    7. Drop logs if necessary on collection side.  
    8. Evaluate whether to adjust limits.  
    9. Observe if 429 recovers.  

### 34.3 Loki Query Slow Runbook  

Trigger condition:  

    Grafana query timeout or Dashboard lag.  

Handling steps:  

    1. Find slow LogQL.  
    2. Narrow time range.  
    3. Add namespace/app conditions.  
    4. Avoid All variables.  
    5. Avoid global json/regexp.  
    6. Check read / querier resources.  
    7. Check MinIO delay.  
    8. Check high cardinality.  
    9. Optimize Dashboard.  
    10. Scale up read if necessary.  

### 34.4 Loki Object Storage Abnormality Runbook  

Trigger condition:  

    Loki logs show s3/minio/storage errors.  

Handling steps:  

    1. Check MinIO Pod.  
    2. Check MinIO Service / Endpoint.  
    3. curl MinIO from Loki Namespace.  
    4. Use mc to check bucket.  
    5. Check access key / secret key.  
    6. Check path-style / insecure / endpoint.  
    7. Check MinIO capacity.  
    8. Check object storage permissions.  
    9. Validate after fixing by push/query.  

### 34.5 Loki Alert Not Triggering Runbook  

Trigger condition:  

    Expected log alert not appearing in AlertManager.  

Handling steps:  

    1. Manually execute rule expr.  
    2. Confirm expr returns value exceeding threshold.  
    3. Confirm for duration is satisfied.  
    4. Query /loki/api/v1/rules.  
    5. Check Ruler logs.  
    6. Test AlertManager /-/ready.  
    7. Check alertmanager_url.  
    8. Check AlertManager UI.  
    9. Check silence / route.  
    10. Use webhook demo to validate notification.  

---

## Thirty-Five, Practical Tasks  

### 35.1 Task One: Complete Full-Chain Health Check  

Execute:  

    kubectl get pods -n logging -o wide  

    kubectl get svc -n logging  

    kubectl get endpoints -n logging  

    kubectl port-forward svc/loki-gateway 3100:80 -n logging  

    curl -s http://127.0.0.1:3100/ready  

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq  

Acceptance:  

    [ ] Loki Pod Running  
    [ ] Gateway Service Normal  
    [ ] Endpoint Not Empty  
    [ ] /ready Returns ready  
    [ ] labels API Normal  

### 35.2 Task Two: Verify Application Logs to Loki  

Execute:  

    kubectl get pod -n app-demo -o wide  

    kubectl logs <pod-name> -n app-demo --tail=50

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

Verification:

    [ ] kubectl logs has logs
    [ ] Loki can retrieve app-demo logs

### 35.3 Task 3: Verify Manual Push

Execution:

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-troubleshooting-test\",
              \"namespace\": \"app-demo\",
              \"app\": \"manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki troubleshooting test\"]
            ]
          }
        ]
      }"

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-troubleshooting-test"}' \
      --data-urlencode 'limit=10' | jq

Verification:

    [ ] Push successful
    [ ] Query successful

### 35.4 Task 4: Simulate Empty Query Troubleshooting

Execute an error query:

    {namespace="not-exist"}

Then troubleshoot in order:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

Verification:

    [ ] Can detect namespace typo via label values
    [ ] Can recover correct query

### 35.5 Task 5: Check High Cardinality Risk

Execution:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Check for existence of:

    trace_id
    request_id
    user_id
    order_id
    session_id

Verification:

    [ ] Can identify high cardinality risk label
    [ ] Can explain mitigation plan

### 35.6 Task 6: Check MinIO

Execution:

    kubectl get pods -n minio -o wide

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside container:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

    mc ls local

Verification:

    [ ] MinIO Pod Running
    [ ] MinIO Endpoint not empty
    [ ] mc can list bucket

### 35.7 Task 7: Verify Ruler

Execution:

    curl -s http://127.0.0.1:3100/loki/api/v1/rules | jq

Verification:

    [ ] Ruler API accessible
    [ ] Can see loaded rules or empty rule list
    [ ] If inaccessible, can follow Ruler troubleshooting process

---

## Thirty-six, Acceptance Checklist

After completing this document, confirm:

    [ ] Can draw Loki collection-write-storage-query-alarm chain
    [ ] Can use kubectl logs to verify application log output
    [ ] Can confirm Alloy covers target nodes
    [ ] Can check Alloy RBAC
    [ ] Can check Alloy loki.write URL
    [ ] Can verify Loki via /ready
    [ ] Can verify Loki log receipt via /labels
    [ ] Can query logs via query_range
    [ ] Can manually push test Loki write
    [ ] Can troubleshoot 400 Bad Request
    [ ] Can troubleshoot 404 Not Found
    [ ] Can troubleshoot 429 Too Many Requests
    [ ] Can troubleshoot 500 Internal Server Error
    [ ] Can troubleshoot Gateway 502 / 503
    [ ] Can troubleshoot empty query_range
    [ ] Can troubleshoot query_range timeout
    [ ] Can troubleshoot Grafana Data Source failure
    [ ] Can troubleshoot empty Dashboard variables
    [ ] Can troubleshoot MinIO / S3 anomalies
    [ ] Can troubleshoot retention not taking effect
    [ ] Can troubleshoot Ruler API unavailable
    [ ] Can troubleshoot Loki alarm not triggering
    [ ] Can troubleshoot AlertManager not receiving alarms
    [ ] Can identify high cardinality label
    [ ] Can troubleshoot log time anomalies
    [ ] Can troubleshoot Loki CrashLoopBackOff
    [ ] Can write Loki production Runbook

---

## Thirty-seven, Summary

This document completes Loki common troubleshooting practice.

Loki troubleshooting cannot focus on a single component.

Must troubleshoot by layering the chain: /think

Does the application output logs?
  ↓
Does Alloy collect logs?
  ↓
Does Alloy write to Loki?
  ↓
Is Loki Gateway healthy?
  ↓
Is Loki write/ingester healthy?
  ↓
Is object storage healthy?
  ↓
Is Loki read/querier healthy?
  ↓
Is Grafana query correct?
  ↓
Does Ruler execute rules?
  ↓
Does AlertManager receive alerts?

Core troubleshooting approach:

    First verify kubectl logs.
    Then verify Alloy.
    Then verify Loki /ready.
    Then verify hand push.
    Then verify labels.
    Then verify query_range.
    Then check Grafana.
    Finally check Ruler / AlertManager.

Common issue diagnosis directions:

    No logs in kubectl logs:
        Application issue or log output method issue.

    Logs in kubectl logs but no logs in Loki:
        Alloy collection or write issue.

    Manual push fails:
        Loki Gateway / write / limits / storage issue.

    Manual push succeeds but business logs missing:
        Alloy configuration or collection scope issue.

    Labels exist but query is empty:
        Time range, label value, LogQL issue.

    Grafana empty but curl works:
        Data Source, variables, time range, Dashboard issue.

    Loki 429:
        Rate limiting, log volume, high cardinality, duplicate collection issue.

    Loki 500:
        Internal components, storage, ring, PVC, configuration issue.

    Retention not effective:
        compactor, retention, delete_request_store, object storage permissions issue.

    Alerts not triggered:
        expr, threshold, for, Ruler, AlertManager routing issue.

In production, the most important thing is not remembering all commands, but establishing a troubleshooting path:

    Troubleshooting level
      ↓
    Choose minimal verification command
      ↓
    Start with simple queries
      ↓
    Narrow down scope step by step
      ↓
    Finally modify configuration or scale up

Next article:

    15-Loki Comprehensive Practice: Building a K8S Logging Platform Loop from Scratch

Key learning points:

    MinIO
    Loki
    Alloy
    Grafana
    Dashboard
    Ruler
    AlertManager
    retention
    limits
    Runbook
    Full Logging Platform Loop Practice

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Troubleshoot Loki operations:
  https://grafana.com/docs/loki/latest/operations/troubleshooting/troubleshoot-operations/

- Troubleshoot log ingestion WRITE:
  https://grafana.com/docs/loki/latest/operations/troubleshooting/troubleshoot-ingest/

- Troubleshoot log queries READ:
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

- Loki HTTP API:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- Request validation and rate limits:
  https://grafana.com/docs/loki/latest/operations/request-validation-rate-limits/

- Log retention:
  https://grafana.com/docs/loki/latest/operations/storage/retention/

- Loki configuration:
  https://grafana.com/docs/loki/latest/configure/

- Query Loki:
  https://grafana.com/docs/loki/latest/query/

- Query best practices:
  https://grafana.com/docs/loki/latest/query/bp-query/

- Grafana Alloy Documentation:
  https://grafana.com/docs/alloy/latest/

- loki.source.kubernetes:
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- Collect Kubernetes logs and forward them to Loki:
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- Grafana Loki Data Source:
  https://grafana.com/docs/grafana/latest/datasources/loki/

- AlertManager Configuration:
  https://prometheus.io/docs/alerting/latest/configuration/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- Kubernetes kubectl logs:
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/

- Kubernetes Services:
  https://kubernetes.io/docs/concepts/services-networking/service/

- Kubernetes NetworkPolicy:
  https://kubernetes.io/docs/concepts/services-networking/network-policies/