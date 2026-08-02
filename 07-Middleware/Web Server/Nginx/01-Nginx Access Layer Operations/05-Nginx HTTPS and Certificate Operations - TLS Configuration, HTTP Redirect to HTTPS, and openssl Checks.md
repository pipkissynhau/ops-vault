# 05-Nginx HTTPS and Certificate Operations: TLS Configuration, HTTP Redirect to HTTPS, and openssl Checks

#Nginx #HTTPS #TLS #SSL Certificates #Certificate Operations #OpenSSL #Web Servers #Access Layer #Middleware #Operations #SRE

---

## Recommended Path

07-Middleware/Web Servers/Nginx/01-Nginx Access Layer Operations/05-Nginx HTTPS and Certificate Operations: TLS Configuration, HTTP Redirect to HTTPS, and openssl Checks.md

---

## I. Document Description

This document compiles common configurations and troubleshooting methods for Nginx HTTPS and certificate operations.

Key points include:

- The role of HTTPS at the access layer
- Basic concepts of TLS/SSL
- Common types of certificate files
- `fullchain.pem` and `privkey.pem`
- Basic Nginx HTTPS configuration
- HTTP redirect to HTTPS
- Configuration of TLS protocol versions
- Basic settings for encryption suites
- Verification of certificate validity periods
- Checking certificate chains
- Ensuring domain name matches the certificate
- Common `openssl s_client` checks
- Issues with certificate permissions and file locations
- Certificate renewal processes
- Troubleshooting common HTTPS problems
- Best practices for production environments

This document is part of the Nginx Access Layer Operations series, Chapter 05.

Learning objectives:

```text
Be able to configure Nginx for HTTPS

→ Understand the functions of certificate and private key files

→ Configure HTTP automatic redirection to HTTPS

→ Use openssl to verify certificate validity and chains

→ Troubleshoot issues with certificate paths, permissions, expiration, and domain name mismatches

→ Establish a production process for certificate renewal and rollback
```

---

## II. The Role of HTTPS in Production

Typical HTTPS access chain:

```text
User's browser / Client

→ CDN / WAF / SLB

→ Nginx on port 443 (HTTPS)

→ Backend HTTP/HTTPS services
```

Common architectures:

```text
Client initiates HTTPS connection

→ Nginx handles TLS handshake

→ Forward request to backend HTTP service
```

Or:

```text
Client initiates HTTPS connection

→ Nginx performs HTTPS processing

→ Request is then forwarded to backend HTTPS service
```

Nginx's role in HTTPS scenarios:

```text
Receives HTTPS requests

Loads certificate and private key files

Completes TLS handshake

Deciphers client requests

Forwards them to the backend service

Records access logs

Configures HTTP-to-HTTPS redirection

Manages TLS protocol versions and encryption suites
```

In short:

```text
Nginx often serves as the entry point for HTTPS, handling certificate loading, TLS handshakes, and managing HTTPS traffic.
```

---

## III. Basic Concepts of TLS/SSL

Strictly speaking, TLS is now the primary protocol used in production.

However, many configurations, documents, and commands still refer to it as `ssl`.

Common terms:

```text
SSL
→ Legacy term; modern production should avoid using SSLv2/SSLv3

TLS
→ Modern secure transport protocol (e.g., TLSv1.2/TLSv1.3)

HTTPS
→ HTTP over TLS

Certificate
→ Used to verify domain identity

Private Key
→ Accompanies the certificate and must be kept secure

Certificate Chain
→ Sequence of certificates, including root certificates, for trust verification
```

Production recommendations:

```text
Disable SSLv2/SSLv3

 Disable TLSv1.0/TLSv1.1

 Prefer TLSv1.2 and TLSv1.3
```

---

## IV. Common Types of Certificate Files

Common certificate-related files include:

```text
.crt
→ Certificate file

.pem
→ PEM-encoded certificate or private key

.key
→ Private key file

.csr
→ Certificate Signing Request file

fullchain.pem
→ Server certificate plus intermediate certificates

privkey.pem
→ Private key file

chain.pem
→ Intermediate certificate chain

ca_bundle.crt
→ CA certificate chain
```

Nginx typically requires two files for basic configuration:

```text
ssl_certificate
→ Certificate file (preferably `fullchain.pem`)

ssl_certificate_key
→ Private key file
```

Example:

```nginx
ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;
```

---

## V. `fullchain.pem` and `privkey.pem`

---

## Scenario 1: What is `fullchain.pem`?

`fullchain.pem` usually contains:

```text
Server certificate

+

Intermediate CA certificates
```

Its purpose is to allow the client to fully verify the certificate chain.

If only a single-domain certificate is configured without intermediate certificates, it may result in:

```text
Some browsers will display the page normally.

Other clients will report an incomplete certificate chain.

`curl` commands will returnAvoid User Misaccess to HTTP

Enhance Security

Prevent Front-end Mixed Content IssuesmacOS and some systems have different parameters for the `date` command. In Linux operations environments, GNU `date` is commonly used.echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"

Check the response:

```bash
curl -I https://example.com
```

---

## Section 17: Certificate Rollback Process

---

## Scenario 35: How to Roll Back in Case of Certificate Updates Failing

If, after updating, the following issues occur:

```text
nginx -t fails

reload fails

The browser displays a certificate error

Client access fails

The certificate domain name does not match

The certificate chain is incorrect
```

Follow this rollback process:

```text
Restore the old fullchain.pem file

Restore the old privkey.pem file

Run nginx -t

Reload Nginx

Use openssl to check the online certificate

Verify access using curl
```

Example:

```bash
cp -a /etc/nginx/certs_backup/example.com/2026-04-25-100000/fullchain.pem /etc/nginx/certs/example.com/fullchain.pem
```

```bash
cp -a /etc/nginx/certs_backup/example.com/2026-04-25-100000/privkey.pem /etc/nginx/certs/example.com/privkey.pem
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
curl -I https://example.com
```

---

## Section 18: Configuring HTTPS for Multiple Domain Names

---

## Scenario 36: Using One Certificate to Cover Multiple Domain Names

If the certificate's SAN includes:

```text
example.com

www.example.com
```

You can configure it like this:

```nginx
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Check the SAN:

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Scenario 37: Using Different Certificates for Multiple Domain Names

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /etc/nginx/certs/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/app.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}

server {
    listen 443 ssl;
    server_name admin.example.com;

    ssl_certificate     /etc/nginx/certs/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/admin.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:9090;
    }
}
```

Note:

```text
The client tells the server which domain name to access via SNI

Nginx returns the corresponding certificate based on the server_name

When testing, use openssl with -servername
```

---

## Section 19: HTTPS to HTTPS Backend

---

## Scenario 38: Nginx Proxying to an HTTPS Backend

Configuration:

```nginx
location / {
    proxy_pass https://backend.example.com;

    proxy_ssl_server_name on;
    proxy_ssl_name backend.example.com;

    proxy_set_header Host backend.example.com;
}
```

Explanation:

```text
proxy_pass https://...
→ Nginx also uses HTTPS when connecting to the backend

proxy_ssl_server_name on
→ Enables SNI for the backend's HTTPS connection

proxy_ssl_name
→ Specifies the domain name used in the backend's TLS handshake
```

Suitable scenarios:

```text
The backend only provides HTTPS

Accessing the backend across networks

Encryption is required for the backend connection

The backend certificate needs to match SNI
```

---

## Section 20: Common Troubleshooting for HTTPS

---

## Scenario 40: The Browser Indicates That the Certificate Has Expired

Troubleshoot:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Possible causes:

```text
The certificate has indeed expired

Nginx has not been reloaded

The updated file is in the wrong location

Not all Nginx nodes have been updated

The CDN/SLB is still using the old certificate
`````bash
curl -I http://example.com
``````bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

```bash
namei -l /etc/nginx/certs/example.com/privkey.pem
```

---

## Certificate Backup

```bash
mkdir -p /etc/nginx/certs_backup/example.com/$(date +%F-%H%M%S)
```

```bash
cp -a /etc/nginx/certs/example.com/* /etc/nginx/certs_backup/example.com/$(date +%F-%H%M%S)/
```

```bash
find /etc/nginx/certs_backup/example.com -type f | tail
```

---

## Summary in One Sentence

The core of Nginx HTTPS configuration includes:

```text
listening on port 443 with SSL support;

setting the SSL certificate, private key, and enabling required protocols;

performing HTTP-to-HTTPS redirection and checking certificate validity and chain integrity;

verifying configurations after reloads.
```

Basic HTTPS configuration example:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

HTTP-to-HTTPS redirection:

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}
```

Recommended TLS protocols:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

Key certificate verification points:

```text
checking if the certificate is expired or contains the correct domain name;
verifying the integrity of the certificate chain;
ensuring the private key matches the certificate;
confirming Nginx has loaded the new certificates;
making sure all nodes have updated their configurations;
checking if CDN/SLB services use separate certificates.
```

Common verification commands:

```bash
openssl x509 -in fullchain.pem -noout -dates
```

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

```bash
curl -v https://example.com
```

Production recommendations:

```text
prefer using the fullchain.pem certificate;
strictly secure the private key and avoid storing it in Git;
back up existing certificates before updating them;
always perform `nginx -t` after updates;
use `openssl` to verify online certificates after reloading;
ensure all Nginx instances have updated their configurations;
monitor for certificate expiration in advance;
avoid using SSLv3/TLSv1.0/TLSv1.1;
if using CDN/SLB, ensure users see the correct layer of certificates.
```