# 12-Nginx Upstream Advanced Governance: Health Checks, Removal, Gradual Release, Weighting, and Fault Flow Switching

#Nginx #upstream #load balancing #health checks #fault flow switching #gradual release #weight adjustment #service removal #access layer #SRE

---

## Recommended Reading Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capability Expansion/12-Nginx Upstream Advanced Governance: Health Checks, Removal, Gradual Release, Weighting, and Fault Flow Switching.md

---

## I. Document Overview

This document outlines advanced governance methods for Nginx upstream in production access layers.

In the previous article 03, basic upstream capabilities were covered:

```text
Basic upstream configuration

Default round-robin scheduling

Weighting

max_fails

fail_timeout

Backup

Downstate

Keepalive

ip_hash

least_conn
```

This article delves further from an advanced SRE perspective:

- Governance strategies for the upstream backend pool
- Boundaries of Nginx open-source health check capabilities
- Passive health checks
- In-depth understanding of max_fails/fail_timeout
- proxy_next_upstream fault retry mechanism
- Risks associated with non-idempotent request retries
- Temporary removal of downstate nodes using down
- Use cases for backup nodes
- Weight-based gradual release
- Gradual release by header
- Gradual release by cookie
- Gradual release by IP address
- Traffic distribution using the map directive
- Fault flow switching process
- Troubleshooting for upstream node failures
- Verification before and after changes
- Production considerations

This article is part of the Nginx Advanced SRE Capability Expansion series, chapter 12.

Objectives of this article:

```text
To understand that upstream is more than just load balancing configuration

→ To be aware of the limitations of Nginx open-source health checks

→ To use max_fails/fail_timeout for basic fault detection

→ To temporarily remove abnormal nodes using down

→ To perform simple weight-based gradual release

→ To distribute traffic using map + header/cookie/IP address

→ To design fault flow switching and rollback processes

→ To prevent duplicate business requests due to incorrect retries

→ To develop a comprehensive governance mindset for access layer upstream
```

---

## II. Problems Addressed by Advanced Upstream Governance

In production, upstream is not just about listing several backend IP addresses.

It needs to address the following challenges:

```text
Backend scale-out

Backend scale-in

Fault node removal

Backup nodes as fallbacks

Gradual release of new versions

Rollback of old versions

Request retries

Isolation of slow nodes

Adjustment of traffic ratios

Reuse of backend connections

Node maintenance

Emergency fault flow switching

Stable access for multiple instances
```

Basic upstream configurations focus on:

```text
Which backend to forward requests to
```

Advanced upstream governance addresses:

```text
When to forward requests

How much traffic to forward

How to remove abnormal nodes in case of failures

How to retry requests during failures

How to gradually release new versions

How to switch back to previous versions in case of issues

How to manage controllable changes to the backend pool
```

In summary:

```text
The core of advanced upstream governance is ensuring that backend traffic can be allocated, monitored, switched, and rolled back.
```

---

## III. Boundaries of Nginx Open-source Health Check Capabilities

---

## 1. Nginx open-source does not support active health checks by default

By default, Nginx open-source upstream health checks are primarily passive.

This means that:

```text
Only when a real request is forwarded to a backend and results in a connection failure, timeout, or abnormal response will Nginx consider the backend faulty.
```

This differs from active health checks, which:

```text
Nginx periodically initiates active requests to the backend /health and decides whether the node is available based on the results. Even without real business requests, anomalies can be detected in advance.
```

---

## 2. Characteristics of passive health checks

Passive health checks rely on actual requests.

Advantages:

```text
Simple configuration

Supported by the open-source version by default

No additional health check interfaces required

Can detect real forwarding failures
```

Disadvantages:

```text
Cannot detect backend anomalies in advance

The first request may encounter a faulty node

The health status is not highly detailed

Cannot actively probe business health APIs

Unable to distinguish between application fakes and active ports

Recovery decisions are less proactive
```

---

## 3. Active health checks usually require additional tools or services

For comprehensive active health checks, the following are typically required:

```text
Nginx Plus

OpenResty / Lua for custom logic

Third-party health check modules

External service discovery systems

Configuration centers with automatic removal mechanismsIf these requests are automatically retried by Nginx, it may lead to the following issues:

```text
Duplicate orders placed

Repeated deductions made

Repeated transmissions

Repeated writes

Disruption in business operations
```

---

## Scenario 11: Cautionous Retries for Write Interfaces

For write interfaces, consider the following configuration:

```nginx
location /api/order/create {
    proxy_pass http://app_backend;

    proxy_next_upstream error timeout;
    proxy_next_upstream_tries 1;
}
```

Or alternatively:

```nginx
location /api/order/create {
    proxy_pass http://app_backend;

    proxy_next_upstream off;
}
```

Production recommendations:

```text
Whether retries are allowed for write interfaces should be determined based on the business's idempotency capabilities.

Do not blindly retry all POST requests at the entry layer.

Core write interfaces should use mechanisms such as business idempotency keys, request IDs, and order numbers to prevent duplicates.
```

---

## Section 7: Temporarily Removing Downstream Nodes

---

## Scenario 12: Manually Removing Abnormal Nodes

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080 down;
}
```

Meaning:

```text
The node 10.0.0.22 will no longer receive traffic.
```

Suitable for:

```text
When a node is abnormal

During node maintenance

If a node fails to be deployed

When a node needs to be taken offline for troubleshooting

For temporary gray-scale rollback

To temporarily divert traffic to another machine
```

---

## Scenario 13: Why Use "down" Instead of Direct Deletion?

Using "down":

```nginx
server 10.0.0.22:8080 down;
```

Advantages:

```text
Retains historical configuration

Facilitates quick recovery

Indicates that the node originally belonged to this upstream group

Clarifies differences in configuration changes

Enables easy rollback
```

Direct deletion is suitable for:

```text
When a node needs to be permanently taken offline

After completing architectural adjustments

When cleaning up configurations
```

---

## Scenario 14: Standard Process for Removing Nodes

```text
Confirm that the node is abnormal.

→ Use curl to individually test whether the health check interface of this node is functioning properly.

→ Check the application logs and resources of this node.

→ Modify the upstream configuration to include the "down" setting for this node.

→ Run `nginx -t` to verify the changes.

→ Reload Nginx.

→ Check the `upstream_addr` in the access.log file to confirm that traffic is no longer directed to this node.

→ Continue to investigate the root cause of the issue with this node.
```

Individually check the backend by running:

```bash
curl -v http://10.0.0.22:8080/health
```

After modifying the configuration, check again:

```bash
nginx -t
```

Reload Nginx:

```bash
systemctl reload nginx
```

Monitor the logs:

```bash
tail -f /var/log/nginx/example.access.json.log
```

If the logs are in JSON format, you can count the number of unique `upstream_addr` values:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

---

## Section 8: Managing Backup Nodes

---

## Scenario 15: Configuring Backup Nodes

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080 backup;
}
```

Meaning:

```text
The node 10.0.0.23 serves as a backup node.

It does not receive traffic when the primary nodes are available.

It only starts receiving traffic when all primary nodes are unavailable.
```

---

## Scenario 16: When Backup Nodes Are Suitable

Suitable for:

```text
When downgrading services

As read-only backup services

For low-specification backup nodes

As temporary disaster recovery nodes

As backup nodes during maintenance windows
```

Example:

```text
When the primary service is unavailable,

the backup node can return a read-only page,

or display a downgrade notice,

or connect to a less capable backup backend.
```

---

## Scenario 17: Risks Associated with Using Backup Nodes

Backup nodes should not merely be labeled as "backup."

It is essential to verify that:

```text
The backup node is truly available for use.

### Scenario 25: Cookie Grading Suitable for Certain Scenarios

**Suitable for:**
- Specified user grading
- Tester grading
- Small-scale user experience validation
- Front-end-controlled grading markers

**Notes:**
- Cookies may be modified by users.
- Not suitable for sensitive permission control.
- The grading logic must be coordinated with the business system.curl -v http://10.0.0.22:8080/health

To check the version:

curl -s http://10.0.0.22:8080/version```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .uri, .status, .upstream_addr, .upstream_status] | @tsv' | tail
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .uri, .request_time, .upstream_response_time, .upstream_addr] | @tsv' | head
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
cp -a /etc/nginx/conf.d/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

```bash
nginx -t
```

```bash
systemctl reload nginx
```

---

## Twenty, One-Sentence Summary

The core of advanced Nginx upstream governance includes:

**Health monitoring**

**Fault isolation**

**Request retry**

**Gradual deployment**

**Traffic routing**

**Failover mechanism**

**Rapid rollback**

Basic health monitoring example:

```nginx
server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
```

Temporary node removal:

```nginx
server 10.0.0.22:8080 down;
```

Backup node configuration:

```nginx
server 10.0.0.23:8080 backup;
``

Weight-based gradual deployment:

```nginx
server 10.0.0.21:8080 weight=9;
server 10.0.0.22:8080 weight=1;
```

Fault retry settings:

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 2;
proxy_next_upstream_timeout 5s;
``

Header-based gradual deployment:

```nginx
map $http_x_canary $backend_pool {
    default app_stable;
    "true"  app_canary;
}
```

Production recommendations:

- Note that the `max_fails` setting in Nginx's open-source version does not perform active health checks.
- Use `down` to temporarily remove a node; permanently remove it only after testing.
- Regularly test backup nodes to ensure their functionality.
- Remember that weight-based gradual deployment is different from user-level gradual deployment.
- Be cautious when using Header or Cookie-based gradual deployment, as they may affect security.
- Avoid retrying non-idempotent requests.
- When implementing failover, closely monitor errors such as 5xx, 504, `request_time`, and `upstream_response_time`.
- Always perform backups before making any changes to Nginx configuration, and verify the results before rolling out changes.
- For complex gradual deployment scenarios and global traffic management, consider using API gateways, service meshes, or dedicated release platforms.