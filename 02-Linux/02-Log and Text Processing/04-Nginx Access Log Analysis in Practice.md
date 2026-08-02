# 04-Nginx Access Log Analysis in Practice

# Linux # Nginx # Log Analysis # accesslog # awk # grep # sort # uniq # HTTP Status Codes # Operations and Maintenance # SRE

---

## Recommended Path

01-Linux Basics and Host Operations and Maintenance/02-Logs and Text Processing/04-Nginx Access Log Analysis in Practice.md

---

## I. Document Description

This document compiles common practical commands for analyzing Nginx access logs.

Key points include:

- Basic structure of Nginx access.log
- Checking the path to Nginx logs
- Viewing Nginx log format
- Counting total visits
- Identifying the most frequently visited IP addresses
- Analyzing status code distribution
- Filtering 4xx/5xx requests
- Investigating 502/504 issues
- Tracking the most popular URLs
- Counting requests from specific IPs
- Monitoring access volume for particular interfaces
- Filtering logs by time period
- Calculating request volumes per minute
- Analyzing slow requests
- Examining User-Agent information
- Investigating abnormal source IP addresses
- Joint troubleshooting using Nginx access.log and error.log

This article is part of the Logs and Text Processing series, focusing on:

```text
How to use commands like grep, awk, sort, uniq, wc to analyze Nginx access.log
```

The goals are:

- Understanding the fields in Nginx access.log
- Counting visits, status codes, URLs, and IPs
- Filtering out 4xx/5xx error requests
- Identifying issues with 502/504 responses
- Analyzing sudden increases in visit volume and abnormal source IPs
- Collaborating with error.log to diagnose backend service issues
---

## II. General Approach to Nginx Log Analysis

When analyzing Nginx logs, don’t start by checking only `error`.

Recommended order:

```text
Confirm the path to access.log
→ Verify the log_format setting
→ View the latest access logs
→ Calculate total request volume
→ Analyze status code distribution
→ Filter 4xx/5xx requests
→ Identify the most frequently visited IP addresses
→ Track the most popular URLs
→ Narrow down the time frame for analysis
→ Combine with error.log to diagnose exceptions
→ If necessary, further investigate backend services, ports, networks, and application logs
```

Common areas of analysis include:

```text
- Sudden increases in visit volume
- Rising error rates
- High numbers of 404 errors
- Frequent occurrences of 500/502/503/504 errors
- Abnormally high requests from specific IPs or interfaces
- Concentrated exceptions during certain time periods
- Backend upstream timeouts
- Backend connection failures
- Abnormal request latency
```

---

## III. Common Nginx Access Log Formats

---

## 1. Common Combined Log Format

Example of a typical Nginx access log:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123 "-" "Mozilla/5.0"
```

When fields are separated by spaces, the common ones include:

```text
$1
→ Client IP address

$4
→ First half of the request time, e.g., [25/Apr/2026:10:00:01

$5
→ Time zone, e.g., +0800]

$6
→ Request method, e.g., "GET"

$7
→ Request URL, e.g., /api/login

$8
→ HTTP protocol version, e.g., HTTP/1.1

$9
→ HTTP status code, e.g., 200

$10
→ Response body size

$11
→ Referer

$12 and subsequent fields
→ User-Agent information, which may contain spaces
```

Note:

```text
The position of these fields depends on the Nginx log_format setting.
Do not blindly apply $7 or $9 without understanding the log format.
Always check the first few lines of the logs using head or tail before analysis.
```

---

## 2. Checking the Path to Nginx Logs

Common locations:

```text
/var/log/nginx/access.log

/var/log/nginx/error.log
```

To view them:

```bash
ls -lh /var/log/nginx/
```

To view access.log:

```bash
tail -n 20 /var/log/nginx/access.log
```

To view error.log:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## 3.### Use Cases

```text
Viewing the IP with the highest number of visits

Troubleshooting abnormal visits

Detecting crawlers or scans

Identifying excessive requests from a specific client
```

---

## Scenario 14: Viewing the top 20 IPs with the most visits

### Command

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

---

## Scenario 15: Counting the number of requests from a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {count++} END {print count}' /var/log/nginx/access.log
```

Alternatively:

```bash
grep "^10.0.0.5 " /var/log/nginx/access.log | wc -l
```

---

## Scenario 16: Viewing the access logs for a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $0}' /var/log/nginx/access.log
```

To view the last 50 entries:

```bash
awk '$1 == "10.0.0.5" {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 17: Analyzing the status code distribution for a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $9}' /var/log/nginx.access.log | sort | uniq -c | sort -nr
```

### Use Cases

```text
Determining if a particular client is experiencing many 4xx errors

Identifying whether a specific source is causing numerous 5xx errors

Analyzing abnormal visits from individual users
```

---

## Scenario 18: Counting IP addresses with 5xx status codes

### Command

```bash
awk '$9 >= 500 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Explanation

This command helps identify which client IPs are primarily responsible for 5xx errors.

---

## Scenario 19: Counting IP addresses with 404 status codes

### Command

```bash
awk '$9 == 404 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Use Cases

```text
Detecting scanners

Identifying crawlers

Determining the sources of requests for incorrect paths

Analyzing issues with frontend static resource paths
```

---

## VII. URL Analysis

---

## Scenario 20: Counting the most visited URLs

### Command

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Explanation

In common Nginx logs:

```text
$7
→ URL path
```

---

## Scenario 21: Viewing the top 20 most visited URLs

### Command

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

---

## Scenario 22: Counting the number of visits to a specific URL

### Command

```bash
awk '$7 == "/api/login" {count++} END {print count}' /var/log/nginx/access.log
```

Or:

```bash
awk '{print $7}' /var/log/nginx/access.log | grep "^/api/login$" | wc -l
```

---

## Scenario 23: Viewing the access logs for a specific URL

### Command

```bash
awk '$7 == "/api/login" {print $0}' /var/log/nginx.access.log
```

---

## Scenario 24: Counting the distribution of 5xx URLs

### Command

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Use Cases

```text
Identifying which interface experiences the most 5xx errors

Determining if failures are concentrated on a specific interface

Analyzing abnormalities in certain types of API calls
```

---

## Scenario 25: Counting the distribution of 404 URLs

### Command

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Use Cases

```text
Analyzing incorrect paths

Identifying missing static resources

Detecting scan requests

Investigating issues with frontend routing configurations
```

---

## VIII. Request Method Analysis

---

## Scenario 26: Count## Scenario 41: View the Latest 504 Requests

### Command

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 42: Statistically Analyze the Distribution of 502 URLs

### Command

```bash
awk '$9 == 502 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 43: Statistically Analyze the Distribution of 504 URLs

### Command

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 44: Combine with error.log to Check for Upstream Errors

### Command

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

Common keywords:

```text
upstream timed out

connect() failed

connection refused

no live upstreams

upstream prematurely closed connection

host not found in upstream
```

---

## Scenario 45: Common Causes of 502 Errors

```text
The backend service is not running.

The backend port is not listening.

There is an error in the Nginx upstream configuration.

The network between Nginx and the backend is unavailable.

The backend service actively closes the connection.

The backend process crashes.

The backend returns an invalid response.

DNS resolution for the upstream server fails.
```

Troubleshooting commands:

```bash
ss -tunlp

nc -zv -w 2 backend_IP backend_port

curl -I http://backend_IP:backend_port

systemctl status backend_service_name

journalctl -u backend_service_name -n 100
```

---

## Scenario 46: Common Causes of 504 Errors

```text
The backend response times out.

The backend processes requests slowly.

The database performs slow queries.

The backend thread pool is full.

The backend connection pool is exhausted.

The backend experiences high CPU/memory/I/O loads.

The Nginx proxy_read_timeout setting is too short.

The network link quality is poor.
```

Troubleshooting commands:

```bash
curl -v http://backend_IP:backend_port

top

free -h

iostat -x 1 5

ss -antp

journalctl -u backend_service_name -n 100
```

---

## Chapter XI: Analysis of Slow Requests

---

## Scenario 47: Prerequisites for Analyzing Slow Requests

If the Nginx log format includes the following fields:

```text
$request_time

$upstream_response_time
```

it is possible to analyze the request duration.

Common examples of `log_format`:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$request_time" "$upstream_response_time";
```

Note:

```text
The position of the `$request_time` field may vary depending on the environment.

Always confirm the `log_format` before starting the analysis.
```

To view the log format:

```bash
grep -R "log_format" /etc/nginx/
```

---

## Scenario 48: Assuming the Last Column Represents `request_time`, Filter Requests That Take More Than 1 Second

### Command

```bash
awk '$NF > 1 {print $0}' /var/log/nginx/access.log
```

To view the latest 50 requests:

```bash
awk '$NF > 1 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 49: Statistically Analyze URLs That Result in Slow Requests

### Command

```bash
awk '$NF > 1 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 50: Statistically Analyze the Source IPs of Slow Requests

### Command

```bash
awk '$NF > 1 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 51: Statistically Analyze Logs with the Highest Request Durations

### Command

```bash
awk '{print $NF,$0}' /var/log/nginx/access.log | sort -nr | head
```

### Explanation

If the last column represents the request duration, this command will display the logs in descending order of time.

---

## Chapter XII: Analysis of User-Agent Information

---

## Scenario 52: View Requests That Contain "curl"

### Command

```## Scenario 64: Check the latest errors in Nginx's error.log

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 65: Check for upstream-related errors

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 66: Check for timeout-related errors

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 67: Check for connect failed-related errors

```bash
grep -Ei "connect\(\) failed|connection refused|no live upstreams|host not found" /var/log/nginx/error.log | tail -n 100
```

---

## Section 17: Common Comprehensive Troubleshooting Scenarios

---

## Scenario 68: Users report slow access

### Troubleshooting Steps

```text
Confirm if all interfaces are experiencing slow performance.

→ Check for sudden increases in traffic.

→ Identify slow request URLs.

→ Analyze the distribution of status codes.

→ Verify if there are timeouts on the backend.

→ Assess host resources.

→ Examine backend application logs.
```

### Commands to Use

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx.access.log | sort | uniq -c | sort -nr | head
```

If the log contains a time-consuming field:

```bash
awk '$NF > 1 {print $7}' /var/log/nginx/access.log | sort |uniq -c | sort -nr | head
```

---

## Scenario 69: A large number of 502 errors occur

### Troubleshooting Steps

```text
Count the number of 502 errors.

→ Identify URLs where these errors are concentrated.

→ Check for upstream errors in the error.log.

→ Test the backend ports.

→ Verify the status of backend services.

→ Examine backend logs.
```

### Commands to Use

```bash
awk '$9 == 502 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 == 502 {print $7}' /var/log/nginx acess.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
nc -zv -w 2 backend_ip backend_port
```

```bash
curl -I http://backend_ip:backend_port
```

---

## Scenario 70: A large number of 504 errors occur

### Troubleshooting Steps

```text
Count the number of 504 errors.

→ Identify URLs where these errors are concentrated.

→ Check for timeouts in the error.log.

→ Verify the backend response time.

→ Examine the database or dependent services.

→ Assess backend host resources.
```

### Commands to Use

```bash
awk '$9 == 504 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 == 504 {print $7}' /var/log/nginx.access.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://backend_ip:backend_port
```

```bash
top
```

```bash
iostat -x 1 5
```

---

## Scenario 71: A large number of 404 errors occur

### Troubleshooting Steps

```text
Identify URLs that result in 404 errors.

→ Determine if static resources are missing.

→ Check for crawl requests.

→ Investigate potential front-end routing issues.

→ Verify if the path has changed since deployment.
```

### Commands to Use

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

```bash
awk '$9 == 404 {print $1}' /var/log/nginx.access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 72: Suspicion of abnormal access from a specific IP

### Troubleshooting Steps

``````bash
awk '$9 >= 500 {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```4xx
→ Client request or permission issues

5xx
→ Server or backend service problems

5xx troubleshooting steps:

```text
Use access.log to analyze 5xx errors

→ Identify URLs and time periods where 5xx errors occur frequently

→ Check error.log for upstream errors, timeouts, or connection failures

→ Test the backend ports

→ Verify the status of the backend services

→ Review the backend application logs and system resources
```

Production recommendations:

```text
Do not assign field numbers directly without confirming the log_format.

Do not simply analyze the entire log file; consider the specific time range of the issue.

Do not rely solely on access.log; error.log is essential for diagnosing 5xx errors.

If there are proxies or load balancers in between, ensure that the real client IP addresses are being recorded.

When analyzing slow requests, make sure to confirm the position of the request_time field.

Since Nginx serves only as an entry point, most 502/504 errors still require further investigation of the backend services.
```