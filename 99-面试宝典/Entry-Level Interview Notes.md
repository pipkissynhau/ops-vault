# Low-Level Job Interview Quick Notes

## Tags
#TheBookOfInterviews #TransportInterview #Linux #Shell #Nginx #LogAnalysis #ScriptFoundation #SortStatistics

---

## 1. How to Understand Commands for Sorting, Statistics, and Extraction

Common script questions for low-level positions, essentially 4 steps:

1. Filter the needed data first
2. Extract a specific column
3. Sort the data
4. Perform statistics and take top results

Common commands are:

- `grep`: Filter keywords
- `awk`: Extract columns, conditional filtering
- `cut`: Split fields by delimiter
- `sort`: Sort
- `uniq -c`: Count duplicate occurrences
- `wc -l`: Count lines
- `head`: Take top N lines
- `tail`: Take last N lines

---

## 2. Detailed Quick Notes for Sorting and Statistics Commands

---

### 2.1 `sort`: Sort

#### Basic syntax
    sort file.txt

Sorts in ascending dictionary order by default.

#### Example
Original file content:
    nginx
    mysql
    apache

Execution:
    sort file.txt

Result:
    apache
    mysql
    nginx

---

### 2.2 `sort -n`: Sort by Numbers

#### Example
Original file content:
    100
    20
    3

Execution:
    sort -n file.txt

Result:
    3
    20
    100

#### Explanation
Without using `-n`, it would sort as strings, not actual numeric values.

---

### 2.3 `sort -r`: Reverse Sort
    sort -r file.txt

---

### 2.4 `sort -nr`: Sort by Numbers in Reverse
This is most commonly used.

#### Example
    echo -e "3\n100\n20" | sort -nr

Result:
    100
    20
    3

#### Most Common Interview Scenario
Used after `uniq -c` for frequency statistics, then sorted by frequency in reverse order.

---

### 2.5 `uniq -c`: Count Occurrences of Duplicate Lines

#### Example
    echo -e "a\na\nb\nb\nb" | uniq -c

Result typically looks like:
          2 a
          3 b

#### Key Point
`uniq -c` must usually come after `sort`, otherwise non-consecutive duplicates won't be counted accurately.

#### Correct Syntax
    awk '{print $1}' access.log | sort | uniq -c

---

### 2.6 `wc -l`: Count Lines

#### Example
    grep "ERROR" app.log | wc -l

Purpose:
Count the number of lines in the filtered result.

#### More Direct Syntax
    grep -c "ERROR" app.log

---

### 2.7 `head`: Take Top N Lines
    head -n 10 access.log

Common uses:
- View log samples
- Take top 10 results from statistics

---

### 2.8 `tail`: Take Last N Lines
    tail -n 20 access.log
    tail -f /var/log/messages

Common uses:
- View latest logs
- Real-time log tracking

---

## 3. Detailed Usage of `grep`, `awk`, `cut`

---

### 3.1 `grep`: Filter Keywords

#### Basic Usage
    grep "ERROR" app.log

#### Common Parameters
    grep -i "error" app.log
    grep -v "INFO" app.log
    grep -n "mysql" app.log
    grep -c "ERROR" app.log

#### Meanings
- `-i`: Ignore case
- `-v`: Inverse match, exclude
- `-n`: Show line numbers
- `-c`: Count matching lines

---

### 3.2 `awk`: Most Common Column Extraction Tool

#### Basic Syntax
    awk '{print $1}' access.log

Means print the first column.

#### Common Column Numbers
Using an Nginx access log example:
    10.0.0.1 - - [01/Apr/2026:10:00:00 +0800] "GET /index HTTP/1.1" 200 123

Common interpretations:
- `$1`: Client IP
- `$7`: URL path
- `$9`: Status code

#### Common Syntax
Get IP:
    awk '{print $1}' access.log

Get URL:
    awk '{print $7}' access.log

Get status code:
    awk '{print $9}' access.log

---

### 3.3 `awk`: Conditional Filtering

Status code equals 404:
    awk '$9==404 {print $7}' access.log

Status code is 5xx:
    awk '$9 ~ /^5/ {print $7}' access.log

#### Explanation
- `$9==404`: 9th column equals 404
- `$9 ~ /^5/`: 9th column starts with 5

---

### 3.4 `cut`: Split by Delimiter

#### Example
    echo "nginx:80:running" | cut -d':' -f1

Result:
    nginx

#### Explanation
- `-d':'`: Specify colon as delimiter
- `-f1`: Take first field

#### Common Scenarios
Very useful when dealing with fixed delimiters in configuration files or command outputs.

---

## 4. Dissecting the Most Common Statistics Commands in Interviews

---

### 4.1 Count URLs with the Most Accesses
    awk '{print $7}' access.log | sort | uniq -c | sort -nr | head

#### Step-by-Step Explanation
1. `awk '{print $7}' access.log`
   - Extract URL from each line

2. `sort`
   - Group identical URLs together

3. `uniq -c`
   - Count occurrences of each URL

4. `sort -nr`
   - Sort by count in descending order

5. `head`
   - Take top results

#### This command can be memorized directly
It's very common in interviews.

---

### 4.2 Count the most visited client IP
    awk '{print $1}' access.log | sort | uniq -c | sort -nr | head

The logic is the same as above, just replace `$7` with `$1`.

---

### 4.3 Count the most frequent 404 URLs
    awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -nr | head

#### Meaning
First filter 404, then count URLs.

---

### 4.4 Count 5xx error occurrences
    awk '$9 ~ /^5/ {count++} END {print count}' access.log

#### Explanation
- Increment `count` by 1 when encountering a 5xx
- Output total count at the end

---

### 4.5 Count ERROR lines
    grep -c "ERROR" app.log

#### If you want to see the content
    grep "ERROR" app.log

#### If you want to see recent ERRORs
    grep "ERROR" app.log | tail -n 20

---

## 5. Shell scripting basics often forgotten

---

### 5.1 Script header
    #!/bin/bash

This indicates using bash for interpretation.

---

### 5.2 Variable assignment
    log_file="/var/log/nginx/access.log"
    echo $log_file

#### Note
No space allowed around the equals sign.

Correct:
    count=10

Error:
    count = 10

---

### 5.3 Assign command output to variable
    count=$(grep -c "ERROR" app.log)

#### Explanation
Store command execution result in variable `count`.

---

### 5.4 If statement
    if [ $count -gt 0 ]; then
        echo "Found errors"
    else
        echo "No errors"
    fi

#### Note
- `[ ]` must have spaces on both sides
- `then` must be preceded by a semicolon or line break
- End with `fi`

#### Common comparisons
- `-gt`: Greater than
- `-lt`: Less than
- `-eq`: Equal to
- `-ge`: Greater than or equal to
- `-le`: Less than or equal to

---

### 5.5 String comparison
    if [ "$status" = "running" ]; then
        echo "Service is normal"
    fi

#### Note
String comparison commonly uses `=`.

---

### 5.6 For loop
    for ip in 10.0.0.1 10.0.0.2 10.0.0.3
    do
        ping -c 1 $ip
    done

#### Read from file
    for ip in $(cat ip_list.txt)
    do
        ping -c 1 $ip
    done

---

### 5.7 Check if file exists
    if [ -f /etc/passwd ]; then
        echo "File exists"
    fi

#### Common checks
- `-f`: Regular file
- `-d`: Directory
- `-e`: File exists

---

## 6. Low-level job common script examples

The following scripts are ideal for basic operations, on-site support, and field support positions.

---

## 6.1 Example 1: Count ERRORs in logs

### Script
    #!/bin/bash

    log_file="/var/log/app.log"
    error_count=$(grep -c "ERROR" $log_file 2>/dev/null)

    echo "Error log count: $error_count"

    if [ $error_count -gt 0 ]; then
        echo "Found error logs"
    else
        echo "No error logs found"
    fi

### Purpose
- Count ERRORs in logs
- Provide simple judgment

### Interview value
This is one of the most basic and realistic scripts.

---

## 6.2 Example 2: Count top 10 most visited URLs in Nginx logs

### Script
    #!/bin/bash

    log_file="/var/log/nginx/access.log"

    echo "Top 10 most visited URLs:"
    awk '{print $7}' $log_file | sort | uniq -c | sort -nr | head -n 10

### Purpose
- Find hot URLs
- Analyze which pages have concentrated traffic

### Interview value
Classic template for log analysis questions.

---

## 6.3 Example 3: Check if port 80 is listening

### Script
    #!/bin/bash

    port_count=$(ss -tunlp | grep -c ":80 ")

    if [ $port_count -gt 0 ]; then
        echo "Port 80 is listening"
    else
        echo "Port 80 is not listening"
    fi

### Purpose
- Determine if web service is running

### Interview value
Very suitable for on-site, on-call, and inspection roles.

---

## 6.4 Example 4: Check if disk usage is too high

### Script
    #!/bin/bash

    usage=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

    echo "Root partition usage: ${usage}%"

    if [ $usage -ge 80 ]; then
        echo "Disk usage is too high"
    else
        echo "Disk usage is normal"
    fi

### Explanation
- `df -h /`: Check root partition
- `awk 'NR==2 {print $5}'`: Take second line's fifth column
- `sed 's/%//'`: Remove percentage sign

### Interview value
Typical question for basic inspection scripts.

---

## 6.5 Example 5: Batch ping hosts

### Script
    #!/bin/bash

```bash
for ip in $(cat ip_list.txt)
do
    ping -c 1 $ip >/dev/null 2>&1

    if [ $? -eq 0 ]; then
        echo "$ip All"
    else
        echo "$ip It's not working."
    fi
done
```

### Explanation
- `$?`: Return code from the previous command
- `0`: Usually indicates success
- `>/dev/null 2>&1`: Suppresses ping output

### Interview Value
Very common, suitable for entry-level positions.

---

## 6.6 Example Six: Check if a Service Process Exists

### Script
    #!/bin/bash

    process_count=$(ps -ef | grep nginx | grep -v grep | wc -l)

    if [ $process_count -gt 0 ]; then
        echo "nginx process exists"
    else
        echo "nginx process does not exist"
    fi

### Notes
- `grep -v grep`: Excludes the line from grep itself
- `wc -l`: Counts the number of matching lines

### Interview Value
Common for basic service troubleshooting.

---

## 6.7 Example Seven: Find Log Files Larger than 1G

### Script
    #!/bin/bash

    echo "Log files larger than 1G:"
    find /var/log -type f -name "*.log" -size +1G

### Purpose
- Find large logs
- Check for abnormal disk usage

### Interview Value
Suitable for questions like "How to check when the disk is full."

---

## 7. How to Explain Your Script in an Interview

When explaining a script, follow this order:

1. First state the script's purpose
2. Then explain the core commands
3. Then describe the logic
4. Finally mention the applicable scenarios

---

### Example Expression
"This script is mainly used for log anomaly inspection. It first uses grep to count ERROR occurrences, saves the result to a variable, and finally uses if to check if it's greater than 0, outputting the inspection result. This approach is suitable for basic log inspection and scheduled task checks."

---

## 8. Common Pitfalls

### 8.1 Forgetting `sort` before `uniq -c`
Correct:
    awk '{print $7}' access.log | sort | uniq -c

---

### 8.2 No space around brackets in [!]
Correct:
    if [ $count -gt 0 ]; then

Error:
    if [$count -gt 0]; then

---

### 8.3 Variable assignment with space
Correct:
    count=10

Error:
    count = 10

---

### 8.4 Directly memorizing field numbers without knowing the fields
More reliable statement:
"I would first use head to check a sample, then confirm the field position."

---

## 9. Key Commands to Memorize Before an Interview

    awk '{print $7}' access.log | sort | uniq -c | sort -nr | head
    awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
    awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -nr | head
    grep -c "ERROR" app.log
    df -h / | awk 'NR==2 {print $5}'
    ss -tunlp | grep -c ":80 "
    ps -ef | grep nginx | grep -v grep | wc -l

---

## 10. One-Sentence Summary
For entry-level positions, script questions focus on using `grep`, `awk`, `sort`, `uniq -c`, `wc -l`, `if`, and `for` to solve realTransport issues like log statistics, port checks, service inspections, and disk checks.