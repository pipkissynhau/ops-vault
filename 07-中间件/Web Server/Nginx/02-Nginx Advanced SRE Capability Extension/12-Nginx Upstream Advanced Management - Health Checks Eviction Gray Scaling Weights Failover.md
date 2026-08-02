# 12-Nginx upstream Advanced Governance: Health Checks, Removal, Gray Release, Weighting, and Fault Switching

#Nginx #upstream #LoadBalance #HealthScreening #FaultCut. #GreyscaleRelease #WeightAdjustment #ServiceRemoval #AccessLayer #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capabilities Expansion/12-Nginx upstream Advanced Governance: Health Checks, Removal, Gray Release, Weighting, and Fault Switching.md

---

## One, Document Description

This document organizes advanced governance methods for Nginx upstream in production access layers.

The previous 03th article has already organized upstream basic capabilities:

```text
upstream Basic Configuration

Default rotation

weight Weights

max_fails

fail_timeout

backup

down

keepalive

ip_hash

least_conn
```

This article further organizes from the perspective of advanced SRE:

- Upstream backend pool governance philosophy
- Boundary of Nginx open-source health check capabilities
- Passive health check
- Deep understanding of max_fails / fail_timeout
- proxy_next_upstream fault retry
- Risk of non-idempotent request retry
- down temporary removal of nodes
- backup usage scenarios of backup nodes
- Weighted gray release
- Gray release by header
- Gray release by cookie
- Gray release by IP
- Using map directive to implement traffic diversion
- Fault switching process
- Upstream node anomaly troubleshooting
- Pre-and-post change verification
- Production notes

This article is the 12th article of the Nginx Advanced SRE Capabilities Expansion series.

This article's goal:

```text
I understand. upstream More than just load balance configuration

→ I know. Nginx Open Source Health Checking Border

→ Yes. max_fails / fail_timeout Basic failure awareness.

→ Yes. down Temporary removal of abnormal nodes

→ Yes. weight Simple greyscale release

→ Yes. map + header / cookie / IP It's basic traffic diversion.

→ Can design fault cut and rollback processes

→ To avoid an error in re-trying leading to double submission

→ It creates an access layer. upstream Governance thinking
```

---

## Two, What Advanced Upstream Governance Solves

In production, upstream is not just writing several backend IPs.

It needs to solve:

```text
Backend amplification

Backend Indentation

Fault removal.

Backup node.

New version of greyscale

Old version rolls back

Request to try again

Slow node quarantine.

Flow ratio adjustment

Backend connection reuse

Node maintenance

Fault cut.

Multiple instances of stable access
```

Ordinary upstream configuration focuses on:

```text
Request to forward to what backend
```

Advanced upstream governance focuses on:

```text
When will you forward it?

How much traffic did you forward?

How to remove them when they're abnormal.

How to try again in case of failure

How is the new version greyscale?

There's been a problem.

How back-end pools control changes
```

One-sentence understanding:

```text
upstream At the heart of high-level governance is the allocation of back-end traffic, observing, switching, rolling back.
```

---

## Three, Boundary of Nginx Open-Source Health Check Capabilities

---

## 1. Nginx Open-Source Default is Not Active Health Check

Nginx open-source default upstream health check is mainly passive perception.

That is:

```text
Only real requests are forwarded back to some backend

If connection fails, timeout or response abnormal

Nginx We're gonna have a problem with the back end.
```

This is different from active health check.

Active health check is:

```text
Nginx Periodically request backend /health

Determine whether nodes are available based on health tests

Even without a real business request, you can detect anomalies in advance.
```

---

## 2. Characteristics of Passive Health Check

Passive health check relies on real requests.

Advantages:

```text
Configure Simple

Open Source Default Support

No additional health check-ups.

I can feel a real transmission failure.
```

Disadvantages:

```text
No back-end anomalies detected in advance.

The first request could have stepped on the failure point.

It's not good enough.

No active detection of operational health interfaces

It's impossible to distinguish between fake work and port survival.

It's not very proactive to restore judgment.
```

---

## 3. Active Health Check Usually Relies on Other Capabilities

If you want to implement complete active health check, it usually relies on:

```text
Nginx Plus

OpenResty / Lua Custom Logic

Third-party health check module

External service discovery system

Configure Center + Autoextract

API Gateway

Kubernetes Service / Ingress Controller

Cloud load balanced health check
```

In production, it needs to be clear:

```text
max_fails / fail_timeout Not a complete active health check.
```

---

## Four, Deep Understanding of max_fails and fail_timeout

---

## Scenario 1: Basic Configuration

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;
}
```

Meaning:

```text
fail_timeout=30s
→ Failed Statistics Time Window

max_fails=3
→ Failed to reach in this time window 3 Minor

Once the conditions are met
→ Nginx For the time being, the node is considered unavailable.
```

Simple understanding:

```text
30 Failed in seconds 3 Minor

The node will be temporarily removed for some time.

After Nginx Will try to use this node again
```

---

## Scenario 2: max_fails=1 More Sensitive

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=1 fail_timeout=10s;
    server 10.0.0.22:8080 max_fails=1 fail_timeout=10s;
}
```

Features:

```text
I'm not sure if it's working.

Remove faster

The risk of miscalculation is higher.
```

Suitable for:

```text
A lot of backends.

Business can't tolerate failure.

Network environment stability

I hope to avoid the anomaly quickly.
```

Risks:

```text
An occasional network shaking could lead to temporary removal of nodes.
```

---

## Scenario 3: Larger max_fails More Lenient

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=5 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=5 fail_timeout=30s;
}
```

Features:

```text
To tolerate more failures

It's not easy to get rid of by mistake.

The failure point may continue to receive some requests.
```

Suitable for:

```text
Network shivering.

Backend occasional timeout

Don't want to take out nodes more often.

Business can tolerate a few failures.
```

---

## Scenario 4: Dual Meaning of fail_timeout

`fail_timeout` can be understood as two layers of meaning:

```text
Failed Statistics Window

Time of observation when node is considered unavailable
```

For example:

```nginx
max_fails=3 fail_timeout=30s
```

Can be understood as:

```text
30 Failed to reach in seconds 3 Minor

Node will be temporarily considered unavailable

Try again after a while.
```

---

## Five, proxy_next_upstream Fault Retry

---

## Scenario 5: What is proxy_next_upstream

`proxy_next_upstream` is used to control:

```text
When a request is forwarded upstream When nodes fail

Nginx Try next upstream Nodes
```

Example:

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_next_upstream error timeout http_502 http_503 http_504;
}
```

Meaning:

```text
Connection error

Connection timed out

502

503

504

In these cases you can try the next backend.
```

---

## Scenario 6: Common Configurable Conditions

Common conditions:

```text
error
→ Error while connecting to backend, sending requests, reading responses

timeout
→ Interactive Timeout with Backend

invalid_header
→ Backend returns illegal responder

http_500
→ Backend Return 500

http_502
→ Backend Return 502

http_503
→ Backend Return 503

http_504
→ Backend Return 504

http_403
→ Backend Return 403

http_404
→ Backend Return 404

off
→ Don't try next. upstream
```

Common production configuration:

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
```

---

## Scenario 7: Limit Retry Count

```nginx
location / {
    proxy_pass http://app_backend;

    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
    proxy_next_upstream_timeout 5s;
}
```

Explanation:

```text
proxy_next_upstream_tries 2
→ At most try 2 Minor

proxy_next_upstream_timeout 5s
→ The total duration of the re-test process does not exceed 5 sec
```

---

## Scenario 8: Why Limit Retry

If not controlled, retry may lead to:

```text
One user request to multiple backends

Back-end pressure amplified.

Flow avalanche in case of malfunction

Slow requests are more serious.

It's not a request to repeat.
```

Therefore, production suggests:

```text
Try again with caution.

Limit the number of retries

Time limit for retrying.

Write the request with special caution.
```

---

## Six, Risk of Retry for Non-Idempotent Requests

---

## Scenario 9: What is Idempotent Request

Idempotency can be simply understood as:

```text
One request and many requests

The end result is unanimous.
```

Requests that are usually more idempotent:

```text
GET Question

HEAD

Part PUT

Part DELETE
```

But whether it's idempotent depends on business implementation.

---

## Scenario 10: What is Non-Idempotent Request

Non-idempotent request refers to:

```text
Duplication of implementation may result in duplication of operations
```

Common non-idempotent operations:

```text
Create order

Payment of deductions

Submission of forms

Add Data

Send Text

Send Mail

Create Resource

Issuance of coupons

Message Delivery

Inventory deductions
```

If these requests are automatically retried by Nginx, it may lead to:

```text
Repeat Next Order

Repeated deductions

Repeat Send

Duplicate Writing

Business disorder
```

---

## Scenario 11: Caution Retry for Write Interfaces

For write interfaces, you can consider:

```nginx
location /api/order/create {
    proxy_pass http://app_backend;

    proxy_next_upstream error timeout;
    proxy_next_upstream_tries 1;
}
```

Or directly:

```nginx
location /api/order/create {
    proxy_pass http://app_backend;

    proxy_next_upstream off;
}
```

Production suggestion:

```text
Whether or not the writing interface allows a re-test should be determined by such capabilities as business skills

Don't try everything at the entrance. POST Request

Core writing interfaces should rely on business buttons, requests IDRepetition prevention mechanisms such as purchase orders
```

---

## Seven, Temporary Removal of Nodes with down

---

## Scenario 12: Manual Removal of Abnormal Nodes

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080 down;
}
```

Meaning:

```text
10.0.0.22 No more traffic.
```

Suitable for:

```text
Node Abnormal

Node maintenance

Node release failed

Node needs a bottom line.

Grey Roll Back

Cutting off a machine.
```

---

## Scenario 13: Why Recommend Using down Instead of Direct Deletion

Using down:

```nginx
server 10.0.0.22:8080 down;
```

Advantages:

```text
Keep History Configuration

Quick recovery.

You know the node was meant to be. upstream

Change differences are clear

Roll back easy.
```

Direct deletion is suitable for:

```text
Node permanent exit

Structural adjustments completed

Configure Cleanup
```

---

## Scenario 14: Standard Process for Removing Nodes

```text
Confirm the anomaly.

→ Alone. curl Health interface for this node

→ View this node to apply logs and resources

→ Modify upstream Add down

→ nginx -t

→ reload Nginx

→ Observation access.log Medium upstream_addr

→ Confirm that traffic no longer enters this node

→ Keep checking the node roots.
```

Check backend separately:

```bash
curl -v http://10.0.0.22:8080/health
```

Modify configuration and check:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

Observe logs:

```bash
tail -f /var/log/nginx/example.access.json.log
```

If it's JSON logs, you can statistics upstream:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

---

## Eight, Backup Node Governance

---

## Scenario 15: Backup Node Configuration

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080 backup;
}
```

Meaning:

```text
10.0.0.23 It's a backup node.

Normal main node without streaming

Only when the main node is not available
```

---

## Scenario 16: When is Backup Suitable

Suitable for:

```text
Downgrade services

Read-only back-up service

Low-specified spare nodes

Temporary disaster preparedness nodes

Maintain window bottom node
```

Example:

```text
When Main Service is not available

Backup node returns read-only page Noodles.

or return the downgrade hint

Or access low-capacity backup backend
```

---

## Scenario 17: Risks of Using Backup Nodes

Backup nodes cannot just be "backup".

Need to confirm:

```text
backup Whether nodes are really available

Compatibility of Versions

Data consistency

Whether to read only

Is there any way to absorb the sudden flow?

Any surveillance?

Whether or not to cause disruption of business data
```

Production suggestion:

```text
backup Node has to practice regularly.

Don't be left unattended for long.

Don't let backup It's a fake pocket.
```

---

## Nine, Weighted Gray Release

---

## Scenario 18: Weighted Gray Release Basics

Assume:

```text
Old version:10.0.0.21

New version:10.0.0.22
```

First give new version 10% traffic:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=9;
    server 10.0.0.22:8080 weight=1;
}
```

Meaning:

```text
The old version. 90%

New version of the convention 10%
```

---

## Scenario 19: Gradual Traffic Increase

First stage:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=9;
    server 10.0.0.22:8080 weight=1;
}
```

Second stage:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=7;
    server 10.0.0.22:8080 weight=3;
}
```

Third stage:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=5;
    server 10.0.0.22:8080 weight=5;
}
```

Fourth stage:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=1;
    server 10.0.0.22:8080 weight=9;
}
```

Final full traffic:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 down;
    server 10.0.0.22:8080 weight=1;
}
```

---

## Scenario 20: When is Weighted Gray Release Suitable

Suitable for:

```text
No status service

New and old versions compatible

Database structure compatibility

Interface protocol compatibility

No user viscosity required

Allow multiple requests from the same user to be made to different versions
```

Not suitable:

```text
The old and new data structure is not compatible

Session exists in local memory

The same user must have a fixed version

Other Organiser

Precision greyscale rule required
```

---

## Ten. Header-Based Canary

---

## Scenario 21: Header-Based Canary Strategy

Decide which upstream to route traffic to based on the request Header.

Example:

```text
X-Canary: true
```

Traffic with this Header enters the new version.

---

## Scenario 22: map + Header-Based Canary Configuration

Configure in the `http` block:

```nginx
map $http_x_canary $backend_pool {
    default app_stable;
    "true"  app_canary;
}
```

Define upstream:

```nginx
upstream app_stable {
    server 10.0.0.21:8080;
}

upstream app_canary {
    server 10.0.0.22:8080;
}
```

location:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://$backend_pool;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Test:

```bash
curl -H "Host: example.com" http://127.0.0.1/
```

```bash
curl -H "Host: example.com" -H "X-Canary: true" http://127.0.0.1/
```

---

## Scenario 23: Suitable Scenarios for Header-Based Canary

Suitable:

```text
Greyscale of internal test personnel

CI/CD Authentication

Specify client greyscale

Through gateway or front end injection Header

Control version by Special Request Header
```

Risks:

```text
Header It can be forged by clients.

Not as a secure border.

The public view needs to be carefully exposed.
```

---

## Eleven. Cookie-Based Canary

---

## Scenario 24: Cookie-Based Canary Strategy

Decide which new version the user enters based on the Cookie.

Example:

```text
Cookie: canary=true
```

Configuration:

```nginx
map $cookie_canary $backend_pool {
    default app_stable;
    "true"  app_canary;
}
```

upstream:

```nginx
upstream app_stable {
    server 10.0.0.21:8080;
}

upstream app_canary {
    server 10.0.0.22:8080;
}
```

location:

§

---

## Scenario 25: Suitable Scenarios for Cookie-Based Canary

Suitable:

```text
Specify Greyscale

Ratter Greyscale

Small-scale user experience validation

Front-end greyscale tag
```

Note:

```text
Cookie Could be modified by the user

Not suitable for sensitive access control.

Greyscale logic and business system alignment
```

---

## Twelve. IP-Based Canary

---

## Scenario 26: geo-Based IP Canary

Can use `geo` to set variables based on IP.

Example:

```nginx
geo $canary_ip {
    default 0;
    1.1.1.1 1;
    2.2.2.2 1;
}
```

Then use map:

```nginx
map $canary_ip $backend_pool {
    0 app_stable;
    1 app_canary;
}
```

upstream:

```nginx
upstream app_stable {
    server 10.0.0.21:8080;
}

upstream app_canary {
    server 10.0.0.22:8080;
}
```

location:

```nginx
location / {
    proxy_pass http://$backend_pool;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

---

## Scenario 27: IP-Based Canary Note on Real IP

If Nginx has:

```text
CDN

WAF

SLB
```

Must first configure real_ip.

Otherwise `$remote_addr` may be the proxy IP, not the real user IP.

Example:

```nginx
set_real_ip_from 10.0.0.10;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

Otherwise IP-based canary will fail or misidentify.

---

## Thirteen. Using split_clients for Proportional Canary

---

## Scenario 28: What is split_clients

`split_clients` can stably split traffic based on variables.

Example:

```nginx
split_clients "${remote_addr}${http_user_agent}" $variant {
    10%     canary;
    *       stable;
}
```

Meaning:

```text
Approximately 10% Requesting access. canary

The rest enter. stable
```

---

## Scenario 29: split_clients + map to Select upstream

```nginx
split_clients "${remote_addr}${http_user_agent}" $variant {
    10%     canary;
    *       stable;
}

map $variant $backend_pool {
    stable app_stable;
    canary app_canary;
}

upstream app_stable {
    server 10.0.0.21:8080;
}

upstream app_canary {
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://$backend_pool;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## Scenario 30: Notes on split_clients

Suitable:

```text
Scale Greyscale

No status service

No complex user image required

Simple version diversion
```

Note:

```text
The diversion ratio is approximate.

Sampling hours may not be accurate.

Selected hash key It affects stability.

If remote_addr It's not real. IP, the result will be different.

Change of equipment or User-Agent Change may come to different versions
```

---

## Fourteen. Notes on proxy_pass Variables

---

## Scenario 31: Using proxy_pass with Variables

Example:

```nginx
proxy_pass http://$backend_pool;
```

This syntax can dynamically select upstream.

But note:

```text
Variables proxy_pass It's fixed. proxy_pass More complicated.

Some scenes DNS Dialysis,URI Combination requires special attention.

Test before configuration goes online
```

---

## Scenario 32: Recommendations for Variable upstream Paths

Recommend keeping simple:

```nginx
location / {
    proxy_pass http://$backend_pool;
}
```

Avoid mixing with complex URI rewriting.

If path rewriting is needed, recommend testing separately:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/api/health
```

Observe the path received by the backend.

---

## Fifteen. Fault Switching Process

---

## Scenario 33: An upstream node is abnormal

Phenomenon:

```text
Partial requests 502

Some upstream_addr High error rate

There's a lot of errors in a backend log

Some node CPU / Memory anomaly
```

Troubleshoot upstream errors:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

Statistical errors upstream:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

Remove node:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
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

Verify traffic:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

---

## Scenario 34: Rolling Back Gray Release Abnormalities

Gray release configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=9;
    server 10.0.0.22:8080 weight=1;
}
```

If new version is abnormal, quickly rollback:

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

Observe:

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .uri, .status, .upstream_addr] | @tsv' | tail
```

---

## Scenario 35: Switch to backup if entire backend pool is abnormal

Configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.22:8080 max_fails=2 fail_timeout=10s;
    server 10.0.0.23:8080 backup;
}
```

If all main nodes are abnormal, backup nodes take over traffic.

But in production, cannot rely solely on automatic switching, need to observe:

```text
backup Is that true?

backup Can you carry it?

Whether or not operations enter a downgrade mode

Is the error rate down?

Acceptability of user experience
```

---

## Sixteen. Upstream Change Release Process

---

## Scenario 36: Pre-check Before Upstream Change

Confirm before change:

```text
What's the purpose of the change?

Additional node deployed

New node health check passed

Compatibility of Versions

Compatibility of database structure

What's the rollback?

Need for phase-down

Need to inform the operator

Monitoring and log observation
```

---

## Scenario 37: Backup Configuration

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

Or backup entire directory:

```bash
tar czf /tmp/nginx-conf-backup-$(date +%F-%H%M%S).tar.gz /etc/nginx
```

---

## Scenario 38: Check New Backend

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
curl -v http://10.0.0.22:8080/health
```

If need to check version:

```bash
curl -s http://10.0.0.22:8080/version
```

---

## Scenario 39: Check After Configuration Modification

```bash
nginx -t
```

View full configuration:

```bash
nginx -T | grep -n "upstream app_backend" -A 30
```

---

## Scenario 40: Verify After reload

```bash
systemctl reload nginx
```

Check Nginx status:

```bash
systemctl status nginx
```

Local access:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/health
```

Check error logs:

```bash
tail -n 100 /var/log/nginx/error.log
```

Check upstream distribution:

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

---

## Seventeen. Upstream Troubleshooting

---

## Scenario 41: A node never receives traffic

Common causes:

```text
Node by down

weight Too low.

Node connection failed to be temporarily removed

max_fails Trigger

backup Node is normally not connected

He's got another hit. upstream

Configure Not reload

Logging upstream_addr Incomplete collection
```

Troubleshoot:

```bash
nginx -T | grep -n "upstream app_backend" -A 30
```

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

---

## Scenario 42: High error rate on a node

Statistical 5xx upstream:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

Check node health:

```bash
curl -v http://10.0.0.22:8080/health
```

Check error logs:

```bash
grep "10.0.0.22" /var/log/nginx/error.log | tail -n 100
```

Handling direction:

```text
Temporary down Remove Nodes

Check this node application log

Check the node host resource

Query Version Difference

Check dependent configuration differences

Recovering traffic after recovery.
```

---

## Scenario 43: High upstream_response_time

JSON log analysis:

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .uri, .request_time, .upstream_response_time, .upstream_addr] | @tsv' | head
```

Common causes:

```text
Backend slow

Database slow query

Back-end pool full

Backend connect pool insufficient

Inadequate resources for a certain node

The network links are abnormal.

Slow external dependence
```

Handling direction:

```text
Position slow nodes

Position slow interface

Check Backend Log

Slow check of databases

Check the mainframe. CPU / Memory / IO

Temporary removal of slow nodes if necessary
```

---

## Eighteen. Production Notes

---

## 1. Nginx Open Source max_fails is not active health check

Do not mistakenly think:

```text
Configure max_fails / fail_timeout

It's equal to... Nginx It's active. /health
```

It is mainly passive failure detection.

---

## 2. Confirm version compatibility before gray release

Must confirm:

```text
Interface Compatibility

Database Compatibility

Cache Compatibility

Message Format Compatibility

Front-end resource compatibility

Rollback Compatibility
```

Otherwise even if Nginx routes correctly, business issues may occur.

---

## 3. Weight-based canary is not user-level canary

weight only controls approximate traffic ratio.

Cannot guarantee:

```text
Regular access of the same user to the same version

Specify user access to new versions

Greyscale by Account

By Tenant Greyscale
```

Complex canary should be placed in:

```text
Business gateway

Application Layer

API Gateway

Service Grid

Greyscale Dissemination Platform
```

---

## 4. Header/ Cookie-based canary is not security boundary

Headers and Cookies can be forged by clients.

They are suitable for:

```text
Test Grayscale

Internal authentication

Unsafe sensitive diversion
```

Not suitable for:

```text
Permission Control

Safe isolation

Sensitive function control
```

---

## 5. Caution with Non-Idempotent Request Retry Configurations

Be extremely cautious with the following requests:

```text
Order.

Payments

Tickets

Send Text

Send Mail

Create Resource

Delete Resource

Submission of forms
```

Retry must be confirmed at the entry layer only after ensuring the business has an idempotency mechanism.

---

## 6. Monitor Alongside Failover Switching

After switching, must observe:

```text
5xx Decline

499 Change

504 Decline

request_time Recovery

upstream_response_time Recovery

Whether the error log is reduced

Recovery of operational indicators
```

---

## 7. Record Reasons for Down Nodes

Recommended to record:

```text
Remove Time

Remove Nodes

Reasons for removal

Operator

Recovery Time

Authentication Results

Causes of failure

Whether to repeat
```

---

## 8. upstream Configurations Should Be Version Controlled

Production recommendation:

```text
Nginx Configure Entry Git

Before Change review

Changed records

Support rapid rollback

To avoid multiple manual changes.
```

---

## Nineteen, Common Commands Summary

---

## Configuration Check

```bash
nginx -t
```

```bash
nginx -T
```

```bash
nginx -T | grep -n "upstream app_backend" -A 30
```

```bash
nginx -T | grep -n "proxy_next_upstream" -A 10 -B 5
```

```bash
nginx -T | grep -n "map" -A 20
```

```bash
nginx -T | grep -n "split_clients" -A 20
```

```bash
nginx -T | grep -n "geo" -A 20
```

---

## Service Management

```bash
systemctl reload nginx
```

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100
```

---

## Backend Node Check

```bash
nc -zv -w 2 10.0.0.21 8080
```

```bash
nc -zv -w 2 10.0.0.22 8080
```

```bash
curl -v http://10.0.0.21:8080/health
```

```bash
curl -v http://10.0.0.22:8080/health
```

```bash
curl -s http://10.0.0.22:8080/version
```

---

## Nginx Entry Validation

```bash
curl -v -H "Host: example.com" http://127.0.0.1/health
```

```bash
curl -H "Host: example.com" -H "X-Canary: true" http://127.0.0.1/
```

```bash
curl -H "Host: example.com" -H "Cookie: canary=true" http://127.0.0.1/
```

---

## error.log Troubleshooting

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "connect() failed" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
grep "10.0.0.22" /var/log/nginx/error.log | tail -n 100
```

---

## JSON access.log Analysis

```bash
cat /var/log/nginx/example.access.json.log | jq -r '.upstream_addr' | sort | uniq -c | sort -nr
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | .upstream_addr' | sort | uniq -c | sort -nr | head
```

```bash
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

The core of advanced Nginx upstream governance is:

```text
Health awareness

Fault removal.

Request to try again

Greyscale Release

Flow diversion

Fault cut.

Quick Roll Back
```

Basic fault awareness:

```nginx
server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
```

Temporary node removal:

```nginx
server 10.0.0.22:8080 down;
```

Backup node:

```nginx
server 10.0.0.23:8080 backup;
```

Weighted gray-scale:

```nginx
server 10.0.0.21:8080 weight=9;
server 10.0.0.22:8080 weight=1;
```

Fault retry:

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 2;
proxy_next_upstream_timeout 5s;
```

Header gray-scale:

```nginx
map $http_x_canary $backend_pool {
    default app_stable;
    "true"  app_canary;
}
```

Production recommendation:

```text
Nginx Open Source max_fails Not an active health check.

down It's suitable for temporary removal, permanent offline and deletion.

backup Node has to practice regularly.

weight Greyscale is not equal to user-level greyscale

Header / Cookie Greyscale is not a safe border.

I'm not asking to try again with caution.

The fault cut must be observed. 5xxI don't know.504I don't know.request_timeI don't know.upstream_response_time

upstream Change must be backed up.nginx -tI don't know.reload, validate, roll back

Complex greyscale and global flow management should be handed over to API Gateway, service grid or distribution platform
```