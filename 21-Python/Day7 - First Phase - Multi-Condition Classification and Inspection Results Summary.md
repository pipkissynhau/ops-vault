# Day7 - First Stage - Multi-Condition Classification and Inspection Result Summary

#Python #PythonLearning #TransportDevelopment #ForCirculation #IfJudgment #elif #List #append #len #split #LogAnalysis #StatusCode #ServiceInspection #CheckScript #Linux #Kubernetes #Obsidian

## Today's Focus

Day1 Learning Content:

- Variables
- Strings
- `print()`
- `f-string`
- `in`
- `strip()`
- `startswith()`
- `endswith()`

Day2 Learning Content:

- `if / elif / else`
- Multi-branch Conditional Judgment
- `upper()` / `lower()`
- `replace()`
- `split()`

Day3 Learning Content:

- `and`
- `or`
- `not`
- Multi-Condition Combination Judgment
- `in`
- `not in`
- `startswith()`
- `endswith()`

Day4 Learning Content:

- List `list`
- Index (Subscript)
- `for` Loop
- Traversing Lists
- `for + if` for Basic Batch Judgment

Day5 Learning Content:

- Result Collection
- `append()`
- Basic Counting
- Batch Filtering
- Collect Results and Output Statistics

Day6 Learning Content:

- `len()` Length Statistics
- Result List Quantity Judgment
- Empty List Judgment
- "Collect Results and Continue Judgment" Script Thinking

Day7's New Stage is:

**Not just filtering one type of result, but classifying the same batch of data by different conditions and outputting more complete inspection results.**

This step is crucial because in real operations, common scenarios are not only:

- Whether there are anomalies
- Whether the quantity is greater than 0

But also need to continue classification, for example:

- Which logs in the logs are `INFO`
- Which are `WARNING`
- Which are `ERROR`
- Which status codes are normal
- Which are client errors
- Which are server errors
- Which services are `running`
- Which are `stopped`
- Which are `failed`

Therefore, the core of Day7 is to start building the fourth level of automation thinking:

**The program should not only find results and count them, but also classify the results and output content that resembles an inspection report.**

---

## Today's Goals

After completing Day7, you should be able to:

1. Understand the significance of multi-condition classification
2. Use `if / elif / else` in a loop
3. Put the same batch of data into different result lists
4. Statistically count the quantities of multiple result lists
5. Output more complete batch inspection results
6. Write a basic script for "classification + statistics + result display"
7. Prepare for subsequent function encapsulation

---

## One, What You'll Learn Today

Day7's content can be summarized as:

1. Multi-condition classification judgment
2. Use of `if / elif / else` in batch processing
3. Splitting one original list into multiple result lists
4. Statistically counting quantities of multiple result lists
5. More complete result output
6. Structure closer to inspection scripts
7. Classroom exercises
8. Homework

---

## Two, Why Day7 is Important

Day6's scripts are mostly like this:

    error_log_list = []

    for log_line in log_list:
        if "ERROR" in log_line:
            error_log_list.append(log_line)

    print(error_log_list)
    print(len(error_log_list))

    if len(error_log, error_log_list) > 0:
        print("Found error logs")
    else:
        print("No error logs found")

This already completes basic filtering, but real scenarios will require further actions:

- Separate error logs, warning logs, and normal logs
- Separate normal status codes from abnormal status codes
- Separate running services from abnormal services
- Separate different types of configuration lines
- Finally output classified results and statistical results

This means:

**Scripts cannot only "filter one type", but also "classify by multiple conditions".**

This is the significance of Day7.

---

## Three, Difference Between Day7 and Day6

### Day6's Core

- Collect one type of result
- Count quantity
- Judge whether there are results

### Day7's Core

- Process multiple types of results simultaneously
- Separate different types of data for collection
- Count quantities separately
- Output more complete classified results

You can also understand it this way:

### Day6 is more like

"Are there any anomalies?"

### Day7 is more like

"What types of anomalies? What types of warnings? What types of normal logs? How many of each?"

---

## Four, Day7's Key Syntax: `if / elif / else`

Basic structure:

    if condition1:
        do thing1
    elif condition2:
        do thing2
    else:
        do thing3

In Day7, it often appears together with `for`:

    for item in item_list:
        if condition1:
            put into list1
        elif condition2:
            put into list2
        else:
            put into list3

This is the core code pattern of Day7.

---

## Five, What is "Classification Collection"

Day6 is more about:

- Find error logs
- Find Kubernetes-related services
- Find `.log` files

Day7 starts learning:

- Put error logs into one list
- Put warning logs into one list
- Put normal logs into one list

In other words:

**The same batch of data, entering different result lists based on different conditions.**

---

## Six, Day7's Key Supplement: What Scenarios Are `in`, `==`, `split()` Suitable For

### 1) `in`

Suitable for finding keywords in an entire line of text.

Example:

    if "running" in service_result:
        running_service_list.append(service_result)

    if "listen" in config_line:
        listen_config_list.append(config_line)

Applicable scenarios:

- Log lines
- Configuration lines
- Full line content of command outputs
- Full text of service inspection results

---

### 2) `==`

Suitable for precise comparison when the variable itself is a clear value.

Example: /think

if pod_status == "Running":
    running_pod_list.append(pod_status)

elif pod_status == "Pending":
    pending_pod_list.append(pod_status)

Applicable Scenarios:

- Pod status
- Service status values
- Single status code exact match
- Variable is a single determined value

---

### 3I'm not sure.`split()`

Suitable for splitting a line of fixed-format text first, then extracting a specific field.

Example:

    access_log = "10.0.0.2 GET /login 404"

    parts = access_log.split()
    status_code = parts[3]

    print(parts)
    print(status_code)

Output result:

    ['10.0.0.2', 'GET', '/login', '404']
    404

Applicable Scenarios:

- Access log splitting
- Command output splitting
- Configuration item splitting
- Extracting a specific field from a line of text

---

## SevenI don't know.The Most Basic Classification Example: Log Classification

Example:

    log_list = [
        "INFO nginx started",
        "WARNING disk usage high",
        "ERROR mysql down",
        "INFO kubelet running",
        "ERROR redis connect failed"
    ]

    info_log_list = []
    warning_log_list = []
    error_log_list = []

    for log_line in log_list:
        if "INFO" in log_line:
            info_log_list.append(log_line)
        elif "WARNING" in log_line:
            warning_log_list.append(log_line)
        elif "ERROR" in log_line:
            error_log_list.append(log_line)

    print(info_log_list)
    print(warning_log_list)
    print(error_log_list)

Here, it's not just finding one type of result, but dividing into three categories.

---

## EightI don't know.Why `elif` Is Very Suitable for Classification Scenarios

Because classification is usually "mutually exclusive".

For example, a log line cannot simultaneously count as `ERROR` and `INFO`.  
So it can be written as:

    if "INFO" in log_line:
        ...
    elif "WARNING" in log_line:
        ...
    elif "ERROR" in log_line:
        ...

This makes the code clearer and more suitable for classification processing.

---

## NineI don't know.Operational Thinking for Status Code Classification

Status codes are very common in operations.  
For example, interface checks, Nginx access log analysis, and service liveness result judgment may all use status code classification.

A simple classification approach could be:

- `2xx`: Normal
- `4xx`: Client request anomaly
- `5xx`: Server-side anomaly
- Others: Temporarily categorized under other status codes

Example:

    status_code_list = ["200", "404", "500", "200", "502", "503"]

It can be split into multiple lists.

---

## TenI don't know.Operational Thinking for Service Status Classification

Service checks are also a typical classification scenario.  
For example, you might receive a batch of service statuses:

- `running`
- `stopped`
- `failed`

At this point, it's not just filtering out anomalies, but often needing to separately statistics:

- Normal services
- Stopped services
- Faulty services

Because this is more like an inspection output.

---

## ElevenI don't know.Operational Thinking for Configuration Classification

Configuration checks are the same.  
For example, in Nginx configuration lines, you might want to classify:

- `listen`
- `server_name`
- Other configurations

This would make it clearer what content is in the configuration file.

Although this step is simple, it closely aligns with real configuration troubleshooting thinking.

---

## TwelveI don't know.Day7's Core Process

Day7's core process can be summarized as:

**First traverse, then collect by different conditions, then separately statistics, and finally output the complete results.**

Written as code:

    error_log_list = []
    warning_log_list = []
    info_log_list = []

    for log_line in log_list:
        if "ERROR" in log_line:
            error_log_list.append(log_line)
        elif "WARNING" in log_line:
            warning_log_list.append(log_line)
        elif "INFO" in log_line:
            info_log_list.append(log_line)

    print(error_log_list)
    print(warning_log_list)
    print(info_log_list)

    print(len(error_log_list))
    print(len(warning_log_list))
    print(len(info_log_list))

---

## ThirteenI don't know.Day7's Result Output Is More Complete Than Day6

Day6 often has this output:

- Error log list
- Error log count
- Whether error logs were found

Day7 will be more like this:

- INFO log list
- WARNING log list
- ERROR log list
- The count of each of the three log types
- Which category needs focused attention

This indicates that the script has started to resemble "inspection result display" rather than just a small judgment.

---

## FourteenI don't know.The Most Basic Complete Example

### Example 1: Log Level Classification /think

log_list = [
    "INFO nginx started",
    "WARNING disk usage high",
    "ERROR mysql down",
    "INFO kubelet running",
    "ERROR redis connect failed"
]

info_log_list = []
warning_log_list = []
error_log_list = []

for log_line in log_list:
    if "INFO" in log_line:
        info_log_list.append(log_line)
    elif "WARNING" in log_line:
        warning_log_list.append(log_line)
    elif "ERROR" in log_line:
        error_log_list.append(log_line)

print(f"INFO Log List: {info_log_list}")
print(f"WARNING Log List: {warning_log_list}")
print(f"ERROR Log List: {error_log_list}")

print(f"INFO Log Count: {len(info_log_list)}")
print(f"WARNING Log Count: {len(warning_log_list)}")
print(f"ERROR Log Count: {len(error_log_list)}")

---

### Example 2: Service Status Classification

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

    running_service_list = []
    stopped_service_list = []
    failed_service_list = []

    for service_status in service_status_list:
        if service_status == "running":
            running_service_list.append(service_status)
        elif service_status == "stopped":
            stopped_service_list.append(service_status)
        elif service_status == "failed":
            failed_service_list.append(service_status)

    print(f"Number of Running Services: {len(running_service_list)}")
    print(f"Number of Stopped Services: {len(stopped_service_list)}")
    print(f"Number of Failed Services: {len(failed_service_list)}")

---

### Example 3: Configuration Classification

    config_line_list = [
        "listen 80;",
        "server_name example.com;",
        "root /usr/share/nginx/html;",
        "listen 443 ssl;"
    ]

    listen_config_list = []
    server_name_config_list = []
    other_config_list = []

    for config_line in config_line_list:
        if "listen" in config_line:
            listen_config_list.append(config_line)
        elif "server_name" in config_line:
            server_name_config_list.append(config_line)
        else:
            other_config_list.append(config_line)

    print(f"listen Configuration List: {listen_config_list}")
    print(f"server_name Configuration List: {server_name_config_list}")
    print(f"Other Configuration List: {other_config_list}")

---

## FifteenI don't know.Day7 Classroom Exercise

Instructions:

**Classroom exercises are for mastering basic classification concepts for the day. Homework assignments are not repeated with classroom exercises and emphasize more on operational scenarios, field splitting, result aggregation, and complete output.**

Below are the practice questions, without reference answers.  
Complete them independently first, then check answers collectively later.

---

### 1I'm not sure.log_level_classify.py

Requirement: Categorize the log list into three types:

- `info_log_list`
- `warning_log_list`
- `error_log_list`

Known variables:

    log_list = [
        "INFO nginx started",
        "WARNING disk usage high",
        "ERROR mysql down",
        "INFO kubelet running",
        "ERROR redis connect failed"
    ]

Output required:

- Three result lists
- Count of each log type

---

### 2I'm not sure.status_code_classify.py

Requirement: Categorize status codes into three types:

- Normal status code list
- Client error status code list
- Server error status code list

Known variables:

    status_code_list = ["200", "404", "500", "200", "502", "503", "404"]

Suggested approach:

- `2xx` for normal list
- `4xx` for client error list
- `5xx` for server error list

Output required:

- Three result lists
- Count of each status code type

Note:

**Pay special attention to the fact that strings and integers cannot be compared together.**

---

### 3I'm not sure.service_status_classify.py

Requirement: Categorize service statuses into three types:

- `running_service_list`
- `stopped_service_list`
- `failed_service_list`

Known variables:

    service_status_list = ["running", "stopped", "failed", "running", "failed", "running"]

Need to output:

- Three result lists
- Count of each of the three service statuses

---

### 4I'm not sure.config_line_classify.py

Requirement: Divide configuration lines into three categories:

- `listen_config_list`
- `server_name_config_list`
- `other_config_list`

Known variables:

    config_line_list = [
        "listen 80;",
        "server_name example.com;",
        "root /usr/share/nginx/html;",
        "listen 443 ssl;",
        "index index.html index.htm;"
    ]

Need to output:

- Three result lists
- Count of each of the three configuration line categories

---

### 5I'm not sure.port_check_result_classify.py

Requirement: Divide port check results into two categories:

- Normal port list
- Abnormal port list

Known variables:

    port_status_list = ["80 open", "443 open", "3306 closed", "6379 open", "22 closed"]

Suggested approach:

- Place entries containing `"open"` in the normal port list
- Place entries containing `"closed"` in the abnormal port list

Need to output:

- Two result lists
- Count of each of the two result categories
- If the number of abnormal ports is greater than 0, output "Abnormal ports found"
- Otherwise, output "No abnormal ports found"

---

## SixteenI don't know.Day7 Class Exercise Reference Answers

### 1I'm not sure.log_level_classify.py

    log_list = [
        "INFO nginx started",
        "WARNING disk usage high",
        "ERROR mysql down",
        "INFO kubelet running",
        "ERROR redis connect failed"
    ]

    info_log_list = []
    warning_log_list = []
    error_log_list = []

    for log_line in log_list:
        if "INFO" in log_line:
            info_log_list.append(log_line)
        elif "WARNING" in log_line:
            warning_log_list.append(log_line)
        elif "ERROR" in log_line:
            error_log_list.append(log_line)

    print(f"Normal log list:{info_log_list}")
    print(f"Normal log count:{len(info_log_list)}")
    print(f"Warning log list:{warning_log_list}")
    print(f"Warning log count:{len(warning_log_list)}")
    print(f"Error log list:{error_log_list}")
    print(f"Error log count:{len(error_log_list)}")

---

### 2I'm not sure.status_code_classify.py

    status_code_list = ["200", "404", "500", "200", "502"]

    normal_list = []
    client_error_list = []
    server_error_list = []
    other_status_code_list = []

    for code in status_code_list:
        if code.startswith("2"):
            normal_list.append(code)
        elif code.startswith("4"):
            client_error_list.append(code)
        elif code.startswith("5"):
            server_error_list.append(code)
        else:
            other_status_code_list.append(code)

    print(f"Normal list:{normal_list}")
    print(f"Normal count:{len(normal_list)}")
    print(f"Client error list:{client_error_list}")
    print(f"Client error count:{len(client_error_list)}")
    print(f"Server error list:{server_error_list}")
    print(f"Server error count:{len(server_error_list)}")
    print(f"Other status code list:{other_status_code_list}")
    print(f"Other status code count:{len(other_status_code_list)}")

---

### 3I'm not sure.service_status_classify.py

    service_status_list = ["running", "stopped", "failed", "running", "failed", "running"]

    running_service_list = []
    stopped_service_list = []
    failed_service_list = []

for service_status in service_status_list:
    if service_status == "running":
        running_service_list.append(service_status)
    elif service_status == "stopped":
        stopped_service_list.append(service_status)
    elif service_status == "failed":
        failed_service_list.append(service_status)

print(f"running_service_list list:{running_service_list}")
print(f"running_service_list count:{len(running_service_list)}")
print(f"stopped_service_list list:{stopped_service_list}")
print(f"stopped_service_list count:{len(stopped_service_list)}")
print(f"failed_service_list list:{failed_service_list}")
print(f"failed_service_list count:{len(failed_service_list)}")

---

### 4I'm not sure.config_line_classify.py

    config_line_list = [
        "listen 80;",
        "server_name example.com;",
        "root /usr/share/nginx/html;",
        "listen 443 ssl;",
        "index index.html index.htm;"
    ]

    listen_config_list = []
    server_name_config_list = []
    other_config_list = []

    for config_line in config_line_list:
        if "listen" in config_line:
            listen_config_list.append(config_line)
        elif "server_name" in config_line:
            server_name_config_list.append(config_line)
        else:
            other_config_list.append(config_line)

    print(f"listen configuration list: {listen_config_list}")
    print(f"listen configuration count: {len(listen_config_list)}")
    print(f"server_name configuration list: {server_name_config_list}")
    print(f"server_name configuration count: {len(server_name_config_list)}")
    print(f"Other configuration list: {other_config_list}")
    print(f"Other configuration count: {len(other_config_list)}")

---

### 5I'm not sure.port_check_result_classify.py

    port_status_list = ["80 open", "443 open", "3306 closed", "6379 open", "22 closed"]

    normal_port_list = []
    abnormal_port_list = []

    for port_status in port_status_list:
        if "open" in port_status:
            normal_port_list.append(port_status)
        else:
            abnormal_port_list.append(port_status)

    print(f"Normal ports:{normal_port_list}")
    print(f"Normal ports count:{len(normal_port_list)}")
    print(f"Abnormal ports:{abnormal_port_list}")
    print(f"Abnormal ports count:{len(abnormal_port_list)}")

    if len(abnormal_port_list) > 0:
        print("Abnormal ports found")
    else:
        print("No abnormal ports found")

---

## Seventeen, Day7 Homework

Notes:

**Homework and class exercises are not repeated, with emphasis on operational scenarios, field splitting, result aggregation, and complete output.**

---

### 1I'm not sure.http_log_classify_homework.py

Requirements:

- Classify access logs by status code
- Split into:
  - `normal_request_list`
  - `client_error_request_list`
  - `server_error_request_list`

Known variables:

    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200",
        "10.0.0.5 GET /admin 403"
    ]

Required output:

- Normal request list
- Client abnormal request list
- Server abnormal request list
- Count for each of the three request types

Hints:

- You can first use `split()` to split each log line
- Status code is at the end of each line
- You can use `startswith("2")`, `startswith("4")`, `startswith("5")` to determine status code classification

File name:

    http_log_classify_homework.py

---

### 2I'm not sure.systemd_service_check_homework.py

Requirements:

- Categorize service inspection results into three types
- Split into:
  - `running_service_list`
  - `stopped_service_list`
  - `failed_service_list`

Known variables:

    service_result_list = [
        "nginx running",
        "mysql failed",
        "docker running",
        "redis stopped",
        "kubelet running"
    ]

Required output:

- Running services list
- Stopped services list
- Failed services list
- Count of each service category

Hints:

- Each item contains service name and status
- Current stage can directly judge whether the string contains `"running"`, `"stopped"`, `"failed"`

Filename:

    systemd_service_check_homework.py

---

### 3I'm not sure.k8s_pod_status_classify_homework.py

Requirements:

- Categorize Pod status into three types
- Split into:
  - `running_pod_list`
  - `pending_pod_list`
  - `failed_pod_list`

Known variables:

    pod_status_list = [
        "Running",
        "Pending",
        "Failed",
        "Running",
        "Pending"
    ]

Required output:

- Running status Pod list
- Pending status Pod list
- Failed status Pod list
- Count of each status category

Hints:

- The status values themselves are explicit
- It's recommended to use `==` for precise judgment

Filename:

    k8s_pod_status_classify_homework.py

---

### 4I'm not sure.inspection_summary_homework.py

Requirements:

- Categorize inspection results into two types
- Split into:
  - `normal_result_list`
  - `abnormal_result_list`

Known variables:

    inspection_result_list = [
        "nginx ok",
        "mysql ok",
        "redis failed",
        "disk warning",
        "kubelet ok"
    ]

Suggested approach:

- Place items containing `"ok"` into normal results list
- Place items containing `"failed"` or `"warning"` into abnormal results list

Required output:

- Normal results list
- Abnormal results list
- Count of normal results
- Count of abnormal results
- If abnormal results count > 0, output: `This patrol has detected an anomaly.`
- Else output: `This inspection is normal.`

Hints:

- This question's focus isn't just on classification
- It also practices the complete structure of "classification + statistics + final conclusion output"

Filename:

    inspection_summary_homework.py

---

## EighteenI don't know.Day7 Post-class Assignment Answers

### 1I'm not sure.http_log_classify_homework.py

    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200",
        "10.0.0.5 GET /admin 403"
    ]

    normal_request_list = []
    client_error_request_list = []
    server_error_request_list = []

    for access_log in access_log_list:
        parts = access_log.split()
        status_code = parts[3]

        if status_code.startswith("2"):
            normal_request_list.append(access_log)
        elif status_code.startswith("4"):
            client_error_request_list.append(access_log)
        elif status_code.startswith("5"):
            server_error_request_list.append(access_log)

    print(f"Normal request list: {normal_request_list}")
    print(f"Normal request count: {len(normal_request_list)}")
    print(f"Client error request list: {client_error_request_list}")
    print(f"Client error request count: {len(client_error_request_list)}")
    print(f"Server error request list: {server_error_request_list}")
    print(f"Server error request count: {len(server_error_request_list)}")

---

### 2I'm not sure.systemd_service_check_homework.py

    service_result_list = [
        "nginx running",
        "mysql failed",
        "docker running",
        "redis stopped",
        "kubelet running"
    ]

    running_service_list = []
    stopped_service_list = []
    failed_service_list = []

```python
for service_result in service_result_list:
    if "running" in service_result:
        running_service_list.append(service_result)
    elif "stopped" in service_result:
        stopped_service_list.append(service_result)
    elif "failed" in service_result:
        failed_service_list.append(service_result)

print(f"Running service list:{running_service_list}")
print(f"Running service count:{len(running_service_list)}")
print(f"Stopped service list:{stopped_service_list}")
print(f"Stopped service count:{len(stopped_service_list)}")
print(f"Failed service list:{failed_service_list}")
print(f"Failed service count:{len(failed_service_list)}")
```

---

### 3I'm not sure.k8s_pod_status_classify_homework.py

```python
pod_status_list = [
    "Running",
    "Pending",
    "Failed",
    "Running",
    "Pending"
]

running_pod_list = []
pending_pod_list = []
failed_pod_list = []

for pod_status in pod_status_list:
    if pod_status == "Running":
        running_pod_list.append(pod_status)
    elif pod_status == "Pending":
        pending_pod_list.append(pod_status)
    elif pod_status == "Failed":
        failed_pod_list.append(pod_status)

print(f"running_pod_list list:{running_pod_list}")
print(f"running_pod_list count:{len(running_pod_list)}")
print(f"pending_pod_list list:{pending_pod_list}")
print(f"pending_pod_list count:{len(pending_pod_list)}")
print(f"failed_pod_list list:{failed_pod_list}")
print(f"failed_pod_list count:{len(failed_pod_list)}")
```

---

### 4I'm not sure.inspection_summary_homework.py

```python
inspection_result_list = [
    "nginx ok",
    "mysql ok",
    "redis failed",
    "disk warning",
    "kubelet ok"
]

normal_result_list = []
abnormal_result_list = []

for inspection_result in inspection_result_list:
    if "ok" in inspection_result:
        normal_result_list.append(inspection_result)
    elif "failed" in inspection_result or "warning" in inspection_result:
        abnormal_result_list.append(inspection_result)

print(f"Normal result list:{normal_result_list}")
print(f"Normal result count:{len(normal_result_list)}")
print(f"Abnormal result list:{abnormal_result_list}")
print(f"Abnormal result count:{len(abnormal_result_list)}")

if len(abnormal_result_list) > 0:
    print("Abnormal issues found in this inspection")
else:
    print("This inspection result is normal")
```

---

## NineteenI don't know.Day7 Problem Occurrences During Homework

### 1I'm not sure.Confusing current element with entire list in `for` loop

Incorrect code:

```python
for log_line in log_list:
    if "INFO" in log_list:
        info_log_list.append(log_line)
```

Correct code:

```python
for log_line in log_list:
    if "INFO" in log_line:
        info_log_list.append(log_line)
```

Explanation:

- `log_list` is the entire list
- `log_line` is the current log line
- When making judgments in loops, you should usually judge "current element", not "entire list"

---

### 2I'm not sure.Forgetting to define empty lists for new categories

Incorrect logic:

```python
else:
    other_status_code_list.append(code)
```

But no:

```python
other_status_code_list = []
```

Correct logic:

```python
other_status_code_list = []
```

```python
for code in status_code_list:
    if code.startswith("2"):
        normal_list.append(code)
    elif code.startswith("4"):
        client_error_list.append(code)
    elif code.startswith("5"):
        server_error_list.append(code)
    else:
        other_status_code_list.append(code)
```

**Note:**  
**Whenever you are about to append to a list, you must have defined the list beforehand.**

---

### 3) Mixing Strings and Integers in Comparisons

**Incorrect way:**

```python
if status_code == 200:
    print("Normal")
```

If the list contains strings:

```python
status_code_list = ["200", "404", "500"]
```

The correct way should be:

```python
if status_code == "200":
    print("Normal")
```

**Note:**  
**Data types must be consistent.**  
Strings should be compared with strings, integers with integers.

---

### 4) Writing Multiple Classification Conditions as Separate `if`

**Not recommended:**

```python
if "ERROR" in log_line:
    error_log_list.append(log_line)

if "WARNING" in log_line:
    warning_log_list.append(log_line)
```

**Recommended:**

```python
if "INFO" in log_line:
    info_log_list.append(log_line)
elif "WARNING" in log_line:
    warning_log_list.append(log_line)
elif "ERROR" in log_line:
    error_log_list.append(log_line)
```

**Note:**  
Classification scenarios are typically mutually exclusive. At this stage, it's recommended to use `if / elif / else`.

---

### 5) Forgetting to Count After Categorization

**Incomplete:**

```python
print(error_log_list)
```

**Complete:**

```python
print(error_log_list)
print(len(error_log_list))
```

**Note:**  
Day7 isn't just about separating results, but also about performing classification statistics.

---

### 6) Using Inadequate Judgment for Specific Status Values

For example:

```python
if "running" in service_status:
    running_service_list.append(service_status)
```

If `service_status` is itself:

```python
"running"
"stopped"
"failed"
```

The recommended way is:

```python
if service_status == "running":
    running_service_list.append(service_status)
```

**Note:**  
- Use `in` to find keywords in an entire line of text  
- Use `==` for exact comparisons of single specific values

---

### 7) Not Splitting and Extracting Fields First for Fixed-Format Text

For example, access logs:

```python
"10.0.0.2 GET /login 404"
```

If you want to extract the status code, you can't just look at the entire line. It's better to split first:

```python
parts = access_log.split()
status_code = parts[3]
```

**Note:**  
The key to this type of question isn't just being able to judge strings, but learning to:  
- Split first  
- Extract fields  
- Classify by fields  

This approach is more aligned with realTransport scripts.

---

## Twenty, What Abilities Will You Gain After Day7

After completing Day7, you will gain these additional abilities compared to Day6:

### 1) Multi-Condition Classification

The same batch of data can be divided into different lists according to different rules.

### 2) More Complete Batch Result Output

Output isn't just a single result list, but multiple classified results.

### 3) Closer to Inspection Script Thinking

You begin to develop an automated awareness of "classification and summarization".

### 4) Starting to Understand Field Splitting

You know to use `split()` to split strings first, then extract fields for judgment.

### 5) Foundation for Future Functions

When learning functions later, you can encapsulate "classification logic", "status check logic", and "log processing logic" separately.

---

## Twenty-One, What Will Day8 Likely Cover

If you master Day7 well, Day8 is very suitable for entering:

- Function basics  
- Defining functions  
- Calling functions  
- Parameters  
- Return values  
- Simply encapsulating the classification logic you've written before

This way, the introduction of functions will be more natural and not too abrupt.

---

## Twenty-Two, External Links

### Python Official Documentation

- Python Tutorial: Control Flow Tools  
  https://docs.python.org/3/tutorial/controlflow.html

- Python Tutorial: Data Structures  
  https://docs.python.org/3/tutorial/datastructures.html

### Beginner References

- Runoob Tutorial: Python3 Conditional Control  
  https://www.runoob.com/python3/python3-if-statement.html

- Runoob Tutorial: Python3 Loop Statements  
  https://www.runoob.com/python3/python3-loop.html

- Runoob Tutorial: Python3 Lists  
  https://www.runoob.com/python3/python3-list.html

- Runoob Tutorial: Python3 String split() Method  
  https://www.runoob.com/python/att-string-split.html

### Operations Understanding Extensions

- Nginx Configuration Basics  
  https://nginx.org/en/docs/

- Kubernetes Documentation  
  https://kubernetes.io/docs/home/

---

## Twenty-Three, Today's One-Sentence Summary

**The essence of Day7 is to upgrade Python from "collecting a category of results and checking if they exist" to "classifying and statistically outputting more complete inspection results for the same batch of data according to different conditions", and beginning to understand the basic processing approach of "splitting fields first and then judging by fields".**