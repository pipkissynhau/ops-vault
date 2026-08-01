# Day2 - Phase 1 - Advanced Conditional Judgments and String Processing

#Python #PythonLearning #TransportDevelopment #PythonFoundation #ConditionalJudgement #StringProcessing #LogAnalysis #Kubernetes #Linux #Obsidian

## Today's Focus

Day2 marks the beginning of transitioning from "being able to write Python code" to "giving code the ability to make judgments".

On Day1, you've already encountered:

- Variables
- Strings
- print()
- f-string
- in
- strip()
- endswith()
- Basic if

Day2 will continue to advance, with the focus not on memorizing syntax, but on learning:

- Output different results based on different situations
- Further process strings
- Write simpleTransport judgment logic as Python scripts

This step is crucial because subsequent tasks like log analysis, status inspection, alert judgment, and automation processing all fundamentally rely on "first judge, then process".

---

## Today's Objectives

After completing Day2, you should be able to:

1. Write `if / elif / else`
2. Output different results based on different states
3. Judge whether keywords exist in logs
4. Handle inconsistent capitalization issues
5. Use string methods for simple cleaning and splitting
6. Write several practicalTransport scripts

---

## What You'll Learn Today

Day2 content includes:

1. `if / elif / else`
2. Multi-condition judgment
3. Additional comparison operators
4. `lower()` and `upper()`
5. `startswith()`
6. `replace()`
7. `split()`
8.Transport scenario practice

---

## WhyTransport Must Learn Conditional Judgments

InTransport scenarios, most scripts aren't "mindless execution", but rather:

- If logs contain ERROR, trigger an alert
- If Pod is not Running, output anomaly
- If CPU exceeds threshold, warn of high risk
- If file doesn't end with `.log`, skip processing
- If status doesn't match expectation, enter troubleshooting

Thus, Python judgment statements essentially write out your mentalTransport judgment logic.

---

## Conditional Judgment Structure

### 1) Basic Structure

Python's multi-branch judgment structure is as follows:

    if condition1:
        execute when condition1 is true
    elif condition2:
        execute when condition2 is true
    else:
        execute when none of the previous conditions are true

### 2) How to Understand

- `if`: First judge the first condition
- `elif`: If the previous condition is not true, continue judging
- `else`: Execute when none of the previous conditions are true

### 3) Notes

- Python uses indentation to represent code blocks
- Indentation for the same level must be consistent
- `if`I don't know.`elif`I don't know.`else` are a group of structures, only one branch will be executed

---

## Log Level Judgment

This is the most typicalTransport scenario.

### Example Code

    log_line = "ERROR kube-apiserver connection refused"

    if "ERROR" in log_line:
        print("This is an error log, needs priority handling")
    elif "WARNING" in log_line:
        print("This is an alert log, needs attention")
    else:
        print("This is a normal log")

### Code Explanation

- `"ERROR" in log_line`: Judge whether string contains `ERROR`
- If the first condition is true, the subsequent `elif` and `else` won't execute
- This pattern is very suitable for initial log classification

##Luck# Significance

You can later extend this logic to:

- Match more log levels
- Write error logs to files
- Count different level logs
- Trigger alert notifications

---

## Service Status Judgment

This approach is suitable for scenarios with fixed status values, like Pod status, service status, or task status.

### Example Code

    pod_status = "CrashLoopBackOff"

    if pod_status == "Running":
        print("Pod is running normally")
    elif pod_status == "Pending":
        print("Pod is waiting for scheduling or resources")
    elif pod_status == "CrashLoopBackOff":
        print("Pod has abnormal restart, needs troubleshooting")
    else:
        print("Pod status needs further confirmation")

### Code Explanation

- `==` represents "is equal to"
- Suitable for comparing a variable with fixed status values
- Multiple statuses can be judged sequentially with `elif`

##Luck# Significance

This pattern is commonly used for:

- Judging Pod status
- Judging node status
- Judging service health status
- Judging whether a task execution was successful

---

## Multi-condition Judgment and Threshold Grading

InTransport, threshold grading is often encountered, such as:

- Normal
- Slightly high
- Too high

### Example Code

    cpu_usage = 92

    if cpu_usage >= 90:
        print("CPU usage is too high")
    elif cpu_usage >= 70:
        print("CPU usage is slightly high")
    else:
        print("CPU usage is normal")

### Code Explanation

- `>=` represents "greater than or equal to"
- When judging ranges, write stricter conditions first
- Otherwise, the logic may be intercepted by earlier conditions

##Luck# Significance

This approach applies to:

- CPU usage grading
- Memory usage grading
- Disk usage grading
- Alert level classification

---

## String Case Handling

In actualTransport scenarios, logs, command outputs, and user inputs often have inconsistent capitalization.

Direct judgment may lead to misjudgment.

---

## VIII. `lower()`: Convert to Lowercase

### Purpose

Convert all characters in a string to lowercase.

### Example Code

    log_level = "Warning"
    print(log_level.lower())

Output result:

    warning

### Use Cases

- Standardize command output content
- Ignore case differences
- Standardize before string comparison

---

## IX. `upper()`: Convert to Uppercase

### Purpose

Convert all characters in a string to uppercase.

### Example Code

    log_level = "warning"
    print(log_level.upper())

Output result:

    WARNING

##Luck# Scenario Example

    log_line = "warning kubelet disk pressure"

if "WARNING" in log_line.upper():
    print("Detected warning log")

### Why This Approach

Because the original log might be:

- warning
- Warning
- WARNING

If you first convert to uppercase and then compare, you won't fail due to case differences.

---

## TenI don't know.`startswith()`: Check if Starts with Specific Content

### Purpose

Check if a string starts with specified content.

### Example Code

    filename = "syslog.log"

    if filename.startswith("sys"):
        print("This is a file starting with 'sys'")

### Common Use Cases in Operations

- Check filename prefix
- Check hostname prefix
- Check service naming prefix
- Check if certain log content starts with fixed format

---

## ElevenI don't know.`replace()`: Replace String Content

### Purpose

Replace part of a string with new content.

### Example Code

    msg = "node is not ready"
    new_msg = msg.replace("not ready", "ready")

    print(new_msg)

Output result:

    node is ready

### Common Use Cases in Operations

- Clean log content
- Replace useless characters in command output
- Correct fixed fragments in text
- Do simple format conversion

### Note

`replace()` does not directly modify the original string, but returns a new string, so you usually need to receive the result.

---

## TwelveI don't know.`split()`: Split String

### Purpose

Split a string into multiple parts according to a delimiter, resulting in a list.

### Example Code

    log_line = "ERROR kubelet cpu high"
    result = log_line.split()

    print(result)

Output result:

    ['ERROR', 'kubelet', 'cpu', 'high']

### Common Use Cases in Operations

- Split log fields
- Analyze command output
- Extract multiple parts from text
- Prepare for learning about lists

### Just Understand This for Now

You don't need to deeply understand what "list" is. Just remember:

`split()` can split a whole string into multiple parts, making it convenient for subsequent value retrieval.

---

## ThirteenI don't know.Comprehensive Example: Log Content Recognition

This example combines Day1 and Day2 knowledge points.

### Example Code

    log_line = " warning kubelet disk pressure "

    clean_line = log_line.strip().upper()

    if "ERROR" in clean_line:
        print("Detected error log")
    elif "WARNING" in clean_line:
        print("Detected warning log")
    else:
        print("Detected normal log")

### Knowledge Points Used Here

- `strip()`: Remove leading/trailing whitespace
- `upper()`: Convert to uppercase
- `if / elif / else`: Do conditional branching
- `in`: Check if keyword exists

### Operational Significance

This is already very close to the basic model of log analysis scripts:

1. Clean raw content first
2. Do unified format processing
3. Branch judgment based on keywords
4. Output different results finally

After you learn about loops, lists, and file reading later, this pattern will become increasingly common.

---

## FourteenI don't know.Key Knowledge to Master Today

Today's most core knowledge is the following:

### 1) Conditional Judgment Structure

- `if`
- `elif`
- `else`

### 2) Comparison Operators

- `==`
- `>=`

### 3) String Processing

- `lower()`
- `upper()`
- `startswith()`
- `replace()`
- `split()`

### 4) Review of Previous Knowledge

- `in`
- `strip()`
- `endswith()`

---

## FifteenI don't know.Today's Hands-on Practice Suggestions

Don't just watch today - type it yourself.

Suggested operations in Ubuntu:

    mkdir -p ~/python-study/day2
    cd ~/python-study/day2

Then save today's examples into these files:

    pod_status_check.py
    log_level_check.py
    cpu_usage_check.py
    filename_prefix_suffix_check.py
    log_split.py
    msg_replace.py

Type each file yourself, then execute:

    python3 filename.py

Don't just copy and call it learned. At this stage, what's most important is:

**Understand with eyes + type it yourself**

---

## SixteenI don't know.Day2 Homework

The following homework you can practice directly.

### Homework 1: pod_status_check.py

Requirement: Output different results based on Pod status.

    pod_status = "Pending"

    if pod_status == "Running":
        print("Pod is running normally")
    elif pod_status == "Pending":
        print("Pod is waiting for scheduling")
    elif pod_status == "CrashLoopBackOff":
        print("Pod has abnormal restart")
    else:
        print("Pod status is unknown")

---

### Homework 2: log_level_check.py

Requirement: Judge log level based on log content.

    log_line = "ERROR etcd connection timeout"

    if "ERROR" in log_line:
        print("Error log")
    elif "WARNING" in log_line:
        print("Warning log")
    else:
        print("Normal log")

---

### Homework 3: log_level_upper_check.py

Requirement: Judge after converting to uppercase.

    log_line = "warning disk usage high"

if "WARNING" in log_line.upper():
    print("Detected WARNING")
else:
    print("No WARNING detected")

---

### Exercise 4: filename_prefix_suffix_check.py

Requirement: Judge both filename prefix and suffix simultaneously.

    filename = "syslog.log"

    if filename.startswith("sys"):
        print("File starts with 'sys'")

    if filename.endswith(".log"):
        print("This is a log file")

---

### Exercise 5: log_split.py

Requirement: Split log content into a list.

    log_line = "ERROR kube-apiserver connection refused"
    parts = log_line.split()

    print(parts)

---

### Exercise 6: cpu_usage_check.py

Requirement: Output different levels of information based on CPU usage.

    cpu_usage = 85

    if cpu_usage >= 90:
        print("CPU usage is too high")
    elif cpu_usage >= 70:
        print("CPU usage is elevated")
    else:
        print("CPU usage is normal")

---

### Exercise 7: msg_replace.py

Requirement: Replace specified content in a string.

    msg = "node is not ready"
    new_msg = msg.replace("not ready", "ready")

    print(new_msg)

---

## Seventeen. Daily Summary

The core of Day2 is not about "learning many functions," but about starting to develop a true scripting mindset:

- Obtain data
- Clean it first
- Make judgments
- Output results finally

This is the basic skeleton of manyTransport scripts.

Although today's content still belongs to Python basics, it has already started to approach actualTransport work.

---

## Eighteen. What You Should Achieve After Completing Day2

After completing Day2, you should be able to:

- Write multi-branch conditions
- Identify simple log levels
- Judge Pod or service status
- Grade CPU usage by thresholds
- Handle case normalization
- Perform basic string cleaning and splitting

This indicates you have moved beyond the stage of just `print()` and are officially entering the stage of "writing simpleTransport judgment scripts."

---

## Nineteen. What Day3 Will Cover

If Day2 is well-practiced, Day3 is suitable for continuing into:

- Logical operators: `and`, `or`, `not`
- More complex condition combinations
- Multiple conditions judged simultaneously
- Judgment logic closer to realTransport rules

This means moving from "single-condition judgment" to "combined condition judgment."

---

## Twenty. External Links

### Python Official Documentation

- Python Tutorial: Control Flow Tools  
  https://docs.python.org/3/tutorial/controlflow.html

- Python Standard Types: String Methods  
  https://docs.python.org/3/library/stdtypes.html#string-methods

### Beginner References

- Runoob Tutorial: Python3 Conditional Control  
  https://www.runoob.com/python3/python3-if-statements.html

- Runoob Tutorial: Python3 Strings  
  https://www.runoob.com/python3/python3-string.html

---

## Twenty-One. Today's One-Sentence Summary

**The essence of Day2 is to let Python scripts learn to "perform different actions based on different situations."**

This step marks your true beginning from "being able to write code" to "being able to writeTransport scripts."