# 17-Nginx Production Access Layer Comprehensive Practice: Standard Architecture, Complete Configuration, Verification Checklist, and Fault Drill

#Nginx #Production Practice #Access Layer #Reverse Proxy #HTTPS #Real IP #Throttling #Security Enhancement #Observability #Change Management #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capability Expansion/17-Nginx Production Access Layer Comprehensive Practice: Standard Architecture, Complete Configuration, Verification Checklist, and Fault Drill.md

---

## I. Document Description

This article is a comprehensive practical guide for Nginx access layer operations and maintenance.

Previously, the following topics have been covered separately:

```text
Nginx Basic Configuration

server / location Matching

upstream Load Balancing

Reverse Proxy Parameters

HTTPS Certificates

Static Resources

Real IP

JSON Access Logs

Common Fault Troubleshooting

Performance Capacity Analysis

Throttling and Connection Control

upstream Advanced Governance

High Availability Architecture

Security Enhancement

Observability

Configuration Release and Change Management
```

This article focuses on combining these contents into a standard production access layer solution, including:

- Production Nginx architecture design
- Recommended directory structure
- Complete configuration breakdown
- Unified proxy headers
- Unified HTTPS configuration
- Unified security response headers
- Real IP configuration
- JSON access logs
- upstream backend pool
- API reverse proxy
- Static resource caching
- Management backend protection
- Throttling and connection control
- stub_status / exporter monitoring
- Certificate verification
- Pre-release verification checklist
- Post-release verification checklist
- Common fault drills
- Standardized production access layer inspection checklist

This article is the 17th in the Nginx Advanced SRE Capability Expansion series.

The objectives of this article are:

```text
To combine all previous Nginx knowledge into a production access layer solution

→ To design a relatively complete Nginx production configuration structure

→ To understand the function of each configuration segment

→ To complete pre-release verification according to the checklist

→ To conduct post-release observations according to the checklist

→ To perform drills for scenarios such as 502 errors, 504 errors, certificate expiration, throttling misfires, and node exceptions

→ To develop standardized governance capabilities for production Nginx access layers
```

---

## II. Standard Production Access Layer Architecture

Recommended basic link:

```text
User / Client

→ DNS

→ CDN / WAF / SLB

→ Multiple Nginx Servers

→ Backend Application upstream

→ Database / Redis / Message Queue / External Dependencies
```

In this link, Nginx is mainly responsible for:

```text
Domain name access

HTTPS termination

HTTP redirection to HTTPS

Reverse proxy

Static resource serving

Real IP cleaning

Access log recording

Basic throttling

Connection control

Security response headers

Sensitive path protection

upstream traffic distribution

Faulty node removal

Monitoring status exposure

Configuration change release
```

In one sentence:

```text
Nginx serves as the business entry layer; it is not just a forwarding tool but also the intersection point for traffic management, security, logging, stability, and change governance.
```

---

## III. Recommended Directory Structure

It is recommended to manage Nginx configurations in separate directories for production use.

Recommended structure:

```text
/etc/nginx/
├── nginx.conf
├── conf.d/
│   ├── example.com.conf
│   ├── admin.example.com.conf
│   └── download.example.com.conf
├── upstreams/
│   ├── app_backend.conf
│   └── admin-backend.conf
├── snippets/
│   ├── proxy-headers.conf
│   ├── proxy-timeouts.conf
│   ├── ssl-common.conf
│   ├── security-headers.conf
│   ├── real-ip.conf
│   └── log-format.conf
├── certs/
│   └── example.com/
│       ├── fullchain.pem
│       └── privkey.pem
└── backup/
```

Directory explanations:

```text
nginx.conf
→ Global configuration, events, http, include sections

conf.d/
→ Business server-specific configurations

upstreams/
→ Backend upstream pool configurations

snippets/
→ Universal configuration snippets

certs/
→ Certificate files

backup/
→ Configuration backup directory, should not be included in the main configuration
```

Notes:

```text
Do not include the backup directory in the conf.d/*.conf inclusion scope.

Do not commit the certificate private key to Git.

It is recommended to manage production configurations via Git.

Configuration releases must follow standard procedures.
```

---

## IV. Example of Main Configuration File nginx.conf

---

## Scenario 1: Basic Structure of nginx.conf

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid/etc/nginx/snippets/real-ip.conf

Example:

```nginx
set_real_ip_from 10.0.0.10;
set_real_ip_from 10.0.0.11;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Explanation:

```text
10.0.0.10 / 10.0.0.11
→ Trusted exit IP addresses for SLB, WAF, CDN, and upstream Nginx servers

real_ip_header X-Forwarded-For
→ Extracts the real client IP address from the X-Forwarded-For header

real_ip_recursive on
→ Recursively retrieves the real client IP address in cases of multiple proxy layers
```

Production prohibition:

```nginx
set_real_ip_from 0.0.0.0/0;
```

Reason:

```text
Any client could potentially forge the X-Forwarded-For header, leading to distorted IP whitelists, rate limiting settings, and audit logs
``````nginx
include /etc/nginx/snippets/proxy-timeouts.conf;
}

location / {
    root /data/www/example;
    index index.html;

    try_files $uri $uri/ /index.html;
}

location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    root /data/www/example;

    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}

location = /index.html {
    root /data/www/example;

    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
}
``````bash
openssl x509 -noout -modulus -in /etc/nginx/certs/example.com/fullchain.pem | openssl md5
```

```bash
openssl rsa -noout -modulus -in /etc/nginx/certs/example.com/privkey.pem | openssl md5
```

The two outputs should be identical.

---

## Scenario 20: Backend Node Check

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

---

## Scenario 21: Static Directory Check

```bash
ls -ld /data/www/example
```

```bash
ls -lh /data/www/example/index.html
```

```bash
find /data/www/example -maxdepth 2 -type f | head
```

---

## Chapter Seventeen: Post-Launch Verification Checklist

---

## Scenario 22: Reload

```bash
systemctl reload nginx
```

Check status:

```bash
systemctl status nginx
```

View logs:

```bash
journalctl -u nginx -n 100
```

---

## Scenario 23: HTTP Redirect Verification

```bash
curl -I http://example.com
```

Expected response:

```text
HTTP/1.1 301

Location: https://example.com/
```

---

## Scenario 24: HTTPS Verification

```bash
curl -I https://example.com
```

Verify with specified node:

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com
```

```bash
curl -I --resolve example.com:443:10.0.0.22 https://example.com
```

---

## Scenario 25: Health Check Verification

```bash
curl -i https://example.com/health
```

Expected response:

```text
HTTP/1.1 200 OK

ok
```

---

## Scenario 26: API Verification

```bash
curl -v https://example.com/api/health
```

View response headers:

```bash
curl -I https://example.com/api/health
```

---

## Scenario 27: SPA Refresh Verification

```bash
curl -I https://example.com/dashboard
```

Expected response:

```text
Return index.html

Not 404
```

---

## Scenario 28: Static Resource Cache Verification

```bash
curl -I https://example.com/static/app.js
```

Check:

```text
Cache-Control

Expires
```

The homepage should not be heavily cached:

```bash
curl -I https://example.com/index.html
```

---

## Scenario 29: Sensitive Paths Verification

```bash
curl -I https://example.com/.git/config
```

```bash
curl -I https://example.com/.env
```

```bash
curl -I https://example.com/backup.sql
```

Expected response:

```text
403

Or

404
```

---

## Scenario 30: Security Response Headers Verification

```bash
curl -I https://example.com | grep -Ei "Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Referrer-Policy|Permissions-Policy"
```

---

## Scenario 31: Real IP Verification

Construct XFF:

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

Check logs:

```bash
tail -n 20 /var/log/nginx/example.access.json.log
```

Confirm:

```text
Do not trust forged XFF without verification.

Check if remote_addr matches expectations.

Verify if realip_remote_addr records the previous proxy IP.
```

---

## Chapter Eighteen: Post-Launch Observations

---

## Scenario 32: Observing access.log

```bash
tail -f /var/log/nginx/example.access.json.log
```

Count status codes:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

---

## Scenario 33: Observing error.log

```bash
tail -f /var/log/nginx/example.error.log
```

Check for upstream errors:

```bash
grep -server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;

keepalive 64;
}

Check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

Verify upstream distribution:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

---

## Chapter 20: Fault Experiment 2: Backend Interface Timeout

---

## Scenario 39: Troubleshooting 504 Errors

View error logs:

```bash
grep -i "upstream timed out" /var/log/nginx/example.error.log | tail -n 100
```

View access.log:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 504) | [.time, .uri, .request_time, .upstream_response_time, .upstream_addr] | @tsv'
```

Directly request the slow backend interface:

```bash
curl -v http://10.0.0.21:8080/api/slow
```

Continue to investigate the backend:

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
journalctl -u app -n 100
```

Solution approach:

```text
First, confirm if the backend is slow.

Then, check the database, Redis, and external dependencies.

Do not blindly increase the proxy_read_timeout setting.

Consider making long-running task interfaces asynchronous.
```

---

## Chapter 21: Fault Experiment 3: Misuse of Rate Limiting

---

## Scenario 40: Troubleshooting Sudden Increase in 429 Errors

Count 429 errors:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

Count 429 URI errors:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .uri' | sort | uniq -c | sort -nr | head
```

Check the rate limiting configuration:

```bash
nginx -T 2>/dev/null | grep -n "limit_req" -A 10 -B 5
```

Check the real IP settings:

```bash
nginx -T 2>/dev/null | grep -n "set_real_ip_from" -A 10
``

Common causes:

```text
Incorrect real IP configuration, causing all users to be identified as SLB IP.

Too low rate setting.

Too small burst setting.

Health checks being subject to rate limiting.

Static resources also being limited.

A large number of mobile network users sharing the same outbound IP.
```

---

## Scenario 41: Quickly Mitigating the Damage Caused by Rate Limiting

Solution methods:

```text
Temporarily increase the rate limit.

Increase the burst value.

Remove rate limits from regular interfaces.

Only retain rate limits for high-risk interfaces such as login pages.

Correct the real IP configuration settings.

Exclude health check paths from rate limiting.
```

Check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

Monitor the situation:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | [.time, .remote_addr, .uri] | @tsv' | tail
```

---

## Chapter 22: Fault Experiment 4: Certificate Issues

---

## Scenario 42: Checking for Expired Certificates

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Check the certificate chain:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

Check the SAN field:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Scenario 43: Certificate Update Process

Back up the old certificate:

```bash
mkdir -p /tmp/cert-backup/example.com/$(date +%F-%H%M%S)
```

```bash
cp -a /etc/nginx/certs/example.com/* /tmp/cert-backup/example.com/$(date +%F-%H%M%S)/
```

Replace the certificate:

```bash
cp```text
Is the proxy timeout configured?

Should proxy_next_upstream be carefully configured?

Is max_fails/fail_timeout configured for upstream?

Is keepalive being used?
```

---

## 5. Log Layer

```text
Is JSON access logging being used?

Are hostnames being recorded?

Is the status being recorded?

Is request_time being recorded?

Is upstream_addr being recorded?

Is upstream_status being recorded?

Is upstream_response_time being recorded?

Are logs collected centrally?

Do logs have a retention period?

Are there disk alerts for the log directory?
```

---

## 6. Security Layer

```text
Are server_tokens disabled?

Is a default server configured?

Are illegal Hosts blocked?

Are .git/.env/backup files protected?

Is autoindex disabled?

Are secure response headers configured?

Does the management backend have an allowlist or authentication?

Is the request body size limited?

Is there any basic rate limiting?
```

---

## 7. Monitoring Layer

```text
Is stub_status enabled?

Has an exporter been deployed?

Is the Prometheus target UP?

Is there HTTP health checking?

Is there monitoring for certificate expiration?

Are there alerts for 5xx/499/429 errors?

Are there alerts for keywords in error.log?

Are there monitors for host CPU/memory/disk/network?
```

---

## 8. Change Management Layer

```text
Is configuration being stored in Git?

Is a backup made before release?

Is nginx -t executed before release?

Is a grayscale deployment performed first?

Is curl --resolve used to verify specific nodes?

Are there rollback files available?

Are there release records?

Are there observations after changes are made?
```

---

## Twenty-Five: Summary of Commonly Used Commands

---

## Configuration Checking

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T > /tmp/nginx-full-config-$(date +%F-%H%M%S).txt 2>&1
```

```bash
nginx -T 2>/dev/null | grep -n "server_name example.com" -A 100
```

```bash
nginx -T 2>/dev/null | grep -n "upstream app_backend" -A 30
```

```bash
nginx -T 2>/dev/null | grep -n "set_real_ip_from" -A 10
```

```bash
nginx -T 2>/dev/null | grep -n "limit_req" -A 10 -B 5
```

---

## Service Status

```bash
systemctl status nginx
```

```bash
systemctl reload nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
ps -ef | grep nginx | grep -v grep
```

```bash
ss -lntp | grep nginx
```

---

## HTTP/HTTPS Verification

```bash
curl -I http://example.com
```

```bash
curl -I https://example.com
```

```bash
curl -i https://example.com/health
```

```bash
curl -v https://example.com/api/health
```

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com
```

---

## Certificate Checking

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates
```

```bash
 openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

```bash
openssl x509 -noout -modulus -in /etc/nginx/certs/example.com/fullchain.pem | openssl md5
```

```bash
openssl rsa -noout -modulus -in /etc/nginx/certs/example.com/privkey.pem | openssl md5
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## Backend Checking

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

---

## Log Analysis

```bash
tail -f /var/log/nginx/example.access.json.log
```

```bash
tail -f /var/log/nginx/example.error.log
```

```bash
ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates"
done
```

---

## Monitoring Checks

```bash
curl http://127.0.0.1:8088/nginx_status
```

```bash
curl http://127.0.0.1:9113/metrics
```

```bash
curl -s http://127.0.0.1:9113/metrics | grep nginx
```

```bash
ss -lntp | grep ':9113'
```

---

## Backup and Rollback

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

```bash
cp -a /etc/nginx/conf.d/example.com.conf /tmp/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
cp -a /tmp/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## Summary in One Sentence

The core of a comprehensive Nginx production access layer lies in:

**Highly available architecture, standardized configuration, unified proxy parameters, use of real IP addresses, HTTPS security, structured logging, controllable throttling, comprehensive monitoring, the ability to roll back releases, and the capability to conduct fault drills.**

A standard production Nginx access layer should include at least the following features:

**Multiple nodes, HTTPS support, HTTP redirect to HTTPS, use of real IP addresses, JSON-based access logs, unified proxy headers, consistent timeout settings, an upstream backend pool, health checks, secure response headers, protection for sensitive paths, basic throttling mechanisms, stub_status and exporter functions, certificate expiration monitoring, a release process, and a rollback process.**

Before going live, it is essential to check the following:

**nginx -t, nginx -T, certificate validity period, certificate SANs, matching of certificates and private keys, health of upstream servers, existence of static directories, correct configuration of real IP addresses, throttling settings, secure response headers, and protection for sensitive paths.**

After going live, it is important to monitor the following:

**access.log, error.log, 5xx errors, 499 errors, 429 errors, request_time, upstream_response_time, upstream_addr, Prometheus metrics, HTTP health checks, and certificate monitoring.**

Production best practices include:

**Avoid using a single Nginx server for extended periods, refrain from manually altering production configuration, always run nginx -t before making changes, do not store backup files in the include directory, be cautious with trusting all X-Forwarded-For headers, prevent stub_status from being exposed publicly, never store private keys in Git repositories, avoid setting overly strict throttling rules without reason, and do not attribute all 502/504 errors to Nginx.** Additionally, the Nginx access layer must be designed to be observable, playable back, drillable, and reusable for learning purposes.