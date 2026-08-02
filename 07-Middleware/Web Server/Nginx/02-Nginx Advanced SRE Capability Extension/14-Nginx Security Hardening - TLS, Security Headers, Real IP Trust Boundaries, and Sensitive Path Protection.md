```nginx
ssl_protocols TLSv1.0 TLSv1.1;
```

---

## Scenario 11 - Enforce TLS 1.2 or Higher

Configuration:

```nginx
ssl_certificate_key /etc/nginx/certs/default/privkey.pem;
ssl_certificate /etc/nginx/certs/default/fullchain.pem;
ssl_protocols TLSv1.2 TLSv1.3;
```

Description:

```text
Force clients to use TLS 1.2 or higher for connections.
```

---

## Scenario 12 - Disable TLS 1.1 and Lower

Configuration:

```nginx
ssl_protocols TLSv1.0 TLSv1.1;
```

Note:

```text
Disabling TLS 1.1 and lower may affect compatibility.
```

---

## 8. HTTPS Force Redirect

If Nginx is configured as an HTTP server, the following configuration can be used to force visitors to switch to HTTPS:

```nginx
location / {
    return 301 https://example.com;
}
```

Or:

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://example.com;
}
```

Verification:

```bash
curl -I http://127.0.0.1
```

Or：

```bash
curl -v -H "Host: example.com" http://127.0.0.1
```

---

## 9. HSTS Configuration

HSTS (HTTP Strict Transport Security) is a security-enhancing mechanism that helps prevent redirect attacks and cache pollution.

Configuration example：

```nginx
http {
    header set HSTS "max-age=3600;
    header set max-age-in-subdomain=1800;
}
```

Description：

```text
Set the HSTS header with a maximum validity period of 3600 seconds, and 1800 seconds for subdomains.
```

---

## 10. Common Security Headers

Some common security headers include:

- `Content-Security-Policy`: used to control resource security
- `X-Frame-Options`: prevents cross-site framing attacks
- `Referrer-Policy`: restricts referrer information
- `Access-Control-Allow-Origin`: specifies allowed domains for access
- `Access-Control-Allow-Headers`: specifies allowed request header fields

Configuration example:

```nginx
http {
    header set Content-Security-Policy "policy=strict;
    header set X-Frame-Options "same-origin";
}
```

Description：

```text
Set the Content-Security-Policy header to prevent cross-site framing attacks; set the X-Frame-Options header to restrict the use of cross-site framing.
```

---

## 11. CSP Basics

CSP (Content Security Policy) is a policy used to control resource security.

Configuration example:

```nginx
http {
    header set Content-Security-Policy "policy=strict;
    header set Content-Security-Allow-Origin 'https://example.com';
}
```

Description：

```text
Set the Content-Security-Policy header to prevent cross-site framing attacks; allow resources to be loaded from https://example.com.
```

---

## 12. Preventing Exposure of Sensitive Files

Sensitive files can be protected from public access by configuring file permissions and using regular expression matching.

For example, to block access to `.git` and `.env` files:

```nginx
http {
    deny all;
    allow only root;
    file_type denied .git .env;
}
```

Or：

```nginx
location /git {
    deny all;
}
```

Description：

```text
Use "deny all" and "allow only root" to block all users from accessing .git and .env files; configure location /git to specifically block access to the /git directory.
```

---

## 13. Blocking Access to .git, .env, and Backup Files

In addition to the above configuration, access to other sensitive files such as backup files can also be blocked.

For example:

```nginx
http {
    deny all;
    allow only root;
    file_type denied .git .env backup;
}
```

Or：

```nginx
location /backup {
    deny all;
}
```

Description：

```text
Use "deny all" and "allow only root" to block all users from accessing .backup files; configure location /backup to specifically block access to the /backup directory.
```

---

## 14. Real IP Trust Boundary

To ensure that only genuine IP addresses can access the server, the `set_real_ip_from` parameter can be configured.

Configuration example:

```nginx
http {
    set_real_ip_from 127.0.0.1;
}
```

Description：

```text
Set the set_real_ip_from parameter to 127.0.0.1 to ensure that only the local IP address can access the server.
```

---

## 15. Admin Backend Access Control

To enhance the security of the admin backend, access can be protected by configuring basic authentication or using HTTPS.

For example, configuring basic authentication:

```nginx
http {
nginx
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
```

Reason:

```text
SSLv2 / SSLv3 are no longer secure.

TLSv1.0 / TLSv1.1 are no longer suitable as modern production baselines.

TLSv1.2 / TLSv1.3 are common choices for production environments.
```

---

## Scenario 11: Basic HTTPS Security Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Explanation:

```text
ssl_protocols
→ Limits the versions of the TLS protocol.

ssl_ciphers
→ Limits cipher suites to TLSv1.2 and below.

ssl_prefer_server_ciphers
→ Determines whether to prefer server-defined cipher suites.
```

---

## Scenario 12: Checking TLS Protocols

To check for TLSv1.2:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2
```

To check for TLSv1.3:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```

To check for TLSv1.0:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1
```

If TLSv1.0 is disabled, the connection should fail or fail to negotiate successfully.

---

## Scenario 13: Checking Online Protocols and Ciphers

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E "Protocol|Cipher"
```

---

## VIII. Forcing HTTP Requests to HTTPS

---

## Scenario 14: HTTP 301 Redirect to HTTPS

Configuration:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    return 301 https://$host$request_uri;
}
```

Function:

```text
Permanently redirects HTTP requests to HTTPS.
```

Verification:

```bash
curl -I http://example.com
```

Expected result:

```text
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

---

## Scenario 15: Precautions for HTTP Redirects

It is necessary to confirm the following:

```text
HTTPS is available.
The certificate is valid.
All subdomains support HTTPS.
The backend correctly recognizes X-Forwarded-Proto.
CDN/SLB also have HTTPS configurations.
There are no HTTP callback interfaces.
```

Do not blindly force all pages to use HTTPS before it is fully stable.

---

## IX. HSTS Configuration

---

## Scenario 16: What is HSTS?

HSTS response header:

```text
Strict-Transport-Security
```

Function:

```text
Instrues browsers to always use HTTPS when accessing this site.
```

Basic configuration:

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

Explanation:

```text
max-age=31536000
→ The browser will remember this setting for one year.

always
→ This header is added regardless of whether the response is 2xx or 3xx.
```

---

## Scenario 17: Complete HSTS Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000" always;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Verification:

```bash
curl -I https://example.com
```

Expected output:

```text
Strict-Transport-Security: max-age=31536000
```

---

## Scenario 18: Caution When Using includeSubDomains and preload

Stronger configuration:

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

Meaning:

```text
includeSubDomains
→ Forces all subdomains to use HTTPS as well.

preload
→ Allows the```markdown
add_header Content-Security-Policy "default-src 'self'; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'; script-src 'self'; object-src 'none'; frame-ancestors 'self';" always;
```

---

## Scenario 25: Be Cautious When Configuring CSP

Excessively strict CSP configurations may lead to the following issues:

```text
Front-end JavaScript cannot be loaded.
CSS cannot be loaded.
Images cannot be loaded.
Third-party SDKs fail to function.
CAPTCHAs stop working.
Statistics scripts become ineffective.
Map components no longer work.
```

Production recommendations include:

```text
First, verify in a testing environment.
Start with a more conservative approach.
If necessary, use Content-Security-Policy-Report-Only for observation.
Do not directly copy strict CSP settings from the internet into production.
```

Example of Report-Only configuration:

```nginx
add_header Content-Security-Policy-Report-Only "default-src 'self';" always;
```

---

## XII. Protecting Sensitive Paths

---

## Scenario 26: Prevent Access to Hidden Files

Configuration:

```nginx
location ~\. {
    deny all;
}
```

Files that can be blocked include:

```text
.git
.env
.svn
.htaccess
.htpasswd
```

Verification methods:

```bash
curl -I http://example.com/.git/config
```

```bash
curl -I http://example.com/.env
```

Expected response:

```text
403 Forbidden

or

404 Not Found
```

---

## Scenario 27: Specifically Prohibit Access to .git Files

More precise configuration:

```nginx
location ~* /\.git {
    return 404;
}
```

It is recommended to return a 404 response:

```text
Do not reveal whether this directory exists.
```

Verification method:

```bash
curl -I http://example.com/.git/config
```

---

## Scenario 28: Block Access to Environment Files

Configuration:

```nginx
location ~* /\.(env|ini|conf)$ {
    return 404;
}
```

Or:

```nginx
location ~* \.(env|ini|conf)$ {
    return 404;
}
```

Note:

```text
Rules should be tailored to the actual business needs.
If your business does provide .conf file downloads, make sure not to mistakenly block them.
```

A more fundamental principle is:

```text
Never place sensitive files in the web root directory.
```

---

## Scenario 29: Block Access to Backup Files and Compressed Packages

Configuration:

```nginx
location ~* \.(bak|backup|old|orig|save|swp|sql|tar|tar.gz|tgz|zip|rar|7z)$ {
    return 404;
}
```

This configuration helps prevent the exposure of:

```text
Database backups
Old configurations
Compressed files
Temporary files
Editor swap files
```

Note:

```text
If your business requires zip file downloads, do not block them across the entire site.
You can enable this setting only in the web root directory or for specific sites.
```

---

## Scenario 30: Block Access to Sensitive Directories

Configuration:

```nginx
location ^~ /private/ {
    return 404;
}

location ^~ /config/ {
    return 404;
}

location ^~ /backup/ {
    return 404;
}
```

---

## XIII. Disabling autoindexing

---

## Scenario 31: Turn Off Directory Browsing

Configuration:

```nginx
autoindex off;
```

Example:

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/example;
    autoindex off;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Reasons for this:

```text
To prevent the exposure of directory file listings.
To avoid exposing backup files.
To prevent revealing the structure of upload directories.
To stop scanners from enumerating files.
```

Verification method:

```bash
curl -I http://example.com/some-dir/
```

---

## XIV. Trusting Real IP Addresses at the Boundary

---

## Scenario 32: Security Issues Related to Real IP Addresses

Many security strategies rely on the client's IP address:

```text
allow / deny
limit_req
limit_conn
Audit logs
IP blocking
Whitelist for management consoles
```

If the real IP configuration is incorrect, it may result in:

```text
Bypassing of the whitelist.
Failure of rate limiting measures.
Incorrectly blocking legitimate users.
Distortion of audit log data.
All users being treated as SLB IPs.
```

---

## Scenario 33: Dangerous```nginx
server {
    listen 80 default_server;
    server_name _;

    return 444;
}

server {
    listen 80;
    server_name example.com www.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    sslcertificate_key /etc/nginx/certs/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;

    server_tokens off;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    client_max_body_size 20m;
    client_header_timeout 10s;
    client_body_timeout 60s;
    send_timeout 60s;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location ~ /\. {
        return 404;
    }

    location ~* \.(bak|backup|old|orig|save|swp|sql|tar|tar.gz|tgz|rar|7z)$ {
```return 404;
}

location ^~ /private/ {
    return 404;
}

location / {
    proxy_pass http://app_backend;

    proxy_http_version 1.1;
    proxy_set_header Connection "";

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
```First, observe using Report-Only mode.

Gradually enable access based on the actual source of each service.

Do not directly copy the strict template into production environment.## 25. In One Sentence

The core of Nginx security reinforcement includes:

- Reducing exposed vulnerabilities
- Strengthening data transmission
- Limiting access points
- Protecting sensitive directories
- Cleansing real IP addresses
- Recording security logs
- Controlling configuration changes

Basic security settings:

```nginx
server_tokens off;
ssl_protocols TLSv1.2 TLSv1.3;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

Protection for sensitive directories:

```nginx
location ~ /\. {
    return 404;
}

location ~* \.(bak|backup|old|sql|tar|tar.gz|zip)$ {
    return 404;
}
```

Principles for real IP security:

- Only trust trusted proxies.
- Do not set `real_ip_from` to `0.0.0.0/0`.
- The origin server should not allow direct public network access without going through a proxy.
- The backend should not unconditionally trust the `X-Forwarded-For` header.

Suggestions for protecting the administration interface:

- Use an IP whitelist.
- Implement Basic Auth or formal authentication methods.
- Consider using VPN or zero-trust security models.
- Apply rate limiting and access logging.

Production considerations:

- Be cautious when using `HSTS includeSubDomains` or `preload`.
- Thoroughly test any CSP (Content Security Policy) changes before deploying them.
- Ensure that `Host` directives are correctly configured in conjunction with `default_server`.
- Do not store sensitive files in the web root directory.
- Never commit private keys to Git repositories.
- Always verify security configurations using `nginx -t` before deployment.
- Make sure that settings are synchronized across multiple nodes and that certificates are updated consistently.
- All security changes must be reversible.