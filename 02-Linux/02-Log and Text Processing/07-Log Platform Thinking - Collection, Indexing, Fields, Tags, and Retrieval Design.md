# 07-Log Platform Thinking: Collection, Indexing, Fields, Tags, and Retrieval Design

# Linux # Log Platform # ELK # Loki # Filebeat # Logstash # Promtail # Log Collection # Log Indexing # Log Retrieval # SRE # Observability

---

## Recommended Reading Path

01-Linux Basics and Server Operations/02-Logs and Text Processing/07-Log Platform Thinking: Collection, Indexing, Fields, Tags, and Retrieval Design.md

---

## I. Document Overview

This document outlines the fundamental design concepts for building a log platform.

The previous 01-06 articles focused on:

```text
Single-machine log viewing

Text filtering

awk / grep / sed / find

Nginx log analysis

Log archiving

JSON log processing

Mail notifications

rsyslog forwarding
```

This article shifts the perspective to the log platform, focusing on:

- Why a log platform is needed
- What problems a log platform solves
- The log collection pipeline
- Log field design
- Log tag design
- Log indexing design
- Log retrieval design
- Log retention periods
- Log volume control
- Common components of log platforms
- Basic concepts of ELK / EFK
- Basic concepts of Loki
- Roles of Filebeat / Logstash / Fluent Bit / Promtail
- Key considerations for designing a production log platform

The goal is to:

- Understand that a log platform is more than just collecting logs
- Know how to design the log collection pipeline
- Recognize the importance of log fields and tags
- Distinguish between full-text retrieval and tag-based retrieval
- Comprehend the differences between Elasticsearch and Loki
- Understand how a log platform supports troubleshooting, auditing, alerts, and cost management

---

## II. Why a Log Platform is Needed

Single-machine log checking is suitable for small-scale scenarios.

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

However, in production environments, single-machine logging encounters many issues:

```text
Services are deployed across multiple machines

An application has multiple instances

Logs are scattered across different nodes

Logs are lost after container restarts

Pods may move to different nodes

The call chains between services are complex

It's difficult to determine which machine is affected in case of an error

Local disk storage for logs has limited capacity

Large log volumes make grep operations slow

There is a need for field-based retrieval and aggregation

Long-term auditing and traceability are required

Alerts need to be based on log data
```

The problems that a log platform aims to solve include:

```text
Centralized collection

Unified storage

Unified retrieval

Unified analysis

Unified alerts

Unified retention periods

Unified access control
```

In simple terms:

```text
Single-machine logging addresses "what happened on this machine"

A log platform addresses "what happened across the entire system"
```

---

## III. Core Problems Solved by Log Platforms

Log platforms typically address five main categories of problems.

---

## 1. Troubleshooting

Used to answer questions such as:

```text
Which service is experiencing an error?

Which instance is affected?

When did the error start?

Is the number of errors increasing?

Which users are impacted?

Which API has the most 5xx responses?

Which services does a specific trace_id pass through?

Where did a particular order number fail?
```

---

## 2. Monitoring and Alerts

Used to trigger alerts based on log data:

```text
The number of errors exceeded the threshold within 5 minutes

The proportion of Nginx 5xx responses exceeded the threshold

The application displayed OOM-related keywords

There were too many database connection failures

There were too many login failures

The number of certain exception stacks suddenly increased
```

---

## 3. Auditing and Tracing

Used to track:

```text
Who logged into the server?

Who executed sudo commands?

Who accessed backend APIs?

Who modified configurations?

Who deleted resources?

Which IP performed sensitive operations?
```

---

## 4. Business Analytics

Used to analyze:

```text
The number of API calls

User access paths

Distribution of error-prone orders

Distribution of API response times

Changes in request volumes during specific activities

Variations in errors after a certain version release
```

---

## 5. Stability Management

Used for long-term management tasks:

```text
Top N errors

Top N slow APIs

Top N problematic services

Top N log volumes

Noise log reduction

Elimination of unnecessary debug logs

Management of high-cost indexes
```

---

## IV. Overall Log Platform Architecture

Common log platform processes include:

```text
Log generation
→ Log collection
→ Log parsing
→ Log filtering
→ Log transmissionUse labels just like Prometheus does.

By default, an inverted index is not created for the entire log content.

Main index labels:

Logs can be queried by time and labels.

The storage cost is usually lower.

It is suitable for cloud-native scenarios.

Loki is more suitable for:

Kubernetes logs:

Queries can be performed by namespace/pod/app/container.

It can be used in conjunction with Prometheus metrics for troubleshooting.

It can also be combined with Grafana for unified viewing of metrics and logs.

Scenarios where it is not suitable:

When there are many complex full-text searches.

When a large number of arbitrary field aggregations are required.

When in-depth retrieval of all text fields is needed, similar to Elasticsearch.

---

## IX. Key Differences Between Elasticsearch and Loki

| Comparison Item | Elasticsearch / OpenSearch | Loki |
|---|---|---|
| Retrieval Method | Strong full-text retrieval | Strong label retrieval |
| Indexing Method | Creates indexes for a large number of fields | Mainly indexes labels |
| Storage Cost | Relatively higher | Usually lower |
| Query Capabilities | Strong in field aggregation and full-text search | Filters by labels and log content |
| Typical Interfaces | Kibana | Grafana |
| Suitable Scenarios | Complex log analysis, auditing, searching | Cloud-native logging, joint troubleshooting of metrics and logs |
| Risk Points | Index bloat, high cost | Unreasonable label design can lead to index explosion |

In one sentence:

Elasticsearch is more like a log search engine.

Loki is more like a time-series system for logged data with labels.

---

## X. Log Field Design

Field design is very important in a logging platform.

If the fields are poorly organized, it will result in:

Difficulties in retrieval.

Difficulties in aggregation.

Difficulties in setting up alerts.

Field conflicts.

Index bloat.

Inability to reuse logs.

Inability to analyze data consistently across different services.

Recommended basic fields:

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

---

## 1. Time Field

Recommended field:

timestamp

Requirements:

It must be accurate.

The time zone must be consistent.

It is best to use the ISO 8601 format.

The time on both the collection side and the server side should be synchronized.

Do not rely solely on the collection time.

Example:

```json
{
  "timestamp": "2026-04-25T10:00:00+08:00"
}
```

---

## 2. Log Level Field

Recommended field:

level

Common values:

debug

info

warn

error

fatal

It is recommended to use either lowercase or uppercase consistently.

For example, using lowercase consistently:

```json
{
  "level": "error"
}
``

It is not recommended to mix different cases on the same platform:

ERROR

Error

error

ERR

---

## 3. Service Field

Recommended field:

service

Example:

```json
{
  "service": "resume-api"
}
```

Uses:

Filter logs by service.

Count the number of service errors.

Set up service-level alerts.

Calculate service-level costs.

---

## 4. Environment Field

Recommended field:

env

Common values:

dev

test

staging

pre

prod

Uses:

Distinguish between test and production logs.

Avoid misinterpreting environment-related logs.

Production alerts should only be triggered for data in the prod environment.

---

## 5. Request Linkage Field

Recommended fields:

trace_id

request_id

Uses:

Track the multiple services that a single request passes through.

Quickly locate cross-service issues.

Use in conjunction with traceability tools for troubleshooting.

Example:

```json
{
  "trace_id": "abc123",
  "request_id": "req-20260425-001"
}
```

---

## 6. HTTP Field

Recommended fields:

method

path

status_code

duration_ms

client_ip

user_agent

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

## XI. Log Label Design

Labels are often used for quick filtering and aggregation.

Common labels:

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

Principles for label design:

Low cardinality.

Stability.

Reusability.

Business relevance.

Do not include unique values in labels.

---

## 1. What is Low Cardinality?

Low cardinality means that a field has a limited number of possible values.

Suitable as labels:

env
→ prod / test / dev

service
→ resume-api / user-api / order-api

namespace
→ defaultpaths:→ Which log files should be collected?

fields
→ Add fields to the logs.

fields_under_root
→ Place fields at the root level.

---

## Twenty-Five: Understanding Basic Promtail Configurations

Common Promtail configurations:

```yaml
scrape_configs:
  - job_name: myapp
    staticconfigs:
      - targets:
          - localhost
        labels:
          job: myapp
          env: prod
          __path__: /var/log/myapp/*.log
```

Explanation:

```text
job
→ Log task name.

env
→ Environment tag.

__path__
→ Collection path.
```

Note:

```text
In Loki, avoid designing labels as high- cardinality fields.

Do not use trace_id, user_id, or request_id as labels.
```

---

## Twenty-Six: A One-Sentence Summary

The core of a log platform is not just to collect logs, but to ensure that they are:

```text
accurately collected,

properly parsed,

fields are stable,

labels are clear,

searchable quickly,

alerts are accurate,

retained properly,

and costs are controlled.
```

The log platform process includes:

```text
Log generation

→ Agent collection

→ Parsing and cleaning

→ Transmission and buffering

→ Storage and indexing

→ Querying and analysis

→ Alert notifications

→ Archival management
```

Key considerations for field design:

```text
timestamp, level, service, env, trace_id, message, status_code, duration_ms
```

Key considerations for label design:

```text
Low cardinality, stability, reusability.

Avoid including request_id, trace_id, user_id, or order_id in labels.
```

Elasticsearch is better suited for:

```text
Complex full-text searches,

field aggregation and analysis,

audit searches,

multi-dimensional log analysis.
```

Loki is better suited for:

```text
Cloud-native log management,

label-based searches,

integration with Prometheus/Grafana,

cost-sensitive scenarios.
```

Production recommendations:

```text
Use structured JSON for application logs.

Do not enable debug mode permanently in production.

Log data must be anonymized.

Standardize service/env/level fields.

Ensure trace_id or request_id is included.

Monitor the log platform itself regularly.

Design log retention periods based on value.

Set up alerts with appropriate thresholds to avoid unnecessary alarms.
```