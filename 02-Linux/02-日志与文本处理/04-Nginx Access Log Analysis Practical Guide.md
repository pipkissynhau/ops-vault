# 04-Nginx Access Log Analysis Practical Guide

#Linux #Nginx #LogAnalysis #accesslog #awk #grep #sort #uniq #HttpStatusCode #Transport #SRE

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/04-Nginx Access Log Analysis Practical Guide.md

---

## One: Document Explanation

This article organizes common practical commands for analyzing Nginx access logs.

This article focuses on:

- Nginx access.log Basic Structure
- Viewing Nginx Log Path
- Viewing Nginx Log Format
- Statistics on Access Volume
- Statistics on Most Frequent IPs
- Statistics on Status Code Distribution
- Filtering 4xx/5xx Requests
- Analyzing 502/504 Issues
- Statistics on Most Frequent URLs
- Statistics on Requests from a Specific IP
- Statistics on Access Volume for a Specific Interface
- Filtering Logs by Time Period
- Statistics by Minute
- Slow Request Analysis
- User-Agent Analysis
- Abnormal Source IP Analysis
- Joint Troubleshooting of Nginx access.log and error.log

This article is the 04th in the Log and Text Processing series, mainly solving:

```text
How to use grepI don't know.awkI don't know.sortI don't know.uniqI don't know.wc Wait for command analysis. Nginx access.log
```

The goal is:

Be able to understand Nginx access.log fields

→ Be able to statistics access volume, status codes, URLs, IPs

→ Be able to filter 4xx/5xx abnormal requests

→ Be able to locate the direction of 502/504 issues

→ Be able to analyze sudden increases in access volume and abnormal source IPs

→ Be able to jointly troubleshoot backend service anomalies with error.log

---

## Two: Overall Approach to Nginx Log Analysis

Nginx log analysis should not start by only checking `error`.

Recommended order:

```text
Confirm. access.log Path

→ Confirm. log_format Log Format

→ View Recent Access Log

→ Statistics of total requests

→ Statistical status code distribution

→ Filter 4xx / 5xx

→ Most statistical visits IP

→ Most statistical visits URL

→ Reduce the scope by period

→ Combined error.log Analyzing anomalies

→ Continue checking backend services, ports, networks, application logs as necessary
```

Common analysis directions:

```text
Whether the number of visits surged

Whether the error rate rises

Is it a lot? 404

Is it a lot? 500 / 502 / 503 / 504

Is it something? IP It's a lot of requests.

Is there an extraordinary number of interfaces?

Is there an anomaly in the concentration of time?

Whether to Backend upstream Timeout

Whether backend connection failed

Is the request time-consuming unusual?
```

---

## Three: Common Format of Nginx access.log

---

## 1. Common combined log format

Common Nginx access.log example:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123 "-" "Mozilla/5.0"
```

When split by space, the common fields are approximately:

```text
$1
→ Client IP

$4
→ First half of the request, for example [25/Apr/2026:10:00:01

$5
→ Time zone, for example +0800]

$6
→ Method of request, e.g. "GET

$7
→ Request URLfor example /api/login

$8
→ HTTP Version of protocol, for example HTTP/1.1"

$9
→ HTTP Status code, for example 200

$10
→ Response Size

$11
→ Referer

$12 and follow-up
→ User-Agent, probably containing spaces
```

Note:

```text
Field location depends Nginx log_format

Don't be blind when you don't understand the log format. $7I don't know.$9

We have to do this first. head or tail View Line Logs
```

---

## 2. Viewing Nginx Log Path

Common paths:

```text
/var/log/nginx/access.log

/var/log/nginx/error.log
```

Viewing:

```bash
ls -lh /var/log/nginx/
```

Viewing access.log:

```bash
tail -n 20 /var/log/nginx/access.log
```

Viewing error.log:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## 3. Viewing Nginx Log Format

Viewing main configuration:

```bash
grep -n "log_format" /etc/nginx/nginx.conf
```

Recursive search:

```bash
grep -R "log_format" /etc/nginx/
```

Viewing access_log configuration:

```bash
grep -R "access_log" /etc/nginx/
```

Viewing error_log configuration:

```bash
grep -R "error_log" /etc/nginx/
```

---

## Four: Basic Access Volume Statistics

---

## Scenario 1: Viewing total lines in access.log

### Command

```bash
wc -l /var/log/nginx/access.log
```

### Explanation

Each line typically represents a request.

This command can roughly count the total number of requests in the current log file.

Note:

```text
This number is not requested the same day if the log files are multiple days away Volume
```

---

## Scenario 2: Viewing latest 100 access log entries

### Command

```bash
tail -n 100 /var/log/nginx/access.log
```

### Applicable Scenario

```text
View Recent Requests

Confirm Log Format

See if there is. 4xx / 5xx

Post-publication observation visits
```

---

## Scenario 3: Real-time viewing of access logs

### Command

```bash
tail -f /var/log/nginx/access.log
```

### Applicable Scenario

```text
Post-issue observation request

Observation visits during pressure measurements

Watch if the request arrives on the interface. Nginx

Check for access.
```

---

## Scenario 4: Real-time viewing of 5xx requests

### Command

```bash
tail -f /var/log/nginx/access.log | awk '$9 >= 500 {print $0}'
```

### Applicable Scenario

```text
Watch if it happens after the release. 5xx

Watch if the error drops after failure resumes.

See if the back end is abnormal at pressure.
```

---

## Five: Status Code Analysis

---

## Scenario 5: Statistics on status code distribution

### Command

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

### Approach

```text
awk '{print $9}'
→ Extract Status Code

sort
→ Sort

uniq -c
→ Number of statistical repetitions

sort -nr
→ In numerical order
```

### Example Result

```text
12000 200
800 404
120 499
50 502
20 504
```

---

## Scenario 6: Statistics on 2xx request count

### Command

```bash
awk '$9 >= 200 && $9 < 300 {count++} END {print count}' /var/log/nginx/access.log
```

### Explanation

2xx typically indicates successful requests.

---

## Scenario 7: Statistics on 3xx request count

### Command

```bash
awk '$9 >= 300 && $9 < 400 {count++} END {print count}' /var/log/nginx/access.log
```

### Explanation

3xx typically indicates redirection.

---

## Scenario 8: Statistics on 4xx request count

### Command

```bash
awk '$9 >= 400 && $9 < 500 {count++} END {print count}' /var/log/nginx/access.log
```

### Explanation

4xx typically indicates client-side request issues or permission problems.

Common status codes:

```text
400
→ 请求格式错误

401
→ 未认证

403
→ 无权限或被拒绝

404
→ 路径不存在

429
→ 请求过多，被限流
```

---

## Scenario 9: Statistics on 5xx request count

### Command

```bash
awk '$9 >= 500 {count++} END {print count}' /var/log/nginx/access.log
```

### Explanation

5xx typically indicates server-side or backend service anomalies.

Common status codes:

```text
500
→ Backend Internal Error

502
→ Gateway connection backend failed or backend returned abnormal

503
→ Service not available

504
→ Backend response timed out
```

---

## Scenario 10: Filtering all 5xx requests

### Command

```bash
awk '$9 >= 500 {print $0}' /var/log/nginx/access.log
```

Only view latest 50 entries:

```bash
awk '$9 >= 500 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 11: Filtering 502 requests

### Command

```bash
awk '$9 == 502 {print $0}' /var/log/nginx/access.log
```

Only view latest 50 entries:

```bash
awk '$9 == 502 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 12: Filtering 504 requests

### Command

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log
```

Only view latest 50 entries:

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Six: IP Analysis

---

## Scenario 13: Statistics on most frequent IPs

### Command

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Applicable Scenario

```text
View the highest access source IP

Check out unusual access

Check for reptiles or scans

Checking a client for too many requests
```

---

## Scenario 14: Viewing top 20 most frequent IPs

### Command

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

---

## Scenario 15: Statistics on request count for a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {count++} END {print count}' /var/log/nginx/access.log
```

Alternatively:

```bash
grep "^10.0.0.5 " /var/log/nginx/access.log | wc -l
```

---

## Scenario 16: Viewing request logs for a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $0}' /var/log/nginx/access.log
```

View latest 50 entries:

```bash
awk '$1 == "10.0.0.5" {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 17: Statistics on status code distribution for a specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

### Applicable Scenario

```text
Check if a client is large 4xx

Check if a source is large. 5xx

Analyse individual user access anomalies
```

---

## Scenario 18: Statistics on 5xx source IPs

### Command

```bash
awk '$9 >= 500 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Explanation

Used to view which client IPs primarily cause 5xx errors.

---

## Scenario 19: Statistics on 404 source IPs

### Command

```bash
awk '$9 == 404 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Applicable Scenario

```text
Check the scanner.

Check the reptiles.

Check for error path request source

Checking front-end static resource path abnormal
```

---

## Seven: URL Analysis

---

## Scenario 20: Statistics on most frequent URLs

### Command

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Explanation

Commonly in Nginx logs:

```text
$7
→ URL Path
```

---

## Scenario 21: Viewing top 20 most frequent URLs

### Command

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

---

## Scenario 22: Statistics on access volume for a specific URL

### Command

```bash
awk '$7 == "/api/login" {count++} END {print count}' /var/log/nginx/access.log
```

Or:

```bash
awk '{print $7}' /var/log/nginx/access.log | grep "^/api/login$" | wc -l
```

---

## Scenario 23: Viewing request logs for a specific URL

### Command

```bash
awk '$7 == "/api/login" {print $0}' /var/log/nginx/access.log
```

---

## Scenario 24: Statistics on 5xx URL distribution

### Command

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Applicable Scenarios

```text
Which interface to locate? 5xx Most

Let's see if the failure is concentrated on a certain interface.

To determine whether or not there is a category API Call abnormal.
```

---

## Scenario 25: Statistics on 404 URL Distribution

### Command

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Applicable Scenarios

```text
Parse error path

Analyse static resources missing

Analyse scanning requests

Analyse front end route configuration
```

---

## VIII. Request Method Analysis

---

## Scenario 26: Statistics on Request Method Distribution

### Command

```bash
awk '{print $6}' /var/log/nginx/access.log | tr -d '"' | sort | uniq -c | sort -nr
```

### Notes

Common logs include:

```text
$6
→ Method of request, e.g. "GET
```

So use:

```bash
tr -d '"'
```

to remove double quotes.

---

## Scenario 27: Filtering POST Requests

### Command

```bash
awk '$6 == "\"POST" {print $0}' /var/log/nginx/access.log
```

or:

```bash
grep '"POST ' /var/log/nginx/access.log
```

---

## Scenario 28: Filtering GET Requests

### Command

```bash
grep '"GET ' /var/log/nginx/access.log
```

---

## IX. Log Analysis by Time

---

## Scenario 29: Viewing Logs for a Specific Day

### Command

```bash
grep "25/Apr/2026" /var/log/nginx/access.log
```

### Notes

Common Nginx time format:

```text
[25/Apr/2026:10:00:01 +0800]
```

---

## Scenario 30: Viewing Logs for a Specific Hour

### Command

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log
```

### Applicable Scenarios

```text
Analysis of requests within an hour of a malfunction

An hour after the check was released.

View peak visits
```

---

## Scenario 31: Viewing Logs for a Specific Minute

### Command

```bash
grep "25/Apr/2026:10:30:" /var/log/nginx/access.log
```

---

## Scenario 32: Statistics on Request Volume by Hour

### Command

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log | wc -l
```

---

## Scenario 33: Statistics on Status Code Distribution by Hour

### Command

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c | sort -nr
```

---

## Scenario 34: Statistics on 5xx Count by Hour

### Command

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log | awk '$9 >= 500 {count++} END {print count}'
```

---

## Scenario 35: Statistics on Request Volume per Minute

### Command

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c
```

### Notes

Common `$4` is like:

```text
[25/Apr/2026:10:00:01
```

`substr($4,2,17)` represents taking:

```text
25/Apr/2026:10:00
```

which aggregates by minute.

---

## Scenario 36: Statistics on Request Volume by Minute in Reverse Order

### Command

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Applicable Scenarios

```text
Find the highest number of minutes.

Position traffic surge point

Check pressure detection or unusual access times
```

---

## Scenario 37: Statistics on 5xx Count by Minute

### Command

```bash
awk '$9 >= 500 {print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

### Applicable Scenarios

```text
Positioning 5xx Most focused time

Analyzing Fault Peak Time

Compared with release, restart, expansion time lines
```

---

## X. 502 / 504 Troubleshooting

---

## Scenario 38: Statistics on 502 Count

### Command

```bash
awk '$9 == 502 {count++} END {print count}' /var/log/nginx/access.log
```

---

## Scenario 39: Statistics on 504 Count

### Command

```bash
awk '$9 == 504 {count++} END {print count}' /var/log/nginx/access.log
```

---

## Scenario 40: Viewing Latest 502 Requests

### Command

```bash
awk '$9 == 502 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 41: Viewing Latest 504 Requests

### Command

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 42: Statistics on 502 URL Distribution

### Command

```bash
awk '$9 == 502 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 43: Statistics on 504 URL Distribution

### Command

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 44: Combining with error.log to View Upstream Errors

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

## Scenario 45: Common Causes of 502

```text
Backend service not started

Backend is not listening

Nginx upstream Configure Error

Nginx Backend's dead.

Backend service actively closes the connection

Backend Process Collapse

Backend returns illegal response

DNS Parsing upstream Failed
```

Troubleshooting commands:

```bash
ss -tunlp
```

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -I http://BackendIP:Backend
```

```bash
systemctl status Backend Service Name
```

```bash
journalctl -u Backend Service Name -n 100
```

---

## Scenario 46: Common Causes of 504

```text
Backend response timed out

Backend processing slow

Database slow query

Back-end pool full

Back-end connectors pool run out.

Backend CPU / Memory / IO Pressure.

Nginx proxy_read_timeout Too short.

Poor quality of network links
```

Troubleshooting commands:

```bash
curl -v http://BackendIP:Backend
```

```bash
top
```

```bash
free -h
```

```bash
iostat -x 1 5
```

```bash
ss -antp
```

```bash
journalctl -u Backend Service Name -n 100
```

---

## XI. Slow Request Analysis

---

## Scenario 47: Prerequisites for Slow Request Analysis

If the Nginx log format includes:

```text
$request_time

$upstream_response_time
```

you can analyze request duration.

Common log_format example:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$request_time" "$upstream_response_time"';
```

Note:

```text
Different settings request_time Field positions are different

We have to confirm before analysis. log_format
```

Check the log format:

```bash
grep -R "log_format" /etc/nginx/
```

---

## Scenario 48: Filtering Requests Greater than 1 Second (Assuming Last Column is request_time)

### Command

```bash
awk '$NF > 1 {print $0}' /var/log/nginx/access.log
```

View latest 50 lines:

```bash
awk '$NF > 1 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 49: Statistics on Slow Request URLs

### Command

```bash
awk '$NF > 1 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 50: Statistics on Slow Request Source IPs

### Command

```bash
awk '$NF > 1 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 51: Statistics on Logs with Highest Request Duration

### Command

```bash
awk '{print $NF,$0}' /var/log/nginx/access.log | sort -nr | head
```

### Notes

If the last column is request duration, this command can view logs in descending order of duration.

---

## XII. User-Agent Analysis

---

## Scenario 52: Viewing Requests Containing "curl"

### Command

```bash
grep -i "curl" /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 53: Viewing Requests Containing "bot"

### Command

```bash
grep -Ei "bot|spider|crawler" /var/log/nginx/access.log | tail -n 50
```

---

## Scenario 54: Statistics on Suspected Spider Source IPs

### Command

```bash
grep -Ei "bot|spider|crawler" /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

---

## XIII. Referer Analysis

---

## Scenario 55: Viewing Referer Field

Common combined logs include:

```text
$11
→ Referer
```

View Referer distribution:

```bash
awk '{print $11}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Note:

```text
Field location depends on log format
User-Agent When containing spaces, subsequent fields are not suitable for simple space processing
```

---

## XIV. Abnormal Access Analysis

---

## Scenario 56: Viewing IPs with Abnormally High Access Volume

### Command

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

### Judgment Direction

```text
Whether to press IP

Is it a reptile?

Whether or not to scan

Whether or not to attack traffic

Whether or not an abnormal retry of a client

Whether the load balance is not correctly transmitted IP
```

---

## Scenario 57: Viewing Most Accessed URLs by a Specific IP

### Command

```bash
awk '$1 == "10.0.0.5" {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 58: Viewing 5xx URLs for a Specific IP

### Command

```bash
awk '$1 == "10.0.0.5" && $9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 59: Viewing Recent Requests for Abnormal IPs

### Command

```bash
awk '$1 == "10.0.0.5" {print $0}' /var/log/nginx/access.log | tail -n 100
```

---

## XV. Notes on Real Client IP

---

## Scenario 60: Why the IP in access.log May Not Be the Real User IP

If there is:

```text
Load Balance

CDN

WAF

Reverse Agent

Ingress Controller

API Gateway
```

before Nginx, then `$remote_addr` may be the IP of the previous proxy, not the real client IP.

Common real IP headers:

```text
X-Forwarded-For

X-Real-IP
```

---

## Scenario 61: Checking if the Log Format Records Real IP

Check log_format:

```bash
grep -R "log_format" /etc/nginx/
```

Focus on whether it includes:

```text
$http_x_forwarded_for

$http_x_real_ip

$remote_addr
```

If real IP is not recorded, subsequent analysis of access sources may be inaccurate.

---

## XVI. Combined Analysis of access.log and error.log

---

## Scenario 62: Why Combined Analysis Is Needed

access.log is suitable for answering:

```text
Who did?

Which one? URL

Return what status code

When did the request take place?

Slow request
```

error.log is suitable for answering:

```text
Why fail?

upstream Timeout

Failed to connect backend

Whether configuration is wrong

Permission denied

Whether the file does not exist

Whether the backend closes the connection early
```

Therefore, troubleshooting 5xx errors typically requires combining both.

---

## Scenario 63: Viewing 5xx Access Logs

```bash
awk '$9 >= 500 {print $0}' /var/log/nginx/access.log | tail -n 100
```

---

## Scenario 64: View Latest Errors in Nginx error.log

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 65: View upstream-related Errors

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 66: View timeout-related Errors

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 67: View connect failed-related Errors

```bash
grep -Ei "connect\(\) failed|connection refused|no live upstreams|host not found" /var/log/nginx/error.log | tail -n 100
```

---

## Seventeen. Common Comprehensive Troubleshooting Scenarios

---

## Scenario 68: User Reports Slow Access

### Troubleshooting Path

```text
Confirm if all interfaces are slow

→ See if access surged

→ View slow requests URL

→ View status code distribution

→ See if backend is timed out

→ View host resource

→ View Backend Application Log
```

### Commands

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

If the log contains time-consuming fields:

```bash
awk '$NF > 1 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 69: Large Number of 502 Errors

### Troubleshooting Path

```text
Statistics 502 Number

→ Find 502 Focus. URL

→ Look. error.log upstream Error

→ Test Backend

→ View backend service status

→ View Backend Log
```

### Commands

```bash
awk '$9 == 502 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 == 502 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -I http://BackendIP:Backend
```

---

## Scenario 70: Large Number of 504 Errors

### Troubleshooting Path

```text
Statistics 504 Number

→ Find 504 Focus. URL

→ Look. error.log timeout

→ Check backend response time

→ Check databases or rely on services

→ Check backend host resource
```

### Commands

```bash
awk '$9 == 504 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://BackendIP:Backend
```

```bash
top
```

```bash
iostat -x 1 5
```

---

## Scenario 71: Large Number of 404 Errors

### Troubleshooting Path

```text
Statistics 404 URL

→ Assess if static resources are missing

→ To determine whether to scan the request

→ To determine whether or not the front end path is a problem

→ Judge whether or not to change path after release
```

### Commands

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

```bash
awk '$9 == 404 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 72: Suspected Abnormal Access from a Specific IP

### Troubleshooting Path

```text
View the IP Request Volume

→ View the IP Visits URL

→ View the IP Status Code Distribution

→ See if reptiles or scans

→ Whether to limit flow or ban
```

### Commands

```bash
awk '$1 == "10.0.0.5" {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$1 == "10.0.0.5" {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$1 == "10.0.0.5" {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Eighteen. Production Troubleshooting Notes

---

## 1. Confirm log_format First

Do not assume directly:

```text
$1 Yes. IP

$7 Yes. URL

$9 It's a status code.
```

Although this is a common format, it's not absolute.

First check:

```bash
head -n 5 /var/log/nginx/access.log
```

Then check:

```bash
grep -R "log_format" /etc/nginx/
```

---

## 2. Statistics Should Combine Time Range

The following command stats the entire log file:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

If the log spans multiple days, the results don't represent the current issue.

When analyzing faults, limit the time range as much as possible:

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c | sort -nr
```

---

## 3. 5xx Errors Should Combine error.log

access.log only shows 502/504 responses.

The real cause is usually in:

```text
Nginx error.log

Backend service log

Backend Service Status

Backend Host Resource

Database or service-dependent
```

---

## 4. Real IP May Be Hidden by Proxy

If Nginx has a load balancer, CDN, WAF, or Ingress in front:

```text
$remote_addr Probably not the user's real. IP
```

Need to confirm if it records:

```text
X-Forwarded-For

X-Real-IP
```

---

## 5. Slow Request Analysis Must Confirm Time-consuming Fields

Do not assume `$NF` is time-consuming.

Must first confirm if `log_format` contains:

```text
$request_time

$upstream_response_time
```

And confirm the field position.

---

## 6. Do Not Only Check Nginx

Nginx is the entry layer.

If appears:

```text
502

504

A lot of overtime.

Slow Request
```

Usually need to continue checking:

```text
Backend

Backend Service Status

Backend Application Log

Database

Cache

Message queue

Network

Host CPU / Memory / IO
```

---

## Nineteen. Common Commands Summary in This Article

---

## Log Path and Format

```bash
ls -lh /var/log/nginx/
```

```bash
tail -n 20 /var/log/nginx/access.log
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -R "log_format" /etc/nginx/
```

```bash
grep -R "access_log" /etc/nginx/
```

```bash
grep -R "error_log" /etc/nginx/
```

---

## Basic Access Volume

```bash
wc -l /var/log/nginx/access.log
```

```bash
tail -n 100 /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/access.log | awk '$9 >= 500 {print $0}'
```

---

## Status Codes

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$9 >= 200 && $9 < 300 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 >= 300 && $9 < 400 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 >= 400 && $9 < 500 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 >= 500 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 >= 500 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 502 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

---

## IP Analysis

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

```bash
awk '$1 == "10.0.0.5" {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$1 == "10.0.0.5" {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$1 == "10.0.0.5" {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$9 >= 500 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## URL Analysis

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 20
```

```bash
awk '$7 == "/api/login" {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$7 == "/api/login" {print $0}' /var/log/nginx/access.log
```

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Request Methods

```bash
awk '{print $6}' /var/log/nginx/access.log | tr -d '"' | sort | uniq -c | sort -nr
```

```bash
grep '"POST ' /var/log/nginx/access.log
```

```bash
grep '"GET ' /var/log/nginx/access.log
```

---

## Time Analysis

```bash
grep "25/Apr/2026" /var/log/nginx/access.log
```

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log
```

```bash
grep "25/Apr/2026:10:30:" /var/log/nginx/access.log
```

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log | wc -l
```

```bash
grep "25/Apr/2026:10:" /var/log/nginx/access.log | awk '{print $9}' | sort | uniq -c | sort -nr
```

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c
```

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 >= 500 {print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## 502 / 504

```bash
awk '$9 == 502 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 == 504 {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$9 == 502 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -Ei "connect\(\) failed|connection refused|no live upstreams|host not found" /var/log/nginx/error.log | tail -n 100
```

---

## Slow Requests

```bash
grep -R "log_format" /etc/nginx/
```

```bash
awk '$NF > 1 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$NF > 1 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$NF > 1 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $NF,$0}' /var/log/nginx/access.log | sort -nr | head
```

---

## User-Agent / Crawlers

```bash
grep -i "curl" /var/log/nginx/access.log | tail -n 50
```

```bash
grep -Ei "bot|spider|crawler" /var/log/nginx/access.log | tail -n 50
```

```bash
grep -Ei "bot|spider|crawler" /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

---

## Twenty. One-Sentence Summary

Nginx access.log analysis core is:

```text
Confirm log format first

→ Restat state code

→ Reanalyze IP and URL

→ And reduce the problem by time frame.

→ Finally. error.log And back-end service continues to locate.
```

Common analysis objects:

```text
$1
→ Client IP

$4
→ Request Time

$6
→ Method of request

$7
→ Request URL

$9
→ HTTP Status Code

$10
→ Response Size
```

Status code analysis:

```text
2xx
→ Request successful.

3xx
→ Redirect

4xx
→ Client request or permission issue

5xx
→ Service or back-end service problems
```

5xx troubleshooting chain:

```text
access.log Statistics 5xx

→ Find out. 5xx Focus. URL And time.

→ error.log Cha. upstream / timeout / connect failed

→ Test Backend

→ View backend service status

→ View backend application logs and host resources
```

Production recommendations:

```text
Don't be unidentified. log_format direct field numbering

Not just the entire log file.

Don't just look. access.logI don't know.5xx It must be combined. error.log

If there's an agent or load balance ahead, identify the real client. End IP Record

Slow request analysis must be confirmed. request_time Field Position

Nginx Just the entrance level.502 / 504 Most of the time, we keep checking back-end services.
```