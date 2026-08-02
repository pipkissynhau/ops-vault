# 15-Nginx Observability: stub_status, nginx-prometheus-exporter, nginx-vts, and Grafana Alerts

#Nginx #Observability #Monitoring #Prometheus #Grafana #Exporter #stub_status #nginx-vts #nginx-prometheus-exporter #nginx-vts-exporter #SRE

---

## Recommended Reading Path

07-Middleware/Web Server/Nginx/02-Nginx Advanced SRE Capability Expansion/15-Nginx Observability: stub_status, nginx-prometheus-exporter, nginx-vts, and Grafana Alerts.md

---

## I. Document Overview

This document outlines methods for building observability at the Nginx access layer, focusing on how to integrate Nginx with Prometheus, Grafana, and logging platforms.

Key points include:

- Nginx observability objectives
- Two approaches to integrating Nginx with Prometheus for monitoring
  - Method 1: `stub_status` + `nginx-prometheus-exporter`
  - Method 2: `nginx-module-vts` + `nginx-vts-exporter`
- Comparisons of the two methods in terms of metrics availability and Grafana dashboard richness
- Configuration details for `stub_status` and `nginx-prometheus-exporter`
- Deployment guidelines for both containerized and systemd-based solutions
- Steps for compiling and installing `nginx-module-vts`
- Configuration of the VTS status page
- Setup of `nginx-vts-exporter`
- Prometheus configuration settings
- Grafana dashboard design considerations
- Monitoring practices such as HTTP liveness checks with Blackbox Exporter and HTTPS certificate expiration alerts
- Analysis of access.log and error.log logs for troubleshooting
- Common alert rules and best practices for production use

This article is part of the Nginx Advanced SRE Capability Expansion series, Chapter 15.

## II. Nginx Observability Objectives

As an access layer component, Nginx needs to provide insights into the following aspects:

- Whether Nginx is running
- Whether its ports are listening
- Its ability to process requests normally
- Abnormalities in the current number of connections
- Sudden spikes in request volumes
- Frequent occurrences of 4xx/5xx errors
- High rates of 499/502/504 responses
- Triggering rate limiting mechanisms (e.g., 429)
- Issues with downstream upstream servers
- Higher error rates for specific backend nodes
- Abnormal traffic patterns for particular `server_name` entries
- Increases in request processing times
- Prolonged `upstream_response_time`
- Approaching expiration of certificates
- Proper logging activities
- Disk space usage due to log accumulation
- Whether only certain Nginx instances are experiencing issues

In summary, Nginx observability is not just about checking if processes are running; it’s about ensuring that the access layer functions smoothly, traffic flows normally, errors are identified promptly, and downstream services are healthy.

---

## III. Two Approaches to Integrating Nginx with Prometheus for Monitoring

There are two common ways to integrate Nginx with Prometheus for monitoring:

---

## Method 1: stub_status + nginx-prometheus-exporter

### Flow:

```text
Nginx's built-in stub_status module

→ nginx/prometheus-exporter

→ Prometheus

→ Grafana

→ Alertmanager
```

### Features:

- **Simple deployment**: No need to recompile Nginx.
- **Uses native Nginx modules**: Suitable for basic monitoring needs.
- **Low maintenance cost**.

### Key Metrics Available:

- Nginx availability status
- Active connections
- Accepted connections
- Handled connections
- Total requests
- Reading, Writing, Waiting operations
- Basic QPS metrics

### Limitations:

- **Limited metrics**: Fewer detailed indicators are available.
- **Lack of granularity**: No server_name or upstream-specific data.
- **Grafana dashboard simplicity**: May not provide sufficient visual insights.

---

## Method 2: nginx-module-vts + nginx-vts-exporter

### Flow:

```text
Compile and install nginx-module-vts

→ Access VTS status page (`/status/format/json`)
→ Use nginx-vts-exporter to send data to Prometheus
→ Grafana processes the data
→ Alertmanager generates alerts

Alternatively, you can use:

```text
/status(format/prometheus) → Directly send data to Prometheus
```

However, in many practical scenarios, the first approach is still preferred:

### Features:

- **Richer metrics**: Provides detailed server zone, upstream, and cache-related data.
- **Better visualization**: Grafana dashboards tend to be more informative.
- **Closer to business perspective**: Offers insights into access layer operations.

### Key Metrics Available:

- Server zone-specific request volumes
- 2xx/3xx/4xx/5xx error rates by server zone
- InNeed to check the status code at the upstream dimension.

Need to examine the metrics at the backend node level.

Hope that Grafana panels can be more comprehensive.

Would like the metrics at the access layer to be more aligned with the business perspective.static_configs:
      - targets:
          - "10.0.0.21:9113"
          - "10.0.0.22:9113"
        labels:
          env: "prod"
          service: "nginx"
          monitor_type: "stub_status"
```Add the following to the `http` block:

```nginx
vhost_traffic_status_zone;
vhosttraffic_status_filter_by_host on;
```

Configure the status page:

```nginx
server {
    listen 127.0.0.1:8089;
    server_name localhost;

    access_log off;

    location /status {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;

        allow 127.0.0.1;
        deny all;
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

---

## Scenario 27: Accessing the VTS Status Page

HTML page:

```bash
curl http://127.0.0.1:8089/status
```

JSON format:

```bash
curl http://127.0.0.1:8089/status/format/json
```

Prometheus format:

```bash
curl http://127.0.0.1:8089/status/format/prometheus
```

If using `nginx-vts-exporter`, it is usually accessed via:

```text
/status FORMAT/json
```

---

## Scenario 28: Allowing the Monitoring Machine to Access the VTS Status Page

If the exporter is located on the monitoring machine, you need to allow access from that machine:

```nginx
server {
    listen 8089;
    server_name localhost;

    access_log off;

    location /status {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;

        allow 127.0.0.1;
        allow 10.0.0.10;
        deny all;
    }
}
```

Verify the access:

```bash
curl http://NginxNodeIP:8089/status/format/json
```

Security recommendations:

```text
Do not expose the VTS status page to the public network.

Only allow access from the local machine or a designated monitoring IP range.

You can listen on 127.0.0.1 or an internal IP address.

Avoid using a public domain name to serve the /status page.
```

---

## Chapter 15: Method 2: Deploying nginx-vts-exporter

---

## Scenario 29: Running nginx-vts-exporter as a Binary

Example:

```bash
/usr/local/bin/nginx-vts-exporter \
  -nginx.scrape_uri=http://127.0.0.1:8089/status/format/json
```

The default metrics port is usually:

```text
9913
```

Verify the access:

```bash
curl http://127.0.0.1:9913/metrics
```

View the metrics:

```bash
curl -s http://127.0.0.1:9913/metrics | grep nginx
```

---

## Scenario 30: Running nginx-vts-exporter in Docker

Example:

```bash
docker run -d \
  --name nginx-vts-exporter \
  --restart=always \
  --network host \
  -e NGINX_STATUS="http://127.0.0.1:8089/status/format/json" \
  -e METRICS_ADDR=":9913" \
  sophos/nginx-vts-exporter:latest
```

View the container:

```bash
docker ps | grep nginx-vts-exporter
```

View the logs:

```bash
docker logs -f nginx-vts-exporter
```

Verify the access:

```bash
curl http://127.0.0.1:9913/metrics
```

---

## Scenario 31: systemd Example for nginx-vts-exporter

Create a service file:

```bash
vi /etc/systemd/system/nginx-vts-exporter.service
```

Content:

```ini
[Unit]
Description=Nginx VTS Exporter
After=network.target nginx.service

[Service]
Type=simple
ExecStart=/usr/local/bin/nginx-vts-exporter \
  -nginx.scrape_uri=http://127.0.0.1:8089/status/format/json
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload the system configuration:

```bash
systemctl daemon-reload
```

Start the service:

```bash
systemctl enable nginx-vts-exporter
```

```bash
systemctl start nginx-vts-exporter
```

Check the status:

```bash
systemctl status nginx-vts-exporter
```

View the logs:

```bash
journalctl -u nginx-vts-exporter -n 100
```

Verify## Scenario 34: Method 2 Grafana Panels

Suitable for displaying:

```text
Nginx server zone request volume

Nginx server zone status code distribution

Nginx server zone traffic

Nginx upstream request volume

Nginx upstream status code distribution

Nginx upstream backend node response time

Nginx upstream backend node traffic

Nginx cache hit rate

Error rates for each server_name

Error rates for each upstream
```

This is also why:

```text
Grafana panels for the nginx-vts solution usually contain more and richer information
```

---

## Chapter 19: Blackbox Exporter HTTP Monitoring

---

## Scenario 35: Why HTTP Monitoring Is Still Needed

Neither Method 1 nor Method 2 can completely replace HTTP monitoring.

Reasons:

```text
The normal operation of the Exporter does not mean that the business URL is functioning properly.

Normal Nginx metrics do not indicate whether the HTTPS certificate is valid.

A normal stub_status does not guarantee that the upstream is working correctly.

The normal operation of vts does not mean that the business API returns a 2xx status code.

Configuration errors may only affect a specific server_name.
```

Additional methods include:

```text
blackbox_exporter

curl for periodic monitoring

SLB health checks

Business /health interface
```

---

## Scenario 36: Prometheus Blackbox Configuration Example

```yaml
scrape_configs:
  - job_name: "blackbox-nginx-http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com/health
          - https://admin.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: target
      - target_label: __address__
        replacement: 10.0.0.10:9115
```

Common metrics:

```text
probe_success

probe_duration_seconds

probe_http_status_code

probe_ssl_earliest_certexpiry
```

---

## Chapter 20: Certificate Expiration Monitoring

---

## Scenario 37: Calculating Remaining Days Until Certificate Expiration with PromQL

```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400
```

To check if it will expire within 30 days:

```promql
(probe_ssl_earliest_certexpiry - time()) / 86400 < 30
```

To check if it will expire within 7 days:

```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
```

---

## Scenario 38: Manually Checking Certificates

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

To view only the expiration date:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate
```

---

## Chapter 21: Metricization of access.log

---

## Scenario 39: Why access.log Is Still Important

Even with vts, access.log remains crucial.

Because the logs provide information such as:

```text
Specific URI

Specific client IP address

Specific User-Agent

Specific request_id

Specific request_uri

Specific upstream_addr

Complete context of a single request

Original records of abnormal requests
```

Recommended JSON log fields include:

```text
time

hostname

remote_addr

realip_remote_addr

xff

host

method

uri

request_uri

status

request_time

upstream_addr

upstream_status

upstream_response_time

user_agent

request_id
```

---

## Scenario 40: Example of JSON access.log Format

```nginx
log_format access_json escape=json
    '{'
    '"time":"$time_iso8601",'
    '"hostname":"$hostname",'
    '"remote_addr":"$remote_addr",'
    '"realip_remote_addr":"$realip_remote_addr",'
    '"xff":"$http_x_forwarded_for",'
    '"host":"$host",'
    '"method":"$request_method",'
    '"uri":"$uri",'
    '"request_uri":"$request_uri",'
    '"status":$status,'
    '"request_time":"$request_time",'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_response_time":"$upstream_response_time",'
    '"user_agent":"$http_user_agent",'
    '"cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | headcurl http://127.0.0.1:9113/metrics```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate
```

---

## Connection Status

```bash
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
```

```bash
ss -ant state established | wc -l
```

```bash
ss -ant state time-wait | wc -l
```

```bash
ss -antp | grep nginx | head
```

---

## Analysis of access.log

```bash
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

```bash
awk '$9 >= 500 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 499 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

```bash
awk '$9 == 429 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## Analysis of JSON access.log

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status >= 500) | [.time, .hostname, .remote_addr, .uri, .status, .upstream_addr, .upstream_status, .upstream_response_time] | @tsv'
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select((.request_time | tonumber) > 1) | [.time, .hostname, .uri, .status, .request_time, .upstream_response_time, .upstream_addr] | @tsv'
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 499) | [.time, .remote_addr, .uri, .request_time] | @tsv'
```

```bash
cat /var/log/nginx/example.access.json.log | jq -r 'select(.status == 429) | .remote_addr' | sort | uniq -c | sort -nr | head
```

---

## Analysis of error.log

```bash
grep -i "upstream" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "upstream timed out" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -Ei "connect\(\) failed|connection refused" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "no live upstreams" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "too many open files" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -i "worker_connections are not enough" /var/log/nginx/error.log | tail -n 100
```

```bash
grep -Ei "limiting requests|limiting connections" /var/log/nginx/error.log | tail -n 100
```

---

## Twenty-Eight, One-Sentence Summary

There are two common approaches to monitoring Nginx with Prometheus:

```text
Method 1:

stub_status

→ nginx/nginx-prometheus-exporter

→ Prometheus

→ Grafana
```

```text
Method 2:

nginx-module-vts

→ /status/format/json

→ nginx-vts-exporter

→ Prometheus

→ Grafana
```

When choosing between them:

```text
For basic monitoring:
→ Method 1 is preferred.

If you need more extensive Grafana dashboards:
→ Method 2 offers better functionality.

For lower maintenance costs:
→ Method 1 is generally better.

For detailed metrics like upstream/servers:
→ Method 2 provides more detailed information.
```

In terms of Grafana's richness in features:

```text
Method 2 combines nginx-vts and nginx-vts-exporter,

whereas Method 1 uses stub_status and nginx-prometheus-exporter.
```

However, in production environments:

```text
Do not blindly adopt vts just for its additional features.

vts involves compiling third-party modules and requires ongoing maintenance.

For general use cases, stick with stub_status + exporter + a logging platform.

Consider using vts only when you need detailed metrics such as upstream servers, status codes, or traffic.
