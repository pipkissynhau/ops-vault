# 07-Log Platform Thinking: Collection, Indexing, Fields, Tags, and Retrieval Design

#Linux #LogPlatform #ELK #Loki #Filebeat #Logstash #Promtail #LogCollection #LogIndex #LogSearch #SRE #Observation

---

## Recommended Path

01-Linux Foundation and Host Maintenance/02-Logs and Text Processing/07-Log Platform Thinking: Collection, Indexing, Fields, Tags, and Retrieval Design.md

---

## One, Document Description

This document organizes fundamental design concepts for building a log platform.

The previous 01-06 articles mainly focus on:

```text
Single Log View

Text Filter

awk / grep / sed / find

Nginx Log Analysis

Log Archive

JSON Log processing

mail Announcements

rsyslog Forward
```

This article begins to focus on the log platform perspective, emphasizing:

- Why do we need a log platform?
- What problems does a log platform solve?
- Log collection pipeline
- Log field design
- Log tag design
- Log indexing design
- Log retrieval design
- Log retention period
- Log volume control
- Common components of a log platform
- ELK / EFK basic concepts
- Loki basic concepts
- Filebeat / Logstash / Fluent Bit / Promtail positioning
- Design considerations for production log platforms

The goal is:

To understand that a log platform is not simply collecting logs

→ To know how to design a log collection pipeline

→ To understand why log fields and tags are important

→ To distinguish between full-text search and tag search

→ To understand the differences between Elasticsearch and Loki

→ To know how a log platform supports troubleshooting, auditing, alerts, and cost control

---

## Two, Why Do We Need a Log Platform?

Single-machine log lookup is suitable for small-scale scenarios.

For example:

```bash
tail -f app.log
```

```bash
grep -i "error" app.log
```

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

But in production environments, single-machine log lookup faces many issues:

```text
Service deployed on multiple machines.

There are several examples of an application

Logs scattered at different points

Log lost after container restart

Pod Float to different nodes

Multi-service connection complex.

I don't know which machine to check when it's down.

This disk log has limited time

It's too big.grep Very slowly.

Need field-based retrieval and aggregation

Need for long-term audit and traceability

We need to warn on the basis of the log.
```

The problems a log platform needs to solve are:

```text
Centralized collection

Unified storage

Unique Search

Harmonization of analysis

Unified alarm.

Harmonization of the reservation cycle

Unified authority control
```

One-sentence understanding:

```text
Single Log Settlement“What happened to this machine?”

Log platform resolved“What happened to the whole system?”
```

---

## Three, Core Problems Solved by a Log Platform

A log platform typically solves five types of problems.

---

## 1. Troubleshooting

Used to answer:

```text
Which service is wrong?

Which case is wrong?

When did the mistake start?

Are there more errors?

Which users are affected?

Which interface? 5xx At most?

Some trace_id After what?

Where did an order number fail?
```

---

## 2. Monitoring and Alerts

Used to trigger alerts based on logs:

```text
5 Within minutes error Quantity above threshold

Nginx 5xx Ratio above threshold

Application appearance OOM Keyword

Too many database failures

Too many login failures

Some kind of abnormal pile has suddenly increased.
```

---

## 3. Audit and Tracking

Used to track:

```text
Who's on the server?

Who did it? sudoWhat?

Who visited the backstage interface?

Who changed the configuration?

Who deleted the resources?

Which one? IP A sensitive operation?
```

---

## 4. Business Analysis

Used to analyze:

```text
Interface Call

User Access Path

Error order distribution

Interface time-consuming distribution

Changes in requests during an activity

Error change after release of a version
```

---

## 5. Stability Governance

Used for long-term governance:

```text
Error TopN

Slow Interface TopN

Unusual services TopN

Log Volume TopN

Noise Log Governance

Useless debug Log governance

High-cost index governance
```

---

## Four, Log Platform Architecture

Common log platform pipeline:

```text
Log Generation
→ Log Collection
→ Log Resolution
→ Log Filter
→ Log Transfer
→ Log Storage
→ Log Index
→ Log Search
→ It's a log call.
→ Log Archive
```

Common architecture:

```text
Apply / System / Containers
        ↓
Filebeat / Fluent Bit / Promtail / Vector / rsyslog
        ↓
Logstash / Kafka / Data-processing layer
        ↓
Elasticsearch / OpenSearch / Loki / ClickHouse / Object Storage
        ↓
Kibana / Grafana / Log Retrieving Platform
        ↓
Police! / Report / Audit / The barrier.
```

Not all scenarios require a complete pipeline.

Small-scale scenarios:

```text
Apply Log
→ Filebeat
→ Elasticsearch
→ Kibana
```

Or:

```text
Apply Log
→ Promtail
→ Loki
→ Grafana
```

Large-scale scenarios:

```text
Apply Log
→ Agent
→ Kafka
→ Logstash / Flink
→ Elasticsearch / ClickHouse / Loki
→ Query Platform / Police platform
```

---

## Five, Log Source Design

The log platform must first clarify what logs to collect.

Common log sources include:

```text
System Log

Apply Log

Nginx / Ingress Log

Standard Output Log for Containers

Kubernetes Component log

Middle Log

Database Log

Audit log

Security Log

Network Device Log

Cloud Platform Operations Log
```

---

## 1. System Logs

Common paths:

```text
/var/log/messages

/var/log/syslog

/var/log/auth.log

/var/log/secure

/var/log/kern.log
```

Common uses:

```text
Host anomaly

SSH Login

sudo Operation

OOM

Disk Error

Core Error

Service startup failed
```

---

## 2. Application Logs

Common paths:

```text
/var/log/myapp/app.log

/data/logs/myapp/app.log

Apply container standard output
```

Common content:

```text
Interface requests

Business Error

Unusual Stack

Reliance on Call

Database error

Cache Error

Message consumption error
```

---

## 3. Nginx / Ingress Logs

Common content:

```text
Visits IP

Request Time

Method of request

URL

Status Code

Time-consuming request

Upstream time

User-Agent

Referer

X-Forwarded-For
```

Suitable for analyzing:

```text
Visits

Status Code

Slow Request

5xx

Unusual IP

reptiles.

Attack scan.

Backend upstream Problem
```

---

## 4. Kubernetes Container Logs

Common paths:

```text
/var/log/containers/*.log

/var/log/pods/*/*.log
```

Container log characteristics:

```text
Pod It floats.

The container will be restarted.

Log path contains namespaceI don't know.podI don't know.container Information

Automatic recognition required Kubernetes metadata

We need to distinguish. stdout and stderr
```

---

## Six, Log Collection Agent Selection

Common log collection components:

```text
Filebeat

Logstash

Fluentd

Fluent Bit

Promtail

Vector

rsyslog
```

---

## 1. Filebeat

Positioning:

```text
Lightweight log collection Agent
```

Common uses:

```text
Collect File Log

Collection Nginx Log

Collection System Log

Send to Elasticsearch / Logstash / Kafka
```

Features:

```text
Lower resource occupancy

Configure relatively simple

and Elastic Stack We're getting mature.

It suits tradition. ELK / EFK scene
```

---

## 2. Logstash

Positioning:

```text
Log resolution, filtering, conversion, route processing
```

Common uses:

```text
grok Parsing Log

Field Cleaning

Field Rename

Desensitivity.

Multiple Target Output

Complex Log Processing
```

Features:

```text
Powerful

Relatively high resource occupancy

Fit as Log Process Layer

It doesn't necessarily fit every machine.
```

---

## 3. Fluent Bit

Positioning:

```text
Lightweight log collection and forwarding Agent
```

Common uses:

```text
Kubernetes Packaging log collection

Edge Node Log Collection

Send to Elasticsearch / Loki / Kafka / S3
```

Features:

```text
Light

Fit for container environment

It's better.

Plugin Enrichment
```

---

## 4. Promtail

Positioning:

```text
Loki Log collection Agent
```

Common uses:

```text
Collect File Log

Collection Kubernetes Container log

Extract labels

Send to Loki
```

Features:

```text
and Loki / Grafana Get it together.

Emphasis on tag search

Other Organiser

Cost is usually lower than Elasticsearch
```

---

## 5. rsyslog

Positioning:

```text
System log collection and forwarding
```

Common uses:

```text
Forward System Log

Concentration Linux syslog

Security Audit Log

Network equipment syslog
```

Features:

```text
Common in the system

Fit syslog Agreements

Fit to traditional system log forward

Not suitable for complex applications JSON Log Resolution
```

---

## Seven, Basic Understanding of ELK / EFK

ELK refers to:

```text
Elasticsearch
→ Storage, Index, Search

Logstash
→ Collect, interpret, filter, convert

Kibana
→ Query, display, dashboard
```

EFK commonly refers to:

```text
Elasticsearch
→ Storage, Index, Search

Fluentd / Fluent Bit
→ Collection and forwarding

Kibana
→ Query and Presentation
```

Common pipeline:

```text
Apply Log
→ Filebeat / Fluent Bit
→ Logstash
→ Elasticsearch
→ Kibana
```

It can also be simplified to:

```text
Apply Log
→ Filebeat
→ Elasticsearch
→ Kibana
```

---

## Eight, Basic Understanding of Loki

Loki is a log system in the Grafana ecosystem.

Typical pipeline:

```text
Apply Log
→ Promtail / Fluent Bit
→ Loki
→ Grafana
```

Loki's core features:

```text
Yeah. Prometheus Use same labels

Default not to create a backwards index for the full log

Main Index Tabs

Log contents query by time and tab

Storage costs are usually lower.

It's perfect for the clouds.
```

Loki is more suitable for:

```text
Kubernetes Log

Press namespace / pod / app / container Question

Combined Prometheus Indicator barriers

Combined Grafana Unified view of indicators and logs
```

Unsuitable scenarios:

```text
There's a lot of complicated full text searches.

Requires a large number of any field aggregates

♪ Need like ♪ Elasticsearch Search all text fields at the same depth
```

---

## Nine, Core Differences Between Elasticsearch and Loki

| Comparison Item | Elasticsearch / OpenSearch | Loki |
|---|---|---|
| Search Method | Strong full-text search | Strong label search |
| Indexing Method | Build indexes for many fields | Mainly index labels |
| Storage Cost | Relatively high | Usually low |
| Query Capability | Strong field aggregation, full-text search | Filter by labels and log content |
| Typical Interface | Kibana | Grafana |
| Suitable Scenarios | Complex log analysis, auditing, search | Cloud-native logs, correlation with metrics for troubleshooting |
| Risk Points | Index bloat, high cost | Unreasonable label design can cause explosion |

One-sentence understanding:

```text
Elasticsearch More like a log search engine.

Loki More like a labeled log time series system
```

---

## Ten, Log Field Design

Field design is very important in a log platform.

If fields are chaotic, it will lead to:

```text
Retrieving difficulties

Convergence difficulty

It's hard to warn.

Field Conflict

Index expansion

Logs are hard to reuse

Different services could not be analysed uniformly
```

Recommended basic fields:

```text
timestamp

level

service

env

host

ip

namespace

pod

container

trace_id

request_id

user_id

method

path

status_code

duration_ms

message
```

---

## 1. Time Field

Recommended field:

```text
timestamp
```

Requirements:

```text
It has to be accurate.

There must be a unified time zone

Best used ISO 8601 Format

Collect and service end time synchronized

Don't just rely on collecting time.
```

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00"
}
```

---

## 2. Log Level Field

Recommended field:

```text
level
```

Common values:

```text
debug

info

warn

error

fatal
```

It is recommended to standardize to lowercase or uppercase.

For example, standardize to lowercase:

```json
{
  "level": "error"
}
```

It is not recommended to mix usage in the same platform:

```text
ERROR

Error

error

ERR
```

---

## 3. Service Field

Recommended field:

```text
service
```

Example:

```json
{
  "service": "resume-api"
}
```

Usage:

```text
Filter Log By Service

Quantity of statistical service errors

Service level alert.

Service-level cost accounting
```

---

## 4. Environment Field

Recommended field:

```text
env
```

Common values:

§
```text
dev

test

staging

pre

prod
```

Usage:

```text
Distinguishing test and production logs

Avoid mischecking the environment

Production alarms only target prod
```

---

## 5. Request Chain Field

Recommended field:

```text
trace_id

request_id
```

Usage:

```text
Multiple services requested at one time

Rapid positioning cross-service issues

We'll link up and track the barrier.
```

Example:

```json
{
  "trace_id": "abc123",
  "request_id": "req-20260425-001"
}
```

---

## 6. HTTP Fields

Recommended Fields:

```text
method

path

status_code

duration_ms

client_ip

user_agent
```

Example:

```json
{
  "method": "GET",
  "path": "/api/login",
  "status_code": 200,
  "duration_ms": 35
}
```

---

## 11. Log Label Design

Labels are commonly used for quick filtering and aggregation.

Common Labels:

```text
env

service

app

namespace

pod

container

node

cluster

region

team

version
```

Label Design Principles:

```text
Low base figure

Stabilization

Reusable

It makes sense.

Do not place a unique value in the label
```

---

## 1. What is Low Cardinality

Low cardinality means the number of possible values for a field is limited.

Suitable as a label:

```text
env
→ prod / test / dev

service
→ resume-api / user-api / order-api

namespace
→ default / prod / monitoring

cluster
→ cluster-a / cluster-b

team
→ sre / backend / platform
```

Not suitable as a label:

```text
request_id

trace_id

user_id

order_id

session_id

Full URL With Random Arguments

Client IP
```

Reason:

```text
These fields are worth too much.

It causes the label to explode.

Indexing and storage pressure surged

Query Performance Decline

Cost increases
```

---

## 2. Loki Labels Should Be Used Cautiously

Loki primarily indexes labels.

Putting high cardinality fields into labels will cause:

```text
label Explosion

Index expansion

Slowness of queries

Writing pressure increased

Storage costs increase
```

Not Recommended Design:

```text
trace_id As label

user_id As label

request_id As label

Full path As label

client_ip As label
```

Recommended:

```text
env

service

namespace

pod

container

cluster

level
```

---

## 12. Log Index Design

Index design determines:

```text
How do you keep the log?

What about the log?

How long do you keep the log?

How expensive is the log?

How's it going?
```

---

## 1. Common Elasticsearch Index Design

Common daily index:

```text
app-prod-2026.04.25

nginx-access-2026.04.25

system-log-2026.04.25
```

Advantages:

```text
Administration of convenience by date

Easy to set a retention cycle

It's cold and thermal.

Easy to delete old index
```

---

## 2. Index by Log Type

Example:

```text
app-log-*

nginx-access-*

nginx-error-*

system-log-*

audit-log-*
```

Advantages:

```text
Fields are more stable

Query clearer

Access is easier to isolate.

Life cycle strategies are easier to set up
```

---

## 3. Not Recommended to Put All Logs into One Index

If all logs are put into a single large index:

```text
Field Chaos

Slowness of queries

Map Conflict

We don't have good clearance.

Life cycle poorly managed

Cost's not controlled.
```

Not Recommended:

```text
all-logs-2026.04.25
```

More Recommended:

```text
app-prod-2026.04.25

nginx-prod-2026.04.25

system-prod-2026.04.25

audit-prod-2026.04.25
```

---

## 4. Index Lifecycle

Logs are not permanently stored in high-performance storage.

Common Lifecycle:

```text
Thermal data
→ Near 7 Oh, my God. I've been searching a lot.

Temperature data
→ 7-30 Oh, my God.

Cold Data
→ 30-180 Days, low frequency queries

Archive data
→ object storage, resume query if necessary
```

---

## 13. Log Parsing Design

The goal of log parsing is to convert raw text into structured fields.

For example, raw Nginx log:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123 "-" "Mozilla/5.0"
```

After parsing:

```json
{
  "client_ip": "10.0.0.5",
  "timestamp": "2026-04-25T10:00:01+08:00",
  "method": "GET",
  "path": "/api/login",
  "status_code": 200,
  "body_bytes_sent": 123,
  "user_agent": "Mozilla/5.0"
}
```

After structuring, you can do:

```text
Press status_code Aggregation

Press path Statistics

Press client_ip Analysis

Press duration_ms I'm looking for a slow request.

Press service Filter

Press trace_id Serial
```

---

## 14. Log Format Recommendations

Application logs should preferably use JSON format.

Recommended:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00",
  "level": "error",
  "service": "resume-api",
  "env": "prod",
  "trace_id": "abc123",
  "request_id": "req-001",
  "message": "database connection timeout",
  "duration_ms": 1200
}
```

Not Recommended with only plain text:

```text
2026-04-25 10:00:00 ERROR database connection timeout
```

Plain text is not unusable, but the disadvantages are:

```text
Field resolution costs

Different service formats are not harmonized

Convergence difficulty

Poor search accuracy

High governance costs
```

---

## 15. Advantages of JSON Logs

Advantages of JSON logs:

```text
Natural structure

Fields Clear

Fit jq Processing

Fit to log platform resolution

Fit to aggregate statistics

It's a good warning rule.

Suitable for cross-service harmonization field
```

Example:

```bash
cat app-json.log | jq -r 'select(.level == "error") | [.timestamp, .service, .message] | @tsv'
```

Statistics for status codes:

```bash
cat access-json.log | jq -r '.status_code' | sort | uniq -c | sort -nr
```

Statistics for slow requests:

```bash
cat access-json.log | jq -r 'select(.duration_ms > 1000) | [.timestamp, .path, .duration_ms] | @tsv'
```

---

## 16. Log Search Design

Common search methods in log platforms:

```text
Keyword Search

Field Search

Label Search

Time frame search

Group Conditions Search

Aggregation Statistics

Context Search
```

---

## 1. Keyword Search

Used to find:

```text
error

timeout

exception

connection refused

OOMKilled

no space left
```

Suitable for:

```text
Rapidly locate abnormal text

Retrieval when field structures are unknown

Check the abnormal stack.
```

---

## 2. Field Search

Used to find:

```text
level = error

service = resume-api

status_code >= 500

duration_ms > 1000

trace_id = abc123
```

Suitable for:

```text
Exact Filter

Aggregation Statistics

The alarm rule.

dashboard
```

---

## 3. Label Search

Common in Loki:

```text
{env="prod", service="resume-api"}
```

Continue filtering content:

```text
{env="prod", service="resume-api"} |= "error"
```

Filter by JSON field:

```text
{env="prod", service="resume-api"} | json | level="error"
```

---

## 17. Common Log Query Issues

---

## 1. Logs Not Found

Common reasons:

```text
Time frame was wrong

Environment Selection Error

service Name inconsistent

It's not collected.

Collection Agent Unusual

Log path configuration error

Pod The label is incorrect

Error selecting index name

Field Name Error

Logs dropped by filter

Log not refreshed to platform
```

Troubleshooting Directions:

```text
Make sure the log exists on this machine.

Reconfirm collection Agent Status

Reconfirm receipt of log platform

Confirm search conditions again
```

---

## 2. Slow Query

Common reasons:

```text
It's too long.

Too big index.

Fields without Index

The keyword is too broad.

Aggregation of high-base digital segments

It's too big.

Cold thermal data query

Unprecision of query conditions
```

Optimization Directions:

```text
Reduce the time frame

Increase service/env Conditions

Avoid Full Fuzzy Search

Use structured fields

Optimizing the index life cycle

Split Index
```

---

## 3. Field Parsing Failure

Common reasons:

```text
Log Format Change

Inconsistent application of version log format

Multiple row abnormal stacks not merged

JSON Format Invalid

Field type conflict

Logstash grok Rules don't match

Collectend parser Configure Error
```

---

## 18. Multi-line Log Handling

Application exceptions stacks are usually multi-line.

For example, Java exceptions:

```text
java.lang.RuntimeException: db timeout
    at com.example.Service.query(Service.java:20)
    at com.example.Controller.login(Controller.java:35)
```

If not merged into multi-lines, log platforms may split one exception into multiple logs.

Consequences:

```text
Abnormal context fractured

Retrieving difficulties

The number of alarms has increased.

Stack incomplete
```

Need to configure multiline on the collection side.

Common merging approaches:

```text
New behavior log starting with time stamp

Lines that do not start with a time stamp merge to the previous one

Java Merge stack lines to an abnormal main line
```

---

## 19. Log Alert Design

Do not simply alert on seeing "error" in logs.

Otherwise, it will lead to:

```text
Too many alarms.

Too loud.

The watchman is numb.

The real malfunction is flooded.
```

Recommended alert methods:

```text
By Time Window

By error rate

Aggregation by service

Filter by Environment

Press critical error alert.

Exclude the known noise
```

Example rule logic:

```text
prod Environmental Service 5 Within minutes error Number > 100

prod Environment Nginx 5xx Percentage > 5%

A service appears. OOMKilled

Login Failed 5 Over in minutes. 50 Minor

The same. IP 404 Request exceeded 500 Minor
```

---

## 20. Log Volume and Cost Control

The biggest issue with log platforms is cost.

Excessive log volume may lead to:

```text
Collection Agent Pressure.

Network bandwidth consumption

Storage costs increase

Index cost increases

Slowness of queries

Log platform cluster under heavy pressure

Retention cycles forced to be shortened
```

Common log volume management methods:

```text
Close Production debug Log

Limits access log Collection range

Sampled low-value HF logs

Filter health check logs

Filter static resource access logs

Desensitivity to log fields

Different retention cycles by service

High-value log long-term storage

Low-value log short-term storage
```

---

## 21. Production Log Field Specification Recommendations

Applications should at least include:

```text
timestamp

level

service

env

trace_id

request_id

message
```

HTTP interface logs should include:

```text
timestamp

service

env

trace_id

client_ip

method

path

status_code

duration_ms

request_size

response_size
```

Error logs should include:

```text
timestamp

level

service

env

trace_id

error_type

error_message

stack

host

pod

container
```

Audit logs should include:

```text
timestamp

operator

action

resource_type

resource_id

source_ip

result

reason
```

---

## 22. Production Log Platform Notes

---

## 1. Logs Must Have a Time Field

Without a time field, it will lead to:

```text
The search time is off.

Unsequence of events

Not the alarm window.

Rewinding difficulty
```

---

## 2. Do Not Keep Debug Logs Long-term in Production

Debug log risks:

```text
Logs are booming.

Disk Pressure

Collect pressure.

Increased index costs

Possible disclosure of sensitive information
```

---

## 3. Sensitive Information Must Be Desensitized

Logs should not contain plaintext:

```text
Password

token

Cookie

ID number

Cell phone number

Bank card

AccessKey

SecretKey

Private Key

Authentication Code
```

---

## 4. trace_id Is Very Important

Problems without trace_id:

```text
Cross-service clearance difficulties

A request cannot be linked

It depends on time and keywords.

Fault tracking is inefficient.
```

---

## 5. Log Format Must Be Stable

Frequent field changes will lead to:

```text
Parsing failed

Index Field Conflict

The alarm is out.

The dashboard is dead.

Inaccuracy of statistics
```

---

## 6. Log Platforms Also Need Monitoring

Need to monitor:

```text
Collection Agent Alive?

Log Write Rate

Number of logs discarded

Kafka Stack

Elasticsearch Cluster Status

Loki Writing failed

Index Size

Disk Usage

Query delay

Is the alarm rule normal?
```

---

## 23. Common Log Platform Troubleshooting Process

---

## Scenario 1: Application Logs Not Found

Troubleshooting Chain:

```text
Confirms whether local logs are generated

→ Confirms whether the log path is correct

→ Confirm collection Agent Whether to run

→ Confirm if the capture configuration matches the path

→ Confirm access to logs

→ Confirm if the log is filtered

→ Confirms whether log platform is successfully written

→ Confirms whether the time frame of the query and the label are correct
```

Confirm Logs Locally:

```bash
tail -n 100 /var/log/myapp/app.log
```

Check Agent Status:

```bash
systemctl status filebeat
```

Or:

```bash
systemctl status promtail
```

Check Agent Logs:

```bash
journalctl -u filebeat -n 100
```

Or:

```bash
journalctl -u promtail -n 100
```

---

## Scenario 2: Log Platform Query is Slow

Troubleshooting Chain:

```text
Reduce the time frame

→ Increase service / env Conditions

→ Avoid Global Fuzzy Search

→ View index or storage status

→ See if the log is skyrocketing

→ See if there is a high-base digital segment aggregate

→ See if Platform resources are bottlenecks
```

---

## Scenario 3: Log Volume Suddenly Surges

Troubleshooting Chain:

```text
Press service Statistics Log

→ I'm looking for the biggest number of logs.

→ The biggest search logs. level

→ Most of the time. message

→ Confirm if it's open. debug

→ Make sure you try again.

→ Confirm if there is an error brush

→ Temporary noise or limit flow

→ Follow-up restoration of root causes
```

Quick Local Confirmation:

```bash
du -sh /var/log/myapp
```

```bash
find /var/log/myapp -type f -size +500M -exec ls -lh {} \;
```

```bash
tail -n 1000 /var/log/myapp/app.log | grep -i error | wc -l
```

---

## 24. Simple Filebeat Configuration Understanding

Common Filebeat configuration file:

```text
/etc/filebeat/filebeat.yml
```

Check Status:

```bash
systemctl status filebeat
```

Check Logs:

```bash
journalctl -u filebeat -n 100
```

Common Collection Configuration Approach:

```yaml
filebeat.inputs:
  - type: filestream
    id: myapp-log
    enabled: true
    paths:
      - /var/log/myapp/*.log
    fields:
      service: myapp
      env: prod
    fields_under_root: true
```

Notes:

```text
paths
→ Which log files to collect

fields
→ Add fields to logs

fields_under_root
→ Place field at root level
```

---

## 25. Simple Promtail Configuration Understanding

Common Promtail configuration:

```yaml
scrape_configs:
  - job_name: myapp
    static_configs:
      - targets:
          - localhost
        labels:
          job: myapp
          env: prod
          __path__: /var/log/myapp/*.log
```

Notes:

```text
job
→ Log Task Name

env
→ Environment Label

__path__
→ Collection Path
```

Attention:

```text
Loki Medium label Do not design high-base segments

Don't. trace_idI don't know.user_idI don't know.request_id When? label
```

---

## 26. One-Sentence Summary

The core of a log platform is not simply collecting logs, but:

```text
Good pick.

Parsing pairs

Field Steady

Tab Clear

Find out fast.

Yes.

I can keep it.

Control the cost.
```

Log Platform Chain:

```text
Log Generation

→ Agent Collection

→ Parse Purge

→ Transfer Buffer

→ Storage Index

→ Query analysis

→ Alert.

→ Archiving governance
```

Field Design Focus:

```text
timestamp

level

service

env

trace_id

message

status_code

duration_ms
```

Label Design Focus:

```text
Low base figure

Stabilization

Reusable

Don't. request_idI don't know.trace_idI don't know.user_idI don't know.order_id Put in tab
```

Elasticsearch is more suitable for:

```text
Complex full text search

Field Aggregation

Audit search

Multi-dimensional log analysis
```

Loki is more suitable for:

```text
Yuhara's birthday.

Search by Tag

and Prometheus / Grafana Association

Cost relative sensitivity scenario
```

Production Recommendations: /think

```text
Apply logs as structured as possible JSON

Production doesn't start long. debug

Logs must be dissensitized.

It has to be unified. service/env/level Fields

There must be. trace_id or request_id

The log platform itself should be monitored.

Log retention cycle to be graded by value

Logs are designed by window and scale. error Just call the police.
```