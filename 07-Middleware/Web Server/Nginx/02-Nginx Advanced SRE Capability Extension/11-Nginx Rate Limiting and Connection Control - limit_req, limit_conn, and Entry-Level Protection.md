# 11-Nginx Rate Limiting and Connection Control: limit_req, limit_conn, and Entry-Level Protection

#Nginx #Rate Limiting #Connection Control #limit_req #limit_conn #Entry-Level Protection #Anti-Bot #Sudden Traffic #SRE #Stability Governance

---

## Recommended Reading Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capability Expansion/11-Nginx Rate Limiting and Connection Control: limit_req, limit_conn, and Entry-Level Protection.md

---

## I. Document Overview

This document outlines the commonly used rate limiting, connection control, and basic traffic protection methods in Nginx at the production entry layer.

Key points include:

- Why rate limiting is necessary at the entry layer
- The scope of Nginx's rate limiting capabilities
- `limit_req`: Request rate limitation
- `limit_req_zone`: Shared memory area for rate limits
- `rate`: Request rate per unit time
- `burst`: Queue for sudden request spikes
- `nodelay`: Immediate processing of sudden requests
- `limit_conn`: Concurrent connection limit
- `limit_conn_zone`: Shared area for concurrent connection limits
- Rate limiting by IP address
- Rate limiting by server_name
- Rate limiting by URI/API dimension
- Anti-bot measures for login interfaces
- Protection for administrative interfaces
- Connection control for download interfaces
- Connection timeout settings
- Protection against slow clients
- Analysis of rate limiting status codes and logs
- Tips for configuring production-level rate limiting
- Common configuration errors and troubleshooting methods

This document is part of the Nginx Advanced SRE Capability Expansion series, chapter 11.

Learning objectives:

```text
Understand the value of entry-layer rate limiting

→ Be able to configure IP-based request rate limits

→ Be able to configure IP-based concurrent connection limits

→ Differentiate between `rate`, `burst`, and `nodelay`

→ Distinguish between request rate limiting and concurrent connection limiting

→ Design protection strategies for scenarios such as login, upload, download, and administrative interfaces

→ Determine whether rate limiting is effective by checking access.log/error.log files

→ Avoid causing business disruptions due to incorrect rate limiting configurations
```

---

## II. Why Nginx Requires Rate Limiting and Connection Control

In production environments, the entry layer may face:

```text
Sudden traffic spikes

Malicious scans

Excessive requests to interfaces

Login attempts from bots

Crawling by spiders

Large file downloads

Slow client connections

Repeated attempts by faulty clients

Overload of downstream services due to upstream traffic

Temporary increases in traffic volume

Abnormal high-frequency access from a specific IP address

Concentrated calls to a particular interface
```

If no protection is implemented at the entry layer, it may lead to:

```text
Overloading of backend services

Exhaustion of database connection pools

Full utilization of Redis resources

Depletion of application thread pools

Accumulation of Nginx connections

Slow response times for legitimate user requests

Unavailability of critical interfaces

An avalanche of alerts

Escalation of system failures
```

The purpose of entry-layer rate limiting is not to "block all abnormal traffic," but rather to:

```text
Protect the backend during peak traffic periods

Dampen sudden spikes in traffic

Minimize the impact of malicious requests

Serve as the first line of defense before system failures spread
```

In short:

```text
Nginx rate limiting is a means of ensuring entry-layer stability, but it is not a complete risk control system, nor does it replace WAF.
```

---

## III. Scope of Nginx's Rate Limiting Capabilities

Nginx can perform the following tasks:

```text
Limit request rates based on IP address

Limit the number of concurrent connections per IP address

Limit the number of connections per server_name

Apply different rate limiting policies based on URI

Provide basic anti-bot protection for login interfaces

Control connections to download interfaces

Protect administrative access

Dampen sudden spikes in traffic
```

However, Nginx is not suitable for handling:

```text
Complex user-level risk control mechanisms

CAPTCHA systems

Detection of abnormal account activities

Device fingerprint recognition

Business-specific rate limits

Distributed and comprehensive rate limiting

Unified rate limiting across data centers

Advanced dynamic blocklists

Detailed user profile-based risk assessment
```

These tasks are better suited for:

```text
Application layers

API gateways

Service meshes

WAF systems

Comprehensive risk control platforms

Redis-based distributed rate limiting solutions

Specialized traffic management platforms
```

---

## IV. Differences Between `limit_req` and `limit_conn`

Nginx uses two main types of rate limiting mechanisms:

```text
limit_req
→ Limits the rate at which requests can be made

limit_conn
→ Limits the number of concurrent connections allowed
```

---

## 1. limit```nginx
http {
    limit_req_zone $binary_remote_addr zone=login_req:10m rate=1 request per second;

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

**Applicable to:**  
Login interface, SMS verification code interface, Password reset interface, Email verification code interface.

**Explanation:**  
- Each IP is allowed a maximum of 1 request per second on average.  
- A temporary burst of up to 3 requests is allowed.  
- Requests exceeding this limit will be rejected.

**Note:**  
Nginx's login rate limiting can only provide basic protection against brute-force attacks. It cannot replace account-level risk control mechanisms, verification codes, or failure-limiting policies at the application layer.```nginx
http {
    limit_req_zone $binary_remote_addr zone=download_req:10m rate=2r/s;
    limit_conn_zone $binaryremote_addr zone=download_conn:10m;

    limit_req_status 429;
    limit_conn_status 429;

    server {
        listen 80;
        server_name example.com;

        location /api/download/ {
            proxy_pass http://app_backend;

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
``````nginx
listen 80;
server_name download.example.com;

location /files/ {
    limit_req zone=download_req burst=5 nodelay;
    limit_conn download_conn 3;

    alias /data/files/;

    send_timeout 300s;
}
}
```

**Description:**

- Limits the rate of download requests.
- Restricts the number of concurrent download connections per IP address.
- Helps prevent a single IP from consuming excessive bandwidth.

---

## Scenario 33: Protecting Admin Templates in the Backend

```nginx
http {
    limit_req_zone $binary_remote_addr zone=admin_req:10m rate=3r/s;
    limit_conn_zone $binary_remote_addr zone(admin_conn:10m);

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

**Note:**

- If there is an SLB, CDN, or WAF in front of Nginx:
  - Use `$binary_remote_addr` for both `limit_req_zone` and `limit_conn_zone` configurations.
  - Ensure that `allow` and `deny` rules are based on the real IP address to avoid misjudgment of the proxy IP.

---

## Chapter 15: Methods for Testing Rate Limiting

---

## Scenario 34: Checking Configuration

```bash
nginx -t
```

**To view detailed configuration:**

```bash
nginx -T | grep -n "limit_req" -A 10 -B 5
nginx -T | grep -n "limit_conn" -A 10 -B 5
```

---

## Scenario 35: Reloading Configuration

```bash
systemctl reload nginx
```

**To check the status:**

```bash
systemctl status nginx
```

---

## Scenario 36: Testing Rate Limiting with Sequential Requests

```bash
for i in {1..30}; do curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/; done
```

**If rate limiting is effective, you should see results like this:**

```text
200

200

429

429
```

---

## Scenario 37: Testing Rate Limiting with Concurrent Requests using `xargs`

```bash
seq 1 50 | xargs -n1 -P20 -I{} curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/
```

**To analyze the results:**

```bash
seq 1 50 | xargs -n1 -P20 -I{} curl -s -o /dev/null -w "%{http_code}\n" -H "Host: example.com" http://127.0.0.1/api/ | sort | uniq -c | sort -nr | head
```

---

## Scenario 38: Checking Rate Limiting Logs in `error.log`

```bash
grep -i "limiting requests" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "limiting connections" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 39: Checking for Code 429 in `access.log`

```bash
awk '$9 == 429 {print $0}' /var/log/nginx/access.log | tail -n 50
```

**To count source IPs with code 429:**

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

**To count URLs with code 429:**

```bash
awk '$9 == 429 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Chapter 16: Real IP Addresses and Rate Limiting

---

## Scenario 40: Why Confirm theawk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head```bash
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

## Summary in One Sentence

The core of Nginx rate limiting and connection control lies in:

```text
limit_req
→ Controls the request rate

limit_conn
→ Controls the number of concurrent connections

client_*_timeout / send_timeout
→ Limits connections occupied by slow clients

keepalive_timeout
→ Controls idle long connections
```

Basic configuration for request rate limiting:

```nginx
limit_req_zone $binary_remote_addr zone=api_req:10m rate=10r/s;

location /api/ {
    limit_req zone=api_req burst=20 nodelay;
}
```

Basic configuration for connection limits:

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
→ Burst queue capacity

nodelay
→ No delay within the burst period, requests are released immediately

limit_req_status
→ Status code returned for request rate limiting

limit_conn_status
→ Status code returned for connection limits
```

Production recommendations:

```text
Apply moderate rate limiting to general APIs.

Implement stricter restrictions on login, CAPTCHA, and password reset interfaces.

Focus on limiting the number of connections for download interfaces.

Use whitelists and authentication in management backends.

Ensure health checks do not accidentally apply rate limits.

Always verify the actual IP address before implementing rate limiting.

Do not set `set_real_ip_from` to `0.0.0.0/0`.

Avoid using high基数 fields like `request_id` or full URLs as rate limiting keys.

Implement a gradual rollout of rate limiting measures and monitor the number of 429 errors and any unintended impacts.

Rate limiting is just one part of a comprehensive risk control system; it serves as the first line of defense at the entry point.
```