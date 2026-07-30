# 01-Log Viewing and Basic grep Keyword Filtering

#Linux #Log Analysis #Text Processing #grep #tail #head #wc #sed #Log Viewing #Ops

---

## Recommended Path

01-Linux Basics and Server Ops/02-Logs and Text Processing/01-Log Viewing and Basic grep Keyword Filtering.md

---

## I. Document Overview

This document covers frequently used scenarios in Linux server operations, including **log viewing, log filtering, keyword filtering, context viewing, and log line count statistics**.

The focus of this article is on:

- Basic log viewing operations
- Checking the total number of log lines
- Viewing the latest logs
- Real-time tracking of logs
- Viewing the first few and last few lines of a file
- Viewing logs within a specified range
- Using grep to filter logs by keywords
- Filtering ignoring case
- Displaying line numbers where matches occur
- Reverse filtering
- Viewing the context around matched lines
- Recursively searching for keywords in directories
- Counting the number of occurrences of keywords
- Basic usage of pipes

This article is part of the Log and Text Processing series, focusing on:

```text
How to quickly view logs, filter them, locate keywords, and count errors
```

The goal is to:

- Be able to quickly open logs
- Track logs in real-time
- Filter logs by keywords
- View the context around error logs
- Count the number of errors
- Perform basic log analysis using grep + wc

---

## II. Core Commands Overview

Common commands used for log viewing and keyword filtering include:

```text
tail
→ View the end of logs or track them in real-time
head
→ View the beginning of a file
wc
→ Count lines, words, or bytes
grep
→ Filter logs by keywords
sed
→ View specific lines within a file
Pipe symbol |
→ Pass the output of one command as input to another
```

In summary:

```text
tail is used for viewing the latest logs
head is used for viewing the beginning of a file
grep is used for filtering logs by keywords
wc is used for counting various elements in logs
sed is used for viewing specific lines based on their numbers
The pipe symbol connects multiple commands together
```

---

## III. Basic Concepts

---

## 1. The Pipe Symbol |

### Function

The pipe symbol is used to:

```text
Pass the output of one command as input to another
```

### Examples

```bash
grep "error" app.log | wc -l
```

### Meaning

```text
First, filter out lines containing "error".
Then, count the number of such lines.
```

### Common Combinations

```bash
tail -f app.log | grep -i error
```

```bash
grep "timeout" app.log | wc -l
```

```bash
grep -i "error" app.log | tail -n 20
```

---

## 2. Basic Log Viewing Approaches

When troubleshooting logs, the common sequence is:

```text
First, check if the log exists.
→ Then, consider its size and number of lines.
→ Next, view the latest logs.
→ After that, filter them by keywords.
→ View the context around errors.
→ Count the number of errors.
→ Finally, consider time ranges and service status for further analysis.
```

Common issues to troubleshoot include:

```text
Service fails to start
API timeouts occur
Programs exit abnormally
Database connections fail
Nginx returns 502/504 errors
Permission denied
Disk writes fail
Configuration files cannot be loaded
Dependent services are unavailable
```

---

## IV. Basic Log Viewing Operations

---

## Scenario 1: Checking the Total Number of Log Lines

### Command

```bash
wc -l app.log
```

### Explanation

```text
-l
→ Counts the total number of lines in the file.
```

This is useful for quickly assessing the size of log files.

Example:

```text
If there are only a few dozen lines, it might indicate that the service just started and has produced little log data.
If there are millions of lines, direct viewing with `cat` would be impractical.
```

### Note

Avoid using:

```bash
cat app.log
```

for large log files. Instead, use:

```bash
tail
```

```bash
head
```

```bash
grep
```

```bash
sed
```

---

## Scenario 2: Viewing the Last 100 Lines of Logs

### Command

```bash
tail -n 100 app.log
```

### Explanation

```text
-n 100
→ Displays the last 100 lines of the file.
```

This is useful for:

```text
Checking recent service errors
Viewing logs immediately after a service## Scenario 14: Using Extended Regular Expressions to Match Multiple Keywords
```
### Command

```bash
grep -E "error|failed|timeout" app.log
```

### Explanation

```text
-E
→ Uses extended regular expressions
```

This command searches for lines that contain any of the following keywords:

```text
error

failed

timeout
```

---

## Scenario 15: Viewing the First 3 Lines of Matched Rows

### Command

```bash
grep -B 3 "error" app.log
```

### Explanation

```text
-B 3
→ Displays the first 3 lines of matched rows
```

### Application Scenarios

```text
- Quickly filtering multiple types of anomalies at once
- Quickly checking for errors, failures, and timeouts
- Observing various anomaly keywords after a release
```

---

## Scenario 16: Viewing the Last 3 Lines of Matched Rows

### Command

```bash
grep -A 3 "error" app.log
```

### Explanation

```text
-A 3
→ Displays the last 3 lines of matched rows
```

### Application Scenarios

```text
- Checking request parameters before an error occurs
- Viewing the context before an exception stack is displayed
- Examining configuration loading logs before a service startup failure
```

---

## Scenario 17: Viewing 3 Lines Before and After Each Matched Row

### Command

```bash
grep -C 3 "error" app.log
```

### Explanation

```text
-C 3
→ Displays 3 lines before and after each matched row
```

### Application Scenarios

```text
- Checking the complete error context
- Examining logs before and after an API exception
- Reviewing information before and after a service startup failure
- Analyzing the chain of exceptions
```

---

## Scenario 18: Recursively Searching for Keywords in Directories

### Command

```bash
grep -r "timeout" /var/log
```

### Idea

- Recursively search for the keyword within directories.

### Explanation

```text
-r
→ Recursively search directories
```

### Application Scenarios

```text
- When you don't know which log file contains the error
- When a service has multiple log files
- Searching for a specific exception in directories
- Looking for a certain field in configuration directories
- Finding keywords in historical logs
```

### Common Variants

- Recursively search for "error":

```bash
grep -ri "error" /var/log/myapp
```

- Recursively search for "timeout" and display line numbers:

```bash
grep -rin "timeout" /var/log/myapp
```

---

## Scenario 19: Counting the Occurrences of a Certain Keyword

### Command

```bash
grep "error" app.log | wc -l
```

### Idea

```text
grep "error" app.log
→ First filter out lines containing "error"
wc -l
→ Then count the number of lines
```

### Application Scenarios

```text
- Counting the number of "errors"
- Tracking the frequency of "timeouts"
- Recording the occurrences of "failures"
- Measuring the number of visits to a specific API
- Identifying how many times a particular IP address appears in logs
- Determining how many times a certain exception occurs in log files
```

---

## Scenario 20: Counting Keywords Ignoring Case Sensitivity

### Command

```bash
grep -i "error" app.log | wc -l
```

### Application Scenarios

```text- Logs may contain "error", "Error", or "ERROR"
- It is necessary to count errors consistently regardless of case
```

---

## Scenario 21: Counting the Number of Multiple Exception Keywords

### Command

```bash
grep -Ei "error|failed|timeout|exception" app.log | wc -l
```

### Application Scenarios

```text- Quickly assessing the volume of exception logs
- Observing whether the number of exceptions increases after a release
- Measuring the scale of exceptions during a fault period
```

---

## VI. Common Log Viewing Techniques

---

## Scenario 22: Viewing the Latest Error Logs

### Command

```bash
grep -i "error" app.log | tail -n 50
```

### Idea

```text
- First identify all occurrences of "error"
- Then display the last 50 lines
```

---

## Scenario 23: Realtime Viewing of "Error" and "Timeout" Logs

### Command

```bash
tail -f app.log | grep -Ei "error|timeout"
```

### Application Scenarios

```text
- Monitoring during releases
- Observing after service restarts
- Checking during API load testing
- Reviewing
```

## Basic grep Filtering

```bash
grep "timeout" app.log
```

```bash
grep -i "timeout" app.log
```

```bash
grep -n "timeout" app.log
```

```bash
grep -v "timeout" app.log
```

```bash
grep -E "error|failed|timeout" app.log
```

```bash
grep -Ei "error|failed|timeout|exception" app.log
```

---

## grep Context Scanning

```bash
grep -B 3 "error" app.log
```

```bash
grep -A 3 "error" app.log
```

```bash
grep -C 3 "error" app.log
```

```bash
grep -C 3 -in "error" app.log
```

---

## grep Recursive Directory Search

```bash
grep -r "timeout" /var/log
```

```bash
grep -ri "error" /var/log/myapp
```

```bash
grep -rin "timeout" /var/log/myapp
```

---

## Keyword Counting

```bash
grep "error" app.log | wc -l
```

```bash
grep -i "error" app.log | wc -l
```

```bash
grep -Ei "error|failed|timeout|exception" app.log | wc -l
```

---

## Common Combinations

```bash
grep -i "error" app.log | tail -n 50
```

```bash
tail -f app.log | grep -Ei "error|timeout"
```

```bash
grep -v "healthcheck" access.log | grep -i "error"
```

```bash
grep -v "healthcheck" access.log | wc -l
```

---

## Summary in One Sentence

The core steps of log viewing and grep filtering are:

```text
First, view the latest logs.

→ Then filter by keywords.

→ Examine the context.

→ Count occurrences.

→ Finally, consider the time range for a comprehensive analysis.
```

Basic operations:

```text
tail
→ View latest logs in real-time

head
→ Show the beginning of the file

wc
→ Count lines

sed
→ Select specific line ranges

```

Keyword filtering:

```text
grep
→ Search by keyword

grep -i
→ Ignore case differences

grep -n
→ Display line numbers

grep -v
→ Exclude matches

grep -A / -B / -C
→ Show context around the match

grep -r
→ Recursively search directories

```

Common troubleshooting techniques:

```text
grep -i "error" app.log | tail -n 50

grep -C 3 -in "error" app.log

tail -f app.log | grep -Ei "error|timeout|exception"

grep -i "error" app.log | wc -l
```

Production tips:

```text
Do not directly use `cat` on large logs.

Ignore case when searching for error-related keywords.

Viewing just one line is insufficient; examine the context.

Check not only for “error” but also for “timeout”, “exception”, “failed”, “oom”, etc.

Be mindful of the time range when counting log entries.
```