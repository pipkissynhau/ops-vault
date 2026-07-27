# 02-Nginx Reverse Proxy Basics: server, location, and proxy_pass Route Rules

#Nginx #Reverse Proxy #Web Server #Access Layer #proxy_pass #server #location #Ops #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Ops/02-Nginx Reverse Proxy Basics: server, location, and proxy_pass Route Rules.md

---

## I. Document Overview

This article outlines the basic configuration of Nginx as a reverse proxy, focusing on the functions of `server`, `location`, and `proxy_pass` as well as their route forwarding rules.

Key points include:

- What is a reverse proxy
- Nginx's role in reverse proxies
- Basic `server` configuration
- `server_name` domain matching
- Basic `location` matching rules
- Differences between `location /` and `location /api/`
- Basic usage of `proxy_pass`
- Differences between using `/` and not using `/` with `proxy_pass`
- Backend URI concatenation rules
- Multi-path forwarding
- Local interface validation
- Common reverse proxy errors
- Best practices for production configuration

This article is part of the Nginx Access Layer Ops series, Chapter 02.

Learning objectives:

```text
Be able to configure basic reverse proxies

→ Understand the functions of server and location

→ Know where requests are forwarded via proxy_pass

→ Distinguish between using `/` and not using `/` with proxy_pass

→ Verify whether requests enter Nginx using curl

→ Troubleshoot common issues with basic reverse proxies
```

---

## II. What is a Reverse Proxy

A reverse proxy can be understood as follows:

```text
The client does not directly access the backend application.

Instead, the client accesses Nginx first,

which then forwards the request to the backend service.
```

Typical workflow:

```text
User's browser / Client

→ Nginx

→ Backend application service

→ Database / Redis / Other dependencies
```

Example:

```text
User visits:

http://example.com/api/users

Nginx receives the request and forwards it to:

http://127.0.0.1:8080/api/users
```

Nginx's roles here include:

```text
Serving as a unified entry point

Hiding the real backend address

Forwarding requests based on domain names and paths

Providing HTTPS termination

Recording access logs

Implementing basic access control

Performing simple rate limiting

Managing load balancing
```

---

## III. Differences Between Reverse Proxies and Forward Proxies

---

## 1. Forward Proxy

A forward proxy acts on behalf of the client.

Common use cases:

```text
Clients accessing external websites through a proxy
```

Workflow:

```text
Client

→ Proxy server

→ Target website
```

The client is aware that it is using a proxy.

---

## 2. Reverse Proxy

A reverse proxy acts on behalf of the server.

Common use cases:

```text
Users access Nginx,

which then forwards requests to the backend application.
```

Workflow:

```text
Client

→ Nginx

→ Backend service
```

Clients usually do not know the real address of the backend service.

---

## 3. Summary in One Sentence

```text
Forward Proxy
→ Helps clients access other services

Reverse Proxy
→ Helps servers receive user requests
```

---

## IV. Basic Structure of Nginx Reverse Proxies

A basic reverse proxy configuration looks like this:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Explanation:

```text
listen 80
→ Nginx listens on port 80

server_name example.com
→ Matches requests for the domain name example.com

location /
→ Matches all paths

proxy_pass http://127.0.0.1:8080
→ Forwards requests to the backend service running on port 8080 of the local host
```

Request flow:

```text
http://example.com/

→ Nginx:80

→ http://127.0.0.1:8080/
```

---

## V. Basic Server Configuration

---

## Scenario 1: Minimum server configuration

Configuration file:

```bash
vi /etc/nginx/conf.d/example.com.conf
```

Content:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Check configuration:

```bash
nginx -t
```

Reload configuration:

```bash
systemctl reload nginx
``→ 127.0.0.1:8080

Requests starting with /admin/
→ 127.0.0.1:9090

Other requests
→ 127.0.0.1:3000
```

Verification:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/users
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/admin/
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

---

## VII. Common Matching Types for location

Common ways to write location in Nginx:

```nginx
location / {
}
```

```nginx
location /api/ {
}
```

```nginx
location = /health {
}
```

```nginx
location ^~ /static/ {
}
```

```nginx
location ~ \.php$ {
}
```

```nginx
location ~* \.(jpg|png|css|js)$ {
}
```

---

## Scenario 8: Ordinary Prefix Matching

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

Characteristics:

```text
Matches based on the URI prefix

Matches requests starting with /api/
```

---

## Scenario 9: Exact Matching =

```nginx
location = /health {
    return 200 "ok\n";
}
```

Only matches:

```text
/health
```

Does not match:

```text
/health/

/health/check
```

Verification:

```bash
curl -i http://127.0.0.1/health
```

---

## Scenario 10: Priority Prefix Matching ^~

```nginx
location ^~ /static/ {
    root /data/www;
}
```

Meaning:

```text
If the /static/ prefix is matched, the regular expression location will not be continued.
```

Suitable for:

```text
Static resource directories

File upload directories

Paths that should not be affected by regular expression locations
```

---

## Scenario 11: Case-Sensitive Regular Expression Matching ~

```nginx
location ~ \.php$ {
    proxy_pass http://127.0.0.1:9000;
}
```

Means:

```text
Matches URIs ending with .php

Case-sensitive
```

---

## Scenario 12: Case-Insensitive Regular Expression Matching ~*

```nginx
location ~* \.(jpg|jpeg|png|gif|css|js)$ {
    root /data/www;
}
```

Means:

```text
Matches common static resource extensions

Case-insensitive
```

---

## VIII. Simplified Understanding of Location Matching Priority

Common priorities can be simply remembered as:

```text
Exact matching =
→ Highest priority

^~ Prefix matching
→ Once matched, regular expressions are not checked further

Regular expression matching ~ / ~*
→ Matches in the order specified in the configuration

Ordinary prefix matching
→ Selects the longest match
```

Example:

```nginx
location = /api/login {
    proxy_pass http://127.0.0.1:8081;
}

location /api/ {
    proxy_pass http://127.0.0.1:8080;
}

location / {
    proxy_pass http://127.0.0.1:3000;
}
```

Requests:

```text
/api/login
→ Matches the location = /api/login

/api/users
→ Matches the location /api/

/index.html
→ Matches the location /
```

---

## IX. Basic Usage of proxy_pass

`proxy_pass` is used to forward requests to backend services.

Common ways to write it:

```nginx
proxy_pass http://127.0.0.1:8080;
```

It can also be forwarded to a domain name:

```nginx
proxy_pass http://backend.example.com;
```

Or to an upstream server:

```nginx
proxy_pass http://app_backend;
```

---

## Scenario 13: Proxying to a Local Backend Service

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Connection process:

```text
Client

→ Nginx:80

→ 127.0.0.1:8080
```

Testing the backend:

```bash
curl -I http://127.0.0.1:8080
```

| `location /api/ { proxy_pass http://127.0.0.1:8080/backend/; }` | `/api/users` | `/backend/users` | Replace `/api/` with `/backend/` |
| `location / { proxy_pass http://127.0.0.1:8080; }` | `/api/users` | `/api/users` | Keep the original URI |
| `location / { proxy_pass http://127.0.0.1:8080/; }` | `/api/users` | `/api/users` | The difference with `location /` is usually not significant |

---

## Fourteen, Two Confurations That Are Most Likely to Be Confused

---

## Scenario 19: Forwarding While Retaining the /api Prefix

Requirement:

```text
When users access /api/users,
the backend also receives /api/users.
```

Configuration:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

Verification:

```bash
curl -v http://example.com/api/users
```

What the backend receives:

```text
/api/users
```

---

## Scenario 20: Forwarding Without Retaining the /api Prefix

Requirement:

```text
When users access /api/users,
the backend receives /users.
```

Configuration:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

Verification:

```bash
curl -v http://example.com/api/users
```

What the backend receives:

```text
/users
```

---

## Fifteen, Recommendations for Setting Proxy Pass Path Rules

It is recommended to clarify the actual backend service routes first.

If the backend route is:

```text
/api/users
```

Nginx should be configured as follows:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

If the backend route is:

```text
/users
```

Nginx should be configured as follows:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

Production recommendation:

```text
Don't guess.

First, directly use curl to check the actual backend route.

Then decide whether to include the / prefix in the proxy_pass.
```

To test the actual backend route:

```bash
curl -v http://127.0.0.1:8080/api/users
```

```bash
curl -v http://127.0.0.1:8080/users
```

---

## Sixteen, Examples of Multi-Path Reverse Proxies

---

## Scenario 21: Frontend and Backend Are Separated

Requirement:

```text
/        → Front-end service 127.0.0.1:3000
/api/    → Back-end API 127.0.0.1:8080
/admin/  → Admin backend 127.0.0.1:9090
```

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }

    location /admin/ {
        proxy_pass http://127.0.0.1:9090;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

Verification:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/api/health
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/admin/
```

---

## Scenario 22: Multiple Domain Names Forward to Different Services

Configuration:

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

server {
    listen 80;
    server_name web.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

Verification:

```bash
curl -I -H "Host: api.example.com" http://127.0.0.1
``nginx -T | grep -n "server_name example.com"
```

nginx -T | grep -n "proxy_pass"
```

---

## Scenario 26: Local Verification via Host

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

---

## Scenario 27: Checking if Requests are Recorded in access.log

```bash
tail -f /var/log/nginx/access.log
```

Or for specific business logs:

```bash
tail -f /var/log/nginx/example.access.log
```

---

## Scenario 28: Checking if error.log Reports Any Issues

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

---

## Section 20: Common Problem Troubleshooting

---

## Scenario 29: 502 Response is Returned

A 502 usually indicates:

```text
Nginx was unable to obtain a response from the backend.
```

Common causes include:

```text
The backend service is not running.

The backend port is not listening.

The proxy_pass address is incorrect.

There is no network connection between Nginx and the backend.

The backend actively closed the connection.

The backend process encountered an exception.

The backend returned an invalid response.
```

Troubleshooting steps:

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -i "connect() failed" /var/log/nginx/error.log | tail -n 50
```

```bash
ss -lntp | grep ':8080'
```

```bash
nc -zv -w 2 127.0.0.1 8080
```

```bash
curl -v http://127.0.0.1:8080
```

---

## Scenario 30: 404 Response is Returned

Possible reasons include:

```text
The request was not forwarded to the intended backend.

The proxy_pass path contains a / or lacks it, and it is incorrect.

The actual backend does not have that route.

An incorrect location block was matched.

The default server was selected.

A file in the static directory does not exist.
```

Troubleshooting steps:

```bash
nginx -T | grep -n "location"
```

```bash
nginx -T | grep -n "proxy_pass"
```

```bash
curl -v http://127.0.0.1:8080/api/users
```

```bash
tail -n 100 /var/log/nginx/access.log
```

---

## Scenario 31: Changes to Configuration Do Not Take Effect

Common reasons include:

```text
The wrong file was modified.

The configuration file was not included correctly.

The nginx -t command failed.

The reload command did not execute successfully.

The requested Host does not match the configuration.

Another server was selected instead.

Browser caching.

There is CDN or SLB caching in front.
```

Troubleshooting steps:

```bash
nginx -t
```

```bash
nginx -T | grep -n "example.com"
```

```bash
grep -R "example.com" /etc/nginx/
```

```bash
systemctl status nginx
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1
```

---

## Scenario 32: Requests Do Not Reach Nginx

Troubleshooting steps:

```bash
ss -lntp | grep ':80'
```

```bash
curl -I http://127.0.0.1
```

```bash
tail -f /var/log/nginx/access.log
```

From a remote testing port:

```bash
nc -zv -w 2 Nginx server IP 80
```

For packet capture:

```bash
tcpdump -i any -nn port 80
```

Common causes include:

```text
The DNS could not resolve the machine's address.

The security group does not allow access.

The firewall blocks the connection.

Nginx is not listening on that port.

The request was directed to another entry point.

The load balancer did not forward it to this node.
```

---

## Scenario 33: Requests Reach Nginx but Not the Backend

Troubleshooting steps:

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
curl -v http://127.0.0.1:8080
```

```bash
nc -zv -w ```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## 23. One-Sentence Summary

The fundamental core of Nginx reverse proxy is:

```text
server
→ Receives requests based on port and domain name

location
→ Matches requests according to URI path

proxy_pass
→ Forwards requests to the backend service
```

The most important path rules are:

```text
proxy_pass without /
→ Usually retains the original URI

proxy_pass with /
→ Replaces the prefix matched by the location rule
```

Typical examples of differences include:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

```text
/api/users
→ Results in /api/users
```

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

```text
/api/users
→ Results in /users
```

The order of verification in production environment is:

```text
First, test the backend service.

Then, test the Nginx local interface.

Next, test the domain name.

Check access.log files.

Check error.log files.
```

Production recommendations include:

```text
Whether to use proxy_pass with / should be determined based on the actual backend routing.

Use the Host header during local testing.

Always back up configuration files before making changes.

Run nginx -t before reloading configurations.

Avoid putting all services in a single large configuration file.

Ensure that logging is enabled for reverse proxy settings.

For 502 errors, first check the backend port and Nginx error.log files.

For 404 errors, carefully examine the proxy_pass path rules and location matches.
```