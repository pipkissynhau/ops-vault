# 10-Nginx Performance and Capacity Analysis: Worker Processes, Connection Count, File Descriptors, and QPS Estimation

#Nginx #Performance Analysis #Capacity Assessment #worker_processes #worker_connections #File Descriptors #QPS #Connection Count #Linux #SRE

---

## Recommended Path

07-Middleware/Web Servers/Nginx/02-Nginx Advanced SRE Capacity Expansion/10-Nginx Performance and Capacity Analysis: Worker Processes, Connection Count, File Descriptors, and QPS Estimation.md

---

## I. Document Overview

This article outlines the key concepts, estimation methods, and troubleshooting commands for Nginx performance and capacity analysis.

Key topics include:

- Where Nginx performance bottlenecks lie
- How to understand worker_processes
- How to understand worker_connections
- How to estimate the theoretical maximum number of connections
- Why bidirectional connections are important in reverse proxy scenarios
- How file descriptor limits affect the connection count
- How to configure systemd LimitNOFILE
- The role of worker_rlimit_nofile
- The relationship between QPS, concurrency, and response time
- How to roughly estimate how many requests Nginx can handle
- How to monitor the current number of connections
- How to check the status of connections
- How to analyze TIME_WAIT, ESTAB, and CLOSE_WAIT states
- How to determine if CPU, memory, bandwidth, or disk logging are becoming bottlenecks
- The impact of Nginx access_log on performance
- The effects of HTTPS, gzip, and keepalive on performance
- Basic methods for performance testing
- Considerations for production capacity assessment

This article is the 10th in the Nginx operations series and the first in the advanced SRE capacity expansion series.

Objectives:

```text
To understand that Nginx performance cannot be assessed solely by QPS

→ To know how to estimate worker processes and connection counts

→ To understand why file descriptors can limit Nginx performance

→ To learn how to estimate QPS based on concurrency and response time

→ To use Linux commands to monitor Nginx's current connection status

→ To identify bottlenecks in CPU, network, disk, or backend services

→ To develop a method for evaluating Nginx capacity and troubleshooting performance issues
```

---

## II. Overall Approach to Nginx Performance Analysis

Nginx performance is not determined by a single metric.

It is necessary to consider multiple factors simultaneously:

```text
Connection Count

QPS

Response Time

CPU Usage

Memory Allocation

Network Bandwidth

File Descriptors

Backend Response Speed

TLS Handshake Resources

Log Writing Costs

Disk I/O Operations

Kernel Parameters

Upstream Load Balancer Settings

Client Keepalive Mechanisms

Backend Keepalive Mechanisms
```

Common misconceptions:

```text
Focusing only on QPS and ignoring response time

Only considering worker_connections and neglecting file descriptors

Changing only Nginx settings without addressing backend issues

Adjusting timeouts without checking for connection backlogs

Only examining Nginx configuration without considering system limitations

Only evaluating a single server without considering SLB/CDN/bottom-layer links
```

In summary:

```text
Nginx capacity = Upper limit of Nginx configuration + Upper limit of the Linux system + Backend capabilities + Network bandwidth + Characteristics of business requests
```

---

## III. Common Sources of Nginx Performance Bottlenecks

---

## Scenario 1: Common Bottleneck Sources

Common bottlenecks include:

```text
Too low worker_connections

Insufficient file descriptor limits

Excessive connections from Nginx to the backend

Slow backend responses leading to connection backlogs

Many slow client connections

High number of TIME_WAIT states

Abnormal accumulation of CLOSE_WAIT states

CPU being consumed by TLS/gzip processes

Excessive access_log writing volume

Disk I/O operations being blocked by logs

Reaching the network bandwidth limit

Insufficient capacity of upstream backend nodes

Inadequate kernel connection queues

Insufficient system ephemeral ports

Upstream SLB/CDN timeouts or rate limiting
```

---

## Scenario 2: Nginx Performes Well, but Slow Backends Drag Down Overall Performance

In reverse proxy scenarios:

```text
Client

→ Nginx

→ Backend Service

→ Database/Redis/External API
```

If the backend is slow, Nginx will exhibit the following issues:

```text
Increased request processing time

Rising connection counts

Longer upstream_response_time

Increase in 504 and 499 errors

Worker connections being occupied

 Longer client waiting times
```

Therefore, when performance problems occur, it is essential to look beyond Nginx itself.

It is necessary to examine:

```text
Nginx request_time

Nginx upstream_response_time

Backend interface processing time

Slow database queries

Time spent on Redis/external dependencies

Backend thread```bash
ss -ant state established | grep ':80 '
``````bash
ss -ant state established | grep ':80 ' | wc -l
```

To view the number of ESTABLISHED connections on port 443:

```bash
ss -ant state established | grep ':443 ' | wc -l
```

---

## Scenario 37: Viewing the number of TIME_WAIT sessions

```bash
ss -ant state time-wait | wc -l
```

Count TIME_WAIT sessions by target port:

```bash
ss -ant state time-wait | awk '{print $4}' | awk -F: '{print $NF}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 38: Viewing the number of CLOSE_WAIT sessions

```bash
ss -ant state close-wait | wc -l
```

To view details of CLOSE_WAIT sessions:

```bash
ss -antp state close-wait | head -n 50
```

Note:

```text
CLOSE_WAIT sessions are often related to applications that do not properly close connections.
If a large number of CLOSE_WAIT sessions are detected in the backend processes, it is necessary to investigate the relevant backend applications.
```

---

## Section XIV: Meanings of Connection States

---

## Scenario 39: Common TCP States

```text
ESTAB / ESTABLISHED
→ A connection has been established.

TIME-WAIT
→ The initiating party is waiting for the old connection to be terminated.

CLOSE-WAIT
→ The other party has closed the connection, but the local application has not yet done so.

SYN-SENT
→ A connection attempt is in progress.

SYN-RECV
→ A SYN packet has been received; the handshake process is pending.

LISTEN
→ The server is listening for incoming connections.
```

---

## Scenario 40: Issues Indicated by Different Connection States

```text
A large number of ESTABLISHED sessions:
→ High concurrent connections. This may be normal or could indicate slow backend processing, leading to connection accumulation.

Many TIME_WAIT sessions:
→ Many short-lived connections are being established and terminated frequently.

A significant number of CLOSE_WAIT sessions:
→ The application is not properly closing connections. It is essential to investigate the corresponding processes.

Numerous SYN_RECV sessions:
→ There is pressure on the handshake queue, possibly due to sudden traffic spikes or attacks.

Many SYN_SENT sessions:
→ The other party is slow in responding or is unavailable.
```

---

## Section XV: CPU Analysis

---

## Scenario 41: Viewing Overall CPU Usage

```bash
top
```

Or:

```bash
mpstat 1 5
```

If `mpstat` is not available:

```bash
apt install -y sysstat
```

Or:

```bash
yum install -y sysstat
```

---

## Scenario 42: Viewing Nginx Process CPU Usage

```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | grep nginx | head
```

---

## Scenario 43: Viewing the CPU Usage of Each Worker Process

```bash
top -p $(pgrep -d',' nginx)
```

---

## Scenario 44: Common Causes of High CPU Usage in Nginx

High CPU usage in Nginx can be caused by:

```text
A large number of requests.

Frequent HTTPS TLS handshakes.

Many short-lived connections.

Excessive gzip compression levels.

Complex regular expression `location` or `rewrite` rules.

Very large `access_log` files.

Numerous 4xx/5xx error responses.

Heavy load from WAF/Lua/third-party modules.

A high number of requests for small static files.

Load testing or attack traffic.
```

---

## Section XVI: Memory Analysis

---

## Scenario 45: Viewing Total Memory Usage

```bash
free -h
```

To view the memory usage of the Nginx process:

```bash
ps -eo pid,ppid,cmd,%mem,rss,vsz --sort=-rss | grep nginx | head
```

---

## Scenario 46: Key Points Regarding Nginx Memory Usage

Nginx generally does not consume a large amount of memory. However, the following scenarios can increase memory pressure:

```text
A large number of connections.

Large buffer configurations.

Large request body buffers.

Large response buffers.

Third-party modules.

Lua scripts.

Cache modules.

A large number of worker processes.

Many long-lived connections.
```

---

## Section XVII: Network Bandwidth Analysis

---

## Scenario 47: Viewing Network Card Traffic

```bash
sar -n DEV 1 5
```

If `sar` is not available:

```bash
apt install -y sysstat
```

Or:

```bash
yum install -y sysstat
```

---

## Scenario 48: Checking for Network Card Errors

```bashActive connections: 291
The server is accepting and processing requests:
 16630948 16630948 31070465
Connections in reading mode: 6, writing mode: 179, waiting mode: 106Backend upstream connections  

Capacity estimation should be more conservative than in the case of static resources.