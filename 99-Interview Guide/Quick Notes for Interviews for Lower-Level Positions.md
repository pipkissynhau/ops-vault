# Quick Notes for Interviews for Lower-Level Positions

## Tags
# Interview Guide # Operations and Maintenance Interview # Linux # Shell # Nginx # Log Analysis # Script Basics # Sorting and Statistics

---

## 1. How to Understand Commands for Sorting, Statistics, and Extraction

Common script questions in lower-level positions essentially involve these four steps:

1. First, filter the required data.
2. Then extract a specific column.
3. Next, sort the data.
4. Finally, perform statistics and select the top few results.

Common commands include:

- `grep`: Filter by keywords.
- `awk`: Extract columns or apply conditional filtering.
- `cut`: Split fields based on delimiters.
- `sort`: Sort the data.
- `uniq -c`: Count the number of occurrences of duplicate entries.
- `wc -l`: Calculate the total number of lines.
- `head`: Display the first few lines.
- `tail`: Display the last few lines.

---

## 2. Detailed Quick Notes on Sorting and Statistics Commands

---

### 2.1 `sort`: Sort Data

#### Basic Usage
    sort file.txt

By default, it sorts data in ascending alphabetical order.

#### Example
Original file content:
    nginx
    mysql
    apache

After execution:
    sort file.txt

Result:
    apache
    mysql
    nginx

---

### 2.2 `sort -n`: Sort Data Numerically

#### Example
Original file content:
    100
    20
    3

After execution:
    sort -n file.txt

Result:
    3
    20
    100

#### Note
Without `-n`, it sorts data as strings, not by actual numerical values.

---

### 2.3 `sort -r`: Sort Data in Reverse Order

    sort -r file.txt

---

### 2.4 `sort -nr`: Sort Data Numerically in Reverse Order
This is the most commonly used option.

#### Example
    echo -e "3\n100\n20" | sort -nr

Result:
    100
    20
    3

#### Application in Interviews
It is often used in combination with `uniq -c` to first count the occurrences of each value and then sort them in reverse order.

---

### 2.5 `uniq -c`: Count the Number of Duplicate Lines

#### Example
    echo -e "a\na\nb\nb\nb" | uniq -c

Result:
          2 a
          3 b

#### Important Note
`uniq -c` should usually be used after `sort` to ensure accurate results if duplicate entries are not consecutive.

#### Correct Usage
    awk '{print $1}' access.log | sort | uniq -c

---

### 2.6 `wc -l`: Count the Total Number of Lines

#### Example
    grep "ERROR" app.log | wc -l

This command counts the number of lines in the filtered result.

#### More Direct Usage
    grep -c "ERROR" app.log

---

### 2.7 `head`: Display the First Few Lines

    head -n 10 access.log

Common uses include:

- Viewing sample log data.
- Obtaining the top few results of a statistical analysis.

---

### 2.8 `tail`: Display the Last Few Lines

    tail -n 20 access.log
    tail -f /var/log/messages

Common uses include:

- Checking the latest log entries.
- Monitoring logs in real time.

---

## 3. Detailed Usage of `grep`, `awk`, and `cut`

---

### 3.1 `grep`: Filter by Keywords

#### Basic Usage
    grep "ERROR" app.log

#### Common Parameters
    grep -i "error" app.log: Ignore case differences.
    grep -v "INFO" app.log: Exclude matches.
    grep -n "mysql" app.log: Display line numbers of matches.
    grep -c "ERROR" app.log: Count the number of matches.

#### Explanation
- `-i`: Ignore case differences.
- `-v`: Exclude matches.
- `-n`: Display line numbers.
- `-c`: Count the number of matches.

---

### 3.2 `awk`: A Powerful Tool for Extracting Columns

#### Basic Usage
    awk '{print $1}' access.log

This command prints the first column of each line in the file `access.log`.

#### Common Column References
In Nginx access logs, for example:

- `$1`: Client IP address.
- `$7`: URL path.
- `$9`: Status code.

#### Common Usage Examples
To extract the client IP address:
    awk '{print $1}' access.log

To extract the URL path:
    awk '{print $7}' access.log

To extract the status code:
    awk '{print $9}' access### Functions
- Count the number of ERROR entries in logs.
- Provide basic judgment based on the statistics.

### Interview Value
This is one of the most fundamental and realistic script examples for job interviews.

---

## 6.2 Example 2: Listing the Top 10 Most Accessed URLs for Nginx

### Script
    #!/bin/bash

    log_file="/var/log/nginx/access.log"

    echo "Top 10 most accessed URLs:"
    awk '{print $7}' $log_file | sort | uniq -c | sort -nr | head -n 10

### Functions
- Identify popular URLs.
- Analyze which pages receive the most visits.

### Interview Value
A classic template for log analysis questions in interviews.

---

## 6.3 Example 3: Checking if Port 80 is Listening

### Script
    #!/bin/bash

    port_count=$(ss -tunlp | grep -c ":80 ")

    if [ $port_count -gt 0 ]; then
        echo "Port 80 is listening."
    else
        echo "Port 80 is not listening."
    fi

### Functions
- Determine whether the web service is running.

### Interview Value
Very suitable for on-site, duty, and inspection roles.

---

## 6.4 Example 4: Checking if Disk Usage is Excessively High

### Script
    #!/bin/bash

    usage=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

    echo "Root partition usage: ${usage}%"

    if [ $usage -ge 80 ]; then
        echo "Disk usage is too high."
    else
        echo "Disk usage is normal."
    fi

### Explanation
- `df -h /`: Displays information about the root partition.
- `awk 'NR==2 {print $5}'`: Extracts the fifth column from the second line.
- `sed 's/%//': Removes percent signs.

### Interview Value
A typical basic inspection script question.

---

## 6.5 Example 5: Batch Pinging Hosts

### Script
    #!/bin/bash

    for ip in $(cat ip_list.txt)
    do
        ping -c 1 $ip >/dev/null 2>&1

        if [ $? -eq 0 ]; then
            echo "$ip is reachable."
        else
            echo "$ip is unreachable."
        fi
    done

### Explanation
- `$?`: The return code of the previous command.
- `0` usually indicates success.
- `>/dev/null 2>&1`: Suppresses ping output.

### Interview Value
Very common and suitable for basic positions.

---

## 6.6 Example 6: Checking if a Service Has Processes

### Script
    #!/bin/bash

    process_count=$(ps -ef | grep nginx | grep -v grep | wc -l)

    if [ $process_count -gt 0 ]; then
        echo "nginx processes exist."
    else
        echo "nginx processes do not exist."
    fi

### Explanation
- `grep -v grep`: Excludes the line containing the grep command itself.
- `wc -l`: Counts the number of matching lines.

### Interview Value
A common task in basic service troubleshooting.

---

## 6.7 Example 7: Identifying Log Files Larger than 1GB

### Script
    #!/bin/bash

    echo "Log files larger than 1GB:"
    find /var/log -type f -name "*.log" -size +1G

### Functions
- Locate large log files.
- Identify potential disk usage issues.

### Interview Value
Useful for solving problems like "how to check when the disk is full."

---

## 7. How to Explain the Scripts You Wrote During Interviews

When explaining scripts, follow this sequence:

1. State the purpose of the script.
2. Describe the core commands used.
3. Explain the logic behind the judgments made.
4. Discuss the applicable scenarios.

---

### Example Explanation
"This script is primarily used for monitoring log exceptions. It first uses grep to count ERROR entries, stores the results in a variable, and then checks if the count is greater than zero using an if statement to display the outcome. This approach is suitable for basic log inspections and scheduled task verifications."

---

## 8. Common Mistakes

### 8.1 Forgetting to Sort Before Using `uniq -c`
Correct:
    awk '{print $7}' access.log | sort | uniq -c

---

### 8.2 Missing Spaces Around Brackets in `if` Statements
Correct:
    if [ $count -gt 0 ]; then

Incorrect:
    if [$count -gt 0]; then

---

### 8.3 Including Spaces When Assigning Variable Values
Correct:
    count=10

Incorrect:
