# 20 Common Footnote Questions for Lower-Level Positions
## Tags
# Interview Guide # Ops Interviews # Linux # Shell # Footnote Questions # Log Analysis # On-Site Operations

---

## 01. Count the number of occurrences of "ERROR" in logs

### Question
Count the number of lines in `app.log` that contain "ERROR".

### Standard Answer
    grep -c "ERROR" app.log

### Explanation
- `grep -c` directly counts the number of matching lines.
- It is more concise than `grep "ERROR" app.log | wc -l`.

---

## 02. View the last 20 lines of logs

### Question
View the last 20 lines of `app.log`.

### Standard Answer
    tail -n 20 app.log

### Explanation
- `tail -n 20` is commonly used to view recent error messages.
- For real-time tracking, use `tail -f`.

---

## 03. Monitor log changes in real time

### Question
Monitor new additions to `/var/log/messages` in real time.

### Standard Answer
    tail -f /var/log/messages

### Explanation
- `-f` indicates follow mode.
- This is often used for troubleshooting to observe logs in real-time.

---

## 04. Count the URL with the most visits in Nginx

### Question
Count the URL that receives the most visits in `access.log`.

### Standard Answer
    awk '{print $7}' access.log | sort | uniq -c | sort -nr | head

### Explanation
- `$7` usually represents the URL.
- `sort | uniq -c` is a common combination for counting unique entries.
- `sort -nr` sorts the results in descending order based on frequency.

---

## 05. Count the client IP with the most visits in Nginx

### Question
Count the client IP that receives the most visits in `access.log`.

### Standard Answer
    awk '{print $1}' access.log | sort | uniq -c | sort -nr | head

### Explanation
- `$1` usually represents the client IP address.

---

## 06. Count the URL with the highest number of 404 errors

### Question
Count the URL that results in the most 404 errors in `access.log`.

### Standard Answer
    awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -nr | head

### Explanation
- `$9==404` first filters out requests with a 404 status code.
- Then it counts the corresponding URLs.

---

## 07. Count the total number of 5xx errors

### Question
Count the total number of requests with 5xx status codes in `access.log`.

### Standard Answer
    awk '$9 ~ /^5/ {count++} END {print count}' access.log

### Explanation
- `/^5/` matches any request starting with the digit 5.
- This will count errors such as 500, 502, and 503.

---

## 08. Locate log files larger than 1GB

### Question
Locate all log files in `/var/log` that are larger than 1GB in size.

### Standard Answer
    find /var/log -type f -name "*.log" -size +1G

### Explanation
- `-type f` specifies that we are looking for regular files.
- `-size +1G` ensures that the file size exceeds 1GB.

---

## 09. Check disk usage

### Question
Check the overall disk usage of the server.

### Standard Answer
    df -h

### Explanation
- `-h` makes the output easier to read in human-readable units.

---

## 10. Check the space occupied by a directory

### Question
Find out how much space the `/var/log` directory is using.

### Standard Answer
    du -sh /var/log

### Explanation
- `-s` calculates the total size.
- `-h` displays the result in human-readable format.

---

## 11. Verify the status of the nginx service

### Question
Check whether the nginx service is running normally.

### Standard Answer
    systemctl status nginx

### Explanation
- This command is suitable for systemd-based systems.

---

## 12. Restart the nginx service

### Question
Restart the nginx service.

### Standard Answer
    systemctl restart nginx

### Explanation
- Checking the status is usually done together with this command.

---

## 13. Verify that nginx configuration is syntax correct

### Question
Check whether the nginx configuration file contains any syntax errors.

### Standard Answer
    nginx -t

### Explanation
- `-t` tests the configuration for correctness.
- Always test the configuration before applying it to production.

---

## 1```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -nr | head
grep -c "ERROR" app.log

## 2. Basic Judgment
    if [ $count -gt 0 ]; then
        echo "Abnormality detected"
    else
        echo "Normal"
    fi

## 3. Basic Loop
    for ip in $(cat ip_list.txt)
    do
        ping -c 1 $ip
    done

## 4. Basic Checks
    df -h
    du -sh /var/log
    systemctl status nginx
    ss -tunlp
    ps -ef | grep nginx | grep -v grep

---

# Common Mistakes

## 1. `uniq -c` should always follow `sort`
Correct:
    awk '{print $7}' access.log | sort | uniq -c

## 2. Spaces are required around the `[` and `]` in `if` statements
Correct:
    if [ $count -gt 0 ]; then

Wrong:
    if [$count -gt 0]; then

## 3. No spaces should be used when assigning variables
Correct:
    count=10

Wrong:
    count = 10

## 4. Always check sample log entries before working with unknown fields
    head -n 5 access.log
```

---

# In Summary
Common basic tasks in low-level operations involve using tools like `grep`, `awk`, `sort`, `uniq -c`, `wc -l`, `if`, and `for` to handle issues related to log analysis, service monitoring, port checks, and disk health checks.