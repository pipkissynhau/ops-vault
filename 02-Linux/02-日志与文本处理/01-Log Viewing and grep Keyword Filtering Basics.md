# 01-Log Viewing and grep Keyword Filtering Basics

#Linux #LogAnalysis #TextProcessing #grep #tail #head #wc #sed #LogView #Transport

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/01-Log Viewing and grep Keyword Filtering Basics.md

---

## Section 1: Document Overview

This document organizes common **log viewing, log filtering, keyword filtering, context viewing, and log line counting** scenarios in Linux host maintenance.

This article focuses on:

- Basic log viewing operations
- Viewing total log lines
- Viewing latest logs
- Real-time log tracking
- Viewing first/last few lines of a file
- Viewing logs within a specific range
- Using grep for keyword-based log filtering
- Case-insensitive filtering
- Displaying matching line numbers
- Reverse filtering
- Viewing context around matching lines
- Recursive keyword search in directories
- Keyword occurrence counting
- Basic usage of the pipe operator

This article is the first in the Logs and Text Processing series, primarily addressing:

```text
How quickly to view logs, filter logs, location keys, number of statistical errors
```

The goal is:

Be able to quickly open logs

→ Be able to real-time track logs

→ Be able to filter logs by keywords

→ Be able to view context around error logs

→ Be able to count error occurrences

→ Be able to perform basic log analysis with grep + wc

---

## Section 2: Core Command Localization

Common commands for log viewing and keyword filtering are as follows:

```text
tail
→ View log tail, track log in real time

head
→ View the beginning of the file

wc
→ Statistics rows, words, bytes

grep
→ Filter the log by keyword

sed
→ View specified line ranges

Pipes |
→ Give the output of the previous command to the latter and proceed.
```

One-sentence understanding:

```text
tail I'm in charge of the latest log.

head Read the beginning of the file.

grep Screening key

wc Responsible for statistics

sed Check by line number.

The conduit is responsible for stringing up multiple commands.
```

---

## Section 3: Basic Concepts

---

## 1. Pipe Operator |

### Function

The function of the pipe operator is:

```text
Output of the previous command as input of the latter command
```

### Example

```bash
grep "error" app.log | wc -l
```

### Meaning

```text
Filter inclusion first error Lines

Recount
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

## 2. Basic Log Viewing Approach

When troubleshooting logs, the common sequence is:

```text
Let's see if the log exists.

→ Read log size and lines again

→ Read the latest log.

→ And then filter by keyword.

→ Look at the wrong context again

→ Recount the number of errors

→ Finally, when considering the time frame and service status
```

Common troubleshooting issues include:

```text
Service startup failed

Interface Timeout

The program exited abnormally.

Database connection failed

Nginx 502 / 504

Permission denied

Failed to write disk

Failed to load profile

Reliance on services not available
```

---

## Section 4: Basic Log Viewing Actions

---

## Scenario 1: View Total Log Lines

### Command

```bash
wc -l app.log
```

### Approach

Directly count the total number of lines in the file.

### Notes

```text
-l
→ Number of statistical lines
```

Commonly used to quickly assess log scale.

Example:

```text
It's only a few dozen lines.
→ Could be the service just started. It's a smaller log.

Logs for millions of lines
→ It's big. It's direct. cat It's not appropriate.
```

### Caution

Not recommended to use:

```bash
cat app.log
```

To view large log files.

For large log files, it's recommended to use:

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

## Scenario 2: View Latest 100 Log Lines

### Command

```bash
tail -n 100 app.log
```

### Approach

View the last 100 lines of the log.

### Common Parameters

```text
-n 100
→ Show Last 100 Okay.
```

### Applicable Scenarios

```text
View Service Recent Error Reporting

View log just started

View recent interface requests

Check the last abnormal stack.

View log before and after service restart
```

### Common Variants

View latest 20 lines:

```bash
tail -n 20 app.log
```

View latest 200 lines:

```bash
tail -n 200 app.log
```

View latest 500 lines:

```bash
tail -n 500 app.log
```

---

## Scenario 3: Real-time Log Viewing

### Command

```bash
tail -f app.log
```

### Approach

Continuously track newly added log content.

### Common Parameters

```text
-f
→ New content for real time tracking files
```

### Applicable Scenarios

```text
Observation log after service restart

Watch logs when interface requests

Observation log at pressure

Checking whether the program continues to report errors

See if the timing is working properly.
```

### Stop Method

```text
Ctrl + C
```

---

## Scenario 4: Real-time Error Log Viewing

### Command

```bash
tail -f app.log | grep -i error
```

### Approach

```text
tail -f app.log
→ Add new log in real time

grep -i error
→ Show only inclusion error Line, ignore case
```

### Applicable Scenarios

```text
Real-time observation services reporting errors

After release, check if there are any error

Observe anomaly log after restart

Observe error output at pressure
```

### Common Variants

Real-time viewing of timeout:

```bash
tail -f app.log | grep -i timeout
```

Real-time viewing of exception:

```bash
tail -f app.log | grep -i exception
```

Real-time viewing of failed:

```bash
tail -f app.log | grep -i failed
```

---

## Scenario 5: View First 10 Lines of a File

### Command

```bash
head file
```

### Notes

Default displays first 10 lines of the file.

### Applicable Scenarios

```text
View Log Format

View CSV Header

View the beginning of the profile

Confirm document content structure
```

---

## Scenario 6: View Last 10 Lines of a File

### Command

```bash
tail file
```

### Notes

Default displays last 10 lines of the file.

### Applicable Scenarios

```text
View Recent Log

View end of file

Confirm Task Final Output
```

---

## Scenario 7: View Specified First N Lines

### Command

```bash
head -n 20 file
```

### Example

View first 50 lines:

```bash
head -n 50 app.log
```

View first 100 lines:

```bash
head -n 100 app.log
```

---

## Scenario 8: View Specified Last N Lines

### Command

```bash
tail -n 20 file
```

### Example

View last 50 lines:

```bash
tail -n 50 app.log
```

View last 100 lines:

```bash
tail -n 100 app.log
```

---

## Scenario 9: View Log Lines from 100 to 120

### Command

```bash
sed -n '100,120p' app.log
```

### Approach

Only print lines within the specified range.

### Common Parameters

```text
-n
→ Silent mode, output all lines without default

p
→ Print content matching
```

### Applicable Scenarios

```text
Based on grep -n Check the surroundings after the wrong line number

View a paragraph in a large file

Do not want to open the entire big log file

An anomaly log to locate the specified line range
```

### Example

View lines 200 to 260:

```bash
sed -n '200,260p' app.log
```

View lines 1000 to 1050:

```bash
sed -n '1000,1050p' app.log
```

---

## Section 5: grep - Keyword Filtering and Context Viewing

---

## Scenario 10: Filter Logs Containing a Specific Keyword

### Command

```bash
grep "timeout" app.log
```

### Approach

Output all lines containing `timeout`.

### Applicable Scenarios

```text
Find Timeout Log

Find Error Log

Find an interface

Find a user ID

Find an order number

Find Some IP

Find an abnormal keyword
```

---

## Scenario 11: Case-insensitive Filtering

### Command

```bash
grep -i "timeout" app.log
```

### Notes

```text
-i
→ Ignore case
```

Can match:

```text
timeout

Timeout

TIMEOUT
```

### Common Usage

```bash
grep -i "error" app.log
```

```bash
grep -i "exception" app.log
```

```bash
grep -i "failed" app.log
```

---

## Scenario 12: Display Matching Line Numbers

### Command

```bash
grep -n "timeout" app.log
```

### Notes

```text
-n
→ Show line numbers for content matching
```

### Applicable Scenarios

```text
Need to locate the wrong line number

Follow-up sed View Error Near

Need to explain log location to others

We need to record the evidence.
```

### Example

First find error line numbers:

```bash
grep -n "error" app.log
```

Assuming the error is on line 1200, then view context:

```bash
sed -n '1180,1220p' app.log
```

---

## Scenario 13: Reverse Filtering - Exclude Specific Keywords

### Command

```bash
grep -v "timeout" app.log
```

### Notes

```text
-v
→ Invert matching, output does not contain keyword lines
```

### Applicable Scenarios

```text
Exclude irrelevant logs

Exclude health check logs

Exclude debug Log

Excludes a fixed noise keyword

Filters normal requests, only anomalies.
```

### Example

Exclude healthcheck:

```bash
grep -v "healthcheck" access.log
```

Exclude DEBUG:

```bash
grep -v "DEBUG" app.log
```

---

## Scenario 14: Use Extended Regular Expressions to Match Multiple Keywords

### Command

```bash
grep -E "error|failed|timeout" app.log
```

### Notes

```text
-E
→ Use Extension Regular
```

This command matches lines containing any of the following keywords:

```text
error

failed

timeout
```

### Case-insensitive Matching

```bash
grep -Ei "error|failed|timeout" app.log
```

### Applicable Scenarios

```text
One-time screening for multiple types of anomalies

Quick view error, failure, timeout

Watch a lot of unusual keywords after release
```

---

## Scenario 15: View 3 Lines Before Matching Lines

### Command

```bash
grep -B 3 "error" app.log
```

### Notes

```text
-B 3
→ Show before matching line 3 Okay.
```

### Applicable Scenarios

```text
Request parameter before error occurs

Context before an abnormal stack

Configure log before service start failed
```

---

## Scenario 16: View the next 3 lines after a matching line

### Command

```bash
grep -A 3 "error" app.log
```

### Explanation

```text
-A 3
→ Show after matching lines 3 Okay.
```

### Applicable Scenarios

```text
View stack information after error

View abnormal follow-up output

View specific causes of failure
```

---

## Scenario 17: View 3 lines before and after a matching line

### Command

```bash
grep -C 3 "error" app.log
```

### Explanation

```text
-C 3
→ Show matching lines around 3 Okay.
```

### Applicable Scenarios

```text
View Full Error Context

View interface abnormal logs

View information before and after service startup failed

Analyse the abnormal link.
```

---

## Scenario 18: Recursively search for keywords in a directory

### Command

```bash
grep -r "timeout" /var/log
```

### Approach

Recursively search for keywords within the directory.

### Explanation

```text
-r
→ Recursive directory
```

### Applicable Scenarios

```text
I don't know which log file was wrong.

Service has multiple log files

Search for an anomaly by directory

Finds a field in the configuration directory

Find Keywords in History Log
```

### Common Variants

Recursive search for error:

```bash
grep -ri "error" /var/log/myapp
```

Recursive search for timeout and show line numbers:

```bash
grep -rin "timeout" /var/log/myapp
```

---

## Scenario 19: Count occurrences of a specific keyword

### Command

```bash
grep "error" app.log | wc -l
```

### Approach

```text
grep "error" app.log
→ Filter Organisation error Lines

wc -l
→ Recount Rows
```

### Applicable Scenarios

```text
Statistics error Number

Statistics timeout Number

Statistics failed Number

Count the number of visits to a certain interface

Count a certain IP Number of occurrences

Count how much an anomaly appears in the log. Minor
```

---

## Scenario 20: Count occurrences of a keyword case-insensitively

### Command

```bash
grep -i "error" app.log | wc -l
```

### Applicable Scenarios

```text
It may appear in the log at the same time error / Error / ERROR

Need to harmonize the number of statistical errors
```

---

## Scenario 21: Count occurrences of multiple error keywords

### Command

```bash
grep -Ei "error|failed|timeout|exception" app.log | wc -l
```

### Applicable Scenarios

```text
Rapidly judge the size of the anomaly log

Watch if the anomaly increases after the release.

Statistical anomaly level during malfunction
```

---

## VI. Common Log Viewing Command Patterns

---

## Scenario 22: View the latest error logs

### Command

```bash
grep -i "error" app.log | tail -n 50
```

### Approach

```text
Find everything first. error

And look at the end. 50 Article
```

---

## Scenario 23: Real-time view of error and timeout logs

### Command

```bash
tail -f app.log | grep -Ei "error|timeout"
```

### Applicable Scenarios

```text
Release observation

Service restart observation

Interface pressure observation

Post-facility recovery observation
```

---

## Scenario 24: Find error and show line numbers

### Command

```bash
grep -in "error" app.log
```

### Explanation

```text
-i
→ Ignore case

-n
→ Show Line Numbers
```

---

## Scenario 25: Find error and view context

### Command

```bash
grep -Cin "error" app.log
```

### Explanation

```text
-C
→ Show context

-i
→ Ignore case

-n
→ Show Line Numbers
```

Note:

```text
grep -C Default needs to specify the number of lines in the context
More commonly written: grep -C 3 -in "error" app.log
```

Recommended syntax:

```bash
grep -C 3 -in "error" app.log
```

---

## Scenario 26: Search for errors after excluding health check logs

### Command

```bash
grep -v "healthcheck" access.log | grep -i "error"
```

### Applicable Scenarios

```text
Too many health checks.

It's not about log interference.

We need to filter noise first.
```

---

## Scenario 27: Count log lines excluding health check logs

### Command

```bash
grep -v "healthcheck" access.log | wc -l
```

---

## VII. Common Log Keywords

---

## 1. General Error Keywords

```text
error

failed

fail

exception

timeout

refused

denied

fatal

panic

killed

oom

no space left

too many open files
```

---

## 2. Network-related Keywords

```text
connection refused

connection timed out

connection reset

no route to host

network unreachable

broken pipe

upstream timed out

host not found
```

---

## 3. Permission-related Keywords

```text
permission denied

access denied

unauthorized

forbidden

authentication failed

invalid token
```

---

## 4. Disk-related Keywords

```text
no space left on device

read-only file system

disk quota exceeded

input/output error

I/O error
```

---

## 5. Memory-related Keywords

```text
out of memory

oom

killed process

cannot allocate memory

memory limit
```

---

## VIII. Production Troubleshooting Notes

---

## 1. Avoid direct cat on large logs

Not recommended:

```bash
cat app.log
```

Large logs are recommended to use:

```bash
tail -n 100 app.log
```

```bash
grep "Keyword" app.log
```

```bash
sed -n 'Start Line,End Linep' app.log
```

---

## 2. Be mindful of case sensitivity in grep

Logs may contain:

```text
error

Error

ERROR
```

Thus, when troubleshooting errors, commonly use:

```bash
grep -i "error" app.log
```

---

## 3. Review context when searching for keywords

Reviewing a single line may not be sufficient.

Recommended to combine:

```bash
grep -A 3 "error" app.log
```

```bash
grep -B 3 "error" app.log
```

```bash
grep -C 3 "error" app.log
```

---

## 4. Don't only look for error logs

Some serious issues may not contain `error`.

Also check:

```text
exception

timeout

failed

refused

denied

oom

killed

no space left

too many open files
```

---

## 5. Count occurrences with time range

For example:

```bash
grep -i "error" app.log | wc -l
```

Can only count error occurrences in the entire file.

If the log file is large and spans multiple days, this count may not represent current failures.

A better approach is to combine time ranges, which will be further processed in the Nginx log analysis and awk section.

---

## IX. Summary of Common Commands in This Article

---

## View line count

```bash
wc -l app.log
```

---

## View log tail

```bash
tail -n 100 app.log
```

```bash
tail -n 20 app.log
```

```bash
tail -n 500 app.log
```

---

## Real-time log viewing

```bash
tail -f app.log
```

```bash
tail -f app.log | grep -i error
```

```bash
tail -f app.log | grep -Ei "error|timeout|exception"
```

---

## View file head and tail

```bash
head file
```

```bash
tail file
```

```bash
head -n 20 file
```

```bash
tail -n 20 file
```

---

## View specific line range

```bash
sed -n '100,120p' app.log
```

```bash
sed -n '200,260p' app.log
```

```bash
sed -n '1000,1050p' app.log
```

---

## Basic grep filtering

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

## grep with context

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

## grep in recursive directory

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

## Keyword statistics

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

## Common combinations

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

## X. One-sentence Summary

The core of log viewing and grep filtering is:

```text
Let's start with the latest log.

→ And then filter by keyword.

→ Look at the context.

→ Recount

→ Final determination of impact in relation to time frames
```

Basic viewing:

```text
tail
→ Read the latest logs, track them in real time.

head
→ Look at the beginning of the file.

wc
→ Number of statistical lines

sed
→ View specified line ranges
```

Keyword filtering:

```text
grep
→ Filter by Keyword

grep -i
→ Ignore case

grep -n
→ Show Line Numbers

grep -v
→ Reverse Filter

grep -A / -B / -C
→ View Context

grep -r
→ Recursive directory search
```

Common troubleshooting patterns:

```text
grep -i "error" app.log | tail -n 50

grep -C 3 -in "error" app.log

tail -f app.log | grep -Ei "error|timeout|exception"

grep -i "error" app.log | wc -l
```

Production recommendations:

```text
Don't direct the big log. cat

Error keyword ignores case

It's not enough to look at one line. It depends on the context.

Don't just check. errorWe have to check it too. timeoutI don't know.exceptionI don't know.failedI don't know.oom Keywords

Keep in mind log time frames when counting numbers
```