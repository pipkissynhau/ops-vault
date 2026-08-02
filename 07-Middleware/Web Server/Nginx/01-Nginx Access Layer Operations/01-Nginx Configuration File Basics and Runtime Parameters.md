# 01-Nginx Configuration File Basics and Runtime Parameters

#Nginx #WebServer #ReverseAgent #AccessLayer #Linux #Middle #Transport #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operations/01-Nginx Configuration File Basics and Runtime Parameters.md

---

## I. Document Description

This document organizes the most fundamental and commonly used configuration file structures and runtime parameters for Nginx access layer operations.

This article focuses on:

- Nginx's position in production
- Common Nginx installation paths
- Nginx main configuration file structure
- `main` Global configuration block
- `events` Event model configuration block
- `http` HTTP configuration block
- `server` Virtual host configuration block
- `location` Route matching configuration block
- `include` Configuration splitting method
- Nginx master / worker process model
- Common runtime commands
- Configuration checking and reloading
- Viewing compilation parameters
- Viewing listening ports
- Viewing log paths
- Configuration change workflow before and after
- Production environment precautions

This article is the 01st article of the Nginx access layer operations series.

This article's objectives:

```text
I can read it. nginx.conf Basic structure

→ I know. Nginx How many layers to configure

→ You know what it means to run a common running parameter?

→ Can secure the configuration.

→ Correct. reload Nginx

→ You know what you should do before and after the configuration changes.
```

---

## II. Nginx's Position in Production

Common roles of Nginx include:

```text
Web Static Resource Server

Countered Proxy

Seven Layer Load Balance Entry

HTTPS Termination Layer

Unified Domain Name Entry

API Access Layer

Separate the entrance back and forth.

Ingress Controller One of the bottom ones.

Gray Distribution Portal

Access log collection portal

Simple flow limit and access control entrance
```

In production architecture, Nginx typically resides in:

```text
User / Client

→ CDN / WAF / SLB

→ Nginx

→ Backend application services

→ Database / Cache / Message queue
```

One-sentence understanding:

```text
Nginx It is an important access layer component before business flows enter back-end services.
```

---

## III. Common Nginx Directory Structure

Paths may vary depending on installation methods.

Common paths are as follows:

```text
/etc/nginx/nginx.conf
→ Main Profile

/etc/nginx/conf.d/
→ Subconfig directory, often for business server Configure

/etc/nginx/sites-enabled/
→ Debian / Ubuntu Common site enabled directory

/etc/nginx/sites-available/
→ Debian / Ubuntu Common Site Available Directory

/var/log/nginx/access.log
→ Default Access Log

/var/log/nginx/error.log
→ Default Error Log

/usr/share/nginx/html
→ Default static file directory

/var/run/nginx.pid
→ Nginx Process PID Documentation

/run/nginx.pid
→ Some systems. PID Documentation
```

Viewing Nginx-related paths:

```bash
whereis nginx
```

Viewing main configuration file:

```bash
ls -lh /etc/nginx/nginx.conf
```

Viewing configuration directory:

```bash
ls -lh /etc/nginx/
```

Viewing sub-configuration directory:

```bash
ls -lh /etc/nginx/conf.d/
```

Viewing log directory:

§
§code_9§§

---

## IV. Viewing Nginx Version and Compilation Parameters

---

## Scenario 1: Viewing Nginx Version

```bash
nginx -v
```

Example output:

```text
nginx version: nginx/1.24.0
```

---

## Scenario 2: Viewing Nginx Compilation Parameters

```bash
nginx -V
```

This command will output:

```text
Nginx Version

OpenSSL Version

Compile module

Profile Path

Log Path

pid File Path

Temporary Directory Path
```

Common focus areas:

```text
--conf-path
→ Main Profile Path

--error-log-path
→ Error Log Path

--http-log-path
→ Access Log Path

--pid-path
→ PID File Path

--with-http_ssl_module
→ Supported HTTPS

--with-http_stub_status_module
→ Supported stub_status

--with-http_gzip_static_module
→ Supported gzip_static
```

Viewing compilation parameters and formatting:

```bash
nginx -V 2>&1 | tr ' ' '\n'
```

Filtering configuration file paths:

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep conf-path
```

Filtering SSL module:

```bash
nginx -V 2>&1 | grep http_ssl_module
```

---

## V. Basic Structure of Nginx Main Configuration File

Typical structure:

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    access_log /var/log/nginx/access.log;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

Nginx configurations are typically divided into several layers:

```text
main
→ Global Configuration

events
→ Event Model Configuration

http
→ HTTP Global Configuration

server
→ Virtual Host Configuration

location
→ URI Route Match Configuration

upstream
→ Backend service pool configuration
```

Hierarchy relationship:

```text
main
├── events
└── http
    ├── upstream
    ├── server
    │   └── location
    └── server
        └── location
```

---

## VI. main Global Configuration Block

`main` is the outermost configuration, not located in any `{}`.

Common configurations:

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;
```

---

## Scenario 3: user

```nginx
user nginx;
```

Purpose:

```text
Assign worker Process running user
```

Common users:

```text
nginx

www-data

www

nobody
```

Viewing Nginx process user:

```bash
ps -ef | grep nginx | grep -v grep
```

Explanation:

```text
master Process is usually by root Start

worker Process is usually by nginx or www-data Wait for low permission users to run
```

Production recommendation:

```text
Don't let worker Long-term process root Organisation

Static directories, log directories, upload directories permissions and Nginx worker User Match
```

---

## Scenario 4: worker_processes

```nginx
worker_processes auto;
```

Purpose:

```text
Assign worker Number of processes
```

Common configuration:

```nginx
worker_processes auto;
```

Or:

```nginx
worker_processes 4;
```

Recommendation:

```text
Most scenes used auto

auto Yes. CPU Core Auto Settings worker Number
```

Viewing CPU core count:

```bash
nproc
```

Viewing Nginx worker count:

```bash
ps -ef | grep "nginx: worker" | grep -v grep | wc -l
```

---

## Scenario 5: worker_rlimit_nofile

```nginx
worker_rlimit_nofile 65535;
```

Purpose:

```text
Settings worker Maximum number of files open by process
```

Nginx may consume file descriptors for each connection, log file, and static file.

Viewing system current limits:

```bash
ulimit -n
```

Viewing Nginx process limits:

```bash
cat /proc/$(pgrep -o nginx)/limits | grep "Max open files"
```

Production note:

```text
worker_rlimit_nofile And the system. limits Cooperation

Change only Nginx It may not be enough.

And pay attention. systemd LimitNOFILE
```

Viewing systemd configuration:

```bash
systemctl cat nginx
```

---

## Scenario 6: error_log

```nginx
error_log /var/log/nginx/error.log warn;
```

Purpose:

```text
Configure Nginx Error log path and log level
```

Common levels:

```text
debug

info

notice

warn

error

crit

alert

emerg
```

Production common:

```nginx
error_log /var/log/nginx/error.log warn;
```

Viewing error log:

```bash
tail -n 100 /var/log/nginx/error.log
```

Real-time viewing of error log:

```bash
tail -f /var/log/nginx/error.log
```

---

## Scenario 7: pid

```nginx
pid /run/nginx.pid;
```

Purpose:

```text
Assign Nginx master Process PID File Path
```

Viewing PID file:

```bash
cat /run/nginx.pid
```

Or:

```bash
cat /var/run/nginx.pid
```

Viewing master process:

```bash
ps -fp $(cat /run/nginx.pid)
```

---

## VII. events Configuration Block

`events` configures Nginx's event handling model.

Common configuration:

```nginx
events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}
```

---

## Scenario 8: worker_connections

```nginx
worker_connections 1024;
```

Purpose:

```text
Each worker Maximum number of connections allowed by process
```

Theoretical maximum connection count approximation:

```text
worker_processes × worker_connections
```

Example:

```text
worker_processes = 4

worker_connections = 1024

The theoretical maximum number of connections is about 4096
```

Note:

```text
The actual number of connections available will also be influenced by document descriptors, nuclear parameters in the system, upstream connections, log files, etc.
```

Viewing current configuration:

```bash
nginx -T | grep worker_connections
```

---

## Scenario 9: use epoll

```nginx
use epoll;
```

Explanation:

```text
Linux Common epoll Event Model
```

Most modern Nginx instances automatically select appropriate event models and typically do not require manual configuration.

Viewing system:

```bash
uname -a
```

---

## Scenario 10: multi_accept

```nginx
multi_accept on;
```

Purpose:

```text
Allow worker Take as many new connections as possible at once
```

Explanation:

```text
High-symmetry may help.

Normal scenes don't have to be configured.

Need to be judged in conjunction with actual pressure and connection models
```

---

## VIII. http Configuration Block

`http` block is the core configuration area for Nginx handling HTTP requests.

Common configuration:

```nginx
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;
    server_tokens off;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## Scenario 11: include mime.types

```nginx
include /etc/nginx/mime.types;
```

Purpose:

```text
Load File Extensions and Content-Type Map relation
```

Example:

```text
.html
→ text/html

.css
→ text/css

.js
→ application/javascript

.png
→ image/png
```

If mime.types is not loaded, static resources may return incorrect Content-Type.

---

## Scenario 12: default_type

```nginx
default_type application/octet-stream;
```

Purpose:

```text
Use default when file type cannot be identified Content-Type
```

Common configuration:

```nginx
default_type application/octet-stream;
```

---

## Scenario 13: access_log

```nginx
access_log /var/log/nginx/access.log main;
```

Purpose:

```text
Configure access log paths and log formats
```

Viewing access log:

```bash
tail -n 100 /var/log/nginx/access.log
```

Real-time viewing of access log:

```bash
tail -f /var/log/nginx/access.log
```

Disabling access log for a specific area:

```nginx
access_log off;
```

Production note:

```text
Do not turn off the production access log at random

If access is particularly large, it should be integrated with log platform, sampling, filtering, disk capacity
```

---

## Scenario 14: log_format

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';
```

Purpose:

```text
Definitions access_log Output Format
```

Common fields:

```text
$remote_addr
→ Client IP

$time_local
→ Local Time

$request
→ Full Request Line

$status
→ HTTP Status Code

$body_bytes_sent
→ Response bytes

$http_referer
→ Referer

$http_user_agent
→ User-Agent
```

Viewing log_format in configuration:

```bash
grep -R "log_format" /etc/nginx/
```

Check which format is used by access_log:

```bash
grep -R "access_log" /etc/nginx/
```

---

## Scenario 15: sendfile

```nginx
sendfile on;
```

Purpose:

```text
Enable efficient file transfer
```

Suitable for:

```text
Static File Downloads

Pictures,CSSI don't know.JS Still resources

Large file transfer
```

Common production configuration:

```nginx
sendfile on;
```

---

## Scenario 16: keepalive_timeout

```nginx
keepalive_timeout 65;
```

Purpose:

```text
Configure client long connection maintenance time
```

Notes:

```text
Shorter may lead to frequent connections

Too long may take over the connection resources

Need to adapt to business access models
```

---

## Scenario 17: server_tokens

```nginx
server_tokens off;
```

Purpose:

```text
Hide Nginx Version Number
```

When enabled, response headers may expose:

```text
Server: nginx/1.24.0
```

After disabling, it generally only shows:

```text
Server: nginx
```

Production recommendation:

```nginx
server_tokens off;
```

---

## Scenario 18: client_max_body_size

```nginx
client_max_body_size 50m;
```

Purpose:

```text
Limit client request size
```

Common scenarios:

```text
Uploading files

Image Upload

Interface submission large JSON

Import File
```

If the request body exceeds the limit, it may return:

```text
413 Request Entity Too Large
```

Check error logs:

```bash
grep -i "client intended to send too large body" /var/log/nginx/error.log
```

---

## Section 9: server Configuration Block

`server` represents a virtual host.

Basic example:

```nginx
server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/example.access.log main;
    error_log  /var/log/nginx/example.error.log warn;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

Common configuration items:

```text
listen
→ Listen Port

server_name
→ Match Domain Name

access_log
→ Current Virtual Host Access Log

error_log
→ Current Virtual Host Error Log

location
→ URI Match Rules
```

---

## Scenario 19: listen

```nginx
listen 80;
```

Purpose:

```text
Listen 80 Port
```

Listen for HTTPS:

```nginx
listen 443 ssl;
```

Listen for IPv6:

```nginx
listen [::]:80;
```

Check listening ports:

```bash
ss -tunlp | grep nginx
```

Or:

```bash
ss -lntp | grep ':80'
```

---

## Scenario 20: server_name

```nginx
server_name example.com;
```

Purpose:

```text
As requested Host Match Virtual Host
```

Multiple domains:

```nginx
server_name example.com www.example.com;
```

Wildcard:

```nginx
server_name *.example.com;
```

Default server:

```nginx
server {
    listen 80 default_server;
    server_name _;
}
```

Check request Host:

```bash
curl -H "Host: example.com" http://127.0.0.1/
```

---

## Section 10: Basic location Configuration Block

`location` is used to match request URIs.

Basic example:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
    }
}
```

Common purposes:

```text
Forward different paths to different backends

Different paths use different static directories

Different path configuration different caches

Different path configuration different permission controls

Different paths configure different log policies
```

Detailed `location` matching rules and `proxy_pass` path rules are organized separately in Chapter 02.

---

## Section 11: Configuration Splitting with include

In production, it's not recommended to put all business configurations in `nginx.conf`.

Common approach:

```nginx
include /etc/nginx/conf.d/*.conf;
```

Notes:

```text
Nginx Load /etc/nginx/conf.d/ All .conf Documentation
```

Check include:

```bash
grep -R "include" /etc/nginx/nginx.conf
```

Check sub-configurations:

```bash
ls -lh /etc/nginx/conf.d/
```

Create business configurations:

```bash
vi /etc/nginx/conf.d/example.com.conf
```

Example:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

---

## Section 12: Nginx master/worker Process Model

Nginx typically contains:

```text
master process

worker process
```

Check processes:

```bash
ps -ef | grep nginx | grep -v grep
```

Example:

```text
root      1000     1  0 10:00 ?  00:00:00 nginx: master process nginx
nginx     1001  1000  0 10:00 ?  00:00:00 nginx: worker process
nginx     1002  1000  0 10:00 ?  00:00:00 nginx: worker process
```

Notes:

```text
master
→ Read Configuration, Manage workerGet the signal.

worker
→ Processing actual client requests
```

Common operations:

```text
reload
→ master Reload configuration, smooth replacement worker

restart
→ Stop and start again.

quit
→ An elegant exit.

stop
→ Quick Stop
```

---

## Section 13: Common Nginx Runtime Commands

---

## Scenario 21: Start Nginx

systemd management:

```bash
systemctl start nginx
```

Direct command:

```bash
nginx
```

---

## Scenario 22: Stop Nginx

```bash
systemctl stop nginx
```

Or:

```bash
nginx -s stop
```

Notes:

```text
stop
→ Quick Stop
```

---

## Scenario 23: Graceful Shutdown of Nginx

```bash
nginx -s quit
```

Notes:

```text
quit
→ Exit when the current connection process is complete
```

---

## Scenario 24: Reload Nginx

Recommended:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

Or:

```bash
nginx -s reload
```

Notes:

```text
reload Reload configuration

Usually compared to restart Smoother

Pre-use in production reloadNot directly. restart
```

---

## Scenario 25: Restart Nginx

```bash
systemctl restart nginx
```

Notes:

```text
restart It will stop and start again.

That's right. reload The impact is greater.

Careful use of the production environment
```

Suitable for:

```text
It's a service anomaly.

reload Invalid

Upgrade Nginx

Special module or system level configuration changes
```

---

## Scenario 26: Reopen Log Files

```bash
nginx -s reopen
```

Purpose:

```text
Announcements Nginx Reopen Log File
```

Common scenarios:

```text
Log after cut

logrotate After implementation

Manual Move Log After
```

---

## Section 14: Configuration Check and Full Configuration Output

---

## Scenario 27: Check Configuration Syntax

```bash
nginx -t
```

Normal output similar to:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If configuration errors exist, it will prompt:

```text
Error File

Error Line Number

Error Reason
```

After modifying configurations in production, it must be executed:

```bash
nginx -t
```

---

## Scenario 28: Output Full Effective Configuration

```bash
nginx -T
```

Purpose:

```text
Output Nginx Full configuration after loading

Including include Include Subconfigation
```

Suitable for:

```text
Check if the configuration is being include

Check for the same name server

Check. location Error Document

Check if configuration is actually effective

Backup Current Full Configuration
```

Save full configuration:

```bash
nginx -T > /tmp/nginx-full-config-$(date +%F-%H%M%S).txt 2>&1
```

---

## Scenario 29: Locate Configuration by Error Line

If `nginx -t` indicates:

```text
nginx: [emerg] invalid number of arguments in "proxy_pass" directive in /etc/nginx/conf.d/app.conf:15
```

Check nearby lines:

```bash
sed -n '10,20p' /etc/nginx/conf.d/app.conf
```

---

## Section 15: Check Listening Ports and Request Validation

---

## Scenario 30: Check if Nginx is Listening on a Port

```bash
ss -tunlp | grep nginx
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

## Scenario 31: Test HTTP Locally

```bash
curl -I http://127.0.0.1
```

Specify Host:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

Access specified path:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

---

## Scenario 32: Test Port Remotely

```bash
nc -zv -w 2 ObjectiveIP 80
```

```bash
nc -zv -w 2 ObjectiveIP 443
```

---

## Section 16: Nginx Log Viewing

---

## Scenario 33: View Access Logs

```bash
tail -n 100 /var/log/nginx/access.log
```

Real-time viewing:

```bash
tail -f /var/log/nginx/access.log
```

Statistical status codes:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

---

## Scenario 34: View Error Logs

```bash
tail -n 100 /var/log/nginx/error.log
```

Real-time viewing:

```bash
tail -f /var/log/nginx/error.log
```

Check upstream errors:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

Check configuration-related errors:

```bash
grep -i "emerg" /var/log/nginx/error.log | tail -n 100
```

---

## Section 17: Standard Configuration Change Process

When modifying Nginx configurations in production environments, it's recommended to follow:

```text
Identification of needs

→ Backup Configuration

→ Modify Configuration

→ nginx -t Inspection

→ Here. curl Authentication

→ reload

→ Again. curl Authentication

→ View access.log / error.log

→ Observation of operational indicators

→ Roll back as necessary
```

---

## Scenario 35: Backup Single Configuration File

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

---

## Scenario 36: Backup Entire Configuration Directory

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

Check backup:

```bash
ls -lh /tmp/nginx-conf-backup-*.tar.gz
```

---

## Scenario 37: Check After Modification

```bash
nginx -t
```

---

## Scenario 38: Reload After Verification

```bash
systemctl reload nginx
```

---

## Scenario 39: Verify After Reload

Check service status:

```bash
systemctl status nginx
```

Check ports:

§

## Scenario 40: Rollback Configuration

If anomalies occur after changes, you can restore the backup:

```bash
cp -a /etc/nginx/conf.d/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

Check Configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Verify:

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

---

## Eighteen, Common Basic Issues

---

## Scenario 41: nginx -t Failed

Troubleshoot:

```bash
nginx -t
```

Check the error file and line number:

```bash
sed -n 'Start Line,End Linep' Profile
```

Common causes:

```text
We're missing the semicolon. ;

Unmatched parenthesis

Command error

Invalid number of parameters

include File does not exist

Certificate path error

Log directory does not exist

Configure in Wrong Context
```

---

## Scenario 42: reload Failed

Troubleshoot:

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
nginx -t
```

```bash
tail -n 100 /var/log/nginx/error.log
```

Common causes:

```text
Configure Syntax Error

Port occupied

Certificate file does not exist

Bad certificate permission

Log Directory Invalid

PID File anomaly
```

---

## Scenario 43: Configuration Not Taking Effect After Modification

Troubleshoot:

```bash
nginx -T | grep -n "server_name"
```

```bash
nginx -T | grep -n "example.com"
```

```bash
grep -R "example.com" /etc/nginx/
```

```bash
systemctl reload nginx
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

Common causes:

```text
Error profile modified

Profile was not used include

nginx -t It didn't pass, so... reload Failed

Browser Cache

Request Host Do not match

Hit! default_server

More than one. server_name Repeat Configuration

There's more up there. CDN / SLB Cache or forward
```

---

## Nineteen, Production Notes

---

## 1. Must Backup Before Modification

Recommended:

```bash
cp -a Profile Profile.$(date +%F-%H%M%S).bak
```

Or:

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## 2. Must Run nginx -t Before reload

Do not directly:

```bash
systemctl reload nginx
```

Recommended:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## 3. Prioritize reload Over restart in Production

```text
reload
→ Smooth Load Configuration with Less Impact

restart
→ Stop and start again.
```

---

## 4. Do Not Put All Business into nginx.conf

Recommended:

```text
nginx.conf
→ Global Configuration

conf.d/*.conf
→ Business. server Configure
```

---

## 5. Do Not Disable access_log Arbitrarily

Access logs are used for:

```text
Check if the request has arrived.

Analyse Status Code

Analysis 5xx

Analytical visit sources

Audit visit conduct

Access log platform
```

If logs are too large, use:

```text
Log Round

Log Sample

Log Platform

Filter low value requests

Disk Capacity Management
```

Instead of simply disabling.

---

## 6. Do Not Expose Version Number

Recommended:

```nginx
server_tokens off;
```

---

## 7. Configuration Files Should Be Version Controlled

Production recommendation:

```text
Nginx Configure Entry Git

Before Change review

Changed records

Support rapid rollback

To avoid multiple manual changes.
```

---

## Twenty, Common Commands Summary

---

## Version and Compile Parameters

```bash
nginx -v
```

```bash
nginx -V
```

```bash
nginx -V 2>&1 | tr ' ' '\n'
```

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep conf-path
```

```bash
nginx -V 2>&1 | grep http_ssl_module
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
nginx -T > /tmp/nginx-full-config-$(date +%F-%H%M%S).txt 2>&1
```

---

## systemctl Management

```bash
systemctl start nginx
```

```bash
systemctl stop nginx
```

```bash
systemctl reload nginx
```

```bash
systemctl restart nginx
```

```bash
systemctl status nginx
```

```bash
systemctl enable nginx
```

---

## nginx Signal Commands

```bash
nginx -s reload
```

```bash
nginx -s stop
```

```bash
nginx -s quit
```

```bash
nginx -s reopen
```

---

## Processes and Ports

```bash
ps -ef | grep nginx | grep -v grep
```

```bash
ps -ef | grep "nginx: worker" | grep -v grep | wc -l
```

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

## Configuration File Inspection

```bash
ls -lh /etc/nginx/
```

```bash
ls -lh /etc/nginx/conf.d/
```

```bash
grep -R "include" /etc/nginx/nginx.conf
```

```bash
grep -R "server_name" /etc/nginx/
```

```bash
grep -R "log_format" /etc/nginx/
```

```bash
grep -R "access_log" /etc/nginx/
```

---

## Log Inspection

```bash
tail -n 100 /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

---

## Request Validation

```bash
curl -I http://127.0.0.1
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

```bash
nc -zv -w 2 ObjectiveIP 80
```

```bash
nc -zv -w 2 ObjectiveIP 443
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
ls -lh /tmp/nginx-conf-backup-*.tar.gz
```

---

## Twenty-one, One-sentence Summary

The core of Nginx configuration basics is:

```text
main
→ Global running parameters

events
→ Connection and Event Model

http
→ HTTP Global Configuration

server
→ A virtual host.

location
→ One. URI Match Rules

upstream
→ A set of backend services
```

Common operation commands:

```text
nginx -t
→ Check Configuration

nginx -T
→ Output Full Configuration

systemctl reload nginx
→ Smooth Reload

systemctl restart nginx
→ Restart Service

nginx -s reopen
→ Reopen Log
```

Production change process:

```text
Backup Configuration

→ Modify Configuration

→ nginx -t

→ curl Organisation

→ reload

→ View Log

→ Observation operations

→ Roll back as necessary
```

Production recommendations:

```text
Don't just fix it. reload

Don't reconfigure without backup.

Don't write all business. nginx.conf

Don't be silly. restart

Don't just close it. access_log

Don't be exposed. Nginx Version Number

Profile to be included in version management

The request and error log must be verified after the change
```