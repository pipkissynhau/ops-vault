# 09-Nginx Common Fault Troubleshooting: 502, 504, HTTPS, Real IP, Blocklist/Allowlist, and Configuration Not Taking Effect

#Nginx #Fault Troubleshooting #502 #504 #HTTPS #RealIP #Blocklist/Allowlist #Configuration Not Effective #Access Layer #Operation and Maintenance #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operation and Maintenance/09-Nginx Common Fault Troubleshooting: 502, 504, HTTPS, Real IP, Blocklist/Allowlist, and Configuration Not Taking Effect.md

---

## I. Document Description

This document outlines common troubleshooting methods for Nginx at the access layer, focusing on frequently occurring issues in production environments.

Key points include:

- General approach to Nginx fault troubleshooting
- Configuration verification and full configuration checking
- Determining if requests reach Nginx
- Checking if requests hit the correct server
- Verifying if requests hit the correct location block
- Ensuring requests are forwarded to the backend
- Troubleshooting 502 Bad Gateway errors
- Troubleshooting 504 Gateway Timeout errors
- Troubleshooting 499 Client Closed Request errors
- Troubleshooting 413 Request Entity Too Large errors
- Troubleshooting 403 Forbidden errors
- Troubleshooting 404 Not Found errors
- Troubleshooting HTTPS certificate issues
- Checking if HTTP to HTTPS redirection works correctly
- Investigating real IP address anomalies
- Identifying why blocklist/allowlist rules are not effective
- Determining why configuration changes do not take effect
- Troubleshooting failures to reload/restart Nginx
- Combined analysis of access.log and error.log files
- Notes for production-level troubleshooting

This document is part of the Nginx Access Layer Operation and Maintenance series, Chapter 09.

Objectives:

```text
To be able to troubleshoot Nginx faults using a standardized approach

→ To identify whether the issue lies with the client, Nginx, upstream proxies, backend services, or certificates

→ To quickly locate problems related to 502, 504, 403, 404, 499, and 413 errors

→ To confirm whether configuration changes have actually taken effect

→ To use access.log, error.log, curl, ss, and tcpdump for troubleshooting

→ To develop a production-level troubleshooting SOP
```

---

## II. General Approach to Nginx Fault Troubleshooting

Do not rush to modify the configuration when encountering Nginx faults.

Recommended troubleshooting sequence:

```text
Confirm the user's access experience

→ Verify domain name resolution and entry IP address

→ Check if requests reach Nginx

→ Ensure requests hit the correct server

→ Confirm whether requests hit the correct location block

→ Verify if proxy_pass, root, or alias settings are correct

→ Verify that backend services are functioning normally

→ Review access.log files

→ Examine error.log files

→ Use curl, ss, or tcpdump for verification

→ Decide whether to modify the configuration
```

In short:

```text
First, determine where the request goes; then identify where the problem lies.
```

---

## III. Overview of Common Troubleshooting Commands

---

## 1. Configuration Verification

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

## 2. Service Status Check

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

---

## 3. Port Listening Verification

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

## 4. Access Log Review

```bash
tail -n 100 /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/access.log
```

---

## 5. Error Log Review

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
tail -f /var/log/nginx/error.log
```

---

## 6. Request Verification

```bash
curl -I http://127.0.0.1
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -k -I --resolve example.com:443:127```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/static/app.js
```

View the access.log file:

```bash
tail -n 100 /var/log/nginx/access.log
```---

## Scenario 25: Counting 499 Requests

```bash
awk '$9 == 499 {print $0}' /var/log/nginx/access.log | tail -n 50
```

Counting requests for the URL with value 499:

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

JSON logs:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | [.time, .remote_addr, .request_uri, .request_time] | @tsv'
```

---

## Scenario 26: Troubleshooting for 499 Requests

Suggestions for investigation:

```text
- Does it concentrate on certain slow APIs?
- Is it occurring with specific client IPs?
- Does it happen during large file downloads?
- Does it occur during uploads?
- Are the timeouts shorter at the upper-layer SLB/CDN?
- Is there excessive processing time on the backend?
- Could it be due to poor network quality on the user end?
```

Analysis commands:

```bash
awk '$9 == 499 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Section X: Troubleshooting for 413 "Request Entity Too Large"

---

## Scenario 27: What Does 413 Mean?

The error code 413 indicates that:

```text
The client's request body is too large, exceeding Nginx's limits.
```

Common scenarios where this occurs include:

```text
- File uploads
- Large JSON requests
- Importing files
- Image uploads
- Uploading compressed packages
```

---

## Scenario 28: Checking for 413 Errors

To check for such errors, run:

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log | tail -n 100
```

To view the current limit settings, execute:

```bash
nginx -T | grep -n "client_max_body_size"
```

---

## Scenario 29: Increasing the Limit Specifically for Upload Interfaces

You can modify your Nginx configuration to increase the maximum request body size for upload interfaces like this:

```nginx
location /api/upload/ {
    client_max_body_size 500m;

    proxy_pass http://app_backend;

    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
}
```

Production recommendations include:

```text
- Only increase the limit for upload interfaces.
- Avoid setting an unlimited limit across the entire site.
- Pay attention to the disk space used by Nginx's temporary directories.
```

To check the disk usage, run:

```bash
df -h
```

```bash
du -sh /var/cache/nginx
```

---

## Section XI: Troubleshooting for 403 Forbidden

---

## Scenario 30: Common Causes of 403

The error code 403 means that:

```text
Nginx understands the request, but refuses to grant access.
```

Common reasons include:

```text
- Insufficient file permissions.
- Insufficient directory permissions.
- The Nginx worker process does not have read permissions.
- The directory lacks an index file and autoindexing is disabled.
- The `allow` or `deny` directives are set incorrectly.
- Basic Auth authentication fails.
- The `deny all` directive is present in the location block.
- SELinux restrictions are in place.
- The user is attempting to access a hidden file.
- Incorrect configuration of the real IP address results in false allowlist rejections.
```

---

## Scenario 31: Commands for Troubleshooting 403 Errors

To check the error logs, run:

```bash
tail -n 100 /var/log/nginx/error.log
```

To check for permission-related issues, execute:

```bash
grep -i "permission denied" /var/log/nginx/error.log | tail -n 100
```

To identify the Nginx user process, use:

```bash
ps -ef | grep nginx | grep -v grep
```

To check file permissions, run:

```bash
ls -lh /data/www/example/index.html
```

To check directory permissions, execute:

```bash
ls -ld /data /data/www /data/www/example
```

To verify the specific permissions for a file or directory, use:

```bash---

## Scenario 41: HTTP Does Not Redirect to HTTPS

Check:

```bash
curl -I http://example.com
```

View configuration:

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
The 80 server is not configured to redirect.

The server_name does not match.

The default_server is being hit.

Nginx has not been reloaded.

The upper-layer CDN/SLB is handling HTTP requests.

Browser cache interference.
```

---

## XIV. Troubleshooting Real IP Address Issues

---

## Scenario 42: access.log Contains Only SLB IP Addresses

Observation:

```text
All remote_addr values in the log are 10.0.0.10.
```

Common causes:

```text
There is an SLB/CDN/WAF in front of Nginx.

The real_ip setting is not configured.

set_real_ip_from does not include the upstream proxy IP.

The real_ip_header configuration is incorrect.

The upstream server does not send X-Forwarded-For.

The configuration has been changed but not reloaded.
```

Troubleshooting steps:

```bash
nginx -T | grep -n "set_real_ip_from"
```

```bash
nginx -T | grep -n "real_ip_header"
```

```bash
tail -n 100 /var/log/nginx/access.log
```

Check the header information using packet capture:

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Forwarded-For"
```

---

## Scenario 43: Backend Logs Contain Only Nginx IP Addresses

Common causes:

```text
Nginx does not pass through the X-Real-IP header.

Nginx does not pass through the X-Forwarded-For header.

The backend server does not read the proxy headers.

The backend framework has not enabled trust for proxies.

The backend directly uses the source IP address of the connection.
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

## Scenario 44: Real IP Address is Being Falsified

Dangerous configuration:

```nginx
set_real_ip_from 0.0.0.0/0;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Troubleshooting steps:

```bash
nginx -T | grep -n "set_real_ip_from"
```

Test IP falsification:

```bash
curl -H "X-Forwarded-For: 9.9.9.9" -H "Host: example.com" http://127.0.0.1/
```

Check the logs:

```bash
tail -n 20 /var/log/nginx/access.log
```

Production best practice:

```text
Only trust trusted proxy IP addresses.

Do not trust 0.0.0.0/0.

The origin server should only allow access from CDN/WAF/SLB.

The backend should only trust the headers cleaned by Nginx.
```

---

## XV. Troubleshooting Why Blocklist/Allowlist Rules Are Not Effective

---

## Scenario 45: allow/deny Rules Do Not Take Effect

Common causes:

```text
Nginx sees the SLB IP address, not the real user's IP.

The real_ip setting is not configured correctly.

set_real_ip_from is configured incorrectly.

The allow rule specifies the real IP, but remote_addr still shows the proxy IP.

The request matches another location in Nginx configuration.

The allow/deny rules are written in the wrong context.

The order of the rules is incorrect.

Users bypass the proxy and connect directly to the origin server.
```

Check the configuration:

```bash
nginx -T | grep -n "allow" -A 10 -B 5
```

```bash
nginx -T | grep -n "deny" -A 10 -B 5
```

Check the source IP addresses in the logs:

```bash
tail -n 50 /var/log/nginx/access.log
```

---

## Scenario 46: Example of an Allowlist in the Management Backend

```nginx
location /admin/ {
    allow 1.1.1.1;
Check for upstream errors in error.log:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

Check for timeout issues:

```bash
grep -i "timeout" /var/log/nginx/error.log | tail -n 100
```

---

## Section 19: Comprehensive Troubleshooting SOP

---

## Scenario 58: User reports that the website cannot be opened

Troubleshooting sequence:

```text
Confirm domain name resolution

→ Confirm port availability

→ Verify if requests are reaching access.log

→ Check if Nginx is running

→ Ensure server_name matches correctly

→ Review error.log

→ Verify if backend services are functioning properly

→ Check if upper-layer CDN/SLB is experiencing issues
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

## Scenario 59: User reports a 502 error on an API

Troubleshooting sequence:

```text
Check access.log to confirm the URL

→ Review error.log for specific upstream errors

→ Verify upstream configuration

→ Check backend port settings

→ Perform a curl health check on the backend

→ Monitor backend service status and logs

→ Determine if any abnormal nodes need to be removed
```

Commands:

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
nc -zv -w 2 backendIP backendPort
```

```bash
curl -v http://backendIP:backendPort/health
```

---

## Scenario 60: User reports slow API performance or a 504 error

Troubleshooting sequence:

```text
Count occurrences of 504 errors

→ Check request_time versus upstream_response_time

→ Verify timeout settings in error.log

→ Directly test the backend API

→ Monitor backend CPU/memory/IO usage

→ Examine databases and dependencies

→ Consider potential API optimization or asynchronous processing
```

Commands:

```bash
awk '$9 == 504 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://backendIP:backendPort/slowAPI
```

```bash
top
```

```bash
iostat -x 1 5
```

---

## Scenario 61: User reports HTTPS errors

Troubleshooting sequence:

```text
Verify the validity of the online certificate

→ Check the certificate's SAN fields

→ Verify the certificate chain

→ Confirm the Nginx configuration file path

→ Ensure consistency across multiple nodes if using certificates

→ Verify if CDN/SLB are using separate certificates
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

## Section 20: Production Best Practices

---

## 1. Locate the issue before modifying configuration

When troubleshooting, first confirm:

```text
Whether requests are reaching Nginx

If Nginx is forwarding them to the backend

Whether the backend is responding

Where the response fails
```

---

## 2. Using nginx -T is more reliable than grep for individual files

Effective configurations may come from multiple include files.

Recommendation:

```bash
nginx -T
```

Instead of just checking:

```bash
cat /etc/nginx/nginx.conf
```

---

## 3. Always back up```bash
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

## Analysis of error.log

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

## HTTPS Verification

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

## Real IP and Blocklist/Allowlist

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
tcpdump -i any -nn host 后端IP and port 后端端口
```

```bash
tcpdump -i any -A -s 0 port 80 | grep -i "X-Forwarded-For"
```

---

## Configuration Backup

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## Summary

The core steps in Nginx troubleshooting are:

```text
Did the request reach Nginx?

→ Which server was targeted?

→ Which location was processed?

→ Was it forwarded to the backend?

→ Did the backend respond correctly?

→ What status code did Nginx return?

→ What specific errors are listed in error.log?
```

Common status codes include:

```text
502
→ Backend connection failed, abnormal closure, or unavailable upstream server

504
→ Backend response timed out or connection timed out

499
→ Client disconnected prematurely

413
→ Request body exceeded the client_max_body_size limit

403
· Insufficient permissions, access denied, no homepage in directory, blocklist refusal

404
· File not found, incorrect path forwarding, or wrong location/root/alias configuration

Key commands include:

```text
nginx -t
→ Check configuration syntax

nginx -T
→ View the fully effective configuration

curl -H "Host: example.com"
→ Verify