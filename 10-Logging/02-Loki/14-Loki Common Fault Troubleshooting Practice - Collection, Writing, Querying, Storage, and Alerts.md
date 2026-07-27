# 14-Loki Common Fault Troubleshooting Practice: Collection, Writing, Querying, Storage, and Alerts

## Document Description

This article is the fourteenth in the specialized study on Loki, systematically organizing common fault troubleshooting methods for Loki in both production and experimental Kubernetes environments.

Previous topics have included:

    01-Loki Basics and Experimental Environment Planning
    02-Loki Architecture Principles and Component Responsibilities
    03-Loki Deployment Modes Comparison and Experimental Selection
    04-Helm-Based Single-Loki Mode Deployment Practice
    05-Connecting Loki Object Storage to MinIO
    06-Grafana-Alloy for Collecting K8S-Pod Logs
    07-Loki Tag Design and High Cardinality Issues
    08-LogQL Basic Queries: Retrieving Namespace, Pod, and Container Logs
    09-Advanced LogQL Queries: json, logfmt, regexp, unwrap
    10-Grafana Integration with Loki for Log Dashboards
    11-Loki Log Alerts: Ruler and AlertManager Collaboration
    12-Loki Production Management: Log Volume, Retention Period, Throttling, Security
    13-Loki Performance and High Availability: Introduction to Simple-Scalable Mode

This article focuses on troubleshooting and does not delve into extensive theory.

Troubleshooting areas include:

    Alloy failing to collect Pod logs
    Alloy having logs but Loki unable to find them
    Loki writing failures
    Loki Gateway 502 / 503 errors
    query_range returning empty results
    query_range timeouts
    Grafana Explore showing no data
    Grafana Dashboard variables being empty
    MinIO/S3 object storage issues
    retention settings not taking effect
    Ruler alerts not triggering
    AlertManager not receiving Loki alerts
    High cardinality causing writing or querying problems
    429 throttling
    Abnormal log timestamps
    JSON/logfmt/regexp parsing errors
    Pods running but no business logs generated
    Loki Pod CrashLoopBackOff
    Loki's own monitoring and emergency handling

The goal of this article is to develop a practical Loki troubleshooting playbook.

---

## Tags

#Loki #Grafana #GrafanaAlloy #LogQL #Kubernetes #Log Troubleshooting #Runbook #MinIO #AlertManager #Ruler #SRE #Observability #Fault Handling #Production Troubleshooting

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/14-Loki Common Fault Troubleshooting Practice: Collection, Writing, Querying, Storage, and Alerts.md

---

## I. Troubleshooting Objectives

After completing this article, you should be able to:

    1. Establish a comprehensive approach to Loki troubleshooting.
    2. Distinguish between collection, writing, querying, storage, and alerting issues.
    3. Master the troubleshooting steps for Alloy failing to collect logs.
    4. Understand how to resolve Loki push writing failures.
    5. Identify solutions for Gateway 502 / 503 errors.
    6. Determine how to address query_range empty results.
    7. Learn how to troubleshoot Grafana Explore/Dashboard anomalies.
    8. Master methods for MinIO/S3 storage issues.
    9. Know how to fix retention settings not functioning properly.
    10. Understand how to resolve Ruler alerts not triggering.
    11. Find solutions for AlertManager not receiving Loki alerts.
    12. Learn how to handle 429 throttling and high cardinality problems.
    13. Identify methods for Loki Pod CrashLoopBackOff.
    14. Organize a production Loki fault handling playbook.
    15. Create a duty-based troubleshooting checklist.

---

## II. Experimental and Production Environment Assumptions

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

Single Loki Gateway:

    loki-gateway.logging.svc.cluster.local

Simple Scalable Loki Gateway:

    loki-ssd-gatewayLogging-ssd.svc.cluster.local

MinIO:

    minio.minio.svc.cluster.local:9000

Grafana:

    grafana### 4.4 Viewing Gateway Logs

    kubectl logs <loki-gateway-pod> -n logging --tail=300

    kubectl logs <loki-gateway-pod> -n logging --tail=500 | grep -Ei "502|503|500|push|query|ready|upstream"

### 4.5 Basic Checks for the Loki API

Port Forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Checking readiness:

    curl -s http://127.0.0.1:3100/ready

Checking metrics:

    curl -s http://127.0.0.1:3100/metrics | head

Checking labels:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Checking namespace values:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

Checking app values:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/app/values | jq

### 4.6 Basic Queries with query_range

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

Querying the last 1 hour:

    END=$(date +%s%N)
    START=$(date -d "1 hour ago" +%s%N)

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq

### 4.7 Manual Push Test

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-troubleshooting-test\",
              \"namespace\": \"app-demo\"",
              \"app\": \"manual-test\"
            },
            \"values\": [
              [\"${TS}\", \"hello loki troubleshooting test\"]
            ]
          }
        ]
      }"

Querying the results:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="manual-troubleshooting-test"}' \
      --data-urlencode 'limit=10' | jq

---

## Section 5: Issue: kubectl logs Shows Logs, but Loki Cannot Access Them

### 5.1 Observation

The application Pod can display logs:

    kubectl logs <pod-name> -n app-demo --tail=50

However, queries in Loki yield no results:

    {namespace="app-demo"}

### 5.2 Troubleshooting Steps

Complete path analysis:

    The application has logs
      ↓
    Is Alloy running?
      ↓
    Is Alloy on the node where the Pod is located?
      ↓
    Does Alloy have permission to read pods/log?
      ↓
    Has Alloy successfully sent the logs to Loki?
      ↓
    Has Loki received the labels?
      ↓
    Are the query time range and labels correct?

### 5.3 Confirming the Node Where the Pod Is Located

    kubectl get pod -n app-demo -o wide

Record:

    Pod:
        <pod-name>

    Node:
        <node-name>

### 5.4 Checking if Alloy is Running on That Node

    kubectl get pod -n logging -o wide | grep alloy

Verify if there is an Alloy Pod running on the same node.

If not:

    Logs from that node may not be being collected.

Common reasons include:

    The Alloy DaemonSet has not been scheduled to that node.
    The node has a taint.
    Alloy lacks the necessary tolerations.
    There are nodeSelector restrictions.
    Insufficient resources.
    Issues with image retrieval.
    PodSecurity constraints.

### 5.5 Checking the Alloy DaemonSet

    kubectlhttp://loki-ssd-gateway.logging-ssd.svc.cluster.local/loki/api/v1/push

Common Issues:

- Incorrect Namespace entry.
- Incorrect Service name entry.
- Omission of /loki/api/v1/push in the path.
- Using port 3100 instead of the correct port 80 for the Gateway Service.
- Grafana querying one Loki while Alloy is writing to another Loki.

### 5.9 Testing the Loki Gateway from the Namespace Where Alloy Is Located

Run the following command using `kubectl`:

```bash
kubectl run curl-loki-test \
  --rm -it \
  --image=curlimages/curl:8.5.0 \
  -n logging \
  -- sh
```

Inside the container, execute these commands:

```bash
curl -s http://loki-gateway.logging.svc.cluster.local/ready
curl -s http://loki-gatewaylogging.svc.cluster.local/loki/api/v1/labels
exit
```

If there are issues, investigate in the following areas: Loki Gateway, Service configuration, DNS settings, and NetworkPolicy rules.

### 5.10 Checking Loki Labels

Use the following command to retrieve labels:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq
```

If the `app-demo` label is not present, it may indicate that Alloy did not successfully write data or that the written label format is incorrect.

Continue to check:

```bash
curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq
```

If `app-demo` is still missing, it could mean that Alloy did not capture logs for `app-demo`, or that the logs were removed during relabeling processes. Alternatively, it might indicate that the logs are stored in a different Loki instance.

---

## Section Six: Issue Two: Logs Can Be Manually Pushed to Loki, But Alloy Fails to Collect Them

### 6.1 Observation

Manual push is successful:

```bash
{job="manual-troubleshooting-test"}
```

Logs can be found using this method, but real Pod logs are not available:

```bash
{namespace="app-demo"}
```

### 6.2 Possible Causes

This suggests that the Loki server and Gateway are likely functioning correctly, as well as the API used for writing logs. The problem may lie in the following areas:

- Alloy's collection configuration.
- Alloy's RBAC (Role-Based Access Control) settings.
- Alloy's scheduling or deployment rules.
- Rules related to relabeling or dropping logs.
- The way application logs are produced and output.

### 6.3 Checking If Alloy Filters Out Certain Namespaces

Review Alloy's configuration using the following command:

```bash
kubectl get cm <alloy-configmap-name> -n logging -o yaml
```

Search for settings related to `keep`, `drop`, `namespace`, and `app-demo`. If these are configured to exclude `app-demo`, it will prevent logs from being collected.

### 6.4 Checking If Only Current Nodes Are Being Collected

If Alloy uses a DaemonSet, it is common to configure it to collect logs only from the current node. Check if the configuration includes settings like `spec.nodeName=` to ensure that logs are only collected from the intended node.

### 6.5 Checking the Method Used for Log Collection

Alloy may use one of these two methods to collect Pod logs:

- `loki.source.kubernetes`: Collects logs directly through the Kubernetes API.
- `loki.source.file`: Reads logs from `/var/log/containers` or `/var/log/pods`.

If using `loki.source.kubernetes`, verify RBAC permissions, Kubernetes API access, and permissions for accessing `pods/log`.

If using `loki.source.file`, check the hostPath mounting settings, file path, file permissions, and any relabeling rules that might affect log collection.

### 6.6 Checking Container Log Paths

For file-based log collection, common paths on nodes include:

- `/var/log/containers/*.log`
- `/var/log/pods/*/*/*.log`

Check these paths on the node using the following commands:

```bash
ls -lh /var/log/containers | head
ls -lh /var/log/pods | head
```

If these paths are not mounted in the Alloy container, log collection will not work.

---

## Section Seven: Issue Three: 400 Bad Request When Pushing Logs to Loki

### 7.1 Observation

When manually pushing logs or using Alloy to write data, a `400 Bad Request` error occurs.

### 7.2 Common Causes

- Incorrect JSON format.
- Errors in the `values` structure.
- The `timestamp### 9.3 Do not increase limits directly at first

Error handling:

    Immediately increasing all limits upon seeing a 429 error.

Correct approach:

    First, identify which component is generating the excessive log volume.

### 9.4 Identify the namespace with the highest log volume

    topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 9.5 Identify the app with the highest log volume

    topk(10,
      sum by (namespace, app) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 9.6 Identify the pod with the highest log volume

    topk(10,
      sum by (namespace, pod) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

### 9.7 Check for high-cardinality labels

View labels:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Be cautious of:

    request_id
    trace_id
    user_id
    order_id
    session_id
    full_url
    error_message

Count the number of values for a specific label:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/trace_id/values | jq '.data | length'

### 9.8 Check for duplicate data collection

Phenomenon:

    The same log appears multiple times in Loki.
    Multiple tools like Alloy, Promtail, and Fluent Bit are writing logs simultaneously.
    Each DaemonSet node collects logs from all pods in the cluster.
    Incorrect configuration of Alloy's field selector.

Check for duplicates:

    kubectl get pods -A | grep -Ei "alloy|promtail|fluent|filebeat"

### 9.9 View discarded metrics from Loki

If /metrics is accessible:

    curl -s http://127.0.0.1:3100/metrics | grep -E "loki_discarded_samples_total|loki_discarded_bytes_total"

These metrics can help determine the reason for data discarding.

### 9.10 Action steps

Sequence of actions:

    1. Identify the application generating excessive logs.
    2. Reduce the log level of that application.
    3. Filter out healthz, metrics, and debug logs.
    4. Correct any high-cardinality label issues.
    5. Eliminate duplicate data collection.
    6. Adjust limits_config appropriately.
    7. Expand write / ingester resources if necessary.
    8. Continuously monitor whether the 429 error persists.

---

## Section X: Issue Six: Loki Push Returns 500 Internal Server Error

### 10.1 Phenomenon

    A 500 Internal Server Error is encountered.

### 10.2 Common causes

    Internal issues with Loki components
    Errors in the write / ingester process
    Unhealthy ring cluster configuration
    Inaccessibility of MinIO / S3
    Non-existent buckets
    Insufficient storage permissions
    Issues with WAL / PVC storage
    Configuration errors
    Mismatch between replication_factor and actual number of replicas
    Empty backend service endpoints

### 10.3 Troubleshoot Loki logs

    kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "error|panic|failed|s3|minio|bucket|ring|ingester|flush|wal|permission"

### 10.4 Troubleshoot Gateway logs

    kubectl logs <loki-gateway-pod> -n logging --tail=300 | grep -Ei "500|502|503|push|upstream"

### 10.5 Troubleshoot MinIO

    kubectl get pod -n minio -o wide

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

    kubectl logs <minio-pod-name> -n minio --tail=200

### 10.6 Test MinIO from the Loki Namespace

    kubectl run curl-minio-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container:

    curl -I http://minio.minio.svc.cluster.local:9000

To exit:

    exit

### 10.7 Solutions

    1. Restore the Loki Pod.
    2. Restore MinIO.
    3. Verify the existence of the bucket.
    4.{namespace="app-demo"}

Check again:

{namespace="app-demo", app="nginx-demo"}

Check again:

{namespace="app-demo", pod=~"nginx-demo-.*"}

Don't start with complex queries.

### 12.6 Specify the query time range

END=$(date +%s%N)
START=$(date -d "1 hour ago" +%s%N)

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode "start=${START}" \
      --data-urlencode "end=${END}" \
      --data-urlencode 'limit=20' \
      --data-urlencode 'direction=BACKWARD' | jq

### 12.7 Verify which Loki is being used

If both Loki and Loki-SSD are available in the environment, confirm:

Which Loki Alloy writes to.
Which Loki Grafana Data Source queries.
Which Loki the curl port forwarding uses.

### 12.8 Check if the app tag exists

If querying:

{namespace="app-demo", app="nginx-demo"}

 yields no results, but:

{namespace="app-demo"}

does, it may mean the app tag does not exist or its name is different.

Check the stream:

curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=5' | jq '.data.result[].stream'

---

## Section Thirteen: Issue Nine: query_range Queries Timout or Are Slow

### 13.1 Symptoms

Grafana Explore reports:

timeout
query timeout
context deadline exceeded
maximum query length exceeded
too many outstanding requests
query exceeded limits

### 13.2 Common Causes

- The query time range is too large.
- There are no namespace/app restrictions on the query.
- Global regular expressions are used.
- Global json/regexp parsing is employed.
- High cardinality of labels.
- Too many series are being queried.
- Too many dashboards are configured.
- The Grafana variable "All" is expanded too much.
- Slow object storage read.
- Insufficient resources for read/querier.
- Limits_config triggers have been reached.

### 13.3 Example of incorrect queries

Not recommended:

{namespace=~".+"} |~ "(?i)error"

Not recommended:

{pod=~".+"} | json | level="error"

Not recommended:

{namespace=~".+"} | regexp `complex regular expression`

### 13.4 Recommended query methods

Recommended:

{namespace="app-prod", app="order-api"} |~ "(?i)error|exception|panic"

Recommended JSON:

{namespace="app-prod", app="order-api"} | json | __error__="" | level="error"

### 13.5 Check Loki read/querry logs

For standalone instances:

kubectl logs <loki-pod-name> -n logging --tail=500 | grep -Ei "query|timeout|limit|series|context|error"

For SSD instances:

kubectl logs -n logging-ssd -l app.kubernetes.io/component/read --tail=500 | grep -Ei "query|timeout|limit|series|context|error"

### 13.6 Solutions

- Reduce the time range.
- Add namespace/app conditions to the query.
- Avoid using the "All" variable.
- Avoid global json/regexp parsing.
- Optimize the number of dashboards.
- Adjust max_query_length/query_timeout limits.
- Expand read/querier resources.
- Check object storage performance.
- Address high cardinality labels.

---

## Section Fourteen: Issue Ten: Grafana Data Source Save & Test Failures

### 14.1 Symptoms

Adding a Loki Data Source in Grafana fails.

Possible error messages include:

Unable to connect with Loki
Bad Gateway
Not found
Data source connected, but no labels found
timeout

### 14.2 Check the URL

Grafana should use Service DNS when accessing Loki within a Kubernetes cluster.

For standalone instances:

http://loki-gateway.logging.svc.cluster.local

For SSD instances:

http://loki-ssd-gatewaylogging-ssd.svc.cluster.local

Do not use:

http://127.0.0.1:3100

unless Grafana is also running locally.

### 14.3 Test from inside the Grafana Pod

Run the following command:

kubectl get pod -n monitoring | grep grafana

Then execute this in the container:

kubectl exec -it <grafana-pod-name> -n monitoring -- sh

Inside the container,## Chapter Sixteen: Fault Twelve: Abnormalities in MinIO/S3 Object Storage

### 16.1 Symptoms

Loki logs may display the following errors:

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

### 16.2 Checking the MinIO Pod

    Use the following commands:

    kubectl get pods -n minio -o wide

    kubectl logs <minio-pod-name> -n minio --tail=300

### 16.3 Checking the MinIO Service/Endpoint

    Execute these commands:

    kubectl get svc -n minio

    kubectl get endpoints minio -n minio

### 16.4 Testing MinIO from the Loki Namespace

    Run the following command in a container:

    kubectl run curl-minio-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n logging \
      -- sh

Inside the container, perform the following operations:

    curl -I http://minio.minio.svc.cluster.local:9000

    To exit the container, type:

    exit

### 16.5 Using mc to Check the Bucket

    Run the following command in a container:

    kubectl run minio-mc \
      --rm -it \
      --image=registry.cn-hangzhou.aliyuncs.com/pub-syq/mc:RELEASE.2025-04-16T18-13-26Z \
      -n minio \
      -- sh

Inside the container, perform the following operations:

    mc alias set local http://minio.minio.svc.cluster.local:9000 minioadmin minioadmin123

    mc ls local

    mc ls local/loki-chunks

    mc ls local/loki-ruler

    mc ls local/loki-admin

    To exit the container, type:

    exit

### 16.6 Checking Loki Storage Configuration

View the values using the following command:

    helm get values loki -n logging -a | grep -n "storage" -A 80

Check the following settings:

    Whether the endpoint is correct.
    Whether the bucketNames are correct.
    Whether the accessKeyId and secretAccessKey are correct.
    Whether s3ForcePathStyle is set to true.
    Whether insecure is configured for HTTP/HTTPS.
    Whether the region is specified.
    Whether the bucket exists.

### 16.7 Common Configuration Errors

    Error 1: The endpoint includes a protocol, but it is not required in this case.
    Possible incorrect configuration: `endpoint: http://minio.minio.svc.cluster.local:9000`
    Correct configuration: `endpoint: minio.minio.svc.cluster.local:9000`
    Refer to the current Helm Chart and Loki configuration requirements for specific details.

    Error 2: HTTP MinIO is configured, but insecure is not enabled.

    Error 3: Path-style is required for MinIO, but s3ForcePathStyle is not set.

    Error 4: The bucket name is incorrect.

    Error 5: Using the root user works in testing, but insufficient permissions occur when using a dedicated user in production.

### 16.8 Solutions

    1. Restore the MinIO Pod.
    2. Restore the Service/Endpoint.
    3. Verify that the bucket exists.
    4. Ensure that the keys are correct.
    5. Confirm the path-style setting.
    6. Check whether HTTP/HTTPS is properly configured.
    7. Check if Loki logs have returned to normal.
    8. Reperform push/query operations for verification.

---

## Chapter Seventeen: Fault Thirteen: Retention Settings Not Taking Effect

### 17.1 Symptoms

Even though a retention period has been configured, objects in MinIO are not deleted as expected.

### 17.2 Common Causes

    The compactor is not enabled.
    retention_enabled is not set to true.
    The retention_period is not specified.
    The retention_stream does not match the required settings.
    There is an error in the delete_request_store configuration.
    The user does not have the necessary permissions to delete objects from object storage.
    The retention_delete_delay has not yet elapsed.
    The compactor schedule has not been executed.
    The current data is still within the retention period.
    There is a conflict between MinIO’s lifecycle settings and Loki’s retention rules.
    The requested data is not expired.

### 17.3 Checking the Values

Use the following command to view the values    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | __error__!=""

regexp：

    {namespace="app-demo", app="regexp-log-demo"} | regexp | __error__!=""{namespace="app-demo", app="logfmt-log-demo"} | logfmt | __error__!=""

regexp:

{namespace="app-demo", app="nginx-demo"} | regexp `your regex` | __error__!=""

### 22.4 Proper Handling of __error__

Querying JSON errors:

{namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | level="error"

Unwrapping:

avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 22.5 Field Name Verification

First, format the output fields:

{namespace="app-demo", app="json-log-demo"}
      | json
      | line_format "level={{.level}} status={{.status}} duration={{.duration_ms}} msg={{.msg}}"

If a field is empty:

The field name might be misspelled.
The JSON structure might be nested.
The log may not be in the expected format.

### 22.6 Handling Steps

1. Check the original logs first.
2. Add a parser if necessary.
3. Verify the __error__ value.
4. Confirm the field names are correct.
5. Apply field filtering as needed.
6. Finally, perform unwrapping and aggregation steps.
7. Avoid writing complex queries from the beginning.```markdown
kubectl get svc -n app-demo

kubectl get endpoints -n app-demo

Check the logs:

{namespace="app-demo", app="nginx-demo"} |~ "(?i)error|timeout|exception|failed|500|502|503"

Examine Pod Events:

kubectl describe pod <pod-name> -n app-demo

Review configurations:

kubectl get cm -n app-demo

kubectl get secret -n app-demo

### 26.3 Common Business Exceptions

Errors that may appear in the logs include:

database connection refused
redis timeout
dns lookup failed
upstream timeout
config file not found
permission denied
authentication failed
500 Internal Server Error
connection reset by peer

### 26.4 Handling Steps

1. Identify the type of error from the logs.
2. Determine whether it is due to dependent services, configurations, permissions, network issues, or code problems.
3. Use kubectl describe and Events for further analysis.
4. Refer to Prometheus metrics as needed.
5. Roll back applications or configurations if necessary.

---

## Chapter 27: Fault 23: High基数 Causes Abnormalities in Loki

### 27.1 Symptoms

- Slow writing to Loki
- A sudden increase in active streams
- An increase in 429 errors
- Slow queries
- Higher memory usage
- An increased number of object storage objects
- The Loki log shows "too many streams"
- Exceeding the maximum active stream limit

### 27.2 Check labels

Use the following command:

curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Be cautious of the following fields:

trace_id
request_id
user_id
order_id
session_id
client_ip
full_url
error_message

### 27.3 Check the number of label values

Run this command:

curl -s http://127.0.0.1:3100/loki/api/v1/label/trace_id/values | jq '.data | length'

### 27.4 Check the number of series

Use the following command:

curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match={"namespace="app-demo"}' \
      | jq '.data | length'

### 27.5 Handling Steps

1. Immediately stop generating labels with high基数.
2. Modify the Alloy relabel configuration.
3. Stop using trace_id / request_id as labels.
4. Keep dynamic fields in the log text.
5. Use the | json command to query trace_id.
6. Wait for the old data retention period to expire.
7. Temporarily reduce the retention of related data if needed.
8. Increase the max_streams limit to prevent further issues.

---

## Chapter 28: Fault 24: Excessive Alloy Resources

### 28.1 Symptoms

- High CPU usage in Alloy
- High memory consumption in Alloy
- Reboots of Alloy
- Failed Alloy requests
- Delayed log collection on nodes
- Significant delay in logging within Loki

### 28.2 Common Causes

- Excessive collection scope
- A sudden increase in the amount of logs
- Heavy regular expression processing
- Complex data masking rules
- Complexity in handling multi-line logs
- Slow output-side Loki
- Accumulation of 429 retry requests
- Unreasonable batch configuration
- Insufficient resource limits

### 28.3 Check Alloy Pods

Use the following commands:

kubectl get pod -n logging -o wide | grep alloy

kubectl describe pod <alloy-pod-name> -n logging

kubectl logs <alloy-pod-name> -n logging --tail=300

If there is a metrics-server, use:

kubectl top pod -n logging | grep alloy

### 28.4 Handling Steps

1. Limit the collection namespace.
2. Filter out health check and debug logs.
3. Simplify complex regular expressions.
4. Increase Alloy resource allocation.
5. Verify if Loki writing is throttled.
6. Check network and Gateway settings.
7. Prevent duplicate log collections.

---

## Chapter 29: Fault 25: Rapid Growth in MinIO Capacity

### 29.1 Symptoms

- Rapid increase in MinIO bucket capacity
- Insufficient disk space
- Slow Loki queries
- Ineffective retention settings
- An extremely large number of objects

### 29.2 Identify the source of high log volume

Top namespaces:

topk(10,
      sum by (namespace) (
        count_over_time({namespace=~".+"}[5m])
      )
    )

Top apps:

topk(10,
      sum### Changes in Chart Fields
Changes in deploymentMode
Changes in storage configuration
Schema compatibility issues
Changes in compactor settings
Changes in ruler configurations
Risks associated with the deprecation of Simple Scalable
Whether adjustments are needed for Grafana Data Source
Whether there have been changes to the Alloy writing address

---

## Section 31: Priority Levels for Loki Fault Emergency Handling

### 31.1 P0: Complete Unavailability of the Logging Platform

Manifestations:

    All Grafana queries fail.
    The Loki /ready status is unavailable.
    Numerous Alloy write attempts fail.
    A large number of Loki Pods experience CrashLoops.
    Object storage becomes unavailable.

Priority Actions:

    1. Restore the Loki Pod to a ready state.
    2. Reestablish the Gateway functionality.
    3. Restore object storage access.
    4. Perform a helm rollback if necessary.
    5. Temporarily suspend log sources with high volumes.
    6. Inform relevant departments that log query capabilities are affected.

### 31.2 P1: Write Operations Fail, but Historical Logs Are Still Accessible

Manifestations:

    Historical logs can be retrieved.
    New logs are not being stored in Loki.
    Alloy encounters failures when attempting to send batches.

Priority Actions:

    1. Investigate Alloy errors.
    2. Check Gateway push processes.
    3. Examine write/ingester operations.
    4. Verify if there are any 429 responses.
    5. Check MinIO writing processes.
    6. Assess whether rate limits have been exceeded.

### 31.3 P2: Slow Queries or Partial Query Failures

Manifestations:

    Simple queries function normally.
    Large Dashboards perform slowly.
    query_range operations result in timeouts.

Priority Actions:

    1. Limit the time range for Dashboards.
    2. Prohibit broad All-type queries.
    3. Optimize LogQL queries.
    4. Expand read capacity if needed.
    5. Verify object storage performance.
    6. Check for cases involving high cardinality data.

### 31.4 P3: Abnormal Alert Settings

Manifestations:

    Logs can be retrieved, but Ruler alerts are not triggered.
    AlertManager does not receive any notifications.

Priority Actions:

    1. Verify the expression used in alerts.
    2. Check the rules API configuration.
    3. Review Ruler log records.
    4. Examine the AlertManager settings.
    5. Verify route and silence settings.

---

## Section 32: Production Troubleshooting Checklist

### 32.1 Collection Layer Checklist

    [ ] kubectl logs yields output for the application.
    [ ] The Alloy Pod is running.
    [ ] The Alloy DaemonSet covers the target nodes.
    [ ] Alloy RBAC grants access to pods/log data.
    [ ] Alloy configuration has been successfully loaded.
    [ ] Alloy does not filter out the target namespace.
    [ ] The Alloy writing URL is correct.
    [ ] There are no failed attempts to send log batches with Alloy.
    [ ] No duplicate log collections are occurring.

### 32.2 Writing Layer Checklist

    [ ] The Loki Gateway is in a ready state.
    [ ] The /loki/api/v1/push path is correct.
    [ ] Manual push operations were successful.
    [ ] There are no 400/429/500 errors related to Loki logs.
    [ ] No rate limit exceeded issues.
    [ ] No active stream limit violations.
    [ ] No timestamp or label-related errors.
    [ ] MinIO is accessible.
    [ ] The target Bucket exists.

### 32.3 Query Layer Checklist

    [ ] The labels API is functioning correctly.
    [ ] Namespace values are available.
    [ ] App-specific values exist.
    [ ] Simple query operations within the query_range are successful.
    [] The specified query time range is correct.
    [ ] Label names and values match the expected ones.
    [ ] The Grafana Data Source is configured properly.
    [ ] Dashboard variables are not empty.
    [ ] Broad All-type queries are avoided.
    [ ] Global complex regexp/json queries are not used.

### 32.4 Storage Layer Checklist

    [ ] The MinIO Pod is running.
    [ ] The MinIO Service Endpoint is functioning correctly.
    [ ] Loki can access MinIO data.
    [ ] The target Bucket exists.
    [ ] AccessKey/SecretKey values are correct.
    [ ] The s3ForcePathStyle setting is appropriate.
    [ ] HTTP/HTTPS configurations are correct.
    [ ] MinIO has sufficient storage capacity.
    [ ] The Compactor is operating normally.
    [ ] Retention settings are configured correctly.

### 32.5 Alert Layer Checklist

    [ ] Ruler3. Use curl to access MinIO from the Loki Namespace.
4. Check the bucket using mc.
5. Verify the access key and secret key.
6. Examine the path-style, insecure settings, and endpoint configuration.
7. Confirm the capacity of MinIO.
8. Verify the object storage permissions.
9. After repairs, perform push/query verification.

### 34.5 Loki Alarms Not Triggering the Runbook

Trigger conditions:

    Expected log alerts are not appearing in the AlertManager.

Action steps:

1. Manually execute the rule expression.
2. Ensure that the return value of the expression exceeds the threshold.
3. Verify that the specified time period has been met.
4. Query /loki/api/v1/rules.
5. Check the Ruler logs.
6. Test the AlertManager /-/ready endpoint.
7. Verify the alertmanager_url configuration.
8. Review the AlertManager UI.
9. Check the silence and route settings.
10. Use a webhook demo to verify notification functionality.

---

## Task 35: Practical Exercises

### 35.1 Task 1: Complete End-to-End Health Checks

Execution:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

    kubectl get endpoints -n logging

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Acceptance criteria:

    [ ] The Loki Pod is running.
    [ ] The Gateway Service is functioning properly.
    [ ] There are endpoints available.
    [ ] The /ready endpoint returns "ready".
    [ ] The labels API is accessible.

### 35.2 Task 2: Verify Application Logs Are Sent to Loki

Execution:

    kubectl get pod -n app-demo -o wide

    kubectl logs <pod-name> -n app-demo --tail=50

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=20' | jq

Acceptance criteria:

    [ ] Logs can be retrieved using kubectl logs.
    [ ] Loki can access the app-demo logs.

### 35.3 Task 3: Verify Manual Data Push

Execution:

    TS=$(date +%s%N)

    curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
      -H "Content-Type: application/json" \
      -d "{
        \"streams\": [
          {
            \"stream\": {
              \"job\": \"manual-troubleshooting-test\",
              \"namespace\": \"app-demo\"",
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

Acceptance criteria:

    [ ] The data push was successful.
    [ ] The query function worked correctly.

### 35.4 Task 4: Troubleshoot Empty Query Results

Execution of an invalid query:

    {namespace="not-exist"}

Subsequent troubleshooting steps:

    curl -s http://127.0.0.1:3100/loki/api/v1/label/namespace/values | jq

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

Acceptance criteria:

    [ ] The wrong namespace can be identified through label values.
    [ ] Correct queries can be executed successfully.

### 35.5 Task 5: Identify High-Base Number Risks

Execution:

    curl -s http://127.0.0.1:3100/loki/api/v1/labels | jq

Check for the presence of:

    trace_id
    request_id
    user_id
    order_id
    session_id

Acceptance criteria:

    [ ] High-Base Number labels can be identified.
### Checking if Grafana Queries are Correct
      ↓
### Whether the Ruler Executes the Rules
      ↓
### Whether the AlertManager Receives Alarms

### Core Troubleshooting Methods:

1. First, verify `kubectl logs`.
2. Then, verify Alloy.
3. Next, check if Loki is in the `/ready` state.
4. Verify the manual log push process.
5. Check the labels used for logging.
6. Verify the `query_range` settings.
7. Examine Grafana's display of data.
8. Finally, confirm that the Ruler and AlertManager are functioning correctly.

### Directions for Common Issues:

- **No logs in `kubectl logs`:** Possible issues with the application or the way logs are being outputted.
- **Logs in `kubectl logs` but no logs in Loki:** Problems with Alloy's data collection or writing process.
- **Manual push fails:** Issues related to Loki Gateway, write operations, limits, or storage.
- **Manual push is successful but business logs are missing:** Configuration issues with Alloy or the scope of log collection.
- **Labels exist but queries yield no results:** Problems with the time range, label values, or LogQL queries.
- **Grafana shows nothing but `curl` returns normal results:** Issues with Data Sources, variables, time ranges, or Dashboards.
- **Loki responds with 429 status code:** Issues related to throttling, log volume, high cardinality, or duplicate data collection.
- **Loki responds with 500 status code:** Internal component failures, storage issues, ring problems, PVC issues, or configuration errors.
- **Retention settings are not effective:** Problems with the compactor, retention policies, `delete_request_store`, or object storage permissions.
- **Alarms do not get triggered:** Issues with expressions, thresholds, routing rules for Ruler and AlertManager.

### In Production, It's More Important to Have a Troubleshooting Process Than to Memorize All Commands:

1. **Identify the level of the issue.**
2. **Choose the simplest command to verify.**
3. **Start with basic queries.**
4. **Gradually narrow down the possible causes.**
5. **Only consider changing configurations or scaling up as a last resort.

### Next Topic:

**15 - Comprehensive Loki Practice: Building a Complete K8S Log Platform from Scratch**

### Key Topics to Learn:

- MinIO
- Loki
- Alloy
- Grafana
- Dashboards
- Ruler
- AlertManager
- Retention settings
- Limits
- Runbooks
- A comprehensive practical guide for building a log platform.

---

## Reference Documents

- **Grafana and Loki Documentation:**  
  https://grafana.com/docs/loki/latest/

- **Troubleshooting Loki Operations:**  
  https://grafana.com/docs/loki/latest/operations/troubleshooting/troubleshoot-operations/

- **Troubleshooting Log Ingestion and Writing:**  
  https://grafana.com/docs/loki/latest/operations/troubleshooting/troubleshoot-ingest/

- **Troubleshooting Log Queries:**  
  https://grafana.com/docs/loki/latest/query/troubleshoot-query/

- **Loki HTTP API:**  
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- **Request Validation and Rate Limits:**  
  https://grafana.com/docs/loki/latest/operations/request-validation-rate-limits/

- **Log Retention:**  
  https://grafana.com/docs/loki/latest/operations/storage/retention/

- **Loki Configuration:**  
  https://grafana.com/docs/loki/latest/configure/

- **Querying Loki:**  
  https://grafana.com/docs/loki/latest/query/

- **Best Practices for Queries:**  
  https://grafana.com/docs/loki/latest/query/bp-query/

- **Grafana and Alloy Documentation:**  
  https://grafana.com/docs/alloy/latest/

- **Loki Source for Kubernetes:**  
  https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/

- **Collecting Kubernetes Logs and Forwarding Them to Loki:**  
  https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/

- **Grafana and Loki Data Source:**  
  https://grafana.com/docs/grafana/latest/datasources/loki/

- **AlertManager Configuration:**  
  https://prometheus.io/docs/alerting/latest/configuration/

- **Kubernetes Logging Architecture:**  
  https://kubernetes.io/docs/concepts/cluster-administration/logging/

- **Kubernetes `kubectl logs` Command:**  
  https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/

- **Kubernetes Services:**  
  https://kubernetes.io/docs/concepts/services-networking/service/

- **Kubernetes NetworkPolicy:**  
  https