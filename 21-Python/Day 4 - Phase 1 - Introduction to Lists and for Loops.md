# Day4 - Phase 1 - Introduction to Lists and for Loops

#Python #Python Learning #Ops Development #Lists #for Loops #Batch Processing #Log Analysis #Kubernetes #Linux #Obsidian

## Today's Focus

From Day1 learning content:

- Variables
- Strings
- `print()`
- `f-string`
- `in`
- `strip()`
- `startswith()`
- `endswith()`

From Day2 learning content:

- `if / elif / else`
- Multi-branch judgments
- `upper()` / `lower()`
- `replace()`
- `split()`

From Day3 learning content:

- `and`
- `or`
- `not`
- Multi-condition combined judgments
- `in`, `not in`
- `startswith()`, `endswith()`

Day4 marks the beginning of a very crucial new phase:

**Moving from “processing one value” to “batch-processing multiple values”**

This is an important milestone where Python begins to demonstrate its automation capabilities.

In real Ops scenarios, it's rare to deal with just one object. More often, we need to:

- Check the status of multiple Pods at once
- Verify multiple ports simultaneously
- Analyze multiple logs in one go
- Process multiple file names together
- Traverse multiple nodes sequentially

Therefore, the core focus of Day4 is not just to learn a few new syntaxes but to develop a new way of thinking:

**Making programs automatically repeat the same action many times**

---

## Today's Goals

After completing Day4, you should be able to:

1. Understand what a list `list` is.
2. Define lists.
3. View the contents of lists.
4. Retrieve elements from lists using indices.
5. Comprehend what a `for` loop is.
6. Use `for` loops to traverse lists.
7. Write basic batch processing scripts.
8. Establish the mindset of “repeating the same operation on multiple objects”.

---

## I. What We Will Learn Today

Day4 covers the following topics:

1. Lists `list`
2. Elements within lists
3. Indices
4. `for` loops
5. Traversing lists
6. Batch processing scenarios in Ops
7. Day4 practice exercises

---

## II. Why Day4 Is Important

From Day1 to Day3, most of the code was used for tasks like:

- Checking the status of a single Pod
- Verifying a single file name
- Analyzing a single log
- Testing a single port

However, in real Ops, we often deal with multiple objects at once:

- A group of Pods
- A set of logs
- Multiple nodes
- A batch of ports
- Several service names

If we continue to handle these tasks manually one by one, it will quickly become unmanageable:

    pod1 = "Running"
    pod2 = "Pending"
    pod3 = "CrashLoopBackOff"

This approach becomes inefficient when the number of objects increases.

That's where Day4 comes in:

**Put multiple values into a container and then process them in batches**

This container is the `list`.  
And the action of “batch processing” mainly relies on the `for` loop.

---

## III. What Is a List `list`

A list can be understood as:

**Placing multiple values in order within a single variable**

For example:

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

In this variable, there is no longer just one value but multiple values.

### You can think of it this way first:

- Strings: Used to store text.
- Integers: Used to store numbers.
- Lists: Used to store multiple values.

---

## IV. What Does a List Look Like

The most noticeable features of a list are:

- It is enclosed in square brackets `[]`.
- Multiple elements are separated by commas.

For example:

    ports = [22, 80, 443]
    pod_list = ["pod-a", "pod-b", "pod-c"]
    log_levels = ["INFO", "WARNING", "ERROR"]

---

## V. Elements Within a List

Each item inside a list is called an “element”.

For example:

    ports = [22, 80, 443]

Here, there are 3 elements:

- 22
- 80
- 443

Another example:

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

Here too, there are 3 elements:

- `"Running"`
- `"Pending"`
- `"CrashLoopBackOff"`

---

## VI. What Types of Data Can a List Store

Lists can hold many different types of data.

### 1) Strings

    services = ["nginx", "docker", "kubelet"]

### 2) Numbers

    ports = [2    cd ~/python-study/day4

Recommended practice files:

    list_basic.py
    list_index_demo.py
    pod_list_print.py
    port_list_print.py
    pod_status_batch_check.py
    log_level.batch_check.py

Execution method:

    python3 filename.py

The focus of this section is to observe the following points:

- Whether the list is defined correctly.
- Whether the `for` loop syntax is correct.
- Whether the indentation is correct.
- Whether the judgment logic is placed inside the loop.
- Whether the output results meet expectations.

---

## Section 18: Day4 Example Exercises

These can be used first to understand lists and `for` loops.

### 1) list_basic.py

Requirement: Define a list and print the entire list.

    pod_list = ["pod-a", "pod-b", "pod-c"]
    print(pod_list)

---

### 2) list_index_demo.py

Requirement: Access elements in the list using indices.

    ports = [22, 80, 443]

    print(ports[0])
    print(ports[1])
    print(ports[2])

---

### 3) pod_list_print.py

Requirement: Use a `for` loop to print multiple Pod names.

    pod_list = ["nginx-pod", "redis-pod", "mysql-pod"]

    for pod in pod_list:
        print(pod)

---

### 4) port_list_print.py

Requirement: Use a `for` loop to print multiple ports.

    ports = [22, 80, 443]

    for port in ports:
        print(port)

---

### 5) pod_status_batch_check.py

Requirement: Batch-check Pod status and output abnormal states.

    pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

    for status in pod_status_list:
        if status != "Running":
            print("Abnormal Pod status found:", status)

---

### 6) log_level_batch_check.py

Requirement: Batch-identify log levels that need attention.

    log_levels = ["INFO", "WARNING", "ERROR"]

    for level in log_levels:
        if level == "WARNING" or level == "ERROR":
            print("Log levels that require attention:", level)

---

## Section 19: Day4 Homework (with Answers)

It is recommended to complete these tasks independently first, and then check against the answers.

Recommended order:

1. Create practice files first.
2. Write the code independently as required.
3. Run the tests.
4. Check against the answers and make corrections.

This way, you can better grasp the core concepts of Day4.

---

### Homework 1: Batch Print Node Names

#### Recommended practice file name

    node_list_print.py

#### Reference answer file name

    node_list_print_answer.py

#### Known variables

    node_list = ["master-1", "node-1", "node-2"]

#### Requirement

Use a `for` loop to print each node name on a separate line.

#### Reference answer

    node_list = ["master-1", "node-1", "node-2"]

    for node in node_list:
        print(node)

---

### Homework 2: Batch Check Pod Status

#### Recommended practice file name

    pod_status_check.py

#### Reference answer file name

    pod_status_check_answer.py

#### Known variables

    pod_status_list = ["Running", "Pending", "Running", "CrashLoopBackOff"]

#### Requirement

Use a `for` loop to iterate through all statuses.  
If the status is not `Running`, output:

    Abnormal Pod status found: Status value

#### Reference answer

    pod_status_list = ["Running", "Pending", "Running", "CrashLoopBackOff"]

    for pod_status in pod_status_list:
        if pod_status != "Running":
            print(f"Abnormal Pod status found: {pod_status}")

---

### Homework 3: Batch Identify Web Ports

#### Recommended practice file name

    web_port_check.py

#### Reference answer file name

    web_port_check_answer.py

#### Known variables

    ports = [22, 80, 443, 3306]

#### Requirement

Use a `for` loop to iterate through all ports.  
If the port is 80 or 443, output:

    Web service port: Port value

#### Reference answer

    ports = [22, 80, 443, 3306]

    for port in ports:
        if port == 80 or port == 443:
            print(f"Web service port: {port}")

---

### Homework 4: Batch Identify Error Logs

#### Recommended practice file name

    error_log_batch_check.py

#### Reference answer file name

    error_log.batch_check_answer.py

#### Known variables

    log```markdown
ports = [22, 80, 443]
print(ports)

This will print the entire list at once, rather than printing each element individually.

If you want to process them one by one, you should use a `for` loop:

for port in ports:
    print(port)
```

---

### 2) Incorrect indentation of `for` loops

#### Wrong example

ports = [22, 80, 443]

for port in ports:
    print(port)

#### Correct example

ports = [22, 80, 443]

for port in ports:
    print(port)

#### Explanation

The code that follows a `for` loop must be indented.

---

### 3) Poor readability due to the same name for the loop variable and the list variable

#### Not recommended practice

ports = [22, 80, 443]

for ports in ports:
    print(ports)

#### Better practice

ports = [22, 80, 443]

for port in ports:
    print(port)

#### Explanation

It is recommended to use different names for the list and its elements:

- Use a plural name or `_list` for the list
- Use a singular name for an individual element

For example:

- `ports` / `port`
- `pod_list` / `pod`
- `log_list` / `log_line`

---

### 4) Incorrect use of array indexing

#### Wrong example

ports = [22, 80, 443]
print(ports[3])

#### Correct understanding

This list has only 3 elements, so the indices are:

- `0`
- `1`
- `2`

Therefore, `ports[3]` will result in an error.

---

### 5) Only looping and printing without any conditional checks

#### Basic example

pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

for status in pod_status_list:
    print(status)

#### Example closer to real-world operations

pod_status_list = ["Running", "Pending", "CrashLoopBackOff"]

for status in pod_status_list:
    if status != "Running":
        print("Abnormal status:", status)
```

#### Explanation

The key point of Day4 is not just “looping through”, but also:

**performing conditional checks during the loop**

---

## 21. Core concepts that must be mastered by Day4

By Day4, you should start developing this way of thinking:

### Thinking from Day1 to Day3

- Process a single value
- Check the value
- Output a result

### Thinking for Day4

- Put multiple values into a list
- Use `for` to access each value one by one
- Apply the same logic to each value

This step is crucial because many automated scripts are based on:

1. Preparing a set of data
2. Processing them in loops
3. Outputting or acting on specific results

---

## 22. Summary of Day4 learning

The core of Day4 is not just learning about lists and `for` loops, but also starting to deal with “batch processing” for the first time.

This means that the way you write scripts is changing:

- You no longer focus on a single value
- But start considering multiple objects at once

For example:

- A group of Pods
- A set of logs
- A collection of nodes
- A list of ports
- A batch of service names

This marks the beginning of automated thinking.

---

## 23. Abilities acquired after completing Day4

By this point, you have gained the following abilities:

### 1) Defining lists

You can store multiple values in a single variable for unified management.

### 2) Accessing list elements

You understand that lists are ordered and that indices start at 0.

### 3) Using `for` loops

You can extract each element from a list one by one.

### 4) Performing basic batch processing

For example:

- Batch printing
- Batch checking
- Filtering and collecting results based on conditions

### 5) Getting closer to real-world operational scripts

Many operational tasks essentially involve:

- Collecting a group of objects
- Checking each one individually
- Identifying any exceptions

---

## 24. What will be covered in Day5

If you have mastered Day4, Day5 is ideal for moving on to:

- Combining lists with strings
- Performing statistical calculations
- Using `append()` to add elements
- Collecting results after batch filtering
- Working on more advanced log analysis and bulk inspection scripts

In other words, you will progress from:

- Simply looping and printing

to:

- Looping, checking, filtering, and collecting data

---

## 25. External links

### Python official documentation

- Python