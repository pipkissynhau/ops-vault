# 08-Production Log Governance: Level Standards, Desensitization, Sampling, Retention Periods, and Cost Control

#Linux #LogGovernance #LogCode #LogicDesensitization #LogSample #LogRetentionCycle #LogCostControl #SRE #Observation #StableGovernance

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/08-Production Log Governance: Level Standards, Desensitization, Sampling, Retention Periods, and Cost Control.md

---

## I. Document Overview

This document organizes core principles and implementation methods for log governance in production environments.

The previous 01-07 articles have covered:

```text
Log View

grep / awk / sed / find / xargs

Nginx access.log Analysis

Log Archive and logrotate

JSON Log processing

mail Announcements

rsyslog Forward

Log platform collection, fields, labels and indexing designs
```

This article further focuses on more critical issues in production environments:

- How to standardize log levels
- What content should be logged
- What content should not be logged
- Why long-term debug logs are not allowed in production
- How to desensitize logs
- Which sensitive fields must not be output in plain text
- How to sample high-frequency logs
- How to reduce noise in health check logs
- How to design log retention periods
- How to control log platform costs
- How to troubleshoot log volume spikes
- How to establish log governance standards and closed-loop processes

The goal is:

To understand that log governance is not simply "logging more"

→ To establish production log level standards

→ To identify sensitive logs and high-risk logs

→ To design log desensitization rules

→ To control high-frequency low-value logs

→ To design retention periods based on log value

→ To govern log platforms from a cost perspective

→ To develop SRE log governance and stability construction thinking

---

## II. Why Production Log Governance is Needed

Logs are crucial for troubleshooting, but more logs are not always better.

Untamed logs can cause many problems:

```text
Logic growth.

The disk is full.

Log platform index expansion

Query is getting slow.

There's been a lot of noise.

The real malfunction is flooded.

Sensitive information leaks

Increased audit risk

Log costs are out of control.

I don't want to read the journal.
```

The problems that production log governance aims to solve are:

```text
You have to fight in your log.

Don't hit the wrong log.

Sensitive information cannot be explicitly struck.

High-frequency low-value logs need to be down.

Key error log to search

Log retention cycle to layer

Log platform costs are manageable
```

One-sentence understanding:

```text
The goal of log governance is not to reduce the log, but to increase its value intensity.
```

---

## III. General Principles of Production Log Governance

Production log governance should follow these principles:

```text
It works.

Correct

Structure

Retrievable

Associated

Call the police.

Auditable

Controlled costs

Do not disclose sensitive information
```

Further breakdown:

```text
It works.
→ Logs help with handicaps, auditing, analysis and rediscovery

Correct
→ Time, level, service name, status code, error information

Structure
→ Use as much as possible JSON Or unify fields

Retrievable
→ I can press it. serviceI don't know.envI don't know.trace_idI don't know.level Question

Associated
→ Yes. trace_id / request_id Serial Request Link

Call the police.
→ errorI don't know.5xxI don't know.timeoutI don't know.OOM Waiting for statistics and alarms.

Auditable
→ Retroactive login, privileges, configuration changes, etc.

Controlled costs
→ Don't let meaningless logs blow up the platform.

Do not disclose sensitive information
→ Password,tokenI don't know.CookieKeys, etc. cannot explicitly enter the log
```

---

## IV. Log Level Standards

---

## Scenario 1: Common Log Levels

Common log levels in production:

```text
DEBUG
→ Debug Information

INFO
→ Normal running information

WARN
→ Potential risks or recoverable anomalies

ERROR
→ Clear error. Needing attention.

FATAL
→ Serious error, process or service may not be available
```

They also commonly appear in lowercase format:

```text
debug

info

warn

error

fatal
```

It is recommended to unify a style.

For example, unify to lowercase:

```json
{
  "level": "error"
}
```

Not recommended to mix:

```text
ERROR

Error

error

ERR
```

Mixing can lead to:

```text
Retrieving Unharmonized

Inaccuracy of statistics

The rules are complicated.

Catalogue field confusion
```

---

## Scenario 2: What Should DEBUG Logs Record

DEBUG is suitable for recording:

```text
Debug Process Information

Develop Quest Details

Context for interim positioning issues

Low-level variable information

Internal process details
```

Production recommendations:

```text
Default closure of the production environment DEBUG

Time to open. DEBUG Must have Time Window

Close as soon as problem position is completed DEBUG

DEBUG Log cannot contain sensitive information
```

Not recommended to keep DEBUG enabled long-term in production:

```text
Full Request Parameters DEBUG

Full SQL DEBUG

Full Response DEBUG

Full circulation process DEBUG

HF debug Hit some.
```

Risks:

```text
Logs are booming.

Disk full.

Increased cost of log platform

Sensitive data leaks

Application performance decline
```

---

## Scenario 3: What Should INFO Logs Record

INFO is suitable for recording normal key events.

For example:

```text
Service started successfully

Service stopped

Configure Load Finished

Time job starts and ends

Critical business process completion

Key external dependence initialized successfully

Version number and summary of start-up parameters

Summary of mandate performance
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "info",
  "service": "order-api",
  "env": "prod",
  "message": "order payment callback processed",
  "trace_id": "abc123",
  "order_id": "order-001",
  "duration_ms": 35
}
```

INFO requirements:

```text
It helps understand how the system is working.

Not too high.

We can't take every step of the way. INFO

Can not include sensitive information
```

---

## Scenario 4: What Should WARN Logs Record

WARN indicates:

```text
There's been an anomaly.

But the system still works.

Needing attention, but not necessarily immediate failure
```

Suitable for recording:

```text
Slower interface response

External dependence happens to be overtime and successful.

Cache failure rate abnormal.

Connection pool approaching limit

Disk Space Approaching Threshold

Use default for configuration items

Failure of non-critical missions

Degrade logic trigger

Limit flow trigger
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "warn",
  "service": "user-api",
  "env": "prod",
  "message": "redis request timeout, fallback to local cache",
  "trace_id": "abc123",
  "duration_ms": 1200
}
```

WARN value:

```text
Early exposure risk

Supporting trend analysis

Auxiliary detection of signs of failure
```

---

## Scenario 5: What Should ERROR Logs Record

ERROR indicates:

```text
Clear error occurred

Request failed or mission failed

We need to check or report.
```

Suitable for recording:

```text
Interface returns 5xx

Database connection failed

Redis Connection failed

External interface call failed

Mission failed

Message consumption failed

File writing failed

Permission verification anomaly

Business critical processes fail
```

ERROR logs must include:

```text
Error occurred

Service Name

Environment

Error Type

Error message

trace_id / request_id

Key Context

Unusual Stack

Failed dependency or interface
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "error",
  "service": "payment-api",
  "env": "prod",
  "trace_id": "abc123",
  "request_id": "req-001",
  "error_type": "DatabaseTimeout",
  "message": "database query timeout",
  "duration_ms": 3000
}
```

---

## Scenario 6: What Should FATAL Logs Record

FATAL indicates:

```text
Service cannot continue

Core dependence not available

Process is about to quit

The system is in critical state of non-availability.
```

Suitable for recording:

```text
Service startup failed

Core Configuration Missing

Database initialization failed

Required certificate does not exist

Port binding failed

Process is about to quit

The non-availability of critical components has made service unstartable.
```

FATAL should generally trigger:

```text
High priority alert.

Automatically pull up or restart

Manual intervention

Fault Rewind
```

---

## V. Common Log Level Usage Errors

---

## Scenario 7: Logging All Normal Requests as ERROR

Error example:

```text
User login failed, password error → ERROR

User access does not exist → ERROR

Normal Parameter Validation Failed → ERROR
```

More reasonable:

```text
User password error
→ INFO or WARNSubject to operational security requirements

Access does not exist
→ INFO or WARN

Parameter verification failed
→ INFO or WARN
```

Reason:

```text
These are usually business foreseeable behaviours.

I can't. ERROR Drowned by normal business noise.
```

---

## Scenario 8: Logging True Errors as INFO

Error example:

```text
Database connection failed → INFO

Payback processing failed → INFO

Message consumption failed → INFO
```

Problem:

```text
Log platform alarm unidentified

The error rate is inaccurate.

It's easy to miss out.
```

More reasonable:

```text
Database connection failed
→ ERROR

Payback processing failed
→ ERROR

Message consumption failed
→ ERROR
```

---

## Scenario 9: Keeping DEBUG Enabled Long-Term in Production

Error approach:

```text
Long-term production environment DEBUG

All SQL Print Both

Print all requests

Print all responses

There's plenty in the cycle. debug
```

Risks:

```text
Logs are booming.

Disk full.

Slower interface

Sensitive data leaks

Increased cost of log platform
```

Production recommendations:

```text
Default INFO

Temporary DEBUG There's gotta be someone in charge.

Temporary DEBUG Time to close.

Temporary DEBUG To record changes

Positioning complete immediate recovery.
```

---

## VI. What Production Logs Should Record

---

## Scenario 10: Application Startup Logs

Recommended to record:

```text
Service Name

Environment

Version Number

Start Time

Listen Port

Configure Summary

Reliance on initialization results

Successful startup
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "info",
  "service": "resume-api",
  "env": "prod",
  "version": "v1.2.3",
  "message": "service started",
  "port": 8080
}
```

---

## Scenario 11: Request Entry Logs

Recommended to record:

```text
trace_id

request_id

client_ip

method

path

status_code

duration_ms

request_size

response_size

user_agent

service

env
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "info",
  "service": "resume-api",
  "env": "prod",
  "trace_id": "abc123",
  "method": "GET",
  "path": "/api/resumes",
  "status_code": 200,
  "duration_ms": 42,
  "client_ip": "10.0.0.5"
}
```

---

## Scenario 12: Dependency Call Logs

Recommended to record:

```text
Dependency name

Chile

Call Method

Call Time

Call Results

Error Reason

Number of retries

trace_id
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "warn",
  "service": "order-api",
  "env": "prod",
  "trace_id": "abc123",
  "dependency": "redis",
  "operation": "GET",
  "duration_ms": 800,
  "message": "redis call slow"
}
```

---

## Scenario 13: Task Execution Logs

Suitable for scheduled tasks, batch processing tasks, and backup tasks.

Recommended to record:

```text
Task Name

Start Time

End Time

Time-consuming execution

Number processed

Number of success

Number of failures

Reason for Failure

Execute Nodes
```

Example:

```json
{
  "timestamp": "2026-04-25T02:00:00+08:00",
  "level": "info",
  "service": "archive-job",
  "env": "prod",
  "job": "log_archive",
  "processed_count": 120,
  "failed_count": 0,
  "duration_ms": 8500,
  "message": "archive job finished"
}
```

---

## Scenario 14: Error Logs

Error logs should include:

```text
Error Type

Error message

Unusual Stack

trace_id

request_id

Non-sensitive identification of users or resources

Dependency name

Failure phase

Time consuming

Whether to try again

Dismissed
```

Not recommended to only write:

```text
error happened
```

Such logs have no troubleshooting value.

---

## VII. What Production Logs Should Not Record

---

## Scenario 15: Prohibited Plain Text Recording of Sensitive Information

Production logs should not record sensitive information in plain text:

```text
Password

Confirm password

Authentication Code

Text Authentication Code

Mailbox authentication code

token

refresh_token

access_token

JWT

Cookie

Session ID

Authorization Header

SecretKey

AccessKey

Private Key

Certificate private key

Bank card number

ID number

Phone number, full number.

Full address of the mailbox

Detailed home address

Database Connection Password

Third parties API Key
```

---

## Scenario 16: Not Recommended to Fully Record Request Bodies

Risks:

```text
Request may contain a password

Could contain token

The request may contain ID, cell number, address.

The request may be large.

High and high.

Possible breaches of privacy and security requirements
```

More reasonable approach:

```text
Record only necessary fields

Sensitive field dissensitisation

Big field break

Only test environmental record complete request Body

Approval and time limits for temporary opening of production
```

---

## Scenario 17: Not Recommended to Fully Record Response Bodies

Risks:

```text
The response may contain user privacy

The response may contain internal data

The response could be very large.

Increased cost of log platform

There was too much noise in the queue.
```

More reasonable approach:

```text
Record status code

Time-consuming record

Record error code

Summary of recorded error information

Record response size if necessary

Do not record a complete response
```

---

## Scenario 18: Not Recommended to Record Full SQL Parameters

Risks:

```text
Could include user privacy

Possible inclusion of business sensitive data

It's a big log.

SQL It could be too long.

Security risk
```

Recommendation:

```text
Make default not print complete SQL Parameters

Send slow query to database slow log

Apply side logs only SQL Type, time-consuming, error summary

Short time to screen. SQL debug
```

---

## VIII. Log Desensitization Design

---

## Scenario 19: What is Log Desensitization

Log desensitization refers to:

```text
Hide, cut, Hash or delete sensitive fields before log writing or during collection
```

Common processing methods:

```text
Shade

Cut

Hash.

Delete Fields

Keep the last few

Retain field if it exists, do not retain field content
```

---

## Scenario 20: Common Desensitization Methods

### Phone Number Desensitization

```text
13812345678
→ 138****5678
```

### ID Number Desensitization

```text
620102199001011234
→ 620102********1234
```

### Email Desensitization

```text
user@example.com
→ u***@example.com
```

### Token Desensitization

```text
Bearer eyJhbGciOiJIUzI1Ni...
→ Bearer ***
```

### Bank Card Desensitization

```text
6222021234567890123
→ 622202*********0123
```

---

## Scenario 21: Simple Phone Number Desensitization Using sed

Example file:

```bash
vi app.log
```

Example content:

```text
user phone=13812345678 login success
```

Desensitization command:

```bash
sed -E 's/([0-9]{3})[0-9]{4}([0-9]{4})/\1****\2/g' app.log
```

Explanation:

```text
Only for simple text processing

Complex scenes should be collected from the application log output layer or log processing layer
```

---

## Scenario 22: Using sed to Desensitize Authorization

```bash
sed -E 's/(Authorization: Bearer )[A-Za-z0-9._-]+/\1***/g' app.log
```

---

## Scenario 23: Using jq to Remove Sensitive Fields

Assuming JSON logs:

```json
{
  "user": "test",
  "password": "123456",
  "token": "abcdef",
  "message": "login"
}
```

Remove sensitive fields:

```bash
jq 'del(.password, .token)' app.json
```

---

## Scenario 24: Using jq to Replace Sensitive Fields

```bash
jq '.password="***" | .token="***"' app.json
```

---

## Scenario 25: Log Platform Collection Layer Desensitization

Common locations:

```text
Filebeat processor

Logstash filter

Fluent Bit filter

Vector transform

Apply Log SDK

Gateway log plugin
```

Recommended priority:

```text
Apply output layer desensitization

→ Collection Agent Second Protection

→ Filter Before Log Platform Index

→ Query Permission Control
```

Do not rely solely on query-side hiding, as sensitive logs may already be stored in the database.

## 9. Log Sampling Design

---

## Scenario 26: Why Log Sampling is Needed

Log sampling is used to control high-frequency, low-value logs.

Typical scenarios:

```text
Health check log

Static resource requests

Repeat Same Error

HF debug

Frequent requests for success

reptile access

Query Interface

Heart beat log
```

If not sampled, it will lead to:

```text
Logs are booming.

Index cost increases

Slowness of queries

The real anomaly is flooded.

Collection Agent The pressure's getting bigger.
```

---

## Scenario 27: Common Sampling Methods

Common methods:

```text
Proportional sampling

Sample by Time Window

Sample by status code

Sample by log level

Sample by interface path

By user or IP Sample

Only wrong logs, not successful logs.

Successful log sample, error log full
```

Recommended approach:

```text
ERROR Full retention

WARN Most reservations

INFO Selective reservations

DEBUG Production Default Not

Health screening and static resources can be filtered or sampled in low proportions
```

---

## Scenario 28: Noise Reduction for Health Check Logs

Common paths for health check logs:

```text
/health

/healthz

/ready

/readiness

/liveness

/metrics
```

If performed every few seconds, the log volume will be extremely large.

Nginx analysis of health check count:

```bash
awk '$7 == "/health" {count++} END {print count}' /var/log/nginx/access.log
```

Statistics of status codes after excluding health checks:

```bash
awk '$7 != "/health" {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

Excluding multiple health check paths:

```bash
awk '$7 !~ /^\/(health|healthz|ready|readiness|liveness|metrics)$/ {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Scenario 29: Noise Reduction for Static Resource Logs

Common suffixes for static resources:

```text
.css

.js

.png

.jpg

jpeg

gif

ico

svg

woff

woff2
```

Statistics of static resource requests:

```bash
awk '$7 ~ /\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/ {count++} END {print count}' /var/log/nginx/access.log
```

Statistics of URLs after excluding static resources:

```bash
awk '$7 !~ /\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/ {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 30: Do Not Sample Error Logs

Not recommended to sample:

```text
ERROR

FATAL

Security Audit Log

Login Failed Login

Permission Change Log

Pay Failure Log

Order Critical Status Change Log

Data Delete Log
```

Reasons:

```text
These logs are very valuable.

Sampling may result in the loss of critical evidence

Impact crash and audit
```

---

## 10. Log Retention Period Design

---

## Scenario 31: Tiered Retention Based on Log Value

Log retention periods should not be one-size-fits-all.

Recommended tiering:

```text
High Value Log
→ Keep it longer.

Low Value Log
→ Keep Short

Low-value HF logs
→ Sample or Filter

Audit log
→ Retention as required

Local Log
→ Short-term reservations

Log Platform
→ Medium-term reservation

Object Storage
→ Long-term Archive
```

---

## Scenario 32: Common Retention Period Examples

```text
Application log on-board
→ 7 Present. 15 days

Here. Nginx access.log
→ 7 Present. 15 days

Here. error.log
→ 15 Present. 30 days

Log Platform Thermal Data
→ 7 Present. 15 days

Log platform temperature data
→ 30 Present. 90 days

Object Storage Archive
→ 90 Present. 180 Days or longer

Security Audit Log
→ Retention as required by the enterprise
```

---

## Scenario 33: Retention Recommendations for Different Log Types

```text
Apply Error Log
→ 30 above sky

Interface Access Log
→ 7 Present. 30 days

Audit log
→ 90 More than day or compliance

debug Log
→ Production Default Not Retain

Health check log
→ As little as possible or short-term reservations

Security Log
→ Long-term reservations

Deployment of change logs
→ Long-term reservations

System Error Log
→ 30 above sky
```

---

## Scenario 34: Relationship Between Local Logs and Platform Logs

Local logs are suitable for:

```text
Near-real-time barrier

Service startup failed check

Log platform under abnormal times

Short-term problem positioning
```

Log platforms are suitable for:

```text
Cross Machine Query

Cross-service linkages

Long-term statistics

Police!

Audit

Rewind
```

Object storage archival is suitable for:

```text
Low frequency query

Compliance reservations

Long Backup

Reduce heat storage costs
```

---

## 11. Log Cost Control

---

## Scenario 35: Where Does Log Cost Come From

Log platform costs typically come from:

```text
Collection Agent Resources

Network Transfer

Message queue

Index writing

Disk Storage

Copy Storage

Query Calculator

Cold thermal layer

Backup Archive

Transport management
```

Elasticsearch-like platform costs mainly concentrate on:

```text
Index

Storage

Question

Cluster resources
```

Loki-like platform costs mainly concentrate on:

```text
Log Write

Tab Index

Object Storage

Query Range
```

---

## Scenario 36: Common Causes of Log Volume Surge

Common causes:

```text
Production is on. DEBUG

Some bug brush

Unusual retesting storm.

Too many health check logs.

Too many static resource logs

reptiles or scan flows

Too many new pages.

Print Log in Cycle

Changes in log format lead to duplicate collections

Collection path configuration too wide

Repeated packaging log collection

Multiline log splits lead to a surge in numbers
```

---

## Scenario 37: Quick Local Judgment of Log Volume

Check directory size:

```bash
du -sh /var/log/myapp
```

Check large files:

```bash
find /var/log/myapp -type f -size +500M -exec ls -lh {} \;
```

Check logs modified in the last hour:

```bash
find /var/log/myapp -type f -mmin -60 -exec ls -lh {} \;
```

Count log lines:

```bash
wc -l /var/log/myapp/app.log
```

Check the number of error lines in the latest 1000 lines:

```bash
tail -n 1000 /var/log/myapp/app.log | grep -i "error" | wc -l
```

---

## Scenario 38: Quick Analysis of Log Volume Sources for Nginx

Count the most accessed URLs:

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Count the most accessed IPs:

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Count requests by minute:

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Count status codes:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Scenario 39: Cost Control Methods for Log Platforms

Common methods:

```text
Limits DEBUG

Filter health check

Filter static resources

Successful request for sampling

Error Request Full

Field Normalization

Reduction of high-base digital segment index

Shorten Low Value Log Retention Cycle

Cold thermal layer

Archive old log to object storage

By Service Statistics Log

Cost-sharing by team

It's an anomaly.
```

---

## 12. Field and Label Cost Control

---

## Scenario 40: High Cardinality Fields Should Not Be Used as Labels

High cardinality fields:

```text
trace_id

request_id

user_id

order_id

session_id

client_ip

Full URL

With arguments URL

Cell phone number

Mailbox

token
```

Should not be used as Loki labels, nor should they be arbitrarily used for high-cost aggregation.

Low cardinality fields suitable for labels:

```text
env

service

namespace

cluster

region

team

level

container

pod
```

Note:

```text
pod Base value ratio service High, but... Kubernetes The scenes are often used.

trace_id / user_id Such a unique value should not be acted upon label
```

---

## Scenario 41: Field Types Should Be Stable

A field should not switch between string and number types.

Incorrect example:

```json
{"status_code": 200}
{"status_code": "500"}
```

Problem:

```text
Field Map Conflict

Convergence Failed

Query anomaly

Index writing failed
```

Recommend to unify:

```json
{"status_code": 200}
```

Or unify as strings, but status codes are more recommended as numbers.

---

## Scenario 42: URL Fields Should Be Normalized

Original URL:

```text
/api/users/10001/profile

/api/users/10002/profile

/api/users/10003/profile
```

If directly used as a field for aggregation, it will lead to high cardinality in path.

More recommended to add normalized path:

```text
/api/users/{id}/profile
```

Log example:

```json
{
  "path": "/api/users/10001/profile",
  "route": "/api/users/{id}/profile"
}
```

Use for querying and statistics:

```text
route
```

Use for troubleshooting specific requests:

```text
path
```

---

## 13. Log Alert Governance

---

## Scenario 43: Do Not Alert for Every Error

If an alert is triggered for every error, it will lead to:

```text
Too many alarms.

Serious misreporting

The watchman is numb.

The real malfunction is flooded.
```

More reasonable approach:

```text
By Time Window

Aggregation by service

Filter by Environment

Call the police at the wrong rate.

Exclude the known noise

Distinction warning and critical
```

---

## Scenario 44: Recommended Log Alert Practices

Example rules:

```text
prod Environmental Service 5 Within minutes ERROR Number > 100

prod Environmental Service ERROR Percentage > 5%

Nginx 5xx Percentage 5 Within minutes > 3%

The same. IP 5 Within minutes 404 Over 500 Minor

Come on. OOMKilled Keyword

Come on. too many open files

Come on. no space left on device

Come on. nf_conntrack: table full
```

---

## Scenario 45: Log Alert Noise Reduction

Noise reduction methods:

```text
Set Duration

Set Aggregation Window

Set Restore Notification

The same kind of alarm merger.

Aggregation by service

Heavy by Example

Maintenance of the silence rule

Excludes known negligible errors

Call the police.
```

Alert grading example:

```text
P1
→ Core business not available, extensive impact

P2
→ Single-service anomalies affecting part of the business

P3
→ Risks but not significant impact on operations

P4
→ Observation type alert or capacity alert
```

---

## 14. Log Desensitization Check

---

## Scenario 46: Local Check for Suspected Sensitive Information

Check tokens:

```bash
grep -RniE "token|access_token|refresh_token|authorization|cookie|secret|password" /var/log/myapp
```

Check phone number patterns:

```bash
grep -RniE "1[3-9][0-9]{9}" /var/log/myapp
```

Check ID number patterns:

```bash
grep -RniE "[0-9]{17}[0-9Xx]" /var/log/myapp
```

Check AccessKey keywords:

```bash
grep -RniE "accesskey|secretkey|ak|sk" /var/log/myapp
```

Note:

```text
These orders only reveal part of the risk.

No substitute for formal dissensitization rules and security audits
```

---

## Scenario 47: Check if Nginx Logs Authorization

Check log format:

```bash
grep -R "log_format" /etc/nginx/
```

Search for Authorization:

```bash
grep -R "http_authorization" /etc/nginx/
```

If the log format records:

```text
$http_authorization
```

It needs careful evaluation, and it's generally not recommended to log in plain text.

---

## 15. Log Governance Inspection Checklist

---

## 1. Log Level

```text
Whether production is default INFO

Whether to open for long periods DEBUG

ERROR Is it really a mistake?

WARN Reasonable use

Existence error Consider it info Situation

Whether or not there's a normal business failure has been heavily hit. error Situation
```

---

## 2. Log Content

```text
Record trace_id / request_id

Record service / env

Record status_code / duration_ms

Is the error log stacked?

Whether the error log has a key context

Whether or not to record complete requests Body

Whether to record a complete response

Whether to record sensitive fields
```

---

## 3. Log Format

```text
Structured JSON

Field names are uniform

Field type stable

Time format consistency

Harmonization of time zones

Multiline logs correctly merged

Is the abnormal stack complete?
```

---

## 4. Log Collection

```text
Whether the acquisition path is accurate

Repeated collections

Whether useless logs were collected

Did you filter the health check?

Whether containers have been collected stdout / stderr

Other Organiser Kubernetes metadata

Did you collect sensitive logs?
```

---

## 5. Log Platform

```text
Whether the index is split by service or type

Whether labels are low base

Reasonability of the retention cycle

Is there a cold thermal layer?

Is there a log monitor?

Whether logs are written to fail alarms

Query performance issues

Is there a quarantine?
```

---

## 6. Log Security

```text
Whether to record the password

Record token

Record Cookie

Do you have an ID number?

Whether to record the full number of cell phones

Recording bank cards

Whether to record key

Is there a dissensitisation rule?

Log Access Control
```

---

## 16. Common Log Governance Issues and Solutions

---

## Scenario 48: Sudden Log Volume Surge

Troubleshooting path:

```text
By Service Statistics Log

→ Press level Statistics Log

→ Most of the time. message

→ Check if it's open. debug

→ Check if there is an error brush

→ Check for health checks or static resources

→ Check if it's abnormally retrying.

→ Temporary Noise Reduction

→ Fix the roots.
```

Local troubleshooting:

```bash
du -sh /var/log/myapp
```

```bash
find /var/log/myapp -type f -mmin -60 -exec ls -lh {} \;
```

```bash
tail -n 1000 /var/log/myapp/app.log | awk '{print $0}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 49: Log Platform Cost Increase

Troubleshooting direction:

```text
Which index is growing fastest?

Which service log is the largest?

Which one? level Most

Collection debug

Collection healthcheck

Whether the field base is too high

Retention of excessive duration

Whether there are too many copies

Whether duplicate collections exist
```

Governance direction:

```text
Adjust collection range

Filter low value logs

Shorten Low Value Log Retention Cycle

Archive Cold Data

Optimizing indexing and labelling

Log costs by service statistics

Promotion of HF valueless logbook revision by operational parties
```

---

## Scenario 50: Sensitive Information Entering Logs

Handling process:

```text
Confirm the leak.

Stop writing immediately.

Modify log output logic

Clear or isolate related logs

Assess the need to delete the index

Assess the need to report security incidents

Increased sensitivity rule

Additional test examples

Why didn't they intercept us earlier?
```

Local quick localization:

```bash
grep -RniE "password|token|authorization|cookie|secret" /var/log/myapp
```

---

## Scenario 51: Unable to Find trace_id in Logs

Common causes:

```text
The portal is not generated. trace_id

Inter-service calls are not passed trace_id

Application log not printed trace_id

Log collection resolve lost field

Field name is not uniform

Discrepancies between different language services
```

Governance direction:

```text
Harmonization trace_id Field name

Level of entry generation trace_id

Interservice transmission trace_id

Log Frame Auto Injected trace_id

Log Platform Resolution trace_id Fields

The barrier. SOP Priority is required trace_id
```

---

## 17. Production Log Specification Examples

---

## Scenario 52: Recommended JSON Log Format

Recommended format:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "error",
  "service": "resume-api",
  "env": "prod",
  "version": "v1.2.3",
  "trace_id": "abc123",
  "request_id": "req-001",
  "message": "database query timeout",
  "error_type": "DatabaseTimeout",
  "duration_ms": 3000
}
```

---

## Scenario 53: Recommended HTTP Access Log Fields

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "info",
  "service": "resume-api",
  "env": "prod",
  "trace_id": "abc123",
  "client_ip": "10.0.0.5",
  "method": "GET",
  "path": "/api/users/10001/profile",
  "route": "/api/users/{id}/profile",
  "status_code": 200,
  "duration_ms": 42,
  "request_size": 512,
  "response_size": 2048
}
```

---

## Scenario 54: Recommended Error Log Fields

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "error",
  "service": "payment-api",
  "env": "prod",
  "trace_id": "abc123",
  "request_id": "req-001",
  "error_type": "PaymentCallbackFailed",
  "message": "payment callback failed",
  "dependency": "payment-gateway",
  "duration_ms": 2000,
  "retry_count": 3
}
```

---

## Scenario 55: Recommended Audit Log Fields

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "info",
  "service": "admin-api",
  "env": "prod",
  "operator": "user-001",
  "action": "delete_resource",
  "resource_type": "resume",
  "resource_id": "resume-001",
  "source_ip": "10.0.0.5",
  "result": "success"
}
```

---

## 18. Production Implementation Recommendations

---

## 1. First Unify Fields

Prioritize unifying:

```text
timestamp

level

service

env

trace_id

message
```

Then gradually supplement:

```text
request_id

status_code

duration_ms

path

route

client_ip

version
```

---

## 2. Prioritize High-Risk Log Governance

Prioritize governance:

```text
Sensitive Information Log

debug Bulk Log

error Brush Log

Health check log

Static Resource Log

Repeat collection log

None trace_id Core service log
```

---

## 3. Establish Log Baselines

At least establish:

```text
Daily logs per service

Each service ERROR Number

Each service 5xx Number

Cost per service log

Each Index Size

Every type of log retention cycle
```

---

## 4. Incorporate Log Governance into Release Checks

Pre-release checks:

```text
New HF log

Whether to add sensitive fields

Whether to adjust log level

Whether or not to destroy JSON Format

Impact trace_id

Whether to introduce a large number debug

Whether to influence log platform resolution rules
```

---

## Nineteen. Common Command Summary in This Article

---

## Check Log Volume

```bash
du -sh /var/log/myapp
```

```bash
find /var/log/myapp -type f -size +500M -exec ls -lh {} \;
```

```bash
find /var/log/myapp -type f -mmin -60 -exec ls -lh {} \;
```

```bash
wc -l /var/log/myapp/app.log
```

```bash
tail -n 1000 /var/log/myapp/app.log | grep -i "error" | wc -l
```

---

## Nginx Log Volume Analysis

```bash
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print substr($4,2,17)}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Health Check Noise Reduction Analysis

```bash
awk '$7 == "/health" {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$7 != "/health" {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$7 !~ /^\/(health|healthz|ready|readiness|liveness|metrics)$/ {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Static Resource Log Analysis

```bash
awk '$7 ~ /\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/ {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$7 !~ /\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/ {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Sensitive Information Check

```bash
grep -RniE "token|access_token|refresh_token|authorization|cookie|secret|password" /var/log/myapp
```

```bash
grep -RniE "1[3-9][0-9]{9}" /var/log/myapp
```

```bash
grep -RniE "[0-9]{17}[0-9Xx]" /var/log/myapp
```

```bash
grep -RniE "accesskey|secretkey|ak|sk" /var/log/myapp
```

```bash
grep -R "log_format" /etc/nginx/
```

```bash
grep -R "http_authorization" /etc/nginx/
```

---

## sed Desensitization Example

```bash
sed -E 's/([0-9]{3})[0-9]{4}([0-9]{4})/\1****\2/g' app.log
```

```bash
sed -E 's/(Authorization: Bearer )[A-Za-z0-9._-]+/\1***/g' app.log
```

---

## jq Desensitization Example

```bash
jq 'del(.password, .token)' app.json
```

```bash
jq '.password="***" | .token="***"' app.json
```

---

## Log Duplicate Content Analysis

```bash
tail -n 1000 /var/log/myapp/app.log | awk '{print $0}' | sort | uniq -c | sort -nr | head
```

```bash
grep -i "error" /var/log/myapp/app.log | sort | uniq -c | sort -nr | head
```

---

## Twenty. One-Sentence Summary

The core of production log governance is not "logging more," but:

```text
Key log to complete

Normal log to be appropriate

Sensitive logs need to be dissensitized.

HF logs are down.

Low-value log short.

High Value Log to Keep

Log costs are manageable
```

Log level recommendations:

```text
DEBUG
→ Production default closed, temporary opening must be time-limited

INFO
→ Record normal critical events

WARN
→ Recording restores anomalies and risk signals

ERROR
→ Record clearly failed

FATAL
→ Recording of serious errors where service cannot continue
```

Log desensitization focus:

```text
Password

token

Cookie

Authorization

Cell phone number

ID number

Bank card

AccessKey / SecretKey

Private key and certificate
```

Log sampling recommendations:

```text
ERROR Full

WARN Most reservations

INFO Selective reservations

DEBUG Default Not

Health screening and static resource noise reduction

Sample of successful request

Failure request should be full
```

Log retention period recommendations:

```text
Local log short-term retention

Interim retention of the Platform ' s log

High-value log long-term retention

Low-value log short-term retention

Audit logs maintained for compliance requirements

Cold data entry object storage archive
```

Cost control focus:

```text
Limits DEBUG

Filter low value logs

Control high-base segments

Avoid duplication of collection

Shorten Low Value Log Retention Cycle

Cold thermal layer

Log and cost by service statistics
```

Production recommendations:

```text
Harmonization timestamp / level / service / env / trace_id / message

Apply logs as structured as possible JSON

Do not explicitly record sensitive fields

Don't open it for long. DEBUG

Don't. trace_idI don't know.user_idI don't know.order_id Put it in. Loki label

Don't see the logs. error Just call the police. It's designed by window, scale and impact range.

Log governance needs to be included in release checks and failure rediscretion
```