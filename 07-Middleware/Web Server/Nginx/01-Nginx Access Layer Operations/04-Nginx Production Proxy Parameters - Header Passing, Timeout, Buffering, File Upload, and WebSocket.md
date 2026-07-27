# 04-Nginx Production Proxy Parameters: Header Passing, Timeout, Buffering, File Upload, and WebSocket

#Nginx #Reverse Proxy #Proxy Parameters #Header Passing #Timeout Configuration #Buffering Configuration #File Upload #WebSocket #Access Layer #Ops #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Ops/04-Nginx Production Proxy Parameters: Header Passing, Timeout, Buffering, File Upload, and WebSocket.md

---

## I. Document Overview

This article compiles commonly used production proxy parameters for Nginx in a reverse proxy role.

Key points include:

- Why you can't just use `proxy_pass`
- Common header passing practices
- Host passing
- `X-Real-IP`
- `X-Forwarded-For`
- `X-Forwarded-Proto`
- `X-Forwarded-Host`
- Proxy connection timeout
- Proxy read timeout
- Proxy send timeout
- Client request body size limits
- File upload parameters
- `proxy_buffering` buffering mechanism
- `proxy_request_buffering` request body buffering
- WebSocket proxy configuration
- SSE / Streaming response configuration
- Example of a complete production proxy configuration
- Common troubleshooting for 413, 499, 502, 504 errors
- Production environment considerations

This article is part of the Nginx Access Layer Ops series, Chapter 04.

Objectives:

```text
To be able to configure a production-ready Nginx reverse proxy

→ To understand the purpose of common `proxy_set_header` directives

→ To adjust timeout parameters based on business requirements

→ To handle file upload size limits

→ To comprehend how Nginx buffering affects responses and uploads

→ To configure WebSocket proxies

→ To troubleshoot common issues such as 413, 499, 502, 504
```

---

## II. Why You Can't Just Use `proxy_pass`

The simplest reverse proxy configuration is:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

While this works, it's often insufficient in a production environment.

Reasons include:

```text
- The backend may not receive the actual Host.
- The backend may not get the client's real IP address.
- The backend cannot determine whether the protocol is HTTP or HTTPS.
- Long requests might time out before reaching the backend.
- Uploading large files could result in a 413 error.
- WebSocket connections might fail to upgrade.
- Streaming responses could be affected by buffering.
- The backend may experience frequent connection creations and closures.
- Lack of necessary context makes troubleshooting more difficult.
```

In production, a more comprehensive proxy configuration typically includes:

```text
Header passing

Timeout controls

Buffering settings

File upload limits

WebSocket support

Access logs and error logs

Appropriate retry strategies

Security measures
```

---

## III. Overview of Production Proxy Parameters

Common production proxy parameters can be categorized into several types:

```text
Header Category
→ `proxy_set_header`

Timeout Category
→ `proxy_connect_timeout`
→ `proxy_send_timeout`
→ `proxy_read_timeout`
→ `send_timeout`
→ `client_body_timeout`
→ `client_header_timeout`

Buffering Category
→ `proxy_buffering`
→ `proxy_buffers`
→ `proxy_buffer_size`
→ `proxybusyBuffers_size`
→ `proxy_request_buffering`

Upload Category
→ `client_max_body_size`
→ `client_body_buffer_size`
→ `client_body_temp_path`

Protocol Category
→ `proxy_http_version`
→ `Upgrade / Connection`
→ WebSocket support

Redirection Category
→ `proxy_redirect`

Error Handling Category
→ `proxy_intercept_errors`
```

---

## IV. Example of a Basic Production Proxy Configuration

A common basic production proxy configuration template is:

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
    proxybusyBuffers_size 32k;
}
```

Explanation:

```text
proxy_pass
→ Redirects requests to the backend upstream server.

proxy_http_version 1.1
→ Establishes a HTTP/1.1 connection with the backend.

proxy_set_header
→ Passes necessary request headers```nginx
proxy_set_header X-Forwarded-Port 80;
```

For HTTPS:

```nginx
proxy_set_header X-Forwarded-Port 443;
```

It is also possible to use a variable:

```nginx
proxy_set_header X-Forwarded-Port $server_port;
```

Function:

```text
Informs the backend about the original incoming port number.
```

---

## VI. Configuration of Common Headers

---

## Scenario 8: Example of HTTP Reverse Proxy Headers

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

## Scenario 9: Converting HTTPS Inbound to Backend HTTP

Common architecture:

```text
Client uses HTTPS

→ Nginx listens on port 443

→ Backend runs on port 8080
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
Although Nginx communicates with the backend over HTTP,

the client initially used HTTPS.

Therefore, the X-Forwarded-Proto header must indicate "https" to the backend.
```

---

## VII. Basics of Timeout Parameters

---

## Scenario 10: Why Configure Timouts

If timeouts are not set properly, the following issues may occur:

```text
Slow backend requests may be prematurely disconnected by Nginx.

Clients may experience long waits.

Nginx worker processes might remain occupied for extended periods.

Large file uploads could fail.

Downloading large files might be interrupted.

Occasional 504 errors might appear.

In the event of a backend failure, connections at the entry layer could accumulate.
```

Timeout settings should be adjusted according to specific business scenarios:

```text
For regular APIs

File uploads

File downloads

Slow query interfaces

Report exports

WebSocket

SSE streaming responses
```

Different scenarios require different timeout values.

---

## Scenario 11: proxy_connect_timeout

Configuration:

```nginx
proxy_connect_timeout 5s;
```

Function:

```text
This setting defines the timeout period for Nginx to establish a connection with the backend service.
```

Common reasons for timeouts:

```text
The backend port is unavailable.

There are network issues with the backend server.

Firewalls may be blocking the connection.

The backend service is not listening.

No response is received after sending a SYN packet.
```

If a timeout occurs while connecting to the backend, errors such as "504 Gateway Time-out" may be displayed, or logs might contain messages like "upstream timed out while connecting to upstream".

Troubleshooting steps:

```bash
nc -zv -w 2 backend_IP backend_port
```

```bash
curl -v http://backend_ip:backend_port/health
```

---

## Scenario 12: proxy_send_timeout

Configuration:

```nginx
proxy_send_timeout 60s;
```

Function:

```text
This setting defines the timeout period for Nginx to send request data to the backend.
``

Common use cases:

```text
Uploading request bodies to the backend

Handling large POST requests

When the backend takes a long time to process requests

In cases of network congestion
```

Note:

```text
This value represents the timeout between two writing operations, not the total duration of the entire request.
```

---

## Scenario 13: proxy_read_timeout

Configuration:

```nginx
proxy_read_timeout 60s;
```

Function:

```text
This setting defines the timeout period for Nginx to wait for responses from the backend.
```

Common scenarios:

```text
Slow execution of backend interfaces

Long-running database queries

Report exports

Slow external API calls

Backend thread pools being blocked
```

If this timeout is exceeded, errors like "504 Gateway Time-out" may be displayed. Logs might also contain messages such as "upstream timed out while reading response header from upstream".

TrouProduction Recommendations:

```text
Do not set the client_max_body_size too high for the entire site.

Only increase the limit for upload interfaces.

This will prevent exceptionally large requests from consuming Nginx and backend resources.
```

---

## Scenario 19: client_body_buffer_size

Configuration:

```nginx
client_body_buffer_size 128k;
```

Function:

```text
Sets the size of the buffer used to read the client request body.
```

If the request body exceeds the buffer size, Nginx may write it to a temporary file.

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

Function:

```text
Specifies the directory where temporary files for client request bodies are stored.
```

Production Notes:

```text
Temporary files may be created when uploading large files.

The disk where the temporary directory is located must have sufficient space.

Directory permissions must allow Nginx worker users to write to it.

Ensure the temporary directory does not fill up the system disk space.
```

To view the directory:

```bash
ls -ld /var/cache/nginx/client_temp
```

To check disk space:

```bash
df -h
```

---

## XI. proxy_request_buffering: Request Body Buffering

---

## Scenario 21: What is proxy_request_buffering?

Configuration:

```nginx
proxy_request_buffering on;
```

Function:

```text
Controls whether Nginx receives the entire client request body before forwarding it to the backend.
```

Default value is usually:

```text
on
```

---

## Scenario 22: Enabling Request Body Buffering

Configuration:

```nginx
proxy_request_buffering on;
```

Behavior:

```text
The client first uploads the request body to Nginx.

Nginx buffers the entire request body before forwarding it to the backend.
```

Advantages:

```text
It protects the backend from being delayed by slow clients.

The backend receives requests more stably.

Suitable for regular file uploads and common APIs.
```

Disadvantages:

```text
Large files may fill up Nginx's temporary disk directory.

The backend may not receive the request body until the upload is complete.

Users might perceive a delay in the backend processing.
```

---

## Scenario 23: Disabling Request Body Buffering

Configuration:

```nginx
proxy_request_buffering off;
```

Behavior:

```text
Nginx receives the client request body while simultaneously forwarding it to the backend.
```

Suitable for:

```text
Streaming uploads of large files.

Backends that need to process data as it arrives.

This reduces the number of temporary files Nginx writes to disk.
```

Risks:

```text
Slow clients may cause delays in the connection with the backend.

The backend is more vulnerable to slow uploads.

The backend must be capable of handling such connections.
```

Example configuration for an upload interface:

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

## XII. Response Buffering: proxy_buffering

---

## Scenario 24: What is proxy_buffering?

Configuration:

```nginx
proxy_buffering on;
```

Function:

```text
Controls whether Nginx buffers the backend's response before sending it to the client.
```

When enabled:

```text
The backend's response first enters Nginx's buffer area.

Nginx then sends it to the client.
```

When disabled:

```text
Nginx tries to send the backend's response to the client in real-time.
```

---

## Scenario 25: Enabling proxy_buffering

Configuration:

```nginx
proxy_buffering on;
```

Suitable for:

```text
Regular web pages.

Common API responses.

When the backend responds quickly but the client's network connection is slow.

This helps reduce delays caused by slow clients.
```

Advantages:

```text：
The backend can release connections more quickly.

Nginx handles the slower transmission to the client.

Overall, throughput becomes more stable.
```

---

## Scenario 26: Disabling proxy_buffering

Configuration:

```nginx
proxy_buffering off;
```

Suitable for:

```text
SSE streaming responses.

Real-time log outputs.

Large model streaming outputs.

Long connection event streams.

When the backend needs to push data to the client in real-time.
```

Note:

```text
Disabling buffering may cause slow clients to consume more time in the Nginx-to-backend connection.

This decision should be based on specific business requirements.
```

---

## Scenario 27proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
}
```

Advantages:

When `Upgrade` is set, `Connection` will be set to `upgrade`.
When `Upgrade` is not set, `Connection` will be set to `close`.
This makes the configuration more standardized.client_max_body_size 100m;```bash
awk '$9 == 499 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Checking for 502 Errors

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
nc -zv -w 2 backend_ip backend_port
```

```bash
curl -v http://backend_ip:backend_port/health
```

---

## Checking for 504 Errors

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
curl -v http://backend_ip:backend_port/health
```

---

## Checking the Temporary Upload Directory

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

## Summary in One Sentence

The key parameters for Nginx as a production proxy include:

- Ensuring headers are correctly forwarded.
- Setting appropriate timeouts to prevent delays.
- Limiting the size of uploaded files.
- Adjusting buffering settings based on specific use cases.
- Supporting protocol upgrades for WebSocket.
- Disabling response buffering for stream-based responses.
- Troubleshooting issues by analyzing both access.log and error.log.

**Common Headers:**
```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

**Common Timouts:**
```nginx
proxy_connect_timeout 5s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

**Upload Settings:**
```nginx
client_max_body_size 100m;
proxy_request_buffering off;
```

**WebSocket Configuration:**
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

**Common Issues and Solutions:**

- **413**: Client-side issue, possibly due to too small a `client_max_body_size`.
- **499**: Client disconnects prematurely, often related to slow requests or client timeouts.
- **502**: Failure in connecting to the backend or backend errors.
- **504**: Backend response timeout or connection timeout.
- **"upstream sent too big header"**: Likely due to an excessively large `proxy_buffer_size`.

**Production Recommendations:**

- Avoid using a generic `proxy_pass` configuration.
- Do not blindly increase global timeouts.
- Adjust upload limits based on specific needs.
- Be cautious when disabling `proxy_buffering`.
- Configure WebSocket and stream-based interfaces in separate locations.
- Pay attention to the disk space used by Nginx's temporary upload directory.
- When encountering a 504 error, investigate whether the issue lies with the backend.