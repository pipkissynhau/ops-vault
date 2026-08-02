# 02-Nginx Reverse Proxy Basics: server, location, and proxy_pass Path Rules

#Nginx #ReverseAgent #WebServer #AccessLayer #proxy_pass #server #location #Transport #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operations/02-Nginx Reverse Proxy Basics: server, location, and proxy_pass Path Rules.md

---

## I. Document Explanation

This document organizes basic Nginx reverse proxy configuration, focusing on the role and path forwarding rules of `server`, `location`, and `proxy_pass`.

Key points covered in this document:

- What is a reverse proxy
- Nginx's role in reverse proxy
- `server` basic configuration
- `server_name` domain matching
- `location` basic matching rules
- Difference between `location /` and `location /api/`
- `proxy_pass` basic usage
- Difference between `proxy_pass` with and without `/`
- Backend URI concatenation rules
- Multi-path forwarding
- Local interface verification
- Common reverse proxy errors
- Production configuration considerations

This document is the 02nd article in the Nginx Access Layer Operations series.

Document objectives:

```text
Can write basic reverse proxy configuration

→ I understand. server and location Role

→ I can read it. proxy_pass To where?

→ I got it. proxy_pass And... / And not. / Path differences

→ Yes. curl Validation of requests for entry Nginx

→ Can check the non-validation of fundamental reverse agents
```

---

## II. What is a Reverse Proxy

A reverse proxy can be understood as:

```text
Client does not directly access backend applications

Client first access Nginx

Nginx Then forward the request to the back end.
```

Typical flow:

```text
User Browser / Client

→ Nginx

→ Backend application services

→ Database / Redis / Other dependency
```

Example:

```text
User access:

http://example.com/api/users

Nginx Request received and forwarded to:

http://127.0.0.1:8080/api/users
```

Nginx's role here:

```text
Unified entrance

Hide Backend Real Address

Forward requests by domain name and path

Provision HTTPS Termination

Record Access Log

Basic access control

Simple limit.

Balance load
```

---

## III. Difference Between Reverse Proxy and Forward Proxy

---

## 1. Forward Proxy

A forward proxy proxies the client.

Common scenarios:

```text
Client access to external websites via proxy
```

Flow:

```text
Client

→ Proxy

→ Target website
```

The client knows it is using a proxy.

---

## 2. Reverse Proxy

A reverse proxy proxies the server.

Common scenarios:

```text
User access Nginx

Nginx Forward to Backend Application
```

Flow:

```text
Client

→ Nginx

→ Backend Services
```

Clients typically don't know the backend service's real address.

---

## 3. One-Sentence Difference

```text
I'm working on it.
→ Helping clients access people.

Reverse Agent
→ Help service providers receive user requests
```

---

## IV. Basic Structure of Nginx Reverse Proxy

A minimal reverse proxy configuration is as follows:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Meaning:

```text
listen 80
→ Nginx Listen 80 Port

server_name example.com
→ Match access domain names example.com

location /
→ Match All Paths

proxy_pass http://127.0.0.1:8080
→ Request for forwarding to this machine 8080 Port back-end service
```

Request flow:

```text
http://example.com/

→ Nginx:80

→ http://127.0.0.1:8080/
```

---

## V. Basic server Configuration

---

## Scenario 1: Minimal server Configuration

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
```

Verification:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Scenario 2: The Role of listen

```nginx
listen 80;
```

Indicates:

```text
Current server Listen 80 Port
```

Common syntax:

```nginx
listen 80;
```

```nginx
listen 443 ssl;
```

```nginx
listen 80 default_server;
```

Check listening ports:

```bash
ss -lntp | grep nginx
```

Check port 80:

```bash
ss -lntp | grep ':80'
```

Check port 443:

```bash
ss -lntp | grep ':443'
```

---

## Scenario 3: The Role of server_name

```nginx
server_name example.com;
```

Indicates:

```text
When Request Host Yes example.com When, match this. server
```

Multiple domains:

```nginx
server_name example.com www.example.com;
```

Wildcard domain:

```nginx
server_name *.example.com;
```

Default match:

```nginx
server_name _;
```

Test specific Host:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Scenario 4: Multiple server Configurations

Example:

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

server {
    listen 80;
    server_name admin.example.com;

    location / {
        proxy_pass http://127.0.0.1:9090;
    }
}
```

Meaning:

```text
Visits app.example.com
→ Forward to 127.0.0.1:8080

Visits admin.example.com
→ Forward to 127.0.0.1:9090
```

Local verification:

```bash
curl -I -H "Host: app.example.com" http://127.0.0.1
```

```bash
curl -I -H "Host: admin.example.com" http://127.0.0.1
```

---

## VI. Basic Understanding of location

`location` is used to match request URI.

Example:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

Indicates:

```text
Match All to / Initial request
```

For example:

```text
/

/api/users

/login

/static/app.js
```

All match `location /`.

---

## Scenario 5: location / Matches All Requests

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

Request forwarding:

```text
http://example.com/

→ http://127.0.0.1:8080/
```

```text
http://example.com/api/users

→ http://127.0.0.1:8080/api/users
```

---

## Scenario 6: location /api/ Matches API Requests

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Matches:

```text
/api/

/api/users

/api/v1/orders
```

Does not match:

```text
/login

/static/app.js

admin/api
```

---

## Scenario 7: Multiple locations

Example:

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

Meaning:

```text
/api/ Initial request
→ 127.0.0.1:8080

/admin/ Initial request
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

## VII. Common location Matching Types

Common Nginx location syntax:

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

Features:

```text
Press URI Prefix Match

Match /api/ Initial request
```

---

## Scenario 9: Exact Match =

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
If it matches /static/ The prefix will no longer match the rule. location
```

Suitable for:

```text
Static Resource Directory

Upload File Directory

I don't want to be regular. location Interference Path
```

---

## Scenario 11: Case-sensitive Regular Expression Match ~

```nginx
location ~ \.php$ {
    proxy_pass http://127.0.0.1:9000;
}
```

Indicates:

```text
Match to .php The end. URI

Case sensitive
```

---

## Scenario 12: Case-insensitive Regular Expression Match ~*

```nginx
location ~* \.(jpg|jpeg|png|gif|css|js)$ {
    root /data/www;
}
```

Indicates:

```text
Match common static resource suffix

Case sensitive
```

---

## VIII. Simplified Understanding of location Matching Priority

Common priority can be simply remembered as:

```text
Exact Match =
→ Highest priority

^~ Prefix Match
→ No more regulars after a hit.

The right match. ~ / ~*
→ Match in configuration order

Normal prefix matching
→ Select Maximum Match
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

Request:

```text
/api/login
→ Hit. location = /api/login

/api/users
→ Hit. location /api/

/index.html
→ Hit. location /
```

---

## IX. Basic Usage of proxy_pass

`proxy_pass` is used to forward requests to backend services.

Common syntax:

```nginx
proxy_pass http://127.0.0.1:8080;
```

It can also forward to a domain:

```nginx
proxy_pass http://backend.example.com;
```

It can also forward to an upstream:

```nginx
proxy_pass http://app_backend;
```

---

## Scenario 13: Proxying to Local Backend Service

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

Flow:

```text
Client

→ Nginx:80

→ 127.0.0.1:8080
```

Test backend:

```bash
curl -I http://127.0.0.1:8080
```

Test Nginx:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Scenario 14: Proxying to Internal Backend Service

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://10.0.0.21:8080;
    }
}
```

First test Nginx to backend network:

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
curl -I http://10.0.0.21:8080
```

Then test Nginx entry:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## X. Core of proxy_pass Path Rules

`proxy_pass` The most common mistake is:

```text
location Path behind

and

proxy_pass Did you bring it back? /
```

Core is divided into two categories:

```text
proxy_pass No, I don't. URI
→ Keep original URI

proxy_pass And... URI
→ Use proxy_pass Back. URI Replace location Match Part
```

Here, URI can be simply understood as:

```text
Path part behind domain name or port
```

For example:

```text
http://127.0.0.1:8080
→ No, I don't. URI

http://127.0.0.1:8080/
→ And... URII don't know.URI Yes. /

http://127.0.0.1:8080/backend/
→ And... URII don't know.URI Yes. /backend/
```

---

## 11. Case where proxy_pass does not include a slash

---

## Scenario 15: location /api/ + proxy_pass without a slash

Configuration:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

Request:

```text
/api/users
```

Forwarded to backend:

```text
http://127.0.0.1:8080/api/users
```

Explanation:

```text
proxy_pass No, I don't. URI

Nginx They'll turn the original. URI Forward original to Backend
```

---

## Scenario 16: location / + proxy_pass without a slash

Configuration:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

Request:

```text
/api/users
```

Forwarded to backend:

```text
http://127.0.0.1:8080/api/users
```

Explanation:

```text
location / Match all requests

proxy_pass No, I don't. URI

Original path remains unchanged
```

---

## 12. Case where proxy_pass includes a slash

---

## Scenario 17: location /api/ + proxy_pass with a slash

Configuration:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

Request:

```text
/api/users
```

Forwarded to backend:

```text
http://127.0.0.1:8080/users
```

Explanation:

```text
location Matches /api/ Replaced with proxy_pass Medium /
```

Result is:

```text
/api/users

♪ Turn into ♪

/users
```

---

## Scenario 18: location /api/ + proxy_pass with /backend/

Configuration:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/backend/;
}
```

Request:

```text
/api/users
```

Forwarded to backend:

```text
http://127.0.0.1:8080/backend/users
```

Explanation:

```text
location Matches /api/ Replaced with /backend/
```

---

## 13. Proxy_pass Path Rule Comparison Table

| Configuration | Request Path | Backend Received Path | Notes |
|---|---|---|---|
| `location /api/ { proxy_pass http://127.0.0.1:8080; }` | `/api/users` | `/api/users` | Preserve original URI |
| `location /api/ { proxy_pass http://127.0.0.1:8080/; }` | `/api/users` | `/users` | Remove `/api/` prefix |
| `location /api/ { proxy_pass http://127.0.0.1:8080/backend/; }` | `/api/users` | `/backend/users` | Replace `/api/` with `/backend/` |
| `location / { proxy_pass http://127.0.0.1:8080; }` | `/api/users` | `/api/users` | Preserve original URI |
| `location / { proxy_pass http://127.0.0.1:8080/; }` | `/api/users` | `/api/users` | `location /` typically shows no significant difference |

---

## 14. Two Most Confusing Configurations

---

## Scenario 19: Forward with /api prefix preserved

Requirement:

```text
User access /api/users

And backend. /api/users
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

Backend received:

```text
/api/users
```

---

## Scenario 20: Forward with /api prefix removed

Requirement:

```text
User access /api/users

Backend Receiving /users
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

Backend received:

```text
/users
```

---

## 15. Production Recommendations for Proxy_pass Path Rules

Recommend to first clarify the actual routing of the backend service.

If the backend routing is:

```text
/api/users
```

Nginx should be configured as:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

If the backend routing is:

```text
/users
```

Nginx should be configured as:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

Production recommendations:

```text
Don't guess.

First straight. curl Backend Confirm Backend Real Path

Then decide. proxy_pass Whether to bring /
```

Test the actual path of the backend:

```bash
curl -v http://127.0.0.1:8080/api/users
```

```bash
curl -v http://127.0.0.1:8080/users
```

---

## 16. Multi-path Reverse Proxy Example

---

## Scenario 21: Frontend and Backend Separation

Requirement:

```text
/        → Front-end services 127.0.0.1:3000
/api/    → Backend API 127.0.0.1:8080
/admin/  → Manage backstage 127.0.0.1:9090
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

## Scenario 22: Multiple Domains Forwarding Different Services

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
```

```bash
curl -I -H "Host: web.example.com" http://127.0.0.1
```

---

## 17. Common Headers for Reverse Proxy

Basic reverse proxy typically recommends adding header transparency.

Example:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Meaning:

```text
Host
→ Sprawl raw Host

X-Real-IP
→ Direct access to records Nginx Client IP

X-Forwarded-For
→ Records proxy links IP

X-Forwarded-Proto
→ Record original request protocol http or https
```

Explanation:

```text
Header It's going to be in. 04 Details in Production Agent Parameters

Real IP It'll be here. 07 It's true. IP Organisation
```

---

## 18. Complete Basic Reverse Proxy Example

Configuration file:

```bash
vi /etc/nginx/conf.d/example.com.conf
```

Content:

```nginx
server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Check:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Verification:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/api/health
```

Check access log:

```bash
tail -f /var/log/nginx/example.access.log
```

Check error log:

```bash
tail -f /var/log/nginx/example.error.log
```

---

## 19. Reverse Proxy Verification Process

After production configuration of reverse proxy, it is recommended to verify in sequence.

---

## Scenario 23: Confirm Backend Service is Normal

```bash
ss -lntp | grep ':8080'
```

```bash
curl -I http://127.0.0.1:8080
```

```bash
curl -v http://127.0.0.1:8080/api/health
```

If the backend is on another machine:

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
curl -I http://10.0.0.21:8080
```

---

## Scenario 24: Confirm Nginx Configuration Syntax

```bash
nginx -t
```

---

## Scenario 25: Confirm Complete Configuration is Loaded

```bash
nginx -T | grep -n "server_name example.com"
```

```bash
nginx -T | grep -n "proxy_pass"
```

---

## Scenario 26: Verify via Host on Local Machine

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

---

## Scenario 27: Check access.log for Requests

```bash
tail -f /var/log/nginx/access.log
```

Or business-specific logs:

```bash
tail -f /var/log/nginx/example.access.log
```

---

## Scenario 28: Check error.log for Errors

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

---

## 20. Common Issue Troubleshooting

---

## Scenario 29: Access Returns 502

502 typically indicates:

```text
Nginx Could not get a response from the back end normally
```

Common causes:

```text
Backend service not started.

Backend is not listening.

proxy_pass Chile

Nginx Backend's dead.

Backend Active Close Connection

Backend Process Abnormal

Backend returns illegal response
```

Troubleshoot:

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

## Scenario 30: Access Returns 404

Possible causes:

```text
Request not forwarded to expected backend

proxy_pass Pathbelt / Or not. / Wrong

The back end does not actually have that route.

Wrong shot. location

Default hit. server

Static directory does not exist
```

Troubleshoot:

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
curl -v http://127.0.0.1:8080/users
```

```bash
tail -n 100 /var/log/nginx/access.log
```

---

## Scenario 31: Configuration Changes Not Taking Effect

Common causes:

```text
Error file modified

Profile was not used include

nginx -t Failed

reload Not implemented successfully

Request Host Do not match

It hit the others. server

Browser Cache

There's more ahead. CDN / SLB Cache
```

Troubleshoot:

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

## Scenario 32: Request Not Reaching Nginx

Troubleshoot:

```bash
ss -lntp | grep ':80'
```

```bash
curl -I http://127.0.0.1
```

```bash
tail -f /var/log/nginx/access.log
```

Test port from remote:

```bash
nc -zv -w 2 NginxServersIP 80
```

Packet capture:

```bash
tcpdump -i any -nn port 80
```

Common causes:

```text
DNS It didn't solve the machine.

Security's not clear.

Firewall's off.

Nginx No listening.

Request to the other entrance.

Load balance not forwarded to this node
```

---

## Scenario 33: Request Reaches Nginx but Not Backend

Troubleshooting:

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
nc -zv -w 2 127.0.0.1 8080
```

If the backend is on another host:

```bash
tcpdump -i any -nn host BackendIP and port Backend
```

---

## 21. Production Precautions

---

## 1. Whether proxy_pass includes / must be explicitly defined

Confirm before deployment:

```text
Whether the backend needs to be preserved /api Prefix

It still needs to be removed. /api Prefix
```

Do not rely on guessing.

---

## 2. Test the backend directly first, then test Nginx

Recommended order:

```text
curl Backend Address

→ curl Nginx This is the entrance.

→ curl Domain Name Entry

→ Look. access.log

→ Look. error.log
```

---

## 3. Must back up configuration before making changes

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

---

## 4. Must run nginx -t before reload

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## 5. Do not mix multiple services in a single large server

Recommendation:

```text
One business, one business. conf Documentation

One domain name. server

Generic Configuration Separate include

It's easy to roll back and check.
```

---

## 6. Reverse proxy configuration must retain access logs

Access logs are used to determine:

```text
Request arrival

What's the status code?

What's the access path?

Source IP Who is it?

Whether or not to hit the target. server
```

---

## 7. Host matching is extremely important

Do not use only:

```bash
curl http://127.0.0.1
```

when testing locally. More recommended:

```bash
curl -H "Host: example.com" http://127.0.0.1
```

Otherwise may hit the default server.

---

## 22. Summary of Common Commands in This Article

---

## Configuration Editing

```bash
vi /etc/nginx/conf.d/example.com.conf
```

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
nginx -T | grep -n "proxy_pass"
```

```bash
nginx -T | grep -n "location"
```

---

## Reload Service

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

---

## Port Check

```bash
ss -lntp | grep nginx
```

```bash
ss -lntp | grep ':80'
```

```bash
ss -lntp | grep ':443'
```

```bash
ss -lntp | grep ':8080'
```

---

## Backend Check

```bash
curl -I http://127.0.0.1:8080
```

```bash
curl -v http://127.0.0.1:8080/api/health
```

```bash
nc -zv -w 2 127.0.0.1 8080
```

```bash
nc -zv -w 2 10.0.0.21 8080
```

---

## Nginx Entry Validation

```bash
curl -I http://127.0.0.1
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

---

## Log Viewing

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "connect() failed" /var/log/nginx/error.log | tail -n 50
```

---

## Packet Capture Verification

```bash
tcpdump -i any -nn port 80
```

```bash
tcpdump -i any -nn host BackendIP and port Backend
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

## 23. One-Sentence Summary

The core of Nginx reverse proxy basics is:

```text
server
→ Receive requests by port and domain

location
→ Press URI Path Matching Request

proxy_pass
→ Forward requests to back-end services
```

The most important path rules:

```text
proxy_pass No, I don't. /
→ Usually keep original URI

proxy_pass And... /
→ Will replace location Matched Prefix
```

Typical differences:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

```text
/api/users
→ /api/users
```

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

```text
/api/users
→ /users
```

Production verification order:

```text
Check backend first

→ Retest Nginx This is the entrance.

→ Retest Domain Name

→ Look. access.log

→ Look. error.log
```

Production recommendations:

```text
proxy_pass Whether to bring / It has to be determined by the real path in the back.

We're going to take the test. Host

Backup before changing configuration

reload I have to. nginx -t

Do not write all business into a huge configuration.

Reverse proxy configuration must keep logs

502 Prioritize backends and Nginx error.log

404 We need to focus. proxy_pass Path rules and location Match
```