# 17-Nginx Production Access Layer Comprehensive Practical Guide: Standard Architecture, Complete Configuration, Verification Checklist, and Fault Simulation

#Nginx #ProductionOperations #AccessLayer #ReverseAgent #HTTPS #RealIp #LimitedFlow #Secure. #Observation #ChangeManagement #SRE

---

## Recommended Path

07-Middlewares/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/17-Nginx Production Access Layer Comprehensive Practical Guide: Standard Architecture, Complete Configuration, Verification Checklist, and Fault Simulation.md

---

## I. Document Description

This article is a comprehensive practical guide for the Nginx access layer operations series.

Previously, the following have been organized separately:

```text
Nginx Basic Configuration

server / location Match

upstream Load Balance

Reverse proxy arguments

HTTPS Certificate

Static resources

Real IP

JSON Access Log

Common failure screening

Performance capacity analysis

Flow and connection control

upstream High-level governance

High Available Structures

Secure.

Observation

Configure Release and Change Management
```

This article focuses on combining these contents into a standard production access layer solution, including:

- Production Nginx architecture design
- Recommended directory structure
- Complete configuration breakdown
- Unified proxy header
- Unified HTTPS configuration
- Unified security response headers
- Real IP configuration
- JSON access log
- Upstream backend pool
- API reverse proxy
- Static resource caching
- Management backend protection
- Rate limiting and connection control
- stub_status / exporter monitoring
- Certificate check
- Pre-deployment verification checklist
- Post-deployment verification checklist
- Common fault simulation
- Production access layer standardization checklist

This article is the 17th in the Nginx Advanced SRE Capabilities Expansion series.

This article's objectives:

```text
Can take all the things in front. Nginx Knowledge Portfolio to Productive Access Layer programme

→ It's more complete. Nginx Production configuration structure

→ Understands the effect of each profile

→ You can check the list before you go online.

→ It's on the list.

→ It works. 502I don't know.504. . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .

→ It can form production. Nginx Standardized governance capacity for access layer
```

---

## II. Production Access Layer Standard Architecture

Recommended base chain:

```text
User / Client

→ DNS

→ CDN / WAF / SLB

→ Multiple Nginx

→ Backend Application upstream

→ Database / Redis / Message queue / External dependency
```

In this chain, Nginx mainly handles:

```text
Domainname access

HTTPS Termination

HTTP Jump HTTPS

Reverse Agent

Static resource services

Real IP Clean

Access log records

Base limit stream

Connection control

Security Response

Sensitive Path Protection

upstream Flow distribution

Fault node removed

Monitor status exposure.

Configure Changes Release
```

One-sentence understanding:

```text
Nginx It is an entry level for operations, not just a forwarding tool, but a meeting point for traffic, security, logs, stability and change of governance.
```

---

## III. Recommended Directory Structure

For production, it is recommended to split and manage Nginx configurations.

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
│   └── admin_backend.conf
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

Directory description:

```text
nginx.conf
→ Global configuration,eventsI don't know.httpI don't know.include Entry

conf.d/
→ Operations server Configure

upstreams/
→ Backend upstream Pool Configuration

snippets/
→ Generic Configuration Snippet

certs/
→ Certificate File

backup/
→ Configure Backup Directory, should not be include Load
```

Note:

```text
backup Do not put the directory conf.d/*.conf Yes. include Scope

Certificate private key not submitted Git

Proposed integration of production configuration Git Management

Configuration releases should be implemented through standard processes
```

---

## IV. Main Configuration nginx.conf Example

---

## Scenario 1: nginx.conf Basic Structure

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 8192;
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server_tokens off;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    keepalive_timeout 65;

    client_header_timeout 10s;
    client_body_timeout 60s;
    send_timeout 60s;

    client_max_body_size 20m;

    include /etc/nginx/snippets/log-format.conf;
    include /etc/nginx/snippets/real-ip.conf;

    include /etc/nginx/upstreams/*.conf;
    include /etc/nginx/conf.d/*.conf;
}
```

Explanation:

```text
worker_processes auto
→ Automatic Matching CPU Core

worker_rlimit_nofile
→ Increase worker Could open file description limit

worker_connections
→ Each worker Maximum number of connections

server_tokens off
→ Hide Nginx Version Number

client_*_timeout
→ Basic slow client protection

include snippets
→ Unifiedly load public configuration clips

include upstreams
→ Unified Load Backend Pool

include conf.d
→ Harmonization of loading operations server
```

---

## V. Unified Log Format

---

## Scenario 2: JSON Access Log Configuration

File:

```text
/etc/nginx/snippets/log-format.conf
```

Configuration:

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"hostname":"$hostname",'
    '"remote_addr":"$remote_addr",'
    '"realip_remote_addr":"$realip_remote_addr",'
    '"xff":"$http_x_forwarded_for",'
    '"request_id":"$request_id",'
    '"host":"$host",'
    '"server_name":"$server_name",'
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

Explanation:

```text
escape=json
→ Prevention User-AgentI don't know.Referer The phrase is broken. JSON

hostname
→ Multiple Nginx to distinguish between nodes

remote_addr
→ Yes. real_ip Processed Client IP

realip_remote_addr
→ real_ip Replace previous jump agent IP

xff
→ Original X-Forwarded-For

request_time
→ Nginx The request is time-consuming.

upstream_response_time
→ Backend response time-consuming

upstream_addr
→ Backend of Actual Life
```

---

## VI. Real IP Configuration

---

## Scenario 3: real-ip.conf

File:

```text
/etc/nginx/snippets/real-ip.conf
```

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
→ SLB / WAF / CDN / Upstream Nginx Credible exits IP

real_ip_header X-Forwarded-For
→ From XFF Other Organiser IP

real_ip_recursive on
→ Recursively search for real clients when multiple agents IP
```

Production Prohibition:

```nginx
set_real_ip_from 0.0.0.0/0;
```

Reason:

```text
Any client can fake it. X-Forwarded-For

IP White lists, restricted flow, audit logs are all distorted.
```

---

## VII. Unified Proxy Header

---

## Scenario 4: proxy-headers.conf

File:

```text
/etc/nginx/snippets/proxy-headers.conf
```

Configuration:

```nginx
proxy_http_version 1.1;
proxy_set_header Connection "";

proxy_set_header Host $host;
proxy_set_header X-Request-ID $request_id;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;
```

Explanation:

```text
Host
→ Keep original Hostfacilitate backend recognition of domain names

X-Request-ID
→ Request tracking. ID

X-Real-IP
→ Single Real Client IP

X-Forwarded-For
→ Agency link IP

X-Forwarded-Proto
→ Tell Backend Original Protocol http / https

Connection ""
→ Cooperation upstream keepalive Use
```

---

## VIII. Unified Proxy Timeout

---

## Scenario 5: proxy-timeouts.conf

File:

```text
/etc/nginx/snippets/proxy-timeouts.conf
```

Configuration:

```nginx
proxy_connect_timeout 5s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 2;
proxy_next_upstream_timeout 5s;
```

Explanation:

```text
proxy_connect_timeout
→ Connect backend timeout

proxy_send_timeout
→ Send request timeout to backend

proxy_read_timeout
→ Waiting for backend response timeout

proxy_next_upstream
→ Try the next node when the backend fails

proxy_next_upstream_tries
→ Limit maximum number of attempts

proxy_next_upstream_timeout
→ Limit the total time to try again
```

Note:

```text
Do not try again without a brain.

Interfacing between orders, payments, vouchers, creation of resources, etc.
```

---

## IX. Unified HTTPS Configuration

---

## Scenario 6: ssl-common.conf

File:

```text
/etc/nginx/snippets/ssl-common.conf
```

Configuration:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers HIGH:!aNULL:!MD5;
ssl_prefer_server_ciphers off;

ssl_session_cache shared:SSL:20m;
ssl_session_timeout 10m;
ssl_session_tickets off;
```

Explanation:

```text
TLSv1.2 / TLSv1.3
→ Current common production baseline

ssl_session_cache
→ Raise TLS Reuse Session Efficiency

ssl_session_tickets off
→ Reduced session instrument key management risk
```

---

## X. Unified Security Response Headers

---

## Scenario 7: security-headers.conf

File:

```text
/etc/nginx/snippets/security-headers.conf
```

Configuration:

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Note:

```text
HSTS includeSubDomains and preload Don't add anything.

CSP To combine front-end physical authentication

The security response head is no substitute for the application of safety repairs.
```

---

## XI. Upstream Backend Pool

---

## Scenario 8: app_backend.conf

File:

```text
/etc/nginx/upstreams/app_backend.conf
```

Configuration:

```nginx
upstream app_backend {
    least_conn;

    server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;

    keepalive 64;
}
```

Explanation:

```text
least_conn
→ Prefer to a backend with fewer connections

max_fails / fail_timeout
→ Passive failure perception

keepalive
→ Nginx Reuse to Backend
```

---

## Scenario 9: admin_backend.conf

File:

```text
/etc/nginx/upstreams/admin_backend.conf
```

Configuration:

```nginx
upstream admin_backend {
    server 10.0.0.31:9090 max_fails=3 fail_timeout=30s;

    keepalive 32;
}
```

---

## XII. Main Site Complete server Configuration

---

## Scenario 10: example.com.conf

File:

```text
/etc/nginx/conf.d/example.com.conf
```

Configuration:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    include /etc/nginx/snippets/ssl-common.conf;
    include /etc/nginx/snippets/security-headers.conf;

    access_log /var/log/nginx/example.access.json.log access_json;
    error_log  /var/log/nginx/example.error.log warn;

    location = /health {
        access_log off;
        return 200 "ok\n";
    }

    location ~ /\. {
        return 404;
    }

    location ~* \.(bak|backup|old|orig|save|swp|sql|tar|tar.gz|tgz|rar|7z)$ {
        return 404;
    }

    location /api/ {
        proxy_pass http://app_backend;

        include /etc/nginx/snippets/proxy-headers.conf;
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
```

Explanation:

```text
80 server
→ HTTP Jump HTTPS

443 server
→ Official business portal

/health
→ SLB / blackbox / I don't know.

/api/
→ Inverse Agent Backend Service

/
→ Frontend SPA Static sites

Static resources
→ Long Cache

index.html
→ Ban Strong Cache

Hide Files and Backup Files
→ Back 404
```

---

## XIII. Management Backend Complete Configuration

---

## Scenario 11: admin.example.com.conf

File:

```text
/etc/nginx/conf.d/admin.example.com.conf
```

Configuration:

```nginx
server {
    listen 80;
    server_name admin.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name admin.example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    include /etc/nginx/snippets/ssl-common.conf;
    include /etc/nginx/snippets/security-headers.conf;

    access_log /var/log/nginx/admin.access.json.log access_json;
    error_log  /var/log/nginx/admin.error.log warn;

    location = /health {
        access_log off;
        return 200 "ok\n";
    }

    location / {
        allow 10.0.0.0/8;
        allow 192.168.0.0/16;
        deny all;

        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://admin_backend;

        include /etc/nginx/snippets/proxy-headers.conf;
        include /etc/nginx/snippets/proxy-timeouts.conf;
    }
}
```

Explanation:

```text
Control the backstage. Don't expose the net.

At least:

IP White list.

Basic Auth or official accreditation

HTTPS

Access Log

Real IP

Access if necessary VPN / Zero trust / SSO / MFA
```

---

## XIV. Rate Limiting Configuration Example

---

## Scenario 12: Define Rate Limiting Zones in the http Block

Add to the `nginx.conf` `http` block:

```nginx
limit_req_zone $binary_remote_addr zone=api_req:20m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login_req:10m rate=1r/s;

limit_conn_zone $binary_remote_addr zone=api_conn:20m;

limit_req_status 429;
limit_conn_status 429;
```

---

## Scenario 13: Ordinary API Rate Limiting

```nginx
location /api/ {
    limit_req zone=api_req burst=20 nodelay;
    limit_conn api_conn 20;

    proxy_pass http://app_backend;

    include /etc/nginx/snippets/proxy-headers.conf;
    include /etc/nginx/snippets/proxy-timeouts.conf;
}
```

---

## Scenario 14: Login Interface Rate Limiting

```nginx
location = /api/login {
    limit_req zone=login_req burst=3 nodelay;
    limit_conn api_conn 3;

    proxy_pass http://app_backend;

    include /etc/nginx/snippets/proxy-headers.conf;
    include /etc/nginx/snippets/proxy-timeouts.conf;
}
```

Note:

```text
Access limits are basic protection.

It also requires application layer authentication codes, account locking, number of failures, wind control policy.
```

---

## XV. stub_status Monitoring Configuration

---

## Scenario 15: Nginx Status Page

File:

```text
/etc/nginx/conf.d/nginx-status.conf
```

Configuration:

```nginx
server {
    listen 127.0.0.1:8088;
    server_name localhost;

    access_log off;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

Verification:

```bash
curl http://127.0.0.1:8088/nginx_status
```

---

## Scenario 16: nginx-prometheus-exporter Container

```bash
docker run -d \
  --name nginx-prometheus-exporter \
  --restart=always \
  --network host \
  nginx/nginx-prometheus-exporter:latest \
  --nginx.scrape-uri=http://127.0.0.1:8088/nginx_status \
  --web.listen-address=:9113
```

Verification:

```bash
curl http://127.0.0.1:9113/metrics
```

View Metrics:

```bash
curl -s http://127.0.0.1:9113/metrics | grep nginx
```

---

## XVI. Pre-Deployment Verification Checklist

---

## Scenario 17: Configuration Syntax Check

```bash
nginx -t
```

---

## Scenario 18: Complete Configuration Check

```bash
nginx -T > /tmp/nginx-full-config-before-release-$(date +%F-%H%M%S).txt 2>&1
```

Check server:

```bash
nginx -T 2>/dev/null | grep -n "server_name example.com" -A 100
```

Check upstream:

```bash
nginx -T 2>/dev/null | grep -n "upstream app_backend" -A 30
```

Check real_ip:

```bash
nginx -T 2>/dev/null | grep -n "set_real_ip_from" -A 10
```

Check certificate:

```bash
nginx -T 2>/dev/null | grep -n "ssl_certificate" -A 5 -B 5
```

---

## Scenario 19: Certificate Check

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates
```

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

Check if certificate and private key match:

```bash
openssl x509 -noout -modulus -in /etc/nginx/certs/example.com/fullchain.pem | openssl md5
```

```bash
openssl rsa -noout -modulus -in /etc/nginx/certs/example.com/privkey.pem | openssl md5
```

The two outputs should be consistent.

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

## XVII. Post-Deployment Verification Checklist

---

## Scenario 22: reload

```bash
systemctl reload nginx
```

Check status:

```bash
systemctl status nginx
```

Check logs:

```bash
journalctl -u nginx -n 100
```

---

## Scenario 23: HTTP Redirect Verification

```bash
curl -I http://example.com
```

Expected:

```text
HTTP/1.1 301

Location: https://example.com/
```

---

## Scenario 24: HTTPS Verification

```bash
curl -I https://example.com
```

Specify node verification: /think

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com
```

```bash
curl -I --resolve example.com:443:10.0.0.22 https://example.com
```

---

## Scenario 25: Health Check Validation

```bash
curl -i https://example.com/health
```

Expected:

```text
HTTP/1.1 200 OK

ok
```

---

## Scenario 26: API Validation

```bash
curl -v https://example.com/api/health
```

Check the response headers:

```bash
curl -I https://example.com/api/health
```

---

## Scenario 27: SPA Refresh Validation

```bash
curl -I https://example.com/dashboard
```

Expected:

```text
Back index.html

I shouldn't. 404
```

---

## Scenario 28: Static Resource Cache Validation

```bash
curl -I https://example.com/static/app.js
```

Pay attention to:

```text
Cache-Control

Expires
```

Home page should not use strong caching:

```bash
curl -I https://example.com/index.html
```

---

## Scenario 29: Sensitive Path Validation

```bash
curl -I https://example.com/.git/config
```

```bash
curl -I https://example.com/.env
```

```bash
curl -I https://example.com/backup.sql
```

Expected:

```text
403

or

404
```

---

## Scenario 30: Security Response Header Validation

```bash
curl -I https://example.com | grep -Ei "Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Referrer-Policy|Permissions-Policy"
```

---

## Scenario 31: Real IP Validation

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
No unconditional trust in forgery. XFF

remote_addr Compliance with expectations

realip_remote_addr Whether to record the previous jump agent IP
```

---

## Eighteen. Post-Deployment Monitoring

---

## Scenario 32: Monitor access.log

```bash
tail -f /var/log/nginx/example.access.json.log
```

Status code statistics:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

---

## Scenario 33: Monitor error.log

```bash
tail -f /var/log/nginx/example.error.log
```

Check upstream errors:

```bash
grep -i "upstream" /var/log/nginx/example.error.log | tail -n 100
```

Check timeouts:

```bash
grep -i "upstream timed out" /var/log/nginx/example.error.log | tail -n 100
```

Check rate limiting:

```bash
grep -Ei "limiting requests|limiting connections" /var/log/nginx/example.error.log | tail -n 100
```

---

## Scenario 34: Monitor 5xx

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .uri, .status, .upstream_addr, .upstream_status, .upstream_response_time] | @tsv' | tail
```

Statistics for 5xx upstream:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

---

## Scenario 35: Monitor Slow Requests

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .uri, .status, .request_time, .upstream_response_time, .upstream_addr] | @tsv' | tail
```

---

## Scenario 36: Monitor 429

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | [.time, .remote_addr, .uri, .status] | @tsv' | tail
```

Statistics for rate-limited IPs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

---

## Nineteen. Fault Simulation 1: Backend Node Failure

---

## Scenario 37: Simulate Backend Node Stop

Stop service on backend node:

```bash
systemctl stop app
```

Check from Nginx node:

```bash
curl -v http://10.0.0.21:8080/health
```

Check Nginx error log:

```bash
grep -Ei "connect\(\) failed|connection refused|upstream" /var/log/nginx/example.error.log | tail -n 100
```

Monitor access.log:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .uri, .status, .upstream_addr] | @tsv' | tail
```

---

## Scenario 38: Temporarily Remove Faulty Node

Modify upstream:

```nginx
upstream app_backend {
    least_conn;

    server 10.0.0.21:8080 down;
    server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;

    keepalive 64;
}
```

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

## Twenty. Fault Simulation 2: Backend Interface Timeout

---

## Scenario 39: 504 Troubleshooting Path

Check error log:

```bash
grep -i "upstream timed out" /var/log/nginx/example.error.log | tail -n 100
```

Check access.log:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 504) | [.time, .uri, .request_time, .upstream_response_time, .upstream_addr] | @tsv'
```

Directly request slow backend interface:

```bash
curl -v http://10.0.0.21:8080/api/slow
```

Continue troubleshooting backend:

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

Troubleshooting approach:

```text
Make sure the back end is slow.

Reconfirm database / Redis / External dependency

Don't just go blind. proxy_read_timeout

Long mission interfaces should consider hierarchization.
```

---

## Twenty-One. Fault Simulation 3: Rate Limiting Misfire

---

## Scenario 40: 429 Sudden Increase Troubleshooting

Statistics for 429:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

Statistics for 429 URI:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .uri' | sort | uniq -c | sort -nr | head
```

Check rate limiting configuration:

```bash
nginx -T 2>/dev/null | grep -n "limit_req" -A 10 -B 5
```

Check real IP:

```bash
nginx -T 2>/dev/null | grep -n "set_real_ip_from" -A 10
```

Common causes:

```text
Real IP Error. All users are considered SLB IP

rate Set too low

burst Set too small

Health check is restricted.

Static resources are restricted.

Large numbers of users of mobile networks share exports IP
```

---

## Scenario 41: Rapidly Mitigate Rate Limiting Misfire

Handling approach:

```text
Temporary easing rate

Increase burst

Get off the regular interface.

Only keep high-risk interfaces such as login restricted

Fix Real IP Configure

Exclude health check path
```

Check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

Monitor:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | [.time, .remote_addr, .uri] | @tsv' | tail
```

---

## Twenty-Two. Fault Simulation 4: Certificate Abnormality

---

## Scenario 42: Certificate Expiry Check

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Check certificate chain:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

Check SAN:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Scenario 43: Certificate Update Process

Backup old certificate:

```bash
mkdir -p /tmp/cert-backup/example.com/$(date +%F-%H%M%S)
```

```bash
cp -a /etc/nginx/certs/example.com/* /tmp/cert-backup/example.com/$(date +%F-%H%M%S)/
```

Replace certificate:

```bash
cp -a fullchain.pem /etc/nginx/certs/example.com/fullchain.pem
```

```bash
cp -a privkey.pem /etc/nginx/certs/example.com/privkey.pem
```

Set permissions:

```bash
chmod 644 /etc/nginx/certs/example.com/fullchain.pem
```

```bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

Check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

Verify:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## Twenty-Three. Fault Simulation 5: Inconsistent Nginx Configuration on a Node

---

## Scenario 44: Symptoms

User feedback:

```text
Sometimes it's normal.

Sometimes. 404

Sometimes the certificate is wrong.

Sometimes. 502

Sometimes you can visit, sometimes you can't.
```

If it's SLB + multiple Nginx nodes, it's likely:

```text
Multiple Nginx Configuration Inconsistent
```

---

## Scenario 45: Verify Each Node

HTTP:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTP ====="
    curl -I --resolve example.com:80:$host http://example.com/
done
```

HTTPS:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTPS ====="
    curl -I --resolve example.com:443:$host https://example.com/
done
```

Check configuration on each node:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    ssh root@$host "nginx -T 2>/dev/null | grep -n 'server_name example.com' -A 80"
done
```

Check certificate on each node:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates"
done
```

---

## Twenty-Four. Production Access Layer Standardization Checklist

---

## 1. Architecture Layer

```text
Is there at least two? Nginx

Is there something ahead? SLB / Keepalived / VIP / CDN

Is there a health check-up?

Whether to support the removal of abnormal nodes

Any excess capacity

Is there a multiple node configuration sync mechanism
```

---

## 2. HTTPS Layer

```text
Enable TLSv1.2 / TLSv1.3

Disable old protocols

Certificate not expired

Certificate SAN Whether to overwrite domain names

Complete certificate chain

Private key permission correct

HTTP Whether to jump HTTPS

Could not close temporary folder: %s
```

---

## 3. Real IP Layer

```text
Configure set_real_ip_from

Do you trust only credible agents?

Prohibited set_real_ip_from 0.0.0.0/0

Log or not remote_addr / realip_remote_addr / xff

allow / deny Is it based on truth? IP

Is the limit based on reality? IP
```

---

## 4. Proxy Layer

```text
Whether to pass Host

Whether to pass X-Real-IP

Whether to pass X-Forwarded-For

Whether to pass X-Forwarded-Proto

Configure proxy timeout

Whether to carefully configure proxy_next_upstream

upstream Configure max_fails / fail_timeout

Whether to use keepalive
```

---

## 5. Logging Layer

```text
Whether to use JSON access log

Record hostname

Record status

Record request_time

Record upstream_addr

Record upstream_status

Record upstream_response_time

Whether the logs are collected centrally

Does the log have a retention cycle?

Whether the log directory contains a disk message Police
```

---

## 6. Security Layer

```text
Whether to close server_tokens

Configure Defaults server

Interception of illegal Host

Protection .git / .env / Backup Files

Whether to close autoindex

Whether to configure a secure response header

Manage the availability of white lists or authentications in the back office

Whether to limit the size of the requested body

Is there a base limit?
```

---

## 7. Monitoring Layer

```text
Whether to open stub_status

Deployment exporter

Prometheus target Whether or not UP

Is there any? HTTP Search.

Could not close temporary folder: %s

Is there any? 5xx / 499 / 429 Police!

Is there any? error.log Keyword Alert

Do you have a host? CPU / Memory / Disk / Network monitoring
```

---

## 8. Change Layer

```text
Configure Entry Git

Whether to backup before release

Before publication nginx -t

Whether or not to publish one greyscale

Whether to use curl --resolve Specify node authentication

Other Organiser

Is there a release record?

Whether there is a change to observe
```

---

## Twenty-Five. Common Commands Summary in This Article

---

## Configuration Check

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

## HTTP / HTTPS Validation

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

## Certificate Check

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

## Backend Check

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

§§code_183§

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.status' | sort | uniq -c | sort -nr
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .uri, .status, .upstream_addr] | @tsv' | tail
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .uri, .request_time, .upstream_response_time, .upstream_addr] | @tsv' | tail
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

---

## error.log Keywords

```bash
grep -i "upstream" /var/log/nginx/example.error.log | tail -n 100
```

```bash
grep -i "upstream timed out" /var/log/nginx/example.error.log | tail -n 100
```

```bash
grep -Ei "connect\(\) failed|connection refused" /var/log/nginx/example.error.log | tail -n 100
```

```bash
grep -Ei "limiting requests|limiting connections" /var/log/nginx/example.error.log | tail -n 100
```

```bash
grep -i "too many open files" /var/log/nginx/example.error.log | tail -n 100
```

---

## Sensitive Path Check

```bash
curl -I https://example.com/.git/config
```

```bash
curl -I https://example.com/.env
```

```bash
curl -I https://example.com/backup.sql
```

```bash
curl -I https://example.com/config.bak
```

---

## Multi-Node Verification

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTPS ====="
    curl -I --resolve example.com:443:$host https://example.com/
done
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    ssh root@$host "nginx -t"
done
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    ssh root@$host "openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates"
done
```

---

## Monitoring Check

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

## Twenty-Six, One-Sentence Summary

The core of Nginx production access layer comprehensive practice is:

```text
Structure is highly usable

Configure Standardised

Harmonization of proxy parameters

Real IP Trustable.

HTTPS Clear.

Log Structure

It's controlled.

Monitor complete.

Release to Roll Back

Faultable exercise.
```

A standard production Nginx access layer should at least have:

```text
Multinodes

HTTPS

HTTP Jump HTTPS

Real IP

JSON access log

Harmonization proxy headers

Harmonization timeout

upstream Back-end pool

Health screening

Security Response

Sensitive Path Protection

Base limit stream

stub_status / exporter

Certificate Expiry Monitor

Release Process

Rollback Process
```

Must check before going live:

```text
nginx -t

nginx -T

Certificate Validity Period

Certificate SAN

Certificate and Private Key Match

upstream Backend health

Static Directory Exists

Real IP Configure

Stream Limit Configuration

Security Response

Sensitive Path Protection
```

Must observe after going live:

```text
access.log

error.log

5xx

499

429

request_time

upstream_response_time

upstream_addr

Prometheus target

HTTP Search.

Certificate Monitor
```

Production governance recommendations:

```text
No long-term single channel. Nginx

Don't change production configurations manually.

Don't skip nginx -t

Do not put the backup file in include Path

Don't trust everything. X-Forwarded-For

Don't let stub_status Visibility.

Do Not Allow Private Keys Git

Don't be brainless.

Don't take everything. 502 / 504 All due to... Nginx

Nginx Access layers must be visible, roll-backable, rehearsable, redisclosed
```