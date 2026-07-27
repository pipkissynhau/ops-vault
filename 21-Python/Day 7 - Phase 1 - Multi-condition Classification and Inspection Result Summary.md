# Day7 - Phase 1 - Multi-condition Classification and Inspection Result Summary

#Python #Python Learning #Ops Development #for Loop #if Statement #elif #List #append #len #split #Log Analysis #Status Code #Service Check #Inspection Script #Linux #Kubernetes #Obsidian

## Today's Focus

Learning content from Day1:

- Variables
- Strings
- `print()`
- `f-string`
- `in`
- `strip()`
- `startswith()`
- `endswith()`

Learning content from Day2:

- `if / elif / else`
- Multi-branch judgments
- `upper()` / `lower()`
- `replace()`
- `split()`

Learning content from Day3:

- `and`
- `or`
- `not`
- Multi-condition combined judgments
- `in`
- `not in`
- `startswith()`
- `endswith()`

Learning content from Day4:

- List `list`
- Indexing
- `for` loop
- Iterating through lists
- Using `for + if` for basic batch processing

Learning content from Day5:

- Result collection
- `append()`
- Basic counting
- Batch filtering
- Collecting results and outputting statistics

Learning content from Day6:

- Using `len()` to count lengths
- Determining the number of result lists
- Checking for empty lists
- Scripting mindset of "continuing processing after collecting results"

The new phase starting in Day7 is:

**Not just filtering out one type of result, but classifying the same set of data based on different conditions and outputting more comprehensive inspection results.**

This step is crucial because in real ops, it's common to go beyond simply checking:

- Whether there are any exceptions
- Whether the quantity is greater than 0

But also to further classify, for example:

- Identifying which logs are `INFO`, `WARNING`, or `ERROR`
- Determining which status codes indicate normal conditions, client errors, or server errors
- Differentiating between services that are `running`, `stopped`, or `failed`

Therefore, the core of Day7 is to begin establishing a fourth level of automation thinking:

**Programs should not only be able to find and count results but also classify them and produce content that resembles an inspection report.**

---

## Today's Goals

After completing Day7, you should be able to:

1. Understand the significance of multi-condition classification
2. Use `if / elif / else` within a loop
3. Assign the same set of data to different result lists
4. Count the number of elements in multiple result lists separately
5. Produce more comprehensive batch inspection results
6. Write basic scripts that perform "classification + statistics + result display"
7. Prepare for future function encapsulation

---

## I. What Will Be Learned Today

Day7 covers the following topics:

1. Multi-condition classification judgments
2. Using `if / elif / else` in batch processing
3. Splitting a single original list into multiple result lists
4. Counting the number of elements in multiple result lists separately
5. Producing more complete result outputs
6. Approaching a structure closer to that of an inspection script
7. Classroom exercises
8. Homework

---

## II. Why Day7 Is Important

Most scripts from Day6 are structured like this:

    error_log_list = []

    for log_line in log_list:
        if "ERROR" in log_line:
            error_log_list.append(log_line)

    print(error_log_list)
    print(len(error_log_list))

    if len(error_log_list) > 0:
        print("Error logs found")
    else:
        print("No error logs found")

This approach can already perform basic filtering, but real-world scenarios require further steps such as:

- Separating error logs from alert logs and normal logs
- Distinguishing between normal status codes and exceptional ones
- Differentiating between running services and failed services
- Classifying different types of configuration lines
- Finally, outputting both the classified results and statistical data

This means that:

**Scripts must not only be able to "filter out one type" but also "classify data based on multiple conditions."**

This is the purpose of Day7.

---

## III. The Difference Between Day7 and Day6

### Core of Day6

- Collecting one type of result
- Counting its quantity
- Determining if any results exist

### Core of Day7

- Handling multiple types of results simultaneously
- Separating different types of data for collection
- Counting each type separately
- Producing more comprehensive classified results

Another way to put it:

### Day6 is more about

"Whether there are any exceptions?"

### Day7 is more about

"Which specific exceptions? What alerts? How many normal cases? And how many of each?"

---

## IV. The Most Critical Syntax in Day7Statistically analyze them separately.

Because this is more like the output of an inspection.

---

## Eleven, Operational Thinking Based on Configuration Classification

The same applies to configuration checks.  
For example, in Nginx configuration lines, you might want to classify them as follows:

- `listen`
- `server_name`
- Other configurations

This way, you can clearly understand what is contained in the configuration file.  

Although this step is simple, it closely reflects the actual approach to troubleshooting configurations.

---

## Twelve, The Core Process of Day7

The core process of Day7 can be summarized as follows:

**First, traverse through all the logs; then classify and collect them based on different criteria; next, statistically analyze each category; finally, display the complete results.**

In code terms, this would look like:

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

## Thirteen, Day7's Output Should Be More Complete Than Day6's

Day6 usually produces output like this:

- List of error logs
- Number of error logs
- Whether any error logs were found

Day7, on the other hand, would produce output like this:

- List of INFO logs
- List of WARNING logs
- List of ERROR logs
- The number of each type of log
- Which type requires special attention

This indicates that the script is now more akin to "inspection result presentation" rather than just a simple judgment system.

---

## Fourteen, The Most Basic Complete Examples

### Example 1: Log Level Classification

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

    print(f"Number of INFO Logs: {len(info_log_list)}")
    print(f"Number of WARNING Logs: {len(warning_log_list)}")
    print(f"Number of ERROR Logs: {len(error_log_list)}")

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
        "server_name example.com;";
        "root /usr/share/nginx/html;</td>
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

    print(f"Listen Configuration List: {listen_config_list}")
    print(f"Server Name Configuration List: {server_name_config_list}")
    print(f"Other Configurations List: {other_config_list.")

---

## Fifteen, Day7 Class Exercises

Note:

**Class exercises are designed to help you master the basic classification concepts of the day. Homework assignments will not repeat these exercises but will focus more on operational scenarios, field segmentation, result aggregation, and comprehensive output.**

Below are the exercise questions without reference answers.  
Complete them independently first, and then check- 三个结果列表
- 三类配置行各自数量

---

### 5）port_check_result_classify.py

要求：把端口检查结果分成两类：

- 正常端口列表
- 异常端口列表

已知变量：

    port_status_list = ["80 open", "443 open", "3306 closed", "6379 open", "22 closed"]

建议思路：

- 包含 `"open"` 的放正常端口列表
- 包含 `"closed"` 的放异常端口列表

需要输出：

- 两个结果列表
- 两类结果各自数量
- 如果异常端口数量大于 0，输出“发现异常端口”
- 否则输出“没有发现异常端口”abnormal_port_list.append(port_status)

print(f"Normal ports: {normal_port_list}")
print(f"Number of normal ports: {len(normal_port_list)}")
print(f"Abnormal ports: {abnormal_port_list}")
print(f"Number of abnormal ports: {len(abnormal_port_list)}")

if len(abnormal_port_list) > 0:
    print("Abnormal ports were found")
else:
    print("No abnormal ports were found")```markdown
print(f"Number of failed services: {len(failed_service_list)}")
```

---

### 3）k8s_pod_status_classify_homework.py

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

    print(f"List of running pods: {running_pod_list}")
    print(f"Number of running pods: {len(running_pod_list)}")
    print(f"List of pending pods: {pending_pod_list}")
    print(f"Number of pending pods: {len(pending_pod_list)}")
    print(f"List of failed pods: {failed_pod_list}")
    print(f"Number of failed pods: {len(failed_pod_list)}")
```

---

### 4）inspection_summary_homework.py

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

    print(f"List of normal results: {normal_result_list}")
    print(f"Number of normal results: {len(normal_result_list)}")
    print(f"List of abnormal results: {abnormal_result_list}")
    print(f"Number of abnormal results: {len(abnormal_result_list)}")

    if len(abnormal_result_list) > 0:
        print("Abnormalities were found in this inspection.")
    else:
        print("The inspection result is normal.")
```

---

## Nineteen, Typical Problems That Arise During Day7 Exercises

### 1）Confusing the current element with the entire list inside a `for` loop

**Incorrect approach:**

    for log_line in log_list:
        if "INFO" in log_list:
            info_log_list.append(log_line)

**Correct approach:**

    for log_line in log_list:
        if "INFO" in log_line:
            info_log_list.append(log_line)

**Explanation:**

- `log_list` refers to the entire list.
- `log_line` represents the current element being processed in the loop.
- When making judgments inside a loop, it is essential to refer to the **current element**, not the entire list.

---

### 2）Failing to define the corresponding empty lists before adding new categories

**Incorrect approach:**

    else:
        other_status_code_list.append(code)

**Without defining the list first:**

    other_status_code_list = []

**Correct approach:**

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

**Explanation:**

* Before adding elements to a list, it is necessary to define that list first. This ensures that the program will not encounter errors due to missing variables.

---

### 3）Comparing strings and integers in the same comparison

**Incorrect approach:**

    if status_code == 200:
        print("Normal")

If the `status_code_list` contains strings like:

    status_code_list = ["200", "404", "500"]

**Correct approach:**

    if status_code == "200":
        print("Normal")

**Explanation:**

* It is essential to ensure that both sides of a comparison use the same data type. Strings should be compared with strings, and integers with integers.

---

### 4）Writing multiple `if` statements for separate classification conditions

**Incorrect approach:**

    if "ERROR" in log_line:
        error_log_list.append(log_line)

    if "WARNING" in log_line:
        warning_log_list.append(log_line)

**Recommended approach:**

    if "INFO" in log_line:
        info_log_list.append(log_line)
    elif "WARNING" in log_line:
        warning_log_list.append(log_line)
    else:
        error_log_list.append(log_line)

**Explanation:**

* In classification scenarios, conditions are usually mutually exclusive. Using `if / elif / else` makes the code more readable and maintainable.

---

### 5）Failing to count the number of items after- Simplify the previously written classification logic into a concise package

This way, introducing the function will be more natural and less abrupt.

---

## Chapter 22: External Links

### Python Official Documentation

- Python Tutorial: Control Flow Tools  
  https://docs.python.org/3/tutorial/controlflow.html

- Python Tutorial: Data Structures  
  https://docs.python.org/3/tutorial/datastructures.html

### Beginner Resources

- Runoob Tutorial: Python3 Conditional Statements  
  https://www.runoob.com/python3/python3-if-statement.html

- Runoob Tutorial: Python3 Loop Statements  
  https://www.runoob.com/python3/python3-loop.html

- Runoob Tutorial: Python3 Lists  
  https://www.runoob.com/python3/python3-list.html

- Runoob Tutorial: Python3 String split() Method  
  https://www.runoob.com/python/att-string-split.html

### Advanced Operations Understanding

- Nginx Configuration Basics  
  https://nginx.org/en/docs/

- Kubernetes Documentation  
  https://kubernetes.io/docs/home/

---

## Chapter 23: Today's Summary

**The essence of Day7 is to elevate Python from merely 'collecting a set of results and determining their presence' to 'classifying the same data based on different criteria, performing statistical analysis, and producing more comprehensive inspection results'. It also marks the beginning of understanding the fundamental approach of 'first splitting fields and then making judgments based on those fields.'**