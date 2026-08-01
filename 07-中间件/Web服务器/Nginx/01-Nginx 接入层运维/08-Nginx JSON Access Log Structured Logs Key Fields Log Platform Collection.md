# 08-Nginx JSON Access Logs: Structured Logs, Key Fields, and Log Platform Collection

#Nginx #JsonLog #AccessLog #StructuredLog #LogPlatform #ELK #Loki #Filebeat #Promtail #Observation #Transport #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operations/08-Nginx JSON Access Logs: Structured Logs, Key Fields, and Log Platform Collection.md

---

## One: Document Overview

This document organizes the design, configuration, verification, and log platform collection methods for Nginx JSON access logs.

This article focuses on:

- Why use JSON access logs
- Issues with Nginx default access.log
- `log_format` foundation
- `escape=json` purpose
- JSON log field design
- Client IP field
- Request field
- Status code field
- Latency field
- Upstream field
- Header field
- trace_id / request_id field
- JSON access_log configuration
- How to verify JSON format validity
- How to analyze JSON logs with jq
- How to statistics 5xx, slow requests, Top URLs, Top IPs
- Filebeat JSON log collection approach
- Promtail / Loki label design philosophy
- Common JSON log errors
- Production environment considerations

This article is the 08th in the Nginx Access Layer Operations series.

This article's objectives:

```text
I can. Nginx access.log Change to structure JSON

→ Understands the meaning of each key field

→ It works. jq Analysis Nginx JSON Log

→ Yes. ELK / Loki / Log Platform Design Fields

→ It can be avoided. JSON Problems with format, conflict of field type, high-base segment explosions, etc.
```

---

## Two: Why Need JSON Access Logs

Nginx default access logs are typically plain text format.

Example:

```text
10.0.0.5 - - [25/Apr/2026:10:00:01 +0800] "GET /api/login HTTP/1.1" 200 123 "-" "Mozilla/5.0"
```

This format is suitable for manual quick viewing, but has issues in log platforms:

```text
Fields need additional resolution

Different fields split by space, easily error-prone

User-Agent Include spaces

Referer Could be empty.

upstream Fields may contain multiple values

Field type unstable

The log platform is not easy to retrieve and aggregate

The alarm rules need to be deciphered first.

High governance costs
```

JSON log advantages:

```text
Natural structure

Fields Clear

Enable log platform resolution

Easy to search by field

Easy to count.

Easy to count slow requests

Statistics-friendly upstream Time consuming

Facilitation trace_id / request_id Association

Easy access. ELK / Loki / ClickHouse / SIEM
```

One-sentence understanding:

```text
Normal access.log It's good for people to see.JSON access.log Better suited to platform analysis and automated governance.
```

---

## Three: Limitations of Nginx Default access.log

Default combined format is similar to:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';
```

Issues:

```text
$request A grouping field

Method of request,URL. . . . . . . . . . . . . . . ..and mixed.

User-Agent With spaces, it's easy to make mistakes by row

RefererI don't know.User-Agent Unstable for empty time format

Cannot store directly by field type

Log Platform Needs grok Or regular resolution.

♪ Once log_format Changed. Parsing rules fail.
```

Example using awk to analyze:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

This command depends on:

```text
The state code is the first. 9 Columns
```

But if log format changes, field positions may become inaccurate.

JSON logs can avoid this strong column number dependency issue.

---

## Four: Nginx log_format Foundation

---

## Scenario 1: What is log_format

`log_format` is used to define Nginx access.log output format.

Basic example:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';
```

Using format:

```nginx
access_log /var/log/nginx/access.log main;
```

Meaning:

```text
main
→ Log Format Name

access_log
→ Use main Format Writing access.log
```

---

## Scenario 2: View Current log_format

```bash
grep -R "log_format" /etc/nginx/
```

Check access_log used format:

```bash
grep -R "access_log" /etc/nginx/
```

Check full configuration:

```bash
nginx -T | grep -n "log_format" -A 30
```

---

## Five: escape=json's Purpose

---

## Scenario 3: Why Need escape=json

If JSON strings contain:

```text
Double quotes

Backslash

Line Break

Special Characters
```

They need to be properly escaped.

Example User-Agent may contain special characters:

```text
Mozilla/5.0 "test"
```

Without escaping, JSON may become invalid.

Nginx supports:

```nginx
log_format json_log escape=json '...';
```

Purpose:

```text
Do the contents of the variable JSON String conversion

Avoid double quotes, backslash etc. JSON Format
```

Production recommendation:

```text
JSON log_format Use as much as possible escape=json
```

---

## Six: JSON Log Field Design Principles

When designing Nginx JSON access.log, fields should meet:

```text
It's a barrier.

Statistics

Retrievable

Can relate

You can call the cops.

It controls costs.
```

Recommended field grouping:

```text
Time field
→ time

Client field
→ remote_addrI don't know.realip_remote_addrI don't know.xff

Request Fields
→ hostI don't know.methodI don't know.uriI don't know.request_uriI don't know.argsI don't know.scheme

Response Fields
→ statusI don't know.body_bytes_sentI don't know.bytes_sent

Time-consuming fields
→ request_timeI don't know.upstream_response_timeI don't know.upstream_connect_timeI don't know.upstream_header_time

upstream Fields
→ upstream_addrI don't know.upstream_status

Header Fields
→ refererI don't know.user_agent

Link Fields
→ request_idI don't know.trace_id

Nginx Fields
→ server_nameI don't know.server_port
```

Field naming suggestions:

```text
Unified lowercase

Use Underline

Not the same field. clientIpI'll call later. client_ip

Do not mix Chinese with English

Field type as stable as possible
```

---

## Seven: Recommended JSON log_format

---

## Scenario 4: Basic JSON Log Format

Recommend writing in `http` block:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"realip_remote_addr":"$realip_remote_addr",'
    '"xff":"$http_x_forwarded_for",'
    '"request_id":"$request_id",'
    '"host":"$host",'
    '"server_name":"$server_name",'
    '"server_port":"$server_port",'
    '"scheme":"$scheme",'
    '"method":"$request_method",'
    '"uri":"$uri",'
    '"request_uri":"$request_uri",'
    '"args":"$args",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"bytes_sent":$bytes_sent,'
    '"request_length":$request_length,'
    '"request_time":"$request_time",'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_connect_time":"$upstream_connect_time",'
    '"upstream_header_time":"$upstream_header_time",'
    '"upstream_response_time":"$upstream_response_time",'
    '"referer":"$http_referer",'
    '"user_agent":"$http_user_agent"'
    '}';
```

Using:

```nginx
access_log /var/log/nginx/example.access.json.log access_json;
```

Explanation:

```text
statusI don't know.body_bytes_sentI don't know.bytes_sentI don't know.request_length
→ Usually output is a number

request_timeI don't know.upstream_*_time
→ Suggest string first, avoid upstream Empty or more upstream Unstandard numbers are generated from time to time

upstream_status
→ Maybe. 200Or maybe. 502Or maybe. 200, 200Or maybe. -

upstream_addr
→ Could be a single address, could be multiple addresses, could be empty.
```

---

## Scenario 5: Why Some Fields Use Strings

Example:

```nginx
"upstream_response_time":"$upstream_response_time"
```

Do not easily write as:

```nginx
"upstream_response_time":$upstream_response_time
```

Reason:

```text
If not upstreamIt could be. -

If you try again, the value could be 0.010, 0.020

It's not legal. JSON Numbers

It leads to the whole line. JSON Could not initialise Bonobo
```

Therefore, production recommendation:

```text
upstream Relevant fields enter the log platform first as strings

Subsequent resolution or conversion by log platform
```

---

## Eight: Complete server Configuration Example

---

## Scenario 6: Nginx JSON access.log Example

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;

    keepalive 64;
}

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/example.access.json.log access_json;
    error_log  /var/log/nginx/example.error.log warn;

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

Check:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Request verification:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

View logs:

```bash
tail -n 5 /var/log/nginx/example.access.json.log
```

---

## Nine: Key Field Explanations

---

## Scenario 7: time

Field:

```json
"time":"$time_iso8601"
```

Explanation:

```text
Request Time

ISO 8601 Format

Fits log platform recognition time fields
```

Example:

```json
"time":"2026-04-25T10:00:01+08:00"
```

---

## Scenario 8: remote_addr

Field:

```json
"remote_addr":"$remote_addr"
```

Explanation:

```text
Nginx Current recognized client IP
```

If configured with real_ip:

```text
remote_addr Could be a real client. IP
```

If no real_ip configured and previous SLB / CDN:

```text
remote_addr Could be the last jump agent. IP
```

---

## Scenario 9: realip_remote_addr

Field:

```json
"realip_remote_addr":"$realip_remote_addr"
```

Explanation:

```text
real_ip Replace remote_addr Previous original jump IP
```

Suitable for troubleshooting:

```text
Which request came from? SLB / CDN / WAF

real_ip Entry into force

Does the proxy link meet expectations?
```

---

## Scenario 10: xff

Field:

```json
"xff":"$http_x_forwarded_for"
```

Explanation:

```text
Original X-Forwarded-For Request Header
```

Suitable for troubleshooting:

```text
Multiple proxy links

Real IP Whether it's passed through.

Is there a forgery? XFF

Is the chain abnormally long?
```

---

## Scenario 11: request_id

Field:

```json
"request_id":"$request_id"
```

Explanation:

```text
Nginx Request generated for request ID

Could be used for single request tracking
```

Production recommendation:

```text
If the company already has trace_idshould, to the extent possible, be generated uniformly or through the portal. ]

request_id I can do it. Nginx Layer-Associated Fields
```

---

## Scenario 12: method, uri, request_uri

Field:

```json
"method":"$request_method"
```

```json
"uri":"$uri"
```

```json
"request_uri":"$request_uri"
```

Differences:

```text
$request_method
→ GET / POST / PUT / DELETE Wait.

$uri
→ Normalized URINone query string

$request_uri
→ Original request URI, usually includes query string
```

Example:

```text
Requests:/api/users?id=1

$uri
→ /api/users

$request_uri
→ /api/users?id=1
```

Production recommendation:

```text
Statistical interface path priority uri or route

Check the full request. request_uri

If query string Sensitive information not recommended for long-term complete record
```

---

## Scenario 13: status

Field:

```json
"status":$status
```

Explanation:

```text
HTTP Response status code
```

Common uses:

```text
Statistics 2xx / 3xx / 4xx / 5xx

Calculating Error Rate

Generate alarm rules

Check. 502 / 504 / 499
```

---

## Scenario 14: request_time

Field:

```json
"request_time":"$request_time"
```

Explanation:

```text
From Nginx Received request start to the total time the response sent complete

Unit is seconds.
```

Example:

```json
"request_time":"0.123"
```

Common uses:

```text
Analysis of slow requests

Statistics P95 / P99

Positioning high-time interface

Whether the client is slow or the back.
```

---

## Scenario 15: upstream_response_time

Field:

```json
"upstream_response_time":"$upstream_response_time"
```

Explanation:

```text
Nginx From Backend upstream Time taken to receive the response

Unit is seconds.
```

Note:

```text
If the request is not forwarded to upstreamMaybe. -

If you try again, there may be multiple values.

For example:0.010, 0.023
```

---

## Scenario 16: upstream_addr

Field:

```json
"upstream_addr":"$upstream_addr"
```

Explanation:

```text
The backend to which this request is actually forwarded
```

Example:

```json
"upstream_addr":"10.0.0.21:8080"
```

When retrying, it may be:

```json
"upstream_addr":"10.0.0.21:8080, 10.0.0.22:8080"
```

Common uses: /think

```text
Which back end did the request reach?

Check one. upstream Node Abnormal

Error distribution of statistical backend examples
```

---

## Scenario 17: upstream_status

Fields:

```json
"upstream_status":"$upstream_status"
```

Description:

```text
Backend upstream Return status code
```

Difference between ```text
$status
→ Nginx Final state code returned to client

$upstream_status
→ Backend upstream Return to Nginx Status code
``` and `$status`:

```text
Nginx Go back on your own. 413 / 403 / 404

At this point, upstream_status Could be empty or -
```

In some cases:

```text
X-Request-ID

X-Trace-ID

traceparent
```

---

## Ten. Adding trace_id / request_id Propagation

---

## Scenario 18: Recording Client-Provided Trace ID

If upstream or client provides:

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

You can record:

```text
$http_x_request_id
→ Request Header X-Request-ID

$http_x_trace_id
→ Request Header X-Trace-ID

$http_traceparent
→ W3C trace context Request Header traceparent
```

Description:

```nginx
proxy_set_header X-Request-ID $request_id;
```

---

## Scenario 19: Passing Request ID to Backend

Configuration:

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

Complete Example:

```text
Backend records the same X-Request-ID

Easy to use Nginx access.log Link to Application Log
```

Purpose:

```nginx
proxy_set_header X-Request-ID $http_x_request_id;
```

---

## Scenario 20: Prioritize Upstream-Provided X-Request-ID

If upstream gateway already provides `X-Request-ID`, you can directly pass it through:

```text
If the client has direct access NginxI don't know.X-Request-ID It could be forged.

A more standardized approach is to create or launder a credible entry point
```

But note:

```bash
tail -n 5 /var/log/nginx/example.access.json.log
```

---

## Eleven. JSON Log Format Validation

---

## Scenario 21: View Latest JSON Log

```bash
tail -n 1 /var/log/nginx/example.access.json.log | jq .
```

---

## Scenario 22: Use jq to Validate JSON Validity

Check one line:

```bash
tail -n 100 /var/log/nginx/example.access.json.log | jq . >/dev/null
```

Validate last 100 lines:

```bash
while read line; do echo "$line" | jq . >/dev/null || echo "$line"; done < /var/log/nginx/example.access.json.log
```

If invalid JSON, jq will report errors.

---

## Scenario 23: Find Unparseable Log Lines

Line-by-line inspection:

```text
能解析的行不输出

不能解析的行会被打印出来
```

Description:

```bash
head -n 3 /var/log/nginx/example.access.json.log
```

---

## Scenario 24: Check if Each Line is a Single JSON

```text
每一行都是一个完整 JSON 对象

不要一条日志跨多行
```

Requirements:

```text
一行一条日志

一行一个 JSON
```

Log platforms typically expect:

```bash
tail -n 1 /var/log/nginx/example.access.json.log | jq .
```

---

## Twelve. Using jq to Analyze Nginx JSON Logs

---

## Scenario 25: Format View Logs

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status'
```

---

## Scenario 26: Extract Status Codes

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

Statistical Status Codes:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .status' | wc -l
```

---

## Scenario 27: Count 5xx Errors

```bash
cat /var/log/nginx/example.access.json.log | jq 'select(.status >= 500)'
```

View 5xx Logs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .remote_addr, .method, .request_uri, .status, .upstream_addr, .upstream_status] | @tsv'
```

Output Core Fields:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | [.time, .remote_addr, .request_uri, .request_time] | @tsv'
```

---

## Scenario 28: Count 499 Requests

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | .uri' | sort | uniq -c | sort -nr | head
```

Count 499 URLs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.remote_addr' | sort | uniq -c | sort -nr | head
```

---

## Scenario 29: Count Most Frequent IPs

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.uri' | sort | uniq -c | sort -nr | head
```

---

## Scenario 30: Count Most Frequent URLs

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .uri' | sort | uniq -c | sort -nr | head
```

---

## Scenario 31: Count TopN 5xx URLs

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .remote_addr, .request_uri, .status, .request_time, .upstream_response_time] | @tsv'
```

---

## Scenario 32: Slow Request Analysis

Since `request_time` is a string, you can use `tonumber` to convert.

Filter requests greater than 1 second:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | .uri' | sort | uniq -c | sort -nr | head
```

Count slow request URLs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

---

## Scenario 33: Upstream Node Error Analysis

Count 5xx corresponding upstream:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_status' | sort | uniq -c | sort -nr
```

Count upstream_status:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.host' | sort | uniq -c | sort -nr
```

---

## Scenario 34: Request Count by Host

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.user_agent | test("bot|spider|crawler"; "i")) | [.remote_addr, .uri, .user_agent] | @tsv'
```

---

## Scenario 35: Analyze Crawlers by User-Agent

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.user_agent | test("bot|spider|crawler"; "i")) | .remote_addr' | sort | uniq -c | sort -nr | head
```

Count suspected crawler IPs:

```text
time

remote_addr

host

method

uri

status

request_time

upstream_addr

upstream_status

user_agent
```

---

## Thirteen. Log Platform Collection Strategy

---

## Scenario 36: Why Log Platforms Prefer JSON

After collecting JSON logs, log platforms can directly obtain fields:

```text
It doesn't have to be complicated. grok

Don't press space.

Fields are more stable

More accurate search.

It's easier to aggregate.

The rules are clearer.
```

Benefits:

```text
status >= 500

request_time > 1

host = example.com

uri = /api/login

upstream_addr = 10.0.0.21:8080
```

Example:

```text
time
→ date

remote_addr
→ ip or keyword

realip_remote_addr
→ ip or keyword

xff
→ keyword or text

host
→ keyword

server_name
→ keyword

method
→ keyword

uri
→ keyword

request_uri
→ keyword or text

status
→ integer

body_bytes_sent
→ long

bytes_sent
→ long

request_length
→ long

request_time
→ float

upstream_addr
→ keyword

upstream_status
→ keyword

upstream_response_time
→ keyword or textPurge it if necessary float

referer
→ keyword or text

user_agent
→ textand can be parsed into browser, system, device fields
```

---

## Scenario 37: Recommended Field Types for Elasticsearch / OpenSearch

Recommended field types:

```text
Field type stable

It's not a number. It's a string.

upstream Multivalue fields do not rotate directly float
```

Note:

```text
job

env

service

host

server_name

status_class

namespace

container
```

---

## Scenario 38: Loki Label Design Recommendations

In Loki, do not use all fields as labels.

Suitable as labels:

```text
request_id

trace_id

remote_addr

xff

request_uri

user_agent

Full uri

args
```

Not suitable as labels:

```text
These fields are too high.

It'll lead to... label Explosion

Writing and query costs increase
```

Reason:

```text
Low-base numbers segment label

High-base digital segments left to search in log contents
```

Recommendation:

```yaml
filebeat.inputs:
  - type: filestream
    id: nginx-access-json
    enabled: true
    paths:
      - /var/log/nginx/*.access.json.log

    parsers:
      - ndjson:
          target: ""
          overwrite_keys: true
          add_error_key: true

    fields:
      service: nginx
      log_type: access
      env: prod
    fields_under_root: true

output.elasticsearch:
  hosts: ["http://10.0.0.10:9200"]
  index: "nginx-access-%{+yyyy.MM.dd}"
```

---

## Fourteen. Filebeat JSON Log Collection Example

---

## Scenario 39: Filebeat filestream JSON Collection

Example configuration:

```text
ndjson
→ One in a row. JSON

target: ""
→ JSON Field to Root Level

add_error_key
→ JSON Add error field when parsing failed

fields
→ Add Custom Fields

index
→ Write by Day nginx-access Index
```

Description:

```bash
systemctl status filebeat
```

---

## Scenario 40: Filebeat Post-Collection Check

Check Filebeat status:

```bash
journalctl -u filebeat -n 100
```

Check Filebeat logs:

```bash
filebeat test config
```

Test configuration:

```bash
filebeat test output
```

Test output:

```yaml
scrape_configs:
  - job_name: nginx-access
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx-access
          env: prod
          service: nginx
          __path__: /var/log/nginx/*.access.json.log
```

---

## Fifteen. Promtail / Loki JSON Log Collection Example

---

## Scenario 41: Promtail Basic Collection Configuration

Example:

```text
__path__
→ Collection Path

job/env/service
→ Loki labels
```

Description:

```yaml
scrape_configs:
  - job_name: nginx-access
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx-access
          env: prod
          service: nginx
          __path__: /var/log/nginx/*.access.json.log

    pipeline_stages:
      - json:
          expressions:
            status: status
            host: host
            uri: uri
            method: method
            request_time: request_time
            remote_addr: remote_addr
      - labels:
          status:
          host:
          method:
```

---

## Scenario 42: Promtail JSON Parsing Example

Example:

```text
Don't. uriI don't know.remote_addrI don't know.request_id These high-base segments are easy to do. label

Example above uri and remote_addr Parsing only, not recommended for insertion labels
```

Note:

```yaml
      - labels:
          status:
          method:
```

More cautious labels:

```text
2xx

3xx

4xx

5xx
```

In production, you can also categorize status codes into:

```logql
{job="nginx-access", env="prod"}
```

Then use as labels to reduce cardinality.

---

## Sixteen. Loki Query Examples

---

## Scenario 43: Query Logs for a Specific Service

```logql
{job="nginx-access", env="prod"} | json | status >= 500
```

---

## Scenario 44: Filter 5xx Errors

If JSON is already parsed:

```logql
{job="nginx-access", env="prod"} | json | request_time > 1
```

---

## Scenario 45: Filter Slow Requests

```logql
{job="nginx-access", env="prod"} |= "upstream timed out"
```

---

## Scenario 46: Filter by Keywords

```logql
{job="nginx-access", env="prod"} | json | request_id="abc123"
```

---

## Scenario 47: Query by request_id

```text
request_id Not recommended for action label

But you can search in log contents
```

Note:

```text
User-Agent with double quotes

Referer with special characters

Request URI with special characters

Result JSON Invalid
```

---

## Seventeen. Common JSON Log Errors

---

## Scenario 48: Forgetting to escape=json

Risk:

```nginx
log_format access_json escape=json
```

Recommendation:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"status":$status,'
    '}';
```

---

## Scenario 49: Extra Comma After Last Field

Error example:

```text
JSON No comma after the last field
```

Problem:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"status":$status'
    '}';
```

Correct:

```nginx
"upstream_response_time":$upstream_response_time
```

---

## Scenario 50: Treating Nullable Fields as Numbers

Error example:

```text
-
```

If value is:

```json
"upstream_response_time":-
```

Log becomes:

```nginx
"upstream_response_time":"$upstream_response_time"
```

This is invalid JSON.

Recommendation:

```json
{"status":200}
{"status":"500"}
```

---

## Scenario 51: Field Type Conflict

Error example:

```text
Log Platform Field Map Conflict

Convergence Failed

Query anomaly
```

Problem:

```text
status Keep numbers.

request_time If you want to be a value, you need a uniform layer conversion

upstream Multi-value fields save string first
```

Production Recommendation:

```text
/api/login?token=xxx

/api/reset?code=123456

/api/user?idcard=xxx
```

---

## Scenario 52: Recording Complete Query String Leads to Sensitive Information Leakage

`request_uri` may contain:

```text
/api/login?token=xxx

/api/reset?code=123456

/api/user?idcard=xxx
```

Risk:

```text
token Disclosure

Authentication code leak

Private disclosure

Dissemination of sensitive information on log platform
```

Production Recommendations:

```text
Careful record. request_uri

Priority statistics uri

Yeah. query string I'm allergic.

Don't put sensitive information on it. URL Arguments
```

---

## Eighteen. JSON access.log Troubleshooting Process

---

## Scenario 53: JSON Log Not Generated

Troubleshooting:

```bash
nginx -T | grep -n "access_log"
```

```bash
nginx -T | grep -n "log_format" -A 30
```

```bash
ls -lh /var/log/nginx/
```

```bash
tail -n 100 /var/log/nginx/error.log
```

Common Causes:

```text
access_log Path error

Log directory does not exist

Nginx worker User without permission to write

server No hits.

Configure Not reload

access_log By off

log_format Synchronising folder
```

---

## Scenario 54: JSON Cannot Be Parsed by jq

Troubleshooting:

```bash
tail -n 1 /var/log/nginx/example.access.json.log | jq .
```

Check log_format:

```bash
nginx -T | grep -n "log_format access_json" -A 40
```

Common Causes:

```text
Nothing. escape=json

Field comma error

String without quotation marks

Number fields may output -

Variable content destruction JSON

Multiline log format abnormal
```

---

## Scenario 55: Log Platform Parsing Failure

Troubleshooting Direction:

```text
Local jq Whether to parse

Whether the acquisition path is correct

Is there a line? JSON

Filebeat / Promtail Whether to press JSON Parsing

Whether field type conflicts

The index template is correct

Whether the log platform has resolved the error field
```

Local Verification:

```bash
tail -n 100 /var/log/nginx/example.access.json.log | jq . >/dev/null
```

Filebeat Logs:

```bash
journalctl -u filebeat -n 100
```

Promtail Logs:

```bash
journalctl -u promtail -n 100
```

---

## Nineteen. Production Notes

---

## 1. JSON Logs Must Ensure One Line Per JSON

Do not let a single access_log line become multiple lines.

Log platforms typically collect logs line by line.

---

## 2. JSON String Fields Must Be Quoted

Example:

```nginx
"host":"$host"
```

Do not write as:

```nginx
"host":$host
```

---

## 3. Fields That May Be Empty or Multi-Value Should Be Strings

Including:

```text
upstream_addr

upstream_status

upstream_response_time

upstream_connect_time

upstream_header_time

referer

xff
```

---

## 4. Do Not Use High Cardinality Fields as Loki Labels

Not recommended as labels:

```text
remote_addr

request_uri

uri

args

request_id

trace_id

user_agent

xff
```

Suitable for labels:

```text
env

service

job

host

method

status_class
```

---

## 5. request_uri May Contain Sensitive Information

If business URLs contain parameters like tokens, codes, phone numbers, or ID numbers, sensitive information should be masked or avoid recording complete query strings.

---

## 6. JSON Log Fields Should Be Long-Term Stable

Do not frequently change field names.

For example, do not call:

```text
request_time
```

Tomorrow:

```text
rt
```

The day after:

```text
duration
```

Field changes affect:

```text
Log Platform Index

Grafana Panel

The alarm rule.

SRE Disablement habits

Historical queries
```

---

## 7. Governance Is Needed When access.log Volume Is Large

JSON logs are typically longer than plain text, which may increase disk and collection costs.

Need to coordinate with:

```text
logrotate

Log Platform Retention Cycle

Low Value Path Filter

Health check noise reduction

Static resource sampling

Log Monitor

Disk alert.
```

---

## 8. Validate JSON Format in Test Environment First

Before deployment, at least verify:

```bash
nginx -t
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

```bash
tail -n 10 /var/log/nginx/example.access.json.log | jq .
```

---

## Twenty. Summary of Common Commands in This Article

---

## View Configuration

```bash
grep -R "log_format" /etc/nginx/
```

```bash
grep -R "access_log" /etc/nginx/
```

```bash
nginx -T | grep -n "log_format" -A 30
```

```bash
nginx -T | grep -n "access_log"
```

```bash
nginx -T | grep -n "log_format access_json" -A 40
```

---

## Configuration Check and Reload

```bash
nginx -t
```

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

---

## Request Validation

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

---

## View JSON Logs

```bash
tail -n 5 /var/log/nginx/example.access.json.log
```

```bash
tail -f /var/log/nginx/example.access.json.log
```

```bash
tail -n 1 /var/log/nginx/example.access.json.log | jq .
```

```bash
tail -n 100 /var/log/nginx/example.access.json.log | jq . >/dev/null
```

---

## Find Illegal JSON Lines

```bash
while read line; do echo "$line" | jq . >/dev/null || echo "$line"; done < /var/log/nginx/example.access.json.log
```

---

## jq Analyze Status Codes

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .status' | wc -l
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .remote_addr, .method, .request_uri, .status, .upstream_addr, .upstream_status] | @tsv'
```

---

## jq Analyze IP and URL

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.remote_addr' | sort | uniq -c | sort -nr | head
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.uri' | sort | uniq -c | sort -nr | head
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .uri' | sort | uniq -c | sort -nr | head
```

---

## jq Analyze Slow Requests

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .remote_addr, .request_uri, .status, .request_time, .upstream_response_time] | @tsv'
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | .uri' | sort | uniq -c | sort -nr | head
```

---

## jq Analyze Upstream

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_status' | sort | uniq -c | sort -nr
```

---

## Filebeat Check

```bash
systemctl status filebeat
```

```bash
journalctl -u filebeat -n 100
```

```bash
filebeat test config
```

```bash
filebeat test output
```

---

## Promtail Check

```bash
systemctl status promtail
```

```bash
journalctl -u promtail -n 100
```

---

## Twenty-One. One-Sentence Summary

The core value of Nginx JSON access.log is:

```text
Can not open message

To a structured log with clear, searchable, statistically actionable fields
```

Recommended Configuration Core:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request_id":"$request_id",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$uri",'
    '"request_uri":"$request_uri",'
    '"status":$status,'
    '"request_time":"$request_time",'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_response_time":"$upstream_response_time",'
    '"user_agent":"$http_user_agent"'
    '}';
```

Key Fields:

```text
time
→ Request Time

remote_addr
→ Client IP

request_id
→ Request Link ID

method / uri / request_uri
→ Request information

status
→ End state code

request_time
→ The request is time-consuming.

upstream_addr / upstream_status / upstream_response_time
→ Backend Information

user_agent
→ Client Information
```

Production Recommendations:

```text
Must use escape=json

We have to make sure of one line. JSON

String fields must be quoted

upstream Multivalue fields suggest first string

request_uri It may contain sensitive information and require careful recording.

JSON Field name needs long-term stability

Loki label Do Not Use request_idI don't know.remote_addrI don't know.request_uri High-base segment

You have to use it before you go online. jq Authentication JSON Legality

When the log gets bigger, it works. logrotate, log platform retention cycle and cost governance
```