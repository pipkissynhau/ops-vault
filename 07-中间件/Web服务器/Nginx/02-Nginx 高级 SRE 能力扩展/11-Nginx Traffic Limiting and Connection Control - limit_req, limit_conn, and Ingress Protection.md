# 11-Nginx Rate Limiting and Connection Control: limit_req, limit_conn, and Entry-Level Protection

#Nginx #LimitedFlow #ConnectionControl #limit_req #limit_conn #LevelProtection #BrushProtection #Flows. #SRE #StableGovernance

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/11-Nginx Rate Limiting and Connection Control: limit_req, limit_conn, and Entry-Level Protection.md

---

## One: Document Overview

This document organizes common rate limiting, connection control, and basic traffic protection methods used in Nginx at the production entry layer.

This article focuses on:

- Why rate limiting is needed at the entry layer
- The scope of Nginx rate limiting capabilities
- `limit_req` Request rate limiting
- `limit_req_zone` Shared memory area
- `rate` Request rate
- `burst` Burst request queue
- `nodelay` Immediate processing of burst requests
- `limit_conn` Connection number limitation
- `limit_conn_zone` Connection limit shared area
- IP-based rate limiting
- server_name-based rate limiting
- URI/interface dimension rate limiting
- Login interface anti-brute-force protection
- Management backend protection
- Download interface connection control
- Connection timeout parameters
- Slow client protection
- Rate limiting status codes and log analysis
- Production rate limiting configuration considerations
- Common misconfiguration troubleshooting methods

This article is the 11th in the Nginx Advanced SRE Capabilities Expansion series.

This article's objectives:

```text
Understands the value of the portal limit.

→ Can Configure Press IP Request rate limit

→ Can Configure Press IP Number of connections limited

→ I understand. rateI don't know.burstI don't know.nodelay The difference.

→ Distinguishing requests from connections

→ Could design protection strategies for settings like login, uploading, downloading, managing backstage etc.

→ Yes. access.log / error.log Validation of restricted flow

→ It's a way to avoid a business accident due to a bad flow limit.
```

---

## Two: Why Nginx Needs Rate Limiting and Connection Control

In production environments, the entry layer may face:

```text
Flows.

Malicious scan.

Interface brush request

Login Blast

reptile capture

Download Big Files

Slow client connection

Unusual client retest

Upstream traffic blasts backend

The flow of activity is on the rise.

Some IP Unusual high frequency visits

Some interface is centrally called
```

If no protection is implemented at the entry layer, it may lead to:

```text
Back-end services are full.

Database connect pool depleted

Redis Filled up.

The application pool is exhausted.

Nginx Connected piles

Normal user request slow

Core interface not available

The alarm storm.

We're going into trouble.
```

The goal of entry layer rate limiting is not to "block all anomalies", but:

```text
Protect the backend when traffic is abnormal

Peaking in sudden flows

Reduced impact in malicious requests

First level of quarantine before trouble spreads.
```

One-sentence understanding:

```text
Nginx The limit is an entrance level stability protection, not a complete wind control system, not a system. WAF alternatives.
```

---

## Three: Nginx Rate Limiting Capabilities Scope

Nginx can do:

```text
Press IP Limiting request speed

Press IP Limit the number of simultaneous connections

Press server_name Limit number of connections

Press URI Match Different Stream Limit Policy

Basic brushing of login interfaces

Connection control of download interfaces

Access protection for management backstage

Peaking the sudden flow.
```

Nginx is not suitable for standalone completion:

```text
Complex user-level wind control

Authentication Code Policy

Account recognition abnormal.

Device fingerprint recognition

Operational level limit

Distributive full limit stream

I'm going to cross the room.

Complex Dynamic Blacklist

Finely fine user image control
```

These are better suited for:

```text
Application Layer

API Gateway

Service Grid

WAF

Wind control system

Redis Distribution restricted stream

Specialized traffic governance platform
```

---

## Four: Differences Between limit_req and limit_conn

Nginx commonly uses two types of limitations:

```text
limit_req
→ Limiting request speed

limit_conn
→ Limit the number of simultaneous connections
```

---

## 1. limit_req

`limit_req` Controls:

```text
Number of requests allowed within unit time
```

Example:

```text
Each IP Maximum per second 5 Request
```

Suitable for:

```text
Login interface brushing

SMS Protection

API HF access restrictions

Search interface protection

Prevention List IP Too high. QPS
```

---

## 2. limit_conn

`limit_conn` Controls:

```text
How many connections are allowed at the same time
```

Example:

```text
Each IP At the same time. 10 Connections
```

Suitable for:

```text
Download interface

Long Connection Interface

Large file transfer

Prevention List IP Too many connections.

Prevent slow client dragdown of connection resources
```

---

## 3. One-Sentence Difference

```text
limit_req
→ Control the frequency of requests

limit_conn
→ Control the number of simultaneous connections
```

Example:

```text
One. IP Request per second 100 Minor
→ Use limit_req Control

One. IP At the same time. 100 Download Connections
→ Use limit_conn Control
```

---

## Five: limit_req Basics

---

## Scenario 1: limit_req Basic Structure

`limit_req` Usually consists of two parts:

```text
limit_req_zone
→ Define restricted-flow area, restricted-flow area key, share memory, speed

limit_req
→ Yes. server or location Use this restricted stream area
```

Example:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=perip_req:10m rate=5r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            limit_req zone=perip_req burst=10 nodelay;

            proxy_pass http://app_backend;
        }
    }
}
```

---

## Scenario 2: limit_req_zone Parameter Explanation

Configuration:

```nginx
limit_req_zone $binary_remote_addr zone=perip_req:10m rate=5r/s;
```

Meaning:

```text
$binary_remote_addr
→ Use client IP As restricted flow key

zone=perip_req:10m
→ Define Name perip_req shared memory area, size 10MB

rate=5r/s
→ Each key Allowed per second 5 Request
```

---

## Scenario 3: Why $binary_remote_addr is Commonly Used

Common keys:

```nginx
$binary_remote_addr
```

Function:

```text
By Client IP Limited flow

Compare $remote_addr More efficient shared memory
```

You can also use:

```nginx
$remote_addr
```

But IP-based rate limiting is more common in production:

```nginx
$binary_remote_addr
```

---

## Scenario 4: Meaning of rate

Configuration:

```nginx
rate=5r/s
```

Represents:

```text
Every restricted stream key Average seconds 5 Request
```

You can also write:

```nginx
rate=60r/m
```

Represents:

```text
Every minute 60 Request
```

Common writing:

```nginx
rate=1r/s
```

```nginx
rate=5r/s
```

```nginx
rate=10r/s
```

```nginx
rate=60r/m
```

---

## Six: burst and nodelay

---

## Scenario 5: What is burst

Configuration:

```nginx
limit_req zone=perip_req burst=10;
```

Meaning:

```text
Allows short breakovers rate Request to enter waiting queue

Line up at the top. 10 Request
```

Without `nodelay`, requests exceeding the average rate but within the burst range will be delayed.

Simple understanding:

```text
rate
→ Average rate

burst
→ Sudden queue capacity
```

---

## Scenario 6: What is nodelay

Configuration:

```nginx
limit_req zone=perip_req burst=10 nodelay;
```

Meaning:

```text
burst Requests within scope are processed without delay

Over burst The request was denied directly.
```

Suitable for:

```text
Allow a short breakout

But they don't want the user to wait.

Hope to quickly reject it after crossing the threshold.
```

---

## Scenario 7: Behavior Without nodelay

Configuration:

```nginx
limit_req zone=perip_req burst=10;
```

Behavior:

```text
Over rate but not more burst Requests will be delayed in line.

Over burst Request denied.
```

Suitable for:

```text
You can accept an interface waiting in line.

Hope the flow is smooth.

I don't want any requests to hit the back end.
```

---

## Scenario 8: Behavior With nodelay

Configuration:

```nginx
limit_req zone=perip_req burst=10 nodelay;
```

Behavior:

```text
burst Request immediate release.

Over burst Rejected afterwards
```

Suitable for:

```text
User experience sensitive interface

Don't want to ask to wait in line.

I hope it fails quickly.
```

---

## Seven: limit_req Basic Examples

---

## Scenario 9: Limiting 5 Requests Per Second by IP

Configuration:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=perip_req:10m rate=5r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            limit_req zone=perip_req burst=10 nodelay;

            proxy_pass http://app_backend;
        }
    }
}
```

Meaning:

```text
Each IP Average seconds 5 Request

Allows an instantity. 10 Request

Reject after exceeding
```

---

## Scenario 10: Strict Rate Limiting for Login Interface

```nginx
http {
    limit_req_zone $binary_remote_addr zone=login_req:10m rate=1r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/login {
            limit_req zone=login_req burst=3 nodelay;

            proxy_pass http://app_backend;
        }
    }
}
```

Suitable for:

```text
Login Interface

SMS validation interface

Reset Password Interface

Mail Authentication Code Interface
```

Explanation:

```text
Each IP Average seconds 1 Minor

Allow a brief break. 3 Minor

Reject after exceeding
```

Note:

```text
Nginx The login limit can only be used as a basic shield.

No alternative to application layers such as account-level wind control, authentication code, failure locking
```

---

## Scenario 11: Search Interface Rate Limiting

```nginx
http {
    limit_req_zone $binary_remote_addr zone=search_req:10m rate=2r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/search {
            limit_req zone=search_req burst=5;

            proxy_pass http://app_backend;
        }
    }
}
```

Explanation:

```text
Search interfaces may consume database or search engine resources

Better than normal. API Tighter flow limits
```

---

## Eight: limit_req_status and Logs

---

## Scenario 12: Setting Rate Limiting Return Status Code

By default, `limit_req` commonly returns:

```text
503
```

Can be configured as:

```nginx
limit_req_status 429;
```

Complete example:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=perip_req:10m rate=5r/s;
    limit_req_status 429;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            limit_req zone=perip_req burst=10 nodelay;

            proxy_pass http://app_backend;
        }
    }
}
```

Recommended:

```text
Yeah. API The flow is restricted. 429 Too Many Requests Clearer.
```

---

## Scenario 13: Viewing Rate Limiting Logs

When requests are rate-limited, error.log may show similar information:

```text
limiting requests, excess: ...
```

View:

```bash
grep -i "limiting requests" /var/log/nginx/error.log | tail -n 100
```

Real-time viewing:

```bash
tail -f /var/log/nginx/error.log
```

---

## Scenario 14: Statistics for 429 Requests

Normal access.log:

```bash
awk '$9 == 429 {print $1,$7,$9}' /var/log/nginx/access.log | tail -n 50
```

Statistics for 429 IPs:

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Statistics for 429 URLs:

```bash
awk '$9 == 429 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

JSON access.log:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | [.time, .remote_addr, .uri, .status] | @tsv'
```

---

## Nine: limit_conn Basics

---

## Scenario 15: limit_conn Basic Structure

`limit_conn` Usually also has two parts:

```text
limit_conn_zone
→ Define Connection Limits key and share memory area

limit_conn
→ Yes. server or location Use this limit
```

Example:

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=perip_conn:10m;

    server {
        listen 80;
        server_name example.com;

        location /download/ {
            limit_conn perip_conn 3;

            root /data/www/example;
        }
    }
}
```

Meaning:

```text
Each IP At the same time. 3 Connection access /download/
```

---

## Scenario 16: limit_conn_zone Parameter Explanation

Configuration:

```nginx
limit_conn_zone $binary_remote_addr zone=perip_conn:10m;
```

Meaning:

```text
$binary_remote_addr
→ By Client IP Statistical Connections

zone=perip_conn:10m
→ Define shared memory area
```

---

## Scenario 17: limit_conn Parameter Explanation

Configuration:

```nginx
limit_conn perip_conn 3;
```

Meaning:

```text
Use perip_conn This connection limit area

Each key Maximum simultaneous connection is 3
```

---

## 10. limit_conn Example

---

## Scenario 18: Limiting Concurrent Connections per IP for Download Interface

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=perip_conn:10m;
    limit_conn_status 429;

    server {
        listen 80;
        server_name download.example.com;

        location /files/ {
            limit_conn perip_conn 3;

            alias /data/files/;
        }
    }
}
```

Applicable to:

```text
File Download

Install package download

Mirror Download

Report Download

Large file export result download
```

Purpose:

```text
Avoid individual IP Overloading both

Protect bandwidth and Nginx Connect Resource
```

---

## Scenario 19: Limiting Connection Count for Management Backend

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=admin_conn:10m;

    server {
        listen 80;
        server_name admin.example.com;

        location / {
            limit_conn admin_conn 5;

            proxy_pass http://admin_backend;
        }
    }
}
```

Description:

```text
Managing the backstage should not normally have a large number of simultaneous connections.

An abnormally high combined hair could be scanned, attacked or misused.
```

---

## Scenario 20: Limiting Total Connections by server_name

```nginx
http {
    limit_conn_zone $server_name zone=perserver_conn:10m;

    server {
        listen 80;
        server_name example.com;

        limit_conn perserver_conn 1000;

        location / {
            proxy_pass http://app_backend;
        }
    }
}
```

Meaning:

```text
Limit Some server_name Total number of connections
```

Applicable to:

```text
Protect the overall connection size of a virtual host

Limiting the use of individual domain names in multi-tensor scenarios
```

---

## 11. limit_conn_status and Logs

---

## Scenario 21: Setting Connection Limit Status Code

```nginx
limit_conn_status 429;
```

Complete Example:

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=perip_conn:10m;
    limit_conn_status 429;

    server {
        listen 80;
        server_name example.com;

        location /download/ {
            limit_conn perip_conn 3;

            alias /data/files/;
        }
    }
}
```

---

## Scenario 22: Viewing Connection Limit Logs

```bash
grep -i "limiting connections" /var/log/nginx/error.log | tail -n 100
```

Real-time Viewing:

```bash
tail -f /var/log/nginx/error.log
```

---

## 12. Request Rate Limiting and Connection Limiting Combination

---

## Scenario 23: Simultaneously Limiting Request Rate and Connection Count for Ordinary API

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_req:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=api_conn:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            limit_req zone=api_req burst=20 nodelay;
            limit_conn api_conn 20;

            proxy_pass http://app_backend;
        }
    }
}
```

Meaning:

```text
Each IP Average seconds 10 individual API Request

Allow surprises 20 individual

Each IP At the same time. 20 Connections
```

---

## Scenario 24: Combined Protection for Login Interface

```nginx
http {
    limit_req_zone $binary_remote_addr zone=login_req:10m rate=1r/s;
    limit_conn_zone $binary_remote_addr zone=login_conn:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name example.com;

        location = /api/login {
            limit_req zone=login_req burst=3 nodelay;
            limit_conn login_conn 3;

            proxy_pass http://app_backend;
        }
    }
}
```

Description:

```text
Limit the frequency of login requests

Limit Same IP Login Connect and Issue

Reduced impact of blasts and brushing interfaces
```

---

## 13. Slow Clients and Connection Protection

---

## Scenario 25: Why Prevent Slow Clients

Slow clients may manifest as:

```text
Send requests slowly Head

Send requests slowly Body

Read response slowly

Long time indentation connection

It's slow to upload.

Slow down.
```

Risks:

```text
Occupation Nginx Connect Resource

Occupation worker_connections

Use backend connection

Slow the entrance level

Causing Connection Stack
```

---

## Scenario 26: Client Request Header Timeout

```nginx
client_header_timeout 10s;
```

Function:

```text
Limit client time to send header
```

Applicable to:

```text
Prevent abnormal client long-term occupancy of connections

Reduce slow-motion impact.
```

---

## Scenario 27: Client Request Body Timeout

```nginx
client_body_timeout 60s;
```

Function:

```text
Limit client time to send requester
```

Applicable to:

```text
Uploading files

Form submission

Large JSON Request

Slow Upload Protection
```

Upload interfaces can appropriately increase the limit:

```nginx
location /api/upload/ {
    client_body_timeout 300s;
    client_max_body_size 500m;

    proxy_pass http://app_backend;
}
```

---

## Scenario 28: Response Send Timeout to Client

```nginx
send_timeout 60s;
```

Function:

```text
Limits Nginx Timeout between writing operations when sending response to client
```

Applicable to:

```text
Prevent the long-term non-receipt response by the client leading to occupancy of the connection
```

---

## Scenario 29: keepalive_timeout Controls Idle Connections

```nginx
keepalive_timeout 65;
```

Description:

```text
Client free long connection maintenance time
```

If a large number of idle connections consume resources, it can be appropriately reduced:

```nginx
keepalive_timeout 30;
```

But do not set it too low, as it will increase TCP handshake overhead.

---

## 14. Production Scenario Configuration Examples

---

## Scenario 30: Ordinary Business API Protection Template

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_req:20m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=api_conn:20m;

    limit_req_status 429;
    limit_conn_status 429;

    upstream app_backend {
        server 10.0.0.21:8080;
        server 10.0.0.22:8080;

        keepalive 64;
    }

    server {
        listen 80;
        server_name example.com;

        client_header_timeout 10s;
        client_body_timeout 60s;
        send_timeout 60s;
        keepalive_timeout 65;

        access_log /var/log/nginx/example.access.log;
        error_log  /var/log/nginx/example.error.log warn;

        location /api/ {
            limit_req zone=api_req burst=20 nodelay;
            limit_conn api_conn 20;

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
}
```

---

## Scenario 31: Login Interface Protection Template

```nginx
http {
    limit_req_zone $binary_remote_addr zone=login_req:10m rate=1r/s;
    limit_conn_zone $binary_remote_addr zone=login_conn:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name example.com;

        location = /api/login {
            limit_req zone=login_req burst=3 nodelay;
            limit_conn login_conn 3;

            proxy_pass http://app_backend;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

---

## Scenario 32: Download Interface Protection Template

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=download_conn:10m;
    limit_req_zone $binary_remote_addr zone=download_req:10m rate=2r/s;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name download.example.com;

        location /files/ {
            limit_req zone=download_req burst=5 nodelay;
            limit_conn download_conn 3;

            alias /data/files/;

            send_timeout 300s;
        }
    }
}
```

Description:

```text
Limit download request rate

Limits IP Number of connections downloaded simultaneously

Prevention List IP Excess bandwidth
```

---

## Scenario 33: Management Backend Protection Template

```nginx
http {
    limit_req_zone $binary_remote_addr zone=admin_req:10m rate=3r/s;
    limit_conn_zone $binary_remote_addr zone=admin_conn:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name admin.example.com;

        location / {
            allow 10.0.0.0/8;
            allow 192.168.0.0/16;
            deny all;

            limit_req zone=admin_req burst=10 nodelay;
            limit_conn admin_conn 5;

            proxy_pass http://admin_backend;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

Note:

```text
If Nginx It's up ahead. SLB / CDN / WAF

allow / deny It must be combined. real_ip Configure

Or it might be the agent. IP
```

---

## 15. Rate Limiting Verification Methods

---

## Scenario 34: Check Configuration

```bash
nginx -t
```

View Complete Configuration:

```bash
nginx -T | grep -n "limit_req" -A 10 -B 5
```

```bash
nginx -T | grep -n "limit_conn" -A 10 -B 5
```

---

## Scenario 35: Reload Configuration

```bash
systemctl reload nginx
```

View Status:

```bash
systemctl status nginx
```

---

## Scenario 36: Test Rate Limiting with Loop Requests

```bash
for i in {1..30}; do curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/; done
```

If rate limiting takes effect, you may see:

```text
200

200

429

429
```

---

## Scenario 37: Concurrent Testing for Rate Limiting

Use `xargs` for simple concurrency:

```bash
seq 1 50 | xargs -n1 -P20 -I{} curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/
```

Statistical Results:

```bash
seq 1 50 | xargs -n1 -P20 -I{} curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/ | sort | uniq -c
```

---

## Scenario 38: View error.log for Rate Limiting Records

```bash
grep -i "limiting requests" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "limiting connections" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 39: View access.log for 429

```bash
awk '$9 == 429 {print $0}' /var/log/nginx/access.log | tail -n 50
```

Statistical 429 Source IPs:

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Statistical 429 URLs:

```bash
awk '$9 == 429 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## 16. Real IP and Rate Limiting

---

## Scenario 40: Why Confirm Real IP Before Rate Limiting

If the link is:

```text
Client

→ SLB / CDN / WAF

→ Nginx
```

And Nginx is not configured with real_ip, then:

```text
$remote_addr = SLB / CDN / WAF IP
```

At this point, rate limiting based on `$binary_remote_addr` will become:

```text
All users share the same agent IP It's restricted.
```

Consequences:

```text
A large number of normal users were mistreated.

The restricted flow is not effective by real client

Visit log statistics anomaly
```

---

## Scenario 41: Configure real_ip Before Rate Limiting

Example:

```nginx
http {
    set_real_ip_from 10.0.0.10;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    limit_req_zone $binary_remote_addr zone=perip_req:10m rate=5r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            limit_req zone=perip_req burst=10 nodelay;

            proxy_pass http://app_backend;
        }
    }
}
```

Description:

```text
Let's go. $remote_addr Turn into a real client IP

Based on $binary_remote_addr Cut flow.
```

---

## Scenario 42: Do Not Trust All X-Forwarded-For

Dangerous Configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Risks:

```text
Clients can forge. X-Forwarded-For

That's how it goes around. IP Limited flow

Pollution Log

By the white list.
```

Correct Principle:

```text
Trust only SLB / CDN / WAF / Upstream Nginx Yes. IP Or a segment.
```

---

## 17. Common Rate Limiting Issues

---

## Scenario 43: Rate Limiting Not Taking Effect

Common Causes:

```text
limit_req_zone Yes, but no. limit_req References

limit_req It's written in error. location

The request didn't hit. location

Configure Not reload

The test is not fast enough.

burst Too big settings

nodelay Misunderstanding

Multiple Nginx Nodal assessment request

Front CDN / SLB Connection reuse or cache made
```

Troubleshooting:

```bash
nginx -T | grep -n "limit_req" -A 10 -B 5
```

```bash
nginx -T | grep -n "location" -A 10
```

```bash
tail -f /var/log/nginx/access.log
```

---

## Scenario 44: Rate Limiting Accidentally Affecting Many Users

Common Causes:

```text
Nginx See? SLB IP

Real IP Not configured

rate Set too low

burst Set too small

We've got a strict log-in limit on the whole station.

Health checks are restricted.

Static resources are also restricted.

Large numbers of users of mobile networks share exports IP
```

Troubleshooting:

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
nginx -T | grep -n "real_ip"
```

Handling Direction:

```text
Fix Real IP

Relax normal interfaces.

Strictly restricted flow of high-risk interfaces only

Exclusion of health check-ups

Distinction APIStatic resources, login interface
```

---

## Scenario 45: Too Many 429 Responses

Troubleshooting:

```bash
awk '$9 == 429 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Continue Judging:

```text
Whether the attack was real or not.

Increased flow of operational activities

Whether or not the client retests the storm

Whether an interface is not designed rationally

Whether the flow limit threshold is too low

Is it true? IP Error
```

---

## Scenario 46: limit_req_zone Memory Insufficient

Possible Errors:

```text
could not allocate node in limit_req zone
```

Troubleshooting:

```bash
grep -i "limit_req" /var/log/nginx/error.log | tail -n 100
```

Handling:

```text
Increase zone Share Memory

Lower key Base

Avoid high-base segments key
```

For example, from:

```nginx
limit_req_zone $binary_remote_addr zone=perip_req:1m rate=5r/s;
```

Adjust to:

```nginx
limit_req_zone $binary_remote_addr zone=perip_req:20m rate=5r/s;
```

---

## 18. Do Not Use High Cardinality Fields Arbitrarily as Rate Limiting Keys

---

## Scenario 47: Common Rate Limiting Keys

Common Low-risk Keys:

```nginx
$binary_remote_addr
```

```nginx
$server_name
```

You can also combine variables, but do so cautiously.

---

## Scenario 48: Not Recommended Keys

Not Recommended to Use Directly:

```text
$request_uri

$args

$http_user_agent

$request_id

$cookie_session

$http_authorization

Full URL

Random Arguments

trace_id
```

Reason:

```text
The base is extremely high.

Shared memory expansion

Flow limit effect differential

Could be a lot of structures built by the attackers. key Shoot! zone
```

---

## 19. Layered Suggestions for Rate Limiting Configuration

---

## 1. Ordinary API

Recommendations:

```text
Medium rate

Medium burst

Not too strict.

Avoiding misalignment of normal users
```

Example:

```nginx
limit_req_zone $binary_remote_addr zone=api_req:20m rate=10r/s;
```

---

## 2. Login / Verification Code / Password Reset

Recommendations:

```text
More stringent. rate

Less burst

Lock with application layer authentication codes and accounts

Logs must be recorded
```

Example:

```nginx
limit_req_zone $binary_remote_addr zone=login_req:10m rate=1r/s;
```

---

## 3. Download Interface

Recommendations:

```text
Focus limit connections

Appropriate limits on the speed of requests

Watch the bandwidth.
```

Example:

```nginx
limit_conn_zone $binary_remote_addr zone=download_conn:10m;
```

---

## 4. Management Backend

Recommendations:

```text
IP White list.

Basic Auth or official accreditation

Moderate limit flow

Real IP Configure correctly

Access Log Complete
```

---

## 5. Health Check

Recommendations:

```text
Don't delay the health check.

Alone. location

You can close unnecessary logs or noises.

But don't interfere. SLB / K8s Search.
```

Example:

§

## 20. Production Notes

---

## 1. Observe Traffic First, Then Configure Rate Limiting

Do not configure based on intuition:

```text
rate=1r/s

burst=1
```

You should first check:

```text
Normal user request frequency

Peak Request

Interface Call Mode

Whether to retry the moving end

Is there a front-end simultaneous request?

Whether there is a batch interface
```

---

## 2. Must Confirm Real IP Before Rate Limiting

If there is a proxy in front, you must first confirm: /think

```text
$remote_addr Whether it is a real client IP
```

Otherwise, it may trigger rate limiting based on SLB IP, potentially affecting all users.

---

## 3. Rate Limiting Recommendations: Start with Gradual Rollout

Production recommendations:

```text
Testing environment first.

Turn it on in a little bit.

Configure a lighter threshold first

Observation 429 Number

We'll tighten it.
```

---

## 4. 429 Errors Should Be Monitored in the Log Platform and Alarms

Recommended monitoring:

```text
429 Number

429 Percentage

Top IP

Top URL

Trends in restricted flow

Whether to focus on an interface
```

Avoid rate limiting incidents that go unnoticed.

---

## 5. Do Not Apply the Same Strict Rate Limiting to the Entire Site

Different paths should be distinguished:

```text
Home Page

Static resources

Normal API

Login Interface

Upload interface

Download interface

Manage backstage

Health screening
```

---

## 6. limit_req Is Not a Complete Anti-Attack Solution

Nginx basic rate limiting can only serve as the first layer of protection.

Complex attacks require:

```text
WAF

CDN

Business wind control

Authentication Code

Account Lock

IP Blacklist

Device fingerprint

Behaviour analysis

Upstream cleaning
```

---

## 7. Rate Limiting Configurations Must Be Rollback-Ready

Backup configurations before deployment:

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

If severe incidents occur, quickly recover:

```bash
cp -a /etc/nginx/conf.d/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

Check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## Twenty-One, Summary of Common Commands in This Article

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "limit_req" -A 10 -B 5
```

```bash
nginx -T | grep -n "limit_conn" -A 10 -B 5
```

```bash
nginx -T | grep -n "limit_req_zone"
```

```bash
nginx -T | grep -n "limit_conn_zone"
```

```bash
nginx -T | grep -n "real_ip"
```

---

## Service Management

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

---

## Rate Limiting Testing

```bash
for i in {1..30}; do curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/; done
```

```bash
seq 1 50 | xargs -n1 -P20 -I{} curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/
```

```bash
seq 1 50 | xargs -n1 -P20 -I{} curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/ | sort | uniq -c
```

---

## error.log Troubleshooting

```bash
grep -i "limiting requests" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "limiting connections" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "limit_req" /var/log/nginx/error.log | tail -n 100
```

```bash
tail -f /var/log/nginx/error.log
```

---

## access.log Analysis

```bash
awk '$9 == 429 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 429 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## JSON access.log Analysis

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | [.time, .remote_addr, .uri, .status] | @tsv'
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .uri' | sort | uniq -c | sort -nr | head
```

---

## Connection Observation

```bash
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
```

```bash
ss -ant state established | wc -l
```

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -antp | grep nginx | head
```

---

## Backup and Rollback

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
cp -a /etc/nginx/conf.d/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## Twenty-Two, One-Sentence Summary

The core of Nginx rate limiting and connection control is:

```text
limit_req
→ Control request rate

limit_conn
→ Control the number of simultaneous connections

client_*_timeout / send_timeout
→ Control slow client occupancy connection

keepalive_timeout
→ Control Free Long Connection
```

Basic request rate limiting configuration:

```nginx
limit_req_zone $binary_remote_addr zone=api_req:10m rate=10r/s;

location /api/ {
    limit_req zone=api_req burst=20 nodelay;
}
```

Basic connection limit configuration:

```nginx
limit_conn_zone $binary_remote_addr zone=download_conn:10m;

location /files/ {
    limit_conn download_conn 3;
}
```

Understanding key parameters:

```text
rate
→ Average request rate

burst
→ Sudden queue capacity

nodelay
→ No delay in the event of an outbreak, release directly.

limit_req_status
→ Request restricted flow back to status code

limit_conn_status
→ Connection limit return status code
```

Production recommendations:

```text
Normal API Moderate limit flow

Login, authentication code, password reset interface more stringent

Download interface focus limit connections

Manage backstage combination white lists and authentication

The health check should not be delayed.

We have to make sure it's true before it's restricted. IP

Do Not Configure set_real_ip_from 0.0.0.0/0

Don't use it. request_idComplete URL Waiting for high-base segments as restricted streams key

Get on the line. 429 Number and number of injuries

It's not a complete wind control system, it's just the first level of protection.
```