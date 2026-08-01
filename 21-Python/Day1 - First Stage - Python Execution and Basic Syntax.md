# Day1 - First Stage - Python Execution and Basic Syntax

tags: #Python #TransportDevelopment #IntroductionToPython #LinuxTravel #LogProcessing #K8s #Day1

## Learning Objectives

Day1's goal is not to understand all theory, but to complete the following minimal loop:

1. Confirm and use Python3 in Ubuntu
2. Be able to create and execute `.py` file
3. Understand `print`, variables, strings, integers' basic usage
4. Be able to read and modify the simplest conditional judgment
5. Be able to complete a basic "log keyword check script"

---

## I. Environment Preparation

Execute in Ubuntu terminal:

    python3 --version

Expected output similar to:

    Python 3.10.12

If not installed, execute:

    sudo apt update
    sudo apt install -y python3

Confirm again:

    python3 --version

### Knowledge Supplement

- `python3` is Python 3 interpreter command
- Interpreter's role: read and execute code in `.py` file
- At this stage, just remember: **Write Python file, then use `python3 Filename.py` to execute**

---

## II. Create First Python File

Create practice directory:

    mkdir -p ~/python-study/day1
    cd ~/python-study/day1

Create file:

    vim hello.py

Write the following content:

    print("hello python")
    print("this is day1")

Execute:

    python3 hello.py

### Expected Result

    hello python
    this is day1

### Knowledge Supplement

#### 1. `print()`

`print()` is used to output content to terminal.

Example:

    print("nginx")
    print("kubernetes")

#### 2. Strings

Content in quotes is usually string, for example:

    "hello python"
    "nginx"
    "/var/log/messages"

String can be understood as: **text type data**

---

## III. Understanding Python File Execution

Create file:

    vim order_demo.py

Write:

    print("step1")
    print("step2")
    print("step3")

Execute:

    python3 order_demo.py

### Expected Result

    step1
    step2
    step3

### Knowledge Supplement

Python executes by default in order from top to bottom.

This is similar to shell scripts.  
For operations, we can first understand Python as a stronger scripting language:

- Can process text
- Can process logs
- Can process files
- Can process command outputs
- Later can call APIs, do automation, write platform backend

---

## IV. Variable Usage

Create file:

    vim variable_demo.py

Write:

    service_name = "nginx"
    log_path = "/var/log/nginx/access.log"
    port = 80

    print(service_name)
    print(log_path)
    print(port)

Execute:

    python3 variable_demo.py

### Expected Result

    nginx
    /var/log/nginx/access.log
    80

### Knowledge Supplement

#### 1. What is a variable

Variable can be understood as: **give a value a name for convenient reuse**

Example:

    service_name = "nginx"

Means save the value of `"nginx"` to the name of `service_name`.

#### 2. Typical content of variables from operations perspective

Variables often store these things later:

- Hostname
- IP address
- Pod name
- Namespace
- File path
- Port number
- Log content
- Alert content

#### 3. Meaning of `=`

This is not mathematical "equality", but "assignment".

Example:

    port = 80

Means assign the value of `80` to variable `port`

---

## V. Basic Data Type Identification

Create file:

    vim type_demo.py

Write:

    service_name = "nginx"
    port = 80
    is_running = True

    print(type(service_name))
    print(type(port))
    print(type(is_running))

Execute:

    python3 type_demo.py

### Expected Result

    <class 'str'>
    <class 'int'>
    <class 'bool'>

### Knowledge Supplement

Day1 only needs to recognize 3 most common types first:

#### 1. `str`

String, text content, for example:

    "nginx"
    "ERROR"
    "/var/log/messages"

#### 2. `int`

Integer, for example:

    80
    6443
    3

#### 3. `bool`

Boolean value, only two values:

    True
    False

Very common in operations:

- Service status
- File existence
- Condition satisfaction

---

## VI. Basic String Usage

Create file:

    vim string_demo.py

Write:

    service_name = "nginx"
    host_ip = "192.168.1.10"

    print("service name is " + service_name)
    print("host ip is " + host_ip)

Execute:

    python3 string_demo.py

### Knowledge Supplement

String concatenation can use `+`.

But later it's more recommended to use `f-string`.

---

## VII. f-string Output

Modify `string_demo.py` to:

    service_name = "nginx"
    host_ip = "192.168.1.10"
    port = 80

```python
print(f"service name is {service_name}")
print(f"host ip is {host_ip}")
print(f"port is {port}")

Run:

    python3 string_demo.py

### Knowledge Supplement

#### 1. What is f-string

Format:

    f"Text {variable}"

Purpose: Directly insert variable values into strings.

#### 2. Why it's recommended

Operational scripts frequently output:

- Host information
- Service name
- Port
- Log content
- Execution results

f-strings offer the best readability.

---

## Eight, Common String Operations

### 1. Get Length

Create file:

    vim str_len_demo.py

Write:

    log_line = "ERROR kube-apiserver connection refused"
    print(len(log_line))

Run:

    python3 str_len_demo.py

#### Knowledge Supplement

`len()` is a function used to get length.

Common uses:

- Determine if content is empty
- Check input length
- Check log content length

---

### 2. Remove leading and trailing whitespace

Create file:

    vim str_strip_demo.py

Write:

    raw_line = "   ERROR disk full   "
    clean_line = raw_line.strip()

    print(raw_line)
    print(clean_line)

Run:

    python3 str_strip_demo.py

#### Knowledge Supplement

`strip()` is a string method used to remove leading and trailing whitespace characters.

Common uses:

- Clean log lines
- Process file read content
- Clean command output

---

### 3. Check ending

Create file:

    vim str_end_demo.py

Write:

    filename = "access.log"
    print(filename.endswith(".log"))

Run:

    python3 str_end_demo.py

#### Knowledge Supplement

`endswith()` is a string method used to determine if a string ends with specific content.

Typical operational scenarios:

- Determine if it's a log file
- Determine if it's a configuration file
- Determine file extension

Examples:

    access.log
    nginx.conf
    values.yaml

---

### 4. Check starting

Create file:

    vim str_start_demo.py

Write:

    line = "ERROR nginx upstream timeout"
    print(line.startswith("ERROR"))

Run:

    python3 str_start_demo.py

#### Knowledge Supplement

`startswith()` is used to determine if a string starts with specified content (string matching defaults to exact match, case-sensitive).

---

### 5. Check if contains keyword

Create file:

    vim str_in_demo.py

Write:

    log_line = "ERROR nginx upstream timeout"

    print("ERROR" in log_line)
    print("timeout" in log_line)
    print("mysql" in log_line)

Run:

    python3 str_in_demo.py

#### Knowledge Supplement

`in` is used to determine if specific content appears in a string (string matching defaults to exact match, case-sensitive).

This is a highly frequent writing pattern in subsequent log analysis and alerting judgments.

---

## Nine, Conditional Judgment `if`

Create file:

    vim if_demo.py

Write:

    log_line = "ERROR kubelet node not ready"

    if "ERROR" in log_line:
        print("found error log")

Run:

    python3 if_demo.py

### Knowledge Supplement

#### 1. Purpose of `if`

`if` represents "if the condition is true, execute the code below".

#### 2. Typical operational thinking in operations

- If log contains `ERROR`, process it
- If file size exceeds 10G, archive it
- If Pod is not Running, alert
- If interface status code is not 200, retry

#### 3. Indentation is very important

Python uses indentation to represent code blocks.

Correct example:

    if "ERROR" in log_line:
        print("found error log")

The 4 spaces before `print()` cannot be omitted.

---

## Ten, Add `else`

Create file:

    vim if_else_demo.py

Write:

    log_line = "INFO nginx started"

    if "ERROR" in log_line:
        print("found error log")
    else:
        print("no error found")

Run:

    python3 if_else_demo.py

### Knowledge Supplement

`else` represents "otherwise".

Structure:

    if condition:
        execute when condition is true
    else:
        execute when condition is false

---

## Eleven, Day1 Core Practice: Log Keyword Check Script

Create file:

    vim log_check.py

Write:

    log_line = "ERROR nginx upstream timeout"

    print(f"log content: {log_line}")

    if "ERROR" in log_line:
        print("Found error level log")
    else:
        print("No error level log found")

    if "timeout" in log_line:
        print("Found timeout keyword")
    else:
        print("No timeout keyword found")

Run:

    python3 log_check.py

### Expected Result

log content: ERROR nginx upstream timeout
Found error-level log
Found timeout keyword

### Knowledge Supplement

This script already has the most basic structure of an operations script:

1. Read an input content
2. Output the current check object
3. Determine if it contains specified keywords
4. Provide results

Although simple, it's already the prototype for subsequent log analysis, status checks, and alert logic.

---

## Twelve, Re-do the variable transformation

Modify `log_check.py`:

    service_name = "nginx"
    log_line = "ERROR nginx upstream timeout"

    print(f"service: {service_name}")
    print(f"log content: {log_line}")

    if "ERROR" in log_line:
        print(f"{service_name} has error logs")

    if "timeout" in log_line:
        print(f"{service_name} has timeout issues")

Execution:

    python3 log_check.py

### Knowledge Supplement

The significance of this approach:

- The same logic can be adapted to different services
- Variables make scripts easier to extend
- It can gradually transition to "processing real file content" and "processing command output"

---

## Thirteen, Day1 Common Errors and Causes

### 1. Chinese parentheses

Error example:

    printI'm sorry."hello"I'm not sure.

Correct example:

    print("hello")

Cause: Python code must use English symbols.

---

### 2. Mismatched quotes

Error example:

    log_line = "ERROR nginx

Correct example:

    log_line = "ERROR nginx"

---

### 3. Indentation error

Error example:

    if "ERROR" in log_line:
    print("found error")

Correct example:

    if "ERROR" in log_line:
        print("found error")

---

### 4. Direct string and integer concatenation

Error example:

    port = 80
    print("port is " + port)

Correct example:

    port = 80
    print(f"port is {port}")

---

## Fourteen, Day1 Recommended Practice Order

Execute in the following order, do not skip:

    mkdir -p ~/python-study/day1
    cd ~/python-study/day1

    python3 --version

    vim hello.py
    python3 hello.py

    vim variable_demo.py
    python3 variable_demo.py

    vim type_demo.py
    python3 type_demo.py

    vim string_demo.py
    python3 string_demo.py

    vim str_strip_demo.py
    python3 str_strip_demo.py

    vim str_end_demo.py
    python3 str_end_demo.py

    vim str_in_demo.py
    python3 str_in_demo.py

    vim if_demo.py
    python3 if_demo.py

    vim if_else_demo.py
    python3 if_else_demo.py

    vim log_check.py
    python3 log_check.py

---

## Fifteen, Day1 Acceptance Criteria

After completing Day1, you should be able to:

- Independently create and execute Python files
- Understand basic variable assignment syntax
- Distinguish between strings and integers
- Use `print()` to output content
- Use f-string to output variables
- Use `strip()` to process text
- Use `startswith()` / `endswith()` to judge text features
- Use `in` to check for keyword existence
- Write the most basic `if` / `else` conditional judgments
- Write a simple log keyword detection script

---

## Sixteen, Day1 Summary

The core of Day1 is not "learning Python theory," but completing the following cognitive shift:

### 1. Python can be seen as an operations scripting tool

At this stage, you can understand Python as a scripting language stronger than shell.

### 2. The first step of operations development is to write judgment logic

For example:

- If logs contain ERROR, output an exception
- If a file ends with `.log`, identify it as a log file
- If output contains Running, judge the status as normal

### 3. Variables + strings + conditional judgments are the most basic foundation

Subsequent file processing, log analysis, command execution, and API calls will all be built on these foundations.

---

## Seventeen, External Reference Links

- Python official documentation: https://docs.python.org/3/
- Python string documentation: https://docs.python.org/3/library/stdtypes.html#text-sequence-type-str
- Python built-in functions documentation: https://docs.python.org/3/library/functions.html
- Runoob Python3 tutorial: https://www.runoob.com/python3/python3-tutorial.html

---

## Eighteen, Homework

### Homework 1

Write a script that defines the variable:

    host_ip = "192.168.10.20"

Output:

    Current host IP is: 192.168.10.20

---

### Homework 2

Write a script that defines the variable:

    filename = "messages.log"

Judge whether it is a `.log` file.

---

### Homework 3

Write a script that defines the variable:

    log_line = "WARNING kubelet cpu usage high"

Requirements:

- Output the original log
- Judge whether it contains `WARNING`
- Judge whether it contains `ERROR`

---

### Homework 4

Write a script that defines the variable:

    raw_msg = "   pod status not ready   "

Requirements:

- Remove leading and trailing spaces
- Output the processed content

---

### Homework 5

Write a script that defines the variable:

    service_name = "kube-apiserver"
    port = 6443

Output:

    Service kube-apiserver listens on port 6443

---
## 19: Day1 Assignment Script Records

### 1. filename_check.py
Purpose: Determine if the file is a `.log` log file.

    filename = "messages.log"

    if filename.endswith(".log"):
        print(f"{filename} is a log file")
    else:
        print(f"{filename} is not a log file")

Notes:
- `endswith(".log")` is used to check if a string ends with `.log`
- Suitable for log file filtering scenarios

---

### 2. host_ip_v1.0.py
Purpose: Output the host IP using string concatenation.

    host_ip = "192.168.10.20"

    print("Current host IP is:" + host_ip)

Notes:
- `+` can concatenate two strings

---

### 3. host_ip_v2.0.py
Purpose: Output the host IP using f-string.

    host_ip = "192.168.10.20"

    print(f"Current host IP is: {host_ip}")

Notes:
- `f""` is the recommended output method
- Suitable for subsequent log, status, and inspection information output

---

### 4. raw_msg_clean.py
Purpose: Remove leading and trailing whitespace from a string.

    raw_msg = "   pod status not ready   "
    clean_msg = raw_msg.strip()

    print(raw_msg)
    print(clean_msg)

Notes:
- `strip()` is used to clean leading/trailing whitespace characters
- Commonly used for processing log content, command outputs, and configuration read results

---

### 5. service_port.py
Purpose: Output service name and port.

    service_name = "kube-apiserver"
    port = 6443

    print(f"Service {service_name} listens on port {port}")

Notes:
- Can output both string and numeric variables simultaneously

---

### 6. warning_check_v1.0.py
Purpose: Determine if a log contains WARNING or ERROR.

    log_line = "WARNING kubelet cpu usage high"

    print(log_line)
    print("WARNING" in log_line)
    print("ERROR" in log_line)

Notes:
- `"Contents" in String` is used to check for specific content inclusion
- Return results are `True` or `False`

---

### 7. warning_check_v2.0.py
Purpose: Use `if` to check if a log contains keywords and output explanations.

    log_line = "WARNING kubelet cpu usage high"

    print(log_line)
    if "WARNING" in log_line:
        print(f"{log_line} contains WARNING")
    if "ERROR" in log_line:
        print(f"{log_line} contains ERROR")

Notes:
- `if` is used for conditional judgments
- The original script had a typo in `log_line1`, which has been corrected to `log_line`

---

### 8. Summary
Core content practiced in this Day1 assignment includes:

- Variable definition
- String output
- f-string
- `strip()`
- `endswith()`
- `in`
- `if / else`

These are the most commonly used foundational skills for writingTransport scripts in subsequent work.
## 20. Next Stage

Day2 Learning Content:

- List `list`
- Loop `for`
- Batch processing multiple objects

This will elevate scripts from "processing a single value" to "processing a batch of values," beginning to approach realTransport scenarios.