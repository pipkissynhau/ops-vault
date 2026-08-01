# 14-Nginx Security Hardening: TLS, Security Response Headers, Trusted IP Boundary, and Sensitive Path Protection

#Nginx #Secure. #TLS #SecurityResponse #RealIp #SensitivePathProtection #AccessControl #HostHeader #SRE #AccessLevelSecure.

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/14-Nginx Security Hardening: TLS, Security Response Headers, Trusted IP Boundary, and Sensitive Path Protection.md

---

## I. Document Overview

This document organizes common security hardening methods for Nginx in production access layers.

This article focuses on:

- Nginx Access Layer Security Boundary
- Disable Version Number Exposure
- Default Server Protection
- Block Illegal Host
- TLS Protocol Version Hardening
- HTTPS Force Redirect
- HSTS Configuration
- Common Security Response Headers
- CSP Basics
- Prevent Sensitive File Exposure
- Block Access to `.git`, `.env`, and Backup Files
- Trusted IP Boundary
- Prevent X-Forwarded-For Forgery
- Management Backend Access Control
- Limit Request Methods
- Limit Request Body Size
- Basic Slow Client Protection
- Nginx File and Directory Permissions
- Logs and Auditing
- Security Configuration Verification Commands
- Common Misconfiguration and Production Notes

This article is part of the Nginx Advanced SRE Capabilities Expansion series, Article 14.

This article's goal:

```text
I understand. Nginx Safe borders as an access layer

→ Configure Foundation TLS Security policy

→ Can configure common security response headers

→ It prevents sensitive paths and hidden files from being exposed.

→ Understand the truth. IP Trust borders

→ It can be avoided. X-Forwarded-For Forged.

→ Increased base access controls for managing backstage

→ Yes. curlI don't know.opensslI don't know.nginx -T Verify security configuration

→ It can form. Nginx Access layer security baseline
```

---

## II. Nginx Access Layer Security Boundary

Nginx often resides at the business entry point:

```text
User / Client

→ CDN / WAF / SLB

→ Nginx

→ Backend Application

→ Database / Cache / Message queue
```

Security hardening Nginx can do:

```text
HTTPS Access

TLS Protocol Version Control

HTTP Jump HTTPS

Security Response

Sensitive path intercept.

Hide File Protection

Manage backstage IP White list.

Base request limit flow

Request size limit

Real IP Clean

Illegal Host Intercept.

Basic access log audit
```

Nginx should not be mistaken as a complete security system.

It cannot fully replace:

```text
WAF

Business knowledge

Apply security verification

SQL Injecting protection.

XSS Content Filter

Authentication Code

Account Wind Control

Permission System

Zero Trust Access

Host security

Container security

Hole scan
```

One-sentence understanding:

```text
Nginx Security reinforcement is the first line of defence in the access layer, not the application of security.
```

---

## III. Security Hardening Overview

Nginx production security hardening can be divided into several categories:

```text
Exposure Control
→ Close Version Number
→ Default server Intercept.
→ Illegal Host Intercept.

Transfer level secure.
→ TLSv1.2 / TLSv1.3
→ HTTP Jump HTTPS
→ HSTS

Response head secure.
→ X-Frame-Options
→ X-Content-Type-Options
→ Referrer-Policy
→ Content-Security-Policy

Path secure.
→ Ban .git
→ Ban .env
→ Ban Backup Files
→ Ban Sensitive Directory

Real IP Clear.
→ set_real_ip_from
→ real_ip_header
→ real_ip_recursive
→ I don't trust anything. XFF

Access control
→ allow / deny
→ Basic Auth
→ Manage background limits

Request control.
→ client_max_body_size
→ client_header_timeout
→ client_body_timeout
→ limit_req
→ limit_conn

Authority and audit
→ Configure File Permissions
→ Certificate Private Key Permissions
→ Log audit
→ Configure Version Management
```

---

## IV. Disable Version Number Exposure

---

## Scenario 1: server_tokens off

By default, Nginx may expose version numbers in response headers or error pages.

Configuration:

```nginx
http {
    server_tokens off;
}
```

Effect:

```text
Hide Nginx Specific Version Number
```

Before enabling, you might see:

```text
Server: nginx/1.24.0
```

After disabling, you typically see:

```text
Server: nginx
```

Verification:

```bash
curl -I http://example.com
```

Or local verification:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Scenario 2: server_tokens Boundary

`server_tokens off` usually only hides version numbers.

It does not necessarily fully hide:

```text
Server: nginx
```

If you need to completely remove or rewrite the Server response header, you usually need:

```text
headers_more Third party module

OpenResty

Prefix WAF / CDN Rewrite

Cloud load equal response header rewrite
```

Production recommendation:

```text
General security baseline at least configured server_tokens off

Don't hide it completely. Server Head blindness introduces unnecessary modules
```

---

## V. Default Server Protection

---

## Scenario 3: Why Need Default Server

If users access unknown domains or directly access IPs:

```text
http://ServersIP

http://unknown.example.com
```

Nginx may hit the default server.

If the default server configuration is improper, it may lead to:

```text
Exposure Default Page

Misentry into operational sites

Access through normal domain names

Host Header Increased risk of attack

Large number of scanning requests in logs
```

---

## Scenario 4: HTTP Default Server Deny Access

Configuration:

```nginx
server {
    listen 80 default_server;
    server_name _;

    return 444;
}
```

Explanation:

```text
default_server
→ Default Virtual Host

server_name _
→ Place name

return 444
→ Nginx Close the connection directly, do not return the response
```

Verification:

```bash
curl -I http://127.0.0.1
```

Or:

```bash
curl -v -H "Host: unknown.example.com" http://127.0.0.1
```

---

## Scenario 5: HTTPS Default Server

HTTPS also recommends configuring a default server.

Example:

```nginx
server {
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate     /etc/nginx/certs/default/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/default/privkey.pem;

    return 444;
}
```

Note:

```text
HTTPS Handshake requires a certificate.

Even if it was. return 444, also require a loaded default certificate

Default HTTPS Certificates should not be randomly mixed with business private keys
```

---

## VI. Illegal Host Interception

---

## Scenario 6: Host Header Risk

Backend applications may use Host to generate:

```text
Callback Address

Reset Password Link

Jump Address

Absolutely. URL

Tenant identification

Reconciliation
```

If Host is not restricted, attackers may construct:

```bash
curl -H "Host: evil.com" http://example.com
```

Risk:

```text
Generate malicious links

Pollution Log

Around domain name bounds

Wrong business hit.

Impact on multi-tenant judgement
```

---

## Scenario 7: Allow Only Specified Domains

Business server explicitly specifies:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Combined with default server:

```nginx
server {
    listen 80 default_server;
    server_name _;

    return 444;
}
```

Unknown Hosts will be intercepted by the default server.

---

## Scenario 8: Add Host Validation Inside Server

You can also add Host validation:

```nginx
if ($host !~* ^(example\.com|www\.example\.com)$) {
    return 444;
}
```

Note:

```text
Nginx Medium if Use caution.

Priority adoption server_name + default_server Solve

Complex Host Policy proposal for use map
```

---

## Scenario 9: Use map to Validate Host

In `http` block:

```nginx
map $host $valid_host {
    default 0;
    example.com 1;
    www.example.com 1;
}
```

In server:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    if ($valid_host = 0) {
        return 444;
    }

    location / {
        proxy_pass http://app_backend;
    }
}
```

Explanation:

```text
map A lot. if It's better to focus on matching rules.
```

---

## VII. TLS Security Hardening

---

## Scenario 10: Disable Old Protocols

Recommended:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

Not recommended:

```nginx
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
```

Reason:

```text
SSLv2 / SSLv3 It's not safe.

TLSv1.0 / TLSv1.1 Not suitable for modern production baseline

TLSv1.2 / TLSv1.3 It's a common production choice.
```

---

## Scenario 11: HTTPS Basic Security Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Explanation:

```text
ssl_protocols
→ Limits TLS Protocol Version

ssl_ciphers
→ Limits TLSv1.2 Encryption packages with below

ssl_prefer_server_ciphers
→ Prefer service-end encryption package order
```

---

## Scenario 12: Check TLS Protocol

Check TLSv1.2:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2
```

Check TLSv1.3:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```

Check TLSv1.0:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1
```

If TLSv1.0 is disabled, the connection should fail or negotiation should fail.

---

## Scenario 13: Check Online Protocol and Cipher

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E "Protocol|Cipher"
```

---

## VIII. HTTP Force Redirect to HTTPS

---

## Scenario 14: HTTP 301 Redirect to HTTPS

Configuration:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    return 301 https://$host$request_uri;
}
```

Effect:

```text
Will HTTP Request for permanent jump to HTTPS
```

Verification:

```bash
curl -I http://example.com
```

Expected:

```text
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

---

## Scenario 15: HTTP Redirect Notes

Need to confirm:

```text
HTTPS Available

Certificate is clear.

Is all sub-domains supported? HTTPS

Backend correctly recognized X-Forwarded-Proto

CDN / SLB Is it true? HTTPS Configure

Existence HTTP Callback Interface
```

Do not blindly enforce full-site redirect before HTTPS is stable.

---

## IX. HSTS Configuration

---

## Scenario 16: What is HSTS

HSTS Response Header:

```text
Strict-Transport-Security
```

Effect:

```text
Tell the browser to be followed by mandatory use HTTPS Visit the site
```

Basic Configuration:

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

Explanation:

```text
max-age=31536000
→ Browser remembers a year

always
→ Even if not. 2xx / 3xx Add the response header as well
```

---

## Scenario 17: Complete HSTS Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Verification:

```bash
curl -I https://example.com
```

Check:

```text
Strict-Transport-Security: max-age=31536000
```

---

## Scenario 18: includeSubDomains and preload Caution

Stronger configuration:

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

Meaning:

```text
includeSubDomains
→ All sub-domains are mandatory. HTTPS

preload
→ Allows adding browser preview tables
```

Production Note:

```text
Only confirmation of long-term support for all sub-domains HTTPS , to use includeSubDomains

preload Once submitted, rollback costs are high.

Do not use randomly to test domain names or unstable domain names preload
```

---

## X. Common Security Response Headers

---

## Scenario 19: Basic Security Response Header Configuration

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Meaning:

```text
X-Frame-Options
→ Limit Pages By iframe Embedded to reduce the risk of click hijacking

X-Content-Type-Options
→ Ban Browser MIME Sniff.

Referrer-Policy
→ Control Referer Discovery range

Permissions-Policy
→ Limit browser privileges
```

---

## Scenario 20: X-Frame-Options

Configuration:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

Optional Values:

```text
DENY
→ It's completely forbidden. iframe Embedded

SAMEORIGIN
→ Allow only co-sources to embed
```

Note:

```text
If business requires third-party systems iframe Embedded, not directly set DENY

Need to combine business needs choices
```

---

## Scenario 21: X-Content-Type-Options

Configuration:

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

Effect:

```text
Disable browser guess based on content MIME Type

Reduce the risk that part of the script will be wrongly executed
```

---

## Scenario 22: Referrer-Policy

Configuration:

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Purpose:

```text
Control browser jumps Referer Information range
```

Common Strategies:

```text
no-referrer
→ Do Not Send Referer

same-origin
→ Send from Same Source

strict-origin-when-cross-origin
→ Trans-domain Only originI don't know.HTTPS Present. HTTP Do Not Send
```

---

## Scenario 23: Permissions-Policy

Configuration:

```nginx
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Purpose:

```text
Limit the browser ' s geographic location, microphones, cameras, etc.
```

Suitable for Ordinary Business Sites:

```text
No cameras.

No need for a microphone.

No geographic location.
```

If the business requires related capabilities, adjust according to business needs.

---

## Section Eleven: CSP Basics

---

## Scenario 24: What is CSP

CSP Response Header:

```text
Content-Security-Policy
```

Purpose:

```text
Limit which scripts, styles, pictures, fonts, interface sources you can load

Lower XSS Impact
```

Example:

```nginx
add_header Content-Security-Policy "default-src 'self'; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'; script-src 'self'; object-src 'none'; frame-ancestors 'self';" always;
```

---

## Scenario 25: CSP Configuration Should Be Cautious

CSP configuration that is too strict may lead to:

```text
Frontend JS Cannot Load

CSS Cannot Load

Pictures cannot be loaded

Third parties SDK Invalid

Validation code failed

Statistical script invalid

Map Component Invalid
```

Production Recommendations:

```text
Testing environment first.

Use a more conservative strategy first.

Use as necessary Content-Security-Policy-Report-Only Observation

Do not just copy the strict online. CSP To production
```

Report-Only Example:

```nginx
add_header Content-Security-Policy-Report-Only "default-src 'self';" always;
```

---

## Section Twelve: Protection of Sensitive Paths

---

## Scenario 26: Prohibit Access to Hidden Files

Configuration:

```nginx
location ~ /\. {
    deny all;
}
```

Can Intercept:

```text
.git

.env

.svn

.htaccess

.htpasswd
```

Verification:

```bash
curl -I http://example.com/.git/config
```

```bash
curl -I http://example.com/.env
```

Expected:

```text
403 Forbidden

or

404 Not Found
```

---

## Scenario 27: Prohibit Access to .git

More Explicit Configuration:

```nginx
location ~* /\.git {
    return 404;
}
```

Recommended to Return 404:

```text
Do not expose the directory
```

Verification:

```bash
curl -I http://example.com/.git/config
```

---

## Scenario 28: Prohibit Access to Environment Files

```nginx
location ~* /\.(env|ini|conf)$ {
    return 404;
}
```

Or:

```nginx
location ~* \.(env|ini|conf)$ {
    return 404;
}
```

Note:

```text
The rules need to combine business practice.

If business does provide .conf File download, avoid error.
```

More Fundamental Principle:

```text
Don't leave sensitive files. Web Root
```

---

## Scenario 29: Prohibit Access to Backup Files and Compressed Packages

Configuration:

```nginx
location ~* \.(bak|backup|old|orig|save|swp|sql|tar|tar.gz|tgz|zip|rar|7z)$ {
    return 404;
}
```

Can Prevent Exposure:

```text
Database Backup

Old Configuration

Compressors

Temporary documents

Editor swap Documentation
```

Note:

```text
If the business itself is downloaded zip We need to stop all of them. zip

It can only be Web Enable root directory or specific site
```

---

## Scenario 30: Prohibit Access to Sensitive Directories

Configuration:

```nginx
location ^~ /private/ {
    return 404;
}

location ^~ /config/ {
    return 404;
}

location ^~ /backup/ {
    return 404;
}
```

---

## Section Thirteen: autoindex Disabled

---

## Scenario 31: Disable Directory Browsing

Configuration:

```nginx
autoindex off;
```

Example:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    autoindex off;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Reason:

```text
Avoid Catalogue List Exposure

Avoid leaking backup files

Avoid leaking upload directory structure

Avoid being scanned to list files
```

Verification:

```bash
curl -I http://example.com/some-dir/
```

---

## Section Fourteen: Real IP Trust Boundary

---

## Scenario 32: Real IP Security Issues

Many security policies rely on client IP:

```text
allow / deny

limit_req

limit_conn

Audit log

Ban IP

Manage background white lists
```

If real IP configuration is incorrect, it may lead to:

```text
White list bypassed.

The flow limit is dead.

Ban Error Object

Log audit distortion

All users are considered SLB IP
```

---

## Scenario 33: Dangerous Configuration: Trust All XFF

Dangerous Configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Risks:

```text
Any client can fake it. X-Forwarded-For

Nginx It's a fake. IP When it's true IP

IP The white list could be bypassed.

The restricted flow could be bypassed.

The audit log is not credible
```

Do not configure this way.

---

## Scenario 34: Correct Configuration: Trust Only Trusted Proxies

Example:

```nginx
set_real_ip_from 10.0.0.10;
set_real_ip_from 10.0.0.11;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Explanation:

```text
Trust only SLB / CDN / WAF / Upstream Nginx Exports IP

Only from credible agents. Header Other Organiser IP
```

---

## Scenario 35: Source Server Limits to Allow Proxy Access Only

If the link is:

```text
User

→ CDN / WAF / SLB

→ Nginx Source
```

Recommendation:

```text
Security only allowed. CDN / WAF / SLB Return source IP Visits Nginx

Do not allow direct access to source sites by users of the Internet Nginx
```

Otherwise attackers may bypass upstream proxies and directly access the source server while forging Headers.

---

## Scenario 36: Verify Whether XFF Can Be Forged

Construct a forged Header:

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

Check Logs:

```bash
tail -n 20 /var/log/nginx/access.log
```

If public users can arbitrarily change the real IP in logs, it indicates a trust boundary issue.

---

## Section Fifteen: Access Control for Management Backend

---

## Scenario 37: Management Backend Uses IP Whitelist

Configuration:

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    proxy_pass http://admin_backend;
}
```

Note:

```text
If Nginx It's up ahead. SLB / CDN / WAF

We have to configure first. real_ip

Otherwise... allow / deny The judge is the agent. IP
```

---

## Scenario 38: Management Backend Basic Auth

Create a Password File:

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

Configuration:

```nginx
location /admin/ {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;

    proxy_pass http://admin_backend;
}
```

Verification:

```bash
curl -I http://example.com/admin/
```

With Authentication:

```bash
curl -I -u admin:Password http://example.com/admin/
```

---

## Scenario 39: Combined Protection for Management Backend

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;

    limit_req zone=admin_req burst=10 nodelay;
    limit_conn admin_conn 5;

    proxy_pass http://admin_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Explanation:

```text
IP White list.

+

Basic Auth

+

Limited flow

+

Connection Limit
```

Suitable for simple management backend protection.

Higher security requirements should use:

```text
VPN

Zero Trust Access

SSO

MFA

Fortress.

Formal system of authority
```

---

## Section Sixteen: Limit Request Methods

---

## Scenario 40: Why Limit Request Methods

Many sites only need:

```text
GET

POST

HEAD
```

If not needed:

```text
PUT

DELETE

TRACE

OPTIONS

PATCH
```

You can limit abnormal methods.

---

## Scenario 41: Reject TRACE

The TRACE method may bring security risks and is typically not needed in production.

Configuration:

```nginx
if ($request_method = TRACE) {
    return 405;
}
```

Verification:

```bash
curl -X TRACE -I http://example.com
```

---

## Scenario 42: Allow Only Common Methods

You can use:

```nginx
location / {
    limit_except GET POST HEAD {
        deny all;
    }

    proxy_pass http://app_backend;
}
```

Note:

```text
limit_except Interfaces suitable for static resources or clear methodological ranges

If API Use PUT / DELETE / PATCHDon't miss the intercept.
```

---

## Section Seventeen: Request Body and Slow Client Protection

---

## Scenario 43: Limit Request Body Size

Configuration:

```nginx
client_max_body_size 20m;
```

Upload interfaces can be individually increased:

```nginx
location /api/upload/ {
    client_max_body_size 500m;

    proxy_pass http://app_backend;
}
```

Security Significance:

```text
Avoid hyper-request consumption Nginx and backend resources

Reduce abnormal upload effects

Reduce risk of temporary cataloguing
```

Check 413:

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 44: Client Request Header Timeout

Configuration:

```nginx
client_header_timeout 10s;
```

Purpose:

```text
Limit client time to send header
```

Suitable for:

```text
Reduce slow-motion impact.

Avoiding unusual client long-term occupancy of connections
```

---

## Scenario 45: Client Request Body Timeout

Configuration:

```nginx
client_body_timeout 60s;
```

Upload interfaces can be individually increased:

```nginx
location /api/upload/ {
    client_body_timeout 300s;
    client_max_body_size 500m;

    proxy_pass http://app_backend;
}
```

---

## Scenario 46: Response Send Timeout

Configuration:

```nginx
send_timeout 60s;
```

Purpose:

```text
Limits Nginx Timeout between writing operations when sending response to client
```

---

## Section Eighteen: Nginx File and Directory Permissions

---

## Scenario 47: Configuration File Permissions

Check Configuration Directory:

```bash
ls -ld /etc/nginx
```

```bash
find /etc/nginx -type f -maxdepth 3 -ls
```

Production Recommendation:

```text
Ordinary users should not have write privileges

The configuration changes should be controlled through the process.

Avoid all direct editing of production profiles
```

---

## Scenario 48: Certificate Private Key Permissions

Check Certificate Permissions:

```bash
ls -lh /etc/nginx/certs/example.com/
```

Private Key Recommendation:

```bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

Certificate Recommendation:

```bash
chmod 644 /etc/nginx/certs/example.com/fullchain.pem
```

Note:

```text
Private key cannot be submitted Git

You can't. Web Root Directory

Could not close temporary folder: %s

Can not open message
```

---

## Scenario 49: Static Directory Permissions

Check Static Directory:

```bash
ls -ld /data/www/example
```

```bash
find /data/www/example -maxdepth 2 -type f -ls
```

Production Recommendation:

```text
Nginx Just read permissions

Static directory should not contain .git / .env / Private Key / Backup SQL

Separate upload directory from static release directory

Publish Users and Nginx Run user permission separation
```

---

## Section Nineteen: Security Logs and Auditing

---

## Scenario 50: Record Key Fields

access.log should at least include:

```text
time

remote_addr

realip_remote_addr

xff

host

method

uri

status

request_time

upstream_addr

user_agent

request_id
```

JSON Log Example:

```nginx
log_format security_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"realip_remote_addr":"$realip_remote_addr",'
    '"xff":"$http_x_forwarded_for",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$uri",'
    '"request_uri":"$request_uri",'
    '"status":$status,'
    '"request_time":"$request_time",'
    '"upstream_addr":"$upstream_addr",'
    '"user_agent":"$http_user_agent",'
    '"request_id":"$request_id"'
    '}';
```

---

## Scenario 51: Pay Attention to Abnormal Status Codes

Common Security-Related Status Codes:

```text
400
→ Unusual requests

401
→ Uncertified

403
→ Rejected

404
→ Scan No Path

405
→ Method is not allowed

413
→ The request is too big.

429
→ It's restricted.

499
→ Client Disconnect

5xx
→ Backend or gateway anomaly
```

Status Code Statistics:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

Statistics 403:

```bash
awk '$9 == 403 {print $1,$7,$12}' /var/log/nginx/access.log | tail -n 50
```

Statistics 404 Top URL:

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Statistics 429 Top IP:

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Section Twenty: Complete Security Hardening Configuration Example

---

## Scenario 52: HTTP + HTTPS + Secure Response Headers + Sensitive Path Protection

```nginx
server {
    listen 80 default_server;
    server_name _;

    return 444;
}

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

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;

    server_tokens off;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    client_max_body_size 20m;
    client_header_timeout 10s;
    client_body_timeout 60s;
    send_timeout 60s;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location ~ /\. {
        return 404;
    }

    location ~* \.(bak|backup|old|orig|save|swp|sql|tar|tar.gz|tgz|rar|7z)$ {
        return 404;
    }

    location ^~ /private/ {
        return 404;
    }

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

---

## Scenario 53: Security Baseline Example with real_ip

```nginx
http {
    server_tokens off;

    set_real_ip_from 10.0.0.10;
    set_real_ip_from 10.0.0.11;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

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
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;

        add_header Strict-Transport-Security "max-age=31536000" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        location /api/ {
            limit_req zone=api_req burst=20 nodelay;
            limit_conn api_conn 20;

            proxy_pass http://app_backend;

            proxy_http_version 1.1;
            proxy_set_header Connection "";

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }

        location ~ /\. {
            return 404;
        }
    }
}
```

---

## 21. Security Configuration Verification

---

## Scenario 54: Check Nginx Configuration

```bash
nginx -t
```

View full configuration:

```bash
nginx -T
```

View security headers:

```bash
nginx -T | grep -n "add_header" -A 10 -B 5
```

View TLS:

```bash
nginx -T | grep -n "ssl_protocols" -A 5 -B 5
```

View hidden file protection:

```bash
nginx -T | grep -n "location ~" -A 10
```

---

## Scenario 55: Verify Response Headers

```bash
curl -I https://example.com
```

Check specific response headers:

```bash
curl -I https://example.com | grep -Ei "Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Referrer-Policy|Permissions-Policy"
```

---

## Scenario 56: Verify TLS

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2
```

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1
```

View protocols and Cipher:

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E "Protocol|Cipher"
```

---

## Scenario 57: Verify HTTP Redirect to HTTPS

```bash
curl -I http://example.com
```

Expected:

```text
301

Location: https://example.com/
```

---

## Scenario 58: Verify Sensitive Paths

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

Expected:

```text
403

or

404
```

---

## Scenario 59: Verify Illegal Host

```bash
curl -v -H "Host: evil.com" http://127.0.0.1/
```

If default server returns 444, curl may see:

```text
Empty reply from server
```

---

## Scenario 60: Verify Real IP Forgery

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

Check logs:

```bash
tail -n 20 /var/log/nginx/access.log
```

Confirm:

```text
Was there a mistake in trusting a forgery? XFF
```

---

## Scenario 61: Error Configuration set_real_ip_from 0.0.0.0/0

Error:

```nginx
set_real_ip_from 0.0.0.0/0;
```

Risk:

```text
Any client can forge the truth. IP
```

Correct:

```nginx
set_real_ip_from SLB_IP;
set_real_ip_from CDN_Return source_IP_Paragraph;
```

---

## Scenario 62: Error Enabling HSTS includeSubDomains

Error scenario:

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

But actually:

```text
Some sub-domain names are not supported HTTPS

Some test domain name certificates are incomplete

Some historical systems can only HTTP
```

Consequences:

```text
Browser Force Access HTTPS

Sub-domain name operations not available

Rollback Hard
```

---

## Scenario 63: CSP Too Strict Causes Frontend Issues

Phenomenon:

```text
Page White Screen

JS Loading failed

CSS Loading failed

Third-party authentication code failed

Console. CSP blocked
```

Handling:

```text
First. Report-Only Observation

Progressive release by physical source of operations

Do not copy the strict template directly to production
```

---

## Scenario 64: Sensitive Files Still in Web Root

Even with interception rules, sensitive files in web root directory still pose risks.

Correct principle:

```text
Don't put sensitive files in it. Web Root Directory

Don't. .git To Static Directory

Don't. .env To Static Directory

Do not place database backups in accessible directory

Do not place private keys in the site directory
```

---

## 22. Production Notes

---

## 1. Security Hardening Should Be Tested Before Deployment

Especially:

```text
HSTS

CSP

Host Intercept.

Limitations on the method of request

Sensitive document interception

Manage background white lists

Real IP Configure
```

These configurations may misfire business operations.

---

## 2. Security Response Headers Cannot Replace Application Fixes

For example:

```text
CSP It can be lowered. XSS Impact

However, no substitute for front-end output encoding and back-end input verification

X-Frame-Options It lowers the click hijacking.

But there's no substitute for second confirmation of sensitive operations.
```

---

## 3. Real IP Is Premise for All IP Policies

Following capabilities depend on real IP:

```text
allow / deny

limit_req

limit_conn

Audit log

Ban IP

Manage background white lists
```

Before configuring these capabilities, must confirm:

```text
$remote_addr Is it a real client? IP
```

---

## 4. Default Server Is Critical

All unknown Host/IP access and scan traffic should be caught by default server.

Avoid unknown requests hitting business server by mistake.

---

## 5. Private Key Must Be Strictly Protected

Private key cannot:

```text
Submit Git

Put it in. Web Root Directory

Put in open object storage

Transfer explicitly via chat tool

Read by ordinary users
```

---

## 6. Security Policies Should Have Rollback Path

Backup before deployment:

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

Rollback:

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

## 7. Security Configuration Should Be Version Controlled

Recommend:

```text
Nginx Configure Entry Git

Before Change review

Changed records

Configure Autocheck

Support rollback

Multinodes Sync
```

---

## 24. Common Commands Summary

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "server_tokens"
```

```bash
nginx -T | grep -n "add_header" -A 10 -B 5
```

```bash
nginx -T | grep -n "ssl_protocols" -A 5 -B 5
```

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "return 444" -A 10 -B 5
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

## Response Header Check

```bash
curl -I https://example.com
```

```bash
curl -I https://example.com | grep -Ei "Strict-Transport-Security|X-Frame-Options|X-Content-Type-Options|Referrer-Policy|Permissions-Policy|Content-Security-Policy"
```

```bash
curl -I http://example.com
```

---

## TLS Check

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2
```

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1
```

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E "Protocol|Cipher"
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
ls -lh /etc/nginx/certs/example.com/
```

```bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

```bash
chmod 644 /etc/nginx/certs/example.com/fullchain.pem
```

---

## Sensitive Path Verification

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

```bash
curl -I https://example.com/private/
```

---

## Host Verification

```bash
curl -v -H "Host: evil.com" http://127.0.0.1/
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

---

## Real IP Verification

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

```bash
tail -n 20 /var/log/nginx/access.log
```

```bash
nginx -T | grep -n "real_ip"
```

---

## Log Analysis

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$9 == 403 {print $1,$7,$9}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## File Permissions Check

```bash
ls -ld /etc/nginx
```

```bash
find /etc/nginx -type f -maxdepth 3 -ls
```

```bash
ls -ld /data/www/example
```

```bash
find /data/www/example -maxdepth 2 -type f -ls
```

---

## Backup and Rollback

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
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

## 25. One-Sentence Summary

Nginx security hardening core is:

```text
Reduce exposure

Reinforce transmission

Limit access

Protect sensitive paths

Purge the truth. IP

Record security logs

Control Configuration Changes
```

Basic security items:

```nginx
server_tokens off;
ssl_protocols TLSv1.2 TLSv1.3;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Sensitive path protection:

```nginx
location ~ /\. {
    return 404;
}

location ~* \.(bak|backup|old|sql|tar|tar.gz|zip)$ {
    return 404;
}
```

Real IP security principle:

```text
Only trusted agents.

Don't. set_real_ip_from 0.0.0.0/0

Don't allow the public network to bypass the agency.

No unconditional trust in the backend X-Forwarded-For
```

Management backend protection recommendation:

```text
IP White list.

Basic Auth

Formal accreditation

VPN / Zero trust

Limited flow

Access Log
```

Production notes:

```text
HSTS includeSubDomains / preload Be careful.

CSP You have to test it before you go online.

Host Interception must be coordinated. default_server

Don't release sensitive files. Web Root Directory

Do Not Enter Private Key Git

We have to get it on the line. nginx -t

Multinodes configuration and certificates must be synchronized

All security changes must be rolled back.
```