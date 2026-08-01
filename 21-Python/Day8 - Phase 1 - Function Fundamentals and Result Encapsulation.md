# Day8 - First Stage - Function Basics and Result Inspection Packaging

#Python #PythonLearning #TransportDevelopment #Functions #def #Parameters #CallFunctions #ResultsCheck #CheckScript #LogAnalysis #StatusCheck #Kubernetes #Linux #Obsidian

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
- Multi-branch Judgment
- `upper()` / `lower()`
- `replace()`
- `split()`

Day3 Learning Content:

- `and`
- `or`
- `not`
- Multi-condition Combination Judgment
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

Day7 Learning Content:

- Multi-condition Categorization
- Multi-result List Collection
- Categorization Statistics
- Result Summary Output
- `split()` Splitting Fixed-format Text
- Starting to Build "Inspection Result Summary" Thinking

Day8 Enters a New Stage:

**No longer just writing code sequentially from top to bottom, but starting to encapsulate "reusable inspection logic" into functions.**

This step is important because realTransport scripts often encounter the following scenarios:

- Repeated use of log inspection logic
- Repeated use of port inspection logic
- Repeated use of service status inspection logic
- Repeated use of Pod status inspection logic
- Repeated use of status code classification logic

If the same logic is rewritten every time, scripts will become longer and harder to maintain.

Therefore, Day8's core is:

**Encapsulate reusable processing logic into functions to make scripts shorter, clearer, and easier to reuse.**

---

## II. Today's Objectives

After completing Day8, you should be able to:

1. Understand what a function is
2. Use `def` to define a function
3. Call a function
4. Understand the role of parameters
5. Encapsulate simple inspection logic into a function
6. Allow the same logic to be reused
7. Lay the foundation for return values and more complete script structures

---

## III. What We'll Learn Today

Day8's content can be summarized as:

1. What is a function
2. Why use functions
3. `def` Defining a function
4. Calling a function
5. Parameter basics
6. Encapsulate simple inspection logic into a function
7. Optimize previous scripts using functions
8. Classroom practice
9. Homework

---

## IV. Why Day8 is Important

The scripts written in previous days mostly have this structure:

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

These scripts can complete tasks, but if we need to continue checking:

- Error logs
- Alarm logs
- Abnormal ports
- Faulty services
- Pending Pods

We'll repeatedly write similar logic.

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

The essence of these logics is all doing the same thing:

**Traverse a batch of data, find target results based on conditions, and output them.**

If we encapsulate such logic into a function, we can achieve:

- Shorter code
- Easier repeated calls
- Easier maintenance
- Closer to real script development

---

## V. What is a Function

A function can be simply understood as:

**Packing a piece of reusable code, giving it a name, and calling it when needed.**

You can think of a function as a "tool".

For example:

- A tool specifically for checking error logs
- A tool specifically for checking abnormal ports
- A tool specifically for checking service status
- A tool specifically for outputting inspection conclusions

Later, just by calling this tool, you don't need to rewrite the code every time.

---

## VI. Why Use Functions

The most direct value of functions has four aspects:

### 1) Reduce repetitive code

No need to rewrite repetitive logic.

### 2) Make code clearer

Seeing a function name can roughly tell you what this logic does.

### 3) Easy to modify

If the logic changes, you only need to modify the function internally once.

### 4) Closer to real development and operation scripts

When writing automation scripts later, functions are a fundamental capability.

---

## VII. The Most Basic Way to Write a Function: `def`

The basic syntax for defining a function:

    def Function Name():
        Code block

Example:

    def say_hello():
        print("hello")

This function is just defined and hasn't been executed yet.

If you want to execute it, you need to call it:

    say_hello()

Output result:

    hello

---

## VIII. Distinguish Between Function Definition and Function Call

This is a very important point for Day8.

### 1) Defining a Function

Just preparing the code:

    def say_hello():
        print("hello")

### 2) Calling a Function

Only then will it be executed:

    say_hello()

Therefore:

- Writing `def` is defining
- Writing the function name with parentheses is calling

---

## IX. Why Functions Need Parentheses After the Name

Because parentheses indicate "call".

For example: /think

print("hello")

Here, `print` is a function.

Self-defined functions follow the same rule:

    def check_nginx():
        print("nginx service check complete")

    check_nginx()

Only when you write `check_nginx()` will the code inside the function execute.

---

## 10. Day8's Most Critical New Ability: Parameters

If functions can only contain fixed content, their value is limited.  
Therefore, functions usually need "parameters".

Parameters can be initially understood as:

**Passing external data to the function for use.**

Basic syntax:

    def function_name(parameter):
        code_block

Example:

    def print_service_name(service_name):
        print(f"Current service name is: {service_name}")

Calling:

    print_service_name("nginx")
    print_service_name("mysql")

Output result:

    Current service name is: nginx
    Current service name is: mysql

Explanation:

- `service_name` is the parameter name
- Passing different values when calling the function allows the function to handle different content

---

## 11. Difference Between Parameter Names and Passed Values

This is a foundational point that must be understood on Day8.

Example:

    def print_service_name(service_name):
        print(service_name)

Here:

- `service_name` is the parameter name inside the function
- `"nginx"`, `"mysql"` are the actual values passed when calling the function

The function receives "values", not the original variable names from external variables.

Example:

    service_a = "nginx"
    service_b = "mysql"

    print_service_name(service_a)
    print_service_name(service_b)

The function internally receives:

- `"nginx"`
- `"mysql"`

The function doesn't know whether the external variable is called `service_a` or `service_b`.

Therefore, to output:

    Current service name is: nginx

The correct approach is to manually concatenate the prompt, rather than trying to retrieve the external variable name.

---

## 12. Don't Rush to Learn Complex Functions - Start with "Encapsulating Output"

On Day8, we don't pursue complex logic initially. Start with the simplest "encapsulating output".

Example:

    def print_separator():
        print("===== Inspection Start =====")

    print_separator()

This simple function helps understand:

- What it means to define a function
- What it means to call a function
- Why functions can reduce repetitive code

---

## 13. Taking It Further: Encapsulating Simple Check Logic

For example, we've written many similar logic blocks before:

    if len(abnormal_result_list) > 0:
        print("Abnormal found")
    else:
        print("Result normal")

This logic is very suitable for encapsulation into a function.

Example:

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Abnormal found")
        else:
            print("No abnormalities found")

Calling:

    abnormal_result_list = ["mysql failed", "redis failed"]

    print_check_result(abnormal_result_list)

This function means:

- Pass the result list
- The function automatically judges the quantity
- Then outputs the conclusion

---

## 14. Parameters Allow Functions to Process Different Data

Example:

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

Calling:

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    failed_service_list = ["mysql failed"]

    print_result_count(error_log_list)
    print_result_count(failed_service_list)

Output result:

    Result count: 2
    Result count: 1

This example shows:

**The same function can repeatedly process different objects as long as different data is passed in.**

---

## 15. What Logic Can Be Encapsulated on Day8

Based on the content from Day4 to Day7, the most suitable logic to encapsulate first includes:

### 1) Output Separator

    def print_separator():
        print("-----")

### 2) Output Check Title

    def print_title(title):
        print(f"===== {title} =====")

### 3) Output Result Count

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

### 4) Output Whether Abnormalities Were Found

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Abnormalities found")
        else:
            print("No abnormalities found")

These functions are not complex, but they are very suitable as introductory functions.

---

## 16. The Most Basic Complete Example

### Example 1: Encapsulate "Result Count Output"

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    warning_log_list = ["WARNING disk usage high"]

    print_result_count(error_log_list)
    print_result_count(warning_log_list)

### Example 2: Encapsulate "Whether Abnormalities Were Found" /think

```python
def print_check_result(result_list):
    if len(result_list) > 0:
        print("Abnormal found")
    else:
        print("No abnormal found")

abnormal_port_list = ["3306 closed", "22 closed"]
print_check_result(abnormal_port_list)

### Example 3: Encapsulate "Service Check Message"

def check_service(service_name):
    print(f"Checking service: {service_name}")

check_service("nginx")
check_service("mysql")
check_service("redis")

### Example 4: Encapsulate "Inspection Title"

def print_inspection_title(title):
    print(f"===== {title} =====")

print_inspection_title("Nginx Log Inspection")
print_inspection_title("MySQL Service Inspection")

---

## Seventeen, Day8's Core Value

Day8's core value lies in:

### 1) Start reducing duplicate code

Pack repeated logic into functions, avoiding rewriting it every time.

### 2) Make the script structure clearer

Write the main flow as calls, and put specific logic into functions, making the code easier to read.

### 3) Begin developing a "modularization" awareness

Although still basic, it's already approaching the idea of splitting different logic into separate functional blocks.

### 4) Prepare for return values in the future

Today we learn "definition" and "calling", making return values easier to learn later.

---

## Eighteen, Classroom Exercise

Note:

**Classroom exercises are for mastering function definition, calling, and parameter passing, without pursuing complex logic initially.**

---

### 1) print_separator.py

Requirements:

- Define a function `print_separator()`
- The function's purpose is to output a line separator:

    ====================

- Call this function 3 times

File name:

    print_separator.py

### Reference Answer

    def print_separator():
        print("====================")

    print_separator()
    print_separator()
    print_separator()

---

### 2) print_service_name.py

Requirements:

- Define a function `print_service_name(service_name)`
- The function's purpose is to output:

    Current service name is: nginx

- Call 3 times, passing in:

    "nginx"
    "mysql"
    "redis"

File name:

    print_service_name.py

### Reference Answer

    def print_service_name(service_name):
        print(f"Current service name is: {service_name}")

    print_service_name("nginx")
    print_service_name("mysql")
    print_service_name("redis")

---

### 3) print_result_count.py

Requirements:

- Define a function `print_result_count(result_list)`
- The function's purpose is to output the count of result lists

Known variables:

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    warning_log_list = ["WARNING disk usage high"]

Requirements output:

- `error_log_list` count
- `warning_log_list` count

File name:

    print_result_count.py

### Reference Answer

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

    error_log_list = ["ERROR mysql down", "ERROR redis down"]
    warning_log_list = ["WARNING disk usage high"]

    print_result_count(error_log_list)
    print_result_count(warning_log_list)

---

### 4) print_check_result.py

Requirements:

- Define a function `print_check_result(result_list)`
- If the list length is greater than 0, output:

    Abnormal found

- Otherwise output:

    No abnormal found

Known variables:

    abnormal_result_list = ["mysql failed"]
    normal_result_list = []

Requirements:

- Call the function to check these two lists

File name:

    print_check_result.py

### Reference Answer

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Abnormal found")
        else:
            print("No abnormal found")

    abnormal_result_list = ["mysql failed"]
    normal_result_list = []

    print_check_result(abnormal_result_list)
    print_check_result(normal_result_list)

---

### 5) print_inspection_title.py

Requirements:

- Define a function `print_inspection_title(title)`
- The function's purpose is to output inspection title

Output format:

    ===== Nginx Service Inspection =====

Requirements:

- Call the function to output:
  - `"Nginx Service inspection"`
  - `"MySQL Log Check"`
  - `"Kubernetes Pod Status check"`

File name:

    print_inspection_title.py

### Reference Answer

    def print_inspection_title(title):
        print(f"===== {title} =====")
```

print_inspection_title("Nginx Service Check")
print_inspection_title("MySQL Log Check")
print_inspection_title("Kubernetes Pod Status Check")

---

## 19. Post-Class Assignments

Instructions:

**The post-class assignments do not repeat classroom exercises. The focus is on "encapsulating the check logic learned earlier into functions" to make the functions applicable in realTransport scenarios.**

---

### 1I'm not sure.log_error_check_function_homework.py

Requirements:

- Define a function `check_error_log(log_list)`
- Functionality: Find logs containing `"ERROR"` from the log list
- Output:
  - Error log list
  - Error log count
  - If count > 0, output "Found error logs"
  - Else output "No error logs found"

Known variables:

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

Filename:

    log_error_check_function_homework.py

### Reference Answer

    def check_error_log(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        print(f"Error Log List: {error_log_list}")
        print(f"Error Log Count: {len(error_log_list)}")

        if len(error_log_list) > 0:
            print("Found error logs")
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

### 2I'm not sure.port_abnormal_check_function_homework.py

Requirements:

- Define a function `check_abnormal_port(port_status_list)`
- Functionality: Find port results containing `"closed"`
- Output:
  - Abnormal port list
  - Abnormal port count
  - If count > 0, output "Found abnormal ports"
  - Else output "No abnormal ports found"

Known variables:

    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

Filename:

    port_abnormal_check_function_homework.py

### Reference Answer

    def check_abnormal_port(port_status_list):
        abnormal_port_list = []

        for port_status in port_status_list:
            if "closed" in port_status:
                abnormal_port_list.append(port_status)

        print(f"Abnormal Port List: {abnormal_port_list}")
        print(f"Abnormal Port Count: {len(abnormal_port_list)}")

        if len(abnormal_port_list) > 0:
            print("Found abnormal ports")
        else:
            print("No abnormal ports found")


    port_status_list = ["80 open", "443 open", "3306 closed", "22 closed"]

    check_abnormal_port(port_status_list)

---

### 3I'm not sure.service_status_count_function_homework.py

Requirements:

- Define a function `count_service_status(service_status_list)`
- Functionality: Count:
  - `running`
  - `stopped`
  - `failed`

Known variables:

    service_status_list = ["running", "stopped", "failed", "running", "failed"]

Required output:

- Running service count
- Stopped service count
- Failed service count

Filename:

    service_status_count_function_homework.py

### Reference Answer

    def count_service_status(service_status_list):
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

print(f"Running Service List: {running_service_list}")
        print(f"Running Service Count: {len(running_service_list)}")
        print(f"Stopped Service List: {stopped_service_list}")
        print(f"Stopped Service Count: {len(stopped_service_list)}")
        print(f"Failed Service List: {failed_service_list}")
        print(f"Failed Service Count: {len(failed_service_list)}")


    service_status_list = ["running", "stopped", "failed", "running", "failed"]

    count_service_status(service_status_list)

---

### 4I'm not sure.http_status_classify_function_homework.py

Requirements:

- Define a function `classify_status_code(status_code_list)`
- The function should classify status codes into:
  - Normal status code list
  - Client error status code list
  - Server error status code list

Known variables:

    status_code_list = ["200", "404", "500", "200", "502"]

Required output:

- Three result lists
- Count of each status code category

File name:

    http_status_classify_function_homework.py

### Reference Answer

    def classify_status_code(status_code_list):
        normal_status_code_list = []
        client_error_status_code_list = []
        server_error_status_code_list = []

        for status_code in status_code_list:
            if status_code.startswith("2"):
                normal_status_code_list.append(status_code)
            elif status_code.startswith("4"):
                client_error_status_code_list.append(status_code)
            elif status_code.startswith("5"):
                server_error_status_code_list.append(status_code)

        print(f"Normal Status Code List: {normal_status_code_list}")
        print(f"Normal Status Code Count: {len(normal_status_code_list)}")
        print(f"Client Error Status Code List: {client_error_status_code_list}")
        print(f"Client Error Status Code Count: {len(client_error_status_code_list)}")
        print(f"Server Error Status Code List: {server_error_status_code_list}")
        print(f"Server Error Status Code Count: {len(server_error_status_code_list)}")


    status_code_list = ["200", "404", "500", "200", "502"]

    classify_status_code(status_code_list)

---

### 5I'm not sure.http_log_status_check_function_homework.py

Requirements:

- Define a function `check_http_log_status(access_log_list)`
- The function should:
  - First use `split()` to split each line of access log
  - Extract status codes
  - Categorize requests by status code into:
    - Normal request list
    - Client error request list
    - Server error request list

Known variables:

    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200"
    ]

Required output:

- Normal request list
- Client error request list
- Server error request list
- Count of each request category

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
                client_error_request_list.append(access_log)
            elif status_code.startswith("5"):
                server_error_request_list.append(access_log)

print(f"Normal request list: {normal_request_list}")
        print(f"Normal request count: {len(normal_request_list)}")
        print(f"Client error request list: {client_error_request_list}")
        print(f"Client error request count: {len(client_error_request_list)}")
        print(f"Server error request list: {server_error_request_list}")
        print(f"Server error request count: {len(server_error_request_list)}")


    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200"
    ]

    check_http_log_status(access_log_list)

---

## 20. Common Mistakes During Problem-Solving

The following errors are very typical in Day8 practice and need to be paid attention to continuously.

### 1) Defined a function but did not call it

Example:

    def print_separator():
        print("-----")

If you do not write:

    print_separator()

The function will not execute.

---

### 2) Forgetting to add parentheses after the function name

Example:

    print_separator

This is just the function name itself, not a call.

Correct syntax:

    print_separator()

---

### 3) Confusing parameter names with actual passed values

Example:

    def print_service_name(service_name):
        print(service_name)

Here:

- `service_name` is the parameter name
- `"nginx"`, `"mysql"` are the actual passed values

The function internally receives "values", not the original variable names from the outside.

---

### 4) Only printing values without prompt messages

Example:

    def print_service_name(service_name):
        print(service_name)

Although it can run, it doesn't meet the question requirements and is not reader-friendly.

A better approach is:

    def print_service_name(service_name):
        print(f"Current service name: {service_name}")

Similarly:

    def print_result_count(result_list):
        print(len(result_list))

Can be optimized to:

    def print_result_count(result_list):
        print(f"Result count: {len(result_list)}")

---

### 5) Directly concatenating the entire list to the conclusion message

Example:

    def print_check_result(result_list):
        if len(result_list) > 0:
            print(f"{result_list} found errors")
        else:
            print(f"{result_list} no errors found")

Although it can run, the output is imprecise.

A better approach is:

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Found errors")
        else:
            print("No errors found")

---

### 6) Combining "collecting results" and "outputting conclusions" in the same loop

This is a critical error point in Day8.

Incorrect example:

    def check_error_log(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)
                print(error_log_list)

            if len(error_log_list) > 0:
                print("Found error logs")
            else:
                print("No error logs found")

The issue is:

- Outputting once per loop iteration
- Drawing conclusions prematurely before full data traversal
- Mixing intermediate states with final results

Correct approach:

**Collect results in the loop, analyze and draw conclusions outside the loop.**

Correct example:

    def check_error_log(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        print(error_log_list)

        if len(error_log_list) > 0:
            print("Found error logs")
        else:
            print("No error logs found")

---

### 7) Confusing list variable with current element variable

Example:

    def check_error_log(log_list):
        for log_line in log_list:
            if "ERROR" in log_list:
                print(log_line)

The judgment object is wrong.  
Should judge the current element `log_line`, not the entire list `log_list`.

Correct syntax:

    if "ERROR" in log_line:

---

### 8) Not splitting fixed-format text before extracting fields

Example with access logs:

    "10.0.0.2 GET /login 404"

If extracting status codes, it's better to first split: /think

```python
parts = access_log.split()
status_code = parts[3]
```

---

### 9I'm not sure.Variable names should closely match their content

At this stage, there's no need to pursue overly complex or highly specialized English variable names, but the following should be achieved:

- You should be able to understand them yourself
- They should closely match the content they store
- They should not conflict with actual content

Examples like:

    error_log_list
    abnormal_port_list
    status_code_list
    normal_request_list

are all acceptable variable names at this stage.

---

### 10I'm not sure.When copying output statements, prompts are often missed

For example, in the status code classification problem, the last category is already "Number of server-side error status codes", but the prompt still reads as "Number of client-side error status codes".

Such errors don't affect the main logic, but they impact the accuracy of result expression.  
After writing, you need to check each line to ensure the output text matches the variable content.

---

## Twenty-one, Day8's Structured Writing Template

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

The core behind this template is:

1. Define result lists first
2. Traverse the original data
3. Classify or filter by conditions
4. Output results uniformly after the loop

---

## Twenty-two, Abilities Gained After Completing Day8

After completing Day8, you'll gain these abilities:

### 1I'm not sure.Define and call functions

You'll start to have the ability to encapsulate logic.

### 2I'm not sure.Use parameters

The same function can process different data.

### 3I'm not sure.Organize repeated logic

Code no longer just piles up line by line.

### 4I'm not sure.Begin to develop basic modular thinking

Although still at an early stage, you're starting to approach the practice of "splitting different functions".

### 5I'm not sure.Integrate previously learned loops, conditions, list collection, and string processing into functions

This is the core ability improvement of Day8.

### 6I'm not sure.Lay the foundation for return values and more complete scripts

It will be easier to understand when learning `return` and multiple function collaboration later.

---

## Twenty-three, What Day9 is Likely to Cover

If Day8 is mastered, Day9 is suitable for entering:

- Basic return values
- `return`
- Functions processing results and returning data
- Main flow receiving function return results
- Further separating "check logic" and "result output"

This will make the function system more complete.

---

## Twenty-four, External Links

### Python Official Documentation

- Python Tutorial: Function Definition  
  https://docs.python.org/3/tutorial/controlflow.html#defining-functions

- Python Tutorial: Data Structures  
  https://docs.python.org/3/tutorial/datastructures.html

### Beginner References

- CSDN Tutorial: Python3 Functions  
  https://www.runoob.com/python3/python3-functions.html

- CSDN Tutorial: Python3 Lists  
  https://www.runoob.com/python3/python3-list.html

- CSDN Tutorial: Python String split() Method  
  https://www.runoob.com/python/att-string-split.html

### Operations Understanding

- Nginx Official Documentation  
  https://nginx.org/en/docs/

- Kubernetes Official Documentation  
  https://kubernetes.io/docs/home/

---

## Twenty-five, Today's One-Sentence Summary

**Day8's essence is to upgrade Python from "writing code sequentially" to "encapsulating reusable logic into functions", and begin forming the basic script structure of "collecting results within loops and outputting uniformly outside loops".**