# Day8 - Phase 1 - Function Basics and Result Inspection Encapsulation

#Python #Python Learning #Ops Development #Functions #def #Parameters #Function Call #Result Inspection #Inspection Script #Log Analysis #Status Check #Kubernetes #Linux #Obsidian

## I. Today's Focus

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
- Multiple conditions combined
- `in`
- `not in`
- `startswith()`
- `endswith()`

Learning content from Day4:

- Lists `list`
- Indeks
- `for` loops
- Traversing lists
- `for + if` for basic batch judgments

Learning content from Day5:

- Result collection
- `append()`
- Basic counting
- Batch filtering
- Collecting results and outputting statistics

Learning content from Day6:

- `len()` for counting length
- Checking the number of result lists
- Checking empty lists
- Script thinking of "continuing judgments after collecting results"

Learning content from Day7:

- Multi-condition classification
- Collecting multiple result lists
- Classification statistics
- Summarizing and outputting results
- `split()` for splitting fixed-format text
- Starting to develop the idea of "inspection result summarization"

The new phase starting in Day8 is:

**No longer just writing code line by line, but beginning to encapsulate "reusable inspection logic" into functions.**

This step is very important because in real ops scripts, you often encounter the following scenarios:

- Repeated use of log inspection logic
- Repeated use of port inspection logic
- Repeated use of service status inspection logic
- Repeated use of Pod status inspection logic
- Repeated use of status code classification logic

If you write the same logic every time, the script will become longer and harder to maintain.

Therefore, the core of Day8 is:

**Encapsulating reusable processing logic into functions to make scripts shorter, clearer, and easier to reuse.**

---

## II. Today's Goals

After completing Day8, you should be able to:

1. Understand what a function is.
2. Use `def` to define functions.
3. Call functions.
4. Understand the role of parameters.
5. Encapsulate simple inspection logic into functions.
6. Reuse the same logic multiple times.
7. Lay the foundation for returning values and more complete script structures in the future.

---

## III. What to Learn Today

The content of Day8 can be summarized as:

1. What a function is.
2. Why use functions.
3. Defining functions with `def`.
4. Calling functions.
5. Basics of parameters.
6. Encapsulating simple inspection logic into functions.
7. Optimizing previously written scripts using functions.
8. In-class exercises.
9. Homework.

---

## IV. Why Day8 Is Important

Most of the scripts written in the previous few days had this structure:

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high"
    ]

    error_log_list = []

    for log_line in log_list:
        if "ERROR" in log_line:
            error_log_list.append(log_line)

    print(error_log_list)
    print(len(error_log_list))

Such scripts can complete tasks, but if you need to continue checking later:

- Error logs
- Alert logs
- Abnormal ports
- Failed services
- Pending Pods

you will constantly have to rewrite similar logic.

For example:

    abnormal_result_list = []

    for item in result_list:
        if "failed" in item:
            abnormal_result_list.append(item)

Or:

    abnormal_port_list = []

    for port_status in port_status_list:
        if "closed" in port_status:
            abnormal_port_list.append(port_status)

Essentially, all this logic does the same thing:

**Traverse a set of data, identify target results based on conditions, and then output them.**

If you encapsulate this kind of logic into functions, you can achieve:

- Shorter code
- Easier repeated calls
- Easier maintenance
- Closer to real script development.

---

## V. What Is a Function

A function can be simply understood as:

**Packing a piece of reusable code together, giving it a name, and calling it whenever needed later.**

You can think of a function as a "tool".

For example:

- There is a tool specifically for checking error logs.
- There is a tool specifically for checking abnormal ports.
- There        print(f"Result count: {len(result_list)}")

Calls:

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    failed_service_list = ["mysql failed"]

    print_result_count(error_log_list)
    print_result_count(failed_service_list)

Output:

    Result count: 2
    Result count: 1

This example demonstrates that:

**The same function can handle different sets of data by simply passing in different parameters.**

---

## Fifteen, What Logic Can Be Encapsulated First on Day8

Based on the content from Day4 to Day7, the following logic is most suitable for encapsulation initially:

### 1) Outputting a separator line

    def print_separator():
        print("-----")

### 2) Outputting a check title

    def print_title(title):
        print(f"===== {title} =====")

### 3) Outputting the result count

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

### 4) Indicating whether abnormalities were found

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Abnormalities found")
        else:
            print("No abnormalities found")

These functions are not complex but serve as excellent starting points for learning how to create functions.

---

## Sixteen, The Most Basic Complete Examples

### Example 1: Encapsulating "Result Count Output"

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    warning_log_list = ["WARNING disk usage high"]

    print_result_count(error_log_list)
    print_result_count(warning_log_list)

### Example 2: Encapsulating "Whether Abnormalities Were Found"

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Abnormalities found")
        else:
            print("No abnormalities found")

    abnormal_port_list = ["3306 closed", "22 closed"]
    print_check_result(abnormal_port_list)

### Example 3: Encapsulating "Service Check Messages"

    def check_service(service_name):
        print(f"Checking service: {service_name}")

    check_service("nginx")
    check_service("mysql")
    check_service("redis")

### Example 4: Encapsulating "Inspection Titles"

    def print_inspection_title(title):
        print(f"===== {title} =====")

    print_inspection_title("Nginx Log Check")
    print_inspection_title("MySQL Service Check")

---

## Seventeen, The Core Value of Day8

The core value of Day8 lies in:

### 1) Starting to Reduce Repetitive Code

By putting repetitive logic into functions, there's no need to rewrite it every time.

### 2) Making the Script Structure Clearer

Main processes are defined through calls, while specific logic is handled within functions, making the code easier to read.

### 3) Developing a "Modular" Thinking Approach

Although still basic at this stage, it’s already moving towards the idea of breaking down different tasks into separate functional blocks.

### 4) Preparing for Future Return Values

Learning about "defining" and "calling" functions today makes it more natural to understand return values later on.

---

## Eighteen, Class Exercises

Note:

**Class exercises are designed to help you master the basics of function definition, calling, and parameter passing. Complex logic is not focused on at this stage.**

---

### 1) print_separator.py

Requirements:

- Define a function `print_separator()`
- The function should output a separator line:

    ====================

- Call this function three times.

File name:

    print_separator.py

### Answer Key

    def print_separator():
        print("====================")

    print_separator()
    print_separator()
    print_separator()

---

### 2) print_service_name.py

Requirements:

- Define a function `print_service_name(service_name)`
- The function should output:

    Current service name is: nginx

- Call the function three times, passing in:

    "nginx"
    "mysql"
    "redis"

File name:

    print_service_name.py

### Answer Key

    def print_service_name(service_name):
        print(f"Current service name is: {service_name}")

    print_service_name("nginx")
    print_service_name("mysql")
    print_service_name("redis")

---

### 3) print_result_count.py

Requirements:

- Define a function `print_result_count(result_list)`
- The function should output the number of elements in the result list.

Given variables:

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    warning_log_list = ["WARNING disk usage high"]

Expected output:

- The number of elements in `error_log_list`
- The number of elements in `warning_log_list`

File name:

   ### 1）log_error_check_function_homework.py

Requirements:

- Define a function `check_error_log(log_list)`
- The function should identify logs that contain the word "ERROR"
- Output:
  - A list of error logs
  - The number of error logs
  - If there are more than 0 error logs, print "Error logs found"
  - Otherwise, print "No error logs found"

Given variables:

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

File name:

    log_error_check_function_homework.py

### Reference Answer

    def check_error_log(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        print(f"Error logs list: {error_log_list}")
        print(f"Number of error logs: {len(error_log_list)}")

        if len(error_log_list) > 0:
            print("Error logs found")
        else:
            print("No error logs found")


    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

    check_error_log(log_list)

---

### 2）port_abnormal_check_function_homework.py

Requirements:

- Define a function `check_abnormal_port(port_status_list)`
- The function should identify port status entries that contain the word "closed"
- Output:
  - A list of abnormal ports
  - The number of abnormal ports
  - If there are more than 0 abnormal ports, print "Abnormal ports found"
  - Otherwise, print "No abnormal ports found"

Given variables:

    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

File name:

    port_abnormal_check_function_homework.py

### Reference Answer

    def check_abnormal_port(port_status_list):
        abnormal_port_list = []

        for port_status in port_status_list:
            if "closed" in port_status:
                abnormal_port_list.append(port_status)

        print(f"Abnormal ports list: {abnormal_port_list}")
        print(f"Number of abnormal ports: {len(abnormal_port_list)}")

        if len(abnormal_port_list) > 0:
            print("Abnormal ports found")
        else:
            print("No abnormal ports found")


    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

    check_abnormal_port(port_status_list)

---

### 3）service_status_count_function_homework.py

Requirements:

- Define a function `count_service_status(service_status_list)`
- The function should count:
  - Services that are "running"
  - Services that are "stopped"
  - Services that are "failed"

Given variables:

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

Output requirements:

- The number of services that are running
- The number of services that are stopped
- The number of services that have failed

File name:

    service_status_count_function_homework.py

### Reference Answer

    def count_service_status(service_status_list):
        running_service_list = []
        stopped_service_list = []
        failed_service_list = []

        for service_status in service_status_list:
            if service_status == "running":
                running_service_list.append.service_status)
            elif service_status == "stopped":
                stopped_service_list.append(service_status)
            else:
                failed_service_list.append(service_status)

        print(f"Running services list: {running_service_list}")
        print(f"Number of running services: {len(running_service_list)}")
        print(f"Stopped services list: {stopped_service_list}")
        print(f"Number of stopped services: {len(stopped_service_list)}")
        print(f"Failed services list: {failed_service_list}")
        print(f"Number of failed services: {len(failed_service_list)}")


    service_status_list = ["running", "stopped", "failed", "running", "failed"]

    count_service_status(service_status_list)

---

### 4）http_status_classify_function_homework.py

Requirements:

- Define a function `classify_status_code(status_code_list)`
- The function should classify status codes into:
  - A list of normal status codes
  - A list of client-side error status codes
  - A list of server-side error status codes

Given variables:

    status_code_list = ["200", "404", "500", "200", "502"]

Output requirements:

- Three lists containing the classified status codes
- The number of each type of status code

File name:

    http_status_classify_function_homework.py

### Reference Answer

    def classify        print(f"List of server error status codes: {server_error_status_code_list}")
        print(f"Number of server error status codes: {len(server_error_status_code_list)}")


    status_code_list = ["200", "404", "500", "200", "502"]

    classify_status_code(status_code_list)

---

### 5）http_log_status_check_function_homework.py

Requirements:

- Define a function `check_http_log_status(access_log_list)`
- Functionality:
  - First, use `split()` to split each line of access logs
  - Extract the status code
  - Classify requests by status code into:
    - A list of normal requests
    - A list of client-side error requests
    - A list of server-side error requests

Given variables:

    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200"
    ]

Expected output:

- A list of normal requests
- A list of client-side error requests
- A list of server-side error requests
- The number of each type of request

File name:

    http_log_status_check_function_homework.py

### Reference Answer

    def check_http_log_status(access_log_list):
        normal_request_list = []
        client_error_request_list = []
        server_error_request_list = []

        for access_log in access_log_list:
            parts = access_log.split()
            status_code = parts[3]

            if status_code.startswith("2"):
                normal_request_list.append(access_log)
            elif status_code.startswith("4"):
                client_error_request_list.append/access_log)
            elif status_code.startswith("5"):
                server_error_request_list.append.access_log)

        print(f"Normal request list: {normal_request_list}")
        print(f"Number of normal requests: {len(normal_request_list)}")
        print(f"Client-side error request list: {client_error_request_list}")
        print(f"Number of client-side error requests: {len(client_error_request_list)}")
        print(f"Server-side error request list: {server_error_request_list}")
        print(f"Number of server-side error requests: {len(server_error_request_list)}")


    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200"
    ]

    check_http_log_status(access_log_list)

---

## Twenty, Typical Errors in the Problem-Solving Process

The following errors are very common in Day8 exercises and need continuous attention.

### 1）Defining a function without calling it

For example:

    def print_separator():
        print("-----")

If you don't write:

    print(separator()

after that, the function will not be executed.

---

### 2）Forgetting to put parentheses after the function name

For example:

    printseparator

This is just the function name itself and does not represent a call.

The correct way to write it is:

    print_separator()

---

### 3）Confusing parameter names with actual values passed in

For example:

    def print_service_name(service_name):
        print(service_name)

Here:

- `service_name` is the parameter name
- `"nginx"` and `"mysql"` are the actual values passed in

What is inside the function is the “value”, not the original name of the external variable.

---

### 4）Only printing the value without a prompt message

For example:

    def print_service_name(service_name):
        print(service_name)

Although it can run, it does not meet the requirements of the question and is also not easy to read.

A more appropriate way to write it is:

    def print_service_name(service_name):
        print(f"The current service name is: {service_name}")

Similarly:

    def print_result_count(result_list):
        print(len(result_list))

It can be optimized to:

    def print_result_count(result_list):
        print(f"Number of results: {len(result_list)}")

---

### 5）Pasting the entire list directly in front of the conclusion message

For example:

    def print_check_result(result_list):
        if len(result_list) > 0:
            print(f"{result_list} contains errors")
        else:
            print(f"{result_list} contains no errors")

Although it can run, the output is not accurately expressed.

A more appropriate way to write it is:

    def print_check_result(result_list):
        if len(result_list)At this stage, there is no need to pursue particularly complex or specialized English variable names, but it is important to ensure that:

- They are understandable to oneself.
- They are consistent with the content stored in the variables.
- They do not conflict with the actual data.

For example:

    error_log_list
    abnormal_port_list
    status_code_list
    normal_request_list

These are all acceptable variable names at this stage.

---

### 10) When copying output statements, it is easy to miss or change the prompt text

For instance, in a question about classifying status codes, the last category should be “Number of server-side abnormal status codes,” but the prompt still reads “Number of client-side abnormal status codes.”

Such errors do not affect the main logic but can impact the accuracy of the results. After writing the code, it is necessary to check each line of the output text to ensure it matches the content of the corresponding variables.

---

## Twenty-One, Structured Writing Template for Day8

After Day8, many problems can be solved using this basic template:

    def function_name(original_list):
        result_list1 = []
        result_list2 = []
        result_list3 = []

        for current_element in original_list:
            if condition1:
                result_list1.append(current_element)
            elif condition2:
                result_list2.append(current_element)
            elif condition3:
                result_list3.append(current_element)

        print(result_list1)
        print(len(result_list1))
        print(result_list2)
        print(len(result_list2))
        print(result_list3)
        print(len(result_list3))

The core idea behind this template is:

1. Define the result lists first.
2. Iterate through the original data.
3. Classify or filter according to specific conditions.
4. Print the results after the loop is complete.

---

## Twenty-Two, Skills Acquired After Learning Day8

After completing Day8, you will gain the following skills:

### 1) Ability to define and call functions

You will begin to have the ability to encapsulate logic into functions.

### 2) Use of parameters

You will be able to use the same function to process different types of data.

### 3) Consolidation of repetitive code

Your code will no longer consist of a series of unrelated sections.

### 4) Development of basic modular thinking

Although still at an early stage, you will start to approach writing code by breaking it down into separate functions with specific purposes.

### 5) Incorporation of previously learned concepts such as loops, conditionals, list collection, and string manipulation into functions

This is the most fundamental skill improvement that Day8 aims to achieve.

### 6) Laying the foundation for return values and more comprehensive scripts

Learning about `return` statements and how multiple functions can work together will become much easier once you have mastered these concepts.

---

## Twenty-Three, What to Expect in Day9

If you have practiced Day8 thoroughly, Day9 is a great time to move on to:

- Basics of return values
- The `return` statement
- Functions that process data and then return results
- The main流程 receiving data returned by functions
- Further separation of “check logic” and “result output”

This will help make your function structure more complete.

---

## Twenty-Four, External Links

### Python Official Documentation

- Python Tutorial: Function Definition  
  https://docs.python.org/3/tutorial/controlflow.html#defining-functions

- Python Tutorial: Data Structures  
  https://docs.python.org/3/tutorial/datastructures.html

### Beginner References

- Runoob Tutorial: Python3 Functions  
  https://www.runoob.com/python3/python3-functions.html

- Runoob Tutorial: Python3 Lists  
  https://www.runoob.com/python3/python3-list.html

- Runoob Tutorial: Python String split() Method  
  https://www.runoob.com/python/att-string-split.html

### Ops-related Extended Reading

- Nginx Official Documentation  
  https://nginx.org/en/docs/

- Kubernetes Official Documentation  
  https://kubernetes.io/docs/home/

---

## Twenty-Five, Today's Key Point

**The essence of Day8 is to transform Python coding from simply writing code in a linear sequence to encapsulating reusable logic into functions and establishing a basic script structure where results are collected within loops and then outputted uniformly outside the loops.**