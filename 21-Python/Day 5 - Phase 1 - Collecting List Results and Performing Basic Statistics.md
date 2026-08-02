# Day5 - Phase 1 - Collecting List Results and Performing Basic Statistics

# Python # Python Learning # Operations and Maintenance Development # Lists # Append # Counting # Log Analysis # Service Checking # Port Checking # Configuration Checking # Linux # Kubernetes # Obsidian

## Today's Focus

Learning content from Day 1:

- Variables
- Strings
- `print()`
- `f-string`
- `in`
- `strip()`
- `startswith()`
- `endswith()`

Learning content from Day 2:

- `if / elif / else`
- Multi-branch judgments
- `upper()` / `lower()`
- `replace()`
- `split()`

Learning content from Day 3:

- `and`
- `or`
- `not`
- Multi-condition combined judgments
- `in`, `not in`
- `startswith()`, `endswith()`

Learning content from Day 4:

- Lists `list`
- Indexing
- `for` loops
- Traversing lists
- Using `for + if` for basic batch judgments

The new phase starting with Day 5 is:

**Not just “batch traversing and judging”, but also “batch filtering, collecting results, and performing basic statistics”**

This step is very important because in real operations and maintenance scripts, it's often not the case that an exception is detected and the process ends immediately. Instead, it's necessary to:

- Collect abnormal logs
- Gather file names that meet certain criteria
- Count how many times exceptions occur
- Save the filtered results for subsequent scripts to process

Therefore, the core of Day 5 is to begin developing a second layer of automated thinking:

**Programs should not only be able to perform batch checks but also save the results of these checks.**

---

## Today's Goals

After completing Day 5, you should be able to:

1. Understand what result collection means
2. Use `append()` to add elements to a list
3. Filter data that meets certain conditions within a loop
4. Save data that meets criteria into a new list
5. Perform basic counting tasks
6. Write basic scripts for “batch checking + result collection + statistical output”
7. Recognize the differences between Day 4 and Day 5 skills

---

## I. What Will Be Learned Today

The content of Day 5 can be summarized as:

1. Using `append()` to add elements
2. Filtering results and saving them in a new list
3. Counting variables
4. Combining list and string judgments
5. Outputting results after batch checking
6. Applying the concept of result collection in operations and maintenance
7. Practical exercises for Day 5

---

## II. Why Is Day 5 Important

Most of the code from Day 4 looks like this:

- Traverse a set of objects
- Check if they meet certain conditions
- Print the results immediately

For example:

    log_list = [
        "INFO nginx started",
        "ERROR mysql connection failed",
        "WARNING disk usage high"
    ]

    for log_line in log_list:
        if "ERROR" in log_line:
            print("Error log found:", log_line)

This is already good, but it's not enough.

In real-world scenarios, you often encounter needs such as:

- Identifying all error logs and outputting them together
- Finding all `.log` files for further processing
- Identifying all kube-related services for subsequent checks
- Counting the total number of exceptions found
- Only taking action if the number of exceptions is greater than 0

This means that:

**Programs cannot just “print each item they encounter”; they must also “save the results first.”**

This is the significance of Day 5.

---

## III. What Is Result Collection

Result collection can be understood as:

**Putting the filtered content into a new list for later use.**

For example:

    error_log_list = []

This list starts out empty, indicating that no error logs have been collected yet.

During the loop, if any error logs are found, they are added to this list.

After the loop ends, this list will contain all the error results.

---

## IV. What Is an Empty List

An empty list is:

**A container that is initially prepared and later used to store filtered results.**

For example:

    error_log_list = []
    log_file_list = []
    kube_service_list = []

They currently have no content, but new elements can be added to them later.

---

## V. What Is `append()`?

`append()` can be understood as:

**Adding a new element at the end of a list.**

For example:

    error_log_list = []
    error_log_list.append("ERROR mysql connection failed")
    error_log_list.append("ERROR kubelet not ready")

After execution, this list will become:

    ["ERROR mysql connection failed", "ERROR kubelet not ready"]

This is a very```
ERROR mysql connection failed,
WARNING disk usage high,
ERROR kubelet not ready
]
```

### 2）`log_line`

Indicates the log line being processed in the current iteration of the loop.

During loop execution, it will change as follows:

First iteration:

```
log_line = "INFO nginx started"
```

Second iteration:

```
log_line = "ERROR mysql connection failed"
```

Third iteration:

```
log_line = "WARNING disk usage high"
```

Fourth iteration:

```
log_line = "ERROR kubelet not ready"
```

So, it's important to remember:

- `log_list` contains the entire batch of data.
- `log_line` refers to the specific log line being processed in the current iteration.

This point is particularly crucial on Day5.

---

## XIV. The Core Process of Day5

The core process of Day5 can be summarized in one sentence:

**First, extract the relevant information; then determine whether it meets the criteria; finally, collect or count it.**

In code terms, this would look like:

```python
error_log_list = []

for log_line in log_list:
    if "ERROR" in log_line:
        error_log_list.append(log_line)
```

The execution sequence is as follows:

### Step 1: Extract the relevant information

```python
for log_line in log_list:
    # Iterate through each element in the original list.
```

### Step 2: Determine whether it meets the criteria

```python
if "ERROR" in log_line:
    # Check if the current log line contains the word "ERROR".
```

### Step 3: Collect or count it

```python
error_log_list.append(log_line)
    # If the criteria are met, add the log line to the result list.
```

If the criteria are satisfied, the log line is added to the `error_log_list`. To count the number of occurrences, the code would look like this:

```python
error_count = error_count + 1
    # Increase the counter by 1 if the condition is true.
```

---

## XV. Basic Examples of Data Collection

### 1) Collecting Error Logs

```python
log_list = [
    "INFO nginx started",
    "ERROR mysql connection failed",
    "WARNING disk usage high",
    "ERROR kubelet not ready"
]
error_log_list = []

for log_line in log_list:
    if "ERROR" in log_line:
        error_log_list.append(log_line)
print(error_log_list)
```

---

### 2) Collecting `.log` Files

```python
file_list = ["syslog.log", "nginx.conf", "message.log", "hosts"]
log_file_list = []

for file_name in file_list:
    if file_name.endswith(".log"):
        log_file_list.append(file_name)
print(log_file_list)
```

---

### 3) Collecting Kube-related Services

```python
service_list = ["kube-apiserver", "nginx", "kubelet", "docker", "kube-proxy"]
kube_service_list = []

for service_name in service_list:
    if service_name.startswith("kube"):
        kube_service_list.append(service_name)
print(kube_service_list)
```

---

### 4) Collecting Non-200 Status Codes

```python
status_code_list = ["200", "200", "500", "404", "200", "502"]
abnormal_status_code_list = []

for status_code in status_code_list:
    if status_code != "200":
        abnormal_status_code_list.append(status_code)
print(abnormal_status_code_list)
```

---

## XVI. Basic Examples of Counting

### 1) Counting the Number of Error Logs

```python
log_list = [
    "INFO nginx started",
    "ERROR mysql connection failed",
    "WARNING disk usage high",
    "ERROR kubelet not ready"
]
error_count = 0

for log_line in log_list:
    if "ERROR" in log_line:
        error_count = error_count + 1
print(error_count)
```

---

### 2) Counting the Number of Kube-related Services

```python
service_list = ["kube-apiserver", "nginx", "kubelet", "docker", "kube-proxy"]
kube_service_count = 0

for service_name in service_list:
    if service_name.startswith("kube"):
        kube_service_count = kube_service_count + 1
print(kube_service_count)
```

---

### 3) Counting the Number of Abnormal Status Codes

```python
status_code_list = ["200", "200", "500", "404", "200", "502"]
abnormal_code_count = 0

for status_code in status_code_list:
    if status_code != "200":
        abnormal_code_countservice_list.append("nginx")
service_list.append("kubelet")

print(service_list)

---

### 2）error_log_collect.py

Requirement: Collect all logs that contain the word `ERROR`.

log_list = [
    "INFO nginx started",
    "ERROR mysql connection failed",
    "WARNING disk usage high",
    "ERROR kubelet not ready"
]

error_log_list = []

for log_line in log_list:
    if "ERROR" in log_line:
        error_log_list.append(log_line)

print(error_log_list)

---

### 3）log_file_collect.py

Requirement: Collect all file names that end with the extension `.log`.

file_list = ["syslog.log", "nginx.conf", "message.log", "hosts"]
log_file_list = []

for file_name in file_list:
    if file_name.endswith(".log"):
        log_file_list.append(file_name)

print(log_file_list)

---

### 4）kube_service_collect.py

Requirement: Collect all service names that start with the prefix `kube`.

service_list = ["kube-apiserver", "nginx", "kubelet", "docker", "kube-proxy"]
kube_service_list = []

for service_name in service_list:
    if service_name.startswith("kube"):
        kube_service_list.append(service_name)

print(kube_service_list)

---

### 5）status_code_collect.py

Requirement: Collect all status codes that are not `200`.

status_code_list = ["200", "200", "500", "404", "200", "502"]
abnormal_status_code_list = []

for status_code in status_code_list:
    if status_code != "200":
        abnormal_status_code_list.append(status_code)

print(abnormal_status_code_list)

---

### 6）error_log_count.py

Requirement: Count the number of logs that contain the word `ERROR` in the given list.

log_list = [
    "INFO nginx started",
    "ERROR mysql connection failed",
    "WARNING disk usage high",
    "ERROR kubelet not ready"
]

error_count = 0

for log_line in log_list:
    if "ERROR" in log_line:
        error_count = error_count + 1

print(error_count)

---

### 7）kube_service_count.py

Requirement: Count the total number of service names that start with the prefix `kube`.

service_list = ["kube-apiserver", "nginx", "kubelet", "docker", "kube-proxy"]
kube_service_count = 0

for service_name in service_list:
    if service_name.startswith("kube"):
        kube_service_count = kube_service_count + 1

print(kube_service_count)

---

### 8）error_log_collect_and_count.py

Requirement: Collect error logs and count their number at the same time.

log_list = [
    "INFO nginx started",
    "ERROR mysql connection failed",
    "WARNING disk usage high",
    "ERROR kubelet not ready"
]

error_log_list = []
error_count = 0

for log_line in log_list:
    if "ERROR" in log_line:
        error_log_list.append(log_line)
        error_count = error_count + 1

print("Error Log List:", error_log_list)
print("Number of Error Logs:", error_count)

---

## Chapter 22: Day5 Homework and Answers

It is recommended to try these problems on your own first, and then check the answers.

Recommended order:

1. Create a practice file first.
2. Write the code independently as required.
3. Run the tests.
4. Then check the answers.

---

### Homework 1: Collect Error Logs

#### Suggested Practice File Name

    error_log_collect_homework.py

#### Given Variables

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed",
        "INFO kubelet running"
    ]

#### Requirements

Define an empty list.
Use a `for` loop to iterate through each log line.
If the log line contains the word `ERROR`, add it to the new list.
Finally, print the error log list.

#### Answer

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed",
        "INFO kubelet running"
    ]

    error_log_list = []

    for log_line in log_list:
        if "ERROR" in log_line:
            error_log_list.append(log_line)

    print(error_log_list)

---

### Homework 2: Collect `.log` File Names

#### Suggested Practice File Name

    log_file_collect_homework.py

#### Given Variables

    file_list = ["syslog.log", "nginx.conf", "message.log", "hosts", "audit.log"]

 #### Requirements

Define an empty list.
Use a `for` loop to iterate through each file name.
If the file name ends with `.log`, add it to the newif service_name.startswith("kube"):
    kube_service_list.append(service_name)

print(kube_service_list)if "ERROR" in log_line:
    error_log_list.append(log_line)