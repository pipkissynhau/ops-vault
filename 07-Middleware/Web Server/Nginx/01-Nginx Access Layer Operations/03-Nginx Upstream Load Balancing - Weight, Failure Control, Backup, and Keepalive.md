# 03-Nginx Upstream Load Balancing: Weight, Failure Control, Backup, and Keepalive

#Nginx #upstream #load balancing #reverse proxy #access layer #middleware #Web server #ops #SRE

---

## Recommended Path

07-Middleware/Web Server/Nginx/01-Nginx Access Layer Ops/03-Nginx Upstream Load Balancing: Weight, Failure Control, Backup, and Keepalive.md

---

## I. Document Overview

This document explains the configuration of Nginx `upstream` backend service pools and basic load balancing capabilities.

Key points include:

- What `upstream` is
- The relationship between `upstream` and `proxy_pass`
- Multiple backend load balancing
- Default round-robin scheduling
- Weight-based routing
- `max_fails` and `fail_timeout` for failure control
- Backup nodes
- Temporarily removing faulty nodes (`down`)
- Keepalive for reusing backend connections
- Basics of `ip_hash` and `least_conn`
- Naming conventions for `upstream`
- Common troubleshooting commands for `upstream`
- Handling exceptions in backend nodes
- Best practices for production environments

This document is part of the Nginx Access Layer Ops series, Chapter 03.

Objectives:

```text
- Understand how to configure `upstream`
- Set up multiple backend services
- Recognize Nginx's default load balancing behavior
- Use weight to distribute traffic proportionally
- Implement basic failure control with `max_fails`/`fail_timeout`
- Configure backup nodes for redundancy
- Temporarily remove faulty nodes from the load balance
- Comprehend the role of keepalive in maintaining connections
- Troubleshoot 502/504 errors and connection failures related to `upstream`
```

---

## II. What is Upstream

`upstream` is used to define a group of backend services.

Simply put:

```text
upstream
→ Backend service pool

proxy_pass
→ Forward requests to this backend service pool
```

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://appbackend;
    }
}
```

Request flow:

```text
Client

→ Nginx

→ app_backend

→ 10.0.0.21:8080 or 10.0.0.22:8080
```

In short:

```text`
`upstream` aggregates multiple backend instances under a single name.
```

---

## III. Why Use Upstream

When configuring only one backend:

```nginx
location / {
    proxy_pass http://10.0.0.21:8080;
}
```

Problems include:

```text
- Only one backend means no traffic distribution
- Downtime of a single node affects the entire service
- Difficulties in scaling or switching backends
- Lack of weight-based routing control
- No backup mechanism for failures
```

With `upstream`:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Advantages include:

```text
- Multiple backends can handle increased traffic
- Easy horizontal scaling of backends
- Weight-based routing for optimal distribution
- Faulty nodes can be temporarily removed from the load balance
- Backup nodes ensure service continuity
- Clearer configuration structure
```

---

## IV. Basic Upstream Structure

---

## Scenario 1: Basic Upstream Configuration

Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backend;
    }
}
```

Explanation:

```text
appbackend
→ Name of the upstream service pool

server 10.0.0.21:8080
→ First backend node

server 10.0.0.22:8080
→ Second backend node

proxy_pass http://app_backend
→ Forwards requests to the app_backend service pool
``### Version Comparison
- The old version accounts for approximately 90%.
- The new version comprises around 10%.

After verifying the stability of the new version, adjustments can be made gradually:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=5;
    server 10.0.0.22:8080 weight=5;
}
```

Ultimately, the configuration can be changed to:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 down;
    server 10.0.0.22:8080 weight=1;
}
```

### Note
- This example only demonstrates a simple weight-based gradual transition.
- More complex methods such as using headers, cookies, or user-level grayscaling will be covered in the advanced governance section.

---

## VII. Temporarily Taking Down Nodes Using `down`
---

## Scenario 7: Temporarily Removing a Backend Node
Configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080 down;
}
```

Meaning:

- The node `10.0.0.22:8080` will no longer receive traffic.

Suitable Scenarios:

- Node maintenance
- Node failure
- Issues with the node version
- Temporarily removing a faulty instance
- Rolling back to a previous configuration

After making the changes, check the configuration and reload Nginx:

```bash
nginx -t
systemctl reload nginx
```

---

## Scenario 8: Differences Between Using `down` and Directly Deleting a Node
When using `down`:

```nginx
server 10.0.0.22:8080 down;
```

Advantages:

- Configuration records are retained.
- It is easy to restore the node later.
- It is clear that the node once existed.
- Rolling back is simple.

If the node is deleted directly:

```nginx
# server 10.0.0.22:8080;
```
or by deleting the entire line, it is suitable for scenarios where the node needs to be permanently removed, such as after service migration or configuration cleanup.

**Production Recommendation:** Use `down` for temporary removal and delete the node permanently only when necessary.

---

## VIII. Using a Backup Node
---

## Scenario 9: Configuring a Backup Node
Example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
    server 10.0.0.23:8080 backup;
}
```

Meaning:

- The node `10.0.0.23` serves as a backup.
- Under normal circumstances, it does not receive requests.
- It only takes over when all primary nodes are unavailable.

Suitable Scenarios:

- Providing temporary backup services
- Downgrading services
- Using as a read-only backup node
- Serving as a low-specification fallback option
- For disaster recovery scenarios

---

## Scenario 10: Precautions When Using a Backup Node
It is essential to verify the following aspects of the backup node:

- Whether the service is truly available.
- The consistency of data.
- Its ability to handle traffic.
- Whether it is read-only only.
- Potential impact on business continuity.
- Whether any special notifications or downgrade responses are required.

**Avoid:** Configuring an unverified backup node, neglecting its maintenance, or allowing it to process production traffic with outdated data.

---

## IX. `max_fails` and `fail_timeout`
---

## Scenario 11: Basic Failure Control
Configuration:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 max_fails=3 fail_timeout=30s;
}
```

Meaning:

- `max_fails=3` means that if failures occur three times within the `fail_timeout` period of 30 seconds, the node will be considered unavailable.
- `fail_timeout=30s` specifies that the failure monitoring is performed every 30 seconds, and this time frame also determines how long a node will be marked as unavailable.

**Simple Explanation:** If a node fails three times within 30 seconds, Nginx will temporarily assume it is unavailable and attempt to connect to it again after some time.

---

## Scenario 12: What Constitutes a Failure## Scenario 19: Notes on ip_hash

Note:

```text
If there are SLB, CDN, or WAF in front of Nginx,

the remote_addr seen by Nginx may be the proxy IP address.

In this case, ip_hash may lose its meaning regarding the actual client.
```

For example:

```text
If all requests come from the same SLB IP address,

ip_hash might cause a large number of requests to be directed to the same backend server.
```

Therefore, in real production environments, it is more recommended to:

```text
Make the application stateless.

Store session data in Redis.

Use a unified authentication system.

Avoid relying on Nginx's ip_hash feature for core session persistence.
```

---

## Section 13: Basics of least_conn

---

## Scenario 20: What is least_conn?

`least_conn` prioritizes forwarding requests to the backend server with the fewest current connections.

Example:

```nginx
upstream app_backend {
    least_conn;

    server 10.0.0.21:8080;
    server 10.0.0.22:8080;
}
```

Suitable scenarios:

```text
When there is a significant difference in request processing times.

When there are many persistent connections.

When the number of backend connections is uneven.

When some requests take longer to process.
```

Note:

```text
least_conn is not a panacea.

If there are large differences in backend performance, it is still necessary to use the weight parameter.

For very short requests, default round-robin scheduling is usually sufficient.
```

---

## Section 14: Complete Example of upstream Configuration

---

## Scenario 21: Basic Production-level upstream Example

Configuration file:

```bash
vi /etc/nginx/conf.d/example.com.conf
```

Content:

```nginx
upstream app_backend {
    server 10.0.0.21:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 10.0.0.22:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 10.0.0.23:8080 backup;

    keepalive 64;
}

server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log warn;

    location / {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;
        proxy_set_header Connection "";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 5s;
    }
}
```

Check the configuration:

```bash
nginx -t
```

Reload the configuration:

```bash
systemctl reload nginx
```

Verify the configuration by making a request:

```bash
curl -v -H "Host: example.com" http://127.0.0.1/
```

View the logs:

```bash
tail -f /var/log/nginx/example.access.log
```

```bash
tail -f /var/log/nginx/example.error.log
```

---

## Scenario 25: Checking Upstream Connections to the Backend

To check the connections to backend port 8080:

```bash
ss -antp | grep ':8080'
```

To count connections by status:

```bash
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c | sort -nr
```

To view TIME_WAIT connections:

```bash
ss -ant state time-wait | grep ':8080' | wc -l
```

---

## Section 16: Common Errors with upstream

---

## Scenario 26: Incorrect naming of the upstream block

Incorrect example:

```nginx
upstream app_backend {
    server 10.0.0.21:8080;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_backnd;
    }
}
```

Issue:

```text
The referenced "app_backnd" does not exist.
```

Check the configuration:

```bash
nginx -t
```

---

## Scenario 27: Incorrect placement of the upstream block

Incorrect example:

```nginx
server {
    listen 80;

    upstream app_backend {
        server 1## Scenario 31: Troubleshooting for 504 Errors

Common causes for 504 errors include:

- Backend response timeout
- Slow backend processing
- Time-consuming database queries
- Full backend thread pool
- Exhausted backend connection pool
- High CPU/memory/I/O load on the backend
- Excessively short Nginx timeout setting
- Abnormal network connections

Troubleshooting commands:

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
curl -v http://backend_ip:backend_port/health
```

```bash
top
```

```bash
free -h
```

```bash
iostat -x 1 5
```

```bash
ss -antp | grep backend_port
```

---

## Section 18: Process for Modifying Upstream Settings

After making changes to the upstream configuration in production, it is recommended to follow these steps:

- Verify the target of the changes.
- Back up the current configuration.
- Modify the upstream settings.
- Run `nginx -t` to check the configuration.
- Test the backend services individually.
- Reload Nginx.
- Check the response using `curl`.
- Review the access.log and error.log files.
- Monitor status codes and business metrics.
- Roll back the changes if necessary.

---

## Scenario 32: Backing Up Configuration Files

To back up configuration files, use the following command:

```bash
cp -a /etc/nginx/conf.d/example.com.conf /etc/nginx/conf.d/example.com.conf.$(date +%F-%H%M%S).bak
```

---

## Scenario 33: Checking Changes After Modifying Configuration

After modifying the configuration, run `nginx -t` to verify its correctness. To view the complete configuration details, use:

```bash
nginx -T | grep -n "upstream app_backend" -A 20
```

---

## Scenario 34: Reloading Configuration

To reload the modified configuration, execute:

```bash
systemctl reload nginx
```

Check the status of Nginx after reloading:

```bash
systemctl status nginx
```

---

## Scenario 35: Rolling Back Configuration

If needed to revert the changes, back up the original configuration file and then restore it using:

```bash
cp -a /etc/nginx/conf.d/example.com.conf.2026-04-25-100000.bak /etc/nginx/conf.d/example.com.conf
```

After restoring the configuration, check it again using `nginx -t` and then reload it:

```bash
systemctl reload nginx
```

---

## Section 19: Production Considerations

- Always verify each backend service individually before modifying the upstream settings in Nginx.
- Do not directly remove a backup node; instead, use the `down` command to temporarily disable it for easy rollback.
- Gradually adjust the weight settings when implementing load balancing.
- The value of `keepalive` should be determined based on factors such as the number of backend connections and request concurrency.
- Be cautious when configuring retries for non-idempotent requests to prevent duplicate transactions.

---

## Section 20: Summary of Commonly Used Commands

- **Configuration verification:** `nginx -t`, `nginx -T`, `nginx -T | grep -n "upstream"`, `nginx -T | grep -n "proxy_pass"`, `nginx -T | grep -n "upstream app_backend" -A 20`.
- **Service reloading:** `systemctl reload nginx`, `systemctl status nginx`.
- **Backend port checking:** `nc -zv -w 2 10.0.0.21 8080`, `nc -zv -w 2 10.0.0.22 8080`, `for ip in ...; do ...; done`.
- **Backend HTTP checking:** `curl -I http://...`, `curl -v http://.../health`.
- **Nginx entry point testing:** `curl -v -H "Host: example.com" http://127.0.0.1/`, `for i in ...; do ...; done`.
- **Log troubleshooting:** `tail -n 100 /var/log/nginx/error.log`, `grep -i "upstream" ...`, etc.
- **Connection monitoring:** `ss -antp | grep ':8080'`, `ss -ant | ...`, `wc -l`.
- **Configuration backup:** `cp -a ...`, `tar czf ...`.

---

## Section 21: Summary of Upstream Settings

The core function of upstream settings in Nginx is to combine multiple backend services into a single pool and then forward requestsBackup nodes must be verified regularly.  
The keepalive mechanism should be evaluated in conjunction with the number of workers and the backend connection capacity.  
For non-idempotent requests, use proxy_next_upstream for retrying with caution.  
Basic upstream functionality in the open-source version of Nginx does not constitute a comprehensive active health check system.  
Always back up your configuration before making changes to the upstream settings, and run `nginx -t` before reloading the server.