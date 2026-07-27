# 01-Nginx Configuration File Basics and Running Parameters

#Nginx #Web Server #Reverse Proxy #Access Layer #Linux #Middleware #Ops #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Ops/01-Nginx Configuration File Basics and Running Parameters.md

---

## I. Document Description

This article outlines the most fundamental and commonly used configuration file structures and running parameters in Nginx access layer operations.

Key points include:

- The role of Nginx in production environments
- Common installation paths for Nginx
- Structure of the main Nginx configuration file
- The `main` global configuration block
- The `events` event model configuration block
- The `http` HTTP configuration block
- The `server` virtual host configuration block
- The `location` route matching configuration block
- The `include` configuration inclusion method
- Nginx master/worker process models
- Common running commands
- Configuration checking and reloading
- Viewing compilation parameters
- Checking listening ports
- Viewing log file locations
- Basic procedures before and after configuration changes
- Production environment considerations

This article is part of the Nginx Access Layer Ops series, Chapter 01.

Objectives:

```text
Understand the basic structure of nginx.conf

→ Know how many layers Nginx configuration is divided into

→ Understand what common running parameters mean

→ Be able to safely check configurations

→ Correctly reload Nginx

→ Know what actions to take before and after modifying configurations
```

---

## II. Nginx's Role in Production Environments

Common roles of Nginx include:

```text
Web static resource server

Reverse proxy server

Layer 7 load balancing entry point

HTTPS termination layer

Unified domain name entry point

API access layer

Front-end and back-end separation entry point

One of the underlying implementations for Ingress Controllers

Gray release entry point

Access log collection entry point

Simple rate limiting and access control entry point
```

In production architectures, Nginx is typically positioned between:

```text
Users/Customers

→ CDN/WAF/SLB

→ Nginx

→ Back-end application services

→ Databases/Caches/MQs
```

In summary:

```text
Nginx serves as a crucial access layer component before business traffic reaches back-end services.
```

---

## III. Common Directory Structures for Nginx

Path locations may vary depending on the installation method.

Common paths include:

```text
/etc/nginx/nginx.conf
→ Main configuration file

/etc/nginx/conf.d/
→ Sub-configuration directory, often used for service-specific server configurations

/etc/nginx/sites-enabled/
→ Common directory for enabling sites in Debian/Ubuntu systems

/etc/nginx/sites-available/
→ Common directory for available sites in Debian/Ubuntu systems

/var/log/nginx/access.log
→ Default access log file

/var/log/nginx/error.log
→ Default error log file

/usr/share/nginx/html
→ Default static file directory

/var/run/nginx.pid
→ Nginx process PID file

/run/nginx.pid
→ PID file on some systems
```

To view related Nginx paths:

```bash
whereis nginx
```

To view the main configuration file:

```bash
ls -lh /etc/nginx/nginx.conf
```

To view the configuration directory:

```bash
ls -lh /etc/nginx/
```

To view sub-configuration directories:

```bash
ls -lh /etc/nginx/conf.d/
```

To view log files:

```bash
ls -lh /var/log/nginx/
```

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

This command will display:

```text
Nginx version

OpenSSL version

Compiled modules

Configuration file path

Log file path

PID file path

Temporary directory path
```

Common areas of interest include:

```text
--conf-path
→ Main configuration file path

--error-log-path
→ Error log file path

--http-log-path
→ Access log file path

--pid-path
→ PID file path

--with-http_ssl_module
→ Whether HTTPS is supported

--with-http_stub_status_module
→ Whether stub_status is supported

--with-http_gzip_static_module
→ Whether gzip_static is supported
```

To view and format compilation parameters:

```bash
nginx -V 2>&1 | tr ' ' '\n'
```

To filter the configuration file path:

```bash
nginx -V 2>&1 | tr ' ' '\n' | grep conf-path
```

To filter SSL modules:

```bash
```nginx
server_name example1.com example2.com;
```

匹配所有子域名：

```nginx
server_name *.example.com;
```

---

## 场景21：root

```nginx
root /usr/share/nginx/html;
```

作用：

```text`
指定虚拟主机的文档根目录
```

示例：

```nginx
root /path/to/html;

location / {
    ...
}
```

访问 `/` 时，Nginx 会从 `/path/to/html` 目录开始搜索资源。

---

## 场景22：index

```nginx
index index.html;
```

作用：

```text`
指定默认首页文件
```

当请求没有指定具体的 URL 时，Nginx 会尝试查找以下文件：

1. `index.html`
2. `index.htm`
3. `index.php`

如果找不到这些文件，Nginx 将返回 404 错误。

---

## 场景23：location

```nginx
location / {
    ...
}
```

作用：

```text
配置特定 URI 的处理规则
```

示例：

```nginx
location /example.com/ {
    root /path/to/example;
    index index.html index.htm;
}
```

当请求 `http://example.com` 时，Nginx 会从 `/path/to/example` 目录开始搜索资源，并返回 `index.html` 或 `index.htm` 文件。

---

## 场景24：ifModule

```nginx
ifModule mod_rewrite.c {
    ...
}
```

作用：

```text
根据模块的存在与否来配置不同的规则
```

示例：

```nginx
ifModule modrewrite.c exists {
    # 当 mod_rewrite 模块存在时，执行这些规则
}
else {
    # 当 mod_rewrite 模块不存在时，执行这些规则
}
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

View the requested Host:

```bash
curl -H "Host: example.com" http://127.0.0.1/
```

---

## Section 10: Basics of the `location` Configuration Block

The `location` block is used to match request URIs.

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

Common uses:

```text
Redirecting different paths to different backends

Using different static directories for different paths

Configuring different caching settings for different paths

Setting different permission controls for different paths

Configuring different logging strategies for different paths
```

Detailed `location` matching rules and `proxy_pass` routing rules are covered in Chapter 02.

---

## Section 11: Splitting Configuration Using `include`

In production, it is not recommended to include all business configurations within the `nginx.conf` file.

Common approach:

```nginx
include /etc/nginx/conf.d/*.conf;
```

Explanation:

```text
Nginx will load all .conf files located in /etc/nginx/conf.d/.
```

View included configurations:

```bash
grep -R "include" /etc/nginx/nginx.conf
```

List sub-configurations:

```bash
ls -lh /etc/nginx/conf.d/
```

Create a new business configuration file:

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

## Section 12: Nginx Master/Worker Process Model

Nginx typically consists of:

```text
master process

worker process
```

View processes:

```bash
ps -ef | grep nginx | grep -v grep
```

Example output:

```text
root      1000     1  0 10:00 ?  00:00:00 nginx: master process nginx
nginx     1001  1000  0 10:00 ?  00:00:00 nginx: worker process
nginx     1002  1000  0 10:00 ?  00:00:00 nginx: worker process
```

Explanation:

```text
master process:
→ Loads configuration, manages worker processes, and receives signals.

worker process:
→ Handles actual client requests.
```

Common operations:

```text
reload:
→ The master process reloads the configuration, causing worker processes to restart smoothly.

restart:
→ Stops and then restarts Nginx, which may cause more significant disruptions.

quit:
→ Exits Nginx gracefully.

stop:
→ Quickly stops Nginx.
```

---

## Section 13: Common Nginx Command Lines

---

## Scenario 21: Starting Nginx

Using systemd:

```bash
systemctl start nginx
```

Direct command:

```bash
nginx
```

---

## Scenario 22: Stopping Nginx

```bash
systemctl stop nginx
```

Or:

```bash
nginx -s stop
```

Explanation:

```text
stop:
→ Quickly stops Nginx.
```

---

## Scenario 23: Gracefully Exiting Nginx

```bash
nginx -s quit
```

Explanation:

```text
quit:
→ Wait for all current connections to be handled before exiting.
```

---

## Scenario 24: Loading Nginx Configuration

Recommended methods:

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

Explanation:

```text
reload reloads the configuration file.
It usually causes less disruption than restarting Nginx.
In production, reload should be used instead of direct restart.
```

---

## Scenario 25: Restarting Nginx

```bash
systemctl restart nginx
```

Explanation:

```text
restart stops and then starts Nginx, which may cause more significant disruptions.
Use it with caution in a production environment.
Suitable for:

– When the service becomes unresponsive or freezes.
– When reload commands fail.
– During Nginx upgrades.
– When there are changes to special modules or system```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
``````bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

```bash
ls -lh /tmp/nginx-conf-backup-*.tar.gz
```

---

## Summary in One Sentence

The core of Nginx configuration basics includes:

```text
main
→ Global runtime parameters

events
→ Connection and event handling model

http
→ HTTP global settings

server
→ A virtual host configuration

location
→ URI matching rules

upstream
→ Group of backend services
```

Common operations commands:

```text
nginx -t
→ Check configuration

nginx -T
→ Display full configuration

systemctl reload nginx
→ Smooth reload

systemctl restart nginx
→ Restart service

nginx -s reopen
→ Reopen logs
```

Production change process:

```text
Back up configuration

→ Make changes to configuration

→ Check configuration with nginx -t

→ Verify locally using curl

→ Reload configuration

→ Monitor logs and observe service performance

→ Roll back changes if necessary
```

Production recommendations:

```text
Do not reload configuration immediately after making changes.

Always back up your configuration before making modifications.

Do not include all services in the same nginx.conf file.

Avoid restarting the server indiscriminately.

Do not disable access_log files without proper reason.

Do not disclose Nginx version numbers.

Keep configuration files under version control.

Always verify request and error logs after making configuration changes.
```