# 06-Nginx Static Resources, Root, Alias, Caching, and Access Control

# Nginx #Static Resources #Root #Alias #Caching #Access Control #Web Server #Access Layer #Middleware #Operation and Maintenance #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operation and Maintenance/06-Nginx Static Resources, Root-Alias, Caching, and Access Control.md

---

## I. Document Overview

This article outlines Nginx-related settings for static resource services, `root`, `alias`, caching control, and access management.

Key topics include:

- Basics of Nginx static resource services
- Differences between `root` and `alias`
- The relationship between `location` and static directory mapping
- The `index` file for setting the homepage
- Basics of `try_files`
- `try_files` configuration for single-page applications (SPA)
- Static resource caching control
- `expires` and `Cache-Control` headers
- Caching strategies for images, CSS, and JS files
- Preventing HTML file caching
- Basics of `gzip` compression
- `autoindex` for directory browsing control
- `allow`/`deny` IP address access control
- Basic Auth access authentication
- Hiding sensitive files
- Protecting `.git`, `.env`, and backup files from access
- Common troubleshooting for 403/404 errors
- Issues with static resource permissions
- Best practices for production environments

This article is part of the Nginx Access Layer Operation and Maintenance series, Chapter 06.

Objectives:

```text
- Be able to use Nginx to serve static resources effectively.
- Understand the differences in path concatenation between `root` and `alias`.
- Correctly configure front-end static websites.
- Resolve 404 errors when SPA pages are refreshed.
- Set up static resource caching properly.
- Restrict access to sensitive directories.
- Diagnose and fix issues related to 403, 404 errors, ineffective caching, and abnormal static file access.
```

---

## II. Role of Nginx in Serving Static Resources

Nginx can not only act as a reverse proxy but also directly provide access to static resources.

Common types of static resources include:

```text
HTML files
CSS stylesheets
JavaScript scripts
Images
Fonts
Downloadable files
Front-end build outputs
Documentation files
Static websites
```

Typical use cases:

```text
- Static build outputs for front-end frameworks like Vue, React, or Next.js.
- Homepages of official websites.
- Front-end pages of management interfaces.
- Image resource directories.
- File download directories.
- Static documentation sites.
- Operation and maintenance download pages.
```

Request flow:

```text
Client → Nginx → Local static files on the server
```

Unlike a reverse proxy:

```text
Reverse Proxy → Nginx forwards requests to the backend service.
Static Resources → Nginx directly retrieves the files from its local storage and returns them to the client.
```

---

## III. Basic Configuration of Static Resources

---

## Scenario 1: Basic configuration for a static website

Assume the static file directory is:

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

Explanation:

```text
root /data/www/example
→ Specifies the root directory for static resources.
index index.html
→ Sets the default homepage file.
try_files $uri $uri/ =404
→ First tries to find the requested file, then searches the directory. If not found, returns a 404 error.
```

Verification:

```bash
nginx -t
```

Reload configuration:

```bash
systemctl reload nginx
```

Local test:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

Access files:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/static/app.js
```

---

## Scenario 2: Creating a test static directory

Create a new directory:

```bash
mkdir -p /data/www/example/static
```

Create a homepage file:

```bash
cat > /data/www/example/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Nginx Static Demo</title>
</head>
<body>
    <h| `location /static/ { root /data/www; }` | `/static/app.js` | `/data/www/static/app.js` | The full URI is concatenated from the root directory |
| `location /static/ { alias /data/assets/; }` | `/static/app.js` | `/data/assets/app.js` | The alias replaces `/static/` in the path |
| `location /download/ { root /data/files; }` | `/download/a.zip` | `/data/files/download/a.zip` | The root directory prefix is retained in the URI |
| `location /download/ { alias /data/files/; }` | `/download/a.zip` | `/data/files/a.zip` | The URI prefix is removed when using the alias |

---

## VII. Notes on Slashes at the End of Aliases

---

## Scenario 11: It is recommended to include slashes in both aliases and location directives

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

## Scenario 12: Omitting slashes in aliases can lead to errors

Not recommended:

```nginx
location /static/ {
    alias /data/assets;
}
```

This may result in incorrect path concatenation.

Production advice:

```text
When using an alias in /xxx/, ensure the alias directory also ends with a slash
```

Recommended configuration:

```nginx
location /static/ {
    alias /data/assets/;
}
```

---

## VIII. Index Pages

---

## Scenario 13: The role of index files

Configuration:

```nginx
index index.html index.htm;
```

Function:

```text
When requesting a directory, the server returns the default homepage file within that directory by default.
```

For example, if you request:

```text
http://example.com/
```

The actual response will be:

```text
/data/www/example/index.html
```

If you request:

```text
http://example.com/docs/
```

Nginx will attempt to find:

```text
/data/www/example/docs/index.html
```

---

## Scenario 14: Multiple index files

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

## IX. Basic Usage of try_files

---

## Scenario 15: What is try_files?

`try_files` is used to attempt finding files in a specified order.

Common configuration:

```nginx
location / {
    try_files $uri $uri/ =404;
}
```

Explanation:

```text
First, it tries to find the file corresponding to `$uri`.

If not found, it then checks the directory corresponding to `$uri/`.

If still absent, a 404 error is returned.
```

For example, if you request:

```text
/static/app.js
```

Nginx will first look for:

```text
/data/www/example/static/app.js
```

If it exists, the file will be returned.

---

## Scenario 16: Using try_files to return a custom 404 page

Configuration:

```nginx
location / {
    try_files $uri $uri/ /404.html;
}
```

If the requested file does not exist, Nginx will return:

```text
/404.html
```

The corresponding file would be located at:

```text
/data/www/example/404.html
```

---

## X. Configuration for Single Page Applications (SPA)

---

## Scenario 17: Why do SPA refreshes result in a 404 error?

Common routing patterns in Vue / React include:

```text
/dashboard

/users/10001

/settings
```

These paths exist in the front-end routing system, but there are no corresponding files on the server:

```text
/data/www/example/dashboard

/data/www/example/users/10001
```

When a user refreshes the page, Nginx will attempt to find the file based on the static file structure. Since it doesn't exist, a 404 error is returned.

---

## Scenario 18: Recommended configuration for SPA

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

Explanation:

```text
First, it tries to find the actual file.

If not found, it checks the directory.

If neither is present, it returns `/index.html`, allowing the front-end routing system to handle the request.
```

This configuration is suitable for:

```text
Vue history mode

React Router BrowserRouter

Static sites with front-backend separation
``JavaScript

JSON

XML

SVG
```

Advantages:

```text
Reduces transmission size

Improves page loading speed

Decreases bandwidth usage
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

These formats are already compressed, so using gzip does not significantly improve the compression ratio.

---

## Scenario 26: Basic Configuration of gzip

It is usually configured in the `http` block:

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 1k;
gzip_types text/plain text/css application/javascript application/json application/xml image/svg+xml;
```

Explanation:

```text
gzip on
→ Enable gzip compression

gzip_comp_level
→ Compression level; higher values consume more CPU resources

gzip_min_length
→ Files smaller than this size will not be compressed

gzip_types
→ Specifies the MIME types to be compressed
```

To check the configuration:

```bash
nginx -t
```

To reload the configuration:

```bash
systemctl reload nginx
```

---

## Scenario 27: Verifying if gzip is Effective

Include a compression header in the request:

```bash
curl -I -H "Accept-Encoding: gzip" -H "Host: example.com" http://127.0.0.1/static/app.js
```

Pay attention to the response header:

```text
Content-Encoding: gzip
```

---

## Section 13: Autoindex for Directory Browsing

---

## Scenario 28: What is autoindex?

`autoindex` allows listing directory files when there is no home page file.

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

## Scenario 29: Autoindex is usually disabled in production

It is recommended to disable `autoindex` in production environments:

```nginx
autoindex off;
```

Reasons:

```text
To prevent the directory structure from being exposed

To avoid revealing file names

To prevent backup files from being disclosed

To prevent historical packages from being exposed

To prevent scanners from exploiting it
```

If you really need a list of downloadable files, consider implementing the following measures:

```text
Access authentication

IP address whitelist

Separate download domain name

Strict directory permissions

Set file retention periods
```

---

## Section 14: IP Access Control

---

## Scenario 30: Basic use of allow / deny

Nginx allows access based on IP addresses.

Allowing only internal network access:

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
Allow access from the IP ranges 10.0.0.0/8 and 192.168.0.0/16

Deny access to all other IP addresses
```

---

## Scenario 31: Allowing access only from a specific IP address

Configuration:

```nginx
location /private/ {
    allow 10.0.0.10;
    deny all;

    root /data/www/example;
}
```

---

## Scenario 32: Denying access from a particular IP address

Configuration:

```nginx
location / {
    deny 1.2.3.4;
    allow all;

    root /data/www/example;
}
```

Note in production environments:

```text
If Nginx is preceded by SLB, CDN, or WAF,

the `allow` and `deny` rules are based on the `remote_addr` seen by Nginx,

which may not necessarily be the actual client IP address.

The configuration of the actual client IP address will be discussed in Chapter 07.
```

---

## Section 15: Basic Auth Access Authentication

---

## Scenario 33: Suitable scenarios for Basic Auth

Suitable for:

```text
Temporary download pages

Internal documentation pages

Entrance to test environments

Simple management backends

Temporary demonstration sites
```

Not suitable for:

```text
Replacing formal authentication systems

Protecting highly sensitive production backends

Implementing complex permission controls
```

---

## Scenario 34: Installing the htpasswd tool

For Ubuntu / Debian:

```bash
apt install -y apache2-utils
```

For RHEL / CentOS / Rocky / AlmaLinux:

```bash
yum install -y httpd-tools
```

Or:

```bash
dnf install -y httpd-tools
```

---

## Scenario 35: Creating authentication files

To```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

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

## Chapter 18: Troubleshooting 403 Forbidden

---

## Scenario 44: Common Causes of 403

Common causes include:

```text
Insufficient file or directory permissions

The Nginx worker user does not have read access

The directory lacks an index file and autoindex is disabled

Access is denied by allow/deny rules

Denial by location rules

SELinux restrictions

Incorrect root/alias path configuration

Parent directory lacks execute permissions
```

---

## Scenario 45: Commands for Troubleshooting 403

To view error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

To filter for permission issues:

```bash
grep -i "permission denied" /var/log/nginx/error.log | tail -n 100
```

To check the Nginx process user:

```bash
ps -ef | grep nginx | grep -v grep
```

To view file permissions:

```bash
ls -lh /data/www/example/index.html
```

To check directory permissions:

```bash
ls -ld /data/www /data/www/example
```

To recursively check permissions:

```bash
namei -l /data/www/example/index.html
```

---

## Scenario 46: 403 Due to Lack of a Home Page in the Directory

If you request:

```text
http://example.com/docs/
```

and the directory exists at:

```text
/data/www/example/docs/
```

but there is no `index.html` file, and `autoindex` is disabled, you may receive a 403 Forbidden response.

Solutions include:

```text
Adding an index.html file

Returning a 404 status explicitly

Carefully enabling autoindex
```

---

## Chapter 19: Troubleshooting 404 Not Found

---

## Scenario 47: Common Causes of 404

Common causes include:

```text
The file truly does not exist

Incorrect root path configuration

Incorrect alias path configuration

Confusion between root and alias settings

Improper try_files configuration

SPA (Single Page Application) not redirecting to index.html

Request hitting the wrong server or location rule

Configuration files not being included correctly

Nginx not reloaded
```

---

## Scenario 48: Commands for Troubleshooting 404

To view the complete configuration:

```bash
nginx -T | grep -n "server_name example.com" -A 80
```

To check the root setting:

```bash
nginx -T | grep -n "root"
```

To check the alias setting:

```bash
nginx -T | grep -n "alias"
```

To check the location settings:

```bash
nginx -T | grep -n "location"
```

To confirm the existence of a file:

```bash
ls -lh /data/www/example/static/app.js
```

To test local access:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/static/app.js
```

To view access logs:

```bash
tail -n 100 /var/log/nginx/access.log
```

To view error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Chapter 20: Troubleshooting Ineffective Caching

---

## Scenario 49: Common Causes of Ineffective Caching

Common causes include:

```text
The location rule does not match the static resource path

The `add_header` directive is not included in the response

The request hits a different server

The file extension does not match the regular expression pattern

The browser forces a fresh refresh

A higher-level CDN service overwrites the cache headers```bash
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

## Creating Test Static Resources

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

## Log Inspection

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

## Basic Auth Configuration

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
curl -I -u admin:密码 http://example.com/download/
```

```bash
curl -I http://example.com/download/
```

---

## Checking forSensitive File Exposures

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

## Summary

The core configuration elements for Nginx static hosting include:

- `root`: Defines the root directory of the server.
- `alias`: Allows you to map a virtual path to a different location.
- `index`: Specifies the default file to serve when no matching files are found.
- `try_files`: Handles various scenarios when handling requests.
- `expires`: Sets cache expiration times for static resources.
- `Cache-Control`: Controls how caches are managed by clients and proxies.
- `autoindex`: Determines whether Nginx should automatically generate index pages.

Differences between `root` and `alias`:

- `root` directs all requests to the specified directory and its subdirectories