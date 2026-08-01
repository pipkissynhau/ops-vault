# Day4 - First Stage - Lists and for Loops Introduction

#Python #PythonLearning #TransportDevelopment #List #ForCirculation #BatchProcessing #LogAnalysis #Kubernetes #Linux #Obsidian

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
- Multi-branch Judgment
- `upper()` / `lower()`
- `replace()`
- `split()`

Day3 Learning Content:

- `and`
- `or`
- `not`
- Multi-condition Combination Judgment
- `in`I don't know.`not in`
- `startswith()`I don't know.`endswith()`

Day4 Enters a Very Critical New Stage:

**From "Processing a Single Value" to "Batch Processing Multiple Values"**

This is an important milestone where Python begins to demonstrate its automation capabilities.

In real operations scenarios, it's rare to process only one object; more often you need to:

- Check the status of multiple Pods at once
- Check multiple ports at once
- Analyze multiple log lines at once
- Process multiple filenames at once
- Traverse multiple nodes at once

Therefore, the core of Day4 is not just understanding a few new syntaxes, but beginning to establish a new way of thinking:

**Let the program automatically repeat the same action many times**

---

## Today's Goals

After completing Day4, you should be able to:

1. Understand what a list `list` is
2. Define a list
3. View the contents of a list
4. Retrieve elements from a list using an index
5. Understand what a `for` loop is
6. Use a `for` loop to traverse a list
7. Write the most basic batch processing script
8. Establish the mindset of "repeating the same operation on multiple objects"

---

## One: What Will We Learn Today

Day4 Content:

1. Lists `list`
2. Elements in a list
3. Indexes (indices)
4. `for` Loop
5. Traversing a list
6. Batch processing scenarios in operations
7. Day4 Practical Exercise

---

## Two: Why Day4 Is Important

From Day1 to Day3, most code is doing things like:

- Judging the status of a single Pod
- Judging a single filename
- Judging a single log line
- Judging a single port

But in real operations, it's often not just processing one item, but a batch:

- A batch of Pods
- A batch of logs
- A batch of nodes
- A batch of ports
- A batch of service names

If you still write them one by one manually, it will quickly become unmanageable:

    pod1 = "Running"
    pod2 = "Pending"
    pod3 = "CrashLoopBackOff"

This approach becomes unsuitable for maintenance when the number of objects increases.

So the significance of Day4 is:

**Putting multiple values into a container and processing them in batches**

This container is `list`.  
The action of "batch processing" mainly relies on `for` loop.

---

## Three: What Is a List `list`

A list can be understood as:

**Putting multiple values in order into a single variable**

For example:

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

This variable no longer contains a single value, but multiple values.

### You Can Understand It This Way

- String: Stores a piece of text
- Integer: Stores a single number
- List: Stores multiple values

---

## Four: What Does a List Look Like

The most obvious feature of a list is:

- Using square brackets `[]`
- Elements are separated by commas

For example:

    ports = [22, 80, 443]
    pod_list = ["pod-a", "pod-b", "pod-c"]
    log_levels = ["INFO", "WARNING", "ERROR"]

---

## Five: Elements in a List

Each item in a list is called an "element."

For example:

    ports = [22, 80, 443]

There are 3 elements here:

- 22
- 80
- 443

Another example:

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

There are also 3 elements here:

- `"Running"`
- `"Pending"`
- `"CrashLoopBackOff"`

---

## Six: What Can a List Store

A list can store many types of data.

### 1) Storing Strings

    services = ["nginx", "docker", "kubelet"]

### 2) Storing Numbers

    ports = [22, 80, 443]

### 3) Keep Data of the Same Type in Early Stages

It's recommended to keep it simple:

- Status lists should store statuses
- Port lists should store ports
- Log lists should store log lines

This makes it easier to understand and maintain.

---

## Seven: Printing the Entire List

You can directly print the entire list:

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]
    print(pod_status_list)

The output will look like this:

    ['Running', 'Pending', 'CrashLoopBackOff']

This indicates Python displays the entire list as is.

---

## Eight: Retrieving Elements by Index

Elements in a list are ordered.  
Python retrieves elements using "indexes."

Note:

**Python indexes start at 0, not 1.**

For example:

    ports = [22, 80, 443]

The corresponding relationship is:

- `ports[0]` → 22
- `ports[1]` → 80
- `ports[2]` → 443

Example:

    ports = [22, 80, 443]

    print(ports[0])
    print(ports[1])
    print(ports[2])

### Additional Notes

If a list has only 3 elements, the maximum index is `2`.  
If you write an index that doesn't exist, such as `ports[3]`, the program will throw an error.

---

## Nine: Why Indexes Start at 0

This might feel unfamiliar at first, but it's not necessary to dwell on the principle now.  
Just remember:

**The first element's index is 0**

For example:

    pod_list = ["pod-a", "pod-b", "pod-c"]

- `pod_list[0]` is `"pod-a"`
- `pod_list[1]` is `"pod-b"`
- `pod_list[2]` is `"pod-c"`

---

## 10. What is `for` Loop

`for` Loop can be understood as:

**Take elements from the list one by one, then repeat the same operation**

This is the most important ability on Day4.

For example:

    ports = [22, 80, 443]

    for port in ports:
        print(port)

This code means:

- Take elements from `ports`
- Take one each time
- Assign to variable `port`
- Then execute `print(port)`

Finally it will print sequentially:

    22
    80
    443

---

## 11. What does `for` Loop syntax look like

Basic structure:

    for variable in list:
        code to repeat

You don't need to memorize syntax terms now, just remember this meaning:

**Take elements from the list one by one, execute the code below once for each element**

---

## 12. What is a loop variable

In this code:

    for port in ports:
        print(port)

- `ports` is a group of ports
- `port` is each port taken out

In other words:

- List variable represents "a batch of data"
- Loop variable represents "the current single data"

When writing code later, try to clearly distinguish between these two.

For example:

- `ports` / `port`
- `pod_list` / `pod`
- `log_list` / `log_line`

---

## 13. Indentation is very important

The code to be executed under `for` loop must be indented.

For example:

    ports = [22, 80, 443]

    for port in ports:
        print(port)

Here:

    print(port)

Belongs to `for` loop because it's indented.

If not indented, Python will throw an error or the logic won't match expectations.

---

## 14. The most basic traversal examples

### 1) Traversing Pod status

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

    for status in pod_status_list:
        print(status)

### 2) Traversing ports

    ports = [22, 80, 443]

    for port in ports:
        print(port)

### 3) Traversing log levels

    log_levels = ["INFO", "WARNING", "ERROR"]

    for level in log_levels:
        print(level)

---

## 15. `for` Loop and `if` Combination

The real value of Day4 is not just printing lists,  
but starting to put Day2 and Day3's judgment logic into batch processing.

For example:

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

    for status in pod_status_list:
        if status != "Running":
            print("Found abnormal Pod status:", status)

This code means:

- Check each Pod status
- Print if it's not `Running`
- This is already very close to real operations scenarios

---

## 16. The operational significance of Day4

After Day4, script capabilities will start to significantly improve.

Because it can go from:

- Judging a single Pod

To:

- Judging a batch of Pods

From:

- Judging a single port

To:

- Judging a group of ports

From:

- Looking at a single log line

To:

- Scanning many log lines

This is one of the basic capabilities of automation.

---

## 17. Today's hands-on recommendations

Recommend creating a practice directory in Ubuntu and completing these sample scripts:

    mkdir -p ~/python-study/day4
    cd ~/python-study/day4

Recommended practice files:

    list_basic.py
    list_index_demo.py
    pod_list_print.py
    port_list_print.py
    pod_status_batch_check.py
    log_level_batch_check.py

Execution method:

    python3 filename.py

The focus of this section is to observe the following points:

- Whether the list is properly defined
- Whether `for` loop syntax is correct
- Whether indentation is correct
- Whether judgment logic is inside the loop
- Whether output results match expectations

---

## 18. Day4 Sample Exercises

These can be used first to understand lists and `for` loops.

### 1) list_basic.py

Requirement: Define a list and print the entire list.

    pod_list = ["pod-a", "pod-b", "pod-c"]
    print(pod_list)

---

### 2) list_index_demo.py

Requirement: Get elements from the list using indexes.

    ports = [22, 80, 443]

    print(ports[0])
    print(ports[1])
    print(ports[2])

---

### 3) pod_list_print.py

Requirement: Use `for` loop to print multiple Pod names.

    pod_list = ["nginx-pod", "redis-pod", "mysql-pod"]

    for pod in pod_list:
        print(pod)

---

### 4) port_list_print.py

Requirement: Use `for` loop to print multiple ports.

    ports = [22, 80, 443]

    for port in ports:
        print(port)

---

### 5) pod_status_batch_check.py

Requirement: Batch check Pod statuses and output abnormal statuses.

pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

for status in pod_status_list:
    if status != "Running":
        print("Found abnormal status: ", status)

---

### 6) log_level_batch_check.py

Requirement: Batch identify log levels that require attention.

    log_levels = ["INFO", "WARNING", "ERROR"]

    for level in log_levels:
        if level == "WARNING" or level == "ERROR":
            print("Log level requiring attention: ", level)

---

## NineteenI don't know.Day4 Assignment (with Reference Answers)

The following exercises are recommended to complete independently first, then check against the reference answers.

Recommended order:

1. First create the practice file
2. Independently write the code as required
3. Run the test
4. Check against the reference answer and correct any errors

This approach makes it easier to truly master the core capabilities of Day4.

---

### Assignment 1: Batch print node names

#### Recommended practice file name

    node_list_print.py

#### Reference answer file name

    node_list_print_answer.py

#### Known variable

    node_list = ["master-1", "node-1", "node-2"]

#### Requirement

Use `for` loop to print each node name on a separate line.

#### Reference Answer

    node_list = ["master-1", "node-1", "node-2"]

    for node in node_list:
        print(node)

---

### Assignment 2: Batch check Pod status

#### Recommended practice file name

    pod_status_check.py

#### Reference answer file name

    pod_status_check_answer.py

#### Known variable

    pod_status_list = ["Running", "Pending", "Running", "CrashLoopBackOff"]

#### Requirement

Use `for` loop to iterate through all statuses.  
If the status is not `Running`, output:

    Found abnormal Pod status: status value

#### Reference Answer

    pod_status_list = ["Running", "Pending", "Running", "CrashLoopBackOff"]

    for pod_status in pod_status_list:
        if pod_status != "Running":
            print(f"Found abnormal Pod status: {pod_status}")

---

### Assignment 3: Batch identify Web ports

#### Recommended practice file name

    web_port_check.py

#### Reference answer file name

    web_port_check_answer.py

#### Known variable

    ports = [22, 80, 443, 3306]

#### Requirement

Use `for` loop to iterate through all ports.  
If the port is 80 or 443, output:

    Web service port: port value

#### Reference Answer

    ports = [22, 80, 443, 3306]

    for port in ports:
        if port == 80 or port == 443:
            print(f"Web service port: {port}")

---

### Assignment 4: Batch identify error logs

#### Recommended practice file name

    error_log_batch_check.py

#### Reference answer file name

    error_log_batch_check_answer.py

#### Known variable

    log_list = [
        "INFO nginx started",
        "WARNING disk usage high",
        "ERROR mysql connection failed",
        "INFO kubelet running"
    ]

#### Requirement

Use `for` loop to iterate through each log line.  
If the log contains `ERROR`, output:

    Found error log: log content

#### Reference Answer

    log_list = [
        "INFO nginx started",
        "WARNING disk usage high",
        "ERROR mysql connection failed",
        "INFO kubelet running"
    ]

    for log_line in log_list:
        if "ERROR" in log_line:
            print(f"Found error log: {log_line}")

---

### Assignment 5: Batch filter `.log` files

#### Recommended practice file name

    log_file_filter.py

#### Reference answer file name

    log_file_filter_answer.py

#### Known variable

    file_list = ["syslog.log", "nginx.conf", "message.log", "hosts"]

#### Requirement

Use `for` loop to iterate through each filename.  
If the filename ends with `.log`, output:

    Log file: filename

#### Reference Answer

    file_list = ["syslog.log", "nginx.conf", "message.log", "hosts"]

    for file_name in file_list:
        if file_name.endswith(".log"):
            print(f"Log file: {file_name}")

---

### Assignment 6: Batch identify kube services

#### Recommended practice file name

    kube_service_filter.py

#### Reference answer file name

    kube_service_filter_answer.py

#### Known variable /think

service_list = ["kube-apiserver", "nginx", "kubelet", "docker"]

#### Requirements

Use `for` to iterate through each service name.  
If the service name starts with `kube`, output:

    Kubernetes-related service: service name

#### Reference Answer

    service_list = ["kube-apiserver", "nginx", "kubelet", "docker"]

    for service in service_list:
        if service.startswith("kube"):
            print(f"Kubernetes-related service: {service}")

---

### Notes When Using the Reference Answer

The reference answer's purpose is not direct copying, but to help check these key points:

1. Whether the list is defined correctly
2. Whether the `for` loop syntax is correct
3. Whether indentation is correct
4. Whether the judgment logic is written inside the loop
5. Whether the output content meets the requirements

---

## Twenty, Day4 Common Error Examples

This section is very important.

On Day4, the most common mistakes for beginners are not failing to write lists, but:

- Confusing the concept of a list with a single value
- `for` loop indentation errors
- Not naming the loop variable
- Thinking that `for` loops can only print, not judge
- Index errors causing errors

---

### 1) Treating the entire list as a single value

#### Example

    ports = [22, 80, 443]
    print(ports)

This will print the entire list, not one by one.

If you want to process each item individually, use `for` loop:

    for port in ports:
        print(port)

---

### 2) `for` Loop Not Indented

#### Error Example

    ports = [22, 80, 443]

    for port in ports:
    print(port)

#### Correct Example

    ports = [22, 80, 443]

    for port in ports:
        print(port)

#### Explanation

`for` The code to be executed below must be indented.

---

### 3) Loop variable and list variable have the same name, leading to poor readability

#### Not Recommended Writing

    ports = [22, 80, 443]

    for ports in ports:
        print(ports)

#### More Recommended Writing

    ports = [22, 80, 443]

    for port in ports:
        print(port)

#### Explanation

It is recommended to name as follows:

- List names use plural or include `_list`
- Single elements use singular

For example:

- `ports` / `port`
- `pod_list` / `pod`
- `log_list` / `log_line`

---

### 4) Index error

#### Error Example

    ports = [22, 80, 443]
    print(ports[3])

#### Correct Understanding

This list has 3 elements, with indexes:

- `0`
- `1`
- `2`

So `ports[3]` will cause an error.

---

### 5) Only looping to print, not adding judgment inside the loop

#### Basic Writing

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

    for status in pod_status_list:
        print(status)

#### More Close to Real-World Operations

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

    for status in pod_status_list:
        if status != "Running":
            print("Abnormal status: ", status)

#### Explanation

The key for Day4 is not "knowing how to iterate", but:

**Making judgments while iterating**

---

## Twenty-One, Day4 Must Master Core Thinking

By Day4, you should start building this mindset:

### Day1~ I'm sorry.Day3 Thinking

- Processing a single value
- Judging a single value
- Outputting a single result

### Day4 Thinking

- Putting multiple values into a list
- Using `for` to retrieve them one by one
- Applying the same logic to each value

This step is very important because many automation scripts' basic form is:

1. Prepare a batch of data
2. Loop through them
3. Output or operate on matching content

---

## Twenty-Two, Day4 Learning Summary

The core of Day4 is not just learning lists and `for` loops,  
but the first time truly touching "batch processing".

This means the way of writing scripts starts to change:

- No longer focusing on a single value
- Starting to consider a batch of objects

For example:

- A batch of Pods
- A batch of logs
- A batch of nodes
- A batch of ports
- A batch of service names

This is the beginning of automation thinking.

---

## Twenty-Three, What Capabilities Are Gained After Day4

By now, you have the following capabilities:

### 1) Defining lists

Putting multiple values into a single variable for unified management.

### 2) Reading list elements

Knowing the list has order and indexes start from 0.

### 3) Using `for` loops

Retrieving elements from the list one by one.

### 4) Performing basic batch processing

For example:

- Batch printing
- Batch judgment
- Batch filtering matching content

### 5) Closer to real-worldTransport scripts

Because mostTransport operations essentially are:

- Getting a batch of objects
- Checking each one
- Finding abnormal items

---

## Twenty-Four, What Day5 Will Cover

If Day4 is mastered, Day5 is suitable for entering:

- Combining lists with strings
- Statistics counting
- `append()` appending elements
- Collecting results after batch filtering
- Closer to real log analysis and batch check scripts

That is, upgrading from:

- Iterating and printing

To:

- Iterating, judging, filtering, collecting

---

## Twenty-Five, External Links

### Python Official Documentation

- Python Tutorial: Data Structures  
  https://docs.python.org/3/tutorial/datastructures.html

- Python Tutorial: Control Flow Tools  
  https://docs.python.org/3/tutorial/controlflow.html

### Beginner References

- Runoob Tutorial: Python3 List  
  https://www.runoob.com/python3/python3-list.html

- Runoob Tutorial: Python3 Loop Statements  
  https://www.runoob.com/python3/python3-loop.html

---

## 26. Today's One-Sentence Summary

**Day4's essence is upgrading Python from "handling a single value" to "batch processing multiple values," which is the important starting point for automated script thinking.**