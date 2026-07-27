# 08-Production Log Governance: Level Standardization, Masking, Sampling, Retention Periods, and Cost Control

# Linux #Log Governance #Log Standards #Log Masking #Log Sampling #Log Retention Periods #Log Cost Control #SRE #Observability #Stability Governance

---

## Recommended Reading Path

01-Linux Basics and Host Operations/02-Logs and Text Processing/08-Production Log Governance: Level Standardization, Masking, Sampling, Retention Periods, and Cost Control.md

---

## I. Document Overview

This document outlines the core principles and implementation methods for log governance in production environments.

Articles 01 to 07 have already covered:

```text
Log Viewing

grep / awk / sed / find / xargs

Nginx access.log Analysis

Log Archiving and logrotate

JSON Log Processing

Mail Notifications

rsyslog Forwarding

Log Platform Collection, Field, Tag, and Index Design
```

This article delves further into more critical aspects of production environment log management:

- How to standardize log levels
- What information should be logged
- What information should not be logged
- Why long-term debugging is not feasible in production
- How to mask logs
- Which sensitive fields must not be logged in plaintext
- How to sample high-frequency logs
- How to reduce noise in health check logs
- How to design log retention periods
- How to control the cost of log platforms
- How to troubleshoot sudden increases in log volume
- How to establish standardized and closed-loop processes for log governance

The goal is:

To understand that log governance is not simply about “logging more”

→ To establish production-level log standardization

→ To identify sensitive and high-risk logs

→ To design log masking rules

→ To control high-frequency, low-value logs

→ To design retention periods based on log value

→ To manage log platform costs from a financial perspective

→ To embrace SRE and stability governance perspectives when managing logs
---

## II. Why Production Log Governance is Needed

Logs are essential for troubleshooting, but more logs do not necessarily equate to better performance.

Unmanaged logs can lead to numerous issues:

```text
Infinite log volume growth

Disk space exhaustion

Log platform index expansion

Slower query times

Increased alarm noise

Overwhelming of actual errors

Leakage of sensitive information

Increased audit risks

Out-of-control log costs

Both development and operations teams are reluctant to review logs
```

Production log governance aims to address these challenges:

```text
Only record necessary logs

Avoid logging unnecessary information

Do not log sensitive data in plaintext

Reduce noise from high-frequency, low-value logs

Ensure critical error logs are searchable

Design tiered log retention periods

Keep log platform costs under control
```

In summary:

```text
The goal of log governance is not to reduce the amount of logs but to increase their value density.
```

---

## III. General Principles for Production Log Governance

The following principles should guide production log governance:

```text
Usefulness

Accuracy

Structure

Searchability

Correlatability

Alertability

Audibility

Cost-effectiveness

No sensitive information leakage
```

Breaking these down further:

```text
Usefulness
→ Logs should aid in troubleshooting, auditing, analysis, and retrospective review

Accuracy
→ Time, level, service name, status code, and error details must be accurate

Structure
→ Use JSON or standardized fields wherever possible

Searchability
→ Logs should be searchable by service, environment, trace_id, or level

Correlatability
→ Requests should be linked through trace_id or request_id

Alertability
→ Errors, 5xx errors, timeouts, OOMs, etc., should trigger alerts and statistics

Audibility
· Login activities, permission changes, configuration modifications, etc., should be traceable

Cost-effectiveness
→ Prevent meaningless logs from overwhelming the platform

No sensitive information leakage
→ Passwords, tokens, Cookies, keys, etc., must not appear in logs in plaintext
```

---

## IV. Log Level Standardization

---

## Scenario 1: Common Log Levels

Common log levels in production include:

```text
DEBUG
→ Debugging information

INFO
→ Normal operation information

WARN
→ Potential risks or recoverable exceptions

ERROR
→ Clear errors that require attention

FATAL
→ Severe errors that may cause the process or service to fail
```

Lowercase formats are also commonly used:

```text
debug

info

warn

error

fatal
```

It is recommended to adopt a consistent style. For example, always use lowercase:

```json
{
  "level": "error"
}
```

Mixed usage should be avoided:

```text
ERROR

Error

error

ERR
```

This can lead to:

```text
Inconsistent search results

Inaccurate statistics

Complicated alert rules

Confusing dashboard fields
`````json
{
  "port": 8080
}
```awk '$7 !~/(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/ {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | headIs there any situation where errors are mistakenly classified as information?

Is there a case where normal business failures are excessively labeled as errors?```bash
awk '$7 != "/health" {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$7 !~ /^\/(health|healthz|ready|readiness|liveness|metrics)$/ {print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Analysis of Static Resource Logs

```bash
awk '$7 ~ /\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2}$/ {count++} END {print count}' /var/log/nginx/access.log
```

```bash
awk '$7 !~ /(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/ {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Inspection of Sensitive Information

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

## Examples of Sed Masking

```bash
sed -E 's/([0-9]{3})[0-9]{4}([0-9]{4})/\1****\2/g' app.log
```

```bash
sed -E 's/(Authorization: Bearer )[A-Za-z0-9._-]+/\1***/g' app.log
```

---

## Examples of jq Masking

```bash
jq 'del(.password, .token)' app.json
```

```bash
jq '.password="***" | .token="***"' app.json
```

---

## Analysis of Duplicated Log Entries

```bash
tail -n 1000 /var/log/myapp/app.log | awk '{print $0}' | sort | uniq -c | sort -nr | head
```

```bash
grep -i "error" /var/log/myapp/app.log | sort | uniq -c | sort -nr | head
```

---

## Twenty, A Summary in One Sentence

The core of production log management is not to "collect more logs," but rather:

```text
Ensure critical logs are complete.

Limit the amount of general logs.

Mask sensitive information.

Reduce noise from high-frequency logs.

Retain low-value logs briefly.

Preserve high-value logs.

Keep log costs under control.
```

Recommended log levels are:

```text
DEBUG
→ Disabled by default in production; use temporarily with time limits.

INFO
→ Record normal and critical events.

WARN
→ Log recoverable exceptions and risk signals.

ERROR
→ Record clear failures.

FATAL
→ Record serious errors that prevent service operation.
``

Key areas for log masking include:

```text
Passwords, tokens, cookies, authorization information, phone numbers, ID cards, bank card details, AccessKeys/SecretKeys, private keys, and certificates.
```

Log sampling recommendations are:

```text
Collect all ERROR logs.

Retain most WARN logs.

Selectively retain INFO logs.

Do not collect DEBUG logs by default.

Reduce noise from health checks and static resource requests.

Sample successful requests, but collect all failed requests.
``

Suggested log retention periods are:

```text
Local logs should be retained for a short time.

Platform logs should be kept for a medium period.

High-value logs should be archived for a long time.

Low-value logs should be discarded after a short period.

Audit logs must be stored according to regulatory requirements.

Cold data should be moved to object storage for archiving.
```

Cost control measures include:

```text
Limit the use of DEBUG logs.

Filter out low-value logs.

Control fields with high cardinality.

Avoid duplicate log collections.

Reduce the retention period of low-value logs.

Implement hot and cold data separation.

Track log volume and costs by service.
``

Production recommendations are:

```text
Use a unified format for timestamps, levels, services, environments, trace IDs, and messages.

 Structure application logs in JSON format.

 Do not store sensitive fields in plain text.

 Avoid keeping DEBUG logs enabled indefinitely.

 Do not include trace IDs, user IDs, or order IDs in Loki labels.

 Design log alerts based on time windows, thresholds, and impact levels.

 Integrate log management into release checks and post