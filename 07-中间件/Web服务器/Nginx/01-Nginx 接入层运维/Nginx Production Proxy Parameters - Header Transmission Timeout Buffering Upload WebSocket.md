# 04-Nginx Production Proxy Parameters: Header Forwarding, Timeout, Buffering, Upload, and WebSocket

#Nginx #ReverseAgent #ProxyArguments #Header'sOnTheAir. #TimeoutConfiguration #BufferConfiguration #UploadingFiles #WebSocket #AccessLayer #Transport #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operations/04-Nginx Production Proxy Parameters: Header Forwarding, Timeout, Buffering, Upload, and WebSocket.md

---

## I. Document Explanation

This document organizes commonly used production proxy parameters when Nginx serves as a reverse proxy.

This article focuses on:

- Why you cannot just use proxy_pass
- Common Header forwarding
- Host forwarding
- X-Real-IP
- X-Forwarded-For
- X-Forwarded-Proto
- X-Forwarded-Host
- Proxy connection timeout
- Proxy read timeout
- Proxy send timeout
- Client request body size limit
- Upload file-related parameters
- proxy_buffering buffering mechanism
- proxy_request_buffering request body buffering
- WebSocket proxy configuration
- SSE/Streaming response configuration
- Production complete proxy configuration example
- Common 413/499/502/504 troubleshooting approaches
- Production environment considerations

This article is the 04th in the Nginx Access Layer Operations series.

This article's goal:

```text
I can write a copy of what's available. Nginx Reverse Proxy Configuration

→ It's common. proxy_set_header Role

→ Can adjust the timeout parameters to the business scene

→ Can process file upload limits

→ I understand. Nginx Impact of buffers on response and upload

→ Configure WebSocket Proxy

→ I can check. 413I don't know.499I don't know.502I don't know.504 Wait for the usual questions.
```

---

## II. Why You Can't Just Use proxy_pass

The simplest reverse proxy configuration is:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

This configuration works, but it's usually insufficient in production.

Reasons:

```text
The back end may not be real. Host

The backend may not get the client real. IP

Backend doesn't know the original protocol is HTTP Still? HTTPS

Long requests may be made Nginx Early timeout

Big file upload may return 413

WebSocket Could not initialise Bonobo

Fluent response could be buffered.

Backend connections may be created and closed frequently

The necessary context was missing for the error check
```

In production, more common proxy configurations include:

```text
Header Passage

Timeout Control

Buffer control

Upload size limit

WebSocket Support

Access logs and errors

Retry policy if necessary

Necessary security restrictions
```

---

## III. Overview of Production Proxy Parameters

Common production proxy parameters can be categorized into several types:

```text
Header Category
→ proxy_set_header

Overtime class
→ proxy_connect_timeout
→ proxy_send_timeout
→ proxy_read_timeout
→ send_timeout
→ client_body_timeout
→ client_header_timeout

Buffer class
→ proxy_buffering
→ proxy_buffers
→ proxy_buffer_size
→ proxy_busy_buffers_size
→ proxy_request_buffering

Upload class
→ client_max_body_size
→ client_body_buffer_size
→ client_body_temp_path

Type of agreement
→ proxy_http_version
→ Upgrade / Connection
→ WebSocket Support

Redirect Category
→ proxy_redirect

Error Processing Class
→ proxy_intercept_errors
```

---

## IV. Basic Production Proxy Configuration Example

A relatively common production proxy base template:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;

    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;

    proxy_buffering on;
    proxy_buffer_size 16k;
    proxy_buffers 8 16k;
    proxy_busy_buffers_size 32k;
}
```

Explanation:

```text
proxy_pass
→ Forward to Backend upstream

proxy_http_version 1.1
→ Use HTTP/1.1 Connect Backend

proxy_set_header
→ Transmit the necessary requests.

proxy_connect_timeout
→ Nginx Connect backend timeout

proxy_send_timeout
→ Nginx Send request timeout to backend

proxy_read_timeout
→ Nginx Waiting for backend response timeout

proxy_buffering
→ Whether to open a response buffer
```

---

## V. Basic Header Forwarding

---

## Scenario 1: Why Header Forwarding is Necessary

Backend applications typically need to know:

```text
What is the original user access domain name?

User Real IP What is it?

Which agents did the request go through?

The original request protocol is HTTP Still? HTTPS

Original Host What is it?

From HTTPS Entry

Request for load balance or gateway in the link
```

If these headers are not forwarded, it may lead to:

```text
Backend Record IP Both. Nginx IP

Application to generate jump-link error

HTTPS Page recognized by backend HTTP

Login Call Address Error

Audit log IP Inaccuracy

Limit flow button IP Inaccuracy

Security strategy error.
```

---

## Scenario 2: Host Forwarding

Configuration:

```nginx
proxy_set_header Host $host;
```

Purpose:

```text
Synchronising folder Host To Backend
```

For example, user access:

```text
https://www.example.com
```

The backend can continue to see:

```text
Host: www.example.com
```

Suitable scenarios:

```text
Backend By Host Distinction of tenants

Backend By Host Generate Callback Address

Backend needs sense original access domain names

Multi-domain name sharing backend
```

If not set, the backend may see:

```text
Host: app_backend

or

Host: 127.0.0.1:8080
```

This may cause business logic anomalies.

---

## Scenario 3: X-Real-IP

Configuration:

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

Purpose:

```text
Connect directly. Nginx Client IP To Backend
```

Note:

```text
$remote_addr Yes. Nginx Directly seen source IP

If Nginx There's more ahead. SLB / CDN / WAF

$remote_addr Could be a top agent. IP, not the real user IP
```

Real IP issues will be addressed separately in the 07th article on multi-level proxy real IP.

---

## Scenario 4: X-Forwarded-For

Configuration:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Purpose:

```text
Recording of surrogate links to requests IP
```

Explanation:

```text
$proxy_add_x_forwarded_for
→ In original X-Forwarded-For Add after $remote_addr
```

For example:

```text
User Real IP:1.1.1.1

SLB IP:2.2.2.2

Nginx Yeah. remote_addr:2.2.2.2
```

The backend may receive:

```text
X-Forwarded-For: 1.1.1.1, 2.2.2.2
```

Production notes:

```text
X-Forwarded-For It can be forged by clients.

Backend cannot unconditionally trust the leftmost IP

It has to be linked to a credible proxy link.
```

---

## Scenario 5: X-Forwarded-Proto

Configuration:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

Purpose:

```text
Tell backend original request protocol is http Still? https
```

If users access:

```text
https://example.com
```

The backend receives:

```text
X-Forwarded-Proto: https
```

Suitable scenarios:

```text
Backend Generation HTTPS Link

Login Callback Address Generation

OAuth Rewind

Back and forth jump

Force HTTPS Decision
```

If this field is missing, the backend may assume the request is HTTP, leading to incorrect redirect URLs.

---

## Scenario 6: X-Forwarded-Host

Configuration:

```nginx
proxy_set_header X-Forwarded-Host $host;
```

Purpose:

```text
Tell Backend Original Host
```

Some frameworks use:

```text
X-Forwarded-Host

X-Forwarded-Proto

X-Forwarded-Port
```

To generate external access URLs.

---

## Scenario 7: X-Forwarded-Port

HTTP:

```nginx
proxy_set_header X-Forwarded-Port 80;
```

HTTPS:

```nginx
proxy_set_header X-Forwarded-Port 443;
```

You can also use variables:

```nginx
proxy_set_header X-Forwarded-Port $server_port;
```

Purpose:

```text
Tell backend original entry port
```

---

## VI. Common Header Production Configuration

---

## Scenario 8: HTTP Reverse Proxy Header Example

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
}
```

---

## Scenario 9: HTTPS Entry to Backend HTTP

Common architecture:

```text
Client HTTPS

→ Nginx 443

→ Backend HTTP 8080
```

Configuration example:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    location / {
        proxy_pass http://app_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 443;
    }
}
```

Explanation:

```text
Although... Nginx Backend is HTTP

But the original user access is HTTPS

So... X-Forwarded-Proto Should tell backend https
```

---

## VII. Timeout Parameter Basics

---

## Scenario 10: Why Timeout Configuration is Needed

If timeouts are not configured properly, it may lead to:

```text
Backend Slow Requested Nginx Disconnect early

The client's waiting long.

Nginx worker The connection was occupied for a long time

Failed to upload large file

Download big file aborted

Interface occasional 504

Accumulation of interfaces in back end failure
```

Timeout configurations should be set according to business scenarios:

```text
Normal API

Uploading files

File Download

Slow query interface

Report Export

WebSocket

SSE Fluent Response
```

Different scenarios should not all use the same timeout settings.

---

## Scenario 11: proxy_connect_timeout

Configuration:

```nginx
proxy_connect_timeout 5s;
```

Purpose:

```text
Nginx Timeout to connect backend services
```

Common causes:

```text
Backend not working

Back-end machine network abnormal.

Firewall blocked.

Backend service not monitored

SYN No response
```

If the connection to the backend times out, it may result in:

```text
504 Gateway Time-out

or error.log Medium upstream timed out while connecting to upstream
```

Troubleshooting:

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## Scenario 12: proxy_send_timeout

Configuration:

```nginx
proxy_send_timeout 60s;
```

Purpose:

```text
Nginx Timeout to send requested data backend
```

Common scenarios:

```text
Upload Request to Backend

Large POST Request

Backend reading requests are slow

Network congestion
```

Note:

```text
This is the time between the two writing operations.

It's not the entire time to send a request.
```

---

## Scenario 13: proxy_read_timeout

Configuration:

```nginx
proxy_read_timeout 60s;
```

Purpose:

```text
Nginx Timeout waiting for backend response data
```

Common scenarios:

```text
Backend implementation slow

Database slow query

Report Export

External interface slow call

Back-end loop pool blocked
```

If this time is exceeded, common results:

```text
504 Gateway Time-out
```

error.log may show:

```text
upstream timed out while reading response header from upstream
```

Troubleshooting:

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://BackendIP:Backend/Slow Interface
```

---

## Scenario 14: send_timeout

Configuration:

```nginx
send_timeout 60s;
```

Purpose:

```text
Nginx Timeout to send response to client
```

Common scenarios:

```text
Client network slow

Client Download Big Files

Client Disconnect

Weak net environment
```

Note:

```text
send_timeout Yes. Nginx To client direction

proxy_read_timeout Backend to Nginx Direction
```

---

## Scenario 15: client_body_timeout

Configuration:

```nginx
client_body_timeout 60s;
```

Purpose:

```text
Client Send Request to Nginx Timeout
```

Suitable for focusing on:

```text
Uploading files

Large Request

A weak web client

Slow attack defense.
```

---

## Scenario 16: client_header_timeout

Configuration:

```nginx
client_header_timeout 10s;
```

Purpose:

```text
Client Send Request Header to Nginx Timeout
```

Production use for:

```text
Avoiding unusual client long-term occupancy of connections

Reduce the risk of slow-motion.
```

---

## VIII. Suggested Timeout Configuration for Ordinary APIs

Ordinary API example:

```nginx
location /api/ {
    proxy_pass http://app_backend;

    proxy_connect_timeout 3s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;

    send_timeout 30s;
}
```

Suitable for:

```text
Normal query interface

General operations API

Not involving big file uploads

Export without a long report
```

Note:

```text
The longer the time is not, the better.

It's too long to let the failure link take over resources.

It's too short.

Combining interfaces required SLADecision on back-end processing capacity and business scene
```

---

## IX. Configuration for Slow Interfaces and Report Exports

If some interfaces indeed take longer, it's recommended to configure them separately with a specific location.

Example:

```nginx
location /api/report/export {
    proxy_pass http://app_backend;

    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;

    send_timeout 300s;
}
```

Suitable for:

```text
Report Export

Big Data Query

Organisation

Long Generate Files
```

Production Recommendations:

```text
Don't put the whole station on hold. proxy_read_timeout It's all very big.

Configure only those interfaces that actually require long and time-consuming

Better recommend backend to a different task + Query Task Status + Download Results
```

---

## Ten. Upload File Related Configuration

---

## Scenario 17: client_max_body_size

Configuration:

```nginx
client_max_body_size 50m;
```

Purpose:

```text
Limit client request size
```

If the uploaded file exceeds the limit, it may return:

```text
413 Request Entity Too Large
```

Check error logs:

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 18: Upload Size Configuration by Path

Normal interface limit 10m:

```nginx
location /api/ {
    client_max_body_size 10m;
    proxy_pass http://app_backend;
}
```

Upload interface limit 200m:

```nginx
location /api/upload/ {
    client_max_body_size 200m;
    proxy_pass http://app_backend;
}
```

Production Recommendations:

```text
Don't set it too big. client_max_body_size

Zoom in only for upload interfaces

Avoid abnormal requests for consumption Nginx and backend resources
```

---

## Scenario 19: client_body_buffer_size

Configuration:

```nginx
client_body_buffer_size 128k;
```

Purpose:

```text
Set the size of the buffer to read the client request
```

If the request body exceeds the buffer, Nginx may write to a temporary file.

Related temporary directory:

```text
client_body_temp_path
```

---

## Scenario 20: client_body_temp_path

Configuration:

```nginx
client_body_temp_path /var/cache/nginx/client_temp;
```

Purpose:

```text
Specify client requester temporary file directory
```

Production Note:

```text
Could write temporary files when uploading large files

A temporary directory should have enough space on the disk.

Directory permissions allowed Nginx worker User Writing

Avoid filling the system disk with a temporary directory
```

Check directory:

```bash
ls -ld /var/cache/nginx/client_temp
```

Check disk space:

```bash
df -h
```

---

## Eleven. Request Body Buffering proxy_request_buffering

---

## Scenario 21: What is proxy_request_buffering

Configuration:

```nginx
proxy_request_buffering on;
```

Purpose:

```text
Control Nginx Whether or not to receive the client request fully and forward it to the backend
```

Default is usually:

```text
on
```

---

## Scenario 22: Enable Request Body Buffering

Configuration:

```nginx
proxy_request_buffering on;
```

Behavior:

```text
Client upload the request first Nginx

Nginx Buffer complete request

Forward to Backend
```

Advantages:

```text
Protect backend to avoid slow client drag-backing directly

Backend requests are more stable

It's for general upload and ordinary API
```

Disadvantages:

```text
Big files may be occupied. Nginx Disk Temporary Directory

Before and after uploading, there may be no request. Body

User feels backend processing started late
```

---

## Scenario 23: Disable Request Body Buffering

Configuration:

```nginx
proxy_request_buffering off;
```

Behavior:

```text
Nginx Side-to-side client requester

Forward side to backend
```

Suitable for:

```text
Big File Streaming

Backend needs to be handled side by side.

Reduction Nginx Temporary Fileboard
```

Risks:

```text
Slow client may slow backend connections

Backends are more vulnerable to slow upload.

Backend capability for slow connection processing required
```

Upload interface example:

```nginx
location /api/upload/ {
    client_max_body_size 500m;
    proxy_request_buffering off;

    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;

    proxy_pass http://app_backend;
}
```

---

## Twelve. Response Buffering proxy_buffering

---

## Scenario 24: What is proxy_buffering

Configuration:

```nginx
proxy_buffering on;
```

Purpose:

```text
Control Nginx Whether to buffer backend response
```

After enabling:

```text
Backend response first. Nginx Buffer zone

Nginx Send to client
```

After disabling:

```text
Nginx Send backend responses to clients in real time as much as possible. End
```

---

## Scenario 25: Enable proxy_buffering

Configuration:

```nginx
proxy_buffering on;
```

Suitable for:

```text
Normal Web Page

Normal API Response

Backend response faster

Client network may be slow

Want to reduce the back end to slow client Drag Stay.
```

Advantages:

```text
Backends release connections faster.

Nginx Send it slowly to client

The whole thing is more stable.
```

---

## Scenario 26: Disable proxy_buffering

Configuration:

```nginx
proxy_buffering off;
```

Suitable for:

```text
SSE Fluent Response

Real-time log output

Large Model Stream Output

Long connection event stream

Backend wants to send data to client in real time
```

Note:

```text
Close buffering Later, slow client may occupy Nginx Longer back end.

Need to be judged in relation to business characteristics
```

---

## Scenario 27: proxy_buffer_size

Configuration:

```nginx
proxy_buffer_size 16k;
```

Purpose:

```text
Set the size of the buffer to read backend response headers
```

If the backend response header is too large, it may cause:

```text
upstream sent too big header
```

Troubleshooting:

```bash
grep -i "upstream sent too big header" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 28: proxy_buffers

Configuration:

```nginx
proxy_buffers 8 16k;
```

Purpose:

```text
Set the number and size of buffers to read backend responses
```

Meaning:

```text
8 buffer zone

Each 16k
```

---

## Scenario 29: proxy_busy_buffers_size

Configuration:

```nginx
proxy_busy_buffers_size 32k;
```

Purpose:

```text
Limit available when sending response to client busy buffer Size
```

This parameter typically needs to be understood and adjusted together with:

```text
proxy_buffer_size

proxy_buffers
```

---

## Thirteen. Streaming Response Configuration

---

## Scenario 30: SSE / Streaming Output Configuration

Suitable for:

```text
Server-Sent Events

Real Time Log

Large model flow back

Keep sending messages
```

Configuration example:

```nginx
location /api/stream/ {
    proxy_pass http://app_backend;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_buffering off;
    proxy_cache off;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Explanation:

```text
proxy_buffering off
→ Avoid Nginx Buffer-backstream response

proxy_read_timeout 3600s
→ Allows long periods of response to read

proxy_cache off
→ Avoid Cache Interference
```

Production Note:

```text
Long connections take over the connection resources

Needs assessment worker_connections

Number of connections to monitor

Can't open to all users without restriction
```

---

## Fourteen. WebSocket Proxy Configuration

---

## Scenario 31: Why WebSocket Needs Special Configuration

WebSocket requires upgrading the protocol from HTTP.

The client request headers will include:

```text
Upgrade: websocket

Connection: Upgrade
```

Nginx needs to pass these headers to the backend.

---

## Scenario 32: WebSocket Basic Configuration

```nginx
location /ws/ {
    proxy_pass http://app_backend;

    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Explanation:

```text
proxy_http_version 1.1
→ WebSocket Yes. HTTP/1.1

Upgrade
→ Transcript protocol upgrade

Connection
→ Tell the backend to upgrade the protocol.

proxy_read_timeout
→ Long connection takes longer read timeout
```

---

## Scenario 33: Use map to Optimize Connection

It's recommended to configure in the http block:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

Then use in location:

```nginx
location /ws/ {
    proxy_pass http://app_backend;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Benefits:

```text
Yes. Upgrade Time Connection Yes upgrade

Nothing. Upgrade Time Connection Yes close

Configure More Normal
```

---

## Scenario 34: Common WebSocket Issues

Common phenomena:

```text
WebSocket Connection failed

101 Switching Protocols No return

The connection will be disconnected soon.

Frontend WebSocket closed

Nginx error.log Come on. upstream timed out
```

Troubleshooting:

```bash
grep -i "upgrade" /var/log/nginx/access.log | tail -n 50
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

Check configuration:

```bash
nginx -T | grep -n "Upgrade" -A 5 -B 5
```

---

## Fifteen. proxy_redirect Basics

---

## Scenario 35: What is proxy_redirect

When the backend returns a redirect response, the response headers may include:

```text
Location: http://127.0.0.1:8080/login
```

If the client sees this address, it may not be accessible.

`proxy_redirect` can rewrite the backend's Location.

---

## Scenario 36: Disable proxy_redirect

Common configuration:

```nginx
proxy_redirect off;
```

Suitable for:

```text
Backend has correctly generated external URL

I don't want to. Nginx Autorew Location

Apply through X-Forwarded-Proto / Host Other Organiser
```

---

## Scenario 37: Manually Rewrite Location

Example:

```nginx
proxy_redirect http://127.0.0.1:8080/ https://example.com/;
```

Meaning:

```text
Put Backend Location Medium http://127.0.0.1:8080/

Replace with

https://example.com/
```

Production Recommendations:

```text
Prioritize backend correct identification Host and X-Forwarded-Proto

Don't overdependence. proxy_redirect Repair operations URL
```

---

## Sixteen. Complete Production Proxy Configuration Example

---

## Scenario 38: Ordinary API Production Proxy Example

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;

    keepalive 64;
}

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    client_max_body_size 20m;
    client_header_timeout 10s;
    client_body_timeout 60s;

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers 8 16k;
        proxy_busy_buffers_size 32k;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 5s;
    }
}
```

---

## Scenario 39: Upload Interface Separate Configuration Example

```nginx
location /api/upload/ {
    proxy_pass http://app_backend;

    client_max_body_size 500m;
    client_body_timeout 300s;

    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;

    proxy_request_buffering off;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

---

## Scenario 40: WebSocket Separate Configuration Example

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream ws_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://ws_backend;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

---

## Seventeen. Common Status Codes and Proxy Parameters Relationship

---

## Scenario 41: 413 Request Entity Too Large

Common causes:

```text
client_max_body_size Too small.

Upload file exceeding limit

The request is too big.

Nginx Or the upper proxy limits are smaller.
```

Troubleshooting:

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

Check configuration:

```bash
nginx -T | grep -n "client_max_body_size"
```

Handling:

```nginx
client_max_body_size 100m;
```

Production Recommendations:

```text
Zoom in only for upload interfaces

Don't go all the way up.
```

---

## Scenario 42: 499 Client Closed Request

499 is a Nginx-specific status code, indicating:

```text
Client actively disconnects
```

Common causes:

```text
User Cancel Request

Browser Timeout

Client network poor

Backend processing too slow.

Nginx Not back yet. Client has been disconnected.

Upper Agent or Load Balance Overtime
```

Troubleshooting:

```bash
awk '$9 == 499 {print $0}' /var/log/nginx/access.log | tail -n 50
```

Statistical 499 URLs:

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Handling direction:

```text
Check backend time-consuming

Check client timeout

Inspection SLB / CDN Timeout

Optimizing slow interfaces

Adjust timeout if necessary

Long mission change.
```

---

## Scenario 43: 502 Bad Gateway

Common causes:

```text
Backend not working

Backend Process Collapse

Backend Active Close Connection

Nginx Failed to connect to backend

Backend returns illegal response

upstream No nodes available
```

Troubleshooting:

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## Scenario 44: 504 Gateway Time-out

Common Causes:

```text
proxy_connect_timeout Timeout

proxy_read_timeout Timeout

Backend processing slow

Database slow query

Back-end pool full

Reliance on services slow

The network links are abnormal.
```

Troubleshooting:

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
curl -v http://BackendIP:Backend/health
```

Resolution Direction:

```text
Where's the back end?

Optimizing backend interfaces

Check databases and dependencies

Zoom in only for specified slow interfaces if necessary proxy_read_timeout

Don't increase the timeout blindly.
```

---

## Eighteen. Proxy Parameter Verification Process

After production configuration is completed, it is recommended to verify in the following order:

```text
Inspection Nginx Configure Syntax:

→ Check if complete configuration is effective

→ Direct Access Backend

→ Pass. Nginx Visits

→ View access.log

→ View error.log

→ Authentication Header Whether it's passed through or not

→ Verify Upload Size

→ Verify Timeout Behaviour

→ Authentication WebSocket or flow interface
```

---

## Scenario 45: Check Configuration Syntax

```bash
nginx -t
```

---

## Scenario 46: View Full Configuration

```bash
nginx -T | grep -n "proxy_set_header" -A 10 -B 5
```

```bash
nginx -T | grep -n "proxy_read_timeout"
```

```bash
nginx -T | grep -n "client_max_body_size"
```

---

## Scenario 47: Test Backend

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

---

## Scenario 48: Test Nginx Entry

```bash
curl -v -H "Host: example.com" http://127.0.0.1/health
```

---

## Scenario 49: Check Logs

```bash
tail -f /var/log/nginx/example.access.log
```

```bash
tail -f /var/log/nginx/example.error.log
```

---

## Scenario 50: Verify Header Passthrough

The backend can temporarily provide a debugging interface that outputs request headers.

Through Nginx request:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/debug/headers
```

Pay attention to whether the backend received:

```text
Host

X-Real-IP

X-Forwarded-For

X-Forwarded-Proto

X-Forwarded-Host
```

---

## Nineteen. Production Notes

---

## 1. Header Passthrough is the Foundation of Production Proxy

It is recommended to at least include:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

---

## 2. Do Not Arbitrarily Increase Timeout

Wrong Practice:

```nginx
proxy_read_timeout 3600s;
```

Full Site Timeout Risk:

```text
Long-term occupancy of resources by faulty connection

Slow request to pull down the entrance level.

The problem is covered up.

Client experience doesn't necessarily change.
```

Recommendation:

```text
Shorter timeout for normal interfaces

Slow interface alone location Zoom In

Long-term tasks are prioritized in different steps
```

---

## 3. Upload Size Limits Should Be Configured by Path

Recommended:

```text
Normal API Small limit

Uploading interfaces to big limits

Control backstage carefully zoom in.

Don't just stand there.
```

---

## 4. Do Not Arbitrarily Disable proxy_buffering

Suitable to Disable:

```text
SSE

WebSocket No ordinary response buffer

Stream Output

Real Time Log
```

Ordinary API is usually more suitable to keep enabled.

---

## 5. WebSocket Must Configure Upgrade

Core Configuration:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

---

## 6. Pay Attention to Temporary Directory Disk Space for Large File Uploads

Need to Check:

```bash
df -h
```

```bash
du -sh /var/cache/nginx
```

Avoid:

```text
Nginx client_body_temp_path Full system disk.

Cause upload failure or influence other services
```

---

## 7. Do Not Only Modify Nginx for Timeout Issues

504 may just be the manifestation.

Continue to troubleshoot:

```text
Backend interface time-consuming

Database slow query

Cache Delay

External dependency timeout

Backend Pond

Backend Connection Pool

Host CPU / Memory / IO
```

---

## Twenty. Summary of Common Commands in This Article

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "proxy_set_header" -A 10 -B 5
```

```bash
nginx -T | grep -n "proxy_read_timeout"
```

```bash
nginx -T | grep -n "client_max_body_size"
```

```bash
nginx -T | grep -n "proxy_buffering"
```

```bash
nginx -T | grep -n "Upgrade" -A 5 -B 5
```

---

## Service Reload

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

---

## Backend Check

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

```bash
curl -I http://BackendIP:Backend
```

---

## Nginx Entry Verification

```bash
curl -v -H "Host: example.com" http://127.0.0.1/health
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Log Viewing

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
tail -f /var/log/nginx/example.access.log
```

```bash
tail -f /var/log/nginx/example.error.log
```

---

## 413 Troubleshooting

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

```bash
nginx -T | grep -n "client_max_body_size"
```

---

## 499 Troubleshooting

```bash
awk '$9 == 499 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## 502 Troubleshooting

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## 504 Troubleshooting

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## Upload Temporary Directory Check

```bash
df -h
```

```bash
du -sh /var/cache/nginx
```

```bash
ls -ld /var/cache/nginx/client_temp
```

---

## Twenty-one. One-sentence Summary

The core of Nginx production proxy parameters is:

```text
Header It's all over.

Timeout should be reasonable.

Upload restricted

Buffers should be on scene.

WebSocket To support protocol upgrades

Fluid response to close response buffer

The faults must be combined. access.log and error.log Check.
```

Common Headers:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Common Timeouts:

```nginx
proxy_connect_timeout 5s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

Upload-related:

```nginx
client_max_body_size 100m;
proxy_request_buffering off;
```

WebSocket Core Configuration:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

Common Issue Correspondence:

```text
413
→ client_max_body_size Too small.

499
→ Clients cut off early, often related to slow requests or client overtime

502
→ Nginx Failed to connect to backend or backend anomaly

504
→ Backend response timeout or connection timeout

upstream sent too big header
→ proxy_buffer_size Maybe too small.
```

Production Recommendations:

```text
Don't just write. proxy_pass

Don't be brainless.

Do not zoom in on the upload limit.

Don't just close it. proxy_buffering

WebSocket The flow interfaces are separate. location Configure

Upload interface to watch Nginx Temporary directory disk

504 Don't just change. NginxWe need to keep checking where the back end is.

Backup before changing the configuration.reload I have to. nginx -t
```