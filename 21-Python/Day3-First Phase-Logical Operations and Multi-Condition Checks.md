# Day3 - First Stage - Logical Operations and Multi-Condition Judgments

#Python #PythonLearning #TransportDevelopment #LogicalOperations #ConditionalJudgement #and #or #not #LogAnalysis #Kubernetes #Linux #Obsidian

## Today's Focus

What you learned on Day1:

- Variables
- Strings
- print()
- f-string
- `in`
- `strip()`
- `endswith()`

What you learned on Day2:

- `if / elif / else`
- Multi-branch judgment
- `upper()` / `lower()`
- `startswith()`
- `replace()`
- `split()`

Day3 will continue forward into a very critical stage:

**Combining multiple conditions for judgment**

This step is important because in real operations scenarios, it's rare to have only one condition.

Often you need to judge like this:

- If CPU is high and memory is also high, it indicates greater risk
- If Pod is not Running or node is not Ready, troubleshooting is needed
- If logs don't contain ERROR, it means no errors found temporarily
- If port is 80 or 443, it indicates a Web service
- If file is not `.log`, it won't be processed

So the core of Day3 is upgrading from "single-condition judgment" to "combined-condition judgment".

---

## Today's Objectives

After completing Day3, you should be able to:

1. Understand `and`
2. Understand `or`
3. Understand `not`
4. Write multiple conditions into one `if`
5. Write scripts closer to real operations logic
6. Establish thinking of "judging with multiple conditions together"

---

## What You'll Learn Today

Day3 content includes:

1. `and`
2. `or`
3. `not`
4. Multi-condition combination judgment
5. Combined condition scenarios in operations
6. Day3 Practical Exercise

---

## Why Day3 is Important

On Days 1 and 2, you could already make these judgments:

- If Pod is Running, output normal
- If logs contain ERROR, output abnormal
- If CPU > 80, output alert

But real scenarios are often more complex.

For example:

- CPU > 80 and memory > 80, then high-priority alert is needed
- Port is 80 or 443, identify as Web service
- If logs don't contain ERROR, consider temporarily normal
- If service name starts with kube and file ends with .log, process it

So Day3 essentially trains you to:

**Don't look at one condition only, learn to combine multiple conditions.**

---

## Logical Operators Overview

Python's 3 most commonly used logical operators:

| Operator | Meaning |
|---|---|
| `and` | And |
| `or` | Or |
| `not` | Not |

You can initially understand them as:

- `and`: Both conditions must be true for the whole to be true
- `or`: As long as one condition is true, the whole is true
- `not`: Invert the original judgment result

---

## `and`: And

### Purpose

`and` represents multiple conditions that must all be satisfied.

### Example Code

    cpu_usage = 85
    memory_usage = 90

    if cpu_usage > 80 and memory_usage > 80:
        print("CPU and memory are both high, need close attention")

### Code Explanation

There are two conditions:

- `cpu_usage > 80`
- `memory_usage > 80`

Only when both conditions are true will `print()` execute.

### Operations Significance

This is suitable for judging "comprehensive risk":

- CPU high
- Memory also high

When both appear together, it's usually more dangerous than a single condition.

---

## `or`: Or

### Purpose

`or` represents multiple conditions where any one being satisfied is enough.

### Example Code

    port = 443

    if port == 80 or port == 443:
        print("This is a common Web service port")

### Code Explanation

The meaning here is:

- If port is 80
- Or if port is 443
- Both are considered common Web service ports

### Operations Significance

This is suitable for:

- Judging multiple valid ports
- Judging multiple states
- Judging multiple keywords

---

## `not`: Not

### Purpose

`not` is used to invert the judgment result.

### Example Code

    pod_status = "Pending"

    if not pod_status == "Running":
        print("Pod status is abnormal")

### Code Explanation

The original condition was:

    pod_status == "Running"

After adding `not`, it becomes:

    pod_status is not Running

### More Understandable Writing

You can also write it as:

    pod_status = "Pending"

    if pod_status != "Running":
        print("Pod status is abnormal")

For your current stage, many scenarios have:

- `not A`
- And
- `A != Value`

Effectively similar

But you still need to recognize `not`, as it will be commonly seen later.

---

## `and` in Real Operations Scenarios

### Scenario 1: CPU and memory both high

    cpu_usage = 88
    memory_usage = 91

    if cpu_usage > 80 and memory_usage > 80:
        print("System resource pressure is significant")

### Scenario 2: Filename prefix and suffix both meet requirements

    filename = "syslog.log"

    if filename.startswith("sys") and filename.endswith(".log"):
        print("This is a system log file to process")

### Scenario 3: Service status and port both meet expectations

    service_status = "active"
    port = 443

if service_status == "active" and port == 443:
    print("HTTPS service is running normally")

---

## VIII. `or` Real-World DevOps Scenarios

### Scenario 1: Identify Web Service Port

    port = 80

    if port == 80 or port == 443:
        print("This is a Web service")

### Scenario 2: Determine if Pod is in Abnormal State

    pod_status = "Pending"

    if pod_status == "Pending" or pod_status == "CrashLoopBackOff":
        print("Pod status is abnormal, requires attention")

### Scenario 3: Identify Error or Warning Logs

    log_line = "WARNING node memory high"

    if "ERROR" in log_line or "WARNING" in log_line:
        print("This is a log that requires attention")

---

## IX. `not` Real-World DevOps Scenarios

### Scenario 1: Pod is Not Running

    pod_status = "CrashLoopBackOff"

    if not pod_status == "Running":
        print("Pod is not normal")

### Scenario 2: File is Not a Log File

    filename = "nginx.conf"

    if not filename.endswith(".log"):
        print("This is not a log file")

### Scenario 3: Log Does Not Contain ERROR

    log_line = "INFO service started"

    if "ERROR" not in log_line:
        print("No error logs detected currently")

### Here you'll also learn a more common writing style

Besides:

    if not "ERROR" in log_line:

A more common and better-understood way is:

    if "ERROR" not in log_line:

This writing style will be commonly seen in the future.

---

## X. Multi-Condition Combination Judgment

Day3's focus isn't just learning `and / or / not`, but learning to put them into real-world judgments.

### Example Code

    cpu_usage = 85
    memory_usage = 70

    if cpu_usage > 80 and memory_usage > 80:
        print("CPU and memory are both too high")
    elif cpu_usage > 80 or memory_usage > 80:
        print("One resource usage is high")
    else:
        print("System resources are normal")

### Code Explanation

The logic of this code is:

1. If both CPU and memory are high, output the most severe result
2. Otherwise, if either one is high, also output a warning
3. Otherwise, output normal

### DevOps Significance

This is already very close to real monitoring judgment logic:

- High risk
- Medium risk
- Normal

---

## XI. String Judgment and Logical Operation Combination

### Example Code

    log_line = "warning kubelet disk pressure"

    if "ERROR" in log_line.upper() or "WARNING" in log_line.upper():
        print("Abnormal level log detected")

### Code Explanation

Here we combine two pieces of knowledge:

- `upper()`: Standardize case
- `or`: Check for ERROR or WARNING

### DevOps Significance

Log text may not have consistent case, so you should first standardize it before keyword recognition for more stability.

---

## XII. Prefix/Suffix and Logical Combination

### Example Code

    filename = "sys-kube.log"

    if filename.startswith("sys") and filename.endswith(".log"):
        print("This is a log file that meets the rules")

### Why This Writing Style is Common

Because in DevOps you often need to judge multiple features at the same time:

- Is the prefix correct?
- Is the suffix correct?
- Is the naming rule correct?
- Is the type correct?

---

## XIII. Day3 Core Thinking You Must Master

From Day1 to Day3, the thinking process is gradually upgrading:

### Day1
Knows how to store values and output.

### Day2
Can make judgments based on single conditions.

### Day3
Can make judgments based on multiple conditions combined.

This means you're one step closer to "real scripts."

Many scripts are essentially these three steps:

1. Retrieve data
2. Combine conditions for judgment
3. Output results or execute actions

---

## XIV. Today's Hands-On Suggestions

Today, I recommend creating these files in Ubuntu:

    mkdir -p ~/python-study/day3
    cd ~/python-study/day3

Then practice these scripts:

    cpu_memory_check.py
    port_check.py
    pod_status_check.py
    log_level_check.py
    filename_rule_check.py
    mixed_condition_check.py

Execution method:

    python3 filename.py

Today's focus is still not on copying, but:

**Type it yourself, run it, and understand the results.**

---

## XV. Day3 Practice Exercises (These can be typed first to understand the logic)

### 1) cpu_memory_check.py

Requirement: Judge CPU and memory simultaneously.

    cpu_usage = 85
    memory_usage = 90

    if cpu_usage > 80 and memory_usage > 80:
        print("CPU and memory are both too high")
    else:
        print("Resource pressure is not severe for now")

---

### 2) port_check.py

Requirement: Judge if it's a common Web service port.

    port = 443

    if port == 80 or port == 443:
        print("This is a Web service port")
    else:
        print("This is not a common Web service port")

---

### 3) pod_status_check.py

Requirement: Judge if Pod is not Running.

    pod_status = "Pending" /think

if pod_status != "Running":
    print("Pod status is abnormal")
else:
    print("Pod status is normal")

---

### 4I'm not sure.log_level_check.py

Requirement: Identify both ERROR and WARNING.

    log_line = "warning node disk pressure"

    if "ERROR" in log_line.upper() or "WARNING" in log_line.upper():
        print("Abnormal log found")
    else:
        print("Log is normal")

---

### 5I'm not sure.filename_rule_check.py

Requirement: Judge both filename prefix and suffix.

    filename = "syslog.log"

    if filename.startswith("sys") and filename.endswith(".log"):
        print("This is the target log file")
    else:
        print("Not the target log file")

---

### 6I'm not sure.mixed_condition_check.py

Requirement: Write a more complete combined judgment.

    cpu_usage = 88
    memory_usage = 75

    if cpu_usage > 80 and memory_usage > 80:
        print("High risk: Both CPU and memory are high")
    elif cpu_usage > 80 or memory_usage > 80:
        print("Medium risk: One resource is high")
    else:
        print("Normal")

---

## SixteenI don't know.Day3 Homework Reference Answers

The following provides reference answers for Day3 homework to reinforce the following knowledge points:

- `not`
- `in`
- `not in`
- `startswith()`
- `endswith()`
- Combined multi-condition judgment

---

### Homework 1: Kubernetes Service Log Identification

#### Question Requirements

- If the service name starts with `kube`
- And the filename ends with `.log`
- Output: `This is... Kubernetes Service Log`

#### Reference Answer

    service_name = "kube-apiserver"
    file_name = "kube-apiserver.log"

    if service_name.startswith("kube") and file_name.endswith(".log"):
        print("This is a Kubernetes service log")

---

### Homework 2: Remote Management Port Identification

#### Question Requirements

- If the port is 22 or 3389
- Output: `This is the remote management end. mouth`

#### Reference Answer

    port = 22

    if port == 22 or port == 3389:
        print("This is a remote management port")

---

### Homework 3: Pod Abnormal Status Judgment

#### Question Requirements

- If the status is not Running
- Output: `Pod Unusual`

#### Reference Answer

    pod_status = "CrashLoopBackOff"

    if pod_status != "Running":
        print("Pod is abnormal")

---

### Homework 4: Log Does Not Contain ERROR

#### Question Requirements

- If the log does not contain ERROR
- Output: `No error log currently available`

#### Reference Answer

    log_line = "info nginx service started"

    if "ERROR" not in log_line.upper():
        print("No error logs currently")

---

### Homework 5: Resource Alert Level Judgment

#### Question Requirements

- If CPU and memory are both greater than 90, output: `Serious alarm.`
- If one of them is greater than 90, output: `A common alarm.`
- Otherwise, output: `Normal`

#### Reference Answer

    cpu_usage = 95
    memory_usage = 96

    if cpu_usage > 90 and memory_usage > 90:
        print("Severe alert")
    elif cpu_usage > 90 or memory_usage > 90:
        print("General alert")
    else:
        print("Normal")

---

## SeventeenI don't know.Day3 Common Error Examples

This section is very important.

The difficulty of Day3 is usually not in the `if` structure itself, but in easily mixing similar string judgment methods.

---

### 1I'm not sure.Mixing `in` and `startswith()`

#### Error Example

    service_name = "kube-apiserver"
    file_name = "kube-apiserver.log"

    if service_name in service_name.startswith("kube") and file_name.endswith(".log"):
        print("This is a Kubernetes service log")

#### Why it's wrong

Because:

    service_name.startswith("kube")

Returns a boolean value:

    True

or:

    False

It is not a string, and cannot be combined with `in` like this.

#### Correct Example

    service_name = "kube-apiserver"
    file_name = "kube-apiserver.log"

    if service_name.startswith("kube") and file_name.endswith(".log"):
        print("This is a Kubernetes service log")

#### Core Conclusion

- `in`: Check for inclusion
- `startswith()`: Check if starts with certain content
- These are not the same judgment methods

---

### 2I'm not sure.Misusing "Inclusion Judgment" as "Starts With Judgment"

#### Example Comparison

##### Inclusion Judgment

    service_name = "my-kube-apiserver"

if "kube" in service_name:
    print("contains kube")

##### Startswith Check

    service_name = "kube-apiserver"

    if service_name.startswith("kube"):
        print("starts with kube")

#### Explanation

- `"kube" in service_name` indicates "contains kube"
- `service_name.startswith("kube")` indicates "starts with kube"

These two results may look similar in some scenarios, but their semantics are different.

---

### 3) Not Paying Attention to What the Expression Returns

#### Example

    file_name = "kube-apiserver.log"

    print(file_name.startswith("kube"))
    print(file_name.endswith(".log"))

#### Output

    True
    True

#### Explanation

Many beginners use these methods without realizing they ultimately return:

- `True`
- `False`

Understanding this is important because only by knowing the return value of an expression can you avoid incorrectly combining different types of expressions.

#### Common Return Value Examples

    service_name.startswith("kube")    # True / False
    file_name.endswith(".log")         # True / False
    "ERROR" in log_line                # True / False
    "ERROR" not in log_line            # True / False

---

### 4) Inconsistent Capitalization Causes Unstable Keyword Checks

#### Unstable Writing Style

    log_line = "warning nginx cpu high"

    if "WARNING" in log_line:
        print("Found alert log")

This code won't trigger in the current example because the original string is lowercase `warning`.

#### More Stable Writing Style

    log_line = "warning nginx cpu high"

    if "WARNING" in log_line.upper():
        print("Found alert log")

#### Core Conclusion

When checking for log keywords, first normalize the case and then compare - this is more reliable.

---

### 5) Writing "Not Running" Too Complexly

#### Less Readable Style

    pod_status = "CrashLoopBackOff"

    if not pod_status == "Running":
        print("Pod abnormal")

#### Recommended Style

    pod_status = "CrashLoopBackOff"

    if pod_status != "Running":
        print("Pod abnormal")

#### Explanation

Both approaches have similar logic, but for beginners:

    !=

is typically more direct and easier to understand.

---

## Eighteen, Day3 Must Master Differentiation Points

By Day3, the most confusing aspect isn't syntax structure, but judgment methods.

The following groups must be thoroughly distinguished.

---

### 1) Contains Check

    "kube" in service_name

Meaning:

**Does the string contain certain content**

---

### 2) Startswith Check

    service_name.startswith("kube")

Meaning:

**Does the string start with certain content**

---

### 3) Endswith Check

    file_name.endswith(".log")

Meaning:

**Does the string end with certain content**

---

### 4) Not Contains Check

    "ERROR" not in log_line.upper()

Meaning:

**Does the string not contain certain content**

---

### Shortest Comparison Example

    service_name = "kube-apiserver"
    file_name = "kube-apiserver.log"
    log_line = "info nginx service started"

    print("kube" in service_name)
    print(service_name.startswith("kube"))
    print(file_name.endswith(".log"))
    print("ERROR" not in log_line.upper())

---

## Nineteen, Day3 Learning Summary

The core of Day3 isn't just learning `and / or / not` words,  
but entering the stage of "combined condition judgment".

This means you can now handle logic closer to realTransport scenarios, such as:

- Judging if a service name follows rules
- Judging if a filename meets processing conditions
- Judging if a Pod is normal
- Judging if error keywords exist in logs
- Judging if resources enter different alert levels

---

## Twenty, What Abilities You Gain After Day3

By now, you have the following basic abilities:

### 1) Can Write Simple Condition Judgments

For example:

- Service status judgment
- Log keyword judgment
- Pod status judgment

### 2) Can Perform Combined Condition Judgments

For example:

- Judging start and end at the same time
- Judging multiple resource metrics at the same time
- Judging multiple rules at the same time

### 3) Can Perform Basic String Rule Judgments

For example:

- Whether it contains keywords
- Whether it starts with certain content
- Whether it ends with certain content

### 4) Begin to Have Basic DevOps Scripting Thinking

Many basic DevOps scripts follow this model:

1. Get data
2. Combine judgments
3. Output results or execute actions

---

## Twenty-One, What Day4 Will Cover

If you master Day3 well, Day4 is suitable for entering:

- List `list`
- `for` Loop
- Processing multiple objects at once
- Batch processing logs, Pods, ports, filenames

This means upgrading from:

- Judging one value at a time

To:

- Processing a group of values at once

This step makes Python more like a real automation script.

---

## Twenty-Two, External Links

### Python Official Documentation

- Python Tutorial: Control Flow Tools  
  https://docs.python.org/3/tutorial/controlflow.html

- Python Standard Types  
  https://docs.python.org/3/library/stdtypes.html

### Beginner References

- CSDN Tutorial: Python3 Conditional Control  
  https://www.runoob.com/python3/python3-if-statements.html

- CSDN Tutorial: Python3 Operators  
  https://www.runoob.com/python3/python3-basic-operators.html

---

## Twenty-Three, Today's One-Sentence Summary /think

**The essence of Day3 is to transition Python scripts from "single-condition judgment" to "multi-condition combination judgment" and begin mastering string rule judgments.**