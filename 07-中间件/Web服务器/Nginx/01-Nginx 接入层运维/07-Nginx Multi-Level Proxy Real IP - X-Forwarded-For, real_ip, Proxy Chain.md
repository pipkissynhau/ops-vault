# 07-Nginx Multi-Level Proxy Real IP: X-Forwarded-For, real_ip, and Proxy Chain

#Nginx #RealIp #XForwardedFor #XRealIP #real_ip #MultipleAgent #LoadBalance #CDN #WAF #AccessLayer #Transport #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operations/07-Nginx Multi-Level Proxy Real IP: X-Forwarded-For, real_ip, and Proxy Chain.md

---

## I. Document Overview

This document organizes methods for obtaining, passing through, recording, and troubleshooting real client IP in Nginx multi-level proxy scenarios.

Key focuses include:

- Why Nginx sees an IP that may not be the real user IP
- Meaning of `$remote_addr`
- Meaning of `X-Forwarded-For`
- Meaning of `X-Real-IP`
- Purpose of `$proxy_add_x_forwarded_for`
- IP changes in multi-level proxy chains
- Meaning of `real_ip_header`
- Meaning of `set_real_ip_from`
- Meaning of `real_ip_recursive`
- How to trust only reliable proxies
- How to prevent clients from forging real IPs
- How to record real IPs in access.log
- How to pass through real IPs to backend
- Relationship between `allow` / `deny` and real IPs
- CDN / SLB / WAF / Nginx multi-layer proxy scenarios
- Common real IP configuration errors
- Real IP troubleshooting commands and verification process

This is the 07th article in the Nginx Access Layer Operations series.

This article's goal:

```text
I can understand the reality of multiple agents. IP Why is it missing?

→ I can read it. X-Forwarded-For Agency link

→ Configure correctly real_ip Module

→ It makes a difference. remote_addrI don't know.realip_remote_addrI don't know.X-Forwarded-For

→ It's safe to trust credible agents.

→ Can make it real. IP Record access.log

→ Can make it real. IP Passover to back-end services

→ It's real. IP Incorrect. Load balance. IPQuestion of forgery
```

---

## II. Why Real IP Matters

Real client IPs are commonly used for:

```text
Access log analysis

Security audit

Login audit

Wind control.

IP Black & White List

Limited flow

Ban the source.

Check user questions

Statistical access sources

Positioning reptiles or scanners

Trace the source of the attack.
```

If real IP retrieval is incorrect, it may lead to:

```text
access.log It's all over. SLB IP

It's all in the back log. Nginx IP

The flow limit is not working.

Black & White List Miscalculation

The audit log is not credible

Securely block the wrong object

It's impossible to determine the source of the attack.

Source in log platform IP Parse error
```

One-sentence understanding:

```text
Real IP It is a base field for access layers, security and auditing.
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

At this time, `$remote_addr` is the real client IP.

---

## Scenario 2: SLB in Front of Nginx

Chain:

```text
Client 1.1.1.1

→ SLB 2.2.2.2

→ Nginx 10.0.0.20
```

Nginx directly sees the source as SLB:

```text
$remote_addr = 2.2.2.2
```

If SLB passes through `X-Forwarded-For`, Nginx may receive:

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

```text
$remote_addr = 4.4.4.4
```

Note:

```text
$remote_addr It's a direct connection. Nginx Last Jump IP

Not necessarily a real user. IP
```

---

## IV. Core Variable Explanations

---

## Scenario 4: $remote_addr

`$remote_addr` represents:

```text
Connect directly to the current Nginx Peer IP
```

If user connects directly to Nginx:

```text
$remote_addr = User Real IP
```

If there is SLB / CDN / WAF in front:

```text
$remote_addr = Previous Agent IP
```

Check log entries for `$remote_addr`:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';
```

---

## Scenario 5: $http_x_forwarded_for

`$http_x_forwarded_for` represents:

```text
In Client Request Header X-Forwarded-For Original content
```

Example request header:

```text
X-Forwarded-For: 1.1.1.1, 2.2.2.2
```

In Nginx:

```text
$http_x_forwarded_for = 1.1.1.1, 2.2.2.2
```

Note:

```text
This value comes from the request header.

Clients can forge.

No unconditional trust.
```

---

## Scenario 6: $proxy_add_x_forwarded_for

`$proxy_add_x_forwarded_for` represents:

```text
In original X-Forwarded-For Add the current after Nginx Yeah. remote_addr
```

Equivalent understanding:

```text
$http_x_forwarded_for, $remote_addr
```

Common configuration:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

If original request header is:

```text
X-Forwarded-For: 1.1.1.1
```

Current Nginx sees:

```text
$remote_addr = 2.2.2.2
```

After forwarding to backend:

```text
X-Forwarded-For: 1.1.1.1, 2.2.2.2
```

---

## Scenario 7: $realip_remote_addr

After configuring `real_ip`, `$remote_addr` may be replaced with real IP.

`$realip_remote_addr` is used to save:

```text
By real_ip Original previous jump before module replacement IP
```

Example:

```text
Original Previous Jump SLB IP:4.4.4.4

real_ip True after Replace IP:1.1.1.1
```

Then it could be:

```text
$remote_addr = 1.1.1.1

$realip_remote_addr = 4.4.4.4
```

This variable is suitable for log writing, allowing simultaneous retention of:

```text
Real Client IP

Previous Jump Agent IP
```

---

## V. X-Forwarded-For Basics

---

## Scenario 8: What is X-Forwarded-For

`X-Forwarded-For` is a common proxy chain header.

Function:

```text
Recording request before proxy client IP And middle agent. IP
```

Format is usually:

```text
X-Forwarded-For: client, proxy1, proxy2
```

Example:

```text
X-Forwarded-For: 1.1.1.1, 2.2.2.2, 3.3.3.3
```

General understanding:

```text
Leftmost
→ First client IP, usually a real user IP

Centre
→ Intermediate agent IP

Rightest
→ Next to the current agent
```

But- On the assumption that...:

```text
Agent links are credible.

It wasn't forged by the client.

Every layer of agent correctly adds X-Forwarded-For
```

---

## Scenario 9: X-Forwarded-For Can Be Forged

Clients can directly construct request headers:

```bash
curl -H "X-Forwarded-For: 9.9.9.9" http://example.com
```

If Nginx or backend unconditionally trusts this value, it may mistakenly take the real IP as:

```text
9.9.9.9
```

Risk:

```text
Around IP White list.

Go around the limit.

Falsification of audit sources

Pollution Log

Security Ban Expire
```

Production principle:

```text
Only those who trust a credible agent. X-Forwarded-For

Don't trust a public network client to get in. X-Forwarded-For
```

---

## VI. X-Real-IP Basics

---

## Scenario 10: What is X-Real-IP

`X-Real-IP` is typically used to pass a client IP.

Common configuration:

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

If Nginx has already obtained the real client IP through real_ip:

```text
$remote_addr = 1.1.1.1
```

Then backend receives:

```text
X-Real-IP: 1.1.1.1
```

---

## Scenario 11: Difference Between X-Real-IP and X-Forwarded-For

```text
X-Real-IP
→ Usually only one. IP

X-Forwarded-For
→ It could be a chain of agents. IP
```

In production, both are usually passed:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

---

## VII. real_ip Module Basics

---

## Scenario 12: What Does real_ip Module Solve

When Nginx has a trusted proxy in front, it wants Nginx to replace:

```text
Previous Jump Agent IP
```

With:

```text
Real Client IP
```

At this time, use real_ip related configurations:

```nginx
set_real_ip_from
real_ip_header
real_ip_recursive
```

---

## Scenario 13: real_ip_header

Configuration:

```nginx
real_ip_header X-Forwarded-For;
```

Function:

```text
Tell Nginx From what? Header Median Real Client IP
```

Common values:

```nginx
real_ip_header X-Forwarded-For;
```

It could also be:

```nginx
real_ip_header X-Real-IP;
```

Some CDN or cloud vendors may use proprietary headers, such as:

```text
CF-Connecting-IP

X-Client-IP

X-Forwarded-For

X-Real-IP
```

In production, it should be based on actual upstream proxy documentation and packet capture results.

---

## Scenario 14: set_real_ip_from

Configuration:

```nginx
set_real_ip_from 10.0.0.0/8;
```

Function:

```text
Declaration of sources IP It's a credible agent.
```

Only when requests come from these trusted proxies will Nginx use the IP in `real_ip_header` to replace `$remote_addr`.

Example:

```nginx
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 192.168.0.0/16;
set_real_ip_from 172.16.0.0/12;
```

Production note:

```text
Don't write anything. 0.0.0.0/0

Don't trust all sources.

Write Only SLBI don't know.CDNI don't know.WAFupstream Nginx The real exit. IP Paragraph
```

---

## Scenario 15: real_ip_recursive

Configuration: /think

```nginx
real_ip_recursive on;
```

Purpose:

```text
Recursive Find X-Forwarded-For First untrustworthy agent IP
```

Simple Understanding:

```text
Look right to left. X-Forwarded-For

Skip Credible Agent IP

Find the first untrustworthy agent. IP As a real client IP
```

Suitable for multi-level trusted proxy chains.

---

## VIII. Basic Configuration Example for Real IP

---

## Scenario 16: SLB in Front of Nginx

Link:

```text
Client

→ SLB 10.0.0.10

→ Nginx 10.0.0.20

→ Backend
```

Assume SLB will pass:

```text
X-Forwarded-For: Client Real IP
```

Nginx Configuration:

```nginx
http {
    set_real_ip_from 10.0.0.10;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://127.0.0.1:8080;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

Explanation:

```text
Only from 10.0.0.10 The request is considered a credible representation request

Nginx From X-Forwarded-For Other Organiser IP

After extraction $remote_addr Turn into a real client IP
```

---

## Scenario 17: All Internal SLB Network Segments are Trusted

If SLB export may be a network segment:

```nginx
set_real_ip_from 10.0.0.0/24;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Suitable for:

```text
Multiple SLB Nodes

Cloud Load Balance Multiple Exports IP

Multiple upstream agents
```

Note:

```text
We need to make sure that this section is used only by credible agents.

Do not add normal client source links to a credible list
```

---

## IX. Multi-Level Proxy real_ip Configuration

---

## Scenario 18: CDN + WAF + SLB + Nginx

Link:

```text
Client 1.1.1.1

→ CDN 2.2.2.2

→ WAF 3.3.3.3

→ SLB 10.0.0.10

→ Nginx 10.0.0.20
```

Nginx Receives:

```text
$remote_addr = 10.0.0.10

X-Forwarded-For: 1.1.1.1, 2.2.2.2, 3.3.3.3
```

Trusted Proxies:

```text
10.0.0.10

3.3.3.3

2.2.2.2
```

Configuration:

```nginx
set_real_ip_from 10.0.0.10;
set_real_ip_from 3.3.3.3;
set_real_ip_from 2.2.2.2;

real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Result:

```text
$remote_addr = 1.1.1.1
```

---

## Scenario 19: Only Trust Last-Hop Proxy

If only trust SLB:

```nginx
set_real_ip_from 10.0.0.10;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

This requires:

```text
SLB It's coming. X-Forwarded-For It's credible.

SLB Already processed and covered or added correctly XFF

Internet users can't just go around. SLB Visits Nginx
```

If public users can bypass SLB and connect directly to Nginx, there is a risk of IP forgery.

Production Recommendation:

```text
Security team or firewall restrictions Nginx Only allowed SLB Visits

Do not allow Internet clients to access the backend directly Nginx
```

---

## X. Record Real IP in access.log

---

## Scenario 20: Default access.log Issues

Default log format is common:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';
```

If not configured real_ip, `$remote_addr` may be:

```text
SLB IP

CDN IP

WAF IP

Upstream Nginx IP
```

Unfavorable for real source analysis.

---

## Scenario 21: Recommended Real IP Log Format

Recommended to record:

```text
$remote_addr

$realip_remote_addr

$http_x_forwarded_for

$proxy_add_x_forwarded_for
```

Example:

```nginx
log_format realip_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"realip_remote_addr":"$realip_remote_addr",'
    '"xff":"$http_x_forwarded_for",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$request_uri",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"request_time":$request_time,'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_response_time":"$upstream_response_time",'
    '"user_agent":"$http_user_agent"'
    '}';
```

Use:

```nginx
access_log /var/log/nginx/example.access.log realip_json;
```

Explanation:

```text
remote_addr
→ real_ip Processed Client IP

realip_remote_addr
→ Replace previous jump agent IP

xff
→ Original X-Forwarded-For Chain
```

---

## Scenario 22: Plain Text Log Format

If not using JSON, you can also:

```nginx
log_format realip_main '$remote_addr [$realip_remote_addr] - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       '"$http_x_forwarded_for"';
```

Use:

```nginx
access_log /var/log/nginx/example.access.log realip_main;
```

---

## XI. Pass Real IP to Backend

---

## Scenario 23: Process real_ip Then Pass Through

If already configured:

```nginx
set_real_ip_from 10.0.0.10;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Then `$remote_addr` has already been replaced with real client IP.

At this point, recommend passing through:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Complete Configuration:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Backend can read:

```text
X-Real-IP

X-Forwarded-For
```

---

## Scenario 24: Backend Records Real IP

Common reading order for backend applications:

```text
Read as a matter of priority to a trusted agent. X-Real-IP

Or parsing X-Forwarded-For

Use connection source otherwise IP
```

But note:

```text
The back end can't trust the client. X-Forwarded-For

It's best to trust it. Nginx Request

And by Nginx Unanimously wash the truth. IP
```

---

## XII. allow / deny with Real IP

---

## Scenario 25: allow / deny Without real_ip Configuration

If the link is:

```text
Client

→ SLB

→ Nginx
```

Nginx without real_ip configuration:

```text
$remote_addr = SLB IP
```

At this time:

```nginx
allow 1.1.1.1;
deny all;
```

May not take effect, because Nginx judges not the user IP but the SLB IP.

---

## Scenario 26: allow / deny After real_ip Configuration

After real_ip configuration:

```text
$remote_addr = Real Client IP
```

At this time, access control:

```nginx
location /admin/ {
    allow 1.1.1.1;
    deny all;

    proxy_pass http://admin_backend;
}
```

Will be judged based on real client IP.

---

## Scenario 27: Management Backend Whitelist Example

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

Production Note:

```text
It has to be done. set_real_ip_from Only trusted agents.

Otherwise the assailant could have forged it. X-Forwarded-For By the white list.
```

---

## XIII. Prevent X-Forwarded-For Forgery

---

## Scenario 28: Error Configuration: Trust All Sources

Dangerous Configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Risk:

```text
Any client can fake it. X-Forwarded-For

Nginx It's a fake. IP When it's true IP

White lists and restricted flow could be bypassed.

Audit log distortion
```

Do not configure this way.

---

## Scenario 29: Correct Approach: Trust Only Upstream Proxies

Recommended:

```nginx
set_real_ip_from 10.0.0.10;
set_real_ip_from 10.0.0.11;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

At the same time, restrict in security group or firewall:

```text
Nginx Only allowed SLB / WAF / CDN Return source IP Visits

Internet users are not allowed direct access Nginx Source
```

---

## Scenario 30: Clean X-Forwarded-For

If Nginx is the outermost entry point, and there are no trusted upstream proxies, it is recommended not to trust the client's own XFF.

You can directly override and pass to backend:

```nginx
proxy_set_header X-Forwarded-For $remote_addr;
```

Instead of appending:

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Applicable Scenario:

```text
Nginx It's the first entrance to the network.

I don't want the client to fake it. XFF Enter Backend

by Nginx Regeneration credible XFF
```

---

## XIV. Common Complete Configuration Examples

---

## Scenario 31: Nginx Behind SLB

```nginx
http {
    set_real_ip_from 10.0.0.10;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    log_format realip_main '$remote_addr [$realip_remote_addr] - $remote_user [$time_local] '
                           '"$request" $status $body_bytes_sent '
                           '"$http_referer" "$http_user_agent" '
                           '"$http_x_forwarded_for"';

    upstream app_backend {
        server 10.0.0.21:8080;
        server 10.0.0.22:8080;
    }

    server {
        listen 80;
        server_name example.com;

        access_log /var/log/nginx/example.access.log realip_main;
        error_log  /var/log/nginx/example.error.log warn;

        location / {
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

## Scenario 32: Nginx as Public First Entry

If Nginx faces the public directly without trusted upstream proxies:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Explanation:

```text
Not from client X-Forwarded-For

Directly $remote_addr Generate credible sources IP
```

---

## Scenario 33: CDN Source to Nginx

Link:

```text
Client

→ CDN

→ Nginx
```

Assume CDN passes:

```text
X-Forwarded-For

or

CF-Connecting-IP
```

If using X-Forwarded-For:

```nginx
set_real_ip_from CDNReturn sourceIPParagraph;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

If using CF-Connecting-IP:

```nginx
set_real_ip_from CDNReturn sourceIPParagraph;
real_ip_header CF-Connecting-IP;
```

Note:

```text
CDN Return source IP This section is based on the manufacturer's documents.

CDN IP Possible changes in paragraph, need to be maintained and updated

Source station should be restricted to allow only CDN Return source IP Visits
```

---

## XV. Verify Real IP Configuration

---

## Scenario 34: Check if Nginx Supports realip Module

Check compilation parameters:

```bash
nginx -V 2>&1 | grep http_realip_module
```

If contains:

```text
--with-http_realip_module
```

Indicates support for realip module.

---

## Scenario 35: Check Current real_ip Configuration

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
nginx -T | grep -n "real_ip_recursive"
```

Check log format:

```bash
nginx -T | grep -n "log_format" -A 20
```

---

## Scenario 36: Construct X-Forwarded-For for Testing

Local testing:

```bash
curl -H "X-Forwarded-For: 1.1.1.1" -H "Host: example.com" http://127.0.0.1/
```

Check logs:

```bash
tail -n 20 /var/log/nginx/example.access.log
```

Note:

```text
If the source of the request 127.0.0.1 It's not here. set_real_ip_from Medium

Nginx You shouldn't trust this fake. XFF
```

If testing environment needs verification, temporarily trust local machine:

```nginx
set_real_ip_from 127.0.0.1;
```

After testing, do not leave unnecessary trust sources in production configuration.

---

## Scenario 37: Simulate Trusted Proxy Testing

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

Check logs:

```bash
tail -n 20 /var/log/nginx/example.access.log
```

Expected:

```text
$remote_addr Could be. 2.2.2.2 or 1.1.1.1

It depends. set_real_ip_from Whether or not to trust an agent in a link IP
```

In production, verify with real proxy IP segments.

---

## Scenario 38: Let Backend Display Request Headers

Can temporarily provide a debugging interface for backend:

```text
/debug/headers
```

Request via Nginx:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/debug/headers
```

Focus on what backend receives:

```text
X-Real-IP

X-Forwarded-For

X-Forwarded-Proto

Host
```

---

## XVI. Common Real IP Troubleshooting

---

## Scenario 39: access.log Full of SLB IP

Phenomenon:

```text
access.log Medium $remote_addr Both. 10.0.0.10
```

Common Causes:

```text
Not configured real_ip

set_real_ip_from It's not included. SLB IP

SLB No. No. X-Forwarded-For

real_ip_header Wrong match.

Requests are not expected. SLB Come here.

The configuration has changed but not yet. reload
```

Troubleshoot:

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
tail -n 100 /var/log/nginx/access.log
```

Packet capture to check request headers:

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Forwarded-For"
```

---

## Scenario 40: Backend Logs Full of Nginx IP

Phenomenon:

```text
Client to see backend IP Both. 10.0.0.20
```

Common Causes:

```text
Nginx No transmission. X-Real-IP

Nginx No transmission. X-Forwarded-For

There's no substitute for the back end. Header

Backend Untrusted Nginx Proxy

Backend not enabled proxy headers Support
```

Check Nginx configuration: /think

```bash
nginx -T | grep -n "proxy_set_header" -A 10
```

Recommended Configuration:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

---

## Scenario 41: Real IP is forged

Phenomenon:

```text
There's an obvious forgery in the log. IP

Users can access Header Change Record IP

The white list is bypassed.
```

Common Causes:

```text
set_real_ip_from It's too wide.

Wrong configuration. 0.0.0.0/0

Nginx It's a public portal that trusts clients. XFF

Source station allows the network to bypass CDN / SLB Straight Company

Backend directly trusts the client. XFF
```

Troubleshooting Configuration:

```bash
nginx -T | grep -n "set_real_ip_from"
```

Dangerous Configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
```

Handling Direction:

```text
Zoom Out set_real_ip_from Scope

Only upstream agents. IP

Security team restrictions only allow proxy access to sources Stop!

The outermost entrance covers the client XFF

Backend only trusts from Nginx The agent.
```

---

## Scenario 42: allow / deny Whitelist is not effective

Common Causes:

```text
Nginx The judge is... SLB IP

Not configured real_ip

set_real_ip_from Wrong.

allow It says the user is real. IPbut remote_addr Still the agent. IP

The agent has no fax. IP

User bypasses proxy company
```

Troubleshooting:

```bash
tail -n 100 /var/log/nginx/example.access.log
```

```bash
nginx -T | grep -n "allow" -A 10 -B 5
```

```bash
nginx -T | grep -n "real_ip"
```

---

## Scenario 43: X-Forwarded-For becomes longer

Cause:

```text
It'll be added to every level of representation.

It's a long, multi-level proxy link.

Request to circulate between agents

Add Backend or Proxy
```

Troubleshooting:

```bash
awk -F'"xff":"' '{print $2}' /var/log/nginx/example.access.log | head
```

Or directly search in regular logs:

```bash
tail -n 100 /var/log/nginx/example.access.log
```

Need to pay attention to:

```text
Existence of proxy loops

Is it multiple? Nginx Organisation

Is there an unusual client who fakes it for a long time? XFF
```

---

## Seventeen. Log Analysis Commands

---

## Scenario 44: Statistics Real Client IP TopN

If the first column in access.log is `$remote_addr`:

```bash
awk '{print $1}' /var/log/nginx/example.access.log | sort | uniq -c | sort -nr | head
```

---

## Scenario 45: Statistics Previous Proxy IP

If the second column records `[$realip_remote_addr]` in the logs, extract according to the actual format.

Example Log:

```text
1.1.1.1 [10.0.0.10] - - [25/Apr/2026:10:00:00 +0800] "GET / HTTP/1.1" 200 ...
```

Statistics Second Column:

```bash
awk '{print $2}' /var/log/nginx/example.access.log | tr -d '[]' | sort | uniq -c | sort -nr | head
```

---

## Scenario 46: Check if XFF is empty

```bash
grep '""' /var/log/nginx/example.access.log | head
```

If it's JSON log and has `"xff":""`:

```bash
grep '"xff":""' /var/log/nginx/example.access.log | head
```

---

## Scenario 47: Query requests for a specific IP

```bash
grep "1.1.1.1" /var/log/nginx/example.access.log | tail -n 50
```

More precise match by first column:

```bash
awk '$1 == "1.1.1.1" {print $0}' /var/log/nginx/example.access.log | tail -n 50
```

---

## Eighteen. Production Notes

---

## 1. Do not unconditionally trust X-Forwarded-For

`X-Forwarded-For` is a request header that can be forged by clients.

Must be used together with:

```text
set_real_ip_from

Security team

Firewall

Credible proxy links
```

---

## 2. Do not configure set_real_ip_from 0.0.0.0/0

Dangerous Configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
```

Risk:

```text
Anyone can fake the truth. IP

Access controls may be bypassed.

Log audits are not credible
```

---

## 3. Source server should restrict access to only proxies

If Nginx is behind CDN / WAF / SLB, should restrict:

```text
Only allowed CDN / WAF / SLB Return source IP Visits Nginx

Internet users are prohibited from bypassing proxy direct links.
```

Otherwise the real IP trust chain will be broken.

---

## 4. After configuring real IP, change log format

Recommend to also record:

```text
$remote_addr

$realip_remote_addr

$http_x_forwarded_for
```

This way, you can see:

```text
The truth of the final identification. IP

Previous Jump Agent IP

Full proxy link
```

---

## 5. Backend should unify real IP reading specification

Backend should not read IPs arbitrarily by each service:

```text
Yes. X-Real-IP

Yes. X-Forwarded-For

Some reading connection sources IP

Some trust is left. IP
```

Recommend to unify:

```text
by Nginx Purge the truth. IP

Backend only trusts from Nginx Yes. X-Real-IP / X-Forwarded-For

Backend frames open a credible proxy configuration
```

---

## 6. Confirm remote_addr is real IP before allow / deny

Before using IP whitelist, check logs:

```bash
tail -n 20 /var/log/nginx/example.access.log
```

Confirm if the first column is real user IP.

---

## 7. Maintain CDN vendor IP ranges

If trusting CDN backsource IP:

```text
CDN IP Possible change in paragraph

Requires regular synchronization

After change reload Nginx

Or it could be real. IP Identification anomalies
```

---

## Nineteen. Summary of Common Commands in This Article

---

## View realip module

```bash
nginx -V 2>&1 | grep http_realip_module
```

---

## View real_ip configuration

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

## Configuration Check and Reload

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

## Construct XFF Test

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
tail -n 100 /var/log/nginx/error.log
```

---

## Packet Capture to View Header

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Forwarded-For"
```

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Real-IP"
```

---

## Statistics Real IP

```bash
awk '{print $1}' /var/log/nginx/example.access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$1 == "1.1.1.1" {print $0}' /var/log/nginx/example.access.log | tail -n 50
```

---

## Statistics Previous Proxy IP

```bash
awk '{print $2}' /var/log/nginx/example.access.log | tr -d '[]' | sort | uniq -c | sort -nr | head
```

---

## Check Access Control Configuration

```bash
nginx -T | grep -n "allow" -A 10 -B 5
```

```bash
nginx -T | grep -n "deny" -A 10 -B 5
```

---

## Twenty. One-Sentence Summary

The core of real IP in multi-level proxy is:

```text
$remote_addr
→ Current Nginx Directly seen source IP

X-Forwarded-For
→ Agency link IP List

X-Real-IP
→ It's often used to transmit individual truths IP

real_ip
→ Jean. Nginx From a credible agent. Header Other Organiser IP
```

Core Configuration:

```nginx
set_real_ip_from Credible agentIPOr a segment.;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Backend Forwarding:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Log recommendation to record:

```text
$remote_addr

$realip_remote_addr

$http_x_forwarded_for
```

Production security focus:

```text
Don't trust everything. X-Forwarded-For

Don't write. set_real_ip_from 0.0.0.0/0

Trust only SLB / CDN / WAF / Upstream Nginx Yes. IP Paragraph

Source station should limit access to credible agents only

Don't trust a public network client directly. XFF

allow / deny We have to confirm that. remote_addr It's already true. IP
```

Common issues:

```text
It's all in the log. SLB IP
→ Not configured real_ip or unrecorded upstream XFF

All the back end. Nginx IP
→ Untransmitted X-Real-IP / X-Forwarded-For

The white list is invalid.
→ remote_addr Still the agent. IP

Real IP Forged.
→ set_real_ip_from Too wide or the source station can be bypassed

XFF Long.
→ Added or abnormally forged by multiple agents
```