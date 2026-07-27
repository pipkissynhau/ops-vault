# 16-Nginx Configuration Deployment and Change Management: nginx -t, reload, Canary Release, Rollback, and Multi-Node Deployment

#Nginx #Configuration Deployment #Change Management #Canary Release #Rollback #Multi-Node Deployment #nginx-t #reload #SRE #Access Layer Governance

---

## Recommended Path

07-Middleware/Web Servers/Nginx/02-Nginx Advanced SRE Capability Expansion/16-Nginx Configuration Deployment and Change Management: nginx -t, reload, Canary Release, Rollback, and Multi-Node Deployment.md

---

## I. Document Overview

This document outlines the processes for deploying Nginx configurations, managing changes, performing canary releases, rolling back changes, and deploying configurations across multiple nodes.

Earlier documents have covered:

```text
Nginx Configuration Basics

Reverse Proxy

upstream Load Balancing

HTTPS Certificates

Real IP Addresses

JSON Logs

Common Troubleshooting

Performance and Capacity Analysis

Throttling and Connection Control

Advanced upstream Governance

High Availability Architecture

Security Enhancements

Observability
```

This document focuses on:

- Why governance is necessary for Nginx configuration changes
- Managing the structure of Nginx configuration files
- Pre-change checks
- Configuration backups
- Creating complete configuration snapshots
- Using `nginx -t` and `nginx -T`
- Distinguishing between `reload` and `restart`
- The principles behind smooth reloading
- The process of deploying configurations on a single node
- The process of deploying configurations across multiple nodes
- Performing canary releases with one Nginx server
- Verifying configurations on specific nodes
- The process of releasing certificate changes
- The process of releasing upstream configuration changes
- The process of releasing security policy changes
- The rollback process
- Configuration version management
- Using Git to manage Nginx configurations
- Basics of automated deployment scripts
- CI/CD deployment approaches
- Common issues in deployment
- Production considerations

This document is part of the 16th installment in the Nginx Advanced SRE Capability Expansion series.

The objectives of this document are:

```text
To be able to deploy Nginx configurations following standard processes

→ To distinguish between `reload` and `restart`

→ To back up and check configurations before deployment

→ To perform canary releases on one Nginx server before deploying in batches

→ To verify configurations on specific nodes using curl

→ To quickly roll back incorrect configurations

→ To incorporate Nginx configuration management into Git

→ To establish a standard operating procedure for changing production Nginx configurations
```

---

## II. Why Governance is Necessary for Nginx Configuration Changes

Nginx often serves as the entry point for business services.

A single configuration error can lead to:

```text
Inaccessibility of the entire domain name

Failure to load HTTPS certificates

Abnormal HTTP redirects

Incorrect reverse proxy settings

Disconnection with upstream servers

Errors in real IP address resolution

Unintended blockage of users by allowlists

Damage to business operations due to improper throttling

404 errors for static resources

404 errors when refreshing SPA pages

Incorrect log formats

 Interruption in Prometheus monitoring

Inconsistent configurations across multiple Nginx servers
```

Common risky practices in production include:

```text
Directly editing production configurations using vi

Failing to back up the configuration

Skipping the `nginx -t` command

Directly restarting Nginx

Only modifying one node out of multiple ones

Only updating one certificate

Modifying files that are not included in the configuration

Failing to notice a reload failure

Not knowing how to roll back changes after online issues arise
```

In summary:

```text
Deploying Nginx configurations is not just about simply editing files; it requires proper governance at the entry point level.
```

---

## III. Principles for Deploying Nginx Configurations

It is recommended that production Nginx configuration deployments follow these principles:

```text
Back up the configuration first

Make changes afterwards

Check the configuration before proceeding

Reload the configuration after making changes

Perform canary releases first

Deploy in batches later

Verify the configurations thoroughly

Complete the process before finishing

Ensure traceability

Enable rollback capabilities
```

The standard process is as follows:

```text
Confirm the change requirements

→ Back up the current configuration

→ Make the necessary changes

→ Use `nginx -t` to check the syntax

→ Use `nginx -T` to verify the complete configuration

→ Perform a canary release on one server

→ Use curl to verify configurations on specific nodes

→ Monitor access.log and error.log files

→ Deploy the changes across the remaining servers

→ Verify all configurations again

→ Record the changes made

→ Maintain a path for rolling back changes
```

---

## IV. Managing the Structure of Nginx Configuration Files

---

## Scenario 1: Common## Scenario 12: Comparing Configurations Before and After Changes

```bash
diff -u /tmp/nginx-full-config-before-2026-04-25-100000.txt /tmp/nginx-full-config-after-2026-04-25-101000.txt | less
```

If you only want to compare a specific file:

```bash
diff -u /tmp/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

---

## VIII. The Difference Between Reload and Restart

---

## Scenario 13: What is a Reload?

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
Reloads the configuration.

New worker processes are started using the new configuration.

Old worker processes exit after completing existing connections.

Existing connections are usually not interrupted intentionally.
```

Suitable for:

```text
Modifying server configurations.

Changing upstream settings.

Adjusting location blocks.

Updating certificates.

Changing log formats.

Modifying security response headers.

Adjusting rate-limiting settings.
```

---

## Scenario 14: What is a Restart?

Execute:

```bash
systemctl restart nginx
```

Meaning:

```text
Stops Nginx and then starts it again.
```

Risks:

```text
Connections may be interrupted.

Temporary unavailability is possible.

If the new configuration contains errors, restart may fail.

The impact is greater than that of a reload.
```

In production environments, try to use:

```bash
systemctl reload nginx
```

Only consider restarting in special circumstances.

---

## Scenario 15: When a Restart May Be Necessary

Scenarios where a restart might be required include:

```text
After modifying systemd's LimitNOFILE setting.

When upgrading Nginx binaries.

When changing certain global process-level parameters.

When adjusting dynamic module loading settings.

When worker processes fail to exit gracefully.

In very rare cases where a reload does not achieve the desired results.
```

However, before performing a production restart, confirm the following:

```text
The timing of the change.

Highly available nodes should be used if possible.

Have a rollback plan in place.

Perform health checks.

Assess the potential impact range.
```

---

## IX. Basic Understanding of Smooth Reloads

---

## Scenario 16: The General Process of Nginx Reloads

The general process of an Nginx reload is as follows:

```text
The master process receives a reload signal.

→ The master process checks the new configuration.

→ The master process starts new worker processes.

→ New worker processes begin accepting new requests using the updated configuration.

→ Old worker processes stop accepting new requests.

→ Old worker processes exit after completing existing connections.
```

To view the status of the master and worker processes:

```bash
ps -ef | grep nginx | grep -v grep
```

---

## Scenario 17: Observing Process Changes After a Reload

Before performing a reload:

```bash
ps -ef | grep "nginx: worker" | grep -v grep
```

After performing the reload:

```bash
systemctl reload nginx
```

Then check the process status again:

```bash
ps -ef | grep "nginx: worker" | grep -v grep
```

Also, examine the logs:

```bash
journalctl -u nginx -n 50
```

Or:

```bash
tail -n 50 /var/log/nginx/error.log
```

---

## X. The Single-Node Deployment Process

---

## Scenario 18: Standard Steps for Single-Node Deployment

```text
Check the current status.

→ Back up the configuration.

→ Make the necessary changes to the configuration.

→ Verify the configuration using `nginx -t`.

→ Use `nginx -T` to confirm the settings are correct.

→ Reload the configuration with `systemctl reload nginx`.

→ Verify the configuration locally using `curl`.

→ Verify it on a specified host using `curl`.

→ Check the access.log and error.log files.

→ Record all changes made.
```

---

## Scenario 19: Examples of Single-Node Deployment Commands

Check the status:

```bash
systemctl status nginx
```

Back up the configuration:

```bash
cp -a /etc/nginx/conf.d/example.com.conf /tmp/example.com.conf.$(date +%F-%H%M%S).bak
```

Verify the configuration:

```bash
nginx -t
```

Reload the configuration:

```bash
systemctl reload nginx
```

Verify locally using HTTP:

```bash
curl -I -H "Host: example.com" http://127.0.0.1/
```

Verify HTTPS:

```bash
curl -k -I --resolve example.com:44```bash
curl -I --resolve example.com:443:127.0.0.1 https://example.com/
```Know when it was modified.

Code review is possible.

Rapid rollback can be done.

Configuration auditing is feasible.

Integration with CI for checking is available.

Unified deployment across multiple nodes is achievable.

This avoids manual modifications that might cause chaos.

**Not recommended:**

- Making long-term changes only on production machines using vi.
- Lacking commit records.
- Failing to conduct reviews.
- Not having rollback versions.
- Manually synchronizing configurations across multiple machines.## Scenario 53: Reload Failed, but the Old Service is Still Running

When Nginx fails to reload, the old worker process may still be running.

**Phenomenon:**

- The service is still functioning normally.
- However, the new configuration has not taken effect.
- There are logs of the reload failure in `journalctl` or `error.log`.

**Troubleshooting:**

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

**Handling:**

- Do not assume that a reload attempt is always successful. Check the return codes and logs carefully.
- Ensure that the configuration has actually been applied to the server.```bash
ssh root@10.0.0.21 "nginx -t && systemctl reload nginx"
```

```bash
for host in 10.0.0.22 10.0.0.23; do
    echo "===== Deploying on $host ====="
    rsync -avz /etc/nginx/conf.d/example.com.conf root@$host:/etc/nginx/conf.d/example.com.conf
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Certificate Verification

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

## Upstream Check

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

## Log Monitoring

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
    echo "===== Rolling back on $host ====="
    scp /tmp/example.com.conf.2026-04-25-100000.bak root@$host:/etc/nginx/conf.d/example.com.conf
    ssh root@$host "nginx -t && systemctl reload nginx"
done
```

---

## Summary

The core steps in deploying Nginx configurations are not just reloading them, but also:

```text
Backup
Verification
Gradual deployment
Validation
Batch processing
Monitoring
Rollback
Documentation
```

A standard production process includes:

```text
Back up the configuration
Modify the configuration
Run nginx -t and nginx -T
Deploy it on one server in a gradual manner
Verify using curl --resolve on a specific node
Monitor logs and performance metrics
Deploy it to the remaining servers in batches
Perform full verification
Ensure a rollback path is available
```

The difference between `reload` and `restart` is that:

```text
Reload
→ Loads new configurations smoothly without interrupting existing connections
Restart
→ Stops and then restarts the service, which carries higher risks
```

When deploying configurations across multiple servers, follow these principles:

```text
Do not deploy all servers simultaneously.
Deploy on one server first before proceeding with batch deployment.
Run nginx -t and reload on each server after deployment.
Verify each server using a specific IP address.
Ensure that all servers are rolled back to the previous configuration in case of issues.
```

It is recommended to:

```text
Include Nginx configuration files in Git.
Do not store certificate private keys in Git.
Automate backup processes for deployment scripts.
Run nginx -t during the CI (Continuous Integration) phase.
Perform gradual verification during