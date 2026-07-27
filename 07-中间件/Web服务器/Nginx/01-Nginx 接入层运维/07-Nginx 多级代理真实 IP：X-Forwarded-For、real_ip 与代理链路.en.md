# 07-Nginx Multi-Level Proxy and Real IP: X-Forwarded-For, real_ip, and Proxy Chain

#Nginx #Real IP #XForwardedFor #XRealIP #real_ip #Multi-Level Proxy #Load Balancing #CDN #WAF #Access Layer #Operation and Maintenance #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operation and Maintenance/07-Nginx Multi-Level Proxy and Real IP: X-Forwarded-For, real_ip, and Proxy Chain.md

---

## I. Document Overview

This document outlines methods for obtaining, passing through, logging, and troubleshooting the real client IP in Nginx multi-level proxy scenarios.

Key points include:

- Why the IP seen by Nginx may not be the real user's IP
- The meaning of `$remote_addr`
- The meaning of `X-Forwarded-For`
- The meaning of `X-Real-IP`
- The function of `$proxy_add_x_forwarded_for`
- IP changes in multi-level proxy chains
- `real_ip_header`
- `set_real_ip_from`
- `real_ip_recursive`
- How to trust only trusted proxies
- How to prevent clients from forging real IPs
- How to log the real IP in access.log
- How to pass the real IP to the backend
- The relationship between `allow`/`deny` and real IP
- CDN/SLB/WAF/Nginx multi-layer proxy scenarios
- Common misconfigurations of real IP
- Commands and verification processes for troubleshooting real IP issues

This document is part of the Nginx Access Layer Operation and Maintenance series, Article 07.

Objectives:

```text
Understand why the real IP may be lost in multi-level proxies

→ Comprehend the X-Forwarded-For proxy chain

→ Configure the real_ip module correctly

→ Distinguish between remote_addr, realip_remote_addr, and X-Forwarded-For

→ Trust only trusted proxies securely

→ Log the real IP in access.log

→ Pass the real IP to backend services

→ Troubleshoot issues such as incorrect real IPs, all load balancing IPs, or forged IPs
```

---

## II. Why Real IP is Important

The real client IP is commonly used for:

```text
Access log analysis

Security audits

Login audits

Risk control assessments

IP blocklists and allowlists

Throttling

Blocking malicious sources

Troubleshooting user issues

Counting access sources

Locating crawlers or scanners

Tracing attack origins
```

Incorrect retrieval of the real IP can lead to:

```text
Access logs containing only SLB IPs

Backend logs containing only Nginx IPs

Invalid throttling rules

Misjudgment in blocklists and allowlists

Unreliable audit logs

Inaccurate security blocking

Inability to trace attack sources

Incorrect analysis of source IPs in log platforms
```

In short:

```text
The real IP is a fundamental field for access layer troubleshooting, security, and auditing.
```

---

## III. Common Proxy Chains

---

## Scenario 1: No Proxy

Chain:

```text
Client 1.1.1.1

→ Nginx 10.0.0.20
```

Nginx sees:

```text
$remote_addr = 1.1.1.1
```

At this point, `$remote_addr` is the real client IP.

---

## Scenario 2: SLB in Front of Nginx

Chain:

```text
Client 1.1.1.1

→ SLB 2.2.2.2

→ Nginx 10.0.0.20
```

Nginx directly sees the SLB IP as the source:

```text
$remote_addr = 2.2.2.2
```

If the SLB passes through `X-Forwarded-For`, Nginx may receive:

```text
X-Forwarded-For: 1.1.1.1
```

---

## Scenario 3: CDN + WAF + SLB + Nginx

Chain:

```text
Client 1.1.1.1

→ CDN 2.2.2.2

→ WAF 3.3.3.3

→ SLB 4.4.4.4

→ Nginx 10.0.0.20

→ Backend Application 10.0.0.21:8080
```

Possible `X-Forwarded-For`:

```text
X-Forwarded-For: 1.1.1.1, 2.2.2.2, 3.3.3.3
```

Nginx directly sees:

```proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;```nginx
log_format realip_main '$remote_addr [$realip_remote_addr] - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       '"$http_x_forwarded_for";
```

Usage:

```nginx
access_log /var/log/nginx/example.access.log realip_main;
```

---

## XI. Passing the Real IP Address to the Backend

---

## Scenario 23: Passing the Real IP After Processing

If the following configuration has been set:

```nginx
set_real_ip_from 10.0.0.10;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Then `$remote_addr` has already been replaced with the real client IP address.

In this case, it is recommended to pass through the following headers:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Complete configuration:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

The backend can then read:

```text
X-Real-IP

X-Forwarded-For
```

---

## Scenario 24: Recording the Real IP at the Backend

Common order of reading at the backend application:

```text
First, read the X-Real-IP passed by a trusted proxy.

Or, parse X-Forwarded-For.

Otherwise, use the source IP of the connection.
```

However, it is important to note that:

```text
The backend should not unconditionally trust the X-Forwarded-For sent by the client.

It is best to only trust requests coming from Nginx.

And let Nginx handle the cleaning of the real IP address uniformly.
```

---

## XII. Using allow / deny with the Real IP Address

---

## Scenario 25: Using allow / deny when real_ip is not configured

If the connection flow is:

```text
Client

→ SLB

→ Nginx
```

And Nginx does not have real_ip configured:

```text
$remote_addr = SLB IP
```

In this case, commands like:

```nginx
allow 1.1.1.1;
deny all;
```

may not work effectively, because Nginx will not check the user's IP address but instead the SLB IP.

---

## Scenario 26: Using allow / deny after configuring real_ip

After real_ip is configured:

```text
$remote_addr = Real client IP address
```

Access control can be implemented as follows:

```nginx
location /admin/ {
    allow 1.1.1.1;
    deny all;

    proxy_pass http://admin_backend;
}
```

In this way, access will be controlled based on the real client IP address.

---

## Scenario 27: Example of an Allowlist for the Management Backend

```nginx
set_real_ip_from 10.0.0.10;
real_ip_header X-Forwarded-For;
real_ip_recursive on;

server {
    listen 80;
    server_name admin.example.com;

    location / {
        allow 1.1.1.1;
        allow 2.2.2.2;
        deny all;

        proxy_pass http://127.0.0.1:9090;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Production note:

```text
It is essential to ensure that set_real_ip_from only trusts trusted proxies.

Otherwise, attackers could forge X-Forwarded-For to bypass the allowlist.
```

---

## Scenario 29: The Correct Approach: Only Trusting Upstream Proxies

It is recommended to configure:

```nginx
set_real_ip_from 10.0.0.10;
set_real_ip_from 10.0.0.11;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

At the same time, restrict access through security groups or firewalls:

```text
Nginx should only allow access from SLB/WAF/CDN origin-pull IPs.

Public network users should not be allowed to directly access the Nginx origin server.
```

---

## Scenario 30: Cleaning X-Forwarded-For

If Nginx is the outermost entry point and there are no trusted proxies in front, it isRemove the 🔤 symbols and translate the following text:

```markdown
Do not use the X-Forwarded-For header transmitted by the client.
Directly use $remote_addr to generate a trusted source IP address.
```

---

## Scenario 33: CDN Origin-Pull with Nginx

Chain of connections:

```text
Client → CDN → Nginx
```

Assume that the CDN sends:

```text
X-Forwarded-For
or
CF-Connecting-IP
```

If using X-Forwarded-For:

```nginx
set_real_ip_from CDN_origin-pull_IP_range;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

If using CF-Connecting-IP:

```nginx
set_real_ip_from CDN_origin-pull_IP_range;
real_ip_header CF-Connecting-IP;
```

Note:

```text
The CDN_origin-pull_IP_range should be based on the manufacturer's documentation.
The CDN IP range may change and needs to be updated regularly.
The origin server should restrict access only to the CDN-origin-pull IP addresses.
```

---

## Section 15: Verifying Real IP Configuration

---

## Scenario 34: Checking if Nginx Supports the realip Module

View the compilation parameters:

```bash
nginx -V 2>&1 | grep http_realip_module
```

If it includes:

```text
--with-http_realip_module
```

it means that the realip module is supported.

---

## Scenario 35: Viewing Current realip Configuration

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
nginx -T | grep -n "real_ip_recursive"
```

View the log format:

```bash
nginx -T | grep -n "log_format" -A 20
```

---

## Scenario 36: Constructing an X-Forwarded-For Test

Local test:

```bash
curl -H "X-Forwarded-For: 1.1.1.1" -H "Host: example.com" http://127.0.0.1/
```

View the logs:

```bash
tail -n 20 /var/log/nginx/example.access.log
```

Note:

```text
If the request comes from 127.0.0.1 and it is not included in set_real_ip_from, Nginx should not trust this forged XFF.
```

If the test environment requires verification, you can temporarily trust the local IP:

```nginx
set_real_ip_from 127.0.0.1;
```

Make sure to remove this temporary setting from the production configuration after the test.

---

## Scenario 37: Simulating a Trusted Proxy Test

Temporary test configuration:

```nginx
set_real_ip_from 127.0.0.1;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Request:

```bash
curl -H "X-Forwarded-For: 1.1.1.1, 2.2.2.2" -H "Host: example.com" http://127.0.0.1/
```

View the logs:

```bash
tail -n 20 /var/log/nginx/example.access.log
```

Expected result:

```text
$remote_addr may become 2.2.2.2 or 1.1.1.1, depending on whether set_real_ip_from trusts the proxy IP in the chain.
```

In production, you should verify this using actual proxy IP ranges.

---

## Scenario 38: Making the Backend Display Request Headers

You can create a temporary debug interface on the backend:

```text
/debug/headers
```

Request it through Nginx:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/debug/headers
```

Pay attention to the following headers received by the backend:

```text
X-Real-IP
X-Forwarded-For
X-Forwarded-Proto
Host
```

---

## Section 16: Troubleshooting Common Real IP Issues

---

## Scenario 39: access.log Contains Only SLB IP

Issue:

```text
In the access.log, $remote_addr is always 10.0.0.10.
```

Common causes:

```text
real_ip configuration is not set up.
set_real_ip_from does not include the SLB IP.
SLB does not send X-Forwarded-For.
The real_ip_header is configured incorrectly.
The request did not come from the expected SLB.
Configuration changes were not applied after reloading Nginx.
```

Troubleshooting steps:

```bash
```bash
grep '"xff":""' /var/log/nginx/example.access.log | head
```

---

## Scenario 47: Checking a Specific IP Request

```bash
grep "1.1.1.1" /var/log/nginx/example.access.log | tail -n 50
```

For more precise matching based on the first column:

```bash
awk '$1 == "1.1.1.1" {print $0}' /var/log/nginx/example.access.log | tail -n 50
```

---

## XVIII. Production Considerations

---

## 1. Do Not Trust X-Forwarded-For Unconditionally

`X-Forwarded-For` is a request header that can be forged by clients.

It must be used in conjunction with:

```text
set_real_ip_from

security groups

firewalls

trusted proxy links
```

---

## 2. Avoid Configuring set_real_ip_from to 0.0.0.0/0

Dangerous configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
```

Risks:

```text
Anyone can forge the real IP address.

Access control may be bypassed.

Log auditing will become unreliable.
```

---

## 3. The Origin Server Should Allow Access Only Through Trusted Proxies

If Nginx is behind a CDN, WAF, or SLB, access should be restricted to:

```text
Only allow CDN/WAF/SLB origin-pull IP addresses to access Nginx.

Prevent public users from bypassing proxies and connecting directly to the origin server.
```

Otherwise, the trust chain for real IP addresses will be compromised.

---

## 4. Change Log Format After Setting Real IP Addresses

It is recommended to record the following:

```text
$remote_addr

$realip_remote_addr

$http_x_forwarded_for
```

This way, during troubleshooting, you can see:

```text
The ultimately identified real IP address,

The proxy IP of the previous hop,

And the complete proxy chain.
```

---

## 5. The Backend Should Also Use a Unified Standard for Reading Real IP Addresses

Do not let different services on the backend read them in different ways:

```text
Some read X-Real-IP,

some read X-Forwarded-For,

some read the connection source IP address,

and some trust the leftmost IP address.
```

It is recommended to unify the process:

```text
Let Nginx clean up the real IP addresses.

The backend should only trust X-Real-IP/X-Forwarded-For sent by Nginx.

Backend frameworks should enable trusted proxy configuration.
```

---

## 6. Always Verify Whether remote_addr Is a Real IP Address Before Using an IP Allowlist

Before adding an IP to the allowlist, check the logs first:

```bash
tail -n 20 /var/log/nginx/example.access.log
```

Confirm whether the first column contains the real user's IP address.

---

## 7. Maintain the CDN Vendor's IP Range

If you trust the CDN origin-pull IP addresses:

```text
The CDN IP range may change.

It needs to be updated regularly.

After any changes, reload Nginx.

Otherwise, real IP identification may become inaccurate.
```

---

## XIX. Summary of Commonly Used Commands in This Chapter

---

## Check the realip Module Configuration

```bash
nginx -V 2>&1 | grep http_realip_module
```

---

## View real_ip Configuration Details

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
nginx -T | grep -n "real_ip_recursive"
```

```bash
nginx -T | grep -n "log_format" -A 20
```

```bash
nginx -T | grep -n "proxy_set_header" -A 10
```

---

## Perform Configuration Checks and Reload Nginx

```bash
nginx -t
```

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

---

## Create XFF Test Cases

```bash
curl -H "X-Forwarded-For: 1.1.1.1" -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -H "X-Forwarded-For: 1.1.1.1, 2.2.2.2" -H "Host: example.com" http://127.0.0.1/
```

---

## View Logs

```bash
tail -n 20 /var/log/nginx/example.access.log
```

```bash
tail -f /var/log/nginx/example.access.log
```

```bash