# 07-Loki Label Design and High Cardinality Issues Experiment

## Documentation Overview

This document is the seventh installment in the dedicated study on Loki, focusing on systematically understanding Loki’s label design principles, Stream concepts, high cardinality issues, and how to observe the impact of incorrect label designs on Loki’s writing, querying, and storage capabilities through experiments.

Previous tasks have been completed:

    01-Loki Basic Understanding and Experimental Environment Setup
    02-Loki Architecture Principles and Component Responsibilities in Practice
    03-Loki Deployment Modes Comparison and Experimental Selection
    04-Loki Single-Node Mode Helm Deployment in Action
    05-Loki Object Storage Integration with MinIO in Practice
    06-Grafana-Alloy for Collecting Kubernetes Pod Logs in Practice

In the previous chapter, we used Grafana Alloy to collect Kubernetes Pod logs and were able to query them within Loki:

    {namespace="app-demo"}
    {namespace="app-demo", pod=~"nginx-demo-.*"}
    {namespace="app-demo", container="nginx"}

This document delves into one of Loki’s core capabilities:

    Label Design

Loki’s query efficiency, writing performance, storage costs, and alert maintainability depend significantly on whether the label design is appropriate.

This chapter addresses the following key questions:

- What are Loki Labels?
- What are Loki Streams?
- What is the difference between Labels and log content?
- What is Cardinality?
- Why can high cardinality affect Loki’s performance?
- Which fields are suitable for use as Labels?
- Which fields are not suitable for use as Labels?
- Are namespace/pod/container/node/app/team suitable as Labels?
- Why are request_id/trace_id/user_id/order_id unsuitable as Labels?
- How to view the current Labels in Loki?
- How to determine how many values a Label has?
- How to observe the problems caused by high cardinality fields through experiments?
- How to place high-cardinality fields in the log content or structured fields rather than Labels?
- How to establish Loki label guidelines for production environments.

---

## Tags

#Loki #LogQL #Label #Stream #Cardinality #High Cardinality #GrafanaAlloy #Kubernetes #Pod Logs #SRE #Log Governance #Observability

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/07-Loki Label Design and High Cardinality Issues Experiment.md

---

## I. Experimental Objectives

After completing this document, you should be able to:

    1. Understand the role of Loki Labels.
    2. Comprehend the meaning of Loki Streams.
    3. Grasp what Cardinality is.
    4. Recognize why high cardinality can impact Loki’s performance.
    5. Identify which fields are suitable for use as Labels.
    6. Determine which fields are not suitable for use as Labels.
    7. Use the Loki API to view current Labels.
    8. Query logs using different Label combinations via LogQL.
    9. Simulate the risks of using request_id/user_id as Labels through experiments.
    10. Adjust Alloy’s relabel configuration to control Labels written to Loki.
    11. Design a recommended set of Labels for Kubernetes Pod logs.
    12. Establish production environment Loki label guidelines.

---

## II. Experimental Environment

### 2.1 Kubernetes Cluster

Experiment nodes:

    k8s-master      10.0.0.20
    k8s-worker01    10.0.0.21
    k8s-worker02    10.0.0.22

Namespaces:

    logging
    app-demo
    monitoring
    minio

Deployed components:

    Loki
    Loki Gateway
    Grafana Alloy
    MinIO
    nginx-demo test application

### 2.2 Prerequisites

The following must be completed in advance:

    [ ] Loki is ready for use.
    [ ] The Alloy DaemonSet is running.
    [ ] Alloy can collect app-demo logs.
    [ ] Loki can query app-demo logs.
    [ ] There is a nginx-demo test application in app-demo.

Verify Loki:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

    curl -s http://127.0.0.1:3100/ready

Expected result:

    ready

Verify logs:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

---

## III. What are Loki Labels

### 3.1 Basic Concepts of Labels    {namespace="app-demo", app="api", request_id="req-000001"}
    {namespace="app-demo", app="api", request_id="req-000002"}
    {namespace="app-demo", app="api", request_id="req-000003"}

Each request has a unique request_id.

Result:

    Each request may result in a new Stream.

If there are 1 million requests per day:

    Nearly 1 million Streams may be generated.

This is a typical example of a high cardinality issue.```bash
curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match={namespace="app-demo"}' | jq

### 9.2 Count the number of series

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match={namespace="app-demo"}' \
      | jq '.data | length'

### 9.3 View the series for nginx-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match={namespace="app-demo", app="nginx-demo"}' \
      | jq

### 9.4 Understand the results

If there are two replicas of nginx-demo, you may see two or more Streams.

This is because:

    The pod names are different.
    The container names are the same.
    The namespace is the same.
    The app is the same.

If each Pod has unique pod Labels, it usually results in at least one independent Stream for each Pod.

---

## Section 10: Experiment 2: Generate more logs and observe Label queries

### 10.1 Access Nginx to generate logs

    kubectl run curl-test \
      --rm -it \
      --image=curlimages/curl:8.5.0 \
      -n app-demo \
      -- sh

Inside the container, execute the following command:

    for i in $(seq 1 20); do
      curl -s http://nginx-demo.app-demo.svc.cluster.local/path-${i} >/dev/null
    done

    exit

### 10.2 Query logs

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "path-"' \
      --data-urlencode 'limit=20' | jq

### 10.3 Query for 404 errors

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="nginx-demo"} |= "404"' \
      --data-urlencode 'limit=20' | jq

### 10.4 Observations and conclusions

This approach to querying is reasonable:

    Labels are used to define specific ranges.
    The log content itself is used for searching dynamic information.

For example, paths like "path-1", "path-2", and "path-3" should not be used as Labels. Instead, they should be included in the log content for search purposes.

---

## Section 11: Experiment 3: Simulate correct structured logs

Nginx access logs are not suitable for simulating business JSON logs. Here, we will create a simple Job to produce structured logs.

### 11.1 Create a JSON log Job

Create a file named `json-log-demo.yaml` with the following content:

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: json-log-demo
      namespace: app-demo
      labels:
        app: json-log-demo
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
                  while [ $i -le 50 ]; do
                    echo "{\"level\":\"info\",\"service\":\"json-log-demo\",\"msg\":\"request success\",\"status\":200,\"duration_ms\":$((RANDOM % 200)),\"trace_id\":\"trace-$i\",\"user_id\":\"user-$i\"}"
                    echo "{\"level\":\"error\",\"service\":\"json-log-demo\",\"msg\":\"database connection failed\",\"status\":500,\"duration_ms\":$((1000 + RANDOM % 2000)),\"trace_id\":\"trace-error-$i\",\"user_id\":\"user-error-$i\"}"
                    i=$((i+1))
                    sleep 1
                  done
    apply:

    kubectl apply -f json-log-demo.yaml

To check the job, run:

    kubectl get job -n app-demo

   ```json
{
  "job": "bad-label-demo",
  "namespace": "app-demo",
  "app": "bad-label-demo",
  "trace_id": "trace-${i}\"
},
{
  "values": [
    ["${TS}", "bad label demo log line trace-${i}"]
  ]
}
]
>>>/dev/null
sleep 0.1
done

### 12.4 Check if the trace_id Label appears

    curl -s "http://127.0.0.1:3100/loki/api/v1/labels" | jq

To view the values of trace_id:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq

To count the number of values:

    curl -s "http://127.0.0.1:3100/loki/api/v1/label/trace_id/values" | jq '.data | length'

### 12.5 Check the number of series for bad-label-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match={"job="bad-label-demo"}' \
      | jq '.data | length'

Expected result:

    Close to 100

Reason:

    Each trace_id generates a different set of labels.
    Each different label set corresponds to a different stream.

### 12.6 Compare with the correct method

Correct method: Include the trace_id in the log text.

Manual push:

    for i in $(seq 1 100); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"good-label-demo\",
                \"namespace\": \"app-demo\"",
                \"app\": \"good-label-demo\"
              },
              \"values\": [
                [\"${TS}\", 「\\\"level\\\":\\\"info\\\",\\\"trace_id\\\":\\\"trace-${i}\\\",\\\"msg\\\":\\\"good label demo log line\\\"}\"]
              ]
            }
          ]
        }" >/dev/null
    done

To count the number of series:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/series" \
      --data-urlencode 'match={"job="good-label-demo"}' \
      | jq '.data | length'

Expected result:

    Usually much smaller than for bad-label-demo.

To query a specific trace_id:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="good-label-demo"} | json | trace_id="trace-50"' \
      --data-urlencode 'limit=10' | jq
```

### 12.7 Experimental Conclusion

Wrong method:

    Using the trace_id as a Label
    ↓
    100 trace IDs
    ↓
    Nearly 100 Streams

Correct method:

    Including the trace_id in the log text
    ↓
    The Label remains stable
    ↓
    Use | json to filter by trace_id during queries

Conclusion:

    Dynamic fields should not be used as Labels.
    They should be included in the log text or structured fields instead.
```

---

## Chapter Thirteen: Experiment Five: Investigating the Impact of High Baseline Numbers on Query Methods

### 13.1 Querying bad-label-demo

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="bad-label-demo"}' \
      --data-urlencode 'limit=10' | jq

### 13.2 Querying a Specific Trace ID

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="bad-label-demo", trace_id="trace-50"}' \
      --data-urlencode 'limit=10' | jq

Querying a specific trace ID seems fast.

However, the downside is:

    Loki creates a separate Stream for each trace ID.
    This increases the cost of writing and indexing data,      action        = "replace"
      target_label  = "team"
    }

      rule {
        source_labels = ["__meta_kubernetes_pod_label_environment"]
        action        = "replace"
        target_label  = "environment"
      }
    }

These labels are suitable for learning environments.

For production environments, scale assessment must also be taken into account.

---

## Chapter Fifteen: How to Remove Inappropriate Labels

### 15.1 Avoid Extracting High-Frequency Fields in Relabeling

Do not extract the following from log content as labels:

    trace_id
    request_id
    user_id
    order_id

### 15.2 If Incorrect Labels Already Exist

If incorrect labels have been generated in the Alloy configuration, the corresponding relabeling rules need to be removed.

For example, an incorrectly configured rule:

    rule {
      source_labels = ["__meta_kubernetes_pod_annotation_trace_id"]
      action        = "replace"
      target_label  = "trace_id"
    }

should be deleted or changed so that it is no longer used as a label.

### 15.3 Do Old Data Disappear Immediately After Labels Are Removed?

No.

Old log data that has already been written to Loki will still retain the old labels until they exceed the retention period and are cleaned up.

Therefore, in production environments:

    Label specifications should be designed as early as possible.
    Incorrect labels can cause issues for some time.
    Correcting them only prevents new data from generating incorrect labels.

---

## Chapter Sixteen: Recommendations for Production Label Specifications

### 16.1 Basic Specifications

Standard labels are recommended:

    cluster
    environment
    namespace
    app
    component
    container
    team
    workload

Optional labels:

    pod
    node
    version
    region
    zone

Labels to use with caution:

    pod
    node
    version

Labels prohibited:

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

### 16.2 Namespace Specifications

Namespace is a field that must be retained.

It is used for:

    Isolation between multiple teams
    Controlling the scope of queries
    Dashboard variables
    Alarm routing
    Cost statistics

Example query:

    {namespace="app-prod"}

### 16.3 App Specifications

It is highly recommended to retain the app field.

Requirement:

    All business pods must have either the app label or the app.kubernetes.io/name label.

Recommended:

    app.kubernetes.io/name

Compatible:

    app

Example:

    labels:
      app.kubernetes.io/name: order-api
      app: order-api

### 16.4 Team Specifications

The team field is used for alarm routing and assigning responsibility.

Example:

    labels:
      team: sre

Query:

    {team="sre"}

Alarm routing:

    team=sre → Notify the SRE team
    team=payment → Notify the payment team

### 16.5 Environment Specifications

The environment field is used to distinguish between different environments.

Recommended values:

    lab
    dev
    test
    staging
    prod

Avoid using confusing values such as:

    production
    Prod
    PROD
    pro
    prd

It should be unified as:

    prod

### 16.6 Version Specifications

The version field can be used for release and troubleshooting purposes.

For example:

    version="v1.2.3"

However, use it with caution:

    If releases are frequent, the number of version values will increase.
    In large-scale environments, consider whether to use it as a label.

---

## Chapter Seventeen: Labels and Dashboard Variables

### 17.1 Commonly Used Variables in Grafana

Recommended variables:

    cluster
    environment
    namespace
    app
    pod
    container
    node
    team

### 17.2 Namespace Variables

Query:

    label_values(namespace)

### 17.3 App Variables

Query:

    label_values({namespace="$namespace"}, app)

### 17.4 Pod Variables

Query:

    label_values({namespace="$namespace", app="$app"}, pod)

### 17.5 Container Variables

Query:

    label_values({namespace="$namespace", pod="$pod"}, container)

### 17.6 Dashboard Design Principles

Do not allow users to start their queries with:

    {namespace=~".+"}

Instead, guide them to:

    First select the cluster
    Then select the namespace
    Next select the app
    Finally, select the pod

This can reduce unnecessary wide-range queries.

---

## Chapter Eighteen: Labels and Alarm Routing

### 18.1 Team Must Be Included in Alarms

Log alarms are recommended to include:

    team
    namespace
error_type="database_error"

The complete error_message is retained in the log text.

### 21.5 Can status_code be used as a Label?

Status_code has limited possible values, for example:

    200
    400
    401
    403
    404
    500
    502
    503

Theoretically, these have a low cardinality.

However, caution is advised.

In access log scenarios, it can also be queried through JSON fields or log content:

    {app="api"} | json | status=500

If it is frequently used in Dashboards and alerts, considering extracting it as a Label is an option.

Principle:

    Low cardinality + High-frequency queries + Clear governance value

are the criteria for using something as a Label.```bash
echo '{"app": "bad-label-demo", "trace_id": "trace-${i}\"}' | curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"bad-label-demo\",
                \"namespace\": \"app-demo\"",
                \"app\": \"bad-label-demo\"
              },
              \"values\": [
                [\"${TS}\", \"bad label demo log line trace-${i}\"]
              ]
            }
          ]
        }" >/dev/null
sleep 0.1
done

Acceptance:

    [ ] The trace_id is used as a Label.
    [ ] The number of values for trace_id is close to 100.
    [ ] The number of bad-label-demo series has increased significantly.

### Task 25.6: Simulate good-label-demo

Execution:

    for i in $(seq 1 100); do
      TS=$(date +%s%N)
      curl -s -X POST "http://127.0.0.1:3100/loki/api/v1/push" \
        -H "Content-Type: application/json" \
        -d "{
          \"streams\": [
            {
              \"stream\": {
                \"job\": \"good-label-demo\"",
                \"namespace\": \"app-demo\",
                \"app\": \"good-label-demo\"
              },
              \"values\": [
                [\"${TS}\", 「\\\"level\\\":\\\"info\\\",\\\"trace_id\\\":\\\"trace-${i}\\\",\\\"msg\\\":\\\"good label demo log line\\\"}\"]
              ]
            }
          ]
        }" >/dev/null
      sleep 0.1
    done

Query:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={job="good-label-demo"} | json | trace_id="trace-50"' \
      --data-urlencode 'limit=10' | jq

Acceptance:

    [ ] The trace_id is not used as a Label.
    [ ] The trace_id can be queried using JSON parsing.
    [ ] The number of good-label-demo series is significantly lower than that of bad-label-demo.

---

## Chapter 26: Acceptance Checklist

After completing this document, you should confirm the following:

    [ ] You understand what a Label is.
    [ ] You understand what a Stream is.
    [ ] You understand what Cardinality is.
    [ ] You understand why high Cardinality is dangerous.
    [ ] You can identify recommended Labels.
    [ ] You can identify prohibited Labels.
    [ ] You can view Label names through the API.
    [ ] You can view Label values through the API.
    [ ] You can view the number of Streams using the series API.
    [ ] You can query logs using namespace.
    [ ] You can query logs using app.
    [ ] You can query logs using pod.
    [ ] You can deploy a JSON log demo.
    [ ] You can query structured logs using | json.
    [ ] You can simulate the high Cardinality issue of using trace_id as a Label.
    [ ] You can compare bad-label-demo and good-label-demo.
    [ ] You can explain why request_id, trace_id, and user_id are not suitable as Labels.
    [ ] You can write production-grade Loki Label specifications.
    [ ] You understand that old data will not disappear immediately after modifying incorrect Labels.

---

## Chapter 27: Common Misconceptions

### Misconception 1: More Labels Mean Better

Wrong.

The more Labels there are, the more combinations there are, which may result in more Streams.

Loki is not suitable for using all fields as Labels.

### Misconception 2: Fields That Are Easy to Query Should Be Used as Labels

Wrong.

Using request_id as a Label for querying is convenient, but it comes at a high cost.

The correct approach is:

    Use stable Labels to narrow down the scope.
    Use the log text or JSON fields to filter dynamic fields.

### Misconception 3: Fields with Low Cardinality Are Always Safe

Not necessarily.

For example, there may be many pods in a very large cluster.

It is necessary to evaluate based on scale and query value.

### Misconception 4: Old Labels Disappear Immediately After Deleting Relabel Rules

Wrong.

Old data still exists.

It is necessary to wait for the retention period to expire or execute a deletion strategy.

### Misconception 5: error_message Can Be Used as a Label

Wrong.

errorhttps://grafana.com/docs/alloy/latest/reference/components/loki/loki.process/

- Recommended Tags for Kubernetes:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/