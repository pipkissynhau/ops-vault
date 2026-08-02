# 10-Nginx Performance and Capacity Analysis: worker, Connection Numbers, File Descriptors, and QPS Estimation

#Nginx #PerformanceAnalysis #CapacityAssessment #worker_processes #worker_connections #FileDescription #QPS #NumberOfConnections #Linux #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/10-Nginx Performance and Capacity Analysis: worker, Connection Numbers, File Descriptors, and QPS Estimation.md

---

## I. Document Description

This document organizes core concepts, estimation methods, and troubleshooting commands for Nginx performance and capacity analysis.

This article focuses on:

- Where Nginx performance bottlenecks originate
- Understanding worker_processes
- Understanding worker_connections
- Estimating theoretical maximum connection numbers
- Why bidirectional connections need to be considered in reverse proxy scenarios
- How file descriptor limits affect connection numbers
- How to configure systemd LimitNOFILE
- The role of worker_rlimit_nofile
- The relationship between QPS, concurrency, and response time
- How to roughly estimate how many requests Nginx can handle
- How to observe current connection numbers
- How to observe connection states
- How to analyze TIME_WAIT / ESTAB / CLOSE_WAIT
- How to determine if CPU, memory, bandwidth, or disk logs have become bottlenecks
- The impact of Nginx access_log on performance
- The impact of HTTPS / gzip / keepalive on performance
- Basic methods for performance stress testing
- Notes for production capacity evaluation

This article is the 10th in the Nginx Access Layer Operations series and the first article in the Nginx Advanced SRE Capabilities Expansion series.

This article's objectives:

```text
I understand. Nginx It's not just about performance. QPS

→ I know. worker And how to estimate the number of connections

→ Can you tell why the file descriptioner is restricted? Nginx

→ Can be estimated on the basis of simultaneous dispatch, response time QPS

→ It works. Linux Command observation Nginx Current connection status

→ I can identify it. CPUNetworks, disks, back-end bottlenecks

→ It can form. Nginx Capacity assessment and performance mapping ideas
```

---

## II. Overall Approach to Nginx Performance Analysis

Nginx performance is not a single-point metric.

Need to simultaneously consider:

```text
Number of connections

QPS

Response Time

CPU

Memory

Network bandwidth

File Description

Backend Response Speed

TLS It's a handshake.

Log Write Costs

Disk IO

kernel parameters

Upstream load balance

Client keepalive

Backend keepalive
```

Common misconceptions:

```text
Just looking. QPSNo response time.

Just looking. worker_connections, without file description

Change only NginxIt's slow without looking back.

Just change the time, not see if the connection piles

Just looking. Nginx Configure, without system limitations

Just one machine, no. SLB / CDN / Back-end whole link
```

One-sentence understanding:

```text
Nginx Capacity = Nginx Configure ceiling + Linux System ceiling + Back-end capability + Network bandwidth + Operational request characteristics are jointly determined.
```

---

## III. Where Are Nginx Performance Bottlenecks Usually Found

---

## Scenario 1: Common Bottleneck Sources

Common bottlenecks include:

```text
worker_connections Too small.

File description is too small

Nginx Too many connections to backend

Backend response slows to connect piles

Clients are too slow to connect

TIME_WAIT Too many.

CLOSE_WAIT Abnormal accumulation

CPU By TLS / gzip Consumption

access_log Too much writing.

Disk IO Filled with logs.

Network bandwidth reached maximum

upstream Backend capacity insufficient

Insufficient kernel connection queue

System ephemeral port Insufficient

Upper SLB / CDN Timeout or Flow Limit
```

---

## Scenario 2: Nginx Itself Is Fast, But Slow Backend Slows Down the Whole System

Reverse proxy scenario:

```text
Client

→ Nginx

→ Backend Services

→ Database / Redis / External interface
```

If the backend is slow, Nginx will manifest as:

```text
The request is time-consuming.

Number of connections rising

upstream_response_time Bigger

504 Increase

499 Increase

worker Connection occupied

Longer waiting time for client
```

Therefore, when performance issues occur, you cannot only focus on Nginx.

Need to simultaneously consider:

```text
Nginx request_time

Nginx upstream_response_time

Backend interface time-consuming

Database slow query

Redis / Time-consuming external dependence

Back-end and connect pool
```

---

## IV. worker_processes

---

## Scenario 3: What Is worker_processes

Configuration example:

```nginx
worker_processes auto;
```

Function:

```text
Assign Nginx worker Number of processes
```

Common configurations:

```nginx
worker_processes auto;
```

Or:

```nginx
worker_processes 4;
```

Recommendation:

```text
Most production sites are used auto

auto Usually. CPU Nucleus Auto Settings worker Number
```

---

## Scenario 4: Check CPU Core Count

```bash
nproc
```

Check CPU details:

```bash
lscpu
```

---

## Scenario 5: Check Nginx Worker Count

```bash
ps -ef | grep "nginx: worker" | grep -v grep
```

Count worker numbers:

```bash
ps -ef | grep "nginx: worker" | grep -v grep | wc -l
```

---

## Scenario 6: Check Current Configuration

```bash
nginx -T | grep -n "worker_processes"
```

---

## Scenario 7: Recommendations for Adjusting worker_processes

Usually recommended:

```text
CPU Intensive scenes
→ worker Count close. CPU Core

Normal reverse proxy scene
→ worker_processes auto

Static resources / High and High
→ auto It's usually enough. Focus on the number of connections.FDBandwidth, Disk IO

Container environment
→ Watch the container. CPU limitI don't know.auto Hostage Possible CPU Impact, needs physical validation
```

Not recommended:

```text
Blind. worker_processes Setup Large

worker Far greater than CPU Core

If you don't press, you'll think worker The more, the faster.
```

Reason:

```text
worker Too much could add context to the switch

It doesn't have to go up and down.

It might complicate the search.
```

---

## V. worker_connections

---

## Scenario 8: What Is worker_connections

Configuration example:

```nginx
events {
    worker_connections 1024;
}
```

Function:

```text
Each worker Maximum number of connections allowed to open simultaneously
```

Theoretical maximum connection count is roughly:

```text
worker_processes × worker_connections
```

Example:

```text
worker_processes = 4

worker_connections = 1024

Maximum theoretical connection number = 4 × 1024 = 4096
```

---

## Scenario 9: Check worker_connections Configuration

```bash
nginx -T | grep -n "worker_connections"
```

---

## Scenario 10: Increase worker_connections

Example:

```nginx
events {
    worker_connections 8192;
}
```

Note:

```text
worker_connections Upon improvement, it must also be confirmed that the file description limit is sufficient

Otherwise, the system will remain limited. Hold it!
```

---

## VI. Connection Number Estimation in Reverse Proxy Scenarios

---

## Scenario 11: Why Can't We Only Look at Theoretical Maximum Connection Numbers

If Nginx is only serving static files:

```text
A client connection

About one connection resource
```

But in reverse proxy scenarios:

```text
Client Connection Nginx

+

Nginx Connect Backend upstream
```

Therefore, a single request may occupy:

```text
Client Side Connection

Backend Connection
```

Rough understanding:

```text
In reverse proxy scene, a simultaneous request may consume the contract 2 Connect resources
```

---

## Scenario 12: Theoretical Connection Number Estimation

Assume:

```text
worker_processes = 4

worker_connections = 8192
```

Theoretical maximum connection number:

```text
4 × 8192 = 32768
```

If it's a reverse proxy scenario, the rough estimate for client concurrent connections is:

```text
32768 / 2 ≈ 16384
```

Also need to subtract:

```text
Log File Descriptor

Listen socket

Temporary documents

upstream keepalive Free connection

Other open files

System Retention Resources
```

Therefore, the actual safe value should be more conservative.

Recommend understanding as:

```text
Theoretically, the value is a ceiling, not a production commitment value.
```

---

## Scenario 13: Static Resource Scenarios and Reverse Proxy Scenarios Are Different

Static resource scenario:

```text
Nginx Read local files directly back

The main bottlenecks may be network bandwidth, disks. IOI don't know.sendfileCache.
```

Reverse proxy scenario:

```text
Nginx Waiting for backend response

The main bottlenecks may be back-end time,upstream Connection, timeout, back-end connect pool
```

The same Nginx configuration can have significantly different capacity under different business models.

---

## VII. File Descriptor Limits

---

## Scenario 14: What Are File Descriptors

Many resources in Linux will consume file descriptors:

```text
Client Connection

Backend Connection

Listen socket

Log File

Static File

Temporary documents

Pipes

Unix socket
```

If file descriptors are insufficient, it may result in:

```text
too many open files

accept4() failed

socket() failed

Connection failed

Nginx Could not open log

Nginx Cannot Create upstream Connection
```

---

## Scenario 15: Check Current File Descriptor Limit in Shell

```bash
ulimit -n
```

---

## Scenario 16: Check File Descriptor Limit for Nginx Master Process

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
```

Check file descriptor limits for all Nginx processes:

```bash
for pid in $(pgrep nginx); do echo "PID=$pid"; cat /proc/$pid/limits | grep "Max open files"; done
```

---

## Scenario 17: Check Current Open File Count for Nginx

Check master:

```bash
ls /proc/$(pgrep -o nginx)/fd | wc -l
```

Check all Nginx processes:

```bash
for pid in $(pgrep nginx); do echo -n "PID=$pid "; ls /proc/$pid/fd | wc -l; done
```

---

## Scenario 18: Check System Used File Handles

```bash
cat /proc/sys/fs/file-nr
```

The output usually has 3 numbers:

```text
Number of files assigned

No file handle

Maximum number of file handles of the system
```

Check system maximum file handles:

```bash
cat /proc/sys/fs/file-max
```

---

## VIII. worker_rlimit_nofile

---

## Scenario 19: What Is worker_rlimit_nofile

Configuration example:

```nginx
worker_rlimit_nofile 65535;
```

Function:

```text
Settings Nginx worker Process opens the maximum number of file describers
```

Usually written in the main global configuration:

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 8192;
}
```

---

## Scenario 20: Relationship Between worker_rlimit_nofile and worker_connections

Assume:

```text
worker_connections = 8192
```

If file descriptor limit is only:

```text
1024
```

Then even if worker_connections is set to a large value, it may still be limited by file descriptor constraints.

Recommendation:

```text
worker_rlimit_nofile >= worker_connections × 2 Or higher
```

In reverse proxy scenarios, it's even more conservative because there are also upstream connections and other file handles.

---

## IX. systemd LimitNOFILE

---

## Scenario 21: Why Changing ulimit Alone May Not Take Effect

If Nginx is managed by systemd:

```bash
systemctl start nginx
```

The file descriptor limit for Nginx is typically controlled by the systemd service.

Only execute in the current shell:

```bash
ulimit -n 65535
```

This does not necessarily affect Nginx started by systemd.

---

## Scenario 22: Check Nginx systemd Configuration

```bash
systemctl cat nginx
```

Check for LimitNOFILE:

```bash
systemctl cat nginx | grep -i LimitNOFILE
```

---

## Scenario 23: Configure systemd LimitNOFILE

Create an override:

```bash
systemctl edit nginx
```

Write:

```ini
[Service]
LimitNOFILE=65535
```

Reload systemd:

```bash
systemctl daemon-reload
```

Restart Nginx:

```bash
systemctl restart nginx
```

Verify:

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
```

Note:

```text
Modify LimitNOFILE After that, usually. restart To make process restrictions work.

reload It does not necessarily change the system-level limitations of existing processes
```

---

## Ten. Recommended Base Capacity Configuration Example

---

## Scenario 24: Medium-Scale Reverse Proxy Configuration Example

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 8192;
    multi_accept on;
}

http {
    sendfile on;
    keepalive_timeout 65;
    server_tokens off;

    include /etc/nginx/conf.d/*.conf;
}
```

systemd override:

```ini
[Service]
LimitNOFILE=65535
```

Explanation:

```text
worker_processes auto
→ Automatic Matching CPU

worker_connections 8192
→ Each worker Maximum number of connections

worker_rlimit_nofile 65535
→ worker File description cap

LimitNOFILE
→ systemd Process file description cap
```

---

## Eleven. Relationship Between QPS, Concurrency, and Response Time

---

## Scenario 25: What is QPS

QPS represents:

```text
Queries Per Second

Request per second
```

Example:

```text
QPS = 1000

For processing per second 1000 Request
```

---

## Scenario 26: What is Concurrency

Concurrency represents:

```text
Requests being processed or maintained at the same time / Number of connections
```

Concurrency is not equal to QPS.

Example:

```text
Every request is short.

It can be higher with a small number of parallels. QPS
```

If requests are slow:

```text
The same. QPS It'll take more co-connections.
```

---

## Scenario 27: Formula for QPS, Concurrency, and Response Time

Approximate relationship:

```text
Rounded ≈ QPS × Average response time
```

It can also be transformed into:

```text
QPS ≈ Rounded / Average response time
```

Where response time is in seconds.

---

## Scenario 28: Example 1: Response Time of 100ms

Assume:

```text
Average response time = 0.1 sec

Number of concurrent requests = 1000
```

Then:

```text
QPS ≈ 1000 / 0.1 = 10000
```

---

## Scenario 29: Example 2: Response Time of 2 Seconds

Assume:

```text
Average response time = 2 sec

Number of concurrent requests = 1000
```

Then:

```text
QPS ≈ 1000 / 2 = 500
```

Conclusion:

```text
The same. 1000 Parallel

The slower the response,QPS Lower

The sooner we respond, the faster we respond.QPS Higher
```

Therefore, optimizing backend response time inherently improves system throughput.

---

## Twelve. Rough Estimation Method for Nginx Capacity

---

## Scenario 30: Estimation Steps

Recommend following these steps:

```text
Confirm. CPU Numerical

→ Confirm. worker_processes

→ Confirm. worker_connections

→ Confirm file description limit

→ Whether it's static resources or reverse agents.

→ Confirm average response time

→ Estimated carry-over

→ Use side by side / Estimated response time QPS

→ Pressure Validation

→ Plus safety.
```

---

## Scenario 31: Rough Estimation Example for Reverse Proxy Capacity

Assume:

```text
CPU Numerical = 4

worker_processes = 4

worker_connections = 8192

No. of theoretical connections = 4 × 8192 = 32768
```

Reverse proxy scenario roughly halves:

```text
Allows to carry clients side by side ≈ 32768 / 2 = 16384
```

Considering safety margin, estimate at 60%:

```text
Safe side by side. ≈ 16384 × 60% ≈ 9830
```

If average response time:

```text
0.2 sec
```

Rough QPS:

```text
QPS ≈ 9830 / 0.2 = 49150
```

Note:

```text
This is a theoretical estimate.

Could actually be. CPUbandwidth, backend, log IOI don't know.TLSlimits of kernel parameters

It has to be checked.
```

---

## Scenario 32: Why Estimated Values and Stress Test Values May Differ Significantly

Possible reasons:

```text
Backend processing slow

There's not enough pressure.

Network bandwidth is insufficient

TLS Higher cost

Log discs become bottlenecks

Nginx Configure Limits

File description limit

Limits on kernel parameters

Short connections make handshakes expensive.

Backend connect pool insufficient

Databases become bottlenecks

It's not true.
```

---

## Thirteen. Observing Current Connections

---

## Scenario 33: View Current TCP Connections for Nginx

```bash
ss -antp | grep nginx | head
```

Count Nginx-related connections:

```bash
ss -antp | grep nginx | wc -l
```

Count all connections by status:

```bash
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
```

---

## Scenario 34: Count Connection States for Port 80

```bash
ss -ant | grep ':80 ' | awk '{print $1}' | sort | uniq -c | sort -nr
```

---

## Scenario 35: Count Connection States for Port 443

```bash
ss -ant | grep ':443 ' | awk '{print $1}' | sort | uniq -c | sort -nr
```

---

## Scenario 36: View ESTABLISHED Connection Count

```bash
ss -ant state established | wc -l
```

View ESTABLISHED for port 80:

```bash
ss -ant state established | grep ':80 ' | wc -l
```

View ESTABLISHED for port 443:

```bash
ss -ant state established | grep ':443 ' | wc -l
```

---

## Scenario 37: View TIME_WAIT Count

```bash
ss -ant state time-wait | wc -l
```

Count TIME_WAIT by target port:

```bash
ss -ant state time-wait | awk '{print $4}' | awk -F: '{print $NF}' | sort | uniq -c | sort -nr | head
```

---

## Scenario 38: View CLOSE_WAIT Count

```bash
ss -ant state close-wait | wc -l
```

View CLOSE_WAIT details:

```bash
ss -antp state close-wait | head -n 50
```

Explanation:

```text
CLOSE_WAIT It's often related to the application of unconnected connections.

If it's big CLOSE_WAIT On backend process, focus on backend applications
```

---

## Fourteen. Connection State Meanings

---

## Scenario 39: Common TCP States

```text
ESTAB / ESTABLISHED
→ Other Organiser

TIME-WAIT
→ Active closeer waiting for the old connection to disappear

CLOSE-WAIT
→ The end is closed, the application is not closed

SYN-SENT
→ Launching connection

SYN-RECV
→ Copy that. SYNWaiting for the handshake to finish

LISTEN
→ Service listening
```

---

## Scenario 40: Problems Indicated by Different States

```text
ESTABLISHED A lot.
→ High parallel connections, possibly normal, or slow back end leading to accumulation.

TIME_WAIT A lot.
→ Short connections are many, connections are often created and closed

CLOSE_WAIT A lot.
→ Application not properly closed, focus on the corresponding process

SYN_RECV A lot.
→ Handshake line pressure. Could be a sudden flow or attack.

SYN_SENT A lot.
→ Active connector-to-end slow or inaccessible
```

---

## Fifteen. CPU Analysis

---

## Scenario 41: View Overall CPU

```bash
top
```

Or:

```bash
mpstat 1 5
```

If mpstat is not available:

```bash
apt install -y sysstat
```

Or:

```bash
yum install -y sysstat
```

---

## Scenario 42: View Nginx Process CPU

```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | grep nginx | head
```

---

## Scenario 43: View CPU per Worker

```bash
top -p $(pgrep -d',' nginx)
```

---

## Scenario 44: Common Causes of High CPU Usage

High CPU usage in Nginx may come from:

```text
It's a lot of requests.

HTTPS TLS Handshakes.

Short connections.

gzip Overcompression

Right. location or rewrite Complex

access_log Big.

Mass 4xx / 5xx Unusual requests

WAF / Lua / Third-party module logical heavy

Very many requests for static files

Balancing or attacking traffic
```

---

## Sixteen. Memory Analysis

---

## Scenario 45: View Memory

```bash
free -h
```

View memory usage for Nginx process:

```bash
ps -eo pid,ppid,cmd,%mem,rss,vsz --sort=-rss | grep nginx | head
```

---

## Scenario 46: Memory Focus Points for Nginx

Nginx typically doesn't consume much memory, but these scenarios increase memory pressure:

```text
Lots of connections

Large buffer Configure

Large Request Body Buffer

Big Response Buffer

Third party module

Lua Script

Cache Module

Mass worker

Long connections.
```

---

## Seventeen. Network Bandwidth Analysis

---

## Scenario 47: View Network Card Traffic

```bash
sar -n DEV 1 5
```

If sar is not available:

```bash
apt install -y sysstat
```

Or:

```bash
yum install -y sysstat
```

---

## Scenario 48: View Network Card Errors

```bash
ip -s link
```

View a specific network card:

```bash
ip -s link show eth0
```

---

## Scenario 49: Signs of Bandwidth Bottlenecks

Possible manifestations:

```text
Nginx CPU Not high.

The back end isn't slow.

But the response slows.

Can't get the download speed up.

Net traffic close to the limit.

Drop or Error Increase

Big file downloads affect other requests
```

Need to pay attention to:

```text
Internet bandwidth

Intranet bandwidth

SLB bandwidth

Cloud host bandwidth limit

Card package PPS

Big file download traffic
```

---

## Eighteen. Disk and Log IO Analysis

---

## Scenario 50: View Disk Usage

```bash
df -h
```

View size of log directory:

```bash
du -sh /var/log/nginx
```

View large log files:

```bash
find /var/log/nginx -type f -size +500M -exec ls -lh {} \;
```

---

## Scenario 51: View Disk IO

```bash
iostat -x 1 5
```

Focus on:

```text
%util

await

r/s

w/s

wkB/s

aqu-sz
```

---

## Scenario 52: Impact of access_log on Performance

High QPS can cause:

```text
Disk Writing Pressure

Log cutting pressure

Log capture pressure

CPU Formatting Log Costs

Log Platform Costs
```

Production recommendations:

```text
Retention required access_log

Noise reduction from health screening

Sample on demand or log for static resources

Use logrotate

Log directory plan disks separately

High-flow scene assessment log writing performance
```

Not recommended to simply disable all access_log, as it may affect:

```text
The barrier.

Audit

Status Code Statistics

Security analysis

Log Platform Alert
```

---

## Nineteen. Impact of keepalive on Performance

---

## Scenario 53: Client keepalive

Configuration:

```nginx
keepalive_timeout 65;
```

Function:

```text
Client and Nginx Reuse Between TCP Connection
```

Benefits:

```text
Reduction TCP Shake hands.

Lower Short Connection Costs

Raise the load efficiency of the page resource
```

Risks:

```text
Maintaining a free connection for a long time will take over the connection resources.

Super-connected several scenes need control. keepalive_timeout
```

---

## Scenario 54: Upstream keepalive

Configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;

    keepalive 64;
}

location / {
    proxy_pass http://app_backend;

    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

Function:

```text
Nginx Reuse Connection to Backend

Reduce Backend TCP Shake hands and... TIME_WAIT
```

Note:

```text
keepalive Not as big as you can.

Need to combine worker Maximum number of connections estimated for quantity and backend
```

Rough estimation:

```text
worker_processes × keepalive = Each upstream Free backend connections that may be maintained
```

Example: /think

```text
worker_processes = 4

keepalive = 64

Probably. 4 × 64 = 256 Free backend connection
```

---

## Twenty, Impact of HTTPS and gzip on Performance

---

## Scenario 55: HTTPS Performance Overhead

HTTPS adds compared to HTTP:

```text
TLS Shake hands.

Credentials consultations

Decrypt calculation

More CPU Consumption
```

Optimization directions:

```text
Open keepalive

Use TLSv1.2 / TLSv1.3

Rationalizing the certificate chain

Reduce Short Connections

Use front CDN / SLB Unmount TLS

Evaluation hardware CPU Capacity
```

---

## Scenario 56: gzip Performance Overhead

gzip benefits:

```text
Reduction of volume of transmission

Save bandwidth

Raise the weak web experience
```

Costs:

```text
Consumption CPU

The higher the compression level CPU The more you spend.

For compressed images, videos,zip Very little.
```

Basic configuration:

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 1k;
gzip_types text/plain text/css application/javascript application/json application/xml image/svg+xml;
```

Production recommendations:

```text
Do not blindly set gzip_comp_level 9

Usually. 4 Present. 6 More common

Pictures, videos, compression packages do not recompress
```

---

## Twenty-one, Basic Monitoring of Nginx stub_status

Detailed monitoring will be organized in the 15th chapter on observability. Here we only provide basic supplements for performance analysis.

---

## Scenario 57: Enabling stub_status

Configuration:

```nginx
server {
    listen 127.0.0.1:8088;
    server_name localhost;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

Check configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Access:

```bash
curl http://127.0.0.1:8088/nginx_status
```

Output example:

```text
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

---

## Scenario 58: Meaning of stub_status Fields

```text
Active connections
→ Current Active Connections

accepts
→ Number of connections accepted

handled
→ Number of connections processed

requests
→ Requests processed

Reading
→ Number of connections reading request headers

Writing
→ Returns the number of connections responding to the client

Waiting
→ keepalive Free pending connections
```

---

## Twenty-two, Basic Performance Testing

---

## Scenario 59: Pre-test Considerations

Must confirm before testing:

```text
Whether to allow pressure production

Whether to notify the team concerned

Whether to set a pressure window

Is there a roll-back and flow limit scheme?

Monitor NginxBackend, database, cache

Are you ready to stop the pressure test?

Is the pressurized machine sufficient?

Whether to avoid affecting real users
```

Do not perform random tests in production environment.

---

## Scenario 60: Simple Testing with ab

Installation:

```bash
apt install -y apache2-utils
```

Or:

```bash
yum install -y httpd-tools
```

Testing:

```bash
ab -n 10000 -c 100 http://127.0.0.1/
```

Meaning:

```text
-n 10000
→ Total request 10000

-c 100
→ Parallel 100
```

---

## Scenario 61: Testing with wrk

Installation methods vary by system.

Ubuntu can try:

```bash
apt install -y wrk
```

Testing:

```bash
wrk -t4 -c200 -d60s http://127.0.0.1/
```

Meaning:

```text
-t4
→ 4 Thread

-c200
→ 200 Conjunction

-d60s
→ Ongoing 60 sec
```

---

## Scenario 62: Observing During Testing

Observing CPU during testing:

```bash
top
```

Observing connections:

```bash
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
```

Observing Nginx logs:

```bash
tail -f /var/log/nginx/access.log
```

Observing error logs:

```bash
tail -f /var/log/nginx/error.log
```

Observing disk IO:

```bash
iostat -x 1 5
```

Observing network interface:

```bash
sar -n DEV 1 5
```

---

## Scenario 63: Test Results Cannot Directly Equal Production Capacity

Test results are affected by many factors:

```text
Pressure gauge performance

Compressor network

Whether it's a machine pressure.

Do we go to the public network?

Is the request true?

Is the backend real?

Data cache

Whether to open HTTPS

Whether to open access_log

Could not close temporary folder: %s

Passes CDN / SLB

Include access, access to databases
```

Production capacity should use:

```text
Pressure results

+

Online surveillance

+

Business peak

+

Safety balance

+

Fault downgrading capability
```

Jointly determined.

---

## Twenty-three, Common Performance Issue Troubleshooting Path

---

## Scenario 64: Nginx Connection Count Exhausted

Manifestations:

```text
New request connection difficulty

Reaction slow.

Error log appears worker_connections are not enough

Client connection failed

Upstream SLB Health checkup abnormal.
```

Troubleshooting:

```bash
grep -i "worker_connections are not enough" /var/log/nginx/error.log | tail -n 100
```

```bash
nginx -T | grep -n "worker_connections"
```

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
```

```bash
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
```

Handling directions:

```text
Increase worker_connections

Increase worker_rlimit_nofile

Increase systemd LimitNOFILE

Analyse whether slow requests lead to connection piles

Analysis of attack or abnormal connection

Analyse whether the backend slows
```

---

## Scenario 65: too many open files

Troubleshooting:

```bash
grep -i "too many open files" /var/log/nginx/error.log | tail -n 100
```

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
```

```bash
for pid in $(pgrep nginx); do echo -n "PID=$pid "; ls /proc/$pid/fd | wc -l; done
```

Handling:

```text
Configure worker_rlimit_nofile

Configure systemd LimitNOFILE

Confirm. worker_connections Reasonable.

Restart Nginx Let the restriction take effect.

Check for an abnormal connection leak.
```

---

## Scenario 66: Many TIME_WAIT

Troubleshooting:

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -ant state time-wait | awk '{print $4}' | awk -F: '{print $NF}' | sort | uniq -c | sort -nr | head
```

Common causes:

```text
Too many short connections

Client does not reconnect

Nginx Did you get to the back end? keepalive

HF health check-ups

Press short connections

Backend connection frequently closed
```

Optimization directions:

```text
Enable client keepalive

Enable upstream keepalive

Reduced frequency of meaningless health examinations

Avoid short-link pressure miscalculation

Check for backend support keepalive
```

---

## Scenario 67: Many CLOSE_WAIT

Troubleshooting:

```bash
ss -antp state close-wait | head -n 50
```

Process statistics:

```bash
ss -antp state close-wait | awk -F'users:' '{print $2}' | sort | uniq -c | sort -nr | head
```

Common causes:

```text
Some application did not properly close the connection

Backend Connection Leak

Client disconnected post-service not available close

Apply bug
```

Handling directions:

```text
Positioning CLOSE_WAIT Which process belongs to

Check to apply connection closure logic

Restarting the anomaly can only be temporary.

Need to repair application root causes
```

---

## Scenario 68: QPS Doesn't Rise But CPU Isn't High

Possible causes:

```text
Backend response slow

Database Slow

Network bandwidth limit

Connection Limit

File description limit

Pressure measuring bottlenecks

DNS / SLB / CDN Limits

Nginx Wait upstream

Log Disk IO Slow
```

Troubleshooting:

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r '[.request_time, .upstream_response_time, .uri] | @tsv' | head
```

```bash
iostat -x 1 5
```

```bash
sar -n DEV 1 5
```

---

## Scenario 69: High CPU

Troubleshooting:

```bash
top
```

```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head
```

```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | grep nginx | head
```

Possible causes:

```text
HTTPS Handshakes.

gzip Compress cost

Request surged.

Logs are booming.

The rules are complicated.

Malicious scan.

Cost of third-party modules
```

---

## Twenty-four, Production Capacity Assessment Checklist

---

## 1. Business Traffic

```text
Day QPS

Peak QPS

Peak duration

Activity flow available

Is there a reptile flow?

Whether large files are downloaded

Is there any? WebSocket / SSE Long Connection

Whether large files are uploaded
```

---

## 2. Request Characteristics

```text
Average response time

P95 / P99 Response Time

Proportion of static requests

Dynamic API Percentage

HTTPS Percentage

Request Body Size

Response Size

Long connection ratio

Slow interface ratio
```

---

## 3. Nginx Configuration

```text
worker_processes

worker_connections

worker_rlimit_nofile

LimitNOFILE

keepalive_timeout

upstream keepalive

proxy_read_timeout

access_log Format and Path

gzip Whether to open

TLS Configure
```

---

## 4. System Resources

```text
CPU Numerical

Memory

Netbandwidth

Disk IO

File description limit

kernel connection queue

System Max.

Log Directory Capacity
```

---

## 5. Backend Capabilities

```text
Backend Examples

Single instance QPS

Backend Connection Pool

Thread pool

Database Connection Pool

Cache Capacity

Slow check

Time-consuming external dependence
```

---

## Twenty-five, Production Notes

---

## 1. Theoretical Connection Count Is Not Actual Capacity

Theoretical:

```text
worker_processes × worker_connections
```

Is just a configuration upper limit.

Actual is also limited by:

```text
File Description

CPU

Memory

Network

Disk

Backend

TLS

Log

kernel parameters
```

Jointly.

---

## 2. Reverse Proxy Should Consider Bidirectional Connections

Nginx maintains simultaneously:

```text
Client Connection

Backend upstream Connection
```

Capacity estimation should be more conservative than static resource scenarios.

---

## 3. QPS Must Be Combined with Response Time

Same 1000 concurrent connections:

```text
Response 0.1 sec
→ Theory QPS High

Response 2 sec
→ QPS Apparently.
```

So backend slowness will directly reduce overall throughput.

---

## 4. File Descriptors Must Be Adjusted Together with Nginx Configuration

Only changing:

```nginx
worker_connections 8192;
```

Is insufficient.

Also need to check:

```text
worker_rlimit_nofile

systemd LimitNOFILE

/proc/PID/limits
```

---

## 5. Testing Must Be Close to Real Business

Do not just test:

```text
Static Home Page
```

And assume API can support same QPS.

Should test in real proportion:

```text
Static resources

Dynamic interface

Access Rights

Query Interface

Writing Interface

Upload Download

Slow Interface
```

---

## 6. access_log Cannot Be Blindly Disabled

Disabling access_log can reduce IO, but will lose:

```text
Capability

Audit capacity

Status Code Statistics

Security analysis

Capacity trend data
```

More reasonable is:

```text
Log Level

Health check noise reduction

Static resource sampling

Log platform governance

Independent Logboard
```

---

## 7. Performance Issues Should Be Layered Diagnosis

Order:

```text
Client

→ CDN / SLB

→ Nginx

→ Backend Services

→ Database / Cache / External dependency

→ Host Resources

→ Network links
```

Do not attribute all slowness to Nginx.

---

## Twenty-six, Common Commands in This Chapter

---

## Nginx Configuration Check

```bash
nginx -T | grep -n "worker_processes"
```

```bash
nginx -T | grep -n "worker_connections"
```

```bash
nginx -T | grep -n "worker_rlimit_nofile"
```

```bash
nginx -T | grep -n "keepalive"
```

```bash
nginx -T | grep -n "gzip"
```

---

## CPU and worker

```bash
nproc
```

```bash
lscpu
```

```bash
ps -ef | grep "nginx: worker" | grep -v grep
```

```bash
ps -ef | grep "nginx: worker" | grep -v grep | wc -l
```

```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | grep nginx | head
```

---

## File Descriptors

```bash
ulimit -n
```

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
```

```bash
for pid in $(pgrep nginx); do echo "PID=$pid"; cat /proc/$pid/limits | grep "Max open files"; done
```

```bash
for pid in $(pgrep nginx); do echo -n "PID=$pid "; ls /proc/$pid/fd | wc -l; done
```

```bash
cat /proc/sys/fs/file-nr
```

```bash
cat /proc/sys/fs/file-max
```

---

## systemd Limits

```bash
systemctl cat nginx
```

```bash
systemctl cat nginx | grep -i LimitNOFILE
```

```bash
systemctl edit nginx
```

```bash
systemctl daemon-reload
```

```bash
systemctl restart nginx
```

---

## Connection States

```bash
ss -antp | grep nginx | head
```

```bash
ss -antp | grep nginx | wc -l
```

```bash
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
```

```bash
ss -ant | grep ':80 ' | awk '{print $1}' | sort | uniq -c | sort -nr
```

```bash
ss -ant | grep ':443 ' | awk '{print $1}' | sort | uniq -c | sort -nr
```

```bash
ss -ant state established | wc -l
```

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -antp state close-wait | head -n 50
```

---

## System Resources

```bash
top
```

```bash
free -h
```

```bash
df -h
```

```bash
iostat -x 1 5
```

```bash
sar -n DEV 1 5
```

```bash
ip -s link
```

---

## Logs and Disk

```bash
du -sh /var/log/nginx
```

```bash
find /var/log/nginx -type f -size +500M -exec ls -lh {} \;
```

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
grep -i "too many open files" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "worker_connections are not enough" /var/log/nginx/error.log | tail -n 100
```

---

## stub_status

```bash
curl http://127.0.0.1:8088/nginx_status
```

---

## Pressure Testing

```bash
ab -n 10000 -c 100 http://127.0.0.1/
```

```bash
wrk -t4 -c200 -d60s http://127.0.0.1/
```

---

## Twenty-Seven, One-Sentence Summary

The core of Nginx performance and capacity analysis isn't just looking at QPS alone, but comprehensively considering:

```text
worker

Number of connections

File Description

CPU

Memory

Network

Disk Log

Backend Response Time

TLS / gzip Cost

Business request model
```

Theoretical Connection Limit:

```text
worker_processes × worker_connections
```

In reverse proxy scenarios, it should be more conservative:

```text
Client Connection + upstream Connection

A request may occupy a two-way connection resource
```

Relationship between QPS, concurrency, and response time:

```text
Parallel ≈ QPS × Average response time

QPS ≈ Parallel / Average response time
```

File descriptors must be checked simultaneously:

```text
worker_rlimit_nofile

systemd LimitNOFILE

/proc/PID/limits

worker_connections
```

Common bottleneck judgment:

```text
CPU High
→ TLSI don't know.gzipRequest volume, log, complex rules

Number of connections high
→ Backend slow, long connections, client slow,worker_connections Small

TIME_WAIT More
→ Short connections are many,keepalive Insufficient

CLOSE_WAIT More
→ Application connection closing anomaly

Disk IO High
→ access_log Writing under pressure

QPS I can't get up, but... CPU Not high.
→ Backend slowness, bandwidth, connection restrictions, pressurizer bottlenecks
```

Production recommendations:

```text
Don't judge the capacity with only theoretical values.

Don't just press static pages for real business.

Don't just change. Nginx Do Not Check Backend

Don't increase blindly. worker Number

Don't just change. worker_connections Forget File Description

Don't be brainless. access_log

The capacity assessment must combine pressure measurement, online monitoring, peak traffic and safety residuals
```