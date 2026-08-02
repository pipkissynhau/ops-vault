# 05-Nginx HTTPS and Certificate Operations: TLS Configuration, HTTP Redirect to HTTPS, and openssl Checks

#Nginx #HTTPS #TLS #SslCertificate #CertificateTransport #OpenSSL #WebServer #AccessLayer #Middle #Transport #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Operations/05-Nginx HTTPS and Certificate Operations: TLS Configuration, HTTP Redirect to HTTPS and openssl Checks.md

---

## One: Document Description

This document organizes common configurations and troubleshooting methods for Nginx HTTPS and certificate operations.

This article focuses on:

- The role of HTTPS in the access layer
- TLS/SSL basic concepts
- Common certificate file types
- `fullchain.pem` and `privkey.pem`
- Nginx HTTPS basic configuration
- HTTP redirect to HTTPS
- TLS protocol version configuration
- Basic encryption suite configuration
- Certificate validity check
- Certificate chain check
- Domain and certificate matching check
- `openssl s_client` common troubleshooting
- Certificate permissions and path issues
- Certificate update process
- Common HTTPS troubleshooting
- Production environment considerations

This article is the 05th article of the Nginx access layer operations series.

This article's goals:

```text
Configure Nginx HTTPS

→ Understand certificate and private key files

→ Configure HTTP Autojump HTTPS

→ It works. openssl Check certificate validity and certificate chain

→ Could check certificate path, permissions, expiry, domain name mismatch

→ Could form a production certificate update and rollback process
```

---

## Two: HTTPS Position in Production

Typical HTTPS access chain:

```text
User Browser / Client

→ CDN / WAF / SLB

→ Nginx 443 HTTPS

→ Backend HTTP / HTTPS Services
```

Common architecture:

```text
Client HTTPS

→ Nginx Termination TLS

→ Backend HTTP
```

It could also be:

```text
Client HTTPS

→ Nginx HTTPS

→ Backend HTTPS
```

Nginx's role in HTTPS scenarios:

```text
Received HTTPS Request

Load certificates and private keys

Completed TLS Shake hands.

Decrypt Client Request

Forward to Backend Service

Record Access Log

Configure HTTP Present. HTTPS Jump

Control TLS Versions and encryption packages
```

One-sentence understanding:

```text
Nginx Often. HTTPS The entrance level.TLS Shake hands and... HTTPS Traffic access.
```

---

## Three: TLS/SSL Basic Concepts

Strictly speaking, TLS is primarily used in production now.

However, many configurations, documents, and commands still use the term `ssl`.

Common terminology:

```text
SSL
→ Old names, historical protocols, modern production should not be used anymore. SSLv2 / SSLv3

TLS
→ Modern secure transmission protocols, for example TLSv1.2 / TLSv1.3

HTTPS
→ HTTP over TLS

Certificate
→ To prove domain name identity

Private Key
→ The certificate must be strictly protected.

Certificate Chain
→ Server Certificate + Intermediate Certificate + Root Certificate Trust Chain
```

Production recommendations:

```text
Disable SSLv2 / SSLv3

Disable TLSv1.0 / TLSv1.1

Enable Priority TLSv1.2 and TLSv1.3
```

---

## Four: Common Certificate File Types

Common certificate-related files:

```text
.crt
→ Certificate File

.pem
→ PEM Format Certificate or Private Key

.key
→ Private key file

.csr
→ Request file for certificate signature

fullchain.pem
→ Server Certificate + Intermediate certificate chain

privkey.pem
→ Private key file

chain.pem
→ Intermediate certificate chain

ca_bundle.crt
→ CA Certificate Chain
```

Nginx common configurations typically require two files:

```text
ssl_certificate
→ Certificate file, recommended fullchain

ssl_certificate_key
→ Private key file
```

Example:

```nginx
ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;
```

---

## Five: fullchain.pem and privkey.pem

---

## Scenario 1: What is fullchain.pem

`fullchain.pem` typically contains:

```text
Server Certificate

+

Centre CA Certificate
```

Function:

```text
To enable the client to fully verify the certificate chain
```

If only a single site certificate is configured without intermediate certificates, it may lead to:

```text
Partial browser access normal

Partial client certificate chain incomplete

curl Report unable to get local issuer certificate

Old system or Java Client TLS Validation Failed
```

Production recommendations:

```text
Nginx Yes. ssl_certificate Use Priority fullchain.pem
```

---

## Scenario 2: What is privkey.pem

`privkey.pem` is the private key file.

Features:

```text
Must match the certificate.

It must be strictly protected.

No leaks.

Could not close temporary folder: %s Git Warehouse

Cannot initialise Evolution's mail component.

We'll keep our privileges as tight as we can.
```

Check permissions:

```bash
ls -l /etc/nginx/certs/example.com/privkey.pem
```

Recommended permissions:

```bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

Confirm owner based on Nginx startup method:

```bash
ps -ef | grep nginx | grep -v grep
```

---

## Six: Nginx HTTPS Basic Configuration

---

## Scenario 3: Minimum HTTPS Configuration

Configuration file:

```bash
vi /etc/nginx/conf.d/example.com.conf
```

Configuration content:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
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
curl -I -k -H "Host: example.com" https://127.0.0.1
```

Remote verification:

```bash
curl -I https://example.com
```

---

## Scenario 4: HTTPS Reverse Proxy Complete Basic Configuration

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;

    keepalive 64;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 443;

        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

Description:

```text
listen 443 ssl
→ Listen HTTPS 443 Port

ssl_certificate
→ Load certificate chain

ssl_certificate_key
→ Load Private Keys

X-Forwarded-Proto https
→ Tell backend original request HTTPS

X-Forwarded-Port 443
→ Tell the backend to the original port. 443
```

---

## Seven: HTTP Redirect to HTTPS

---

## Scenario 5: Why Configure HTTP Redirect to HTTPS

In production, it's typically desired:

```text
User access http://example.com

Automatically Jump to https://example.com
```

Reasons:

```text
Harmonization HTTPS Entry

Reduction of express HTTP Visits

Avoid user error access HTTP

Increased security

Avoid front-end mixing content problems
```

---

## Scenario 6: HTTP 301 Redirect to HTTPS

Configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}
```

Description:

```text
301
→ Redirect permanently

$host
→ Request Host

$request_uri
→ Original URI, with path and query parameters
```

Example:

```text
http://example.com/api/users?id=1

Jump to:

https://example.com/api/users?id=1
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
curl -I http://example.com
```

Expected to see:

```text
HTTP/1.1 301 Moved Permanently
Location: https://example.com/
```

---

## Scenario 7: Complete HTTP and HTTPS Configuration

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## Eight: TLS Protocol Version Configuration

---

## Scenario 8: Configure TLSv1.2 and TLSv1.3

Recommended configuration:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

Complete example:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

Not recommended:

```nginx
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
```

Reason:

```text
SSLv3I don't know.TLSv1.0I don't know.TLSv1.1 No longer suitable for modern production safety baselines
```

---

## Scenario 9: Check Supported TLS Protocols

Test TLSv1.2:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2
```

Test TLSv1.3:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```

Test TLSv1.0:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1
```

If TLSv1.0 is disabled, the connection should fail or negotiation should fail.

---

## Nine: Basic Encryption Suite Configuration

---

## Scenario 10: Basic ssl_ciphers Configuration

Example:

```nginx
ssl_ciphers HIGH:!aNULL:!MD5;
ssl_prefer_server_ciphers off;
```

Description:

```text
ssl_ciphers
→ Assign TLSv1.2 Encryption packages up to

ssl_prefer_server_ciphers
→ Prefer Service Encryption Package Order
```

Note:

```text
TLSv1.3 Yes. cipher suite Not exactly. ssl_ciphers Control

Different. Nginx / OpenSSL Version support capacity is different
```

Production recommendations:

```text
Don't copy it over the Internet. cipher Configure

To combine the system. OpenSSL Version, client compatibility and company security baseline validation
```

Check OpenSSL version:

```bash
openssl version
```

Check Nginx compiled OpenSSL information:

```bash
nginx -V 2>&1 | grep -i openssl
```

---

## Ten: Certificate Path and Permissions Management

---

## Scenario 11: Recommended Certificate Directory Structure

Recommended directory:

```text
/etc/nginx/certs/example.com/
├── fullchain.pem
└── privkey.pem
```

Create directory:

```bash
mkdir -p /etc/nginx/certs/example.com
```

Check files:

```bash
ls -lh /etc/nginx/certs/example.com/
```

Set private key permissions:

```bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

Set certificate permissions:

```bash
chmod 644 /etc/nginx/certs/example.com/fullchain.pem
```

---

## Scenario 12: Certificate File Missing Causes Nginx Startup Failure

Common error:

```text
cannot load certificate

BIO_new_file() failed

No such file or directory
```

Troubleshoot:

```bash
nginx -t
```

```bash
ls -lh /etc/nginx/certs/example.com/fullchain.pem
```

```bash
ls -lh /etc/nginx/certs/example.com/privkey.pem
```

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 13: Certificate Permission Issues

Common error:

```text
Permission denied
```

Troubleshoot:

```bash
ls -l /etc/nginx/certs/example.com/
```

```bash
namei -l /etc/nginx/certs/example.com/privkey.pem
```

Description:

```text
namei -l You can view path privileges by level
```

---

## Eleven: Check Certificate Validity

---

## Scenario 14: Check Local Certificate Validity

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates
```

Output example:

```text
notBefore=Apr 25 00:00:00 2026 GMT
notAfter=Jul 24 23:59:59 2026 GMT
```

Only check expiration time:

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -enddate
```

---

## Scenario 15: Check Online Domain Certificate Validity

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Only check expiration time:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate
```

---

## Scenario 16: Check Certificate Remaining Days

```bash
end_date=$(echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
end_ts=$(date -d "$end_date" +%s)
now_ts=$(date +%s)
echo $(( (end_ts - now_ts) / 86400 ))
```

Description:

```text
How much of the output certificate has expired? days
```

Note:

```text
macOS and selected systems date Different command parameters

Linux The environment is usually used. GNU date
```

---

## Twelve: Check Certificate Domain Matching

---

## Scenario 17: View Certificate Subject

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -subject
```

---

## Scenario 18: View Certificate SAN /think

Modern certificates primarily focus on SAN, which stands for Subject Alternative Name.

Commands:

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

Output may include:

```text
DNS:example.com, DNS:www.example.com
```

Explanation:

```text
Access domain names must be SAN Medium

Just looking. CN Not enough.
```

---

## Scenario 19: Online Certificate Domain Check

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -subject
```

Check SAN:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Thirteen. Check Certificate and Private Key Match

---

## Scenario 20: Use modulus to Check Certificate and Private Key

Check certificate:

```bash
openssl x509 -noout -modulus -in /etc/nginx/certs/example.com/fullchain.pem | openssl md5
```

Check private key:

```bash
openssl rsa -noout -modulus -in /etc/nginx/certs/example.com/privkey.pem | openssl md5
```

If the two outputs are identical, it indicates the certificate and private key match.

---

## Scenario 21: Private Key with Password Causes Nginx Startup Issues

If the private key is encrypted, Nginx may require interactive password input during startup or reload.

In production, it's generally not recommended to use encrypted private keys with Nginx.

Check private key:

```bash
openssl rsa -in /etc/nginx/certs/example.com/privkey.pem -check -noout
```

If prompted for a password, it indicates the private key may be encrypted.

---

## Fourteen. Common openssl s_client Checks

---

## Scenario 22: Check TLS Handshake Information

```bash
openssl s_client -connect example.com:443 -servername example.com
```

Focus on:

```text
Certificate chain

Server certificate

SSL handshake

Protocol

Cipher

Verify return code
```

---

## Scenario 23: Check Certificate Chain Validity

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

Focus on:

```text
Certificate chain

Verify return code: 0 (ok)
```

If you see:

```text
Verify return code: 21

unable to verify the first certificate
```

It may indicate:

```text
Certificate chain incomplete

Missing intermediate certificate

Clients don't trust that. CA

fullchain Malformed configuration
```

---

## Scenario 24: Check Online Protocols and Cipher Suites

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E "Protocol|Cipher"
```

Example output:

```text
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
```

---

## Scenario 25: Check Without SNI

```bash
openssl s_client -connect example.com:443
```

With SNI:

```bash
openssl s_client -connect example.com:443 -servername example.com
```

In production, it's recommended to use `-servername`.

Reason:

```text
One. Nginx Possible Multiple Configurations HTTPS Domain Name

The service needs to be based on SNI Return correct certificate

No, I don't. SNI Could get Default Certificate
```

---

## Fifteen. curl HTTPS Checks

---

## Scenario 26: Check HTTPS Response Headers

```bash
curl -I https://example.com
```

---

## Scenario 27: Display Detailed TLS Process

```bash
curl -v https://example.com
```

You can see:

```text
TLS Protocol Version

Certificate Subject

Certificate SAN

Certificate Validity Period

HTTP Response Header
```

---

## Scenario 28: Ignore Certificate Verification Test

```bash
curl -k -I https://example.com
```

Explanation:

```text
-k
→ Ignore Certificate Validation
```

Suitable for:

```text
Temporary test visa

Current IP Test HTTPS

Exclusion of application layers
```

Production note:

```text
Don't. -k Use it as a normal way of visiting.

-k It's just a check.
```

---

## Scenario 29: Test Local HTTPS by Specifying Host

When DNS hasn't resolved to the new machine yet, you can test locally.

```bash
curl -k -I -H "Host: example.com" https://127.0.0.1
```

It's more recommended to use `--resolve` to simulate DNS:

```bash
curl -I --resolve example.com:443:127.0.0.1 https://example.com
```

Test remote specified IP:

```bash
curl -I --resolve example.com:443:10.0.0.20 https://example.com
```

Explanation:

```text
--resolve You can specify a domain name to resolve to a certain IP

Fit to certificate and SNI Test

More than simple. -H Host Closer to reality. HTTPS Visits
```

---

## Sixteen. Certificate Update Process

---

## Scenario 30: Standard Certificate Update Process

Recommended production certificate update process:

```text
Confirm new certificate file

→ Check certificate validity period

→ Check certificate domain name SAN

→ Checks whether certificates match private keys

→ Backup old certificates

→ Replace Certificate File

→ nginx -t

→ reload Nginx

→ openssl Check Online Certificates

→ curl Authentication HTTPS

→ Observation error.log

→ Record certificate due date
```

---

## Scenario 31: Backup Old Certificate

```bash
mkdir -p /etc/nginx/certs_backup/example.com/$(date +%F-%H%M%S)
```

```bash
cp -a /etc/nginx/certs/example.com/* /etc/nginx/certs_backup/example.com/$(date +%F-%H%M%S)/
```

Check backup:

```bash
find /etc/nginx/certs_backup/example.com -type f | tail
```

---

## Scenario 32: Replace Certificate Files

```bash
cp -a fullchain.pem /etc/nginx/certs/example.com/fullchain.pem
```

```bash
cp -a privkey.pem /etc/nginx/certs/example.com/privkey.pem
```

Set permissions:

```bash
chmod 644 /etc/nginx/certs/example.com/fullchain.pem
```

```bash
chmod 600 /etc/nginx/certs/example.com/privkey.pem
```

---

## Scenario 33: Check Nginx After Update

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Check status:

```bash
systemctl status nginx
```

Check error log:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 34: Check Online Certificate After Update

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Check SAN:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

Check response:

```bash
curl -I https://example.com
```

---

## Seventeen. Certificate Rollback Process

---

## Scenario 35: Rollback Certificate Update Issues

If after update you encounter:

```text
nginx -t Failed

reload Failed

Browser Note Certificate Error

Client access failed

Certificate domain not matching

Certificate chain anomaly
```

Rollback process:

```text
Restore old fullchain.pem

Restore old privkey.pem

nginx -t

reload Nginx

openssl Check Online Certificates

curl Authentication Access
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

## Eighteen. Multi-Domain HTTPS Configuration

---

## Scenario 36: One Certificate Covers Multiple Domains

If the certificate SAN includes:

```text
example.com

www.example.com
```

You can configure:

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

Check SAN:

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Scenario 37: Multiple Domains Use Different Certificates

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
Client through SNI Tell the service to access which domain name

Nginx Based on server_name Return corresponding certificate

When testing openssl We have to. -servername
```

---

## Nineteen. HTTPS to Backend HTTPS

---

## Scenario 38: Nginx Proxy to HTTPS Backend

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
→ Nginx Use backend also HTTPS

proxy_ssl_server_name on
→ Enable to Backend HTTPS Yes. SNI

proxy_ssl_name
→ Specify Backend TLS Domain name for handshake
```

Suitable scenarios:

```text
Backend Only HTTPS

Cross-network access backend

There are encryption requirements for back-end links.

Backend certificate required SNI Match
```

---

## Scenario 39: Troubleshoot Backend HTTPS Certificate Issues

Test backend:

```bash
openssl s_client -connect backend.example.com:443 -servername backend.example.com
```

curl backend:

```bash
curl -v https://backend.example.com
```

Nginx error log:

```bash
tail -n 100 /var/log/nginx/error.log
```

Common issues:

```text
Backend Certificate Expiry

Backend certificate domain not matched

Backend certificate chain incomplete

Nginx No tape at the back end. SNI

Backend only supports specified TLS Version
```

---

## Twenty. Common HTTPS Troubleshooting

---

## Scenario 40: Browser Prompt for Expired Certificate

Troubleshoot:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Possible causes:

```text
The certificate does expire.

Nginx Nope. reload

Update to Error Path

Multiple Nginx Node Unupdated

Front CDN / SLB Still use old certificates
```

---

## Scenario 41: Browser Prompt for Mismatched Certificate Domain

Troubleshoot SAN:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

Possible causes:

§

## Scenario 42: curl Prompt for "unable to get local issuer certificate"

Troubleshoot:

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

Possible causes:

```text
Certificate chain incomplete

Nginx Configures separate site certificates instead of fullchain

Client missing root certificate

Error certificate order
```

Handling direction:

```text
ssl_certificate Use fullchain.pem

Confirm certificate chain integrity

Restart reload Nginx

Use openssl Inspection Verify return code
```

---

## Scenario 43: Nginx reload Failed

Troubleshoot:

```bash
nginx -t
```

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
tail -n 100 /var/log/nginx/error.log
```

Common causes:

```text
Certificate path does not exist

Private key path does not exist

Certificate does not match private key

Private key privileges are not valid

Certificate file format error

listen 443 ssl Writing Error

443 Port occupied
```

---

## Scenario 44: 443 Port Unreachable

Troubleshoot:

```bash
ss -lntp | grep ':443'
```

```bash
systemctl status nginx
```

```bash
nginx -t
```

Local test:

```bash
curl -k -I https://127.0.0.1
```

Remote test:

```bash
nc -zv -w 2 example.com 443
```

Packet capture:

```bash
tcpdump -i any -nn port 443
```

Possible causes:

```text
Nginx No listening. 443

Security team not released. 443

Firewall not released 443

SLB Not listening 443

DNS Pointing error IP

Request to another node
```

---

## Scenario 45: HTTP No Redirect to HTTPS

Troubleshoot:

```bash
curl -I http://example.com
```

Check configuration:

```bash
nginx -T | grep -n "return 301" -A 5 -B 5
```

```bash
nginx -T | grep -n "listen 80" -A 20
```

Possible causes:

```text
80 server Other Organiser

He's got another hit. server

server_name Do not match

Front CDN / SLB Got it. HTTP

Nginx Not reload

Browser Cache Interference
```

---

## Twenty-One. Production Notes

---

## 1. Do Not Store Certificates and Private Keys in Git

Private keys are high-sensitivity information.

Should not be placed in:

```text
Git Warehouse

Open Object Storage

Share Disk

Unencrypted Backup

Chat Tool Express Transfer
```

---

## 2. Recommend Using fullchain for ssl_certificate

Recommended:

```nginx
ssl_certificate /etc/nginx/certs/example.com/fullchain.pem;
```

Avoid configuring only individual site certificates, as it may result in an incomplete certificate chain.

---

## 3. Must Run nginx -t Before reload

After updating the certificate, you must execute:

```bash
nginx -t
```

Then execute:

```bash
systemctl reload nginx
```

---

## 4. All Nodes in Multi-Node Nginx Must Be Updated

If there are multiple Nginx instances:

```text
Nginx-1

Nginx-2

Nginx-3
```

Ensure that:

```text
Every certificate updated

Every one. nginx -t

Every one. reload

Validation of certificates per unit
```

Otherwise, you may encounter:

```text
Visit new certificates sometimes

Sometimes accessing old certificates

Some users still report certificate expiration
```

---

## 5. Note CDN / SLB Certificates

If the chain is:

```text
User

→ CDN / SLB

→ Nginx
```

There may be two layers of certificates:

```text
CDN / SLB Uplink Certificate

Nginx Up Source Certificate
```

The certificate typically seen by the user's browser is the CDN / SLB certificate.

Therefore, when the certificate expires, confirm that:

```text
Which level of certificate expired?
```

---

## 6. Monitor Certificate Expiration in Advance

Recommended:

```text
Early 30 Chile

Early 15 God damn it!

Early 7 God damn it!
```

Do not wait for user reports to discover certificate expiration.

---

## 7. Do Not Enable Outdated TLS Protocols

Not recommended:

```nginx
ssl_protocols SSLv3 TLSv1 TLSv1.1;
```

Recommended:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

---

## 8. Pay Attention to SNI for HTTPS Backend

If Nginx proxies to an HTTPS backend, recommend configuring:

```nginx
proxy_ssl_server_name on;
proxy_ssl_name backend.example.com;
```

Otherwise, the backend may return a default certificate or fail the handshake.

---

## Twenty-Two, Summary of Common Commands

---

## Nginx Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "ssl_certificate" -A 5 -B 5
```

```bash
nginx -T | grep -n "listen 443" -A 20
```

```bash
nginx -T | grep -n "return 301" -A 5 -B 5
```

---

## Service Management

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
journalctl -u nginx -n 100
```

---

## Port Check

```bash
ss -lntp | grep ':443'
```

```bash
ss -lntp | grep ':80'
```

```bash
nc -zv -w 2 example.com 443
```

```bash
tcpdump -i any -nn port 443
```

---

## Local Certificate Check

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -dates
```

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -enddate
```

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -subject
```

```bash
openssl x509 -in /etc/nginx/certs/example.com/fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## Certificate and Private Key Match Check

```bash
openssl x509 -noout -modulus -in /etc/nginx/certs/example.com/fullchain.pem | openssl md5
```

```bash
openssl rsa -noout -modulus -in /etc/nginx/certs/example.com/privkey.pem | openssl md5
```

```bash
openssl rsa -in /etc/nginx/certs/example.com/privkey.pem -check -noout
```

---

## Online Certificate Check

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -subject
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

---

## openssl s_client

```bash
openssl s_client -connect example.com:443 -servername example.com
```

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2
```

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_3
```

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | grep -E "Protocol|Cipher"
```

---

## curl HTTPS Check

```bash
curl -I https://example.com
```

```bash
curl -v https://example.com
```

```bash
curl -k -I https://example.com
```

```bash
curl -k -I -H "Host: example.com" https://127.0.0.1
```

```bash
curl -I --resolve example.com:443:127.0.0.1 https://example.com
```

```bash
curl -I --resolve example.com:443:10.0.0.20 https://example.com
```

---

## HTTP Redirect Check

```bash
curl -I http://example.com
```

```bash
curl -L -I http://example.com
```

---

## Certificate Directory and Permissions

```bash
mkdir -p /etc/nginx/certs/example.com
```

```bash
ls -lh /etc/nginx/certs/example.com/
```

```bash
chmod 644 /etc/nginx/certs/example.com/fullchain.pem
```

```bash
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

## Twenty-Three, One-Sentence Summary

The core of Nginx HTTPS configuration is:

```text
listen 443 ssl

ssl_certificate

ssl_certificate_key

ssl_protocols

HTTP Jump HTTPS

Certificate validity check

Certificate Chain Check

reload Backline Authentication
```

Basic HTTPS configuration:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

HTTP redirect to HTTPS:

```nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}
```

Recommended TLS protocol:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

Certificate check focus:

```text
Expiry of certificate

Could not close temporary folder: %s

Complete certificate chain

Whether certificates match private keys

Nginx Whether new certificates are loaded

Do multiple nodes all update

CDN / SLB Is there a separate certificate?
```

Common check commands:

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
ssl_certificate Use Priority fullchain.pem

The private key must be strictly protected. Git

Backup before certificate updates

Update must nginx -t

reload Afterward. openssl Check Online Certificates

Multiple Nginx Must Update All

Certificate expiration must be monitored in advance

Do Not Enable SSLv3 / TLSv1.0 / TLSv1.1

If there's one in front... CDN / SLBCheck which level the user sees.
```