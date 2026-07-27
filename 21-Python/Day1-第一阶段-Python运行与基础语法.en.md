# Day1-Phase 1-Python Execution and Basic Syntax

tags: #Python #Ops Development #Python Introduction #Linux Ops #Log Processing #K8s #Day1

## Learning Objectives

The goal of Day1 is not to understand all the theories, but to complete the following minimum loop:

1. Confirm and use Python3 in Ubuntu
2. Be able to create and execute `.py` files
3. Understand the basic usage of `print`, variables, strings, and integers
4. Be able to read and modify simple conditionals
5. Complete a basic “log keyword check script”

---

## I. Environment Preparation

Execute in the Ubuntu terminal:

    python3 --version

The expected output is similar to:

    Python 3.10.12

If it is not installed, execute:

    sudo apt update
    sudo apt install -y python3

Confirm again:

    python3 --version

### Additional Knowledge Points

- `python3` is the command for the Python 3 interpreter
- The role of an interpreter: to read and execute code in `.py` files
- For now, just remember: **Write Python files and then execute them with `python3 filename.py`**

---

## II. Creating the First Python File

Create a practice directory:

    mkdir -p ~/python-study/day1
    cd ~/python-study/day1

Create a file:

    vim hello.py

Write the following content:

    print("hello python")
    print("this is day1")

Execute:

    python3 hello.py

### Expected Result

    hello python
    this is day1

### Additional Knowledge Points

#### 1. `print()`

`print()` is used to output content to the terminal.

Examples:

    print("nginx")
    print("kubernetes")

#### 2. Strings

Content within quotes is usually a string, for example:

    "hello python"
    "nginx"
    "/var/log/messages"

A string can be understood as: **text-based data**

---

## III. Understanding How Python Files Are Executed

Create a file:

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

### Additional Knowledge Points

Python executes code line by line from top to bottom by default.

This is similar to shell scripts.  
For operations and maintenance, Python can be seen as a more powerful scripting language:

- It can handle text
- Process logs
- Manipulate files
- Deal with command outputs
- And later on, it can also be used to call APIs, automate tasks, or develop platform backends

---

## IV. Using Variables

Create a file:

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
    /var/log/nginx.access.log
    80

### Additional Knowledge Points

#### 1. What are Variables

A variable can be thought of as: **giving a name to a value so it can be reused later**

For example:

    service_name = "nginx"

This means saving the value `"nginx"` under the name `service_name`.

#### 2. Common Contents for Variables in Operations and Maintenance

Variables often store the following types of information:

- Hostnames
- IP addresses
- Pod names
- Namespaces
- File paths
- Port numbers
- Log contents
- Alert messages

#### 3. The Meaning of `=`

Here, it does not mean “equal” in mathematics, but “assignment”.

For example:

    port = 80

This means assigning the value `80` to the variable `port`.

---

## V. Identifying Basic Data Types

Create a file:

    vim type_demo.py

Write:

    service_name = "nginx"
    port = 80
    is_running = True

    print(type(service_name))
    print(type(port))
    print(type(is_RUNNING))

Execute:

    python3 type_demo.py

### Expected Result

    <class 'str'>
    <class 'int'>
    <class 'bool'>

### Additional Knowledge Points

For Day1, you only need to recognize the 3 most common types:

#### 1. `str`

A string is text content, for example:

    "nginx"
    "ERROR"
    "/var/log/messages"

#### 2. `int`

An integer is a whole number, for example:

    80
    6443
    3

#### 3. `bool`

A boolean value has only two possible values:

    True
    False    log_line = "ERROR kubelet node not ready"

    if "ERROR" in log_line:
        print("found error log")

Run:

    python3 if_demo.py

### Additional Knowledge Points

#### 1. The Function of `if`

`if` means "if the condition is met, then execute the following code".

#### 2. Typical Thinking in Operations and Maintenance

- If the log contains "ERROR", process it.
- If a file is larger than 10G, archive it.
- If a Pod is not in the Running state, trigger an alarm.
- If the API status code is not 200, retry.

#### 3. Indentation Is Very Important

In Python, indentation is used to denote code blocks.

Correct example:

    if "ERROR" in log_line:
        print("found error log")

The 4 spaces before `print()` cannot be omitted.

---

## Ten: Adding an `else` Block

Create a file:

    vim if_else_demo.py

Write in it:

    log_line = "INFO nginx started"

    if "ERROR" in log_line:
        print("found error log")
    else:
        print("no error found")

Run:

    python3 if_else_demo.py

### Additional Knowledge Points

`else` means "otherwise".

Structure:

    if condition:
        Execute when the condition is met
    else:
        Execute when the condition is not met

---

## Eleven: Day1 Core Practice: Script for Checking Log Keywords

Create a file:

    vim log_check.py

Write in it:

    log_line = "ERROR nginx upstream timeout"

    print(f"log content: {log_line}")

    if "ERROR" in log_line:
        print("Error-level log found")
    else:
        print("No error-level log found")

    if "timeout" in log_line:
        print("Timeout keyword found")
    else:
        print("Timeout keyword not found")

Run:

    python3 log_check.py

### Expected Results

    log content: ERROR nginx upstream timeout
    Error-level log found
    Timeout keyword found

### Additional Knowledge Points

This script already has the basic structure of an operations and maintenance script:

1. Read an input.
2. Output the current object being checked.
3. Determine if it contains specified keywords.
4. Provide the result.

Although simple, it serves as a prototype for subsequent log analysis, status checks, and alarm logic.

---

## Twelve: Performing Another Variableization Modification

Modify `log_check.py`:

    service_name = "nginx"
    log_line = "ERROR nginx upstream timeout"

    print(f"service: {service_name}")
    print(f"log content: {log_line}")

    if "ERROR" in log_line:
        print(f"{service_name} has error logs")

    if "timeout" in log_line:
        print(f"{service_name} has timeout issues")

Run:

    python3 log_check.py

### Additional Knowledge Points

The significance of doing this is:

- The same logic can be applied to different services.
- Variables make the script more extensible.
- In the future, it will gradually lead to processing actual file content and command outputs.

---

## Thirteen: Day1 Common Errors and Their Causes

### 1. Chinese Brackets

Incorrect example:

    print（"hello"）

Correct example:

    print("hello")

Reason: Python code must use English symbols.

---

### 2. Mismatched Quotes

Incorrect example:

    log_line = "ERROR nginx

Correct example:

    log_line = "ERROR nginx"

---

### 3. Indentation Errors

Incorrect example:

    if "ERROR" in log_line:
    print("found error")

Correct example:

    if "ERROR" in log_line:
        print("found error")

---

### 4. Direct Concatenation of Strings and Integers

Incorrect example:

    port = 80
    print("port is " + port)

Correct example:

    port = 80
    print(f"port is {port}")

---

## Fourteen: Day1 Practical Execution Sequence Suggestions

Execute in the following order without skipping any steps:

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

## Fif### 2. host_ip_v1.0.py
Function: Outputs the host IP address by concatenating strings.

    host_ip = "192.168.10.20"

    print("The current host IP is: " + host_ip)

Explanation:
- The `+` operator can be used to concatenate two strings.

---

### 3. host_ip_v2.0.py
Function: Outputs the host IP address using f-string.

    host_ip = "192.168.10.20"

    print(f"The current host IP is: {host_ip}")

Explanation:
- `f""` is a more recommended way to format output strings.
- It is suitable for displaying logs, status information, and inspection results.

---

### 4. raw_msg_clean.py
Function: Removes leading and trailing spaces from a string.

    raw_msg = "   pod status not ready   "
    clean_msg = raw_msg.strip()

    print(raw_msg)
    print(clean_msg)

Explanation:
- The `strip()` method is used to remove any空白 characters at the beginning or end of a string.
- This is commonly used when processing log contents, command outputs, and configuration files.

---

### 5. service_port.py
Function: Displays the service name and its associated port number.

    service_name = "kube-apiserver"
    port = 6443

    print(f"The service {service_name} listens on port {port}")

Explanation:
- It is possible to display both string and numeric variables together in this way.

---

### 6. warning_check_v1.0.py
Function: Checks if a log line contains the words "WARNING" or "ERROR".

    log_line = "WARNING kubelet cpu usage high"

    print(log_line)
    print("WARNING" in log_line)
    print("ERROR" in log_line)

Explanation:
- The expression `"content" in string` is used to check if a specified word is present within a string.
- The return value will be `True` or `False`.

---

### 7. warning_check_v2.0.py
Function: Uses `if` statements to check if a log line contains specific keywords and displays corresponding messages.

    log_line = "WARNING kubelet cpu usage high"

    print(log_line)
    if "WARNING" in log_line:
        print(f"{log_line} contains the word WARNING")
    if "ERROR" in log_line:
        print(f"{log_line} contains the word ERROR")

Explanation:
- `if` statements are used for conditional checks.
- The original script had an error with the variable name `log_line1`; it has been corrected to `log_line`.

---

### 8. Summary
The key concepts practiced in today's Day1 assignment include:

- Variable definition
- String output
- f-string formatting
- Using `strip()` to remove whitespace
- The `endswith()` method
- Checking for strings using `in`
- Conditional statements with `if / else`

These are fundamental skills essential for writing effective operations and maintenance scripts.

## Twenty, Next Phase

Content for Day2:

- Lists `list`
- Loops `for`
- Batch processing of multiple items

Learning these concepts will enable you to advance your script development from handling individual values to managing multiple data sets, bringing you closer to real-world operational scenarios.