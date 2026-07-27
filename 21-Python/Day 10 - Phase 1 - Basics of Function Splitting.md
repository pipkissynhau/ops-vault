# Day10 - Phase 1 - Basics of Function Splitting

#Python #Python Learning #Ops Development #Functions #return #Function Splitting #Main Process #Result Processing #Configuration Checking #Service Inspection #Log Handling #Linux #Obsidian

## I. Today's Focus

Learning content for Day1:

- Variables
- Strings
- `print()`
- `f-string`
- `in`
- `strip()`
- `startswith()`
- `endswith()`

Learning content for Day2:

- `if / elif / else`
- Multi-branch judgments
- `upper()` / `lower()`
- `replace()`
- `split()`

Learning content for Day3:

- `and`
- `or`
- `not`
- Multiple conditional combinations
- `in`
- `not in`
- `startswith()`
- `endswith()`

Learning content for Day4:

- Lists `list`
- Indeksing
- `for` loops
- Traversing lists
- `for + if` for basic batch processing

Learning content for Day5:

- Result collection
- `append()`
- Basic counting
- Batch filtering
- Collecting results and generating statistics

Learning content for Day6:

- `len()` for calculating length
- Checking the number of result lists
- Checking empty lists
- Script thinking of "continuing processing after collecting results"

Learning content for Day7:

- Multi-condition classification
- Collecting multiple result lists
- Categorized statistics
- Summarizing and outputting results
- Using `split()` to split text in fixed formats
- Starting to develop the mindset of "inspection result summarization"

Learning content for Day8:

- The `def` function statement
- Function invocation
- Parameters
- Encapsulating repetitive logic into functions
- Collecting results inside loops and outputting them outside

Learning content for Day9:

- The `return` keyword
- The difference between `print()` and `return`
- Functions returning a single value
- Functions returning a list of values
- The main process receiving function return values
- The main process continuing with further processing, statistics, and output

The new phase starting in Day10 is:

**Starting to connect the several small functions learned in Day9 in sequence.**

This step remains fundamental; there's no need to overcomplicate it at first.  
On Day10, don't aim for a "complete engineered script" yet—focus on mastering one thing:

**Separating "result filtering → quantity statistics → conclusion generation" into three functions and calling them in order within the main process.**

So, the core of Day10 is:

**Building upon what was learned in Day9, learning how to connect multiple simple functions in sequence.**

---

## II. Today's Goals

After completing Day10, you should be able to:

1. Understand why a small script can be split into multiple functions.
2. Write separate functions for "filtering, counting, and generating conclusions."
3. **Pass the return value of one function to the next.**
4. Write a basic main process.
5. Comprehend how these three functions work together in sequence.
6. Lay the foundation for more advanced function splitting in Day11.

---

## III. What to Learn Today

The content of Day10 can be summarized as:

1. Why a script should be split into multiple functions.
2. How to connect three basic functions in sequence.
3. How the main process calls these functions in order.
4. How to break down a small script into "filtering + counting + conclusion generation."
5. Classroom exercises.
6. Homework.

---

## IV. Why Day10 Is Important

By now, you already know how to write functions like these:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        return error_log_list

And also functions like these:

    def get_result_count(result_list):
        return len(result_list)

    def get_check_result(result_count):
        if result_count > 0:
            return "Abnormalities detected"
        else:
            return "No abnormalities detected"

What needs to be done in Day10 is not complicated:

**It's not about learning new, difficult syntax—it's about connecting these three functions in sequence.**

That is:

- First, filter the data.
- Then, count the results.
- Finally, generate a conclusion.

This is the focus of Day10.

---

## V. The Difference Between Day10 and Day9

### Day9 Focuses More On:

- Single functions
- One function returning one result
- The main process receiving the result and either printing it or making a judgment based on it.

For example:

    def get_result_count(result_list):
        return len(result_list)

For example, the function `get_result_count()` can be reused in many scenarios:

- To count the number of error logs.
- To determine the number of ports with exceptions.
- To identify the number of failed services.
- To count the number of backup files.

This is precisely the advantage of breaking down code into smaller functions.

---

### 3) The main process becomes easier to understand

The main process can be structured in steps like this:

- Step one: Obtain the list of results.
- Step two: Calculate the total number of results.
- Step three: Determine the inspection conclusion.
- Step four: Output the final result.

This approach makes the logic much clearer.

---

## 9. Let’s first look at an example where functions are not split

For instance:

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

    failed_service_list = []

    for service_status in service_status_list:
        if service_status == "failed":
            failed_service_list.append(service_status)

    failed_count = len(failed_service_list)

    if failed_count > 0:
        check_result = "Failed services were detected"
    else:
        check_result = "No failed services were found"

    print(f"List of failed services: {failed_service_list}")
    print(f"Number of failed services: {failed_count}")
    print(check_result)

This script is correct and will run, but it combines the following tasks:

- Filtering.
- Counting.
- Making a judgment.

into a single block of code.

---

## 10. Now let’s look at the version with split functions

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
            return "Failed services were detected"
        else:
            return "No failed services were found"

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

    failed_service_list = get_failed_service_list(service_status_list)
    failed_count = get_result_count(failed_service_list)
    check_result = get_check_result(failed_count)

    print(f"List of failed services: {failed_service_list}")
    print(f"Number of failed services: {failed_count}")
    print(check_result)

This version is more suitable for Day10.

---

## 11. The key concept to understand about Day10

Don’t initially think of Day10 as something overly complex involving multiple functions working together.

A better way to understand it is:

**Simply call the three small functions from Day9 in the correct order.**

As long as you grasp this sequence, Day10 will be clear:

1. `get_xxx_list()`
2. `get_result_count()`
3. `get_check_result()`

Then, in the main process, you can write:

1. `result_list = get_xxx_list(...)`
2. `result_count = get_result_count(result_list)`
3. `check_result = get_check_result(result_count)`

This constitutes the basic structure of Day10.

---

## 12. What is the main process?

At the beginning of Day10, you should gradually understand the concept of the “main process”.

The main process can be simply thought of as:

**The part of the code that is responsible for calling functions in sequence.**

For example:

    failed_service_list = get_failed_service_list(service_status_list)
    failed_count = get_result_count(failed_service_list)
    check_result = get_check_result(failed_count)

These three lines represent the most critical part of the main process.

You can think of it this way:

- Functions handle the data.
- The main process manages the order in which these functions are called.

---

## 13. What does the main process resemble?

The main process can be likened to “the sequence of steps in a small script”.

For instance, when checking backup files:

    backup_file_list = get_backup_file_list(file_name_list)
    backup_count = get_result_count(backup_file_list)
    check_result = get_check_result(backup_count)

It’s not performing complex calculations; instead, it is organizing the following tasks:

- First, identify the backup files.
- Second, count their number.
- Third, determine if there are any issues.

Therefore, the main process is like a “step-by-step guide”.

---

## 14. The core value of Day10

The core value of Day10 lies in:

### 1) Realizing that a small script can be divided into multiple functions

You will no longer rely on single-function scripts but start organizing code in a step-by-step manner.

### 2) Understanding that functions can be    check_result = get_check_result(backup_count)

    print(f"Backup file list: {backup_file_list}")
    print(f"Number of backup files: {backup_count}")
    print(check_result)

---

### Example 3: Checking After Normalizing Service Names

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
            return "Kubernetes-related services found"
        else:
            return "No Kubernetes-related services found"

    service_name_list = ["NGINX", "KUBELET", "MySQL", "KUBE-PROXY"]

    new_service_name_list = normalize_service_name_list(service_name_list)
    kube_service_list = get_kube_service_list(new_service_name_list)
    kube_service_count = get_result_count(kube_service_list)
    check_result = get_check_result(kube_service_count)

    print(f"Kubernetes-related service list: {kube_service_list}")
    print(f"Number of Kubernetes-related services: {kube_service_count}")
    print(check_result)

This example includes an additional step, but the overall process remains the same:

- Process the data first
- Filter the results
- Count the number of items
- Draw a conclusion

For Day10, it's important to understand this sequence.

---

## Exercise 16: Class Practice

Instructions:

**The focus of this class exercise is to master the basic sequence of chaining three functions.**

---

### 1) error_log_function_chain.py

Requirements:

- Define the function `get_error_log_list(log_list)` to return a list of error logs.
- Define the function `get_result_count(result_list)` to return the number of items in the result list.
- Define the function `get_check_result(result_count)` to return "Error logs found" if the count is greater than 0, otherwise "No error logs found".
- Call these three functions in sequence in the main process.
- Output:
  - The list of error logs
  - The number of error logs
  - The check conclusion

Given variables:

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

- Define the function `get_abnormal_port_list(port_status_list)` to return a list of abnormal ports.
- Define the function `get_result_count(result_list)` to return the number of items in the result list.
- Define the function `get_check_result(result_count)` to return "Abnormal ports found" if the count is greater than 0, otherwise "No abnormal ports found".
- Call these three functions in sequence in the main process.
- Output:
  - The list of abnormal ports
  - The number of abnormal ports
  - The check conclusion

Given variables:

    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

File name:

    port_function_chain.py

---

### 3) service_function_chain.py

Requirements:

- Define the function `get_failed_service_list(service_status_list)` to return a list of failed services.
- Define the function `get_result_count(result_list)` to return the number of items in the result list.
- Define the function `get_check_result(result_count)` to return "Failed services found" if the count is greater than 0, otherwise "No failed services found".
- Call these three functions in sequence in the main process.
- Output:
  - The list of failed services
  - The number of failed services
  - The check conclusion

Given variables:

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

File name:

    service_function_chain.py

---

### 4) backup_file_function_chain.py

Requirements:

- Define the function `get_backup_file_list(file_name_list)` to return a list of files ending with `.bak`.
- Define the function `get_result_count(result_list)` to return the number of items in the result list.
- Define the function `get_check_result(result_count)` to return "Backup files found" if the count is greater than 0, otherwise "No backup files found".
- Call these three functions in sequence in the main process.
- Output:
  - The list of backup files
  - The number of backup files
  - The check conclusion

Given variables:

- Define the function `get_check_result(result_count)`. If the number is greater than 0, return "Compressed files found"; otherwise, return "No compressed files found".
- In the main process, call these three functions in sequence.
- Output:
  - List of compressed files
  - Number of compressed files
  - Inspection conclusion

Given variables:

    file_name_list = ["syslog", "backup.tar.gz", "nginx.conf", "redis.rdb.gz"]

File name:

    gz_file_function_chain_homework.py

---

### 3）kube_service_function_chain.homework.py

Requirements:

- Define the function `normalize_service_name_list(service_name_list)`. Convert all service names to lowercase and return the resulting list.
- Define the function `get_kube_service_list.service_name_list)`. From the normalized list, identify services that start with "kube" and return them.
- Define the function `get_result_count(result_list)`. Return the number of elements in the result list.
- Define the function `get_check_result(result_count)`. If the number is greater than 0, return "Kubernetes-related services found"; otherwise, return "No Kubernetes-related services found".
- In the main process, perform these four steps in sequence.
- Output:
  - List of Kubernetes-related services
  - Number of Kubernetes-related services
  - Inspection conclusion

Given variables:

    service_name_list = ["NGINX", "KUBELET", "MySQL", "KUBE-PROXY"]

File name:

    kube_service_function_chain_homework.py

---

### 4）listen_replace_function_chain.homework.py

Requirements:

- Define the function `replace.listen_port(config_list)`. Replace all occurrences of `"listen 80"` with `"listen 8080"` and return the new list.
- Define the function `get_result_count(result_list)`. Return the number of elements in the result list.
- Define the function `get_check_result(result_count)`. If the number is greater than 0, return "Listen configuration processing completed"; otherwise, return "No configurations to process".
- In the main process, call these three functions in sequence.
- Output:
  - List of replaced configurations
  - Number of configurations
  - Inspection conclusion

Given variables:

    config_list = ["listen 80", "server_name nginx.local", "listen 443 ssl"]

File name:

    listen_replace_function_chain.homework.py

---

### 5）python_process_function_chain_homework.py

Requirements:

- Define the function `get_python_process_list(process_name_list)`. Identify process names that contain "python" and return them.
- Define the function `get_result_count(result_list)`. Return the number of elements in the result list.
- Define the function `get_check_result(result_count)`. If the number is greater than 0, return "Python-related processes found"; otherwise, return "No Python-related processes found".
- In the main process, call these three functions in sequence.
- Output:
  - List of Python-related processes
  - Number of processes
  - Inspection conclusion

Given variables:

    process_name_list = ["python3 app.py", "nginx: master process", "redis-server", "python worker.py"]

File name:

    python_process_function_chain.homework.py

---

## Eighteen, Day10 Common Error Precautions

### 1）The return value of the previous function is not passed to the next function

For example:

    error_log_list = get_error_log_list(log_list)
    error_count = get_result_count(log_list)

In this case, the `error_log_list` that should be counted is actually the original `log_list`, not the filtered one.

Correct way to write it:

    error_log_list = get_error_log_list(log_list)
    error_count = get_result_count(error_log_list)

---

### 2）`get_check_result()` receives the wrong object

For example:

    check_result = get_check_result(error_log_list)

If this function is designed to receive a count, then the correct parameter should be the count:

    check_result = get_check_result(error_count)

---

### 3）Although the three functions are defined, they are not called in the main process

For example, even though the functions are defined, they are not actually executed in the subsequent code. Day10 is not about just writing function definitions; it's about using them effectively.

---

### 4）Variable names are all set to `result`, making them difficult to distinguish later on

For example:

    result = get_error_log_list(log_list)
    result = get_result_count(result)
    result = get_check_result(result)

Although it will run, the code is less readable.

A clearer way to write it:

    error_log_list = get_error_log_list(log_list)
    error_count = get_result_count(error_log_list)
    check_result = get_check_result(error_count)

---

### 5）The responsibilities ofreturn "No results to process"

source_list = [...]

new_list = handle_xxx_list(source_list)
result_count = get_result_count(new_list)
check_result = get_check_result(result_count)

print(f"Processed result list: {new_list}")
print(f"Number of results: {result_count}")
print(check_result)

---

## Section 20: What Skills Will Be Acquired After Completing Day10

After completing Day10, the following skills will be added:

### 1) Ability to break down a small script into multiple basic functions

Instead of only knowing how to write individual functions, you will learn how to design them step by step.

### 2) Knowing how to connect functions through return values

The result of one function can be passed on to the next function.

### 3) Being able to write basic main procedures

Main procedures will have a clear sequence, rather than being just a bunch of code piled together.

### 4) Understanding that "functions handle tasks, while main procedures organize them"

This is very important for learning about files, commands, and error handling later on.

---

## Section 21: Plans for Tomorrow

On Day11, we will move on to advanced function splitting and organizing main procedures.

Key points for tomorrow include:

- Building upon the "three functions in sequence" from Day10, further improve understanding of the call order between functions
- Learn how to combine "processing functions + filtering functions + statistical functions + conclusion functions"
- Continue practicing how main procedures receive and pass return values in sequence
- Start clearly distinguishing between "functions handling tasks" and "main procedures organizing them"
- Naturally incorporate simple string processing methods, such as `strip()`, `replace()`, or `lower()` in appropriate exercises
- Prepare for basic file reading on Day12, so that we can gradually transition from using hardcoded lists to reading data from files for processing

---

## Section 22: External Links

### Python Official Documentation

- Python Tutorial: Function Definition  
  https://docs.python.org/3/tutorial/controlflow.html#defining-functions

- Python Tutorial: More About Functions  
  https://docs.python.org/3/tutorial/controlflow.html#more-on-defining-functions

### Beginner Resources

- Runoob Tutorial: Python3 Functions  
  https://www.runoob.com/python3/python3-functions.html

- Runoob Tutorial: Python3 Return Values  
  https://www.runoob.com/python3/python3-function.html

### Common Python String Methods

- Python Official Documentation: Text Sequence Type `str`  
  https://docs.python.org/3/library/stdtypes.html#text-sequence-type-str

### Additional Resources for Operations and Maintenance

- systemctl Official Manual  
    [https://www.freedesktop.org/software/systemd/man/systemctl.html](https://www.freedesktop.org/software/systemd/man/systemctl.html)

- Linux man-pages Project  
    [https://www.kernel.org/doc/man_pages/](https://www.kernel.org/doc/man_pages/)
 
- Python Official Documentation: Defining Functions  
    [https://docs.python.org/3/tutorial/controlflow.html#defining-functions](https://docs.python.org/3/tutorial/controlflow.html#defining-functions)


---

## Section 23: Today's Summary in One Sentence

**The essence of Day10 is to connect the few small functions you learned on Day9 in a sequential order, starting with mastering the basic structure of "filtering → summarizing → concluding" functions.**