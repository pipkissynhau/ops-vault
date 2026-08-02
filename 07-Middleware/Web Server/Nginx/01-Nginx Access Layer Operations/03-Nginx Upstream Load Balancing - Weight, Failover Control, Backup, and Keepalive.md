# 03-Nginx upstream Load Balancing: Weight, Failure Control, Backup, and Keepalive

#Nginx #upstream #LoadBalance #ReverseAgent #AccessLayer #Middle #WebServer #Transport #SRE

---

## Recommended Path

07-Middlewares/Web Server/Nginx/01-Nginx Access Layer Operations/03-Nginx upstream Load Balancing: Weight, Failure Control, Backup, and Keepalive.md

---

## I. Document Explanation

This document organizes Nginx `upstream` backend service pool configuration and basic load balancing capabilities.

This article focuses on:

- What is upstream
- Relationship between upstream and proxy_pass
- Multi-backend load balancing
- Default round-robin strategy
- weight weights
- max_fails and fail_timeout
- backup standby nodes
- down temporarily offline nodes
- keepalive backend persistent connections
- ip_hash foundation
- least_conn foundation
- upstream naming conventions
- upstream common troubleshooting commands
- Handling logic when backend nodes fail
- Production environment considerations

This article is the 03rd article in the Nginx access layer operations series.

This article's goal:

```text
I can read it. upstream Configure

→ Can configure multiple backends

→ I understand. Nginx Default Load Balance Method

→ Available weight Control of traffic ratio

→ Available max_fails / fail_timeout Base Failed Control

→ Available backup Configure Backup Nodes

→ Available down Temporary removal of nodes

→ I understand. keepalive Reuse backend connection

→ I can check. upstream Relevant 502 / 504 / Connection failure
```

---

## II. What is upstream

`upstream` is used to define a group of backend services.

Simple understanding:

```text
upstream
→ Back-end service pool

proxy_pass
→ Forward the request to this back-end service pool.
```

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Request chain:

```text
Client

→ Nginx

→ app_backend

→ 10.0.0.21:8080 or 10.0.0.22:8080
```

One-sentence understanding:

```text
upstream Abstracts multiple backend examples into a single backend name.
```

---

## III. Why Need upstream

If only writing a single backend:

```nginx
location / {
    proxy_pass http://10.0.0.21:8080;
}
```

The problem is:

```text
Only one backend

I can't share the flow.

It's easy to influence business with a back end.

It's not easy to expand.

Toggle Backend Inconvenient

Cannot configure weights

Cannot configure backup nodes
```

After using upstream:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080;
}

location / {
    proxy_pass http://app_backend;
}
```

Can achieve:

```text
Multiple back-end distribution requests

Backend Horizontal amplification

Distribution of flows by weight

Node failure base removed

Backup node.

Configure clearer
```

---

## IV. Upstream Basic Structure

---

## Scenario 1: Basic upstream Configuration

Configuration example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Explanation:

```text
app_backend
→ upstream Name

server 10.0.0.21:8080
→ First Backend Node

server 10.0.0.22:8080
→ Second Backend Node

proxy_pass http://app_backend
→ Forward Request to app_backend This back-end pool.
```

Check configuration:

```bash
nginx -t
```

Reload configuration:

```bash
systemctl reload nginx
```

Verification entry:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Scenario 2: Where to Place upstream

`upstream` is usually written inside `http` block, cannot be written inside `server` or `location`.

Common writing:

```nginx
http {
    upstream app_backend {
        server 10.0.0.21:8080;
        server 10.0.0.22:8080;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://app_backend;
        }
    }
}
```

If using `/etc/nginx/conf.d/*.conf`, you can also directly write in sub-configuration files:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Prerequisite is that the file is included in `http` block:

```nginx
include /etc/nginx/conf.d/*.conf;
```

Check include:

```bash
grep -R "include" /etc/nginx/nginx.conf
```

---

## Scenario 3: Upstream Naming Conventions

Recommended naming:

```text
Business name_backend

Service Name_upstream

Environment_Service Name_backend
```

Example:

```nginx
upstream resume_api_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}
```

```nginx
upstream admin_web_backend {
    server 10.0.0.23:3000;
    server 10.0.0.24:3000;
}
```

Not recommended to name arbitrarily:

```nginx
upstream test {
    server 10.0.0.21:8080;
}
```

```nginx
upstream backend {
    server 10.0.0.21:8080;
}
```

Reason:

```text
I don't know what to do with more business.

Easy and other upstream Conflict

It's hard to locate when you're in a barrier.
```

---

## V. Default Load Balancing: Round Robin

Nginx upstream defaults to using round-robin.

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}
```

Default effect:

```text
I don't think so. 1 Request → 10.0.0.21:8080

I don't think so. 2 Request → 10.0.0.22:8080

I don't think so. 3 Request → 10.0.0.21:8080

I don't think so. 4 Request → 10.0.0.22:8080
```

Round-robin is suitable for:

```text
Back-end examples are similar.

The service capacity is almost equal.

No special session binding requirements

Normal non-state Web/API Services
```

---

## Scenario 4: Verifying Round-Robin Effect

If backend two instances return different content:

```text
10.0.0.21 Back app-1

10.0.0.22 Back app-2
```

Continuous requests:

```bash
for i in {1..10}; do curl -s -H "Host: example.com" http://127.0.0.1/; echo; done
```

May see:

```text
app-1
app-2
app-1
app-2
```

Note:

```text
Actual results are likely keepaliveImpacts, reconnection, cache, back-end response speed, etc.

Validate load balance with backend logs Look.
```

---

## VI. weight Weight Configuration

---

## Scenario 5: Configuring Different Weights

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=3;
    server 10.0.0.22:8080 weight=1;
}
```

Meaning:

```text
10.0.0.21 Weights 3

10.0.0.22 Weights 1
```

Approximate effect:

```text
Approximately 75% Other Organiser 10.0.0.21

Approximately 25% Other Organiser 10.0.0.22
```

Suitable scenarios:

```text
Backend machine configuration is different

A few connections for the new node.

Greyscale Release

Temporary reduction of a node flow

Some nodes are better.
```

---

## Scenario 6: Using Weight for Gray Release

For example, new version deployed in `10.0.0.22`, first give small traffic:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=9;
    server 10.0.0.22:8080 weight=1;
}
```

Meaning:

```text
The old version. 90%

New version of the convention 10%
```

After verifying the new version is stable, you can gradually adjust:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=5;
    server 10.0.0.22:8080 weight=5;
}
```

Finally fully switch:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 down;
    server 10.0.0.22:8080 weight=1;
}
```

Note:

```text
It's just a simple weight of ash.

More complicated. header / cookie / User-level greyscale will be collated in the senior governance section.
```

---

## VII. down Temporarily Offline Nodes

---

## Scenario 7: Temporarily Removing Backend Nodes

Configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080 down;
}
```

Meaning:

```text
10.0.0.22:8080 No more traffic.
```

Suitable scenarios:

```text
Node maintenance

Node Abnormal

There's a problem with the node version.

Examples of temporary removal failures

Grey Roll Back
```

After modification, check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## Scenario 8: Difference Between down and Directly Deleting Nodes

Using `down`:

```nginx
server 10.0.0.22:8080 down;
```

Advantages:

```text
Keep Profile Record

Facilitating follow-up recovery

I knew there was this node.

Roll back easy.
```

Direct deletion:

```nginx
# server 10.0.0.22:8080;
```

Or delete the entire line.

Suitable for:

```text
Node permanent exit

Service migration completed

Configure Cleanup
```

Production recommendation:

```text
Temporary removal down

permanent offline and delete
```

---

## VIII. backup Standby Nodes

---

## Scenario 9: Configuring backup Nodes

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080 backup;
}
```

Meaning:

```text
10.0.0.23 It's a backup node.

Requests are normally not received

Requests are accepted only when the main node is not available
```

Suitable scenarios:

```text
Temporary standby services

Downgrade services

Read-only Node

Low Specific Bottom Point

Disaster Backend
```

---

## Scenario 10: backup Usage Notes

backup nodes need to confirm:

```text
Is the service really available?

Data consistency

Whether traffic can be carried

Whether to read only

Whether or not to create operational inconsistencies

Do you need a special tip or downgrade? Response
```

Not recommended:

```text
Configure Any Unverified backup Nodes

backup Nodes are permanently unmaintained

backup Node data lags, but still picks up production flows
```

---

## IX. max_fails and fail_timeout

---

## Scenario 11: Basic Failure Control

Configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;
}
```

Meaning:

```text
max_fails=3
→ Yes. fail_timeout Time window, failed to reach 3 Minor

fail_timeout=30s
→ Failed to count window as 30 Seconds, and the time the node is considered unavailable is also relevant to it.
```

Simple understanding:

```text
30 Failed in seconds 3 Minor

Nginx For the time being, the node is considered unavailable.

Try again in a while.
```

---

## Scenario 12: What Counts as Failure

Common failures include:

```text
Failed to connect backend

Connection timed out

Read response timeout

Backend Close Connection Early

Partial proxy error
```

Note:

```text
HTTP 500 / 502 / 503 Whether or not to count failed, to combine proxy_next_upstream Configure Understanding

Default behaviour and specific version, configuration relevant

You don't have to rely on production. max_fails Understanding all health screening capabilities
```

Nginx open-source version's upstream health check is passive.

That is:

```text
Only if it's true and failed.

Nginx That's how you feel about the node.
```

---

## Scenario 13: Failure Control Configuration Example

More conservative configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.22:8080 max_fails=2 fail_timeout=10s;
}
```

More lenient configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=5 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=5 fail_timeout=30s;
}
```

Consider when choosing:

```text
Back end stability

Whether the network is vibrating or not

Whether to allow rapid removal

Misjudgement cost

Number of Backend Examples

Business error tolerance
```

---

## X. proxy_next_upstream Foundation

`proxy_next_upstream` is used to control when to try the next upstream node.

Example:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_next_upstream error timeout http_502 http_503 http_504;
}
```

Meaning:

```text
When there's a connection error, timeout,502I don't know.503I don't know.504 Time

Nginx Try to forward to the next backend
```

Common values:

```text
error
→ Error while connecting to backend, sending request, reading response

timeout
→ Timeout

http_500
→ Backend Return 500

http_502
→ Backend Return 502

http_503
→ Backend Return 503

http_504
→ Backend Return 504

off
→ Do not try the next node
```

---

## Scenario 14: Configuring Retry Count

Example:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
    proxy_next_upstream_timeout 5s;
}
```

Explanation:

```text
proxy_next_upstream_tries
→ Maximum number of attempts

proxy_next_upstream_timeout
→ Total retry time limit
```

Production note:

```text
Retesting increases availability

But it can also magnify back-end pressure.

Don't ask to be careful.
```

Non-idempotent request example:

```text
Create order

Payment of deductions

Submission of forms

Write Database

Send Message
```

Such requests may cause repeated business operations if forwarded repeatedly.

---

## XI. keepalive Backend Persistent Connections

---

## Scenario 15: Why upstream Needs keepalive

By default, Nginx may frequently establish connections to backend services.

A large number of short connections may bring:

```text
TCP It's a handshake.

Backend connection fluctuations

TIME_WAIT Increase

Port resource consumption

Increased back-end service pressure
```

`keepalive` allows Nginx to reuse connections to the backend.

---

## Scenario 16: Configuring upstream keepalive

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;

    keepalive 64;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

Explanation:

```text
keepalive 64
→ Each worker Hold until upstream Free connections

proxy_http_version 1.1
→ Use HTTP/1.1 Support long connections

proxy_set_header Connection ""
→ Avoid Sending to Backend Connection: close
```

---

## Scenario 17: Applicable scenarios for keepalive

Suitable for:

```text
High and Medium HTTP API

Nginx More short requests to the backend

Backend Support HTTP keepalive

Wishing to reduce the cost of connection construction

I hope it's down. TIME_WAIT
```

Notes to be aware of:

```text
keepalive Not as big as you can.

Use of backend connectivity resources beyond the General Assembly

To combine. worker Quantity and backend carrying capacity estimates
```

Example:

```text
worker_processes = 4

keepalive = 64

Theoretically. Nginx Maybe for each. upstream Hold 4 × 64 = 256 Free backend connection
```

---

## Twelve. ip_hash Basics

---

## Scenario 18: What is ip_hash

`ip_hash` distributes requests to a fixed backend based on the client's IP.

Example:

```nginx
upstream app_backend {
    ip_hash;

    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}
```

Suitable scenarios:

```text
Backend has local session

Need to get the same client to the same back end as possible

Keep Simple Sessions
```

---

## Scenario 19: Notes on ip_hash

Note:

```text
If Nginx There's more ahead. SLB / CDN / WAF

Nginx Yeah. remote_addr Could be an agent. IP

ip_hash Could lose real client significance.
```

Example:

```text
All requests come from the same source. SLB IP

ip_hash Could lead to a large number of requests going to the same back end.
```

Therefore, in real production environments, it is more recommended to use:

```text
Apply no status

session Fire! Redis

Use of unified accreditation centres

Avoid dependency Nginx ip_hash Keep core session
```

---

## Thirteen. least_conn Basics

---

## Scenario 20: What is least_conn

`least_conn` prioritizes forwarding requests to the backend with the fewest current connections.

Example:

```nginx
upstream app_backend {
    least_conn;

    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}
```

Suitable scenarios:

```text
There's a lot of time difference in requests.

Long connections.

Backend connections are uneven

Longer processing of some requests
```

Note:

```text
least_conn Not everything.

If back end performance varies significantly, it still needs to be combined weight

If the request is very short, it's usually enough.
```

---

## Fourteen. Complete upstream Configuration Example

---

## Scenario 21: Production Basic upstream Example

Configuration file:

```bash
vi /etc/nginx/conf.d/example.com.conf
```

Content:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 10.0.0.23:8080 backup;

    keepalive 64;
}

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 5s;
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

Validation:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

Check logs:

```bash
tail -f /var/log/nginx/example.access.log
```

```bash
tail -f /var/log/nginx/example.error.log
```

---

## Fifteen. Backend Checks for upstream

---

## Scenario 22: Checking each backend port

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
nc -zv -w 2 10.0.0.23 8080
```

---

## Scenario 23: Checking each backend HTTP response

```bash
curl -I http://10.0.0.21:8080
```

```bash
curl -I http://10.0.0.22:8080
```

```bash
curl -I http://10.0.0.23:8080
```

If there is a health check path:

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

```bash
curl -v http://10.0.0.23:8080/health
```

---

## Scenario 24: Verification through Nginx entry point

```bash
curl -v -H "Host: example.com" http://127.0.0.1/health
```

Continuous requests:

```bash
for i in {1..20}; do curl -s -H "Host: example.com" http://127.0.0.1/health; echo; done
```

---

## Scenario 25: Viewing Nginx connections to backend

View connections to backend 8080:

```bash
ss -antp | grep ':8080'
```

Statistical by status:

```bash
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c | sort -nr
```

View TIME_WAIT:

```bash
ss -ant state time-wait | grep ':8080' | wc -l
```

---

## Sixteen. Common upstream Errors

---

## Scenario 26: Upstream name written incorrectly

Error example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backnd;
    }
}
```

Problem:

```text
proxy_pass Reference app_backnd does not exist
```

Check:

```bash
nginx -t
```

---

## Scenario 27: Upstream written in wrong context

Error example:

```nginx
server {
    listen 80;

    upstream app_backend {
        server 10.0.0.21:8080;
    }
}
```

Problem:

```text
upstream It can't be written in server Internal
```

Check:

```bash
nginx -t
```

---

## Scenario 28: Backend port unreachable

Phenomenon:

```text
Visits Nginx Back 502
```

Error log may appear:

```text
connect() failed

connection refused

upstream timed out

no live upstreams
```

Troubleshoot:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
curl -v http://10.0.0.21:8080
```

---

## Scenario 29: All upstreams unavailable

Phenomenon:

```text
Nginx Back 502

error.log Come on. no live upstreams
```

Troubleshoot:

```bash
grep -i "no live upstreams" /var/log/nginx/error.log | tail -n 100
```

Check each backend individually:

```bash
for ip in 10.0.0.21 10.0.0.22 10.0.0.23; do echo "check $ip"; nc -zv -w 2 $ip 8080; done
```

---

## Seventeen. upstream Related 502 / 504 Troubleshooting

---

## Scenario 30: 502 Troubleshooting Path

Common causes of 502:

```text
Backend service not started

Backend is not listening

upstream IP Or wrong port

Nginx Backend's dead.

Backend Process Collapse

Backend Active Close Connection

Backend returns illegal response
```

Troubleshooting commands:

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
nginx -T | grep -n "upstream"
```

```bash
nginx -T | grep -n "proxy_pass"
```

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## Scenario 31: 504 Troubleshooting Path

Common causes of 504:

```text
Backend response timed out

Backend processing slow

Database slow query

Back-end pool full

Back-end connectors pool run out.

Backend CPU / Memory / IO Pressure high.

Nginx Timeout set too short

The network links are abnormal.
```

Troubleshooting commands:

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://BackendIP:Backend/health
```

```bash
top
```

```bash
free -h
```

```bash
iostat -x 1 5
```

```bash
ss -antp | grep Backend
```

---

## Eighteen. upstream Change Process

After production modification of upstream, it is recommended to follow:

```text
Confirm change target.

→ Backup Configuration

→ Modify upstream

→ nginx -t

→ Single Test Backend

→ reload Nginx

→ curl Nginx Entry

→ View access.log / error.log

→ Observation status code and operational indicators

→ Roll back as necessary
```

---

## Scenario 32: Backup configuration

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

---

## Scenario 33: Check after configuration modification

```bash
nginx -t
```

View full configuration:

```bash
nginx -T | grep -n "upstream app_backend" -A 20
```

---

## Scenario 34: Reload configuration

```bash
systemctl reload nginx
```

View status:

```bash
systemctl status nginx
```

---

## Scenario 35: Rollback configuration

```bash
cp -a /etc/nginx/conf.d/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

Check:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

---

## Nineteen. Production Notes

---

## 1. Backend must be validated separately first

Do not only modify Nginx.

First validate each backend:

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

---

## 2. Backup node cannot just be "written as backup"

Must regularly validate:

```text
backup Service started

backup Data availability

backup Compatibility of Versions

backup Can you take the flow?

backup Whether data consistency issues arise
```

---

## 3. down is suitable for temporary removal

Recommended temporary removal:

```nginx
server 10.0.0.22:8080 down;
```

Do not directly delete, convenient for rollback.

---

## 4. Weight adjustment should be done gradually

During gray release, it is recommended to:

```text
First 90 / 10

Again. 70 / 30

Again. 50 / 50

Final Full
```

Do not directly switch to full volume without verification.

---

## 5. keepalive is not better the larger

Need to combine with:

```text
worker_processes

keepalive Number

Maximum number of backend connections

Backend Connection Pool

Request and dispatch

Whether the backend supports long connections
```

---

## 6. Caution with retrying non-idempotent requests

For the following requests, be cautious in configuring `proxy_next_upstream`:

```text
Payments

Order.

Create Resource

Delete Resource

Submission of forms

Message Send
```

Avoid repeated forwarding of requests causing business duplication.

---

## 7. Open source Nginx is not complete active health check

Basic Nginx upstream is more passive failure detection.

Do not mistakenly believe:

```text
It's configured. max_fails It's an active health check.
```

Active health check usually requires:

```text
Nginx Plus

Third party module

External health check-ups and configuration management

Service Discovery System

Ingress Controller / Gateway capacity
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
nginx -T | grep -n "upstream"
```

```bash
nginx -T | grep -n "proxy_pass"
```

```bash
nginx -T | grep -n "upstream app_backend" -A 20
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

## Backend Port Check

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
nc -zv -w 2 10.0.0.23 8080
```

```bash
for ip in 10.0.0.21 10.0.0.22 10.0.0.23; do echo "check $ip"; nc -zv -w 2 $ip 8080; done
```

---

## Backend HTTP Check

```bash
curl -I http://10.0.0.21:8080
```

```bash
curl -I http://10.0.0.22:8080
```

§

---

## Nginx Entry Point Verification

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/health
```

```bash
for i in {1..20}; do curl -s -H "Host: example.com" http://127.0.0.1/health; echo; done
```

---

## Log Troubleshooting

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "connect() failed" /var/log/nginx/error.log | tail -n 50
```

```bash
grep -i "no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

---

## Connection Monitoring

```bash
ss -antp | grep ':8080'
```

```bash
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c | sort -nr
```

§


---

## Configuration Backup

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## Twenty-one, One-Sentence Summary

The core function of upstream is:

```text
Forming multiple backend examples into a backend service pool

Again. proxy_pass Unique forwarding
```

Basic structure:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

location / {
    proxy_pass http://app_backend;
}
```

Common capabilities:

```text
Default rotation
→ Backend instance rotates requests

weight
→ Control of traffic ratio

down
→ Temporary removal of nodes

backup
→ When main node is not available

max_fails / fail_timeout
→ Base Failed Control

keepalive
→ Resume connections to the backend

ip_hash
→ Client based IP Keep Simple Sessions

least_conn
→ Prioritize forwarding to less connected backends
```

Production recommendations:

```text
upstream The back end must be left alone. curl Authentication

Weights will be adjusted gradually.

Preference for temporary removal nodes down

backup Node must be regularly validated

keepalive To combine. worker Digital and back-end connectivity capacity assessment

I'm asking for caution. proxy_next_upstream Try again

Nginx Open Source Foundation upstream It's not a complete active health check.

Modify upstream We'll get back up.reload I have to. nginx -t
```