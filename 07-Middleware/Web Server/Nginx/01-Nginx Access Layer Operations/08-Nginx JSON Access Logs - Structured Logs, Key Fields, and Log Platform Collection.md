# 08-Nginx JSON Access Logs: Structured Logs, Key Fields, and Log Platform Collection

# Nginx # JSON Logs # Access Logs # Structured Logs # Log Platforms # ELK # Loki # Filebeat # Promtail # Observability # Operations # SRE

---

## Recommended Reading Path

07-Middleware/Web Servers/Nginx/01-Nginx Access Layer Operations/08-Nginx JSON Access Logs: Structured Logs, Key Fields, and Log Platform Collection.md

---

## I. Document Overview

This document outlines the design, configuration, validation of Nginx JSON access logs, as well as methods for collecting them using log platforms.

Key points include:

- Why use JSON access logs
- Issues with Nginx's default access.log format
- Basics of `log_format`
- The role of `escape=json`
- Design principles for JSON log fields
- Client IP fields
- Request fields
- Status code fields
- Time consumption fields
- `upstream` fields
- Header fields
- `trace_id`/`request_id` fields
- Configuration of JSON access logs
- How to validate JSON format validity
- How to analyze JSON logs using jq
- Methods for tracking 5xx errors, slow requests, top URLs, and top IPs
- Approach to collecting JSON logs with Filebeat
- Tag design considerations for Promtail/Loki
- Common mistakes in JSON logging
- Best practices for production environments

This document is part of the Nginx Access Layer Operations series, Chapter 08.

Objectives:

```text
- Convert Nginx access.log into structured JSON format
- Understand the meaning of each key field
- Analyze Nginx JSON logs using jq
- Design fields suitable for ELK/Loki/log platforms
- Prevent issues such as incorrect JSON format, field type conflicts, and high基数 field problems
```

---

## II. Why Use JSON Access Logs

Nginx's default access log is usually in plain text format.

For example:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123 "-" "Mozilla/5.0"
```

While this format is suitable for quick human review, it poses challenges when used in log platforms:

```text
- Fields require additional parsing
- Different fields are separated by spaces, making errors prone to occur
- The User-Agent field contains spaces
- The Referer field may be empty
- The `upstream` field may contain multiple values
- Field types are not consistent
- Log platform retrieval and aggregation become difficult
- Alarm rules require prior field parsing
- High costs for later maintenance
```

Advantages of JSON logs:

```text
- Naturally structured
- Clear fields
- Easy to parse by log platforms
- Convenient for field-based searches
- Useful for tracking status codes and slow requests
- Helpful for analyzing `upstream` time consumption
- Facilitate association with `trace_id`/`request_id`
- Easy integration with ELK/Loki/ClickHouse/SIEM
```

In short:

```text
Plain access.log is suitable for human reading, while JSON access.log is better suited for platform analysis and automated management.
```

---

## III. Limitations of Nginx's Default access.log Format

The default combined format is similar to:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';
```

Problems include:

```text
- The `request` field is a composite field, mixing request method, URL, and protocol
- The User-Agent field contains spaces, leading to potential errors when processed column by column
- The format becomes unstable when Referer or User-Agent are empty
- Fields cannot be stored directly based on their type
- Log platforms require additional parsing using tools like grok or regular expressions
- Changes to the `log_format` can easily break existing parsing rules
```

For example, analyzing logs with awk requires:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

This command assumes that the status code is always in the 9th column. However, changes to the log format may alter this position.

JSON logs eliminate the need for such strict column dependencies.

---

## IV. Basics of Nginx `log_format`

---

## Scenario 1: What is `log_format`?

`log_format` is used to define the output format of Nginx access logs.

Basic example:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
```json
"upstream_response_time":"$upstream_response_time",
"referer":"$http_referer",
"user_agent":"$http_user_agent"
};
```## Ten: Adding Trace ID/Request ID Passthrough

---

## Scenario 18: Recording the Trace ID Sent by the Client

If the upstream or client sends:

```text
X-Request-ID

X-Trace-ID

traceparent
```

It can be recorded as follows in Nginx:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"request_id":"$request_id",'
    '"http_x_request_id":"$http_x_request_id",'
    '"http_x_trace_id":"$http_x_trace_id",'
    '"traceparent":"$http_traceparent",'
    '"remote_addr":"$remote_addr",'
    '"method":"$request_method",'
    '"request_uri":"$request_uri",'
    '"status":$status,'
    '"request_time":"$request_time"'
    '}';
```

Explanation:

```text
$http_x_request_id
→ Request header X-Request-ID

$http_x_trace_id
→ Request header X-Trace-ID

$http_traceparent
→ W3C trace context request header traceparent
```

---

## Scenario 19: Passing the Request ID to the Backend

Configuration:

```nginx
proxy_set_header X-Request-ID $request_id;
```

Complete example:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Request-ID $request_id;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Function:

```text
The backend logs can include the same X-Request-ID

This facilitates linking Nginx access.log files with application logs.
```

---

## Scenario 20: Prioritizing the X-Request-ID Sent by the Upstream

If an `X-Request-ID` has already been provided by the upstream gateway, it should be passed through directly:

```nginx
proxy_set_header X-Request-ID $http_x_request_id;
```

However, note that:

```text
If the client can directly access Nginx, the X-Request-ID may be forged

It is more secure to generate or validate it at a trusted entry point.
```

---

## Eleven: JSON Log Format Validation

---

## Scenario 21: Viewing the Latest JSON Logs

```bash
tail -n 5 /var/log/nginx/example.access.json.log
```

---

## Scenario 22: Using jq to Verify if JSON is Valid

To view a single line:

```bash
tail -n 1 /var/log/nginx/example.access.json.log | jq .
```

To verify the last 100 lines:

```bash
tail -n 100 /var/log/nginx/example.access.json.log | jq . >/dev/null
```

If there is invalid JSON,jq will report an error.

---

## Scenario 23: Identifying Unparseable Log Lines

Check each line individually:

```bash
while read line; do echo "$line" | jq . >/dev/null || echo "$line"; done < /var/log/nginx/example.access.json.log
```

Explanation:

```text
Lines that can be parsed will not be displayed.

Unparseable lines will be printed out.
```

---

## Scenario 24: Ensuring Each Line Contains a Complete JSON Object

```bash
head -n 3 /var/log/nginx/example.access.json.log
```

Requirement:

```text
Each line must contain a complete JSON object.

No single log should span multiple lines.
```

Log platforms typically expect JSON logs to be structured in such a way that:

```text
Each line contains one log entry.

Each line represents a complete JSON object.
```

---

## Twelve: Using jq to Analyze Nginx JSON Logs

---

## Scenario 25: Formatting and Viewing Logs

```bash
tail -n 1 /var/log/nginx/example.access.json.log | jq .
```

---

## Scenario 26: Extracting Status Codes

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status'
```

Counting status codes:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

---

## Scenario 27: Counting 5xx Status Codes

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .status' | wc -l
```

Viewing 5xx logs:

```bash
cat /var/log/nginx/example.access.json.log | jq 'select(.status >= 500)'
```

Outputting core fields:

```bash
cat /var/log/nginx/example.access.json.log |cat /var/log/nginx/example.access.json.log | jq -r '.upstream_status' | sort | uniq -c | sort -nr{job="nginx-access", env="prod"} | json | request_id="abc123"
```

Note:

It is not recommended to use `request_id` as a label, but it can be used for querying in log content.```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```