# Day3 - Phase 1 - Logical Operations and Multi-Condition Judgments

#Python #Python Learning #Ops Development #Logical Operations #Condition Judgments #and #or #not #Log Analysis #Kubernetes #Linux #Obsidian

## Today's Focus

On Day1, you learned:

- Variables
- Strings
- print()
- f-string
- `in`
- `strip()`
- `endswith()`

On Day2, you learned:

- `if / elif / else`
- Multi-branch Judgments
- `upper()` / `lower()`
- `startswith()`
- `replace()`
- `split()`

Today, you will move on to a very crucial stage:

**Combining Multiple Conditions for Judgment**

This step is essential because in real Ops scenarios, it's rare to have only one condition. Often, you need to evaluate things like this:

- If both CPU and memory usage are high, it indicates a greater risk.
- If a Pod is not Running or the node is not Ready, further investigation is needed.
- If there are no ERROR messages in the logs, it means no errors have been detected for now.
- If a port is 80 or 443, it indicates a Web service.
- If a file doesn't end with `.log`, it should not be processed.

Therefore, the core of Day3 is to upgrade from “single-condition judgments” to “combined condition judgments”.

---

## Today's Goals

After completing Day3, you should be able to:

1. Understand `and`.
2. Understand `or`.
3. Understand `not`.
4. Write multiple conditions within an `if` statement.
5. Create small scripts that reflect real Ops logic.
6. Develop the mindset of evaluating multiple conditions together.

---

## I. What to Learn Today

Day3 covers the following topics:

1. `and`
2. `or`
3. `not`
4. Combined condition judgments.
5. Real-world Ops scenarios involving combined conditions.
6. Day3 practice exercises.

---

## II. Why Day3 Is Important

By now, you can already make basic judgments like:

- If a Pod is Running, display “Normal”.
- If there are ERROR messages in the logs, display “Abnormal”.
- If CPU usage exceeds 80%, trigger an alarm.

However, real-world situations are often more complex. For example:

- An alarm should be triggered only if both CPU and memory usage exceed 80%.
- A port must be 80 or 443 to identify it as a Web service.
- Only if there are no ERROR messages in the logs can it be considered normal for now.
- A file should be processed only if its name starts with “kube” and ends with“.log”.

Therefore, Day3 is essentially training you to:

**Avoid focusing on just one condition; learn to combine multiple conditions together.**

---

## III. Overview of Logical Operators

The three most commonly used logical operators in Python are:

| Operator | Meaning |
|---|---|
| `and` | Both conditions must be true. |
| `or` | At least one condition must be true. |
| `not` | The opposite of the original judgment result. |

You can think of it this way:

- `and`: The whole statement is true only if both conditions are true.
- `or`: The whole statement is true if at least one condition is true.
- `not`: It inverts the original judgment result.

---

## IV. `and`: Both Conditions Must Be True

### Function

`and` means that all multiple conditions must be met simultaneously.

### Example Code

    cpu_usage = 85
    memory_usage = 90

    if cpu_usage > 80 and memory_usage > 80:
        print("Both CPU and memory usage are too high, requiring special attention.")

### Explanation

Here, we have two conditions:

- `cpu_usage > 80`
- `memory_usage > 80`

Only when both conditions are true will the `print()` statement be executed.

### Ops Implication

This is very useful for assessing “comprehensive risks”:

- High CPU usage
- High memory usage

If both occur, it usually indicates a more serious issue than either alone.

---

## V. `or`: At Least One Condition Must Be True

### Function

`or` means that as long as at least one of the multiple conditions is true, the whole statement is true.

### Example Code

    port = 443

    if port == 80 or port == 443:
        print("This is a common Web service port.")

### Explanation

Here, it means that:

- If the port is 80
- Or if the port is 443
- Either situation indicates a common Web service port.

### Ops Implication

This is particularly useful for:

- Ident### Why This Type of Writing Is Common

In operations and maintenance, it's often necessary to evaluate not just one factor but multiple characteristics simultaneously:

- Whether the prefix is correct.
- Whether the suffix is correct.
- Whether the naming rule is followed.
- Whether the type is appropriate.

---

## Thirteen, Day 3: Core Thinking You Must Master

From Day 1 to Day 3, your approach gradually advances:

### Day 1
You can store values and output them.

### Day 2
You can make judgments based on a single condition.

### Day 3
You can make judgments by combining multiple conditions.

This means you're one step closer to creating "real scripts." Many scripts essentially follow these three steps:

1. Gather data.
2. Apply combined conditional checks.
3. Output results or perform actions.

---

## Fourteen, Today's Practical Suggestions

Today, I recommend you create the following files in Ubuntu:

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

    python3 file_name.py

Today, the focus is not on copying but on:

**Writing the code yourself, running it, and understanding the results.**

---

## Fifteen, Day 3 Example Exercises (You Can Copy These First to Understand the Logic)

### 1) cpu_memory_check.py

Requirement: Check both CPU and memory usage simultaneously.

    cpu_usage = 85
    memory_usage = 90

    if cpu_usage > 80 and memory_usage > 80:
        print("Both CPU and memory are high.")
    else:
        print("Resource pressure is not severe at present.")

---

### 2) port_check.py

Requirement: Determine whether the port is a common Web port.

    port = 443

    if port == 80 or port == 443:
        print("This is a common Web service port.")
    else:
        print("This is not a common Web service port.")

---

### 3) pod_status_check.py

Requirement: Check whether the Pod status is not "Running."

    pod_status = "Pending"

    if pod_status != "Running":
        print("The Pod status is abnormal.")
    else:
        print("The Pod status is normal.")

---

### 4) log_level_check.py

Requirement: Identify both "ERROR" and "WARNING" in the log line.

    log_line = "warning node disk pressure"

    if "ERROR" in log_line.upper() or "WARNING" in log_line.upper():
        print("Abnormal logs were found.")
    else:
        print("The logs are normal.")

---

### 5) filename_rule_check.py

Requirement: Check both the prefix and suffix of the file name.

    filename = "syslog.log"

    if filename.startswith("sys") and filename.endswith(".log"):
        print("This is the target log file.")
    else:
        print("This is not the target log file.")

---

### 6) mixed_condition_check.py

Requirement: Perform a more comprehensive combined judgment.

    cpu_usage = 88
    memory_usage = 75

    if cpu_usage > 80 and memory_usage > 80:
        print("High risk: Both CPU and memory are high.")
    elif cpu_usage > 80 or memory_usage > 80:
        print("Medium risk: One resource is high.")
    else:
        print("Normal.")It is crucial to understand this because only by knowing the return value of an expression can we avoid mistakenly combining different types of expressions.