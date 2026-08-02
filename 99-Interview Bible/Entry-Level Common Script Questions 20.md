# 20 Common Script Questions for Entry-Level Positions
## Tags
#TheBookOfInterviews #TransportInterview #Linux #Shell #ScriptTheme #LogAnalysis #FieldTransportation

---

## 01. Count the Number of Times ERROR Appears in Logs

### Question
Count the number of lines containing `ERROR` in `app.log`.

### Standard Answer
    grep -c "ERROR" app.log

### Explanation
- `grep -c` directly counts matching lines
- More concise than `grep "ERROR" app.log | wc -l`

---

## 02. View Last 20 Lines of Log

### Question
View the last 20 lines of content in `app.log`.

### Standard Answer
    tail -n 20 app.log

### Explanation
- `tail -n 20` is commonly used to check latest errors
- Real-time tracking can use `tail -f`

---

## 03. Monitor Log Changes in Real-Time

### Question
Monitor new content added to `/var/log/messages` in real-time.

### Standard Answer
    tail -f /var/log/messages

### Explanation
- `-f` indicates follow
- Often used for real-time log observation during troubleshooting

---

## 04. Count Most Accessed URLs in Nginx

### Question
Count the URL with the most accesses in `access.log`.

### Standard Answer
    awk '{print $7}' access.log | sort | uniq -c | sort -nr | head

### Explanation
- `$7` is typically the URL
- `sort | uniq -c` is the high-frequency combination
- `sort -nr` sorts by count in descending order

---

## 05. Count Most Frequent Client IPs in Nginx

### Question
Count the client IP with the most accesses in `access.log`.

### Standard Answer
    awk '{print $1}' access.log | sort | uniq -c | sort -nr | head

### Explanation
- `$1` is typically the client IP

---

## 06. Count 404 Errors by URL

### Question
Count the URL with the most 404 errors in `access.log`.

### Standard Answer
    awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -nr | head

### Explanation
- `$9==404` first filters by status code
- Then counts the URL

---

## 07. Count Total 5xx Errors

### Question
Count the total number of requests with 5xx status codes in `access.log`.

### Standard Answer
    awk '$9 ~ /^5/ {count++} END {print count}' access.log

### Explanation
- `/^5/` indicates starts with 5
- Can match 500, 502, 503, etc.

---

## 08. Find Log Files Larger than 1G

### Question
Find log files larger than 1G in `/var/log`.

### Standard Answer
    find /var/log -type f -name "*.log" -size +1G

### Explanation
- `-type f` indicates regular files
- `-size +1G` indicates larger than 1G

---

## 09. Check Disk Usage

### Question
Check server disk usage.

### Standard Answer
    df -h

### Explanation
- `-h` makes capacity display more intuitive

---

## 10. Check Space Usage of a Directory

### Question
Check how much space the directory `/var/log` occupies.

### Standard Answer
    du -sh /var/log

### Explanation
- `-s` summarizes
- `-h` is human-readable

---

## 11. Check Nginx Service Status

### Question
Check if the Nginx service is running normally.

### Standard Answer
    systemctl status nginx

### Explanation
- Applicable to systemd systems

---

## 12. Restart Nginx Service

### Question
Restart the Nginx service.

### Standard Answer
    systemctl restart nginx

### Explanation
- Usually paired with `systemctl status nginx` for status checks

---

## 13. Check Nginx Configuration for Syntax Errors

### Question
Check if the Nginx configuration file is correct.

### Standard Answer
    nginx -t

### Explanation
- `-t` indicates configuration test
- Test before reload after configuration changes

---

## 14. Reload Nginx Configuration

### Question
Reload the Nginx configuration without stopping the service.

### Standard Answer
    nginx -s reload

### Explanation
- Often used for smooth effect after configuration changes

---

## 15. Check if Port 80 is Listening

### Question
Check if port 80 is actively listening.

### Standard Answer
    ss -tunlp | grep ":80 "

### Explanation
- `ss` is more common in new systems than `netstat`
- Can also use:
      lsof -i:80

---

## 16. Check if a Process Exists

### Question
Check if the Nginx process exists.

### Standard Answer
    ps -ef | grep nginx | grep -v grep

### Explanation
- `grep -v grep` excludes the grep itself

---

## 17. Count Number of Nginx Processes

### Question
Count the number of Nginx processes.

### Standard Answer
    ps -ef | grep nginx | grep -v grep | wc -l

### Explanation
- `wc -l` counts the number of lines

---

## 18. Batch Ping a Group of Hosts

### Question
Read IPs from `ip_list.txt` and test reachability for each.

### Standard Answer
    for ip in $(cat ip_list.txt)
    do
        ping -c 1 $ip >/dev/null 2>&1
        if [ $? -eq 0 ]; then
            echo "$ip All"
        else
            echo "$ip It's not working."
        fi
    done

### Explanation
- `$?` is the return code of the previous command
- `0` indicates success

---

## 19. Check if a File Exists

### Question
Check if `/etc/passwd` exists.

### Standard Answer
    if [ -f /etc/passwd ]; then
        echo "File exists"
    else
        echo "File does not exist"
    fi

### Explanation
- `-f` checks for a regular file
- `-d` checks for a directory
- `-e` checks for existence

---

## 20. Check if Root Partition Usage Exceeds 80%

### Question
Check root partition usage, output alert if it exceeds 80%.

### Standard Answer
    usage=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

    if [ $usage -ge 80 ]; then
        echo "High disk usage"
    else
        echo "Disk usage is normal"
    fi

### Explanation
- `NR==2` indicates taking the second line
- `print $5` extracts the usage field
- `sed 's/%//'` removes the percentage sign

---

# Common Script Examples for Entry-Level Positions

## Example 1: Count ERROR Logs and Check for Anomalies
    #!/bin/bash

    log_file="/var/log/app.log"
    error_count=$(grep -c "ERROR" $log_file 2>/dev/null)

    echo "Error log count: $error_count"

    if [ $error_count -gt 0 ]; then
        echo "Found error logs"
    else
        echo "No error logs found"
    fi

---

## Example 2: Count Top 10 Most Accessed URLs
    #!/bin/bash

    log_file="/var/log/nginx/access.log"

    echo "Top 10 most accessed URLs:"
    awk '{print $7}' $log_file | sort | uniq -c | sort -nr | head -n 10

---

## Example 3: Check if Port 80 is Listening
    #!/bin/bash

    port_count=$(ss -tunlp | grep -c ":80 ")

    if [ $port_count -gt 0 ]; then
        echo "Port 80 is listening"
    else
        echo "Port 80 is not listening"
    fi

---

## Example 4: Check Root Partition Usage
    #!/bin/bash

    usage=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

    echo "Root partition usage: ${usage}%"

    if [ $usage -ge 80 ]; then
        echo "High disk usage"
    else
        echo "Disk usage is normal"
    fi

---

## Example 5: Batch Check Host Connectivity
    #!/bin/bash

    for ip in $(cat ip_list.txt)
    do
        ping -c 1 $ip >/dev/null 2>&1

        if [ $? -eq 0 ]; then
            echo "$ip is reachable"
        else
            echo "$ip is unreachable"
        fi
    done

---

# Key Patterns to Memorize Before Interview

## 1. Log Statistics Standard Patterns
    awk '{print $7}' access.log | sort | uniq -c | sort -nr | head
    awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
    awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -nr | head
    grep -c "ERROR" app.log

## 2. Basic Condition Checks
    if [ $count -gt 0 ]; then
        echo "Anomaly detected"
    else
        echo "Normal"
    fi

## 3. Basic Loops
    for ip in $(cat ip_list.txt)
    do
        ping -c 1 $ip
    done

## 4. Basic System Checks
    df -h
    du -sh /var/log
    systemctl status nginx
    ss -tunlp
    ps -ef | grep nginx | grep -v grep

---

# Common Pitfalls

## 1. `uniq -c` usually requires `sort` first
Correct:
    awk '{print $7}' access.log | sort | uniq -c

## 2. `if` must have spaces around the brackets
Correct:
    if [ $count -gt 0 ]; then

Error:
    if [$count -gt 0]; then

## 3. Variable assignment cannot have spaces
Correct:
    count=10

Error:
    count = 10

## 4. Check log field samples first if uncertain
    head -n 5 access.log

---

# One-Sentence Summary
Common script questions for entry-level operations roles essentially require: mastering `grep`, `awk`, `sort`, `uniq -c`, `wc -l`, `if`, and `for` to solve real-world issues like log statistics, service checks, port checks, and disk inspections.