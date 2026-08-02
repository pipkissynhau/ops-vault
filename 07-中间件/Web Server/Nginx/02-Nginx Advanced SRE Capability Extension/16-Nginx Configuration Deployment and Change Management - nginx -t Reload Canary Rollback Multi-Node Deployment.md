# 16-Nginx Configuration Deployment and Change Management: nginx -t, reload, Gray Release, Rollback, and Multi-Node Deployment

#Nginx #ConfigureReleases #ChangeManagement #GreyscaleRelease #RollBack #MultinodesRelease #nginx-t #reload #SRE #AccessLayerGovernance

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/16-Nginx Configuration Deployment and Change Management: nginx -t, reload, Gray Release, Rollback, and Multi-Node Deployment.md

---

## One: Document Explanation

This document organizes the Nginx configuration deployment, change management, gray validation, rollback, and multi-node deployment process.

Previously organized:

```text
Nginx Configure Foundation

Reverse Agent

upstream Load Balance

HTTPS Certificate

Real IP

JSON Log

Common failure screening

Performance capacity analysis

Flow and connection control

upstream High-level governance

High Available Structures

Secure.

Observation
```

This article focuses on:

- Why Nginx configuration changes need governance
- Nginx configuration file structure management
- Pre-change checks
- Configuration backup
- Full configuration snapshot
- `nginx -t`
- `nginx -T`
- `reload` vs `restart` distinction
- Smooth reload principle
- Single-node configuration deployment process
- Multi-node configuration deployment process
- Gray release of one Nginx
- Node-specific validation
- Certificate change deployment process
- upstream change deployment process
- Security policy change deployment process
- Rollback process
- Configuration version management
- Git management of Nginx configuration
- Basic automation deployment script
- CI/CD deployment thinking
- Common deployment troubleshooting
- Production considerations

This article is the 16th in the Nginx Advanced SRE Capabilities Expansion series.

This article's goal:

```text
Can be published according to standard process Nginx Configure

→ It makes a difference. reload and restart

→ Backup and check configuration before posting

→ It'll give you one of those. Nginx Revolume release

→ Yes. curl Specify node authentication

→ Quick Rollback Error Configuration

→ I can. Nginx Configure Inclusion Git Management

→ It can form production. Nginx Configure Changes SOP
```

---

## Two: Why Nginx Configuration Deployment Needs Governance

Nginx typically resides in the business entry layer.

A single configuration error may lead to:

```text
The entire domain name is not accessible

HTTPS Certificate Loading Failed

HTTP Jump abnormal.

Invert proxy path error

Backend upstream It's not working.

Real IP Parsing error

White List Error User

Influencing operations

Static resources 404

SPA Refresh 404

Log format error

Prometheus Monitor aborted.

Multiple Nginx Configuration Inconsistent
```

Common dangerous operations in production:

```text
Direct vi Change production configuration

Do Not Backup

Not implemented nginx -t

Direct restart Nginx

All but one.

Only one certificate is updated

Modified Not include Other Organiser

reload Losing but not finding out.

I don't know how to roll back after the online problems.
```

One-sentence understanding:

```text
Nginx The configuration release is not a simple change of document, but a change of governance at the entrance level.
```

---

## Three: Nginx Configuration Deployment Principles

Recommended principles for production Nginx configuration deployment:

```text
Backup first

Change

Check first.

Reload

Grayscale first

Revolume

Authenticate first

It's over again.

Traceable

But roll back
```

Standard process:

```text
Confirmation of change requirements

→ Backup Current Configuration

→ Modify Configuration

→ nginx -t Check Syntax:

→ nginx -T Confirm full configuration

→ Grayscale release of one

→ curl Specify node authentication

→ Observation access.log / error.log

→ Batch release of remaining nodes

→ Full Authentication

→ Record changes

→ Keep Rollback Path
```

---

## Four: Nginx Configuration File Structure Management

---

## Scenario 1: Common Configuration Structure

Common structure:

```text
/etc/nginx/
├── nginx.conf
├── conf.d/
│   ├── example.com.conf
│   ├── admin.example.com.conf
│   └── api.example.com.conf
├── certs/
│   └── example.com/
│       ├── fullchain.pem
│       └── privkey.pem
└── snippets/
    ├── proxy-headers.conf
    ├── security-headers.conf
    └── ssl-common.conf
```

Recommendation:

```text
nginx.conf
→ Set global configuration and include

conf.d/
→ Business. server Configure

certs/
→ Could not close temporary folder: %s

snippets/
→ Reusable Snippets

backup/
→ Not recommended include Within Path
```

---

## Scenario 2: View include Relationships

View includes in the complete configuration:

```bash
nginx -T | grep -n "include"
```

View main configuration:

```bash
sed -n '1,200p' /etc/nginx/nginx.conf
```

View conf.d files:

```bash
ls -lh /etc/nginx/conf.d/
```

---

## Scenario 3: Do Not Place Backup Files in include Matching Scope

If the main configuration contains:

```nginx
include /etc/nginx/conf.d/*.conf;
```

Then these files will be loaded:

```text
example.com.conf

example.com.conf.bak.conf

test.conf

old.conf
```

Risks:

```text
Backup files are also available. Nginx Load

Repeat server_name

Repeat listen

Reactivate old configuration

Checking difficulties
```

Not recommended:

```text
/etc/nginx/conf.d/example.com.conf.bak.conf
```

Recommended backup to:

```text
/tmp/nginx-backup/

or

/etc/nginx/backup/
```

But ensure it is not included.

---

## Five: Pre-Change Checks

---

## Scenario 4: Confirm Current Nginx Status

```bash
systemctl status nginx
```

Check processes:

```bash
ps -ef | grep nginx | grep -v grep
```

Check listening ports:

```bash
ss -lntp | grep nginx
```

Check recent errors:

```bash
journalctl -u nginx -n 100
```

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 5: Confirm Current Configuration Syntax is Normal

Before changing, execute:

```bash
nginx -t
```

If the current configuration is already abnormal, do not proceed with deployment.

Need to first clarify:

```text
Whether or not to run the old configuration on the current line

Has the profile been corrupted?

reload Have you ever failed?

Is the current process using an old configuration
```

---

## Scenario 6: Save Current Full Configuration Snapshot

```bash
nginx -T > /tmp/nginx-full-config-before-$(date +%F-%H%M%S).txt 2>&1
```

View snapshot:

```bash
ls -lh /tmp/nginx-full-config-before-*.txt
```

Purpose:

```text
Keep fully effective configuration before change

Easy. diff

It's easy to roll back.

It's easy to reset a malfunction.
```

---

## Six: Configuration Backup

---

## Scenario 7: Backup Single Business Configuration

```bash
cp -a /etc/nginx/conf.d/example.com.conf /tmp/example.com.conf.$(date +%F-%H%M%S).bak
```

View backup:

```bash
ls -lh /tmp/example.com.conf.*.bak
```

---

## Scenario 8: Backup Entire Nginx Configuration Directory

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

View backup:

```bash
ls -lh /tmp/nginx-conf-backup-*.tar.gz
```

---

## Scenario 9: Backup Certificate Directory

```bash
tar czf /tmp/nginx-certs-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx/certs
```

View backup:

```bash
ls -lh /tmp/nginx-certs-backup-*.tar.gz
```

---

## Seven: Post-Configuration Checks

---

## Scenario 10: Check Syntax

After modifying the configuration, must execute:

```bash
nginx -t
```

Successful example:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If failed, cannot reload.

---

## Scenario 11: View Full Effective Configuration

```bash
nginx -T > /tmp/nginx-full-config-after-$(date +%F-%H%M%S).txt 2>&1
```

View for a specific domain:

```bash
nginx -T 2>/dev/null | grep -n "server_name example.com" -A 80
```

View upstream:

```bash
nginx -T 2>/dev/null | grep -n "upstream app_backend" -A 30
```

View certificate configuration:

```bash
nginx -T 2>/dev/null | grep -n "ssl_certificate" -A 5 -B 5
```

View rate-limiting configuration:

```bash
nginx -T 2>/dev/null | grep -n "limit_req" -A 10 -B 5
```

---

## Scenario 12: Compare Before and After Configuration Changes

```bash
diff -u /tmp/nginx-full-config-before-2026-04-25-100000.txt /tmp/nginx-full-config-after-2026-04-25-101000.txt | less
```

If only comparing a specific file:

```bash
diff -u /tmp/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

---

## Eight: reload vs restart Difference

---

## Scenario 13: What is reload

Execute:

```bash
systemctl reload nginx
```

Or:

```bash
nginx -s reload
```

Meaning:

```text
Reload Configuration

Start New worker Use new configuration

Old worker Exit after processing existing connections

Usually do not actively interrupt existing connections
```

Suitable for:

```text
Modify server Configure

Modify upstream

Modify location

Update Certificate

Modify Log Format

Modify Security Response Header

Adjust restricted stream configuration
```

---

## Scenario 14: What is restart

Execute:

```bash
systemctl restart nginx
```

Meaning:

```text
Stop Nginx

Start again. Nginx
```

Risks:

```text
Possible disconnection

Short Time Not Available

If there is a problem with the new configuration, it may fail to start.

Impact ratio reload Larger
```

In production, prioritize:

```bash
systemctl reload nginx
```

Only consider restart in special scenarios.

---

## Scenario 15: When Might restart Be Needed

Scenarios that may require restart:

```text
systemd LimitNOFILE modified

Nginx Binary Upgrade

Partial global process-level parameter changes

Dynamic module loading adjustments

worker Process cannot be smoothed out

Very few. reload Failure to meet expectations
```

But before production restart, confirm:

```text
Change Window

High Available Nodes

Rollback Programme

Health screening

Scope of impact
```

---

## Nine: Smooth reload Basic Understanding

---

## Scenario 16: Nginx reload General Process

Nginx reload general process:

```text
master Copy that. reload Signal

→ master Check for new configuration

→ master Start New worker

→ New worker Accept new requests using the new configuration

→ Old worker Stop receiving new requests

→ Old worker Exit after processing existing connections
```

Check master and worker:

```bash
ps -ef | grep nginx | grep -v grep
```

---

## Scenario 17: Check Process Changes After reload

Before reload:

```bash
ps -ef | grep "nginx: worker" | grep -v grep
```

After reload:

```bash
systemctl reload nginx
```

After reload:

```bash
ps -ef | grep "nginx: worker" | grep -v grep
```

Check logs:

```bash
journalctl -u nginx -n 50
```

```bash
tail -n 50 /var/log/nginx/error.log
```

---

## Ten: Single-Node Deployment Process

---

## Scenario 18: Standard Single-Node Deployment Steps

```text
Confirm Current Status

→ Backup Configuration

→ Modify Configuration

→ nginx -t

→ nginx -T Confirm Configuration

→ systemctl reload nginx

→ curl Organisation

→ curl Assign Host Authentication

→ View access.log

→ View error.log

→ Record changes
```

---

## Scenario 19: Single-Node Deployment Command Example

Confirm status:

```bash
systemctl status nginx
```

Backup:

```bash
cp -a /etc/nginx/conf.d/example.com.conf /tmp/example.com.conf.$(date +%F-%H%M%S).bak
```

Check configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Local validation:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

HTTPS validation:

```bash
curl -k -I --resolve example.com:443:127.0.0.1 https://example.com/
```

Check error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Eleven: Multi-Node Deployment Process

---

## Scenario 20: Why Multi-Node Can't Be Fully Changed at Once

If there are multiple Nginx instances:

```text
Nginx-1:10.0.0.21

Nginx-2:10.0.0.22

Nginx-3:10.0.0.23
```

Not recommended to reload all at once.

Reason:

```text
If there's a problem with the configuration, it's likely the whole station will fail at the same time.

If there's a problem with the certificate, all nodes HTTPS Failed

If the white list is wrong, all nodes are wrong.

If the flow is too tight, all nodes together. 429

If the path goes wrong, all nodes together. 404 / 502
```

Recommendation:

```text
It's a gray one.

Authentication no problem

Revolume release of remaining nodes
```

---

## Scenario 21: Standard Multi-Node Deployment Process

```text
Prepare configuration for local or managed nodes

→ Sync to First Nginx

→ First one. nginx -t

→ First one. reload

→ Specify First IP Authentication

→ Watch Log

→ Confirm no anomaly.

→ Batch Sync Surplus Nodes

→ Each nginx -t

→ Each reload

→ Each assigned IP Authentication

→ Full observation surveillance and logs
```

---

## Scenario 22: Gray Release to One Node

Synchronize configuration to the first node:

```bash
rsync -avz /etc/nginx/conf.d/example.com.conf root@10.0.0.21:/etc/nginx/conf.d/example.com.conf
```

Check and reload on the first node:

```bash
ssh root@10.0.0.21 "nginx -t && systemctl reload nginx"
```

Specify the first node for HTTP verification:

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com/
```

Specify the first node for HTTPS verification:

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com/
```

Check error logs on the first node:

```bash
ssh root@10.0.0.21 "tail -n 100 /var/log/nginx/error.log"
```

---

## Scenario 23: Batch Deployment of Remaining Nodes

```bash
for host in 10.0.0.22 10.0.0.23; do
    echo "===== deploy $host ====="
    rsync -avz /etc/nginx/conf.d/example.com.conf root@$host:/etc/nginx/conf.d/example.com.conf
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Scenario 24: Batch Verification of All Nodes

HTTP:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTP ====="
    curl -I --resolve example.com:80:$host http://example.com/
done
```

HTTPS:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTPS ====="
    curl -I --resolve example.com:443:$host https://example.com/
done
```

---

## Twelve. Node-Specific Verification Methods

---

## Scenario 25: Why Node-Specific Verification is Needed

If there is an SLB in front:

```text
User

→ SLB

→ Multiple Nginx
```

Direct access to the domain:

```bash
curl -I https://example.com
```

You may not know which Nginx node the request lands on.

Node-specific verification is required during deployment:

```bash
curl --resolve example.com:443:10.0.0.21 https://example.com/
```

This bypasses DNS resolution and directly accesses the specified Nginx node.

---

## Scenario 26: Specify HTTP Node

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com/
```

---

## Scenario 27: Specify HTTPS Node

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com/
```

---

## Scenario 28: Specify Host for Local Verification

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

HTTPS is recommended:

```bash
curl -I --resolve example.com:443:127.0.0.1 https://example.com/
```

---

## Thirteen. Certificate Change Deployment Process

---

## Scenario 29: Pre-Deployment Certificate Check

Check the validity period of the new certificate:

```bash
openssl x509 -in fullchain.pem -noout -dates
```

Check SAN:

```bash
openssl x509 -in fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

Check if the certificate and private key match:

```bash
openssl x509 -noout -modulus -in fullchain.pem | openssl md5
```

```bash
openssl rsa -noout -modulus -in privkey.pem | openssl md5
```

The two outputs should be consistent.

---

## Scenario 30: Backup Old Certificates

```bash
mkdir -p /tmp/cert-backup/example.com/$(date +%F-%H%M%S)
```

```bash
cp -a /etc/nginx/certs/example.com/* /tmp/cert-backup/example.com/$(date +%F-%H%M%S)/
```

---

## Scenario 31: Replace Certificates

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

Check configuration:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

---

## Scenario 32: Post-Certificate Change Verification

Check the validity period of the certificate in production:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

Check SAN:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

Check HTTPS:

```bash
curl -I https://example.com/
```

Specify node check:

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com/
```

---

## Fourteen. Upstream Change Deployment Process

---

## Scenario 33: Pre-Deployment Check for New Upstream Node

Check backend port:

```bash
nc -zv -w 2 10.0.0.22 8080
```

Check health interface:

```bash
curl -v http://10.0.0.22:8080/health
```

Check version:

```bash
curl -s http://10.0.0.22:8080/version
```

---

## Scenario 34: Gray Release of Upstream

Start with low weight:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=9;
    server 10.0.0.22:8080 weight=1;
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

Observe upstream distribution:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

Observe 5xx upstream errors:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

---

## Scenario 35: Upstream Rollback for Abnormalities

If new nodes are abnormal, temporarily remove them:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=1;
    server 10.0.0.22:8080 down;
}
```

Check and reload:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## Fifteen. Security Policy Change Deployment Process

---

## Scenario 36: Why Caution is Needed for Security Policy Changes

Security policies include:

```text
allow / deny

real_ip

limit_req

limit_conn

HSTS

CSP

Host Intercept.

Sensitive path intercept.

Limitations on the method of request

Basic Auth
```

These configurations may inadvertently affect business operations.

Common incidents:

```text
White List misconfiguration prevents users from accessing

real_ip The error led to all users being treated as SLB IP

limit_req It's too much. 429

HSTS includeSubDomains Causing sub-domain names abnormal.

CSP Overwhelming front white screens

Host Interception leading to rejection of joint jurisdiction

OPTIONS Interception leads to cross-border failure.
```

---

## Scenario 37: Recommendations for Security Policy Deployment

```text
Test Environment Validation

→ One greyscale. Nginx

→ Specify node authentication

→ Observation 403 / 404 / 429

→ Watch frontend

→ Observation API Call

→ It's confirmed and then released in bulk.
```

Check status code distribution:

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

Check 403:

```bash
awk '$9 == 403 {print $1,$7,$9}' /var/log/nginx/access.log | tail -n 50
```

Check 429:

```bash
awk '$9 == 429 {print $1,$7,$9}' /var/log/nginx/access.log | tail -n 50
```

---

## Sixteen. Rollback Process

---

## Scenario 38: Single File Rollback

Check backup:

```bash
ls -lh /tmp/example.com.conf.*.bak
```

Rollback:

```bash
cp -a /tmp/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
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
curl -I -H "Host: example.com" http://127.0.0.1/
```

---

## Scenario 39: Full Directory Rollback

Check backup package:

```bash
ls -lh /tmp/nginx-conf-backup-*.tar.gz
```

Extract to temporary directory and check:

```bash
mkdir -p /tmp/nginx-rollback-check
```

```bash
tar xzf /tmp/nginx-conf-backup-2026-04-25-100000.tar.gz -C /tmp/nginx-rollback-check
```

Confirm content before recovery:

```bash
cp -a /tmp/nginx-rollback-check/etc/nginx/* /etc/nginx/
```

Check:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

---

## Scenario 40: Multi-Node Rollback

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== rollback $host ====="
    scp /tmp/example.com.conf.2026-04-25-100000.bak root@$host:/etc/nginx/conf.d/example.com.conf
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

Verify:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host ====="
    curl -I --resolve example.com:80:$host http://example.com/
done
```

---

## Seventeen. Configuration Version Management

---

## Scenario 41: Why Nginx Configurations Should Be in Git

Benefits of putting Nginx configurations into Git:

```text
Know who changed what?

Know when to change.

Yeah. code review

We can roll back fast.

Portfolio audit.

Accessible. CI Inspection

Unique release of multiple nodes

It avoids manual manipulation.
```

Not recommended:

```text
Long-term only on production machines. vi Modify

No records submitted

Nothing. review

No Rollback

Multiple machine manual sync
```

---

## Scenario 42: Recommended Git Directory Structure

```text
nginx-config-repo/
├── README.md
├── nginx.conf
├── conf.d/
│   ├── example.com.conf
│   ├── admin.example.com.conf
│   └── api.example.com.conf
├── snippets/
│   ├── proxy-headers.conf
│   ├── security-headers.conf
│   └── ssl-common.conf
└── scripts/
    ├── check-nginx.sh
    ├── deploy-one.sh
    └── deploy-all.sh
```

Note:

```text
Certificate Private Key Do Not Enter Git

Do not enter the code. Git

.htpasswd Do not recommend explicit entry Git

Sensitive configuration requires a secure key management scheme
```

---

## Scenario 43: Git Commit Workflow Example

```bash
git checkout -b change-nginx-example-com
```

```bash
git status
```

```bash
git diff
```

```bash
git add conf.d/example.com.conf
```

```bash
git commit -m "update nginx example.com upstream config"
```

```bash
git push origin change-nginx-example-com
```

Production recommendation:

```text
Pass. Pull Request / Merge Request review Later.
```

---

## Eighteen. Automation Check Script

---

## Scenario 44: Local Check Script

Script path:

```bash
vi scripts/check-nginx.sh
```

Script content:

```bash
#!/bin/bash

set -e

echo "===== nginx syntax check ====="
nginx -t

echo "===== nginx full config check ====="
nginx -T >/tmp/nginx-full-config-check.txt 2>&1

echo "===== check result ====="
echo "nginx config check passed"
```

Permissions:

```bash
chmod +x scripts/check-nginx.sh
```

Execute:

```bash
./scripts/check-nginx.sh
```

---

## Scenario 45: Remote Batch Check Script

Script path:

```bash
vi scripts/check-all-nginx.sh
```

Script content:

```bash
#!/bin/bash

set -e

HOSTS="10.0.0.21 10.0.0.22 10.0.0.23"

for host in $HOSTS; do
    echo "===== check $host ====="
    ssh root@$host "nginx -t"
done
```

Permissions:

```bash
chmod +x scripts/check-all-nginx.sh
```

Execute:

```bash
./scripts/check-all-nginx.sh
```

---

## Nineteen. Automation Deployment Script Basics

---

## Scenario 46: Deploy to Single Node

Script path:

```bash
vi scripts/deploy-one.sh
```

Script content:

```bash
#!/bin/bash

set -e

HOST="$1"

if [ -z "$HOST" ]; then
    echo "usage: $0 <host>"
    exit 1
fi

echo "===== deploy to $HOST ====="

ssh root@$HOST "mkdir -p /tmp/nginx-backup"

ssh root@$HOST "tar czf /tmp/nginx-backup/nginx-conf-backup-\$(date +%F-%H%M%S).tar.gz /etc/nginx"

rsync -avz conf.d/ root@$HOST:/etc/nginx/conf.d/
rsync -avz snippets/ root@$HOST:/etc/nginx/snippets/

ssh root@$HOST "nginx -t && systemctl reload nginx"

echo "===== deploy finished: $HOST ====="
```

Permissions:

```bash
chmod +x scripts/deploy-one.sh
```

Execute:

```bash
./scripts/deploy-one.sh 10.0.0.21
```

---

## Scenario 47: Batch Deployment Script

Script path:

```bash
vi scripts/deploy-all.sh
```

Script content:

```bash
#!/bin/bash

set -e

HOSTS="10.0.0.21 10.0.0.22 10.0.0.23"

for host in $HOSTS; do
    echo "===== deploy $host ====="

    ssh root@$host "mkdir -p /tmp/nginx-backup"
    ssh root@$host "tar czf /tmp/nginx-backup/nginx-conf-backup-\$(date +%F-%H%M%S).tar.gz /etc/nginx"

    rsync -avz conf.d/ root@$host:/etc/nginx/conf.d/
    rsync -avz snippets/ root@$host:/etc/nginx/snippets/

    ssh root@$host "nginx -t && systemctl reload nginx"
done

echo "===== all deploy finished ====="
```

Permissions:

```bash
chmod +x scripts/deploy-all.sh
```

Execute:

```bash
./scripts/deploy-all.sh
```

---

## Twenty. CI/CD Deployment Strategy

---

## Scenario 48: CI Check Content for Nginx Configuration

Recommended checks during CI phase:

```text
Configure File Syntax:

Whether sensitive documents exist

Whether or not to wrongly submit a private key

Whether to include errors include

Is it dangerous? set_real_ip_from 0.0.0.0/0

Is it too strict? HSTS preload

Include repetition server_name

Whether to include obvious errors upstream

Compatibility with naming norms
```

---

## Scenario 49: Use Container for Configuration Syntax Check

You can use an Nginx image mounted with the configuration for checking.

Example:

```bash
docker run --rm \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/conf.d:/etc/nginx/conf.d:ro \
  -v $(pwd)/snippets:/etc/nginx/snippets:ro \
  nginx:stable \
  nginx -t
```

Note:

```text
If configuration depends on certificate path, need CI provides a test certificate or mock Path

If configuration relies on custom modules,CI Mirror must contain corresponding modules

Otherwise, the results and production are not consistent.
```

---

## Scenario 50: Recommended CI/CD Deployment Phases

Recommended deployment phase:

```text
Manual approval

→ One greyscale.

→ Auto nginx -t

→ reload

→ Auto curl Authentication

→ Autoview exporter / blackbox Status

→ Manual confirmation

→ Batch release of remaining nodes

→ Automatically verify all nodes

→ Output release record
```

---

## Twenty-One. Common Deployment Troubleshooting

---

## Scenario 51: nginx -t Fails

Common causes:

```text
Minimal

Unmatched parenthesis

Command error

Command Context Error

include File does not exist

Certificate path does not exist

Certificate does not match private key

Log directory does not exist

Port repeated listening

Module does not exist
```

Check the line near the error:

```bash
sed -n '10,30p' /etc/nginx/conf.d/example.com.conf
```

View full error message:

```bash
nginx -t
```

---

## Scenario 52: reload executed but configuration not taking effect

Common causes:

```text
Error file modified

The file was not taken include

nginx -t It's a failure.reload Actual unsuccessful

He's got another hit. server

He's got another hit. location

Changed. HTTPActual visits HTTPS

Changed one. Visited the other one.

Front CDN / SLB Cache

Browser Cache
```

Troubleshooting:

```bash
nginx -T | grep -n "example.com" -A 80
```

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com/
```

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com/
```

Check logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Scenario 53: reload failed but old service still running

When Nginx reload fails, old worker processes may still be running.

Phenomenon:

```text
Business is normal.

But the new configuration didn't work.

journalctl or error.log Yes. reload Failed Record
```

Troubleshooting:

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
tail -n 100 /var/log/nginx/error.log
```

```bash
nginx -t
```

Resolution:

```text
Don't think reload It'll work.

Must read the return results and logs

Must verify that the configuration is actually effective
```

---

## Scenario 54: partial user anomalies after deployment

Common causes:

```text
Multiple Nginx Configuration Inconsistent

Only partial certificate error

Only partial nodes upstream Wrong match.

SLB To the anomaly.

Some station. Nginx Nope. reload

Some station. Nginx real_ip Configure Different

Some station. Nginx The white list is different.
```

Verify per node:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host ====="
    curl -I --resolve example.com:443:$host https://example.com/
done
```

Check configuration per node:

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== $host ====="
    ssh root@$host "nginx -T 2>/dev/null | grep -n 'server_name example.com' -A 40"
done
```

---

## Scenario 55: increase in 502 errors after deployment

Troubleshooting:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -Ei "connect\(\) failed|connection refused|no live upstreams|upstream prematurely closed" /var/log/nginx/error.log | tail -n 100
```

Check upstream:

```bash
nginx -T | grep -n "upstream app_backend" -A 30
```

Check backend:

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

---

## Scenario 56: increase in 429 errors after deployment

Troubleshoot rate limiting configuration:

```bash
nginx -T | grep -n "limit_req" -A 10 -B 5
```

Stat 429 errors:

```bash
awk '$9 == 429 {print $1,$7,$9}' /var/log/nginx/access.log | tail -n 50
```

Stat 429 IPs:

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

Focus on:

```text
Real IP Correct?

rate Is it too low?

burst Is it too small?

Do you have any health checks?

Whether the login interface is fully installed
```

---

## Twenty-two, Production Deployment Notes

---

## 1. Must backup before deployment

At least backup:

```text
Change files

Full /etc/nginx

Certificate Directory

nginx -T Output
```

---

## 2. Must run nginx -t before reload

Fixed workflow:

```bash
nginx -t
```

```bash
systemctl reload nginx
```

Cannot skip.

---

## 3. Multi-node deployments must be gradual

Do not deploy all Nginx nodes simultaneously.

Recommendation:

```text
First one.

Authentication

Revolume
```

---

## 4. Verification must specify nodes

Use:

```bash
curl --resolve
```

Avoid SLB random forwarding interference.

---

## 5. Certificate changes must be verified per node

After certificate deployment, check:

```text
Valid period

SAN

Certificate Chain

Nginx Whether or not reload

Can you get a new certificate online?

Consistency of multiple nodes
```

---

## 6. Security policy changes require extra caution

Especially:

```text
real_ip

allow / deny

limit_req

limit_conn

HSTS

CSP

Host Intercept.

Limitations on the method of request
```

These configurations are prone to accidental damage.

---

## 7. Configuration deployment must have rollback time targets

Before deployment, must know:

```text
Where's the rollback file?

What's going on?

Who executes rollback?

How to verify when rolling back

How long do you expect to recover?
```

---

## 8. Configuration must be in Git

Production recommendation:

```text
Git Management

Change review

Autocheck

Greyscale Release

Autovalidation

But roll back
```

---

## 9. Do not test unfamiliar configurations directly in production

Unfamiliar configurations should first be tested in:

```text
Test Environment

Advance Environment

Single Test Nginx

Container environment
```

Then moved to production.

---

## 10. Must observe after deployment

At least observe:

```text
access.log

error.log

5xx

499

429

request_time

upstream_response_time

blackbox Search.

Prometheus exporter Status

Operational feedback
```

---

## Twenty-three, Common Commands Summary

---

## Status Check

```bash
systemctl status nginx
```

```bash
ps -ef | grep nginx | grep -v grep
```

```bash
ss -lntp | grep nginx
```

```bash
journalctl -u nginx -n 100
```

```bash
tail -n 100 /var/log/nginx/error.log
```

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

§

---

## Backup

```bash
cp -a /etc/nginx/conf.d/example.com.conf /tmp/example.com.conf.$(date +%F-%H%M%S).bak
```

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

```bash
tar czf /tmp/nginx-certs-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx/certs
```

---

## reload / restart

```bash
systemctl reload nginx
```

```bash
nginx -s reload
```

```bash
systemctl restart nginx
```

```bash
journalctl -u nginx -n 100
```

---

## Local Verification

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -I --resolve example.com:443:127.0.0.1 https://example.com/
```

---

## Node-specific Verification

```bash
curl -I --resolve example.com:80:10.0.0.21 http://example.com/
```

```bash
curl -I --resolve example.com:443:10.0.0.21 https://example.com/
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTP ====="
    curl -I --resolve example.com:80:$host http://example.com/
done
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== check $host HTTPS ====="
    curl -I --resolve example.com:443:$host https://example.com/
done
```

---

## Multi-node Deployment

```bash
rsync -avz /etc/nginx/conf.d/example.com.conf root@10.0.0.21:/etc/nginx/conf.d/example.com.conf
```

```bash
ssh root@10.0.0.21 "nginx -t && systemctl reload nginx"
```

```bash
for host in 10.0.0.22 10.0.0.23; do
    echo "===== deploy $host ====="
    rsync -avz /etc/nginx/conf.d/example.com.conf root@$host:/etc/nginx/conf.d/example.com.conf
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Certificate Check

```bash
openssl x509 -in fullchain.pem -noout -dates
```

```bash
openssl x509 -in fullchain.pem -noout -text | grep -A 2 "Subject Alternative Name"
```

```bash
openssl x509 -noout -modulus -in fullchain.pem | openssl md5
```

```bash
openssl rsa -noout -modulus -in privkey.pem | openssl md5
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

---

## upstream Check

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
curl -v http://10.0.0.22:8080/health
```

```bash
curl -s http://10.0.0.22:8080/version
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

---

## Log Observation

```bash
tail -f /var/log/nginx/access.log
```

```bash
tail -f /var/log/nginx/error.log
```

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 429 {print $1,$7,$9}' /var/log/nginx/access.log | tail -n 50
```

---

## Rollback

```bash
cp -a /tmp/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

```bash
nginx -t
```

```bash
systemctl reload nginx
```

```bash
for host in 10.0.0.21 10.0.0.22 10.0.0.23; do
    echo "===== rollback $host ====="
    scp /tmp/example.com.conf.2026-04-25-100000.bak root@$host:/etc/nginx/conf.d/example.com.conf
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Twenty-four, One-sentence Summary

The core of Nginx configuration deployment is not "just reload", but:

```text
Backup

Inspection

Greyscale

Authentication

Batch

Observation

Roll back

Records
```

Production fixed workflow:

```text
Backup Configuration

→ Modify Configuration

→ nginx -t

→ nginx -T

→ One greyscale.

→ curl --resolve Specify node authentication

→ Observation logs and monitoring

→ Batch release of remaining nodes

→ Full Authentication

→ Keep Rollback Path
```

Difference between `reload` and `restart`:

```text
reload
→ Smooth loading of new configurations, usually without interrupting existing connections

restart
→ Stop and start again.
```

Multi-node deployment principle:

```text
Do not publish it in full at the same time

It's a gray one.

Revolume release

Every one. nginx -t

Every one. reload

Each one has been assigned. IP Authentication
```

Rollback principle:

```text
Backup must be available before posting

I have to roll back. nginx -t

I have to roll back. reload

You have to verify it when rolling back.

Multinodes must be rolled back together.
```

Production recommendations:

```text
Nginx Configure Inclusion Git

Certificate Private Key Do Not Enter Git

Release script to autobackup

CI Phase nginx -t

Greyscale validation during distribution

The security strategy must be carefully changed.

upstream Change must be observed. 5xx and upstream_addr

Change of certificate must be checked for validity,SAN And certificate chain

All releases must be traceable, roll-backable, reset.
```