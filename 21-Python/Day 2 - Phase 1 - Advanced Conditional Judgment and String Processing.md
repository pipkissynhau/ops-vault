# Day2 - Phase 1 - Advanced Conditional Judgment and String Processing

#Python #Python Learning #Ops Development #Python Basics #Conditional Judgment #String Processing #Log Analysis #Kubernetes #Linux #Obsidian

## Today's Focus

Day2 marks the beginning of moving from "being able to write Python code" to "making the code capable of making judgments."

In Day1, you have already learned:

- Variables
- Strings
- print()
- f-string
- in
- strip()
- endswith()
- The most basic if statement

On Day2, we will take another step forward. The focus is not on memorizing syntax, but on learning how to:

- Output different results based on different situations
- Perform further processing on strings
- Write simple Ops judgment logic into Python scripts

This step is crucial because, whether it's log analysis, status monitoring, alarm judgment, or automated processing, everything essentially involves "judging first, then processing."

---

## Today's Goals

After completing Day2, you should be able to:

1. Write `if / elif / else` statements
2. Output different results depending on the state
3. Determine whether a keyword is present in logs
4. Handle cases where case sensitivity varies
5. Use string methods for simple cleaning and splitting
6. Create several practical Ops scripts

---

## I. What to Learn Today

Day2 covers the following topics:

1. `if / elif / else` statements
2. Multiple conditional judgments
3. Additional comparison operators
4. `lower()` and `upper()`
5. `startswith()`
6. `replace()`
7. `split()`
8. Practice in Ops scenarios

---

## II. WhyOps Must Learn Conditional Judgment

In Ops scenarios, most scripts are not "mindlessly executed." Instead, they often perform actions such as:

- Triggering an alarm if a log contains "ERROR"
- Reporting an exception if a Pod is not in the "Running" state
- Issuing a high-risk warning if CPU usage exceeds a threshold
- Ignoring files that do not end with ".log"
- Initiating troubleshooting procedures if the status does not meet expectations

Therefore, Python's judgment statements essentially help you translate the Ops logic in your mind into code.

---

## III. Conditional Judgment Structures

### 1) Basic Structure

Python's multi-branch judgment structure is as follows:

    if condition1:
        Execute when condition1 is true
    elif condition2:
        Execute when condition2 is true
    else:
        Execute when none of the previous conditions are true

### 2) How to Understand It

- `if`: First, check the first condition.
- `elif`: If the first condition is false, check the next one.
- `else`: If none of the previous conditions are true, execute this block.

### 3) Important Notes

- Python uses indentation to define code blocks.
- Indentation at the same level must be consistent.
- `if`, `elif`, and `else` form a group, and only one branch will be executed.

---

## IV. Log Level Judgment

This is a typical Ops scenario.

### Example Code

    log_line = "ERROR kube-apiserver connection refused"

    if "ERROR" in log_line:
        print("This is an error log that needs immediate attention.")
    elif "WARNING" in log_line:
        print("This is an alarm log that requires monitoring.")
    else:
        print("This is a regular log.")

### Explanation

- `"ERROR" in log_line`: Checks if the string contains "ERROR".
- If the first condition is true, the subsequent `elif` and `else` blocks will not be executed.
- This pattern is ideal for preliminary log classification.

### Operational Significance

You can expand this logic to:

- Match more log levels
- Save error logs in specific files
- Count the number of logs at different levels
- Trigger alarm notifications accordingly

---

## V. Service Status Judgment

This approach is suitable for scenarios where status values are fixed, such as Pod status, service status, or task status.

### Example Code

    pod_status = "CrashLoopBackOff"

    if pod_status == "Running":
        print("The Pod is running normally.")
    elif pod_status == "Pending":
        print("The Pod is waiting for scheduling or resources.")
    elif pod_status == "CrashLoopBackOff":
        print("The Pod is restarting due to an issue. Further investigation is needed.")
    else:
        print("The Pod status needs further confirmation.")

### Explanation

- `==` checks whether two values are equal.
- This is useful for comparing a variable with a fixed status value.
- Multiple statuses can be checked using `elif` in sequence.

### Operational Significance

This pattern is commonly used to:

- Check Pod status
- Monitor node health
- Assess service availability
- Determine the success or failure of tasks

- `replace()`
- `split()`

### 4）Review of Previous Knowledge Points

- `in`
- `strip()`
- `endswith()`

---

## Fifteen, Today's Practical Suggestions

Today, don't just read; you must actually write the code yourself.

It is recommended that you perform the following operations in Ubuntu:

    mkdir -p ~/python-study/day2
    cd ~/python-study/day2

Then save today's examples in these files respectively:

    pod_status_check.py
    log_level_check.py
    cpu_usage_check.py
    filename_prefix_suffix_check.py
    log_split.py
    msg_replace.py

Write each file by yourself and then execute it:

    python3 文件名.py

Don't just copy them; that doesn't mean you have learned. At this stage, the most important thing is:

**Understanding what you see + Being able to write it out**

---

## Sixteen, Day2 Homework

You can practice the following homework directly.

### Homework 1: pod_status_check.py

Requirement: Output different results based on the Pod status.

    pod_status = "Pending"

    if pod_status == "Running":
        print("Pod is running normally")
    elif pod_status == "Pending":
        print("Pod is waiting for scheduling")
    elif pod_status == "CrashLoopBackOff":
        print("Pod is restarting abnormally")
    else:
        print("Pod status is unknown")

---

### Homework 2: log_level_check.py

Requirement: Determine the log level based on the log content.

    log_line = "ERROR etcd connection timeout"

    if "ERROR" in log_line:
        print("Error log")
    elif "WARNING" in log_line:
        print("Warning log")
    else:
        print("Regular log")

---

### Homework 3: log_level_upper_check.py

Requirement: Convert the entire string to uppercase before making the judgment.

    log_line = "warning disk usage high"

    if "WARNING" in log_line.upper():
        print("WARNING detected")
    else:
        print("WARNING not detected")

---

### Homework 4: filename_prefix_suffix_check.py

Requirement: Determine both the prefix and suffix of a file name.

    filename = "syslog.log"

    if filename.startswith("sys"):
        print("The file starts with 'sys'")
    if filename.endswith(".log"):
        print("This is a log file")

---

### Homework 5: log_split.py

Requirement: Split the log content into a list.

    log_line = "ERROR kube-apiserver connection refused"
    parts = log_line.split()

    print(parts)

---

### Homework 6: cpu_usage_check.py

Requirement: Output different levels of information based on CPU usage.

    cpu_usage = 85

    if cpu_usage >= 90:
        print("CPU usage is too high")
    elif cpu_usage >= 70:
        print("CPU usage is relatively high")
    else:
        print("CPU usage is normal")

---

### Homework 7: msg_replace.py

Requirement: Replace specified content in a string.

    msg = "node is not ready"
    new_msg = msg.replace("not ready", "ready")

    print(new_msg)

---

## Seventeen, Today's Summary

The core of Day2 is not about "learning many functions" but about beginning to develop a true script-based mindset:

- Obtain data
- Clean the data first
- Make judgments
- Finally, output the results

This is the basic structure of many operational scripts.

Although today's content still belongs to the basics of Python, it has already started to approach real operational work.

---

## Eighteen, What You Should Achieve After Today's Learning

After completing Day2, you should be able to:

- Write multi-branch conditionals
- Identify simple log levels
- Determine the status of Pods or services
- Grade CPU usage based on thresholds
- Handle cases where uppercase and lowercase need to be unified
- Perform basic string cleaning and splitting operations

This means you have moved beyond just knowing how to use `print()`. You have officially entered the stage of "writing simple operational judgment scripts".

---

## Nineteen, What Will Day3 Cover Tomorrow

If you practice Day2 thoroughly, Day3 will be perfect for continuing with:

- Logical operators: `and`, `or`, `not`
- More complex condition combinations
- Simultaneous evaluation of multiple conditions
- Judgment logic that is closer to real operational rules

In other words, it will progress from "single-condition judgments" to "combined condition judgments".

---

## Twenty, External Links

### Python Official Documentation

- Python Tutorial: Control Flow Tools  
  https://docs.python.org/3/tutorial/controlflow.html

- Python Standard Types: String Methods  
  https://docs.python.org/3/library/stdtypes.html#string-methods

### Beginner References

- Runoob Tutorial: Python3 Conditionals  
 