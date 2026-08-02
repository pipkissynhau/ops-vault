# 06-Nginx Static Resources, root, alias, Caching, and Access Control

#Nginx #StaticResources #root #alias #Cache #AccessControl #WebServer #AccessLayer #Middle #Transport #SRE

---

## Recommended Path

07-Middlewares/Web Server/Nginx/01-Nginx Access Layer Operations/06-Nginx Static Resources, root-alias, Caching, and Access Control.md

---

## One: Document Description

This document organizes content about Nginx static resource services, `root`, `alias`, caching control, and access control.

Key points covered in this article include:

- Nginx Static Resource Service Basics
- Difference between `root` and `alias`
- Relationship between `location` and static directory mapping
- `index` homepage file
- `try_files` basics
- `try_files` configuration for single-page applications (SPAs)
- Static resource caching control
- `expires` and `Cache-Control`
- Caching strategies for images, CSS, and JS
- Disabling caching for HTML
- `gzip` compression basics
- `autoindex` directory browsing control
- `allow` / `deny` IP access control
- Basic Auth access authentication
- Hiding sensitive files
- Preventing access to `.git`, `.env`, and backup files
- Common troubleshooting for 403 / 404 errors
- Static resource permission issues
- Production environment considerations

This article is part of the Nginx Access Layer Operations series, Article 06.

This article's objectives:

```text
It works. Nginx Provide static resource access

→ I understand. root and alias The path clutter difference

→ correctly configure front-end static sites

→ I can handle it. SPA Refresh 404 Problem

→ Configure static resource caches

→ It limits access to sensitive paths.

→ I can check. 403I don't know.404Cache failure, static file access anomaly
```

---

## Two: Role of Nginx Static Resource Service

Nginx can not only act as a reverse proxy but also directly provide static resource access.

Common static resources include:

```text
HTML

CSS

JavaScript

Picture

Fonts

Download File

Front-end builder

Document File

Static sites
```

Typical scenarios:

```text
Frontend Vue / React / Next Static construction product

Home Page of the Official Network

Manage frontend pages in the background

Photo Resource Directory

File Download Directory

Static Document Sites

Transport Download Page
```

Request flow:

```text
Client

→ Nginx

→ Local disk static file
```

Unlike reverse proxy:

```text
Reverse Agent
→ Nginx Forward to Backend Service

Static resources
→ Nginx Read this machine file directly and return
```

---

## Three: Basic Static Resource Configuration

---

## Scenario 1: Most Basic Static Website Configuration

Assume static file directory:

```text
/data/www/example
```

Directory contents:

```text
/data/www/example/index.html

/data/www/example/static/app.js

/data/www/example/static/app.css
```

Nginx configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Meaning:

```text
root /data/www/example
→ Static Resource Roots Directory

index index.html
→ Default homepage file

try_files $uri $uri/ =404
→ Find the corresponding request file, then find the directory. 404
```

Check configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Local verification:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

Access file:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.js
```

---

## Scenario 2: Create Test Static Directory

Create directory:

```bash
mkdir -p /data/www/example/static
```

Create homepage:

```bash
cat > /data/www/example/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nginx Static Demo</title>
</head>
<body>
    <h1>Hello Nginx Static</h1>
</body>
</html>
EOF
```

Create test JS:

```bash
cat > /data/www/example/static/app.js <<'EOF'
console.log("hello nginx static");
EOF
```

View files:

```bash
find /data/www/example -type f -maxdepth 3 -ls
```

---

## Four: root Basics

---

## Scenario 3: What is root

`root` is used to specify the root directory for static resources.

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Request:

```text
http://example.com/index.html
```

Actual file path:

```text
/data/www/example/index.html
```

Request:

```text
http://example.com/static/app.js
```

Actual file path:

```text
/data/www/example/static/app.js
```

One-sentence understanding:

```text
root 会把 URI 拼接到 root 目录后面。
```

---

## Scenario 4: root Written in server

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Explanation:

```text
server 内的所有 location 默认继承 root

除非 location 中重新定义 root 或 alias
```

---

## Scenario 5: root Written in location

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ {
        root /data/www/example;
    }
}
```

Request:

```text
/static/app.js
```

Actual file path:

```text
/data/www/example/static/app.js
```

Note:

```text
即使 root 写在 location 中，URI 仍然会拼接在 root 后面
```

---

## Five: alias Basics

---

## Scenario 6: What is alias

`alias` is used to map a specific URI path to a designated directory.

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ {
        alias /data/assets/;
    }
}
```

Request:

```text
/static/app.js
```

Actual file path:

```text
/data/assets/app.js
```

One-sentence understanding:

```text
alias 会用指定目录替换 location 匹配到的路径前缀。
```

---

## Scenario 7: Common Uses of alias

Suitable for:

```text
URL 路径和磁盘目录不一致

多个 URL 映射不同目录

静态资源单独放在另一个目录

下载目录不在站点根目录下

图片目录独立管理
```

Example:

```nginx
location /downloads/ {
    alias /data/files/public/;
}
```

Request:

```text
/downloads/manual.pdf
```

Actual file:

```text
/data/files/public/manual.pdf
```

---

## Six: Core Differences Between root and alias

---

## Scenario 8: root Path Concatenation

Configuration:

```nginx
location /static/ {
    root /data/www/example;
}
```

Request:

```text
/static/app.js
```

Actual path:

```text
/data/www/example/static/app.js
```

Rule:

```text
root 目录 + 完整 URI
```

---

## Scenario 9: alias Path Replacement

Configuration:

```nginx
location /static/ {
    alias /data/assets/;
}
```

Request:

```text
/static/app.js
```

Actual path:

```text
/data/assets/app.js
```

Rule:

```text
alias 目录替换 location 匹配到的 /static/
```

---

## Scenario 10: root and alias Comparison Table

| Configuration | Request URI | Actual File Path | Notes |
|---|---|---|---|
| `location /static/ { root /data/www; }` | `/static/app.js` | `/data/www/static/app.js` | root appends complete URI |
| `location /static/ { alias /data/assets/; }` | `/static/app.js` | `/data/assets/app.js` | alias replaces `/static/` |
| `location /download/ { root /data/files; }` | `/download/a.zip` | `/data/files/download/a.zip` | root preserves URI prefix |
| `location /download/ { alias /data/files/; }` | `/download/a.zip` | `/data/files/a.zip` | alias removes URI prefix |

---

## Seven: alias Trailing Slash Considerations

---

## Scenario 11: Recommend Adding Slash to alias and location

Recommended configuration:

```nginx
location /static/ {
    alias /data/assets/;
}
```

Request:

```text
/static/app.js
```

Actual path:

```text
/data/assets/app.js
```

---

## Scenario 12: Missing Slash in alias Can Cause Errors

Not recommended:

```nginx
location /static/ {
    alias /data/assets;
}
```

May lead to unexpected path concatenation.

Production recommendation:

```text
location /xxx/ 使用 alias 时

alias 目录也使用 / 结尾
```

Recommended:

```nginx
location /static/ {
    alias /data/assets/;
}
```

---

## Eight: index Homepage File

---

## Scenario 13: Purpose of index

Configuration:

```nginx
index index.html index.htm;
```

Purpose:

```text
当请求目录时，默认返回目录下的首页文件
```

For example request:

```text
http://example.com/
```

Actual response:

```text
/data/www/example/index.html
```

If request:

```text
http://example.com/docs/
```

Will attempt:

```text
/data/www/example/docs/index.html
```

---

## Scenario 14: Multiple index Files

Configuration:

```nginx
index index.html index.htm default.html;
```

Search order:

```text
index.html

index.htm

default.html
```

---

## Nine: try_files Basics

---

## Scenario 15: What is try_files

`try_files` is used to attempt file lookup in sequence.

Common configuration:

```nginx
location / {
    try_files $uri $uri/ =404;
}
```

Meaning:

```text
先查找 $uri 对应文件

再查找 $uri/ 对应目录

都没有就返回 404
```

For example request: /think

```text
/static/app.js
```

Try:

```text
/data/www/example/static/app.js
```

If the file exists, return it.

---

## Scenario 16: try_files Returns a Custom 404 Page

Configuration:

```nginx
location / {
    try_files $uri $uri/ /404.html;
}
```

If the file does not exist, return:

```text
/404.html
```

Corresponding file:

```text
/data/www/example/404.html
```

---

## Ten. Single Page Application (SPA) Configuration

---

## Scenario 17: Why SPA Refresh Causes 404

Common frontend routing in Vue / React:

```text
/dashboard

/users/10001

/settings
```

These paths exist in frontend routing but do not correspond to files on disk:

```text
/data/www/example/dashboard

/data/www/example/users/10001
```

If a user refreshes the page, Nginx will look for static files and return 404 if not found.

---

## Scenario 18: Recommended SPA Configuration

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Meaning:

```text
先找真实文件

再找目录

都找不到就返回 /index.html

由前端路由接管
```

Suitable for:

```text
Vue history 模式

React Router BrowserRouter

前后端分离静态站点
```

---

## Scenario 19: SPA + API Reverse Proxy

Common configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Explanation:

```text
/api/ 请求转发给后端

其他路径作为前端路由处理

静态文件优先按真实文件返回
```

---

## Eleven. Static Resource Cache Control

---

## Scenario 20: Why Configure Caching

Static resources typically include:

```text
CSS

JS

图片

字体

图标
```

These files can be cached by browsers to reduce:

```text
重复请求

页面加载时间

Nginx 带宽压力

后端压力

用户访问延迟
```

However, HTML files generally should not be cached long-term, as it may lead to:

```text
前端更新后用户还看到旧页面

JS / CSS 文件版本不匹配

发布后页面异常
```

---

## Scenario 21: Set Long-Term Caching for Static Resources

Configuration:

```nginx
location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public";
}
```

Explanation:

```text
expires 30d
→ 浏览器缓存 30 天

Cache-Control public
→ 允许中间缓存和浏览器缓存
```

Suitable for:

```text
带 hash 文件名的静态资源

例如 app.8f3a1c.js

例如 style.2ab91.css
```

---

## Scenario 22: Disable Strong Caching for HTML

Configuration:

```nginx
location ~* \.html$ {
    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

Explanation:

```text
HTML 不长期缓存

避免前端发布后入口文件不更新
```

---

## Scenario 23: Recommended Caching Strategy for Frontend Build Artifacts

Common frontend build artifacts:

```text
index.html

assets/app.8f3a1c.js

assets/style.a83bc.css

assets/logo.91af2.png
```

Recommendation:

```text
index.html
→ 不强缓存

带 hash 的 JS / CSS / 图片
→ 长缓存
```

Configuration example:

```nginx
location = /index.html {
    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}

location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

---

## Scenario 24: Verify Cache Response Headers

Check response headers:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.js
```

Focus on:

```text
Cache-Control

Expires

Last-Modified

ETag
```

Check homepage response headers:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/index.html
```

---

## Twelve. Gzip Compression Basics

---

## Scenario 25: Why Enable gzip

gzip can compress text-based resources, such as:

```text
HTML

CSS

JavaScript

JSON

XML

SVG
```

Benefits:

```text
减少传输大小

提升页面加载速度

降低带宽压力
```

Not suitable for repeated compression:

```text
jpg

png

gif

mp4

zip

gz
```

These are already compressed, so gzip provides minimal benefit.

---

## Scenario 26: Basic gzip Configuration

Typically written in the `http` block:

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 1k;
gzip_types text/plain text/css application/javascript application/json application/xml image/svg+xml;
```

Explanation:

```text
gzip on
→ 开启 gzip

gzip_comp_level
→ 压缩级别，越高越耗 CPU

gzip_min_length
→ 小于该大小不压缩

gzip_types
→ 指定压缩 MIME 类型
```

Check configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

---

## Scenario 27: Verify if gzip is Active

Request with compression header:

```bash
curl -I -H "Accept-Encoding: gzip" -H "Host: example.com" http://127.0.0.1/static/app.js
```

Focus on response headers:

```text
Content-Encoding: gzip
```

---

## Thirteen. autoindex Directory Browsing

---

## Scenario 28: What is autoindex

`autoindex` can list directory files when no index file exists.

Configuration:

```nginx
location /download/ {
    alias /data/files/;
    autoindex on;
}
```

Access:

```text
http://example.com/download/
```

You may see a list of directory files.

---

## Scenario 29: Disable autoindex in Production

Production recommendation:

```nginx
autoindex off;
```

Reason:

```text
避免暴露目录结构

避免暴露文件名

避免泄露备份文件

避免泄露历史包

避免被扫描器利用
```

If a file download list is indeed needed, it should be paired with:

```text
访问认证

IP 白名单

独立下载域名

严格目录权限

文件保留周期
```

---

## Fourteen. IP Access Control

---

## Scenario 30: allow / deny Basics

Nginx can control access by IP.

Allow only internal network access:

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    proxy_pass http://127.0.0.1:8080;
}
```

Meaning:

```text
允许 10.0.0.0/8

允许 192.168.0.0/16

其他全部拒绝
```

---

## Scenario 31: Allow Only Single IP Access

Configuration:

```nginx
location /private/ {
    allow 10.0.0.10;
    deny all;

    root /data/www/example;
}
```

---

## Scenario 32: Deny Specific IP

Configuration:

```nginx
location / {
    deny 1.2.3.4;
    allow all;

    root /data/www/example;
}
```

Production note:

```text
如果 Nginx 前面还有 SLB / CDN / WAF

allow / deny 判断的是 Nginx 看到的 remote_addr

不一定是真实客户端 IP

真实 IP 配置会在第 07 篇单独整理
```

---

## Fifteen. Basic Auth Access Authentication

---

## Scenario 33: When is Basic Auth Suitable

Suitable for:

```text
临时下载页面

内部文档页面

测试环境入口

简单管理后台保护

临时演示站点
```

Not suitable for:

```text
替代正式认证系统

保护高敏感生产后台

复杂权限控制
```

---

## Scenario 34: Install htpasswd Tool

Ubuntu / Debian:

```bash
apt install -y apache2-utils
```

RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y httpd-tools
```

Or:

```bash
dnf install -y httpd-tools
```

---

## Scenario 35: Create Authentication File

Create user:

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

Append user:

```bash
htpasswd /etc/nginx/.htpasswd user1
```

View file:

```bash
cat /etc/nginx/.htpasswd
```

Set permissions:

```bash
chmod 640 /etc/nginx/.htpasswd
```

---

## Scenario 36: Configure Basic Auth

Configuration:

```nginx
location /download/ {
    alias /data/files/;

    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
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

Verify:

```bash
curl -I -u admin:密码 http://example.com/download/
```

Unauthenticated access:

```bash
curl -I http://example.com/download/
```

Expected:

```text
401 Unauthorized
```

---

## Sixteen. Access Control for Sensitive Files

---

## Scenario 37: Prevent Access to Hidden Files

Hidden files include:

```text
.env

.git

.svn

.htaccess

.htpasswd
```

Configuration:

```nginx
location ~ /\. {
    deny all;
}
```

Explanation:

```text
匹配以 . 开头的隐藏路径

直接拒绝访问
```

---

## Scenario 38: Prevent Access to .git Directory

Configuration:

```nginx
location ~* /\.git {
    deny all;
}
```

Check if exposed:

```bash
curl -I http://example.com/.git/config
```

Security expectation:

```text
403 Forbidden

或

404 Not Found
```

---

## Scenario 39: Prevent Access to Environment Configuration Files

Configuration:

```nginx
location ~* \.(env|ini|conf|bak|backup|old|sql|tar|gz|zip)$ {
    deny all;
}
```

Explanation:

```text
防止直接访问敏感配置、备份文件、SQL 文件、压缩包
```

Production note:

```text
这个规则要结合业务实际

如果业务确实需要下载 zip，需要避免误伤
```

More secure approach:

```text
敏感文件不要放在 Web 根目录下
```

---

## Scenario 40: Prevent Access to Specific Directories

Configuration:

```nginx
location ^~ /private/ {
    deny all;
}
```

Or return 404:

```nginx
location ^~ /private/ {
    return 404;
}
```

Explanation:

```text
return 404 比 403 更不容易暴露目录存在性
```

---

## Seventeen. Complete Static Resource Configuration Example

---

## Scenario 41: Ordinary Static Site Configuration

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    index index.html;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    location ~ /\. {
        deny all;
    }
}
```

---

## Scenario 42: SPA + API + Static Cache Configuration

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    index index.html;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location = /index.html {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    location ~ /\. {
        deny all;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## Scenario 43: Download Directory + Basic Auth Configuration

```nginx
server {
    listen 80;
    server_name download.example.com;

    location /files/ {
        alias /data/files/;

        autoindex off;

        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }

    location ~ /\. {
        deny all;
    }
}
```

---

## Eighteen. 403 Forbidden Troubleshooting

---

## Scenario 44: Common Causes of 403

Common causes:

```text
文件或目录权限不足

Nginx worker 用户无权限读取

目录没有 index 文件且 autoindex 关闭

被 allow / deny 拒绝

被 location 规则 deny all

SELinux 限制

root / alias 路径写错

父目录没有执行权限
```

---

## Scenario 45: 403 Troubleshooting Commands

Check error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

Filter permission:

```bash
grep -i "permission denied" /var/log/nginx/error.log | tail -n 100
```

Check Nginx process user:

```bash
ps -ef | grep nginx | grep -v grep
```

Check file permissions:

```bash
ls -lh /data/www/example/index.html
```

Check directory permissions:

```bash
ls -ld /data/www /data/www/example
```

Check permissions hierarchically:

```bash
namei -l /data/www/example/index.html
```

---

## Scenario 46: 403 Due to Missing Index File in Directory

If the request is: /think

```text
http://example.com/docs/
```

Directory exists:

```text
/data/www/example/docs/
```

But directory does not contain:

```text
index.html
```

And:

```nginx
autoindex off;
```

May return:

```text
403 Forbidden
```

Handling method:

```text
增加 index.html

或明确 return 404

或谨慎开启 autoindex
```

---

## 19. 404 Not Found Troubleshooting

---

## Scenario 47: Common Causes of 404

Common causes:

```text
文件确实不存在

root 路径写错

alias 路径写错

root 和 alias 混淆

try_files 配置不正确

SPA 没有回退到 index.html

请求命中了错误 server

请求命中了错误 location

配置文件未被 include

Nginx 未 reload
```

---

## Scenario 48: 404 Troubleshooting Commands

Check full configuration:

```bash
nginx -T | grep -n "server_name example.com" -A 80
```

Check root:

```bash
nginx -T | grep -n "root"
```

Check alias:

```bash
nginx -T | grep -n "alias"
```

Check location:

```bash
nginx -T | grep -n "location"
```

Confirm file existence:

```bash
ls -lh /data/www/example/static/app.js
```

Local request:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/static/app.js
```

Check access log:

```bash
tail -n 100 /var/log/nginx/access.log
```

Check error log:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## 20. Cache Not Working Troubleshooting

---

## Scenario 49: Common Causes of Cache Not Working

Common causes:

```text
location 没匹配到静态资源规则

add_header 没有出现在当前响应中

请求命中了其他 server

文件后缀不在正则范围内

浏览器强制刷新

上层 CDN 覆盖缓存头

后端返回了其他 Cache-Control

HTML 被错误长缓存

Nginx 未 reload
```

---

## Scenario 50: Check Cache Response Headers

Check JS:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.js
```

Check CSS:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.css
```

Check homepage:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/index.html
```

Pay attention to:

```text
Cache-Control

Expires

ETag

Last-Modified
```

Check full configuration:

```bash
nginx -T | grep -n "Cache-Control" -A 5 -B 5
```

---

## 21. Production Notes

---

## 1. Do not mix root and alias incorrectly

Remember:

```text
root
→ root 目录 + 完整 URI

alias
→ alias 目录替换 location 匹配前缀
```

Most likely to make mistakes:

```nginx
location /static/ {
    root /data/assets;
}
```

It actually looks for:

```text
/data/assets/static/xxx
```

If the real directory is:

```text
/data/assets/xxx
```

Should use:

```nginx
location /static/ {
    alias /data/assets/;
}
```

---

## 2. Alias trailing slash should be standardized

Recommended:

```nginx
location /static/ {
    alias /data/assets/;
}
```

Avoid path concatenation issues.

---

## 3. SPA should configure try_files fallback

Recommended:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Otherwise, frontend routing refresh may result in 404.

---

## 4. index.html should not be long cached

After frontend deployment, the entry HTML should avoid strong caching.

Recommended:

```nginx
location = /index.html {
    expires -1;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

---

## 5. Static resources with hash are suitable for long caching

For example:

```text
app.8f3a1c.js

style.a9c12.css
```

Suitable for:

```nginx
expires 30d;
```

Even longer, but should be combined with deployment strategy.

---

## 6. autoindex is disabled by default in production

Not recommended:

```nginx
autoindex on;
```

Unless directory listing is explicitly needed, and combined with authentication, whitelist, and permission controls.

---

## 7. Sensitive files should not be placed in web root

Nginx rules are just one form of protection.

More fundamentally:

```text
.env

.git

数据库备份

配置文件

私钥

压缩包

SQL 文件
```

Should not be placed in public web root directory.

---

## 8. Basic Auth is only for simple protection

Basic Auth is suitable for temporary or internal pages, not for complex permission systems.

Production core backend recommends using:

```text
正式登录认证

SSO

VPN

堡垒机

零信任接入

IP 白名单

多因素认证
```

---

## 9. Static file permissions should be minimized

Recommended:

```text
Nginx worker 用户可读

不需要写权限

上传目录和静态目录分离

发布用户和运行用户权限分离
```

---

## 22. Summary of Common Commands in This Article

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "root"
```

```bash
nginx -T | grep -n "alias"
```

```bash
nginx -T | grep -n "try_files"
```

```bash
nginx -T | grep -n "Cache-Control" -A 5 -B 5
```

```bash
nginx -T | grep -n "server_name example.com" -A 80
```

---

## Service Reload

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

---

## Create Test Static Resources

```bash
mkdir -p /data/www/example/static
```

```bash
cat > /data/www/example/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nginx Static Demo</title>
</head>
<body>
    <h1>Hello Nginx Static</h1>
</body>
</html>
EOF
```

```bash
cat > /data/www/example/static/app.js <<'EOF'
console.log("hello nginx static");
EOF
```

```bash
find /data/www/example -type f -maxdepth 3 -ls
```

---

## Request Verification

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/index.html
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.js
```

```bash
curl -v -H "Host: example.com" http://127.0.0.1/static/app.js
```

---

## Cache Verification

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.js
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.css
```

```bash
curl -I -H "Host: example.com" http://127.0.0.1/index.html
```

---

## gzip Verification

```bash
curl -I -H "Accept-Encoding: gzip" -H "Host: example.com" http://127.0.0.1/static/app.js
```

---

## Permission Troubleshooting

```bash
ps -ef | grep nginx | grep -v grep
```

```bash
ls -lh /data/www/example/index.html
```

```bash
ls -ld /data/www /data/www/example
```

```bash
namei -l /data/www/example/index.html
```

---

## Log Viewing

```bash
tail -n 100 /var/log/nginx/access.log
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
grep -i "permission denied" /var/log/nginx/error.log | tail -n 100
```

---

## Basic Auth

```bash
apt install -y apache2-utils
```

```bash
yum install -y httpd-tools
```

```bash
dnf install -y httpd-tools
```

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

```bash
htpasswd /etc/nginx/.htpasswd user1
```

```bash
cat /etc/nginx/.htpasswd
```

```bash
chmod 640 /etc/nginx/.htpasswd
```

```bash
curl -I -u admin:Password http://example.com/download/
```

```bash
curl -I http://example.com/download/
```

---

## Sensitive File Exposure Check

```bash
curl -I http://example.com/.git/config
```

```bash
curl -I http://example.com/.env
```

```bash
curl -I http://example.com/backup.sql
```

```bash
curl -I http://example.com/config.bak
```

---

## 23. One-Sentence Summary

Nginx static configuration core is:

```text
root

alias

index

try_files

expires

Cache-Control

autoindex

allow / deny

auth_basic
```

Difference between root and alias:

```text
root
→ root Contents + Full URI

alias
→ alias Directory Replace location Match Prefix
```

Recommended SPA configuration:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

Recommended static caching:

```text
index.html
→ No Cache

And... hash Yes. JS / CSS / Picture
→ Long Cache
```

Security recommendations:

```text
Production Default Closes autoindex

Ban Access .git / .env / Hide File

Don't release sensitive files. Web Root Directory

Download directory with authentication or IP White list.

Basic Auth Only for simple protection.

Static directory privileges only Nginx Readable.
```

Common issues:

```text
403
→ Insufficient permissions, no home page, by denyI don't know.autoindex Close

404
→ File does not exist,root/alias It's wrong.try_files Error. Wrong hit. server/location

Cache is invalid
→ location Not matched, response header not set,CDN Overwrite, not reload

SPA Refresh 404
→ Missing try_files $uri $uri/ /index.html
```

Production recommendations:

```text
root and alias You have to use it before you go online. curl Verify the actual path

alias The slash at the end of the directory should be regulated

index.html Long Cache not recommended

And... hash Static resources can sustain caches.

Do not enter static directories for sensitive files

Backup before changing the configuration.reload I have to. nginx -t
```