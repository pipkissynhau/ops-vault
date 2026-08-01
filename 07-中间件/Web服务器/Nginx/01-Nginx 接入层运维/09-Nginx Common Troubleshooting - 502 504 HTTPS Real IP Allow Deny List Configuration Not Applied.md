# 09-Nginx Common Troubleshooting: 502, 504, HTTPS, Real IP, Black/White List, and Configuration Not Taking Effect

#Nginx #FaultCheck. #502 #504 #HTTPS #RealIp #Black&WhiteList #ConfigureNotEffective #AccessLayer #Transport #SRE

---

## Recommended Path

07-Middlewares/Web Server/Nginx/01-Nginx Access Layer Operations/09-Nginx Common Troubleshooting: 502, 504, HTTPS, Real IP, Black/White List, and Configuration Not Taking Effect.md

---

## I. Document Description

This document compiles common troubleshooting methods for Nginx access layer issues, focusing on high-frequency problems in production environments to establish a troubleshooting path.

This article covers:

- General troubleshooting approach for Nginx
- Configuration checks and full configuration verification
- Whether the request reaches Nginx
- Whether the request hits the correct server
- Whether the request hits the correct location
- Whether the request is forwarded to the backend
- 502 Bad Gateway troubleshooting
- 504 Gateway Timeout troubleshooting
- 499 Client Closed Request troubleshooting
- 413 Request Entity Too Large troubleshooting
- 403 Forbidden troubleshooting
- 404 Not Found troubleshooting
- HTTPS certificate troubleshooting
- HTTP redirect to HTTPS not taking effect troubleshooting
- Real IP anomaly troubleshooting
- Black/White List not taking effect troubleshooting
- Configuration changes not taking effect troubleshooting
- Nginx reload / restart failure troubleshooting
- Combined analysis of access.log and error.log
- Production troubleshooting notes

This article is the 09th in the Nginx access layer operations series.

This article's objectives:

```text
We can follow a fixed path. Nginx Fault

→ Can distinguish between problems on the client,NginxUpstream agent, back-end service or certificate level

→ It can be quickly located. 502 / 504 / 403 / 404 / 499 / 413

→ Can judge if the configuration really works

→ It's a combination. access.logI don't know.error.logI don't know.curlI don't know.ssI don't know.tcpdump Check.

→ It creates a production-access barrier. SOP
```

---

## II. General Troubleshooting Approach for Nginx

Do not modify Nginx configuration immediately when encountering issues.

Recommended troubleshooting order:

```text
Confirm user access

→ Confirm domain name resolution and entry IP

→ Confirm the request. Nginx

→ Make sure it hits right. server

→ Make sure it hits right. location

→ Confirm. proxy_pass / root / alias Correct?

→ Confirm whether back-end services are normal.

→ View access.log

→ View error.log

→ Combined curl / ss / tcpdump Authentication

→ Whether or not to change the configuration
```

One-sentence summary:

```text
First, to determine where the request goes and then to determine where the problem is.
```

---

## III. Common Troubleshooting Commands Overview

---

## 1. Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T > /tmp/nginx-full-config-$(date +%F-%H%M%S).txt 2>&1
```

---

## 2. Service Status

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

---

## 3. Port Listening

```bash
ss -tunlp | grep nginx
```

```bash
ss -lntp | grep ':80'
```

```bash
ss -lntp | grep ':443'
```

---

## 4. Access Log

```bash
tail -n 100 /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/access.log
```

---

## 5. Error Log

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
tail -f /var/log/nginx/error.log
```

---

## 6. Request Validation

```bash
curl -I http://127.0.0.1
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -k -I --resolve example.com:443:127.0.0.1 https://example.com
```

---

## 7. Backend Connectivity

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## 8. Packet Capture

```bash
tcpdump -i any -nn port 80
```

```bash
tcpdump -i any -nn port 443
```

```bash
tcpdump -i any -nn host BackendIP and port Backend
```

---

## IV. Step 1: Confirm Whether the Request Reaches Nginx

---

## Scenario 1: User Reports Access Issues - Check Whether the Request Reaches Nginx

Real-time access log check:

```bash
tail -f /var/log/nginx/access.log
```

Then access from the client:

```bash
curl -v http://example.com/
```

If access.log has no logs, the request may not have reached the current Nginx.

Common causes:

```text
DNS It's not this machine.

Request made. CDN / SLB / Other Nginx Nodes

Security's not clear.

The firewall's not clear.

Nginx No listening port

SLB Other Organiser

The user visited the wrong domain name

This is not the current business entrance.
```

---

## Scenario 2: Check Domain Resolution

```bash
nslookup example.com
```

Or:

```bash
dig example.com
```

If dig is not available on the local machine:

```bash
apt install -y dnsutils
```

Or:

```bash
yum install -y bind-utils
```

Need to confirm:

```text
Which domain did you solve? IP

Whether to parse to Current Nginx

Parsing To SLB / CDN

Are there multiples? A Records

Whether or not DNS Cache not refreshed
```

---

## Scenario 3: Check Whether Nginx is Listening on the Port

```bash
ss -lntp | grep ':80'
```

```bash
ss -lntp | grep ':443'
```

If no listening:

```bash
systemctl status nginx
```

```bash
nginx -t
```

```bash
journalctl -u nginx -n 100
```

---

## Scenario 4: Remote Test Port Reachability

```bash
nc -zv -w 2 example.com 80
```

```bash
nc -zv -w 2 example.com 443
```

If the port is unreachable, focus on checking:

```text
Security team

Firewall

SLB Listen

Nginx Listen

Network of cloud manufacturers ACL

Whether the server binds the public network IP

Passage of request CDN / WAF
```

---

## V. Step 2: Confirm Whether the Request Hits the Correct Server

---

## Scenario 5: Test with Specified Host on the Local Machine

Do not use only:

```bash
curl -I http://127.0.0.1
```

Recommended:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

Reason:

```text
Nginx Pass. Host Match server_name

No, I don't. Host Possible hit. default_server

Could not close temporary folder: %s
```

---

## Scenario 6: Check server_name Configuration

```bash
nginx -T | grep -n "server_name"
```

Check for a specific domain:

```bash
nginx -T | grep -n "example.com"
```

View the configuration context for the domain:

```bash
nginx -T | grep -n "server_name example.com" -A 80
```

---

## Scenario 7: Request Hits default_server

Check the default server:

```bash
nginx -T | grep -n "default_server" -A 50
```

Common issues:

```text
server_name Wrong

Request Host Do not match

No matching domain name configured

Multiple server_name Repeat

listen Port is different

HTTP and HTTPS server Separated, but only one.
```

---

## VI. Step 3: Confirm Whether the Request Hits the Correct Location

---

## Scenario 8: Check location Configuration

```bash
nginx -T | grep -n "location"
```

Check location for a specific server:

```bash
nginx -T | grep -n "server_name example.com" -A 120
```

---

## Scenario 9: Common Manifestations of Location Matching Errors

Common phenomena:

```text
/api Requesting Frontend location Take over.

Static resource forwarded backwards to proxy

Managing backstage path hits normal. location

The exact match didn't hit.

Right. location He took the request.

SPA Refresh 404
```

Common causes:

```text
location Order error

Right. location Prioritization Error

^~ Not used

location / Too broad.

/api/ and /api No difference.

proxy_pass And... / And not. / Wrong
```

---

## Scenario 10: Validate Different Paths

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/static/app.js
```

Check access.log:

```bash
tail -n 100 /var/log/nginx/access.log
```

---

## VII. 502 Bad Gateway Troubleshooting

---

## Scenario 11: What is 502

502 typically indicates:

```text
Nginx Could not close temporary folder: %s upstream Access to effective responses
```

Common causes:

```text
Backend service not started

Backend is not listening

proxy_pass Chile

upstream Node All Not Available

Nginx Backend's dead.

Backend Process Collapse

Backend Active Close Connection

Backend returns illegal response

Backend protocols do not match, e.g. HTTP Match HTTPS Or vice versa.
```

---

## Scenario 12: Check error.log Immediately for 502

```bash
tail -n 100 /var/log/nginx/error.log
```

Filter common errors:

```bash
grep -Ei "connect\(\) failed|connection refused|no live upstreams|upstream prematurely closed|bad gateway" /var/log/nginx/error.log | tail -n 100
```

---

## Scenario 13: connection refused

Error similar to:

```text
connect() failed (111: Connection refused) while connecting to upstream
```

Explanation:

```text
Nginx Access to target IP

But the target port is not wired or rejected.
```

Troubleshooting:

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

Check on the backend machine:

```bash
ss -lntp | grep Backend
```

---

## Scenario 14: no live upstreams

Error similar to:

```text
no live upstreams while connecting to upstream
```

Explanation:

```text
Nginx Yes. upstream No backend available
```

Common causes:

```text
All backends fail.

max_fails / fail_timeout Caused temporary inoperable

All Nodes by down

backup Node is not available

upstream Configure Error
```

Troubleshooting:

```bash
nginx -T | grep -n "upstream" -A 30
```

```bash
grep -i "no live upstreams" /var/log/nginx/error.log | tail -n 100
```

Check each backend one by one:

```bash
for ip in 10.0.0.21 10.0.0.22 10.0.0.23; do echo "check $ip"; nc -zv -w 2 $ip 8080; done
```

---

## Scenario 15: upstream prematurely closed connection

Error similar to:

```text
upstream prematurely closed connection while reading response header from upstream
```

Explanation:

```text
Backend Nginx Close connection early before reading response
```

Common causes:

```text
Backend Process Collapse

Apply Backend Abnormal Exit

Backend active breakup

Backend connect pool anomaly

Backend returns data incomplete

Application Frame Abnormal
```

Troubleshooting:

```bash
grep -i "upstream prematurely closed" /var/log/nginx/error.log | tail -n 100
```

Check further on the backend:

```bash
journalctl -u Backend Service Name -n 100
```

```bash
tail -n 100 /var/log/Backend Application/app.log
```

---

## Scenario 16: HTTP / HTTPS Protocol Configuration Error

Error example:

Backend is actually HTTP:

```text
http://10.0.0.21:8080
```

But Nginx is written as:

```nginx
proxy_pass https://10.0.0.21:8080;
```

Or backend is actually HTTPS, but Nginx is written as: /think

```nginx
proxy_pass http://backend.example.com;
```

Troubleshooting:

```bash
curl -v http://BackendIP:Backend/
```

```bash
curl -vk https://BackendIP:Backend/
```

Check which protocol is working.

---

## Scenario 17: 502 Troubleshooting Summary

```text
Look. error.log

→ Cha. upstream Error keyword

→ Inspection proxy_pass / upstream Configure

→ nc Check Backend

→ curl Check Backend HTTP Response

→ Check backend service status

→ Check backend application logs

→ Network checks, firewalls, security units.

→ Grab the bag if necessary.
```

---

## VIII. 504 Gateway Timeout Troubleshooting

---

## Scenario 18: What is 504

504 typically indicates:

```text
Nginx Waiting for backend response timeout
```

Common causes:

```text
Backend processing slow

Database slow query

Back-end pool full

Back-end connectors pool run out.

External dependency interface slow

Backend CPU / Memory / IO Pressure high.

Nginx proxy_read_timeout Too short.

Nginx Backend network links are abnormal.
```

---

## Scenario 19: Check upstream timed out

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

Common errors:

```text
upstream timed out while connecting to upstream

upstream timed out while reading response header from upstream

upstream timed out while reading upstream
```

Meaning differences:

```text
connecting to upstream
→ Connect backend timeout

reading response header
→ When the back end response head is over

reading upstream
→ Read backend response timeout
```

---

## Scenario 20: Statistics for 504 URLs

Normal access.log:

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

JSON access.log:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 504) | .uri' | sort | uniq -c | sort -nr | head
```

---

## Scenario 21: Check 504 Request Duration

Normal logs if the last column is duration:

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

JSON logs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 504) | [.time, .remote_addr, .request_uri, .request_time, .upstream_response_time] | @tsv'
```

---

## Scenario 22: Directly Test Backend Interface

```bash
curl -v http://BackendIP:Backend/Slow Interface
```

If the backend interface is very slow, the problem is not in Nginx itself.

Continue troubleshooting the backend:

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

```bash
journalctl -u Backend Service Name -n 100
```

---

## Scenario 23: Do Not Just Increase Timeout for 504

Not recommended to blindly set global timeouts:

```nginx
proxy_read_timeout 3600s;
```

Correct approach:

```text
Make sure which interface is slow.

Make sure the back end is slow.

If it's a long mission interface, alone. location Adjustment

It's more about long assignments.
```

Slow interface specific configuration example:

```nginx
location /api/report/export {
    proxy_pass http://app_backend;

    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
}
```

---

## IX. 499 Client Closed Request Troubleshooting

---

## Scenario 24: What is 499

499 is a Nginx-specific status code, indicating:

```text
Client in Nginx Active disconnect before returning response
```

Common causes:

```text
User Close Page

Browser requests timeout

Client network disconnected

Upper Agent Timeout

Backend's too slow to wait.

Mobile weak end net

Download or upload aborted
```

---

## Scenario 25: Statistics for 499 Requests

```bash
awk '$9 == 499 {print $0}' /var/log/nginx/access.log | tail -n 50
```

Statistics for 499 URLs:

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

JSON logs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | [.time, .remote_addr, .request_uri, .request_time] | @tsv'
```

---

## Scenario 26: 499 Troubleshooting Directions

Troubleshooting:

```text
Whether to focus on a slow interface

Whether to focus on certain clients IP

Whether or not the download of a large file occurred

Whether it happened in upload

Upper SLB / CDN Shorter timeout

Is the back end excessive?

Whether user network quality is available Bad
```

Analysis commands:

```bash
awk '$9 == 499 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## X. 413 Request Entity Too Large Troubleshooting

---

## Scenario 27: What is 413

413 indicates:

```text
Client request is too big to exceed Nginx Limits
```

Common scenarios:

```text
Uploading files

Large JSON Request

Import File

Image Upload

Compact Package Upload
```

---

## Scenario 28: Check 413 Errors

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

Check current limits:

```bash
nginx -T | grep -n "client_max_body_size"
```

---

## Scenario 29: Increase Limits Specifically for Upload Interfaces

```nginx
location /api/upload/ {
    client_max_body_size 500m;

    proxy_pass http://app_backend;

    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
}
```

Production recommendations:

```text
Zoom in only for upload interfaces

Don't go all the way up.

I'm also concerned. Nginx Temporary directory disk space
```

Check disk space:

```bash
df -h
```

```bash
du -sh /var/cache/nginx
```

---

## XI. 403 Forbidden Troubleshooting

---

## Scenario 30: Common Causes of 403

403 indicates:

```text
Nginx Request understood but visit denied
```

Common causes:

```text
Inadequate document permissions

Inadequate directory permissions

Nginx worker User without read permission

No directory. index and autoindex Close

allow / deny Reject

Basic Auth Not adopted

location Medium deny all

SELinux Limits

Visited banned hidden files

Real IP The configuration error caused the white list to be wrongly rejected
```

---

## Scenario 31: 403 Troubleshooting Commands

Check error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

Check permission errors:

```bash
grep -i "permission denied" /var/log/nginx/error.log | tail -n 100
```

Check Nginx user:

```bash
ps -ef | grep nginx | grep -v grep
```

Check file permissions:

```bash
ls -lh /data/www/example/index.html
```

Check directory permissions:

```bash
ls -ld /data /data/www /data/www/example
```

Check permissions hierarchically:

```bash
namei -l /data/www/example/index.html
```

---

## Scenario 32: Directory Missing index Causes 403

If the request is:

```text
/docs/
```

The actual directory exists:

```text
/data/www/example/docs/
```

But there is no:

```text
index.html
```

And:

```nginx
autoindex off;
```

It may return 403.

Handling method:

```text
Increase index.html

Or configuration try_files Back 404

Or start with caution. autoindex
```

---

## XII. 404 Not Found Troubleshooting

---

## Scenario 33: Common Causes of 404

404 indicates:

```text
No resources requested
```

Common causes:

```text
Static file does not exist

root Path error

alias Path error

root / alias Confusion

try_files Configure Error

SPA No retreat. index.html

proxy_pass Pathbelt / Or not. / Wrong match.

There's no way back.

Request error server

Request error location

Configure Not reload
```

---

## Scenario 34: 404 Troubleshooting for Static Resources

Check configuration:

```bash
nginx -T | grep -n "root"
```

```bash
nginx -T | grep -n "alias"
```

Confirm file existence:

```bash
ls -lh /data/www/example/static/app.js
```

Request validation:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/static/app.js
```

Check error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 35: 404 Troubleshooting for Reverse Proxy

Focus on checking:

```text
Whether the backend really has that path

proxy_pass Whether to bring /

location Match correctly

/api Whether the prefix was retained or removed
```

Directly test backend:

```bash
curl -v http://127.0.0.1:8080/api/users
```

```bash
curl -v http://127.0.0.1:8080/users
```

Check Nginx configuration:

```bash
nginx -T | grep -n "proxy_pass" -A 5 -B 5
```

---

## Scenario 36: SPA Refresh 404

Phenomenon:

```text
First page opens

Click front end route normal

Refresh /dashboard Back 404
```

Cause:

```text
The front-end route has no real file on disk

Nginx No retreat. index.html
```

Configuration:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

---

## XIII. HTTPS Troubleshooting

---

## Scenario 37: Certificate Expired

Check online certificate:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Common causes:

```text
The certificate does expire.

Update to Error Path

Nginx Not reload

Node in multiple nodes not updated

CDN / SLB Certificate not updated

Browser accesses upper-level proxy certificates
```

---

## Scenario 38: Certificate Domain Mismatch

Check SAN:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

Common causes:

```text
The certificate does not contain the domain name

server_name Matched Error server

SNI Not with

Nginx Return Default Certificate

Visits IP Not a domain name.

Could not close temporary folder: %s
```

---

## Scenario 39: Incomplete Certificate Chain

Check certificate chain:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

Pay attention to:

```text
Verify return code: 0 (ok)
```

If it's not 0, there may be:

```text
Missing intermediate certificate

ssl_certificate It's useless. fullchain

Certificate chain order error

Client does not trust CA
```

---

## Scenario 40: Local HTTPS Testing

```bash
curl -k -I -H "Host: example.com" https://127.0.0.1
```

More recommended:

```bash
curl -I --resolve example.com:443:127.0.0.1 https://example.com
```

If testing a remote node:

```bash
curl -I --resolve example.com:443:10.0.0.20 https://example.com
```

---

## Scenario 41: HTTP Not Redirecting to HTTPS

Check:

```bash
curl -I http://example.com
```

Check configuration:

```bash
nginx -T | grep -n "return 301" -A 5 -B 5
```

```bash
nginx -T | grep -n "listen 80" -A 30
```

Correct configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}
```

Common causes:

```text
80 server Other Organiser

server_name Do not match

Hit. default_server

Nginx Not reload

Upper CDN / SLB Got it. HTTP

Browser Cache Interference
```

---

## XIV. Real IP Anomaly Troubleshooting

---

## Scenario 42: access.log Shows Only SLB IP

Phenomenon:

```text
Log remote_addr Both. 10.0.0.10
```

Common causes:

```text
Nginx It's up ahead. SLB / CDN / WAF

Not configured real_ip

set_real_ip_from No upstream agent. IP

real_ip_header Wrong match.

It's not going upstream. X-Forwarded-For

The configuration has changed but not yet. reload
```

Troubleshooting:

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
tail -n 100 /var/log/nginx/access.log
```

Capture packets to check Header:

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Forwarded-For"
```

---

## Scenario 43: Backend Logs Show Only Nginx IP

Common causes:

```text
Nginx No transmission. X-Real-IP

Nginx No transmission. X-Forwarded-For

There's no substitute for the back end. Header

Backend not enabled trust proxy

Backend direct to connect source IP
```

Check Nginx configuration:

```bash
nginx -T | grep -n "proxy_set_header" -A 10
```

Recommended configuration:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

---

## Scenario 44: Real IP Forged

Dangerous configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Troubleshooting:

```bash
nginx -T | grep -n "set_real_ip_from"
```

Test forging:

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

Check logs:

```bash
tail -n 20 /var/log/nginx/access.log
```

Production principles:

```text
Only trusted agents. IP

Don't trust me. 0.0.0.0/0

Source station only allowed CDN / WAF / SLB Visits

Backend Only Trust Nginx After cleaning. Header
```

---

## XV. Whitelist/Blacklist Not Working Troubleshooting

---

## Scenario 45: allow/deny Not Working

Common causes:

```text
Nginx See? SLB IPNot a real user IP

Not configured real_ip

set_real_ip_from Wrong match.

allow It's true. IPbut remote_addr Or an agent? IP

He's got another hit. location

allow / deny Write in Wrong Context

Wrong sequence of rules

User bypasses proxy direct links
```

Check configuration: /think

```bash
nginx -T | grep -n "allow" -A 10 -B 5
```

```bash
nginx -T | grep -n "deny" -A 10 -B 5
```

Check the source IP in the logs:

```bash
tail -n 50 /var/log/nginx/access.log
```

---

## Scenario 46: Example of Managing Backend Whitelist

```nginx
location /admin/ {
    allow 1.1.1.1;
    allow 2.2.2.2;
    deny all;

    proxy_pass http://admin_backend;
}
```

If Nginx is behind SLB, configure real_ip first:

```nginx
set_real_ip_from 10.0.0.10;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

---

## Sixteen. Troubleshooting Configuration Changes Not Taking Effect

---

## Scenario 47: Common Reasons for Configuration Changes Not Taking Effect

Common reasons:

```text
Error file modified

The file was not taken include

nginx -t Failed

reload Not implemented successfully

He's got another hit. server

He's got another hit. location

server_name Do not match

HTTP and HTTPS Only one of the configurations was changed.

It's up ahead. CDN / SLB Cache

Browser Cache

Multiple Nginx Only one.
```

---

## Scenario 48: Confirming Whether the Configuration is Loaded

Check include:

```bash
nginx -T | grep -n "include"
```

Check configuration keywords:

```bash
nginx -T | grep -n "example.com"
```

```bash
nginx -T | grep -n "proxy_pass"
```

```bash
nginx -T | grep -n "client_max_body_size"
```

---

## Scenario 49: Confirming Whether reload Was Successful

```bash
nginx -t
```

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

## Scenario 50: Confirming That You Are Accessing the Current Node

Local testing:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

Remote testing with specified IP:

```bash
curl -I --resolve example.com:80:10.0.0.20 http://example.com
```

HTTPS with specified IP:

```bash
curl -I --resolve example.com:443:10.0.0.20 https://example.com
```

---

## Seventeen. Troubleshooting Nginx reload / restart Failures

---

## Scenario 51: Check nginx -t First When reload Fails

```bash
nginx -t
```

Common errors:

```text
Minimal

Unmatched parenthesis

Command error

Context Error

Certificate path does not exist

Certificate does not match private key

Port Conflict

Log directory does not exist

Insufficient Permissions
```

---

## Scenario 52: Check Configuration by Line Number

If error:

```text
/etc/nginx/conf.d/app.conf:15
```

Check nearby lines:

```bash
sed -n '10,20p' /etc/nginx/conf.d/app.conf
```

---

## Scenario 53: Port is Occupied

```bash
ss -lntp | grep ':80'
```

```bash
ss -lntp | grep ':443'
```

If another process is using the port, confirm whether another Web service was mistakenly started.

---

## Scenario 54: Log Directory Permission Issues

Check log directory:

```bash
ls -ld /var/log/nginx
```

Check error log:

```bash
tail -n 100 /var/log/nginx/error.log
```

If custom log directory:

```bash
ls -ld /data/logs/nginx
```

Confirm Nginx user:

```bash
ps -ef | grep nginx | grep -v grep
```

---

## Eighteen. Combined Analysis of access.log and error.log

---

## Scenario 55: What access.log is Suitable for Answering

access.log is suitable for answering:

```text
Request arrival Nginx

Who did?

Which one? URL

Return what status code?

How long does it take?

Which one did you call? upstream

Whether the request for a certain period of time surged

Some IP Unusual access
```

---

## Scenario 56: What error.log is Suitable for Answering

error.log is suitable for answering:

```text
Why? 502

Why? 504

Failed to connect backend

Whether backend timeout

Inadequate permission

Failed to load certificate

Whether configuration is wrong

Whether the file does not exist

Whether to close the connection upstream
```

---

## Scenario 57: Combined Commands for Troubleshooting 5xx Errors

Statistical status codes:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

Check 5xx URLs:

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Check error.log upstream errors:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

Check timeout:

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

---

## Nineteen. Comprehensive Troubleshooting SOP

---

## Scenario 58: User Reports Website is Unreachable

Troubleshooting order:

```text
Confirm domain name resolution

→ Confirm port availability

→ Confirm the request. access.log

→ Confirm. Nginx Whether to run

→ Confirm. server_name Matches

→ View error.log

→ Confirm whether back-end services are normal.

→ Confirm Top CDN / SLB Is it unusual?
```

Commands:

```bash
nslookup example.com
```

```bash
nc -zv -w 2 example.com 80
```

```bash
nc -zv -w 2 example.com 443
```

```bash
systemctl status nginx
```

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 59: User Reports 502 Interface Issues

Troubleshooting order:

```text
View access.log Confirm. URL

→ View error.log Specific upstream Error

→ Inspection upstream Configure

→ Check Backend

→ curl Backend health check

→ View backend service status and log

→ To determine whether or not to remove the anomaly
```

Commands:

§

```bash
awk '$9 == 502 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
nginx -T | grep -n "upstream" -A 30
```

```bash
nc -zv -w 2 BackendIP Backend
```

```bash
curl -v http://BackendIP:Backend/health
```

---

## Scenario 60: User Reports Slow Interface or 504

Troubleshooting order:

```text
Statistics 504 URL

→ View request_time / upstream_response_time

→ View error.log timeout

→ Direct Request Backend Interface

→ Check Backend CPU / Memory / IO

→ Database and dependency checks

→ Determine whether interface optimization or hierarchization is required
```

Commands:

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://BackendIP:Backend/Slow Interface
```

```bash
top
```

```bash
iostat -x 1 5
```

---

## Scenario 61: User Reports HTTPS Errors

Troubleshooting order:

```text
Check online certificate validity period

→ Check Certificate SAN

→ Check the certificate chain

→ Inspection Nginx Configure Path

→ Check if multiple node certificates are inconsistent

→ Inspection CDN / SLB Whether to use separate certificates
```

Commands:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

```bash
nginx -T | grep -n "ssl_certificate" -A 5 -B 5
```

---

## Twenty. Production Notes

---

## 1. Do Not Modify Configuration First, Locate the Chain First

When troubleshooting, first confirm:

```text
Is the request made? Nginx

Nginx Did you forward it to the backend?

Did the back end respond?

Where did the response fail?
```

---

## 2. nginx -T is More Reliable Than grep Single File

Because the actual effective configuration may come from multiple include files.

Recommended:

```bash
nginx -T
```

Do not only check:

```bash
cat /etc/nginx/nginx.conf
```

---

## 3. Must Backup Before Production Modifications

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

Or:

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## 4. Must Run nginx -t Before reload

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## 5. Multi Nginx Instances Must Have Consistent Configurations

When troubleshooting, confirm:

```text
Which one is the request? Nginx

Is there only one anomaly?

Whether a certificate is not updated

Whether a particular station is configured is not available reload

Whether or not SLB Backend is abnormal
```

---

## 6. 502 and 504 Errors Usually Require Further Backend Checks

Nginx is only the entry layer.

When encountering 502 / 504, do not only modify Nginx.

Also check:

```text
Backend Service Status

Backend Log

Backend Resources

Database

Cache

External dependency

Network links
```

---

## 7. HTTPS Issues Must Distinguish Which Layer's Certificate

If the chain is:

```text
User

→ CDN / SLB

→ Nginx
```

Users typically see the CDN / SLB certificate, not necessarily Nginx's certificate.

---

## 8. Whitelist/Blacklist Must Combine with Real IP

If Nginx has a proxy in front, first confirm:

```text
remote_addr Is it a real client? IP
```

Otherwise allow/deny may judge based on SLB IP.

---

## Twenty-one. Summary of Common Commands in This Article

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "server_name"
```

```bash
nginx -T | grep -n "location"
```

```bash
nginx -T | grep -n "proxy_pass"
```

```bash
nginx -T | grep -n "upstream" -A 30
```

```bash
nginx -T | grep -n "ssl_certificate" -A 5 -B 5
```

---

## Service Status

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
systemctl reload nginx
```

```bash
systemctl restart nginx
```

---

## Ports and Network

```bash
ss -tunlp | grep nginx
```

```bash
ss -lntp | grep ':80'
```

```bash
ss -lntp | grep ':443'
```

```bash
nc -zv -w 2 example.com 80
```

```bash
nc -zv -w 2 example.com 443
```

```bash
nc -zv -w 2 BackendIP Backend
```

---

## curl Verification

```bash
curl -I http://127.0.0.1
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -v http://BackendIP:Backend/health
```

```bash
curl -I --resolve example.com:80:10.0.0.20 http://example.com
```

```bash
curl -I --resolve example.com:443:10.0.0.20 https://example.com
```

---

## access.log Analysis

```bash
tail -f /var/log/nginx/access.log
```

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 502 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 504 {print $0}' /var/log/nginx/access.log | tail -n 50
```

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## error.log Analysis

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
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -Ei "connect\(\) failed|connection refused|upstream prematurely closed|no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "permission denied" /var/log/nginx/error.log | tail -n 100
```

---

## HTTPS Check

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

```bash
curl -I https://example.com
```

```bash
curl -k -I -H "Host: example.com" https://127.0.0.1
```

---

## Real IP and Black/White Lists

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
nginx -T | grep -n "allow" -A 10 -B 5
```

```bash
nginx -T | grep -n "deny" -A 10 -B 5
```

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

---

## Packet Capture

```bash
tcpdump -i any -nn port 80
```

```bash
tcpdump -i any -nn port 443
```

```bash
tcpdump -i any -nn host BackendIP and port Backend
```

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Forwarded-For"
```

---

## Backup Configuration

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## Twenty-Two, One-Sentence Summary

Nginx troubleshooting core flow is:

```text
Is the request made? Nginx

→ Which hit? server

→ Which hit? location

→ Whether to forward to backend

→ Backend Response Normal

→ Nginx Return what status code?

→ error.log What did it say?
```

Common status code meanings:

```text
502
→ Backend connection failed, backend abnormally closed,upstream Not Available

504
→ Backend response timeout or connection timeout

499
→ Client early disconnect

413
→ Request exceeding client_max_body_size

403
→ Insufficient access, denied, no home page, white list denied

404
→ File does not exist, path forward error,location/root/alias Configure Error
```

Key commands:

```text
nginx -t
→ Check configuration syntax

nginx -T
→ View Full Effect Configuration

curl -H "Host: example.com"
→ Organisation server_name

tail access.log
→ See if the request arrives and the status code.

tail error.log
→ Look. Nginx Specific cause of error

nc / curl Backend
→ Check backend connectivity

openssl s_client
→ Inspection HTTPS Certificates and TLS

tcpdump
→ Check if the request really reaches a level
```

Production recommendations:

```text
Don't change the configuration first.

Don't just look. nginx.confLook. nginx -T

Don't. reload Skip nginx -t

Don't. 502 / 504 All due to... Nginx

Don't be unconfirmed. IP Previously Configure White List

Do not ignore CDN / SLB / WAF This floor.

Don't forget the many. Nginx There may be inconsistencies between configuration and certificate

We need to sink behind the barrier. SOP and conclusion
```