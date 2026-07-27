# Day9 - Phase 1: Basics of Return Values

#Python #Python Learning #Ops Development #Functions #return #Return Values #Parameters #Result Processing #Script Encapsulation #Log Analysis #Status Check #Kubernetes #Linux #Obsidian

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
- Indexing
- `for` loops
- Traversing lists
- Using `for + if` for basic batch judgments

Learning content for Day5:

- Result collection
- `append()`
- Basic counting
- Batch filtering
- Collecting results and outputting statistics

Learning content for Day6:

- `len()` for counting length
- Checking the number of result lists
- Checking empty lists
- The script mindset of "continuing judgments after collecting results"

Learning content for Day7:

- Multi-condition classification
- Collecting multiple result lists
- Classifying and tallying
- Summarizing and outputting results
- Using `split()` to parse fixed-format text
- Beginning to develop the idea of "inspecting results in summary"

Learning content for Day8:

- The `def` function statement
- Function invocation
- Parameters
- Encapsulating repetitive logic into functions
- Collecting results inside loops and outputting them collectively outside loops

The new phase starting with Day9 is:

**Functions are no longer just used to "print results directly inside them"; instead, they begin to return processed results for external use.**

This step is crucial because in real-world scripts, many functions are not designed to print results for their own sake but rather to:

- Pass inspection results to the main process for further evaluation
- Deliver filtered lists to subsequent functions for further processing
- Return statistical data for subsequent output, alerts, or file writing

Therefore, the core focus of Day9 is:

**Understanding `return` and enabling functions to evolve from "only printing results" to "being able to return results."**

---

## II. Today's Goals

After completing Day9, you should be able to:

1. Understand what a return value is.
2. Know the difference between `print()` and `return`.
3. Use `return` to return a result.
4. Save the result returned by a function in a variable.
5. Continue to process, count, or output the returned value.
6. Lay the foundation for multiple functions collaborating in Day10.

---

## III. What Will Be Learned Today

The content of Day9 can be summarized as follows:

1. What a return value is.
2. Why `return` is used.
3. The difference between `print()` and `return`.
4. Returning a single result.
5. Returning a list of results.
6. The main process receiving the results returned by a function.
7. Further processing of the returned values.
8. In-class exercises.
9. Homework.

---

## IV. Why Day9 Is Important

By the end of Day8, you should already be able to encapsulate logic into functions, such as:

    def check_error_log(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        print(f"Error log list: {error_log_list}")
        print(f"Number of error logs: {len(error_log_list)}")

Such functions can be executed, and their output can be seen.

However, if you want to do the following things later on:

- Decide whether to trigger an alert after counting the number of error logs.
- Pass the error log list to another function for further processing.
- Write the list of abnormal ports to a file.
- Send the list of failed services to a webhook notification module.

Then simply "printing results inside the function" is no longer sufficient.

In such cases, it is better to:

**Have the function return the result, and let the main process decide how to use it later.**

For example:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        return error_log_list

In the main process, you can then write:

    result = get_error_log_list(log_list)
    print(result)
    print(len(result))

This wayfailed_service_list = get_failed_service_list(service_status_list)
print(f"Failed service list: {failed_service_list}")
print(f"Number of failed services: {len(failed_service_list)}")

This is a very important practical application of Day9.

---

## XII. Differences Between Day8 and Day9

### Day8 focuses more on:

- Directly using `print()` inside functions
- The main goal is to learn how to encapsulate logic first

For example:

    def print_check_result(result_list):
        if len(result_list) > 0:
            print("Abnormalities detected")
        else:
            print("No abnormalities found")

---

### Day9 focuses more on:

- Using `return` inside functions
- The main process receives the returned value and processes it further

For example:

    def get_check_result(result_list):
        if len(result_list) > 0:
            return "Abnormalities detected"
        else:
            return "No abnormalities found"

    result = get_check_result(["mysql failed"])
    print(result)

---

## XIII. Don't Pursue Complexity Yet; First Learn to “Return a Result”

Example:

    def get_separator():
        return "===================="

    line = get Separator()
    print(line)

Although this example is simple, it helps understand that:

- Functions can return content
- The returned value must be received and processed
- After receiving it, it can be printed

---

## XIV. Take It One Step Further: Return a Count

Example:

    def get_result_count(result_list):
        return len(result_list)

    error_log_list = ["ERROR mysql down", "ERROR redis down"]

    count = get_result_count(error_log_list)
    print(f"Number of results: {count}")

This example shows that:

**Functions can return the results of their processing, rather than just printing them themselves.**

---

## XV. Take It One Step Further: Return a Filtered List

Example:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)

        return error_log_list

    log_list = [
        "INFO nginx started",
        "ERROR mysql down",
        "WARNING disk usage high",
        "ERROR redis connect failed"
    ]

    result = get_error_log_list(log_list)
    print(f"Error log list: {result}")
    print(f"Number of error logs: {len(result)}")

---

## XVI. The Core Value of Day9

The core value of Day9 lies in:

### 1) Functions truly begin to have the ability to “produce results”

They no longer just print content themselves; instead, they pass the results on to other parts of the program.

### 2) The responsibilities of the main process and functions start to separate

- Functions are responsible for processing data
- The main process is responsible for receiving the results and deciding how to use them

### 3) It prepares the way for multiple functions to work together

This will become even more evident in Day10: the return value of one function can be used as input for another function.

### 4) It brings us closer to real-world script development

In real-world scripts, many functions are not designed to “print results” themselves; instead, they are meant to “return results for use elsewhere.”

---

## XVII. The Most Basic Complete Examples

### Example 1: Returning a String

    def get_service_name():
        return "nginx"

    service_name = get_service_name()
    print(service_name)

### Example 2: Returning a Count

    def get_result_count(result_list):
        return len(result_list)

    warning_log_list = ["WARNING disk usage high"]
    count = get_result_count(warning_log_list)
    print(f"Number of results: {count}")

### Example 3: Returning a Check Conclusion

    def get_check_result(result_list):
        if len(result_list) > 0:
            return "Abnormalities detected"
        else:
            return "No abnormalities found"

    result = get_check_result(["3306 closed"])
    print(result)

### Example 4: Returning a Filtered List

    def get_closed_port_list(port_status_list):
        closed_port_list = []

        for port_status in port_status_list:
            if "closed" in port_status:
                closed_port_list.append(port_status)

        return closed_port_list

    port_status_list = ["80 open", "3306 closed", "22 closed"]
    result = get_closed_port_list(port_status_list)
    print(result)

---

## XVIII. Class Exercises

Note:

**These class exercises are designed to help you master the basic use of `return`. The focus is on “returning results” rather than just “printing them.”**

---

### 1) get_separator.py

Requirements:

- Define a function `get_separator()`
- The### Detection of Abnormalities

- Otherwise, return:

    No abnormalities detected

Known variables:

    abnormal_result_list = ["mysql failed"]
    normal_result_list = []

Requirements:

- Obtain the inspection conclusions for both lists separately.
- Print the conclusions outside the function.

File name:

    get_check_result.py

Reference answer:

    def get_check_result(result_list):
        if len(result_list) > 0:
            return "Detection of abnormalities"
        else:
            return "No abnormalities detected"

    abnormal_result_list = ["mysql failed"]
    normal_result_list = []

    abnormal_result = get_check_result(abnormal_result_list)
    normal_result = get_check_result(normal_result_list)

    print(f"Inspection conclusion for abnormal result list: {abnormal_result}")
    print(f"Inspection conclusion for normal result list: {normal_result}")- Use `return` to return a list of server-side error status codes

Given variables:

    status_code_list = ["200", "404", "500", "200", "502"]

Requirements:

- Receive the return value outside the function
- Output:
  - The list of server-side error status codes
  - The number of server-side error status codes

File name:

    http_status_return_homework.py

Reference answer:

    def get_server_error_status_code_list(status_code_list):
        server_error_status_code_list = []

        for status_code in status_code_list:
            if status_code.startswith("5"):
                server_error_status_code_list.append(status_code)

        return server_error_status_code_list

    status_code_list = ["200", "404", "500", "200", "502"]
    server_error_status_code_list = get_server_error_status_code_list(status_code_list)

    print(f"List of server-side error status codes: {server_error_status_code_list}")
    print(f"Number of server-side error status codes: {len(server_error_status_code_list)}")

---

### 5) http_log_status_return_homework.py

Requirements:

- Define a function `get_client_error_request_list(access_log_list)`
- Functionality:
  - First, use `split()` to split each line of the access log
  - Extract the status code
  - Identify all client-side error requests, i.e., requests with a status code starting with "4"
- Do not print directly inside the function
- Use `return` to return the list of client-side error requests

Given variables:

    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200"
    ]

Requirements:

- Receive the return value outside the function
- Output:
  - The list of client-side error requests
  - The number of client-side error requests

File name:

    http_log_status_return_homework.py

Reference answer:

    def get_client_error_request_list(access_log_list):
        client_error_request_list = []

        for access_log_line in access_log_list:
            parts = access_log_line.split()
            status_code = parts[3]

            if status_code.startswith("4"):
                client_error_request_list.append(access_log_line)

        return client_error_request_list

    access_log_list = [
        "10.0.0.1 GET /index 200",
        "10.0.0.2 GET /login 404",
        "10.0.0.3 POST /api/order 500",
        "10.0.0.4 GET /health 200"
    ]

    client_error_request_list = get_client_error_request_list(access_log_list)

    print(f"List of client-side error requests: {client_error_request_list}")
    print(f"Number of client-side error requests: {len(client_error_request_list)}")

---

## Twenty, Day9 Common Errors and Mistakes

### 1) Confusing `return` with `print()`

For example:

    def get_service_name():
        print("nginx")

This is just printing the result; it does not return anything.

If you want to return a value, you should write:

    def get_service_name():
        return "nginx"

---

### 2) The function has already used `return`, but nothing outside receives it

For example:

    def get_port():
        return 3306

    get_port()

Although the function is executed, the returned result is not used later on.

A more appropriate way to write it is:

    port = get_port()
    print(port)

---

### 3) Wanting the function to return a result, but still writing the main logic as a direct print

For example:

    def get_result_count(result_list):
        print(len(result_list))

If the question asks for "the number of results", you should write:

    def get_result_count(result_list):
        return len(result_list)

---

### 4) Using `return` inside a loop, causing it to end prematurely

For example:

    def get_error_log_list(log_list):
        error_log_list = []

        for log_line in log_list:
            if "ERROR" in log_line:
                error_log_list.append(log_line)
            return error_log_list

This will return after the first iteration, and the remaining data will not be processed.

The correct approach is:

- Complete the entire loop
- Then return the result at the end

Correct way to write it:

    def get_error_log_list(log_list):
        error_log_list = []

    result = get_result_count(warning_log_list)

It can run, but the readability is average.

A clearer way to write it would be:

    error_count = get_result_count(error_log_list)
    warning_count = get_result_count(warning_log_list)

It's recommended to develop this habit gradually:

- For temporary results that are immediately printed, you can use `result`.
- When you need to retain the results for a longer time or distinguish them based on their business meaning, try to include the business context in the variable names.

---

## 21. Structured Problem-Solving Template for Day9

Many problems on Day9 can be solved using this basic template:

    def function_name(original_list):
        result_list = []

        for current_element in original_list:
            if condition_is_met:
                result_list.append(current_element)

        return result_list

    result = function_name(original_list)

    print(f"Result list: {result}")
    print(f"Number of results: {len(result)}")

    if len(result) > 0:
        print("Abnormalities found")
    else:
        print("No abnormalities found")

If the function returns a single value, it can also be written like this:

    def function_name(original_list):
        return len(original_list)

    count = function_name(original_list)
    print(count)

If the function returns a formatted string, it can be written as follows:

    def function_name(original_value):
        return f"Formatted result: {original_value}"

    result = function_name(original_value)
    print(result)

The core idea behind this template is:

1. The function is responsible for processing the data.
2. `return` is used to pass the results on.
3. The main flow is responsible for receiving and displaying the results.

---

## 22. Skills Acquired After Completing Day9

After completing Day9, you will gain the following skills:

### 1) Understanding Return Values

You will know that functions can not only print results but also return them.

### 2) Using `return`

You will be able to pass the processed results to other parts of the program for further use.

### 3) Receiving Function Returns

You will start to develop the awareness that "functions handle the processing, and the main flow uses the results."

### 4) Preparing for Multiple Functions Working Together

The return value from one function can be used by another function.

### 5) Moving Towards More Real-World Script Structures

Functions will gradually evolve from being solely for display to producing actual results.

---

## 23. What You Are Likely to Work On in Day10

If you have mastered Day9, Day10 is a good time to start:

- Splitting a small script into multiple functions.
- Having one function for filtering, another for counting, and yet another for outputting.
- Starting to form clearer main flow structures.

This will make subsequent tasks such as file processing, command execution, and log analysis much more straightforward.

---

## 24. External Links

### Python Official Documentation

- Python Tutorial: Function Definition  
  https://docs.python.org/3/tutorial/controlflow.html#defining-functions

- Python Tutorial: More About Functions  
  https://docs.python.org/3/tutorial/controlflow.html#more-on-defining-functions

### Beginner References

- Runoob Tutorial: Python3 Functions  
  https://www.runoob.com/python3/python3-functions.html

- Runoob Tutorial: Python3 Return Values  
  https://www.runoob.com/python3/python3-function.html

### Advanced Topics for Operations and Maintenance

- Nginx Official Documentation  
  https://nginx.org/en/docs/

- Kubernetes Official Documentation  
  https://kubernetes.io/docs/home/

---

## 25. Today's Summary

### 1) One-Sentence Summary

**The essence of Day9 is to upgrade functions from directly displaying results to using `return` to pass the results back to the main flow for further processing. This marks an important step towards more efficient function collaboration.**

### 2) Today's Skills

After completing Day9, you should have acquired the following basic skills:

- Distinguish between `print()` and `return`.
- Be able to write functions that return single values.
- Be able to write functions that return lists of results.
- Be able to receive return values outside of functions.
- Be able to further process, analyze, and display these return values.
- Understand the basic division of labor where "functions handle processing, and the main flow uses the results."

### 3) Today's Thinking Progress

The focus of Day8 was to "encapsulate repetitive logic into functions."
The focus of Day9 is to "give functions the ability to produce results."

This means that script structures are starting to evolve from:

- Functions doing everything by themselves

to:

- Functions handling specific tasks,
- The main flow managing the overall process,
- Different functions collaborating through return values