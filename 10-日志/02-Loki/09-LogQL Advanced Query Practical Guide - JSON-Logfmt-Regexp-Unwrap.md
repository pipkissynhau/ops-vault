# 09-LogQL Advanced Query Practice: json-logfmt-regexp-unwrap

## Document Overview

This is the ninth article in the Loki specialized learning series, aimed at systematically mastering advanced LogQL query capabilities. The focus is on JSON log parsing, logfmt log parsing, field extraction using regexp, line_format for formatted output, label_format for temporary label handling, unwrap for numeric field conversion, and generating metric queries from logs.

Previously completed:

    01-Loki Basic Understanding and Experiment Environment Planning
    02-Loki Architecture Principles and Component Responsibilities Practical Observation
    03-Loki Deployment Mode Comparison and Experiment Selection
    04-Loki Single-Node Helm Deployment Practice
    05-Loki Object Storage Integration with MinIO Practice
    06-Grafana-Alloy Collection of K8S-Pod Logs Practice
    07-Loki Label Design and High Cardinality Problem Experiment
    08-LogQL Basic Query Practice: Namespace-Pod-Container Log Retrieval

The 08th article focused on basic queries:

    {namespace="app-demo"}
    {namespace="app-demo", app="nginx-demo"}
    {namespace="app-demo"} |= "ERROR"
    {namespace="app-demo"} |~ "(?i)error|exception|panic"

This article enters advanced query territory, focusing on solving the following issues:

- How to parse JSON logs;
- How to parse logfmt logs;
- How to extract fields from plain text logs using regexp;
- How to filter status, level, duration_ms by fields;
- How to handle JSON parsing errors;
- How to use __error__ to filter parsing failures;
- How to use line_format to optimize log display;
- How to use label_format to generate temporary labels;
- How to use unwrap to convert log fields to numeric values;
- How to calculate average, maximum, and P95 of duration_ms;
- How to count ERROR log entries;
- How to count 5xx log entries;
- How to group statistics by app, pod, and status;
- How to generate alertable metric queries from logs;
- How to avoid advanced queries slowing down Loki.

This article is suitable for those who have already mastered basic LogQL queries.

---

## Tags

#Loki #LogQL #JsonLog #logfmt #regexp #unwrap #line_format #label_format #Grafana #Kubernetes #PodLog #SRE #Observation #LogAnalysis

---

## Recommended Path

Recommended path:

    10-Logs/02-Loki/09-LogQL Advanced Query Practice: json-logfmt-regexp-unwrap.md

---

## One: Experiment Objectives

After completing this article, you should be able to:

    1. Understand the execution method of LogQL pipeline.
    2. Use | json to parse JSON logs.
    3. Use | logfmt to parse logfmt logs.
    4. Use | regexp to extract fields from plain text logs.
    5. Use field filtering for level="error".
    6. Use field filtering for status >= 500.
    7. Use field filtering for duration_ms > 1000.
    8. Understand the purpose of __error__.
    9. Filter parsing errors like JSONParserErr.
    10. Use line_format to change query result display.
    11. Use label_format to temporarily generate or adjust labels.
    12. Use unwrap to convert fields to numeric series.
    13. Use count_over_time to count log entries.
    14. Use rate to count log rate.
    15. Use avg_over_time to calculate average duration.
    16. Use max_over_time to calculate maximum duration.
    17. Use quantile_over_time to calculate P95 / P99.
    18. Convert log queries to metric queries.
    19. Write LogQL suitable for Grafana Dashboard.
    20. Understand performance considerations for advanced LogQL.

---

## Two: Experiment Environment

### 2.1 Kubernetes Cluster

Experimental nodes:

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
    nginx-demo
    json-log-demo
    logfmt-log-demo

### 2.2 Prerequisites

Confirm the following:

    [ ] Loki is running normally
    [ ] Loki Gateway is accessible
    [ ] Alloy is collecting Pod logs normally
    [ ] app-demo Namespace exists
    [ ] Can query app-demo logs via LogQL
    [ ] Already mastered basic queries, line filters, and label selectors

Check Loki:

    kubectl get pods -n logging -o wide

    kubectl get svc -n logging

Port forwarding:

    kubectl port-forward svc/loki-gateway 3100:80 -n logging

Verification:

    curl -s http://127.0.0.1:3100/ready

Expected:

    ready

Check basic logs:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo"}' \
      --data-urlencode 'limit=10' | jq

---

## Three: LogQL Pipeline Basics

### 3.1 What is a Pipeline

LogQL queries can be divided into two parts:

    Stream Selector
      +
    Pipeline

Example:

    {namespace="app-demo", app="json-log-demo"} | json | level="error"

Where:

# 3.2 Pipeline Execution from Left to Right

LogQL executes pipelines from left to right.

Example:

    {namespace="app-demo", app="json-log-demo"}
      |= "database"
      | json
      | level="error"

Execution order:

    1. First find the log stream for app-demo/json-log-demo via Label.
    2. Then retain only log lines containing "database".
    3. Then parse the remaining logs as JSON.
    4. Then filter logs with level="error".

This is more efficient than the following query:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | level="error"
      |= "database"

Reason:

    String filtering earlier reduces the amount of logs to parse later.
    The query uses fewer resources.

# 3.3 Recommended Execution Order

Recommended order:

    1. Use Label Selector to narrow log streams.
    2. Use Line Filter to narrow log lines.
    3. Use Parser to parse fields.
    4. Use Field Filter to filter fields.
    5. Use line_format / label_format to adjust display.
    6. Use unwrap to convert numeric fields.
    7. Perform aggregation statistics.

Example:

    {namespace="app-demo", app="json-log-demo"}
      |= "duration_ms"
      | json
      | status >= 500
      | unwrap duration_ms
      | __error__=""
      [5m]

---

## Four. Preparing JSON Log Demo

### 4.1 Creating JSON Log Job

Create file:

    json-log-demo.yaml

Content:

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
                  while [ $i -le 120 ]; do
                    echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"info\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/orders\",\"method\":\"GET\",\"status\":200,\"duration_ms\":$((20 + RANDOM % 200)),\"trace_id\":\"trace-$i\",\"user_id\":\"user-$i\",\"msg\":\"request success\"}"
                    echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"warn\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/orders\",\"method\":\"GET\",\"status\":404,\"duration_ms\":$((50 + RANDOM % 400)),\"trace_id\":\"trace-warn-$i\",\"user_id\":\"user-warn-$i\",\"msg\":\"resource not found\"}"
                    echo "{\"timestamp\":\"$(date -Iseconds)\",\"level\":\"error\",\"service\":\"json-log-demo\",\"path\":\"/api/v1/payment\",\"method\":\"POST\",\"status\":500,\"duration_ms\":$((1000 + RANDOM % 3000)),\"trace_id\":\"trace-error-$i\",\"user_id\":\"user-error-$i\",\"msg\":\"database connection failed\"}"
                    echo "this is not a json line"
                    i=$((i+1))
                    sleep 1
                  done

Apply:

    kubectl apply -f json-log-demo.yaml

Check:

    kubectl get job -n app-demo

    kubectl get pod -n app-demo | grep json-log-demo

Check logs: /think

kubectl logs <json-log-demo-pod> -n app-demo --tail=20

### 4.2 Explanation

This Job outputs four types of logs:

    info:
        status=200
        duration_ms is low
        msg=request success

    warn:
        status=404
        duration_ms is moderate
        msg=resource not found

    error:
        status=500
        duration_ms is high
        msg=database connection failed

    Non-JSON lines:
        this is not a json line

Non-JSON lines are used for later demonstration:

    __error__
    JSONParserErr

---

## FiveI don't know.Using JSON to Parse Logs

### 5.1 Querying Raw JSON Logs

LogQL:

    {namespace="app-demo", app="json-log-demo"}

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"}' \
      --data-urlencode 'limit=20' | jq

### 5.2 Using JSON Parsing

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json' \
      --data-urlencode 'limit=20' | jq

### 5.3 Querying level=error

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | level="error"

API:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | level="error"' \
      --data-urlencode 'limit=20' | jq

### 5.4 Querying status=500

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | status=500

### 5.5 Querying status>=500

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | status >= 500

### 5.6 Querying duration_ms>1000

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | duration_ms > 1000

### 5.7 Querying specified path

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | path="/api/v1/payment"

### 5.8 Querying specified trace_id

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | trace_id="trace-error-10"

Explanation:

    trace_id should not be used as a Loki Label.
    But it can be queried as a JSON field.
    This is the recommended usage.

---

## SixI don't know.Handling JSON Parsing Errors

### 6.1 Why There Is __error__

When a stage in the LogQL pipeline fails to parse, Loki does not discard the log line but adds a system label:

    __error__

For example:

    this is not a json line

Executing:

    | json

may result in:

    __error__="JSONParserErr"

### 6.2 Viewing Parsing Error Logs

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | __error__ != ""

Meaning:

    Only view log lines where errors occurred in the pipeline.

### 6.3 Filtering Out JSON Parsing Errors

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | __error__=""

Meaning:

    Only retain log lines that parsed successfully.

### 6.4 Filtering Specific Errors

LogQL:

    {namespace="app-demo", app="json-log-demo"} | json | __error__!="JSONParserErr"

Meaning:

    Exclude JSONParserErr.

### 6.5 Querying error Logs and Excluding Parsing Errors

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | level="error"

### 6.6 Why We Need to Handle __error__

If __error__ is not handled, it may result in:

    Mixed parsing failure logs in query results.
    Failure in unwrap stage conversion.
    Error in metric queries.
    Dashboard panel has no data or errors.
    Alert rule execution failure.

In production, it is recommended:

    Pay attention to __error__ whenever using json / logfmt / regexp / unwrap.

---

## SevenI don't know.Using line_format to Optimize Display

### 7.1 What is line_format

line_format is used to change the display format of log lines in query results.

It does not alter the original log storage, only affects the query return results.

Suitable for:

    Displaying only key fields
    Simplifying Grafana Explore output
    Making troubleshooting logs clearer
    Creating log table panels

### 7.2 Display level, status, duration, msg

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | line_format "{{.level}} status={{.status}} duration={{.duration_ms}}ms msg={{.msg}}"

Effect similar to:

    error status=500 duration=2780ms msg=database connection failed

### 7.3 Display trace_id and path

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | line_format "trace={{.trace_id}} method={{.method}} path={{.path}} status={{.status}}"

### 7.4 Query errors and format

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | level="error"
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 7.5 Notes

line_format only affects display.

It will NOT:

    Modify logs stored in Loki.
    Modify labels.
    Modify the original log body.
    Reduce storage costs for already occurred logs.

---

## Eight, Using label_format for Temporary Label Processing

### 8.1 What is label_format

label_format is used to create, modify, or format temporary labels during queries.

It is suitable for:

    Temporarily creating display fields
    Formatting fields
    Supplementing labels in query results
    Dashboard display optimization

Note:

    label_format generates temporary labels during queries.
    It is not equal to persistent Labels written to Loki during ingestion.
    It will not change the underlying storage.

### 8.2 Create temporary result labels

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | label_format result="{{.method}}_{{.status}}"

### 8.3 Combine with line_format for display

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | label_format result="{{.method}}_{{.status}}"
      | line_format "result={{.result}} path={{.path}} duration={{.duration_ms}}ms msg={{.msg}}"

### 8.4 label_format is not a solution for high cardinality

Do not use high cardinality fields as ingestion labels long-term just because label_format is convenient.

Difference:

    Ingestion phase Label:
        Affects Loki Stream, indexing, writing, and storage costs.

    Query phase label_format:
        Only affects current query results.

---

## Nine, Prepare logfmt Log Demo

### 9.1 What is logfmt

logfmt is a key=value formatted log.

Example:

    level=info service=logfmt-demo status=200 duration_ms=32 msg="request success"

Features:

    More concise than JSON.
    Easy to read.
    Easy to parse.
    Widely used by many Go services and infrastructure components.

### 9.2 Create logfmt-log-demo Job

Create file:

    logfmt-log-demo.yaml

Content: /think

apiVersion: batch/v1
kind: Job
metadata:
  name: logfmt-log-demo
  namespace: app-demo
  labels:
    app: logfmt-log-demo
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
                echo "this is not logfmt maybe"
                i=$((i+1))
                sleep 1
              done

Application:

    kubectl apply -f logfmt-log-demo.yaml

Check:

    kubectl get pod -n app-demo | grep logfmt-log-demo

    kubectl logs <logfmt-log-demo-pod> -n app-demo --tail=20

---

## Ten. Using logfmt to Parse Logs

### 10.1 Querying Raw logfmt Logs

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"}

### 10.2 Using logfmt Parsing

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt

### 10.3 Querying level=error

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | level="error"

### 10.4 Querying status=500

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | status=500

### 10.5 Querying status>=500

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | status >= 500

### 10.6 Querying duration_ms>1000

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | duration_ms > 1000

### 10.7 Querying msg

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | msg="upstream timeout"

### 10.8 Filtering Parsing Errors

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | __error__=""

### 10.9 Formatting Display

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"}
      | logfmt
      | __error__=""
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

---

## Eleven. Using regexp to Parse Plain Text Logs

### 11.1 When is regexp suitable?

regexp is suitable for parsing:

    Nginx access log
    Java plain text logs
    Python plain text logs
    Old application unstructured logs
    Logs that cannot be transformed into JSON/logfmt

Disadvantages:

    Complex regular expressions.
    Poor readability.
    Less performance-friendly than structured parsing.
    Easily breaks when log format changes.

Production recommendations:

    Prefer JSON/logfmt over regexp if possible.
    regexp is suitable for legacy systems and transitional scenarios.

### 11.2 Nginx Default Log Example

Nginx access log might look like: /think

10.244.1.10 - - [01/May/2026:12:00:00 +0000] "GET /notfound HTTP/1.1" 404 153 "-" "curl/8.5.0" "-"

### 11.3 regexp Extract method, path, status

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`

Explanation:

    ?P<method>:
        Extract the method field.

    ?P<path>:
        Extract the path field.

    ?P<status>:
        Extract the status field.

    ?P<body_bytes_sent>:
        Extract the response size.

### 11.4 Query 404

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | status="404"

### 11.5 Query 5xx

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | status=~"5.."

### 11.6 Format Display Nginx Logs

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | line_format "{{.method}} {{.path}} status={{.status}} bytes={{.body_bytes_sent}} ip={{.remote_addr}}"

### 11.7 Handling Regex Parsing Failures

If some logs don't match the regex, it may produce __error__.

Filter successfully parsed logs:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | __error__=""

View parsing failures:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | __error__!=""

---

## TwelveI don't know.unwrap Basics

### 12.1 What is unwrap

unwrap is used to convert a field in logs into a numeric sample.

For example JSON logs:

    {"duration_ms": 1200}

Execute:

    | json | unwrap duration_ms

You can convert duration_ms into a calculable numeric sequence.

Then you can use:

    avg_over_time
    max_over_time
    min_over_time
    quantile_over_time
    sum_over_time
    count_over_time

### 12.2 What Fields are Suitable for unwrap

Suitable:

    duration_ms
    latency_ms
    request_time
    response_time
    size_bytes
    status if needed for statistics
    cost
    queue_size
    retry_count
    batch_size

Not suitable:

    msg
    trace_id
    user_id
    path
    level

### 12.3 Must Pay Attention to __error__ After unwrap

If the field is not numeric, unwrap will produce pipeline error.

Therefore common writing:

    | unwrap duration_ms
    | __error__=""

Note:

    __error__="" should be placed after the stage that generates errors.

---

## ThirteenI don't know.Using unwrap to Calculate JSON Log Latency

### 13.1 Average duration_ms

LogQL:

    avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

Explanation:

    Query the average of duration_ms in the past 5 minutes.

### 13.2 Maximum duration_ms

LogQL:

    max_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 13.3 Minimum duration_ms

LogQL:

    min_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 13.4 P95 duration_ms

LogQL:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 13.5 P99 duration_ms

LogQL:

    quantile_over_time(
      0.99,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 13.6 P95 for Error Requests Only

LogQL:

quantile_over_time(
  0.95,
  {namespace="app-demo", app="json-log-demo"}
    | json
    | __error__=""
    | level="error"
    | unwrap duration_ms
    | __error__="" [5m]
)

**Explanation:**

The first `__error__=""` filters JSON parsing errors.
The second `__error__=""` filters unwrap conversion errors.

---

## FourteenI don't know.Using unwrap to Calculate logfmt Log Duration

### 14.1 Average Duration

LogQL:

    avg_over_time(
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 14.2 Maximum Duration

LogQL:

    max_over_time(
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 14.3 P95 Duration

LogQL:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 14.4 Counting Average Duration for 5xx Requests Only

LogQL:

    avg_over_time(
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | __error__=""
        | status >= 500
        | unwrap duration_ms
        | __error__="" [5m]
    )

---

## FifteenI don't know.count_over_time: Counting Log Entries

### 15.1 Counting Log Entries in app-demo Over 5 Minutes

LogQL:

    count_over_time({namespace="app-demo"}[5m])

### 15.2 Counting error Log Entries for json-log-demo

LogQL:

    count_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | __error__=""
        | level="error" [5m]
    )

### 15.3 Counting error Log Entries by app

LogQL:

    sum by (app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 15.4 Counting error Log Entries by pod

LogQL:

    sum by (pod) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 15.5 Counting JSON status>=500 Entries

LogQL:

    sum by (app) (
      count_over_time(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    )

---

## SixteenI don't know.rate: Counting Log Rate

### 16.1 Counting Log Entries Per Second

LogQL:

    rate({namespace="app-demo"}[5m])

### 16.2 Counting error Log Rate

LogQL:

    sum by (app) (
      rate(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 16.3 Counting 5xx Log Rate

LogQL:

    sum by (app) (
      rate(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    )

### 16.4 Difference Between rate and count_over_time

count_over_time:

Counts the total number of log entries in a time window.

rate:

Counts the average rate per unit time.

Example:

There were 300 error entries in the last 5 minutes:
count_over_time = 300

Average 1 error per second:
rate ≈ 1

Commonly used in alerts:

count_over_time

Commonly used in dashboard trends:

rate

---

## SeventeenI don't know.Counting by Status Code

### 17.1 Counting JSON Logs by status

LogQL:

    sum by (status) (
      count_over_time(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__="" [5m]
      )
    )

### 17.2 Counting Only 5xx

LogQL:

sum by (status) (
  count_over_time(
    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | status >= 500 [5m]
  )
)

### 17.3 logfmt Statistics by status

LogQL:

    sum by (status) (
      count_over_time(
        {namespace="app-demo", app="logfmt-log-demo"}
          | logfmt
          | __error__="" [5m]
      )
    )

### 17.4 Nginx regexp Statistics by status

LogQL:

    sum by (status) (
      count_over_time(
        {namespace="app-demo", app="nginx-demo"}
          | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
          | __error__="" [5m]
      )
    )

---

## EighteenI don't know.TopK Queries

### 18.1 Top 5 error Pod

LogQL:

    topk(5,
      sum by (pod) (
        count_over_time(
          {namespace="app-demo"}
            |~ "(?i)error|exception|panic|failed" [5m]
        )
      )
    )

### 18.2 Top 5 Slow Request path

Based on JSON logs:

    topk(5,
      max by (path) (
        max_over_time(
          {namespace="app-demo", app="json-log-demo"}
            | json
            | __error__=""
            | unwrap duration_ms
            | __error__="" [5m]
        )
      )
    )

Note:

    This query is used to find the path with the highest maximum duration in the last 5 minutes.

### 18.3 Top 5 5xx app

LogQL:

    topk(5,
      sum by (app) (
        count_over_time(
          {namespace="app-demo"}
            | json
            | __error__=""
            | status >= 500 [5m]
        )
      )
    )

Note:

    If not all logs in app-demo are JSON, JSONParserErr may occur.
    Already filtered by __error__=""

---

## NineteenI don't know.Advanced Queries in Grafana Explore

### 19.1 Query JSON error

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"

### 19.2 Query slow requests

    {namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000

### 19.3 Format display

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 19.4 Metric Query Example: P95

Switch to Metrics query mode in Grafana Explore:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 19.5 Note the query type in Explore

When querying logs in Grafana Explore, you can directly display log lines.

If using:

    avg_over_time
    quantile_over_time
    count_over_time
    rate

It returns metric series, suitable for visualization or alerts.

---

## TwentyI don't know.Common Advanced Query Templates

### 20.1 JSON error logs

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | level="error"

### 20.2 JSON 5xx logs

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | status >= 500

### 20.3 JSON slow requests

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | duration_ms > 1000

### 20.4 JSON query trace_id

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | trace_id="trace-error-10"

### 20.5 JSON formatted display

### 20.6 logfmt error Logs

    {namespace="app-demo", app="logfmt-log-demo"}
      | logfmt
      | __error__=""
      | level="error"

### 20.7 logfmt 5xx Logs

    {namespace="app-demo", app="logfmt-log-demo"}
      | logfmt
      | __error__=""
      | status >= 500

### 20.8 regexp Extract Nginx 404

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | __error__=""
      | status="404"

### 20.9 Count error Instances

    sum by (app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 20.10 Count 5xx Instances

    sum by (app) (
      count_over_time(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__=""
          | status >= 500 [5m]
      )
    )

### 20.11 Average Latency

    avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 20.12 P95 Latency

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 20.13 Top error Pod

    topk(5,
      sum by (pod) (
        count_over_time(
          {namespace="app-demo"}
            |~ "(?i)error|exception|panic|failed" [5m]
        )
      )
    )

### 20.14 Top Slow Request Paths

    topk(5,
      max by (path) (
        max_over_time(
          {namespace="app-demo", app="json-log-demo"}
            | json
            | unwrap duration_ms
            | __error__="" [5m]
        )
      )
    )

---

## Twenty-oneI don't know.Advanced Query Troubleshooting

### 21.1 json Query Returns No Results

Possible causes:

    Logs are not JSON.
    Field name is incorrect.
    JSON parsing failed.
    Query time range is incorrect.
    app label is incorrect.
    Alloy has not collected logs.

Troubleshooting:

    {namespace="app-demo", app="json-log-demo"}

    {namespace="app-demo", app="json-log-demo"} | json | __error__!=""

### 21.2 status>=500 Query Returns No Results

Possible causes:

    status field does not exist.
    status is a string but cannot be compared.
    Logs are not JSON.
    Parsing failed.
    No 5xx logs in current time range.

First check fields:

    {namespace="app-demo", app="json-log-demo"} | json | line_format "status={{.status}} msg={{.msg}}"

### 21.3 unwrap Error

Possible causes:

    duration_ms field does not exist.
    duration_ms is not a number.
    Non-JSON lines exist.
    Parsing failed.
    __error__ filter is not applied.

Correct syntax:

    avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 21.4 regexp Cannot Extract Fields

Possible causes:

    Regular expression does not match log format.
    No named capture groups used.
    Quote escaping error.
    Logs have different formats.
    Incorrect app/container used.

Troubleshooting:

    First check raw logs:

        {namespace="app-demo", app="nginx-demo"}

    Then gradually add regexp.

### 21.5 line_format No Output

Possible causes:

    Field name is incorrect.
    Parser did not parse successfully.
    __error__ not handled.
    Used non-existent template variables.

Troubleshooting:

    {namespace="app-demo", app="json-log-demo"} | json | __error__=""

Then confirm field names.

### 21.6 Metric Query Returns Error

Common causes:

The metric query contains __error__.
The unwrap field cannot be converted to a number.
The query range is too large.
Loki query timeout.
Error occurred during parsing.

Solution:

Add after the stage that produces the error:

    | __error__=""

---

## Twenty-two, Performance Considerations

### 22.1 Filter First, Then Parse

Recommended:

    {namespace="app-demo", app="json-log-demo"} |= "database" | json | level="error"

Not Recommended:

    {namespace="app-demo", app="json-log-demo"} | json | level="error" |= "database"

Reason:

    Filtering with strings first reduces the amount of JSON parsing required.

### 22.2 Use Labels to Narrow the Scope First

Recommended:

    {namespace="app-demo", app="json-log-demo"} | json | level="error"

Not Recommended:

    {namespace=~".+"} | json | level="error"

### 22.3 Avoid Large Range Regular Expressions

Regular expressions are costly.

Recommendation:

    First use Labels to narrow the scope.
    Then use |= for coarse filtering.
    Finally use regexp for parsing.

Example:

    {namespace="app-demo", app="nginx-demo"}
      |= "GET"
      | regexp `...`

### 22.4 Control the Time Range

Advanced queries consume more resources than basic string queries.

Recommendation:

    For troubleshooting, query the last 15 minutes first.
    Use reasonable intervals for dashboards.
    Avoid large alert windows.
    Do not frequently query large time ranges (e.g., 7 days).

### 22.5 JSON / logfmt is Better than regexp

Production recommendation for application output:

    JSON
    logfmt

Avoid long-term reliance on regexp for parsing complex text logs.

---

## Twenty-three, Production Implementation Recommendations

### 23.1 Recommended Application Output JSON Logs

Recommended JSON log fields:

    timestamp
    level
    service
    trace_id
    span_id
    method
    path
    status
    duration_ms
    msg
    error_type

Example:

    {"timestamp":"2026-05-01T12:00:00+08:00","level":"error","service":"order-api","trace_id":"abc123","method":"POST","path":"/api/v1/order","status":500,"duration_ms":1200,"msg":"database connection failed","error_type":"database_error"}

### 23.2 Field Naming Should Be Consistent

Avoid mixing:

    duration
    durationMs
    duration_ms
    cost
    elapsed

Recommended uniformity:

    duration_ms

Avoid mixing:

    level
    severity
    log_level

Recommended uniformity:

    level

### 23.3 Status Code Fields Should Be Numbers

Recommended:

    status: 500

Avoid:

    status: "500"

Numbers are more suitable for comparison and statistics.

### 23.4 Do Not Use trace_id as a Label

trace_id should be placed in the log body.

Query:

    {namespace="app-prod", app="order-api"} | json | trace_id="abc123"

### 23.5 Advanced Queries Should Be Templated

Recommended toDeposition in the knowledge base:

    Error log query template
    5xx query template
    Slow request query template
    P95 query template
    Trace query template
    Timeout query template
    Database error query template
    CUDA OOM query template

---

## Twenty-four, Hands-on Tasks

### 24.1 Task One: Deploy JSON Log Demo

Execute:

    kubectl apply -f json-log-demo.yaml

    kubectl get pod -n app-demo | grep json-log-demo

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

Acceptance:

    [ ] Can see JSON logs
    [ ] Has info logs
    [ ] Has warn logs
    [ ] Has error logs
    [ ] Has non-JSON lines

### 24.2 Task Two: Use json to Query error

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error"' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Can query level=error logs

### 24.3 Task Three: Query status>=500

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | status >= 500' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Can query status 500 logs

### 24.4 Task Four: Query duration_ms>1000

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | duration_ms > 1000' \
      --data-urlencode 'limit=20' | jq

Acceptance:

    [ ] Can query slow request logs

### 24.5 Task Five: Using line_format

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json | __error__="" | level="error" | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"' \
      --data-urlencode 'limit=20' | jq

Verification:

    [ ] Query results are more clearly displayed

### 24.6 Task Six: Deploying logfmt Demo

Execute:

    kubectl apply -f logfmt-log-demo.yaml

    kubectl get pod -n app-demo | grep logfmt-log-demo

    kubectl logs <logfmt-log-demo-pod> -n app-demo --tail=20

Verification:

    [ ] Can see logfmt logs

### 24.7 Task Seven: Using logfmt for Queries

Execute:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="logfmt-log-demo"} | logfmt | __error__="" | level="error"' \
      --data-urlencode 'limit=20' | jq

Verification:

    [ ] Can query logfmt error logs

### 24.8 Task Eight: Using unwrap to Calculate Average Duration

Execute in Grafana Explore or API:

    avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

Verification:

    [ ] Can see average duration_ms value

### 24.9 Task Nine: Using quantile_over_time to Calculate P95

Execute:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

Verification:

    [ ] Can see P95 duration_ms

### 24.10 Task Ten: Counting Error Logs

Execute:

    sum by (app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

Verification:

    [ ] Can count error logs by app

---

## Twenty-five, Verification Checklist

After completing this document, confirm:

    [ ] Understand LogQL pipeline
    [ ] Understand json parser
    [ ] Understand logfmt parser
    [ ] Understand regexp parser
    [ ] Understand __error__
    [ ] Can filter __error__=""
    [ ] Can query JSON level=error
    [ ] Can query JSON status>=500
    [ ] Can query JSON duration_ms>1000
    [ ] Can query log body by trace_id
    [ ] Can format logs with line_format
    [ ] Can create temporary query labels with label_format
    [ ] Can query logfmt level=error
    [ ] Can extract Nginx method/path/status with regexp
    [ ] Can extract duration_ms with unwrap
    [ ] Can calculate avg_over_time
    [ ] Can calculate max_over_time
    [ ] Can calculate quantile_over_time
    [ ] Can use count_over_time
    [ ] Can use rate
    [ ] Can group statistics by app/pod/status
    [ ] Can write error log statistics query
    [ ] Can write 5xx log statistics query
    [ ] Can write P95 duration query
    [ ] Understand the performance principle of filtering before parsing

---

## Twenty-six, Common Misconceptions

### 26.1 Misconception One: All Logs Are Suitable for JSON Parsing

Incorrect.

Only JSON-formatted logs are suitable:

    | json

Non-JSON logs will produce:

    __error__="JSONParserErr"

### 26.2 Misconception Two: unwrap Does Not Need to Handle Errors

Incorrect.

unwrap may produce errors if the field does not exist or cannot be converted to a number.

Recommendation:

    | unwrap duration_ms
    | __error__=""

### 26.3 Misconception Three: line_format Changes Original Logs

Incorrect.

line_format only changes how query results are displayed.

It does not modify data stored in Loki.

### 26.4 Misconception Four: regexp Is the Best Parsing Method

Incorrect.

regexp is flexible but has high cost and poor maintainability.

Production recommends:

    JSON
    logfmt

### 26.5 Misconception Five: trace_id Queries Must Be Made as Labels

Incorrect.

trace_id should be in log body or structured fields.

Query:

    {namespace="app-prod", app="api"} | json | trace_id="abc123"

Avoid:

    {trace_id="abc123"}

### 26.6 Misconception Six: limit Can Reduce All Query Costs

Not entirely correct.

limit only restricts the number of returned results.

True cost reduction comes from:

    Narrowing label scope
    Narrowing time range
    Filtering before parsing
    Avoiding broad regexp

---

## Twenty-seven, Production Notes

### 27.1 Application Logs Should Be Structured

Recommended:

    JSON
    logfmt

Not recommended for long-term dependency:

    Unformatted text
    Long stack mixed fields
    Multi-line format messy logs

### 27.2 Log Fields Should Be Unified

Key fields should be unified:

    level
    service
    trace_id
    method
    path
    status
    duration_ms
    msg
    error_type

After unifying fields, Dashboards and alerts can reuse them.

### 27.3 Avoid Making Advanced Queries Become Global Scans

Error:

    {namespace=~".+"} | json | level="error"

Recommended:

    {environment="prod", namespace="order", app="order-api"} | json | level="error"

### 27.4 Slow Queries Should BeDeposition into Recording Rule or Dashboard

If certain log metrics are used long-term, such as:

    error count
    5xx count
    P95 duration

Consider using:

    Loki Ruler
    Recording Rule
    Grafana Dashboard Fixed Panel

### 27.5 Alert Queries Should Be Simple and Stable

Alert rules should not be overly complex.

Recommended:

    Clear label scope
    Reasonable time window
    Clear error handling
    Avoid large-range regex
    Include runbook_url
    Include team label

---

## Twenty-Eight, Summary

This article completes the advanced LogQL queryActual.

Core capabilities include:

    json:
        Parse JSON logs.

    logfmt:
        Parse key=value logs.

    regexp:
        Extract fields from plain text.

    __error__:
        Identify and filter pipeline parsing errors.

    line_format:
        Change log display format.

    label_format:
        Create or adjust temporary labels during queries.

    unwrap:
        Convert log fields into numeric series.

    count_over_time:
        Count log entries.

    rate:
        Count log rate.

    avg_over_time / max_over_time / quantile_over_time:
        Statistic numeric field trends and percentiles.

The most recommended log format in production is:

    Application outputs JSON / logfmt structured logs
      ↓
    Loki uses low-cardinality labels to narrow scope
      ↓
    LogQL uses json / logfmt to parse fields
      ↓
    Filter by status, level, duration_ms using fields
      ↓
    Use unwrap to generate metrics
      ↓
    Grafana displays trends
      ↓
    Loki Ruler triggers log alerts

Key principles:

    First use labels to narrow scope.
    Then use string filtering.
    Then parse fields.
    Then perform statistics.
    Avoid global wide-range parsing.
    Do not use high-cardinality fields as labels.
    Put trace_id / user_id / order_id into log body, use json queries.

Next article will enter:

    10-Grafana Integration with Loki and Log DashboardActual

Key learning points:

    Add Loki Data Source in Grafana
    Explore log queries
    Create Namespace / App / Pod variables
    Create ERROR log panel
    Create 5xx panel
    Create slow request panel
    Jump from Pod metrics to Loki logs
    Build a log Dashboard usable for production troubleshooting

---

## Reference Documents

- Grafana Loki Documentation:
  https://grafana.com/docs/loki/latest/

- Query Loki:
  https://grafana.com/docs/loki/latest/query/

- Log queries:
  https://grafana.com/docs/loki/latest/query/log_queries/

- LogQL Reference:
  https://grafana.com/docs/loki/latest/query/query_reference/

- Metric queries:
  https://grafana.com/docs/loki/latest/query/metric_queries/

- Query examples:
  https://grafana.com/docs/loki/latest/query/query_examples/

- Query best practices:
  https://grafana.com/docs/loki/latest/query/bp-query/

- LogQL template functions:
  https://grafana.com/docs/loki/latest/query/template_functions/

- Loki HTTP API:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- Grafana Explore:
  https://grafana.com/docs/grafana/latest/explore/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/