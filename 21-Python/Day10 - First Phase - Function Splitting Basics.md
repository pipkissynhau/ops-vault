# Day10 - First Stage - Function Decomposition Basics

#Python #PythonLearning #TransportDevelopment #Functions #return #FunctionSplit #MainProcess #ResultProcessing #ConfigureCheck #ServiceInspection #LogProcessing #Linux #Obsidian

## I. Today's Focus

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
- Multi-condition Combined Judgment
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
- "Script Thinking After Collecting Results" for Continued Judgment

Day7 Learning Content:

- Multi-condition Classification
- Multi-result List Collection
- Classification Statistics
- Result Summary Output
- `split()` Decomposing Fixed-format Text
- Starting to Establish "Inspection Result Summary" Thinking

Day8 Learning Content:

- Function `def`
- Function Call
- Parameters
- Encapsulating Repeated Logic into Functions
- Collecting Results in Loops, Unified Output Outside Loops

Day9 Learning Content:

- `return` Return Values
- Difference Between `print()` and `return`
- Function Returning Single Result
- Function Returning List Results
- Main Flow Receiving Function Return Values
- Main Flow Continuing Judgment, Statistics, and Output

Day10 Enters a New Stage:

**Start Connecting the Several Small Functions Learned on Day9 in Sequence.**

This step remains foundational and doesn't require thinking too complexly at once.  
Day10 first focuses on mastering one thing rather than pursuing "complete engineering script":

**Split "Filtering Results → Statistics → Conclusion" into 3 Functions, Then Have the Main Flow Call Them Sequentially.**

Thus, Day10's core is:

**On the Basis of Day9, Learn the Simplest Multi-function Chaining.**

---

## II. Today's Goals

After completing Day10, you should be able to:

1. Understand why a small script can be decomposed into multiple functions
2. Separate "Filtering, Statistics, Conclusion" into distinct parts
3. **Pass the return value of the previous function to the next function**
4. Write the simplest main flow
5. Understand how three functions are connected sequentially
6. Lay the foundation for Day11's advanced function decomposition

---

## III. What Will Be Learned Today

Day10's content can be summarized as:

1. Why a script should be decomposed into multiple functions
2. How three basic functions are connected sequentially
3. How the main flow calls functions in sequence
4. How to decompose a small script into "Filtering + Statistics + Conclusion"
5. Classroom Practice
6. Homework

---

## IV. Why Day10 Is Important

On Day9, you already know how to write such a function:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        return error_log_list

You also know how to write such a function:

    def get_result_count(result_list):
        return len(result_list)

And such a function:

    def get_check_result(result_count):
        if result_count > 0:
            return "Found anomalies"
        else:
            return "No anomalies found"

What Day10 does isn't complex:

**It's not learning new difficult syntax, but connecting these three functions sequentially.**

That is:

- Filter first
- Then count
- Then generate conclusion

This is Day10's focus.

---

## V. Differences Between Day10 and Day9

### Day9 Focuses More On:

- Single function
- A function returning a single result
- Main flow receiving and printing/judging the result

For example:

    def get_result_count(result_list):
        return len(result_list)

---

### Day10 Focuses More On:

- Multiple simple functions
- **A function's return value is used by the next function**
- Main flow connecting several functions sequentially

For example:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        return error_log_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Found error logs"
        else:
            return "No error logs found"

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

error_log_list = get_error_log_list(log_list)
error_count = get_result_count(error_log_list)
check_result = get_check_result(error_count)

print(f"Error log list:{error_log_list}")
print(f"Number of error logs:{error_count}")
print(check_result)

---

## Six. Day10: Understand the Simplest Structure First

Day10 don't think too complex, just remember this simplest structure first:

    def Function1(original_data):
        return result_list

    def Function2(result_list):
        return result_count

    def Function3(result_count):
        return check_result

    result_list = Function1(original_data)
    result_count = Function2(result_list)
    check_result = Function3(result_count)

    print(result_list)
    print(result_count)
    print(check_result)

Just understand this structure first, Day10 won't be messy.

---

## Seven. What is "Function Decomposition"?

Function decomposition can be simply understood as:

**Don't write all logic together, but divide it into different functions by steps.**

For example, a small script needs to do three things:

1. Find abnormal ports
2. Count abnormal ports
3. Give check conclusion

Then it can be decomposed into three functions:

### Function 1: Filter Abnormal Ports

    def get_abnormal_port_list(port_status_list):
        abnormal_port_list = []

        for port_status in port_status_list:
            if "closed" in port_status:
                abnormal_port_list.append(port_status)

        return abnormal_port_list

### Function 2: Count Results

    def get_result_count(result_list):
        return len(result_list)

### Function 3: Return Conclusion

    def get_check_result(result_count):
        if result_count > 0:
            return "Found abnormal ports"
        else:
            return "No abnormal ports found"

### Main Flow

    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

    abnormal_port_list = get_abnormal_port_list(port_status_list)
    abnormal_count = get_result_count(abnormal_port_list)
    check_result = get_check_result(abnormal_count)

    print(f"Abnormal Port List: {abnormal_port_list}")
    print(f"Abnormal Port Count: {abnormal_count}")
    print(check_result)

---

## Eight. Why Decompose into 3 Functions

### 1) Each Function Does One Thing

For example:

- `get_abnormal_port_list()` only responsible for filtering
- `get_result_count()` only responsible for counting
- `get_check_result()` only responsible for returning conclusion

This makes it clearer.

---

### 2) Easier to Reuse Later

For example `get_result_count()` can be reused in many places:

- Error log count
- Abnormal port count
- Faulty service count
- Backup file count

This is the benefit of decomposition.

---

### 3) Main Flow Easier to Understand

Main flow like steps:

- First, get result list
- Second, get result count
- Third, get check conclusion
- Fourth, unified output

This makes the logic smoother.

---

## Nine. Look at a "Non-Decomposed" Writing Style

For example:

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

    failed_service_list = []

    for service_status in service_status_list:
        if service_status == "failed":
            failed_service_list.append(service_status)

    failed_count = len(failed_service_list)

    if failed_count > 0:
        check_result = "Found faulty services"
    else:
        check_result = "No faulty services found"

    print(f"Faulty Service List: {failed_service_list}")
    print(f"Faulty Service Count: {failed_count}")
    print(check_result)

This script has no errors and can run.

But it combines:

- Filtering
- Counting
- Judgment

All in one place.

---

## Ten. Look at the "Decomposed" Writing Style

    def get_failed_service_list(service_status_list):
        failed_service_list = []

        for service_status in service_status_list:
            if service_status == "failed":
                failed_service_list.append(service_status)

        return failed_service_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Found faulty services"
        else:
            return "No faulty services found"

service_status_list = ["running", "stopped", "failed", "running", "failed"]

    failed_service_list = get_failed_service_list(service_status_list)
    failed_count = get_result_count(failed_service_list)
    check_result = get_check_result(failed_count)

    print(f"Faulty Service List: {failed_service_list}")
    print(f"Faulty Service Count: {failed_count}")
    print(check_result)

This version is more suitable for Day10.

---

## Eleven. The Core Understanding of Day10

Do not interpret Day10 as "complex multi-function collaboration".

A more suitable understanding is:

**Treat Day9's 3 small functions as sequential calls.**

Once you understand this sequence, Day10 is sufficient:

1. `get_xxx_list()`
2. `get_result_count()`
3. `get_check_result()`

Then in the main flow write:

1. `result_list = get_xxx_list(...)`
2. `result_count = get_result_count(result_list)`
3. `check_result = get_check_result(result_count)`

This is the fundamental main line of Day10.

---

## Twelve. What is the Main Flow

Starting from Day10, you need to gradually understand the concept of "main flow".

You can initially understand the main flow as:

**The part of code responsible for sequentially calling functions.**

For example:

    failed_service_list = get_failed_service_list(service_status_list)
    failed_count = get_result_count(failed_service_list)
    check_result = get_check_result(failed_count)

These three lines are the most critical part of the main flow.

You can understand it this way:

- Functions handle data
- Main flow arranges the call order

---

## Thirteen. What Does Main Flow Represent

The main flow can be understood as "the process arrangement of a small script".

For example when checking backup files:

    backup_file_list = get_backup_file_list(file_name_list)
    backup_count = get_result_count(backup_file_list)
    check_result = get_check_result(backup_count)

It's not doing complex calculations, but arranging:

- Find backup files first
- Then count them
- Then check for anomalies

So the main flow is like "a step-by-step guide".

---

## Fourteen. Core Value of Day10

The core value of Day10 is:

### 1) Start understanding a small script can be split into multiple functions

No longer just single functions, but organize by steps.

### 2) Start understanding functions can be chained

**The result of the previous function can be passed to the next function.**

### 3) Start understanding the role of main flow

Main flow doesn't handle all details, but schedules the order.

### 4) Lay the foundation for Day11's advanced splitting

First master the basic three-step structure, then move to clearer main flow organization.

---

## Fifteen. The Most Basic Complete Example

### Example 1: Error Log Check

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        return error_log_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Found error logs"
        else:
            return "No error logs found"

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

    error_log_list = get_error_log_list(log_list)
    error_count = get_result_count(error_log_list)
    check_result = get_check_result(error_count)

    print(f"Error Log List: {error_log_list}")
    print(f"Error Log Count: {error_count}")
    print(check_result)

---

### Example 2: Backup File Check

    def get_backup_file_list(file_name_list):
        backup_file_list = []

        for file_name in file_name_list:
            if file_name.endswith(".bak"):
                backup_file_list.append(file_name)

        return backup_file_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Found backup files"
        else:
            return "No backup files found"

    file_name_list = ["nginx.conf", "mysql.bak", "redis.conf", "etcd.bak"]

backup_file_list = get_backup_file_list(file_name_list)
backup_count = get_result_count(backup_file_list)
check_result = get_check_result(backup_count)

---

### Example 3: Kubernetes Service Check After Normalization

    def normalize_service_name_list(service_name_list):
        new_service_name_list = []

        for service_name in service_name_list:
            new_service_name_list.append(service_name.lower())

        return new_service_name_list

    def get_kube_service_list(service_name_list):
        kube_service_list = []

        for service_name in service_name_list:
            if service_name.startswith("kube"):
                kube_service_list.append(service_name)

        return kube_service_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Found Kubernetes-related services"
        else:
            return "No Kubernetes-related services found"

    service_name_list = ["NGINX", "KUBELET", "MySQL", "KUBE-PROXY"]

    new_service_name_list = normalize_service_name_list(service_name_list)
    kube_service_list = get_kube_service_list(new_service_name_list)
    kube_service_count = get_result_count(kube_service_list)
    check_result = get_check_result(kube_service_count)

    print(f"Kubernetes-related services list: {kube_service_list}")
    print(f"Kubernetes-related services count: {kube_service_count}")
    print(check_result)

This example adds one more step compared to the previous ones, but the essence remains:

- Process first
- Filter next
- Count then
- Draw conclusion

Day10: Just understand this order first.

---

## Sixteen. Classroom Exercise

Notes:

**The classroom exercise focuses on mastering the basic three-function chain first.**

---

### 1) error_log_function_chain.py

Requirements:

- Define function `get_error_log_list(log_list)`, returning error log list
- Define function `get_result_count(result_list)`, returning result count
- Define function `get_check_result(result_count)`, returning "Found error logs" if count > 0, else "No error logs found"
- Call these three functions sequentially in main flow
- Output:
  - Error log list
  - Error log count
  - Check conclusion

Known variable:

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

File name:

    error_log_function_chain.py

---

### 2) port_function_chain.py

Requirements:

- Define function `get_abnormal_port_list(port_status_list)`, returning abnormal port list
- Define function `get_result_count(result_list)`, returning result count
- Define function `get_check_result(result_count)`, returning "Found abnormal ports" if count > 0, else "No abnormal ports found"
- Call these three functions sequentially in main flow
- Output:
  - Abnormal port list
  - Abnormal port count
  - Check conclusion

Known variable:

    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

File name:

    port_function_chain.py

---

### 3) service_function_chain.py

Requirements:

- Define function `get_failed_service_list(service_status_list)`, returning failed service list
- Define function `get_result_count(result_list)`, returning result count
- Define function `get_check_result(result_count)`, returning "Found failed services" if count > 0, else "No failed services found"
- Call these three functions sequentially in main flow
- Output:
  - Failed service list
  - Failed service count
  - Check conclusion

Known variable:

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

File name:

    service_function_chain.py

---

### 4) backup_file_function_chain.py

Requirements:

- Define function `get_backup_file_list(file_name_list)`, returning file list ending with `.bak`
- Define function `get_result_count(result_list)`, returning result count
- Define function `get_check_result(result_count)`, returning "Found backup files" if count > 0, else "No backup files found"
- Call these three functions sequentially in main flow
- Output:
  - Backup file list
  - Backup file count
  - Check conclusion

Known variable:

file_name_list = ["nginx.conf", "mysql.bak", "redis.conf", "etcd.bak"]

File name:

    backup_file_function_chain.py

---

### 5) config_clean_function_chain.py

Requirements:

- Define function `clean_config_list(config_list)`, use `strip()` to clean whitespace around configuration items and return new list
- Define function `get_result_count(result_list)`, return result count
- Define function `get_check_result(result_count)`, return "Configuration cleaning completed" if count > 0, otherwise return "No configuration to clean"
- Call these three functions sequentially in main workflow
- Output:
  - Cleaned configuration list
  - Configuration count
  - Inspection conclusion

Known variables:

    config_list = [
        " listen 80 ",
        " server_name nginx.local ",
        " root /data/www "
    ]

File name:

    config_clean_function_chain.py

---

## SeventeenI don't know.Post-class Assignment

Notes:

**Post-class assignments continue to focus on solid foundation, with emphasis on three-function chaining, and at most adding one lightweight processing action.**

---

### 1) warning_log_function_chain_homework.py

Requirements:

- Define function `get_warning_log_list(log_list)`, find logs containing `"WARNING"` and return
- Define function `get_result_count(result_list)`, return result count
- Define function `get_check_result(result_count)`, return "Warning logs found" if count > 0, otherwise return "No warning logs found"
- Call these three functions sequentially in main workflow
- Output:
  - Warning log list
  - Warning log count
  - Inspection conclusion

Known variables:

    log_list = [
        "INFO nginx started",
        "WARNING disk usage high",
        "ERROR mysql down",
        "WARNING memory usage high"
    ]

File name:

    warning_log_function_chain_homework.py

---

### 2) gz_file_function_chain_homework.py

Requirements:

- Define function `get_gz_file_list(file_name_list)`, find all filenames ending with `.gz` and return
- Define function `get_result_count(result_list)`, return result count
- Define function `get_check_result(result_count)`, return "Compressed files found" if count > 0, otherwise return "No compressed files found"
- Call these three functions sequentially in main workflow
- Output:
  - Compressed file list
  - Compressed file count
  - Inspection conclusion

Known variables:

    file_name_list = ["syslog", "backup.tar.gz", "nginx.conf", "redis.rdb.gz"]

File name:

    gz_file_function_chain_homework.py

---

### 3) kube_service_function_chain_homework.py

Requirements:

- Define function `normalize_service_name_list(service_name_list)`, convert service names to lowercase and return
- Define function `get_kube_service_list(service_name_list)`, find services starting with `"kube"` from normalized list and return
- Define function `get_result_count(result_list)`, return result count
- Define function `get_check_result(result_count)`, return "Kubernetes-related services found" if count > 0, otherwise return "No Kubernetes-related services found"
- Call these four steps sequentially in main workflow
- Output:
  - Kubernetes-related service list
  - Kubernetes-related service count
  - Inspection conclusion

Known variables:

    service_name_list = ["NGINX", "KUBELET", "MySQL", "KUBE-PROXY"]

File name:

    kube_service_function_chain_homework.py

---

### 4) listen_replace_function_chain_homework.py

Requirements:

- Define function `replace_listen_port(config_list)`, replace `"listen 80"` with `"listen 8080"` and return new list
- Define function `get_result_count(result_list)`, return result count
- Define function `get_check_result(result_count)`, return "listen configuration processed" if count > 0, otherwise return "No configuration to process"
- Call these three functions sequentially in main workflow
- Output:
  - Modified configuration list
  - Configuration count
  - Inspection conclusion

Known variables:

    config_list = ["listen 80", "server_name nginx.local", "listen 443 ssl"]

File name:

    listen_replace_function_chain_homework.py

---

### 5) python_process_function_chain_homework.py

Requirements:

- Define function `get_python_process_list(process_name_list)`, find process names containing `"python"` and return
- Define function `get_result_count(result_list)`, return result count
- Define function `get_check_result(result_count)`, return "Python-related processes found" if count > 0, otherwise return "No Python-related processes found"
- Call these three functions sequentially in main workflow
- Output:
  - Python-related process list
  - Process count
  - Inspection conclusion

Known variables:

    process_name_list = ["python3 app.py", "nginx: master process", "redis-server", "python worker.py"]

File name:

    python_process_function_chain_homework.py

---

## EighteenI don't know.Day10 Common Errors Preview

### 1) The return value of the previous function was not passed to the next function

Example:

    error_log_list = get_error_log_list(log_list)
    error_count = get_result_count(log_list)

Here should be counting `error_log_list`, not the original `log_list`.

Correct way: /think

error_log_list = get_error_log_list(log_list)
error_count = get_result_count(error_log_list)

---

### 2I'm not sure.`get_check_result()` Receiving the wrong object

For example:

    check_result = get_check_result(error_log_list)

If this function is designed to receive a count, then pass the count instead:

    check_result = get_check_result(error_count)

---

### 3I'm not sure.Although three functions are written, the main workflow is not connected

For example, although functions are defined, there is no actual call below.  
Day10 is not about writing function definitions alone, but about using them effectively.

---

### 4I'm not sure.Variables are all named `result`, making it unclear later

For example:

    result = get_error_log_list(log_list)
    result = get_result_count(result)
    result = get_check_result(result)

Although it runs, the readability is poor.

A clearer approach:

    error_log_list = get_error_log_list(log_list)
    error_count = get_result_count(error_log_list)
    check_result = get_check_result(error_count)

---

### 5I'm not sure.Function responsibilities have merged again

For example:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        print(error_log_list)
        print(len(error_log_list))

        if len(error_log_list) > 0:
            print("Found error logs")
        else:
            print("No error logs found")

This function combines:

- Filtering
- Counting
- Judging
- Outputting

all in one place.

Day10 recommends splitting them.

---

### 6I'm not sure.After processing `replace()` or `strip()`, new results are not passed forward

For example:

    clean_config_list(config_list)
    result_count = get_result_count(config_list)

This does not use the processed new list.

---

## NineteenI don't know.Day10 Structured Problem-Solving Template

### Template One: Basic Three-Part Structure

    def get_xxx_list(source_list):
        result_list = []

        for item in source_list:
            if condition is met:
                result_list.append(item)

        return result_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Abnormal found"
        else:
            return "No abnormalities found"

    source_list = [...]

    result_list = get_xxx_list(source_list)
    result_count = get_result_count(result_list)
    check_result = get_check_result(result_count)

    print(f"Result list: {result_list}")
    print(f"Result count: {result_count}")
    print(check_result)

---

### Template Two: Process First, Then Count, Then Conclusion

    def handle_xxx_list(source_list):
        new_list = []

        for item in source_list:
            new_result = some processing
            new_list.append(new_result)

        return new_list

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Processing complete"
        else:
            return "No results to process"

    source_list = [...]

    new_list = handle_xxx_list(source_list)
    result_count = get_result_count(new_list)
    check_result = get_check_result(result_count)

    print(f"Processed result list: {new_list}")
    print(f"Result count: {result_count}")
    print(check_result)

---

## TwentyI don't know.Skills Gained After Completing Day10

After completing Day10, you will gain these abilities:

### 1I'm not sure.Ability to split a small script into multiple base functions

No longer just writing single functions, but designing them step-by-step.

### 2I'm not sure.Ability to connect functions via return values

The output of one function can be passed to the next.

### 3I'm not sure.Ability to write the most basic main workflow

The main workflow has a clear sequence, rather than a block of code.

### 4I'm not sure.Understanding of "functions handle processing, main workflow organizes logic"

This is crucial for future learning about files, commands, and error handling.

---

## Twenty-OneI don't know.Next Day Plan

Day11 will enter: Advanced Function Splitting and Main Workflow Organization.

Tomorrow's focus includes:

- Building on Day10's "Three Function Chain," continue to strengthen understanding of function call order  
- Learn to further combine "processing function + filtering function + statistical function + conclusion function"  
- Continue practicing how the main flow sequentially receives and passes return values  
- Begin to clearly distinguish between "function handling" and "main flow organization"  
- Naturally incorporate lightweight string processing in appropriate exercises, such as `strip()`, `replace()`, or `lower()`  
- Prepare for Day12 file reading basics, enabling gradual transition from "hardcoded lists in code" to "reading data from files and processing"  

---

## Twenty-TwoI don't know.External Links

### Python Official Documentation

- Python Tutorial: Function Definition  
  https://docs.python.org/3/tutorial/controlflow.html#defining-functions

- Python Tutorial: More About Functions  
  https://docs.python.org/3/tutorial/controlflow.html#more-on-defining-functions

### Beginner References

- CSDN Tutorial: Python3 Functions  
  https://www.runoob.com/python3/python3-functions.html

- CSDN Tutorial: Python3 Return Values  
  https://www.runoob.com/python3/python3-function.html

### Python Common String Methods

- Python Official Documentation: Text Sequence Type `str`  
  https://docs.python.org/3/library/stdtypes.html#text-sequence-type-str

### Operations Extension Understanding

- systemctl Official Manual  
    [https://www.freedesktop.org/software/systemd/man/systemctl.html](§§url_0§§)

- Linux man-pages Project  
    [https://www.kernel.org/doc/man-pages/](§§url_1§§)
 
- Python Official Documentation: Defining Functions  
    [https://docs.python.org/3/tutorial/controlflow.html#defining-functions](§§url_2§§)


---

## Twenty-ThreeI don't know.Today's One-Sentence Summary

**Day10's essence is connecting several small functions already written on Day9 in sequence, first mastering the basic structure of "filtering → statistics → conclusion" function decomposition.**