# 09-Advanced LogQL Queries in Practice: json-logfmt-regexp-unwrap

## Document Description

This article is the ninth installment in the dedicated study on Loki, aimed at systematically mastering advanced LogQL query capabilities. It focuses on JSON log parsing, logfmt log parsing, extracting fields using regex, formatting output with line_format, creating temporary labels with label_format, converting numerical fields using unwrap, and generating metric queries from logs.

Previous topics have covered:

    01-Basic Understanding of Loki and Experimental Environment Setup
    02-Practical Observation of Loki's Architecture Principles and Component Responsibilities
    03-Comparison of Loki Deployment Modes and Experimental Selection
    04-Practical Helm Deployment of Loki in a Standalone Mode
    05-Practical Integration of Loki Object Storage with MinIO
    06-Practical Collection of K8S-Pod Logs Using Grafana-Alloy
    07-Loki Tag Design and Experiments on High Cardinality Issues
    08-Basic LogQL Queries in Practice: Retrieving Logs by Namespace, Pod, and Container

In Article 08, basic queries were explored:

    {namespace="app-demo"}
    {namespace="app-demo", app="nginx-demo"}
    {namespace="app-demo"} |= "ERROR"
    {namespace="app-demo"} |~ "(?i)error|exception|panic"

This article delves into advanced queries, focusing on solving the following problems:

- How to parse JSON logs;
- How to parse logfmt logs;
- How to extract fields from plain text logs using regex;
- How to filter logs based on status, level, and duration_ms;
- How to handle JSON parsing errors;
- How to use __error__ to filter data with parsing failures;
- How to optimize log display using line_format;
- How to generate temporary labels using label_format;
- How to convert log fields into numerical values using unwrap;
- How to calculate the average, maximum value, and P95 of duration_ms;
- How to count the number of ERROR logs;
- How to count the number of 5xx logs;
- How to group statistics by app, pod, and status;
- How to generate alarm-worthy metric queries from logs;
- How to prevent advanced queries from slowing down Loki.

This article is suitable for those who have already mastered basic LogQL queries.

---

## Tags

#Loki #LogQL #JSONLogs #logfmt #regexp #unwrap #line_format #label_format #Grafana #Kubernetes #PodLogs #SRE #Observability #LogAnalysis

---

## Recommended Reading Path

Recommended path:

    10-Logs/02-Loki/09-Advanced LogQL Queries in Practice: json-logfmt-regexp-unwrap.md

---

## I. Experimental Objectives

Upon completing this article, you should be able to:

    1. Understand how the LogQL pipeline operates.
    2. Use | json to parse JSON logs.
    3. Use | logfmt to parse logfmt logs.
    4. Use | regexp to extract fields from plain text logs.
    5. Filter logs based on level="error".
    6. Filter logs based on status >= 500.
    7. Filter logs based on duration_ms > 1000.
    8. Understand the role of __error__.
    9. Filter out parsing errors such as JSONParserErr.
    10. Use line_format to customize the display of query results.
    11. Use label_format to temporarily generate or adjust labels.
    12. Use unwrap to convert fields into numerical sequences.
    13. Use count_over_time to count the number of logs.
    14. Use rate to calculate log rates.
    15. Use avg_over_time to compute average durations.
    16. Use max_over_time to find maximum durations.
    17. Use quantile_over_time to calculate P95/P99 values.
    18. Convert log queries into metric queries.
    19. Write LogQL queries suitable for Grafana Dashboards.
    20. Understand the performance considerations of advanced LogQL queries.

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
    nginx-demo
    json-log-demo
    logfmt-log-demo

### 2.2 Prerequisites

It is necessary to confirm the            app.kubernetes.io/name: json-log-demo
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

To apply this configuration, execute the following command:

    kubectl apply -f json-log-demo.yaml

To check the job creation, use these commands:

    kubectl get job -n app-demo

    kubectl get pod -n app-demo | grep json-log-demo

To view the logs generated by the Job, use:

    kubectl logs <json-log-demo-pod> -n app-demo --tail=20

### 4.2 Description

This Job generates four types of logs:

- **info**: Status code 200, low duration milliseconds, message "request success".
- **warn**: Status code 404, moderate duration milliseconds, message "resource not found".
- **error**: Status code 500, high duration milliseconds, message "database connection failed".
- **non-JSON lines**: Message "this is not a json line", used for demonstration purposes.

### 5.2 Using JSON to Parse Logs

#### LogQL Queries

To query the original JSON logs:

    {namespace="app-demo", app="json-log-demo"}

Or, using the `json` function in LogQL:

    {namespace="app-demo", app="json-log-demo"} | json

To filter logs with a specific level (e.g., `error`):

    {namespace="app-demo", app="json-log-demo"} | json | level="error"

#### API Queries

Similar to LogQL, you can use the `json` function in the API requests:

    curl -G -s "http://127.0.0.1:3100/loki/api/v1/query_range" \
      --data-urlencode 'query={namespace="app-demo", app="json-log-demo"} | json' \
      --data-urlencode 'limit=20' | jq

These commands will return the logs in JSON format, making it easier to process and analyze them.The Dashboard panel shows no data or reports an error.
The alarm rule execution failed.

Production recommendations:

Whenever using json, logfmt, regexp, or unwrap, pay attention to __error__.

---

## VII. Optimizing Display with line_format

### 7.1 What is line_format

line_format is used to change the way log lines are displayed in query results.

It does not alter the original log storage; it only affects the results returned by queries.

Suitable for:

- Showing only key fields
- Simplifying Grafana Explore outputs
- Making troubleshooting logs clearer
- Creating log table panels

### 7.2 Displaying level, status, duration, and msg

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | line_format "{{.level}} status={{.status}} duration={{.duration_ms}}ms msg={{.msg}}"

Example output:

    error status=500 duration=2780ms msg=database connection failed

### 7.3 Displaying trace_id and path

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__=""
      | line_format "trace={{.trace_id}} method={{.method}} path={{.path}} status={{.status}}"

### 7.4 Querying errors and formatting them

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__``
      | level="error"
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

### 7.5 Precautions

line_format only affects the display.

It does not:

- Modify the logs stored in Loki.
- Change labels.
- Alter the original log content.
- Reduce storage costs.

---

## VIII. Temporarily Handling Tags with label_format

### 8.1 What is label_format

label_format is used to create, modify, or format temporary tags during query processes.

It is suitable for:

- Temporarily creating display fields
- Formatting fields
- Adding tags to query results
- Optimizing Dashboard displays

Note:

- label_format generates temporary tags for the current query.
- These are not the same as persistent labels written to Loki during collection.
- It does not affect underlying storage.

### 8.2 Creating a temporary result tag

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__``
      | label_format result="{{.method}}_{{.status}}"

### 8.3 Displaying with line_format

LogQL:

    {namespace="app-demo", app="json-log-demo"}
      | json
      | __error__``
      | label_format result "{{.result}} path={{.path}} duration={{.duration_ms}}ms msg={{.msg}}"
      | line_format "result={{.result}} path={{.path}} duration={{.duration_ms}}ms msg={{.msg}}"

### 8.4 Label_format is not a solution for high cardinality fields

Do not use label_format for high cardinality fields as collection tags just because it is convenient.

Difference:

- Collection-phase labels:
    Affect Loki Stream, indexing, writing, and storage costs.

- Query-phase label_format:
    Only affects the current query results.

---

## IX. Preparing a logfmt Log Demo

### 9.1 What is logfmt

logfmt is a type of log that uses key=value format.

Example:

    level=info service=logfmt-demo status=200 duration_ms=32 msg="request success"

Features:

- More concise than JSON.
- Easy to read and parse.
- Commonly used in many Go services and infrastructure components.

### 9.2 Creating a logfmt-log-demo Job

Create a file:

    logfmt-log-demo.yaml

Content:

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
                    echo "level=info service=logfmt-log-demo method=GET path=/api/v1/users status### 10.6 Query for duration_ms > 1000

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | duration_ms > 1000

### 10.7 Query for msg

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | msg="upstream timeout"

### 10.8 Filter for parsing errors

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"} | logfmt | __error__=""

### 10.9 Formatted display

LogQL:

    {namespace="app-demo", app="logfmt-log-demo"}
      | logfmt
      | __error__=""
      | line_format "[{{.level}}] {{.method}} {{.path}} status={{.status}} duration={{.duration_ms}}ms trace={{.trace_id}} msg={{.msg}}"

---

## Chapter Eleven: Using Regexp to Parse Plain Text Logs

### 11.1 When to Use Regexp

Regexp is suitable for parsing:

    Nginx access logs
    Plain text logs in Java and Python
    Unstructured logs from legacy applications
    Logs that cannot be converted to JSON/logfmt format

Disadvantages:

    Complex regular expressions.
    Poor readability.
    Lower performance compared to structured parsing.
    Likely to fail when the log format changes.

Production recommendations:

    Prefer JSON/logfmt if possible over regex.
    Use regex for historical systems or transitional scenarios.

### 11.2 Example of Nginx Default Logs

A typical Nginx access log might look like this:

    10.244.1.10 - - [01/May/2026:12:00:00 +0000] "GET /notfound HTTP/1.1" 404 153 "-" "curl/8.5.0" "-"

### 11.3 Using Regexp to Extract method, path, and status

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`

Explanation:

    ?P<method>:
        Extracts the method field.

    ?P<path>:
        Extracts the path field.

    ?P<status>:
        Extracts the status field.

    ?P<body_bytes_sent>:
        Extracts the response size in bytes.

### 11.4 Querying for 404 Errors

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | status="404"

### 11.5 Querying for 5xx Errors

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | status=~"5.."

### 11.6 Formatted Display of Nginx Logs

LogQL:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?P<status>\d{3}) (?P<body_bytes_sent>\d+)`
      | line_format "{{.method}} {{.path}} status={{.status}} bytes={{.body_bytes_sent}} ip={{.remote_addr}}"

### 11.7 Handling Regexp Parsing Failures

If some logs do not match the regex pattern, __error__ will be generated.

To filter out successfully parsed logs:

    {namespace="app-demo", app="nginx-demo"}
      | regexp `^(?P<remote_addr>\S+) - - \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) (?P<protocol>[^"]+)" (?PLogQL:

    quantile_over_time(
      0.99,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 13.6 Only calculate the P95 for error requests

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

Explanation:

    The first __error__="" filters out errors in JSON parsing.
    The second __error__="" filters out errors in the unwrap conversion.

---

## Section Fourteen: Using unwrap to calculate logfmt log processing time

### 14.1 Average processing time

LogQL:

    avg_over_time(
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 14.2 Maximum processing time

LogQL:

    max_over_time(
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 14.3 P95 processing time

LogQL:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | unwrap duration_ms
        | __error__="" [5m]
    )

### 14.4 Only calculate the average processing time for 5xx requests

LogQL:

    avg_over_time(
      {namespace="app-demo", app="logfmt-log-demo"}
        | logfmt
        | __error__``
        | status >= 500
        | unwrap duration_ms
        | __error__="" [5m]
    )

---

## Section Fifteen: count_over_time: Counting the number of logs

### 15.1 Counting the number of logs in app-demo for 5 minutes

LogQL:

    count_over_time({namespace="app-demo"}[5m])

### 15.2 Counting the number of error logs in json-log-demo

LogQL:

    count_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | __error__``
        | level="error" [5m]
    )

### 15.3 Counting the number of error logs grouped by app

LogQL:

    sum by (app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 15.4 Counting the number of error logs grouped by pod

LogQL:

    sum by (pod) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 15.5 Counting the number of JSON logs with status >= 500

LogQL:

    sum by (app) (
      count_over_time(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__``
          | status >= 500 [5m]
      )
    )

---

## Section Sixteen: rate: Calculating log rates

### 16.1 Calculating the rate of logs per second

LogQL:

    rate({namespace="app-demo"}[5m])

### 16.2 Calculating the rate of error logs

LogQL:

    sum by (app) (
      rate(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

### 16.3 Calculating the rate of 5xx logs

LogQL:

    sum by (app) (
      rate(
        {namespace="app-demo", app="json-log-demo"}
          | json
          | __error__``
          | status >= 500 [5m]
      )
    )

### 16.4 The difference between rate and count_over_time

count_over_time:

    Counts the total number of logs within a time window.

rate:

    Calculates the average rate per unit of time.

For example:

    If there were 300 error logs in the last 5 minutes:
        count_over_time = 300

    If there was an average of 1 error log per second:
        rate ≈ 1

Common usage in alerts:

    count_over_time

Common usage in dashboard trends:

    rate```markdown
sum by (app) (
  count_over_time(
    {namespace="app-demo"}
      | json
      | __error__=""
      | status >= 500 [5m]
  )
)
```

Note:

If not all logs in `app-demo` are in JSON format, `JSONParserErr` may occur. This has been filtered out using `__error__=""`.The regular expression does not match the log format. Named capture groups are not being used. There are errors with quoting and escaping characters. The logs contain different formats, and the wrong app/container is being referenced.

### Investigation:

First, check the original logs:

        {namespace="app-demo", app="nginx-demo"}

Then gradually add regular expressions.

### 21.5 line_format produces no output

Possible reasons:

    The field name was misspelled.
    The parser failed to parse it successfully.
    The __error__ variable was not processed.
    An invalid template variable was used.

Investigation:

    {namespace="app-demo", app="json-log-demo"} | json | __error__=""

Then confirm the field names.

### 21.6 Error occurs when querying metrics

Common reasons:

    There is an __error__ in the metric query.
    The unwrap field cannot be converted to a numerical value.
    The search range is too wide.
    The Loki query timed out.
    An error occurred during the parsing stage.

Solution:

    Add the following after the stage where the error occurred:

        | __error__=""

---

## Section 22: Performance Considerations

### 22.1 Filter first, then parse

Recommended approach:

    {namespace="app-demo", app="json-log-demo"} |= "database" | json | level="error"

Not recommended approach:

    {namespace="app-demo", app="json-log-demo"} | json | level="error" |= "database"

Reason:

    Filtering the data as a string first can reduce the amount of JSON that needs to be parsed later on.

### 22.2 Use labels to narrow down the search range

Recommended approach:

    {namespace="app-demo", app="json-log-demo"} | json | level="error"

Not recommended approach:

    {namespace=~".+"} | json | level="error"

### 22.3 Avoid using large-scale regular expressions

Regular expressions are resource-intensive.

Suggestion:

    First, use labels to narrow down the search range.
    Then use |= for a rough filter.
    Finally, perform the regular expression parsing.

Example:

    {namespace="app-demo", app="nginx-demo"}
      |= "GET"
      | regexp `...`

### 22.4 Control the time range

Advanced queries consume more resources than basic string searches.

Suggestion:

    For troubleshooting, start by checking the last 15 minutes of data.
    Use appropriate intervals for dashboards.
    Keep alert windows reasonably sized.
    Avoid frequently querying large datasets from 7 days ago.

### 22.5 JSON/logfmt are superior to regular expressions

In production scenarios, it is recommended to use:

    JSON
    logfmt

Try to avoid relying on regular expressions for parsing complex text logs over the long term.

---

## Section 23: Production Implementation Suggestions

### 23.1 Recommend using JSON logs

Recommended JSON log fields include:

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

### 23.2 Use consistent field naming

Avoid mixing up:

    duration
    durationMs
    duration_ms
    cost
    elapsed

It is recommended to use:

    duration_ms

Avoid mixing up:

    level
    severity
    log_level

It is recommended to use:

    level

### 23.3 Use numbers for status code fields

Suggestion:

    status: 500

Avoid using:

    status: "500"

Numbers are more suitable for comparison and statistical analysis.

### 23.4 Do not use trace_id as a label

Trace IDs should be included in the actual log content.

Example:

    {namespace="app-prod", app="order-api"} | json | trace_id="abc123"

### 23.5 Advanced queries should have templates

It is recommended to create templates in a knowledge base for:

    Error log queries
    5xx error queries
    Slow request queries
    P95 value queries
    Trace queries
    Timeout queries
    Database error queries
    CUDA OOM queries

---

## Section 24: Practical Tasks

### 24.1 Task 1: Deploy a JSON log demo

Execution:

    kubectl apply -f json-log-demo.yaml

    kubectl get pod -n app-demo | grep json-log-demo

    kubectl logs### 24.8 Task Eight: Calculate Average Duration Using unwrap

Execute in Grafana Explore or API:

    avg_over_time(
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

Acceptance Criteria:

    [ ] The average value of duration_ms should be visible.

### 24.9 Task Nine: Calculate P95 Using quantile_over_time

Execute:

    quantile_over_time(
      0.95,
      {namespace="app-demo", app="json-log-demo"}
        | json
        | unwrap duration_ms
        | __error__="" [5m]
    )

Acceptance Criteria:

    [ ] The P95 value of duration_ms should be visible.

### 24.10 Task Ten: Count the Number of error Logs

Execute:

    sum by (app) (
      count_over_time(
        {namespace="app-demo"}
          |~ "(?i)error|exception|panic|failed" [5m]
      )
    )

Acceptance Criteria:

    [ ] The number of error logs should be able to be counted by app.

---

## Chapter Twenty-Five: Acceptance Checklist

After completing this document, you should confirm that you have understood the following:

    [ ] LogQL pipelines
    [ ] json parsers
    [ ] logfmt parsers
    [ ] regexp parsers
    [ ] The meaning of __error__
    [ ] How to filter logs with __error__="" 
    [ ] How to query JSON logs where level=error
    [ ] How to query JSON logs where status>=500
    [ ] How to query JSON logs where duration_ms>1000
    [ ] How to retrieve log contents using trace_id
    [ ] How to format log displays using line_format
    [ ] How to create temporary query tags using label_format
    [ ] How to use logfmt to query level=error logs
    [ ] How to extract Nginx method/path/status using regexp
    [ ] How to extract duration_ms using unwrap
    [ ] How to calculate avg_over_time
    [ ] How to calculate max_over_time
    [ ] How to calculate quantile_over_time
    [ ] How to use count_over_time
    [ ] How to use rate
    [ ] How to group statistics by app/pod/status
    [ } How to write queries for counting error logs
    [ ] How to write queries for counting 5xx logs
    [ ] How to write queries for calculating P95 latency
    [ ] The performance principle of filtering before parsing

---

## Chapter Twenty-Six: Common Misconceptions

### 26.1 Misconception One: All Logs Are Suitable for JSON Parsing

Incorrect.

Only JSON-formatted logs are suitable for parsing with:

    | json

Non-JSON logs will result in:

    __error__="JSONParserErr"

### 26.2 Misconception Two: unwrap Does Not Require Error Handling

Incorrect.

unwrap may encounter errors if fields do not exist or cannot be converted to numbers.

Recommendation:

    | unwrap duration_ms
    | __error__=""

### 26.3 Misconception Three: line_format Changes the Original Logs

Incorrect.

line_format only affects the display of query results; it does not modify the data stored in Loki.

### 26.4 Misconception Four: regexp Is the Best Parsing Method

Incorrect.

Regexp is flexible but can be costly and difficult to maintain. For production use, JSON and logfmt are more recommended.

### 26.5 Misconception Five: trace_id Queries Must Be Stored as Labels

Incorrect.

trace_id should be included in the log content or structured fields. Example query:

    {namespace="app-prod", app="api"} | json | trace_id="abc123"

Not:

    {trace_id="abc123"}

### 26.6 Misconception Six: limit Can Reduce the Cost of All Queries

Inaccurate.

limit only limits the number of results returned. To truly reduce costs, consider:

    - Narrowing the label range
    - Narrowing the time frame
    - Filtering before parsing
    - Avoiding extensive regexp queries

---

## Chapter Twenty-Seven: Production Considerations

### 27.1 Application Logs Should Be Structured

Recommendation:

    JSON
    logfmt

Avoid long-term reliance on:

    Unstructured text
    Extremely long logs with mixed fields
    Logs in multiple-line formats that are difficult to parse

### 27.2 Log Fields Must Be Unified

Standardize key fields such as:

    level
    service
    trace_id
    method
    path
    status
    duration_ms
    msg
    error_type

Unified fields enable reus- LogQL Template Functions:
  https://grafana.com/docs/loki/latest/query/template_functions/

- Loki HTTP API:
  https://grafana.com/docs/loki/latest/reference/loki-http-api/

- Grafana Explore:
  https://grafana.com/docs/grafana/latest/explore/

- Kubernetes Logging Architecture:
  https://kubernetes.io/docs/concepts/cluster-administration/logging/